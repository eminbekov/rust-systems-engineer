<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 04.10 Обработка ошибок](10-error-handling.md) · [📋 Этап 04](README.md) · [04.12 Контрольная точка →](12-checkpoint.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/04-structs-enums-collections-errors/11-project-phone-book.md) <!-- nav -->

# Урок 04.11 — Проект: телефонная книга

> **Этап 04 — Структуры, перечисления, коллекции и ошибки** · Урок 11 из 12 · ⏱ около 3 часов (2 занятия)

## Что вы узнаете

- Как объединить структуры, перечисления, `BTreeMap`, строки, `Result` и модули в одной настоящей программе.
- Как спроектировать небольшой *язык команд* и превратить текст в перечисление (разбор команд, parser).
- Как отделить логику, которую можно тестировать (библиотечный крейт), от ввода/вывода (бинарный крейт).
- Как читать ввод программы из терминала или из конвейера и как загружать и сохранять текстовый файл.
- Как работать по-профессиональному: один коммит Git на каждую веху, фича-ветка, слияние, тег и чистые `cargo fmt`, `cargo clippy` и `cargo test`.

## Перед началом

- Вы прошли [Урок 04.10 — Обработка ошибок](10-error-handling.md). Этот проект использует всё из Уроков 04.1–04.10, особенно [модули](06-modules-crates-and-packages.md), [строки](08-strings-in-depth.md), [таблицы](09-hash-maps.md) и [ошибки](10-error-handling.md).
- Вы помните основы Git из Этапа 01: [коммиты](../01-git-and-version-control/03-your-first-repository-and-commits.md), [история](../01-git-and-version-control/04-viewing-history-and-changes.md), [ветки и слияние](../01-git-and-version-control/06-branches-and-merging.md) и [соглашения о коммитах](../01-git-and-version-control/09-teamwork-pull-requests-and-conventions.md). В этом проекте вы используете их все.
- Вы умеете писать и запускать тесты ([Урок 02.8](../02-programming-basics/08-your-first-tests.md)).
- Этот урок — **проект**: он даёт цель, точные правила (*контракт*) и план с проверками. Полного решения он **не** даёт. Небольшие подсказки спрятаны под «Подсказка»; сначала попробуйте без них.

## Новые слова

| Слово | Значение |
|---|---|
| **REPL** (Read-Eval-Print Loop, цикл «чтение — вычисление — печать») | Программа, которая повторяет: прочитать команду, выполнить её, напечатать результат и прочитать следующую. Так работают командные оболочки и приглашение Python. |
| **язык команд** (command language) | Небольшой набор слов, которые пользователь вводит, чтобы управлять программой, например `add alice 5550100`. |
| **разборщик** (parser) | Код, который превращает текст в структурированные данные. Здесь: строку текста в перечисление `Command`. |
| **приглашение** (prompt) | Короткий знак (здесь `> `), который программа печатает, показывая, что ждёт ввода. |
| **конвейер** (pipe, `\|`) | Возможность командной оболочки, которая отправляет вывод одной команды на вход другой: `printf 'a\n' \| cargo run`. |
| **терминал и конвейер** | Когда печатает человек, стандартный ввод — это *терминал*. Когда данные в него подаёт другая программа через конвейер, стандартный ввод терминалом *не* является. Программа может спросить, что именно перед ней. |
| **EOF** (End Of File, конец файла) | Сигнал, что ввода больше нет. В терминале его посылают клавишами `Ctrl+D`. Тогда `read_line` возвращает `0` байт. |
| **сброс буфера** (flush) | Принудительно вывести буферизованный вывод прямо сейчас. `print!` без перевода строки может остаться в буфере, пока вы не сделаете flush. |
| **библиотечный крейт** (library crate) | Код в `src/lib.rs`, который может использовать другой код (включая ваш бинарный крейт и тесты). |
| **бинарный крейт** (binary crate) | Код в `src/main.rs`, который становится запускаемой программой. |
| **контракт** (contract) | Точные записанные правила того, как программа должна себя вести, включая каждое сообщение. |
| **веха** (milestone) | Запланированный шаг с чёткой целью и проверкой («готово, когда»). Обозначается M1, M2, ... |
| **приёмочный тест** (acceptance test) | Проверка, которая снаружи показывает, что программа делает то, что сказано в контракте. |
| **линтер** (linter) | Инструмент, который читает ваш код и предупреждает о подозрительных или неидиоматичных местах. Линтер Rust называется **Clippy**. |
| **`rustfmt`** | Официальный инструмент, который форматирует код Rust в едином стандартном стиле. Запускается командой `cargo fmt`. |
| **README** | Текстовый файл (здесь `README.md`), который рассказывает читателю, что это за проект и как им пользоваться. |
| **фича-ветка** (feature branch) | Ветка Git, созданная для одной части работы и влитая обратно в `main`, когда работа готова. |
| **тег** (tag) | Постоянная именованная метка на одном коммите в Git, например `v0.1.0`. |
| **Conventional Commits** | Стиль именования сообщений коммитов: `feat: add search`, `fix: ...`, `docs: ...`, `test: ...`, `chore: ...`. |
| **CLI** (Command-Line Interface, интерфейс командной строки) | Способ пользоваться программой, вводя команды в терминале. |
| **I/O** (Input/Output, ввод/вывод) | Чтение из внешнего мира и запись в него: клавиатура, экран, файлы. |
| **CI** (Continuous Integration, непрерывная интеграция) | Сервер, который автоматически собирает и тестирует ваш код после каждой отправки в удалённый репозиторий. |

