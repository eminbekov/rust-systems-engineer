<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 22.2 OS track: a freestanding Rust binary](02-os-track-a-freestanding-binary.md) · [📋 Phase 22](README.md) · [22.4 OS track: interrupts and paging →](04-os-track-interrupts-and-paging.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/22-specializations/03-os-track-booting-a-kernel-in-qemu.md) <!-- nav -->

# Lesson 22.3 — OS track: booting a kernel in QEMU

> **Phase 22 — Specializations** · Lesson 3 of 30 · ⏱ about 2.5 hours (2 sittings)

## What you will learn

- What the QEMU `virt` machine looks like: where its memory and its devices are.
- How to run QEMU in a container and make `cargo run` boot your kernel.
- How a linker script places the kernel in memory, and what the first assembly instructions must do.
- How to talk to a device through memory-mapped registers, and how to build `println!` on top of a UART driver.
- How to print panic messages and switch the machine off.

## Before you start

- You finished [Lesson 22.2 — OS track: a freestanding Rust binary](02-os-track-a-freestanding-binary.md); you
  have `code/p22-kernel` with `src/lib.rs`, the `.cargo/config.toml` and the `host-test` alias.
- You know sections and symbols of ELF files ([Lesson 12.12](../12-computer-systems/12-linking-and-loading.md)) and
  the stack ([Lesson 03.1](../03-ownership-and-borrowing/01-memory-stack-and-heap.md)).
- You can build and run a Docker image ([Phase 11](../11-linux-shell-and-containers/README.md)).

## New words

| Word | Meaning |
|---|---|
| **memory map** | The list of address ranges of a machine and what is behind each one: RAM or a device. |
| **MMIO** (Memory-Mapped I/O, input/output) | Devices that appear as addresses: reading or writing such an address talks to the device, not to RAM. |
| **register** (of a device) | One small control or data cell of a device, at a fixed address. Not the same as a CPU register. |
| **UART** (Universal Asynchronous Receiver-Transmitter) | A simple chip that sends and receives bytes one bit at a time over a serial line. |
| **serial console** | Text that a machine sends over its UART; QEMU shows it in your terminal. |
| **PL011** | The UART model made by Arm that the `virt` machine has. |
| **volatile access** | A memory read or write that the compiler must do exactly as written, never remove or merge. Needed for devices. |
| **linker script** | A file that tells the linker at which addresses to put the code and data, and which symbols to define. |
| **BSS** (Block Started by Symbol) | The section of zero-initialised global data. It takes no space in the file; the kernel must fill it with zeros itself. |
| **stack pointer** (`sp`) | The CPU register that points to the top of the stack. |
| **`global_asm!`** | A Rust macro that puts assembly code into the program outside of any function. |
| **PSCI** (Power State Coordination Interface) | An Arm standard for asking the firmware to switch CPUs or the whole machine on and off. |
| **Cargo runner** | A program that `cargo run` starts instead of your binary, with the binary's path as argument. |
| **DTB** (Device Tree Blob) | A file that describes a machine's hardware. QEMU puts one at the start of RAM. |

**Abbreviations from earlier lessons:** OS (operating system), CPU (Central Processing Unit), RAM (Random Access
Memory, the main working memory), ELF (Executable and Linkable Format, the file format of the kernel) and QEMU (Quick
Emulator). GIC (Generic Interrupt Controller) appears in the memory map; Lesson 22.4 uses it.

## 1. The machine we boot

QEMU is a program that emulates a whole computer: its CPU, memory and devices. The `virt` machine is a
simple arm64 board. Its most important addresses:

```text
 address        what is there
 0x0000_0000    flash memory (firmware, unused by us)
 0x0800_0000    GIC: the interrupt controller (Lesson 22.4)
 0x0900_0000    UART PL011: our serial console
 0x4000_0000    RAM starts (128 MiB with -m 128M)
                0x4000_0000  the DTB that QEMU writes here
                0x4008_0000  our kernel
```

When you start QEMU with `-kernel <file>` and the file is an ELF, QEMU copies each part of the file to the
address written in the file, sets the program counter to the ELF entry point and starts one CPU. There is no
firmware, no bootloader and no operating system: your `_start` is the first instruction that runs.

