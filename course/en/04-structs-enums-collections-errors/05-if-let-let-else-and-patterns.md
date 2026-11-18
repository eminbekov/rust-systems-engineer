<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 04.4 Option and match](04-option-and-match.md) · [📋 Phase 04](README.md) · [04.6 Modules, crates and packages →](06-modules-crates-and-packages.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/04-structs-enums-collections-errors/05-if-let-let-else-and-patterns.md) <!-- nav -->

# Lesson 04.5 — if let, let else and patterns

> **Phase 04 — Structs, enums, collections and errors** · Lesson 5 of 12 · ⏱ about 3.5 hours (2 sittings)

## What you will learn

- `if let`: the short form of a `match` that cares about one case.
- `while let`: a loop that runs while a pattern keeps matching.
- `let else`: take a value apart, or leave early when it does not fit.
- The `matches!` macro, and "let chains" (several conditions in one `if let`).
- Patterns everywhere: in `let`, in function parameters, in `for` loops; refutable and irrefutable patterns; slice patterns.

## Before you start

- You finished [Lesson 04.4 — Option and match](04-option-and-match.md). You know `Option`, `match` and exhaustiveness.
- You know what a `Result` looks like (`Ok(value)` / `Err(error)`); the full story is in [Lesson 04.10 — Error handling](10-error-handling.md).

## New words

| Word | Meaning |
|---|---|
| **`if let`** | `if let PATTERN = VALUE { ... }`: run the block only if the value fits the pattern. |
| **`while let`** | `while let PATTERN = VALUE { ... }`: repeat the block as long as the value fits the pattern. |
| **`let else`** | `let PATTERN = VALUE else { ... };`: bind the variables if the pattern fits, otherwise run the `else` block, which must leave the current code path. |
| **refutable pattern** | A pattern that can fail to match, such as `Some(x)` (the value may be `None`). |
| **irrefutable pattern** | A pattern that always matches, such as `x`, `(a, b)` or `Point { x, y }`. |
| **diverge** | To never continue normally: `return`, `break`, `continue` and `panic!` all diverge. |
| **let chain** | Several conditions joined with `&&` in one `if` or `while`, where some of them may be `let` patterns. |
| **destructuring** | Taking a compound value (tuple, struct, enum, array) apart into its pieces with a pattern. |
| **slice pattern** | A pattern that matches the elements of an array or slice, such as `[first, .., last]`. |
| **`Vec`** (vector) | A growable list of values of one type. It has its own lesson, [Lesson 04.7 — Vectors](07-vectors.md); here we only use `push` and `pop`. |
| **stack** (the data structure) | A pile where you add and remove only at the top: the last item added is the first one removed. (Not the same as the memory stack of Lesson 03.1.) |
| **macro** | Code that writes code. You call it with `!`, such as `println!` or `matches!`. |
| **lint** | A check that finds suspicious or unidiomatic code and warns about it. `cargo clippy` runs many lints. |

## 1. `if let`: a `match` with one interesting arm

Sometimes you care about one case only. A full `match` forces you to write the boring arm too:

```rust
fn main() {
    let config_port: Option<u16> = Some(8080);

    // with match: the second arm is only there to satisfy the compiler
    match config_port {
        Some(port) => println!("match: port {port}"),
        _ => (),
    }

    // with if let: shorter
    if let Some(port) = config_port {
        println!("if let: port {port}");
    }

    // with else
    let nothing: Option<u16> = None;
    if let Some(port) = nothing {
        println!("port {port}");
    } else {
        println!("no port configured, using the default");
    }
}
```

**What happens:**

1. In the `match`, the arm `_ => ()` means "otherwise do nothing". `()` is the "unit" value: no value at all.
2. `if let Some(port) = config_port { ... }` reads: "if `config_port` fits the pattern `Some(port)`, bind `port` and run the block". Note the single `=`: it is not a comparison, it is pattern matching. The pattern is on the left, the value on the right.
3. If the pattern does not fit, the block is skipped. An optional `else` block runs instead.
4. The variable `port` exists only inside the block of the `if let`.

```text
match: port 8080
if let: port 8080
no port configured, using the default
```

`if let` also works with your own enums and with `else if let` chains:

```rust
#[derive(Debug)]
enum Command {
    Quit,
    Move { x: i32, y: i32 },
    Say(String),
}

fn main() {
    let commands = [
        Command::Say(String::from("hi")),
        Command::Move { x: 4, y: 9 },
        Command::Quit,
    ];

    for command in &commands {
        if let Command::Say(text) = command {
            println!("someone said {text:?}");
        } else if let Command::Move { x, .. } = command {
            println!("moved horizontally to {x}");
        } else {
            println!("something else: {:?}", command);
        }
    }
}
```

