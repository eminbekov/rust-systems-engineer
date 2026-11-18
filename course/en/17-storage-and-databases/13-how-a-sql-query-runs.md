<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 17.12 MVCC: multi-version concurrency control](12-mvcc.md) · [📋 Phase 17](README.md) · [17.14 Project: your own key-value store →](14-project-key-value-store.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/17-storage-and-databases/13-how-a-sql-query-runs.md) <!-- nav -->

# Lesson 17.13 — How a SQL query runs

> **Phase 17 — Storage and database internals** · Lesson 13 of 15 · ⏱ about 135 minutes

## What you will learn

- The path of a query inside a database: parser, analyzer, rewriter, planner and executor.
- How to read a plan from `EXPLAIN` and `EXPLAIN ANALYZE`: nodes, costs, estimated and actual rows.
- How the planner chooses between a sequential scan and an index, using statistics, and what happens when the
  statistics are old.
- How the three join methods work: nested loop, hash join and merge join.
- How an executor is built from iterators (the "Volcano" model), by writing a tiny one in Rust.

## Before you start

- You finished [Lesson 17.12 — MVCC: multi-version concurrency control](12-mvcc.md).
- You know SQL (Structured Query Language) joins and indexes from [Lesson 16.5 — SQL and PostgreSQL basics](../16-backend-services/05-sql-and-postgresql-basics.md),
  B-trees from [Lesson 17.6 — B-trees](06-b-trees.md), and `EXPLAIN (ANALYZE, BUFFERS)` from
  [Lesson 17.9 — The page cache and the buffer pool](09-page-cache-and-buffer-pool.md).
- You know iterators, `map`, `filter` and `flat_map` (Phase 05) and hash maps from
  [Lesson 07.6 — How hash tables work](../07-algorithms-and-data-structures/06-hash-tables.md).
- The container `p17-postgres` is running.

## New words

| Word | Meaning |
|---|---|
| **parser** | The part that checks the SQL text against the grammar and turns it into a tree (a *parse tree*). |
| **analyzer** | The part that looks up every name in the parse tree (tables, columns, functions) and checks types. |
| **rewriter** | The part that applies rules to the tree, for example replaces a view by its definition. |
| **planner** (optimizer) | The part that chooses *how* to run the query: which scans, which join methods, in which order. |
| **plan** | A tree of operators (*nodes*) that the executor runs. `EXPLAIN` prints it. |
| **executor** | The part that runs the plan and produces the rows. |
| **cost** | The planner's estimate of the work for a node, in arbitrary units (about "one page read in order" = 1). |
| **selectivity** | The fraction of rows that a condition keeps. `id = 42` on a big table has a tiny selectivity; `amount > 0` keeps almost everything. |
| **statistics** | Facts about each column that `ANALYZE` collects from a sample: number of distinct values, the most common values, a histogram. |
| **nested loop join** | For every row of one input, look through the other input for matching rows. |
| **hash join** | Build a hash table from the smaller input, then look up every row of the bigger input in it. |
| **merge join** | Sort both inputs by the join key, then walk through them side by side. |
| **Volcano model** (iterator model) | An executor design where every node has one operation, "give me the next row", and pulls rows from its children. |

## 1. The journey of a query

When you send `SELECT c.name, o.amount FROM orders o JOIN customers c ON c.id = o.customer_id WHERE c.city = 'city-7'`,
the text passes through five stages:

```text
 SQL text
    |
    v
 parser      -> parse tree:   SELECT [c.name, o.amount] FROM [orders o, customers c] WHERE [...]
    |
    v
 analyzer    -> every name resolved: "orders" is table 16501, "c.city" is column 3 of type text
    |
    v
 rewriter    -> views and rules expanded (nothing to do here)
    |
    v
 planner     -> plan: which scan for each table, which join method, which order
    |
    v
 executor    -> pulls rows through the plan, sends them to the client
```

