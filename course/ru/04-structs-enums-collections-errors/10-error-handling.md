<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 04.9 Хеш-таблицы (HashMap)](09-hash-maps.md) · [📋 Этап 04](README.md) · [04.11 Проект: телефонная книга →](11-project-phone-book.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/04-structs-enums-collections-errors/10-error-handling.md) <!-- nav -->

# Урок 04.10 — Обработка ошибок

> **Этап 04 — Структуры, перечисления, коллекции и ошибки** · Урок 10 из 12 · ⏱ около 5 часов (3 занятия)

## Что вы узнаете

- Чем отличается баг (используйте `panic!`) от ожидаемого сбоя (используйте `Result`).
- Как читать и писать `Result<T, E>` и как с ним справляются `unwrap`, `expect`, `match` и вспомогательные методы.
- Как оператор `?` передаёт ошибку вызывающему коду одним символом.
- Как спроектировать собственный тип ошибки с `Display`, `Error` и `From` и когда достаточно `Box<dyn Error>`.
- Как тестировать случаи с ошибками и как программа сообщает о сбое командной оболочке через коды завершения.

## Перед началом

- Вы прошли [Урок 04.9 — Хеш-таблицы (HashMap)](09-hash-maps.md).
- Вы знаете перечисления, `Option` и `match` ([Урок 04.3](03-enums.md), [Урок 04.4](04-option-and-match.md)), `if let` и `let else` ([Урок 04.5](05-if-let-let-else-and-patterns.md)), строки и `parse` ([Урок 04.8](08-strings-in-depth.md)).
- Создайте проект и два небольших файла с данными:

```bash
cd ~/projects/rust
cargo new code/p04-errors
cd code/p04-errors
printf '42\n' > number.txt
printf 'forty-two\n' > bad_number.txt
```

## Новые слова

| Слово | Значение |
|---|---|
| **ошибка** (error) | Что-то, что пошло не так во время работы программы: файла нет, пользователь ввёл текст там, где ждали число, сеть недоступна. |
| **баг** (bug) | Ошибка в самой программе. Неправилен код, а не внешний мир. |
| **исправимая ошибка** (recoverable error) | Ошибка, которую программа может разумно обработать (повторить попытку, показать сообщение, взять значение по умолчанию). В Rust она моделируется через `Result`. |
| **паника** (panic) | Намеренная аварийная остановка в ситуации, из которой программа не может продолжить работу, почти всегда из-за бага. Её запускают `panic!` или неудачный `unwrap`. |
| **раскрутка стека** (unwinding) | Что по умолчанию делает паника: Rust проходит назад по выполняющимся функциям, освобождая значения каждой из них, а затем завершает поток. |
| **трассировка стека** (backtrace) | Список функций, которые выполнялись в момент паники, от самой внутренней к внешним. |
| **`Result<T, E>`** | Перечисление с двумя вариантами: `Ok(T)` хранит успешное значение, `Err(E)` хранит значение ошибки. |
| **`unwrap`** | Метод, который возвращает значение внутри `Ok` / `Some` и паникует, если там `Err` / `None`. |
| **`expect`** | Как `unwrap`, но сообщение паники задаёте вы. |
| **распространять** (propagate) | Передавать ошибку вызывающему коду, а не обрабатывать её там, где она произошла. |
| **оператор `?`** | Короткий способ распространить ошибку: для `Ok(v)` он даёт `v`; для `Err(e)` он возвращает `Err(e)` из текущей функции. |
| **I/O** (Input/Output, ввод/вывод) | Чтение из внешнего мира и запись в него: файлы, клавиатура, сеть. Ввод/вывод — самый частый источник ошибок. |
| **блок `impl`** | Место, где вы пишете методы типа или реализацию трейта для типа (`impl Trait for Type`). |
| **`Display`** | Трейт, стоящий за `{}`. Он решает, как значение выглядит для человека. Вы реализуете его, чтобы дать ошибке сообщение. |
| **трейт `Error`** | Стандартный трейт (`std::error::Error`), который помечает тип как ошибку. |
| **трейт `From`** | Стандартный трейт преобразования. `impl From<A> for B` говорит, как превратить `A` в `B`; его использует оператор `?`. |
| **`Box<dyn Error>`** | Коробка (указатель в куче), которая может хранить любое значение, являющееся `Error`. Универсальный тип ошибки. (`dyn` объясняется в [Уроке 06.1](../06-rust-advanced/01-trait-objects-and-dynamic-dispatch.md).) |
| **охранное условие** (match guard) | Дополнительное условие `if` после образца в ветке `match`: `Ok(n) if n > 0 => ...`. Ветка срабатывает, только когда условие истинно. |
| **stdout / stderr** (standard output / standard error, стандартный вывод / стандартный вывод ошибок) | Два канала вывода программы. Обычные результаты идут в stdout (`println!`), сообщения об ошибках — в stderr (`eprintln!`). |
| **код завершения** (exit code) | Число, которое программа возвращает командной оболочке при завершении. `0` означает успех; любое другое число — сбой. |

## 1. Два вида проблем

Программы дают сбои двумя очень разными способами.

1. **Баг.** Программист ошибся. Индекс вышел за границы, случилось то, что «никогда не
   случается». Исправить это во время работы программы никто не может; правильная реакция —
   громко остановиться, чтобы баг нашли и починили.
2. **Ожидаемый сбой.** Программа верна, но мир — нет: файла не существует, пользователь ввёл
   `abc` там, где нужно число, диск заполнен. Хорошая программа должна **ожидать** такое и
   реагировать: показать сообщение, спросить снова, взять значение по умолчанию.

В Rust для этих двух видов проблем есть два разных инструмента, поэтому вы всегда видите, какой
это случай:

| Ситуация | Инструмент | Результат |
|---|---|---|
| Баг, нарушенное обещание, «невозможное» состояние | `panic!` | программа останавливается (с сообщением) |
| Сбой, который может случиться и в корректном коде | `Result<T, E>` | вызывающий код получает значение и решает сам |
| Значения может просто не быть | `Option<T>` | `Some(v)` или `None` |

Компилятор помогает: функция, которая может дать сбой, говорит об этом **в своём типе**, и вы не
можете воспользоваться успешным значением, не учтя возможность ошибки.

## 2. `panic!`: остановиться при баге

```rust
fn main() {
    println!("start");
    let data: Vec<i32> = Vec::new();
    if data.is_empty() {
        panic!("the data list must never be empty here");
    }
    println!("never printed");
}
```

```text
start

thread 'main' (3973753) panicked at src/main.rs:5:9:
the data list must never be empty here
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
```

**Что происходит:**

1. `panic!("...")` печатает сообщение, имя потока и место в коде (файл, строка, столбец), затем
   запускает **раскрутку стека**: Rust выходит из выполняющихся функций, освобождая значения,
   которыми они владеют.
2. Поскольку это был главный поток, завершается вся программа. Её **код завершения** — `101` (мы
   проверили: `echo $?` печатает `101`). Коды завершения объясняются в разделе 10.
3. `never printed` не появляется: код после паники не выполняется.

