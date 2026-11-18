<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 07.15 Two pointers and sliding window](15-two-pointers-and-sliding-window.md) · [📋 Phase 07](README.md) · [07.17 Greedy algorithms →](17-greedy-algorithms.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/07-algorithms-and-data-structures/16-dynamic-programming.md) <!-- nav -->

# Lesson 07.16 — Dynamic programming

> **Phase 07 — Algorithms and data structures** · Lesson 16 of 20 · ⏱ about 2.5 hours (2 sittings)

## What you will learn

- Why a plain recursive solution can be absurdly slow, and how to count the repeated work.
- Two cures: *memoization* (top-down) and a *table* (bottom-up), and how to shrink the table.
- A recipe for dynamic programming: state, recurrence, base cases, order, answer.
- Two classic problems: the fewest coins, and the longest common subsequence.
- How to test a dynamic-programming solution against a brute-force one.

## Before you start

- You finished [Lesson 07.15 — Two pointers and sliding window](15-two-pointers-and-sliding-window.md).
- You know recursion, base cases and the call stack ([Lesson 07.7](07-recursion.md)) and vectors ([Lesson 07.3](03-arrays-and-vectors-under-the-hood.md)).
- You work in `code/p07-algorithms`. This lesson adds the module `src/dp.rs` (and `pub mod dp;` in `src/lib.rs`) and one benchmark.

## New words

| Word | Meaning |
|---|---|
| **dynamic programming** (DP) | Solving a problem by solving its smaller versions once, saving the answers, and combining them. ("Programming" here means "filling a table", not writing code.) |
| **subproblem** | A smaller version of the same problem, for example `fib(3)` inside `fib(5)`. |
| **overlapping subproblems** | The same subproblem is needed again and again by different parts of the solution. This is what makes a plain recursion slow and DP useful. |
| **optimal substructure** | The best answer for a problem is built from the best answers of its subproblems. |
| **memoization** (memo) | Saving the result of a call in a notebook (a vector or map), so the next call with the same input only looks it up. Also called *top-down* DP. |
| **table** (bottom-up) | Filling an array of answers from the smallest subproblem to the largest, in an order where everything you need is already filled. |
| **state** | The few numbers that completely describe a subproblem, for example "the amount `a`". The table has one cell per state. |
| **recurrence** | The formula that gives the answer of a state from the answers of smaller states, for example `fib(n) = fib(n-1) + fib(n-2)`. |
| **subsequence** | Some characters of a text, in the same order, not necessarily next to each other. `"AC"` is a subsequence of `"ABC"`. |
| **LCS** (Longest Common Subsequence) | The longest sequence that is a subsequence of both of two texts. |

## 1. Repeated work: the cost of a plain recursion

The Fibonacci numbers start with 0 and 1, and every next number is the sum of the previous two: 0, 1, 1, 2, 3, 5, 8, 13, … A direct translation of that rule into Rust, followed by a version of it that counts its own calls. Create `src/dp.rs`:

```rust
/// Fibonacci by plain recursion. Time O(2^n), very slow.
pub fn fib_naive(n: u32) -> u64 {
    if n < 2 {
        u64::from(n)
    } else {
        fib_naive(n - 1) + fib_naive(n - 2)
    }
}

/// The same recursion, but it counts how many calls were made.
pub fn fib_counting(n: u32, calls: &mut u64) -> u64 {
    *calls += 1;
    if n < 2 {
        u64::from(n)
    } else {
        fib_counting(n - 1, calls) + fib_counting(n - 2, calls)
    }
}
```

Look at the calls for `fib(5)`. Each box is one call; the same call appears again and again:

```text
                          fib(5)
                  /                  \
             fib(4)                  fib(3)             fib(3) is computed 2 times
            /      \                /      \            fib(2) is computed 3 times
        fib(3)    fib(2)        fib(2)    fib(1)
        /    \    /    \        /    \
    fib(2) fib(1) fib(1) fib(0) fib(1) fib(0)
    /    \
 fib(1) fib(0)                                          15 calls for n = 5
```

Create `examples/e16_calls.rs` and run it in release mode (the optimized build from [Lesson 05.12](../05-rust-intermediate/12-benchmarking-basics.md)):