The first three stages only understand and check the query. If you misspell a column, the analyzer stops with
`ERROR: column "nmae" does not exist`. The interesting decisions happen in the **planner**: SQL says *what* you want,
never *how* to get it, so the database must choose. For one query there can be thousands of possible plans; the planner
estimates the cost of many of them and picks the cheapest.

## 2. A first plan

Create two tables in `psql` (`docker exec -it p17-postgres psql -U postgres`): 10,000 customers in 50 cities and
200,000 orders, then collect statistics with `ANALYZE`:

```sql
CREATE TABLE customers (id int PRIMARY KEY, name text NOT NULL, city text NOT NULL);
CREATE TABLE orders (id int PRIMARY KEY, customer_id int NOT NULL REFERENCES customers, amount int NOT NULL);
INSERT INTO customers SELECT i, 'customer-' || i, 'city-' || (i % 50) FROM generate_series(1, 10000) AS i;
INSERT INTO orders SELECT i, 1 + (i * 7919) % 10000, (i * 31) % 1000 FROM generate_series(1, 200000) AS i;
ANALYZE customers, orders;
```

`1 + (i * 7919) % 10000` spreads the orders evenly over the customers: every customer gets 20 orders. Now ask for the
plan of a simple query, and for the size of the table:

```sql
EXPLAIN SELECT * FROM orders WHERE customer_id = 42;
SELECT relpages, reltuples FROM pg_class WHERE relname = 'orders';
```

```text
                        QUERY PLAN
-----------------------------------------------------------
 Seq Scan on orders  (cost=0.00..3582.00 rows=20 width=12)
   Filter: (customer_id = 42)
(2 rows)

 relpages | reltuples
----------+-----------
     1082 |    200000
(1 row)
```

**What happens:**

1. The plan has one node, **Seq Scan**: read every page of `orders` and test every row with the `Filter`. There is no
   index on `customer_id`, so there is no other choice.
2. `cost=0.00..3582.00`: the first number is the cost before the first row can come out (the *startup* cost), the
   second the cost to produce all rows. The unit is "reading one page in order" (`seq_page_cost = 1`).
3. Where does 3582 come from? 1082 pages × 1, plus 200,000 rows × 0.01 (`cpu_tuple_cost`, handling a row), plus
   200,000 × 0.0025 (`cpu_operator_cost`, one `=` comparison): 1082 + 2000 + 500 = **3582**. The cost model is simple
   arithmetic with a few settings.
4. `rows=20` is an **estimate**: 200,000 rows divided by about 10,000 distinct values of `customer_id`. `width=12` is
   the estimated size of one result row in bytes.

`EXPLAIN` only plans. `EXPLAIN ANALYZE` also **runs** the query and adds what really happened:

```sql
EXPLAIN ANALYZE SELECT * FROM orders WHERE customer_id = 42;
```

```text
                                               QUERY PLAN
---------------------------------------------------------------------------------------------------------
 Seq Scan on orders  (cost=0.00..3582.00 rows=20 width=12) (actual time=0.113..4.456 rows=20.00 loops=1)
   Filter: (customer_id = 42)
   Rows Removed by Filter: 199980
   Buffers: shared hit=1082
 Planning Time: 0.012 ms
 Execution Time: 4.463 ms
(6 rows)
```

`actual time=0.113..4.456` is in milliseconds (first row..last row), `rows=20.00` is the real number of rows (the
estimate was right), `loops=1` means the node ran once. 199,980 rows were read and thrown away, from all 1,082 pages.

> ⚠️ **Common mistake:** `EXPLAIN ANALYZE` really executes the statement. `EXPLAIN ANALYZE DELETE FROM orders` deletes
> the rows. To analyze a writing statement safely, wrap it: `BEGIN; EXPLAIN ANALYZE ...; ROLLBACK;`.

## 3. Index or not: selectivity and statistics

Create an index and ask again:

```sql
CREATE INDEX orders_customer_id_idx ON orders (customer_id);
EXPLAIN ANALYZE SELECT * FROM orders WHERE customer_id = 42;
```

```text
CREATE INDEX
                                                             QUERY PLAN
------------------------------------------------------------------------------------------------------------------------------------
 Bitmap Heap Scan on orders  (cost=4.45..76.54 rows=20 width=12) (actual time=0.032..0.055 rows=20.00 loops=1)
   Recheck Cond: (customer_id = 42)
   Heap Blocks: exact=20
   Buffers: shared hit=20 read=2
   ->  Bitmap Index Scan on orders_customer_id_idx  (cost=0.00..4.45 rows=20 width=0) (actual time=0.025..0.025 rows=20.00 loops=1)
         Index Cond: (customer_id = 42)
         Index Searches: 1
         Buffers: shared read=2
 Planning:
   Buffers: shared hit=15 read=1
 Planning Time: 0.093 ms
 Execution Time: 0.069 ms
(12 rows)
```

**What happens:**

1. The plan is now a tree of two nodes. Read it from the **innermost** node (the most indented, marked `->`) outwards:
   that is the order in which data flows.
2. **Bitmap Index Scan** walks the B-tree index (2 pages) and collects the addresses of the 20 matching rows into a
   bitmap: one bit per table page that has a match.
3. **Bitmap Heap Scan** then reads only those 20 table pages (`Heap Blocks: exact=20`), in page order, and re-checks the
   condition. ("Heap" is PostgreSQL's word for the table's main storage; it is not the heap memory of Phase 03.)
4. Cost 76.54 instead of 3582, time 0.069 ms instead of 4.463 ms. The planner chose the index because it predicted few
   rows.

Is an index always better? Compare two range conditions:

```sql
EXPLAIN SELECT * FROM orders WHERE customer_id > 100;
EXPLAIN SELECT * FROM orders WHERE customer_id < 300;
```

```text
                          QUERY PLAN
---------------------------------------------------------------
 Seq Scan on orders  (cost=0.00..3582.00 rows=198001 width=12)
   Filter: (customer_id > 100)
(2 rows)

                                       QUERY PLAN
-----------------------------------------------------------------------------------------
 Bitmap Heap Scan on orders  (cost=72.96..1233.44 rows=6279 width=12)
   Recheck Cond: (customer_id < 300)
   ->  Bitmap Index Scan on orders_customer_id_idx  (cost=0.00..71.39 rows=6279 width=0)
         Index Cond: (customer_id < 300)
(4 rows)
```

`customer_id > 100` keeps 99% of the rows: the condition is hardly **selective** at all. Reading 198,001
rows through an index would jump around the table and touch every page anyway, more slowly than reading the pages in
order, so the planner ignores the index. `< 300` keeps about 3%, and the index wins. In between there is a point where
the choice flips; the planner finds it with the cost model.

The estimates (198,001 and 6,279 rows) come from **statistics** that `ANALYZE` collected from a sample of the table:

```sql
SELECT n_distinct FROM pg_stats WHERE tablename = 'orders' AND attname = 'customer_id';
```

```text
 n_distinct
------------
      10001
(1 row)
```

The true number is 10,000; the statistics are an estimate from a sample, and that is usually good enough. `pg_stats`
also stores the most common values and a histogram of each column, which give the estimates for `>` and `<`.


## 4. Three ways to join

The planner has three join methods. For a join of `orders` (big) with `customers` (small):

```text
nested loop                     hash join                         merge join
for each customer c:            build: hash table of customers    sort orders by customer_id
  for each order o with           id -> customer row              sort customers by id
    o.customer_id = c.id:       probe: for each order o,          walk both lists together,
      output (c, o)               look up o.customer_id             like merging two sorted lists
                                  in the hash table
good when the outer side is     good for big inputs without       good when both inputs are already
small and the inner side has    a useful index; needs memory      sorted (for example by an index)
an index                        for the hash table
```

