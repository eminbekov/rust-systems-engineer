<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 17.8 Compaction and Bloom filters](08-compaction-and-bloom-filters.md) · [📋 Phase 17](README.md) · [17.10 Transactions and ACID →](10-transactions-and-acid.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/17-storage-and-databases/09-page-cache-and-buffer-pool.md) <!-- nav -->

# Lesson 17.9 — The page cache and the buffer pool

> **Phase 17 — Storage and database internals** · Lesson 9 of 15 · ⏱ about 3 hours (2 sittings)

## What you will learn

- What the kernel's page cache does for every program, and how to watch it fill and empty.
- Why a database keeps its **own** cache of pages, the buffer pool, on top of the page cache.
- How a buffer pool works inside: frames, a page table, dirty pages and LRU eviction.
- Why LRU fails on big scans, and what the clock algorithm is.
- How to watch PostgreSQL's buffer pool with `EXPLAIN (ANALYZE, BUFFERS)` and `pg_buffercache`.

## Before you start

- You know the page cache from [Lesson 13.11 — Memory-mapped files](../13-operating-systems/11-memory-mapped-files.md)
  and `write` versus `sync_all` from [Lesson 13.12 — File systems from the inside](../13-operating-systems/12-file-systems-inside.md).
- You know caches, hits and misses from [Lesson 12.9 — The memory hierarchy and caches](../12-computer-systems/09-memory-hierarchy-and-caches.md).
- You know the WAL (write-ahead log) from [Lesson 17.4 — The write-ahead log (WAL)](04-write-ahead-log.md) and PostgreSQL in Docker
  from [Lesson 16.5 — SQL and PostgreSQL basics](../16-backend-services/05-sql-and-postgresql-basics.md).
- Docker Desktop is running.

## New words

| Word | Meaning |
|---|---|
| **page cache** | RAM (Random Access Memory, the main memory of the computer) that the kernel uses to keep recently used file data. Every `read` and `write` goes through it. |
| **buffer pool** | A database's own cache of pages in its own memory. The database decides which pages stay and when changed pages are written. |
| **frame** | One slot of the buffer pool, big enough for exactly one page. |
| **page table** (of a buffer pool) | A map from page number to the frame that holds the page. Not the virtual-memory page table of Lesson 13.10. |
| **dirty page** | A page changed in memory and not yet written back to the file. |
| **eviction** | Removing a page from a cache to make room for another one. The removed page is the *victim*. |
| **LRU** (Least Recently Used) | An eviction rule: remove the page whose last use is the oldest. |
| **sequential flooding** | A big scan that pushes every useful page out of an LRU cache, so the cache gets no hits. |
| **clock algorithm** | A cheap approximation of LRU: every frame has a "used" bit, and a "hand" goes round the frames giving each used page a second chance. |
| **checkpoint** | The moment when a database writes all dirty pages to its data files, so that crash recovery can start from there. |
| **double caching** | The same page sits in RAM twice: in the database's buffer pool and in the kernel's page cache. |
| **`O_DIRECT`** | A Linux flag for `open` that skips the page cache: data moves directly between the disk and your buffer. |
| **hit ratio** | Hits divided by all accesses. A hit ratio of 0.99 means 99 of 100 pages were found in memory. |

## 1. Two caches between a database and the disk

When a program calls `read`, the kernel first looks in the **page cache**. If the file page is there, the kernel
copies it from RAM in microseconds. If not, it reads the page from the disk and keeps a copy for next time.
`write` changes the page in the page cache and returns; the kernel writes the dirty page to the disk later, or when
you call `sync_all` (Lesson 13.12).

A database usually adds a second cache **inside its own process**, the **buffer pool**:

```text
   +----------------------------------+
   |   database process               |
   |   +--------------------------+   |
   |   |  buffer pool (frames)    |   |  <- the database decides what stays
   |   +--------------------------+   |
   +----------------|-----------------+
                    | read / write of whole pages
   +----------------v-----------------+
   |   kernel: page cache             |  <- the kernel decides what stays
   +----------------|-----------------+
                    |
                  disk
```

A page can then be in RAM **twice**, in a frame and in the page cache: **double caching**. First let us watch the
page cache itself.

## 2. Watch the page cache

