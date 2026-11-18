<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 22.9 eBPF track: project and next steps](09-ebpf-track-project-and-next-steps.md) · [📋 Phase 22](README.md) · [22.11 Embedded track: GPIO and embedded-hal →](11-embedded-track-gpio-and-embedded-hal.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/22-specializations/10-embedded-track-microcontrollers-and-no-std.md) <!-- nav -->

# Lesson 22.10 — Embedded track: microcontrollers and no_std

> **Phase 22 — Specializations** · Lesson 10 of 30 · ⏱ about 3 hours (2 sittings)

> 🧭 **Not your track?** This lesson starts the embedded track (Lessons 22.10–22.13). If you chose
> another track, skip to [Lesson 22.14 — Databases track: how query engines work](14-databases-track-how-query-engines-work.md)
> or go straight to the [checkpoint, Lesson 22.30](30-checkpoint-your-specialization.md).

## What you will learn

- What a microcontroller is, and how a program runs on a chip that has no operating system.
- Why Rust fits embedded work well, and which board this track uses (and why).
- What `#![no_std]`, `#![no_main]` and a panic handler are, and what `core` gives you without `std`.
- How to cross-compile: target triples, `rustup target add`, `.cargo/config.toml`, `memory.x`.
- How to build your first firmware that switches on an LED (Light-Emitting Diode, a small lamp), and how it gets onto the board.

## Before you start

- You finished [Lesson 22.1 — Choosing a specialization](01-choosing-a-specialization.md) and picked this track.
- You know `unsafe` from [Lesson 06.6](../06-rust-advanced/06-unsafe-rust-and-miri.md), bits and
  bit masks from [Lesson 12.4](../12-computer-systems/04-bitwise-operations.md), and how the linker
  works from [Lesson 12.12 — Linking and loading programs](../12-computer-systems/12-linking-and-loading.md).
- If you did the OS (operating system) track, [Lesson 22.2](02-os-track-a-freestanding-binary.md) already showed you a
  freestanding binary. Here you see the same idea on a real chip; read Sections 4–5 quickly.
- No hardware is needed for this lesson. Everything except the last step (putting the program on the
  board) works on your Mac alone.

**Where the code goes.** Each embedded lesson uses its own package in `code/p22-<name>`, for example
`code/p22-hello-led`. The track project in Lesson 22.13 is `code/p22-morse`.

## New words

| Word | Meaning |
|---|---|
| **embedded system** | A computer that is built into a device to do one job: a washing machine controller, a fitness band, a sensor in a factory. |
| **MCU** (Microcontroller Unit) | A microcontroller: one small chip that holds a CPU (Central Processing Unit) core, flash memory, RAM (Random Access Memory) and peripherals. |
| **firmware** | The program that runs on a microcontroller. It is stored in the chip's flash memory. |
| **flash memory** | Memory that keeps its content without power. The firmware lives there. |
| **SRAM** (Static Random Access Memory) | The fast working memory of a microcontroller. It is empty when the power goes off. |
| **peripheral** | A hardware block inside the chip that does one job: pins, timers, serial ports, an analog-to-digital converter. |
| **register** (peripheral) | A small piece of memory inside a peripheral. Writing a number to it changes what the hardware does. |
| **GPIO** (General-Purpose Input/Output) | A pin that your program can switch on or off, or read. Lesson 22.11 covers it. |
| **`no_std`** | A crate attribute that says "do not use the standard library". The crate can use only `core` (and `alloc` if you add an allocator). |
| **`core`** | The part of the Rust standard library that needs no operating system: `Option`, `Result`, iterators, slices, integer math, `fmt`. |
| **panic handler** | The function that runs when a `no_std` program panics. You must provide exactly one. |
| **cross-compilation** | Building a program on one computer (your Mac) for a different kind of computer (the microcontroller). |
| **target triple** | The name of a platform that `rustc` builds for, for example `thumbv8m.main-none-eabihf`. |
| **Arm Cortex-M** | A family of small CPU cores designed by Arm for microcontrollers. |
| **linker script** | A text file that tells the linker where flash and RAM are and where to put each part of the program. |
| **vector table** | A table at the start of flash. Its first entries tell the CPU the start address of the stack and the address of the first function to run. |
| **debug probe** | A small adapter between your computer and the board. It writes the firmware into flash and lets you debug. |
| **SWD** (Serial Wire Debug) | The two-wire connection that Arm chips use for debugging and flashing. |
| **ELF** (Executable and Linkable Format) | The file format of the built firmware before it is written to flash. |

