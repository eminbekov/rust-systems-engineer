<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 10.4 Проверка ввода: никогда не доверяйте вводу](04-input-validation.md) · [📋 Этап 10](README.md) · [10.6 Аутентификация и хеширование паролей →](06-authentication-and-password-hashing.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/10-security-fundamentals/05-injection-attacks.md) <!-- nav -->

# Урок 10.5 — Атаки внедрения: SQL, команды и пути

> **Этап 10 — Основы безопасности** · Урок 5 из 15 · ⏱ около 135 минут

## Что вы узнаете

- Что такое атака внедрения: данные, которые получатель принимает за код.
- Внедрение SQL с настоящей безвредной демонстрацией и лекарство: параметризованные запросы.
- Внедрение команд, внедрение аргументов и как запускать программы без оболочки.
- Обход пути (`../`) и как удерживать доступ к файлам внутри одной папки.
- Экранирование вывода для HTML и общее правило, покрывающее любой вид внедрения.

## Перед началом

- Вы прошли [Урок 10.4 — Проверка ввода](04-input-validation.md).
- Вы знаете `std::fs` и `Path` (в [Уроке 04.10](../04-structs-enums-collections-errors/10-error-handling.md) использовались файлы) и `Result`.
- Создайте пакет. Порядок важен: `cargo new code/p10-injection`, затем `cd code/p10-injection`, затем `cargo add rusqlite@0.43 --features bundled`, затем `mkdir src/bin` и `rm src/main.rs`. Фича `bundled` заставляет Cargo скомпилировать движок базы данных SQLite внутрь вашей программы, поэтому ничего устанавливать не нужно. Первая сборка занимает около минуты.
- **Безопасность:** все атаки в этом уроке направлены только на вашу собственную программу, в папке `sandbox`, а нагрузка только печатает слово `INJECTED`.

## Новые слова

| Слово | Значение |
|---|---|
| **внедрение** (injection) | Атака, при которой ввод помещается в команду, запрос или документ так, что меняется их структура, и получатель принимает данные за код. |
| **SQL** (Structured Query Language, язык структурированных запросов) | Язык, на котором общаются с большинством баз данных: `SELECT name FROM users WHERE id = 5`. |
| **внедрение SQL** (SQL injection) | Внедрение в команду SQL. Атакующий может читать, менять или удалять данные. |
| **SQLite** | Небольшой движок базы данных, который живёт внутри вашей программы и хранит данные в файле или в памяти. |
| **параметризованный запрос** (parameterized query, или *prepared statement*) | Команда SQL с местозаполнителями (`?1`) вместо значений. Значения отправляются отдельно и никогда не читаются как SQL. |
| **командная оболочка** (shell) | Программа (`sh`, `bash`, `zsh`), которая читает строку текста, делит её на слова и запускает программы. Она также понимает `;`, `|`, `&&`, `$(…)` и многое другое. |
| **внедрение команд** (command injection) | Внедрение в текст, который выполнит оболочка. |
| **внедрение аргументов** (argument injection) | Ввод передаётся программе как аргумент и начинается с `-`, поэтому программа принимает его за параметр. |
| **обход пути** (path traversal) | Использование `..` (родительской папки) или абсолютного пути, чтобы добраться до файлов вне папки, которую вы хотели разрешить. |
| **символическая ссылка** (symbolic link, symlink) | Особый файл, который указывает на другой файл или папку. |
| **XSS** (Cross-Site Scripting, межсайтовый скриптинг) | Внедрение скрипта в веб-страницу, из-за чего браузер посетителя выполняет код атакующего. |
| **HTML** (HyperText Markup Language, язык гипертекстовой разметки) | Язык веб-страниц с тегами вроде `<p>` и `<script>`. |

## 1. Одна идея за любым внедрением

Компьютеры получают **команды** (код) и **данные** (значения). Беда начинается, когда программа собирает один
текст из обоих и отправляет его интерпретатору, который не может понять, где кончились данные:

```text
"SELECT ... WHERE name = '"  +  name  +  "'"
 ───────── code ─────────      data      code
```

Если `name` содержит кавычку, данные заканчиваются раньше, и остальное читается как код. То же верно для
команды оболочки, пути к файлу, HTML-страницы, строки журнала. Лекарство — всегда одно из следующих, в таком
порядке предпочтения:

1. **Держите данные и код в разных каналах.** Параметры для SQL; список аргументов вместо текста для
   оболочки. Интерпретатору не нужно гадать.
2. **Если приходится собирать текст, ограничьте части списком разрешённого** (Урок 10.4) **и экранируйте** именно
   для этого языка.
3. **Используйте наименьшие привилегии** (Урок 10.1), чтобы успешное внедрение могло сделать немногое:
   учётная запись базы данных, которая не может удалять таблицы, процесс, который не может читать весь диск.

## 2. Внедрение SQL

Создайте `src/bin/sql.rs`. Он строит базу данных в памяти с тремя пользователями и ищет по ней дважды: один
раз неправильно и один раз правильно:

```rust
use rusqlite::{Connection, Result, params};

fn setup() -> Result<Connection> {
    let conn = Connection::open_in_memory()?;
    conn.execute_batch(
        "CREATE TABLE users (name TEXT NOT NULL, role TEXT NOT NULL);
         INSERT INTO users VALUES ('ana', 'user'), ('bob', 'user'), ('root', 'admin');",
    )?;
    Ok(conn)
}

/// BROKEN: the user's text becomes part of the SQL command.
fn find_users_bad(conn: &Connection, name: &str) -> Result<Vec<String>> {
    let sql = format!("SELECT name || ':' || role FROM users WHERE name = '{name}'");
    println!("  sql: {sql}");
    let mut stmt = conn.prepare(&sql)?;
    stmt.query_map([], |row| row.get(0))?.collect()
}

/// FIXED: the text travels separately, as a parameter (?1).
fn find_users(conn: &Connection, name: &str) -> Result<Vec<String>> {
    let sql = "SELECT name || ':' || role FROM users WHERE name = ?1";
    println!("  sql: {sql}   (?1 = {name:?})");
    let mut stmt = conn.prepare(sql)?;
    stmt.query_map(params![name], |row| row.get(0))?.collect()
}

fn main() -> Result<()> {
    let conn = setup()?;
    let attack = "nobody' OR '1'='1";

    println!("normal input, broken code:");
    println!("  -> {:?}", find_users_bad(&conn, "ana")?);
    println!("attack input, broken code:");
    println!("  -> {:?}", find_users_bad(&conn, attack)?);
    println!("attack input, fixed code:");
    println!("  -> {:?}", find_users(&conn, attack)?);
    Ok(())
}
```

Запустите `cargo run --bin sql`:

```text
normal input, broken code:
  sql: SELECT name || ':' || role FROM users WHERE name = 'ana'
  -> ["ana:user"]
attack input, broken code:
  sql: SELECT name || ':' || role FROM users WHERE name = 'nobody' OR '1'='1'
  -> ["ana:user", "bob:user", "root:admin"]
attack input, fixed code:
  sql: SELECT name || ':' || role FROM users WHERE name = ?1   (?1 = "nobody' OR '1'='1")
  -> []
```

**Что происходит:**

1. `Connection::open_in_memory()` создаёт одноразовую базу данных SQLite в памяти. `execute_batch` выполняет
   несколько команд SQL за один вызов и строит таблицу `users`.
2. `find_users_bad` использует `format!`, чтобы **вставить** имя в текст SQL. Для `ana` всё в порядке.
3. Текст атаки — `nobody' OR '1'='1`. После вставки команда выглядит как
   `... WHERE name = 'nobody' OR '1'='1'`. Первая кавычка закрывает строку, а `OR '1'='1'` — условие, которое
   всегда истинно, поэтому возвращается **каждая строка**, включая администратора.
4. `find_users` пишет в SQL местозаполнитель `?1` и передаёт значение через `params![name]`. База данных
   получает команду и значение **раздельно**. Весь текст атаки, вместе с кавычками, — просто имя, и такого
   имени ни у кого нет: результат пуст.

Та же дыра может делать гораздо больше в других местах: войти без пароля (`admin'--` превращает остаток
команды в комментарий), читать другие таблицы через `UNION` или менять данные. Некоторые интерфейсы баз данных
даже выполняют несколько команд из одной строки. Одна кавычка во вводе никогда не должна менять то, что делает
программа.

