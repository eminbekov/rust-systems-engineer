<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 11.6 Services with systemd](06-services-with-systemd.md) · [📋 Phase 11](README.md) · [11.8 Bash scripting basics →](08-bash-scripting-basics.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/11-linux-shell-and-containers/07-package-managers.md) <!-- nav -->

# Lesson 11.7 — Package managers

> **Phase 11 — Linux, the shell and containers** · Lesson 7 of 16 · ⏱ about 120 minutes

## What you will learn

- What a package, a repository and a package manager are, and why you do not download programs by hand on Linux.
- How to find, install, update and remove software with `apt`, and how to see which files a package installed.
- Why packages are signed, how repositories are configured, and what is risky about adding a third-party one.
- How to turn your own Rust program into a `.deb` package, and when to use `cargo install` instead.

## Before you start

- You finished [Lesson 11.6 — Services with systemd](06-services-with-systemd.md).
- You used `apt-get install` twice already (Lessons 11.2 and 11.5) with only a short explanation. Now you learn what it does.
- All commands run in the VM (`limactl shell dev`). Section 6 uses the Rust you installed in Lesson 11.5.

## New words

| Word | Meaning |
|---|---|
| **package** | One piece of software packed as an archive: the files, plus metadata (name, version, description) and a list of the other packages it needs. |
| **package manager** | A program that installs, updates and removes packages and keeps a database of what is installed. |
| **repository** (repo) | A server with many packages and an index file that lists them. |
| **dependency** | A package that another package needs in order to work. The package manager installs dependencies automatically. |
| **apt** (Advanced Package Tool) | The package manager of Debian and Ubuntu. The command `apt-get` is its older, script-friendly form. |
| **dpkg** (Debian package) | The low-level tool that unpacks one `.deb` file and records it in the database. `apt` calls `dpkg`. |
| **.deb** | The file format of a Debian/Ubuntu package. |
| **package index** | The list of packages and versions of a repository that `apt` downloads with `apt update`. |
| **GPG** (GNU Privacy Guard) signature | A cryptographic signature. The repository signs its index; `apt` checks the signature with a public key it already has. |
| **keyring** | A file with the public keys that `apt` trusts for a repository. |
| **suite / component** | A suite is a release line of the distribution (`resolute`, `resolute-updates`, `resolute-security`). A component is a part of it: `main`, `universe`, `restricted`, `multiverse`. |
| **PPA** (Personal Package Archive) | A repository run by one person or team on Launchpad, outside the official Ubuntu archive. |
| **version** | The text that identifies a build of a package, like `1.8.1-4ubuntu2`: upstream version, then the distribution's revision. |

## 1. Why a package manager

You could download a program from a website, unpack it and copy it somewhere. For a complete system this does not work: who updates it? What does it need? Where are its files? Which version is it? A **package manager** answers these questions:

```text
      repository (a server)                 your machine
  +-------------------------+        +--------------------------+
  |  index: names, versions |  apt   |  database of installed   |
  |  jq_1.8.1.deb           | -----> |  packages (dpkg)         |
  |  libjq1_1.8.1.deb ...   |        |  files in /usr, /etc ... |
  +-------------------------+        +--------------------------+
        signed by the distribution            checks the signature
```

- A package knows its **dependencies**. Install `jq` and `apt` also installs the libraries it needs.
- Packages are **signed**, so a changed or fake package is rejected.
- Security fixes arrive through the same channel, and one command updates the whole system.
- Every file belongs to a package, so you can **remove** software cleanly.

You already know a package manager: **Cargo**. A *crate* is its package, *crates.io* is the repository, `Cargo.lock` records the exact versions. On your Mac, **Homebrew** (Lesson 11.1) does the same for command-line tools. On Ubuntu the system package manager is `apt`.

## 2. Packages and repositories

Ubuntu and Debian packages are `.deb` files. **`dpkg`** is the tool that installs a single `.deb`; it knows nothing about the internet or about dependencies. **`apt`** is the layer above: it reads the repository list, downloads packages, works out the dependencies and calls `dpkg`. In daily work you use `apt`.

