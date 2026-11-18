<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 08.10 Антипаттерны и «запахи» кода](10-anti-patterns-and-code-smells.md) · [📋 Этап 08](README.md) · [08.12 Структура проекта и рабочие пространства →](12-project-layout-and-workspaces.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/08-software-design/11-conventions-naming-clippy-and-api-guidelines.md) <!-- nav -->

# Урок 08.11 — Соглашения: имена, Clippy и Rust API Guidelines

> **Этап 08 — Проектирование ПО: принципы, паттерны и соглашения** · Урок 11 из 17 · ⏱ около 135 минут

## Что вы узнаете

- Соглашения Rust об именах: регистр и префиксы `as_`, `to_`, `into_`, `is_`, `iter`.
- Как `rustfmt` задаёт единое форматирование для всей команды и как его настроить.
- Как пользоваться группами lint-ов Clippy, уровнями lint-ов (`allow`, `warn`, `deny`, `expect`) и таблицей `[lints]` в `Cargo.toml`.
- Чего Rust API Guidelines требуют от публичного API (Application Programming Interface, программный интерфейс: элементы, которые ваш крейт предлагает другому коду) — на примере одной небольшой библиотеки.

## Перед началом

- Вы прошли [Урок 08.10 — Антипаттерны и «запахи» кода](10-anti-patterns-and-code-smells.md).
- Вы знаете контроль качества из [Урока 05.6](../05-rust-intermediate/06-testing-in-depth.md) и doc-комментарии из [Урока 05.7](../05-rust-intermediate/07-documentation.md).
- Создайте два учебных пакета: `cargo new code/p08-conventions --lib` (для раздела 5) и `cargo new code/p08-gate` (для раздела «Попробуйте сами»). Остальные примеры можно запускать в любом черновом пакете.

## Новые слова