Чтобы увидеть, откуда пришла паника, запустите с переменной окружения `RUST_BACKTRACE=1`:
`RUST_BACKTRACE=1 cargo run`. Rust тогда напечатает **трассировку стека** — список вызовов функций,
который привёл к панике; ищите в нём строки с именами ваших файлов.

Многие ошибки вызывают паники, хотя вы не писали `panic!`: индекс за границами
([Урок 04.7](07-vectors.md)), отсутствующий ключ при `map[key]` ([Урок 04.9](09-hash-maps.md)), деление целого числа на ноль,
переполнение целого числа в отладочной сборке ([Урок 02.2](../02-programming-basics/02-data-types.md)).

### `unwrap` и `expect`

`unwrap` и `expect` — это сокращения: «дай мне значение и **запаникуй**, если его нет».

```rust
fn main() {
    let good: i32 = "42".parse().unwrap();
    println!("good = {good}");
    let bad: i32 = "forty-two".parse().unwrap();
    println!("bad = {bad}");
}
```

```text
good = 42

thread 'main' (3973991) panicked at src/main.rs:4:40:
called `Result::unwrap()` on an `Err` value: ParseIntError { kind: InvalidDigit }
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
```

`parse` возвращает `Result` (следующий раздел). `unwrap()` для `Err` паникует и печатает значение
ошибки в отладочной форме. `expect` делает то же самое, но сообщение выбираете вы:

```rust
fn main() {
    let text = "17x";
    let n: i32 = text.parse().expect("the settings file should contain a number");
    println!("{n}");
}
```

```text

thread 'main' (3974235) panicked at src/main.rs:3:31:
the settings file should contain a number: ParseIntError { kind: InvalidDigit }
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
```

> 📏 **Соглашение:** если уж паниковать, предпочитайте `expect` вместо `unwrap` и пишите то, что вы
> *ожидали* увидеть («файл настроек должен содержать число»), а не то, что пошло не так. Сообщение
> поможет тому, кто читает отчёт о падении, — возможно, вам самим через полгода.

Когда паника допустима?

- в **тестах** (паника — это способ, которым тест падает);
- в **небольших примерах и быстрых экспериментах**;
- когда вы **можете доказать**, что значение есть. Например, `"42".parse::<i32>().unwrap()` для
  литерала, который написали сами, или после проверки `is_some()`;
- когда продолжать было бы хуже, чем остановиться (нарушено внутреннее обещание).

> 🔒 **Безопасность:** никогда не вызывайте `unwrap()` для значения, пришедшего снаружи программы:
> ввод пользователя, файл, сеть. Злоумышленник (или просто опечатка) превратит ваш `unwrap` в
> аварию, а падающий сервер — это отказ в обслуживании. Обрабатывайте `Err`.

## 3. `Result<T, E>`: обработать ожидаемый сбой

`Result` — обычное перечисление из стандартной библиотеки (и, как `Option`, всегда доступно без
импорта):

```rust
enum Result<T, E> {
    Ok(T),
    Err(E),
}
```

`T` — тип успешного значения, `E` — тип значения ошибки. (Буквы обозначают типы, которые вы
выбираете; общая возможность называется обобщениями, [Урок 05.1](../05-rust-intermediate/01-generics.md).) Например,
`Result<i32, ParseIntError>` — это «либо `i32`, либо ошибка, которая говорит, почему разбор не
удался». Обрабатывать его можно через `match`, как вы делали с `Option`:

```rust
fn main() {
    for text in ["42", "-7", "forty-two", "", "99999999999"] {
        match text.parse::<i32>() {
            Ok(n) => println!("{:?} -> number {}", text, n),
            Err(e) => println!("{:?} -> error: {}", text, e),
        }
    }
}
```

**Что происходит:**

1. `text.parse::<i32>()` пытается прочитать `i32`. Он возвращает `Result<i32, ParseIntError>`.
2. Ветка `Ok(n)` выполняется при успехе и даёт число. Ветка `Err(e)` выполняется при сбое; `e` — это
   `ParseIntError`, а `{}` печатает понятную человеку причину (тип реализует `Display`).
3. Ошибка не останавливает программу. Каждый из пяти входов обрабатывается по-своему.

```text
"42" -> number 42
"-7" -> number -7
"forty-two" -> error: invalid digit found in string
"" -> error: cannot parse integer from empty string
"99999999999" -> error: number too large to fit in target type
```

### Вспомогательные методы

`match` всегда корректен, но для небольших задач есть методы покороче:

```rust
fn main() {
    let a: Result<i32, String> = Ok(5);
    let b: Result<i32, String> = Err(String::from("broken"));

    println!("{}", a.is_ok());
    println!("{}", b.is_err());
    println!("{}", a.clone().unwrap_or(0));
    println!("{}", b.clone().unwrap_or(0));
    println!("{:?}", a.clone().ok());
    println!("{:?}", b.clone().ok());

    let parsed: Option<i32> = "12".parse().ok();
    println!("{:?}", parsed);

    let n: i32 = "oops".parse().unwrap_or_default();
    println!("{n}");

    let missing: Option<i32> = None;
    let r: Result<i32, String> = missing.ok_or(String::from("no value"));
    println!("{:?}", r);

    let e: Result<i32, String> = "x".parse::<i32>().map_err(|err| format!("bad number: {err}"));
    println!("{:?}", e);
}
```

**Что происходит:**

1. `is_ok()` и `is_err()` отвечают «да» или «нет».
2. `unwrap_or(0)` даёт значение внутри `Ok` или запасное `0` для `Err`. Он никогда не паникует.
   (`.clone()` делает копию, чтобы мы могли снова использовать `a` и `b`; `clone` вы изучили в
   [Уроке 03.3](../03-ownership-and-borrowing/03-move-copy-and-clone.md).)
3. `.ok()` превращает `Result<T, E>` в `Option<T>`: `Ok(v)` становится `Some(v)`, `Err(_)` становится
   `None` (подробности ошибки отбрасываются).
4. `unwrap_or_default()` использует значение типа по умолчанию (`0` для `i32`), когда есть ошибка.
5. `Option::ok_or(err)` работает в обратную сторону: `None` становится `Err(err)`.
6. `map_err` меняет часть с **ошибкой** и не трогает `Ok`. Его аргумент `|err| format!(...)` — это
   *замыкание*: маленькая безымянная функция, записанная прямо на месте, где `err` — её параметр.
   У замыканий будет отдельный урок ([Урок 05.4](../05-rust-intermediate/04-closures.md)); пока читайте это как «возьми ошибку и построй из неё
   более понятное сообщение».

```text
true
true
5
0
Some(5)
None
Some(12)
0
Err("no value")
Err("bad number: invalid digit found in string")
```

> ⚠️ **Частая ошибка:** использовать `.ok()` или `unwrap_or_default()` везде, «чтобы ошибка
> исчезла». Это скрывает настоящие проблемы: программа продолжает работу с неверным значением, и
> никто не знает почему. Отбрасывайте ошибку только тогда, когда вы решили, что она вам
> действительно безразлична.

## 4. Ошибки внешнего мира: файлы

