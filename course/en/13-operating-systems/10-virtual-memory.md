<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 13.9 Threads and scheduling](09-threads-and-scheduling.md) · [📋 Phase 13](README.md) · [13.11 Memory-mapped files →](11-memory-mapped-files.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/13-operating-systems/10-virtual-memory.md) <!-- nav -->

# Lesson 13.10 — Virtual memory

> **Phase 13 — Operating systems from the inside** · Lesson 10 of 18 · ⏱ about 2.5 hours (2 sittings)

## What you will learn

- Why every process gets its own private *address space*, and how the kernel and the CPU (Central Processing Unit) give each process the illusion of owning the whole memory.
- What pages and page tables are, and how a virtual address is split into a page number and an offset.
- How to read the memory map of a process in `/proc/self/maps`.
- Why allocating memory costs almost nothing until you touch it (demand paging and page faults), and how `fork` shares memory with copy-on-write.
- What a segmentation fault really is.

## Before you start

- You finished [Lesson 13.9 — Threads and scheduling](09-threads-and-scheduling.md).
- You know processes, `fork` and `waitpid` ([Lesson 13.5](05-fork-exec-and-wait.md)) and what a system call is ([Lesson 13.3](03-system-calls.md)).
- You remember the stack and the heap ([Lesson 03.1](../03-ownership-and-borrowing/01-memory-stack-and-heap.md)), raw pointers and `unsafe` ([Lesson 06.6](../06-rust-advanced/06-unsafe-rust-and-miri.md)), caches ([Lesson 12.9](../12-computer-systems/09-memory-hierarchy-and-caches.md)) and how a program is loaded ([Lesson 12.12](../12-computer-systems/12-linking-and-loading.md)).
- You work as in [Lesson 13.1](01-what-an-operating-system-does.md): write code on the Mac, copy it into the VM `dev` with `rsync`, build and run it there. This lesson has its own package, `code/p13-vmem`. Create it on the Mac:

```bash
cd ~/projects/rust
cargo new code/p13-vmem
cd code/p13-vmem
cargo add nix@0.32 --features feature,process,resource
rm src/main.rs && mkdir src/bin
```

Each example is one file in `src/bin/`. In the VM (replace `student` with your Mac user name):

```bash
rsync -a --delete --exclude target /Users/student/projects/rust/code/p13-vmem/ ~/p13-vmem/
cd ~/p13-vmem
cargo run --bin pages
```

## New words

| Word | Meaning |
|---|---|
| **physical memory** | The real RAM (Random Access Memory) chips. Each byte has one *physical address*. |
| **virtual address** | The address a program uses, for example in a pointer. The CPU translates it to a physical address on every access. |
| **address space** | The full range of virtual addresses one process can use. Every process has its own. |
| **page** | A fixed-size block of virtual memory, usually 4 KiB (4,096 bytes). Memory is mapped page by page. |
| **frame** | A page-sized block of physical memory. A page is stored in a frame. |
| **page table** | A table, kept by the kernel for each process, that says which frame holds each page and what the process may do with it (read, write, execute). |
| **MMU** (Memory Management Unit) | The part of the CPU that translates virtual addresses to physical ones by reading the page table. |
| **TLB** (Translation Lookaside Buffer) | A small cache inside the CPU that remembers recent translations, so the MMU does not read the page table every time. |
| **page fault** | The CPU stops the program because a page is not mapped, or not allowed. The kernel then fixes it (maps a frame) or kills the program. |
| **minor / major fault** | A minor fault is fixed without the disk (for example, give a fresh zero page). A major fault must read data from the disk first. |
| **demand paging** | The kernel gives a frame only when the page is touched for the first time, not when memory is requested. |
| **RSS** (Resident Set Size) | How much of a process's memory is really in RAM right now. |
| **copy-on-write** (CoW) | After `fork`, parent and child share the same frames. A frame is copied only when one of them writes to it. |
| **segmentation fault** (SIGSEGV) | The signal the kernel sends when a program touches an address it may not use. |
| **ASLR** (Address Space Layout Randomization) | The kernel puts the stack, heap and code at random addresses on each run, which makes attacks harder. |
| **OOM killer** (Out Of Memory killer) | The part of the kernel that kills a process when RAM and swap are full. |

