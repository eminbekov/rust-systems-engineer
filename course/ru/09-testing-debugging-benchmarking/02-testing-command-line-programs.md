<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 09.1 Пирамида тестов и стратегия тестирования](01-the-test-pyramid.md) · [📋 Этап 09](README.md) · [09.3 Тестовые двойники: фейки, стабы и моки →](03-test-doubles.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/09-testing-debugging-benchmarking/02-testing-command-line-programs.md) <!-- nav -->

# Урок 09.2 — Сквозные тесты программ командной строки

> **Этап 09 — Тестирование, отладка и бенчмарки** · Урок 2 из 14 · ⏱ около 120 минут

## Что вы узнаете

- Как тестировать готовую программу командной строки (stdin, аргументы, stdout, stderr, код возврата) с помощью крейта `assert_cmd`.
- Как описывать ожидаемый вывод с помощью *предикатов* (крейт `predicates`), а не сравнением целых строк.
- Как давать каждому тесту собственные временные файлы и папки с помощью крейта `tempfile`.
- Что должен проверять сквозной тест (end-to-end, сокращённо E2E), а что стоит оставить модульным тестам.

## Перед началом

- Вы прошли [Урок 09.1 — Пирамида тестов и стратегия тестирования](01-the-test-pyramid.md). Там вы написали сквозной тест с `std::process::Command` и `env!("CARGO_BIN_EXE_…")`. Он работал, но вам приходилось собирать вывод вручную, превращать байты в текст и сравнивать всё точно. Этот урок заменяет такой код инструментом, созданным для этой задачи.
- Вы знаете, как программа читает стандартный ввод и аргументы ([Урок 02.7](../02-programming-basics/07-reading-input-and-parsing.md)) и как сообщает об ошибках кодом возврата ([Урок 04.10](../04-structs-enums-collections-errors/10-error-handling.md)).

## Новые слова

| Слово | Значение |
|---|---|
| **assert_cmd** | Крейт, который запускает вашу программу в тесте и даёт короткие читаемые проверки кода возврата, stdout и stderr. |
| **predicates** | Крейт маленьких проверок «да/нет» над значениями («содержит ли этот текст `red`?»), которые можно объединять через `and`, `or` и `not`. |
| **предикат** (predicate) | Одна такая проверка: значение, которое для заданного входа отвечает `true` или `false`. |
| **tempfile** | Крейт, который создаёт временные файлы и папки и удаляет их автоматически. |
| **временная папка** (temporary directory) | Папка, созданная только для одного теста во временном месте системы и удаляемая, когда тест заканчивается. |
| **фикстура** (fixture) | Подготовленные данные, нужные тесту до запуска: файлы, папки, входной текст. |
| **строитель** (builder) | Объект, который настраивают вызовом нескольких методов подряд (`.arg(…).env(…).assert()`), каждый из которых возвращает сам объект. |
| **stdin** (standard input, стандартный ввод) | Канал, по которому программа читает ввод, переданный ей по конвейеру или набранный пользователем. |
| **тайм-аут** (timeout) | Ограничение по времени. Программа, не завершившаяся за него, останавливается, и тест падает, а не ждёт вечно. |
| **зависимость для разработки** (dev-dependency) | Крейт, используемый только тестами и примерами и указанный в `[dev-dependencies]` в `Cargo.toml`; он не входит в выпускаемую программу. |

## 1. Зачем для этого крейт?

Тест из Урока 09.1 выглядел так:

```rust
let output = program().args(["ten", "3"]).output().unwrap();
assert_eq!(output.status.code(), Some(2));
let stderr = String::from_utf8(output.stderr).unwrap();
assert_eq!(stderr, "error: 'ten' is not a valid amount\n");
```

Он работает, но у него есть проблемы, которые растут с каждым новым тестом: вы повторяете преобразование байтов в
текст; чтобы подать стандартный ввод, нужно писать дополнительный код с потоками или конвейерами; программа, которая
ждёт ввода, заставит тест зависнуть навсегда; а когда `assert_eq!` падает, вы видите две строки, но не видите команду,
которая их породила.

