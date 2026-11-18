<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 22.16 Databases track: executing queries](16-databases-track-executing-queries.md) · [📋 Phase 22](README.md) · [22.18 Networking track: high-performance servers →](18-networking-track-high-performance-servers.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/22-specializations/17-databases-track-project-and-next-steps.md) <!-- nav -->

# Lesson 22.17 — Databases track: project and next steps

> **Phase 22 — Specializations** · Lesson 17 of 30 · ⏱ about 20 hours (10 sittings)

## What you will learn

- How to grow the parser and executor of Lessons 22.15–22.16 into a small, complete query tool.
- How to infer a type per column, and how SQL's (Structured Query Language) `NULL` changes comparisons, filters and aggregates.
- How to add `ORDER BY` and `LIMIT`, and why `NULL` needs a defined place in the sort order.
- How to test a query engine against DataFusion (differential testing) and measure it.
- Where to go next in database engineering: books, courses, projects and communities.

## Before you start

- You finished [Lesson 22.16 — Databases track: executing queries](16-databases-track-executing-queries.md)
  and have `code/p22-query` with `plan.rs` and `exec.rs`.
- You can write a command-line program with `clap` and errors with `thiserror` or `anyhow`
  ([Lesson 05.11](../05-rust-intermediate/11-practical-crates-serde-clap-errors.md)) and test command-line
  programs end to end ([Lesson 09.2](../09-testing-debugging-benchmarking/02-testing-command-line-programs.md)).
- You know `hyperfine` from [Lesson 09.11](../09-testing-debugging-benchmarking/11-benchmarking-in-depth.md).
- A new Git repository for the project: `cargo new code/p22-minisql`; one commit per milestone.

## New words

| Word | Meaning |
|---|---|
| **`NULL`** | SQL's marker for "no value" or "unknown". In a CSV file, an empty field. |
| **three-valued logic** | SQL logic with three results: true, false and unknown (`NULL`). `NULL = 1` is unknown, not false. |
| **schema inference** | Guessing the type of each column by looking at its values. |
| **type coercion** | Converting a value to another type so that two values can be compared, for example `Int 7` to `Float 7.0`. |
| **null ordering** | The rule that says where `NULL` goes when sorting: first or last. |

## 1. `NULL` in five rules

Real data has holes: a person whose age is unknown. SQL marks them with `NULL`, and `NULL` is not a value
like the others. It means "unknown", so most operations with it give "unknown" too:

| Expression | Result | Why |
|---|---|---|
| `NULL = 30`, `NULL < 30`, `NULL = NULL` | `NULL` | comparing with something unknown gives unknown |
| `age IS NULL` | `true` or `false` | the only safe way to test for `NULL` |
| `NOT NULL` | `NULL` | the opposite of unknown is still unknown |
| `false AND NULL` / `true OR NULL` | `false` / `true` | the result is known whatever the unknown is |
| `true AND NULL` / `false OR NULL` | `NULL` | the result depends on the unknown |

A `WHERE` clause keeps a row **only if** the condition is `true`; `false` and `NULL` both drop it. For
aggregates: `COUNT(*)` counts rows, `COUNT(col)` counts non-`NULL` values, and `SUM`, `AVG`, `MIN`, `MAX`
ignore `NULL`s; over no values they return `NULL`.

> ⚠️ **Common mistake:** `WHERE age <> 30` does **not** return rows where `age` is `NULL`. "Unknown is
> not 30" is unknown. If you want them, write `WHERE age <> 30 OR age IS NULL`.

## The project

### What you will build

`p22-minisql`: a command-line tool that runs one SQL query over one CSV (Comma-Separated Values) file and prints the result as
CSV. It is a small but honest query engine: a parser front end, a plan, typed columns with `NULL`,
filter, projection, hash aggregation, sort and limit, tested against Apache DataFusion.

```text
$ minisql people.csv "SELECT city, COUNT(*) AS n FROM people GROUP BY city ORDER BY city"
city,n
Lisbon,3
Oslo,3
Tartu,2
```

Use this file as `people.csv` in the project folder (Hana's age and Dana's score are empty, so `NULL`):

