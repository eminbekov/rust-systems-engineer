<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 22.15 Трек баз данных: разбор SQL](15-databases-track-parsing-sql.md) · [📋 Этап 22](README.md) · [22.17 Трек баз данных: проект и следующие шаги →](17-databases-track-project-and-next-steps.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/22-specializations/16-databases-track-executing-queries.md) <!-- nav -->

# Урок 22.16 — Трек баз данных: выполнение запросов

> **Этап 22 — Специализации** · Урок 16 из 30 · ⏱ около 2,5 часа (2 занятия)

## Что вы узнаете

- Как загрузить CSV-файл (Comma-Separated Values, значения, разделённые запятыми) в таблицу типизированных
  значений, чтобы по ней можно было выполнять запросы SQL (Structured Query Language, язык структурированных запросов).
- Как вычислить выражение вроде `age >= 30 AND city = 'Lisbon'` для одной строки.
- Как реализовать операторы сканирования, фильтра, проекции и хеш-агрегации с `GROUP BY`.
- Как тот же фильтр выглядит в векторизованном движке на вычислительных ядрах Apache Arrow.
- Как тестировать исполнитель и сравнивать его с эталонным движком.

## Перед началом

- Вы прошли [Урок 22.15 — Трек баз данных: разбор SQL](15-databases-track-parsing-sql.md), и у вас есть
  `code/p22-query` с `src/plan.rs`. Этот урок добавляет в тот же пакет `src/exec.rs`.
- Вы знаете хеш-таблицы ([Урок 04.9](../04-structs-enums-collections-errors/09-hash-maps.md)) и как они
  устроены внутри ([Урок 07.6](../07-algorithms-and-data-structures/06-hash-tables.md)), а также трейт-объекты
  для ошибок (`Box<dyn Error>`, [Урок 06.1](../06-rust-advanced/01-trait-objects-and-dynamic-dispatch.md)).

## Новые слова

| Слово | Значение |
|---|---|
| **исполнитель** (executor) | Часть движка запросов, которая выполняет план и выдаёт строки результата. |
| **вычисление** (evaluation) | Получение значения выражения для одной строки, например `age >= 30` → `true`. |
| **хеш-агрегация** (hash aggregation) | Группировка, при которой каждая строка попадает в запись словаря для своего ключа группы, где обновляются накопленные итоги. |
| **ключ группы** (group key) | Значения столбцов `GROUP BY` для одной строки, например `["Lisbon"]`. |
| **аккумулятор** (accumulator) | Текущее состояние одного агрегата для одной группы: счётчик, сумма. |
| **вычислительное ядро** (compute kernel) | Функция Arrow, которая работает сразу с целыми массивами: сравнение, фильтр, сумма. |
| **маска** (mask) | Массив `true`/`false`, по значению на строку, который говорит, какие строки оставить. |

## 1. Значения, таблицы и чтение CSV

Наш исполнитель работает с простой таблицей в памяти: списком имён столбцов и списком строк, где каждая
строка — `Vec<Value>`. Добавьте в пакет крейт для CSV:

```bash
cd ~/projects/rust/code/p22-query
cargo add csv@1
cp ../p22-datafusion/people.csv .
```

Создайте `src/exec.rs` и добавьте `pub mod exec;` в `src/lib.rs`. Первая часть файла:

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

**Что происходит:**

1. `Value` — тип одной ячейки. `Bool` в таблицах не хранится; это результат сравнений вроде `age >= 30`.
   Выведенные `Ord` и `Hash` позволят потом использовать значения как ключи словаря.
2. `ExecError` перечисляет, что может пойти не так *при выполнении* правильного плана: столбец, которого
   нет в таблице, сравнение текста с числом и простой столбец рядом с агрегатом без `GROUP BY`.
3. `csv::Reader::from_path` открывает файл и считает первую строку заголовком. Крейт `csv` правильно
   обрабатывает поля в кавычках вроде `"Lisbon, Portugal"`, а простой `split(',')` — нет.
4. Каждое поле становится `Int`, если разбирается как `i64`, иначе `Text`. Это угадывание типа **по
   значению**; настоящий движок выводит один тип **на столбец** (это делает проект в Уроке 22.17).

## 2. Вычисление выражений

Дальше — вычислитель выражений. Он рекурсивно обходит дерево `Expr`, как преобразователь в Уроке 22.15:

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

**Что происходит:**

1. Столбец ищется по имени, и его значение копируется из строки.
2. Для `Binary` сначала вычисляются обе стороны, затем применяется оператор. `match` по кортежу
   `(op, &l, &r)` обрабатывает каждую допустимую комбинацию: `AND`/`OR` требуют двух булевых значений;
   сравнения — двух значений **одного** типа.