**What happens:**

1. `command` is a `&Command` (we loop over `&commands`), so the pattern variables `text` and `x` are references. No data is moved out of the array.
2. `Command::Move { x, .. }` takes only `x` and ignores the other fields with `..`.
3. The chain `if let ... else if let ... else` tests the cases one after another. The loop prints one line per command.

```text
someone said "hi"
moved horizontally to 4
something else: Quit
```

> 📏 **Convention:** use `if let` when you handle **one** case and ignore the rest. Use `match` when you handle **two or more** cases, or when you want the compiler to check that you covered everything (an `if let` is never checked for completeness). `cargo clippy` knows this rule. Run on the first example it says:
>
> ```text
> warning: you seem to be trying to use `match` for destructuring a single pattern. Consider using `if let`
>  --> src/main.rs:3:5
>   |
> 3 | /     match port {
> 4 | |         Some(p) => println!("port {p}"),
> 5 | |         _ => (),
> 6 | |     }
>   | |_____^ help: try: `if let Some(p) = port { println!("port {p}") }`
>   |
>   = note: `#[warn(clippy::single_match)]` on by default
> ```

## 2. `while let`: loop while the pattern fits

`while let` repeats as long as the pattern matches. The classic use is emptying a **stack**. Here we use a `Vec` — a growable list that [Lesson 04.7](07-vectors.md) explains. All you need now: `vec![1, 2, 3]` makes a list, and `.pop()` removes the last item and returns it as an `Option` (`None` when the list is empty).

```rust
fn main() {
    let mut stack = vec![1, 2, 3];
    while let Some(top) = stack.pop() {
        println!("popped {top}");
    }
    println!("stack is empty now: {:?}", stack);

    let mut letters = "abc".chars();
    while let Some(c) = letters.next() {
        println!("letter {c}");
    }
}
```

**What happens:**

1. `stack.pop()` returns `Some(3)`, then `Some(2)`, then `Some(1)`, then `None`.
2. While it gives `Some(top)`, the pattern fits and the body runs. When it gives `None`, the pattern fails and the loop ends. No manual "is it empty?" check is needed.
3. The second loop does the same with `next()` on the characters of a text: `next()` returns `Some(char)` until the text is finished.

```text
popped 3
popped 2
popped 1
stack is empty now: []
letter a
letter b
letter c
```

> 💡 **Tip:** for walking over a list you normally write `for item in &list`. `while let` is for the cases where the loop body also changes the thing it takes from (like popping from a stack), or where the number of steps is not known in advance.

## 3. `let else`: get it, or leave

Very often you want: "take the value out of this `Option`/`Result`, and if it is not there, stop here". With `match` or `if let` this leads to nesting: the happy path ends up indented deeper and deeper. **`let else`** keeps the happy path flat.

```rust
fn describe(text: &str) -> String {
    let Ok(port) = text.parse::<u16>() else {
        return format!("{text:?} is not a port number");
    };
    format!("port {port} is {}", if port < 1024 { "privileged" } else { "free to use" })
}

fn main() {
    println!("{}", describe("80"));
    println!("{}", describe("8080"));
    println!("{}", describe("http"));
    println!("{}", describe("70000"));

    for line in ["10", "x", "20"] {
        let Ok(n) = line.parse::<i32>() else {
            println!("skipping {line:?}");
            continue;
        };
        println!("got {n}");
    }
}
```

**What happens:**

1. `text.parse::<u16>()` tries to read a number from `0` to `65535` (a valid port number). It returns a `Result`.
2. `let Ok(port) = ... else { ... };` means: "if the result fits `Ok(port)`, create the variable `port` and go on. Otherwise run the `else` block."
3. The `else` block **must leave** the normal flow of the code: `return`, `continue`, `break`, or `panic!`. Here the first function returns an error text; the loop at the end `continue`s with the next line.
4. After the `let else` statement, `port` (or `n`) is a normal variable in the rest of the function. No extra indentation.
5. `{text:?}` prints the text with quotes, so empty or space-filled text is visible. `"70000"` is a number but too big for `u16`, so it fails as well.

```text
port 80 is privileged
port 8080 is free to use
"http" is not a port number
"70000" is not a port number
got 10
skipping "x"
got 20
```

If the `else` block can fall through, the compiler refuses:

```rust
let Ok(n) = text.parse::<i32>() else {
    println!("not a number");
};
```

```text
error[E0308]: `else` clause of `let...else` does not diverge
 --> src/main.rs:3:42
  |
