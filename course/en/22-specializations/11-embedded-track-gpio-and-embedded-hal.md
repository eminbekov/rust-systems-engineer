<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 22.10 Embedded track: microcontrollers and no_std](10-embedded-track-microcontrollers-and-no-std.md) · [📋 Phase 22](README.md) · [22.12 Embedded track: async with Embassy →](12-embedded-track-async-with-embassy.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/22-specializations/11-embedded-track-gpio-and-embedded-hal.md) <!-- nav -->

# Lesson 22.11 — Embedded track: GPIO and embedded-hal

> **Phase 22 — Specializations** · Lesson 11 of 30 · ⏱ about 2.5 hours (2 sittings)

## What you will learn

- What GPIO (General-Purpose Input/Output) pins are, how to wire an LED (Light-Emitting Diode) and a button safely, and what pull-up resistors do.
- What the HAL (Hardware Abstraction Layer) does for you, by switching a pin with one raw register write.
- The `embedded-hal` 1.0 traits `OutputPin`, `InputPin` and `DelayNs`, and why drivers are written against them.
- How to write a generic driver once and test it on your Mac with fake pins.
- How Rust's ownership stops two parts of the firmware from using the same pin.

## Before you start

- You finished [Lesson 22.10 — Embedded track: microcontrollers and no_std](10-embedded-track-microcontrollers-and-no-std.md)
  and can build `code/p22-hello-led`.
- You know traits and generics ([Lessons 05.1–05.2](../05-rust-intermediate/02-traits.md)), associated
  types ([Lesson 06.2](../06-rust-advanced/02-advanced-traits-and-types.md)) and test doubles
  ([Lesson 09.3](../09-testing-debugging-benchmarking/03-test-doubles.md)).
- Hardware (optional): the Pico 2 with the Debug Probe, a breadboard, a push button and jumper wires.

## New words

| Word | Meaning |
|---|---|
| **GPIO** (General-Purpose Input/Output) | A pin that your program can drive high or low (output) or read (input). |
| **high / low** | The two levels of a digital pin. On the RP2350, high is about 3.3 volts, low is about 0 volts. |
| **LED** (Light-Emitting Diode) | A small lamp that lights when current flows from its long leg (anode) to its short leg (cathode). |
| **resistor** | A part that limits current. Its value is measured in ohms (Ω). |
| **pull-up resistor** | A resistor that connects an input to high, so the input reads high when nothing else drives it. The RP2350 has built-in ones. |
| **active low** | A signal that means "on" when it is low. A button wired to ground with a pull-up is active low. |
| **bounce** | The fast, random on-off flicker of a mechanical button contact during the first milliseconds of a press. |
| **PAC** (Peripheral Access Crate) | A crate generated from the chip's register description. It gives one type per register, but no high-level logic. |
| **`embedded-hal`** | A crate of traits (output pin, input pin, delay, and buses such as I²C (Inter-Integrated Circuit) and SPI (Serial Peripheral Interface), two common ways to talk to sensor chips) that HALs implement and drivers use. |
| **driver** | Code that controls one device (an LED pattern, a sensor, a display) through `embedded-hal` traits. |
| **volatile access** | A memory read or write that the compiler must not remove, merge or reorder. Needed for hardware registers. |

## 1. GPIO pins, LEDs and buttons

A **GPIO** pin is a tiny switch that your program controls. As an **output**, the program connects it
either to 3.3 volts (**high**) or to 0 volts (**low**). As an **input**, the program reads which of the
two levels is on the wire.

**An LED needs a resistor.** An LED lets almost unlimited current through once its voltage is reached
(about 2 volts for a red LED). Too much current burns the LED or the pin. A resistor in series limits
it. Ohm's law gives the value: *current = voltage / resistance*. With 3.3 V from the pin and 2 V across
the LED, 1.3 V remain for the resistor; 1.3 V / 330 Ω ≈ 4 mA, a safe current for an RP2350 pin.

```text
Pico 2 pin 21 (GPIO 16) ---[ 330 Ω ]---(long leg) LED (short leg)--- pin 23 (GND)

Pico 2 pin 20 (GPIO 15) ---[ button ]--- pin 18 (GND)
```

