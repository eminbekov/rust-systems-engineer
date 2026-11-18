<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 07.18 Problem-solving practice](18-problem-solving-practice.md) · [📋 Phase 07](README.md) · [07.20 Checkpoint →](20-checkpoint.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/07-algorithms-and-data-structures/19-project-route-finder.md) <!-- nav -->

# Lesson 07.19 — Project: route finder

> **Phase 07 — Algorithms and data structures** · Lesson 19 of 20 · ⏱ about 6 hours (3 sittings)

## What you will learn

- How to turn a text file of roads into a graph, and a graph algorithm into a command-line tool.
- How to use BFS (Breadth-First Search: fewest roads) and Dijkstra's algorithm (shortest distance) on the same map, and why they give different answers.
- How to split a program into a library and a thin binary, report errors clearly, and test both the code and the tool.
- How to measure how the two algorithms grow with the size of the map.

This is a **project lesson**. It gives you the target (the contract), a plan with small milestones, hidden hints, and commands that prove each milestone is done. The code is yours to write.

## Before you start

- You finished [Lesson 07.18 — Problem-solving practice](18-problem-solving-practice.md) and the graph lessons [07.13](13-graphs-bfs-and-dfs.md) and [07.14](14-shortest-paths-and-topological-sort.md). You will use BFS, Dijkstra and `BinaryHeap` ([Lesson 07.12](12-heaps-and-priority-queues.md)).
- You know the project style of [Lesson 05.13 — Project: minigrep](../05-rust-intermediate/13-project-minigrep.md): a library with the logic, a thin binary, `clap`, `thiserror`, integration tests and a `criterion` benchmark ([Lesson 05.11](../05-rust-intermediate/11-practical-crates-serde-clap-errors.md), [Lesson 05.12](../05-rust-intermediate/12-benchmarking-basics.md)).
- Your project goes to `~/projects/rust/code/p07-route-finder`, inside your learning repository (the root `.gitignore` already ignores `target/`). Run `git status` in `~/projects/rust` first: it should say `nothing to commit, working tree clean`.
- You are online: Cargo must download crates.

## New words

| Word | Meaning |
|---|---|
| **edge list** | A way to store a graph as a list of its edges, one per line: here `from,to,distance`. It is easy to write by hand and to read from a file. |
| **weighted graph** | A graph whose edges have a number (a *weight*), here the length of a road in kilometres. |
| **undirected** | An edge that works in both directions: a road you can drive from A to B and from B to A. |
| **route** | A path from the start city to the goal city: the list of cities, in order. |
| **exit status** | The number a program returns to the shell when it ends. `0` means success; any other number means failure. `echo $?` prints it. |
| **V and E** | The number of vertices (here cities) and of edges (here roads) of a graph. Running times of graph algorithms are written with them: O(V + E). |
| **acceptance test** | A test that checks the finished behaviour from the outside, for example by running the program and reading its output. The "Done when" commands are acceptance tests. |

## The project

### What you will build

`route-finder` reads a map from a text file (a list of two-way roads with lengths) and prints a route between two cities, with the number of roads and the total distance. You choose the algorithm: `bfs` finds the route with the **fewest roads**; `dijkstra` (the default) finds the route with the **shortest total distance**. They often give different routes, and you will see it in your own program.

You will work in this order: set up the command line, read and check the map, build the graph and BFS, add Dijkstra, connect everything with clear errors and exit statuses, then measure and document.

```text
p07-route-finder/
├── Cargo.toml
├── README.md
├── benches/
│   └── routes.rs        ← criterion benchmark (milestone M6)
├── maps/
│   ├── small.txt        ← the sample map of this lesson
│   └── broken.txt       ← a map with a bad line (milestone M5)
├── src/
│   ├── lib.rs           ← the logic: map, graph, BFS, Dijkstra, errors
│   └── main.rs          ← the terminal: clap, printing, exit status
└── tests/
    └── cli.rs           ← integration tests (milestone M5)
```

(You may split `lib.rs` into modules such as `graph.rs` and `map.rs`; the contract below does not care.)

### Contract

**Usage**

```text
route-finder --map <FILE> --from <CITY> --to <CITY> [--algo bfs|dijkstra]
```