```rust
use p07_algorithms::dp::{fib_counting, fib_memo, fib_two_vars};
use std::time::Instant;

fn main() {
    for n in [5, 10, 20, 30] {
        let mut calls = 0;
        let value = fib_counting(n, &mut calls);
        println!("fib({n}) = {value}, recursive calls: {calls}");
    }
    let start = Instant::now();
    let mut calls = 0;
    let value = fib_counting(40, &mut calls);
    println!(
        "fib(40) = {value}, recursive calls: {calls}, took {:?}",
        start.elapsed()
    );
    println!("fib_memo(90) = {:?}", fib_memo(90));
    println!("fib_two_vars(93) = {:?}", fib_two_vars(93));
    println!("fib_two_vars(94) = {:?}", fib_two_vars(94));
}
```

(The functions `fib_memo` and `fib_two_vars` come in Section 2; add them first, or comment those three lines out.)

```bash
cargo run -q --release --example e16_calls
```

```text
fib(5) = 5, recursive calls: 15
fib(10) = 55, recursive calls: 177
fib(20) = 6765, recursive calls: 21891
fib(30) = 832040, recursive calls: 2692537
fib(40) = 102334155, recursive calls: 331160281, took 382.728166ms
fib_memo(90) = Some(2880067194370816120)
fib_two_vars(93) = Some(12200160415121876738)
fib_two_vars(94) = None
```

**What happens:**

1. The number of calls grows by about 1.6 times each time `n` grows by 1 (177 → 21,891 → 2,692,537 for n = 10, 20, 30). This is *exponential* growth, O(2ⁿ) (strictly about 1.6ⁿ). Asking for `fib(40)` makes 331 million calls and needs about 0.4 seconds on this Mac; `fib(100)` would need tens of thousands of years.
2. Almost all calls repeat work. There are only 41 *different* subproblems `fib(0)` to `fib(40)`. If each were solved once, 41 small steps would be enough.
3. The last two lines are a preview: the faster versions below compute `fib(90)` instantly. `fib(94)` does not fit into a `u64`, so we return `None` instead of panicking.

This situation, a recursion whose subproblems overlap, is the signal for dynamic programming. (When subproblems do **not** overlap, as in merge sort, recursion is already fine.)

## 2. Two cures: memo and table

**Cure 1: memoization (top-down).** Keep the recursion, but add a notebook. Before computing `fib(n)`, look in the notebook; after computing, write the result in it.

```rust
/// Top-down: recursion plus a memo (a notebook of answers already found).
/// Returns `None` when the result does not fit into `u64` (from n = 94 on).
pub fn fib_memo(n: usize) -> Option<u64> {
    let mut memo: Vec<Option<u64>> = vec![None; n + 1];
    fib_memo_rec(n, &mut memo)
}

fn fib_memo_rec(n: usize, memo: &mut Vec<Option<u64>>) -> Option<u64> {
    if n < 2 {
        return Some(n as u64);
    }
    if let Some(known) = memo[n] {
        return Some(known); // already solved: no new work
    }
    let result = fib_memo_rec(n - 1, memo)?.checked_add(fib_memo_rec(n - 2, memo)?)?;
    memo[n] = Some(result);
    Some(result)
}
```

**What happens:**

1. `memo[n]` is `None` while `fib(n)` is unknown and `Some(value)` after it is solved. The memo is passed down by `&mut` so that all calls share it.
2. The first time a state is needed it is computed (two recursive calls); every later time it is a single lookup. There are `n + 1` states and each is computed once: **O(n) time, O(n) space**.
3. `checked_add` returns `None` on overflow, and `?` passes that `None` upward. The function never panics ([Lesson 04.10](../04-structs-enums-collections-errors/10-error-handling.md)).

**Cure 2: a table (bottom-up).** Do not recurse at all. Fill an array from the smallest state up, so that when you compute `table[i]`, the cells it needs are already filled.

```rust
/// Bottom-up: fill the answers from small to large. Time O(n), space O(n).
pub fn fib_table(n: usize) -> Option<u64> {
    let mut table = vec![0u64; n.max(1) + 1];
    table[1] = 1;
    for i in 2..=n {
        table[i] = table[i - 1].checked_add(table[i - 2])?;
    }
    Some(table[n])
}
```

Bottom-up has no call stack: no risk of a stack overflow, and no function-call cost. Top-down is easier to write when you already have the recursive formula and when only some states are needed.

**Shrinking the table.** `table[i]` needs only the two cells before it, so the whole array is a waste. Keep two variables:

```rust
/// Bottom-up with only the last two answers. Time O(n), space O(1).
pub fn fib_two_vars(n: usize) -> Option<u64> {
    if n == 0 {
        return Some(0);
    }
    // `older` and `newer` are fib(i - 1) and fib(i); we start at i = 1
    let (mut older, mut newer) = (0u64, 1u64);
    for _ in 1..n {
        (older, newer) = (newer, older.checked_add(newer)?);
    }
    Some(newer)
}
```

> 📏 **Convention:** First get a correct DP with a full table. Reduce the memory only if it matters, and only after the tests pass.

> 🧪 **Testing:** The slow `fib_naive` is the judge for small `n`, and the overflow border is a classic edge case. Add to `src/dp.rs`:

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use crate::testutil::Rng;

    #[test]
    fn fib_small_values() {
        let expected = [0, 1, 1, 2, 3, 5, 8, 13, 21, 34, 55];
        for (n, &want) in expected.iter().enumerate() {
            assert_eq!(fib_naive(n as u32), want);
            assert_eq!(fib_memo(n), Some(want));
            assert_eq!(fib_table(n), Some(want));
            assert_eq!(fib_two_vars(n), Some(want));
        }
    }

    #[test]
    fn fib_versions_agree_and_overflow_is_detected() {
        for n in 0..=25 {
            assert_eq!(fib_naive(n as u32), fib_two_vars(n).unwrap());
        }
        assert_eq!(fib_two_vars(93), Some(12_200_160_415_121_876_738));
        assert_eq!(fib_memo(93), Some(12_200_160_415_121_876_738));
        assert_eq!(fib_table(93), Some(12_200_160_415_121_876_738));
        assert_eq!(fib_two_vars(94), None);
        assert_eq!(fib_memo(94), None);
        assert_eq!(fib_table(94), None);
    }
}
```

`fib(93)` = 12,200,160,415,121,876,738 is the largest Fibonacci number that fits into a `u64`; `fib(94)` does not. (Later tests of this lesson go into the same `tests` module.)

## 3. The recipe

Every DP solution answers five questions. Write the answers as comments before the code:

1. **State.** Which few numbers describe a subproblem? (Fibonacci: `n`.)
2. **Recurrence.** How does the answer of a state come from smaller states? (`fib(n) = fib(n-1) + fib(n-2)`.)
3. **Base cases.** Which states have an answer without a recurrence? (`fib(0) = 0`, `fib(1) = 1`.)
4. **Order.** In which order are the states filled so that each recurrence only reads filled cells? (Increasing `n`.)
5. **Answer.** Which cell holds the final answer? (`table[n]`.)

The running time is almost always **(number of states) × (work per state)**, and the memory is the number of states (or less, if the recurrence reads only a few rows back).

## 4. Example: the fewest coins

**The problem.** Coins of some values (unlimited number of each) and an `amount`. Find the **fewest** coins whose values add up exactly to the amount, or say it is impossible.

1. **State:** `best[a]` = the fewest coins for the amount `a` (or "impossible").
2. **Recurrence:** the last coin can be any coin `c` with `c <= a`; before it we needed `best[a - c]` coins. So `best[a] = 1 + min(best[a - c])` over all usable coins `c`.
3. **Base case:** `best[0] = 0` (zero coins make zero).
4. **Order:** `a` from 1 up to `amount`.
5. **Answer:** `best[amount]`.

```text
coins = [1, 3, 4], amount = 6

a        0  1  2  3  4  5  6
best[a]  0  1  2  1  1  2  2

best[6] = 1 + min( best[6-1] = best[5] = 2,     <- use coin 1
                   best[6-3] = best[3] = 1,     <- use coin 3
                   best[6-4] = best[2] = 2 )    <- use coin 4
        = 1 + 1 = 2       (3 + 3)
