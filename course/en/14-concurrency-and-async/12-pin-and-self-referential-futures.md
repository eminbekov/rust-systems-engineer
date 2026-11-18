<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 14.11 Build a tiny async runtime](11-build-a-tiny-async-runtime.md) · [📋 Phase 14](README.md) · [14.13 Tokio in depth: tasks and channels →](13-tokio-tasks-and-channels.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/14-concurrency-and-async/12-pin-and-self-referential-futures.md) <!-- nav -->

# Lesson 14.12 — Pin and self-referential futures

> **Phase 14 — Concurrency and async in depth** · Lesson 12 of 16 · ⏱ about 120 minutes

## What you will learn

- Why a value that points into itself breaks when it is moved, and why many futures are such values.
- What `Pin` promises, and what the auto trait `Unpin` means.
- The two ways to pin a future, `Box::pin` and `pin!`, and when to use which.
- How to read the compiler error "… cannot be unpinned" and fix it.
- How to write a future that wraps another future, with *pin projection* from the `pin-project-lite` crate.

## Before you start

- You finished [Lesson 14.11 — Build a tiny async runtime](11-build-a-tiny-async-runtime.md). There you used `Pin<Box<dyn Future>>` and `Pin::new` without a full explanation; this lesson gives it.
- You know raw pointers and the `{:p}` format from [Lesson 06.6 — Unsafe Rust and Miri](../06-rust-advanced/06-unsafe-rust-and-miri.md), and *moves* from [Phase 03](../03-ownership-and-borrowing/README.md).
- You know `tokio::select!` and `tokio::time::sleep` from [Lesson 06.9](../06-rust-advanced/09-async-await-introduction.md).
- Your code for this lesson goes in `~/projects/rust/code/p14-pin`. Add the dependencies:

```bash
cd ~/projects/rust
cargo new --lib code/p14-pin
cd code/p14-pin
cargo add futures@0.3 pin-project-lite@0.2
cargo add tokio@1 --features rt-multi-thread,macros,time
mkdir src/bin
```

## New words

| Word | Meaning |
|---|---|
| **self-referential** | A value that contains a pointer or reference to another part of itself. |
| **move** (in memory) | Copying the bytes of a value to a new address, after which the old place is not used. Every Rust move can do this. |
| **dangling pointer** | A pointer to a place where the value no longer is. Reading through it is undefined behaviour. |
| **`Pin<P>`** | A wrapper around a pointer `P` (such as `&mut T` or `Box<T>`) that promises: the value it points to will not be moved again until it is dropped. |
| **pinning** | Putting a value behind `Pin`, so that it stays at one address. |
| **`Unpin`** | An auto trait meaning "moving this value is always safe, even when it is pinned". Almost all types implement it; compiler-made futures do not. |
| **auto trait** | A trait the compiler implements for a type automatically when all its fields have it, like `Send` and `Sync`. |
| **`PhantomPinned`** | A marker type with no data. A struct that contains it is not `Unpin`. |
| **pin projection** | Getting a pinned reference to one field (`Pin<&mut Field>`) from a pinned reference to the whole struct (`Pin<&mut Struct>`). |

## 1. The problem: a value that points into itself

A move in Rust copies the bytes of a value to a new place. Usually that is completely fine. It stops being fine when a value stores the address of one of its own fields. Create `src/bin/moving.rs`:

```rust
/// A struct that keeps a pointer to one of its own fields.
struct SelfRef {
    name: String,
    name_ptr: *const String,
}

impl SelfRef {
    fn new(name: &str) -> Self {
        SelfRef {
            name: name.to_string(),
            name_ptr: std::ptr::null(),
        }
    }

    fn remember_address(&mut self) {
        self.name_ptr = &self.name;
    }

    fn report(&self, label: &str) {
        let real: *const String = &self.name;
        println!(
            "{label}: field at {real:p}, pointer says {:p}, same: {}",
            self.name_ptr,
            real == self.name_ptr
        );
    }
}

fn main() {
    let mut first = SelfRef::new("ferris");
    first.remember_address();
    first.report("before move");

    let moved = first; // a move copies the bytes to a new place
    moved.report("after move ");
}
```

`cargo run --bin moving` (your addresses will differ):

```text
before move: field at 0x16d751d90, pointer says 0x16d751d90, same: true
after move : field at 0x16d751db0, pointer says 0x16d751d90, same: false
```

**What happens:**

