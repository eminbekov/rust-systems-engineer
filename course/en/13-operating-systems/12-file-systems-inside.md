<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 13.11 Memory-mapped files](11-memory-mapped-files.md) · [📋 Phase 13](README.md) · [13.13 Non-blocking I/O and epoll →](13-non-blocking-io-and-epoll.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/13-operating-systems/12-file-systems-inside.md) <!-- nav -->

# Lesson 13.12 — File systems from the inside

> **Phase 13 — Operating systems from the inside** · Lesson 12 of 18 · ⏱ about 2.5 hours (2 sittings)

## What you will learn

- How the kernel's VFS (Virtual File System) layer lets one set of system calls work with many kinds of file systems.
- What an inode is, what a directory really contains, and how to read both from Rust.
- The difference between hard links and symbolic links, and what "deleting" a file really does.
- Why `write` does not mean "on the disk", and how `fsync` and the atomic-replace pattern make data survive a crash.

## Before you start

- You finished [Lesson 13.11 — Memory-mapped files](11-memory-mapped-files.md) and know the page cache.
- You know files and descriptors ([Lesson 13.4](04-files-and-file-descriptors.md)), the Linux directory tree ([Lesson 11.2](../11-linux-shell-and-containers/02-the-linux-file-system.md)) and permissions ([Lesson 11.4](../11-linux-shell-and-containers/04-users-groups-and-permissions.md)).
- Create `code/p13-fs` on the Mac and copy it to the VM as in Lesson 13.10. This time it is a library with binaries:

```bash
cd ~/projects/rust
cargo new --lib code/p13-fs
mkdir code/p13-fs/src/bin
```

The examples use only the standard library. They run in the VM, in a small file system that you create in
Section 2. The outputs below come from a test machine where the user is `root`; in your VM you see your own
user name.

## New words

| Word | Meaning |
|---|---|
| **file system** | A way to organise files on a storage device: where the data blocks are, which names exist, who may read them. ext4 is the usual one on Linux. |
| **VFS** (Virtual File System) | The layer in the kernel that offers the same calls (`open`, `read`, `rename`…) for every file system and passes them to the right one. |
| **mount** | Attaching a file system to a directory (the *mount point*), so that its files appear there. |
| **block device** | A device that stores data in fixed-size blocks, like a disk. A *loop device* makes an ordinary file look like one. |
| **inode** (index node) | The record of one file inside a file system: its type, size, owner, permissions, times and where its data is. It has a number, but no name. |
| **directory entry** | One line of a directory: a name and the inode number it points to. |
| **hard link** | A second directory entry for the same inode. All names are equal; none is the "original". |
| **symbolic link** (symlink) | A small special file that contains a path to another file. |
| **link count** | How many directory entries point to an inode. The data is freed when it reaches 0 and no process has the file open. |
| **unlink** | The system call behind "delete": it removes one directory entry. |
| **durability** | The promise that data survives a crash or power loss once the program was told "saved". |
| **`fsync`** | The system call that waits until a file's data and metadata are on the storage device. In Rust: `File::sync_all`. |
| **journal** | An area of the file system where ext4 first writes what it is going to change, so that a crash cannot leave the structure half-changed. |
| **sparse file** | A file with "holes": ranges that were never written and use no disk blocks. |

## 1. One interface, many file systems

When your program calls `open("/proc/uptime")`, `open("/home/student/notes.txt")` or
`open("/sys/fs/cgroup/cpu.max")`, it uses the same system call. Yet the first file is invented by the kernel
on the fly, the second is stored on a disk, and the third controls resource limits (Lesson 13.16). The part
of the kernel that makes this possible is the **VFS**:

```text
   your program:  open  read  write  rename  unlink  stat ...
                    |
           +--------v---------+
           |       VFS        |   same calls for everything
           +--+-----+-----+---+
              |     |     |
          +---v-+ +-v---+ +v-------+
          | ext4| |proc | | tmpfs  |  ... overlayfs, cgroup2, nfs
          +--+--+ +-----+ +--------+
             |     (no disk: RAM / kernel data)
     +-------v-------+
     |  block device |  a disk, or a loop device backed by a file
     +---------------+
```