## 1. Why virtual memory exists

Imagine a hotel. Each guest gets a key card that says "room 12". In one hotel, room 12 of guest A and
room 12 of guest B are different real rooms: the front desk keeps a private list for each guest that says
"your room 12 is really room 304 on the third floor". Guests never see the real room numbers. They cannot
walk into somebody else's room, because their list does not contain it.

Virtual memory works the same way:

- Every **process** gets its own **address space**: a huge range of addresses that look like they all
  belong to it.
- The addresses inside your program (every pointer, every `&x`) are **virtual addresses**.
- The kernel keeps a private **page table** for each process. On every memory access, the **MMU** in the
  CPU uses that table to translate the virtual address into a **physical address** in RAM.

This gives three big benefits:

1. **Isolation.** A process cannot read or damage the memory of another process: its table simply has no
   entry for those frames. A bug in one program does not crash the others.
2. **A simple picture for every program.** Every program can be linked and loaded as if it were alone in
   memory ([Lesson 12.12](../12-computer-systems/12-linking-and-loading.md)).
3. **Flexibility.** The kernel can give RAM later, share it between processes, or move it to the disk,
   and the program never notices.

## 2. Pages and page tables

Mapping each byte separately would need an enormous table. So memory is mapped in blocks of one size,
called **pages** (virtual) and **frames** (physical). On Linux on most CPUs a page is **4 KiB**. A virtual
address is then split into two parts:

```text
virtual address 0x0000_aaaa_db09_bd60   (page size 4096 = 0x1000)

  page number = address / 4096 = 0xaaaadb09b   -> looked up in the page table
  offset      = address % 4096 = 0xd60 (3424)  -> copied unchanged

  page table of this process
  +-------------+--------------+-------------+
  | page number | frame number | permissions |
  +-------------+--------------+-------------+
  | 0xaaaadb09b | 0x3f2a1      | rw-         |
  | ...         | ...          | ...         |
  +-------------+--------------+-------------+

physical address = frame 0x3f2a1 * 4096 + 0xd60
```

The offset is the same in both addresses. Only the page number is replaced by a frame number. Real page
tables are trees of several levels (usually four on arm64 and x86-64), so that empty parts of the address space
need no table at all, but the idea is exactly this lookup.

Reading a page table from memory on every access would be slow. The CPU keeps recent translations in the
**TLB**, a tiny cache just like the data caches of [Lesson 12.9](../12-computer-systems/09-memory-hierarchy-and-caches.md).
Most accesses hit the TLB and cost nothing extra.

Let us look at real addresses. Create `src/bin/pages.rs`:

```rust
use nix::unistd::{SysconfVar, sysconf};

fn main() {
    let page = sysconf(SysconfVar::PAGE_SIZE)
        .expect("sysconf failed")
        .expect("page size is unknown") as usize;
    println!("page size: {page} bytes");

    let on_stack: u64 = 42;
    let on_heap = Box::new(7u64);
    let places = [
        ("stack", &on_stack as *const u64 as usize),
        ("heap", &*on_heap as *const u64 as usize),
        ("code", main as fn() as usize),
    ];
    for (name, addr) in places {
        println!(
            "{name:>5}: address {addr:#014x} -> page {:#011x}, offset {:4}",
            addr / page,
            addr % page
        );
    }
}
```

**What happens:**

1. `sysconf(SysconfVar::PAGE_SIZE)` asks the system for the page size. It returns `Result<Option<_>>`:
   the call can fail, and some values can be "unknown". Never write `4096` by hand.
2. `&on_stack as *const u64 as usize` turns a reference into a raw pointer and then into a plain number,
   the virtual address. This is safe: we only look at the number, we never use it as a pointer.
3. `main as fn() as usize` is the address of the machine code of `main` itself.
4. For each address we print the page number (`/`) and the offset inside the page (`%`).

Run it twice with `cargo run --bin pages`:

```text
page size: 4096 bytes
stack: address 0xffffc5b538d0 -> page 0xffffc5b53, offset 2256
 heap: address 0xaaaadb09bd60 -> page 0xaaaadb09b, offset 3424
 code: address 0xaaaab6319ec8 -> page 0xaaaab6319, offset 3784
page size: 4096 bytes
stack: address 0xffffc6a6e620 -> page 0xffffc6a6e, offset 1568
 heap: address 0xaaaacc550d60 -> page 0xaaaacc550, offset 3424
 code: address 0xaaaac2ef9ec8 -> page 0xaaaac2ef9, offset 3784
```

The addresses change on every run. This is **ASLR**: the kernel chooses random places for the stack, the
heap and the program, so that an attacker cannot guess where things are. Notice that the *offsets* of the
heap value and of `main` stay the same: only whole pages move.

> 🔒 **Security:** ASLR is one of the defences that make memory bugs harder to exploit. It does not fix
> the bug; safe Rust prevents most such bugs in the first place ([Lesson 10.12](../10-security-fundamentals/12-memory-safety-and-reviewing-unsafe.md)).

## 3. The map of a process: `/proc/self/maps`

The address space is mostly empty. Only some ranges, called **mappings** (or areas), are in use. The
kernel shows them in the file `/proc/<pid>/maps`; `/proc/self` always means "the process that reads it".
Create `src/bin/maps.rs`:

```rust
use std::fs;

fn main() {
    let maps = fs::read_to_string("/proc/self/maps").expect("cannot read /proc/self/maps");
    for line in maps.lines() {
        println!("{line}");
    }
}
```

`cargo run --bin maps` prints about 25 lines. Here is the output with the middle part shortened
(the `...` line is ours):

```text
aaaad19e0000-aaaad1a2f000 r-xp 00000000 00:34 316190                     /work/target/debug/maps
aaaad1a3d000-aaaad1a40000 r--p 0005d000 00:34 316190                     /work/target/debug/maps
aaaad1a40000-aaaad1a41000 rw-p 00060000 00:34 316190                     /work/target/debug/maps
aaab075f9000-aaab0761a000 rw-p 00000000 00:00 0                          [heap]
ffff9e240000-ffff9e3dc000 r-xp 00000000 00:57 951780                     /usr/lib/aarch64-linux-gnu/libc.so.6
ffff9e3dc000-ffff9e3ed000 ---p 0019c000 00:57 951780                     /usr/lib/aarch64-linux-gnu/libc.so.6
...
ffff9e47b000-ffff9e47d000 r-xp 00000000 00:00 0                          [vdso]
ffffc2483000-ffffc24a4000 rw-p 00000000 00:00 0                          [stack]
```

(The path of your program is under your project folder, not `/work`.) Each line is one mapping:

| Column | Example | Meaning |
|---|---|---|
| address range | `aaaad19e0000-aaaad1a2f000` | start and end virtual address (end not included), in hexadecimal |
| permissions | `r-xp` | **r**ead, **w**rite, e**x**ecute; `p` = private (copy-on-write), `s` = shared |
| offset | `0005d000` | where in the file this mapping starts |
| device, inode | `00:34 316190` | which file (Lesson 13.12 explains inodes); `0` for memory with no file |
| path | `[heap]` | the file, or a name in brackets for special areas |

**What happens** when you read the map from top to bottom:

1. Three mappings of the program file: the machine code (`r-x`, can run but not be changed), constant
   data (`r--`) and global variables (`rw-`). These are the ELF segments from Lesson 12.12.
2. `[heap]`: the classic heap that grows upwards. Small `Box` and `Vec` values live here.
3. The C library `libc.so.6` (and, in the hidden lines, the dynamic loader `ld-linux-aarch64.so.1`), mapped
   the same way. The `---p` line is a gap with no permissions at all: touching it crashes the program.
4. `[vdso]` is a tiny library that the kernel itself puts into every process, so that some calls such as
   "what time is it?" do not need a real system call.
5. `[stack]`: the stack of the main thread, near the top of the address space.