Чтение файла может не удаться по-разному: такого файла нет, нет прав доступа, содержимое не является корректным UTF-8 (кодировка текста из [Урока 04.8](08-strings-in-depth.md)). Функция стандартной библиотеки `std::fs::read_to_string` читает весь файл в `String` и
возвращает `Result<String, std::io::Error>`. `io::Error` — стандартный тип ошибки для проблем
**ввода/вывода**.

Сначала создайте файл `config.txt`: `printf 'port=8080\nname=demo\n' > config.txt`.

```rust
use std::fs;

fn main() {
    match fs::read_to_string("config.txt") {
        Ok(text) => println!("read {} bytes", text.len()),
        Err(e) => println!("cannot read: {e}"),
    }
    match fs::read_to_string("does-not-exist.txt") {
        Ok(text) => println!("read {} bytes", text.len()),
        Err(e) => println!("cannot read: {e}"),
    }
}
```

**Что происходит:**

1. Относительный путь `config.txt` ищется в *текущей директории*: в папке, где вы запустили
   `cargo run` (папка проекта).
2. Первый вызов успешен; в тексте 20 байт (`port=8080` + перевод строки + `name=demo` + перевод
   строки).
3. Второго файла не существует, поэтому вызов возвращает `Err`. `{e}` печатает сообщение
   операционной системы: `No such file or directory (os error 2)`.

```text
read 20 bytes
cannot read: No such file or directory (os error 2)
```

## 5. Оператор `?`

Теперь напишем функцию, которая читает число из файла. Она может дать сбой двумя способами: файл
не читается, и текст не является числом. Только с `match`:

```rust
use std::fs;
use std::io;
use std::num::ParseIntError;

fn read_number(path: &str) -> Result<i32, String> {
    let text = match fs::read_to_string(path) {
        Ok(t) => t,
        Err(e) => return Err(format!("cannot read {path}: {e}")),
    };
    match text.trim().parse::<i32>() {
        Ok(n) => Ok(n),
        Err(e) => Err(format!("{path} does not contain a number: {e}")),
    }
}

fn read_number_with_question_mark(path: &str) -> Result<i32, io::Error> {
    let text = fs::read_to_string(path)?;
    Ok(text.trim().len() as i32)
}

fn parse_twice(a: &str, b: &str) -> Result<i32, ParseIntError> {
    let x: i32 = a.parse()?;
    let y: i32 = b.parse()?;
    Ok(x + y)
}

fn main() {
    println!("{:?}", read_number("number.txt"));
    println!("{:?}", read_number("bad_number.txt"));
    println!("{:?}", read_number("nothing.txt"));
    println!("{:?}", read_number_with_question_mark("number.txt"));
    println!("{:?}", parse_twice("40", "2"));
    println!("{:?}", parse_twice("40", "two"));
}
```

**Что происходит:**

1. `read_number` обрабатывает каждый сбой через `match` и `return Err(...)`. Это работает, но
   длинно. К тому же обе ошибки превращаются в сообщение `String`.
2. `let text = fs::read_to_string(path)?;` — та же идея в один символ. **Оператор `?`** смотрит на
   `Result` слева от себя:
   - если это `Ok(v)`, всё выражение становится `v`, и функция продолжает работу;
   - если это `Err(e)`, функция **немедленно возвращает `Err(e)`**.
3. В `read_number_with_question_mark` собственный тип ошибки функции — `io::Error`, тот же, что и у
   `read_to_string`, поэтому ошибка проходит без изменений. (`.len()` нужен только для того,
   чтобы получить число.)
4. `parse_twice` использует `?` дважды. Если `a` плох, мы возвращаемся, даже не глядя на `b`.
   Успешный путь читается как прямой код, без вложенности.

```text
Ok(42)
Err("bad_number.txt does not contain a number: invalid digit found in string")
Err("cannot read nothing.txt: No such file or directory (os error 2)")
Ok(2)
Ok(42)
Err(ParseIntError { kind: InvalidDigit })
```

Что делает `?`, записанное подробно (упрощённо):

```text
let x = a.parse::<i32>()?;      is roughly:

let x = match a.parse::<i32>() {
    Ok(v)  => v,
    Err(e) => return Err(From::from(e)),   // converts the error type if needed (section 7)
};
```

`?` работает и с `Option` в функции, которая возвращает `Option`: `None` заставляет функцию
вернуть `None`.

### `?` работает только в функциях, которые возвращают `Result` (или `Option`)

```rust
use std::fs;

fn main() {
    let text = fs::read_to_string("config.txt")?;
    println!("{text}");
}
```

```text
error[E0277]: the `?` operator can only be used in a function that returns `Result` or `Option` (or another type that implements `FromResidual`)
 --> src/main.rs:4:48
  |
3 | fn main() {
  | --------- this function should return `Result` or `Option` to accept `?`
4 |     let text = fs::read_to_string("config.txt")?;
  |                                                ^ cannot use the `?` operator in a function that returns `()`
  |
help: consider adding return type
  |
3 ~ fn main() -> Result<(), Box<dyn std::error::Error>> {
```

**Что означает ошибка:** `?` нужно куда-то отправить ошибку: в возвращаемое значение функции.
`main` возвращает `()` (ничего), поэтому отправлять некуда. Компилятор предлагает исправление. В
следующем разделе мы сделаем именно это.

## 6. `main` может возвращать `Result`

```rust
use std::error::Error;
use std::fs;

fn main() -> Result<(), Box<dyn Error>> {
    let text = fs::read_to_string("nums.txt")?;
    let mut total = 0;
    for line in text.lines() {
        let n: i32 = line.trim().parse()?;
        total += n;
    }
    println!("total = {total}");
    Ok(())
}
```

**Что происходит:**

1. `-> Result<(), Box<dyn Error>>` означает: при успехе `main` даёт `()`, при сбое — *какую-нибудь
   ошибку*. `Box<dyn Error>` — универсальный тип ошибки (раздел 8).
2. Теперь могут сработать оба `?`: чтение файла и разбор строки. Разные типы ошибок автоматически
   преобразуются в `Box<dyn Error>`.
3. `Ok(())` в конце — успешное значение: «всё хорошо, возвращать нечего». Обратите внимание: после
   последнего выражения нет точки с запятой.
4. Если `main` возвращает `Err`, Rust печатает `Error: ` и ошибку в отладочной форме и завершает
   программу с кодом `1`.

Попробуйте три разные ситуации. В терминале в папке проекта:

```bash
printf '10\n20\n30\n' > nums.txt
cargo run -q
printf '10\nabc\n' > nums.txt
cargo run -q
rm nums.txt
cargo run -q
```

```text
total = 60
Error: ParseIntError { kind: InvalidDigit }
Error: Os { code: 2, kind: NotFound, message: "No such file or directory" }
```

Второй и третий запуски заканчиваются кодом завершения 1 (его показывает `echo $?`). Сообщения
верны, но не очень дружелюбны (`Error: ParseIntError { kind: InvalidDigit }`). В следующих
разделах мы построим лучшие.

## 7. Собственный тип ошибки

Строки в роли ошибок годятся для крошечных программ, но их нельзя сравнивать, разбирать через
`match` или расширять. Настоящий тип ошибки — это **перечисление** с одним вариантом на каждый вид
сбоя.

