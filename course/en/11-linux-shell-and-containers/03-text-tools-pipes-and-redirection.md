<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 11.2 The Linux file system](02-the-linux-file-system.md) · [📋 Phase 11](README.md) · [11.4 Users, groups and permissions →](04-users-groups-and-permissions.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/11-linux-shell-and-containers/03-text-tools-pipes-and-redirection.md) <!-- nav -->

# Lesson 11.3 — Text tools, pipes and redirection

> **Phase 11 — Linux, the shell and containers** · Lesson 3 of 16 · ⏱ about 2.5 hours (2 sittings)

## What you will learn

- The three standard streams of every program (input, output, errors) and how to redirect them with `>`, `>>`, `<`, `2>` and `2>&1`.
- How a pipe `|` connects programs, and how to build small pipelines.
- The everyday text tools: `head`, `tail`, `wc`, `sort`, `uniq`, `cut`, `tr`, `grep`, `sed`, `awk`, `xargs`, `tee`.
- How to analyse a log file in one line, and how your own Rust program fits into pipelines.

## Before you start

- You finished [Lesson 11.2 — The Linux file system](02-the-linux-file-system.md).
- You know pipes and `grep` at the level of [Lesson 00.4](../00-getting-started/04-terminal-basics.md) (section "Pipes and searching").
- Sections 1 to 5 are typed in the VM (`limactl shell dev`). Section 6 uses Rust and is typed on your **Mac**, where Rust is installed; redirections work the same way there.

## New words

| Word | Meaning |
|---|---|
| **standard stream** | One of three channels that every program gets at start: *standard input* (stdin), *standard output* (stdout) and *standard error* (stderr). |
| **file descriptor** (fd) | A small number the operating system uses to name an open file or stream. stdin is 0, stdout is 1, stderr is 2. |
| **redirection** | Sending a stream to a file (or reading it from a file) instead of the terminal. |
| **pipe** | The `\|` operator: the output of the left program becomes the input of the right program. |
| **pipeline** | Several commands joined by pipes. |
| **filter** | A program that reads text from stdin, changes or selects some of it, and writes the result to stdout. `grep`, `sort` and `cut` are filters. |
| **regular expression** (regex) | A short pattern language for text, such as `[0-9]+` for "one or more digits". `grep -E` and `sed` understand it. |
| **field / delimiter** | A *field* is one column of a line; a *delimiter* is the character that separates columns (a space, a comma, a colon). |
| **log file** | A text file where a program writes what happened, one event per line. |
| **command substitution** | `$(command)`: the shell runs the command and puts its output into the line. |

## 1. Three streams

Every program starts with three open channels:

```text
                 +-----------+
  keyboard  ---> | 0 stdin   |
                 |  program  | ---> 1 stdout ---> terminal screen
                 |           | ---> 2 stderr ---> terminal screen
                 +-----------+
```

- **stdin (0)** is where the program reads its input. By default: the keyboard.
- **stdout (1)** is where it writes its results. By default: the terminal.
- **stderr (2)** is where it writes error and progress messages. By default: also the terminal.

Two output channels exist so that *results* and *messages for the human* do not mix. The screen shows both, so you
do not see the difference, until you redirect. First, a small data file. It is a made-up web server log with five fields per line:
IP (Internet Protocol) address, method, path, status code and bytes sent. The addresses are from ranges reserved for examples.

```bash
mkdir ~/text-lab
cd ~/text-lab
cat > access.log <<'EOF'
203.0.113.5 GET /index.html 200 512
198.51.100.7 GET /api/items 200 1280
203.0.113.5 GET /missing.png 404 0
192.0.2.44 POST /api/items 201 64
203.0.113.5 GET /api/items 200 1280
198.51.100.7 GET /index.html 200 512
192.0.2.44 GET /admin 403 0
203.0.113.9 GET /api/items 500 0
198.51.100.7 GET /api/items 200 1280
203.0.113.5 GET /index.html 304 0
192.0.2.44 GET /missing.png 404 0
203.0.113.5 DELETE /api/items/7 204 0
EOF
wc -l access.log
```

```text
12 access.log
```

**What happens:**

1. `cat > access.log <<'EOF'` is a *here-document* (as in Lesson 00.4): everything up to the line `EOF` goes into the file. The quotes around `'EOF'` stop the shell from changing `$` signs inside.
2. `wc -l` ("word count", lines) counts the lines: 12.

## 2. Redirection

Run `ls` on one file that exists and one that does not:

```bash
ls access.log /nope
```

