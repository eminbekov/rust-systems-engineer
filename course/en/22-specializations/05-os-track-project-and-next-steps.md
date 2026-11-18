<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 22.4 OS track: interrupts and paging](04-os-track-interrupts-and-paging.md) · [📋 Phase 22](README.md) · [22.6 eBPF track: what eBPF is →](06-ebpf-track-what-ebpf-is.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/22-specializations/05-os-track-project-and-next-steps.md) <!-- nav -->

# Lesson 22.5 — OS track: project and next steps

> **Phase 22 — Specializations** · Lesson 5 of 30 · ⏱ about 20 hours (10 sittings)

> **This is the last lesson of the OS track.** When you finish the project, continue with
> [Lesson 22.30 — Checkpoint: your specialization](30-checkpoint-your-specialization.md) and answer the OS question
> set. The lessons between here and 22.30 belong to other tracks; if you want to look at another track first, the
> next one starts at [Lesson 22.6 — eBPF track: what eBPF is](06-ebpf-track-what-ebpf-is.md).

## What you will learn

- How to turn the kernel of Lessons 22.2–22.4 into a small interactive kernel with a shell on the serial console.
- How to receive input with UART interrupts and share data safely between an interrupt handler and normal code.
- How to give a kernel a heap, so that `Vec` and `String` work.
- How to test a kernel automatically by booting it in QEMU with scripted input.
- Where to go next: books, documentation, open-source kernels and communities.

## Before you start

- You finished [Lesson 22.4 — OS track: interrupts and paging](04-os-track-interrupts-and-paging.md); your
  `code/p22-kernel` handles exceptions, turns on the MMU and counts timer ticks.
- You wrote a memory allocator in [Lesson 12.14 — Project: your own memory allocator](../12-computer-systems/14-project-your-own-allocator.md).
- You wrote a shell in [Lesson 13.8 — Project: your own shell](../13-operating-systems/08-project-your-own-shell.md);
  this kernel shell is much simpler.

## New words

| Word | Meaning |
|---|---|
| **kernel shell** | A tiny command loop inside the kernel itself: read a line from the serial console, run a built-in command. |
| **ring buffer** | A fixed-size queue whose end wraps around to its start; it needs no heap. |
| **critical section** | A piece of code that must not be interrupted while it uses shared data. On one core: mask interrupts. |
| **heap** (kernel) | A memory region from which the kernel's allocator hands out blocks for `Box`, `Vec` and `String`. |
| **`#[global_allocator]`** | The attribute that tells Rust which allocator `alloc` uses (Lesson 12.13). |
| **SPI** (Shared Peripheral Interrupt) | A GIC interrupt from a device that any core could handle, such as the UART (INTID 33 on `virt`). |
| **smoke test** | A quick end-to-end test that checks that the whole system starts and answers at all. |
| **semihosting** | A way for a program inside QEMU to ask QEMU itself for services, such as "exit with code 1". |

