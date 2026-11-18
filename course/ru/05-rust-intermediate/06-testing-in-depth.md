<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 05.5 Итераторы](05-iterators.md) · [📋 Этап 05](README.md) · [05.7 Документация →](07-documentation.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/05-rust-intermediate/06-testing-in-depth.md) <!-- nav -->

# Урок 05.6 — Тестирование подробно

> **Этап 05 — Rust: средний уровень** · Урок 6 из 14 · ⏱ около 3,5 часа (2 занятия)

## Что вы узнаете

- Как работать «тест вперёд» (TDD: красный, зелёный, рефакторинг) и как читать упавший тест.
- Атрибуты тестов `#[should_panic]`, `#[ignore]`, тесты, возвращающие `Result`, и полезные параметры `cargo test`.
- Как тестировать ошибки, граничные случаи, переполнение и числа с плавающей точкой.
- Чем отличаются модульные тесты, интеграционные тесты (папка `tests/`) и документационные тесты и как устроить библиотеку так, чтобы её можно было тестировать.
- Как запускать «контроль качества» `cargo fmt`, `cargo clippy` и `cargo test`, как запускать его автоматически перед каждым коммитом и как находить ошибку в истории через `git bisect`.

## Перед началом

- Вы прошли [Урок 05.5 — Итераторы](05-iterators.md).
- Первые тесты вы писали в [Уроке 02.8 — Ваши первые тесты](../02-programming-basics/08-your-first-tests.md): `#[test]`, `assert_eq!`, `cargo test`. Краткое напоминание — в разделе 1.
- Вы знаете модули и крейты ([Урок 04.6](../04-structs-enums-collections-errors/06-modules-crates-and-packages.md)) и обработку ошибок ([Урок 04.10](../04-structs-enums-collections-errors/10-error-handling.md)).
- Git: ветки и коммиты ([Уроки 01.3–01.6](../01-git-and-version-control/03-your-first-repository-and-commits.md)).

## Новые слова

| Слово | Значение |
|---|---|
| **модульный тест** (unit test) | Маленький тест одной функции или типа, написанный в том же файле, что и код, внутри `#[cfg(test)] mod tests`. Видит приватные элементы. |
| **интеграционный тест** (integration test) | Тест в папке `tests/`, который использует ваш крейт снаружи, как это сделала бы другая программа. Видит только публичный API (Application Programming Interface, программный интерфейс: элементы, которые ваш крейт предлагает другому коду). |
| **документационный тест** (doc test) | Пример внутри комментария документации, который `cargo test` компилирует и запускает (Урок 05.7). |
| **TDD** (Test-Driven Development, разработка через тестирование) | Способ работы: сначала пишется падающий тест, затем код, который его проходит, затем наводится порядок. |
| **красный / зелёный / рефакторинг** | Три шага TDD: тест падает (красный), тест проходит (зелёный), вы улучшаете код, пока тесты остаются зелёными. |
| **граничный случай** (edge case) | Необычный, но допустимый вход на границе разрешённого: пустой текст, ноль, максимальное значение. Большинство ошибок живёт там. |
| **табличный тест** (table-driven test) | Один тест, который проходит по списку случаев (вход, ожидаемый результат). |
| **регрессия** (regression) | Ошибка, которая вернулась, или то, что раньше работало и перестало. *Регрессионный тест* защищает от неё. |
| **нестабильный тест** (flaky test) | Тест, который иногда проходит, а иногда падает без изменений кода. Избегайте их. |
| **библиотечный крейт / бинарный крейт** | Крейт, собираемый из `src/lib.rs` (код для использования другими) / из `src/main.rs` (программа, которую запускают). |
| **хук Git** (Git hook) | Скрипт в `.git/hooks/`, который Git запускает в определённый момент, например перед коммитом. |
| **`git bisect`** | Инструмент Git, который находит коммит, внёсший ошибку, двоичным поиском по истории. |
| **CI** (Continuous Integration, непрерывная интеграция) | Сервер, который автоматически запускает ваши проверки при каждой отправке. Этап 09 строит конвейеры CI; хук из этого урока — их локальная версия. |

## 1. Напоминание и новый учебный пакет

Из Урока 02.8 вы знаете, что функция с пометкой `#[test]` — это тест, что он проходит, если не паникует, и
что `assert!`, `assert_eq!` и `assert_ne!` паникуют, когда условие ложно. Все они принимают необязательное
сообщение с теми же подстановками, что и `println!`:

```rust
assert_eq!(parse_duration("45s"), Ok(45), "input was {:?}", "45s");
assert!(width > 2, "width must be greater than 2 but was {width}");
```

Сообщение печатается только когда тест падает, и именно его вы читаете первым. Хорошее сообщение говорит,
*что проверялось*.

Начиная с этого урока мы тренируемся в **библиотечном** пакете. Создайте его рядом с остальным кодом:

```bash
cd ~/projects/rust
git switch main
git switch -c lesson/05-06-testing
cargo new --lib code/p05-textkit
```

`--lib` создаёт `src/lib.rs` вместо `src/main.rs`. Библиотеку нельзя запустить, её можно только тестировать
и использовать из другого кода, что нам и нужно. (Отредактируйте `Cargo.toml`, если хотите, но имя по умолчанию
`p05-textkit` подходит. В коде Rust крейт называется `p05_textkit`: дефисы становятся подчёркиваниями.)

## 2. Разработка через тестирование: красный, зелёный, рефакторинг

TDD означает писать тест **до** кода. Звучит наоборот, но это даёт три вещи: вы сначала продумываете
*интерфейс*, точно знаете, когда закончили, и каждая строка кода существует, потому что её потребовал тест.

Мы напишем `slugify` — функцию, которая превращает заголовок вроде `"Hello World"` в текст, безопасный для
URL (Uniform Resource Locator, адрес в интернете): `"hello-world"`. Замените содержимое `src/lib.rs`:

```rust
pub fn slugify(_input: &str) -> String {
    todo!()
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn lowercases_and_joins_words_with_dashes() {
        assert_eq!(slugify("Hello World"), "hello-world");
    }
}
```

**Что происходит:**

1. `todo!()` — макрос, который компилируется, но при выполнении паникует с сообщением «not yet implemented».
   Это заглушка, чтобы код компилировался, а *тест* мог упасть по-своему.
2. `_input` начинается с подчёркивания, поэтому компилятор не предупреждает о неиспользуемом параметре.
3. Тест вызывает `slugify` так, как это сделал бы пользователь, и указывает ожидаемый результат.

**Шаг 1, красный.** Запустите `cargo test` (из `code/p05-textkit`). Тест должен упасть:

```text
running 1 test
test tests::lowercases_and_joins_words_with_dashes ... FAILED

failures:

---- tests::lowercases_and_joins_words_with_dashes stdout ----

thread 'tests::lowercases_and_joins_words_with_dashes' panicked at src/lib.rs:2:5:
not yet implemented

failures:
    tests::lowercases_and_joins_words_with_dashes

test result: FAILED. 0 passed; 1 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s

error: test failed, to rerun pass `--lib`
```

Важно увидеть, как тест падает первым: это доказывает, что тест **может** упасть. Тест, который никогда не
был красным, возможно, ничего не проверяет.

**Шаг 2, зелёный.** Напишите самый простой код, который проходит тест, и запустите снова:

```rust
/// Turns a title into a lowercase, dash-separated text that is safe to use in a URL.
pub fn slugify(input: &str) -> String {
    let mut slug = String::new();
    let mut last_was_dash = true; // true at the start: no leading dash
    for ch in input.chars() {
        if ch.is_alphanumeric() {
            for lower in ch.to_lowercase() {
                slug.push(lower);
            }
            last_was_dash = false;
        } else if !last_was_dash {
            slug.push('-');
            last_was_dash = true;
        }
    }
    if slug.ends_with('-') {
        slug.pop();
    }
    slug
}
```

**Что происходит:**

1. Мы идём по символам. Буква или цифра (`is_alphanumeric`, работает и для букв не английского алфавита)
   добавляется в нижнем регистре. `to_lowercase()` возвращает итератор, потому что в некоторых алфавитах один
   символ может превратиться в несколько, поэтому мы добавляем каждый результат.
2. Любой другой символ (пробел, знак препинания) становится **одним** дефисом, но только если последним
   записанным не был дефис. `last_was_dash` начинается с `true`, поэтому дефиса в начале никогда не бывает.
3. В конце мы убираем завершающий дефис.
4. Комментарий `///` над функцией — это документация (Урок 05.7).

**Шаг 3, рефакторинг.** При зелёных тестах можно улучшать имена, убирать дублирование или упрощать, запуская
тесты после каждого изменения. Если они остаются зелёными, поведение то же. Затем повторите цикл со следующим
требованием: напишите новый падающий тест, сделайте его зелёным.

Добавьте больше тестов на поведение, которое вам важно. Каждое имя — предложение, говорящее, что делает код:

```rust
    #[test]
    fn slugify_collapses_punctuation_and_trims_dashes() {
        assert_eq!(slugify("  Rust -- is... fun!  "), "rust-is-fun");
    }

    #[test]
    fn slugify_keeps_digits_and_non_ascii_letters() {
        assert_eq!(slugify("Привет, мир 2026"), "привет-мир-2026");
    }

    #[test]
    fn slugify_of_empty_text_is_empty() {
        assert_eq!(slugify(""), "");
        assert_eq!(slugify("!!!"), "");
    }
```

(В файле тесты находятся внутри `mod tests { … }`; для единообразия переименуйте и первый тест, чтобы его имя тоже начиналось с `slugify_`.)

> 📏 **Соглашение:** называйте тест по поведению (`slugify_of_empty_text_is_empty`), а не по функции
> (`test_slugify_1`). Когда он падает, одно имя говорит, что сломалось. Строите тело из трёх частей:
> **arrange** (подготовить вход), **act** (вызвать код), **assert** (проверить результат). Одно поведение — один тест.

### Как читать падение

Когда проверка падает, Rust печатает оба значения. Допустим, кто-то сломал тест, ожидая `Ok(54)`:

```text
---- tests::parse_duration_adds_all_parts stdout ----

thread 'tests::parse_duration_adds_all_parts' panicked at src/lib.rs:92:9:
assertion `left == right` failed
  left: Ok(45)
 right: Ok(54)
```

`left` — первый аргумент `assert_eq!` (фактическое значение), `right` — второй (ожидаемое). Файл и строка
(`src/lib.rs:92:9`) указывают на проверку. В RustRover можно также нажать зелёную стрелку на полях (рядом с номерами
строк) возле `#[test]`, чтобы запустить один тест, и красный маркер, чтобы перейти к месту падения.

## 3. Тестирование ошибок, граничных случаев и переполнения

Хорошие тесты в основном про то, что происходит на **границе**. Добавим вторую функцию, которая может
завершаться ошибкой, и проверим каждый способ, которым она может упасть. `parse_duration` читает текст вроде
`"1h30m15s"` и возвращает число секунд:

```rust
#[derive(Debug, PartialEq)]
pub enum ParseError {
    Empty,
    MissingNumber,
    MissingUnit,
    UnknownUnit(char),
    TooLarge,
}

/// Parses a duration such as "1h30m15s" and returns the number of seconds.
pub fn parse_duration(input: &str) -> Result<u64, ParseError> {
    let input = input.trim();
    if input.is_empty() {
        return Err(ParseError::Empty);
    }
    let mut total: u64 = 0;
    let mut digits = String::new();
    for ch in input.chars() {
        if ch.is_ascii_digit() {
            digits.push(ch);
            continue;
        }
        if digits.is_empty() {
            return Err(ParseError::MissingNumber);
        }
        let value: u64 = digits.parse().map_err(|_| ParseError::TooLarge)?;
        let seconds = match ch {
            'h' => value.checked_mul(3600),
            'm' => value.checked_mul(60),
            's' => Some(value),
            other => return Err(ParseError::UnknownUnit(other)),
        };
        let seconds = seconds.ok_or(ParseError::TooLarge)?;
        total = total.checked_add(seconds).ok_or(ParseError::TooLarge)?;
        digits.clear();
    }
    if !digits.is_empty() {
        return Err(ParseError::MissingUnit);
    }
    Ok(total)
}
```

