<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 08.15 Приложение двенадцати факторов](15-the-twelve-factor-app.md) · [📋 Этап 08](README.md) · [08.17 Контрольная точка →](17-checkpoint.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/08-software-design/16-project-refactor-a-messy-program.md) <!-- nav -->

# Урок 08.16 — Проект: рефакторинг запутанной программы

> **Этап 08 — Проектирование ПО: принципы, паттерны и соглашения** · Урок 16 из 17 · ⏱ около 6 часов (3 занятия)

## Что вы узнаете

- Как защитить программу без тестов с помощью *golden-тестов*, которые запускают настоящий бинарник.
- Как менять структуру программы на 180 строк малыми шагами, пока её вывод остаётся побайтово тем же.
- Как применить уроки этого этапа вместе: разделение ответственности, слои, типы вместо строк, ввод-вывод за трейтами, небольшие функции и модульные тесты.
- Как найти, записать и осознанно сохранить (пока) ошибки, обнаруженные во время рефакторинга.

## Перед началом

- Вы прошли [Урок 08.15 — Приложение двенадцати факторов](15-the-twelve-factor-app.md) и, в частности, [Урок 08.13 — Рефакторинг и код-ревью](13-refactoring-and-code-review.md): вы будете постоянно использовать его цикл (изменить, проверить, закоммитить).
- Вы умеете тестировать программу командной строки снаружи через `std::process::Command` и `env!("CARGO_BIN_EXE_<name>")` ([Урок 05.13](../05-rust-intermediate/13-project-minigrep.md), веха M5). Этап 09 объясняет это подробно.
- Вы знаете модули ([Урок 04.6](../04-structs-enums-collections-errors/06-modules-crates-and-packages.md)), newtype и enum как типы ([Урок 08.5](05-type-driven-design.md)), трейты `Read`/`Write` ([Урок 08.3](03-solid-principles.md), инверсия зависимостей) и слои ([Урок 08.14](14-layers-and-hexagonal-architecture.md)).
- Работайте в ветке: `git switch -c project/p08-refactor`.

## Новые слова

| Слово | Значение |
|---|---|
| **golden-тест** (golden test) | Тест, который запускает программу на фиксированном входе и сравнивает весь вывод с сохранённым файлом («золотым» файлом). Другое название характеризационного теста для программы, печатающей текст. |
| **golden-файл** (golden file) | Сохранённый ожидаемый вывод. |
| **совместимость по ошибкам** (bug-compatible) | Сохранение известной ошибки в точности как есть во время рефакторинга, чтобы изменение структуры и исправление ошибки были двумя отдельными, проверяемыми шагами. |
| **странность** (quirk) | Неожиданное, но существующее поведение, например правило округления или особенность форматирования. |
| **stdin / stdout / stderr** | Стандартный ввод, стандартный вывод и стандартный поток ошибок: три потока, которые есть у каждой программы. Ввод приходит из stdin; обычный вывод идёт в stdout; сообщения об ошибках — в stderr. |
| **код возврата** (exit status) | Число, которое программа возвращает командной оболочке при завершении: `0` означает успех. |
| **CSV** (Comma-Separated Values, значения, разделённые запятыми) | Текстовый формат с одной записью в строке и полями, разделёнными запятыми. |
| **I/O** (Input/Output, ввод-вывод) | Всё, где программа общается с внешним миром: чтение файлов, клавиатура, печать. |

## Знакомство с запутанной программой

У небольшого магазина есть программа `till`, которая читает заказы из стандартного ввода и печатает отчёт. Она работает. Никто не решается её трогать. Ваша задача — сделать её чистой, **не меняя того, что она печатает**.

Создайте пакет и добавьте вторую часть `Cargo.toml` (бинарник называется `till`):

```bash
cd ~/projects/rust
cargo new code/p08-refactor
cd code/p08-refactor
```

Добавьте в конец `Cargo.toml`:

```toml
[[bin]]
name = "till"
path = "src/main.rs"
```

Поместите этот код в `src/main.rs`. Это намеренно плохая программа. Прочитайте её один раз и перечислите её проблемы, прежде чем читать дальше.

