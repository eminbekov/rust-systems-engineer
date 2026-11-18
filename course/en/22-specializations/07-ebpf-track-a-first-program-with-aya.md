<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 22.6 eBPF track: what eBPF is](06-ebpf-track-what-ebpf-is.md) · [📋 Phase 22](README.md) · [22.8 eBPF track: maps and tracing →](08-ebpf-track-maps-and-tracing.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/22-specializations/07-ebpf-track-a-first-program-with-aya.md) <!-- nav -->

# Lesson 22.7 — eBPF track: a first program with Aya

> **Phase 22 — Specializations** · Lesson 7 of 30 · ⏱ about 135 minutes

## What you will learn

- How an Aya project is organised: a kernel-side crate, a user-side crate and a build script that connects them.
- How to write an eBPF program in Rust that runs on a tracepoint and logs a message.
- How to load and attach it from a Tokio program, and how its log messages reach your terminal.
- How to build and run it in the VM, and how to look at it from the outside with `bpftool` and `readelf`.

## Before you start

- You finished [Lesson 22.6 — eBPF track: what eBPF is](06-ebpf-track-what-ebpf-is.md): the VM `dev` has nightly
  Rust with `rust-src`, `bpf-linker` and `bpftool`.
- You know Cargo workspaces ([Lesson 08.12](../08-software-design/12-project-layout-and-workspaces.md)), build
  scripts ([Lesson 06.10](../06-rust-advanced/10-ffi-calling-c.md)) and Tokio
  ([Lesson 14.13](../14-concurrency-and-async/13-tokio-tasks-and-channels.md)).
- `no_std` and panic handlers are explained below in two sentences; the OS track explains them in depth in
  [Lesson 22.2](02-os-track-a-freestanding-binary.md) if you want more.

## New words

| Word | Meaning |
|---|---|
| **kernel side / user side** | The eBPF program that runs in the kernel, and the normal program that loads it and reads its results. |
| **`aya-ebpf`** | The crate for the kernel side: macros such as `#[tracepoint]`, maps, helpers. It is `no_std`. |
| **`aya`** | The crate for the user side: load an object file, attach programs, read maps. |
| **`aya-build`** | A crate for `build.rs` that builds the kernel side with nightly and puts the object file into `OUT_DIR`. |
| **`aya-log` / `aya-log-ebpf`** | A logging pair: `info!` in the kernel side writes records into a map; the user side turns them into normal log lines. |
| **context** | The value the kernel passes to your program, here `TracePointContext`: access to the event and to the current process. |
| **object file** | The compiled eBPF program: an ELF file with code sections, map definitions and a license. |
| **`OUT_DIR`** | A folder that Cargo gives each build script for generated files; `env!("OUT_DIR")` finds it at compile time. |
| **runner** | A program that `cargo run` starts with your binary as argument (Lesson 22.3); here `sudo -E`. |
| **`RUST_LOG`** | An environment variable that tells `env_logger` which log levels to print, for example `info`. |
| **GPL** (GNU General Public License) | A software license. The kernel lets only GPL-compatible eBPF programs use many helpers. |

**Abbreviations from earlier lessons:** BPF (Berkeley Packet Filter, the name eBPF grew from), OS (operating system),
VM (virtual machine; the Linux VM `dev`), CPU (Central Processing Unit), ELF (Executable and Linkable Format), ID
(identifier), UTF-8 (the Unicode text encoding, Lesson 12.5) and CI (continuous integration, Lesson 09.12).

## 1. The shape of an Aya project

An Aya project is a Cargo **workspace** with (at least) two crates:

```text
code/p22-tracer/
├── Cargo.toml              ← the workspace: members and shared versions
├── .cargo/config.toml      ← `cargo run` uses `sudo -E` as runner
├── tracer-ebpf/            ← kernel side: no_std, built for bpfel-unknown-none
│   ├── Cargo.toml
│   ├── build.rs
│   └── src/{lib.rs, main.rs}
└── tracer/                 ← user side: normal Rust, Tokio
    ├── Cargo.toml
    ├── build.rs            ← builds tracer-ebpf with nightly (aya-build)
    └── src/main.rs         ← embeds the object file, loads, attaches
```

