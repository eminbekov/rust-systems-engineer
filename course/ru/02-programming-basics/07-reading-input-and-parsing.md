<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 02.6 Вывод, форматирование и комментарии](06-printing-formatting-and-comments.md) · [📋 Этап 02](README.md) · [02.8 Ваши первые тесты →](08-your-first-tests.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/02-programming-basics/07-reading-input-and-parsing.md) <!-- nav -->

# Урок 02.7 — Чтение ввода и преобразование строк

> **Этап 02 — Основы программирования на Rust** · Урок 7 из 11 · ⏱ около 3 часов (2 занятия)

## Что вы узнаете

- Как программа читает строку, которую набирает пользователь (стандартный ввод).
- Почему прочитанный текст заканчивается переводом строки и как `trim` его убирает.
- Как превратить текст в число с помощью `parse` и что такое `Result`.
- Как обработать неверный ввод без аварийного завершения и как спрашивать снова, пока ввод не станет верным.
- Как сделать так, чтобы приглашение появилось до ввода пользователя, и как проверять код ввода из терминала.

## Перед началом

- Вы прошли [Урок 02.6 — Вывод, форматирование и комментарии](06-printing-formatting-and-comments.md).
- Вы помните `loop`, `break` и `match` из [Урока 02.5](05-control-flow.md).
- Вы помните терминал и идею перенаправления потоков из [Урока 02.6](06-printing-formatting-and-comments.md).

## Новые слова

| Слово | Значение |
|---|---|
| **stdin** (standard input, стандартный ввод) | Обычный поток ввода программы. В терминале это то, что вы набираете. |
| **I/O** (Input/Output, ввод/вывод) | Чтение данных в программу и запись данных из неё. |
| **EOF** (End Of File, конец файла) | Сигнал, что ввода больше нет. В терминале его отправляет `Ctrl+D` на пустой строке. |
| **буфер** (buffer) | Участок памяти, в котором хранятся данные на пути внутрь или наружу. |
| **разбор** (parse) | Прочитать текст и превратить его в значение какого-то типа, например `"42"` в число `42`. |
| **`Result`** | Тип, который равен либо `Ok(value)` (получилось), либо `Err(error)` (не получилось). Подробно его изучает Этап 04, Урок 10. |
| **`expect`** | Метод `Result`: если `Ok`, отдаёт значение; если `Err`, паникует с вашим сообщением. |
| **trim** | Убрать пробельные символы (пробелы, табуляции, переводы строк) в начале и в конце текста. |
| **конвейер** (pipe, `\|`) | Символ терминала, который отправляет вывод одной команды на ввод следующей. |
| **ссылка** (reference, `&`) | Способ позволить другому фрагменту кода использовать значение, не отдавая его. Это изучает Этап 03. |
| **UTF-8** (Unicode Transformation Format, 8-bit) | Стандартный способ хранить текст Unicode в виде байтов. Текст в Rust всегда корректный UTF-8. |
| **сброс буфера** (flush) | Принудительно отправить буфер сейчас, а не ждать. |

## 1. Как приходит ввод

Программа общается с миром через **потоки**. Выходные вы знаете из Урока 02.6: stdout и stderr. Входной — это **stdin**: клавиатура, когда вы запускаете программу в терминале, или вывод другой команды, когда вы их соединяете.

```text
 keyboard --> terminal --> [ stdin ] --> your program --> [ stdout ] --> terminal screen
                                                      \-> [ stderr ] --> terminal screen
```

Программа не видит отдельных клавиш. Терминал собирает то, что вы набираете, и когда вы нажимаете **Enter**, отправляет всю строку, включая символ перевода строки `\n` в конце.

## 2. Чтение строки

```rust
use std::io;

fn main() {
    let mut input = String::new();
    println!("What is your name?");
    io::stdin()
        .read_line(&mut input)
        .expect("failed to read a line");
    println!("raw:     {:?}", input);
    let name = input.trim();
    println!("trimmed: {:?}", name);
    println!("Hello, {name}!");
}
```

Запустите так: `echo "Aziza" | cargo run -q --bin 07_read_line` (конвейер `|` отправляет текст `Aziza` в stdin; раздел 8 объясняет это) или выполните `cargo run -q --bin 07_read_line` и наберите своё имя.

