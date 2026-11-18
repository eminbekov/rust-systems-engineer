<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 00.4 The terminal: your first commands](04-terminal-basics.md) · [📋 Phase 00](README.md) · [00.6 Your first Rust program →](06-your-first-rust-program.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/00-getting-started/05-install-rust-and-rustrover.md) <!-- nav -->

# Lesson 00.5 — Installing Rust and RustRover

> **Phase 00 — Getting started** · Lesson 5 of 8 · ⏱ about 105 minutes

## What you will learn

- Which tools you need and what each one does: `rustup`, `rustc`, Cargo, Clippy, `rustfmt`, RustRover and Git.
- How to install the Rust toolchain on a Mac with Apple silicon, and how to check that it works.
- What stable, beta and nightly releases are, and what an "edition" is.
- How to install and set up the RustRover IDE.
- How to fix the three most common installation problems.

## Before you start

- You finished [Lesson 00.4 — The terminal: your first commands](04-terminal-basics.md). You know how to open the terminal, run a command and read `command not found`.
- You have an internet connection and about 5 GB of free disk space.
- Already installed Rust? Do not reinstall: just do the checks in section 3 and compare.

## New words

| Word | Meaning |
|---|---|
| **toolchain** | The set of programs needed to build Rust code: the compiler, the standard library, and helpers. |
| **rustup** | The installer and updater of Rust toolchains. You install it once, then it manages everything else. |
| **rustc** | The Rust compiler (Lesson 00.3). It turns one source file into an executable. |
| **Cargo** | Rust's build tool and package manager. It creates projects, runs the compiler for you, downloads libraries and runs tests. You use it all the time. |
| **Clippy** | A "linter" for Rust: a program that reads your code and gives advice about mistakes and bad style. Run it with `cargo clippy`. |
| **rustfmt** | A tool that formats your code in the standard Rust style. Run it with `cargo fmt`. |
| **release channel** | One of three kinds of Rust release: **stable**, **beta** and **nightly**. |
| **edition** | A named version of the language rules, such as 2021 or 2024. New projects use the newest edition. |
| **target triple** | The name of a platform, written as `cpu-vendor-os`, for example `aarch64-apple-darwin`. |
| **linker** | A program that joins the pieces produced by the compiler into one executable file. |
| **Xcode Command Line Tools** | A package from Apple with a C compiler, a linker, `git` and other developer tools. Rust needs its linker on macOS. |
| **IDE** (Integrated Development Environment) | A program for writing code: editor, run buttons, error highlighting and more. We use **RustRover**. |
| **Git** | The version control tool of Phase 01. Here we only check that it is installed. |
| **Homebrew** | A popular package manager for macOS: a command, `brew`, to install developer tools. Optional now. |
| **URL** (Uniform Resource Locator) | A web address such as `https://rustup.rs`. |
| **CPU** (Central Processing Unit) | The processor chip that runs instructions (Lesson 00.3). |
| **OS** (Operating System) | The main program of a computer that runs other programs; here macOS (Lesson 00.3). |
| **PATH** | The list of directories where the shell looks for commands (Lesson 00.4). |
| **HTTPS** (HyperText Transfer Protocol Secure) | The web protocol with encryption. Addresses that use it start with `https://`. |
| **VPN** (Virtual Private Network) | A service that sends your internet traffic through another server. Sometimes it blocks downloads. |
| **LLVM** | The code-generating engine inside the Rust compiler. (The name was once short for "Low Level Virtual Machine".) |
| **ARM** | The name of a family of CPU designs. The 64-bit form is called arm64 or AArch64. |
| **PC** (Personal Computer) | An ordinary desktop or laptop computer, for example a Windows PC. |
| **GB** (gigabyte) | About a billion bytes, a unit of size for memory and disks. |

## 1. What we are going to install

| Tool | What it is | Where it comes from |
|---|---|---|
| Command Line Tools | C compiler, linker, `git` | Apple (`xcode-select --install`) |
| `rustup` | manager of toolchains | the official site rustup.rs |
| `rustc`, `cargo`, `clippy`, `rustfmt`, offline docs | the toolchain | installed by `rustup` |
| RustRover | the IDE | JetBrains |