**Что происходит:**

1. Тип ошибки — перечисление с вариантом на каждый способ упасть (Урок 04.10). Он выводит `Debug` и
   `PartialEq`, чтобы тесты могли сравнивать его через `assert_eq!`.
2. Цикл собирает цифры в `String`. Встретив не цифру, он считает её единицей (`h`, `m`, `s`), превращает
   собранные цифры в число, умножает и добавляет к `total`.
3. `checked_mul` и `checked_add` возвращают `None` вместо «заворачивания», когда результат не помещается в
   `u64`; мы превращаем это в `ParseError::TooLarge`.

> 🔒 **Безопасность:** код, разбирающий ввод от пользователей или из сети, никогда не должен доверять числам. Без
> методов `checked_` строка `"9999999999999999999h"` вызвала бы переполнение: панику в отладочной сборке и
> молча неверный ответ в релизной. Тест, который подаёт **самые большие и самые плохие** входы, — способ найти
> такие ошибки. Напишите его.

Теперь тесты. Каждая группа покрывает один вид случаев:

```rust
    #[test]
    fn parse_duration_adds_all_parts() {
        assert_eq!(parse_duration("1h30m15s"), Ok(5415));
        assert_eq!(parse_duration("45s"), Ok(45));
    }

    #[test]
    fn parse_duration_rejects_bad_input() {
        assert_eq!(parse_duration(""), Err(ParseError::Empty));
        assert_eq!(parse_duration("h"), Err(ParseError::MissingNumber));
        assert_eq!(parse_duration("10"), Err(ParseError::MissingUnit));
        assert_eq!(parse_duration("5x"), Err(ParseError::UnknownUnit('x')));
    }

    #[test]
    fn parse_duration_detects_overflow() {
        assert_eq!(
            parse_duration("99999999999999999999s"),
            Err(ParseError::TooLarge)
        );
        assert_eq!(
            parse_duration("9999999999999999999h"),
            Err(ParseError::TooLarge)
        );
    }
```

**Что происходит:** первый тест проверяет «счастливый путь». Во втором по одной проверке на **каждый** вариант
ошибки. Третий подаёт числа, слишком большие для `u64`: первое падает в `parse()`, второе — в `checked_mul`.
Сравнивать целые значения `Result` через `assert_eq!` — самый ясный способ проверить ошибку. Когда важно лишь то,
*что это ошибка*, используйте `assert!(result.is_err())` или `assert!(matches!(result, Err(ParseError::UnknownUnit(_))))`:
макрос `matches!` возвращает `true`, когда значение подходит под образец (образцы, Урок 04.5).

### Табличные тесты

Когда для одной проверки есть много входов, положите их в список и пройдите циклом:

```rust
    #[test]
    fn table_of_valid_durations() {
        let cases = [("1m", 60), ("2h", 7200), ("1h1m1s", 3661), ("0s", 0)];
        for (text, expected) in cases {
            assert_eq!(parse_duration(text), Ok(expected), "input was {text:?}");
        }
    }
```

**Что происходит:** добавление случая — ещё один кортеж. Сообщение `input was {text:?}` здесь важно: цикл работает
внутри одного теста, и без него вы не узнали бы, **какой** случай упал. (`{:?}` печатает текст в кавычках, из-за чего
видны пробелы и пустые строки.)

### Числа с плавающей точкой

Числа с плавающей точкой редко бывают точными (Урок 02.2): `0.1 + 0.2` — это не `0.3`. Никогда не сравнивайте их
через `assert_eq!`. Сравнивайте разность с небольшим допуском:

```rust
    #[test]
    fn floats_are_compared_with_a_tolerance() {
        let sum: f64 = 0.1 + 0.2;
        assert!(sum != 0.3, "0.1 + 0.2 is not exactly 0.3");
        assert!((sum - 0.3).abs() < 1e-9);
    }
```

`1e-9` означает 0.000000001. `.abs()` делает разность положительной, поэтому проверка работает в обе стороны.

## 4. Другие атрибуты тестов и параметры `cargo test`

### `#[should_panic]`

Некоторый код *должен* паниковать (Урок 04.10: из-за ошибок программиста, а не ожидаемых сбоев). Чтобы проверить это,
добавьте `#[should_panic]`:

```rust
    #[test]
    #[should_panic(expected = "index out of bounds")]
    fn indexing_past_the_end_panics() {
        let v: Vec<i32> = Vec::new();
        let _ = v[0];
    }
```

**Что происходит:** тест проходит, **только если** код паникует, а `expected = "…"` дополнительно требует, чтобы
сообщение паники **содержало** этот текст. Всегда пишите `expected`, иначе паника по совсем другой причине сделает
тест успешным. (`let _ = …` сообщает компилятору, что мы сознательно игнорируем значение.)

### Тесты, возвращающие `Result`

Тест может возвращать `Result<(), E>`. Тогда внутри можно использовать `?` вместо `unwrap`, а `Err` роняет тест:

```rust
    #[test]
    fn test_can_return_a_result() -> Result<(), ParseError> {
        let seconds = parse_duration("2m")?;
        assert_eq!(seconds, 120);
        Ok(())
    }
```

### `#[ignore]`

Медленные тесты (сетевой вызов, большой файл) можно по умолчанию пропускать:

```rust
    #[test]
    #[ignore = "slow: pretend this calls the network"]
    fn slow_test() {
        std::thread::sleep(std::time::Duration::from_secs(2));
    }
```

