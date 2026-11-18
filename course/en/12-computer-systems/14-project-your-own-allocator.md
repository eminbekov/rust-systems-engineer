<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 12.13 How memory allocators work](13-how-memory-allocators-work.md) · [📋 Phase 12](README.md) · [12.15 Checkpoint →](15-checkpoint.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/12-computer-systems/14-project-your-own-allocator.md) <!-- nav -->

# Lesson 12.14 — Project: your own memory allocator

> **Phase 12 — Computer systems** · Lesson 14 of 15 · ⏱ about 8 hours (4 sittings)

## What you will learn

- How to turn the allocator ideas of Lesson 12.13 into two real `GlobalAlloc` implementations: a bump allocator and a free-list allocator.
- How to keep raw-pointer code correct: `// SAFETY:` comments, `# Safety` documentation, small unsafe blocks, and Miri on every test.
- How to test an allocator: unit tests for each rule, a random stress test that checks every byte, and a test with several threads.
- How to run real programs on your own allocator and read its statistics.

## Before you start

- You finished [Lesson 12.13 — How memory allocators work](13-how-memory-allocators-work.md): `Layout`, `GlobalAlloc`, `align_up`, bump allocation, free lists, splitting and coalescing.
- You can write `unsafe` code and run Miri ([Lesson 06.6](../06-rust-advanced/06-unsafe-rust-and-miri.md)), and you know const generics such as `[u8; N]` ([Lesson 05.1](../05-rust-intermediate/01-generics.md)) and `thread::scope` ([Lesson 05.10](../05-rust-intermediate/10-threads-channels-and-shared-state.md)).
- Create the project as a library with its own Git history, as in earlier projects:

```bash
cd ~/projects/rust
cargo new --lib code/p12-allocator
cd code/p12-allocator
mkdir examples tests
```

## New words

| Word | Meaning |
|---|---|
| **spin lock** | A lock that waits by checking an atomic flag again and again ("spinning") instead of asking the operating system to put the thread to sleep. It never allocates, so an allocator can use it. Lesson 14.7 builds one in depth. |
| **lock guard** | A value that proves you hold a lock and releases the lock when it is dropped (RAII, Resource Acquisition Is Initialization, Lesson 08.7). |
| **stress test** | A test that performs thousands of random operations and checks, after each one, that nothing broke. |
| **invariant** | A rule that must always be true for a data structure, for example "the free list is sorted by address and no two free blocks touch". |
| **abort** | Stopping the process immediately, without unwinding. Rust aborts when the global allocator returns null (the signal `SIGABRT`, exit code 134). |
| **`NonNull<T>`** | A raw pointer that is never null (`std::ptr::NonNull`). `Option<NonNull<T>>` has the same size as a pointer (the niche from Lesson 12.11). |

## The project

### What you will build

A library crate `p12_allocator` with two allocators that can replace Rust's global allocator:

1. **`BumpAlloc<N>`**: a bump allocator over an arena of `N` bytes that lives inside the allocator value itself.
2. **`FreeListAlloc<N>`**: a first-fit free-list allocator over its own arena of `N` bytes, with splitting and coalescing. Its core is `FreeListHeap`, a plain struct that you can test without making it global.

Two example programs run real Rust code (a `Vec`, `format!`, a `HashMap`) on top of them. At the end every test also passes under Miri.

```text
   p12-allocator/
   ├── Cargo.toml
   ├── README.md
   ├── src/
   │   ├── lib.rs          ← module list and re-exports
   │   ├── spin.rs         ← SpinLock (given code, below)
   │   ├── align.rs        ← align_up                      (M1)
   │   ├── bump.rs         ← BumpAlloc<N>                  (M2)
   │   └── free_list.rs    ← FreeListHeap, FreeListAlloc<N> (M3, M4)
   ├── examples/
   │   ├── bump_global.rs        (M2)
   │   └── free_list_global.rs   (M4)
   └── tests/
       ├── stress.rs       (M5)
       └── threads.rs      (M4)
```

### Contract

**No dependencies.** Only the standard library. Edition 2024.

**Public API** (`src/lib.rs` re-exports these names):

