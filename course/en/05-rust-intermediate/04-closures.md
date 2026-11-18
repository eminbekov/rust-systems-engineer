<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 05.3 Lifetimes](03-lifetimes.md) · [📋 Phase 05](README.md) · [05.5 Iterators →](05-iterators.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/05-rust-intermediate/04-closures.md) <!-- nav -->

# Lesson 05.4 — Closures

> **Phase 05 — Rust intermediate** · Lesson 4 of 14 · ⏱ about 4 hours (2 sittings)

## What you will learn

- What a closure is, how to write one, and how it differs from a normal function.
- How a closure *captures* variables from the place where it was created: by reference, by mutable reference, or by move.
- The three closure traits `Fn`, `FnMut` and `FnOnce`, and which one the compiler picks.
- How to pass closures into functions and return them from functions.
- Where you already use closures in the standard library (`sort_by_key`, `retain`, `map`, `unwrap_or_else`, …).

## Before you start

- You finished [Lesson 05.3 — Lifetimes](03-lifetimes.md).
- You know traits and trait bounds ([Lesson 05.2](02-traits.md)) and generics ([Lesson 05.1](01-generics.md)). Closures are used through trait bounds.
- You remember ownership: move, borrow, mutable borrow ([Lessons 03.2 to 03.5](../03-ownership-and-borrowing/02-ownership-rules.md)).
- You have the package `code/p05-lessons` from Lesson 05.1.

## New words

| Word | Meaning |
|---|---|
| **closure** | An anonymous (nameless) function, written `\|x\| x + 1`, that can use variables from the place where it was created. |
| **capture** | When a closure uses a variable defined outside it, it *captures* that variable: it borrows it or takes ownership of it. |
| **environment** | The variables around the closure that it may capture. |
| **`Fn`** | The trait of closures that only *read* what they captured. They can be called many times, even at the same time. |
| **`FnMut`** | The trait of closures that *change* what they captured. They can be called many times, one after another. |
| **`FnOnce`** | The trait of closures that may *consume* (move out) what they captured. They can be called only once. |
| **`move` closure** | A closure written `move \|…\| …` that takes ownership of everything it captures. |
| **callback** | A function (or closure) that you give to other code so that it calls you back at the right moment. |
| **function pointer** | A value of type `fn(i32) -> i32` that points to an ordinary function (or to a closure that captures nothing). |
| **memoization** | Saving the result of a calculation so that the next call with the same input is instant. |

## 1. What a closure is

A closure is a small function without a name that you can store in a variable or pass to other code.
Its parameters are written between two vertical bars `|…|`, followed by the body:

```rust
fn add_one_fn(x: i32) -> i32 {
    x + 1
}

fn main() {
    let add_one_closure = |x: i32| -> i32 { x + 1 };
    let add_one_short = |x| x + 1;
    let add = |a, b| a + b;
    let hello = || println!("hello from a closure");

    println!("{}", add_one_fn(5));
    println!("{}", add_one_closure(5));
    println!("{}", add_one_short(5));
    println!("{}", add(2, 3));
    hello();

    let multiline = |x: i32| {
        let doubled = x * 2;
        doubled + 1
    };
    println!("{}", multiline(10));
}
```

**What happens:**

1. `add_one_fn` is a normal function, for comparison.
2. `|x: i32| -> i32 { x + 1 }` is the closure in its longest form: parameter with type, return type, a
   block body.
3. `|x| x + 1` is the short form. Types are optional because the compiler works them out from how the
   closure is used. The body of a one-line closure needs no braces.
4. `|| println!(…)` has no parameters, so the bars are empty.
5. A closure with several lines needs braces `{ … }`; the last expression (without `;`) is the result.
6. You call a closure just like a function: `add(2, 3)`.

```text
6
6
6
5
hello from a closure
21
```

So far a closure looks like a shorter function. The real difference is in the next section.

### A closure has one fixed type

Parameter types are inferred **once**, from the first use. A closure cannot be generic:

```rust
fn main() {
    let identity = |x| x;
    let s = identity(String::from("text"));
    let n = identity(5);
    println!("{s} {n}");
}
```

```text
error[E0308]: mismatched types
 --> examples/infer_err.rs:4:22
  |
4 |     let n = identity(5);
  |             -------- ^ expected `String`, found integer
  |             |
  |             arguments to this function are incorrect
  |
note: expected because the closure was earlier called with an argument of type `String`
```

**What it means:** the first call made `x` a `String`; the second call with a number contradicts it.
If you need one operation for many types, write a generic function (Lesson 05.1).

