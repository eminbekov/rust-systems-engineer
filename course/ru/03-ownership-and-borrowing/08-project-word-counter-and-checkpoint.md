<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 03.7 Исправляем частые ошибки владения](07-fixing-common-ownership-errors.md) · [📋 Этап 03](README.md) · [Этап 04 — Структуры, перечисления, коллекции и ошибки →](../04-structs-enums-collections-errors/README.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/03-ownership-and-borrowing/08-project-word-counter-and-checkpoint.md) <!-- nav -->

# Урок 03.8 — Проект: счётчик слов и контрольная точка

> **Этап 03 — Владение и заимствование** · Урок 8 из 8 · ⏱ около 3 часов (2 занятия)

## Что вы узнаете

- Как построить законченный небольшой инструмент командной строки, где владение и заимствование делают настоящую работу: один владелец текста и много функций, которые его только заимствуют.
- Как читать файл или стандартный ввод, использовать аргументы командной строки, выводить ошибки в нужное место и возвращать код завершения.
- Как работать по вехам с тестами, `cargo fmt`, `cargo clippy`, README и **одним коммитом Git на каждую веху** в отдельной ветке.
- Готовы ли вы к Этапу 04 (контрольная точка в конце урока).

## Перед началом

- Вы прошли [Урок 03.7 — Исправляем частые ошибки владения](07-fixing-common-ownership-errors.md).
- Ваша папка курса `~/projects/rust` — репозиторий Git (Этап 01), и `git status` там работает. В её `.gitignore` должна быть строка `target/`, чтобы папки сборки Cargo не попадали в коммиты ([Урок 01.3](../01-git-and-version-control/03-your-first-repository-and-commits.md)).
- Держите под рукой этот пример текста. На вехе M1 вы сохраните его как `sample.txt` внутри папки проекта (пока не создавайте папку: `cargo new` создаёт её сам). Файл должен содержать ровно эти пять строк (четвёртая пустая):

```text
Rust is a systems programming language.
Rust gives you control over memory, and the compiler checks your work.
Ownership is the wonderful idea behind it all.

Learn it slowly. Rust rewards patience.
```

## Новые слова

| Слово | Значение |
|---|---|
| **CLI** (Command-Line Interface, интерфейс командной строки) | Программа, которой пользуются, набирая её имя и аргументы в терминале. |
| **аргумент** (argument) | Слово, которое вы набираете после имени программы, например `sample.txt` в `word-counter sample.txt`. |
| **стандартный ввод** (*stdin*) | Поток данных, поступающих в программу. По умолчанию это клавиатура; с конвейером он приходит из другой программы или файла. |
| **стандартный вывод** (*stdout*) | Поток, куда программа печатает обычные результаты. Сюда пишет `println!`. |
| **стандартный поток ошибок** (*stderr*) | Второй поток вывода для сообщений об ошибках. Сюда пишет `eprintln!`. Он отделяет ошибки от результатов. |
| **EOF** (End Of File, конец файла) | Сигнал, что ввода больше нет. В терминале его посылают клавишами **Ctrl-D**. |
| **конвейер** (pipe, `\|`) | Возможность оболочки (shell) направлять вывод одной программы на ввод другой: `echo hi \| my-program`. |
| **код завершения** (exit code) | Число, которое программа возвращает системе при завершении. `0` означает успех, любое другое число — неудачу. |
| **`Option`** | Значение, которое `Some(value)` или `None`. Используется, когда значения может не быть. |
| **`Result`** | Значение, которое `Ok(value)` или `Err(error)`. Используется, когда операция может завершиться неудачей. |
| **`match`** | Способ выполнить разный код для каждой возможной формы значения, например `Some` / `None` или `Ok` / `Err`. |
| **веха** (milestone) | Небольшой шаг проекта со своей целью и способом проверить, что он выполнен. |
| **приёмочный тест** (acceptance test) | Проверка, показывающая, что готовая работа ведёт себя так, как сказано в контракте. Здесь: команды с ожидаемым выводом. |
| **`wc`** | Классическая программа Unix («word count»), печатающая число строк, слов и байтов. Наш инструмент — её маленький родственник. |

## 1. Новые инструменты для этого проекта

Проекту нужны несколько вещей, которые вы ещё не использовали. Каждая небольшая. Прочтите этот раздел один раз, а потом пользуйтесь им как справочником.

### Аргументы командной строки

Когда вы запускаете `cargo run -- sample.txt`, всё после `--` передаётся **вашей** программе. (`--` говорит Cargo: «перестань читать параметры, остальное — для программы».) Вы получаете их через `std::env::args()`:

```rust
use std::env;

fn main() {
    let args: Vec<String> = env::args().collect();
    println!("all arguments: {:?}", args);
    println!("number of arguments: {}", args.len());
    println!("argument 1: {:?}", args.get(1));
    println!("argument 2: {:?}", args.get(2));

    match args.get(1) {
        Some(path) => println!("you gave me the path '{path}'"),
        None => println!("you gave me no path"),
    }
}
```

**Что происходит:**

