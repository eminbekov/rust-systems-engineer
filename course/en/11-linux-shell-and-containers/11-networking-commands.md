<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 11.10 SSH and remote servers](10-ssh-and-remote-servers.md) · [📋 Phase 11](README.md) · [11.12 Containers and Docker basics →](12-containers-and-docker-basics.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/11-linux-shell-and-containers/11-networking-commands.md) <!-- nav -->

# Lesson 11.11 — Networking commands

> **Phase 11 — Linux, the shell and containers** · Lesson 11 of 16 · ⏱ about 120 minutes

## What you will learn

- How to read the network settings of a Linux machine: addresses, routes, names.
- How to check step by step why a connection fails: address, name, route, port, program.
- How to see which programs listen on which ports (`ss`), and how to test a port (`nc`).
- How to talk HTTP from the command line with `curl`, and how to read its exit codes.
- Why a Rust server must choose its bind address (`127.0.0.1` or `0.0.0.0`) with care.

## Before you start

- You finished [Lesson 11.10 — SSH and remote servers](10-ssh-and-remote-servers.md). You know what a port is in general terms (a number that selects a program).
- You know processes and exit statuses ([Lesson 11.5](05-processes-and-signals.md)) and pipes ([Lesson 11.3](03-text-tools-pipes-and-redirection.md)).
- All commands run **inside the VM** (`limactl shell dev`). Most tools are already installed there. Install the rest once (`apt` is explained in [Lesson 11.7](07-package-managers.md)):

```bash
sudo apt-get update
sudo apt-get install -y bind9-dnsutils netcat-openbsd traceroute
```

- Rust is installed in the VM ([Lesson 11.5](05-processes-and-signals.md)). The small Rust program of Section 7 is written on the Mac in `code/p11-bind` and copied to the VM as in that lesson.
- The outputs below were produced on Ubuntu 26.04. Addresses and times are different on your machine; the shape is the same.

## New words

| Word | Meaning |
|---|---|
| **IP address** (Internet Protocol address) | The number that identifies a network interface, for example `172.30.0.2` (IPv4: four numbers) or `::1` (IPv6: longer). |
| **interface** | A network connection of the computer, such as `eth0` (a cable or virtual cable) or `lo` (the loopback). |
| **prefix** (`/24`, `/16`) | How many bits at the start of an address name the network. `172.30.0.2/16` means: the network is `172.30.0.0` and all addresses `172.30.x.y` are on the same local network. |
| **gateway** (router) | The computer that forwards packets to other networks. The *default route* says: "everything else goes to the gateway". |
| **DNS** (Domain Name System) | The system that turns names such as `example.com` into IP addresses. A *resolver* program asks a *name server*. |
| **TCP** (Transmission Control Protocol) | The protocol that gives a reliable two-way connection between two programs. HTTP and SSH use it. |
| **UDP** (User Datagram Protocol) | A simpler protocol that sends single messages without a connection. DNS usually uses it. |
| **socket** | One end of a network connection, identified by an address and a port. A program that *listens* has a socket waiting for connections. |
| **bind** | To attach a listening socket to an address and a port. The *bind address* decides which networks can reach the program. |
| **HTTP** (HyperText Transfer Protocol) | The text protocol of the web: a request (`GET /`) and a response (a status code such as `200`, headers, a body). |
| **status code** | The number in an HTTP response: `200` ok, `404` not found, `500` server error. |
| **TLS** (Transport Layer Security) | The encryption layer behind `https://` ([Lesson 10.11](../10-security-fundamentals/11-tls-and-certificates.md)). |
| **ICMP** (Internet Control Message Protocol) | The small messages used by `ping` and `traceroute`. |

## 1. A ladder for broken connections

"The service does not answer" can have six different causes. The commands of this lesson test them from the bottom up. Always climb the ladder in this order; the first step that fails is your problem:

| Step | Question | Command | Typical failure |
|---|---|---|---|
| 1 | Does this machine have an address? | `ip -br addr` | no address on `eth0` |
| 2 | Does it know a way out? | `ip route` | no `default` route |
| 3 | Does the name become an address? | `getent hosts NAME`, `dig` | `Could not resolve host` |
| 4 | Is the other machine reachable? | `ping`, `traceroute` | `100% packet loss` |
| 5 | Is a program listening on the port? | `nc -zv HOST PORT`, `ss -ltn` | `Connection refused` |
| 6 | Does the program answer correctly? | `curl -v URL` | wrong status code, empty reply |