Each file system is **mounted** on a directory. `findmnt` lists the mounts, and `/proc/self/mounts` is the
raw list. Containers (Lesson 13.15) use this heavily: their whole root directory is a mount of the
*overlayfs* file system.

## 2. Make a small file system to explore

The VM's main disk has thousands of files. A fresh, tiny ext4 file system is easier to understand, and you
can delete it afterwards. A file can act as a disk through a **loop device**. In the VM:

```bash
cd ~
truncate -s 64M disk.img          # an empty 64 MiB file
mkfs.ext4 -q disk.img             # write an empty ext4 file system into it
sudo mkdir -p /mnt/small
sudo mount -o loop disk.img /mnt/small
sudo chown "$USER" /mnt/small     # let your user create files there
findmnt -n -o TARGET,FSTYPE,SOURCE /mnt/small
cd /mnt/small
ls -la
```

```text
/mnt/small ext4 /dev/loop0
total 17
drwxr-xr-x 3 root root  1024 Nov 18 07:11 .
drwxr-xr-x 1 root root  4096 Nov 18 07:11 ..
drwx------ 2 root root 12288 Nov 18 07:11 lost+found
```

**What happens:**

1. `truncate -s 64M` creates a 64 MiB file without writing data (more on that in "Try it yourself").
2. `mkfs.ext4` ("make file system") writes the ext4 structures into the file: tables of inodes, maps of free
   blocks, the journal, and an empty root directory.
3. `mount -o loop` connects the file to a loop device `/dev/loop0` and mounts the file system on
   `/mnt/small`. Mounting needs root, so it uses `sudo`.
4. The new file system already contains `lost+found`, a directory where the repair tool `fsck` puts pieces
   of files it finds after a crash.

Look at its capacity:

```bash
df -h .
df -i .
```

```text
Filesystem      Size  Used Avail Use% Mounted on
/dev/loop0       55M   46K   51M   1% /mnt/small
Filesystem     Inodes IUsed IFree IUse% Mounted on
/dev/loop0      16384    12 16372    1% /mnt/small
```

`df -h` shows space; `df -i` shows **inodes**. ext4 creates a fixed number of inodes when it is made, here
16,384. Each file needs one. Remember this; it comes back in "Try it yourself".

## 3. Inodes and directories

A file in ext4 has two separate parts:

- An **inode**: a record with the file's type, size, owner, permissions, times, link count, and the
  addresses of its data blocks. The inode has a **number** but **no name**.
- One or more **directory entries** that give it a name.

A **directory** is just a special file whose content is a table of entries:

```text
directory /mnt/small (inode 2)            inode table
+-------------+-------+                   +-------+------+------+------+----------------+
| name        | inode |                   | inode | type | size | links| data blocks    |
+-------------+-------+                   +-------+------+------+------+----------------+
| .           | 2     |                   | 2     | dir  | 1024 | 4    | block 1201     |
| ..          | ...   |                   | 13    | file | 3    | 1    | block 8705     |
| lost+found  | 11    |                   | 2049  | dir  | 1024 | 2    | ...            |
| notes.txt   | 13    |  ------------->   | ...   |      |      |      |                |
| sub         | 2049  |                   +-------+------+------+------+----------------+
+-------------+-------+
```

Opening `/mnt/small/notes.txt` means: find the root directory, look up the entry `mnt`, then `small`, then
`notes.txt`, and read inode 13. Create some files:

```bash
echo hi > notes.txt
mkdir sub
ln -s notes.txt link
```

Rust shows inode data through the trait `std::os::unix::fs::MetadataExt`, which adds Unix-only methods to
`Metadata`. Create `src/bin/inspect.rs`:

```rust
use std::env;
use std::fs;
use std::os::unix::fs::MetadataExt;

fn main() -> std::io::Result<()> {
    for path in env::args().skip(1) {
        let meta = fs::symlink_metadata(&path)?; // do not follow symbolic links
        let kind = if meta.is_dir() {
            "directory"
        } else if meta.file_type().is_symlink() {
            "symlink"
        } else {
            "file"
        };
        println!(
            "{path}: {kind}, inode {}, links {}, size {} bytes, mode {:o}",
            meta.ino(),
            meta.nlink(),
            meta.size(),
            meta.mode() & 0o7777
        );
    }
    Ok(())
}
```