## Проект

### Что вы создадите

Программа **телефонная книга** с именем `p04-phonebook`. Она читает команды с клавиатуры (или из
конвейера) по одной в строке и отвечает на экране:

```text
> add alice +998-90-111-22-33
ok: added +998-90-111-22-33 to alice
> add alice +998-91-444-55-66
ok: added +998-91-444-55-66 to alice
> get alice
alice: +998-90-111-22-33, +998-91-444-55-66
> find al
alice
> quit
```

(Знаки `> ` и ваш ввод видны только в терминале; раздел «Контракт» объясняет почему.) Её можно
запустить с именем файла, и тогда книга загружается из этого файла в начале и сохраняется в него
в конце.

Ваш код лежит в `~/projects/rust/code/p04-phonebook`.

### Контракт

**Использование**

```text
p04-phonebook [FILE]
```

- Без `FILE` телефонная книга живёт только в памяти.
- С `FILE` программа загружает книгу из него в начале и сохраняет книгу в него, когда программа
  завершается нормально (`quit` или конец ввода). Отсутствие файла — **не** ошибка: это означает
  пустую книгу, а файл создаётся в конце.
- Программа читает стандартный ввод построчно до `quit` или конца ввода (EOF).
- Приглашение `> ` (без перевода строки после него) она печатает **только когда стандартный ввод —
  терминал**. В конвейере вывод должен содержать только ответы.
- Пустая строка (или строка только из пробелов) ничего не делает и ничего не печатает.
- Код завершения `0` при нормальном выходе. Код `1`, если файл не удаётся прочитать, он содержит
  плохие данные или в него не удаётся записать. Эти сообщения идут в **stderr** в виде
  `error: <сообщение>`.

**Правила**

- **Имя** — одно слово (без пробелов). Имена не зависят от регистра: храните и печатайте их в
  нижнем регистре.
- **Номер телефона** — необязательный `+`, затем только цифры и `-`, **не менее 7 цифр**.
  Верно: `+998-90-123-45-67`, `5550100`. Неверно: `12345`, `12a45678`, `+`, `""`.
- У одного имени может быть несколько номеров; храните их в порядке добавления.
- Один и тот же номер дважды для одного имени — ошибка.
- Команды тоже не зависят от регистра (`QUIT` работает), но их аргументы используются как введены (имена затем приводятся к нижнему регистру).

**Команды и точный вывод** (каждый ответ — один `println!`; `list`, `find` и `help` печатают несколько строк)

| Ввод | Вывод |
|---|---|
| `add <name> <phone>` | `ok: added <phone> to <name>` |
| тот же номер ещё раз | `error: <name> already has <phone>` |
| плохой номер | `error: invalid phone '<phone>'` |
| `get <name>` | `<name>: <phone1>, <phone2>` (в порядке добавления) |
| `del <name>` | `ok: deleted <name>` |
| `get` или `del` для неизвестного имени | `error: no entry for '<name>'` |
| `list` | по одной строке `<name>: <phones>` на имя, отсортировано по имени; или `(empty)` |
| `find <prefix>` | имена, начинающиеся с префикса (без учёта регистра), отсортированные, по одному в строке; или `(no matches)` |
| `count` | `<n> names, <m> numbers` |
| `help` | список команд (формат ниже) |
| `quit` | завершает цикл (ничего не печатает) |
| неверное число аргументов | `error: usage: add <name> <phone>`, `error: usage: get <name>`, `error: usage: del <name>`, `error: usage: find <prefix>`; для `list`, `count`, `help`, `quit` с лишними словами: `error: usage: this command takes no arguments` |
| любое другое первое слово | `error: unknown command '<word>' (type help)` |

Текст `help`:

```text
commands:
  add <name> <phone>
  get <name>
  del <name>
  list
  find <prefix>
  count
  help
  quit
```

**Формат файла** (что пишет `save` и читает `load`): одна строка на имя, в порядке имён. Строка —
это имя, за которым идёт каждый телефон, всё через одиночные пробелы, и она заканчивается
переводом строки. Пустая книга даёт пустой файл. Пример:

```text
alice +1-555-0100 +1-555-0199
bob 5550123
```

При загрузке пустые строки пропускаются; неверный или повторяющийся телефон заставляет всю
загрузку завершиться ошибкой `error: bad data on line <N>: <reason>` (строки считаются с 1),
например `error: bad data on line 2: invalid phone '12'`.

**Предлагаемые типы** (детали можно менять, но поведение сохраните):