```rust
use std::fmt;
use std::num::ParseIntError;

#[derive(Debug)]
enum ConfigError {
    MissingKey(String),
    BadNumber { key: String, source: ParseIntError },
}

impl fmt::Display for ConfigError {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        match self {
            ConfigError::MissingKey(key) => write!(f, "missing key '{key}'"),
            ConfigError::BadNumber { key, source } => {
                write!(f, "key '{key}' is not a number: {source}")
            }
        }
    }
}

impl std::error::Error for ConfigError {}

/// Finds `key=value` in the text and returns the value as a number.
fn get_number(text: &str, key: &str) -> Result<u32, ConfigError> {
    for line in text.lines() {
        if let Some((k, v)) = line.split_once('=') {
            if k.trim() == key {
                return match v.trim().parse::<u32>() {
                    Ok(n) => Ok(n),
                    Err(e) => Err(ConfigError::BadNumber { key: key.to_string(), source: e }),
                };
            }
        }
    }
    Err(ConfigError::MissingKey(key.to_string()))
}

fn main() {
    let text = "port=8080\nname=demo\nworkers=many\n";
    for key in ["port", "workers", "timeout"] {
        match get_number(text, key) {
            Ok(n) => println!("{key} = {n}"),
            Err(e) => println!("error: {e}"),
        }
    }
    println!("{:?}", get_number(text, "timeout"));
}
```

**Что происходит:**

1. У `ConfigError` два варианта. `MissingKey` несёт имя ключа. `BadNumber` несёт ключ и исходную
   `ParseIntError` (чтобы информация не терялась).
2. `impl fmt::Display for ConfigError` даёт ошибке сообщение для человека. `impl Trait for Type`
   означает «этот тип обладает возможностями этого трейта» ([Урок 05.2](../05-rust-intermediate/02-traits.md) научит трейтам как следует; пока
   просто копируйте форму). Метод `fmt` получает `&self` и форматтер `f`; `write!(f, "...")`
   работает как `println!`, но пишет в `f`. `match self` выдаёт свой текст для каждого варианта.
3. `impl std::error::Error for ConfigError {}` имеет пустое тело. Этого достаточно: у `Error`
   есть поведение по умолчанию для всего, но тип обязан быть `Debug` + `Display`. Это позволяет
   использовать ошибку с `Box<dyn Error>` и в других библиотеках.
4. `get_number` возвращает `Result<u32, ConfigError>`: каждый сбой — один из наших двух вариантов.
5. `{e}` в `main` использует `Display` для сообщения пользователю; `{:?}` (последняя строка)
   использует `Debug`, который показывает вариант для разработчика.

```text
port = 8080
error: key 'workers' is not a number: invalid digit found in string
error: missing key 'timeout'
Err(MissingKey("timeout"))
```

### `From` заставляет `?` преобразовывать ошибки

Если у одной функции несколько видов сбоев, хочется, чтобы `?` оборачивал каждый из них в ваше
перечисление ошибок. Для этого реализуйте трейт `From`: `impl From<ParseIntError> for AppError`
говорит Rust, как превратить `ParseIntError` в `AppError`. Оператор `?` вызывает его за вас.

```rust
use std::fmt;
use std::num::ParseIntError;

#[derive(Debug)]
enum AppError {
    BadNumber(ParseIntError),
    Negative(i32),
}

impl fmt::Display for AppError {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        match self {
            AppError::BadNumber(e) => write!(f, "not a number: {e}"),
            AppError::Negative(n) => write!(f, "{n} is negative"),
        }
    }
}

impl std::error::Error for AppError {}

impl From<ParseIntError> for AppError {
    fn from(e: ParseIntError) -> AppError {
        AppError::BadNumber(e)
    }
}

fn parse_positive(text: &str) -> Result<i32, AppError> {
    let n: i32 = text.trim().parse()?;
    if n < 0 {
        return Err(AppError::Negative(n));
    }
    Ok(n)
}

fn main() {
    for text in ["15", " 8 ", "-3", "abc"] {
        match parse_positive(text) {
            Ok(n) => println!("{:?} -> {n}", text),
            Err(e) => println!("{:?} -> error: {e}", text),
        }
    }
}
```

**Что происходит:**

1. `text.trim().parse()?` может завершиться `ParseIntError`, но функция возвращает `AppError`.
   Оператор `?` видит несоответствие и вызывает `AppError::from(e)`, который оборачивает ошибку в
   вариант `BadNumber`.
2. Отрицательное число — *другой* сбой, который создаётся вручную: `return Err(AppError::Negative(n))`.
3. Вызывающий код видит один тип ошибки, `AppError`, что бы ни пошло не так внутри.

```text
"15" -> 15
" 8 " -> 8
"-3" -> error: -3 is negative
"abc" -> error: not a number: invalid digit found in string
```

Если вы забудете реализацию `From`, компилятор точно скажет, чего не хватает:

```rust
use std::num::ParseIntError;

#[derive(Debug)]
enum AppError {
    BadNumber(ParseIntError),
}

fn parse(text: &str) -> Result<i32, AppError> {
    let n = text.parse::<i32>()?;
    Ok(n)
}

fn main() {
    println!("{:?}", parse("5"));
}
```

```text
error[E0277]: `?` couldn't convert the error to `AppError`
 --> src/main.rs:9:32
  |
8 | fn parse(text: &str) -> Result<i32, AppError> {
  |                         --------------------- expected `AppError` because of this
9 |     let n = text.parse::<i32>()?;
  |                  --------------^ the trait `From<ParseIntError>` is not implemented for `AppError`
  |                  |
  |                  this can't be annotated with `?` because it has type `Result<_, ParseIntError>`
  |
note: `AppError` needs to implement `From<ParseIntError>`
```

Исправление — `impl From<ParseIntError> for AppError`, показанный выше.

> 📏 **Соглашение:** называйте типы ошибок по их назначению: `ConfigError`, `ParseError`. Пишите
> сообщения с маленькой буквы и без точки в конце (`missing key 'port'`), чтобы их можно было
> вставлять в более длинные предложения: `could not load config: missing key 'port'`.

## 8. `Box<dyn Error>`: один тип на все случаи

Писать перечисление ошибок и его реализации `From` — это труд. В небольшой программе или в `main`
часто хочется: «любая ошибка подойдёт, просто передай её наверх и напечатай». Для этого есть
`Box<dyn Error>`:

```rust
use std::error::Error;
use std::fs;

fn run() -> Result<(), Box<dyn Error>> {
    let text = fs::read_to_string("number.txt")?;
    let n: i32 = text.trim().parse()?;
    println!("the number is {n}");

    let text2 = fs::read_to_string("bad_number.txt")?;
    let n2: i32 = text2.trim().parse()?;
    println!("the second number is {n2}");
    Ok(())
}

fn main() {
    if let Err(e) = run() {
        println!("error: {e}");
    }
}
```

**Что происходит:**

1. У `run` четыре возможных сбоя с двумя разными типами ошибок (`io::Error` и `ParseIntError`).
   `Box<dyn Error>` может хранить любую из них, а `?` преобразует каждую автоматически.
2. Первое число читается и печатается. Во втором файле лежит `forty-two`, поэтому второй
   `parse()?` падает, и `run` возвращает `Err`.
