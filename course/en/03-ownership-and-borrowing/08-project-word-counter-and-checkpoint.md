<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 03.7 Fixing common ownership errors](07-fixing-common-ownership-errors.md) · [📋 Phase 03](README.md) · [Phase 04 — Structs, enums, collections and errors →](../04-structs-enums-collections-errors/README.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/03-ownership-and-borrowing/08-project-word-counter-and-checkpoint.md) <!-- nav -->

# Lesson 03.8 — Project: word counter, and checkpoint

> **Phase 03 — Ownership and borrowing** · Lesson 8 of 8 · ⏱ about 3 hours (2 sittings)

## What you will learn

- How to build a complete small command-line tool where ownership and borrowing do real work: one owner of the text, many functions that only borrow it.
- How to read a file or standard input, use command-line arguments, print errors to the right place and return an exit code.
- How to work in milestones with tests, `cargo fmt`, `cargo clippy`, a README and **one Git commit per milestone** on a branch.
- Whether you are ready for Phase 04 (the checkpoint at the end of this lesson).

## Before you start

- You finished [Lesson 03.7 — Fixing common ownership errors](07-fixing-common-ownership-errors.md).
- Your course folder `~/projects/rust` is a Git repository (Phase 01), and `git status` works there. Its `.gitignore` must contain the line `target/` so that Cargo's build folders are not committed ([Lesson 01.3](../01-git-and-version-control/03-your-first-repository-and-commits.md)).
- Keep this sample text at hand. In milestone M1 you will save it as `sample.txt` inside the project folder (do not create the folder yet: `cargo new` creates it itself). The file must contain exactly these five lines (the fourth is empty):

```text
Rust is a systems programming language.
Rust gives you control over memory, and the compiler checks your work.
Ownership is the wonderful idea behind it all.

Learn it slowly. Rust rewards patience.
```

## New words

| Word | Meaning |
|---|---|
| **CLI** (Command-Line Interface) | A program that you use by typing its name and arguments in the terminal. |
| **argument** | A word you type after the program name, such as `sample.txt` in `word-counter sample.txt`. |
| **standard input** (*stdin*) | The stream of data that flows into a program. By default it is your keyboard; with a pipe it comes from another program or a file. |
| **standard output** (*stdout*) | The stream where a program prints normal results. `println!` writes here. |
| **standard error** (*stderr*) | A second output stream for error messages. `eprintln!` writes here. It keeps errors apart from results. |
| **EOF** (End Of File) | The signal that there is no more input. In the terminal you send it with **Ctrl-D**. |
| **pipe** (`\|`) | A shell feature that sends the output of one program into the input of another: `echo hi \| my-program`. |
| **exit code** | A number a program returns to the system when it ends. `0` means success, anything else means failure. |
| **`Option`** | A value that is `Some(value)` or `None`. Used when a value may be missing. |
| **`Result`** | A value that is `Ok(value)` or `Err(error)`. Used when an operation can fail. |
| **`match`** | A way to run different code for each possible shape of a value, such as `Some` / `None` or `Ok` / `Err`. |
| **milestone** | A small step of a project with its own goal and a way to check that it is done. |
| **acceptance test** | A check that shows the finished work behaves as the contract says. Here: commands with expected output. |
| **`wc`** | A classic Unix program ("word count") that prints line, word and byte counts. Our tool is a small cousin of it. |

## 1. New tools for this project

The project needs a few things you have not used yet. Each is small. Read this section once, then use it as a reference.

### Command-line arguments

When you run `cargo run -- sample.txt`, everything after `--` is given to **your** program. (`--` tells Cargo: "stop reading options, the rest is for the program".) You get them with `std::env::args()`:

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

**What happens:**

