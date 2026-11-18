<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 22.15 Databases track: parsing SQL](15-databases-track-parsing-sql.md) · [📋 Phase 22](README.md) · [22.17 Databases track: project and next steps →](17-databases-track-project-and-next-steps.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/22-specializations/16-databases-track-executing-queries.md) <!-- nav -->

# Lesson 22.16 — Databases track: executing queries

> **Phase 22 — Specializations** · Lesson 16 of 30 · ⏱ about 2.5 hours (2 sittings)

## What you will learn

- How to load a CSV (Comma-Separated Values) file into a table of typed values, so that SQL (Structured
  Query Language) queries can run over it.
- How to evaluate expressions such as `age >= 30 AND city = 'Lisbon'` for one row.
- How to implement the operators scan, filter, projection and hash aggregation with `GROUP BY`.
- How the same filter looks in a vectorized engine, using Apache Arrow compute kernels.
- How to test an executor and compare it with a reference engine.

## Before you start

- You finished [Lesson 22.15 — Databases track: parsing SQL](15-databases-track-parsing-sql.md) and have
  `code/p22-query` with `src/plan.rs`. This lesson adds `src/exec.rs` to the same package.
- You know hash maps ([Lesson 04.9](../04-structs-enums-collections-errors/09-hash-maps.md)) and how they
  work inside ([Lesson 07.6](../07-algorithms-and-data-structures/06-hash-tables.md)), and trait objects
  for errors (`Box<dyn Error>`, [Lesson 06.1](../06-rust-advanced/01-trait-objects-and-dynamic-dispatch.md)).

## New words

| Word | Meaning |
|---|---|
| **executor** | The part of a query engine that runs a plan and produces result rows. |
| **evaluation** | Computing the value of an expression for one row, for example `age >= 30` → `true`. |
| **hash aggregation** | Grouping rows by putting each one into a map entry for its group key and updating running totals there. |
| **group key** | The values of the `GROUP BY` columns for one row, for example `["Lisbon"]`. |
| **accumulator** | The running state of one aggregate for one group: a count, a sum. |
| **compute kernel** | An Arrow function that works on whole arrays at once: compare, filter, sum. |
| **mask** | An array of `true`/`false`, one per row, that says which rows to keep. |

## 1. Values, tables and reading CSV

Our executor works on a simple in-memory table: a list of column names and a list of rows, where each
row is a `Vec<Value>`. Add the CSV crate to the package:

```bash
cd ~/projects/rust/code/p22-query
cargo add csv@1
cp ../p22-datafusion/people.csv .
```

Create `src/exec.rs` and add `pub mod exec;` to `src/lib.rs`. The first part of the file:

```rust
//! Executes a `Query` over a table held in memory.

use std::collections::BTreeMap;
use std::fmt;

use crate::plan::{BinOp, Expr, Literal, Query, SelectItem};

#[derive(Debug, Clone, PartialEq, Eq, PartialOrd, Ord, Hash)]
pub enum Value {
    Int(i64),
    Text(String),
    Bool(bool),
}

impl fmt::Display for Value {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            Value::Int(i) => write!(f, "{i}"),
            Value::Text(s) => write!(f, "{s}"),
            Value::Bool(b) => write!(f, "{b}"),
        }
    }
}

/// A table in memory: column names and rows of values.
#[derive(Debug, Clone, PartialEq)]
pub struct Table {
    pub columns: Vec<String>,
    pub rows: Vec<Vec<Value>>,
}

#[derive(Debug, PartialEq)]
pub enum ExecError {
    UnknownColumn(String),
    TypeMismatch(String),
    NotGrouped(String),
}

impl fmt::Display for ExecError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            ExecError::UnknownColumn(c) => write!(f, "unknown column `{c}`"),
            ExecError::TypeMismatch(m) => write!(f, "type mismatch: {m}"),
            ExecError::NotGrouped(c) => write!(f, "column `{c}` must appear in GROUP BY"),
        }
    }
}

impl std::error::Error for ExecError {}

/// Reads a CSV file with a header row. Fields that look like whole numbers become `Int`.
pub fn read_csv(path: &str) -> Result<Table, Box<dyn std::error::Error>> {
    let mut reader = csv::Reader::from_path(path)?;
    let columns = reader.headers()?.iter().map(String::from).collect();
    let mut rows = Vec::new();
    for record in reader.records() {
        let row = record?
            .iter()
            .map(|field| match field.parse::<i64>() {
                Ok(i) => Value::Int(i),
                Err(_) => Value::Text(field.to_string()),
            })
            .collect();
        rows.push(row);
    }
    Ok(Table { columns, rows })
}
```