1. `remember_address` stores the address of `self.name` inside the struct itself. Creating a raw pointer is safe; only reading through it needs `unsafe`.
2. `let moved = first;` moves the struct. Its bytes now live at a new address (here 32 bytes further on the stack).
3. The field moved, but the stored pointer still holds the **old** address. It is now a dangling pointer. We only compare addresses; reading through `name_ptr` here would be undefined behaviour.

The borrow checker does not let you write this with normal references (`&String` pointing into the same struct), exactly because of this danger. That is why the example needs raw pointers.

## 2. Why futures are self-referential

An `async fn` can hold a reference to its own local variable across an `.await`. Create `src/bin/future_size.rs`:

```rust
async fn tick() {}

async fn sum_after_tick() -> u64 {
    let numbers = [1_u64, 2, 3, 4];
    let first = &numbers[0]; // a reference into a local of the future
    tick().await; // `numbers` and `first` must survive this point
    *first + numbers[3]
}

fn main() {
    let future = sum_after_tick();
    println!("the future is {} bytes", std::mem::size_of_val(&future));
    println!("result = {}", futures::executor::block_on(future));
}
```

```text
the future is 48 bytes
result = 5
```

**What happens:**

1. From Lesson 14.10 you know that the compiler turns `sum_after_tick` into a state machine. Everything that lives across `tick().await` is stored **inside the future**: the array `numbers` (32 bytes), the reference `first` (8 bytes) and the state number.
2. `first` points to `numbers[0]`, and both are inside the same future. The future is self-referential, just like `SelfRef`.
3. Before the first poll the reference does not exist yet, so moving the future is fine (we move it into `block_on` here). After the first poll, `first` holds an address inside the future. If the future moved then, `first` would dangle.

So the rule for futures is: **once a future has been polled, it must not move**. `Pin` is how Rust makes this rule part of the type system: `poll` takes `self: Pin<&mut Self>`, so you cannot call it without first promising not to move the future.

## 3. `Pin` and `Unpin`

`Pin<P>` wraps a pointer: `Pin<&mut T>`, `Pin<Box<T>>`. It adds no data and no run-time cost. It only removes one ability: with a `Pin<&mut T>` you cannot get a plain `&mut T` (in safe code), and with a plain `&mut T` you could move the value out with `std::mem::swap` or `std::mem::replace`. No `&mut T`, no moving.

But most types do not care about moving at all. An `i32`, a `String` or a `Vec` holds no pointer into itself. For them `Pin` would be a useless burden, so Rust has the auto trait **`Unpin`**: "moving this value is safe even when it is pinned". For an `Unpin` type, `Pin::new(&mut value)` and `pin.get_mut()` are allowed in safe code, and `Pin` is just a formality. You used this in Lesson 14.10 (`Pin::new(inner)` on `YieldOnce`).

Which types are **not** `Unpin`?

| Type | `Unpin`? | Why |
|---|---|---|
| `i32`, `String`, `Vec<T>`, `Box<T>` | yes | no pointers into themselves |
| `Pin<Box<T>>` for any `T` | yes | moving the box moves only the pointer; the value stays on the heap |
| the future of an `async fn` or `async` block | no | it may hold references into itself after the first poll |
| `tokio::time::Sleep` | no | it contains `PhantomPinned`, because Tokio's timer keeps a pointer to it |
| your own struct | yes, if all fields are `Unpin` | auto trait, like `Send` |

## 4. How to pin a future

There are two ways. Both give you something that you can poll again and again.

**`Box::pin(future)`** puts the future on the heap and returns `Pin<Box<F>>`. Moving the box later only moves the pointer, so the future itself never moves. Use it when the future must be stored somewhere: in a struct, in a `Vec`, in a queue of tasks (exactly what `BoxFuture` in Lesson 14.11 was), or returned from a function. It costs one heap allocation.

**`std::pin::pin!(future)`** pins the future in the current function (on the stack) and gives `Pin<&mut F>`. No allocation. The future cannot leave the function, which is fine for the most common case: a future you poll several times inside a loop.

That common case is `select!` in a loop. You want one deadline for the whole loop, not a new deadline in every round, so you create the `sleep` future once and give `select!` a `&mut` to it. (`tokio::time::interval(d)` is a timer that "ticks" every `d`; each `ticks.tick().await` waits for the next tick.)

