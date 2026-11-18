<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 07.13 Graphs: BFS and DFS](13-graphs-bfs-and-dfs.md) · [📋 Phase 07](README.md) · [07.15 Two pointers and sliding window →](15-two-pointers-and-sliding-window.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/07-algorithms-and-data-structures/14-shortest-paths-and-topological-sort.md) <!-- nav -->

# Lesson 07.14 — Shortest paths and topological sort

> **Phase 07 — Algorithms and data structures** · Lesson 14 of 20 · ⏱ about 3 hours (2 sittings)

## What you will learn

- Why BFS (Breadth-First Search, Lesson 07.13) is not enough when edges have different lengths, and what a weighted graph is.
- Dijkstra's algorithm: the idea, the code with a `BinaryHeap`, why it needs non-negative weights, and its cost O((V + E) log V).
- Topological sort: how to put tasks in an order that respects "A must be done before B", and how to detect a cycle.
- How to test these algorithms against slow references, and when to reach for a library.

## Before you start

- You finished [Lesson 07.13 — Graphs: BFS and DFS](13-graphs-bfs-and-dfs.md) and [Lesson 07.12 — Heaps and priority queues](12-heaps-and-priority-queues.md).
- You know `Reverse` and `BinaryHeap` from Lesson 07.12, and the `Graph` type with `bfs` from Lesson 07.13.
- In `code/p07-algorithms` create two files, `src/shortest_paths.rs` and `src/topological.rs`, and add `pub mod shortest_paths;` and `pub mod topological;` to `src/lib.rs`.

## New words

| Word | Meaning |
|---|---|
| **weighted graph** | A graph whose edges have a number (a *weight*), for example the length of a road. |
| **shortest path** | A path whose **total weight** is the smallest. In an unweighted graph, the path with the fewest edges. |
| **Dijkstra's algorithm** | An algorithm that finds the shortest paths from one start vertex to all others, when no weight is negative. It was invented by Edsger Dijkstra. |
| **relaxing an edge** | Checking whether going through a vertex gives a shorter way to its neighbour than the best way known so far, and if so, updating it. |
| **stale entry** | An entry in the heap that was valid when pushed but was made out of date by a later, shorter result. We skip it when it comes out. |
| **DAG** (Directed Acyclic Graph) | A directed graph without cycles. Dependencies between tasks normally form a DAG. |
| **topological order** | An order of the vertices of a DAG in which every edge `a -> b` has `a` before `b`. |
| **in-degree** | The number of edges that lead **into** a vertex. |
| **Kahn's algorithm** | The topological sort that repeatedly takes a vertex with in-degree 0. It was published by Arthur Kahn. |

## 1. Why BFS is not enough

BFS (Lesson 07.13) finds the path with the **fewest edges**. In a road map, roads have lengths, and the route with the fewest roads can be much longer than another route.
A small map with five towns (the numbers are lengths in kilometres):

```text
          4
    A --------- B
    |         / |
  1 |      2 /  | 1
    |      /    |
    C --------- D ---- E
          5           3
```

The edges: `A-B` 4, `A-C` 1, `C-B` 2, `B-D` 1, `C-D` 5, `D-E` 3. From `A` to `D`:

| Route | Roads | Length |
|---|---|---|
| `A, B, D` (BFS finds this: 2 edges) | 2 | 4 + 1 = 5 |
| `A, C, D` | 2 | 1 + 5 = 6 |
| `A, C, B, D` (the shortest) | 3 | 1 + 2 + 1 = **4** |

The best route has more roads than the others. We need an algorithm that adds up the weights: **Dijkstra's algorithm**.

## 2. Dijkstra's algorithm

**The idea.** Keep for every vertex the best known distance from the start (at first: infinity for all, 0 for the start). Repeat: take the not-yet-finished vertex with the **smallest** known distance; that distance is now final.
Then **relax** its edges: for each neighbour, check whether `distance[vertex] + weight` is better than the neighbour's known distance, and if it is, update it.

Why is the smallest distance final? Every other unfinished vertex is at least as far. Any other way to our vertex would have to pass through such a vertex, and then add more weight (it cannot be less than zero). So no shorter way exists.
**This only holds if no weight is negative.** With a negative edge, a longer-looking road could become cheaper later, and the argument breaks down.

To find "the unfinished vertex with the smallest distance" quickly we use a **min-heap** (Lesson 07.12). We push `(distance, vertex)` pairs. When a better distance is found for a vertex, we push a new pair, and the old pair stays in the heap. When an old
pair finally comes out, we recognize it as a **stale entry** (its distance is bigger than the best known) and skip it. This is simpler than removing it from the heap.

Dijkstra on the map above, from `A`. The heap holds `(distance, vertex)`; the best distances are written as `A=0`, and so on:

```text
start:       A=0                                            heap: (0,A)
pop (0,A):   B = 4 (via A), C = 1 (via A)                   heap: (1,C) (4,B)
pop (1,C):   B = 1+2 = 3 is better than 4: update B;
             D = 1+5 = 6                                    heap: (3,B) (4,B)* (6,D)
pop (3,B):   D = 3+1 = 4 is better than 6: update D         heap: (4,B)* (4,D) (6,D)*
pop (4,B)*:  stale: B is already 3, skip
pop (4,D):   E = 4+3 = 7                                    heap: (6,D)* (7,E)
pop (6,D)*:  stale, skip
pop (7,E):   nothing to improve                             heap: empty
result:      A=0  B=3  C=1  D=4  E=7      (* = stale entry)
```

### The code

Put the weighted graph and Dijkstra into `src/shortest_paths.rs`:

```rust
//! Weighted graphs and Dijkstra's shortest-path algorithm.

use std::cmp::Reverse;
use std::collections::BinaryHeap;

/// A graph whose edges have a length (a weight). Vertices are the numbers `0..vertex_count`.
#[derive(Debug, Clone)]
pub struct WeightedGraph {
    adjacency: Vec<Vec<(usize, u32)>>, // for each vertex: (neighbour, weight)
    directed: bool,
}

impl WeightedGraph {
    /// An undirected graph (every road works both ways) with no edges.
    pub fn new(vertex_count: usize) -> Self {
        WeightedGraph {
            adjacency: vec![Vec::new(); vertex_count],
            directed: false,
        }
    }

    /// A directed graph with no edges.
    pub fn new_directed(vertex_count: usize) -> Self {
        WeightedGraph {
            directed: true,
            ..WeightedGraph::new(vertex_count)
        }
    }

    pub fn vertex_count(&self) -> usize {
        self.adjacency.len()
    }

    /// Adds an edge with the given weight.
    ///
    /// # Panics
    /// Panics if a vertex number is not smaller than `vertex_count()`.
    pub fn add_edge(&mut self, from: usize, to: usize, weight: u32) {
        let n = self.vertex_count();
        assert!(
            from < n && to < n,
            "edge {from} -> {to}: a vertex number must be below {n}"
        );
        self.adjacency[from].push((to, weight));
        if !self.directed {
            self.adjacency[to].push((from, weight));
        }
    }

    /// The `(neighbour, weight)` pairs of `vertex`.
    pub fn neighbours(&self, vertex: usize) -> &[(usize, u32)] {
        &self.adjacency[vertex]
    }
}

/// The result of Dijkstra's algorithm for one start vertex.
#[derive(Debug, PartialEq, Eq)]
pub struct Paths {
    /// The length of the shortest path from the start, or `None` if unreachable.
    pub distance: Vec<Option<u64>>,
    /// The vertex before this one on a shortest path, or `None` for the start and unreachable vertices.
    pub previous: Vec<Option<usize>>,
}

/// Dijkstra's algorithm: shortest paths from `start` to every vertex.
/// Weights are `u32`, so they cannot be negative, which the algorithm requires.
pub fn dijkstra(graph: &WeightedGraph, start: usize) -> Paths {
    let n = graph.vertex_count();
    let mut distance: Vec<Option<u64>> = vec![None; n];
    let mut previous = vec![None; n];
    // A min-heap of (distance, vertex): `Reverse` makes the smallest distance come out first.
    let mut heap = BinaryHeap::new();

    distance[start] = Some(0);
    heap.push(Reverse((0u64, start)));
    while let Some(Reverse((d, vertex))) = heap.pop() {
        if distance[vertex].is_some_and(|best| d > best) {
            continue; // a stale entry: a shorter way to `vertex` was found after this was pushed
        }
        for &(neighbour, weight) in graph.neighbours(vertex) {
            let candidate = d + u64::from(weight);
            if distance[neighbour].is_none_or(|best| candidate < best) {
                distance[neighbour] = Some(candidate);
                previous[neighbour] = Some(vertex);
                heap.push(Reverse((candidate, neighbour)));
            }
        }
    }
    Paths { distance, previous }
}

/// The shortest path from the start of `paths` to `to`, including both ends, or `None`.
pub fn path_to(paths: &Paths, to: usize) -> Option<Vec<usize>> {
    paths.distance[to]?; // unreachable
    let mut path = vec![to];
    let mut current = to;
    while let Some(before) = paths.previous[current] {
        path.push(before);
        current = before;
    }
    path.reverse();
    Some(path)
}
```

**What happens:**

1. An edge is a pair `(neighbour, weight)` with `weight: u32`. An unsigned type cannot be negative, so **the type makes the main requirement of Dijkstra impossible to break** (📏 make invalid states unrepresentable). If your data has negative numbers,
   you cannot use this function, and the compiler tells you.
2. Distances are `u64`, and `u64::from(weight)` widens the weight before adding. A path can have many edges of up to `u32::MAX` each, and the sum would overflow a `u32` (a test below checks this).
3. `Reverse((0u64, start))` puts the pair `(distance, vertex)` into a min-heap. Tuples compare field by field, so the smallest distance comes out first.
4. The loop pops the closest entry. If its distance `d` is bigger than the best known distance of that vertex, it is a stale entry, and `continue` skips it. Otherwise we **relax** every edge: if the way through this vertex is
   better than the best known distance to the neighbour (or there is none yet), we record it, remember `previous`, and push the new pair.