Watch the planner choose. A join over one city, and a join for one customer:

```sql
EXPLAIN SELECT c.name, o.amount FROM orders o JOIN customers c ON c.id = o.customer_id WHERE c.city = 'city-7';
EXPLAIN SELECT c.name, o.amount FROM orders o JOIN customers c ON c.id = o.customer_id WHERE c.id = 42;
```

The two plans:

```text
                                 QUERY PLAN
----------------------------------------------------------------------------
 Hash Join  (cost=201.50..3808.72 rows=4000 width=17)
   Hash Cond: (o.customer_id = c.id)
   ->  Seq Scan on orders o  (cost=0.00..3082.00 rows=200000 width=8)
   ->  Hash  (cost=199.00..199.00 rows=200 width=17)
         ->  Seq Scan on customers c  (cost=0.00..199.00 rows=200 width=17)
               Filter: (city = 'city-7'::text)
(6 rows)

                                         QUERY PLAN
--------------------------------------------------------------------------------------------
 Nested Loop  (cost=4.74..85.05 rows=20 width=17)
   ->  Index Scan using customers_pkey on customers c  (cost=0.29..8.30 rows=1 width=17)
         Index Cond: (id = 42)
   ->  Bitmap Heap Scan on orders o  (cost=4.45..76.54 rows=20 width=8)
         Recheck Cond: (customer_id = 42)
         ->  Bitmap Index Scan on orders_customer_id_idx  (cost=0.00..4.45 rows=20 width=0)
               Index Cond: (customer_id = 42)
(7 rows)
```

**What happens:**

1. One city: 200 customers (one in 50). The planner scans them, builds the **Hash** from them (the small side), then
   streams all 200,000 orders past it: a **Hash Join**. Its startup cost 201.50 is the price of building the hash
   table before the first row can come out.
2. One customer: the outer side is a single row found through the primary key, and the inner side has an index. A
   **Nested Loop** that runs the inner index lookup once is the cheapest plan.
3. The query text is the same except for the `WHERE`. The plan depends on the **data**, through the estimates.

> ⚡ **Performance:** to make a slow query fast, first read its `EXPLAIN (ANALYZE, BUFFERS)`: find the node with the
> biggest `actual time`, compare estimated and actual rows, and look at `Buffers` (Lesson 17.9). Guessing which index
> to add without reading the plan wastes time and slows down every write.

## 5. When the statistics are old

Autovacuum normally runs `ANALYZE` for you after enough changes. Switch it off for
this table, add 100,000 orders for customer 7, and look at the estimate before and after a manual `ANALYZE`:

```sql
ALTER TABLE orders SET (autovacuum_enabled = false);
INSERT INTO orders SELECT i, 7, 500 FROM generate_series(200001, 300000) AS i;
EXPLAIN ANALYZE SELECT count(*) FROM orders WHERE customer_id = 7;
ANALYZE orders;
EXPLAIN ANALYZE SELECT count(*) FROM orders WHERE customer_id = 7;
```

The `Bitmap Heap Scan` line of the two plans:

```text
   ->  Bitmap Heap Scan on orders  (cost=4.53..112.67 rows=30 width=0) (actual time=1.400..6.671 rows=100020.00 loops=1)
   ->  Bitmap Heap Scan on orders  (cost=1141.70..4017.57 rows=100310 width=0) (actual time=1.007..6.194 rows=100020.00 loops=1)
```

Before `ANALYZE` the planner expected 30 rows and got 100,020: it knew that the table had grown (it checks the current
size), but still believed that every customer has the same number of orders. After `ANALYZE` the estimate is close.
Here the plan stayed the same, because the new rows sit together at the end of the table. In a join, an estimate that
is wrong by a factor of 3,000 easily makes the planner choose a nested loop that runs 100,000 times: **wrong estimates
are the most common reason for slow plans**. When `rows=` estimated and `rows=` actual differ a lot, suspect the
statistics first. Switch autovacuum back on with `ALTER TABLE orders RESET (autovacuum_enabled);`.

