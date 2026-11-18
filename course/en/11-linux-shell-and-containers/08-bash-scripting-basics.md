<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 11.7 Package managers](07-package-managers.md) · [📋 Phase 11](README.md) · [11.9 Robust bash scripts →](09-robust-bash-scripts.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/11-linux-shell-and-containers/08-bash-scripting-basics.md) <!-- nav -->

# Lesson 11.8 — Bash scripting basics

> **Phase 11 — Linux, the shell and containers** · Lesson 8 of 16 · ⏱ about 2.5 hours (2 sittings)

## What you will learn

- What a shell script is, what the first line `#!` does, and how to run a script.
- Variables, quoting, command substitution, arithmetic and the special variables `$1`, `$#`, `$@`, `$?`.
- Conditions (`if`, `[[ ]]`, `(( ))`), loops (`for`, `while`), `case` and functions.
- How to give a script good exit statuses and error messages, how to write a small test for it, and how to debug it.

## Before you start

- You finished [Lesson 11.7 — Package managers](07-package-managers.md).
- You know pipes, redirection, exit statuses and the tools of [Lesson 11.3](03-text-tools-pipes-and-redirection.md) and [Lesson 11.5](05-processes-and-signals.md). A script is a way to save such commands in a file.
- Rust is installed in the VM (Lesson 11.5); the last script uses `cargo`.

**Where the scripts live.** Scripts are text files. Write them on your Mac, in RustRover, in the folder `~/projects/rust/code/p11-scripts` (create it first), and run them in the VM. In the VM that folder is visible, read-only, under your Mac name. Go there once and stay:

```bash
cd /Users/student/projects/rust/code/p11-scripts
```

(Replace `student` with the name that `ls /Users` shows.) Reading and running a script does not need a writable folder. Some scripts below create small files; those go in the VM home `~`, never in `/Users/...`. We run scripts as `bash NAME.sh`, which always works. Commit your scripts to Git on the Mac, as you learned in [Phase 01](../01-git-and-version-control/README.md).

## New words

| Word | Meaning |
|---|---|
| **script** | A text file with shell commands that the shell runs from top to bottom. |
| **bash** (Bourne Again SHell) | The shell of Ubuntu and the language of this lesson. |
| **shebang** | The first line `#!/usr/bin/env bash`: it tells the system which program must run the file. |
| **variable** | A named piece of text: `name="Ann"`; read it with `$name`. |
| **quoting** | Putting text in `"..."` or `'...'` so that the shell does not split or change it. |
| **word splitting** | The shell cuts an unquoted value into separate words at spaces. The source of many bugs. |
| **command substitution** | `$(command)`: replaced by the output of the command. |
| **positional parameters** | `$1`, `$2` … the arguments given to the script; `$#` is their count, `$@` is all of them. |
| **function** | A named group of commands inside a script that you can call again and again. |
| **array** | A variable that holds a list of values. |
| **exit status** | The number the script returns: 0 means success (Lesson 11.5). |

## 1. The first script

Create `hello.sh` in `code/p11-scripts` on your Mac:

```bash
#!/usr/bin/env bash
# hello.sh - the smallest useful script
name="world"
echo "Hello, $name!"
echo "This script is called $0 and was started by $USER in $PWD"
```

Run it in the VM:

```bash
bash hello.sh
```

```text
Hello, world!
This script is called hello.sh and was started by student in /home/student/p11-scripts
```

(The folder in the second line is `/Users/student/projects/rust/code/p11-scripts` on your VM.) **What happens:**

