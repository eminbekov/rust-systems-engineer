<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 22.14 Databases track: how query engines work](14-databases-track-how-query-engines-work.md) · [📋 Phase 22](README.md) · [22.16 Databases track: executing queries →](16-databases-track-executing-queries.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/22-specializations/15-databases-track-parsing-sql.md) <!-- nav -->

# Lesson 22.15 — Databases track: parsing SQL

> **Phase 22 — Specializations** · Lesson 15 of 30 · ⏱ about 2.5 hours (2 sittings)

## What you will learn

- What a tokenizer and a parser do, and what a syntax tree of SQL (Structured Query Language) looks like.
- How to parse SQL with the `sqlparser` crate and read its syntax tree.
- Why an engine converts the big general syntax tree into its own small plan types.
- How to write that conversion with `match`, `let … else` and slice patterns, and reject what you do not support.
- How to test a SQL front end.

## Before you start

- You finished [Lesson 22.14 — Databases track: how query engines work](14-databases-track-how-query-engines-work.md).
- You know patterns in depth ([Lesson 06.3](../06-rust-advanced/03-patterns-in-depth.md)), `Box` for
  recursive types ([Lesson 05.9](../05-rust-intermediate/09-smart-pointers.md)), error types
  ([Lesson 04.10](../04-structs-enums-collections-errors/10-error-handling.md)) and parsing a protocol
  ([Lesson 15.7](../15-networking/07-designing-a-protocol.md)).

## New words

| Word | Meaning |
|---|---|
| **token** | The smallest meaningful piece of text: a keyword (`SELECT`), a name (`age`), a number (`30`), a symbol (`>=`). |
| **tokenizer** (lexer) | The part that splits text into tokens. |
| **parser** | The part that reads tokens and builds a tree according to the grammar. |
| **grammar** | The rules that say which sequences of tokens form valid statements. |
| **syntax tree** (AST, Abstract Syntax Tree) | The structure of the text as a tree of values: a `Select` with a list of columns, a `WHERE` expression, … |
| **dialect** | One variant of SQL: PostgreSQL, MySQL, SQLite and others accept slightly different syntax. |
| **operator precedence** | The rule that decides which operator binds stronger: in `a > 1 AND b = 2`, the comparisons are done first. |
| **identifier** | A name in SQL: a table, column or function name. |
| **literal** | A fixed value written in the query: `30`, `'Oslo'`. |

## 1. From text to a tree

A parser works in two steps. The **tokenizer** cuts the text into tokens:

```text
SELECT name , age FROM people WHERE age > 30
  |     |   |  |   |     |      |    |  |  |
 kw   ident , ident kw  ident    kw ident > number
```

The **parser** then reads the tokens from left to right and builds a tree that follows the SQL
**grammar**. For the `WHERE` part, the tree is:

```text
        BinaryOp (>)
        /          \
 Identifier(age)  Number(30)
```

**Operator precedence** decides the shape. In `age > 30 AND city = 'Oslo'`, `AND` binds weaker than
`>` and `=`, so the tree is `AND(age > 30, city = 'Oslo')`, never `age > (30 AND city) = 'Oslo'`. A
good parser takes care of this for you.

Writing a full SQL parser by hand is a large job (the compilers track, Lessons 22.22–22.25, shows how
parsers are written). For a query engine, the Rust ecosystem has a ready one: **`sqlparser`**, the
parser that DataFusion and many other databases use. It supports many **dialects** (PostgreSQL, MySQL,
SQLite, and more) and gives you the syntax tree as ordinary Rust enums and structs.

## 2. Parsing with `sqlparser`

Create the package for this lesson and the next one:

```bash
cd ~/projects/rust
cargo new code/p22-query
cd code/p22-query
cargo add sqlparser@0.64
```

A first look, in `src/main.rs`:

```rust
use sqlparser::dialect::GenericDialect;
use sqlparser::parser::Parser;

fn main() {
    let sql = "select  name ,age from people where age>30";
    let statements = Parser::parse_sql(&GenericDialect {}, sql).unwrap();
    println!("{} statement(s)", statements.len());
    // Display turns the syntax tree back into SQL text.
    println!("{}", statements[0]);
}
```