```rust
use std::time::Duration;
use tokio::time::{interval, sleep};

#[tokio::main]
async fn main() {
    let deadline = sleep(Duration::from_millis(350));
    let mut ticks = interval(Duration::from_millis(100));
    loop {
        tokio::select! {
            _ = &mut deadline => break,
            _ = ticks.tick() => println!("tick"),
        }
    }
}
```

This does not compile:

```text
error[E0277]: `PhantomPinned` cannot be unpinned
  --> src/bin/select_loop.rs:9:9
   |
 9 | /         tokio::select! {
10 | |             _ = &mut deadline => break,
11 | |             _ = ticks.tick() => println!("tick"),
12 | |         }
   | |_________within `(PhantomData<&()>, PhantomPinned)`, the trait `Unpin` is not implemented for `PhantomPinned`
   |
   = note: consider using the `pin!` macro
           consider using `Box::pin` if you need to access the pinned value outside of the current scope
   = note: required because it appears within the type `(PhantomData<&()>, PhantomPinned)`
   = note: required for `Sleep` to implement `Unpin`
   = note: required for `&mut Sleep` to implement `Future`
```

How to read it, from the bottom up: `select!` needs `&mut Sleep` to be a future; `&mut F` is a future only if `F` is `Unpin` (otherwise polling through a plain `&mut` could move it); `Sleep` is not `Unpin` because it contains `PhantomPinned`. The compiler even suggests the fix. Here is the working version, `src/bin/select_loop.rs`:

```rust
use std::pin::pin;
use std::time::Duration;
use tokio::time::{interval, sleep};

#[tokio::main]
async fn main() {
    let mut deadline = pin!(sleep(Duration::from_millis(350)));
    let mut ticks = interval(Duration::from_millis(100));
    loop {
        tokio::select! {
            _ = &mut deadline => break,
            _ = ticks.tick() => println!("tick"),
        }
    }
    println!("deadline reached");
}
```

```text
tick
tick
tick
tick
deadline reached
```

**What happens:**

1. `pin!(sleep(…))` pins the `Sleep` future in `main` and gives `Pin<&mut Sleep>`. That pinned pointer is itself a future, and it *is* `Unpin` (it is only a pointer; the `Sleep` behind it stays where it is), so `&mut deadline` is a future too, and `select!` accepts it.
2. `interval` ticks at 0, 100, 200 and 300 ms (the first tick is immediate), so four ticks are printed.
3. At 350 ms the same `deadline` future, which was polled in every round, finishes, and the loop breaks. If you wrote `sleep(Duration::from_millis(350))` directly inside `select!`, a fresh sleep would be created in every round, and with a tick every 100 ms it would never finish.

> 💡 **Tip:** The same error appears with a plain `async fn` future, for example `Pin::new(&mut hello())`: "`{async fn body of hello()}` cannot be unpinned", with the same note "consider using the `pin!` macro". The answer is always one of the two: `pin!` if the future stays in this function, `Box::pin` if it must go somewhere else.

## 5. Writing a wrapper future: pin projection

Sometimes you write a future that contains another future, like `timeout` contains the future it limits. The inner future may be self-referential, so it must stay pinned too. Inside `poll` you have `self: Pin<&mut Self>`, and you need `Pin<&mut Inner>` for the field. Getting that is called **pin projection**.

Doing it by hand needs `unsafe` (`Pin::get_unchecked_mut` and `Pin::new_unchecked`) and a careful promise that you never move the field. The small crate `pin-project-lite` does it for you, safely, with a macro. Here is a wrapper `Timed` that measures how long its inner future took and how many polls it needed. Put it into `src/lib.rs`:

```rust
//! A future wrapper that needs pin projection.

use pin_project_lite::pin_project;
use std::future::Future;
use std::pin::Pin;
use std::task::{Context, Poll};
use std::time::{Duration, Instant};

pin_project! {
    /// Wraps a future and measures how long it ran and how often it was polled.
    pub struct Timed<F> {
        #[pin]
        inner: F,
        polls: u32,
        started: Option<Instant>,
    }
}

/// What `Timed` returns: the inner output plus the measurements.
#[derive(Debug)]
pub struct Report<T> {
    pub output: T,
    pub polls: u32,
    pub elapsed: Duration,
}

pub fn timed<F: Future>(inner: F) -> Timed<F> {
    Timed {
        inner,
        polls: 0,
        started: None,
    }
}

impl<F: Future> Future for Timed<F> {
    type Output = Report<F::Output>;

    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output> {
        let this = self.project();
        *this.polls += 1;
        let started = *this.started.get_or_insert_with(Instant::now);
        match this.inner.poll(cx) {
            Poll::Pending => Poll::Pending,
            Poll::Ready(output) => Poll::Ready(Report {
                output,
                polls: *this.polls,
                elapsed: started.elapsed(),
            }),
        }
    }
}
```

