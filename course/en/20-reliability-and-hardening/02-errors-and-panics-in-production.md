<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 20.1 What reliability means](01-what-reliability-means.md) · [📋 Phase 20](README.md) · [20.3 Fuzzing in depth →](03-fuzzing-in-depth.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/20-reliability-and-hardening/02-errors-and-panics-in-production.md) <!-- nav -->

# Lesson 20.2 — Errors and panics in production

> **Phase 20 — Reliability and hardening** · Lesson 2 of 11 · ⏱ about 135 minutes

## What you will learn

- The difference between an expected error and a bug, and why only bugs should panic.
- What a panic does to a thread, to a `Mutex` and to a Tokio task, and how to keep one panic from taking the whole service down.
- How to make every panic visible with a *panic hook*: one structured log line and a counter.
- When to choose `panic = "unwind"` and when `panic = "abort"`, and what each one means for cleanup and restarts.
- How Clippy lints keep `unwrap`, `expect` and indexing out of production code.

## Before you start

- You finished [Lesson 20.1 — What reliability means](01-what-reliability-means.md).
- You know `Result`, `?` and `panic!` from [Lesson 04.10 — Error handling](../04-structs-enums-collections-errors/10-error-handling.md), and why an attacker likes panics from [Lesson 10.12 — Memory safety and reviewing unsafe code](../10-security-fundamentals/12-memory-safety-and-reviewing-unsafe.md).
- You know threads and `Mutex` from [Lesson 05.10](../05-rust-intermediate/10-threads-channels-and-shared-state.md), and Tokio tasks from [Lesson 14.13 — Tokio in depth](../14-concurrency-and-async/13-tokio-tasks-and-channels.md).
- Create the package for this lesson:

```bash
cd ~/projects/rust
cargo new --lib code/p20-panics
cd code/p20-panics
cargo add tokio@1 --features rt-multi-thread,macros
mkdir src/bin
```

## New words

| Word | Meaning |
|---|---|
| **expected error** | A problem that is a normal part of the job: a missing file, bad user input, a timeout. It is returned as `Err` and handled. |
| **bug** | A mistake in the program: something the code assumed can never happen did happen. In Rust it usually shows up as a panic. |
| **unwinding** | What Rust does after a panic by default: it walks back up the stack of the panicking thread and runs `Drop` for every value on the way. |
| **abort** | Ending the whole process at once, without unwinding and without running `Drop`. |
| **poisoned lock** | A `Mutex` whose holder panicked while holding it. Later `lock()` calls return `Err(PoisonError)` to warn that the data may be half-changed. |
| **panic payload** | The value that a panic carries, usually the message as `&str` or `String`. |
| **panic hook** | A function that Rust calls on every panic, before unwinding starts. The default hook prints the `thread '…' panicked at …` message. |
| **`catch_unwind`** | A function in `std::panic` that runs a closure and returns `Err(payload)` instead of letting a panic leave the closure. |
| **isolation** | Keeping parts of a program apart, so that a failure in one part does not spread into the others. |
| **supervisor** | A program that starts another program and restarts it when it exits, such as systemd (Lesson 11.6) or Kubernetes (Phase 21). |
| **lint** | An automatic check of source code for patterns that are often mistakes. Clippy is Rust's main linter. |

## 1. Errors and bugs are different things

A service meets two kinds of trouble:

- **Expected errors.** A user sends a name that is too long. A database query times out. A file is missing. These are part of normal life, so the code returns `Err(...)`, the caller decides what to do, and the user gets a clear answer such as "400 Bad Request".
- **Bugs.** The code reads element 4 of a list with four elements. It divides by a count that is zero. Something that "can never happen" happened. Rust stops the current thread with a **panic**, because continuing with impossible data could make things worse.

The rule for production code follows from this: **every problem that can happen with valid use is an `Err`. A panic means there is a bug to fix.** In Lesson 04.10 you learned the tools for the first kind. This lesson is about the second kind: bugs *will* reach production, and a reliable service must survive them, report them and keep serving everyone else.

## 2. What a panic does

By default a panic **unwinds** only the thread in which it happens. Other threads keep running. Create `src/bin/thread_panic.rs`:

```rust
use std::sync::{Arc, Mutex};
use std::thread;

fn main() {
    let orders = Arc::new(Mutex::new(vec![10, 20, 30]));

    let worker_orders = Arc::clone(&orders);
    let worker = thread::spawn(move || {
        let mut list = worker_orders.lock().unwrap();
        list.push(40);
        let fifth = list[4]; // BUG: there are only four orders
        println!("fifth order: {fifth}");
    });

    let result = worker.join();
    println!("worker finished with an error: {}", result.is_err());

    match orders.lock() {
        Ok(list) => println!("orders: {list:?}"),
        Err(poisoned) => {
            println!("the lock is poisoned");
            println!("orders anyway: {:?}", poisoned.into_inner());
        }
    }
}
```

```bash
cargo run -q --bin thread_panic
```

```text
thread '<unnamed>' (5911757) panicked at src/bin/thread_panic.rs:11:25:
index out of bounds: the len is 4 but the index is 4
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
worker finished with an error: true
the lock is poisoned
orders anyway: [10, 20, 30, 40]
```

**What happens:**

1. The worker locks the list, adds `40`, and reads index 4 of a list with four elements. That panics, and the worker thread unwinds: its values are dropped, including the lock guard, so the mutex is unlocked.
2. `worker.join()` returns `Err` instead of the closure's result. The main thread did not crash; it learned that the worker failed.
3. Because the worker panicked *while holding the lock*, the mutex is now **poisoned**. `orders.lock()` returns `Err(PoisonError)`. Rust cannot know whether the worker finished its change or stopped in the middle, so it warns you.
4. `poisoned.into_inner()` gives you the data anyway. Here the change was complete, so using it is fine. In real code, decide carefully: if a half-done change is possible, reset the data or stop.

The process exited with code 0: one bug in one thread did not stop the program. That is the basic tool for **isolation**: put independent work in independent threads or tasks.

## 3. Containing panics in a service

A server handles many requests at the same time. A bug that a rare request triggers should fail *that* request, not the whole server. Tokio helps: a panic inside a spawned task stops only that task, and its `JoinHandle` returns a `JoinError` that says "this task panicked".

Put this in `src/lib.rs`:

```rust
//! Helpers for handling panics in a long-running program.

use std::any::Any;

/// Turns a panic payload into readable text.
/// `panic!("text")` carries a `&str`, `panic!("{x}")` and most std panics carry a `String`.
pub fn panic_message(payload: &(dyn Any + Send)) -> &str {
    if let Some(text) = payload.downcast_ref::<&str>() {
        text
    } else if let Some(text) = payload.downcast_ref::<String>() {
        text
    } else {
        "<non-text panic payload>"
    }
}

/// A request handler with a bug: it panics for an empty name.
pub fn greet(name: &str) -> String {
    let first = name.chars().next().unwrap(); // BUG: panics on ""
    format!("Hello, {name}! Your name starts with {first}.")
}
```

and this in `src/bin/tasks.rs`:

```rust
use p20_panics::{greet, panic_message};

#[tokio::main]
async fn main() {
    let requests = ["Ada", "", "Linus"];
    for name in requests {
        // Every request runs in its own task, like in a real server.
        let handle = tokio::spawn(async move { greet(name) });
        match handle.await {
            Ok(reply) => println!("200 OK: {reply}"),
            Err(err) if err.is_panic() => {
                let payload = err.into_panic();
                println!(
                    "500 Internal Server Error (bug: {})",
                    panic_message(payload.as_ref())
                );
            }
            Err(err) => println!("task was cancelled: {err}"),
        }
    }
    println!("the server is still running");
}
```

```bash
cargo run -q --bin tasks
```

```text
200 OK: Hello, Ada! Your name starts with A.

thread 'tokio-rt-worker' (5915415) panicked at src/lib.rs:19:37:
called `Option::unwrap()` on a `None` value
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
500 Internal Server Error (bug: called `Option::unwrap()` on a `None` value)
200 OK: Hello, Linus! Your name starts with L.
the server is still running
```

**What happens:**

