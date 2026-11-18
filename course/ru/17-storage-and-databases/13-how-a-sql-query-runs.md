<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 17.12 MVCC: многоверсионное управление конкурентным доступом](12-mvcc.md) · [📋 Этап 17](README.md) · [17.14 Проект: собственное хранилище «ключ-значение» →](14-project-key-value-store.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/17-storage-and-databases/13-how-a-sql-query-runs.md) <!-- nav -->

# Урок 17.13 — Как выполняется SQL-запрос

> **Этап 17 — Как устроены хранилища и базы данных** · Урок 13 из 15 · ⏱ около 135 минут

## Что вы узнаете

- Путь запроса внутри базы данных: парсер, анализатор, переписчик, планировщик и исполнитель.
- Как читать план из `EXPLAIN` и `EXPLAIN ANALYZE`: узлы, стоимости, ожидаемое и фактическое число строк.
- Как планировщик выбирает между последовательным сканированием и индексом по статистике и что происходит, когда
  статистика устарела.
- Как работают три способа соединения: вложенный цикл, хеш-соединение и соединение слиянием.
- Как исполнитель строится из итераторов (модель «Volcano») — на примере крошечного исполнителя на Rust.

## Перед началом

- Вы прошли [Урок 17.12 — MVCC: многоверсионное управление конкурентным доступом](12-mvcc.md).
- Вы знаете соединения таблиц и индексы из [Урока 16.5 — Основы SQL и PostgreSQL](../16-backend-services/05-sql-and-postgresql-basics.md),
  B-деревья из [Урока 17.6 — B-деревья](06-b-trees.md) и `EXPLAIN (ANALYZE, BUFFERS)` из
  [Урока 17.9 — Страничный кэш и пул буферов](09-page-cache-and-buffer-pool.md).
- Вы знаете итераторы, `map`, `filter` и `flat_map` (Этап 05) и хеш-таблицы из
  [Урока 07.6 — Как работают хеш-таблицы](../07-algorithms-and-data-structures/06-hash-tables.md).
- Контейнер `p17-postgres` запущен.

## Новые слова

| Слово | Значение |
|---|---|
| **парсер** (parser) | Часть, которая проверяет текст SQL (Structured Query Language, язык структурированных запросов) по грамматике и превращает его в дерево (*дерево разбора*, parse tree). |
| **анализатор** (analyzer) | Часть, которая находит каждое имя из дерева разбора (таблицы, столбцы, функции) и проверяет типы. |
| **переписчик** (rewriter) | Часть, которая применяет к дереву правила, например заменяет представление (view) его определением. |
| **планировщик** (planner, optimizer) | Часть, которая выбирает, *как* выполнять запрос: какие сканирования, какие способы соединения и в каком порядке. |
| **план** (plan) | Дерево операторов (*узлов*), которое выполняет исполнитель. Его печатает `EXPLAIN`. |
| **исполнитель** (executor) | Часть, которая выполняет план и выдаёт строки. |
| **стоимость** (cost) | Оценка планировщика, сколько работы потребует узел, в условных единицах (примерно «одна страница, прочитанная по порядку» = 1). |
| **селективность** (selectivity) | Доля строк, которую оставляет условие. У `id = 42` на большой таблице крошечная селективность; `amount > 0` оставляет почти всё. |
| **статистика** (statistics) | Сведения о каждом столбце, которые `ANALYZE` собирает по выборке: число различных значений, самые частые значения, гистограмма. |
| **соединение вложенным циклом** (nested loop join) | Для каждой строки одного входа просмотреть другой вход в поисках подходящих строк. |
| **хеш-соединение** (hash join) | Построить хеш-таблицу по меньшему входу, затем искать в ней каждую строку большего входа. |
| **соединение слиянием** (merge join) | Отсортировать оба входа по ключу соединения, затем пройти их бок о бок. |
| **модель Volcano** (iterator model, модель итераторов) | Устройство исполнителя, при котором у каждого узла одна операция — «дай следующую строку», и он вытягивает строки из своих детей. |

## 1. Путешествие запроса