GND means ground, the 0-volt side of the circuit. The board's green LED on GPIO 25 already has its
resistor, so this lesson uses it, and you only need to wire the button.

**A button needs a defined level.** When the button above is open, GPIO 15 is connected to nothing.
An input connected to nothing "floats": it picks up electrical noise and reads high or low at random.
The fix is a **pull-up resistor** inside the chip that weakly connects the pin to 3.3 V:

```text
          3.3 V
            |
         [pull-up]        button open:    pin reads HIGH
            |             button pressed: pin is tied to GND, reads LOW
GPIO 15 ----+----[ button ]---- GND
```

So the button is **active low**: pressed means low. You choose the pull-up in code with `Pull::Up`.

> ⚠️ **Common mistake:** never connect an output pin directly to ground or to 3.3 V, and never connect
> two output pins together. If one drives high and the other low, a large current flows and can damage
> the chip. Use a resistor, or configure the pin as an input.

## 2. What the HAL does for you: one raw register write

In Lesson 22.10 you saw that peripherals are registers at fixed addresses. The RP2350 has a block
called **SIO** (Single-cycle I/O; I/O means input/output) at address `0xd000_0000`. One of its registers, `GPIO_OUT_XOR` at
offset `0x28`, flips every output pin whose bit you write as 1. To flip GPIO 25, write `1 << 25`.

Let us do this by hand once. The firmware lives in the package `code/p22-gpio/firmware`; Section 4
creates it and explains its files, so for now only read the code. It goes into
`code/p22-gpio/firmware/src/bin/raw.rs`, a second binary next to `src/main.rs`:

```rust
#![no_std]
#![no_main]

use core::panic::PanicInfo;
use cortex_m_rt::entry;
use embassy_rp::gpio::{Level, Output};

// RP2350 datasheet: the SIO block starts at 0xd000_0000, and the
// GPIO_OUT_XOR register at offset 0x28 flips every output whose bit is 1.
const SIO_GPIO_OUT_XOR: *mut u32 = 0xd000_0028 as *mut u32;

#[entry]
fn main() -> ! {
    let p = embassy_rp::init(Default::default());
    // Let the HAL do the setup: pad, pin function, output enable.
    let _led = Output::new(p.PIN_25, Level::Low);

    loop {
        // SAFETY: SIO_GPIO_OUT_XOR is a valid register of the RP2350 that is
        // always mapped, and writing a bit mask only flips those output pins.
        unsafe { core::ptr::write_volatile(SIO_GPIO_OUT_XOR, 1 << 25) };
        cortex_m::asm::delay(75_000_000); // about 0.5 s at 150 MHz
    }
}

#[panic_handler]
fn panic(_info: &PanicInfo) -> ! {
    loop {}
}
```

**What happens:**

1. `SIO_GPIO_OUT_XOR` is a raw pointer to the register. Creating a raw pointer is safe; using it is not.
2. `Output::new` still does the setup work: it connects the pin to the SIO block, enables the output
   driver and configures the electrical "pad". By hand, this would be three more registers in two other
   peripheral blocks.
3. `write_volatile` writes the bit mask. **Volatile** tells the compiler "this write has an effect
   outside the program, do not remove or merge it". A normal write to memory that is never read again
   could be optimized away.
4. `cortex_m::asm::delay(75_000_000)` burns at least 75 million CPU (Central Processing Unit) cycles, about half a second at 150 MHz.

```bash
cargo build --release --bin raw
```

```text
    Finished `release` profile [optimized + debuginfo] target(s) in 0.81s
```

On the board (`cargo run --release --bin raw`), the green LED blinks once per second. It works, but
look at what you had to know: an address, an offset, a bit number, and that two other blocks are
already set up. One wrong digit writes into some other peripheral, and the compiler cannot help.

The Rust embedded world solves this in layers:

```text
+------------------------------------------------------+
| your application (firmware main)                     |
+------------------------------------------------------+
| drivers: LED patterns, sensors, displays             |  use only embedded-hal traits
+------------------------------------------------------+
| embedded-hal traits: OutputPin, InputPin, DelayNs... |  the shared contract
+------------------------------------------------------+
| HAL: embassy-rp (Output, Input, Delay...)            |  safe types, implements the traits
+------------------------------------------------------+
| PAC: rp-pac (one type per register)                  |  generated from the chip's description
+------------------------------------------------------+
| hardware registers at fixed addresses                |
+------------------------------------------------------+
```