The order matters: first the Apple tools, then Rust, then the IDE.

## 2. Step 1 — Apple's Command Line Tools

On a Mac, Rust uses Apple's linker to produce the final executable. Check whether you already have
the tools. In the terminal run:

```text
% xcode-select -p
/Applications/Xcode.app/Contents/Developer
```

If you see a path (either `/Applications/Xcode.app/Contents/Developer` or
`/Library/Developer/CommandLineTools`), you are done. If you see an error such as
`xcode-select: error: unable to get active developer directory`, install the tools:

```text
% xcode-select --install
```

A window opens. Click **Install** and agree to the licence. The download can take several minutes. Then
run `xcode-select -p` again. Another check, which also confirms that you have a C compiler:

```text
% cc --version
Apple clang version 21.0.0 (clang-2100.4.1.1)
Target: arm64-apple-darwin27.2.0
Thread model: posix
```

(Your version numbers will be different. The important word is `arm64`.)

## 3. Step 2 — Install Rust with rustup

Go to the official page [rust-lang.org/tools/install](https://www.rust-lang.org/tools/install) and copy the
command for macOS. At the time of writing it is:

```text
% curl --proto '=https' --tlsv1.2 https://sh.rustup.rs -sSf | sh
```

What happens:

1. `curl` downloads something from the internet. `--proto '=https' --tlsv1.2` means "only allow a secure connection (HTTPS)". `-sSf` means: be quiet (`-s`), but show errors (`-S`), and fail on a server error (`-f`).
2. The address `https://sh.rustup.rs` is an **official** one: it is a script that installs `rustup`.
3. `|` (the pipe from Lesson 00.4) sends the downloaded script into `sh`, which runs it.

> 🔒 **Security:** Running a script straight from the internet is only safe when you trust the source
> *and* the address. Check three things before you press Enter: the page is the official
> `rust-lang.org` site, the address in the command is exactly `https://sh.rustup.rs`, and you typed or
> copied it from that page (not from an email or a chat message). If you prefer, you can open
> `https://sh.rustup.rs` in the browser first and read the script.

The installer prints a welcome text and shows what it is going to do: where the files go, which
toolchain it installs (the default is `stable`), and that it will change your `PATH`. Then it shows
a menu:

```text
1) Proceed with standard installation (default - just press enter)
2) Customize installation
3) Cancel installation
```

Press **Enter** for the standard installation. (The exact wording of the installer can change a bit
between versions; I did not run it again on my machine, which already has Rust.) It downloads a few
hundred megabytes. At the end it prints a line like `Rust is installed now. Great!`.

What the installer did on your computer:

- Created `~/.rustup`: the toolchains (the compiler and the standard library).
- Created `~/.cargo`: tools that you install, and `~/.cargo/bin`, which contains the programs `rustc`, `cargo`, `rustup`, `rustfmt` and others.
- Added a line `. "$HOME/.cargo/env"` to your shell start-up file (for zsh it is `~/.zshenv`). This line adds `~/.cargo/bin` to your `PATH` (the list from Lesson 00.4) every time you open a terminal.

**Now close the terminal tab and open a new one.** The old window does not know about the new
`PATH`. (If you do not want to close it, run `source "$HOME/.cargo/env"` once.)

### Check it works

```text
% rustc --version
rustc 1.100.0 (0ef1bc386 2026-11-09)
% cargo --version
cargo 1.100.0 (3acc79c0c 2026-11-05)
% rustup --version
rustup 1.29.3 (edbc831ce 2026-11-02)
info: This is the version for the rustup toolchain manager, not the rustc compiler.
info: the currently active `rustc` version is `rustc 1.100.0 (0ef1bc386 2026-11-09)`
```

Your version numbers may be newer. If the first number of `rustc` is lower than `1.100`, run
`rustup update` (see below). What does a version like `1.100.0` mean? Three numbers: major
`1`, minor `100`, patch `0`. Rust 1.0 appeared in 2015; since then a new minor version has appeared every
six weeks.

Where do the programs live? Ask `which`:

```text
% which rustc cargo rustup
/Users/student/.cargo/bin/rustc
/Users/student/.cargo/bin/cargo
/Users/student/.cargo/bin/rustup
```

All three are in `~/.cargo/bin`. This is the same directory the installer added to `PATH`.

Now ask the compiler about itself, with the "verbose version" option:

```text
% rustc -vV
rustc 1.100.0 (0ef1bc386 2026-11-09)
binary: rustc
commit-hash: 0ef1bc386059596dada8fb65b50425aab2ea637a
commit-date: 2026-11-09
host: aarch64-apple-darwin
release: 1.100.0
LLVM version: 23.1.5
```

The line `host: aarch64-apple-darwin` is the **target triple** of your computer: `aarch64` is the CPU
family (arm64, as in Lesson 00.3), `apple` is the vendor, `darwin` is the OS core of macOS. **It must say
`aarch64-apple-darwin`.** If it says `x86_64-apple-darwin`, the installer was started in a Terminal
running under Rosetta (an emulation layer for Intel programs). Uninstall (section 6) and install again in a normal terminal.

The `LLVM version` line names the engine under the compiler that actually generates the machine code
(`rustc` is the front part; LLVM is the code generator). You do not need to know more now.

### Toolchains, components and channels

```text
% rustup show
Default host: aarch64-apple-darwin
rustup home:  /Users/student/.rustup

installed toolchains
--------------------
stable-aarch64-apple-darwin (active, default)

active toolchain
----------------
name: stable-aarch64-apple-darwin
active because: it's the default toolchain
installed targets:
  aarch64-apple-darwin
```

(On my own computer the output also lists extra toolchains and targets that I installed for other
work; I removed those lines. A fresh installation looks like this.)

```text
% rustup component list --installed
cargo-aarch64-apple-darwin
clippy-aarch64-apple-darwin
rust-docs-aarch64-apple-darwin
rust-std-aarch64-apple-darwin
rustc-aarch64-apple-darwin
rustfmt-aarch64-apple-darwin
```

What happens:

1. `rustup show` lists the installed **toolchains**. The name `stable-aarch64-apple-darwin` = channel `stable` + target triple.
2. `rustup component list --installed` lists the **components** inside the toolchain. You can see the compiler (`rustc`), the build tool (`cargo`), the formatter (`rustfmt`), the linter (`clippy`), the standard library (`rust-std`) and the offline documentation (`rust-docs`).

Rust has three **release channels**:

| Channel | What it is | Do you use it? |
|---|---|---|
| **stable** | A tested release, published every 6 weeks. | **Yes, always, in this course.** |
| **beta** | The next stable release, in testing. | No. |
| **nightly** | A build made every night, with experimental features. | Only in a few advanced lessons (Phase 06 and later). |

Keep your toolchain fresh. Every few weeks run:

```text
% rustup update
```

It downloads the newest stable toolchain, if there is one, and replaces the old one. It is safe.

Finally, the **edition**. Rust promises that old code continues to compile, but sometimes new, cleaner rules are
needed. These are bundled into an *edition*: 2015, 2018, 2021 and 2024. The edition is chosen per
project in its `Cargo.toml` file. Cargo creates new projects with the newest edition, **2024**, and this course uses
2024 everywhere. You will see the line `edition = "2024"` in Lesson 00.6.

### The documentation is already on your disk

`rust-docs` means that you have the official books and the standard-library reference **offline**:

```text
% rustup doc --path
/Users/student/.rustup/toolchains/stable-aarch64-apple-darwin/share/doc/rust/html/index.html
```

`rustup doc` opens this page in your browser. Other useful options: `rustup doc --book` (the Rust
book), `rustup doc --std` (the standard library), `rustup doc --cargo` (the Cargo book),
`rustup doc --error-codes` (the list of compiler error codes, useful in Lesson 00.7).

## 4. Step 3 — Install RustRover

**RustRover** is an IDE made by JetBrains, specially for Rust. It highlights errors as you type, runs and
debugs your programs with a click, and has a terminal and a Git panel built in. According to the
[RustRover page](https://www.jetbrains.com/rust/), it is **free for non-commercial use** (for example for
learning), and paid if you use it to earn money; check the current conditions on the page when you download.

1. Open [jetbrains.com/rust/download](https://www.jetbrains.com/rust/download/) in the browser.
2. Choose macOS and the version for **Apple Silicon** (there are separate disk images for Intel and Apple silicon). Download the `.dmg` file.
3. Open the downloaded `.dmg` and drag **RustRover** into **Applications**.
4. Start it from Applications (or press `Cmd + Space` and type "RustRover"). The first time, macOS may ask "are you sure you want to open it?": confirm.
5. Accept the user agreement, choose the free non-commercial licence (or sign in with a JetBrains account).

(JetBrains also offers the **Toolbox App**, which installs and updates all their IDEs. It is optional.)

### First setup

RustRover can find the Rust you installed in `~/.cargo/bin` by itself. Check it:

1. On the welcome screen choose **Open**, select the folder `~/projects/rust` and trust the project. (The folder opens as a project; it has no `Cargo.toml` yet, which is fine for now.)
2. Open the settings: menu **RustRover → Settings** (shortcut `Cmd + ,`). The names of the pages can differ a little between versions; use the search box at the top of the settings window.
3. Go to **Languages & Frameworks → Rust**. The field **Toolchain location** should show a path ending in `.cargo/bin`, and the field below it should show the standard library path. If it is empty, press the folder button and select `~/.cargo/bin`.
4. In the same Rust settings, find the **Rustfmt** page and enable **Run rustfmt on Save**. Now your code is formatted automatically each time you save.
5. Find the **External Linters** page, choose **Clippy** as the tool, and enable running it on the fly. Now Clippy's advice appears right in the editor.
6. Markdown files (like this lesson) open in RustRover with a preview. Open a `.md` file and use the buttons at the top right of the editor to switch between source and preview.
7. The built-in terminal: menu **View → Tool Windows → Terminal** (shortcut `Option + F12`). It is the same terminal as in Lesson 00.4, already inside the project folder.

> 📏 **Convention:** Format code on every save and fix Clippy's warnings. In professional Rust teams
> this is required; code that fails `cargo fmt --check` or `cargo clippy` is not merged.

## 5. Step 4 — Check Git

Git is part of Apple's Command Line Tools, so you probably have it already. Check:

```text
% git --version
git version 2.54.0 (Apple Git-157)
```

(Yours may be different; any recent version is fine.) **Git** is the tool that keeps the history of your
files, as explained in Lessons 00.1 and 00.2. Phase 01 teaches it from zero, including how to configure
your name and e-mail in it. Today you only need to see that the command exists.

If you see `command not found: git`, go back to section 2 and install the Command Line Tools.

> 💡 **Tip:** Another tool that many developers install is **Homebrew** (`brew`), a package manager
> for macOS. You do not need it for this phase. Instructions are on [brew.sh](https://brew.sh/), and
> later phases will tell you when a tool is easier to install with it.

## 6. When something goes wrong

| Symptom | Cause | Fix |
|---|---|---|
| `zsh: command not found: cargo` (or `rustc`) | The terminal was opened before the installation, or `~/.cargo/bin` is not in `PATH`. | Close the tab and open a new one. If it still fails: `source "$HOME/.cargo/env"`, then `echo $PATH` and look for `.cargo/bin`. Check that `ls ~/.cargo/bin` lists `cargo`. |
| `error: linker 'cc' not found` when building | The Command Line Tools are missing. | `xcode-select --install`, then try again. |
| `rustc -vV` says `host: x86_64-apple-darwin` | Rust was installed from a Terminal running under Rosetta. | Run `rustup self uninstall`, open a normal terminal, install again. |
| `curl: (6) Could not resolve host` or other network errors | No internet, a VPN or a proxy blocks the download. | Check the connection, disable the VPN, try again. |
| RustRover shows "Cargo project not found" or red `use` lines | The IDE does not know the toolchain or the project. | Set the toolchain in settings (section 4). For projects: right-click `Cargo.toml` and choose **Attach Cargo Project** (Lesson 00.6). |

To remove Rust completely (you will not need this now): `rustup self uninstall`.

## Common mistakes

1. **Installing from a random website.** Search engines show fake download pages. *Fix:* always begin at `rust-lang.org` or `rustup.rs`.
2. **Not restarting the terminal.** You install Rust, then the old terminal says `command not found: cargo`. *Fix:* close the tab, open a new one.
3. **Using `sudo` for the installer.** `rustup` installs into your home directory and needs no administrator rights. Using `sudo` creates files that belong to the administrator and causes permission errors later. *Fix:* never put `sudo` before `curl ... | sh` here.
4. **Installing Rust with Homebrew *and* rustup.** Two copies fight over the `rustc` name. *Fix:* use rustup only; if you installed one with `brew`, run `brew uninstall rust`.
5. **Skipping the checks.** The installation seemed fine, but the target is wrong or the PATH is incomplete. *Fix:* run the checks in section 3 and compare the output.

## Try it yourself

Do these checks in order and write the results into `~/projects/rust/notes/day2.md` with `>>`.

1. `rustc --version` and `cargo --version`: are both at `1.100` or newer?
2. `rustc -vV | grep host`: is the host `aarch64-apple-darwin`? (Here `grep host` keeps only the line with the word `host`; the pipe is from Lesson 00.4.)
3. `which cargo`: does the path end with `.cargo/bin/cargo`?
4. `rustup component list --installed | grep -E 'clippy|rustfmt'`: are both there?
5. `git --version`: is Git installed?
6. In RustRover: open the settings and take a look at the Toolchain location. Is the path right?
7. Run `rustup doc --book` and find the first chapter, "Getting Started".

For example, step 1 together with the note: `rustc --version >> ~/projects/rust/notes/day2.md`.

## Exercises

### Exercise 1 — What is where? (easy)

Answer from your own computer: (a) the full path of the `cargo` program, (b) the folder that holds
all toolchains, (c) the channel of your active toolchain.

<details>
<summary>Hint</summary>

(a) `which`. (b) The installer created it in your home folder; `rustup show` prints it as "rustup home". (c) The first part of the toolchain name.

</details>

<details>
<summary>Answer</summary>

(a) `which cargo` → `/Users/<you>/.cargo/bin/cargo`.
(b) `rustup show` → `rustup home: /Users/<you>/.rustup` (the toolchains are inside, in `.rustup/toolchains`).
(c) The active toolchain is `stable-aarch64-apple-darwin`, so the channel is `stable`.

Why it works: `rustup` keeps everything in two folders in your home directory and puts the commands into `~/.cargo/bin`.

</details>

### Exercise 2 — Read the triple (easy)

Split `aarch64-apple-darwin` into its three parts and say what each means. Then explain why a program built
for this triple will not run on a PC with an Intel CPU running Windows.

<details>
<summary>Hint</summary>

Remember Lesson 00.3: the CPU family decides the machine code.

</details>

<details>
<summary>Answer</summary>

- `aarch64` — the CPU family (64-bit ARM, the same as arm64).
- `apple` — the vendor.
- `darwin` — the operating system core (macOS).

A Windows PC with an Intel CPU needs x86-64 machine code and an executable format for Windows. The program built for this triple contains arm64 machine code in the Mach-O format, so neither the CPU nor the OS can run it.

Why it works: machine code is specific to the CPU family, and the file format is specific to the OS.

</details>

### Exercise 3 — Find an error code (medium)

Use the offline documentation to find the explanation of the compiler error code `E0425`. Then
find the same explanation in the terminal.

<details>
<summary>Hint</summary>

`rustup doc --error-codes` opens the index. For the terminal there is a `rustc` option that begins with `--explain`.

</details>

<details>
<summary>Answer</summary>

Browser: `rustup doc --error-codes`, then look for E0425 in the list. Terminal: `rustc --explain E0425`.

The text begins with: "An unresolved name was used." That means: you used a name (a variable or function) that does not exist where you used it.

Why it works: every compiler error has a code, and the documentation for every code is shipped with the toolchain. Lesson 00.7 uses this a lot.

</details>

### Exercise 4 — Simulate a missing PATH (hard)

Using what you learned in Lesson 00.4 (section 10), run a clean shell whose `PATH` contains only
`/usr/bin:/bin`, and ask it for the Rust version. Then run it again with `~/.cargo/bin` added. Explain both results.

<details>
<summary>Hint</summary>

Use `env -i PATH=... /bin/zsh -f -c '...'`. The directory with `cargo` is `$HOME/.cargo/bin`. You need to give this clean shell `HOME`, too.

</details>

<details>
<summary>Answer</summary>

```text
% env -i PATH=/usr/bin:/bin /bin/zsh -f -c 'cargo --version'
zsh:1: command not found: cargo
% env -i HOME=$HOME PATH=$HOME/.cargo/bin:/usr/bin:/bin /bin/zsh -f -c 'cargo --version'
cargo 1.100.0 (3acc79c0c 2026-11-05)
```

Why it works: in the first command no directory of `PATH` contains a file `cargo`, so the shell says "command not found". The second adds `~/.cargo/bin`, so the shell finds the program. The same thing happens in a terminal that was opened before the installer changed your start-up file. (`rustup` also needs `HOME` to find `~/.rustup`, which is why we pass it.)

</details>

## Quiz

1. What is the difference between `rustup`, `rustc` and `cargo`?

<details>
<summary>Answer</summary>

`rustup` installs and updates toolchains. `rustc` is the compiler. `cargo` is the build tool and package manager that calls `rustc` for you.

</details>

2. Which release channel do we use, and how often does it get a new version?

<details>
<summary>Answer</summary>

`stable`. A new stable version comes out every 6 weeks.

</details>

3. What must the `host` line of `rustc -vV` say on your Mac?

<details>
<summary>Answer</summary>

`aarch64-apple-darwin`.

</details>

4. After installing Rust, `cargo` is "not found" in the old terminal. Why, and what do you do?

<details>
<summary>Answer</summary>

The old terminal started before the installer added `~/.cargo/bin` to `PATH`. Close it and open a new one (or run `source "$HOME/.cargo/env"`).

</details>

5. What do Clippy and rustfmt do?

<details>
<summary>Answer</summary>

Clippy gives advice about mistakes and bad style (`cargo clippy`). rustfmt formats the code in the standard style (`cargo fmt`).

</details>

6. Where is the Rust documentation available offline, and how do you open it?

<details>
<summary>Answer</summary>

It is installed with the toolchain (the `rust-docs` component). Run `rustup doc` (or `rustup doc --book`, `--std`, `--cargo`).

</details>

## Summary

- Install in this order: Apple Command Line Tools, Rust with `rustup`, then RustRover.
- The official Rust installer is one command from rust-lang.org; check the address before running a script from the internet.
- `~/.cargo/bin` holds `rustc`, `cargo` and `rustup`; it must be in `PATH`. Open a new terminal after installing.
- Check `rustc --version`, `cargo --version` and that `host` is `aarch64-apple-darwin`.
- We use the `stable` channel and edition 2024; update with `rustup update` every few weeks.
- The books and the standard-library reference are offline: `rustup doc`.
- RustRover is free for non-commercial use; enable rustfmt on save and Clippy in its settings.
- Git comes with Apple's tools; Phase 01 teaches it.

## Sources and further reading

- [The Rust Programming Language — Installation](https://doc.rust-lang.org/book/ch01-01-installation.html) — the official installation instructions, with troubleshooting.
- [Install Rust](https://www.rust-lang.org/tools/install) — the official download page.
- [The rustup book](https://rust-lang.github.io/rustup/) — the full manual of `rustup`: toolchains, components, channels, overrides.
- [rustup — Channels](https://rust-lang.github.io/rustup/concepts/channels.html) — stable, beta and nightly in detail.
- [The Rust Edition Guide](https://doc.rust-lang.org/edition-guide/rust-2024/index.html) — what Rust 2024 changed.
- [RustRover — Installation guide](https://www.jetbrains.com/help/rust/installation-guide.html) — JetBrains' official installation steps.

---

[🏠 Start](../START-HERE.md) · [← 00.4 The terminal: your first commands](04-terminal-basics.md) · [📋 Phase 00](README.md) · [00.6 Your first Rust program →](06-your-first-rust-program.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