```text
1 statement(s)
SELECT name, age FROM people WHERE age > 30
```

**What happens:**

1. `Parser::parse_sql` takes a dialect and a string and returns `Result<Vec<Statement>, ParserError>`:
   the text may contain several statements separated by `;`.
2. `GenericDialect` accepts a wide, common subset of SQL. With `PostgreSqlDialect` or `MySqlDialect` you
   get the exact rules of one database.
3. Printing a `Statement` with `{}` turns the tree back into SQL. The spacing and capitals are now
   normal: the tree does not remember how the text looked, only what it means.

Print it with `{:#?}` instead and you see the tree. Here it is, much shortened (the real output is more
than 100 lines, because every node also stores its position in the text):

```text
Query(
    Query {
        body: Select(
            Select {
                projection: [
                    UnnamedExpr(Identifier(Ident { value: "name", .. })),
                    UnnamedExpr(Identifier(Ident { value: "age", .. })),
                ],
                from: [
                    TableWithJoins {
                        relation: Table { name: ObjectName([Identifier(Ident { value: "people", .. })]), .. },
                        joins: [],
                    },
                ],
                selection: Some(
                    BinaryOp {
                        left: Identifier(Ident { value: "age", .. }),
                        op: Gt,
                        right: Value(ValueWithSpan { value: Number("30", false), .. }),
                    },
                ),
                group_by: Expressions([], []),
                ..
            },
        ),
        order_by: None,
        limit_clause: None,
        ..
    },
)
```

Wrong SQL gives an `Err` whose message says what the parser expected and where. To see it, replace
the body of `main` after `let sql = …` with
`println!("{}", Parser::parse_sql(&GenericDialect {}, sql).unwrap_err());` and set `sql` to
`"SELEC name FROM people"`, then to `"SELECT name FROM people WHERE"`. The two messages are:

```text
sql parser error: Expected: an SQL statement, found: SELEC at Line: 1, Column: 1
sql parser error: Expected: an expression, found: EOF
```

(EOF means End Of File: the text ended too early.)

## 3. Our own plan types

The `sqlparser` tree describes **all** of SQL: window functions, `JOIN`s, subqueries, dozens of
dialect extras. Our engine will support a small part. So the first job of the engine's front end is to
**convert** the big tree into small types of our own, and to reject everything else with a clear message.
This has three benefits: the executor (Lesson 22.16) only sees cases it can handle; the compiler checks
that every case is covered; and our types do not change when a new version of `sqlparser` adds fields.

Create `src/plan.rs` and declare it in `src/lib.rs` with one line, `pub mod plan;`. The types:

```rust
//! Our own small query plan, built from the sqlparser syntax tree.

use std::fmt;

use sqlparser::ast::{self, SetExpr, Statement, TableFactor};
use sqlparser::dialect::GenericDialect;
use sqlparser::parser::Parser;

#[derive(Debug, Clone, PartialEq)]
pub enum Literal {
    Int(i64),
    Text(String),
}

#[derive(Debug, Clone, Copy, PartialEq)]
pub enum BinOp {
    Eq,
    NotEq,
    Lt,
    LtEq,
    Gt,
    GtEq,
    And,
    Or,
}

#[derive(Debug, Clone, PartialEq)]
pub enum Expr {
    Column(String),
    Literal(Literal),
    Binary {
        left: Box<Expr>,
        op: BinOp,
        right: Box<Expr>,
    },
}

#[derive(Debug, Clone, PartialEq)]
pub enum SelectItem {
    Column(String),
    CountStar,
    Sum(String),
}

#[derive(Debug, Clone, PartialEq)]
pub struct Query {
    pub table: String,
    pub items: Vec<SelectItem>,
    pub filter: Option<Expr>,
    pub group_by: Vec<String>,
}

#[derive(Debug, PartialEq)]
pub enum PlanError {
    Syntax(String),
    Unsupported(String),
}

impl fmt::Display for PlanError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            PlanError::Syntax(msg) => write!(f, "syntax error: {msg}"),
            PlanError::Unsupported(what) => write!(f, "not supported: {what}"),
        }
    }
}

impl std::error::Error for PlanError {}
```

