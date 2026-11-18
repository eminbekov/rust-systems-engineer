<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← Phase 11 — Linux, the shell and containers](README.md) · [📋 Phase 11](README.md) · [11.2 The Linux file system →](02-the-linux-file-system.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/11-linux-shell-and-containers/01-linux-and-a-virtual-machine.md) <!-- nav -->

# Lesson 11.1 — Linux and a virtual machine on your Mac

> **Phase 11 — Linux, the shell and containers** · Lesson 1 of 16 · ⏱ about 120 minutes

## What you will learn

- What Linux is, what a *distribution* is, and why almost every server in the world runs Linux.
- What a virtual machine (VM) is and why your Mac needs one to run Linux.
- How to install Lima, create the course VM called `dev`, enter it, leave it, stop it and start it again.
- Where your Mac files appear inside the VM, and what to do when the VM does not start.

## Before you start

- You finished [Lesson 00.4 — The terminal: your first commands](../00-getting-started/04-terminal-basics.md). You know `ls`, `cd`, paths, pipes, `PATH`, `chmod +x` and `sudo`.
- You know what an operating system does ([Lesson 00.3 — How a computer runs a program](../00-getting-started/03-how-a-computer-runs-a-program.md)).
- You have about 10 GB of free disk space and a stable internet connection (the first start downloads an operating system image).

**How the code of this phase is organised.** Most of this phase is typed in a terminal, not written in
Rust files. Small Rust programs go in `~/projects/rust/code/p11-<topic>` (for example `code/p11-heartbeat`),
shell scripts go in `code/p11-scripts`, and the final project lives in `code/p11-deploy`. From Lesson 11.2
on, every command in a `bash` block is typed **inside the Linux VM** unless the lesson says otherwise.

## New words

| Word | Meaning |
|---|---|
| **Linux** | The *kernel* (the core of an operating system) that most servers, phones (Android) and cloud machines use. Many people also say "Linux" for a whole system built around that kernel. |
| **kernel** | The central part of an operating system. It talks to the hardware and gives programs memory, files and network. |
| **distribution** (distro) | A complete Linux system: the kernel plus a package manager, tools and settings, put together by a team. Examples: Ubuntu, Debian, Fedora. |
| **Ubuntu** | A popular distribution. **LTS** (Long-Term Support) versions get security updates for many years. This course uses Ubuntu 26.04 LTS. |
| **VM** (Virtual Machine) | A whole computer simulated by a program. It has its own virtual CPU (Central Processing Unit, the processor), memory, disk and operating system. |
| **host / guest** | The real computer (your Mac) is the *host*. The operating system inside the VM is the *guest*. |
| **hypervisor** | The part of the host that runs VMs. On a Mac it is Apple's *Virtualization framework*, built into macOS. |
| **Lima** | A free tool that creates and starts Linux VMs on a Mac. Its command is `limactl`. |
| **instance** | One VM that Lima manages. Ours is named `dev`. |
| **template** | A ready-made description of a VM (which image to download, how much memory). We use the template `ubuntu-lts`. |
| **mount** | Making a folder of the host visible inside the guest. |
| **Homebrew** | The package manager for macOS (mentioned in Lesson 00.5). Its command is `brew`. |
| **arm64** (also *aarch64*) | The CPU family of Apple Silicon chips such as the M4. Your VM is arm64 too, because it uses your real CPU. |
| **GiB** (gibibyte) | 1024 × 1024 × 1024 bytes. Close to a gigabyte (GB), and the unit Linux tools print. |

## 1. What Linux is, and what a distribution is

Your Mac runs **macOS**. Most servers you will work with (in the cloud, in a company's rack, inside
containers) run **Linux**. A systems engineer must be at home there, so this phase teaches Linux from
the first command.

