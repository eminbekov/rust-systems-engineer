<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 22.14 Трек баз данных: как работают движки запросов](14-databases-track-how-query-engines-work.md) · [📋 Этап 22](README.md) · [22.16 Трек баз данных: выполнение запросов →](16-databases-track-executing-queries.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/22-specializations/15-databases-track-parsing-sql.md) <!-- nav -->

# Урок 22.15 — Трек баз данных: разбор SQL

> **Этап 22 — Специализации** · Урок 15 из 30 · ⏱ около 2,5 часа (2 занятия)

## Что вы узнаете

- Что делают токенизатор и парсер и как выглядит синтаксическое дерево SQL (Structured Query Language, язык структурированных запросов).
- Как разбирать SQL крейтом `sqlparser` и читать его синтаксическое дерево.
- Почему движок превращает большое общее синтаксическое дерево в собственные маленькие типы плана.
- Как написать это преобразование с `match`, `let … else` и образцами срезов и отклонить то, что вы не поддерживаете.
- Как тестировать фронтенд SQL.

## Перед началом

- Вы прошли [Урок 22.14 — Трек баз данных: как работают движки запросов](14-databases-track-how-query-engines-work.md).
- Вы знаете образцы подробно ([Урок 06.3](../06-rust-advanced/03-patterns-in-depth.md)), `Box` для
  рекурсивных типов ([Урок 05.9](../05-rust-intermediate/09-smart-pointers.md)), типы ошибок
  ([Урок 04.10](../04-structs-enums-collections-errors/10-error-handling.md)) и разбор протокола
  ([Урок 15.7](../15-networking/07-designing-a-protocol.md)).

## Новые слова

| Слово | Значение |
|---|---|
| **токен** (token) | Наименьший осмысленный кусок текста: ключевое слово (`SELECT`), имя (`age`), число (`30`), символ (`>=`). |
| **токенизатор** (tokenizer, lexer) | Часть, которая разбивает текст на токены. |
| **парсер** (parser) | Часть, которая читает токены и строит дерево по грамматике. |
| **грамматика** (grammar) | Правила, которые говорят, какие последовательности токенов образуют правильные операторы. |
| **синтаксическое дерево** (AST, Abstract Syntax Tree, абстрактное синтаксическое дерево) | Структура текста в виде дерева значений: `Select` со списком столбцов, выражением `WHERE`, … |
| **диалект** (dialect) | Один из вариантов SQL: PostgreSQL, MySQL, SQLite и другие принимают немного разный синтаксис. |
| **приоритет операторов** (operator precedence) | Правило, которое решает, какой оператор связывает сильнее: в `a > 1 AND b = 2` сначала выполняются сравнения. |
| **идентификатор** (identifier) | Имя в SQL: имя таблицы, столбца или функции. |
| **литерал** (literal) | Фиксированное значение, записанное в запросе: `30`, `'Oslo'`. |

## 1. От текста к дереву

Парсер работает в два шага. **Токенизатор** режет текст на токены:

```text
SELECT name , age FROM people WHERE age > 30
  |     |   |  |   |     |      |    |  |  |
 kw   ident , ident kw  ident    kw ident > number
```

Затем **парсер** читает токены слева направо и строит дерево по **грамматике** SQL. Для части `WHERE`
дерево такое:

```text
        BinaryOp (>)
        /          \
 Identifier(age)  Number(30)
```

Форму дерева определяет **приоритет операторов**. В `age > 30 AND city = 'Oslo'` оператор `AND` связывает
слабее, чем `>` и `=`, поэтому дерево — `AND(age > 30, city = 'Oslo')`, а никогда не
`age > (30 AND city) = 'Oslo'`. Хороший парсер берёт это на себя.

