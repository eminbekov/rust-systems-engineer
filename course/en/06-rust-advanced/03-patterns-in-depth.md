<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 06.2 Advanced traits and types](02-advanced-traits-and-types.md) · [📋 Phase 06](README.md) · [06.4 Interior mutability →](04-interior-mutability.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/06-rust-advanced/03-patterns-in-depth.md) <!-- nav -->

# Lesson 06.3 — Patterns in depth

> **Phase 06 — Rust advanced** · Lesson 3 of 12 · ⏱ about 3 hours (2 sittings)

## What you will learn

- Every place where Rust accepts a pattern, not only `match`.
- The difference between refutable and irrefutable patterns, and why the compiler cares.
- The full pattern syntax: literals, ranges, `|`, destructuring, `..`, guards, `@` bindings.
- Slice patterns such as `[first, .., last]`, useful for parsing commands.
- How patterns work with references, and what changed in edition 2024.
- `let else`, let chains and the `matches!` macro.

## Before you start

- You finished [Lesson 06.2 — Advanced traits and types](02-advanced-traits-and-types.md).
- You know `match`, `Option` and enums from [Lesson 04.4 — Option and match](../04-structs-enums-collections-errors/04-option-and-match.md) and `if let`, `let else` from [Lesson 04.5](../04-structs-enums-collections-errors/05-if-let-let-else-and-patterns.md). This lesson goes deeper, so a quick re-read of Lesson 04.5 helps.
- Create a practice project: `cargo new code/p06-patterns`.

## New words

| Word | Meaning |
|---|---|
| **pattern** | A shape that a value is compared with. It can also name parts of the value: `Some(x)`, `(a, b)`, `1..=5`. |
| **refutable pattern** | A pattern that can fail to match some values. `Some(x)` fails for `None`. |
| **irrefutable pattern** | A pattern that always matches. `x` and `(a, b)` always match. |
| **destructuring** | Taking a value apart (a tuple, struct, enum, array) and giving names to its parts. |
| **wildcard** | The pattern `_`. It matches anything and does not bind a name. |
| **match guard** | An extra `if condition` after a pattern in a `match` arm. |
| **`@` binding** | `name @ pattern`: test a value against a pattern and also keep it under a name. |
| **slice pattern** | A pattern for arrays and slices, such as `[first, .., last]`. |
| **exhaustive** | A `match` is exhaustive when its arms cover every possible value. Rust requires this. |
| **binding mode** | How a pattern variable receives its value: by move/copy, or as a reference. |
| **let chain** | An `if` or `while` condition that joins `let` patterns and boolean tests with `&&`. |

## 1. Where patterns are allowed

You already know `match` and `if let`. Patterns appear in many more places:

```rust
fn print_point(&(x, y): &(i32, i32)) {
    println!("point at {x}, {y}");
}

fn main() {
    // 1. `let` takes a pattern
    let (name, age) = ("Ana", 31);
    println!("{name} is {age}");

    // 2. function parameters take patterns
    print_point(&(3, 4));

    // 3. `for` takes a pattern
    for (index, letter) in "abc".chars().enumerate() {
        println!("{index}: {letter}");
    }

    // 4. `while let` repeats while the pattern matches
    let mut stack = vec![1, 2, 3];
    while let Some(top) = stack.pop() {
        println!("popped {top}");
    }

    // 5. `if let` and `let else`
    let maybe_number = Some(7);
    if let Some(n) = maybe_number {
        println!("got {n}");
    }
    let Some(n) = maybe_number else {
        return;
    };
    println!("still {n}");
}
```

```text
Ana is 31
point at 3, 4
0: a
1: b
2: c
popped 3
popped 2
popped 1
got 7
still 7
```

**What happens:**

1. `let (name, age) = ("Ana", 31);` — the left side of every `let` is a pattern. The tuple pattern `(name, age)` splits the tuple. (The plain `let x = 5;` is also a pattern: just a name.) The `{name}` inside `println!` prints the variable with that name.
2. `fn print_point(&(x, y): &(i32, i32))` — a function parameter is a pattern. This one removes the `&` and splits the pair in the signature.
3. `for (index, letter) in ...enumerate()` — the loop variable is a pattern too.
4. `while let Some(top) = stack.pop()` repeats as long as the pattern matches. `pop` returns `None` when the vector is empty, and the loop ends.
5. `if let` runs its block only when the pattern matches. `let ... else` handles the opposite case: if the pattern does not match, the `else` block must leave (`return`, `break`, `continue` or `panic!`).

