<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 22.23 Трек компиляторов: интерпретатор обхода дерева](23-compilers-track-an-interpreter.md) · [📋 Этап 22](README.md) · [22.25 Трек компиляторов: проект и следующие шаги →](25-compilers-track-project-and-next-steps.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/22-specializations/24-compilers-track-webassembly.md) <!-- nav -->

# Урок 22.24 — Трек компиляторов: WebAssembly и Rust

> **Этап 22 — Специализации** · Урок 24 из 30 · ⏱ около 3 часов (2 занятия)

## Что вы узнаете

- Что такое WebAssembly: переносимый формат инструкций для стековой машины с модулями, импортами, экспортами и песочницей.
- Как читать и писать текстовый формат WebAssembly (WAT) и запускать модуль в Wasmtime.
- Как компилировать Rust в WebAssembly: программу для WASI и библиотеку, которая экспортирует функции.
- Как написать генератор кода, который компилирует выражения Pebble в WebAssembly.
- Как запускать WebAssembly из собственной программы на Rust с крейтом `wasmtime` и останавливать бесконечные циклы топливом.

## Перед началом

- Вы прошли [Урок 22.23 — Трек компиляторов: интерпретатор обхода дерева](23-compilers-track-an-interpreter.md).
- Вы знаете, что такое машинный код и стек, из [Урока 12.6 — Как процессор выполняет инструкции](../12-computer-systems/06-how-the-cpu-runs-instructions.md)
  и [Урока 12.8 — От Rust до машинного кода](../12-computer-systems/08-from-rust-to-machine-code.md).
- Установите две целевые платформы Rust и инструмент командной строки Wasmtime (его сборка занимает около
  десяти минут):

```bash
rustup target add wasm32-wasip1 wasm32-unknown-unknown
cargo install --locked wasmtime-cli
wasmtime --version
```

```text
wasmtime 50.0.2
```

В этом уроке рядом с `code/p22-pebble` появятся три новых маленьких пакета: `code/p22-wasm-hello`,
`code/p22-wasm-math` и `code/p22-wasm-host`. Каждый вы создадите, когда урок до него дойдёт.

## Новые слова

| Слово | Значение |
|---|---|
| **WebAssembly** (Wasm) | Переносимый двоичный формат инструкций. Компилятор один раз создаёт файл `.wasm`, и любая среда выполнения WebAssembly (браузер, Wasmtime) запускает его на любом CPU (Central Processing Unit, центральный процессор) и в любой операционной системе. |
| **модуль** (module) | Один файл WebAssembly: функции, память и списки **импортов** (что ему нужно от хоста) и **экспортов** (что он предлагает). |
| **хост** (host) | Программа, которая загружает и запускает модуль WebAssembly, например инструмент `wasmtime` или ваша собственная программа на Rust. |
| **стековая машина** (stack machine) | Машина, инструкции которой берут операнды со стека и кладут на него результат, а не называют регистры. |
| **WAT** (WebAssembly Text format, текстовый формат WebAssembly) | Удобочитаемая текстовая форма модуля с инструкциями вроде `i64.add`. |
| **линейная память** (linear memory) | Память модуля: один массив байтов, который модуль может читать и писать, и ничего за его пределами. |
| **песочница** (sandbox) | Среда, в которой код может делать только то, что явно разрешил хост. |
| **ловушка** (trap) | WebAssembly немедленно останавливает выполняемый код из-за ошибки, например деления на ноль; хост получает ошибку. |
| **WASI** (WebAssembly System Interface, системный интерфейс WebAssembly) | Стандартный набор импортов для системных вещей — файлов, часов, стандартного вывода, — чтобы программы могли работать вне браузера. |
| **Wasmtime** | Среда выполнения WebAssembly, написанная на Rust; доступна как инструмент командной строки и как крейт `wasmtime`. |
| **`cdylib`** | Тип крейта для библиотеки, которую могут загружать другие языки и хосты; для WebAssembly он даёт модуль `.wasm` с экспортами. |
| **генератор кода** (code generator) | Часть компилятора, которая превращает синтаксическое дерево в инструкции целевой машины. |
| **топливо** (fuel) | Бюджет работы: Wasmtime вычитает топливо по мере выполнения кода и выдаёт ловушку, когда оно кончается. |

## 1. Что такое WebAssembly

Машинный код привязан к одному виду процессора: программа, скомпилированная для чипа arm64 вашего Mac,
не запускается на сервере x86-64 (Урок 12.7). **WebAssembly** — набор инструкций для машины, которой
не существует в виде чипа. Компиляторы создают его, а **среда выполнения** (runtime) переводит его в
настоящий машинный код прямо перед запуском. Один и тот же файл `.wasm` работает в любом браузере, на
серверах, в граничных сетях и внутри других программ.

Три свойства делают его интересным для системных инженеров:

- **Переносимость:** один файл для любого процессора и любой операционной системы.
- **Скорость:** среда выполнения компилирует его в машинный код, поэтому он работает почти со скоростью
  нативного кода.
- **Песочница:** модуль может вычислять и пользоваться своей линейной памятью. Всё остальное (файлы,
  сеть, печать, даже часы) нужно **импортировать** у хоста. Модуль, который ничего не импортирует,
  может только вычислять. Поэтому WebAssembly используют для плагинов: хост безопасно выполняет
  недоверенный код.

WebAssembly — **стековая машина**. Инструкции берут операнды со стека и кладут на него результат:

```text
 instruction        stack after it (top on the right)
 local.get $a       [2]
 local.get $b       [2, 3]
 i64.add            [5]          pops 3 and 2, pushes 2 + 3
```

Вот целый модуль в текстовом формате (WAT). Сохраните его как `add.wat` где угодно, например в
`code/p22-wasm-hello/`:

```text
(module
  ;; add(a, b) = a + b, for 64-bit integers
  (func (export "add") (param $a i64) (param $b i64) (result i64)
    local.get $a
    local.get $b
    i64.add))
```

**Что происходит:**

1. `(module ...)` содержит всё. Строки, начинающиеся с `;;`, — комментарии.
2. `(func (export "add") ...)` определяет функцию и экспортирует её под именем `add`. Она принимает два
   параметра типа `i64` (64-битное целое) и возвращает один.
3. Тело — три инструкции. Значение, оставшееся на стеке в конце, — это результат.

Запустите его через `wasmtime --invoke`, который вызывает одну экспортированную функцию с аргументами:

```bash
wasmtime --invoke add add.wat 2 3
wasmtime --invoke add add.wat 9223372036854775807 1
```

```text
warning: using `--invoke` with a function that takes arguments is experimental and may break in the future
warning: using `--invoke` with a function that returns values is experimental and may break in the future
5
warning: using `--invoke` with a function that takes arguments is experimental and may break in the future
warning: using `--invoke` with a function that returns values is experimental and may break in the future
-9223372036854775808
```

Предупреждения идут в стандартный поток ошибок; добавьте `2>/dev/null`, чтобы скрыть их (дальше мы так
и делаем). Обратите внимание на второй результат: `i64.add` при переполнении **переносит значение по
модулю**, а не сообщает об ошибке. Wasmtime принимает текстовый формат напрямую и сам переводит его в
двоичный.

## 2. Программы на Rust для WASI

В этом уроке Rust может компилировать для двух целевых платформ WebAssembly:

| Целевая платформа | Для чего | Есть ли `std`? |
|---|---|---|
| `wasm32-wasip1` | целые программы с `main`, которые запускает среда выполнения WASI, например Wasmtime | да: файлы, `println!`, аргументы — через импорты WASI |
| `wasm32-unknown-unknown` | библиотеки для любого хоста; никакой операционной системы | те части, которым не нужна операционная система |

(`wasm32` означает 32-битные адреса в линейной памяти; целые числа при этом могут быть 64-битными.)
Создайте первый пакет и замените `src/main.rs`:

```bash
cd ~/projects/rust
cargo new code/p22-wasm-hello
cd code/p22-wasm-hello
echo "remember the milk" > notes.txt
```

```rust
use std::fs;

fn main() {
    println!("hello from WebAssembly");
    let args: Vec<String> = std::env::args().collect();
    println!("arguments: {args:?}");
    match fs::read_to_string("notes.txt") {
        Ok(text) => println!("notes.txt has {} bytes", text.len()),
        Err(e) => println!("cannot read notes.txt: {e}"),
    }
}
```

Соберите его для WASI и запустите дважды: один раз как есть, другой — передав ему текущую папку:

```bash
cargo build --target wasm32-wasip1
wasmtime target/wasm32-wasip1/debug/p22-wasm-hello.wasm one two
wasmtime --dir . target/wasm32-wasip1/debug/p22-wasm-hello.wasm
```

```text
hello from WebAssembly
arguments: ["p22-wasm-hello.wasm", "one", "two"]
cannot read notes.txt: No such file or directory (os error 44)
hello from WebAssembly
arguments: ["p22-wasm-hello.wasm"]
notes.txt has 18 bytes
```

**Что происходит:**

1. `--target wasm32-wasip1` заставляет Cargo создать `target/wasm32-wasip1/debug/p22-wasm-hello.wasm`
   вместо нативной программы. Его первые четыре байта — `\0asm`, «магическое число» WebAssembly.
2. `println!` и `std::env::args` работают: стандартная библиотека Rust для этой платформы вызывает
   импорты WASI, а Wasmtime их предоставляет.
3. Первый запуск не может прочитать `notes.txt`, хотя файл лежит рядом: у модуля **нет доступа ни к
   одной папке**, пока хост его не выдаст. (Ошибка 44 — код WASI для «нет такого файла».)