```text
name,age,city,score
Ana,34,Lisbon,7.5
Ben,27,Oslo,6.0
Chen,41,Lisbon,8.25
Dana,19,Oslo,
Eli,52,Tartu,9.0
Fay,30,Oslo,5.5
Gus,45,Tartu,7.0
Hana,,Lisbon,6.5
```

### Contract

**Command line** (use `clap`):

```text
minisql <FILE.csv> "<SQL>"      run one query, print the result as CSV to stdout
minisql --schema <FILE.csv>     print one line "<column>: <type>" per column, then "<N> rows"
```

- The table name in `FROM` must be the file name without `.csv` (`people` for `people.csv`). Otherwise:
  `error: unknown table `<name>``.
- Every error is printed to stderr as `error: <message>`, and the exit code is 1. Success exits with 0.

**Types and values:**

- Column types are inferred per column: `Int` (`i64`) if every non-empty field parses as `i64`; otherwise
  `Float` (`f64`) if every non-empty field parses as `f64`; otherwise `Text`. An empty field is `NULL`.
- `Int` and `Float` compare numerically with each other (`score > 7`); `Text` compares with `Text` only.
  Any other comparison is `error: type mismatch: …`.
- Output: one header line, then one line per row; values separated by `,`; `NULL` printed as an empty
  field; floats printed with Rust's `{:?}` format (`6.0`, `8.25`, `25.333333333333332`). Text with a
  comma or a quote is quoted CSV-style (use the `csv` crate's writer).

**SQL subset:**

| Part | Supported |
|---|---|
| `SELECT` | `*`, columns, `COUNT(*)`, `COUNT(col)`, `SUM`, `AVG`, `MIN`, `MAX` of a column, each with optional `AS alias` |
| `FROM` | exactly one table, no joins, no subqueries |
| `WHERE` | `= <> < <= > >=`, `AND`, `OR`, `NOT`, parentheses, `IS NULL`, `IS NOT NULL`, integer, float and `'text'` literals |
| `GROUP BY` | one or more columns |
| `ORDER BY` | one or more output columns, each `ASC` (default) or `DESC`; `NULL` is larger than every value, so it comes last with `ASC` and first with `DESC` |
| `LIMIT` | a non-negative integer |

Output column names: the alias if given; otherwise the column name, or `count`, `sum_<col>`,
`avg_<col>`, `min_<col>`, `max_<col>` (`count_<col>` for `COUNT(col)`). Aggregate semantics follow
Section 1. `AVG` is always `Float`; `SUM` keeps the column type. An aggregate query without `GROUP BY`
returns exactly one row. Anything else is `error: not supported: …`.

### Step-by-step plan

#### M1 — Command line, CSV and schema inference

Create the project, add `clap` (with the `derive` feature), `csv`, `sqlparser` and an error crate.
Write `read_table(path) -> Result<Table, …>` with the inference rules above and the `--schema` command.
Put the logic in `src/lib.rs` (and modules), keep `src/main.rs` small.

<details>
<summary>Hint</summary>

Read all records first (`Vec<csv::StringRecord>`), then decide each column's type in a second pass over
the strings, then convert. A column with only empty fields can be `Text`. Store values as an enum with a
`Null` variant: `enum Value { Null, Int(i64), Float(f64), Text(String) }`.

</details>

**Done when:**

```bash
cargo run -q -- --schema people.csv
```

```text
name: Text
age: Int
city: Text
score: Float
8 rows
```

Unit tests cover the three types, an empty field, and a column that mixes `1` and `1.5` (it must be
`Float`). Commit: `M1: CSV and schema inference`.

#### M2 — The plan

Extend the converter of Lesson 22.15 to the full subset: `*`, aliases, all aggregates, `NOT`, `IS NULL`,
float literals, `ORDER BY` and `LIMIT`. Resolve `*` into the table's columns here, and check the table name.

<details>
<summary>Hint</summary>

In `sqlparser` 0.64: an alias is `SelectItem::ExprWithAlias { expr, alias }`; `*` is
`SelectItem::Wildcard(_)`; `NOT x` is `Expr::UnaryOp { op: UnaryOperator::Not, expr }`; `x IS NULL` is
`Expr::IsNull(expr)`. `query.order_by` is `Option<OrderBy>`, and `OrderBy.kind` is
`OrderByKind::Expressions(Vec<OrderByExpr>)`; each `OrderByExpr` has `expr` and
`options.sort: Option<OrderBySort>` (`Asc` or `Desc`). A number with a `.` becomes a float literal.

</details>

**Done when:** `cargo test` passes with tests for at least ten accepted and five rejected queries, and:

```bash
cargo run -q -- people.csv "SELECT name FROM people JOIN pets ON people.name = pets.owner"
echo "exit code: $?"
```

```text
error: not supported: JOIN
exit code: 1
```

Commit: `M2: plan`.

#### M3 — Filter and projection with `NULL`

Write the evaluator with three-valued logic: comparisons return `Value::Null` when a side is `NULL`;
`AND`, `OR` and `NOT` follow the table in Section 1; `IS NULL` never returns `NULL`. The filter keeps only
rows whose condition is `true`. Projection preserves the input order of rows.

<details>
<summary>Hint</summary>

Represent a condition result as `Option<bool>` (`None` = unknown) inside the evaluator. Then `AND` is a
small `match` on `(Option<bool>, Option<bool>)`: `(Some(false), _) | (_, Some(false)) => Some(false)`,
`(Some(true), Some(true)) => Some(true)`, everything else `None`.

</details>

**Done when** these three commands print exactly:

```bash
cargo run -q -- people.csv "SELECT name, age FROM people WHERE age >= 30 AND city <> 'Oslo'"
cargo run -q -- people.csv "SELECT name FROM people WHERE age < 30"
cargo run -q -- people.csv "SELECT name, score FROM people WHERE score > 7"
```

```text
name,age
Ana,34
Chen,41
Eli,52
Gus,45
name
Ben
Dana
name,score
Ana,7.5
Chen,8.25
Eli,9.0
```

Note that Hana (age `NULL`) is in neither of the first two results, and that `score > 7` compares a
`Float` column with an `Int` literal. Commit: `M3: filter and projection`.

#### M4 — Aggregates, ORDER BY and LIMIT

Implement the five aggregates with `NULL` rules, `GROUP BY`, `ORDER BY` with the null ordering of the
contract, and `LIMIT`. Sorting happens after aggregation and projection, and `LIMIT` last.

<details>
<summary>Hint</summary>

Give every aggregate an accumulator struct, for example `Avg { sum: f64, count: i64 }`, with
`update(&Value)` and `finish() -> Value`. For sorting, write one `fn compare(a: &Value, b: &Value) -> Ordering`
that puts `Null` after everything, and use `sort_by` with a closure that walks the `ORDER BY` keys and
reverses the result for `DESC`. `sort_by` is stable, which keeps the output predictable.

</details>

**Done when:**

```bash
cargo run -q -- people.csv "SELECT city, COUNT(*) AS n, COUNT(age) AS with_age, AVG(age) AS avg_age, MAX(score) AS best FROM people GROUP BY city ORDER BY city"
cargo run -q -- people.csv "SELECT name, score FROM people ORDER BY score DESC LIMIT 3"
cargo run -q -- people.csv "SELECT COUNT(*) AS n, SUM(age) AS total, MIN(name) AS first FROM people WHERE city = 'Paris'"
```

```text
city,n,with_age,avg_age,best
Lisbon,3,2,37.5,8.25
Oslo,3,3,25.333333333333332,6.0
Tartu,2,2,48.5,9.0
name,score
Dana,
Eli,9.0
Chen,8.25
n,total,first
0,,
```

Dana comes first in the second result because her score is `NULL`, and `NULL` sorts as the largest
value. The third query matches no rows but still returns one row. Commit: `M4: aggregates, ORDER BY, LIMIT`.

#### M5 — Differential tests, a benchmark and a README

1. Add `datafusion = "55"` and `tokio` as **dev-dependencies** and write an integration test
   (`tests/against_datafusion.rs`) that runs at least 15 queries in both engines and compares the CSV text.
2. Generate a CSV file with 1,000,000 rows (write a small `src/bin/gen.rs` that prints random rows from a
   fixed seed) and measure one `GROUP BY` query with `hyperfine`.
3. Write `README.md`: the supported SQL, the `NULL` rules, how to run tests, and your benchmark numbers.

<details>
<summary>Hint</summary>

In the test, register the CSV file in a `SessionContext` with `SessionConfig::new().with_target_partitions(1)`
(one partition keeps the row order of unsorted results), `collect()` the batches, and write them with
`datafusion::arrow::csv::WriterBuilder::new().with_header(true).build(&mut buffer)`. Arrow's CSV writer
prints `NULL` as an empty field and floats like Rust's `{:?}`, which is why the contract uses those
rules. For queries without `ORDER BY`, sort the lines of both outputs before comparing.

</details>

**Done when:** `cargo test` runs the differential test and it passes; `cargo clippy --all-targets` and
`cargo fmt --check` are clean; the README shows a `hyperfine` result such as (example; your numbers
depend on your computer and code):

```text
Benchmark 1: ./target/release/minisql big.csv "SELECT city, COUNT(*), AVG(age) FROM big GROUP BY city"
  Time (mean ± σ):     412.3 ms ±   6.1 ms    [User: 371.8 ms, System: 38.9 ms]
  Range (min … max):   405.2 ms … 424.0 ms    10 runs
```

Commit: `M5: differential tests and benchmark`.

### Stretch goals

- **Columnar engine:** store each column as a typed `Vec` (or an Arrow array) and filter with masks, as in
  Lesson 22.16. Compare the benchmark before and after.
- **Binding:** resolve column names to positions once, before execution, and measure the difference.
- **More SQL:** `HAVING`, `COUNT(DISTINCT col)`, arithmetic (`age + 1`), `LIKE`.
- **A second table:** `minisql a.csv b.csv "… JOIN … ON …"` with a hash join (build a map from the
  smaller table, probe it with the bigger one).
- **Use DataFusion as a library:** write a `TableProvider` so that DataFusion can query a format of your
  own, for example your key-value store from Lesson 17.14.

## Common mistakes

> ⚠️ **Common mistake: comparing floats with `==` in tests.** `25.333333333333332` is the exact `f64`
> result of `76.0 / 3.0`; a slightly different order of additions can change the last digit. For `AVG`,
> sum in the same order as the reference, or compare with a small tolerance in your own unit tests.

> ⚠️ **Common mistake: sorting before aggregating.** `ORDER BY` refers to output columns (`n`, `avg_age`),
> which only exist after aggregation and projection. Sort the result table, not the input rows.

> ⚠️ **Common mistake: `f64` and `Ord`.** `f64` does not implement `Ord` (because of `NaN`, "Not a Number", which is not equal even to itself), so a derived
> `Ord` on a `Value` with `Float(f64)` does not compile. Write the comparison yourself with
> `f64::total_cmp`, or wrap floats in a type that implements `Ord`.

## Next steps in database engineering

**Learn more**

- *How Query Engines Work* by Andy Grove (the creator of DataFusion) — a short book that builds a query
  engine step by step; free to read online.
- *Architecture of a Database System* (Hellerstein, Stonebraker, Hamilton) — a classic free paper that
  explains how all the parts of a real database fit together.
- *Database Internals* by Alex Petrov — storage engines and distributed databases in depth.
- Carnegie Mellon University's (CMU) free courses: 15-445/645 (database systems) and 15-721 (advanced database
  systems: vectorized execution, query optimization, modern analytical engines). Lectures are online.

