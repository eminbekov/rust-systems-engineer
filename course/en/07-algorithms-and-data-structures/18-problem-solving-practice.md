<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 07.17 Greedy algorithms](17-greedy-algorithms.md) · [📋 Phase 07](README.md) · [07.19 Project: route finder →](19-project-route-finder.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/07-algorithms-and-data-structures/18-problem-solving-practice.md) <!-- nav -->

# Lesson 07.18 — Problem-solving practice

> **Phase 07 — Algorithms and data structures** · Lesson 18 of 20 · ⏱ about 135 minutes

## What you will learn

- A repeatable method for solving an algorithm problem: understand, small examples, brute force, find the waste, a better algorithm, tests, benchmark.
- How to read the size of the input and choose the complexity you can afford.
- How to turn a slow solution into a fast one by looking for the repeated work.
- Five interview-style problems on stacks, sliding windows, heaps, graphs and dynamic programming, each with tests and a stated complexity.

## Before you start

- You finished [Lesson 07.17 — Greedy algorithms](17-greedy-algorithms.md) and the lessons before it: this lesson uses stacks and queues (07.5), hash tables (07.6), BFS (Breadth-First Search, 07.13), heaps (07.12), sliding windows (07.15) and dynamic programming (07.16).
- You work in `code/p07-algorithms`. This lesson adds the module `src/practice.rs` (and `pub mod practice;` in `src/lib.rs`) and one benchmark.
- Try every problem yourself **before** you open the hints. The hints are in steps: open one, try again, open the next. The goal of this lesson is the habit, not the answers.

## New words

| Word | Meaning |
|---|---|
| **interview-style problem** | A short, self-contained question ("find …", "count …") with a clear input and output, of the kind asked in job interviews. It tests thinking, not memory of a library. |
| **brute force** | The simplest correct solution: try every possibility. It is slow but easy to get right, so it is the best judge for your faster solution. |
| **judge** | In this course: a slow, obviously correct function (or a standard library call) that the tests use to check a fast function on many inputs. |
| **constraint** | A limit on the input given with the problem, for example "at most 100,000 numbers". It tells you which complexity is fast enough. |
| **bottleneck** | The part of an algorithm that costs most of the time. Make that part faster first. |
| **trade-off** | Getting one thing by paying with another, for example speed for memory: a hash map makes the search fast but uses O(n) extra memory. |

## 1. The method

A beginner reads a problem, starts typing, gets stuck and starts again. The method below replaces that with seven small steps. Each step has one question.

| Step | Question | Output |
|---|---|---|
| 1. Understand | What are the input and the output? What are the limits? What if the input is empty? | A restated problem in your own words, and the constraints. |
| 2. Small examples | What are the answers for three tiny inputs, including an edge case? | A few `assert_eq!` lines, written before the code. |
| 3. Brute force | What is the simplest correct solution, however slow? | A working function and its complexity. |
| 4. Find the waste | What does the brute force compute again and again, or search for slowly? | A sentence: "it repeats …" or "it searches … linearly". |
| 5. Better algorithm | Which tool removes the waste? Sorting, a hash map, two pointers, a heap, a table? | A faster function and its complexity. |
| 6. Tests | Does the fast function agree with the brute force on thousands of small random inputs? | A judge test, plus the edge cases from step 2. |
| 7. Benchmark | How much faster is it, on a realistic size? | Two numbers from `criterion` (only if speed matters). |

**Read the limits (step 1).** The size `n` of the input tells you which complexity is fast enough. A computer does roughly 10⁸ to 10⁹ simple steps per second, so:

| Input size `n` | Complexity you can afford | Typical tools |
|---|---|---|
| up to about 20 | O(2ⁿ), O(n!) | try all subsets or orders (brute force) |
| up to about 1,000 to 5,000 | O(n²) | nested loops, a 2D table |
| up to about 100,000 to 1,000,000 | O(n log n) | sorting, heaps, binary search |
| more than 1,000,000 | O(n) or O(log n) | one pass, hash map, two pointers |

> 💡 **Tip:** If you are stuck at step 5, go back to step 3 and look at the brute force again. The better algorithm is almost always hiding in its slowest line.