**Чего не может местозаполнитель.** Параметр заменяет **значение**, но никогда не имя таблицы, имя столбца или
`ASC`/`DESC`. Для них выбирайте из фиксированного списка. `match` — это список разрешённого; неизвестное слово
становится `None`, и вызывающий код должен отклонить запрос:

```rust
fn order_clause(requested: &str) -> Option<&'static str> {
    match requested {
        "name" => Some("ORDER BY name"),
        "role" => Some("ORDER BY role"),
        _ => None,
    }
}
```

Программа вставляет только фиксированные тексты из своего кода, никогда не текст пользователя.

> 🔒 **Безопасность:** *всегда* используйте параметры для значений, в любой библиотеке баз данных, даже когда
> «ввод — всего лишь число». Также дайте приложению учётную запись базы данных только с нужными правами
> (читать и писать свои таблицы, но не `DROP`).

## 3. Внедрение команд

Программы часто запускают другие программы. В Rust это делается через `std::process::Command`. Опасный
способ — отдать *оболочке* один большой текст. Сохраните `src/bin/command.rs`:

```rust
use std::process::Command;

/// BROKEN: the name is pasted into a text that a shell will interpret.
fn greet_bad(name: &str) {
    let script = format!("echo Hello, {name}");
    println!("  running: sh -c {script:?}");
    let output = Command::new("sh")
        .arg("-c")
        .arg(&script)
        .output()
        .expect("cannot start sh");
    print!("{}", String::from_utf8_lossy(&output.stdout));
}

/// FIXED: no shell. The name is one argument, whatever characters it contains.
fn greet(name: &str) {
    println!(
        "  running: echo with one argument {:?}",
        format!("Hello, {name}")
    );
    let output = Command::new("echo")
        .arg(format!("Hello, {name}"))
        .output()
        .expect("cannot start echo");
    print!("{}", String::from_utf8_lossy(&output.stdout));
}

fn main() {
    let attack = "ana; echo INJECTED";
    println!("broken:");
    greet_bad(attack);
    println!("fixed:");
    greet(attack);
}
```

```text
broken:
  running: sh -c "echo Hello, ana; echo INJECTED"
Hello, ana
INJECTED
fixed:
  running: echo with one argument "Hello, ana; echo INJECTED"
Hello, ana; echo INJECTED
```

**Что происходит:**

1. `greet_bad` строит текст `echo Hello, ana; echo INJECTED` и запускает его через `sh -c`, что означает
   «интерпретируй этот текст как команду оболочки». Оболочка читает `;` как конец первой команды, поэтому
   запускает **вторую команду**, которую программист не писал. Вместо `echo INJECTED` атакующий мог бы
   подставить любую команду.
2. `greet` запускает `echo` напрямую: `Command::new("echo").arg(...)`. **Оболочки нет**, поэтому `;` — обычный
   символ. Весь текст — один аргумент, и `echo` его печатает. Ничего не запускается.

Правило: **никогда не собирайте текст для оболочки из ввода.** Запускайте нужную программу напрямую и
передавайте каждый аргумент через `.arg()` или `.args()`. Если вам кажется, что нужна оболочка, почти всегда
нужен лучший дизайн.

### Внедрение аргументов

Даже без оболочки *программа* может принять аргумент, начинающийся с `-`, за **параметр**. Сохраните
`src/bin/option.rs`. Он создаёт небольшую папку `sandbox/docs/a.txt` и показывает список для «имени файла»,
введённого пользователем:

```rust
use std::fs;
use std::path::Path;
use std::process::Command;

fn list(target: &str, with_separator: bool) {
    let mut command = Command::new("ls");
    if with_separator {
        command.arg("--");
    }
    let output = command.arg(target).output().expect("cannot start ls");
    print!("{}", String::from_utf8_lossy(&output.stdout));
    print!("{}", String::from_utf8_lossy(&output.stderr));
}

fn main() {
    // A small folder to look at: sandbox/docs/a.txt
    let _ = fs::remove_dir_all("sandbox");
    fs::create_dir_all("sandbox/docs").unwrap();
    fs::write("sandbox/docs/a.txt", "hello").unwrap();
    std::env::set_current_dir(Path::new("sandbox")).unwrap();

    println!("expected input \".\":");
    list(".", false);
    println!("attack input \"-R\", no separator:");
    list("-R", false);
    println!("attack input \"-R\", with \"--\":");
    list("-R", true);

    std::env::set_current_dir("..").unwrap();
    fs::remove_dir_all("sandbox").unwrap();
}
```