**Abbreviations from earlier lessons:** OS (operating system), QEMU (Quick Emulator), UART (Universal Asynchronous
Receiver-Transmitter, the serial port; PL011 is its model on `virt`), MMU (Memory Management Unit), GIC (Generic
Interrupt Controller), INTID (interrupt ID), IRQ (Interrupt Request), PSCI (Power State Coordination Interface), RAM
(Random Access Memory), VM (virtual machine) and CI (continuous integration, Lesson 09.12). New in the hints: FIFO
(First In, First Out, a hardware queue of bytes) and `IMSC` (the UART's Interrupt Mask Set/Clear register).

## 1. Where you are

Your kernel already does a lot: it boots on bare metal, prints, handles exceptions and interrupts, and owns its page
table. What it cannot do yet is **listen**. In this project it gets a keyboard (the receive side of the UART), a
command loop and a heap. That is a real milestone: the same structure (input interrupt, buffer, command handler,
allocator) is inside every kernel.

The project is your own work; this lesson gives the contract, a plan with milestones, small hints and the acceptance
checks. There is no full solution — every part uses only what you learned in Lessons 22.2–22.4 plus the register
facts given in the hints.

## The project

### What you will build

`p22-kernel` v1: a kernel that boots in QEMU, prints a prompt and runs commands that you type:

```text
p22-kernel ready. Type `help`.
p22-kernel> help
commands: help, ticks, mem, history, fault, off
p22-kernel> ticks
ticks: 13
p22-kernel> mem
kernel: 0x40080000..0x400a1000, heap free: 1023 KiB
p22-kernel> history
  1 help
  2 ticks
  3 mem
  4 history
p22-kernel> off
bye
```

This output is from a reference solution; your addresses and numbers will differ.

### Contract

| Input line | Output |
|---|---|
| `help` | `commands: help, ticks, mem, history, fault, off` |
| `ticks` | `ticks: <n>`, where the timer adds 1 every 100 ms |
| `mem` | `kernel: 0x40080000..<stack top>, heap free: <n> KiB` |
| `history` | every non-empty command so far, numbered: `{:>3} <command>` |
| `fault` | reads `0xC000_0000`; the page fault handler panics (the run ends) |
| `off` | `bye`, then PSCI `SYSTEM_OFF` |
| empty line | nothing; a new prompt |
| anything else | `unknown command: <the word>` |

Rules:

- After booting, the kernel prints `` p22-kernel ready. Type `help`. `` once. The prompt is exactly `p22-kernel> `
  (with a space). Typed characters are echoed. Backspace (byte `0x7f`) removes the last character. Both `\r` (from a terminal) and `\n` (from `printf`) end a line.
- Lines longer than 64 characters are cut at 64; nothing crashes.
- Pure logic (parsing, line editing, the ring buffer) lives in `src/lib.rs` and has unit tests.
- `cargo clippy` is clean, `cargo fmt --check` passes, there is a `README.md`, and there is one Git commit per
  milestone.

### Step-by-step plan

#### M1 — Read the keyboard by polling

**Goal:** the kernel reads bytes from the UART, echoes them, and on Enter prints `you typed: <line>`. The line `off`
prints `bye` and switches the machine off.

**Guidance:** add `Uart::read_byte(&mut self) -> Option<u8>` to `src/uart.rs`. Keep the line in a fixed array
(`[u8; 64]` plus a length): there is no heap yet. Remove the demo code of Lesson 22.4 from `kernel_main` (keep
`exceptions::init()`, `mmu::init()` and `timer::init()`). The contract wants a tick every 100 ms: if you did not
do Exercise 1 of Lesson 22.4, change both `arm(200)` calls to `arm(100)` now.

<details>
<summary>Hint</summary>

PL011 flag register (`0x18`) bit 4 is `RXFE`, "receive FIFO empty". While it is 0, a byte is waiting: read the data
register (`0x00`) with `read_volatile` and keep the low 8 bits. `core::str::from_utf8(&buf[..len])` turns the bytes
into a `&str`.

</details>

**Done when:** piped input works (the pipe goes into QEMU's serial line):

```bash
printf 'hello\noff\n' | cargo run -q
```

```text
p22-kernel ready. Type `help`.
p22-kernel> hello
you typed: hello
p22-kernel> off
bye
```

Commit: `git commit -am "p22-kernel: M1 polling input"`.

#### M2 — A line editor and a command parser, tested on the Mac

**Goal:** the commands of the contract (without `history` for now), with the parsing and editing logic in
`src/lib.rs`.

**Guidance:** design two small types. `parse(line: &str) -> Result<Command, &str>` returns a `Command` enum or the
unknown word. A `LineEditor<const N: usize>` takes one byte with `push(byte)` and says what the terminal should show
(echo the byte, erase one character, nothing, or "line complete"). Const generics (`<const N: usize>`) let the
tests use a tiny editor of 2 bytes.

<details>
<summary>Hint</summary>

An enum such as `enum Echo { Byte(u8), Erase, Nothing, Line }` keeps the editor free of printing, so it can be tested.
To erase a character on the terminal, the kernel prints `"\x08 \x08"`: back, space, back. Ignore bytes outside
`b' '..=b'~'`.

</details>

**Done when:** `cargo host-test` runs at least six tests (parser: known command, unknown word, empty line; editor:
backspace, too-long line, Enter) and they pass, and

```bash
printf 'help\nwhat\noff\n' | cargo run -q
```

```text
p22-kernel ready. Type `help`.
p22-kernel> help
commands: help, ticks, mem, history, fault, off
p22-kernel> what
unknown command: what
p22-kernel> off
bye
```

#### M3 — Interrupt-driven input

**Goal:** the kernel sleeps in `wfi` until a key arrives. The UART interrupt handler moves received bytes into a ring
buffer; `kernel_main` takes them out.

**Guidance:** write `RingBuffer<const N: usize>` in `src/lib.rs` with `push(byte) -> bool` (false when full) and
`pop() -> Option<u8>`, and test it, including wrap-around. The buffer is shared between the handler and
`kernel_main`, which is a data race unless you prevent it. On one core the fix is a **critical section**: mask IRQs
while `kernel_main` touches the buffer.

<details>
<summary>Hint 1: the hardware</summary>

- The UART interrupt is INTID **33** (an SPI). Enable it in the GIC distributor with bit 1 of the register at
  `GICD + 0x104` (the second enable register: INTIDs 32–63), and set its target byte `GICD + 0x800 + 33` to `0x01`
  (core 0) with a `u8` volatile write.
- In the UART, enable the receive interrupts with the interrupt mask register `IMSC` at offset `0x38`: bit 4 (receive)
  and bit 6 (receive timeout).
- In `handle_irq`, when the INTID is 33, read bytes until `read_byte` returns `None`. Reading the data register
  clears the interrupt. Then write `EOIR` as for the timer.

</details>

<details>
<summary>Hint 2: sharing safely</summary>

A wrapper type works well: `struct IrqCell<T>(UnsafeCell<T>)` with `unsafe impl<T> Sync for IrqCell<T> {}` and a
method `with(&self, f: impl FnOnce(&mut T) -> R) -> R` that saves `DAIF` (`mrs {}, daif`), masks IRQs
(`msr daifset, #2`), calls `f`, and restores the saved `DAIF` (`msr daif, {}`). Write a `// SAFETY:` comment that
says why it is sound: one core, and interrupts are masked while `f` runs. A `static INPUT: IrqCell<RingBuffer<64>>`
can then be used from both sides. Never wait for a lock inside an interrupt handler.

</details>

**Done when:** the main loop contains `wfi` and no busy polling of the UART, the ring-buffer tests pass, and the
tick counter runs while the kernel waits for input (about 10 ticks per second):

```bash
(sleep 1; printf 'ticks\n'; sleep 1; printf 'ticks\noff\n') | cargo run -q
```

```text
p22-kernel ready. Type `help`.
p22-kernel> ticks
ticks: 6
p22-kernel> ticks
ticks: 16
p22-kernel> off
bye
```

The first number depends on how fast the container starts; the difference between the two should be about 10.

#### M4 — A heap, `Vec` and `String`

**Goal:** `history` and `mem` from the contract, using `alloc::vec::Vec<String>` for the history.

**Guidance:** reserve 1 MiB for the heap in `linker.ld` after the stack, with two symbols `__heap_start` and
`__heap_end` (align the start to 4096). Use your allocator from Lesson 12.14 as the `#[global_allocator]`, or the
crate `linked_list_allocator = "0.10"`. Initialise the heap in `kernel_main` after `mmu::init()` and before the
first allocation. Add `extern crate alloc;` to `src/main.rs` (in a `no_std` crate `alloc` must be named explicitly).

<details>
<summary>Hint</summary>

With the crate: `static ALLOCATOR: LockedHeap = LockedHeap::empty();` marked `#[global_allocator]`, then
`unsafe { ALLOCATOR.lock().init(start as *mut u8, end - start) }`. `ALLOCATOR.lock().free()` gives the free bytes for
`mem`. The allocator uses atomic instructions, which is one more reason to turn the MMU (and normal memory) on first.

</details>

**Done when:**

```bash
printf 'ticks\nmem\nhistory\noff\n' | cargo run -q
```

```text
p22-kernel ready. Type `help`.
p22-kernel> ticks
ticks: 0
p22-kernel> mem
kernel: 0x40080000..0x400a1000, heap free: 1023 KiB
p22-kernel> history
  1 ticks
  2 mem
  3 history
p22-kernel> off
bye
```

(`ticks: 0` is normal here: piped input arrives right after boot.)

#### M5 — An automated test in QEMU, and the finish

**Goal:** a script `test.sh` that boots the kernel with scripted input and checks the answers, plus a README.

**Guidance:** capture the whole output in a shell variable, then check expected lines with `grep -q`. Exit with
code 1 at the first failure, so that a CI job (Lesson 09.12) could run it later.

<details>
<summary>Hint</summary>

```sh
out=$(printf 'help\nwhat\nticks\nhistory\noff\n' | cargo run -q 2>&1)
check() {
  if printf '%s\n' "$out" | grep -q "$1"; then echo "ok:   $1"; else echo "FAIL: $1"; exit 1; fi
}
check 'commands: help'
```

</details>

**Done when:**

```bash
./test.sh
```

```text
ok:   commands: help
ok:   unknown command: what
ok:   ticks: 
ok:     2 ticks
ok:   bye
all checks passed
```

and `cargo host-test`, `cargo clippy` and `cargo fmt --check` are clean, the README explains how to build, run and
test the kernel, and `git log --oneline` shows one commit per milestone.

### Stretch goals

- **A `fault` that does not kill the kernel.** For a data abort from a `fault` command, print the details and return
  to the shell instead of panicking. (Hint: the handler can set `frame.elr` to a "recovery" function.)
- **4 KiB pages.** Replace the RAM block with a level-2 table and level-3 tables, and map the kernel's code
  read-only and its data no-execute.
- **A second core.** Start core 1 with the PSCI call `CPU_ON` (`0xC400_0003`), give it its own stack, and let it
  count something. Now you need real locks (Lesson 14.7) instead of masking interrupts.
- **User mode.** Run a tiny function at EL0 with `eret`, and let it ask the kernel for a service with `svc` — your
  first system call, seen from the kernel side (Lesson 13.3).
- **Real test results.** Start QEMU with `-semihosting` and use the semihosting "exit" call so that a failing kernel
  test makes QEMU exit with a non-zero code.

## 2. Next steps

You have built the core of a kernel from scratch on arm64. Here is where to go deeper.

**Read and follow:**

- *Writing an OS in Rust* by Philipp Oppermann (os.phil-opp.com) — the same journey on x86-64, with the
  `bootloader` crate (0.11), the `x86_64` crate (0.15) and the `uart_16550` crate (0.9). Doing it after this track
  shows you what is the same on every machine (stack, page tables, interrupts) and what is specific to one.
- *Operating Systems: Three Easy Pieces* (OSTEP) — a free book about the ideas behind every kernel: scheduling,
  virtual memory, concurrency, file systems.
- *The Embedonomicon* — the Rust Embedded team's guide to `no_std` binaries, linker scripts and the start-up code.
- *Operating System development tutorials in Rust on the Raspberry Pi* — a long series of arm64 kernel lessons in
  Rust, from the first instruction to virtual memory and exceptions.
- The *Arm Architecture Reference Manual* and the *Learn the architecture* guides — the source of truth for every
  register you touched.

**Open-source projects to contribute to:**

| Project | What it is | A good first step |
|---|---|---|
| Rust for Linux | Rust support inside the Linux kernel: drivers and safe abstractions. | Read the documentation, build a kernel with Rust enabled in your VM, follow the mailing list and Zulip. |
| Redox OS | A complete Unix-like operating system written in Rust, with a microkernel. | Run it in QEMU and pick an issue marked for newcomers. |
| Theseus | A research OS in Rust that explores new structure (state spill, live evolution). | Read the book, build it, start with documentation or small fixes. |
| Hermit | A unikernel in Rust: an application and a small kernel linked into one image for virtual machines. | Run the examples, then look at the issue tracker. |

**Communities:** the Rust for Linux Zulip chat, the OSDev wiki and forum (a huge collection of hardware facts), the
Redox chat, and the `#os-dev` style channels in Rust community spaces. When you ask a question, say which machine,
which QEMU version and which exact output you see — exactly as you did in Lesson 00.7.

**Phase 24** shows how to choose an issue, talk to maintainers and send your first contribution.

> 💡 **Tip:** Keep a "kernel facts" page in your learning journal: every register, offset and bit you used, with the
> source. Kernel work is full of such facts, and your own list is faster than any manual.

## Common mistakes

> ⚠️ **Common mistake:** Allocating before the heap is initialised (for example a `Vec::new()` that grows in
> `exceptions::init`). The allocator returns null and `alloc` reports an allocation error, which panics. Initialise
> the heap right after the MMU.

> ⚠️ **Common mistake:** Taking a spin lock in `kernel_main` and the same lock in the interrupt handler. If the
> interrupt arrives while `kernel_main` holds the lock, the handler waits forever for a lock that can only be
> released after the handler returns: a deadlock on one core. Mask interrupts in the normal code instead.

> ⚠️ **Common mistake:** Forgetting the `u8` write for the UART target register (`GICD + 0x821`), or enabling bit 33
> in the first enable register. Interrupts 32–63 are in the *second* register (`0x104`), at bit `33 - 32 = 1`.

> ⚠️ **Common mistake:** Testing only with `printf`. A real terminal sends `\r` for Enter and `0x7f` for Backspace;
> type a few commands by hand with `cargo run` too (quit QEMU with Ctrl-A X).

## Quiz

1. Why does the kernel shell need a ring buffer between the UART interrupt and `kernel_main`?

<details>
<summary>Answer</summary>

The interrupt handler must finish quickly and cannot run the command itself, and bytes can arrive faster than
`kernel_main` handles them. The ring buffer stores them in order, in fixed memory without a heap.

</details>

2. On one core, why is masking IRQs a correct critical section for data shared with an interrupt handler?

<details>
<summary>Answer</summary>

The only other code that can touch the data is the handler, and it can only run when an IRQ is taken. With IRQs
masked, nothing else can run until they are unmasked. On several cores this is not enough; you also need locks.

</details>

3. What does a kernel need before `Vec::push` works?

<details>
<summary>Answer</summary>

`extern crate alloc;`, a `#[global_allocator]`, and a memory region that the allocator was initialised with (here
1 MiB reserved by the linker script).

</details>

4. Why does the project use `printf '...' | cargo run -q` for acceptance tests?

<details>
<summary>Answer</summary>

The pipe feeds the kernel's serial input, `-q` hides Cargo's own lines, and `off` ends the run, so the whole kernel
can be tested from a script without a person typing.

</details>

5. Name two open-source kernels written in Rust and one way to start contributing.

<details>
<summary>Answer</summary>

For example Redox OS and Theseus (also Hermit, and Rust for Linux inside Linux). Start by building and running the
project, reading its contributor guide, and picking a small issue marked for newcomers or a documentation fix.

</details>

## Summary

- The project turns the kernel into an interactive system: UART input, a command loop, a heap and an automated
  QEMU test.
- Interrupt handlers stay short: they move bytes into a ring buffer; the main loop sleeps in `wfi` and does the work.
- On one core, a critical section is "mask IRQs, touch the data, restore the mask".
- `extern crate alloc` plus a `#[global_allocator]` over a reserved region gives `Vec` and `String`.
- Pure logic is tested with `cargo host-test`; the whole kernel is tested by booting it with scripted input.
- Next: *Writing an OS in Rust* for x86-64, OSTEP for the theory, and Rust for Linux, Redox, Theseus and Hermit for
  real contributions.

## Sources and further reading

- [Writing an OS in Rust](https://os.phil-opp.com/) — the well-known Rust kernel blog for x86-64.
- [Operating Systems: Three Easy Pieces](https://pages.cs.wisc.edu/~remzi/OSTEP/) — the free OS textbook.
- [The Embedonomicon](https://docs.rust-embedded.org/embedonomicon/) — `no_std` binaries and linker scripts in depth.
- [Operating System development tutorials in Rust on the Raspberry Pi](https://github.com/rust-embedded/rust-raspberrypi-OS-tutorials) — arm64 kernel lessons in Rust.
- [Rust for Linux](https://rust-for-linux.com/) — Rust in the Linux kernel, with links to the mailing list and Zulip.
- [Redox OS](https://www.redox-os.org/) — a Rust operating system.
- [Theseus OS](https://github.com/theseus-os/Theseus) — a research operating system in Rust.
- [Hermit](https://hermit-os.org/) — a Rust unikernel.
- [`linked_list_allocator` documentation](https://docs.rs/linked_list_allocator/0.10.6/linked_list_allocator/) — the
  allocator crate suggested for M4.
- [OSDev Wiki](https://wiki.osdev.org/) — the community wiki of hobby operating system developers.

---

[🏠 Start](../START-HERE.md) · [← 22.4 OS track: interrupts and paging](04-os-track-interrupts-and-paging.md) · [📋 Phase 22](README.md) · [22.6 eBPF track: what eBPF is →](06-ebpf-track-what-ebpf-is.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
