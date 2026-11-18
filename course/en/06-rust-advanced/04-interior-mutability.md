<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 06.3 Patterns in depth](03-patterns-in-depth.md) · [📋 Phase 06](README.md) · [06.5 Lifetimes in depth →](05-lifetimes-in-depth.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/06-rust-advanced/04-interior-mutability.md) <!-- nav -->

# Lesson 06.4 — Interior mutability

> **Phase 06 — Rust advanced** · Lesson 4 of 12 · ⏱ about 3 hours (2 sittings)

## What you will learn

- Why Rust sometimes needs to change a value through a shared reference `&T`.
- How `Cell<T>` and `RefCell<T>` move the borrowing check from compile time to run time.
- How `Rc<RefCell<T>>` gives several owners a value they can all change.
- `OnceCell`, `OnceLock`, `LazyCell` and `LazyLock`: values that are set once.
- Why these types are not thread safe, and what to use between threads.
- How to choose the right type, and the classic mistakes.

## Before you start

- You finished [Lesson 06.3 — Patterns in depth](03-patterns-in-depth.md).
- You know the borrowing rules ([Lesson 03.5](../03-ownership-and-borrowing/05-mutable-references-and-the-borrow-rules.md)), `Box` and `Rc` ([Lesson 05.9](../05-rust-intermediate/09-smart-pointers.md)) and threads with `Arc` and `Mutex` ([Lesson 05.10](../05-rust-intermediate/10-threads-channels-and-shared-state.md)). If you met `RefCell` there, this lesson takes it further; nothing is assumed.
- Create a practice project: `cargo new code/p06-interior-mutability`.

## New words

| Word | Meaning |
|---|---|
| **interior mutability** | Changing a value through a shared reference `&T`. The type itself guarantees that this is safe. |
| **`Cell<T>`** | A box that lets you `get` a copy and `set` a new value through `&`. For small `Copy` values. |
| **`RefCell<T>`** | A box that hands out `&T` and `&mut T` borrows and checks the borrowing rules while the program runs. |
| **`Ref` / `RefMut`** | The guard values returned by `borrow()` and `borrow_mut()`. While a guard exists, the borrow is active. |
| **run-time check** | A check done while the program runs, not while it compiles. A failed check makes the program panic. |
| **`OnceCell` / `OnceLock`** | A box that starts empty, can be filled exactly once, and then gives out shared references. `OnceLock` also works between threads. |
| **`LazyCell` / `LazyLock`** | A value computed by a closure the first time someone reads it. `LazyLock` works between threads. |
| **test double / mock** | A fake object used in a test instead of the real one. It often records how it was used. |
| **memoization** | Saving the result of a function so that the same call is not computed twice. |
| **`UnsafeCell<T>`** | The low-level building block under all the types in this lesson. The only legal way to change data behind `&` (needs `unsafe`). |
| **`Send` / `Sync`** | Marker traits. `Send`: a value can be moved to another thread. `Sync`: a shared reference `&T` can be used from many threads. |

## 1. The problem: the rule is too strict sometimes

Rust has a simple rule from [Lesson 03.5](../03-ownership-and-borrowing/05-mutable-references-and-the-borrow-rules.md): at any time you have **either many shared references (`&T`) or one mutable reference (`&mut T`)**. The compiler checks it before the program runs.

Sometimes a method looks read-only from the outside, but needs to remember something inside. A counter of visits is a good example:

```rust
struct Counter {
    visits: u32,
}

impl Counter {
    fn visit(&self) {
        self.visits += 1;
    }
}

fn main() {
    let counter = Counter { visits: 0 };
    counter.visit();
    println!("{}", counter.visits);
}
```

```text
error[E0594]: cannot assign to `self.visits`, which is behind a `&` reference
 --> src/main.rs:7:9
  |
7 |         self.visits += 1;
  |         ^^^^^^^^^^^^^^^^ `self` is a `&` reference, so it cannot be written to
  |
help: consider changing this to be a mutable reference
  |
6 |     fn visit(&mut self) {
  |               +++
```

**What happens:**

1. `visit(&self)` takes a shared reference, so it cannot write to `self.visits`.
2. The compiler suggests `&mut self`. That works, but then every caller needs a `mut` variable and exclusive access, even for something small like a counter.

**Interior mutability** is the solution: a type that allows changes through `&self` but keeps the rules safe in its own way. There are two ideas:

