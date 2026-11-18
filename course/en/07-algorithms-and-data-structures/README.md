<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 06.12 Checkpoint: Stage 1 complete](../06-rust-advanced/12-checkpoint-stage-1-complete.md) · [07.1 What an algorithm is, and Big-O notation →](01-algorithms-and-big-o.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/07-algorithms-and-data-structures/README.md) <!-- nav -->

# Phase 07 — Algorithms and data structures

> **Stage 2 — Engineering fundamentals** · ⏱ about 5–6 weeks at ~10 hours per week

## What this phase is about

An **algorithm** is a precise list of steps that solves a problem. A **data structure** is a way of keeping data in memory so that the steps are cheap. In this phase you learn the classic ones: arrays and vectors, linked lists, stacks and queues, hash tables, trees, heaps and graphs; and the classic algorithms: sorting, searching, shortest paths, and the techniques behind them (recursion, two pointers, sliding window, dynamic programming, greedy choice).

Why does a systems engineer need this? Because every program is built from them. A database index is a tree, a cache is a hash table with a list, a scheduler is a priority queue, a router finds shortest paths in a graph, and a build tool sorts tasks topologically. More importantly, you learn to **estimate the cost of code before you run it**: Big-O notation tells you whether a program that is fast on 100 items will still be fast on 10 million.

Everything is built in Rust, in one practice package, `code/p07-algorithms`, with one module per topic. You write the structures yourself to see how they work, test them against the standard library on thousands of random inputs, and measure them with `criterion`. For every topic the lesson also tells you which standard type to use in real code (`Vec`, `VecDeque`, `HashMap`, `BTreeMap`, `BinaryHeap`, `sort`, `binary_search`).

The phase ends with a project, a route finder on a small map, and a checkpoint.

## What you will be able to do

- [ ] Describe an algorithm's time and memory with Big-O notation, and say which case (best, worst, average, amortized) you mean.
- [ ] Explain how arrays, vectors, linked lists, stacks, queues, hash tables, trees, heaps and graphs are laid out in memory, and what each operation costs.
- [ ] Implement the basic data structures in safe Rust, and test them against the standard library with random operations.
- [ ] Write recursive algorithms with a correct base case, know the depth of the recursion, and turn a recursion into a loop.
- [ ] Implement and compare sorting algorithms and binary search, and know when to use `sort`, `sort_unstable` and `binary_search`.
- [ ] Traverse a graph with BFS and DFS, find shortest paths and order tasks topologically.
- [ ] Solve problems with two pointers, sliding window, dynamic programming and greedy choices.
- [ ] Measure an algorithm with `criterion` and choose the right standard collection for a job.

## Lessons

| # | Lesson | Time |
|---|---|---|
| 07.1 | [What an algorithm is, and Big-O notation](01-algorithms-and-big-o.md) | 2.5 h |
| 07.2 | [Analyzing time and space](02-analyzing-time-and-space.md) | 135 min |
| 07.3 | [Arrays and vectors under the hood](03-arrays-and-vectors-under-the-hood.md) | 3 h |
| 07.4 | [Linked lists](04-linked-lists.md) | 2.5 h |
| 07.5 | [Stacks and queues](05-stacks-and-queues.md) | 2.5 h |
| 07.6 | [How hash tables work](06-hash-tables.md) | 2.5 h |
| 07.7 | [Recursion and the call stack](07-recursion.md) | 135 min |
| 07.8 | [Simple sorting algorithms](08-simple-sorting-algorithms.md) | 3 h |
| 07.9 | [Merge sort and quicksort](09-merge-sort-and-quicksort.md) | 2.5 h |
| 07.10 | [Binary search](10-binary-search.md) | 135 min |
| 07.11 | [Trees and binary search trees](11-trees-and-binary-search-trees.md) | 3 h |
| 07.12 | [Heaps and priority queues](12-heaps-and-priority-queues.md) | 2.5 h |
| 07.13 | [Graphs: BFS and DFS](13-graphs-bfs-and-dfs.md) | 3 h |
| 07.14 | [Shortest paths and topological sort](14-shortest-paths-and-topological-sort.md) | 3 h |
| 07.15 | [Two pointers and sliding window](15-two-pointers-and-sliding-window.md) | 2.5 h |
| 07.16 | [Dynamic programming](16-dynamic-programming.md) | 2.5 h |
| 07.17 | [Greedy algorithms](17-greedy-algorithms.md) | 120 min |
| 07.18 | [Problem-solving practice](18-problem-solving-practice.md) | 135 min |
| 07.19 | [Project: route finder](19-project-route-finder.md) | 6 h |
| 07.20 | [Checkpoint](20-checkpoint.md) | 120 min |

## Projects in this phase

- **`code/p07-algorithms`** — the practice package that grows through the whole phase: one module per topic, unit tests next to the code, examples in `examples/`, benchmarks in `benches/`.
- **Project: route finder** (Lesson 07.19, package `code/p07-route-finder`) — find the shortest routes on a small map, with the graph, queue and heap structures of this phase, tests and benchmarks.
- **Checkpoint** (Lesson 07.20) — a self-test quiz and practical tasks before you move to software design.

## How to study this phase

1. **Type the code yourself** and run `cargo test` after every function. A data structure you only read about does not stay in your head.
2. **Draw before you code.** For every algorithm, draw the steps on a tiny input (4 or 5 items) on paper. If you cannot draw it, you cannot write it yet.
3. **Always compare with the standard library.** The tests in this phase check your code against `Vec`, `VecDeque`, `HashMap` or `sort` on many inputs. Write such a test before you trust any algorithm.
4. **State the complexity of everything you write**, out loud or in a comment: "time `O(n log n)`, space `O(n)`". Then check it with a benchmark. Numbers differ per computer; look at the ratios.
5. **Do the exercises in order and keep a log of the ones that took you long.** Come back to them after the checkpoint.

---

[🏠 Start](../START-HERE.md) · [← 06.12 Checkpoint: Stage 1 complete](../06-rust-advanced/12-checkpoint-stage-1-complete.md) · [07.1 What an algorithm is, and Big-O notation →](01-algorithms-and-big-o.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