> 🔒 **Security:** every raw register write is `unsafe` for a reason: the compiler cannot check the
> address. Keep such code in one small, reviewed module with a `// SAFETY:` comment, exactly as in
> [Lesson 10.12](../10-security-fundamentals/12-memory-safety-and-reviewing-unsafe.md). Application
> code should only use the HAL.

## 3. The `embedded-hal` traits

`embedded-hal` 1.0 is a small crate of **traits** only. Every HAL implements them for its own types,
and every **driver** is generic over them. The three you need now:

```rust
// From embedded-hal 1.0 (simplified)
pub trait ErrorType {
    type Error: core::fmt::Debug;
}

pub trait OutputPin: ErrorType {
    fn set_low(&mut self) -> Result<(), Self::Error>;
    fn set_high(&mut self) -> Result<(), Self::Error>;
}

pub trait InputPin: ErrorType {
    fn is_high(&mut self) -> Result<bool, Self::Error>;
    fn is_low(&mut self) -> Result<bool, Self::Error>;
}

pub trait DelayNs {
    fn delay_ns(&mut self, ns: u32);
    fn delay_us(&mut self, us: u32) { /* default: calls delay_ns */ }
    fn delay_ms(&mut self, ms: u32) { /* default: calls delay_us */ }
}
```

**What happens:**

1. Every pin operation returns a `Result`, because some pins can fail: a pin on an I/O expander
   chip is set over a bus, and the bus can fail. For the RP2350's own pins, the error type is
   `Infallible`, an enum with no values, so the operation can never fail.
2. `ErrorType` holds the associated type `Error` once, so `OutputPin` and `InputPin` share it.
3. `DelayNs` needs only `delay_ns`; `delay_us` and `delay_ms` have default methods built on it.

Now the driver. Create a library package next to the firmware:

```bash
cd ~/projects/rust
mkdir code/p22-gpio
cargo new --lib code/p22-gpio/driver --name p22-gpio-driver
cd code/p22-gpio/driver
cargo add embedded-hal@1
```

`code/p22-gpio/driver/src/lib.rs` (the tests follow in Section 4):

```rust
#![no_std]

use embedded_hal::delay::DelayNs;
use embedded_hal::digital::OutputPin;

/// Blinks `led` `times` times: on for `ms` milliseconds, then off for `ms`.
pub fn blink<P, D>(led: &mut P, delay: &mut D, times: u32, ms: u32) -> Result<(), P::Error>
where
    P: OutputPin,
    D: DelayNs,
{
    for _ in 0..times {
        led.set_high()?;
        delay.delay_ms(ms);
        led.set_low()?;
        delay.delay_ms(ms);
    }
    Ok(())
}
```

> 📏 **Convention:** a driver takes the pins and delays it needs as **generic parameters** bounded by
> `embedded-hal` traits, and never names a HAL type. Only the firmware's `main` knows which chip it runs
> on. Published driver crates are usually named after the device, for example a crate for a sensor chip.

`blink` knows nothing about the RP2350. It works with any type that is an output pin and any type that
can wait: an RP2350 pin, an ESP32 pin, a pin on an expander chip, or a fake pin in a test.

## 4. Testing the driver with fake pins

A **fake** (Lesson 09.3) is a simple, working stand-in. A fake pin records the levels it was set to; a
fake delay adds up the time instead of waiting. Add this test module to the end of `src/lib.rs`:

```rust
#[cfg(test)]
mod tests {
    extern crate std;

    use super::*;
    use core::convert::Infallible;
    use embedded_hal::digital::ErrorType;
    use std::vec::Vec;

    /// A fake pin that remembers every level it was set to.
    #[derive(Default)]
    struct FakePin {
        levels: Vec<bool>,
    }

    impl ErrorType for FakePin {
        type Error = Infallible;
    }

    impl OutputPin for FakePin {
        fn set_high(&mut self) -> Result<(), Infallible> {
            self.levels.push(true);
            Ok(())
        }
        fn set_low(&mut self) -> Result<(), Infallible> {
            self.levels.push(false);
            Ok(())
        }
    }

    /// A fake delay that only adds up the time instead of waiting.
    #[derive(Default)]
    struct FakeDelay {
        total_ns: u64,
    }

    impl DelayNs for FakeDelay {
        fn delay_ns(&mut self, ns: u32) {
            self.total_ns += ns as u64;
        }
    }

    #[test]
    fn blinks_three_times() {
        let mut pin = FakePin::default();
        let mut delay = FakeDelay::default();
        blink(&mut pin, &mut delay, 3, 200).unwrap();
        assert_eq!(pin.levels, [true, false, true, false, true, false]);
        assert_eq!(delay.total_ns, 6 * 200 * 1_000_000);
    }
}
```

**What happens:**

1. `extern crate std;` inside the test module brings `std` back **only for the tests**. The library
   itself stays `no_std`; tests run on the Mac, where `std` exists, so they may use `Vec`.
2. `FakePin` implements the same two traits as a real pin. `blink` cannot tell the difference.
3. `FakeDelay` only implements `delay_ns`. The default `delay_ms` calls it, so 200 ms arrive as
   200,000,000 nanoseconds in total, and the test runs instantly.

```bash
cargo test
```

```text
running 1 test
test tests::blinks_three_times ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

> 🧪 **Testing:** this is the most important embedded testing pattern. Write drivers against
> `embedded-hal` traits, test them with fakes on the host, and run only a short final check on the
> board. The crate `embedded-hal-mock` offers ready-made mocks with expectations if you need more.

**Using the driver on the chip.** Create the firmware package as a second package next to the driver.
It is a copy of `code/p22-hello-led` with a new name:

```bash
cd ~/projects/rust
cp -R code/p22-hello-led code/p22-gpio/firmware
cd code/p22-gpio/firmware
rm -rf target
```

```text
code/p22-gpio/
├── driver/      ← no_std library, tested on the Mac with `cargo test`
│   ├── Cargo.toml
│   └── src/lib.rs
└── firmware/    ← binary for the chip, built with `cargo build --release`
    ├── .cargo/config.toml
    ├── build.rs
    ├── memory.x
    ├── Cargo.toml
    └── src/
        ├── main.rs
        └── bin/raw.rs
```

Put the raw example from Section 2 into `src/bin/raw.rs` (create the folder with `mkdir src/bin`).
Replace `code/p22-gpio/firmware/Cargo.toml`:

```toml
[package]
name = "p22-gpio-firmware"
version = "0.1.0"
edition = "2024"
default-run = "p22-gpio-firmware"

[dependencies]
p22-gpio-driver = { path = "../driver" }
cortex-m = "0.7"
cortex-m-rt = "0.7"
embassy-rp = { version = "0.10", features = ["rp235xa", "critical-section-impl", "time-driver"] }
embassy-time = "0.5"
embedded-hal = "1"

[profile.release]
debug = true
```

New here: `default-run` tells `cargo run` which of the two binaries to start, the path dependency
points to the driver, and the `time-driver` feature lets `embassy-time` read the chip's hardware timer. `code/p22-gpio/firmware/src/main.rs`:

```rust
#![no_std]
#![no_main]

use core::panic::PanicInfo;
use cortex_m_rt::entry;
use embassy_rp::gpio::{Input, Level, Output, Pull};
use embassy_time::Delay;
use embedded_hal::delay::DelayNs;
use p22_gpio_driver::blink;

#[entry]
fn main() -> ! {
    let p = embassy_rp::init(Default::default());
    let mut led = Output::new(p.PIN_25, Level::Low);
    // The button connects GPIO 15 to ground, so pressed = low.
    let button = Input::new(p.PIN_15, Pull::Up);
    let mut delay = Delay;

    loop {
        if button.is_low() {
            blink(&mut led, &mut delay, 3, 100).unwrap();
        } else {
            delay.delay_ms(10);
        }
    }
}