`assert_cmd` решает эти проблемы. Его тесты читаются как описание поведения: *запусти эту программу с такими
аргументами и таким вводом; она должна завершиться успешно; stdout должен быть таким.*

## 2. Тестируемая программа: `linecount`

Сегодняшняя программа считает строки (или, с `--words`, слова) файла или стандартного ввода, если файл не задан.
У неё есть и пути ошибок: отсутствующий файл и неверные аргументы. Создайте пакет:

```bash
cd ~/projects/rust
git switch main
git switch -c lesson/09-02-cli-tests
cargo new code/p09-cli-tests
cd code/p09-cli-tests
cargo add --dev assert_cmd@2 predicates@3 tempfile@3
```

`cargo add --dev` записывает три крейта в `[dev-dependencies]`. Откройте `Cargo.toml` и добавьте в конец раздел
`[[bin]]`, чтобы программа называлась `linecount`, а не `p09-cli-tests`:

```toml
[package]
name = "p09-cli-tests"
version = "0.1.0"
edition = "2024"

[dependencies]

[dev-dependencies]
assert_cmd = "2"
predicates = "3"
tempfile = "3"

[[bin]]
name = "linecount"
path = "src/main.rs"
```

Замените `src/main.rs`:

```rust
use std::io::{self, Read};
use std::process::ExitCode;

/// Reads the whole input: a file if a path is given, otherwise standard input.
fn read_input(path: Option<&str>) -> io::Result<String> {
    match path {
        Some(path) => std::fs::read_to_string(path),
        None => {
            let mut text = String::new();
            io::stdin().read_to_string(&mut text)?;
            Ok(text)
        }
    }
}

fn main() -> ExitCode {
    let args: Vec<String> = std::env::args().skip(1).collect();
    let (count_words, path) = match args.as_slice() {
        [] => (false, None),
        [flag] if flag == "--words" => (true, None),
        [path] => (false, Some(path.as_str())),
        [flag, path] if flag == "--words" => (true, Some(path.as_str())),
        _ => {
            eprintln!("usage: linecount [--words] [FILE]");
            return ExitCode::from(2);
        }
    };

    let text = match read_input(path) {
        Ok(text) => text,
        Err(error) => {
            eprintln!("error: cannot read the input: {error}");
            return ExitCode::from(1);
        }
    };

    if count_words {
        println!("{} words", text.split_whitespace().count());
    } else {
        println!("{} lines", text.lines().count());
    }
    ExitCode::SUCCESS
}
```

**Что происходит:**

1. `match args.as_slice()` смотрит на аргументы как на срез и выбирает форму: нет аргументов, `--words`, путь
   или `--words` и путь. Всё остальное — ошибка использования: сообщение в stderr и код возврата 2.
2. `read_input` читает файл, если есть путь, и стандартный ввод в противном случае. Оба варианта могут не удаться;
   неудача превращается в код возврата 1 и сообщение в stderr.
3. `main` возвращает `ExitCode`. Это способ завершить программу выбранным кодом возврата, не вызывая `process::exit`.
4. `text.lines().count()` считает строки; `text.split_whitespace().count()` считает слова.

Попробуйте:

```bash
printf 'roses are red\nviolets are blue\n' | cargo run -q -- --words
cargo run -q -- nothing.txt; echo "exit code: $?"
cargo run -q -- a b c; echo "exit code: $?"
```

```text
6 words
error: cannot read the input: No such file or directory (os error 2)
exit code: 1
usage: linecount [--words] [FILE]
exit code: 2
```

## 3. Первый тест с `assert_cmd`

Создайте `tests/cli.rs`:

```rust
use std::time::Duration;

use assert_cmd::Command;
use predicates::prelude::*;

fn linecount() -> Command {
    let mut cmd = Command::cargo_bin("linecount").unwrap();
    cmd.timeout(Duration::from_secs(10));
    cmd
}

#[test]
fn counts_lines_from_standard_input() {
    linecount()
        .write_stdin("one\ntwo\nthree\n")
        .assert()
        .success()
        .stdout("3 lines\n")
        .stderr("");
}
```