## 2. A worked example: two numbers that add up

**1. Understand.** A slice of `i32` numbers (not sorted) and a `target`. Return two *different* positions `i < j` with `values[i] + values[j] == target`, or `None`. Limit: up to 100,000 numbers. (In Lesson 07.15 the slice was sorted; now it is not, so two pointers do not apply.)

**2. Small examples.** `[2, 7, 11, 15]` with target 9 gives positions `(0, 1)`. `[3, 3]` with 6 gives `(0, 1)`: the same *value* twice is fine, the same *position* twice is not. `[1, 2, 3]` with 100 gives `None`. The empty slice gives `None`.

**3. Brute force.** Try every pair:

```rust
/// Two different positions `i < j` with `values[i] + values[j] == target`.
/// Brute force: try every pair. Time O(n * n), space O(1).
pub fn two_sum_slow(values: &[i32], target: i32) -> Option<(usize, usize)> {
    for i in 0..values.len() {
        for j in (i + 1)..values.len() {
            if i64::from(values[i]) + i64::from(values[j]) == i64::from(target) {
                return Some((i, j));
            }
        }
    }
    None
}
```

**What happens:** two nested loops visit about n²/2 pairs: **O(n²) time**, O(1) space. The sum is computed in `i64` so that it cannot overflow. For 100,000 numbers this is about five billion steps: too slow.

**4. Find the waste.** For each number `values[j]` the inner loop *searches the rest of the slice* for the partner `target - values[j]`. A search in a slice costs O(n). A hash map answers "have I seen this value, and where?" in O(1) on average.

```text
values = [2, 7, 11, 15], target = 9

j=0  value 2  partner 7   seen = {}          -> not found; remember 2 -> 0
j=1  value 7  partner 2   seen = {2: 0}      -> found at 0  => answer (0, 1)
```

**5. Better algorithm.** Walk through the slice once. For each value, ask the map for its partner; if it is there we are done, otherwise remember the current value and its position.

```rust
/// The same, with a hash map from value to position.
/// Time O(n) on average, space O(n).
pub fn two_sum(values: &[i32], target: i32) -> Option<(usize, usize)> {
    let mut seen: HashMap<i32, usize> = HashMap::new();
    for (j, &value) in values.iter().enumerate() {
        // the partner we are looking for (None if it does not fit into i32)
        let partner = i32::try_from(i64::from(target) - i64::from(value)).ok();
        if let Some(&i) = partner.and_then(|p| seen.get(&p)) {
            return Some((i, j));
        }
        seen.insert(value, j);
    }
    None
}
```

**What happens:**

1. `seen` maps a value to the position where we saw it. At the start of step `j` it contains only positions smaller than `j`, so every answer has `i < j`.
2. `i32::try_from(...)` turns the `i64` partner back into an `i32`; it gives `Err` (and `.ok()` gives `None`) when the partner does not fit, in which case no number in the slice can be it.
3. `partner.and_then(|p| seen.get(&p))` looks the partner up; the pattern `Some(&i)` copies the position out of the reference.
4. The lookup happens *before* the insert, so a number cannot pair with itself. For `[3, 3]` the first 3 is stored, and the second 3 finds it.
5. One pass with O(1) average map operations: **O(n) time on average**, **O(n) space** for the map. This is the trade-off: time paid with memory ([Lesson 07.6](06-hash-tables.md) explains why a hash map lookup is O(1) on average and O(n) in the worst case).