**What happens:**

1. `Expr` is **recursive**: a `Binary` contains two more `Expr`s. `Box<Expr>` stores them on the heap, so
   `Expr` has a fixed size (see "Common mistakes").
2. `SelectItem` lists the only things our `SELECT` supports: a plain column, `COUNT(*)` and `SUM(column)`.
3. `Query` is a small logical plan: `Scan(table) → Filter(filter) → Aggregate(group_by) / Projection(items)`.
4. `PlanError` separates "the SQL is wrong" (`Syntax`) from "the SQL is fine, but we do not support it"
   (`Unsupported`). `Display` makes readable messages; `std::error::Error` lets `?` and `Box<dyn Error>` use it.

## 4. Converting the statement

Add the conversion function to `src/plan.rs`:

```rust
fn unsupported<T>(what: impl fmt::Display) -> Result<T, PlanError> {
    Err(PlanError::Unsupported(what.to_string()))
}

/// Parses one SELECT statement into our own `Query`.
pub fn parse_query(sql: &str) -> Result<Query, PlanError> {
    let mut statements =
        Parser::parse_sql(&GenericDialect {}, sql).map_err(|e| PlanError::Syntax(e.to_string()))?;
    if statements.len() != 1 {
        return unsupported("more than one statement");
    }
    let Statement::Query(query) = statements.remove(0) else {
        return unsupported("statements other than SELECT");
    };
    if query.order_by.is_some() || query.limit_clause.is_some() {
        return unsupported("ORDER BY and LIMIT");
    }
    let SetExpr::Select(select) = *query.body else {
        return unsupported("UNION and other set operations");
    };

    // FROM: exactly one table, no joins.
    let [from] = select.from.as_slice() else {
        return unsupported("zero or several tables in FROM");
    };
    if !from.joins.is_empty() {
        return unsupported("JOIN");
    }
    let TableFactor::Table { name, .. } = &from.relation else {
        return unsupported("subqueries in FROM");
    };
    let table = name.to_string();

    let items = select
        .projection
        .iter()
        .map(select_item)
        .collect::<Result<Vec<_>, _>>()?;
    let filter = select.selection.as_ref().map(expr).transpose()?;
    let group_by = match &select.group_by {
        ast::GroupByExpr::Expressions(exprs, _) => exprs
            .iter()
            .map(|e| match expr(e)? {
                Expr::Column(c) => Ok(c),
                other => unsupported(format!("GROUP BY {other:?}")),
            })
            .collect::<Result<Vec<_>, _>>()?,
        ast::GroupByExpr::All(_) => return unsupported("GROUP BY ALL"),
    };
    Ok(Query {
        table,
        items,
        filter,
        group_by,
    })
}
```

**What happens:**

1. `unsupported` is a small helper: it builds the error and can stand where any `Result<T, _>` is expected.
2. `let Statement::Query(query) = … else { return … };` is **`let … else`**: if the pattern does not
   match (for example a `DELETE`), we return early. After it, `query` is available directly.
3. `query.body` is a `Box<SetExpr>`; `*query.body` moves the value out of the box so we can match on it.
4. `let [from] = select.from.as_slice() else { … }` is a **slice pattern**: it matches only a slice of
   exactly one element. Zero tables or `FROM a, b` (two tables) both go to `else`.
5. `.map(select_item).collect::<Result<Vec<_>, _>>()?` converts every select item; `collect` into a
   `Result` stops at the first error, and `?` returns it.
6. `select.selection` is the `WHERE` part, an `Option<ast::Expr>`. `.map(expr)` gives
   `Option<Result<Expr, _>>`, and `.transpose()` turns it into `Result<Option<Expr>, _>`, ready for `?`.
7. `GROUP BY` must list plain columns in our engine; anything else is rejected.

## 5. Converting expressions and aggregates

The two helpers do the same kind of matching one level deeper:

```rust
fn select_item(item: &ast::SelectItem) -> Result<SelectItem, PlanError> {
    match item {
        ast::SelectItem::UnnamedExpr(ast::Expr::Identifier(id)) => {
            Ok(SelectItem::Column(id.value.clone()))
        }
        ast::SelectItem::UnnamedExpr(ast::Expr::Function(f)) => aggregate(f),
        other => unsupported(format!("select item `{other}`")),
    }
}

fn aggregate(f: &ast::Function) -> Result<SelectItem, PlanError> {
    let name = f.name.to_string().to_uppercase();
    let ast::FunctionArguments::List(list) = &f.args else {
        return unsupported(format!("function `{f}`"));
    };
    match (name.as_str(), list.args.as_slice()) {
        ("COUNT", [ast::FunctionArg::Unnamed(ast::FunctionArgExpr::Wildcard)]) => {
            Ok(SelectItem::CountStar)
        }
        (
            "SUM",
            [ast::FunctionArg::Unnamed(ast::FunctionArgExpr::Expr(ast::Expr::Identifier(id)))],
        ) => Ok(SelectItem::Sum(id.value.clone())),
        _ => unsupported(format!("function `{f}`")),
    }
}

fn expr(e: &ast::Expr) -> Result<Expr, PlanError> {
    match e {
        ast::Expr::Identifier(id) => Ok(Expr::Column(id.value.clone())),
        ast::Expr::Nested(inner) => expr(inner),
        ast::Expr::Value(v) => match &v.value {
            ast::Value::Number(n, _) => n
                .parse()
                .map(|i| Expr::Literal(Literal::Int(i)))
                .or_else(|_| unsupported(format!("number {n}"))),
            ast::Value::SingleQuotedString(s) => Ok(Expr::Literal(Literal::Text(s.clone()))),
            other => unsupported(format!("value {other}")),
        },
        ast::Expr::BinaryOp { left, op, right } => {
            let op = match op {
                ast::BinaryOperator::Eq => BinOp::Eq,
                ast::BinaryOperator::NotEq => BinOp::NotEq,
                ast::BinaryOperator::Lt => BinOp::Lt,
                ast::BinaryOperator::LtEq => BinOp::LtEq,
                ast::BinaryOperator::Gt => BinOp::Gt,
                ast::BinaryOperator::GtEq => BinOp::GtEq,
                ast::BinaryOperator::And => BinOp::And,
                ast::BinaryOperator::Or => BinOp::Or,
                other => return unsupported(format!("operator {other}")),
            };
            Ok(Expr::Binary {
                left: Box::new(expr(left)?),
                op,
                right: Box::new(expr(right)?),
            })
        }
        other => unsupported(format!("expression `{other}`")),
    }
}
```

**What happens:**

1. A function call like `COUNT(*)` is `Expr::Function`. We match the upper-case name **and** the argument
   list at the same time with a tuple pattern: `("COUNT", [Wildcard])` or `("SUM", [one column])`.
2. `expr` is **recursive**, like the tree: for `BinaryOp` it converts the left and right sides by calling
   itself. `Nested` is an expression in parentheses; the parentheses only mattered for parsing, so we unwrap them.
3. `sqlparser` keeps numbers as **text** (`Number("30", false)`), because SQL numbers can be bigger than
   any Rust type. We parse them into `i64` and reject numbers that do not fit.
4. `{other}` uses the `Display` of the `sqlparser` node, so error messages show SQL, not Rust debug output.

> 📏 **Convention:** write one small conversion function per kind of syntax node (`select_item`,
> `aggregate`, `expr`), each with a final `other => unsupported(...)` arm. The code then mirrors the shape
> of the tree, and every unsupported case has a readable error.

Now replace `src/main.rs` with a small demo:

```rust
use p22_query::plan::parse_query;

fn main() {
    for sql in [
        "SELECT name, age FROM people WHERE age > 30 AND city = 'Oslo'",
        "SELECT city, COUNT(*), SUM(age) FROM people GROUP BY city",
        "SELECT name FROM people ORDER BY name",
        "SELECT AVG(age) FROM people",
        "DELETE FROM people",
        "SELEC name FROM people",
    ] {
        println!("{sql}");
        match parse_query(sql) {
            Ok(q) => println!("  {q:?}"),
            Err(e) => println!("  error: {e}"),
        }
    }
}
```

