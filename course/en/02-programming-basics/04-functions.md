<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 02.3 Operators and expressions](03-operators-and-expressions.md) · [📋 Phase 02](README.md) · [02.5 Control flow: conditions and loops →](05-control-flow.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/02-programming-basics/04-functions.md) <!-- nav -->

# Lesson 02.4 — Functions

> **Phase 02 — Programming basics in Rust** · Lesson 4 of 11 · ⏱ about 2.5 hours (2 sittings)

## What you will learn

- Why we split programs into functions, and how to write and call one.
- Parameters and return values, and why every parameter needs a type.
- How the last expression of a function becomes its result, and when to use `return`.
- How to return several values with a tuple.
- What recursion is, and what happens when it never stops.
- How to keep functions small, with good names, and how to check them with assertions.

## Before you start

- You finished [Lesson 02.3 — Operators and expressions](03-operators-and-expressions.md). In particular, remember: a block's value is its last expression **without** a semicolon.
- Put the examples into `src/bin/` of `~/projects/rust/code/p02-basics`.

## New words

| Word | Meaning |
|---|---|
| **function** | A named block of code that does one job. You can run it many times by calling it. |
| **call** | Running a function: `add(3, 4)`. |
| **parameter** | A variable in the function's definition that receives a value: the `a` in `fn add(a: i32)`. |
| **argument** | The actual value you pass in a call: the `3` in `add(3, 4)`. |
| **return value** | The value that a function gives back to the caller. |
| **return type** | The type of the return value, written after `->`. |
| **signature** | The first line of a function: name, parameters and return type. |
| **tail expression** | The last expression of a function body, written without `;`. It is the return value. |
| **recursion** | A function that calls itself. |
| **base case** | The simple case in which a recursive function stops calling itself. |
| **call stack** | The area of memory where the computer keeps the data of the functions that are running right now. |
| **stack frame** | The piece of the call stack that belongs to one function call. |
| **stack overflow** | The crash that happens when the call stack becomes full, usually because of endless recursion. |
| **DRY** (Don't Repeat Yourself) | A principle: do not copy the same code in several places; put it into one function. |
| **KISS** (Keep It Simple, Stupid) | A principle: prefer the simplest solution that works. |
| **predicate** | A function that answers a yes/no question and returns `bool`, for example `is_even`. |

## 1. Why functions?

Imagine you need to convert minutes to seconds in ten places. You could write `minutes * 60` ten times. But what if the rule changes? You would search for ten places. A **function** gives the calculation a name and one home:

```text
          +----------------------+
 inputs   |                      |  output
 -------> |   function "area"    | -------> 
 (3, 4)   |   width * height     |   12
          +----------------------+
```

Functions give you three things:

1. **Reuse** — write once, call many times (the **DRY** principle: Don't Repeat Yourself).
2. **Names** — `is_leap_year(2024)` explains itself better than a long boolean expression.
3. **Small pieces** — you can test and understand a small function without reading the whole program.

You have already used one function all the time: `fn main()`, the starting point of every program.

## 2. Defining and calling a function

```rust
fn main() {
    say_hello();
    let total = add(3, 4);
    println!("3 + 4 = {total}");
    println!("10 + 20 = {}", add(10, 20));
    print_square(7);
}

fn say_hello() {
    println!("Hello from a function!");
}

fn add(a: i32, b: i32) -> i32 {
    a + b
}

fn print_square(n: i32) {
    println!("{n} squared is {}", n * n);
}
```

What happens:

1. The program starts in `main`. `main` has no parameters and returns nothing.
2. `say_hello();` **calls** the function `say_hello`. The computer jumps into its body, runs the `println!`, and comes back.
3. `fn add(a: i32, b: i32) -> i32` is a **signature**. The keyword `fn` starts a function. Then come the name `add`, the **parameters** in parentheses (`a` and `b`, both `i32`), an arrow `->`, and the **return type** `i32`.
4. `add(3, 4)` is a call with two **arguments**: `3` goes into `a`, `4` goes into `b`. The body `a + b` has no semicolon, so it is the **tail expression** and becomes the return value `7`. It is stored in `total`.
5. A call is an expression, so it can be used anywhere a value fits: `add(10, 20)` inside `println!`.
6. `print_square` has no `->`: it returns nothing (more exactly, it returns `()`, the unit value from Lesson 02.2).
7. The functions are defined **after** `main`. The order does not matter in Rust: a function can be used before the line where it is written.

```text
Hello from a function!
3 + 4 = 7
10 + 20 = 30
7 squared is 49
```

The rules of the signature:

- **Every parameter must have a type.** Rust never guesses types of parameters. (Types of local variables inside the function are still inferred.)
- The return type is written after `->`. If there is no `->`, the function returns `()`.
- Parameters are matched by **position**: the first argument goes to the first parameter.

> 📏 **Convention:** Function names use `snake_case` and start with a verb or describe the result: `calculate_tax`, `is_valid`, `to_celsius`. A function that returns `bool` often starts with `is_`, `has_` or `can_`. Such a function is called a **predicate**.

## 3. Returning values

The value of the function is the value of its body block, i.e. the **tail expression**. This is exactly the rule from Lesson 02.3. You may also use the keyword `return` to leave the function early:

```rust
fn double(x: i32) -> i32 {
    x * 2
}

fn double_explicit(x: i32) -> i32 {
    return x * 2;
}

fn clamp_to_100(x: i32) -> i32 {
    if x > 100 {
        return 100;
    }
    x
}

fn main() {
    println!("{}", double(21));
    println!("{}", double_explicit(21));
    println!("{}", clamp_to_100(250));
    println!("{}", clamp_to_100(42));
}
```

What happens:

1. `double` uses the tail expression: the usual Rust style.
2. `double_explicit` uses `return x * 2;` with a semicolon. It does the same.
3. `clamp_to_100` has an **early return**: `if x > 100 { return 100; }` means "only when `x` is bigger than 100, leave the function at once with `100`". (`if` is explained in the next lesson. Here, just read it as "when".) If the `if` does not fire, the last line `x` is the result.

```text
42
42
100
42
```

> 📏 **Convention:** Use the tail expression for the normal result. Use `return` for early exits, for example to handle an unusual case at the top of the function.

### Returning several values: a tuple

A function returns one value. To return several, return a tuple (Lesson 02.2) and destructure it:

```rust
fn divide_with_remainder(a: i32, b: i32) -> (i32, i32) {
    (a / b, a % b)
}

fn main() {
    let (quotient, remainder) = divide_with_remainder(17, 5);
    println!("17 = 5 * {quotient} + {remainder}");
    let pair = divide_with_remainder(20, 6);
    println!("{:?}", pair);
}
```

```text
17 = 5 * 3 + 2
(3, 2)
```

The return type `(i32, i32)` is a tuple of two numbers, and `(a / b, a % b)` builds that tuple.

## 4. Functions call functions

Functions are building blocks. Small functions combine into bigger ones:

```rust
fn area(width: u32, height: u32) -> u32 {
    width * height
}

fn is_big_room(width: u32, height: u32) -> bool {
    area(width, height) > 20
}

fn main() {
    println!("area of 4x6 = {}", area(4, 6));
    println!("4x6 is big? {}", is_big_room(4, 6));
    println!("3x5 is big? {}", is_big_room(3, 5));
}
```

```text
area of 4x6 = 24
4x6 is big? true
3x5 is big? false
```

`is_big_room` does not repeat the multiplication; it asks `area`. If the formula for the area ever changes, only one function changes.

> 📏 **Convention:** A good function does **one job** and has a name that says which. If you need the word "and" to describe it ("reads the file and counts the words and prints"), split it. If a function does not fit on one screen (about 20–30 lines), it is probably doing too much. This is the **KISS** principle: Keep It Simple, Stupid.

## 5. Parameters are copies

When you call `bump(a)`, the function gets its own variable `x` holding the **value** of `a`. For the simple types of this phase (numbers, `bool`, `char`), the value is copied. Parameters are immutable by default, just like `let` variables:

```rust
fn bump(x: i32) -> i32 {
    x += 1;
    x
}

fn main() {
    println!("{}", bump(1));
}
```

```text
error[E0384]: cannot assign to immutable argument `x`
 --> src/main.rs:2:5
  |
2 |     x += 1;
  |     ^^^^^^ cannot assign to immutable argument
  |
help: consider making this binding mutable
  |
1 | fn bump(mut x: i32) -> i32 {
  |         +++
```

The fix is `mut x`. It makes the function's **own copy** changeable; the caller's variable stays untouched:

```rust
fn bump(mut x: i32) -> i32 {
    x += 1;
    x
}

fn main() {
    let a = 1;
    let b = bump(a);
    println!("a = {a}, b = {b}");
}
```

```text
a = 1, b = 2
```

`a` is still `1`. (What happens with values that are not simple copies, such as text, is the main topic of Phase 03: ownership and borrowing. For now your functions will take numbers, `bool` and `char`, and text in the form `&str`.)

Variables created inside a function exist only inside it (the scope rule from Lesson 02.1):

```rust
fn compute() -> i32 {
    let secret = 42;
    secret
}

fn main() {
    let result = compute();
    println!("{result}");
    println!("{secret}");
}
```

```text
error[E0425]: cannot find value `secret` in this scope
 --> src/main.rs:9:16
  |
9 |     println!("{secret}");
  |                ^^^^^^ not found in this scope
```

The only way out of a function is its return value.

## 6. Recursion and the call stack

A function may call **itself**. This is **recursion**. It works when you can describe a problem through a smaller copy of the same problem. The factorial `n!` is `n * (n-1)!`, and `1!` is `1`:

```rust
fn factorial(n: u64) -> u64 {
    if n <= 1 {
        1
    } else {
        n * factorial(n - 1)
    }
}

fn main() {
    println!("5! = {}", factorial(5));
    println!("10! = {}", factorial(10));
    println!("20! = {}", factorial(20));
}
```

What happens:

1. `if condition { A } else { B }` chooses between two values: `A` when the condition is true, `B` otherwise. (Lesson 02.5 explains `if` fully. Note that `if` is an expression here, and its value is the function's tail expression.)
2. `factorial(5)` needs `factorial(4)`, which needs `factorial(3)`, and so on, until `factorial(1)`. That is the **base case**: it returns `1` without calling itself. Every recursion needs a base case.
3. Then the answers come back: `1`, then `2 * 1`, `3 * 2`, `4 * 6`, `5 * 24 = 120`.

```text
5! = 120
10! = 3628800
20! = 2432902008176640000
```

Each call needs memory for its parameters and local variables. This memory is a **stack frame** on the **call stack**. Each new call puts a frame on top, and each return removes the top frame:

```text
 factorial(1)   <- top: runs now, returns 1
 factorial(2)   waiting for factorial(1)
 factorial(3)   waiting for factorial(2)
 factorial(4)   waiting for factorial(3)
 factorial(5)   waiting for factorial(4)
 main           waiting for factorial(5)
```

The stack is limited (a few megabytes). A recursion without a base case fills it:

```rust
fn forever(n: u64) -> u64 {
    forever(n + 1) + 1
}

fn main() {
    println!("{}", forever(0));
}
```

```text
warning: function cannot return without recursing
 --> src/main.rs:1:1
  |
1 | fn forever(n: u64) -> u64 {
  | ^^^^^^^^^^^^^^^^^^^^^^^^^ cannot return without recursing

thread 'main' (3970470) has overflowed its stack
fatal runtime error: stack overflow, aborting
```

The compiler even warns you. The program is killed by the operating system. This is a **stack overflow**. (Phase 03, Lesson 1 explains the stack and the heap in detail.) When a problem can be solved with a loop (next lesson), a loop is often the better tool; recursion is best when the problem is naturally nested.

## Common mistakes

**1. A semicolon on the tail expression.**

```rust
fn add_one(x: i32) -> i32 {
    x + 1;
}
```

```text
error[E0308]: mismatched types
 --> src/main.rs:1:23
  |
1 | fn add_one(x: i32) -> i32 {
  |    -------            ^^^ expected `i32`, found `()`
  |    |
  |    implicitly returns `()` as its body has no tail or `return` expression
2 |     x + 1;
  |          - help: remove this semicolon to return this value
```

The signature promises an `i32`, but the semicolon makes the body return `()`. Remove the `;`.

**2. A parameter without a type.** `fn add(a, b) -> i32` gives `error: expected one of ':', '@', or '|', found ','` and a hint `give it a type`. Write `fn add(a: i32, b: i32)`.

**3. The wrong number of arguments.**

```text
error[E0061]: this function takes 2 arguments but 1 argument was supplied
 --> src/main.rs:6:20
  |
6 |     println!("{}", add(1));
  |                    ^^^--- argument #2 of type `i32` is missing
```

**4. The wrong type of argument.**

```text
error[E0308]: mismatched types
 --> src/main.rs:6:27
  |
6 |     println!("{}", add(1, 2.5));
  |                    ---    ^^^ expected `i32`, found floating-point number
  |                    |
  |                    arguments to this function are incorrect
```

Convert with `as`, or change the function.

**5. Doing too much in one function.** Split it. Name the pieces.

**6. Recursion with no base case.** It ends with a stack overflow.

> 🧪 **Testing:** Functions are easy to test because they have clear inputs and outputs. Call the function with a few inputs, including edge cases, and compare with the answer you expect:
>
> ```rust
> fn is_even(n: i32) -> bool {
>     n % 2 == 0
> }
>
> fn main() {
>     assert_eq!(is_even(4), true);
>     assert_eq!(is_even(7), false);
>     assert_eq!(is_even(0), true);
>     assert_eq!(is_even(-2), true);
>     println!("is_even: all checks passed");
> }
> ```
>
> ```text
> is_even: all checks passed
> ```
>
> Good test inputs: a normal case, zero, a negative number, the largest value. In Lesson 02.8 you will move such checks into real tests that `cargo test` runs for you.

> 💡 **Tip:** RustRover can **extract a function** for you: select some code, then choose *Refactor → Extract Method* (the entry may be called *Extract Function*; the macOS shortcut is `⌥⌘M`). It creates the function with the right parameters. Refactoring means improving the structure of code without changing what it does.

## Try it yourself

1. Create `src/bin/04_functions.rs` with the first program of section 2. Run `cargo run --bin 04_functions`.
2. Add a function `fn cube(n: i32) -> i32` and print `cube(3)`; it must print `27`.
3. Add a function `fn average(a: f64, b: f64) -> f64` and print `average(3.0, 4.0)`; it must print `3.5`.
4. Write a function `is_adult(age: u32) -> bool` that is true for 18 and over (use `age >= 18`). Add four `assert_eq!` checks: 17, 18, 0 and 99.
5. Break something on purpose: add a `;` after the tail expression of `cube`, or call `cube()` with no argument. Find the error code and the `help` text each time.
6. Run `cargo fmt`, `cargo clippy`. Commit:

```bash
cd ~/projects/rust
git add code/p02-basics
git commit -m "feat(p02): add function examples"
```

## Exercises

### Exercise 1 — square (easy)

Write `fn square(n: i32) -> i32`. Check it for `0`, `5` and `-4` with `assert_eq!`, and print `square: all checks passed`.

<details>
<summary>Hint</summary>

The body is one expression: `n * n`, without a semicolon.

</details>

<details>
<summary>Answer</summary>

```rust
fn square(n: i32) -> i32 {
    n * n
}

fn main() {
    assert_eq!(square(0), 0);
    assert_eq!(square(5), 25);
    assert_eq!(square(-4), 16);
    println!("square: all checks passed");
}
```

Why it works: a negative number squared is positive, so `square(-4)` is `16`. The output is `square: all checks passed`.

</details>

### Exercise 2 — Fix the signature (easy)

This program has three mistakes. Find and fix them with the compiler's help. The program must print `7`.

```rust
fn add(a, b: i32) -> i32 {
    a + b;
}

fn main() {
    println!("{}", add(3, 4.0));
}
```

<details>
<summary>Hint</summary>

Read the errors one by one: a parameter without a type, a semicolon at the end of the body, and an argument of the wrong type.

</details>

<details>
<summary>Answer</summary>

```rust
fn add(a: i32, b: i32) -> i32 {
    a + b
}

fn main() {
    println!("{}", add(3, 4));
}
```

Why it works: every parameter has a type; the tail expression has no `;`, so the body returns an `i32`; and `4` is an integer, not a float. Output: `7`.

</details>

### Exercise 3 — Temperature functions (medium)

Write `celsius_to_fahrenheit(c: f64) -> f64` and `fahrenheit_to_celsius(f: f64) -> f64`. Check that 100 °C is 212 °F, that 32 °F is 0 °C, and that converting 36.6 °C to Fahrenheit and back gives 36.6 again (compare with a tolerance of `1e-9`).

<details>
<summary>Hint</summary>

`F = C * 9 / 5 + 32` and `C = (F - 32) * 5 / 9`. Use float literals (`9.0`, not `9`).

</details>

<details>
<summary>Answer</summary>

```rust
fn celsius_to_fahrenheit(c: f64) -> f64 {
    c * 9.0 / 5.0 + 32.0
}

fn fahrenheit_to_celsius(f: f64) -> f64 {
    (f - 32.0) * 5.0 / 9.0
}

fn main() {
    assert_eq!(celsius_to_fahrenheit(100.0), 212.0);
    assert_eq!(fahrenheit_to_celsius(32.0), 0.0);
    let round_trip = fahrenheit_to_celsius(celsius_to_fahrenheit(36.6));
    assert!((round_trip - 36.6).abs() < 1e-9);
    println!("round trip of 36.6 C = {round_trip}");
    println!("temperature: all checks passed");
}
```

```text
round trip of 36.6 C = 36.6
temperature: all checks passed
```

Why it works: a function call is an expression, so one call can be the argument of another. The exactly representable values (100, 212, 32, 0) can be compared with `assert_eq!`; the round trip can have a tiny error, so it uses a tolerance.

</details>

### Exercise 4 — Two results (medium)

Write `fn min_max(a: i32, b: i32) -> (i32, i32)` that returns the smaller number first and the bigger number second. Use the methods `a.min(b)` and `a.max(b)`. Destructure the result of `min_max(9, 4)` and print it. Add three `assert_eq!` checks, one of them with equal numbers.

<details>
<summary>Hint</summary>

The tail expression is the tuple `(a.min(b), a.max(b))`. A tuple can be compared with `assert_eq!`: `assert_eq!(min_max(9, 4), (4, 9));`.

</details>

<details>
<summary>Answer</summary>

```rust
fn min_max(a: i32, b: i32) -> (i32, i32) {
    (a.min(b), a.max(b))
}

fn main() {
    let (small, big) = min_max(9, 4);
    println!("small = {small}, big = {big}");
    assert_eq!(min_max(9, 4), (4, 9));
    assert_eq!(min_max(-1, 3), (-1, 3));
    assert_eq!(min_max(5, 5), (5, 5));
    println!("min_max: all checks passed");
}
```

```text
small = 4, big = 9
min_max: all checks passed
```

Why it works: the function returns one value, a tuple, which holds both results.

</details>

### Exercise 5 — Recursion (hard)

Write two recursive functions: `digit_sum(n: u32) -> u32` (the sum of the digits of `n`) and `gcd(a: u32, b: u32) -> u32` (the greatest common divisor, by Euclid's algorithm: `gcd(a, 0)` is `a`, otherwise `gcd(b, a % b)`). For each, name the base case. Add assertions: `digit_sum(4721) == 14`, `digit_sum(1000) == 1`, `gcd(12, 18) == 6`, `gcd(17, 5) == 1`, `gcd(0, 9) == 9`.

<details>
<summary>Hint</summary>

For `digit_sum`: if `n < 10` the answer is `n` itself (base case); otherwise it is the last digit `n % 10` plus `digit_sum(n / 10)`. Use `if … { … } else { … }` as the tail expression.

</details>

<details>
<summary>Answer</summary>

```rust
fn digit_sum(n: u32) -> u32 {
    if n < 10 {
        n
    } else {
        n % 10 + digit_sum(n / 10)
    }
}

fn gcd(a: u32, b: u32) -> u32 {
    if b == 0 {
        a
    } else {
        gcd(b, a % b)
    }
}

fn main() {
    assert_eq!(digit_sum(7), 7);
    assert_eq!(digit_sum(4721), 14);
    assert_eq!(digit_sum(1000), 1);
    assert_eq!(gcd(12, 18), 6);
    assert_eq!(gcd(17, 5), 1);
    assert_eq!(gcd(0, 9), 9);
    println!("digit_sum and gcd: all checks passed");
}
```

Why it works: the base cases are `n < 10` and `b == 0`. In every other case the argument gets smaller (`n / 10` is smaller than `n`; the pair `(b, a % b)` has a smaller second number), so the recursion ends. The output is `digit_sum and gcd: all checks passed`.

</details>

## Quiz

1. What is the difference between a parameter and an argument?

<details>
<summary>Answer</summary>

A parameter is the variable in the function definition (`a` in `fn add(a: i32)`). An argument is the actual value you pass when you call it (`3` in `add(3)`).

</details>

2. Which part of the signature says what the function returns?

<details>
<summary>Answer</summary>

The return type after the arrow `->`. Without it, the function returns `()`.

</details>

3. What is wrong with `fn f(x: i32) -> i32 { x * 2; }`?

<details>
<summary>Answer</summary>

The semicolon discards the value, so the body returns `()`, not `i32`. Error `E0308`. Remove the `;`.

</details>

4. Do you need to define a function before you call it?

<details>
<summary>Answer</summary>

No. In Rust, the order of function definitions in a file does not matter.

</details>

5. How can a function return two values?

<details>
<summary>Answer</summary>

Return a tuple, for example `-> (i32, i32)`, and destructure it in the caller.

</details>

6. What is a base case, and what happens without one?

<details>
<summary>Answer</summary>

The simple case in which a recursive function returns without calling itself. Without it, the function calls itself forever, the call stack fills up, and the program crashes with a stack overflow.

</details>

7. Why is `fn is_even(n: i32) -> bool` a good name and signature?

<details>
<summary>Answer</summary>

The name starts with `is_` and asks a yes/no question; the return type `bool` matches. The function does one job and is easy to test.

</details>

## Summary

- A function has a name, typed parameters and an optional return type: `fn name(a: T) -> R { … }`.
- Every parameter needs a type; the return type is written after `->`.
- The tail expression (the last expression without `;`) is the result. `return` is for early exits.
- A function can return several values as a tuple.
- Parameters are immutable copies; add `mut` to change the function's own copy.
- Variables inside a function are invisible outside it.
- Recursion needs a base case, or the stack overflows.
- Keep functions small, with one job and a clear name (DRY, KISS). Check them with `assert_eq!`.

## Sources and further reading

- [The Rust Programming Language — 3.3 Functions](https://doc.rust-lang.org/book/ch03-03-how-functions-work.html) — the official book chapter on this topic.
- [Rust by Example — Functions](https://doc.rust-lang.org/rust-by-example/fn.html) — runnable examples of functions.
- [The Rust Reference — Functions](https://doc.rust-lang.org/reference/items/functions.html) — the exact syntax and rules.
- [Rust API Guidelines — Naming](https://rust-lang.github.io/api-guidelines/naming.html) — the official rules for function names.
- [Error index — E0308](https://doc.rust-lang.org/error_codes/E0308.html) — the "mismatched types" error, with examples.

---

[🏠 Start](../START-HERE.md) · [← 02.3 Operators and expressions](03-operators-and-expressions.md) · [📋 Phase 02](README.md) · [02.5 Control flow: conditions and loops →](05-control-flow.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