```rust
use std::io::{self, BufRead};
use std::process;

fn main() {
    let args: Vec<String> = std::env::args().collect();
    let mode = if args.len() > 1 {
        args[1].clone()
    } else {
        String::from("receipt")
    };
    if mode != "receipt" && mode != "summary" && mode != "csv" {
        eprintln!("usage: till [receipt|summary|csv]");
        process::exit(2);
    }

    let stdin = io::stdin();
    let mut errors = 0;
    let mut orders = 0;
    let mut grand_total: u64 = 0;
    let mut cat_names: Vec<String> = Vec::new();
    let mut cat_totals: Vec<u64> = Vec::new();
    let mut cust_names: Vec<String> = Vec::new();
    let mut cust_totals: Vec<u64> = Vec::new();
    let mut line_no = 0;

    if mode == "csv" {
        println!("customer,item,qty,total_cents");
    }

    for line in stdin.lock().lines() {
        let line = line.unwrap();
        line_no += 1;
        let line = line.trim();
        if line.is_empty() || line.starts_with('#') {
            continue;
        }
        let parts: Vec<&str> = line.split(',').collect();
        if parts.len() != 5 {
            println!(
                "error: line {}: expected 5 fields, got {}",
                line_no,
                parts.len()
            );
            errors += 1;
            continue;
        }
        let name = parts[0].trim();
        let item = parts[1].trim();
        let qty: u64 = match parts[2].trim().parse() {
            Ok(q) => q,
            Err(_) => {
                println!(
                    "error: line {}: bad quantity '{}'",
                    line_no,
                    parts[2].trim()
                );
                errors += 1;
                continue;
            }
        };
        let price: u64 = match parts[3].trim().parse() {
            Ok(p) => p,
            Err(_) => {
                println!("error: line {}: bad price '{}'", line_no, parts[3].trim());
                errors += 1;
                continue;
            }
        };
        let category = parts[4].trim();

        let mut total = qty * price;
        if qty >= 10 {
            total = total - total * 10 / 100;
        }
        let tax_percent;
        if category == "food" {
            tax_percent = 7;
        } else if category == "drink" {
            tax_percent = 20;
        } else if category == "book" {
            tax_percent = 0;
        } else {
            println!(
                "warning: line {}: unknown category '{}', using 15%",
                line_no, category
            );
            tax_percent = 15;
        }
        total = total + total * tax_percent / 100;
        if name.starts_with("vip_") {
            total = total - total * 5 / 100;
        }

        orders += 1;
        grand_total += total;
        let mut found = false;
        for i in 0..cat_names.len() {
            if cat_names[i] == category {
                cat_totals[i] += total;
                found = true;
            }
        }
        if !found {
            cat_names.push(category.to_string());
            cat_totals.push(total);
        }
        let mut found = false;
        for i in 0..cust_names.len() {
            if cust_names[i] == name {
                cust_totals[i] += total;
                found = true;
            }
        }
        if !found {
            cust_names.push(name.to_string());
            cust_totals.push(total);
        }

        if mode == "receipt" {
            let dollars = total / 100;
            let cents = total % 100;
            if cents < 10 {
                println!(
                    "{:<8} {:<10} x{:<3} = {}.0{}",
                    name, item, qty, dollars, cents
                );
            } else {
                println!(
                    "{:<8} {:<10} x{:<3} = {}.{}",
                    name, item, qty, dollars, cents
                );
            }
        } else if mode == "csv" {
            println!("{},{},{},{}", name, item, qty, total);
        }
    }

    if mode == "receipt" {
        let dollars = grand_total / 100;
        let cents = grand_total % 100;
        if cents < 10 {
            println!("TOTAL: {}.0{} ({} orders)", dollars, cents, orders);
        } else {
            println!("TOTAL: {}.{} ({} orders)", dollars, cents, orders);
        }
    } else if mode == "summary" {
        println!("== summary ==");
        for i in 0..cat_names.len() {
            let dollars = cat_totals[i] / 100;
            let cents = cat_totals[i] % 100;
            if cents < 10 {
                println!("{:<8}: {}.0{}", cat_names[i], dollars, cents);
            } else {
                println!("{:<8}: {}.{}", cat_names[i], dollars, cents);
            }
        }
        if !cust_names.is_empty() {
            let mut best = 0;
            for i in 0..cust_names.len() {
                if cust_totals[i] > cust_totals[best] {
                    best = i;
                }
            }
            println!(
                "top customer: {} ({}.{})",
                cust_names[best],
                cust_totals[best] / 100,
                cust_totals[best] % 100
            );
        }
        println!("grand total: {}.{}", grand_total / 100, grand_total % 100);
        println!("orders: {}", orders);
    }

    if errors > 0 {
        eprintln!("{} error(s)", errors);
        process::exit(1);
    }
}
```

