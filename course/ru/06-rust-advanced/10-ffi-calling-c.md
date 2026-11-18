<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 06.9 Введение в async/await](09-async-await-introduction.md) · [📋 Этап 06](README.md) · [06.11 Проект: многопоточный веб-сервер →](11-project-multithreaded-web-server.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/06-rust-advanced/10-ffi-calling-c.md) <!-- nav -->

# Урок 06.10 — FFI: вызов кода на C

> **Этап 06 — Rust: продвинутый уровень** · Урок 10 из 12 · ⏱ около 4 часов (2 занятия)

## Что вы узнаете

- Что такое FFI и C ABI и зачем они системным программистам.
- Как объявлять и вызывать функции C из Rust (`unsafe extern "C"`), с правильными типами и строками (`CString`, `CStr`).
- Как скомпилировать собственный код на C с помощью скрипта `build.rs` и крейта `cc`.
- Как обернуть небезопасные вызовы C в небольшой безопасный API на Rust (проверки, `Result`, `Drop`).
- Как вызывать Rust из C и что делает `bindgen`.
- Правила, которые не дают FFI-коду вызвать неопределённое поведение.

## Перед началом

- Вы прошли [Урок 06.9 — Введение в async/await](09-async-await-introduction.md).
- Вы знаете `unsafe`, сырые указатели и комментарии `// SAFETY:` из [Урока 06.6 — Небезопасный Rust и Miri](06-unsafe-rust-and-miri.md). В этом уроке они нужны в каждом разделе.
- Установлен компилятор C. На вашем Mac он появился вместе с инструментами командной строки Xcode на Этапе 00. Проверьте:

  ```sh
  clang --version
  ```

- Создайте проект: `cd ~/projects/rust && cargo new --lib code/p06-ffi --name geo`.

## Новые слова

| Слово | Значение |
|---|---|
| **FFI** (Foreign Function Interface, интерфейс внешних функций) | Способ, которым один язык программирования вызывает функции, написанные на другом языке. |
| **ABI** (Application Binary Interface, двоичный интерфейс приложений) | Низкоуровневые правила вызова функции в скомпилированном коде: куда кладутся аргументы, откуда возвращается результат, как типы расположены в памяти. Два куска скомпилированного кода могут работать вместе, только если следуют одному ABI. |
| **C ABI** | ABI языка C. Это общий язык почти всех языков программирования, поэтому любой язык может вызвать C и быть вызванным из C. |
| **libc** | Стандартная библиотека C: функции вроде `strlen`, `abs`, `malloc`, `free`. Ваши программы на Rust уже её подключают. |
| **заголовочный файл** (header file, `.h`) | Файл C, в котором перечислены функции и типы библиотеки, без их кода. |
| **NUL-байт** (NUL byte) | Байт со значением `0`. Строки C заканчиваются им, чтобы C знал, где текст кончается. |
| **сырой указатель** (raw pointer) | Указатель без правил безопасности: `*const T` (чтение) или `*mut T` (чтение и запись). Может быть нулевым или висячим. Вы познакомились с ними в Уроке 06.6. |
| **нулевой указатель** (null pointer) | Указатель, который никуда не указывает (адрес 0). Функции C используют его, чтобы сказать «ничего». |
| **висячий указатель** (dangling pointer) | Указатель на память, которая уже освобождена или больше не содержит значения. |
| **`#[repr(C)]`** | Атрибут, который заставляет структуру Rust иметь то же расположение в памяти, что и такая же структура в C. |
| **API** (Application Programming Interface, программный интерфейс) | То, что библиотека предлагает своим пользователям: её функции и типы. «Безопасный API» — такой, которым нельзя пользоваться так, чтобы нарушить безопасность памяти. |
| **обратный вызов** (callback) | Функция, которую вы передаёте другому коду, чтобы он вызвал её обратно. |
| **скрипт сборки** (build script, `build.rs`) | Программа на Rust в корне пакета, которую Cargo запускает до компиляции вашего крейта. Она может компилировать код на C, создавать файлы и печатать инструкции для Cargo. |
| **статическая библиотека** (static library) | Файл (`.a` на macOS и Linux) со скомпилированным кодом, который компоновщик копирует в итоговую программу. |
| **компоновка** (linking) | Последний шаг сборки: он соединяет скомпилированные части и находит функции, которые они вызывают друг у друга. |
| **bindgen** | Инструмент, который читает заголовочный файл C и пишет соответствующие объявления на Rust. |
| **RAII** (Resource Acquisition Is Initialization, получение ресурса есть инициализация) | Шаблон: значение владеет ресурсом (память, файл, выделение в C) и возвращает его в своём `Drop`. |
| **UB** (Undefined Behaviour, неопределённое поведение) | Состояние программы, при котором язык ничего не гарантирует, поэтому компилятор может выдать любой результат. |
| **UTF-8** (8-bit Unicode Transformation Format, 8-битный формат преобразования Юникода) | Стандартный способ хранить текст в виде байтов. `str` в Rust всегда корректный UTF-8; строки C — просто байты. |

## 1. Зачем нужен FFI

Важные библиотеки мира написаны на C: `zlib` (сжатие), SQLite (база данных), OpenSSL (криптография) и собственные интерфейсы операционной системы. Системный инженер должен уметь ими пользоваться — и быть доступным для них: другие языки (Python, Java) тоже вызывают Rust через C.

C был создан до появления многих идей Rust. В C функция не говорит, кто владеет указателем, как долго он действителен и может ли он быть нулевым. Rust ничего из этого проверить не может, поэтому **каждый вызов через границу — `unsafe`**, и вы, программист, обещаете, что правила соблюдены. Искусство FFI — сделать эту границу маленькой и хорошо охраняемой:

```text
 your safe Rust code
┌────────────────────────────────────────┐
│  geo::distance(&a, &b)                 │   safe API: types, Result, Drop
├────────────────────────────────────────┤
│  small module with `unsafe` blocks     │   checks + // SAFETY: comments
├────────────────────────────────────────┤
│  unsafe extern "C" { fn point_distance }│   raw declarations
└───────────────────┬────────────────────┘
                    ▼
        C library (point_distance …)
```

## 2. Вызов уже существующих функций C

Ваша программа на Rust уже скомпонована со стандартной библиотекой C (libc), поэтому её функции доступны. Их нужно только *объявить*. Сделайте черновой проект для экспериментов этого раздела (`cargo new code/p06-ffi-libc`) и положите это в его `src/main.rs`:

```rust
use std::ffi::{CString, c_char, c_int};

unsafe extern "C" {
    // `safe fn`: calling it cannot cause undefined behaviour.
    pub safe fn abs(input: c_int) -> c_int;

    // A plain `fn` in an `unsafe extern` block is unsafe to call.
    pub fn strlen(s: *const c_char) -> usize;
}

fn main() {
    println!("abs(-7) = {}", abs(-7));

    let text = CString::new("systems").unwrap();
    // SAFETY: `text.as_ptr()` points to a valid, NUL-terminated string
    // that stays alive until the end of the call.
    let length = unsafe { strlen(text.as_ptr()) };
    println!("strlen = {length}");
}
```

**Что происходит:**

1. `unsafe extern "C" { ... }` — блок **объявлений**: он сообщает компилятору Rust «эти функции существуют где-то и используют C ABI». Кода внутри нет. (В редакции Rust 2024 слово `unsafe` перед `extern` обязательно: вы обещаете, что объявления верны.)
2. `pub safe fn abs(...)` помечает функцию как безопасную для вызова: она не может повредить память. Тогда компилятор позволяет вызвать `abs(-7)` без блока `unsafe`.
3. У `pub fn strlen(...)` пометки нет, поэтому её вызов небезопасен. Она читает память по указателю, пока не найдёт NUL-байт, так что неверный указатель — катастрофа.
4. `CString::new("systems")` делает строку в стиле C: байты текста и NUL-байт в конце. `text.as_ptr()` даёт указатель `*const c_char` на первый байт.
5. Блок `unsafe` содержит вызов, а комментарий `// SAFETY:` объясняет, почему вызов корректен: указатель действителен, строка заканчивается NUL, а `text` жив во время вызова.

```text
abs(-7) = 7
strlen = 7
```

### Типы: `c_int`, `c_char` и другие

`int` в C не везде одного размера. Поэтому в `std::ffi` есть псевдонимы типов, которые на вашей платформе всегда совпадают с C: `c_int` (C `int`), `c_char`, `c_long`, `c_double` и так далее. Используйте их в объявлениях вместо `i32`.

| Тип в C | Тип в Rust |
|---|---|
| `int` | `c_int` (`i32` на вашем Mac) |
| `long` | `c_long` (64 бита на macOS и Linux, 32 бита на Windows) |
| `double` | `c_double` (`f64`) |
| `char` | `c_char` |
| `size_t` | `usize` |
| `const char *` | `*const c_char` |
| `int *` (выходной) | `*mut c_int` |
| `struct Point *` | `*mut Point` (с `#[repr(C)]` у `Point`) |

### Строки: `CString` и `CStr`

`String` в Rust — не строка C: в конце у неё нет NUL, а внутри могут быть NUL-байты. Между ними преобразуют два типа:

- **`CString`** — *владеющая* строка C. Вы создаёте её из данных Rust, чтобы **передать в C**. Создание не удаётся, если внутри текста есть NUL-байт.
- **`CStr`** — *заимствованный* вид строки C. Вы создаёте её из указателя, который дал C, чтобы **прочитать в Rust**.

```rust
use std::ffi::{CStr, CString};

fn main() {
    let ok = CString::new("hello").unwrap();
    println!("{:?}", ok);
    println!("{:?}", ok.as_bytes_with_nul());

    let bad = CString::new("he\0llo");
    println!("{:?}", bad);

    let bytes = b"from C\0";
    let view = CStr::from_bytes_with_nul(bytes).unwrap();
    println!("{}", view.to_str().unwrap());
}
```

**Что происходит:**

1. `CString::new("hello")` возвращает `Ok`. `as_bytes_with_nul()` показывает пять букв и последний `0`.
2. `CString::new("he\0llo")` возвращает `Err(NulError(2, ...))`: NUL на позиции 2 для C обрезал бы строку, поэтому Rust отказывается.
3. `CStr::from_bytes_with_nul` проверяет, что байты заканчиваются ровно одним NUL, и заимствует их. `to_str()` проверяет, что это корректный UTF-8, и даёт `&str`.

```text
"hello"
[104, 101, 108, 108, 111, 0]
Err(NulError(2, [104, 101, 0, 108, 108, 111]))
from C
```

> 🔒 **Безопасность:** текст снаружи не всегда корректный UTF-8. Никогда не вызывайте `.unwrap()` у `to_str()` для строк, которые дал C или пользователь; обрабатывайте ошибку (верните код ошибки или используйте `to_string_lossy()`). И не позволяйте NUL-байту в пользовательском тексте молча обрезать строку: это приводило к настоящим уязвимостям.

## 3. Компиляция собственного кода на C

Большая часть настоящей работы с FFI использует библиотеку на C, которую вы собираете или подключаете сами. Мы напишем маленькую библиотеку и обёртку на Rust. Структура пакета:

```text
code/p06-ffi/
├── Cargo.toml
├── build.rs            ← компилирует код на C
├── c/
│   ├── geometry.h      ← объявления на C
│   └── geometry.c      ← код на C
├── src/
│   ├── lib.rs          ← безопасный API на Rust
│   └── ffi.rs          ← «сырые» объявления `unsafe extern`
└── examples/
    └── demo.rs
```

`c/geometry.h`:

```c
#ifndef GEOMETRY_H
#define GEOMETRY_H

#include <stddef.h>

typedef struct {
    double x;
    double y;
} Point;

/* Distance between two points. */
double point_distance(const Point *a, const Point *b);

/* Sum of `len` integers. */
long sum_array(const int *data, size_t len);

/* Divides a by b and stores the result in *out.
   Returns 0 on success and -1 if b is zero. */
int safe_divide(int a, int b, int *out);

/* Returns a new string "Hello, <name>!" allocated with malloc.
   The caller must give it back to free_greeting. */
char *make_greeting(const char *name);
void free_greeting(char *greeting);

/* Calls `callback(value)` for each of the `len` integers and stores
   the results in `out`. */
void map_ints(const int *data, size_t len, int (*callback)(int), int *out);

#endif
```

`c/geometry.c`:

```c
#include "geometry.h"

#include <math.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

double point_distance(const Point *a, const Point *b) {
    double dx = a->x - b->x;
    double dy = a->y - b->y;
    return sqrt(dx * dx + dy * dy);
}

long sum_array(const int *data, size_t len) {
    long total = 0;
    for (size_t i = 0; i < len; i++) {
        total += data[i];
    }
    return total;
}

int safe_divide(int a, int b, int *out) {
    if (b == 0) {
        return -1;
    }
    *out = a / b;
    return 0;
}

char *make_greeting(const char *name) {
    size_t size = strlen(name) + 9; /* "Hello, " + "!" + NUL */
    char *result = malloc(size);
    if (result == NULL) {
        return NULL;
    }
    snprintf(result, size, "Hello, %s!", name);
    return result;
}

void free_greeting(char *greeting) {
    free(greeting);
}

void map_ints(const int *data, size_t len, int (*callback)(int), int *out) {
    for (size_t i = 0; i < len; i++) {
        out[i] = callback(data[i]);
    }
}
```