Linux is, strictly speaking, only a **kernel**. The kernel is the first program the computer starts. It
controls the CPU, memory, disks and network card, and it offers *system calls* (requests that programs make to
the operating system, [Lesson 00.3](../00-getting-started/03-how-a-computer-runs-a-program.md)). Alone, a
kernel is not usable: you also need a shell, commands such as `ls`, a way to install software, and
settings. A **distribution** collects all of that:

```text
  +-----------------------------------------------+
  |  your programs: cargo, your Rust service ...  |
  +-----------------------------------------------+
  |  the shell (bash) and tools: ls, grep, ps ... |   <- from the distribution
  |  package manager: apt                         |
  +-----------------------------------------------+
  |  the Linux kernel                             |   <- the "Linux" itself
  +-----------------------------------------------+
  |  hardware (CPU, memory, disk, network)        |
  +-----------------------------------------------+
```

Different distributions make different choices, but the kernel and the basic commands are the same. We
choose **Ubuntu 26.04 LTS** because it is very common on servers, it is well documented, and almost every
tutorial works on it. Its relatives (Debian, Fedora, Alpine and others) differ mainly in the package
manager and in some file locations. You will see the package manager in
[Lesson 11.7](07-package-managers.md).

> 💡 **Tip:** "LTS" means a long support time: Ubuntu 26.04 LTS receives security updates for five years
> by default. Companies choose LTS versions for servers, because they do not want to reinstall every six
> months.

macOS and Linux are cousins: both follow the same family of rules (called *Unix*), so `ls`, `cd`, `grep`
and pipes work the same way in both. But the details differ: the file layout, the `ls` options, how
programs are installed. That is why we practise on a real Linux.

## 2. What a virtual machine is, and why you need one

A Mac cannot run the Linux kernel directly: macOS is already using the hardware. The solution is a
**virtual machine**: a program (the hypervisor) that pretends to be a complete computer. The Linux kernel
starts inside it and believes it runs on real hardware.

```text
   Your Mac (host)
   +--------------------------------------------------------+
   |  macOS, RustRover, your terminal                       |
   |                                                        |
   |  +--------------------------------------------------+  |
   |  |  Virtual machine "dev" (guest)                   |  |
   |  |    Ubuntu 26.04: kernel, bash, apt, systemd      |  |
   |  |    virtual CPUs, 4 GiB memory, virtual disk      |  |
   |  +--------------------------------------------------+  |
   +--------------------------------------------------------+
```

The VM is **isolated**. If you delete all files inside it, or break the system by a wrong command, your
Mac does not notice. You can throw the VM away and create a new one in a few minutes. This makes it the
perfect place to learn system administration: mistakes are cheap.

Because your Mac has an arm64 chip, the VM is also arm64, and the CPU runs the guest code directly, so
the VM is fast. Later, in [Lesson 11.12](12-containers-and-docker-basics.md), you will see that Docker on
a Mac also uses a small hidden Linux VM: containers need the Linux kernel too.

> ⚡ **Performance:** A VM needs real memory and CPU time from your Mac. If your Mac has 16 GB of RAM (random-access memory, the working memory) or
> more, the defaults are comfortable. Stop the VM (Section 5) when you do not use it.

## 3. Install Lima and create the VM