**What happens:**

1. `Value` is the type of one cell. `Bool` is not stored in tables; it is the result of comparisons
   such as `age >= 30`. Deriving `Ord` and `Hash` lets values be keys of a map later.
2. `ExecError` lists what can go wrong *while running* a valid plan: a column that the table does not have,
   comparing text with a number, and a plain column next to an aggregate without `GROUP BY`.
3. `csv::Reader::from_path` opens the file and treats the first line as the header. The `csv` crate
   handles quoted fields such as `"Lisbon, Portugal"` correctly, which a simple `split(',')` does not.
4. Each field becomes `Int` if it parses as `i64`, otherwise `Text`. This guesses the type **per value**;
   a real engine infers one type **per column** (the project in Lesson 22.17 does).

## 2. Evaluating expressions

Next, the expression evaluator. It walks the `Expr` tree recursively, like the converter in Lesson 22.15:

```rust
fn column_index(columns: &[String], name: &str) -> Result<usize, ExecError> {
    columns
        .iter()
        .position(|c| c == name)
        .ok_or_else(|| ExecError::UnknownColumn(name.to_string()))
}

/// Evaluates an expression for one row.
pub fn eval(expr: &Expr, columns: &[String], row: &[Value]) -> Result<Value, ExecError> {
    match expr {
        Expr::Column(name) => Ok(row[column_index(columns, name)?].clone()),
        Expr::Literal(Literal::Int(i)) => Ok(Value::Int(*i)),
        Expr::Literal(Literal::Text(s)) => Ok(Value::Text(s.clone())),
        Expr::Binary { left, op, right } => {
            let l = eval(left, columns, row)?;
            let r = eval(right, columns, row)?;
            let result = match (op, &l, &r) {
                (BinOp::And, Value::Bool(a), Value::Bool(b)) => *a && *b,
                (BinOp::Or, Value::Bool(a), Value::Bool(b)) => *a || *b,
                (BinOp::And | BinOp::Or, _, _) => {
                    return Err(ExecError::TypeMismatch(format!("{l} {op:?} {r}")));
                }
                (_, Value::Int(_), Value::Int(_)) | (_, Value::Text(_), Value::Text(_)) => match op
                {
                    BinOp::Eq => l == r,
                    BinOp::NotEq => l != r,
                    BinOp::Lt => l < r,
                    BinOp::LtEq => l <= r,
                    BinOp::Gt => l > r,
                    BinOp::GtEq => l >= r,
                    BinOp::And | BinOp::Or => unreachable!(),
                },
                _ => return Err(ExecError::TypeMismatch(format!("{l} {op:?} {r}"))),
            };
            Ok(Value::Bool(result))
        }
    }
}
```

**What happens:**

1. A column is looked up by name and its value is copied out of the row.
2. For `Binary`, both sides are evaluated first, then the operator is applied. The `match` on the tuple
   `(op, &l, &r)` handles each legal combination: `AND`/`OR` need two booleans; comparisons need two
   values of the **same** type.
3. Two `Int`s compare as numbers and two `Text`s compare alphabetically, through the derived `PartialOrd`.
   `'Oslo' > 3` falls to the last arm and becomes a `TypeMismatch` error, not a wrong answer.
4. `unreachable!()` marks a case that the earlier arms already excluded; it panics if our reasoning is
   ever wrong.

> ⚡ **Performance:** `column_index` searches the column names for every row and every column reference.
> Real engines **bind** the plan once before execution: they replace names with positions (DataFusion's
> `age@0` in Lesson 22.14). For a few thousand rows this does not matter; for millions it does.