```text
pub fn align_up(addr: usize, align: usize) -> Option<usize>

pub struct BumpAlloc<const N: usize>
    pub const fn new() -> BumpAlloc<N>
    pub fn used(&self) -> usize                  // bytes used so far, padding included
    unsafe impl GlobalAlloc

pub struct FreeListHeap
    pub const fn empty() -> FreeListHeap
    pub unsafe fn add_region(&mut self, start: *mut u8, size: usize)
    pub fn alloc(&mut self, layout: Layout) -> *mut u8        // null if nothing fits
    pub unsafe fn dealloc(&mut self, ptr: *mut u8, layout: Layout)
    pub fn free_bytes(&self) -> usize
    pub fn free_blocks(&self) -> usize

pub struct FreeListAlloc<const N: usize>
    pub const fn new() -> FreeListAlloc<N>
    pub fn stats(&self) -> (usize, usize)        // (free bytes, free blocks)
    unsafe impl GlobalAlloc

pub struct SpinLock<T>, pub struct SpinGuard<'a, T>   // given code
```

**Behaviour:**

1. `align_up(addr, align)` rounds up to a multiple of `align` (a power of two) and returns `None` instead of overflowing.
2. Every block returned by `alloc` has at least `layout.size()` bytes, starts at a multiple of `layout.align()`, lies completely inside the arena, and does not overlap any other live block.
3. When no block fits, `alloc` returns a null pointer. It never panics and never allocates.
4. `BumpAlloc` reuses memory only when **all** its blocks are freed (`used()` is then 0 again).
5. `FreeListHeap` keeps its free list **sorted by address**, uses **first fit**, **splits** a block when the rest can still hold a free-block header, and **coalesces** a freed block with free neighbours on both sides. When every block has been freed, the heap is again **one** free block with all its bytes.
6. A free-list block takes at least 16 bytes (room for the header described in M3), and its size is a multiple of 8. `free_bytes()` counts these real sizes.
7. Both allocators are safe to use from many threads at once: all shared state is behind a `SpinLock`.

**Code rules:** every `unsafe` block has a `// SAFETY:` comment that says why it is correct; every `pub unsafe fn` has a `# Safety` section in its documentation; `cargo clippy --all-targets -- -D warnings` and `cargo fmt --check` are clean; there is a short `README.md` (what the crate is, how to run the tests and examples, the design in five lines); and every milestone ends with one Git commit.

**Given code.** An allocator cannot use `std::sync::Mutex` safely: a lock must never allocate, and you would have to prove that for every platform. Use this small spin lock instead. Put it into `src/spin.rs` as it is; you do not need to understand every detail yet (Lesson 14.7 explains it):

```rust
//! A minimal spin lock. Given code: Lesson 14.7 explains how it works.

use std::cell::UnsafeCell;
use std::ops::{Deref, DerefMut};
use std::sync::atomic::{AtomicBool, Ordering};

/// A lock that waits by spinning. It never allocates, so an allocator may use it.
pub struct SpinLock<T> {
    locked: AtomicBool,
    value: UnsafeCell<T>,
}

// SAFETY: only one thread at a time can reach `value`, through a `SpinGuard`.
unsafe impl<T: Send> Sync for SpinLock<T> {}

impl<T> SpinLock<T> {
    pub const fn new(value: T) -> SpinLock<T> {
        SpinLock {
            locked: AtomicBool::new(false),
            value: UnsafeCell::new(value),
        }
    }

    pub fn lock(&self) -> SpinGuard<'_, T> {
        while self.locked.swap(true, Ordering::Acquire) {
            std::hint::spin_loop();
        }
        SpinGuard { lock: self }
    }
}

/// Gives access to the value; unlocks when dropped.
pub struct SpinGuard<'a, T> {
    lock: &'a SpinLock<T>,
}

impl<T> Deref for SpinGuard<'_, T> {
    type Target = T;
    fn deref(&self) -> &T {
        // SAFETY: the guard proves that this thread holds the lock.
        unsafe { &*self.lock.value.get() }
    }
}

impl<T> DerefMut for SpinGuard<'_, T> {
    fn deref_mut(&mut self) -> &mut T {
        // SAFETY: the guard proves that this thread holds the lock.
        unsafe { &mut *self.lock.value.get() }
    }
}

impl<T> Drop for SpinGuard<'_, T> {
    fn drop(&mut self) {
        self.lock.locked.store(false, Ordering::Release);
    }
}
```

