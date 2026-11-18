<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 07.6 How hash tables work](06-hash-tables.md) · [📋 Phase 07](README.md) · [07.8 Simple sorting algorithms →](08-simple-sorting-algorithms.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/07-algorithms-and-data-structures/07-recursion.md) <!-- nav -->

# Lesson 07.7 — Recursion and the call stack

> **Phase 07 — Algorithms and data structures** · Lesson 7 of 20 · ⏱ about 135 minutes

## What you will learn

- What recursion is: a function that calls itself on a smaller problem, with a base case that stops it.
- What happens on the call stack during a recursive call, and what a stack overflow is.
- How to see the cost of a recursive algorithm: depth (stack space), number of calls (time), and why naive Fibonacci is exponential.
- Divide and conquer: halving a problem keeps the depth small.
- How to turn recursion into a loop with your own stack.

## Before you start

- You finished [Lesson 07.6 — How hash tables work](06-hash-tables.md).
- You know the call stack and stack frames ([Lesson 03.1](../03-ownership-and-borrowing/01-memory-stack-and-heap.md), where a stack overflow was shown), recursive types with `Box` ([Lesson 05.9](../05-rust-intermediate/09-smart-pointers.md)), and the stack of Lesson 07.5.

## New words

| Word | Meaning |
|---|---|
| **recursion** | A function calling itself. Each call solves a smaller version of the same problem. |
| **base case** | The input small enough to answer directly, without another call. It stops the recursion. |
| **recursive case** | The part that reduces the problem and calls the function again. |
| **call stack** | The stack of frames of the function calls that are running now (Lesson 03.1). A call adds a frame; a return removes it. |
| **recursion depth** | How many calls are waiting on the stack at the same time. It is the space cost of a recursion. |
| **divide and conquer** | A method: split a problem into parts (often two halves), solve each recursively, and combine the answers. |
| **tail call** | A call that is the very last thing a function does. Some languages reuse the frame for it; Rust does not promise to. |
| **exponential** | Growing like `2ⁿ`: each extra step of `n` doubles the work (Lesson 07.1). |

## 1. A function that calls itself

The factorial `n!` is `n * (n-1) * ... * 1`. Notice that `n! = n * (n-1)!`. This is recursion: **the answer for `n` is made from the answer for a smaller `n`**. Every recursive function needs two parts:

1. a **base case** — an input small enough to answer at once (`0!` and `1!` are `1`);
2. a **recursive case** — a call with a *smaller* input, so that the calls come closer to the base case.

Without a base case the function calls itself forever. Without a smaller input it also never finishes. Create `src/recursion.rs`, add `pub mod recursion;` to `src/lib.rs`, and write the first function:

```rust
/// n! = n * (n-1) * ... * 1. Returns `None` if the result does not fit in a `u64`.
/// Base case: n <= 1. Recursive case: n * (n-1)!.
pub fn factorial(n: u64) -> Option<u64> {
    if n <= 1 {
        Some(1)
    } else {
        factorial(n - 1).and_then(|smaller| smaller.checked_mul(n))
    }
}
```

**What happens:** if `n <= 1`, the answer is `1` (the base case). Otherwise `factorial(n - 1)` gives `(n-1)!`, and `and_then` multiplies it by `n` with `checked_mul`. A result too large for a `u64` becomes `None` instead of an overflow (Lesson 07.2), and `None` travels back up through all the calls.

### What the computer does: the call stack

`examples/e07_trace.rs` prints a line when a call starts and when it returns, indented by the depth:

```rust
fn factorial(n: u64, depth: usize) -> u64 {
    let indent = "  ".repeat(depth);
    println!("{indent}factorial({n}) called");
    let result = if n <= 1 {
        1
    } else {
        n * factorial(n - 1, depth + 1)
    };
    println!("{indent}factorial({n}) returns {result}");
    result
}

fn main() {
    factorial(4, 0);
}
```

```text
factorial(4) called
  factorial(3) called
    factorial(2) called
      factorial(1) called
      factorial(1) returns 1
    factorial(2) returns 2
  factorial(3) returns 6
factorial(4) returns 24
```

Read the output as a picture of the **call stack** (Lesson 03.1). `factorial(4)` cannot finish before `factorial(3)` finishes, which waits for `factorial(2)`, which waits for `factorial(1)`. Four frames are on the stack at the deepest moment; then the calls return one by one, and each multiplies:

```text
 time ─────────────────────────────────────────────────▶
 stack  [f4] [f4 f3] [f4 f3 f2] [f4 f3 f2 f1] [f4 f3 f2] [f4 f3] [f4] []
         call  call    call       call (base)    return    return return return
                                  returns 1      2*1       3*2    4*6
```

The **depth** is 4 for `factorial(4)`, and in general `n`. Each frame uses some bytes of stack (the parameters, the local variables, the return address). So the space cost of a recursion is **its depth times the size of a frame**.

### The stack has a limit

The stack is small (about 8 MiB for the main thread of a program on macOS, less for other threads). A recursion that goes too deep runs out of it. `examples/e07_overflow.rs`:

```rust
fn depth(n: u64) -> u64 {
    if n == 0 { 0 } else { 1 + depth(n - 1) }
}

fn main() {
    println!("depth(1000) = {}", depth(1_000));
    println!("depth(100000000) = {}", depth(100_000_000));
}
```

```text
depth(1000) = 1000

thread 'main' (5430643) has overflowed its stack
fatal runtime error: stack overflow, aborting
```

`depth(1000)` is fine. A hundred million nested calls is not: the program is stopped by the system (a stack overflow, as in Lesson 03.1). It is not a panic: you cannot catch it.

> ⚠️ **Common mistake:** Do not rely on the compiler to remove the recursion. In a *release* build (`cargo run --release --example e07_overflow`) the optimizer happens to turn this particular function into a loop, and the program prints the answer. Rust does **not guarantee** that
> it optimizes tail calls, so a program that works in release mode may crash in debug mode (and the other way around is possible too). If the depth can be large or comes from outside, do not use recursion, or limit the depth.

> 🔒 **Security:** A recursive function that follows the shape of external input (nested JSON, JavaScript Object Notation, a common text format for data; a deeply nested expression, a long chain of links) can be crashed by an attacker who sends deep nesting. Limit the depth, or use a loop and your own stack (section 4).

## 2. Recursion that is cheap

**Euclid's algorithm** finds the greatest common divisor (the largest number that divides both). The idea: `gcd(a, b)` equals `gcd(b, a % b)`, and `gcd(a, 0)` is `a`.

```rust
/// Greatest common divisor (Euclid's algorithm): gcd(a, 0) = a, gcd(a, b) = gcd(b, a % b).
pub fn gcd(a: u64, b: u64) -> u64 {
    if b == 0 { a } else { gcd(b, a % b) }
}
```

For `gcd(48, 18)`: `gcd(18, 12)`, `gcd(12, 6)`, `gcd(6, 0)`, so the answer is 6. The remainder shrinks quickly: at least by half every two steps, so the depth is `O(log n)`.

Now two ways to sum a slice. Both are `O(n)` in time. They differ in depth.

```rust
/// Sums a slice as "first item + sum of the rest": the recursion is n calls deep.
pub fn sum_recursive(items: &[i64]) -> i64 {
    match items.split_first() {
        None => 0,
        Some((first, rest)) => first + sum_recursive(rest),
    }
}

/// Sums a slice as "sum of the left half + sum of the right half": only about log n calls deep.
pub fn sum_halves(items: &[i64]) -> i64 {
    match items.len() {
        0 => 0,
        1 => items[0],
        n => {
            let (left, right) = items.split_at(n / 2);
            sum_halves(left) + sum_halves(right)
        }
    }
}
```

**What happens:**

1. `sum_recursive` cuts off the first item (`split_first`) and adds it to the sum of the rest. The slice shrinks by 1 each time, so the depth is **`n`**. For a slice of a million items it would use a million frames.
2. `sum_halves` is **divide and conquer**: it cuts the slice into two halves (`split_at`), sums each half recursively, and adds the two results. The slice halves each time, so the depth is only **`log n`**: about 20 for a million items. The total number of calls is still about `2n`, so the time is `O(n)`.

```text
sum_halves([1,2,3,4,5,6,7,8])
                 ┌───────── [1..8] ─────────┐
           [1,2,3,4]                    [5,6,7,8]
          [1,2]   [3,4]                [5,6]   [7,8]
         [1] [2] [3] [4]              [5] [6] [7] [8]       depth: 4 = log₂ 8 + 1
```