**6. Tests.** Add a `tests` module. It uses the `Rng` generator of `src/testutil.rs` with the `range` method from [Lesson 07.15](15-two-pointers-and-sliding-window.md) (`range` works with `i64`), and adds a small helper `random_text` that makes a random text from the letters of an alphabet; the later problems use both:

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use crate::testutil::Rng;
    use std::collections::HashSet;

    /// A random text of `len` characters taken from `alphabet`.
    fn random_text(rng: &mut Rng, alphabet: &str, len: i64) -> String {
        let letters: Vec<char> = alphabet.chars().collect();
        (0..len)
            .map(|_| letters[rng.range(0, letters.len() as i64 - 1) as usize])
            .collect()
    }

    #[test]
    fn two_sum_examples() {
        assert_eq!(two_sum(&[2, 7, 11, 15], 9), Some((0, 1)));
        assert_eq!(two_sum(&[3, 3], 6), Some((0, 1)));
        assert_eq!(two_sum(&[1, 2, 3], 100), None);
        assert_eq!(two_sum(&[], 0), None);
        assert_eq!(two_sum(&[i32::MAX, i32::MIN], -1), Some((0, 1)));
    }

    #[test]
    fn two_sum_agrees_with_brute_force() {
        let mut rng = Rng::new(1);
        for _ in 0..2000 {
            let len = rng.range(0, 10);
            let v: Vec<i32> = (0..len).map(|_| rng.range(-10, 10) as i32).collect();
            let target = rng.range(-20, 20) as i32;
            let fast = two_sum(&v, target);
            assert_eq!(fast.is_some(), two_sum_slow(&v, target).is_some(), "{v:?}");
            if let Some((i, j)) = fast {
                assert!(i < j && v[i] + v[j] == target);
            }
        }
    }
}
```

(`HashSet` is used by tests in the exercises.) The last example in `two_sum_examples` is an edge case: `i32::MAX + i32::MIN` is `-1`, and a naive `target - value` in `i32` would overflow. The fast function and the brute force may return *different* pairs when several exist, so the random test compares only "is there an answer?" and then checks that the fast pair adds up.

**7. Benchmark.** Both versions must look at everything when no pair exists. We use 5,000 even numbers and an odd target. Add to `Cargo.toml` `[[bench]]` with `name = "practice"` and `harness = false` (criterion is in `[dev-dependencies]` already), and create `benches/practice.rs`:

```rust
use criterion::{Criterion, criterion_group, criterion_main};
use p07_algorithms::practice::{two_sum, two_sum_slow};
use std::hint::black_box;

fn bench_two_sum(c: &mut Criterion) {
    // no pair adds up to the target, so both versions must look at everything
    let values: Vec<i32> = (0..5_000).map(|i| i * 2).collect();
    let target = 1; // an odd target, all numbers are even
    let mut group = c.benchmark_group("two_sum n = 5000");
    group.bench_function("brute force", |b| {
        b.iter(|| two_sum_slow(black_box(&values), black_box(target)))
    });
    group.bench_function("hash map", |b| {
        b.iter(|| two_sum(black_box(&values), black_box(target)))
    });
    group.finish();
}

criterion_group!(benches, bench_two_sum);
criterion_main!(benches);
```

```bash
cargo bench --bench practice -- --warm-up-time 1 --measurement-time 3
```

```text
two_sum n = 5000/brute force
                        time:   [3.4147 ms 3.5832 ms 3.7948 ms]
two_sum n = 5000/hash map
                        time:   [157.32 µs 171.30 µs 188.69 µs]
```

(An Apple M4 Mac; your numbers will differ.) The hash map is about 21 times faster for n = 5,000, and the gap grows with n, because O(n²) grows faster than O(n). Notice that it is *not* 5,000 times faster: hashing and allocating cost real time. For tiny inputs (a dozen numbers) the brute force can even win. That is why step 7 exists.

> ⚡ **Performance:** An asymptotic improvement (O(n²) to O(n)) is the big win only when `n` is big. Always check the constraints, and measure on a realistic size before you "optimize".

## 3. Five problems

The five problems below are your exercises. For each: restate it, write three examples as tests, write a brute force or a trusted judge, then the real solution. **State the time and space complexity before you open the answer**; every answer gives its own. Put your solutions in `src/practice.rs` (the answers use the names shown).

## Common mistakes

**1. Not handling the empty input.** `values[0]` on an empty slice:

```rust
fn max_subarray_sum(values: &[i64]) -> i64 {
    let mut best = values[0];
    let mut ending_here = values[0];
    for &value in &values[1..] {
        ending_here = value.max(ending_here + value);
        best = best.max(ending_here);
    }
    best
}

fn main() {
    println!("{}", max_subarray_sum(&[-2, 1, -3, 4]));
    println!("{}", max_subarray_sum(&[]));
}
```

```text
4