The Linux tool `fincore` ("file in core"; *core* is an old word for RAM) shows how much of a file is in the page
cache. Run a container with a Docker volume, because a volume is a normal ext4 file system (on the container's own
overlay file system `fincore` shows nothing):

```bash
docker run --rm -it -v p17-pagecache:/data ubuntu:26.04
apt-get update && apt-get install -y util-linux-extra
cd /data
```

`util-linux-extra` is the Ubuntu package that contains `fincore`. Inside the container:

```bash
dd if=/dev/urandom of=data.bin bs=1M count=256 status=none
fincore data.bin
sync data.bin
dd if=data.bin iflag=nocache count=0 status=none
fincore data.bin
dd if=data.bin of=/dev/null bs=1M
fincore data.bin
dd if=data.bin of=/dev/null bs=1M
```

```text
  RES PAGES  SIZE FILE
 256M 65536  256M data.bin
RES PAGES  SIZE FILE
 0B     0  256M data.bin
256+0 records in
256+0 records out
268435456 bytes (268 MB, 256 MiB) copied, 0.0969864 s, 2.8 GB/s
  RES PAGES  SIZE FILE
 256M 65536  256M data.bin
256+0 records in
256+0 records out
268435456 bytes (268 MB, 256 MiB) copied, 0.0184123 s, 15 GB/s
```

**What happens:**

1. `dd` writes 256 MiB (mebibytes of 1,048,576 bytes) of random bytes. All 65,536 pages (4 KiB each) are **resident** (`RES`): the data we just
   wrote sits in the page cache.
2. `sync data.bin` writes this file's dirty pages to the disk. The kernel can drop clean pages, not dirty ones.
3. `dd ... iflag=nocache count=0` copies nothing, but asks the kernel (with the system call `posix_fadvise`) to drop
   this file's pages. Now 0 pages are resident.
4. The first read must use the disk: 2.8 GB/s (gigabytes per second). It fills the cache again.
5. The second read comes from RAM: 15 GB/s, about five times faster. Your numbers will differ; on a slow cloud disk
   the gap can be a hundred times.

Leave with `exit` and remove the volume: `docker volume rm p17-pagecache`.

> 💡 **Tip:** `free -h` on Linux shows a column `buff/cache`. That memory is not "used up": the kernel gives it back
> the moment programs need it. A server with little "free" memory and a big page cache is healthy.

## 3. Why a database keeps its own cache

The kernel knows nothing about the meaning of the pages, and a database must control three things:

1. **The order of writes.** The WAL rule (Lesson 17.4): a changed data page may reach the disk only *after* the WAL
   records that describe the change are durable. The kernel writes dirty pages whenever it likes. A buffer pool keeps
   changed pages in its own memory and writes them only when the rule allows.
2. **What to keep.** The database knows that the root page of a B-tree (Lesson 17.6) is used by every lookup, and that
   pages of a one-time full scan will not be needed again. The kernel only sees "page 7 of some file".
3. **When to write.** At a **checkpoint**, the database writes all dirty pages and notes in the WAL "everything before
   this point is in the data files". After a crash, recovery replays the WAL only from the last checkpoint.

Databases deal with double caching differently:

| Database | Buffer pool | Page cache |
|---|---|---|
| PostgreSQL | `shared_buffers`, 128 MB (megabytes) by default; often set to about a quarter of RAM | used as a second level |
| MySQL InnoDB | gets most of the RAM | usually skipped with `O_DIRECT` |
| SQLite | a small cache per connection | relied on heavily |

