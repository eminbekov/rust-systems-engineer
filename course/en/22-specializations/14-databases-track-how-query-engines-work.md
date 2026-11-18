<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 22.13 Embedded track: project and next steps](13-embedded-track-project-and-next-steps.md) · [📋 Phase 22](README.md) · [22.15 Databases track: parsing SQL →](15-databases-track-parsing-sql.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/22-specializations/14-databases-track-how-query-engines-work.md) <!-- nav -->

# Lesson 22.14 — Databases track: how query engines work

> **Phase 22 — Specializations** · Lesson 14 of 30 · ⏱ about 120 minutes

> 🧭 **Not your track?** This lesson starts the databases track (Lessons 22.14–22.17). If you chose
> another track, skip to [Lesson 22.18 — Networking track: high-performance servers](18-networking-track-high-performance-servers.md)
> or go straight to the [checkpoint, Lesson 22.30](30-checkpoint-your-specialization.md).

## What you will learn

- What a query engine is, and the stages from SQL (Structured Query Language) text to a result: parse, plan, optimize, execute.
- What logical and physical plans are, and how relational operators form a tree.
- The iterator ("Volcano") execution model, and why Rust iterators already work this way.
- Why analytical engines store data in columns, and what Apache Arrow is.
- How to run SQL over a CSV (Comma-Separated Values) file with Apache DataFusion and read its plans with `EXPLAIN`.

## Before you start

- You finished [Lesson 22.1 — Choosing a specialization](01-choosing-a-specialization.md) and picked this track.
- You know SQL from [Lesson 16.5 — SQL and PostgreSQL basics](../16-backend-services/05-sql-and-postgresql-basics.md)
  and the big picture of [Lesson 17.13 — How a SQL query runs](../17-storage-and-databases/13-how-a-sql-query-runs.md).
  This track builds the "query" half of a database yourself; Phase 17 covered the storage half.
- You know iterators ([Lesson 05.5](../05-rust-intermediate/05-iterators.md)), trees
  ([Lesson 07.11](../07-algorithms-and-data-structures/11-trees-and-binary-search-trees.md)), cache-friendly
  code ([Lesson 12.10](../12-computer-systems/10-cache-friendly-code.md)) and Tokio basics
  ([Lesson 14.13](../14-concurrency-and-async/13-tokio-tasks-and-channels.md)).

**Where the code goes.** Each lesson of this track uses its own package in `code/p22-<name>`. The
track project in Lesson 22.17 is `code/p22-minisql`.

## New words

| Word | Meaning |
|---|---|
| **query engine** | The part of a database (or a library) that takes a query, plans it and computes the result. |
| **SQL** (Structured Query Language) | The standard language for asking questions about tables. |
| **CSV** (Comma-Separated Values) | A text file where each line is a row and commas separate the columns. |
| **relational operator** | One basic step on tables: scan, filter, project, aggregate, sort, join, limit. |
| **projection** | Keeping only some columns (the `SELECT name, city` part). |
| **aggregation** | Turning many rows into one value per group: `COUNT`, `SUM`, `AVG` with `GROUP BY`. |
| **logical plan** | A tree of relational operators that says *what* to compute. |
| **physical plan** | A tree of concrete algorithms that says *how* to compute it. |
| **optimizer** | The stage that rewrites a plan into a cheaper plan with the same result. |
| **Volcano model** | Execution where every operator has a `next()` method that pulls one row from its child. |
| **vectorized execution** | Execution where operators process a batch of thousands of values at once. |
| **columnar format** | Storing each column's values next to each other in memory, instead of each row's. |
| **Apache Arrow** | A standard columnar in-memory format, with libraries in many languages (the Rust crate is `arrow`). |
| **Apache DataFusion** | A query engine written in Rust, built on Arrow. Other systems embed it. |
| **record batch** | Arrow's unit of data: a schema plus one array per column, all with the same number of rows. |

## 1. What a query engine does

A database has two big halves. The **storage engine** keeps bytes safe on disk: pages, B-trees, the
write-ahead log (Phase 17). The **query engine** answers questions about the data. You give it SQL text:

```sql
SELECT city, COUNT(*) AS n, AVG(age) AS avg_age
FROM people
WHERE age >= 30
GROUP BY city
ORDER BY city
```