| Слово | Значение |
|---|---|
| **соглашение** (convention) | Принятый способ делать что-то, который язык не навязывает, но которому все следуют, чтобы код выглядел привычно. |
| **`rustfmt`** | Официальный инструмент, который переписывает код Rust в стандартный вид. Запускается командой `cargo fmt`. |
| **уровень lint-а** (lint level) | Что происходит, когда lint срабатывает: `allow` (ничего), `warn` (предупреждение), `deny` (ошибка), `forbid` (ошибка, которую нельзя отключить), `expect` (как `allow`, но предупреждение, если lint *не* сработал). |
| **группа lint-ов** (lint group) | Именованный набор lint-ов, которые включают или выключают вместе, например `clippy::pedantic`. |
| **TOML** (Tom's Obvious Minimal Language, «минималистичный язык Тома») | Текстовый формат файлов `Cargo.toml`, `rustfmt.toml` и `clippy.toml`. |
| **RFC** (Request for Comments, запрос комментариев) | Письменное предложение об изменении Rust. Правила именования Rust взяты из RFC с номером 430. |
| **Rust API Guidelines** | Чек-лист, написанный командой библиотек Rust, для проектирования публичных API (API — Application Programming Interface, программный интерфейс: функции и типы, которые крейт предлагает другому коду), которые «ощущаются по-rust-овски». |
| **`#[non_exhaustive]`** | Атрибут, который сообщает другим крейтам: «позже я могу добавить варианты или поля», поэтому им нужно обрабатывать неизвестное. |
| **`#[must_use]`** | Атрибут, из-за которого компилятор предупреждает, если вызывающий код игнорирует результат. |
| **CI** (Continuous Integration, непрерывная интеграция) | Сервер, который автоматически запускает ваши проверки (формат, lint-ы, тесты) при каждом изменении. Этап 09 строит конвейеры CI. |

## 1. Зачем нужны соглашения

Соглашение ничего не стоит и окупается каждый день. Если все проекты на Rust форматируют код одинаково,
называют вещи одинаково и используют одинаковые настройки lint-ов, вы читаете чужой код так же быстро, как
свой, инструменты работают без настройки, а код-ревью (Урок 08.13) обсуждает проектирование, а не пробелы и
имена. Большинство соглашений этого урока проверяет инструмент:

| Соглашение | Кто проверяет | Команда |
|---|---|---|
| Имена | компилятор (регистр) и Clippy | `cargo build`, `cargo clippy` |
| Оформление кода | `rustfmt` | `cargo fmt --check` |
| Идиомы, типичные ошибки | Clippy | `cargo clippy -- -D warnings` |
| Документация | lint `missing_docs`, документационные тесты | `cargo build`, `cargo test --doc` |
| Проектирование публичного API | люди, по чек-листу API Guidelines | код-ревью |

## 2. Имена

### 2.1 Регистр

Компилятор предупреждает, когда у имени неправильный стиль. В черновом пакете, `src/main.rs`:

```rust
struct shopping_cart {
    ItemCount: u32,
}

const maxItems: u32 = 10;

fn AddItem(cart: &mut shopping_cart) {
    cart.ItemCount += 1;
}

fn main() {
    let mut cart = shopping_cart { ItemCount: 0 };
    AddItem(&mut cart);
    println!("{} of {}", cart.ItemCount, maxItems);
}
```

`cargo build` компилирует код и печатает четыре предупреждения (показаны первые два):

```text
warning: type `shopping_cart` should have an upper camel case name
 --> src/main.rs:1:8
  |
1 | struct shopping_cart {
  |        ^^^^^^^^^^^^^ help: convert the identifier to upper camel case: `ShoppingCart`

warning: structure field `ItemCount` should have a snake case name
 --> src/main.rs:2:5
  |
2 |     ItemCount: u32,
  |     ^^^^^^^^^ help: convert the identifier to snake case: `item_count`
```

Правила (из API Guidelines, которые следуют RFC 430):

| Элемент | Стиль | Пример |
|---|---|---|
| Типы: `struct`, `enum`, `trait`, псевдонимы типов; варианты `enum` | `UpperCamelCase` | `ShoppingCart`, `Zone::Foreign` |
| Функции, методы, модули, переменные, поля | `snake_case` | `add_item`, `item_count` |
| Константы и статические переменные | `SCREAMING_SNAKE_CASE` | `MAX_ITEMS` |
| Параметры обобщённых типов | одна заглавная буква или короткое `UpperCamelCase` | `T`, `Item` |
| Времена жизни | короткое, строчными | `'a` |

В `UpperCamelCase` аббревиатура считается одним словом: `Uuid`, `HttpServer`, а не `UUID` или `HTTPServer`. Имя
пакета обычно с дефисами (`p08-conventions`); имя крейта в коде — с подчёркиваниями (`p08_conventions`).

### 2.2 Слова со смыслом

Некоторые префиксы означают одно и то же во всей стандартной библиотеке, поэтому по имени видно, что делает метод и сколько он стоит:

| Имя | Значит | Стоимость | Пример из стандартной библиотеки |
|---|---|---|---|
| `as_x` | заимствованный вид данных как другой заимствованный тип | бесплатно | `String::as_str`, `Vec::as_slice` |
| `to_x` | новое значение, созданное из заимствованного (по значению для `Copy`-типов) | может быть дорого | `str::to_uppercase` |
| `into_x` | потребляет `self` и возвращает данные в другой форме | обычно дёшево | `String::into_bytes` |
| `x()` | геттер: имя поля без `get_` | бесплатно | `Vec::len` |
| `x_mut()` | то же, но изменяемое заимствование | бесплатно | `slice::first_mut` |
| `is_x`, `has_x` | вопрос с ответом «да/нет» | дёшево | `Vec::is_empty` |
| `iter`, `iter_mut`, `into_iter` | заимствовать элементы, заимствовать изменяемо, потребить | бесплатно | `Vec::iter` |
| `new`, `with_x`, `from_x` | конструкторы | | `Vec::with_capacity` |

```rust
fn main() {
    let text = String::from("Hello");

    let view: &str = text.as_str(); // as_: a free, borrowed view
    let upper: String = view.to_uppercase(); // to_: new value, may cost work
    println!("{view} {upper}");

    let bytes: Vec<u8> = text.into_bytes(); // into_: consumes `text`
    println!("{bytes:?}");

    let mut numbers = vec![1, 2, 3];
    for n in numbers.iter_mut() {
        *n *= 10; // iter_mut: borrows items mutably
    }
    for n in numbers {
        print!("{n} "); // into_iter: consumes the vector
    }
    println!();
}
```

**Что происходит:**

1. `as_str` только смотрит на `text`; `to_uppercase` выделяет новую `String`. Префикс говорит, какой из них дешёвый.
2. `into_bytes` принимает `self` по значению. После него `text` перемещён; повторное использование было бы ошибкой компиляции.
3. Последний `for` принимает `numbers` по значению, то есть вызывает `into_iter`.

```text
Hello HELLO
[72, 101, 108, 108, 111]
10 20 30 
```

> 📏 **Соглашение:** не называйте геттер `get_name`. Называйте его `name`. Слово `get` оставляют для «получить по
> ключу», как `HashMap::get`. В Rust пишут `user.name()`, а не `user.get_name()`.

## 3. Форматирование с `rustfmt`

Споры о форматировании тратят время, а большой diff, полный пробелов, скрывает настоящее изменение. Rust
заканчивает обсуждение одним инструментом — `rustfmt`, который поставляется с тулчейном и запускается через
Cargo. Возьмите этот неопрятный, но корректный код:

```rust
fn   add(a:i32,b:i32)->i32{
a+b
}

fn main(){
    let total=add( 2,3 );
    println!("{total}");
        if total>3 {println!("big");}
}
```

`cargo fmt --check` только *проверяет*: печатает, что бы он изменил, и завершается с кодом 1 — именно это использует задание CI.

```text
Diff in .../src/main.rs:1:
-fn   add(a:i32,b:i32)->i32{
-a+b
+fn add(a: i32, b: i32) -> i32 {
+    a + b
 }
 
-fn main(){
-    let total=add( 2,3 );
+fn main() {
+    let total = add(2, 3);
     println!("{total}");
-        if total>3 {println!("big");}
+    if total > 3 {
+        println!("big");
+    }
 }
```

`cargo fmt` (без `--check`) переписывает файлы. RustRover умеет то же самое через *Reformat Code*, и можно
включить запуск `rustfmt` при сохранении.

**Настройка.** Почти никто не меняет значения по умолчанию, и в этом смысл: стиль по умолчанию — тот, что
используют другие проекты. Если нужно, создайте `rustfmt.toml` в корне проекта. Например, `max_width = 60` (по
умолчанию 100) разбивает более длинные строки:

```rust
fn main() {
    let message = format!(
        "{} items cost {} in total, including {} tax",
        3, 100, 20
    );
    println!("{message}");
}
```

Многие параметры существуют только в nightly-компиляторе. На stable `rustfmt` их игнорирует и сообщает об этом:
``Warning: can't set `imports_granularity = Crate`, unstable features are only available in nightly channel.``
Когда ручное оформление несёт смысл, например таблица чисел, поставьте `#[rustfmt::skip]` перед элементом, и `rustfmt` его не тронет.

> 📏 **Соглашение:** запускайте `cargo fmt` перед каждым коммитом и `cargo fmt --check` в CI. Никогда не смешивайте
> в одном коммите изменение только форматирования и настоящее изменение: ревью становится невозможным.

## 4. Clippy подробно

В Уроке 08.10 вы запускали `cargo clippy`. Теперь научимся им управлять.

### 4.1 Группы lint-ов

В Clippy сотни lint-ов, разложенных по группам:

| Группа | Уровень по умолчанию | Что входит |
|---|---|---|
| `clippy::correctness` | deny (ошибка) | Код, который почти наверняка неверен. |
| `clippy::suspicious` | warn | Код, который, вероятно, неверен. |
| `clippy::style` | warn | Код, который работает, но неидиоматичен. |
| `clippy::complexity` | warn | Код, который делает простое сложным способом. |
| `clippy::perf` | warn | Код, который легко ускорить. |
| `clippy::pedantic` | allow (выключено) | Строгие, субъективные lint-ы. Некоторые дают ложные срабатывания. |
| `clippy::nursery` | allow (выключено) | Новые lint-ы, которые ещё проверяются. |
| `clippy::restriction` | allow (выключено) | Lint-ы, запрещающие возможность (`unwrap_used`, `indexing_slicing`). Не включайте группу целиком; выбирайте отдельные lint-ы. |
| `clippy::cargo` | allow (выключено) | Проверки `Cargo.toml` (нет описания, дубликаты версий). |

Включайте группу из командной строки через `-W` (warn) или `-D` (deny). Этот код корректен, и Clippy по умолчанию молчит:

```rust
fn average(values: &[f64]) -> f64 {
    values.iter().sum::<f64>() / values.len() as f64
}

fn main() {
    let numbers = vec![1.0, 2.0, 4.5];
    println!("{}", average(&numbers));
}
```

С `cargo clippy -- -W clippy::pedantic`:

```text
warning: casting `usize` to `f64` may cause a loss of precision (`usize` can be up to 64 bits wide depending on the target architecture, but `f64`'s mantissa is only 52 bits wide)
 --> src/main.rs:2:34
  |
2 |     values.iter().sum::<f64>() / values.len() as f64
  |                                  ^^^^^^^^^^^^^^^^^^^
  = note: `-W clippy::cast-precision-loss` implied by `-W clippy::pedantic`
  = help: to override `-W clippy::pedantic` add `#[allow(clippy::cast_precision_loss)]`
```

Для `pedantic` это типично: lint *прав* в теории (на практике в срезе не бывает 2^52 элементов), а вы решаете, что здесь это не важно.

### 4.2 Уровни lint-ов и исключения

Уровень выбирают в трёх местах: в `Cargo.toml` для всего пакета (раздел 4.3), в исходном коде для крейта или
модуля (`#![warn(missing_docs)]`) и в исходном коде для одного элемента (`#[allow(clippy::cast_precision_loss)]`
перед функцией).

