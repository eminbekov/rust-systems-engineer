<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 00.3 How a computer runs a program](03-how-a-computer-runs-a-program.md) · [📋 Phase 00](README.md) · [00.5 Installing Rust and RustRover →](05-install-rust-and-rustrover.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/00-getting-started/04-terminal-basics.md) <!-- nav -->

# Lesson 00.4 — The terminal: your first commands

> **Phase 00 — Getting started** · Lesson 4 of 8 · ⏱ about 2.5 hours (2 sittings)

## What you will learn

- What the terminal, the shell and a command are, and how to read the parts of a command.
- How the folders on your Mac are organised, and how to move around them with paths.
- The core commands: `pwd`, `ls`, `cd`, `mkdir`, `touch`, `echo`, `cat`, `cp`, `mv`, `rm`.
- How to get help, combine commands with a pipe, and save output to a file.
- What `PATH` and file permissions are, and why they cause the famous "command not found" and "permission denied" errors.

## Before you start

- You finished [Lesson 00.3 — How a computer runs a program](03-how-a-computer-runs-a-program.md).
- Open the Terminal application: press `Cmd + Space`, type `Terminal`, press Enter. Keep it open next to this lesson.
- Everything in this lesson is safe: we work inside the course folder and only create and delete things that we create ourselves.

## New words

| Word | Meaning |
|---|---|
| **terminal** | A window in which you type commands and see text answers. |
| **shell** | The program inside the terminal that reads your command and runs it. On macOS the default shell is **zsh**. |
| **CLI** (Command-Line Interface) | A way to use programs by typing commands. The opposite of a **GUI** (Graphical User Interface), where you click windows and buttons. |
| **prompt** | The short text at the start of the line (often ending in `%`) that says: "I am waiting for your command". |
| **command** | A word that tells the shell which program to run, for example `ls`. |
| **argument** | Extra information given to a command, for example the name of a folder. |
| **option** (flag) | A switch that changes how a command works, written with a dash: `-l`. |
| **directory** (folder) | A container for files and other directories. |
| **path** | The "address" of a file or directory, for example `/Users/student/projects`. |
| **absolute path** | A path that starts at the very top of the file system, with `/`. |
| **relative path** | A path that starts from the directory where you are now. |
| **home directory** | Your personal directory, written `~`. Your documents, downloads and projects live here. |
| **working directory** | The directory where you currently are in the terminal. Also called the current directory. |
| **hidden file** | A file whose name starts with a dot (for example `.gitignore`). `ls` does not show it unless you ask. |
| **pipe** | The symbol `\|` that sends the output of one command into the next command. |
| **redirect** | The symbols `>` and `>>` that send the output of a command into a file instead of the screen. |
| **environment variable** | A named setting that every program started from the shell can read, for example `HOME` or `PATH`. |
| **PATH** | An environment variable: the list of directories where the shell looks for commands. |
| **permission** | A flag on a file that says who may read it, change it or run it. |
| **script** | A text file with commands that the shell can run one after another. |

## 1. What a terminal is

Until now you have used your Mac by clicking. The terminal is the other way: you **type** what
you want, press Enter, and the computer answers with text. This is called a command-line
interface (**CLI**). Programmers use it all day because typing is faster, can be repeated and
saved, and nearly every programming tool (Git, Cargo, the Rust compiler) is controlled this way.

Three words are often mixed up:

- The **terminal** is the window.
- The **shell** is the program inside it that understands what you type. Yours is `zsh`.
- The **prompt** is the text that the shell shows, waiting for you.

Your prompt probably looks like `yourname@yourmac ~ %`. In this lesson we write only `%` for the
prompt. **You never type the `%`**, only what comes after it. Lines without `%` are the
answer of the computer.

> 💡 **Tip:** The terminal is not dangerous by itself. It just does what you say, literally (remember
> the strict helper from Lesson 00.2). The only dangerous commands are those that delete, and we handle them
> carefully in section 6.

## 2. Anatomy of a command