```text
What is your name?
raw:     "Aziza\n"
trimmed: "Aziza"
Hello, Aziza!
```

Что происходит:

1. `use std::io;` делает модуль `io` стандартной библиотеки (`std`) доступным под коротким именем `io`. *Модуль* — это именованная группа элементов; их изучает Этап 04, урок 6.
2. `String::new()` создаёт пустой `String`, растущий текстовый тип из Урока 02.6. Он объявлен `mut`, потому что `read_line` будет в него писать. Эта переменная — **буфер**, который получает текст.
3. `io::stdin()` даёт доступ к стандартному вводу. `.read_line(&mut input)` ждёт, пока пользователь нажмёт Enter, затем **дописывает** строку в `input`.
4. `&mut input` значит: «одолжить `input` методу `read_line` и разрешить ему менять его». `&` создаёт **ссылку**: способ дать доступ, не отдавая значение. Ссылки полностью объясняет Этап 03. Пока запомните: `read_line` всегда нужен `&mut` перед буфером.
5. `read_line` возвращает `Result`, потому что чтение может не удаться (например, если ввод — не корректный UTF-8). Метод `.expect("…")` говорит: «если получилось — продолжай; если нет — паникуй с этим сообщением». Более вежливо обрабатывать ошибки мы будем в разделе 5.
6. `{:?}` показывает `"Aziza\n"`: в конце текста есть скрытый `\n`. Поэтому при отладке мы используем Debug.
7. `input.trim()` возвращает текст без пробельных символов с обоих концов. Теперь `name` — это `"Aziza"`.

Две детали, которые нужно запомнить:

- `read_line` **дописывает**. Если вы используете тот же `String` для следующей строки, не очистив его, новая строка добавится после старой:

```rust
use std::io;

fn main() {
    let mut input = String::new();
    io::stdin().read_line(&mut input).expect("read failed");
    io::stdin().read_line(&mut input).expect("read failed");
    println!("{:?}", input);
}
```

При вводе `one` и `two` на разных строках напечатается `"one\ntwo\n"`. Создавайте новый `String` для каждой строки (как в цикле ниже) или вызывайте `input.clear()`.

- В конце ввода (**EOF**) `read_line` возвращает `Ok(0)` — ноль байтов — и оставляет буфер пустым. Цикл, который читает строки, должен проверять это, иначе будет крутиться вечно:

```rust
use std::io;

fn main() {
    let mut input = String::new();
    let bytes = io::stdin()
        .read_line(&mut input)
        .expect("failed to read a line");
    println!("bytes read: {bytes}");
    println!("text: {:?}", input);
}
```

```text
bytes read: 3
text: "hi\n"
```

Это вывод для ввода `hi`. С пустым вводом (`cargo run -q --bin 07_eof < /dev/null`) результат — `bytes read: 0` и `text: ""`.

## 3. От текста к числу: `parse`

Всё, что вы читаете, — текст. Чтобы считать, его нужно **разобрать** (parse) в число:

```rust
use std::io;

fn main() {
    let mut input = String::new();
    io::stdin()
        .read_line(&mut input)
        .expect("failed to read a line");
    let number: i32 = input.trim().parse().expect("please type a whole number");
    println!("{number} doubled is {}", number * 2);
}
```

```text
21 doubled is 42
```

(для ввода `21`.) Что происходит:

1. `input.trim()` убирает перевод строки. Это необходимо: `"21\n"` — не корректное число.
2. `.parse()` пытается превратить текст в число. **В какой тип?** Компилятор узнаёт его из `let number: i32`. Поэтому нужна аннотация типа (вспомните ошибку `E0284` в Уроке 02.2). Тип можно также написать рядом с методом: `.parse::<i32>()`.
3. `parse` возвращает `Result`, как и `read_line`. Его `.expect(…)` отдаёт число, если текст был корректным.
4. При вводе `abc` программа паникует:

```text
thread 'main' (4056609) panicked at src/main.rs:8:44:
please type a whole number: ParseIntError { kind: InvalidDigit }
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
```

Паника допустима в быстром эксперименте, но настоящая программа не должна падать из-за опечатки. Чтобы обрабатывать ошибки, нужно понять `Result`.

### Что `parse` принимает и отвергает