Build it in `~/p13-fs` and run it from `/mnt/small`:

```bash
~/p13-fs/target/debug/inspect notes.txt sub link .
```

```text
notes.txt: file, inode 13, links 1, size 3 bytes, mode 644
sub: directory, inode 2049, links 2, size 1024 bytes, mode 755
link: symlink, inode 14, links 1, size 9 bytes, mode 777
.: directory, inode 2, links 4, size 1024 bytes, mode 755
```

**What happens:**

1. `fs::symlink_metadata` calls `lstat`, which describes the link itself. `fs::metadata` (`stat`) would
   follow `link` and describe `notes.txt` instead.
2. `ino()` is the inode number, `nlink()` the link count, `size()` the size in bytes. `mode()` contains the
   type and the permission bits; `& 0o7777` keeps the permissions, printed in octal (Lesson 11.4).
3. `link` has size 9: its content is the 9 bytes of the text `notes.txt`. Its mode 777 is not used; access
   is checked on the target.
4. The root directory has inode 2 (ext4 always uses 2) and 4 links: the entry `small` in `/mnt`, its own
   `.`, and the `..` inside `lost+found` and `sub`. A new directory, like `sub`, always starts with 2.

`ls -i` and `stat` show the same data:

```bash
ls -li
stat notes.txt
```

```text
total 14
  14 lrwxrwxrwx 1 root root     9 Nov 18 07:11 link -> notes.txt
  11 drwx------ 2 root root 12288 Nov 18 07:11 lost+found
  13 -rw-r--r-- 1 root root     3 Nov 18 07:11 notes.txt
2049 drwxr-xr-x 2 root root  1024 Nov 18 07:11 sub
  File: notes.txt
  Size: 3         	Blocks: 2          IO Block: 1024   regular file
Device: 7,0	Inode: 13          Links: 1
Access: (0644/-rw-r--r--)  Uid: (    0/    root)   Gid: (    0/    root)
Access: 2026-11-18 07:11:54.548525013 +0000
Modify: 2026-11-18 07:11:54.548525013 +0000
Change: 2026-11-18 07:11:54.548525013 +0000
 Birth: 2026-11-18 07:11:54.548525013 +0000
```

`Blocks: 2` counts 512-byte units: the 3-byte file still uses one whole 1 KiB block of this small file
system. `Device: 7,0` identifies the file system (the loop device). An inode number is unique only *inside*
one file system, so a file is identified by the pair (device, inode).

## 4. Hard links, symbolic links and deleting

Create `src/bin/links.rs`:

```rust
use std::fs;
use std::os::unix::fs::{MetadataExt, symlink};

fn show(name: &str) -> std::io::Result<()> {
    let meta = fs::symlink_metadata(name)?;
    println!("{name:<10} inode {:>8}  links {}", meta.ino(), meta.nlink());
    Ok(())
}

fn main() -> std::io::Result<()> {
    fs::create_dir_all("demo")?;
    std::env::set_current_dir("demo")?;
    for old in ["original", "hardlink", "softlink"] {
        let _ = fs::remove_file(old); // start clean; ignore "not found"
    }

    fs::write("original", "hello\n")?;
    fs::hard_link("original", "hardlink")?; // a second name for the same inode
    symlink("original", "softlink")?; // a new small file that stores a path
    for name in ["original", "hardlink", "softlink"] {
        show(name)?;
    }
    println!("softlink points to {:?}", fs::read_link("softlink")?);

    fs::remove_file("original")?;
    println!("after removing 'original':");
    println!("  hardlink -> {:?}", fs::read_to_string("hardlink"));
    println!(
        "  softlink -> {:?}",
        fs::read_to_string("softlink").map_err(|e| e.kind())
    );
    Ok(())
}
```

Run it in `/mnt/small` (`~/p13-fs/target/debug/links`):

```text
original   inode       15  links 2
hardlink   inode       15  links 2
softlink   inode       16  links 1
softlink points to "original"
after removing 'original':
  hardlink -> Ok("hello\n")
  softlink -> Err(NotFound)
```

**What happens:**

1. `fs::hard_link` adds a second directory entry for inode 15. Both names show the same inode and the link
   count 2. Neither is more "real" than the other.