### Запуск тестов: нужные параметры

`cargo test [ФИЛЬТР] [-- ФЛАГИ]`. Всё **до** `--` предназначено Cargo, всё **после** передаётся программе тестов.

| Команда | Что делает |
|---|---|
| `cargo test` | запустить все тесты (модульные, интеграционные и документационные) |
| `cargo test slugify` | запустить только тесты, в имени которых **есть** `slugify` |
| `cargo test -- --ignored` | запустить **только** тесты с `#[ignore]` |
| `cargo test -- --include-ignored` | запустить всё, включая игнорируемые тесты |
| `cargo test -- --list` | показать список тестов, не запуская их |
| `cargo test -- --nocapture` | показывать вывод `println!` у успешных тестов (по умолчанию он скрыт) |
| `cargo test -- --test-threads=1` | запускать тесты по одному, по порядку |
| `cargo test --lib` / `--test api` / `--doc` | запустить только модульные тесты / один файл интеграционных / документационные тесты |
| `cargo test --release` | собрать тесты с оптимизациями (для медленных, тяжёлых тестов) |

Пример запуска с фильтром:

```text
$ cargo test slugify
running 4 tests
test tests::slugify_collapses_punctuation_and_trims_dashes ... ok
test tests::slugify_keeps_digits_and_non_ascii_letters ... ok
test tests::slugify_of_empty_text_is_empty ... ok
test tests::slugify_lowercases_and_joins_words_with_dashes ... ok

test result: ok. 4 passed; 0 failed; 0 ignored; 0 measured; 7 filtered out; finished in 0.00s
```

И полного запуска, где виден игнорируемый тест:

```text
running 11 tests
test tests::slow_test ... ignored, slow: pretend this calls the network
test tests::parse_duration_adds_all_parts ... ok
...
test result: ok. 10 passed; 0 failed; 1 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

(Порядок строк может различаться между запусками, потому что Rust запускает тесты **параллельно** в нескольких потоках.)

### Тесты должны быть независимыми

Поскольку тесты идут параллельно и без гарантии порядка, **тест никогда не должен зависеть от другого теста**
или от общего состояния, такого как глобальная переменная или один и тот же временный файл. Тест, который работает
в одиночку, но падает вместе с другими, или падает время от времени, — это **нестабильный тест (flaky test)**.
Лекарство — дать каждому тесту собственные данные (своё имя временного файла, свою структуру) и не полагаться на
часы, случайные числа или сеть. `--test-threads=1` помогает выяснить, не виноват ли параллелизм.

## 5. Модульные, интеграционные и документационные тесты

| Вид | Где | Что видит | Типичное применение |
|---|---|---|---|
| модульный тест | `#[cfg(test)] mod tests` в том же файле | **всё**, включая приватные функции | одна функция или тип |
| интеграционный тест | файл в `tests/` | **только публичный API** | работает ли крейт так, как его использует пользователь? |
| документационный тест | пример в комментарии `///` | только публичный API | примеры остаются верными (Урок 05.7) |

`#[cfg(test)]` означает «компилировать этот код только при тестировании», так что тестовый код не попадает в
итоговую программу.

Интеграционные тесты лежат в папке `tests/` в корне пакета, рядом с `src/`. Каждый файл там компилируется как
**отдельный крейт**, который импортирует ваш, как любой другой пользователь:

```text
p05-textkit/
├── Cargo.toml
├── src/
│   └── lib.rs
└── tests/
    ├── api.rs
    └── common/
        └── mod.rs      ← вспомогательный код, общий для интеграционных тестов
```

`tests/api.rs`:

```rust
mod common;

use p05_textkit::{ParseError, parse_duration, slugify};

#[test]
fn slugs_of_sample_titles() {
    let slugs: Vec<String> = common::sample_titles().into_iter().map(slugify).collect();
    assert_eq!(slugs, vec!["hello-world", "rust-friends", "spaced-out"]);
}

#[test]
fn public_api_parses_and_reports_errors() {
    assert_eq!(parse_duration("1m"), Ok(60));
    assert_eq!(parse_duration("1x"), Err(ParseError::UnknownUnit('x')));
}
```

`tests/common/mod.rs`:

```rust
pub fn sample_titles() -> Vec<&'static str> {
    vec!["Hello World", "Rust & Friends", "  spaced  out  "]
}
```

**Что происходит:**

1. Файл теста начинается с `use p05_textkit::…`: он импортирует библиотеку по имени крейта (пакет называется
   `p05-textkit`, а в коде дефис становится подчёркиванием).
2. `mod common;` подключает вспомогательный модуль. Он лежит в `common/mod.rs`, **а не** в `tests/common.rs`:
   Cargo считает тестовым крейтом каждый файл прямо в `tests/`, а нам не нужен тестовый крейт «common» с нулём
   тестов в выводе.
3. У вспомогательного модуля функции `pub`, потому что файл теста — это другой крейт по отношению к нему.

Вывод `cargo test` теперь состоит из трёх частей: модульные тесты, затем каждый файл интеграционных тестов,
затем документационные тесты:

```text
     Running unittests src/lib.rs (...)
running 11 tests
...
test result: ok. 10 passed; 0 failed; 1 ignored; ...

     Running tests/api.rs (...)
running 2 tests
test public_api_parses_and_reports_errors ... ok
test slugs_of_sample_titles ... ok
test result: ok. 2 passed; 0 failed; 0 ignored; ...

   Doc-tests p05_textkit
running 0 tests
```

### Интеграционным тестам нужна библиотека

Интеграционные тесты импортируют ваш код, поэтому код должен быть **библиотекой**. Пакет только с `src/main.rs`
(бинарный крейт) импортировать нельзя:

```text
error[E0432]: unresolved import `demo2`
 --> tests/it.rs:1:5
  |
1 | use demo2::double;
  |     ^^^^^ use of unresolved module or unlinked crate `demo2`
```