Для одного исключения есть два атрибута. `#[allow(...)]` молчит навсегда. `#[expect(...)]` (стабилен с Rust 1.81)
делает то же самое, **но предупреждает, когда lint перестаёт срабатывать**, поэтому устаревшие исключения
исчезают. Всегда указывайте причину:

```rust
#[expect(
    clippy::cast_precision_loss,
    reason = "slice lengths are far below 2^52, so the conversion is exact"
)]
fn average(values: &[f64]) -> f64 {
    values.iter().sum::<f64>() / values.len() as f64
}

fn main() {
    let numbers = vec![1.0, 2.0, 4.5];
    println!("{}", average(&numbers));
}
```

При включённом `pedantic` `cargo clippy` молчит, а `cargo run` печатает `2.5`. Если позже вы измените `average`
так, что приведения типов больше нет (скажем, она делит на `3.0`), исключение становится ненужным, и Clippy
сообщает об этом:

```text
warning: this lint expectation is unfulfilled
 --> src/main.rs:2:5
  |
2 |     clippy::cast_precision_loss,
  |     ^^^^^^^^^^^^^^^^^^^^^^^^^^^
  |
  = note: slice lengths are far below 2^52, so the conversion is exact
  = note: `#[warn(unfulfilled_lint_expectations)]` on by default