3 |       let Ok(n) = text.parse::<i32>() else {
  |  __________________________________________^
4 | |         println!("not a number");
5 | |     };
  | |_____^ expected `!`, found `()`
  |
  = note:   expected type `!`
          found unit type `()`
  = help: try adding a diverging expression, such as `return` or `panic!(..)`
  = help: ...or use `match` instead of `let...else`
```

If the block did not leave, `n` would have no value after it. The message calls leaving "diverging"; the type `!` (read "never") is the type of an expression that never finishes normally, such as `return` or `panic!()`.

> 📏 **Convention:** "early return" style: check the unusual cases first and leave, so that the main logic is not buried in nested `if`s. `let else` is the tool for that.

> 💡 **Tip:** if your `else` block does nothing but `return None` (for an `Option`) or `return Err(e)`, there is a shorter way, the `?` operator. `cargo clippy` will suggest it (`clippy::question_mark`). You will learn `?` in [Lesson 04.10](10-error-handling.md). Use `let else` when the `else` block does real work: builds a message, logs, or continues a loop.

## 4. `matches!` and let chains

### `matches!`

When you only need a yes/no answer — "does this value fit this pattern?" — use the macro `matches!`:

```rust
fn is_vowel(c: char) -> bool {
    matches!(c, 'a' | 'e' | 'i' | 'o' | 'u')
}

fn main() {
    println!("{} {}", is_vowel('e'), is_vowel('z'));

    let score = Some(7);
    println!("{}", matches!(score, Some(n) if n > 5));
    println!("{}", matches!(score, None));

    let word = "education";
    let vowels = word.chars().filter(|c| is_vowel(*c)).count();
    println!("{word} has {vowels} vowels");
}
```

**What happens:**

1. `matches!(value, pattern)` gives `true` if the value fits the pattern, else `false`. It is a short `match` with `true`/`false` arms.
2. The pattern may use `|` (or) and a guard (`if n > 5`), exactly as in a `match` arm.
3. `word.chars().filter(...).count()` counts the characters for which the closure gives `true` (iterators: [Lesson 05.5](../05-rust-intermediate/05-iterators.md)).

```text
true false
true
false
education has 5 vowels
```

### Let chains

Since Rust 1.88 (edition 2024, the edition of this course) an `if` or `while` can join several conditions and `let` patterns with `&&`. Without it you would need nested `if let`s:

```rust
fn main() {
    let width: Option<u32> = Some(12);
    let height: Option<u32> = Some(5);

    if let Some(w) = width
        && let Some(h) = height
        && w > h
    {
        println!("wide: {w} x {h}");
    }

    let age: Option<u32> = Some(17);
    if let Some(a) = age && a >= 18 {
        println!("adult");
    } else {
        println!("not an adult, or unknown");
    }
}
```

**What happens:**

1. The first `if` has three parts joined by `&&`: two patterns and a normal condition. They are checked left to right; if one fails, the rest is skipped. The variables `w` and `h` are available in the later parts and in the block.
2. The second `if` combines a pattern with a condition. If `age` is `None`, or less than 18, the `else` block runs.

```text
wide: 12 x 5
not an adult, or unknown
```

## 5. Patterns are everywhere

You have used patterns in `match`. In fact, patterns appear in many places in Rust. A plain `let` is already a pattern!

```rust
struct Point {
    x: i32,
    y: i32,
}

fn sum_pair(&(a, b): &(i32, i32)) -> i32 {
    a + b
}