Профессиональное решение, которое применяют `ripgrep` и большинство CLI-инструментов (CLI, Command-Line Interface — интерфейс командной строки): логика лежит в `src/lib.rs`, а
`src/main.rs` остаётся крошечным: он только читает аргументы, вызывает библиотеку и печатает. Именно это вы сделаете
в проекте minigrep (Урок 05.13).

### Приватные функции

Интеграционные тесты не могут вызывать приватные функции:

```text
error[E0603]: function `secret` is private
 --> tests/it.rs:4:23
  |
4 |     assert_eq!(demo3::secret(), 42);
  |                       ^^^^^^ private function
  |
note: the function `secret` is defined here
```

Модульные тесты в том же модуле могут. Нужно ли вообще тестировать приватные функции — предмет споров; хорошее
правило: сначала тестируйте **поведение, видимое через публичный API**. Приватный помощник тестируйте напрямую, если в
нём есть собственная хитрая логика.

## 6. Контроль качества: fmt, clippy, test

Три команды, запущенные в таком порядке, образуют *контроль качества* для любого проекта на Rust:

```bash
cargo fmt --check
cargo clippy -- -D warnings
cargo test
```

- `cargo fmt` форматирует код в стандартном стиле Rust. С `--check` он только сообщает о различиях и завершается с
  кодом ошибки, если они есть. (Без `--check` он исправляет файлы.)
- `cargo clippy` — официальный линтер: он находит код, который работает, но не идиоматичен или подозрителен.
  `-- -D warnings` превращает каждое предупреждение в ошибку.
- `cargo test` запускает тесты.

Например, этот код компилируется, и его тест проходит, но контроль качества его отклоняет:

```rust
pub fn is_blank(text: &str) -> bool {
    if text.trim().len() == 0 { return true; }
    return false;
}
```

```text
$ cargo fmt --check
Diff in src/lib.rs:1:
 pub fn is_blank(text: &str) -> bool {
-    if text.trim().len() == 0 { return true; }
+    if text.trim().len() == 0 {
+        return true;
+    }
     return false;
 }

$ cargo clippy -- -D warnings
error: unneeded `return` statement
 --> src/lib.rs:3:5
  |
3 |     return false;
  |     ^^^^^^^^^^^^
  |
help: remove `return`

error: length comparison to zero
 --> src/lib.rs:2:8
  |
2 |     if text.trim().len() == 0 { return true; }
  |        ^^^^^^^^^^^^^^^^^^^^^^ help: using `is_empty` is clearer and more explicit: `text.trim().is_empty()`
```

**Что это значит:** Clippy учит идиоматичному Rust: заканчивайте функцию выражением и используйте `is_empty()`.
Относитесь к его сообщениям как к бесплатному код-ревью. Исправленная функция:
`pub fn is_blank(text: &str) -> bool { text.trim().is_empty() }`.

> 💡 **Совет:** в RustRover Clippy можно запускать вместо обычной проверки сборки (Settings → Languages & Frameworks →
> Rust → External Linters), тогда предупреждения появляются в редакторе, пока вы печатаете. В первый раз настраивать
> это не нужно: достаточно трёх команд в терминале.

## 7. За пределами основ: что будет дальше

Они вам пока не нужны, но названия стоит знать, потому что вы встретите их в настоящих проектах (Этап 09 учит им
как следует):

- **Тестирование на основе свойств** (`proptest`, версия 1.12): вместо вручную выбранных примеров вы формулируете
  свойство («для любого `u64` `n` выполнено `parse_duration(&humanize(n)) == Ok(n)`»), а библиотека пробует сотни
  случайных входов и затем сжимает упавший вход до наименьшего.
- **Снапшот-тесты** (`insta`, версия 1.49): сохраняют вывод функции в файл и показывают разницу, когда он меняется.
- **`cargo nextest`** (версия 0.9): более быстрый запуск тестов с более понятным выводом, ставится командой
  `cargo install cargo-nextest`.
- **Покрытие** (`cargo-llvm-cov`, версия 0.9): показывает, какие строки выполнили ваши тесты. 100 % покрытия не доказывает
  правильность кода, но красная строка говорит, что её не тестирует ничто.
- **Мутационное тестирование** (`cargo-mutants`, версия 27): нарочно меняет ваш код (превращает `+` в `-`) и проверяет, что
  какой-то тест падает. Изменение, которое никто не заметил, означает недостающий тест.
- **CI** (Continuous Integration, непрерывная интеграция): запускает контроль качества при каждой отправке, чтобы никто
  не мог влить сломанный код. Этап 09 строит её с помощью GitHub Actions и GitLab CI.

## Частые ошибки

**1. Тест, который не может упасть.** Например, `assert!(true)` или цикл по пустому списку. Всегда один раз смотрите, как
новый тест падает (красный шаг).

**2. `#[should_panic]` без `expected`.** Любая паника, даже из-за опечатки, делает тест успешным.

**3. Сравнение дробных чисел через `assert_eq!`.** Используйте допуск (Раздел 3).

**4. Цикл в тесте без сообщения.** Когда падает один случай, вы не знаете какой. Добавьте `"input was {x:?}"`.

**5. Тесты, зависящие друг от друга или от общих файлов.** Они падают случайно. Дайте каждому тесту свои данные.

**6. Импорт бинарного крейта из `tests/`** (`E0432`, раздел 5). Перенесите логику в `src/lib.rs`.

**7. Тестирование только счастливого пути.** Проверяйте пустой вход, один элемент, максимум и каждый вариант ошибки.

**8. Изменение теста, чтобы он проходил.** Если код верен, а тест неверен, исправьте тест и поймите почему. Если тест
верен, ошибка в коде. Никогда не заглушайте падающий тест, не поняв его.

## Попробуйте сами

Примените урок на практике в `code/p05-textkit`, затем воспользуйтесь двумя инструментами Git, которые защищают качество
вашей истории.