1. `use std::env;` делает доступным имя `env` (`use` вносит в файл имя из стандартной библиотеки).
2. `env::args()` выдаёт аргументы по одному. `.collect()` собирает их в `Vec<String>` (итераторы и `collect` объясняет Урок 05.5). **Первый элемент — всегда путь самой программы**, поэтому первый аргумент пользователя лежит по индексу 1.
3. `args.get(1)` возвращает **`Option`**: `Some(&String)`, если по индексу 1 есть элемент, и `None`, если нет. Он никогда не паникует (в отличие от `args[1]`).
4. У `match` по одной ветке на каждый случай: `Some(path)` выполняется, когда значение есть, и даёт ему имя `path`; `None` выполняется в остальных случаях. Компилятор проверяет, что вы рассмотрели оба случая.

```text
$ cargo run -q -- sample.txt
all arguments: ["target/debug/p03-word-counter", "sample.txt"]
number of arguments: 2
argument 1: Some("sample.txt")
argument 2: None
you gave me the path 'sample.txt'
$ cargo run -q
all arguments: ["target/debug/p03-word-counter"]
number of arguments: 1
argument 1: None
argument 2: None
you gave me no path
```

(`-q` означает «тихо» (quiet): Cargo ничего не печатает от себя. Имя программы в первом элементе зависит от имени вашего проекта.)

### Чтение всего файла

`std::fs::read_to_string(path)` читает файл в `String`. Чтение может не удаться (нет такого файла, нет прав), поэтому функция возвращает **`Result`**, который равен либо `Ok(the_string)`, либо `Err(the_error)`. Вы разбираете его через `match`:

```rust
use std::fs;

fn main() {
    match fs::read_to_string("sample.txt") {
        Ok(text) => println!("read {} bytes", text.len()),
        Err(error) => println!("could not read: {error}"),
    }
    match fs::read_to_string("missing.txt") {
        Ok(text) => println!("read {} bytes", text.len()),
        Err(error) => println!("could not read: {error}"),
    }
}
```

```text
read 199 bytes
could not read: No such file or directory (os error 2)
```

Тот, кто получает `Ok(text)`, становится **владельцем** `String`. Это ровно правило «один владелец»: с этого момента у текста в вашей программе один владелец.

### Чтение стандартного ввода

`std::io::read_to_string(std::io::stdin())` читает всё из стандартного ввода до EOF:

```rust
use std::io;

fn main() {
    let text = io::read_to_string(io::stdin()).expect("could not read standard input");
    println!("read {} bytes", text.len());
}
```

```text
$ printf 'one\ntwo\n' | cargo run -q
read 8 bytes
$ cargo run -q < sample.txt
read 199 bytes
```

`expect("message")` означает «если это `Err`, остановить программу с этим сообщением». В быстрой демонстрации это нормально; в проекте вы будете печатать понятную ошибку. В интерактивном терминале программа ждёт, пока вы наберёте текст; нажмите **Ctrl-D** на пустой строке, чтобы сказать «всё».

### Ошибки идут в stderr; у неудачи есть код завершения

```rust
use std::process;

fn main() {
    println!("this goes to standard output");
    eprintln!("this goes to standard error");
    process::exit(3);
}
```

```text
$ cargo run -q
this goes to standard output
this goes to standard error
$ echo $?
3
$ cargo run -q 2>/dev/null
this goes to standard output
```

`eprintln!` печатает в stderr. Переменная оболочки `$?` содержит код завершения последней команды. `2>/dev/null` выбрасывает stderr: остаётся только обычный вывод. Хороший инструмент печатает результаты в stdout, проблемы — в stderr и завершается ненулевым кодом при неудаче.

> 📏 **Соглашение:** эти три привычки делают инструмент командной строки пригодным для скриптов. Программы, которые печатают ошибки в stdout или возвращают `0` после неудачи, неожиданны и трудны для автоматизации.

## Частые ошибки

**1. Использовать `args[1]` вместо `args.get(1)`.** Если пользователь не указал аргумент, `args[1]` паникует. `get` возвращает `Option`, который можно обработать.

**2. Забывать, что `args[0]` — путь к программе.** Первый аргумент пользователя лежит по индексу 1.

**3. Печатать ошибки через `println!`.** Используйте `eprintln!` для ошибок и `std::process::exit(1)`, чтобы сообщить о неудаче.

**4. Бесконечно ждать stdin.** Если запустить программу без файла и без конвейера, она ждёт ввода с клавиатуры. Нажмите Ctrl-D или дайте ей файл либо конвейер.

**5. Считать буквы через `len()`.** Он считает байты (Урок 03.6). Проекту нужны оба числа.

## Попробуйте сами

1. Создайте черновой проект: `cargo new code/p03-cli-tools`. Поместите программу `args` в `src/main.rs`. Запустите её без аргументов, с `sample.txt` и с тремя аргументами.
2. Замените программу на читающую файл. Сначала создайте `sample.txt` в папке проекта с пятью строками из раздела «Перед началом». Что говорит сообщение `Err`, если вы ошиблись в имени файла?
3. Замените её на программу со стандартным вводом. Запустите тремя способами: наберите что-нибудь и нажмите Ctrl-D; `printf 'a b c\n' | cargo run -q`; `cargo run -q < sample.txt`.
4. Запустите программу с кодом завершения и проверьте `echo $?`.
5. Закоммитьте эту песочницу в `main`: `git add code/p03-cli-tools && git commit -m "p03: cli tools playground"`.

## Проект

### Что вы создадите

