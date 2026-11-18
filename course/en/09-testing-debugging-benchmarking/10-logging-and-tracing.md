<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 09.9 Debugging in depth: LLDB and RustRover](09-debugging-with-lldb-and-rustrover.md) · [📋 Phase 09](README.md) · [09.11 Benchmarking in depth →](11-benchmarking-in-depth.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/09-testing-debugging-benchmarking/10-logging-and-tracing.md) <!-- nav -->

# Lesson 09.10 — Logging and tracing

> **Phase 09 — Testing, debugging and benchmarking** · Lesson 10 of 14 · ⏱ about 105 minutes

## What you will learn

- Why `println!` is not enough for a real program, and what **log levels**, **events**, **fields** and **spans** are.
- How to use the `tracing` crate and print the events with `tracing-subscriber`.
- How to turn logging up and down without recompiling, with the `RUST_LOG` environment variable.
- How `#[instrument]` records the arguments and the result of a function, and how to measure the time a function takes.
- How to see logs in tests, and what you must never write into a log.

## Before you start

- You finished [Lesson 09.9 — Debugging in depth: LLDB and RustRover](09-debugging-with-lldb-and-rustrover.md). A debugger needs a debug build and you sitting in front of it; logs are what you have when you are not.
- You used print debugging in [Lesson 02.9](../02-programming-basics/09-debugging-basics.md) and you know `Result` and errors ([Lesson 04.10](../04-structs-enums-collections-errors/10-error-handling.md)).
- You know how to add a dependency with `cargo add` ([Lesson 05.8](../05-rust-intermediate/08-cargo-dependencies-and-versions.md)) and how attribute macros like `#[derive(Debug)]` work.
- Your code goes in `~/projects/rust/code/p09-tracing`.

## New words

| Word | Meaning |
|---|---|
| **log** | A record of what a program did, written while it runs, so you can read it later. |
| **log level** | How important a message is: `ERROR`, `WARN`, `INFO`, `DEBUG` or `TRACE`. You choose which levels to show. |
| **event** | One thing that happened at one moment: "order received". In `tracing`, `info!(...)` creates an event. |
| **field** | A named value attached to an event or a span, such as `total=2500`. Fields make logs searchable. |
| **span** | A period of time, with a start and an end, that has a name and fields, for example "processing order 2". Every event inside it is printed together with the span. |
| **subscriber** | The part of the program that receives events and spans and decides what to do with them: print them, write JSON, count them. |
| **`tracing`** | The crate with the macros (`info!`, `#[instrument]`, …) that your code calls. It does not print anything by itself. |
| **`tracing-subscriber`** | The crate with ready-made subscribers, such as one that prints lines to the terminal. |
| **environment variable** | A named setting that the shell gives to a program, for example `RUST_LOG=debug`. |
| **structured logging** | Logging where each event is data (fields), not only text, so a tool can filter and count them. |
| **JSON** (JavaScript Object Notation) | A text format for data made of `{"name": value}` pairs, which many tools can read. |

## 1. Why not just `println!`?

`println!` works for a ten-minute bug. For a program that runs for months, it has problems:

1. **You cannot switch it off.** Every debug `println!` is there for everybody, always, and the output of your real program mixes with it.
2. **No importance.** A line "disk is full" and a line "x = 3" look the same.
3. **No context.** If a server handles 100 orders at once, a line "charged" does not tell you *which* order.
4. **It is text only.** A tool cannot ask "show me all lines with `order_id=2`" unless you format the line very carefully.

A **logging** library solves all four. The important idea of the `tracing` crate: the code only *describes* what happens, with macros. A *subscriber*, chosen once in `main`, decides how to show it.

```text
 your code and libraries                 chosen once, in main
 ┌─────────────────────────┐   events   ┌──────────────────────────┐
 │ info!(total = 2500, ..) │ ─────────▶ │ subscriber               │ ──▶ terminal lines
 │ #[instrument] fn charge │   spans    │  + filter (RUST_LOG)     │ ──▶ JSON for a tool
 └─────────────────────────┘            └──────────────────────────┘
```

The five levels, from most to least important:

| Level | Use it for | Example |
|---|---|---|
| `ERROR` | Something failed and the work could not be done | "could not open the database" |
| `WARN` | Something is wrong, but the program continues | "disk is 91% full" |
| `INFO` | Normal important events, the story of the program | "server started" |
| `DEBUG` | Details for developers | "computed total" |
| `TRACE` | Very detailed steps, probably too many | "entering loop, i = 3" |