```rust
// src/phonebook.rs
pub enum PhoneBookError { InvalidPhone(String), Duplicate { name: String, phone: String }, NoEntry(String) }
pub fn is_valid_phone(phone: &str) -> bool;
pub struct PhoneBook { /* a BTreeMap<String, Vec<String>> */ }
impl PhoneBook {
    pub fn new() -> PhoneBook;
    pub fn add(&mut self, name: &str, phone: &str) -> Result<(), PhoneBookError>;
    pub fn get(&self, name: &str) -> Result<&[String], PhoneBookError>;
    pub fn delete(&mut self, name: &str) -> Result<(), PhoneBookError>;
    pub fn count_names(&self) -> usize;
    pub fn count_numbers(&self) -> usize;
    pub fn list(&self) -> Vec<(&String, &Vec<String>)>;
    pub fn find(&self, prefix: &str) -> Vec<&String>;
}

// src/command.rs
pub enum Command { Add { name: String, phone: String }, Get(String), Del(String),
                   List, Find(String), Count, Help, Quit }
pub enum ParseError { Unknown(String), Usage(&'static str) }
pub fn parse_command(line: &str) -> Result<Command, ParseError>;
pub fn execute(book: &mut PhoneBook, command: Command) -> String;   // the text to print

// src/storage.rs
pub enum StorageError { Io(std::io::Error), BadLine { line: usize, reason: String } }
pub fn to_text(book: &PhoneBook) -> String;
pub fn from_text(text: &str) -> Result<PhoneBook, StorageError>;
pub fn load(path: &str) -> Result<PhoneBook, StorageError>;
pub fn save(book: &PhoneBook, path: &str) -> Result<(), StorageError>;
```

Почему код устроен именно так? Три модуля содержат **логику** и ничего не знают о клавиатуре, поэтому тесты
могут вызывать их напрямую. С терминалом общается только `src/main.rs`. Такое разделение (логика в
библиотеке, тонкий `main`) типично для настоящих программ на Rust. Для `ParseError`,
`PhoneBookError` и `StorageError` нужны сообщения `Display`, совпадающие с контрактом, чтобы
`main` мог печатать `error: {e}`.

Структура проекта:

```text
code/p04-phonebook/
├── .gitignore         ← содержит: /target
├── Cargo.toml
├── README.md
└── src/
    ├── lib.rs         ← объявляет модули; "pub mod phonebook;" ...
    ├── main.rs        ← цикл: прочитать, разобрать, выполнить, напечатать
    ├── phonebook.rs   ← PhoneBook, PhoneBookError, is_valid_phone
    ├── command.rs     ← Command, ParseError, parse_command, execute
    └── storage.rs     ← StorageError, to_text, from_text, load, save
```

В коде Rust библиотечный крейт называется `p04_phonebook` (дефис в имени пакета становится
подчёркиванием, см. [Урок 04.6](06-modules-crates-and-packages.md)). В `main.rs` вы пишете, например,
`use p04_phonebook::{parse_command, execute, PhoneBook};` после того, как реэкспортируете эти имена в `lib.rs` через `pub use`.

### Пошаговый план

Выполняйте вехи **по порядку**. Закончите каждую, выполните её проверки «Готово, когда» и
**сделайте коммит Git**, прежде чем переходить к следующей. Сообщение коммита называет веху в
стиле Conventional Commits из [Урока 01.9](../01-git-and-version-control/09-teamwork-pull-requests-and-conventions.md).

#### M1 — Проект и репозиторий

**Цель:** пустой проект, который собирается, с README и правильным `.gitignore`, закоммиченный в Git.

**Руководство:**

1. Создайте проект: `cd ~/projects/rust && cargo new code/p04-phonebook && cd code/p04-phonebook`.
2. Проверьте, в каком репозитории Git вы находитесь: `git rev-parse --show-toplevel`. Должна
   напечататься папка вашего учебного репозитория из [Урока 01.10](../01-git-and-version-control/10-project-your-learning-repo-and-checkpoint.md). Если вместо этого ошибка («not a git repository»), выполните здесь `git init`.
3. Когда `cargo new` выполняется внутри существующего репозитория Git, он **не** создаёт
   `.gitignore`. Создайте его из единственной строки `/target`, чтобы папка сборки никогда не
   попадала в коммиты. Файл `Cargo.lock` оставьте в Git: этот проект — программа, а программа должна
   коммитить свой lock-файл (объяснение в [Уроке 05.8](../05-rust-intermediate/08-cargo-dependencies-and-versions.md)).
4. Создайте `src/lib.rs` с тремя строками модулей (`pub mod phonebook;`, `pub mod command;`,
   `pub mod storage;`) и тремя пустыми файлами `src/phonebook.rs`, `src/command.rs`, `src/storage.rs`.
5. Напишите короткий `README.md` с заголовком и одним предложением. Вы расширите его в M7.

<details>
<summary>Подсказка</summary>

Команды по порядку: `printf '/target\n' > .gitignore`, `touch src/phonebook.rs src/command.rs src/storage.rs`.
Затем из корня репозитория `git status` должен показать новые файлы, но **не** `target/`.

</details>

**Готово, когда:**

```bash
cargo build
git status --short -uall
git add .
git commit -m "chore: start p04-phonebook"
git log --oneline -1
```

- `cargo build` заканчивается строкой `Finished` и без предупреждений.
- До `git add` команда `git status --short -uall` перечисляет каждый новый файл `code/p04-phonebook`
  с пометкой `??` (например, `?? code/p04-phonebook/.gitignore` и `?? code/p04-phonebook/Cargo.toml`) и
  **ни одного пути, содержащего `target/`**. (Параметр `-uall` показывает каждый неотслеживаемый файл, а не только папку.)