Когда вы отправляете `SELECT c.name, o.amount FROM orders o JOIN customers c ON c.id = o.customer_id WHERE c.city = 'city-7'`,
текст проходит пять стадий:

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

Первые три стадии только понимают и проверяют запрос. Если вы опечатались в имени столбца, анализатор остановится с
`ERROR: column "nmae" does not exist`. Интересные решения принимает **планировщик**: SQL говорит, *что* вы хотите, но
никогда — *как* это получить, поэтому выбирать должна база данных. Для одного запроса могут существовать тысячи
возможных планов; планировщик оценивает стоимость многих из них и выбирает самый дешёвый.

## 2. Первый план

Создайте в `psql` (`docker exec -it p17-postgres psql -U postgres`) две таблицы: 10 000 клиентов в 50 городах и
200 000 заказов, затем соберите статистику через `ANALYZE`:

```sql
CREATE TABLE customers (id int PRIMARY KEY, name text NOT NULL, city text NOT NULL);
CREATE TABLE orders (id int PRIMARY KEY, customer_id int NOT NULL REFERENCES customers, amount int NOT NULL);
INSERT INTO customers SELECT i, 'customer-' || i, 'city-' || (i % 50) FROM generate_series(1, 10000) AS i;
INSERT INTO orders SELECT i, 1 + (i * 7919) % 10000, (i * 31) % 1000 FROM generate_series(1, 200000) AS i;
ANALYZE customers, orders;
```

`1 + (i * 7919) % 10000` равномерно распределяет заказы по клиентам: каждый клиент получает 20 заказов. Теперь
попросите план простого запроса и размер таблицы:

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

**Что происходит:**

1. В плане один узел, **Seq Scan**: прочитать каждую страницу `orders` и проверить каждую строку условием `Filter`.
   Индекса по `customer_id` нет, поэтому другого выбора нет.
2. `cost=0.00..3582.00`: первое число — стоимость до выдачи первой строки (стоимость *запуска*, startup), второе —
   стоимость выдачи всех строк. Единица — «прочитать одну страницу по порядку» (`seq_page_cost = 1`).
3. Откуда 3582? 1082 страницы × 1, плюс 200 000 строк × 0,01 (`cpu_tuple_cost`, обработка строки), плюс
   200 000 × 0,0025 (`cpu_operator_cost`, одно сравнение `=`): 1082 + 2000 + 500 = **3582**. Модель стоимости — простая
   арифметика с несколькими настройками.
4. `rows=20` — **оценка**: 200 000 строк, делённые примерно на 10 000 различных значений `customer_id`. `width=12` —
   ожидаемый размер одной строки результата в байтах.

`EXPLAIN` только планирует. `EXPLAIN ANALYZE` ещё и **выполняет** запрос и добавляет, что произошло на самом деле:

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

`actual time=0.113..4.456` — в миллисекундах (первая строка..последняя строка), `rows=20.00` — фактическое число строк
(оценка была верной), `loops=1` значит, что узел выполнился один раз. 199 980 строк были прочитаны и выброшены, со всех
1082 страниц.

> ⚠️ **Частая ошибка:** `EXPLAIN ANALYZE` действительно выполняет оператор. `EXPLAIN ANALYZE DELETE FROM orders`
> удаляет строки. Чтобы безопасно проанализировать пишущий оператор, оберните его: `BEGIN; EXPLAIN ANALYZE ...; ROLLBACK;`.

## 3. Индекс или нет: селективность и статистика

Создайте индекс и спросите снова:

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

**Что происходит:**

1. План теперь — дерево из двух узлов. Читайте его с **самого внутреннего** узла (с наибольшим отступом, помечен `->`)
   наружу: в таком порядке текут данные.
2. **Bitmap Index Scan** проходит B-дерево индекса (2 страницы) и собирает адреса 20 подходящих строк в битовую карту
   (bitmap): по биту на каждую страницу таблицы, где есть совпадение.
3. **Bitmap Heap Scan** затем читает только эти 20 страниц таблицы (`Heap Blocks: exact=20`) в порядке страниц и
   перепроверяет условие. («Heap», куча, — так PostgreSQL называет основное хранилище таблицы; это не куча из Этапа 03.)