Написать полный парсер SQL вручную — большая работа (направление компиляторов, Уроки 22.22–22.25,
показывает, как пишут парсеры). Для движка запросов в экосистеме Rust есть готовый: **`sqlparser`** —
парсер, которым пользуются DataFusion и многие другие базы данных. Он поддерживает много **диалектов**
(PostgreSQL, MySQL, SQLite и другие) и отдаёт синтаксическое дерево обычными перечислениями и структурами Rust.

## 2. Разбор с помощью `sqlparser`

Создайте пакет для этого урока и следующего:

```bash
cd ~/projects/rust
cargo new code/p22-query
cd code/p22-query
cargo add sqlparser@0.64
```

Первый взгляд, в `src/main.rs`:

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

**Что происходит:**

1. `Parser::parse_sql` принимает диалект и строку и возвращает `Result<Vec<Statement>, ParserError>`:
   текст может содержать несколько операторов через `;`.
2. `GenericDialect` принимает широкое общее подмножество SQL. С `PostgreSqlDialect` или `MySqlDialect` вы
   получаете точные правила одной базы данных.
3. Печать `Statement` через `{}` превращает дерево обратно в SQL. Пробелы и регистр теперь обычные:
   дерево не помнит, как выглядел текст, только что он значит.

Если напечатать его через `{:#?}`, вы увидите дерево. Вот оно, сильно сокращённое (настоящий вывод длиннее
100 строк, потому что каждый узел хранит ещё и свою позицию в тексте):

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

Неправильный SQL даёт `Err`, сообщение которого говорит, что парсер ожидал и где. Чтобы увидеть его,
замените тело `main` после `let sql = …` на
`println!("{}", Parser::parse_sql(&GenericDialect {}, sql).unwrap_err());` и задайте `sql` сначала как
`"SELEC name FROM people"`, затем как `"SELECT name FROM people WHERE"`. Два сообщения:

```text
sql parser error: Expected: an SQL statement, found: SELEC at Line: 1, Column: 1
sql parser error: Expected: an expression, found: EOF
```

(EOF — End Of File, конец файла: текст закончился слишком рано.)

## 3. Собственные типы плана

Дерево `sqlparser` описывает **весь** SQL: оконные функции, `JOIN`, подзапросы, десятки особенностей
диалектов. Наш движок будет поддерживать малую часть. Поэтому первая задача фронтенда движка —
**преобразовать** большое дерево в маленькие собственные типы и отклонить всё остальное понятным
сообщением. Это даёт три выгоды: исполнитель (Урок 22.16) видит только случаи, которые умеет обрабатывать;
компилятор проверяет, что каждый случай учтён; а наши типы не меняются, когда новая версия `sqlparser`
добавляет поля.

Создайте `src/plan.rs` и объявите его в `src/lib.rs` одной строкой `pub mod plan;`. Типы:

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

**Что происходит:**

1. `Expr` **рекурсивен**: `Binary` содержит ещё два `Expr`. `Box<Expr>` хранит их в куче, поэтому у `Expr`
   фиксированный размер (см. «Частые ошибки»).
2. `SelectItem` перечисляет единственное, что поддерживает наш `SELECT`: простой столбец, `COUNT(*)` и `SUM(column)`.
3. `Query` — маленький логический план: `Scan(table) → Filter(filter) → Aggregate(group_by) / Projection(items)`.
4. `PlanError` отделяет «SQL неправильный» (`Syntax`) от «SQL правильный, но мы это не поддерживаем»
   (`Unsupported`). `Display` даёт читаемые сообщения; `std::error::Error` позволяет использовать тип с `?` и `Box<dyn Error>`.

## 4. Преобразование оператора

Добавьте функцию преобразования в `src/plan.rs`:

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

**Что происходит:**

1. `unsupported` — маленький помощник: строит ошибку и подходит везде, где ожидается любой `Result<T, _>`.
2. `let Statement::Query(query) = … else { return … };` — это **`let … else`**: если образец не совпал
   (например, это `DELETE`), мы досрочно возвращаемся. После этого `query` доступен напрямую.