Скомпилируйте её через `cargo build`. Rustc компилирует её без предупреждений; `cargo clippy` сообщает об одном (`unneeded late initialization`). Это код, который работает, но его нельзя безопасно тестировать и менять: один `main` на 170 строк, повторяющийся код форматирования, магические числа (`10`, `7`, `20`, `15`, `5`), строки
для категорий и режимов, параллельные `Vec`, которые нужно держать согласованными, и печать, смешанная с вычислениями.

### Что она делает

На входе по одному заказу в строке: `customer,item,quantity,unit_price_in_cents,category`. Пустые строки и строки, начинающиеся с `#`, игнорируются. Создайте два входных файла и пустой:

```bash
mkdir -p tests/golden
cat > tests/golden/orders1.in <<'DATA'
# customer,item,qty,unit_price_cents,category
ana,coffee,2,350,drink
bo,sandwich,1,890,food
vip_cy,novel,3,1299,book
ana,water,12,100,drink
dan,gadget,1,2500,toys
DATA
cat > tests/golden/orders2.in <<'DATA'
ana,coffee,two,350,drink
bo,sandwich,1,890

cy,tea,1,250,drink
# a comment
cy,cake,1,300,food
dan,pen,1,-5,office
DATA
: > tests/golden/empty.in
```

Запустите программу в трёх режимах (`receipt` — по умолчанию):

```bash
cargo run -q -- receipt < tests/golden/orders1.in
cargo run -q -- summary < tests/golden/orders1.in
cargo run -q -- csv < tests/golden/orders1.in
```

```text
ana      coffee     x2   = 8.40
bo       sandwich   x1   = 9.52
vip_cy   novel      x3   = 37.03
ana      water      x12  = 12.96
warning: line 6: unknown category 'toys', using 15%
dan      gadget     x1   = 28.75
TOTAL: 96.66 (5 orders)
```

```text
warning: line 6: unknown category 'toys', using 15%
== summary ==
drink   : 21.36
food    : 9.52
book    : 37.03
toys    : 28.75
top customer: vip_cy (37.3)
grand total: 96.66
orders: 5
```

```text
customer,item,qty,total_cents
ana,coffee,2,840
bo,sandwich,1,952
vip_cy,novel,3,3703
ana,water,12,1296
warning: line 6: unknown category 'toys', using 15%
dan,gadget,1,2875
```

**Что происходит:**

1. Каждый заказ оценивается: количество, умноженное на цену за штуку; скидка 10% от десяти штук; налог по категории (еда 7%, напитки 20%, книги 0%, всё остальное 15% с предупреждением); и скидка 5% для клиентов, чьё имя начинается с `vip_`. Всё в целых центах, каждое деление округляет вниз.
2. `receipt` печатает строку на заказ и итог. `summary` печатает суммы по категориям в порядке первого появления, клиента с наибольшей суммой, общий итог и число заказов. `csv` печатает «сырые» центы.
3. Присмотритесь к `top customer: vip_cy (37.3)`. Правильная сумма — `37.03`: сводка форматирует центы без ведущего нуля. Это **ошибка**, и `grand total: 0.0` для пустого ввода имеет ту же причину. В этом проекте вы её **не** исправляете: сначала структура меняется при идентичном выводе; исправление ошибки — отдельный шаг (Дополнительная цель 1).

Теперь пути ошибок:

```bash
cargo run -q -- receipt < tests/golden/orders2.in; echo "exit status: $?"
cargo run -q -- bogus < tests/golden/orders1.in; echo "exit status: $?"
```

```text
error: line 1: bad quantity 'two'
error: line 2: expected 5 fields, got 4
cy       tea        x1   = 3.00
cy       cake       x1   = 3.21
error: line 7: bad price '-5'
TOTAL: 6.21 (2 orders)
3 error(s)
exit status: 1
usage: till [receipt|summary|csv]
exit status: 2
```

Обратите внимание, куда что идёт: строки `error:` и `warning:` печатаются в стандартный вывод, вперемешку с отчётом; только `3 error(s)` и сообщение об использовании идут в стандартный поток ошибок. Код возврата — `1`, если хоть одна строка была плохой, и `2` при неверном режиме. Всё это — *поведение*, даже если оно странное, и рефакторинг обязан его сохранить.

## Проект

### Что вы создадите

Вы не будете писать новые возможности. Вы возьмёте программу выше и за пять вех превратите её в небольшую слоистую протестированную программу с **набором golden-тестов**, который после каждого шага доказывает, что вывод не изменился ни на байт.

### Контракт

**Поведение (должно остаться идентичным, включая странности):**