1. Работайте «тест вперёд» в ветке `lesson/05-06-testing` (созданной в разделе 1): напишите падающий тест `slugify`,
   увидьте красный, сделайте зелёным и делайте коммит после каждого цикла:

   ```bash
   cd ~/projects/rust
   git add .
   git commit -m "test(p05): failing test for slugify"
   # ... implement ...
   git commit -am "feat(p05): implement slugify"
   ```

   Затем добавьте `parse_duration` и все тесты из этого урока, а также `tests/api.rs` и `tests/common/mod.rs`. Запустите
   контроль качества: `cargo fmt`, `cargo clippy -- -D warnings`, `cargo test`. Сделайте коммит
   `feat(p05): parse_duration with tests`.

2. **Установите хук pre-commit.** *Хук Git* — это скрипт, который Git запускает в определённый момент. Хук `pre-commit`
   запускается перед созданием каждого коммита, и если он завершается с ошибкой, **коммит отменяется**. Создайте файл
   `.git/hooks/pre-commit` (папка `.git` скрыта: используйте терминал):

   ```bash
   cat > .git/hooks/pre-commit <<'EOF'
   #!/bin/sh
   # Runs before every commit. A non-zero exit code cancels the commit.
   cd code/p05-textkit || exit 1
   cargo fmt --check || { echo "pre-commit: run cargo fmt first"; exit 1; }
   cargo clippy --quiet -- -D warnings || { echo "pre-commit: clippy found problems"; exit 1; }
   cargo test --quiet || { echo "pre-commit: tests failed"; exit 1; }
   EOF
   chmod +x .git/hooks/pre-commit
   ```

   `chmod +x` делает файл исполняемым, то есть разрешает запускать его как программу. Теперь попробуйте закоммитить
   изменение, которое ломает `slugify` (например, уберите приведение к нижнему регистру):

   ```bash
   git add -A
   git commit -m "refactor: simplify slugify"
   ```

   ```text
   assertion `left == right` failed
     left: "Hello-World"
    right: "hello-world"

   failures:
       tests::slugify_collapses_punctuation_and_trims_dashes
       tests::slugify_keeps_digits_and_non_ascii_letters
       tests::slugify_lowercases_and_joins_words_with_dashes

   test result: FAILED. 7 passed; 3 failed; 1 ignored; 0 measured; 0 filtered out; finished in 0.00s

   error: test failed, to rerun pass `--lib`
   pre-commit: tests failed
   ```

   `git log --oneline` не показывает нового коммита: хук защитил вашу историю. Отмените плохое изменение
   (`git restore code/p05-textkit/src/lib.rs`, а если оно было подготовлено — сначала `git restore --staged`). Два
   замечания: хук живёт в `.git/` и **не** передаётся через репозиторий (CI, Этап 09, — общая версия); а в
   экстренном случае `git commit --no-verify` его пропускает. Используйте это только осознанно.

3. **Найдите ошибку с помощью `git bisect`.** Представьте, что сегодня вы заметили падающий тест, а последние 30
   коммитов слились в пятно. `git bisect` находит первый плохой коммит *двоичным поиском*: он переключается на
   коммит в середине диапазона, вы говорите «хороший» или «плохой», и он снова делит диапазон пополам. Для 30
   коммитов нужно около пяти шагов вместо тридцати. С тестом это может быть полностью автоматическим:

   ```bash
   git bisect start
   git bisect bad                      # the current commit is broken
   git bisect good <hash-of-an-old-good-commit>
   git bisect run sh -c 'cd code/p05-textkit && cargo test --quiet'
   ```

   `git bisect run` запускает вашу команду на каждом шаге: код выхода 0 означает «хороший», ненулевой (от 1 до 127,
   кроме 125) — «плохой». В конце он печатает виновника:

   ```text
   Bisecting: 1 revision left to test after this (roughly 1 step)
   [0dde25671b6ff3128c8cea15b4489765f8843614] refactor(textkit): simplify slugify
   ...
   0dde25671b6ff3128c8cea15b4489765f8843614 is the first bad commit
   commit 0dde25671b6ff3128c8cea15b4489765f8843614
   Author: Student <s@example.com>
       refactor(textkit): simplify slugify
    code/p05-textkit/src/lib.rs | 4 +---
    1 file changed, 1 insertion(+), 3 deletions(-)
   bisect found first bad commit
   ```

   Когда закончите, `git bisect reset` возвращает вас в ветку, где вы были. Bisect хорошо работает только когда
   **коммиты небольшие и каждый коммит собирается**: ещё одна причина привычек из Уроков 05.2 и 05.4. Чтобы потренироваться,
   сделайте пять коммитов в одноразовой ветке, сломайте тест в третьем и дайте `git bisect run` его найти.

4. Слейте ветку урока в `main`, как раньше.

## Упражнения

### Упражнение 1 — Напишите недостающие тесты (легко)

Функция ниже работает. Напишите для неё тесты в `#[cfg(test)] mod tests`: хотя бы один обычный палиндром, один с
игнорированием регистра и знаков препинания, два не-палиндрома и граничные случаи (пустой текст, одна буква, текст без букв).

```rust
pub fn is_palindrome(text: &str) -> bool {
    let cleaned: Vec<char> = text
        .chars()
        .filter(|c| c.is_alphanumeric())
        .flat_map(|c| c.to_lowercase())
        .collect();
    cleaned.iter().eq(cleaned.iter().rev())
}
```

<details>
<summary>Подсказка</summary>

Сначала подумайте о граничных случаях: `""`, `"a"`, `"?!"` (букв нет вообще: очищенный текст пуст, поэтому ответ `true`),
`"Aa"` (разный регистр). Используйте `assert!(is_palindrome(...))` и `assert!(!is_palindrome(...))`.

</details>