## 2. Your first events

Create the project and add the two crates. The feature `env-filter` of `tracing-subscriber` gives us the `RUST_LOG` filter that Section 5 uses:

```bash
cd ~/projects/rust/code
cargo new p09-tracing
cd p09-tracing
cargo add tracing@0.1
cargo add tracing-subscriber@0.3 --features env-filter
```

`Cargo.toml` now contains:

```toml
[dependencies]
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter"] }
```

Put this in `src/main.rs`:

```rust
use tracing::{debug, error, info, trace, warn};

fn main() {
    tracing_subscriber::fmt().without_time().with_ansi(false).init();

    trace!("very detailed step");
    debug!("a value for developers");
    info!("server started");
    warn!("disk is 91% full");
    error!("could not open the database");
}
```

```bash
cargo run
```

```text
 INFO p09_tracing: server started
 WARN p09_tracing: disk is 91% full
ERROR p09_tracing: could not open the database
```

**What happens:**

1. `tracing_subscriber::fmt()` creates the builder of the "print lines" subscriber, and `.init()` installs it as the subscriber of the whole program. This must happen once, at the start of `main`.
2. `.without_time()` removes the time stamp, and `.with_ansi(false)` removes the colours, so that the book output is short and plain. In a real terminal you want both: the time and the colours.
3. The macros `trace!`, `debug!`, `info!`, `warn!` and `error!` create events with the given level.
4. Only `INFO` and above were shown. The builder starts with the maximum level `INFO`, so `trace!` and `debug!` do nothing (they cost almost nothing, see Section 8). `p09_tracing` is the **target**: the module that wrote the event.

To see everything, you can raise the level in the code with `.with_max_level(tracing::Level::TRACE)`. Better, as Section 5 shows, you can change it from outside without recompiling.

> ⚡ **Performance:** An event whose level is switched off is skipped very early: the values in `info!(total = expensive(), ...)` are **not** computed when `INFO` is off. This is why you can leave `debug!` and `trace!` lines in the code permanently, unlike `println!`.

## 3. Fields: data instead of text

Do not build a sentence with `format!`. Give the values as **fields** after the message or before it:

```rust
use tracing::info;

fn main() {
    tracing_subscriber::fmt().with_ansi(false).without_time().init();
    let name = "Ada";
    let list = vec![1, 2];
    info!(name, "plain value");
    info!(name = %name, "display");
    info!(name = ?name, list = ?list, "debug");
}
```

```text
 INFO p09_tracing: plain value name="Ada"
 INFO p09_tracing: display name=Ada
 INFO p09_tracing: debug name="Ada" list=[1, 2]
```

**What happens:**

1. `info!(name, "...")` records the variable `name` as a field with the same name. A plain `&str` is recorded as text and shown in quotes.
2. `name = %name` records the value with its **`Display`** form (the form of `{}` in `println!`, Lesson 05.2): no quotes.
3. `name = ?name` records the value with its **`Debug`** form (the form of `{:?}`). Types such as `Vec` or your own `struct` have no `Display`, so you use `?` for them.
4. The message is the last argument, a plain string. Keep it a **constant** sentence: "order received". The variable parts go into fields. A log tool can then count all events with the message "order received".

> 📏 **Convention:** Message = what happened, in lowercase, without the values. Fields = the values: `info!(order_id, total_cents, "order accepted")`. Use the same field names everywhere (`order_id`, not `id` in one place and `orderId` in another).

## 4. Spans and `#[instrument]`

An event says "something happened". A **span** says "for the duration of this work, remember these fields". All events inside the span carry them.

The easiest way to create a span is the attribute `#[instrument]` on a function: it creates a span named after the function, which lasts for the call, and records the arguments as fields. Here is the program that we will use for the rest of the lesson. Replace `src/main.rs`:

```rust
use tracing::{debug, info, instrument, warn};
use tracing_subscriber::EnvFilter;

#[derive(Debug)]
struct Order {
    id: u32,
    customer: String,
    prices_cents: Vec<u64>,
}

#[instrument(skip(order), fields(order_id = order.id), err)]
fn charge(order: &Order, limit_cents: u64) -> Result<u64, String> {
    let total: u64 = order.prices_cents.iter().sum();
    debug!(total, items = order.prices_cents.len(), "computed total");
    if total > limit_cents {
        warn!(total, limit_cents, "total is over the limit");
        return Err(format!("order {} is over the limit", order.id));
    }
    info!(total, "charged");
    Ok(total)
}

#[instrument(skip_all, fields(order_id = order.id))]
fn process(order: Order) {
    info!(customer = %order.customer, "order received");
    match charge(&order, 5_000) {
        Ok(total) => info!(total, "order done"),
        Err(_) => info!("order rejected"),
    }
}

fn main() {
    tracing_subscriber::fmt()
        .with_env_filter(EnvFilter::from_default_env())
        .with_ansi(false)
        .without_time()
        .init();

    process(Order {
        id: 1,
        customer: "Ada".to_string(),
        prices_cents: vec![1_000, 1_500],
    });
    process(Order {
        id: 2,
        customer: "Bob".to_string(),
        prices_cents: vec![4_000, 2_500],
    });
}
```

**What happens:**

1. `#[instrument]` wraps the function: when it is called, a span with the function's name is entered, and when it returns, the span is left. Every event in the function, and in the functions it calls, is printed with the names and fields of the spans around it.
2. By default `#[instrument]` records **every argument** with `Debug`. `skip(order)` leaves `order` out, because the whole `Order` is large (and may hold private data, see Section 8). `skip_all` leaves out all arguments. `fields(order_id = order.id)` adds one chosen field of our own.
3. `err` means: if the function returns `Err`, log the error as an `ERROR` event automatically. It works for functions that return `Result` whose error type has `Display`.
4. `.with_env_filter(EnvFilter::from_default_env())` makes the level depend on the `RUST_LOG` variable (next section). Without `RUST_LOG`, only `ERROR` is shown.

Run it with `RUST_LOG=info`. The syntax `NAME=value command` sets the variable only for that one command:

```bash
RUST_LOG=info cargo run
```

```text
 INFO process{order_id=1}: p09_tracing: order received customer=Ada
 INFO process{order_id=1}:charge{limit_cents=5000 order_id=1}: p09_tracing: charged total=2500
 INFO process{order_id=1}: p09_tracing: order done total=2500
 INFO process{order_id=2}: p09_tracing: order received customer=Bob
 WARN process{order_id=2}:charge{limit_cents=5000 order_id=2}: p09_tracing: total is over the limit total=6500 limit_cents=5000
ERROR process{order_id=2}:charge{limit_cents=5000 order_id=2}: p09_tracing: error=order 2 is over the limit
 INFO process{order_id=2}: p09_tracing: order rejected
```

**How to read it:**

1. The text before the second colon is the **span path**: `process{order_id=2}:charge{limit_cents=5000 order_id=2}` means "inside `process` for order 2, inside `charge`". The curly braces hold the fields of each span.
2. Because of the spans, the line `total is over the limit` tells us **which order** it is about, although `warn!` itself does not mention it. That was problem 3 from Section 1.
3. The `ERROR` line was created by `err`, and we did not write it. The program continues after it (`order rejected`).
4. The order 1 lines show the happy path: three `INFO` lines, no warnings.

## 5. Choosing what to see with `RUST_LOG`

The `EnvFilter` reads the variable `RUST_LOG` when the program starts. You change the amount of logging without changing the code:

```bash
cargo run                     # no RUST_LOG: only ERROR
RUST_LOG=warn cargo run       # WARN and ERROR
RUST_LOG=debug cargo run      # everything from DEBUG up
```

```text
ERROR p09_tracing: error=order 2 is over the limit
```

```text
 WARN p09_tracing: total is over the limit total=6500 limit_cents=5000
ERROR p09_tracing: error=order 2 is over the limit
```

With `RUST_LOG=debug` the extra `DEBUG` lines appear, for example:

```text
DEBUG process{order_id=1}:charge{limit_cents=5000 order_id=1}: p09_tracing: computed total total=2500 items=2
```

(Notice that with `RUST_LOG=warn` the span names are gone from the lines. The spans made by `#[instrument]` have the level `INFO`, so `warn` switches them off, and they are no longer part of the printed context. Use at least `info` when you need to know which order a line belongs to.)

The value of `RUST_LOG` is a comma-separated list of **directives**. A directive is `level` or `target=level`:

| `RUST_LOG` | Meaning |
|---|---|
| `info` | Everything at `INFO` and above, from all crates |
| `warn,p09_tracing=debug` | `WARN` for everything, but `DEBUG` for our own crate |
| `p09_tracing=trace` | Only our crate, at every level; dependencies stay at the default (`ERROR`) |
| `debug,hyper=warn` | `DEBUG` everywhere, but quiet for the crate `hyper` |

This is the main use of targets: when a library (a dependency) is too chatty, you silence it with one directive and keep full detail for your own code.

> 💡 **Tip:** Keep a good default for the case when `RUST_LOG` is not set. Instead of `EnvFilter::from_default_env()` write `EnvFilter::try_from_default_env().unwrap_or_else(|_| EnvFilter::new("info"))`. Then the program shows `INFO` by default and the variable only overrides it.

## 6. How long did it take? Spans as stopwatches

A span has a start and an end, so the subscriber can measure its duration. Add `.with_span_events(FmtSpan::CLOSE)` to the builder (and the import `use tracing_subscriber::fmt::format::FmtSpan;`). Then every span prints one extra event when it closes. This time, keep the time stamp, because the timing fields are shown with it:

```text
2026-11-18T06:40:57.072353Z  INFO process{order_id=2}:charge{limit_cents=5000 order_id=2}: p09_tracing: close time.busy=13.4µs time.idle=1.50µs
2026-11-18T06:40:57.072364Z  INFO process{order_id=2}: p09_tracing: close time.busy=32.7µs time.idle=1.54µs
```

(Example output; your numbers will differ.) `time.busy` is the time the code was running inside the span, and `time.idle` the time when the span was open but the code was waiting (this matters for asynchronous code, [Lesson 06.9](../06-rust-advanced/09-async-await-introduction.md)). Here `charge` took 13.4 microseconds, and the whole `process` took 32.7 microseconds. It is a cheap way to see which part of the program is slow before you reach for a benchmark (Lesson 09.11).

For machines instead of people, the same events can be printed as one **JSON** object per line, which log tools can search. Add the feature `json` (`cargo add tracing-subscriber@0.3 --features env-filter,json`) and call `.json()` instead of `.with_ansi(false)`:

```text
{"level":"WARN","fields":{"message":"total is over the limit","total":6500,"limit_cents":5000},"target":"p09_tracing","span":{"limit_cents":5000,"order_id":2,"name":"charge"},"spans":[{"order_id":2,"name":"process"},{"limit_cents":5000,"order_id":2,"name":"charge"}]}
```

Every field is a separate key, so a tool can answer "show all `WARN` events for `order_id` 2" without parsing a sentence. This is **structured logging**.

## 7. Logs in tests

When a test fails, a log of what happened inside is often the fastest explanation. `tracing-subscriber` has `with_test_writer()`, which sends the lines through the test harness' output capture, so they are shown only for tests that fail (or with `--nocapture`). Add this at the end of `src/main.rs`:

```rust
#[cfg(test)]
mod tests {
    use super::*;

    fn init_logs() {
        // `try_init` returns an error if a subscriber already exists; ignore it,
        // because every test calls this function.
        let _ = tracing_subscriber::fmt()
            .with_env_filter(EnvFilter::from_default_env())
            .with_ansi(false)
            .with_test_writer()
            .without_time()
            .try_init();
    }

    #[test]
    fn small_order_is_charged() {
        init_logs();
        let order = Order {
            id: 7,
            customer: "Test".to_string(),
            prices_cents: vec![100, 200],
        };
        assert_eq!(charge(&order, 1_000), Ok(300));
    }

    #[test]
    fn big_order_is_rejected() {
        init_logs();
        let order = Order {
            id: 8,
            customer: "Test".to_string(),
            prices_cents: vec![9_000],
        };
        assert!(charge(&order, 1_000).is_err());
    }
}
```

**What happens:**

1. A subscriber can be installed **only once** per process. All tests of the crate run in one process (on different threads), so each test calls `init_logs()`, and `try_init()` quietly does nothing when the subscriber is already set.
2. `with_test_writer()` is what makes the lines belong to the test that printed them.
3. Normal `cargo test` shows nothing extra. `RUST_LOG=debug cargo test -- --nocapture` shows the logs of all tests:

```bash
cargo test
```