5. `path_to` follows `previous` backwards, like `shortest_path` in Lesson 07.13.

**Cost.** Each edge is relaxed at most once per time its starting vertex is processed, and a vertex is processed once (stale entries are skipped), so at most E pairs are pushed. A heap operation costs O(log E), and E < V² means log E ≤ 2 log V.
The total is **O((V + E) log V)** time and **O(V + E)** memory. With an unsorted array instead of a heap, finding the minimum costs O(V) each time and the total is O(V²), which is better only for very dense graphs.

### Two compile errors you may meet

Forgetting that the heap holds `Reverse` values, or mixing the `u32` weight with the `u64` distance:

```rust
use std::cmp::Reverse;
use std::collections::BinaryHeap;

pub fn first_vertex(heap: &mut BinaryHeap<Reverse<(u64, usize)>>) -> Option<usize> {
    let (_distance, vertex) = heap.pop()?;
    Some(vertex)
}

pub fn add(d: u64, weight: u32) -> u64 {
    d + weight
}
```

```text
error[E0308]: mismatched types
 --> src/lib.rs:5:9
  |
5 |     let (_distance, vertex) = heap.pop()?;
  |         ^^^^^^^^^^^^^^^^^^^   ----------- this expression has type `Reverse<(u64, usize)>`
  |         |
  |         expected `Reverse<(u64, usize)>`, found `(_, _)`
  |
help: try wrapping the pattern in `std::cmp::Reverse`
  |
5 |     let std::cmp::Reverse((_distance, vertex)) = heap.pop()?;
  |         ++++++++++++++++++                   +

error[E0308]: mismatched types
  --> src/lib.rs:10:9
   |
10 |     d + weight
   |         ^^^^^^ expected `u64`, found `u32`

error[E0277]: cannot add `u32` to `u64`
  --> src/lib.rs:10:7
   |
10 |     d + weight
   |       ^ no implementation for `u64 + u32`
```

The first error: the heap returns `Reverse<(u64, usize)>`, so the pattern must say `Reverse((distance, vertex))`. The second: Rust never converts numbers silently. Write `d + u64::from(weight)` (a safe widening, which cannot fail).

## 3. Testing Dijkstra