```text
SELECT name, age FROM people WHERE age > 30 AND city = 'Oslo'
  Query { table: "people", items: [Column("name"), Column("age")], filter: Some(Binary { left: Binary { left: Column("age"), op: Gt, right: Literal(Int(30)) }, op: And, right: Binary { left: Column("city"), op: Eq, right: Literal(Text("Oslo")) } }), group_by: [] }
SELECT city, COUNT(*), SUM(age) FROM people GROUP BY city
  Query { table: "people", items: [Column("city"), CountStar, Sum("age")], filter: None, group_by: ["city"] }
SELECT name FROM people ORDER BY name
  error: not supported: ORDER BY and LIMIT
SELECT AVG(age) FROM people
  error: not supported: function `AVG(age)`
DELETE FROM people
  error: not supported: statements other than SELECT
SELEC name FROM people
  error: syntax error: sql parser error: Expected: an SQL statement, found: SELEC at Line: 1, Column: 1
```

Look at the first filter: `And` is at the top, with the two comparisons below it. The parser applied
operator precedence, and our conversion kept the shape.

> 🔒 **Security:** a parser is the first code that touches untrusted input. Our conversion never
> panics on strange SQL: every unexpected case returns `Err`. A query engine that accepts SQL from users
> should also limit the length of the text and the depth of nesting, because a deeply nested expression
> makes recursive functions use a lot of stack. `sqlparser` has a built-in recursion limit for this.

## 6. Testing the front end

Put the tests at the end of `src/plan.rs`:

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn select_with_filter_and_group_by() {
        let q =
            parse_query("SELECT city, COUNT(*) FROM people WHERE age >= 30 GROUP BY city").unwrap();
        assert_eq!(q.table, "people");
        assert_eq!(
            q.items,
            [SelectItem::Column("city".into()), SelectItem::CountStar]
        );
        assert_eq!(q.group_by, ["city"]);
        assert_eq!(
            q.filter,
            Some(Expr::Binary {
                left: Box::new(Expr::Column("age".into())),
                op: BinOp::GtEq,
                right: Box::new(Expr::Literal(Literal::Int(30))),
            })
        );
    }

    #[test]
    fn joins_are_rejected() {
        let err = parse_query("SELECT a FROM t JOIN u ON t.id = u.id").unwrap_err();
        assert_eq!(err, PlanError::Unsupported("JOIN".into()));
    }
}
```

```bash
cargo test
```

```text
running 2 tests
test plan::tests::joins_are_rejected ... ok
test plan::tests::select_with_filter_and_group_by ... ok

test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

> 🧪 **Testing:** test both sides of a front end: queries that must be accepted (with the exact plan you
> expect) and queries that must be rejected (with the exact error). Snapshot tests
> ([Lesson 09.5](../09-testing-debugging-benchmarking/05-snapshot-testing.md)) work well for plans: store
> the `{:#?}` output of many queries and review the changes. A fuzzer
> ([Lesson 09.8](../09-testing-debugging-benchmarking/08-fuzzing.md)) can check that `parse_query` never
> panics on random input.

## Common mistakes

> ⚠️ **Common mistake: a recursive enum without `Box`.** If `Binary` holds `Expr` directly, the type
> would contain itself and have infinite size:
>
> ```text
> error[E0072]: recursive type `Expr` has infinite size
>  --> src/main.rs:2:1
>   |
> 2 | enum Expr {
>   | ^^^^^^^^^
> 3 |     Column(String),
> 4 |     Binary { left: Expr, right: Expr },
>   |                    ---- recursive without indirection
> ```
>
> Fix: `Box<Expr>`, as the compiler's help line suggests.

> ⚠️ **Common mistake: assuming every number fits.** `WHERE age > 99999999999999999999` is valid SQL.
> Calling `.parse::<i64>().unwrap()` on it would panic. Our `expr` returns
> `Unsupported("number 99999999999999999999")` instead.