## 3. The operators: scan, filter, projection, aggregation

Now the operators. `execute` runs the plan from the bottom up:

```rust
/// Runs scan -> filter -> (aggregate | projection).
pub fn execute(query: &Query, table: &Table) -> Result<Table, ExecError> {
    // Scan + filter.
    let mut rows = Vec::new();
    for row in &table.rows {
        let keep = match &query.filter {
            None => true,
            Some(f) => eval(f, &table.columns, row)? == Value::Bool(true),
        };
        if keep {
            rows.push(row);
        }
    }

    let has_aggregates = query
        .items
        .iter()
        .any(|i| !matches!(i, SelectItem::Column(_)));
    if has_aggregates || !query.group_by.is_empty() {
        aggregate(query, &table.columns, &rows)
    } else {
        project(query, &table.columns, &rows)
    }
}

fn project(query: &Query, columns: &[String], rows: &[&Vec<Value>]) -> Result<Table, ExecError> {
    let mut indexes = Vec::new();
    let mut names = Vec::new();
    for item in &query.items {
        if let SelectItem::Column(name) = item {
            indexes.push(column_index(columns, name)?);
            names.push(name.clone());
        }
    }
    let rows = rows
        .iter()
        .map(|row| indexes.iter().map(|&i| row[i].clone()).collect())
        .collect();
    Ok(Table {
        columns: names,
        rows,
    })
}
```

**What happens:**

1. The **scan** is the loop over `table.rows`; the **filter** keeps a row only when the condition
   evaluates to exactly `Bool(true)`. The kept rows are references (`&Vec<Value>`): nothing is copied yet.
2. If the query has aggregates or a `GROUP BY`, the rows go to `aggregate`, otherwise to `project`.
3. `project` first resolves the selected names to column positions (once, not per row), then copies
   those cells out of each row.

The aggregation is a **hash aggregation**: one map entry per group, updated row by row:

```rust
fn aggregate(query: &Query, columns: &[String], rows: &[&Vec<Value>]) -> Result<Table, ExecError> {
    let key_indexes = query
        .group_by
        .iter()
        .map(|c| column_index(columns, c))
        .collect::<Result<Vec<_>, _>>()?;

    // One running total per group: (count, one sum per SUM item).
    let mut groups: BTreeMap<Vec<Value>, (i64, Vec<i64>)> = BTreeMap::new();
    let sum_indexes = query
        .items
        .iter()
        .filter_map(|i| match i {
            SelectItem::Sum(c) => Some(column_index(columns, c)),
            _ => None,
        })
        .collect::<Result<Vec<_>, _>>()?;

    for row in rows {
        let key: Vec<Value> = key_indexes.iter().map(|&i| row[i].clone()).collect();
        let entry = groups
            .entry(key)
            .or_insert_with(|| (0, vec![0; sum_indexes.len()]));
        entry.0 += 1;
        for (total, &i) in entry.1.iter_mut().zip(&sum_indexes) {
            match &row[i] {
                Value::Int(v) => *total += v,
                other => return Err(ExecError::TypeMismatch(format!("SUM of `{other}`"))),
            }
        }
    }

    let mut names = Vec::new();
    for item in &query.items {
        names.push(match item {
            SelectItem::Column(c) if query.group_by.contains(c) => c.clone(),
            SelectItem::Column(c) => return Err(ExecError::NotGrouped(c.clone())),
            SelectItem::CountStar => "count".to_string(),
            SelectItem::Sum(c) => format!("sum_{c}"),
        });
    }

    let mut out = Vec::new();
    for (key, (count, sums)) in groups {
        let mut sums = sums.into_iter();
        let row = query
            .items
            .iter()
            .map(|item| match item {
                SelectItem::Column(c) => {
                    let pos = query.group_by.iter().position(|g| g == c).unwrap();
                    key[pos].clone()
                }
                SelectItem::CountStar => Value::Int(count),
                SelectItem::Sum(_) => Value::Int(sums.next().unwrap()),
            })
            .collect();
        out.push(row);
    }
    Ok(Table {
        columns: names,
        rows: out,
    })
}
```