> 💡 **Tip:** you can read the map of any of your own processes: `cat /proc/$$/maps` shows the map of your
> current shell (`$$` is the PID, Process Identifier, of the shell). `pmap <pid>` prints a shorter version.

## 4. Demand paging: memory is given when it is touched

When a program asks for memory, the kernel does **not** find free RAM at once. It only notes "this range
of addresses is now allowed" in the process's mapping list. The page table stays empty. Then:

1. The program touches an address in the new range for the first time.
2. The MMU finds no frame for that page and raises a **page fault**: the CPU pauses the program and jumps
   into the kernel.
3. The kernel sees that the address is allowed, takes a free frame, fills it with zeros, writes it into
   the page table, and lets the program continue at the same instruction.

The program does not notice anything, except a short delay. This is **demand paging**. A fault fixed
from RAM like this is a **minor fault**. If the data had to come from the disk first (for example a page
of a program file that was not in memory, Lesson 13.11), it would be a **major fault**.

Let us watch it. Create `src/bin/lazy.rs`:

```rust
use std::fs;
use std::hint::black_box;

use nix::sys::resource::{UsageWho, getrusage};
use nix::unistd::{SysconfVar, sysconf};

/// Resident set size of this process in KiB, read from /proc/self/status.
fn rss_kib() -> u64 {
    let status = fs::read_to_string("/proc/self/status").expect("cannot read status");
    let line = status
        .lines()
        .find(|l| l.starts_with("VmRSS:"))
        .expect("no VmRSS line");
    line.split_whitespace().nth(1).unwrap().parse().unwrap()
}

fn report(when: &str) {
    let faults = getrusage(UsageWho::RUSAGE_SELF)
        .expect("getrusage failed")
        .minor_page_faults();
    println!(
        "{when:<28} RSS = {:>7} KiB, minor faults = {faults:>6}",
        rss_kib()
    );
}

fn main() {
    const SIZE: usize = 256 * 1024 * 1024; // 256 MiB
    let page = sysconf(SysconfVar::PAGE_SIZE).unwrap().unwrap() as usize;

    report("start");
    let mut buffer = vec![0u8; SIZE];
    report("after vec![0u8; 256 MiB]");

    for i in (0..SIZE).step_by(page) {
        buffer[i] = 1; // one write per page
    }
    report("after one write per page");

    black_box(&buffer);
}
```

**What happens:**

1. `rss_kib` reads `/proc/self/status`, a text file the kernel writes for every process, and takes the
   number from the line `VmRSS:  1660 kB`. RSS is the memory really in RAM.
2. `getrusage(UsageWho::RUSAGE_SELF)` asks the kernel for resource usage of this process;
   `minor_page_faults()` is the number of minor faults since the start.
3. `vec![0u8; SIZE]` asks for 256 MiB of zeros. For such a big block the allocator asks the kernel
   directly for a new mapping, and the kernel promises that new pages are full of zeros, so nothing has to
   be written yet.
4. The loop writes one byte in every page: `step_by(page)` jumps 4,096 bytes at a time.
5. `black_box(&buffer)` tells the compiler "assume this value is used", so the optimizer cannot delete
   the loop.

Run it with `cargo run --bin lazy`:

```text
start                        RSS =    1660 KiB, minor faults =    102
after vec![0u8; 256 MiB]     RSS =    1668 KiB, minor faults =    105
after one write per page     RSS =  263808 KiB, minor faults =  65640
```

The allocation of 256 MiB changed RSS by only 8 KiB. The loop then caused about 65,536 page faults:
256 MiB / 4 KiB = 65,536 pages, one fault each, and RSS grew by 256 MiB. Memory became real only when
it was touched.

> ⚡ **Performance:** a page fault costs roughly a microsecond. That is nothing once, but a program that
> touches millions of new pages spends real time in the kernel. Reusing a buffer is cheaper than
> allocating a new one each time.

Your numbers will differ a little. On some systems the kernel uses **transparent huge pages**: when a
large aligned range is touched, it maps one 2 MiB page at once instead of 512 small ones, and you see
fewer faults. You can check the setting with `cat /sys/kernel/mm/transparent_hugepage/enabled`.