2. `symlink` creates a *new* inode (16) whose content is the path `original`. `fs::read_link` reads that
   path without following it.
3. `fs::remove_file` calls `unlink`: it removes the entry `original` and lowers the link count to 1. The
   data stays, because `hardlink` still points to the inode.
4. The symbolic link now points to a name that does not exist: it is *dangling*, and opening it gives
   `NotFound`.

| | hard link | symbolic link |
|---|---|---|
| what it is | another entry for the same inode | a file that stores a path |
| target removed | still works | dangling, `NotFound` |
| across file systems | no (inode numbers are per file system) | yes |
| to a directory | not allowed | allowed |

**Deleting a file that is open.** There is no "delete" system call, only `unlink`. The kernel frees the
inode and its blocks when **both** the link count is 0 **and** no process has the file open. Create
`src/bin/unlinked.rs`:

```rust
use std::fs::{self, File};
use std::io::{Read, Seek, SeekFrom, Write};
use std::os::fd::AsRawFd;

fn main() -> std::io::Result<()> {
    let mut file = File::options()
        .read(true)
        .write(true)
        .create(true)
        .truncate(true)
        .open("secret.tmp")?;
    file.write_all(b"still here\n")?;

    fs::remove_file("secret.tmp")?; // the name is gone ...
    println!("exists by name? {}", fs::exists("secret.tmp")?);
    let link = fs::read_link(format!("/proc/self/fd/{}", file.as_raw_fd()))?;
    println!("the descriptor points to {link:?}");

    file.seek(SeekFrom::Start(0))?; // ... but the inode lives while it is open
    let mut text = String::new();
    file.read_to_string(&mut text)?;
    print!("read through the open descriptor: {text}");
    Ok(())
}
```

```text
exists by name? false
the descriptor points to "/mnt/small/secret.tmp (deleted)"
read through the open descriptor: still here
```

`/proc/self/fd/3` is a symbolic link the kernel makes for every open descriptor (Lesson 13.4); it marks the
file as `(deleted)`. This explains a classic puzzle on servers: you delete a 10 GB log file, but `df` still
shows the disk as full. A running program still has it open. The space returns when that program closes
the file or exits.

> 🔒 **Security:** programs use this on purpose for temporary files: create, open, unlink at once. No other
> process can find the file by name, and the data disappears even if the program crashes.

## 5. Durability: when is data really saved?

`write` returns as soon as the data is in the **page cache** (Lesson 13.11). The kernel writes "dirty"
pages to the disk later, usually within about 30 seconds. If the power fails before that, the data is lost,
although `write` returned success.

`fsync` (in Rust `File::sync_all`) waits until the file's data and metadata are on the device. It is slow,
milliseconds instead of microseconds, so programs call it only where it matters.

There is a second problem. Imagine a program saves its settings like this:

```text
1. open settings.txt with truncate   -> file is now empty
2. write the new content             -> crash here = empty or half-written file
```

A crash between the steps destroys the old settings *and* the new ones. The standard fix is the
**atomic-replace pattern**, which relies on one guarantee: `rename` within one file system replaces the
target in one step. Anyone who opens the name sees either the old file or the new file, never a mix.
Put this into `src/lib.rs`:

```rust
use std::fs::{self, File};
use std::io::{self, Write};
use std::path::Path;

/// Replaces the file at `path` with `data` so that, even after a crash, the file holds
/// either the complete old content or the complete new content, never a mix.
pub fn atomic_write(path: &Path, data: &[u8]) -> io::Result<()> {
    let dir = path.parent().unwrap_or(Path::new("."));
    let dir = if dir.as_os_str().is_empty() {
        Path::new(".")
    } else {
        dir
    };
    let file_name = path
        .file_name()
        .ok_or_else(|| io::Error::new(io::ErrorKind::InvalidInput, "path has no file name"))?;
    let tmp = dir.join(format!(".{}.tmp", file_name.to_string_lossy()));

    let mut file = File::create(&tmp)?; // 1. write a temporary file in the same directory
    file.write_all(data)?;
    file.sync_all()?; // 2. make sure its data is on the disk
    drop(file);

    fs::rename(&tmp, path)?; // 3. atomically give it the real name
    File::open(dir)?.sync_all()?; // 4. make the new directory entry durable too
    Ok(())
}
```