#[panic_handler]
fn panic(_info: &PanicInfo) -> ! {
    loop {}
}
```

**What happens:**

1. `Input::new(p.PIN_15, Pull::Up)` makes GPIO 15 an input with the internal pull-up switched on.
2. `embassy_time::Delay` implements `DelayNs` with the hardware timer, and `Output` implements
   `OutputPin`, so they fit into `blink`.
3. Every 10 ms the loop **polls** (checks) the button. While it is pressed, the LED blinks three times
   quickly. This is called a **super loop**: one endless loop that checks everything in turn.
4. `.unwrap()` is safe here because the error type is `Infallible`.

```bash
cargo build --release
cargo size --release
```

```text
    Finished `release` profile [optimized + debuginfo] target(s) in 7.44s
   text	   data	    bss	    dec	    hex	filename
  10932	      0	    304	  11236	   2be4	p22-gpio-firmware
```

Example behaviour on the board (`cargo run --release`): nothing happens until you press the button;
while you hold it, the green LED flashes in groups of three.

> ⚡ **Performance:** the super loop never sleeps; the CPU runs at full speed even when nothing
> happens, and a long `blink` blocks the button check. Lesson 22.12 replaces this with `async` tasks
> that sleep until the button or a timer wakes them.

## 5. Ownership of pins

`embassy_rp::init` returns each pin exactly once, as a value of its own type (`PIN_15`, `PIN_25`, …).
Passing a pin to `Output::new` moves it. What if a second part of the program tries to use the same pin
as an input? Change the button line to use `p.PIN_25`:

```text
error[E0382]: use of moved value: `p.PIN_25`
  --> src/main.rs:16:29
   |
14 |     let mut led = Output::new(p.PIN_25, Level::Low);
   |                               -------- value moved here
15 |     // The button connects GPIO 15 to ground, so pressed = low.
16 |     let button = Input::new(p.PIN_25, Pull::Up);
   |                             ^^^^^^^^ value used here after move
   |
   = note: move occurs because `p.PIN_25` has type `Peri<'_, PIN_25>`, which does not implement the `Copy` trait
```

This is the ownership rule from [Lesson 03.2](../03-ownership-and-borrowing/02-ownership-rules.md),
working on hardware: two drivers fighting over one pin is a real bug class in firmware, and here it is a
compile error. The type `Peri<'_, PIN_25>` is Embassy's wrapper for "the right to use pin 25".

## Common mistakes

> ⚠️ **Common mistake: no time driver.** Without the `time-driver` feature of `embassy-rp`,
> `embassy_time::Delay` has no clock to read, and linking fails:
>
> ```text
>   = note: rust-lld: error: undefined symbol: _embassy_time_now
>           >>> referenced by lib.rs:149 (.../embassy-time-driver-0.2.2/src/lib.rs:149)
> ```
>
> Fix: add `"time-driver"` to the features of `embassy-rp`.

> ⚠️ **Common mistake: `cargo test` in the firmware folder.** The firmware's `.cargo/config.toml` makes
> every build use the chip target, and the chip has no test harness:
>
> ```text
> error[E0463]: can't find crate for `test`
> ```
>
> Fix: run tests in `code/p22-gpio/driver`, which builds for your Mac. Keep logic in the driver.

> ⚠️ **Common mistake: a floating input.** `Input::new(p.PIN_15, Pull::None)` with an open button reads
> random values; the LED blinks "by itself" when you touch the wires. Fix: `Pull::Up` (button to
> ground) or `Pull::Down` (button to 3.3 V).

## Try it yourself

1. In `code/p22-gpio/driver`, change the test to call `blink` with `times = 0`. What do you expect in
   `pin.levels`? Run `cargo test` and check (an empty list, and `total_ns` is 0).
2. In the firmware, change `blink(&mut led, &mut delay, 3, 100)` to five slow blinks of 300 ms and
   build again.

## Exercises

### Exercise 1 — Pick a resistor (easy)

A green LED needs about 2.2 V. You want at most 3 mA from a 3.3 V pin. What resistor value do you need
at least? Is 330 Ω OK?

<details>
<summary>Hint</summary>

The resistor gets the rest of the voltage: 3.3 − 2.2 volts. Then *R = V / I*.

</details>

<details>
<summary>Answer</summary>

