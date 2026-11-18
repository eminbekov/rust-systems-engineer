<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 10.4 Input validation: never trust input](04-input-validation.md) · [📋 Phase 10](README.md) · [10.6 Authentication and password hashing →](06-authentication-and-password-hashing.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/10-security-fundamentals/05-injection-attacks.md) <!-- nav -->

# Lesson 10.5 — Injection attacks: SQL, commands and paths

> **Phase 10 — Security fundamentals** · Lesson 5 of 15 · ⏱ about 135 minutes

## What you will learn

- What an injection attack is: data that the receiver mistakes for code.
- SQL injection with a real, harmless demonstration, and the cure: parameterized queries.
- Command injection, argument injection, and how to run programs without a shell.
- Path traversal (`../`), and how to keep file access inside one folder.
- Escaping output for HTML, and the general rule that covers every kind of injection.

## Before you start

- You finished [Lesson 10.4 — Input validation](04-input-validation.md).
- You know `std::fs` and `Path` ([Lesson 04.10](../04-structs-enums-collections-errors/10-error-handling.md) used files) and `Result`.
- Create the package. The order matters: `cargo new code/p10-injection`, then `cd code/p10-injection`, then `cargo add rusqlite@0.43 --features bundled`, then `mkdir src/bin` and `rm src/main.rs`. The feature `bundled` makes Cargo compile the SQLite database engine inside your program, so nothing has to be installed. The first build takes about a minute.
- **Safety:** all attacks in this lesson hit only your own program, in a folder named `sandbox`, and the payload only prints the word `INJECTED`.

## New words

| Word | Meaning |
|---|---|
| **injection** | An attack where input is placed into a command, query or document in a way that changes its structure, so the receiver treats data as code. |
| **SQL** (Structured Query Language) | The language used to talk to most databases: `SELECT name FROM users WHERE id = 5`. |
| **SQL injection** | Injection into a SQL command. The attacker can read, change or delete data. |
| **SQLite** | A small database engine that lives inside your program and stores data in a file or in memory. |
| **parameterized query** (or *prepared statement*) | A SQL command with placeholders (`?1`) for the values. The values are sent separately and are never read as SQL. |
| **shell** | The program (`sh`, `bash`, `zsh`) that reads a line of text, splits it into words and runs programs. It also understands `;`, `|`, `&&`, `$(…)` and more. |
| **command injection** | Injection into a text that a shell will run. |
| **argument injection** | The input is passed as an argument to a program, and it starts with `-`, so the program treats it as an option. |
| **path traversal** | Using `..` (the parent folder) or an absolute path to reach files outside the folder you meant to allow. |
| **symbolic link** (symlink) | A special file that points to another file or folder. |
| **XSS** (Cross-Site Scripting) | Injection of script into a web page, so the visitor's browser runs the attacker's code. |
| **HTML** (HyperText Markup Language) | The language of web pages, with tags such as `<p>` and `<script>`. |

## 1. The one idea behind every injection

Computers receive **commands** (code) and **data** (values). Trouble starts when a program builds one
text from both and then sends the text to an interpreter, which cannot tell where the data ended:

```text
"SELECT ... WHERE name = '"  +  name  +  "'"
 ───────── code ─────────      data      code
```

If `name` contains a quote, the data ends early and the rest is read as code. The same is true for a
shell command, a file path, an HTML page, a log line. The cure is always one of these, in this order of
preference:

1. **Keep data and code in separate channels.** Parameters for SQL; a list of arguments instead of a
   shell text. The interpreter never has to guess.
2. **If you must build a text, allow-list the pieces** (Lesson 10.4) **and escape** for that exact language.
3. **Use the least privilege** (Lesson 10.1), so that a successful injection can do little: a database
   account that cannot drop tables, a process that cannot read the whole disk.

## 2. SQL injection

Create `src/bin/sql.rs`. It builds an in-memory database with three users and searches it twice, once
the broken way and once the right way:

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

Run `cargo run --bin sql`:

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

**What happens:**

1. `Connection::open_in_memory()` creates a throw-away SQLite database in memory.
   `execute_batch` runs several SQL commands in one call and builds the table `users`.