- Командная строка, формат ввода, три режима, правила расчёта цены, форматы отчётов, предупреждения и ошибки, что идёт в стандартный вывод и стандартный поток ошибок, и коды возврата `0`, `1` и `2` — в точности как в разделе «Что она делает».
- До Дополнительной цели 1 цены без ведущего нуля в строках `top customer` и `grand total` сводки остаются.

**Структура (что вы строите):**

- Библиотека (`src/lib.rs`) с модулями и тонкий `src/main.rs`.
- Правила расчёта цены лежат в модуле, который ничего не печатает и ничего не читает.
- Категории и режимы — `enum`; деньги — тип; в правилах не осталось магических чисел.
- Логика программы — функция, которая принимает ввод и два выходных потока как параметры (трейты `BufRead` и `Write`), поэтому её можно тестировать без запуска процесса.
- Модульные тесты для каждого модуля плюс golden-тесты.
- `cargo fmt --check`, `cargo clippy --all-targets -- -D warnings` и `cargo test` проходят, а короткий `README.md` описывает программу.

### Пошаговый план

Работайте по порядку. После каждой вехи: выполните её команды «Готово, когда», затем сделайте коммит. Никогда не двигайтесь дальше с красным тестом. Если шаг не удался, `git restore .` и попробуйте шаг поменьше.

---

#### M1 — Golden-тесты: зафиксировать поведение

**Цель:** набор тестов, который запускает настоящий бинарник `till` на фиксированных входах и сравнивает stdout, stderr и код возврата с сохранёнными файлами. Всё после этой вехи опирается на него.

**Руководство:**

1. Создайте `tests/golden.rs`. Cargo сначала собирает бинарник и передаёт тесту его путь в `env!("CARGO_BIN_EXE_till")` (Урок 05.13). Этот помощник запускает программу на входном файле и возвращает то, что она напечатала:

```rust
use std::fs;
use std::io::Write;
use std::path::PathBuf;
use std::process::{Command, Stdio};

fn golden_dir() -> PathBuf {
    PathBuf::from(env!("CARGO_MANIFEST_DIR"))
        .join("tests")
        .join("golden")
}

/// Runs `till` with `args`, feeds it the file `input`, and returns (stdout, stderr, exit code).
fn run_till(args: &[&str], input: &str) -> (String, String, i32) {
    let text = fs::read(golden_dir().join(input)).unwrap();
    let mut child = Command::new(env!("CARGO_BIN_EXE_till"))
        .args(args)
        .stdin(Stdio::piped())
        .stdout(Stdio::piped())
        .stderr(Stdio::piped())
        .spawn()
        .unwrap();
    // The program may exit before it reads its input (wrong mode), so ignore a broken pipe.
    let _ = child.stdin.take().unwrap().write_all(&text);
    let output = child.wait_with_output().unwrap();
    (
        String::from_utf8_lossy(&output.stdout).into_owned(),
        String::from_utf8_lossy(&output.stderr).into_owned(),
        output.status.code().unwrap(),
    )
}
```

2. Напишите функцию `check(case, args, input)`. Она вызывает `run_till`, затем сравнивает три результата с файлами `tests/golden/<case>.stdout`, `.stderr` и `.code` (код возврата как текст). У неё есть второй режим: когда задана переменная окружения `UPDATE_GOLDEN`, она **записывает** эти файлы вместо сравнения. Так вы впервые создаёте golden-файлы. Используйте `std::env::var("UPDATE_GOLDEN").is_ok()`.
3. Напишите как минимум эти девять тестов, каждый — вызов `check`:

| Имя теста | Аргументы | Входной файл |
|---|---|---|
| `receipt_basic` | `receipt` | `orders1.in` |
| `default_mode_is_receipt` | *(нет)* | `orders1.in` |
| `summary_basic` | `summary` | `orders1.in` |
| `csv_basic` | `csv` | `orders1.in` |
| `receipt_with_errors` | `receipt` | `orders2.in` |
| `summary_with_errors` | `summary` | `orders2.in` |
| `empty_receipt` | `receipt` | `empty.in` |
| `empty_summary` | `summary` | `empty.in` |
| `unknown_mode` | `bogus` | `orders1.in` |

4. Один раз запустите `UPDATE_GOLDEN=1 cargo test`, чтобы создать golden-файлы, и **прочитайте их**. Сравните с выводом, напечатанным выше. Golden-файл, который вы ни разу не видели, ничего не доказывает.
5. Добавьте минимум три собственных случая, покрывающих правила, которых нет в файлах выше: граница между девятью и десятью штуками, два клиента с одинаковой суммой (кто «top customer»?), клиент `vip_` с оптовым заказом.

<details>
<summary>Подсказка 1 — функция `check`</summary>