1.1 V / 0.003 A ≈ 367 Ω, so at least about 370 Ω; the next common value is 390 Ω. With 330 Ω the current
is 1.1 / 330 ≈ 3.3 mA, slightly over your limit but still safe for the pin. A bigger resistor only makes
the LED a little dimmer.

</details>

### Exercise 2 — Generic toggle with a test (medium)

In the driver, write `pub fn toggle_n<P: StatefulOutputPin>(led: &mut P, n: u32) -> Result<(), P::Error>`
that calls `toggle` `n` times. **Write a test** with a fake pin that starts low and check that after 3
toggles it is high. (`StatefulOutputPin` needs `is_set_high` and `is_set_low`; `toggle` has a default.)

<details>
<summary>Hint</summary>

Give the fake pin a `state: bool` field. `set_high` sets it to `true`, `is_set_high` returns
`Ok(self.state)`. Import `embedded_hal::digital::StatefulOutputPin`.

</details>

<details>
<summary>Answer</summary>

```rust
use embedded_hal::digital::StatefulOutputPin;

pub fn toggle_n<P: StatefulOutputPin>(led: &mut P, n: u32) -> Result<(), P::Error> {
    for _ in 0..n {
        led.toggle()?;
    }
    Ok(())
}

#[cfg(test)]
mod toggle_tests {
    use super::*;
    use core::convert::Infallible;
    use embedded_hal::digital::ErrorType;

    struct StatePin {
        state: bool,
    }

    impl ErrorType for StatePin {
        type Error = Infallible;
    }

    impl OutputPin for StatePin {
        fn set_high(&mut self) -> Result<(), Infallible> {
            self.state = true;
            Ok(())
        }
        fn set_low(&mut self) -> Result<(), Infallible> {
            self.state = false;
            Ok(())
        }
    }

    impl StatefulOutputPin for StatePin {
        fn is_set_high(&mut self) -> Result<bool, Infallible> {
            Ok(self.state)
        }
        fn is_set_low(&mut self) -> Result<bool, Infallible> {
            Ok(!self.state)
        }
    }

    #[test]
    fn three_toggles_from_low_end_high() {
        let mut pin = StatePin { state: false };
        toggle_n(&mut pin, 3).unwrap();
        assert!(pin.state);
    }
}
```

Why it works: the default `toggle` reads `is_set_low` and calls `set_high` or `set_low`. Low → high →
low → high after three calls. This fake needs no `Vec`, so the test module does not need `extern crate std`.

</details>

### Exercise 3 — A debouncer (hard)

A pressed button bounces for a few milliseconds. Write a `no_std` `Debouncer` in the driver that you
feed one raw sample every millisecond with `update(pressed: bool) -> Option<bool>`. It returns
`Some(new_state)` only when the raw value has been the same for 5 samples in a row **and** differs from
the last reported state; otherwise `None`. **Write tests**: a clean press, a bouncing press
(`true, false, true, true, true, true, true`) and a short glitch (two `true` samples) that must be ignored.

<details>
<summary>Hint</summary>

Keep three fields: the last reported `state`, the last raw `candidate` value, and a `count` of how many
samples in a row equal `candidate`. Reset `count` to 1 when the raw value changes.

</details>

<details>
<summary>Answer</summary>

```rust
pub struct Debouncer {
    state: bool,
    candidate: bool,
    count: u8,
}

impl Debouncer {
    pub const STABLE: u8 = 5;

    pub const fn new() -> Self {
        Self {
            state: false,
            candidate: false,
            count: 0,
        }
    }

    pub fn update(&mut self, pressed: bool) -> Option<bool> {
        if pressed == self.candidate {
            self.count = self.count.saturating_add(1);
        } else {
            self.candidate = pressed;
            self.count = 1;
        }
        if self.count >= Self::STABLE && self.candidate != self.state {
            self.state = self.candidate;
            return Some(self.state);
        }
        None
    }
}

impl Default for Debouncer {
    fn default() -> Self {
        Self::new()
    }
}

#[cfg(test)]
mod debounce_tests {
    use super::*;

    fn feed(d: &mut Debouncer, samples: &[bool]) -> Option<bool> {
        let mut last = None;
        for &s in samples {
            if let Some(v) = d.update(s) {
                last = Some(v);
            }
        }
        last
    }

    #[test]
    fn clean_press_is_reported_once() {
        let mut d = Debouncer::new();
        assert_eq!(feed(&mut d, &[true; 5]), Some(true));
        assert_eq!(feed(&mut d, &[true; 5]), None);
    }

    #[test]
    fn bouncing_press_is_reported_after_it_settles() {
        let mut d = Debouncer::new();
        assert_eq!(
            feed(&mut d, &[true, false, true, true, true, true, true]),
            Some(true)
        );
    }

    #[test]
    fn short_glitch_is_ignored() {
        let mut d = Debouncer::new();
        assert_eq!(feed(&mut d, &[true, true, false, false, false]), None);
    }
}
```