**What happens:**

1. `pin_project! { … }` wraps a normal struct definition. `#[pin]` marks the fields that must stay pinned: here only `inner`.
2. The macro generates a method `project(self: Pin<&mut Self>)`. It returns a helper struct with one field per original field: `inner` becomes `Pin<&mut F>` (because of `#[pin]`), and `polls` and `started` become plain `&mut u32` and `&mut Option<Instant>`. Pinning a counter would be pointless; we just change it.
3. `this.inner.poll(cx)` polls the inner future through its pinned reference, so it works for any `F: Future`, also for futures that are not `Unpin`. No `Unpin` bound is needed, unlike `Join2` in Lesson 14.10.
4. `get_or_insert_with(Instant::now)` sets the start time at the first poll and keeps it afterwards.
5. The macro also makes sure that `Timed<F>` is `Unpin` only when `F` is, which is the correct rule.

Use it from `src/bin/timed.rs`:

```rust
use p14_pin::timed;
use std::time::Duration;

async fn slow_answer() -> u32 {
    tokio::time::sleep(Duration::from_millis(100)).await;
    tokio::time::sleep(Duration::from_millis(50)).await;
    42
}

#[tokio::main]
async fn main() {
    let report = timed(slow_answer()).await;
    println!(
        "output = {}, polls = {}, elapsed = {} ms",
        report.output,
        report.polls,
        report.elapsed.as_millis()
    );
}
```

```text
output = 42, polls = 3, elapsed = 159 ms
```

Three polls: one at the start, one after each sleep. The time is a little more than 150 ms, because a timer always fires at or after its deadline, never before.

> 🔒 **Security:** `pin-project-lite` exists because hand-written projection is easy to get wrong in `unsafe` code: one `mem::swap` on a field that should be pinned, or a `Drop` implementation that moves it, is undefined behaviour that the compiler cannot catch. Prefer the macro to your own `unsafe` blocks.

## Common mistakes

**1. `&mut future` in `select!` without pinning.** You get "`…` cannot be unpinned" (Section 4). Fix: `let mut future = pin!(future);` before the loop.

**2. Creating the future inside the loop.** `tokio::select! { _ = sleep(d) => … }` inside a `loop` starts a new `sleep` in every round, so a "deadline" never arrives while other branches keep firing. Create the future once, pin it, and pass `&mut` to it.

**3. `Box::pin` everywhere "to make the error go away".** It works, but it allocates on every call. Inside one function `pin!` is enough. Use `Box::pin` when you store, return, or keep futures of different types in one collection (`Pin<Box<dyn Future>>`).

**4. Adding `Unpin` bounds to your own generic code without need.** `fn run<F: Future + Unpin>(f: F)` rejects every `async fn` future. Pin the future inside instead (`let f = pin!(f);`), or project with `pin-project-lite` in a wrapper.

**5. Thinking `Pin` moves or fixes anything at run time.** `Pin` is only a promise checked by the compiler. It does not copy, lock or allocate anything. `Box::pin` allocates because of the `Box`, not because of the `Pin`.

## Try it yourself

1. Create `code/p14-pin` as in "Before you start", run `moving` and `future_size`.
2. In `future_size.rs`, change the array to `[1_u64; 100]` and run again. Predict the new size of the future before you look. (816 bytes: it grows by the size of the array, because the array lives inside the future.)
3. Write the `select!` loop without `pin!`, read the whole compiler error, then fix it.
4. Add `Timed` to `src/lib.rs` and run `timed`. Change `slow_answer` to sleep three times and check that `polls` becomes 4.
5. Run `cargo clippy --all-targets` and `cargo fmt`, then commit: `git add code/p14-pin && git commit -m "p14-pin: pinning and a Timed wrapper"`.

## Exercises

### Exercise 1 — Who is `Unpin`? (easy)