2. `find_users_bad` uses `format!` to **paste** the name into the SQL text. For `ana` this is fine.
3. The attack text is `nobody' OR '1'='1`. After pasting, the command is
   `... WHERE name = 'nobody' OR '1'='1'`. The first quote closes the string, and `OR '1'='1'` is a
   condition that is always true, so **every row** comes back, including the administrator.
4. `find_users` writes the placeholder `?1` in the SQL and gives the value with `params![name]`. The
   database receives the command and the value **separately**. The whole attack text, quotes included,
   is only a name, and no user has that name: the result is empty.

The same hole can do much more in other places: log in without a password (`admin'--` turns the rest of the
command into a comment), read other tables with `UNION`, or change data. Some database interfaces even run
several commands in one string. A single quote in an input should never be able to change what the program does.

**What a placeholder cannot do.** A parameter replaces a **value**, never a table name, a column name or
`ASC`/`DESC`. For those, choose from a fixed list. A `match` is an allow-list; an unknown word becomes
`None`, and the caller must refuse the request:

```rust
fn order_clause(requested: &str) -> Option<&'static str> {
    match requested {
        "name" => Some("ORDER BY name"),
        "role" => Some("ORDER BY role"),
        _ => None,
    }
}
```

The program inserts only the fixed texts from its own code, never the user's text.

> 🔒 **Security:** *always* use parameters for values, in every database library, even when "the input is
> only a number". Also give the application a database account with only the rights it needs
> (read and write its own tables, not `DROP`).

## 3. Command injection

Programs often run other programs. In Rust you do this with `std::process::Command`. The dangerous way
is to give a *shell* one big text. Save `src/bin/command.rs`:

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

**What happens:**

1. `greet_bad` builds the text `echo Hello, ana; echo INJECTED` and runs it with `sh -c`, which means
   "interpret this text as a shell command". The shell reads `;` as the end of the first command, so it runs
   a **second command** that the programmer never wrote. Instead of `echo INJECTED`, an attacker could
   put any command.
2. `greet` starts `echo` directly: `Command::new("echo").arg(...)`. There is **no shell**, so `;` is
   an ordinary character. The whole text is one argument, and `echo` prints it. Nothing runs.

The rule: **never build a shell text from input.** Start the program you want directly, and give each
argument with `.arg()` or `.args()`. If you think you need a shell, you almost always need a better
design.

### Argument injection

Even without a shell, the *program* may treat an argument that starts with `-` as an **option**. Save
`src/bin/option.rs`. It makes a small folder `sandbox/docs/a.txt` and lists a "file name" typed by a user:

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

**What happens:**

1. The expected input `.` lists the folder: `docs`.
2. The attack input `-R` is not a file name. It is the option "recursive", so `ls` also lists everything
   inside `docs`. The user has changed what the command does. With other programs, options can write
   files or run other programs.
3. A single `--` before the input means "all that follows is a name, not an option" (a convention that
   most command-line tools follow). Now `ls` correctly says that there is no file called `-R`. (The
   exact error wording differs between operating systems.)
4. The program removes `sandbox` at the end.

Other defenses: reject input that starts with `-` (Lesson 10.4: an allow-list for file names), and
prefer a Rust library over running a program when one exists.

## 4. Path traversal

A program offers files from a folder, for example `sandbox/public`. A visitor asks for the name
`../secret.txt`. If the program joins the base folder and the name, the path becomes
`sandbox/public/../secret.txt`, which means `sandbox/secret.txt`, outside the folder. Save
`src/bin/path.rs`:

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

**What happens:**

1. `read_bad` joins the base folder and the name: the attack `../secret.txt` returns `THE SECRET`.
2. `safe_path` looks at the name **component by component**. `Path::components()` splits a path into
   parts: `Normal` for a plain name, `ParentDir` for `..`, `RootDir` for a leading `/`, and so on. We accept only
   `Normal` parts. A name with `..`, a leading `/` or a drive prefix gives `None`.