3. `query.body` — это `Box<SetExpr>`; `*query.body` перемещает значение из `Box`, чтобы по нему можно было
   сопоставлять.
4. `let [from] = select.from.as_slice() else { … }` — **образец среза** (slice pattern): он совпадает
   только со срезом ровно из одного элемента. Ноль таблиц или `FROM a, b` (две таблицы) уходят в `else`.
5. `.map(select_item).collect::<Result<Vec<_>, _>>()?` преобразует каждый элемент списка `SELECT`;
   `collect` в `Result` останавливается на первой ошибке, а `?` её возвращает.
6. `select.selection` — часть `WHERE`, `Option<ast::Expr>`. `.map(expr)` даёт `Option<Result<Expr, _>>`, а
   `.transpose()` превращает это в `Result<Option<Expr>, _>`, готовый для `?`.
7. `GROUP BY` в нашем движке должен перечислять простые столбцы; всё остальное отклоняется.

## 5. Преобразование выражений и агрегатов

Два помощника делают такое же сопоставление на уровень глубже:

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

**Что происходит:**

1. Вызов функции вроде `COUNT(*)` — это `Expr::Function`. Мы сопоставляем имя в верхнем регистре **и**
   список аргументов одновременно образцом-кортежем: `("COUNT", [Wildcard])` или `("SUM", [один столбец])`.
2. `expr` **рекурсивна**, как и дерево: для `BinaryOp` она преобразует левую и правую части, вызывая саму
   себя. `Nested` — выражение в скобках; скобки были важны только для разбора, поэтому мы их снимаем.
3. `sqlparser` хранит числа **текстом** (`Number("30", false)`), потому что числа SQL могут быть больше
   любого типа Rust. Мы разбираем их в `i64` и отклоняем числа, которые не помещаются.
4. `{other}` использует `Display` узла `sqlparser`, поэтому сообщения об ошибках показывают SQL, а не
   отладочный вывод Rust.

> 📏 **Соглашение:** пишите по одной маленькой функции преобразования на каждый вид узла синтаксиса
> (`select_item`, `aggregate`, `expr`), и у каждой — последняя ветка `other => unsupported(...)`. Тогда код
> повторяет форму дерева, а у каждого неподдерживаемого случая есть понятная ошибка.

Теперь замените `src/main.rs` небольшой демонстрацией:

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

Посмотрите на первый фильтр: наверху `And`, под ним два сравнения. Парсер применил приоритет операторов,
а наше преобразование сохранило форму.

> 🔒 **Безопасность:** парсер — первый код, который трогает недоверенный ввод. Наше преобразование
> никогда не паникует на странном SQL: каждый неожиданный случай возвращает `Err`. Движок запросов,
> принимающий SQL от пользователей, должен ещё ограничивать длину текста и глубину вложенности, потому что
> глубоко вложенное выражение заставляет рекурсивные функции расходовать много стека. В `sqlparser` для
> этого есть встроенный предел рекурсии.

## 6. Тестируем фронтенд

Поместите тесты в конец `src/plan.rs`:

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

> 🧪 **Тестирование:** тестируйте обе стороны фронтенда: запросы, которые должны приниматься (с точным
> ожидаемым планом), и запросы, которые должны отклоняться (с точной ошибкой). Для планов хорошо подходят
> снапшот-тесты ([Урок 09.5](../09-testing-debugging-benchmarking/05-snapshot-testing.md)): сохраните вывод
> `{:#?}` для многих запросов и проверяйте изменения. Фаззер
> ([Урок 09.8](../09-testing-debugging-benchmarking/08-fuzzing.md)) может проверять, что `parse_query`
> никогда не паникует на случайном вводе.

## Частые ошибки

> ⚠️ **Частая ошибка: рекурсивное перечисление без `Box`.** Если `Binary` содержит `Expr` напрямую, тип
> содержал бы сам себя и имел бы бесконечный размер:
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
> Исправление: `Box<Expr>`, как подсказывает строка `help` компилятора.

