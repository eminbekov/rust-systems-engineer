<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 06.7 Декларативные макросы](07-declarative-macros.md) · [📋 Этап 06](README.md) · [06.9 Введение в async/await →](09-async-await-introduction.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/06-rust-advanced/08-procedural-macros.md) <!-- nav -->

# Урок 06.8 — Процедурные макросы

> **Этап 06 — Rust: продвинутый уровень** · Урок 8 из 12 · ⏱ около 3 часов (2 занятия)

## Что вы узнаете

- Что такое процедурный макрос, какие бывают три вида (похожий на функцию, derive, атрибутный) и почему они живут в отдельном крейте.
- Как читать код с помощью `syn` и писать код с помощью `quote`.
- Как шаг за шагом написать все три вида в небольшом рабочем пространстве Cargo.
- Как сообщать об ошибках так, чтобы они указывали на точное место в коде пользователя.
- Как отлаживать и тестировать процедурный макрос (`cargo expand`, модульные тесты, `trybuild`).

## Перед началом

- Вы прошли [Урок 06.7 — Декларативные макросы](07-declarative-macros.md). Вы должны знать, что такое макрос и токен.
- Вы знаете трейты и обобщения ([Урок 05.2](../05-rust-intermediate/02-traits.md), [Урок 05.1](../05-rust-intermediate/01-generics.md)) и умеете подключать крейты ([Урок 05.8](../05-rust-intermediate/08-cargo-dependencies-and-versions.md)).
- Установите `cargo-expand`, как показано в Уроке 06.7 (раздел 8).

## Новые слова

| Слово | Значение |
|---|---|
| **процедурный макрос** (procedural macro) | Макрос, написанный как обычная функция Rust. Он работает внутри компилятора, получает код как список токенов и возвращает новый код тоже как токены. |
| **макрос, похожий на функцию** (function-like macro) | Процедурный макрос, который вызывается как `name!(...)`. |
| **derive-макрос** (derive macro) | Процедурный макрос, который применяется как `#[derive(Name)]` над структурой или перечислением. Он добавляет новый код (обычно `impl`) рядом с типом. |
| **атрибутный макрос** (attribute macro) | Процедурный макрос, который применяется как `#[name]` над элементом, например функцией. Он получает элемент и заменяет его новым кодом. |
| **TokenStream** | Тип для «списка токенов». Макрос получает такой список и возвращает такой же. |
| **токен** (token) | Самый маленький кусочек исходного кода: имя, число, символ вроде `+`. |
| **AST** (Abstract Syntax Tree, абстрактное синтаксическое дерево) | Дерево, описывающее смысл кода, например «структура с именем User и двумя полями». |
| **API** (Application Programming Interface, программный интерфейс) | То, что библиотека предлагает своим пользователям: её функции, типы и сообщения. |
| **syn** | Крейт, который разбирает `TokenStream` в AST, который можно исследовать обычным кодом Rust. |
| **quote** | Крейт, который строит `TokenStream` из кода, написанного внутри `quote! { ... }`, с «дырками» `#variable`. |
| **proc-macro2** | Крейт со своей копией типов токенов. Он работает в обычных программах и тестах, а не только внутри компилятора. |
| **спан** (span) | Место в исходном коде, откуда взят токен. Ошибки используют спаны, чтобы указать на нужную строку. |
| **рабочее пространство** (workspace) | Группа пакетов Cargo в одном дереве папок, у которых общие `Cargo.lock` и папка `target`. |
| **тип крейта** (crate type) | Во что собирается крейт: в исполняемый файл, библиотеку или (здесь) библиотеку процедурных макросов. |
| **время компиляции** (compile time) | Время, когда `cargo build` запускает компилятор — до запуска вашей программы. |

## 1. Что такое процедурный макрос

В Уроке 06.7 вы писали макросы как *образцы*. Для многих задач этого хватает, но образец не может заглянуть внутрь `struct`, посчитать её поля, прочитать их имена или выполнить настоящую логику. **Процедурный макрос** может, потому что это настоящая функция Rust:

```text
 your source code            the compiler                 your macro (a Rust function)
┌─────────────────┐   tokens   ┌─────────────┐  TokenStream  ┌──────────────────────┐
│ #[derive(Foo)]  │ ─────────► │  finds the  │ ────────────► │ fn derive_foo(input) │
│ struct User {…} │            │  macro call │               │   -> TokenStream     │
└─────────────────┘            └─────────────┘               └──────────┬───────────┘
                                      ▲                                 │ new tokens
                                      └─────────────────────────────────┘
                     the compiler compiles the original code AND the new code
```

Процедурный макрос:

1. получает код как `TokenStream`;
2. делает всё, что умеет делать код Rust (циклы, `if`, строки, вычисления);
3. возвращает новый `TokenStream`, который компилятор компилирует вместо исходного кода (или рядом с ним).

Есть три вида:

| Вид | Как пишет пользователь | Что получает | Что делает |
|---|---|---|---|
| Похожий на функцию | `my_macro!(...)` | токены между скобками | заменяет вызов новым кодом |
| Derive | `#[derive(MyTrait)]` над структурой или перечислением | структуру или перечисление | **добавляет** новый код (обычно `impl`); исходный остаётся |
| Атрибутный | `#[my_attr]` или `#[my_attr(args)]` над элементом | аргументы и элемент | **заменяет** элемент новым кодом |