The test strategy is the same as for BFS: a slow reference that relaxes all edges until nothing changes, random graphs including zero weights, and the sample from the picture. Two extra tests: with every weight 1, Dijkstra must give the BFS distances of Lesson 07.13, and very long paths must not overflow. Add to the end of `src/shortest_paths.rs`:

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use crate::graphs::{Graph, bfs};
    use crate::testutil::Rng;

    /// The graph of the lesson: A = 0, B = 1, C = 2, D = 3, E = 4.
    fn sample() -> WeightedGraph {
        let mut graph = WeightedGraph::new(5);
        for (a, b, weight) in [
            (0, 1, 4),
            (0, 2, 1),
            (2, 1, 2),
            (1, 3, 1),
            (2, 3, 5),
            (3, 4, 3),
        ] {
            graph.add_edge(a, b, weight);
        }
        graph
    }

    /// Pseudo-random edges (the `Rng` of Lesson 07.3): `(from, to, weight)`.
    fn random_edges(
        seed: u64,
        vertices: usize,
        edges: usize,
        max_weight: u32,
    ) -> Vec<(usize, usize, u32)> {
        let mut rng = Rng::new(seed);
        (0..edges)
            .map(|_| {
                let from = rng.below(vertices);
                let to = rng.below(vertices);
                let weight = (rng.next_u64() % (u64::from(max_weight) + 1)) as u32;
                (from, to, weight)
            })
            .collect()
    }

    fn build(vertices: usize, edges: &[(usize, usize, u32)], directed: bool) -> WeightedGraph {
        let mut graph = if directed {
            WeightedGraph::new_directed(vertices)
        } else {
            WeightedGraph::new(vertices)
        };
        for &(from, to, weight) in edges {
            graph.add_edge(from, to, weight);
        }
        graph
    }

    /// A slow but obviously correct reference: relax all edges until nothing changes.
    fn reference_distances(graph: &WeightedGraph, start: usize) -> Vec<Option<u64>> {
        let mut distance = vec![None; graph.vertex_count()];
        distance[start] = Some(0);
        loop {
            let mut changed = false;
            for vertex in 0..graph.vertex_count() {
                if let Some(d) = distance[vertex] {
                    for &(next, weight) in graph.neighbours(vertex) {
                        let candidate = d + u64::from(weight);
                        if distance[next].is_none_or(|old| candidate < old) {
                            distance[next] = Some(candidate);
                            changed = true;
                        }
                    }
                }
            }
            if !changed {
                return distance;
            }
        }
    }

    #[test]
    fn the_example_of_the_lesson() {
        let paths = dijkstra(&sample(), 0);
        assert_eq!(
            paths.distance,
            [Some(0), Some(3), Some(1), Some(4), Some(7)]
        );
        assert_eq!(path_to(&paths, 3), Some(vec![0, 2, 1, 3])); // 3 edges, length 4
        assert_eq!(path_to(&paths, 0), Some(vec![0]));
    }

    #[test]
    fn unreachable_vertices() {
        let graph = build(4, &[(0, 1, 5)], false);
        let paths = dijkstra(&graph, 0);
        assert_eq!(paths.distance, [Some(0), Some(5), None, None]);
        assert_eq!(path_to(&paths, 3), None);
    }

    #[test]
    fn matches_the_reference() {
        for seed in 1..=40 {
            for directed in [false, true] {
                let edges = random_edges(seed, 20, 45, 20); // weights 0..=20, zero allowed
                let graph = build(20, &edges, directed);
                for start in 0..20 {
                    assert_eq!(
                        dijkstra(&graph, start).distance,
                        reference_distances(&graph, start),
                        "seed {seed}, directed {directed}, start {start}"
                    );
                }
            }
        }
    }

    #[test]
    fn unit_weights_give_the_bfs_distances() {
        for seed in 1..=30 {
            let edges = random_edges(seed, 25, 40, 0);
            let mut weighted = WeightedGraph::new(25);
            let mut plain = Graph::new(25);
            for &(from, to, _) in &edges {
                weighted.add_edge(from, to, 1);
                plain.add_edge(from, to);
            }
            let dijkstra_distance = dijkstra(&weighted, 0).distance;
            let bfs_distance: Vec<Option<u64>> = bfs(&plain, 0)
                .distance
                .iter()
                .map(|d| d.map(|d| d as u64))
                .collect();
            assert_eq!(dijkstra_distance, bfs_distance, "seed {seed}");
        }
    }

    #[test]
    fn long_distances_do_not_overflow() {
        let mut graph = WeightedGraph::new(4);
        for v in 0..3 {
            graph.add_edge(v, v + 1, u32::MAX);
        }
        assert_eq!(
            dijkstra(&graph, 0).distance[3],
            Some(3 * u64::from(u32::MAX))
        );
    }
}
```

**What happens:**

1. `the_example_of_the_lesson` checks the numbers from the trace above: `A=0 B=3 C=1 D=4 E=7`, and that the route to `D` is `A, C, B, D`: three edges, not the two edges BFS would choose.
2. `matches_the_reference` compares Dijkstra with the slow reference for 40 seeds × directed and undirected × all 20 start vertices. The weights include 0, an important edge case: zero-length roads must not make the algorithm loop.
3. `unit_weights_give_the_bfs_distances` builds the same random edges in both graph types with weight 1 and compares the distances. The two algorithms must agree whenever they should.
4. `long_distances_do_not_overflow` uses three edges of `u32::MAX`: the sum (12,884,901,885) is bigger than a `u32`, and the `u64` distance is exact.

```bash
cd ~/projects/rust/code/p07-algorithms
cargo test shortest_paths
```

```text
running 5 tests
test shortest_paths::tests::unreachable_vertices ... ok
test shortest_paths::tests::long_distances_do_not_overflow ... ok
test shortest_paths::tests::the_example_of_the_lesson ... ok
test shortest_paths::tests::unit_weights_give_the_bfs_distances ... ok
test shortest_paths::tests::matches_the_reference ... ok

test result: ok. 5 passed; 0 failed; 0 ignored; 0 measured; 53 filtered out; finished in 0.01s
```

(The 53 filtered-out tests come from the main text of Lessons 07.1 to 07.13; with the tests of the exercises your number is bigger.)

## 4. Topological sort

Now a different question. You have tasks, and some must be finished before others: "design before code", "code before test", "test and docs before ship". In which order can you do all of them? This is a **directed** graph: an edge `a -> b` means "a before b".
An order that respects every edge is a **topological order**. It exists exactly when the graph has **no cycle** (a **DAG**): if `a` must precede `b` and `b` must precede `a`, nothing can be started.

**Kahn's algorithm.** Count for every vertex its **in-degree**: how many edges lead into it, that is, how many things it still waits for. A vertex with in-degree 0 can be done now. Do it, and tell its successors: each of them waits for one thing less. Any successor whose in-degree becomes 0 is
ready. Repeat until nothing is ready. If all vertices were done, the order is found. If some were never ready, they wait for each other in a cycle.

The tasks `design(0) code(1) test(2) docs(3) ship(4)` with the rules `0->1, 1->2, 0->3, 2->4, 3->4`:

```text
in-degrees:  design 0, code 1, test 1, docs 1, ship 2          ready: [design]
do design:   code 1->0, docs 1->0                              ready: [code, docs]
do code:     test 1->0                                         ready: [docs, test]
do docs:     ship 2->1                                         ready: [test]
do test:     ship 1->0                                         ready: [ship]
do ship:                                                       order: design code docs test ship
```

The ready vertices wait in a queue (Lesson 07.5). Put this into `src/topological.rs`:

```rust
//! Topological sort of a directed graph.