Небольшой инструмент командной строки **`p03-word-counter`**, родственник Unix-программы `wc`. Он читает текст из файла (или из стандартного ввода) и печатает:

- сколько в нём **строк**, **слов**, **символов** и **байтов**;
- **самое длинное слово**;
- по желанию, сколько раз встречается заданное **слово** (без учёта регистра).

Проект посвящён владению. В основе дизайна одно правило: **`main` владеет текстом; каждая функция анализа его только заимствует.**

```text
                 ┌───────────────────────────────────────────┐
                 │ main                                      │
 file / stdin ──►│ text: String  (the one owner)             │
                 │   │                                       │
                 │   ├── &text ──► count_lines(&str)  -> usize
                 │   ├── &text ──► count_words(&str)  -> usize
                 │   ├── &text ──► count_chars(&str)  -> usize
                 │   ├── &text ──► longest_word(&str) -> &str   (a slice of text!)
                 │   └── &text ──► count_occurrences(&str, &str) -> usize
                 └───────────────────────────────────────────┘
```

### Контракт

**Где живёт:** `code/p03-word-counter` (бинарный проект Cargo). Весь код может быть в `src/main.rs`.

**Использование:**

```text
p03-word-counter [FILE|-] [WORD]
```

- Без аргументов или с `-` в качестве первого аргумента: читать текст из стандартного ввода.
- Иначе первый аргумент — путь к текстовому файлу.
- Необязательный второй аргумент `WORD` — слово, которое нужно посчитать.

Через Cargo: `cargo run -q -- sample.txt rust`.

**Вывод при успехе** (в stdout, код завершения `0`), ровно эти строки в этом порядке:

```text
lines: <number>
words: <number>
chars: <number>
bytes: <number>
longest word: <word>
occurrences of "<WORD>": <number>      (only if WORD was given)
```

**Определения:**

| Пункт | Правило |
|---|---|
| lines | число строк, как его считает `str::lines()` (то есть завершающий перевод строки не добавляет пустую строку, а у пустого текста 0 строк) |
| words | части, разделённые любыми пробельными символами (`split_whitespace`); знаки препинания остаются частью слова |
| chars | число `char` (не байтов) |
| bytes | `text.len()` |
| longest word | слово с наибольшим числом `char`; при равенстве — **первое**; если слов нет, печатать `longest word: (none)` |
| occurrences | число слов, равных `WORD`, если оба привести к нижнему регистру; слово должно совпасть целиком (то есть `Rust,` не совпадает с `rust`) |

**Ошибки:** если файл нельзя прочитать, напечатайте `error: cannot read '<path>': <reason>` в **stderr** и завершитесь с кодом `1`. Пример: `error: cannot read 'missing.txt': No such file or directory (os error 2)`.

**Обязательные сигнатуры функций** (именно их вызывают тесты; сохраните имена и типы):

```rust
fn count_lines(text: &str) -> usize
fn count_words(text: &str) -> usize
fn count_chars(text: &str) -> usize
fn longest_word(text: &str) -> &str
fn count_occurrences(text: &str, word: &str) -> usize
```

**Правила владения (проверяются на M5):**

1. Каждая функция анализа принимает `&str`, никогда не `String` и не `&String`.
2. `longest_word` возвращает срез своих входных данных. Она не должна выделять память или клонировать.
3. Текстом владеет одна переменная в `main` (или в читающей функции, которая его возвращает). Нигде нет `.clone()` текста.
4. Разрешённые выделения: `to_lowercase()` для сравнения без учёта регистра.

**Правила качества (каждый проект этого курса):** `cargo test` проходит, `cargo clippy` не показывает предупреждений, `cargo fmt --check` чист, в проекте есть короткий `README.md`, и каждая веха — отдельный коммит Git.

### Пошаговый план

Работайте в ветке, чтобы в `main` всегда лежал работающий код (см. [Урок 01.6](../01-git-and-version-control/06-branches-and-merging.md)).

#### M1 — Каркас и чтение ввода

**Цель:** создать проект и ветку; прочитать текст из файла или stdin; напечатать `bytes: <n>`; правильно сообщать об ошибках чтения.

**Руководство:**

1. Из `~/projects/rust`: `git switch -c p03-word-counter`, затем `cargo new code/p03-word-counter`. (Cargo не создаёт новый репозиторий Git внутри существующего; проверьте через `git status`.) Сохраните пять строк примера как `sample.txt` в папке проекта.
2. Напишите функцию `read_input(path: Option<&String>) -> Result<String, String>`: она возвращает текст или сообщение об ошибке.
3. В `main` соберите аргументы, вызовите `read_input(args.get(1))`, разберите результат через `match`: при `Ok` продолжайте; при `Err` напечатайте `error: …` через `eprintln!` и вызовите `process::exit(1)`.
4. Напечатайте `bytes: ` и затем `text.len()`.

<details>
<summary>Подсказка</summary>

`match` внутри `read_input` может иметь две ветки. **Охранное условие** (match guard, `if` после образца) позволяет ветке проверить условие: `Some(p) if p != "-"` означает «путь указан, и это не дефис». Всё остальное (`_`) читает stdin.