The lesson: when a problem can be split in halves, the recursion depth is `O(log n)`. This idea powers merge sort and quicksort (Lesson 07.9) and binary search (Lesson 07.10).

## 3. Recursion that is expensive: Fibonacci

The Fibonacci numbers are `0, 1, 1, 2, 3, 5, 8, ...`: each is the sum of the two before it. The definition is recursive, and it is tempting to copy it directly:

```rust
/// The n-th Fibonacci number (0, 1, 1, 2, 3, 5, 8, ...), straight from the definition.
pub fn fib_recursive(n: u32) -> u64 {
    if n < 2 {
        n as u64
    } else {
        fib_recursive(n - 1) + fib_recursive(n - 2)
    }
}
```

It is correct, and terribly slow. To see why, count the calls:

```rust
/// How many calls `fib_recursive(n)` makes (including the first call).
pub fn fib_calls(n: u32) -> u64 {
    if n < 2 {
        1
    } else {
        1 + fib_calls(n - 1) + fib_calls(n - 2)
    }
}
```

```rust
/// The same numbers with a loop: n steps and no recursion.
pub fn fib_iterative(n: u32) -> u64 {
    let (mut current, mut next) = (0u64, 1u64);
    for _ in 0..n {
        (current, next) = (next, current + next);
    }
    current
}
```

`fib_calls` has the same shape as `fib_recursive` but counts calls instead of adding numbers. `fib_iterative` computes the same numbers with a loop that keeps the last two values. `examples/e07_calls.rs` prints both numbers:

```rust
use p07_algorithms::recursion::{fib_calls, fib_iterative, hanoi};

fn main() {
    println!("{:>4} {:>12} {:>14}", "n", "fib(n)", "calls");
    for n in [10, 20, 30, 35] {
        println!("{:>4} {:>12} {:>14}", n, fib_iterative(n), fib_calls(n));
    }

    println!();
    let mut moves = Vec::new();
    hanoi(3, 'A', 'C', 'B', &mut moves);
    println!("hanoi(3): {moves:?}");
    for n in [10, 20] {
        let mut moves = Vec::new();
        hanoi(n, 'A', 'C', 'B', &mut moves);
        println!("hanoi({n}) needs {} moves", moves.len());
    }
}
```

```text
   n       fib(n)          calls
  10           55            177
  20         6765          21891
  30       832040        2692537
  35      9227465       29860703

hanoi(3): [('A', 'C'), ('A', 'B'), ('C', 'B'), ('A', 'C'), ('B', 'A'), ('B', 'C'), ('A', 'C')]
hanoi(10) needs 1023 moves
hanoi(20) needs 1048575 moves
```

(We will come back to the second half of the output in a moment.) `fib(35)` is only 9 million, but it takes **almost 30 million calls**. The calls multiply: `fib(n)` calls `fib(n-1)` and `fib(n-2)`, and those call the same smaller ones again and again:

```text
                     fib(5)
                 ┌──────┴──────┐
              fib(4)         fib(3)           fib(3) is computed twice,
           ┌─────┴─────┐    ┌───┴───┐         fib(2) three times, ...
        fib(3)       fib(2) fib(2) fib(1)
       ┌──┴──┐
    fib(2) fib(1)
```

The number of calls grows by a factor of about 1.6 for each step of `n` (the golden ratio): **exponential**, about `O(1.6ⁿ)`. The recursion depth is small (`n`), but the time is huge. This pattern, a recursion that solves the same smaller problem many times, is exactly what **memoization** and dynamic programming fix (Lesson 07.16). The loop version is `O(n)` time and `O(1)` space.

### Towers of Hanoi

A classic that shows how recursion lets you solve a hard-looking problem by trusting the smaller one. Three pegs, `n` disks of different sizes on the first peg. Move all disks to another peg, one at a time, never placing a bigger disk on a smaller one. Recursive thinking: to move `n` disks, (1) move the `n-1` smaller disks out of the way to the helper peg, (2) move the biggest disk, (3) move the `n-1` disks onto it.