3. Два `Int` сравниваются как числа, а два `Text` — по алфавиту, через выведенный `PartialOrd`.
   `'Oslo' > 3` попадает в последнюю ветку и становится ошибкой `TypeMismatch`, а не неверным ответом.
4. `unreachable!()` отмечает случай, который уже исключили предыдущие ветки; он паникует, если наше
   рассуждение когда-нибудь окажется неверным.

> ⚡ **Производительность:** `column_index` ищет среди имён столбцов для каждой строки и каждой ссылки на
> столбец. Настоящие движки один раз **связывают** (bind) план перед выполнением: заменяют имена позициями
> (`age@0` у DataFusion в Уроке 22.14). Для нескольких тысяч строк это неважно; для миллионов — важно.

## 3. Операторы: сканирование, фильтр, проекция, агрегация

Теперь операторы. `execute` выполняет план снизу вверх:

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

**Что происходит:**

1. **Сканирование** — это цикл по `table.rows`; **фильтр** оставляет строку, только когда условие
   вычисляется ровно в `Bool(true)`. Оставленные строки — ссылки (`&Vec<Value>`): пока ничего не копируется.
2. Если в запросе есть агрегаты или `GROUP BY`, строки идут в `aggregate`, иначе — в `project`.
3. `project` сначала превращает выбранные имена в позиции столбцов (один раз, а не для каждой строки),
   а затем копирует эти ячейки из каждой строки.

Агрегация — это **хеш-агрегация**: по одной записи словаря на группу, обновляемой строка за строкой:

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

**Что происходит:**

1. **Ключ группы** строки — вектор её значений `GROUP BY`, например `[Text("Lisbon")]`. Без `GROUP BY`
   ключ — пустой вектор, поэтому все строки попадают в одну группу.
2. `groups` сопоставляет каждому ключу его **аккумуляторы**: счётчик и по одной накапливаемой сумме на
   каждый элемент `SUM`. `entry(key).or_insert_with(...)` создаёт их, когда ключ встречается впервые.
3. Когда все строки просмотрены (агрегация — разрыв конвейера), каждая запись словаря становится одной
   строкой результата.
4. Простой столбец в `SELECT` должен быть одним из столбцов `GROUP BY`, потому что только у них одно
   значение на группу. Иначе мы возвращаем `NotGrouped`, как любая база данных SQL.
5. `BTreeMap` хранит ключи отсортированными, поэтому результат упорядочен по ключу группы. Это удобно для
   тестов, но SQL без `ORDER BY` не обещает никакого порядка. `HashMap` был бы чуть быстрее.

Наконец, небольшая функция печати в конце `src/exec.rs`:

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

## 4. Выполняем SQL над CSV-файлом

Замените `src/main.rs`:

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

Сравните второй результат с ответом DataFusion в Уроке 22.14: количества те же (2, 1, 2), а суммы дают
те же средние (75 / 2 = 37,5, 30, 97 / 2 = 48,5). `AVG` в нашем движке пока нет; его добавляет проект.

## 5. Тот же фильтр, векторизованный с Arrow

Наш исполнитель обрабатывает по одной строке, и каждое `Value` — перечисление, которое процессор (CPU,
Central Processing Unit) должен проверить перед каждым сравнением. Векторизованный движок работает с целыми
столбцами. Вот фильтр из раздела 3, написанный на **вычислительных ядрах** Arrow, в отдельном пакете:

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

**Что происходит:**

1. Схема говорит: столбец `name` — текст UTF-8 (`Utf8`; UTF-8 — кодировка Unicode из
   [Урока 12.5](../12-computer-systems/05-text-encoding-ascii-and-utf8.md)), столбец `age` — 64-битное целое,
   ни один не может быть null (`false`). `RecordBatch` проверяет, что массивы соответствуют схеме и имеют
   одинаковую длину.
2. `Int64Array` хранит возрасты одним простым буфером значений `i64`. `gt_eq` сравнивает весь столбец со
   скаляром 30 в одном плотном цикле, без проверок перечислений, и возвращает **маску**.
3. `filter_record_batch` применяет маску сразу ко всем столбцам пакета.
4. `sum` складывает столбец. Она возвращает `Option`, потому что сумма никаких значений (или только null)
   в SQL — это `NULL`.
5. `Arc` используется потому, что массивы Arrow — общие данные только для чтения: один пакет могут держать
   многие операторы.

Именно так внутри работает `FilterExec` в DataFusion: пакеты на входе, маска от ядра, пакеты на выходе.

> 🧪 **Тестирование:** исполнитель легко тестировать на маленьких таблицах, построенных в коде (без файлов).
> Проверяйте обычные результаты, пустой вход и каждую ошибку. Затем используйте эталонный движок: выполните
> тот же SQL в DataFusion и сравните строки (сначала отсортируйте обе стороны, потому что порядок строк не
> определён). Проект это автоматизирует.