```rust
use std::env;
use std::fs;
use std::io;
use std::process;

fn read_input(path: Option<&String>) -> Result<String, String> {
    match path {
        Some(p) if p != "-" => match fs::read_to_string(p) {
            Ok(text) => Ok(text),
            Err(e) => Err(format!("cannot read '{p}': {e}")),
        },
        _ => match io::read_to_string(io::stdin()) {
            Ok(text) => Ok(text),
            Err(e) => Err(format!("cannot read standard input: {e}")),
        },
    }
}

fn main() {
    let args: Vec<String> = env::args().collect();

    let text = match read_input(args.get(1)) {
        Ok(text) => text,
        Err(message) => {
            eprintln!("error: {message}");
            process::exit(1);
        }
    };

    println!("bytes: {}", text.len());
}
```

</details>

**Готово, когда:**

```text
$ echo "hello world" | cargo run -q
bytes: 12
$ cargo run -q -- sample.txt
bytes: 199
$ cargo run -q -- missing.txt
error: cannot read 'missing.txt': No such file or directory (os error 2)
$ cargo run -q -- missing.txt 2>/dev/null; echo "exit code: $?"
exit code: 1
```

Затем запустите `cargo clippy` и `cargo fmt` и сделайте коммит: `git add code/p03-word-counter && git commit -m "p03-word-counter: M1 read input from file or stdin"`.

#### M2 — Подсчёт строк, слов и символов

**Цель:** добавить `count_lines`, `count_words` и `count_chars` с тестами и печатать четыре строки результата.

**Руководство:**

1. Напишите три функции с сигнатурами из контракта. Каждая — одна строка кода с методом `&str` из Урока 03.6 и последующим `.count()`.
2. Напишите тесты **сначала** (или хотя бы вместе): пустой текст, одно слово, лишние пробелы, завершающий перевод строки, пустая строка в середине и кириллическое слово, чтобы доказать, что `chars` и `bytes` различаются.
3. Напечатайте `lines:`, `words:`, `chars:`, `bytes:` в `main`, передавая `&text`.

<details>
<summary>Подсказка</summary>

Три полезных метода: `lines()`, `split_whitespace()` и `chars()`. Каждый выдаёт то, у чего можно вызвать `.count()`. В тестах `assert_eq!(count_lines("one\n\nthree"), 3);` документирует правило пустой строки.

</details>

**Готово, когда:**

```text
$ cargo run -q -- sample.txt
lines: 5
words: 32
chars: 199
bytes: 199
$ printf 'привет мир\n' | cargo run -q
lines: 1
words: 2
chars: 11
bytes: 20
$ printf '' | cargo run -q
lines: 0
words: 0
chars: 0
bytes: 0
$ cargo test
...
test result: ok. N passed; 0 failed
```

(`N` — число ваших тестов; не меньше 3.) Коммит: `p03-word-counter: M2 count lines, words and chars`.

#### M3 — Самое длинное слово (возврат среза)

**Цель:** добавить `longest_word(text: &str) -> &str`, которая возвращает срез входных данных, и печатать `longest word: …`.

**Руководство:**

1. Заведите две переменные: лучшее слово на данный момент (`&str`, изначально `""`) и его длину в `char` (изначально `0`).
2. Для каждого слова из `split_whitespace()` вычислите длину через `word.chars().count()`. Если она **строго больше** лучшей, замените лучшее. (Строго больше означает, что при равенстве остаётся первое.)
3. Верните лучшее. Возвращённый `&str` указывает в входной текст; копия не делается.
4. В `main`, если результат пуст, напечатайте `longest word: (none)`.
5. Тесты: обычный текст, пустой текст, равенство (побеждает первое) и кириллическое слово, которое длиннее в символах, но не в байтах.

<details>
<summary>Подсказка</summary>

```rust
fn longest_word(text: &str) -> &str {
    let mut longest = "";
    let mut longest_len = 0;
    for word in text.split_whitespace() {
        // compare lengths in chars; update both variables when `word` is longer
    }
    longest
}
```

Почему `longest = word;` компилируется? `word` — срез `text`, а сигнатура функции говорит, что результат — представление `text`. Если бы вы попробовали сохранить приведённую к нижнему регистру `String`, вы встретили бы `E0597` (Урок 03.7, случай 7).

</details>

**Готово, когда:**

```text
$ cargo run -q -- sample.txt
lines: 5
words: 32
chars: 199
bytes: 199
longest word: programming
$ printf 'aa bb cc\n' | cargo run -q
lines: 1
words: 3
chars: 9
bytes: 9
longest word: aa
$ printf '' | cargo run -q
lines: 0
words: 0
chars: 0
bytes: 0
longest word: (none)
$ printf 'привет мир\n' | cargo run -q | tail -1
longest word: привет
```

Коммит: `p03-word-counter: M3 longest word as a slice of the input`.

#### M4 — Подсчёт заданного слова

**Цель:** поддержать необязательный второй аргумент и `count_occurrences(text, word)`.

**Руководство:**

1. Приведите `word` к нижнему регистру один раз, до цикла (`let target = word.to_lowercase();`).
2. Для каждого токена из `split_whitespace()` приведите его к нижнему регистру и сравните с `target` через `==`. Считайте совпадения.
3. В `main`, если `args.get(2)` равен `Some(word)`, напечатайте `occurrences of "<word>": <n>` (слово печатайте так, как его набрал пользователь).
4. Тесты: разный регистр (`Rust rust RUST`), слово, являющееся началом другого (`rusty` не должно совпадать с `rust`), и отсутствие совпадений.