```text
ls: cannot access '/nope': No such file or directory
access.log
```

The first line is stderr, the second is stdout. (The order on your screen can differ.) Now redirect:

| Operator | Meaning |
|---|---|
| `> file` | write stdout to `file`; **replace** the file if it exists |
| `>> file` | **append** stdout to `file` |
| `< file` | read stdin from `file` |
| `2> file` | write stderr (stream 2) to `file` |
| `2>&1` | send stderr to the same place as stdout |
| `&> file` | send both streams to `file` (a short form in bash) |

```bash
ls access.log /nope > out.txt
```

```text
ls: cannot access '/nope': No such file or directory
```

**What happens:** stdout went into `out.txt` (it contains `access.log`); the error was not redirected, so it still
appears on the screen. Redirect the error and the output separately, then together:

```bash
ls access.log /nope 2> err.txt
cat err.txt
ls access.log /nope > both.txt 2>&1
cat both.txt
```

```text
access.log
ls: cannot access '/nope': No such file or directory
ls: cannot access '/nope': No such file or directory
access.log
```

**What happens:**

1. The first `ls` printed `access.log` (stdout, not redirected) and put the error into `err.txt`, which `cat` showed.
2. In the second, `> both.txt` points stdout at the file, and `2>&1` then says "stream 2 goes where stream 1 goes now": both are in `both.txt`.

> ⚠️ **Common mistake:** the **order** matters. `cmd > file 2>&1` sends both to the file. `cmd 2>&1 > file` first
> points stderr at the screen (where stdout is *now*) and only then moves stdout to the file, so the errors still
> appear on the screen. Read the redirections from left to right.

`>` destroys what was in the file; `>>` appends. Both create the file if it does not exist:

```bash
echo one > l.txt
echo two >> l.txt
cat l.txt
echo three > l.txt
cat l.txt
```

```text
one
two
three
```

**What happens:** after `echo three > l.txt` the file contains only `three`; `one` and `two` are gone.

The special file `/dev/null` ([Lesson 11.2](02-the-linux-file-system.md)) throws data away, so `2>/dev/null` means
"I do not want to see the errors". Use `<` to feed a file to a program that reads stdin:

```bash
wc -l < access.log
ls access.log /nope 2>/dev/null
```

```text
12
access.log
```

> 💡 **Tip:** `wc -l access.log` prints the file name, `wc -l < access.log` prints only the number, because `wc` does not know
> the name of its stdin. This is handy in scripts.

> 🔒 **Security:** `>` can overwrite an important file in a second. Before you type `> something`, check that
> the name is the one you mean. In scripts, keep output files in a dedicated directory. Bash can refuse to overwrite an existing file: after `set -o noclobber`, `echo x > old.txt` fails with `cannot overwrite existing file`, and `>|` forces the overwrite.

## 3. Pipes

A **pipe** connects the stdout of one command to the stdin of the next, with no file in between:

```text
  access.log --> [ cut ] --pipe--> [ sort ] --pipe--> [ uniq -c ] --> terminal
```

The commands run **at the same time**; data flows through as soon as it is produced. If the reader is slow, the
writer waits. No temporary file is needed, and a huge file is never loaded in memory as a whole. Only stdout is
piped; stderr still goes to the screen, unless you write `2>&1` before the pipe.

```bash
ls /usr/bin | wc -l
(echo out; echo err >&2) | wc -l
```

```text
444
err
1
```

**What happens:** the first number (the count of files in `/usr/bin`) depends on your system. The second command prints `out` to stdout (into the pipe)
and `err` to stderr (to the screen), so `wc -l` counted **one** line. With `2>&1` inside the parentheses, `wc` would count 2.

This idea, small programs that do one job and join through pipes, is the *Unix philosophy*. Learn the small tools, and you can
answer many questions without writing a program.

## 4. The text toolbox

All examples use `access.log` from Section 1.

### Look at the beginning, the end, the size

```bash
head -n 3 access.log
tail -n 2 access.log
wc access.log
```

```text
203.0.113.5 GET /index.html 200 512
198.51.100.7 GET /api/items 200 1280
203.0.113.5 GET /missing.png 404 0
192.0.2.44 GET /missing.png 404 0
203.0.113.5 DELETE /api/items/7 204 0
 12  60 419 access.log
```

**What happens:** `head -n 3` shows the first 3 lines, `tail -n 2` the last 2. `wc` without options prints lines, words and bytes.
`tail -f file` ("follow") keeps running and prints new lines as the program appends them; it is the standard way to watch a
log live. Stop it with `Ctrl + C`. To read a long file page by page use `less file` (arrow keys, `/word` to search, `q` to quit).