| Option | Meaning |
|---|---|
| `--map <FILE>` | Required. The map file. |
| `--from <CITY>`, `--to <CITY>` | Required. The names of the start and the goal city. Names are case-sensitive. |
| `--algo bfs\|dijkstra` | Optional, default `dijkstra`. Any other value is a `clap` error. |
| `-h`, `--help`, `-V`, `--version` | Help and version (made by `clap`). |

**The map file.** Plain text, one road per line: `from,to,distance`.

- Spaces around the three fields are ignored. Empty lines and lines that start with `#` are skipped.
- A road works in **both** directions. A city exists as soon as it appears in a road.
- The distance is a whole number of kilometres, at least 1 (it must fit into `u32`).
- If the same pair of cities appears twice (in either order), the **shorter** distance is kept.

**Errors in the map** (each one stops the program; the line number starts at 1):

| Problem | Message part after the file name |
|---|---|
| a line without exactly three fields | `line 3: expected 3 fields "from,to,distance" but found 2` |
| an empty city name | `line 3: empty city name` |
| a distance that is not a whole number of at least 1 | `line 2: invalid distance "abc" (a whole number of at least 1 is needed)` |
| a road from a city to itself | `line 4: a road cannot start and end in the same city ("A")` |

**Output (stdout)** when a route is found — exactly three lines:

```text
path: Alder -> Cedar -> Fir -> Elm
roads: 3
distance: 20
```

`roads` is the number of roads on the route and `distance` the sum of their lengths in km. If `--from` and `--to` are the same city, the route is that one city: `path: Elm`, `roads: 0`, `distance: 0`.

**Errors (stderr)**, one line each, always starting with `route-finder: `:

| Situation | Line on stderr |
|---|---|
| the map file cannot be read | `route-finder: cannot read maps/missing.txt: No such file or directory (os error 2)` |
| the map has a bad line | `route-finder: maps/broken.txt: line 2: invalid distance "abc" (a whole number of at least 1 is needed)` |
| a city is not on the map | `route-finder: unknown city "Nowhere"` |
| the cities are not connected | `route-finder: no route from Alder to Holly` |

Nothing but a successful result goes to stdout.

**Exit status:** `0` for a found route; `1` for every error above; `2` for a wrong command line (`clap` does that).

**What the library must offer** (so that your tests can use it; the names are fixed, everything else is free):

```rust
pub enum Algo { Bfs, Dijkstra }                       // also a clap ValueEnum
pub struct Graph { /* yours */ }                      // city_count(), road_count()
pub fn parse_map(text: &str) -> Result<Graph, MapError>;
pub struct Route { pub cities: Vec<String>, pub distance: u64 }   // roads() = cities.len() - 1
pub fn find_route(graph: &Graph, from: &str, to: &str, algo: Algo) -> Result<Route, RouteError>;
```

`MapError` and `RouteError` are `thiserror` enums whose messages are the texts above (without `route-finder: ` and without the file name).

**Code rules**

- The logic lives in `src/lib.rs` (and modules). `src/main.rs` only parses the arguments, calls the library, prints, and turns errors into an exit status.
- Use your own BFS and Dijkstra with `std::collections::BinaryHeap` (and `Reverse`) and `VecDeque`. No graph crate.
- No `unwrap()` or `expect()` outside tests, except where an invariant makes a failure impossible (add a comment that says why).
- Dependencies: `clap` (feature `derive`), `thiserror`, `anyhow`; dev-dependencies `assert_cmd`, `predicates`, `tempfile`, `criterion`. Add each one in the milestone that first uses it (Cargo warns about unused dependencies).
- `cargo fmt --check`, `cargo clippy --all-targets -- -D warnings` and `cargo test` pass with no warnings.

**The sample map.** Create `maps/small.txt` in M1. It has eight towns; two of them (`Gorse` and `Holly`) are an island with no road to the rest.

```text
# A small map. Every line is a two-way road: from,to,distance (in km).
Alder,Birch,7
Alder,Cedar,9
Alder,Fir,14
Birch,Cedar,10
Birch,Dogwood,15
Cedar,Dogwood,11
Cedar,Fir,2
Dogwood,Elm,6
Elm,Fir,9

# A second, separate island of two towns: no road leads to it.
Gorse,Holly,3
```

