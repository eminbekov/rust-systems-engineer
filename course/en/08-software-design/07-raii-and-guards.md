<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 08.6 The builder pattern](06-builder-pattern.md) · [📋 Phase 08](README.md) · [08.8 State machines and the typestate pattern →](08-state-machines-and-typestate.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/08-software-design/07-raii-and-guards.md) <!-- nav -->

# Lesson 08.7 — RAII and guards

> **Phase 08 — Software design: principles, patterns and conventions** · Lesson 7 of 17 · ⏱ about 135 minutes

## What you will learn

- What RAII means and why it makes cleanup impossible to forget.
- How to write your own **guard**: a rollback transaction and a connection that returns itself to a pool.
- The exact rules for the order in which values are dropped, and the `let _ =` trap.
- What happens when a guard is held too long, and how to shorten its life.
- Why `Drop` cannot report errors, and the "explicit `finish`, `Drop` as a safety net" pattern.

## Before you start

- You finished [Lesson 08.6 — The builder pattern](06-builder-pattern.md).
- You know `Drop` and `drop(x)` from [Lesson 05.9 — Smart pointers](../05-rust-intermediate/09-smart-pointers.md), `Mutex` from [Lesson 05.10](../05-rust-intermediate/10-threads-channels-and-shared-state.md), `RefCell` from [Lesson 06.4](../06-rust-advanced/04-interior-mutability.md), and lifetimes in structs from [Lesson 05.3](../05-rust-intermediate/03-lifetimes.md).

## New words

| Word | Meaning |
|---|---|
| **resource** | Something that is taken and must be given back: memory, an open file, a lock, a database connection, a slot in a pool. |
| **RAII** (Resource Acquisition Is Initialization) | A design where a value *takes* a resource when it is created and *gives it back* when it is dropped. Because Rust drops values automatically, the cleanup cannot be forgotten. |
| **guard** | A value whose life means "I am holding something". While it exists, the resource is held; when it is dropped, the resource is released. `MutexGuard` is the classic one. |
| **scope guard** | A guard written to run some code at the end of a scope, whatever way the scope ends (normal end, `return`, `?`, panic). |
| **rollback** | Undoing the changes of an unfinished operation, so that the data returns to its earlier state. |
| **commit** | Declaring the changes of an operation final. |
| **temporary** | A value created in the middle of an expression and never stored in a variable. It is dropped at the end of the statement. |
| **unwinding** | What a panic does by default: it walks back up the stack, dropping every live value on the way. |
| **leak** | A value that is never dropped, so its cleanup never runs (`std::mem::forget` does this on purpose). It is safe in Rust, but rarely wanted. |

## 1. RAII: the cleanup that cannot be forgotten

Every resource must be given back: a file closed, a lock released, a connection returned. If you must remember to write `close()` on every path out of a function, including early returns and errors, you will forget one sooner or later.

RAII removes the problem. You put the release code in `Drop`. The value is dropped at the end of its scope, however the scope ends, and the compiler inserts the call for you (Lesson 05.9). The standard library is built on this:

| Type | Resource | Released when the value is dropped |
|---|---|---|
| `File` | An open file | The file is closed |
| `MutexGuard` (from `mutex.lock()`) | The right to use the data inside the `Mutex` | The lock is released |
| `Ref` / `RefMut` (from `RefCell::borrow` / `borrow_mut`) | A shared or exclusive borrow | The borrow counter goes back down |
| `Box`, `Vec`, `String` | Heap memory | The memory is freed |

A **guard** is a value of this kind, whose purpose is to hold something for a while. You do not need the word `unsafe` or any special feature to write one: a struct and `impl Drop` are enough.

## 2. A scope guard: transactions with rollback

Think about a bank transfer made of several steps. If the third step fails, the first two must be undone. Without RAII you must write the undo code on every error path. With a guard, the undo is written once:

```rust
struct Account {
    balance: i64,
}

struct Transaction<'a> {
    account: &'a mut Account,
    original: i64,
    committed: bool,
}

impl Account {
    fn begin(&mut self) -> Transaction<'_> {
        let original = self.balance;
        Transaction {
            account: self,
            original,
            committed: false,
        }
    }
}

impl Transaction<'_> {
    fn withdraw(&mut self, amount: i64) -> Result<(), String> {
        if self.account.balance < amount {
            return Err(format!(
                "cannot withdraw {amount}: balance is {}",
                self.account.balance
            ));
        }
        self.account.balance -= amount;
        Ok(())
    }

    fn commit(mut self) {
        self.committed = true; // `self` is dropped at the end of this method
    }
}

impl Drop for Transaction<'_> {
    fn drop(&mut self) {
        if !self.committed {
            self.account.balance = self.original;
            println!("rolled back to {}", self.original);
        }
    }
}

fn pay(account: &mut Account, amounts: &[i64]) -> Result<(), String> {
    let mut tx = account.begin();
    for amount in amounts {
        tx.withdraw(*amount)?; // an early return drops `tx` without commit
    }
    tx.commit();
    Ok(())
}

fn main() {
    let mut account = Account { balance: 100 };
    println!("{:?}", pay(&mut account, &[30, 20]));
    println!("balance: {}", account.balance);
    println!("{:?}", pay(&mut account, &[30, 40]));
    println!("balance: {}", account.balance);
}
```

**What happens:**

1. `account.begin()` creates a `Transaction` that borrows the account mutably (`&'a mut Account`) and remembers the original balance.
2. `withdraw` changes the balance, or returns an error if the money is not enough.
3. `commit(mut self)` marks the transaction as final. It takes `self` by value, so the transaction is dropped at the end of `commit`, and `Drop` sees `committed == true` and does nothing.
4. `impl Drop for Transaction`: if the transaction was **not** committed, it puts the original balance back and prints a message.
5. In `pay`, the `?` returns early when a withdrawal fails. The early return drops `tx`, and the guard rolls back. There is no undo code in `pay` at all.
6. `main` makes two payments. The first (30 + 20) succeeds. The second (30 + 40) fails at 40, because only 20 is left after the first 30, and the balance returns to 50.

```text
Ok(())
balance: 50
rolled back to 50
Err("cannot withdraw 40: balance is 20")
balance: 50
```

Notice the order: `rolled back to 50` is printed when `pay` returns, before `main` prints the `Err`. And the program never has a half-finished state in `account.balance`, because the guard also covers a panic in the middle of `pay`.

> 🧪 **Testing:** A guard is easy to test through its effect: run the failing path, then check the data. Here are two tests for `pay`, placed in the same file:

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn successful_payment_is_committed() {
        let mut account = Account { balance: 100 };
        assert!(pay(&mut account, &[30, 20]).is_ok());
        assert_eq!(account.balance, 50);
    }

    #[test]
    fn failed_payment_is_rolled_back_completely() {
        let mut account = Account { balance: 100 };
        assert!(pay(&mut account, &[60, 60]).is_err());
        assert_eq!(account.balance, 100);
    }
}
```

They pass:

```text
running 2 tests
test tests::failed_payment_is_rolled_back_completely ... ok
test tests::successful_payment_is_committed ... ok

test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

## 3. A guard that gives back: a pool

Opening a database connection is slow, so servers keep a **pool** of ready connections. A client takes one, uses it, and must return it. A guard makes returning automatic:

```rust
use std::cell::RefCell;

struct Pool {
    free: RefCell<Vec<u32>>,
}

struct Conn<'a> {
    pool: &'a Pool,
    id: u32,
}

impl Pool {
    fn new(size: u32) -> Pool {
        Pool {
            free: RefCell::new((1..=size).collect()),
        }
    }

    fn acquire(&self) -> Option<Conn<'_>> {
        let id = self.free.borrow_mut().pop()?;
        println!("take {id}");
        Some(Conn { pool: self, id })
    }
}

impl Drop for Conn<'_> {
    fn drop(&mut self) {
        println!("give back {}", self.id);
        self.pool.free.borrow_mut().push(self.id);
    }
}

fn main() {
    let pool = Pool::new(2);
    let a = pool.acquire().unwrap();
    let b = pool.acquire().unwrap();
    println!("third: {}", pool.acquire().is_none());
    drop(a);
    let c = pool.acquire().unwrap();
    println!("c has id {}, b has id {}", c.id, b.id);
}
```