You can see the allocation itself with `strace` ([Lesson 13.3](03-system-calls.md)), from the folder
`~/p13-vmem` in the VM:

```bash
strace -e trace=mmap,munmap ./target/debug/lazy
```

The last lines:

```text
start                        RSS =    1664 KiB, minor faults =     84
mmap(NULL, 268439552, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0xffffacbcf000
after vec![0u8; 256 MiB]     RSS =    1672 KiB, minor faults =     87
after one write per page     RSS =  263812 KiB, minor faults =  64089
munmap(0xffffacbcf000, 268439552)       = 0
+++ exited with 0 +++
```

`mmap` created the new mapping of 256 MiB plus one page for the allocator's bookkeeping, and `munmap`
removed it when `buffer` was dropped. `mmap` gets a whole lesson next: [Lesson 13.11](11-memory-mapped-files.md).

## 5. `fork` and copy-on-write

In [Lesson 13.5](05-fork-exec-and-wait.md) you learned that `fork` makes a copy of the process. Copying
all of its memory would be slow. Instead the kernel uses **copy-on-write**:

```text
right after fork                   after the child writes to page B
parent table    child table        parent table    child table
  A -> frame 7    A -> frame 7       A -> frame 7    A -> frame 7
  B -> frame 8    B -> frame 8       B -> frame 8    B -> frame 21 (new copy)
  (both marked read-only)
```

1. Both page tables point to the same frames, and all writable pages are marked read-only.
2. When one process writes, the MMU raises a page fault. The kernel copies just that one frame, gives
   the writer its private copy with write permission, and lets it continue.

The child has the **same virtual addresses** as the parent but, after a write, different data. Create
`src/bin/cow.rs`:

```rust
use nix::sys::wait::waitpid;
use nix::unistd::{ForkResult, fork};

fn main() {
    let mut value: i32 = 1;

    // SAFETY: the program has only one thread, so the child is a full copy.
    match unsafe { fork() }.expect("fork failed") {
        ForkResult::Child => {
            value = 100;
            println!("child : value = {value:3} at {:p}", &value);
            std::process::exit(0);
        }
        ForkResult::Parent { child } => {
            waitpid(child, None).expect("waitpid failed");
            println!("parent: value = {value:3} at {:p}", &value);
        }
    }
}
```

**What happens:**

1. `fork` is `unsafe` in `nix` because forking a program with several threads can leave locks in a broken
   state in the child (Lesson 13.5). Our program has one thread.
2. The child writes `100`. This write causes a copy-on-write fault, and the child gets its own copy of the
   page with `value`.
3. The parent waits for the child, then prints its own `value`.
4. `{:p}` prints the address of `value`.

```text
child : value = 100 at 0xfffff99f2e64
parent: value =   1 at 0xfffff99f2e64
```

The same address shows two different values. That is only possible because the two processes have two
different address spaces, and the same virtual address is translated to two different frames.

## 6. When memory access goes wrong

If the kernel looks at a page fault and finds that the address is **not allowed** (not in any mapping, or
a write to a read-only page), it cannot fix it. It sends the signal **SIGSEGV** (segmentation fault) to the
process, and by default the process dies. Create `src/bin/crash.rs`:

```rust
fn main() {
    let address = 0x10 as *const u8;
    println!("reading address {address:p} ...");
    // SAFETY: none! Nothing is mapped at this address. This is a deliberate crash.
    let byte = unsafe { std::ptr::read_volatile(address) };
    println!("never printed: {byte}");
}
```

`read_volatile` reads exactly from that address (the compiler may not remove or change the read). Run it,
then print the exit status of the last command:

```bash
cargo run --bin crash
echo "exit=$?"
```

```text
reading address 0x10 ...
Segmentation fault
exit=139
```

The shell reports the signal (some systems add `(core dumped)`). The exit status 139 is 128 + 11, and 11 is
the number of SIGSEGV ([Lesson 11.5](../11-linux-shell-and-containers/05-processes-and-signals.md)). The
first page of the address space is never mapped on purpose, so that using a null or nearly-null pointer
crashes loudly instead of reading garbage.

