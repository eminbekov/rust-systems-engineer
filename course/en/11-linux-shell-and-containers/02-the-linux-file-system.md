<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 11.1 Linux and a virtual machine on your Mac](01-linux-and-a-virtual-machine.md) · [📋 Phase 11](README.md) · [11.3 Text tools, pipes and redirection →](03-text-tools-pipes-and-redirection.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/11-linux-shell-and-containers/02-the-linux-file-system.md) <!-- nav -->

# Lesson 11.2 — The Linux file system

> **Phase 11 — Linux, the shell and containers** · Lesson 2 of 16 · ⏱ about 120 minutes

## What you will learn

- How the Linux file system is one tree that starts at `/`, and what the main top-level directories are for.
- Why "everything is a file" in Linux, and how to read system information from `/proc` and `/dev`.
- How to inspect files (`ls -l`, `stat`, `file`, `du`, `df`) and understand hard links and symbolic links.
- How to copy, move and delete safely, and how to find files with `find`.
- Where programs keep their configuration, logs and data, and how to copy a project from your Mac into the VM (the virtual machine `dev` from Lesson 11.1).

## Before you start

- You finished [Lesson 11.1 — Linux and a virtual machine on your Mac](01-linux-and-a-virtual-machine.md) and the VM `dev` runs.
- You know paths, `ls`, `cd`, `cp`, `mv`, `rm`, `mkdir` from [Lesson 00.4](../00-getting-started/04-terminal-basics.md). Here we go deeper and do not repeat the basics.
- Enter the VM: `limactl shell dev`. Every command in this lesson is typed there.

## New words

| Word | Meaning |
|---|---|
| **root directory** | The top of the tree, written `/`. Every other file and directory is below it. (Not the same as the user `root`, who is the administrator.) |
| **FHS** (Filesystem Hierarchy Standard) | The agreement about what the top-level directories (`/etc`, `/var`, `/usr` …) are used for. |
| **inode** | The record inside the file system that stores everything about a file except its name: size, owner, permissions, times, and where the data is. |
| **hard link** | A second *name* for the same file (the same inode). |
| **symbolic link** (symlink) | A tiny file that contains a *path* to another file, like a shortcut. |
| **virtual file system** | Files that do not live on a disk: the kernel creates them on the fly, such as `/proc` and `/dev/null`. |
| **mount point** | A directory where a disk or another file system is attached to the tree. |
| **hidden file** | A file whose name starts with a dot, such as `.bashrc`. `ls` does not show it unless you ask. |
| **MiB / KiB** | Kibibyte and mebibyte: 1024 bytes and 1024 × 1024 bytes. Tools print them as `K`, `M`, `G`. |

## 1. One tree that starts at `/`

On a Mac you have `/Users`, `/Applications`, `/Volumes`. Linux has one tree with one top, the **root
directory** `/`. Even a second disk does not get a letter; it is *mounted* at some directory inside the tree.

Look at the top of the tree:

```bash
ls /
ls -ld /bin /sbin /lib
```

```text
bin  boot  dev  etc  home  lib  media  mnt  opt  proc  root  run  sbin  srv  sys  tmp  usr  var
lrwxrwxrwx 1 root root 7 Apr 20  2026 /bin -> usr/bin
lrwxrwxrwx 1 root root 7 Apr 20  2026 /lib -> usr/lib
lrwxrwxrwx 1 root root 8 Apr 20  2026 /sbin -> usr/sbin
```

Your dates will differ, and the VM may show a few more entries (such as `lost+found`). The `->` arrows mark
symbolic links (Section 5). Each top-level directory has a job, defined by the **FHS** (Filesystem Hierarchy Standard):