- Последняя команда показывает одну строку: короткий хеш и `chore: start p04-phonebook`.

#### M2 — Тип `PhoneBook`

**Цель:** протестированный `PhoneBook` с `add`, `get`, `delete`, `count_names` и `count_numbers` и проверкой телефонов.

**Руководство:**

- В `src/phonebook.rs` напишите `PhoneBookError` с `Debug`, `PartialEq`, `Display` и `Error` ([Урок 04.10](10-error-handling.md)).
  Текст `Display` должен совпадать с контрактом: `invalid phone '12'`, `alice already has 5550100`, `no entry for 'bob'`.
- Храните контакты в `BTreeMap<String, Vec<String>>`. `BTreeMap` держит имена отсортированными, а это нужно
  `list` и `find` позже. (`HashMap` тоже подойдёт, если вы отсортируете в M3.)
- `is_valid_phone`: уберите необязательный ведущий `+` (`strip_prefix`), затем пройдите по символам: цифра
  считается, `-` допустим, всё остальное делает телефон неверным. В конце потребуйте не менее 7 цифр.
- `add` должен **сначала проверить**, затем привести имя к нижнему регистру, затем через API `entry` (API, Application Programming Interface: функции, которые даёт тип, здесь `entry` и его помощники, урок 04.9) получить вектор
  для этого имени, затем отклонить дубликат, затем сделать `push`. Осторожно: неудачный `add` не должен оставлять
  пустой вектор. Подумайте, когда `entry(...).or_default()` создаёт вектор и где вы проверяете телефон.
- Пишите тесты по ходу дела. По тесту на каждое правило: верные и неверные телефоны; add + get с разным
  регистром имени; порядок номеров; дубликат; неверный телефон не создаёт имя; delete + повторный delete;
  оба счётчика.

<details>
<summary>Подсказка</summary>

Каркас (только форма):

```rust
#[derive(Debug, Default)]
pub struct PhoneBook {
    entries: BTreeMap<String, Vec<String>>,
}

pub fn add(&mut self, name: &str, phone: &str) -> Result<(), PhoneBookError> {
    if !is_valid_phone(phone) { return Err(PhoneBookError::InvalidPhone(phone.to_string())); }
    let name = name.to_lowercase();
    let phones = self.entries.entry(name.clone()).or_default();
    // if phones.contains(&phone.to_string()) { return Err(...Duplicate...) }
    // phones.push(...); Ok(())
}
```

`#[derive(Default)]` даёт вам `PhoneBook::default()`, поэтому `new()` может просто вызывать его.
`contains` нужен `&String`, поэтому передайте `&phone.to_string()`.
Для проверки дубликата `name` нужен и после того, как он использован для `entry`, поэтому в каркасе он
клонируется (`name.clone()`). В тестах `book.get("alice").unwrap()` возвращает `&[String]`; сравнивайте с
`["a", "b"]`.

</details>

**Готово, когда:**

```bash
cargo test
```

печатает список ваших тестов, все `ok`, и `test result: ok.` минимум с **6 passed** и `0 failed`. Затем:

```bash
cargo clippy --all-targets -- -D warnings
cargo fmt --check
git add . && git commit -m "feat: add PhoneBook with add, get, delete and counts"
```

`clippy` и `fmt --check` ничего не печатают (или печатают `Finished`) и завершаются с кодом 0.

#### M3 — Отсортированные `list` и `find`

**Цель:** `PhoneBook::list` и `PhoneBook::find`, с сортировкой и без учёта регистра в префиксе.

**Руководство:**

- `list` возвращает все контакты, отсортированные по имени. С `BTreeMap` это просто порядок обхода.
- `find(prefix)`: приведите префикс к нижнему регистру, пройдите по ключам, оставьте те, для которых
  `starts_with` истинно.
- Тесты: добавьте `carol`, `alice`, `alan` в таком порядке; `list` даёт `alan`, `alice`, `carol`;
  `find("AL")` даёт `alan`, `alice`; `find("z")` пуст; `list` пустой книги пуст.

<details>
<summary>Подсказка</summary>

`self.entries.iter().collect()` даёт `Vec<(&String, &Vec<String>)>` уже в порядке ключей. Для `find`
создайте `Vec<&String>`, пройдите циклом по `self.entries.keys()` и сделайте `push` подходящих.

</details>

**Готово, когда:** `cargo test` проходит с новыми тестами (минимум 2 новых), и

```bash
git add . && git commit -m "feat: add sorted list and find"
```

#### M4 — Разбор команд

**Цель:** `Command`, `ParseError` и `parse_command` в `src/command.rs`, полностью протестированные, без кода ввода/вывода.

**Руководство:**

- `parse_command(line)` получает строку, которая **не пустая**. Разбейте её на слова через `split_whitespace` и
  соберите в `Vec<&str>` ([Урок 04.8](08-strings-in-depth.md)).
- Приведите первое слово к нижнему регистру и сделайте `match` по нему вместе с числом слов. `match` по кортежу вроде
  `(word.as_str(), words.len())` позволяет писать ветки `("add", 3)` для правильной формы и
  `("add", _)` для неправильной формы (ошибка использования).