As an adjacency list (each town with its neighbours and the road lengths):

```text
Alder:   Birch 7,   Cedar 9,   Fir 14
Birch:   Alder 7,   Cedar 10,  Dogwood 15
Cedar:   Alder 9,   Birch 10,  Dogwood 11, Fir 2
Dogwood: Birch 15,  Cedar 11,  Elm 6
Elm:     Dogwood 6, Fir 9
Fir:     Alder 14,  Cedar 2,   Elm 9
Gorse:   Holly 3
Holly:   Gorse 3
```

From `Alder` to `Elm`, BFS finds `Alder -> Fir -> Elm`: 2 roads, 14 + 9 = 23 km. Dijkstra finds `Alder -> Cedar -> Fir -> Elm`: 3 roads, 9 + 2 + 9 = 20 km. Fewer roads is not the same as a shorter distance.

### Step-by-step plan

Work through the milestones in order. After each one, run its "Done when" commands, then commit. Do not open the hints before you tried.

> 📏 **Convention:** Commit messages as in Phase 05: a type, the scope in brackets, a colon and a short sentence in the imperative mood: `feat(route-finder): parse the command line`. Types you need: `feat`, `test`, `docs`, `perf`, `chore`. One commit per milestone. Use `git add .` inside the project folder and look at `git status --short` before you commit.

---

#### M1 — Set up the project and the command line

**Goal:** a project that builds, is under Git, and understands the command line of the contract.

**Guidance:**

1. Make a branch and the package: `cd ~/projects/rust && git switch -c project/07-route-finder && cargo new code/p07-route-finder`. In `Cargo.toml` change the package `name` to `route-finder`, so that the binary is called `route-finder`.
2. Add only what this milestone uses: `cargo add clap --features derive`.
3. Create `maps/small.txt` with the text above.
4. In `src/main.rs` define the command-line struct with `#[derive(Parser)]` and an `Algo` enum with `#[derive(ValueEnum)]` (for now in `main.rs`; you move it to the library in M3). Give every argument a doc comment. `main` only parses the arguments and prints them with `{:?}` (derive `Debug` as well).

<details>
<summary>Hint 1 — the fields</summary>

`map: PathBuf`, `from: String`, `to: String`, `algo: Algo`, each marked `#[arg(long)]`. For the enum field use `#[arg(long, value_enum, default_value_t = Algo::Dijkstra)]`. The enum needs `Clone` and `Copy` as well as `ValueEnum` and `Debug` (the default value is copied).

</details>

<details>
<summary>Hint 2 — the version</summary>

`#[command(version)]` on the struct makes `--version` print the package name and version from `Cargo.toml`.

</details>

**Done when:**

```bash
cargo run -q -- -h
```

prints a usage line `Usage: route-finder [OPTIONS] --map <MAP> --from <FROM> --to <TO>`, the four options, and `[default: dijkstra] [possible values: bfs, dijkstra]` for `--algo`. (The description texts are yours.) For example:

```text
Find a route between two cities on a map of roads

Usage: route-finder [OPTIONS] --map <MAP> --from <FROM> --to <TO>

Options:
      --map <MAP>    The map file: one road per line, as from,to,distance
      --from <FROM>  The city to start in
      --to <TO>      The city to go to
      --algo <ALGO>  bfs finds the fewest roads, dijkstra the shortest distance [default: dijkstra] [possible values: bfs, dijkstra]
  -h, --help         Print help (see more with '--help')
  -V, --version      Print version
```

and:

```bash
cargo run -q -- --map maps/small.txt --from Alder --to Elm --algo bfs
cargo run -q -- --map maps/small.txt --from Alder ; echo "exit: $?"
```

```text
Cli { map: "maps/small.txt", from: "Alder", to: "Elm", algo: Bfs }
error: the following required arguments were not provided:
  --to <TO>

Usage: route-finder --map <MAP> --from <FROM> --to <TO>

For more information, try '--help'.
exit: 2
```

Commit: `git add . && git commit -m "feat(route-finder): parse the command line with clap"`.

---

#### M2 — The graph and the map parser