1. `use std::env;` makes the name `env` available (`use` brings a name from the standard library into the file).
2. `env::args()` produces the arguments one by one. `.collect()` gathers them into a `Vec<String>` (Lesson 05.5 explains iterators and `collect`). **The first item is always the program's own path**, so the user's first argument is at index 1.
3. `args.get(1)` returns an **`Option`**: `Some(&String)` if there is an item at index 1, `None` if not. It never panics (unlike `args[1]`).
4. `match` has one arm for each case: `Some(path)` runs when a value exists and gives it the name `path`; `None` runs otherwise. The compiler checks that you covered both cases.

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

(`-q` means "quiet": Cargo prints nothing of its own. The program name in the first item depends on your project name.)

### Reading a whole file

`std::fs::read_to_string(path)` reads a file into a `String`. Reading can fail (no such file, no permission), so it returns a **`Result`**, which is either `Ok(the_string)` or `Err(the_error)`. You `match` on it:

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

Whoever gets the `Ok(text)` becomes the **owner** of the `String`. That is exactly the "one owner" rule: from now on, the text has one owner in your program.

### Reading standard input

`std::io::read_to_string(std::io::stdin())` reads everything from standard input until EOF:

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

`expect("message")` means "if this is an `Err`, stop the program with this message". It is fine in a quick demo; in the project you will print a friendly error instead. In an interactive terminal, the program waits for you to type; press **Ctrl-D** on an empty line to say "that's all".

### Errors go to stderr; failure has an exit code

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

`eprintln!` prints to stderr. The shell variable `$?` holds the exit code of the last command. `2>/dev/null` throws stderr away: only the normal output remains. A good tool prints results on stdout, problems on stderr, and exits with a non-zero code when it fails.

> 📏 **Convention:** these three habits are what make a command-line tool usable in scripts. Programs that print errors to stdout, or return `0` after a failure, are surprising and hard to automate.

## Common mistakes

**1. Using `args[1]` instead of `args.get(1)`.** If the user gave no argument, `args[1]` panics. `get` returns an `Option` you can handle.

**2. Forgetting that `args[0]` is the program path.** The first user argument is at index 1.

**3. Printing errors with `println!`.** Use `eprintln!` for errors and `std::process::exit(1)` to signal failure.

**4. Waiting forever on stdin.** If you run the program with no file and no pipe, it waits for typing. Press Ctrl-D, or give it a file or a pipe.

**5. Counting letters with `len()`.** It counts bytes (Lesson 03.6). The project needs both numbers.

## Try it yourself

1. Create a scratch project: `cargo new code/p03-cli-tools`. Put the `args` program into `src/main.rs`. Run it with no arguments, with `sample.txt`, and with three arguments.
2. Replace the program with the file-reading one. First create `sample.txt` in the project folder with the five lines from "Before you start". What does the `Err` message say when you misspell the file name?
3. Replace it with the standard-input one. Run it three ways: type something and press Ctrl-D; `printf 'a b c\n' | cargo run -q`; `cargo run -q < sample.txt`.
4. Run the exit-code program and check `echo $?`.
5. Commit this playground on `main`: `git add code/p03-cli-tools && git commit -m "p03: cli tools playground"`.

## The project

### What you will build

A small command-line tool, **`p03-word-counter`**, a cousin of the Unix `wc`. It reads a text from a file (or from standard input) and prints:

- how many **lines**, **words**, **characters** and **bytes** it has;
- the **longest word**;
- optionally, how many times a given **word** appears (ignoring upper/lower case).

The project is about ownership. The design has one rule: **`main` owns the text; every analysis function only borrows it.**

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

### Contract

**Where it lives:** `code/p03-word-counter` (a Cargo binary project). All code may be in `src/main.rs`.

**Usage:**

```text
p03-word-counter [FILE|-] [WORD]
```

- No arguments, or `-` as the first argument: read the text from standard input.
- Otherwise the first argument is the path of a text file.
- The optional second argument `WORD` is a word to count.

With Cargo: `cargo run -q -- sample.txt rust`.

**Output on success** (to stdout, exit code `0`), exactly these lines in this order:

```text
lines: <number>
words: <number>
chars: <number>
bytes: <number>
longest word: <word>
occurrences of "<WORD>": <number>      (only if WORD was given)
```

**Definitions:**

| Item | Rule |
|---|---|
| lines | the number of lines, as counted by `str::lines()` (so a final line break does not add an empty line, and an empty text has 0 lines) |
| words | pieces separated by any whitespace (`split_whitespace`); punctuation stays part of the word |
| chars | the number of `char`s (not bytes) |
| bytes | `text.len()` |
| longest word | the word with the most `char`s; on a tie the **first** one; if there are no words, print `longest word: (none)` |
| occurrences | the number of words equal to `WORD` when both are lowercased; the whole word must match (so `Rust,` does not match `rust`) |

**Errors:** if the file cannot be read, print `error: cannot read '<path>': <reason>` to **stderr** and exit with code `1`. Example: `error: cannot read 'missing.txt': No such file or directory (os error 2)`.

**Required function signatures** (these are what the tests call; keep the names and types):

```rust
fn count_lines(text: &str) -> usize
fn count_words(text: &str) -> usize
fn count_chars(text: &str) -> usize
fn longest_word(text: &str) -> &str
fn count_occurrences(text: &str, word: &str) -> usize
```

**Ownership rules (checked in M5):**

1. Every analysis function takes `&str`, never `String` or `&String`.
2. `longest_word` returns a slice of its input. It must not allocate or clone.
3. The text is owned by one variable in `main` (or in a reading function that returns it). There is no `.clone()` of the text anywhere.
4. Allowed allocations: `to_lowercase()` for the case-insensitive comparison.

**Quality rules (every project in this course):** `cargo test` passes, `cargo clippy` shows no warnings, `cargo fmt --check` is clean, a short `README.md` exists in the project, and each milestone is a separate Git commit.

### Step-by-step plan

Work on a branch, so that `main` always holds working code (see [Lesson 01.6](../01-git-and-version-control/06-branches-and-merging.md)).

#### M1 — Skeleton and reading input

**Goal:** create the project and a branch; read the text from a file or from stdin; print `bytes: <n>`; report read errors correctly.

**Guidance:**

1. From `~/projects/rust`: `git switch -c p03-word-counter`, then `cargo new code/p03-word-counter`. (Cargo does not create a new Git repository inside an existing one; check with `git status`.) Save the five lines of sample text as `sample.txt` in the project folder.
2. Write a function `read_input(path: Option<&String>) -> Result<String, String>`: it returns the text, or an error message.
3. In `main`, collect the arguments, call `read_input(args.get(1))`, `match` the result: on `Ok` continue; on `Err` print `error: …` with `eprintln!` and call `process::exit(1)`.
4. Print `bytes: ` followed by `text.len()`.

<details>
<summary>Hint</summary>

The `match` inside `read_input` can have two arms. A **match guard** (`if` after a pattern) lets one arm test a condition: `Some(p) if p != "-"` means "a path was given and it is not a dash". Everything else (`_`) reads stdin.

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

**Done when:**

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

Then run `cargo clippy` and `cargo fmt`, and commit: `git add code/p03-word-counter && git commit -m "p03-word-counter: M1 read input from file or stdin"`.

#### M2 — Counting lines, words and characters

**Goal:** add `count_lines`, `count_words` and `count_chars`, with tests, and print four result lines.

**Guidance:**

1. Write the three functions with the signatures from the contract. Each is one line of code using a `&str` method from Lesson 03.6, followed by `.count()`.
2. Write tests **first** (or at least together): empty text, one word, extra spaces, a final line break, a blank line in the middle, and a Cyrillic word to prove that `chars` and `bytes` differ.
3. Print `lines:`, `words:`, `chars:`, `bytes:` in `main`, passing `&text`.

<details>
<summary>Hint</summary>

Three useful methods: `lines()`, `split_whitespace()` and `chars()`. Each gives something you can call `.count()` on. In tests, `assert_eq!(count_lines("one\n\nthree"), 3);` documents the blank-line rule.