thread 'main' (4806768) panicked at src/main.rs:2:20:
index out of bounds: the len is 0 but the index is 0
```

**What it means:** step 2 of the method asks for an edge case for exactly this reason. **Fix:** return an `Option` and use `split_first()?` as in Exercise 5 below.

**2. Depending on the order of a `HashMap`.** A `HashMap` does not promise any order, and Rust deliberately varies it between runs. The same program, run four times:

```rust
use std::collections::HashMap;

fn main() {
    let mut groups: HashMap<&str, usize> = HashMap::new();
    for key in ["aet", "ant", "abt"] {
        groups.insert(key, key.len());
    }
    let keys: Vec<_> = groups.keys().collect();
    println!("{keys:?}");
}
```

```text
["aet", "ant", "abt"]
["abt", "aet", "ant"]
["ant", "abt", "aet"]
["ant", "abt", "aet"]
```

**What it means:** a test that compares a result built from a `HashMap` iteration with a fixed expected list will fail from time to time (a flaky test). **Fix:** sort before you return or compare, or use a `BTreeMap`, which keeps its keys in order.

**3. Starting to code before the examples.** If you cannot write three examples with their answers, you have not understood the problem yet. Writing the examples as tests first also gives you a free check for the brute force.

## Try it yourself

1. Take the worked example and *change the problem*: return **all** pairs of positions that add up to the target (not only the first). Walk through the seven steps in writing: what changes in the brute force, in the waste, in the better algorithm, in the complexity (hint: the number of answers can itself be O(n²))?
2. Run `cargo test` after you have added `two_sum` and its tests; then `cargo bench --bench practice -- --warm-up-time 1 --measurement-time 3` and write down the two numbers.
3. `cargo fmt && cargo clippy --all-targets -- -D warnings && cargo test`, then `git add code/p07-algorithms && git commit -m "feat(p07): practice problems"` after each finished problem if you like (one small commit per problem keeps the history readable).

## Exercises

### Exercise 1 — Balanced brackets (easy)

A text may contain the brackets `(`, `)`, `[`, `]`, `{`, `}` and any other characters. The brackets are *balanced* when every closing bracket matches the most recent still-open bracket of the **same kind** and nothing stays open at the end. Write `pub fn is_balanced(text: &str) -> bool`. Examples: `"(a[b]{c})"` is balanced; `"(]"`, `"(()"`, `")("` and `"]"` are not; `""` is. State the complexity.

<details>
<summary>Hint 1</summary>

The most recent still-open bracket: which data structure gives you "last in, first out" ([Lesson 07.5](05-stacks-and-queues.md))?

</details>

<details>
<summary>Hint 2</summary>

Push every opening bracket. For a closing bracket, pop and compare with the expected opening one. What if the stack is empty (`pop` gives `None`)? What is left in the stack at the end?

</details>

<details>
<summary>Answer</summary>

```rust
/// Exercise 1: are the brackets `()`, `[]`, `{}` in the text balanced?
/// Time O(n), space O(n) for the stack.
pub fn is_balanced(text: &str) -> bool {
    let mut stack: Vec<char> = Vec::new();
    for c in text.chars() {
        match c {
            '(' | '[' | '{' => stack.push(c),
            ')' | ']' | '}' => {
                let open = match c {
                    ')' => '(',
                    ']' => '[',
                    _ => '{',
                };
                if stack.pop() != Some(open) {
                    return false;
                }
            }
            _ => {} // other characters are ignored
        }
    }
    stack.is_empty()
}
```

Tests: examples, and a judge that removes the pairs `()`, `[]`, `{}` from the text again and again until nothing changes (slow, but obviously correct):

```rust
    #[test]
    fn brackets() {
        assert!(is_balanced(""));
        assert!(is_balanced("(a[b]{c})"));
        assert!(!is_balanced("(]"));
        assert!(!is_balanced("(()"));
        assert!(!is_balanced(")("));
        assert!(!is_balanced("]"));
    }

    #[test]
    fn brackets_agree_with_repeated_removal() {
        fn judge(text: &str) -> bool {
            let mut s: String = text.chars().filter(|c| "()[]{}".contains(*c)).collect();
            loop {
                let shorter = s.replace("()", "").replace("[]", "").replace("{}", "");
                if shorter == s {
                    return s.is_empty();
                }
                s = shorter;
            }
        }
        let mut rng = Rng::new(2);
        for _ in 0..3000 {
            let len = rng.range(0, 10);
            let text = random_text(&mut rng, "()[]{}x", len);
            assert_eq!(is_balanced(&text), judge(&text), "{text:?}");
        }
    }
