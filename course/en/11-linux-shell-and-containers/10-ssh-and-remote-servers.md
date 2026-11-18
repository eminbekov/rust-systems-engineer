<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 11.9 Robust bash scripts](09-robust-bash-scripts.md) · [📋 Phase 11](README.md) · [11.11 Networking commands →](11-networking-commands.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/11-linux-shell-and-containers/10-ssh-and-remote-servers.md) <!-- nav -->

# Lesson 11.10 — SSH and remote servers

> **Phase 11 — Linux, the shell and containers** · Lesson 10 of 16 · ⏱ about 135 minutes

## What you will learn

- What SSH is, and how your Mac reaches the VM `dev` through it.
- How to make a key pair, put the public key on a server, and log in without a password.
- How to use the `~/.ssh/config` file, the host-key check (`known_hosts`) and `ssh-agent`.
- How to run a command on a server, and copy files and folders with `scp` and `rsync`.
- How to reach a service that is hidden on a server with port forwarding.

## Before you start

- You finished [Lesson 11.9 — Robust bash scripts](09-robust-bash-scripts.md) and the VM `dev` from [Lesson 11.1](01-linux-and-a-virtual-machine.md) is running (`limactl list` shows `Running`).
- You know file permissions ([Lesson 11.4](04-users-groups-and-permissions.md)): SSH refuses keys that other users can read.
- **Where the commands run.** In this lesson the **Mac is the client** and the **VM is the server**, so commands are typed in a normal Mac terminal tab, not inside `limactl shell`. Only the lines marked "on the server" run in the VM. Your Mac already has the programs `ssh`, `ssh-keygen`, `scp` and `rsync`.
- The user name in the examples is `student`; the VM has the same user name as your Mac user ([Lesson 11.1](01-linux-and-a-virtual-machine.md)). Replace it by yours.

## New words

| Word | Meaning |
|---|---|
| **SSH** (Secure Shell) | A protocol and a program for logging in to another computer over the network and running commands there. All traffic is encrypted. |
| **client / server** | The computer you type on is the *client* (program `ssh`). The computer you log in to is the *server* (program `sshd`, the SSH daemon). |
| **key pair** | Two linked files made together: a **private key** (secret, stays with you) and a **public key** (not secret, can be given to anyone). |
| **ed25519** | A modern, short and fast type of key. Use it unless a server is very old. |
| **passphrase** | A password that encrypts your private key file, so that a stolen file is useless without it. |
| **fingerprint** | A short text (such as `SHA256:dKwg/m…`) computed from a key. Two people compare fingerprints to check that they see the same key. |
| **`authorized_keys`** | A file on the server (`~/.ssh/authorized_keys`) that lists the public keys allowed to log in as that user. |
| **host key** | The key pair of the **server** itself. It proves to the client that it talks to the right machine. |
| **`known_hosts`** | A file on the client (`~/.ssh/known_hosts`) with the host keys of servers you have connected to before. |
| **man-in-the-middle** | An attack where someone sits between you and the server and pretends to be the server. The host-key check protects against it. |
| **`ssh-agent`** | A small program that keeps your decrypted private key in memory, so you type the passphrase once. |
| **port** | A number from 0 to 65535 that selects a program on a computer. SSH uses port 22 by default ([Lesson 11.11](11-networking-commands.md) explains ports). |
| **loopback** (`127.0.0.1`, `localhost`) | The address of "this same computer". A service that listens only on loopback cannot be reached from outside. |
| **port forwarding** (tunnel) | Letting SSH carry traffic of another program: a port on one computer is connected, through the encrypted SSH connection, to a port on the other. |
| **`rsync`** | A program that copies only the parts of files that changed. Often used with SSH. |

## 1. What SSH is, and how it reaches the VM

Years ago people logged in to remote machines with programs that sent everything, passwords included, as plain text. **SSH** replaces them: client and server first agree on a secret key, then everything is encrypted. Today it is the normal way to administer Linux servers, to copy files to them, and (in the project of Lesson 11.15) to deploy programs.

```text
 your Mac (client)                          server
 +-------------------+   encrypted SSH    +----------------------+
 |  ssh, scp, rsync  | =================> |  sshd  (port 22)     |
 |  private key      |                    |  authorized_keys     |
 |  known_hosts      |                    |  host key            |
 +-------------------+                    +----------------------+
```