3. `read_safe` adds a second layer. `fs::canonicalize` asks the file system for the **real** path: it
   removes `..` and **follows symbolic links**. Then `starts_with` checks that the real path is still
   inside the real base folder. (`canonicalize` fails when the file does not exist, which is fine here:
   no file, no answer.)
4. The symbolic link `link.txt` lives inside `public` and points to `../secret.txt`. The names pass step 1
   (a plain name!), so reading with only that check returns the secret. Step 2 sees that the real path
   leaves the folder and refuses.
5. `#[cfg(unix)]` compiles that part only on macOS and Linux, where `symlink` exists.

> 🔒 **Security:** a check followed by a use ("check the path, then open it") has a small gap in
> between, in which a file can be replaced by a link (a *race*). For high-risk programs open the file first
> and check the open file, or use system features that confine a process to one folder. For most programs the two
> checks above are the right level.

## 5. HTML injection and XSS

When a program puts user text into an HTML page, the browser reads `<script>` in that text as code. This is
**cross-site scripting (XSS)**. The cure is to **escape** the text for HTML at the place where you write it
into the page. Save `src/bin/html.rs`:

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

**What happens:**

1. `escape_html` replaces the five characters that have a special meaning in HTML with their
   *character references* (`&lt;` means `<`). The browser shows them as text and does not run them.
2. The raw line contains a working `<script>` tag; the escaped line contains only harmless text.
3. The tests (run them with `cargo test --bin html`) check that all five characters are replaced, and that `&`
   is handled **once**: `&lt;` in the input becomes `&amp;lt;`, shown to the visitor as the literal text `&lt;`.

In real programs do not write this by hand: use a template library that escapes automatically, and
turn the automatic escaping on by default. The same idea applies to every output language: JSON
(JavaScript Object Notation, a text format for data: use `serde_json`, never paste strings), CSV (comma-separated
values, a table as text), URLs (Uniform Resource Locators, web addresses), log lines ([Lesson 10.3](03-the-owasp-top-10.md)).

## 6. Finding injection in a code review

> 💡 **Tip:** when you review code, search for `format!` and `+` near the words `SELECT`, `sh -c`,
> `join(`, `<`. Every hit needs an answer to "where does this text come from?". The cure for each: SQL,
> parameters; commands, no shell and `--`; paths, `Normal` components plus `canonicalize`; HTML, escaping;
> logs, `{:?}`.

## Common mistakes

> ⚠️ **Common mistake:** "I removed the quotes from the input, so SQL is safe." Removing characters is
> sanitizing: attackers find another way (other encodings, other special characters), and real names such
> as `O'Brien` break. Use parameters.

> ⚠️ **Common mistake:** checking `name.contains("..")` and calling the path safe. A deny-list: it misses
> absolute paths, links and other forms. Use the component check.

> ⚠️ **Common mistake:** thinking that Rust makes injection impossible. Rust protects memory. An
> injection is a logic bug and compiles fine, as the broken functions in this lesson show.

## Try it yourself

1. Type and run `sql`, `command`, `option`, `path` and `html`.
2. In `sql.rs`, change the attack text to `x' OR name = 'root`. What does the broken function return?
3. In `command.rs`, try the attack `$(echo 1+1)`. Which version prints `Hello, 1+1` and which prints the
   text `$(echo 1+1)` unchanged? Why?

## Exercises

### Exercise 1 — Name the attack (easy)

What is wrong in each line? (a) `format!("DELETE FROM notes WHERE id = {id}")` where `id` is a `String`
from a web request. (b) `Command::new("sh").arg("-c").arg(format!("convert {file} out.png"))`.
(c) `fs::read(format!("uploads/{name}"))`. (d) `format!("<h1>Hello, {name}</h1>")` sent to a browser.

<details>
<summary>Hint</summary>

For each line ask where the input goes: a database, a shell, the file system or a browser.

</details>

<details>
<summary>Answer</summary>

(a) SQL injection: use a parameter (`id = ?1`) and an integer type. (b) Command injection (and argument
injection): run `convert` directly with `.arg(file)` (after `--` if the program supports it), or reject unusual names. (c) Path
traversal: `name` can be `../../x`; accept only `Normal` components and check the real path.
(d) XSS: escape `name` for HTML.