> ⚠️ **Common mistake: expecting one exact SQL.** Dialects differ. `GenericDialect` accepts a trailing
> comma: `SELECT name, FROM people` parses as `SELECT name FROM people`. If your engine must behave like
> PostgreSQL, use `PostgreSqlDialect` and add tests for such cases.

## Try it yourself

1. Add `"SELECT name FROM people WHERE (age > 30)"` to the demo. The parentheses become `Nested`, and the
   plan is the same as without them.
2. Add `"SELECT name FROM people; SELECT age FROM people"` and check the error.
3. Try `"SELECT \"Age\" FROM people"`. The column is `Age` with capital A: quoted identifiers keep their
   case. (Our engine treats names as exact text; real databases fold unquoted names to lower case.)

## Exercises

### Exercise 1 — Predict the error (easy)

What does `parse_query("SELECT * FROM people")` return? Predict first, then run it.

<details>
<summary>Hint</summary>

`*` in the select list is not `UnnamedExpr`. Which arm of `select_item` catches it?

</details>

<details>
<summary>Answer</summary>

```text
Err(Unsupported("select item `*`"))
```

`*` is `ast::SelectItem::Wildcard`, which only the last arm (`other => …`) matches. Supporting it means
expanding `*` into all columns of the table, which needs the table's schema; Lesson 22.17 does that.

</details>

### Exercise 2 — MIN and MAX (medium)

Add `SelectItem::Min(String)` and `SelectItem::Max(String)` and accept `MIN(column)` and `MAX(column)`,
in any letter case. **Write a test** for `SELECT MIN(age), max(age) FROM people`.

<details>
<summary>Hint</summary>

The name is already upper-cased. One match arm can accept several names with `"SUM" | "MIN" | "MAX"` and
then choose the variant with a second `match`.

</details>

<details>
<summary>Answer</summary>

In `aggregate`, replace the `SUM` arm:

```rust
        (
            "SUM" | "MIN" | "MAX",
            [ast::FunctionArg::Unnamed(ast::FunctionArgExpr::Expr(ast::Expr::Identifier(id)))],
        ) => {
            let column = id.value.clone();
            Ok(match name.as_str() {
                "SUM" => SelectItem::Sum(column),
                "MIN" => SelectItem::Min(column),
                _ => SelectItem::Max(column),
            })
        }
```

and add the test:

```rust
    #[test]
    fn min_and_max() {
        let q = parse_query("SELECT MIN(age), max(age) FROM people").unwrap();
        assert_eq!(
            q.items,
            [SelectItem::Min("age".into()), SelectItem::Max("age".into())]
        );
    }
```

Why it works: `to_uppercase()` makes `max` and `MAX` equal before matching. The `_` arm of the inner
`match` can only be `"MAX"`, because the outer pattern allowed just three names.

</details>

### Exercise 3 — LIMIT (hard)

Add `pub limit: Option<u64>` to `Query`. Accept `LIMIT n` (a non-negative whole number), still reject
`ORDER BY`, and reject `LIMIT … OFFSET …` with `Unsupported("OFFSET and other LIMIT forms")`. **Write tests**
for `LIMIT 2`, for no limit, and for `LIMIT 2 OFFSET 1`.

<details>
<summary>Hint</summary>

In `sqlparser` 0.64, `query.limit_clause` is an `Option<ast::LimitClause>`. The standard form is
`LimitClause::LimitOffset { limit, offset, limit_by }`. Convert `limit` with your own `expr` function and
accept only `Expr::Literal(Literal::Int(n))` with `n >= 0`.

</details>

<details>
<summary>Answer</summary>

Replace the `ORDER BY and LIMIT` check in `parse_query`:

```rust
    if query.order_by.is_some() {
        return unsupported("ORDER BY");
    }
    let limit = match &query.limit_clause {
        None => None,
        Some(ast::LimitClause::LimitOffset {
            limit: Some(e),
            offset: None,
            limit_by,
        }) if limit_by.is_empty() => match expr(e)? {
            Expr::Literal(Literal::Int(n)) if n >= 0 => Some(n as u64),
            _ => return unsupported("LIMIT that is not a whole number"),
        },
        Some(_) => return unsupported("OFFSET and other LIMIT forms"),
    };
```