| Directory | What lives there | Example |
|---|---|---|
| `/home` | Home directories of normal users | `/home/student` |
| `/root` | Home of the administrator `root` | |
| `/etc` | System-wide **configuration** files (text) | `/etc/hostname`, `/etc/passwd` |
| `/usr` | Installed programs, libraries and shared data | `/usr/bin/ls`, `/usr/share` |
| `/usr/local` | Programs you install yourself, outside the package manager | `/usr/local/bin` |
| `/bin`, `/sbin`, `/lib` | Old names; on Ubuntu they are symlinks to `/usr/bin`, `/usr/sbin`, `/usr/lib` (see `->` above) | |
| `/var` | Data that **changes while the system runs**: logs, caches, databases | `/var/log`, `/var/lib` |
| `/tmp` | Temporary files, any user may write, may be cleared at reboot | |
| `/opt` | Big add-on software packaged by vendors | |
| `/srv` | Data served by this machine (web sites, files) | |
| `/mnt`, `/media` | Places to mount disks and USB (Universal Serial Bus) sticks | |
| `/dev` | **Devices** as files | `/dev/null` |
| `/proc`, `/sys` | Live information from the **kernel** | `/proc/meminfo` |
| `/run` | Runtime data of running services (sockets, PID files: files that hold a process ID, the number of a process) | |
| `/boot` | Files used to start the system (kernel image) | |

> 💡 **Tip:** A short memory rule: `/etc` is **E**ditable **T**ext **C**onfig, `/var` is things that **var**y,
> `/usr` is programs that **us**e the system, `/home` is yours.

Two facts that surprise people coming from macOS:

1. **Names are case-sensitive.** `Notes.txt` and `notes.txt` are two different files. (By default macOS treats them as the same file.)
2. **There is no drive letter and no `/Users`.** Your Mac folder appears at `/Users/...` in the VM only because Lima mounts it there ([Lesson 11.1](01-linux-and-a-virtual-machine.md)).

> 💡 **Tip:** On Ubuntu 26.04 many basic commands such as `ls`, `cp` and `cat` come from *uutils coreutils*,
> a reimplementation of the classic tools written in Rust. Check with `ls --version`. The commands behave
> like the classic ones; only a few error messages are worded differently.

## 2. Everything is a file

A Unix idea: programs read and write **files**, and the system offers many things *as* files — hardware,
processes, kernel settings. Then the same tools (`cat`, `grep`, `>`) work on everything.

**Devices in `/dev`.** Two special files you will use constantly:

```bash
echo "this text disappears" > /dev/null
head -c 8 /dev/zero | od -An -tx1
```

```text
 00 00 00 00 00 00 00 00
```

**What happens:**

1. `/dev/null` is the "black hole": everything written to it is thrown away. It is used to silence output ([Lesson 11.3](03-text-tools-pipes-and-redirection.md)).
2. `/dev/zero` produces endless zero bytes. `head -c 8` takes the first 8 bytes, and `od -An -tx1` ("octal dump", no addresses, hexadecimal bytes) prints them. Others: `/dev/urandom` gives random bytes.

**Kernel information in `/proc`.** `/proc` is a **virtual file system**: nothing is stored on a disk, the kernel
makes the content when you read it.

```bash
head -3 /proc/meminfo
cat /proc/loadavg
cat /proc/self/comm
readlink /proc/self/cwd
```

```text
MemTotal:        8124192 kB
MemFree:          272820 kB
MemAvailable:    7099324 kB
0.00 0.10 0.09 1/459 2512
cat
/home/student
```

**What happens:**

1. `/proc/meminfo` describes memory. The command `free` that you used in Lesson 11.1 only reads it and formats it.
2. `/proc/loadavg` has the *load average*: how many processes wanted the CPU (Central Processing Unit, the processor) over the last 1, 5 and 15 minutes (the first three numbers), then the running/total process counts and the last PID.
3. `/proc/self` always means *the process that reads it*. Here that is `cat` itself, so `comm` (the command name) says `cat`. `readlink /proc/self/cwd` prints the working directory of `readlink` — your current directory.
4. Every running process has its own folder `/proc/<PID>/`. [Lesson 11.5](05-processes-and-signals.md) uses them.

