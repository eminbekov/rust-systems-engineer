<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 05.10 Потоки, каналы и общее состояние](10-threads-channels-and-shared-state.md) · [📋 Этап 05](README.md) · [05.12 Основы бенчмарков →](12-benchmarking-basics.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/05-rust-intermediate/11-practical-crates-serde-clap-errors.md) <!-- nav -->

# Урок 05.11 — Практичные крейты: serde, clap, thiserror и anyhow

> **Этап 05 — Rust: средний уровень** · Урок 11 из 14 · ⏱ около 4,5 часа (3 занятия)

## Что вы узнаете

- Как превращать структуры Rust в JSON (и обратно) с помощью `serde` и `serde_json` и как читать настройки в формате TOML.
- Как создать инструмент командной строки с аргументами, опциями и подкомандами с помощью `clap`.
- Как описывать аккуратные типы ошибок для библиотеки с помощью `thiserror`.
- Как сообщать об ошибках с контекстом в приложении с помощью `anyhow`.
- Как эти четыре крейта работают вместе в одной небольшой программе, `taskbox`.

## Перед началом

- Вы прошли [Урок 05.10 — Потоки, каналы и общее состояние](10-threads-channels-and-shared-state.md).
- Вы умеете добавлять зависимости и фичи: [Урок 05.8 — Cargo: зависимости и версии](08-cargo-dependencies-and-versions.md).
- Вы знаете обработку ошибок через `Result`, `?` и собственные ошибки: [Урок 04.10 — Обработка ошибок](../04-structs-enums-collections-errors/10-error-handling.md).
- Вы знаете структуры и перечисления ([Урок 04.1](../04-structs-enums-collections-errors/01-structs.md), [Урок 04.3](../04-structs-enums-collections-errors/03-enums.md)) и трейты ([Урок 05.2](02-traits.md)).
- Ваш компьютер должен быть подключён к интернету: Cargo скачивает крейты.

## Новые слова