Derive-макросы вы уже использовали: `#[derive(Debug)]` — один из них (встроенный в компилятор), и `#[derive(Serialize)]` из `serde` ([Урок 05.11](../05-rust-intermediate/11-practical-crates-serde-clap-errors.md)) — это derive-макрос, который написал кто-то другой теми самыми средствами, что вы изучаете в этом уроке. В разделе 7 вы увидите и атрибутные макросы.

### Одно правило: отдельный крейт

Крейт, в котором лежат процедурные макросы, должен быть особым видом библиотеки с `proc-macro = true` в `Cargo.toml`. Его компилируют и *запускает компилятор*, поэтому он должен быть собран раньше кода, который его использует. Поэтому всегда нужны два пакета: один с макросами и один, который их использует. **Рабочее пространство** (workspace) Cargo хранит оба в одном месте.

## 2. Настройка рабочего пространства

Создайте папку рабочего пространства и два пакета:

```sh
mkdir -p ~/projects/rust/code/p06-procmacros && cd ~/projects/rust/code/p06-procmacros
printf '[workspace]\nresolver = "3"\nmembers = []\n' > Cargo.toml
cargo new --lib toolbox-macros
cargo new app
```

Cargo сам добавит `toolbox-macros` и `app` в `members`. Теперь отредактируйте `toolbox-macros/Cargo.toml`:

```toml
[package]
name = "toolbox-macros"
version = "0.1.0"
edition = "2024"

[lib]
proc-macro = true

[dependencies]
syn = { version = "3.0", features = ["full"] }
quote = "1.0"
proc-macro2 = "1.0"
```

и `app/Cargo.toml`:

```toml
[package]
name = "app"
version = "0.1.0"
edition = "2024"

[dependencies]
toolbox-macros = { path = "../toolbox-macros" }
```

**Что происходит:**

1. `[lib] proc-macro = true` говорит Cargo собрать этот крейт как библиотеку процедурных макросов.
2. `syn` разбирает токены. Фича `"full"` позволяет ему разбирать любые элементы Rust, в том числе функции (это нужно для атрибутного макроса). Без `"full"` `syn` разбирает только «входные данные derive» (структуры, перечисления, объединения).
3. `quote` и `proc-macro2` — помощники, описанные в «Новых словах». Их использует почти каждый процедурный макрос.
4. В `app` запись `path = "../toolbox-macros"` заставляет Cargo взять локальный пакет, а не crates.io.

Выполните `cargo build` в папке рабочего пространства, чтобы скачать и скомпилировать зависимости. (`syn` 3.x актуален на момент написания. Многие учебники написаны для `syn` 2; код этого урока проверен с версией 3.)

## 3. Макрос, похожий на функцию

Наш первый макрос берёт строковый литерал и превращает его в верхний регистр **во время компиляции**: `shout_const!("welcome")` становится `"WELCOME"` до запуска программы.

`toolbox-macros/src/lib.rs`:

```rust
use proc_macro::TokenStream;
use quote::quote;
use syn::{parse_macro_input, LitStr};

/// `shout_const!("hello")` becomes the string literal `"HELLO"`,
/// computed while the compiler runs.
#[proc_macro]
pub fn shout_const(input: TokenStream) -> TokenStream {
    let text = parse_macro_input!(input as LitStr);
    let upper = text.value().to_uppercase();
    quote! { #upper }.into()
}
```

`app/src/main.rs`:

```rust
use toolbox_macros::shout_const;

const BANNER: &str = shout_const!("welcome");

fn main() {
    println!("{}", BANNER);
    println!("{}", shout_const!("compile time"));
}
```

**Что происходит:**

1. `#[proc_macro]` помечает `shout_const` как макрос, похожий на функцию. Имя функции — это имя макроса. Сигнатура всегда `fn(TokenStream) -> TokenStream`.
2. `parse_macro_input!(input as LitStr)` просит `syn` разобрать входные токены как строковый литерал (`LitStr`). Если на входе не строковый литерал, макрос сообщает об ошибке компиляции и сразу возвращает управление.
3. `text.value()` даёт настоящую `String` из литерала. `to_uppercase()` — обычный Rust.
4. `quote! { #upper }` строит токены. `#upper` — это «дырка»: `quote` подставляет туда значение переменной `upper`. `String` превращается в строковый литерал в выходных токенах.
5. `.into()` преобразует `proc_macro2::TokenStream` (его делает `quote!`) в `proc_macro::TokenStream` (его ждёт компилятор).
6. В `main.rs` компилятор заменяет `shout_const!("welcome")` на `"WELCOME"` и компилирует программу.

```text
WELCOME
COMPILE TIME
```

Если пользователь передаст не строковый литерал, `syn` создаст ошибку, которая на него указывает:

```rust
use toolbox_macros::shout_const;

fn main() {
    println!("{}", shout_const!(42));
}
```