- **Make it impossible to break the rules** (`Cell`): never hand out a reference to the inside.
- **Check the rules while the program runs** (`RefCell`): count the active borrows, and panic on a violation.

> 📏 **Convention:** Interior mutability is a tool, not a default. Prefer ordinary `&mut self`. Use `Cell`/`RefCell` when the design needs shared access (several owners, a callback, a test double).

## 2. `Cell<T>`: replace the whole value

`Cell<T>` never gives you a reference to the value inside. You can only *copy the value out* (`get`) or *put a new value in* (`set`). Because no reference to the inside exists, nothing can be broken.

```rust
use std::cell::Cell;

struct Counter {
    visits: Cell<u32>,
}

impl Counter {
    fn visit(&self) {
        self.visits.set(self.visits.get() + 1);
    }
}

fn main() {
    let counter = Counter {
        visits: Cell::new(0),
    };
    counter.visit();
    counter.visit();
    println!("visits: {}", counter.visits.get());

    let old = counter.visits.replace(100);
    println!("replaced {old} with {}", counter.visits.get());

    let taken = counter.visits.take();
    println!("took {taken}, left {}", counter.visits.get());
}
```

```text
visits: 2
replaced 2 with 100
took 100, left 0
```

**What happens:**

1. `visits: Cell<u32>` replaces the plain `u32`. The method `visit(&self)` now compiles: `set` takes `&self`.
2. `self.visits.set(self.visits.get() + 1)` reads a copy, adds one and stores the result.
3. `replace(100)` stores 100 and returns the old value (2).
4. `take()` returns the current value (100) and leaves the default (`0` for numbers) in its place.
5. `Cell` has no run-time cost for borrow counting: it is as cheap as the plain value.

`get` needs `T: Copy`. For other types, use `replace` or `take`:

```text
error[E0599]: the method `get` exists for struct `Cell<String>`, but its trait bounds were not satisfied
 --> src/main.rs:5:21
  |
5 |     let copy = name.get();
  |                     ^^^
  |
  = note: the following trait bounds were not satisfied:
          `String: Copy`

For more information about this error, try `rustc --explain E0599`.
error: could not compile `demo` (bin "demo") due to 1 previous error
```

For a `Cell<String>` you would write `name.take()` (gets the text, leaves an empty `String`) or `name.replace(new_value)`. If you need to *look inside* a non-`Copy` value, use `RefCell`.

## 3. `RefCell<T>`: borrow checking at run time

`RefCell<T>` owns a value and lets you borrow it, like the compiler does, but the checks happen **while the program runs**:

- `borrow()` returns a `Ref<T>`: a shared borrow. You can have many at once.
- `borrow_mut()` returns a `RefMut<T>`: an exclusive borrow. You can have only one, and no `Ref` may exist at the same time.
- If you break the rule, the program **panics**. `try_borrow()` and `try_borrow_mut()` return a `Result` instead.

The guards give the borrow back when they are dropped (at the end of their scope), exactly like `MutexGuard` from [Lesson 05.10](../05-rust-intermediate/10-threads-channels-and-shared-state.md).

```rust
use std::cell::RefCell;

struct Logger {
    lines: RefCell<Vec<String>>,
}

impl Logger {
    fn log(&self, message: &str) {
        self.lines.borrow_mut().push(message.to_string());
    }

    fn count(&self) -> usize {
        self.lines.borrow().len()
    }
}

fn main() {
    let logger = Logger {
        lines: RefCell::new(Vec::new()),
    };
    logger.log("starting");
    logger.log("working");
    println!("{} lines", logger.count());
    println!("{:?}", logger.lines.borrow());

    // try_borrow_mut reports a conflict instead of panicking
    let first = logger.lines.borrow();
    match logger.lines.try_borrow_mut() {
        Ok(_) => println!("got a mutable borrow"),
        Err(e) => println!("cannot borrow mutably: {e}"),
    }
    println!("first borrow still sees {} lines", first.len());
}
```

```text
2 lines
["starting", "working"]
cannot borrow mutably: RefCell already borrowed
first borrow still sees 2 lines
```

**What happens:**

1. `lines: RefCell<Vec<String>>` lives inside a struct whose methods take `&self`.
2. `self.lines.borrow_mut().push(...)` takes an exclusive borrow for the length of that statement, pushes, and releases it.
3. `self.lines.borrow().len()` is a shared borrow for one statement.
4. `first` is a `Ref` kept alive on purpose. While it exists, `try_borrow_mut()` fails and returns `Err`, which prints `RefCell already borrowed`.
5. Compare this with a plain `&mut`: the compiler would have refused the program. `RefCell` accepts it and checks at run time instead.