```rust
/// Towers of Hanoi: moves `n` disks from peg `from` to peg `to`, using peg `via` as a helper.
/// Every move is recorded as (from, to). The number of moves is 2^n - 1.
pub fn hanoi(n: u32, from: char, to: char, via: char, moves: &mut Vec<(char, char)>) {
    if n == 0 {
        return;
    }
    hanoi(n - 1, from, via, to, moves); // 1. move the n-1 smaller disks out of the way
    moves.push((from, to)); // 2. move the biggest disk
    hanoi(n - 1, via, to, from, moves); // 3. put the n-1 smaller disks back on top of it
}
```

The `moves` vector records each step. The second half of the output above shows the moves for 3 disks, `hanoi(3)`: 7 moves. The count follows `moves(n) = 2 * moves(n-1) + 1`, which gives `2ⁿ - 1`: 1,023 moves for 10 disks and **1,048,575 for 20**. This is an inherently exponential problem: no cleverness can make it faster, because the answer itself has `2ⁿ - 1` steps.

## 4. From recursion to a loop

Any recursion can become a loop. For `fib` a loop was natural. For a recursion over a *nested* structure, you keep your own **stack** of the work still to do (Lesson 07.5), instead of using the call stack:

```rust
/// A number or a list of nested things: `[1, [2, 3], [[4], 5]]`.
pub enum Nested {
    Number(i64),
    List(Vec<Nested>),
}

/// The sum of all numbers, by recursion. The depth of the recursion is the depth of the nesting.
pub fn sum_nested(item: &Nested) -> i64 {
    match item {
        Nested::Number(n) => *n,
        Nested::List(items) => items.iter().map(sum_nested).sum(),
    }
}

/// The same, with our own stack instead of the call stack.
pub fn sum_nested_with_stack(item: &Nested) -> i64 {
    let mut to_visit = vec![item];
    let mut total = 0;
    while let Some(current) = to_visit.pop() {
        match current {
            Nested::Number(n) => total += n,
            Nested::List(items) => to_visit.extend(items.iter()),
        }
    }
    total
}
```

**What happens:**

1. `Nested` is a number or a list of `Nested` values: a recursive type (Lesson 05.9; the `Vec` provides the indirection).
2. `sum_nested` follows the type: a number gives itself, a list sums its items recursively. Its depth equals the nesting depth.
3. `sum_nested_with_stack` uses a `Vec` as a stack: pop an item; if it is a number, add it; if it is a list, push all its items. The stack lives on the **heap** and grows as long as memory lasts, so deep nesting cannot overflow the call stack. The order of visiting is different, but a sum does not care.

Which one should you use? Recursion when the depth is small and known (a balanced tree, halving) and the code becomes clearer. A loop with your own stack when the depth can be large or comes from outside input.

## 5. Tests