Safe Rust cannot produce such a crash: every reference points to valid memory. You only get here with
`unsafe` code or a bug in C code called through FFI ([Lesson 06.10](../06-rust-advanced/10-ffi-calling-c.md)).

**Running out of memory.** Because of demand paging, Linux usually lets programs request more memory than
exists (this is called *overcommit*). If they really touch it all, the kernel first moves rarely used
pages to *swap* space on the disk. When there is nothing left, the **OOM killer** chooses a process, most
often the biggest one, and kills it with SIGKILL. You will put limits on memory and watch this happen in a
safe way in [Lesson 13.16](16-cgroups.md).

## Common mistakes

1. **Reading the wrong column as "memory used".** `ps -o pid,vsz,rss,comm` shows VSZ (virtual size,
   all mappings) and RSS. A program with VSZ 2 GB may use only 5 MB of RAM. *Fix:* look at RSS (and, for
   containers, the cgroup numbers of Lesson 13.16).

2. **Hard-coding the page size.** Writing `4096` works on most Linux PCs (personal computers), but macOS on Apple silicon uses
   16 KiB pages and some arm64 Linux systems use 16 KiB or 64 KiB. *Fix:* ask with
   `sysconf(SysconfVar::PAGE_SIZE)`.

3. **Expecting the same address on every run.** Tests or logs that compare raw addresses fail because of
   ASLR. *Fix:* compare offsets or values, never absolute addresses.

4. **Mistaking a stack overflow for a different bug.** The stack has a *guard page* below it: a mapping
   with no permissions. Very deep recursion runs into it. This program uses 8 KiB of stack per call:

   ```rust
   fn depth(n: u64) -> u64 {
       let big = [n; 1024]; // 8 KiB on the stack in every call
       if n == 0 { 0 } else { big[0] + depth(n - 1) }
   }

   fn main() {
       println!("{}", depth(1_000_000));
   }
   ```

   ```text
   thread 'main' (551) has overflowed its stack
   fatal runtime error: stack overflow, aborting
   Aborted
   ```

   Rust notices that the fault address is in the guard page and prints a clear message before aborting
   (exit status 134 = 128 + 6, SIGABRT). *Fix:* use a loop or put large data on the heap (`vec!`, `Box`).

## Try it yourself

All commands run in the VM.

1. Show the map of your shell and count its lines:

   ```bash
   cat /proc/$$/maps | head -5
   wc -l /proc/$$/maps
   ```

2. Compare virtual and resident size of a few processes:

   ```bash
   ps -o pid,vsz,rss,comm -p $$,1
   ```

   VSZ is always much bigger than RSS.

3. Look at the memory of the whole machine:

   ```bash
   head -5 /proc/meminfo
   ```

   `MemTotal` is the RAM of the VM; `MemAvailable` is what new programs could still get.

## Exercises

Put the helper functions of exercises 1 and 2 into `src/lib.rs` of `code/p13-vmem` (create the file). Binaries in
`src/bin/` can use them as `p13_vmem::<name>`.

### Exercise 1 — Page and offset (easy)

Write `pub fn page_and_offset(addr: usize, page_size: usize) -> (usize, usize)`. It must panic with a
message containing "power of two" if `page_size` is not a power of two. Write tests: `0x1234` with 4,096
gives `(1, 0x234)`; `4096` gives `(1, 0)`; `0x1234` with 16,384 gives `(0, 0x1234)`; a page size of 1,000
panics.

<details>
<summary>Hint</summary>

`usize` has the method `is_power_of_two()`. For a power of two, `addr % page_size` equals
`addr & (page_size - 1)` ([Lesson 12.4](../12-computer-systems/04-bitwise-operations.md)). Use
`#[should_panic(expected = "power of two")]` for the last test.

</details>

<details>
<summary>Answer</summary>