**Goal:** `parse_map` turns the text of a map into a `Graph`, or a `MapError` with the line number. The graph can say how many cities and roads it has.

**Guidance:**

1. Add `cargo add thiserror`. Create `src/lib.rs` (and modules if you like) and define `MapError` with one variant per problem of the contract; the messages come from `#[error("...")]`.
2. Store the graph as an **adjacency list** ([Lesson 07.13](13-graphs-bfs-and-dfs.md)): give every city a number (its *id*), keep the names in a `Vec<String>` and a `HashMap<String, usize>` from the name to the id, and for each id a `Vec<(usize, u32)>` of (neighbour id, road length).
3. `add_road(a, b, km)` creates the cities when they are new and adds the road in **both** directions. If the road already exists, keep the shorter length (in both directions!).
4. `parse_map` goes through `text.lines().enumerate()`, skips empty and `#` lines, splits on `,`, trims, checks the four problems of the contract in a sensible order and calls `add_road`.
5. Tests first (see [Lesson 05.6](../05-rust-intermediate/06-testing-in-depth.md)): the sample map has 8 cities and 10 roads; each error has a test that checks the exact variant, including the line number; `"A,B,10\nB,A,4\n"` gives one road of length 4.
6. For now `main` reads the file, calls `parse_map`, and prints `loaded 8 cities and 10 roads` (this line is temporary; M5 replaces it). Make a library target: with `src/lib.rs` next to `src/main.rs`, `main.rs` can use the library by the package name with underscores: `route_finder::parse_map`.

<details>
<summary>Hint 1 — splitting a line</summary>

`let fields: Vec<&str> = line.split(',').map(str::trim).collect();` then check `fields.len() == 3` first. A distance such as `-5` or `3.5` fails `parse::<u32>()`; `0` parses but the contract wants at least 1. Check the empty name and the self-road before the distance, or after: the contract does not fix the order, but be consistent and test your order.

</details>

<details>
<summary>Hint 2 — the shorter road</summary>

Search the neighbour list of `a` for `b`. If it is there, replace the length by `min(old, km)`, and do the same in the list of `b`. If not, push the new pair to both lists. Keep `road_count` honest: it is half of the sum of the lengths of all neighbour lists.

</details>

**Done when:**

```bash
cargo test
```

shows all your tests passing (at least: the counts, the four errors, the shorter road, comments and empty lines skipped), and:

```bash
cargo run -q -- --map maps/small.txt --from Alder --to Elm
```

```text
loaded 8 cities and 10 roads
```

Commit: `feat(route-finder): read a map into a graph`.

---

#### M3 — BFS: the route with the fewest roads

**Goal:** `find_route(&graph, from, to, Algo::Bfs)` returns the route with the fewest roads, or the right error.

**Guidance:**

1. Define `Route`, `RouteError` (with `thiserror`: `UnknownCity` and `NoRoute`) and move `Algo` into the library (derive `clap::ValueEnum` there; `main.rs` now imports it).
2. Write BFS on ids: a `VecDeque` of cities to visit, a `seen` vector, and a `previous: Vec<Option<usize>>` that remembers from which city you reached each city ([Lesson 07.13](13-graphs-bfs-and-dfs.md)). When you reach the goal, follow `previous` back to the start and reverse.
3. `find_route` turns the names into ids (unknown name → `UnknownCity`), runs BFS (nothing found → `NoRoute { from, to }`), turns the ids back into names, and sums the road lengths along the path into `distance` (use `u64`).
4. For `Algo::Dijkstra` put `todo!("M4")` for now.
5. Tests: from `Alder` to `Elm` the path is `Alder, Fir, Elm` with 2 roads and 23 km; from a city to itself the route is that city with 0 roads; `Alder` to `Holly` is `NoRoute`; `Alder` to `Nowhere` is `UnknownCity("Nowhere")`; a hand-made line graph `A-B-C-D` gives the path `A, B, C, D`.

<details>
<summary>Hint 1 — the distance of a path</summary>

`path.windows(2)` gives each pair of neighbouring cities on the path ([Lesson 07.15](15-two-pointers-and-sliding-window.md)). You need a way to ask the graph for the length of the road between two neighbours; a small method `road_length(a, b) -> Option<u32>` that searches the neighbour list is enough. The path came from the graph, so a missing road is impossible; say that in a comment where you unwrap.