4. `--dir .` выдаёт доступ к текущей папке, и та же программа читает файл. Доступ выдаётся по папкам —
   тем, кто запускает программу, а не самой программой.

> ⚡ **Производительность:** отладочная сборка весит около 2,5 МБ; `cargo build --release --target wasm32-wasip1`
> даёт около 100 КБ. Размер важен, когда модули скачивают или запускают тысячи раз. Релизные сборки,
> `opt-level = "s"` и `strip = true` в `Cargo.toml` (Урок 19.8) делают модули меньше.

## 3. Библиотека на Rust как модуль WebAssembly

Для библиотеки, которую вызывает хост, используйте тип крейта `cdylib` и `wasm32-unknown-unknown`.
Создайте пакет:

```bash
cd ~/projects/rust
cargo new --lib code/p22-wasm-math
```

Добавьте это в `Cargo.toml`:

```toml
[lib]
crate-type = ["cdylib"]
```

и замените `src/lib.rs` на:

```rust
/// The greatest common divisor, exported to WebAssembly under the name `gcd`.
#[unsafe(no_mangle)]
pub extern "C" fn gcd(mut a: i64, mut b: i64) -> i64 {
    while b != 0 {
        (a, b) = (b, a % b);
    }
    a
}

/// Integer division; WebAssembly stops ("traps") when `b` is zero.
#[unsafe(no_mangle)]
pub extern "C" fn divide(a: i64, b: i64) -> i64 {
    a / b
}
```

**Что происходит:**

1. `extern "C"` даёт функциям простое стабильное соглашение о вызовах (Урок 06.10) — то, которое
   используют экспорты WebAssembly.
2. `#[unsafe(no_mangle)]` сохраняет в модуле точные имена `gcd` и `divide`; без него компилятор
   придумал бы более длинные уникальные имена. Атрибут помечен `unsafe`, потому что две функции с одним
   и тем же неизменённым именем в одной программе были бы серьёзной ошибкой, которую компилятор не
   всегда может обнаружить.
3. Цикл — алгоритм Евклида, тот же, что в `programs/loops.pb` из Урока 22.23.

Соберите и вызовите:

```bash
cargo build --release --target wasm32-unknown-unknown
wasmtime --invoke gcd target/wasm32-unknown-unknown/release/p22_wasm_math.wasm 1071 462 2>/dev/null
wasmtime --invoke divide target/wasm32-unknown-unknown/release/p22_wasm_math.wasm 7 0
```

```text
21
Error: failed to run main module `target/wasm32-unknown-unknown/release/p22_wasm_math.wasm`

Caused by:
    0: failed to invoke `divide`
    1: error while executing at wasm backtrace:
           ...
    2: wasm trap: wasm `unreachable` instruction executed
```

(Трассировка в середине сокращена.) Код на Rust запаниковал («attempt to divide by zero»). На этой
платформе паника не может раскрутить стек, поэтому она становится инструкцией `unreachable`, и
WebAssembly выдаёт **ловушку**: модуль немедленно останавливается, а хост получает ошибку. Сам Wasmtime
не упал. Релизный модуль весит около 20 КБ.

## 4. Генератор кода для Pebble

Теперь — компиляторная часть направления. Вместо того чтобы *выполнять* синтаксическое дерево, мы
*переводим* его в инструкции WebAssembly, а выполняет их Wasmtime. Поскольку WebAssembly — стековая
машина, перевод выражения прост: **сначала выдать код операндов, затем оператор**. Для `x * x + 1`:

```text
 tree                    instructions                stack
 (+ (* x x) 1)           local.get $x                [x]
                         local.get $x                [x, x]
                         i64.mul                     [x*x]
                         i64.const 1                 [x*x, 1]
                         i64.add                     [x*x+1]
```

Это обход дерева в **обратном порядке** (post-order): сначала дети, потом узел. Мы компилируем одно
выражение в модуль с одной экспортированной функцией `run`, параметры которой — переменные выражения.
Каждое значение Pebble становится `i64`; `true` и `false` становятся 1 и 0. Сохраните как `src/wasm.rs`
в `code/p22-pebble` и добавьте `pub mod wasm;` в `src/lib.rs`:

```rust
//! A code generator: compiles a Pebble expression to WebAssembly text (WAT).

use crate::ast::{BinOp, Expr, UnOp};

/// Compiles `expr` into a module that exports `run`. The names in `params`
/// become the parameters of `run`; every value is a 64-bit integer.
pub fn compile_expr(expr: &Expr, params: &[&str]) -> Result<String, String> {
    let mut body = String::new();
    emit(expr, params, &mut body)?;
    let params: String = params
        .iter()
        .map(|p| format!(" (param ${p} i64)"))
        .collect();
    Ok(format!(
        "(module\n  (func (export \"run\"){params} (result i64)\n{body}  )\n)\n"
    ))
}

/// Appends the instructions that leave the value of `expr` on the stack.
fn emit(expr: &Expr, params: &[&str], out: &mut String) -> Result<(), String> {
    match expr {
        Expr::Int(n) => push(out, &format!("i64.const {n}")),
        Expr::Bool(b) => push(out, &format!("i64.const {}", *b as i64)),
        Expr::Var(name, pos) => {
            if !params.contains(&name.as_str()) {
                return Err(format!(
                    "{}:{}: unknown variable `{name}`",
                    pos.line, pos.col
                ));
            }
            push(out, &format!("local.get ${name}"));
        }
        Expr::Unary(UnOp::Neg, e, _) => {
            push(out, "i64.const 0"); // -e is computed as 0 - e
            emit(e, params, out)?;
            push(out, "i64.sub");
        }
        Expr::Binary(op, l, r, _) => {
            emit(l, params, out)?;
            emit(r, params, out)?;
            let (instr, is_comparison) = match op {
                BinOp::Add => ("i64.add", false),
                BinOp::Sub => ("i64.sub", false),
                BinOp::Mul => ("i64.mul", false),
                BinOp::Div => ("i64.div_s", false),
                BinOp::Rem => ("i64.rem_s", false),
                BinOp::Eq => ("i64.eq", true),
                BinOp::Ne => ("i64.ne", true),
                BinOp::Lt => ("i64.lt_s", true),
                BinOp::Le => ("i64.le_s", true),
                BinOp::Gt => ("i64.gt_s", true),
                BinOp::Ge => ("i64.ge_s", true),
            };
            push(out, instr);
            if is_comparison {
                // A comparison leaves a 32-bit 0 or 1; widen it to i64.
                push(out, "i64.extend_i32_u");
            }
        }
    }
    Ok(())
}

/// Appends one indented instruction line.
fn push(out: &mut String, instr: &str) {
    out.push_str("    ");
    out.push_str(instr);
    out.push('\n');
}

#[cfg(test)]
mod tests {
    use super::*;
    use crate::ast::Stmt;
    use crate::parser::parse;

    fn expr(src: &str) -> Expr {
        match parse(&format!("{src};")).unwrap().remove(0) {
            Stmt::Expr(e) => e,
            other => panic!("not an expression: {other}"),
        }
    }

    #[test]
    fn operands_first_then_the_operator() {
        let wat = compile_expr(&expr("1 + 2 * x"), &["x"]).unwrap();
        let body: Vec<&str> = wat.lines().skip(2).map(str::trim).collect();
        assert_eq!(
            body,
            [
                "i64.const 1",
                "i64.const 2",
                "local.get $x",
                "i64.mul",
                "i64.add",
                ")",
                ")"
            ]
        );
    }

    #[test]
    fn unknown_variables_are_compile_errors() {
        let err = compile_expr(&expr("x + y"), &["x"]).unwrap_err();
        assert_eq!(err, "1:5: unknown variable `y`");
    }
}
```

**Что происходит:**

1. `compile_expr` сначала генерирует тело, затем оборачивает его в текст модуля с одним
   `(param $name i64)` на каждую переменную. Результат — текст WAT в `String`.
2. `emit` — обход в обратном порядке. Литералы кладут на стек константу, переменные кладут параметр
   через `local.get` (или выдают ошибку, если имя — не параметр: **ошибка компиляции**, найденная до
   того, как что-либо выполняется).
3. В WebAssembly нет инструкции «сменить знак» для целых чисел, поэтому `-e` компилируется как `0 - e`.
4. Каждый оператор Pebble соответствует одной инструкции. `div_s` и `lt_s` — **знаковые** версии (`_s`);
   в WebAssembly есть и беззнаковые (`_u`), потому что биты `i64` означают разные числа при двух
   прочтениях (Урок 12.2).
5. Сравнения дают 32-битное значение (`i32`) 0 или 1. `i64.extend_i32_u` расширяет его до `i64`, чтобы
   все значения на нашем стеке были одного типа. (WebAssembly проверяет типы при загрузке модуля;
   несовпадение — ошибка загрузки, а не падение.)
6. Тесты проверяют порядок инструкций и ошибку компиляции.

> ⚠️ **Частая ошибка:** если вы оставили операторы из упражнений Урока 22.22 (`**`, `&&`, `||`, `!`), два выражения
> `match` в `emit` их не покрывают, и сборка останавливается с ошибкой `E0004` («non-exhaustive patterns», не все
> варианты разобраны). Пока добавьте ветку `` Expr::Unary(UnOp::Not, ..) => return Err("`!` is not supported by the compiler yet".to_string()), ``
> и последней веткой в `match` по оператору —
> `` _ => return Err(format!("`{}` is not supported by the compiler yet", op.symbol())), ``.
> По-настоящему эти операторы компилирует проект Урока 22.25.