**What happens:**

1. The **group key** of a row is the vector of its `GROUP BY` values, for example `[Text("Lisbon")]`.
   Without `GROUP BY` the key is the empty vector, so all rows fall into one group.
2. `groups` maps each key to its **accumulators**: a count and one running sum per `SUM` item.
   `entry(key).or_insert_with(...)` creates them the first time a key appears.
3. After all rows are seen (aggregation is a pipeline breaker), each map entry becomes one output row.
4. A plain column in `SELECT` must be one of the `GROUP BY` columns, because only those have a single
   value per group. Otherwise we return `NotGrouped`, like every SQL database.
5. A `BTreeMap` keeps the keys sorted, so the output is ordered by the group key. That is convenient for
   tests, but SQL does not promise any order without `ORDER BY`. A `HashMap` would be a little faster.

Finally, a small printer at the end of `src/exec.rs`:

```rust
/// Prints a table with columns separated by " | ".
pub fn print_table(table: &Table) {
    println!("{}", table.columns.join(" | "));
    for row in &table.rows {
        let cells: Vec<String> = row.iter().map(|v| v.to_string()).collect();
        println!("{}", cells.join(" | "));
    }
    let n = table.rows.len();
    println!("({n} {})", if n == 1 { "row" } else { "rows" });
}
```

## 4. Running SQL over the CSV file

Replace `src/main.rs`:

```rust
use p22_query::exec::{execute, print_table, read_csv};
use p22_query::plan::parse_query;

fn main() -> Result<(), Box<dyn std::error::Error>> {
    let table = read_csv("people.csv")?;
    for sql in [
        "SELECT name, age FROM people WHERE age >= 30 AND city = 'Lisbon'",
        "SELECT city, COUNT(*), SUM(age) FROM people WHERE age >= 30 GROUP BY city",
        "SELECT COUNT(*) FROM people",
        "SELECT name FROM people WHERE town = 'Oslo'",
    ] {
        println!("> {sql}");
        let query = parse_query(sql)?;
        match execute(&query, &table) {
            Ok(result) => print_table(&result),
            Err(e) => println!("error: {e}"),
        }
        println!();
    }
    Ok(())
}
```

```bash
cargo run
```

```text
> SELECT name, age FROM people WHERE age >= 30 AND city = 'Lisbon'
name | age
Ana | 34
Chen | 41
(2 rows)

> SELECT city, COUNT(*), SUM(age) FROM people WHERE age >= 30 GROUP BY city
city | count | sum_age
Lisbon | 2 | 75
Oslo | 1 | 30
Tartu | 2 | 97
(3 rows)

> SELECT COUNT(*) FROM people
count
8
(1 row)

> SELECT name FROM people WHERE town = 'Oslo'
error: unknown column `town`
```

Compare the second result with DataFusion's answer in Lesson 22.14: the counts are the same (2, 1, 2),
and the sums give the same averages (75 / 2 = 37.5, 30, 97 / 2 = 48.5). Our engine has no `AVG` yet;
the project adds it.

## 5. The same filter, vectorized with Arrow

Our executor handles one row at a time, and each `Value` is an enum that the CPU (Central Processing Unit) must inspect before
every comparison. A vectorized engine works on whole columns. Here is the filter of Section 3 written
with Arrow **compute kernels**, in a separate package:

```bash
cd ~/projects/rust
cargo new code/p22-arrow
cd code/p22-arrow
cargo add arrow@60 --no-default-features --features prettyprint
```

`src/main.rs`:

```rust
use std::sync::Arc;

use arrow::array::{Int64Array, StringArray};
use arrow::compute::kernels::cmp::gt_eq;
use arrow::compute::{filter_record_batch, sum};
use arrow::datatypes::{DataType, Field, Schema};
use arrow::error::ArrowError;
use arrow::record_batch::RecordBatch;
use arrow::util::pretty::pretty_format_batches;

fn main() -> Result<(), ArrowError> {
    let schema = Arc::new(Schema::new(vec![
        Field::new("name", DataType::Utf8, false),
        Field::new("age", DataType::Int64, false),
    ]));
    let names = StringArray::from(vec!["Ana", "Ben", "Chen", "Dana"]);
    let ages = Int64Array::from(vec![34, 27, 41, 19]);
    let batch = RecordBatch::try_new(schema, vec![Arc::new(names), Arc::new(ages.clone())])?;

    // WHERE age >= 30: one kernel call computes the mask for the whole column.
    let mask = gt_eq(&ages, &Int64Array::new_scalar(30))?;
    println!("mask: {mask:?}");

    let adults = filter_record_batch(&batch, &mask)?;
    println!("{}", pretty_format_batches(&[adults])?);

    // SUM(age) over the whole column.
    println!("sum of all ages: {:?}", sum(&ages));
    Ok(())
}
```

```text
mask: BooleanArray
[
  true,
  false,
  true,
  false,
]
+------+-----+
| name | age |
+------+-----+
| Ana  | 34  |
| Chen | 41  |
+------+-----+
sum of all ages: Some(121)
```

**What happens:**

1. The schema says: column `name` is UTF-8 text (`Utf8`; UTF-8 is the Unicode text encoding from [Lesson 12.5](../12-computer-systems/05-text-encoding-ascii-and-utf8.md)), column `age` is a 64-bit integer, neither may
   be null (`false`). A `RecordBatch` checks that the arrays match the schema and have the same length.
2. `Int64Array` stores the ages as one plain buffer of `i64` values. `gt_eq` compares the whole column
   with the scalar 30 in one tight loop, without any enum checks, and returns a **mask**.
3. `filter_record_batch` applies the mask to every column of the batch at once.
4. `sum` adds the column. It returns `Option`, because the sum of no values (or only nulls) is `NULL` in SQL.
5. `Arc` is used because Arrow arrays are shared, read-only data: many operators may hold the same batch.

This is how DataFusion's `FilterExec` works inside: batches in, a mask computed by a kernel, batches out.

> 🧪 **Testing:** an executor is easy to test with small tables built in code (no files). Check normal
> results, empty input, and every error. Then use a reference engine: run the same SQL in DataFusion and
> compare the rows (sort both first, because row order is not defined). The project automates this.

Here is such a test module for `p22-query`. Put it at the very end of `src/exec.rs`; the exercises below add more
tests to it. The helper `people()` builds a three-row table in code, so the tests need no file:

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use crate::plan::parse_query;

    fn people() -> Table {
        let row = |n: &str, a: i64, c: &str| {
            vec![Value::Text(n.into()), Value::Int(a), Value::Text(c.into())]
        };
        Table {
            columns: vec!["name".into(), "age".into(), "city".into()],
            rows: vec![
                row("Ana", 34, "Lisbon"),
                row("Ben", 27, "Oslo"),
                row("Chen", 41, "Lisbon"),
            ],
        }
    }

    #[test]
    fn filter_and_project() {
        let q = parse_query("SELECT name FROM people WHERE age > 30").unwrap();
        let out = execute(&q, &people()).unwrap();
        assert_eq!(
            out.rows,
            [[Value::Text("Ana".into())], [Value::Text("Chen".into())]]
        );
    }

    #[test]
    fn group_by_counts_and_sums() {
        let q = parse_query("SELECT city, COUNT(*), SUM(age) FROM people GROUP BY city").unwrap();
        let out = execute(&q, &people()).unwrap();
        assert_eq!(out.columns, ["city", "count", "sum_age"]);
        assert_eq!(
            out.rows[0],
            [Value::Text("Lisbon".into()), Value::Int(2), Value::Int(75)]
        );
    }

    #[test]
    fn comparing_text_with_a_number_is_an_error() {
        let q = parse_query("SELECT name FROM people WHERE city > 3").unwrap();
        let err = execute(&q, &people()).unwrap_err();
        assert!(matches!(err, ExecError::TypeMismatch(_)));
    }
}
```

`cargo test` now runs the two parser tests of Lesson 22.15 and the three new ones:

```text
running 5 tests
test plan::tests::joins_are_rejected ... ok
test exec::tests::filter_and_project ... ok
test exec::tests::comparing_text_with_a_number_is_an_error ... ok
test plan::tests::select_with_filter_and_group_by ... ok
test exec::tests::group_by_counts_and_sums ... ok