</details>

<details>
<summary>Hint 2 — the path</summary>

Start with `vec![goal]`, then repeat `current = previous[current]` and push, until `current == start`. The vector is backwards, so call `reverse()`. When `from == to`, the loop does not run and the path is `[goal]`.

</details>

**Done when:** `cargo test` passes with your BFS tests (at least the five above), `cargo clippy --all-targets -- -D warnings` is silent, and `cargo run -q -- --map maps/small.txt --from Alder --to Elm` still prints `loaded 8 cities and 10 roads` (main is not connected yet).

Commit: `feat(route-finder): find the route with the fewest roads (BFS)`.

---

#### M4 — Dijkstra: the shortest distance

**Goal:** `Algo::Dijkstra` returns the route with the smallest total distance.

**Guidance:**

1. Keep `best: Vec<u64>` (the shortest known distance to each city, `u64::MAX` at the start, 0 for the start city), `previous` as before, and a **min-heap** of (distance, city). `BinaryHeap` is a max-heap, so store `Reverse((distance, city))` ([Lesson 07.12](12-heaps-and-priority-queues.md)). You wrote the same algorithm in [Lesson 07.14](14-shortest-paths-and-topological-sort.md); here it works on city ids and may stop as soon as it pops the goal.
2. Pop the closest city. If its popped distance is bigger than `best[city]`, it is an old entry: skip it. If it is the goal, stop and rebuild the path. Otherwise, for each neighbour, if `distance + road` is smaller than `best[neighbour]`, update `best` and `previous` and push.
3. The same `find_route` code builds the `Route`; the distance now comes straight from the algorithm.
4. Tests: `Alder` to `Elm` is `Alder, Cedar, Fir, Elm` with 3 roads and 20 km (and BFS gave 23 km: write a test that says so); `Alder` to `Holly` is `NoRoute`; the shorter of two parallel roads is used. Then write a **judge** test: compare Dijkstra with a slow, obviously correct function on many small random maps (hint below).

<details>
<summary>Hint 1 — a judge for Dijkstra</summary>

The slow function: keep a vector of best distances (`Option<u64>`), set the start to `Some(0)`, and repeat "for every road, in both directions, try to improve the other end" until a whole pass changes nothing. It is O(cities × roads) but it is very hard to get wrong. Generate maps with a seeded pseudo-random generator: copy `src/testutil.rs` with its `Rng` from `code/p07-algorithms` ([Lesson 07.3](03-arrays-and-vectors-under-the-hood.md), with `range` from [Lesson 07.15](15-two-pointers-and-sliding-window.md)) into this package and declare it with `#[cfg(test)] mod testutil;` in `src/lib.rs`. Then make maps with, for example, up to 8 cities and 14 roads of length 1 to 9, and compare the distances from city `c0` to every city.

</details>

<details>
<summary>Hint 2 — why skip old entries?</summary>

A city can be pushed several times, each time with a shorter distance. The heap cannot update an entry in place, so the old, longer entries stay inside. When one comes out, `best[city]` is already smaller. Without the check, you do the neighbour work again for nothing.

</details>

**Done when:** `cargo test` passes (BFS and Dijkstra tests, including the judge), clippy is silent.

Commit: `feat(route-finder): find the shortest route (Dijkstra)`.

---

#### M5 — The tool: output, errors, exit statuses

**Goal:** the program behaves exactly like the contract, and integration tests prove it.

**Guidance:**

1. `cargo add anyhow`. In `src/main.rs`: read the file (`std::fs::read_to_string`), `parse_map`, `find_route`, print the three lines. Put this in a function `run(&Cli) -> anyhow::Result<()>` and let `main` call it, print `route-finder: {error:#}` to stderr on `Err` and return `ExitCode::FAILURE` (status 1) — [Lesson 05.13](../05-rust-intermediate/13-project-minigrep.md) did the same.
2. Use `.with_context(...)` ([Lesson 05.11](../05-rust-intermediate/11-practical-crates-serde-clap-errors.md)) so that the messages match the contract: `cannot read <file>` for the read error, and the file name in front of a `MapError`. The format `{error:#}` prints the whole chain separated by `: `.
3. Create `maps/broken.txt` with two lines: `A,B,5` and `B,C,abc` (for example `printf 'A,B,5\nB,C,abc\n' > maps/broken.txt`).
4. Integration tests in `tests/cli.rs` with `assert_cmd` ([Lesson 05.13](../05-rust-intermediate/13-project-minigrep.md)): `cargo add --dev assert_cmd predicates tempfile`. Test at least: the default (Dijkstra) route; `--algo bfs`; a missing route (status 1, empty stdout, the exact stderr line); an unknown city; a bad map written to a temporary file with `tempfile` (the message contains the file name and `line 2`); a missing file; a wrong command line (status 2).