If you use `borrow_mut()` while a borrow is alive, you get a panic, not an error message at compile time:

```rust
use std::cell::RefCell;

fn main() {
    let data = RefCell::new(vec![1, 2, 3]);
    let reader = data.borrow();
    println!("reader sees {} items", reader.len());
    data.borrow_mut().push(4); // the reader is still alive: this panics
    println!("never printed");
}
```

```text
reader sees 3 items

thread 'main' (4075730) panicked at src/main.rs:7:10:
RefCell already borrowed
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
```

> ⚠️ **Common mistake:** `RefCell` turns a compile error into a possible crash in production. Keep every `borrow()` / `borrow_mut()` short: do not store the guard in a variable that lives long, and do not call other code while you hold it.

> 🧪 **Testing:** `RefCell` is the standard tool for a *test double*. A real object takes `&self`, so a fake one needs to record calls through `&self` too. You will build one in Exercise 2.

## 4. `Rc<RefCell<T>>`: several owners that can change the value

`Rc<T>` gives shared ownership, but only read access. Put a `RefCell` inside and every owner can change the value:

```rust
use std::cell::RefCell;
use std::rc::Rc;

fn main() {
    // Rc: several owners. RefCell: mutation through a shared handle.
    let cart = Rc::new(RefCell::new(vec![String::from("bread")]));

    let alice = Rc::clone(&cart);
    let bob = Rc::clone(&cart);

    alice.borrow_mut().push(String::from("milk"));
    bob.borrow_mut().push(String::from("eggs"));

    println!("{:?}", cart.borrow());
    println!("owners: {}", Rc::strong_count(&cart));
}
```

```text
["bread", "milk", "eggs"]
owners: 3
```

**What happens:**

1. `Rc::new(RefCell::new(vec![...]))` creates one list on the heap with a reference counter and a borrow flag.
2. `Rc::clone(&cart)` does not copy the list. It creates another owner and increments the counter. Now `cart`, `alice` and `bob` point to the same list.
3. `alice.borrow_mut().push(...)` and `bob.borrow_mut().push(...)` change it one after the other. Each borrow ends at the semicolon.
4. `Rc::strong_count` shows 3 owners.

This pattern is common in graphs, trees with parent pointers, and event handlers. Be careful with cycles (an `Rc` that points back to itself through another `Rc`): the counter never reaches zero and the memory leaks.

## 5. Not for threads

`Cell`, `RefCell` and `Rc` check nothing about other threads. The compiler knows this through the marker traits `Send` and `Sync`: `Rc` is not `Send`, and `RefCell` is not `Sync`. So the compiler stops you:

```rust
use std::cell::RefCell;
use std::rc::Rc;
use std::thread;

fn main() {
    let counter = Rc::new(RefCell::new(0));
    let for_thread = Rc::clone(&counter);
    let handle = thread::spawn(move || {
        *for_thread.borrow_mut() += 1;
    });
    handle.join().unwrap();
    println!("{}", counter.borrow());
}
```

```text
error[E0277]: `Rc<RefCell<i32>>` cannot be sent between threads safely
  --> src/main.rs:8:32
   |
 8 |       let handle = thread::spawn(move || {
   |                    ------------- ^------
   |                    |             |
   |  __________________|_____________within this `{closure@src/main.rs:8:32: 8:39}`
   | |                  |
   | |                  required by a bound introduced by this call
 9 | |         *for_thread.borrow_mut() += 1;
10 | |     });
   | |_____^ `Rc<RefCell<i32>>` cannot be sent between threads safely
   |
   = help: within `{closure@src/main.rs:8:32: 8:39}`, the trait `Send` is not implemented for `Rc<RefCell<i32>>`
note: required because `for_thread` is used within this closure
  --> src/main.rs:9:10
   |
 9 |         *for_thread.borrow_mut() += 1;
   |          ^^^^^^^^^^
note: required by a bound in `spawn`
  --> /rustc/0ef1bc386059596dada8fb65b50425aab2ea637a/library/std/src/thread/functions.rs:125:0
```

The thread-safe counterparts have the same shape:

| Single thread | Several threads |
|---|---|
| `Rc<T>` | `Arc<T>` |
| `RefCell<T>` | `Mutex<T>` or `RwLock<T>` |
| `Cell<T>` | atomic types such as `AtomicU32` (Phase 14) |
| `OnceCell<T>` | `OnceLock<T>` |
| `LazyCell<T>` | `LazyLock<T>` |

