<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 22.7 eBPF track: a first program with Aya](07-ebpf-track-a-first-program-with-aya.md) · [📋 Phase 22](README.md) · [22.9 eBPF track: project and next steps →](09-ebpf-track-project-and-next-steps.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/22-specializations/08-ebpf-track-maps-and-tracing.md) <!-- nav -->

# Lesson 22.8 — eBPF track: maps and tracing

> **Phase 22 — Specializations** · Lesson 8 of 30 · ⏱ about 135 minutes

## What you will learn

- The main kinds of eBPF maps, and how to declare and use them on both sides with Aya.
- How to share a data type between the kernel side and the user side in a common crate.
- How to read tracepoint arguments, and why memory of a process must be read with a helper.
- How to send events to user space through a ring buffer.
- How to attach a kprobe to a kernel function, and why per-CPU maps make counting cheap and correct.

## Before you start

- You finished [Lesson 22.7 — eBPF track: a first program with Aya](07-ebpf-track-a-first-program-with-aya.md); the
  workspace `code/p22-tracer` builds and runs in the VM.
- You know the tracepoint format file from [Lesson 22.6](06-ebpf-track-what-ebpf-is.md) and `#[repr(C)]` from
  [Lesson 12.11 — Memory layout of Rust types](../12-computer-systems/11-memory-layout-of-rust-types.md).

## New words

| Word | Meaning |
|---|---|
| **hash map** (eBPF) | A map from keys to values with a fixed maximum number of entries, for example "command name → count". |
| **array map** | A map whose keys are the numbers `0..N`; all entries exist from the start. |
| **per-CPU map** | A map in which every CPU has its own copy of each value; user space adds the copies together. |
| **ring buffer** (eBPF) | A map that works as a queue of variable-size records from the kernel to user space. |
| **event** | One record about something that happened, sent from the kernel side to the user side. |
| **probe read** | Copying memory with a helper (`bpf_probe_read_*`), which fails safely instead of crashing if the address is bad. |
| **user memory / kernel memory** | Memory of a process (where `cat` keeps its file name) / memory of the kernel itself. eBPF reads them with different helpers. |
| **`#[repr(C)]`** | "Lay out this struct like C does": a fixed field order, so that both sides agree on every byte (Lesson 12.11). |
| **kprobe** | A hook at the start of a kernel function (Lesson 22.6); here `vfs_read`, which handles every `read`. |
| **`vfs_read`** | The kernel function behind the `read` system call. VFS (Virtual File System) is the layer that all file systems share. |

**Abbreviations from earlier lessons:** BPF (Berkeley Packet Filter, the name eBPF grew from), VM (virtual machine;
the Linux VM `dev`), CPU (Central Processing Unit) and IP (Internet Protocol; an IP address identifies a machine on a
network, Lesson 15.2).

## 1. Maps: the memory between kernel and user space

An eBPF program cannot keep data in a global variable from one run to the next, and it cannot hand data to user
space directly. Everything that must survive or travel goes into a **map**. Each map has a type, a key type, a value
type and a maximum size, all fixed when it is created. The most useful types:

| Map type | Aya type (both sides) | Typical use |
|---|---|---|
| hash map | `HashMap<K, V>` | counters or state per key: per process, per IP address |
| per-CPU hash map | `PerCpuHashMap<K, V>` | the same, but with no contention between CPUs |
| array | `Array<V>` | configuration, a few fixed slots |
| per-CPU array | `PerCpuArray<V>` | fast global counters (Lesson 22.9) |
| ring buffer | `RingBuf` | a stream of events to user space |

On the kernel side a map is a `static` with the `#[map]` attribute; the macro puts its definition into the `maps`
section of the object file, and `Ebpf::load` creates it in the kernel. On the user side you get a handle by name with
`ebpf.map("NAME")` (borrow it) or `ebpf.take_map("NAME")` (own it, for example to move it into a task) and convert it
into the typed wrapper with `try_from`.

This lesson turns the tracer of Lesson 22.7 into a small tool with two programs:

- a **tracepoint** program on `sys_enter_openat` that sends one event per opened file through a ring buffer;
- a **kprobe** program on `vfs_read` that counts reads per command in a per-CPU hash map.

The user side prints the events as they come, and the top counters when you press Ctrl-C.

## 2. A shared type in a common crate