and it returns a table. Inside, the query passes through stages:

```text
 SQL text
    |  1. parse      (Lesson 22.15)
    v
 syntax tree (AST)
    |  2. plan       resolve tables and columns, build operators
    v
 logical plan        WHAT to compute
    |  3. optimize   rewrite to a cheaper equivalent plan
    v
 optimized logical plan
    |  4. physical planning   pick algorithms
    v
 physical plan       HOW to compute
    |  5. execute    (Lesson 22.16)
    v
 result rows
```

AST means Abstract Syntax Tree: the structure of the text, as a tree of Rust values. Many modern
systems (analytics databases, stream processors, data tools) are built from exactly these pieces,
and in Rust the pieces exist as crates: `sqlparser` for step 1 and DataFusion for steps 2–5.

## 2. Plans are trees of operators

Every SQL query can be written with a few **relational operators**. Each takes one or two tables
(streams of rows) and produces a new table:

| Operator | SQL part | What it does |
|---|---|---|
| Scan | `FROM people` | reads all rows of a table |
| Filter | `WHERE age >= 30` | keeps rows for which a condition is true |
| Projection | `SELECT city, …` | keeps or computes some columns |
| Aggregate | `GROUP BY city` + `COUNT(*)`, `AVG(age)` | one output row per group |
| Sort | `ORDER BY city` | orders the rows |
| Limit | `LIMIT 10` | stops after N rows |
| Join | `JOIN … ON …` | combines rows of two tables |

The query from Section 1 becomes a **tree**. Data flows from the leaf (the scan) up to the root:

```text
            Sort (city)
                 |
      Projection (city, n, avg_age)
                 |
   Aggregate (group by city: COUNT(*), AVG(age))
                 |
          Filter (age >= 30)
                 |
            Scan (people)
```

This is the **logical plan**: it says *what* the result is, not *how* to compute it. The **physical
plan** chooses algorithms: for example, aggregate with a hash table or by sorting first; read the CSV
file with one thread or many.

Between the two, the **optimizer** rewrites the logical plan. Two classic rules:

- **Projection pushdown:** the query uses only `age` and `city`, so the scan reads only those two
  columns.
- **Predicate pushdown** (a predicate is a condition): move filters as close to the scan as possible,
  so fewer rows travel up the tree. Some file formats can even skip whole blocks of data that cannot match.

## 3. The iterator ("Volcano") model, in plain Rust

How do the operators pass rows to each other? The classic answer (from the Volcano research system of
the early 1990s) is: every operator has a `next()` method. The root calls `next()` on its child, which
calls `next()` on its child, down to the scan. Each call **pulls** one row up through the tree.

You already know this model: it is exactly how Rust iterators work. Create `code/p22-pipeline` with
`cargo new code/p22-pipeline` and put this into `src/main.rs`:

```rust
struct Person {
    name: &'static str,
    age: u32,
    city: &'static str,
}

fn person(name: &'static str, age: u32, city: &'static str) -> Person {
    Person { name, age, city }
}

fn main() {
    let people = [
        person("Ana", 34, "Lisbon"),
        person("Ben", 27, "Oslo"),
        person("Chen", 41, "Lisbon"),
        person("Dana", 19, "Oslo"),
    ];

    // SELECT name, city FROM people WHERE age >= 30
    let result: Vec<(&str, &str)> = people
        .iter()
        .inspect(|p| println!("scan    {}", p.name))
        .filter(|p| p.age >= 30)
        .inspect(|p| println!("  keep  {}", p.name))
        .map(|p| (p.name, p.city))
        .collect();

    println!("result: {result:?}");
}
```

**What happens:**

1. `people.iter()` is the **scan**, `.filter(...)` the **filter**, `.map(...)` the **projection**.
2. `.collect()` is the root: it calls `next()` again and again. Each call travels down the chain to the
   scan and brings one row back up.
3. `inspect` only prints what passes by, so we can watch the order of work.

```text
scan    Ana
  keep  Ana
scan    Ben
scan    Chen
  keep  Chen
scan    Dana
result: [("Ana", "Lisbon"), ("Chen", "Lisbon")]
```