So the kernel must be linked for an address inside RAM. We use `0x4008_0000`, which leaves the first
512 KiB of RAM for the DTB.

## 2. QEMU in a container, started by `cargo run`

You never install QEMU on your Mac. It runs in a container from the official `ubuntu:26.04` image. Create the
file `qemu/Dockerfile` inside `code/p22-kernel`:

```dockerfile
FROM ubuntu:26.04
RUN apt-get update && apt-get install -y --no-install-recommends qemu-system-arm && rm -rf /var/lib/apt/lists/*
```

Build the image once (it takes a minute or two):

```bash
docker build -t course-qemu qemu
docker image ls --filter reference='course-qemu'
```

```text
IMAGE                ID             DISK USAGE   CONTENT SIZE   EXTRA
course-qemu:latest   89f2b5e40a87        378MB         81.3MB
```

The package `qemu-system-arm` contains `qemu-system-aarch64`, the emulator for 64-bit Arm machines. Next, a
small script `qemu.sh` in the package folder starts it:

```sh
#!/bin/sh
# Boots the kernel ELF file that Cargo passes as $1 in QEMU, inside a container.
set -eu
tty_flag=""
if [ -t 0 ]; then tty_flag="-t"; fi  # a real terminal: let Ctrl-A X reach QEMU
docker run --rm -i $tty_flag -v "$PWD":/work -w /work course-qemu \
  qemu-system-aarch64 -machine virt -cpu cortex-a72 -m 128M \
  -nographic -nic none -kernel "$1"
```

Make it executable with `chmod +x qemu.sh`. The QEMU options:

| Option | Meaning |
|---|---|
| `-machine virt` | Emulate the `virt` board. |
| `-cpu cortex-a72` | A real 64-bit Arm CPU model. Without it, `virt` starts a 32-bit CPU. |
| `-m 128M` | 128 MiB of RAM. |
| `-nographic` | No window: the serial console goes to your terminal. Quit with **Ctrl-A**, then **X**. |
| `-nic none` | No network card (we have no driver, and QEMU would look for an extra boot file). |
| `-kernel "$1"` | Load this ELF file and start at its entry point. |

`-v "$PWD":/work` makes the current folder visible inside the container, so the path that Cargo passes
(`target/.../p22-kernel`) works there too. Therefore always run `cargo run` from the package folder.

Now connect everything in `.cargo/config.toml`:

```toml
[build]
target = "aarch64-unknown-none-softfloat"

[target.aarch64-unknown-none-softfloat]
# Use our linker script, and boot the kernel in QEMU on `cargo run`.
rustflags = ["-C", "link-arg=-Tlinker.ld"]
runner = "./qemu.sh"

[alias]
# Runs the tests of src/lib.rs on the Mac instead of the kernel target.
host-test = "test --lib --target aarch64-apple-darwin"
```

**What happens:**

1. `rustflags` adds `-Tlinker.ld` to the linker command, so the linker uses our linker script (next section).
2. `runner` tells Cargo: for `cargo run`, do not start the binary (impossible on a Mac); start `./qemu.sh`
   with the path of the binary instead.
3. Both settings are inside `[target.aarch64-unknown-none-softfloat]`, so they apply only to the kernel
   target. `cargo host-test` builds for the Mac and is not affected.

## 3. The linker script

Create `linker.ld` in the package folder (next to `Cargo.toml`):

```text
ENTRY(_start)

SECTIONS
{
    . = 0x40080000;

    .text : {
        KEEP(*(.text.boot))
        *(.text .text.*)
    }
    .rodata : { *(.rodata .rodata.*) }
    .data : { *(.data .data.*) }

    .bss (NOLOAD) : ALIGN(16) {
        __bss_start = .;
        *(.bss .bss.*)
        *(COMMON)
        . = ALIGN(16);
        __bss_end = .;
    }

    . = ALIGN(16);
    . += 0x10000;
    __stack_top = .;

    /DISCARD/ : { *(.eh_frame .eh_frame_hdr) }
}
```

**What happens:**