## 2. Capturing the environment

A normal function can use only its parameters. A closure can also use the variables that exist around
it. This is called **capturing**. The compiler chooses the weakest kind of capture that makes the body
work.

### Capture by shared reference

```rust
fn main() {
    let name = String::from("Dilnoza");
    let hello = || println!("hello, {name}");
    hello();
    hello();
    println!("name is still mine: {name}");
}
```

**What happens:** the closure only *reads* `name`, so it borrows it with `&name`. We can call it many
times, and we can still use `name` outside, because reading by many readers is allowed (the borrowing
rules from Lesson 03.5).

```text
hello, Dilnoza
hello, Dilnoza
name is still mine: Dilnoza
```

### Capture by mutable reference

```rust
fn main() {
    let mut count = 0;
    let mut increment = || {
        count += 1;
        println!("count is now {count}");
    };
    increment();
    increment();
    println!("final count: {count}");
}
```

**What happens:**

1. The closure *changes* `count`, so it borrows it with `&mut count`.
2. Because it changes captured state, the closure variable itself must be declared `mut`:
   `let mut increment`. (Calling it changes its inner state, like calling a method with `&mut self`.)
3. The mutable borrow lasts as long as the closure is used. After the last call, `count` is free again.

```text
count is now 1
count is now 2
final count: 2
```

If you use `count` while the closure still holds the mutable borrow, the borrow rules apply:

```rust
fn main() {
    let mut count = 0;
    let mut increment = || count += 1;
    println!("before: {count}");
    increment();
}
```

```text
error[E0502]: cannot borrow `count` as immutable because it is also borrowed as mutable
 --> examples/capture_mut_err.rs:4:24
  |
3 |     let mut increment = || count += 1;
  |                         -- ----- first borrow occurs due to use of `count` in closure
  |                         |
  |                         mutable borrow occurs here
4 |     println!("before: {count}");
  |                        ^^^^^ immutable borrow occurs here
5 |     increment();
  |     --------- mutable borrow later used here
```

**What it means:** the closure holds `&mut count` from line 3 until its last call on line 5. Reading
`count` in between would break the "one writer, or many readers" rule. Fix: finish using the closure
first, or print after the last call.

### Capture by move

Add the keyword `move` before the bars and the closure **takes ownership** of every variable it uses:

```rust
fn main() {
    let data = vec![1, 2, 3];
    let owns_data = move || println!("data inside the closure: {:?}", data);
    owns_data();
    owns_data();
}
```

```text
data inside the closure: [1, 2, 3]
data inside the closure: [1, 2, 3]
```

After the move, the outside variable is gone (Lesson 03.3):

```rust
fn main() {
    let data = vec![1, 2, 3];
    let owns_data = move || println!("{:?}", data);
    owns_data();
    println!("{:?}", data);
}
```

```text
error[E0382]: borrow of moved value: `data`
 --> examples/capture_move_err.rs:5:22
  |
2 |     let data = vec![1, 2, 3];
  |         ---- move occurs because `data` has type `Vec<i32>`, which does not implement the `Copy` trait
3 |     let owns_data = move || println!("{:?}", data);
  |                     -------                  ---- variable moved due to use in closure
  |                     |
  |                     value moved into closure here
4 |     owns_data();
5 |     println!("{:?}", data);
  |                      ^^^^ value borrowed here after move
```

**What it means:** `data` was moved into the closure on line 3. The compiler even suggests cloning
(`data.clone()`) before the move, if you really need both. For types that implement `Copy`
(numbers, `bool`, `char`), `move` copies the value and the original stays usable.

> 💡 **Tip:** You need `move` mainly when the closure must live **longer** than the function that
> created it. Two typical cases: returning a closure (Section 4) and giving a closure to a new
> thread (Lesson 05.10).

## 3. The three closure traits: `Fn`, `FnMut`, `FnOnce`

How does a function say "give me a closure"? Through a trait bound. Every closure automatically
implements one, two or all three of these traits, depending on **what its body does with the
captured values** (not on whether you wrote `move`):

| Trait | The body… | Can be called | Picture |
|---|---|---|---|
| `FnOnce` | may *move a captured value out* (consume it) | once | a one-way ticket |
| `FnMut` | *changes* a captured value, but does not move it out | many times, one by one | a notebook you write in |
| `Fn` | only *reads* captured values (or captures nothing) | many times, even shared | a reference book |