**What happens:**

1. `Pool` keeps the free ids in a `RefCell<Vec<u32>>` (interior mutability, Lesson 06.4), so `acquire` can take `&self` and many guards can share one pool.
2. `acquire` pops an id and returns a `Conn` guard that holds `&Pool`. If no id is left, `pop()?` returns `None` early.
3. `impl Drop for Conn` pushes the id back into the pool.
4. `main` takes both ids (2, then 1), sees that a third request fails, drops `a` explicitly (which returns id 2), and the next `acquire` gets that same id back.
5. At the end of `main` the remaining guards are dropped in reverse order of creation (`c`, then `b`).

```text
take 2
take 1
third: true
give back 2
take 2
c has id 2, b has id 1
give back 2
give back 1
```

The lifetime `'_` in `Conn<'_>` connects the guard to the pool: the compiler will not let a guard outlive the pool it belongs to. In a real program a pool shared between threads would use `Mutex` instead of `RefCell`; the guard idea is the same.

## 4. The rules of dropping, and the `let _` trap

You learned the main rule in Lesson 05.9: values are dropped when they go out of scope, in the reverse order of their creation. Here is the complete list:

- **Local variables** are dropped in the reverse order of declaration.
- **Struct fields** are dropped in the order they are written in the struct, *after* the struct's own `drop` has run.
- **Temporaries** are dropped at the end of the statement that created them.
- A value that was **moved** is dropped by its new owner, not by the old one. `drop(x)` moves `x` into a function that ends at once.
- Function parameters are dropped when the function returns.

Now the trap that catches almost everybody once:

```rust
struct Guard(&'static str);

impl Drop for Guard {
    fn drop(&mut self) {
        println!("release {}", self.0);
    }
}

fn main() {
    let _ = Guard("underscore"); // not a variable: dropped at once
    println!("after `let _ =`");
    let _named = Guard("named"); // a variable: lives until the end of the block
    println!("after `let _named =`");
}
```

**What happens:**

1. `let _ = Guard("underscore");` does **not** create a variable. The underscore means "ignore this value", so the guard is a temporary and is dropped at the end of the statement.
2. `let _named = Guard("named");` creates a variable (the leading underscore only silences the "unused variable" warning), so the guard lives until the end of `main`.

```text
release underscore
after `let _ =`
after `let _named =`
release named
```

If you write `let _ = mutex.lock().unwrap();`, the lock is released immediately and you are not protected at all. Always name a guard: `let _guard = ...`.

> ⚠️ **Common mistake:** `let _ = guard;` and `let _guard = guard;` look almost the same and behave very differently. When the *life* of a value matters, give it a name.

Also remember that Rust does not guarantee that `Drop` runs in every case. It does not run if the process is killed, if `std::process::exit` is called, if the program aborts, or if the value is leaked with `std::mem::forget`. During a panic, values *are* dropped (unwinding), unless the program was built to abort on panic.

> 🔒 **Security:** Do not rely on `Drop` for something that must happen even if the program is killed, such as saving the only copy of important data. Use it to release resources and keep invariants, not as your only protection.

## 5. A guard held too long

A guard holds a resource for its whole life, and a variable lives until the end of its block. That is often longer than you intend:

```rust
use std::cell::RefCell;

fn main() {
    let cell = RefCell::new(vec![1, 2, 3]);
    let first = cell.borrow_mut(); // a guard: lives until the end of main
    println!("first guard holds {} items", first.len());
    let second = cell.borrow_mut(); // a second guard at the same time
    println!("never printed: {}", second.len());
}
```

```text
first guard holds 3 items

thread 'main' (5600741) panicked at src/main.rs:7:23:
RefCell already borrowed
```

