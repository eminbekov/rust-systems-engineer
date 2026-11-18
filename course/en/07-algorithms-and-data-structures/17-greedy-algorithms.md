<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 07.16 Dynamic programming](16-dynamic-programming.md) · [📋 Phase 07](README.md) · [07.18 Problem-solving practice →](18-problem-solving-practice.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/07-algorithms-and-data-structures/17-greedy-algorithms.md) <!-- nav -->

# Lesson 07.17 — Greedy algorithms

> **Phase 07 — Algorithms and data structures** · Lesson 17 of 20 · ⏱ about 120 minutes

## What you will learn

- What a greedy algorithm is: always take the choice that looks best right now, and never change your mind.
- Why greedy is fast and simple, and why it is **sometimes wrong**.
- Three classic greedy algorithms: choosing the most meetings, merging ranges, and counting rooms with a heap.
- How to find out whether a greedy rule is correct: a plain-words proof idea, and a test against a brute-force judge.

## Before you start

- You finished [Lesson 07.16 — Dynamic programming](16-dynamic-programming.md). We use its `min_coins` as the judge for a greedy coin algorithm.
- You know sorting ([Lesson 07.8](08-simple-sorting-algorithms.md), [Lesson 07.9](09-merge-sort-and-quicksort.md)) and `BinaryHeap` ([Lesson 07.12](12-heaps-and-priority-queues.md)).
- You work in `code/p07-algorithms`. This lesson adds the module `src/greedy.rs` (and `pub mod greedy;` in `src/lib.rs`).

## New words

| Word | Meaning |
|---|---|
| **greedy algorithm** | An algorithm that builds the answer step by step and at each step takes the choice that looks best *now*, without looking ahead and without undoing it later. |
| **greedy-choice property** | The fact that, for a given problem, a locally best choice can always be extended to a best answer overall. A greedy algorithm is correct exactly when the problem has it. |
| **exchange argument** | A way to prove a greedy rule: take any best answer, swap one of its choices for the greedy choice, and show that the answer does not get worse. |
| **counterexample** | One concrete input on which a rule gives a wrong answer. One is enough to show that a rule is wrong. |
| **half-open range** | A range `[start, end)` that includes `start` but not `end`. Two half-open ranges that only touch (`[1, 2)` and `[2, 3)`) do not overlap. |
| **min-heap** | A heap whose `pop` gives the *smallest* item. Rust's `BinaryHeap` gives the biggest, so we wrap items in `Reverse` to flip the order. |

## 1. The greedy idea, and where it fails

You pay 63 cents with the fewest coins. Without thinking you take the biggest coin that fits (25), then again (25), then 10, then 1, 1, 1: six coins. That habit is a *greedy algorithm*: at each step take the best-looking choice, never reconsider.

```rust
use std::cmp::Reverse;
use std::collections::BinaryHeap;

/// Pays `amount` by always taking the biggest coin that still fits.
/// Returns the number of coins, or `None` if some rest cannot be paid.
/// Time O(C log C) for C coin values, space O(C).
pub fn greedy_coins(coins: &[usize], amount: usize) -> Option<usize> {
    let mut sorted: Vec<usize> = coins.iter().copied().filter(|&c| c > 0).collect();
    sorted.sort_unstable_by(|a, b| b.cmp(a)); // biggest first
    let mut rest = amount;
    let mut count = 0;
    for coin in sorted {
        count += rest / coin; // take as many of this coin as fit
        rest %= coin;
    }
    if rest == 0 { Some(count) } else { None }
}
```

Create `src/greedy.rs` with these lines. The two `use` lines are needed in Section 5; until then the compiler warns that they are unused.

**What happens:**

1. We copy the coin values, drop zeros (a zero coin would divide by zero) and sort them from big to small.
2. For each coin, `rest / coin` is how many of it fit (integer division), and `rest % coin` is what is left.
3. If something is left when the coins are used up, the amount cannot be paid: `None`.
4. It is one pass over the coins: **O(C log C)** for the sort plus a few operations; space O(C) for the copy.