3. `main` обрабатывает результат через `if let Err(e)` и печатает сообщение через `{e}` (текст
   `Display`), который дружелюбнее отладочной формы.

```text
the number is 42
error: invalid digit found in string
```

`Box<dyn Error>` хорошо подходит для `main`, небольших утилит и прототипов. Цена — вызывающий код не
может легко разобрать через `match` точный вид ошибки. Библиотекам и более крупным программам
стоит использовать собственное перечисление.

> 💡 **Совет:** два популярных крейта убирают большую часть шаблонного кода этого раздела.
> **`thiserror`** генерирует `Display`, `Error` и `From` для вашего перечисления из нескольких
> атрибутов. **`anyhow`** даёт готовый универсальный тип ошибки с сообщениями-контекстом — для
> приложений. Оба вы изучите в [Уроке 05.11 — Практические крейты](../05-rust-intermediate/11-practical-crates-serde-clap-errors.md). Сначала поймите ручной способ, как здесь.

## 9. Что и когда использовать?

| Вопрос | Ответ |
|---|---|
| Вызывающий код дал обещание и нарушил его (баг) | `panic!` (или `assert!`) |
| Может ли это упасть, даже когда мой код корректен (ввод, файлы, сеть)? | возвращайте `Result` |
| Значения может просто не быть, и «нет значения» — не ошибка | возвращайте `Option` |
| Пишете **библиотеку** | своё перечисление ошибок; никогда не паникуйте от неверного ввода |
| Пишете небольшое **приложение** или `main` | `Box<dyn Error>` (позже: `anyhow`) |
| Прототип или тест | `unwrap` / `expect` допустимы |
| Хотите проигнорировать ошибку | решите это явно и с комментарием: `let _ = ...;` |

Rust также предупреждает, когда вы полностью забыли про `Result`:

```rust
fn main() {
    "5".parse::<i32>();
    std::fs::remove_file("whatever.txt");
}
```

```text
warning: unused `Result` that must be used
 --> src/main.rs:2:5
  |
2 |     "5".parse::<i32>();
  |     ^^^^^^^^^^^^^^^^^^
  |
  = note: this `Result` may be an `Err` variant, which should be handled
  = note: `#[warn(unused_must_use)]` (part of `#[warn(unused)]`) on by default
help: use `let _ = ...` to ignore the resulting value
```

(Мы сократили сообщение. Такое предупреждение выдаётся для каждой строки.) Никогда не оставляйте
это предупреждение в коде: оно означает, что ошибка может быть молча потеряна.

## 10. Сообщение о сбое: stderr и коды завершения

Программа командной строки общается с двумя адресатами: с человеком в терминале и с другими
программами или скриптами. Два соглашения делают программу «хорошим гражданином»:

- **Сообщения идут в правильный поток.** `println!` пишет в **stdout**, `eprintln!` пишет в
  **stderr**. Результаты идут в stdout, чтобы их можно было передать по конвейеру другой
  программе; ошибки и предупреждения идут в stderr, чтобы не смешиваться с результатами.
- **Код завершения отличает успех от сбоя.** `0` — успех, всё остальное — сбой. Его проверяют
  shell-скрипты, инструменты сборки и серверы CI (Continuous Integration, непрерывная интеграция). `std::process::exit(code)`
  завершает программу с заданным кодом.

```rust
use std::process;

fn run(arg: &str) -> Result<u32, String> {
    arg.parse::<u32>().map_err(|e| format!("'{arg}' is not a valid count: {e}"))
}

fn main() {
    match run("ten") {
        Ok(n) => println!("count = {n}"),
        Err(message) => {
            eprintln!("error: {message}");
            process::exit(2);
        }
    }
}
```

**Что происходит:**

1. `run("ten")` завершается ошибкой, поэтому выполняется ветка `Err`.
2. `eprintln!` пишет сообщение в stderr.
3. `process::exit(2)` немедленно останавливает программу с кодом завершения `2` (мы проверили:
   `echo $?` печатает `2`). Код после него не выполняется, а живые значения не освобождаются,
   поэтому используйте его только в `main`, в самом конце.

```text
error: 'ten' is not a valid count: invalid digit found in string
```

> 📏 **Соглашение:** код завершения `0` — успех, `1` — общий сбой, `2` — неверное использование
> (плохие аргументы). `fn main() -> Result<...>` даёт вам `0` и `1` автоматически.

> 🔒 **Безопасность:** сообщения об ошибках — это информация. Показывайте пользователям короткое
> полезное сообщение (`cannot open the settings file`). Не печатайте секреты (пароли, токены),
> внутренние пути к файлам или полные отладочные дампы тем, кто не должен их видеть; подробности
> лучше записывать в приватный журнал.

## 11. Тестирование ошибок

> 🧪 **Тестирование:** ветки с ошибками — тоже код, и ломаются они чаще всего потому, что их никто
> не запускает. Проверяйте хороший случай **и** каждый способ сбоя. Полезные инструменты:
> `assert_eq!(f(x), Err(...))` (типу ошибки нужны `PartialEq` и `Debug`), `assert!(f(x).is_err())`,
> `matches!(value, pattern)` (истинно, когда значение подходит под образец), `#[should_panic]` для
> функций, которые обязаны паниковать, и тестовые функции, возвращающие `Result`.

```rust
fn parse_age(text: &str) -> Result<u8, String> {
    match text.trim().parse::<u8>() {
        Ok(age) if age <= 150 => Ok(age),
        Ok(age) => Err(format!("{age} is not a realistic age")),
        Err(e) => Err(format!("invalid age: {e}")),
    }
}

fn main() {
    println!("{:?}", parse_age("30"));
    println!("{:?}", parse_age("200"));
    println!("{:?}", parse_age("-1"));
    println!("{:?}", parse_age("abc"));
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn accepts_a_normal_age() {
        assert_eq!(parse_age("30"), Ok(30));
    }

    #[test]
    fn rejects_unrealistic_age() {
        assert_eq!(parse_age("200"), Err(String::from("200 is not a realistic age")));
    }

    #[test]
    fn rejects_text() {
        assert!(parse_age("abc").is_err());
    }

    #[test]
    fn matches_macro() {
        assert!(matches!(parse_age("-1"), Err(_)));
    }

    #[test]
    #[should_panic(expected = "invalid age")]
    fn unwrap_on_error_panics() {
        parse_age("x").unwrap();
    }

    #[test]
    fn test_can_return_a_result() -> Result<(), String> {
        let age = parse_age(" 41 ")?;
        assert_eq!(age, 41);
        Ok(())
    }
}
```

**Что происходит:**

1. `Ok(age) if age <= 150 => ...` — это **охранное условие** (match guard): ветка срабатывает, только
   если истинно дополнительное условие после `if`. Тип `u8` хранит от 0 до 255, поэтому `200`
   разбирается нормально, но не проходит условие и попадает во вторую ветку. `-1` не помещается в
   `u8`, поэтому это ошибка разбора.
2. `assert_eq!(parse_age("30"), Ok(30))` сравнивает значения `Result` целиком; это работает, потому
   что `Result<u8, String>` реализует `PartialEq` и `Debug`.