```rust
/// Splits an address into (page number, offset inside the page).
/// `page_size` must be a power of two.
pub fn page_and_offset(addr: usize, page_size: usize) -> (usize, usize) {
    assert!(
        page_size.is_power_of_two(),
        "page size must be a power of two"
    );
    (addr / page_size, addr & (page_size - 1))
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn splits_address_into_page_and_offset() {
        assert_eq!(page_and_offset(0x1234, 4096), (1, 0x234));
        assert_eq!(page_and_offset(4096, 4096), (1, 0));
        assert_eq!(page_and_offset(4095, 4096), (0, 4095));
        assert_eq!(page_and_offset(0x1234, 16384), (0, 0x1234));
    }

    #[test]
    #[should_panic(expected = "power of two")]
    fn rejects_bad_page_size() {
        page_and_offset(10, 1000);
    }
}
```

Why it works: a power of two minus one is a mask of ones (`0xfff` for 4,096), so `&` keeps exactly the low
bits, the offset. The division drops those bits and leaves the page number.

</details>

### Exercise 2 — Parse RSS, with a test (medium)

`rss_kib` in `lazy.rs` reads a real file, so it is hard to test. Write
`pub fn parse_rss_kib(status: &str) -> Option<u64>` that takes the *text* of a status file. Test it with a
short sample text that contains `VmRSS:\t    1684 kB`, and with a text that has no `VmRSS` line (kernel
threads have none).

<details>
<summary>Hint</summary>

Use `lines().find(...)`, then `split_whitespace().nth(1)`, then `parse().ok()`. The `?` operator works on
`Option` too.

</details>

<details>
<summary>Answer</summary>

```rust
/// Finds the `VmRSS:` line in the text of /proc/<pid>/status and returns the value in KiB.
pub fn parse_rss_kib(status: &str) -> Option<u64> {
    let line = status.lines().find(|l| l.starts_with("VmRSS:"))?;
    line.split_whitespace().nth(1)?.parse().ok()
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn finds_rss_line() {
        let text = "Name:\tlazy\nVmSize:\t  264000 kB\nVmRSS:\t    1684 kB\nThreads:\t1\n";
        assert_eq!(parse_rss_kib(text), Some(1684));
    }

    #[test]
    fn missing_rss_line_gives_none() {
        assert_eq!(parse_rss_kib("Name:\tkthreadd\n"), None);
    }
}
```

Why it works: separating "read the file" from "understand the text" makes the logic a pure function, which
is easy to test with any input. The real program just calls
`parse_rss_kib(&fs::read_to_string("/proc/self/status")?)`.

</details>

### Exercise 3 — Measure copy-on-write (hard)

Allocate 64 MiB with `vec![1u8; SIZE]` (this writes every byte, so all pages are in RAM). Then `fork`. In
the child, write one byte per page and print how many minor faults this caused. In the parent, after
`waitpid`, print `buffer[0]`. Predict the numbers before you run it.

<details>
<summary>Hint</summary>

Read `getrusage(UsageWho::RUSAGE_SELF)` in the child before and after the loop. Each process has its own
counters.

</details>

<details>
<summary>Answer</summary>

```rust
use std::hint::black_box;

use nix::sys::resource::{UsageWho, getrusage};
use nix::sys::wait::waitpid;
use nix::unistd::{ForkResult, SysconfVar, fork, sysconf};

fn minor_faults() -> i64 {
    getrusage(UsageWho::RUSAGE_SELF)
        .unwrap()
        .minor_page_faults()
}

fn main() {
    const SIZE: usize = 64 * 1024 * 1024;
    let page = sysconf(SysconfVar::PAGE_SIZE).unwrap().unwrap() as usize;
    let mut buffer = vec![1u8; SIZE]; // vec![1; …] writes every byte: all pages are in RAM

    // SAFETY: single-threaded program.
    match unsafe { fork() }.expect("fork failed") {
        ForkResult::Child => {
            let before = minor_faults();
            for i in (0..SIZE).step_by(page) {
                buffer[i] = 2; // the first write to each shared page copies it
            }
            let after = minor_faults();
            black_box(&buffer);
            println!("pages in the buffer : {}", SIZE / page);
            println!("faults while writing: {}", after - before);
            std::process::exit(0);
        }
        ForkResult::Parent { child } => {
            waitpid(child, None).unwrap();
            println!("parent still sees   : {}", buffer[0]);
        }
    }
}
```

