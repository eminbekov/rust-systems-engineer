<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 00.5 Installing Rust and RustRover](05-install-rust-and-rustrover.md) · [📋 Phase 00](README.md) · [00.7 Reading errors and getting help →](07-reading-errors-and-getting-help.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/00-getting-started/06-your-first-rust-program.md) <!-- nav -->

# Lesson 00.6 — Your first Rust program

> **Phase 00 — Getting started** · Lesson 6 of 8 · ⏱ about 3 hours (2 sittings)

## What you will learn

- How to create a Rust project with Cargo and what every file in it is for.
- How to build and run a program, and the difference between `cargo run`, `cargo build` and `cargo check`.
- What each part of `fn main() { println!("Hello, world!"); }` means.
- How to change the program, format it with `cargo fmt` and check it with `cargo clippy`.
- How to compile one file directly with `rustc`, and how to run a project in RustRover.
- Why Cargo already created a Git repository for you.

## Before you start

- You finished [Lesson 00.5 — Installing Rust and RustRover](05-install-rust-and-rustrover.md). `rustc --version` and `cargo --version` work in a **new** terminal tab.
- The folder `~/projects/rust/code` exists (you created it in [Lesson 00.4](04-terminal-basics.md)). If it does not, run `mkdir -p ~/projects/rust/code`.
- Open the terminal and go to the course folder: `cd ~/projects/rust`.

## New words