The repositories are configured in `/etc/apt/sources.list.d/`. On Ubuntu 26.04 the file is `ubuntu.sources` (in the *deb822* format, a list of `Key: value` lines). Without the comment lines:

```bash
grep -v '^#' /etc/apt/sources.list.d/ubuntu.sources
```

```text
Types: deb
URIs: http://archive.ubuntu.com/ubuntu/
Suites: resolute resolute-updates resolute-backports
Components: main universe restricted multiverse
Signed-By: /usr/share/keyrings/ubuntu-archive-keyring.gpg

Types: deb
URIs: http://security.ubuntu.com/ubuntu/
Suites: resolute-security
Components: main universe restricted multiverse
Signed-By: /usr/share/keyrings/ubuntu-archive-keyring.gpg
```

**What happens:** the address (`URIs`) may differ on your VM (for example `ports.ubuntu.com` for arm64). Read the fields:

| Field | Meaning |
|---|---|
| `Types: deb` | binary packages (`deb-src` would be source code) |
| `URIs` | the server |
| `Suites` | `resolute` is Ubuntu 26.04 as released; `-updates` has later bug fixes; `-security` has security fixes; `-backports` has newer versions of some software |
| `Components` | `main` is software supported by Canonical; `universe` is community-maintained; `restricted` and `multiverse` have non-free parts |
| `Signed-By` | the keyring file with the public key used to verify this repository **only** |

> 🔒 **Security:** `apt` will not install packages from a repository whose index is not signed with the key named in `Signed-By`. This is what stops somebody on the network (or a hacked mirror) from handing you a modified `sudo`. We come back to this in Section 5.

## 3. Everyday `apt`

The package index is a local copy. Refresh it before you install anything:

```bash
sudo apt-get update
```

```text
Hit:4 http://archive.ubuntu.com/ubuntu resolute-backports InRelease
Get:5 http://archive.ubuntu.com/ubuntu resolute-updates/main arm64 Packages [1121 kB]
Fetched 1258 kB in 3s (470 kB/s)
Reading package lists...
```

(Only the last lines are shown; the first lines list every repository: `Hit` means "no change", `Get` means "downloaded".) Now look for a package, read about it and check which version you would get:

```bash
apt search '^jq$'
apt show jq | head -n 8
apt-cache policy jq
```

```text
Sorting...
Full Text Search...
jq/resolute-updates,resolute-security 1.8.1-4ubuntu2 arm64
  lightweight and flexible command-line JSON processor

Package: jq
Version: 1.8.1-4ubuntu2
Priority: optional
Section: utils
Origin: Ubuntu
Maintainer: Ubuntu Developers <ubuntu-devel-discuss@lists.ubuntu.com>
Original-Maintainer: ChangZhuo Chen (陳昌倬) <czchen@debian.org>
Bugs: https://bugs.launchpad.net/ubuntu/+filebug
jq:
  Installed: (none)
  Candidate: 1.8.1-4ubuntu2
  Version table:
     1.8.1-4ubuntu2 500
        500 http://archive.ubuntu.com/ubuntu resolute-updates/main arm64 Packages
        500 http://security.ubuntu.com/ubuntu resolute-security/main arm64 Packages
     1.8.1-4ubuntu1 500
        500 http://archive.ubuntu.com/ubuntu resolute/main arm64 Packages
```

**What happens:**

1. `apt search` searches names and descriptions; the pattern `^jq$` is a regular expression for the exact name ([Lesson 11.3](03-text-tools-pipes-and-redirection.md)). The text after the name tells in which suites it is.
2. `apt show` prints the metadata: version, size, dependencies (`Depends:`), the homepage.
3. `apt-cache policy` shows what is installed, what the **candidate** is (the version `apt` would install) and from which repositories each version is available. The number `500` is the priority.