In short: `swap(true, …)` sets the flag and returns the old value; while the old value was `true`, another thread holds the lock, so we try again. The guard gives `&T` / `&mut T` through `Deref` and `DerefMut` (Lesson 05.9) and clears the flag in `Drop`.

**Required tests** (unit tests in the module they test; the names are part of the contract, so the expected outputs below match):

| Module | Test | Checks |
|---|---|---|
| `align` | `align_up_keeps_aligned_addresses` | `align_up(0, 8)` and `align_up(4096, 4096)` do not move |
| `align` | `align_up_rounds_up` | `1 → 8`, `13 → 16` (align 4), `4097 → 8192` (align 4096) |
| `align` | `align_up_reports_overflow` | `align_up(usize::MAX, 8)` is `None` |
| `bump` | `bump_blocks_are_aligned_and_separate` | a `u8` and then a `u64`: both non-null, the second aligned to 8 and after the first |
| `bump` | `bump_returns_null_when_full` | in a 64-byte arena, a 64-byte block works and then 1 more byte gives null |
| `bump` | `bump_resets_when_everything_is_freed` | alloc, dealloc, then `used() == 0` and allocation works again |
| `free_list` | `free_list_starts_with_one_block` | a fresh 1024-byte region is 1 block of 1024 bytes |
| `free_list` | `free_list_splits_and_reuses` | `alloc(100, align 8)` leaves 1024 − 104 free; after freeing, the same address comes back |
| `free_list` | `free_list_respects_alignment` | after an 8-byte block, a block with alignment 64 is aligned to 64 |
| `free_list` | `free_list_returns_null_when_full` | a 1024-byte block fills the region; 1 more byte gives null |
| `free_list` | `free_list_coalesces_neighbours` | three 32-byte blocks a, b, c; free a and c (2 free blocks), then b: 1 block of 1024 bytes |

For the `free_list` tests, use a region that is aligned to 16 so that the numbers are exact: `#[repr(align(16))] struct Region([u8; 1024]);`.

### Step-by-step plan

#### M1 — `align_up` that cannot overflow

**Goal:** `src/align.rs` with `align_up` and its three tests; `src/lib.rs` with `mod align; mod spin;` and `pub use` lines; the given `src/spin.rs`.

**Guidance:** start from the version in Lesson 12.13, but use `checked_add` so that a huge address gives `None` instead of wrapping around to a small one. Keep the `debug_assert!` that `align` is a power of two.

<details>
<summary>Hint</summary>

`addr.checked_add(align - 1)` returns an `Option<usize>`; `.map(|a| a & !(align - 1))` finishes the rounding inside the `Option`.

</details>

**Done when:**

```bash
cargo test --lib align_up
```

```text
running 3 tests
test align::tests::align_up_keeps_aligned_addresses ... ok
test align::tests::align_up_reports_overflow ... ok
test align::tests::align_up_rounds_up ... ok

test result: ok. 3 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.01s
```

(Tests run in parallel, so the order of the `test …` lines can differ. Later, when other tests exist, the "filtered out" number grows; that is fine.) Then `git add -A` and `git commit -m "feat(p12-allocator): align_up with overflow check"`.

#### M2 — The bump allocator

**Goal:** `BumpAlloc<N>` in `src/bump.rs` implements `GlobalAlloc`, passes its three tests, and runs a real program as the global allocator.

**Guidance:**