Try the first command. Type it and press Enter:

```text
% echo hello
hello
```

What happens:

1. `%` is the prompt (not typed).
2. `echo` is the **command**: a program that prints the words you give it.
3. `hello` is the **argument**: what to print.
4. The shell ran `echo`, which printed `hello`, then showed a new prompt.

A command can also have **options** (switches). Options usually start with a dash:

```text
% ls -l notes
        ^  ^
        |  +-- argument: which directory to list
        +----- option: "long" format
```

The general shape is: `command [options] [arguments]`, separated by spaces. Remember two rules:

- **Spaces separate words.** If a name contains a space, you must put it in quotes (see section 6).
- **Case matters.** `Notes` and `notes` are different names. `LS` is not `ls`.

## 3. Where am I? Paths

Files on your Mac live in a tree of directories. At the top is `/`, the **root**. Everything is below it.

```text
/                          <- root: the top of everything
├── Applications
├── System
└── Users
    └── student            <- your home directory, written ~
        ├── Documents
        ├── Downloads
        └── projects
            └── rust       <- ~/projects/rust, your course folder
                ├── course
                ├── ru
                └── code
```

(Your user name is not `student`; use your own wherever you see it.)

The command `pwd` (print working directory) tells you where you are:

```text
% pwd
/Users/student
```

A **path** is the route from somewhere to a file. There are two kinds:

| Kind | Starts with | Example | Meaning |
|---|---|---|---|
| Absolute | `/` | `/Users/student/projects/rust` | The full route from the root. Works from anywhere. |
| Relative | anything else | `projects/rust` | The route from where you are now. |

Four special names make paths short:

| Name | Means |
|---|---|
| `~` | your home directory (`/Users/student`) |
| `.` | the current directory |
| `..` | the parent directory (one level up) |
| `/` | the root |

## 4. Looking around and moving: `ls` and `cd`

`ls` lists what is in a directory. Without arguments it lists the current one.

```text
% ls
Documents
Downloads
projects
```

> The real `ls` prints names side by side in columns on your screen. In this lesson we show one name per
> line, to save space.

Useful options:

| Command | Meaning |
|---|---|
| `ls -l` | long format: permissions, owner, size, date |
| `ls -a` | show all, including hidden files (names starting with a dot) |
| `ls -la` | both together |
| `ls -R` | list sub-directories too (recursive) |

```text
% ls -a
.
..
Documents
Downloads
projects
```

`.` and `..` are always there. Your Mac also hides files such as `.zshrc` in your home directory.
In Finder, press `Cmd + Shift + .` to show or hide hidden files.

`cd` (change directory) moves you:

```text
% cd projects
% pwd
/Users/student/projects
% cd rust
% pwd
/Users/student/projects/rust
% cd ..
% pwd
/Users/student/projects
% cd ~
% pwd
/Users/student
```

What happens:

1. `cd projects` — a relative path: go into `projects` inside the current directory.
2. `cd rust` — go one more level down.
3. `cd ..` — go up one level, back to `projects`.
4. `cd ~` — jump home, from anywhere. `cd` with no argument does the same thing. `cd -` goes back to the previous directory.

> 💡 **Tip:** Press **Tab** while typing a name. The shell completes it (`cd pro` + Tab becomes
> `cd projects/`). If several names fit, press Tab twice to see them. This saves typing and prevents typos.

Now move into the course folder and look at it. This is your real folder (the files you see will be a little different):

```text
% cd ~/projects/rust
% ls
GLOSSARY.md
START-HERE.md
course
ru
```

## 5. Creating things: `mkdir`, `touch`, `echo`, `cat`

Create the folders that the course will use (see [Lesson 00.1](01-how-to-use-this-course.md)). Look at each result:

```text
% mkdir code
% mkdir notes
% ls
GLOSSARY.md
START-HERE.md
code
course
notes
ru
```

`mkdir` (make directory) creates a directory. With `-p` it creates a whole chain and does not
complain if it already exists:

```text
% mkdir -p code/demo/src
% ls -R code
demo

code/demo:
src

code/demo/src:
```

(`code/demo` is only for practice. We remove it in section 6. The real project folders will be created by Cargo in Lesson 00.6.)

`touch` creates an empty file (or updates the date of an existing one). `echo` prints text, and with
a **redirect** it writes the text into a file:

```text
% touch notes/day1.md
% echo 'Day 1: I learned the terminal.' > notes/day1.md
% cat notes/day1.md
Day 1: I learned the terminal.
% echo 'Next: install Rust.' >> notes/day1.md
% cat notes/day1.md
Day 1: I learned the terminal.
Next: install Rust.
```

What happens:

1. `touch notes/day1.md` — create the empty file `day1.md` inside `notes`.
2. `echo '...' > file` — the `>` sends the output of `echo` into the file. **It replaces everything that was in the file.**
3. `cat file` (concatenate) — print the content of a file on the screen.
4. `echo '...' >> file` — `>>` **appends** to the end of the file instead of replacing.

> ⚠️ **Common mistake:** Using `>` when you wanted `>>`. The old content of the file is gone, with no
> question asked and no Trash to recover it from.

## 6. Copy, move, rename, delete

| Command | What it does | Example |
|---|---|---|
| `cp source target` | copy a file | `cp notes/day1.md notes/backup.md` |
| `cp -r source target` | copy a directory with everything inside | `cp -r code code2` |
| `mv source target` | move a file, or rename it | `mv notes/backup.md notes/old.md` |
| `rm file` | delete a file | `rm notes/old.md` |
| `rm -r directory` | delete a directory with everything inside | `rm -r code2` |
| `rmdir directory` | delete an **empty** directory | `rmdir code2` |

```text
% cp notes/day1.md notes/day1-backup.md
% ls notes
day1-backup.md
day1.md
% mv notes/day1-backup.md notes/old.md
% ls notes
day1.md
old.md
% rm notes/old.md
% ls notes
day1.md
```

What happens:

1. `cp` made a second file with the same content.
2. `mv` moved it to a new name. Moving to a new name in the same directory *is* renaming.
3. `rm` deleted it. `ls` confirms that only `day1.md` is left.

Now remove the practice folder from section 5. First look, then delete, then look again (`ls code` prints nothing when the folder is empty):

```text
% ls code
demo
% rm -r code/demo
% ls code
```

### Deleting is final

> 🔒 **Security:** `rm` does **not** use the Trash. Deleted files are gone. Follow these habits:
> 1. Before `rm something`, run `ls something` with the same argument and check the result.
> 2. Prefer `rm file` over `rm -r directory`, and think twice before `rm -r`.
> 3. Never run a command containing `rm -rf` that you copied from the internet without understanding every part.
> 4. Never type `rm -r` followed by `/`, `~` or `*` unless you are absolutely sure. `rm -r ~` would delete your whole home directory.

### Names with spaces

Spaces separate words, so `mkdir my notes` makes **two** directories, `my` and `notes`. Use quotes:

```text
% mkdir 'my notes'
% ls
code
my notes
notes
% rmdir 'my notes'
```

> 📏 **Convention:** Programmers avoid spaces in file and directory names. Use lowercase letters, digits,
> dashes and underscores: `p00-hello`, `day1.md`, `my_notes`. Then you never need quotes.

## 7. Getting help

You will not remember every option, and you do not have to. Use these:

- **`man command`** opens the manual page of a command. Use the arrow keys to scroll, `/word` to search, and **`q` to quit**. Try `man ls`.
- **`command --help`** prints a short summary, for programs that support it (`cargo --help` works, for instance).
  Many of the older tools built into macOS do not understand `--help`:

```text
% ls --help
ls: unrecognized option `--help'
usage: ls [-@ABCFGHILOPRSTUWXabcdefghiklmnopqrstuvwxy1%,] [--color=when] [-D format] [file ...]
```

  The message is not useless: the second line, "usage", lists all the options. Use `man ls` for the
  explanation.
- **`type command`** and **`which command`** tell you what a command is and where it lives:

```text
% type cd
cd is a shell builtin
% which ls
/bin/ls
```

`cd` is built into the shell itself, so it has no file. `ls` is a program in the file `/bin/ls`.

## 8. Pipes and searching

The real power of the shell is combining small commands. The **pipe** `|` takes the output of the left
command and uses it as the input of the right one.

```text
% cat notes/day1.md | wc -l
       2
% ls | head -2
GLOSSARY.md
START-HERE.md
% grep Rust notes/day1.md
Next: install Rust.
% find . -name '*.md'
./notes/day1.md
```

What happens:

1. `wc -l` (word count, lines) counts the lines of its input. The pipe fed it the content of `day1.md`: 2 lines. (`wc -w` counts words.)
2. `head -2` shows only the first 2 lines of its input (`tail` shows the last ones).
3. `grep Rust file` prints the lines of a file that contain the word `Rust`. `grep -i` ignores upper/lower case, `grep -n` also shows line numbers.
4. `find . -name '*.md'` searches the current directory (`.`) and everything below it for names that match `*.md`. The `*` is a **wildcard**: it stands for "any characters". The quotes stop the shell from expanding the `*` itself.

> 💡 **Tip:** You can use the wildcard directly with other commands: `ls *.md` lists all Markdown files in the
> current directory.

## 9. Shortcuts that save hours

| Keys | What they do |
|---|---|
| **Tab** | complete a name |
| **↑ / ↓** | previous / next command from history |
| **Ctrl + R** | search the history: type a few letters of an old command |
| **Ctrl + C** | **stop** the running command (the most important key!) |
| **Ctrl + A / Ctrl + E** | jump to the start / the end of the line |
| **Ctrl + L** (or `clear`) | clear the screen |
| **Cmd + T** | open a new terminal tab |

> 💡 **Tip:** If a program seems stuck and the prompt does not come back, press **Ctrl + C**. This is
> the universal "stop".

## 10. How the shell finds commands: `PATH`

When you type `git`, how does the shell know where the program `git` is? It looks through a list of
directories, in order, until it finds a file with that name. The list is stored in an **environment
variable** called `PATH`. You can print any variable with `echo` and a `$` in front of its name:

```text
% echo $HOME
/Users/student
% echo $SHELL
/bin/zsh
% echo $PATH
/usr/local/bin:/usr/bin:/bin:/usr/sbin:/sbin
```

(The last line is a typical value. Yours will be longer.) `PATH` is a list of directories separated by
colons. The shell looks at `/usr/local/bin` first, then `/usr/bin`, and so on.

To see that this is real, we start a clean shell with a deliberately short `PATH`. The program `git` lives in
`/usr/bin`, so a `PATH` that contains only `/bin` cannot find it:

```text
% which git
/usr/bin/git
% env -i PATH=/bin /bin/zsh -f -c 'git --version'
zsh:1: command not found: git
% env -i PATH=/usr/bin:/bin /bin/zsh -f -c 'git --version'
git version 2.54.0 (Apple Git-157)
```

What happens:

1. `which git` — the file lives in `/usr/bin/git`.
2. The second command starts a fresh shell whose `PATH` is only `/bin`. It does not look in `/usr/bin`, so it says `command not found`, although `git` is installed.
3. The third one has `/usr/bin` in its `PATH` and finds `git`.

> 💡 **Tip:** On a new Mac, `/usr/bin/git` is only a small placeholder until Apple's developer tools are installed. If a window appears and offers to install the "command line developer tools", you can click **Install** now, or **Cancel** and do it in Lesson 00.5. The idea of `PATH` is the same either way.

You will need this in [Lesson 00.5](05-install-rust-and-rustrover.md): the Rust installer adds the
directory `~/.cargo/bin` to your `PATH`, and the most common installation problem is a terminal
that was opened *before* that change. The cure is to close the terminal tab and open a new one.

## 11. Files that can run: permissions

In Lesson 00.3 you learned that a program is a file. What makes a file *runnable*? A **permission**.
`ls -l` shows them. Here is what it shows for a small script file `hello.sh` (do not type this yet: we create the file a few lines below):

```text
% ls -l hello.sh
-rw-r--r--  1 student  staff  38 Nov 17 16:26 hello.sh
```

How to read it, from the left:

| Part | Meaning |
|---|---|
| `-` | the type: `-` is a file, `d` is a directory |
| `rw-` | the owner may **r**ead and **w**rite, but not e**x**ecute (run) |
| `r--` | the group may only read |
| `r--` | everybody else may only read |
| `1` | number of links (ignore) |
| `student` `staff` | the owner and the group |
| `38` | the size in bytes |
| `Nov 17 16:26` | the date of the last change |
| `hello.sh` | the name |

(A `@` or `+` right after the permissions on your screen is only a note about extra attributes. Ignore it.)

Let us make a **script**: a text file with commands. We write it with `cat` and a "here-document" (the
text between the two `XX` lines goes into the file):

```text
% cat > hello.sh <<'XX'
#!/bin/zsh
echo "Hello from a script"
XX
% ./hello.sh
zsh: permission denied: ./hello.sh
% chmod +x hello.sh
% ls -l hello.sh
-rwxr-xr-x  1 student  staff  38 Nov 17 16:26 hello.sh
% ./hello.sh
Hello from a script
```

What happens:

1. The first line `#!/bin/zsh` ("shebang") tells the system which program should read this file. The second line is the command.
2. `./hello.sh` — the `./` means "in the current directory". Without `./` the shell would search `PATH`, not the current directory. The result is **permission denied**: the file has no `x` (execute) permission.
3. `chmod +x hello.sh` — change mode: add the execute permission. Now `ls -l` shows `-rwxr-xr-x`: three `x`.
4. The script runs. A compiled Rust program is also just a file with the `x` permission.