```bash
cargo test --test cli
```

```text
running 1 test
test counts_lines_from_standard_input ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.13s
```

**Что происходит:**

1. `assert_cmd::Command` — обёртка над стандартным `Command` с дополнительными методами. `Command::cargo_bin("linecount")`
   находит бинарный файл, который Cargo собрал для этого пакета (имя берётся из `[[bin]]`), и возвращает `Result`,
   потому что бинарного файла может не быть.
2. `cmd.timeout(…)` задаёт ограничение в десять секунд. Если программа ждёт ввода, которого нет, тест упадёт
   через десять секунд, а не зависнет. Мы положили это во вспомогательную функцию `linecount()`, чтобы ни один тест это не забыл.
3. `.write_stdin("…")` подаёт стандартный ввод. Без него программа ничего бы не прочитала.
4. `.assert()` запускает программу и возвращает объект `Assert`, который хранит всё, что программа сделала.
   Каждый следующий вызов — это проверка: `.success()` требует код возврата 0; `.stdout("3 lines\n")` требует, чтобы stdout был
   *в точности* таким текстом; `.stderr("")` требует, чтобы в stderr ничего не писалось.
5. Проверки — цепочка вызовов методов (*строитель*), поэтому тест читается сверху вниз как предложение из
   введения.

> 📏 **Соглашение:** выносите общую подготовку (имя бинарного файла, тайм-аут) в одну вспомогательную функцию вроде `linecount()`.
> Тогда каждый тест начинается с одной и той же безопасной основы.

## 4. Предикаты: описываем, что мы ожидаем

Точное сравнение — правильный выбор, когда вывод короткий и фиксированный. Оно неверно, когда часть вывода вы не
контролируете: путь, время или сообщение, написанное операционной системой. Для таких случаев `.stdout(…)` и
`.stderr(…)` принимают также **предикат**.

Предикат — крошечный объект с одним вопросом: «приемлемо ли это значение?» Предикаты не зависят от
`assert_cmd`, поэтому их можно попробовать в обычном тесте (добавьте в `tests/cli.rs`):

```rust
#[test]
fn predicates_answer_yes_or_no() {
    let has_red = predicate::str::contains("red");
    assert!(has_red.eval("roses are red"));
    assert!(!has_red.eval("violets are blue"));

    let red_roses = predicate::str::contains("red").and(predicate::str::contains("roses"));
    assert!(red_roses.eval("roses are red"));
    assert!(!red_roses.eval("apples are red"));

    let not_empty = predicate::str::is_empty().not();
    assert!(not_empty.eval("x"));
}
```

**Что происходит:**

1. `use predicates::prelude::*` подключает `predicate` (модуль с конструкторами) и трейты, которые дают предикатам
   их методы.
2. `predicate::str::contains("red")` создаёт предикат; `.eval(text)` задаёт вопрос и возвращает `bool`.
3. `.and(…)`, `.or(…)` и `.not()` объединяют предикаты в новый.

Чаще всего вам понадобятся эти:

| Предикат | Истинен, когда текст… |
|---|---|
| `predicate::str::contains("x")` | содержит `x` где-нибудь |
| `predicate::str::starts_with("x")` / `ends_with("x")` | начинается / заканчивается на `x` |
| `predicate::str::is_empty()` | пуст |
| `predicate::eq("x")` | в точности равен `x` (его использует внутри `.stdout("x")`) |
| `p1.and(p2)`, `p1.or(p2)`, `p.not()` | удовлетворяет обоим / хотя бы одному / не удовлетворяет предикату |

Теперь используем предикаты в тесте неудачи. Добавьте:

```rust
#[test]
fn a_missing_file_fails_with_exit_code_1() {
    linecount()
        .arg("no-such-file.txt")
        .assert()
        .failure()
        .code(1)
        .stdout("")
        .stderr(
            predicate::str::starts_with("error: cannot read the input")
                .and(predicate::str::contains("No such file")),
        );
}

#[test]
fn too_many_arguments_print_the_usage_and_exit_with_2() {
    linecount()
        .args(["--words", "a.txt", "b.txt"])
        .assert()
        .code(2)
        .stderr("usage: linecount [--words] [FILE]\n");
}
```

**Что происходит:**

1. `.failure()` требует ненулевой код возврата; `.code(1)` требует ровно 1. Проверять точный код стоит: скрипты,
   которые вызывают вашу программу, принимают решения по коду.
2. Для отсутствующего файла первая половина сообщения наша (`error: cannot read the input`), а вторая приходит от
   операционной системы. Нашу часть мы проверяем строго (`starts_with`), а часть системы — мягко
   (`contains("No such file")`), чтобы тест не зависел от точной формулировки системного сообщения.
3. `.args([...])` передаёт несколько аргументов сразу; `.arg(x)` передаёт один.
4. Сообщение об использовании целиком наше, поэтому мы сравниваем его точно.

> 💡 **Совет:** проверяя сообщение об ошибке, проверяйте ту часть, которую написали *вы*, и ту, что говорит пользователю, *что пошло не так*.
> Не проверяйте формулировки сообщений, созданных другими библиотеками или системой: они меняются между версиями
> и между операционными системами.

## 5. Файлы и папки: `tempfile`

Многие программы командной строки читают или пишут файлы. Тест никогда не должен трогать ваши настоящие файлы, и два
теста никогда не должны использовать один и тот же файл, потому что Rust запускает тесты **параллельно**. Крейт
`tempfile` даёт каждому тесту своё пустое место:

```rust
#[test]
fn counts_words_in_a_file() {
    let dir = tempfile::tempdir().unwrap();
    let file = dir.path().join("poem.txt");
    std::fs::write(&file, "roses are red\nviolets are blue\n").unwrap();

    linecount()
        .arg("--words")
        .arg(&file)
        .assert()
        .success()
        .stdout("6 words\n");
}

#[test]
fn a_relative_path_is_resolved_in_the_current_directory() {
    let dir = tempfile::tempdir().unwrap();
    std::fs::write(dir.path().join("notes.txt"), "a\nb\n").unwrap();

    linecount()
        .current_dir(dir.path())
        .arg("notes.txt")
        .assert()
        .success()
        .stdout("2 lines\n");
}

#[test]
fn input_can_come_from_a_file() {
    use std::io::Write;
    let mut file = tempfile::NamedTempFile::new().unwrap();
    writeln!(file, "first").unwrap();
    writeln!(file, "second").unwrap();

    linecount()
        .pipe_stdin(file.path())
        .unwrap()
        .assert()
        .success()
        .stdout("2 lines\n");
}

#[test]
fn the_temporary_directory_is_removed_when_dropped() {
    let path;
    {
        let dir = tempfile::tempdir().unwrap();
        path = dir.path().to_path_buf();
        assert!(path.exists());
    }
    assert!(!path.exists());
}
```

**Что происходит:**

1. `tempfile::tempdir()` создаёт новую пустую папку с уникальным случайным именем во временном месте системы и
   возвращает `TempDir`. `dir.path()` — её путь. Тест строит свою фикстуру внутри неё через `std::fs::write`.
2. Папка **удаляется автоматически, когда значение `TempDir` уничтожается** (когда `dir` выходит из области видимости в конце
   теста; [Урок 03.2](../03-ownership-and-borrowing/02-ownership-rules.md) объяснял, как Rust уничтожает значения). Последний тест доказывает это: после окончания внутреннего блока пути больше нет.
   Вам не нужно писать код очистки, и упавший тест тоже за собой убирает.
3. `.current_dir(dir.path())` запускает программу внутри этой папки, поэтому относительный путь `notes.txt`
   разрешается там.