Add at the bottom of `src/recursion.rs`:

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use crate::testutil::Rng;

    #[test]
    fn factorial_values_and_overflow() {
        assert_eq!(factorial(0), Some(1));
        assert_eq!(factorial(5), Some(120));
        assert_eq!(factorial(20), Some(2_432_902_008_176_640_000));
        assert_eq!(factorial(21), None); // 21! does not fit into a u64
    }

    #[test]
    fn gcd_examples() {
        assert_eq!(gcd(48, 18), 6);
        assert_eq!(gcd(17, 5), 1);
        assert_eq!(gcd(0, 9), 9);
        assert_eq!(gcd(9, 0), 9);
    }

    #[test]
    fn both_sums_match_the_iterator_sum() {
        let mut rng = Rng::new(4);
        for _ in 0..200 {
            let items: Vec<i64> = (0..rng.below(50))
                .map(|_| rng.below(1000) as i64 - 500)
                .collect();
            let expected: i64 = items.iter().sum();
            assert_eq!(sum_recursive(&items), expected);
            assert_eq!(sum_halves(&items), expected);
        }
    }

    #[test]
    fn fibonacci_versions_agree_and_calls_grow_like_fib() {
        for n in 0..=25 {
            assert_eq!(fib_recursive(n), fib_iterative(n), "n = {n}");
            // the number of calls is 2 * fib(n + 1) - 1
            assert_eq!(fib_calls(n), 2 * fib_iterative(n + 1) - 1, "n = {n}");
        }
        assert_eq!(fib_iterative(50), 12_586_269_025);
    }

    #[test]
    fn hanoi_makes_legal_moves_and_two_to_the_n_minus_one_of_them() {
        for n in 0..=10u32 {
            let mut moves = Vec::new();
            hanoi(n, 'A', 'C', 'B', &mut moves);
            assert_eq!(moves.len(), (1usize << n) - 1);

            // play the moves on three stacks of disks; a bigger disk must never lie on a smaller one
            let mut pegs: [Vec<u32>; 3] = [(1..=n).rev().collect(), Vec::new(), Vec::new()];
            let index = |peg: char| (peg as u8 - b'A') as usize;
            for (from, to) in moves {
                let disk = pegs[index(from)].pop().expect("the source peg has a disk");
                assert!(pegs[index(to)].last().is_none_or(|top| *top > disk));
                pegs[index(to)].push(disk);
            }
            assert_eq!(pegs[2].len(), n as usize); // all disks are on peg C
        }
    }

    fn random_nested(rng: &mut Rng, depth: u32) -> Nested {
        if depth == 0 || rng.below(3) == 0 {
            Nested::Number(rng.below(100) as i64)
        } else {
            let count = rng.below(4);
            Nested::List((0..count).map(|_| random_nested(rng, depth - 1)).collect())
        }
    }

    #[test]
    fn nested_sums_agree() {
        let mut rng = Rng::new(12);
        for _ in 0..300 {
            let tree = random_nested(&mut rng, 6);
            assert_eq!(sum_nested(&tree), sum_nested_with_stack(&tree));
        }
        let fixed = Nested::List(vec![
            Nested::Number(1),
            Nested::List(vec![Nested::Number(2), Nested::Number(3)]),
            Nested::List(vec![
                Nested::List(vec![Nested::Number(4)]),
                Nested::Number(5),
            ]),
        ]);
        assert_eq!(sum_nested(&fixed), 15);
        assert_eq!(sum_nested_with_stack(&fixed), 15);
    }
}
```

**What happens:**

1. The factorial test includes the overflow limit: `20!` fits into a `u64`, `21!` does not.
2. `both_sums_match_the_iterator_sum` compares both recursive sums with `iter().sum()` on 200 random slices.
3. The Fibonacci test checks that the recursive and the loop version agree for `n` from 0 to 25, and that the number of calls follows the formula `2 * fib(n + 1) - 1`. It is a good example of testing an *analysis* result, not only the answer.
4. The Hanoi test checks the count `2ⁿ - 1` and then **plays the moves** on three stacks, checking that no bigger disk is put on a smaller one and that all disks end on peg `C`. A test that simulates is stronger than one that only counts.
5. The last test compares the recursive and the stack-based sum on 300 random nested structures.

```text
running 6 tests
test recursion::tests::gcd_examples ... ok
test recursion::tests::factorial_values_and_overflow ... ok
test recursion::tests::hanoi_makes_legal_moves_and_two_to_the_n_minus_one_of_them ... ok
test recursion::tests::both_sums_match_the_iterator_sum ... ok
test recursion::tests::nested_sums_agree ... ok
test recursion::tests::fibonacci_versions_agree_and_calls_grow_like_fib ... ok

test result: ok. 6 passed; 0 failed; 0 ignored; 0 measured; 20 filtered out; finished in 0.00s
```

## 6. Measuring

`benches/recursion.rs` compares the two Fibonacci versions at `n = 30`. Register it in `Cargo.toml` like in Lesson 07.2 (`name = "recursion"`, `harness = false`).

```rust
use criterion::{Criterion, criterion_group, criterion_main};
use p07_algorithms::recursion::{fib_iterative, fib_recursive};
use std::hint::black_box;

fn bench_fib(c: &mut Criterion) {
    let mut group = c.benchmark_group("fibonacci(30)");
    group.bench_function("recursive", |b| b.iter(|| fib_recursive(black_box(30))));
    group.bench_function("loop", |b| b.iter(|| fib_iterative(black_box(30))));
    group.finish();
}