1. Fields: the arena `UnsafeCell<[u8; N]>` (an `UnsafeCell` because the allocator hands out writable memory through `&self`), and `SpinLock<BumpState>` where `BumpState` holds `next` (an offset into the arena) and `live` (number of live blocks).
2. `new` must be a `const fn`, because a `static` is initialised at compile time: `UnsafeCell::new([0; N])` and `SpinLock::new(…)` are both `const`.
3. `BumpAlloc` contains an `UnsafeCell`, so it is not `Sync` automatically. Add `unsafe impl<const N: usize> Sync for BumpAlloc<N> {}` with a `// SAFETY:` comment that explains why sharing is fine.
4. In `alloc`: take the lock; compute the aligned **address**, not the aligned offset (the arena itself may start at any address); check the end against `N`; return `null_mut()` on any failure.
5. Turn the offset into a pointer with `base.wrapping_add(offset)`, where `base` is `self.arena.get().cast::<u8>()`. Do not turn a number back into a pointer with `as *mut u8`: the pointer would lose the information about which allocation it belongs to, and Miri reports that.
6. Clippy will ask for a `Default` implementation, because `new()` takes no arguments; add one that calls `new()`.

<details>
<summary>Hint</summary>

The core of `alloc`, without the error handling: `let start = align_up(base as usize + state.next, layout.align())?; let offset = start - base as usize; let end = offset + layout.size();` then compare `end` with `N`. Because `alloc` returns a pointer and not an `Option`, use `let … else { return ptr::null_mut(); }` for each step that can fail.

</details>

Now write `examples/bump_global.rs`. It installs the bump allocator for the whole program and then asks for more memory than the arena has:

```rust
use p12_allocator::BumpAlloc;

#[global_allocator]
static GLOBAL: BumpAlloc<{ 1024 * 1024 }> = BumpAlloc::new();

fn main() {
    let numbers: Vec<u64> = (1..=1000).collect();
    let text = format!("sum = {}", numbers.iter().sum::<u64>());
    println!("{text}");
    println!("used {} of {} bytes", GLOBAL.used(), 1024 * 1024);

    // Now ask for more than the arena holds.
    let big = vec![0u8; 2 * 1024 * 1024];
    println!("never printed: {}", big.len());
}
```

**Done when:**

```bash
cargo test --lib bump
cargo run --example bump_global; echo "exit code $?"
```

```text
running 3 tests
test bump::tests::bump_blocks_are_aligned_and_separate ... ok
test bump::tests::bump_resets_when_everything_is_freed ... ok
test bump::tests::bump_returns_null_when_full ... ok

test result: ok. 3 passed; 0 failed; 0 ignored; 0 measured; 3 filtered out; finished in 0.00s

sum = 500500
used 9720 of 1048576 bytes
memory allocation of 2097152 bytes failed
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
exit code 134
```

Your `used` number may be a little different: it includes the memory that the standard library allocated before and during `main` (for example the buffer of standard output). The last lines are the important ones: your `alloc` returned null, so Rust reported the failed allocation and aborted (exit code 134). Commit: `feat(p12-allocator): bump allocator`.

#### M3 — The free-list heap: alloc, split, dealloc

**Goal:** `FreeListHeap` in `src/free_list.rs` with `empty`, `add_region`, `alloc`, `dealloc`, `free_bytes`, `free_blocks`, and the first four `free_list` tests. Coalescing comes in M4.

**Guidance:**

1. The free list lives inside the free memory. Each free block starts with a header:

```text
struct FreeBlock {
    size: usize,                       // size of this free block in bytes
    next: Option<NonNull<FreeBlock>>,  // next free block, higher address
}
```

That is 16 bytes with alignment 8, so the smallest block is 16 bytes and every block size is a multiple of 8 (contract rule 6). A helper `fn block_size(layout: Layout) -> Option<usize>` rounds `layout.size().max(16)` up to a multiple of 8. The heap itself holds only `head: Option<NonNull<FreeBlock>>`.