1. The **panic payload** is a trait object (Lesson 06.1) of the trait `Any` from `std::any`: a value whose real type is not known at compile time, but which can be asked for it at run time. `downcast_ref::<T>()` asks "is it a `T`?" and returns `Some(&T)` if it is. Panic messages are almost always `&str` or `String`, so `panic_message` tries both.
2. The empty name makes `greet` panic inside its task. `handle.await` returns `Err(err)`, `err.is_panic()` is true, and `err.into_panic()` hands us the payload.
3. The "server" turns the bug into a 500 answer for that one request, and the next request is served normally. One user saw an error; the other two did not notice anything.

Web frameworks give you the same isolation. In axum (Lesson 16.3) each connection runs in its own task, so a panicking handler only breaks its own connection. The `CatchPanicLayer` middleware from `tower-http` goes one step further and turns a panic into a proper "500 Internal Server Error" response.

> ⚠️ **Common mistake:** Isolation is not a fix. The bug is still there and will hit the next user with an empty name. Isolation buys you time; the panic report (next section) tells you what to fix.

## 4. Making every panic visible: the panic hook

The default panic message goes to standard error as several lines of free text. In production your logs are collected by a machine (Lesson 16.12), and several unstructured lines are easy to miss. You also want a *number*: "how many panics happened?" is one of the best alarm signals a service has.

`std::panic::set_hook` replaces the function that runs on every panic. Create `src/bin/hook.rs`:

```rust
use std::panic;
use std::sync::atomic::{AtomicU64, Ordering};
use std::thread;

/// How many panics happened since start. A real service exports this as a metric.
static PANICS: AtomicU64 = AtomicU64::new(0);

fn install_panic_hook() {
    panic::set_hook(Box::new(|info| {
        let count = PANICS.fetch_add(1, Ordering::Relaxed) + 1;
        let thread = thread::current();
        let thread_name = thread.name().unwrap_or("<unnamed>");
        let location = info
            .location()
            .map(|l| format!("{}:{}", l.file(), l.line()))
            .unwrap_or_else(|| "unknown".to_string());
        let message = info.payload_as_str().unwrap_or("<non-text payload>");
        eprintln!(
            "level=ERROR event=panic thread={thread_name} location={location} \
             message={message:?} panics_total={count}"
        );
    }));
}

fn main() {
    install_panic_hook();

    let worker = thread::Builder::new()
        .name("report-worker".to_string())
        .spawn(|| {
            let sizes: Vec<u64> = Vec::new();
            let average = sizes.iter().sum::<u64>() / sizes.len() as u64; // BUG: divides by 0
            println!("average: {average}");
        })
        .unwrap();

    let _ = worker.join();
    println!(
        "main continues, panics so far: {}",
        PANICS.load(Ordering::Relaxed)
    );
}
```

```bash
cargo run -q --bin hook
```

```text
level=ERROR event=panic thread=report-worker location=src/bin/hook.rs:32 message="attempt to divide by zero" panics_total=1
main continues, panics so far: 1
```

**What happens:**

1. `set_hook` takes a boxed closure. Rust calls it on every panic in every thread, *before* unwinding starts, with a `PanicHookInfo` value named `info`.
2. `info.location()` gives the file and line of the panic; `info.payload_as_str()` gives the message when it is text.
3. The hook increments an atomic counter (Lesson 14.5) and writes one line of `key=value` pairs. A log system can search for `event=panic` and alert on `panics_total`.
4. `thread::Builder::new().name(...)` gives the thread a name, which makes the log line much more useful than `<unnamed>`.
5. The default hook is replaced, so the usual multi-line message is gone. Exercise 3 shows how to keep it as well.

In a real service you would call `tracing::error!` (Lesson 09.10) in the hook instead of `eprintln!`, so that panics flow into the same logs as everything else, and you would export the counter as a metric (Lesson 16.12).

> 💡 **Tip:** Set `RUST_BACKTRACE=1` in the environment of your service. The default hook then prints a backtrace: the list of functions that were running when the panic happened. In your own hook, `std::backtrace::Backtrace::capture()` gives the same list and respects the same variable.

