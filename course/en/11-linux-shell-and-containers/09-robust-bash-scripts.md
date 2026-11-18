<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 11.8 Bash scripting basics](08-bash-scripting-basics.md) · [📋 Phase 11](README.md) · [11.10 SSH and remote servers →](10-ssh-and-remote-servers.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/11-linux-shell-and-containers/09-robust-bash-scripts.md) <!-- nav -->

# Lesson 11.9 — Robust bash scripts

> **Phase 11 — Linux, the shell and containers** · Lesson 9 of 16 · ⏱ about 2.5 hours (2 sittings)

## What you will learn

- Why a plain bash script keeps running after an error, and how "strict mode" (`set -euo pipefail`) stops it.
- The places where strict mode does not help, so that it does not surprise you.
- How to quote variables correctly, and why one missing pair of quotes can delete the wrong files.
- How to clean up after yourself with `mktemp` and `trap`.
- How to write a complete script with options, a help text and safe defaults, how to check it with ShellCheck, and how to test it.

## Before you start

- You finished [Lesson 11.8 — Bash scripting basics](08-bash-scripting-basics.md). You can write a script with a first line `#!/usr/bin/env bash`, variables, `if`, `for`, functions and arguments (`$1`, `$#`), and you know that every command ends with an exit status ([Lesson 11.5](05-processes-and-signals.md)).
- You know pipes and redirection ([Lesson 11.3](03-text-tools-pipes-and-redirection.md)) and permissions ([Lesson 11.4](04-users-groups-and-permissions.md)).
- Docker Desktop is optional here. It is only used in Section 7 to run ShellCheck without installing it.

**Where the files go.** Write the scripts on your Mac in `~/projects/rust/code/p11-scripts` (RustRover or any editor), then copy them into the VM and run them there. The Mac folder is read-only inside the VM ([Lesson 11.1](01-linux-and-a-virtual-machine.md)), so copy first (replace `student` by your Mac user name; `ls /Users` shows it):

```bash
mkdir -p ~/scripts
cp /Users/student/projects/rust/code/p11-scripts/*.sh ~/scripts/
cd ~/scripts
```

Repeat the `cp` after every edit. Every command of this lesson runs inside the VM.

## New words

| Word | Meaning |
|---|---|
| **strict mode** | The three settings `set -e`, `set -u` and `set -o pipefail`, which make a script stop at the first error instead of going on. |
| **shell option** | A switch of the shell itself, turned on with `set -o name` (or a short flag such as `set -e`) and off with `set +o name` (`set +e`). |
| **unbound variable** | A variable that was never given a value. Bash normally treats it as empty text; with `set -u` it is an error. |
| **word splitting** | Bash cuts the value of an unquoted variable into separate words at spaces and tabs. `name="my notes.txt"` becomes two words. |
| **glob** (globbing) | A pattern such as `*.txt` that bash expands into the list of matching file names. |
| **quoting** | Putting a value in quotes (`"$name"`) so that bash keeps it as **one** word and does not glob it. |
| **`trap`** | A shell command that says: run this code when a certain event happens, for example when the script ends or receives a signal. |
| **temporary file** | A file that exists only while a program works. `mktemp` creates one with a unique name that nobody else can guess. |
| **atomic** | An action that is either completely done or not done at all, never half done. Renaming a file is atomic. |
| **array** | A numbered list of values in one variable: `files=(a b c)`. `${files[0]}` is the first value, `${#files[@]}` is the count. |
| **option** | An argument that starts with a dash and changes how a command works: `-k 3`, `-n`. |
| **usage message** | The short help text a script prints when you call it wrongly or with `-h`. |
| **dry run** | A mode in which a program says what it **would** do and changes nothing. |
| **ShellCheck** | A free program that reads a shell script and warns about typical mistakes. Programs of this kind are called *linters*. |
| **linter** | A tool that reads source code, without running it, and reports mistakes and bad style. `cargo clippy` is the linter of Rust. |

## 1. A script that does not stop

Bash was made for typing commands by hand. If one command fails, you read the message and decide. A script has nobody to decide, but bash behaves the same way: **it goes on with the next line**. Create `fragile.sh`:

```bash
#!/usr/bin/env bash
# A fragile script: it keeps going after errors.
cd /no/such/folder
echo "now I am in: $(pwd)"
echo "cleaning up"
```

Run it and look at the exit status:

```bash
bash fragile.sh
echo "exit status: $?"
```

```text
fragile.sh: line 3: cd: /no/such/folder: No such file or directory
now I am in: /home/student/scripts
cleaning up
exit status: 0
```

**What happens:**

1. `cd` fails and prints an error, but bash does not care: it executes line 4.
2. The script continues in the **wrong folder**. If the next line had been `rm -rf *`, it would have deleted the files of whatever folder you started in.
3. The last command (`echo`) succeeded, so the script ends with **status 0**: the caller is told "everything is fine". A deploy tool, a service manager or a CI (continuous integration) job (Lesson 09.12) would believe it.

A robust script must (a) stop when something goes wrong and (b) tell the truth about it in its exit status. The rest of the lesson is about how.

## 2. Strict mode

Put this line right after the first line of every script:

```bash
set -euo pipefail
```

It is three switches written together. Test each of them.

**`-e` (exit on error).** The script exits when a simple command fails (returns a status other than 0). Copy `fragile.sh` to `strict.sh` and add the line:

```bash
#!/usr/bin/env bash
set -euo pipefail
cd /no/such/folder
echo "now I am in: $(pwd)"
```

```bash
bash strict.sh
echo "exit status: $?"
```

```text
strict.sh: line 3: cd: /no/such/folder: No such file or directory
exit status: 1
```

The script stopped at the failing line and passed the failure on (status 1).

**`-u` (unset is an error).** In `unset.sh`:

```bash
#!/usr/bin/env bash
set -u
echo "target is: $TARGET"
echo "not reached"
```

```bash
bash unset.sh
echo "exit status: $?"
```

```text
unset.sh: line 3: TARGET: unbound variable
exit status: 1
```

Without `-u`, `$TARGET` would silently be empty. That is how this famous accident happens. Look at the typo (`targt`) in `danger.sh`:

```bash
#!/usr/bin/env bash
# DANGER demo: we only echo the command, we never run it.
target=/tmp/lab
echo rm -rf "$targt"/
```

```text
rm -rf /
```

The misspelled variable is empty, so "delete the folder `$target/`" became "delete everything". With `set -u` at the top (`danger2.sh`) the same typo stops the script with `danger2.sh: line 4: targt: unbound variable` and nothing is deleted. (These scripts only print the command. Never run such a command for real.)

> 🔒 **Security:** Scripts often run as root (from `sudo`, from systemd, from a deploy tool). A script that deletes, moves or changes permissions must never work with a path that might be empty. Besides `set -u`, you can write `rm -rf -- "${target:?target is not set}/"`. The form `${name:?message}` stops the script with the message when the variable is unset **or empty**.

**`-o pipefail`.** A pipeline (`a | b | c`, [Lesson 11.3](03-text-tools-pipes-and-redirection.md)) normally has the status of its **last** command only. In `pipefail.sh`:

```bash
#!/usr/bin/env bash
false | cat
echo "without pipefail the pipeline status was: $?"
set -o pipefail
false | cat
echo "with pipefail the pipeline status was: $?"
```

```text
without pipefail the pipeline status was: 0
with pipefail the pipeline status was: 1
```

`false` always fails. `cat` succeeds, so the pipeline looked fine. With `pipefail` the pipeline fails if **any** command in it fails. Think of `curl https://example.org/file | tar -xz`: if `curl` cannot download, you want to know.

> 💡 **Tip:** Strict mode is a safety net, not a replacement for thinking. The short version to remember: `-e` stop on error, `-u` stop on typos, `pipefail` do not hide a failure inside a pipe.

## 3. Where `set -e` does not help

`set -e` has exceptions. Knowing them saves hours. Run `esurprise.sh`, which contains three cases (the comment above each is the rule):

```bash
#!/usr/bin/env bash
set -e
# 1. grep returns 1 when nothing matches. That is not a crash!
echo "hello" | grep -q xyz || echo "1: grep found nothing, fine"

# 2. Arithmetic: the value 0 counts as 'false' and gives status 1.
count=0
((count++)) || echo "2: ((count++)) returned status 1 when count was 0"
echo "count is now $count"

# 3. 'local x=$(cmd)' hides the status of cmd.
f() { local out=$(false); echo "3: local hid the failure, out=[$out]"; }
f
g() { local out; out=$(false); echo "not reached"; }
g
echo "also not reached"
```