The traits form a chain: every `Fn` closure is also `FnMut`, and every `FnMut` is also `FnOnce`.
So a function that asks for `FnOnce` accepts every closure; one that asks for `Fn` accepts the fewest.

```rust
fn call_twice<F: Fn()>(f: F) {
    f();
    f();
}

fn call_three_times<F: FnMut()>(mut f: F) {
    f();
    f();
    f();
}

fn call_once<F: FnOnce() -> String>(f: F) -> String {
    f()
}

fn main() {
    let name = String::from("Rust");
    call_twice(|| println!("Fn: reads {name}"));

    let mut total = 0;
    call_three_times(|| total += 10);
    println!("FnMut: total = {total}");

    let owned = String::from("moved out");
    let result = call_once(move || owned);
    println!("FnOnce: got {result}");
}
```

**What happens:**

1. `F: Fn()` means "a closure that takes nothing, returns nothing, and only reads". The first closure
   reads `name`, so it qualifies.
2. `F: FnMut()` allows a closure that changes its environment. The parameter is `mut f` because
   calling an `FnMut` needs a mutable binding.
3. `F: FnOnce() -> String` takes a closure returning a `String`. The body `owned` *moves* the string
   out as the result, so the closure can be called only once.
4. The syntax `Fn(A, B) -> R` in a bound lists the argument types and the return type.

```text
Fn: reads Rust
Fn: reads Rust
FnMut: total = 30
FnOnce: got moved out
```

The compiler checks that the closure fits the trait that you asked for:

```rust
fn call_twice<F: Fn()>(f: F) {
    f();
    f();
}

fn main() {
    let s = String::from("hello");
    call_twice(move || drop(s));
}
```

```text
error[E0507]: cannot move out of `s`, a captured variable in an `Fn` closure
 --> examples/fnonce_err.rs:8:29
  |
7 |     let s = String::from("hello");
  |         -   --------------------- move occurs because `s` has type `String`, which does not implement the `Copy` trait
  |         |
  |         captured outer variable
8 |     call_twice(move || drop(s));
  |                -------      ^ `s` is moved here
  |                |
  |                captured by this `Fn` closure
  |
help: `Fn` and `FnMut` closures require captured values to be able to be consumed multiple times, but `FnOnce` closures may consume them only once
```

**What it means:** `drop(s)` consumes `s`, so the closure is `FnOnce`, but `call_twice` calls it twice.
You can see the same rule when you call a consuming closure yourself:

```rust
fn main() {
    let s = String::from("hello");
    let consume = move || s;
    let a = consume();
    let b = consume();
    println!("{a} {b}");
}
```

```text
error[E0382]: use of moved value: `consume`
 --> examples/fnonce_twice_err.rs:5:13
  |
4 |     let a = consume();
  |             --------- `consume` moved due to this call
5 |     let b = consume();
  |             ^^^^^^^ value used here after move
  |
note: closure cannot be invoked more than once because it moves the variable `s` out of its environment
```

> 📏 **Convention:** When you write a function that takes a closure, ask for the **weakest** trait that
> works: `FnOnce` if you call it once, `FnMut` if you call it several times and it may change state,
> `Fn` only if it must be callable concurrently or you need to be sure it does not change anything.
> The weaker your demand, the more closures callers can pass.

## 4. Closures as arguments and as return values

You have seen closures as arguments. To **return** a closure, use `impl Fn…` (Lesson 05.2) and
`move`, so the closure owns what it needs:

```rust
fn make_adder(n: i32) -> impl Fn(i32) -> i32 {
    move |x| x + n
}

fn make_counter() -> impl FnMut() -> u32 {
    let mut count = 0;
    move || {
        count += 1;
        count
    }
}

fn compose<A, B, C>(f: impl Fn(A) -> B, g: impl Fn(B) -> C) -> impl Fn(A) -> C {
    move |x| g(f(x))
}

fn main() {
    let add5 = make_adder(5);
    println!("{} {}", add5(1), add5(10));

    let mut counter = make_counter();
    println!("{} {} {}", counter(), counter(), counter());

    let double_then_text = compose(|x: i32| x * 2, |y: i32| format!("<{y}>"));
    println!("{}", double_then_text(21));
}
```

**What happens:**

1. `make_adder(5)` creates a closure that remembers `n = 5`. Each call to `make_adder` makes a new
   closure with its own `n`.
2. `make_counter` returns an `FnMut`. The closure owns its own `count` (moved in with `move`), which
   survives between calls even though `make_counter` has finished long ago.