```text
expected input ".":
docs
attack input "-R", no separator:
docs

./docs:
a.txt
attack input "-R", with "--":
ls: -R: No such file or directory
```

**Что происходит:**

1. Ожидаемый ввод `.` показывает папку: `docs`.
2. Ввод атаки `-R` — не имя файла. Это параметр «рекурсивно», поэтому `ls` показывает ещё и всё внутри `docs`.
   Пользователь изменил то, что делает команда. У других программ параметры могут записывать файлы или
   запускать другие программы.
3. Один `--` перед вводом означает «всё, что дальше, — имя, а не параметр» (соглашение, которому следует
   большинство инструментов командной строки). Теперь `ls` правильно сообщает, что файла `-R` нет. (Точный
   текст ошибки зависит от операционной системы.)
4. В конце программа удаляет `sandbox`.

Другие меры: отклонять ввод, начинающийся с `-` (Урок 10.4: список разрешённого для имён файлов), и
предпочитать библиотеку Rust запуску программы, когда она существует.

## 4. Обход пути

Программа отдаёт файлы из папки, например `sandbox/public`. Посетитель запрашивает имя `../secret.txt`. Если
программа соединяет базовую папку и имя, путь становится `sandbox/public/../secret.txt`, то есть
`sandbox/secret.txt`, вне папки. Сохраните `src/bin/path.rs`:

```rust
use std::fs;
use std::io;
use std::path::{Component, Path, PathBuf};

const BASE: &str = "sandbox/public";

/// BROKEN: joins whatever the user typed.
fn read_bad(name: &str) -> io::Result<String> {
    fs::read_to_string(Path::new(BASE).join(name))
}

/// FIXED, step 1: accept only plain file names and folders. No "..", no "/" at the start.
fn safe_path(name: &str) -> Option<PathBuf> {
    let mut result = PathBuf::from(BASE);
    for part in Path::new(name).components() {
        match part {
            Component::Normal(p) => result.push(p),
            _ => return None, // "..", ".", "/", "C:" ...
        }
    }
    Some(result)
}

/// FIXED, step 2 (defense in depth): after the file system resolved everything,
/// the real path must still be inside the real base folder.
fn read_safe(name: &str) -> Result<String, String> {
    let path = safe_path(name).ok_or("name is not allowed")?;
    let real = fs::canonicalize(&path).map_err(|_| "no such file")?;
    let base = fs::canonicalize(BASE).map_err(|_| "no base folder")?;
    if !real.starts_with(&base) {
        return Err(String::from("path leaves the base folder"));
    }
    fs::read_to_string(real).map_err(|_| String::from("cannot read"))
}

fn main() {
    // Set up: sandbox/public/hello.txt is public, sandbox/secret.txt is not.
    let _ = fs::remove_dir_all("sandbox");
    fs::create_dir_all(BASE).unwrap();
    fs::write("sandbox/public/hello.txt", "hello, visitor").unwrap();
    fs::write("sandbox/secret.txt", "THE SECRET").unwrap();

    println!("broken, \"hello.txt\":       {:?}", read_bad("hello.txt"));
    println!(
        "broken, \"../secret.txt\":   {:?}",
        read_bad("../secret.txt")
    );
    println!("fixed,  \"hello.txt\":       {:?}", read_safe("hello.txt"));
    println!(
        "fixed,  \"../secret.txt\":   {:?}",
        read_safe("../secret.txt")
    );
    println!("fixed,  \"/etc/hosts\":      {:?}", read_safe("/etc/hosts"));

    // A symbolic link inside the public folder that points outside:
    #[cfg(unix)]
    {
        std::os::unix::fs::symlink("../secret.txt", "sandbox/public/link.txt").unwrap();
        println!(
            "step 1 only, \"link.txt\":   {:?}",
            safe_path("link.txt").map(fs::read_to_string)
        );
        println!("fixed,  \"link.txt\":        {:?}", read_safe("link.txt"));
    }
    fs::remove_dir_all("sandbox").unwrap();
}
```