Is it correct? [Lesson 07.16](16-dynamic-programming.md) gave us a trusted judge, `min_coins`. Add this to `src/greedy.rs` (the random tests below use the `Rng` of `src/testutil.rs`, with the `range` method from Lesson 07.15):

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use crate::dp::min_coins;
    use crate::testutil::Rng;

    #[test]
    fn greedy_coins_works_for_the_usual_coins() {
        let coins = [1, 5, 10, 25];
        for amount in 0..=300 {
            assert_eq!(
                greedy_coins(&coins, amount),
                min_coins(&coins, amount),
                "amount = {amount}"
            );
        }
    }

    #[test]
    fn greedy_coins_is_wrong_for_1_3_4() {
        // greedy: 4 + 1 + 1 = 3 coins. best: 3 + 3 = 2 coins.
        assert_eq!(greedy_coins(&[1, 3, 4], 6), Some(3));
        assert_eq!(min_coins(&[1, 3, 4], 6), Some(2));
    }
}
```

Both tests pass. The first says: for the coins 1, 5, 10, 25 greedy gives the best answer for every amount from 0 to 300. The second *documents a counterexample*: with coins 1, 3, 4 greedy needs 3 coins for 6, while 3 + 3 needs 2. One counterexample proves that "biggest coin first" is **not** correct in general.

The brute-force judge can also *find* counterexamples for you. Try this temporary test (coin systems with a `1`, so every amount can be paid):

```rust
    #[test]
    fn greedy_coins_is_always_optimal() {
        let mut rng = Rng::new(5);
        for _ in 0..1000 {
            let mut coins = vec![1usize];
            for _ in 0..rng.range(1, 3) {
                coins.push(rng.range(2, 9) as usize);
            }
            let amount = rng.range(0, 30) as usize;
            assert_eq!(
                greedy_coins(&coins, amount),
                min_coins(&coins, amount),
                "coins = {coins:?}, amount = {amount}"
            );
        }
    }
```

```text
---- greedy::tests::greedy_coins_is_always_optimal stdout ----

thread 'greedy::tests::greedy_coins_is_always_optimal' (6548020) panicked at src/greedy.rs:224:13:
assertion `left == right` failed: coins = [1, 7, 8, 7], amount = 21
  left: Some(7)
 right: Some(3)
```

**What happens:** the random test found a counterexample on its own: greedy pays 21 as 8 + 8 + 1 + 1 + 1 + 1 + 1 (7 coins), but 7 + 7 + 7 is 3 coins. (The coin 7 appears twice in the list; that does not matter.) The test is *expected* to fail, so delete it again after you have seen the message. (The number in brackets is a thread number, and the line number depends on your file; yours will differ.)

> 💡 **Tip:** The lesson of this section: **never trust a greedy rule because it feels right.** Either you can argue why it is correct (Section 2), or you test it against a slow, correct algorithm. This is the same technique as in Lesson 07.15, now used to *find* a bug in an idea.

## 2. When is greedy correct?

A greedy algorithm is correct when the problem has the *greedy-choice property*: some best answer contains the greedy choice. The usual proof is an **exchange argument**, told in plain words:

1. Take any best answer (an "optimal" one).
2. If it already contains the greedy choice, you are done.
3. If not, **swap** one of its choices for the greedy choice. Show that the result is still valid and not worse.
4. So a best answer with the greedy choice exists. Repeat the argument on what is left.

For the coins 1, 5, 10, 25 the swap argument works (such coin systems are called *canonical*). For the coins 1, 3, 4 it fails: forcing the greedy coin 4 into the best answer 3 + 3 gives 4 + 1 + 1, which is worse.

Practical checklist:

| Question | If the answer is "no" |
|---|---|
| Can I state the rule in one sentence ("sort by X, then take if it fits")? | It is probably not greedy; think dynamic programming (DP, [Lesson 07.16](16-dynamic-programming.md)). |
| Can I explain with an exchange argument why it is safe? | Do not trust it yet. |
| Does it agree with a brute-force judge on thousands of tiny random inputs? | The rule is wrong. Find the counterexample and change the rule. |

## 3. Choosing the most meetings

**The problem.** One room, many meeting requests, each a half-open time range `[start, end)`. Choose the **most** meetings so that no two overlap.

Which rule? "Earliest start first"? "Shortest first"? Both have counterexamples. The rule that works: **earliest end first**. Sort by end time; take a meeting whenever it starts at or after the end of the last one taken. Finishing early leaves the most room for the rest.

```text
time  0   1   2   3   4   5   6   7   8   9   10  11
A     [--------------------------------------)  (0,10)
B         [------)  (1,3)                                 <- taken (ends first)
C             [----------)  (2,5)                          starts at 2 < 3: skip
D                     [----------)  (4,7)                  <- taken (4 >= 3)
E                             [----------)  (6,9)          starts at 6 < 7: skip
F                                     [----------)  (8,11) <- taken (8 >= 7)
```

Sorted by end, the order is B, C, D, E, A, F. The result is B, D, F: 3 meetings. "Earliest start first" would take A (it starts at 0) and block everything else: 1 meeting.

```rust
/// A meeting is the half-open time range `[start, end)`.
pub type Meeting = (u32, u32);