3. `compose` takes two closures and returns a new one that applies `f` and then `g`. This is how
   small pieces are combined into bigger ones.
4. The `let mut counter` is needed because calling an `FnMut` changes it.

```text
6 15
1 2 3
<42>
```

Without `move`, the closure would borrow `n`, but `n` dies when `make_adder` returns:

```rust
fn make_adder(n: i32) -> impl Fn(i32) -> i32 {
    |x| x + n
}
```

```text
error[E0373]: closure may outlive the current function, but it borrows `n`, which is owned by the current function
 --> examples/return_closure_err.rs:2:5
  |
2 |     |x| x + n
  |     ^^^     - `n` is borrowed here
  |     |
  |     may outlive borrowed value `n`
  |
note: closure is returned here
help: to force the closure to take ownership of `n` (and any other referenced variables), use the `move` keyword
  |
2 |     move |x| x + n
  |     ++++
```

**What it means:** this is the dangling-reference problem from Lesson 05.3 in a new place. The fix is
exactly what the compiler suggests: add `move`.

### A closure that is passed around and called later: `retry`

A useful systems example: retry an operation that may fail, such as a network call. The operation is a
closure, called several times, and it is allowed to change state (so `FnMut`):

```rust
fn retry<F>(mut operation: F, attempts: u32) -> Result<u32, String>
where
    F: FnMut() -> Result<u32, String>,
{
    let mut last_error = String::from("no attempts made");
    for attempt in 1..=attempts {
        match operation() {
            Ok(value) => return Ok(value),
            Err(e) => {
                println!("attempt {attempt} failed: {e}");
                last_error = e;
            }
        }
    }
    Err(last_error)
}

fn main() {
    let mut calls = 0;
    let result = retry(
        || {
            calls += 1;
            if calls < 3 {
                Err(String::from("connection refused"))
            } else {
                Ok(calls)
            }
        },
        5,
    );
    println!("{:?} after {} calls", result, calls);

    let hopeless = retry(|| Err(String::from("always fails")), 2);
    println!("{:?}", hopeless);
}
```

**What happens:**

1. `1..=attempts` is a range from 1 to `attempts` including the end (Lesson 02.5).
2. `retry` calls `operation()`. On success it returns at once; on failure it prints the error, keeps it
   and tries again.
3. The first closure fails twice, then succeeds on call 3. It changes the captured `calls`, so it is `FnMut`.
4. After `retry` has finished, `calls` is readable again, because the closure was moved into `retry`
   and is gone.
5. The second closure never succeeds, so `retry` gives up and returns the last error.

```text
attempt 1 failed: connection refused
attempt 2 failed: connection refused
Ok(3) after 3 calls
attempt 1 failed: always fails
attempt 2 failed: always fails
Err("always fails")
```

(Real retry code also waits between attempts and gives up for errors that retrying cannot fix. Phase 18
returns to this idea under the name *backoff*.)

## 5. Closures in the standard library

You have been using closures-shaped APIs for a while. Here is a tour of the common ones:

```rust
use std::collections::HashMap;

fn main() {
    let mut words = vec!["pear", "fig", "banana", "kiwi"];
    words.sort_by_key(|w| w.len());
    println!("by length: {:?}", words);
    words.sort_by(|a, b| b.cmp(a));
    println!("reverse alphabetical: {:?}", words);

    let mut numbers = vec![1, 2, 3, 4, 5, 6];
    numbers.retain(|n| n % 2 == 0);
    println!("only even: {:?}", numbers);

    let maybe: Option<i32> = Some(4);
    println!("{:?}", maybe.map(|n| n * 10));
    let nothing: Option<i32> = None;
    println!("{}", nothing.unwrap_or_else(|| -1));
    println!("{}", maybe.map_or(0, |n| n + 1));
    println!("{:?}", maybe.filter(|n| *n > 10));

    let parsed: Result<i32, String> = "x1".parse::<i32>().map_err(|e| format!("bad number: {e}"));
    println!("{:?}", parsed);

    let mut groups: HashMap<usize, Vec<&str>> = HashMap::new();
    for w in ["a", "bb", "cc", "d"] {
        groups.entry(w.len()).or_insert_with(Vec::new).push(w);
    }
    println!("{:?}", groups.get(&2));

    println!("{:?}", (5 > 3).then(|| "yes"));
}
```

**What happens:**

1. `sort_by_key(|w| w.len())` — the closure computes the sorting key for each element. Words with equal keys keep their relative order ("pear" before "kiwi", as in the input).
2. `sort_by(|a, b| b.cmp(a))` — the closure compares two elements and returns an `Ordering`; swapping `a`
   and `b` reverses the order.