1. `ENTRY(_start)` writes the address of `_start` into the ELF header as the entry point.
2. `.` is the **location counter**: "the address we are at now". `. = 0x40080000;` starts there.
3. The output section `.text` (code) collects `.text.boot` first, then all other code. `KEEP` stops the linker
   from removing it, and putting it first makes `_start` the very first byte at `0x4008_0000`.
4. `.rodata` (constant data such as strings) and `.data` (globals with a start value) follow.
5. `.bss` holds globals that start as zero. `NOLOAD` means: it takes no bytes in the file. The script
   defines two symbols, `__bss_start` and `__bss_end`, so that the boot code knows what to fill with zeros.
6. After the BSS we skip 64 KiB (`0x10000`) and define `__stack_top`. The stack grows **down** on arm64, so
   the area below `__stack_top` is our stack.
7. `/DISCARD/` throws away unwinding tables (`.eh_frame`), which a kernel with `panic = "abort"` never uses.

```text
0x4008_0000  +-------------------+  <- _start (from .text.boot)
             | .text   code      |
             | .rodata constants |
             | .data   globals   |
             | .bss    zeros     |  __bss_start .. __bss_end
             +-------------------+
             |   64 KiB stack    |  grows downwards
             +-------------------+  <- __stack_top
```

## 4. The first instructions, in assembly

Rust code needs a stack from its very first instruction (you saw `str x30, [sp, ...]` in Lesson 22.2). Nobody
has set `sp` yet, so the first few instructions must be assembly. Create `src/boot.s`:

```text
.section .text.boot
.global _start
_start:
    // Only CPU core 0 continues; any other core sleeps forever.
    mrs     x1, mpidr_el1
    and     x1, x1, #0xff
    cbnz    x1, park

    // Set the stack pointer to the top of our stack area.
    ldr     x1, =__stack_top
    mov     sp, x1

    // Fill the .bss section with zeros.
    ldr     x1, =__bss_start
    ldr     x2, =__bss_end
zero_bss:
    cmp     x1, x2
    b.hs    bss_done
    str     xzr, [x1], #8
    b       zero_bss
bss_done:

    bl      kernel_main

park:
    wfe
    b       park
```

**What happens:**

1. `.section .text.boot` puts this code into the section that the linker script places first.
2. `mrs x1, mpidr_el1` reads a system register that holds the number of this CPU core. On a machine with
   several cores, all of them may start here; only core 0 continues (`cbnz` jumps to `park` if the number is
   not zero). QEMU starts one core by default, but the check makes the code safe.
3. `ldr x1, =__stack_top` loads the 64-bit address of the linker symbol; `mov sp, x1` makes it the stack pointer.
4. The loop writes the zero register `xzr` to every 8 bytes from `__bss_start` to `__bss_end`.
   `str xzr, [x1], #8` stores and then adds 8 to `x1`. Rust expects a `static` with value `0` to really be zero.
5. `bl kernel_main` calls our Rust function. It never returns, but if it did, the core would go to `park`.
6. `wfe` ("wait for event") puts the core to sleep until something happens; the loop sends it back to sleep.

The `_start` of Lesson 22.2 is gone: the entry point is now in assembly, and Rust starts at `kernel_main`.

## 5. Talking to the UART

A device on `virt` is a set of **registers at fixed addresses**. For the PL011 UART at `0x0900_0000` you need
two of them:

| Offset | Register | Use |
|---|---|---|
| `0x00` | data register | Writing a byte here sends it over the serial line. |
| `0x18` | flag register | Bit 5 (`TXFF`) is 1 while the transmit queue is full; then you must wait. |

Writing to such an address must be a **volatile** write. A normal write to memory can be removed by the
optimizer if nobody reads the value afterwards, or merged with the next write. For a device each access is an
action ("send this byte"), so it must happen exactly as written: `core::ptr::write_volatile` and
`read_volatile` guarantee that.

Create `src/uart.rs`:

```rust
//! A tiny driver for the PL011 UART of the QEMU `virt` machine.

use core::fmt;
use core::ptr::{read_volatile, write_volatile};

const UART_BASE: usize = 0x0900_0000;
const DATA: usize = 0x00; // data register: write a byte here to send it
const FLAGS: usize = 0x18; // flag register
const TX_FULL: u32 = 1 << 5; // flag bit: the transmit queue is full

pub struct Uart;

impl Uart {
    pub fn write_byte(&mut self, byte: u8) {
        let flags = (UART_BASE + FLAGS) as *const u32;
        let data = (UART_BASE + DATA) as *mut u32;
        // SAFETY: on the QEMU `virt` machine these addresses are the registers
        // of the PL011 UART, and volatile access is how device registers are used.
        unsafe {
            while read_volatile(flags) & TX_FULL != 0 {}
            write_volatile(data, byte as u32);
        }
    }
}

impl fmt::Write for Uart {
    fn write_str(&mut self, s: &str) -> fmt::Result {
        for byte in s.bytes() {
            if byte == b'\n' {
                self.write_byte(b'\r');
            }
            self.write_byte(byte);
        }
        Ok(())
    }
}

#[macro_export]
macro_rules! print {
    ($($arg:tt)*) => {{
        use core::fmt::Write as _;
        let _ = write!($crate::uart::Uart, $($arg)*);
    }};
}

#[macro_export]
macro_rules! println {
    () => { $crate::print!("\n") };
    ($($arg:tt)*) => {{
        $crate::print!($($arg)*);
        $crate::print!("\n");
    }};
}
```

**What happens:**

1. `Uart` is a struct with no fields: the device has only one instance and all its state is in the hardware.
2. `write_byte` waits while the "transmit full" bit is set, then writes the byte. Turning a number into a
   pointer (`as *mut u32`) is safe; using it is `unsafe`, and the `// SAFETY:` comment says why it is fine
   here (Lesson 06.6).
3. `fmt::Write` is the trait from `core` that formatting needs: implement `write_str`, and you get
   `write!(uart, "...{}", x)` with all the formatting of `println!`. No heap is needed.
4. A serial terminal needs `\r` (carriage return: go to the start of the line) before `\n` (new line).
5. `print!` and `println!` are our own macros (Lesson 06.7) with the same names as in `std`. `#[macro_export]`
   makes them usable everywhere in the crate; `$crate::uart::Uart` finds the driver from any module.

> ⚡ **Performance:** QEMU's UART never fills up, so the wait loop ends at once. A real UART sends about 11,500
> bytes per second at 115200 baud: a kernel that prints a lot spends real time in this loop. Real kernels use a
> buffer and an interrupt that says "ready for more".

## 6. Switching off, and the first boot

When the kernel is done, it should switch the machine off, so that QEMU exits. Arm machines do this through
**PSCI**, a small standard interface to the firmware. On `virt`, QEMU itself answers PSCI calls that the kernel
makes with the `hvc` (hypervisor call) instruction. Create `src/power.rs`:

```rust
//! Turning the machine off with PSCI (Power State Coordination Interface).

const PSCI_SYSTEM_OFF: u64 = 0x8400_0008;

pub fn shutdown() -> ! {
    // SAFETY: on QEMU `virt` the firmware interface answers `hvc #0`;
    // SYSTEM_OFF never returns.
    unsafe {
        core::arch::asm!("hvc #0", in("x0") PSCI_SYSTEM_OFF, options(noreturn));
    }
}
```

`asm!` puts one instruction into the function (Lesson 06.6 showed `unsafe`; the Rust Reference describes
`asm!` in detail). `in("x0") PSCI_SYSTEM_OFF` places the function number `0x8400_0008` ("system off") into
register `x0` before the instruction. `options(noreturn)` promises that execution never continues after it.

Now replace `src/main.rs`:

```rust
#![no_std]
#![no_main]

mod power;
mod uart;

use core::panic::PanicInfo;

core::arch::global_asm!(include_str!("boot.s"));

#[unsafe(no_mangle)]
pub extern "C" fn kernel_main() -> ! {
    println!("Hello from p22-kernel!");
    let answer = 6 * 7;
    println!("6 * 7 = {answer}");
    println!("kernel_main is at {:#x}", kernel_main as *const () as usize);
    power::shutdown();
}

