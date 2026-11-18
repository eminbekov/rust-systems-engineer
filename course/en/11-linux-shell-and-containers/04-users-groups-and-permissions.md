<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 11.3 Text tools, pipes and redirection](03-text-tools-pipes-and-redirection.md) · [📋 Phase 11](README.md) · [11.5 Processes and signals →](05-processes-and-signals.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/11-linux-shell-and-containers/04-users-groups-and-permissions.md) <!-- nav -->

# Lesson 11.4 — Users, groups and permissions

> **Phase 11 — Linux, the shell and containers** · Lesson 4 of 16 · ⏱ about 135 minutes

## What you will learn

- What users and groups are, how Linux stores them, and what `root` and `sudo` mean.
- How to create users and groups, and why services should run as their own user.
- How to read and change permissions on files and directories with `chmod`, `chown` and `umask`.
- What the special bits do (sticky, setuid, setgid) and how to build a shared directory for a team.
- How a Rust program creates a private file, with a test.

## Before you start

- You finished [Lesson 11.3 — Text tools, pipes and redirection](03-text-tools-pipes-and-redirection.md).
- From [Lesson 00.4](../00-getting-started/04-terminal-basics.md) you know `ls -l`, the letters `r`, `w`, `x`, `chmod +x` and the idea of `sudo`. Here we explain the whole system.
- Sections 1 to 6 are typed in the VM (`limactl shell dev`). Section 7 uses Rust on your Mac.

## New words

| Word | Meaning |
|---|---|
| **user** | An account that owns files and runs programs. Every process runs as exactly one user. |
| **UID** (User ID) | The number that identifies a user. The kernel uses numbers; names are only for humans. |
| **group** | A named set of users. A user has one *primary* group and may belong to more *supplementary* groups. |
| **GID** (Group ID) | The number that identifies a group. |
| **root** | The administrator, the user with UID 0. The kernel does not check permissions for root. |
| **sudo** | A program that runs one command as another user, usually as root, if you are allowed to. |
| **permission bits** | Nine switches on every file: read, write and execute, for the owner, the group and everybody else. |
| **octal mode** | The permission bits written as three digits, such as `755` or `640`. |
| **umask** | A mask that removes permission bits from every new file and directory. |
| **sticky bit** | On a directory: only the owner of a file may delete it, even if others can write into the directory. |
| **setuid / setgid** | A bit that makes a program run as the *owner* / *group* of the file; on a directory, `setgid` makes new files inherit the directory's group. |
| **least privilege** | The security rule: give every user and program only the rights it needs, and no more. |
| **system user** | A user without a login, created to run one service. Its UID is below 1000. |

## 1. Users and groups

Every process runs **as a user**, and every file belongs to **one user and one group**. Ask who you are:

```bash
whoami
id
id root
```

```text
student
uid=1000(student) gid=1000(student) groups=1000(student),27(sudo)
uid=0(root) gid=0(root) groups=0(root)
```

**What happens:**