### Cut columns, sort, count

```bash
cut -d' ' -f1 access.log | sort | uniq -c
```

```text
      3 192.0.2.44
      3 198.51.100.7
      5 203.0.113.5
      1 203.0.113.9
```

**What happens:**

1. `cut -d' ' -f1` uses a space as the **delimiter** (`-d`) and keeps field number 1 (`-f1`): the IP address.
2. `sort` puts the lines in order. By default the order is alphabetical, line by line.
3. `uniq -c` collapses **neighbouring** equal lines into one and writes how many there were (`-c`, count).

`uniq` only compares neighbours, so you nearly always put `sort` before it. Without `sort`, equal lines that are not next to each other are not merged.
Now rank the result, most frequent first:

```bash
cut -d' ' -f1 access.log | sort | uniq -c | sort -rn | head -n 2
```

```text
      5 203.0.113.5
      3 198.51.100.7
```

**What happens:** `sort -rn` sorts **n**umerically (`-n`) in **r**everse order (`-r`). Without `-n`, `10` comes before `9`,
because text is compared character by character:

```bash
printf '10\n9\n100\n' | sort
printf '10\n9\n100\n' | sort -n
```

```text
10
100
9
9
10
100
```

`sort -u` keeps one copy of each line (unique). `printf` is like `echo`, but understands `\n` for a new line.

### Search: `grep`

```bash
grep 404 access.log
grep -c GET access.log
grep -v GET access.log
grep -n admin access.log
grep -E ' (4|5)[0-9][0-9] ' access.log
```

```text
203.0.113.5 GET /missing.png 404 0
192.0.2.44 GET /missing.png 404 0
10
192.0.2.44 POST /api/items 201 64
203.0.113.5 DELETE /api/items/7 204 0
7:192.0.2.44 GET /admin 403 0
203.0.113.5 GET /missing.png 404 0
192.0.2.44 GET /admin 403 0
203.0.113.9 GET /api/items 500 0
192.0.2.44 GET /missing.png 404 0
```

**What happens:**

1. `grep 404 access.log` prints the lines that contain `404`.
2. `-c` counts matching lines instead of printing them: 10 lines contain `GET`.
3. `-v` inverts: the lines that do **not** contain `GET`.
4. `-n` adds the line number (`7:`). `-i` ignores case, and `-r` searches all files in a directory recursively (`grep -r admin .`).
5. `-E` turns on *extended regular expressions*. `(4|5)[0-9][0-9]` means: a `4` or a `5`, then two digits, so it finds status codes 400 to 599. Spaces around it make sure we do not match inside other numbers.

> ⚠️ **Common mistake:** `grep 404` also matches `1404` or a path `/page404`. Make the pattern more exact (`' 404 '`) or cut the field out first.

### Change text: `tr`, `sed`

```bash
echo "Hello World" | tr a-z A-Z
echo "a,b,c" | tr ',' '\n'
sed 's/GET/get/' access.log | head -n 2
sed -n '2,3p' access.log
```

```text
HELLO WORLD
a
b
c
203.0.113.5 get /index.html 200 512
198.51.100.7 get /api/items 200 1280
198.51.100.7 GET /api/items 200 1280
203.0.113.5 GET /missing.png 404 0
```

**What happens:**

1. `tr` ("translate") replaces single **characters**: lowercase letters with uppercase, commas with new lines. `tr -s ' '` squeezes repeated spaces into one.
2. `sed` is a "stream editor". `s/GET/get/` is the substitution command: replace the first `GET` of each line with `get`. Add `g` at the end (`s/a/b/g`) to replace every occurrence in a line.
3. `sed -n '2,3p'` means: do not print by default (`-n`), print (`p`) lines 2 to 3.

`sed -i 's/old/new/' file` edits the **file itself**; make a copy first, because there is no undo.

### Use columns: `awk`

`awk` reads a line, splits it into fields `$1`, `$2` … and runs your small program on each line:

```bash
awk '{print $1, $4}' access.log | head -n 3
awk '{sum += $5} END {print sum}' access.log
awk '$4 >= 400 {print $2, $3, $4}' access.log
```

```text
203.0.113.5 200
198.51.100.7 200
203.0.113.5 404
4928
GET /missing.png 404
GET /admin 403
GET /api/items 500
GET /missing.png 404
```

**What happens:**