3. `retain(|n| n % 2 == 0)` — keep only the elements for which the closure returns `true`.
4. `Option::map` transforms the value inside `Some`; `None` stays `None`. `map_or(default, f)` gives
   the default for `None`. `filter` turns `Some(x)` into `None` if the closure says `false`.
5. `unwrap_or_else(|| -1)` — the closure runs **only if** the value is `None`. Compare
   `unwrap_or(expensive())`, where the argument is always computed. Use `_else` when the fallback costs something.
6. `map_err` transforms the error inside a `Result` (the same idea as `From` in Lesson 05.2).
7. `entry(key).or_insert_with(Vec::new)` — a function name can be used where a closure is expected.
   The new vector is created only when the key is missing.
8. `bool::then(|| value)` returns `Some(value)` when the bool is true, else `None`.

```text
by length: ["fig", "pear", "kiwi", "banana"]
reverse alphabetical: ["pear", "kiwi", "fig", "banana"]
only even: [2, 4, 6]
Some(40)
-1
5
None
Err("bad number: invalid digit found in string")
Some(["bb", "cc"])
Some("yes")
```

> 💡 **Tip:** The pattern "pass a closure instead of a value" is for **lazy** evaluation (do the work
> only if needed) and for **customizing behaviour** (sort order, filter rule). Next lesson combines it
> with iterators, where closures are everywhere.

## 6. Closures inside structs, and function pointers

A struct can store a closure. Its type is unknown to you, so the struct is generic over it:

```rust
use std::collections::HashMap;

struct Memo<F: Fn(u64) -> u64> {
    func: F,
    cache: HashMap<u64, u64>,
    calls: u32,
}

impl<F: Fn(u64) -> u64> Memo<F> {
    fn new(func: F) -> Self {
        Memo { func, cache: HashMap::new(), calls: 0 }
    }

    fn get(&mut self, arg: u64) -> u64 {
        if let Some(&value) = self.cache.get(&arg) {
            return value;
        }
        self.calls += 1;
        let value = (self.func)(arg);
        self.cache.insert(arg, value);
        value
    }
}

fn main() {
    let mut square = Memo::new(|x| {
        println!("  computing {x} * {x}");
        x * x
    });
    println!("{}", square.get(12));
    println!("{}", square.get(12));
    println!("{}", square.get(3));
    println!("closure ran {} times", square.calls);
}
```

**What happens:**

1. `Memo<F: Fn(u64) -> u64>` stores a closure `func` and a `HashMap` of earlier answers. This is
   **memoization**: a slow calculation runs once per input.
2. `(self.func)(arg)` calls the closure stored in a field. The parentheses are required, otherwise Rust
   thinks you call a *method* named `func`.
3. The second `get(12)` finds the answer in `cache`, so "computing 12 * 12" is printed only once.

```text
  computing 12 * 12
144
144
  computing 3 * 3
9
closure ran 2 times
```

**Function pointers.** An ordinary function can be passed as a value too. Its type is written
`fn(i32) -> i32` (lowercase `fn`; not to be confused with the trait `Fn`):

```rust
fn double(x: i32) -> i32 {
    x * 2
}

fn apply(f: fn(i32) -> i32, x: i32) -> i32 {
    f(x)
}

fn apply_generic<F: Fn(i32) -> i32>(f: F, x: i32) -> i32 {
    f(x)
}

fn main() {
    println!("{}", apply(double, 21));
    println!("{}", apply(|x| x + 1, 41)); // a closure without captures becomes a fn pointer
    let offset = 100;
    println!("{}", apply_generic(|x| x + offset, 1));
    println!("{}", apply_generic(double, 4));
}
```

```text
42
42
101
8
```

**What happens:** `apply` accepts only function pointers. A closure that captures nothing can be
turned into one, but a closure that captures (`offset`) cannot; for that you need the generic `Fn`
form. Because function pointers also implement `Fn`, `apply_generic` accepts both. **Prefer the
generic `Fn`-trait bounds** in your own code; use `fn(...)` pointers when talking to C code (Lesson 06.10).

> ⚡ **Performance:** A generic `F: Fn…` is monomorphized (Lesson 05.1): each closure gets its own
> specialised copy of the function and the call can even be inlined, so closures are as fast as writing the
> code by hand. This is also why you cannot put two *different* closures into one `Vec` directly:
> they are different types. A `Vec` of mixed closures needs trait objects (`Box<dyn Fn…>`),
> explained in Lesson 06.1.