- `ParseError::Usage` несёт текст использования; `ParseError::Unknown` несёт слово как введено. Реализуйте
  `Display` по контракту: `usage: add <name> <phone>` и `unknown command 'frobnicate' (type help)`.
- Тесты: один тест разбирает каждую команду; один — с лишними пробелами вокруг слов; по одному на каждый вид
  неверного числа аргументов; один на неизвестную команду; один на `QUIT` в верхнем регистре.

<details>
<summary>Подсказка</summary>

Ядро функции:

```rust
let words: Vec<&str> = line.split_whitespace().collect();
let Some(first) = words.first() else { /* the line was blank */ };
match (first.to_lowercase().as_str(), words.len()) {
    ("add", 3) => Ok(Command::Add { name: words[1].to_string(), phone: words[2].to_string() }),
    ("add", _) => Err(ParseError::Usage("add <name> <phone>")),
    // ... get, del, find, list, count, help, quit ...
    _ => Err(ParseError::Unknown(first.to_string())),
}
```

Образец с альтернативами вроде `("list" | "count" | "help" | "quit", _)` позволяет одной ветке покрыть
несколько слов. Ветки точной формы (`("list", 1)`) поставьте до неё.

</details>

**Готово, когда:** `cargo test` проходит минимум с **4 новыми** тестами для `parse_command`. Затем
`cargo clippy --all-targets -- -D warnings` и `cargo fmt --check` молчат, и:

```bash
git add . && git commit -m "feat: parse commands into the Command enum"
```

#### M5 — Выполнение команд и цикл

**Цель:** `execute` и `main`: программа работает из конвейера и из терминала. (Файла пока нет.)

**Руководство:**

- `execute(&mut book, command) -> String` возвращает текст, который нужно напечатать для команды, ровно как в
  контракте. Для `list` и `find` соедините строки через `"\n"`. Тестируйте её **без** `main`: разберите строку,
  выполните её, сравните строку.
- `src/lib.rs` реэкспортирует имена, нужные `main`: `pub use command::{execute, parse_command, Command};` и
  `pub use phonebook::PhoneBook;`.
- `main`: создайте `PhoneBook`; заведите `String` для строки; `loop`: если ввод — терминал, напечатайте
  приглашение и сделайте flush; очистите буфер строки и вызовите `stdin.read_line(&mut line)`; результат `Ok(0)` значит EOF, поэтому
  `break`; пустая строка значит `continue`; иначе `parse_command`; `Quit` завершает цикл; команда выполняется, а её текст
  печатается через `println!`; `ParseError` печатается как `error: {e}`.
- Проверка терминала: `use std::io::IsTerminal;` и затем `std::io::stdin().is_terminal()`.
- Flush: `use std::io::Write;` и `std::io::stdout().flush()` после `print!("> ")`. Без flush приглашение
  может появиться с опозданием, потому что `print!` без перевода строки буферизуется.
- Если чтение ввода не удалось, напечатайте `error: cannot read input: {e}` в stderr и `exit(1)`.

<details>
<summary>Подсказка</summary>

Цикл в общих чертах:

```rust
let stdin = std::io::stdin();
let interactive = stdin.is_terminal();
let mut line = String::new();
loop {
    if interactive { print!("> "); std::io::stdout().flush().expect("cannot flush stdout"); }
    line.clear();                              // read_line appends, so clear first
    let bytes_read = match stdin.read_line(&mut line) { Ok(n) => n, Err(e) => { /* print, exit(1) */ } };
    if bytes_read == 0 { break; }
    if line.trim().is_empty() { continue; }
    match parse_command(&line) {
        Ok(Command::Quit) => break,
        Ok(command) => println!("{}", execute(&mut book, command)),
        Err(e) => println!("error: {e}"),
    }
}
```

Обратите внимание: ошибки разбора здесь печатаются в **stdout** (это ответы на языке программы,
поэтому контракт требует их в обычном выводе), а ошибки файла в M6 идут в stderr.

</details>

**Готово, когда:** сначала главный приёмочный тест, сессия через конвейер:

```bash
printf 'add Alice +1-555-0100\nadd alice +1-555-0199\nget ALICE\ncount\ndel bob\nfrobnicate\nquit\n' | cargo run -q
```

должна напечатать ровно:

```text
ok: added +1-555-0100 to alice
ok: added +1-555-0199 to alice
alice: +1-555-0100, +1-555-0199
1 names, 2 numbers
error: no entry for 'bob'
error: unknown command 'frobnicate' (type help)
```

Затем сессия побольше, которая показывает остальные команды:

```bash
printf 'add alice +1-555-0100\nadd alice +1-555-0100\nadd bob 12\nadd bob\n\n   \nadd Carol 5550123\nadd alan +998-90-111-22-33\nlist\nfind AL\nfind z\ndel carol\nlist\ncount\nhelp\nquit\nadd never 1234567\n' | cargo run -q
```

должна напечатать ровно:

```text
ok: added +1-555-0100 to alice
error: alice already has +1-555-0100
error: invalid phone '12'
error: usage: add <name> <phone>
ok: added 5550123 to carol
ok: added +998-90-111-22-33 to alan
alan: +998-90-111-22-33
alice: +1-555-0100
carol: 5550123
alan
alice
(no matches)
ok: deleted carol
alan: +998-90-111-22-33
alice: +1-555-0100
2 names, 2 numbers
commands:
  add <name> <phone>
  get <name>
  del <name>
  list
  find <prefix>
  count
  help
  quit
```