Both sides must agree on the layout of an event, byte by byte. Put the type into a third crate that both depend on.
Create `tracer-common/Cargo.toml`:

```toml
[package]
name = "tracer-common"
version = "0.1.0"
edition.workspace = true

[dependencies]
```

And `tracer-common/src/lib.rs`:

```rust
//! Types shared by the eBPF program and the user-space program.
#![cfg_attr(not(test), no_std)]

/// One `openat` call, sent from the kernel through the ring buffer.
#[repr(C)]
#[derive(Clone, Copy)]
pub struct OpenEvent {
    pub pid: u32,
    pub comm: [u8; 16],
    pub filename: [u8; 64],
}

/// The text before the first zero byte (C strings end with a zero byte).
pub fn c_str(bytes: &[u8]) -> &str {
    let end = bytes.iter().position(|&b| b == 0).unwrap_or(bytes.len());
    core::str::from_utf8(&bytes[..end]).unwrap_or("<not utf-8>")
}

/// With no filter every command is wanted; otherwise only the named one.
pub fn wanted(filter: Option<&str>, comm: &str) -> bool {
    match filter {
        None => true,
        Some(name) => name == comm,
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn stops_at_the_first_zero_byte() {
        assert_eq!(c_str(b"cat\0\0\0"), "cat");
    }

    #[test]
    fn takes_everything_without_a_zero_byte() {
        assert_eq!(c_str(b"systemd"), "systemd");
        assert_eq!(c_str(b""), "");
    }
}
```

**What happens:**

1. `#![cfg_attr(not(test), no_std)]`: the crate is `no_std` so that the eBPF side can use it, but its tests run with
   `std` on any machine (the same trick as in Lesson 22.2).
2. `#[repr(C)]` fixes the order and the padding of the fields. Without it the compiler may reorder fields, and the
   two sides, built by different toolchains for different targets, could disagree.
3. Only fixed-size arrays inside: an event must have a size known at compile time, because the kernel side reserves
   exactly `size_of::<OpenEvent>()` bytes (84 here).
4. `c_str` and `wanted` are pure helpers for the user side; they live here so they can be tested anywhere.

Add the crate to the workspace: in the root `Cargo.toml` the member lists become

```toml
members = ["tracer", "tracer-common", "tracer-ebpf"]
default-members = ["tracer", "tracer-common"]
```

and both `tracer/Cargo.toml` and `tracer-ebpf/Cargo.toml` get the dependency line
`tracer-common = { path = "../tracer-common" }` under `[dependencies]`. This version of the tracer no longer uses
`aya-log`, so remove the lines `aya-log`, `aya-log-ebpf`, `env_logger` and `log` from the workspace and from both
crates, and change the Tokio line to `tokio = { version = "1", features = ["macros", "rt-multi-thread", "signal",
"time"] }` (the user side now sleeps with `tokio::time`).

## 3. Reading the arguments of a tracepoint

The context of a tracepoint points to the **record** described by the format file. From Lesson 22.6:

```text
	field:const char * filename;	offset:24;	size:8;	signed:0;
```

So 24 bytes after the start of the record there is an 8-byte **pointer** to the file name. Two steps read it:

1. `ctx.read_at::<*const u8>(24)` copies the pointer itself out of the record (kernel memory).
2. The pointer points into the memory of the process that called `openat`, **user memory**, which the kernel may not
   simply dereference: the page could be swapped out, or the address could be invalid. The helper
   `bpf_probe_read_user_str_bytes(ptr, &mut buffer)` copies a zero-terminated string from there, at most
   `buffer.len()` bytes, and returns an error instead of crashing if the address is bad.

The verifier enforces this. If you write `*filename` instead of using the helper, loading fails:

```text
; (*event).filename[0] = *filename; // read user memory directly: wrong!
124: (71) r1 = *(u8 *)(r8 +0)
R8 invalid mem access 'scalar'
```

The verifier tracks what each register holds. `r8` holds a number that came out of `read_at` — for the verifier just
a "scalar", not a pointer it trusts — so reading memory through it is refused.

## 4. The kernel side

Replace `tracer-ebpf/src/main.rs`:

```rust
#![no_std]
#![no_main]

use aya_ebpf::{
    EbpfContext,
    helpers::bpf_probe_read_user_str_bytes,
    macros::{kprobe, map, tracepoint},
    maps::{PerCpuHashMap, RingBuf},
    programs::{ProbeContext, TracePointContext},
};
use tracer_common::OpenEvent;

/// Events for user space: 256 KiB of space in the kernel.
#[map]
static EVENTS: RingBuf = RingBuf::with_byte_size(256 * 1024, 0);

/// How often each command called `vfs_read`, keyed by command name.
/// Every CPU has its own counters, so no two CPUs write the same value.
#[map]
static READS: PerCpuHashMap<[u8; 16], u64> = PerCpuHashMap::with_max_entries(1024, 0);

/// Offset of `filename` in the record of the `sys_enter_openat` tracepoint
/// (see /sys/kernel/tracing/events/syscalls/sys_enter_openat/format).
const FILENAME_OFFSET: usize = 24;

#[tracepoint]
pub fn openat(ctx: TracePointContext) -> u32 {
    let _ = try_openat(&ctx);
    0
}

fn try_openat(ctx: &TracePointContext) -> Result<(), i32> {
    // SAFETY: the format file says that a pointer is stored at this offset.
    let filename: *const u8 = unsafe { ctx.read_at(FILENAME_OFFSET)? };
    let Some(mut entry) = EVENTS.reserve::<OpenEvent>(0) else {
        return Ok(()); // the ring buffer is full: drop this event
    };
    let event = entry.as_mut_ptr();
    // SAFETY: `event` points to our reserved space in the ring buffer.
    unsafe {
        (*event).pid = ctx.tgid();
        (*event).comm = ctx.command().unwrap_or([0; 16]);
        if bpf_probe_read_user_str_bytes(filename, &mut (*event).filename).is_err() {
            (*event).filename[0] = 0;
        }
    }
    entry.submit(0);
    Ok(())
}

#[kprobe]
pub fn count_reads(ctx: ProbeContext) -> u32 {
    let Ok(comm) = ctx.command() else { return 0 };
    match READS.get_ptr_mut(&comm) {
        // SAFETY: the pointer points to this CPU's value inside the map.
        Some(count) => unsafe { *count += 1 },
        None => {
            let _ = READS.insert(&comm, &1, 0);
        }
    }
    0
}

#[cfg(not(test))]
#[panic_handler]
fn panic(_info: &core::panic::PanicInfo) -> ! {
    loop {}
}

#[unsafe(link_section = "license")]
#[unsafe(no_mangle)]
static LICENSE: [u8; 13] = *b"Dual MIT/GPL\0";
```

**What happens:**

1. Two maps are declared as statics. `RingBuf::with_byte_size(256 * 1024, 0)` asks for 256 KiB (the size must be a
   power-of-two number of pages). The per-CPU hash map holds up to 1024 commands.
2. `EVENTS.reserve::<OpenEvent>(0)` reserves space for one event **inside the ring buffer itself**. If the buffer is
   full (user space reads too slowly), it returns `None` and the event is dropped; the kernel never waits.
3. We fill the reserved space in place through a raw pointer and `submit` it, which makes it visible to user space.
   Every reserved entry must be submitted or discarded on every path; the verifier checks that too.
4. `ctx.tgid()` is the process ID that you know from `ps` (the "thread group ID"); `ctx.pid()` would be the ID of
   the single thread.
5. `count_reads` is a kprobe program: `ProbeContext` gives access to the registers of the probed function, and the
   same `command()` helper. It runs on **every** `read` of every process, so it must be tiny.
6. With a per-CPU map, `get_ptr_mut` returns a pointer to **this CPU's** value. An eBPF program is never moved to
   another CPU while it runs, so `*count += 1` cannot race with another CPU. With a normal hash map two CPUs could
   read the same old value and both write `old + 1`, losing a count.
7. The first time a command is seen, `insert` creates its entry with the value 1.

## 5. The user side

Replace `tracer/src/main.rs`:

```rust
use std::time::Duration;

use aya::maps::{PerCpuHashMap, RingBuf};
use aya::programs::{KProbe, TracePoint};
use tracer_common::{OpenEvent, c_str, wanted};

#[tokio::main]
async fn main() -> anyhow::Result<()> {
    // `tracer cat` shows only what programs named `cat` do.
    let only = std::env::args().nth(1);

    let mut ebpf = aya::Ebpf::load(aya::include_bytes_aligned!(concat!(
        env!("OUT_DIR"),
        "/tracer"
    )))?;

    let openat: &mut TracePoint = ebpf.program_mut("openat").unwrap().try_into()?;
    openat.load()?;
    openat.attach("syscalls", "sys_enter_openat")?;

    let reads: &mut KProbe = ebpf.program_mut("count_reads").unwrap().try_into()?;
    reads.load()?;
    reads.attach("vfs_read", 0)?;

    let mut events = RingBuf::try_from(ebpf.take_map("EVENTS").unwrap())?;
    let filter = only.clone();
    tokio::spawn(async move {
        loop {
            while let Some(item) = events.next() {
                // SAFETY: the eBPF program writes exactly one OpenEvent per item.
                let event = unsafe { (item.as_ptr() as *const OpenEvent).read_unaligned() };
                let comm = c_str(&event.comm);
                if wanted(filter.as_deref(), comm) {
                    println!("{:>7} {:<12} {}", event.pid, comm, c_str(&event.filename));
                }
            }
            tokio::time::sleep(Duration::from_millis(50)).await;
        }
    });

    println!("Tracing openat and vfs_read. Press Ctrl-C to stop.");
    tokio::signal::ctrl_c().await?;

    let counts: PerCpuHashMap<_, [u8; 16], u64> =
        PerCpuHashMap::try_from(ebpf.map("READS").unwrap())?;
    let mut top: Vec<(u64, String)> = counts
        .iter()
        .filter_map(Result::ok)
        .map(|(comm, per_cpu)| (per_cpu.iter().sum(), c_str(&comm).to_string()))
        .filter(|(_, comm)| wanted(only.as_deref(), comm))
        .collect();
    top.sort_unstable_by(|a, b| b.cmp(a));
    println!("\nvfs_read calls by command:");
    for (n, comm) in top.iter().take(5) {
        println!("{n:>8} {comm}");
    }
    Ok(())
}
```

**What happens:**

1. The two programs are loaded and attached separately. `KProbe::attach("vfs_read", 0)` means "the kernel function
   `vfs_read`, at offset 0", its first instruction.
2. `take_map("EVENTS")` takes ownership of the ring buffer, so it can move into the spawned task. `RingBuf::next()`
   returns the next record, or `None` when the buffer is empty; the task then sleeps 50 ms and checks again.
3. A record is a slice of bytes. `read_unaligned` copies an `OpenEvent` out of it; the `unsafe` is justified because
   the kernel side always writes exactly that type.
4. After Ctrl-C, `ebpf.map("READS")` borrows the counter map. Each value is a `PerCpuValues<u64>`: one number per CPU,
   which we add with `.iter().sum()`.
5. The optional command-line argument filters both outputs through `wanted`.

## 6. Run it

Copy the workspace into the VM as in Lesson 22.7, then:

```bash
cargo build --release
cargo test -p tracer-common
cargo run --release -- cat
```

In a second VM terminal, run `cat /etc/hostname > /dev/null` and `cat /etc/os-release > /dev/null`, then press Ctrl-C
in the first one:

```text
running 2 tests
test tests::stops_at_the_first_zero_byte ... ok
test tests::takes_everything_without_a_zero_byte ... ok

test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

```text
Tracing openat and vfs_read. Press Ctrl-C to stop.
  50541 cat          /etc/ld.so.cache
  50541 cat          /lib/aarch64-linux-gnu/libc.so.6
  50541 cat          /etc/hostname
  50542 cat          /etc/ld.so.cache
  50542 cat          /lib/aarch64-linux-gnu/libc.so.6
  50542 cat          /etc/os-release

vfs_read calls by command:
       6 cat
```

This output comes from a Debian container with the classic `cat`; the `cat` of Ubuntu 26.04 opens more files (you
saw the list in Lesson 22.6). Without the `cat` argument the tool shows every process on the machine — try it for a
second, and you see how busy even an idle system is.

> ⚡ **Performance:** `count_reads` runs on every `read` of the whole machine. That is why it does almost nothing:
> one helper call, one map lookup, one addition. Never send an event per `read` to user space; count in the kernel and
> read the totals. A per-CPU map also avoids the cache-line ping-pong between CPUs that a shared counter causes
> (Lesson 12.10).

> 🔒 **Security:** This tool sees the file names opened by every user on the machine. Tracing data is sensitive: keep
> such tools root-only and do not send their output anywhere you would not send the files themselves.

## Common mistakes

> ⚠️ **Common mistake:** Dereferencing a pointer from a tracepoint record (`*filename`). The verifier rejects it with
> `invalid mem access 'scalar'`. Read user memory with `bpf_probe_read_user_str_bytes` (strings) or
> `bpf_probe_read_user` (fixed-size values).

> ⚠️ **Common mistake:** Forgetting `#[repr(C)]` on the shared event type. Both sides compile, but the user side may
> read the fields at the wrong offsets and print garbage.