1. `{print $1, $4}` prints field 1 and field 4 of every line.
2. `{sum += $5}` adds field 5 (bytes) to a variable for every line; `END {...}` runs once after the last line. The total is 4928 bytes.
3. `$4 >= 400 {...}` is a *condition* before the braces: the action runs only for lines where the status is 400 or more.

Single quotes keep the shell from touching `$1`. `awk` is a small language; two or three patterns like these cover most daily needs.

### Tools that connect: `tee`, `xargs`, `$( )`

```bash
ls /etc | wc -l | tee count.txt
printf 'x.txt\ny.txt\n' | xargs touch
find . -name '*.txt' | xargs wc -l | tail -n 1
echo "the log has $(wc -l < access.log) lines"
```

```text
96
  6 total
the log has 12 lines
```

**What happens:**

1. `tee` copies stdin to a file **and** to stdout, like a T-junction. You see the number `96` (yours will differ) and it is saved in `count.txt` too.
2. `xargs` turns lines on its stdin into **arguments** of another command. Here it ran `touch x.txt y.txt`. The difference from a pipe: `touch` does not read stdin, so `... | touch` would not work; `xargs` makes it work. In the third command `xargs` builds `wc -l file1 file2 ...` from the names found; `tail -n 1` keeps the `total` line.
3. `$(command)` is **command substitution**: the shell runs the command and puts its output into the line before running `echo`.

> ⚠️ **Common mistake:** `find ... | xargs rm` breaks on names with spaces (it splits at spaces). The safe form separates names with the zero byte:
> `find . -name '*.tmp' -print0 | xargs -0 rm`. Run it with `echo` first (`xargs -0 echo rm`) to see what it would do.

## 5. A real question in one line

"Which request paths cause the most errors?" Break it into steps, and test each step before adding a pipe:

```bash
awk '$4 >= 400 {print $3}' access.log | sort | uniq -c | sort -rn
```

```text
      2 /missing.png
      1 /api/items
      1 /admin
```

**What happens:**

1. `awk` keeps the path (field 3) of the lines with a status of 400 or more.
2. `sort | uniq -c` counts equal paths. `sort -rn` puts the most frequent first.

Check all together: total bytes sent per IP address.

```bash
awk '{b[$1] += $5} END {for (ip in b) print ip, b[ip]}' access.log | sort
```

```text
192.0.2.44 64
198.51.100.7 3072
203.0.113.5 1792
203.0.113.9 0
```

**What happens:** `b[$1] += $5` uses an *associative array* (a table with text keys): for each IP it adds the bytes. After the last line, `for (ip in b)` prints every key and total. The `sort` makes the order stable, because `awk` does not promise any order.

> 📏 **Convention:** Build pipelines **step by step**: run the first command alone, look at the output, add `| next`, look again.
> If a pipeline grows beyond three or four steps, or you need it more than once, it becomes a script ([Lesson 11.8](08-bash-scripting-basics.md)).

## 6. Your Rust programs and the streams

Rust programs use the same three streams: `println!` writes to stdout, `eprintln!` writes to stderr, and `std::io::stdin()` reads stdin.
A program that follows this rule works in pipelines with `grep` and `sort` without any extra work. (This is also why programs
write their *logs* to stderr or stdout and leave the files to the shell; [Lesson 08.15](../08-software-design/15-the-twelve-factor-app.md) calls this a 12-factor habit, and [Lesson 09.10](../09-testing-debugging-benchmarking/10-logging-and-tracing.md) covers logging.)

Open a **Mac** terminal tab. We write a tiny filter, `p11-streams`: it counts the lines of its stdin that contain a word.

```bash
cd ~/projects/rust
cargo new code/p11-streams
```

Replace `code/p11-streams/src/main.rs`:

```rust
use std::io::{self, BufRead};

/// Counts how many lines contain `word`.
fn count_matching<I>(lines: I, word: &str) -> usize
where
    I: Iterator<Item = String>,
{
    lines.filter(|line| line.contains(word)).count()
}

fn main() {
    let Some(word) = std::env::args().nth(1) else {
        eprintln!("usage: p11-streams WORD < input");
        std::process::exit(2);
    };

    eprintln!("reading standard input...");
    let stdin = io::stdin();
    let lines = stdin
        .lock()
        .lines()
        .map(|line| line.expect("cannot read a line"));
    println!("{}", count_matching(lines, &word));
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn counts_only_lines_with_the_word() {
        let lines = ["a 404", "b 200", "c 404"].iter().map(|s| s.to_string());
        assert_eq!(count_matching(lines, "404"), 2);
    }
}
```