The trick is in `tracer/build.rs`: when you build the user side with your normal stable toolchain, its build script
starts a **second** Cargo build of `tracer-ebpf` with nightly, for the target `bpfel-unknown-none`, using
`bpf-linker`. The resulting object file lands in `OUT_DIR`, and `tracer/src/main.rs` includes its bytes at compile
time. You run one command; you get one binary that carries its eBPF program inside.

The program of this lesson attaches to the tracepoint `sched:sched_process_exec`, which fires every time a process
starts a new program (you traced it with bpftrace in Exercise 2 of Lesson 22.6). For each event it logs the process
ID and the program name.

## 2. The workspace

On your Mac, create the folders:

```bash
cd ~/projects/rust
mkdir -p code/p22-tracer/.cargo code/p22-tracer/tracer/src code/p22-tracer/tracer-ebpf/src
cd code/p22-tracer
```

`Cargo.toml` (the workspace):

```toml
[workspace]
resolver = "3"
members = ["tracer", "tracer-ebpf"]
default-members = ["tracer"]

[workspace.package]
edition = "2024"

[workspace.dependencies]
aya = "0.16"
aya-build = "0.2"
aya-ebpf = "0.3"
aya-log = "0.4"
aya-log-ebpf = "0.3"
anyhow = "1"
cargo_metadata = "0.23"
env_logger = "0.11"
log = "0.4"
tokio = { version = "1", features = ["macros", "rt-multi-thread", "net", "signal"] }
which = "8"

[profile.release.package.tracer-ebpf]
debug = 2
codegen-units = 1
```

**What happens:**

1. `default-members = ["tracer"]`: a plain `cargo build` builds only the user side. The kernel side is never built
   for your normal target; the build script builds it for eBPF.
2. `[workspace.dependencies]` lists every version once; the member crates write `aya.workspace = true`.
3. The profile settings for `tracer-ebpf` keep debug information (the loader uses it to understand types) and
   compile it as one unit, which `bpf-linker` handles best.

`.cargo/config.toml`:

```toml
[target."cfg(all())"]
runner = "sudo -E"
```

`cfg(all())` is a condition that is always true, so this applies to every target: `cargo run` starts
`sudo -E target/release/tracer`. `-E` keeps your environment variables (such as `RUST_LOG`) under `sudo`.

## 3. The kernel side

`tracer-ebpf/Cargo.toml`:

```toml
[package]
name = "tracer-ebpf"
version = "0.1.0"
edition.workspace = true

[dependencies]
aya-ebpf.workspace = true
aya-log-ebpf.workspace = true

[build-dependencies]
which.workspace = true

[[bin]]
name = "tracer"
path = "src/main.rs"
```

`tracer-ebpf/build.rs` makes Cargo rebuild the eBPF code when `bpf-linker` changes:

```rust
fn main() {
    // Rebuild this crate when the bpf-linker program changes.
    let linker = which::which("bpf-linker").expect("bpf-linker is not installed");
    println!("cargo:rerun-if-changed={}", linker.display());
}
```

`tracer-ebpf/src/lib.rs` is one line, so that the crate also has a (empty) library part:

```rust
#![no_std]
```

And the program itself, `tracer-ebpf/src/main.rs`:

```rust
#![no_std]
#![no_main]

use aya_ebpf::{EbpfContext, macros::tracepoint, programs::TracePointContext};
use aya_log_ebpf::info;

#[tracepoint]
pub fn tracer(ctx: TracePointContext) -> u32 {
    match try_tracer(&ctx) {
        Ok(()) => 0,
        Err(_) => 1,
    }
}

fn try_tracer(ctx: &TracePointContext) -> Result<(), i32> {
    let comm = ctx.command()?;
    let len = comm.iter().position(|&b| b == 0).unwrap_or(comm.len());
    // SAFETY: the kernel writes the command name as ASCII text.
    let name = unsafe { core::str::from_utf8_unchecked(&comm[..len]) };
    info!(ctx, "exec: pid {} runs {}", ctx.pid(), name);
    Ok(())
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

1. `#![no_std]` and `#![no_main]`: there is no operating system library inside the kernel and no `main`. The
   program is only the functions marked as eBPF programs.
2. `#[tracepoint]` turns `tracer` into an eBPF program of type "tracepoint" and puts it into the section
   `tracepoint` of the object file. The kernel calls it with a `TracePointContext` and expects a `u32` back (for
   tracepoints the value is ignored; 0 is the convention).