use crate::graphs::Graph;
use std::collections::VecDeque;

/// Orders the vertices so that for every edge `a -> b`, `a` comes before `b`.
/// Returns `None` if the graph has a cycle (then no such order exists).
pub fn topological_sort(graph: &Graph) -> Option<Vec<usize>> {
    let n = graph.vertex_count();

    // in_degree[v] = how many edges lead into v = how many things v still waits for.
    let mut in_degree = vec![0usize; n];
    for vertex in 0..n {
        for &next in graph.neighbours(vertex) {
            in_degree[next] += 1;
        }
    }

    let mut ready: VecDeque<usize> = (0..n).filter(|&v| in_degree[v] == 0).collect();
    let mut order = Vec::with_capacity(n);
    while let Some(vertex) = ready.pop_front() {
        order.push(vertex);
        for &next in graph.neighbours(vertex) {
            in_degree[next] -= 1;
            if in_degree[next] == 0 {
                ready.push_back(next); // everything `next` waited for is done
            }
        }
    }

    // If some vertices never became ready, they lie on or behind a cycle.
    (order.len() == n).then_some(order)
}
```

**What happens:**

1. The first loop visits every edge once and counts the in-degrees. It reuses the `Graph` type of Lesson 07.13 (create it with `Graph::new_directed`).
2. `ready` starts with all vertices that wait for nothing. The main loop takes one, appends it to `order`, and removes one "wait" from each successor. A successor that reaches 0 is ready and joins the queue.
3. A vertex on a cycle never reaches in-degree 0, because the vertex it waits for waits for it. So `order.len() < n` means a cycle, and `then_some` turns the check into `None` or `Some(order)`.
   `bool::then_some(x)` returns `Some(x)` if the bool is true, else `None`.
4. Duplicate edges and a vertex with an edge to itself work correctly: a duplicate counts twice and is subtracted twice; a self-loop is a cycle and returns `None`.

**Cost.** Every vertex enters the queue at most once and every edge is looked at twice (once to count, once to subtract): **O(V + E)** time, **O(V)** extra memory. The order is not unique; Kahn's algorithm with a queue gives one of them. (You can also build a topological order with DFS (Depth-First Search, Lesson 07.13), by writing
the vertices in the reverse of the order in which DFS finishes them, but Kahn's algorithm also reports cycles without extra work.)

Create `examples/e14_build_order.rs` and run it:

```rust
use p07_algorithms::graphs::Graph;
use p07_algorithms::topological::topological_sort;