```text
error: expected string literal
 --> src/main.rs:4:33
  |
4 |     println!("{}", shout_const!(42));
  |                                 ^^
```

Стрелка указывает на `42` — именно на тот ввод, который был неверным. Так работает **спан**. Это хорошее сообщение вы получили бесплатно от `parse_macro_input!`.

> 💡 **Совет:** `proc_macro` — крейт, встроенный в компилятор. В `Cargo.toml` его добавлять не нужно. `proc_macro2`, `syn` и `quote` — обычные крейты с crates.io.

## 4. Derive-макрос

Теперь derive-макрос. Мы хотим, чтобы работало вот это:

```rust
#[derive(FieldNames)]
struct User {
    name: String,
    age: u32,
}

// User::type_name()   -> "User"
// User::field_names() -> ["name", "age"]
```

Добавьте в `toolbox-macros/src/lib.rs` (строка `use` в начале теперь перечисляет больше имён):

```rust
use proc_macro::TokenStream;
use proc_macro2::TokenStream as TokenStream2;
use quote::quote;
use syn::{parse_macro_input, Data, DeriveInput, Fields, LitStr};

/// Derive macro: adds `type_name()` and `field_names()` to a struct.
#[proc_macro_derive(FieldNames)]
pub fn derive_field_names(input: TokenStream) -> TokenStream {
    expand_field_names(input.into()).into()
}

fn expand_field_names(input: TokenStream2) -> TokenStream2 {
    let ast: DeriveInput = match syn::parse2(input) {
        Ok(ast) => ast,
        Err(error) => return error.to_compile_error(),
    };
    let name = &ast.ident;

    let fields = match &ast.data {
        Data::Struct(data) => match &data.fields {
            Fields::Named(named) => &named.named,
            _ => {
                return syn::Error::new_spanned(name, "FieldNames needs a struct with named fields")
                    .to_compile_error();
            }
        },
        _ => {
            return syn::Error::new_spanned(name, "FieldNames can only be used on structs")
                .to_compile_error();
        }
    };

    let field_names = fields.iter().map(|f| f.ident.as_ref().unwrap().to_string());
    let (impl_generics, ty_generics, where_clause) = ast.generics.split_for_impl();

    quote! {
        impl #impl_generics #name #ty_generics #where_clause {
            pub fn type_name() -> &'static str {
                stringify!(#name)
            }

            pub fn field_names() -> &'static [&'static str] {
                &[#(#field_names),*]
            }
        }
    }
}
```

**Что происходит:**

1. `#[proc_macro_derive(FieldNames)]` делает функцию реализацией `#[derive(FieldNames)]`. Имя в скобках — то, что пишут пользователи.
2. Функция derive только преобразует типы (`.into()`) и вызывает `expand_field_names`. Вся настоящая работа — в обычной функции, которая использует типы `proc_macro2`. Такое разделение делает возможным тестирование (раздел 8).
3. `DeriveInput` — это AST «структуры, перечисления или объединения». Его главные части: `ident` (имя типа), `generics` и `data` (поля или варианты).
4. Мы делаем `match` по `ast.data`. Если это структура с *именованными* полями, получаем список полей. Для перечисления или кортежной структуры возвращаем ошибку.
5. `syn::Error::new_spanned(name, "message")` создаёт ошибку, которая указывает на токены `name`. `.to_compile_error()` превращает её в токены, из-за которых компилятор напечатает эту ошибку. Мы **возвращаем** эти токены, а не паникуем.
6. `fields.iter().map(...)` делает список имён полей в виде `String`.
7. `split_for_impl()` делит обобщённые параметры на три части, которые нужны блоку `impl`: `impl<T>`, `Type<T>` и предложение `where`. Благодаря этому макрос работает и для `struct Pair<T> { ... }`.
8. В `quote!` конструкция `#name` вставляет имя типа. `#(#field_names),*` — это *повторение*, как в Уроке 06.7: она записывает каждый элемент `field_names` через запятую.

Используйте его в `app/src/main.rs`:

```rust
use toolbox_macros::FieldNames;

#[derive(FieldNames)]
struct User {
    name: String,
    age: u32,
}

#[derive(FieldNames)]
struct Pair<T> {
    left: T,
    right: T,
}

fn main() {
    println!("{} has {:?}", User::type_name(), User::field_names());
    println!("{} has {:?}", Pair::<u8>::type_name(), Pair::<u8>::field_names());

    let user = User { name: "Aziza".to_string(), age: 31 };
    println!("{} is {}", user.name, user.age);
    let pair = Pair { left: 1, right: 2 };
    println!("{}", pair.left + pair.right);
}
```

```text
User has ["name", "age"]
Pair has ["left", "right"]
Aziza is 31
3
```

### Что видит макрос

Как на самом деле выглядит `DeriveInput`? Его можно напечатать через `eprintln!("{:#?}", ast);` внутри макроса (для `Debug` нужна фича `"extra-traits"` у `syn`). Вот начало настоящего вывода для `User`, в сокращении:

```text
DeriveInput {
    attrs: [],
    vis: Visibility::Inherited,
    ident: Ident {
        ident: "User",
        span: #0 bytes(134..138),
    },
    generics: Generics { lt_token: None, params: [], gt_token: None, where_clause: None },
    data: Data::Struct {
        struct_token: Token![struct],
        fields: Fields::Named {
            brace_token: Brace,
            named: [
                Field {
                    ident: Some(Ident { ident: "name", ... }),
                    ty: Type::Path { ... "String" ... },
                    ...
                },
                Token![,],
                Field { ident: Some(Ident { ident: "age", ... }), ... },
```

Чтение такого дерева — главный навык при написании процедурных макросов: вы находите нужную часть (`data → fields → named → ident`) и пишете `match`, который до неё добирается.

### Ошибки, которые указывают на нужное место

Попробуйте применить макрос к тому, что он не поддерживает:

```rust
use toolbox_macros::FieldNames;

#[derive(FieldNames)]
enum Color {
    Red,
    Green,
}

fn main() {}
```

```text
error: FieldNames can only be used on structs
 --> src/main.rs:4:6
  |
4 | enum Color {
  |      ^^^^^
```

Для `struct Meters(f64);` будет `error: FieldNames needs a struct with named fields` с указанием на `Meters`.

> 📏 **Соглашение:** процедурный макрос не должен вызывать `panic!` из-за неверного ввода пользователя. Возвращайте `syn::Error::...to_compile_error()` с понятным сообщением и правильным спаном. Паника тоже работает, но пользователь увидит `error: proc macro panicked`, а ваш текст — только в строке «help».

### Что компилятор компилирует на самом деле

`cargo expand` (Урок 06.7) работает и для процедурных макросов. Для пакета `app` он показывает:

```text
struct User {
    name: String,
    age: u32,
}
impl User {
    pub fn type_name() -> &'static str {
        "User"
    }
    pub fn field_names() -> &'static [&'static str] {
        &["name", "age"]
    }
}
struct Pair<T> {
    left: T,
    right: T,
}
impl<T> Pair<T> {
    pub fn type_name() -> &'static str {
        "Pair"
    }
    ...
}
```

Исходная структура на месте (derive-макросы **добавляют**), а `impl` — новый. Для `Pair<T>` `split_for_impl` создал `impl<T> Pair<T>`.

## 5. Атрибутный макрос

Атрибутный макрос получает **два** потока токенов: аргументы в скобках атрибута и элемент, над которым стоит атрибут. Он возвращает код, который заменяет элемент.

Напишем `#[log_call]`: он заставляет функцию печатать строку при каждом вызове.

Сначала добавьте `ItemFn` в строку `use syn::{ ... }` в начале файла. Затем добавьте макрос:

```rust
/// Attribute macro: prints a line each time the function is called.
#[proc_macro_attribute]
pub fn log_call(_args: TokenStream, item: TokenStream) -> TokenStream {
    let mut function = parse_macro_input!(item as ItemFn);
    let name = function.sig.ident.to_string();
    let body = &function.block;
    function.block = syn::parse_quote!({
        ::std::println!("[log] calling {}", #name);
        #body
    });
    quote! { #function }.into()
}
```

Используйте его:

```rust
use toolbox_macros::log_call;

#[log_call]
fn add(a: i32, b: i32) -> i32 {
    a + b
}

fn main() {
    println!("{}", add(2, 3));
}
```

**Что происходит:**

1. `#[proc_macro_attribute]` помечает макрос, который применяется как `#[log_call]`. Первый параметр — аргументы (здесь пустые, поэтому `_args`). Второй — элемент.
2. `ItemFn` — AST целой функции: сигнатура (`sig`), видимость, атрибуты и тело (`block`). Для него нужна фича `"full"` у `syn`.
3. Имя функции мы читаем из `function.sig.ident`.
4. `syn::parse_quote!` похож на `quote!`, но разбирает токены в тип `syn`, здесь — в блок. Новый блок сначала печатает, затем выполняет старое тело (`#body`).
5. `quote! { #function }` превращает изменённую функцию обратно в токены. Компилятор компилирует *её* вместо исходной.
6. Мы написали `::std::println!` с ведущим `::`. Результат макроса вставляется в код пользователя, а у пользователя может быть собственный `println`. Запись, начинающаяся с `::std`, всегда означает настоящую стандартную библиотеку.

```text
[log] calling add
5
```

> 📏 **Соглашение:** в сгенерированном коде используйте полные пути (`::std::vec::Vec`, `::core::option::Option`), а не короткие имена. Процедурные макросы **не гигиеничны** для имён, в отличие от `macro_rules!`: сгенерированный код может столкнуться с именами пользователя.

## 6. Отладка процедурного макроса

1. **`cargo expand`** показывает итоговый код (см. выше). Смотрите сначала сюда.
2. **`eprintln!` внутри макроса** печатает, пока работает компилятор. Текст появится в выводе `cargo build`. Используйте `eprintln!("{:#?}", ast)` (с фичей `"extra-traits"`), чтобы увидеть дерево. Удалите его перед коммитом.
3. **Маленький тестовый крейт** с одной структурой. Пусть он падает быстро, и читайте первую ошибку: ошибки макросов тянутся друг за другом.
4. **`cargo build -p app`** пересобирает только то, что изменилось. Когда вы правите крейт с макросами, пересобираются все крейты, которые его используют.