</details>

**Done when:**

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

(`N` is the number of your tests; at least 3.) Commit: `p03-word-counter: M2 count lines, words and chars`.

#### M3 — The longest word (returning a slice)

**Goal:** add `longest_word(text: &str) -> &str`, which returns a slice of the input, and print `longest word: …`.

**Guidance:**

1. Keep two variables: the best word so far (a `&str`, starting as `""`) and its length in `char`s (starting at `0`).
2. For each word from `split_whitespace()`, compute its length with `word.chars().count()`. If it is **strictly longer** than the best, replace the best. (Strictly longer means a tie keeps the first.)
3. Return the best. The returned `&str` points into the input text; no copy is made.
4. In `main`, if the result is empty, print `longest word: (none)`.
5. Tests: normal text, empty text, a tie (first wins), and a Cyrillic word that is longer in characters but not in bytes.

<details>
<summary>Hint</summary>

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

Why can `longest = word;` compile? `word` is a slice of `text`, and the function's signature says the result is a view into `text`. If you had tried to store a lowercased `String` instead, you would meet `E0597` (Lesson 03.7, case 7).

</details>

**Done when:**

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

Commit: `p03-word-counter: M3 longest word as a slice of the input`.

#### M4 — Counting a given word

**Goal:** support the optional second argument and `count_occurrences(text, word)`.

**Guidance:**

1. Lowercase `word` once, before the loop (`let target = word.to_lowercase();`).
2. For each token from `split_whitespace()`, lowercase it and compare it with `target` using `==`. Count the matches.
3. In `main`, if `args.get(2)` is `Some(word)`, print `occurrences of "<word>": <n>` (print the word as the user typed it).
4. Tests: different cases (`Rust rust RUST`), a word that is a prefix of another (`rusty` must not match `rust`), and no match.

<details>
<summary>Hint</summary>

`to_lowercase()` creates a new `String`. Comparing a `String` with a `String` works with `==`. To print only when the argument exists, `if let Some(word) = args.get(2) { … }` is a short form of `match` with a single `Some` arm (Lesson 04.5 teaches `if let`). `args.get(2)` gives `Option<&String>`; pass `word` to the function, which expects `&str` (deref coercion, Lesson 03.6).

</details>

**Done when:**

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

Commit: `p03-word-counter: M4 count occurrences of a word`.

#### M5 — Quality, README, ownership review, merge

**Goal:** make the project clean and documented, check the ownership rules, and merge the branch.

**Guidance:**

1. Run `cargo fmt`, then `cargo clippy` (fix every warning), then `cargo test`.
2. Check the ownership rules of the contract. For example, `grep -n "clone()" src/main.rs` must print nothing; every analysis function takes `&str`.
3. Write `README.md` in the project: one paragraph about what the tool does, the usage line, an example run, how to run the tests, and a short "Ownership notes" section (who owns the text; which function returns a borrowed slice).
4. Commit: `p03-word-counter: M5 README, fmt and clippy clean`.
5. Merge the branch into `main` and tag the milestone:

   ```bash
   git switch main
   git merge p03-word-counter
   git tag p03-done
   git log --oneline
   ```

<details>
<summary>Hint</summary>

A good README is short. Include the exact command `cargo run -q -- sample.txt rust` and its expected output. If `git merge` says "Fast-forward", that is fine: your branch only had new commits on top of `main`. Delete the finished branch with `git branch -d p03-word-counter`.

</details>

**Done when:**

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

(The test count is yours; with the tests suggested above it is at least 7. `grep` prints nothing when there is no match. `cargo clippy` must show no `warning:` lines; the exact `Finished` line is not important. The log shows your own hashes first.)

### Stretch goals