/// The biggest number of meetings that do not overlap in one room.
/// Greedy: sort by END time, take a meeting whenever it starts after the last
/// taken one has ended. Time O(n log n), extra space O(n) for the sorted copy.
pub fn max_meetings(meetings: &[Meeting]) -> usize {
    let mut sorted = meetings.to_vec();
    sorted.sort_by_key(|&(_, end)| end);
    let mut last_end = 0;
    let mut count = 0;
    for (start, end) in sorted {
        if start >= last_end {
            count += 1;
            last_end = end;
        }
    }
    count
}
```

**What happens:**

1. `type Meeting = (u32, u32);` gives a name to a tuple of start and end. `to_vec()` copies the input, because we must not reorder the caller's data.
2. `sort_by_key(|&(_, end)| end)` sorts by the second item of each tuple. The sort is stable and takes O(n log n) ([Lesson 07.9](09-merge-sort-and-quicksort.md)).
3. `start >= last_end` is the overlap test. With half-open ranges a meeting that starts exactly when the last one ended is fine, so `>=`, not `>` (see Common mistakes).
4. The loop is O(n) after the sort: **O(n log n) time**, O(n) extra space.

**Why it is correct (the exchange argument).** Take any best schedule. Look at its first meeting `X`, and at the meeting `G` with the earliest end among all meetings. `G` ends no later than `X`, so replace `X` by `G`: it cannot overlap the second meeting of the schedule, because it ends earlier. The schedule is still valid and just as big. So some best schedule starts with `G`; remove `G` and every meeting that overlaps it, and repeat.

The judge tries all 2ⁿ subsets of meetings (n up to 10) and keeps the biggest one without overlaps:

```rust
    #[test]
    fn meetings_example() {
        let meetings = [(0, 10), (1, 3), (2, 5), (4, 7), (6, 9), (8, 11)];
        assert_eq!(max_meetings(&meetings), 3); // (1,3) (4,7) (8,11)
        assert_eq!(max_meetings(&[]), 0);
        assert_eq!(max_meetings(&[(0, 10), (1, 2), (3, 4)]), 2);
        assert_eq!(max_meetings(&[(1, 2), (2, 3)]), 2); // touching is allowed
    }

    /// Judge: try every subset of meetings (2^n) and keep the biggest one
    /// in which no two meetings overlap.
    fn max_meetings_slow(m: &[Meeting]) -> usize {
        let n = m.len();
        let mut best = 0;
        for mask in 0u32..(1 << n) {
            let chosen: Vec<Meeting> = (0..n)
                .filter(|&i| mask & (1 << i) != 0)
                .map(|i| m[i])
                .collect();
            let ok = chosen
                .iter()
                .enumerate()
                .all(|(i, a)| chosen[i + 1..].iter().all(|b| a.1 <= b.0 || b.1 <= a.0));
            if ok {
                best = best.max(chosen.len());
            }
        }
        best
    }

    fn random_meetings(rng: &mut Rng, max_len: i64) -> Vec<Meeting> {
        (0..rng.range(0, max_len))
            .map(|_| {
                let start = rng.range(0, 12);
                (start as u32, (start + rng.range(1, 6)) as u32)
            })
            .collect()
    }

    #[test]
    fn max_meetings_agrees_with_the_judge() {
        let mut rng = Rng::new(11);
        for _ in 0..500 {
            let m = random_meetings(&mut rng, 10);
            assert_eq!(max_meetings(&m), max_meetings_slow(&m), "{m:?}");
        }
    }