```

```rust
/// The fewest coins that add up to `amount`, or `None` if it is impossible.
/// Every coin value must be at least 1. Time O(amount * coins), space O(amount).
pub fn min_coins(coins: &[usize], amount: usize) -> Option<usize> {
    // best[a] = the fewest coins for the amount a (None = impossible)
    let mut best: Vec<Option<usize>> = vec![None; amount + 1];
    best[0] = Some(0);
    for a in 1..=amount {
        for &coin in coins {
            // `checked_sub` is None when the coin is bigger than the amount
            if let Some(before) = a.checked_sub(coin).and_then(|rest| best[rest]) {
                let candidate = before + 1;
                if best[a].is_none_or(|current| candidate < current) {
                    best[a] = Some(candidate);
                }
            }
        }
    }
    best[amount]
}
```

**What happens:**

1. `None` means "impossible". It is safer than a magic "infinity" number (see Common mistakes).
2. For each amount `a` and each coin, `a.checked_sub(coin)` gives the rest `a - coin`, or `None` when the coin is too big. `.and_then(|rest| best[rest])` then looks up the answer for the rest; it is `None` too if that rest is impossible.
3. `is_none_or` is `true` when `best[a]` is still `None` or when the new candidate is smaller than the current value.
4. Two loops: `amount` states times `coins.len()` coins, so **O(amount · coins) time**; one vector: **O(amount) space**.

> 💡 **Tip:** Try the greedy idea "always take the biggest coin that fits" on coins `[1, 3, 4]` and amount 6: it picks 4 + 1 + 1 = 3 coins, but the DP found 3 + 3 = 2. Greedy does not always work; DP always does. [Lesson 07.17](17-greedy-algorithms.md) shows when greedy is safe.

The judge is a brute-force recursion that tries every first coin, with no memory. It is exponential, which does not matter for amounts up to 14:

```rust
/// Tries every way (exponential time). Used only as a judge in tests.
pub fn min_coins_slow(coins: &[usize], amount: usize) -> Option<usize> {
    if amount == 0 {
        return Some(0);
    }
    coins
        .iter()
        .filter(|&&coin| coin <= amount)
        .filter_map(|&coin| min_coins_slow(coins, amount - coin))
        .map(|count| count + 1)
        .min()
}
```

```rust
    #[test]
    fn coins_examples() {
        assert_eq!(min_coins(&[1, 3, 4], 6), Some(2)); // 3 + 3
        assert_eq!(min_coins(&[2], 3), None);
        assert_eq!(min_coins(&[5, 2], 0), Some(0));
        assert_eq!(min_coins(&[], 5), None);
        assert_eq!(min_coins(&[1, 5, 10, 25], 63), Some(6)); // 25+25+10+1+1+1
    }

```

The random test uses `Rng` from `src/testutil.rs` with the `range` method that you added in Lesson 07.15 (the `use crate::testutil::Rng;` line is already at the top of the `tests` module). `range` returns an `i64`, so the counts and the amount are converted with `as usize`:

```rust
    #[test]
    fn coins_agree_with_the_slow_version() {
        let mut rng = Rng::new(2024);
        for _ in 0..500 {
            let count = rng.range(0, 4) as usize;
            let coins: Vec<usize> = (0..count).map(|_| rng.range(1, 7) as usize).collect();
            let amount = rng.range(0, 14) as usize;
            assert_eq!(
                min_coins(&coins, amount),
                min_coins_slow(&coins, amount),
                "coins = {coins:?}, amount = {amount}"
            );
        }
    }
```

The random test generates 0 to 4 coins of value 1 to 7 and amounts up to 14, so it covers "no coins", "impossible amounts" and duplicate coins.

## 5. A table with two dimensions: longest common subsequence

**The problem.** Find the length of the longest common subsequence (LCS) of two texts. For `"AGCAT"` and `"GAC"` it is 2 (for example `"AC"` or `"GC"`).

1. **State:** `table[i][j]` = the LCS length of the first `i` characters of `a` and the first `j` characters of `b`. Two numbers, so a two-dimensional table.
2. **Recurrence:** look at the last characters `a[i-1]` and `b[j-1]`. If they are equal, they extend the best answer of the shorter texts: `table[i-1][j-1] + 1`. If not, one of them is not used: take the better of `table[i-1][j]` and `table[i][j-1]`.
3. **Base cases:** with an empty text the answer is 0: row 0 and column 0 are zeros.
4. **Order:** rows top to bottom, each row left to right (every cell needs only cells above or to the left).
5. **Answer:** the bottom-right cell.

```text
            b:   ""  G  A  C
 a: ""            0  0  0  0
    A             0  0  1  1
    G             0  1  1  1
    C             0  1  1  2
    A             0  1  2  2
    T             0  1  2  2     <- the answer is the last cell: 2