#[panic_handler]
fn panic(info: &PanicInfo) -> ! {
    println!("KERNEL PANIC: {info}");
    power::shutdown();
}
```

**What happens:**

1. `global_asm!(include_str!("boot.s"))` reads `src/boot.s` at compile time and assembles it into the
   program. Now the symbol `_start` comes from there.
2. `kernel_main` is called by the assembly, so it needs a fixed name (`no_mangle`) and the C calling
   convention. It prints three lines with our `println!`, including its own address, then switches off.
3. The panic handler now prints the message: `PanicInfo` implements `Display`, so `{info}` shows the text and
   the source location.

Boot it:

```bash
cargo run
```

```text
   Compiling p22-kernel v0.1.0 (/Users/student/projects/rust/code/p22-kernel)
    Finished `dev` profile [unoptimized + debuginfo] target(s) in 0.83s
     Running `/Users/student/projects/rust/code/p22-kernel/./qemu.sh target/aarch64-unknown-none-softfloat/debug/p22-kernel`
Hello from p22-kernel!
6 * 7 = 42
kernel_main is at 0x40080c88
```

Your own kernel booted on an (emulated) machine and printed text. QEMU exited by itself because the kernel
switched the machine off. Check where things went with the tools from Lesson 22.2:

```bash
"$LLVM_BIN/llvm-readobj" --file-headers target/aarch64-unknown-none-softfloat/debug/p22-kernel | grep Entry
"$LLVM_BIN/llvm-nm" -n target/aarch64-unknown-none-softfloat/debug/p22-kernel | grep -E '_start|kernel_main$|__bss|__stack'
```

```text
  Entry: 0x40080000
0000000040080000 T _start
0000000040080c88 T kernel_main
0000000040082a90 B __bss_end
0000000040082a90 B __bss_start
0000000040092a90 B __stack_top
```

The entry point is at `0x4008_0000`, as the linker script said. The BSS is empty for now (start = end), and the
stack top is 64 KiB (`0x10000`) above it.

## 7. A panic in the kernel

Add two lines before `power::shutdown();` in `kernel_main`, to read past the end of an array:

```rust
    let values = [1, 2, 3];
    let index = answer / 10;
    println!("value: {}", values[index]);
```

```text
Hello from p22-kernel!
6 * 7 = 42
kernel_main is at 0x40080cd0
KERNEL PANIC: panicked at src/main.rs:19:27:
index out of bounds: the len is 3 but the index is 4
```

The bounds check of Rust works in the kernel exactly as in normal programs, and now you see the message and the
line. Without it, reading `values[4]` would silently return whatever lies after the array. Remove the two lines
again.

> 🔒 **Security:** In a kernel, a silent out-of-bounds read can leak another program's data, and a write can
> take over the machine. A panic that stops the kernel is much better than continuing with broken memory.

> 🧪 **Testing:** The panic output is an easy test oracle: a script can run `cargo run`, capture the output, and
> check for `KERNEL PANIC`. Lesson 22.5 shows how to turn this into automated tests in QEMU.

## Common mistakes

> ⚠️ **Common mistake:** The linker does not get the linker script (a typo in `rustflags`, or `linker.ld` in
> the wrong folder). The boot code then refers to symbols that nobody defines:
>
> ```text
> rust-lld: error: undefined symbol: __stack_top
> rust-lld: error: undefined symbol: __bss_end
> ```
>
> `-Tlinker.ld` is read relative to the package folder; keep `linker.ld` next to `Cargo.toml`.

> ⚠️ **Common mistake:** Leaving out `-nic none`. QEMU then adds a network card and looks for its boot file:
>
> ```text
> qemu-system-aarch64: failed to find romfile "efi-virtio.rom"
> ```

> ⚠️ **Common mistake:** A kernel that never calls `shutdown` (for example an endless loop) keeps QEMU running.
> It is not frozen: press **Ctrl-A**, then **X**, to quit QEMU.

> ⚠️ **Common mistake:** Writing to a device with a plain `*ptr = value`. It may work in a debug build and
> disappear in a release build, because the optimizer is allowed to remove a write that is never read. Always
> use `read_volatile` and `write_volatile` for device registers.

## Try it yourself

1. Build the image, create `qemu.sh`, `linker.ld`, `src/boot.s`, `src/uart.rs`, `src/power.rs`, and boot with
   `cargo run`.
2. Change the text of the first `println!` and boot again.
3. Remove `-cpu cortex-a72` from `qemu.sh` and boot. QEMU refuses with "The image is from incompatible
   architecture": the default CPU of `virt` is 32-bit. Put the option back.
4. Run `cargo host-test` and `cargo clippy`, then commit: `git commit -am "p22-kernel: boot in QEMU"`
   (add the new files with `git add` first).

## Exercises

### Exercise 1 — How big is the kernel in memory? (easy)

Print the address of `__bss_end` and of `__stack_top` from `kernel_main`, and how many KiB the kernel uses from
`0x4008_0000` to the stack top.

<details>
<summary>Hint</summary>

Declare the linker symbols in Rust as `unsafe extern "C" { static __bss_end: u8; }`. You never read their value;
you only take their address with `&raw const __bss_end`.

</details>

<details>
<summary>Answer</summary>

Add this to `kernel_main` before `power::shutdown();`:

```rust
    unsafe extern "C" {
        static __bss_end: u8;
        static __stack_top: u8;
    }
    let bss_end = &raw const __bss_end as usize;
    let stack_top = &raw const __stack_top as usize;
    println!("kernel data ends at {bss_end:#x}, stack top is {stack_top:#x}");
    println!("the kernel uses {} KiB", (stack_top - 0x4008_0000) / 1024);
