<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 22.12 Embedded track: async with Embassy](12-embedded-track-async-with-embassy.md) · [📋 Phase 22](README.md) · [22.14 Databases track: how query engines work →](14-databases-track-how-query-engines-work.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/22-specializations/13-embedded-track-project-and-next-steps.md) <!-- nav -->

# Lesson 22.13 — Embedded track: project and next steps

> **Phase 22 — Specializations** · Lesson 13 of 30 · ⏱ about 20 hours (10 sittings)

## What you will learn

- How to plan firmware as a host-tested `no_std` library plus a thin Embassy firmware.
- How Morse code timing works, and how to turn text into on/off steps without a heap.
- How to use the async `DelayNs` trait from `embedded-hal-async`, and how to cancel a running pattern with `select`.
- Where to go next in embedded Rust: books, projects to contribute to, and communities.

## Before you start

- You finished [Lesson 22.12 — Embedded track: async with Embassy](12-embedded-track-async-with-embassy.md)
  and can build `code/p22-embassy`.
- You remember the fake pins of [Lesson 22.11](11-embedded-track-gpio-and-embedded-hal.md) and
  iterators from [Lesson 05.5](../05-rust-intermediate/05-iterators.md).
- A Git repository in `code/p22-morse` (`git init`), one commit per milestone.
- Hardware is optional. Milestones M1–M4 are checked on your Mac; M5 needs the board for the final check.

## New words

| Word | Meaning |
|---|---|
| **Morse code** | A way to send letters as short and long signals: a dot (short) and a dash (long). |
| **unit** (Morse) | The length of one dot. Every other length in Morse code is a multiple of it. |
| **`embedded-hal-async`** | The async twin of `embedded-hal`: the same kinds of traits, but their methods are `async fn`. |
| **`Signal`** | A tiny `embassy-sync` type that holds the latest value; a task waits until another task "signals" a new one. |
| **beacon** | A device that repeats the same message again and again, like a lighthouse. |

## 1. Morse code in five rules

Morse code turns every letter and digit into dots and dashes: `S` is `...`, `O` is `---`, so `SOS` is
`... --- ...`. The timing is defined in **units**:

| Element | Length |
|---|---|
| dot | LED (Light-Emitting Diode) on for 1 unit |
| dash | LED on for 3 units |
| gap between the dots and dashes of one letter | off for 1 unit |
| gap between letters | off for 3 units |
| gap between words (a space) | off for 7 units |

Example: the text `E E` (E is a single dot) is: on 1, off 7, on 1. `SOS` is
`on1 off1 on1 off1 on1` (S, 5 units) + `off3` + `on3 off1 on3 off1 on3` (O, 11 units) + `off3` + S
(5 units) = **27 units**. With a unit of 100 ms, `SOS` takes 2.7 seconds.

```text
SOS, one character = one unit:

#_#_#___###_###_###___#_#_#
 S        O          S
```

## The project

### What you will build

`p22-morse`: a **Morse beacon**. The green LED of the Pico 2 repeats a message in Morse code. Each
press of the button switches to the next message (`SOS` → `HELLO` → `RUST` → `SOS` …), and the new
message starts **at once**, even in the middle of a letter. `defmt` logs which message is playing.

The design follows the rule of this track: almost all logic lives in a `no_std` library that you test
on your Mac; the firmware only connects pins, timers and tasks.

```text
code/p22-morse/
├── morse/        ← no_std library: encoding, timings, play(); tested with `cargo test`
│   ├── Cargo.toml
│   └── src/lib.rs
├── firmware/     ← Embassy firmware for the Pico 2; built with `cargo build --release`
│   ├── .cargo/config.toml
│   ├── build.rs
│   ├── memory.x
│   ├── Cargo.toml
│   └── src/main.rs
└── README.md
```

### Contract

**Library `morse`** (`code/p22-morse/morse`, package name `morse`, `#![no_std]`, no heap):

```rust
/// Morse code of one letter or digit, for example 'A' -> ".-". Case does not matter.
/// Any other character (space, punctuation, letters outside A-Z) gives None.
pub fn encode_char(c: char) -> Option<&'static str>;

/// One step of a pattern: LED on or off for `units` units.
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub struct Step {
    pub on: bool,
    pub units: u8,
}

/// All steps for `text`, using the five rules. Unknown characters are skipped.
/// Spaces at the start or end produce no steps; there is no gap after the last symbol.
pub fn timings(text: &str) -> Timings<'_>;   // Timings implements Iterator<Item = Step>

/// Sum of the units of all steps of `text`.
pub fn total_units(text: &str) -> u32;

/// Plays `text` on `led`, one unit = `unit_ms` milliseconds. The LED is off at the end.
pub async fn play<P, D>(led: &mut P, delay: &mut D, text: &str, unit_ms: u32) -> Result<(), P::Error>
where
    P: embedded_hal::digital::OutputPin,
    D: embedded_hal_async::delay::DelayNs;
```