3. `#[should_panic(expected = "invalid age")]` делает так, что тест проходит, только если функция
   паникует и сообщение паники содержит этот текст. Здесь `unwrap()` для `Err` паникует, и
   сообщение включает наш текст.
4. Тест может возвращать `Result<(), E>`. Тогда внутри теста работает `?`, а `Err` проваливает
   тест.

```text
Ok(30)
Err("200 is not a realistic age")
Err("invalid age: invalid digit found in string")
Err("invalid age: invalid digit found in string")
```

```text
running 6 tests
test tests::accepts_a_normal_age ... ok
test tests::rejects_text ... ok
test tests::test_can_return_a_result ... ok
test tests::matches_macro ... ok
test tests::rejects_unrealistic_age ... ok
test tests::unwrap_on_error_panics - should panic ... ok

test result: ok. 6 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

## Частые ошибки

**1. `unwrap()` для пользовательского ввода или файла.** Он компилируется, работает в вашем тесте и
падает в реальной работе. Обрабатывайте `Err` через `match`, `?` или запасное значение (раздел 2).

**2. `?` в функции, которая возвращает `()`.** Ошибка E0277 из раздела 5. Измените возвращаемый тип
на `Result<(), Box<dyn Error>>` (или другой `Result`) и завершите функцию строкой `Ok(())`.

**3. Забыли `Ok(())` в конце функции, возвращающей `Result<(), E>`.**

```rust
fn show(n: i32) -> Result<(), String> {
    println!("{n}");
}
```

```text
error[E0308]: mismatched types
 --> src/main.rs:1:20
  |
1 | fn show(n: i32) -> Result<(), String> {
  |    ----            ^^^^^^^^^^^^^^^^^^ expected `Result<(), String>`, found `()`
  |    |
  |    implicitly returns `()` as its body has no tail or `return` expression
```

Тело заканчивается без значения, а функция должна вернуть `Result`. Исправление: добавьте `Ok(())`
последней строкой.

**4. Присвоили `Result` обычной переменной.**

```rust
fn main() {
    let n: i32 = "5".parse();
    println!("{n}");
}
```

```text
error[E0308]: mismatched types
 --> src/main.rs:2:18
  |
2 |     let n: i32 = "5".parse();
  |            ---   ^^^^^^^^^^^ expected `i32`, found `Result<_, _>`
  |            |
  |            expected due to this
  |
help: consider using `Result::expect` to unwrap the `Result<_, _>` value, panicking if the value is a `Result::Err`
```

`parse` возвращает `Result`, а не просто число. Обработайте его через `match`, `?` или (когда
допустимо) `unwrap` / `expect`.

**5. Разные типы ошибок без реализации `From`.** Ошибка E0277 из раздела 7. Реализуйте
`From<ЧужаяОшибка> for ВашаОшибка` или используйте `Box<dyn Error>`.

**6. Выбрасывание информации.** Запись `Err(_) => println!("error")` или `.ok()` теряет причину.
Печатайте или возвращайте значение ошибки.

## Попробуйте сами

Проделайте эксперимент из раздела 6 сами, а затем расширьте его:

1. Замените `src/main.rs` программой `main` из раздела 6 (той, что с `nums.txt`). Создайте файл и
   запустите:

```bash
printf '10\n20\n30\n' > nums.txt
cargo run -q
```

Ожидаемый вывод:

```text
total = 60
```

2. Поместите в `nums.txt` плохую строку и запустите снова. Затем проверьте код завершения:

```bash
printf '10\nabc\n' > nums.txt
cargo run -q
echo $?
```

Ожидаемый вывод:

```text
Error: ParseIntError { kind: InvalidDigit }
1
```

3. Сделайте сбой дружелюбнее. Превратите `main` в функцию `run()`, возвращающую
   `Result<(), Box<dyn Error>>`, и напишите новый `main`, как в разделе 8, который печатает
   `error: {e}` в stderr через `eprintln!` и вызывает `std::process::exit(1)`.
4. Удалите `nums.txt` и запустите снова. Что говорит новое сообщение?
5. Сохраните работу: `git add .` и `git commit -m "Practice error handling with Result and ?"`.
   (Не коммитьте `nums.txt`: если Git его показывает, добавьте его в `.gitignore`, как вы учились в
   Этапе 01.)

## Упражнения

### Упражнение 1 — Безопасное деление (легко)

Напишите `fn divide(a: i32, b: i32) -> Result<i32, String>`, которая возвращает
`Err("division by zero")`, когда `b` равно `0`. Вызовите её с `(10, 2)`, `(1, 0)` и напечатайте
результат третьего вызова (`7, 0`) через `match`, который говорит `result: …` или `error: …`.
Ожидаемый вывод:

```text
Ok(5)
Err("division by zero")
error: division by zero
```

<details>
<summary>Подсказка</summary>

Сначала проверьте `b == 0` и сделайте `return Err(String::from("division by zero"));`. Иначе
верните `Ok(a / b)`. В `main` первые два результата печатайте через `{:?}`.

</details>

<details>
<summary>Ответ</summary>

```rust
fn divide(a: i32, b: i32) -> Result<i32, String> {
    if b == 0 {
        return Err(String::from("division by zero"));
    }
    Ok(a / b)
}

fn main() {
    println!("{:?}", divide(10, 2));
    println!("{:?}", divide(1, 0));
    match divide(7, 0) {
        Ok(n) => println!("result: {n}"),
        Err(e) => println!("error: {e}"),
    }
}
```

Почему это работает: проверка выполняется до деления, которое запаниковало бы, а вызывающий код
сам решает, как реагировать на `Err`.

</details>

### Упражнение 2 — Сложить два текста, с тестами (легко)

Напишите `fn add_texts(a: &str, b: &str) -> Result<i32, ParseIntError>`, которая разбирает оба
текста (игнорируя пробелы вокруг них) и возвращает сумму. Используйте `?`. Напишите три теста:
успех, плохой первый аргумент и плохой второй. (`ParseIntError` находится в `std::num`.)

<details>
<summary>Подсказка</summary>

`let x: i32 = a.trim().parse()?;`, то же самое для `b`, затем `Ok(x + y)`. Для успеха используйте
`assert_eq!(…, Ok(42))`, для сбоев — `.is_err()`.

</details>

<details>
<summary>Ответ</summary>

```rust
use std::num::ParseIntError;

fn add_texts(a: &str, b: &str) -> Result<i32, ParseIntError> {
    let x: i32 = a.trim().parse()?;
    let y: i32 = b.trim().parse()?;
    Ok(x + y)
}

fn main() {
    println!("{:?}", add_texts("40", " 2 "));
    println!("{:?}", add_texts("40", "two"));
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn adds_numbers() {
        assert_eq!(add_texts("40", " 2 "), Ok(42));
    }

    #[test]
    fn first_is_bad() {
        assert!(add_texts("x", "2").is_err());
    }

    #[test]
    fn second_is_bad() {
        assert!(add_texts("1", "").is_err());
    }
}
```

```text
Ok(42)
Err(ParseIntError { kind: InvalidDigit })
```

Почему это работает: каждый `?` досрочно возвращает `ParseIntError`, если его разбор не удался;
иначе он достаёт число.

</details>

### Упражнение 3 — Ошибки из файла (средне)

Напишите `fn double_from_file(path: &str) -> Result<i32, Box<dyn Error>>`, которая читает файл,
разбирает его текст (после `trim`) как `i32` и возвращает удвоенное число. В `main` вызовите её
для `number.txt`, `bad_number.txt` и `missing.txt` (последнего нет) и напечатайте `path: result`
или `path: error: message`. Ожидаемый вывод:

```text
number.txt: 84
bad_number.txt: error: invalid digit found in string
missing.txt: error: No such file or directory (os error 2)
```

<details>
<summary>Подсказка</summary>

`fs::read_to_string(path)?`, затем `text.trim().parse()?` в `i32`. Импортируйте `std::error::Error`
и `std::fs`. Запускайте программу из папки проекта, где лежат два файла с данными.

</details>

<details>
<summary>Ответ</summary>

```rust
use std::error::Error;
use std::fs;

fn double_from_file(path: &str) -> Result<i32, Box<dyn Error>> {
    let text = fs::read_to_string(path)?;
    let n: i32 = text.trim().parse()?;
    Ok(n * 2)
}

fn main() {
    for path in ["number.txt", "bad_number.txt", "missing.txt"] {
        match double_from_file(path) {
            Ok(n) => println!("{path}: {n}"),
            Err(e) => println!("{path}: error: {e}"),
        }
    }
}
```

Почему это работает: `Box<dyn Error>` принимает и `io::Error`, и `ParseIntError`, а `?`
преобразует их автоматически.

</details>

### Упражнение 4 — Сумма строк с номером строки (средне)

Напишите `fn sum_lines(text: &str) -> Result<i64, LineError>`. В каждой строке одно целое число;
пустые строки пропускаются. На первой плохой строке верните `LineError { line, problem }` с
номером строки (счёт с 1) и `ParseIntError`. Реализуйте `Display` для `LineError`, чтобы он печатал
`line 3: invalid digit found in string`. Напишите три теста: хороший текст, пустой текст и плохая
строка (проверьте `err.line`).

<details>
<summary>Подсказка</summary>

Используйте `for (index, line) in text.lines().enumerate()`. Пропускайте, когда
`line.trim().is_empty()`. При `Err(e)` от `parse::<i64>()` возвращайте
`Err(LineError { line: index + 1, problem: e })`. Выведите `Debug` и `PartialEq` для структуры,
чтобы тесты могли сравнивать. В тестах `unwrap_err()` возвращает ошибку из `Err` (и паникует на
`Ok`).

</details>

<details>
<summary>Ответ</summary>

```rust
use std::fmt;
use std::num::ParseIntError;

#[derive(Debug, PartialEq)]
struct LineError {
    line: usize,
    problem: ParseIntError,
}

impl fmt::Display for LineError {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        write!(f, "line {}: {}", self.line, self.problem)
    }
}