<details>
<summary>Hint 1 — the test skeleton</summary>

```rust
use assert_cmd::Command;

#[test]
fn finds_the_shortest_route_by_default() {
    Command::cargo_bin("route-finder")
        .unwrap()
        .args(["--map", "maps/small.txt", "--from", "Alder", "--to", "Elm"])
        .assert()
        .success()
        .stdout("path: Alder -> Cedar -> Fir -> Elm\nroads: 3\ndistance: 20\n");
}
```

Integration tests run with the package folder as the current directory, so the relative path `maps/small.txt` works. `.stderr(predicates::str::contains("line 2"))` checks a part of a message; `.code(1)` checks the exit status.

</details>

<details>
<summary>Hint 2 — the error chain</summary>

`parse_map(&text).with_context(|| cli.map.display().to_string())?` wraps the `MapError` in a context that is the file name; with `{error:#}` it prints `maps/broken.txt: line 2: invalid distance ...`. `find_route(...)?` converts a `RouteError` into `anyhow::Error` by itself, because it implements `std::error::Error`.

</details>

**Done when:**

```bash
cargo run -q -- --map maps/small.txt --from Alder --to Elm
cargo run -q -- --map maps/small.txt --from Alder --to Elm --algo bfs
cargo run -q -- --map maps/small.txt --from Elm --to Elm
```

```text
path: Alder -> Cedar -> Fir -> Elm
roads: 3
distance: 20
path: Alder -> Fir -> Elm
roads: 2
distance: 23
path: Elm
roads: 0
distance: 0
```

and each of these prints one line on stderr and ends with `exit: 1`:

```bash
cargo run -q -- --map maps/small.txt --from Alder --to Holly ; echo "exit: $?"
cargo run -q -- --map maps/small.txt --from Alder --to Nowhere ; echo "exit: $?"
cargo run -q -- --map maps/broken.txt --from A --to C ; echo "exit: $?"
cargo run -q -- --map maps/missing.txt --from A --to C ; echo "exit: $?"
```

```text
route-finder: no route from Alder to Holly
exit: 1
route-finder: unknown city "Nowhere"
exit: 1
route-finder: maps/broken.txt: line 2: invalid distance "abc" (a whole number of at least 1 is needed)
exit: 1
route-finder: cannot read maps/missing.txt: No such file or directory (os error 2)
exit: 1
```

`cargo test` passes (unit and integration tests), `cargo fmt --check` and `cargo clippy --all-targets -- -D warnings` are silent.

Commit: `feat(route-finder): print routes and report errors with exit statuses`.

---

#### M6 — Measure, document, release

**Goal:** a benchmark that shows how BFS and Dijkstra grow with the size of the map, a README, and a tag.

**Guidance:**

1. `cargo add --dev criterion@0.8`, then declare the benchmark in `Cargo.toml` (**after** you create the file, see Common mistakes):

```toml
[[bench]]
name = "routes"
harness = false
```

2. In `benches/routes.rs` build maps shaped like a **grid** of `size × size` towns, with roads between right and lower neighbours. Parse each map **once** (outside the timed code), then time `find_route` from the first corner to the opposite corner, for BFS and Dijkstra, for sizes 10, 30 and 100 (100, 900 and 10,000 towns). Use `benchmark_group` and `BenchmarkId` so that the report shows both algorithms side by side. Use `std::hint::black_box` ([Lesson 05.12](../05-rust-intermediate/12-benchmarking-basics.md)).
3. Run it with `cargo bench --bench routes -- --warm-up-time 1 --measurement-time 3` and put the six numbers in your README as a table, with the sentence "measured on my computer; numbers differ per computer".
4. Write `README.md`: what the tool does, the usage line, the map format, an example run of each algorithm on `maps/small.txt`, the complexity of both algorithms (BFS: O(V + E); Dijkstra with a binary heap: O((V + E) log V)), the table of your measurements, and how to run the tests.
5. Final checks and a tag.