```

**What happens:** each bit of `mask` says whether meeting `i` is chosen. Two ranges do not overlap when one ends before (or when) the other starts: `a.1 <= b.0 || b.1 <= a.0`. `random_meetings` makes up to 9 meetings that start at 0 to 12 and last 1 to 6 units, so many overlap. All three tests pass; the greedy agrees with the judge on 500 random inputs.

## 4. Merging ranges

**The problem.** Closed ranges `[start, end]` are given in any order. Merge the ones that overlap or touch: `(1,3), (2,6), (6,7), (8,10)` become `(1,7), (8,10)`.

Greedy rule: sort by **start**; keep a list of merged ranges; for each range, if it starts at or before the end of the last merged range, extend that one, otherwise open a new one.

```rust
/// Merges ranges that overlap or touch. Ranges are closed: `[start, end]`.
/// Greedy: sort by START, then grow the last result range or open a new one.
/// Time O(n log n), space O(n).
pub fn merge_ranges(ranges: &[(i32, i32)]) -> Vec<(i32, i32)> {
    let mut sorted = ranges.to_vec();
    sorted.sort_unstable();
    let mut merged: Vec<(i32, i32)> = Vec::new();
    for (start, end) in sorted {
        match merged.last_mut() {
            Some(last) if start <= last.1 => last.1 = last.1.max(end),
            _ => merged.push((start, end)),
        }
    }
    merged
}
```

**What happens:**

1. Tuples compare by the first item and then the second, so `sort_unstable()` sorts by start. We do not need stability here, and `sort_unstable` is a little faster.
2. `merged.last_mut()` is `Some(&mut last)` when the list is not empty. The *match guard* `if start <= last.1` selects the "overlaps or touches" case; there we only grow the end: `last.1.max(end)` (the new range may lie completely inside the old one).
3. Otherwise (`_`: the list is empty or there is a gap) we start a new merged range.

Time O(n log n) for the sort, O(n) for the loop; space O(n) for the output. The judge for this one cannot be a copy of the algorithm, so we test a *property*: the merged ranges are sorted with gaps between them, and they cover exactly the same points as the input. To see the half-points we double all numbers:

```rust
    #[test]
    fn merge_ranges_example() {
        let merged = merge_ranges(&[(8, 10), (1, 3), (2, 6), (15, 18), (17, 20), (6, 7)]);
        assert_eq!(merged, vec![(1, 7), (8, 10), (15, 20)]);
        assert_eq!(merge_ranges(&[]), vec![]);
        assert_eq!(merge_ranges(&[(1, 2), (3, 4)]), vec![(1, 2), (3, 4)]);
    }

    #[test]
    fn merge_ranges_covers_the_same_points() {
        let mut rng = Rng::new(21);
        for _ in 0..500 {
            let ranges: Vec<(i32, i32)> = (0..rng.range(0, 6))
                .map(|_| {
                    let start = rng.range(0, 15) as i32;
                    (start, start + rng.range(0, 5) as i32)
                })
                .collect();
            let merged = merge_ranges(&ranges);
            // the result is sorted and its ranges do not touch each other
            assert!(merged.windows(2).all(|w| w[0].1 < w[1].0), "{merged:?}");
            // the same points are covered (points are doubled to see the half-points)
            for t in 0..=50 {
                let covers = |rs: &[(i32, i32)]| rs.iter().any(|&(s, e)| 2 * s <= t && t <= 2 * e);
                assert_eq!(covers(&ranges), covers(&merged), "{ranges:?} at {t}");
            }
        }
    }