**Contribute** (Lesson 24.4 teaches the contribution workflow in detail)

- **Apache DataFusion** (`github.com/apache/datafusion`): a very welcoming project with many
  "good first issue" tasks, from new SQL functions to optimizer rules. Read its contributor guide first.
- **sqlparser-rs** (`github.com/apache/datafusion-sqlparser-rs`): add missing syntax of a dialect, with
  tests. Small, well-defined changes are a great first contribution.
- **arrow-rs** (`github.com/apache/arrow-rs`): compute kernels, file formats, performance.
- Other open-source databases written in Rust: Polars (DataFrames), GlueSQL, Turso (a rewrite of SQLite),
  Databend and RisingWave (cloud data warehouses and streaming).

**Communities**

- The Apache DataFusion community: its contributor guide lists the mailing list, the chat channels and the
  regular community calls.
- The CMU Database Group's public talks and seminars (linked from the course pages), where engineers from
  many database companies present their systems.

> 🧭 **End of the databases track.** Continue with [Lesson 22.18 — Networking track: high-performance servers](18-networking-track-high-performance-servers.md)
> if you want a second track, or go to the [checkpoint, Lesson 22.30](30-checkpoint-your-specialization.md).

## Quiz

1. Which rows does `SELECT name FROM people WHERE age < 30` return for the project's `people.csv`, and
   why is Hana not among them?

