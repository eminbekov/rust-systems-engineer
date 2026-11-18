<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 07.19 Project: route finder](19-project-route-finder.md) · [📋 Phase 07](README.md) · [Phase 08 — Software design: principles, patterns and conventions →](../08-software-design/README.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/07-algorithms-and-data-structures/20-checkpoint.md) <!-- nav -->

# Lesson 07.20 — Checkpoint

> **Phase 07 — Algorithms and data structures** · Lesson 20 of 20 · ⏱ about 120 minutes

## What you will learn

This lesson teaches nothing new. It checks that you can use what Phase 07 taught, and it shows you where to go back if something is still weak. You will:

- Answer 20 questions about Big-O, vectors, lists, stacks, queues, hash tables, recursion, sorting, searching, trees, heaps, graphs and the techniques of the last lessons.
- Solve 5 practical tasks: four with tests, and one table of complexities from memory.
- Find the lessons you should repeat, with direct links.

## Before you start

- You finished [Lesson 07.19 — Project: route finder](19-project-route-finder.md). Your `route-finder` works, its tests pass, and it is committed.
- Do the quiz **closed book**: no lessons, no compiler, no search. Write your answer on paper or in your notes first, then open the hidden answer. A wrong answer found now is cheap.
- For the practical tasks you may use the compiler and the lessons, but try first without hints. Put the code of Tasks 1 to 4 in a new module `src/checkpoint.rs` of `code/p07-algorithms` (add `pub mod checkpoint;` to `src/lib.rs`). Task 4 uses `min_coins` from Lesson 07.16 and `greedy_coins` from Lesson 07.17.
- Your learning repository is clean (`git status` in `~/projects/rust`).

## New words

| Word | Meaning |
|---|---|
| **checkpoint** | A pause at the end of a phase to test yourself before you go on. |
| **self-test** | A test that you give yourself and mark yourself, to find gaps. |
| **closed book** | Answering from memory only, without looking anything up. |
| **quality gate** | The commands that must pass before you accept your own work: `cargo fmt --check`, `cargo clippy --all-targets -- -D warnings`, `cargo test`. |

## Self-test quiz

Mark one point for each answer that is **correct and complete**. Be honest: nobody checks but you.

### Big-O, vectors, lists, stacks, queues, hash tables

1. What does "O(n²)" say about an algorithm, and why do we write O(n²) for a running time of 3n² + 10n + 7?

<details>
<summary>Answer</summary>

It says that the running time grows about with the square of the input size `n`: ten times more input means about a hundred times more work. For big `n` the biggest term dominates (3n² beats 10n and 7), and constant factors like 3 depend on the computer and the details of the code, so Big-O drops them ([Lesson 07.1](01-algorithms-and-big-o.md)).

</details>

2. What is the time complexity of `while n > 1 { n /= 2; }`, and of two nested loops where the inner loop runs `i` times for each `i` in `0..n`?

<details>
<summary>Answer</summary>

The first loop halves `n` each time: about log₂ n steps, so **O(log n)**. The second does 0 + 1 + 2 + … + (n − 1) = n(n − 1) / 2 steps, so **O(n²)** ([Lesson 07.2](02-analyzing-time-and-space.md)).

</details>

3. Why is `Vec::push` called O(1) *amortized* although some pushes copy all items? And what does `Vec::insert(0, x)` cost?

<details>
<summary>Answer</summary>

When the vector is full, it allocates a bigger buffer (the capacity grows by a factor) and copies the items. That is rare, so the cost averaged over many pushes is constant. `insert(0, x)` shifts every item one place to the right: **O(n)** ([Lesson 07.3](03-arrays-and-vectors-under-the-hood.md)).

</details>

4. When is a linked list better than a `Vec`, and why is it still rarely the right choice in Rust?

<details>
<summary>Answer</summary>