| Слово | Значение |
|---|---|
| **сериализация** (serialization) | Превращение значения в памяти (структуры) в текст или байты, которые можно сохранить или отправить. |
| **десериализация** (deserialization) | Обратное: превращение текста или байтов обратно в значение. |
| **serde** (SERialization / DEserialization) | Стандартный фреймворк сериализации в Rust. Он определяет трейты `Serialize` и `Deserialize`. |
| **JSON** (JavaScript Object Notation, «запись объектов JavaScript») | Популярный текстовый формат данных: `{"name": "Ana", "age": 30}`. |
| **TOML** (Tom's Obvious Minimal Language, «очевидный минимальный язык Тома») | Текстовый формат для файлов настроек. Его использует `Cargo.toml`. |
| **YAML** (YAML Ain't Markup Language, «YAML — не язык разметки») | Ещё один текстовый формат для настроек, который применяют многие инструменты. |
| **derive-макрос** (derive macro) | Строка `#[derive(...)]`. Компилятор запускает код, который пишет реализацию трейта за вас. (Собственные макросы вы напишете на [Этапе 06](../06-rust-advanced/08-procedural-macros.md).) |
| **атрибут** (attribute) | Строка `#[...]`, которая даёт компилятору или макросу дополнительную информацию о следующем элементе. |
| **CLI** (Command-Line Interface, интерфейс командной строки) | Программа, которой пользуются, вводя команду и аргументы в терминале. |
| **аргумент** (argument) | Слово, которое вы вводите после имени программы. **Позиционный аргумент** определяется своим местом. |
| **опция** (option) | Аргумент с именем, например `--count 3` или `-c 3`. |
| **флаг** (flag) | Опция без значения; она либо есть, либо нет, например `--shout`. |
| **подкоманда** (subcommand) | Команда внутри команды, например `git commit` или `cargo build`. |
| **код возврата** (exit code) | Число, которое программа возвращает оболочке при завершении. `0` означает успех; всё остальное — неудачу. |
| **ошибка библиотеки** (library error) | Тип ошибки крейта, которым пользуются другие. Он должен быть точным, чтобы вызывающий код мог реагировать на каждый случай. |
| **ошибка приложения** (application error) | Ошибка на верхнем уровне программы. Обычно её просто показывают пользователю с достаточным контекстом, чтобы понять причину. |
| **контекст** (context) | Короткое сообщение, добавленное к ошибке, которое говорит, что программа делала («не удалось загрузить задачи из tasks.json»). |
| **цепочка ошибок** (error chain) | Ошибка вместе с ошибками, которые её вызвали («не удалось загрузить…», потому что «файл не является корректным JSON», потому что «expected ident at line 1»). |
| **EOF** (End Of File, конец файла) | Точка, после которой входных данных больше нет. |

## 1. Зачем эти четыре крейта?

Стандартная библиотека Rust намеренно небольшая. Для повседневных задач сообщество создало крейты, которыми пользуется почти каждый проект. Эти четыре вы встретите почти в любой настоящей программе на Rust:

| Крейт | Задача | Где чаще всего |
|---|---|---|
| `serde` (+ `serde_json`, `toml`) | Преобразование структур в текстовые форматы и обратно. | библиотеки и приложения |
| `clap` | Чтение аргументов командной строки, написание `--help`. | приложения |
| `thiserror` | Описание типов ошибок с малым количеством кода. | **библиотеки** (и библиотечная часть приложения) |
| `anyhow` | Один универсальный тип ошибки с контекстом. | **приложения** (`main` и код рядом с ним) |

Простое правило для ошибок: **библиотека точно говорит вызывающему, что пошло не так (`thiserror`); приложение лишь сообщает об этом человеку (`anyhow`)**. Программа `taskbox` в разделе 6 использует оба подхода.

Версии, использованные в этом уроке (проверено 2026-11-17): `serde` 1.0, `serde_json` 1.0, `toml` 1.2, `clap` 4.6, `thiserror` 2.0, `anyhow` 1.0. Записывать нужно только короткую форму: `"1"`, `"4"`, `"2"`.

## 2. serde: превращение данных в текст и обратно

### Подготовка

Создайте проект и добавьте крейты (`derive` — это **фича** `serde`, см. [Урок 05.8](08-cargo-dependencies-and-versions.md)):

```bash
cd ~/projects/rust/code
cargo new p05-serde-lab
cd p05-serde-lab
cargo add serde@1 --features derive
cargo add serde_json@1
```

### Сериализация и десериализация структуры

```rust
use serde::{Deserialize, Serialize};

#[derive(Debug, Serialize, Deserialize)]
struct Book {
    title: String,
    pages: u32,
    tags: Vec<String>,
}

fn main() {
    let book = Book {
        title: String::from("Rust in Action"),
        pages: 456,
        tags: vec![String::from("rust"), String::from("systems")],
    };

    let json = serde_json::to_string(&book).unwrap();
    println!("{json}");

    let pretty = serde_json::to_string_pretty(&book).unwrap();
    println!("{pretty}");

    let text = r#"{"title":"Zero To Production","pages":528,"tags":["web"]}"#;
    let parsed: Book = serde_json::from_str(text).unwrap();
    println!("{parsed:?}");
}
```

```text
{"title":"Rust in Action","pages":456,"tags":["rust","systems"]}
{
  "title": "Rust in Action",
  "pages": 456,
  "tags": [
    "rust",
    "systems"
  ]
}
Book { title: "Zero To Production", pages: 528, tags: ["web"] }
```

**Что происходит:**

1. `#[derive(Serialize, Deserialize)]` — это **derive-макрос**. При компиляции он пишет код, который умеет превращать `Book` в данные и обратно. Реализации трейтов `Serialize` и `Deserialize` вы получаете бесплатно.
2. `serde_json::to_string(&book)` создаёт компактный текст JSON. `to_string_pretty` добавляет переносы строк и отступы для людей.
3. `r#"..."#` — это **сырая строка**: обратные косые черты и кавычки внутри не нужно экранировать.
4. `serde_json::from_str(text)` разбирает текст в значение. Тип берётся из аннотации `let parsed: Book`. `from_str` возвращает `Result`, потому что текст может быть неверным (мы используем `unwrap` только в коротких примерах).

Сам serde — лишь фреймворк (трейты и derive). Каждый формат живёт в своём крейте: `serde_json` для JSON, `toml` для TOML и другие для таких форматов, как MessagePack. Ваша структура с одним derive работает со всеми ними.

### Ошибки, которые вы увидите

Данным снаружи верить нельзя ([Урок 02.7](../02-programming-basics/07-reading-input-and-parsing.md): никогда не доверяйте вводу). `from_str` говорит точно, что не так:

```rust
use serde::Deserialize;

#[derive(Debug, Deserialize)]
struct Book {
    title: String,
    pages: u32,
}

fn main() {
    let missing = r#"{"title":"No pages"}"#;
    let wrong_type = r#"{"title":"Bad","pages":"many"}"#;
    let broken = r#"{"title":"Broken","pages":12"#;
    let negative = r#"{"title":"Neg","pages":-5}"#;

    for text in [missing, wrong_type, broken, negative] {
        match serde_json::from_str::<Book>(text) {
            Ok(book) => println!("ok: {} has {} pages", book.title, book.pages),
            Err(e) => println!("error: {e}"),
        }
    }
}
```

```text
error: missing field `pages` at line 1 column 20
error: invalid type: string "many", expected u32 at line 1 column 29
error: EOF while parsing an object at line 1 column 28
error: invalid value: integer `-5`, expected u32 at line 1 column 25
```

**Что происходит:** все четыре входа отвергнуты с понятным сообщением и позицией (строка и столбец). Проверку выполняют типы Rust: `pages: u32` означает, что значение должно быть целым числом, которое помещается в `u32`, поэтому `"many"` и `-5` — ошибки. Отсутствующее поле — тоже ошибка, если вы не сказали иначе (следующий раздел).

> 🔒 **Безопасность:** десериализация в точные типы — сильная защита. Ваша программа никогда не увидит строку там, где ждёт число. Но тип не проверяет *смысл*: `u32` со значением `4000000000` допустим для `pages`. Проверяйте диапазоны и бизнес-правила сами, как вы увидите на Этапе 10.

### Атрибуты: настройка формата

Реальные данные редко совпадают с вашими именами в Rust. Используйте атрибуты `#[serde(...)]`:

```rust
use serde::{Deserialize, Serialize};

#[derive(Debug, Serialize, Deserialize)]
struct Settings {
    #[serde(rename = "userName")]
    user_name: String,
    #[serde(default = "default_port")]
    port: u16,
    #[serde(default)]
    verbose: bool,
    #[serde(skip_serializing_if = "Option::is_none")]
    nickname: Option<String>,
}

fn default_port() -> u16 {
    8080
}

#[derive(Debug, Serialize, Deserialize, PartialEq)]
#[serde(rename_all = "lowercase")]
enum Level {
    Low,
    High,
}

fn main() {
    let text = r#"{"userName":"ana"}"#;
    let s: Settings = serde_json::from_str(text).unwrap();
    println!("{s:?}");
    println!("{}", serde_json::to_string(&s).unwrap());

    let with_nick = Settings { nickname: Some(String::from("A")), ..s };
    println!("{}", serde_json::to_string(&with_nick).unwrap());

    println!("{}", serde_json::to_string(&Level::High).unwrap());
    let l: Level = serde_json::from_str("\"low\"").unwrap();
    println!("{l:?}");
}
```

```text
Settings { user_name: "ana", port: 8080, verbose: false, nickname: None }
{"userName":"ana","port":8080,"verbose":false}
{"userName":"ana","port":8080,"verbose":false,"nickname":"A"}
"high"
Low
```

**Что происходит:**

1. `rename = "userName"` сопоставляет ключ JSON `userName` с полем Rust `user_name` (Rust использует `snake_case`, многие файлы JSON — `camelCase`).
2. `default = "default_port"` вызывает функцию, когда ключа нет, поэтому `port` становится `8080`. Простой `#[serde(default)]` берёт значение типа по умолчанию (`false` для `bool`).
3. `skip_serializing_if = "Option::is_none"` пропускает ключ, когда значение — `None`. Поэтому в первом выводе нет `nickname`, а во втором он есть.
4. `rename_all = "lowercase"` у перечисления записывает варианты `Low` и `High` как `"low"` и `"high"`.
5. `..s` — синтаксис обновления структуры: взять остальные поля из `s`.

### JSON без структуры

Когда форма заранее неизвестна, используйте `serde_json::Value` и макрос `json!`:

```rust
use serde_json::{json, Value};

fn main() {
    let v: Value = serde_json::from_str(r#"{"name":"ana","scores":[3,5,8]}"#).unwrap();
    println!("name = {}", v["name"]);
    println!("second score = {}", v["scores"][1]);
    println!("missing = {}", v["nope"]);

    let built = json!({ "ok": true, "count": 3 });
    println!("{built}");
}
```

```text
name = "ana"
second score = 5
missing = null
{"count":3,"ok":true}
```

`Value` — перечисление всех видов JSON (null, bool, число, строка, массив, объект). Обращение к несуществующему ключу даёт `null`, а не панику. Ключи объекта выводятся отсортированными (`count` раньше `ok`). Когда можете, предпочитайте типизированные структуры: они документируют данные, а компилятор проверяет ваш код.

### TOML для настроек

Добавьте `cargo add toml@1`. Тот же derive работает:

```rust
use serde::{Deserialize, Serialize};

#[derive(Debug, Serialize, Deserialize)]
struct Config {
    name: String,
    port: u16,
    features: Vec<String>,
}

fn main() {
    let text = r#"
name = "demo"
port = 8080
features = ["fast", "safe"]
"#;
    let config: Config = toml::from_str(text).unwrap();
    println!("{config:?}");
    println!("{}", toml::to_string(&config).unwrap());
}
```

```text
Config { name: "demo", port: 8080, features: ["fast", "safe"] }
name = "demo"
port = 8080
features = ["fast", "safe"]
```

> 📏 **Соглашение:** в мире Rust файлы настроек обычно пишут на TOML (его использует Cargo, и в нём есть комментарии), а обмен данными идёт в JSON. Избегайте старого крейта `serde_yaml`: он помечен как устаревший (deprecated), и автор больше его не поддерживает.

> ⚠️ **Частая ошибка:** `#[derive(Serialize)]` без фичи `derive`:
>
> ```text
> error: cannot find derive macro `Serialize` in this scope
>   --> src/main.rs:3:10
>   |
> 3 | #[derive(Serialize)]
>   |          ^^^^^^^^^
>   |
> note: `Serialize` is a trait, not a derive macro
> ```
>
> Трейт `Serialize` существует, но derive-макрос спрятан за фичей `derive`. Выполните `cargo add serde --features derive`.

## 3. clap: аргументы командной строки

Программа, запущенная из терминала, получает **аргументы**. Их можно прочитать через `std::env::args()` и разобрать самостоятельно, но тогда придётся писать текст справки, сообщения об ошибках и проверки. `clap` (Command Line Argument Parser — разборщик аргументов командной строки) делает всё это по структуре.

```bash
cd ~/projects/rust/code
cargo new greet
cd greet
cargo add clap@4 --features derive
```

```rust
use clap::Parser;

/// Greets people from the command line.
#[derive(Parser, Debug)]
#[command(version, about)]
struct Cli {
    /// Name of the person to greet
    name: String,

    /// How many times to greet
    #[arg(short, long, default_value_t = 1)]
    count: u32,

    /// Print in capital letters
    #[arg(short, long)]
    shout: bool,
}

fn main() {
    let cli = Cli::parse();
    for _ in 0..cli.count {
        let line = format!("Hello, {}!", cli.name);
        if cli.shout {
            println!("{}", line.to_uppercase());
        } else {
            println!("{line}");
        }
    }
}
```

**Что происходит:**

1. `#[derive(Parser)]` создаёт `Cli::parse()`, который читает настоящую командную строку, проверяет её и возвращает заполненный `Cli`. Если что-то не так, он печатает ошибку и завершает программу с кодом возврата 2.
2. У поля `name` нет атрибута, поэтому это **обязательный позиционный аргумент**.
3. `#[arg(short, long, default_value_t = 1)]` создаёт **опцию** с коротким именем `-c` и длинным `--count`. Тип `u32` проверяется автоматически. Если пользователь ничего не указал, берётся значение по умолчанию `1`.
4. `shout: bool` с `#[arg(short, long)]` становится **флагом**: `-s` или `--shout`.
5. Doc-комментарии (`///`) становятся текстами справки. `#[command(version, about)]` берёт версию из `Cargo.toml`, а описание — из doc-комментария структуры.

Запустите (`cargo build`, затем `./target/debug/greet` или `cargo run -- <аргументы>`; `--` отделяет аргументы самого Cargo от аргументов вашей программы):

```bash
./target/debug/greet --help
```

```text
Greets people from the command line

Usage: greet [OPTIONS] <NAME>

Arguments:
  <NAME>  Name of the person to greet

Options:
  -c, --count <COUNT>  How many times to greet [default: 1]
  -s, --shout          Print in capital letters
  -h, --help           Print help
  -V, --version        Print version
```

```bash
./target/debug/greet --version
./target/debug/greet Ana -c 2 -s
```

```text
greet 0.1.0
HELLO, ANA!
HELLO, ANA!
```

Неправильное использование даёт полезные ошибки бесплатно:

```bash
./target/debug/greet
./target/debug/greet Ana --count many
./target/debug/greet Ana --color
```

```text
error: the following required arguments were not provided:
  <NAME>

Usage: greet <NAME>

For more information, try '--help'.
error: invalid value 'many' for '--count <COUNT>': invalid digit found in string

For more information, try '--help'.
error: unexpected argument '--color' found

  tip: to pass '--color' as a value, use '-- --color'

Usage: greet <NAME>

For more information, try '--help'.
```

Каждый из этих запусков завершился с кодом возврата 2. (Проверьте это командой `echo $?` сразу после команды.)

### Подкоманды

Большие инструменты группируют действия: `git commit`, `cargo build`. В `clap` подкоманда — это `enum` с пометкой `#[derive(Subcommand)]`. Полностью вы увидите это в разделе 6:

```rust
#[derive(Subcommand)]
enum Command {
    /// Add a new task
    Add {
        /// The text of the task
        title: String,
    },
    /// Show the tasks
    List {
        /// Show finished tasks too
        #[arg(short, long)]
        all: bool,
    },
}
```

Каждый вариант — это подкоманда; его поля — её аргументы.

### Тестирование разборщика аргументов

Чтобы протестировать разборщик, программу запускать не нужно. `try_parse_from` берёт аргументы из списка и возвращает `Result`, а не завершает программу:

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn parses_name_and_defaults() {
        let cli = Cli::try_parse_from(["greet", "Ana"]).unwrap();
        assert_eq!(cli.name, "Ana");
        assert_eq!(cli.count, 1);
        assert!(!cli.shout);
    }

    #[test]
    fn parses_options() {
        let cli = Cli::try_parse_from(["greet", "Ana", "-c", "3", "--shout"]).unwrap();
        assert_eq!(cli.count, 3);
        assert!(cli.shout);
    }

    #[test]
    fn missing_name_is_an_error() {
        assert!(Cli::try_parse_from(["greet"]).is_err());
    }
}
```

```text
test tests::parses_name_and_defaults ... ok
test tests::parses_options ... ok
test tests::missing_name_is_an_error ... ok

test result: ok. 3 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

Первый элемент списка — имя программы (`argv[0]`), как в настоящей командной строке. (В [Уроке 05.13](13-project-minigrep.md) вы также будете тестировать целую программу снаружи.)

> 📏 **Соглашение:** пишите doc-комментарий для каждого аргумента: это и документация в коде, и текст `--help`. Следуйте привычкам командной строки: `-h`/`--help`, `-V`/`--version`, длинные опции в `kebab-case`, ошибки в поток ошибок и ненулевой код возврата при неудаче.

## 4. thiserror: типы ошибок для библиотек

В [Уроке 04.10](../04-structs-enums-collections-errors/10-error-handling.md) вы писали ошибку вручную: `enum`, `impl Display`, `impl Error` и `impl From<...>` для каждой исходной ошибки. Это много повторяющегося кода. `thiserror` пишет его за вас с помощью derive-макроса.

```bash
cargo add thiserror@2
```

```rust
use std::fmt;
use thiserror::Error;

// By hand (from Lesson 04.10):
#[derive(Debug)]
enum ByHand {
    TooShort(usize),
    BadChar(char),
}

impl fmt::Display for ByHand {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        match self {
            ByHand::TooShort(n) => write!(f, "password too short: {n} characters"),
            ByHand::BadChar(c) => write!(f, "bad character {c:?}"),
        }
    }
}

impl std::error::Error for ByHand {}

// With thiserror: the same, in half the lines.
#[derive(Debug, Error)]
enum PasswordError {
    #[error("password too short: {0} characters")]
    TooShort(usize),
    #[error("bad character {0:?}")]
    BadChar(char),
    #[error("could not read the number")]
    Number(#[from] std::num::ParseIntError),
}

fn check(pw: &str) -> Result<(), PasswordError> {
    if pw.len() < 8 {
        return Err(PasswordError::TooShort(pw.len()));
    }
    if let Some(c) = pw.chars().find(|c| c.is_whitespace()) {
        return Err(PasswordError::BadChar(c));
    }
    Ok(())
}

fn parse_len(text: &str) -> Result<usize, PasswordError> {
    let n: usize = text.parse()?;
    Ok(n)
}

fn main() {
    println!("{}", ByHand::TooShort(3));
    println!("{}", ByHand::BadChar(' '));
    println!("{:?}", check("abc"));
    println!("{}", check("abc").unwrap_err());
    println!("{}", check("abcd efgh").unwrap_err());
    println!("{:?}", check("abcdefgh"));
    let e = parse_len("x1").unwrap_err();
    println!("{e}");
    println!("source: {}", std::error::Error::source(&e).unwrap());
}
```

```text
password too short: 3 characters
bad character ' '
Err(TooShort(3))
password too short: 3 characters
bad character ' '
Ok(())
could not read the number
source: invalid digit found in string
```

**Что происходит:**

1. `#[derive(Error)]` реализует `std::error::Error` для перечисления. Ему также нужен `Debug` (`#[derive(Debug, Error)]`).
2. `#[error("...")]` у каждого варианта пишет сообщение `Display`. `{0}` — первое поле кортежного варианта; именованные поля пишутся `{name}`. `{0:?}` использует форму `Debug`.
3. `#[from]` у поля делает две вещи: реализует `From<ParseIntError> for PasswordError` (поэтому `?` преобразует ошибку автоматически, как в `parse_len`) и помечает поле как **источник** (source) ошибки. Поэтому `Error::source(&e)` возвращает исходное сообщение.
4. Вывод написанного вручную `ByHand` и версии с `thiserror` одинаков. Derive лишь убирает шаблонный код.

Если забыть `#[from]`, `?` не сможет преобразовать ошибку, и компилятор скажет об этом:

```rust
#[derive(Debug, Error)]
enum MyError {
    #[error("bad number")]
    Number(std::num::ParseIntError),
}

fn parse(text: &str) -> Result<i32, MyError> {
    let n: i32 = text.parse()?;
    Ok(n)
}
```

```text
error[E0271]: type mismatch resolving `<i32 as FromStr>::Err == MyError`
  --> src/main.rs:10:23
   |
10 |     let n: i32 = text.parse()?;
   |                       ^^^^^ expected `MyError`, found `ParseIntError`
```

Исправление: напишите `Number(#[from] std::num::ParseIntError)`.

> 📏 **Соглашение:** одно перечисление ошибок на модуль или крейт, с вариантами, названными по тому, *что пошло не так* (`NotFound`, `Io`, `Json`), а не по крейту, который ошибку породил. Начинайте сообщения со строчной буквы и не ставьте точку в конце: ошибки часто соединяют цепочкой через `: `.

## 5. anyhow: обработка ошибок в приложениях

В `main` редко нужно разбирать случаи ошибок через `match`. Нужно остановиться, показать понятное сообщение с причиной и вернуть код возврата, означающий неудачу. `anyhow` даёт один тип ошибки, `anyhow::Error`, способный хранить любую ошибку, и псевдоним `anyhow::Result<T>`.

```bash
cargo add anyhow@1
```

```rust
use anyhow::{anyhow, bail, ensure, Context, Result};

fn parse_port(text: &str) -> Result<u16> {
    let port: u16 = text
        .trim()
        .parse()
        .with_context(|| format!("`{text}` is not a valid port number"))?;
    ensure!(port >= 1024, "port {port} is reserved for the system");
    Ok(port)
}

fn pick(name: &str) -> Result<u32> {
    match name {
        "small" => Ok(1),
        "big" => Ok(100),
        "" => bail!("the name is empty"),
        other => Err(anyhow!("unknown size `{other}`")),
    }
}

fn main() {
    for text in ["8080", "80", "http"] {
        match parse_port(text) {
            Ok(p) => println!("port = {p}"),
            Err(e) => {
                println!("short : {e}");
                println!("chain : {e:#}");
            }
        }
    }
    println!("{:?}", pick("small").unwrap());
    for name in ["", "huge"] {
        println!("{}", pick(name).unwrap_err());
    }

    let e = parse_port("http").unwrap_err();
    println!("debug view:\n{e:?}");
    println!(
        "is ParseIntError inside? {}",
        e.downcast_ref::<std::num::ParseIntError>().is_some()
    );
}
```

```text
port = 8080
short : port 80 is reserved for the system
chain : port 80 is reserved for the system
short : `http` is not a valid port number
chain : `http` is not a valid port number: invalid digit found in string
1
the name is empty
unknown size `huge`
debug view:
`http` is not a valid port number

Caused by:
    invalid digit found in string
is ParseIntError inside? true
```

**Что происходит:**

1. `Result<u16>` здесь — это `anyhow::Result<u16>`, краткая запись `Result<u16, anyhow::Error>`.
2. `.with_context(|| ...)` добавляет сообщение к ошибке, если она есть. Замыкание выполняется только при неудаче, поэтому на успешном пути текст не строится. (`.context("fixed text")` делает то же для фиксированного сообщения.) Работает для любого `Result`, ошибка которого реализует `std::error::Error`, и для `Option`.
3. `?` автоматически преобразует `ParseIntError` в `anyhow::Error`, сохраняя исходную ошибку внутри.
4. `ensure!(condition, "message")` возвращает ошибку, если условие ложно. `bail!("message")` сразу возвращает ошибку. `anyhow!("message")` создаёт значение ошибки. Все три принимают аргументы в стиле `format!`.
5. `{e}` печатает только самое внешнее сообщение. `{e:#}` печатает всю **цепочку** в одной строке. `{e:?}` печатает сообщение и список «Caused by:». Когда `main` возвращает `Err(e)`, Rust печатает форму `{e:?}`.
6. `downcast_ref` позволяет спросить: «исходной ошибкой была `ParseIntError`?». Это запасной выход, когда всё-таки нужно реагировать на один случай.

### Библиотека или приложение?

| | Библиотека (`thiserror`) | Приложение (`anyhow`) |
|---|---|---|
| Кто читает ошибку? | Другой код. Он разбирает варианты через `match`. | Человек, в терминале или в логе. |
| Тип ошибки | Точный `enum` | `anyhow::Error` |
| Контекст | В вариантах | Сообщения `.context(...)` |
| Типичное место | `src/lib.rs`, модули | `fn main()` и код рядом |

> ⚠️ **Частая ошибка:** возвращать `anyhow::Error` из функции библиотеки. Вызывающий код больше не видит случаи без `downcast`. Библиотеки должны возвращать собственный тип ошибки. Приложения могут использовать `anyhow` везде, потому что дальше по цепочке никому не нужно делать `match`.

## 6. Собираем всё вместе: taskbox

`taskbox` — небольшой список задач в терминале. Он хранит задачи в файле JSON. Он использует все четыре крейта и имеет обычную форму инструмента на Rust: **библиотека** с логикой и тонкий **бинарник** (`main.rs`) с частью для командной строки.

```text
taskbox/
├── Cargo.toml
└── src/
    ├── lib.rs     ← логика: Task, TaskError (thiserror), serde, тесты
    └── main.rs    ← командная строка: clap + anyhow
```

Создайте его:

```bash
cd ~/projects/rust/code
cargo new --lib taskbox
cd taskbox
cargo add serde@1 --features derive
cargo add serde_json@1 thiserror@2 anyhow@1
cargo add clap@4 --features derive
```

`cargo new --lib` создаёт `src/lib.rs`. В пакете могут быть и `src/lib.rs`, и `src/main.rs`; `main.rs` может использовать библиотеку по имени пакета (`taskbox`). Теперь в `Cargo.toml` есть:

```toml
[dependencies]
anyhow = "1"
clap = { version = "4", features = ["derive"] }
serde = { version = "1", features = ["derive"] }
serde_json = "1"
thiserror = "2"
```

`src/lib.rs`:

```rust
use std::fs;
use std::path::Path;

use serde::{Deserialize, Serialize};
use thiserror::Error;

#[derive(Debug, PartialEq, Serialize, Deserialize)]
pub struct Task {
    pub id: u32,
    pub title: String,
    pub done: bool,
}

#[derive(Debug, Error)]
pub enum TaskError {
    #[error("cannot read or write the task file")]
    Io(#[from] std::io::Error),
    #[error("the task file is not valid JSON")]
    Json(#[from] serde_json::Error),
    #[error("there is no task with id {0}")]
    NotFound(u32),
}

/// Reads all tasks from `path`. A missing file means "no tasks yet".
pub fn load(path: &Path) -> Result<Vec<Task>, TaskError> {
    if !path.exists() {
        return Ok(Vec::new());
    }
    let text = fs::read_to_string(path)?;
    Ok(serde_json::from_str(&text)?)
}

/// Writes all tasks to `path`.
pub fn save(path: &Path, tasks: &[Task]) -> Result<(), TaskError> {
    let text = serde_json::to_string_pretty(tasks)?;
    fs::write(path, text)?;
    Ok(())
}

/// Adds a task and returns its new id.
pub fn add(tasks: &mut Vec<Task>, title: &str) -> u32 {
    let id = tasks.iter().map(|t| t.id).max().unwrap_or(0) + 1;
    tasks.push(Task {
        id,
        title: title.to_string(),
        done: false,
    });
    id
}

/// Marks a task as done.
pub fn complete(tasks: &mut [Task], id: u32) -> Result<(), TaskError> {
    let task = tasks
        .iter_mut()
        .find(|t| t.id == id)
        .ok_or(TaskError::NotFound(id))?;
    task.done = true;
    Ok(())
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn add_gives_increasing_ids() {
        let mut tasks = Vec::new();
        assert_eq!(add(&mut tasks, "first"), 1);
        assert_eq!(add(&mut tasks, "second"), 2);
        assert_eq!(tasks.len(), 2);
    }

    #[test]
    fn complete_marks_done() {
        let mut tasks = Vec::new();
        let id = add(&mut tasks, "write lesson");
        complete(&mut tasks, id).unwrap();
        assert!(tasks[0].done);
    }

    #[test]
    fn complete_unknown_id_is_an_error() {
        let mut tasks = Vec::new();
        let err = complete(&mut tasks, 7).unwrap_err();
        assert_eq!(err.to_string(), "there is no task with id 7");
    }

    #[test]
    fn json_round_trip() {
        let tasks = vec![Task { id: 1, title: String::from("a"), done: true }];
        let text = serde_json::to_string(&tasks).unwrap();
        let back: Vec<Task> = serde_json::from_str(&text).unwrap();
        assert_eq!(tasks, back);
    }
}
```

`src/main.rs`:

```rust
use std::path::PathBuf;

use anyhow::{Context, Result};
use clap::{Parser, Subcommand};
use taskbox::{add, complete, load, save};

/// A tiny task list that is stored in a JSON file.
#[derive(Parser)]
#[command(version, about)]
struct Cli {
    /// The file that stores the tasks
    #[arg(short, long, default_value = "tasks.json")]
    file: PathBuf,

    #[command(subcommand)]
    command: Command,
}

#[derive(Subcommand)]
enum Command {
    /// Add a new task
    Add {
        /// The text of the task
        title: String,
    },
    /// Show the tasks
    List {
        /// Show finished tasks too
        #[arg(short, long)]
        all: bool,
    },
    /// Mark a task as done
    Done {
        /// The id of the task
        id: u32,
    },
}

fn main() -> Result<()> {
    let cli = Cli::parse();
    let mut tasks = load(&cli.file)
        .with_context(|| format!("failed to load tasks from {}", cli.file.display()))?;

    match cli.command {
        Command::Add { title } => {
            let id = add(&mut tasks, &title);
            save(&cli.file, &tasks)?;
            println!("added task {id}");
        }
        Command::List { all } => {
            for task in tasks.iter().filter(|t| all || !t.done) {
                let mark = if task.done { "x" } else { " " };
                println!("[{mark}] {}: {}", task.id, task.title);
            }
        }
        Command::Done { id } => {
            complete(&mut tasks, id)?;
            save(&cli.file, &tasks)?;
            println!("task {id} is done");
        }
    }
    Ok(())
}
```

**Что происходит:**

1. `lib.rs` ничего не знает о терминале. `Task` использует `serde`, чтобы сохраняться; `TaskError` использует `thiserror` с `#[from]`, так что `?` превращает `io::Error` или `serde_json::Error` в `TaskError`.
2. `load` считает отсутствующий файл пустым списком, а всё остальное — ошибкой. `save` записывает красивый JSON.
3. `add` и `complete` — обычные функции над `Vec<Task>` или `[Task]`, поэтому их легко тестировать без файлов (четыре теста в файле).
4. `main.rs` описывает командную строку с помощью `clap`: глобальная опция `--file` и три подкоманды.
5. `fn main() -> Result<()>` использует `anyhow::Result`. Любая ошибка, дошедшая до конца `main`, печатается, и программа завершается с кодом 1. `with_context` добавляет имя файла к ошибкам из `load`.
6. `use taskbox::{...}`: бинарник использует библиотеку как любой другой крейт.

Соберите и попробуйте (сборка лежит в `target/debug/taskbox`; файл данных `tasks.json` появится в текущей папке):

```bash
cargo build
mkdir -p run && cd run
../target/debug/taskbox --help
```

```text
A tiny task list that is stored in a JSON file

Usage: taskbox [OPTIONS] <COMMAND>

Commands:
  add   Add a new task
  list  Show the tasks
  done  Mark a task as done
  help  Print this message or the help of the given subcommand(s)

Options:
  -f, --file <FILE>  The file that stores the tasks [default: tasks.json]
  -h, --help         Print help
  -V, --version      Print version
```

```bash
../target/debug/taskbox add "write lesson"
../target/debug/taskbox add "run the code"
../target/debug/taskbox list
../target/debug/taskbox done 1
../target/debug/taskbox list
../target/debug/taskbox list --all
```

```text
added task 1
added task 2
[ ] 1: write lesson
[ ] 2: run the code
task 1 is done
[ ] 2: run the code
[x] 1: write lesson
[ ] 2: run the code
```

И пути с ошибками:

```bash
../target/debug/taskbox done 99
echo "not json" > bad.json
../target/debug/taskbox -f bad.json list
```

```text
Error: there is no task with id 99
Error: failed to load tasks from bad.json

Caused by:
    0: the task file is not valid JSON
    1: expected ident at line 1 column 2
```

Второе сообщение — полная **цепочка ошибок**: контекст, добавленный в `main` («failed to load…»), затем сообщение `TaskError` от `thiserror` («the task file is not valid JSON»), затем исходное сообщение от `serde_json`. Каждый слой добавил то, что знал. Каждая неудачная команда завершается с кодом возврата 1.

Файл `tasks.json` выглядит так (так работает `serde_json::to_string_pretty`):

```text
[
  {
    "id": 1,
    "title": "write lesson",
    "done": true
  },
  {
    "id": 2,
    "title": "run the code",
    "done": false
  }
]
```

Запустите тесты командой `cargo test`:

```text
test tests::add_gives_increasing_ids ... ok
test tests::complete_marks_done ... ok
test tests::complete_unknown_id_is_an_error ... ok
test tests::json_round_trip ... ok
test result: ok. 4 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

> 🧪 **Тестирование:** держите логику в библиотеке, а тонкий слой (терминал, файлы, `main`) — снаружи. Логику можно проверять обычными модульными тестами. В [Уроке 05.13](13-project-minigrep.md) вы увидите, как тестировать целую программу с файлами.

### Git: сохраните свою работу

Самое время потренировать привычки из [Этапа 01](../01-git-and-version-control/03-your-first-repository-and-commits.md) и [Урока 05.8](08-cargo-dependencies-and-versions.md). `taskbox` лежит внутри вашего учебного репозитория (`~/projects/rust`), поэтому работайте в ветке и коммитьте только эту папку:

```bash
git switch -c project/05-taskbox
printf 'run/\ntasks.json\n' >> .gitignore
git add .
git status --short
git commit -m "feat(p05): add taskbox with serde, clap, thiserror and anyhow"
```

Зачем дополнительные строки в `.gitignore`? `tasks.json` и папка `run` — это *данные*, созданные при пробных запусках программы, а не исходный код. Не кладите их в Git (этот `.gitignore` лежит внутри папки `taskbox` и действует только для неё). `Cargo.lock` остаётся, потому что в этом пакете есть бинарник. `git add .` добавляет в индекс только текущую папку, а не весь репозиторий, а `git status --short` показывает, что попало в индекс, до коммита.

## Частые ошибки

1. **Нет фичи `derive` у `serde` или `clap`** — «cannot find derive macro» (раздел 2). `cargo add serde --features derive`, `cargo add clap --features derive`.
2. **Забыли `#[from]` в перечислении `thiserror`** — `E0271` или `E0277` на `?` (раздел 4).
3. **Возвращаете `anyhow::Error` из библиотеки** — вызывающие не могут разбирать случаи. В библиотеках используйте `thiserror`.
4. **Используете `unwrap()` для пользовательских данных** в настоящем инструменте. Возвращайте ошибки через `?` и показывайте их; `unwrap` годится для тестов и быстрых примеров.
5. **Запускаете `cargo run -- args` без `--`.** Cargo попытался бы прочитать `--shout` как собственную опцию. Ставьте `--` перед аргументами вашей программы.
6. **Коммитите файлы с данными** (`tasks.json`) в Git. Добавьте их в `.gitignore`.
7. **Доверяете разобранным значениям**, раз типы подошли. `u32` принимает `4000000000`. Проверяйте диапазоны.
8. **Используете устаревший крейт** вроде `serde_yaml` в новом коде. Проверьте `cargo info` и страницу крейта на отметку «deprecated».

## Попробуйте сами

1. Создайте `taskbox` в точности как в разделе 6 и выполните команды. Сравните свой вывод с уроком.
2. Выполните `../target/debug/taskbox add` (без заголовка). Прочитайте ошибку. Выполните `../target/debug/taskbox done abc`. Что говорит `clap`?
3. Выполните `../target/debug/taskbox --version`. Откуда берётся версия? (Посмотрите в `Cargo.toml`.) Измените её на `0.2.0`, пересоберите и запустите снова.
4. Откройте `tasks.json`, испортите его, удалив скобку, и выполните `list`. Прочитайте цепочку ошибок.
5. Выполните `cargo tree --depth 1`. У вас 5 прямых зависимостей. Выполните `cargo tree` без параметров, чтобы увидеть, от скольких крейтов вы реально зависите. Это ваша цепочка поставок ([Урок 05.8](08-cargo-dependencies-and-versions.md)).

## Упражнения

### Упражнение 1 — Цикл «туда и обратно» с тестом (легко)

Определите `struct Point { x: i32, y: i32 }` с `Debug`, `PartialEq`, `Serialize` и `Deserialize`. Напишите **тест**, который сериализует `Point { x: 3, y: -4 }`, проверяет, что JSON ровно `{"x":3,"y":-4}`, разбирает его обратно и сравнивает с оригиналом.

<details>
<summary>Подсказка</summary>

`serde_json::to_string(&p).unwrap()` и `serde_json::from_str::<Point>(&json).unwrap()`. Сравнивайте строки и значения через `assert_eq!`.

</details>

<details>
<summary>Ответ</summary>

```rust
use serde::{Deserialize, Serialize};

#[derive(Debug, PartialEq, Serialize, Deserialize)]
pub struct Point {
    pub x: i32,
    pub y: i32,
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn point_round_trip() {
        let p = Point { x: 3, y: -4 };
        let json = serde_json::to_string(&p).unwrap();
        assert_eq!(json, r#"{"x":3,"y":-4}"#);
        let back: Point = serde_json::from_str(&json).unwrap();
        assert_eq!(back, p);
    }
}
```

Почему это работает: derive пишет поля в порядке, в котором они объявлены в структуре. `PartialEq` нужен, чтобы `assert_eq!` мог сравнить две точки. Тест «туда и обратно» — первый тест, который стоит написать для любого сериализуемого типа.

</details>

### Упражнение 2 — Добавьте контекст с anyhow (легко)

Напишите `pub fn read_point(text: &str) -> anyhow::Result<Point>`, которая разбирает JSON в `Point` из Упражнения 1 и добавляет контекст `could not read the point`. Напишите **тест**, который проверяет: для текста `{` сообщение ошибки равно `could not read the point`, а форма `{:#}` начинается с `could not read the point: `.

<details>
<summary>Подсказка</summary>

`serde_json::from_str(text).context("could not read the point")?` — `Context` нужно импортировать из `anyhow`.

</details>

<details>
<summary>Ответ</summary>

```rust
use anyhow::{Context, Result};

pub fn read_point(text: &str) -> Result<Point> {
    let point: Point = serde_json::from_str(text).context("could not read the point")?;
    Ok(point)
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn read_point_adds_context() {
        let err = read_point("{").unwrap_err();
        assert_eq!(err.to_string(), "could not read the point");
        assert!(format!("{err:#}").starts_with("could not read the point: "));
        assert_eq!(read_point(r#"{"x":1,"y":2}"#).unwrap(), Point { x: 1, y: 2 });
    }
}
```

Почему это работает: `{err}` показывает только самое новое (внешнее) сообщение. `{err:#}` добавляет причины после `: `. `?` преобразует ошибку с контекстом в `anyhow::Error`.

</details>

### Упражнение 3 — Тест разборщика clap (средне)

Напишите `#[derive(Parser)] pub struct Args { word: String, times: usize }`, где `times` — опция `--times`/`-t` со значением по умолчанию `2`. Напишите `fn repeat(args: &Args) -> String`, которая склеивает `word` `times` раз через пробелы. Напишите **тест** с `try_parse_from`: `["repeat", "hi"]` даёт `"hi hi"`, а `["repeat", "hi", "--times", "3"]` даёт `"hi hi hi"`.

<details>
<summary>Подсказка</summary>

`#[arg(short, long, default_value_t = 2)]` у `times`. Текст соберите так: `vec![args.word.as_str(); args.times].join(" ")`.

</details>

<details>
<summary>Ответ</summary>

```rust
use clap::Parser;

#[derive(Parser, Debug)]
pub struct Args {
    /// Word to repeat
    pub word: String,
    /// How many times
    #[arg(short, long, default_value_t = 2)]
    pub times: usize,
}

pub fn repeat(args: &Args) -> String {
    vec![args.word.as_str(); args.times].join(" ")
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn repeat_uses_default_and_option() {
        let a = Args::try_parse_from(["repeat", "hi"]).unwrap();
        assert_eq!(repeat(&a), "hi hi");
        let b = Args::try_parse_from(["repeat", "hi", "--times", "3"]).unwrap();
        assert_eq!(repeat(&b), "hi hi hi");
    }
}
```

Почему это работает: `try_parse_from` запускает тот же разборщик, что и настоящая командная строка, поэтому тест проверяет значения по умолчанию и опции без запуска процесса. `vec![x; n]` создаёт вектор из `n` копий, а `join(" ")` их склеивает.

</details>

### Упражнение 4 — Перечисление ошибок с thiserror (средне)

Напишите `pub fn parse_age(text: &str) -> Result<u8, AgeError>`. У `AgeError` два варианта: `NotANumber` (сообщение `age must be a number`) и `OutOfRange(u32)` (сообщение `age {0} is not realistic`, для чисел больше 150). Пробелы вокруг текста допустимы. Напишите **тест** для `" 42 "`, `"abc"` и `"200"`, включая сообщение последней ошибки.

<details>
<summary>Подсказка</summary>

Добавьте к перечислению `Debug`, `Error` и `PartialEq`. Используйте `text.trim().parse::<u32>()` с `.map_err(|_| AgeError::NotANumber)?`.

</details>

<details>
<summary>Ответ</summary>

```rust
use thiserror::Error;

#[derive(Debug, Error, PartialEq)]
pub enum AgeError {
    #[error("age must be a number")]
    NotANumber,
    #[error("age {0} is not realistic")]
    OutOfRange(u32),
}

pub fn parse_age(text: &str) -> Result<u8, AgeError> {
    let n: u32 = text.trim().parse().map_err(|_| AgeError::NotANumber)?;
    if n > 150 {
        return Err(AgeError::OutOfRange(n));
    }
    Ok(n as u8)
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn age_errors() {
        assert_eq!(parse_age(" 42 "), Ok(42));
        assert_eq!(parse_age("abc"), Err(AgeError::NotANumber));
        assert_eq!(parse_age("200"), Err(AgeError::OutOfRange(200)));
        assert_eq!(parse_age("200").unwrap_err().to_string(), "age 200 is not realistic");
    }
}
```

Почему это работает: разбор сначала в `u32` позволяет дать *полезную* ошибку `OutOfRange` для `200` (и для `300`, которое даже не поместилось бы в `u8`) вместо расплывчатого «не число». `n as u8` безопасно, потому что в этой точке `n <= 150`. Перечисление выводит `PartialEq` только для того, чтобы можно было применять `assert_eq!` к `Result`.

</details>

### Упражнение 5 — Новая подкоманда для taskbox (сложно)

Добавьте в `taskbox` команду `remove <id>`. Напишите в `lib.rs` функцию `pub fn remove(tasks: &mut Vec<Task>, id: u32) -> Result<Task, TaskError>` (она возвращает удалённую задачу или `TaskError::NotFound`). Добавьте подкоманду в `main.rs`; она печатает `removed task <id>: <title>`. Напишите **тест** для `remove`. Закоммитьте изменение в Git с хорошим сообщением (в стиле этого этапа: `feat(p05): …`).

<details>
<summary>Подсказка</summary>

Найдите позицию через `tasks.iter().position(|t| t.id == id)`, превратите `None` в ошибку через `.ok_or(TaskError::NotFound(id))?`, затем `tasks.remove(position)`. В `main.rs` добавьте вариант `Remove { id: u32 }` и ветку `match`, которая загружает, удаляет, сохраняет и печатает.

</details>

<details>
<summary>Ответ</summary>

В `src/lib.rs`:

```rust
/// Removes a task and returns it.
pub fn remove(tasks: &mut Vec<Task>, id: u32) -> Result<Task, TaskError> {
    let position = tasks
        .iter()
        .position(|t| t.id == id)
        .ok_or(TaskError::NotFound(id))?;
    Ok(tasks.remove(position))
}
```

В модуле `tests`:

```rust
    #[test]
    fn remove_takes_the_task_out() {
        let mut tasks = Vec::new();
        let first = add(&mut tasks, "a");
        add(&mut tasks, "b");
        let removed = remove(&mut tasks, first).unwrap();
        assert_eq!(removed.title, "a");
        assert_eq!(tasks.len(), 1);
        assert!(matches!(remove(&mut tasks, 99), Err(TaskError::NotFound(99))));
    }
```

В `src/main.rs`: измените строку `use` на `use taskbox::{add, complete, load, remove, save};`, добавьте вариант в `enum Command`:

```rust
    /// Remove a task
    Remove {
        /// The id of the task
        id: u32,
    },
```

и ветку в `match`:

```rust
        Command::Remove { id } => {
            let task = remove(&mut tasks, id)?;
            save(&cli.file, &tasks)?;
            println!("removed task {id}: {}", task.title);
        }
```

Команды `taskbox add x`, `taskbox add y`, `taskbox remove 1`, `taskbox list` печатают `added task 1`, `added task 2`, `removed task 1: x`, `[ ] 2: y`. Команда `taskbox remove 5` печатает `Error: there is no task with id 5` и завершается с кодом 1. `cargo test` сообщает 5 passed.

Git: `git add .`, затем `git commit -m "feat(p05): add the remove command to taskbox"`.

Почему это работает: `position` возвращает `Option<usize>`, `ok_or` заменяет `None` нужной ошибкой, а `?` возвращает её. Тип ошибки библиотеки проходит через `anyhow` в `main` без дополнительного кода. `matches!` проверяет значение на соответствие образцу в тесте и не требует `PartialEq` у ошибки.

</details>

## Вопросы для самопроверки

1. Чем отличаются `serde` и `serde_json`?

<details>
<summary>Ответ</summary>

`serde` — фреймворк: трейты `Serialize` и `Deserialize` и derive-макросы. `serde_json` — один из форматов данных, который их использует (JSON). У других форматов (TOML и так далее) свои крейты.

</details>

2. Что делает `#[derive(Serialize, Deserialize)]`?

<details>
<summary>Ответ</summary>

При компиляции он генерирует код, который преобразует структуру или перечисление в формат данных и обратно. Без него вам пришлось бы писать эти реализации вручную.

</details>

3. Как `clap` узнаёт, какие поля обязательные, какие — опции, а какие — флаги?

<details>
<summary>Ответ</summary>

По их виду: обычное поле — обязательный позиционный аргумент; `Option<T>` или поле со значением по умолчанию — необязательный; `#[arg(short, long)]` делает его именованной опцией; `bool` с `#[arg(...)]` — флаг. Doc-комментарии становятся текстом справки.

</details>

4. Когда использовать `thiserror`, а когда `anyhow`?

<details>
<summary>Ответ</summary>

`thiserror` — в библиотеках, чтобы описать точные типы ошибок, которые вызывающий код может разбирать. `anyhow` — в приложениях (особенно в `main`), чтобы собрать любую ошибку, добавить контекст и напечатать её.

</details>

5. Что делает `#[from]` в перечислении `thiserror`?

<details>
<summary>Ответ</summary>

Он реализует `From<SourceError>` для перечисления, поэтому `?` автоматически преобразует исходную ошибку, и делает исходную ошибку доступной через `Error::source`.

</details>

6. Чем отличаются `{e}`, `{e:#}` и `{e:?}` для `anyhow::Error`?

<details>
<summary>Ответ</summary>

`{e}` — только самое внешнее сообщение. `{e:#}` — вся цепочка в одной строке через `: `. `{e:?}` — сообщение плюс список «Caused by:» (форма, которую печатает `main`, когда возвращает `Err`).

</details>

7. Как протестировать разборщик `clap` без запуска программы?

<details>
<summary>Ответ</summary>

Через `Cli::try_parse_from([...])`, передав имя программы и аргументы списком. Он возвращает `Result`, а не завершает процесс.

</details>

8. Почему `taskbox` держит логику в `lib.rs`, а не в `main.rs`?

<details>
<summary>Ответ</summary>

В библиотеке нет кода для терминала, поэтому её функции можно тестировать обычными модульными тестами и переиспользовать. `main.rs` тонкий: он читает командную строку, вызывает библиотеку и печатает.

</details>

## Итоги

- `serde` преобразует данные в текстовые форматы и обратно; `#[derive(Serialize, Deserialize)]` вместе с `serde_json` или `toml` закрывает большинство потребностей. Атрибуты `rename`, `default` и `skip_serializing_if` настраивают формат.
- Десериализация в типизированные структуры проверяет форму внешних данных; смысл вы по-прежнему проверяете сами.
- `clap` строит интерфейс командной строки по структуре: аргументы, опции, флаги, подкоманды, `--help` и ошибки бесплатно. Тестируйте его через `try_parse_from`.
- `thiserror` убирает шаблонный код перечислений ошибок: `#[error("...")]` для сообщений и `#[from]` для преобразований.
- `anyhow` — для приложений: `anyhow::Result`, `.context(...)`, `bail!`, `ensure!` и печать цепочки ошибок.
- Хорошая структура: логика в `lib.rs` (тестируемая, с точными ошибками), тонкий `main.rs` (clap + anyhow).
- Держите файлы данных вне Git; для бинарников держите `Cargo.lock` в репозитории.

## Источники и что почитать

- [Serde documentation](https://serde.rs/) — руководство: derive, атрибуты, форматы данных, собственные реализации.
- [serde_json on docs.rs](https://docs.rs/serde_json/) — `from_str`, `to_string`, `Value`, `json!`.
- [clap on docs.rs](https://docs.rs/clap/) — руководство по derive и справочник.
- [thiserror on docs.rs](https://docs.rs/thiserror/) — все формы `#[error]`, `#[from]` и `#[source]`.
- [anyhow on docs.rs](https://docs.rs/anyhow/) — `Context`, `bail!`, `ensure!`, downcasting.
- [The Rust Programming Language — Chapter 9: Error Handling](https://doc.rust-lang.org/book/ch09-00-error-handling.html) — основа, на которой стоят `thiserror` и `anyhow`.
- [Command Line Applications in Rust](https://rust-cli.github.io/book/) — книга официальной рабочей группы по CLI: аргументы, вывод, коды возврата, тестирование.

---

[🏠 Начало](../START-HERE.md) · [← 05.10 Потоки, каналы и общее состояние](10-threads-channels-and-shared-state.md) · [📋 Этап 05](README.md) · [05.12 Основы бенчмарков →](12-benchmarking-basics.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