- **Punctuation:** strip punctuation from the ends of each word before comparing (so `Rust,` matches `rust`). Hint: `word.trim_matches(|c: char| !c.is_alphanumeric())` — the part in `| … |` is a *closure*, a tiny unnamed function (Lesson 05.4).
- **Top words:** print the three most frequent words. This needs a `HashMap` (Lesson 04.9); come back after Phase 04.
- **More options:** accept several files and print totals, as `wc` does. Keep each file's text owned by one variable at a time.
- **Pull request:** push the branch to GitHub or GitLab and open a pull request for yourself (Lessons 01.8 and 01.9). Read your own diff before merging.
- **Benchmark:** feed it a 100 MB file (`yes "some words here" | head -c 100000000 > big.txt`) and measure with `time cargo run --release -q -- big.txt`. Is the cost in reading or counting?

## Self-test quiz

Answer from memory first, then open the answers.

1. What is the difference between the stack and the heap? Give one thing that lives on each.

<details>
<summary>Answer</summary>

The stack holds fixed-size values in frames that appear and disappear with function calls (fast, small). The heap holds data of unknown or changing size, handed out by an allocator (flexible, slower, must be freed). Example: an `i32` lives on the stack; the letters of a `String` live on the heap, while the three-word handle of the `String` is on the stack.

</details>

2. State the three ownership rules.

<details>
<summary>Answer</summary>

Each value has an owner; there is only one owner at a time; when the owner goes out of scope, the value is dropped.

</details>

3. What exactly is copied by `let t = s;` when `s` is a `String`? What happens to `s`?

<details>
<summary>Answer</summary>

Only the 24-byte handle (pointer, length, capacity). The heap letters are not copied. `s` is marked as moved and cannot be used again.

</details>

4. Which of these are `Copy`: `i32`, `String`, `&str`, `(i32, bool)`, `Vec<u8>`?

<details>
<summary>Answer</summary>

`i32`, `&str` and `(i32, bool)` are `Copy`. `String` and `Vec<u8>` are not.

</details>

5. When should you use `.clone()`?

<details>
<summary>Answer</summary>

When you truly need two independent owners of the data. It makes a deep copy at a real cost, so try borrowing first.

</details>

6. A function only needs to read a `String`. What should its parameter type be, and what does the caller write?

<details>
<summary>Answer</summary>

`&str`. The caller writes `&my_string` (deref coercion turns `&String` into `&str`). A string literal also works directly.

</details>

7. State the borrowing rules.

<details>
<summary>Answer</summary>

At any time: any number of shared references (`&T`) or exactly one mutable reference (`&mut T`), never both; references must always be valid.

</details>

8. Why does `for n in &v { v.push(*n); }` not compile?

<details>
<summary>Answer</summary>

The loop holds a shared borrow of `v` while `push` needs a mutable borrow (`E0502`). `push` could move the buffer and invalidate the loop's view.

</details>

9. When does a borrow end?

<details>
<summary>Answer</summary>

At the last use of the reference (the NLL rule, Non-Lexical Lifetimes), not at the end of the block.

</details>

10. Why can a function not return `&String` that points to a local `String`?

<details>
<summary>Answer</summary>

The local is dropped when the function ends, so the reference would dangle (`E0106` or `E0515`). Return the `String` itself.

</details>

11. What is the type of the literal `"hello"`, and how big is it on a 64-bit machine?

<details>
<summary>Answer</summary>

`&str`, 16 bytes (pointer and length). The text itself lives inside the program.

</details>

12. Why does `"привет".len()` equal 12 but `.chars().count()` equal 6?

<details>
<summary>Answer</summary>

`len()` counts bytes; each Cyrillic letter takes 2 bytes in UTF-8 (Unicode Transformation Format, 8-bit: the way Rust stores text as bytes). `chars()` goes through characters, and there are 6.

</details>

13. Why is `let c = names[0];` an error for `Vec<String>`, and what are two fixes?

<details>
<summary>Answer</summary>

It would move a `String` out of the vector's middle (`E0507`). Borrow it (`&names[0]`) or clone it (`names[0].clone()`).