3. The pattern `tracer` → `try_tracer` lets you use `?` inside, and turn any error into a return value at one place.
4. `ctx.command()` calls a helper that copies the command name of the current process (`comm`, up to 16 bytes,
   padded with zero bytes). We cut at the first zero byte. The `from_utf8_unchecked` is safe because command names
   are plain text; checking UTF-8 inside eBPF would cost many instructions.
5. `info!` from `aya-log-ebpf` does **not** format text in the kernel. It writes a small binary record (the
   message template and the two values) into a map named `AYA_LOGS`; the user side formats it later.
6. The panic handler is required by `no_std` (Lesson 22.2). An eBPF program can never actually panic: if the
   compiler left a reachable panic, the verifier would reject the loop in it. Write code without `unwrap` and
   indexing that could fail.
7. `LICENSE` goes into a section named `license`. The kernel reads it; helpers marked "GPL only" are refused to
   programs with an incompatible license.

## 4. The user side

`tracer/Cargo.toml`:

```toml
[package]
name = "tracer"
version = "0.1.0"
edition.workspace = true

[dependencies]
anyhow.workspace = true
aya.workspace = true
aya-log.workspace = true
env_logger.workspace = true
log.workspace = true
tokio.workspace = true

[build-dependencies]
anyhow.workspace = true
aya-build.workspace = true
cargo_metadata.workspace = true
tracer-ebpf = { path = "../tracer-ebpf" }

[lints.cargo]
unused_dependencies = "allow"
```

The build dependency on `tracer-ebpf` tells Cargo that the user side depends on the kernel side, so a change there
rebuilds everything. Rust code never uses it directly, which is why the `unused_dependencies` lint is switched off.

`tracer/build.rs`:

```rust
use anyhow::{Context as _, anyhow};

fn main() -> anyhow::Result<()> {
    let cargo_metadata::Metadata { packages, .. } = cargo_metadata::MetadataCommand::new()
        .no_deps()
        .exec()
        .context("MetadataCommand::exec")?;
    let ebpf_package = packages
        .into_iter()
        .find(|package| package.name.as_str() == "tracer-ebpf")
        .ok_or_else(|| anyhow!("tracer-ebpf package not found"))?;
    let root_dir = ebpf_package
        .manifest_path
        .parent()
        .ok_or_else(|| anyhow!("no parent for {}", ebpf_package.manifest_path))?;
    let package = aya_build::Package {
        name: ebpf_package.name.as_str(),
        root_dir: root_dir.as_str(),
        ..Default::default()
    };
    aya_build::build_ebpf([package], aya_build::Toolchain::default())
}
```

**What happens:**

1. `cargo_metadata` asks Cargo for the list of packages in the workspace, and the script finds `tracer-ebpf` and
   its folder.
2. `aya_build::build_ebpf` runs `cargo +nightly build --target bpfel-unknown-none -Z build-std=core --release` for
   that package and copies the object file into `OUT_DIR`. `Toolchain::default()` means "the toolchain named
   `nightly`".

`tracer/src/main.rs`:

```rust
use std::time::Duration;

use aya::programs::TracePoint;
use log::warn;
use tokio::signal;

#[tokio::main]
async fn main() -> anyhow::Result<()> {
    env_logger::init();

    // The eBPF object file was built by build.rs and is embedded here.
    let mut ebpf = aya::Ebpf::load(aya::include_bytes_aligned!(concat!(
        env!("OUT_DIR"),
        "/tracer"
    )))?;

    // Move log records from the kernel to env_logger, ten times per second.
    match aya_log::EbpfLogger::init(&mut ebpf) {
        Ok(mut logger) => {
            std::thread::spawn(move || {
                loop {
                    logger.flush();
                    std::thread::sleep(Duration::from_millis(100));
                }
            });
        }
        Err(e) => warn!("failed to initialize eBPF logger: {e}"),
    }

    let program: &mut TracePoint = ebpf.program_mut("tracer").unwrap().try_into()?;
    program.load()?;
    program.attach("sched", "sched_process_exec")?;

    println!("Tracing exec calls. Press Ctrl-C to stop.");
    signal::ctrl_c().await?;
    println!("Exiting...");
    Ok(())
}
```

**What happens:**

