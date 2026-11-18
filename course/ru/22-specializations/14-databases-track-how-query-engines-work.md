<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 22.13 Трек встраиваемых систем: проект и следующие шаги](13-embedded-track-project-and-next-steps.md) · [📋 Этап 22](README.md) · [22.15 Трек баз данных: разбор SQL →](15-databases-track-parsing-sql.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/22-specializations/14-databases-track-how-query-engines-work.md) <!-- nav -->

# Урок 22.14 — Трек баз данных: как работают движки запросов

> **Этап 22 — Специализации** · Урок 14 из 30 · ⏱ около 120 минут

> 🧭 **Не ваше направление?** Этот урок открывает направление баз данных (Уроки 22.14–22.17). Если вы
> выбрали другое направление, переходите к [Уроку 22.18 — Трек сетей: высокопроизводительные серверы](18-networking-track-high-performance-servers.md)
> или сразу к [контрольной точке, Уроку 22.30](30-checkpoint-your-specialization.md).

## Что вы узнаете

- Что такое движок запросов и через какие стадии проходит запрос от текста SQL (Structured Query Language, язык структурированных запросов) до результата: разбор, план, оптимизация, выполнение.
- Что такое логический и физический планы и как реляционные операторы образуют дерево.
- Модель выполнения через итераторы («Volcano») и почему итераторы Rust уже работают именно так.
- Почему аналитические движки хранят данные по столбцам и что такое Apache Arrow.
- Как выполнить SQL над CSV-файлом (Comma-Separated Values, значения, разделённые запятыми) с помощью Apache DataFusion и прочитать его планы через `EXPLAIN`.

## Перед началом

- Вы прошли [Урок 22.1 — Выбор специализации](01-choosing-a-specialization.md) и выбрали это направление.
- Вы знаете SQL по [Уроку 16.5 — Основы SQL и PostgreSQL](../16-backend-services/05-sql-and-postgresql-basics.md)
  и общую картину из [Урока 17.13 — Как выполняется SQL-запрос](../17-storage-and-databases/13-how-a-sql-query-runs.md).
  В этом направлении вы сами строите «запросную» половину базы данных; Этап 17 разбирал половину хранения.
- Вы знаете итераторы ([Урок 05.5](../05-rust-intermediate/05-iterators.md)), деревья
  ([Урок 07.11](../07-algorithms-and-data-structures/11-trees-and-binary-search-trees.md)), код, дружественный
  к кэшу ([Урок 12.10](../12-computer-systems/10-cache-friendly-code.md)), и основы Tokio
  ([Урок 14.13](../14-concurrency-and-async/13-tokio-tasks-and-channels.md)).

**Где лежит код.** Каждый урок этого направления использует свой пакет в `code/p22-<name>`. Проект
направления в Уроке 22.17 — `code/p22-minisql`.

## Новые слова

| Слово | Значение |
|---|---|
| **движок запросов** (query engine) | Часть базы данных (или библиотека), которая принимает запрос, планирует его и вычисляет результат. |
| **SQL** (Structured Query Language, язык структурированных запросов) | Стандартный язык для вопросов к таблицам. |
| **CSV** (Comma-Separated Values, значения, разделённые запятыми) | Текстовый файл, где каждая строка — запись, а столбцы разделены запятыми. |
| **реляционный оператор** (relational operator) | Один базовый шаг над таблицами: сканирование, фильтр, проекция, агрегация, сортировка, соединение, ограничение. |
| **проекция** (projection) | Оставить только некоторые столбцы (часть `SELECT name, city`). |
| **агрегация** (aggregation) | Превращение многих строк в одно значение на группу: `COUNT`, `SUM`, `AVG` с `GROUP BY`. |
| **логический план** (logical plan) | Дерево реляционных операторов, которое говорит, *что* вычислить. |
| **физический план** (physical plan) | Дерево конкретных алгоритмов, которое говорит, *как* это вычислить. |
| **оптимизатор** (optimizer) | Стадия, которая переписывает план в более дешёвый с тем же результатом. |
| **модель Volcano** (Volcano model) | Выполнение, при котором у каждого оператора есть метод `next()`, вытягивающий одну строку из дочернего оператора. |
| **векторизованное выполнение** (vectorized execution) | Выполнение, при котором операторы обрабатывают пакет из тысяч значений за раз. |
| **столбцовый формат** (columnar format) | Хранение значений каждого столбца рядом друг с другом в памяти, а не значений каждой строки. |
| **Apache Arrow** | Стандартный столбцовый формат данных в памяти с библиотеками для многих языков (крейт для Rust — `arrow`). |
| **Apache DataFusion** | Движок запросов на Rust, построенный на Arrow. Другие системы встраивают его к себе. |
| **пакет записей** (record batch) | Единица данных Arrow: схема плюс по одному массиву на столбец, все с одинаковым числом строк. |