> ⚠️ **Common mistake:** Trying to edit files in `/proc` with an editor or to "free space" by deleting them.
> They are not on a disk. Read them; write to them only if a guide says exactly what, because it changes the running kernel.

## 3. Paths, hidden files, spaces

You know absolute paths (start with `/`) and relative paths. Practise in a lab folder; all files stay in your
VM home, so you can delete everything at the end:

```bash
cd ~
mkdir lab
cd lab
mkdir -p projects/demo/src
touch Notes.txt notes.txt .hidden
ls
```

```text
Notes.txt
notes.txt
projects
```

**What happens:**

1. `mkdir -p projects/demo/src` creates a chain of directories at once. `-p` means "parents": missing directories on the way are created, and there is no error if they exist.
2. `touch` creates empty files. `Notes.txt` and `notes.txt` live side by side because names are case-sensitive.
3. `ls` does not show `.hidden`. Names that start with a dot are **hidden files**. The order of names depends on the locale (language settings); with the default `C.UTF-8` capital letters come first.

```bash
ls -a
```

```text
.
..
.hidden
Notes.txt
notes.txt
projects
```

**What happens:** `ls -a` ("all") shows hidden files and the two special entries: `.` is "this directory" and
`..` is "the parent directory". You meet hidden files every day: `~/.bashrc` (shell settings), `~/.cargo`
(Cargo), `~/.ssh` (keys, [Lesson 11.10](10-ssh-and-remote-servers.md)), `~/.config` (settings of many programs).

`~` is shorthand for your home directory, `cd -` goes back to the previous directory, and `cd` alone goes
home. A name with a **space** is one name, but the shell splits words at spaces. `rm my file.txt` makes `rm` look for two
files, `my` and `file.txt`, and fail with `No such file or directory`. Quote the name (`rm "my file.txt"`) or put a
backslash before the space (`my\ file.txt`).

> 📏 **Convention:** Avoid spaces and capital letters in names you create for programs: use
> `lowercase-with-dashes` or `snake_case`. Cargo does it, and it saves you from quoting.

## 4. Looking at a file: size, type, owner

`ls -l` shows the details of Lesson 00.4. Three more tools answer more questions:

```bash
cd ~/lab
echo "line one" > a.txt
stat a.txt
file a.txt /etc
```

```text
  File: a.txt
  Size: 9         	Blocks: 8          IO Block: 4096   regular file
Device: 0,69	Inode: 959145      Links: 1
Access: (0664/-rw-rw-r--)  Uid: ( 1000/ student)   Gid: ( 1000/ student)
Access: 2026-11-18 06:33:34.239697004 +0000
Modify: 2026-11-18 06:33:34.239697004 +0000
Change: 2026-11-18 06:33:34.239697004 +0000
 Birth: 2026-11-18 06:33:34.239697004 +0000
a.txt: ASCII text
/etc:  directory
```

**What happens:**

1. `stat` prints everything the file system knows about a file: the size in bytes (9 = `line one` plus the newline), the **inode** number, the number of links, the permissions as an octal number `0664` (Lesson 11.4), owner and group numbers, and four times: *Access* (last read), *Modify* (content changed), *Change* (metadata changed), *Birth* (created).
2. `file` looks at the **content** (not the name) and says what it is: text, an image, a program. Linux does not rely on `.txt` or `.rs` endings.

Sizes of many files, and free space of disks:

```bash
du -sh ~/lab
du -sh /usr/bin
df -h /
```

```text
20K	/home/student/lab
57M	/usr/bin
Filesystem      Size  Used Avail Use% Mounted on
overlay         453G   88G  343G  21% /
```