**What happens:**

1. `cell.borrow_mut()` returns a `RefMut` guard. The `RefCell` remembers "someone is borrowing this exclusively".
2. The first guard is still alive (it lives until the end of `main`), so the second `borrow_mut()` finds the cell busy and panics. With a `Mutex`, asking for a second lock in the same thread can **deadlock** (wait forever), which is worse than a panic.

The cure is to end the guard's life as early as possible:

- Put the guarded work in an inner block `{ ... }`; the guard is dropped at the closing brace.
- Or call `drop(guard)` explicitly.
- Or avoid naming the guard when one statement is enough: `cell.borrow_mut().push(4);` (the guard is a temporary and is dropped at the semicolon).

Exercise 2 asks you to fix such a program.

> ⚡ **Performance:** A lock that is held while you do slow work (reading a file, waiting for the network) makes every other thread wait. Take the lock, copy or change what you need, and drop the guard before the slow part.

## 6. `Drop` cannot fail

The `drop` method returns nothing. It cannot tell the caller "closing failed", and it must not panic (a panic while another panic is unwinding aborts the whole program). But real cleanup can fail: flushing a file to disk can return an error.

The standard pattern has two parts:

1. An **explicit method** that consumes the value and returns a `Result`: `finish()`, `close()`, `commit()`. The caller can handle the error.
2. A `Drop` implementation as a **safety net**: it does a best-effort cleanup (or only a warning), because the caller may have forgotten, or an early return may have happened.

```rust
struct Report {
    lines: Vec<String>,
    finished: bool,
}

impl Report {
    fn new() -> Report {
        Report {
            lines: Vec::new(),
            finished: false,
        }
    }

    fn add(&mut self, line: &str) {
        self.lines.push(line.to_string());
    }

    // The explicit way: can report a problem to the caller.
    fn finish(mut self) -> Result<usize, String> {
        if self.lines.is_empty() {
            return Err("empty report".to_string()); // `self` is dropped here, not finished
        }
        self.finished = true;
        println!("saved {} lines", self.lines.len());
        Ok(self.lines.len())
    }
}

// The safety net: cannot return an error, so it only warns.
impl Drop for Report {
    fn drop(&mut self) {
        if !self.finished {
            println!("warning: report dropped without finish()");
        }
    }
}

fn main() {
    let mut good = Report::new();
    good.add("hello");
    println!("{:?}", good.finish());

    let mut forgotten = Report::new();
    forgotten.add("lost");
    println!("end of main");
}
```

**What happens:**

1. `finish(mut self)` takes the report by value, does the real work, can return `Err`, and sets `finished` to `true` on success.
2. `Drop` checks the flag. If `finish` was never called (or failed), it warns.
3. `good.finish()` prints `saved 1 lines`; no warning appears for it, because `finished` is `true` when the value is dropped at the end of `finish`.
4. `forgotten` is never finished, so at the end of `main` its `drop` prints the warning.

```text
saved 1 lines
Ok(1)
end of main
warning: report dropped without finish()
```

The standard library uses exactly this design: `BufWriter` flushes in `drop` as a best effort and ignores errors, so the documentation tells you to call `flush()` yourself when you care about errors.

**When NOT to use RAII and `Drop`:**

- If the cleanup is optional or can fail in ways the caller must handle, give it an explicit method (above).
- Do not hide heavy, slow or surprising work in `drop`. It runs silently, at scope end.
- Do not use `Drop` for ordinary program logic ("when this value disappears, send an email"). The place where the code runs becomes invisible.
- Do not write a guard for a resource that has no "held" period. A plain function is simpler.

## Common mistakes

**Mistake 1: `let _ = guard`.** See Section 4. The guard is dropped at once.

**Mistake 2: `Drop` with a panic or an `unwrap` inside.** A failing `drop` during unwinding aborts the program. Log and continue instead.

**Mistake 3: a guard in a long-lived variable.** A lock guard stored in a struct field or a variable at the top of `main` is held for the whole program.