Прочитайте код на C один раз и обратите внимание на типичные привычки C: функции возвращают **код ошибки** и пишут результат через указатель (`safe_divide`); память берётся из `malloc` и должна вернуться в `free` (`make_greeting`/`free_greeting`); массив — это указатель плюс длина.

Теперь скрипт сборки. `Cargo.toml`:

```toml
[package]
name = "geo"
version = "0.1.0"
edition = "2024"

[dependencies]

[build-dependencies]
cc = "1.11"
```

`build.rs`:

```rust
fn main() {
    cc::Build::new()
        .file("c/geometry.c")
        .compile("geometry");

    println!("cargo::rerun-if-changed=c/geometry.c");
    println!("cargo::rerun-if-changed=c/geometry.h");
}
```

**Что происходит:**

1. Cargo видит `build.rs` в корне пакета. Перед компиляцией `geo` он компилирует и запускает этот скрипт. Зависимости скрипта указываются в `[build-dependencies]`.
2. `cc::Build::new().file(...).compile("geometry")` находит ваш компилятор C (`clang` на macOS), компилирует файл, упаковывает его в `libgeometry.a` и печатает инструкции для Cargo, по которым Cargo подключит эту библиотеку к вашему крейту.
3. Строки, начинающиеся с `cargo::`, — инструкции для Cargo. `cargo::rerun-if-changed=...` значит: запускать скрипт снова только при изменении этих файлов.

### Сырые объявления

`src/ffi.rs` переносит заголовок C в Rust, функцию за функцией:

```rust
//! Raw declarations of the C functions. Everything here is unsafe to use.

use std::ffi::{c_char, c_double, c_int, c_long};

#[repr(C)]
#[derive(Debug, Clone, Copy)]
pub struct Point {
    pub x: c_double,
    pub y: c_double,
}

unsafe extern "C" {
    pub fn point_distance(a: *const Point, b: *const Point) -> c_double;
    pub fn sum_array(data: *const c_int, len: usize) -> c_long;
    pub fn safe_divide(a: c_int, b: c_int, out: *mut c_int) -> c_int;
    pub fn make_greeting(name: *const c_char) -> *mut c_char;
    pub fn free_greeting(greeting: *mut c_char);
    pub fn map_ints(
        data: *const c_int,
        len: usize,
        callback: extern "C" fn(c_int) -> c_int,
        out: *mut c_int,
    );
}
```

**Что происходит:**

1. `#[repr(C)]` у `Point` заставляет Rust разместить два поля `f64` в том порядке и с теми выравниваниями, какие использует C: 16 байт, `x` по смещению 0, `y` по смещению 8. Без него Rust может переставить поля, и C прочитает мусор.
2. Каждое объявление должно в точности совпадать с заголовком C. Компилятор **не может это проверить**: если вы объявите `int` как `i64`, получите неопределённое поведение без предупреждения. (Поэтому люди используют `bindgen`, см. раздел 7.)
3. `callback: extern "C" fn(c_int) -> c_int` — указатель на функцию с соглашением о вызовах C. Функцию Rust можно использовать как обратный вызов, только если она объявлена `extern "C"`.

## 4. Безопасная обёртка

Сырые функции не стоит использовать по всей программе. Оберните их один раз, в `src/lib.rs`, чтобы остальная программа никогда не писала `unsafe`:

```rust
//! A safe wrapper around a tiny C library.

mod ffi;

use std::ffi::{CStr, CString, c_int};

pub use ffi::Point;

/// Distance between two points.
pub fn distance(a: &Point, b: &Point) -> f64 {
    // SAFETY: both pointers come from references, so they are valid,
    // aligned and not null; the C function only reads them.
    unsafe { ffi::point_distance(a, b) }
}

/// Sum of all numbers in the slice.
pub fn sum(data: &[i32]) -> i64 {
    // SAFETY: `data.as_ptr()` is valid for `data.len()` reads of `i32`
    // (an empty slice gives a dangling but non-null pointer, and C reads nothing).
    unsafe { ffi::sum_array(data.as_ptr(), data.len()) as i64 }
}

/// Error returned by [`divide`].
#[derive(Debug, PartialEq)]
pub struct DivideByZero;

/// Divides `a` by `b`.
pub fn divide(a: i32, b: i32) -> Result<i32, DivideByZero> {
    let mut out: c_int = 0;
    // SAFETY: `&mut out` is a valid pointer to one `c_int`.
    let code = unsafe { ffi::safe_divide(a, b, &mut out) };
    if code == 0 { Ok(out) } else { Err(DivideByZero) }
}

/// A greeting allocated by C. It is given back to C when dropped.
pub struct Greeting {
    raw: *mut std::ffi::c_char,
}

impl Greeting {
    /// Returns `None` if `name` contains a NUL byte or C ran out of memory.
    pub fn new(name: &str) -> Option<Greeting> {
        let name = CString::new(name).ok()?;
        // SAFETY: `name` is a valid NUL-terminated string for the whole call.
        let raw = unsafe { ffi::make_greeting(name.as_ptr()) };
        if raw.is_null() { None } else { Some(Greeting { raw }) }
    }

    pub fn as_str(&self) -> &str {
        // SAFETY: `raw` is non-null (checked in `new`), points to a
        // NUL-terminated string, and lives until `drop`.
        unsafe { CStr::from_ptr(self.raw) }.to_str().unwrap_or("")
    }
}

impl Drop for Greeting {
    fn drop(&mut self) {
        // SAFETY: `raw` came from `make_greeting` and is freed exactly once.
        unsafe { ffi::free_greeting(self.raw) }
    }
}

/// Applies `callback` to every number, using C to do the loop.
pub fn map(data: &[i32], callback: extern "C" fn(i32) -> i32) -> Vec<i32> {
    let mut out = vec![0; data.len()];
    // SAFETY: `data` and `out` both have `data.len()` elements.
    unsafe { ffi::map_ints(data.as_ptr(), data.len(), callback, out.as_mut_ptr()) };
    out
}
```

**Что происходит, функция за функцией:**