/// Adds up one integer per line. Empty lines are skipped.
/// On the first bad line, returns its number (counting from 1).
fn sum_lines(text: &str) -> Result<i64, LineError> {
    let mut total = 0;
    for (index, line) in text.lines().enumerate() {
        let line = line.trim();
        if line.is_empty() {
            continue;
        }
        match line.parse::<i64>() {
            Ok(n) => total += n,
            Err(e) => return Err(LineError { line: index + 1, problem: e }),
        }
    }
    Ok(total)
}

fn main() {
    println!("{:?}", sum_lines("10\n20\n\n30\n"));
    match sum_lines("10\n20\nthirty\n40\n") {
        Ok(total) => println!("total {total}"),
        Err(e) => println!("error: {e}"),
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn sums_good_lines() {
        assert_eq!(sum_lines("10\n20\n\n30\n"), Ok(60));
    }

    #[test]
    fn empty_text_is_zero() {
        assert_eq!(sum_lines(""), Ok(0));
    }

    #[test]
    fn reports_line_number() {
        let err = sum_lines("1\n2\nx\n4").unwrap_err();
        assert_eq!(err.line, 3);
    }
}
```

```text
Ok(60)
error: line 3: invalid digit found in string
```

Почему это работает: ошибка — это структура, которая несёт дополнительную информацию (номер
строки) рядом с исходной причиной. Функция принимает `&str`, а не имя файла, поэтому тестам не
нужны файлы; позже можно добавить небольшую обёртку, которая читает файл и вызывает `sum_lines`.

</details>

### Упражнение 5 — Разбор температуры со своим типом ошибки (сложно)

Напишите `fn parse_temperature(text: &str) -> Result<f64, TempError>` для входов вроде `"21.5C"`
или `"212F"`. Она возвращает градусы Цельсия: `C` оставляет число, `F` преобразует по формуле
`(value - 32) * 5 / 9`. Определите перечисление `TempError` с тремя вариантами: `Empty`,
`UnknownUnit(char)` и `BadNumber(ParseFloatError)`. Реализуйте `Display`, `Error` и
`From<ParseFloatError>`, чтобы работал `?`. Напишите тесты: Цельсий, Фаренгейт (`212F` — это
`100.0`), пустой ввод, неизвестная единица (`30K`) и плохое число (`abcC`). Ожидаемый вывод для
пяти образцов:

```text
"21.5C" -> 21.5 C
"212F" -> 100.0 C
"" -> error: the input is empty
"30K" -> error: unknown unit 'K' (use C or F)
"abcC" -> error: bad number: invalid float literal
```

<details>
<summary>Подсказка</summary>

Сделайте `trim` текста. `text.chars().last()` даёт единицу как `Option<char>`: используйте
`let Some(unit) = ... else { return Err(TempError::Empty); };`. Часть с числом —
`&text[..text.len() - unit.len_utf8()]`; разберите её через `parse::<f64>()?` (для этого нужна
реализация `From`). Затем `match unit { 'C' => ..., 'F' => ..., other => Err(TempError::UnknownUnit(other)) }`.
Выведите `Debug` и `PartialEq` для перечисления; `ParseFloatError` поддерживает оба. В строке
формата используйте `{:.1}`, чтобы напечатать один знак после запятой.

</details>

<details>
<summary>Ответ</summary>

```rust
use std::fmt;
use std::num::ParseFloatError;

#[derive(Debug, PartialEq)]
enum TempError {
    Empty,
    UnknownUnit(char),
    BadNumber(ParseFloatError),
}

impl fmt::Display for TempError {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        match self {
            TempError::Empty => write!(f, "the input is empty"),
            TempError::UnknownUnit(c) => write!(f, "unknown unit '{c}' (use C or F)"),
            TempError::BadNumber(e) => write!(f, "bad number: {e}"),
        }
    }
}

impl std::error::Error for TempError {}

impl From<ParseFloatError> for TempError {
    fn from(e: ParseFloatError) -> TempError {
        TempError::BadNumber(e)
    }
}