## 6. An executor made of iterators

How does the executor run such a tree? Most databases use the **Volcano model**: every node is an object with one
operation, `next()`, "give me the next row (or nothing)". A node calls `next()` on its children as often as it needs.
The root is pulled by the client, row by row.

Rust iterators work exactly like this: `Iterator::next` is that operation, and `filter`, `map`, `flat_map` are nodes.
Build a tiny executor:

```bash
cd ~/projects/rust
cargo new --lib code/p17-executor
cd code/p17-executor
```

Replace `src/lib.rs` with:

```rust
use std::collections::HashMap;

/// One value in a row.
#[derive(Debug, Clone, PartialEq, Eq, Hash, PartialOrd, Ord)]
pub enum Value {
    Int(i64),
    Text(String),
}

pub type Row = Vec<Value>;

/// Every operator is an iterator of rows: the "Volcano" model.
pub type Rows<'a> = Box<dyn Iterator<Item = Row> + 'a>;

/// Seq Scan: returns every row of a table.
pub fn scan(table: &[Row]) -> Rows<'_> {
    Box::new(table.iter().cloned())
}

/// Filter: passes on only the rows for which `predicate` is true.
pub fn filter<'a>(input: Rows<'a>, predicate: impl Fn(&Row) -> bool + 'a) -> Rows<'a> {
    Box::new(input.filter(move |row| predicate(row)))
}

/// Project: keeps only the columns at the given positions, in that order.
pub fn project<'a>(input: Rows<'a>, columns: Vec<usize>) -> Rows<'a> {
    Box::new(input.map(move |row| columns.iter().map(|&c| row[c].clone()).collect()))
}

/// Hash Join: builds a hash table from `build`, then streams `probe` through it.
/// Output rows are the probe row followed by the matching build row.
pub fn hash_join<'a>(
    build: Rows<'a>,
    build_key: usize,
    probe: Rows<'a>,
    probe_key: usize,
) -> Rows<'a> {
    let mut table: HashMap<Value, Vec<Row>> = HashMap::new();
    for row in build {
        table.entry(row[build_key].clone()).or_default().push(row);
    }
    Box::new(probe.flat_map(move |p| {
        let matches = table.get(&p[probe_key]).cloned().unwrap_or_default();
        matches.into_iter().map(move |b| {
            let mut joined = p.clone();
            joined.extend(b);
            joined
        })
    }))
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn filter_and_project_work_together() {
        let table = vec![
            vec![Value::Int(1), Value::Text("a".into())],
            vec![Value::Int(2), Value::Text("b".into())],
        ];
        let rows: Vec<Row> =
            project(filter(scan(&table), |r| r[0] == Value::Int(2)), vec![1]).collect();
        assert_eq!(rows, vec![vec![Value::Text("b".into())]]);
    }
}
```

**What happens:**

1. A `Row` is a `Vec<Value>`; a column is a position in it. `Value` derives `PartialOrd` and `Ord` so that rows can be
   compared with `>` and sorted. (Comparing an `Int` with a `Text` is allowed by the derive, which orders variants by
   their position; a real database would reject it in the analyzer.)
2. `Rows<'a>` is a boxed iterator: "some node that produces rows". `Box<dyn Iterator>` lets us plug nodes of different
   types together at run time, as a planner does.
3. `scan` produces copies of the table's rows; `filter` and `project` wrap their input and do their work only when
   somebody asks for the next row. Nothing happens before that: the iterators are **lazy**.
4. `hash_join` has two phases. The *build* phase runs in the function itself: it reads the whole `build` input into a
   `HashMap` from key to rows. The *probe* phase is lazy: `flat_map` turns each probe row into zero, one or many joined
   rows.

Now run the query from Section 4 on tiny tables. Create `src/bin/query.rs`:

```rust
use p17_executor::{Row, Value, filter, hash_join, project, scan};
use std::cell::Cell;

fn main() {
    // customers(id, name, city) and orders(id, customer_id, amount)
    let customers: Vec<Row> = (1..=4)
        .map(|i| {
            vec![
                Value::Int(i),
                Value::Text(format!("customer-{i}")),
                Value::Text(format!("city-{}", i % 2)),
            ]
        })
        .collect();
    let orders: Vec<Row> = (1..=8)
        .map(|i| vec![Value::Int(i), Value::Int(1 + i % 4), Value::Int(i * 40)])
        .collect();

    // SELECT c.name, o.amount
    // FROM orders o JOIN customers c ON c.id = o.customer_id
    // WHERE o.amount > 100 AND c.city = 'city-1'
    let pulled = Cell::new(0); // counts rows read from `orders`
    let order_rows = scan(&orders).inspect(|_| pulled.set(pulled.get() + 1));
    let big_orders = filter(Box::new(order_rows), |o| o[2] > Value::Int(100));
    let city_1 = filter(scan(&customers), |c| c[2] == Value::Text("city-1".into()));
    let joined = hash_join(city_1, 0, big_orders, 1); // o.* then c.*
    let result = project(joined, vec![4, 2]); // c.name, o.amount

    for row in result.take(2) {
        println!("{row:?}");
    }
    println!(
        "rows read from orders: {} of {}",
        pulled.get(),
        orders.len()
    );
}
```

```bash
cargo run --bin query
```

```text
[Text("customer-1"), Int(160)]
[Text("customer-3"), Int(240)]
rows read from orders: 6 of 8
```

**What happens:**

1. The plan is built by nesting function calls, from the leaves to the root: scans, filters, the hash join with the
   small filtered `customers` as the build side, then the projection of `c.name` (column 4 of a joined row) and
   `o.amount` (column 2).
2. `inspect` counts every row that the scan of `orders` hands out. `Cell` allows changing the counter from inside the
   closure while `main` still reads it.
3. `result.take(2)` is a `LIMIT 2`. After two output rows nobody calls `next()` any more, so the scan of `orders` stopped
   after 6 of its 8 rows. This is the strength of the pull model: a `LIMIT` stops all the work below it.
4. The hash join's build side was read completely, because a hash table must be finished before the first probe. That
   is the startup cost you saw in PostgreSQL's `Hash Join`.

> 🧪 **Testing:** operators are small pure functions of their inputs, so they are easy to test with tiny hand-made
> tables: an empty input, no matches, several matches for one key. Exercises 2 and 3 test the joins this way, and check
> two join methods against each other.

## Common mistakes

> ⚠️ **Common mistake:** reading a plan from the top. The top node is the **last** step. Data flows from the most
> indented nodes upwards.

> ⚠️ **Common mistake:** treating `cost` as milliseconds. Costs are units of the planner's model, useful only to
> compare plans of the same query. Real time is in `actual time` of `EXPLAIN ANALYZE`.

> ⚠️ **Common mistake:** adding an index and expecting every query to use it. If a condition keeps a large part of
> the table, a sequential scan is cheaper, and the planner is right to ignore the index (Section 3).

## Try it yourself

1. Now that `orders` has 300,000 rows and fresh statistics, `EXPLAIN` the city join of Section 4 again. The plan
   changed to a `Nested Loop` with an `Index Scan using orders_customer_id_idx` (cost 5261.13): same query, new data,
   new plan. Then run `SET enable_nestloop = off;` and `EXPLAIN` once more: the planner must fall back to the
   `Hash Join`, which it now estimates at 5611.33. `RESET enable_nestloop;` turns it back on. These switches exist
   for experiments like this, not for production.
2. In `query.rs`, change `take(2)` to `take(10)`. All 8 orders are read, and only 3 rows come out.

## Exercises

### Exercise 1 — Read a plan (easy)