```

> ⚠️ **Частая ошибка:** `#![allow(clippy::all)]` или `#![allow(warnings)]` на весь крейт, чтобы «убрать шум». Это
> выключает инструмент. Разрешайте один lint, на одном элементе, с причиной.

### 4.3 Таблица `[lints]`, `clippy.toml` и `--fix`

Начиная с Cargo 1.74, уровни lint-ов можно хранить в `Cargo.toml`, чтобы все команды (и RustRover) видели одну политику:

```toml
[lints.rust]
missing_docs = "warn"

[lints.clippy]
pedantic = { level = "warn", priority = -1 }
unwrap_used = "deny"
```

**Что происходит:**

1. `[lints.rust]` задаёт уровни для lint-ов компилятора; `[lints.clippy]` — для lint-ов Clippy.
2. `pedantic` — группа. `priority = -1` означает «применить первой», чтобы отдельный lint, например `unwrap_used`,
   мог переопределить группу. (Без приоритета lint Clippy `lint_groups_priority` сообщает о конфликте группы и отдельного lint-а.)
3. `unwrap_used` — из группы `restriction`: мы выбираем именно этот lint намеренно. С ним любой `.unwrap()` у
   `Result` — это ``error: used `unwrap()` on a `Result` value``, и проверка не проходит; публичная функция без
   doc-комментария даёт `warning: missing documentation for a function`.