```text
pages in the buffer : 16384
faults while writing: 16385
parent still sees   : 1
```

Why it works: after `fork` all 16,384 pages are shared and read-only. The first write to each one is a
copy-on-write fault (one extra fault comes from the program itself, for example the stack). The parent
keeps its original frames, so it still sees `1`.

</details>

## Quiz

1. What does the MMU do?

<details>
<summary>Answer</summary>

On every memory access it translates the virtual address used by the program into a physical address,
using the page table of the current process (helped by the TLB cache). If there is no valid entry, it
raises a page fault.

</details>

2. A program calls `vec![0u8; 1 << 30]` (1 GiB) and then sleeps. Why does RSS stay small?

<details>
<summary>Answer</summary>

Because of demand paging: the kernel only created a mapping. Frames are given on the first touch of each
page, and the program has not touched them. Zero pages need no writing in advance.

</details>

3. What is the difference between a minor and a major page fault?

<details>
<summary>Answer</summary>

A minor fault is fixed without the disk (a fresh zero page, a copy-on-write copy, a page already in RAM).
A major fault must first read the data from the disk, so it is much slower.

</details>

4. After `fork`, the parent and the child print the address of the same variable. Why are the addresses
   equal while the values can differ?

<details>
<summary>Answer</summary>

The child's address space is a copy of the parent's, so the virtual addresses are the same. But each
process has its own page table; after a write, copy-on-write gives the writer its own frame, so the same
virtual address leads to different physical memory.

</details>

5. A program dies with exit status 139. What happened?

<details>
<summary>Answer</summary>

It was killed by signal 11 (139 = 128 + 11), SIGSEGV: it touched an address that is not mapped, or wrote
to a page it may only read.

</details>

## Summary

- Every process has its own virtual address space; the MMU translates virtual to physical addresses with the process's page table, and the TLB caches the translations.
- Memory is managed in pages (usually 4 KiB on Linux); an address is a page number plus an offset. Ask for the page size with `sysconf`.
- `/proc/self/maps` lists the mappings of a process: program, heap, libraries, vdso, stack, each with permissions.
- Demand paging gives a frame only on the first touch; that touch is a (minor) page fault. RSS shows what is really in RAM.
- `fork` shares all frames with copy-on-write; a page is copied only when someone writes to it.
- A fault on an address that is not allowed becomes SIGSEGV; deep recursion hits the stack guard page. When RAM runs out, the OOM killer ends a process.

## Sources and further reading

- [proc_pid_maps(5) — Linux manual page](https://man7.org/linux/man-pages/man5/proc_pid_maps.5.html) — every column of `/proc/<pid>/maps`.
- [proc_pid_status(5) — Linux manual page](https://man7.org/linux/man-pages/man5/proc_pid_status.5.html) — the fields of `/proc/<pid>/status`, including `VmRSS` and `VmSize`.
- [The Linux kernel documentation — Memory Management](https://docs.kernel.org/admin-guide/mm/index.html) — overview for administrators, including transparent huge pages and overcommit.
- [Operating Systems: Three Easy Pieces](https://pages.cs.wisc.edu/~remzi/OSTEP/) — free book; the chapters "Address Spaces", "Paging: Introduction" and "Paging: Faster Translations (TLBs)" explain this lesson in depth.
- [nix::sys::resource on docs.rs](https://docs.rs/nix/0.32.0/nix/sys/resource/index.html) — `getrusage` and the `Usage` type.
- [getrusage(2) — Linux manual page](https://man7.org/linux/man-pages/man2/getrusage.2.html) — what the counters mean.

---

[🏠 Start](../START-HERE.md) · [← 13.9 Threads and scheduling](09-threads-and-scheduling.md) · [📋 Phase 13](README.md) · [13.11 Memory-mapped files →](11-memory-mapped-files.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