```

```text
kernel_main is at 0x40080ce4
kernel data ends at 0x40082cf0, stack top is 0x40092cf0
the kernel uses 75 KiB
```

Why it works: a linker symbol is only an address; there is no variable behind it. `&raw const` takes the address
without creating a reference to memory that is not really a `u8`. 64 KiB of the 75 KiB are the stack.

</details>

### Exercise 2 — Test the newline rule on your Mac (medium)

Move the "send `\r` before every `\n`" rule out of the driver into `src/lib.rs`, as a pure function
`pub fn serial_bytes(text: &str, emit: impl FnMut(u8))`, and write two tests for it. The driver calls it.

<details>
<summary>Hint</summary>

In a test, `emit` can be a closure that pushes into a `Vec<u8>` (the tests have `std`). `b"a\r\nb"` is a byte
string you can compare with the vector.

</details>

<details>
<summary>Answer</summary>

In `src/lib.rs`:

```rust
/// Calls `emit` for every byte of `text`, sending "\r\n" for each "\n",
/// because a serial terminal needs both to start a new line.
pub fn serial_bytes(text: &str, mut emit: impl FnMut(u8)) {
    for byte in text.bytes() {
        if byte == b'\n' {
            emit(b'\r');
        }
        emit(byte);
    }
}
```

Inside `mod tests`:

```rust
    #[test]
    fn newline_becomes_carriage_return_and_newline() {
        let mut sent = Vec::new();
        serial_bytes("a\nb", |byte| sent.push(byte));
        assert_eq!(sent, b"a\r\nb");
    }

    #[test]
    fn text_without_newline_is_unchanged() {
        let mut sent = Vec::new();
        serial_bytes("ok", |byte| sent.push(byte));
        assert_eq!(sent, b"ok");
    }
```

And in `src/uart.rs` the body of `write_str` becomes:

```rust
        p22_kernel::serial_bytes(s, |byte| self.write_byte(byte));
        Ok(())
```

```text
test result: ok. 4 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

Why it works: the closure decides where the bytes go — into the UART in the kernel, into a `Vec` in the test. The
logic is tested once, on the Mac, and the kernel prints exactly as before.

</details>

### Exercise 3 — How long did the boot take? (hard)

Every arm64 CPU has a system counter: `CNTPCT_EL0` counts ticks since power-on and `CNTFRQ_EL0` says how many ticks
there are per second. Read both with `asm!` and print the time since power-on in microseconds.

<details>
<summary>Hint</summary>

`core::arch::asm!("mrs {}, cntpct_el0", out(reg) ticks);` reads a system register into a Rust variable. Multiply
before you divide, or the result is 0.

</details>

<details>
<summary>Answer</summary>