When you are finished, delete the practice file: `rm hello.sh`.

## 12. Trust, `sudo` and copy-pasted commands

> 🔒 **Security:** Every command runs with your rights, so it can delete or change anything you can.
> - Read a command **before** you press Enter. Understand each part, or do not run it.
> - `sudo` runs a command as the administrator. Never put `sudo` in front of a command that you do not understand, and never because an error message "fixed itself" on a forum. Our course asks for `sudo` almost never.
> - Some install instructions say `curl ... | sh`: download a script and run it at once. It is
>   as safe as the website it comes from. We will use it once, for the official Rust installer, and we
>   will check the address carefully.
> - Never paste a command that contains your passwords or tokens into a chat or a forum.

## Common mistakes

1. **Typing the prompt.** You copied `% ls` including the `%`. *Fix:* type only `ls`.
2. **A typo in the command.** Typing `sl` instead of `ls`. The shell answers `zsh: command not found: sl`. Read the name; fix the typo.
3. **Wrong directory.** `cat notes/day1.md` says `cat: notes/day1.md: No such file or directory` because you are not in `~/projects/rust`. *Fix:* run `pwd`, then `cd` to the right place.
4. **Forgetting `./`.** The script is in the current directory, but `hello.sh` says `zsh: command not found: hello.sh`. *Fix:* `./hello.sh`.
5. **Making a directory that exists.** `mkdir notes` says `mkdir: notes: File exists`. Not a danger; use `mkdir -p`.
6. **`rmdir` on a non-empty directory.** It says `rmdir: notes: Directory not empty`. Use `rm -r notes` — but only after checking with `ls notes`.
7. **Trying to `cd` into a file.** `cd file.txt` says `cd: not a directory: file.txt`.
8. **Unquoted spaces.** `cat my file.txt` tries to read two files, `my` and `file.txt`:

```text
% cat my file.txt
cat: my: No such file or directory
```

   *Fix:* `cat 'my file.txt'`.

## Try it yourself

Follow these steps exactly. You will end with a real note file for your course.