<details>
<summary>Ответ</summary>

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn palindromes_ignore_case_and_punctuation() {
        assert!(is_palindrome("level"));
        assert!(is_palindrome("A man, a plan, a canal: Panama"));
        assert!(is_palindrome("Aa"));
    }

    #[test]
    fn non_palindromes() {
        assert!(!is_palindrome("hello"));
        assert!(!is_palindrome("ab"));
    }

    #[test]
    fn empty_and_single_character_texts_are_palindromes() {
        assert!(is_palindrome(""));
        assert!(is_palindrome("a"));
        assert!(is_palindrome("?!"));
    }
}
```

`cargo test` сообщает `3 passed`.

Почему это работает: тесты покрывают обычные входы и границы. Решение, что `"?!"` — палиндром (пустой очищенный текст),
теперь задокументировано тестом, а это полезно: если кто-то изменит правило, тест об этом скажет.

</details>

### Упражнение 2 — Проверьте пути ошибок (средне)

Напишите тесты для этой функции. Покройте корректный порт, порт с пробелами вокруг, граничные значения `0`, `65535`,
`70000`, пустой текст, нечисловой текст и отрицательное число. Для `70000` проверьте, что сообщение об ошибке
содержит «too large».

```rust
pub fn parse_port(text: &str) -> Result<u16, String> {
    match text.trim().parse::<u16>() {
        Ok(0) => Err(String::from("port 0 is not allowed")),
        Ok(port) => Ok(port),
        Err(e) => Err(format!("invalid port {text:?}: {e}")),
    }
}
```

<details>
<summary>Подсказка</summary>

Для успешных случаев используйте `assert_eq!(parse_port("8080"), Ok(8080))`. Для неудач —
`assert!(parse_port("0").is_err())`. Чтобы заглянуть внутрь ошибки, используйте `parse_port("70000").unwrap_err()`,
который возвращает `String`, затем `message.contains("too large")`. Добавьте сообщение вторым аргументом `assert!`, чтобы
при падении оно было видно.

</details>

<details>
<summary>Ответ</summary>

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn valid_ports() {
        assert_eq!(parse_port("8080"), Ok(8080));
        assert_eq!(parse_port(" 443 "), Ok(443));
        assert_eq!(parse_port("65535"), Ok(65535));
    }

    #[test]
    fn invalid_ports_are_errors() {
        assert!(parse_port("0").is_err());
        assert!(parse_port("").is_err());
        assert!(parse_port("abc").is_err());
        assert!(parse_port("-1").is_err());
        let message = parse_port("70000").unwrap_err();
        assert!(message.contains("too large"), "message was: {message}");
    }
}
```

`cargo test` сообщает `2 passed`.

Почему это работает: `65535` — наибольший допустимый `u16` (граница), `70000` — сразу за границей, а `0` — особое
значение, которое функция отвергает намеренно. Граничные значения ловят ошибки «на единицу».

</details>

### Упражнение 3 — Табличный тест (средне)

Напишите один тест для `slugify`, который проходит циклом по как минимум шести парам `(вход, ожидаемое)`, включая
`"C++ & Rust"` (→ `"c-rust"`), только знаки препинания (`"---"` → `""`), дату `"2026-11-17"` (→ без изменений) и буквы
не английского алфавита. Сообщение при падении должно показывать, какой вход упал.

<details>
<summary>Подсказка</summary>

`let cases = [("Hello World", "hello-world"), …];`, затем `for (input, expected) in cases { assert_eq!(slugify(input), expected, "slugify({input:?})"); }`.

</details>

<details>
<summary>Ответ</summary>

```rust
    #[test]
    fn slugify_table() {
        let cases = [
            ("Hello World", "hello-world"),
            ("  many   spaces ", "many-spaces"),
            ("C++ & Rust", "c-rust"),
            ("2026-11-17", "2026-11-17"),
            ("---", ""),
            ("ÅÄÖ", "åäö"),
        ];
        for (input, expected) in cases {
            assert_eq!(slugify(input), expected, "slugify({input:?})");
        }
    }
```

Почему это работает: дефис в `"2026-11-17"` — знак препинания, он заменяется одним дефисом, что возвращает тот же
текст. Если случай падает, сообщение `slugify("C++ & Rust")` вместе со значениями left/right точно его указывает.
Чтобы добавить случай, добавляется одна строка.

</details>

### Упражнение 4 — TDD: обратное к `parse_duration` (сложно)

Работайте «тест вперёд». Напишите `pub fn humanize(total_seconds: u64) -> String`, которая превращает `5415` в
`"1h30m15s"`, `0` — в `"0s"`, `3600` — в `"1h"`, а `61` — в `"1m1s"` (нулевые части опускаются). **Сначала** напишите тесты и
посмотрите, как они падают (используйте `todo!()`), затем реализуйте. Добавьте тест, что
`parse_duration(&humanize(n)) == Ok(n)` для нескольких значений `n`. Делайте коммит после красного и после зелёного.

<details>
<summary>Подсказка</summary>

Часы — это `total / 3600`, минуты — `(total % 3600) / 60`, секунды — `total % 60`. Стройте результат в `String`: добавьте
`"{h}h"`, если `h > 0`, `"{m}m"`, если `m > 0`, и `"{s}s"`, если `s > 0` **или** строка всё ещё пуста (так `0` становится `"0s"`).

</details>

<details>
<summary>Ответ</summary>

```rust
/// Turns a number of seconds into text such as "1h30m15s". The opposite of `parse_duration`.
pub fn humanize(total_seconds: u64) -> String {
    let hours = total_seconds / 3600;
    let minutes = (total_seconds % 3600) / 60;
    let seconds = total_seconds % 60;
    let mut out = String::new();
    if hours > 0 {
        out.push_str(&format!("{hours}h"));
    }
    if minutes > 0 {
        out.push_str(&format!("{minutes}m"));
    }
    if seconds > 0 || out.is_empty() {
        out.push_str(&format!("{seconds}s"));
    }
    out
}

// add these two tests to your existing `mod tests`
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn humanize_examples() {
        assert_eq!(humanize(5415), "1h30m15s");
        assert_eq!(humanize(0), "0s");
        assert_eq!(humanize(3600), "1h");
        assert_eq!(humanize(61), "1m1s");
    }

    #[test]
    fn humanize_and_parse_duration_are_inverse() {
        for seconds in [0, 1, 59, 60, 61, 3599, 3600, 86_399, 1_000_000] {
            assert_eq!(parse_duration(&humanize(seconds)), Ok(seconds));
        }
    }
}
```