1. `whoami` prints your user name.
2. `id` prints your numbers. `uid=1000`: your user ID. `gid=1000`: your primary group. `groups=...`: all groups; `27(sudo)` means you may use `sudo`. (In the Lima VM your UID is the same as your Mac user's, often 501, and the groups differ a little.)
3. `id root` shows the administrator: **UID 0**.

The kernel only knows numbers. Names live in plain text files:

```bash
head -n 3 /etc/passwd
grep '^student' /etc/passwd
getent group sudo
```

```text
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
student:x:1000:1000::/home/student:/bin/bash
sudo:x:27:student
```

Each line of `/etc/passwd` has seven fields separated by colons:

| Field | Example | Meaning |
|---|---|---|
| 1 | `student` | login name |
| 2 | `x` | the password is not here (it is in `/etc/shadow`) |
| 3 | `1000` | UID |
| 4 | `1000` | primary GID |
| 5 | (empty) | a comment, often the full name |
| 6 | `/home/student` | home directory |
| 7 | `/bin/bash` | login shell; `/usr/sbin/nologin` means "no interactive login" |

`/etc/group` lists groups the same way (`name:x:GID:members`). `getent` ("get entries") asks the system's user database, which may also include
accounts that are not in the file (a company directory, for example). The hashed passwords live in `/etc/shadow`, which only `root` can read:

```bash
cat /etc/shadow
ls -l /etc/passwd /etc/shadow
```

```text
cat: /etc/shadow: Permission denied
-rw-r--r-- 1 root root   884 Nov 18 06:27 /etc/passwd
-rw-r----- 1 root shadow 503 Nov 18 06:27 /etc/shadow
```

**What happens:** `/etc/passwd` must be readable by everybody, because programs look up user names in it. The secrets were moved to `/etc/shadow` with
mode `rw-r-----`: root can read and write, the group `shadow` can read, others have nothing. Remember this pattern: **public data and secret data in different files with different permissions**.

User numbers have conventions: **0** is root, **1 to 999** are *system users* (for services), **1000 and up** are normal people.

## 2. `root` and `sudo`

`root` ignores permissions: it can read, change and delete anything. That is why you do not work as root. You work as a normal user and ask
for the rights of root only for the single command that needs them:

```bash
whoami
sudo whoami
sudo -u nobody whoami
```

```text
student
root
nobody
```

**What happens:**

1. `sudo command` runs `command` as root, after checking that you are allowed to (the file `/etc/sudoers`). On a real server it asks for **your** password; in the Lima VM it does not (Lesson 11.1).
2. `sudo -u nobody whoami` runs the command as another user (`-u`), here `nobody`, a user without any rights.
3. To edit the rules, use `sudo visudo`, which checks the syntax before saving. A broken sudoers file can lock you out.

> 💡 **Tip:** On Ubuntu 26.04 the `sudo` command is *sudo-rs*, a rewrite of `sudo` in Rust. Check with `sudo --version`.
> The commands in this lesson work the same way.

> 🔒 **Security:** Apply the rule of **least privilege**. Do not log in as root, do not run a service as root, do not `sudo` a command you do not understand.
> A bug in a program that runs as root can destroy the whole machine; the same bug in a program with its own small user is limited to that user's files.

## 3. Creating users and groups

Administration of accounts needs root rights, so these commands start with `sudo`. We create two people and a group:

```bash
sudo useradd -m -s /bin/bash alice
sudo useradd -m -s /bin/bash bob
sudo groupadd team
sudo usermod -aG team alice
sudo usermod -aG team bob
id alice
getent group team
```

```text
uid=1001(alice) gid=1001(alice) groups=1001(alice),1003(team)
team:x:1003:alice,bob
```

**What happens:**

1. `useradd -m -s /bin/bash alice` creates the user with a home directory (`-m`, make home) and the login shell `bash` (`-s`). The system picks the next free UID (1001 here) and creates a group named `alice` too.
2. `groupadd team` creates a group.
3. `usermod -aG team alice` adds (`-a`, append) alice to the supplementary group (`-G`) `team`. **Always use `-a` together with `-G`**: without `-a`, the user is removed from all other groups.
4. `id alice` confirms it. (The numbers in your VM may differ.) A user who is already logged in must log in again to see a new group.

> ⚠️ **Common mistake:** `sudo usermod -G team alice` without `-a` replaces the whole group list. If alice was in `sudo`, she just lost it.

On Debian and Ubuntu there is also `adduser`, a friendlier interactive wrapper that asks for a password and a name. The low-level
`useradd` is the same on all distributions and is better for scripts.

**A user for a service.** A web server or your Rust program should not run as you, and never as root. Give it a **system user**:

```bash
sudo useradd --system --no-create-home --shell /usr/sbin/nologin heartbeat
id heartbeat
sudo -u heartbeat whoami
```

```text
uid=999(heartbeat) gid=999(heartbeat) groups=999(heartbeat)
heartbeat
```

**What happens:** `--system` takes a UID below 1000; `--no-create-home` makes no home folder; `--shell /usr/sbin/nologin` means nobody can log in as this user, it can only
run its own service. [Lesson 11.6](06-services-with-systemd.md) creates exactly this user for a Rust service. Remove the demonstration user now:

```bash
sudo userdel heartbeat
```

## 4. Permissions: files and directories

`ls -l` shows ten characters at the start of a line: the type and three groups of three bits.

```text
-rwxr-xr-x  1  student  student  ...  name
│└┬┘└┬┘└┬┘     │        │
│ │  │  │      │        └ group
│ │  │  │      └ owner
│ │  │  └ other (everybody else)
│ │  └ group bits
│ └ owner bits
└ type: - file, d directory, l symlink
```

The kernel checks the rules in this order: if you are the **owner**, the owner bits decide; else if you are in the file's **group**, the group bits decide;
else the **other** bits decide. The bits have a different meaning for files and for directories:

| Bit | On a file | On a directory |
|---|---|---|
| `r` read | read the content | list the names inside |
| `w` write | change the content | create, rename and delete entries inside |
| `x` execute | run it as a program | enter it (`cd`) and reach the entries by name |

Directories are the surprising part. Try it:

```bash
cd ~
mkdir perm-lab
cd perm-lab
mkdir box
echo secret > box/file.txt
chmod -x box
ls box
ls -l box
cat box/file.txt
```

```text
file.txt
total 0
-????????? ? ? ? ?            ? file.txt
cat: box/file.txt: Permission denied
```

**What happens:** without `x` on the directory you can still list the *names* (you have `r`), but you cannot get the details of the entries and cannot open the file inside, even though the file itself is readable. Now the other way round:

```bash
chmod +x box
chmod -r box
ls box
cat box/file.txt
chmod +r box
```

```text
ls: cannot open directory 'box': Permission denied
secret
```

**What happens:** with `x` but without `r` you cannot list the directory, yet you can open a file if you know its exact name. Nothing is changed at the end: `chmod +r box` restored the bit.

### `chmod`: symbolic and octal

`chmod` ("change mode") has two notations. The **symbolic** one says who (`u` owner, `g` group, `o` other, `a` all), what to do (`+` add, `-` remove, `=` set) and which bits:

```bash
touch notes.txt
ls -l notes.txt
chmod u+x,g-w,o-r notes.txt
ls -l notes.txt
chmod a=r notes.txt
ls -l notes.txt
```

```text
-rw-rw-r-- 1 student student 0 Nov 18 06:50 notes.txt
-rwxr--r-- 1 student student 0 Nov 18 06:50 notes.txt
-r--r--r-- 1 student student 0 Nov 18 06:50 notes.txt
```

(The first line shows that your own new file is group-writable. Section 5 explains why.)

The **octal** notation writes each group of bits as one digit: read = 4, write = 2, execute = 1, and you add them. `rwx` = 4+2+1 = 7, `rw-` = 6, `r-x` = 5, `r--` = 4, `---` = 0.

| Mode | Bits | Typical use |
|---|---|---|
| `644` | `rw-r--r--` | a normal file, readable by all |
| `600` | `rw-------` | a private file: secrets, SSH (Secure Shell, Lesson 11.10) keys, tokens |
| `755` | `rwxr-xr-x` | a program or a public directory |
| `700` | `rwx------` | a private directory |
| `640` | `rw-r-----` | owner writes, the group reads (like `/etc/shadow`) |

```bash
chmod 755 notes.txt
stat -c '%a %A %n' notes.txt
```

```text
755 -rwxr-xr-x notes.txt
```

**What happens:** `stat -c` prints chosen fields (`%a` octal mode, `%A` letters, `%n` name).

> ⚠️ **Common mistake:** `chmod 777`. It lets everybody on the machine change the file, and is often an attempt to "make the error go away". Find out who needs which access and give exactly that.
> Also be careful with `chmod -R` (recursive): files and directories need different modes (`x` on directories, usually not on files).

When you meet "Permission denied" on a deep path, `namei -l` shows the permissions of **every directory on the way**, and the one that blocks you:

```bash
mkdir -p a/b
touch a/b/c
chmod 700 a
namei -l ~/perm-lab/a/b/c
```

```text
f: /home/student/perm-lab/a/b/c
drwxr-xr-x root    root    /
drwxr-xr-x root    root    home
drwxr-x--- student student student
drwxrwxr-x student student perm-lab
drwx------ student student a
drwxrwxr-x student student b
-rw-rw-r-- student student c
```

**What happens:** each line is one part of the path with its mode, owner and group. Anyone but `student` is stopped at `a` (`drwx------`) or at the home directory. (The listing looks the same for you, with your own user name.)

## 5. Owner, group and `umask`

`chown` ("change owner") changes the owner and the group; `chgrp` changes only the group. Only root may give a file to another owner:

```bash
sudo chown alice notes.txt
sudo chown alice:team notes.txt
ls -l notes.txt
sudo chown student:student notes.txt
```

```text
-rwxr-xr-x 1 alice team 0 Nov 18 06:50 notes.txt
```

**What happens:** `owner:group` sets both. After the last command the file is yours again. `chown -R` does it for a whole tree.

A new file gets its mode from a program's request (usually `666` for files, `777` for directories) **minus** the **umask**:

```bash
umask
touch u1
umask 077
touch u2
mkdir d2
umask 002
ls -ld u1 u2 d2
```

```text
0002
drwx------ 2 student student 4096 Nov 18 06:50 d2
-rw-rw-r-- 1 student student    0 Nov 18 06:50 u1
-rw------- 1 student student    0 Nov 18 06:50 u2
```

**What happens:**

1. `umask` prints the current mask: `0002` removes only the "write for others" bit, so `666` becomes `664` (`rw-rw-r--`). Ubuntu uses `002` for normal users, because each user has a group of their own; the common value on other systems is `022`.
2. After `umask 077` everything is removed from group and others: files `600`, directories `700`.
3. The change lasts for the current shell only. The last `umask 002` puts it back.

> 🔒 **Security:** A program that writes a secret (a key, a token, a password file) must create it **private from the start**, with mode `600`.
> If it writes first and runs `chmod` afterwards, there is a moment when other users can read it. Section 7 shows how to do it in Rust.

## 6. Special bits and a shared directory

Three more bits exist, shown in the `x` position:

| Bit | Octal | Where | Effect |
|---|---|---|---|
| sticky `t` | `1000` | directory | only the file's owner (or root) can delete or rename entries |
| setgid `s` | `2000` | directory | new files get the directory's group |
| setuid `s` | `4000` | program file | the program runs as the file's owner |

You have used all three:

```bash
ls -ld /tmp
ls -l /usr/bin/passwd
```

```text
drwxrwxrwt 1 root root 4096 Nov 18 06:35 /tmp
-rwsr-xr-x 1 root root 142552 Feb  2  2026 /usr/bin/passwd
```

**What happens:** `/tmp` is writable for everybody, and `t` (sticky) stops one user from deleting another user's files there. `passwd` has `s` in the owner bits: when any user runs it, it runs as root, so it can change `/etc/shadow`. Setuid programs are dangerous if they have bugs, so there are very few of them.

**A shared directory.** Alice and bob work together; nobody else should see the files. Create a directory with the group `team` and the `setgid` bit (`2770`):

```bash
sudo mkdir /srv/team-share
sudo chown root:team /srv/team-share
sudo chmod 2770 /srv/team-share
ls -ld /srv/team-share
sudo -u alice touch /srv/team-share/a.txt
sudo ls -l /srv/team-share
sudo -u bob bash -c 'echo hi >> /srv/team-share/a.txt'
ls /srv/team-share
```

```text
drwxrws--- 2 root team 4096 Nov 18 06:50 /srv/team-share
total 0
-rw-r--r-- 1 alice team 0 Nov 18 06:50 a.txt
bash: line 1: /srv/team-share/a.txt: Permission denied
ls: cannot open directory '/srv/team-share': Permission denied
```

**What happens:**

1. `2770` = setgid (`2`) + `rwx` for owner (`7`), `rwx` for group (`7`), nothing for others (`0`). `ls` shows `s` in the group bits. Your own user `student` is neither owner nor in `team`, so the last `ls` is denied: that is the goal.
2. The file alice created has the group `team`: this is the work of the `setgid` bit; without it the group would be alice's own group.
3. But bob cannot write to it: alice's `umask` gave the file `644`, so the group has no `w`. The directory is shared, the files are not, until the creator's umask allows it. Try again with `umask 002`:

```bash
sudo -u alice bash -c 'umask 002; touch /srv/team-share/b.txt'
sudo -u bob bash -c 'echo hi >> /srv/team-share/b.txt'
sudo cat /srv/team-share/b.txt
```

```text
hi
```

**What happens:** `b.txt` got `664` (`rw-rw-r--`) and group `team`, so bob can append. Team members usually set `umask 002` in their shell settings for this reason.

Clean up everything from this lesson (the `-r` of `userdel` also removes the home directory):

```bash
sudo rm -r /srv/team-share
sudo userdel -r alice
sudo userdel -r bob
sudo groupdel team
rm -r ~/perm-lab
```

(A message such as `userdel: alice mail spool (/var/mail/alice) not found` is harmless: these users never received mail.)

## 7. Permissions from Rust

Open a **Mac** terminal tab (permissions work the same way on macOS). A program that stores a secret should create the file with the right mode **at the moment of creation**. On Unix systems Rust has
`std::os::unix::fs::OpenOptionsExt`:

```bash
cd ~/projects/rust
cargo new code/p11-secretfile
```

Replace `code/p11-secretfile/src/main.rs`:

```rust
use std::fs::OpenOptions;
use std::io::{self, Write};
use std::os::unix::fs::{OpenOptionsExt, PermissionsExt};
use std::path::Path;

/// Creates a NEW file that only the owner can read and write (mode 600).
fn write_secret(path: &Path, text: &str) -> io::Result<()> {
    let mut file = OpenOptions::new()
        .write(true)
        .create_new(true) // fail if the file already exists
        .mode(0o600) // rw------- (the umask can only remove bits)
        .open(path)?;
    file.write_all(text.as_bytes())
}

fn main() -> io::Result<()> {
    let path = Path::new("token.txt");
    write_secret(path, "not-a-real-token\n")?;
    let mode = std::fs::metadata(path)?.permissions().mode();
    println!("created {} with mode {:o}", path.display(), mode & 0o777);
    Ok(())
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn secret_file_is_private_and_not_overwritten() {
        let path = std::env::temp_dir().join(format!("p11-secretfile-{}", std::process::id()));
        write_secret(&path, "x").unwrap();

        let mode = std::fs::metadata(&path).unwrap().permissions().mode();
        assert_eq!(mode & 0o777, 0o600);
        assert!(write_secret(&path, "y").is_err(), "second write must fail");

        std::fs::remove_file(&path).unwrap();
    }
}
```

**What happens:**

1. `OpenOptions` describes how to open a file. `.create_new(true)` means "create it, and fail if it already exists", so the program never overwrites or follows a file that an attacker prepared earlier.
2. `.mode(0o600)` sets the permission bits of the new file in the same system call. The `0o` prefix writes an octal number in Rust. The umask can only remove bits, so the result is never wider than `600`.
3. `mode & 0o777` keeps the nine permission bits and hides the file-type bits that `mode()` also returns. `{:o}` prints in octal.
4. The test creates a file in the temporary directory (with the process ID in the name), checks the mode, checks that a second `write_secret` fails, and removes the file.

```bash
cargo run -q
ls -l token.txt
cargo run -q
cargo test
```

```text
created token.txt with mode 600
-rw------- 1 student staff 17 Nov 18 11:46 token.txt
Error: Os { code: 17, kind: AlreadyExists, message: "File exists" }
```

(The `cargo test` output ends with `test result: ok. 1 passed`. `ls -l` shows your own user and group names.) The second run fails on purpose: the file existed. Delete `token.txt` when you are done. [Lesson 10.8](../10-security-fundamentals/08-secrets-management.md) explains how to manage secrets; here you saw the permission part.

## Common mistakes

1. **`chmod 777` to fix "Permission denied".** Everybody can now change the file. *Fix:* find the real cause with `ls -l` and `namei -l`; give the needed access to one user or group only.
2. **`usermod -G` without `-a`.** The user loses all other groups. *Fix:* `usermod -aG group user`.
3. **Forgetting `x` on a directory.** `cd: Permission denied`, or files that exist cannot be opened. *Fix:* directories need `x`; use `chmod 755` / `750` / `700` for directories, not `644`.
4. **A new group does not apply.** You added yourself to a group but still get "Permission denied". *Fix:* log out and in (or enter the VM again); a running shell keeps the old group list.
5. **Running a service as root.** One bug gives an attacker the whole machine. *Fix:* create a system user for it (Lesson 11.6).
6. **Secrets created with the default mode.** A token file with `644` is readable by every user. *Fix:* create it with `600` from the start.

## Try it yourself

Do the steps in order; each shows its expected result.

1. Run `id` and `groups`. Which groups do you belong to?
2. Create `mkdir ~/perm-lab && cd ~/perm-lab && touch report.txt`. Make it readable and writable by you only, and show `ls -l report.txt`. Expected: `-rw-------`.
3. Add `x` for the owner with the symbolic notation, then set `640` with the octal notation. Check with `stat -c '%a %A' report.txt`. Expected: `640 -rw-r-----`.
4. Create the user `alice` and the group `team`, and add alice to the group. Run `sudo -u alice cat ~/perm-lab/report.txt` and explain why you see "Permission denied" (hint: `namei -l`).
5. Clean up: `sudo userdel -r alice`, `sudo groupdel team`, `rm -r ~/perm-lab`.

## Exercises

### Exercise 1 — A private directory (medium)

Create `~/private-notes` that only you can enter or list, and a file `idea.txt` inside it. Show with `sudo -u nobody cat ~/private-notes/idea.txt` that another user cannot read it. Clean up.

<details>
<summary>Hint</summary>

A directory needs `rwx` for the owner and nothing for the others: mode `700`. If you set it before creating the file, the file is protected from the start.

</details>

<details>
<summary>Answer</summary>

```bash
mkdir ~/private-notes
chmod 700 ~/private-notes
echo "an idea" > ~/private-notes/idea.txt
sudo -u nobody cat ~/private-notes/idea.txt
rm -r ~/private-notes
```

```text
cat: /home/student/private-notes/idea.txt: Permission denied
```

Why it works: user `nobody` is "other" for this directory. Without the `x` bit for others it cannot even reach the file; the permission of the file itself does not matter. (Your home directory may already block other users, but protecting the inner directory also protects it if the home is ever opened.)

</details>

### Exercise 2 — Write a permission check (medium)

A **check** is a command whose exit status says yes or no. Write a shell check that succeeds only if `~/perm-lab/report.txt` has exactly the mode `600`, and prints `ok` or `FAIL`. Test it for mode `600` and for mode `644`.

<details>
<summary>Hint</summary>

`stat -c '%a' file` prints the octal mode. Compare it with `[ "$(stat -c '%a' file)" = 600 ]`.

</details>

<details>
<summary>Answer</summary>

```bash
mkdir -p ~/perm-lab && cd ~/perm-lab && touch report.txt
chmod 600 report.txt
[ "$(stat -c '%a' report.txt)" = 600 ] && echo ok || echo FAIL
chmod 644 report.txt
[ "$(stat -c '%a' report.txt)" = 600 ] && echo ok || echo FAIL
cd ~ && rm -r ~/perm-lab
```

```text
ok
FAIL
```

Why it works: `$(...)` puts the output of `stat` into the line, `[ a = b ]` compares two strings and sets the exit status, and `&&` / `||` pick the message. Lesson 11.8 turns checks like this into scripts.

</details>

### Exercise 3 — Extend the Rust test (hard)

In `p11-secretfile`, add a second test: the new file must have **no** permission bits for the group or for others (`mode & 0o077 == 0`). Changing the umask inside a test is awkward, so only check the result.

<details>
<summary>Hint</summary>

`0o077` is the mask of the six group and other bits. If the mode ANDed with it is zero, none of those bits is set.

</details>

<details>
<summary>Answer</summary>

```rust
    #[test]
    fn secret_file_has_no_group_or_other_bits() {
        let path = std::env::temp_dir().join(format!("p11-secretfile-bits-{}", std::process::id()));
        write_secret(&path, "x").unwrap();

        let mode = std::fs::metadata(&path).unwrap().permissions().mode();
        assert_eq!(
            mode & 0o077,
            0,
            "group/other bits must be empty, got {:o}",
            mode & 0o777
        );

        std::fs::remove_file(&path).unwrap();
    }
```

Add it inside `mod tests`; `cargo test` should report `2 passed`. Why it works: `& 0o077` isolates the six lower bits; the assertion message prints the real mode if the test ever fails.

</details>

## Quiz

1. What is the UID of `root`, and what is special about it?

<details>
<summary>Answer</summary>

Zero. The kernel does not apply permission checks to it.

</details>

2. Where are users stored, and where are the password hashes?

<details>
<summary>Answer</summary>

Users in `/etc/passwd` (readable by all), groups in `/etc/group`, password hashes in `/etc/shadow` (readable only by root and the group `shadow`).

</details>

3. What does the `x` bit mean on a directory?

<details>
<summary>Answer</summary>

Permission to enter it and to reach its entries by name. Without it, the files inside cannot be opened even when they are readable.

</details>

4. Why `usermod -aG` and not `usermod -G`?

<details>
<summary>Answer</summary>

`-a` appends. Without it, `-G` replaces the user's whole supplementary group list.

</details>

5. Why does `/tmp` have the sticky bit?

<details>
<summary>Answer</summary>

Everyone may create files there, and the sticky bit stops users from deleting each other's files.

</details>

6. Why should a service run as its own system user?

<details>
<summary>Answer</summary>

Least privilege: if the service has a bug or is attacked, the attacker gets only the rights of that small user, not of root or of a person.

</details>

## Summary

- Every process runs as a user; every file has an owner and a group. The kernel uses the numbers UID and GID; names are in `/etc/passwd` and `/etc/group`.
- `root` (UID 0) is not checked. Work as a normal user and use `sudo` for single commands. Follow least privilege.
- `useradd`, `groupadd`, `usermod -aG` and `userdel` manage accounts. Services get a system user with `nologin`.
- Permissions are `rwx` for owner, group and other. For directories, `x` means "enter". Octal: r=4, w=2, x=1.
- `chmod` (symbolic or octal), `chown`, `umask` change who can do what. `namei -l` explains "Permission denied".
- Special bits: sticky (`/tmp`), setuid (`passwd`), setgid (a shared team directory).
- Secrets must be created private from the start: in Rust, `OpenOptions` with `.create_new(true)` and `.mode(0o600)`.

## Sources and further reading

- [Ubuntu Server documentation — User management](https://documentation.ubuntu.com/server/how-to/security/user-management/) — creating and managing users on Ubuntu.
- [chmod(1) manual](https://man7.org/linux/man-pages/man1/chmod.1.html) — modes, symbolic and octal.
- [credentials(7)](https://man7.org/linux/man-pages/man7/credentials.7.html) — how Linux treats user and group IDs.
- [sudo-rs](https://github.com/trifectatechfoundation/sudo-rs) — the Rust implementation of `sudo` that Ubuntu 26.04 uses.
- [Rust standard library — std::os::unix::fs::OpenOptionsExt](https://doc.rust-lang.org/std/os/unix/fs/trait.OpenOptionsExt.html) — the `mode` method used in Section 7.

---

[🏠 Start](../START-HERE.md) · [← 11.3 Text tools, pipes and redirection](03-text-tools-pipes-and-redirection.md) · [📋 Phase 11](README.md) · [11.5 Processes and signals →](05-processes-and-signals.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