## Common mistakes

**1. Using the variable while a closure holds a mutable borrow** (`E0502`, Section 2). Finish with
the closure first, or restructure.

**2. Using a variable after `move`** (`E0382`, Section 2). Clone before the move, or do not move.

**3. A closure that consumes its capture but is called twice** (`E0507` / `E0382`, Section 3). Ask
for `FnOnce` in your function, or change the closure so it does not move the value out (clone it, or
only read it).

**4. Returning a closure without `move`** (`E0373`, Section 4). Add `move`.

**5. Calling the closure with different types** (`E0308`, Section 1). A closure has one signature; use
a generic function for many types.

**6. Forgetting `mut` on the closure variable** for a closure that changes its state:

```rust
fn main() {
    let mut count = 0;
    let increment = || count += 1;
    increment();
}
```

```text
error[E0596]: cannot borrow `increment` as mutable, as it is not declared as mutable
```

Declare it as `let mut increment`.

## Try it yourself

This time you practise **staging only part of your changes**, which keeps commits focused.

1. Start a branch and create a small example:

   ```bash
   cd ~/projects/rust
   git switch main
   git switch -c lesson/05-04-closures
   ```

   Create `code/p05-lessons/examples/closures_demo.rs`:

   ```rust
   fn main() {
       let price = 10;
       let total = price * 3;
       println!("total: {total}");
       println!("done");
   }
   ```

   Commit it: `git add .` and `git commit -m "feat(p05): closures demo"`.

2. Now make **two unrelated changes** in the same file: change `price * 3` to `price * 4`, and add a
   new function at the end:

   ```rust
   fn debug_dump(value: i32) {
       println!("debug: {value}");
   }
   ```

3. You want to commit the first change alone, because the debug helper is not ready. Use
   **interactive staging**: `git add -p` (p for *patch*). Git shows each changed piece (a *hunk*) and
   asks what to do. Both changes are close together, so Git shows one hunk; press `s` to *split* it:

   ```bash
   git add -p
   ```

   ```text
   (1/1) Stage this hunk [y,n,q,a,d,s,e,p,P,?]? s
   Split into 2 hunks.
   @@ -1,6 +1,6 @@
    fn main() {
        let price = 10;
   -    let total = price * 3;
   +    let total = price * 4;
        println!("total: {total}");
        println!("done");
    }
   (1/2) Stage this hunk [y,n,q,a,d,k,K,j,J,g,/,e,p,P,?]? y
   @@ -4,3 +4,7 @@
   ...
   (2/2) Stage this hunk [y,n,q,a,d,K,J,g,/,e,p,P,?]? n
   ```

   The letters mean: `y` stage this hunk, `n` do not stage it, `s` split into smaller hunks, `q`
   quit, `?` help. Now check what is staged and what is not:

   ```bash
   git status --short
   git diff --staged --stat
   ```

   ```text
   MM code/p05-lessons/examples/closures_demo.rs
    code/p05-lessons/examples/closures_demo.rs | 2 +-
    1 file changed, 1 insertion(+), 1 deletion(-)
   ```

   `MM` means: the file has changes in the staging area *and* more changes that are not staged. Commit
   only the staged part:

   ```bash
   git commit -m "fix(p05): charge four units instead of three"
   git add .
   git commit -m "feat(p05): add debug_dump helper"
   ```

   You now have two small commits with honest messages instead of one commit named "stuff".

4. Finish the lesson as usual: add `retry` and `make_counter` examples, commit each, merge into `main`.

> 📏 **Convention:** One commit = one idea. Commits that mix a bug fix, a rename and a new feature
> are hard to review and hard to undo. `git add -p` is the tool that lets you separate them even after
> you have already edited everything.

## Exercises

### Exercise 1 — A multiplier factory (easy)

Write `fn make_multiplier(factor: i32) -> impl Fn(i32) -> i32` so that `make_multiplier(3)(4)` is 12.
Write tests: one for the result and one showing that two multipliers do not affect each other.

<details>
<summary>Hint</summary>

The returned closure must own `factor`, so write `move |x| x * factor`.

</details>

<details>
<summary>Answer</summary>