Маленькая программа превращает аргументы командной строки в модуль. Сохраните как
`examples/compile.rs`:

```rust
//! Usage: cargo run --example compile -- "<expression>" [parameter ...]
use p22_pebble::ast::Stmt;
use p22_pebble::parser::parse;
use p22_pebble::wasm::compile_expr;

fn main() {
    let args: Vec<String> = std::env::args().skip(1).collect();
    let Some((src, params)) = args.split_first() else {
        eprintln!("usage: compile <expression> [parameter ...]");
        std::process::exit(2);
    };
    let params: Vec<&str> = params.iter().map(String::as_str).collect();
    let result = parse(&format!("{src};")).and_then(|mut program| match program.remove(0) {
        Stmt::Expr(e) => compile_expr(&e, &params),
        _ => Err("expected a single expression".to_string()),
    });
    match result {
        Ok(wat) => print!("{wat}"),
        Err(message) => {
            eprintln!("error: {message}");
            std::process::exit(1);
        }
    }
}
```

Скомпилируйте два выражения и запустите их:

```bash
cargo run -q --example compile -- "x * x + 1" x > square.wat
cat square.wat
wasmtime --invoke run square.wat 7 2>/dev/null
cargo run -q --example compile -- "(a - b) * (a - b) < 10" a b > near.wat
wasmtime --invoke run near.wat 5 7 2>/dev/null
wasmtime --invoke run near.wat 5 70 2>/dev/null
```

```text
(module
  (func (export "run") (param $x i64) (result i64)
    local.get $x
    local.get $x
    i64.mul
    i64.const 1
    i64.add
  )
)
50
1
0
```

Вы написали компилятор: на входе исходный текст Pebble, на выходе WebAssembly, а настоящая среда
выполнения превращает его в машинный код. Сравните с интерпретатором из Урока 22.23, который обходит
дерево *при каждом* запуске; здесь дерево обходится один раз, во время компиляции.

Скомпилированный код ведёт себя не всегда так же, как интерпретатор, и это важно:

```bash
cargo run -q --example compile -- "a / b" a b > div.wat
wasmtime --invoke run div.wat 7 0 2>&1 | tail -1
```

```text
    2: wasm trap: integer divide by zero
```

Деление на ноль — ловушка (интерпретатор давал ошибку выполнения с позицией), а переполнение переносит
значение по модулю (интерпретатор сообщал об ошибке). Две реализации одного языка должны совпадать; в
проекте (Урок 22.25) вы добьётесь этого и проверите тестами.

> 🧪 **Тестирование:** генератор кода тестируют на двух уровнях: *форма* вывода (список инструкций, как
> в нашем модульном тесте) и *поведение* (запустить модуль и проверить результат). Второй вид ловит
> больше ошибок; в проекте и интерпретатор, и скомпилированный модуль запускаются на одних и тех же
> программах, а результаты сравниваются (дифференциальное тестирование).

## 5. Запуск WebAssembly из Rust

Команда `wasmtime` сама является программой на Rust, построенной на крейте `wasmtime`, и ваши программы
могут использовать тот же крейт, чтобы загружать и запускать модули: система плагинов, движок правил
или тестовый стенд вашего компилятора. Создайте пакет:

```bash
cd ~/projects/rust
cargo new code/p22-wasm-host
cd code/p22-wasm-host
cargo add wasmtime@50
cargo add anyhow@1
```

Замените `src/main.rs`:

```rust
use anyhow::Result;
use wasmtime::{Config, Engine, Instance, Module, Store};

const WAT: &str = r#"
(module
  (func (export "add") (param $a i64) (param $b i64) (result i64)
    local.get $a
    local.get $b
    i64.add)
  (func (export "spin") (loop $again br $again)))
"#;

fn main() -> Result<()> {
    let mut config = Config::new();
    config.consume_fuel(true); // count work, so endless loops can be stopped
    let engine = Engine::new(&config)?;
    let module = Module::new(&engine, WAT)?; // compiles the module to machine code

    let mut store = Store::new(&engine, ());
    store.set_fuel(1_000_000)?;
    let instance = Instance::new(&mut store, &module, &[])?;

    let add = instance.get_typed_func::<(i64, i64), i64>(&mut store, "add")?;
    println!("add(40, 2) = {}", add.call(&mut store, (40, 2))?);

    let spin = instance.get_typed_func::<(), ()>(&mut store, "spin")?;
    match spin.call(&mut store, ()) {
        Ok(()) => println!("spin returned"),
        Err(e) => println!("spin stopped: {}", e.root_cause()),
    }
    println!("fuel left: {}", store.get_fuel()?);
    Ok(())
}
```

`cargo run` (первая сборка компилирует Wasmtime и занимает несколько минут):