**What happens:**

1. The temporary file lives in the **same directory**, so it is on the same file system; otherwise `rename`
   cannot work (see Common mistakes). For `settings.txt`, `parent()` is an empty path, which we turn into `.`.
2. `sync_all` makes the new content durable *before* it gets the real name.
3. `fs::rename` switches the name in one step. Until now, the old file was untouched.
4. The rename changed the *directory*, which is also data. Opening the directory and calling `sync_all`
   on it makes the new entry durable.

A small program that uses it, `src/bin/save.rs`:

```rust
use std::path::Path;

fn main() -> std::io::Result<()> {
    p13_fs::atomic_write(Path::new("settings.txt"), b"volume = 7\n")?;
    println!("saved");
    Ok(())
}
```

Watch the system calls in `/mnt/small`:

```bash
strace -e trace=openat,write,fsync,renameat ~/p13-fs/target/debug/save
```

The important lines (the loading of libraries at the start is cut):

```text
write(3, "volume = 7\n", 11)            = 11
fsync(3)                                = 0
renameat(AT_FDCWD, "./.settings.txt.tmp", AT_FDCWD, "settings.txt") = 0
openat(AT_FDCWD, ".", O_RDONLY|O_CLOEXEC) = 3
fsync(3)                                = 0
```

ext4 also protects its *own* structures with a **journal**: before changing inodes and directories, it
writes a note of the change to the journal. After a crash it replays or drops those notes, so the file
system stays consistent. The journal does not protect *your file's content*; that is the job of `fsync`
and the pattern above. Databases (Phase 17) build on exactly these tools.

> 🧪 **Testing:** you cannot easily test "a crash in the middle", but you can test the visible contract:
> the new content is there, no temporary file is left, and errors are reported. Exercise 3 does this.

## Common mistakes

1. **Renaming across file systems.** Writing the temporary file to `/tmp` and renaming it into another file
   system fails:

   ```rust
   use std::fs;

   fn main() -> std::io::Result<()> {
       fs::write("/tmp/draft.txt", "text\n")?;
       fs::rename("/tmp/draft.txt", "/mnt/small/draft.txt")?;
       Ok(())
   }
   ```

   ```text
   Error: Os { code: 18, kind: CrossesDevices, message: "Invalid cross-device link" }
   ```

   A rename only changes directory entries, and an inode cannot move to another file system. The `mv`
   command hides this by copying and deleting, which is **not** atomic. *Fix:* create the temporary file
   next to the target.

2. **"I called `write`, so it is saved."** Only `sync_all` (or `sync_data`) promises that. *Fix:* call it
   where durability matters, and remember the directory after a rename.

3. **Following a symlink by accident.** `fs::metadata` follows a symbolic link and describes the target, so
   a check like "is this a directory?" answers for the target, not for the link. *Fix:* use
   `fs::symlink_metadata` when you want to know about the link itself.

4. **Disk "full" with free space.** `No space left on device` can mean "no free inodes". *Fix:* check
   `df -i` as well as `df -h`.

## Try it yourself

1. **Run out of inodes.** In `/mnt/small`:

   ```bash
   mkdir many && cd many
   for i in $(seq 1 17000); do touch f$i || break; done
   df -h /mnt/small; df -i /mnt/small
   ```

   ```text
   touch: cannot touch 'f16365': No space left on device
   Filesystem      Size  Used Avail Use% Mounted on
   /dev/loop0       55M  420K   50M   1% /mnt/small
   Filesystem     Inodes IUsed IFree IUse% Mounted on
   /dev/loop0      16384 16384     0  100% /mnt/small
   ```

   The disk is 1% full, and yet no new file can be created. Clean up: `cd .. && rm -r many`.

2. **See a sparse file.** `truncate` wrote no data, so the image file has holes:

   ```bash
   cd ~
   ls -lh disk.img
   du -h disk.img
   ```

   ```text
   -rw-r--r-- 1 root root 64M Nov 18 07:14 disk.img
   8.5M	disk.img
   ```

   `ls` shows the apparent size; `du` shows the blocks really used.

3. **Clean up** only after you have finished the exercises below:

   ```bash
   cd ~
   sudo umount /mnt/small
   rm disk.img
   ```

## Exercises