Two failures look different and mean different things. **Connection refused** comes back at once: the machine is there, but nothing listens on that port. A **timeout** means silence: packets are lost on the way or a firewall (a filter that drops packets) throws them away.

## 2. Addresses and routes

```bash
ip -br addr show up
ip route
```

Example output:

```text
lo               UNKNOWN        127.0.0.1/8 ::1/128
eth0@if3158      UP             172.30.0.2/16
default via 172.30.0.1 dev eth0
172.30.0.0/16 dev eth0 proto kernel scope link src 172.30.0.2
```

**What happens:**

1. `ip` is the tool to inspect and change the network of Linux. `-br` ("brief") prints one line per interface; `show up` hides interfaces that are switched off.
2. `lo` is the **loopback**. Its address `127.0.0.1` (and `::1` in IPv6) always means "this very computer". Traffic to it never leaves the machine.
3. `eth0` is the real connection. `172.30.0.2/16` is its address and prefix. In your VM it will show other numbers (Lima usually uses `192.168.5.x`).
4. `ip route` is the routing table: `172.30.0.0/16 dev eth0` says addresses of the own network are reached directly through `eth0`; `default via 172.30.0.1` says *everything else goes to the gateway* `172.30.0.1`.

Ask which way a packet to a certain address takes:

```bash
ip route get 1.1.1.1
```

```text
1.1.1.1 via 172.30.0.1 dev eth0 src 172.30.0.2 uid 0
    cache
```

## 3. Names: `/etc/hosts`, `/etc/resolv.conf`, `dig`

Programs work with names; the network works with addresses. Linux resolves a name in this order: first the file `/etc/hosts` (a small list that you control), then the name server named in `/etc/resolv.conf`.

```bash
getent hosts example.com
dig +short example.com
```

```text
172.66.147.243  example.com
104.20.23.154   example.com
172.66.147.243
104.20.23.154
```

`getent hosts` asks the system in the same way as your programs do (it also reads `/etc/hosts`). `dig` is the diagnostic tool: it asks a name server directly. `+short` prints only the answers. One name can have several addresses.

For the full answer, look at `dig example.com A` (shortened):

```text
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 2327
;; QUESTION SECTION:
;example.com.			IN	A

;; ANSWER SECTION:
example.com.		307	IN	A	172.66.147.243
example.com.		307	IN	A	104.20.23.154

;; Query time: 2 msec
;; SERVER: 127.0.0.11#53(127.0.0.11) (UDP)
```

**What happens:** `A` is the record type "IPv4 address" (`AAAA` is IPv6, `MX` mail servers, `NS` the name servers of the domain). `307` is the number of seconds that the answer may be cached (TTL, time to live). `SERVER:` is the name server that answered; it comes from `/etc/resolv.conf`. `status: NOERROR` is success; for a name that does not exist you see `status: NXDOMAIN`, and `getent hosts` prints nothing and returns status 2.

You can override a name for this machine with `/etc/hosts` (needs `sudo`):

```bash
echo "127.0.0.1 myservice.test" | sudo tee -a /etc/hosts
getent hosts myservice.test
```

```text
127.0.0.1       myservice.test
```

(The name ends in `.test`, a domain reserved for such experiments. Remove the line again with an editor, or `sudo sed -i '/myservice.test/d' /etc/hosts`.)

## 4. Is it reachable? `ping` and `traceroute`

```bash
ping -c 3 localhost
ping -c 2 example.com
```

```text
PING example.com (172.66.147.243) 56(84) bytes of data.
64 bytes from 172.66.147.243: icmp_seq=1 ttl=63 time=82.1 ms
64 bytes from 172.66.147.243: icmp_seq=2 ttl=63 time=80.5 ms

--- example.com ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1006ms
rtt min/avg/max/mdev = 80.496/81.277/82.059/0.781 ms
```

`ping` sends small ICMP "echo" messages and waits for the replies. `-c 2` means two packets (without it, `ping` runs until `Ctrl + C`). `time` is the round trip in milliseconds. `ttl` is the "time to live" counter of the packet: each router lowers it by one. `ping localhost` tests only your own machine.