> ⚠️ **Common mistake:** Keep the hook simple. It runs in the middle of a failure: no locks that the panicking code might hold, no network calls, and certainly no `unwrap`. A panic inside the panic hook aborts the process.

## 5. Unwind or abort?

You can choose what a panic does for the whole program with one setting in `Cargo.toml`:

```toml
[profile.release]
panic = "abort"
```

With `"abort"` (the default is `"unwind"`), the first panic in *any* thread ends the whole process at once. Run the task example in release mode with this setting and print the exit code:

```bash
cargo run -q --release --bin tasks
echo "exit code: $?"
```

```text
200 OK: Hello, Ada! Your name starts with A.

thread 'tokio-rt-worker' (5923630) panicked at src/lib.rs:19:37:
called `Option::unwrap()` on a `None` value
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
exit code: 134
```

The hook still runs (the message is printed), but then the process stops. "Linus" was never served. Exit code 134 means "killed by the signal SIGABRT" (signal abort, Lesson 13.7). Neither `JoinError` nor `catch_unwind` can catch an abort, and no `Drop` runs, so buffered log lines or half-written files are not flushed.

Both choices are used in production:

| | `panic = "unwind"` (default) | `panic = "abort"` |
|---|---|---|
| One panic stops | one thread or task | the whole process |
| `Drop` and cleanup | run during unwinding | do not run |
| Good for | servers with many independent requests | programs where a bug means the state in memory cannot be trusted |
| Recovery | the service keeps running; fix the bug later | a **supervisor** restarts the process; it rebuilds its state from durable storage |
| Binary size | a little larger | a little smaller |

The `abort` style is called *crash-only* design: never try to repair the state in memory, just crash and start fresh. It is a good fit for a storage engine that might be in the middle of a write: restarting and replaying the write-ahead log (Lesson 17.4) is safer than continuing. It only works if something restarts the process: `Restart=on-failure` in a systemd unit (Lesson 11.6), or Kubernetes (Phase 21).

> 🧪 **Testing:** The test harness always uses unwinding, whatever you write in `[profile.release]`, so `#[should_panic]` tests and `catch_unwind` keep working in `cargo test`.

Remove the `[profile.release]` lines again before you continue.

## 6. Keeping panics out of production code

The cheapest panic is the one that never reaches production. Clippy (Lesson 05.6) has lints that point at every place where code *may* panic. They are off by default, because `unwrap` is fine in examples and tests. Turn them on for this package in `Cargo.toml`:

```toml
[lints.clippy]
unwrap_used = "warn"
expect_used = "warn"
panic = "warn"
indexing_slicing = "warn"
```

and allow them inside tests with a file `clippy.toml` next to `Cargo.toml`:

```toml
allow-unwrap-in-tests = true
allow-expect-in-tests = true
allow-panic-in-tests = true
allow-indexing-slicing-in-tests = true
```

```bash
cargo clippy -q --lib
```

```text
warning: used `unwrap()` on an `Option` value
  --> src/lib.rs:19:17
   |
19 |     let first = name.chars().next().unwrap(); // BUG: panics on ""
   |                 ^^^^^^^^^^^^^^^^^^^^^^^^^^^^
   |
   = note: if this value is `None`, it will panic
   = help: for further information visit https://rust-lang.github.io/rust-clippy/rust-1.100.0/index.html#unwrap_used
   = note: requested on the command line with `-W clippy::unwrap-used`
```

**What happens:**

1. `[lints.clippy]` turns on four lints for the package; Cargo passes them to Clippy as command-line flags, which is why the note says "requested on the command line".
2. `unwrap_used` found exactly the bug that our task example hit. `indexing_slicing` would also flag `list[4]` in `thread_panic.rs` (run `cargo clippy --bins` to see it).
3. `clippy.toml` keeps tests free to use `unwrap()`: in a test, a panic *is* the right way to fail.

The fix is to return the expected error instead of panicking. Exercise 2 asks you to write it, with a test.

> 📏 **Convention:** When a panic is truly impossible, write `expect("reason")` with the reason, and allow the lint at that one place with `#[allow(clippy::expect_used)]` and a comment. Then every remaining `expect` in the code base is a documented decision, and a reviewer can check the reason.