```rust
use std::sync::{Arc, Mutex};
use std::thread;

fn main() {
    // The thread-safe version: Arc instead of Rc, Mutex instead of RefCell.
    let counter = Arc::new(Mutex::new(0));
    let for_thread = Arc::clone(&counter);
    let handle = thread::spawn(move || {
        *for_thread.lock().unwrap() += 1;
    });
    handle.join().unwrap();
    println!("{}", counter.lock().unwrap());
}
```

```text
1
```

**What happens:** this is the program from the error, with `Arc` and `Mutex`. `lock()` waits until no other thread holds the lock, and `unwrap()` handles the case where another thread panicked while holding it.

## 6. Values that are set once: `OnceCell`, `OnceLock`, `LazyCell`, `LazyLock`

Some values are written exactly once and then only read: a configuration loaded at start, a table computed on first use. For these, the standard library has special cells:

```rust
use std::cell::{LazyCell, OnceCell};
use std::sync::{LazyLock, OnceLock};

// A global that is set exactly once, from any thread.
static CONFIG: OnceLock<String> = OnceLock::new();

// A global that is computed the first time somebody reads it.
static PRIMES: LazyLock<Vec<u32>> = LazyLock::new(|| {
    println!("(computing primes)");
    (2..30).filter(|n| (2..*n).all(|d| n % d != 0)).collect()
});

fn main() {
    // OnceCell: single-threaded "write once, read many"
    let nickname: OnceCell<String> = OnceCell::new();
    println!("before: {:?}", nickname.get());
    println!("set: {:?}", nickname.set(String::from("Sasha")));
    println!("second set: {:?}", nickname.set(String::from("Alex")));
    println!("after: {:?}", nickname.get());

    // get_or_init: set on first use
    let cell: OnceCell<u32> = OnceCell::new();
    println!("{}", cell.get_or_init(|| 41 + 1));
    println!("{}", cell.get_or_init(|| 0)); // the closure does not run again

    // OnceLock: the same, but for statics and threads
    println!("{:?}", CONFIG.set(String::from("production")));
    println!("{:?}", CONFIG.set(String::from("testing")));
    println!("config = {}", CONFIG.get().unwrap());

    // LazyLock: initialised on the first access
    println!("before first use");
    println!("first prime: {}", PRIMES[0]);
    println!("{} primes below 30", PRIMES.len());

    // LazyCell: a lazy value without thread support
    let lazy = LazyCell::new(|| {
        println!("(lazy value computed)");
        7 * 6
    });
    println!("answer: {}", *lazy);
    println!("answer again: {}", *lazy);
}
```

```text
before: None
set: Ok(())
second set: Err("Alex")
after: Some("Sasha")
42
42
Ok(())
Err("testing")
config = production
before first use
(computing primes)
first prime: 2
10 primes below 30
(lazy value computed)
answer: 42
answer again: 42
```

**What happens:**

1. `OnceCell::new()` starts empty. `get()` returns `None`.
2. `set(value)` fills it and returns `Ok(())`. A second `set` returns `Err` with the rejected value: the cell never changes again.
3. `get_or_init(closure)` returns the value, and runs the closure only if the cell is still empty.
4. `static CONFIG: OnceLock<String>` is a global variable that can be filled exactly once, even if many threads try at the same moment. Plain `OnceCell` cannot be a `static`, because it is not `Sync`.
5. `static PRIMES: LazyLock<Vec<u32>>` holds a closure. It runs on the **first access** (`PRIMES[0]`), as you see from "(computing primes)" printed after "before first use". Later accesses reuse the result.
6. `LazyCell` is the same without thread support, for a local variable.

> 📏 **Convention:** For global constants that need work to build (`HashMap`, regular expressions, tables) use `LazyLock`. It replaces the older `lazy_static!` macro and most uses of the `once_cell` crate. Both are in the standard library now.

## 7. What is under the hood: `UnsafeCell`

All the types above are built on `std::cell::UnsafeCell<T>`. It is the only way in Rust to get a `*mut T` (a *raw pointer* that allows writing; Lesson 06.6 explains raw pointers) from a `&UnsafeCell<T>`. Without it the compiler assumes that data behind `&T` never changes and optimizes with that assumption; changing such data would be *undefined behaviour*: the rules of the language are broken, and the compiled program may do anything, including seeming to work until it suddenly does not. You do not use `UnsafeCell` directly, except when writing your own cell or lock (Lesson 06.6 explains `unsafe`).