Вычислите `base = golden_dir().join(case)`. Тогда `base.with_extension("stdout")` — путь к `<case>.stdout`. В режиме обновления используйте `fs::write`; в режиме сравнения — `fs::read_to_string` и `assert_eq!(actual, expected, "stdout of {case}")`. Для кода возврата сравните `code.to_string()` с содержимым файла `.code`.

</details>

<details>
<summary>Подсказка 2 — тест падает с «Broken pipe»</summary>

При неверном режиме программа завершается до чтения ввода; запись в её закрытый ввод тогда завершается ошибкой. В `run_till` игнорируйте ошибку `write_all` (`let _ = ...`), как это делает помощник выше.

</details>

**Готово, когда:**

```bash
cargo test
```

показывает, что все ваши golden-тесты проходят (имена и порядок строк на вашем экране будут другими):

```text
test default_mode_is_receipt ... ok
test unknown_mode ... ok
test summary_with_errors ... ok
test summary_basic ... ok
test receipt_basic ... ok
test empty_receipt ... ok
test empty_summary ... ok
test csv_basic ... ok
test receipt_with_errors ... ok
test result: ok. 9 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.25s
```

**И** страховочная сетка действительно ловит изменения. В `src/main.rs` замените `tax_percent = 20;` на `tax_percent = 21;` и запустите `cargo test`: должно упасть несколько тестов (в эталонном запуске 6 из 9):

```text
test receipt_basic ... FAILED
test summary_basic ... FAILED
test csv_basic ... FAILED
...
test result: FAILED. 3 passed; 6 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.23s
```

Верните как было, и `cargo test` снова зелёный. Тест, который не может упасть, бесполезен; эта «проверка мутацией» доказывает, что ваши могут. Коммит: `git add . && git commit -m "test(p08): golden tests for till"`.

---

#### M2 — Модули и слои

**Цель:** разделить длинный `main` на библиотеку с модулями, чтобы у каждого модуля была одна задача, а правила ничего не печатали.

**Руководство:**

1. Создайте `src/lib.rs` и вынесите код из `main`. Предлагаемые модули (Уроки 08.2 и 08.14):
   - `parse`: превращает строку текста в заказ или ошибку;
   - `pricing`: по заказу вычисляет цену (и налог), **без печати и без ввода**;
   - `report`: форматирует текст чека, сводки и CSV;
   - `app`: цикл, который читает строки, вызывает остальные модули и решает код возврата.
2. Переносите **по одной части** и запускайте `cargo test` после каждого переноса. Хороший порядок: форматирование денег (повторяющийся код `if cents < 10`) в одну функцию; затем `parse`; затем `pricing`; затем `report`; последним цикл в `app`.
3. На этом этапе типы могут оставаться простыми (`String`, `u64`). Цель M2 — *где живёт код*, а не *какие типы он использует*.
4. `src/main.rs` сжимается до: прочитать аргументы, вызвать библиотеку, завершиться с возвращённым кодом.
5. Правило зависимостей из Урока 08.14: `pricing` и `parse` не должны импортировать `std::io`.

<details>
<summary>Подсказка 1 — сохраните сообщения</summary>

Сообщения вроде `error: line 2: expected 5 fields, got 4` должны остаться символ в символ. Перенесите текст `format!` без изменений, улучшите потом.

</details>

<details>
<summary>Подсказка 2 — порядок строк вывода</summary>

Предупреждения и ошибки печатаются во время работы цикла, вперемешку со строками чека. Если ваш новый дизайн сначала собирает результаты, а потом печатает, golden-тесты покажут, что порядок изменился. Печатайте в том же порядке.

</details>

**Готово, когда:**

```bash
cargo test
wc -l src/main.rs
grep -n "std::io" src/pricing.rs src/parse.rs
```

`cargo test` по-прежнему сообщает, что все golden-тесты проходят (`9 passed` или больше, если вы добавили случаи); `wc -l` печатает число меньше 30; а `grep` ничего не печатает (нет совпадений). Коммит: `git add . && git commit -m "refactor(p08): split till into modules"`.

---

#### M3 — Типы вместо строк и чисел

**Цель:** сделать неверные состояния трудными для записи (Урок 08.5) и убрать магические числа (Урок 08.10).

**Руководство:**