```

```rust
/// The length of the longest common subsequence of two texts.
/// Time O(n * m), space O(n * m).
pub fn lcs_len(a: &str, b: &str) -> usize {
    let a: Vec<char> = a.chars().collect();
    let b: Vec<char> = b.chars().collect();
    // table[i][j] = answer for the first i characters of a and the first j of b
    let mut table = vec![vec![0usize; b.len() + 1]; a.len() + 1];
    for i in 1..=a.len() {
        for j in 1..=b.len() {
            table[i][j] = if a[i - 1] == b[j - 1] {
                table[i - 1][j - 1] + 1
            } else {
                table[i - 1][j].max(table[i][j - 1])
            };
        }
    }
    table[a.len()][b.len()]
}
```

**What happens:**

1. The texts become `Vec<char>` so that we can index characters (a `String` cannot be indexed by position, [Lesson 04.8](../04-structs-enums-collections-errors/08-strings-in-depth.md)).
2. `vec![vec![0; m + 1]; n + 1]` makes `n + 1` rows of `m + 1` zeros. Row 0 and column 0 are the base cases.
3. The cell in row `i` and column `j` talks about `a[i - 1]` and `b[j - 1]`, because the table has one extra row and column for the empty prefix. Forgetting this shift is the most common bug.
4. `n · m` states with O(1) work each: **O(n · m) time and space**. Since a row needs only the row above, the space can be reduced to two rows (Exercise 2).

The judge tries every subsequence of `a` (2ⁿ of them, selected by the bits of a number) and checks whether it is a subsequence of `b`:

```rust
    /// Judge for lcs_len: try every subsequence of `a` (2^n of them).
    fn lcs_slow(a: &str, b: &str) -> usize {
        let a: Vec<char> = a.chars().collect();
        let mut best = 0;
        for mask in 0u32..(1 << a.len()) {
            let picked: Vec<char> = (0..a.len())
                .filter(|&i| mask & (1 << i) != 0)
                .map(|i| a[i])
                .collect();
            let mut rest = b.chars();
            if picked.iter().all(|c| rest.any(|x| x == *c)) {
                best = best.max(picked.len());
            }
        }
        best
    }

    #[test]
    fn lcs_examples_and_judge() {
        assert_eq!(lcs_len("AGCAT", "GAC"), 2);
        assert_eq!(lcs_len("", "abc"), 0);
        assert_eq!(lcs_len("abc", "abc"), 3);
        assert_eq!(lcs_len("abc", "xyz"), 0);
        let mut rng = Rng::new(7);
        for _ in 0..300 {
            let make = |rng: &mut Rng| -> String {
                (0..rng.range(0, 8))
                    .map(|_| (b'a' + rng.range(0, 2) as u8) as char)
                    .collect()
            };
            let a = make(&mut rng);
            let b = make(&mut rng);
            assert_eq!(lcs_len(&a, &b), lcs_slow(&a, &b), "a = {a:?}, b = {b:?}");
        }
    }
```

In `lcs_slow`, `rest.any(...)` consumes the iterator `rest` as it searches, so each picked character is looked for *after* the previous one: that is exactly the subsequence rule. The texts use only the letters `a`, `b`, `c` (`b'a' + 0..=2`) so that common subsequences are frequent. Run `cargo test dp`:

```text
running 5 tests
test dp::tests::coins_examples ... ok
test dp::tests::fib_small_values ... ok
test dp::tests::fib_versions_agree_and_overflow_is_detected ... ok
test dp::tests::lcs_examples_and_judge ... ok
test dp::tests::coins_agree_with_the_slow_version ... ok

test result: ok. 5 passed; 0 failed; 0 ignored; 0 measured; 70 filtered out; finished in 0.04s
```

(The order of the lines can differ. The 70 filtered-out tests come from the main text of Lessons 07.1 to 07.15; with the tests of the exercises your number is bigger.)

## 6. Measuring the difference

One benchmark, in the style of [Lesson 05.12](../05-rust-intermediate/12-benchmarking-basics.md). Add to `Cargo.toml` (`criterion = "0.8"` must be in `[dev-dependencies]`):

```toml
[[bench]]
name = "dp"
harness = false
```

`benches/dp.rs`:

```rust
use criterion::{Criterion, criterion_group, criterion_main};
use p07_algorithms::dp::{fib_memo, fib_naive, fib_table, fib_two_vars};
use std::hint::black_box;