Install it. The option `-s` ("simulate") shows what would happen without doing it:

```bash
sudo apt-get install -s jq | grep -E 'NEW|additional|newly'
sudo apt-get install -y jq
jq --version
echo '{"name":"course","n":3}' | jq .name
```

```text
The following additional packages will be installed:
The following NEW packages will be installed:
0 upgraded, 3 newly installed, 0 to remove and 0 not upgraded.
...
Setting up libonig5:arm64 (6.9.10-1build1) ...
Setting up libjq1:arm64 (1.8.1-4ubuntu2) ...
Setting up jq (1.8.1-4ubuntu2) ...
jq-1.8.1
"course"
```

**What happens:**

1. The simulation reports "3 newly installed": you asked for `jq`, and its two libraries are the **dependencies**.
2. `-y` answers "yes" to the question "Do you want to continue?". Without it `apt` waits for you. Scripts always need `-y`.
3. The steps are: download the three `.deb` files (`Get:`), unpack them (`Unpacking`) and configure them (`Setting up`). The long output of the real command is shortened above.
4. `jq` is a JSON (JavaScript Object Notation) filter: it printed the field `name`.

`apt` and `apt-get` do the same work. Use `apt` when you type (it has a nicer display and a progress bar), and `apt-get` in scripts: its output and options are stable, `apt` itself warns *"apt does not have a stable CLI interface. Use with caution in scripts."*

The commands you need, in one table:

| Command | What it does |
|---|---|
| `sudo apt-get update` | refresh the package index |
| `apt list --upgradable` | show packages with a newer version |
| `sudo apt-get upgrade` | install newer versions of installed packages (never removes anything) |
| `sudo apt-get full-upgrade` | like `upgrade`, but may also install or remove packages to resolve changed dependencies |
| `sudo apt-get install PKG` | install a package (and its dependencies) |
| `sudo apt-get remove PKG` | remove the program; keep its configuration files |
| `sudo apt-get purge PKG` | remove the program **and** its configuration files |
| `sudo apt-get autoremove` | remove dependencies that nothing needs any more |
| `apt-mark showmanual` | packages you installed yourself (the rest came as dependencies) |

Remove `jq` and its leftover libraries:

```bash
sudo apt-get remove -y jq
sudo apt-get autoremove -y
```

```text
Removing jq (1.8.1-4ubuntu2) ...
Removing libjq1:arm64 (1.8.1-4ubuntu2) ...
Removing libonig5:arm64 (6.9.10-1build1) ...
```

**What happens:** `remove` took `jq`; `autoremove` then found that `libjq1` and `libonig5` were installed only as its dependencies, and removed them too. A package that has configuration files in `/etc` keeps them after `remove` (the state `rc` in `dpkg -l`) until you `purge` it. That is why a reinstall often "remembers" your settings.

> 💡 **Tip:** Update and upgrade regularly on a real server: `sudo apt-get update && sudo apt-get upgrade`. Ubuntu can install security updates by itself with the package `unattended-upgrades`. In the VM you are free to try.

## 4. What did the package install?

Because the database knows every file, you can ask it questions:

```bash
sudo apt-get install -y jq > /dev/null
dpkg -L jq | head -n 5
dpkg -S /usr/bin/jq
dpkg -l jq | tail -n 1
```

```text
/.
/usr
/usr/bin
/usr/bin/jq
/usr/share
jq: /usr/bin/jq
ii  jq             1.8.1-4ubuntu2 arm64        lightweight and flexible command-line JSON processor
```

**What happens:**

1. `dpkg -L PKG` lists the files **of** a package.
2. `dpkg -S PATH` is the reverse: which package **owns** a file. Use it when you find an unknown file in `/usr/bin` or `/etc`.
3. `dpkg -l PKG` prints the status. `ii` means "wanted: install, state: installed". `rc` means removed, configuration left.