**Mistake 4: calling `.drop()`.** You cannot: `a.drop()` is error E0040 (explicit destructor calls are not allowed). Use `drop(a)` (Lesson 05.9).

## Try it yourself

1. Create the package and copy the transaction example of Section 2 (`Account`, `Transaction` and `main`) into `src/main.rs`:

```bash
cd ~/projects/rust
cargo new code/p08-raii
cd code/p08-raii
```

2. Run it with `cargo run`. You see the five lines from Section 2.

3. Add the two tests from the 🧪 box and run `cargo test`. They pass.

4. Break the guard on purpose: in `Drop`, remove the `if !self.committed` check, so the guard *always* rolls back. Run `cargo test` again. Which test fails, and why? Put the check back.

5. Save your work:

```bash
cargo fmt
cargo clippy
git add .
git commit -m "feat(p08): transaction guard"
```

**What happens:** In step 4 the test `successful_payment_is_committed` fails, because the guard rolls back a transaction that was committed. This shows that the tests describe the contract of the guard in both directions.

## Exercises

### Exercise 1 — Predict the drops (easy)

Read the program below and write down the output **before** you run it. Then run it and compare.

```rust
struct Noisy(&'static str);

impl Drop for Noisy {
    fn drop(&mut self) {
        println!("drop {}", self.0);
    }
}

fn make() -> Noisy {
    let _temp = Noisy("temp");
    Noisy("result")
}

fn main() {
    let a = Noisy("a");
    let _b = make();
    {
        let _c = Noisy("c");
        println!("inner");
    }
    let _ = Noisy("d");
    println!("end");
    drop(a);
}
```

<details>
<summary>Hint</summary>

Which values are temporaries or locals of `make`? What does `let _ =` do? Which variables are still alive at the end of `main`?

</details>

<details>
<summary>Answer</summary>

```text
drop temp
inner
drop c
drop d
end
drop a
drop result
```

Why it works: `_temp` is dropped when `make` returns (`drop temp`). The inner block ends before `drop c`. `let _ = Noisy("d")` drops at once, so `drop d` comes before `end`. `drop(a)` is explicit. `_b`, the last variable alive, is dropped at the end of `main` (`drop result`).

</details>

### Exercise 2 — Shorten the guard (medium)

This program panics. Find out why, and fix it **without** changing `total`.

```rust
use std::cell::RefCell;

fn total(cell: &RefCell<Vec<i32>>) -> i32 {
    cell.borrow().iter().sum()
}

fn main() {
    let cell = RefCell::new(vec![1, 2]);
    let mut items = cell.borrow_mut();
    items.push(3);
    println!("total: {}", total(&cell));
}
```

<details>
<summary>Hint</summary>

`items` is a `RefMut` guard that lives until the end of `main`. Where could you end its life earlier?

</details>

<details>
<summary>Answer</summary>

```rust
use std::cell::RefCell;

fn total(cell: &RefCell<Vec<i32>>) -> i32 {
    cell.borrow().iter().sum()
}

fn main() {
    let cell = RefCell::new(vec![1, 2]);
    {
        let mut items = cell.borrow_mut();
        items.push(3);
    } // the guard is dropped here
    println!("total: {}", total(&cell));
}
```

Why it works: the inner block drops `items` at its closing brace, so `total` can borrow the cell again. `drop(items);` before the `println!` would work too.

</details>

### Exercise 3 — Count the active tickets (hard)

Write `Counter` with a method `enter()` that returns a `Ticket` guard. `counter.active()` must return how many tickets are alive right now. **Write a test**: 0 at the start, 2 after two tickets, 1 after one is dropped.

<details>
<summary>Hint</summary>

Use `Cell<u32>` for the counter (it allows changes through `&self`). `enter` adds one, and `Ticket`'s `Drop` subtracts one.

</details>

<details>
<summary>Answer</summary>