<details>
<summary>Hint 1 — the grid map text</summary>

Name the towns `t{row}_{col}` and write the roads as text, then call `parse_map` on it:

```rust
fn grid_map(size: usize) -> String {
    let mut text = String::new();
    for row in 0..size {
        for col in 0..size {
            let km = 1 + (row * 7 + col * 13) % 9; // lengths from 1 to 9
            if col + 1 < size {
                text.push_str(&format!("t{row}_{col},t{row}_{},{km}\n", col + 1));
            }
            if row + 1 < size {
                text.push_str(&format!("t{row}_{col},t{}_{col},{km}\n", row + 1));
            }
        }
    }
    text
}
```

The goal city is `format!("t{}_{}", size - 1, size - 1)`.

</details>

<details>
<summary>Hint 2 — benchmarking several sizes</summary>

`group.bench_with_input(BenchmarkId::new("bfs", size * size), &size, |b, _| b.iter(|| ...))` creates one entry per algorithm and size. The loop over sizes goes outside, the loop over the two algorithms inside.

</details>

**Done when:**

```bash
cargo bench --bench routes -- --warm-up-time 1 --measurement-time 3
```

prints six timings, `bfs` and `dijkstra` for 100, 900 and 10000 towns. Example output (yours will be different; only the shape matters: both grow roughly in proportion to the number of towns, and Dijkstra is a few times slower than BFS because of the heap):

```text
corner to corner/bfs/100
                        time:   [1.0759 µs 1.0829 µs 1.0898 µs]
corner to corner/dijkstra/100
                        time:   [1.9750 µs 1.9903 µs 2.0097 µs]
corner to corner/bfs/900
                        time:   [5.1621 µs 5.1828 µs 5.2069 µs]
corner to corner/dijkstra/900
                        time:   [15.430 µs 15.502 µs 15.576 µs]
corner to corner/bfs/10000
                        time:   [53.554 µs 53.865 µs 54.138 µs]
corner to corner/dijkstra/10000
                        time:   [217.96 µs 224.54 µs 232.52 µs]
```

(Criterion also prints progress lines and outlier reports; they are left out. These numbers come from an Apple M4 Mac.) Then:

```bash
cargo fmt --check && cargo clippy --all-targets -- -D warnings && cargo test
git add . && git commit -m "perf(route-finder): benchmark both algorithms, add the README"
git tag -a route-finder-v0.1.0 -m "route-finder 0.1.0"
git switch main && git merge project/07-route-finder
```

`git tag` lists `route-finder-v0.1.0`, and `git log --oneline -6` shows one commit per milestone.

### Stretch goals

- **`--all`:** print the distance from the start city to every reachable city, sorted by name (Dijkstra without a goal).
- **Map from standard input:** `--map -` reads the map from stdin, so that `cat maps/small.txt | route-finder --map - ...` works.
- **One-way roads:** a line `from>to,distance` is a road in one direction only. What must change in `add_road` and in the road count?
- **Ties:** when two routes have the same distance, always print the one that is first alphabetically. Write the test first.
- **A\* search:** add coordinates to the cities, and use the straight-line distance as a guess to make Dijkstra faster. Compare with the benchmark.
- **A second check:** build a map in which every road is 1 km long. BFS and Dijkstra must then find routes with the same number of roads. Write that test.

## Common mistakes

**1. Declaring the benchmark before the file exists.** If `Cargo.toml` has `[[bench]] name = "routes"` and `benches/routes.rs` is missing, **every** Cargo command fails, not only `cargo bench`:

```text
error: Cargo.toml: can't find `routes` bench at `benches/routes.rs` or `benches/routes/main.rs`. Please specify bench.path if you want to use a non-default path.
error: could not parse `route-finder` (manifest) due to 1 previous error
```