4. `NamedTempFile::new()` создаёт один временный файл. `writeln!` пишет в него строки. `.pipe_stdin(path)` подаёт
   содержимое файла на стандартный ввод программы; он возвращает `Result`, поэтому нужен `.unwrap()`.

> 🔒 **Безопасность:** не составляйте имена временных файлов вручную (`/tmp/test.txt`). Другие тесты или другие пользователи
> той же машины могут использовать то же имя или заранее подложить там файл. `tempfile` создаёт уникальные и
> непредсказуемые имена и открывает файл безопасно.

## 6. Когда тест падает

Сквозной тест полезен, только если его сообщение о падении помогает. Измените ожидаемый вывод на `"4 lines\n"` и запустите тест:

```text
---- counts_lines_from_standard_input stdout ----
thread 'counts_lines_from_standard_input' (4665549) panicked at tests/cli.rs:18:10:
Unexpected stdout, failed diff original var
├── original: 4 lines
├── diff:
│   ---	orig
│   +++	var
│   @@ -1 +1 @@
│   -4 lines
│   +3 lines
└── var as str: 3 lines
command=`".../target/debug/linecount"`
stdin=(the three input lines, one per line)
code=0
stdout="3 lines\n"
stderr=""
```

**Что это значит:** вы получаете разницу между ожидаемым и фактическим выводом, а под ней — **полную картину
запуска**: запущенную команду, полученный ввод, код возврата и оба канала вывода. Обычно этого достаточно, чтобы понять
падение, ничего не запуская вручную. Если вы потребуете `.success()` от программы, которая падает, сообщение будет
`Unexpected failure.` и тот же блок с `code=1` и текстом stderr. Если падает предикат, сообщение называет его,
например `failed var.contains(permission denied)`.

## 7. Что должен проверять сквозной тест

Каждый сквозной тест стоит сотни миллисекунд, как вы измерили в Уроке 09.1, поэтому используйте их для того, что может
показать только настоящий процесс:

| Проверяйте сквозным тестом | Оставьте модульным и интеграционным тестам |
|---|---|
| код возврата при успехе и при каждом виде неудачи | как разбирается или форматируется число |
| основной формат вывода (stdout) | каждый граничный случай логики |
| канал ошибок (stderr), начало сообщений | точную формулировку каждого сообщения |
| чтение stdin, аргументов, файлов и окружения | длинные списки комбинаций входов |
| программа не зависает (тайм-аут) | внутренние структуры данных |

Ещё два инструмента `assert_cmd` стоит знать. `.env("NAME", "value")` задаёт переменную окружения для
программы (а `.env_remove("NAME")` убирает), так тестируют программу, которая читает настройки из
окружения. `.timeout(…)` мы уже использовали.

## Частые ошибки

**1. Неверное имя бинарного файла.** `Command::cargo_bin("linecounter")` паникует, а сообщение перечисляет настоящие имена:

```text
thread 'wrong_name_demo' (4674179) panicked at …/assert_cmd-2.2.2/src/cargo.rs:232:5:
`CARGO_BIN_EXE_linecounter` is unset
help: available binary names are "linecount"
```

**Что это значит:** нужно имя бинарного файла (`[[bin]] name` или имя пакета, если `[[bin]]` нет), а не имя пакета
или файла.

**2. Забытый перевод строки.** `println!` завершает строку символом `\n`. `.stdout("3 lines")` упадёт; пишите `.stdout("3 lines\n")`
или используйте `predicate::str::contains("3 lines")`.

**3. Фиксированное имя файла.** Тесты, которые пишут в `test.txt` в текущей папке, работают по одному и падают случайным
образом при параллельном запуске (*нестабильный тест*, Урок 05.6). Используйте `tempdir()`.

**4. Проверка точного текста сообщения операционной системы.** `No such file or directory (os error 2)` на других
системах другое. Проверяйте только свою часть, как в разделе 4.