`O_DIRECT` has strict rules (buffer address, offset and length aligned to the disk's block size), so only programs
with their own cache use it. On macOS the similar tool is `fcntl(fd, F_NOCACHE, 1)`.

## 4. A buffer pool in Rust

Build a small buffer pool, so that the words become code:

```bash
cd ~/projects/rust
cargo new --lib code/p17-bufpool
cd code/p17-bufpool
cargo add tempfile@3 --dev
```

```text
 file pages.db:  | page 0 | page 1 | page 2 | page 3 | page 4 | ...   (4096 bytes each)

 page_table: {0 -> 2, 3 -> 0, 4 -> 1}

 frames:   [0] page 3, dirty    [1] page 4, clean    [2] page 0, clean
            last_used = 7        last_used = 5        last_used = 9
```

Page `id` starts at byte `id * 4096` of the file. To read or write at a byte position (an **offset**) we use
`read_at` and `write_all_at` from the trait `std::os::unix::fs::FileExt`. They work like the system calls `pread`
and `pwrite`: the offset is a parameter, and the file's cursor does not move. Replace `src/lib.rs` with:

```rust
use std::collections::HashMap;
use std::fs::File;
use std::io;
use std::os::unix::fs::FileExt;

pub const PAGE_SIZE: usize = 4096;

#[derive(Clone)]
struct Frame {
    page_id: Option<u64>, // which page is in this frame (None = empty)
    data: Box<[u8; PAGE_SIZE]>,
    dirty: bool,    // changed in memory, not yet written to the file
    last_used: u64, // "time" of the last access, for LRU
}

#[derive(Debug, Default, Clone, Copy)]
pub struct Stats {
    pub hits: u64,
    pub misses: u64,
    pub disk_writes: u64,
}

pub struct BufferPool {
    file: File,
    frames: Vec<Frame>,
    page_table: HashMap<u64, usize>, // page id -> frame index
    tick: u64,                       // counts accesses, our "time"
    pub stats: Stats,
}

impl BufferPool {
    pub fn new(file: File, capacity: usize) -> BufferPool {
        assert!(capacity > 0, "a pool needs at least one frame");
        let empty = Frame {
            page_id: None,
            data: Box::new([0; PAGE_SIZE]),
            dirty: false,
            last_used: 0,
        };
        BufferPool {
            file,
            frames: vec![empty; capacity],
            page_table: HashMap::new(),
            tick: 0,
            stats: Stats::default(),
        }
    }

    /// Gives `f` read access to the page.
    pub fn read<R>(&mut self, id: u64, f: impl FnOnce(&[u8]) -> R) -> io::Result<R> {
        let i = self.frame_for(id)?;
        Ok(f(&self.frames[i].data[..]))
    }

    /// Gives `f` write access to the page and marks the page dirty.
    pub fn write<R>(&mut self, id: u64, f: impl FnOnce(&mut [u8]) -> R) -> io::Result<R> {
        let i = self.frame_for(id)?;
        self.frames[i].dirty = true;
        Ok(f(&mut self.frames[i].data[..]))
    }

    /// Writes every dirty page to the file and makes it durable.
    pub fn flush_all(&mut self) -> io::Result<()> {
        for i in 0..self.frames.len() {
            self.write_back(i)?;
        }
        self.file.sync_data()
    }

    /// The pages that are in memory now, sorted.
    pub fn cached_pages(&self) -> Vec<u64> {
        let mut ids: Vec<u64> = self.page_table.keys().copied().collect();
        ids.sort();
        ids
    }

    /// Finds the frame that holds page `id`, loading the page if needed.
    fn frame_for(&mut self, id: u64) -> io::Result<usize> {
        self.tick += 1;
        if let Some(&i) = self.page_table.get(&id) {
            self.stats.hits += 1;
            self.frames[i].last_used = self.tick;
            return Ok(i);
        }
        self.stats.misses += 1;
        let i = self.choose_victim();
        self.write_back(i)?;
        if let Some(old) = self.frames[i].page_id.take() {
            self.page_table.remove(&old);
        }
        let frame = &mut self.frames[i];
        frame.data.fill(0); // bytes past the end of the file stay zero
        let n = self.file.read_at(&mut frame.data[..], offset(id))?;
        assert!(n == 0 || n == PAGE_SIZE, "pages are always written whole");
        frame.page_id = Some(id);
        frame.last_used = self.tick;
        self.page_table.insert(id, i);
        Ok(i)
    }

    /// An empty frame if there is one, otherwise the least recently used one.
    fn choose_victim(&self) -> usize {
        let key = |i: &usize| (self.frames[*i].page_id.is_some(), self.frames[*i].last_used);
        (0..self.frames.len()).min_by_key(key).unwrap()
    }

    /// If the frame is dirty, writes its page to the file.
    fn write_back(&mut self, i: usize) -> io::Result<()> {
        let frame = &mut self.frames[i];
        if let (true, Some(id)) = (frame.dirty, frame.page_id) {
            self.file.write_all_at(&frame.data[..], offset(id))?;
            frame.dirty = false;
            self.stats.disk_writes += 1;
        }
        Ok(())
    }
}

/// Page `id` starts at this byte of the file.
fn offset(id: u64) -> u64 {
    id * PAGE_SIZE as u64
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn least_recently_used_page_is_evicted() {
        let file = tempfile::tempfile().unwrap();
        let mut pool = BufferPool::new(file, 2);
        for id in [0, 1, 0, 2] {
            pool.read(id, |_| ()).unwrap();
        }
        assert_eq!(pool.stats.hits, 1);
        assert_eq!(pool.cached_pages(), vec![0, 2]); // page 1 was used longest ago
    }
}
```

**What happens:**

1. A `Frame` holds one page (in a `Box`, on the heap) and three notes: which page it is, whether it is dirty, and
   when it was last used. `vec![empty; capacity]` clones the empty frame `capacity` times.
2. `read` and `write` take a **closure** that receives the page bytes. The caller never keeps a reference into the
   pool, so the borrow checker is happy and the pool may reuse a frame on the next call. `write` sets `dirty`.
3. `frame_for` is the heart. On a **hit** the page table knows the frame; we only update `last_used`. On a **miss**
   we choose a victim frame, write it back if it is dirty, forget its old page and read the new page into it.
   A page that does not exist in the file yet reads as zeros (`read_at` returns 0 at the end of the file).
4. `choose_victim` takes the frame with the smallest key `(is it used?, last_used)`. `false` sorts before `true`,
   so an empty frame wins; among full frames, the oldest `last_used` wins. That is **LRU**. Scanning all frames costs
   O(n); real pools use lists or the clock algorithm (Section 6) to avoid it.
5. `write_back` is called **before** the frame is reused. Forget it, and changes silently vanish (Common mistakes).
6. `flush_all` writes every dirty page, then calls `sync_data` so the pages survive a crash. On Linux that is
   `fdatasync`; on macOS the standard library calls `fcntl(fd, F_FULLFSYNC)` for both `sync_data` and `sync_all`,
   which also empties the drive's own write cache.

```bash
cargo test --lib
```

```text
running 1 test
test tests::least_recently_used_page_is_evicted ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

> 🧪 **Testing:** `tempfile::tempfile()` gives a real file that is deleted automatically, so every test starts with a
> clean private file. The test checks behaviour you can name ("the least recently used page goes"), not private
> fields.

## 5. Hits, misses and eviction

In this phase small programs live in `src/bin/`. Create `src/bin/lru.rs`:

```rust
use p17_bufpool::BufferPool;
use std::fs::OpenOptions;

fn main() -> std::io::Result<()> {
    let file = OpenOptions::new()
        .read(true)
        .write(true)
        .create(true)
        .truncate(true)
        .open("pages.db")?;
    let mut pool = BufferPool::new(file, 3); // room for 3 pages

    // store the number id * 100 at the start of pages 0 to 5
    for id in 0..6u64 {
        let bytes = (id * 100).to_le_bytes();
        pool.write(id, |page| page[..8].copy_from_slice(&bytes))?;
    }
    pool.flush_all()?;
    println!("after writing: {:?}", pool.stats);

    pool.stats = Default::default();
    for id in [0, 1, 2, 0, 3, 0, 4] {
        let hits_before = pool.stats.hits;
        let value = pool.read(id, first_u64)?;
        let hit = pool.stats.hits > hits_before;
        let kind = if hit { "hit " } else { "miss" };
        let cached = pool.cached_pages();
        println!("page {id}: {kind} value={value:<3} in memory: {cached:?}");
    }
    println!("{:?}", pool.stats);
    Ok(())
}

/// Reads the first 8 bytes of a page as a little-endian number.
fn first_u64(page: &[u8]) -> u64 {
    u64::from_le_bytes(page[..8].try_into().unwrap())
}
```

```bash
cargo run --bin lru
```

```text
after writing: Stats { hits: 0, misses: 6, disk_writes: 6 }
page 0: miss value=0   in memory: [0, 4, 5]
page 1: miss value=100 in memory: [0, 1, 5]
page 2: miss value=200 in memory: [0, 1, 2]
page 0: hit  value=0   in memory: [0, 1, 2]
page 3: miss value=300 in memory: [0, 2, 3]
page 0: hit  value=0   in memory: [0, 2, 3]
page 4: miss value=400 in memory: [0, 3, 4]
Stats { hits: 2, misses: 5, disk_writes: 0 }
```

**What happens:**

1. Writing 6 pages through 3 frames: 6 misses (the pool reads each page first; it cannot know the page is new).
   Pages 0, 1 and 2 were evicted while dirty (3 writes) and `flush_all` wrote pages 3, 4 and 5: 6 writes.
2. Reading page 0 evicts page 3, the least recently used; page 1 evicts page 4; page 2 evicts page 5.
3. Page 0 again: a **hit**. When page 3 comes, page 1 is the oldest (page 0 was just used), so page 1 goes.
4. The values come back correctly from the file, and `disk_writes: 0`: reading never makes a page dirty, so evicting
   a clean page costs nothing.

> ⚡ **Performance:** the hit ratio here is 2/7. Real databases aim for 0.99 and more, because one miss can cost as
> much as thousands of hits. When a database is slow, the hit ratio of its buffer pool is one of the first numbers to
> check.

## 6. When LRU fails: a big scan

Create `src/bin/scan.rs`: a pool with 9 frames reads the same 10 pages again and again.

```rust
use p17_bufpool::BufferPool;
use std::fs::OpenOptions;

fn main() -> std::io::Result<()> {
    let file = OpenOptions::new()
        .read(true)
        .write(true)
        .create(true)
        .truncate(true)
        .open("scan.db")?;
    let mut pool = BufferPool::new(file, 9); // 9 frames, but we scan 10 pages
    for round in 1..=3 {
        pool.stats = Default::default();
        for id in 0..10 {
            pool.read(id, |_| ())?;
        }
        let s = pool.stats;
        println!("scan {round}: {} hits, {} misses", s.hits, s.misses);
    }
    Ok(())
}
```

```bash
cargo run --bin scan
```

```text
scan 1: 0 hits, 10 misses
scan 2: 0 hits, 10 misses
scan 3: 0 hits, 10 misses
```

After pages 0 to 8 the pool is full, and page 9 evicts page 0. The next round starts with page 0: miss, and it evicts
page 1, the very next page we need. Every access throws out the page needed soonest: nine frames, **zero** hits.
This is **sequential flooding**. In a real database, one big report query over a huge table would push out the small
hot pages that every other query needs.

Databases fight it in two ways:

- **Scan rings.** PostgreSQL gives a big sequential scan a small private **ring** of frames (256 KiB) that it reuses,
  so the scan cannot flood the main pool.
- **The clock algorithm.** Exact LRU needs an update on every access. PostgreSQL uses a variant of the clock: every
  frame has a usage counter; a "hand" goes round the frames like a clock hand, lowers each counter it passes and
  evicts the first frame whose counter is zero. Often-used pages survive several rounds. Exercise 3 builds the
  simplest version, with one bit per frame.

## 7. PostgreSQL's buffer pool from the outside

Start a throw-away PostgreSQL 18 for the rest of this phase (no volume):

```bash
docker run -d --name p17-postgres -e POSTGRES_PASSWORD=dev-password \
  -p 127.0.0.1:5432:5432 postgres:18
docker exec -it p17-postgres psql -U postgres
```

Wait a few seconds after `docker run`: the server needs a moment before it accepts connections. If port 5432 is busy
because `p16-postgres` still runs, stop it first with `docker stop p16-postgres`. At the `postgres=#` prompt:

```sql
SHOW shared_buffers;
CREATE TABLE readings (id bigint PRIMARY KEY, sensor int, value double precision);
INSERT INTO readings SELECT i, i % 100, random() FROM generate_series(1, 200000) AS i;
SELECT pg_relation_size('readings') / 8192 AS pages;
VACUUM readings;
```

```text
 shared_buffers
----------------
 128MB
(1 row)

CREATE TABLE
INSERT 0 200000
 pages
-------
  1274
(1 row)

VACUUM
```

PostgreSQL pages are 8 KiB (8192 bytes); the table has 1,274 of them. (`VACUUM` tidies the new table; Lesson 17.12
explains it.) Quit with `\q`, restart the server so that its buffer pool is empty, connect again and read the whole
table twice:

```bash
docker restart p17-postgres
docker exec -it p17-postgres psql -U postgres
```

```sql
SET max_parallel_workers_per_gather = 0;
EXPLAIN (ANALYZE, BUFFERS, COSTS OFF) SELECT count(*) FROM readings;
EXPLAIN (ANALYZE, BUFFERS, COSTS OFF) SELECT count(*) FROM readings;
```

The important lines of the two plans:

```text
   ->  Seq Scan on readings (actual time=0.073..6.171 rows=200000.00 loops=1)
         Buffers: shared read=1274
 Execution Time: 11.252 ms

   ->  Seq Scan on readings (actual time=0.005..5.440 rows=200000.00 loops=1)
         Buffers: shared hit=1274
 Execution Time: 10.512 ms
```

**What happens:**

1. `SET max_parallel_workers_per_gather = 0` asks for a single process, which makes the plan shorter.
2. `EXPLAIN (ANALYZE, BUFFERS)` runs the query and counts pages: `shared read` means "not in the buffer pool, asked
   the kernel", `shared hit` means "found in the buffer pool". Lesson 17.13 reads the rest of a plan.
3. First run: 1,274 misses. Second run: 1,274 hits.
4. The first run was hardly slower. Restarting the container emptied PostgreSQL's buffer pool but **not** the kernel's
   page cache, so every "read" was served from RAM by the kernel: double caching, working in our favour.

Now watch dirty pages and a checkpoint. `pg_buffercache` is an extension shipped with PostgreSQL that shows the
frames of the buffer pool:

```sql
CREATE EXTENSION pg_buffercache;
UPDATE readings SET value = 0 WHERE id <= 1000;
SELECT count(*) FILTER (WHERE isdirty) AS dirty
FROM pg_buffercache WHERE relfilenode = pg_relation_filenode('readings');
CHECKPOINT;
SELECT count(*) FILTER (WHERE isdirty) AS dirty
FROM pg_buffercache WHERE relfilenode = pg_relation_filenode('readings');
```

```text
CREATE EXTENSION
UPDATE 1000
 dirty
-------
    19
(1 row)

CHECKPOINT
 dirty
-------
     0
(1 row)
```

The `UPDATE` made 19 pages of the table dirty and returned long before they were written: the change was already safe
because its WAL records were durable. `CHECKPOINT` wrote the dirty pages to the data files, like `flush_all` in your
pool. Keep the container for the next lessons; quit `psql` with `\q`.

> 🔒 **Security:** a buffer pool holds real rows in RAM, with personal data and sometimes password hashes. A memory
> dump of a database process (for debugging, or a core file after a crash) contains them. Protect such dumps like the
> database itself.

## Common mistakes

> ⚠️ **Common mistake:** forgetting to write a dirty page back before its frame is reused. Comment out
> `self.write_back(i)?;` in `frame_for` and run the test from Exercise 2:
>
> ```text
> thread 'tests::dirty_page_survives_eviction' (6049315) panicked at src/lib.rs:134:9:
> assertion `left == right` failed
>   left: 0
>  right: 1
> ```
>
> No error, no crash: the change is simply lost. Only a test catches this kind of bug.

> ⚠️ **Common mistake:** measuring a query or a file read once and drawing conclusions. Say whether you measured
> "cold" or "warm": the page cache and the buffer pool can make the second run many times faster.

> ⚠️ **Common mistake:** thinking that evicting a page "saves" it. Eviction only moves it to the page cache.
> Durability comes from `sync_data` / `sync_all` and, in a database, from the WAL.

## Try it yourself

1. Change the pool size in `lru.rs` from 3 to 2, predict the hits, run it. Then try 4. You should get
   `hits: 1, misses: 6` with 2 frames and `hits: 2, misses: 5` with 4: one more frame does not always help.
2. In `scan.rs`, change 9 frames to 10. Rounds 2 and 3 now print `10 hits, 0 misses`: when everything fits, LRU is
   perfect; one frame less, and it is useless.
3. In `psql`, run `EXPLAIN (ANALYZE, BUFFERS, COSTS OFF) SELECT * FROM readings WHERE id = 5;` twice. An index lookup
   touches a few pages (the second run printed `Buffers: shared hit=6` for us), not 1,274.

## Exercises

### Exercise 1 — Predict the hits (easy)

A pool with 2 frames uses LRU. The pages are read in the order `1, 2, 1, 3, 2, 1`. Which accesses are hits, and which
pages are in memory at the end? Work it out on paper, then check it with a small program in `src/bin/`.

<details>
<summary>Hint</summary>

Write the frames as a list from "least recently used" to "most recently used". A hit moves the page to the end; a
miss on a full list removes the first page.

</details>

<details>
<summary>Answer</summary>

```text
1: miss  [1]
2: miss  [1, 2]
1: hit   [2, 1]
3: miss  [1, 3]     (2 goes)
2: miss  [3, 2]     (1 goes)
1: miss  [2, 1]     (3 goes)
```

One hit, five misses; pages 1 and 2 stay. A copy of `lru.rs` with `BufferPool::new(file, 2)` and the loop over
`[1, 2, 1, 3, 2, 1]` prints `Stats { hits: 1, misses: 5, disk_writes: 0 }` and `[1, 2]`.

Why it works: LRU always evicts the page whose last use is the oldest; a hit only refreshes that time.

</details>

### Exercise 2 — Test that a dirty page survives eviction (medium)

Write a test `dirty_page_survives_eviction`: a pool with **one** frame writes the byte 42 into page 0, then reads page
1, which forces page 0 out. Check that the pool counted one disk write, that byte 0 of the *file* is 42, and that
reading page 0 through the pool gives 42 again.

<details>
<summary>Hint</summary>

`file.try_clone()` gives a second handle to the same file: give one to the pool and keep one in the test. Read the file
with `read_exact_at` from `FileExt` (already imported through `use super::*`).

</details>

<details>
<summary>Answer</summary>

```rust
    #[test]
    fn dirty_page_survives_eviction() {
        let file = tempfile::tempfile().unwrap();
        let mut pool = BufferPool::new(file.try_clone().unwrap(), 1);
        pool.write(0, |page| page[0] = 42).unwrap();
        pool.read(1, |_| ()).unwrap(); // the only frame is needed: page 0 must go
        assert_eq!(pool.stats.disk_writes, 1);

        let mut byte = [0u8; 1];
        file.read_exact_at(&mut byte, 0).unwrap();
        assert_eq!(byte[0], 42, "page 0 was not written to the file");
        assert_eq!(pool.read(0, |page| page[0]).unwrap(), 42);
    }
```

Why it works: with one frame, every miss evicts. The test reads the file behind the pool's back, so it proves that
eviction really wrote the page. Without `write_back` it fails (Common mistakes).

</details>

### Exercise 3 — The clock algorithm (hard)

Replace LRU with the clock algorithm. Give `Frame` a field `referenced: bool` (set to `true` on every hit and every
load) and the pool a field `hand: usize`. To choose a victim: take an empty frame if there is one; otherwise look at
the frame under the hand and move the hand one step. If that frame is referenced, clear its bit (a second chance) and
go on; if not, it is the victim. Test: a pool of 3 frames reads `0, 1, 2, 3`, then page 1, then page 4; page 1 must
survive and page 2 must go.

<details>
<summary>Hint</summary>

`choose_victim` now changes the pool (the hand and the bits), so it takes `&mut self`. The loop always ends: after at
most one full round, every bit is clear.

</details>

<details>
<summary>Answer</summary>

```rust
    /// An empty frame if there is one; otherwise the clock algorithm picks one.
    fn choose_victim(&mut self) -> usize {
        if let Some(i) = self.frames.iter().position(|f| f.page_id.is_none()) {
            return i;
        }
        loop {
            let i = self.hand;
            self.hand = (self.hand + 1) % self.frames.len();
            if self.frames[i].referenced {
                self.frames[i].referenced = false; // second chance
            } else {
                return i;
            }
        }
    }
```

Add `referenced: false` to the empty frame and `hand: 0` in `new`, and set `referenced = true` next to both
`last_used = self.tick` lines in `frame_for`. The test:

```rust
    #[test]
    fn clock_gives_a_second_chance() {
        let file = tempfile::tempfile().unwrap();
        let mut pool = BufferPool::new(file, 3);
        for id in [0, 1, 2, 3] {
            pool.read(id, |_| ()).unwrap();
        }
        assert_eq!(pool.cached_pages(), vec![1, 2, 3]);
        pool.read(1, |_| ()).unwrap(); // page 1 is used again
        pool.read(4, |_| ()).unwrap(); // page 1 survives, page 2 goes
        assert_eq!(pool.cached_pages(), vec![1, 3, 4]);
    }
```

Why it works: when page 3 comes, all three bits are set, so the hand clears them all, comes back to frame 0 and evicts
page 0; the hand now points at frame 1. Reading page 1 sets its bit again, so for page 4 the hand clears it, moves on
and evicts page 2. With this version `lru.rs` evicts page 0 at "page 3", which LRU kept: the clock only approximates
LRU, but it costs one bit per access and is much easier to make fast with many threads.

</details>

## Quiz

1. A program reads a file twice, and the second read is much faster. Where did the data come from the second time?

<details>
<summary>Answer</summary>

From the kernel's page cache in RAM: the first read loaded the file pages from the disk, and the kernel kept them.

</details>

2. Give two reasons why a database does not leave all caching to the kernel.

<details>
<summary>Answer</summary>

Any two of: it must control the **order** of writes (a data page only after its WAL records are durable); it knows
which pages are valuable (B-tree roots) and which are not (pages of a one-time scan); it decides **when** to write dirty
pages (checkpoints), which keeps recovery short.

</details>

3. What is a dirty page, and what must happen before its frame can hold another page?

<details>
<summary>Answer</summary>

A page changed in memory but not yet written to the file. It must be written back first (in a real database, after its
WAL records are durable); otherwise the change is lost.

</details>

4. Why did 10 pages scanned through 9 frames get zero hits with LRU?

<details>
<summary>Answer</summary>

Every miss evicted the least recently used page, which in a repeating scan is always the page needed next: sequential
flooding.

</details>

5. In `EXPLAIN (ANALYZE, BUFFERS)`, what is the difference between `shared hit` and `shared read`? Does `read` mean
   that the disk was used?

<details>
<summary>Answer</summary>

`hit`: the page was in PostgreSQL's buffer pool. `read`: it was not, so PostgreSQL asked the kernel. Not necessarily the
disk: the kernel may have had the page in its page cache.

</details>

## Summary

- Every file access goes through the kernel's page cache; `fincore` shows it, and a warm read is many times faster
  than a cold one.
- A database keeps its own buffer pool to control the order and the moment of writes and to choose what stays;
  PostgreSQL's is `shared_buffers`, 128 MB by default.
- A buffer pool is frames, a page table and dirty flags; a miss picks a victim, writes it back if dirty, and loads the
  new page.
- LRU evicts the page unused for the longest time and fails on big scans; databases add scan rings and the clock
  algorithm.
- Eviction is not durability: that needs `sync_data` / `sync_all` and the WAL.
- `EXPLAIN (ANALYZE, BUFFERS)` shows hits and reads; `pg_buffercache` shows dirty pages; `CHECKPOINT` writes them.

## Sources and further reading

- [PostgreSQL: Resource Consumption](https://www.postgresql.org/docs/current/runtime-config-resource.html) — `shared_buffers` and how to size it.
- [PostgreSQL: pg_buffercache](https://www.postgresql.org/docs/current/pgbuffercache.html) — the extension used in Section 7.
- [PostgreSQL: Using EXPLAIN](https://www.postgresql.org/docs/current/using-explain.html) — includes the `BUFFERS` option.
- [PostgreSQL: WAL Configuration](https://www.postgresql.org/docs/current/wal-configuration.html) — checkpoints and when dirty pages are written.
- [fincore(1)](https://man7.org/linux/man-pages/man1/fincore.1.html) and [open(2)](https://man7.org/linux/man-pages/man2/open.2.html) — the tool from Section 2, and `O_DIRECT` with its alignment rules.
- [Rust std: FileExt](https://doc.rust-lang.org/std/os/unix/fs/trait.FileExt.html) — `read_at` and `write_all_at`.
- [CMU 15-445/645 Database Systems](https://15445.courses.cs.cmu.edu/) — CMU is Carnegie Mellon University; a free university course; its buffer pool lectures go much deeper.

---

[🏠 Start](../START-HERE.md) · [← 17.8 Compaction and Bloom filters](08-compaction-and-bloom-filters.md) · [📋 Phase 17](README.md) · [17.10 Transactions and ACID →](10-transactions-and-acid.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