fn main() {
    // tuples
    let (name, age) = ("Zed", 41);
    println!("{name} is {age}");

    // nested
    let ((a, b), c) = ((1, 2), 3);
    println!("{}", a + b + c);

    // struct with a literal inside
    let p = Point { x: 0, y: 5 };
    match p {
        Point { x: 0, y } => println!("on the y axis at {y}"),
        Point { x, y: 0 } => println!("on the x axis at {x}"),
        Point { x, y } => println!("at ({x}, {y})"),
    }

    // function parameters
    println!("{}", sum_pair(&(3, 4)));

    // for loops
    for (i, ch) in "hey".chars().enumerate() {
        println!("{i}: {ch}");
    }
    for &(from, to) in &[(1, 2), (2, 3)] {
        println!("{from} -> {to}");
    }

    // ignoring parts
    let (first, _, third) = (10, 20, 30);
    println!("{first} {third}");
    let (head, ..) = (1, 2, 3, 4);
    println!("{head}");
}
```

**What happens:**

1. `let (name, age) = ("Zed", 41);` **destructures** a tuple into two variables.
2. Patterns can be nested to any depth: `((a, b), c)` takes apart a tuple that contains a tuple.
3. In a struct pattern you can put a literal for a field (`x: 0`) to match only certain values, and a name (`y`) to capture the rest. As in `match`, arms are tried in order.
4. A function parameter can be a pattern: `&(a, b): &(i32, i32)` takes a reference to a pair and destructures it right in the parameter list. The `&` in a pattern means "the value is a reference; look at what it points to".
5. In a `for` loop, `(i, ch)` takes each pair apart; `.enumerate()` hands out `(index, item)` pairs. In `for &(from, to) in &[...]` the `&` pattern copies the numbers out of the reference. (This `&n` form is what you saw in `for &n in numbers` in Lesson 04.4.)
6. `_` ignores one value; `..` ignores "all the rest" (in a tuple, the rest of the positions; in a struct, the rest of the fields).

```text
Zed is 41
6
on the y axis at 5
7
0: h
1: e
2: y
1 -> 2
2 -> 3
10 30
1
```

### Refutable and irrefutable patterns

Some patterns always fit (**irrefutable**): `x`, `(a, b)`, `Point { x, y }`. Others may fail (**refutable**): `Some(x)` fails on `None`, `Ok(n)` fails on `Err`.

- `let` (plain), function parameters and `for` loops need **irrefutable** patterns: what would they do if it did not fit?
- `if let`, `while let`, `let else` and `match` arms accept **refutable** patterns: they have a plan for the mismatch.

```rust
let maybe: Option<i32> = Some(3);
let Some(x) = maybe;
```

```text
error[E0005]: refutable pattern in local binding
 --> src/main.rs:3:9
  |
3 |     let Some(x) = maybe;
  |         ^^^^^^^ pattern `None` not covered
  |
  = note: `let` bindings require an "irrefutable pattern", like a `struct` or an `enum` with only one variant
  = note: for more information, visit https://doc.rust-lang.org/book/ch19-02-refutability.html
  = note: the matched value is of type `Option<i32>`
help: you might want to use `let...else` to handle the variant that isn't matched
  |
3 |     let Some(x) = maybe else { todo!() };
  |                         ++++++++++++++++
```

The compiler even suggests the cure: `let else`.

### Slice patterns

A pattern can also take apart an array or a slice ([Lesson 03.6 — Slices and string types](../03-ownership-and-borrowing/06-slices-and-string-types.md)):

```rust
fn describe(slice: &[i32]) -> String {
    match slice {
        [] => String::from("empty"),
        [x] => format!("one element: {x}"),
        [first, second] => format!("two elements: {first} and {second}"),
        [first, .., last] => format!("starts with {first}, ends with {last}"),
    }
}

fn sum(slice: &[i32]) -> i32 {
    match slice {
        [] => 0,
        [first, rest @ ..] => first + sum(rest),
    }
}

fn main() {
    println!("{}", describe(&[]));
    println!("{}", describe(&[7]));
    println!("{}", describe(&[7, 8]));
    println!("{}", describe(&[7, 8, 9, 10]));
    println!("{}", sum(&[1, 2, 3, 4]));
}
```

**What happens:**

1. `[]` matches an empty slice; `[x]` exactly one element; `[first, second]` exactly two.
2. `[first, .., last]` matches any slice with at least two elements and picks the first and the last. The `..` stands for "any number of elements in between".
3. In `sum`, `[first, rest @ ..]` splits the slice into the first element and *the rest as a smaller slice*. The function calls itself with `rest` — this is **recursion** (a function that calls itself). It stops at the empty slice, which has sum 0.
4. The compiler checks that the four arms of `describe` cover every length, so the `match` is exhaustive.

```text
empty
one element: 7
two elements: 7 and 8
starts with 7, ends with 10
10
```

### `_` and `_name` are not the same

An underscore alone ignores a value and **does not bind** it. A name starting with an underscore (`_b`) **is** a real variable (the underscore only silences the "unused variable" warning). The difference matters for ownership:

```rust
fn main() {
    let a = String::from("kept");
    let _ = a; // `_` does not bind anything, so nothing moves
    println!("{a} is still here");

    let b = String::from("moved");
    let _b = b; // `_b` is a real variable, so b is moved into it
    println!("{_b}");
}
```

```text
kept is still here
moved
```

If you print `b` after `let _b = b;`, you get the usual `error[E0382]: borrow of moved value: `b``.

## 6. Which tool to choose?

| You want to … | Use |
|---|---|
| handle every variant or case, and have the compiler check completeness | `match` |
| act on **one** pattern and ignore the rest | `if let` (with `else` for the opposite case) |
| repeat while a pattern keeps matching (`pop`, `next`) | `while let` |
| take a value out, or leave the function/loop if it is not there | `let else` |
| get a yes/no answer about a pattern | `matches!` |
| combine a pattern with extra conditions | let chain (`if let ... && cond`) |

## 7. Testing

> 🧪 **Testing:** code that uses `let else` has two exits — the normal one and the early one — and **each needs a test**. `matches!` is handy in assertions when you care about the *kind* of value, not every detail: `assert!(matches!(value, Some(Shape::Circle(_))))`.

```rust
#[derive(Debug)]
enum Shape {
    Circle(f64),
    Square(f64),
}