```

Why it works: a closing bracket must match the *latest* unmatched opening bracket, which is the top of a stack. `stack.pop() != Some(open)` handles both mistakes at once: an empty stack (`None`) and a bracket of the wrong kind. At the end the stack must be empty. **Time O(n)**, one pass; **space O(n)** for the stack in the worst case (a text of only opening brackets). The judge is O(n²) in the worst case but fine for 10 characters.

</details>

### Exercise 2 — The longest block without a repeat (medium)

Write `pub fn longest_unique_run(text: &str) -> usize`: the length of the longest block of neighbouring characters in which no character appears twice. `"abcabcbb"` gives 3 (`"abc"`), `"bbbb"` gives 1, `"pwwkew"` gives 3 (`"wke"`), `"abba"` gives 2, `""` gives 0. State the complexity of the brute force and of your solution.

<details>
<summary>Hint 1</summary>

Brute force: check every block (start, end) for repeats: O(n³) with a set. The waste: when the window `[left, right]` has no repeats and you add one character, you only need to know whether *that* character is already inside. This is a sliding window ([Lesson 07.15](15-two-pointers-and-sliding-window.md)).

</details>

<details>
<summary>Hint 2</summary>

Remember for each character the last position where you saw it. When the new character at `right` was last seen at `position`, and that position is inside the window, move `left` to `position + 1`. Use `left.max(position + 1)`: why `max`? Think of `"abba"`.

</details>

<details>
<summary>Answer</summary>

```rust
/// Exercise 2: the length of the longest block of characters without a repeat.
/// Sliding window with the last position of each character.
/// Time O(n) on average, space O(k) for k different characters.
pub fn longest_unique_run(text: &str) -> usize {
    let mut last_seen: HashMap<char, usize> = HashMap::new();
    let mut left = 0; // the window is chars[left..=right]
    let mut best = 0;
    for (right, c) in text.chars().enumerate() {
        if let Some(&position) = last_seen.get(&c) {
            left = left.max(position + 1); // jump over the earlier copy
        }
        last_seen.insert(c, right);
        best = best.max(right - left + 1);
    }
    best
}
```

Tests:

```rust
    #[test]
    fn unique_run() {
        assert_eq!(longest_unique_run(""), 0);
        assert_eq!(longest_unique_run("abcabcbb"), 3);
        assert_eq!(longest_unique_run("bbbb"), 1);
        assert_eq!(longest_unique_run("pwwkew"), 3);
        assert_eq!(longest_unique_run("abba"), 2);
    }

    #[test]
    fn unique_run_agrees_with_brute_force() {
        let mut rng = Rng::new(3);
        for _ in 0..2000 {
            let len = rng.range(0, 12);
            let text = random_text(&mut rng, "abcd", len);
            let chars: Vec<char> = text.chars().collect();
            let mut best = 0;
            for start in 0..chars.len() {
                for end in start..chars.len() {
                    let block = &chars[start..=end];
                    if block.iter().collect::<HashSet<_>>().len() == block.len() {
                        best = best.max(block.len());
                    }
                }
            }
            assert_eq!(longest_unique_run(&text), best, "{text:?}");
        }
    }