**What happens:** `du` ("disk usage") adds up the space of a directory tree; `-s` gives one total, `-h` uses
`K`, `M`, `G`. `df` ("disk free") shows each file system. Your numbers will differ. In your VM the `Filesystem` is a device such as
`/dev/vda1` (as in Lesson 11.1). When "no space left on device" appears, run `df -h` first (full disk?),
then `du -sh *` in the suspect directory (which folder is big?). Rust's `target/` folder is a classic answer.

> ⚡ **Performance:** `target/` of a Rust project can reach several GiB. Delete it with `cargo clean`
> when you need the space; the next build recreates it.

## 5. Hard links and symbolic links

A file name is only a **label** that points to an **inode**. A *hard link* is a second label for the same inode.
A *symbolic link* is a different thing: a small file that stores a path.

```text
  hard link                           symbolic link
  a.txt ---+                          soft.txt --> "a.txt" (a path in text)
           +--> inode 959145 -> data                |
  hard.txt-+                                        v
                                      a.txt --> inode 959145 -> data
```

Try both:

```bash
cd ~/lab
ln a.txt hard.txt
ln -s a.txt soft.txt
ls -li a.txt hard.txt soft.txt
```

```text
959145 -rw-rw-r-- 2 student student 9 Nov 18 06:33 a.txt
959145 -rw-rw-r-- 2 student student 9 Nov 18 06:33 hard.txt
959142 lrwxrwxrwx 1 student student 5 Nov 18 06:33 soft.txt -> a.txt
```

**What happens:**

1. `ln a.txt hard.txt` creates a hard link. `ls -i` prints the inode number first: `a.txt` and `hard.txt` have the **same** inode, and the link count (the number after the permissions) is 2.
2. `ln -s a.txt soft.txt` creates a symbolic link (`-s`). It has its own inode and the type `l`; `ls -l` shows `-> a.txt`, the path stored inside.

Now remove the original name and see the difference:

```bash
echo "line two" >> hard.txt
cat a.txt
rm a.txt
cat hard.txt
cat soft.txt
```

```text
line one
line two
line one
line two
cat: soft.txt: No such file or directory
```

**What happens:**

1. Writing through `hard.txt` changed the same data, so `cat a.txt` sees both lines.
2. `rm a.txt` removes one *name*. The data stays while at least one name remains: `hard.txt` still works.
3. `soft.txt` points to the path `a.txt`, which no longer exists: a **broken link**. `cat` follows the path and fails.

Symbolic links are far more common. Look at the `ls -ld` output in Section 1 again: `/bin -> usr/bin`. `readlink -f /bin` prints the
final real target: `/usr/bin`. Use symlinks to give a stable name to something that changes, e.g.
`current -> release-2026-11-18`. Clean up:

```bash
rm soft.txt hard.txt
```

## 6. Copy, move, delete — carefully

You know `cp`, `mv`, `rm`. The traps appear with directories:

```bash
cd ~/lab
mkdir -p projects/demo/src projects/other
echo "fn main() {}" > projects/demo/src/main.rs
cp -r projects projects-copy
ls projects-copy
cp -r projects projects-copy
ls projects-copy
```

```text
demo
other
demo
other
projects
```

**What happens:**

1. `cp -r` ("recursive") copies a directory with everything inside. Without `-r`, `cp` refuses to copy a directory.
2. The first copy created `projects-copy` as a copy of `projects`. The **second** time `projects-copy` already existed, so `cp` put the source *inside* it: `projects-copy/projects`. The behaviour depends on whether the target exists. Check with `ls` before you copy.
3. `cp -a` ("archive") also keeps permissions and dates; it is the right choice for backups.

`mv` works on directories without `-r`. `rmdir` removes only **empty** directories, which is a safety net
(`rmdir: failed to remove 'projects/other': Directory not empty`). `rm -r` removes a directory with everything
inside, and `rm -f` ("force") hides "no such file" errors and never asks.