У некоторых lint-ов есть *настройки*. Они лежат в `clippy.toml` в корне проекта. Например, с
`too-many-arguments-threshold = 3` Clippy сообщает `this function has too many arguments (4/3)` для функции с
четырьмя параметрами.

`cargo clippy --fix` применяет безопасные подсказки. Он отказывается работать, если есть незакоммиченные
изменения (сначала закоммитьте или передайте `--allow-dirty`). Успешный запуск печатает `Fixed src/main.rs (1
fix)`. Потом прочитайте `git diff`: инструмент хорош, но за коммит отвечаете вы.

> 🧪 **Тестирование:** контроль качества проекта — три команды: `cargo fmt --check`, `cargo clippy --all-targets -- -D
> warnings` и `cargo test`. `--all-targets` заставляет Clippy проверять и тесты с примерами, а не только основной код.

## 5. Rust API Guidelines

**Rust API Guidelines** — это чек-лист (около 50 пунктов, у каждого свой код вроде `C-GETTER`), который команда
библиотек Rust написала по долгому опыту работы со стандартной библиотекой. Он отвечает на вопрос: «Как должна
выглядеть публичная сторона моего крейта?»

| Группа | Чего требует (избранные пункты) |
|---|---|
| Имена | Регистр; `as_`/`to_`/`into_`; геттеры без `get_`; `iter`/`iter_mut`/`into_iter`. |
| Совместимость | Реализуйте общие трейты: `Debug` всегда, а `Clone`, `PartialEq`, `Default`, `Display` — когда есть смысл; `From` и `TryFrom` для преобразований; типы ошибок реализуют `std::error::Error`. |
| Документация | У каждого публичного элемента есть doc-комментарий; запускаемый пример; разделы `# Errors`, `# Panics` и `# Safety`, когда они применимы. |
| Предсказуемость | Конструкторы — ассоциированные функции (`new`); никаких выходных параметров. |
| Гибкость | Принимайте самый общий заимствованный тип (`&str`, `&[T]`). |
| Безопасность типов | Newtype для разных смыслов; `enum` вместо аргумента `bool`; строители для множества параметров (Урок 08.6). |
| Надёжность | Проверяйте аргументы в конструкторах. |
| Защита от будущих изменений | Держите поля структур приватными; помечайте `#[non_exhaustive]` публичные `enum`, которые могут вырасти. |

Запоминать это не нужно; вы открываете чек-лист, когда публичный API почти готов, и проходите по нему. Вот
небольшая библиотека, которая применяет несколько пунктов. В `code/p08-conventions/src/lib.rs`:

```rust
//! Temperatures that cannot be physically impossible.

#![warn(missing_docs)]

use std::fmt;

const ABSOLUTE_ZERO: f64 = -273.15;

/// A temperature in degrees Celsius, never below absolute zero.
///
/// # Examples
///
/// ```
/// use p08_conventions::Celsius;
///
/// let warm = Celsius::new(21.5).unwrap();
/// assert_eq!(warm.degrees(), 21.5);
/// assert!(!warm.is_freezing());
/// assert!(Celsius::new(-300.0).is_err());
/// ```
#[derive(Debug, Clone, Copy, PartialEq, PartialOrd, Default)]
pub struct Celsius(f64);

/// The ways in which creating a [`Celsius`] can fail.
#[derive(Debug, Clone, PartialEq)]
#[non_exhaustive]
pub enum TemperatureError {
    /// The value is below absolute zero (-273.15).
    BelowAbsoluteZero,
    /// The value is not a number (`NaN`).
    NotANumber,
}

impl Celsius {
    /// Creates a temperature.
    ///
    /// # Errors
    ///
    /// Returns [`TemperatureError::BelowAbsoluteZero`] for values under
    /// absolute zero, and [`TemperatureError::NotANumber`] for `NaN`.
    pub fn new(degrees: f64) -> Result<Celsius, TemperatureError> {
        if degrees.is_nan() {
            Err(TemperatureError::NotANumber)
        } else if degrees < ABSOLUTE_ZERO {
            Err(TemperatureError::BelowAbsoluteZero)
        } else {
            Ok(Celsius(degrees))
        }
    }