1. `#!/usr/bin/env bash` is the **shebang**. When you run the file as a program, the kernel reads these two characters `#!` and starts the named program with the file as its input. `env bash` finds `bash` in your `PATH`, which works on every system. Bash itself sees the line as a comment, like all lines that start with `#`.
2. `name="world"` stores text in a variable. **No spaces around `=`**: `name = "world"` is an error (Section 2).
3. `$name`, `$0` (the script's name as you typed it), `$USER` and `$PWD` (the current folder) are replaced by their values inside the double quotes.
4. The script's exit status is the status of its last command: here 0.

To run the file directly (`./hello.sh`, as in [Lesson 00.4](../00-getting-started/04-terminal-basics.md)), it needs the **execute permission** ([Lesson 11.4](04-users-groups-and-permissions.md)). A file you create in the VM does not have it yet:

```bash
cat hello.sh > ~/run-me.sh
~/run-me.sh
chmod +x ~/run-me.sh
~/run-me.sh
rm ~/run-me.sh
```

```text
bash: /home/student/run-me.sh: Permission denied
Hello, world!
This script is called /home/student/run-me.sh and was started by student in /home/student/p11-scripts
```

**What happens:** the first call fails with status 126 ("found, but not executable"); after `chmod +x` it works, and `$0` is now the path you typed. Put a script in a folder on your `PATH` and you can call it by name from anywhere.

## 2. Variables, quoting, expansions

```bash
name="Ann Lee"
echo "Hello, $name"
echo 'Hello, $name'
count=3
echo "count is ${count}x"
echo "sum: $((count * 4 + 1))"
echo "files: $(ls | wc -l)"
echo "${undefined:-default value}"
echo "length: ${#name}"
```

```text
Hello, Ann Lee
Hello, $name
count is 3x
sum: 13
files: 7
default value
length: 7
```

**What happens:**

| Syntax | Meaning |
|---|---|
| `"$name"` | double quotes: the variable is replaced, the text stays **one** word |
| `'$name'` | single quotes: nothing is replaced, the text is literal |
| `${count}x` | braces mark where the name ends |
| `$((expression))` | arithmetic on integers: `+ - * / %` |
| `$(command)` | command substitution: the output of the command |
| `${var:-default}` | the value of `var`, or `default` if `var` is empty or not set |
| `${#var}` | the length of the text |

The most important rule of shell scripting is: **quote your variables.** An unquoted `$name` is cut into words at spaces. Try it with a file whose name has a space (in the VM home, so it is writable):

```bash
mkdir ~/quote-lab
cd ~/quote-lab
touch "my file.txt"
f="my file.txt"
ls $f
ls "$f"
cd /Users/student/projects/rust/code/p11-scripts
rm -r ~/quote-lab
```

```text
ls: cannot access 'my': No such file or directory
ls: cannot access 'file.txt': No such file or directory
my file.txt
```

**What happens:** `ls $f` became `ls my file.txt` — two arguments. `ls "$f"` stayed one. With `rm` instead of `ls` this bug deletes the wrong files ([Lesson 11.2](02-the-linux-file-system.md)). Write `"$var"` always, unless you really want the splitting. [Lesson 11.9](09-robust-bash-scripts.md) goes further with robust scripts.

> 🔒 **Security:** Unquoted variables and text from the outside (file names, arguments, downloaded data) are the root of *command injection*, as in [Lesson 10.5](../10-security-fundamentals/05-injection-attacks.md). Quote everything, and never build a command by gluing user text into a string for `eval` or `bash -c`.

The shell sets special variables for you:

| Variable | Meaning |
|---|---|
| `$0` | the name of the script |
| `$1`, `$2` … | the first, second … argument |
| `$#` | the number of arguments |
| `$@` | all arguments, each one a separate word when written `"$@"` |
| `$?` | the exit status of the last command |
| `$$` | the PID (process ID) of the shell running the script |
| `$USER`, `$HOME`, `$PWD` | environment variables: user, home folder, current folder |

## 3. Conditions

`if` runs a command and looks at its **exit status**: 0 is true, anything else false. The command is usually a test:

```bash
[[ -f greet.sh ]] && echo "greet.sh is a file"
[[ -d /etc ]] && echo "/etc is a directory"
[[ "abc" == a* ]] && echo "glob match"
[[ 5 -gt 3 ]] && echo "5 > 3"
[[ "5" == "05" ]] || echo "strings differ"
x=7
if (( x % 2 == 0 )); then echo even; else echo odd; fi
```

```text
greet.sh is a file
/etc is a directory
glob match
5 > 3
strings differ
odd
```

(`greet.sh` exists after Section 5; create it first or the first line will not appear.) **The tests:**

| Test | True when |
|---|---|
| `-f PATH`, `-d PATH`, `-e PATH` | a regular file / a directory / anything exists |
| `-z STR`, `-n STR` | the string is empty / not empty |
| `A == B`, `A != B` | strings are equal / different (with `[[ ]]`, `B` may be a pattern like `a*`) |
| `A -eq B`, `-ne`, `-lt`, `-le`, `-gt`, `-ge` | numbers: equal, not equal, less, less or equal, greater, greater or equal |
| `! TEST` | the test is false |

`[[ ... ]]` is the bash test (also written `[ ... ]` in the older, more fragile form you met in Lesson 11.2). `(( ... ))` is for **arithmetic**: inside it you write `x % 2 == 0`, `n > 0`, without `$`. The full form:

```bash
if [[ -f "$file" ]]; then
    echo "found"
elif [[ -d "$file" ]]; then
    echo "directory"
else
    echo "missing"
fi
```

`cmd1 && cmd2` runs `cmd2` only if `cmd1` succeeded, `cmd1 || cmd2` only if it failed ([Lesson 11.2](02-the-linux-file-system.md)). The spaces inside `[[ ]]` are mandatory: `[[-f x]]` is an error.

## 4. Loops, `case`, functions, arrays

```bash
for i in 1 2 3; do echo "i=$i"; done
for f in *.sh; do echo "script: $f"; done
for ((i = 0; i < 3; i++)); do echo -n "$i "; done; echo
n=3; while (( n > 0 )); do echo "countdown $n"; n=$((n - 1)); done
```

```text
i=1
i=2
i=3
script: check-project.sh
script: greet.sh
script: hello.sh
script: log-report.sh
script: test-greet.sh
0 1 2 
countdown 3
countdown 2
countdown 1
```

(The list of scripts is what your folder contains at the end of the lesson.) **What happens:**

1. `for x in LIST; do ...; done` repeats for each word. The list may be fixed words, a **glob** like `*.sh` (the shell expands it to the matching file names, even names with spaces stay one word), or `"$@"`.
2. `for ((i = 0; i < 3; i++))` is the counting form.
3. `while CONDITION; do ...; done` repeats as long as the condition succeeds. To walk through the lines of a file, use `while read -r line; do ...; done < file`: `read` takes one line per round from stdin (`-r` stops it from treating backslashes specially).

`case` compares one value with several patterns:

```bash
for ext in rs toml md; do
  case "$ext" in
    rs) echo "$ext: Rust source" ;;
    toml|yaml) echo "$ext: config" ;;
    *) echo "$ext: something else" ;;
  esac
done
```

```text
rs: Rust source
toml: config
md: something else
```

**Functions** group commands. A function receives arguments as `$1`, `$2`; it "returns" text through `echo` (capture it with `$(...)`) and a status through `return`:

```bash
add() { local a="$1" b="$2"; echo $((a + b)); }
result=$(add 2 3)
echo "2+3=$result"
is_even() { (( $1 % 2 == 0 )); }
is_even 4 && echo "4 is even"
is_even 5 || echo "5 is odd"
```

```text
2+3=5
4 is even
5 is odd
```

**What happens:** `local` keeps variables inside the function (without it they are global). A function's status is the status of its last command, so `is_even` can be used in `&&` / `||` and `if` like any command. **Arrays** hold lists:

```bash
files=(a.txt b.txt "c d.txt")
echo "${#files[@]} items; second is ${files[1]}"
for f in "${files[@]}"; do echo "[$f]"; done
```

```text
3 items; second is b.txt
[a.txt]
[b.txt]
[c d.txt]
```

`"${files[@]}"` expands to every element as a separate word, spaces kept. Indexes start at 0.

## 5. Real scripts with good manners

A script that others (and cron jobs, CI and systemd units) call must behave like the programs you have met: **arguments in, results on stdout, errors on stderr, an exit status that tells the truth.**

Create `greet.sh`:

```bash
#!/usr/bin/env bash
# greet.sh [NAME]... - greet every name given, or the world
if [[ $# -eq 0 ]]; then
    echo "Hello, world!"
    exit 0
fi

for name in "$@"; do
    echo "Hello, $name!"
done
echo "(greeted $# name(s))"
```

```bash
bash greet.sh
bash greet.sh Ann Bob
bash greet.sh "Ann Lee"
```

```text
Hello, world!
Hello, Ann!
Hello, Bob!
(greeted 2 name(s))
Hello, Ann Lee!
(greeted 1 name(s))
```

**What happens:** `$#` is the number of arguments; with none, the script greets the world and ends with `exit 0`. `"$@"` keeps each argument as one word, so `"Ann Lee"` is one name.

Now a script that is useful with the log file from Lesson 11.3. Create `access.log` in the VM home (the 12 lines from that lesson, if you deleted them) and write `log-report.sh`:

```bash
#!/usr/bin/env bash
# log-report.sh FILE - count requests per status code in a web server log
# Each line of FILE looks like: 203.0.113.5 GET /index.html 200 512

usage() {
    echo "usage: $0 LOGFILE" >&2
}

if [[ $# -ne 1 ]]; then
    usage
    exit 2
fi

log="$1"
if [[ ! -f "$log" ]]; then
    echo "error: '$log' is not a file" >&2
    exit 1
fi

total=$(wc -l < "$log")
echo "total requests: $total"

for code in $(cut -d' ' -f4 "$log" | sort -u); do
    count=$(grep -c " $code " "$log")
    printf '%s: %s\n' "$code" "$count"
done

errors=$(awk '$4 >= 400' "$log" | wc -l)
if (( errors * 100 / total >= 30 )); then
    echo "WARNING: $errors of $total requests failed"
    exit 3
fi
echo "ok: $errors of $total requests failed"
```

```bash
cd ~
bash /Users/student/projects/rust/code/p11-scripts/log-report.sh access.log
echo "exit=$?"
```

```text
total requests: 12
200: 5
201: 1
204: 1
304: 1
403: 1
404: 2
500: 1
WARNING: 4 of 12 requests failed
exit=3
```

**What happens:**

1. `usage` is a **function**; it writes to **stderr** (`>&2`), because it is a message for a human, not a result. The script ends with status 2 for wrong usage, 1 for a bad file, 3 for "too many failures", 0 for success. Document your codes in a comment.
2. `for code in $(...)` loops over the status codes that appear in the file (each is a plain number, so the unquoted substitution is fine here).
3. `total=$(wc -l < "$log")` captures a number; `(( errors * 100 / total >= 30 ))` computes the percentage of failures (integer arithmetic) and compares it with 30.
4. The same pieces from Lesson 11.3 (`cut`, `sort`, `grep -c`, `awk`) are now one tool you can call again.

Try the failures:

```bash
bash /Users/student/projects/rust/code/p11-scripts/log-report.sh; echo "exit=$?"
bash /Users/student/projects/rust/code/p11-scripts/log-report.sh nofile.log; echo "exit=$?"
```

```text
usage: /Users/student/projects/rust/code/p11-scripts/log-report.sh LOGFILE
exit=2
error: 'nofile.log' is not a file
exit=1
```

(`$0` is the path you typed, so the usage line shows it.) A third script runs the Rust quality gate (`fmt`, `clippy`, `test` from Phase 05) in every folder you give it. Create `check-project.sh`:

```bash
#!/usr/bin/env bash
# check-project.sh DIR... - run the Rust quality gate in each project folder

failed=0

for dir in "$@"; do
    echo "== $dir"
    if [[ ! -f "$dir/Cargo.toml" ]]; then
        echo "   skipped: no Cargo.toml"
        continue
    fi
    if (cd "$dir" && cargo fmt --check && cargo clippy -q -- -D warnings && cargo test -q); then
        echo "   PASS"
    else
        echo "   FAIL"
        failed=$((failed + 1))
    fi
done

echo "failed projects: $failed"
if [[ $failed -gt 0 ]]; then
    exit 1
fi
```

**What happens:** the parentheses `( cd "$dir" && ... )` run the commands in a **subshell**, so the `cd` does not change the folder of the script itself. `if (command...)` uses the exit status of the whole chain: it is true only if all three cargo commands succeed. `-D warnings` makes Clippy treat warnings as errors. The counter `failed` becomes the exit status: the script itself can be used in `&&` chains and in CI ([Lesson 09.12](../09-testing-debugging-benchmarking/12-continuous-integration.md)). To try it, make a good project, a project whose test fails, and a folder without `Cargo.toml` in the VM home, and run:

```bash
cd /Users/student/projects/rust/code/p11-scripts
bash check-project.sh ~/proj-lab/good ~/proj-lab/bad ~/proj-lab/notes 2>&1 | grep -E '^(==|   (PASS|FAIL|skipped)|failed)'
```

```text
== /home/student/proj-lab/good
   PASS
== /home/student/proj-lab/bad
   FAIL
== /home/student/proj-lab/notes
   skipped: no Cargo.toml
failed projects: 1
```

(`grep` only keeps the summary lines of the output. To try the script, create a project `good` with `cargo new --vcs none good`, a project `bad` with a test that does `assert_eq!(1 + 1, 3)`, and a folder `notes` without `Cargo.toml`, all in `~/proj-lab`.)

## 6. Test and debug a script

A script is code: it needs tests. A shell test runs the script and compares what it prints (and its exit status) with what you expect. Create `test-greet.sh`:

```bash
#!/usr/bin/env bash
# test-greet.sh - a tiny test for greet.sh
cd "$(dirname "$0")" || exit 1

check() {
    local description="$1" expected="$2" actual="$3"
    if [[ "$actual" == "$expected" ]]; then
        echo "PASS: $description"
    else
        echo "FAIL: $description"
        echo "  expected: $expected"
        echo "  actual:   $actual"
        failures=$((failures + 1))
    fi
}

failures=0
check "one name" $'Hello, Ann!\n(greeted 1 name(s))' "$(bash greet.sh Ann)"
check "no name" "Hello, world!" "$(bash greet.sh)"
check "two names" $'Hello, Ann!\nHello, Bob!\n(greeted 2 name(s))' "$(bash greet.sh Ann Bob)"

if (( failures > 0 )); then
    exit 1
fi
```

```bash
bash test-greet.sh; echo "exit=$?"
```

```text
PASS: one name
PASS: no name
PASS: two names
exit=0
```

**What happens:** `$(dirname "$0")` is the folder of the test script, so it works from any directory. `check` compares the real output (`$(...)` captures stdout) with the expected text; `$'...\n...'` is the bash way to write a newline inside a string. The test exits with status 1 when something differs, so a build server can notice. A test is only trustworthy if you have seen it fail once: change `Hello,` to `Hi,` in a *copy* of `greet.sh` and the matching line prints `FAIL`.

For finding bugs, bash has two helpers:

```bash
bash -n greet.sh; echo "syntax ok? $?"
bash -x greet.sh Ann
```

```text
syntax ok? 0
+ [[ 1 -eq 0 ]]
+ for name in "$@"
+ echo 'Hello, Ann!'
Hello, Ann!
+ echo '(greeted 1 name(s))'
(greeted 1 name(s))
```

**What happens:** `bash -n` only checks the syntax, nothing runs. `bash -x` ("trace") prints each command with its variables expanded, after a `+`, before it runs: the quickest way to see what a script really does. You can also write `set -x` inside the script around a suspicious part. These lines go to stderr.

## Common mistakes

1. **Spaces around `=`.** `name = "Ann"` → `name: command not found`. *Fix:* `name="Ann"`.
2. **Unquoted variables.** `rm $file` with a space in the name deletes the wrong things. *Fix:* `rm "$file"`.
3. **Missing spaces in tests.** `[[$x -eq 1]]` → `command not found`. *Fix:* spaces inside the brackets: `[[ $x -eq 1 ]]`.
4. **`=` instead of `-eq` for numbers, `-eq` instead of `==` for text.** `[[ "5" == "05" ]]` is false, `[[ 5 -eq 05 ]]` is true. *Fix:* use `-eq`, `-lt`, … for numbers and `==` for text.
5. **Dividing by zero in `(( ))`.** `(( errors * 100 / total ))` with `total=0` prints `division by 0` but the script **goes on** (see Exercise 2). *Fix:* test for zero first.
6. **Reading `$?` too late.** Another command in between overwrites it. *Fix:* `status=$?` at once.
7. **Forgetting that scripts run in their own process.** A `cd` or `export` in a script does not change your terminal. *Fix:* that is by design; use `source script.sh` only for scripts that set up your shell.
8. **Parsing the output of `ls`.** Names with spaces break it. *Fix:* use a glob (`for f in *.sh`) or `find ... -print0`.

## Try it yourself

1. Create `hello.sh`, run it with `bash hello.sh`, then run it directly after making a copy in the VM home with `chmod +x`.
2. Write `greet.sh` and run it with no arguments, one, and a name with a space.
3. Write `log-report.sh` and run it on `access.log` and on a file with only the first 6 lines (`head -n 6 access.log > ~/small.log`). Expected exit status: 3 for the full file, 0 for the small one (`ok: 1 of 6 requests failed`).
4. Add `set -x` as the second line of a copy of `greet.sh` and read the trace. Then remove the copy.
5. Write `test-greet.sh` and run it. Break `greet.sh` in a copy and watch the test fail.

## Exercises

### Exercise 1 — Number the arguments (easy)

Write `args.sh ARG...` that prints `got N argument(s)` and then each argument on its own line, with a number: `1: first`. An argument with a space must stay one argument.

<details>
<summary>Hint</summary>

`$#` is the count. Loop over `"$@"` (with the quotes) and keep a counter with `n=$((n + 1))`.

</details>

<details>
<summary>Answer</summary>

```bash
#!/usr/bin/env bash
# args.sh ARG... - number the arguments
echo "got $# argument(s)"
n=1
for arg in "$@"; do
    echo "$n: $arg"
    n=$((n + 1))
done
```

```bash
bash args.sh a "b c" d
bash args.sh
```

```text
got 3 argument(s)
1: a
2: b c
3: d
got 0 argument(s)
```

Why it works: `"$@"` keeps `b c` as one word, so the loop runs three times; with no arguments the loop body does not run at all.

</details>

### Exercise 2 — Fix a real bug (medium)

Run `log-report.sh` on an **empty** file (`: > empty.log`). What does it print, and what is wrong? Fix the script so that an empty log is reported as `ok: no requests` with status 0.

<details>
<summary>Hint</summary>

Look at the line with `(( errors * 100 / total >= 30 ))` and think of `total=0`. Check `total` before the division.

</details>

<details>
<summary>Answer</summary>

Before the fix:

```text
total requests: 0
./log-report.sh: line 29: ((: errors * 100 / total >= 30 : division by 0 (error token is "total >= 30 ")
ok: 0 of 0 requests failed
exit=0
```

The script prints an error but still says `ok` and exits with 0: the failed `(( ))` counts as "false", so the `if` is skipped. Add the guard right after `echo "total requests: $total"`:

```bash
if (( total == 0 )); then
    echo "ok: no requests"
    exit 0
fi
```

```text
total requests: 0
ok: no requests
exit=0
```

Why it works: the guard handles the special case before the division. The lesson: an error message in a script does not stop it, unless you stop it yourself. The next lesson, on robust scripts, shows how to make a script stop at the first failure.

</details>

### Exercise 3 — Write a test for the exit statuses (hard)

Write `test-log-report.sh` that checks four exit statuses of `log-report.sh`: no argument (2), a missing file (1), a log where every request failed (3) and a log where none failed (0). The test creates its own small log files in a temporary folder and removes them at the end. It prints `PASS`/`FAIL` for each case and exits 0 only if all pass.

<details>
<summary>Hint</summary>

`mktemp -d` creates a new empty temporary folder and prints its name. Write a function `expect_status EXPECTED LABEL ARGS...`: run the script with `> /dev/null 2>&1` to hide its output, save `$?` at once into a variable, and compare. Use `shift 2` to remove the first two arguments before you pass the rest on.

</details>

<details>
<summary>Answer</summary>

```bash
#!/usr/bin/env bash
# test-log-report.sh - check the exit status of log-report.sh
cd "$(dirname "$0")" || exit 1

data=$(mktemp -d)
printf '203.0.113.5 GET /a 200 1\n203.0.113.5 GET /a 200 1\n' > "$data/ok.log"
printf '203.0.113.5 GET /a 500 0\n203.0.113.5 GET /a 500 0\n' > "$data/bad.log"

failures=0

# expect_status EXPECTED LABEL [ARGS...]
expect_status() {
    local expected="$1" label="$2"
    shift 2
    bash log-report.sh "$@" > /dev/null 2>&1
    local actual=$?
    if [[ $actual -eq $expected ]]; then
        echo "PASS: $label -> $actual"
    else
        echo "FAIL: $label -> $actual (expected $expected)"
        failures=$((failures + 1))
    fi
}

expect_status 2 "no argument"
expect_status 1 "missing file" "$data/missing.log"
expect_status 3 "all requests failed" "$data/bad.log"
expect_status 0 "no request failed" "$data/ok.log"

rm -r "$data"
[[ $failures -eq 0 ]]
```

```text
PASS: no argument -> 2
PASS: missing file -> 1
PASS: all requests failed -> 3
PASS: no request failed -> 0
```

To see the test fail, change `exit 3` to `exit 4` in a **copy** of `log-report.sh`: the third line becomes `FAIL: all requests failed -> 4 (expected 3)` and the exit status of the test is 1.

Why it works: `shift 2` drops the first two arguments, `"$@"` passes the rest, and `local actual=$?` reads the status immediately. The very last line is the exit status of the test: 0 only if no case failed. This is the same idea as a Rust `assert_eq!`, written in shell. The test needs no files from you, so it also runs on the read-only folder.

</details>

## Quiz

1. What does `#!/usr/bin/env bash` on the first line do?

<details>
<summary>Answer</summary>

When the file is run directly, the kernel starts the named program (`bash`, found via `env` in `PATH`) and gives it the file. Bash itself treats the line as a comment.

</details>

2. What is the difference between `"$@"` and `$@`?

<details>
<summary>Answer</summary>

`"$@"` keeps every argument as one word (even with spaces). Unquoted `$@` splits the arguments again at spaces.

</details>

3. When is the condition of an `if` "true"?

<details>
<summary>Answer</summary>

When the command in it ends with exit status 0.

</details>

4. Where should error messages of a script go, and why?

<details>
<summary>Answer</summary>

To stderr (`>&2`), so that stdout contains only results that other programs can use in pipelines.

</details>

5. What do `bash -n` and `bash -x` do?

<details>
<summary>Answer</summary>

`-n` checks the syntax without running; `-x` prints every command, with variables expanded, before it runs.

</details>

## Summary

- A script is a text file of shell commands; the shebang `#!/usr/bin/env bash` names the interpreter; run it with `bash NAME.sh` or after `chmod +x` as `./NAME.sh`.
- Variables: `name="value"` (no spaces), `"$name"` (quote!), `$(command)`, `$((arithmetic))`, `${var:-default}`.
- Special variables: `$0`, `$1…`, `$#`, `"$@"`, `$?`, `$$`.
- Conditions use exit statuses: `if`, `[[ ]]` for files, text and numbers, `(( ))` for arithmetic, `&&` and `||`.
- Loops `for` and `while`, `case`, functions with `local`, arrays.
- A good script: arguments in, results to stdout, errors to stderr, honest exit statuses, a usage message.
- Test scripts with a small test script; debug with `bash -n` and `bash -x`.
- Next: [Lesson 11.9 — Robust bash scripts](09-robust-bash-scripts.md) continues with scripts that you can trust in production.

## Sources and further reading

- [bash(1) manual page](https://man7.org/linux/man-pages/man1/bash.1.html) — the complete reference of the language and its exit statuses.
- [GNU Bash Manual — Shell Expansions](https://www.gnu.org/software/bash/manual/html_node/Shell-Expansions.html) — quoting, parameter expansion, command substitution, arithmetic.
- [GNU Bash Manual — Conditional Constructs](https://www.gnu.org/software/bash/manual/html_node/Conditional-Constructs.html) — `if`, `case`, `[[ ]]`, `(( ))`.
- [GNU Bash Manual — Bash Conditional Expressions](https://www.gnu.org/software/bash/manual/html_node/Bash-Conditional-Expressions.html) — the list of file and string tests.
- [ShellCheck](https://www.shellcheck.net/) — a tool that finds common mistakes in shell scripts.

---

[🏠 Start](../START-HERE.md) · [← 11.7 Package managers](07-package-managers.md) · [📋 Phase 11](README.md) · [11.9 Robust bash scripts →](09-robust-bash-scripts.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