```

Why it works: `left` never moves back (that is the `max`: in `"abba"` the last `a` was seen at position 0, which is already left of the window), and `right` moves forward once per character, so the loop is **O(n)** with O(1) average map operations. Space O(k) for k different characters. The judge is the brute force: every block, with a `HashSet` to detect repeats (O(n³), fine for 12 characters).

</details>

### Exercise 3 — The k biggest numbers (medium)

Write `pub fn k_largest(values: &[i32], k: usize) -> Vec<i32>` that returns the `k` biggest numbers, biggest first. If there are fewer than `k` numbers, return all of them. Do it in O(n log k) time without sorting the whole input, and say why that matters when `k` is small and `n` is huge (for example the top 10 of 100 million numbers).

<details>
<summary>Hint 1</summary>

Sorting everything costs O(n log n). You only need to remember the `k` biggest numbers seen so far. When a new number arrives, which one of the remembered numbers is the first to leave?

</details>

<details>
<summary>Hint 2</summary>

The one that leaves is the *smallest* of the remembered ones: a min-heap of size `k` (`BinaryHeap<Reverse<i32>>`, as in [Lesson 07.12](12-heaps-and-priority-queues.md)). Push the new number; if the heap has more than `k` items, pop. At the end sort the (at most `k`) items.

</details>

<details>
<summary>Answer</summary>

```rust
/// Exercise 3: the `k` biggest numbers, biggest first.
/// A min-heap of size `k` keeps the k biggest seen so far.
/// Time O(n log k), space O(k).
pub fn k_largest(values: &[i32], k: usize) -> Vec<i32> {
    let mut heap: BinaryHeap<Reverse<i32>> = BinaryHeap::new();
    for &value in values {
        heap.push(Reverse(value));
        if heap.len() > k {
            heap.pop(); // drop the smallest of the k + 1
        }
    }
    let mut result: Vec<i32> = heap.into_iter().map(|Reverse(v)| v).collect();
    result.sort_unstable_by(|a, b| b.cmp(a));
    result
}
```

Test, with the standard library as the judge (sort everything, take `k`):

```rust
    #[test]
    fn k_largest_examples_and_judge() {
        assert_eq!(k_largest(&[3, 1, 5, 12, 2, 11], 3), vec![12, 11, 5]);
        assert_eq!(k_largest(&[1, 2], 5), vec![2, 1]);
        assert_eq!(k_largest(&[1, 2], 0), Vec::<i32>::new());
        let mut rng = Rng::new(4);
        for _ in 0..1000 {
            let len = rng.range(0, 12);
            let v: Vec<i32> = (0..len).map(|_| rng.range(-5, 5) as i32).collect();
            let k = rng.range(0, 14) as usize;
            let mut expected = v.clone();
            expected.sort_unstable_by(|a, b| b.cmp(a));
            expected.truncate(k);
            assert_eq!(k_largest(&v, k), expected, "{v:?} k = {k}");
        }
    }