4. Стоимость 76,54 вместо 3582, время 0,069 мс вместо 4,463 мс. Планировщик выбрал индекс, потому что предсказал мало
   строк.

Всегда ли индекс лучше? Сравните два условия на диапазон:

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

`customer_id > 100` оставляет 99% строк: условие почти совсем не **селективно**. Чтение 198 001 строки через индекс
прыгало бы по таблице и всё равно затронуло бы каждую страницу — медленнее, чем чтение страниц по порядку, поэтому
планировщик игнорирует индекс. `< 300` оставляет около 3%, и индекс выигрывает. Где-то посередине есть точка, в которой
выбор переключается; планировщик находит её с помощью модели стоимости.

Оценки (198 001 и 6279 строк) берутся из **статистики**, которую `ANALYZE` собрал по выборке из таблицы:

```sql
SELECT n_distinct FROM pg_stats WHERE tablename = 'orders' AND attname = 'customer_id';
```

```text
 n_distinct
------------
      10001
(1 row)
```

Настоящее число — 10 000; статистика — это оценка по выборке, и обычно её достаточно. `pg_stats` хранит ещё самые
частые значения и гистограмму каждого столбца — из них получаются оценки для `>` и `<`.

## 4. Три способа соединения

У планировщика три способа соединения. Для соединения `orders` (большая) с `customers` (маленькая):

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

Посмотрите, как выбирает планировщик. Соединение по одному городу и соединение для одного клиента:

```sql
EXPLAIN SELECT c.name, o.amount FROM orders o JOIN customers c ON c.id = o.customer_id WHERE c.city = 'city-7';
EXPLAIN SELECT c.name, o.amount FROM orders o JOIN customers c ON c.id = o.customer_id WHERE c.id = 42;
```

Два плана:

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

**Что происходит:**

1. Один город: 200 клиентов (каждый пятидесятый). Планировщик сканирует их, строит по ним **Hash** (маленькая сторона),
   затем пропускает мимо него все 200 000 заказов: это **Hash Join**. Его стоимость запуска 201,50 — цена построения
   хеш-таблицы до выдачи первой строки.
2. Один клиент: внешняя сторона — единственная строка, найденная по первичному ключу, а у внутренней стороны есть
   индекс. **Nested Loop**, который один раз выполняет поиск по внутреннему индексу, — самый дешёвый план.
3. Текст запросов одинаков, кроме `WHERE`. План зависит от **данных** — через оценки.

> ⚡ **Производительность:** чтобы ускорить медленный запрос, сначала прочитайте его `EXPLAIN (ANALYZE, BUFFERS)`:
> найдите узел с наибольшим `actual time`, сравните ожидаемое и фактическое число строк и посмотрите на `Buffers`
> (Урок 17.9). Угадывать, какой индекс добавить, не читая плана, — потеря времени и замедление каждой записи.

## 5. Когда статистика устарела

Обычно autovacuum сам запускает `ANALYZE` после достаточного числа изменений. Отключите его для этой таблицы, добавьте
100 000 заказов клиента 7 и посмотрите на оценку до и после ручного `ANALYZE`:

```sql
ALTER TABLE orders SET (autovacuum_enabled = false);
INSERT INTO orders SELECT i, 7, 500 FROM generate_series(200001, 300000) AS i;
EXPLAIN ANALYZE SELECT count(*) FROM orders WHERE customer_id = 7;
ANALYZE orders;
EXPLAIN ANALYZE SELECT count(*) FROM orders WHERE customer_id = 7;
```

Строка `Bitmap Heap Scan` двух планов:

```text
   ->  Bitmap Heap Scan on orders  (cost=4.53..112.67 rows=30 width=0) (actual time=1.400..6.671 rows=100020.00 loops=1)
   ->  Bitmap Heap Scan on orders  (cost=1141.70..4017.57 rows=100310 width=0) (actual time=1.007..6.194 rows=100020.00 loops=1)
```