```rust
fn make_multiplier(factor: i32) -> impl Fn(i32) -> i32 {
    move |x| x * factor
}

fn main() {
    let triple = make_multiplier(3);
    println!("{} {}", triple(4), triple(-2));
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn multiplies_by_the_captured_factor() {
        let triple = make_multiplier(3);
        assert_eq!(triple(4), 12);
        assert_eq!(triple(-2), -6);
    }

    #[test]
    fn two_multipliers_do_not_interfere() {
        let by_two = make_multiplier(2);
        let by_ten = make_multiplier(10);
        assert_eq!(by_two(5), 10);
        assert_eq!(by_ten(5), 50);
    }
}
```

```text
12 -6
```

Why it works: each call to `make_multiplier` creates a new closure that owns its own copy of
`factor`. It only reads it, so the closure is `Fn`.

</details>

### Exercise 2 — Sort with a custom rule (easy)

Write `fn sort_words(words: &mut [&str])` that sorts by length (shorter first), and words of equal length
alphabetically. Write a test with `["pear", "fig", "apple", "kiwi", "date"]`.

<details>
<summary>Hint</summary>

`words.sort_by(|a, b| …)`. Compare lengths with `a.len().cmp(&b.len())`. An `Ordering` has a method
`.then(other)` that uses `other` only if the first comparison says "equal": `.then(a.cmp(b))`.

</details>

<details>
<summary>Answer</summary>

```rust
fn sort_words(words: &mut [&str]) {
    words.sort_by(|a, b| a.len().cmp(&b.len()).then(a.cmp(b)));
}

fn main() {
    let mut words = vec!["pear", "fig", "apple", "kiwi", "date"];
    sort_words(&mut words);
    println!("{:?}", words);
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn shorter_first_then_alphabetical() {
        let mut words = vec!["pear", "fig", "apple", "kiwi", "date"];
        sort_words(&mut words);
        assert_eq!(words, vec!["fig", "date", "kiwi", "pear", "apple"]);
    }
}
```

```text
["fig", "date", "kiwi", "pear", "apple"]
```

Why it works: the three four-letter words are ordered alphabetically ("date", "kiwi", "pear") because the
length comparison is "equal" and `.then` falls through to the text comparison.

</details>

### Exercise 3 — A counter with its own state (medium)

Write `fn make_counter() -> impl FnMut() -> u32` that returns 1 on the first call, 2 on the second, and so
on. Write tests: counting up, and two counters that do not share state.

<details>
<summary>Hint</summary>

Declare `let mut count = 0;` inside `make_counter`, then return `move || { count += 1; count }`. In
tests, the closure variable must be `let mut c = make_counter();`.

</details>

<details>
<summary>Answer</summary>

```rust
fn make_counter() -> impl FnMut() -> u32 {
    let mut count = 0;
    move || {
        count += 1;
        count
    }
}

fn main() {
    let mut a = make_counter();
    let mut b = make_counter();
    println!("{} {} {}", a(), a(), b());
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn counts_up_from_one() {
        let mut c = make_counter();
        assert_eq!(c(), 1);
        assert_eq!(c(), 2);
        assert_eq!(c(), 3);
    }

    #[test]
    fn each_counter_has_its_own_state() {
        let mut a = make_counter();
        let mut b = make_counter();
        a();
        a();
        assert_eq!(b(), 1);
    }
}
```

```text
1 2 1
```

Why it works: `count` is moved into the closure, so every closure created by `make_counter` carries a private
`count`. Changing it makes the closure `FnMut`.

</details>

### Exercise 4 — Write and test `retry` (hard)

Write `fn retry<F>(operation: F, attempts: u32) -> Result<u32, String>` for an `FnMut() -> Result<u32, String>`. It
calls the operation up to `attempts` times, returns the first `Ok`, and otherwise returns the **last** error.
For `attempts == 0` it must not call the operation and must return an `Err`. Write three tests: success on the third
try (check the number of calls), giving up with the last error, and zero attempts.

<details>
<summary>Hint</summary>

Keep `let mut last_error = String::from("no attempts made");` and loop `for _ in 0..attempts`. Inside,
`match operation()` with `Ok(v) => return Ok(v)` and `Err(e) => last_error = e`. Declare the parameter
as `mut operation`. In tests the closure changes an outer `calls` counter; read `calls` after `retry` returns.

</details>

<details>
<summary>Answer</summary>