## 1. Что делает движок запросов

У базы данных две большие половины. **Движок хранения** (storage engine) бережно хранит байты на диске:
страницы, B-деревья, журнал упреждающей записи (Этап 17). **Движок запросов** отвечает на вопросы о
данных. Вы даёте ему текст SQL:

```sql
SELECT city, COUNT(*) AS n, AVG(age) AS avg_age
FROM people
WHERE age >= 30
GROUP BY city
ORDER BY city
```

и он возвращает таблицу. Внутри запрос проходит стадии:

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

AST (Abstract Syntax Tree, абстрактное синтаксическое дерево) — структура текста в виде дерева значений
Rust. Многие современные системы (аналитические базы данных, потоковые обработчики, инструменты для
данных) собраны именно из этих частей, и в Rust эти части существуют как крейты: `sqlparser` для шага 1 и
DataFusion для шагов 2–5.

## 2. Планы — это деревья операторов

Любой SQL-запрос можно записать несколькими **реляционными операторами**. Каждый принимает одну или две
таблицы (потоки строк) и выдаёт новую таблицу:

| Оператор | Часть SQL | Что делает |
|---|---|---|
| Scan (сканирование) | `FROM people` | читает все строки таблицы |
| Filter (фильтр) | `WHERE age >= 30` | оставляет строки, для которых условие истинно |
| Projection (проекция) | `SELECT city, …` | оставляет или вычисляет некоторые столбцы |
| Aggregate (агрегация) | `GROUP BY city` + `COUNT(*)`, `AVG(age)` | одна выходная строка на группу |
| Sort (сортировка) | `ORDER BY city` | упорядочивает строки |
| Limit (ограничение) | `LIMIT 10` | останавливается после N строк |
| Join (соединение) | `JOIN … ON …` | объединяет строки двух таблиц |

Запрос из раздела 1 превращается в **дерево**. Данные текут от листа (сканирования) вверх к корню:

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

Это **логический план**: он говорит, *что* является результатом, но не *как* его вычислить.
**Физический план** выбирает алгоритмы: например, агрегировать хеш-таблицей или сначала сортировкой;
читать CSV-файл одним потоком или многими.

Между ними **оптимизатор** переписывает логический план. Два классических правила:

- **Проталкивание проекции** (projection pushdown): запрос использует только `age` и `city`, поэтому
  сканирование читает только эти два столбца.
- **Проталкивание предикатов** (predicate pushdown; предикат — это условие): перемещать фильтры как можно
  ближе к сканированию, чтобы вверх по дереву шло меньше строк. Некоторые форматы файлов позволяют даже
  пропускать целые блоки данных, которые не могут подойти.

## 3. Модель итераторов («Volcano») на обычном Rust

Как операторы передают друг другу строки? Классический ответ (из исследовательской системы Volcano
начала 1990-х): у каждого оператора есть метод `next()`. Корень вызывает `next()` у дочернего оператора,
тот — у своего дочернего, и так до сканирования. Каждый вызов **вытягивает** одну строку вверх по дереву.

Эту модель вы уже знаете: именно так работают итераторы Rust. Создайте `code/p22-pipeline` командой
`cargo new code/p22-pipeline` и поместите в `src/main.rs`:

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