> 🔒 **Security:** In code that handles untrusted input (parsers, request handlers), treat every warning of these lints as a possible denial of service. Lesson 10.12 showed how one crafted input can crash a server that panics.

## Common mistakes

**1. Using `catch_unwind` as error handling.** `catch_unwind` is a safety net at a boundary (one job, one request, one plugin), not a way to "try" risky code. Expected problems must be `Result`. Also, some panics cannot be caught at all: with `panic = "abort"`, or when a panic happens inside the panic hook.

**2. Ignoring a `JoinError`.** `let _ = handle.await;` throws away the information that a task panicked. Always look at the result of a task you care about, at least to log it.

**3. Spreading poison.** If every `lock()` is followed by `.unwrap()`, one panic while holding a lock makes every later `lock()` panic too (Exercise 1 shows it). Decide in one place what a poisoned lock means for your data.

**4. Choosing `abort` without a supervisor.** With `panic = "abort"` and nothing that restarts the process, the first bug in production stops the service until a human notices.

## Try it yourself

1. Create `code/p20-panics`, add `src/bin/thread_panic.rs` and run it. Change `list[4]` to `list[3]` and run again: the lock is no longer poisoned.
2. Add `src/lib.rs` and `src/bin/tasks.rs`. Run them and check that "Linus" is served after the panic.
3. Add `src/bin/hook.rs`. Run it once normally and once with `RUST_BACKTRACE=1 cargo run -q --bin hook`. Does your hook print a backtrace? (It does not: only the default hook reads the variable by itself.)
4. Add `panic = "abort"` to `[profile.release]`, run `cargo run -q --release --bin tasks; echo $?`, then remove the setting again.
5. Turn on the four Clippy lints and run `cargo clippy --lib --bins`. Count the warnings.

## Exercises

### Exercise 1 — Poison spreads (easy)

In `thread_panic.rs`, replace the whole `match orders.lock() { … }` with these two lines, run the program and explain the output:

```rust
    let list = orders.lock().unwrap();
    println!("orders: {list:?}");
```

<details>
<summary>Hint</summary>

What does `lock()` return for a poisoned mutex, and what does `unwrap()` do with that?

</details>

<details>
<summary>Answer</summary>

```text
thread '<unnamed>' (5934422) panicked at src/bin/ex1.rs:11:25:
index out of bounds: the len is 4 but the index is 4
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
worker finished with an error: true

thread 'main' (5934392) panicked at src/bin/ex1.rs:18:30:
called `Result::unwrap()` on an `Err` value: PoisonError { .. }
```

(The file in the output is `src/bin/ex1.rs` because the code was saved under that name; yours shows `thread_panic.rs`.) The worker's panic poisoned the mutex. `lock()` returns `Err(PoisonError)`, and `unwrap()` turns it into a second panic, now in the main thread, which ends the program with exit code 101. One bug became two failures.

</details>

### Exercise 2 — Fix the handler, with tests (medium)

Write `pub fn greet_checked(name: &str) -> Result<String, String>` in `src/lib.rs`. It returns `Err("name must not be empty")` for an empty name and the same greeting as `greet` otherwise. Write two tests, one for each case, and check that `cargo clippy --lib` no longer warns about this function.

<details>
<summary>Hint</summary>

`let Some(first) = name.chars().next() else { return Err(...); };` handles the empty case without `unwrap`.

</details>

<details>
<summary>Answer</summary>

```rust
/// The fixed handler: an empty name is an expected error, not a bug.
pub fn greet_checked(name: &str) -> Result<String, String> {
    let Some(first) = name.chars().next() else {
        return Err("name must not be empty".to_string());
    };
    Ok(format!("Hello, {name}! Your name starts with {first}."))
}

#[cfg(test)]
mod greet_tests {
    use super::*;

    #[test]
    fn empty_name_is_an_error_not_a_panic() {
        assert_eq!(greet_checked(""), Err("name must not be empty".to_string()));
    }

    #[test]
    fn normal_name_is_greeted() {
        assert_eq!(
            greet_checked("Ada"),
            Ok("Hello, Ada! Your name starts with A.".to_string())
        );
    }
}
```

