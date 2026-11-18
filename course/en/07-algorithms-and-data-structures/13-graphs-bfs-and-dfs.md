<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 07.12 Heaps and priority queues](12-heaps-and-priority-queues.md) · [📋 Phase 07](README.md) · [07.14 Shortest paths and topological sort →](14-shortest-paths-and-topological-sort.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/07-algorithms-and-data-structures/13-graphs-bfs-and-dfs.md) <!-- nav -->

# Lesson 07.13 — Graphs: BFS and DFS

> **Phase 07 — Algorithms and data structures** · Lesson 13 of 20 · ⏱ about 3 hours (2 sittings)

## What you will learn

- What a graph is, the words for its parts (vertex, edge, path, cycle, connected), and how to store one as an adjacency list.
- Breadth-first search (BFS): how it finds the path with the fewest edges, and why it needs a queue.
- Depth-first search (DFS): how it explores as far as possible first, with recursion and with an explicit stack, and when recursion is dangerous.
- How to test graph code against a slow reference, and what the cost O(V + E) means.

## Before you start

- You finished [Lesson 07.12 — Heaps and priority queues](12-heaps-and-priority-queues.md). You know queues and stacks ([Lesson 07.5](05-stacks-and-queues.md)) and recursion ([Lesson 07.7](07-recursion.md)).
- You know `Vec<Vec<T>>` ([Lesson 04.7](../04-structs-enums-collections-errors/07-vectors.md)) and `Option` ([Lesson 04.4](../04-structs-enums-collections-errors/04-option-and-match.md)).
- In `code/p07-algorithms` create `src/graphs.rs` and add `pub mod graphs;` to `src/lib.rs`.

## New words

| Word | Meaning |
|---|---|
| **graph** | A set of **vertices** (things) and **edges** (links between two vertices). Roads between cities, friends in a network, web pages and their links. |
| **vertex** (plural **vertices**) | One thing in a graph (also called a *node*). |
| **edge** | A link between two vertices. |
| **undirected / directed** | In an undirected graph an edge works both ways (a two-way road). In a directed graph an edge has a direction, `a -> b` (a one-way street, a web link). |
| **neighbour** | A vertex that you can reach from another vertex by one edge. |
| **path** | A sequence of vertices where each neighbouring pair is joined by an edge. Its *length* is the number of edges. |
| **cycle** | A path that starts and ends at the same vertex. |
| **connected** | An undirected graph is connected if there is a path between every two vertices. A **connected component** is a maximal group of vertices that can reach each other. |
| **adjacency list** | A way to store a graph: for each vertex, a list of its neighbours. |
| **adjacency matrix** | Another way: a table `V × V` with a mark in cell `(a, b)` when there is an edge `a -> b`. |
| **V and E** | The number of vertices and the number of edges. Running times of graph algorithms are written with them: O(V + E). |
| **BFS** (Breadth-First Search) | Visit all vertices at distance 1, then all at distance 2, and so on. Uses a **queue**. |
| **DFS** (Depth-First Search) | Go as deep as possible along one path, then step back and try the next. Uses a **stack** (or recursion). |
| **bipartite** | A graph whose vertices can be split into two groups so that every edge joins vertices from different groups. |

## 1. Graphs

A graph is the most general "things and links" structure. A list is a graph where each vertex has one next vertex; a tree (Lesson 07.11) is a graph without cycles and with a root.
Here is a small undirected graph with 7 vertices and 7 edges:

```text
   0 --- 1 --- 2
   |     |     |
   3 --- 4 ----+         5 --- 6
```

The edges are `0-1`, `1-2`, `0-3`, `1-4`, `2-4`, `3-4` and `5-6`. A path from `0` to `2` is `0, 1, 2` (length 2). A cycle is `1, 2, 4, 1`. The graph is not connected: it has **two connected components**,
`{0, 1, 2, 3, 4}` and `{5, 6}`, and no path leads from `0` to `6`.

Typical questions, and the algorithms of this lesson that answer them:

| Question | Answer with |
|---|---|
| Is there a route from A to B? | BFS or DFS |
| What is the route with the fewest steps? | BFS |
| How many separate groups are there? | BFS or DFS in a loop |
| Can every task be done after the tasks it needs? | topological sort (Lesson 07.14) |
| What is the shortest route when roads have lengths? | Dijkstra (Lesson 07.14) |

## 2. Storing a graph

**Adjacency matrix**: a `V × V` table; cell `(a, b)` says whether the edge exists. Checking one edge is O(1), but the table needs O(V²) memory, even if there are few edges, and finding the
neighbours of a vertex means scanning a whole row.

**Adjacency list**: for each vertex a list of its neighbours. It needs O(V + E) memory, and the neighbours of a vertex are right there. This is the default for almost all graphs, which are *sparse* (few edges compared with V²).
The sample graph as lists:

```text
0: [1, 3]
1: [0, 2, 4]
2: [1, 4]
3: [0, 4]
4: [1, 2, 3]
5: [6]
6: [5]
```

In Rust that is a `Vec<Vec<usize>>`: the vertices are the numbers `0..V`, and `adjacency[v]` is the list of neighbours of `v`. When your vertices are names (cities), you give every name a number and keep a
`Vec<String>` plus a `HashMap<String, usize>` on the side (the project of Lesson 07.19 does this).

Put the graph type into `src/graphs.rs`:

```rust
//! Graphs as adjacency lists, with breadth-first and depth-first search.

use std::collections::VecDeque;

/// A graph whose vertices are the numbers `0..vertex_count`, stored as adjacency lists.
#[derive(Debug, Clone)]
pub struct Graph {
    adjacency: Vec<Vec<usize>>,
    directed: bool,
    edge_count: usize,
}

impl Graph {
    /// An undirected graph with `vertex_count` vertices and no edges.
    pub fn new(vertex_count: usize) -> Self {
        Graph {
            adjacency: vec![Vec::new(); vertex_count],
            directed: false,
            edge_count: 0,
        }
    }

    /// A directed graph with `vertex_count` vertices and no edges.
    pub fn new_directed(vertex_count: usize) -> Self {
        Graph {
            directed: true,
            ..Graph::new(vertex_count)
        }
    }

    pub fn vertex_count(&self) -> usize {
        self.adjacency.len()
    }

    pub fn edge_count(&self) -> usize {
        self.edge_count
    }

    /// Adds the edge `from -> to` (and `to -> from` in an undirected graph).
    ///
    /// # Panics
    /// Panics if a vertex number is not smaller than `vertex_count()`.
    pub fn add_edge(&mut self, from: usize, to: usize) {
        let n = self.vertex_count();
        assert!(
            from < n && to < n,
            "edge {from} -> {to}: a vertex number must be below {n}"
        );
        self.adjacency[from].push(to);
        if !self.directed {
            self.adjacency[to].push(from);
        }
        self.edge_count += 1;
    }

    /// The vertices that can be reached from `vertex` by one edge.
    pub fn neighbours(&self, vertex: usize) -> &[usize] {
        &self.adjacency[vertex]
    }
}
```

**What happens:**

1. `vec![Vec::new(); vertex_count]` makes `vertex_count` empty lists (the `vec!` macro clones the empty `Vec`).
2. `new_directed` uses *struct update syntax* `..Graph::new(vertex_count)`: all fields not named come from the second value, and only `directed` is changed.
3. `add_edge` adds `to` to the list of `from`; in an undirected graph also `from` to the list of `to`. `edge_count` counts each edge once.
4. The `assert!` stops the program with a clear message when a vertex number is wrong, instead of an unclear "index out of bounds" later. A caller that gets vertex numbers from outside the program should check them first (🔒 never trust input);
   the project of Lesson 07.19 returns an error instead of panicking.
5. `neighbours` returns a slice `&[usize]`: callers can read the list but not change it.

## 3. Breadth-first search (BFS)

BFS explores the graph in **waves**. First the start vertex, then all its neighbours (distance 1), then all their new neighbours (distance 2), and so on. The first time BFS reaches a vertex, it has used the
**fewest possible edges**, so BFS finds shortest paths in graphs where every edge counts the same.