## 8. Which type to use

| Type | Threads | Gives you | Cost | Use it for |
|---|---|---|---|---|
| `Cell<T>` | one | copies and replacement | none | Counters, flags, small `Copy` values |
| `RefCell<T>` | one | `&T` and `&mut T` checked at run time | a counter, panics on conflict | Test doubles, caches, graphs with `Rc` |
| `OnceCell<T>` / `LazyCell<T>` | one | write once, then `&T` | tiny | Values computed on first use |
| `OnceLock<T>` / `LazyLock<T>` | many | write once, then `&T` | small | Global configuration, tables |
| `Mutex<T>` / `RwLock<T>` | many | exclusive (or shared) access by locking | locking | Shared mutable state between threads |

## 9. A classic: caching with `&self`

A *memoizing* function remembers its results. With `RefCell` the cache can live inside a struct whose method takes `&self`:

```rust
use std::cell::RefCell;
use std::collections::HashMap;

struct Fib {
    cache: RefCell<HashMap<u64, u64>>,
    computed: RefCell<u32>,
}

impl Fib {
    fn new() -> Self {
        Fib {
            cache: RefCell::new(HashMap::new()),
            computed: RefCell::new(0),
        }
    }

    // &self, not &mut self: callers can share one Fib.
    fn get(&self, n: u64) -> u64 {
        // The temporary borrow ends at the end of this statement.
        let cached = self.cache.borrow().get(&n).copied();
        if let Some(value) = cached {
            return value;
        }
        let value = if n < 2 {
            n
        } else {
            self.get(n - 1) + self.get(n - 2)
        };
        *self.computed.borrow_mut() += 1;
        self.cache.borrow_mut().insert(n, value);
        value
    }
}

fn main() {
    let fib = Fib::new();
    println!("fib(50) = {}", fib.get(50));
    println!("computed {} values", fib.computed.borrow());
    println!("fib(50) again = {}", fib.get(50));
    println!("computed {} values", fib.computed.borrow());
}
```

```text
fib(50) = 12586269025
computed 51 values
fib(50) again = 12586269025
computed 51 values
```

**What happens:**

1. `get(&self, n)` looks in the cache. The temporary `Ref` from `self.cache.borrow()` is dropped at the end of the `let cached = ...;` statement, so the borrow is released **before** the recursive calls.
2. If the value is missing, `get` calls itself for `n - 1` and `n - 2`. Each of those borrows the cache briefly, too.
3. The result is stored with `borrow_mut().insert(...)`.
4. 51 values are computed for `fib(50)`, once each (the numbers 0 to 50). The second call reads the cache, so the counter stays at 51. Without the cache this recursion would need billions of calls.

Now the same function with a mistake: the guard is stored in a variable and lives until the end of the function.

```rust
use std::cell::RefCell;
use std::collections::HashMap;

struct Fib {
    cache: RefCell<HashMap<u64, u64>>,
}

impl Fib {
    fn get(&self, n: u64) -> u64 {
        let cache = self.cache.borrow(); // this Ref lives until the end of the function
        if let Some(value) = cache.get(&n) {
            return *value;
        }
        let value = if n < 2 {
            n
        } else {
            self.get(n - 1) + self.get(n - 2)
        };
        self.cache.borrow_mut().insert(n, value); // conflict with `cache`
        value
    }
}

fn main() {
    let fib = Fib {
        cache: RefCell::new(HashMap::new()),
    };
    println!("{}", fib.get(5));
}
```

```text

thread 'main' (4077050) panicked at src/main.rs:19:20:
RefCell already borrowed
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
```

**What happens:** the variable `cache` holds a `Ref` for the whole function. When the recursion reaches the base case, the line `self.cache.borrow_mut().insert(...)` asks for an exclusive borrow while this function's own shared borrow still exists. The compiler cannot see it; `RefCell` panics at run time. The fix is the version above: do not keep the guard; take what you need (`.copied()`) and let the guard drop.

> 🔒 **Security:** A panic that depends on input (for example, a request that triggers a double borrow) can be used to crash a server. Treat `RefCell` panics as bugs, and cover the code with tests.

## Common mistakes

### 1. Storing a `Ref` or `RefMut` in a long-lived variable

The guard keeps the borrow alive. Use the value in one expression, or copy what you need out and drop the guard (see the broken `Fib` example in section 9).