> ⚠️ **Common mistake:** Taking an offset from the wrong format file (`sys_enter_open` instead of
> `sys_enter_openat`, or from another kernel). Nothing fails; you just read the wrong 8 bytes. Always check
> `/sys/kernel/tracing/events/.../format` on the target kernel.

> ⚠️ **Common mistake:** A plain `HashMap` with `get` + `insert(count + 1)` for counters. On a machine with many CPUs
> the counts come out too low, because two CPUs can update the same value at the same time. Use a per-CPU map and add
> the values in user space.

## Try it yourself

1. Create `tracer-common`, update the manifests and replace both `main.rs` files as shown.
2. Run `cargo run --release -- cat` and generate events from a second terminal.
3. Run it without an argument for two seconds and look at the top five readers.
4. Commit: `git add -A && git commit -m "p22-tracer: ring buffer events and kprobe counters"`.

## Exercises

### Exercise 1 — Count writes instead (easy)

Attach `count_reads` to `vfs_write` instead of `vfs_read` (and change the heading of the summary). Run
`cargo run --release -- cat`, and in the second terminal `cat /etc/os-release > /dev/null`. How many writes does
`cat` make?

<details>
<summary>Hint</summary>

Only the user side changes: the name passed to `KProbe::attach`. The eBPF program does not know which function it is
attached to.

</details>

<details>
<summary>Answer</summary>

In `tracer/src/main.rs`:

```rust
    reads.attach("vfs_write", 0)?;
```

```rust
    println!("\nvfs_write calls by command:");
```

```text
vfs_write calls by command:
       1 cat
```

Why it works: a kprobe program can be attached to any kernel function; the same counter code now counts writes. `cat`
reads the small file in one go and writes it with one `write` call.

</details>

### Exercise 2 — Test the filter (medium)

`wanted` decides what the tool prints, but it has no test yet. Write a test `wanted_filters_by_name` in
`tracer-common` that checks three cases: no filter accepts any command, a filter accepts the same name, and a filter
rejects another name. Run `cargo test -p tracer-common`.

<details>
<summary>Hint</summary>

`wanted(None, "systemd")` should be `true`, `wanted(Some("cat"), "cat")` `true`, `wanted(Some("cat"), "catalog")`
`false`.

</details>

<details>
<summary>Answer</summary>

```rust
    #[test]
    fn wanted_filters_by_name() {
        assert!(wanted(None, "systemd"));
        assert!(wanted(Some("cat"), "cat"));
        assert!(!wanted(Some("cat"), "catalog"));
    }
```

```text
test result: ok. 3 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

Why it works: the test runs on any machine, without root and without a kernel, because the common crate is plain
Rust. The `catalog` case makes sure that the filter compares whole names, not prefixes.

</details>

### Exercise 3 — Read or write? (hard)

Add the `flags` argument of `openat` to the event and print `read ` or `write` before each file name. The format file
shows `flags` at offset 32 (8 bytes). The low two bits of the flags are the access mode: `0` means read-only, `1`
write-only, `2` read-write. Test it with `cargo run --release -- touch` and `touch /tmp/p22-new-file`.

<details>
<summary>Hint</summary>

Add `pub flags: u32` to `OpenEvent` (after `pid`, so that the struct stays nicely aligned), read the value with
`ctx.read_at::<u64>(32)`, and store it with `as u32`. In user space: `event.flags & 0b11 == 0` means read-only.

</details>

<details>
<summary>Answer</summary>

`tracer-common/src/lib.rs`:

```rust
pub struct OpenEvent {
    pub pid: u32,
    pub flags: u32,
    pub comm: [u8; 16],
    pub filename: [u8; 64],
}
```

`tracer-ebpf/src/main.rs` (new constant, and two new lines in `try_openat`):

```rust
/// Offset of `flags` in the same record.
const FLAGS_OFFSET: usize = 32;
```

```rust
    // SAFETY: the format file says that `flags` is stored at this offset.
    let flags: u64 = unsafe { ctx.read_at(FLAGS_OFFSET)? };