fn main() {
    let tasks = ["design", "code", "test", "docs", "ship"];
    // (a, b) means: task `a` must be finished before task `b` can start.
    let rules = [(0, 1), (1, 2), (0, 3), (2, 4), (3, 4)];

    let mut graph = Graph::new_directed(tasks.len());
    for (a, b) in rules {
        graph.add_edge(a, b);
    }

    match topological_sort(&graph) {
        Some(order) => {
            let names: Vec<&str> = order.iter().map(|&i| tasks[i]).collect();
            println!("do the tasks in this order: {}", names.join(" -> "));
        }
        None => println!("the rules contain a cycle: no order exists"),
    }

    graph.add_edge(4, 0); // "ship" must come before "design": a cycle
    match topological_sort(&graph) {
        Some(_) => println!("an order exists"),
        None => println!("after adding ship -> design: the rules contain a cycle: no order exists"),
    }
}
```

```bash
cargo run --example e14_build_order
```

```text
do the tasks in this order: design -> code -> docs -> test -> ship
after adding ship -> design: the rules contain a cycle: no order exists
```

Build tools, package managers (the order in which dependencies are compiled; Cargo does this), spreadsheet formulas and course prerequisites all use topological sort.

### Testing topological sort

An order has a **checkable property**: for every edge, the source comes before the target. We do not need a reference answer, because many orders are correct. And we compare the "is there a cycle" answer with a slow reference (a transitive closure: can `v` reach itself?).
Add to the end of `src/topological.rs`:

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use crate::testutil::Rng;

    /// A pseudo-random directed graph (the `Rng` of Lesson 07.3).
    fn random_graph(seed: u64, vertices: usize, edges: usize, acyclic: bool) -> Graph {
        let mut rng = Rng::new(seed);
        let mut graph = Graph::new_directed(vertices);
        for _ in 0..edges {
            let a = rng.below(vertices);
            let b = rng.below(vertices);
            if acyclic {
                if a != b {
                    graph.add_edge(a.min(b), a.max(b)); // edges only go "upwards": no cycle is possible
                }
            } else {
                graph.add_edge(a, b);
            }
        }
        graph
    }

    /// Does some vertex lead back to itself? Slow reference: the transitive closure.
    fn has_cycle(graph: &Graph) -> bool {
        let n = graph.vertex_count();
        let mut reach = vec![vec![false; n]; n];
        for (a, row) in reach.iter_mut().enumerate() {
            for &b in graph.neighbours(a) {
                row[b] = true;
            }
        }
        for k in 0..n {
            for i in 0..n {
                for j in 0..n {
                    if reach[i][k] && reach[k][j] {
                        reach[i][j] = true;
                    }
                }
            }
        }
        (0..n).any(|v| reach[v][v])
    }

    #[test]
    fn a_small_example() {
        let mut graph = Graph::new_directed(5);
        for (a, b) in [(0, 1), (1, 2), (0, 3), (3, 4), (2, 4)] {
            graph.add_edge(a, b);
        }
        assert_eq!(topological_sort(&graph), Some(vec![0, 1, 3, 2, 4]));
    }

    #[test]
    fn empty_and_edgeless_graphs() {
        assert_eq!(topological_sort(&Graph::new_directed(0)), Some(vec![]));
        assert_eq!(
            topological_sort(&Graph::new_directed(3)),
            Some(vec![0, 1, 2])
        );
    }

    #[test]
    fn cycles_are_detected() {
        let mut triangle = Graph::new_directed(3);
        for (a, b) in [(0, 1), (1, 2), (2, 0)] {
            triangle.add_edge(a, b);
        }
        assert_eq!(topological_sort(&triangle), None);

        let mut self_loop = Graph::new_directed(2);
        self_loop.add_edge(1, 1);
        assert_eq!(topological_sort(&self_loop), None);
    }

    #[test]
    fn the_order_respects_every_edge() {
        for seed in 1..=50 {
            let graph = random_graph(seed, 15, 30, true);
            let order =
                topological_sort(&graph).expect("a graph with only upward edges has no cycle");
            assert_eq!(order.len(), 15);
            let mut position = [0; 15];
            for (i, &v) in order.iter().enumerate() {
                position[v] = i;
            }
            for a in 0..15 {
                for &b in graph.neighbours(a) {
                    assert!(position[a] < position[b], "edge {a} -> {b}, seed {seed}");
                }
            }
        }
    }

    #[test]
    fn sorting_succeeds_exactly_when_there_is_no_cycle() {
        for seed in 1..=100 {
            let graph = random_graph(seed, 6, 7, false);
            assert_eq!(
                topological_sort(&graph).is_some(),
                !has_cycle(&graph),
                "seed {seed}"
            );
        }
    }
}
```

**What happens:**

1. `random_graph(..., acyclic: true)` only creates edges from a smaller to a bigger number, so a cycle is impossible. `the_order_respects_every_edge` stores the position of each vertex in the order and checks `position[a] < position[b]` for every edge.
2. `sorting_succeeds_exactly_when_there_is_no_cycle` uses graphs that may have cycles, and `has_cycle` is the slow referee (O(V³), fine for 6 vertices).
3. `a_small_example` pins down the exact order of the lesson. Another correct implementation could return a different valid order; if you change the algorithm, this one test may need an update, while the property test stays.

```bash
cargo test topological
```

```text
running 5 tests
test topological::tests::empty_and_edgeless_graphs ... ok
test topological::tests::a_small_example ... ok
test topological::tests::cycles_are_detected ... ok
test topological::tests::the_order_respects_every_edge ... ok
test topological::tests::sorting_succeeds_exactly_when_there_is_no_cycle ... ok

test result: ok. 5 passed; 0 failed; 0 ignored; 0 measured; 58 filtered out; finished in 0.01s
```

> 🧪 **Testing:** When many answers are correct, test the **property** that makes an answer correct ("every edge goes forward in the order") instead of one fixed answer. This is the idea behind property-based testing (Phase 09).

## 5. What to use in real code

The standard library has no graph algorithms; `BinaryHeap`, `Reverse` and `VecDeque` are the building blocks, as here. The `petgraph` crate has `dijkstra`, `toposort` and many more (A\*, Bellman–Ford, strongly connected components).
Know the following relatives of Dijkstra, which you may need one day:

| Problem | Algorithm | Time |
|---|---|---|
| shortest paths, **negative** weights allowed | Bellman–Ford | O(V · E) |
| shortest path with a good guess of the distance to the goal | A\* ("A star") | depends on the guess |
| shortest paths between **all** pairs | Floyd–Warshall | O(V³) |
| all weights equal | BFS | O(V + E) |

## Common mistakes

**1. Dijkstra with negative weights.** The proof "the smallest distance is final" fails. Keep weights unsigned, as this lesson does, or use Bellman–Ford.

**2. Forgetting the stale check.** Without `if distance[vertex].is_some_and(|best| d > best) { continue; }` the result is still correct, but every stale entry is processed again with all its edges. On big graphs that can be many times slower.

**3. Using `u32` for the sum of distances.** Weights fit in `u32`, but paths do not. Use `u64` for sums (or check for overflow with `checked_add`).

**4. Starting topological sort on an undirected graph.** An undirected edge `a - b` is also `b -> a`, a cycle of two. Use `Graph::new_directed`.