The rows are not processed "all scans first, then all filters". Each row goes up the whole pipeline
before the next one is read. That is **pipelining**: no operator needs to keep the whole table in
memory. Some operators cannot pipeline: a sort or an aggregate must see *all* input rows before it can
produce its first output row. They are called **pipeline breakers**.

## 4. Rows or columns?

The Volcano model moves one row per `next()` call. For analytical queries over millions of rows, that
is slow: one function call (often a dynamic call through a trait object) per row per operator, and the
CPU (Central Processing Unit) jumps between different kinds of work all the time.

Modern analytical engines make two changes:

1. **Columnar layout.** Store each column's values next to each other:

```text
Row layout (array of structs)          Columnar layout (struct of arrays)

[Ana | 34 | Lisbon]                    names:  [Ana, Ben, Chen, Dana]
[Ben | 27 | Oslo  ]                    ages:   [ 34,  27,   41,   19]
[Chen| 41 | Lisbon]                    cities: [Lisbon, Oslo, Lisbon, Oslo]
[Dana| 19 | Oslo  ]
```

   `AVG(age)` now reads one compact array of integers. The CPU cache is filled only with useful data
   ([Lesson 12.10](../12-computer-systems/10-cache-friendly-code.md)), and the compiler can use SIMD
   (Single Instruction, Multiple Data: one instruction that works on several numbers at once).

2. **Vectorized execution.** Operators pass **batches** (for example 8,192 rows) instead of single rows.
   A filter computes a whole array of true/false values in one tight loop, then keeps the matching rows.
   The per-row overhead almost disappears.

**Apache Arrow** is the standard for this columnar data in memory. An Arrow **record batch** is a
schema (column names and types) plus one array per column. Each array is a buffer of values plus a
**validity bitmap**: one bit per row that says whether the value is present or `NULL`. Because the
format is the same in every language and tool, systems can hand data to each other without converting it.

> ⚡ **Performance:** row layout is better when you read or write whole rows one at a time (an online
> shop updating one order). Column layout is better when you scan a few columns of many rows (a report
> over a year of orders). This is the difference between OLTP (Online Transaction Processing) and OLAP
> (Online Analytical Processing) databases.

## 5. A real engine: Apache DataFusion

**DataFusion** is a complete query engine in Rust, built on Arrow. It has a SQL front end (it uses the
`sqlparser` crate of Lesson 22.15), a logical planner, an optimizer, a parallel vectorized executor, and
readers for CSV, JSON (JavaScript Object Notation), Parquet (a columnar file format) and more.
Databases, stream processors and data tools embed it instead of writing their own engine.

Create the package and a small CSV file:

```bash
cd ~/projects/rust
cargo new code/p22-datafusion
cd code/p22-datafusion
cargo add datafusion@55
cargo add tokio@1 --features rt-multi-thread,macros
```

`people.csv` (in the package folder, next to `Cargo.toml`):

```text
name,age,city
Ana,34,Lisbon
Ben,27,Oslo
Chen,41,Lisbon
Dana,19,Oslo
Eli,52,Tartu
Fay,30,Oslo
Gus,45,Tartu
Hana,23,Lisbon
```

`src/main.rs`:

```rust
use datafusion::prelude::*;

#[tokio::main]
async fn main() -> datafusion::error::Result<()> {
    let ctx = SessionContext::new();
    ctx.register_csv("people", "people.csv", CsvReadOptions::new())
        .await?;

    let sql = "SELECT city, COUNT(*) AS n, AVG(age) AS avg_age \
               FROM people WHERE age >= 30 \
               GROUP BY city ORDER BY city";
    let df = ctx.sql(sql).await?;
    df.show().await?;
    Ok(())
}
```

**What happens:**

1. `SessionContext` holds the catalog (which tables exist) and the settings.
2. `register_csv` registers the file as the table `people`. DataFusion reads the header and a sample
   of rows to infer the schema: `name` is text, `age` is a 64-bit integer, `city` is text.
3. `ctx.sql(...)` parses, plans and optimizes the query and returns a `DataFrame`: a plan that has not
   run yet. `show()` executes it and prints the result as a table.
4. DataFusion is async because reading files (or object storage over the network) is I/O
   (input/output); `#[tokio::main]`
   provides the runtime.

```bash
cargo run
```