test result: ok. 5 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

## Common mistakes

> ⚠️ **Common mistake: treating "not true" as "false".** The filter keeps a row only when the condition
> is `Bool(true)`. In real SQL a condition can also be `NULL` (unknown), and `NULL` rows are dropped too.
> Writing `!= Value::Bool(false)` instead would keep them by mistake once you add `NULL`.

> ⚠️ **Common mistake: forgetting that aggregates are pipeline breakers.** You cannot print a group's
> `COUNT(*)` before you have seen all rows. Emit the output rows only after the input loop ends.

> ⚠️ **Common mistake: `COUNT(*)` of no rows.** With our code, `SELECT COUNT(*) FROM people WHERE age > 100`
> prints `count` and `(0 rows)`. SQL says the answer is one row with `0`. Exercise 3 fixes it.

## Try it yourself

1. Run `SELECT city FROM people GROUP BY city`. You get the distinct cities, sorted, because the
   `BTreeMap` keys are sorted.
2. Run `SELECT name, COUNT(*) FROM people` and read the error.
3. In `p22-arrow`, change the scalar to `40` and check the mask.

## Exercises

### Exercise 1 — Predict the output (easy)

Without running it, write the output of `SELECT city, COUNT(*) FROM people WHERE city <> 'Oslo' GROUP BY city`
for `people.csv`. Then run it. (`<>` is SQL for "not equal".)

<details>
<summary>Hint</summary>

Count the people in Lisbon and in Tartu in the CSV file of Lesson 22.14. Remember that the output is sorted
by the group key.

</details>

<details>
<summary>Answer</summary>

```text
city | count
Lisbon | 3
Tartu | 2
(2 rows)
```

Lisbon has Ana, Chen and Hana; Tartu has Eli and Gus. `<>` becomes `BinOp::NotEq` in the plan.

</details>

### Exercise 2 — Tests for OR and GROUP BY errors (medium)

In the `tests` module of `src/exec.rs`, **write two tests** using the small `people()` table from the
module (Ana 34 Lisbon, Ben 27 Oslo, Chen 41 Lisbon): `age < 30 OR city = 'Lisbon'` must keep 3 rows, and
`SELECT name, COUNT(*) FROM people` must fail with `NotGrouped("name")`.

<details>
<summary>Hint</summary>

The existing tests show the pattern: `parse_query(...)`, then `execute(&q, &people())`, then
`unwrap()` or `unwrap_err()`. `ExecError` derives `PartialEq`, so you can compare errors with `assert_eq!`.

</details>

<details>
<summary>Answer</summary>

```rust
    #[test]
    fn or_keeps_rows_matching_either_side() {
        let q = parse_query("SELECT name FROM people WHERE age < 30 OR city = 'Lisbon'").unwrap();
        let out = execute(&q, &people()).unwrap();
        assert_eq!(out.rows.len(), 3);
    }

    #[test]
    fn plain_column_next_to_aggregate_is_an_error() {
        let q = parse_query("SELECT name, COUNT(*) FROM people").unwrap();
        let err = execute(&q, &people()).unwrap_err();
        assert_eq!(err, ExecError::NotGrouped("name".into()));
    }
```

Why it works: Ben matches the left side (27 < 30), Ana and Chen match the right side, so all three rows
stay. In the second query `name` is a plain column, there is no `GROUP BY`, and the check in `aggregate`
returns the error.

</details>

### Exercise 3 — One row for an aggregate without GROUP BY (hard)

Change `aggregate` so that a query **without** `GROUP BY` always returns exactly one row, even when no rows
pass the filter: `COUNT(*)` is `0` and, in our engine without `NULL`, every `SUM` is `0` too. **Write a
test** for `SELECT COUNT(*), SUM(age) FROM people WHERE age > 100`.

<details>
<summary>Hint</summary>