</details>

14. What does `E0382` mean, and what are the three labels to look for?

<details>
<summary>Answer</summary>

Use of a moved value. The labels show where the value was created, where it was moved, and where it was used afterwards.

</details>

15. In this project, why can `longest_word` return a `&str` without allocating?

<details>
<summary>Answer</summary>

Each word from `split_whitespace()` is a slice of the input text. Returning one of them is returning a view into the caller's text; the signature ties the result to the input.

</details>

## Practical tasks

Each task should have tests. Put the tasks in `code/p03-checkpoint` (one `src/bin/` file each, or one project per task) and commit each one.

### Task 1 — First and last word

Write `fn first_and_last_word(text: &str) -> (&str, &str)`. For `"  the quick brown fox "` it returns `("the", "fox")`; for one word, that word twice; for no words, `("", "")`. No allocation.

<details>
<summary>Hint</summary>

Loop over `text.split_whitespace()`. Remember the first word once (use a `bool` flag), and update `last` on every word.

</details>

<details>
<summary>Answer</summary>

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

### Task 2 — Remove vowels

Write `fn remove_vowels(text: &str) -> String` that returns the text without the letters `aeiouAEIOU`. The input must stay usable by the caller. Test normal text, text with non-English letters (they stay), and the empty string.

<details>
<summary>Hint</summary>

Create `let mut result = String::new();`, go through `text.chars()`, and `push` each character that is not in `"aeiouAEIOU"`. `"aeiouAEIOU".contains(c)` tests membership.

</details>

<details>
<summary>Answer</summary>

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

### Task 3 — Append all

Write `fn append_all(dst: &mut String, parts: &[String])` that appends every part to `dst`, separated by `", "` (no separator before the first part, and nothing is appended for an empty slice). After the call in `main`, the vector of parts must still be usable.

<details>
<summary>Hint</summary>

`parts.iter().enumerate()` gives `(index, part)` pairs. Push `", "` when the index is greater than 0, then `push_str(part)`. `part` is a `&String`, which works where `&str` is expected.

</details>

<details>
<summary>Answer</summary>

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

### Task 4 — Debug four errors

This program has four ownership errors. Fix them all, in the order the compiler reports them, with the smallest sensible changes. Commit after you finish. The expected output is shown below the code.

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

Expected output:

```text
Dr. Ada: 2 people
Ada is still usable
```

<details>
<summary>Hint</summary>

The compiler reports: `E0596` (the parameter is not `mut`), `E0507` (cannot move out of an index), `E0502` (push while `view` is alive and used later) and `E0382` (use of `first` after it was moved). Fixes: `mut name`; `people[0].clone()`; print `view.len()` before the `push`; pass `first.clone()` to `add_title`.

</details>

<details>
<summary>Answer</summary>

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

A better design avoids the two clones: make `add_title` take `&str` and return a new `String` (`format!("Dr. {name}")`), and borrow `&people[0]` instead of cloning it. Both versions are valid; the second one is cheaper.

</details>

### Task 5 — Draw the memory

For this code, draw (in text) what lives on the stack and on the heap after the last line, and say how many separate heap allocations exist:

```rust
let a = String::from("hi");
let b = &a;
let c = a.clone();
```

<details>
<summary>Hint</summary>

`a` is a handle plus heap letters. `b` is an 8-byte address of `a`. `c` is a clone: a second handle plus a second piece of heap.

</details>

<details>
<summary>Answer</summary>

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

Two heap allocations. `b` owns nothing; it points to `a`'s handle. `a` and `c` are independent owners.

</details>

## If you got something wrong