> ⚠️ **Common mistake:** Reading a failed `ping` as "the server is down". Many servers and firewalls ignore ICMP on purpose. A working `curl` or `nc` to the real port is the better proof; a failed `ping` alone proves nothing.

`traceroute -n example.com` lists the routers on the way (`-n` skips name lookups). Lines with `*` are routers that do not answer. It is useful to see *where* packets stop, but today many networks hide the path, so treat the result as a hint.

## 5. Ports and sockets: `ss` and `nc`

A **port** lets one machine run many network programs: the address finds the machine, the port finds the program. A program that wants to accept connections *listens* on a port. `ss` (socket statistics) lists the sockets:

```bash
ss -ltn
```

```text
State  Recv-Q Send-Q Local Address:Port  Peer Address:Port
LISTEN 0      128          0.0.0.0:8080       0.0.0.0:*
```

| Option | Meaning |
|---|---|
| `-l` | only **l**istening sockets |
| `-t` / `-u` | **t**CP / **u**DP |
| `-n` | **n**umbers, no name lookups (faster, clearer) |
| `-p` | show the **p**rocess that owns the socket (you see other users' processes only with `sudo`) |

Read the line: `0.0.0.0:8080` means "listening on port 8080 on **all** IPv4 addresses". Add `-p` to see the program:

```text
LISTEN 0      128          0.0.0.0:8080       0.0.0.0:*    users:(("p11-bind",pid=129,fd=3))
```

`ss -tn state established` lists open connections. For a connection from one program to another on the same machine you see two lines, one for each end:

```text
Recv-Q Send-Q Local Address:Port  Peer Address:Port
0      0          127.0.0.1:60338    127.0.0.1:9000
0      0          127.0.0.1:9000     127.0.0.1:60338
```

The client end uses a random high port (`60338`, an *ephemeral* port chosen by the system); the server end is the port it listens on (`9000`). `ss -lun` lists UDP sockets.

**`nc`** (netcat) opens plain connections. With `-z` it only checks whether the port is open, `-v` prints the result, and `-w 2` waits at most two seconds:

```bash
nc -zv 127.0.0.1 8080
nc -zv -w 2 127.0.0.1 9
echo "status: $?"
```

```text
Connection to 127.0.0.1 8080 port [tcp/http-alt] succeeded!
nc: connect to 127.0.0.1 port 9 (tcp) failed: Connection refused
status: 1
```

You can also make `nc` the listener to experiment: `nc -l 9000` waits for a connection on port 9000 and prints whatever arrives. The exit status (0 open, 1 not) makes `nc -z` easy to use in scripts, as in Exercise 1.

## 6. HTTP with `curl`

`curl` sends a request and prints the answer. For a web server it is the most useful tool of this lesson. First, a server to talk to: the Rust program of Section 7 answers every request with one line. Here is what `curl` shows (the server listens on port 8080):

```bash
curl -s http://127.0.0.1:8080/
curl -i -s http://127.0.0.1:8080/
curl -s -o /dev/null -w "code=%{http_code} size=%{size_download}\n" http://127.0.0.1:8080/
```

```text
hello from p11-bind
HTTP/1.1 200 OK
Content-Length: 20
Connection: close

hello from p11-bind
code=200 size=20
```

| Option | Meaning |
|---|---|
| `-s` | **s**ilent: no progress bar (but also no error text; add `-S` to still show errors) |
| `-i` | include the response **h**eaders in the output (`-I` sends a `HEAD` request: headers only) |
| `-o file` | write the body to a file (`-o /dev/null` throws it away) |
| `-w 'format'` | print values after the transfer, such as `%{http_code}` and `%{time_total}` |
| `-m 2` | give up after 2 seconds (always use a limit in scripts) |
| `-f` | **f**ail: exit with an error for status 400 and above instead of printing the error page |
| `-v` | **v**erbose: show every step, including DNS, the TLS handshake and the request |

`-v` against a real `https` site shows the whole chain of this lesson in a few lines (a selection of the output of `curl -sv https://example.com -o /dev/null 2>&1`):

```text
* Host example.com:443 was resolved.
* IPv4: 172.66.147.243, 104.20.23.154
* TLSv1.3 (OUT), TLS handshake, Client hello (1):
* TLSv1.3 (IN), TLS handshake, Server hello (2):
* Server certificate:
*   subject: CN=example.com
*   subjectAltName: "example.com" matches cert's "example.com"
> GET / HTTP/2
> Host: example.com
```

Lines with `*` are `curl`'s own notes, `>` is what it sent, `<` is what came back. You can see the name resolved to addresses (step 3), the TCP connection (step 5), the TLS handshake with the certificate check, and then the request.

**Exit codes.** `curl` tells you *where* it failed, which maps to the ladder:

```bash
curl -sS http://127.0.0.1:9/;            echo "exit: $?"
curl -sS http://no-such-name.invalid/;   echo "exit: $?"
curl -sS -m 2 http://10.255.255.1/;      echo "exit: $?"
```

```text
curl: (7) Failed to connect to 127.0.0.1 port 9 after 0 ms: Could not connect to server
exit: 7
curl: (6) Could not resolve host: no-such-name.invalid
exit: 6
curl: (28) Connection timed out after 2003 milliseconds
exit: 28
```

| Exit code | Meaning | Ladder step |
|---|---|---|
| 6 | could not resolve the host name | 3 (DNS) |
| 7 | connection refused (nothing listens, or no way) | 5 (port) |
| 28 | timeout (silence: firewall, lost packets) | 2, 4 or 5 |
| 22 | with `-f`: the server answered with status 400 or higher | 6 |

(`10.255.255.1` is a private address that nobody answers, so the connection hangs until the limit. Harmless, but do not use it as a server.)

## 7. Your Rust server and the bind address

Every network program must say **which address** it listens on. This one setting causes a large share of "it works on my machine" problems, and Docker in the next lessons makes it more important. Create a small server on your Mac:

```bash
cd ~/projects/rust
cargo new code/p11-bind
```

Replace `src/main.rs` (the file uses only the standard library, so `Cargo.toml` stays as it is):

```rust
use std::io::{Read, Write};
use std::net::{TcpListener, TcpStream};

/// Reads the request (and ignores it), then answers with a fixed page.
fn handle(mut stream: TcpStream) -> std::io::Result<()> {
    let mut buffer = [0u8; 1024];
    let _ = stream.read(&mut buffer)?;
    let body = "hello from p11-bind\n";
    write!(
        stream,
        "HTTP/1.1 200 OK\r\nContent-Length: {}\r\nConnection: close\r\n\r\n{}",
        body.len(),
        body
    )
}

fn main() -> std::io::Result<()> {
    // The address to listen on, for example 127.0.0.1:8080 or 0.0.0.0:8080.
    let addr = std::env::args()
        .nth(1)
        .unwrap_or_else(|| "127.0.0.1:8080".to_string());
    let listener = TcpListener::bind(&addr)?;
    println!("listening on {}", listener.local_addr()?);

    for stream in listener.incoming() {
        // A broken connection must not stop the server.
        if let Err(error) = stream.and_then(handle) {
            eprintln!("connection error: {error}");
        }
    }
    Ok(())
}
```

**What happens:**

1. `TcpListener::bind(&addr)` creates a listening socket for the address text from the first command-line argument (default `127.0.0.1:8080`). `local_addr()` shows what the system really chose.
2. `listener.incoming()` yields each new connection. `handle` reads the request bytes, ignores them and writes an HTTP response: a status line, headers, an empty line (`\r\n\r\n`), then the body. `Content-Length` tells the client how many body bytes follow.
3. A client such as `nc -z` connects and closes at once, so writing the answer fails with `Broken pipe`. The `if let Err` prints the error and goes on; with a bare `?` in `main` one such client would stop the whole server.

Copy the project into the VM and build it, as you did in Lesson 11.5 (replace `student` by your Mac user name):

```bash
mkdir -p ~/code/p11-bind
cp -r /Users/student/projects/rust/code/p11-bind/Cargo.toml /Users/student/projects/rust/code/p11-bind/src ~/code/p11-bind/
cd ~/code/p11-bind && cargo build --release
```

Start it for **loopback only**, in the background, and look at the socket:

```bash
./target/release/p11-bind 127.0.0.1:8080 &
sleep 1
ss -ltn | grep 8080
curl -s http://127.0.0.1:8080/
```

```text
listening on 127.0.0.1:8080
LISTEN 0      128        127.0.0.1:8080       0.0.0.0:*
hello from p11-bind
```

Now the important experiment, with **two separate computers** on one network. The server machine has the address `172.30.0.2`, and `curl` runs on the other machine. Example output from such a pair (you can repeat it later with two containers). With the server bound to `127.0.0.1` the other computer gets:

```text
$ curl -sS -m 3 http://172.30.0.2:8080/
curl: (7) Failed to connect to 172.30.0.2 port 8080 after 0 ms: Could not connect to server
```

After stopping the server and starting it with `0.0.0.0:8080` (**all** addresses of the machine), `ss` shows `0.0.0.0:8080` and the same request from the other computer works:

```text
$ curl -sS -m 3 http://172.30.0.2:8080/
hello from p11-bind
```

| Bind address | Who can connect |
|---|---|
| `127.0.0.1:8080` | only programs on the same machine |
| `0.0.0.0:8080` | anyone who can reach any address of the machine (other computers, containers, the Mac through Lima's port forwarding) |
| `192.168.5.15:8080` | only through that one address |

> 🔒 **Security:** Bind to `127.0.0.1` by default and open `0.0.0.0` only for services that must be reachable, ideally behind a firewall or a reverse proxy. A database or admin page on `0.0.0.0` is one of the most common ways that data leaks. In Docker (Lesson 11.12) the reverse mistake appears: a server inside a container that listens on `127.0.0.1` cannot be reached from outside the container, even when the port is published.

Three more things you will meet:

- **Address already in use.** Two programs cannot listen on the same address and port. The second one fails at `bind` with `Error: Os { code: 98, kind: AddrInUse, message: "Address already in use" }`. Find the owner with `ss -ltnp | grep 8080`.
- **Permission denied on a low port.** On Linux only root may listen on ports below 1024 by default. A normal user gets `Error: Os { code: 13, kind: PermissionDenied, message: "Permission denied" }` for port 80. Use a high port (`8080`) and let a proxy or the container platform map it, instead of running the service as root.
- **`localhost` is not always `127.0.0.1`.** `p11-bind localhost:8081` prints `listening on [::1]:8081`: the system gave IPv6 first, and `curl http://127.0.0.1:8081/` would be refused. In services write numeric addresses.

Stop the server when you are done: `kill %1` (or `pkill p11-bind`).

🧪 **Testing:** To test a network program without fixed ports, bind to **port 0**. The system picks a free port, and `local_addr()` tells it. Exercise 3 writes such a test.

## Common mistakes

1. **Testing from the wrong side.** `curl http://127.0.0.1:8080` working on the server says nothing about other computers. *Fix:* check the bind address with `ss -ltn` and test from a second machine.
2. **Treating `ping` as the truth** (ICMP may be blocked), or a timeout as "refused". *Fix:* use `nc -zv` and `curl`, and read the difference between refused (answered "no") and timeout (silence).
3. **Forgetting `-m` in scripts.** A `curl` or `nc` without a time limit can hang a deploy script for minutes. *Fix:* `curl -m 5`, `nc -w 2`, `ssh -o ConnectTimeout=5`.
4. **Using `0.0.0.0` for everything.** *Fix:* loopback by default.
5. **Editing `/etc/resolv.conf` by hand.** On Ubuntu it is managed by `systemd-resolved` and your change is overwritten. *Fix:* use `resolvectl status` to see the real configuration, and `/etc/hosts` for local names.

## Try it yourself

1. Run `ip -br addr show up` and `ip route` in the VM. Which address does `eth0` have, and what is the gateway?
2. Run `getent hosts example.com` and `dig +short example.com`. Add a line to `/etc/hosts` for `myservice.test` and resolve it. Remove the line again.
3. Build `p11-bind` in the VM. Start it on `127.0.0.1:8080`, find it with `ss -ltnp`, and call it with `curl -i`. Then try to start a second copy and read the error.
4. Run `curl -sv https://example.com -o /dev/null 2>&1 | head -20` and find: the resolved addresses, the TLS version, the request line.

## Exercises

### Exercise 1 — A port checker and its test (medium)

Write `check-port.sh HOST PORT` in `code/p11-scripts`: it uses `nc -z -w 2`, prints `HOST:PORT is open` and exits 0, or prints `HOST:PORT is closed or filtered` to standard error and exits 1. Missing arguments must stop it with a usage message. Then write `test-check-port.sh`, which starts `nc -l 127.0.0.1 9099` in the background and checks status 0 for port 9099 and status 1 for port 9 (kill the listener with a `trap`).

<details>
<summary>Hint</summary>

`host=${1:?usage: check-port.sh HOST PORT}` stops with a message. In the test, `nc -l ... &` and `server=$!`. Wait half a second (`sleep 0.5`) so the listener is ready. Store a status with `rc=0; command || rc=$?`.

</details>

<details>
<summary>Answer</summary>

```bash
#!/usr/bin/env bash
# check-port.sh HOST PORT - exit 0 if a TCP connection to HOST:PORT works, 1 if not.
set -u

host=${1:?usage: check-port.sh HOST PORT}
port=${2:?usage: check-port.sh HOST PORT}

if nc -z -w 2 "$host" "$port" 2> /dev/null; then
    echo "$host:$port is open"
else
    echo "$host:$port is closed or filtered" >&2
    exit 1
fi
```

```bash
#!/usr/bin/env bash
# test-check-port.sh - needs nc (netcat-openbsd) and check-port.sh in the same folder
set -u
here=$(cd "$(dirname "$0")" && pwd)
failures=0

nc -l 127.0.0.1 9099 > /dev/null &     # a listener on port 9099
server=$!
trap 'kill "$server" 2> /dev/null' EXIT
sleep 0.5

rc=0; bash "$here/check-port.sh" 127.0.0.1 9099 > /dev/null 2>&1 || rc=$?
if (( rc == 0 )); then echo "ok   - open port"; else echo "FAIL - open port (status $rc)"; failures=$((failures + 1)); fi

rc=0; bash "$here/check-port.sh" 127.0.0.1 9 > /dev/null 2>&1 || rc=$?
if (( rc == 1 )); then echo "ok   - closed port"; else echo "FAIL - closed port (status $rc)"; failures=$((failures + 1)); fi

(( failures == 0 ))
```

Running `bash test-check-port.sh` prints:

```text
ok   - open port
ok   - closed port
```

Why it works: `nc -l` accepts exactly one connection, which the first check uses; the second check targets a port where nobody listens, so `nc -z` fails with "Connection refused". The test never needs the network outside the machine.

</details>

### Exercise 2 — Fix the unreachable server (easy)

A colleague says: "My Rust service in the container listens on `127.0.0.1:3000`, I published port 3000, and I still get an error from outside." Explain the cause and what the service must change. (You will reproduce it with Docker in Lesson 11.13.)

<details>
<summary>Hint</summary>

Look at Section 7: who can connect to a socket bound to `127.0.0.1`? Where is that loopback when the program runs in a container?

</details>

<details>
<summary>Answer</summary>

Inside a container the loopback `127.0.0.1` belongs to the container only. A published port arrives on the container's network interface (`eth0`), not on its loopback, so a service bound to `127.0.0.1` never sees the connection. The service must bind to `0.0.0.0:3000` (best: read the address from an environment variable such as `BIND`, default `127.0.0.1` for local work, set to `0.0.0.0` in the container).

Why it works: the bind address chooses which interfaces accept connections; `0.0.0.0` means every interface of the machine, including the one Docker connects to the outside.

</details>

### Exercise 3 — Test the server on port 0 (hard)

Add a test to `p11-bind`: bind a listener to `127.0.0.1:0`, accept one connection in a thread and run `handle` on it, connect as a client, send a request, read the whole reply, and check that it starts with `HTTP/1.1 200 OK` and ends with `hello from p11-bind\n`.

<details>
<summary>Hint</summary>

`TcpListener::bind("127.0.0.1:0")`, then `listener.local_addr()` for the port. The server side runs in `std::thread::spawn(move || ...)`. The client calls `write_all` and `read_to_string`, which works because the response says `Connection: close`. Call `join()` on the thread at the end.

</details>

<details>
<summary>Answer</summary>

Add to `src/main.rs`:

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use std::thread;

    #[test]
    fn answers_with_the_fixed_page() {
        // Port 0 asks the system for any free port, so tests never collide.
        let listener = TcpListener::bind("127.0.0.1:0").unwrap();
        let addr = listener.local_addr().unwrap();
        let server = thread::spawn(move || {
            let (stream, _) = listener.accept().unwrap();
            handle(stream).unwrap();
        });

        let mut client = TcpStream::connect(addr).unwrap();
        client.write_all(b"GET / HTTP/1.1\r\n\r\n").unwrap();
        let mut reply = String::new();
        client.read_to_string(&mut reply).unwrap();
        server.join().unwrap();

        assert!(reply.starts_with("HTTP/1.1 200 OK"));
        assert!(reply.ends_with("hello from p11-bind\n"));
    }
}
```

`cargo test` prints:

```text
running 1 test
test tests::answers_with_the_fixed_page ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

Why it works: the test is a complete client and server in one process. Because the system picks the port, it never clashes with a running service or a parallel test. `use super::*` brings `handle`, `TcpListener` and `TcpStream` into the test module, and the traits `Read` and `Write` come with it too.

</details>

## Quiz

1. What is the difference between "connection refused" and a timeout?

<details>
<summary>Answer</summary>

Refused comes back at once: the machine is reachable, but nothing listens on that port (or the machine rejected it). A timeout is silence: packets are lost or a firewall drops them.

</details>

2. What does the line `0.0.0.0:8080` in `ss -ltn` tell you?

<details>
<summary>Answer</summary>

A program listens on port 8080 on all IPv4 addresses of the machine, so other computers can connect too. `127.0.0.1:8080` would mean only the same machine.

</details>

3. Which file is read first when a program resolves a name, `/etc/hosts` or the name server?

<details>
<summary>Answer</summary>

`/etc/hosts`. Only if the name is not there, the system asks the name server from `/etc/resolv.conf`.

</details>

4. What do `curl` exit codes 6, 7 and 28 mean?

<details>
<summary>Answer</summary>

6: the host name could not be resolved. 7: connection refused (nothing listening or no route). 28: the operation timed out.

</details>

5. Why does the test in Exercise 3 bind to port 0?

<details>
<summary>Answer</summary>

Port 0 makes the system pick any free port. The test never conflicts with another program or another test that runs at the same time; `local_addr()` returns the port that was chosen.

</details>

6. Why can a failed `ping` be misleading?

<details>
<summary>Answer</summary>

Many servers and firewalls drop ICMP, so `ping` fails although the web service works. Test the real service port with `nc -zv` or `curl`.

</details>

## Summary

- Climb the ladder: address (`ip -br addr`), route (`ip route`), name (`getent hosts`, `dig`), reachability (`ping`), port (`nc -zv`, `ss -ltn`), application (`curl -v`).
- Refused means "answered no"; timeout means silence. `curl` exit codes 6, 7 and 28 point to DNS, a closed port and a timeout.
- `ss -ltnp` shows which program listens where; the local address in that line is the bind address.
- `127.0.0.1` is reachable only from the same machine, `0.0.0.0` from every network of the machine. Bind to loopback by default and open the address on purpose.
- Rust: `TcpListener::bind(addr)`, handle a broken connection without stopping the server, and test with port 0.
- Always put time limits (`curl -m`, `nc -w`) in scripts.

## Sources and further reading

- [ip(8) manual page](https://man7.org/linux/man-pages/man8/ip.8.html) — the `ip` command with addresses, routes and links.
- [ss(8) manual page](https://man7.org/linux/man-pages/man8/ss.8.html) — all options and filters of `ss`.
- [curl documentation](https://curl.se/docs/manpage.html) — the `curl` options and exit codes.
- [BIND 9 manual pages (dig)](https://bind9.readthedocs.io/en/latest/manpages.html) — the DNS lookup tool `dig` and its options.
- [std::net::TcpListener](https://doc.rust-lang.org/std/net/struct.TcpListener.html) — the Rust standard library API used in Section 7.
- [The Rust Programming Language — Chapter 21, Final Project: a Web Server](https://doc.rust-lang.org/book/ch21-00-final-project-a-web-server.html) — a longer example of a TCP server.

---

[🏠 Start](../START-HERE.md) · [← 11.10 SSH and remote servers](10-ssh-and-remote-servers.md) · [📋 Phase 11](README.md) · [11.12 Containers and Docker basics →](12-containers-and-docker-basics.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