    /// The value in degrees Celsius.
    #[must_use]
    pub fn degrees(self) -> f64 {
        self.0
    }

    /// Converts to degrees Fahrenheit.
    #[must_use]
    pub fn to_fahrenheit(self) -> f64 {
        self.0 * 9.0 / 5.0 + 32.0
    }

    /// Is the temperature below the freezing point of water?
    #[must_use]
    pub fn is_freezing(self) -> bool {
        self.0 < 0.0
    }
}

impl fmt::Display for TemperatureError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            TemperatureError::BelowAbsoluteZero => write!(f, "below absolute zero"),
            TemperatureError::NotANumber => write!(f, "not a number"),
        }
    }
}

impl std::error::Error for TemperatureError {}
```

**Что происходит:** каждая особенность — это один пункт руководства.

1. **Безопасность типов, надёжность:** `Celsius` — newtype с *приватным* полем. Создать значение можно только
   через `Celsius::new`, которая проверяет значение; `Celsius` ниже абсолютного нуля существовать не может
   (Урок 08.5).
2. **Имена:** `degrees()` — геттер без `get_`; `to_fahrenheit` принимает `Copy`-значение по значению; `is_freezing` — вопрос.
3. **Совместимость:** `Debug`, `Clone`, `Copy`, `PartialEq`, `PartialOrd`, `Default` выведены автоматически. Тип ошибки
   реализует `Display` и `std::error::Error`, поэтому работает с `?` и с `Box<dyn Error>`.
4. **Защита от будущих изменений:** `#[non_exhaustive]` позволяет позже добавить вариант, не ломая код в других
   крейтах, который сопоставляет с образцом этот `enum` (им придётся написать ветку `_`).
5. **Документация:** `#![warn(missing_docs)]` заставляет компилятор жаловаться на каждый публичный элемент без
   doc-комментария; `# Errors` говорит, когда `new` завершается ошибкой; пример — документационный тест (Урок 05.7).
6. `#[must_use]` заставляет компилятор предупреждать, если кто-то пишет `c.to_fahrenheit();` и выбрасывает результат.

Запустите `cargo test`, `cargo clippy -- -W clippy::pedantic` и `cargo fmt --check`. Clippy и `fmt` ничего не
печатают, кроме `Checking`/`Finished`, а тест печатает:

```text
   Doc-tests p08_conventions
test src/lib.rs - Celsius (line 13) ... ok
test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

Теперь удалите строку `/// Converts to degrees Fahrenheit.` и соберите снова. Компилятор жалуется, называя конкретный метод:

```text
warning: missing documentation for a method
  --> src/lib.rs:58:5
   |
58 |     pub fn to_fahrenheit(self) -> f64 {
   |     ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
```

Верните строку на место. (Свои функции `#[test]` вы добавите в Упражнении 3.)

> 💡 **Совет:** руководства написаны для *публичных* API, которыми пользуются другие люди. Для приватных
> вспомогательных функций внутри вашей программы идеи тоже полезны, но не тратьте час на `#[non_exhaustive]` для
> `enum`, который виден только вашему коду.

## Частые ошибки

1. **Спорить о стиле вместо запуска `cargo fmt`.** Решает инструмент; обсуждение закончено.
2. **Префикс `get_` у геттеров** (`get_name`). Используйте `name`. Clippy этого не ловит; ловит код-ревью.
3. **`allow` без причины.** Предпочитайте `#[expect(..., reason = "...")]`.
4. **Включать группу `restriction` целиком.** Её lint-ы противоречат друг другу. Выбирайте отдельные lint-ы.
5. **Делать поле `pub` «пока что».** Публичное поле остаётся частью API навсегда. Начинайте с приватного и добавляйте методы.

## Попробуйте сами

Запустите контроль качества на неряшливой программе. В `code/p08-gate/src/main.rs`:

```rust
struct shopping_cart { items: Vec<u32> }

impl shopping_cart {
    fn get_total(&self)->u32 {
        let mut total=0;
        for i in 0..self.items.len() { total+=self.items[i]; }
        return total;
    }
}

fn main(){
    let cart=shopping_cart{items:vec![1,2,3]};
    println!("{}", cart.get_total());
}
```