> ⚠️ **Частая ошибка: думать, что любое число помещается.** `WHERE age > 99999999999999999999` — правильный
> SQL. Вызов `.parse::<i64>().unwrap()` на нём вызвал бы панику. Наша `expr` вместо этого возвращает
> `Unsupported("number 99999999999999999999")`.

> ⚠️ **Частая ошибка: ожидать один точный SQL.** Диалекты различаются. `GenericDialect` принимает
> завершающую запятую: `SELECT name, FROM people` разбирается как `SELECT name FROM people`. Если ваш движок
> должен вести себя как PostgreSQL, используйте `PostgreSqlDialect` и добавьте тесты на такие случаи.

## Попробуйте сами

1. Добавьте в демонстрацию `"SELECT name FROM people WHERE (age > 30)"`. Скобки становятся `Nested`, и
   план такой же, как без них.
2. Добавьте `"SELECT name FROM people; SELECT age FROM people"` и проверьте ошибку.
3. Попробуйте `"SELECT \"Age\" FROM people"`. Столбец — `Age` с заглавной A: идентификаторы в кавычках
   сохраняют регистр. (Наш движок считает имена точным текстом; настоящие базы данных приводят имена без
   кавычек к нижнему регистру.)

## Упражнения

### Упражнение 1 — Предскажите ошибку (легко)

Что вернёт `parse_query("SELECT * FROM people")`? Сначала предскажите, потом запустите.

<details>
<summary>Подсказка</summary>

`*` в списке `SELECT` — это не `UnnamedExpr`. Какая ветка `select_item` его поймает?

</details>

<details>
<summary>Ответ</summary>

```text
Err(Unsupported("select item `*`"))
```

`*` — это `ast::SelectItem::Wildcard`, который совпадает только с последней веткой (`other => …`).
Поддержка `*` означает развернуть его во все столбцы таблицы, а для этого нужна схема таблицы; это делает
Урок 22.17.

</details>

### Упражнение 2 — MIN и MAX (средне)

Добавьте `SelectItem::Min(String)` и `SelectItem::Max(String)` и принимайте `MIN(column)` и `MAX(column)` в
любом регистре. **Напишите тест** для `SELECT MIN(age), max(age) FROM people`.

<details>
<summary>Подсказка</summary>

Имя уже приведено к верхнему регистру. Одна ветка `match` может принимать несколько имён через
`"SUM" | "MIN" | "MAX"`, а затем выбирать вариант вторым `match`.

</details>

<details>
<summary>Ответ</summary>

В `aggregate` замените ветку `SUM`:

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

и добавьте тест:

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

Почему это работает: `to_uppercase()` делает `max` и `MAX` одинаковыми до сопоставления. Ветка `_`
внутреннего `match` может быть только `"MAX"`, потому что внешний образец пропустил всего три имени.

</details>

### Упражнение 3 — LIMIT (сложно)

Добавьте `pub limit: Option<u64>` в `Query`. Принимайте `LIMIT n` (неотрицательное целое число),
по-прежнему отклоняйте `ORDER BY` и отклоняйте `LIMIT … OFFSET …` с
`Unsupported("OFFSET and other LIMIT forms")`. **Напишите тесты** для `LIMIT 2`, для запроса без
ограничения и для `LIMIT 2 OFFSET 1`.

<details>
<summary>Подсказка</summary>

В `sqlparser` 0.64 `query.limit_clause` — это `Option<ast::LimitClause>`. Стандартная форма —
`LimitClause::LimitOffset { limit, offset, limit_by }`. Преобразуйте `limit` своей функцией `expr` и
принимайте только `Expr::Literal(Literal::Int(n))` с `n >= 0`.

</details>

<details>
<summary>Ответ</summary>

Замените проверку `ORDER BY and LIMIT` в `parse_query`:

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

добавьте `limit` в `Query { … }` в конце функции и протестируйте:

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