1. `include_bytes_aligned!` puts the object file into the binary at compile time, aligned in memory as the ELF parser
   needs. `Ebpf::load` parses it and creates its maps in the kernel (here the `AYA_LOGS` map of `aya-log`).
2. `EbpfLogger::init` connects to that map. The logger owns the map; we move it into a thread that empties it every
   100 ms and passes each record to `env_logger`. (The `aya-log` documentation shows a variant that waits for data
   with Tokio instead of a timer; the simple thread is enough here.)
3. `program_mut("tracer")` finds the program by its function name; `try_into()` checks that it really is a
   tracepoint program.
4. `load()` sends the bytecode to the kernel: this is where the **verifier** runs. `attach("sched",
   "sched_process_exec")` connects it to the tracepoint (category and name, as in tracefs).
5. The program stays attached while `ebpf` exists. When `main` returns (after Ctrl-C), `ebpf` is dropped, the file
   descriptors close and the kernel removes the program. Nothing is left behind.

## 5. Build and run in the VM

Copy the workspace into the VM and build it there (replace `student` with your Mac user name, as in Phase 13):

```bash
limactl shell dev
rsync -a --delete --exclude target /Users/student/projects/rust/code/p22-tracer/ ~/p22-tracer/
cd ~/p22-tracer
cargo build --release
```

The first build downloads crates and builds `core` for eBPF; it takes a minute. The end of the output (from the
verification container, where the workspace was in `/work`; yours shows your folder in the VM):

```text
warning: tracer@0.1.0:    Compiling tracer-ebpf v0.1.0 (/work/tracer-ebpf)
warning: tracer@0.1.0:    Compiling aya-log-ebpf v0.3.0
warning: tracer@0.1.0:     Finished `release` profile [optimized] target(s) in 12.72s
    Finished `release` profile [optimized] target(s) in 36.95s
```

The `warning: tracer@0.1.0:` lines are not problems: they are the output of the inner nightly build, which a build
script can only show as warnings. Now run it, with log level `info`:

```bash
RUST_LOG=info cargo run --release
```

Open a second terminal, enter the VM again (`limactl shell dev`) and run `ls` and `date` there. The first terminal
shows (among lines for other programs that start on the machine):

```text
     Running `sudo -E target/release/tracer`
Tracing exec calls. Press Ctrl-C to stop.
[2026-11-18T08:57:04Z INFO  tracer] exec: pid 6573 runs ls
[2026-11-18T08:57:04Z INFO  tracer] exec: pid 6574 runs date
```

Press Ctrl-C to stop; the program prints `Exiting...`. Your first Rust code ran inside the Linux kernel.

> 🧪 **Testing:** Run the tracer, then run a known command in the second terminal, and check that its line appears.
> This manual end-to-end check is the first test of every eBPF program. Exercise 2 adds an automated test that
> checks the object file without root.

## 6. Look at it from the outside

While the tracer runs, use `bpftool` in the second terminal:

```bash
sudo bpftool prog list name tracer
sudo bpftool map list name AYA_LOGS
```

```text
5664: tracepoint  name tracer  tag 0526e951d91946cd  gpl
	loaded_at 2026-11-18T08:37:24+0000  uid 0
	xlated 4064B  jited 2568B  memlock 8192B  map_ids 626,625
	btf_id 258
625: ringbuf  name AYA_LOGS  flags 0x0
	key 0B  value 0B  max_entries 131072  memlock 144200B
```

Your program has the type `tracepoint`, the license is GPL-compatible, the verifier-checked bytecode (`xlated`) is
4064 bytes and the machine code made by the JIT (`jited`) is 2568 bytes. The log map is a **ring buffer** of 128 KiB.
After you stop the tracer, both commands print nothing: the kernel removed them.

The object file itself is an ELF file for the "Linux BPF" machine:

```bash
f=$(find target/release/build -path '*out/tracer' -type f | head -1)
file "$f"
readelf -S -W "$f" | grep -E 'tracepoint|license|maps'
```

```text
target/release/build/tracer/dd119d04b2095d25/out/tracer: ELF 64-bit LSB relocatable, eBPF, version 1 (SYSV), with debug_info, not stripped
  [ 3] tracepoint        PROGBITS        0000000000000000 0002a0 000f48 00  AX  0   0  8
  [ 5] license           PROGBITS        0000000000000000 0011e8 00000d 00   A  0   0  1
  [ 7] maps              PROGBITS        0000000000000000 001224 00001c 00  WA  0   0  4
```