1. Запустите `cargo fmt --check`. Он печатает diff (первая часть — `-struct shopping_cart { items: Vec<u32> }`).
2. Запустите `cargo clippy`. Вы увидите три предупреждения: имя типа, `unneeded return statement` и цикл по индексу.
3. Исправьте всё вручную: переименуйте тип в `ShoppingCart`, переименуйте `get_total` в `total`, замените цикл и
   `return` на `self.items.iter().sum()` и запустите `cargo fmt`.
4. Запустите `cargo fmt --check && cargo clippy -- -D warnings && cargo run`. Контроль качества молчит (кроме `Checking`/`Finished`), а программа печатает `6`.

## Упражнения

### Упражнение 1 — Выберите правильные имена (легко)

Переименуйте эти методы типа `Name` по соглашениям: `GetValue(&self) -> &str`, `fn to_str_ref(&self) -> &str` (бесплатный
вид внутреннего текста), `fn isEmpty(&self) -> bool`, `fn into_string_owned(self) -> String` и `fn get_len(&self) -> usize`.

<details>
<summary>Подсказка</summary>

Методы пишутся в `snake_case`. Бесплатный заимствованный вид — `as_...`. У геттера нет `get_`. Метод, потребляющий `self`, — `into_...`.

</details>

<details>
<summary>Ответ</summary>

`value`, `as_str`, `is_empty`, `into_string`, `len`.

Почему это работает: у `GetValue` неверный регистр и привкус `get_`; бесплатный вид — `as_str` (как `String::as_str`); вопрос — `is_empty`;
потребляющее преобразование — `into_string`; длина — `len`.

</details>

### Упражнение 2 — Задайте политику lint-ов (средне)

В библиотечном пакете добавьте в `Cargo.toml` таблицу `[lints]`, которая заставляет компилятор предупреждать об отсутствующей документации, включает
группу Clippy `pedantic` как предупреждения и делает `clippy::unwrap_used` ошибкой. Напишите публичную функцию, вызывающую `unwrap()`, и запустите
`cargo clippy`. Что вы ожидаете увидеть?

<details>
<summary>Подсказка</summary>

Две таблицы, `[lints.rust]` и `[lints.clippy]`. Группе нужен `priority = -1`.

</details>

<details>
<summary>Ответ</summary>

```toml
[lints.rust]
missing_docs = "warn"

[lints.clippy]
pedantic = { level = "warn", priority = -1 }
unwrap_used = "deny"
```

`cargo clippy` печатает ``error: used `unwrap()` on a `Result` value`` (проверка не проходит), а также предупреждения об отсутствующем разделе
`# Panics` (`missing_panics_doc`, lint из `pedantic`) и о публичных элементах без документации.

</details>

### Упражнение 3 — Документация с тестом (средне)

Напишите `pub fn is_leap_year(year: u32) -> bool` для григорианского календаря (делится на 4, кроме лет, делящихся на 100, если только они не делятся и на 400) в
библиотечном пакете `code/p08-calendar`. Задокументируйте её разделом `# Examples`, документационный тест которого проверяет 2024, 1900 и 2000, и добейтесь,
чтобы проходила `cargo clippy -- -W clippy::pedantic`.

<details>
<summary>Подсказка</summary>

Пример — блок кода внутри комментариев `///`, и он должен подключать ваш крейт по имени с подчёркиванием (`p08_calendar`). Если Clippy предложит
`is_multiple_of`, послушайтесь: этот метод есть у беззнаковых целых.

</details>

<details>
<summary>Ответ</summary>