To keep the order "oldest first" we use a **queue** (Lesson 07.5): vertices wait in line, and we always take the one that has waited longest. To avoid going in circles, we remember which vertices were already
seen. BFS on the sample graph from `0` (the queue's front is on the left):

```text
start:    queue [0]            dist: 0:0
take 0:   new 1, 3             queue [1, 3]        dist: 1:1  3:1
take 1:   0 seen; new 2, 4     queue [3, 2, 4]     dist: 2:2  4:2
take 3:   0, 4 seen            queue [2, 4]
take 2:   1, 4 seen            queue [4]
take 4:   1, 2, 3 seen         queue []            done: 5 and 6 were never reached
```

Add to `src/graphs.rs`:

```rust
/// What breadth-first search learns about every vertex.
#[derive(Debug, PartialEq, Eq)]
pub struct BfsResult {
    /// Number of edges on the shortest path from the start, or `None` if unreachable.
    pub distance: Vec<Option<usize>>,
    /// The vertex from which we first reached this vertex, or `None` for the start and unreachable vertices.
    pub previous: Vec<Option<usize>>,
}

/// Breadth-first search from `start`: visits vertices in order of their distance from `start`.
pub fn bfs(graph: &Graph, start: usize) -> BfsResult {
    let n = graph.vertex_count();
    let mut distance = vec![None; n];
    let mut previous = vec![None; n];
    let mut queue = VecDeque::new();

    distance[start] = Some(0); // mark when we put a vertex in the queue, not when we take it out
    queue.push_back(start);
    while let Some(vertex) = queue.pop_front() {
        let next_distance = distance[vertex].expect("a queued vertex has a distance") + 1;
        for &neighbour in graph.neighbours(vertex) {
            if distance[neighbour].is_none() {
                distance[neighbour] = Some(next_distance);
                previous[neighbour] = Some(vertex);
                queue.push_back(neighbour);
            }
        }
    }
    BfsResult { distance, previous }
}

/// A path with the fewest edges from `from` to `to`, including both ends, or `None` if there is none.
pub fn shortest_path(graph: &Graph, from: usize, to: usize) -> Option<Vec<usize>> {
    let result = bfs(graph, from);
    result.distance[to]?; // unreachable: give up
    let mut path = vec![to];
    let mut current = to;
    while let Some(before) = result.previous[current] {
        path.push(before);
        current = before;
    }
    path.reverse();
    Some(path)
}
```

**What happens:**

1. `distance[v]` is `None` while `v` is unseen, and `Some(d)` once it is in the queue. It does two jobs: it is the "seen" flag and the answer. `previous[v]` remembers who found `v` first.
2. The start gets distance 0 and goes into the queue. The loop takes the front vertex and looks at each neighbour. A neighbour that has no distance yet is new: it gets `distance + 1`, a `previous`, and a place at the **back** of the queue.
3. We mark a vertex **when it enters the queue**, not when it leaves. Then no vertex is ever queued twice. (Common mistakes explains what goes wrong otherwise.)
4. `shortest_path` runs BFS and follows `previous` from the target back to the start, building the path backwards, then reverses it. `result.distance[to]?` returns `None` early when `to` was never reached.
   The chain of `previous` values is a tree (the *BFS tree*) rooted at the start.

**Cost.** Every vertex is queued at most once: V queue operations. Every neighbour list is read once, so every edge is looked at once (twice in an undirected graph): E steps. Total **O(V + E)** time, **O(V)** extra memory.

## 4. Depth-first search (DFS)

DFS goes the other way: from a vertex it takes the first unseen neighbour and continues from there, as deep as it can. When it gets stuck, it steps **back** and tries the next neighbour. It is a maze walker who
always takes the first unexplored corridor and returns to the last junction at a dead end. The pending junctions are kept on a **stack**, which in a recursive function is the call stack (Lesson 07.7).

DFS from `0` on the sample graph. Each line is a vertex we enter, with the neighbour it chooses next:

```text
visit 0:  neighbours [1, 3]       -> go to 1
visit 1:  neighbours [0, 2, 4]    -> 0 seen, go to 2
visit 2:  neighbours [1, 4]       -> 1 seen, go to 4
visit 4:  neighbours [1, 2, 3]    -> 1, 2 seen, go to 3
visit 3:  neighbours [0, 4]       -> both seen: step back
(back at 4, 1 and 0: nothing new)          order: 0 1 2 4 3
```

Add the recursive version:

```rust
/// Depth-first search with recursion. Returns the vertices in the order they are first visited.
pub fn dfs_recursive(graph: &Graph, start: usize) -> Vec<usize> {
    let mut visited = vec![false; graph.vertex_count()];
    let mut order = Vec::new();
    visit(graph, start, &mut visited, &mut order);
    order
}

fn visit(graph: &Graph, vertex: usize, visited: &mut [bool], order: &mut Vec<usize>) {
    visited[vertex] = true;
    order.push(vertex);
    for &neighbour in graph.neighbours(vertex) {
        if !visited[neighbour] {
            visit(graph, neighbour, visited, order);
        }
    }
}
```

**What happens:** `visit` marks the vertex, records it, and calls itself for every neighbour that is not yet visited. The `visited` flags are what stop the walk from going in circles around a cycle. When the loop over the neighbours
ends, the function returns, which is the "step back". The recursion depth equals the length of the current path, up to V.

That depth is the weakness. A graph that is a long chain (100,000 or more vertices) makes the recursion so deep that the call stack overflows. The version with an **explicit stack** (a `Vec` on the heap) has no such limit:

```rust
/// Depth-first search with an explicit stack: the same order, but no deep recursion.
pub fn dfs_iterative(graph: &Graph, start: usize) -> Vec<usize> {
    let mut visited = vec![false; graph.vertex_count()];
    let mut order = Vec::new();
    let mut stack = vec![start];
    while let Some(vertex) = stack.pop() {
        if visited[vertex] {
            continue; // it was pushed twice before we reached it
        }
        visited[vertex] = true;
        order.push(vertex);
        // Push in reverse, so that the first neighbour is popped (visited) first.
        for &neighbour in graph.neighbours(vertex).iter().rev() {
            if !visited[neighbour] {
                stack.push(neighbour);
            }
        }
    }
    order
}
```

**What happens:**

1. The stack holds vertices that wait to be visited; `pop` takes the **newest** one, the opposite of the queue in BFS. That single change turns breadth-first into depth-first.
2. Unlike BFS, we check `visited` when we **take** a vertex out, because a vertex can be pushed several times before it is visited (from different neighbours). The second time, `continue` skips it.
3. A stack gives the last pushed neighbour first, so we push in reverse order; then the first neighbour is visited first, and the order is exactly the same as in the recursive version (a test will prove it).

To see the difference, create `examples/e13_long_chain.rs`:

```rust
use p07_algorithms::graphs::{Graph, dfs_iterative, dfs_recursive};

fn main() {
    let n = 200_000;
    let mut graph = Graph::new(n);
    for v in 1..n {
        graph.add_edge(v - 1, v);
    }
    println!("iterative: {} vertices", dfs_iterative(&graph, 0).len());
    println!("recursive: {} vertices", dfs_recursive(&graph, 0).len());
}
```

```bash
cargo run --example e13_long_chain
```

```text
iterative: 200000 vertices

thread 'main' (5429589) has overflowed its stack
fatal runtime error: stack overflow, aborting
```

The explicit stack handles 200,000 vertices; the recursion crashes. (The number in parentheses is a thread id and differs on every run; a release build crashes too.) For graphs built from untrusted data, always prefer BFS or the iterative DFS.

> 🔒 **Security:** A program that recurses once per vertex of an input graph can be crashed by anyone who sends a long chain. Limit the depth or use an explicit stack when the input is not trusted.

**Cost.** The same as BFS: each vertex is visited once and each edge is looked at once, **O(V + E)** time; the memory is **O(V)** (the flags and the stack).

### BFS or DFS?

| | BFS | DFS |
|---|---|---|
| Data structure | queue (first in, first out) | stack (last in, first out) or recursion |
| Visits | wave by wave, nearest first | one deep path at a time |
| Finds | the path with the **fewest edges** | *a* path (not necessarily the shortest) |
| Typical uses | shortest route by steps, distance levels, mazes | connected components, cycle detection, ordering tasks (Lesson 07.14) |

One more use of the same walk: a loop over all vertices, starting a new search at each unvisited one, counts the **connected components**. Add:

```rust
/// The number of connected components of an undirected graph
/// (groups of vertices that can reach each other).
pub fn count_components(graph: &Graph) -> usize {
    let mut seen = vec![false; graph.vertex_count()];
    let mut components = 0;
    for start in 0..graph.vertex_count() {
        if seen[start] {
            continue; // already part of an earlier component
        }
        components += 1;
        seen[start] = true;
        let mut stack = vec![start];
        while let Some(vertex) = stack.pop() {
            for &neighbour in graph.neighbours(vertex) {
                if !seen[neighbour] {
                    seen[neighbour] = true;
                    stack.push(neighbour);
                }
            }
        }
    }
    components
}
```

Every search starts at a vertex that no earlier search has reached, so each search finds one new component. The `seen` flags are shared, so the total is still O(V + E).

## 5. Testing graph code

A graph test has two difficulties: you need inputs, and you need the right answer. For inputs we generate random graphs. For the answer we write a **slow, obviously correct reference**: relax the distances over all edges again and again, until nothing changes.
It is O(V · E), too slow for real use, but so simple that a bug is unlikely. Add to the end of `src/graphs.rs`:

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use crate::testutil::Rng;

    /// The graph of the lesson:  0 - 1 - 2     5 - 6 (a separate pair)
    ///                           |   |   |
    ///                           3 - 4 --+
    fn sample() -> Graph {
        let mut graph = Graph::new(7);
        for (a, b) in [(0, 1), (1, 2), (0, 3), (1, 4), (2, 4), (3, 4), (5, 6)] {
            graph.add_edge(a, b);
        }
        graph
    }

    /// A pseudo-random graph (the `Rng` of Lesson 07.3).
    fn random_graph(seed: u64, vertices: usize, edges: usize, directed: bool) -> Graph {
        let mut rng = Rng::new(seed);
        let mut graph = if directed {
            Graph::new_directed(vertices)
        } else {
            Graph::new(vertices)
        };
        for _ in 0..edges {
            let from = rng.below(vertices);
            let to = rng.below(vertices);
            graph.add_edge(from, to);
        }
        graph
    }

    /// A slow but obviously correct reference: relax all edges until nothing changes.
    fn reference_distances(graph: &Graph, start: usize) -> Vec<Option<usize>> {
        let mut distance = vec![None; graph.vertex_count()];
        distance[start] = Some(0);
        loop {
            let mut changed = false;
            for vertex in 0..graph.vertex_count() {
                if let Some(d) = distance[vertex] {
                    for &next in graph.neighbours(vertex) {
                        if distance[next].is_none_or(|old| d + 1 < old) {
                            distance[next] = Some(d + 1);
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
    fn counts_vertices_and_edges() {
        let graph = sample();
        assert_eq!(graph.vertex_count(), 7);
        assert_eq!(graph.edge_count(), 7);
        assert_eq!(graph.neighbours(1), [0, 2, 4]);
    }

    #[test]
    fn bfs_distances_and_paths() {
        let graph = sample();
        let result = bfs(&graph, 0);
        assert_eq!(
            result.distance,
            [Some(0), Some(1), Some(2), Some(1), Some(2), None, None]
        );
        assert_eq!(shortest_path(&graph, 0, 2), Some(vec![0, 1, 2]));
        assert_eq!(shortest_path(&graph, 3, 3), Some(vec![3]));
        assert_eq!(shortest_path(&graph, 0, 6), None);
    }

    #[test]
    fn dfs_orders_agree() {
        let graph = sample();
        assert_eq!(dfs_recursive(&graph, 0), [0, 1, 2, 4, 3]);
        assert_eq!(dfs_iterative(&graph, 0), [0, 1, 2, 4, 3]);
        for seed in 1..=20 {
            let graph = random_graph(seed, 30, 40, seed % 2 == 0);
            assert_eq!(
                dfs_recursive(&graph, 0),
                dfs_iterative(&graph, 0),
                "seed {seed}"
            );
        }
    }

    #[test]
    fn bfs_matches_the_reference() {
        for seed in 1..=30 {
            for directed in [false, true] {
                let graph = random_graph(seed, 25, 35, directed);
                for start in 0..graph.vertex_count() {
                    assert_eq!(
                        bfs(&graph, start).distance,
                        reference_distances(&graph, start),
                        "seed {seed}, directed {directed}, start {start}"
                    );
                }
            }
        }
    }

    #[test]
    fn components() {
        assert_eq!(count_components(&sample()), 2);
        assert_eq!(count_components(&Graph::new(4)), 4);
        assert_eq!(count_components(&Graph::new(0)), 0);
    }

    #[test]
    fn a_long_chain_is_fine_for_bfs_and_iterative_dfs() {
        let n = 100_000;
        let mut graph = Graph::new(n);
        for v in 1..n {
            graph.add_edge(v - 1, v);
        }
        assert_eq!(bfs(&graph, 0).distance[n - 1], Some(n - 1));
        assert_eq!(dfs_iterative(&graph, 0).len(), n);
    }

    #[test]
    #[should_panic(expected = "a vertex number must be below 5")]
    fn add_edge_checks_vertex_numbers() {
        Graph::new(5).add_edge(0, 7);
    }
}
```

**What happens:**

1. `sample()` is the graph of the pictures, so the expected values in the tests are the ones we derived by hand (distances 0, 1, 2, 1, 2 and the DFS order `0 1 2 4 3`).
2. `random_graph` creates graphs with random edges, including repeated edges and loops from a vertex to itself. Such "dirty" inputs are good for tests. `rng.below(vertices)` gives a vertex number from 0 to `vertices - 1`.
3. `bfs_matches_the_reference` compares BFS with the slow reference for 30 seeds × 2 kinds (undirected and directed) × all 25 start vertices: 1,500 checks. `dfs_orders_agree` proves that both DFS versions visit in the same order.
4. The long-chain test shows that BFS and the iterative DFS survive 100,000 vertices in a test thread. `#[should_panic(expected = "...")]` checks that `add_edge` refuses a wrong vertex number with our message.

```bash
cd ~/projects/rust/code/p07-algorithms
cargo test graphs
```

```text
running 7 tests
test graphs::tests::components ... ok
test graphs::tests::counts_vertices_and_edges ... ok
test graphs::tests::bfs_distances_and_paths ... ok
test graphs::tests::dfs_orders_agree ... ok
test graphs::tests::add_edge_checks_vertex_numbers - should panic ... ok
test graphs::tests::bfs_matches_the_reference ... ok
test graphs::tests::a_long_chain_is_fine_for_bfs_and_iterative_dfs ... ok

test result: ok. 7 passed; 0 failed; 0 ignored; 0 measured; 46 filtered out; finished in 0.01s
```

(The 46 filtered-out tests come from the main text of Lessons 07.1 to 07.12; with the tests of the exercises your number is bigger.)

> 🧪 **Testing:** For algorithms with a simple definition ("the shortest path is the smallest number of edges"), write the slowest obviously-correct solution first, and use it as the referee. It costs ten lines and
> finds the bugs in the clever version.

## 6. What to use in real code

The standard library has no graph type; `Vec<Vec<usize>>` (as here) and `VecDeque` are the standard tools, and `HashMap<K, Vec<K>>` when the vertices are not numbers. For big or complicated graphs use the `petgraph` crate
(version 0.8 at the time of writing), which offers graph types and the algorithms of this phase. Know how BFS and DFS work, because you will use them directly in every problem that has "steps", "neighbours", "regions" or "dependencies".

## Common mistakes

**1. Changing the graph while looping over its neighbours.** The list returned by `neighbours` is borrowed from the graph:

```rust
use p07_algorithms::graphs::Graph;

pub fn double_the_edges_of_zero(graph: &mut Graph) {
    for &neighbour in graph.neighbours(0) {
        graph.add_edge(0, neighbour);
    }
}
```

```text
error[E0502]: cannot borrow `*graph` as mutable because it is also borrowed as immutable
 --> src/lib.rs:5:9
  |
4 |       for &neighbour in graph.neighbours(0) {
  |       -                 -------------------
  |       |                 |
  |       |                 immutable borrow occurs here
  |  _____|                 immutable borrow later used here
  | |
5 | |         graph.add_edge(0, neighbour);
  | |         ^^^^^^^^^^^^^^^^^^^^^^^^^^^^ mutable borrow occurs here
6 | |     }
  | |_____- this for loop borrows `*graph` immutably, preventing mutation within its body
```

The loop holds a shared borrow of the list while `add_edge` wants to change it (and if it could, the list might grow and move in memory while we read it). Fix: copy the neighbours first: `let neighbours = graph.neighbours(0).to_vec();`, then loop over the copy.

**2. No "seen" marks.** Without `distance[neighbour].is_none()` (or `visited`), a graph with a cycle sends the search around the cycle forever, and the queue or stack grows without end. Every graph search needs a "seen" set.

**3. Marking a vertex too late in BFS.** If BFS marks a vertex only when it *leaves* the queue, the same vertex can be put into the queue many times (once from each neighbour that sees it first). The result is still correct,
but the queue can grow far beyond V and the time grows. Mark at the moment of `push_back`. (The DFS with an explicit stack is different: there the duplicates are handled by the check after `pop`.)

**4. Recursion on untrusted or huge graphs.** Section 4: use the explicit stack.

## Try it yourself

1. Add the code of Sections 2 to 5 step by step. Run `cargo test graphs`.
2. Run `cargo run --example e13_long_chain`. Change `n` to `10_000`. Does the recursive version survive? Find the size where it starts to crash on your computer.
3. In `bfs`, replace `queue.pop_front()` by `queue.pop_back()` (this makes it a stack). Run the tests. Which tests fail? What did you build instead of BFS?
4. Run `cargo fmt`, `cargo clippy --all-targets`, and commit: `git add code/p07-algorithms && git commit -m "feat(p07): graphs, BFS and DFS"`.

## Exercises

### Exercise 1 — BFS by hand (easy)

For the sample graph, run BFS from vertex `4` by hand. Write the queue after each step and the final list of distances for the vertices `0` to `6`. In what order are the vertices taken out of the queue?

<details>
<summary>Hint</summary>

The neighbours of `4`, in the order the edges were added, are `[1, 2, 3]`; the neighbours of `1` are `[0, 2, 4]`. Vertices `5` and `6` are not connected to `4`.

</details>

<details>
<summary>Answer</summary>

```text
start:   queue [4]           4:0
take 4:  new 1, 2, 3         queue [1, 2, 3]     1:1 2:1 3:1
take 1:  new 0               queue [2, 3, 0]     0:2
take 2:  nothing new         queue [3, 0]
take 3:  nothing new         queue [0]
take 0:  nothing new         queue []
```

Order out of the queue: `4, 1, 2, 3, 0`. Distances for vertices 0 to 6: `[2, 1, 1, 1, 0, none, none]`.

</details>

### Exercise 2 — A property test (easy)

Write a test: in an undirected random graph, for every edge `u - v`, either both ends are reachable from vertex 0 and their BFS distances differ by at most 1, or neither is reachable.

<details>
<summary>Hint</summary>

Compute `distance` once. For every `u` and every `v` in `graph.neighbours(u)`, `match (distance[u], distance[v])`: both `Some`, both `None`, or a mixed case that must never happen. `usize::abs_diff` gives the difference without underflow.

</details>

<details>
<summary>Answer</summary>

```rust
    #[test]
    fn neighbours_differ_by_at_most_one() {
        for seed in 1..=20 {
            let graph = random_graph(seed, 30, 40, false);
            let distance = bfs(&graph, 0).distance;
            for u in 0..graph.vertex_count() {
                for &v in graph.neighbours(u) {
                    match (distance[u], distance[v]) {
                        (Some(a), Some(b)) => assert!(a.abs_diff(b) <= 1),
                        (None, None) => {}
                        _ => panic!("an edge joins a reachable and an unreachable vertex"),
                    }
                }
            }
        }
    }
```

Why it works: if `u` is at distance `a`, its neighbour `v` is at distance at most `a + 1` (go through `u`), and symmetrically `u` is at most `b + 1`, so the distances differ by at most 1. And if one end of an edge is reachable, the other end is too.
This test checks a *property* of BFS without knowing the right answer.

</details>

### Exercise 3 — State the complexity (medium)

1. What are the time and the extra memory of BFS on a graph with V vertices and E edges, stored as an adjacency list? What changes with an adjacency matrix?
2. In the worst case, how deep is the recursion of `dfs_recursive`, and what is the largest size of the queue in `bfs` for a graph that is a star (one centre vertex joined to V − 1 others)?
3. Why does the check `visited[vertex]` after `pop` not make `dfs_iterative` slower than O(V + E)?

<details>
<summary>Hint</summary>

For (2): in a chain, how many vertices wait on the call stack? In a star, how many vertices does the centre put into the queue at once? For (3): how many times can a vertex be pushed at most?

</details>

<details>
<summary>Answer</summary>

1. Adjacency list: **O(V + E)** time, **O(V)** extra memory. With an adjacency matrix, finding the neighbours of a vertex scans a row of V cells, so BFS costs **O(V²)**, whatever E is.
2. Recursion depth up to **V** (a chain). The queue of BFS on a star holds up to **V − 1** vertices at once (all leaves wait after the centre is taken out). The memory of BFS is O(V) in the worst case.
3. A vertex is pushed at most once per edge that leads to it, so the total number of pushes is at most E; the number of pops is the same. Skipping an already visited vertex costs O(1). Total **O(V + E)**.

</details>

### Exercise 4 — Levels (medium)

Write `levels(graph: &Graph, start: usize) -> Vec<Vec<usize>>` that groups the reachable vertices by their BFS distance: element 0 is `[start]`, element 1 holds the vertices at distance 1, and so on. Test it on the sample graph from vertex 0.

<details>
<summary>Hint</summary>

Call `bfs` and go through `distance` with `enumerate()`. For a vertex with distance `Some(d)`, make sure `levels` has at least `d + 1` groups (`levels.resize(d + 1, Vec::new())`) and push the vertex into `levels[d]`.

</details>

<details>
<summary>Answer</summary>

```rust
/// The vertices grouped by their distance from `start`: level 0 is `start` itself.
pub fn levels(graph: &Graph, start: usize) -> Vec<Vec<usize>> {
    let distance = bfs(graph, start).distance;
    let mut levels: Vec<Vec<usize>> = Vec::new();
    for (vertex, d) in distance.iter().enumerate() {
        if let Some(d) = *d {
            if levels.len() <= d {
                levels.resize(d + 1, Vec::new());
            }
            levels[d].push(vertex);
        }
    }
    levels
}
```

The test, inside the `tests` module:

```rust
    #[test]
    fn bfs_from_vertex_four() {
        let graph = sample();
        let result = bfs(&graph, 4);
        assert_eq!(
            result.distance,
            [Some(2), Some(1), Some(1), Some(1), Some(0), None, None]
        );
        assert_eq!(levels(&graph, 0), [vec![0], vec![1, 3], vec![2, 4]]);
    }
```

Why it works: BFS gives each reachable vertex its distance, and the distances in a BFS are consecutive (if there is a vertex at distance `d > 0`, there is one at `d − 1`), so `resize` never leaves an empty gap. The test also repeats the result of Exercise 1.

</details>

### Exercise 5 — Two groups (hard)

A graph is *bipartite* if its vertices can be split into two groups so that every edge joins vertices from different groups (for example, "teams" so that nobody has a rival in their own team). Write `is_bipartite(graph: &Graph) -> bool` with BFS: give the start vertex one colour, give its neighbours the other colour, and so on; if an edge joins two vertices of the same colour, the answer is `false`. Handle several components. Test it on cycles (even and odd length), on a loop edge, and compare with a brute force over all colourings for small random graphs.

<details>
<summary>Hint</summary>

Keep `colour: Vec<Option<bool>>`. For every uncoloured vertex, start a new BFS with colour `false`. When you see a neighbour with no colour, give it `!colour_of_current` and queue it; when it has the same colour as the current vertex, return `false`. A brute force: for every bit mask of `n` bits, check that every edge `(v, w)` has different bits `v` and `w`.

</details>

<details>
<summary>Answer</summary>

```rust
/// Can the vertices be split into two groups so that every edge joins different groups?
pub fn is_bipartite(graph: &Graph) -> bool {
    let mut colour: Vec<Option<bool>> = vec![None; graph.vertex_count()];
    for start in 0..graph.vertex_count() {
        if colour[start].is_some() {
            continue;
        }
        colour[start] = Some(false);
        let mut queue = VecDeque::from([start]);
        while let Some(vertex) = queue.pop_front() {
            let mine = colour[vertex].expect("a queued vertex has a colour");
            for &neighbour in graph.neighbours(vertex) {
                match colour[neighbour] {
                    None => {
                        colour[neighbour] = Some(!mine);
                        queue.push_back(neighbour);
                    }
                    Some(theirs) if theirs == mine => return false,
                    Some(_) => {}
                }
            }
        }
    }
    true
}
```

The tests, inside the `tests` module:

```rust
    fn cycle(n: usize) -> Graph {
        let mut graph = Graph::new(n);
        for v in 0..n {
            graph.add_edge(v, (v + 1) % n);
        }
        graph
    }

    #[test]
    fn bipartite_examples() {
        assert!(is_bipartite(&cycle(4)));
        assert!(!is_bipartite(&cycle(3)));
        assert!(!is_bipartite(&cycle(5)));
        assert!(is_bipartite(&Graph::new(0)));
        assert!(!is_bipartite(&sample())); // 1, 2 and 4 form a triangle
        let mut loop_graph = Graph::new(2);
        loop_graph.add_edge(1, 1);
        assert!(!is_bipartite(&loop_graph));
    }

    #[test]
    fn bipartite_matches_brute_force() {
        for seed in 1..=40 {
            let graph = random_graph(seed, 7, 6 + (seed % 4) as usize, false);
            let n = graph.vertex_count();
            let brute = (0..1u32 << n).any(|mask| {
                (0..n).all(|v| {
                    graph
                        .neighbours(v)
                        .iter()
                        .all(|&w| (mask >> v) & 1 != (mask >> w) & 1)
                })
            });
            assert_eq!(is_bipartite(&graph), brute, "seed {seed}");
        }
    }
```

Why it works: in a bipartite graph the colours along a BFS must alternate, so the colour of every vertex is forced by its distance from the start of its component (even distance, one colour; odd distance, the other). An edge between two vertices of the same colour closes an odd cycle, which can never be split in two groups. The time is **O(V + E)**. A vertex with an edge to itself is its own neighbour with the same colour, so the loop test returns `false`.

</details>

## Quiz

1. What is the difference between a directed and an undirected graph?

<details>
<summary>Answer</summary>

In an undirected graph an edge works in both directions; in a directed graph an edge goes from one vertex to another only. In an adjacency list, an undirected edge appears in two lists, and a directed edge in one.

</details>

2. Which data structure does BFS use, and which does DFS use?

<details>
<summary>Answer</summary>

BFS uses a queue (first in, first out), DFS uses a stack (last in, first out) or the call stack through recursion.

</details>

3. Why does BFS find the path with the fewest edges?

<details>
<summary>Answer</summary>

It visits the vertices wave by wave: all at distance 1, then all at distance 2, and so on. The first time it reaches a vertex, no shorter path can exist, because all shorter ones would have been tried earlier.

</details>

4. Why can recursive DFS crash on a large graph, and what is the fix?

<details>
<summary>Answer</summary>

Each recursive call uses space on the call stack, and the depth can reach V (for a long chain). The call stack is small, so it overflows. Use DFS with an explicit stack kept in a `Vec`.

</details>

## Summary

- A graph is vertices plus edges; edges can be directed, and a path, a cycle and a connected component are the basic words.
- The usual storage is an adjacency list, `Vec<Vec<usize>>`: O(V + E) memory and fast access to the neighbours. A matrix needs O(V²).
- BFS uses a queue, visits in waves and finds the path with the fewest edges. Mark vertices when they enter the queue, and keep `distance` and `previous` to rebuild the path.
- DFS uses a stack or recursion and explores one deep path at a time. The recursive version can overflow the call stack on a long chain; the iterative version cannot.
- Both run in O(V + E) time and O(V) memory. A loop that starts a search at each unvisited vertex counts connected components.
- Test graph code with random graphs and a slow, obviously correct reference, plus properties such as "neighbours differ by at most one level".

## Sources and further reading

- [Rust standard library — std::collections::VecDeque](https://doc.rust-lang.org/std/collections/struct.VecDeque.html) — the queue that BFS uses.
- [Rust standard library — Option::is_none_or](https://doc.rust-lang.org/std/option/enum.Option.html#method.is_none_or) — the helper used in the reference implementation.
- [The Rust Programming Language — Storing Lists of Values with Vectors](https://doc.rust-lang.org/book/ch08-01-vectors.html) — the vector chapter, the base of every adjacency list.
- [petgraph on docs.rs](https://docs.rs/petgraph/latest/petgraph/) — a graph library for production code, with BFS, DFS, Dijkstra and more.

---

[🏠 Start](../START-HERE.md) · [← 07.12 Heaps and priority queues](12-heaps-and-priority-queues.md) · [📋 Phase 07](README.md) · [07.14 Shortest paths and topological sort →](14-shortest-paths-and-topological-sort.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