> 🔒 **Security:** There is **no trash can** on the command line. `rm` deletes at once and for good.
> - `rm -rf` plus a mistake is how people lose their work. A classic: `rm -rf $DIR/` when the variable `DIR` is empty becomes `rm -rf /`.
> - Run `ls` with the same pattern *first*, then `rm`. Use `rm -i` ("interactive") to be asked for each file when you are unsure.
> - Quote variables (`"$DIR"`) and avoid `sudo rm -rf` unless you can say out loud what it deletes.

## 7. Finding files: `find`, `which`, `type`

`find` walks a directory tree and prints the paths that match tests. Prepare some files and try it:

```bash
cd ~/lab
echo "[package]" > projects/demo/Cargo.toml
head -c 3000 /dev/zero > big.bin
find . -name "*.rs"
find . -type d
find . -type f -size +1k
find . -name "*.md" -o -name "*.toml"
```

```text
./projects/demo/src/main.rs
.
./projects
./projects/other
./projects/demo
./projects/demo/src
./big.bin
./projects/other/notes.md
./projects/demo/Cargo.toml
```

**What happens:**

1. `find . -name "*.rs"`: start in `.` (here) and print every path whose name matches. Quote the pattern, so that the shell does not expand `*` first ([Lesson 00.4](../00-getting-started/04-terminal-basics.md)). The pattern is case-sensitive; `-iname` ignores case.
2. `-type d` means directories, `-type f` regular files, `-type l` symbolic links.
3. `-size +1k` matches files bigger than 1 KiB (`-size -1k` smaller).
4. `-o` means *or*. Two tests next to each other mean *and*.

Other useful tests: `-maxdepth 1` (do not go deeper), `-mtime -1` (changed in the last day), `-newer file`
(changed more recently than `file`). `find` can also run a command on every match:

```bash
find . -name "*.rs" -exec wc -l {} \;
```

```text
1 ./projects/demo/src/main.rs
```

**What happens:** for each match, `find` replaces `{}` with the path and runs `wc -l` (count lines). The
`\;` ends the command; the backslash stops the shell from using the `;` itself. In the next lesson you
will meet `xargs`, the pipe-based way to do the same.

To find *programs*, ask the shell:

```bash
type cd
type bash
command -v bash
```

```text
cd is a shell builtin
bash is /usr/bin/bash
/usr/bin/bash
```

**What happens:** `type` says how the shell would run a name: a **builtin** (part of the shell itself, like `cd`),
a program file at a given path, an alias or a function. `which` and `command -v` print the path. These are the
tools to answer "which `cargo` am I running?" once you have more than one.

## 8. Where programs keep their things

When something does not work, you need to know where to look. Rules of thumb for Ubuntu:

| What | Where | Example |
|---|---|---|
| System-wide settings | `/etc/<program>/` or `/etc/<program>.conf` | `/etc/ssh/sshd_config` |
| Your own settings | `~/.config/<program>/`, or a dotfile `~/.<program>` | `~/.config/git/config`, `~/.bashrc` |
| Programs of the system | `/usr/bin` | `/usr/bin/ls` |
| Programs you install yourself | `/usr/local/bin`, or `~/.local/bin`, or `~/.cargo/bin` | `~/.cargo/bin/cargo` |
| Changing data of services | `/var/lib/<program>/` | databases |
| Logs | `/var/log/` (and the *journal*, Lesson 11.6) | `/var/log/dpkg.log` |
| Caches | `/var/cache`, `~/.cache` | |
| Runtime sockets, PID files | `/run` | |

Two rules follow from this table. **Configuration is text**, so you can read, search and copy it. And
**code and data are separated**: the program is in `/usr/bin`, its data in `/var/lib`, so you can update the
program without touching the data. Rust programs follow the same layout when you deploy them in [Lesson 11.6](06-services-with-systemd.md).

### Bring a Rust project from your Mac into the VM

Your Mac files are visible at `/Users/<your-mac-name>/` but read-only. To build a project in Linux, copy it
into the VM home (the example uses the name `student`; use your own, `ls /Users` shows it):