```rust
use std::cell::Cell;

struct Counter {
    active: Cell<u32>,
}

struct Ticket<'a> {
    counter: &'a Counter,
}

impl Counter {
    fn new() -> Counter {
        Counter {
            active: Cell::new(0),
        }
    }

    fn enter(&self) -> Ticket<'_> {
        self.active.set(self.active.get() + 1);
        Ticket { counter: self }
    }

    fn active(&self) -> u32 {
        self.active.get()
    }
}

impl Drop for Ticket<'_> {
    fn drop(&mut self) {
        self.counter.active.set(self.counter.active.get() - 1);
    }
}

fn main() {
    let counter = Counter::new();
    let first = counter.enter();
    let _second = counter.enter();
    println!("active: {}", counter.active());
    drop(first);
    println!("active: {}", counter.active());
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn active_follows_the_lifetime_of_the_tickets() {
        let counter = Counter::new();
        assert_eq!(counter.active(), 0);
        let first = counter.enter();
        let _second = counter.enter();
        assert_eq!(counter.active(), 2);
        drop(first);
        assert_eq!(counter.active(), 1);
    }
}
```

Why it works: the count is not stored separately by hand; it follows the lives of the guards. It cannot go out of step, even when a function returns early. This is how a "number of active requests" gauge is often built.

</details>

## Quiz

1. What does RAII stand for, and what does it guarantee in Rust?

<details>
<summary>Answer</summary>

Resource Acquisition Is Initialization. The resource is released in `drop` when the value goes out of scope, however the scope ends, so cleanup cannot be forgotten.

</details>

2. What is the difference between `let _ = guard;` and `let _guard = guard;`?

<details>
<summary>Answer</summary>

`let _ =` drops the value at once; `let _guard =` keeps it alive until the end of the scope.

</details>

3. In what order are local variables dropped?

<details>
<summary>Answer</summary>

In the reverse order of declaration (the last created is dropped first).

</details>

4. Why should `Drop` not be the only place for cleanup that can fail?

<details>
<summary>Answer</summary>

`drop` cannot return an error, and it must not panic. An explicit method returning `Result` lets the caller handle failures; `Drop` is the safety net.

</details>

5. Name two situations in which `Drop` does not run.

<details>
<summary>Answer</summary>

`std::process::exit`, an abort, a killed process, or `std::mem::forget` on the value.

</details>

## Summary

- **RAII**: a value takes a resource when created and gives it back in `Drop`, so cleanup is written once and cannot be forgotten.
- A **guard** is a struct with `impl Drop` that represents "I am holding something": a lock, a borrow, a connection, a transaction.
- A rollback guard undoes unfinished work on every exit path, including early `return`, `?` and panics; `commit(self)` disarms it.
- Variables drop in reverse order; fields in declaration order; temporaries at the end of the statement. `let _ = x` drops immediately, `let _x = x` does not.
- Keep guards short-lived: inner blocks, `drop(guard)` or temporaries. Do not hold locks across slow work.
- `Drop` cannot fail or be called by hand. Pair an explicit `finish`/`close` that returns `Result` with a `Drop` safety net.
- `Drop` does not run on `process::exit`, abort or `mem::forget`, so it is not a place for must-happen logic.

## Sources and further reading

- [Rust Design Patterns — RAII with guards](https://rust-unofficial.github.io/patterns/patterns/behavioural/RAII.html) — the pattern with a `Mutex` example.
- [std::ops::Drop](https://doc.rust-lang.org/std/ops/trait.Drop.html) — the trait, with the drop order rules and notes on panics.
- [The Rust Reference — Destructors](https://doc.rust-lang.org/reference/destructors.html) — the exact rules for when and in which order values are dropped.
- [std::sync::MutexGuard](https://doc.rust-lang.org/std/sync/struct.MutexGuard.html) — the most-used guard in the standard library.
- [std::mem::forget](https://doc.rust-lang.org/std/mem/fn.forget.html) — why leaking is allowed, and what it means for guards.

---

[🏠 Start](../START-HERE.md) · [← 08.6 The builder pattern](06-builder-pattern.md) · [📋 Phase 08](README.md) · [08.8 State machines and the typestate pattern →](08-state-machines-and-typestate.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