До `ANALYZE` планировщик ожидал 30 строк и получил 100 020: он знал, что таблица выросла (он проверяет текущий размер),
но по-прежнему считал, что у каждого клиента одинаковое число заказов. После `ANALYZE` оценка близка к правде. Здесь
план остался тем же, потому что новые строки лежат вместе в конце таблицы. В соединении оценка, ошибочная в 3000 раз,
легко заставит планировщика выбрать вложенный цикл, который выполнится 100 000 раз: **неверные оценки — самая частая
причина медленных планов**. Когда ожидаемые и фактические `rows=` сильно расходятся, сначала подозревайте статистику.
Включите autovacuum обратно: `ALTER TABLE orders RESET (autovacuum_enabled);`.

## 6. Исполнитель из итераторов

Как исполнитель выполняет такое дерево? Большинство баз данных используют **модель Volcano**: каждый узел — объект с
одной операцией `next()`: «дай следующую строку (или ничего)». Узел вызывает `next()` у своих детей столько раз,
сколько ему нужно. Клиент вытягивает строки из корня, одну за другой.

Итераторы Rust работают ровно так: `Iterator::next` — это та самая операция, а `filter`, `map`, `flat_map` — узлы.
Соберём крошечный исполнитель:

```bash
cd ~/projects/rust
cargo new --lib code/p17-executor
cd code/p17-executor
```

Замените `src/lib.rs` на:

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

**Что происходит:**

1. `Row` — это `Vec<Value>`; столбец — позиция в нём. `Value` выводит `PartialOrd` и `Ord`, чтобы строки можно было
   сравнивать через `>` и сортировать. (Выведенная через derive реализация разрешает сравнивать `Int` с `Text`,
   упорядочивая варианты по их позиции; настоящая база данных отвергла бы его в анализаторе.)
2. `Rows<'a>` — итератор, упакованный в `Box`: «какой-то узел, выдающий строки». `Box<dyn Iterator>` позволяет соединять узлы
   разных типов во время выполнения, как это делает планировщик.
3. `scan` выдаёт копии строк таблицы; `filter` и `project` оборачивают свой вход и работают, только когда кто-то
   просит следующую строку. До этого ничего не происходит: итераторы **ленивые**.
4. У `hash_join` две фазы. Фаза *построения* (build) выполняется в самой функции: она читает весь вход `build` в
   `HashMap` из ключа в строки. Фаза *проверки* (probe) ленивая: `flat_map` превращает каждую строку probe в ноль, одну
   или много соединённых строк.

Теперь выполним запрос из раздела 4 на крошечных таблицах. Создайте `src/bin/query.rs`:

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

**Что происходит:**

1. План строится вложенными вызовами функций, от листьев к корню: сканирования, фильтры, хеш-соединение с маленькими
   отфильтрованными `customers` на стороне построения, затем проекция `c.name` (столбец 4 соединённой строки) и
   `o.amount` (столбец 2).
2. `inspect` считает каждую строку, которую выдаёт сканирование `orders`. `Cell` позволяет менять счётчик изнутри
   замыкания, пока `main` его ещё читает.
3. `result.take(2)` — это `LIMIT 2`. После двух выходных строк никто больше не вызывает `next()`, поэтому сканирование
   `orders` остановилось после 6 из 8 строк. В этом сила модели вытягивания: `LIMIT` останавливает всю работу под ним.
4. Сторона построения хеш-соединения прочитана полностью, потому что хеш-таблица должна быть готова до первой
   проверки. Это и есть стоимость запуска, которую вы видели у `Hash Join` в PostgreSQL.

> 🧪 **Тестирование:** операторы — маленькие чистые функции от своих входов, поэтому их легко тестировать на крошечных
> самодельных таблицах: пустой вход, нет совпадений, несколько совпадений для одного ключа. Упражнения 2 и 3 так
> тестируют соединения и проверяют два способа соединения друг против друга.

## Частые ошибки

> ⚠️ **Частая ошибка:** читать план сверху. Верхний узел — **последний** шаг. Данные текут от узлов с наибольшим
> отступом вверх.

> ⚠️ **Частая ошибка:** считать `cost` миллисекундами. Стоимости — единицы модели планировщика, полезные только для
> сравнения планов одного запроса. Настоящее время — в `actual time` у `EXPLAIN ANALYZE`.