1. Open Terminal. Type `pwd` and read the answer.
2. Type `cd ~/projects/rust` and then `pwd` again. Check that the path ends with `projects/rust`.
3. Type `ls`. Check that you see `course` and `ru`.
4. Create the folders (if they do not exist yet): `mkdir -p code notes`. Then `ls` to see them.
5. Create your first note: `echo '# Day 1' > notes/day1.md`.
6. Add two more lines with `>>`: `echo 'I can use the terminal.' >> notes/day1.md` and `echo 'Next: install Rust.' >> notes/day1.md`.
7. Show the file: `cat notes/day1.md`. Expected:

```text
# Day 1
I can use the terminal.
Next: install Rust.
```

8. Count the lines with a pipe: `cat notes/day1.md | wc -l`. Expected: `3`.
9. Press the **↑** key until the `mkdir` command comes back. Press **Ctrl + C** to cancel it without running.
10. Run `man ls`, scroll with the arrows, and press `q` to leave.

## Exercises

### Exercise 1 — Navigate (easy)

Starting from your home directory, write the commands to: go to `~/projects/rust/code`, show the
current directory, go one level up, and go home again.

<details>
<summary>Hint</summary>

You need `cd`, `pwd`, `cd ..` and `cd ~`.

</details>

<details>
<summary>Answer</summary>

```text
% cd ~/projects/rust/code
% pwd
/Users/student/projects/rust/code
% cd ..
% cd ~
```

Why it works: `~` expands to the home directory, so the first path is absolute. `..` is the parent, and `cd ~` goes home from anywhere.

</details>

### Exercise 2 — A small structure (easy)

Inside `~/projects/rust/notes`, create a folder `week1`. Inside it, create the files `monday.md` and
`tuesday.md`, each with one line of text. Then list the folder.

<details>
<summary>Hint</summary>

`mkdir`, then `echo '...' > file` twice, then `ls week1`.

</details>

<details>
<summary>Answer</summary>

```text
% cd ~/projects/rust/notes
% mkdir week1
% echo 'Monday: terminal basics' > week1/monday.md
% echo 'Tuesday: first program' > week1/tuesday.md
% ls week1
monday.md
tuesday.md
```

Why it works: `>` creates the file if it does not exist, so `touch` is not needed.

</details>

### Exercise 3 — Copy, rename, delete (medium)

Copy `monday.md` to `monday-copy.md`, rename the copy to `old.md`, and delete `old.md`. After each
step show the folder. At the end only `monday.md` and `tuesday.md` should remain.

<details>
<summary>Hint</summary>

`cp`, `mv`, `rm`, and `ls` between them. Run `ls week1` before `rm`.

</details>

<details>
<summary>Answer</summary>

```text
% cd ~/projects/rust/notes/week1
% cp monday.md monday-copy.md
% mv monday-copy.md old.md
% ls
monday.md
old.md
tuesday.md
% rm old.md
% ls
monday.md
tuesday.md
```

Why it works: `mv` within one directory renames; `ls` before `rm` confirms that we delete the right file.

</details>

### Exercise 4 — Pipes (medium)

You have a file `words.txt` with the line `one two three`. Create it, then count its words with
`wc` and show the name of every `.md` file in `~/projects/rust/notes` and below.

<details>
<summary>Hint</summary>

`echo` with `>`, then `wc -w words.txt`. For the search use `find` with `-name`.

</details>

<details>
<summary>Answer</summary>

```text
% cd ~/projects/rust/notes
% echo 'one two three' > words.txt
% wc -w words.txt
       3 words.txt
% find . -name '*.md'
./day1.md
./week1/monday.md
./week1/tuesday.md
% rm words.txt
```

Why it works: `wc -w` counts words; `find` walks through all subdirectories. The order of the files that `find` prints may differ on your computer.

</details>

### Exercise 5 — Find the mistake (hard)

A friend types these commands in `~/projects/rust` and gets errors. Explain each problem and fix it.

```text
% cd notes/week 1
% cat notes/day1.md | wc -l
% hello.sh
```