```bash
mkdir -p ~/code
cp -r /Users/student/projects/rust/code/p11-heartbeat ~/code/
```

If the source does not exist yet, create any folder on the Mac first (Lesson 11.5 creates `p11-heartbeat`).
To bring a result **back** to the Mac, remember that the mount is read-only: copy it with `scp`, or use
`limactl copy`. Check the exact syntax with `limactl copy --help`; you will learn `scp` in Lesson 11.10.

Remove the lab of Sections 3 to 7 when you are done:

```bash
rm -r ~/lab
```

## Common mistakes

1. **Using Mac habits for case.** You created `Cargo.toml` and typed `cargo.toml`: `No such file or directory`. *Fix:* names are case-sensitive; check with `ls`.
2. **`cp -r` into an existing directory** gives `target/source/...` instead of a copy. *Fix:* check if the target exists; use `cp -a source/. target/` to copy the *contents*.
3. **Unquoted names with spaces** make the command see several arguments. *Fix:* quote them, or avoid spaces.
4. **Deleting the wrong thing with `rm -rf`.** *Fix:* run `ls` with the same arguments first; do not use `-f` out of habit.
5. **Following a broken symlink.** `cat link` says `No such file or directory`, although `ls -l` shows the link. *Fix:* `readlink link` shows the path stored in the link; check that this target exists.
6. **Trying to change Mac files from the VM.** `Read-only file system`. *Fix:* work in `~` of the VM.

## Try it yourself

Do this from a clean `~`:

1. `mkdir -p ~/lab/a/b/c` and `cd ~/lab/a/b/c`. Print where you are with `pwd`, then go up two levels with `cd ../..` and `pwd` again.
2. Create `~/lab/readme.txt` with some text. Make a symbolic link `~/lab/a/link.txt` to it, using an **absolute** path. Read the text through the link.
3. Run `find ~/lab -type f -o -type l` and check that you see both.
4. Run `du -sh ~/lab` and `df -h ~`.
5. Delete the lab: `rm -r ~/lab`, and check with `ls ~`.

## Exercises

### Exercise 1 — Explore the tree (easy)

Without a guide, find out what is inside `/etc/hostname`, which directory holds your login's settings, and
which file lists all users of the system. Use `cat`, `ls -a ~` and the table in Section 1.

<details>
<summary>Hint</summary>

Users are listed in a text file in `/etc` whose name starts with `pass`. The settings of your shell are hidden files in `~`.

</details>

<details>
<summary>Answer</summary>

```bash
cat /etc/hostname
ls -a ~
cat /etc/passwd
```

`/etc/hostname` holds the machine name (`lima-dev`). Hidden files such as `.bashrc` and `.profile` in `~`
hold your shell settings. `/etc/passwd` lists the users (you will read it in Lesson 11.4).

Why it works: configuration is plain text in `/etc`, user-specific settings are dotfiles in the home directory.

</details>

### Exercise 2 — A hard link survives (medium)

Create `x.txt`, a hard link `y.txt` and a symlink `z.txt` to `x.txt`. Delete `x.txt`. Which of `y.txt` and
`z.txt` can you still read, and why? Use `ls -li` to support your answer.

<details>
<summary>Hint</summary>

Compare the inode numbers and the link counts before the delete.

</details>

<details>
<summary>Answer</summary>

```bash
cd ~
echo data > x.txt
ln x.txt y.txt
ln -s x.txt z.txt
ls -li x.txt y.txt z.txt
rm x.txt
cat y.txt
cat z.txt
rm y.txt z.txt
```

`y.txt` is still readable (`data`); `z.txt` gives `No such file or directory`.

Why it works: `x.txt` and `y.txt` are two names of one inode, and the data stays while a name remains. `z.txt`
only stores the text `x.txt`, a path that no longer leads anywhere.

</details>

### Exercise 3 — Write a check for a directory layout (medium)