1. Замените строку режима на `enum Mode { Receipt, Summary, Csv }` с одной функцией, преобразующей слово из командной строки; неизвестные слова дают `None`, а `main` печатает сообщение об использовании.
2. Замените строку категории на `enum Category { Food, Drink, Book, Other(String) }`. `Other` помнит исходное слово, потому что предупреждение и сводка его печатают.
3. Введите `struct Cents(u64)` для денег с `Display`, печатающим `dollars.cents` с двумя цифрами для центов. Деньги форматирует **одно** место. Повторяющийся код `if cents < 10` должен исчезнуть.
4. Странность сводки (центы без ведущего нуля) — *не* тот же формат. Сохраните её, но сделайте видимой: отдельный чётко названный метод с комментарием, который говорит, что он сохраняет старое поведение до осознанного исправления.
5. Замените `println!` ошибок разбора на `enum ParseError`, реализующий `Display` с в точности прежними сообщениями.
6. Дайте имя каждому числу в правилах: порог оптового заказа, оптовая скидка, префикс VIP (Very Important Person, особо важная персона), скидка VIP и один процент налога на категорию. `match` по категории без ветки `_` показывает, какой налог применяется.
7. Замените параллельные `Vec` (`cat_names` и `cat_totals`, `cust_names` и `cust_totals`) одним `Vec<(String, Cents)>` на список или небольшой структурой. Сохраните *порядок первого появления*, потому что от него зависят golden-тесты (`HashMap` печатал бы в случайном порядке).

<details>
<summary>Подсказка 1 — где живут слова</summary>

Слова `"food"`, `"drink"`, `"book"` и `"receipt"`, `"summary"`, `"csv"` должны встречаться каждое в **одной** функции, преобразующей текст в тип (а для категории — ещё в одной, преобразующей обратно). Везде в остальном вы используете `enum`.

</details>

<details>
<summary>Подсказка 2 — округление</summary>

Помощник `fn percent(self, p: u64) -> Cents`, вычисляющий `self * p / 100`, держит правило округления в одном месте. Не «упрощайте» `total - total * 5 / 100` до `total * 95 / 100`: golden-тесты это поймают, как в Уроке 08.13.

</details>

**Готово, когда:**

```bash
cargo test
cargo clippy --all-targets -- -D warnings
grep -rn '{}.0{}' src
grep -n '"food"\|"drink"\|"book"' src/*.rs
```

Тесты проходят; Clippy молчит (только `Checking` и `Finished`); третья команда ничего не печатает; а четвёртая показывает совпадения только в **одном** исходном файле (том, что преобразует между текстом и `Category`). Коммит: `git add . && git commit -m "refactor(p08): types for mode, category and money"`.

---

#### M4 — Ввод-вывод за трейтами

**Цель:** логика программы не должна знать о настоящей клавиатуре и экране, чтобы её можно было тестировать строками в памяти (Уроки 08.3 и 08.14).

**Руководство:**

1. Превратите цикл в `app` в функцию такой формы (имена ваши):

```rust
pub fn run(
    mode: Mode,
    input: impl BufRead,
    out: &mut impl Write,
    err: &mut impl Write,
) -> io::Result<i32>
```

   Она читает строки из `input`, пишет отчёт и предупреждения в `out`, пишет `N error(s)` в `err` и возвращает код возврата. `BufRead` и `Write` — трейты стандартной библиотеки: `&[u8]` является `BufRead`; `Vec<u8>` является `Write` (Урок 08.3).
2. Функции отчёта принимают `out: &mut impl Write` и используют `writeln!` вместо `println!`. `writeln!` возвращает `io::Result`, поэтому используйте `?`.
3. С настоящим терминалом общается только `main.rs`: он передаёт `stdin.lock()`, `io::stdout().lock()` и `io::stderr().lock()` в `run`, затем вызывает `std::process::exit` с возвращённым кодом.
4. Добавьте минимум три теста, вызывающих `run` со строкой на входе и `Vec<u8>` на выходах, без запуска процесса: чек для одного заказа, ввод с плохой строкой (проверьте вывод, поток ошибок и код возврата) и сводка.

<details>
<summary>Подсказка 1 — строка как ввод</summary>

`"ana,coffee,2,350,drink\n".as_bytes()` — это `&[u8]`, реализующий `BufRead`. После `run` преобразуйте буферы через `String::from_utf8(out).unwrap()`.

</details>

<details>
<summary>Подсказка 2 — `lines()` возвращает результаты</summary>

`input.lines()` выдаёт `io::Result<String>`. Применяйте `?` к строке, чтобы ошибка чтения стала ошибкой `run`. В `main` решите, что печатать, когда сама `run` завершается ошибкой.

</details>

**Готово, когда:**

```bash
cargo test
grep -n "stdin\|stdout\|stderr\|println" src/*.rs
```