## 1. What a microcontroller is

Your Mac has a big CPU, gigabytes of RAM, a disk and an operating system. When you start a program,
the operating system loads it, gives it memory, and cleans up after it.

A **microcontroller** is a whole tiny computer on one chip. It has a CPU core, a little flash memory
for the program, a little SRAM for data, and many **peripherals**: pins, timers, serial ports. There
is usually **no operating system**. Your program is the only program on the chip.

| | Your Mac (M4) | Raspberry Pi Pico 2 (RP2350 chip) |
|---|---|---|
| CPU | 10+ cores, about 4 GHz | 2 small cores, 150 MHz |
| Working memory | 16–24 GB | 520 KB of SRAM |
| Program storage | a disk of hundreds of GB | 4 MB of flash |
| Operating system | macOS | none: your firmware owns the chip |
| Price | expensive | about 5 US dollars |

Three things change when there is no operating system:

1. **Nobody loads your program.** The chip starts it directly from flash when power comes on.
2. **Your program never ends.** There is nobody to return to. `main` has the type `fn() -> !`
   ("never returns"), and the program usually ends in an endless loop.
3. **Hardware is just memory addresses.** A peripheral register has a fixed address. To switch a
   pin on, the program writes a number to that address. This is called **memory-mapped I/O**.

```text
Address space of the RP2350 (simplified)

0x1000_0000  +------------------+
             | flash (program)  |  <- your firmware lives here
0x2000_0000  +------------------+
             | SRAM (data)      |  <- stack and variables
0x4000_0000  +------------------+
             | peripherals      |  <- timers, serial ports, ...
0xd000_0000  +------------------+
             | SIO (fast GPIO)  |  <- write here to switch pins
             +------------------+
```

## 2. Why Rust for embedded work

- **No runtime and no garbage collector.** A Rust program needs nothing from an operating system.
  The code you write is the code that runs, and its memory use is predictable.
- **Memory safety on a chip without protection.** A microcontroller usually cannot stop a bad
  pointer: writing to a wrong address silently breaks a peripheral or other data. The borrow checker
  catches most such bugs at compile time.
- **Ownership models hardware.** A pin is a value. When you give `PIN_25` to an LED driver, it is
  *moved*, and no other part of the program can use the same pin by mistake.
- **Shared interfaces.** The `embedded-hal` crate defines traits like "an output pin" and "a delay".
  A driver written against these traits works on any chip (Lesson 22.11).
- **Modern tools.** Cargo builds for the chip, `probe-rs` flashes and debugs it, and Embassy gives you
  `async`/`await` on bare metal (Lesson 22.12).

## 3. The board for this track: Raspberry Pi Pico 2

This track uses the **Raspberry Pi Pico 2**, a board with the **RP2350** chip. Reasons:

- **Cheap and easy to buy.** About 5 US dollars, sold by Raspberry Pi resellers in most countries.
- **Very good Rust support.** The Embassy project has a maintained HAL (Hardware Abstraction Layer)
  for it, `embassy-rp`, and `probe-rs` supports the chip.
- **Excellent official documents.** The RP2350 datasheet explains every register clearly.
- **An official, cheap debug probe.** The Raspberry Pi Debug Probe (about 12 US dollars) plugs into
  the Pico 2 with one cable. Flashing and logging then work with `cargo run`.
- **A modern Arm core.** The RP2350 has two Arm Cortex-M33 cores (it also has two RISC-V cores, an open instruction set, which
  this track does not use; RISC means Reduced Instruction Set Computer).

Shopping list for Lessons 22.11–22.13: a Pico 2 (the "Pico 2 H" has pins already soldered), a
Raspberry Pi Debug Probe, a breadboard, two LEDs, two 330 Ω resistors, one push button, jumper wires
and a USB (Universal Serial Bus) cable.