(Последняя строка, `add never ...`, идёт после `quit`, поэтому она не должна выполниться.) Проверьте и
граничные случаи:

```bash
printf 'list\nfind a\ncount\n' | cargo run -q
printf 'add x 1234567' | cargo run -q
```

```text
(empty)
(no matches)
0 names, 0 numbers
ok: added 1234567 to x
```

У второй команды нет завершающего перевода строки и нет `quit`: конец ввода должен корректно завершить
программу. Запустите `cargo run` без конвейера и убедитесь, что видите приглашение `> `. Введите `help`, затем
`quit`. Наконец, `cargo test`, `cargo clippy --all-targets -- -D warnings`, `cargo fmt --check` и:

```bash
git add . && git commit -m "feat: add the command loop"
```

#### M6 — Загрузка и сохранение (в фича-ветке)

**Цель:** `cargo run -- contacts.txt` помнит контакты между запусками. Вы разрабатываете это в
**фича-ветке** и сливаете её, как это сделала бы команда.

**Руководство:**

1. Создайте ветку: `git switch -c feature/save-load`.
2. В `src/storage.rs` напишите `StorageError` (`Io` и `BadLine`) с `Display`, `Error` и `From<std::io::Error>`
   ([Урок 04.10](10-error-handling.md)). Сообщения: `file error: <io error>` и `bad data on line <N>: <reason>`.
3. `to_text(&PhoneBook) -> String` следует формату файла из контракта. `from_text(&str) -> Result<PhoneBook,
   StorageError>` читает его: для каждой непустой строки первое слово — имя, а каждое следующее слово — телефон для
   `add`; если `add` не удался, верните `BadLine` с номером строки (счёт с 1) и текстом ошибки (`e.to_string()`).
4. `load(path)` использует `std::fs::read_to_string`. Если `kind()` ошибки равен `std::io::ErrorKind::NotFound`, верните
   пустую книгу; любая другая ошибка становится `StorageError::Io`. `save(&book, path)` использует `std::fs::write(path, to_text(book))`.
5. Тесты для чистых функций (файлы не нужны): проверка «туда и обратно»: `to_text`, затем `from_text`, затем `to_text` даёт тот же текст;
   плохой телефон на строке 2 сообщается как строка 2; `load` несуществующего файла даёт пустую книгу.
6. В `main`: прочитайте имя файла через `std::env::args().nth(1)`. Это даёт `Option<String>`: `args()` выдаёт сначала
   имя программы, затем каждый аргумент, а `nth(1)` берёт второй элемент или `None`, если его нет. Если путь
   задан, сделайте `load` в начале (при ошибке напечатайте `error: {e}` в stderr и `exit(1)`). Когда цикл закончится, сделайте `save`
   (та же обработка ошибок).
7. Слияние: `git switch main`, затем `git merge --no-ff feature/save-load`, затем удалите ветку.

<details>
<summary>Подсказка</summary>

`match` с охранным условием — ясный способ считать «не найдено» успехом:

```rust
match std::fs::read_to_string(path) {
    Ok(text) => from_text(&text),
    Err(e) if e.kind() == std::io::ErrorKind::NotFound => Ok(PhoneBook::new()),
    Err(e) => Err(StorageError::Io(e)),
}
```

В `main` держите путь как `Option<String>` и используйте `match &path { Some(p) => ..., None => ... }`,
чтобы выбрать между загрузкой файла и началом с `PhoneBook::new()`. Небольшая вспомогательная функция,
которая сохраняет книгу и завершает программу при ошибке, делает `main` читаемым.

</details>

**Готово, когда:**

```bash
rm -f contacts.txt
printf 'add alice +1-555-0100\nadd Bob 5550123\nadd alice +1-555-0199\nquit\n' | cargo run -q -- contacts.txt
cat contacts.txt
```

печатает

```text
ok: added +1-555-0100 to alice
ok: added 5550123 to bob
ok: added +1-555-0199 to alice
alice +1-555-0100 +1-555-0199
bob 5550123
```

(последние две строки — от `cat`). Второй запуск должен помнить данные и сохранить изменения:

```bash
printf 'list\ndel bob\nadd zed 7777777\nquit\n' | cargo run -q -- contacts.txt
cat contacts.txt
```

```text
alice: +1-555-0100, +1-555-0199
bob: 5550123
ok: deleted bob
ok: added 7777777 to zed
alice +1-555-0100 +1-555-0199
zed 7777777
```

Затем проверьте путь ошибки и код завершения:

```bash
printf 'alice 1111111\nbob 12\n' > bad.txt
cargo run -q -- bad.txt < /dev/null
echo $?
rm bad.txt contacts.txt
```

```text
error: bad data on line 2: invalid phone '12'
1
```

Не коммитьте `contacts.txt` (вы его удалили). Теперь слияние:

```bash
cargo test && cargo clippy --all-targets -- -D warnings && cargo fmt --check
git add . && git commit -m "feat: load and save the phone book"
git switch main
git merge --no-ff feature/save-load
git branch -d feature/save-load
git log --oneline --graph
```