`parse::<T>()` работает для чисел, `bool`, `char` и других типов. Посмотрите, что он принимает (`Ok`) и отвергает (`Err`):

```rust
fn main() {
    println!("{:?}", "42".parse::<i32>());
    println!("{:?}", " 42 ".parse::<i32>());
    println!("{:?}", "42\n".parse::<i32>());
    println!("{:?}", "abc".parse::<i32>());
    println!("{:?}", "".parse::<i32>());
    println!("{:?}", "3.5".parse::<i32>());
    println!("{:?}", "3.5".parse::<f64>());
    println!("{:?}", "99999999999".parse::<i32>());
    println!("{:?}", "-5".parse::<u32>());
    println!("{:?}", "+7".parse::<i32>());
    println!("{:?}", "true".parse::<bool>());
    println!("{:?}", "x".parse::<char>());
}
```

```text
Ok(42)
Err(ParseIntError { kind: InvalidDigit })
Err(ParseIntError { kind: InvalidDigit })
Err(ParseIntError { kind: InvalidDigit })
Err(ParseIntError { kind: Empty })
Err(ParseIntError { kind: InvalidDigit })
Ok(3.5)
Err(ParseIntError { kind: PosOverflow })
Err(ParseIntError { kind: InvalidDigit })
Ok(7)
Ok(true)
Ok('x')
```

Замечания:

- Пробелы и переводы строк вокруг числа делают его **некорректным** (строки 2 и 3). Всегда делайте `trim()` сначала.
- Пустой текст, буквы и число с дробной частью не являются корректными значениями `i32`. Текст `"3.5"` корректен для `f64`.
- Число, слишком большое для типа, даёт `PosOverflow` («положительное переполнение»). Отрицательное число для беззнакового типа даёт `InvalidDigit`, потому что `-` не цифра для `u32`.
- Ведущий `+` принимается.

## 4. Разбираемся с `Result`

`Result` — это значение, которое **или** успех, **или** неудача:

```text
Result<i32, ParseIntError>
   |
   +-- Ok(42)                         the number was parsed
   +-- Err(ParseIntError { ... })     it failed, and here is the reason
```

Две части в угловых скобках — это типы двух случаев: здесь значение успеха — `i32`, а значение ошибки — `ParseIntError`. (Типы с `< >` хранят другие типы; обобщения объясняет Этап 05, урок 1.)

Rust заставляет вас разобраться с обоими случаями. Нельзя использовать число, пока вы не проверили, что оно есть. Сравните:

```rust
fn main() {
    let n: i32 = "42".parse();
    println!("{n}");
}
```

```text
error[E0308]: mismatched types
 --> src/main.rs:2:18
  |
2 |     let n: i32 = "42".parse();
  |            ---   ^^^^^^^^^^^^ expected `i32`, found `Result<_, _>`
  |            |
  |            expected due to this
  |
help: consider using `Result::expect` to unwrap the `Result<_, _>` value, panicking if the value is a `Result::Err`
```

Текст пока не число; это `Result`, который может *содержать* число. Чтобы получить значение, нужно обработать оба случая. `expect` обрабатывает неудачу паникой. Вежливый способ — `match`, который вы знаете из Урока 02.5:

```rust
fn main() {
    match "abc".parse::<i32>() {
        Ok(n) => println!("number {n}"),
        Err(e) => println!("error message: {e}"),
    }
    match "".parse::<i32>() {
        Ok(n) => println!("number {n}"),
        Err(e) => println!("error message: {e}"),
    }
    match "99999999999".parse::<i32>() {
        Ok(n) => println!("number {n}"),
        Err(e) => println!("error message: {e}"),
    }
}
```

```text
error message: invalid digit found in string
error message: cannot parse integer from empty string
error message: number too large to fit in target type
```

Что происходит: образец `Ok(n)` подходит для успеха и называет значение внутри `n`. Образец `Err(e)` подходит для неудачи и называет ошибку `e`. С `{e}` (Display) ошибка печатается как сообщение, понятное человеку, — гораздо приятнее, чем вид Debug `ParseIntError { kind: InvalidDigit }`.

## 5. Спрашиваем снова, пока ввод не станет верным

Теперь объединим `loop`, `read_line`, `parse` и `match`:

```rust
use std::io;

fn main() {
    loop {
        println!("Type a whole number:");
        let mut input = String::new();
        io::stdin()
            .read_line(&mut input)
            .expect("failed to read a line");
        match input.trim().parse::<i32>() {
            Ok(n) => {
                println!("Thank you! {n} squared is {}", n * n);
                break;
            }
            Err(e) => {
                println!("'{}' is not a number ({e}). Try again.", input.trim());
            }
        }
    }
}
```

Что происходит:

1. `loop` повторяется, пока не будет прочитано корректное число.
2. На каждой итерации создаётся **новый** `String`, поэтому строки не накапливаются.
3. Если текст — число, ветка `Ok` печатает квадрат, и `break` выходит из цикла.
4. Иначе ветка `Err` печатает сообщение, и цикл начинается снова.

Для входных строк `hello`, пустая строка, `3.5`, `12`:

```text
Type a whole number:
'hello' is not a number (invalid digit found in string). Try again.
Type a whole number:
'' is not a number (cannot parse integer from empty string). Try again.
Type a whole number:
'3.5' is not a number (invalid digit found in string). Try again.
Type a whole number:
Thank you! 12 squared is 144
```

> ⚠️ **Частая ошибка:** если поток ввода закончится (EOF), `read_line` вернёт `Ok(0)` и пустой текст; цикл выше будет вечно печатать сообщение «not a number». Надёжный цикл проверяет число байтов: `if bytes == 0 { break; }`. Упражнение 5 делает это.

## 6. Приглашение на той же строке

Вы хотите, чтобы курсор ждал сразу после вопроса: `Your age: ▮`. Для этого используется `print!` (без перевода строки). Но Rust отправляет вывод в терминал построчно, поэтому приглашение может не появиться до того, как программа начнёт ждать ввод. Нужно **сбросить буфер** вывода (flush):

```rust
use std::io::{self, Write};

fn main() {
    print!("Your age: ");
    io::stdout().flush().expect("failed to flush");
    let mut input = String::new();
    io::stdin()
        .read_line(&mut input)
        .expect("failed to read a line");
    println!("You typed: {}", input.trim());
}
```

```text
Your age: You typed: 30
```

(Вывод получен для ввода `30` через конвейер, поэтому набранный текст не виден; в настоящем терминале вы видите свой набор после `Your age: `.)

Что происходит:

1. `use std::io::{self, Write};` импортирует две вещи: сам модуль `io` (`self`) и `Write` — **трейт**. Трейт — это набор возможностей; метод `flush` принадлежит `Write`, а чтобы использовать методы трейта, его нужно импортировать. (Трейты изучает Этап 05, урок 2.)
2. `io::stdout().flush()` сразу отправляет на экран всё, что ждёт в буфере stdout. Он возвращает `Result`, поэтому мы вызываем `expect`.

## 7. Несколько значений на одной строке

Если пользователь набрал `3 4 5`, разбейте строку на слова с помощью `split_whitespace()` и разберите каждое:

```rust
use std::io;

fn main() {
    let mut line = String::new();
    io::stdin().read_line(&mut line).expect("read failed");
    let mut sum = 0;
    let mut count = 0;
    for word in line.split_whitespace() {
        match word.parse::<i32>() {
            Ok(n) => {
                sum += n;
                count += 1;
            }
            Err(_) => println!("skipping {word:?}"),
        }
    }
    println!("count = {count}, sum = {sum}");
}
```

Для ввода `3 4 x 5`:

```text
skipping "x"
count = 3, sum = 12
```

`split_whitespace()` выдаёт слова по одному, игнорируя любое число пробелов и табуляций, а `for` проходит по ним. Образец `Err(_)` значит «ошибка; подробности мне не нужны» (`_` игнорирует значение).

## 8. Логика в функции и её тестирование

Код ввода трудно тестировать, потому что он ждёт человека. Решение — отделить **работу** от **чтения**: читайте в `main`, а решение вынесите в функцию, которая принимает текст и возвращает `Result`. Затем проверьте функцию через `assert_eq!`:

```rust
use std::num::ParseIntError;

fn parse_number(text: &str) -> Result<i32, ParseIntError> {
    text.trim().parse()
}

fn main() {
    assert_eq!(parse_number("42"), Ok(42));
    assert_eq!(parse_number("  -7\n"), Ok(-7));
    assert!(parse_number("abc").is_err());
    assert!(parse_number("").is_err());
    assert!(parse_number("2147483648").is_err());
    println!("parse_number: all checks passed");
}
```