```text
add(40, 2) = 42
spin stopped: wasm trap: all fuel consumed by WebAssembly
fuel left: 0
```

**Что происходит:**

1. `Engine` хранит настройки компилятора; `consume_fuel(true)` заставляет сгенерированный машинный код
   считать свою работу. `Module` — скомпилированный код; `Module::new` принимает текст WAT (и двоичный
   `.wasm`).
2. `Store` хранит состояние работающих экземпляров (их память, их топливо); `set_fuel` выдаёт ему бюджет.
   `Instance::new` создаёт работающий экземпляр; пустой срез `&[]` означает «без импортов», поэтому этот
   модуль может только вычислять.
3. `get_typed_func::<(i64, i64), i64>` находит экспорт и один раз проверяет его тип, поэтому потом
   `call` дешёвый и типобезопасный.
4. `spin` — бесконечный цикл (`br $again` прыгает обратно в начало `loop`). Без топлива вызов никогда
   бы не вернулся; с топливом Wasmtime выдаёт ловушку, когда бюджет израсходован, и хост продолжает
   работу. Это ограничение числа шагов из упражнения 3 Урока 22.23, сделанное средой выполнения.

> 🔒 **Безопасность:** WebAssembly даёт изоляцию памяти (модуль не может читать память хоста) и доступ
> на основе возможностей (ничего не достижимо, пока не импортировано). Но сам по себе он не ограничивает
> время и память: задавайте топливо (или прерывание по эпохам) и ограничения памяти, когда запускаете
> код, которому не доверяете, и обновляйте Wasmtime, потому что среда выполнения — часть вашей
> поверхности атаки.

## Частые ошибки

**Сборка не для той платформы.** `cargo build` без `--target` собирает нативную программу;
`wasmtime target/debug/p22-wasm-hello` тогда не работает, потому что этот файл — не модуль WebAssembly.
Файл `.wasm` лежит в `target/<target>/...`.

**Забыть `#[unsafe(no_mangle)]` или `crate-type = ["cdylib"]`.** Без них в модуле нет экспорта `gcd`, и
`wasmtime --invoke gcd` заканчивается сообщением ``no func export named `gcd` found``. Проверить, что
экспортирует модуль, можно, прочитав его текст: программа `wasm-tools`
(`cargo install --locked wasm-tools`, затем `wasm-tools print module.wasm`) печатает любой `.wasm` как
WAT.

**Тестировать панику внутри функций `extern "C"`.** Модульный тест вроде
`#[should_panic] fn dividing_by_zero_panics() { divide(7, 0); }` в `p22-wasm-math` не проходит, а
прерывает весь запуск тестов на вашем Mac:

```text
thread 'tests::dividing_by_zero_panics' (6484616) panicked at src/lib.rs:13:5:
attempt to divide by zero
...
panic in a function that cannot unwind
...
thread caused non-unwinding panic. aborting.
```

Функция `extern "C"` обещает не раскручивать стек, поэтому паника внутри неё прерывает процесс.
Экспортируемые функции вообще не должны паниковать: проверяйте входные данные и возвращайте значение,
означающее «ошибку», или держите логику в обычной функции Rust, тестируйте её, а функция `extern "C"`
пусть только вызывает её.

**Считать, что скомпилированный код ведёт себя как интерпретатор.** В WebAssembly переполнение
переносит значение по модулю, а деление на ноль даёт ловушку. Решите, что это значит в *вашем языке*, а
затем заставьте обе реализации делать именно это.

## Попробуйте сами

1. Выполните `xxd target/wasm32-wasip1/release/p22-wasm-hello.wasm | head -1` и найдите магическое
   число `0061 736d` (`\0asm`) и версию `0100 0000` (1).
2. Скомпилируйте `-x` через `examples/compile.rs` и прочитайте WAT: вы увидите `i64.const 0`,
   `local.get $x`, `i64.sub`.
3. В `p22-wasm-host` замените `set_fuel(1_000_000)` на `set_fuel(3)`. `add` по-прежнему работает: его
   трём инструкциям нужно всего несколько единиц топлива (большинство простых инструкций стоят одну
   единицу).

## Упражнения

### Упражнение 1 — WAT вручную (легко)

Напишите `rect.wat` с двумя экспортированными функциями: `area(w, h) = w * h` и
`perimeter(w, h) = 2 * (w + h)`. Проверьте: `area` для 6 и 7 равна 42, `perimeter` — 26.

<details>
<summary>Подсказка</summary>

Для `perimeter` сначала положите `2`, затем вычислите `w + h`, затем умножьте: перед `i64.mul` стек —
`[2, w+h]`.

</details>

<details>
<summary>Ответ</summary>

```text
(module
  ;; area(w, h) = w * h
  (func (export "area") (param $w i64) (param $h i64) (result i64)
    local.get $w
    local.get $h
    i64.mul)
  ;; perimeter(w, h) = 2 * (w + h)
  (func (export "perimeter") (param $w i64) (param $h i64) (result i64)
    i64.const 2
    local.get $w
    local.get $h
    i64.add
    i64.mul))
```