**What happens:**

1. `std::env::args().nth(1)` is the first argument after the program name. With no argument the program prints a usage message to **stderr** and exits with status 2 (a non-zero status means failure; Lesson 11.5).
2. The progress message `reading standard input...` goes to stderr with `eprintln!`, so it never pollutes the result.
3. `stdin.lock().lines()` gives the lines of stdin as an iterator. `count_matching` counts the ones that contain the word. The result goes to stdout with `println!`.
4. The function does not read stdin itself: it takes any iterator of lines, so the test can pass a fixed list. (This is the idea of testable design from [Phase 08](../08-software-design/README.md).)

Make a small input and try every kind of redirection (`-q` makes Cargo itself quiet):

```bash
printf 'a 404\nb 200\nc 404\n' > in.txt
cargo run -q -- 404 < in.txt
cat in.txt | cargo run -q -- 404
cargo run -q -- 404 < in.txt > count.txt
cat count.txt
cargo run -q -- 404 < in.txt 2>/dev/null
cargo run -q
echo "status: $?"
```

```text
reading standard input...
2
reading standard input...
2
reading standard input...
2
2
usage: p11-streams WORD < input
status: 2
```

**What happens:** lines 1–2 come from the first command (stderr then stdout). Lines 3–4 are the same with a pipe from `cat`.
The third command sent stdout to `count.txt`, so only the progress message appears on the screen; `cat count.txt` prints the result `2`.
With `2>/dev/null` only the result `2` is visible. The last command has no argument: the usage message and status 2.

🧪 **Testing:**

```bash
cargo test
```

```text
running 1 test
test tests::counts_only_lines_with_the_word ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

Clean up when you finish: `rm in.txt count.txt` in the project folder, and in the VM `rm -r ~/text-lab`.

## Common mistakes

1. **`sort` forgotten before `uniq -c`.** The counts are wrong because only neighbours are merged. *Fix:* `... | sort | uniq -c`.
2. **`>` instead of `>>`.** The old content of the log is gone. *Fix:* use `>>` to append; check the name twice before `>`.
3. **`2>&1` in the wrong place.** `cmd 2>&1 > file` does not put the errors in the file. *Fix:* `cmd > file 2>&1`.
4. **Reading and writing the same file.** `sort access.log > access.log` empties the file first (the shell opens it for writing before `sort` runs). *Fix:* write to another file: `sort access.log > sorted.log`.
5. **Unquoted patterns.** `grep ' 404 ' file` is fine, but `awk {print $1}` without quotes is changed by the shell. *Fix:* put the awk program in single quotes.
6. **Numbers sorted as text.** `10` before `9`. *Fix:* `sort -n`.

## Try it yourself

In the VM, in `~/text-lab`:

1. Show the lines with a status of 200 and count them in one command (`grep -c`). Expected: `5`.
2. Show the distinct methods used (`GET`, `POST`, `DELETE`), one per line, sorted. (`cut`, `sort -u`.)
3. Save the lines of status 404 into `not-found.log`, and then append the lines of status 403 to the same file. Check it with `cat`.
4. Run `ls access.log /nope > all.txt 2>&1` and then `ls access.log /nope 2>&1 > all2.txt`. Compare what appears on the screen and what each file contains.

## Exercises

### Exercise 1 — Count with a pipe (easy)

How many requests did the IP address `203.0.113.5` make? Use `grep` and `wc`, then try the shorter way with only `grep`.

<details>
<summary>Hint</summary>

`grep` prints the matching lines; `wc -l` counts lines. `grep` also has an option that counts.

</details>

<details>
<summary>Answer</summary>

```bash
grep '^203.0.113.5 ' access.log | wc -l
grep -c '^203.0.113.5 ' access.log
```

Both print `5`.

Why it works: `^` anchors the pattern to the start of the line, and the space after the address prevents matching `203.0.113.55` (the dots also match any character, which is fine for this data). `-c` counts the matching lines.

</details>

### Exercise 2 — Add tests to the Rust filter (medium)

In `code/p11-streams`, add two tests to `count_matching`: one for **empty input** (the result is 0) and one for a word that **never occurs** (also 0).

<details>
<summary>Hint</summary>

Pass `std::iter::empty()` for the empty case. The function accepts any iterator of `String`, so the iterator needs a concrete item type: `std::iter::empty::<String>()`.

</details>

<details>
<summary>Answer</summary>

Add inside `mod tests`:

```rust
    #[test]
    fn empty_input_counts_zero() {
        assert_eq!(count_matching(std::iter::empty::<String>(), "404"), 0);
    }

    #[test]
    fn missing_word_counts_zero() {
        let lines = ["a 200", "b 200"].iter().map(|s| s.to_string());
        assert_eq!(count_matching(lines, "404"), 0);
    }