**Fix:** create `benches/routes.rs` first, and add the `[[bench]]` entry afterwards.

**2. Using BFS and calling it "the shortest route".** BFS counts roads, not kilometres. On the sample map it prints 23 km where Dijkstra finds 20. The word "shortest" needs a definition; the contract defines two.

**3. Forgetting that the same city can be pushed to the heap many times.** The program still gives the right answer, but it does extra work. The check "is this heap entry old?" is a small line that keeps Dijkstra at O((V + E) log V).

**4. Letting `main` do the work.** If the logic is in `main.rs`, you can only test it by starting the program. Keep `main` thin and test the library with fast unit tests, and the tool with a few integration tests.

## Quiz

1. Why does BFS give 23 km and Dijkstra 20 km from `Alder` to `Elm`?

<details>
<summary>Answer</summary>

BFS finds the route with the fewest roads (2 roads, 14 + 9 = 23 km), and it ignores the lengths. Dijkstra finds the smallest sum of lengths (3 roads, 9 + 2 + 9 = 20 km).

</details>

2. Why can Dijkstra's algorithm not be used with negative road lengths?

<details>
<summary>Answer</summary>

It takes the closest unvisited city and treats its distance as final. That is correct only if a later road can never make a path shorter. With a negative road it could. (Distances in this project are at least 1.)

</details>

3. What are the time complexities of BFS and Dijkstra (with a binary heap) on a graph with V cities and E roads?

<details>
<summary>Answer</summary>

BFS: O(V + E). Dijkstra: O((V + E) log V), because each heap operation costs O(log V). Our measurement shows both growing about in proportion to the number of towns on a grid, with Dijkstra several times slower.

</details>

4. Why is the library separate from `main.rs`?

<details>
<summary>Answer</summary>

So that the logic can be tested directly and quickly with unit tests (and by the benchmark), without starting a program. `main.rs` only connects the terminal (arguments, printing, exit status) to the library.

</details>

5. The map has the line `A,B,10` and later `B,A,4`. What is the road between A and B?

<details>
<summary>Answer</summary>

One two-way road of 4 km: the contract keeps the shorter length, in both directions.

</details>

## Summary

- A text file of roads is an edge list; reading it into an adjacency list gives a graph that BFS and Dijkstra can walk.
- BFS finds the route with the fewest roads in O(V + E); Dijkstra finds the shortest total distance in O((V + E) log V) with a min-heap (`BinaryHeap<Reverse<_>>`). "Shortest" always needs a definition.
- A library plus a thin binary lets you test the logic with unit tests and the tool with integration tests; `thiserror` gives precise errors in the library and `anyhow` adds context in `main`.
- Clear errors have a fixed shape (`route-finder: ...` on stderr) and fixed exit statuses (0, 1, 2); the acceptance tests check both.
- A slow, obviously correct function is the best judge for a clever algorithm; compare on many small random maps.
- Benchmark over several input sizes to see growth, not just one number; record the numbers in the README and say that they differ per computer.

## Sources and further reading

- [The Rust Standard Library — BinaryHeap](https://doc.rust-lang.org/std/collections/struct.BinaryHeap.html) — the heap, and the Dijkstra example in its documentation (read it after you finish your own).
- [The Rust Standard Library — VecDeque](https://doc.rust-lang.org/std/collections/struct.VecDeque.html) — the queue for BFS.
- [clap documentation (derive tutorial)](https://docs.rs/clap/latest/clap/_derive/_tutorial/index.html) — arguments, `ValueEnum` and default values.
- [thiserror](https://docs.rs/thiserror/latest/thiserror/) and [anyhow](https://docs.rs/anyhow/latest/anyhow/) — error types and error context.
- [assert_cmd](https://docs.rs/assert_cmd/latest/assert_cmd/) and [predicates](https://docs.rs/predicates/latest/predicates/) — testing a command-line program.
- [Criterion.rs User Guide](https://bheisler.github.io/criterion.rs/book/) — benchmark groups and `BenchmarkId`.

---

[🏠 Start](../START-HERE.md) · [← 07.18 Problem-solving practice](18-problem-solving-practice.md) · [📋 Phase 07](README.md) · [07.20 Checkpoint →](20-checkpoint.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