> ⚠️ **Частая ошибка:** добавить индекс и ждать, что каждый запрос будет им пользоваться. Если условие оставляет
> большую часть таблицы, последовательное сканирование дешевле, и планировщик прав, игнорируя индекс (раздел 3).

## Попробуйте сами

1. Теперь, когда в `orders` 300 000 строк и свежая статистика, снова выполните `EXPLAIN` соединения по городу из
   раздела 4. План сменился на `Nested Loop` с `Index Scan using orders_customer_id_idx` (стоимость 5261.13): тот же
   запрос, новые данные, новый план. Затем выполните `SET enable_nestloop = off;` и ещё раз `EXPLAIN`: планировщику
   придётся вернуться к `Hash Join`, который он теперь оценивает в 5611.33. `RESET enable_nestloop;` включает всё
   обратно. Эти переключатели существуют для таких экспериментов, а не для продакшена.
2. В `query.rs` замените `take(2)` на `take(10)`. Будут прочитаны все 8 заказов, а на выходе окажутся только 3 строки.

## Упражнения

### Упражнение 1 — Прочитайте план (легко)

В плане `Hash Join` из раздела 4: какой узел выдаёт первые строки, какие узлы выполняются полностью, прежде чем
соединение сможет что-то выдать, и сколько строк планировщик ожидал от соединения?

<details>
<summary>Подсказка</summary>

Начните с узла с наибольшим отступом. Узел `Hash` должен прочитать весь свой вход, прежде чем будет готов.

</details>

<details>
<summary>Ответ</summary>

Первые строки выдаёт `Seq Scan on customers c` (с `Filter` по городу). Он и узел `Hash` над ним выполняются полностью
первыми: хеш-таблица примерно из 200 клиентов должна быть готова. Затем `Seq Scan on orders o` пропускает 200 000 строк
через `Hash Join`, от которого ожидалось `rows=4000` (200 клиентов × 20 заказов).

Почему это работает: стоимость запуска `Hash Join` (201,50) примерно равна полной стоимости `Hash` (199,00) — это
показывает, что сторона построения выполняется до первой выходной строки.

</details>

### Упражнение 2 — Протестируйте хеш-соединение (средне)

Напишите тест `hash_join_repeats_matches_and_drops_rows_without_a_partner`: два клиента, три заказа, из которых два
принадлежат клиенту 1, а один — клиенту 3, которого не существует. Соединение должно вернуть две строки, обе для
клиента 1.

<details>
<summary>Подсказка</summary>

Вспомогательная функция `fn row(values: &[i64]) -> Row`, превращающая `&[1, 10]` в строку из `Value::Int`, делает тест
коротким.

</details>

<details>
<summary>Ответ</summary>

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

Почему это работает: внутреннее соединение возвращает по строке на каждую подходящую пару; у заказа без подходящего
клиента пары нет, поэтому он исчезает; клиент с двумя заказами появляется дважды.

</details>

### Упражнение 3 — Соединение вложенным циклом (сложно)

Добавьте `pub fn nested_loop_join<'a>(outer: Rows<'a>, outer_key: usize, inner: &'a [Row], inner_key: usize) -> Rows<'a>`:
для каждой внешней строки просмотрите весь срез `inner`. Затем напишите тест, который соединяет одни и те же данные
обоими способами и проверяет, что после сортировки результаты равны.

<details>
<summary>Подсказка</summary>

Внутреннюю сторону нужно читать заново для каждой внешней строки, поэтому это срез, а не итератор (итератор можно
прочитать только один раз). Соберите совпадения одной внешней строки в `Vec` и верните его из `flat_map`. Два
соединения располагают столбцы в разном порядке, если не выбрать стороны аккуратно: используйте `orders` как сторону
проверки в хеш-соединении и как внешнюю сторону во вложенном цикле. Тест использует вспомогательную функцию `row` из
Упражнения 2.

</details>

<details>
<summary>Ответ</summary>

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

Почему это работает: оба способа выдают каждую подходящую пару ровно один раз, только в разном порядке, поэтому
отсортированные результаты должны совпадать. Проверка одной реализации другой (простой и очевидно правильной — против
быстрой) — сильный приём. Вложенный цикл делает `outer × inner` сравнений, здесь 20 × 5; хеш-соединение — около
`outer + inner` операций, поэтому планировщик предпочитает его для больших входов.