**Что происходит:**

1. `people.iter()` — это **сканирование**, `.filter(...)` — **фильтр**, `.map(...)` — **проекция**.
2. `.collect()` — корень: он снова и снова вызывает `next()`. Каждый вызов проходит вниз по цепочке до
   сканирования и приносит одну строку наверх.
3. `inspect` только печатает то, что проходит мимо, чтобы можно было наблюдать порядок работы.

```text
scan    Ana
  keep  Ana
scan    Ben
scan    Chen
  keep  Chen
scan    Dana
result: [("Ana", "Lisbon"), ("Chen", "Lisbon")]
```

Строки обрабатываются не «сначала все сканирования, потом все фильтры». Каждая строка проходит весь
конвейер, прежде чем читается следующая. Это **конвейерная обработка** (pipelining): ни одному оператору не
нужно держать всю таблицу в памяти. Некоторые операторы не могут работать конвейерно: сортировка или
агрегация должны увидеть *все* входные строки, прежде чем выдадут первую строку результата. Их называют
**разрывами конвейера** (pipeline breakers).

## 4. Строки или столбцы?

Модель Volcano передаёт одну строку за вызов `next()`. Для аналитических запросов по миллионам строк это
медленно: по вызову функции (часто динамическому, через трейт-объект) на строку на оператор, и процессор
(CPU, Central Processing Unit) всё время перескакивает между разными видами работы.

Современные аналитические движки делают два изменения:

1. **Столбцовое размещение.** Значения каждого столбца хранятся рядом друг с другом:

```text
Row layout (array of structs)          Columnar layout (struct of arrays)

[Ana | 34 | Lisbon]                    names:  [Ana, Ben, Chen, Dana]
[Ben | 27 | Oslo  ]                    ages:   [ 34,  27,   41,   19]
[Chen| 41 | Lisbon]                    cities: [Lisbon, Oslo, Lisbon, Oslo]
[Dana| 19 | Oslo  ]
```

   `AVG(age)` теперь читает один компактный массив целых чисел. Кэш процессора заполняется только полезными
   данными ([Урок 12.10](../12-computer-systems/10-cache-friendly-code.md)), а компилятор может использовать
   SIMD (Single Instruction, Multiple Data, «одна инструкция — много данных»: инструкция, которая работает
   сразу с несколькими числами).

2. **Векторизованное выполнение.** Операторы передают **пакеты** (например, по 8 192 строки) вместо
   отдельных строк. Фильтр вычисляет целый массив значений «истина/ложь» в одном плотном цикле, а затем
   оставляет подходящие строки. Накладные расходы на строку почти исчезают.

**Apache Arrow** — стандарт для таких столбцовых данных в памяти. **Пакет записей** Arrow — это схема
(имена и типы столбцов) плюс по массиву на столбец. Каждый массив — буфер значений плюс **битовая карта
валидности** (validity bitmap): по биту на строку, который говорит, есть ли значение или это `NULL`.
Поскольку формат одинаков во всех языках и инструментах, системы могут передавать данные друг другу без
преобразования.

> ⚡ **Производительность:** строковое размещение лучше, когда вы читаете или пишете целые строки по одной
> (интернет-магазин обновляет один заказ). Столбцовое лучше, когда вы сканируете несколько столбцов многих
> строк (отчёт за год заказов). В этом разница между базами OLTP (Online Transaction Processing, оперативная
> обработка транзакций) и OLAP (Online Analytical Processing, оперативная аналитическая обработка).

## 5. Настоящий движок: Apache DataFusion

**DataFusion** — полноценный движок запросов на Rust, построенный на Arrow. У него есть фронтенд SQL (он
использует крейт `sqlparser` из Урока 22.15), логический планировщик, оптимизатор, параллельный
векторизованный исполнитель и чтение CSV, JSON (JavaScript Object Notation, нотация объектов JavaScript),
Parquet (столбцовый формат файлов) и другого. Базы данных, потоковые обработчики и инструменты для данных
встраивают его вместо того, чтобы писать свой движок.