```text
test tests::small_order_is_charged ... ok
test tests::big_order_is_rejected ... ok
test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

> 🧪 **Testing:** Put `RUST_LOG=debug cargo test failing_test_name` in your tool box. When a test fails, its log is printed with the failure report, and you see the steps that led to it without adding a single `println!`.

## 8. What to log, and what never to log

1. **Libraries use only the macros; applications install the subscriber.** If a library called `tracing_subscriber::fmt().init()` itself, it would break the choice of the program that uses it. Only `main` (or a test) calls `init`.
2. **Log decisions and boundaries**, not every line: a request arrives, an order is accepted, a retry happens, a connection fails. If you log everything at `INFO`, the important lines disappear in noise.
3. **Pick the level honestly.** `ERROR` for failures that need a person to look at them. If every rejected order is `ERROR`, nobody reads `ERROR` any more.
4. **Do not log the same failure twice.** In our program `err` already logs the error, so `process` only says "order rejected" at `INFO`.
5. **Do not use logs to replace metrics**: "how many orders per second?" is a number to count, not a line to grep (a later phase covers metrics).

> 🔒 **Security:** **Never log secrets or personal data**: passwords, tokens, API (Application Programming Interface) keys, card numbers, full request bodies. Logs are copied to many places, kept for months and read by many people. Note that `#[instrument]` with no `skip` writes **every argument** with `Debug`: a function `login(user: &str, password: &str)` would print the password. Use `skip` or `skip_all`, and name the safe fields yourself. Also be careful with text from outside: a user can send a newline and fake a log line. With structured fields (`name = %input`) the value stays inside its field, and the JSON format escapes it.

> ⚡ **Performance:** Logging that is *enabled* costs real time: formatting, locking and writing. Do not log inside the innermost loop of a hot path at `INFO`. Use `trace!` there, which is switched off by default.

## Common mistakes

**1. No subscriber installed.** If `main` never calls `.init()`, all events are silently dropped and you see nothing. Add the line at the top of `main`.

**2. Calling `init()` twice.** `.init()` panics when a subscriber is already installed (for example once in `main` and once in a library function). In tests use `try_init()`.

**3. `RUST_LOG` set, but nothing changes.** `EnvFilter` is used only if you pass it with `.with_env_filter(...)`. Without that call (as in Section 2), `RUST_LOG` is ignored.

**4. Putting the value inside the message.** `info!("charged {total} cents")` is just text, and a tool cannot filter on it. Write `info!(total, "charged")`.

**5. `#[instrument]` on a function with a large or secret argument.** Use `skip` or `skip_all`.

## Try it yourself

1. Create `code/p09-tracing` with the program of Section 4 and run it with no `RUST_LOG`, with `RUST_LOG=warn`, with `RUST_LOG=info` and with `RUST_LOG=debug`. Compare the number of lines (`... 2>&1 | wc -l`).
2. Add `.with_span_events(FmtSpan::CLOSE)` and keep the time stamp. Which is slower, `charge` or `process`, and why?
3. Add the two tests of Section 7 and run `RUST_LOG=debug cargo test -- --nocapture --test-threads=1`.

## Exercises

### Exercise 1 — A directive (easy)

Which `RUST_LOG` value shows `WARN` and `ERROR` events for all crates but `DEBUG` events (and above) only for the crate `p09_tracing`? Write it, then run the program with it. How many lines does it print?

<details>
<summary>Hint</summary>

A directive is `target=level`; several directives are separated by commas. A directive without a target sets the default level.

</details>

<details>
<summary>Answer</summary>

```bash
RUST_LOG=warn,p09_tracing=debug cargo run
```

It prints the full `DEBUG` output, 9 lines. Our crate is at `debug`, so all its events are shown; `warn` applies to everything else and no other crates log anything in this program.

Why it works: the filter picks the most specific directive that matches an event's target. Our events have the target `p09_tracing`, which matches `p09_tracing=debug`.

</details>

### Exercise 2 — A boundary test (medium)

Write a test that an order whose total is **exactly** the limit is accepted (`charge(&order, 1_000)` with prices `[400, 600]` must return `Ok(1000)`). Which comparison in `charge` does this test protect?

<details>
<summary>Hint</summary>

Put the test next to the other tests in `mod tests` and call `init_logs()` first. The code says `if total > limit_cents`.

</details>

<details>
<summary>Answer</summary>

```rust
    #[test]
    fn order_exactly_at_the_limit_is_charged() {
        init_logs();
        let order = Order {
            id: 9,
            customer: "Test".to_string(),
            prices_cents: vec![400, 600],
        };
        assert_eq!(charge(&order, 1_000), Ok(1_000));
    }
```