```rust
    let (ticks, freq): (u64, u64);
    // SAFETY: reading the counter and its frequency has no side effects.
    unsafe {
        core::arch::asm!("mrs {}, cntpct_el0", out(reg) ticks);
        core::arch::asm!("mrs {}, cntfrq_el0", out(reg) freq);
    }
    let micros = ticks * 1_000_000 / freq;
    println!("counter: {ticks} ticks at {freq} Hz = {micros} us since power-on");
```

```text
counter: 142455 ticks at 62500000 Hz = 2279 us since power-on
```

Why it works: `mrs` copies a system register into a general register, and `out(reg) ticks` tells the compiler to
pick a register and store it into `ticks` afterwards. QEMU's counter runs at 62.5 MHz; booting to this line took
about 2 ms (your number will differ on every run).

</details>

## Quiz

1. Why is the kernel linked at `0x4008_0000` and not at `0x0`?

<details>
<summary>Answer</summary>

On `virt`, RAM starts at `0x4000_0000`; below that are flash and devices. QEMU puts the DTB at the start of RAM, so
the kernel goes a little higher.

</details>

2. Which two jobs must the assembly code do before the first Rust function runs?

<details>
<summary>Answer</summary>

Set the stack pointer (Rust code uses the stack at once) and fill the BSS with zeros (Rust expects zero-initialised
statics to be zero). It also parks all cores except core 0.

</details>

3. Why must device registers be accessed with `read_volatile` / `write_volatile`?

<details>
<summary>Answer</summary>

Each access to a device is an action. The compiler may remove, repeat or merge normal memory accesses; volatile
accesses happen exactly as written.

</details>

4. What does the `runner` setting in `.cargo/config.toml` do?

<details>
<summary>Answer</summary>

`cargo run` starts the runner program (`./qemu.sh`) with the path of the built binary as its argument, instead of
starting the binary itself.

</details>

5. How does the kernel make QEMU exit?

<details>
<summary>Answer</summary>

It makes a PSCI `SYSTEM_OFF` call: function number `0x8400_0008` in `x0`, then `hvc #0`. QEMU plays the firmware,
switches the machine off and exits.

</details>

## Summary

- `virt` has RAM at `0x4000_0000` and devices below it (UART at `0x0900_0000`); QEMU loads an ELF kernel and jumps
  to its entry point.
- QEMU runs in a container; `qemu.sh` plus the Cargo `runner` make `cargo run` boot the kernel.
- The linker script places `.text.boot` first at `0x4008_0000` and defines `__bss_start`, `__bss_end` and
  `__stack_top`.
- The boot assembly parks other cores, sets `sp`, clears the BSS and calls `kernel_main`.
- Devices are memory-mapped registers; use volatile accesses. `fmt::Write` on a tiny UART driver gives you
  `println!`.
- The panic handler prints `PanicInfo`; PSCI `SYSTEM_OFF` through `hvc #0` ends the run.

## Sources and further reading

- [QEMU — 'virt' generic virtual platform](https://www.qemu.org/docs/master/system/arm/virt.html) — the machine
  used in this track, its devices and its boot rules.
- [Arm PrimeCell UART (PL011) Technical Reference Manual](https://developer.arm.com/documentation/ddi0183/latest/) —
  all registers of the UART.
- [Arm Power State Coordination Interface (PSCI)](https://developer.arm.com/documentation/den0022/latest/) — the
  specification of `SYSTEM_OFF` and the other calls.
- [GNU ld — Linker Scripts](https://sourceware.org/binutils/docs/ld/Scripts.html) — the linker script language,
  which `rust-lld` understands too.
- [The Rust Reference — Inline assembly](https://doc.rust-lang.org/reference/inline-assembly.html) — `asm!` and
  `global_asm!` in detail.
- [`core::ptr::write_volatile`](https://doc.rust-lang.org/core/ptr/fn.write_volatile.html) — the exact rules of
  volatile accesses.

---

[🏠 Start](../START-HERE.md) · [← 22.2 OS track: a freestanding Rust binary](02-os-track-a-freestanding-binary.md) · [📋 Phase 22](README.md) · [22.4 OS track: interrupts and paging →](04-os-track-interrupts-and-paging.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