```

Why it works: the heap never holds more than `k + 1` items, so each of the `n` numbers costs O(log k) for `push` and `pop`: **time O(n log k)**, plus O(k log k) for the final sort; **space O(k)**. For the top 10 of 100 million numbers the heap has at most 11 items: the memory is tiny and the input can be read as a stream, while "sort everything" would need all numbers in memory and O(n log n) time. For `k = 0` each number is pushed and popped at once, so the result is empty.

</details>

### Exercise 4 — Count the islands (medium)

A map is a list of equal-length strings of `'1'` (land) and `'0'` (water). An *island* is a group of land cells connected up, down, left or right (not diagonally). Write `pub fn count_islands(grid: &[&str]) -> usize`. Examples: `["11000", "11000", "00100", "00011"]` has 3 islands; `["101", "010", "101"]` has 5 (diagonals do not connect). State the complexity.

<details>
<summary>Hint 1</summary>

This is a graph problem in disguise: every land cell is a vertex, and neighbouring land cells are connected by an edge ([Lesson 07.13](13-graphs-bfs-and-dfs.md)). An island is a connected component. How do you count components?

</details>

<details>
<summary>Hint 2</summary>

Scan all cells. When you find a land cell that you have not visited, count one island and visit the whole island with BFS (a queue) so that you never count its cells again. Keep a `seen` grid. A recursive DFS (Depth-First Search) can overflow the stack on a big map, so BFS or a stack of your own is safer.

</details>

<details>
<summary>Answer</summary>

```rust
/// Exercise 4: how many islands ('1' = land, '0' = water, 4 directions)?
/// Breadth-first search from each unvisited land cell.
/// Time O(rows * columns), space O(rows * columns).
pub fn count_islands(grid: &[&str]) -> usize {
    let land: Vec<Vec<bool>> = grid
        .iter()
        .map(|row| row.chars().map(|c| c == '1').collect())
        .collect();
    let mut seen: Vec<Vec<bool>> = land.iter().map(|row| vec![false; row.len()]).collect();
    let mut islands = 0;
    for r in 0..land.len() {
        for c in 0..land[r].len() {
            if !land[r][c] || seen[r][c] {
                continue;
            }
            islands += 1; // a new island: flood it so we never count it again
            seen[r][c] = true;
            let mut queue = VecDeque::from([(r, c)]);
            while let Some((row, col)) = queue.pop_front() {
                for (dr, dc) in [(-1i32, 0i32), (1, 0), (0, -1), (0, 1)] {
                    let (nr, nc) = (row as i32 + dr, col as i32 + dc);
                    if nr < 0 || nc < 0 {
                        continue;
                    }
                    let (nr, nc) = (nr as usize, nc as usize);
                    let is_land = land.get(nr).and_then(|line| line.get(nc)) == Some(&true);
                    if is_land && !seen[nr][nc] {
                        seen[nr][nc] = true;
                        queue.push_back((nr, nc));
                    }
                }
            }
        }
    }
    islands
}
```

Tests:

```rust
    #[test]
    fn islands_examples() {
        assert_eq!(count_islands(&[]), 0);
        assert_eq!(count_islands(&["000", "000"]), 0);
        assert_eq!(count_islands(&["111", "111"]), 1);
        assert_eq!(count_islands(&["11000", "11000", "00100", "00011"]), 3);
        assert_eq!(count_islands(&["101", "010", "101"]), 5); // diagonals do not connect
    }
```

Why it works: each land cell is put into the queue at most once (it is marked in `seen` *when it is queued*), and for each cell we look at 4 neighbours: **time O(rows · columns)**, **space O(rows · columns)** for `seen` and the queue. The signed `i32` arithmetic and the `nr < 0 || nc < 0` check avoid `usize` underflow on the border; `land.get(nr).and_then(|line| line.get(nc))` avoids an out-of-range index on the other borders. For a stronger test, compare with a recursive flood fill on 500 random grids of up to 5 × 5 cells; write it as practice, with the same pattern as the other judges.

</details>

### Exercise 5 — The best block sum (hard)

Write `pub fn max_subarray_sum(values: &[i64]) -> Option<i64>`: the biggest sum of a non-empty block of neighbouring numbers (`None` for an empty slice). The numbers may be negative. `[-2, 1, -3, 4, -1, 2, 1, -5, 4]` gives 6 (the block `4, -1, 2, 1`). `[-4, -2, -9]` gives `-2`. State the complexity of the brute force and of your solution.

<details>
<summary>Hint 1</summary>

Brute force: for every start, add up to every end: O(n²). The waste: the sums of blocks that start at the same place share all but one item. Dynamic programming idea ([Lesson 07.16](16-dynamic-programming.md)): let the *state* be "the best sum of a block that **ends** at position `i`".

</details>

<details>
<summary>Hint 2</summary>

The best block that ends at `i` is either the number `values[i]` alone, or the best block that ends at `i - 1`, extended by `values[i]`. So `ending_here = max(value, ending_here + value)`. The answer is the maximum of all `ending_here` values. You only need the previous value, so one variable is enough.

</details>

<details>
<summary>Answer</summary>

```rust
/// Exercise 5: the biggest sum of a block of neighbouring numbers (Kadane).
/// `None` for an empty slice. Time O(n), space O(1).
pub fn max_subarray_sum(values: &[i64]) -> Option<i64> {
    let (&first, rest) = values.split_first()?;
    let mut best = first;
    let mut ending_here = first; // the best sum of a block that ends at this item
    for &value in rest {
        ending_here = value.max(ending_here + value);
        best = best.max(ending_here);
    }
    Some(best)
}
```

Test, with the O(n²) brute force as the judge:

```rust
    #[test]
    fn kadane() {
        assert_eq!(max_subarray_sum(&[]), None);
        assert_eq!(max_subarray_sum(&[-3]), Some(-3));
        assert_eq!(max_subarray_sum(&[-2, 1, -3, 4, -1, 2, 1, -5, 4]), Some(6));
        assert_eq!(max_subarray_sum(&[-4, -2, -9]), Some(-2));
        let mut rng = Rng::new(6);
        for _ in 0..1000 {
            let len = rng.range(1, 12);
            let v: Vec<i64> = (0..len).map(|_| rng.range(-9, 9)).collect();
            let mut best = i64::MIN;
            for start in 0..v.len() {
                let mut sum = 0;
                for &x in &v[start..] {
                    sum += x;
                    best = best.max(sum);
                }
            }
            assert_eq!(max_subarray_sum(&v), Some(best), "{v:?}");
        }
    }