> 💡 **Tip:** if you already own an ESP32-C3 board, the `esp-hal` crate supports it well. The ideas of
> this track are the same, but crate names, target and pin numbers differ, so follow the `esp-hal`
> book for the details.

## 4. Without the standard library: `std`, `core` and `alloc`

The Rust standard library has three layers:

```text
+-------------------------------------------------------------+
| std   : files, threads, network, println!, time, HashMap    |  needs an OS
+-------------------------------------------------------------+
| alloc : Box, Vec, String, Rc                                |  needs a heap allocator
+-------------------------------------------------------------+
| core  : Option, Result, iterators, slices, &str, integers,  |  needs nothing
|         fmt, atomics, panic!                                |
+-------------------------------------------------------------+
```

`std` asks the operating system for files, threads and output, so it cannot exist on a bare chip. The
attribute `#![no_std]` at the top of a crate says: "link only `core`". You keep almost all of the
language: structs, enums, traits, generics, closures, iterators, pattern matching. You lose
`println!`, `Vec`, `String` and `HashMap` (`alloc` brings `Vec` and `String` back if you add a heap
allocator, but most firmware avoids the heap).

A great habit: put your **logic** in a small `no_std` library and test it **on your Mac**. Tests use
the normal test harness, because `cargo test` builds for the Mac.

```bash
cd ~/projects/rust
cargo new --lib code/p22-sensor
cd code/p22-sensor
```

The RP2350 has a temperature sensor. Its 12-bit reading (0 to 4095 stands for 0 to 3.3 volts) turns
into degrees with the datasheet formula *T = 27 − (V − 0.706) / 0.001721*. Firmware avoids slow
floating-point work in hot code, so we compute in **millidegrees** with integers. Put this into
`src/lib.rs`:

```rust
#![no_std]

/// Converts a raw 12-bit reading of the RP2350 temperature sensor
/// into thousandths of a degree Celsius (millidegrees).
pub fn temperature_millicelsius(raw: u16) -> i32 {
    // 12 bits: 0..=4095 stands for 0..=3300 millivolts.
    let microvolts = (raw as i64 * 3_300_000 / 4095) as i32;
    // Datasheet formula: T = 27 - (V - 0.706) / 0.001721
    27_000 - (microvolts - 706_000) * 1000 / 1721
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn about_27_degrees_at_706_millivolts() {
        // 706 mV is about 876 on the 12-bit scale.
        let t = temperature_millicelsius(876);
        assert!((26_000..28_000).contains(&t), "got {t}");
    }

    #[test]
    fn warmer_chip_gives_lower_voltage() {
        assert!(temperature_millicelsius(850) > temperature_millicelsius(876));
    }
}
```

**What happens:**

1. `#![no_std]` (with `!`, so it applies to the whole crate) removes `std`. `assert!`, ranges and
   integer math all come from `core`, so the code still compiles.
2. The multiplication is done in `i64` on purpose (in 32 bits it overflows, see "Common mistakes").
   Working in microvolts and millidegrees keeps everything an integer.
3. `cargo test` compiles the library for your Mac and links the test harness, so the tests run here.

```text
running 2 tests
test tests::about_27_degrees_at_706_millivolts ... ok
test tests::warmer_chip_gives_lower_voltage ... ok

test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

> 🧪 **Testing:** most firmware bugs are logic bugs: wrong formulas, wrong state changes, off-by-one
> timing. Keep that logic in `no_std` functions without hardware types, and test it on your Mac in
> milliseconds. Only the thin layer that touches pins needs the board.

## 5. A program for a chip: `no_main` and the panic handler

Let us see what the compiler says when we build an ordinary program for the chip. First install the
**target** (the standard library parts for this kind of CPU). The name is explained in Section 6.

```bash
rustup target add thumbv8m.main-none-eabihf
```

```text
info: downloading component rust-std
```

Now a fresh `cargo new` project, built for the microcontroller:

```bash
cd ~/projects/rust
cargo new code/p22-std-on-mcu
cd code/p22-std-on-mcu
cargo build --target thumbv8m.main-none-eabihf
```

```text
error[E0463]: can't find crate for `std`
  |
  = note: the `thumbv8m.main-none-eabihf` target may not support the standard library
  = note: `std` is required by `p22_std_on_mcu` because it does not declare `#![no_std]`