<details>
<summary>Answer</summary>

Ben (27) and Dana (19). Hana's age is `NULL`, so `NULL < 30` is unknown, and `WHERE` keeps only rows whose
condition is `true`.

</details>

2. What do `COUNT(*)` and `COUNT(age)` return for the Lisbon group, and why do they differ?

<details>
<summary>Answer</summary>

3 and 2. `COUNT(*)` counts rows (Ana, Chen, Hana); `COUNT(age)` counts non-`NULL` ages, and Hana's age is `NULL`.

</details>

3. Why does `ORDER BY score DESC LIMIT 3` return Dana first?

<details>
<summary>Answer</summary>

By the contract (the same rule as PostgreSQL and DataFusion), `NULL` is larger than every value. In
descending order the largest comes first, and Dana's score is `NULL`.

</details>

4. Why is DataFusion a good test oracle for this project, and what must you do for queries without `ORDER BY`?

<details>
<summary>Answer</summary>

It is a mature, well-tested engine that follows standard SQL rules, so a difference usually means a bug in
your engine (or a deliberate difference in the contract). Without `ORDER BY` the row order is not defined,
so sort both outputs before comparing (or use one partition and compare order-preserving queries only).

</details>

5. Why does sorting happen after aggregation?

<details>
<summary>Answer</summary>