```

Why it works: the recurrence `ending_here = max(value, ending_here + value)` says "start a new block here, or extend the previous one"; a negative `ending_here` is never worth extending, so it is dropped. This is dynamic programming with a state reduced to one variable (the algorithm is known as *Kadane's algorithm*). **Time O(n)**, **space O(1)**. `split_first()?` returns `None` for an empty slice, so there is no `values[0]` panic. The test starts `best` at `i64::MIN`, so all-negative inputs are covered. (Sums of huge numbers could overflow `i64`; the problem limits the numbers, so we do not check here.)

</details>

## Quiz

1. What are the seven steps of the method?

<details>
<summary>Answer</summary>

Understand (input, output, limits, edge cases); small examples (as tests); brute force; find the waste; a better algorithm; tests against the brute force; benchmark.

</details>

2. The input has 200,000 numbers. Can you use an O(n²) algorithm?

<details>
<summary>Answer</summary>

No. n² is 40 billion steps, which takes tens of seconds. You need O(n log n) or O(n): sorting, a heap, a hash map, two pointers or a sliding window.

</details>

3. In `two_sum`, why is the lookup done before the insert?

<details>
<summary>Answer</summary>

So that a number cannot be paired with itself, which would break the rule that the two positions are different. With the lookup first, the map contains only earlier positions.

</details>

4. Which part of `k_largest` makes it O(n log k) and not O(n log n)?

<details>
<summary>Answer</summary>

The heap never grows beyond `k + 1` items, so each `push` and `pop` costs O(log k), not O(log n).

</details>

## Summary

- Solve a problem in seven steps: understand, small examples, brute force, find the waste, a better algorithm, tests, benchmark.
- The input size decides the complexity you can afford: about 20 means exponential is fine, a few thousand means O(n²), a few hundred thousand means O(n log n), more means O(n).
- The better algorithm hides in the slowest line of the brute force: a repeated search becomes a hash map lookup, a repeated sum becomes a sliding window, a repeated minimum becomes a heap.
- Always keep the brute force as a judge and compare on thousands of random small inputs; write the edge cases (empty, one item, duplicates, borders) as tests first.
- Do not depend on the order of a `HashMap`; sort the output.
- Speed-ups are worth the most when `n` is large; measure before you decide.

## Sources and further reading

- [The Rust Standard Library — std::collections](https://doc.rust-lang.org/std/collections/index.html) — `HashMap`, `HashSet`, `BinaryHeap`, `VecDeque` and a guide to choosing between them.
- [The Rust Standard Library — HashMap](https://doc.rust-lang.org/std/collections/struct.HashMap.html) — `entry`, `or_default`, `get`, and the note that the iteration order is arbitrary.
- [The Rust Standard Library — BinaryHeap](https://doc.rust-lang.org/std/collections/struct.BinaryHeap.html) — the heap used in `k_largest`.
- [The Rust Programming Language — 8.3 Storing Keys with Associated Values in Hash Maps](https://doc.rust-lang.org/book/ch08-03-hash-maps.html) — the official chapter on hash maps.
- [Criterion.rs User Guide](https://bheisler.github.io/criterion.rs/book/) — the benchmark library used for step 7.

---

[🏠 Start](../START-HERE.md) · [← 07.17 Greedy algorithms](17-greedy-algorithms.md) · [📋 Phase 07](README.md) · [07.19 Project: route finder →](19-project-route-finder.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