error: cannot resolve a prelude import

error: cannot find macro `println` in this scope
 --> src/main.rs:2:5
  |
2 |     println!("Hello, world!");
  |     ^^^^^^^

error: `#[panic_handler]` function required, but not found

For more information about this error, try `rustc --explain E0463`.
```

The compiler tells us three things: there is no `std` for this target, so there is no `println!`, and
we need a **panic handler**. On a normal computer, `std` handles a panic: it prints the message and
stops the thread. Without `std`, you decide what a panic does. Replace `src/main.rs`:

```rust
#![no_std]
#![no_main]

use core::panic::PanicInfo;

#[panic_handler]
fn panic(_info: &PanicInfo) -> ! {
    loop {}
}
```

**What happens:**

1. `#![no_main]` says "there is no normal `main`". A normal `main` is called by start-up code from
   `std` and the operating system; on the chip, something else must start the program.
2. `#[panic_handler]` marks the one function that runs on a panic. It gets a `PanicInfo` (message and
   location) and must never return, so its type is `-> !`. Here it just loops forever.
3. The program builds, but the linker warns that it does not know where the program starts:

```text
warning: linker stderr: rust-lld: cannot find entry symbol _start; not setting start address
  |
  = note: `#[warn(linker_messages)]` on by default

warning: `p22-std-on-mcu` (bin "p22-std-on-mcu") generated 1 warning
    Finished `dev` profile [unoptimized + debuginfo] target(s) in 0.27s
```

This is the last missing piece: start-up code. On the RP2350, the very first code that runs is the
**boot ROM** (ROM: Read-Only Memory), a small program built into the chip at the factory; it then
hands control to your program. When an Arm Cortex-M chip powers on, it reads the
**vector table** at the start of the program: entry 0 is the initial stack pointer, entry 1 is the
address of the **reset handler**, the first function to run. The crate `cortex-m-rt` (runtime)
provides the vector table and a reset handler that prepares RAM and then calls your function marked
`#[entry]`.

```text
power on
   |
   v
boot ROM inside the RP2350 ---- finds the IMAGE_DEF block in the first 4 KiB of flash
   |
   v
vector table (cortex-m-rt) ---- stack pointer = end of RAM, reset handler address
   |
   v
reset handler (cortex-m-rt) --- copies initial values of statics into RAM, zeroes the rest
   |
   v
your #[entry] fn main() -> !  - runs forever
```

You can delete `code/p22-std-on-mcu` now; the real firmware comes next.

## 6. Cross-compiling: target triple, config and memory map

The target name `thumbv8m.main-none-eabihf` describes the platform in parts:

| Part | Meaning |
|---|---|
| `thumbv8m.main` | The Arm "Thumb" instruction set, version 8-M Mainline: the Cortex-M33 core of the RP2350. |
| `none` | No operating system. |
| `eabihf` | EABI (Embedded Application Binary Interface: the rules for calling functions and laying out data) with **h**ard **f**loat: the chip has a floating-point unit. |

Typing `--target` every time is tiring, so a firmware project has three extra files:

```text
code/p22-hello-led/
├── .cargo/
│   └── config.toml   ← default target and how to run the firmware
├── build.rs          ← copies memory.x for the linker
├── memory.x          ← where flash and RAM are
├── Cargo.toml
└── src/
    └── main.rs
```

Create the package and the folder:

```bash
cd ~/projects/rust
cargo new code/p22-hello-led
cd code/p22-hello-led
mkdir .cargo
```

`.cargo/config.toml`:

```toml
[build]
target = "thumbv8m.main-none-eabihf"

[target."thumbv8m.main-none-eabihf"]
runner = "probe-rs run --chip RP235x"
```