```text
broken, "hello.txt":       Ok("hello, visitor")
broken, "../secret.txt":   Ok("THE SECRET")
fixed,  "hello.txt":       Ok("hello, visitor")
fixed,  "../secret.txt":   Err("name is not allowed")
fixed,  "/etc/hosts":      Err("name is not allowed")
step 1 only, "link.txt":   Some(Ok("THE SECRET"))
fixed,  "link.txt":        Err("path leaves the base folder")
```

**Что происходит:**

1. `read_bad` соединяет базовую папку и имя: атака `../secret.txt` возвращает `THE SECRET`.
2. `safe_path` смотрит на имя **по компонентам**. `Path::components()` делит путь на части: `Normal` для
   обычного имени, `ParentDir` для `..`, `RootDir` для начального `/` и так далее. Мы принимаем только части
   `Normal`. Имя с `..`, начальным `/` или префиксом диска даёт `None`.
3. `read_safe` добавляет второй слой. `fs::canonicalize` спрашивает у файловой системы **настоящий** путь: он
   убирает `..` и **следует по символическим ссылкам**. Затем `starts_with` проверяет, что настоящий путь всё
   ещё внутри настоящей базовой папки. (`canonicalize` завершается ошибкой, если файла нет; здесь это
   нормально: нет файла — нет ответа.)
4. Символическая ссылка `link.txt` лежит внутри `public` и указывает на `../secret.txt`. Имя проходит шаг 1
   (обычное имя!), поэтому чтение только с этой проверкой возвращает секрет. Шаг 2 видит, что настоящий путь
   выходит из папки, и отказывает.
5. `#[cfg(unix)]` компилирует эту часть только на macOS и Linux, где существует `symlink`.

> 🔒 **Безопасность:** проверка с последующим использованием («проверить путь, потом открыть») оставляет
> маленький промежуток, в котором файл можно подменить ссылкой (*гонка*). Для программ с высоким риском
> сначала откройте файл и проверяйте открытый файл либо используйте возможности системы, ограничивающие
> процесс одной папкой. Для большинства программ двух проверок выше достаточно.

## 5. Внедрение HTML и XSS

Когда программа помещает пользовательский текст в HTML-страницу, браузер читает `<script>` в этом тексте как
код. Это **межсайтовый скриптинг (XSS)**. Лекарство — **экранировать** текст для HTML в том месте, где вы
записываете его в страницу. Сохраните `src/bin/html.rs`:

```rust
/// Makes text safe to place inside HTML text or inside a quoted attribute.
fn escape_html(text: &str) -> String {
    let mut out = String::with_capacity(text.len());
    for c in text.chars() {
        match c {
            '&' => out.push_str("&amp;"),
            '<' => out.push_str("&lt;"),
            '>' => out.push_str("&gt;"),
            '"' => out.push_str("&quot;"),
            '\'' => out.push_str("&#39;"),
            other => out.push(other),
        }
    }
    out
}

fn main() {
    let comment = "<script>alert('INJECTED')</script>";
    println!("raw:     <p>{comment}</p>");
    println!("escaped: <p>{}</p>", escape_html(comment));
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn escapes_the_five_special_characters() {
        assert_eq!(
            escape_html("<a href=\"x\">'&'</a>"),
            "&lt;a href=&quot;x&quot;&gt;&#39;&amp;&#39;&lt;/a&gt;"
        );
    }

    #[test]
    fn ampersand_is_escaped_first_and_only_once() {
        assert_eq!(escape_html("&lt;"), "&amp;lt;");
    }
}
```

```text
raw:     <p><script>alert('INJECTED')</script></p>
escaped: <p>&lt;script&gt;alert(&#39;INJECTED&#39;)&lt;/script&gt;</p>
```

**Что происходит:**

1. `escape_html` заменяет пять символов, имеющих особый смысл в HTML, их *символьными ссылками* (`&lt;`
   означает `<`). Браузер показывает их как текст и не выполняет.
2. Сырая строка содержит работающий тег `<script>`; экранированная — только безвредный текст.
3. Тесты (запустите их через `cargo test --bin html`) проверяют, что все пять символов заменяются и что `&`
   обрабатывается **один раз**: `&lt;` во вводе становится `&amp;lt;`, и посетитель видит буквальный текст `&lt;`.