## 7. Где это встречается в настоящих проектах

- `serde` — это два крейта: `serde` (трейты `Serialize` и `Deserialize`) и `serde_derive` (derive-макросы). Крейт `serde` реэкспортирует derive-макросы (когда включена его фича `derive`), поэтому пользователи пишут `use serde::Serialize;` и получают оба. Так принято поставлять трейт вместе с его derive-макросом, потому что крейт типа `proc-macro` может экспортировать *только* макросы, а не трейты.
- `clap` (`#[derive(Parser)]`), `thiserror` (`#[derive(Error)]`) и `tokio` (`#[tokio::main]` — атрибутный макрос, который вы будете использовать в [Уроке 06.9](09-async-await-introduction.md)) — тоже процедурные макросы.

> 🔒 **Безопасность:** процедурный макрос — это программа, которую компилятор запускает на вашем компьютере во время сборки. Она может читать ваши файлы и пользоваться сетью с вашими правами, так же как скрипт `build.rs`. Вредоносный макрос в зависимости — риск для *цепочки поставок* (атака через библиотеки, которыми вы пользуетесь). Мы изучим это на этапе о безопасности. Пока: подключайте только крейты, которым доверяете, и смотрите на популярность крейта и его авторов.

> ⚡ **Производительность:** `syn` с `"full"` и `"extra-traits"` медленно компилируется. Включайте только нужные фичи. Derive-макрос добавляет время компиляции каждому крейту, который его использует, но не добавляет затрат во время выполнения: сгенерированный код — обычный код.

## 8. Тестирование процедурного макроса

Есть три уровня — от дешёвого к точному.

**1. Интеграционные тесты в крейте пользователя.** Положите файл с тестами в `app/tests/`. Он использует макрос так, как использовал бы пользователь:

```rust
use toolbox_macros::{shout_const, FieldNames};

#[derive(FieldNames)]
struct Point {
    x: i32,
    y: i32,
}

#[test]
fn derive_lists_field_names() {
    assert_eq!(Point::type_name(), "Point");
    assert_eq!(Point::field_names(), ["x", "y"]);
}

#[test]
fn shout_const_makes_upper_case() {
    assert_eq!(shout_const!("rust"), "RUST");
}

#[test]
fn point_fields_are_usable() {
    let p = Point { x: 1, y: 2 };
    assert_eq!(p.x + p.y, 3);
}
```

**2. Модульные тесты раскрытия.** Поскольку `expand_field_names` принимает и возвращает типы `proc_macro2`, её можно вызвать из обычного `#[test]` внутри крейта с макросами. Добавьте в конец `toolbox-macros/src/lib.rs`:

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn struct_gets_both_functions() {
        let input = quote! { struct Point { x: i32, y: i32 } };
        let output = expand_field_names(input).to_string();
        assert!(output.contains("type_name"));
        assert!(output.contains("\"x\""));
        assert!(output.contains("\"y\""));
    }

    #[test]
    fn enum_is_rejected_with_a_compile_error() {
        let input = quote! { enum Color { Red, Green } };
        let output = expand_field_names(input).to_string();
        assert!(output.contains("compile_error"));
        assert!(output.contains("only be used on structs"));
    }
}
```

Функции, использующие `proc_macro::TokenStream` (без `2`), паникуют, если вызвать их вне вызова макроса, поэтому мы держим логику в функциях на `proc_macro2`.

**3. Тесты на ошибку компиляции с `trybuild`.** Вы хотите быть уверены, что сообщение об ошибке хорошее. Крейт `trybuild` компилирует маленькие файлы и сравнивает вывод компилятора с сохранённым файлом `.stderr`. В `app/Cargo.toml`:

```toml
[dev-dependencies]
trybuild = "1.0"
```

`app/tests/ui.rs`:

```rust
#[test]
fn ui() {
    let t = trybuild::TestCases::new();
    t.compile_fail("tests/ui/*.rs");
}
```

Положите пример с `enum Color` в `app/tests/ui/enum.rs`. Первый запуск запишет вывод компилятора в `app/wip/enum.stderr`, а тест упадёт с сообщением `successfully created new stderr files`. Прочитайте файл и, если он такой, как вы хотите, переместите его:

```sh
mv app/wip/enum.stderr app/tests/ui/enum.stderr
cargo test
```

```text
running 1 test
test tests/ui/enum.rs ... ok
test ui ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.13s
```

> 🧪 **Тестирование:** проверяйте основной сценарий в `tests/`, логику — модульными тестами на `proc_macro2`, а сообщения об ошибках — через `trybuild`. Ваши пользователи читают эти сообщения, так что они часть вашего API (Application Programming Interface, программного интерфейса: того, что ваш код предлагает людям, которые им пользуются).

## Частые ошибки

**1. Забыли `proc-macro = true`.**

```text
error: the `proc_macro` attribute is only usable with crates of the `proc-macro` crate type
 --> src/lib.rs:4:3
  |
4 | #[proc_macro]
  |   ^^^^^^^^^^