<details>
<summary>Подсказка</summary>

`to_lowercase()` создаёт новую `String`. Сравнивать `String` со `String` можно через `==`. Чтобы печатать только когда аргумент есть, `if let Some(word) = args.get(2) { … }` — короткая форма `match` с единственной веткой `Some` (Урок 04.5 учит `if let`). `args.get(2)` даёт `Option<&String>`; передайте `word` в функцию, которая ожидает `&str` (приведение при разыменовании, Урок 03.6).

</details>

**Готово, когда:**

```text
$ cargo run -q -- sample.txt rust
lines: 5
words: 32
chars: 199
bytes: 199
longest word: programming
occurrences of "rust": 3
$ cargo run -q -- sample.txt RUST | tail -1
occurrences of "RUST": 3
$ cargo run -q -- sample.txt zebra | tail -1
occurrences of "zebra": 0
$ printf 'one two\n' | cargo run -q -- - TWO | tail -1
occurrences of "TWO": 1
```

Коммит: `p03-word-counter: M4 count occurrences of a word`.

#### M5 — Качество, README, проверка владения, слияние

**Цель:** привести проект в порядок и задокументировать, проверить правила владения и слить ветку.

**Руководство:**

1. Запустите `cargo fmt`, затем `cargo clippy` (исправьте все предупреждения), затем `cargo test`.
2. Проверьте правила владения из контракта. Например, `grep -n "clone()" src/main.rs` не должен ничего выводить; каждая функция анализа принимает `&str`.
3. Напишите `README.md` в проекте: один абзац о том, что делает инструмент, строку использования, пример запуска, как запускать тесты и короткий раздел «Ownership notes» (кто владеет текстом; какая функция возвращает заимствованный срез).
4. Коммит: `p03-word-counter: M5 README, fmt and clippy clean`.
5. Слейте ветку в `main` и поставьте тег:

   ```bash
   git switch main
   git merge p03-word-counter
   git tag p03-done
   git log --oneline
   ```

<details>
<summary>Подсказка</summary>

Хороший README короткий. Включите точную команду `cargo run -q -- sample.txt rust` и её ожидаемый вывод. Если `git merge` пишет «Fast-forward», это нормально: в вашей ветке были только новые коммиты поверх `main`. Удалите законченную ветку командой `git branch -d p03-word-counter`.

</details>

**Готово, когда:**

```text
$ cargo test
test result: ok. 7 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
$ cargo clippy
    Finished `dev` profile [unoptimized + debuginfo] target(s) in 0.11s
$ cargo fmt --check && echo fmt-ok
fmt-ok
$ grep -n "clone()" src/main.rs
$ git log --oneline
...  p03-word-counter: M5 README, fmt and clippy clean
...  p03-word-counter: M4 count occurrences of a word
...  p03-word-counter: M3 longest word as a slice of the input
...  p03-word-counter: M2 count lines, words and chars
...  p03-word-counter: M1 read input from file or stdin
```

(Число тестов — ваше; с предложенными выше тестами их не меньше 7. `grep` ничего не печатает, когда совпадений нет. `cargo clippy` не должен показывать строк `warning:`; точная строка `Finished` не важна. В журнале первыми показаны ваши собственные хеши.)

### Дополнительные цели

- **Знаки препинания:** убирайте знаки препинания с концов каждого слова перед сравнением (чтобы `Rust,` совпадало с `rust`). Подсказка: `word.trim_matches(|c: char| !c.is_alphanumeric())` — часть между `| … |` это *замыкание*, крошечная безымянная функция (Урок 05.4).
- **Частые слова:** напечатайте три самых частых слова. Для этого нужен `HashMap` (Урок 04.9); вернитесь после Этапа 04.
- **Больше параметров:** примите несколько файлов и напечатайте итоги, как это делает `wc`. Следите, чтобы текстом каждого файла в каждый момент владела одна переменная.
- **Pull request:** отправьте ветку на GitHub или GitLab и откройте pull request для самого себя (Уроки 01.8 и 01.9). Прочтите собственный diff перед слиянием.
- **Бенчмарк:** подайте ему файл в 100 МБ (`yes "some words here" | head -c 100000000 > big.txt`) и измерьте через `time cargo run --release -q -- big.txt`. Основная цена — в чтении или в подсчёте?

## Вопросы для самопроверки

Сначала отвечайте по памяти, затем открывайте ответы.

1. Чем стек отличается от кучи? Приведите по одному примеру того, что живёт в каждой.

<details>
<summary>Ответ</summary>

Стек хранит значения фиксированного размера в кадрах, которые появляются и исчезают с вызовами функций (быстро, небольшой). Куча хранит данные неизвестного или меняющегося размера, которые выдаёт аллокатор (гибко, медленнее, нужно освобождать). Пример: `i32` живёт на стеке; буквы `String` живут в куче, а трёхсловный дескриптор `String` лежит на стеке.

</details>

2. Сформулируйте три правила владения.

<details>
<summary>Ответ</summary>

У каждого значения есть владелец; в каждый момент владелец один; когда владелец выходит из области видимости, значение уничтожается.

</details>

3. Что именно копируется при `let t = s;`, когда `s` — это `String`? Что происходит с `s`?