```text
+--------+---+---------+
| city   | n | avg_age |
+--------+---+---------+
| Lisbon | 2 | 37.5    |
| Oslo   | 1 | 30.0    |
| Tartu  | 2 | 48.5    |
+--------+---+---------+
```

Check it by hand: people aged 30 or more are Ana 34 and Chen 41 (Lisbon, average 37.5), Fay 30 (Oslo),
Eli 52 and Gus 45 (Tartu, average 48.5).

> 💡 **Tip:** DataFusion is a big crate. The first build compiled 245 crates and took 3 min 39 s with two
> build jobs, and the `target` folder grew to several gigabytes. On macOS a debug build may also print a
> harmless linker warning `__eh_frame section too large`. Later builds are fast.

### Reading the plans with `EXPLAIN`

Put `EXPLAIN` in front of the query to see the plans instead of the result. Save this as
`examples/explain.rs` and run `cargo run --example explain`:

```rust
use datafusion::prelude::*;

#[tokio::main]
async fn main() -> datafusion::error::Result<()> {
    // One partition: a simpler plan that is the same on every computer.
    let config = SessionConfig::new().with_target_partitions(1);
    let ctx = SessionContext::new_with_config(config);
    ctx.register_csv("people", "people.csv", CsvReadOptions::new())
        .await?;

    let sql = "EXPLAIN SELECT city, COUNT(*) AS n, AVG(age) AS avg_age \
               FROM people WHERE age >= 30 \
               GROUP BY city ORDER BY city";
    ctx.sql(sql).await?.show().await?;
    Ok(())
}
```

The output is a table with two rows; here it is with the table borders removed so that it fits:

```text
logical_plan
Sort: people.city ASC NULLS LAST
  Projection: people.city, count(Int64(1)) AS count(*) AS n, avg(people.age) AS avg_age
    Aggregate: groupBy=[[people.city]], aggr=[[count(Int64(1)), avg(CAST(people.age AS Float64))]]
      Filter: people.age >= Int64(30)
        TableScan: people projection=[age, city], partial_filters=[people.age >= Int64(30)]

physical_plan
ProjectionExec: expr=[city@0 as city, count(Int64(1))@1 as n, avg(people.age)@2 as avg_age]
  SortExec: expr=[city@0 ASC NULLS LAST], preserve_partitioning=[false]
    AggregateExec: mode=Single, gby=[city@1 as city], aggr=[count(Int64(1)), avg(people.age)]
      FilterExec: age@0 >= 30
        DataSourceExec: file_groups={1 group: [[.../p22-datafusion/people.csv]]}, projection=[age, city], file_type=csv, has_header=true
```

**What happens:**

1. The **logical plan** is the tree from Section 2, printed root first; indentation shows the children.
2. `TableScan: people projection=[age, city]` is **projection pushdown**: the optimizer saw that `name`
   is never used. `partial_filters=[...]` shows that the filter was also offered to the scan
   (**predicate pushdown**); for CSV the scan cannot use it, so the `Filter` stays.
3. `COUNT(*)` became `count(Int64(1))` (count a constant 1 for every row) and `AVG(age)` got a cast to
   `Float64`: the planner chose exact types.
4. In the **physical plan**, every node is an `...Exec` algorithm. `city@0` means "column number 0 of
   the input batch": names are resolved to positions before execution. `AggregateExec` uses a hash table.
5. With the default settings DataFusion splits the work into one **partition** per CPU core and adds
   `RepartitionExec` nodes to move rows between threads. Try it: remove the `config` lines and run again.

> 🧪 **Testing:** a mature engine like DataFusion is a great **test oracle**: run the same query in
> your own engine and in DataFusion and compare the results. This is *differential testing*, and the
> track project uses it.

## Common mistakes

> ⚠️ **Common mistake: forgetting that `DataFrame` is lazy.** `let df = ctx.sql(sql).await?;` only
> plans the query. Nothing is read until you call `show()`, `collect()` or a similar method. If you time
> only `ctx.sql`, you measure planning, not execution.

> ⚠️ **Common mistake: a relative path from the wrong folder.** `register_csv("people", "people.csv", …)`
> looks for the file relative to the **current directory** of the process. Run `cargo run` in the
> package folder, or use a full path. Run from `src/`, the program stops with:
>
> ```text
> Error: Plan("No files found at file:///Users/you/projects/rust/code/p22-datafusion/src/people.csv. Cannot infer schema from an empty location; either add data files or declare an explicit schema for the table.")
> ```