2. Write one private helper first: `unsafe fn insert(&mut self, ptr: *mut u8, size: usize)`. It walks the list to the first block with a higher address, writes a new `FreeBlock` at `ptr` with `ptr.cast::<FreeBlock>().write(…)`, and links it in. `add_region` (after aligning the start to 8 and cutting the size to a multiple of 8) and `dealloc` both just call `insert`.
3. `alloc` walks the list with a `prev` pointer. For each block, decide whether the request fits: the start address is the block address rounded up to `layout.align()` (at least 8). Any **front** gap must be 0 or at least 16 bytes (so that it can stay a free block), and the **rest** after the block must also be 0 or at least 16. If the block fits: unlink it, `insert` the front gap and the rest if they are not empty, and return the start.
4. `free_bytes` and `free_blocks` walk the list. An iterator built with `std::iter::from_fn` keeps them short.
5. `FreeListHeap` holds raw pointers, so it is not `Send`. M4 needs that; add `unsafe impl Send for FreeListHeap {}` with a `// SAFETY:` comment (the heap only points into memory that it manages).

<details>
<summary>Hint 1: the fit check</summary>

Write it as a pure function on numbers, which is easy to test: `fn place(block_addr, block_size, size, align) -> Option<(usize, usize)>` returns `(front gap, start address)`. If the first aligned start leaves a front gap of 1 to 15 bytes, try again from `block_addr + 16`. Return `None` if `start + size` passes the end of the block or if the rest is 1 to 15 bytes.

</details>

<details>
<summary>Hint 2: changing the list</summary>

To unlink, set either `prev.next` or `self.head` to the block's `next`. Get a `&mut FreeBlock` from a `NonNull` with `unsafe { block.as_mut() }`, and read with `block.as_ref()`. Copy `size` and `next` out of the block **before** you write a new header into the same memory.

</details>

**Done when:**

```bash
cargo test --lib free_list
cargo +nightly miri test --lib
```

The first command shows four tests: `free_list_respects_alignment`, `free_list_returns_null_when_full`, `free_list_splits_and_reuses` and `free_list_starts_with_one_block`, all `... ok`, ending in `test result: ok. 4 passed`. Miri prints the same test names and ends with `test result: ok. 10 passed` (3 + 3 + 4) and no "Undefined Behavior". Commit: `feat(p12-allocator): free-list heap with first fit and splitting`.

#### M4 — Coalescing and a global free-list allocator

**Goal:** freed blocks merge with their neighbours; `FreeListAlloc<N>` implements `GlobalAlloc`; the threads test and the second example pass.

**Guidance:**

1. Coalescing goes into `insert`, because the list is sorted: after linking the new block between `prev` and `next`, first check whether the new block ends exactly where `next` begins (then absorb `next`), then whether `prev` ends exactly where the new block begins (then let `prev` absorb the new block). This order handles the case where a block closes the gap between two free blocks.
2. Clippy may suggest a *let chain* for code like `if let Some(next) = current { if … { … } }`: `if let Some(next) = current && addr + size == next.as_ptr() as usize { … }` (Lesson 04.5).
3. `FreeListAlloc<N>` has an arena `UnsafeCell<[u8; N]>` and `SpinLock<(bool, FreeListHeap)>`. The `bool` says whether the arena was already given to the heap: `new()` is `const`, and a `const fn` cannot know the arena's final address, so the first `alloc` calls `add_region`.
4. `stats()` takes the lock and returns `(free_bytes(), free_blocks())`.

<details>
<summary>Hint</summary>

The coalescing test in numbers: after `a`, `b`, `c` (32 bytes each, at 0, 32, 64), freeing `a` gives the list `[0..32)`, `[96..1024)`; freeing `c` merges it with the block after it: `[0..32)`, `[64..1024)`, two blocks; freeing `b` touches both: `[0..1024)`, one block.

</details>

Write `tests/threads.rs`. It uses the allocator directly (not as the global allocator) from four threads at once:

```rust
use p12_allocator::FreeListAlloc;
use std::alloc::{GlobalAlloc, Layout};
use std::thread;

static ALLOC: FreeListAlloc<{ 64 * 1024 }> = FreeListAlloc::new();

#[test]
fn free_list_alloc_works_from_many_threads() {
    let rounds = if cfg!(miri) { 20 } else { 10_000 };
    thread::scope(|s| {
        for t in 0..4u8 {
            s.spawn(move || {
                let layout = Layout::from_size_align(48, 8).unwrap();
                for _ in 0..rounds {
                    // SAFETY: the layout has a non-zero size.
                    let p = unsafe { ALLOC.alloc(layout) };
                    assert!(!p.is_null());
                    // SAFETY: `p` points to 48 bytes that this thread owns.
                    unsafe {
                        p.write_bytes(t, 48);
                        assert!(std::slice::from_raw_parts(p, 48).iter().all(|&b| b == t));
                        ALLOC.dealloc(p, layout);
                    }
                }
            });
        }
    });
    assert_eq!(ALLOC.stats().1, 1, "all blocks merged again");
}
```