<details>
<summary>Ответ</summary>

Только 24-байтовый дескриптор (указатель, длина, ёмкость). Буквы в куче не копируются. `s` помечается как перемещённая и больше не может использоваться.

</details>

4. Какие из них `Copy`: `i32`, `String`, `&str`, `(i32, bool)`, `Vec<u8>`?

<details>
<summary>Ответ</summary>

`i32`, `&str` и `(i32, bool)` — `Copy`. `String` и `Vec<u8>` — нет.

</details>

5. Когда следует использовать `.clone()`?

<details>
<summary>Ответ</summary>

Когда вам действительно нужны два независимых владельца данных. Он делает глубокую копию по реальной цене, поэтому сначала попробуйте заимствовать.

</details>

6. Функции нужно только читать `String`. Каким должен быть тип параметра и что пишет вызывающий код?

<details>
<summary>Ответ</summary>

`&str`. Вызывающий код пишет `&my_string` (приведение при разыменовании превращает `&String` в `&str`). Строковый литерал тоже подходит напрямую.

</details>

7. Сформулируйте правила заимствования.

<details>
<summary>Ответ</summary>

В любой момент: любое количество разделяемых ссылок (`&T`) или ровно одна изменяемая ссылка (`&mut T`), но не обе; ссылки всегда должны быть действительными.

</details>

8. Почему `for n in &v { v.push(*n); }` не компилируется?

<details>
<summary>Ответ</summary>

Цикл держит разделяемое заимствование `v`, а `push` требует изменяемого (`E0502`). `push` может переместить буфер и сделать недействительным представление цикла.

</details>

9. Когда заканчивается заимствование?

<details>
<summary>Ответ</summary>

На последнем использовании ссылки (правило NLL, Non-Lexical Lifetimes), а не в конце блока.

</details>

10. Почему функция не может вернуть `&String`, указывающую на локальную `String`?

<details>
<summary>Ответ</summary>

Локальная переменная уничтожается при завершении функции, поэтому ссылка висела бы (`E0106` или `E0515`). Верните саму `String`.

</details>

11. Какого типа литерал `"hello"` и какого он размера на 64-битной машине?

<details>
<summary>Ответ</summary>

`&str`, 16 байтов (указатель и длина). Сам текст живёт внутри программы.

</details>

12. Почему `"привет".len()` равно 12, а `.chars().count()` равно 6?

<details>
<summary>Ответ</summary>

`len()` считает байты; каждая кириллическая буква занимает 2 байта в UTF-8 (Unicode Transformation Format, 8-bit: способ, которым Rust хранит текст в байтах). `chars()` проходит по символам, а их 6.

</details>

13. Почему `let c = names[0];` — ошибка для `Vec<String>` и какие два исправления есть?

<details>
<summary>Ответ</summary>

Это переместило бы `String` из середины вектора (`E0507`). Заимствуйте (`&names[0]`) или клонируйте (`names[0].clone()`).

</details>

14. Что означает `E0382` и на какие три подписи нужно смотреть?

<details>
<summary>Ответ</summary>

Использование перемещённого значения. Подписи показывают, где значение было создано, где оно было перемещено и где использовано после этого.

</details>

15. Почему в этом проекте `longest_word` может вернуть `&str` без выделения памяти?

<details>
<summary>Ответ</summary>

Каждое слово из `split_whitespace()` — срез входного текста. Возврат одного из них — это возврат представления в текст вызывающего кода; сигнатура связывает результат со входом.

</details>

## Практические задания

У каждого задания должны быть тесты. Поместите задания в `code/p03-checkpoint` (по файлу в `src/bin/` на задание или по проекту на задание) и закоммитьте каждое.

### Задание 1 — Первое и последнее слово

Напишите `fn first_and_last_word(text: &str) -> (&str, &str)`. Для `"  the quick brown fox "` она возвращает `("the", "fox")`; для одного слова — это слово дважды; для отсутствия слов — `("", "")`. Без выделения памяти.

<details>
<summary>Подсказка</summary>

Обходите `text.split_whitespace()`. Запомните первое слово один раз (используйте флаг `bool`) и обновляйте `last` на каждом слове.

</details>

<details>
<summary>Ответ</summary>

```rust
fn first_and_last_word(text: &str) -> (&str, &str) {
    let mut first = "";
    let mut last = "";
    let mut seen_any = false;
    for word in text.split_whitespace() {
        if !seen_any {
            first = word;
            seen_any = true;
        }
        last = word;
    }
    (first, last)
}

fn main() {
    let text = String::from("  the quick brown fox ");
    let (first, last) = first_and_last_word(&text);
    println!("first = '{first}', last = '{last}'");
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn several_words() {
        assert_eq!(first_and_last_word("a b c"), ("a", "c"));
    }

    #[test]
    fn one_word_is_both() {
        assert_eq!(first_and_last_word("solo"), ("solo", "solo"));
    }

    #[test]
    fn empty_text() {
        assert_eq!(first_and_last_word("   "), ("", ""));
    }
}
```

```text
first = 'the', last = 'fox'
```

</details>

### Задание 2 — Удалите гласные