The paths follow the layout of [Lesson 11.2](02-the-linux-file-system.md): the program is in `/usr/bin`, the documentation in `/usr/share/doc`. Remember `/bin` is a symlink to `/usr/bin`; `dpkg -S` knows only the real path:

```bash
dpkg -S /usr/bin/ls
dpkg -S /usr/bin/ps
```

```text
coreutils-from-uutils: /usr/bin/ls
procps: /usr/bin/ps
```

**What happens:** on Ubuntu 26.04 the file `/usr/bin/ls` belongs to the package `coreutils-from-uutils`, which links to the Rust implementation (Lesson 11.2). Use `dpkg -S $(which ps)` to find the package of a command.

## 5. Trust: signatures and third-party repositories

Installing a package means running code from its author **as root** (the installer scripts of a package run as root). So you must trust whoever signs it.

- The Ubuntu archive is signed by Ubuntu's key. `apt` checks the signature of the index (`InRelease`) and the index lists the checksum of every package, so a package cannot be changed unnoticed.
- A **third-party repository** (a vendor's, or a PPA) has its own key. Adding it gives that vendor the same power as Ubuntu itself over your machine: every `apt upgrade` will install what they publish, as root.

If you must add one, the safe pattern is the same as in the Ubuntu file you saw: download the vendor's public key into its **own keyring file**, and bind the repository to that key with `Signed-By`, so the key is trusted only for that repository. This is the pattern (it is **not** a real repository, do not run it):

```text
# /etc/apt/sources.list.d/example-vendor.sources
Types: deb
URIs: https://packages.example.com/ubuntu
Suites: stable
Components: main
Signed-By: /etc/apt/keyrings/example-vendor.gpg
```

> 🔒 **Security:** Before you add a repository, ask: who runs it, why do I trust them, and do I need it? Prefer the Ubuntu archive. Take the key from the vendor's official page over HTTPS (the encrypted web protocol, Lesson 10.11). Never put a key in the global `trusted.gpg.d` folder (it would be trusted for *every* repository) and never copy-paste a `curl ... | sudo bash` command you have not read ([Lesson 00.4](../00-getting-started/04-terminal-basics.md)). When you remove the software, remove the `.sources` and the key as well.

## 6. Packaging your own program

A Rust program can also be distributed as a `.deb`. Then `apt` installs it, `dpkg -S` shows its owner, and `apt remove` removes it. The tool **cargo-deb** builds the package from `Cargo.toml`. Install it in the VM (Cargo from Lesson 11.5; `--locked` uses the dependency versions that the author tested, as in [Phase 10](../10-security-fundamentals/README.md)):

```bash
cargo install --locked cargo-deb --version 3.8.0
```

```text
   Compiling ar v0.9.0
   Compiling cargo_toml v1.0.1
   Compiling xz2 v0.1.7
    Finished `release` profile [optimized] target(s) in 16.21s
  Installing /home/student/.cargo/bin/cargo-deb
   Installed package `cargo-deb v3.8.0` (executable `cargo-deb`)
```

(The output is shortened; compiling takes a minute.) Open the copy of the project in the VM, `~/code/p11-heartbeat/Cargo.toml`, and add two lines to `[package]`, and a new section at the end:

```toml
[package]
name = "p11-heartbeat"
version = "0.1.0"
edition = "2024"
description = "A tiny demo program that prints a heartbeat line"
license = "MIT"

[dependencies]
signal-hook = "0.4"

[package.metadata.deb]
maintainer = "Course Student <student@example.com>"
copyright = "2026, Course Student"
extended-description = "A tiny demo program from the Rust systems course that prints a heartbeat line."
section = "utils"
priority = "optional"
assets = [
    ["target/release/p11-heartbeat", "usr/bin/", "755"],
]
```

**What happens:** `description` and `license` are standard Cargo fields (cargo-deb warns when they are missing). The section `[package.metadata.deb]` is read only by cargo-deb. `assets` lists the files to put in the package: *source path*, *destination folder inside the package* and the *mode*. The maintainer line is a placeholder; use your own name and address for a real package. Build, inspect and install:

```bash
cd ~/code/p11-heartbeat
cargo build --release
cargo deb
dpkg-deb --contents target/debian/p11-heartbeat_0.1.0-1_arm64.deb
sudo apt-get install -y ./target/debian/p11-heartbeat_0.1.0-1_arm64.deb
dpkg -S /usr/bin/p11-heartbeat
timeout 3 p11-heartbeat; echo "status: $?"
```

```text
/home/student/code/p11-heartbeat/target/debian/p11-heartbeat_0.1.0-1_arm64.deb
drwxr-xr-x 0/0               0 2026-11-18 00:00 ./usr/
drwxr-xr-x 0/0               0 2026-11-18 00:00 ./usr/share/
drwxr-xr-x 0/0               0 2026-11-18 00:00 ./usr/share/doc/
drwxr-xr-x 0/0               0 2026-11-18 00:00 ./usr/share/doc/p11-heartbeat/
-rw-r--r-- 0/0             149 2026-11-18 00:00 ./usr/share/doc/p11-heartbeat/copyright
drwxr-xr-x 0/0               0 2026-11-18 00:00 ./usr/bin/
-rwxr-xr-x 0/0          398120 2026-11-18 00:00 ./usr/bin/p11-heartbeat
Setting up p11-heartbeat (0.1.0-1) ...
p11-heartbeat: /usr/bin/p11-heartbeat
starting: pid 7179, greeting world, every 2 s
heartbeat 1 from world
heartbeat 2 from world
got signal 15: shutting down
bye
status: 124
```

**What happens:**

1. `cargo deb` printed the path of the new package. `dpkg-deb --contents` lists what is inside: the program and a copyright file. (cargo-deb also found the dependency `libc6` automatically; `dpkg-deb --info FILE` shows the metadata.)
2. `apt-get install ./file.deb` (the `./` is required, it tells `apt` this is a file) installs a local package and its dependencies.
3. `dpkg -S` shows that the program now belongs to the package `p11-heartbeat`.
4. `timeout 3 PROGRAM` runs a program and stops it after 3 seconds by sending `SIGTERM`. Our Rust program handled it, printed `bye`, and `timeout` reported its own status `124` ("the time limit was reached"). It is the signal handling from [Lesson 11.5](05-processes-and-signals.md) at work.

Remove the package when you finish: `sudo apt-get remove -y p11-heartbeat`.

**Other package managers.** `cargo install --locked NAME` downloads a *Rust* program from crates.io, compiles it and puts it in `~/.cargo/bin` for your user. Use it for developer tools (`cargo-deb`, `ripgrep`). Use `apt` for system software and services. Keep the two apart: a program installed by `apt` lives in `/usr/bin` and is updated with the system; a program from `cargo install` lives in your home and is updated by you. `which NAME` tells you which one you run. Ubuntu also has `snap` packages, which bring their own libraries in a sandbox; they are managed by `snap`, not by `apt`.

## Common mistakes

1. **`E: Unable to locate package NAME`.** The index is old, the name is wrong, or the component is not enabled. *Fix:* `sudo apt-get update`, then `apt search NAME`.
2. **`Could not get lock /var/lib/dpkg/lock-frontend` / `Permission denied`.** Another `apt` is running (often an automatic update), or you forgot `sudo`. *Fix:* wait and retry; never delete lock files by hand.
3. **Forgetting `-y` in a script.** The script hangs on "Do you want to continue? [Y/n]". *Fix:* `sudo apt-get install -y ...`; in automation also `DEBIAN_FRONTEND=noninteractive`, which stops configuration questions.
4. **`apt install ./file.deb` without `./`.** `apt` looks for a package named `file.deb` in the repository. *Fix:* give a path.
5. **`sudo apt-get upgrade` without `update` first.** It upgrades from an old index. *Fix:* run `update` first.
6. **Adding a repository key to `trusted.gpg.d`.** It is now trusted for every repository. *Fix:* a keyring file per repository and `Signed-By`.
7. **Mixing sources of the same tool.** `cargo` from `apt` and from `rustup` at the same time: you run a different version than you think. *Fix:* check with `which -a cargo`; keep one.

## Try it yourself

1. Run `sudo apt-get update`. How many repositories (lines starting with `Hit:` or `Get:`) does it contact?
2. Search for the package `htop` with `apt search '^htop$'`, show its dependencies with `apt-cache depends htop`, then install it with `sudo apt-get install -y htop`. Run `htop`, press `q`, and find its files with `dpkg -L htop | grep bin`.
3. Use `dpkg -S $(readlink -f $(which sudo))` — what package does `sudo` come from? (`which sudo` gives a symlink; `readlink -f` follows it to the real file, Lesson 11.2.)
4. Remove `htop` with `remove`, then check `dpkg -l htop`. Then run `autoremove`.
5. Section 6: build and install the `.deb`, run it with `timeout 3`, and remove it.

## Exercises

### Exercise 1 — Who owns this file? (easy)

Find out which package provides the command `ps`, and which package owns the file `/etc/hostname`. Explain the second answer.

<details>
<summary>Hint</summary>

`dpkg -S` takes a path; use `which ps` or `command -v ps` to find it. A file created at installation by a script, not shipped in a package, has no owner.

</details>

<details>
<summary>Answer</summary>

```bash
dpkg -S $(which ps)
dpkg -S /etc/hostname
```

```text
procps: /usr/bin/ps
dpkg-query: no path found matching pattern /etc/hostname
```

Why it works: `ps` is part of the package `procps`. `/etc/hostname` is written when the system is set up (it belongs to this machine, not to a package), so no package owns it.

</details>

### Exercise 2 — Preview before you change (medium)

Using only simulation (nothing may be installed), find out how many new packages `apt` would install for `htop` and their names.

<details>
<summary>Hint</summary>

Use `-s`. The simulation prints one line `Inst NAME (...)` for each package it would install, and a summary line with `newly installed`.

</details>

<details>
<summary>Answer</summary>

```bash
apt-get install -s htop | grep -E '^Inst|newly'
```

```text
Inst htop (3.4.1-5build2 Ubuntu:26.04/resolute [arm64])
0 upgraded, 1 newly installed, 0 to remove and 0 not upgraded.
```

(The versions and the number of packages depend on what is installed on your VM, and `htop` must not be installed yet.)

Why it works: `-s` (simulate) resolves the dependencies and prints the plan without changing the system. Always read the plan of a big upgrade before you confirm it.

</details>

### Exercise 3 — Write an installation check (medium)

Write a one-line **check** that exits with status 0 if a package is installed and with a non-zero status if it is not, printing nothing. Test it with `jq` before and after installing it.

<details>
<summary>Hint</summary>

`dpkg -s PKG` returns non-zero for a package that is not installed. Hide its output with redirections ([Lesson 11.3](03-text-tools-pipes-and-redirection.md)).

</details>

<details>
<summary>Answer</summary>

```bash
sudo apt-get remove -y -qq jq > /dev/null
dpkg -s jq > /dev/null 2>&1; echo "installed? $?"
sudo apt-get install -y -qq jq > /dev/null
dpkg -s jq > /dev/null 2>&1; echo "installed? $?"
```

```text
installed? 1
installed? 0
```

Why it works: the exit status is the answer (0 = yes), and `> /dev/null 2>&1` discards both output streams. Scripts use this to install a package only when it is missing ([Lesson 11.8](08-bash-scripting-basics.md)).

</details>

### Exercise 4 — Test the package (hard)

Write a **package test** for `p11-heartbeat_0.1.0-1_arm64.deb`: a short shell sequence that exits with status 0 only if the package contains `./usr/bin/p11-heartbeat`, prints `PASS` or `FAIL`, and does not install anything.

<details>
<summary>Hint</summary>

`dpkg-deb --contents FILE` lists the files. Search the list with `grep -q`.

</details>

<details>
<summary>Answer</summary>

```bash
DEB=~/code/p11-heartbeat/target/debian/p11-heartbeat_0.1.0-1_arm64.deb
dpkg-deb --contents "$DEB" | grep -q ' \./usr/bin/p11-heartbeat$' && echo PASS || echo FAIL
dpkg-deb --contents "$DEB" | grep -q ' \./usr/bin/missing$' && echo PASS || echo FAIL
```

```text
PASS
FAIL
```

Why it works: the first check finds the path at the end of a line in the contents list; the second is a deliberate failure to prove that the test can fail. A test that cannot fail proves nothing.

</details>

## Quiz

1. What is the difference between `dpkg` and `apt`?

<details>
<summary>Answer</summary>

`dpkg` installs one `.deb` file and records it, with no repositories and no dependency solving. `apt` reads repositories, downloads packages, resolves dependencies and calls `dpkg`.

</details>

2. Why run `apt-get update` before `install`?

<details>
<summary>Answer</summary>

`update` refreshes the local package index. Without it `apt` may not know a package, or may pick an old version.

</details>

3. What is the difference between `remove` and `purge`?

<details>
<summary>Answer</summary>

`remove` deletes the program but keeps its configuration files; `purge` deletes the configuration too.

</details>

4. Which command tells you which package a file belongs to?

<details>
<summary>Answer</summary>

`dpkg -S /path/to/file`.

</details>

5. Why is adding a third-party repository a security decision?

<details>
<summary>Answer</summary>

Its packages are installed and upgraded as root, so the vendor gets the same power over your machine as the distribution. You must trust them and keep their key bound to only their repository.

</details>

6. When do you use `cargo install` and when `apt`?

<details>
<summary>Answer</summary>

`cargo install --locked` for Rust developer tools in your home folder; `apt` for system software and services that are installed system-wide and updated with the system.

</details>

## Summary

- A package manager installs software with its dependencies, verifies signatures, and records every file so removal is clean.
- On Ubuntu: `dpkg` handles one `.deb`; `apt` handles repositories and dependencies. Repositories are described in `/etc/apt/sources.list.d/` with suites, components and `Signed-By`.
- Workflow: `apt-get update`, `apt search`, `apt show`, `apt-cache policy`, `apt-get install -y`, `remove` / `purge` / `autoremove`.
- `dpkg -L` lists the files of a package, `dpkg -S` finds the owner of a file, `dpkg -l` shows the state.
- A repository is trusted with root rights: prefer the official archive, bind third-party keys with `Signed-By`.
- `cargo deb` packages a Rust program; `apt-get install ./file.deb` installs it. `cargo install --locked` is for developer tools in your home.
- Check a package with `dpkg -s` and its exit status.

## Sources and further reading

- [Ubuntu Server documentation — Package management](https://documentation.ubuntu.com/server/how-to/software/package-management/) — `apt` and `dpkg` on Ubuntu.
- [apt(8) manual](https://manpages.ubuntu.com/manpages/noble/man8/apt.8.html) — all `apt` commands.
- [sources.list(5) — deb822 format](https://manpages.ubuntu.com/manpages/noble/man5/sources.list.5.html) — fields of `.sources` files, including `Signed-By`.
- [dpkg(1) manual](https://manpages.ubuntu.com/manpages/noble/man1/dpkg.1.html) — `-L`, `-S`, `-l`, `-s` and more.
- [cargo-deb on GitHub](https://github.com/kornelski/cargo-deb) — the `[package.metadata.deb]` options.
- [The Cargo Book — cargo install](https://doc.rust-lang.org/cargo/commands/cargo-install.html) — what `cargo install` does and where it installs.

---

[🏠 Start](../START-HERE.md) · [← 11.6 Services with systemd](06-services-with-systemd.md) · [📋 Phase 11](README.md) · [11.8 Bash scripting basics →](08-bash-scripting-basics.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