| Topic | Review |
|---|---|
| Stack, heap, pointers, capacity | [Lesson 03.1 — Memory: the stack and the heap](01-memory-stack-and-heap.md) |
| The three ownership rules, scope, drop order, moving into functions | [Lesson 03.2 — The ownership rules](02-ownership-rules.md) |
| Move vs `Copy` vs `.clone()`, `E0382`, `E0507` | [Lesson 03.3 — Move, copy and clone](03-move-copy-and-clone.md) |
| `&T`, dereferencing, dangling references, NLL | [Lesson 03.4 — References and borrowing](04-references-and-borrowing.md) |
| `&mut T`, the borrowing rules, `E0499`, `E0502`, `E0506` | [Lesson 03.5 — Mutable references and the borrowing rules](05-mutable-references-and-the-borrow-rules.md) |
| Slices, `String` vs `&str`, UTF-8, string methods | [Lesson 03.6 — Slices and string types](06-slices-and-string-types.md) |
| Reading compiler errors, the fix toolbox, Clippy, Git as a safety net | [Lesson 03.7 — Fixing common ownership errors](07-fixing-common-ownership-errors.md) |
| Commits, branches and merging for the project | [Lesson 01.3 — Your first repository and commits](../01-git-and-version-control/03-your-first-repository-and-commits.md) and [Lesson 01.6 — Branches and merging](../01-git-and-version-control/06-branches-and-merging.md) |
| Writing tests | [Lesson 02.8 — Your first tests](../02-programming-basics/08-your-first-tests.md) |

## What's next

You now have the most important idea of Rust in your hands. From here on, nearly everything is built on the questions "who owns this?" and "who may borrow it, and for how long?".

In **Phase 04** you will use these ideas to build your own types: structs, enums, `Option`, `match`, modules, `Vec`, `HashMap` and proper error handling. You will start with [Lesson 04.1 — Structs](../04-structs-enums-collections-errors/01-structs.md). Later, in Phase 05, lifetimes (Lesson 05.3) will finish the story that started with `E0106` in Lesson 03.4.

If you scored well on the quiz and finished the project with clean `fmt` and `clippy`, move on. If one area felt shaky, repeat its lesson's exercises before you continue. Ownership is worth the extra day.

## Summary

- A CLI tool reads **arguments** (`env::args`, `get` returns an `Option`), reads **input** (`fs::read_to_string` returns a `Result`; `io::read_to_string(io::stdin())` reads stdin) and reports failure with **stderr** and a non-zero **exit code**.
- `match` handles each case of an `Option` or a `Result`; the compiler makes sure no case is forgotten.
- The project's design uses ownership well: `main` owns the text; analysis functions take `&str`; `longest_word` returns a slice of its input without allocating.
- Work in milestones: tests with each step, `cargo fmt`, `cargo clippy`, one commit per milestone on a branch, a README, then merge and tag.
- The checkpoint covers the whole phase: memory, the three ownership rules, moves/`Copy`/`clone`, shared and mutable references, the borrowing rules, slices and strings, and reading errors.

## Sources and further reading

- [The Rust Programming Language — 12 An I/O Project: Building a Command Line Program](https://doc.rust-lang.org/book/ch12-00-an-io-project.html) — the official project that builds a bigger tool (you will do a version of it in Phase 05).
- [std::env::args](https://doc.rust-lang.org/std/env/fn.args.html) — command-line arguments, including a note about invalid Unicode.
- [std::fs::read_to_string](https://doc.rust-lang.org/std/fs/fn.read_to_string.html) — reading a file into a `String`.
- [std::io::read_to_string](https://doc.rust-lang.org/std/io/fn.read_to_string.html) — reading everything from a reader such as standard input.
- [std::process::exit](https://doc.rust-lang.org/std/process/fn.exit.html) — ending the program with an exit code.
- [Rustlings](https://github.com/rust-lang/rustlings) — small exercises; do the groups on move semantics, references and strings to practise this phase.

---

[🏠 Start](../START-HERE.md) · [← 03.7 Fixing common ownership errors](07-fixing-common-ownership-errors.md) · [📋 Phase 03](README.md) · [Phase 04 — Structs, enums, collections and errors →](../04-structs-enums-collections-errors/README.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