fn describe(text: &str) -> String {
    let Ok(port) = text.parse::<u16>() else {
        return format!("{text:?} is not a port number");
    };
    format!("port {port} is {}", if port < 1024 { "privileged" } else { "free to use" })
}

fn make_shape(kind: &str, size: f64) -> Option<Shape> {
    match kind {
        "circle" => Some(Shape::Circle(size)),
        "square" => Some(Shape::Square(size)),
        _ => None,
    }
}

fn main() {
    println!("{}", describe("80"));
    println!("{:?}", make_shape("circle", 2.0));
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn valid_ports() {
        assert_eq!(describe("80"), "port 80 is privileged");
        assert_eq!(describe("8080"), "port 8080 is free to use");
    }

    #[test]
    fn invalid_ports() {
        assert_eq!(describe("http"), "\"http\" is not a port number");
        assert_eq!(describe("70000"), "\"70000\" is not a port number");
        assert_eq!(describe(""), "\"\" is not a port number");
    }

    #[test]
    fn matches_checks_the_variant_only() {
        assert!(matches!(make_shape("circle", 2.0), Some(Shape::Circle(_))));
        assert!(matches!(make_shape("square", 1.0), Some(Shape::Square(s)) if s > 0.5));
        assert!(matches!(make_shape("hexagon", 1.0), None));
    }
}
```

**What happens:**

1. `valid_ports` takes the normal exit of `describe`; `invalid_ports` takes the early exit three times, including the empty text and the out-of-range number (edge cases).
2. `matches_checks_the_variant_only` checks which variant came back without comparing the payload — and the third assertion shows that `matches!(x, None)` works as well.
3. (Cargo prints harmless "field is never read" warnings for this demo; they are left out.)

```text
running 3 tests
...
test result: ok. 3 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

> 🔒 **Security:** `let else` makes validation of untrusted input easy to write and read: each check is one line, and everything after it can rely on the checked value. Prefer a list of such checks over one deeply nested block.

## Common mistakes

> ⚠️ **Common mistake:** writing `==` where you mean a pattern: `if Some(x) == value` does not bind `x`. Use `if let Some(x) = value`.

> ⚠️ **Common mistake:** a `let else` block that does not leave (`E0308`, section 3). End it with `return`, `continue`, `break` or `panic!`.

> ⚠️ **Common mistake:** a refutable pattern in a plain `let` (`E0005`, section 5). Use `let else`, `if let` or `match`.

> ⚠️ **Common mistake:** expecting the compiler to check an `if let ... else if let ...` chain for completeness. It does not; only `match` is checked. If you add a new enum variant later, such a chain silently ignores it.

> ⚠️ **Common mistake:** `let _x = value;` when you wanted to ignore a value without moving it. Use `let _ = value;` or simply do not bind it.

> ⚠️ **Common mistake:** an endless `while let` because the value is never updated, for example `while let Some(x) = list.first() { ... }` with a body that never removes the item. Make sure each iteration changes what the next check sees (like `pop`).

## Try it yourself

1. Create a project:

   ```text
   cd ~/projects/rust
   cargo new code/p04-patterns
   cd code/p04-patterns
   ```