/// "21.5C" -> 21.5, "212F" -> 100.0 (always returns Celsius)
fn parse_temperature(text: &str) -> Result<f64, TempError> {
    let text = text.trim();
    let Some(unit) = text.chars().last() else {
        return Err(TempError::Empty);
    };
    let number_part = &text[..text.len() - unit.len_utf8()];
    let value: f64 = number_part.trim().parse()?;
    match unit {
        'C' => Ok(value),
        'F' => Ok((value - 32.0) * 5.0 / 9.0),
        other => Err(TempError::UnknownUnit(other)),
    }
}

fn main() {
    for text in ["21.5C", "212F", "", "30K", "abcC"] {
        match parse_temperature(text) {
            Ok(c) => println!("{:?} -> {:.1} C", text, c),
            Err(e) => println!("{:?} -> error: {}", text, e),
        }
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn celsius() {
        assert_eq!(parse_temperature("21.5C"), Ok(21.5));
    }

    #[test]
    fn fahrenheit_boiling_point() {
        assert_eq!(parse_temperature("212F"), Ok(100.0));
    }

    #[test]
    fn empty_input() {
        assert_eq!(parse_temperature("  "), Err(TempError::Empty));
    }

    #[test]
    fn unknown_unit() {
        assert_eq!(parse_temperature("30K"), Err(TempError::UnknownUnit('K')));
    }

    #[test]
    fn bad_number() {
        assert!(matches!(parse_temperature("abcC"), Err(TempError::BadNumber(_))));
    }
}
```

Почему это работает: у каждого сбоя свой вариант, поэтому тесты (и вызывающий код) могут их
различать. Реализация `From` позволяет `?` обернуть ошибку разбора числа с плавающей точкой.
Срез по `text.len() - unit.len_utf8()` безопасен, потому что мы отрезаем ровно один целый символ
с конца ([Урок 04.8](08-strings-in-depth.md)). Сравнение чисел с плавающей точкой через `==` здесь работает, потому что
результаты точные (21.5 и 100.0 в этом вычислении представимы точно); вообще же сравнивайте числа
с плавающей точкой с небольшим допуском.

</details>

## Вопросы для самопроверки

1. Когда использовать `panic!`, а когда `Result`?

<details>
<summary>Ответ</summary>

`panic!` — для багов и нарушенных обещаний, ситуаций, из которых программа не может осмысленно
продолжить. `Result` — для сбоев, которые случаются и в корректном коде (плохой ввод, нет файла,
ошибка сети), чтобы вызывающий код мог решить, что делать.

</details>

2. Что делает `unwrap()` для `Err(e)`? А для `Ok(v)`?

<details>
<summary>Ответ</summary>

Для `Ok(v)` он возвращает `v`. Для `Err(e)` он паникует, печатая значение ошибки.
`expect("msg")` делает то же самое, но с вашим сообщением.

</details>

3. Что делает оператор `?`?

<details>
<summary>Ответ</summary>

Он смотрит на `Result`. Для `Ok(v)` выражение становится `v`. Для `Err(e)` текущая функция
немедленно возвращает `Err(e)` (преобразуя ошибку через `From`, если типы различаются). Он
работает только в функциях, которые возвращают `Result` (или `Option`).

</details>

4. Почему это не компилируется: `fn main() { let t = std::fs::read_to_string("a.txt")?; }`?

<details>
<summary>Ответ</summary>

`main` возвращает `()`, а `?` требует, чтобы функция возвращала `Result` или `Option`, — ему нужно
куда отправить ошибку. Измените сигнатуру на `fn main() -> Result<(), Box<dyn Error>>` и
завершите функцию строкой `Ok(())`.

</details>

5. Что нужно написать, чтобы `?` мог преобразовать `ParseIntError` в вашу `AppError`?

<details>
<summary>Ответ</summary>

Реализацию `From<ParseIntError> for AppError`: её функция `from` оборачивает ошибку в один из
вариантов перечисления.

</details>

6. Для чего хорош `Box<dyn Error>` и в чём его недостаток?

<details>
<summary>Ответ</summary>

Это универсальный тип ошибки: в него можно положить любую ошибку, а `?` преобразует её
автоматически. Это удобно для `main`, небольших утилит и прототипов. Недостаток: вызывающий код
не может легко разобрать через `match` точный вид ошибки, поэтому библиотекам стоит использовать
собственные перечисления ошибок.

</details>

7. В какой поток нужно писать сообщение об ошибке и какой код завершения означает успех?

<details>
<summary>Ответ</summary>

Ошибки идут в stderr (`eprintln!`), результаты — в stdout (`println!`). Код завершения `0`
означает успех; любое другое число — сбой.

</details>

8. Как проверить в тесте, что функция возвращает ошибку?

<details>
<summary>Ответ</summary>

`assert!(f(x).is_err())`, `assert_eq!(f(x), Err(expected))` (нужны `PartialEq` и `Debug` у типа
ошибки) или `assert!(matches!(f(x), Err(_)))`. `#[should_panic]` используйте только для функций,
которые должны паниковать.

</details>

## Итоги

- Баг останавливает программу через `panic!`; ожидаемый сбой возвращается как `Result<T, E>`.
- `unwrap` и `expect` паникуют на `Err`. Используйте их в тестах и для доказуемых случаев, но
  никогда — для внешних данных.
- `match` обрабатывает все случаи. Вспомогательные методы `unwrap_or`, `ok`, `ok_or` и `map_err`
  покрывают небольшие задачи.
- Оператор `?` досрочно возвращает ошибку (и преобразует её через `From`); ему нужна функция,
  возвращающая `Result` или `Option`.
- `main` может возвращать `Result<(), Box<dyn Error>>`. Возвращённый `Err` печатает `Error: …` и
  завершает программу с кодом `1`.
- Хороший тип ошибки — перечисление с `Debug`, `Display` и `Error` плюс реализации `From` для
  ошибок, которые оно оборачивает.
- Библиотеки используют собственные перечисления; небольшие приложения могут использовать
  `Box<dyn Error>` (позже: `thiserror` и `anyhow`).
- Отправляйте ошибки в stderr, используйте код завершения `0` для успеха, никогда не печатайте
  секреты в сообщениях и тестируйте случаи сбоев так же тщательно, как случаи успеха.

## Источники и что почитать

- [The Rust Programming Language — 9. Error Handling](https://doc.rust-lang.org/book/ch09-00-error-handling.html) — глава официальной книги (panic, `Result`, `?` и когда что использовать).
- [std::result::Result](https://doc.rust-lang.org/std/result/index.html) — все методы `Result` с примерами.
- [std::error::Error](https://doc.rust-lang.org/std/error/trait.Error.html) — трейт `Error`.
- [Rust by Example — Error handling](https://doc.rust-lang.org/rust-by-example/error.html) — много коротких примеров, включая собственные ошибки и `Box<dyn Error>`.
- [std::process::exit](https://doc.rust-lang.org/std/process/fn.exit.html) — коды завершения и то, что `exit` не освобождает.
- [The Rust API Guidelines — Error types](https://rust-lang.github.io/api-guidelines/interoperability.html) — как выглядит хороший тип ошибки для библиотек.

---

[🏠 Начало](../START-HERE.md) · [← 04.9 Хеш-таблицы (HashMap)](09-hash-maps.md) · [📋 Этап 04](README.md) · [04.11 Проект: телефонная книга →](11-project-phone-book.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