criterion_group!(benches, bench_fib);
criterion_main!(benches);
```

Run `cargo bench --bench recursion -- --warm-up-time 1 --measurement-time 3`:

```text
fibonacci(30)/recursive time:   [1.4337 ms 1.4422 ms 1.4523 ms]
fibonacci(30)/loop      time:   [8.5653 ns 8.6005 ns 8.6363 ns]
```

(Apple M4 Pro; **your numbers will differ**.) The recursive version takes 1.44 ms (2.7 million calls); the loop takes 8.6 ns. The loop is about **170,000 times faster**, for the same answer. Big-O predicted the direction (`1.6ⁿ` against `n`); the benchmark shows the size.

## 7. When to use recursion

- **Good fit:** the data is recursive (trees, nested lists, expressions, directories) and shallow or balanced; the problem splits naturally (divide and conquer); backtracking (try, undo, try again).
- **Bad fit:** the depth is as large as `n` for big `n` (use a loop); the same subproblems repeat (use memoization, Lesson 07.16, or a loop).
- **Always:** know the base case, make sure every call gets closer to it, and know the depth.

## Common mistakes

**1. No base case, or a call that does not get smaller.** `fn f(n: u64) -> u64 { f(n - 1) }` never stops (and `n - 1` on 0 panics in a debug build before the stack overflows).

**2. Recomputing the same thing** (naive Fibonacci). If a recursion calls itself twice on overlapping problems, count the calls before trusting it.

**3. Deep recursion on input you do not control.** See the security note in section 1.

## Try it yourself

1. Type `recursion.rs` yourself. Run `cargo test recursion`, `cargo clippy` and `cargo fmt`.
2. Run `e07_trace` for `factorial(6)`. Predict the maximum indentation before you run it.
3. Run `e07_calls` with `fib_calls(40)`. How long does it take? (Do not go much higher.)
4. Run `e07_overflow` in debug and in release mode. What differs?
5. Commit: `git add code/p07-algorithms && git commit -m "feat(p07): recursion examples"`.

## Exercises

### Exercise 1 — Time and depth (easy)

For each function with input size `n`, give the time and the maximum recursion depth: (a) `factorial(n)`; (b) `sum_recursive` on a slice of `n` items; (c) `sum_halves`; (d) `fib_recursive(n)`; (e) `hanoi(n, ..)`.

<details>
<summary>Hint</summary>

The depth is the longest chain of calls waiting for each other. The time is the total number of calls (times the work per call).

</details>

<details>
<summary>Answer</summary>

(a) time `O(n)`, depth `n`. (b) time `O(n)`, depth `n`. (c) time `O(n)`, depth about `log n`. (d) time exponential, about `O(1.6ⁿ)`; depth `n`. (e) time `O(2ⁿ)`, depth `n`.

</details>

### Exercise 2 — Binary digits (medium)

Write `to_binary(n: u32) -> String` by recursion: the binary digits of `n / 2`, followed by the last digit of `n`. Write a test that compares it with `format!("{n:b}")` for all `n` below 2,000 and for `u32::MAX`. What are its time and depth?

<details>
<summary>Hint</summary>

The base case is `n < 2`: the text is just `"0"` or `"1"`. Otherwise recurse on `n / 2` and add `'0'` or `'1'` at the end.

</details>

<details>
<summary>Answer</summary>

```rust
/// The binary digits of `n` as text, by recursion: the digits of `n / 2`, then the last digit.
pub fn to_binary(n: u32) -> String {
    if n < 2 {
        n.to_string()
    } else {
        let mut digits = to_binary(n / 2);
        digits.push(if n.is_multiple_of(2) { '0' } else { '1' });
        digits
    }
}
```

```rust
    #[test]
    fn to_binary_matches_the_standard_formatting() {
        for n in 0..2000u32 {
            assert_eq!(to_binary(n), format!("{n:b}"));
        }
        assert_eq!(to_binary(u32::MAX), "1".repeat(32));
    }