Напишите `fn remove_vowels(text: &str) -> String`, которая возвращает текст без букв `aeiouAEIOU`. Входные данные должны остаться пригодными для вызывающего кода. Протестируйте обычный текст, текст с неанглийскими буквами (они остаются) и пустую строку.

<details>
<summary>Подсказка</summary>

Создайте `let mut result = String::new();`, обходите `text.chars()` и добавляйте через `push` каждый символ, которого нет в `"aeiouAEIOU"`. `"aeiouAEIOU".contains(c)` проверяет принадлежность.

</details>

<details>
<summary>Ответ</summary>

```rust
fn remove_vowels(text: &str) -> String {
    let mut result = String::new();
    for c in text.chars() {
        if !"aeiouAEIOU".contains(c) {
            result.push(c);
        }
    }
    result
}

fn main() {
    let original = String::from("Ownership and Borrowing");
    println!("{}", remove_vowels(&original));
    println!("{original}");
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn removes_both_cases() {
        assert_eq!(remove_vowels("Ownership"), "wnrshp");
    }

    #[test]
    fn keeps_non_ascii_letters() {
        assert_eq!(remove_vowels("привет"), "привет");
    }

    #[test]
    fn empty_in_empty_out() {
        assert_eq!(remove_vowels(""), "");
    }
}
```

```text
wnrshp nd Brrwng
Ownership and Borrowing
```

</details>

### Задание 3 — Дописать всё

Напишите `fn append_all(dst: &mut String, parts: &[String])`, которая дописывает каждую часть к `dst`, разделяя `", "` (перед первой частью разделителя нет, а для пустого среза ничего не дописывается). После вызова в `main` вектор частей должен остаться пригодным.

<details>
<summary>Подсказка</summary>

`parts.iter().enumerate()` выдаёт пары `(index, part)`. Добавляйте `", "`, когда индекс больше 0, затем `push_str(part)`. `part` — это `&String`, который подходит там, где ожидается `&str`.

</details>

<details>
<summary>Ответ</summary>

```rust
fn append_all(dst: &mut String, parts: &[String]) {
    for (i, part) in parts.iter().enumerate() {
        if i > 0 {
            dst.push_str(", ");
        }
        dst.push_str(part);
    }
}

fn main() {
    let mut line = String::from("items: ");
    let parts = vec![String::from("a"), String::from("b"), String::from("c")];
    append_all(&mut line, &parts);
    println!("{line}");
    println!("parts still usable: {}", parts.len());
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn joins_with_commas() {
        let mut s = String::from(">");
        append_all(&mut s, &[String::from("x"), String::from("y")]);
        assert_eq!(s, ">x, y");
    }

    #[test]
    fn nothing_to_append() {
        let mut s = String::from("keep");
        append_all(&mut s, &[]);
        assert_eq!(s, "keep");
    }
}
```

```text
items: a, b, c
parts still usable: 3
```

</details>

### Задание 4 — Отладьте четыре ошибки

В этой программе четыре ошибки владения. Исправьте их все в том порядке, в котором их сообщает компилятор, минимальными разумными изменениями. Сделайте коммит, когда закончите. Ожидаемый вывод показан под кодом.

```rust
fn add_title(name: String) -> String {
    name.insert_str(0, "Dr. ");
    name
}

fn main() {
    let mut people = vec![String::from("Ada"), String::from("Grace")];
    let first = people[0];
    let titled = add_title(first);
    let view = &people;
    people.push(String::from("Linus"));
    println!("{titled}: {} people", view.len());
    println!("{first} is still usable");
}
```

Ожидаемый вывод:

```text
Dr. Ada: 2 people
Ada is still usable
```

<details>
<summary>Подсказка</summary>

Компилятор сообщает: `E0596` (параметр не `mut`), `E0507` (нельзя переместить из индекса), `E0502` (push, пока `view` жива и используется позже) и `E0382` (использование `first` после перемещения). Исправления: `mut name`; `people[0].clone()`; напечатать `view.len()` до `push`; передать `first.clone()` в `add_title`.

</details>

<details>
<summary>Ответ</summary>

```rust
fn add_title(mut name: String) -> String {
    name.insert_str(0, "Dr. ");
    name
}

fn main() {
    let mut people = vec![String::from("Ada"), String::from("Grace")];
    let first = people[0].clone();
    let titled = add_title(first.clone());
    let view = &people;
    println!("{titled}: {} people", view.len());
    people.push(String::from("Linus"));
    println!("{first} is still usable");
}
```

```text
Dr. Ada: 2 people
Ada is still usable
```

Лучший проект обходится без двух клонов: пусть `add_title` принимает `&str` и возвращает новую `String` (`format!("Dr. {name}")`), а вместо клонирования заимствуйте `&people[0]`. Обе версии допустимы; вторая дешевле.

</details>

### Задание 5 — Нарисуйте память

Для этого кода нарисуйте (текстом), что лежит на стеке и в куче после последней строки, и скажите, сколько отдельных выделений в куче существует:

```rust
let a = String::from("hi");
let b = &a;
let c = a.clone();
```

<details>
<summary>Подсказка</summary>

`a` — дескриптор плюс буквы в куче. `b` — 8-байтовый адрес `a`. `c` — клон: второй дескриптор плюс второй кусок кучи.

</details>

<details>
<summary>Ответ</summary>