And `examples/free_list_global.rs`, which runs a word counter three times on your allocator and prints its state:

```rust
use p12_allocator::FreeListAlloc;
use std::collections::HashMap;

#[global_allocator]
static GLOBAL: FreeListAlloc<{ 4 * 1024 * 1024 }> = FreeListAlloc::new();

fn main() {
    let (free, blocks) = GLOBAL.stats();
    println!("at start of main: {free} bytes free in {blocks} blocks");

    for round in 1..=3 {
        let mut words: HashMap<String, usize> = HashMap::new();
        for i in 0..10_000 {
            *words.entry(format!("word{}", i % 500)).or_insert(0) += 1;
        }
        let (free, blocks) = GLOBAL.stats();
        println!(
            "round {round}: {} words, {free} bytes free in {blocks} blocks",
            words.len()
        );
    }

    let (free, blocks) = GLOBAL.stats();
    println!("at the end:  {free} bytes free in {blocks} blocks");
}
```

**Done when:**

```bash
cargo test --lib free_list
cargo test --test threads
cargo run --release --example free_list_global
```

```text
running 5 tests
test free_list::tests::free_list_coalesces_neighbours ... ok
test free_list::tests::free_list_respects_alignment ... ok
test free_list::tests::free_list_returns_null_when_full ... ok
test free_list::tests::free_list_splits_and_reuses ... ok
test free_list::tests::free_list_starts_with_one_block ... ok

test result: ok. 5 passed; 0 failed; 0 ignored; 0 measured; 6 filtered out; finished in 0.00s

running 1 test
test free_list_alloc_works_from_many_threads ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.02s

at start of main: 4193680 bytes free in 1 blocks
round 1: 500 words, 4150792 bytes free in 2 blocks
round 2: 500 words, 4150792 bytes free in 2 blocks
round 3: 500 words, 4150792 bytes free in 2 blocks
at the end:  4192592 bytes free in 1 blocks
```

How to read the example: the standard library used 624 bytes before `main` started (4,194,304 − 4,193,680). During a round, the map and its strings are alive; when `words` is dropped at the end of the round, everything is freed and coalesced, so every round starts from the same state and needs the same memory. At the end, 1,088 more bytes are in use than at the start of `main`: mostly the buffer of standard output (1 KiB), which was created by the first `println!` and lives until the program exits. Your numbers can differ a little; the pattern (the same numbers every round, one block at the end) must be the same. Commit: `feat(p12-allocator): coalescing and global free-list allocator`.

#### M5 — Stress test, Miri, README

**Goal:** a random stress test, a clean Miri run of everything, a README, and a clean quality gate.

Write `tests/stress.rs`. It performs thousands of random allocations and frees, fills every block with a marker byte, checks the marker before freeing, and finally checks that the heap is one block again:

```rust
use p12_allocator::FreeListHeap;
use std::alloc::Layout;

fn next_random(state: &mut u64) -> u64 {
    *state ^= *state << 13;
    *state ^= *state >> 7;
    *state ^= *state << 17;
    *state
}

#[test]
fn random_allocs_and_frees_keep_every_block_intact() {
    let steps = if cfg!(miri) { 300 } else { 20_000 };
    let mut region = vec![0u8; 64 * 1024];
    let mut heap = FreeListHeap::empty();
    // SAFETY: `region` lives until the end of the test and is used only by the heap.
    unsafe { heap.add_region(region.as_mut_ptr(), region.len()) };
    let initial = heap.free_bytes();

    let mut seed = 0x2545_F491_4F6C_DD1D;
    let mut live: Vec<(*mut u8, Layout, u8)> = Vec::new();
    for step in 0..steps {
        let r = next_random(&mut seed);
        if !r.is_multiple_of(3) || live.is_empty() {
            let size = 1 + (r >> 8) as usize % 300;
            let align = 1 << ((r >> 20) % 7); // 1, 2, 4, …, 64
            let layout = Layout::from_size_align(size, align).unwrap();
            let p = heap.alloc(layout);
            if p.is_null() {
                continue; // full for now: fine
            }
            assert_eq!(p as usize % align, 0, "misaligned block at step {step}");
            let fill = (step % 251) as u8;
            // SAFETY: `p` points to `size` bytes that we own.
            unsafe { p.write_bytes(fill, size) };
            live.push((p, layout, fill));
        } else {
            let index = (r >> 8) as usize % live.len();
            let (p, layout, fill) = live.swap_remove(index);
            // SAFETY: `p` points to `layout.size()` bytes that we still own.
            let bytes = unsafe { std::slice::from_raw_parts(p, layout.size()) };
            assert!(
                bytes.iter().all(|&b| b == fill),
                "block overwritten before step {step}"
            );
            // SAFETY: `p` came from `heap.alloc(layout)`.
            unsafe { heap.dealloc(p, layout) };
        }
    }
    for (p, layout, _) in live.drain(..) {
        // SAFETY: every remaining block came from `heap.alloc(layout)`.
        unsafe { heap.dealloc(p, layout) };
    }
    assert_eq!(heap.free_bytes(), initial);
    assert_eq!(
        heap.free_blocks(),
        1,
        "everything coalesced back into one block"
    );
}
```

An allocation in two of three steps and a free in the third keep the heap filling up and emptying, so splitting, coalescing and the "full" case are all exercised. `cfg!(miri)` is `true` only under Miri, which is very much slower, so the test does fewer steps there. Then write `README.md`.

**Done when:**

```bash
cargo test --test stress
cargo +nightly miri test
cargo clippy --all-targets -- -D warnings
cargo fmt --check
```

```text
running 1 test
test random_allocs_and_frees_keep_every_block_intact ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.07s
```

`cargo +nightly miri test` runs all tests under Miri and ends three times with `test result: ok`: 11 unit tests, the stress test, the threads test, and 0 doc tests, without any "Undefined Behavior" message (it takes about 10 seconds). `cargo clippy` prints only `Checking p12-allocator …` and `Finished …`, and `cargo fmt --check` prints nothing. Commit: `test(p12-allocator): stress test, Miri, README`.

> 🧪 **Testing:** When the stress test fails, it tells you the step. Make it fail early and repeatably: print the step, lower `steps` until it just fails, and check the free list (`free_blocks`, `free_bytes`) after every operation. Because the random generator has a fixed seed, every run is identical, which is what makes such a bug findable.

### Stretch goals

1. **Size classes.** Add `SegregatedAlloc`: separate free lists for blocks of 16, 32, 64, 128, 256 and 512 bytes, fed from a bump arena; bigger requests go to the free-list heap. Compare the example's speed with `FreeListAlloc`.
2. **Benchmark.** Build a version of `free_list_global` that uses `System` and compare both with `hyperfine` (Lesson 05.12) on a bigger workload. Write the numbers into the README, with the computer you used.
3. **Best fit.** Add a best-fit search mode and measure fragmentation: after the stress test's allocation phase, how many free blocks and how many free bytes are left with each strategy?
4. **`realloc` in place.** Override `GlobalAlloc::realloc`: if the block after the current one is free and big enough, grow into it without copying. Count how often it succeeds in the example.
5. **Peak usage.** Track the largest amount of memory in use at any time and print it in the examples.

## Common mistakes

**1. Reading the header after overwriting it.** In `alloc`, the rest of a split block can start inside the old header. Copy `size` and `next` into local variables first, then unlink, then `insert` the pieces.

**2. Aligning the offset instead of the address.** `align_up(offset, align)` is wrong when the arena does not start at an aligned address. Align `base + offset`, then subtract `base` again.