1. `distance`: ссылки Rust `&Point` всегда действительны и выровнены, и они приводятся к `*const Point`. Функция C только читает. Одна строка `unsafe`.
2. `sum`: срез Rust — это ровно «указатель + длина», то же соглашение, что в C. Обёртка передаёт оба значения, поэтому C никогда не прочитает за концом.
3. `divide`: превращает шаблон C «код ошибки + выходной указатель» в `Result` Rust. Вызывающий код указателя не видит.
4. `Greeting`: строка выделена функцией `malloc` языка C. **Rust не должен её освобождать** (аллокатор Rust — другой!). Структура владеет указателем и вызывает `free_greeting` из C в `Drop`. Это **RAII**: память освобождается ровно один раз, даже при раннем `return` или панике. Конструктор проверяет NUL внутри имени и нулевой результат.
5. `map`: передаёт `extern "C" fn` из Rust как обратный вызов. `out` выделяет Rust с правильной длиной, поэтому C пишет внутри него.

Используйте это в `examples/demo.rs`:

```rust
use geo::{Greeting, Point, distance, divide, map, sum};

extern "C" fn square(x: i32) -> i32 {
    x * x
}

fn main() {
    let a = Point { x: 0.0, y: 0.0 };
    let b = Point { x: 3.0, y: 4.0 };
    println!("distance = {}", distance(&a, &b));

    println!("sum = {}", sum(&[10, 20, 30]));

    println!("10 / 3 = {:?}", divide(10, 3));
    println!("1 / 0 = {:?}", divide(1, 0));

    let greeting = Greeting::new("Bobur").unwrap();
    println!("{}", greeting.as_str());

    println!("squares = {:?}", map(&[1, 2, 3, 4], square));
}
```

Запустите `cargo run --example demo`:

```text
distance = 5
sum = 60
10 / 3 = Ok(3)
1 / 0 = Err(DivideByZero)
Hello, Bobur!
squares = [1, 4, 9, 16]
```

> 📏 **Соглашение:** держите объявления `unsafe extern` в одном приватном модуле (`ffi`), а остальной программе давайте только безопасные функции. Называйте модуль `ffi` или `sys`. Большие проекты делят это на два крейта: `foo-sys` (сырые объявления) и `foo` (безопасная обёртка).

> 🧪 **Тестирование:** тестируйте **обёртку**, а не библиотеку C. Проверяйте обычный случай, пустой ввод и каждую ветку ошибки. Добавьте тесты в конец `lib.rs`:

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn distance_of_3_4_5() {
        let a = Point { x: 0.0, y: 0.0 };
        let b = Point { x: 3.0, y: 4.0 };
        assert_eq!(distance(&a, &b), 5.0);
    }

    #[test]
    fn sum_of_numbers() {
        assert_eq!(sum(&[1, 2, 3, 4]), 10);
        assert_eq!(sum(&[]), 0);
    }

    #[test]
    fn divide_ok_and_error() {
        assert_eq!(divide(10, 2), Ok(5));
        assert_eq!(divide(1, 0), Err(DivideByZero));
    }

    #[test]
    fn greeting_round_trip() {
        let g = Greeting::new("Aziza").unwrap();
        assert_eq!(g.as_str(), "Hello, Aziza!");
    }

    #[test]
    fn greeting_rejects_nul() {
        assert!(Greeting::new("a\0b").is_none());
    }

    extern "C" fn double(x: i32) -> i32 {
        x * 2
    }

    #[test]
    fn map_doubles() {
        assert_eq!(map(&[1, 2, 3], double), vec![2, 4, 6]);
    }
}
```

`cargo test` напечатает `test result: ok. 6 passed; 0 failed`.

**Miri и FFI.** В Уроке 06.6 вы использовали Miri, чтобы находить неопределённое поведение в коде Rust. Miri — *интерпретатор Rust*; он не может запускать скомпилированный код на C. Вот что происходит:

```text
$ cargo +nightly miri test    (shortened output)
test tests::distance_of_3_4_5 ... error: unsupported operation: can't call foreign function `point_distance` on OS `macos`
  --> src/lib.rs:13:14
   |
13 |     unsafe { ffi::point_distance(a, b) }
   |              ^^^^^^^^^^^^^^^^^^^^^^^^^ unsupported operation occurred here
   |
   = help: this means the program tried to do something Miri does not support; it does not indicate a bug in the program
```

Итак, Miri проверяет только чисто «растовую» часть FFI-крейта. Для стороны C используют санитайзеры C и Valgrind; такие инструменты рассмотрены на следующих этапах.

## 5. Вызов Rust из C

У границы два направления. Чтобы C мог вызвать функцию Rust, сделайте её `extern "C"` и отключите искажение имён (Rust обычно переписывает имена функций, чтобы разные элементы не сталкивались; C нужно простое имя):

```rust
use std::ffi::{CStr, c_char, c_int};

/// Adds two numbers. Callable from C as `rustmath_add`.
#[unsafe(no_mangle)]
pub extern "C" fn rustmath_add(a: c_int, b: c_int) -> c_int {
    a.wrapping_add(b)
}

/// Returns the number of characters (not bytes) in a UTF-8 C string,
/// or -1 if the pointer is null or the text is not valid UTF-8.
///
/// # Safety
/// `text` must be null or point to a NUL-terminated string.
#[unsafe(no_mangle)]
pub unsafe extern "C" fn rustmath_char_count(text: *const c_char) -> c_int {
    if text.is_null() {
        return -1;
    }
    // SAFETY: the caller promised that `text` is a NUL-terminated string.
    let text = unsafe { CStr::from_ptr(text) };
    match text.to_str() {
        Ok(s) => s.chars().count() as c_int,
        Err(_) => -1,
    }
}
```

В `Cargo.toml` запросите статическую библиотеку:

```toml
[lib]
crate-type = ["staticlib"]
```

Напишите заголовок C вручную (инструмент **cbindgen** может создать его из вашего кода на Rust), `rustmath.h`:

```c
#ifndef RUSTMATH_H
#define RUSTMATH_H

int rustmath_add(int a, int b);
int rustmath_char_count(const char *text);

#endif
```

и программу на C, `main.c`:

```c
#include <stdio.h>
#include "rustmath.h"