Your VM already runs an SSH server. When you type `limactl shell dev`, Lima uses SSH inside to enter the VM. Lima also forwards a port of your Mac (`127.0.0.1`, a number such as 60022) to the SSH port of the VM, and writes a ready-made client configuration. Ask for its path (the command is from the [Lima SSH documentation](https://lima-vm.io/docs/usage/ssh/)):

```bash
limactl list --format '{{.SSHConfigFile}}' dev
limactl list --format '{{.SSHLocalPort}}' dev
```

Example output (yours will look similar; the number and the user name differ):

```text
/Users/student/.lima/dev/ssh.config
60022
```

Now log in with that file (`-F` means "use this configuration file"; `lima-dev` is the name of the VM inside it):

```bash
ssh -F /Users/student/.lima/dev/ssh.config lima-dev
```

You get the VM prompt, the same as with `limactl shell dev`. Type `exit` to come back. The file `ssh.config` looks like this (shortened example; Lima may add more lines):

```text
Host lima-dev
  IdentityFile "/Users/student/.lima/_config/user"
  User student
  Hostname 127.0.0.1
  Port 60022
```

It says: *to reach `lima-dev`, connect to this computer (`127.0.0.1`) on port 60022 as user `student`, using the key file `~/.lima/_config/user`.* That key belongs to Lima. In the next sections you will make **your own** key and your own configuration, which is what you do on every real server.

> 💡 **Tip:** Lima's manual also shows how to make `ssh lima-dev` work without `-F`: add the line `Include ~/.lima/*/ssh.config` at the very top of your `~/.ssh/config`. You may do that later; the lesson uses `-F` so that nothing in your own SSH setup changes until you decide.

## 2. A key pair

A password can be guessed or stolen from the network. A **key pair** is stronger. You make two files. The server gets the **public** key and uses it to ask a question that only the owner of the matching **private** key can answer (the exact mathematics is in Lessons 10.9 and 10.10). The private key never leaves your computer.

```bash
mkdir -p ~/.ssh && chmod 700 ~/.ssh
ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519_course -C "course key"
```

**What happens:**

1. `mkdir -p ~/.ssh && chmod 700 ~/.ssh` makes sure the folder exists and only you can enter it.
2. `-t ed25519` selects the key type. `-f` is the file name of the **private** key. We choose a new name, `id_ed25519_course`, so that a key you may already have is never overwritten. If the file exists, `ssh-keygen` asks before replacing it: answer `n`.
3. `-C "course key"` is a comment stored in the public key, to recognise it later.
4. `ssh-keygen` asks twice for a **passphrase**. Choose a good one, and do not leave it empty on your own laptop.

```text
Generating public/private ed25519 key pair.
Enter passphrase for "/Users/student/.ssh/id_ed25519_course" (empty for no passphrase):
Enter same passphrase again:
Your identification has been saved in /Users/student/.ssh/id_ed25519_course
Your public key has been saved in /Users/student/.ssh/id_ed25519_course.pub
The key fingerprint is:
SHA256:dKwg/mn4QANWK3exzGbEL4/V6PQuMHA9Z8G52Lx/Dmk course key
The key's randomart image is:
+--[ED25519 256]--+
|    ..o  . .     |
|   . =.o .+      |
|  + + Oo.*oo     |
| . =.=ooOoB      |
|    +o BS= .     |
|   . ++.o o  .   |
|    o +o . .E    |
|     +  . .....  |
|      .  .   o.  |
+----[SHA256]-----+
```

(The passphrase is not shown while you type. The fingerprint and the picture are different for every key.) Look at the two files:

```bash
ls -l ~/.ssh/id_ed25519_course*
cat ~/.ssh/id_ed25519_course.pub
```

```text
-rw-------  1 student  staff  444 Nov 18 07:21 /Users/student/.ssh/id_ed25519_course
-rw-r--r--  1 student  staff   92 Nov 18 07:21 /Users/student/.ssh/id_ed25519_course.pub
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIDRRpgo4mvqU4ryMgO/X0/m3CyoR/scpsTXxMw3nNtOG course key
```

The private key has mode `600` ([Lesson 11.4](04-users-groups-and-permissions.md)): only you can read it. The public key is one line: the type, the key, and your comment. **You may send the `.pub` file anywhere. Never send or upload the file without `.pub`.**

> 🔒 **Security:** If someone copies your private key and it has no passphrase, they can log in as you everywhere the public key is installed. With a passphrase they must break the encryption first. Never put a private key in Git, in a chat, or in a container image ([Lesson 10.8](../10-security-fundamentals/08-secrets-management.md)). SSH itself refuses a private key with loose permissions:
>
> ```text
> WARNING: UNPROTECTED PRIVATE KEY FILE!
> Permissions 0644 for '/Users/student/.ssh/id_ed25519_course' are too open.
> This private key will be ignored.
> ```
>
> The cure is `chmod 600 ~/.ssh/id_ed25519_course`.

## 3. Install the public key and connect

For the server to accept your key, its line must be in `~/.ssh/authorized_keys` **of the user you log in as**, on the server. We have no password, so we use the working Lima connection to append it. The text is sent through a pipe ([Lesson 11.3](03-text-tools-pipes-and-redirection.md)): `cat` writes the public key to the standard input of `ssh`, and the remote shell appends it:

```bash
cfg=$(limactl list --format '{{.SSHConfigFile}}' dev)
cat ~/.ssh/id_ed25519_course.pub | ssh -F "$cfg" lima-dev \
  'mkdir -p ~/.ssh && chmod 700 ~/.ssh && cat >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys'
```

**What happens:**

1. `cfg=...` stores Lima's config path in a shell variable ([Lesson 11.8](08-bash-scripting-basics.md)); `"$cfg"` is quoted ([Lesson 11.9](09-robust-bash-scripts.md)).
2. The text in single quotes is **one command line sent to the server**. There the shell creates `~/.ssh`, **appends** (`>>`) the key from standard input to `authorized_keys`, and sets the modes. `sshd` ignores the file if the folder or file can be written by others.
3. The backslash at the end of the first line only continues the command on the next line.

Check on the server how many keys it now has (we cut each line to 40 characters to keep the secret part short):

```bash
ssh -F "$cfg" lima-dev 'wc -l ~/.ssh/authorized_keys; cut -c1-40 ~/.ssh/authorized_keys'
```

```text
2 /home/student/.ssh/authorized_keys
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAICeo
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIDRR
```

(One line is Lima's key, the other is yours; the order may differ.)

Now connect with **your** key. Replace the port by the one from `limactl list --format '{{.SSHLocalPort}}' dev`:

```bash
ssh -p 60022 -i ~/.ssh/id_ed25519_course student@127.0.0.1
```

`-p` is the port, `-i` the private key file, and `student@127.0.0.1` means "user `student` on the computer `127.0.0.1`". The first time, SSH asks something important:

```text
The authenticity of host '[127.0.0.1]:60022 ([127.0.0.1]:60022)' can't be established.
ED25519 key fingerprint is: SHA256:tI7wAQhRCc1SVRgGMjJoJz0zJ/h12bzaKinlaWSmyGE
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '[127.0.0.1]:60022' (ED25519) to the list of known hosts.
Enter passphrase for key '/Users/student/.ssh/id_ed25519_course':
```

**What happens:**

1. The client has never seen this server's **host key**. It shows its fingerprint and asks you to confirm it. On a real server you compare it with the fingerprint that the administrator gives you through another channel. Here you can check it yourself, on the server: `limactl shell dev ssh-keygen -l -f /etc/ssh/ssh_host_ed25519_key.pub` prints the same `SHA256:` text. After `yes`, the key is stored in `~/.ssh/known_hosts`.
2. Next the client proves that it owns the private key. It asks for the passphrase that unlocks the key *on your Mac*; the passphrase is never sent to the server.
3. You are in. `exit` leaves.

**If the host key changes later**, SSH stops and shouts:

```text
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
@    WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!     @
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
IT IS POSSIBLE THAT SOMEONE IS DOING SOMETHING NASTY!
...
Offending ED25519 key in /Users/student/.ssh/known_hosts:1
  remove with:
  ssh-keygen -f '/Users/student/.ssh/known_hosts' -R '[127.0.0.1]:60022'
Host key for [127.0.0.1]:60022 has changed and you have requested strict checking.
Host key verification failed.
```

This is the man-in-the-middle alarm. On a real server, **stop and ask** why the key changed. With the Lima VM the cause is harmless: you deleted and recreated the VM (a new VM has new host keys), or another VM now uses the same port on `127.0.0.1`. Then remove the old line with the command SSH prints (`ssh-keygen -R ...`) and connect again.

> ⚠️ **Common mistake:** Typing `yes` without reading. The prompt exists to protect you. Also never turn the check off with `StrictHostKeyChecking no` for real servers. (Lima's own config file does it for the local VM, where the risk is different.)

## 4. The configuration file

Typing `-p 60022 -i ... student@127.0.0.1` every time is tiresome and error-prone. Put these settings under a short name in `~/.ssh/config`. Create it with an editor, or:

```bash
cat >> ~/.ssh/config <<'EOF'
Host dev
    HostName 127.0.0.1
    Port 60022
    User student
    IdentityFile ~/.ssh/id_ed25519_course
    IdentitiesOnly yes
EOF
chmod 600 ~/.ssh/config
```

`>>` appends, so your existing entries stay. Each line is a *keyword* and a value:

| Keyword | Meaning |
|---|---|
| `Host dev` | The **alias**: what you type after `ssh`. The lines below belong to it until the next `Host`. |
| `HostName` | The real address to connect to. |
| `Port` | The port of `sshd` (22 if you leave it out). |
| `User` | The user name on the server. |
| `IdentityFile` | Which private key to use. |
| `IdentitiesOnly yes` | Offer only this key, not every key that is in the agent. (Too many offered keys can make a server hang up.) |

Check what SSH will really use with `ssh -G`, which prints the final settings without connecting:

```bash
ssh -G dev | grep -E '^(hostname|port|user|identityfile) '
```

```text
user student
hostname 127.0.0.1
port 60022
identityfile ~/.ssh/id_ed25519_course
```

From now on `ssh dev` is enough, and so are `scp` and `rsync` with `dev:` as the host. If it fails, `ssh -v dev` prints each step (`Offering public key`, `Server accepts key`, `Permission denied`) and almost always shows the reason.

> ⚠️ **Common mistake:** The Lima port can change after the VM was stopped and started or recreated. If `ssh dev` says `Connection refused`, run `limactl list --format '{{.SSHLocalPort}}' dev` again and fix the `Port` line.

## 5. The agent: type the passphrase once

Without help, SSH asks for your passphrase at every connection. `ssh-agent` keeps the unlocked key in memory. Two ways to use it:

```bash
ssh-add ~/.ssh/id_ed25519_course     # type the passphrase once
ssh-add -l                           # list the keys in the agent
```

```text
Identity added: /Users/student/.ssh/id_ed25519_course (course key)
256 SHA256:dKwg/mn4QANWK3exzGbEL4/V6PQuMHA9Z8G52Lx/Dmk course key (ED25519)
```

or let SSH add the key at the first use. Add one line to the `Host dev` block:

```text
    AddKeysToAgent yes
```

Then the first `ssh dev` asks for the passphrase, and the next ones do not:

```text
Enter passphrase for key '/Users/student/.ssh/id_ed25519_course':
lima-dev
```

`ssh-add -d file` removes one key from the agent, and `ssh-add -D` removes all keys. macOS starts an agent for you in every login session, so you usually do not start it. (On a Mac the option `UseKeychain yes` in `~/.ssh/config` also stores the passphrase in the macOS keychain; it is specific to Apple's `ssh` and described in `man ssh_config` on your Mac.) The agent is also what makes scripts possible: a deploy script that runs `ssh` with `-o BatchMode=yes` cannot type a passphrase, but it can use a key that the agent already holds.

> 🔒 **Security:** The option `ForwardAgent yes` lets the *server* use your agent. Everyone who is administrator on that server can then use your keys while you are connected. Leave it off, and use `ProxyJump` (`ssh -J jump dev`) when you must pass through a middle machine.

## 6. Run a command on the server

`ssh dev 'command'` runs one command there, prints its output here, and ends. This is how scripts use servers:

```bash
ssh dev hostname
ssh dev 'echo "I am $(whoami) in $PWD"'
ssh dev 'exit 3'; echo "status: $?"
```

```text
lima-dev
I am student in /home/student
status: 3
```

**What happens:**

1. The command runs in a shell on the server, in your home folder. Its output comes back through the encrypted connection.
2. **The exit status of the remote command becomes the exit status of `ssh`** ([Lesson 11.5](05-processes-and-signals.md)). Scripts with `set -e` ([Lesson 11.9](09-robust-bash-scripts.md)) stop when a remote command fails. (If SSH itself fails, the status is 255.)

The quotes decide *where* a `$` is expanded. Compare:

```bash
WHERE=mac
ssh dev "echo $WHERE"
ssh dev 'echo ${WHERE:-unset}'
```

```text
mac
unset
```

Double quotes: **your** shell replaces `$WHERE` before `ssh` starts, the server only sees `echo mac`. Single quotes: the text goes to the server unchanged, and the server's shell expands it; the variable does not exist there. Use single quotes for commands that must use the server's values.

Standard input and output travel through SSH too, so pipes work in both directions (the idea of [Lesson 11.3](03-text-tools-pipes-and-redirection.md)):

```bash
echo "line one" | ssh dev 'cat > ~/note.txt'
ssh dev 'cat ~/note.txt'
ssh dev 'tar -czf - -C /etc hostname' | tar -tzf -
```

```text
line one
hostname
```

The first line sends local text into a file on the server. The last one packs a file *on the server* into a stream (`-f -` means standard output) and lists the archive *here*. Remove the test file with `ssh dev 'rm ~/note.txt'`.

## 7. Copy files: `scp` and `rsync`

**`scp`** (secure copy) works like `cp` but with `host:path` on one side:

```bash
mkdir -p ~/work/app/src && cd ~/work
echo 'fn main() {}' > app/src/main.rs
echo hello > hello.txt
scp hello.txt dev:~/
scp -r app dev:~/
scp dev:/etc/hostname ./remote-hostname.txt
```

```text
hello.txt                                     100%    6    43.1KB/s   00:00
```

`dev:~/` is "the home folder on `dev`". `-r` copies a folder with everything in it. The third command copies **from** the server to the current folder. (`scp` prints one progress line for every file when you run it in a terminal.)

**`rsync`** is better for folders you copy again and again, for example the source of a project: the first run copies everything, later runs send only what changed. It needs to be installed on both sides. On the Mac it exists; in the VM install it once: `ssh dev 'sudo apt-get update && sudo apt-get install -y rsync'` ([Lesson 11.7](07-package-managers.md)). Try it with a small project:

```bash
mkdir -p proj/src && echo 'fn main() {}' > proj/src/main.rs && echo 'v1' > proj/Cargo.toml
mkdir proj/target && echo junk > proj/target/big.bin
rsync -av --exclude target proj/ dev:~/proj/
```

```text
sending incremental file list
created directory /home/student/proj
./
Cargo.toml
src/
src/main.rs

sent 249 bytes  received 106 bytes  710.00 bytes/sec
total size is 16  speedup is 0.05
```

**What happens:**

1. `-a` ("archive") copies folders recursively and keeps modes, times and links. `-v` lists what it does. `--exclude target` skips the build folder, which is big and useless on another system.
2. **The trailing slash matters.** `proj/` means "the *contents* of `proj`"; `proj` without a slash would create `~/proj/proj`. The same on the target side. Try both and compare with `ssh dev 'find ~/slash1 ~/slash2 -type f'`:

   ```bash
   mkdir -p t1/dir && echo x > t1/dir/f
   rsync -a t1/dir dev:~/slash1/
   rsync -a t1/dir/ dev:~/slash2/
   ```

   ```text
   /home/student/slash1/dir/f
   /home/student/slash2/f
   ```

3. Change a file and run again with `-n` (`--dry-run`: say what *would* happen, [Lesson 11.9](09-robust-bash-scripts.md)). Only the changed file is listed:

   ```bash
   echo 'version two' > proj/Cargo.toml
   rsync -avn --exclude target proj/ dev:~/proj/
   ```

   ```text
   sending incremental file list
   Cargo.toml

   sent 140 bytes  received 20 bytes  320.00 bytes/sec
   total size is 25  speedup is 0.16 (DRY RUN)
   ```

4. `--delete` also removes files on the target that no longer exist here, so the target becomes an exact mirror. It is powerful and dangerous. **Always do a `-n` run first and read the `deleting` lines.**

> 💡 **Tip:** `limactl copy` copies between your Mac and the VM without any SSH setup: `limactl copy hello.txt dev:/tmp/`. It uses `rsync` or `scp` inside ([Lima reference](https://lima-vm.io/docs/reference/limactl_copy/)). You will use `rsync` over your own `dev` alias in the project (Lesson 11.15).

Clean up: `ssh dev 'rm -rf ~/proj ~/hello.txt ~/app ~/slash1 ~/slash2'`.

## 8. Port forwarding

Suppose a service on the server listens only on the loopback address, for example a database or an admin page on port 8080. That is a good, safe setting: nobody on the network can reach it. But you want to look at it from your Mac. SSH can carry the connection.

**Local forwarding (`-L`)** opens a port on **your** computer and sends everything to an address as seen from the **server**:

```bash
ssh -N -f -L 18080:127.0.0.1:8080 dev
curl -s http://127.0.0.1:18080/
```

```text
secret page, only on the server loopback
```

```text
   your Mac                              server
   curl --> 127.0.0.1:18080 ==SSH==> sshd --> 127.0.0.1:8080 (the hidden service)
```

**What happens:**

1. `-L 18080:127.0.0.1:8080` reads as: *listen on my port 18080; send each connection through SSH; the server connects to `127.0.0.1:8080`* (from the server's point of view, so this is the server's own loopback).
2. `-N` means "do not run a command, only forward" and `-f` sends `ssh` to the background after login.
3. `curl` talks to your own port 18080 and gets the answer of the hidden service. Without the tunnel, `curl` from another machine to the server's port 8080 fails with status 7 (`Connection refused`).

Find and stop the tunnel when you are done (a forgotten tunnel stays open):

```bash
pgrep -af 'ssh .*-L 18080'
pkill -f 'ssh .*-L 18080'
```

**Remote forwarding (`-R`)** is the opposite: it opens a port on the **server** that leads back to your computer. Useful to show a service on your laptop to a server for a short time:

```bash
ssh -N -f -R 9000:127.0.0.1:8000 dev
ssh dev 'curl -s http://127.0.0.1:9000/'
```

```text
page from my laptop
```

Here a web server on your Mac's port 8000 became reachable on the server as `127.0.0.1:9000`. Both kinds listen on loopback only by default, so tunnels do not open your machines to the network. You can store a tunnel in the config with `LocalForward 18080 127.0.0.1:8080` (or `RemoteForward`) inside a `Host` block, and add `ExitOnForwardFailure yes` so that `ssh` fails if the port is busy.

## 9. A safer server

Everything above worked because the **server** accepts keys. A server that is reachable from the internet also needs a few lines in its `sshd` configuration. Ubuntu reads extra files from `/etc/ssh/sshd_config.d/*.conf`. In the VM (a server) you could create `/etc/ssh/sshd_config.d/10-hardening.conf`:

```text
PasswordAuthentication no
KbdInteractiveAuthentication no
PermitRootLogin no
MaxAuthTries 3
```

These lines turn off passwords (bots try millions of them), forbid logging in directly as `root` (use a normal user and `sudo`, [Lesson 11.4](04-users-groups-and-permissions.md)), and allow three attempts per connection. **Always check the file before you reload `sshd`**, because a mistake can lock you out:

```bash
sudo sshd -t && echo "config ok"
sudo sshd -T | grep -E '^(passwordauthentication|permitrootlogin|maxauthtries) '
sudo systemctl reload ssh
```

```text
config ok
maxauthtries 3
permitrootlogin no
passwordauthentication no
```

`sshd -t` only tests the files and prints nothing when they are fine. With a typo it fails and tells you the line, for example `line 5: Bad configuration option: Foo`. `sshd -T` prints the settings that are really in force. Keep your current session open while you test a new login in a second terminal; the unit is named `ssh` on Ubuntu ([Lesson 11.6](06-services-with-systemd.md)).

> 🔒 **Security:** Key-only login, no root login, and updates ([Lesson 11.7](07-package-managers.md)) already stop most attacks on SSH. Do not change the port expecting security; it only reduces log noise.

## Common mistakes

1. **`Permission denied (publickey)`.** The server did not accept any key. *Fix:* run `ssh -v dev` and look at `Offering public key`; check that the right `.pub` line is in `authorized_keys`, that `~/.ssh` is `700` and `authorized_keys` is `600` on the server, and that you use the right user name.
2. **The private key is readable by others** (`UNPROTECTED PRIVATE KEY FILE`). *Fix:* `chmod 600` the key.
3. **Sending the wrong file.** `scp ~/.ssh/id_ed25519_course dev:` copies the **private** key to a server. *Fix:* only ever copy `.pub` files; look twice before pressing Enter.
4. **`rsync` without the trailing slash** creates `proj/proj`. *Fix:* write `proj/` and `dev:~/proj/`, and test with `-n`.
5. **`rsync --delete` without a dry run.** *Fix:* `-n` first. A wrong target path deletes real files. Also stop tunnels you no longer need: `pgrep -af 'ssh .*-L'` and `pkill`.

## Try it yourself

1. Make the key `~/.ssh/id_ed25519_course` and install it in the VM as in Sections 2 and 3. Log in once with `-p`, `-i` and `student@127.0.0.1`.
2. Create the `Host dev` entry. Check it with `ssh -G dev`, then run `ssh dev hostname`.
3. Start a tiny web server in the VM that listens only on loopback: `ssh dev 'cd /tmp && python3 -m http.server 8080 --bind 127.0.0.1'` (stop it with `Ctrl + C` when you finish). In a second Mac tab open the tunnel of Section 8 and fetch the page with `curl`. (Python is installed in Ubuntu cloud images; if `python3` is missing, run `sudo apt-get install -y python3` first.)
4. Copy a small folder to the VM with `rsync -avn` first, then for real, and run the same command again: what changes in the output?

## Exercises

### Exercise 1 — Verify a fingerprint (easy)

Connect to a host you have not used before in a way that makes SSH show its fingerprint (for example `ssh -o UserKnownHostsFile=/tmp/kh-test -p 60022 student@127.0.0.1`, answering `no`). Then compute the host key fingerprint on the server and compare. What does it prove if they are equal?

<details>
<summary>Hint</summary>

On the server the host keys are files `/etc/ssh/ssh_host_*_key.pub`. `ssh-keygen -l -f FILE` prints a fingerprint. Use `ssh dev` for the second command.

</details>

<details>
<summary>Answer</summary>

```bash
ssh -o UserKnownHostsFile=/tmp/kh-test -p 60022 student@127.0.0.1    # answer: no
ssh dev ssh-keygen -l -f /etc/ssh/ssh_host_ed25519_key.pub
```

The second command prints a line such as `256 SHA256:tI7wAQhRCc1SVRgGMjJoJz0zJ/h12bzaKinlaWSmyGE root@... (ED25519)`.

Why it works: equal fingerprints prove that the server you reached holds the private half of the host key you checked over a channel you already trust (your existing login). A man in the middle would have a different key and a different fingerprint. Remove the test file afterwards: `rm /tmp/kh-test`.

</details>

### Exercise 2 — A checked connection script and its test (medium)

Write `check-ssh.sh HOST` in `code/p11-scripts`: it connects with `-o BatchMode=yes -o ConnectTimeout=5`, runs `echo ok`, prints `HOST: reachable` and exits 0, or prints `HOST: NOT reachable (reason)` to standard error and exits 1. Then write `test-check-ssh.sh GOOD_HOST` that checks status 0 for the good host and status 1 for `no-such-host.invalid`.

<details>
<summary>Hint</summary>

`if out=$(ssh ... 2>&1); then` captures the output and uses the status at the same time. In the test, store the status with `rc=0; bash check-ssh.sh X || rc=$?`.

</details>

<details>
<summary>Answer</summary>

```bash
#!/usr/bin/env bash
# check-ssh.sh HOST - exit 0 if we can log in to HOST with a key, 1 otherwise.
set -u

host=${1:?usage: check-ssh.sh HOST}

if out=$(ssh -o BatchMode=yes -o ConnectTimeout=5 "$host" 'echo ok' 2>&1); then
    echo "$host: reachable"
    exit 0
fi
echo "$host: NOT reachable ($out)" >&2
exit 1
```

```bash
#!/usr/bin/env bash
# test-check-ssh.sh GOOD_HOST - needs a working host alias in ~/.ssh/config
set -u
here=$(cd "$(dirname "$0")" && pwd)
good=${1:?usage: test-check-ssh.sh GOOD_HOST}
failures=0

rc=0; bash "$here/check-ssh.sh" "$good" > /dev/null 2>&1 || rc=$?
if (( rc == 0 )); then echo "ok   - good host"; else echo "FAIL - good host (status $rc)"; failures=$((failures + 1)); fi

rc=0; bash "$here/check-ssh.sh" no-such-host.invalid > /dev/null 2>&1 || rc=$?
if (( rc == 1 )); then echo "ok   - bad host"; else echo "FAIL - bad host (status $rc)"; failures=$((failures + 1)); fi

(( failures == 0 ))
```

Run `bash test-check-ssh.sh dev` (with the key in the agent, because `BatchMode` cannot ask for a passphrase):

```text
ok   - good host
ok   - bad host
```

Why it works: `BatchMode=yes` forbids all questions, so a script fails fast instead of waiting; `ConnectTimeout` limits the waiting for a dead host. The `.invalid` domain is reserved and never resolves, so the failure is certain. The last line makes the test's own status 0 or 1.

</details>

## Quiz

1. Which of the two files of a key pair may you give to other people, and where does it go on the server?

<details>
<summary>Answer</summary>

The public key (`.pub`). Its line goes into `~/.ssh/authorized_keys` of the user you want to log in as, on the server. The private key never leaves your computer.

</details>

2. What does the `known_hosts` file protect you from?

<details>
<summary>Answer</summary>

From connecting to an impostor server (man-in-the-middle). The client remembers the host key of each server; if it later differs, SSH refuses and warns.

</details>

3. What is the difference between `ssh dev "echo $HOME"` and `ssh dev 'echo $HOME'`?

<details>
<summary>Answer</summary>

With double quotes your local shell replaces `$HOME` by your local home folder before `ssh` runs. With single quotes the text is sent unchanged and the server's shell expands it to the server's home folder.

</details>

4. You run `rsync -a proj dev:~/proj/`. Where does `main.rs` end up, and how do you fix it if you wanted `~/proj/src/main.rs`?

<details>
<summary>Answer</summary>

At `~/proj/proj/src/main.rs`, because `proj` without a slash copies the folder itself. Write `rsync -a proj/ dev:~/proj/`.

</details>

5. What does `ssh -N -f -L 18080:127.0.0.1:8080 dev` do?

<details>
<summary>Answer</summary>

It opens port 18080 on your computer (on loopback). Connections to it are carried through SSH to `dev`, which connects to `127.0.0.1:8080` on the server side. `-N` runs no remote command and `-f` goes to the background.

</details>

## Summary

- SSH gives an encrypted login, remote commands, file copy and tunnels. Lima gives the VM an SSH server and a ready config file (`ssh -F ~/.lima/dev/ssh.config lima-dev`).
- Make an ed25519 key pair with a passphrase; the public key goes into `~/.ssh/authorized_keys` on the server, the private key never leaves your computer, and its mode must be `600`.
- Check host-key fingerprints at the first connection. A changed host key is an alarm, not an annoyance.
- Put host details into `~/.ssh/config` (`Host`, `HostName`, `Port`, `User`, `IdentityFile`); `ssh -G` shows the result and `ssh -v` explains failures.
- `ssh-agent` (`ssh-add`, `AddKeysToAgent`) keeps the unlocked key so you type the passphrase once and scripts can work.
- `ssh host 'cmd'` passes the exit status back; single quotes expand variables on the server, double quotes on your side.
- `scp` copies simply, `rsync -av` copies only changes; mind the trailing slash and use `-n` before `--delete`.
- `-L` and `-R` forward ports over the encrypted connection; on servers use keys only, no root login, and test `sshd` configuration with `sshd -t`.

## Sources and further reading

- [OpenSSH manual pages](https://www.openssh.com/manual.html) — the index of the official documentation of OpenSSH.
- [ssh(1)](https://man.openbsd.org/ssh) and [ssh_config(5)](https://man.openbsd.org/ssh_config) — every command-line option and every config keyword.
- [ssh-keygen(1)](https://man.openbsd.org/ssh-keygen) and [ssh-agent(1)](https://man.openbsd.org/ssh-agent) — key types, fingerprints and the agent.
- [sshd_config(5)](https://man.openbsd.org/sshd_config) — all server settings.
- [scp(1)](https://man.openbsd.org/scp) and [rsync(1)](https://man7.org/linux/man-pages/man1/rsync.1.html) — the copy tools in detail.
- [Lima documentation: SSH](https://lima-vm.io/docs/usage/ssh/) — how Lima exposes SSH for the VM.

---

[🏠 Start](../START-HERE.md) · [← 11.9 Robust bash scripts](09-robust-bash-scripts.md) · [📋 Phase 11](README.md) · [11.11 Networking commands →](11-networking-commands.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