**Lima** (Linux virtual machines for macOS) is a small open-source tool. It downloads an official Ubuntu
image, starts a VM with Apple's hypervisor, and gives you simple commands. It also shares your Mac files
with the VM and forwards ports. Its home page is [lima-vm.io](https://lima-vm.io/).

Lima is installed with Homebrew. If `brew --version` already works in your terminal, skip the next
paragraph. Otherwise open [brew.sh](https://brew.sh/) and run the install command that is printed on the
home page. (Read it before you press Enter, as in [Lesson 00.4](../00-getting-started/04-terminal-basics.md);
the Homebrew installer asks for your Mac password once, and it is the real, official installer.)

> 🔒 **Security:** Installing software is trusting its author. Install Lima only from the official sources:
> Homebrew's `lima` package or [lima-vm.io](https://lima-vm.io/docs/installation/). Never type your Mac
> password into anything you did not start yourself.

Now install Lima (this command is on the Lima installation page):

```bash
brew install lima
```

Check that it works:

```bash
limactl --version
```

Example output (yours will look similar; the version number will be newer or older):

```text
limactl version 2.1.0
```

**What happens:**

1. `brew install lima` downloads the `lima` package and puts the program `limactl` ("Lima control") on your `PATH`.
2. `limactl --version` prints the version. If you see `command not found`, close the terminal tab and open a new one, as with Rust in Lesson 00.5.

Create and start the course VM. We give it the name `dev` and use the template `ubuntu-lts`, which means
"the current Ubuntu LTS", at the time of writing 26.04:

```bash
limactl start --name=dev template:ubuntu-lts
```

**What happens:**

1. `limactl start` creates an instance (if it does not exist yet) and starts it.
2. `--name=dev` is the instance name. **Every lesson of this course uses the name `dev`.** Do not choose another one, or the commands will not match.
3. `template:ubuntu-lts` selects the template.
4. A text menu may appear. Choose **Proceed with the current configuration** and press Enter. (Add `--tty=false` to the command to skip the menu.)
5. Lima downloads the Ubuntu image (several hundred megabytes), creates a virtual disk, boots the VM, and installs a few helper programs inside it. The first start takes several minutes. Later starts take about half a minute.

Example output (yours will look similar; it is long, and only the main lines are shown):

```text
INFO[0000] Using the existing instance "dev" ...        <- only if it exists
INFO[0000] Starting the instance "dev" with VM driver "vz"
INFO[0002] Downloading the image (ubuntu-26.04-server-cloudimg-arm64.img)
INFO[0120] Attempting to download the nerdctl archive ...
INFO[0150] [hostagent] Waiting for the essential requirement 1 of 3: "ssh"
INFO[0180] [hostagent] The essential requirement 1 of 3 is satisfied
...
INFO[0200] READY. Run `limactl shell dev` to open the shell.
```

The exact lines differ between Lima versions. The important one is the last: **READY**. The VM runs.
By default Lima gives it 4 CPUs (or fewer if your Mac has fewer), 4 GiB of memory (or half of your Mac's
memory, if that is less) and a 100 GiB virtual disk. The disk file grows only as you use it, so it does
not take 100 GiB at once. (You can change this with the options `--cpus`, `--memory` and `--disk` on
the first start; the defaults are fine for the course.)

> ⚠️ **Common mistake:** Pressing `Ctrl + C` during the first start, or closing the terminal. The instance is
> then half-created. Run `limactl stop dev` and `limactl start dev`, or, if it keeps failing, see Section 6.

## 4. Enter the VM and look around

To open a shell in the VM:

```bash
limactl shell dev
```

Your prompt changes. You are now **inside Linux**. Example (yours will look similar, with your own user name):

```text
student@lima-dev:/Users/student/projects/rust$
```

The prompt shows `user@hostname:directory$`. Lima creates a user in the VM with **the same name as your
Mac user**, and the host name is `lima-` plus the instance name. Your Mac user can run `sudo` in the VM
without a password (Lima configures it for convenience; on a real server you would not do this).

Try a few commands. These outputs come from an Ubuntu 26.04 system; yours will match except for the
numbers (kernel version, memory, user name):

```bash
cat /etc/os-release
```

```text
PRETTY_NAME="Ubuntu 26.04.1 LTS"
NAME="Ubuntu"
VERSION_ID="26.04"
VERSION="26.04.1 LTS (Resolute Raccoon)"
VERSION_CODENAME=resolute
ID=ubuntu
ID_LIKE=debian
HOME_URL="https://www.ubuntu.com/"
SUPPORT_URL="https://help.ubuntu.com/"
BUG_REPORT_URL="https://bugs.launchpad.net/ubuntu/"
PRIVACY_POLICY_URL="https://www.ubuntu.com/legal/terms-and-policies/privacy-policy"
UBUNTU_CODENAME=resolute
LOGO=ubuntu-logo
```

**What happens:** `/etc/os-release` is a small text file that every modern distribution has. It says
which distribution and version you run. `ID_LIKE=debian` means that Ubuntu is built on Debian, so Debian
instructions usually work on Ubuntu too.

```bash
uname -m
uname -s
whoami
hostname
echo $SHELL
```

```text
aarch64
Linux
student
lima-dev
/bin/bash
```

**What happens:**

1. `uname` ("Unix name") prints facts about the system. `-m` is the machine (CPU family): `aarch64` is the Linux name for arm64. `-s` is the kernel name: `Linux`.
2. `whoami` prints your user name. `hostname` prints the name of this computer.
3. `echo $SHELL` prints your login shell: `/bin/bash`. Bash (Bourne Again SHell) is the default shell of Ubuntu. On your Mac the shell is zsh; the two are very similar for everyday use, and Lesson 11.8 teaches bash scripts.

Two more commands show the resources of the VM. The numbers depend on your configuration:

```bash
nproc
free -h
df -h /
```

Example output (yours will look similar):

```text
4
               total        used        free      shared  buff/cache   available
Mem:           3.8Gi       412Mi       2.9Gi       1.0Mi       703Mi       3.4Gi
Filesystem      Size  Used Avail Use% Mounted on
/dev/vda1        96G  2.1G   94G   3% /
```

**What happens:**

1. `nproc` ("number of processing units") prints the number of virtual CPUs.
2. `free -h` shows memory in human-readable units (`-h`): total, used, free. `Gi` means GiB.
3. `df -h /` ("disk free") shows the size and usage of the disk that holds `/`, the root of the file system. Lesson 11.2 explains `/`.

To leave the VM and return to your Mac, type `exit` (or press `Ctrl + D`). The prompt changes back to
the Mac prompt. **The VM keeps running** in the background. You can enter it again at any time.

## 5. Where are my files? Stop, start, delete

### Files of the Mac inside the VM

Lima **mounts** your Mac home folder into the VM at the same path, for example `/Users/student`. So the
folder `~/projects/rust` of your Mac is visible in the VM as `/Users/student/projects/rust`. This is
convenient: you edit code in RustRover on the Mac, and read it from Linux.

By default the mount is **read-only**: the VM can read your Mac files, but it cannot change or delete
them. This protects your Mac from mistakes made in the VM. Try it (the path begins with your Mac user name;
below it is `student`):

```bash
ls /Users/student/projects/rust
touch /Users/student/projects/rust/test-file
```

Example output (yours will look similar):

```text
GLOSSARY.md  PROGRESS.md  README.md  START-HERE.md  course  ru
touch: cannot touch '/Users/student/projects/rust/test-file': Read-only file system
```

**What happens:**

1. `ls` lists the course folder of your Mac, through the mount.
2. `touch` tries to create an empty file and fails: `Read-only file system`. This is the expected behaviour, not a problem.

The **home folder of the Linux user** is a different place: `/home/student.guest` (in older Lima versions
`/home/student.linux`). This folder lives inside the VM disk and **is writable**. The shell starts there
when you enter from a folder that does not exist in the VM. `~` inside the VM always means this
folder. Rule of thumb for this phase:

| Where | Path | You can write? | Use it for |
|---|---|---|---|
| Mac files seen from the VM | `/Users/<your-mac-name>/...` | no (read-only) | reading your lesson files and Rust code |
| VM home | `~` (`/home/<name>.guest`) | yes | everything you create while practising Linux |

When a lesson needs Rust code in the VM, you copy the folder from the Mac into the VM home with `cp -r`.
Building inside the VM's own disk is also faster than building through a shared folder.

> 📏 **Convention:** the outputs in this phase were recorded on an Ubuntu 26.04 test system whose user
> `student` has the home `/home/student`. In your VM the same folder is `/home/<your-name>.guest`. When an
> output shows `/home/student/...`, read it as "my VM home" (`~`); everything else is the same.

### Stop and start

Run these on your **Mac** terminal (not inside the VM):

```bash
limactl list
limactl stop dev
limactl list
limactl start dev
```

Example output of the two `list` commands (yours will look similar):

```text
NAME    STATUS     SSH                ARCH       CPUS    MEMORY    DISK      DIR
dev     Running    127.0.0.1:60022    aarch64    4       4GiB      100GiB    ~/.lima/dev

NAME    STATUS     SSH                ARCH       CPUS    MEMORY    DISK      DIR
dev     Stopped    -                  aarch64    4       4GiB      100GiB    ~/.lima/dev
```

**What happens:**

1. `limactl list` shows all your instances, their status (`Running` or `Stopped`) and where Lima stores them (`~/.lima/dev`). The **SSH** column is the address that Lima uses to talk to the VM; you will learn SSH (Secure Shell, encrypted remote login) in [Lesson 11.10](10-ssh-and-remote-servers.md).
2. `limactl stop dev` shuts the VM down like a normal computer: the files inside stay.
3. `limactl start dev` boots it again. Everything you created inside is still there.

When you finish working for the day, run `limactl stop dev` to give memory back to your Mac.

### Delete and recreate

If you want to start from zero, the VM can be deleted. **This removes every file inside it**, so copy
anything important to the Mac first (for example with `cp` into a writable place; Lesson 11.2 shows how):

```bash
limactl stop dev
limactl delete dev
limactl start --name=dev template:ubuntu-lts
```

> 💡 **Tip:** You can always delete the VM. Treat it as a lab that you can rebuild. Keep what matters
> (your Rust projects, your scripts) in Git, as you learned in [Phase 01](../01-git-and-version-control/README.md).

## 6. If the VM does not start

Problems are normal at the first start. Go through this list from top to bottom:

1. **Read the last lines of the output.** Most messages say what is wrong (no disk space, no network).
2. **Check that nothing is half-started:** `limactl list`. If the status is `Broken` or the instance is stuck, run `limactl stop --force dev` (force stop, flag `-f`), then `limactl start dev`.
3. **Look at the logs.** Lima writes them to files in the instance folder: `~/.lima/dev/ha.stderr.log` ("host agent" errors) and `ha.stdout.log`. Show the end with `tail -n 30 ~/.lima/dev/ha.stderr.log`.
4. **Update Lima:** `brew upgrade lima` (an old version may not know the current Ubuntu image). Then try again.
5. **Check free disk space** on the Mac (about 10 GB is needed) and your network. The first start downloads the image from the internet.
6. **Delete and recreate** the instance (Section 5): `limactl delete dev`, then the `start` command again.
7. **Ask for help.** Copy the error text into a search or to Claude; also see the Lima documentation in "Sources".

**Fallback if you cannot get the VM to work.** You can still follow most of this phase with Docker, which
the course also uses. Docker Desktop is a program that runs Linux containers on a Mac. This command opens
a throw-away Ubuntu 26.04 shell (`--rm` removes it when you exit, `-it` makes it interactive):

```bash
docker run --rm -it ubuntu:26.04
```

Differences you must know: the container starts as the user `root` (the administrator), has no `sudo`
and few tools installed (`apt-get update` and `apt-get install sudo procps` add them), keeps nothing
after you exit, and has no `systemd`, so [Lesson 11.6](06-services-with-systemd.md) needs the extra steps given
there. Docker itself is taught in [Lesson 11.12](12-containers-and-docker-basics.md).

## Common mistakes

1. **Running Linux commands in the Mac terminal.** You typed `apt install ...` on the Mac and got `command not found`. *Fix:* the prompt tells you where you are. Mac prompt: `%`. VM prompt: `student@lima-dev:...$`. Enter the VM with `limactl shell dev`.
2. **Running `limactl` commands inside the VM.** `limactl: command not found`. *Fix:* `limactl` manages the VM from the outside. `exit` first.
3. **A different instance name.** `limactl shell dev` says `instance "dev" does not exist`. *Fix:* check `limactl list`; create it with `--name=dev`.
4. **Trying to save files into `/Users/...` from the VM.** `Read-only file system`. *Fix:* write in the VM home `~`, and copy to the Mac only when you really need it (Lesson 11.2).
5. **Forgetting to stop the VM** and wondering why the fan runs. *Fix:* `limactl stop dev`.

## Try it yourself

Do these steps in order. After each, the expected result is given.

1. On the Mac: `limactl list`. Expected: a line for `dev` with status `Running`.
2. Enter: `limactl shell dev`. Expected: the prompt shows `@lima-dev`.
3. Run `cat /etc/os-release | grep VERSION_ID`. Expected: `VERSION_ID="26.04"`.
4. Make a file in the VM home and read it back:

   ```bash
   cd ~
   echo "hello from linux" > first.txt
   cat first.txt
   ls -l first.txt
   ```

   Expected: the text `hello from linux`, and a line starting with `-rw-rw-r--` (the permissions you know from Lesson 00.4; [Lesson 11.4](04-users-groups-and-permissions.md) explains why the group may write here).
5. Leave with `exit`, stop the VM with `limactl stop dev`, start it with `limactl start dev`, enter again and run `cat ~/first.txt`. Expected: the file is still there.

## Exercises

### Exercise 1 — Which computer am I on? (easy)

Without looking at the lesson, name the command that prints the distribution name and version, and the command
that prints the CPU family. Then run both in the VM and in your Mac terminal. Which of the two commands works
on the Mac, and what does it print there?

<details>
<summary>Hint</summary>

One command reads a file in `/etc`, the other is `uname` with an option. macOS has no `/etc/os-release`, but it has `uname`.

</details>

<details>
<summary>Answer</summary>

```bash
cat /etc/os-release
uname -m
```

In the VM: the Ubuntu 26.04 description and `aarch64`. On the Mac: `cat /etc/os-release` fails with
`No such file or directory`, and `uname -m` prints `arm64` (macOS says `arm64`, Linux says `aarch64`, for
the same CPU family).

Why it works: `/etc/os-release` is a convention of Linux distributions, not of Unix in general. `uname` is
older and exists everywhere, but the names differ.

</details>

### Exercise 2 — Prove the mount is read-only (easy)

Inside the VM, find your Mac folder `~/projects/rust` (hint: it starts with `/Users/`). Show that you can list it
but not create a file there, and that you can create a file in `~`.

<details>
<summary>Hint</summary>

Use `ls /Users` to find your Mac user name. Use `touch` to try to create a file.

</details>

<details>
<summary>Answer</summary>

```bash
ls /Users
touch /Users/student/projects/rust/x
touch ~/x
ls -l ~/x
rm ~/x
```

The first `touch` fails with `Read-only file system`, the second works. (Replace `student` with the name
printed by `ls /Users`.)

Why it works: Lima mounts the Mac home folder read-only by default, while `~` lives on the VM disk.

</details>

### Exercise 3 — Write a check, not a guess (medium)

A **test** answers yes or no. Write a single command line that exits with status 0 (success) if the VM
runs Ubuntu 26.04 and with a non-zero status otherwise, prints nothing, and then print the status with `echo $?`.
Run it on the VM and also with `26.05` instead of `26.04` to see the failure.

<details>
<summary>Hint</summary>

`grep -q` ("quiet") prints nothing and only sets the exit status. Every command leaves its status in the special variable `$?` (0 means success).

</details>

<details>
<summary>Answer</summary>

```bash
grep -q '^VERSION_ID="26.04"$' /etc/os-release
echo $?
grep -q '^VERSION_ID="26.05"$' /etc/os-release
echo $?
```

```text
0
1
```

Why it works: `grep` exits with 0 when it finds a match and 1 when it does not. `^` and `$` mean "start
and end of the line", so only the exact line matches. [Lesson 11.5](05-processes-and-signals.md) explains exit
statuses in detail, and [Lesson 11.8](08-bash-scripting-basics.md) uses them in scripts.

</details>

### Exercise 4 — Give the VM a second life (medium)

Stop the VM, start it, and confirm that a file in `~` survived. Then explain in one sentence the
difference between `limactl stop dev` and `limactl delete dev`.

<details>
<summary>Answer</summary>

`stop` powers the VM off but keeps its disk (all files stay); `delete` removes the instance and its disk
(all files inside are lost). That is why `first.txt` from "Try it yourself" survives a stop and would be
gone after a delete.

</details>

## Quiz

1. Is "Linux" a complete operating system or a kernel? What do we call a complete system built around it?

<details>
<summary>Answer</summary>

Strictly, Linux is the kernel. A complete system is a *distribution*, for example Ubuntu.

</details>

2. What are the *host* and the *guest*?

<details>
<summary>Answer</summary>

The host is the real computer (your Mac with macOS). The guest is the operating system running inside the virtual machine (Ubuntu).

</details>

3. Why is the VM called `dev` in every lesson?

<details>
<summary>Answer</summary>

So that all commands in the course (`limactl shell dev`, `limactl stop dev`) work exactly as written for every reader.

</details>

4. You typed `touch /Users/me/projects/rust/a` in the VM and got `Read-only file system`. What went wrong?

<details>
<summary>Answer</summary>

Nothing is broken. Lima mounts the Mac home folder read-only by default. Create files in the VM home `~` instead.

</details>

5. What does the VM keep after `limactl stop dev`? After `limactl delete dev`?

<details>
<summary>Answer</summary>

After `stop`: everything (the disk stays). After `delete`: nothing, the instance and its disk are removed.

</details>

6. Which prompt tells you that you are inside the VM?

<details>
<summary>Answer</summary>

A prompt like `student@lima-dev:~$` (user, `@`, host name `lima-dev`, directory, `$`). The Mac prompt of this course ends with `%`.

</details>

## Summary

- Linux is a kernel; a distribution (Ubuntu) adds the tools around it. This course uses Ubuntu 26.04 LTS.
- A virtual machine is a simulated computer. Lima creates and runs Linux VMs on your Mac with `limactl`.
- The course VM is created once with `limactl start --name=dev template:ubuntu-lts` and entered with `limactl shell dev`.
- Your Mac home is visible in the VM at `/Users/<name>` but read-only; the VM home `~` is writable.
- `limactl list`, `limactl stop dev`, `limactl start dev` and `limactl delete dev` manage its life.
- When the VM misbehaves: read the output, check the logs in `~/.lima/dev`, update Lima, then delete and recreate.
- A throw-away `docker run --rm -it ubuntu:26.04` is a fallback for most commands of this phase.

## Sources and further reading

- [Lima — home page](https://lima-vm.io/) — what Lima is and why it exists.
- [Lima — Installation](https://lima-vm.io/docs/installation/) — the official install commands.
- [Lima — Usage](https://lima-vm.io/docs/usage/) — create, start, shell, mounts and the guest home.
- [Lima — limactl start](https://lima-vm.io/docs/reference/limactl_start/) — all options of `start`, such as `--cpus` and `--memory`.
- [Ubuntu — Release cycle](https://ubuntu.com/about/release-cycle) — what LTS means and how long a release is supported.
- [Homebrew](https://brew.sh/) — the package manager used to install Lima.

---

[🏠 Start](../START-HERE.md) · [← Phase 11 — Linux, the shell and containers](README.md) · [📋 Phase 11](README.md) · [11.2 The Linux file system →](02-the-linux-file-system.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