```bash
bash esurprise.sh
echo "exit status: $?"
```

```text
1: grep found nothing, fine
2: ((count++)) returned status 1 when count was 0
count is now 1
3: local hid the failure, out=[]
exit status: 1
```

**What happens:**

1. `set -e` also does not stop a command used as the **condition** of `if` or on the left of `||` and `&&`: you are already handling the failure. That is why `grep ... || echo` above is safe. `grep` uses status 1 for "no match", which is information, not an error.
2. `((count++))` evaluates to the **old value** of `count`, which is 0, and arithmetic treats 0 as false, so the status is 1. Under `set -e` this line alone would stop the script. Write `count=$((count + 1))` instead.
3. `local out=$(false)` has two commands: `local` (which succeeds) and the substitution. The status of `local` wins and the failure disappears. The fix is to declare and assign in **two lines** (`local out; out=$(false)`), as the function `g` does: there the script stops.

> ⚠️ **Common mistake:** Believing that `set -e` makes a script safe. It does not catch every error. Use it together with explicit checks for things you can predict ("does this folder exist?") and with ShellCheck (Section 7).

## 4. Quoting

Bash processes a command line in steps. Two of them surprise beginners: **word splitting** (an unquoted `$name` is cut at spaces) and **globbing** (a `*` in an unquoted value is replaced by file names). The cure is simple: **double-quote every variable and every `$(...)`** unless you really want splitting. Run `quoting.sh`:

```bash
#!/usr/bin/env bash
set -euo pipefail
dir=$(mktemp -d)
cd "$dir"
touch "my notes.txt"
echo "--- unquoted"
for f in $(ls); do echo "item: [$f]"; done
echo "--- glob and quotes"
for f in *; do echo "item: [$f]"; done
echo "--- unquoted variable in a command"
name="my notes.txt"
rm $name || echo "rm failed"
rm "$name"
find . -type f | wc -l
cd /
rmdir "$dir"
```

```text
--- unquoted
item: [my]
item: [notes.txt]
--- glob and quotes
item: [my notes.txt]
--- unquoted variable in a command
rm: cannot remove 'my': No such file or directory
rm: cannot remove 'notes.txt': No such file or directory
rm failed
0
```

**What happens:**

1. `mktemp -d` creates a new empty folder with a random name under `/tmp` (more in Section 5). The script works there, so it cannot hurt your files.
2. `$(ls)` returns the text `my notes.txt`. Unquoted, it is split into two words, and the loop runs twice with names that do not exist. Never loop over the output of `ls`. A glob (`*`) gives each file name as **one** word, spaces included.
3. `rm $name` became `rm my notes.txt`: two wrong names. `rm "$name"` is right.

The rules, in order of importance:

| Rule | Example |
|---|---|
| Quote every variable | `"$file"`, `"${dir}/out"` |
| Quote every command substitution | `"$(date +%F)"` |
| Pass all arguments on with `"$@"` | `cp -- "$@" /backup` |
| End the options with `--` before names | `rm -- "$file"` |
| Use `[[ ... ]]` for tests | `[[ -d $dir ]]` (no quotes needed inside `[[ ]]`) |
| Use single quotes for text that must stay literal | `'$HOME is not expanded here'` |

Two of these need an example. `"$@"` keeps each argument as one word. With the arguments `"a b"` and `c`, `"$@"` gives two words (`a b` and `c`), while an unquoted `$@` or `$*` gives three (`a`, `b`, `c`) and `"$*"` glues everything into one. `--` protects against file names that start with a dash:

```bash
touch ./-rf
rm -- -rf && echo "removed a file called -rf"
```

```text
removed a file called -rf
```

Without `--`, `rm -rf` would be read as the options `-r` and `-f`. Anyone who can create files in a folder (an upload, an unpacked archive) can name them like options (the idea of Lesson 10.5).

## 5. Clean up with `mktemp` and `trap`