Required results (they are your acceptance tests):

| Call | Result |
|---|---|
| `encode_char('A')`, `encode_char('s')`, `encode_char('5')`, `encode_char('?')` | `Some(".-")`, `Some("...")`, `Some(".....")`, `None` |
| `timings("E E")` collected | `[on 1, off 7, on 1]` |
| `total_units("SOS")`, `total_units("PARIS")`, `total_units("")` | `27`, `43`, `0` |
| `total_units("A?A")` | the same as `total_units("AA")` |
| `total_units("  E  ")` | `1` |
| `play` of `"SOS"` with `unit_ms = 100` | total delay 2,700 ms, last pin level low |

**Firmware** (`code/p22-morse/firmware`, package `p22-morse-firmware`): LED on GPIO (General-Purpose
Input/Output) pin 25, button on
GPIO 15 to ground (pull-up), messages `["SOS", "HELLO", "RUST"]`, unit 150 ms, a 7-unit pause between
repeats, `info!("playing {}", message)` each time a message starts.

### Step-by-step plan

#### M1 — Encode characters

Create the library with `cargo new --lib morse --name morse` inside `code/p22-morse`, add
`#![no_std]`, and write `encode_char` with a table of the 26 letters and 10 digits (the rules of the ITU,
the International Telecommunication Union; see Sources). Write the tests from the first row of the contract table.

<details>
<summary>Hint</summary>

A `const TABLE: [(char, &str); 36]` and `c.to_ascii_uppercase()` are enough. `iter().find(...)` returns
an `Option<&(char, &str)>`; `.map(|(_, code)| *code)` turns it into `Option<&'static str>`.

</details>

**Done when:**

```bash
cd ~/projects/rust/code/p22-morse/morse
cargo test
```

```text
test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

(The count is the number of your tests; it must be at least 1 and all must pass.) Commit: `M1: encode characters`.

#### M2 — Timings as an iterator

Write `Step`, the `Timings` iterator and `total_units`. No `Vec`: the iterator keeps a few fields of
state and produces one `Step` per `next` call. Add the tests for rows 2–5 of the contract table.

<details>
<summary>Hint</summary>

Useful state: the `Chars` of the text, the remaining symbols of the current letter (`&'static [u8]`),
the gap that must come before the next symbol (0, 1, 3 or 7), and whether anything was produced yet. A
gap and the following symbol are two steps; one way is to remember the symbol in an `Option<u8>` field
and return it on the next call.

</details>

> ⚠️ **Gotcha:** a bug in an iterator's state often means `next` never returns `None`. Then `sum()`
> loops "forever"; in a debug build it may end with `attempt to add with overflow` after a long time.
> If a test seems to hang, test `timings(...).take(20).collect::<Vec<_>>()` first and print it.

**Done when:** `cargo test` passes, including `total_units("PARIS") == 43`. (PARIS is the classic
test word for Morse speed: 43 units plus a 7-unit word gap is exactly 50.) Commit: `M2: timings`.

#### M3 — Play with async fakes

Add `embedded-hal-async` (`cargo add embedded-hal-async@1`) and write `play`. For the tests, add
`cargo add --dev embassy-futures@0.1` and write a fake async delay:

```rust
struct FakeDelay(u64); // total nanoseconds

impl embedded_hal_async::delay::DelayNs for FakeDelay {
    async fn delay_ns(&mut self, ns: u32) {
        self.0 += ns as u64;
    }
}
```

Run `play` in a test with `embassy_futures::block_on(...)` and a fake pin from Lesson 22.11.

<details>
<summary>Hint</summary>

`play` is a `for step in timings(text)` loop: set the pin, then `delay.delay_ms(step.units as u32 * unit_ms).await`.
After the loop, `led.set_low()` makes the "LED off at the end" rule true even for an empty text.

</details>

**Done when:** a test checks that playing `"SOS"` with `unit_ms = 100` adds up to
`2_700 * 1_000_000` nanoseconds and that the last level is `false`; `cargo test`,
`cargo clippy --all-targets` and `cargo fmt --check` are clean. Commit: `M3: play`.

#### M4 — The firmware

Copy `code/p22-embassy` to `code/p22-morse/firmware`, rename the package to `p22-morse-firmware`, and add
`morse = { path = "../morse" }` and `embassy-futures = "0.1"`. In `main`:

1. a `static NEXT: Signal<CriticalSectionRawMutex, usize> = Signal::new();` (from
   `embassy_sync::signal::Signal`);
2. a `button` task that, after a debounced press, computes the next index and calls `NEXT.signal(i)`;
3. a loop in `main` that logs the message, then waits with
   `select(play_and_pause, NEXT.wait())`, where `play_and_pause` is an `async` block that plays the
   message and then waits 7 units. If `NEXT` wins, switch to the new index.

<details>
<summary>Hint</summary>

`embassy_time::Delay` implements the async `DelayNs` too, so `morse::play(&mut led, &mut Delay, ...)`
works on the chip without changes. Because `select` drops the losing future, a press stops the current
message immediately; the next round of the loop starts the new one. Remember: the LED might be on when
the future is dropped. Set it low before playing again, or accept one short flash.

</details>

**Done when:**

```bash
cd ~/projects/rust/code/p22-morse/firmware
cargo build --release
cargo size --release
```

```text
    Finished `release` profile [optimized + debuginfo] target(s) in 1.42s
   text	   data	    bss	    dec	    hex	filename
  20140	     56	   1568	  21764	   5504	p22-morse-firmware
```

Your numbers will differ a little; `text` must stay below 32768 (32 KiB). `cargo clippy --release` is
clean. Commit: `M4: firmware`.

#### M5 — On the board, and a README

Flash with `cargo run --release` (board and probe needed). Write `code/p22-morse/README.md`: what the
beacon does, the wiring (button GPIO 15 ↔ GND, the ground pin), how to run the tests (`cd morse && cargo test`) and how to
flash (`cd firmware && cargo run --release`).

**Done when:** the LED repeats `SOS`, each press switches the message at once, and the log shows
(example; the exact lines depend on your code):

```text
INFO  playing SOS
└─ p22_morse_firmware::____embassy_main_task::{async_fn#0} @ src/main.rs:41
INFO  playing HELLO
└─ p22_morse_firmware::____embassy_main_task::{async_fn#0} @ src/main.rs:41
```

Without hardware, M5 is done when the README is written and `cargo build --release` works. Commit:
`M5: README`.

### Stretch goals

- **Speed control:** a long press (more than 1 second, measured with `embassy_time::Instant`) cycles
  the unit between 100, 150 and 250 ms.
- **Your own text:** read a message over USB (Universal Serial Bus) serial with `embassy-usb` (see the Embassy examples for
  the RP2350).
- **Decoder:** the opposite direction. A `no_std` decoder that takes on/off durations from a button
  and produces letters; test it on the Mac with recorded durations.
- **Low-power check:** count how often the executor wakes up with a `static` atomic counter and log it
  every 10 seconds.

## Common mistakes

> ⚠️ **Common mistake: `std` in the library.** If `morse` loses its `#![no_std]` line, `cargo test`
> still passes on the Mac, but `cargo build --release` in `firmware` fails, because the library is
> compiled for the chip too:
>
> ```text
> error[E0463]: can't find crate for `std`
>   |
>   = note: the `thumbv8m.main-none-eabihf` target may not support the standard library
>   = note: `std` is required by `morse` because it does not declare `#![no_std]`
> ```
>
> Build the firmware after every milestone, not only at the end.

> ⚠️ **Common mistake: one workspace for both packages.** If you make `code/p22-morse` a Cargo workspace
> and put `.cargo/config.toml` with the chip target at the top, `cargo test` builds the library for the
> chip and fails (`can't find crate for test`). Keep the config inside `firmware/`, as in the layout above.

> ⚠️ **Common mistake: the wrong `DelayNs`.** `embedded_hal::delay::DelayNs` (blocking) and
> `embedded_hal_async::delay::DelayNs` (async) have the same name. `play` must use the async one, or
> `.await` on `delay_ms` does not compile.

## Next steps in embedded Rust

You can now build, test and run async firmware. Good directions from here:

**Learn more**

- *The Embedded Rust Book* and *The Embedonomicon* (links below): the foundations, start-up code,
  linker scripts, interrupts and concurrency.
- *Discovery* (the micro:bit v2 edition): a second board with sensors, a different chip family, and
  many hands-on chapters.
- *The Embassy Book*: networking with `embassy-net`, USB with `embassy-usb`, Wi-Fi on the Pico 2 W.
- *Comprehensive Rust — Bare-metal Rust*: a compact course on `no_std`, MMIO (memory-mapped I/O) and writing drivers.
- The RP2350 datasheet: once you are comfortable, read the chapters on PIO (Programmable Input/Output: small
  state machines that drive pins on their own) and DMA (Direct Memory Access, copying data without the CPU, the Central Processing Unit).

