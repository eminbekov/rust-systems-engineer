<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 22.11 Embedded track: GPIO and embedded-hal](11-embedded-track-gpio-and-embedded-hal.md) · [📋 Phase 22](README.md) · [22.13 Embedded track: project and next steps →](13-embedded-track-project-and-next-steps.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/22-specializations/12-embedded-track-async-with-embassy.md) <!-- nav -->

# Lesson 22.12 — Embedded track: async with Embassy

> **Phase 22 — Specializations** · Lesson 12 of 30 · ⏱ about 105 minutes

## What you will learn

- Why the super loop from Lesson 22.11 does not scale, and what interrupts are.
- How Embassy runs `async` tasks on a chip without an operating system and without a heap.
- How to write and spawn tasks, wait for timers and pin edges, and pass messages with a `Channel`.
- How to log from the chip with `defmt`, and how to wait for "whatever comes first" with `select`.
- How to test async firmware logic on your Mac.

## Before you start

- You finished [Lesson 22.11 — Embedded track: GPIO and embedded-hal](11-embedded-track-gpio-and-embedded-hal.md).
- You know how futures work from [Lesson 14.10 — How futures work: Future, poll and Waker](../14-concurrency-and-async/10-how-futures-work.md)
  and built an executor in [Lesson 14.11 — Build a tiny async runtime](../14-concurrency-and-async/11-build-a-tiny-async-runtime.md).
  Channels between tasks are in [Lesson 14.13](../14-concurrency-and-async/13-tokio-tasks-and-channels.md).
- Hardware (optional): the Pico 2, the Debug Probe and the button on GPIO (General-Purpose Input/Output) pin 15 from Lesson 22.11.

## New words

| Word | Meaning |
|---|---|
| **interrupt** | A hardware signal that makes the CPU (Central Processing Unit) stop what it is doing, run a short handler function, and then continue. |
| **interrupt handler** | The function that runs for one interrupt, also called an ISR (Interrupt Service Routine). |
| **edge** | The moment a digital signal changes: a **rising edge** goes low → high, a **falling edge** goes high → low. |
| **Embassy** | A framework of crates for `async` embedded Rust: an executor, timers, synchronization, and HALs (Hardware Abstraction Layers). |
| **task** (Embassy) | An `async fn` marked `#[embassy_executor::task]` that runs concurrently with other tasks. |
| **spawn** | Start a task so that the executor runs it. |
| **`Channel`** | A fixed-size queue from `embassy-sync`; one task sends values, another receives them. |
| **critical section** | A short piece of code during which interrupts cannot run, so shared data is safe. |
| **`defmt`** | "Deferred formatting": a logging crate that sends tiny binary messages from the chip and formats them on your computer. |
| **RTT** (Real-Time Transfer) | A way to send data from the chip's RAM (Random Access Memory) to your computer through the debug probe, without extra wires. |

## 1. The problem with the super loop

The firmware of Lesson 22.11 polled the button every 10 ms in one endless loop. Now imagine you want
three things at the same time: a heartbeat LED (Light-Emitting Diode) that blinks every half second, a button that must be
noticed at once, and a log message for each press. In a super loop, every piece of code must be
written as small steps that never wait, and you must track all their states yourself. And the CPU never
sleeps, even when nothing happens.

The hardware has a better tool: **interrupts**. A peripheral can raise an interrupt when something
happens, for example "pin 15 had a falling edge" or "the timer reached this value". The CPU then
stops, runs the **interrupt handler**, and continues. Between interrupts, the CPU can sleep.

Writing everything as interrupt handlers by hand is hard: shared state, small handlers, careful
ordering. This is where `async` helps. You already know the idea from Phase 14:

- an `async fn` becomes a **future**, a state machine that the compiler builds for you;
- an **executor** calls `poll` on futures; a future that cannot continue returns `Pending`;
- a **waker** tells the executor "poll this future again".

Embassy connects this to the hardware: a future waiting for a pin registers its waker, the pin's
interrupt handler calls `wake`, and the executor polls only that task. When no task can continue, the
executor puts the CPU to sleep with the `wfe` ("wait for event") instruction.

```text
  task A: heartbeat         task B: watch_button          main task
  ---------------           --------------------          ---------
  led.toggle()              wait_for_falling_edge()       PRESSES.receive()
  Timer 500 ms  -- Pending  -- Pending                    -- Pending
        |                         |                              |
        +------------ executor: nothing ready -> CPU sleeps -----+
        |                         |                              |
  timer interrupt -> wake A   pin interrupt -> wake B            |
  poll A: toggle again        poll B: send(1) ------------> wake main, log
```