`ORDER BY` names output columns, and aggregated columns such as `n` or `avg_age` exist only after
aggregation. Sorting the input rows would sort the wrong thing.

</details>

## Summary

- `p22-minisql` combines the parser front end, a typed plan and the executor into a real tool.
- Per-column type inference, `NULL` as empty fields, and numeric coercion between `Int` and `Float` make
  the engine handle real CSV data.
- `NULL` follows three-valued logic: comparisons give unknown, `WHERE` keeps only `true`, aggregates skip
  `NULL`, and `NULL` has a defined place in the sort order.
- Sorting and `LIMIT` run last, on the output table.
- Differential tests against DataFusion and a `hyperfine` benchmark make the engine trustworthy and measurable.
- Next steps: How Query Engines Work, the CMU courses, and contributions to DataFusion, sqlparser-rs and arrow-rs.

## Sources and further reading

- [How Query Engines Work (Andy Grove)](https://howqueryengineswork.com/) — a free online book about building a query engine.
- [Architecture of a Database System](https://dsf.berkeley.edu/papers/fntdb07-architecture.pdf) — the classic overview paper (a PDF, Portable Document Format, file).
- [CMU 15-721 Advanced Database Systems](https://15721.courses.cs.cmu.edu/) — modern analytical engines, vectorization and optimizers.
- [Apache DataFusion contributor guide](https://datafusion.apache.org/contributor-guide/index.html) — how to start contributing, and where the community talks.
- [PostgreSQL — comparison functions and operators](https://www.postgresql.org/docs/current/functions-comparison.html) — how a real database treats `NULL` in comparisons and `IS NULL`.
- [PostgreSQL — sorting rows](https://www.postgresql.org/docs/current/queries-order.html) — `ORDER BY`, `ASC`/`DESC` and `NULLS FIRST`/`LAST`.

---

[🏠 Start](../START-HERE.md) · [← 22.16 Databases track: executing queries](16-databases-track-executing-queries.md) · [📋 Phase 22](README.md) · [22.18 Networking track: high-performance servers →](18-networking-track-high-performance-servers.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