> ⚠️ **Common mistake: mixing `arrow` versions.** DataFusion 55 uses its own version of the `arrow`
> crate (59.x at the time of writing), and the newest `arrow` on crates.io may be newer. If you add
> `arrow` separately with another version, the types do not match (`expected RecordBatch, found
> RecordBatch`). Use the re-export `datafusion::arrow` instead.

## Try it yourself

1. Change the filter to `age >= 40` and predict the result before you run it.
2. Add `LIMIT 2` at the end of the query in `examples/explain.rs`. There is no new node: the optimizer
   merges the limit into the sort (`Sort: … fetch=2`), and the physical plan uses
   `SortExec: TopK(fetch=2)`, which keeps only the best 2 rows instead of sorting everything.
3. Run `examples/explain.rs` without the `config` lines and count the `RepartitionExec` nodes.

## Exercises

### Exercise 1 — Draw the plan (easy)

Draw the logical plan tree for
`SELECT name FROM people WHERE city = 'Oslo' ORDER BY name LIMIT 1`. Which operator is the root, and
which operators are pipeline breakers?

<details>
<summary>Hint</summary>

Start at the leaf: what must happen first? Then read the SQL clauses in the order the engine needs them:
`FROM`, `WHERE`, `SELECT`, `ORDER BY`, `LIMIT`.

</details>

<details>
<summary>Answer</summary>

```text
Limit (1)
   |
Sort (name)
   |
Projection (name)
   |
Filter (city = 'Oslo')
   |
Scan (people)
```

The root is `Limit`. `Sort` is a pipeline breaker: it must read all rows before it knows the first one.
DataFusion's optimizer goes one step further and merges the two: its logical plan shows
`Sort: people.name ASC NULLS LAST, fetch=1`, and the physical plan uses `SortExec: TopK(fetch=1)`, a
"top-N" operator that keeps only the best row while it reads.

</details>

### Exercise 2 — Watch pushdown (medium)

Use `EXPLAIN` in `code/p22-datafusion` for `SELECT name FROM people WHERE age > 40`. Which columns does
the `TableScan` read, and why is it not only `name`?

<details>
<summary>Hint</summary>

The scan must deliver every column that any operator above it needs, not only the columns in `SELECT`.

</details>

<details>
<summary>Answer</summary>

The scan reads `projection=[name, age]`: `name` for the output and `age` for the filter. `city` is
never read into memory. Projection pushdown keeps the columns that *any* operator uses, and drops the rest.

</details>

### Exercise 3 — Columns and a test (hard)

Create a library `code/p22-columns` with a `Person` struct (as in Section 3, with `pub` fields), a
`PeopleColumns` struct with three `Vec` fields, `pub fn to_columns(rows: &[Person]) -> PeopleColumns`,
and `pub fn average_age(cols: &PeopleColumns) -> Option<f64>` that reads only the `ages` column and
returns `None` for an empty table. **Write a test** for two rows and for an empty table.

<details>
<summary>Hint</summary>

Derive `Default` for `PeopleColumns` to start with three empty vectors. Sum the ages as `u64` to avoid
overflow, then divide as `f64`.

</details>

<details>
<summary>Answer</summary>

```rust
pub struct Person {
    pub name: &'static str,
    pub age: u32,
    pub city: &'static str,
}

/// The same table, stored column by column.
#[derive(Debug, Default, PartialEq)]
pub struct PeopleColumns {
    pub names: Vec<&'static str>,
    pub ages: Vec<u32>,
    pub cities: Vec<&'static str>,
}

pub fn to_columns(rows: &[Person]) -> PeopleColumns {
    let mut cols = PeopleColumns::default();
    for p in rows {
        cols.names.push(p.name);
        cols.ages.push(p.age);
        cols.cities.push(p.city);
    }
    cols
}

/// AVG(age): reads only the `ages` column. `None` for an empty table.
pub fn average_age(cols: &PeopleColumns) -> Option<f64> {
    if cols.ages.is_empty() {
        return None;
    }
    let sum: u64 = cols.ages.iter().map(|&a| a as u64).sum();
    Some(sum as f64 / cols.ages.len() as f64)
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn columns_and_average() {
        let rows = [
            Person { name: "Ana", age: 34, city: "Lisbon" },
            Person { name: "Ben", age: 27, city: "Oslo" },
        ];
        let cols = to_columns(&rows);
        assert_eq!(cols.ages, [34, 27]);
        assert_eq!(cols.cities, ["Lisbon", "Oslo"]);
        assert_eq!(average_age(&cols), Some(30.5));
        assert_eq!(average_age(&PeopleColumns::default()), None);
    }
}
```