</details>

## Вопросы для самопроверки

1. Какая стадия обработки запроса сообщит `ERROR: column "nmae" does not exist`, а какая выбирает между
   последовательным сканированием и индексом?

<details>
<summary>Ответ</summary>

О неизвестных именах сообщает анализатор. Сканирования и соединения выбирает планировщик.

</details>

2. Что означают два числа в `cost=4.45..76.54`?

<details>
<summary>Ответ</summary>

Ожидаемую стоимость запуска (до первой строки) и ожидаемую полную стоимость (за все строки) в единицах планировщика,
где последовательное чтение одной страницы стоит 1.

</details>

3. Почему PostgreSQL игнорирует индекс для `customer_id > 100`?

<details>
<summary>Ответ</summary>

Условие оставляет около 99% строк. Чтение их через индекс всё равно затронуло бы каждую страницу, да ещё в худшем
порядке, поэтому последовательное сканирование дешевле.

</details>

4. `EXPLAIN ANALYZE` показывает ожидаемые `rows=30` и фактические `rows=100020`. Что проверить первым делом?

<details>
<summary>Ответ</summary>

Статистику: выполнить `ANALYZE` для таблицы (и выяснить, почему этого не сделал autovacuum). Неверные оценки ведут к
неверным планам.

</details>

5. Почему в модели Volcano `LIMIT 2` удешевляет весь запрос?

<details>
<summary>Ответ</summary>

Каждый узел выдаёт строки, только когда родитель просит следующую. После двух строк корень перестаёт просить, и узлы
под ним тоже перестают работать.

</details>

## Итоги

- Запрос проходит парсер, анализатор, переписчик, планировщик и исполнитель; планировщик выбирает, *как* его выполнять.
- План — дерево; читайте его от узла с наибольшим отступом вверх. `EXPLAIN` показывает оценки, `EXPLAIN ANALYZE`
  выполняет запрос и добавляет фактические времена и строки.
- Стоимости — единицы модели, построенные из настроек вроде `seq_page_cost` и `cpu_tuple_cost`.
- Планировщик оценивает селективность по статистике: индекс — для немногих строк, последовательное сканирование — для
  многих. Устаревшая статистика даёт неверные оценки; `ANALYZE` её обновляет.
- Соединения: вложенный цикл (маленькая внешняя сторона, индекс на внутренней), хеш-соединение (большие входы),
  соединение слиянием (отсортированные входы).
- Исполнители вытягивают строки через дерево итераторов; `Iterator` в Rust — ровно эта модель, а ленивость делает
  `LIMIT` дешёвым.

## Источники и что почитать

- [PostgreSQL: The Path of a Query](https://www.postgresql.org/docs/current/query-path.html) — стадии из раздела 1 со ссылками на каждую.
- [PostgreSQL: Using EXPLAIN](https://www.postgresql.org/docs/current/using-explain.html) — как читать планы, с множеством примеров.
- [PostgreSQL: Statistics Used by the Planner](https://www.postgresql.org/docs/current/planner-stats.html) и [Row Estimation Examples](https://www.postgresql.org/docs/current/row-estimation-examples.html) — откуда берётся `rows=`.
- [PostgreSQL: Query Planning settings](https://www.postgresql.org/docs/current/runtime-config-query.html) — `seq_page_cost`, `cpu_tuple_cost`, `enable_hashjoin` и другие.
- [SQLite: The SQLite Query Optimizer Overview](https://www.sqlite.org/queryplanner.html) — те же идеи на примере другой базы данных, очень понятно.
- [CMU 15-445/645 Database Systems](https://15445.courses.cs.cmu.edu/) — CMU — Carnegie Mellon University (Университет Карнеги — Меллона); лекции о выполнении и оптимизации запросов.

---

[🏠 Начало](../START-HERE.md) · [← 17.12 MVCC: многоверсионное управление конкурентным доступом](12-mvcc.md) · [📋 Этап 17](README.md) · [17.14 Проект: собственное хранилище «ключ-значение» →](14-project-key-value-store.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