In the `Hash Join` plan of Section 4: which node produces the first rows, which nodes run fully before the join can
output anything, and how many rows did the planner expect from the join?

<details>
<summary>Hint</summary>

Start from the most indented node. A `Hash` node must read all its input before it is ready.

</details>

<details>
<summary>Answer</summary>

`Seq Scan on customers c` (with the `Filter` on the city) produces the first rows. It and the `Hash` node above it run
completely first: the hash table of about 200 customers must be finished. Then `Seq Scan on orders o` streams 200,000
rows through the `Hash Join`, which was expected to output `rows=4000` (200 customers × 20 orders).

Why it works: the startup cost of the `Hash Join` (201.50) is about the total cost of the `Hash` (199.00), which shows
that the build side runs before the first output row.

</details>

### Exercise 2 — Test the hash join (medium)

Write a test `hash_join_repeats_matches_and_drops_rows_without_a_partner`: two customers, three orders, of which two
belong to customer 1 and one belongs to a customer 3 who does not exist. The join must return two rows, both for
customer 1.

<details>
<summary>Hint</summary>

A helper `fn row(values: &[i64]) -> Row` that turns `&[1, 10]` into a row of `Value::Int`s keeps the test short.

</details>

<details>
<summary>Answer</summary>

```rust
    fn row(values: &[i64]) -> Row {
        values.iter().map(|&v| Value::Int(v)).collect()
    }

    #[test]
    fn hash_join_repeats_matches_and_drops_rows_without_a_partner() {
        let customers = vec![row(&[1, 10]), row(&[2, 20])];
        let orders = vec![row(&[100, 1]), row(&[101, 1]), row(&[102, 3])];
        let joined: Vec<Row> = hash_join(scan(&customers), 0, scan(&orders), 1).collect();
        assert_eq!(
            joined,
            vec![row(&[100, 1, 1, 10]), row(&[101, 1, 1, 10])] // order 102 has no customer 3
        );
    }
```

Why it works: an inner join returns one row for every matching pair; an order without a matching customer has no pair,
so it disappears; a customer with two orders appears twice.

</details>

### Exercise 3 — A nested loop join (hard)

Add `pub fn nested_loop_join<'a>(outer: Rows<'a>, outer_key: usize, inner: &'a [Row], inner_key: usize) -> Rows<'a>`:
for every outer row, look through the whole `inner` slice. Then write a test that joins the same data with both methods
and checks that the results are equal after sorting.

<details>
<summary>Hint</summary>

The inner side must be read again for every outer row, so it is a slice, not an iterator (an iterator can be read only
once). Collect the matches of one outer row into a `Vec` and return it from `flat_map`. The two joins put the columns
in a different order unless you choose the sides carefully: use `orders` as the probe side of the hash join and as the
outer side of the nested loop. The test uses the `row` helper from Exercise 2.

</details>

<details>
<summary>Answer</summary>

```rust
/// Nested Loop Join: for every outer row, scans the whole inner table again.
pub fn nested_loop_join<'a>(
    outer: Rows<'a>,
    outer_key: usize,
    inner: &'a [Row],
    inner_key: usize,
) -> Rows<'a> {
    Box::new(outer.flat_map(move |o| {
        let mut matches = Vec::new();
        for i in inner.iter().filter(|i| i[inner_key] == o[outer_key]) {
            let mut joined = o.clone();
            joined.extend(i.iter().cloned());
            matches.push(joined);
        }
        matches
    }))
}
```

```rust
    #[test]
    fn nested_loop_and_hash_join_agree() {
        let customers: Vec<Row> = (1..=5).map(|i| row(&[i, i * 10])).collect();
        let orders: Vec<Row> = (1..=20).map(|i| row(&[i, i % 7])).collect();
        let mut by_hash: Vec<Row> = hash_join(scan(&customers), 0, scan(&orders), 1).collect();
        let mut by_loop: Vec<Row> = nested_loop_join(scan(&orders), 1, &customers, 0).collect();
        by_hash.sort();
        by_loop.sort();
        assert_eq!(by_hash, by_loop);
        assert_eq!(by_hash.len(), 15); // orders whose i % 7 is 1..=5
    }
```