```

## 5. Counting rooms with a min-heap

**The problem.** Meetings `[start, end)`. What is the **fewest rooms** so that overlapping meetings never share a room?

Greedy rule: process the meetings by **start** time. Keep the end times of the rooms in use in a **min-heap**, so the room that frees up first is on top. For a new meeting: if the earliest end is at or before its start, that room is free, so reuse it (pop); then put the meeting's end into the heap. The size of the heap at the end is the number of rooms.

```rust
/// The fewest rooms needed so that no two overlapping meetings share a room.
/// Greedy: walk the meetings by START time; a min-heap holds the END time of
/// every room in use. Time O(n log n), space O(n).
pub fn min_rooms(meetings: &[Meeting]) -> usize {
    let mut sorted = meetings.to_vec();
    sorted.sort_by_key(|&(start, _)| start);
    let mut ends: BinaryHeap<Reverse<u32>> = BinaryHeap::new();
    for (start, end) in sorted {
        if ends
            .peek()
            .is_some_and(|&Reverse(earliest_end)| earliest_end <= start)
        {
            ends.pop(); // that room is free: reuse it
        }
        ends.push(Reverse(end));
    }
    ends.len()
}
```

**What happens:**

1. `BinaryHeap` is a **max**-heap: `pop` and `peek` give the biggest item ([Lesson 07.12](12-heaps-and-priority-queues.md); remember that this heap is the data structure, not the heap memory of Phase 03). `Reverse(x)` is a wrapper that flips the comparison, so the *smallest* end time is on top.
2. `peek()` looks at the top without removing it. `is_some_and(...)` is `false` for an empty heap, and otherwise runs the closure on the top item; the pattern `&Reverse(earliest_end)` unwraps the reference and the wrapper.
3. Reusing a room means: remove its old end time (`pop`), then add the new meeting's end (`push`). A new room means just `push`.
4. Each meeting does O(1) heap operations of O(log n) each: **O(n log n) time**, O(n) space.

The judge counts, for each moment `t`, how many meetings are running (`s <= t < e`); the answer must be the busiest moment:

```rust
    #[test]
    fn min_rooms_agrees_with_counting_overlaps() {
        assert_eq!(min_rooms(&[(0, 30), (5, 10), (15, 20)]), 2);
        assert_eq!(min_rooms(&[(7, 10), (2, 4)]), 1);
        assert_eq!(min_rooms(&[]), 0);
        let mut rng = Rng::new(33);
        for _ in 0..500 {
            let m = random_meetings(&mut rng, 12);
            // judge: the biggest number of meetings that are running at one moment
            let busiest = (0..=20u32)
                .map(|t| m.iter().filter(|&&(s, e)| s <= t && t < e).count())
                .max()
                .unwrap_or(0);
            assert_eq!(min_rooms(&m), busiest, "{m:?}");
        }
    }
```

Run `cargo test greedy`. The seven tests of this lesson pass:

```text
test result: ok. 7 passed; 0 failed; 0 ignored; 0 measured; 75 filtered out; finished in 0.04s
```

(The 75 filtered-out tests come from the main text of Lessons 07.1 to 07.16; with the tests of the exercises your number is bigger.)

> ⚡ **Performance:** All three algorithms spend their time in the sort: O(n log n). The greedy loops after the sort are linear. That is why greedy solutions are usually much faster than DP solutions that need a table. No benchmark is needed here: use the standard `sort` family and `BinaryHeap`, which are already well optimized.

## 6. Greedy, DP or brute force?

| If the problem… | Use |
|---|---|
| has a rule you can prove with an exchange argument | Greedy: fast and short. |
| asks for the best of many combinations and a local rule fails (counterexample!) | Dynamic programming ([Lesson 07.16](16-dynamic-programming.md)). |
| is tiny (n ≤ 20) | Brute force is also fine, and it is the best judge for tests. |

Standard tools used in this lesson: `sort_by_key` and `sort_unstable` (sorting), `BinaryHeap` with `Reverse` (a min-heap), `slice::windows` (looking at neighbours), `Vec::last_mut` (the last item).

## Common mistakes

**1. Sorting by a floating-point key.** `sort_by_key` needs a key with a total order (`Ord`), and `f64` has none because of `NaN`:

```rust
fn main() {
    let mut prices = vec![2.5, 0.5, 1.0];
    prices.sort_by_key(|&p| p);
    println!("{prices:?}");
}
```

```text
error[E0277]: the trait bound `{float}: Ord` is not satisfied
 --> src/main.rs:3:12
  |
3 |     prices.sort_by_key(|&p| p);
  |            ^^^^^^^^^^^ the trait `Ord` is not implemented for `{float}`