```

```rust
        (*event).flags = flags as u32;
```

`tracer/src/main.rs`, inside `if wanted(...)`:

```rust
                    let mode = if event.flags & 0b11 == 0 {
                        "read "
                    } else {
                        "write"
                    };
                    println!(
                        "{:>7} {:<12} {mode} {}",
                        event.pid,
                        comm,
                        c_str(&event.filename)
                    );
```

```text
Tracing openat and vfs_read. Press Ctrl-C to stop.
  60866 touch        read  /etc/ld.so.cache
  60866 touch        read  /lib/aarch64-linux-gnu/libc.so.6
  60866 touch        write /tmp/p22-new-file
```

Why it works: tracepoint arguments are just fields of the record at fixed offsets, and `flags` is a plain number in
kernel memory, so `read_at` is enough. Because both sides use the same `OpenEvent`, adding a field changes both at
once.

</details>

## Quiz

1. Why can an eBPF program not simply keep a counter in a normal `static mut` variable and let user space read it?

<details>
<summary>Answer</summary>

Data that must survive between runs or be read by user space lives in maps; the loader creates them in the kernel and
user space reads them with the `bpf` system call. Maps also give the verifier a known size and type to check.

</details>

2. What do `reserve` and `submit` of a ring buffer do, and what happens when the buffer is full?

<details>
<summary>Answer</summary>

`reserve` takes space for one record inside the ring buffer; the program fills it in place; `submit` makes it visible
to user space. When the buffer is full, `reserve` returns `None` and the program drops the event instead of waiting.

</details>

3. Why is the file name read with `bpf_probe_read_user_str_bytes` and not with `*ptr`?

<details>
<summary>Answer</summary>

The pointer points into the memory of the calling process, which may be invalid or not present. The helper copies
safely and returns an error on a bad address; the verifier refuses a direct dereference of such a value.

</details>

4. Why is a per-CPU hash map the right choice for counting `vfs_read` calls?

<details>
<summary>Answer</summary>

Each CPU updates only its own copy, so increments from different CPUs can never overwrite each other, and there is no
contention on one shared value. User space adds the per-CPU values.

</details>

5. Why does the shared event type need `#[repr(C)]`?

<details>
<summary>Answer</summary>

Both sides must place every field at the same offset. `#[repr(C)]` fixes the order and padding; the default Rust
layout may differ between builds and targets.

</details>

## Summary

- Maps are the only way to keep state and to talk to user space: hash maps, arrays, per-CPU variants and ring
  buffers, declared with `#[map]` and opened by name on the user side.
- A `#[repr(C)]` type in a `no_std` common crate is shared by both sides; its pure helpers are tested with `cargo test`.
- Tracepoint arguments are read at the offsets from the format file with `read_at`; user memory is copied with
  `bpf_probe_read_user_*` helpers, or the verifier refuses the program.
- Ring buffer: `reserve` → fill → `submit` in the kernel; `next()` in user space.
- A kprobe attaches to a kernel function (`vfs_read`); count in the kernel with a per-CPU map and sum in user space.

## Sources and further reading

- [Linux kernel documentation — BPF maps](https://docs.kernel.org/bpf/maps.html) — every map type and its rules.
- [Linux kernel documentation — BPF ring buffer](https://docs.kernel.org/bpf/ringbuf.html) — how `reserve` and
  `submit` work.
- [`aya::maps`](https://docs.rs/aya/0.16.0/aya/maps/index.html) — the user-side map types.
- [`aya_ebpf::maps`](https://docs.rs/aya-ebpf/0.3.0/aya_ebpf/maps/index.html) — the kernel-side map types.
- [bpf-helpers(7)](https://man7.org/linux/man-pages/man7/bpf-helpers.7.html) — the manual page of all helper functions,
  including `bpf_probe_read_user_str`.

---

[🏠 Start](../START-HERE.md) · [← 22.7 eBPF track: a first program with Aya](07-ebpf-track-a-first-program-with-aya.md) · [📋 Phase 22](README.md) · [22.9 eBPF track: project and next steps →](09-ebpf-track-project-and-next-steps.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