В настоящих программах не пишите это вручную: используйте библиотеку шаблонов, которая экранирует
автоматически, и включайте автоматическое экранирование по умолчанию. Та же идея относится к любому языку
вывода: JSON (JavaScript Object Notation, текстовый формат данных: используйте `serde_json`, никогда не склеивайте строки), CSV (comma-separated values, значения через запятую — таблица в виде текста), URL (Uniform Resource Locator, веб-адрес), строки журнала ([Урок 10.3](03-the-owasp-top-10.md)).

## 6. Поиск внедрения при код-ревью

> 💡 **Совет:** при ревью кода ищите `format!` и `+` рядом со словами `SELECT`, `sh -c`, `join(`, `<`. На каждое
> найденное место нужен ответ на вопрос «откуда берётся этот текст?». Лекарство для каждого: SQL — параметры;
> команды — без оболочки и с `--`; пути — компоненты `Normal` плюс `canonicalize`; HTML — экранирование;
> журналы — `{:?}`.

## Частые ошибки

> ⚠️ **Частая ошибка:** «Я убрал кавычки из ввода, так что SQL безопасен». Удаление символов — это очистка:
> атакующие находят другой путь (другие кодировки, другие специальные символы), а настоящие имена вроде
> `O'Brien` ломаются. Используйте параметры.

> ⚠️ **Частая ошибка:** проверять `name.contains("..")` и считать путь безопасным. Это список запрещённого: он
> пропускает абсолютные пути, ссылки и другие формы. Используйте проверку компонентов.

> ⚠️ **Частая ошибка:** думать, что Rust делает внедрение невозможным. Rust защищает память. Внедрение — это
> логическая ошибка, и она прекрасно компилируется, как показывают сломанные функции этого урока.

## Попробуйте сами

1. Наберите и запустите `sql`, `command`, `option`, `path` и `html`.
2. В `sql.rs` измените текст атаки на `x' OR name = 'root`. Что возвращает сломанная функция?
3. В `command.rs` попробуйте атаку `$(echo 1+1)`. Какая версия печатает `Hello, 1+1`, а какая печатает текст
   `$(echo 1+1)` без изменений? Почему?

## Упражнения

### Упражнение 1 — Назовите атаку (легко)

Что не так в каждой строке? (a) `format!("DELETE FROM notes WHERE id = {id}")`, где `id` — `String` из веб-запроса.
(b) `Command::new("sh").arg("-c").arg(format!("convert {file} out.png"))`. (c) `fs::read(format!("uploads/{name}"))`.
(d) `format!("<h1>Hello, {name}</h1>")`, отправленное в браузер.

<details>
<summary>Подсказка</summary>

Для каждой строки спросите, куда попадает ввод: в базу данных, оболочку, файловую систему или браузер.

</details>

<details>
<summary>Ответ</summary>

(a) Внедрение SQL: используйте параметр (`id = ?1`) и целочисленный тип. (b) Внедрение команд (и аргументов):
запускайте `convert` напрямую через `.arg(file)` (после `--`, если программа это поддерживает) или отклоняйте
необычные имена. (c) Обход пути: `name` может быть `../../x`; принимайте только компоненты `Normal` и проверяйте
настоящий путь. (d) XSS: экранируйте `name` для HTML.

</details>

### Упражнение 2 — Протестируйте проверку пути (средне)

Напишите **тест** для `safe_path` в `path.rs`: он должен отклонять `../secret.txt`, `/etc/hosts` и
`docs/../../secret.txt` и принимать `docs/a.txt`.

<details>
<summary>Подсказка</summary>

Добавьте `#[cfg(test)] mod tests` в конец файла. `safe_path("docs/a.txt")` должно быть равно
`Some(PathBuf::from("sandbox/public/docs/a.txt"))`.

</details>

<details>
<summary>Ответ</summary>

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn rejects_parent_and_absolute_paths() {
        assert!(safe_path("../secret.txt").is_none());
        assert!(safe_path("/etc/hosts").is_none());
        assert!(safe_path("docs/../../secret.txt").is_none());
        assert_eq!(
            safe_path("docs/a.txt"),
            Some(PathBuf::from("sandbox/public/docs/a.txt"))
        );
    }
}
```

Почему это работает: `docs/../../secret.txt` содержит компонент `ParentDir`, а любой компонент, не являющийся
`Normal`, заставляет `safe_path` вернуть `None`. Принятое имя становится базовой папкой плюс его части.

</details>

### Упражнение 3 — Тесты для SQL (сложно)

Напишите два **теста** для `find_users` в `sql.rs`: (1) пользователя с апострофом в имени (`O'Brien`) можно
сохранить и найти; (2) текст атаки не находит никого.