**Contribute** (Lesson 24.4 teaches the contribution workflow in detail)

- **Write a driver.** Pick a cheap sensor (temperature, distance, light) that has no Rust driver yet,
  write it against `embedded-hal` 1.0 traits, test it with fakes, and publish it. This is the most common
  first contribution in embedded Rust.
- **Embassy** (`github.com/embassy-rs/embassy`): examples, documentation and HAL (Hardware Abstraction Layer) features for many chips.
- **probe-rs** (`github.com/probe-rs/probe-rs`): support for new chips and probes, and the debugger.
- **rp-hal** (`github.com/rp-rs/rp-hal`): the other HAL family for RP2040 and RP2350.
- Look for issues labelled "good first issue" or "help wanted"; read the contributing guide first.

**Communities**

- The Rust Embedded Working Group, which maintains `embedded-hal`, `cortex-m` and the books
  (`github.com/rust-embedded/wg`), and its Matrix chat room `#rust-embedded:matrix.org`.
- The Embassy Matrix room `#embassy-rs:matrix.org`.
- The list *Awesome Embedded Rust*: HALs, drivers, tools and blogs.

> 🧭 **End of the embedded track.** Continue with [Lesson 22.14 — Databases track: how query engines work](14-databases-track-how-query-engines-work.md)
> if you want a second track, or go to the [checkpoint, Lesson 22.30](30-checkpoint-your-specialization.md).

## Quiz

1. Why is almost all the logic of the beacon in the `morse` library and not in the firmware?

<details>
<summary>Answer</summary>

The library is `no_std` without hardware types, so it can be tested on the Mac in milliseconds with
fakes. The firmware only connects pins, timers and tasks, and that small part is checked on the board.

</details>

2. How many units does `total_units("SOS")` return, and why?

<details>
<summary>Answer</summary>

27: S is 5 units (three dots with two 1-unit gaps), O is 11 (three dashes with two gaps), plus two
3-unit letter gaps: 5 + 3 + 11 + 3 + 5 = 27.

</details>

3. How does a button press stop a message in the middle?

<details>
<summary>Answer</summary>

`main` waits with `select(play_and_pause, NEXT.wait())`. When the button task signals, `NEXT.wait()` is
ready first, so `select` returns and drops the playing future, which cancels its timer. The loop then
starts the new message.

</details>

4. Why does the same `play` function work in a Mac test and on the chip?

<details>
<summary>Answer</summary>

It is generic over the `OutputPin` and async `DelayNs` traits. The test passes a fake pin and a fake
delay; the firmware passes `Output` and `embassy_time::Delay`, which implement the same traits.

</details>

## Summary

- Plan firmware as a `no_std` logic library (tested on the host) plus a thin Embassy firmware.
- Morse timing is defined in units: dot 1, dash 3, gaps 1 / 3 / 7.
- An iterator with a few fields of state produces a whole pattern without a heap.
- `embedded-hal-async` makes delays and pins usable in generic `async fn`s, testable with `block_on`.
- `select` plus `Signal` lets a new event cancel a running pattern immediately.
- Next steps: drivers, Embassy, probe-rs, rp-hal, and the Rust Embedded community.

## Sources and further reading

- [ITU-R M.1677-1 — International Morse code](https://www.itu.int/rec/R-REC-M.1677-1-200910-I/) — the official definition of the code and its timing.
- [`embedded-hal-async` 1.0](https://docs.rs/embedded-hal-async/1.0.0/embedded_hal_async/) — the async `DelayNs` and other traits.
- [`embassy-sync` `Signal`](https://docs.rs/embassy-sync/0.8.0/embassy_sync/signal/struct.Signal.html) — how `signal` and `wait` work.
- [Discovery, micro:bit v2 edition](https://docs.rust-embedded.org/discovery-mb2/) — the next book to read.
- [Comprehensive Rust — Bare-metal Rust](https://google.github.io/comprehensive-rust/bare-metal.html) — a compact course on bare-metal Rust and drivers.
- [Awesome Embedded Rust](https://github.com/rust-embedded/awesome-embedded-rust) — a curated list of HALs, drivers and tools.
- [Rust Embedded Working Group](https://github.com/rust-embedded/wg) — who maintains the core crates, and how to join.

---

[🏠 Start](../START-HERE.md) · [← 22.12 Embedded track: async with Embassy](12-embedded-track-async-with-embassy.md) · [📋 Phase 22](README.md) · [22.14 Databases track: how query engines work →](14-databases-track-how-query-engines-work.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