A script often needs a scratch file or folder. Do not invent a name like `/tmp/work`: another run or another user may use the same name, and an attacker can prepare it in advance. `mktemp` creates a file (or, with `-d`, a folder) with a **random** name and safe permissions (only you can read it) and prints the name.

When the script ends, the scratch files must go, **also when it ends with an error**. `trap` does that. Run `trap.sh`:

```bash
#!/usr/bin/env bash
set -euo pipefail
tmp=$(mktemp -d)
cleanup() {
    echo "cleanup: removing $tmp"
    rm -rf "$tmp"
}
trap cleanup EXIT
echo "working in $tmp"
echo data > "$tmp/work.txt"
false   # an error: the script stops here
echo "not reached"
```

```bash
bash trap.sh
echo "exit status: $?"
```

```text
working in /tmp/tmp.LL1HIzxbjs
cleanup: removing /tmp/tmp.LL1HIzxbjs
exit status: 1
```

**What happens:**

1. `trap cleanup EXIT` registers the function `cleanup`. `EXIT` is a special event: it fires **whenever the script ends**, on normal end, on `exit`, and after an error caught by `set -e`.
2. `false` fails, `set -e` ends the script, and the trap runs before bash returns. The exit status is still 1: the trap does not hide the failure.
3. The folder name is random; yours will differ.

What if somebody presses `Ctrl + C` or a service manager sends `SIGTERM` ([Lesson 11.5](05-processes-and-signals.md))? Name the signals in a second trap, and exit with the conventional status 128 + signal number (`SIGINT` is 2, so 130). The `EXIT` trap still runs afterwards:

```bash
#!/usr/bin/env bash
set -euo pipefail
tmp=$(mktemp -d)
trap 'echo "cleanup $tmp"; rm -rf "$tmp"' EXIT
trap 'echo "interrupted"; exit 130' INT TERM
echo "sleeping, pid $$"
sleep 30 &
wait $!
```

Start it in the background, send `SIGTERM` with `kill -TERM $!`, and read the status with `wait $!` and `echo $?`. The result:

```text
sleeping, pid 14
interrupted
cleanup /tmp/tmp.s0MXYBEICj
exit status: 130
```

**What happens:** `$$` is the process number of the script. The script waits for `sleep` with `wait`, because bash runs a trap only between commands, not in the middle of a long foreground command; `wait` can be interrupted. (The `sleep 30` itself keeps running after the script has gone. In a real script you would also stop child processes you started.)

## 6. A complete script: `backup.sh`

Now we put the habits together in one tool: it makes a compressed backup of a folder and keeps only the newest few. It uses **options**, a **usage message**, **validation**, a **dry run**, a **temporary file** and an **atomic** rename. Save it as `code/p11-scripts/backup.sh` on the Mac (and make it executable there with `chmod +x`):

```bash
#!/usr/bin/env bash
# backup.sh - make a compressed backup of a directory and keep only the newest few.
set -euo pipefail

prog=${0##*/}
keep=3
dry_run=0

usage() {
    cat <<USAGE
Usage: $prog [-k COUNT] [-n] SOURCE_DIR DEST_DIR

Writes DEST_DIR/NAME-YYYYmmdd-HHMMSS.tar.gz, where NAME is the name of SOURCE_DIR,
then deletes the oldest backups so that only COUNT remain.

  -k COUNT  how many backups to keep (default: $keep)
  -n        dry run: print what would happen, change nothing
  -h        show this help
USAGE
}

log() { printf '%s: %s\n' "$prog" "$*" >&2; }
die() { log "error: $*"; exit 2; }

while getopts ':k:nh' opt; do
    case $opt in
        k) keep=$OPTARG ;;
        n) dry_run=1 ;;
        h) usage; exit 0 ;;
        :) die "option -$OPTARG needs a value" ;;
        \?) die "unknown option -$OPTARG" ;;
    esac
done
shift $((OPTIND - 1))

[[ $# -eq 2 ]] || { usage >&2; exit 2; }
[[ $keep =~ ^[1-9][0-9]*$ ]] || die "COUNT must be a positive number, got '$keep'"
source_dir=$1
dest_dir=$2
[[ -d $source_dir ]] || die "not a directory: $source_dir"

name=$(basename -- "$(realpath -- "$source_dir")")
stamp=${BACKUP_STAMP:-$(date +%Y%m%d-%H%M%S)}
target=$dest_dir/$name-$stamp.tar.gz

if (( dry_run )); then
    log "dry run: would create $target"
else
    mkdir -p -- "$dest_dir"
    tmp=$(mktemp -- "$dest_dir/.$name.XXXXXX")
    trap 'rm -f -- "$tmp"' EXIT
    tar -czf "$tmp" -C "$source_dir" .
    mv -- "$tmp" "$target"
    log "created $target"
fi

shopt -s nullglob
backups=("$dest_dir/$name"-*.tar.gz)
count=${#backups[@]}
if (( dry_run )); then
    count=$(( count + 1 ))   # the backup that a real run would have created
fi
excess=$(( count - keep ))
for (( i = 0; i < excess; i++ )); do
    if (( dry_run )); then
        log "dry run: would delete ${backups[i]}"
    else
        rm -- "${backups[i]}"
        log "deleted ${backups[i]}"
    fi
done
```