**Embassy does not need a heap.** Every task is stored in its own `static` memory, sized at compile
time. If the tasks do not fit in RAM, the linker says so; a running program cannot run out of task
memory.

## 2. A firmware with three tasks

Make a copy of the hello-LED project for this lesson and add the new crates:

```bash
cd ~/projects/rust
cp -R code/p22-hello-led code/p22-embassy
cd code/p22-embassy
rm -rf target
```

`Cargo.toml` (change the name and the dependencies):

```toml
[package]
name = "p22-embassy"
version = "0.1.0"
edition = "2024"

[dependencies]
cortex-m = "0.7"
cortex-m-rt = "0.7"
defmt = "1"
defmt-rtt = "1"
embassy-executor = "0.10"
embassy-rp = { version = "0.10", features = ["rp235xa", "critical-section-impl", "time-driver", "executor-thread", "defmt"] }
embassy-sync = "0.8"
embassy-time = "0.5"
panic-probe = { version = "1", features = ["print-defmt"] }

[profile.release]
debug = true
```

`executor-thread` makes `embassy-rp` provide an executor for the chip, `defmt` turns on log support in
the HAL, and `panic-probe` is a ready-made panic handler that prints the panic message through the probe
and stops. `defmt` needs its own linker script, so add one line at the end of `main` in `build.rs`:

```rust
    // defmt needs its own linker script for the log strings.
    println!("cargo:rustc-link-arg-bins=-Tdefmt.x");
```

and choose the log level in `.cargo/config.toml` (add at the end):

```toml
[env]
DEFMT_LOG = "info"
```

Now `src/main.rs`:

```rust
#![no_std]
#![no_main]

use defmt::info;
use embassy_executor::Spawner;
use embassy_rp::gpio::{Input, Level, Output, Pull};
use embassy_sync::blocking_mutex::raw::CriticalSectionRawMutex;
use embassy_sync::channel::Channel;
use embassy_time::Timer;
use {defmt_rtt as _, panic_probe as _};

/// Button presses travel from the button task to `main` through this channel.
static PRESSES: Channel<CriticalSectionRawMutex, u32, 4> = Channel::new();

#[embassy_executor::task]
async fn heartbeat(mut led: Output<'static>) {
    loop {
        led.toggle();
        Timer::after_millis(500).await;
    }
}

#[embassy_executor::task]
async fn watch_button(mut button: Input<'static>) {
    let mut count = 0;
    loop {
        button.wait_for_falling_edge().await;
        Timer::after_millis(20).await; // let the contact stop bouncing
        if button.is_low() {
            count += 1;
            PRESSES.send(count).await;
        }
        button.wait_for_high().await;
    }
}

#[embassy_executor::main(
    executor = "embassy_rp::executor::Executor",
    entry = "cortex_m_rt::entry"
)]
async fn main(spawner: Spawner) {
    let p = embassy_rp::init(Default::default());
    info!("started");

    spawner.spawn(heartbeat(Output::new(p.PIN_25, Level::Low)).unwrap());
    spawner.spawn(watch_button(Input::new(p.PIN_15, Pull::Up)).unwrap());

    loop {
        let n = PRESSES.receive().await;
        info!("button pressed {} times", n);
    }
}
```

**What happens:**

1. `use {defmt_rtt as _, panic_probe as _};` links the two crates without naming anything from them:
   `defmt-rtt` sends logs over RTT, and `panic-probe` provides the `#[panic_handler]`. So there is no
   panic handler in our file any more.
2. `#[embassy_executor::main(...)]` turns `async fn main` into the real entry point: it creates the
   RP2350 executor, uses `cortex_m_rt::entry` to start, and runs `main` as the first task. `Spawner`
   is the handle for starting more tasks.
3. `#[embassy_executor::task]` turns an `async fn` into a task. Calling `heartbeat(led)` does not run
   it; it returns a `Result` with a **spawn token**, or an error if that task is already running (each
   task has room for one instance by default). `unwrap()` takes the token, and `spawner.spawn(...)`
   hands it to the executor.