Тесты проходят, включая ваши тесты `run` в памяти, и все совпадения `grep` находятся в `src/main.rs`. Коммит: `git add . && git commit -m "refactor(p08): run() takes BufRead and Write"`.

---

#### M5 — Модульные тесты и контроль качества

**Цель:** быстрые тесты для каждой части, чистый контроль качества, документация.

**Руководство:**

1. Модульные тесты (`#[cfg(test)] mod tests` в каждом файле): форматирование `Cents` (`5` даёт `0.05`, `3703` даёт `37.03`, `0` даёт `0.00`) и метод-странность (`3703` даёт `37.3`); разбор (корректная строка, пустая строка, комментарий и по одному тесту на каждое сообщение об ошибке); расчёт цены (каждая ставка налога, граница в десять штук, скидка VIP и её округление, неизвестная категория).
2. Каждый тест проверяет одно правило и имеет имя, которое его формулирует (Урок 05.6). Тесты странности названы так, чтобы следующий читатель знал: она намеренная.
3. Запустите контроль качества: `cargo fmt --check`, `cargo clippy --all-targets -- -D warnings`, `cargo test`.
4. Напишите `README.md` (Урок 05.7): что делает `till`, формат ввода, режимы, коды возврата и как запускать тесты. Перечислите «известные странности» и упомяните центы без ведущего нуля.
5. Просмотрите собственный diff как ревьюер (Урок 08.13): нет ли `pub`, который мог бы быть приватным? Функции длиннее 30 строк? `unwrap()` на вводе?

<details>
<summary>Подсказка 1 — сколько тестов?</summary>

Разумный результат — примерно десять-пятнадцать небольших модульных тестов рядом с девятью или более golden-тестами. Golden-тесты защищают всю программу; модульные тесты облегчают поиск места сбоя.

</details>

**Готово, когда:**

```bash
cargo fmt --check && cargo clippy --all-targets -- -D warnings && cargo test
```

не печатает ни разницы форматирования, ни предупреждений Clippy и показывает два набора пройденных тестов: ваши модульные (`test result: ok. 13 passed` в эталонном запуске) и golden-тесты (`ok. 9 passed`):

```text
test result: ok. 13 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
...
test result: ok. 9 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.25s
```

И `git diff main -- tests/golden` показывает **отсутствие изменений в golden-файлах**, созданных вами в M1 (только добавленные случаи, если они есть). Коммит: `git add . && git commit -m "test(p08): unit tests, README and clean quality gate"`. В конце перенесите проект в основную ветку: `git switch main && git merge project/p08-refactor`.

---

### Дополнительные цели

1. **Исправьте ошибку осознанно.** Сделайте так, чтобы сводка печатала `37.03` и `0.00`. Это изменение *поведения*: измените код, посмотрите, как падают golden-тесты `summary_basic`, `summary_with_errors` и `empty_summary`, убедитесь, что новый текст — то, что вам нужно, пересоздайте эти файлы через `UPDATE_GOLDEN=1` и закоммитьте код и golden-файлы вместе с сообщением `fix(p08): pad cents in the summary`. Удалите метод-странность.
2. **Стратегия.** Замените `match mode` в `run` трейтом `Renderer` с тремя реализациями (`ReceiptRenderer`, `SummaryRenderer`, `CsvRenderer`), как в Уроке 08.9. Код стал проще или просто длиннее? Запишите свой ответ.
3. **Рабочее пространство.** Перенесите библиотеку в крейт `till-core`, а тонкую программу — в `till-cli` (Урок 08.12) с `[workspace.lints]`. Теперь компилятор принудительно следит, чтобы ядро не зависело от программы.
4. **Тест свойств.** С помощью крейта `proptest` (Этап 09) проверьте, что итог чека всегда равен сумме итогов заказов.
5. **Аргументы через clap.** Замените ручную обработку аргументов на `clap` (Урок 05.11), сохранив сообщение об использовании и код возврата `2` идентичными, либо измените их осознанно, обновив golden-файлы отдельным коммитом.

## Частые ошибки

1. **Рефакторинг до появления golden-тестов.** Ничто не подскажет вам, что вывод не изменился.
2. **Создать golden-файлы и никогда их не прочитать.** Если в программе была ошибка в случае, на который вы не смотрели, вы зафиксировали ошибку, не зная о ней.
3. **«Исправить» форматирование центов при переносе.** Это меняет вывод. Сохраните его, протестируйте и исправьте отдельным коммитом (Дополнительная цель 1).
4. **Изменить порядок вывода.** Предупреждения и ошибки печатаются между строками чека. Дизайн, который сначала собирает, а потом печатает, это меняет.
5. **Использовать `HashMap` для итогов.** Порядок его обхода случаен, поэтому сводка отличалась бы от запуска к запуску. Сохраняйте порядок первого появления.
6. **Большие шаги.** Один коммит «всё сразу». Когда golden-тест падает, вы не знаете, какое изменение это сделало.
7. **Переусложнение.** Пять трейтов и три крейта на 180 строк — противоположность урока. У каждой абстракции должна быть причина, которую можно сказать одним предложением.