```text
parse_number: all checks passed
```

Что происходит:

1. Параметр `text: &str` — это кусок текста, на который функция только смотрит (почему тип `&str`, вы узнаете на Этапе 03).
2. Возвращаемый тип `Result<i32, ParseIntError>` сообщает в сигнатуре, что функция может завершиться неудачей. Тип ошибки лежит в `std::num`.
3. Тело — просто `text.trim().parse()`; его тип определяется возвращаемым типом.
4. `Ok(42)` создаёт значение успеха для сравнения. `.is_err()` отвечает `true` для `Err`.
5. `2147483648` — это `i32::MAX + 1`: слишком большое число, значит, ошибка. Крайний случай!

> 🧪 **Тестирование:** всегда проверяйте **плохие** входные данные, а не только хорошие: пустой текст, буквы, пробелы, слишком большое число, отрицательное число для беззнакового типа. Злоумышленники и уставшие пользователи находят их первыми.

### Подаём ввод вашей программе из терминала

Набирать ввод каждый раз не нужно. Терминал умеет соединять потоки:

| Команда | Что делает |
|---|---|
| `echo "42" \| cargo run -q --bin x` | `echo` печатает `42` и перевод строки; конвейер `\|` отправляет это в stdin программы. |
| `printf '10\nabc\n5\n' \| cargo run -q --bin x` | `printf` позволяет написать несколько строк через `\n`. |
| `cargo run -q --bin x < input.txt` | `<` читает stdin из файла. |
| `cargo run -q --bin x < /dev/null` | Пустой ввод, чтобы проверить EOF. |

Именно так вы будете проверять приёмочные тесты проекта в Уроке 02.10.

> 🔒 **Безопасность:** **Никогда не доверяйте вводу.** Всё, что приходит снаружи — с клавиатуры, из файлов, из сети, — может быть неверным, огромным или враждебным. Проверяйте перед использованием: разберите, проверьте диапазон (`age <= 150`) и решите, что делать при неудаче. Не используйте `expect` или `unwrap` для пользовательского ввода в настоящей программе: пользователь может обрушить вашу программу одним неверным нажатием. Заметьте также, что `read_line` читает строку любой длины; программа, которая должна противостоять злоумышленникам, ограничивает объём чтения. Вы увидите это на Этапе 10.

## Частые ошибки

**1. Забыли `mut` у буфера.**

```text
error[E0596]: cannot borrow `input` as mutable, as it is not declared as mutable
 --> src/main.rs:5:27
  |
5 |     io::stdin().read_line(&mut input).expect("read failed");
  |                           ^^^^^^^^^^ cannot borrow as mutable
  |
help: consider changing this to be mutable
  |
4 |     let mut input = String::new();
  |         +++
```

Напишите `let mut input = String::new();`.

**2. Забыли `trim()`.** `"42\n".parse::<i32>()` — ошибка (`InvalidDigit`). Сообщение «invalid digit found in string» для ввода, который *выглядит* нормально, — обычный признак.

**3. Игнорирование `Result`.** Если вызвать `read_line` и отбросить результат, компилятор предупредит:

```text
warning: unused `Result` that must be used
 --> src/main.rs:5:5
  |
5 |     io::stdin().read_line(&mut input);
  |     ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  |
  = note: this `Result` may be an `Err` variant, which should be handled
```

Добавьте `.expect("…")` или обработайте через `match`.

**4. Повторное использование буфера.** `read_line` дописывает. Создавайте новый `String` или вызывайте `clear()`.

**5. Нет типа для `parse`.** `E0284 type annotations needed`: добавьте `let n: i32` или `parse::<i32>()`.

**6. `unwrap` или `expect` для пользовательского ввода.** Годится для быстрой проверки; неправильно для настоящей программы.

> ⚡ **Производительность:** `read_line` в новый `String` для каждой строки каждый раз выделяет память. Для небольших программ это неважно. Для программ, которые читают миллионы строк, используют один буфер с `clear()` или буферизованные читатели (Этап 05).

## Попробуйте сами