(Assume `week 1` is a directory with a space in its name, the script `hello.sh` is in the current directory, and it has
the `x` permission.)

<details>
<summary>Hint</summary>

Look at spaces, at the current directory, and at how the shell finds scripts.

</details>

<details>
<summary>Answer</summary>

1. `cd notes/week 1` gives `cd` two arguments. *Fix:* `cd 'notes/week 1'` (or use Tab completion, which adds the backslash for you: `cd notes/week\ 1`).
2. After the first command the friend is in `notes/week 1`, so `notes/day1.md` is the wrong relative path: there is no `notes` folder here. *Fix:* `cd ~/projects/rust` first, or use the absolute path `cat ~/projects/rust/notes/day1.md | wc -l`.
3. `hello.sh` is not found because the current directory is not in `PATH`. *Fix:* `./hello.sh`.

Why it works: you always have to ask: where am I (`pwd`), what exactly did the shell receive (quotes!), and does the shell look for the command in the right place (`PATH`, `./`).

</details>

## Quiz

1. What is the difference between a terminal and a shell?

<details>
<summary>Answer</summary>

The terminal is the window. The shell (zsh) is the program inside it that reads and runs your commands.

</details>

2. What do `.`, `..` and `~` mean in a path?

<details>
<summary>Answer</summary>

`.` is the current directory, `..` is its parent, `~` is your home directory.

</details>

3. What is the difference between `>` and `>>`?

<details>
<summary>Answer</summary>

`>` replaces the content of the file with the output; `>>` appends the output to the end.

</details>

4. Why should you run `ls` before `rm`?

<details>
<summary>Answer</summary>

`rm` is final, there is no Trash. `ls` with the same argument shows exactly what would be deleted.

</details>

5. What does `command not found` mean, and how can `PATH` cause it?

<details>
<summary>Answer</summary>

The shell could not find a program with that name in any directory listed in `PATH`. Either the name is mistyped, the program is not installed, or the directory that contains it is not in `PATH`.

</details>

6. What does `chmod +x file` do, and why is it needed?

<details>
<summary>Answer</summary>

It adds the execute permission. Without it the system refuses to run the file ("permission denied").

</details>

7. Which key combination stops a running command?

<details>
<summary>Answer</summary>

Ctrl + C.

</details>

## Summary

- The terminal is the window, the shell (zsh) understands commands, the prompt waits for them.
- A command has the shape `command [options] [arguments]`; spaces separate words, case matters.
- Paths are absolute (start with `/`) or relative; `~` is home, `.` current, `..` parent.
- Core commands: `pwd`, `ls`, `cd`, `mkdir`, `touch`, `echo`, `cat`, `cp`, `mv`, `rm`.
- `>` replaces a file, `>>` appends; `|` feeds the output of one command to the next.
- `rm` is final: check with `ls` first; avoid `rm -r` unless sure.
- Use `man`, `--help`, `type` and `which` to get help; use Tab and ↑ to type less; Ctrl + C stops.
- `PATH` is the list of directories where the shell looks for commands; permissions decide whether a file can run.

## Sources and further reading

- [Terminal User Guide for Mac](https://support.apple.com/guide/terminal/welcome/mac) — Apple's official guide to the Terminal application.
- [The Z Shell Documentation](https://zsh.sourceforge.io/Doc/) — the official manual of zsh, the default shell on macOS.
- [Rust Book — Installation](https://doc.rust-lang.org/book/ch01-01-installation.html) — the section "Command Line Notation" describes how the book writes terminal examples.
- [Pro Git — Git Basics](https://git-scm.com/book/en/v2) — the official Git book; every Git command in Phase 01 is typed in the terminal that you just learned.

---

[🏠 Start](../START-HERE.md) · [← 00.3 How a computer runs a program](03-how-a-computer-runs-a-program.md) · [📋 Phase 00](README.md) · [00.5 Installing Rust and RustRover →](05-install-rust-and-rustrover.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