2. Put this into `src/main.rs`:

   ```rust
   fn main() {
       let inputs = ["42", "abc", "7", "", "-3"];
       let mut total = 0;

       for text in inputs {
           let Ok(number) = text.parse::<i32>() else {
               println!("skipping {text:?}");
               continue;
           };
           total += number;
       }

       println!("total = {total}");
   }
   ```

3. Run `cargo run`. Expected output:

   ```text
   skipping "abc"
   skipping ""
   total = 46
   ```

4. Change the program so that numbers below zero are also skipped, using a let chain: `if let Ok(n) = text.parse::<i32>() && n >= 0 { total += n; } else { ... }`. Which version reads better, the `let else` or the chain? (Both are fine; the habit to build is to compare.)
5. Run `cargo clippy` and `cargo fmt`; fix everything they report. Then commit:

   ```text
   cd ~/projects/rust
   git add code/p04-patterns
   git commit -m "feat(p04): let else and patterns practice"
   ```

## Exercises

### Exercise 1 — Report only even numbers (easy)

Write `fn describe_even(value: Option<i32>) -> String` that returns `"N is even"` when the value is `Some(N)` with an even `N`, and `"nothing to report"` in every other case. Use `if let` with a let chain. Test even, odd and `None`.

<details>
<summary>Hint</summary>

`if let Some(n) = value && n % 2 == 0 { return format!(...); }`, then return the fallback text after the `if`.

</details>

<details>
<summary>Answer</summary>

```rust
fn describe_even(value: Option<i32>) -> String {
    if let Some(n) = value
        && n % 2 == 0
    {
        return format!("{n} is even");
    }
    String::from("nothing to report")
}

fn main() {
    println!("{}", describe_even(Some(4)));
    println!("{}", describe_even(Some(5)));
    println!("{}", describe_even(None));
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn even_number_is_reported() {
        assert_eq!(describe_even(Some(4)), "4 is even");
    }

    #[test]
    fn odd_or_missing_is_not() {
        assert_eq!(describe_even(Some(5)), "nothing to report");
        assert_eq!(describe_even(None), "nothing to report");
    }
}
```

Output of `cargo run`:

```text
4 is even
nothing to report
nothing to report
```

Why it works: the `if` body runs only when the pattern fits **and** the condition holds. In all other cases the function falls through to the last line. (Without let chains you would write a nested `if` inside the `if let`, and `cargo clippy` would suggest collapsing it.)

</details>

### Exercise 2 — Sum with slice patterns (medium)

Write `fn sum(slice: &[i32]) -> i32` using a `match` with the slice patterns `[]` and `[first, rest @ ..]` and recursion. Test a normal list, a list that adds to zero, and the empty list.

<details>
<summary>Hint</summary>

For `[]` the sum is `0`. For `[first, rest @ ..]` the sum is `first + sum(rest)`; `first` is a reference to a number, and adding a number to a reference of a number works.

</details>

<details>
<summary>Answer</summary>

```rust
fn sum(slice: &[i32]) -> i32 {
    match slice {
        [] => 0,
        [first, rest @ ..] => first + sum(rest),
    }
}

fn main() {
    println!("{}", sum(&[1, 2, 3, 4]));
    println!("{}", sum(&[]));
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn sums_numbers() {
        assert_eq!(sum(&[1, 2, 3, 4]), 10);
        assert_eq!(sum(&[-5, 5]), 0);
    }

    #[test]
    fn empty_sum_is_zero() {
        assert_eq!(sum(&[]), 0);
    }
}
```

Output of `cargo run`:

```text
10
0
```

Why it works: each call peels off the first element and handles a shorter slice, until the slice is empty. The two patterns together cover every possible slice, so the `match` is exhaustive. (For very long slices, a loop or `.iter().sum()` is better than recursion, because every call uses stack memory.)

</details>

### Exercise 3 — Parse `key=value` lines (medium)

Write `fn parse_kv(line: &str) -> Result<(String, i32), String>` for lines like `"retries = 3"`. Use `split_once('=')` and `let else` twice: when there is no `=` return `Err` with a message, and when the value is not a number return `Err` with another message. Test one good line and both kinds of bad lines.

`Result` is explained in Lesson 04.10; for now: `Ok(value)` means success, `Err(message)` means failure with a reason.

<details>
<summary>Hint</summary>

`let Some((key, value)) = line.split_once('=') else { return Err(...); };` Then `let Ok(number) = value.trim().parse::<i32>() else { return Err(...); };` Finally `Ok((key.trim().to_string(), number))`.

</details>

<details>
<summary>Answer</summary>