Inserting or removing at a node you already hold (or at the front) is O(1), with no shifting. But access by position is O(n), every node is a separate allocation (slow for the cache of the CPU, the Central Processing Unit, and more memory), and ownership makes lists harder to write in Rust (`Box`, `Option`, `Rc`). A `Vec` or a `VecDeque` is usually faster and simpler ([Lesson 07.4](04-linked-lists.md)).

</details>

5. Which standard types give you a stack and a queue, and why is `Vec::remove(0)` a bad way to take from the front of a queue?

<details>
<summary>Answer</summary>

A `Vec` is a stack (`push` and `pop` at the end, both O(1)). A `VecDeque` is a queue (`push_back` and `pop_front`, both O(1)). `Vec::remove(0)` shifts all other items, which costs O(n) every time ([Lesson 07.5](05-stacks-and-queues.md)).

</details>

6. What are the average and the worst-case lookup time of a `HashMap`, what is a collision, and why is the iteration order not fixed?

<details>
<summary>Answer</summary>

Average **O(1)**, worst case **O(n)** when many keys end up in the same place. A collision is two different keys that the hash function sends to the same slot, so the table must handle both. The order of iteration depends on the hash values and the table size (and Rust's default hasher is seeded randomly), so it can change between runs: sort when you need a fixed order ([Lesson 07.6](06-hash-tables.md)).

</details>

### Recursion, sorting and searching

7. What two parts does every correct recursive function have, and what happens without one of them?

<details>
<summary>Answer</summary>

A base case that returns without recursing, and a recursive case that moves towards the base case. Without a base case the function calls itself until the call stack is full: "thread has overflowed its stack" ([Lesson 07.7](07-recursion.md)).

</details>

8. What are the best and the worst case of insertion sort, and where is it still used?

<details>
<summary>Answer</summary>

Best case O(n) (the input is already sorted), worst case O(n²) (reverse order); it is stable and works in place. Because it has almost no overhead, real sorting functions use it for very short pieces ([Lesson 07.8](08-simple-sorting-algorithms.md)).

</details>

9. State the time and the extra space of merge sort, and the best, average and worst case of quicksort.

<details>
<summary>Answer</summary>

Merge sort: O(n log n) in every case, O(n) extra space, stable. Quicksort: O(n log n) on average, O(n²) in the worst case (always unlucky pivots), nearly in place, not stable ([Lesson 07.9](09-merge-sort-and-quicksort.md)).

</details>

10. What is the difference between `sort` and `sort_unstable` in the standard library?

<details>
<summary>Answer</summary>

`sort` is stable: equal items keep their original order. `sort_unstable` makes no such promise, which usually makes it a little faster and lets it avoid extra memory. Use `sort_unstable` when equal items are indistinguishable ([Lesson 07.9](09-merge-sort-and-quicksort.md)).

</details>

11. What must be true to use binary search, what is its time, and what do `Ok(i)` and `Err(i)` mean in the result of `binary_search`?

<details>
<summary>Answer</summary>

The data must be sorted (and random access must be cheap). Time **O(log n)**. `Ok(i)` means that the value is at index `i` (with equal values, any of them). `Err(i)` means that it is not there and `i` is the place where it could be inserted to keep the order. The midpoint should be computed as `lo + (hi - lo) / 2` to avoid overflow ([Lesson 07.10](10-binary-search.md)).

</details>

### Trees, heaps and graphs

12. What is the cost of a lookup in a binary search tree that is balanced, and in one that is a degenerate chain? What does an in-order traversal produce?

<details>
<summary>Answer</summary>

Balanced: **O(log n)**. Degenerate (for example after inserting already sorted keys): **O(n)**, because the tree is a list. An in-order traversal (left subtree, node, right subtree) visits the keys in sorted order ([Lesson 07.11](11-trees-and-binary-search-trees.md)).

</details>

13. What is on top of Rust's `BinaryHeap`, what do `peek`, `push` and `pop` cost, how do you get a min-heap, and is this heap the heap memory of Phase 03?

<details>
<summary>Answer</summary>

The biggest item (it is a max-heap). `peek` is O(1); `push` and `pop` are O(log n). A min-heap is made by wrapping items in `std::cmp::Reverse`. No: the heap *data structure* (a tree stored in a vector) has nothing to do with the heap *memory* where `Box` and `Vec` buffers live ([Lesson 07.12](12-heaps-and-priority-queues.md)).

</details>

14. Which data structure drives BFS (Breadth-First Search) and which drives DFS (Depth-First Search)? Which one finds the path with the fewest edges in an unweighted graph? What is their time?

<details>
<summary>Answer</summary>

BFS uses a queue and DFS a stack (or recursion, which is the call stack). BFS finds the path with the fewest edges. Both take **O(V + E)** time for V vertices and E edges with an adjacency list ([Lesson 07.13](13-graphs-bfs-and-dfs.md)).

</details>

15. What does Dijkstra's algorithm require, what is its time with a binary heap, and what does a topological sort require?

<details>
<summary>Answer</summary>

Dijkstra requires non-negative edge weights; its time is **O((V + E) log V)**. A topological sort needs a directed graph without cycles (a DAG, Directed Acyclic Graph): with a cycle there is no order in which every vertex comes after the ones it depends on ([Lesson 07.14](14-shortest-paths-and-topological-sort.md)).

</details>

### Techniques, practice and the project

16. A two-pointers loop contains a `while` inside a `for`. Why can it still be O(n), and what does the technique need from the data?

<details>
<summary>Answer</summary>

The pointers never move back: each of them moves forward at most `n` times in the whole run, so the total number of moves is at most 2n (an amortized argument). The technique needs a property that justifies dropping a candidate for good: sorted data for pointers from both ends, non-negative numbers for a shrinking window ([Lesson 07.15](15-two-pointers-and-sliding-window.md)).

</details>

17. What makes a problem suitable for dynamic programming, and what are the two styles? Why is the naive Fibonacci recursion slow?

<details>
<summary>Answer</summary>

Overlapping subproblems (and optimal substructure). Top-down: recursion plus a memo; bottom-up: a table filled from the smallest state. The naive recursion solves the same `fib(k)` again and again, so the number of calls grows exponentially (about 1.6ⁿ), while the dynamic programming (DP) versions do O(n) work ([Lesson 07.16](16-dynamic-programming.md)).

</details>

18. When is a greedy algorithm correct? Give a counterexample for "biggest coin first".

<details>
<summary>Answer</summary>

When the problem has the greedy-choice property, which you can prove with an exchange argument (and check against a brute-force judge). Counterexample: coins 1, 3, 4 and amount 6. Greedy takes 4 + 1 + 1 (3 coins), while 3 + 3 needs only 2 ([Lesson 07.17](17-greedy-algorithms.md)).

</details>

19. The input has 200,000 numbers and your brute force is O(n²). What do you do, according to the method of Lesson 07.18?

<details>
<summary>Answer</summary>

n² is 40 billion steps: too slow. Look at the slowest line of the brute force and find the waste (a repeated search, a repeated sum, a repeated minimum), and replace it with a tool that removes it (hash map, sorting, two pointers or a window, a heap, a table) to reach O(n log n) or O(n). Keep the brute force as the judge for the tests, then measure ([Lesson 07.18](18-problem-solving-practice.md)).

</details>

20. In your route finder, why does BFS print 23 km and Dijkstra 20 km from `Alder` to `Elm`?

<details>
<summary>Answer</summary>

BFS minimizes the number of roads (2 roads, 14 + 9 km) and ignores their lengths. Dijkstra minimizes the sum of lengths (3 roads, 9 + 2 + 9 km) ([Lesson 07.19](19-project-route-finder.md)).

</details>

## Practical tasks

Write the code in `src/checkpoint.rs`. For each task: tests first (examples, edge cases, a comparison with the standard library or a slow judge where it makes sense), then the code, then the quality gate.

### Task 1 — Binary search by hand (Lessons 07.10 and 07.15)

Write `lower_bound(sorted: &[i32], x: i32) -> usize` (the first index whose value is **not smaller** than `x`; the length of the slice if there is none), `upper_bound` (the first index whose value is **bigger** than `x`), and `count_equal(sorted, x)` that uses both. Do not use `binary_search` or `partition_point` in the code; use `partition_point` as the judge in a test on many random sorted slices. State the time complexity.

<details>
<summary>Hint</summary>

Keep `lo` and `hi` so that the answer is always in `lo..=hi`. While `lo < hi`: take `mid = lo + (hi - lo) / 2`; if `sorted[mid] < x` the answer is to the right (`lo = mid + 1`), otherwise `mid` could be the answer (`hi = mid`). For `upper_bound` use `<=` instead of `<`. Test `i32::MAX` as a value: why must `upper_bound` not compute `lower_bound(x + 1)`?

</details>

<details>
<summary>Answer</summary>

```rust
/// Task 1: the first index whose value is NOT smaller than `x`
/// (so `sorted.len()` if all values are smaller). Time O(log n).
pub fn lower_bound(sorted: &[i32], x: i32) -> usize {
    let (mut lo, mut hi) = (0, sorted.len()); // the answer is in lo..=hi
    while lo < hi {
        let mid = lo + (hi - lo) / 2; // no overflow, unlike (lo + hi) / 2
        if sorted[mid] < x {
            lo = mid + 1;
        } else {
            hi = mid;
        }
    }
    lo
}

/// The first index whose value is bigger than `x`.
pub fn upper_bound(sorted: &[i32], x: i32) -> usize {
    let (mut lo, mut hi) = (0, sorted.len());
    while lo < hi {
        let mid = lo + (hi - lo) / 2;
        if sorted[mid] <= x {
            lo = mid + 1;
        } else {
            hi = mid;
        }
    }
    lo
}

/// How many times `x` occurs in a sorted slice. Time O(log n).
pub fn count_equal(sorted: &[i32], x: i32) -> usize {
    upper_bound(sorted, x) - lower_bound(sorted, x)
}
```

Tests (in the `tests` module of `src/checkpoint.rs`, which starts with `use super::*;` and `use crate::testutil::Rng;`; the `range` method of `Rng` comes from Lesson 07.15):

```rust
    #[test]
    fn bounds_examples() {
        let v = [1, 2, 2, 2, 5];
        assert_eq!(lower_bound(&v, 2), 1);
        assert_eq!(upper_bound(&v, 2), 4);
        assert_eq!(count_equal(&v, 2), 3);
        assert_eq!(lower_bound(&v, 0), 0);
        assert_eq!(lower_bound(&v, 9), 5);
        assert_eq!(count_equal(&[], 1), 0);
        assert_eq!(upper_bound(&[i32::MAX], i32::MAX), 1);
    }

    #[test]
    fn bounds_agree_with_partition_point() {
        let mut rng = Rng::new(10);
        for _ in 0..1000 {
            let len = rng.range(0, 12);
            let mut v: Vec<i32> = (0..len).map(|_| rng.range(-5, 5) as i32).collect();
            v.sort();
            let x = rng.range(-6, 6) as i32;
            assert_eq!(
                lower_bound(&v, x),
                v.partition_point(|&e| e < x),
                "{v:?} {x}"
            );
            assert_eq!(
                upper_bound(&v, x),
                v.partition_point(|&e| e <= x),
                "{v:?} {x}"
            );
            assert_eq!(count_equal(&v, x), v.iter().filter(|&&e| e == x).count());
        }
    }
```

Why it works: the loop halves the range `lo..hi` each time, so it runs about log₂ n times: **O(log n)**; `count_equal` is two searches, also O(log n). `x + 1` would overflow for `i32::MAX`, so `upper_bound` has its own loop.

</details>

### Task 2 — A calculator with a stack (Lesson 07.5)

*Reverse Polish notation* writes the operator after its two numbers: `3 4 +` means 3 + 4, and `3 4 + 2 *` means (3 + 4) · 2. Write `eval_rpn(tokens: &[&str]) -> Result<i64, String>` for the operators `+ - * /` and whole numbers. It must return an error (not panic) for: an operator without two numbers, division by zero, a token that is not a number, overflow, leftover numbers, and an empty expression. State the complexity.

<details>
<summary>Hint</summary>

Walk through the tokens with a `Vec<i64>` as the stack. A number is pushed. An operator pops the **second** operand first (`b`), then the first (`a`), and pushes `a op b`. Use `checked_add`, `checked_sub`, `checked_mul` and `checked_div`. At the end exactly one number must remain.

</details>

<details>
<summary>Answer</summary>

```rust
/// Task 2: evaluates an expression in reverse Polish notation (the operator
/// comes after its two numbers): `["3", "4", "+", "2", "*"]` is (3 + 4) * 2.
/// Time O(n), space O(n).
pub fn eval_rpn(tokens: &[&str]) -> Result<i64, String> {
    let mut stack: Vec<i64> = Vec::new();
    for &token in tokens {
        match token {
            "+" | "-" | "*" | "/" => {
                let b = stack.pop().ok_or("not enough numbers")?;
                let a = stack.pop().ok_or("not enough numbers")?;
                let result = match token {
                    "+" => a.checked_add(b),
                    "-" => a.checked_sub(b),
                    "*" => a.checked_mul(b),
                    _ => {
                        if b == 0 {
                            return Err("division by zero".to_string());
                        }
                        a.checked_div(b)
                    }
                };
                stack.push(result.ok_or("overflow")?);
            }
            number => {
                let value = number
                    .parse::<i64>()
                    .map_err(|_| format!("bad token {number:?}"))?;
                stack.push(value);
            }
        }
    }
    match stack.as_slice() {
        [value] => Ok(*value),
        [] => Err("empty expression".to_string()),
        _ => Err("too many numbers".to_string()),
    }
}
```

Test:

```rust
    #[test]
    fn rpn() {
        assert_eq!(eval_rpn(&["3", "4", "+", "2", "*"]), Ok(14));
        assert_eq!(
            eval_rpn(&["5", "1", "2", "+", "4", "*", "+", "3", "-"]),
            Ok(14)
        );
        assert_eq!(eval_rpn(&["7"]), Ok(7));
        assert_eq!(eval_rpn(&["7", "2", "-"]), Ok(5));
        assert_eq!(eval_rpn(&["+"]), Err("not enough numbers".to_string()));
        assert_eq!(
            eval_rpn(&["1", "0", "/"]),
            Err("division by zero".to_string())
        );
        assert_eq!(
            eval_rpn(&["1", "x", "+"]),
            Err("bad token \"x\"".to_string())
        );
        assert_eq!(eval_rpn(&["1", "2"]), Err("too many numbers".to_string()));
        assert_eq!(eval_rpn(&[]), Err("empty expression".to_string()));
        assert_eq!(
            eval_rpn(&["9223372036854775807", "1", "+"]),
            Err("overflow".to_string())
        );
    }
```

Why it works: each token is handled once in O(1): **time O(n)**, **space O(n)** for the stack. Popping `b` before `a` matters for `-` and `/`. The `ok_or("…")?` lines turn an `Option` into a `Result` and convert the `&str` into a `String` through `From`.

</details>

### Task 3 — The shortest way out of a maze (Lesson 07.13)

A maze is a list of strings: `S` is the start, `E` the exit, `#` a wall and `.` an open cell. You move up, down, left or right. Write `maze_steps(maze: &[&str]) -> Option<usize>`, the fewest steps from `S` to `E`, or `None` (no `S` or no `E`, or no way). Test it with at least: a maze with a wall in the way, an unreachable exit, and an open room where the answer equals the Manhattan distance (|row difference| + |column difference|). State the complexity.

<details>
<summary>Hint</summary>

The cells are the vertices and neighbouring open cells are connected by edges, all of the same length: BFS gives the fewest steps ([Lesson 07.13](13-graphs-bfs-and-dfs.md)). Keep a grid of `Option<usize>` with the number of steps to each cell. The first time BFS takes `E` out of the queue its step count is the answer. Beware of `usize` underflow on the first row and column: use `i32` for the neighbour arithmetic.

</details>

<details>
<summary>Answer</summary>

```rust
/// Task 3: the fewest steps from 'S' to 'E' in a maze ('#' is a wall,
/// '.' is open), moving up, down, left or right. `None` if there is no way.
/// Time O(rows * columns), space O(rows * columns).
pub fn maze_steps(maze: &[&str]) -> Option<usize> {
    let grid: Vec<Vec<char>> = maze.iter().map(|row| row.chars().collect()).collect();
    let find = |target: char| {
        grid.iter()
            .enumerate()
            .find_map(|(r, row)| row.iter().position(|&c| c == target).map(|c| (r, c)))
    };
    let (start, goal) = (find('S')?, find('E')?);
    let mut steps: Vec<Vec<Option<usize>>> = grid.iter().map(|row| vec![None; row.len()]).collect();
    steps[start.0][start.1] = Some(0);
    let mut queue = VecDeque::from([start]);
    while let Some((r, c)) = queue.pop_front() {
        if (r, c) == goal {
            return steps[r][c];
        }
        let here = steps[r][c]?;
        for (dr, dc) in [(-1i32, 0i32), (1, 0), (0, -1), (0, 1)] {
            let (nr, nc) = (r as i32 + dr, c as i32 + dc);
            if nr < 0 || nc < 0 {
                continue;
            }
            let (nr, nc) = (nr as usize, nc as usize);
            let open = grid
                .get(nr)
                .and_then(|row| row.get(nc))
                .is_some_and(|&ch| ch != '#');
            if open && steps[nr][nc].is_none() {
                steps[nr][nc] = Some(here + 1);
                queue.push_back((nr, nc));
            }
        }
    }
    None
}
```

Tests:

```rust
    #[test]
    fn mazes() {
        let maze = ["S.#", "..#", "#.E"];
        assert_eq!(maze_steps(&maze), Some(4));
        assert_eq!(maze_steps(&["SE"]), Some(1));
        assert_eq!(maze_steps(&["S#E"]), None);
        assert_eq!(maze_steps(&["S.."]), None); // no exit
        assert_eq!(maze_steps(&[]), None);
        // in an open room the answer is the Manhattan distance
        let room = ["S....", ".....", "....E"];
        assert_eq!(maze_steps(&room), Some(6));
    }
```

Why it works: BFS visits the cells in order of their distance from `S`, and each cell is queued once, so the first time it reaches `E` the number of steps is the smallest. **Time O(rows · columns), space O(rows · columns).** `find('S')?` returns `None` early when there is no start or no exit.

</details>

### Task 4 — Find where greedy fails (Lessons 07.16 and 07.17)

Write `first_greedy_failure(coins: &[usize], max_amount: usize) -> Option<usize>`: the smallest amount from 1 to `max_amount` for which `greedy_coins` gives a different result than `min_coins`, or `None` if greedy is always right up to `max_amount`. Test it with the coins `[1, 5, 10, 25]` (never fails up to 300), `[1, 3, 4]` (first failure at 6) and `[1, 5, 12, 25]`. Predict the last answer by hand first.

<details>
<summary>Hint</summary>

It is one line: search the range `1..=max_amount` for the first amount where the two functions disagree (`find`). For `[1, 5, 12, 25]` try the amounts around 15: greedy takes 12 and then 1 + 1 + 1.

</details>

<details>
<summary>Answer</summary>

```rust
/// Task 4: the smallest amount (up to `max_amount`) for which "biggest coin
/// first" needs more coins than the best answer, or `None` if it never fails.
pub fn first_greedy_failure(coins: &[usize], max_amount: usize) -> Option<usize> {
    (1..=max_amount).find(|&amount| greedy_coins(coins, amount) != min_coins(coins, amount))
}
```

Test:

```rust
    #[test]
    fn greedy_failures() {
        assert_eq!(first_greedy_failure(&[1, 5, 10, 25], 300), None);
        assert_eq!(first_greedy_failure(&[1, 3, 4], 100), Some(6));
        assert_eq!(first_greedy_failure(&[1, 5, 12, 25], 100), Some(15));
    }
```

Why it works: `min_coins` (dynamic programming) is the trusted judge, and `find` returns the first amount where the greedy answer differs. For `[1, 5, 12, 25]` the first failure is 15: greedy pays 12 + 1 + 1 + 1 (4 coins), while 5 + 5 + 5 needs 3. The cost: for each of the `max_amount` amounts the DP costs O(amount · coins), so the total is about O(max_amount² · coins), which is fine for a few hundred.

</details>

### Task 5 — Complexities from memory (Phase 07 as a whole)

Without looking anything up, write the time complexity of each operation. For `HashMap::get`, say both the average and the worst case. Then check your table.

| # | Operation |
|---|---|
| a | `Vec::push` (amortized) |
| b | `Vec::insert(0, x)` |
| c | `VecDeque::pop_front` |
| d | `HashMap::get` |
| e | `BTreeMap::get` |
| f | `BinaryHeap::pop` |
| g | `slice::sort` |
| h | `slice::binary_search` |
| i | BFS on a graph with V vertices and E edges (adjacency list) |
| j | Dijkstra with a `BinaryHeap` on the same graph |

<details>
<summary>Hint</summary>

Think about what the operation has to touch: shifting items is O(n); halving a range or walking down a balanced tree is O(log n); a hash lookup goes straight to a slot; a sort compares n items about log n times each.

</details>

<details>
<summary>Answer</summary>

| # | Complexity |
|---|---|
| a | O(1) amortized |
| b | O(n) |
| c | O(1) |
| d | O(1) on average, O(n) in the worst case |
| e | O(log n) |
| f | O(log n) |
| g | O(n log n) |
| h | O(log n) (the slice must be sorted) |
| i | O(V + E) |
| j | O((V + E) log V) |

If you missed three or more, repeat Lessons 07.1 to 07.3 and then the lesson of each missed structure.

</details>

## If you got something wrong

| Topic | Lesson to repeat |
|---|---|
| Algorithms, Big-O notation | [Lesson 07.1 — What an algorithm is, and Big-O notation](01-algorithms-and-big-o.md) |
| Counting the time and space of loops and recursion | [Lesson 07.2 — Analyzing time and space](02-analyzing-time-and-space.md) |
| `Vec`, capacity, amortized push, arrays | [Lesson 07.3 — Arrays and vectors under the hood](03-arrays-and-vectors-under-the-hood.md) |
| Linked lists | [Lesson 07.4 — Linked lists](04-linked-lists.md) |
| Stacks, queues, `VecDeque` | [Lesson 07.5 — Stacks and queues](05-stacks-and-queues.md) |
| Hash tables, collisions, load factor | [Lesson 07.6 — How hash tables work](06-hash-tables.md) |
| Recursion, base case, call stack | [Lesson 07.7 — Recursion and the call stack](07-recursion.md) |
| Insertion, selection and bubble sort | [Lesson 07.8 — Simple sorting algorithms](08-simple-sorting-algorithms.md) |
| Merge sort, quicksort, `sort` and `sort_unstable` | [Lesson 07.9 — Merge sort and quicksort](09-merge-sort-and-quicksort.md) |
| Binary search, `binary_search`, `partition_point` | [Lesson 07.10 — Binary search](10-binary-search.md) |
| Trees, binary search trees, traversals | [Lesson 07.11 — Trees and binary search trees](11-trees-and-binary-search-trees.md) |
| Heaps, `BinaryHeap`, priority queues | [Lesson 07.12 — Heaps and priority queues](12-heaps-and-priority-queues.md) |
| Graphs, BFS, DFS | [Lesson 07.13 — Graphs: BFS and DFS](13-graphs-bfs-and-dfs.md) |
| Dijkstra, topological sort | [Lesson 07.14 — Shortest paths and topological sort](14-shortest-paths-and-topological-sort.md) |
| Two pointers, sliding window | [Lesson 07.15 — Two pointers and sliding window](15-two-pointers-and-sliding-window.md) |
| Dynamic programming | [Lesson 07.16 — Dynamic programming](16-dynamic-programming.md) |
| Greedy algorithms | [Lesson 07.17 — Greedy algorithms](17-greedy-algorithms.md) |
| The problem-solving method, choosing a complexity | [Lesson 07.18 — Problem-solving practice](18-problem-solving-practice.md) |
| Library plus thin binary, tests of a command-line tool, benchmarks | [Lesson 07.19 — Project: route finder](19-project-route-finder.md) |
| Tests that compare with a slow judge, `criterion` | [Lesson 05.6 — Testing in depth](../05-rust-intermediate/06-testing-in-depth.md) and [Lesson 05.12 — Benchmarking basics](../05-rust-intermediate/12-benchmarking-basics.md) |

When you repeat a lesson, do not only reread it. Redo its exercises on a branch, and write the tests first.

## What's next

You finished Phase 07. You can now estimate the cost of code with Big-O, choose the right standard collection, write the classic sorting, searching, tree, heap and graph algorithms, recognize two pointers, dynamic programming and greedy problems, and prove with tests and benchmarks that a faster solution is correct and really faster. Before you go on:

1. Run the quality gate in `code/p07-algorithms` and in `code/p07-route-finder`, then commit the checkpoint code: `git add code/p07-algorithms && git commit -m "feat(p07): checkpoint tasks"`.
2. Add a few lines to your learning notes: which problems you solved without hints, which needed all the hints, and which stretch goals of the route finder you plan to try.
3. Keep practising in small doses: one problem a day with the seven steps of Lesson 07.18 builds the habit faster than a long weekend session.

In **Phase 08 — Software design: principles, patterns and conventions** you move from single algorithms to whole programs: how to keep code simple (the principles KISS, DRY and YAGNI: Keep It Simple, Don't Repeat Yourself, You Aren't Gonna Need It), how to split it into parts with clear responsibilities, the SOLID principles (five design rules named by their initial letters), composition, type-driven design, and common patterns such as the builder and the state machine. Start with [Lesson 08.1 — Good design: KISS, DRY and YAGNI](../08-software-design/01-good-design-kiss-dry-yagni.md).

## Sources and further reading

- [The Rust Standard Library — std::collections](https://doc.rust-lang.org/std/collections/index.html) — the guide that compares `Vec`, `VecDeque`, `HashMap`, `BTreeMap`, `BinaryHeap` and the cost of their operations.
- [The Rust Standard Library — slice](https://doc.rust-lang.org/std/primitive.slice.html) — `sort`, `sort_unstable`, `binary_search`, `partition_point`, `windows`.
- [The Rust Programming Language — Chapter 8: Common Collections](https://doc.rust-lang.org/book/ch08-00-common-collections.html) — vectors, strings and hash maps.
- [The Rust Programming Language — Chapter 15: Smart Pointers](https://doc.rust-lang.org/book/ch15-00-smart-pointers.html) — `Box` and `Rc`, used for trees and lists.
- [Rust by Example](https://doc.rust-lang.org/rust-by-example/) — short runnable examples for practice.
- [Criterion.rs User Guide](https://bheisler.github.io/criterion.rs/book/) — benchmarking.

---

[🏠 Start](../START-HERE.md) · [← 07.19 Project: route finder](19-project-route-finder.md) · [📋 Phase 07](README.md) · [Phase 08 — Software design: principles, patterns and conventions →](../08-software-design/README.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