| Word | Meaning |
|---|---|
| **project** (package) | A folder with a `Cargo.toml` file and source code that Cargo can build. Cargo calls it a *package*. |
| **crate** | The unit that the compiler builds: either a program (a *binary crate*) or a library. Phase 04 explains it fully. |
| **`Cargo.toml`** | The settings file of a project: its name, version, edition and libraries. |
| **TOML** (Tom's Obvious, Minimal Language) | A simple text format for settings: `name = "value"` lines grouped under `[headers]`. |
| **`Cargo.lock`** | A file that Cargo writes by itself. It records the exact versions of everything used in the last build. |
| **function** | A named block of code. The function called `main` is where every Rust program starts. |
| **macro** | A special kind of helper that expands into more code. Its name ends with `!`, like `println!`. Phase 05 teaches how they work. |
| **string** | A piece of text in quotes: `"Hello"`. |
| **semicolon** `;` | The mark that ends a statement (one step) in Rust. |
| **comment** | Text for humans that the compiler ignores. In Rust it starts with `//`. |
| **build** | Compile the code into an executable file. |
| **binary** | An executable file. Cargo also uses this word for a program crate. |
| **profile** | A set of compiler settings. **debug** (the default) builds fast and is easy to debug; **release** builds slowly but makes the program faster. |
| **`target`** | The folder where Cargo puts everything it builds. |
| **Git repository** | A folder that Git tracks. It contains a hidden `.git` folder with the history. Phase 01 teaches it. |
| **`.gitignore`** | A file that lists what Git should not track, such as build output. |
| **IDE** (Integrated Development Environment) | A program for writing code: an editor with run buttons, error marks and other helpers. We use **RustRover**. |

## 1. Create the project

In the terminal, from `~/projects/rust`, run:

```text
% cargo new code/p00-hello
    Creating binary (application) `p00-hello` package
note: see more `Cargo.toml` keys and their definitions at https://doc.rust-lang.org/cargo/reference/manifest.html
```

What happens:

1. `cargo new` creates a new project (a **package**).
2. `code/p00-hello` is the path of the new folder. The *last* part of the path, `p00-hello`, becomes the name of the project.
3. `binary (application)` means: a program that you can run (not a library for other programs).
4. The `note:` line is only a link to the documentation.

> 📏 **Convention:** Project names in this course look like `p00-hello`: `p`, the phase number, a dash and a
> short name in lowercase with dashes between words. Cargo itself requires only that the name starts with a
> letter and has letters, digits, `-` or `_`.

Go into the project and look at it. The `-a` option also shows hidden files:

```text
% cd code/p00-hello
% ls -a
.
..
.git
.gitignore
Cargo.toml
src
```

The folder tree:

```text
p00-hello/
├── .git/            ← hidden: the Git history (empty for now)
├── .gitignore       ← a list of files Git should ignore
├── Cargo.toml       ← settings of the project
└── src/
    └── main.rs      ← YOUR source code
```

## 2. The files Cargo made

### `Cargo.toml`

```text
% cat Cargo.toml
[package]
name = "p00-hello"
version = "0.1.0"
edition = "2024"

[dependencies]
```

What happens:

1. `[package]` — a header: the next lines describe the package.
2. `name = "p00-hello"` — the name of the project (and of the program it produces).
3. `version = "0.1.0"` — the version of *your* program, in the form major.minor.patch. `0.1.0` means "just started".
4. `edition = "2024"` — the language edition from Lesson 00.5.
5. `[dependencies]` — here you will list the libraries (called *crates*) that your program uses. It is empty: we use only what comes with Rust.

### `src/main.rs`

```text
% cat src/main.rs
fn main() {
    println!("Hello, world!");
}
```

This is a complete program. `src` means "source". Rust source files end with `.rs`. `main.rs` is the
starting file of a program.

## 3. Run it

```text
% cargo run
   Compiling p00-hello v0.1.0 (/Users/student/projects/rust/code/p00-hello)
    Finished `dev` profile [unoptimized + debuginfo] target(s) in 0.16s
     Running `target/debug/p00-hello`
Hello, world!
```

What happens:

1. `cargo run` first builds the project, then starts the program. These are the two steps from Lesson 00.3: compile, then run.
2. `Compiling ...` — Cargo called `rustc` to turn `src/main.rs` into machine code. The time (`0.16s`) will be different on your machine.
3. `Finished `dev` profile [unoptimized + debuginfo]` — the build is ready. "dev" is the *debug* profile: no optimisation, with extra information for debugging tools.
4. `Running `target/debug/p00-hello`` — Cargo starts the executable file that it made. Its path is shown.
5. `Hello, world!` — the output of **your** program.

Run it again:

```text
% cargo run
    Finished `dev` profile [unoptimized + debuginfo] target(s) in 0.00s
     Running `target/debug/p00-hello`
Hello, world!
```

Nothing changed in the code, so Cargo did not compile again. It only ran the program.

Now look at what appeared:

```text
% ls
Cargo.lock
Cargo.toml
src
target
```

- `target/` is new. All build results are here. You can delete it at any time (`cargo clean`); the next build recreates it.
- `Cargo.lock` is new. Cargo writes it; never edit it by hand.

The program itself is `target/debug/p00-hello`. It is a normal executable file (Lesson 00.3), so you can
start it directly, without Cargo:

```text
% ./target/debug/p00-hello
Hello, world!
```

## 4. What each part of the program means

```rust
fn main() {
    println!("Hello, world!");
}
```

What happens, piece by piece:

1. `fn` — a keyword (a reserved word of the language) that says: "a function starts here". `fn` is short for *function*.
2. `main` — the name of the function. Rust starts every program by running the function named `main`.
3. `()` — the list of inputs of the function. Empty parentheses mean that `main` takes no input.
4. `{` and `}` — the curly braces enclose the body of the function: the steps it performs.
5. `println!` — a **macro** that prints text and then ends the line (`ln` = line). The `!` tells you that it is a macro, not a normal function.
6. `("Hello, world!")` — the argument: a **string**. Text in Rust is written in **double** quotes.
7. `;` — the semicolon ends the step. Almost every line in a function ends with one.

Two style rules to adopt from day one:

> 📏 **Convention:** Indent with **4 spaces** (not the Tab key). You do not need to count them: RustRover and
> `cargo fmt` do it for you.

> 📏 **Convention:** Function and variable names are written in `snake_case`: lowercase words
> separated by underscores (`print_report`). You will see more naming rules in Phase 02.

## 5. Change the program

Open `src/main.rs` in RustRover (or any editor) and replace the content with this:

```rust
fn main() {
    // This program prints several lines.
    println!("Hello, Rust!");
    println!("I am learning to program.");
    print!("This line has no newline, ");
    println!("so this text continues it.");
    println!("Two plus three is {}.", 2 + 3);
    let name = "Ada";
    println!("My name is {}.", name);
}
```

Save the file (`Cmd + S`) and run `cargo run`:

```text
% cargo run
   Compiling p00-hello v0.1.0 (/Users/student/projects/rust/code/p00-hello)
    Finished `dev` profile [unoptimized + debuginfo] target(s) in 0.13s
     Running `target/debug/p00-hello`
Hello, Rust!
I am learning to program.
This line has no newline, so this text continues it.
Two plus three is 5.
My name is Ada.
```

What happens:

1. `// This program prints several lines.` — a **comment**. The compiler ignores everything from `//` to the end of the line. Use comments to explain *why*, not to repeat *what* the code does.
2. The next two `println!` lines print their text and move to a new line.
3. `print!` (no `ln`) prints the text and does **not** move to a new line. That is why the next text continues on the same line.
4. In `println!("Two plus three is {}.", 2 + 3)`, the `{}` is a hole. Rust puts the value after the comma into it. The value is calculated first: `2 + 3` is `5`.
5. `let name = "Ada";` — creates a named box `name` that holds the text `"Ada"`. This is a **variable**, from Lesson 00.2; Lesson 02.1 teaches it properly. `{}` in the last line is filled with its value.
6. The output has no `{}` at all: the holes were replaced by values.

Printing and formatting in full detail: Lesson 02.6.

## 6. The Cargo commands you need every day

| Command | What it does | When to use it |
|---|---|---|
| `cargo run` | build, then run | most often |
| `cargo build` | build only (debug profile) | to see if it compiles and produce the file |
| `cargo check` | check the code **without** producing an executable | fastest feedback while you write |
| `cargo build --release` | build with optimisations (release profile) | to measure speed or to give the program to someone |
| `cargo fmt` | format the code | before every commit |
| `cargo clippy` | advice about mistakes and style | before every commit |
| `cargo clean` | delete the `target` folder | to free disk space or to start clean |

Look at `check` and `--release`:

```text
% cargo check
    Checking p00-hello v0.1.0 (/Users/student/projects/rust/code/p00-hello)
    Finished `dev` profile [unoptimized + debuginfo] target(s) in 0.04s
% cargo build --release
   Compiling p00-hello v0.1.0 (/Users/student/projects/rust/code/p00-hello)
    Finished `release` profile [optimized] target(s) in 0.08s
% ls -lh target/debug/p00-hello target/release/p00-hello
-rwxr-xr-x  1 student  wheel   465K Nov 17 16:25 target/debug/p00-hello
-rwxr-xr-x  1 student  wheel   426K Nov 17 16:25 target/release/p00-hello
```

What happens:

1. `cargo check` says "Checking" instead of "Compiling": it analysed the code and reported errors, but made no executable. It is much faster on big projects.
2. `cargo build --release` made a second executable in `target/release/`, with optimisations. For a one-line program the difference is tiny; for real programs a release build can be many times faster.
3. The two files have different sizes, because the debug build contains extra information. (Your sizes will differ.)

> ⚡ **Performance:** Always measure speed with `--release`. A debug build is meant for development; it can
> be 10–100 times slower than the release build for heavy computations. Never judge the speed of
> Rust from a debug build.

### Format and lint

```text
% cargo fmt --check
```

If this prints nothing, your code is already formatted. Here is what happens when it is not. Imagine
a messy file:

```rust
fn main(){
println!("Hello, Rust!"   );
        println!("Second line");
}
```

`cargo fmt --check` shows the differences and fails (exit code 1):

```text
Diff in /Users/student/projects/rust/code/p00-hello/src/main.rs:1:
-fn main(){
-println!("Hello, Rust!"   );
-        println!("Second line");
+fn main() {
+    println!("Hello, Rust!");
+    println!("Second line");
 }
```

Lines starting with `-` would be removed, lines with `+` would be added. Run `cargo fmt` (without
`--check`) and it rewrites the file for you:

```text
% cargo fmt
% cat src/main.rs
fn main() {
    println!("Hello, Rust!");
    println!("Second line");
}
```

`cargo clippy` runs the linter. For correct, clean code it prints only the `Checking` and `Finished` lines. Clippy
speaks up when your code works but could be better. For example, suppose the last line of our program were
`println!("My name is {} and I like {}.", "Ada", "Rust")`. It works, but the texts are already known, so they can
be written directly in the first string. Clippy says:

```text
warning: literal with an empty format string
 --> src/main.rs:8:46
  |
8 |     println!("My name is {} and I like {}.", "Ada", "Rust");
  |                                              ^^^^^^^^^^^^^
  = note: `#[warn(clippy::print_literal)]` on by default
help: try
8 -     println!("My name is {} and I like {}.", "Ada", "Rust");
8 +     println!("My name is Ada and I like Rust.");
```

> 💡 **Tip:** Make `cargo fmt` and `cargo clippy` a habit. Every project in this course must pass both.
> RustRover does it for you if you enabled "run rustfmt on save" and Clippy in Lesson 00.5.

## 7. Compile a single file with `rustc`

Cargo calls `rustc` for you. You can call it yourself for a single file. Make a throw-away folder:

```text
% mkdir -p /tmp/rust-try && cd /tmp/rust-try
% cat > hello.rs <<'XX'
fn main() {
    println!("Hello, Rust");
}
XX
% rustc hello.rs
% ./hello
Hello, Rust
```

What happens:

1. `rustc hello.rs` compiled the file and wrote an executable named `hello` next to it. There is no `target` folder and no `Cargo.toml`.
2. `./hello` ran it (Lesson 00.4: `./` is "in this folder").

For anything bigger than one file, always use Cargo: it manages the settings, the folders and (later) the libraries.
By default `rustc` uses an older edition than Cargo does; another reason to use Cargo.

Remove the throw-away folder: `cd ~ && rm -r /tmp/rust-try`.

## 8. Run it in RustRover

1. In RustRover choose **File → Open**, select the folder `~/projects/rust/code/p00-hello` (the one that contains `Cargo.toml`) and click **Open**. Choose **This Window** or **New Window**.
2. If RustRover asks whether to trust the project, choose **Trust Project**. If it offers to "Attach Cargo project" or shows a notification about Cargo, accept: that is how the IDE learns about `Cargo.toml`.
3. Open `src/main.rs`. Next to `fn main` you will see a green triangle in the margin. Click it and choose **Run 'Run p00-hello'** (or press `Ctrl + R`).
4. A **Run** panel opens at the bottom and shows the same lines that the terminal showed.
5. Try `Cmd + Option + L` (**Code → Reformat Code**) to format the file, and look at the red and yellow marks in the editor: they are the compiler's and Clippy's messages shown in place.

> 💡 **Tip:** The IDE and the terminal do exactly the same work: both call Cargo. Knowing the commands makes you
> independent of any IDE, and servers have no IDE at all.

## 9. A first look at Git

Remember `.git` and `.gitignore` from section 1? Cargo created a **Git repository** in the project, because
version control is standard practice. Look at the `.gitignore` file:

```text
% cat .gitignore
/target
```

It tells Git to ignore the `target` folder, because build results are big, can be recreated at any time and do
not belong in the history. Ask Git what it sees (this is just a look; Phase 01 explains everything):

```text
% git status --short
?? .gitignore
?? Cargo.lock
?? Cargo.toml
?? src/
```

Each `??` means "new file that Git does not track yet". `target/` is not in the list because of `.gitignore`.
Nothing is saved in the history yet; the first **commit** is a topic for [Lesson 01.3](../01-git-and-version-control/03-your-first-repository-and-commits.md).

> ⚠️ **Common mistake:** If a project is created *inside* a folder that is already a Git repository (for example
> once you turn `~/projects/rust` into your learning repository in Phase 01), `cargo new` does **not** create another
> `.git` and `.gitignore`. Then you must make sure that `target` is ignored by the main repository.
> Phase 01 shows how.

## Common mistakes

1. **Forgetting `!` in `println!`.**

```rust
fn main() {
    println("Hello, world!");
}
```

```text
error[E0423]: cannot find function `println` in this scope
 --> src/main.rs:2:5
  |
2 |     println("Hello, world!");
  |     ^^^^^^^ not found in this scope
  |
  = note: a macro named `println` exists in another namespace
help: use `!` to invoke the macro
  |
2 |     println!("Hello, world!");
  |            +
```

   *Meaning:* there is no *function* called `println`, but there is a *macro*. The compiler suggests the fix with a `+` under the missing `!`.

2. **Forgetting a semicolon.**

```rust
fn main() {
    println!("first")
    println!("second");
}
```

```text
error: expected `;`, found `println`
 --> src/main.rs:2:22
  |
2 |     println!("first")
  |                      ^ help: add `;` here
3 |     println!("second");
  |     ------- unexpected token
```

   *Meaning:* line 2 ended without `;`. The `^` points exactly to the place where it is missing.

3. **An unclosed string.** Here the closing quote is missing:

```rust
fn main() {
    println!("Hello, world!);
}
```

```text
error[E0765]: unterminated double quote string
 --> src/main.rs:2:14
  |
2 |       println!("Hello, world!);
  |  ______________^
3 | | }
  | |__^
```

   *Meaning:* a string that starts with `"` must end with `"`. The compiler marks everything from the opening quote to the end of the file.

4. **Single quotes for text.** `println!('Hello, world!');` gives `error[E0762]: unterminated character literal`.
   In Rust single quotes are for one character; text uses double quotes.

5. **No `main` function.** If the function has another name:

```text
error[E0601]: `main` function not found in crate `p00_hello`
 --> src/main.rs:3:2
  |
3 | }
  |  ^ consider adding a `main` function to `src/main.rs`
```

   *Meaning:* the program has no starting point. Note that in the crate name `p00_hello` the dash became an underscore.

6. **An unclosed brace.** If the final `}` is missing: `error: this file contains an unclosed delimiter`. A *delimiter* is a bracket, a brace or a quote. The message points to the `{` that has no partner.

7. **Running Cargo outside a project.**

```text
% cargo run
error: could not find `Cargo.toml` in `/Users/student/projects/rust/code/empty` or any parent directory
```

   *Meaning:* you are in a folder without a project. Use `pwd`, then `cd` into the project (the folder that contains `Cargo.toml`).

8. **Creating a project in a folder that exists.** `cargo new code/p00-hello` twice gives `error: destination ... already exists` and a hint to use `cargo init`. Choose a new name or delete the old folder.

9. **Not saving the file.** You edit in RustRover but see the old output. Save (`Cmd + S`) before `cargo run` in the terminal. (`cargo run` inside RustRover saves automatically.)

All these messages are explained in detail in [Lesson 00.7](07-reading-errors-and-getting-help.md).

## Try it yourself

Do it from zero, without looking at the lesson, as a test of your memory. Use a new project name `p00-trial`.

1. `cd ~/projects/rust`
2. `cargo new code/p00-trial`
3. `cd code/p00-trial`
4. Replace `src/main.rs` with a program that prints your name on one line and the line `I am learning Rust` on the next.
5. `cargo run`. Check the output.
6. Make a deliberate mistake (remove one `;`), run `cargo run` again, and read the error. Then fix it.
7. `cargo fmt` and `cargo clippy`. Both should print nothing wrong.
8. Open the project in RustRover and run it with the green triangle.

## Exercises

### Exercise 1 — Three lines (easy)

In `p00-hello`, write a program that prints exactly these three lines. Use your own details.

```text
Name: Ada
City: London
Favourite language: Rust
```

<details>
<summary>Hint</summary>

Three `println!` lines. Each line of text goes in double quotes, and each `println!` ends with `;`.

</details>

<details>
<summary>Answer</summary>

```rust
fn main() {
    println!("Name: Ada");
    println!("City: London");
    println!("Favourite language: Rust");
}
```

Output:

```text
Name: Ada
City: London
Favourite language: Rust
```

Why it works: each `println!` prints one line and ends it with a line break.

</details>

### Exercise 2 — A banner (easy)

Create a **new** project `code/p00-banner` with `cargo new`. Make it print this box:

```text
+----------+
| Rust 1.0 |
+----------+
```

<details>
<summary>Hint</summary>

Run `cargo new code/p00-banner` from `~/projects/rust`, edit `code/p00-banner/src/main.rs`, and run `cargo run` inside the new project folder. The box has three lines, so you need three `println!`.

</details>

<details>
<summary>Answer</summary>

```rust
fn main() {
    println!("+----------+");
    println!("| Rust 1.0 |");
    println!("+----------+");
}
```

Output:

```text
+----------+
| Rust 1.0 |
+----------+
```

Why it works: text inside the quotes is printed exactly as it is, including the symbols `+`, `-` and `|`. After the program is ready, check `cargo fmt --check` and `cargo clippy`: both pass without messages.

</details>

### Exercise 3 — Fix three programs (medium)

Each program below does not compile. Say what is wrong without running it, then fix it and check with
`cargo run` in a scratch project.

```rust
// Program A
fn main() {
    println("Hello, world!");
}
```

```rust
// Program B
fn main() {
    println!("first")
    println!("second");
}
```

```rust
// Program C
fn start() {
    println!("Hello, world!");
}
```

<details>
<summary>Hint</summary>

A: look at the end of the name `println`. B: look at the end of line 2. C: what name does Rust look for as the starting point?

</details>

<details>
<summary>Answer</summary>

- **A** — `println` must be written `println!` (it is a macro).
- **B** — the first `println!` needs a semicolon at the end.
- **C** — the function must be named `main`, not `start`.

Fixed versions:

```rust
fn main() {
    println!("Hello, world!");
}
```

```rust
fn main() {
    println!("first");
    println!("second");
}
```

```rust
fn main() {
    println!("Hello, world!");
}
```

Outputs: A and C print `Hello, world!`; B prints `first` and `second` on two lines.

Why it works: all three errors are about the exact spelling that the compiler demands: the `!` of a macro, the `;` that ends a step, and the name `main` that marks the start.

</details>

### Exercise 4 — Debug against release (medium)

In `p00-hello`, build both profiles and compare the sizes of the two executables with `ls -lh`. Which is
bigger, and why? Where does each file live?

<details>
<summary>Hint</summary>

`cargo build` and `cargo build --release`; the files are in two sub-folders of `target`.

</details>

<details>
<summary>Answer</summary>

```text
% cargo build
% cargo build --release
% ls -lh target/debug/p00-hello target/release/p00-hello
-rwxr-xr-x  1 student  wheel   465K Nov 17 16:25 target/debug/p00-hello
-rwxr-xr-x  1 student  wheel   426K Nov 17 16:25 target/release/p00-hello
```

The debug file (`target/debug/`) is bigger because it contains extra information for debugging tools and unoptimised code. The release file (`target/release/`) is smaller and, for real programs, much faster. Your numbers will differ a little.

Why it works: the two profiles pass different options to `rustc`, so they produce different files, and Cargo keeps them in separate folders so that they do not overwrite each other.

</details>

### Exercise 5 — A times table, tidy and clean (hard)

In a new project `code/p00-table`, print the lines of the 7 times table from `7 x 2` to `7 x 6`.
Let Rust do the arithmetic (do not type the results yourself). The code must pass `cargo fmt --check` and
`cargo clippy` with no messages.

Expected output:

```text
7 x 2 = 14
7 x 3 = 21
7 x 4 = 28
7 x 5 = 35
7 x 6 = 42
```

<details>
<summary>Hint</summary>

Use `{}` in the text and `7 * 2` after the comma. Why from 2 and not from 1? Try `7 * 1` and read what Clippy says.

</details>

<details>
<summary>Answer</summary>

```rust
fn main() {
    println!("7 x 2 = {}", 7 * 2);
    println!("7 x 3 = {}", 7 * 3);
    println!("7 x 4 = {}", 7 * 4);
    println!("7 x 5 = {}", 7 * 5);
    println!("7 x 6 = {}", 7 * 6);
}
```

Why it works: the multiplication is calculated first and then put into `{}`. If you write `7 * 1`, Clippy answers "this operation has no effect", because multiplying by one changes nothing. (In Phase 02 you will do this with a loop, so you do not need to repeat the line.)

</details>

## Quiz

1. What does `cargo new code/p00-hello` create, and what is the name of the project?

<details>
<summary>Answer</summary>

A project folder `code/p00-hello` with `Cargo.toml`, `src/main.rs`, and a Git repository (`.git`, `.gitignore`). The name is the last part of the path: `p00-hello`.

</details>

2. What is the difference between `cargo run`, `cargo build` and `cargo check`?

<details>
<summary>Answer</summary>

`cargo build` compiles the program. `cargo run` builds it and then starts it. `cargo check` only analyses the code for errors and makes no executable, so it is the fastest.

</details>

3. What does the `!` in `println!` mean?

<details>
<summary>Answer</summary>

That `println` is a macro, not a normal function.

</details>

4. Where does Cargo put the executable of a debug build, and what is `target`?

<details>
<summary>Answer</summary>

In `target/debug/<project name>`. `target` is the folder for all build results; it can be deleted and recreated, and it is ignored by Git.

</details>

5. Why do we measure speed only with `--release`?

<details>
<summary>Answer</summary>

The debug profile does not optimise the code, so it can be many times slower than the release profile.

</details>

6. What do `cargo fmt` and `cargo clippy` do?

<details>
<summary>Answer</summary>

`cargo fmt` rewrites the code in the standard style. `cargo clippy` shows warnings with advice about mistakes and style.

</details>

7. Why is `target/` listed in `.gitignore`?

<details>
<summary>Answer</summary>

Build results are large, can be recreated at any time and do not belong in the history of the source code.

</details>

## Summary

- `cargo new code/pNN-name` creates a project; `Cargo.toml` holds its settings and `src/main.rs` is the starting file.
- Every Rust program starts in `fn main()`. `println!` prints a line; the `!` marks a macro; `{}` is filled with a value.
- `cargo run` = build + run. `cargo check` is the fast check. `cargo build --release` makes the optimised version.
- `target/` has all build output and is ignored by Git; `Cargo.lock` is written by Cargo.
- Run `cargo fmt` and `cargo clippy` before every commit; both must be clean.
- `rustc file.rs` compiles a single file, but Cargo is the normal way.
- RustRover runs the same Cargo commands from a green triangle.
- Compiler errors have a code, a place (`file:line:column`) and often a fix suggestion. Lesson 00.7 teaches you to read them.

## Sources and further reading

- [The Rust Programming Language — 1.2 Hello, World!](https://doc.rust-lang.org/book/ch01-02-hello-world.html) — the official walk-through of the first program with `rustc`.
- [The Rust Programming Language — 1.3 Hello, Cargo!](https://doc.rust-lang.org/book/ch01-03-hello-cargo.html) — the official explanation of Cargo, `Cargo.toml` and `cargo build`.
- [The Cargo Book — Creating a New Package](https://doc.rust-lang.org/cargo/guide/creating-a-new-project.html) — what `cargo new` does in detail.
- [The Cargo Book — Package Layout](https://doc.rust-lang.org/cargo/guide/project-layout.html) — where files go in a Cargo project.
- [Standard library — `println!`](https://doc.rust-lang.org/std/macro.println.html) — the reference of the macro, including all formatting options (Lesson 02.6 teaches them).
- [Clippy documentation](https://doc.rust-lang.org/clippy/) — how Clippy works and the list of its checks.

---

[🏠 Start](../START-HERE.md) · [← 00.5 Installing Rust and RustRover](05-install-rust-and-rustrover.md) · [📋 Phase 00](README.md) · [00.7 Reading errors and getting help →](07-reading-errors-and-getting-help.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