Вот такой тестовый модуль для `p22-query`. Поставьте его в самый конец `src/exec.rs`; упражнения ниже добавят в него
новые тесты. Вспомогательная функция `people()` строит таблицу из трёх строк прямо в коде, поэтому тестам не нужен
файл:

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

Теперь `cargo test` запускает два теста парсера из Урока 22.15 и три новых:

```text
running 5 tests
test plan::tests::joins_are_rejected ... ok
test exec::tests::filter_and_project ... ok
test exec::tests::comparing_text_with_a_number_is_an_error ... ok
test plan::tests::select_with_filter_and_group_by ... ok
test exec::tests::group_by_counts_and_sums ... ok

test result: ok. 5 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

## Частые ошибки

> ⚠️ **Частая ошибка: считать «не истину» «ложью».** Фильтр оставляет строку, только когда условие равно
> `Bool(true)`. В настоящем SQL условие может быть ещё и `NULL` (неизвестно), и такие строки тоже
> отбрасываются. Если написать `!= Value::Bool(false)`, такие строки по ошибке останутся, когда вы добавите `NULL`.

> ⚠️ **Частая ошибка: забыть, что агрегаты — разрывы конвейера.** Нельзя напечатать `COUNT(*)` группы,
> пока не просмотрены все строки. Выдавайте строки результата только после окончания цикла по входу.

> ⚠️ **Частая ошибка: `COUNT(*)` по пустому набору.** С нашим кодом
> `SELECT COUNT(*) FROM people WHERE age > 100` печатает `count` и `(0 rows)`. По правилам SQL ответ — одна
> строка с `0`. Упражнение 3 это исправляет.

## Попробуйте сами

1. Выполните `SELECT city FROM people GROUP BY city`. Вы получите различные города в отсортированном виде,
   потому что ключи `BTreeMap` отсортированы.
2. Выполните `SELECT name, COUNT(*) FROM people` и прочитайте ошибку.
3. В `p22-arrow` замените скаляр на `40` и проверьте маску.

## Упражнения

### Упражнение 1 — Предскажите вывод (легко)

Не запуская, запишите вывод
`SELECT city, COUNT(*) FROM people WHERE city <> 'Oslo' GROUP BY city` для `people.csv`. Затем запустите.
(`<>` в SQL означает «не равно».)

<details>
<summary>Подсказка</summary>

Посчитайте людей из Lisbon и Tartu в CSV-файле из Урока 22.14. Помните, что результат отсортирован по ключу группы.

</details>

<details>
<summary>Ответ</summary>

```text
city | count
Lisbon | 3
Tartu | 2
(2 rows)
```

В Lisbon — Ana, Chen и Hana; в Tartu — Eli и Gus. `<>` в плане становится `BinOp::NotEq`.

</details>

### Упражнение 2 — Тесты для OR и ошибок GROUP BY (средне)

В модуле `tests` файла `src/exec.rs` **напишите два теста** на маленькой таблице `people()` из модуля
(Ana 34 Lisbon, Ben 27 Oslo, Chen 41 Lisbon): `age < 30 OR city = 'Lisbon'` должен оставить 3 строки, а
`SELECT name, COUNT(*) FROM people` должен завершиться ошибкой `NotGrouped("name")`.

<details>
<summary>Подсказка</summary>

Существующие тесты показывают образец: `parse_query(...)`, затем `execute(&q, &people())`, затем `unwrap()`
или `unwrap_err()`. `ExecError` выводит `PartialEq`, поэтому ошибки можно сравнивать через `assert_eq!`.

</details>

<details>
<summary>Ответ</summary>

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

Почему это работает: Ben подходит под левую часть (27 < 30), Ana и Chen — под правую, поэтому все три
строки остаются. Во втором запросе `name` — простой столбец, `GROUP BY` нет, и проверка в `aggregate`
возвращает ошибку.

</details>

### Упражнение 3 — Одна строка для агрегата без GROUP BY (сложно)

Измените `aggregate` так, чтобы запрос **без** `GROUP BY` всегда возвращал ровно одну строку, даже если
фильтр не пропустил ни одной строки: `COUNT(*)` равен `0`, и в нашем движке без `NULL` каждый `SUM` тоже
равен `0`. **Напишите тест** для `SELECT COUNT(*), SUM(age) FROM people WHERE age > 100`.

<details>
<summary>Подсказка</summary>

После цикла по входу, если `query.group_by` пуст и `groups` пуст, вставьте пустой ключ с нулевыми
аккумуляторами. Всё остальное остаётся прежним.

</details>

<details>
<summary>Ответ</summary>

Перед `let mut out = Vec::new();` в `aggregate`:

```rust
    // Without GROUP BY, an aggregate query always returns exactly one row.
    if query.group_by.is_empty() && groups.is_empty() {
        groups.insert(Vec::new(), (0, vec![0; sum_indexes.len()]));
    }