Do these while `/mnt/small` is still mounted.

### Exercise 1 — Count the links of a directory (easy)

After "Try it yourself" step 1 (with `many` removed again), in `/mnt/small`, run `ls -lid /mnt/small /mnt/small/. /mnt/small/sub/..`. Why do all three lines show the
same inode, and why is the link count now 5 (it was 4 in Section 3)?

<details>
<summary>Hint</summary>

Which directories did you create after Section 3? Every directory has an entry `..` that points to its
parent.

</details>

<details>
<summary>Answer</summary>

```text
2 drwxr-xr-x 5 root root 1024 Nov 18 07:14 /mnt/small
2 drwxr-xr-x 5 root root 1024 Nov 18 07:14 /mnt/small/.
2 drwxr-xr-x 5 root root 1024 Nov 18 07:14 /mnt/small/sub/..
```

Why it works: the three paths are three directory entries that point to inode 2. The link count is 5:
`small` in `/mnt`, `.` in the directory itself, and `..` in each of its three subdirectories `lost+found`,
`sub` and `demo` (created by `links`). The `many` directory of "Try it yourself" added one more while it existed.

</details>

### Exercise 2 — Is it the same file? (medium)

Write `pub fn same_file(a: &Path, b: &Path) -> io::Result<bool>` in `src/lib.rs`. Two paths are the same
file when they have the same device **and** the same inode. Write a test that creates a file, a hard link,
a symbolic link and a copy in a fresh temporary directory, and checks that the first two are the same file
and the copy is not. What should a symbolic link give?

<details>
<summary>Hint</summary>

`MetadataExt` gives `dev()` and `ino()`. Use `fs::metadata` (it follows symbolic links). For the test
directory, use `std::env::temp_dir()` plus a unique name, and delete it at the end.

</details>

<details>
<summary>Answer</summary>

```rust
/// True when both paths lead to the same inode on the same file system.
pub fn same_file(a: &Path, b: &Path) -> io::Result<bool> {
    use std::os::unix::fs::MetadataExt;
    let (ma, mb) = (fs::metadata(a)?, fs::metadata(b)?);
    Ok(ma.dev() == mb.dev() && ma.ino() == mb.ino())
}

#[cfg(test)]
mod tests {
    use super::*;
    use std::os::unix::fs::symlink;
    use std::path::PathBuf;

    /// A fresh, empty directory for one test.
    fn test_dir(name: &str) -> PathBuf {
        let dir = std::env::temp_dir().join(format!("p13-fs-{name}-{}", std::process::id()));
        let _ = fs::remove_dir_all(&dir);
        fs::create_dir_all(&dir).unwrap();
        dir
    }

    #[test]
    fn hard_link_and_symlink_are_the_same_file_but_a_copy_is_not() -> io::Result<()> {
        let dir = test_dir("same");
        let a = dir.join("a");
        fs::write(&a, "data")?;
        fs::hard_link(&a, dir.join("hard"))?;
        symlink(&a, dir.join("soft"))?;
        fs::copy(&a, dir.join("copy"))?;

        assert!(same_file(&a, &dir.join("hard"))?);
        assert!(same_file(&a, &dir.join("soft"))?); // metadata() follows the symlink
        assert!(!same_file(&a, &dir.join("copy"))?);
        fs::remove_dir_all(&dir)
    }
}
```

Why it works: a hard link is the same inode by definition. `fs::metadata` follows the symbolic link to
its target, so it also reports the same inode. A copy has its own inode with equal content. Comparing
only inode numbers would be wrong: two different file systems can both have an inode 13.

</details>

### Exercise 3 — Test the atomic write (hard)

Write tests for `atomic_write`: (a) after replacing an existing file it holds the new content, and the
directory contains no other file (no leftover `.tmp`); (b) writing into a directory that does not exist
returns an `Err`. Then think: which part of the guarantee can your tests *not* check?

<details>
<summary>Hint</summary>

Reuse `test_dir` from Exercise 2. Collect the names with `fs::read_dir(&dir)?.map(|e| e.unwrap().file_name())`.

</details>

<details>
<summary>Answer</summary>

Add to the `tests` module:

```rust
#[test]
fn replaces_content_and_leaves_no_temp_file() -> io::Result<()> {
    let dir = test_dir("atomic");
    let path = dir.join("config.toml");
    fs::write(&path, "old = true\n")?;

    atomic_write(&path, b"new = true\n")?;

    assert_eq!(fs::read_to_string(&path)?, "new = true\n");
    let names: Vec<_> = fs::read_dir(&dir)?
        .map(|e| e.unwrap().file_name())
        .collect();
    assert_eq!(names, vec!["config.toml"]);
    fs::remove_dir_all(&dir)
}

#[test]
fn missing_directory_is_an_error() {
    let path = Path::new("/no/such/dir/file.txt");
    assert!(atomic_write(path, b"x").is_err());
}
```

`cargo test` (with the test of Exercise 2):

```text
test result: ok. 3 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

Why it works: the tests check the visible contract. What they cannot check is the crash guarantee itself:
that a power loss in the middle leaves the old or the new file. That would need a machine you can switch
off, or special tools that simulate lost writes. This is why the order of the four steps must be correct by
reasoning, not only by tests.

</details>

## Quiz

1. What does a directory contain?

<details>
<summary>Answer</summary>

A list of entries: names and the inode numbers they point to. The names live in the directory; the inode
itself has no name.

</details>

2. A file has two hard links. You delete one name. What happens to the data?

<details>
<summary>Answer</summary>

Nothing: the link count drops from 2 to 1, and the data stays available through the other name. It is
freed only when the count is 0 and no process has the file open.

</details>

3. You deleted a huge log file, but `df -h` shows no more free space. What is the likely reason?

<details>
<summary>Answer</summary>

A running process still has the file open. The inode and its blocks are freed only when the last
descriptor is closed. `ls -l /proc/<pid>/fd` shows it as `(deleted)`.

</details>

4. Why must the temporary file in the atomic-replace pattern be in the same directory as the target?

<details>
<summary>Answer</summary>

`rename` is atomic only within one file system; across file systems it fails with "Invalid cross-device
link". The same directory is guaranteed to be on the same file system.

</details>

5. What is the difference between `write` returning `Ok` and `sync_all` returning `Ok`?

<details>
<summary>Answer</summary>

After `write` the data is in the page cache, in RAM; a power loss can still lose it. After `sync_all` the
data and metadata are on the storage device.

</details>

## Summary

- The VFS gives every file system (ext4, proc, tmpfs, overlayfs, cgroup2…) the same system calls; each is mounted on a directory.
- A file is an inode (type, size, owner, permissions, times, link count, data blocks) plus one or more directory entries that name it; a file is identified by (device, inode).
- Hard links are extra names for one inode; symbolic links are small files that store a path and can dangle.
- "Delete" is `unlink`: the data is freed when no names and no open descriptors remain.
- `write` only reaches the page cache; `sync_all` makes data durable. Replace files safely with temp file → `sync_all` → `rename` → sync the directory.
- A file system can be full of inodes before it is full of data: check `df -i`.

## Sources and further reading

- [inode(7) — Linux manual page](https://man7.org/linux/man-pages/man7/inode.7.html) — every field of an inode and how `stat` reports it.
- [rename(2) — Linux manual page](https://man7.org/linux/man-pages/man2/rename.2.html) — the atomic-replace guarantee and the `EXDEV` error.
- [fsync(2) — Linux manual page](https://man7.org/linux/man-pages/man2/fsync.2.html) — what `fsync` promises, including the note about syncing the directory.
- [std::os::unix::fs::MetadataExt](https://doc.rust-lang.org/std/os/unix/fs/trait.MetadataExt.html) — inode number, link count, mode and more from Rust.
- [The Linux kernel documentation — Overview of the Linux Virtual File System](https://docs.kernel.org/filesystems/vfs.html) — how the VFS is built (advanced).
- [Operating Systems: Three Easy Pieces](https://pages.cs.wisc.edu/~remzi/OSTEP/) — the chapters "Files and Directories", "File System Implementation" and "Crash Consistency: FSCK and Journaling".

---

[🏠 Start](../START-HERE.md) · [← 13.11 Memory-mapped files](11-memory-mapped-files.md) · [📋 Phase 13](README.md) · [13.13 Non-blocking I/O and epoll →](13-non-blocking-io-and-epoll.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