## 2. Refutable and irrefutable patterns

A pattern is **irrefutable** if it matches every possible value of the type: `x`, `(a, b)`, `_`. A pattern is **refutable** if some values do not match: `Some(x)` does not match `None`; `1..=5` does not match 6.

The rule: `let`, `for` and function parameters accept only irrefutable patterns, because there is no code path for "no match". `if let`, `while let`, `let else` and `match` arms accept refutable patterns.

```rust
fn main() {
    let input: Option<i32> = Some(5);
    let Some(value) = input;
    println!("{value}");
}
```

```text
error[E0005]: refutable pattern in local binding
 --> src/main.rs:3:9
  |
3 |     let Some(value) = input;
  |         ^^^^^^^^^^^ pattern `None` not covered
  |
  = note: `let` bindings require an "irrefutable pattern", like a `struct` or an `enum` with only one variant
  = note: for more information, visit https://doc.rust-lang.org/book/ch19-02-refutability.html
  = note: the matched value is of type `Option<i32>`
help: you might want to use `let...else` to handle the variant that isn't matched
  |
3 |     let Some(value) = input else { todo!() };
  |                             ++++++++++++++++
```

**What happens:**

1. `input` is an `Option<i32>`, so it can be `None`.
2. `let Some(value) = input;` has no answer for `None`. The compiler refuses with `E0005` and names the uncovered case.
3. Add an `else` block (`let ... else { return };` or `else { panic!(...) }`) or use `match` / `if let`.

The opposite mistake is only a warning: an irrefutable pattern in `if let` is pointless, because it always matches.

```text
warning: irrefutable `if let` pattern
 --> src/main.rs:3:8
  |
3 |     if let x = value {
  |        ^^^^^^^^^^^^^
  |
  = note: this pattern will always match, so the `if let` is useless
  = help: consider replacing the `if let` with a `let`
  = note: `#[warn(irrefutable_let_patterns)]` on by default
```

## 3. Literals, ranges and alternatives

Inside `match` arms you can combine several simple patterns:

```rust
fn describe(n: i32) -> &'static str {
    match n {
        0 => "zero",
        2 | 4 | 6 | 8 => "even digit",
        1..=9 => "odd digit",
        10.. => "big",
        i32::MIN..=-1 => "negative",
    }
}

fn kind(c: char) -> &'static str {
    match c {
        'a' | 'e' | 'i' | 'o' | 'u' => "vowel",
        'a'..='z' => "consonant",
        '0'..='9' => "digit",
        _ => "other",
    }
}

fn main() {
    for n in [-5, 0, 2, 7, 10, 500] {
        println!("{n:>4} -> {}", describe(n));
    }
    for c in ['e', 'x', '7', '#'] {
        println!("{c} -> {}", kind(c));
    }
}
```

```text
  -5 -> negative
   0 -> zero
   2 -> even digit
   7 -> odd digit
  10 -> big
 500 -> big
e -> vowel
x -> consonant
7 -> digit
# -> other
```

**What happens:**

1. `0` matches the literal value 0.
2. `2 | 4 | 6 | 8` is an **or-pattern**: it matches 2, 4, 6 or 8.
3. `1..=9` is an inclusive range: 1 up to and including 9. Arms are tried from top to bottom, so the even digits were already taken by the arm above, and only the odd digits reach this arm. Ranges work for integers and `char`. The form `10..` means "10 and above", and `i32::MIN..=-1` means "all negative numbers".
4. The arms together cover every `i32`, so the `match` is **exhaustive** and needs no `_`. For `char` we need `_`, because we cannot list all characters.
5. The arms are tested from top to bottom, so `'a' | 'e' | ...` must come before the range `'a'..='z'`.

If you forget a case, the compiler tells you which values are missing:

```text
error[E0004]: non-exhaustive patterns: `100_u8..=u8::MAX` not covered
 --> src/main.rs:3:22
  |