1. Создайте `src/bin/07_read_line.rs` с первой программой из раздела 2. Запустите её, набрав своё имя, и через конвейер: `echo "Rust" | cargo run -q --bin 07_read_line`.
2. Создайте `src/bin/07_double.rs`, которая спрашивает `Enter a number:` (через `print!` и `flush`), разбирает ввод как `f64` и печатает удвоенное значение. Обработайте неверный ввод через `match`: напечатайте `That is not a number.` и остановитесь.
3. Проверьте её из терминала: `echo 2.5 | cargo run -q --bin 07_double`, `echo abc | cargo run -q --bin 07_double` и `cargo run -q --bin 07_double < /dev/null`.
4. Превратите её в цикл, который спрашивает снова, пока ввод не станет верным (раздел 5). Остановитесь, если ввод закончился.
5. Добавьте функцию `parse_number` из раздела 8 в файл и напишите пять проверок `assert!`.
6. Выполните `cargo fmt`, `cargo clippy`. Сделайте коммит:

```bash
cd ~/projects/rust
git add code/p02-basics
git commit -m "feat(p02): add input and parsing examples"
```

## Упражнения

### Упражнение 1 — Приветствие (легко)

Спросите `What is your name?`, прочитайте одну строку и напечатайте `Hello, <name>!` без перевода строки внутри приветствия. Запустите через конвейер для проверки.

<details>
<summary>Подсказка</summary>

Нужны `let mut input = String::new();`, `io::stdin().read_line(&mut input)` и `input.trim()`.

</details>

<details>
<summary>Ответ</summary>

```rust
use std::io;

fn main() {
    println!("What is your name?");
    let mut input = String::new();
    io::stdin()
        .read_line(&mut input)
        .expect("failed to read a line");
    println!("Hello, {}!", input.trim());
}
```

Проверка: `echo Aziza | cargo run -q --bin x` печатает `What is your name?` и `Hello, Aziza!`. Без `trim()` приветствие разорвалось бы на две строки, потому что `input` заканчивается на `\n`.

</details>

### Упражнение 2 — Сложение двух чисел (легко)

Прочитайте два числа с двух строк и напечатайте `a + b = sum`. Вспомогательная функция `read_number() -> i32` должна выполнять чтение и разбор (пока можно использовать `expect`).

<details>
<summary>Подсказка</summary>

Тело вспомогательной функции — это три строки из раздела 3. Её последнее выражение — разобранное число без точки с запятой. Вызовите её дважды в `main`.

</details>

<details>
<summary>Ответ</summary>

```rust
use std::io;

fn read_number() -> i32 {
    let mut input = String::new();
    io::stdin().read_line(&mut input).expect("failed to read a line");
    input.trim().parse().expect("please type a whole number")
}

fn main() {
    let a = read_number();
    let b = read_number();
    println!("{a} + {b} = {}", a + b);
}
```

Проверка: `printf '20\n22\n' | cargo run -q --bin x` печатает `20 + 22 = 42`. Возвращаемый тип `i32` сообщает `parse`, какой тип создавать.

</details>

### Упражнение 3 — Проверяющая функция с тестами (средне)

Напишите `fn parse_age(text: &str) -> Result<u8, String>`. Она должна обрезать текст, разобрать `u8` и принимать только возраст от 0 до 150. При неудаче она возвращает `Err` с понятным сообщением (`String`, созданный через `format!`). Проверьте: `"29"` → `Ok(29)`, `" 0\n"` → `Ok(0)`, `"150"` → `Ok(150)`, `"151"` → `Err`, `"abc"`, `"-3"` и `"300"` → `Err`.

<details>
<summary>Подсказка</summary>

Используйте `match text.trim().parse::<u8>()`. В ветке `Ok(age)` примените `if age <= 150 { Ok(age) } else { Err(format!(…)) }`. Заметьте, что тип `u8` уже отвергает `-3` и `300` за вас.

</details>

<details>
<summary>Ответ</summary>