Создайте пакет и небольшой CSV-файл:

```bash
cd ~/projects/rust
cargo new code/p22-datafusion
cd code/p22-datafusion
cargo add datafusion@55
cargo add tokio@1 --features rt-multi-thread,macros
```

`people.csv` (в папке пакета, рядом с `Cargo.toml`):

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

**Что происходит:**

1. `SessionContext` хранит каталог (какие таблицы существуют) и настройки.
2. `register_csv` регистрирует файл как таблицу `people`. DataFusion читает заголовок и образец строк,
   чтобы вывести схему: `name` — текст, `age` — 64-битное целое, `city` — текст.
3. `ctx.sql(...)` разбирает, планирует и оптимизирует запрос и возвращает `DataFrame` — план, который ещё
   не выполнялся. `show()` выполняет его и печатает результат таблицей.
4. DataFusion асинхронный, потому что чтение файлов (или объектного хранилища по сети) — это ввод-вывод
   (I/O, input/output); `#[tokio::main]` даёт рантайм.

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

Проверьте вручную: люди от 30 лет — это Ana 34 и Chen 41 (Lisbon, среднее 37,5), Fay 30 (Oslo), Eli 52 и
Gus 45 (Tartu, среднее 48,5).

> 💡 **Совет:** DataFusion — большой крейт. Первая сборка скомпилировала 245 крейтов и заняла 3 мин 39 с
> при двух заданиях сборки, а папка `target` выросла до нескольких гигабайт. На macOS отладочная сборка
> может ещё напечатать безобидное предупреждение компоновщика `__eh_frame section too large`. Следующие
> сборки быстрые.

### Читаем планы через `EXPLAIN`

Поставьте `EXPLAIN` перед запросом, чтобы увидеть планы вместо результата. Сохраните это как
`examples/explain.rs` и выполните `cargo run --example explain`:

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

Вывод — таблица из двух строк; здесь она показана без рамок, чтобы поместилась:

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

**Что происходит:**

1. **Логический план** — это дерево из раздела 2, напечатанное от корня; отступы показывают дочерние узлы.
2. `TableScan: people projection=[age, city]` — это **проталкивание проекции**: оптимизатор увидел, что
   `name` нигде не используется. `partial_filters=[...]` показывает, что фильтр тоже предложили
   сканированию (**проталкивание предикатов**); для CSV сканирование не может им воспользоваться, поэтому
   `Filter` остаётся.
3. `COUNT(*)` превратился в `count(Int64(1))` (посчитать константу 1 для каждой строки), а `AVG(age)`
   получил приведение к `Float64`: планировщик выбрал точные типы.
4. В **физическом плане** каждый узел — алгоритм `...Exec`. `city@0` означает «столбец номер 0 входного
   пакета»: имена заменены позициями ещё до выполнения. `AggregateExec` использует хеш-таблицу.
5. С настройками по умолчанию DataFusion делит работу на **разделы** (partitions), по одному на ядро
   процессора, и добавляет узлы `RepartitionExec`, чтобы перемещать строки между потоками. Попробуйте:
   уберите строки с `config` и запустите снова.

> 🧪 **Тестирование:** зрелый движок вроде DataFusion — отличный **тестовый оракул** (test oracle):
> выполните один и тот же запрос в своём движке и в DataFusion и сравните результаты. Это
> *дифференциальное тестирование*, и проект направления его использует.

## Частые ошибки

> ⚠️ **Частая ошибка: забыть, что `DataFrame` ленивый.** `let df = ctx.sql(sql).await?;` только планирует
> запрос. Ничего не читается, пока вы не вызовете `show()`, `collect()` или похожий метод. Если засекать
> время только для `ctx.sql`, вы измеряете планирование, а не выполнение.

> ⚠️ **Частая ошибка: относительный путь не из той папки.** `register_csv("people", "people.csv", …)` ищет
> файл относительно **текущей папки** процесса. Запускайте `cargo run` в папке пакета или используйте
> полный путь. Если запустить из `src/`, программа остановится с:
>
> ```text
> Error: Plan("No files found at file:///Users/you/projects/rust/code/p22-datafusion/src/people.csv. Cannot infer schema from an empty location; either add data files or declare an explicit schema for the table.")
> ```