**5. Слишком короткая жизнь `TempDir`.** `let path = tempfile::tempdir().unwrap().path().to_path_buf();` уничтожает
`TempDir` в конце этой строки, поэтому папка удаляется сразу. Держите сам `TempDir` в переменной (`let dir = …;`)
столько, сколько тесту нужна папка.

## Попробуйте сами

1. Создайте пакет, `Cargo.toml` и `src/main.rs`, как в разделе 2. Выполните три команды из раздела 2 вручную.
2. Создайте `tests/cli.rs` со всеми тестами из разделов 3–5. Запустите `cargo test --test cli`; вы должны увидеть девять проходящих тестов.
3. Измените `.stdout("3 lines\n")` на `.stdout("4 lines\n")`, запустите тест и прочитайте падение. Верните как было.
4. Запустите контроль качества: `cargo fmt --check && cargo clippy --all-targets -- -D warnings && cargo test`.
   (`--all-targets` заставляет Clippy проверять и тесты.)
5. Закоммитьте: `git add code/p09-cli-tests && git commit -m "Add p09-cli-tests: end-to-end tests with assert_cmd"`.

## Упражнения

### Упражнение 1 — Любые пробельные символы (легко)

Напишите тест, проверяющий, что `linecount --words` считает `"a  b\tc\n"` (два пробела, табуляция) как `3 words`.

<details>
<summary>Подсказка</summary>

Используйте `.arg("--words")` и `.write_stdin("a  b\tc\n")`. Ожидаемый stdout — `"3 words\n"`.

</details>

<details>
<summary>Ответ</summary>

```rust
#[test]
fn words_are_separated_by_any_whitespace() {
    linecount()
        .arg("--words")
        .write_stdin("a  b\tc\n")
        .assert()
        .success()
        .stdout("3 words\n");
}
```

Почему это работает: `split_whitespace` делит по любой последовательности пробельных символов (пробелы, табуляции, переводы
строк), поэтому два пробела и табуляция всё равно дают три слова.

</details>

### Упражнение 2 — Папка вместо файла (средне)

Что делает программа, если путь — это *папка*? Напишите тест, который создаёт временную папку, передаёт её как
аргумент и проверяет код возврата и начало сообщения.

<details>
<summary>Подсказка</summary>

`.arg(dir.path())` принимает `&Path`. Чтение папки как файла — ошибка ввода-вывода, поэтому программа идёт по пути
«cannot read the input». Не проверяйте системную часть сообщения.

</details>

<details>
<summary>Ответ</summary>

```rust
#[test]
fn a_directory_is_not_a_file() {
    let dir = tempfile::tempdir().unwrap();
    linecount()
        .arg(dir.path())
        .assert()
        .failure()
        .code(1)
        .stderr(predicate::str::starts_with("error: cannot read the input"));
}
```

Почему это работает: `read_to_string` для папки возвращает ошибку, поэтому программа печатает наше сообщение и возвращает
код возврата 1. Тест держит `dir` живым до конца функции, поэтому папка существует, пока работает программа.

</details>

### Упражнение 3 — Ввод, который не текст (сложно)

Программа читает текст. Что произойдёт, если на вход придут два байта, не являющиеся допустимым UTF-8 (`0xff, 0xfe`)? (UTF-8, Unicode Transformation Format 8-bit, формат преобразования Юникода, — стандартный способ хранить текст в байтах; `String` в Rust всегда должна содержать допустимый UTF-8.)
Напишите тест, который отправляет их на стандартный ввод и проверяет результат. Какой код возврата использует программа и кто его выбрал?

<details>
<summary>Подсказка</summary>

`write_stdin` принимает всё, что можно превратить в `Vec<u8>`, поэтому подойдёт `vec![0xff, 0xfe]`. Сообщение об ошибке
стандартной библиотеки содержит «valid UTF-8».

</details>

<details>
<summary>Ответ</summary>

```rust
#[test]
fn input_that_is_not_utf8_is_rejected() {
    linecount()
        .write_stdin(vec![0xff, 0xfe])
        .assert()
        .failure()
        .code(1)
        .stderr(predicate::str::contains("valid UTF-8"));
}
```