int main(void) {
    printf("2 + 3 = %d\n", rustmath_add(2, 3));
    printf("characters in \"salom\": %d\n", rustmath_char_count("salom"));
    printf("null pointer: %d\n", rustmath_char_count(NULL));
    return 0;
}
```

Соберите и скомпонуйте:

```sh
cargo build --release
clang main.c target/release/librustmath.a -o demo
./demo
```

```text
2 + 3 = 5
characters in "salom": 5
null pointer: -1
```

**Что происходит:**

1. `#[unsafe(no_mangle)]` сохраняет имя символа `rustmath_add`. В редакции 2024 этот атрибут нужно писать внутри `unsafe(...)`, потому что неверное имя может сломать программу.
2. `crate-type = ["staticlib"]` заставляет `cargo build` создать `librustmath.a` — файл со всем кодом, включая нужные части стандартной библиотеки Rust.
3. `clang` компилирует `main.c` и компонует его с этим файлом. Программа на C вызывает Rust как любую функцию C.
4. `rustmath_char_count` — `unsafe extern "C"`, потому что её безопасность зависит от того, что передаст вызывающий код на C. Раздел `# Safety` в документации сообщает вызывающему правило. Она всё равно проверяет нулевой указатель — это дёшево.
5. `wrapping_add` избегает паники Rust при переполнении: **паника никогда не должна пересекать границу в C**.

> ⚠️ **Частая ошибка:** позволить панике вылететь из функции `extern "C"`. Раскрутка стека через кадры C в C не определена. Начиная с Rust 1.81, Rust вместо этого аварийно завершает процесс. Настоящий вывод (в сокращении) программы, в которой `extern "C" fn boom()` вызывает `panic!("oh no")`:
>
> ```text
> calling boom
>
> thread 'main' panicked at src/main.rs:2:5:
> oh no
>
> thread 'main' panicked at library/core/src/panicking.rs:225:5:
> panic in a function that cannot unwind
> thread caused non-unwinding panic. aborting.
> ```
>
> (Процесс завершается с кодом 134. Это сигнал `SIGABRT`, «abort»: программа просит операционную систему остановить её немедленно.) Внутри функций `extern "C"` возвращайте коды ошибок и избегайте `unwrap()`, `expect()` и индексации массивов, которые могут паниковать. Если паника возможна, перехватите её через `std::panic::catch_unwind` и превратите в код ошибки.

## 6. Правила границы

Когда вы пишете `unsafe` FFI-код, проверяйте этот список для **каждого** вызова. Каждый пункт — настоящий источник ошибок.

1. **Типы совпадают.** Объявление на Rust имеет те же типы аргументов, ту же раскладку структур (`#[repr(C)]`) и то же соглашение о вызовах (`extern "C"`), что и код на C.
2. **Указатели действительны.** Не нулевые (если C не принимает null), выровненные, указывают на достаточно инициализированной памяти и живы всё время вызова. Классическая ошибка — временное значение (см. Упражнение 3).
3. **Длины верны.** Указатель на массив всегда идёт вместе с длиной, и длина никогда не больше реальных данных. Неверная длина — это **переполнение буфера**.
4. **Строки заканчиваются NUL**, когда идут в C (`CString`), и не считаются UTF-8, когда приходят из C (`CStr::to_str` возвращает `Result`).
5. **Владение ясно.** Кто выделил, тот и освобождает — тем же аллокатором. Память из `malloc` языка C возвращается через `free` языка C. Память Rust C никогда не освобождает.
6. **Изменяемость и алиасинг.** Если C пишет через указатель, дайте ему `*mut`, полученный из `&mut`. Пока C держит указатель, не используйте те же данные из Rust.
7. **Паники не пересекают границу** ни в одном направлении.
8. **Потоки.** Если библиотека C не потокобезопасна, не вызывайте её из нескольких потоков одновременно (`Send` и `Sync` для ваших типов-обёрток сообщают это компилятору; сырые указатели по умолчанию не являются ни тем, ни другим, что вас защищает).
9. **Обратные вызовы.** Обратный вызов должен быть `extern "C"`. Если сторона C может сохранить указатель и вызвать его позже, он должен оставаться действительным всё это время. API на C часто добавляют аргумент `void *user_data`, чтобы можно было передать состояние в обратный вызов.
10. **Документируйте.** Каждый блок `unsafe` получает комментарий `// SAFETY:`, каждая `unsafe fn` — раздел `# Safety`.

> 🔒 **Безопасность:** FFI — это место, где заканчивается безопасность памяти Rust. Несовпадение длины или указатель, использованный после `free`, дают те же эксплуатируемые ошибки, что и в C. Держите небезопасную поверхность крошечной, тестируйте обёртку на неожиданных входах (пустой, огромный, NUL внутри) и отдавайте код на ревью.

## 7. `bindgen`: объявления, написанные машиной

Копировать объявления вручную легко с ошибкой. **bindgen** читает заголовок C и пишет их сам. В `Cargo.toml` добавьте `bindgen = "0.73"` в `[build-dependencies]` (ему нужен `libclang`, который идёт с инструментами командной строки Xcode), а в `build.rs`:

```rust
use std::env;
use std::path::PathBuf;

fn main() {
    cc::Build::new().file("c/geometry.c").compile("geometry");

    let bindings = bindgen::Builder::default()
        .header("c/geometry.h")
        .parse_callbacks(Box::new(bindgen::CargoCallbacks::new()))
        .generate()
        .expect("bindgen could not generate bindings");

    let out_dir = PathBuf::from(env::var("OUT_DIR").unwrap());
    bindings
        .write_to_file(out_dir.join("bindings.rs"))
        .expect("could not write bindings.rs");
}
```

а в коде на Rust подключите созданный файл:

```rust
include!(concat!(env!("OUT_DIR"), "/bindings.rs"));
```

`OUT_DIR` — папка, которую Cargo выдаёт скрипту сборки для его выходных файлов. Вот часть настоящего созданного файла:

```rust
#[repr(C)]
#[derive(Debug, Copy, Clone)]
pub struct Point {
    pub x: f64,
    pub y: f64,
}
const _: () = {
    ["Size of Point"][::std::mem::size_of::<Point>() - 16usize];
    ["Alignment of Point"][::std::mem::align_of::<Point>() - 8usize];
    ["Offset of field: Point::x"][::std::mem::offset_of!(Point, x) - 0usize];
    ["Offset of field: Point::y"][::std::mem::offset_of!(Point, y) - 8usize];
};
unsafe extern "C" {
    pub fn point_distance(a: *const Point, b: *const Point) -> f64;
}
unsafe extern "C" {
    pub fn map_ints(
        data: *const ::std::os::raw::c_int,
        len: usize,
        callback: ::std::option::Option<
            unsafe extern "C" fn(arg1: ::std::os::raw::c_int) -> ::std::os::raw::c_int,
        >,
        out: *mut ::std::os::raw::c_int,
    );
}
```