> ⚠️ **Частая ошибка: смешение версий `arrow`.** DataFusion 55 использует собственную версию крейта `arrow`
> (на момент написания 59.x), а самый новый `arrow` на crates.io может быть новее. Если добавить `arrow`
> отдельно в другой версии, типы не совпадут (`expected RecordBatch, found RecordBatch`). Используйте
> реэкспорт `datafusion::arrow`.

## Попробуйте сами

1. Измените фильтр на `age >= 40` и предскажите результат до запуска.
2. Добавьте `LIMIT 2` в конец запроса в `examples/explain.rs`. Нового узла не появится: оптимизатор
   сливает ограничение с сортировкой (`Sort: … fetch=2`), а физический план использует
   `SortExec: TopK(fetch=2)`, который хранит только 2 лучшие строки вместо сортировки всего.
3. Запустите `examples/explain.rs` без строк с `config` и посчитайте узлы `RepartitionExec`.

## Упражнения

### Упражнение 1 — Нарисуйте план (легко)

Нарисуйте дерево логического плана для
`SELECT name FROM people WHERE city = 'Oslo' ORDER BY name LIMIT 1`. Какой оператор — корень и какие
операторы — разрывы конвейера?

<details>
<summary>Подсказка</summary>

Начните с листа: что должно случиться первым? Затем читайте части SQL в том порядке, в котором они нужны
движку: `FROM`, `WHERE`, `SELECT`, `ORDER BY`, `LIMIT`.

</details>

<details>
<summary>Ответ</summary>

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

Корень — `Limit`. `Sort` — разрыв конвейера: он должен прочитать все строки, прежде чем узнает первую.
Оптимизатор DataFusion идёт на шаг дальше и сливает их: его логический план показывает
`Sort: people.name ASC NULLS LAST, fetch=1`, а физический план использует `SortExec: TopK(fetch=1)` —
оператор «top-N», который во время чтения хранит только лучшую строку.

</details>

### Упражнение 2 — Посмотрите на проталкивание (средне)

Используйте `EXPLAIN` в `code/p22-datafusion` для `SELECT name FROM people WHERE age > 40`. Какие столбцы
читает `TableScan` и почему не только `name`?

<details>
<summary>Подсказка</summary>

Сканирование должно выдать каждый столбец, который нужен любому оператору над ним, а не только столбцы из `SELECT`.

</details>

<details>
<summary>Ответ</summary>

Сканирование читает `projection=[name, age]`: `name` для вывода и `age` для фильтра. `city` в память вообще
не читается. Проталкивание проекции оставляет столбцы, которые использует *любой* оператор, и отбрасывает остальные.

</details>

### Упражнение 3 — Столбцы и тест (сложно)

Создайте библиотеку `code/p22-columns` со структурой `Person` (как в разделе 3, с полями `pub`), структурой
`PeopleColumns` с тремя полями `Vec`, функцией `pub fn to_columns(rows: &[Person]) -> PeopleColumns` и
функцией `pub fn average_age(cols: &PeopleColumns) -> Option<f64>`, которая читает только столбец `ages` и
возвращает `None` для пустой таблицы. **Напишите тест** для двух строк и для пустой таблицы.

<details>
<summary>Подсказка</summary>

Выведите `Default` для `PeopleColumns`, чтобы начать с трёх пустых векторов. Складывайте возрасты как `u64`,
чтобы избежать переполнения, затем делите как `f64`.

</details>

<details>
<summary>Ответ</summary>

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

Почему это работает: `average_age` трогает только один `Vec<u32>` — единый компактный блок памяти, в этом и
состоит идея столбцового хранения. Для пустого случая возвращается `None`, так же как `AVG` по пустому
набору строк возвращает `NULL` в SQL.

</details>

## Вопросы для самопроверки