Why it works: the empty name is now an expected error that the caller can turn into "400 Bad Request". There is no path left in this function that can panic, so Clippy has nothing to report for it.

</details>

### Exercise 3 — Keep the default message too (medium)

Change the hook so that it first writes the one-line `level=ERROR event=panic message=…` record and then also runs the default hook, so the usual message (and, with `RUST_BACKTRACE=1`, the backtrace) still appears. Test it with a thread that reads index 0 of an empty `Vec<u32>`.

<details>
<summary>Hint</summary>

`panic::take_hook()` returns the current hook (the default one) as a boxed closure. Move it into your new closure and call it at the end.

</details>

<details>
<summary>Answer</summary>

```rust
use std::panic;
use std::thread;

fn main() {
    let default_hook = panic::take_hook();
    panic::set_hook(Box::new(move |info| {
        let message = info.payload_as_str().unwrap_or("<non-text payload>");
        eprintln!("level=ERROR event=panic message={message:?}");
        default_hook(info);
    }));

    let worker = thread::spawn(|| {
        let empty: Vec<u32> = Vec::new();
        empty[0]
    });
    println!("worker failed: {}", worker.join().is_err());
}
```

```text
level=ERROR event=panic message="index out of bounds: the len is 0 but the index is 0"

thread '<unnamed>' (5934631) panicked at src/bin/ex3.rs:14:14:
index out of bounds: the len is 0 but the index is 0
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
worker failed: true
```

Why it works: `take_hook` removes the default hook and gives it to you; `move` puts it inside the new closure, which calls it after writing its own line. With `RUST_BACKTRACE=1` the default hook adds the `stack backtrace:` list.

</details>

### Exercise 4 — A job runner that survives panics (hard)

Write `pub fn run_isolated<T>(job: impl FnOnce() -> T + std::panic::UnwindSafe) -> Result<T, String>`: it runs `job`, returns `Ok(value)` normally and `Err(message)` if the job panics. Use `catch_unwind` and `panic_message`. Write three tests: a good job returns its value; a job that panics with `"disk is full"` gives `Err("disk is full")`; in a list of three jobs where the second panics, the first and third still give their values.

<details>
<summary>Hint</summary>

`std::panic::catch_unwind(job)` returns `Result<T, Box<dyn Any + Send>>`. Use `map_err` to turn the box into a `String`. `UnwindSafe` is a marker trait that says "it is OK to look at this closure's data after a panic"; simple closures have it automatically.

</details>

<details>
<summary>Answer</summary>

```rust
/// Runs one job and turns a panic into an `Err` with the panic message.
/// This works only with `panic = "unwind"` (the default).
pub fn run_isolated<T>(job: impl FnOnce() -> T + std::panic::UnwindSafe) -> Result<T, String> {
    std::panic::catch_unwind(job).map_err(|payload| panic_message(payload.as_ref()).to_string())
}

#[cfg(test)]
mod runner_tests {
    use super::*;

    #[test]
    fn a_good_job_returns_its_value() {
        assert_eq!(run_isolated(|| 2 + 2), Ok(4));
    }

    #[test]
    fn a_panicking_job_becomes_an_error() {
        let result: Result<u32, String> = run_isolated(|| panic!("disk is full"));
        assert_eq!(result, Err("disk is full".to_string()));
    }

    #[test]
    fn later_jobs_still_run_after_a_panic() {
        let results: Vec<Result<u32, String>> = vec![
            run_isolated(|| 1),
            run_isolated(|| panic!("job 2 failed")),
            run_isolated(|| 3),
        ];
        assert_eq!(results, vec![Ok(1), Err("job 2 failed".to_string()), Ok(3)]);
    }
}
```

```bash
cargo test -q --lib runner_tests
```

```text
running 3 tests
...
test result: ok. 3 passed; 0 failed; 0 ignored; 0 measured; 3 filtered out; finished in 0.00s
```

(The number of filtered-out tests depends on how many other tests you have.) Why it works: `catch_unwind` stops the unwinding at the edge of the closure, so the panic cannot leave `run_isolated`. This is the thread-pool stretch goal of Project 06.11: wrap every job like this, and a buggy job no longer kills a worker thread. (The panic messages are still printed by the hook; the test harness hides them because the tests pass.)