Обратите внимание на две вещи. Созданный блок `const _` проверяет размер и смещения структуры **во время компиляции**: сборка упадёт, если Rust и C расходятся. А обратный вызов — это `Option<unsafe extern "C" fn(...)>`: в C указатель на функцию может быть `NULL`, а `Option` — это способ Rust сказать «возможно, ничего». bindgen — более безопасный путь для больших заголовков; безопасную обёртку вы всё равно пишете вручную. Обратный инструмент, **cbindgen**, пишет заголовок C по вашим функциям `extern "C"` на Rust.

## Частые ошибки

**1. Вызов `unsafe`-функции без `unsafe`.**

```text
error[E0133]: call to unsafe function `strlen` is unsafe and requires unsafe block
 --> src/main.rs:9:13
  |
9 |     let n = strlen(text.as_ptr());
  |             ^^^^^^^^^^^^^^^^^^^^^ call to unsafe function
  |
  = note: consult the function's documentation for information on how to avoid undefined behavior
```

Исправление: оберните вызов в `unsafe { ... }` с комментарием `// SAFETY:` или сделайте безопасную функцию-обёртку.

**2. Блок `extern` в старом стиле (редакция 2024).**

```text
error: extern blocks must be unsafe
 --> src/main.rs:3:1
  |
3 |   extern "C" {
  |   ^
  |   |
  |  _help: needs `unsafe` before the extern keyword: `unsafe`
```

Исправление: пишите `unsafe extern "C" { ... }`.

**3. Передача `String` Rust там, где C ждёт указатель.**

```text
error[E0308]: mismatched types
 --> src/main.rs:9:29
  |
9 |     let n = unsafe { strlen(text) };
  |                      ------ ^^^^ expected `*const i8`, found `String`
  |
  = note: expected raw pointer `*const i8`
                  found struct `String`
```

Исправление: `let c = CString::new(text)?; strlen(c.as_ptr())`.

**4. `#[no_mangle]` в старом стиле (редакция 2024).**

```text
error: unsafe attribute used without unsafe
 --> src/main.rs:1:3
  |
1 | #[no_mangle]
  |   ^^^^^^^^^ usage of unsafe attribute
  |
help: wrap the attribute in `unsafe(...)`
```

Исправление: `#[unsafe(no_mangle)]`.

**5. Висячий указатель из временного значения.** Компилятор предупреждает вас (см. Упражнение 3).

**6. Неверное объявление.** Объявление `fn abs(x: i64) -> i64` для C-функции `int abs(int)`: ошибки нет, результаты неверны или происходят падения. Компилятор не может проверить C. Используйте типы из `std::ffi` и `bindgen` для больших заголовков.

## Попробуйте сами

1. Постройте пакет `geo`: создайте файлы из разделов 3 и 4 (`c/geometry.h`, `c/geometry.c`, `build.rs`, `src/ffi.rs`, `src/lib.rs`, `examples/demo.rs`) и добавьте `cc` в `[build-dependencies]`.
2. Выполните `cargo run --example demo`. Сравните с выводом в разделе 4.
3. Выполните `cargo test`. Все шесть тестов из раздела 4 должны пройти.
4. Добавьте тест `distance_of_6_8_10`, который проверяет, что расстояние между `(0, 0)` и `(6, 8)` равно `10.0`. Запустите `cargo test` снова: проходят семь тестов.
5. Сделайте коммит в Git: `git add code/p06-ffi && git commit -m "feat(p06-ffi): safe wrapper around a small C library"`. Добавьте `target/` в `.gitignore`, если его там нет.

## Упражнения

### Упражнение 1 — `atoi` из C (легко)

Объявите функцию C `int atoi(const char *)` и напишите `fn parse_with_c(text: &str) -> Option<i32>`, которая преобразует текст в число с её помощью. Она должна вернуть `None`, если в тексте есть NUL-байт. Выведите результат для `"1234"`, `"42abc"`, `"hello"` и `"a\0b"`.

<details>
<summary>Подсказка</summary>

Создайте `CString` через `CString::new(text).ok()?`. Передайте `text.as_ptr()` внутри блока `unsafe`. `atoi` возвращает 0 для текста, который не является числом.

</details>

<details>
<summary>Ответ</summary>

```rust
use std::ffi::{CString, c_char, c_int};

unsafe extern "C" {
    fn atoi(s: *const c_char) -> c_int;
}

fn parse_with_c(text: &str) -> Option<i32> {
    let text = CString::new(text).ok()?;
    // SAFETY: `text` is a valid NUL-terminated string that outlives the call.
    Some(unsafe { atoi(text.as_ptr()) })
}

fn main() {
    println!("{:?}", parse_with_c("1234"));
    println!("{:?}", parse_with_c("42abc"));
    println!("{:?}", parse_with_c("hello"));
    println!("{:?}", parse_with_c("a\0b"));
}
```

```text
Some(1234)
Some(42)
Some(0)
None
```

Почему это работает: `atoi` останавливается на первом символе, который не цифра, и возвращает 0, если цифр нет вообще (поэтому она не отличает «0» от ошибки; `str::parse` в Rust — лучший инструмент, а здесь только практика). `CString::new` отвергает текст с NUL внутри.

</details>

### Упражнение 2 — `count_positive` с тестами (средне)

Добавьте функцию C `size_t count_positive(const int *data, size_t len)` в `geometry.c` и `geometry.h`. Объявите её в `ffi.rs`, напишите безопасную обёртку `pub fn count_positive(data: &[i32]) -> usize` и тесты. Также напишите тест, который проверяет, что у `Point` раскладка как в C: размер 16, выравнивание 8, `x` по смещению 0, `y` по смещению 8.

<details>
<summary>Подсказка</summary>

Обёртка почти такая же, как `sum`. Для теста раскладки используйте `std::mem::size_of`, `std::mem::align_of` и `std::mem::offset_of!(Point, y)`. Не забудьте добавить новую функцию C и в заголовок.

</details>

<details>
<summary>Ответ</summary>

Добавьте в `geometry.h` (перед `#endif`):

```c
/* Counts how many of the `len` integers are greater than zero. */
size_t count_positive(const int *data, size_t len);
```

Добавьте в `geometry.c`:

```c
size_t count_positive(const int *data, size_t len) {
    size_t count = 0;
    for (size_t i = 0; i < len; i++) {
        if (data[i] > 0) {
            count++;
        }
    }
    return count;
}
```