Почему это работает: `read_to_string` отказывается принимать байты, не являющиеся допустимым UTF-8, поэтому `read_input` возвращает ошибку,
а `main` завершается с кодом 1. *Автор программы* решил, что «cannot read the input» — это код 1, а «неверные аргументы» —
код 2; тест делает это решение видимым и сохраняет его неизменным.

</details>

## Вопросы для самопроверки

1. Что делает `Command::cargo_bin("linecount")`?

<details>
<summary>Ответ</summary>

Находит бинарный файл, который Cargo собрал для цели с именем `linecount` в текущем пакете, и возвращает
`assert_cmd::Command` для него (или ошибку, если такого файла нет).

</details>

2. Чем отличается `.stdout("x\n")` от `.stdout(predicate::str::contains("x"))`?

<details>
<summary>Ответ</summary>

Первое требует, чтобы stdout был в точности `x\n`. Второе требует лишь, чтобы `x` встретился где-нибудь. Используйте
точную форму для короткого фиксированного вывода и предикат, когда часть вывода меняется.

</details>

3. Почему тесты должны использовать `tempfile`, а не фиксированное имя файла?

<details>
<summary>Ответ</summary>

Тесты запускаются параллельно, поэтому два теста с одним файлом мешали бы друг другу и падали бы случайным образом. `tempfile` также
удаляет файлы автоматически и избегает предсказуемых имён.

</details>

4. Когда `TempDir` удаляет свою папку?

<details>
<summary>Ответ</summary>

Когда значение `TempDir` уничтожается, обычно в конце тестовой функции, в том числе когда тест падает.

</details>

5. Зачем вспомогательная функция `linecount()` задаёт тайм-аут?

<details>
<summary>Ответ</summary>

Программа, которая ждёт ввода, иначе заставила бы тест работать вечно. С тайм-аутом тест падает по истечении лимита,
и проблема становится видна.

</details>

## Итоги

- `assert_cmd` запускает вашу готовую программу в тесте: `Command::cargo_bin("name")`, затем `.arg`, `.args`, `.write_stdin`,
  `.env`, `.current_dir`, `.timeout`, затем `.assert()` и проверки `.success()`, `.failure()`, `.code(n)`, `.stdout(…)`, `.stderr(…)`.
- `.stdout(…)` и `.stderr(…)` принимают точную строку или предикат из крейта `predicates`; предикаты объединяются через
  `and`, `or` и `not`.
- `tempfile::tempdir()` и `NamedTempFile` дают каждому тесту уникальные файлы, которые удаляются автоматически.
- Упавший тест `assert_cmd` печатает команду, ввод, код возврата и оба вывода.
- Коды возврата и собственные сообщения проверяйте строго, а чужие сообщения — мягко.
- Используйте сквозные тесты для того, что показывает только настоящий процесс; детали логики оставьте модульным тестам.

## Источники и что почитать

- [assert_cmd on docs.rs](https://docs.rs/assert_cmd/2/assert_cmd/) — полный API крейта с примерами.
- [predicates on docs.rs](https://docs.rs/predicates/3/predicates/) — все предикаты и способы их объединять.
- [tempfile on docs.rs](https://docs.rs/tempfile/3/tempfile/) — `tempdir`, `NamedTempFile` и гарантии, которые они дают.
- [Command Line Applications in Rust — Testing](https://rust-cli.github.io/book/tutorial/testing.html) — руководство рабочей группы по CLI (Command-Line Interface, интерфейс командной строки) о тестировании программы командной строки.
- [The Cargo Book — Specifying dependencies: development dependencies](https://doc.rust-lang.org/cargo/reference/specifying-dependencies.html) — что значит `[dev-dependencies]`.

---

[🏠 Начало](../START-HERE.md) · [← 09.1 Пирамида тестов и стратегия тестирования](01-the-test-pyramid.md) · [📋 Этап 09](README.md) · [09.3 Тестовые двойники: фейки, стабы и моки →](03-test-doubles.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