```bash
wasmtime --invoke area rect.wat 6 7 2>/dev/null
wasmtime --invoke perimeter rect.wat 6 7 2>/dev/null
```

```text
42
26
```

Почему это работает: тело каждой функции оставляет на стеке ровно одно `i64`, что совпадает с
`(result i64)`. Если забыть инструкцию, Wasmtime откажется загружать модуль с ошибкой типов, ещё до
того, как что-либо выполнится.

</details>

### Упражнение 2 — Протестируйте библиотеку на Mac (средне)

У `cdylib` всё равно могут быть модульные тесты, которые выполняются нативно. Добавьте в
`p22-wasm-math` тест, который проверяет `gcd(1071, 462) == 21`, `gcd(17, 5) == 1` и `gcd(12, 0) == 12`,
и запустите `cargo test` (без `--target`). Почему `divide(7, 0)` нельзя протестировать так же?

<details>
<summary>Подсказка</summary>

`cargo test` собирает для вашей собственной машины, где функции `extern "C"` — обычные функции, которые
можно вызывать. Перечитайте раздел «Частые ошибки» о панике в функциях `extern "C"`.

</details>

<details>
<summary>Ответ</summary>

В конце `src/lib.rs`:

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn gcd_of_known_pairs() {
        assert_eq!(gcd(1071, 462), 21);
        assert_eq!(gcd(17, 5), 1);
        assert_eq!(gcd(12, 0), 12);
    }
}
```

```text
test tests::gcd_of_known_pairs ... ok
```

Почему это работает: тест вызывает `gcd` напрямую как функцию Rust. `divide(7, 0)` паникует внутри
функции `extern "C"`, которая не может раскрутить стек, поэтому тестовый процесс прерывается, вместо
того чтобы сообщить об упавшем или прошедшем тесте. Перенесите такую логику в обычную функцию
(например, возвращающую `Option<i64>`) и тестируйте её.

</details>

### Упражнение 3 — Хост, который запускает скомпилированный Pebble (сложно)

Напишите `src/bin/run.rs` в `p22-wasm-host`: `cargo run --bin run -- <module.wat> [numbers...]`
загружает модуль, вызывает его экспорт `run` с числами в качестве аргументов `i64` (любое их
количество), печатает результат и использует топливо. Проверьте его на `square.wat`, `near.wat` и
`div.wat` из раздела 4.

<details>
<summary>Подсказка</summary>

Число параметров неизвестно во время компиляции, поэтому используйте нетипизированный API (Application Programming Interface, программный интерфейс):
`instance.get_func(...)` даёт `Func`, а `func.call(&mut store, &params, &mut results)` принимает срезы
`wasmtime::Val` (`Val::I64(n)`). `Module::from_file` читает файлы `.wat` и `.wasm`.

</details>

<details>
<summary>Ответ</summary>

```rust
//! Usage: cargo run --bin run -- <module.wat> [i64 argument ...]
use anyhow::{Context, Result};
use wasmtime::{Config, Engine, Instance, Module, Store, Val};

fn main() -> Result<()> {
    let args: Vec<String> = std::env::args().skip(1).collect();
    let (path, numbers) = args
        .split_first()
        .context("usage: run <module.wat> [arg ...]")?;

    let mut config = Config::new();
    config.consume_fuel(true);
    let engine = Engine::new(&config)?;
    let module = Module::from_file(&engine, path)?;
    let mut store = Store::new(&engine, ());
    store.set_fuel(1_000_000)?;
    let instance = Instance::new(&mut store, &module, &[])?;

    let run = instance
        .get_func(&mut store, "run")
        .context("the module has no `run` export")?;
    let params = numbers
        .iter()
        .map(|n| n.parse::<i64>().map(Val::I64))
        .collect::<Result<Vec<Val>, _>>()?;
    let mut results = [Val::I64(0)];
    run.call(&mut store, &params, &mut results)?;
    println!("{}", results[0].unwrap_i64());
    Ok(())
}
```

```bash
cargo run -q --bin run -- ../p22-pebble/square.wat 7
cargo run -q --bin run -- ../p22-pebble/near.wat 5 70
cargo run -q --bin run -- ../p22-pebble/div.wat 7 0
cargo run -q --bin run -- ../p22-pebble/square.wat 1 2
```

```text
50
0
Error: error while executing at wasm backtrace:
    0:     0x27 - <unknown>!<wasm function 0>

Caused by:
    wasm trap: integer divide by zero