After the input loop, if `query.group_by` is empty and `groups` is empty, insert the empty key with zero
accumulators. Everything after that stays the same.

</details>

<details>
<summary>Answer</summary>

Before `let mut out = Vec::new();` in `aggregate`:

```rust
    // Without GROUP BY, an aggregate query always returns exactly one row.
    if query.group_by.is_empty() && groups.is_empty() {
        groups.insert(Vec::new(), (0, vec![0; sum_indexes.len()]));
    }
```

and the test:

```rust
    #[test]
    fn count_of_no_rows_is_zero() {
        let q = parse_query("SELECT COUNT(*), SUM(age) FROM people WHERE age > 100").unwrap();
        let out = execute(&q, &people()).unwrap();
        assert_eq!(out.rows, [[Value::Int(0), Value::Int(0)]]);
    }
```

Why it works: without `GROUP BY` every row has the empty key, so there is one group if any row passed and
none otherwise. Adding the empty group by hand makes both cases return one row. With `GROUP BY`, no input
rows correctly means no groups. (Real SQL returns `NULL` for `SUM` here; the project adds `NULL`.)

</details>

## Quiz

1. Why does `eval` return a `Result` instead of a `Value`?

<details>
<summary>Answer</summary>

Evaluation can fail on valid SQL: an unknown column, or comparing text with a number. Returning an error
lets the engine report it instead of panicking or giving a wrong answer.

</details>

2. What is a group key, and what is it when there is no `GROUP BY`?

<details>
<summary>Answer</summary>

The values of the `GROUP BY` columns for one row; rows with equal keys belong to the same group. Without
`GROUP BY` the key is empty, so all rows form one group.

</details>

3. Why must a plain column in `SELECT` appear in `GROUP BY` when the query has aggregates?

<details>
<summary>Answer</summary>

Each output row stands for a whole group. A `GROUP BY` column has one value per group; any other column can
have many different values in the group, so there is no single value to show.

</details>

4. Why is the Arrow filter faster than our row-by-row filter for large data?

<details>
<summary>Answer</summary>

It works on a whole column stored as one plain buffer of numbers, in one tight loop without enum checks or
per-row function calls. That uses the CPU cache well and lets the compiler use SIMD (Single Instruction, Multiple Data) instructions.

</details>

5. What does a mask contain?

<details>
<summary>Answer</summary>

One boolean per row: `true` for rows that pass the filter, `false` for the others. `filter_record_batch`
keeps the rows marked `true` in every column.

</details>

## Summary

- An executor turns a plan into rows: scan, filter, then either projection or aggregation.
- Expressions are evaluated recursively per row; type errors become `ExecError`, never wrong answers.
- Hash aggregation keeps one entry per group key with running accumulators, and outputs rows only after
  all input is read.
- Resolving column names to positions once ("binding") avoids repeated work per row.
- Arrow compute kernels do the same filter and sum on whole columns at once: the vectorized model.
- Test an executor with small in-code tables and compare it with a reference engine such as DataFusion.

## Sources and further reading

- [`csv` crate documentation](https://docs.rs/csv/1.4.0/csv/) — reading and writing CSV, quoting rules and the tutorial.
- [`arrow` crate documentation](https://docs.rs/arrow/60.1.0/arrow/) — arrays, record batches and the `compute` kernels.
- [Arrow compute kernels — `cmp`](https://docs.rs/arrow/60.1.0/arrow/compute/kernels/cmp/index.html) — comparison kernels such as `gt_eq`.
- [DataFusion — Architecture](https://docs.rs/datafusion/55.3.0/datafusion/index.html) — the "Architecture" part of the crate docs explains plans, `ExecutionPlan` and streams of record batches.
- [CMU 15-445/645 Database Systems](https://15445.courses.cs.cmu.edu/) — the free Carnegie Mellon University (CMU) course, with lectures on query execution, joins and aggregation.

---

[🏠 Start](../START-HERE.md) · [← 22.15 Databases track: parsing SQL](15-databases-track-parsing-sql.md) · [📋 Phase 22](README.md) · [22.17 Databases track: project and next steps →](17-databases-track-project-and-next-steps.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