1. Назовите стадии, которые проходит запрос внутри движка запросов.

<details>
<summary>Ответ</summary>

Разбор (текст → синтаксическое дерево), планирование (→ логический план), оптимизация (→ более дешёвый
логический план), физическое планирование (→ физический план с конкретными алгоритмами), выполнение (→ результат).

</details>

2. Чем логический план отличается от физического?

<details>
<summary>Ответ</summary>

Логический план говорит, *что* вычислить, через реляционные операторы (фильтр, агрегация, …). Физический —
*как*: какой алгоритм, какие позиции столбцов, сколько разделов и потоков.

</details>

3. Почему цепочка итераторов Rust — пример модели Volcano?

<details>
<summary>Ответ</summary>

У каждого адаптера (`filter`, `map`) есть `next()`, который вытягивает один элемент из адаптера под ним.
Потребитель в конце (`collect`) управляет всем, многократно вызывая `next()`.

</details>

4. Что такое разрыв конвейера? Приведите два примера.

<details>
<summary>Ответ</summary>

Оператор, который должен прочитать весь свой вход, прежде чем выдать результат, поэтому строки не могут
проходить через него по одной. Примеры: сортировка, агрегация (и сторона построения в хеш-соединении).

</details>

5. Почему аналитические движки предпочитают столбцовый формат?

<details>
<summary>Ответ</summary>

Запросы обычно читают несколько столбцов многих строк. В столбцовом размещении эти значения упакованы
вместе, поэтому движок читает меньше памяти, хорошо использует кэш процессора и может обрабатывать пакеты
плотными циклами и SIMD.

</details>

6. Что делают проталкивание проекции и проталкивание предикатов?

<details>
<summary>Ответ</summary>

Проталкивание проекции заставляет сканирование читать только используемые столбцы. Проталкивание
предикатов перемещает фильтры вниз, к сканированию (или внутрь него), чтобы по плану шло меньше строк.

</details>

## Итоги

- Движок запросов превращает SQL в результат по стадиям: разбор, логический план, оптимизация, физический план, выполнение.
- Планы — деревья реляционных операторов: сканирование, фильтр, проекция, агрегация, сортировка, ограничение, соединение.
- Модель Volcano вытягивает по одной строке через `next()`; итераторы Rust работают ровно так.
- Сортировка и агрегация — разрывы конвейера; фильтр и проекция работают конвейерно.
- Аналитические движки используют столбцовые данные и векторизованные пакеты; Apache Arrow — стандартный формат в памяти.
- Apache DataFusion — полноценный движок запросов на Rust; `EXPLAIN` показывает его логический и
  физический планы, включая проталкивание проекции и предикатов.

## Источники и что почитать

- [Документация Apache DataFusion](https://datafusion.apache.org/) — руководство пользователя и архитектура движка.
- [DataFusion: `EXPLAIN` и чтение планов](https://datafusion.apache.org/user-guide/explain-usage.html) — как читать логические и физические планы.
- [Apache Arrow columnar format](https://arrow.apache.org/docs/format/Columnar.html) — официальное описание массивов, буферов и битовых карт валидности.
- [`datafusion` на docs.rs](https://docs.rs/datafusion/55.3.0/datafusion/) — `SessionContext`, `DataFrame` и остальной API (Application Programming Interface, программный интерфейс).
- [CMU 15-445/645 Database Systems](https://15445.courses.cs.cmu.edu/) — бесплатный курс Университета Карнеги — Меллона (CMU, Carnegie Mellon University) с лекциями о выполнении и оптимизации запросов.
- [Volcano — An Extensible and Parallel Query Evaluation System (Graefe, 1994)](https://doi.org/10.1109/69.273032) — статья, из которой пришла модель итераторов.

---

[🏠 Начало](../START-HERE.md) · [← 22.13 Трек встраиваемых систем: проект и следующие шаги](13-embedded-track-project-and-next-steps.md) · [📋 Этап 22](README.md) · [22.15 Трек баз данных: разбор SQL →](15-databases-track-parsing-sql.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