3 |     let text = match n {
  |                      ^ pattern `100_u8..=u8::MAX` not covered
  |
  = note: the matched value is of type `u8`
help: ensure that all possible cases are being handled by adding a match arm with a wildcard pattern or an explicit pattern as shown
  |
5 ~         10..=99 => "two digits",
6 ~         100_u8..=u8::MAX => todo!(),
  |
```

> 💡 **Tip:** A plain name in a pattern is always a *new variable* that matches everything. It is never a comparison with an existing variable of the same name. To compare with a value, use a constant, a literal, or a match guard (section 5).

## 4. Destructuring structs, enums and tuples

Destructuring takes a value apart in the pattern itself. You can go as deep as you like:

```rust
#[derive(Debug)]
struct Point {
    x: i32,
    y: i32,
}

#[derive(Debug)]
enum Shape {
    Circle {
        center: Point,
        radius: u32,
    },
    Rect {
        top_left: Point,
        bottom_right: Point,
    },
    Dot(Point),
}

fn describe(shape: &Shape) -> String {
    match shape {
        Shape::Circle {
            center: Point { x: 0, y: 0 },
            radius,
        } => {
            format!("circle at the origin, radius {radius}")
        }
        Shape::Circle { radius, .. } => format!("circle, radius {radius}"),
        Shape::Rect {
            top_left: Point { x: x1, y: y1 },
            bottom_right: Point { x: x2, y: y2 },
        } => format!("rectangle {}x{}", x2 - x1, y1 - y2),
        Shape::Dot(Point { x, .. }) => format!("dot with x = {x}"),
    }
}

fn main() {
    let shapes = [
        Shape::Circle {
            center: Point { x: 0, y: 0 },
            radius: 5,
        },
        Shape::Circle {
            center: Point { x: 2, y: 3 },
            radius: 1,
        },
        Shape::Rect {
            top_left: Point { x: 1, y: 6 },
            bottom_right: Point { x: 5, y: 2 },
        },
        Shape::Dot(Point { x: 9, y: 9 }),
    ];
    for shape in &shapes {
        println!("{}", describe(shape));
    }

    // Ignoring parts of a value
    let (first, _, third) = (1, 2, 3);
    let _unused = first + third; // a leading underscore silences the "unused" warning
    let numbers = (10, 20, 30, 40, 50);
    let (head, .., last) = numbers;
    println!("{head} {last}");
}
```

```text
circle at the origin, radius 5
circle, radius 1
rectangle 4x4
dot with x = 9
10 50
```

**What happens:**

1. `Shape::Circle { center: Point { x: 0, y: 0 }, radius }` checks that the shape is a circle whose center is exactly `(0, 0)`, and binds `radius`. A literal inside a pattern is a test.
2. `Shape::Circle { radius, .. }` — the `..` means "I do not care about the other fields". The shorthand `radius` means `radius: radius`.
3. `Point { x: x1, y: y1 }` renames the fields while binding them. Four numbers are bound in one pattern and used for `x2 - x1` (width) and `y1 - y2` (height).
4. `Shape::Dot(Point { x, .. })` takes only `x` out of a tuple variant.
5. `match shape` where `shape` is `&Shape`: the bindings become references automatically (section 7).
6. `_` ignores one value, `..` ignores a run of values: `(head, .., last)` takes the first and last item of a 5-tuple. A name that starts with an underscore (`_unused`) is a real variable, but Rust does not warn if you do not use it.

## 5. Guards and `@` bindings

A **match guard** is an `if` after the pattern. The arm matches only if the pattern matches and the condition is true. The `@` operator keeps the matched value under a name while testing it against a pattern:

```rust
fn classify(pair: (i32, i32)) -> String {
    match pair {
        (x, y) if x == y => format!("{x} equals {y}"),
        (x, y) if x + y == 0 => String::from("opposites"),
        (x, _) if x % 2 == 0 => format!("{x} is even"),
        _ => String::from("nothing special"),
    }
}

fn age_group(age: u32) -> String {
    match age {
        n @ 0..=12 => format!("child, {n} years"),
        n @ 13..=19 => format!("teenager, {n} years"),
        n => format!("adult, {n} years"),
    }
}

fn main() {
    for pair in [(3, 3), (4, -4), (6, 1), (5, 2)] {
        println!("{pair:?}: {}", classify(pair));
    }
    for age in [7, 15, 40] {
        println!("{}", age_group(age));
    }

    // A guard applies to the WHOLE `|` pattern
    let n = 4;
    match n {
        2 | 4 | 6 if n > 4 => println!("even digit above 4"),
        2 | 4 | 6 => println!("even digit up to 4"),
        _ => println!("something else"),
    }
}
```

```text
(3, 3): 3 equals 3
(4, -4): opposites
(6, 1): 6 is even
(5, 2): nothing special
child, 7 years
teenager, 15 years
adult, 40 years
even digit up to 4
```

**What happens:**

1. `(x, y) if x == y` — the pattern alone matches every pair. The guard narrows it. This is how you compare two parts of the same value.
2. Arms are tried from the top. `(4, -4)` fails the first guard (4 is not -4), passes the second (4 + -4 == 0).
3. `n @ 0..=12` tests that the value is in 0 to 12 *and* binds the number to `n`. Without `@` you would have to choose between the range and the name.
4. `2 | 4 | 6 if n > 4` — a guard applies to the **whole** or-pattern, not just to `6`. For `n = 4` the pattern matches but the guard is false, so the next arm runs.
5. The compiler does not use guards when it checks exhaustiveness. A `match` that ends with a guarded arm still needs a final arm without a guard.

> 📏 **Convention:** Keep guards short. If a guard needs several lines, move the logic into a function (`if is_valid(&x)`) or into an `if` inside the arm.

## 6. Slice patterns

A **slice pattern** matches arrays and slices by their length and their elements. It is perfect for command parsers:

```rust
fn run(command: &[&str]) -> String {
    match command {
        [] => String::from("empty command"),
        ["quit"] => String::from("bye"),
        ["add", name, phone] => format!("adding {name} with {phone}"),
        ["remove", name] => format!("removing {name}"),
        ["echo", words @ ..] => words.join(" "),
        [first, .., last] => format!("unknown command from {first} to {last}"),
        [single] => format!("unknown command {single}"),
    }
}

fn main() {
    let lines = [
        "",
        "quit",
        "add ana 555-0101",
        "remove bo",
        "echo hello brave new world",
        "move left up high",
        "jump",
    ];
    for line in lines {
        let parts: Vec<&str> = line.split_whitespace().collect();
        println!("{:<28} => {}", format!("{line:?}"), run(&parts));
    }

    let numbers = [1, 2, 3, 4, 5];
    let [a, b, rest @ ..] = numbers;
    println!("{a} {b} {rest:?}");
}
```

```text
""                           => empty command
"quit"                       => bye
"add ana 555-0101"           => adding ana with 555-0101
"remove bo"                  => removing bo
"echo hello brave new world" => hello brave new world
"move left up high"          => unknown command from move to high
"jump"                       => unknown command jump
1 2 [3, 4, 5]
```

**What happens:**

1. `command` is a `&[&str]`: a slice of words.
2. `[]` matches an empty slice. `["quit"]` matches exactly one element that equals `"quit"`.
3. `["add", name, phone]` matches exactly three elements, the first equal to `"add"`, and binds the other two.
4. `["echo", words @ ..]` matches `"echo"` followed by any number of elements. `words @ ..` binds the rest as a sub-slice. Its type is `&[&str]`, so `join(" ")` works.
5. `[first, .., last]` needs at least two elements and ignores everything between them.
6. `[single]` handles the one remaining case: exactly one word that is not `quit`. Together the arms cover every possible length, so the `match` is exhaustive and the compiler accepts it without `_`.
7. The last lines show the same syntax with `let` on an array: `let [a, b, rest @ ..] = numbers;`. For an array with a known length this pattern is irrefutable.

> 🧪 **Testing:** Slice patterns make parser functions easy to test: pass a slice like `&["add", "ana", "555"]` and compare the result. You will do this in Exercise 2.

## 7. Patterns and references

When you match on a reference, Rust helps you. If the value is `&Option<String>` and the pattern is `Some(nick)`, the variable `nick` becomes a `&String`: a *reference* to the inside. This is called the **default binding mode**. Nothing is moved:

```rust
#[derive(Debug)]
struct User {
    name: String,
    nickname: Option<String>,
}

fn main() {
    let user = User {
        name: String::from("Alexandra"),
        nickname: Some(String::from("Sasha")),
    };

    // Matching a REFERENCE: `name` becomes a reference, nothing is moved
    if let Some(nick) = &user.nickname {
        println!("{} is called {nick}", user.name);
    }
    println!("user is still usable: {user:?}");

    // `&` in a pattern removes one layer of reference
    let numbers = [1, 2, 3];
    let total: i32 = numbers.iter().map(|&n| n * 10).sum();
    println!("{total}");

    // `ref mut` takes a mutable borrow while destructuring
    let mut pair = (String::from("a"), String::from("b"));
    let (ref mut first, _) = pair;
    first.push('!');
    println!("{pair:?}");

    // `matches!` returns a bool for a pattern
    let c = 'k';
    println!("{}", matches!(c, 'a'..='m'));
}
```

```text
Alexandra is called Sasha
user is still usable: User { name: "Alexandra", nickname: Some("Sasha") }
60
("a!", "b")
true
```

**What happens:**

1. `if let Some(nick) = &user.nickname` matches on a reference, so `nick` is a `&String`. Afterwards `user` can still be used.
2. `.map(|&n| n * 10)` — the iterator gives `&i32`. The pattern `&n` removes the reference and `n` is an `i32`.
3. `let (ref mut first, _) = pair;` uses `ref mut` to take a *mutable borrow* of the field instead of moving it. `first.push('!')` changes the string inside `pair`.
4. `matches!(c, 'a'..='m')` expands to a `match` that returns `true` if the pattern matches and `false` otherwise. It is the shortest way to test a pattern in a condition.

What if you forget the `&`? The value is moved out of `user`:

```text
error[E0382]: borrow of partially moved value: `user`
  --> src/main.rs:13:22
   |
10 |     if let Some(name) = user.name {
   |                 ---- value partially moved here
...
13 |     println!("{:?}", user);
   |                      ^^^^ value borrowed here after partial move
   |
   = note: partial move occurs because value has type `String`, which does not implement the `Copy` trait
help: borrow this binding in the pattern to avoid moving the value
   |
10 |     if let Some(ref name) = user.name {
   |                 +++
```

The compiler suggests `ref name`. In modern code you would rather match on `&user.name`.

**Edition 2024 change.** In edition 2024, if the default binding mode already makes `n` a reference, you may not write `ref`, `ref mut`, `mut` or `&` explicitly inside the same pattern:

```text
error: cannot explicitly borrow within an implicitly-borrowing pattern
 --> src/main.rs:4:14
  |
4 |         Some(ref n) => println!("{n}"),
  |              ^^^ explicit `ref` binding modifier not allowed when implicitly borrowing
  |
  = note: for more information, see <https://doc.rust-lang.org/reference/patterns.html#binding-modes>
note: matching on a reference type with a non-reference pattern implicitly borrows the contents
 --> src/main.rs:4:9
  |
4 |         Some(ref n) => println!("{n}"),
  |         ^^^^^^^^^^^ this non-reference pattern matches on a reference type `&_`
help: remove the unnecessary binding modifier
  |
4 -         Some(ref n) => println!("{n}"),
4 +         Some(n) => println!("{n}"),
  |
```

The fix is the one the compiler shows: remove `ref`. Explicit `ref` is still allowed when the value is matched by value (like `let (ref mut first, _) = pair;`), but you rarely need it.

## 8. Let chains

A **let chain** joins `let` patterns and ordinary conditions with `&&` in one `if` (or `while`). This feature is available in edition 2024:

```rust
fn main() {
    let width: Option<u32> = Some(8);
    let height: Option<u32> = Some(5);

    // A let chain mixes patterns and boolean conditions with `&&`
    if let Some(w) = width
        && let Some(h) = height
        && w > h
    {
        println!("wide: {w} x {h}");
    }

    let words = ["apple", "bob"];
    if let Some(first) = words.first()
        && first.len() > 3
    {
        println!("first word is long: {first}");
    }
}
```

```text
wide: 8 x 5
first word is long: apple
```

**What happens:**

1. Conditions are checked left to right. Each `let` that succeeds makes its variable available for the conditions after it and inside the block.
2. The first `if` needs both options to be `Some` *and* `w > h`. If any part fails, the block is skipped.
3. Without let chains you would write nested `if let` blocks, or `match` on a tuple.
4. `words.first()` returns `Option<&&str>`. `first.len()` works because the method call looks through the references automatically.

## Common mistakes

### 1. Using a refutable pattern in `let`

`let Some(x) = opt;` gives `E0005`. Use `let Some(x) = opt else { ... };`, `if let`, or `match`.

### 2. Moving a value out by accident

`if let Some(name) = user.name` moves the `String` out of `user` (`E0382 borrow of partially moved value`). Match on a reference: `if let Some(name) = &user.name`.

### 3. A plain name does not compare

```rust
let limit = 10;
match n {
    limit => println!("n equals limit"), // wrong: `limit` is a NEW variable, it matches every n
    _ => println!("n is different"),
}
```

The first arm matches everything. The compiler warns that the `_` arm is unreachable. Use a guard: `x if x == limit`.

### 4. Writing `ref` after the 2024 edition rules apply

`cannot explicitly borrow within an implicitly-borrowing pattern`: delete `ref`.

### 5. Forgetting the order of arms

A broad arm (`n`, `_`, a wide range) before a narrow one hides it. Put specific patterns first.

## Try it yourself

1. Open `~/projects/rust/code/p06-patterns`.
2. Copy the `slices.rs` example (section 6) into `src/main.rs` and run `cargo run`.
3. Add a new command `["mv", from, to]` that returns `moving {from} to {to}`. Run again with the line `mv a.txt b.txt`.
4. Add a command `["help", ..]` that prints `help is coming soon` no matter how many words follow. Put it before the catch-all arms.
5. Run `cargo fmt` and `cargo clippy`.
6. Commit: `git add code/p06-patterns` and `git commit -m "feat(p06): slice-pattern command parser"`.

## Exercises

### Exercise 1 — Letter grades (easy)

Write `fn grade(score: u32) -> char`: 90 to 100 gives `'A'`, 80 to 89 `'B'`, 70 to 79 `'C'`, 0 to 69 `'F'`, and anything above 100 gives `'?'`. Write a test for the boundaries (0, 69, 70, 89, 90, 100) and one for 101.

<details>
<summary>Hint</summary>

Use range patterns with `..=` in a `match`. The last arm is `_`. A test is a function with `#[test]` inside `mod tests`.

</details>

<details>
<summary>Answer</summary>

```rust
fn grade(score: u32) -> char {
    match score {
        90..=100 => 'A',
        80..=89 => 'B',
        70..=79 => 'C',
        0..=69 => 'F',
        _ => '?', // scores above 100 are invalid
    }
}

fn main() {
    for score in [95, 85, 72, 10, 101] {
        println!("{score} -> {}", grade(score));
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn boundaries() {
        assert_eq!(grade(100), 'A');
        assert_eq!(grade(90), 'A');
        assert_eq!(grade(89), 'B');
        assert_eq!(grade(70), 'C');
        assert_eq!(grade(69), 'F');
        assert_eq!(grade(0), 'F');
    }

    #[test]
    fn invalid_score() {
        assert_eq!(grade(101), '?');
    }
}
```

```text
95 -> A
85 -> B
72 -> C
10 -> F
101 -> ?
```

```text

running 2 tests
..
test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

Why it works: ranges are checked in order; since they do not overlap, the order does not matter here. The `_` arm catches everything above 100.

</details>

### Exercise 2 — A command parser with a test (medium)

Define `enum Command<'a>` with `Add { name, phone }`, `Remove(name)`, `List` and `Say(words)`. Write `fn parse<'a>(parts: &[&'a str]) -> Option<Command<'a>>` using a slice pattern. Return `None` for unknown or malformed commands. Write tests for a valid `add`, for `say` with several words, and for bad input.

<details>
<summary>Hint</summary>

One arm per command: `["add", name, phone]`, `["remove", name]`, `["list"]`, `["say", words @ ..]`. The last arm is `_ => None`. For `Say`, turn the sub-slice into a vector with `words.to_vec()`. The lifetime `'a` belongs to the text pieces; you learn why it is needed in [Lesson 06.5](05-lifetimes-in-depth.md).

</details>

<details>
<summary>Answer</summary>

```rust
#[derive(Debug, PartialEq)]
enum Command<'a> {
    Add { name: &'a str, phone: &'a str },
    Remove(&'a str),
    List,
    Say(Vec<&'a str>),
}

fn parse<'a>(parts: &[&'a str]) -> Option<Command<'a>> {
    match parts {
        ["add", name, phone] => Some(Command::Add { name, phone }),
        ["remove", name] => Some(Command::Remove(name)),
        ["list"] => Some(Command::List),
        ["say", words @ ..] => Some(Command::Say(words.to_vec())),
        _ => None,
    }
}

fn main() {
    let line = "add ana 555-0101";
    let parts: Vec<&str> = line.split_whitespace().collect();
    println!("{:?}", parse(&parts));
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn parses_add() {
        assert_eq!(
            parse(&["add", "ana", "555"]),
            Some(Command::Add {
                name: "ana",
                phone: "555"
            })
        );
    }

    #[test]
    fn parses_say_with_many_words() {
        assert_eq!(
            parse(&["say", "hi", "there"]),
            Some(Command::Say(vec!["hi", "there"]))
        );
    }

    #[test]
    fn rejects_wrong_arity_and_unknown() {
        assert_eq!(parse(&["add", "ana"]), None);
        assert_eq!(parse(&["fly"]), None);
        assert_eq!(parse(&[]), None);
    }
}
```

```text
Some(Add { name: "ana", phone: "555-0101" })
```

```text

running 3 tests
...
test result: ok. 3 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

Why it works: each arm checks both the length and the first word, so `add ana` with only two words falls through to `_ => None`. Shorthand `Command::Add { name, phone }` uses the variables `name` and `phone` bound by the pattern.

</details>

### Exercise 3 — Quadrants with guards (medium)

Write `fn quadrant(point: (i32, i32)) -> &'static str` that returns `"origin"`, `"on the x axis"`, `"on the y axis"`, or `"first quadrant"` to `"fourth quadrant"` (first: x and y positive; second: x negative, y positive; and so on). Print the answer for seven points.

<details>
<summary>Hint</summary>

Put the special cases first: `(0, 0)`, `(_, 0)`, `(0, _)`. For the quadrants use guards: `(x, y) if x > 0 && y > 0`. The last arm can be `_` for the fourth quadrant.

</details>

<details>
<summary>Answer</summary>

```rust
fn quadrant(point: (i32, i32)) -> &'static str {
    match point {
        (0, 0) => "origin",
        (_, 0) => "on the x axis",
        (0, _) => "on the y axis",
        (x, y) if x > 0 && y > 0 => "first quadrant",
        (x, y) if x < 0 && y > 0 => "second quadrant",
        (x, y) if x < 0 && y < 0 => "third quadrant",
        _ => "fourth quadrant",
    }
}

fn main() {
    for p in [(0, 0), (5, 0), (0, -2), (1, 1), (-1, 1), (-1, -1), (1, -1)] {
        println!("{p:?} -> {}", quadrant(p));
    }
}
```

```text
(0, 0) -> origin
(5, 0) -> on the x axis
(0, -2) -> on the y axis
(1, 1) -> first quadrant
(-1, 1) -> second quadrant
(-1, -1) -> third quadrant
(1, -1) -> fourth quadrant
```

Why it works: the arms are tested in order. By the time we reach the guards, neither coordinate is zero, so four sign combinations remain, and the last arm handles the one that is left.

</details>

### Exercise 4 — Parse `key = value` (hard)

Write `fn parse_setting(line: &str) -> Option<(String, u32)>` that splits a line at `=`, trims spaces, parses the right side as `u32` and returns `Some((key, number))`. Return `None` when there is no `=`, when the number is invalid, or when the key is empty. Use `let else`. Test one valid and three invalid lines.

<details>
<summary>Hint</summary>

`line.split_once('=')` returns `Option<(&str, &str)>`. On `Option`, the `?` operator returns `None` early. For the number use `let Ok(number) = value.trim().parse::<u32>() else { return None; };`.

</details>

<details>
<summary>Answer</summary>

```rust
fn parse_setting(line: &str) -> Option<(String, u32)> {
    let (key, value) = line.split_once('=')?;
    let Ok(number) = value.trim().parse::<u32>() else {
        return None;
    };
    if key.trim().is_empty() {
        return None;
    }
    Some((key.trim().to_string(), number))
}

fn main() {
    for line in ["port = 8080", "port=abc", "no equals sign", " = 5"] {
        println!("{line:?} -> {:?}", parse_setting(line));
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn valid_line() {
        assert_eq!(
            parse_setting("port = 8080"),
            Some((String::from("port"), 8080))
        );
    }

    #[test]
    fn invalid_lines() {
        assert_eq!(parse_setting("port=abc"), None);
        assert_eq!(parse_setting("no equals sign"), None);
        assert_eq!(parse_setting(" = 5"), None);
    }
}
```

```text
"port = 8080" -> Some(("port", 8080))
"port=abc" -> None
"no equals sign" -> None
" = 5" -> None
```

```text

running 2 tests
..
test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

Why it works: `?` on an `Option` returns `None` from the function when the value is `None`. `let else` does the same for any refutable pattern, here `Ok(number)`. Clippy suggests `?` in the first case, so the code uses it there.

</details>

## Quiz

1. Name five places where a pattern can appear besides `match`.

<details>
<summary>Answer</summary>

`let` statements, function (and closure) parameters, `for` loops, `if let` / `while let` / `let else`, and let chains.

</details>

2. Why does `let Some(x) = opt;` not compile?

<details>
<summary>Answer</summary>

`Some(x)` is refutable (it does not match `None`), but `let` needs an irrefutable pattern because there is no code to run when it fails. Use `let ... else`, `if let` or `match`.

</details>

3. What does `n @ 13..=19` do?

<details>
<summary>Answer</summary>

It matches a value in the range 13 to 19 and binds that value to the name `n` at the same time.

</details>

4. How does a guard interact with an or-pattern like `2 | 4 | 6 if cond`?

<details>
<summary>Answer</summary>

The guard applies to the whole or-pattern. If the value matches any alternative but the guard is false, the arm is skipped (it does not try another alternative).

</details>

5. What does `["echo", words @ ..]` match?

<details>
<summary>Answer</summary>

A slice with at least one element where the first element is `"echo"`. The remaining elements (possibly none) are bound as the sub-slice `words`.

</details>

6. What is the type of `nick` in `if let Some(nick) = &user.nickname` when `nickname` is `Option<String>`?

<details>
<summary>Answer</summary>

`&String`. Matching on a reference makes the binding a reference (the default binding mode), so nothing is moved out of `user`.

</details>

7. What is a let chain, and which condition does `if let Some(w) = a && w > 3` express?

<details>
<summary>Answer</summary>

A let chain joins `let` patterns and boolean conditions with `&&`. The example is true only if `a` is `Some`, and the value inside is greater than 3; `w` can be used in the block.

</details>

## Summary

- Patterns appear in `let`, function parameters, `for`, `match`, `if let`, `while let`, `let else` and let chains.
- Irrefutable patterns always match (`let`, `for`, parameters need them); refutable patterns can fail (`if let`, `while let`, `match`, `let else`).
- Patterns can use literals, ranges (`..=`), alternatives (`|`), wildcards (`_`, `..`), and can destructure tuples, structs, enums and nested values.
- Guards (`if`) add conditions; `name @ pattern` keeps the value while testing it.
- Slice patterns (`[first, .., last]`, `[cmd, rest @ ..]`) are ideal for parsing.
- Matching a reference gives reference bindings; in edition 2024 you must not write `ref` where this already happens.
- `matches!` tests a pattern as a `bool`. A `match` must be exhaustive, and arms are tried from top to bottom.

## Sources and further reading

- [The Rust Programming Language — 19 Patterns and Matching](https://doc.rust-lang.org/book/ch19-00-patterns.html) — the chapter overview.
- [19.1 All the Places Patterns Can Be Used](https://doc.rust-lang.org/book/ch19-01-all-the-places-for-patterns.html) — every place a pattern is allowed.
- [19.2 Refutability](https://doc.rust-lang.org/book/ch19-02-refutability.html) — refutable and irrefutable patterns.
- [19.3 Pattern Syntax](https://doc.rust-lang.org/book/ch19-03-pattern-syntax.html) — the full syntax with examples.
- [The Rust Reference — Patterns](https://doc.rust-lang.org/reference/patterns.html) — exact rules, including binding modes and the edition 2024 changes.
- [std::matches! macro](https://doc.rust-lang.org/std/macro.matches.html) — documentation of `matches!`.

---

[🏠 Start](../START-HERE.md) · [← 06.2 Advanced traits and types](02-advanced-traits-and-types.md) · [📋 Phase 06](README.md) · [06.4 Interior mutability →](04-interior-mutability.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