The section names are the ones the macros chose: the program in `tracepoint`, your `LICENSE` in `license`.

## Common mistakes

> ⚠️ **Common mistake:** Dropping the logger. If you write `let _ = aya_log::EbpfLogger::init(&mut ebpf);`, the
> logger and its map are closed at once, and loading the program fails with
> `fd 10 is not pointing to valid bpf_map`. Keep the logger alive (here: inside the thread).

> ⚠️ **Common mistake:** Running without `RUST_LOG=info`. `env_logger` prints only errors by default, so the program
> runs but shows no `exec:` lines.

> ⚠️ **Common mistake:** Running the binary directly as a normal user (`target/release/tracer`). Loading fails with
> `the BPF_PROG_LOAD syscall returned Operation not permitted (os error 1)`. Use `cargo run --release` (the runner
> adds `sudo -E`) or `sudo -E target/release/tracer`.

> ⚠️ **Common mistake:** Building on the Mac. The user side uses Linux system calls and does not build for macOS.
> Edit on the Mac, build and run in the VM.

## Try it yourself

1. Create the workspace exactly as in Sections 2–4, copy it into the VM and run it.
2. While it runs, start a few programs in the second terminal (`ls`, `date`, `uname -a`) and find their lines.
3. Look at the program and the map with `bpftool` while it runs, and again after Ctrl-C.
4. Commit on your Mac: `git add code/p22-tracer && git commit -m "p22-tracer: first eBPF program"`.

## Exercises

### Exercise 1 — Which CPU? (easy)

Add a second log line at the `debug` level that shows the number of the CPU the program runs on. Use the helper
`bpf_get_smp_processor_id()` from `aya_ebpf::helpers`, and run with `RUST_LOG=debug`.

<details>
<summary>Hint</summary>

Helpers are `unsafe` functions. Import `debug` next to `info` from `aya_log_ebpf`.

</details>

<details>
<summary>Answer</summary>

Change the imports and add three lines after the `info!` line in `tracer-ebpf/src/main.rs`:

```rust
use aya_ebpf::{
    EbpfContext, helpers::bpf_get_smp_processor_id, macros::tracepoint,
    programs::TracePointContext,
};
use aya_log_ebpf::{debug, info};
```

```rust
    // SAFETY: this helper only returns the number of the current CPU.
    let cpu = unsafe { bpf_get_smp_processor_id() };
    debug!(ctx, "  (on CPU {})", cpu);
```

```text
[2026-11-18T08:59:10Z INFO  tracer] exec: pid 21198 runs ls
[2026-11-18T08:59:10Z DEBUG tracer]   (on CPU 8)
[2026-11-18T08:59:10Z INFO  tracer] exec: pid 21199 runs date
[2026-11-18T08:59:10Z DEBUG tracer]   (on CPU 7)
```

Why it works: an eBPF program runs on whichever CPU the event happened on, so two commands can be traced on two
different CPUs at the same time. Lesson 22.8 uses this fact for per-CPU maps.

</details>

### Exercise 2 — Test the object file without root (medium)

Write a unit test in `tracer/src/main.rs` that parses the embedded object file with the crate `aya-obj` (add
`aya-obj = "0.6"` under `[dev-dependencies]` in `tracer/Cargo.toml`) and checks that it contains a program named
`tracer` and the license `Dual MIT/GPL`. Run it with `cargo test --release` — no `sudo` needed.

<details>
<summary>Hint</summary>

`aya_obj::Object::parse(bytes)` reads an object file in user space without talking to the kernel. The result has a
`programs` hash map and a `license` field of type `CString`.

</details>

<details>
<summary>Answer</summary>

```rust
#[cfg(test)]
mod tests {
    #[test]
    fn object_file_has_the_program_and_a_license() {
        let bytes = aya::include_bytes_aligned!(concat!(env!("OUT_DIR"), "/tracer"));
        let object = aya_obj::Object::parse(bytes).expect("not a valid eBPF object");
        assert!(object.programs.contains_key("tracer"));
        assert_eq!(object.license.to_str(), Ok("Dual MIT/GPL"));
    }
}
```