```

**Fix:** `prices.sort_by(|a, b| a.total_cmp(b))`, which uses the total order that `f64` offers through `total_cmp`, or scale to integers.

**2. `>` instead of `>=` for touching ranges.** If the meetings `(1, 2)` and `(2, 3)` may share the boundary, the test is `start >= last_end`. With `>` the second meeting is wrongly rejected:

```rust
fn main() {
    // meetings as (start, end) in minutes; touching meetings may share a room
    let mut meetings = vec![(1u32, 2u32), (2, 3)];
    meetings.sort_by_key(|&(_, end)| end);
    let mut last_end = 0;
    let mut count = 0;
    for (start, end) in meetings {
        if start > last_end {
            count += 1;
            last_end = end;
        }
    }
    println!("{count} meetings (the right answer is 2)");
}
```

```text
1 meetings (the right answer is 2)
```

The test `meetings_example` with `(1, 2), (2, 3)` catches this. Always write down whether the ranges are open, closed or half-open before you write the comparison, and test the border where two ranges touch.

**3. Trusting a rule that works on the examples.** "Biggest coin first" works for 63 cents. The rule is wrong anyway (Section 1). Check a greedy rule with a judge before you rely on it.

## Try it yourself

1. Add `greedy_coins_is_always_optimal` from Section 1, read the counterexample, then delete the test. Change the coin list in `greedy_coins_works_for_the_usual_coins` to `[1, 5, 12, 25]` and find the first amount that fails (use a smaller range and read the message).
2. Draw the timeline of the meetings `(0, 6), (1, 2), (3, 4), (5, 8)` on paper and run the greedy rule by hand. What does `max_meetings` return? Check with a test.
3. Run `cargo fmt && cargo clippy --all-targets -- -D warnings && cargo test`, then `git add code/p07-algorithms && git commit -m "feat(p07): greedy algorithms"`.

## Exercises

### Exercise 1 — A wrong greedy, caught by a test (easy)

Write the **wrong** rule "shortest meeting first" as a function `max_meetings_shortest_first(meetings: &[Meeting]) -> usize` (sort by `end - start`, take a meeting if it overlaps none already taken). Then write a test with three meetings that shows it is worse than `max_meetings`.

<details>
<summary>Hint</summary>

A short meeting in the middle that overlaps two longer ones that do not overlap each other. For example `(0, 5)`, `(4, 6)` and `(5, 10)`: the short one is `(4, 6)`.

</details>

<details>
<summary>Answer</summary>

```rust
    /// The WRONG greedy: shortest meeting first.
    fn max_meetings_shortest_first(meetings: &[Meeting]) -> usize {
        let mut sorted = meetings.to_vec();
        sorted.sort_by_key(|&(start, end)| end - start);
        let mut taken: Vec<Meeting> = Vec::new();
        for (start, end) in sorted {
            if taken.iter().all(|&(s, e)| end <= s || e <= start) {
                taken.push((start, end));
            }
        }
        taken.len()
    }

    #[test]
    fn shortest_first_loses_to_earliest_end_first() {
        let meetings = [(0, 5), (4, 6), (5, 10)];
        assert_eq!(max_meetings_shortest_first(&meetings), 1);
        assert_eq!(max_meetings(&meetings), 2);
    }
```

Why it works: "shortest first" takes `(4, 6)` (length 2), and then both `(0, 5)` and `(5, 10)` overlap it: 1 meeting. "Earliest end first" takes `(0, 5)` and `(5, 10)`: 2 meetings. One counterexample is enough to reject a rule. Both tests pass (`cargo test`).

</details>

### Exercise 2 — Complexity (medium)

State the time and the extra space of `max_meetings`, `merge_ranges` and `min_rooms` for `n` meetings, and say which step dominates. Then say how the time of `min_rooms` would change if you replaced the heap by a plain `Vec` of end times, searched with `iter().min()` for each meeting.

<details>
<summary>Hint</summary>

Sorting costs O(n log n). A heap `push` or `pop` costs O(log n). Searching an unsorted `Vec` for its smallest item costs O(its length).

</details>

<details>
<summary>Answer</summary>

All three: time **O(n log n)**, dominated by the sort; extra space O(n) (the sorted copy, the output or the heap). With a plain `Vec` of up to `n` end times and a linear search for each of the `n` meetings, `min_rooms` would cost O(n) per meeting and **O(n²)** in total. The heap brings the search down to O(log n) per meeting (a peek is O(1), a push or pop O(log n)).

</details>

### Exercise 3 — Jump game (medium)

An array of non-negative numbers: `jumps[i]` is the longest jump you can make from position `i` (you may jump any shorter distance too). You start at position 0. Write `pub fn can_reach_end(jumps: &[usize]) -> bool` that says whether you can reach the last position, in O(n) time and O(1) space. Test it with examples and against a slow recursive judge.

<details>
<summary>Hint</summary>

Walk from left to right and remember the **farthest** position you can reach so far. If you stand on a position that is farther than that, you are stuck: return `false`. Otherwise update the farthest position with `i + jumps[i]`.

</details>

<details>
<summary>Answer</summary>

```rust
/// Can you reach the last position? `jumps[i]` is the longest jump from `i`.
/// Greedy: remember the farthest position reachable so far. Time O(n), space O(1).
pub fn can_reach_end(jumps: &[usize]) -> bool {
    let mut farthest = 0;
    for (i, &jump) in jumps.iter().enumerate() {
        if i > farthest {
            return false; // position i cannot be reached
        }
        farthest = farthest.max(i + jump);
    }
    true
}