Why it works: both methods produce every matching pair exactly once, only in a different order, so sorted results must
be equal. Testing one implementation against another (a simple, obviously correct one against a fast one) is a
powerful technique. The nested loop does `outer × inner` comparisons, 20 × 5 here; the hash join does about
`outer + inner` operations, which is why the planner prefers it for big inputs.

</details>

## Quiz

1. Which stage of query processing would report `ERROR: column "nmae" does not exist`, and which chooses between a
   sequential scan and an index?

<details>
<summary>Answer</summary>

The analyzer reports unknown names. The planner chooses the scans and joins.

</details>

2. What do the two numbers in `cost=4.45..76.54` mean?

<details>
<summary>Answer</summary>

The estimated startup cost (before the first row) and the estimated total cost (for all rows), in the planner's units
where reading one page sequentially costs 1.

</details>

3. Why does PostgreSQL ignore the index for `customer_id > 100`?

<details>
<summary>Answer</summary>

The condition keeps about 99% of the rows. Reading them through the index would touch every page anyway, in a worse
order, so the sequential scan is cheaper.

</details>

4. `EXPLAIN ANALYZE` shows `rows=30` estimated and `rows=100020` actual. What do you check first?

<details>
<summary>Answer</summary>

The statistics: run `ANALYZE` on the table (and find out why autovacuum did not). Wrong estimates lead to wrong plans.

</details>

5. In the Volcano model, why does a `LIMIT 2` make the whole query cheaper?

<details>
<summary>Answer</summary>

Every node produces rows only when its parent asks for the next one. After two rows the root stops asking, so the
nodes below stop working too.

</details>

## Summary

- A query goes through the parser, analyzer, rewriter, planner and executor; the planner chooses *how* to run it.
- A plan is a tree; read it from the most indented node up. `EXPLAIN` shows estimates, `EXPLAIN ANALYZE` runs the query
  and adds actual times and rows.
- Costs are model units built from settings like `seq_page_cost` and `cpu_tuple_cost`.
- The planner uses statistics to estimate selectivity: an index for few rows, a sequential scan for many. Old
  statistics give wrong estimates; `ANALYZE` refreshes them.
- Joins: nested loop (small outer side, indexed inner side), hash join (big inputs), merge join (sorted inputs).
- Executors pull rows through a tree of iterators; Rust's `Iterator` is exactly this model, and laziness makes `LIMIT`
  cheap.

## Sources and further reading

- [PostgreSQL: The Path of a Query](https://www.postgresql.org/docs/current/query-path.html) — the stages of Section 1, with links to each one.
- [PostgreSQL: Using EXPLAIN](https://www.postgresql.org/docs/current/using-explain.html) — how to read plans, with many examples.
- [PostgreSQL: Statistics Used by the Planner](https://www.postgresql.org/docs/current/planner-stats.html) and [Row Estimation Examples](https://www.postgresql.org/docs/current/row-estimation-examples.html) — where `rows=` comes from.
- [PostgreSQL: Query Planning settings](https://www.postgresql.org/docs/current/runtime-config-query.html) — `seq_page_cost`, `cpu_tuple_cost`, `enable_hashjoin` and friends.
- [SQLite: The SQLite Query Optimizer Overview](https://www.sqlite.org/queryplanner.html) — the same ideas explained for another database, very readable.
- [CMU 15-445/645 Database Systems](https://15445.courses.cs.cmu.edu/) — CMU is Carnegie Mellon University; lectures on query execution and optimization.

---

[🏠 Start](../START-HERE.md) · [← 17.12 MVCC: multi-version concurrency control](12-mvcc.md) · [📋 Phase 17](README.md) · [17.14 Project: your own key-value store →](14-project-key-value-store.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