**What happens:**

1. `prog=${0##*/}` takes `$0` (the script's name as typed) and cuts everything up to the last `/`, so messages say `backup.sh`. `${name##pattern}` removes the longest matching beginning.
2. `usage` prints a here-document (`<<USAGE ... USAGE`): the lines up to the word `USAGE` are the text. `log` and `die` print to **standard error** (`>&2`, [Lesson 11.3](03-text-tools-pipes-and-redirection.md)), so messages never mix with real output. `die` exits with status 2, the usual "wrong usage" status.
3. `getopts ':k:nh' opt` is bash's built-in option parser. `k:` means "option `-k` takes a value" (it arrives in `$OPTARG`); `n` and `h` are plain switches; the leading `:` lets us report errors ourselves. The `case` handles each letter.
4. `shift $((OPTIND - 1))` removes the processed options, so `$1` and `$2` are the two remaining arguments. We insist on exactly two.
5. `[[ $keep =~ ^[1-9][0-9]*$ ]]` checks with a regular expression that `-k` is a positive whole number. **Check input before you use it** (Lesson 10.4).
6. The new archive is written to a `mktemp` file in the **same folder**, and only when `tar` has succeeded does `mv` rename it. A rename inside one folder is atomic, so nobody ever sees a half-written `.tar.gz`; and if `tar` fails, the `EXIT` trap removes the temporary file.
7. `shopt -s nullglob` makes a pattern without matches expand to **nothing** (not to the pattern text). `backups=(...)` stores the matching names in an **array**. The names contain a timestamp in the form year-month-day-time, so sorting them alphabetically also sorts them by age: the oldest are first.
8. `excess` is how many are too many. The loop deletes `backups[0]`, `backups[1]`, ... (the oldest). In a dry run the new file does not exist yet, so we add 1 to the count.
9. `BACKUP_STAMP` lets a test fix the timestamp; normally `date` provides it.

Prepare a small lab and try a dry run first (the time stamp in the output is the time of your run):

```bash
mkdir -p "/tmp/lab/my project" && echo hi > "/tmp/lab/my project/a.txt"
cd /tmp/lab
bash ~/scripts/backup.sh -n "my project" out
ls out
```

```text
backup.sh: dry run: would create out/my project-20261118-071250.tar.gz
ls: cannot access 'out': No such file or directory
```

The dry run created nothing, not even the folder `out`. Wrong input is refused with status 2 (try `-k 0`, or a folder that does not exist). Now make five backups with fixed stamps and keep three:

```bash
for day in 1 2 3 4 5; do
  BACKUP_STAMP=2026111$day-100000 bash ~/scripts/backup.sh -k 3 "my project" out
done
ls out
```

```text
backup.sh: created out/my project-20261111-100000.tar.gz
backup.sh: created out/my project-20261112-100000.tar.gz
backup.sh: created out/my project-20261113-100000.tar.gz
backup.sh: created out/my project-20261114-100000.tar.gz
backup.sh: deleted out/my project-20261111-100000.tar.gz
backup.sh: created out/my project-20261115-100000.tar.gz
backup.sh: deleted out/my project-20261112-100000.tar.gz
```

and `ls out` shows the three newest:

```text
my project-20261113-100000.tar.gz
my project-20261114-100000.tar.gz
my project-20261115-100000.tar.gz
```

Clean up the lab: `rm -r /tmp/lab`.

## 7. Check and test your scripts

**Syntax check.** `bash -n script.sh` reads the script without running it and reports syntax errors. For a script whose `if` has no `fi`:

```text
syn.sh: line 4: syntax error: unexpected end of file from `if' command on line 2
```

**Trace.** `set -x` makes bash print each command after expansion, with a `+` in front, until `set +x`. Use it around the lines you doubt.

**ShellCheck** is the most useful helper. Running it on the unsafe `quoting.sh` and `esurprise.sh` gives (shortened):

```text
In quoting.sh line 7:
for f in $(ls); do echo "item: [$f]"; done
         ^---^ SC2045 (error): Iterating over ls output is fragile. Use globs.

In esurprise.sh line 12:
f() { local out=$(false); echo "3: local hid the failure, out=[$out]"; }
            ^-^ SC2155 (warning): Declare and assign separately to avoid masking return values.
```

It found the mistakes of Sections 3 and 4 without running anything. You do not have to install it: the authors publish an image on Docker Hub ([Lesson 11.12](12-containers-and-docker-basics.md) explains Docker; for now just run the line). Run this on your **Mac** in the folder with the scripts:

```bash
docker run --rm -v "$PWD":/mnt koalaman/shellcheck:stable backup.sh
```

`backup.sh` produces no output: it is clean. (Inside the VM you can install it with `sudo apt-get install -y shellcheck`; [Lesson 11.7](07-package-managers.md) explains `apt`.) Fix every warning, or silence one deliberately with a comment `# shellcheck disable=SC2086` plus the reason.

> 🧪 **Testing:** A script deserves a test like any program. The test below runs `backup.sh` in a scratch folder and compares the results. Save it as `test-backup.sh` next to the script:

```bash
#!/usr/bin/env bash
# test-backup.sh - tests for backup.sh. Run it with: bash test-backup.sh
set -u

here=$(cd "$(dirname "$0")" && pwd)
script=$here/backup.sh
failures=0

check() {   # check "description" expected actual
    if [[ $2 == "$3" ]]; then
        echo "ok   - $1"
    else
        echo "FAIL - $1 (expected [$2], got [$3])"
        failures=$((failures + 1))
    fi
}

count_files() {   # count_files DIR: how many entries are in DIR
    local entries=("$1"/*)
    echo "${#entries[@]}"
}

work=$(mktemp -d)
trap 'rm -rf "$work"' EXIT
mkdir -p "$work/data dir"
echo hello > "$work/data dir/a.txt"

# 1. Five runs with "keep 2" leave exactly two archives.
for n in 1 2 3 4 5; do
    BACKUP_STAMP=20260101-00000$n "$script" -k 2 "$work/data dir" "$work/out" 2> /dev/null
done
check "keeps only 2" 2 "$(count_files "$work/out")"

# 2. A dry run changes nothing.
"$script" -n "$work/data dir" "$work/out" 2> /dev/null
check "dry run changes nothing" 2 "$(count_files "$work/out")"

# 3. Bad input gives status 2.
"$script" -k abc "$work/data dir" "$work/out" 2> /dev/null
check "bad count gives status 2" 2 "$?"

if (( failures > 0 )); then
    echo "$failures test(s) failed"
    exit 1
fi
echo "all tests passed"
```

```bash
bash test-backup.sh
```

```text
ok   - keeps only 2
ok   - dry run changes nothing
ok   - bad count gives status 2
all tests passed
```

**What happens:** `check` compares an expected and an actual value and counts failures. This test script does **not** use `set -e` (a failing `backup.sh` call is part of the test, not a reason to stop), but it does use `set -u`. The final `exit 1` makes the test usable in a CI job. A test that never fails proves nothing: to see that it works, break the pruning on purpose (for example set `excess=0` in a copy of `backup.sh`) and the test reports `FAIL - keeps only 2 (expected [2], got [5])`.

## Common mistakes

1. **No quotes around a variable.** `rm $file` breaks on spaces and globs. *Fix:* `rm -- "$file"`; run ShellCheck.
2. **`cd` without a check.** `cd "$dir"; rm -rf *` deletes in the wrong folder when `cd` fails. *Fix:* `set -e`, or `cd "$dir" || exit 1`, and prefer `rm -rf -- "${dir:?}/"*` or paths that do not need `cd`.
3. **A fixed temporary name** such as `/tmp/out.txt`. *Fix:* `mktemp`, and a `trap ... EXIT` that removes it.
4. **`set -e` in a script that uses `grep`** and treats status 1 as a crash. *Fix:* `if grep -q ...; then`, or `grep ... || true`.
5. **Messages on standard output.** They end up inside the output that another program reads. *Fix:* errors and logs go to standard error (`>&2`).

## Try it yourself

Do this from `~/scripts` inside the VM.

1. Write `hello.sh` with strict mode that prints `Hello, NAME!` where NAME is `$1`, and prints a usage line to standard error and exits with status 2 if there is no argument. Test both cases with `echo $?`.
2. Run `bash -n hello.sh`, then `bash -x hello.sh Ann`.
3. Remove the quotes around `"$1"`, call it with `"Ann Lee"`, and ask ShellCheck what it thinks.

## Exercises

### Exercise 1 — Find the three bugs (easy)

This script should print the number of lines of each `.txt` file in a folder given as `$1`. It has three problems that strict mode or quoting would fix. Name them and write the corrected script.

```bash
#!/bin/bash
for f in $(ls $1/*.txt); do
    n=$(wc -l < $f)
    echo $f $n
done
```

<details>
<summary>Hint</summary>

Think about: a folder or file name with a space; a missing argument; a folder with no `.txt` files.

</details>

<details>
<summary>Answer</summary>

```bash
#!/usr/bin/env bash
set -euo pipefail

dir=${1:?usage: count-lines.sh DIR}
shopt -s nullglob
for f in "$dir"/*.txt; do
    n=$(wc -l < "$f")
    printf '%s %s\n' "$f" "$n"
done
```

Why it works: `${1:?...}` stops with a message when the argument is missing (without it `$1/*.txt` would be `/*.txt`). The glob `"$dir"/*.txt` replaces `ls`, keeps spaces in names, and `nullglob` makes the loop run zero times when nothing matches (otherwise the loop would run once with the text `*.txt`). `"$f"` is quoted in `wc` and `printf`.

</details>

### Exercise 2 — A safe `cleanup-old.sh` and its test (medium)

Write `cleanup-old.sh DIR DAYS`, which **prints** (does not delete) the files in `DIR` that were changed more than `DAYS` days ago, using `find`. It must refuse a missing `DIR` and a non-numeric `DAYS`, with status 2. Then write `test-cleanup-old.sh`: in a `mktemp -d` folder create one old file (`touch -d '10 days ago'`) and one new file, and check that `DAYS=5` prints only the old file, `DAYS=20` prints nothing, and `DAYS=abc` gives status 2.

<details>
<summary>Hint</summary>

`find "$dir" -type f -mtime +"$days" -print`. Check with `[[ $days =~ ^[0-9]+$ ]]`. In the test, capture the output with `out=$(bash cleanup-old.sh "$work" 5)` and reuse `check` from `test-backup.sh`.

</details>

<details>
<summary>Answer</summary>

```bash
#!/usr/bin/env bash
set -euo pipefail

die() { echo "cleanup-old.sh: $*" >&2; exit 2; }

[[ $# -eq 2 ]] || die "usage: cleanup-old.sh DIR DAYS"
dir=$1
days=$2
[[ -d $dir ]] || die "not a directory: $dir"
[[ $days =~ ^[0-9]+$ ]] || die "DAYS must be a number, got '$days'"

find "$dir" -type f -mtime +"$days" -print
```

and the test:

```bash
#!/usr/bin/env bash
set -u

here=$(cd "$(dirname "$0")" && pwd)
failures=0
check() {
    if [[ $2 == "$3" ]]; then echo "ok   - $1"; else echo "FAIL - $1 (expected [$2], got [$3])"; failures=$((failures + 1)); fi
}

work=$(mktemp -d)
trap 'rm -rf "$work"' EXIT
touch -d '10 days ago' "$work/old.txt"
touch "$work/new.txt"

check "5 days lists the old file" "$work/old.txt" "$(bash "$here/cleanup-old.sh" "$work" 5)"
check "20 days lists nothing" "" "$(bash "$here/cleanup-old.sh" "$work" 20)"
bash "$here/cleanup-old.sh" "$work" abc 2> /dev/null
check "bad DAYS gives status 2" 2 "$?"

(( failures == 0 )) || { echo "$failures test(s) failed"; exit 1; }
echo "all tests passed"
```

Running the test prints three `ok` lines and `all tests passed`.

Why it works: every input is checked before use, errors go to standard error with status 2, and the script only prints, so a mistake cannot delete anything. `-mtime +N` means "modified more than N days ago". The scratch folder is private and removed by the trap even when a check fails, and `touch -d` makes a file look old without waiting.

</details>

## Quiz

1. What does `set -e` do, and name one situation where it does not stop the script.

<details>
<summary>Answer</summary>

It ends the script when a simple command fails. It does not stop for a command used as an `if` condition, on the left of `||` or `&&`, or in a failed `local x=$(cmd)` (the status of `local` is used).

</details>

2. Which exit status does a pipeline `false | cat` have with and without `pipefail`?

<details>
<summary>Answer</summary>

Without `pipefail` it is 0 (the status of the last command, `cat`). With `pipefail` it is 1.

</details>

3. What is the difference between `$@`, `"$@"` and `"$*"`?

<details>
<summary>Answer</summary>

`"$@"` expands to all arguments, each as one separate word (correct for passing arguments on). `"$*"` joins them into one word. Unquoted `$@` and `$*` are split again at spaces, which breaks arguments that contain spaces.

</details>

4. When does the `EXIT` trap run?

<details>
<summary>Answer</summary>

Whenever the script ends: at the last line, at `exit`, after an error stopped it with `set -e`, and after a signal that was turned into an `exit` by another trap. It does not run if the script is killed with `SIGKILL`.

</details>

5. Why does `backup.sh` write to a `mktemp` file and then `mv` it?

<details>
<summary>Answer</summary>

Renaming inside one folder is atomic, so a reader never sees a half-written archive. If `tar` fails, the temporary file is removed by the trap and no broken backup remains under the final name.

</details>

## Summary

- Without strict mode a bash script goes on after errors and may report success. Start with `set -euo pipefail`.
- `set -e` has exceptions (conditions, `||`, `local x=$(...)`, `((i++))` at zero); handle expected failures yourself.
- Quote every variable and `$(...)`, pass arguments with `"$@"`, end options with `--`, and use `[[ ]]` for tests.
- Use `mktemp` for scratch files and `trap ... EXIT` so that clean-up always happens; turn `INT` and `TERM` into `exit`.
- A good script has `-h`, validates its input, writes messages to standard error, exits with status 2 on wrong usage, and offers a dry run when it deletes things.
- Check with `bash -n`, `set -x` and ShellCheck, and write a test script for it.

## Sources and further reading

- [GNU Bash Reference Manual — The Set Builtin](https://www.gnu.org/software/bash/manual/html_node/The-Set-Builtin.html) — the official description of `set -e`, `-u` and `pipefail`.
- [GNU Bash Reference Manual — Shell Expansions](https://www.gnu.org/software/bash/manual/html_node/Shell-Expansions.html) — word splitting, filename expansion and quoting in detail.
- [GNU Bash Reference Manual — Bourne Shell Builtins (trap, getopts)](https://www.gnu.org/software/bash/manual/html_node/Bourne-Shell-Builtins.html) — `trap`, `getopts`, `shift` and the others.
- [ShellCheck](https://www.shellcheck.net/) — the online checker and the wiki page for every warning code such as SC2086.
- [mktemp(1) manual page](https://man7.org/linux/man-pages/man1/mktemp.1.html) — how temporary files and folders are created.

---

[🏠 Start](../START-HERE.md) · [← 11.8 Bash scripting basics](08-bash-scripting-basics.md) · [📋 Phase 11](README.md) · [11.10 SSH and remote servers →](10-ssh-and-remote-servers.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