```text
test tests::object_file_has_the_program_and_a_license ... ok
test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

Why it works: `aya` itself uses `aya-obj` to read object files before it loads them. The test catches a broken
build (a renamed function, a missing license) in seconds, and it runs in CI (Lesson 09.12) without any privileges.

</details>

### Exercise 3 — Filter in the kernel (hard)

Change the eBPF program so that it logs **only** processes whose command name is `ls`. The comparison must happen in
the kernel, so that no record is written for other programs.

<details>
<summary>Hint</summary>

`comm` is a `[u8; 16]` array. Compare a fixed-size part of it, including the terminating zero byte, with a byte
string: `comm[..3] == *b"ls\0"`. A fixed-size comparison is easy for the verifier.

</details>

<details>
<summary>Answer</summary>

In `try_tracer`, right after `let comm = ctx.command()?;`:

```rust
    if comm[..3] != *b"ls\0" {
        return Ok(()); // not `ls`: do nothing
    }
```

With `ls` and `date` started in the second terminal, the whole output is now:

```text
Tracing exec calls. Press Ctrl-C to stop.
[2026-11-18T09:00:45Z INFO  tracer] exec: pid 29580 runs ls
Exiting...
```

Why it works: the zero byte at index 2 makes sure that `lsblk` (which starts with `ls`) does not match. Filtering in
the kernel is cheaper than filtering in user space: rejected events never cross into user space at all.

</details>

## Quiz

1. Why does the user-side crate have a `build.rs`?

<details>
<summary>Answer</summary>

It builds the kernel-side crate with nightly for `bpfel-unknown-none` (through `aya-build`) and puts the object file
into `OUT_DIR`, so that `include_bytes_aligned!` can embed it into the user-side binary.

</details>

2. Where does the verifier run in this program?

<details>
<summary>Answer</summary>

In `program.load()`, when the bytecode is sent to the kernel with the `bpf` system call. If it is rejected, `load()`
returns an error with the verifier log.

</details>

3. Does `info!` in the eBPF program format the text inside the kernel?

<details>
<summary>Answer</summary>

No. It writes a compact binary record into the `AYA_LOGS` ring buffer map; `aya-log` in user space reads the record
and formats the message.

</details>

4. What happens to the eBPF program when the user-side program exits?

<details>
<summary>Answer</summary>

Its file descriptors are closed, so the kernel detaches and removes the program and its maps. `bpftool prog list`
no longer shows it.

</details>

5. Why does the kernel care about the `license` section?

<details>
<summary>Answer</summary>

Many helper functions are only available to GPL-compatible programs; the kernel checks the license string when it
loads the program.

</details>

## Summary

- An Aya project has a `no_std` kernel-side crate, a normal user-side crate, and a `build.rs` that builds the kernel
  side with nightly and embeds the object file.
- `#[tracepoint]` marks a program; the context gives the current process (`pid()`, `command()`); `info!` writes log
  records into a map.
- The user side: `Ebpf::load` → keep an `EbpfLogger` alive → `program_mut(..).try_into()` → `load()` (verifier) →
  `attach(category, name)`.
- `cargo run --release` with the `sudo -E` runner and `RUST_LOG=info` runs it in the VM.
- `bpftool prog list`, `bpftool map list` and `readelf -S` show the program, its map and its object file.

## Sources and further reading

- [The Aya book](https://aya-rs.dev/book/) — the official guide; its first chapters build a similar program.
- [`aya` documentation](https://docs.rs/aya/0.16.0/aya/) — loading, programs and maps on the user side.
- [`aya-ebpf` documentation](https://docs.rs/aya-ebpf/0.3.0/aya_ebpf/) — macros, contexts and helpers on the kernel side.
- [`aya-log` documentation](https://docs.rs/aya-log/0.4.0/aya_log/) — how log records travel to user space.
- [aya-template](https://github.com/aya-rs/aya-template) — the official project template (for `cargo generate`) that
  this layout follows.
- [Linux kernel documentation — Event Tracing](https://docs.kernel.org/trace/events.html) — tracepoint events and
  their formats.

---

[🏠 Start](../START-HERE.md) · [← 22.6 eBPF track: what eBPF is](06-ebpf-track-what-ebpf-is.md) · [📋 Phase 22](README.md) · [22.8 eBPF track: maps and tracing →](08-ebpf-track-maps-and-tracing.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