```rust
fn parse_kv(line: &str) -> Result<(String, i32), String> {
    let Some((key, value)) = line.split_once('=') else {
        return Err(format!("missing '=' in {line:?}"));
    };
    let Ok(number) = value.trim().parse::<i32>() else {
        return Err(format!("{:?} is not a number", value.trim()));
    };
    Ok((key.trim().to_string(), number))
}

fn main() {
    println!("{:?}", parse_kv("retries = 3"));
    println!("{:?}", parse_kv("broken"));
    println!("{:?}", parse_kv("a=b"));
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn valid_line() {
        assert_eq!(parse_kv("retries = 3"), Ok((String::from("retries"), 3)));
    }

    #[test]
    fn invalid_lines() {
        assert_eq!(parse_kv("broken"), Err(String::from("missing '=' in \"broken\"")));
        assert_eq!(parse_kv("a=b"), Err(String::from("\"b\" is not a number")));
    }
}
```

Output of `cargo run`:

```text
Ok(("retries", 3))
Err("missing '=' in \"broken\"")
Err("\"b\" is not a number")
```

Why it works: each `let else` checks one thing and leaves with a specific message, so the success path reads straight down with no nesting. Because the `else` blocks build messages, `clippy` does not suggest `?` here.

</details>

### Exercise 4 — A tiny command interpreter (hard)

Define `enum Command { Add(i32), Reset, Print }` (derive `Debug` and `PartialEq`). Write `fn parse_command(line: &str) -> Option<Command>`:

- `"add 5"` gives `Add(5)`, `"reset"` gives `Reset`, `"print"` gives `Print`;
- anything else (empty text, `"add"`, `"add x"`, `"reset now"`, `"bogus"`) gives `None`.

Then write `fn apply(total: i32, command: &Command) -> i32` (`Add` adds, `Reset` returns 0, `Print` prints `total = N` and returns the total unchanged). Test valid and invalid lines, and `apply`.

<details>
<summary>Hint</summary>

Split the line with `let mut parts = line.split_whitespace();` and match on a tuple of three calls: `match (parts.next(), parts.next(), parts.next()) { (Some("add"), Some(arg), None) => ..., (Some("reset"), None, None) => ..., ... , _ => None }`. The third `None` rejects extra words such as in `"reset now"`.

</details>

<details>
<summary>Answer</summary>

```rust
#[derive(Debug, PartialEq)]
enum Command {
    Add(i32),
    Reset,
    Print,
}

fn parse_command(line: &str) -> Option<Command> {
    let mut parts = line.split_whitespace();
    match (parts.next(), parts.next(), parts.next()) {
        (Some("add"), Some(arg), None) => match arg.parse::<i32>() {
            Ok(n) => Some(Command::Add(n)),
            Err(_) => None,
        },
        (Some("reset"), None, None) => Some(Command::Reset),
        (Some("print"), None, None) => Some(Command::Print),
        _ => None,
    }
}

fn apply(total: i32, command: &Command) -> i32 {
    match command {
        Command::Add(n) => total + n,
        Command::Reset => 0,
        Command::Print => {
            println!("total = {total}");
            total
        }
    }
}

fn main() {
    let mut total = 0;
    for line in ["add 5", "add 7", "print", "bogus", "add x", "reset", "print"] {
        match parse_command(line) {
            Some(command) => total = apply(total, &command),
            None => println!("cannot understand {line:?}"),
        }
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn parses_valid_commands() {
        assert_eq!(parse_command("add 5"), Some(Command::Add(5)));
        assert_eq!(parse_command("add -2"), Some(Command::Add(-2)));
        assert_eq!(parse_command("reset"), Some(Command::Reset));
        assert_eq!(parse_command("print"), Some(Command::Print));
    }

    #[test]
    fn rejects_invalid_commands() {
        assert_eq!(parse_command(""), None);
        assert_eq!(parse_command("add"), None);
        assert_eq!(parse_command("add x"), None);
        assert_eq!(parse_command("reset now"), None);
        assert_eq!(parse_command("bogus"), None);
    }

    #[test]
    fn apply_changes_total() {
        assert_eq!(apply(10, &Command::Add(5)), 15);
        assert_eq!(apply(10, &Command::Reset), 0);
    }
}
```

Output of `cargo run`:

```text
total = 12
cannot understand "bogus"
cannot understand "add x"
total = 0
```