```text
 STACK                                   HEAP
┌───────────────────┐
│ a: ptr ───────────┼────────────────► [ h | i ]   (allocation 1)
│    len = 2, cap = 2│
├───────────────────┤
│ b: &a  (address of a, 8 bytes)
├───────────────────┤
│ c: ptr ───────────┼────────────────► [ h | i ]   (allocation 2)
│    len = 2, cap = 2│
└───────────────────┘
```

Два выделения в куче. `b` ничем не владеет; она указывает на дескриптор `a`. `a` и `c` — независимые владельцы.

</details>

## Если что-то не получилось

| Тема | Повторите |
|---|---|
| Стек, куча, указатели, ёмкость | [Урок 03.1 — Память: стек и куча](01-memory-stack-and-heap.md) |
| Три правила владения, область видимости, порядок уничтожения, перемещение в функции | [Урок 03.2 — Правила владения](02-ownership-rules.md) |
| Перемещение против `Copy` против `.clone()`, `E0382`, `E0507` | [Урок 03.3 — Перемещение, копирование и клонирование](03-move-copy-and-clone.md) |
| `&T`, разыменование, висячие ссылки, NLL | [Урок 03.4 — Ссылки и заимствование](04-references-and-borrowing.md) |
| `&mut T`, правила заимствования, `E0499`, `E0502`, `E0506` | [Урок 03.5 — Изменяемые ссылки и правила заимствования](05-mutable-references-and-the-borrow-rules.md) |
| Срезы, `String` против `&str`, UTF-8, строковые методы | [Урок 03.6 — Срезы и строковые типы](06-slices-and-string-types.md) |
| Чтение ошибок компилятора, набор исправлений, Clippy, Git как страховка | [Урок 03.7 — Исправляем частые ошибки владения](07-fixing-common-ownership-errors.md) |
| Коммиты, ветки и слияние для проекта | [Урок 01.3 — Ваш первый репозиторий и коммиты](../01-git-and-version-control/03-your-first-repository-and-commits.md) и [Урок 01.6 — Ветки и слияние](../01-git-and-version-control/06-branches-and-merging.md) |
| Написание тестов | [Урок 02.8 — Ваши первые тесты](../02-programming-basics/08-your-first-tests.md) |

## Что дальше

Теперь в ваших руках самая важная идея Rust. Дальше почти всё строится на вопросах «кто этим владеет?» и «кто может это заимствовать и на какое время?».

На **Этапе 04** вы используете эти идеи, чтобы строить собственные типы: структуры, перечисления, `Option`, `match`, модули, `Vec`, `HashMap` и правильную обработку ошибок. Начнёте вы с [Урока 04.1 — Структуры](../04-structs-enums-collections-errors/01-structs.md). Позже, на Этапе 05, времена жизни (Урок 05.3) завершат историю, которая началась с `E0106` в Уроке 03.4.

Если вы хорошо справились с вопросами и закончили проект с чистыми `fmt` и `clippy`, двигайтесь дальше. Если какая-то область казалась шаткой, повторите упражнения её урока, прежде чем продолжить. Владение стоит лишнего дня.

## Итоги

- Инструмент командной строки читает **аргументы** (`env::args`, `get` возвращает `Option`), читает **ввод** (`fs::read_to_string` возвращает `Result`; `io::read_to_string(io::stdin())` читает stdin) и сообщает о неудаче через **stderr** и ненулевой **код завершения**.
- `match` обрабатывает каждый случай `Option` или `Result`; компилятор следит, чтобы ни один случай не был забыт.
- Дизайн проекта хорошо использует владение: `main` владеет текстом; функции анализа принимают `&str`; `longest_word` возвращает срез своих входных данных без выделения памяти.
- Работайте по вехам: тесты на каждом шаге, `cargo fmt`, `cargo clippy`, один коммит на веху в ветке, README, затем слияние и тег.
- Контрольная точка охватывает весь этап: память, три правила владения, перемещение/`Copy`/`clone`, разделяемые и изменяемые ссылки, правила заимствования, срезы и строки, а также чтение ошибок.

## Источники и что почитать

- [The Rust Programming Language — 12 An I/O Project: Building a Command Line Program](https://doc.rust-lang.org/book/ch12-00-an-io-project.html) — официальный проект, который строит более крупный инструмент (вы сделаете его версию на Этапе 05).
- [std::env::args](https://doc.rust-lang.org/std/env/fn.args.html) — аргументы командной строки, включая замечание о некорректном Unicode.
- [std::fs::read_to_string](https://doc.rust-lang.org/std/fs/fn.read_to_string.html) — чтение файла в `String`.
- [std::io::read_to_string](https://doc.rust-lang.org/std/io/fn.read_to_string.html) — чтение всего из источника вроде стандартного ввода.
- [std::process::exit](https://doc.rust-lang.org/std/process/fn.exit.html) — завершение программы с кодом выхода.
- [Rustlings](https://github.com/rust-lang/rustlings) — небольшие упражнения; выполните группы про семантику перемещения, ссылки и строки, чтобы закрепить этот этап.

---

[🏠 Начало](../START-HERE.md) · [← 03.7 Исправляем частые ошибки владения](07-fixing-common-ownership-errors.md) · [📋 Этап 03](README.md) · [Этап 04 — Структуры, перечисления, коллекции и ошибки →](../04-structs-enums-collections-errors/README.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