`git log --oneline --graph` показывает ветку и коммит слияния примерно так (ваши хеши будут другими):

```text
*   890f511 Merge branch 'feature/save-load'
|\
| * f7a35b3 feat: load and save the phone book
|/
* 40e2830 feat: add the command loop
* 64b0477 feat: parse commands into the Command enum
...
```

(Если Git откроет редактор для сообщения слияния, сохраните его и закройте.)

#### M7 — Качество, README и тег релиза

**Цель:** проект чистый, задокументированный и помечен тегом `v0.1.0`.

**Руководство:**

- Запустите `cargo fmt` (он переписывает файлы в стандартном стиле), затем `cargo clippy --all-targets -- -D warnings`. Параметр
  `-D warnings` превращает каждое предупреждение в ошибку. Исправьте всё, о чём сообщает Clippy; читайте каждое
  сообщение, потому что Clippy часто объясняет идиоматичный способ.
- Найдите в коде `unwrap()` и `expect(` вне тестов. Каждый из них должен быть случаем, про который вы можете доказать,
  что он не упадёт (или его нужно заменить нормальной обработкой ошибок). Объясните оставшиеся `expect` хорошим сообщением.
- Расширьте `README.md`: что программа делает, как её запустить (`cargo run -- contacts.txt`), таблицу команд, формат файла
  и как запускать тесты.
- Поставьте тег релиза.

<details>
<summary>Подсказка</summary>

В хорошем README такие заголовки: *Что это*, *Запуск*, *Команды*, *Формат файла*, *Тесты*.
Создайте аннотированный тег командой `git tag -a v0.1.0 -m "p04-phonebook 0.1.0"` и посмотрите его через `git tag`
и `git show v0.1.0 --stat`.

</details>

**Готово, когда:**

```bash
cargo fmt --check && cargo clippy --all-targets -- -D warnings && cargo test
git status --short
git add . && git commit -m "docs: write the README and tidy the code"
git tag -a v0.1.0 -m "p04-phonebook 0.1.0"
git tag
```

- первая строка не печатает предупреждений и заканчивается на `test result: ok.`;
- `git status --short` до коммита перечисляет только изменения README и кода (без `target/` и `contacts.txt`);
- `git tag` печатает `v0.1.0`;
- `git log --oneline` показывает по одному коммиту на веху (M1–M7) с сообщениями в стиле Conventional Commits.

Если вы используете удалённый репозиторий (GitHub или GitLab, [Урок 01.8](../01-git-and-version-control/08-remotes-github-and-gitlab.md)), отправьте ветку и тег: `git push` и `git push --tags`.

### Дополнительные цели

1. **Имена с пробелами:** примите `add "Ali Valiyev" +998-90-123-45-67` (кавычки вокруг имени). Подумайте, как должны
   измениться формат файла и `parse_command`.
2. **`rename <old> <new>`:** перенесите все номера на новое имя. А что, если новое имя уже существует?
3. **`undo`:** отмена последнего изменения. Храните `Vec` обратных операций (перечисление!) и берите из него через `pop`.
4. **Интеграционный тест:** запустите настоящую программу из теста в `tests/cli.rs` и сравните её вывод с контрактом.
5. **Код завершения при `del` неизвестных имён** в неинтерактивном запуске (вернуть 3 в конце, если какая-то команда не удалась). Хорошая ли это идея? Обсудите сами с собой, затем решите.
6. **Лучшие ошибки:** замените ручной код `Display` и `Error` крейтом `thiserror` после [Урока 05.11](../05-rust-intermediate/11-practical-crates-serde-clap-errors.md).

<details>
<summary>Подсказка к дополнительной цели 4</summary>

Cargo передаёт интеграционным тестам путь к скомпилированной программе в `env!("CARGO_BIN_EXE_p04-phonebook")`. Запустите её через
`std::process::Command`, задайте `.stdin(Stdio::piped())` и `.stdout(Stdio::piped())`, запишите ввод через
`write_all`, вызовите `wait_with_output()` и сравните `String::from_utf8(output.stdout)` с ожидаемым текстом.
Помните, что приглашение не печатается, потому что stdin — конвейер.

</details>

## Частые ошибки

**1. Печать приглашения в конвейере.** Ожидаемый вывод приёмочных тестов не содержит `> `. Печатайте его только если `stdin.is_terminal()`.

**2. Забыли flush.** У `print!("> ")` нет перевода строки, поэтому текст может остаться в буфере, и пользователь печатает, не видя приглашения. Вызовите `io::stdout().flush()`.

**3. `read_line` дописывает.** Если вы никогда не вызываете `line.clear()`, вторая команда содержит ещё и первую.

**4. Не обрезали строку.** `read_line` сохраняет перевод строки в конце. `split_whitespace` убирает его для разборщика, но правильная проверка пустой строки — `line.trim().is_empty()`.

**5. Опора на порядок `HashMap`.** Если вы использовали `HashMap`, то `list` и `find` обязаны сортировать; иначе вывод меняется от запуска к запуску, и приёмочный тест падает случайным образом.

**6. Недоделанный `add`.** Если `entry(...).or_default()` выполняется до проверки, неверный телефон может оставить имя с пустым списком номеров (`count` скажет `1 names, 0 numbers`). Сначала проверяйте.