```rust
fn parse_age(text: &str) -> Result<u8, String> {
    match text.trim().parse::<u8>() {
        Ok(age) => {
            if age <= 150 {
                Ok(age)
            } else {
                Err(format!("{age} is too big for an age"))
            }
        }
        Err(_) => Err(format!("'{}' is not a whole number from 0 to 255", text.trim())),
    }
}

fn main() {
    assert_eq!(parse_age("29"), Ok(29));
    assert_eq!(parse_age(" 0\n"), Ok(0));
    assert_eq!(parse_age("150"), Ok(150));
    assert_eq!(parse_age("151"), Err(String::from("151 is too big for an age")));
    assert!(parse_age("abc").is_err());
    assert!(parse_age("-3").is_err());
    assert!(parse_age("300").is_err());
    println!("{:?}", parse_age("151"));
    println!("{:?}", parse_age("abc"));
    println!("parse_age: all checks passed");
}
```

```text
Err("151 is too big for an age")
Err("'abc' is not a whole number from 0 to 255")
parse_age: all checks passed
```

Почему это работает: тип `u8` ограничивает разбор диапазоном 0–255, а `if` добавляет настоящее правило (не больше 150). `Ok(age)` и `Err(…)` создают два вида `Result`.

</details>

### Упражнение 4 — Среднее по строке (средне)

Прочитайте одну строку с несколькими числами (возможно, десятичными), например `10 20 x 30`. Пропускайте слова, которые не являются числами (для каждого печатайте предупреждение в **stderr**), и напечатайте среднее остальных. Если чисел нет вообще, напечатайте `no numbers to average` вместо деления на ноль.

<details>
<summary>Подсказка</summary>

Используйте `split_whitespace()`, `parse::<f64>()` и два счётчика: `sum` и `count`. Для деления напишите `sum / count as f64`. Сначала проверьте `count == 0`.

</details>

<details>
<summary>Ответ</summary>

```rust
use std::io;

fn main() {
    let mut line = String::new();
    io::stdin().read_line(&mut line).expect("failed to read a line");
    let mut sum = 0.0;
    let mut count = 0;
    for word in line.split_whitespace() {
        match word.parse::<f64>() {
            Ok(x) => {
                sum += x;
                count += 1;
            }
            Err(_) => eprintln!("skipping {word:?}"),
        }
    }
    if count == 0 {
        println!("no numbers to average");
    } else {
        println!("average of {count} numbers = {}", sum / count as f64);
    }
}
```

Для `echo "10 20 x 30"` вывод: `skipping "x"` (в stderr) и `average of 3 numbers = 20`. Для пустой строки напечатается `no numbers to average`.

Почему это работает: предупреждения идут в stderr, поэтому не смешиваются с результатом; а проверка `count == 0` избегает деления на ноль (для чисел с плавающей точкой получился бы `NaN` — неверный ответ без аварии).

</details>

### Упражнение 5 — Сумма до `done` (сложно)

Напишите программу, которая читает строки до строки `done` или до конца ввода. Каждая строка, являющаяся корректным `i64`, добавляется к сумме. О строках, которые некорректны, сообщается в stderr с причиной (`skipped "abc": invalid digit found in string`), и они подсчитываются. В конце напечатайте `accepted N, skipped M, sum = S`. Программа никогда не должна паниковать из-за ввода и должна останавливаться на EOF.

<details>
<summary>Подсказка</summary>

Используйте `loop`. Каждый раз читайте в **новый** `String`. Если `read_line` вернул `0`, сделайте `break`. Обрежьте текст, сделайте `break`, если он равен `"done"`, затем `match text.parse::<i64>()`.

</details>

<details>
<summary>Ответ</summary>

```rust
use std::io;

fn main() {
    let mut sum = 0i64;
    let mut good = 0;
    let mut bad = 0;
    loop {
        let mut line = String::new();
        let bytes = io::stdin().read_line(&mut line).expect("failed to read a line");
        if bytes == 0 {
            break;
        }
        let text = line.trim();
        if text == "done" {
            break;
        }
        match text.parse::<i64>() {
            Ok(n) => {
                sum += n;
                good += 1;
            }
            Err(e) => {
                eprintln!("skipped {text:?}: {e}");
                bad += 1;
            }
        }
    }
    println!("accepted {good}, skipped {bad}, sum = {sum}");
}
```

Проверка с `printf '10\nabc\n5\n\n-3\ndone\n99\n' | cargo run -q --bin x`:

```text
skipped "abc": invalid digit found in string
skipped "": cannot parse integer from empty string
accepted 3, skipped 2, sum = 12
```