Запуск `cargo test` показывает, что все тесты зелёные (в проверенном проекте: `12 passed` среди модульных тестов, а
игнорируемый медленный тест считается отдельно).

Почему это работает: второй тест — крошечная версия *свойства* («преобразование туда и обратно возвращает исходное»),
и это идея, на которой держится тестирование на основе свойств (Раздел 7). Функция `parse_duration` из урока
используется повторно, поэтому этот тест защищает и её. (Если вы помещаете тесты в существующий `mod tests`, удалите
повторяющиеся строки `mod tests` и `use super::*;`.)

</details>

## Вопросы для самопроверки

1. Почему стоит увидеть, как новый тест падает, прежде чем делать его зелёным?

<details>
<summary>Ответ</summary>

Это доказывает, что тест может упасть. Тест, который никогда не был красным, может ничего не проверять (например, утверждать то, что всегда истинно), и зелёный результат тогда ничего не значит.

</details>

2. Что проверяет `#[should_panic(expected = "text")]` и зачем добавлять `expected`?

<details>
<summary>Ответ</summary>

Тест проходит, только если происходит паника и сообщение паники содержит «text». Без `expected` любая паника (даже из-за опечатки или несвязанной ошибки) сделала бы тест успешным.

</details>

3. Почему тесты должны быть независимыми друг от друга?

<details>
<summary>Ответ</summary>

Cargo запускает тесты параллельно и без гарантированного порядка. Тесты с общим состоянием (глобальная переменная, файл) могут падать случайно (нестабильные тесты) или зависеть от порядка запуска.

</details>

4. Чем модульный тест отличается от интеграционного в Rust?

<details>
<summary>Ответ</summary>

Модульный тест лежит в том же файле (`#[cfg(test)] mod tests`) и видит приватные элементы. Интеграционный тест — файл в `tests/`, компилируется как отдельный крейт и может использовать только публичный API вашей библиотеки.

</details>

5. Почему нельзя писать интеграционные тесты для пакета, в котором есть только `src/main.rs`?

<details>
<summary>Ответ</summary>

Интеграционные тесты импортируют ваш код по имени крейта, а бинарный крейт импортировать нельзя (`E0432`). Перенесите логику в `src/lib.rs`, а `main.rs` оставьте тонкой обёрткой.

</details>

6. Как сравнивать два значения `f64` в тесте и почему?

<details>
<summary>Ответ</summary>

Проверяйте, что абсолютная разность меньше небольшого допуска, например `(a - b).abs() < 1e-9`. Арифметика с плавающей точкой обычно неточна (`0.1 + 0.2 != 0.3`), поэтому `assert_eq!` упал бы на безобидном округлении.

</details>

7. Что делает `git bisect run <команда>`?

<details>
<summary>Ответ</summary>

Автоматически находит первый плохой коммит: многократно переключается на коммит между известным хорошим и известным плохим, запускает команду, считает код выхода 0 хорошим, а ненулевой — плохим, и делит диапазон пополам, пока не определит виновный коммит.

</details>

## Итоги

- Работайте «тест вперёд», когда можете: красный (падающий тест), зелёный (самый простой код), рефакторинг (улучшение при зелёных тестах).
- Тестируйте счастливый путь, каждый вариант ошибки, границы (пусто, ноль, максимум) и переполнение; дробные числа сравнивайте с допуском.
- `#[should_panic(expected = …)]`, тесты с `Result` и `#[ignore]` решают особые случаи; `cargo test имя`, `-- --nocapture`, `-- --list`, `-- --ignored` управляют запуском.
- Модульные тесты лежат рядом с кодом и видят приватное; интеграционные лежат в `tests/`, требуют библиотечного крейта и видят только публичный API; документационные тесты — в следующем уроке.
- Тесты идут параллельно, поэтому должны быть независимыми, детерминированными и иметь хорошие сообщения при падении.
- Контроль качества — это `cargo fmt --check`, `cargo clippy -- -D warnings`, `cargo test`; хук `pre-commit` может запускать его перед каждым коммитом.
- `git bisect run cargo test` находит коммит, который что-то сломал, если коммиты небольшие и каждый собирается.

## Источники и что почитать

- [The Rust Programming Language — 11. Writing Automated Tests](https://doc.rust-lang.org/book/ch11-00-testing.html) — официальная глава о тестах, включая `should_panic` и `ignore`.
- [The Rust Programming Language — 11.3 Test Organization](https://doc.rust-lang.org/book/ch11-03-test-organization.html) — модульные и интеграционные тесты, `tests/common/mod.rs`.
- [The Cargo Book — cargo test](https://doc.rust-lang.org/cargo/commands/cargo-test.html) — все параметры `cargo test`.
- [Документация Clippy](https://doc.rust-lang.org/stable/clippy/) — как использовать и настраивать Clippy; список проверок есть на его сайте.
- [Макрос std::matches!](https://doc.rust-lang.org/std/macro.matches.html) — проверка значения на соответствие образцу.
- [Документация Git — git bisect](https://git-scm.com/docs/git-bisect) и [Хуки Git](https://git-scm.com/docs/githooks) — двоичный поиск ошибок и список хуков.
- [proptest на docs.rs](https://docs.rs/proptest) — тестирование на основе свойств, на будущее.

---

[🏠 Начало](../START-HERE.md) · [← 05.5 Итераторы](05-iterators.md) · [📋 Этап 05](README.md) · [05.7 Документация →](07-documentation.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