**3. Using `std::sync::Mutex`, `println!` or a `Vec` inside `alloc`.** Anything that may allocate can call back into your allocator while it holds its own lock: a deadlock or endless recursion (Lesson 12.13). Use the spin lock and plain fields; debug with the counters from `stats()` after the fact.

**4. Forgetting `cfg!(miri)` in long tests.** Under Miri a test that does 20,000 steps runs for many minutes. Scale the work down under Miri; it still finds the bugs, because it checks every single access.

**5. Testing only one thread.** The global allocator is called from every thread. Without the threads test, a missing lock can pass all other tests and corrupt memory in a real program much later.

## Quiz

1. Why must `BumpAlloc::new` and `FreeListAlloc::new` be `const fn`?

<details>
<summary>Answer</summary>

The global allocator is a `static`, and a `static` is initialised at compile time, so its initial value must come from a `const` expression. The first allocation happens before `main`, so there is no moment at run time to set it up.

</details>

2. Why does `FreeListAlloc` give the arena to the heap on the first `alloc` instead of in `new`?

<details>
<summary>Answer</summary>

In a `const fn` the final address of the arena is not known yet: the `static` is placed in memory by the linker and the loader (Lesson 12.12). The free-list headers must store real addresses, so the arena can only be added to the heap at run time, on first use.

</details>

3. Why is the free list kept sorted by address?

<details>
<summary>Answer</summary>

Then a block's neighbours in memory are its neighbours in the list, so coalescing only has to compare the new block with the block before and after it while inserting. In an unsorted list, finding neighbours would need a search through the whole list.

</details>

4. What does Rust do when the global allocator returns null, and how did you see it?

<details>
<summary>Answer</summary>

It calls the allocation error handler, which prints "memory allocation of N bytes failed" and aborts the process. The `bump_global` example showed this message and exit code 134 (`SIGABRT`).

</details>

5. Why is a spin lock acceptable here, although it wastes CPU time while waiting?

<details>
<summary>Answer</summary>

The lock is held only for a few hundred instructions (one list operation), so waiting is very short, and it never allocates or calls the operating system. A sleeping lock would have to be proven not to allocate on every platform.

</details>

## Summary

- You built two real global allocators: a bump allocator for speed and simplicity, and a free-list allocator that reuses memory with first fit, splitting and coalescing.
- `UnsafeCell` arenas, `unsafe impl Sync`, `const fn new`, and a lock that never allocates are the building blocks of every `GlobalAlloc`.
- Keep pointer provenance by deriving block pointers from the arena pointer (`wrapping_add`), and align addresses, not offsets.
- Test each rule in a unit test, check every byte in a random stress test with a fixed seed, test from several threads, and run everything under Miri.
- Real programs (`Vec`, `format!`, `HashMap`) ran on your allocator, and its statistics showed memory being freed and merged after every round.

## Sources and further reading

- [std::alloc::GlobalAlloc](https://doc.rust-lang.org/std/alloc/trait.GlobalAlloc.html) — the exact rules your implementation must follow.
- [Philipp Oppermann — Allocator Designs](https://os.phil-opp.com/allocator-designs/) — the same bump and linked-list designs, built step by step for a kernel.
- [std::ptr::NonNull](https://doc.rust-lang.org/std/ptr/struct.NonNull.html) and [std::ptr — Provenance](https://doc.rust-lang.org/std/ptr/index.html) — non-null raw pointers, and why pointers must be derived from the allocation they point into.
- [std::cell::UnsafeCell](https://doc.rust-lang.org/std/cell/struct.UnsafeCell.html) — the only legal way to change data behind a shared reference.
- [Miri on GitHub](https://github.com/rust-lang/miri) — `cfg(miri)`, flags, and what Miri can and cannot detect.
- [The Rust Programming Language — 20.1 Unsafe Rust](https://doc.rust-lang.org/book/ch20-01-unsafe-rust.html) — the official chapter on `unsafe`, `unsafe impl` and raw pointers.

---

[🏠 Start](../START-HERE.md) · [← 12.13 How memory allocators work](13-how-memory-allocators-work.md) · [📋 Phase 12](README.md) · [12.15 Checkpoint →](15-checkpoint.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