```

`n` is halved at each call, so the depth is about `log₂ n` (at most 32 for a `u32`), and the time is `O(log n)` calls (the string building adds a small cost per digit).

</details>

### Exercise 3 — All subsets (hard)

Write `subsets<T: Clone>(items: &[T]) -> Vec<Vec<T>>` that returns all `2ⁿ` subsets of the items (including the empty one) by recursion on the first item. Test that the count is `2ⁿ` and that all subsets are different for `n` from 0 to 10. State the time and the depth.

<details>
<summary>Hint</summary>

Every subset either does not contain the first item (these are the subsets of the rest) or contains it (these are the subsets of the rest, each with the first item added). Base case: the empty slice has one subset, the empty one.

</details>

<details>
<summary>Answer</summary>

```rust
/// Every subset of `items` (2^n of them), by recursion on the first item:
/// the subsets without it, and the same subsets with it added at the front.
pub fn subsets<T: Clone>(items: &[T]) -> Vec<Vec<T>> {
    match items.split_first() {
        None => vec![Vec::new()],
        Some((first, rest)) => {
            let without = subsets(rest);
            let mut result = without.clone();
            for mut subset in without {
                subset.insert(0, first.clone());
                result.push(subset);
            }
            result
        }
    }
}
```

```rust
    #[test]
    fn subsets_has_two_to_the_n_distinct_subsets() {
        for n in 0..=10usize {
            let items: Vec<usize> = (0..n).collect();
            let all = subsets(&items);
            assert_eq!(all.len(), 1 << n);
            let distinct: HashSet<Vec<usize>> = all.into_iter().collect();
            assert_eq!(distinct.len(), 1 << n);
        }
    }
```

The depth is `n`. The time is `O(n * 2ⁿ)`: there are `2ⁿ` subsets with up to `n` items each. As with Hanoi, the output itself is exponentially big, so the exponential time is unavoidable. (The test reuses `HashSet` from Lesson 07.2.)

</details>

## Quiz

1. What two parts must every recursive function have?

<details>
<summary>Answer</summary>

A base case that returns without a further call, and a recursive case that calls the function with a smaller input, so the calls approach the base case.

</details>

2. What is the recursion depth of `sum_recursive` and of `sum_halves` on one million items?

<details>
<summary>Answer</summary>

About one million for `sum_recursive` (one frame per item, so it is likely to overflow the stack) and about 20 for `sum_halves` (the slice halves each call).

</details>

3. Why does naive `fib_recursive(40)` take so long, though its depth is only 40?

<details>
<summary>Answer</summary>

It makes about 330 million calls: each call makes two more and they repeat the same smaller subproblems. The time is exponential even though the stack depth is small.

</details>

4. Your program reads nested data from the network and processes it recursively. What can go wrong, and what do you do?

<details>
<summary>Answer</summary>

An attacker can send very deep nesting and overflow the stack, which aborts the process. Limit the depth (and return an error), or process the data with a loop and your own stack on the heap.

</details>

## Summary

- Recursion: a base case, plus a recursive case with a smaller input. The call stack holds one frame per waiting call; the depth is the space cost.
- The stack is small. Deep recursion overflows it, and Rust does not promise tail-call optimization, so do not count on the optimizer.
- Halving (divide and conquer) gives depth `O(log n)`; peeling one item gives depth `O(n)`.
- A recursion that solves overlapping subproblems (naive Fibonacci) is exponential; a loop or memoization fixes it.
- Some problems are exponential by nature (Hanoi, all subsets): the output itself is that big.
- Any recursion can be turned into a loop with your own stack on the heap, which is safe for deep or hostile input.
- Test recursive code against a simple reference, test the base case, and test the counts that your analysis predicts.

## Sources and further reading

- [The Rust Programming Language — 3.5 Control Flow](https://doc.rust-lang.org/book/ch03-05-control-flow.html) — loops, as the alternative to recursion.
- [std::primitive::slice — split_first and split_at](https://doc.rust-lang.org/std/primitive.slice.html#method.split_first) — the two slice methods used for the recursion on slices.
- [Option::and_then](https://doc.rust-lang.org/std/option/enum.Option.html#method.and_then) — chaining steps that can each fail with `None`.
- [The Rust Programming Language — 15.1 Using Box<T> to Point to Data on the Heap](https://doc.rust-lang.org/book/ch15-01-box.html) — recursive types, as used by `Nested`.
- [std::thread::Builder::stack_size](https://doc.rust-lang.org/std/thread/struct.Builder.html#method.stack_size) — how to give a thread a bigger stack, when you really need deep recursion.

---

[🏠 Start](../START-HERE.md) · [← 07.6 How hash tables work](06-hash-tables.md) · [📋 Phase 07](README.md) · [07.8 Simple sorting algorithms →](08-simple-sorting-algorithms.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