```

Исправление: добавьте `[lib] proc-macro = true` в `Cargo.toml` крейта с макросами.

**2. Экспорт обычных элементов из крейта с макросами.**

```text
error: `proc-macro` crate types currently cannot export any items other than functions tagged with `#[proc_macro]`, `#[proc_macro_derive]`, or `#[proc_macro_attribute]`
 --> src/lib.rs:4:1
  |
4 | pub const ANSWER: i32 = 42;
  | ^^^^^^^^^^^^^^^^^^^^^^^^^^^
```

Исправление: крейт с макросами может экспортировать только макросы. Сделайте вспомогательные элементы приватными (без `pub`) или вынесите общий код в третий, обычный крейт (поэтому `serde` и `serde_derive` — разные крейты).

**3. Возврат не того типа `TokenStream`.**

```text
error[E0308]: mismatched types
  --> src/lib.rs:12:5
   |
11 | pub fn five(_input: TokenStream) -> TokenStream {
   |                                     ----------- expected `TokenStream` because of return type
12 |     quote! { 5 }
   |     ^^^^^^^^^^^^ expected `TokenStream`, found `proc_macro2::TokenStream`
   |
help: call `Into::into` on this expression to convert `proc_macro2::TokenStream` into `TokenStream`
```

Исправление: добавьте `.into()`: `quote! { 5 }.into()`.

**4. Забыли фичу `syn`.** Использование `ItemFn` без фичи `"full"`:

```text
error[E0432]: unresolved import `syn::ItemFn`
note: found an item that was configured out
  |
  | #[cfg(feature = "full")]
  |       ---------------- the item is gated behind the `full` feature
```

Исправление: `syn = { version = "3.0", features = ["full"] }`.

**5. Паника при неверном вводе.** `panic!` или `unwrap()` в макросе дают пользователю:

```text
error: proc macro panicked
 --> src/main.rs:2:5
  |
2 |     oops::boom!();
  |     ^^^^^^^^^^^^^
  |
  = help: message: I do not like this input
```

Это работает, но не указывает на настоящую проблему. Возвращайте `syn::Error` со спаном.

**6. Использование процедурного макроса в том же крейте, где он определён.** Это не работает: макрос должен быть скомпилирован раньше кода, который его использует. Всегда используйте второй пакет (или тест в `tests/`).

## Попробуйте сами

1. Постройте рабочее пространство из разделов 2–5. Выполните `cargo run -p app` и убедитесь, что видите `WELCOME`, имена полей и `[log] calling add`.
2. Выполните `cargo expand -p app` и найдите блок `impl User`.
3. Сломайте нарочно: замените `shout_const!("welcome")` на `shout_const!(42)`. Прочитайте ошибку и найдите спан.
4. Выполните `cargo test --workspace`. Вы должны увидеть, что интеграционные тесты, модульные тесты и тест `trybuild` проходят.
5. Сохраните работу (учебный репозиторий из Этапа 01):

   ```sh
   git add code/p06-procmacros
   git commit -m "feat(p06-procmacros): add shout_const, FieldNames and log_call macros"
   ```

   Не коммитьте папку `target`. Если `git status` её показывает, сначала добавьте `target/` в `.gitignore`.

## Упражнения

Все упражнения расширяют крейт `toolbox-macros` из этого урока.

### Упражнение 1 — `const_len!` (легко)

Напишите макрос, похожий на функцию, `const_len!("hello")`, который раскрывается в число байтов строкового литерала: `5`. Проверьте его на `const N: usize = const_len!("hello");`.

<details>
<summary>Подсказка</summary>

Скопируйте `shout_const`. Вместо `to_uppercase()` вызовите `text.value().len()`. Переменную типа `usize` можно вставить через `#len` в `quote!`.

</details>

<details>
<summary>Ответ</summary>

```rust
/// `const_len!("text")` becomes the number `4` (the length in bytes).
#[proc_macro]
pub fn const_len(input: TokenStream) -> TokenStream {
    let text = parse_macro_input!(input as LitStr);
    let len = text.value().len();
    quote! { #len }.into()
}
```

Тест (в `app/tests/derive.rs`):

```rust
use toolbox_macros::const_len;

#[test]
fn const_len_counts_bytes() {
    const N: usize = const_len!("hello");
    assert_eq!(N, 5);
}
```

`cargo test` покажет `test const_len_counts_bytes ... ok`.

Почему это работает: длину считает компилятор. `N` — константа, хотя макрос выполнил настоящий код.

</details>

### Упражнение 2 — `field_count()` с тестом (средне)

Расширьте `FieldNames` так, чтобы он ещё генерировал `pub fn field_count() -> usize`. Добавьте модульный тест в крейт с макросами, который проверяет, что сгенерированные токены содержат `field_count`, и дополните интеграционный тест проверкой, что `Point::field_count()` равно `2`.

<details>
<summary>Подсказка</summary>

Внутри блока `quote!` добавьте вторую функцию с телом `Self::field_names().len()`. В тесте достаточно `output.contains("field_count")`.

</details>

<details>
<summary>Ответ</summary>

Внутри `impl` в `quote!`, после `field_names`:

```rust
            pub fn field_count() -> usize {
                Self::field_names().len()
            }
```

Модульный тест в крейте с макросами:

```rust
    #[test]
    fn struct_gets_field_count() {
        let input = quote! { struct Point { x: i32, y: i32 } };
        let output = expand_field_names(input).to_string();
        assert!(output.contains("field_count"));
    }
```

В интеграционном тесте добавьте `assert_eq!(Point::field_count(), 2);` в `derive_lists_field_names`.

Почему это работает: `field_count` использует сгенерированную `field_names`, поэтому список известен только в одном месте.

</details>

### Упражнение 3 — `#[log_call("DEBUG")]` (средне)

Сделайте так, чтобы `log_call` принимал необязательную метку в скобках. `#[log_call("DEBUG")]` печатает `[DEBUG] calling add`; простой `#[log_call]` по-прежнему печатает `[log] calling add`.

<details>
<summary>Подсказка</summary>

Первый параметр атрибутного макроса (`args`) содержит токены в скобках. Сначала проверьте `args.is_empty()`. Если не пусто, разберите как `LitStr` через `parse_macro_input!(args as LitStr)` и возьмите `.value()`. Вставьте метку через `#tag`.

</details>

<details>
<summary>Ответ</summary>

```rust
#[proc_macro_attribute]
pub fn log_call(args: TokenStream, item: TokenStream) -> TokenStream {
    let tag = if args.is_empty() {
        "log".to_string()
    } else {
        parse_macro_input!(args as LitStr).value()
    };
    let mut function = parse_macro_input!(item as ItemFn);
    let name = function.sig.ident.to_string();
    let body = &function.block;
    function.block = syn::parse_quote!({
        ::std::println!("[{}] calling {}", #tag, #name);
        #body
    });
    quote! { #function }.into()
}
```

Тест:

```rust
use toolbox_macros::log_call;

#[log_call("DEBUG")]
fn add(a: i32, b: i32) -> i32 {
    a + b
}

#[test]
fn tagged_log_still_returns_value() {
    assert_eq!(add(2, 3), 5);
}
```

Запустите `cargo test -- --nocapture`, чтобы увидеть строку `[DEBUG] calling add`.

Почему это работает: аргументы атрибута и элемент — два отдельных потока токенов, поэтому макрос может прочитать оба.

</details>

### Упражнение 4 — тест `trybuild` для кортежной структуры (средне)

Добавьте тест на ошибку компиляции, который показывает, что `#[derive(FieldNames)] struct Meters(f64);` отвергается с сообщением `FieldNames needs a struct with named fields`.

<details>
<summary>Подсказка</summary>

Создайте `app/tests/ui/tuple.rs` со структурой и пустой `fn main() {}`. Запустите `cargo test`, затем переместите файл из `app/wip/` в `app/tests/ui/`, как для перечисления.

</details>

<details>
<summary>Ответ</summary>

`app/tests/ui/tuple.rs`:

```rust
use toolbox_macros::FieldNames;

#[derive(FieldNames)]
struct Meters(f64);

fn main() {}
```

`app/tests/ui/tuple.stderr` (это настоящий вывод):

```text
error: FieldNames needs a struct with named fields
 --> tests/ui/tuple.rs:4:8
  |
4 | struct Meters(f64);
  |        ^^^^^^
```

После этого `cargo test` покажет `test tests/ui/tuple.rs ... ok`.

Почему это работает: `trybuild` компилирует каждый файл, подходящий под `tests/ui/*.rs`, и сравнивает вывод компилятора с файлом `.stderr` рядом. Если кто-то позже испортит сообщение об ошибке, тест упадёт.

</details>

### Упражнение 5 — derive `VariantNames` для перечислений (сложно)

Напишите `#[derive(VariantNames)]`, который добавляет `pub fn variant_names() -> &'static [&'static str]` к **перечислению**. Для `enum Color { Red, Green, Blue }` он возвращает `["Red", "Green", "Blue"]`. Для структуры должна быть ошибка компиляции с понятным сообщением.

<details>
<summary>Подсказка</summary>

Делайте `match` по `ast.data`: на этот раз нужен `Data::Enum(data)`, а `data.variants` — это список. У каждого варианта есть `ident`. Используйте `let ... else`, чтобы вернуть ошибку сразу: `let Data::Enum(data) = &ast.data else { return ... };`.

</details>

<details>
<summary>Ответ</summary>

```rust
/// Derive macro: adds `variant_names()` to an enum.
#[proc_macro_derive(VariantNames)]
pub fn derive_variant_names(input: TokenStream) -> TokenStream {
    let ast = parse_macro_input!(input as DeriveInput);
    let name = &ast.ident;

    let Data::Enum(data) = &ast.data else {
        return syn::Error::new_spanned(name, "VariantNames can only be used on enums")
            .to_compile_error()
            .into();
    };

    let variant_names = data.variants.iter().map(|v| v.ident.to_string());
    let (impl_generics, ty_generics, where_clause) = ast.generics.split_for_impl();

    quote! {
        impl #impl_generics #name #ty_generics #where_clause {
            pub fn variant_names() -> &'static [&'static str] {
                &[#(#variant_names),*]
            }
        }
    }
    .into()
}
```

Тест:

```rust
use toolbox_macros::VariantNames;

#[derive(VariantNames)]
enum Color {
    Red,
    Green,
    Blue,
}

#[test]
fn variant_names_for_enum() {
    assert_eq!(Color::variant_names(), ["Red", "Green", "Blue"]);
    let _ = (Color::Red, Color::Green, Color::Blue);
}
```

Почему это работает: форма та же, что у `FieldNames`. Отличается только та часть дерева, по которой вы идёте: `Data::Enum` и `variants` вместо `Data::Struct` и `fields`.

</details>

## Вопросы для самопроверки

1. Какие три вида процедурных макросов бывают? Приведите по одному примеру.

<details>
<summary>Ответ</summary>

Похожий на функцию (`shout_const!("x")`), derive (`#[derive(FieldNames)]`) и атрибутный (`#[log_call]`). Настоящие примеры: `Serialize` из `serde` — derive, а `#[tokio::main]` — атрибутный макрос.

</details>

2. Почему процедурному макросу нужен отдельный крейт?

<details>
<summary>Ответ</summary>

Компилятор запускает макрос, пока компилирует код, который его использует, поэтому макрос должен быть скомпилирован первым — как отдельная библиотека с `proc-macro = true`. Крейт такого типа может экспортировать только макросы.

</details>

3. Что делают `syn` и `quote`?

<details>
<summary>Ответ</summary>

`syn` разбирает `TokenStream` в дерево (AST), которое можно исследовать обычным кодом Rust. `quote!` строит `TokenStream` из похожего на Rust кода с «дырками» `#variable`.

</details>

4. Чем отличаются `proc_macro::TokenStream` и `proc_macro2::TokenStream`? Зачем использовать второй?

<details>
<summary>Ответ</summary>

Первый существует только внутри компилятора и не годится для обычных тестов. Второй — обычный тип, работающий где угодно. `quote!` и `syn` используют `proc_macro2`. Преобразование делается через `.into()` на краях функции макроса.

</details>

5. Как макрос должен сообщить, что пользователь применил его неверно?

<details>
<summary>Ответ</summary>

Вернуть `syn::Error::new_spanned(tokens, "message").to_compile_error()`. Тогда компилятор напечатает сообщение и укажет на заданные токены. Не используйте `panic!`.

</details>

6. Зачем derive-макрос использует `split_for_impl()`?

<details>
<summary>Ответ</summary>

Он делит обобщённые параметры типа на три части, которые нужны `impl` (`impl<T>`, `Type<T>` и предложение `where`), поэтому сгенерированный `impl` работает и для обобщённых типов.

</details>

7. Почему процедурный макрос может быть риском для безопасности?

<details>
<summary>Ответ</summary>

Это программа, которая запускается на вашем компьютере во время сборки, с вашими правами. Вредоносный макрос в зависимости мог бы читать файлы или использовать сеть.

</details>

## Итоги

- Процедурный макрос — это функция Rust, которая во время компиляции превращает `TokenStream` в новый `TokenStream`.
- Виды: похожий на функцию (`name!`), derive (`#[derive(Name)]`) и атрибутный (`#[name]`).
- Они живут в библиотечном крейте с `proc-macro = true`, который может экспортировать только макросы. Используйте рабочее пространство со вторым пакетом, который их применяет.
- `syn` разбирает код в AST (`LitStr`, `DeriveInput`, `ItemFn`). `quote!` пишет код обратно, с `#var` и `#(#items),*`.
- Сообщайте об ошибках через `syn::Error::new_spanned(...).to_compile_error()`, а не через `panic!`.
- В сгенерированном коде используйте полные пути, например `::std::println!`.
- Отлаживайте через `cargo expand` и `eprintln!`. Тестируйте интеграционными тестами, модульными тестами на `proc_macro2` и `trybuild`.
- Макрос запускает код на вашей машине во время сборки, поэтому доверяйте зависимостям с макросами.

## Источники и что почитать

- [The Rust Programming Language — 20.5 Macros (procedural macros)](https://doc.rust-lang.org/book/ch20-05-macros.html) — вводная часть книги о трёх видах макросов с примером derive.
- [The Rust Reference — Procedural macros](https://doc.rust-lang.org/reference/procedural-macros.html) — точные правила для каждого вида.
- [syn on docs.rs](https://docs.rs/syn/3/syn/) — полный список типов AST, фич и макрос `parse_macro_input!`.
- [quote on docs.rs](https://docs.rs/quote/latest/quote/) — синтаксис `quote!`: `#var`, повторения и `ToTokens`.
- [proc-macro-workshop](https://github.com/dtolnay/proc-macro-workshop) — упражнения (derive для builder, атрибут sorted и другие) от автора `syn` и `quote`.
- [trybuild](https://docs.rs/trybuild/latest/trybuild/) — тесты на ошибку компиляции для макросов.

---

[🏠 Начало](../START-HERE.md) · [← 06.7 Декларативные макросы](07-declarative-macros.md) · [📋 Этап 06](README.md) · [06.9 Введение в async/await →](09-async-await-introduction.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