**7. Коммит `target/` или `contacts.txt`.** Проверяйте `git status` перед каждым `git add .`. Результаты сборки и тестовые данные в Git не нужны.

**8. `unwrap()` для ввода пользователя или файла.** Опечатка уронит программу. Обрабатывайте каждый `Result` (Урок 04.10).

## Вопросы для самопроверки

1. Почему проект держит логику в библиотечном крейте, а в `main.rs` — только код для терминала?

<details>
<summary>Ответ</summary>

Логику в библиотеке можно тестировать, вызывая функции напрямую, не вводя ничего в терминал и не читая
файлы. `main.rs` остаётся маленьким (прочитать строку, вызвать библиотеку, напечатать ответ), поэтому
непротестированного кода мало.

</details>

2. Как программа узнаёт, печатать ли приглашение `> `?

<details>
<summary>Ответ</summary>

Она спрашивает `std::io::stdin().is_terminal()` (трейт `IsTerminal`). Если печатает человек — это
терминал, и приглашение показывается; конвейер или файл терминалом не являются, и приглашение
пропускается, так что скрипты получают чистый вывод.

</details>

3. Что возвращает `read_line` в конце ввода и что вы делаете?

<details>
<summary>Ответ</summary>

`Ok(0)`: прочитано ноль байт, это значит EOF. Цикл должен сделать `break`. (Пустая строка — это
`Ok(1)` из-за символа перевода строки, а не `Ok(0)`.)

</details>

4. Почему в `load` отсутствие файла данных не считается ошибкой?

<details>
<summary>Ответ</summary>

При первом запуске программы файла ещё нет, и это нормальная ситуация. Контракт говорит, что
отсутствующий файл означает пустую книгу, а файл создаётся при сохранении. Другие ошибки
ввода/вывода (нет прав, вместо файла папка) — настоящие ошибки.

</details>

5. Зачем в M6 мы использовали фича-ветку и `git merge --no-ff`?

<details>
<summary>Ответ</summary>

Ветка держит незаконченную работу в стороне от `main`, пока она не пройдёт все проверки. `--no-ff`
всегда создаёт коммит слияния, поэтому в истории видно, что группа коммитов была одной новой возможностью.
Это тренировка командной работы.

</details>

6. Что делает `cargo clippy --all-targets -- -D warnings`?

<details>
<summary>Ответ</summary>

Запускает линтер Clippy на всём коде, включая тесты, а `-D warnings` превращает каждое
предупреждение в ошибку, поэтому команда завершается неудачей (код не 0), пока все предупреждения
не исправлены. Так команды держат кодовую базу чистой в CI.

</details>

## Итоги

- Настоящая программа строится маленькими проверяемыми шагами: сначала контракт, затем вехи, и по коммиту на каждую.
- Держите логику (`lib.rs` и модули) отдельно от ввода/вывода (`main.rs`), чтобы логику было легко тестировать.
- Превращайте текст в перечисление разборщиком (`parse_command`), затем выполняйте перечисление одним `match`.
- Используйте `BTreeMap`, когда нужен отсортированный вывод, проверяйте до изменения данных и храните ошибки как перечисления с `Display`.
- Терминал и конвейер различаются: печатайте приглашение только для терминала, делайте flush после `print!` и считайте `Ok(0)` от `read_line` концом ввода.
- Отсутствие файла может быть нормальной ситуацией; остальные ошибки ввода/вывода — настоящие, о них сообщают в stderr с кодом завершения 1.
- Работайте по-профессиональному: `.gitignore` для `target/`, Conventional Commits, фича-ветка со слиянием, тег и чистые `cargo fmt`, `cargo clippy` и `cargo test`.

## Источники и что почитать

- [The Rust Programming Language — 12. An I/O Project: Building a Command Line Program](https://doc.rust-lang.org/book/ch12-00-an-io-project.html) — похожий проект в официальной книге (`minigrep` вы напишете на Этапе 05).
- [std::io::Stdin](https://doc.rust-lang.org/std/io/struct.Stdin.html) — `read_line`, `lines` и другие способы читать ввод.
- [std::io::IsTerminal](https://doc.rust-lang.org/std/io/trait.IsTerminal.html) — как отличить терминал от конвейера.
- [std::fs](https://doc.rust-lang.org/std/fs/index.html) — `read_to_string`, `write` и другие функции работы с файлами.
- [The Cargo Book — Cargo.toml vs Cargo.lock](https://doc.rust-lang.org/cargo/guide/cargo-toml-vs-cargo-lock.html) — почему программа коммитит свой `Cargo.lock`.
- [Pro Git — 3.2 Git Branching: Basic Branching and Merging](https://git-scm.com/book/en/v2/Git-Branching-Basic-Branching-and-Merging) — ветки и слияния подробно.
- [Conventional Commits 1.0.0](https://www.conventionalcommits.org/en/v1.0.0/) — спецификация стиля сообщений коммитов.
- [Clippy documentation](https://doc.rust-lang.org/clippy/) — что проверяет Clippy и как его настраивать.

---

[🏠 Начало](../START-HERE.md) · [← 04.10 Обработка ошибок](10-error-handling.md) · [📋 Этап 04](README.md) · [04.12 Контрольная точка →](12-checkpoint.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