Why it works: `average_age` touches only one `Vec<u32>`, a single compact block of memory, which is the
whole idea of columnar storage. The empty case returns `None`, just like `AVG` over no rows returns
`NULL` in SQL.

</details>

## Quiz

1. Name the stages a query passes through inside a query engine.

<details>
<summary>Answer</summary>

Parse (text → syntax tree), plan (→ logical plan), optimize (→ cheaper logical plan), physical planning
(→ physical plan with concrete algorithms), execute (→ result).

</details>

2. What is the difference between a logical and a physical plan?

<details>
<summary>Answer</summary>

The logical plan says *what* to compute with relational operators (filter, aggregate, …). The physical
plan says *how*: which algorithm, which column positions, how many partitions and threads.

</details>

3. Why is a Rust iterator chain an example of the Volcano model?

<details>
<summary>Answer</summary>

Each adapter (`filter`, `map`) has a `next()` that pulls one item from the adapter below it. The
consumer at the end (`collect`) drives everything by calling `next()` repeatedly.

</details>

4. What is a pipeline breaker? Give two examples.

<details>
<summary>Answer</summary>

An operator that must read all of its input before it can produce output, so rows cannot flow through
it one by one. Examples: sort, aggregate (and the build side of a hash join).

</details>

5. Why do analytical engines prefer a columnar format?

<details>
<summary>Answer</summary>

Queries usually read a few columns of many rows. In columnar layout those values are packed together,
so the engine reads less memory, uses the CPU cache well, and can process batches with tight loops and SIMD.

</details>

6. What do projection pushdown and predicate pushdown do?

<details>
<summary>Answer</summary>

Projection pushdown makes the scan read only the columns that are used. Predicate pushdown moves
filters down to (or into) the scan, so fewer rows travel through the plan.

</details>

## Summary

- A query engine turns SQL into a result in stages: parse, logical plan, optimize, physical plan, execute.
- Plans are trees of relational operators: scan, filter, projection, aggregate, sort, limit, join.
- The Volcano model pulls one row at a time with `next()`; Rust iterators work exactly like this.
- Sort and aggregate are pipeline breakers; filter and projection pipeline.
- Analytical engines use columnar data and vectorized batches; Apache Arrow is the standard in-memory format.
- Apache DataFusion is a full query engine in Rust; `EXPLAIN` shows its logical and physical plans,
  including projection and predicate pushdown.

## Sources and further reading

- [Apache DataFusion documentation](https://datafusion.apache.org/) — the user guide and the architecture of the engine.
- [DataFusion: `EXPLAIN` and reading plans](https://datafusion.apache.org/user-guide/explain-usage.html) — how to read logical and physical plans.
- [Apache Arrow columnar format](https://arrow.apache.org/docs/format/Columnar.html) — the official description of arrays, buffers and validity bitmaps.
- [`datafusion` on docs.rs](https://docs.rs/datafusion/55.3.0/datafusion/) — `SessionContext`, `DataFrame` and the rest of the API (Application Programming Interface).
- [CMU 15-445/645 Database Systems](https://15445.courses.cs.cmu.edu/) — a free Carnegie Mellon University (CMU) course with lectures on query execution and optimization.
- [Volcano — An Extensible and Parallel Query Evaluation System (Graefe, 1994)](https://doi.org/10.1109/69.273032) — the paper behind the iterator model.

---

[🏠 Start](../START-HERE.md) · [← 22.13 Embedded track: project and next steps](13-embedded-track-project-and-next-steps.md) · [📋 Phase 22](README.md) · [22.15 Databases track: parsing SQL →](15-databases-track-parsing-sql.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