</details>

## Quiz

1. Which of these should be a panic: a missing configuration file, a user name that is too long, or reading past the end of a buffer that your own code sized?

<details>
<summary>Answer</summary>

Only the last one: it is a bug in your code. A missing file and a too-long name are expected errors and should be returned as `Err`.

</details>

2. A thread panics while holding a `Mutex`. What happens to the mutex, and why?

<details>
<summary>Answer</summary>

It becomes poisoned: later `lock()` calls return `Err(PoisonError)`. Rust cannot know whether the change to the data was finished, so it warns everyone who uses the data next. `into_inner()` still gives access to it.

</details>

3. In a Tokio server, a handler panics. Does the server stop?

<details>
<summary>Answer</summary>

No (with the default `panic = "unwind"`). Only that task stops; its `JoinHandle` returns a `JoinError` with `is_panic() == true`. Other tasks keep running.

</details>

4. Name two things a good panic hook does in a service.

<details>
<summary>Answer</summary>

It writes a structured log line with the location and the message (so the log system finds it), and it increments a panic counter that is exported as a metric and alerted on. It can also call the previous hook.

</details>

5. Why can `panic = "abort"` be the *more* reliable choice for a storage engine?

<details>
<summary>Answer</summary>

After a bug in the middle of a write, the state in memory may be inconsistent. Aborting and restarting from durable data (for example by replaying the write-ahead log) is safer than continuing with doubtful state. It requires a supervisor that restarts the process.

</details>

6. What does `allow-unwrap-in-tests = true` in `clippy.toml` do, and why is it reasonable?

<details>
<summary>Answer</summary>

It stops the `unwrap_used` lint from warning inside tests. In a test a panic is the normal way to fail, so `unwrap()` there is fine; the lint is meant for production code.

</details>

## Summary

- Expected problems are `Err`; a panic means a bug. Bugs will reach production, so the service must survive, report and isolate them.
- A panic unwinds only its thread or task. `join()` returns `Err`, a Tokio `JoinError` says `is_panic()`, and a mutex held during the panic is **poisoned**.
- Run independent work in independent tasks: one bad request then fails alone. `catch_unwind` is the same safety net for synchronous jobs.
- A **panic hook** (`std::panic::set_hook`) turns every panic into one structured log line and a counter; `take_hook` lets you keep the default message.
- `panic = "unwind"` keeps the process alive; `panic = "abort"` stops it at once (exit code 134) and needs a supervisor. Choose per program.
- The Clippy lints `unwrap_used`, `expect_used`, `panic` and `indexing_slicing`, allowed in tests via `clippy.toml`, find possible panics before production does.

## Sources and further reading

- [The Rust Programming Language — 9.3 To panic! or Not to panic!](https://doc.rust-lang.org/book/ch09-03-to-panic-or-not-to-panic.html) — the official guidance on when to panic and when to return `Result`.
- [std::panic::set_hook](https://doc.rust-lang.org/std/panic/fn.set_hook.html) and [std::panic::catch_unwind](https://doc.rust-lang.org/std/panic/fn.catch_unwind.html) — the hook API (Application Programming Interface) and the limits of catching panics.
- [std::sync::Mutex — poisoning](https://doc.rust-lang.org/std/sync/struct.Mutex.html) — what poisoning means and how to recover the data.
- [tokio::task::JoinError](https://docs.rs/tokio/1.54.1/tokio/task/struct.JoinError.html) — `is_panic`, `into_panic` and cancelled tasks.
- [The Cargo Book — Profiles: panic](https://doc.rust-lang.org/cargo/reference/profiles.html) — the `panic` setting and why tests always unwind.
- [Clippy lints](https://rust-lang.github.io/rust-clippy/master/index.html) — search for `unwrap_used`, `indexing_slicing` and the `allow-…-in-tests` options.

---

[🏠 Start](../START-HERE.md) · [← 20.1 What reliability means](01-what-reliability-means.md) · [📋 Phase 20](README.md) · [20.3 Fuzzing in depth →](03-fuzzing-in-depth.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