`[build] target` makes every `cargo build` use the chip target. `runner` is the command that
`cargo run` uses instead of starting the program on your Mac: it hands the built file to `probe-rs`,
which writes it to the chip (Section 8). The quotes around the target name matter: the name contains
a dot, and without quotes TOML (the config file format of Cargo) reads it as two nested keys (Cargo warns that the key "will be ignored
or error in a future version").

`memory.x` is a **linker script** fragment. The linker script `link.x` from `cortex-m-rt` includes it
to learn the memory map:

```text
MEMORY {
    FLASH : ORIGIN = 0x10000000, LENGTH = 4096K
    RAM   : ORIGIN = 0x20000000, LENGTH = 512K
}

SECTIONS {
    /* The RP2350 boot ROM looks for this block in the first 4 KiB of flash. */
    .start_block : ALIGN(4)
    {
        KEEP(*(.start_block));
    } > FLASH
} INSERT AFTER .vector_table;

/* Program code starts after the boot block. */
_stext = ALIGN(ADDR(.start_block) + SIZEOF(.start_block), ALIGNOF(.text));
```

**What happens:**

1. `MEMORY` names two regions: 4 MiB of flash at address `0x1000_0000` and 512 KiB of SRAM at
   `0x2000_0000` (the numbers come from the RP2350 datasheet).
2. `.start_block` is placed right after the vector table. `embassy-rp` puts a small `IMAGE_DEF` block
   there; the boot ROM refuses to start a program without it.
3. `_stext` moves the program code behind that block. The Embassy examples for the RP2350 use a longer
   version of this file with optional extra sections for Raspberry Pi's `picotool`; for this track the
   short one produces exactly the same flash image.

`build.rs` (a **build script**: Cargo compiles and runs it before building your crate) copies
`memory.x` to a place where the linker looks, and passes the linker script:

```rust
use std::env;
use std::fs;
use std::path::PathBuf;

fn main() {
    // Copy memory.x next to the build output so the linker finds it.
    let out = PathBuf::from(env::var_os("OUT_DIR").unwrap());
    fs::write(out.join("memory.x"), include_bytes!("memory.x")).unwrap();
    println!("cargo:rustc-link-search={}", out.display());
    println!("cargo:rerun-if-changed=memory.x");

    // Ask the linker to use the linker script from cortex-m-rt.
    println!("cargo:rustc-link-arg-bins=--nmagic");
    println!("cargo:rustc-link-arg-bins=-Tlink.x");
}
```

The build script runs **on your Mac**, so it may use `std` and `println!`. Its `println!` lines are
instructions for Cargo, not output for you.

## 7. Your first firmware: switch on the LED

`Cargo.toml`:

```toml
[package]
name = "p22-hello-led"
version = "0.1.0"
edition = "2024"

[dependencies]
cortex-m = "0.7"
cortex-m-rt = "0.7"
embassy-rp = { version = "0.10", features = ["rp235xa", "critical-section-impl"] }

[profile.release]
debug = true
```

`embassy-rp` is the HAL for the RP2040 and RP2350 chips. The feature `rp235xa` selects the RP2350A
chip of the Pico 2, and `critical-section-impl` provides a small locking primitive the HAL needs.
`debug = true` keeps debug information in release builds; it is not written to flash, it only helps
the debugger.

`src/main.rs`:

```rust
#![no_std]
#![no_main]

use core::panic::PanicInfo;
use cortex_m_rt::entry;
use embassy_rp::gpio::{Level, Output};

#[entry]
fn main() -> ! {
    // Take ownership of all peripherals of the chip.
    let p = embassy_rp::init(Default::default());
    // GPIO 25 is wired to the green LED on the Pico 2 board.
    let _led = Output::new(p.PIN_25, Level::High);

    loop {
        // Nothing else to do: keep the LED on forever.
        cortex_m::asm::wfi();
    }
}

#[panic_handler]
fn panic(_info: &PanicInfo) -> ! {
    loop {}
}
```

**What happens:**

1. `#[entry]` from `cortex-m-rt` makes `main` the function that the reset handler calls. Its type must
   be `fn() -> !`.
2. `embassy_rp::init` sets up the clocks and returns a struct `p` with one field per peripheral:
   `p.PIN_0` … `p.PIN_29`, timers, and so on. You get it exactly once.
3. `Output::new(p.PIN_25, Level::High)` *moves* pin 25 into an output driver and sets it high
   (3.3 volts), so the LED lights up. The name `_led` keeps the driver alive without an
   "unused variable" warning; if it were dropped, the pin would be reset.
4. `cortex_m::asm::wfi()` ("wait for interrupt") puts the core to sleep until something happens. It
   saves power compared with an empty busy loop.

```bash
cargo build --release
```

```text
   ...
   Compiling embassy-rp v0.10.0
   Compiling p22-hello-led v0.1.0 (/Users/you/projects/rust/code/p22-hello-led)
    Finished `release` profile [optimized + debuginfo] target(s) in 50.33s
warning: the following packages contain code that will be rejected by a future version of Rust: proc-macro-error2 v2.0.1
```

The warning is about a dependency of a dependency, not about your code; you can ignore it. The result
is a 32-bit Arm ELF file, `target/thumbv8m.main-none-eabihf/release/p22-hello-led`. To see how much
flash and RAM the firmware really uses, install `cargo-binutils` once:

```bash
rustup component add llvm-tools
cargo install --locked cargo-binutils
cargo size --release
```

```text
   text	   data	    bss	    dec	    hex	filename
   7104	      0	    284	   7388	   1cdc	p22-hello-led
```

`text` is code and constants (in flash), `data` is initial values of statics (in flash *and* RAM),
`bss` is statics that start as zero (in RAM). About 7 KB out of 4 MiB: there is a lot of room left.
`cargo size --release -- -A` lists every section with its address (shortened):

```text
section               size        addr
.vector_table          276  0x10000000
.start_block            20  0x10000114
.text                 4904  0x10000128
.rodata               1904  0x10001450
.bss                   284  0x20000000
.debug_info         673245         0x0
```

The vector table sits at the very start of flash (`0x1000_0000`), the boot block follows it, then the
code. The `.debug_*` sections have address 0: they stay on your Mac and are never written to flash.

> ⚡ **Performance:** always measure firmware size in **release** mode. A debug build of the same
> program is many times bigger and slower, and on a small chip it may not even fit.

## 8. Getting the firmware onto the board

You need hardware for this section. Without it, read along; the next lessons work the same way.

**With the Debug Probe (recommended).** Connect the probe's "D" cable to the 3-pin DEBUG connector of
the Pico 2 (SWCLK = clock, GND = ground, SWDIO = data: this is SWD), plug both into USB, and install the flashing tool once:

```bash
cargo install --locked probe-rs-tools
probe-rs list
```

With nothing connected, `probe-rs list` prints:

```text
No debug probes were found.
```

With the probe connected, it lists your Debug Probe. Now `cargo run` builds the
firmware and calls the runner from `.cargo/config.toml`:

```bash
cargo run --release
```

Without a probe, the build succeeds and the runner stops with `Error: No connected probes were found.`
With the board connected, the output looks like this (example; yours will look similar):

```text
    Finished `release` profile [optimized + debuginfo] target(s) in 0.05s
     Running `probe-rs run --chip RP235x target/thumbv8m.main-none-eabihf/release/p22-hello-led`
      Erasing ✔ 100% [####################]   8.00 KiB @  57.23 KiB/s (took 0s)
  Programming ✔ 100% [####################]   8.00 KiB @  38.42 KiB/s (took 0s)
     Finished in 0.71s
```

The green LED is on. `probe-rs run` keeps running and would show log messages (Lesson 22.12); press
Ctrl+C to stop it. The firmware stays in flash and starts again on every power-on.

**Without a probe.** Hold the BOOTSEL (boot select) button while you plug in USB. The Pico 2 appears as a USB drive,
and Raspberry Pi's `picotool` (`picotool load -u -v -x -t elf <file>`) writes the ELF file to it. This
works, but you get no logs and no debugger, so this track uses the probe.

> 🔒 **Security:** firmware in flash can usually be read back by anyone with a probe. Never put secrets
> such as Wi-Fi passwords or API (Application Programming Interface) keys into firmware for a device you give away, unless the chip's
> secure-boot and lock features (the RP2350 has them) are set up.

## Common mistakes

> ⚠️ **Common mistake: integer overflow in sensor math.** The first version of
> `temperature_millicelsius` used `raw as i32 * 3_300_000`. For `raw = 876` that is about 2.9 billion,
> more than `i32::MAX` (about 2.1 billion). The tests on the Mac caught it in debug mode:
>
> ```text
> thread 'tests::about_27_degrees_at_706_millivolts' (5817275) panicked at src/lib.rs:7:22:
> attempt to multiply with overflow
> ```
>
> In a release build on the chip it would have silently wrapped around and given a wrong temperature.
> Fix: do the multiplication in `i64`, or divide earlier. This is one more reason to test logic on
> the host.

> ⚠️ **Common mistake: no `memory.x`.** If `memory.x` is missing (or `build.rs` does not copy it),
> the linker stops:
>
> ```text
>   = note: rust-lld: error: .../cortex-m-rt/.../out/link.x:23: cannot find linker script memory.x
>           >>> INCLUDE memory.x
> ```
>
> Fix: put `memory.x` in the package root and use the `build.rs` from Section 6.

> ⚠️ **Common mistake: target not installed.** Building before `rustup target add` gives:
>
> ```text
> error[E0463]: can't find crate for `core`
>   |
>   = note: the `thumbv8m.main-none-eabihf` target may not be installed
>   = help: consider downloading the target with `rustup target add thumbv8m.main-none-eabihf`
> ```
>
> Fix: run the command from the help line.

## Try it yourself

1. In `code/p22-sensor`, add a test for `raw = 0` (0 volts). The formula gives `437227`, about
   437 °C. It is not a real temperature (the sensor never reads 0 volts); the point is to see that the
   formula still works without overflow at the edge.
2. In `code/p22-hello-led`, change `Level::High` to `Level::Low` and run `cargo build --release`.
   The build still succeeds; on the board, the LED would now stay off.

## Exercises

### Exercise 1 — Read the triple (easy)

Explain each part of the target name `thumbv6m-none-eabi` (used by the older RP2040 chip). What is
different from `thumbv8m.main-none-eabihf`?

<details>
<summary>Hint</summary>

Compare the parts one by one with the table in Section 6. What does the missing `hf` mean?

</details>

<details>
<summary>Answer</summary>

`thumbv6m` is the Thumb instruction set of Armv6-M (the Cortex-M0+ core of the RP2040), `none` means
no operating system, `eabi` is the embedded ABI. There is no `hf`: the Cortex-M0+ has no floating-point
unit, so floating-point math is done in software. The RP2350 has the newer Armv8-M Mainline cores and a
hardware floating-point unit, hence `thumbv8m.main` and `eabihf`.

</details>

### Exercise 2 — A tested conversion (medium)

In `code/p22-sensor`, write `pub fn millivolts(raw: u16) -> u32` that converts a 12-bit reading to
millivolts (0 → 0, 4095 → 3300) without overflow, and **write tests** for 0, 4095 and 2048. Keep the
crate `#![no_std]`.

<details>
<summary>Hint</summary>

Multiply first, then divide, in a type that is big enough: 4095 × 3300 = 13,513,500 fits into `u32`.

</details>

<details>
<summary>Answer</summary>

```rust
pub fn millivolts(raw: u16) -> u32 {
    raw as u32 * 3300 / 4095
}

#[cfg(test)]
mod millivolt_tests {
    use super::*;

    #[test]
    fn edges_and_middle() {
        assert_eq!(millivolts(0), 0);
        assert_eq!(millivolts(4095), 3300);
        assert_eq!(millivolts(2048), 1650);
    }
}
```

Why it works: `u32` holds up to about 4.29 billion, and the largest product is 13.5 million. Integer
division rounds down, so 2048 × 3300 / 4095 = 1650.4… becomes 1650.

</details>

### Exercise 3 — What does a panic do? (hard)

In `code/p22-hello-led`, a panic now loops forever, and the LED stays in whatever state it had. Change
the panic handler so that it calls `cortex_m::asm::udf()` instead of looping. Explain what `udf` does
and why a debugger shows this better than an endless loop.
Build with `cargo build --release`.

<details>
<summary>Hint</summary>

`udf` means "permanently undefined instruction". What does the CPU do when it meets an instruction
that does not exist?

</details>

<details>
<summary>Answer</summary>

```rust
#[panic_handler]
fn panic(_info: &PanicInfo) -> ! {
    cortex_m::asm::udf()
}
```

`udf` executes an undefined instruction, which makes the CPU raise a fault (the `HardFault` exception).
A debugger such as `probe-rs` stops there and shows that something went wrong, while an endless loop
looks exactly like a working program that is just busy. Lesson 22.12 uses `panic-probe`, a ready-made
handler that also prints the panic message through the probe.

</details>

## Quiz

1. Name three things a program on a microcontroller does not have, compared with a program on macOS.

<details>
<summary>Answer</summary>

No operating system (nobody loads it, gives it memory or cleans up), no `std` (no files, threads,
`println!`), and very little memory (hundreds of kilobytes of RAM). Also: it never returns from `main`.

</details>

2. What is the difference between `core` and `std`?

<details>
<summary>Answer</summary>

`core` is the part of the standard library that needs nothing from an operating system: types,
traits, iterators, integer math, formatting. `std` adds things that need an operating system: files,
threads, networking, printing. `no_std` crates use only `core` (and maybe `alloc`).

</details>

3. Why must a panic handler and `main` have the return type `!`?

<details>
<summary>Answer</summary>

There is nothing to return to: no operating system and no caller. `!` ("never") tells the compiler
the function never returns, for example because it loops forever or stops the CPU.

</details>

4. What do `memory.x` and `build.rs` do in a firmware project?

<details>
<summary>Answer</summary>

`memory.x` tells the linker where flash and RAM are (and, for the RP2350, where the boot blocks go).
`build.rs` runs on the Mac before the build, copies `memory.x` where the linker finds it and passes the
`link.x` linker script from `cortex-m-rt`.

</details>

5. What does the line `let _led = Output::new(p.PIN_25, Level::High);` do with ownership?

<details>
<summary>Answer</summary>

It moves `PIN_25` out of the peripherals struct into the `Output` driver. After that no other code can
use pin 25 by accident. The variable keeps the driver alive for the whole program.

</details>

## Summary

- A microcontroller is a whole small computer on one chip: CPU core, flash, SRAM and peripherals,
  usually without an operating system.
- Firmware starts from flash at power-on, never returns, and controls hardware by writing to
  memory-mapped registers.
- `#![no_std]` links only `core`; `#![no_main]` drops the normal `main`; a `#[panic_handler]` decides
  what a panic does; `cortex-m-rt` provides the vector table and `#[entry]`.
- Cross-compiling needs a target (`rustup target add thumbv8m.main-none-eabihf`), a default target in
  `.cargo/config.toml`, and a memory map in `memory.x`.
- Keep logic in small `no_std` libraries and test it on the Mac; only the hardware layer needs the board.
- `probe-rs` with a debug probe flashes the firmware through `cargo run`.

## Sources and further reading

- [The Embedded Rust Book](https://docs.rust-embedded.org/book/) — the official introduction to embedded Rust, including `no_std` and the start-up process.
- [The Embedonomicon](https://docs.rust-embedded.org/embedonomicon/) — how `cortex-m-rt`, the vector table and linker scripts work, built from zero.
- [Raspberry Pi Pico 2 datasheet](https://datasheets.raspberrypi.com/pico/pico-2-datasheet.pdf) — the board, its pins and its LED.
- [RP2350 datasheet](https://datasheets.raspberrypi.com/rp2350/rp2350-datasheet.pdf) — the chip: memory map, boot ROM, temperature sensor formula.
- [`embassy-rp` on docs.rs](https://docs.rs/embassy-rp/0.10.0/embassy_rp/) — the HAL used in this track.
- [rustc platform support: thumbv8m.main-none-eabihf](https://doc.rust-lang.org/rustc/platform-support/thumbv8m.main-none-eabi.html) — what the target supports.
- [probe-rs](https://probe.rs/) — installing and using the flashing and debugging tool.

---

[🏠 Start](../START-HERE.md) · [← 22.9 eBPF track: project and next steps](09-ebpf-track-project-and-next-steps.md) · [📋 Phase 22](README.md) · [22.11 Embedded track: GPIO and embedded-hal →](11-embedded-track-gpio-and-embedded-hal.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