A **test** is a command that answers yes or no through its exit status. The command `[ -d path ]` succeeds if `path`
is a directory; `[ -f path ]` if it is a regular file; `[ -L path ]` if it is a symbolic link. Write three checks:
`/etc` is a directory; `/bin` is a symbolic link; `/etc/hostname` is a regular file. Print `ok` or `FAIL` for each with `&&` and `||`.

<details>
<summary>Hint</summary>

`cmd1 && echo ok || echo FAIL` prints `ok` when `cmd1` succeeds and `FAIL` otherwise. Mind the spaces inside `[ ... ]`.

</details>

<details>
<summary>Answer</summary>

```bash
[ -d /etc ] && echo ok || echo FAIL
[ -L /bin ] && echo ok || echo FAIL
[ -f /etc/hostname ] && echo ok || echo FAIL
[ -f /etc ] && echo ok || echo FAIL
```

```text
ok
ok
ok
FAIL
```

The last line is a deliberate failure: `/etc` is a directory, not a regular file.

Why it works: `[` is a command of bash (see `type [`) that tests a condition and sets the exit status; `&&` runs the next command only
after success, `||` only after failure. Lesson 11.8 builds scripts from these pieces.

</details>

## Quiz

1. How many "roots" does the Linux file system have, and how do you attach a second disk?

<details>
<summary>Answer</summary>

One root, `/`. A second disk is *mounted* at a directory in the tree.

</details>

2. Where do you look for the system configuration of a program? For its logs?

<details>
<summary>Answer</summary>

Configuration: `/etc`. Logs: `/var/log` (and the journal, Lesson 11.6).

</details>

3. What is the difference between a hard link and a symbolic link?

<details>
<summary>Answer</summary>

A hard link is another name for the same inode, so the data lives as long as one name remains. A symbolic link is a small file with a path; it breaks if the target disappears.

</details>

4. What does `/dev/null` do?

<details>
<summary>Answer</summary>

It discards everything written to it, and reads as empty. It is used to hide output.

</details>

5. Why does `rm my file.txt` fail for a file called `my file.txt`?

<details>
<summary>Answer</summary>

The shell splits at spaces, so `rm` received two names (`my` and `file.txt`). Quote the name: `rm "my file.txt"`.

</details>

## Summary

- The file system is one tree starting at `/`. `/etc` holds configuration, `/var` changing data and logs, `/usr` programs, `/home` user files, `/tmp` temporary files.
- Names are case-sensitive; a dot at the start hides a file; quote names with spaces.
- Everything is a file: `/dev/null` swallows data, `/proc` shows live kernel information.
- `stat`, `file`, `du`, `df` tell you about files, content, sizes and free space.
- A hard link is another name for the same inode; a symbolic link stores a path and can break.
- `cp -r`, `mv`, `rm -r` need care; there is no trash can. `find` with `-name`, `-type`, `-size` finds files.
- Your Mac files are read-only in the VM; copy a project into `~` to build it there.

## Sources and further reading

- [Filesystem Hierarchy Standard 3.0](https://refspecs.linuxfoundation.org/FHS_3.0/fhs-3.0.html) — the official description of every top-level directory.
- [Ubuntu — man page of find](https://manpages.ubuntu.com/manpages/noble/man1/find.1.html) — all tests and actions of `find`.
- [proc(5) — the /proc file system](https://man7.org/linux/man-pages/man5/proc.5.html) — what the files in `/proc` contain.
- [inode(7)](https://man7.org/linux/man-pages/man7/inode.7.html) — what an inode stores.
- [uutils coreutils](https://uutils.github.io/coreutils/docs/) — the Rust implementation of the basic commands used by Ubuntu 26.04.

---

[🏠 Start](../START-HERE.md) · [← 11.1 Linux and a virtual machine on your Mac](01-linux-and-a-virtual-machine.md) · [📋 Phase 11](README.md) · [11.3 Text tools, pipes and redirection →](03-text-tools-pipes-and-redirection.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