Why it works: a value is reported only after it has stayed the same for `STABLE` samples, so bounces
(short changes) restart the count. `saturating_add` keeps `count` from overflowing when the button is
held for a long time. `const fn new` allows a `static` debouncer in firmware, and `Default` is what
Clippy expects next to a `new` without arguments.

</details>

## Quiz

1. Why does a button input need a pull-up (or pull-down) resistor?

<details>
<summary>Answer</summary>

Without it, the open button leaves the pin connected to nothing, so it "floats" and reads random levels.
The pull-up gives a defined high level when the button is open; pressing it ties the pin to ground (low).

</details>

2. What does `write_volatile` guarantee that a normal write does not?

<details>
<summary>Answer</summary>

The compiler must perform the write exactly as written: not remove it, merge it with other writes or
move it past other volatile accesses. Hardware registers have effects outside the program, so normal
optimizations would break them.

</details>

3. Why does `OutputPin::set_high` return a `Result`, and why can `blink` use `?`?

<details>
<summary>Answer</summary>

Some pins can fail (for example, pins on an expander chip behind a bus). `blink` returns
`Result<(), P::Error>`, the same error type as the pin, so `?` passes a pin error up to the caller.

</details>

4. Why are drivers written against `embedded-hal` traits instead of `embassy_rp::gpio::Output`?

<details>
<summary>Answer</summary>

So that one driver works with every HAL and every chip, and so that it can be tested on the host with
fake pins and delays.

</details>

5. What does the error `E0382: use of moved value: p.PIN_25` protect you from?

<details>
<summary>Answer</summary>

From two drivers using the same pin at the same time (for example, one as an output, one as an input).
Each pin is a value that can be moved into exactly one driver.

</details>

## Summary

- A GPIO pin is high (3.3 V) or low (0 V); outputs drive it, inputs read it.
- An LED needs a series resistor; a button needs a pull-up or pull-down. A button to ground with a
  pull-up is active low.
- Under the HAL, a pin change is a volatile write to a register; the HAL hides addresses, bits and setup.
- `embedded-hal` 1.0 defines the traits `OutputPin`, `InputPin`, `StatefulOutputPin` and `DelayNs`;
  drivers are generic over them.
- Test drivers on the Mac with fakes (`extern crate std;` in the test module); keep the firmware thin.
- Each pin is a value: moving it into a driver prevents double use at compile time.

## Sources and further reading

- [`embedded-hal` 1.0 documentation](https://docs.rs/embedded-hal/1.0.0/embedded_hal/) — the traits and the rules for drivers and HALs.
- [The Embedded Rust Book — Static guarantees](https://docs.rust-embedded.org/book/static-guarantees/index.html) — how types and ownership model peripherals.
- [`embassy-rp` gpio module](https://docs.rs/embassy-rp/0.10.0/embassy_rp/gpio/index.html) — `Output`, `Input`, `Pull` and more.
- [RP2350 datasheet](https://datasheets.raspberrypi.com/rp2350/rp2350-datasheet.pdf) — the SIO block and GPIO registers (chapter "Processor subsystem").
- [`core::ptr::write_volatile`](https://doc.rust-lang.org/core/ptr/fn.write_volatile.html) — the exact rules for volatile access.

---

[🏠 Start](../START-HERE.md) · [← 22.10 Embedded track: microcontrollers and no_std](10-embedded-track-microcontrollers-and-no-std.md) · [📋 Phase 22](README.md) · [22.12 Embedded track: async with Embassy →](12-embedded-track-async-with-embassy.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