### 2. `Cell<String>` with `get`

`E0599 ... trait bounds were not satisfied: String: Copy`. Use `take()` or `replace()`, or use `RefCell`.

### 3. Sending `Rc<RefCell<T>>` to another thread

`E0277 ... cannot be sent between threads safely`. Switch to `Arc<Mutex<T>>`.

### 4. Reference cycles with `Rc`

Two `Rc`s that point to each other are never freed. Use `Weak<T>` for the back pointer (`Rc::downgrade`; see chapter 15.6 of the Rust Book in the sources below).

### 5. Using `RefCell` to avoid learning the borrow checker

If you reach for `Rc<RefCell<T>>` in every struct, the design probably needs a clearer owner. Use it when you really need shared ownership.

## Try it yourself

1. Open `~/projects/rust/code/p06-interior-mutability`.
2. Copy the `Logger` example from section 3 into `src/main.rs`.
3. Add a method `fn last(&self) -> Option<String>` that returns a **copy** of the last logged line (`self.lines.borrow().last().cloned()`).
4. Call `logger.last()` twice and print the results. You should see `Some("working")` twice.
5. Break it on purpose: in `main`, write `let _guard = logger.lines.borrow_mut();` and then call `logger.log("x")`. Run it and read the panic. Remove the line.
6. Run `cargo fmt` and `cargo clippy`, then commit: `git add code/p06-interior-mutability` and `git commit -m "feat(p06): logger with RefCell"`.

## Exercises

### Exercise 1 — A clicker with `Cell` (easy)

Create `struct Clicker` with a `Cell<u32>` field. Methods `click(&self)` (adds one) and `reset(&self) -> u32` (sets 0 and returns the old value). Write a test that clicks twice through a shared reference `&Clicker` and checks the count, and a test for `reset`.

<details>
<summary>Hint</summary>

`self.count.set(self.count.get() + 1)` for `click`, and `self.count.replace(0)` for `reset`.

</details>

<details>
<summary>Answer</summary>

```rust
use std::cell::Cell;

struct Clicker {
    count: Cell<u32>,
}

impl Clicker {
    fn new() -> Self {
        Clicker {
            count: Cell::new(0),
        }
    }

    fn click(&self) {
        self.count.set(self.count.get() + 1);
    }

    fn reset(&self) -> u32 {
        self.count.replace(0)
    }
}

fn main() {
    let clicker = Clicker::new();
    clicker.click();
    clicker.click();
    clicker.click();
    println!("count = {}", clicker.count.get());
    println!("reset returned {}", clicker.reset());
    println!("count = {}", clicker.count.get());
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn counts_clicks_through_a_shared_reference() {
        let clicker = Clicker::new();
        let shared = &clicker;
        shared.click();
        shared.click();
        assert_eq!(clicker.count.get(), 2);
    }

    #[test]
    fn reset_returns_old_value() {
        let clicker = Clicker::new();
        clicker.click();
        assert_eq!(clicker.reset(), 1);
        assert_eq!(clicker.count.get(), 0);
    }
}
```

```text
count = 3
reset returned 3
count = 0
```

```text

running 2 tests
..
test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

Why it works: `Cell::set` and `Cell::replace` take `&self`, so the methods need no `&mut self`. The test proves it by calling `click` through a `&Clicker`.

</details>

### Exercise 2 — A mock notifier (medium)

Given `trait Notifier { fn send(&self, message: &str); }` and `fn check_temperature(notifier: &dyn Notifier, celsius: i32)` (it sends `too hot: N` for 40 and above, and `too cold: N` for -20 and below), write a `MockNotifier` inside the test module. It records messages in a `RefCell<Vec<String>>`. Test that for the temperatures 20, 45 and -30 exactly two messages were sent.

<details>
<summary>Hint</summary>

`send(&self, ...)` takes `&self`, so the mock needs `RefCell` to push into its vector: `self.sent.borrow_mut().push(message.to_string())`. At the end compare with `*mock.sent.borrow()`.

</details>

<details>
<summary>Answer</summary>

```rust
trait Notifier {
    fn send(&self, message: &str);
}

struct PrintNotifier;

impl Notifier for PrintNotifier {
    fn send(&self, message: &str) {
        println!("NOTIFY: {message}");
    }
}

fn check_temperature(notifier: &dyn Notifier, celsius: i32) {
    if celsius >= 40 {
        notifier.send(&format!("too hot: {celsius}"));
    } else if celsius <= -20 {
        notifier.send(&format!("too cold: {celsius}"));
    }
}