## Вопросы для самопроверки

1. Почему golden-тесты пишут *до* того, как трогать код?

<details>
<summary>Ответ</summary>

Они — страховочная сетка: фиксируют текущее поведение, так что после каждого маленького шага вы за секунды знаете, изменился ли вывод.

</details>

2. Сводка печатает `37.3` вместо `37.03`. Почему вы сохраняете это во время рефакторинга?

<details>
<summary>Ответ</summary>

Потому что рефакторинг не должен менять поведение. Если изменение структуры и исправление ошибки смешаны, падение golden-теста может означать и то и другое, а ревью становится сложнее. Исправьте это потом отдельным коммитом, который заодно обновит golden-файлы.

</details>

3. Почему `run` принимает `impl BufRead` и `&mut impl Write`, а не использует `println!` и настоящий стандартный ввод?

<details>
<summary>Ответ</summary>

Чтобы логику можно было тестировать со строками и буферами в памяти, без запуска процесса, и чтобы только `main` знал о настоящем терминале. Это инверсия зависимостей (Урок 08.3) и порт ядра (Урок 08.14).

</details>

4. Почему модуль расчёта цены ничего не печатает?

<details>
<summary>Ответ</summary>

Модуль с одной задачей (вычислять) легко тестировать на простых значениях и легко переиспользовать; печать — другая ответственность (разделение ответственности, Урок 08.2).

</details>

5. Почему `HashMap` — рискованный выбор для итогов по категориям здесь?

<details>
<summary>Ответ</summary>

Порядок его обхода не определён и может отличаться между запусками, поэтому строки сводки появлялись бы в другом порядке, а golden-тесты падали бы непредсказуемо. `Vec` пар сохраняет порядок первого появления.

</details>

## Итоги

- Перед рефакторингом программы без тестов зафиксируйте её поведение **golden-тестами**, которые запускают настоящий бинарник и сравнивают stdout, stderr и код возврата; проверьте, что они могут упасть.
- Рефакторьте малыми шагами (модули, затем типы, затем трейты ввода-вывода, затем модульные тесты) и держите каждый golden-тест зелёным после каждого шага; коммитьте после каждого.
- Ошибки и странности, которые вы нашли, сначала **сохраняются** и исправляются позже отдельным коммитом, с осознанным обновлением golden-файлов.
- Правила расчёта цены не печатают; логика принимает `BufRead` и `Write`; с настоящим терминалом общается только `main`. Это разделение ответственности и инверсия зависимостей в несколько десятков строк.
- Заменяйте строки и голые числа на `enum`, тип денег и именованные константы; используйте `match` без веток `_`, чтобы компилятор вас охранял.
- Завершите контролем качества (`fmt`, `clippy -D warnings`, `test`), README и ревью собственного diff.

## Источники и что почитать

- [The Rust Programming Language — Chapter 11.3: Test Organization](https://doc.rust-lang.org/book/ch11-03-test-organization.html) — модульные тесты, интеграционные тесты и папка `tests`.
- [Refactoring — каталог Мартина Фаулера](https://refactoring.com/catalog/) — названия приёмов, которые вы использовали.
- [The Rust Programming Language — Chapter 12: An I/O Project](https://doc.rust-lang.org/book/ch12-00-an-io-project.html) — библиотека плюс тонкий `main`, похожая структура.
- [Стандартная библиотека Rust — `BufRead`](https://doc.rust-lang.org/std/io/trait.BufRead.html) и [`Write`](https://doc.rust-lang.org/std/io/trait.Write.html) — два трейта, лежащие в основе M4.
- [The Cargo Book — Environment variables](https://doc.rust-lang.org/cargo/reference/environment-variables.html) — переменные окружения, которые Cargo задаёт для тестов (включая `CARGO_BIN_EXE_<name>`).

---

[🏠 Начало](../START-HERE.md) · [← 08.15 Приложение двенадцати факторов](15-the-twelve-factor-app.md) · [📋 Этап 08](README.md) · [08.17 Контрольная точка →](17-checkpoint.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