// The judge and the test go into the `tests` module at the end of `src/greedy.rs`:
    /// Judge: try every jump length by recursion.
    fn can_reach_slow(jumps: &[usize], from: usize) -> bool {
        if from + 1 >= jumps.len() {
            return true;
        }
        (1..=jumps[from]).any(|step| can_reach_slow(jumps, from + step))
    }

    #[test]
    fn jump_game() {
        assert!(can_reach_end(&[2, 3, 1, 1, 4]));
        assert!(!can_reach_end(&[3, 2, 1, 0, 4]));
        assert!(can_reach_end(&[0]));
        assert!(can_reach_end(&[]));
        // random comparison with the judge: see Rng in the tests of this lesson
    }
```

Why it works: every position up to `farthest` is reachable (you can stop at any shorter distance), so the only way to fail is to meet a position beyond `farthest`. The `[3, 2, 1, 0, 4]` example gets stuck at the `0` at position 3, because `farthest` is 3 and position 4 is out of reach. To use the judge on random inputs, build `jumps` with the `Rng` of this lesson (values 0 to 3, length 1 to 9) and compare both functions. Tests pass.

</details>

## Quiz

1. What is a greedy algorithm?

<details>
<summary>Answer</summary>

An algorithm that builds the answer in steps, takes at each step the choice that looks best at that moment, and never undoes it. It is usually fast (often a sort plus one pass) but correct only for problems that have the greedy-choice property.

</details>

2. Greedy gives 3 coins for 6 with coins 1, 3 and 4, but the best is 2. What does this show, and what do you do?

<details>
<summary>Answer</summary>

It is a counterexample: it proves that "biggest coin first" is not correct for every coin system. For such problems use dynamic programming ([Lesson 07.16](16-dynamic-programming.md)), which tries all last coins.

</details>

3. Why does "earliest end first" choose the most meetings?

<details>
<summary>Answer</summary>

By the exchange argument: in any best schedule, replace its first meeting by the meeting that ends earliest. That meeting cannot overlap the rest of the schedule (it ends no later), so the schedule stays valid and the same size. Repeating this shows the greedy schedule is as big as a best one.

</details>

4. Why do we wrap the end times in `Reverse` in `min_rooms`?

<details>
<summary>Answer</summary>

`BinaryHeap` is a max-heap: it gives the biggest item. We need the smallest end time (the room that frees up first), and `Reverse` flips the comparison so the smallest value is on top.

</details>

## Summary

- A greedy algorithm takes the best-looking choice at each step and never undoes it. It is short and fast, often "sort, then one pass".
- Greedy is correct only when the problem has the greedy-choice property. Argue it with an exchange argument, and always test it against a brute-force judge.
- A counterexample (coins 1, 3, 4 and amount 6) is enough to reject a rule; a random test against a trusted algorithm can find one for you.
- Earliest end first maximizes the number of non-overlapping meetings. Sort by start and extend the last range to merge ranges. A min-heap (`BinaryHeap` with `Reverse`) counts the rooms.
- Decide the borders of ranges (open, closed, half-open) before you write the comparison, and test the case where two ranges touch.
- The cost is usually O(n log n), dominated by the sort. If a local rule fails, use dynamic programming.

## Sources and further reading

- [The Rust Standard Library — slice::sort_by_key and sort_unstable](https://doc.rust-lang.org/std/primitive.slice.html#method.sort_by_key) — the sorting methods used here.
- [The Rust Standard Library — BinaryHeap](https://doc.rust-lang.org/std/collections/struct.BinaryHeap.html) — `push`, `pop`, `peek`, and the note about using `Reverse` for a min-heap.
- [The Rust Standard Library — std::cmp::Reverse](https://doc.rust-lang.org/std/cmp/struct.Reverse.html) — the wrapper that flips the order.
- [The Rust Standard Library — f64::total_cmp](https://doc.rust-lang.org/std/primitive.f64.html#method.total_cmp) — sorting floating-point numbers.
- [The Rust Programming Language — 6.2 The match Control Flow Construct](https://doc.rust-lang.org/book/ch06-02-match.html) — `match`, including the guards used in `merge_ranges`.

---

[🏠 Start](../START-HERE.md) · [← 07.16 Dynamic programming](16-dynamic-programming.md) · [📋 Phase 07](README.md) · [07.18 Problem-solving practice →](18-problem-solving-practice.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