В `ffi.rs`, внутри блока `extern`:

```rust
    pub fn count_positive(data: *const c_int, len: usize) -> usize;
```

В `lib.rs`:

```rust
/// How many numbers in the slice are greater than zero.
pub fn count_positive(data: &[i32]) -> usize {
    // SAFETY: `data.as_ptr()` is valid for `data.len()` reads of `i32`.
    unsafe { ffi::count_positive(data.as_ptr(), data.len()) }
}
```

Тесты:

```rust
    #[test]
    fn counts_positive_numbers() {
        assert_eq!(count_positive(&[3, -1, 0, 7, -5, 2]), 3);
        assert_eq!(count_positive(&[]), 0);
    }

    #[test]
    fn point_has_the_c_layout() {
        assert_eq!(std::mem::size_of::<Point>(), 16);
        assert_eq!(std::mem::align_of::<Point>(), 8);
        assert_eq!(std::mem::offset_of!(Point, x), 0);
        assert_eq!(std::mem::offset_of!(Point, y), 8);
    }
```

`cargo test` покажет `test result: ok. 8 passed`.

Почему это работает: в скрипте сборки есть `rerun-if-changed` для обоих файлов C, поэтому Cargo перекомпилирует их. `#[repr(C)]` гарантирует раскладку, которую проверяет тест.

</details>

### Упражнение 3 — Висячий указатель (средне)

Эта программа компилируется с предупреждением. Прочитайте её, объясните, что не так, и исправьте.

```rust
use std::ffi::{CString, c_char};

unsafe extern "C" {
    fn strlen(s: *const c_char) -> usize;
}

fn main() {
    let ptr = CString::new("systems").unwrap().as_ptr();
    let n = unsafe { strlen(ptr) };
    println!("{n}");
}
```

<details>
<summary>Подсказка</summary>

Когда уничтожается `CString`, созданный через `CString::new(...).unwrap()`? Посмотрите на конец инструкции `let ptr = ...;`.

</details>

<details>
<summary>Ответ</summary>

Компилятор говорит:

```text
warning: this creates a dangling pointer because temporary `CString` is dropped at end of statement
 --> src/main.rs:8:48
  |
8 |     let ptr = CString::new("systems").unwrap().as_ptr();
  |               -------------------------------- ^^^^^^ pointer created here
  |               |
  |               this `CString` is dropped at end of statement
  |
  = help: bind the `CString` to a variable such that it outlives the pointer returned by `as_ptr`
```

`CString` — временное значение. Оно освобождается на `;`, поэтому `ptr` указывает на освобождённую память. Тогда `strlen` читает память, которая нам больше не принадлежит: неопределённое поведение (в одном запуске напечаталось `1`, но возможен любой результат или падение). Исправление: храните `CString` в переменной, которая живёт, пока используется указатель.

```rust
use std::ffi::{CString, c_char};

unsafe extern "C" {
    fn strlen(s: *const c_char) -> usize;
}

fn main() {
    let text = CString::new("systems").unwrap();
    // SAFETY: `text` lives until the end of `main`, so the pointer stays valid.
    let n = unsafe { strlen(text.as_ptr()) };
    println!("{n}");
}
```

```text
7
```

Почему это работает: `text` владеет байтами и уничтожается в конце `main`, после вызова. Указатель действителен лишь настолько, насколько жив владелец за ним.

</details>

### Упражнение 4 — Экспорт `rust_reverse` в C (сложно)

Создайте статическую библиотеку с функцией `ssize_t rust_reverse(const char *input, char *out, size_t cap)`. Она записывает символы `input` в обратном порядке в буфер `out` (как строку с NUL в конце) и возвращает число записанных байтов. Она возвращает `-1` для нулевого указателя, текста, не являющегося корректным UTF-8, или слишком маленького буфера. Вызовите её из программы на C с буфером в 16 байт, а затем с буфером в 4 байта, для текста `"salom"`.

<details>
<summary>Подсказка</summary>

Объявите её как `pub unsafe extern "C" fn` с `#[unsafe(no_mangle)]` и параметрами `*const c_char`, `*mut c_char`, `usize`; возвращайте `isize`. Постройте перевёрнутую `String` через `text.chars().rev().collect()`. Нужно `bytes.len() + 1` байт места (для NUL). Копируйте через `std::ptr::copy_nonoverlapping`, а последний `0` пишите через `*out.add(len) = 0`.

</details>

<details>
<summary>Ответ</summary>

`src/lib.rs` (в `Cargo.toml` есть `crate-type = ["staticlib"]`):

```rust
use std::ffi::{CStr, c_char};

/// Writes the reverse of `input` into `out` (as a NUL-terminated string).
/// Returns the number of bytes written (without the NUL), or -1 on error:
/// a null pointer, text that is not valid UTF-8, or a buffer that is too small.
///
/// # Safety
/// `input` must be null or a NUL-terminated string. `out` must be null or
/// valid for writes of `cap` bytes.
#[unsafe(no_mangle)]
pub unsafe extern "C" fn rust_reverse(input: *const c_char, out: *mut c_char, cap: usize) -> isize {
    if input.is_null() || out.is_null() {
        return -1;
    }
    // SAFETY: the caller promised that `input` is a NUL-terminated string.
    let Ok(text) = unsafe { CStr::from_ptr(input) }.to_str() else {
        return -1;
    };
    let reversed: String = text.chars().rev().collect();
    let bytes = reversed.as_bytes();
    if bytes.len() + 1 > cap {
        return -1;
    }
    // SAFETY: `out` is valid for `cap` bytes and `bytes.len() + 1 <= cap`.
    unsafe {
        std::ptr::copy_nonoverlapping(bytes.as_ptr(), out as *mut u8, bytes.len());
        *out.add(bytes.len()) = 0;
    }
    bytes.len() as isize
}
```

`main.c`:

```c
#include <stdio.h>
#include <sys/types.h>

ssize_t rust_reverse(const char *input, char *out, size_t cap);

int main(void) {
    char buffer[16];
    ssize_t n = rust_reverse("salom", buffer, sizeof buffer);
    printf("%zd: %s\n", n, buffer);

    char tiny[4];
    n = rust_reverse("salom", tiny, sizeof tiny);
    printf("too small: %zd\n", n);
    return 0;
}
```

```sh
cargo build --release
clang main.c target/release/librevlib.a -o demo
./demo
```

```text
5: molas
too small: -1
```

(Статическая библиотека называется по имени пакета; здесь пакет называется `revlib`.)