fn bench_fib(c: &mut Criterion) {
    let mut group = c.benchmark_group("fib 30");
    group.bench_function("naive recursion", |b| b.iter(|| fib_naive(black_box(30))));
    group.bench_function("memo (top-down)", |b| b.iter(|| fib_memo(black_box(30))));
    group.bench_function("table (bottom-up)", |b| b.iter(|| fib_table(black_box(30))));
    group.bench_function("two variables", |b| b.iter(|| fib_two_vars(black_box(30))));
    group.finish();
}

criterion_group!(benches, bench_fib);
criterion_main!(benches);
```

```bash
cargo bench --bench dp -- --warm-up-time 1 --measurement-time 3
```

```text
fib 30/naive recursion  time:   [1.4161 ms 1.4206 ms 1.4255 ms]
fib 30/memo (top-down)  time:   [84.362 ns 84.663 ns 85.002 ns]
fib 30/table (bottom-up)
                        time:   [42.399 ns 42.889 ns 43.700 ns]
fib 30/two variables    time:   [8.2058 ns 8.2905 ns 8.3895 ns]
```

(Numbers from an Apple M4 Mac; yours will differ, the ratios will be similar.)

**What happens:** the memo version is about 17,000 times faster than the naive recursion for `n = 30`, and the two-variable version about 170,000 times. The memo costs a vector allocation and the recursion costs calls, so it is slower than the table; the table allocates a vector, so it is slower than two variables. For `n = 40` the gap would be 100 times larger, because the naive version grows exponentially and the others grow linearly.

> ⚡ **Performance:** DP turns an exponential algorithm into a polynomial one by remembering. The price is memory for the table. Counting the states times the work per state tells you the cost before you write any code.

## Common mistakes

**1. "Infinity" that overflows.** Many DP solutions mark "impossible" with the biggest number. Then adding one to it breaks:

```rust
fn min_coins(coins: &[usize], amount: usize) -> usize {
    let mut best = vec![usize::MAX; amount + 1]; // "infinity" for "impossible"
    best[0] = 0;
    for a in 1..=amount {
        for &coin in coins {
            if coin <= a && best[a - coin] + 1 < best[a] {
                best[a] = best[a - coin] + 1;
            }
        }
    }
    best[amount]
}

fn main() {
    println!("{}", min_coins(&[2], 3));
}
```

```text
thread 'main' (4697954) panicked at src/main.rs:6:29:
attempt to add with overflow
```

**What it means:** `best[1]` is "infinity" (`usize::MAX`), and `best[1] + 1` overflows. **Fix:** use `Option<usize>` as in `min_coins`, or skip impossible states explicitly, or use a "big but safe" value such as `amount + 1` for infinity.

**2. Stack overflow in the top-down version.** Memoization still recurses, one call deeper for each step. A big `n` exhausts the call stack ([Lesson 07.7](07-recursion.md)):

```rust
fn fib_memo(n: usize, memo: &mut Vec<Option<u64>>) -> u64 {
    if n < 2 {
        return n as u64;
    }
    if let Some(known) = memo[n] {
        return known;
    }
    let result = fib_memo(n - 1, memo).wrapping_add(fib_memo(n - 2, memo));
    memo[n] = Some(result);
    result
}