fn main() {
    let notifier = PrintNotifier;
    for t in [20, 45, -30] {
        check_temperature(&notifier, t);
    }
}

#[cfg(test)]
mod tests {
    use super::*;
    use std::cell::RefCell;

    // A test double: it records what it was asked to send.
    struct MockNotifier {
        sent: RefCell<Vec<String>>,
    }

    impl Notifier for MockNotifier {
        fn send(&self, message: &str) {
            self.sent.borrow_mut().push(message.to_string());
        }
    }

    #[test]
    fn sends_only_for_extreme_values() {
        let mock = MockNotifier {
            sent: RefCell::new(Vec::new()),
        };
        check_temperature(&mock, 20);
        check_temperature(&mock, 45);
        check_temperature(&mock, -30);
        assert_eq!(*mock.sent.borrow(), vec!["too hot: 45", "too cold: -30"]);
    }
}
```

```text
NOTIFY: too hot: 45
NOTIFY: too cold: -30
```

```text

running 1 test
.
test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

Why it works: the function takes `&dyn Notifier` ([Lesson 06.1](01-trait-objects-and-dynamic-dispatch.md)), so a test can pass the mock instead of the printing notifier. The mock is defined in the test module, so it does not appear in the real program.

</details>

### Exercise 3 — Two closures, one counter (medium)

Write `fn make_counter() -> (impl Fn(), impl Fn() -> u32)`. The first closure adds one to a hidden counter. The second returns the current value. Both closures must share the same counter. Test that after two increments `read()` returns 2.

<details>
<summary>Hint</summary>

Make `let count = Rc::new(Cell::new(0));`, clone the `Rc` once for the first closure, and move the original into the second. Both closures need the `move` keyword.

</details>

<details>
<summary>Answer</summary>

```rust
use std::cell::Cell;
use std::rc::Rc;

// Two closures that share one counter.
fn make_counter() -> (impl Fn(), impl Fn() -> u32) {
    let count = Rc::new(Cell::new(0));
    let for_increment = Rc::clone(&count);
    let increment = move || for_increment.set(for_increment.get() + 1);
    let read = move || count.get();
    (increment, read)
}

fn main() {
    let (increment, read) = make_counter();
    increment();
    increment();
    println!("read() = {}", read());
    increment();
    println!("read() = {}", read());
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn read_sees_increments() {
        let (increment, read) = make_counter();
        assert_eq!(read(), 0);
        increment();
        increment();
        assert_eq!(read(), 2);
    }
}
```

```text
read() = 2
read() = 3
```

```text

running 1 test
.
test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

Why it works: `Rc` gives both closures ownership of the same `Cell`, and `Cell` allows `set` through a shared handle. Each closure only needs `Fn`, not `FnMut`.

</details>

### Exercise 4 — Settings loaded once (hard)

Write `fn settings() -> &'static HashMap<&'static str, u32>` that builds the map `{"port": 8080, "workers": 4}` the first time it is called and returns the same map every time. Print `(loading settings)` inside the initializer. Write a test that two calls return the same address and that `port` is 8080.

<details>
<summary>Hint</summary>

Use `static SETTINGS: OnceLock<HashMap<...>> = OnceLock::new();` and `SETTINGS.get_or_init(|| ...)`. Compare addresses with `std::ptr::eq(a, b)`.

</details>

<details>
<summary>Answer</summary>

```rust
use std::collections::HashMap;
use std::sync::OnceLock;

static SETTINGS: OnceLock<HashMap<&'static str, u32>> = OnceLock::new();

fn settings() -> &'static HashMap<&'static str, u32> {
    SETTINGS.get_or_init(|| {
        println!("(loading settings)");
        HashMap::from([("port", 8080), ("workers", 4)])
    })
}

fn main() {
    println!("port = {}", settings()["port"]);
    println!("workers = {}", settings()["workers"]);
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn settings_are_loaded_once_and_shared() {
        let first = settings();
        let second = settings();
        assert!(std::ptr::eq(first, second));
        assert_eq!(first["port"], 8080);
    }
}
```

```text
(loading settings)
port = 8080
workers = 4
```

```text

running 1 test
.
test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

Why it works: `get_or_init` runs the closure only for the first caller (even with several threads) and returns `&'static` to the stored value afterwards. `std::ptr::eq` shows that both calls got the very same map.

</details>

## Quiz

1. What does interior mutability mean?