```

и тест:

```rust
    #[test]
    fn count_of_no_rows_is_zero() {
        let q = parse_query("SELECT COUNT(*), SUM(age) FROM people WHERE age > 100").unwrap();
        let out = execute(&q, &people()).unwrap();
        assert_eq!(out.rows, [[Value::Int(0), Value::Int(0)]]);
    }
```

Почему это работает: без `GROUP BY` у каждой строки пустой ключ, поэтому группа одна, если хоть одна строка
прошла, и ни одной в противном случае. Добавление пустой группы вручную даёт одну строку в обоих случаях.
С `GROUP BY` отсутствие входных строк правильно означает отсутствие групп. (Настоящий SQL возвращает здесь
`NULL` для `SUM`; проект добавляет `NULL`.)

</details>

## Вопросы для самопроверки

1. Почему `eval` возвращает `Result`, а не `Value`?

<details>
<summary>Ответ</summary>

Вычисление может завершиться ошибкой и на правильном SQL: неизвестный столбец или сравнение текста с числом.
Возврат ошибки позволяет движку сообщить о ней, а не паниковать и не давать неверный ответ.

</details>

2. Что такое ключ группы и чему он равен, когда нет `GROUP BY`?

<details>
<summary>Ответ</summary>

Значения столбцов `GROUP BY` для одной строки; строки с равными ключами принадлежат одной группе. Без
`GROUP BY` ключ пуст, поэтому все строки образуют одну группу.

</details>

3. Почему простой столбец в `SELECT` должен быть в `GROUP BY`, когда в запросе есть агрегаты?

<details>
<summary>Ответ</summary>

Каждая строка результата представляет целую группу. У столбца `GROUP BY` одно значение на группу; у
любого другого столбца в группе может быть много разных значений, и показать одно нечего.

</details>

4. Почему фильтр Arrow быстрее нашего построчного фильтра на больших данных?

<details>
<summary>Ответ</summary>

Он работает с целым столбцом, хранящимся одним простым буфером чисел, в одном плотном цикле без проверок
перечислений и без вызова функции на каждую строку. Это хорошо использует кэш процессора и позволяет
компилятору применять инструкции SIMD (Single Instruction, Multiple Data, «одна инструкция — много данных»).

</details>

5. Что содержит маска?

<details>
<summary>Ответ</summary>

По одному булеву значению на строку: `true` для строк, прошедших фильтр, и `false` для остальных.
`filter_record_batch` оставляет во всех столбцах строки, отмеченные `true`.

</details>

## Итоги

- Исполнитель превращает план в строки: сканирование, фильтр, затем проекция или агрегация.
- Выражения вычисляются рекурсивно для каждой строки; ошибки типов становятся `ExecError`, а не неверными ответами.
- Хеш-агрегация хранит по записи на ключ группы с накапливаемыми аккумуляторами и выдаёт строки только после
  чтения всего входа.
- Однократная замена имён столбцов позициями («связывание») избавляет от повторной работы на каждой строке.
- Вычислительные ядра Arrow делают тот же фильтр и сумму сразу над целыми столбцами: это векторизованная модель.
- Тестируйте исполнитель на маленьких таблицах в коде и сравнивайте с эталонным движком, например DataFusion.

## Источники и что почитать

- [Документация крейта `csv`](https://docs.rs/csv/1.4.0/csv/) — чтение и запись CSV, правила кавычек и руководство.
- [Документация крейта `arrow`](https://docs.rs/arrow/60.1.0/arrow/) — массивы, пакеты записей и ядра `compute`.
- [Вычислительные ядра Arrow — `cmp`](https://docs.rs/arrow/60.1.0/arrow/compute/kernels/cmp/index.html) — ядра сравнения вроде `gt_eq`.
- [DataFusion — Architecture](https://docs.rs/datafusion/55.3.0/datafusion/index.html) — раздел «Architecture» в документации крейта объясняет планы, `ExecutionPlan` и потоки пакетов записей.
- [CMU 15-445/645 Database Systems](https://15445.courses.cs.cmu.edu/) — бесплатный курс Университета Карнеги — Меллона (CMU, Carnegie Mellon University) с лекциями о выполнении запросов, соединениях и агрегации.

---

[🏠 Начало](../START-HERE.md) · [← 22.15 Трек баз данных: разбор SQL](15-databases-track-parsing-sql.md) · [📋 Этап 22](README.md) · [22.17 Трек баз данных: проект и следующие шаги →](17-databases-track-project-and-next-steps.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