## Try it yourself

1. Add `shortest_paths.rs` and its tests, then `topological.rs` and its tests. Run `cargo test`.
2. In `dijkstra`, delete the stale check (the `if ... continue` lines). Do the tests still pass? Why is the answer still correct? (See mistake 2.) Put the check back.
3. In `topological_sort`, make `ready` a `Vec<usize>` used as a stack (`pop()` instead of `pop_front()`, `push(next)` instead of `push_back(next)`). Run `cargo test topological`. Two tests fail: which ones, and why do the property tests still pass?
4. Run `cargo fmt`, `cargo clippy --all-targets`, and commit: `git add code/p07-algorithms && git commit -m "feat(p07): Dijkstra and topological sort"`.

## Exercises

### Exercise 1 — Dijkstra by hand (easy)

The undirected graph has the edges `0-1` (7), `0-2` (9), `0-5` (14), `1-2` (10), `1-3` (15), `2-3` (11), `2-5` (2), `3-4` (6), `4-5` (9). Run Dijkstra from vertex `0` by hand. Give the final distances to all vertices and, for each vertex, the `previous` vertex.

<details>
<summary>Hint</summary>

Pop order by distance: vertex 0 (0), then 1 (7), then 2 (9). From 2, vertex 5 gets 9 + 2 = 11 (better than 14). Continue with the smallest unfinished distance.

</details>

<details>
<summary>Answer</summary>

```text
distance:  0:0   1:7   2:9   3:20   4:20   5:11
previous:  0:-   1:0   2:0   3:2    4:5    5:2
```

Why: `1` and `2` are reached directly (7 and 9). From `2`: `3` gets 9 + 11 = 20, `5` gets 9 + 2 = 11, which beats 14 (the direct road). From `5` (11): `4` gets 11 + 9 = 20. From `1`: `3` would be 7 + 15 = 22, worse than 20. From `3` or `4`: no improvement. The path to `4` is `0, 2, 5, 4` (length 20).

</details>

### Exercise 2 — Paths add up (easy)

Write a test: for random undirected graphs, for every reachable target, `path_to` starts at the start and ends at the target, and the sum of the lightest edge between every two consecutive vertices of the path equals `distance[target]`.

<details>
<summary>Hint</summary>

`path.windows(2)` gives each consecutive pair. For a pair `(a, b)`, look at `graph.neighbours(a)`, keep the entries whose neighbour is `b`, and take the minimum weight (there can be parallel edges). Add the weights as `u64`.

</details>

<details>
<summary>Answer</summary>

```rust
    #[test]
    fn paths_add_up_to_the_distances() {
        for seed in 1..=30 {
            let edges = random_edges(seed, 15, 30, 9);
            let graph = build(15, &edges, false);
            let paths = dijkstra(&graph, 0);
            for target in 0..15 {
                match (path_to(&paths, target), paths.distance[target]) {
                    (Some(path), Some(distance)) => {
                        assert_eq!((path[0], path[path.len() - 1]), (0, target));
                        let length: u64 = path
                            .windows(2)
                            .map(|pair| {
                                graph
                                    .neighbours(pair[0])
                                    .iter()
                                    .filter(|&&(next, _)| next == pair[1])
                                    .map(|&(_, weight)| u64::from(weight))
                                    .min()
                                    .expect("neighbours on a path are joined by an edge")
                            })
                            .sum();
                        assert_eq!(length, distance, "seed {seed}, target {target}");
                    }
                    (None, None) => {}
                    other => panic!("path and distance disagree: {other:?}"),
                }
            }
        }
    }
```

Why it works: the test checks that the **path** and the **number** tell the same story, which catches a wrong `previous` even when the distances are right. The `(None, None)` arm covers unreachable targets, and the last arm panics if only one of the two answers exists.

</details>

### Exercise 3 — State the complexity (medium)

1. What is the time of Dijkstra with a binary heap, for V vertices and E edges? What is the maximum number of entries in the heap?
2. On a very dense graph (E close to V²), the variant of Dijkstra that finds the closest unfinished vertex by scanning an array costs O(V²). Which of the two is better then, and why?
3. What is the time of topological sort? What does it return for a graph with a cycle?

<details>
<summary>Hint</summary>

For (2), compare V² with (V + E) log V when E ≈ V². Remember that a heap operation costs a logarithm.

</details>

<details>
<summary>Answer</summary>

1. **O((V + E) log V)** time. Every successful relaxation pushes one entry, so the heap holds at most **E** entries (plus the start).
2. With E ≈ V², the heap version costs about V² log V, so the **array version with O(V²) is better** for very dense graphs: it avoids the logarithm. For sparse graphs (E close to V), the heap version is far better: V log V against V².
3. **O(V + E)**. For a graph with a cycle it returns `None`, because the vertices of the cycle never reach in-degree 0.

</details>

### Exercise 4 — The longest chain (medium)