</details>

### Exercise 2 — Test the path check (medium)

Write a **test** for `safe_path` in `path.rs`: it must refuse `../secret.txt`, `/etc/hosts` and
`docs/../../secret.txt`, and accept `docs/a.txt`.

<details>
<summary>Hint</summary>

Add a `#[cfg(test)] mod tests` at the end of the file. `safe_path("docs/a.txt")` should equal
`Some(PathBuf::from("sandbox/public/docs/a.txt"))`.

</details>

<details>
<summary>Answer</summary>

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

Why it works: `docs/../../secret.txt` contains a `ParentDir` component, and any component that is not
`Normal` makes `safe_path` return `None`. The accepted name becomes the base folder plus its parts.

</details>

### Exercise 3 — SQL tests (hard)

Write two **tests** for `find_users` in `sql.rs`: (1) a user whose name contains an apostrophe
(`O'Brien`) can be stored and found; (2) the attack text finds nobody.

<details>
<summary>Hint</summary>

Use `setup()` to get a connection. Insert with `conn.execute("INSERT INTO users VALUES (?1, 'user')", params!["O'Brien"])`.

</details>

<details>
<summary>Answer</summary>

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

Why it works: parameters make the apostrophe an ordinary character both when storing and when searching.
The same property that stops the attack also lets real names with apostrophes work.

</details>

## Quiz

1. In one sentence, what is injection?

<details>
<summary>Answer</summary>

Input is mixed into a command, query or document so that the receiver reads part of the data as code.

</details>

2. Why does a parameterized query stop SQL injection?

<details>
<summary>Answer</summary>

The command and the values are sent separately, so the database never reads the value as SQL, whatever characters
it has.

</details>

3. Why is `Command::new("echo").arg(text)` safer than `sh -c`?

<details>
<summary>Answer</summary>

There is no shell, so `;`, `|`, `$(…)` and other special characters are not interpreted; the text is passed as one
argument.

</details>

4. What does `--` do in a command line, and why is it useful?

<details>
<summary>Answer</summary>

It says that all later words are not options. It stops a user's input that begins with `-` from being read as
an option.

</details>

## Summary

- Injection is data read as code. Keep data and code in separate channels.
- SQL: always use parameters for values; allow-list table and column names; give the database account minimal rights.
- Commands: no shell, use `Command::new(program).arg(x)`; add `--` or reject a leading `-` for argument injection.
- Paths: accept only `Normal` components, then `canonicalize` and check `starts_with` the base folder.
- HTML and other output languages: escape for that language where you write the value, ideally automatically.
- Rust protects memory, not logic: all of these bugs compile.

## Sources and further reading

- [OWASP SQL Injection Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/SQL_Injection_Prevention_Cheat_Sheet.html) — parameterized queries, allow-lists and least privilege.
- [OWASP OS Command Injection Defense Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/OS_Command_Injection_Defense_Cheat_Sheet.html) — avoiding the shell and handling arguments.
- [OWASP Path Traversal](https://owasp.org/www-community/attacks/Path_Traversal) — examples and defenses.
- [OWASP Cross Site Scripting Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Cross_Site_Scripting_Prevention_Cheat_Sheet.html) — output encoding for each HTML context.
- [rusqlite documentation](https://docs.rs/rusqlite/0.43.0/rusqlite/) — the SQLite crate used here, including `params!`.
- [Rust — `std::process::Command`](https://doc.rust-lang.org/std/process/struct.Command.html) — starting programs with separate arguments.
- [Rust — `std::path::Component`](https://doc.rust-lang.org/std/path/enum.Component.html) — the parts of a path.
- [Rust — `std::fs::canonicalize`](https://doc.rust-lang.org/std/fs/fn.canonicalize.html) — the real path, with links resolved.

---

[🏠 Start](../START-HERE.md) · [← 10.4 Input validation: never trust input](04-input-validation.md) · [📋 Phase 10](README.md) · [10.6 Authentication and password hashing →](06-authentication-and-password-hashing.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