Error: expected 1 arguments, got 2
```

Почему это работает: `Func::call` сверяет число и типы аргументов с типом функции и возвращает ошибку,
а не вызывает функцию с неправильными аргументами. Ловушки возвращаются как обычные значения `Err`,
поэтому `?` передаёт их в `main`, который печатает их и завершается с кодом 1.

</details>

## Вопросы для самопроверки

1. Почему один файл `.wasm` может работать и на Mac с arm64, и на сервере x86-64?

<details>
<summary>Ответ</summary>

WebAssembly — набор инструкций для виртуальной машины, а не для настоящего процессора. Среда
выполнения на каждой машине компилирует модуль в собственный код этой машины прямо перед запуском.

</details>

2. Что может делать модуль WebAssembly, который ничего не импортирует?

<details>
<summary>Ответ</summary>

Только вычислять: пользоваться своей линейной памятью и своими функциями и возвращать результаты хосту.
Он не может читать файлы, пользоваться сетью, печатать или читать часы, потому что всё это нужно
импортировать.

</details>

3. Чем `wasm32-wasip1` отличается от `wasm32-unknown-unknown`?

<details>
<summary>Ответ</summary>

`wasm32-wasip1` — для целых программ, которые используют импорты WASI (файлы, стандартный вывод,
аргументы) и работают в среде выполнения WASI. `wasm32-unknown-unknown` не предполагает никакой
операционной системы; его используют для библиотек, хост которых предоставляет всё.

</details>

4. В каком порядке генератор кода для стековой машины выдаёт части `a - b` и почему?

<details>
<summary>Ответ</summary>

Сначала код для `a`, затем для `b`, затем `i64.sub`. Оператор берёт операнды со стека, поэтому они
должны оказаться там до его выполнения (обход в обратном порядке).

</details>

5. Что такое ловушка и что происходит с хостом, когда модуль выдаёт ловушку?

<details>
<summary>Ответ</summary>

Ловушка — немедленная остановка кода WebAssembly из-за ошибки, например деления на ноль, инструкции
`unreachable` или закончившегося топлива. Хост получает значение ошибки и продолжает работу.

</details>

6. Назовите два различия между скомпилированными выражениями Pebble и интерпретатором из Урока 22.23.

<details>
<summary>Ответ</summary>

В WebAssembly переполнение переносит значение по модулю, а в интерпретаторе это ошибка; деление на ноль
в WebAssembly даёт ловушку, а в интерпретаторе — ошибку выполнения с позицией. (Ещё: логические
значения становятся целыми 0 и 1.)

</details>

## Итоги

- WebAssembly — переносимый формат инструкций для стековой машины с песочницей; среда выполнения вроде
  Wasmtime компилирует его в машинный код.
- У модуля есть функции, линейная память, импорты и экспорты; без импортов он может только вычислять.
- `wasm32-wasip1` собирает целые программы на Rust, использующие WASI; папки нужно выдавать через `--dir`.
- `wasm32-unknown-unknown` с `crate-type = ["cdylib"]` и `#[unsafe(no_mangle)] extern "C"` собирает
  библиотеки, которые экспортируют функции; паника там становится ловушкой.
- Генератор кода для стековой машины выдаёт сначала операнды, затем оператор: обход синтаксического
  дерева в обратном порядке.
- Крейт `wasmtime` встраивает среду выполнения в ваши программы; топливо останавливает бесконечные
  циклы.

## Источники и что почитать

- [WebAssembly](https://webassembly.org/) — официальный сайт со списком возможностей и сред выполнения.
- [WebAssembly Core Specification](https://webassembly.github.io/spec/core/) — точное определение каждой инструкции и проверки модулей.
- [MDN: Understanding WebAssembly text format](https://developer.mozilla.org/en-US/docs/WebAssembly/Guides/Understanding_the_text_format) — дружелюбное руководство по WAT (MDN, MDN Web Docs, — веб-документация Mozilla).
- [Wasmtime documentation](https://docs.wasmtime.dev/) и [установка CLI](https://docs.wasmtime.dev/cli-install.html) — инструмент командной строки (CLI, command-line interface, интерфейс командной строки), топливо и встраивание.
- [wasmtime crate on docs.rs](https://docs.rs/wasmtime/50/wasmtime/) — `Engine`, `Store`, `Module`, `Instance`, `Func`.
- [WASI](https://wasi.dev/) — системный интерфейс WebAssembly и его версии.
- [rustc book: wasm32-wasip1](https://doc.rust-lang.org/rustc/platform-support/wasm32-wasip1.html) и [wasm32-unknown-unknown](https://doc.rust-lang.org/rustc/platform-support/wasm32-unknown-unknown.html) — что поддерживает каждая целевая платформа Rust.

---

[🏠 Начало](../START-HERE.md) · [← 22.23 Трек компиляторов: интерпретатор обхода дерева](23-compilers-track-an-interpreter.md) · [📋 Этап 22](README.md) · [22.25 Трек компиляторов: проект и следующие шаги →](25-compilers-track-project-and-next-steps.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