<details>
<summary>Подсказка</summary>

Используйте `setup()`, чтобы получить соединение. Вставляйте через `conn.execute("INSERT INTO users VALUES (?1, 'user')", params!["O'Brien"])`.

</details>

<details>
<summary>Ответ</summary>

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn an_apostrophe_is_just_data() {
        let conn = setup().unwrap();
        conn.execute("INSERT INTO users VALUES (?1, 'user')", params!["O'Brien"])
            .unwrap();
        assert_eq!(find_users(&conn, "O'Brien").unwrap(), vec!["O'Brien:user"]);
    }

    #[test]
    fn the_attack_text_finds_nobody() {
        let conn = setup().unwrap();
        assert!(find_users(&conn, "nobody' OR '1'='1").unwrap().is_empty());
    }
}
```

Почему это работает: параметры делают апостроф обычным символом и при сохранении, и при поиске. То же
свойство, которое останавливает атаку, позволяет работать настоящим именам с апострофами.

</details>

## Вопросы для самопроверки

1. Что такое внедрение, одним предложением?

<details>
<summary>Ответ</summary>

Ввод смешивается с командой, запросом или документом так, что получатель читает часть данных как код.

</details>

2. Почему параметризованный запрос останавливает внедрение SQL?

<details>
<summary>Ответ</summary>

Команда и значения отправляются раздельно, поэтому база данных никогда не читает значение как SQL, какие бы
символы в нём ни были.

</details>

3. Почему `Command::new("echo").arg(text)` безопаснее, чем `sh -c`?

<details>
<summary>Ответ</summary>

Оболочки нет, поэтому `;`, `|`, `$(…)` и другие специальные символы не интерпретируются; текст передаётся как один
аргумент.

</details>

4. Что делает `--` в командной строке и чем это полезно?

<details>
<summary>Ответ</summary>

Он говорит, что все следующие слова — не параметры. Это не даёт пользовательскому вводу, начинающемуся с `-`,
быть прочитанным как параметр.

</details>

## Итоги

- Внедрение — это данные, прочитанные как код. Держите данные и код в разных каналах.
- SQL: всегда параметры для значений; список разрешённого для имён таблиц и столбцов; минимальные права учётной записи базы данных.
- Команды: без оболочки, `Command::new(program).arg(x)`; добавляйте `--` или отклоняйте начальный `-` против внедрения аргументов.
- Пути: принимайте только компоненты `Normal`, затем `canonicalize` и проверка `starts_with` базовой папки.
- HTML и другие языки вывода: экранируйте для этого языка там, где записываете значение, лучше автоматически.
- Rust защищает память, а не логику: все эти ошибки компилируются.

## Источники и что почитать

- [OWASP SQL Injection Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/SQL_Injection_Prevention_Cheat_Sheet.html) — параметризованные запросы, списки разрешённого и наименьшие привилегии.
- [OWASP OS Command Injection Defense Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/OS_Command_Injection_Defense_Cheat_Sheet.html) — как избегать оболочки и обращаться с аргументами.
- [OWASP Path Traversal](https://owasp.org/www-community/attacks/Path_Traversal) — примеры и защита.
- [OWASP Cross Site Scripting Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Cross_Site_Scripting_Prevention_Cheat_Sheet.html) — кодирование вывода для каждого контекста HTML.
- [Документация rusqlite](https://docs.rs/rusqlite/0.43.0/rusqlite/) — крейт SQLite, использованный здесь, включая `params!`.
- [Rust — `std::process::Command`](https://doc.rust-lang.org/std/process/struct.Command.html) — запуск программ с раздельными аргументами.
- [Rust — `std::path::Component`](https://doc.rust-lang.org/std/path/enum.Component.html) — части пути.
- [Rust — `std::fs::canonicalize`](https://doc.rust-lang.org/std/fs/fn.canonicalize.html) — настоящий путь с разрешёнными ссылками.

---

[🏠 Начало](../START-HERE.md) · [← 10.4 Проверка ввода: никогда не доверяйте вводу](04-input-validation.md) · [📋 Этап 10](README.md) · [10.6 Аутентификация и хеширование паролей →](06-authentication-and-password-hashing.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