Почему это работает: функция никогда не пишет больше `cap` байт. Она проверяет наличие места **до** записи — это правило и предотвращает переполнение буфера. Она использует `chars().rev()`, чтобы многобайтные буквы UTF-8 остались целыми.

</details>

### Упражнение 5 — Свой `Greeting` для длинных имён (сложно)

Функция C `make_greeting` выделяет `strlen(name) + 9` байт. Напишите тест для `Greeting::new` с именем из 10 000 символов и проверьте, что у `as_str()` длина 10 008. Затем ответьте: что случилось бы, если бы `Greeting` *не* реализовывал `Drop`? Напишите короткий цикл в духе теста, который привёл бы к утечке памяти.

<details>
<summary>Подсказка</summary>

`"x".repeat(10_000)` создаёт длинное имя. «Hello, » — это 7 символов, а «!» — 1. Цикл из `Greeting::new("x").unwrap()` с вызовом `std::mem::forget(g)` даёт такую же утечку, как отсутствие `Drop`.

</details>

<details>
<summary>Ответ</summary>

```rust
    #[test]
    fn long_name_works() {
        let name = "x".repeat(10_000);
        let g = Greeting::new(&name).unwrap();
        assert_eq!(g.as_str().len(), 10_008);
    }
```

Без `Drop` память C из `malloc` никогда бы не освобождалась. Такой цикл терял бы около 10 байт на итерацию, бесконечно:

```rust
for _ in 0..1_000_000 {
    let g = Greeting::new("x").unwrap();
    std::mem::forget(g); // skips Drop: same as having no Drop at all
}
```

Почему это работает: `10_000 + 7 + 1 = 10_008`. Rust не управляет памятью, пришедшей из C; вернуть её может только ваша реализация `Drop`, поэтому тип-обёртка и владеет указателем (RAII).

</details>

## Вопросы для самопроверки

1. Что делает `unsafe extern "C" { fn f(); }`?

<details>
<summary>Ответ</summary>

Объявляет, что функция `f` существует вне Rust и использует соглашение о вызовах C. Кода в блоке нет. `unsafe` означает, что вы обещаете: объявления соответствуют настоящим функциям.

</details>

2. Почему вызовы функций C по умолчанию `unsafe`?

<details>
<summary>Ответ</summary>

Компилятор Rust не видит код C и не может проверить указатели, длины, владение и типы. Правила должен гарантировать программист. Функцию можно пометить `safe fn` в объявлении, только если она не может вызвать неопределённое поведение.

</details>

3. Чем отличаются `CString` и `CStr`?

<details>
<summary>Ответ</summary>

`CString` — владеющая строка C, созданная в Rust (чтобы передать в C; заканчивается NUL, внутри NUL быть не может). `CStr` — заимствованный вид уже существующей строки C (чтобы прочитать указатель, который дал C).

</details>

4. Почему `Greeting` вызывает `free_greeting`, а не позволяет Rust освободить указатель?

<details>
<summary>Ответ</summary>

Память выделена функцией `malloc` языка C. Её нужно освобождать парной функцией C. Аллокатор Rust — другой, и освобождение не тем аллокатором — неопределённое поведение.

</details>

5. Что делает `#[repr(C)]`?

<details>
<summary>Ответ</summary>

Заставляет структуру использовать раскладку памяти C (порядок полей и выравнивание), чтобы Rust и C согласовывали размеры и смещения.

</details>

6. Что происходит, если паника вылетает из функции `extern "C"`?

<details>
<summary>Ответ</summary>

Начиная с Rust 1.81 процесс аварийно завершается (`panic in a function that cannot unwind`). Раскрутка стека через C не разрешена, поэтому такие функции не должны паниковать: возвращайте коды ошибок или перехватывайте паники через `catch_unwind`.

</details>

7. Почему Miri падает на тестах, которые вызывают библиотеку C?

<details>
<summary>Ответ</summary>

Miri интерпретирует код Rust и не может запускать чужой скомпилированный код. Он сообщает `can't call foreign function`.

</details>

## Итоги

- FFI позволяет Rust вызывать C, а C — Rust через C ABI. Граница `unsafe`: вы обещаете, что типы, указатели, длины и владение верны.
- Объявляйте функции C в `unsafe extern "C" { ... }`; используйте типы из `std::ffi` (`c_int`, `c_char`); помечайте структуры `#[repr(C)]`.
- Используйте `CString`, чтобы передать текст в C, и `CStr`, чтобы прочитать текст из C. Обрабатывайте NUL внутри и некорректный UTF-8.
- Компилируйте собственный код на C через `build.rs` и крейт `cc`; `bindgen` пишет объявления по заголовкам.
- Прячьте небезопасный код в одном небольшом модуле и отдавайте безопасный API: `Result` вместо кодов ошибок, RAII (`Drop`) для выделений в C.
- Экспортируйте Rust через `#[unsafe(no_mangle)] pub extern "C" fn`, собирайте `staticlib` и никогда не позволяйте панике пересекать границу.
- Документируйте каждый блок `unsafe` комментарием `// SAFETY:` и тестируйте безопасную обёртку, включая ошибки и пустой ввод.

## Источники и что почитать

- [The Rustonomicon — Foreign Function Interface](https://doc.rust-lang.org/nomicon/ffi.html) — длинное официальное руководство по FFI, обратным вызовам и паникам.
- [std::ffi module](https://doc.rust-lang.org/std/ffi/index.html) — `CString`, `CStr` и псевдонимы типов вроде `c_int`.
- [Edition Guide — Unsafe extern blocks](https://doc.rust-lang.org/edition-guide/rust-2024/unsafe-extern.html) — почему редакция 2024 требует `unsafe extern` и как работает `safe fn`.
- [Edition Guide — Unsafe attributes](https://doc.rust-lang.org/edition-guide/rust-2024/unsafe-attributes.html) — почему `#[unsafe(no_mangle)]`.
- [Cargo Book — Build scripts](https://doc.rust-lang.org/cargo/reference/build-scripts.html) — что умеет `build.rs` и инструкции `cargo::`.
- [cc crate](https://docs.rs/cc/latest/cc/) — компиляция кода на C из скрипта сборки.
- [The bindgen User Guide](https://rust-lang.github.io/rust-bindgen/) — создание объявлений Rust по заголовкам C.

---

[🏠 Начало](../START-HERE.md) · [← 06.9 Введение в async/await](09-async-await-introduction.md) · [📋 Этап 06](README.md) · [06.11 Проект: многопоточный веб-сервер →](11-project-multithreaded-web-server.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