Why it works: matching on a tuple of `Option`s lets one `match` describe the whole shape of a command, including "no extra words". Text is parsed into the `Command` enum once; after that `apply` is a complete, compiler-checked `match` over the enum. This parse-then-apply structure is the core of every command-line tool and interpreter, and you will use it in the phone book project ([Lesson 04.11](11-project-phone-book.md)).

</details>

## Quiz

1. When do you prefer `if let` over `match`?

<details>
<summary>Answer</summary>

When only one pattern matters and the rest should be ignored (or handled in a single `else`). If you need to handle several cases, or want the compiler to check completeness, use `match`.

</details>

2. What is the difference between `if let Some(x) = value` and `if Some(x) == value`?

<details>
<summary>Answer</summary>

`if let` is pattern matching: it binds `x` to the inner value when the pattern fits. `==` compares two existing values; here `x` would have to exist already, and nothing is bound.

</details>

3. What must the `else` block of a `let else` do?

<details>
<summary>Answer</summary>

It must diverge — leave the normal flow with `return`, `break`, `continue` or `panic!` (anything of type `!`). Otherwise the variables from the pattern would be undefined after the statement (`E0308`).

</details>

4. What does `while let Some(top) = stack.pop()` do?

<details>
<summary>Answer</summary>

It removes and processes items from the end of the list one by one, and stops when `pop()` returns `None` (the list is empty).

</details>

5. What is the difference between a refutable and an irrefutable pattern? Where is each allowed?

<details>
<summary>Answer</summary>

An irrefutable pattern always matches (`x`, `(a, b)`); a refutable one can fail (`Some(x)`). Plain `let`, function parameters and `for` need irrefutable patterns. `match`, `if let`, `while let` and `let else` accept refutable ones.

</details>

6. What does `[first, .., last]` match?

<details>
<summary>Answer</summary>

A slice or array with at least two elements; it binds the first and last elements and ignores everything between.

</details>

7. What is the difference between `let _ = value;` and `let _value = value;`?

<details>
<summary>Answer</summary>

`_` binds nothing, so `value` is not moved and can be used later. `_value` is a real variable: the value is moved into it.

</details>

8. What does `cargo clippy` suggest when a `let else` only returns `None`?

<details>
<summary>Answer</summary>

To use the `?` operator instead (lint `question_mark`). You will learn `?` in Lesson 04.10.

</details>

## Summary

- `if let PATTERN = VALUE { ... } else { ... }` handles one case of a `match`; the compiler does not check it for completeness.
- `while let` repeats while the pattern fits: the natural tool for `pop()` and `next()`.
- `let else` binds variables or leaves (`return`, `continue`, `break`, `panic!`); it keeps the happy path flat.
- `matches!(value, pattern)` returns `true` or `false`; a let chain (`if let ... && cond`) combines patterns and conditions (edition 2024).
- Patterns work in `let`, function parameters, `for` loops and closures; `_` ignores one value, `..` ignores the rest, `_name` still moves.
- Irrefutable patterns are required in plain `let`, parameters and `for`; refutable ones go to `match`, `if let`, `while let` and `let else`.
- Slice patterns (`[first, .., last]`, `[first, rest @ ..]`) take arrays and slices apart; [Lesson 06.3](../06-rust-advanced/03-patterns-in-depth.md) goes deeper into patterns.
- Test both exits of every `let else`.

## Sources and further reading

- [The Rust Programming Language — 6.3 Concise Control Flow with if let and let else](https://doc.rust-lang.org/book/ch06-03-if-let.html) — the official book chapter on this topic.
- [The Rust Programming Language — 19.1 All the Places Patterns Can Be Used](https://doc.rust-lang.org/book/ch19-01-all-the-places-for-patterns.html) — the full list of places where patterns appear.
- [The Rust Programming Language — 19.2 Refutability](https://doc.rust-lang.org/book/ch19-02-refutability.html) — refutable and irrefutable patterns.
- [The Rust Programming Language — 19.3 Pattern Syntax](https://doc.rust-lang.org/book/ch19-03-pattern-syntax.html) — every pattern form in one place.
- [Rust by Example — let-else](https://doc.rust-lang.org/rust-by-example/flow_control/let_else.html) — runnable `let else` examples.
- [std::matches!](https://doc.rust-lang.org/std/macro.matches.html) — documentation of the `matches!` macro.

---

[🏠 Start](../START-HERE.md) · [← 04.4 Option and match](04-option-and-match.md) · [📋 Phase 04](README.md) · [04.6 Modules, crates and packages →](06-modules-crates-and-packages.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