In a DAG of tasks, the longest chain of dependencies decides how many "rounds" you need if you can run any number of independent tasks at the same time. Write `longest_chain(graph: &Graph) -> Option<usize>` that returns the number of vertices on the longest path, or `None` if the graph has a cycle. Test it on the build-order example (the answer is 4), the empty graph, a graph without edges and a graph with a cycle.

<details>
<summary>Hint</summary>

Get the topological order first. Keep `depth[v]`, the number of vertices on the longest chain that ends in `v`, starting at 1. Visit the vertices in topological order; for each edge `v -> w`, `depth[w] = max(depth[w], depth[v] + 1)`. The answer is the biggest depth.

</details>

<details>
<summary>Answer</summary>

```rust
/// The number of vertices on the longest chain of dependencies, or `None` if there is a cycle.
pub fn longest_chain(graph: &Graph) -> Option<usize> {
    let order = topological_sort(graph)?;
    let mut depth = vec![1usize; graph.vertex_count()]; // a vertex alone is a chain of length 1
    for &vertex in &order {
        for &next in graph.neighbours(vertex) {
            depth[next] = depth[next].max(depth[vertex] + 1);
        }
    }
    Some(depth.into_iter().max().unwrap_or(0))
}
```

The test, inside the `tests` module:

```rust
    #[test]
    fn longest_chain_examples() {
        let mut graph = Graph::new_directed(5);
        for (a, b) in [(0, 1), (1, 2), (0, 3), (2, 4), (3, 4)] {
            graph.add_edge(a, b);
        }
        assert_eq!(longest_chain(&graph), Some(4)); // 0 -> 1 -> 2 -> 4
        assert_eq!(longest_chain(&Graph::new_directed(0)), Some(0));
        assert_eq!(longest_chain(&Graph::new_directed(3)), Some(1));
        graph.add_edge(4, 0);
        assert_eq!(longest_chain(&graph), None);
    }
```

Why it works: in topological order, when we reach a vertex all its predecessors have been processed, so its `depth` is final and can be passed on to its successors. This is a first taste of dynamic programming (Lesson 07.16): the answer for a vertex is built from the answers of the vertices before it. Time O(V + E).

</details>

## Quiz

1. Why can BFS not find the shortest route on a road map with lengths?

<details>
<summary>Answer</summary>

BFS minimizes the **number of edges**, not the sum of their weights. A route with more but shorter roads can be shorter in total.

</details>

2. Why does Dijkstra's algorithm require non-negative weights?

<details>
<summary>Answer</summary>

It treats the closest unfinished vertex as final, because any other way to it would have to pass through a farther vertex and add more weight. A negative edge could make such a way shorter, so the reasoning (and the result) would be wrong.

</details>

3. What is a stale entry in the heap, and what do we do with it?

<details>
<summary>Answer</summary>

An entry `(distance, vertex)` whose distance is bigger than the best known distance of the vertex, because a shorter way was found after it was pushed. When it is popped, we skip it.

</details>

4. In Kahn's algorithm, what does it mean if some vertices were never put in the queue?

<details>
<summary>Answer</summary>

They lie on a cycle or depend on one: each waits for something that waits for it, so their in-degree never reaches 0. No topological order exists.

</details>

## Summary

- On a weighted graph the shortest path is the one with the smallest total weight; BFS only minimizes the number of edges.
- Dijkstra's algorithm repeatedly takes the closest unfinished vertex (from a min-heap of `(distance, vertex)` pairs wrapped in `Reverse`) and relaxes its edges. Stale heap entries are skipped.
- It needs non-negative weights (use unsigned weights, so the type enforces it) and costs O((V + E) log V). Sum distances in `u64`.
- A topological order puts every edge's source before its target and exists exactly for DAGs. Kahn's algorithm repeatedly takes vertices with in-degree 0; it runs in O(V + E) and detects cycles.
- Test shortest paths against a slow relaxation reference and against BFS for unit weights; test topological orders by their property and cycle detection against a transitive closure.

## Sources and further reading

- [Rust standard library — std::collections::BinaryHeap](https://doc.rust-lang.org/std/collections/struct.BinaryHeap.html) — the official page includes a Dijkstra example with `Reverse`-style ordering.
- [Rust standard library — std::cmp::Reverse](https://doc.rust-lang.org/std/cmp/struct.Reverse.html) — the wrapper that turns a max-heap into a min-heap.
- [Rust standard library — bool::then_some](https://doc.rust-lang.org/std/primitive.bool.html#method.then_some) — turning a condition into an `Option`.
- [petgraph::algo on docs.rs](https://docs.rs/petgraph/latest/petgraph/algo/index.html) — ready-made `dijkstra`, `toposort`, `bellman_ford`, `astar` and more for production code.

---

[🏠 Start](../START-HERE.md) · [← 07.13 Graphs: BFS and DFS](13-graphs-bfs-and-dfs.md) · [📋 Phase 07](README.md) · [07.15 Two pointers and sliding window →](15-two-pointers-and-sliding-window.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