fn main() {
    let n = 1_000_000;
    let mut memo = vec![None; n + 1];
    println!("{}", fib_memo(n, &mut memo));
}
```

```text
thread 'main' (4698048) has overflowed its stack
fatal runtime error: stack overflow, aborting
```

**Fix:** use the bottom-up table (a loop has no call stack), or make sure the recursion depth is small.

**3. Off-by-one between the table and the text.** In `lcs_len`, cell `table[i][j]` describes the first `i` characters, so the characters it compares are `a[i - 1]` and `b[j - 1]`. Writing `a[i]` panics on the last row or compares the wrong characters. The tests with empty and one-letter texts catch it.

**4. Reading a cell that is not filled yet.** If the loops run in the wrong order, the recurrence reads a zero that is only a starting value, and the answer is wrong *without* any error message. This is why each solution states its order, and why a brute-force judge is valuable.

## Try it yourself

1. Add `fib_counting` and the example from Section 1 and run it. Change `40` to `45` and predict the number of calls before you run (hint: each step multiplies by about 1.6).
2. Run `min_coins(&[1, 3, 4], 6)` by hand with a pencil: fill the table as in the picture of Section 4. Then change the coins to `[2, 5]` and the amount to 3 and check that the answer is `None`.
3. Run `cargo fmt && cargo clippy --all-targets -- -D warnings && cargo test`, then `git add code/p07-algorithms && git commit -m "feat(p07): dynamic programming"`.

## Exercises

### Exercise 1 — Climbing stairs (easy)

A staircase has `n` steps. You can climb 1 or 2 steps at a time. In how many different ways can you reach the top? Write `pub fn ways_to_climb(n: usize) -> u64` with O(n) time and O(1) space, and a test. (There is 1 way for `n = 0`, 1 way for `n = 1`, 2 ways for `n = 2`.)

<details>
<summary>Hint</summary>

The last move is either 1 step (from step `n - 1`) or 2 steps (from step `n - 2`). So `ways(n) = ways(n - 1) + ways(n - 2)`. It is the Fibonacci rule with different starting values. Test with the numbers 1, 1, 2, 3, 5, 8, 13.

</details>

<details>
<summary>Answer</summary>

```rust
/// Ways to climb `n` stairs with steps of 1 or 2. Time O(n), space O(1).
pub fn ways_to_climb(n: usize) -> u64 {
    let (mut one_before, mut now) = (1u64, 1u64); // ways(0), ways(1)
    for _ in 1..n {
        (one_before, now) = (now, one_before + now);
    }
    now
}

// The test goes into the `tests` module at the end of `src/dp.rs`:
    #[test]
    fn stairs() {
        let expected = [1, 1, 2, 3, 5, 8, 13];
        for (n, &want) in expected.iter().enumerate() {
            assert_eq!(ways_to_climb(n), want, "n = {n}");
        }
    }
```

Why it works: state `n`, recurrence `ways(n) = ways(n-1) + ways(n-2)`, base cases `ways(0) = ways(1) = 1`, and only the last two values are kept. (For very large `n` the `u64` would overflow; use `checked_add` as in `fib_two_vars` if you need that.)

</details>

### Exercise 2 — Complexity and memory (medium)

(a) State the time and space complexity of `min_coins` and `lcs_len`, using `A` for the amount, `C` for the number of coins, and `n`, `m` for the text lengths. (b) How can `lcs_len` use only O(min(n, m)) memory? (c) Why can the memory of `min_coins` **not** be reduced the same way?

<details>
<summary>Hint</summary>

For (b), look at which cells the recurrence reads: only row `i - 1` and row `i`. For (c), which earlier cells does `best[a]` read?

</details>

<details>
<summary>Answer</summary>

(a) `min_coins`: time O(A · C), space O(A). `lcs_len`: time O(n · m), space O(n · m).

(b) The recurrence of cell `(i, j)` reads only `(i-1, j-1)`, `(i-1, j)` and `(i, j-1)`: the previous row and the current row. Keep just two rows (or one row plus one variable) and let the shorter text be the one that runs across the row: then the memory is O(min(n, m)). The price: you can no longer rebuild the actual subsequence from the table, only get its length.

(c) `best[a]` reads `best[a - c]` for *every* coin `c`, so it may need a cell that is up to the biggest coin value behind. You would have to keep the last `max(coins)` cells, which is not smaller than O(A) in general.

</details>

### Exercise 3 — 0/1 knapsack (medium)

You have items with weights and values. A bag holds total weight at most `capacity`. Each item can be taken **at most once**. Write `pub fn knapsack(weights: &[usize], values: &[u32], capacity: usize) -> u32` with the biggest total value. Test it with small examples that you solve by hand.

<details>
<summary>Hint</summary>

State: `best[c]` = the biggest value for weight limit `c`. For each item `(w, v)`: `best[c] = max(best[c], best[c - w] + v)`. To use each item only once with a single vector, loop `c` from `capacity` **down** to `w`. What goes wrong if you loop upward?

</details>

<details>
<summary>Answer</summary>

```rust
/// 0/1 knapsack: the biggest total value with total weight <= capacity.
/// Each item is used at most once. Time O(items * capacity), space O(capacity).
pub fn knapsack(weights: &[usize], values: &[u32], capacity: usize) -> u32 {
    // best[c] = the biggest value with weight limit c
    let mut best = vec![0u32; capacity + 1];
    for (&w, &v) in weights.iter().zip(values) {
        // go from big c to small c, so that the item is used at most once
        for c in (w..=capacity).rev() {
            best[c] = best[c].max(best[c - w] + v);
        }
    }
    best[capacity]
}