Почему это работает: образец с `offset: None` и условие `if limit_by.is_empty()` принимают только простой
`LIMIT n`. Все остальные формы (с `OFFSET`, `LIMIT 1, 2` из MySQL, `LIMIT ALL`) попадают в `Some(_)`.
Заметьте, что демонстрация в `main` теперь должна показывать для `ORDER BY` сообщение `not supported: ORDER BY`.

</details>

## Вопросы для самопроверки

1. Чем токенизатор отличается от парсера?

<details>
<summary>Ответ</summary>

Токенизатор разбивает текст на токены (ключевые слова, имена, числа, символы). Парсер читает токены и
строит синтаксическое дерево по грамматике.

</details>

2. Зачем движок превращает дерево `sqlparser` в собственные типы?

<details>
<summary>Ответ</summary>

Дерево `sqlparser` охватывает весь SQL. Наши маленькие типы содержат только то, что поддерживает движок,
поэтому неподдерживаемые возможности отклоняются в одном месте с понятными ошибками, компилятор проверяет,
что исполнитель обрабатывает каждый случай, а изменения в `sqlparser` не расползаются по движку.

</details>

3. Почему `Expr` объявлен с полями `Box<Expr>`?

<details>
<summary>Ответ</summary>

`Expr` рекурсивен (бинарное выражение содержит выражения). Без косвенности его размер был бы бесконечным.
У `Box` фиксированный размер (один указатель), а внутреннее значение хранится в куче.

</details>

4. Что проверяет `let [from] = select.from.as_slice() else { … }`?

<details>
<summary>Ответ</summary>

Что в `FROM` ровно одна таблица. Образец среза `[x]` совпадает только со срезами длины один; при любой
другой длине выполняется блок `else`.

</details>

5. Почему `sqlparser` хранит число вроде `30` строкой?

<details>
<summary>Ответ</summary>

У чисел SQL нет фиксированного размера и точности, поэтому парсер сохраняет точный текст и оставляет движку
решать, какой тип использовать (и что делать, если число не помещается).

</details>

## Итоги

- Разбор состоит из двух шагов: токенизатор делает токены, парсер строит синтаксическое дерево по
  грамматике и приоритету операторов.
- `sqlparser` разбирает многие диалекты SQL в перечисления и структуры Rust; `Display` превращает дерево обратно в SQL.
- Движок запросов превращает общее дерево в собственные маленькие типы плана и отклоняет всё остальное.
- `let … else`, образцы срезов, образцы-кортежи и `Option::transpose` делают преобразование коротким.
- Рекурсивным типам нужен `Box`; числа нужно разбирать без паники.
- Тестируйте принятые и отклонённые запросы; парсеру хорошо подходят снапшот-тесты и фаззинг.

## Источники и что почитать

- [`sqlparser` на docs.rs](https://docs.rs/sqlparser/0.64.0/sqlparser/) — API (Application Programming Interface, программный интерфейс), диалекты и типы AST.
- [datafusion-sqlparser-rs на GitHub](https://github.com/apache/datafusion-sqlparser-rs) — исходный код, примеры и список поддерживаемого синтаксиса.
- [The Rust Reference — Statements](https://doc.rust-lang.org/reference/statements.html) — точные правила `let` и `let … else`.
- [The Rust Reference — Patterns](https://doc.rust-lang.org/reference/patterns.html) — все виды образцов, включая образцы срезов.
- [Документация PostgreSQL — синтаксис SQL](https://www.postgresql.org/docs/current/sql-syntax.html) — как настоящая база данных определяет токены, идентификаторы и приоритет операторов.

---

[🏠 Начало](../START-HERE.md) · [← 22.14 Трек баз данных: как работают движки запросов](14-databases-track-how-query-engines-work.md) · [📋 Этап 22](README.md) · [22.16 Трек баз данных: выполнение запросов →](16-databases-track-executing-queries.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