```

```text
running 3 tests
test tests::counts_only_lines_with_the_word ... ok
test tests::missing_word_counts_zero ... ok
test tests::empty_input_counts_zero ... ok

test result: ok. 3 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

(The order of the `test` lines can differ on your computer: tests run in parallel.)

Why it works: the function only needs an iterator of lines, so tests do not need a real stdin or a file. Edge cases (empty input, no match) are where bugs hide.

</details>

### Exercise 3 — Bytes per IP, biggest first (hard)

Print the total bytes per IP address, the biggest total first, in the form `198.51.100.7 3072`.

<details>
<summary>Hint</summary>

Section 5 has the `awk` program. The order is the job of `sort`: sort numerically (`-n`), in reverse, by the **second** field (`-k2`).

</details>

<details>
<summary>Answer</summary>

```bash
awk '{b[$1] += $5} END {for (ip in b) print ip, b[ip]}' access.log | sort -k2 -rn
```

```text
198.51.100.7 3072
203.0.113.5 1792
192.0.2.44 64
203.0.113.9 0
```

Why it works: `awk` produces `IP total` lines in any order; `sort -k2 -rn` orders them by the second column, as numbers, largest first.

</details>

## Quiz

1. What are the numbers of stdin, stdout and stderr?

<details>
<summary>Answer</summary>

0, 1 and 2.

</details>

2. What is the difference between `>` and `>>`?

<details>
<summary>Answer</summary>

`>` replaces the content of the file; `>>` appends to the end.

</details>

3. Why does `cmd 2>&1 > file` still print errors on the screen?

<details>
<summary>Answer</summary>

Redirections are applied from left to right. `2>&1` points stderr at where stdout is at that moment (the screen); only afterwards stdout moves to the file.

</details>

4. Why must `sort` come before `uniq`?

<details>
<summary>Answer</summary>

`uniq` merges only neighbouring equal lines. `sort` puts equal lines next to each other.

</details>

5. Why does a Rust program print its progress messages with `eprintln!` and not with `println!`?

<details>
<summary>Answer</summary>

So that the result on stdout stays clean for the next program in the pipeline or for a file; the messages for the human go to stderr.

</details>

## Summary

- Every program has stdin (0), stdout (1) and stderr (2). Rust: `stdin()`, `println!`, `eprintln!`.
- `>` replaces, `>>` appends, `<` reads from a file, `2>` redirects errors, `2>&1` joins errors with output; the order is read left to right.
- A pipe `|` connects stdout to the next stdin; commands run at the same time.
- Core tools: `head`, `tail`, `wc`, `sort` (`-n`, `-r`, `-u`), `uniq -c`, `cut`, `tr`, `grep` (`-c -v -n -i -E`), `sed 's/a/b/'`, `awk`, `tee`, `xargs`.
- `$(command)` puts a command's output into another command.
- Build pipelines step by step; when a pipeline gets long or is used again, turn it into a script.

## Sources and further reading

- [GNU Bash Reference Manual — Redirections](https://www.gnu.org/software/bash/manual/html_node/Redirections.html) — all redirection operators and their order rules.
- [GNU Bash Reference Manual — Pipelines](https://www.gnu.org/software/bash/manual/html_node/Pipelines.html) — how pipelines run.
- [GNU grep manual](https://www.gnu.org/software/grep/manual/grep.html) — every `grep` option and the regular-expression syntax.
- [GNU sed manual](https://www.gnu.org/software/sed/manual/sed.html) — the stream editor in detail.
- [POSIX specification of awk](https://pubs.opengroup.org/onlinepubs/9799919799/utilities/awk.html) — the standard definition of the `awk` language (POSIX, the Portable Operating System Interface, is the standard that Unix-like systems such as Linux and macOS follow).
- [Rust standard library — std::io](https://doc.rust-lang.org/std/io/index.html) — `stdin`, `stdout`, `stderr` and the `BufRead` trait.

---

[🏠 Start](../START-HERE.md) · [← 11.2 The Linux file system](02-the-linux-file-system.md) · [📋 Phase 11](README.md) · [11.4 Users, groups and permissions →](04-users-groups-and-permissions.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