// The test goes into the `tests` module at the end of `src/dp.rs`:
    #[test]
    fn knapsack_examples() {
        assert_eq!(knapsack(&[1, 3, 4, 5], &[1, 4, 5, 7], 7), 9); // items 3 and 4
        assert_eq!(knapsack(&[], &[], 10), 0);
        assert_eq!(knapsack(&[5], &[10], 4), 0);
        assert_eq!(knapsack(&[2, 3, 4], &[3, 4, 5], 5), 7); // items 1 and 2
        assert_eq!(knapsack(&[1, 1, 1], &[1, 2, 3], 2), 5); // items 2 and 3
    }
```

Why it works: when `c` goes downwards, `best[c - w]` has not yet been updated for the current item, so the item is counted once. If `c` went upwards, `best[c - w]` would already include the item and it could be taken many times (that is the "unlimited" version, like the coins). Time O(items · capacity), space O(capacity). For a stronger test, write a brute-force judge that tries all subsets and compare on random inputs, as `coins_agree_with_the_slow_version` does.

</details>

## Quiz

1. What makes a recursive solution a candidate for dynamic programming?

<details>
<summary>Answer</summary>

Overlapping subproblems: the recursion solves the same smaller problem many times (like `fib(3)` in `fib(5)`), and the best answer is built from the best answers of the smaller problems (optimal substructure). If the subproblems do not overlap (merge sort), plain recursion is enough.

</details>

2. What is the difference between top-down and bottom-up DP?

<details>
<summary>Answer</summary>

Top-down keeps the recursion and adds a memo; it computes only the states that are needed, but it uses the call stack and can overflow it. Bottom-up fills a table from the smallest state to the largest in a loop; it has no recursion and is usually faster, but you must find a correct filling order.

</details>

3. What are the time and space of `min_coins` for amount `A` and `C` coin values?

<details>
<summary>Answer</summary>

Time O(A · C) (A states, C coins each), space O(A) (one vector of A + 1 cells).

</details>

4. Why is `usize::MAX` a dangerous marker for "impossible" in a DP table?

<details>
<summary>Answer</summary>

The recurrence adds 1 to values from the table, and `usize::MAX + 1` overflows (a panic in debug builds, a wrong value in release builds). Use `Option`, skip impossible states, or use a safe big value such as `amount + 1`.

</details>

## Summary

- A recursion with overlapping subproblems repeats work exponentially. Counting the calls shows it: `fib(40)` makes 331 million calls for 41 distinct subproblems.
- Dynamic programming solves each subproblem once: top-down with a memo, or bottom-up with a table. The cost is (number of states) × (work per state).
- Recipe: state, recurrence, base cases, order, answer. Write them as comments first.
- Keep a full table first; shrink it to a few rows or variables only when the recurrence reads only those.
- Use `Option` or `checked_*` for "impossible" and for overflow; test with empty inputs and the border values.
- A brute-force judge (all subsets, all subsequences, plain recursion) on tiny random inputs finds off-by-one and ordering bugs. The standard library has no DP helper, so you write the table yourself.

## Sources and further reading

- [The Rust Programming Language — 8.1 Storing Lists of Values with Vectors](https://doc.rust-lang.org/book/ch08-01-vectors.html) — vectors, used for all tables here.
- [The Rust Standard Library — Option](https://doc.rust-lang.org/std/option/enum.Option.html) — `is_none_or`, `and_then`, `map_or`.
- [The Rust Standard Library — u64::checked_add](https://doc.rust-lang.org/std/primitive.u64.html#method.checked_add) — arithmetic that reports overflow.
- [Criterion.rs User Guide](https://bheisler.github.io/criterion.rs/book/) — the benchmark library used in Section 6.
- [Rust by Example — Iterators](https://doc.rust-lang.org/rust-by-example/trait/iter.html) — short runnable examples of the iterator methods used in the judges.

---

[🏠 Start](../START-HERE.md) · [← 07.15 Two pointers and sliding window](15-two-pointers-and-sliding-window.md) · [📋 Phase 07](README.md) · [07.17 Greedy algorithms →](17-greedy-algorithms.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