<details>
<summary>Answer</summary>

Changing a value through a shared reference `&T`. The type (`Cell`, `RefCell`, `Mutex`, ...) makes this safe, either by never giving out references or by checking the borrowing rules in another way.

</details>

2. Why can `Cell<T>::get` be used only when `T: Copy`?

<details>
<summary>Answer</summary>

`get` returns a copy of the value. For a type that is not `Copy` (like `String`) that would be a move or an expensive clone, which `Cell` does not do. Use `take`, `replace`, or `RefCell`.

</details>

3. When does `RefCell` panic?

<details>
<summary>Answer</summary>

When you call `borrow_mut()` while any other borrow is active, or `borrow()` while a `RefMut` is active. The check happens while the program runs.

</details>

4. What is the difference between `borrow_mut()` and `try_borrow_mut()`?

<details>
<summary>Answer</summary>

`borrow_mut()` panics if the value is already borrowed. `try_borrow_mut()` returns a `Result`: `Err` when a borrow is active.

</details>

5. What does `Rc<RefCell<Vec<i32>>>` give you that `Rc<Vec<i32>>` does not?

<details>
<summary>Answer</summary>

All owners can change the vector. `Rc<Vec<i32>>` shares the vector but only for reading.

</details>

6. Why can you not send an `Rc<RefCell<i32>>` to another thread, and what do you use?

<details>
<summary>Answer</summary>

`Rc` uses a reference counter that is not thread safe, so it is not `Send`; `RefCell` is not `Sync`. Use `Arc<Mutex<i32>>` (or `Arc<RwLock<i32>>`).

</details>

7. What is the difference between `OnceLock` and `LazyLock`?

<details>
<summary>Answer</summary>

`OnceLock` starts empty and you fill it by calling `set` or `get_or_init`. `LazyLock` is created together with its initializer closure and computes the value automatically on first access.

</details>

8. Why is holding a `Ref` in a long-lived variable dangerous?

<details>
<summary>Answer</summary>

The borrow stays active as long as the guard lives. Any `borrow_mut()` during that time panics. Keep guards short, or copy out what you need and let the guard drop.

</details>

## Summary

- Interior mutability changes a value through `&T`. It is a tool for shared access, not a default.
- `Cell<T>` copies values in and out, with no references and no cost. `RefCell<T>` lends `&T`/`&mut T` and checks the rules at run time; a violation panics.
- `Rc<RefCell<T>>` is shared ownership plus mutation; watch for reference cycles.
- `OnceCell`/`OnceLock` are filled once; `LazyCell`/`LazyLock` compute on first use. Prefer `LazyLock` over `lazy_static!`.
- These single-thread types are not `Send`/`Sync`. Between threads use `Arc`, `Mutex`, `RwLock`, `OnceLock`, `LazyLock` (and atomics, Phase 14).
- Keep `Ref`/`RefMut` guards short. Do not hold a borrow across a call that may borrow again.
- Everything is built on `UnsafeCell`.

## Sources and further reading

- [The Rust Programming Language — 15.5 RefCell<T> and the Interior Mutability Pattern](https://doc.rust-lang.org/book/ch15-05-interior-mutability.html) — the official explanation, including `Rc<RefCell<T>>`.
- [The Rust Programming Language — 15.6 Reference Cycles Can Leak Memory](https://doc.rust-lang.org/book/ch15-06-reference-cycles.html) — how cycles leak and how `Weak<T>` prevents it.
- [std::cell module](https://doc.rust-lang.org/std/cell/index.html) — an overview of `Cell`, `RefCell`, `OnceCell`, `LazyCell` and `UnsafeCell`, with a guide to choosing between them.
- [std::cell::RefCell](https://doc.rust-lang.org/std/cell/struct.RefCell.html) — all methods, including `try_borrow`.
- [std::sync::OnceLock](https://doc.rust-lang.org/std/sync/struct.OnceLock.html) — a thread-safe write-once cell.
- [std::sync::LazyLock](https://doc.rust-lang.org/std/sync/struct.LazyLock.html) — lazy statics in the standard library.
- [std::cell::UnsafeCell](https://doc.rust-lang.org/std/cell/struct.UnsafeCell.html) — the foundation of interior mutability.

---

[🏠 Start](../START-HERE.md) · [← 06.3 Patterns in depth](03-patterns-in-depth.md) · [📋 Phase 06](README.md) · [06.5 Lifetimes in depth →](05-lifetimes-in-depth.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