add `limit` to the `Query { … }` at the end of the function, and test:

```rust
    #[test]
    fn limit() {
        assert_eq!(
            parse_query("SELECT name FROM people LIMIT 2")
                .unwrap()
                .limit,
            Some(2)
        );
        assert_eq!(parse_query("SELECT name FROM people").unwrap().limit, None);
        assert_eq!(
            parse_query("SELECT name FROM people LIMIT 2 OFFSET 1").unwrap_err(),
            PlanError::Unsupported("OFFSET and other LIMIT forms".into())
        );
    }
```

Why it works: the pattern with `offset: None` and the guard `if limit_by.is_empty()` accepts only a plain
`LIMIT n`. Every other form (with `OFFSET`, MySQL's `LIMIT 1, 2`, `LIMIT ALL`) falls to `Some(_)`. Note
that the demo in `main` must now show `ORDER BY` as `not supported: ORDER BY`.

</details>

## Quiz

1. What is the difference between a tokenizer and a parser?

<details>
<summary>Answer</summary>

The tokenizer splits text into tokens (keywords, names, numbers, symbols). The parser reads the tokens
and builds a syntax tree according to the grammar.

</details>

2. Why does the engine convert the `sqlparser` tree into its own types?

<details>
<summary>Answer</summary>

The `sqlparser` tree covers all of SQL. Our own small types contain only what the engine supports, so
unsupported features are rejected in one place with clear errors, the compiler checks that the executor
handles every case, and changes in `sqlparser` do not spread through the engine.

</details>

3. Why is `Expr` declared with `Box<Expr>` fields?

<details>
<summary>Answer</summary>

`Expr` is recursive (a binary expression contains expressions). Without indirection, its size would be
infinite. A `Box` has a fixed size (one pointer) and stores the inner value on the heap.

</details>

4. What does `let [from] = select.from.as_slice() else { … }` check?

<details>
<summary>Answer</summary>

That `FROM` contains exactly one table. A slice pattern `[x]` matches only slices of length one; any other
length runs the `else` block.

</details>

5. Why does `sqlparser` store a number like `30` as a string?

<details>
<summary>Answer</summary>

SQL numbers have no fixed size or precision, so the parser keeps the exact text and lets the engine decide
which type to use (and what to do when it does not fit).

</details>

## Summary

- Parsing has two steps: a tokenizer makes tokens, a parser builds a syntax tree following the grammar
  and operator precedence.
- `sqlparser` parses many SQL dialects into Rust enums and structs; `Display` turns a tree back into SQL.
- A query engine converts the general tree into its own small plan types and rejects everything else.
- `let … else`, slice patterns, tuple patterns and `Option::transpose` keep the conversion short.
- Recursive types need `Box`; numbers must be parsed without panicking.
- Test accepted and rejected queries; snapshot tests and fuzzing fit a parser well.

## Sources and further reading

- [`sqlparser` on docs.rs](https://docs.rs/sqlparser/0.64.0/sqlparser/) — the API (Application Programming Interface), the dialects and the AST types.
- [datafusion-sqlparser-rs on GitHub](https://github.com/apache/datafusion-sqlparser-rs) — the source code, examples and the list of supported syntax.
- [The Rust Reference — Statements](https://doc.rust-lang.org/reference/statements.html) — the exact rules of `let` and `let … else`.
- [The Rust Reference — Patterns](https://doc.rust-lang.org/reference/patterns.html) — all pattern kinds, including slice patterns.
- [PostgreSQL documentation — SQL syntax](https://www.postgresql.org/docs/current/sql-syntax.html) — how a real database defines tokens, identifiers and operator precedence.

---

[🏠 Start](../START-HERE.md) · [← 22.14 Databases track: how query engines work](14-databases-track-how-query-engines-work.md) · [📋 Phase 22](README.md) · [22.16 Databases track: executing queries →](16-databases-track-executing-queries.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