Почему это работает: `10 + 5 - 3 = 12`; строка `99` после `done` никогда не читается. Для ввода `1`, `2` и затем EOF (без `done`) цикл останавливается при `bytes == 0` и печатает `accepted 2, skipped 0, sum = 3`.

</details>

## Вопросы для самопроверки

1. Почему текст из `read_line` заканчивается на `\n`?

<details>
<summary>Ответ</summary>

Терминал отправляет всю строку, включая символ перевода строки, который появляется при нажатии Enter. Используйте `trim()`, чтобы убрать его.

</details>

2. Почему буфер должен быть `mut` и передаваться как `&mut input`?

<details>
<summary>Ответ</summary>

`read_line` пишет в буфер, поэтому буфер должен быть изменяемым (`mut`), а `&mut` одалживает его методу с разрешением менять.

</details>

3. Что возвращает `"abc".parse::<i32>()`?

<details>
<summary>Ответ</summary>

`Result`: `Err(ParseIntError { kind: InvalidDigit })`. Корректный текст дал бы `Ok(number)`.

</details>

4. Чем отличаются `expect` и `match` при обработке `Result`?

<details>
<summary>Ответ</summary>

`expect` отдаёт значение при `Ok` и паникует при `Err`. `match` позволяет решить, что делать в каждом случае, например напечатать сообщение и спросить снова.

</details>

5. Что возвращает `read_line` в конце ввода?

<details>
<summary>Ответ</summary>

`Ok(0)` — прочитано ноль байтов — и буфер остаётся пустым.

</details>

6. Зачем вызывать `flush()` после `print!`?

<details>
<summary>Ответ</summary>

Вывод буферизуется построчно, поэтому текст без перевода строки может остаться в буфере. `flush()` сразу отправляет его на экран, так что приглашение появляется до того, как программа начнёт ждать ввод.

</details>

7. Как проверить программу, читающую из stdin, не набирая вручную?

<details>
<summary>Ответ</summary>

Подайте ввод через конвейер `echo "text" | cargo run`, либо `printf` для многих строк, либо перенаправьте файл `< file.txt`. А ещё лучше вынести логику в функцию, которая принимает текст, и проверить её через `assert_eq!`.

</details>

## Итоги

- `std::io::stdin().read_line(&mut buffer)` читает одну строку **вместе с** `\n` и дописывает её; буфер должен быть `mut`.
- Делайте `trim()` перед `parse()`. `parse::<T>()` (или `let` с типом) превращает текст в значение.
- `read_line` и `parse` возвращают `Result`: `Ok(value)` или `Err(error)`. `expect` паникует при `Err`; `match` позволяет обработать его.
- Цикл с `match` может спрашивать снова, пока ввод не станет верным. Проверяйте EOF (`Ok(0)`).
- Для приглашения на той же строке используйте `print!` и `io::stdout().flush()`.
- Выносите решения в функции, которые принимают текст и возвращают `Result`; тестируйте их на хороших и плохих входных данных.
- Никогда не доверяйте вводу. Проверяйте программы из терминала с помощью `|` и `<`.

## Источники и что почитать

- [The Rust Programming Language — 2. Programming a Guessing Game](https://doc.rust-lang.org/book/ch02-00-guessing-game-tutorial.html) — чтение ввода, `parse` и `match` по `Result` в форме игры (проект этого этапа).
- [Standard library — std::io::Stdin](https://doc.rust-lang.org/std/io/struct.Stdin.html) — документация `read_line` и других способов чтения.
- [Standard library — str::parse](https://doc.rust-lang.org/std/primitive.str.html#method.parse) — что делает `parse` и какие типы поддерживает.
- [Standard library — std::result::Result](https://doc.rust-lang.org/std/result/enum.Result.html) — все методы `Result` (`is_ok`, `is_err`, `expect`…).
- [Standard library — std::io](https://doc.rust-lang.org/std/io/index.html) — весь модуль ввода и вывода: читатели, писатели и трейт `Write`, используемый для `flush`.

---

[🏠 Начало](../START-HERE.md) · [← 02.6 Вывод, форматирование и комментарии](06-printing-formatting-and-comments.md) · [📋 Этап 02](README.md) · [02.8 Ваши первые тесты →](08-your-first-tests.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