Write `fn assert_unpin<T: Unpin>(_value: &T) {}` and call it with `&5_i32`, `&String::from("text")`, a `Pin<Box<dyn Future<Output = ()>>>`, the future of an `async fn hello() {}`, and a `tokio::time::sleep(…)` future. Predict which calls compile, then check with `cargo build`.

<details>
<summary>Hint</summary>

Look at the table in Section 3. Comment out one call at a time to see each error alone.

</details>

<details>
<summary>Answer</summary>

The first three compile. The last two fail:

```text
error[E0277]: `{async fn body of hello()}` cannot be unpinned
error[E0277]: `PhantomPinned` cannot be unpinned
```

Why it works: `i32` and `String` contain no pointers into themselves. `Pin<Box<…>>` is only a pointer to the heap, so moving it moves nothing important. The `async fn` future may become self-referential after its first poll, and `Sleep` contains `PhantomPinned` on purpose.

</details>

### Exercise 2 — Test `Timed` (medium)

Add tests to `src/lib.rs`: `timed(async { 42 })` returns output 42 after exactly **one** poll; an `async` block that calls `tokio::task::yield_now().await` once and then returns `"done"` needs exactly **two** polls. Use `futures::executor::block_on` for the first test and `#[tokio::test]` for the second.

<details>
<summary>Hint</summary>

`yield_now` from Tokio works like your `yield_now` from Lesson 14.11: it returns `Pending` once and wakes itself. It needs a Tokio runtime, which `#[tokio::test]` creates. Check `report.polls` with `assert_eq!`.

</details>

<details>
<summary>Answer</summary>

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn a_ready_future_is_polled_once() {
        let report = futures::executor::block_on(timed(async { 42 }));
        assert_eq!(report.output, 42);
        assert_eq!(report.polls, 1);
    }

    #[tokio::test]
    async fn a_future_that_yields_once_is_polled_twice() {
        let report = timed(async {
            tokio::task::yield_now().await;
            "done"
        })
        .await;
        assert_eq!(report.output, "done");
        assert_eq!(report.polls, 2);
    }
}
```

Why it works: an `async` block with no `.await` inside is `Ready` on its first poll. `yield_now` adds exactly one `Pending`, so `Timed` counts two polls. These tests do not depend on time, so they are never flaky.

</details>

### Exercise 3 — Your own `timeout` (hard)

Write `WithTimeout<F>` with two pinned fields: the inner future and a `tokio::time::Sleep`. Its output is `Result<F::Output, TimedOut>`, where `TimedOut` is your own unit struct. Add `with_timeout(limit, inner)`, and two `#[tokio::test]`s: `async { 7 }` with a 200 ms limit gives `Ok(7)`; a 500 ms `sleep` with a 20 ms limit gives `Err(TimedOut)`.

<details>
<summary>Hint</summary>

Both fields get `#[pin]`: `Sleep` is not `Unpin`, and `F` may not be. In `poll`, project once, poll `inner` first (if it is ready, return `Ok`), then poll `sleep` (if it is ready, return `Err(TimedOut)`), otherwise `Pending`. Both inner futures received the same waker, so whichever is ready first wakes the task. Derive `Debug` and `PartialEq` on `TimedOut` for `assert_eq!`.

</details>

<details>
<summary>Answer</summary>

```rust
pin_project! {
    /// Like `tokio::time::timeout`: gives up when `sleep` finishes first.
    pub struct WithTimeout<F> {
        #[pin]
        inner: F,
        #[pin]
        sleep: tokio::time::Sleep,
    }
}

/// The error of `WithTimeout`.
#[derive(Debug, PartialEq)]
pub struct TimedOut;

pub fn with_timeout<F: Future>(limit: Duration, inner: F) -> WithTimeout<F> {
    WithTimeout {
        inner,
        sleep: tokio::time::sleep(limit),
    }
}

impl<F: Future> Future for WithTimeout<F> {
    type Output = Result<F::Output, TimedOut>;

    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output> {
        let this = self.project();
        if let Poll::Ready(output) = this.inner.poll(cx) {
            return Poll::Ready(Ok(output));
        }
        match this.sleep.poll(cx) {
            Poll::Ready(()) => Poll::Ready(Err(TimedOut)),
            Poll::Pending => Poll::Pending,
        }
    }
}
```

The two tests go into the `tests` module:

```rust
    #[tokio::test]
    async fn fast_future_wins() {
        let result = with_timeout(Duration::from_millis(200), async { 7 }).await;
        assert_eq!(result, Ok(7));
    }

    #[tokio::test]
    async fn slow_future_times_out() {
        let slow = tokio::time::sleep(Duration::from_millis(500));
        let result = with_timeout(Duration::from_millis(20), slow).await;
        assert_eq!(result, Err(TimedOut));
    }
```

`cargo test` shows `test result: ok. 4 passed; 0 failed` for the library (with the two tests of Exercise 2). Why it works: `project()` gives `Pin<&mut F>` and `Pin<&mut Sleep>`, so both can be polled although neither is `Unpin`. Polling `inner` first means that a future which is ready at the same moment as the deadline still wins; `tokio::time::timeout` makes the same choice.

</details>

## Quiz

1. Why does a move break a self-referential value?

<details>
<summary>Answer</summary>

A move copies the bytes to a new address. A pointer stored inside the value still holds the old address, so after the move it points to a place where the field no longer is (a dangling pointer).

</details>

2. Why can the future of an `async fn` be self-referential?

<details>
<summary>Answer</summary>

Local variables that live across an `.await` are stored inside the future. If one of them is a reference to another local (`let first = &numbers[0];`), the future contains a pointer into itself after its first poll.

</details>

3. What does `Pin<&mut T>` forbid, and how?

<details>
<summary>Answer</summary>

Moving the `T` out or swapping it, because safe code cannot get a plain `&mut T` from it (unless `T: Unpin`). Without `&mut T` there is no `mem::swap` or `mem::replace`, so the value stays where it is until it is dropped.

</details>

4. What does `Unpin` mean, and which common types are not `Unpin`?

<details>
<summary>Answer</summary>

"Moving this value is safe even when it is pinned"; for such types `Pin` is a formality. Compiler-made futures (`async fn`, `async` blocks) and types containing `PhantomPinned`, such as `tokio::time::Sleep`, are not `Unpin`.

</details>

5. When do you use `pin!` and when `Box::pin`?

<details>
<summary>Answer</summary>

`pin!` when the future stays in the current function, for example a future polled again and again in a `select!` loop; it costs nothing. `Box::pin` when the future must be stored, returned or kept with futures of other types (`Pin<Box<dyn Future>>`); it costs a heap allocation.

</details>

6. What does `#[pin]` do in `pin_project!`?

<details>
<summary>Answer</summary>

It marks a field as structurally pinned: `project()` returns `Pin<&mut Field>` for it, so it can be polled even if it is not `Unpin`. Fields without `#[pin]` are returned as plain `&mut Field`.

</details>

## Summary

- A move copies bytes to a new address; a value with a pointer into itself is broken by it.
- Futures from `async fn` can hold references to their own locals across `.await`, so after the first poll they must not move.
- `Pin<P>` promises that the pointed-to value will not move again; `poll` takes `Pin<&mut Self>`, which enforces the rule. It costs nothing at run time.
- `Unpin` types (almost all normal types) ignore pinning; compiler-made futures and types with `PhantomPinned` are not `Unpin`.
- Pin with `pin!` inside one function (free) or with `Box::pin` to store or return a future (one allocation).
- "… cannot be unpinned" means "pin it first"; wrappers around futures use `pin-project-lite` for safe pin projection.

## Sources and further reading

- [std::pin module](https://doc.rust-lang.org/std/pin/index.html) — the official explanation of pinning, `Unpin` and projection, with all the rules.
- [std::pin::pin! macro](https://doc.rust-lang.org/std/pin/macro.pin.html) — pinning on the stack.
- [The Rust Programming Language — 17.5 A Closer Look at the Traits for Async](https://doc.rust-lang.org/book/ch17-05-traits-for-async.html) — `Pin` and `Unpin` in the official book.
- [pin-project-lite on docs.rs](https://docs.rs/pin-project-lite/0.2.17/pin_project_lite/) — the macro used in this lesson.
- [Tokio tutorial — Select](https://tokio.rs/tokio/tutorial/select) — `select!` in loops, with `&mut` and pinning.
- [Asynchronous Programming in Rust — Pinning](https://rust-lang.github.io/async-book/part-reference/pinning.html) — the chapter about pinning in the official async book.

---

[🏠 Start](../START-HERE.md) · [← 14.11 Build a tiny async runtime](11-build-a-tiny-async-runtime.md) · [📋 Phase 14](README.md) · [14.13 Tokio in depth: tasks and channels →](13-tokio-tasks-and-channels.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