4. Task arguments must be `'static`: the task lives forever in its static memory, so it may not borrow
   anything from `main`'s stack. `Output<'static>` is a pin driver that owns its pin for the whole program.
5. `Timer::after_millis(500).await` returns `Pending` until the hardware timer fires. Meanwhile the
   other tasks run, or the CPU sleeps.
6. `button.wait_for_falling_edge().await` sets up the pin interrupt for a high → low change and sleeps
   until it comes. The 20 ms timer afterwards is a simple debounce: a short bounce is over by then.
   `wait_for_high` waits for the release, so holding the button counts once.
7. `PRESSES` is a `static` `Channel` that holds up to 4 `u32` values. `CriticalSectionRawMutex`
   protects it with a **critical section** (interrupts are briefly switched off), so tasks and interrupt
   handlers can use it safely. `send(...).await` waits if the channel is full; `receive().await` waits
   if it is empty.

```bash
cargo build --release
cargo size --release
```

```text
    Finished `release` profile [optimized + debuginfo] target(s) in 5.50s
   text	   data	    bss	    dec	    hex	filename
  19052	     56	   1568	  20676	   50c4	p22-embassy
```

About 19 KB of flash and 1.6 KB of RAM for an executor, a timer queue, three tasks, a channel and
logging.

> ⚡ **Performance:** compare with the super loop. There, the CPU ran all the time. Here, after each
> step the executor finds nothing to do and executes `wfe`: the core sleeps until the next interrupt.
> On a battery-powered device this is the difference between days and months.

## 3. Logging with defmt

`println!` does not exist on the chip, and formatting text is slow and needs a lot of flash. `defmt`
("deferred formatting") solves this: the format strings stay in the ELF (Executable and Linkable Format) file on your Mac, and the chip
sends only a small number (which string) plus the raw arguments. `probe-rs` reads them over **RTT**
(through the debug probe) and does the formatting on your computer.

Example output of `cargo run --release` with the board (you need the hardware; yours will look
similar). Press the button twice:

```text
      Erasing ✔ 100% [####################]  20.00 KiB @  60.12 KiB/s (took 0s)
  Programming ✔ 100% [####################]  20.00 KiB @  41.05 KiB/s (took 0s)
     Finished in 0.83s
INFO  started
└─ p22_embassy::____embassy_main_task::{async_fn#0} @ src/main.rs:43
INFO  button pressed 1 times
└─ p22_embassy::____embassy_main_task::{async_fn#0} @ src/main.rs:50
INFO  button pressed 2 times
└─ p22_embassy::____embassy_main_task::{async_fn#0} @ src/main.rs:50
```

`DEFMT_LOG = "info"` in `.cargo/config.toml` sets which levels are compiled in (`trace`, `debug`,
`info`, `warn`, `error`). Levels below it are removed at compile time and cost nothing.

> 💡 **Tip:** with `panic-probe` and `print-defmt`, a panic such as `unwrap()` on an `Err` prints the
> message and the file and line, then stops the program. This is the fastest way to find bugs on the board.

## 4. Waiting for whatever comes first: `select`

Often a task must wait for one of several things: a button press **or** a timeout. The crate
`embassy-futures` has `select`, which polls two futures and returns as soon as one of them is ready.
Add it with `cargo add embassy-futures@0.1`, then change the end of `main`:

```rust
    loop {
        match select(PRESSES.receive(), Timer::after_secs(5)).await {
            Either::First(n) => info!("button pressed {} times", n),
            Either::Second(()) => info!("no press for 5 seconds"),
        }
    }
```

and add `use embassy_futures::select::{Either, select};` at the top.

**What happens:**

1. `select(a, b)` is itself a future. Polling it polls `a` and `b`.
2. When the channel delivers a value first, the result is `Either::First(n)`; when the timer fires
   first, `Either::Second(())`.
3. The future that lost is **dropped**. Dropping `Timer` cancels it; dropping `receive()` simply stops
   waiting, and no value is lost. This is cancellation as in
   [Lesson 14.14](../14-concurrency-and-async/14-cancellation-timeouts-and-graceful-shutdown.md).

Example output after 5 seconds without a press: `INFO  no press for 5 seconds`.

## 5. Testing async logic on the Mac

Task logic can often be written as a plain `async fn` that takes a `Channel` (or a trait) as a parameter.
Then a test on the Mac can run it with a tiny executor: `embassy_futures::block_on` polls a future in a
loop until it is ready. `embassy-sync` needs a critical-section implementation; on the Mac, the
`critical-section` crate with the `std` feature provides one. Exercise 3 shows the complete setup.

> 🧪 **Testing:** keep the "what" (logic: counting, state changes, protocols) in `async fn`s with
> parameters, and keep the "how" (which pin, which timer) in `main`. Test the "what" on your Mac with
> `block_on` and channels; check the "how" on the board with `defmt` logs.

## Common mistakes

> ⚠️ **Common mistake: forgetting `.await`.** `Timer::after_millis(500);` without `.await` creates a
> future and throws it away. Nothing waits, and the LED toggles as fast as the CPU can. The compiler warns:
>
> ```text
> warning: unused `Timer` that must be used
>   --> src/main.rs:19:9
>    |
> 19 |         Timer::after_millis(500);
>    |         ^^^^^^^^^^^^^^^^^^^^^^^^
>    |
>    = note: futures do nothing unless you `.await` or poll them
> ```

> ⚠️ **Common mistake: a borrowed task argument.** A task may not borrow from `main`:
>
> ```text
> error: Arguments for tasks must live forever. Try using the `'static` lifetime.
>   --> src/main.rs:16:36
>    |
> 16 | async fn heartbeat(mut led: Output<'_>) {
>    |                                    ^^
> ```
>
> Fix: move owned values (like `Output<'static>`) into the task, or share data through `static` items
> such as a `Channel`.

> ⚠️ **Common mistake: no `defmt.x`.** Without the `-Tdefmt.x` line in `build.rs`, linking fails:
>
> ```text
>   = note: rust-lld: error: undefined symbol: _defmt_timestamp
>           rust-lld: error: undefined symbol: _defmt_panic
> ```

> ⚠️ **Common mistake: blocking inside a task.** `cortex_m::asm::delay(...)` or a long calculation
> inside a task never returns `Pending`, so **no other task runs** until it finishes, and the button
> is not noticed. In tasks, always wait with `.await` (for example `Timer::after_millis`).

## Try it yourself

1. Change the heartbeat period from 500 ms to 100 ms and build. On the board, the LED flickers fast,
   and the button still works at once: the tasks do not slow each other down.
2. Set `DEFMT_LOG = "warn"` and build again. The `info!` lines are removed, and `cargo size --release`
   shows `text` going down from 19052 to 18956 bytes. Set it back to `"info"`.

## Exercises

### Exercise 1 — Double blink (easy)

Change `heartbeat` so the LED blinks twice quickly (on 100 ms, off 100 ms, two times) and then stays
off for 600 ms, like a heartbeat. Use `set_high` and `set_low`.

<details>
<summary>Hint</summary>

A `for _ in 0..2` loop inside the `loop`, and one more `Timer` after it.

</details>

<details>
<summary>Answer</summary>

```rust
#[embassy_executor::task]
async fn heartbeat(mut led: Output<'static>) {
    loop {
        for _ in 0..2 {
            led.set_high();
            Timer::after_millis(100).await;
            led.set_low();
            Timer::after_millis(100).await;
        }
        Timer::after_millis(600).await;
    }
}
```

Why it works: each `.await` gives the CPU back to the executor, so the button task keeps working
during the pattern. One cycle takes 1 second.

</details>

### Exercise 2 — A second LED (medium)

Wire an LED with a 330 Ω resistor to GPIO 16 (Lesson 22.11, Section 1). In `main`, create
`Output::new(p.PIN_16, Level::Low)` and make it light for 200 ms after each button press, while the
heartbeat continues on GPIO 25. Build with `cargo build --release`.

<details>
<summary>Hint</summary>

`main` is a task too. After `PRESSES.receive().await`, set the pin high, wait with `Timer`, set it low.

</details>

<details>
<summary>Answer</summary>

```rust
    let mut press_led = Output::new(p.PIN_16, Level::Low);
    loop {
        let n = PRESSES.receive().await;
        info!("button pressed {} times", n);
        press_led.set_high();
        Timer::after_millis(200).await;
        press_led.set_low();
    }
```

Why it works: `p.PIN_16` has not been moved yet, so `main` can own it. While `main` waits 200 ms, the
heartbeat and button tasks still run. A press during those 200 ms waits in the channel (it holds 4
values) and is logged afterwards.

</details>

### Exercise 3 — Test task logic on the Mac (hard)

Create a library `code/p22-presses` (`no_std`) with
`pub async fn wait_for_presses<M: RawMutex, const N: usize>(presses: &Channel<M, u32, N>, goal: u32) -> u32`,
which receives counters until one is at least `goal` and returns it. **Write a test** that puts 1, 2, 3, 4
into a channel with `try_send`, runs the function with `block_on` and goal 3, and checks that it returns
3 and that 4 is still in the channel.

<details>
<summary>Hint</summary>

Dependencies: `cargo add embassy-sync@0.8`, then `cargo add --dev embassy-futures@0.1` and
`cargo add --dev critical-section@1 -F std` (the `std` feature gives a critical section for the Mac;
`-F` means `--features`). Generic over `M: RawMutex`, the function works with any mutex kind.

</details>

<details>
<summary>Answer</summary>

```rust
#![no_std]

use embassy_sync::blocking_mutex::raw::RawMutex;
use embassy_sync::channel::Channel;

/// Waits until the press counter reaches at least `goal` and returns that counter.
pub async fn wait_for_presses<M: RawMutex, const N: usize>(
    presses: &Channel<M, u32, N>,
    goal: u32,
) -> u32 {
    loop {
        let count = presses.receive().await;
        if count >= goal {
            return count;
        }
    }
}

#[cfg(test)]
mod tests {
    use super::*;
    use embassy_futures::block_on;
    use embassy_sync::blocking_mutex::raw::CriticalSectionRawMutex;

    #[test]
    fn stops_at_the_goal() {
        let ch: Channel<CriticalSectionRawMutex, u32, 4> = Channel::new();
        for n in 1..=4 {
            ch.try_send(n).unwrap();
        }
        let got = block_on(wait_for_presses(&ch, 3));
        assert_eq!(got, 3);
        // The fourth press is still waiting in the channel.
        assert_eq!(ch.try_receive().unwrap(), 4);
    }
}
```

```text
running 1 test
test tests::stops_at_the_goal ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

Why it works: `block_on` is a minimal executor that polls the future until it is `Ready`. The values
are already in the channel, so `receive()` never has to wait. The same function, unchanged, can be
called from a task on the chip with the `static PRESSES` channel.

</details>

## Quiz

1. What does the Embassy executor do when no task can continue?

<details>
<summary>Answer</summary>

It puts the CPU to sleep (`wfe`) until an interrupt wakes a task. Then it polls only the woken task.

</details>

2. How does `button.wait_for_falling_edge().await` know when to continue?

<details>
<summary>Answer</summary>

The future enables the pin interrupt for a falling edge and stores its waker. When the edge happens,
the interrupt handler wakes the task, and the executor polls it again; now the future is `Ready`.

</details>

3. Why must task arguments be `'static`?

<details>
<summary>Answer</summary>

A task lives in its own static memory and may run forever, longer than any function's stack frame.
Borrowed data from `main` could disappear while the task still uses it.

</details>

4. What is the advantage of `defmt` over formatting text on the chip?

<details>
<summary>Answer</summary>

The chip sends only an index of the format string and raw arguments; the strings stay on the computer.
This needs much less flash and CPU time, and unused log levels are removed at compile time.

</details>

5. In `select(PRESSES.receive(), Timer::after_secs(5))`, what happens to the timer when a press
   arrives first?

<details>
<summary>Answer</summary>

`select` returns `Either::First(n)` and drops the other future. Dropping the timer future cancels it.

</details>

## Summary

- A super loop wastes power and gets complex; interrupts plus `async` let tasks sleep until hardware
  wakes them.
- Embassy's executor polls tasks, sleeps with `wfe` when idle, and stores every task statically: no heap.
- Tasks are `async fn`s with `#[embassy_executor::task]`, started with `spawner.spawn(task(...).unwrap())`;
  their arguments must be `'static`.
- `Timer::after_millis`, `wait_for_falling_edge` and `Channel::receive` are futures that hardware
  interrupts complete.
- `defmt` logs over RTT through the probe; `select` waits for the first of two futures and cancels the other.
- Test task logic as plain `async fn`s on the Mac with `block_on`.

## Sources and further reading

- [The Embassy Book](https://embassy.dev/book/) — the official guide: executor, tasks, time, HALs.
- [`embassy-executor` on docs.rs](https://docs.rs/embassy-executor/0.10.0/embassy_executor/) — tasks, spawners and the main macro.
- [`embassy-sync` on docs.rs](https://docs.rs/embassy-sync/0.8.0/embassy_sync/) — `Channel`, `Signal`, `Mutex` and more.
- [`embassy-time` on docs.rs](https://docs.rs/embassy-time/0.5.1/embassy_time/) — `Timer`, `Duration`, `Instant`.
- [The defmt book](https://defmt.ferrous-systems.com/) — how deferred formatting and log levels work.

---

[🏠 Start](../START-HERE.md) · [← 22.11 Embedded track: GPIO and embedded-hal](11-embedded-track-gpio-and-embedded-hal.md) · [📋 Phase 22](README.md) · [22.13 Embedded track: project and next steps →](13-embedded-track-project-and-next-steps.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