```rust
//! Calendar helpers.

/// Tells whether a year is a leap year in the Gregorian calendar.
///
/// A year is a leap year if it is divisible by 4, except years divisible by 100,
/// unless they are also divisible by 400.
///
/// # Examples
///
/// ```
/// use p08_calendar::is_leap_year;
///
/// assert!(is_leap_year(2024));
/// assert!(!is_leap_year(1900));
/// assert!(is_leap_year(2000));
/// ```
#[must_use]
pub fn is_leap_year(year: u32) -> bool {
    year.is_multiple_of(4) && (!year.is_multiple_of(100) || year.is_multiple_of(400))
}
```

`cargo test --doc` печатает `test src/lib.rs - is_leap_year (line 10) ... ok`. Первый вариант, написанный как `year % 4 == 0 && ...`, компилируется, но lint Clippy
`manual_is_multiple_of` предлагает ровно код выше.

</details>

## Вопросы для самопроверки

1. Что обещают префиксы `as_`, `to_` и `into_`?

<details>
<summary>Ответ</summary>

`as_` — бесплатный заимствованный вид; `to_` создаёт новое значение из заимствованного (может стоить работы); `into_` потребляет `self`.

</details>

2. Чем `cargo fmt` отличается от `cargo fmt --check`?

<details>
<summary>Ответ</summary>

`cargo fmt` переписывает файлы. `cargo fmt --check` ничего не меняет: он печатает различия и завершается с кодом 1, если они есть, так что скрипт или задание CI может упасть.

</details>

3. Почему `#[expect(lint, reason = "...")]` лучше, чем `#[allow(lint)]`?

<details>
<summary>Ответ</summary>

`expect` предупреждает, когда lint больше не срабатывает, поэтому ненужные исключения удаляются, а `reason` объясняет следующему читателю, зачем исключение существует.

</details>

4. Почему поле `Celsius` приватное?

<details>
<summary>Ответ</summary>

Чтобы единственным способом создать значение была `Celsius::new`, которая проверяет правило. Публичное поле позволило бы любому построить невозможное значение и стало бы
постоянной частью API.

</details>

## Итоги

- Соглашения делают код привычным и позволяют инструментам его проверять: компилятор (регистр), `rustfmt` (оформление), Clippy (идиомы) и API Guidelines (публичное проектирование).
- Регистр: типы — `UpperCamelCase`, функции и переменные — `snake_case`, константы — `SCREAMING_SNAKE_CASE`. Префиксы несут смысл: `as_` (бесплатный вид), `to_` (новое значение),
  `into_` (потребляет), `is_`/`has_` (вопрос); у геттеров нет `get_`.
- `cargo fmt` форматирует, `cargo fmt --check` проверяет; оставляйте стиль по умолчанию.
- Lint-ы Clippy делятся на группы; `pedantic` субъективна, `restriction` нужна для выбора отдельных lint-ов. Уровни: `allow`, `warn`, `deny`, `forbid` и `expect`. Политику
  проекта задавайте в таблице `[lints]` файла `Cargo.toml`, а для исключений используйте `#[expect(..., reason = "...")]`.
- Контроль качества — это `cargo fmt --check`, `cargo clippy --all-targets -- -D warnings` и `cargo test`.
- Rust API Guidelines: реализуйте общие трейты, проверяйте значения в конструкторах, держите поля приватными, документируйте запускаемыми примерами и применяйте
  `#[non_exhaustive]` и `#[must_use]`, где они уместны.

## Источники и что почитать

- [Rust API Guidelines](https://rust-lang.github.io/api-guidelines/) — полный чек-лист с объяснением и примером для каждого пункта.
- [Rust API Guidelines — Naming](https://rust-lang.github.io/api-guidelines/naming.html) — регистр, преобразования (`as_`, `to_`, `into_`), геттеры и имена итераторов.
- [The Rust Style Guide](https://doc.rust-lang.org/stable/style-guide/) — правила форматирования, которые реализует `rustfmt`.
- [Конфигурация rustfmt](https://rust-lang.github.io/rustfmt/) — все параметры `rustfmt.toml` и какие из них требуют nightly.
- [Документация Clippy](https://doc.rust-lang.org/stable/clippy/) — группы lint-ов, настройка и `clippy.toml`.
- [The Cargo Book — раздел `[lints]`](https://doc.rust-lang.org/cargo/reference/manifest.html) — как задавать уровни lint-ов для пакета.
- [The Rust Reference — Diagnostic attributes](https://doc.rust-lang.org/reference/attributes/diagnostics.html) — `allow`, `warn`, `deny`, `forbid`, `expect` и `must_use`.

---

[🏠 Начало](../START-HERE.md) · [← 08.10 Антипаттерны и «запахи» кода](10-anti-patterns-and-code-smells.md) · [📋 Этап 08](README.md) · [08.12 Структура проекта и рабочие пространства →](12-project-layout-and-workspaces.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