Why it works: the total is 1000, and `1000 > 1000` is false, so the order is charged. The test protects the `>` against being changed into `>=` by mistake (an off-by-one error). If someone makes that change, this test fails and its log shows the `WARN` line.

</details>

### Exercise 3 — A leaking span (hard)

A colleague writes `#[instrument]` (without arguments) on `fn process(order: Order)`. Run the program with `RUST_LOG=info` and look at the first line. What leaks, and what do you write instead?

<details>
<summary>Hint</summary>

`#[instrument]` records every argument with `Debug`, and `Order` derives `Debug`. Compare with `skip_all`.

</details>

<details>
<summary>Answer</summary>

The first line becomes:

```text
 INFO process{order=Order { id: 1, customer: "Ada", prices_cents: [1000, 1500] }}: p09_tracing: order received customer=Ada
```

The complete order (the customer's name and every price) is printed in every line of every event inside `process`. Instead write:

```rust
#[instrument(skip_all, fields(order_id = order.id))]
```

and the span becomes the short `process{order_id=1}`. Why it works: `skip_all` stops the automatic recording of the arguments, and `fields(...)` adds only the values that you decided are safe and useful.

</details>

## Quiz

1. What is the difference between an event and a span?

<details>
<summary>Answer</summary>

An event is one moment ("order received"). A span is a period of time with a name and fields; the events inside it are printed with its context, and its duration can be measured.

</details>

2. Why is it safe to leave `debug!` lines in the code?

<details>
<summary>Answer</summary>

When the level is switched off, the macro skips the event early, before it computes the field values or formats anything, so the cost is tiny. You can turn the lines on later with `RUST_LOG=debug`, without recompiling.

</details>

3. What does `info!(total = ?x, "done")` do differently from `info!(total = %x, "done")`?

<details>
<summary>Answer</summary>

`?` records the value with its `Debug` form (a `&str` gets quotes); `%` records it with its `Display` form (no quotes). Types without `Display` need `?`.

</details>

4. Who should call `.init()` to install the subscriber: a library or the program?

<details>
<summary>Answer</summary>

The program (`main`, or a test). A library only uses the macros, so the program decides where the output goes.

</details>

5. `RUST_LOG=warn` hides the span names in the lines. Why?

<details>
<summary>Answer</summary>

The spans made by `#[instrument]` have the level `INFO`. With `warn` they are disabled, so they are not part of the printed context.

</details>

6. Name two things that must never be written to a log.

<details>
<summary>Answer</summary>

Passwords, tokens, API keys, card numbers or other personal data. Also be careful with unchecked input that could fake log lines; structured fields keep it inside its field.

</details>

## Summary

- Logs are for when you cannot attach a debugger: a release build, a long run, a server. `tracing` lets the code describe events and spans; `tracing-subscriber` decides how to print them.
- Five levels: `ERROR`, `WARN`, `INFO`, `DEBUG`, `TRACE`. Disabled levels are almost free.
- Put values in fields (`info!(total, "charged")`), not in the message. `%` means `Display`, `?` means `Debug`.
- `#[instrument]` makes a span for a function; use `skip`, `skip_all`, `fields(...)` and `err`.
- `EnvFilter` with `RUST_LOG` turns logging up or down without recompiling: `warn,p09_tracing=debug`.
- Span close events give timings; JSON output gives data for tools; `with_test_writer()` shows logs of failing tests.
- Only the program installs the subscriber. Never log secrets.

## Sources and further reading

- [tracing on docs.rs](https://docs.rs/tracing/latest/tracing/) — the macros, spans, fields and the rules of levels.
- [tracing-subscriber on docs.rs](https://docs.rs/tracing-subscriber/latest/tracing_subscriber/) — the `fmt` subscriber, `EnvFilter` directive syntax, JSON output and `with_test_writer`.
- [The tracing project on GitHub](https://github.com/tokio-rs/tracing) — the examples folder, and the explanation of why `tracing` replaced simple logging for asynchronous programs.
- [The `log` crate on docs.rs](https://docs.rs/log/latest/log/) — the older, simpler logging facade that many libraries still use; `tracing` can receive its messages too.

---

[🏠 Start](../START-HERE.md) · [← 09.9 Debugging in depth: LLDB and RustRover](09-debugging-with-lldb-and-rustrover.md) · [📋 Phase 09](README.md) · [09.11 Benchmarking in depth →](11-benchmarking-in-depth.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