```rust
fn retry<F>(mut operation: F, attempts: u32) -> Result<u32, String>
where
    F: FnMut() -> Result<u32, String>,
{
    let mut last_error = String::from("no attempts made");
    for _ in 0..attempts {
        match operation() {
            Ok(value) => return Ok(value),
            Err(e) => last_error = e,
        }
    }
    Err(last_error)
}

fn main() {
    let mut calls = 0;
    let result = retry(
        || {
            calls += 1;
            if calls < 3 {
                Err(String::from("not yet"))
            } else {
                Ok(calls)
            }
        },
        5,
    );
    println!("{:?}", result);
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn succeeds_on_the_third_try() {
        let mut calls = 0;
        let result = retry(
            || {
                calls += 1;
                if calls < 3 { Err(String::from("no")) } else { Ok(calls) }
            },
            5,
        );
        assert_eq!(result, Ok(3));
        assert_eq!(calls, 3);
    }

    #[test]
    fn gives_up_and_returns_the_last_error() {
        let mut calls = 0;
        let result = retry(
            || {
                calls += 1;
                Err(format!("error #{calls}"))
            },
            2,
        );
        assert_eq!(result, Err(String::from("error #2")));
    }

    #[test]
    fn zero_attempts_never_calls_the_operation() {
        let mut calls = 0;
        let result = retry(
            || {
                calls += 1;
                Ok(1)
            },
            0,
        );
        assert!(result.is_err());
        assert_eq!(calls, 0);
    }
}
```

```text
Ok(3)
```

Why it works: `retry` takes the closure by value (moved in), so after the call `calls` is readable
again. The tests check behaviour (return values and number of calls) rather than internals.

</details>

## Quiz

1. What is the difference between a closure and a function?

<details>
<summary>Answer</summary>

A closure is anonymous, usually has inferred types, and can use ("capture") variables from the surrounding code. A function can use only its own parameters and global items.

</details>

2. When is the keyword `move` required?

<details>
<summary>Answer</summary>

When the closure must own its captured values because it outlives the place where they live, for example when you return a closure from a function or send it to another thread. `move` makes the closure take ownership of everything it uses.

</details>

3. Which of `Fn`, `FnMut`, `FnOnce` does a closure implement if its body moves a captured `String` out?

<details>
<summary>Answer</summary>

Only `FnOnce`. Moving the value out consumes it, so the closure can run just once.

</details>

4. You write a function that calls its closure argument several times and the closure may change a counter. Which bound do you choose?

<details>
<summary>Answer</summary>

`FnMut`. (`Fn` would reject closures that change state; `FnOnce` would allow you to call only once.)

</details>

5. Why does `unwrap_or_else(|| compute())` sometimes beat `unwrap_or(compute())`?

<details>
<summary>Answer</summary>

`unwrap_or(compute())` always runs `compute()` before the call, even if the value is `Some`. With `unwrap_or_else` the closure runs only when the value is `None`, so the work is done only if needed.

</details>

6. Why can't a `Vec` hold two different closures even if both have the signature `Fn(i32) -> i32`?

<details>
<summary>Answer</summary>

Every closure has its own unique anonymous type. A `Vec<T>` holds values of one type. To mix closures you need trait objects such as `Box<dyn Fn(i32) -> i32>` (Lesson 06.1).

</details>

## Summary

- A closure is an anonymous function written `|args| body`. Types are usually inferred, and fixed after the first use.
- Closures capture their environment by shared reference, by mutable reference, or by move (`move`); the compiler picks the weakest kind that works.
- `Fn` reads, `FnMut` changes, `FnOnce` consumes. Ask for the weakest trait that your function needs.
- To return a closure use `impl Fn…` and `move`.
- The standard library takes closures everywhere: `sort_by_key`, `retain`, `map`, `unwrap_or_else`, `or_insert_with`.
- Store a closure in a struct with a generic parameter `F: Fn…`; prefer trait bounds over `fn` pointers.
- Git habit: `git add -p` stages part of a file, so each commit holds exactly one idea.

## Sources and further reading

- [The Rust Programming Language — 13.1 Closures](https://doc.rust-lang.org/book/ch13-01-closures.html) — the official chapter, with capturing and the `Fn` traits.
- [Rust by Example — Closures](https://doc.rust-lang.org/rust-by-example/fn/closures.html) — short examples of capturing, input parameters and output.
- [std::ops::FnMut](https://doc.rust-lang.org/std/ops/trait.FnMut.html) — the trait documentation with the relations between the three traits.
- [Git documentation — git add (interactive mode and --patch)](https://git-scm.com/docs/git-add) — all the answers for `git add -p`.

---

[🏠 Start](../START-HERE.md) · [← 05.3 Lifetimes](03-lifetimes.md) · [📋 Phase 05](README.md) · [05.5 Iterators →](05-iterators.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
