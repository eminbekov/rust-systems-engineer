<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 22.8 eBPF track: maps and tracing](08-ebpf-track-maps-and-tracing.md) · [📋 Phase 22](README.md) · [22.10 Embedded track: microcontrollers and no_std →](10-embedded-track-microcontrollers-and-no-std.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/22-specializations/09-ebpf-track-project-and-next-steps.md) <!-- nav -->

# Lesson 22.9 — eBPF track: project and next steps

> **Phase 22 — Specializations** · Lesson 9 of 30 · ⏱ about 20 hours (10 sittings)

> **This is the last lesson of the eBPF track.** When you finish the project, continue with
> [Lesson 22.30 — Checkpoint: your specialization](30-checkpoint-your-specialization.md) and answer the eBPF question
> set. The lessons between here and 22.30 belong to other tracks; the next one starts at
> [Lesson 22.10 — Embedded track: microcontrollers and no_std](10-embedded-track-microcontrollers-and-no-std.md).

## What you will learn

- What XDP is: where it runs, what a program can decide for each packet, and how the verifier forces bounds checks.
- How to build a small test network inside the VM with a network namespace and a veth pair.
- How to write, load and attach a first XDP program with Aya.
- How to build a packet counter and IP blocklist (the project), with tests and an end-to-end check.
- Where to go next in eBPF: books, documentation, open-source projects and communities.

## Before you start

- You finished [Lesson 22.8 — eBPF track: maps and tracing](08-ebpf-track-maps-and-tracing.md).
- You know layers, packets and headers ([Lesson 15.1](../15-networking/01-how-the-internet-works.md)), IPv4 addresses
  ([Lesson 15.2](../15-networking/02-ip-addresses-and-routing.md)) and network namespaces
  ([Lesson 13.15](../13-operating-systems/15-namespaces.md)).
- You know byte order ([Lesson 12.1](../12-computer-systems/01-bits-bytes-and-number-systems.md)): network headers are
  big-endian.

## New words

| Word | Meaning |
|---|---|
| **XDP action** | The answer an XDP program returns for a packet: pass it on, drop it, send it back, or redirect it. |
| **`XDP_PASS` / `XDP_DROP`** | "Give the packet to the normal network stack" / "throw it away now". |
| **`XDP_TX` / `XDP_REDIRECT`** | "Send it back out of the same interface" / "send it to another interface or CPU". |
| **`XDP_ABORTED`** | "Something went wrong": the packet is dropped and an error event is counted. |
| **packet data / `data_end`** | The start and the end address of the bytes of a packet; every read must stay before `data_end`. |
| **bounds check** | A comparison that proves to the verifier that a read stays inside the packet. |
| **network namespace** | A separate copy of the network stack: its own interfaces, addresses and routes (Lesson 13.15). |
| **veth pair** (virtual Ethernet) | Two virtual network interfaces connected like the two ends of a cable. |
| **native / generic XDP** | XDP run by the network driver (fast) / XDP run a bit later by the kernel for any interface (works everywhere). |
| **Ethernet header** | The first 14 bytes of a frame: destination and source MAC (Media Access Control) address and the EtherType. |
| **EtherType** | A 2-byte number that says what follows the Ethernet header, for example `0x0800` for IPv4. |
| **ICMP** (Internet Control Message Protocol) | The protocol of `ping` (IP protocol number 1). |
| **blocklist** | A list of addresses whose packets must be dropped. |

**Abbreviations from earlier lessons:** BPF (Berkeley Packet Filter, the name eBPF grew from), VM (virtual machine;
the Linux VM `dev`), CPU (Central Processing Unit), IP (Internet Protocol; IPv4 and IPv6 are its versions 4 and 6),
TCP (Transmission Control Protocol) and UDP (User Datagram Protocol) from Lesson 15.3, ARP (Address Resolution
Protocol, which finds the MAC address for an IP address), LSM (Linux Security Modules, Lesson 22.6) and SKB (from
`sk_buff`, "socket buffer", the kernel's packet structure).

## 1. XDP: one decision per packet

XDP runs your program in the network driver, for every received packet, **before** the kernel builds its usual packet
structure (`sk_buff`) and before any firewall rule, socket or routing decision:

```text
  network card ──> driver ──> [ XDP program ] ──> kernel network stack ──> sockets ──> your server
                                   │
                                   ├── XDP_DROP      (gone, almost no cost)
                                   ├── XDP_TX        (bounced back out)
                                   └── XDP_REDIRECT  (to another interface or CPU)
```

Because it runs so early, dropping a packet in XDP is very cheap: this is how large services survive floods of
unwanted traffic. The price is that you see only **raw bytes**: the Ethernet header, then (maybe) an IPv4 header, then
(maybe) TCP, UDP or ICMP — you parse them yourself.

The context `XdpContext` gives two addresses: `ctx.data()`, the first byte of the packet, and `ctx.data_end()`, one
byte after the last. The verifier refuses any read that it cannot prove to be before `data_end`. So every header
access starts with a check like this helper:

```rust
#[inline(always)]
fn ptr_at<T>(ctx: &XdpContext, offset: usize) -> Option<*const T> {
    let start = ctx.data();
    let end = ctx.data_end();
    if start + offset + core::mem::size_of::<T>() > end {
        return None;
    }
    Some((start + offset) as *const T)
}
```

After the `if`, the verifier knows that `size_of::<T>()` bytes at `start + offset` are inside the packet, and it
accepts reads through the returned pointer. `#[inline(always)]` keeps the check in the same function as the read,
where the verifier can see both. A packet that is too short simply returns `None`.

Two modes matter in practice. **Native** XDP is run by the driver itself; it is the fast path, and the virtual `veth`
interfaces of this lesson support it. **Generic** XDP (also called SKB mode) runs a little later in the kernel and works
with any interface; it is slower but useful for testing. Aya's `XdpMode::Default` lets the kernel choose.

## 2. A test network in the VM

Never attach a dropping program to the VM's real network interface: you could cut your own connection. Build a private
test network instead: a second network namespace, `p22-peer`, connected to the main one by a veth pair. Save this as
`netns.sh` in the project folder:

```sh
#!/bin/sh
# Creates a second network namespace connected to this one with a veth pair.
set -eu
ip netns add p22-peer
ip link add p22-host type veth peer name p22-peer netns p22-peer
ip addr add 10.22.0.1/24 dev p22-host
ip link set p22-host up
ip netns exec p22-peer ip addr add 10.22.0.2/24 dev p22-peer
ip netns exec p22-peer ip link set p22-peer up
```

```text
 main namespace (the VM)                 namespace p22-peer
 ┌──────────────────────┐   veth pair   ┌──────────────────────┐
 │ p22-host 10.22.0.1   ├───────────────┤ p22-peer 10.22.0.2   │
 │   ↑ XDP runs here    │               │   ping 10.22.0.1     │
 └──────────────────────┘               └──────────────────────┘
```

**What happens:**

1. `ip netns add p22-peer` creates an empty network namespace.
2. `ip link add ... type veth peer name p22-peer netns p22-peer` creates both ends of the virtual cable and moves one
   end into the new namespace at once.
3. Each end gets an address in `10.22.0.0/24` and is switched on.
4. `ip netns exec p22-peer <command>` runs a command inside the other namespace. A `ping` from there to `10.22.0.1`
   arrives on `p22-host`, where your XDP program will sit.

Run it once in the VM with `sudo ./netns.sh` (it must be run again after a reboot of the VM). To remove the test
network later: `sudo ip link del p22-host; sudo ip netns del p22-peer`.

## 3. The smallest XDP program

Create a new workspace, `code/p22-netwatch`, by copying the structure of `p22-tracer` from Lesson 22.7: the same
`.cargo/config.toml`, the same two `build.rs` files, and `Cargo.toml` files where `tracer` is renamed to `netwatch`
everywhere. The workspace dependencies are those of Lesson 22.8 (no `aya-log`):

```toml
[workspace]
resolver = "3"
members = ["netwatch", "netwatch-ebpf"]
default-members = ["netwatch"]

[workspace.package]
edition = "2024"

[workspace.dependencies]
aya = "0.16"
aya-build = "0.2"
aya-ebpf = "0.3"
anyhow = "1"
cargo_metadata = "0.23"
tokio = { version = "1", features = ["macros", "rt-multi-thread", "signal", "time"] }
which = "8"

[profile.release.package.netwatch-ebpf]
debug = 2
codegen-units = 1
```

`netwatch-ebpf/src/main.rs` counts every packet and lets it pass:

```rust
#![no_std]
#![no_main]

use aya_ebpf::{
    bindings::xdp_action,
    macros::{map, xdp},
    maps::PerCpuArray,
    programs::XdpContext,
};

/// One packet counter per CPU.
#[map]
static PACKETS: PerCpuArray<u64> = PerCpuArray::with_max_entries(1, 0);

#[xdp]
pub fn netwatch(_ctx: XdpContext) -> u32 {
    if let Some(count) = PACKETS.get_ptr_mut(0) {
        // SAFETY: the pointer points to this CPU's counter in the map.
        unsafe { *count += 1 };
    }
    xdp_action::XDP_PASS
}

#[cfg(not(test))]
#[panic_handler]
fn panic(_info: &core::panic::PanicInfo) -> ! {
    loop {}
}

#[unsafe(link_section = "license")]
#[unsafe(no_mangle)]
static LICENSE: [u8; 13] = *b"Dual MIT/GPL\0";
```

`netwatch/src/main.rs` attaches it to an interface and prints the total every second:

```rust
use std::ffi::CString;
use std::time::Duration;

use aya::maps::PerCpuArray;
use aya::programs::{Xdp, XdpMode};

#[tokio::main]
async fn main() -> anyhow::Result<()> {
    let iface = std::env::args().nth(1).unwrap_or_else(|| "lo".to_string());

    let mut ebpf = aya::Ebpf::load(aya::include_bytes_aligned!(concat!(
        env!("OUT_DIR"),
        "/netwatch"
    )))?;
    let program: &mut Xdp = ebpf.program_mut("netwatch").unwrap().try_into()?;
    program.load()?;
    program.attach(&CString::new(iface.as_str())?, XdpMode::Default)?;

    let packets: PerCpuArray<_, u64> = PerCpuArray::try_from(ebpf.map("PACKETS").unwrap())?;
    println!("Counting packets on {iface}. Press Ctrl-C to stop.");
    loop {
        tokio::select! {
            _ = tokio::signal::ctrl_c() => break,
            _ = tokio::time::sleep(Duration::from_secs(1)) => {
                let total: u64 = packets.get(&0, 0)?.iter().sum();
                println!("packets: {total}");
            }
        }
    }
    Ok(())
}
```

**What happens:**

1. `#[xdp]` makes `netwatch` an XDP program. It returns an action; `XDP_PASS` hands every packet on unchanged.
2. A per-CPU array with one slot is the cheapest possible global counter: packets arrive on many CPUs at once, and
   each CPU increments its own copy.
3. `program.attach(name, XdpMode::Default)` attaches it to the interface. Aya takes the name as a C string
   (`CString`), because the kernel interface uses C strings.
4. `tokio::select!` (Lesson 14.14) waits for whichever comes first: Ctrl-C, or the next one-second tick, when the
   per-CPU values are added and printed.

Build and run in the VM (after `sudo ./netns.sh`), and ping from the peer namespace in a second terminal:

```bash
cargo run --release -- p22-host
```

```bash
sudo ip netns exec p22-peer ping -c 3 -i 0.2 10.22.0.1
```

```text
3 packets transmitted, 3 received, 0% packet loss, time 412ms
```

```text
Counting packets on p22-host. Press Ctrl-C to stop.
packets: 2
packets: 8
packets: 9
```

More than three packets arrive: besides the three ICMP echo requests, the peer sends ARP (Address Resolution Protocol)
and IPv6 neighbour messages. While the program runs, `ip link show p22-host` shows it attached:

```text
12: p22-host@if11: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 xdp qdisc noqueue state UP mode DEFAULT group default qlen 1000
    link/ether 06:ca:3f:b4:d4:b2 brd ff:ff:ff:ff:ff:ff link-netns p22-peer
    prog/xdp id 8285 name netwatch tag c1449139186ca030 jited
```

The word `xdp` and the line `prog/xdp` show that a native XDP program named `netwatch` is attached. When you stop
`netwatch`, the program is detached automatically.

## The project

### What you will build

`p22-netwatch` v1: an XDP tool that counts packets per protocol on one interface and drops all IPv4 packets from
blocked addresses. With `10.22.0.2` blocked and three pings from the peer, the reference solution prints:

```text
blocking 10.22.0.2
watching p22-host, Ctrl-C to stop
icmp 0 tcp 0 udp 0 other 1 dropped 0
icmp 3 tcp 0 udp 0 other 2 dropped 3
icmp 3 tcp 0 udp 0 other 2 dropped 3
```

and the ping in the peer namespace reports `3 packets transmitted, 0 received, 100% packet loss`.

### Contract

- **Usage:** `netwatch <interface> [--block <ipv4-address>]...`, run as root (through the runner).
- **Start-up:** one line `blocking <address>` per blocked address, then `watching <interface>, Ctrl-C to stop`.
- **Every second:** one line `icmp <n> tcp <n> udp <n> other <n> dropped <n>`, totals since start.
- **Counting:** IPv4 packets go to `icmp`, `tcp` or `udp` by their protocol number (1, 6, 17), everything else
  (other IPv4 protocols, IPv6, ARP, too-short packets) to `other`.
- **Blocking:** an IPv4 packet whose **source** address is in the blocklist is dropped (`XDP_DROP`) and also counted in
  `dropped`. All other packets pass.
- **Errors:** an invalid address such as `--block 10.22.0.300` ends the program with an error before anything is
  attached.
- **Quality:** the pure logic (classifying a protocol, parsing the arguments) lives in `netwatch-common` with unit
  tests; `cargo clippy` and `cargo fmt --check` are clean; there is a README; one commit per milestone.

### Step-by-step plan

#### M1 — Count by protocol

**Goal:** five counters instead of one, filled by parsing the Ethernet and IPv4 headers with bounds checks.

**Guidance:** add the crate `network-types = "0.2"` to the kernel side; it has `#[repr(C)]` structs `EthHdr` and
`Ipv4Hdr`. Use the `ptr_at` helper from Section 1 for each header. Make the per-CPU array five slots long and give
the slot numbers names (constants) in a new `netwatch-common` crate that both sides use.

<details>
<summary>Hint</summary>

`EthHdr::LEN` is the size of the Ethernet header, so the IPv4 header is at `ptr_at::<Ipv4Hdr>(ctx, EthHdr::LEN)`.
`EthHdr`'s field `ether_type` is a raw `u16` in network byte order; compare it with `EtherType::Ipv4 as u16`, which
the crate defines already converted. `Ipv4Hdr` has the fields `proto: u8` and `src_addr: [u8; 4]`.

</details>

**Done when:** three pings from the peer give a line with `icmp 3` (the `other` count varies):

```text
watching p22-host, Ctrl-C to stop
icmp 0 tcp 0 udp 0 other 2 dropped 0
icmp 3 tcp 0 udp 0 other 6 dropped 0
```

#### M2 — A blocklist

**Goal:** `--block <address>` (repeatable) fills a map, and the XDP program drops packets from those sources.

**Guidance:** a `HashMap<[u8; 4], u8>` on the kernel side (the value is unused; the key is the address bytes in
network order, exactly as they appear in the header). On the user side, parse each address with
`std::net::Ipv4Addr`, and insert `ip.octets()` **after** loading the program but before or after attaching — think
about which is better, and write the reason in the README.

<details>
<summary>Hint</summary>

User side: `let mut blocked: HashMap<_, [u8; 4], u8> = HashMap::try_from(ebpf.map_mut("BLOCKED").unwrap())?;` and
`blocked.insert(&ip.octets(), &1, 0)?;`. Kernel side: `if unsafe { BLOCKED.get(&src) }.is_some() { ... XDP_DROP }`.
Count the drop in slot `DROPPED` before you return.

</details>

**Done when:** with `cargo run --release -- p22-host --block 10.22.0.2`, a ping from the peer fails:

```bash
sudo ip netns exec p22-peer ping -c 3 -i 0.2 -W 1 10.22.0.1
```

```text
3 packets transmitted, 0 received, 100% packet loss, time 412ms
```

and `netwatch` prints `dropped 3`. Without `--block` the same ping works again.

#### M3 — Pure logic and its tests

**Goal:** move every decision that does not need the kernel into `netwatch-common` and test it on any machine.

**Guidance:** at least two pure functions: `bucket(is_ipv4: bool, proto: u8) -> u32` (which counter slot a packet
goes to) and an argument parser that returns the interface name and the list of blocked addresses, or an error.
The kernel side calls `bucket`; the user side calls the parser.

<details>
<summary>Hint</summary>

Keep `netwatch-common` `#![cfg_attr(not(test), no_std)]`. The argument parser needs `Vec` and `String`, which are
not in `core`: put it in the user-side crate's library part (`netwatch/src/lib.rs`) instead, or behind a Cargo feature
`user` of the common crate, as the official Aya template does. Test cases: ICMP/TCP/UDP/other; no arguments; `--block`
without an address; an invalid address.

</details>

**Done when:** `cargo test` (in the VM, from the workspace) runs at least six tests and they pass, and the program
refuses a bad address:

```text
Error: invalid IPv4 address syntax
```

#### M4 — An end-to-end test script

**Goal:** `test.sh` checks the whole behaviour: a ping passes without a block, ICMP is counted, a ping fails with a
block, and three packets are dropped.

**Guidance:** start `netwatch` in the background with `&`, remember its process ID in `$!`, run the ping, wait a
little so that one more line is printed, stop it with `kill -INT`, and check its saved output with `grep`. Run the
script with `sudo` after `./netns.sh`.

<details>
<summary>Hint</summary>

```sh
$bin p22-host --block 10.22.0.2 > /tmp/netwatch-block.txt &
sleep 1
if ip netns exec p22-peer ping -c 3 -i 0.2 -W 1 10.22.0.1 > /dev/null; then
  echo "FAIL: ping should be blocked"; kill -INT $!; exit 1
fi
```

</details>

**Done when:**

```bash
cargo build --release && sudo ./test.sh
```

```text
ok:   ping passes without a block
ok:   icmp packets counted
ok:   ping is blocked
ok:   3 packets dropped
all checks passed
```

#### M5 — Finish

**Goal:** a project you can show. `cargo clippy` and `cargo fmt --check` clean, a README with the commands of
Sections 2–3 and M4, a short "how it works" paragraph with the diagram of Section 1, and `git log --oneline` with one
commit per milestone.

**Done when:** a friend can clone the repository, follow the README in a fresh VM, and see `all checks passed`.

### Stretch goals

- **Block by destination port.** Parse the TCP and UDP headers too (`TcpHdr`, `UdpHdr` from `network-types`) and drop,
  for example, all UDP packets to port 9. Remember a new bounds check for every header.
- **Live blocklist changes.** Keep the program running and add or remove addresses while it runs (for example: read
  commands from standard input). The kernel side does not change at all — that is the power of maps.
- **Statistics per source.** A `PerCpuHashMap<[u8; 4], u64>` with packet counts per source address, and a "top 5"
  print every second.
- **`XDP_TX`.** Answer pings yourself: swap the MAC and IP addresses, change the ICMP type to "echo reply", fix the
  checksum and return `XDP_TX`.
- **A benchmark.** Measure how many packets per second your program handles with a packet generator, and compare with
  `XDP_PASS` only (Lesson 19.2 on load testing).

## 4. Next steps

You have written tracepoint, kprobe and XDP programs in Rust and shared maps and types between the kernel and user
space. To go further:

**Read and follow:**

- *The Aya book* — the official guide, with chapters on more program types (classifiers, LSM, uprobes, cgroup
  programs) and on BTF-based programs.
- *Learning eBPF* by Liz Rice (O'Reilly; the company Isovalent offers it as a free download) — a clear introduction
  to how eBPF works, with C and Go examples whose ideas carry over to Aya.
- *BPF Performance Tools* by Brendan Gregg — the reference book for tracing and performance analysis with eBPF.
- *The XDP tutorial* (xdp-project) — step-by-step lessons on packet processing with XDP (in C; translate them to Aya as
  exercises).
- *The Linux kernel BPF documentation* — the source of truth for program types, maps, helpers and the verifier.

**Open-source projects to contribute to:**

| Project | What it is | A good first step |
|---|---|---|
| Aya | The Rust eBPF library you used. | Read the issues labelled for newcomers, improve an example or the book, add a test. |
| bpf-linker | The linker that builds eBPF from Rust. | Build it from source, run its tests, look at open issues. |
| bpfman | A Rust program (built on Aya) that manages eBPF programs on a machine and in Kubernetes. | Run it in the VM, follow its getting-started guide, pick a documentation or small code issue. |
| bpftrace | The tracing language from Lesson 22.6 (C++). | Write and contribute a useful tool script, or improve the documentation. |
| Cilium | Kubernetes networking and security built on eBPF (Go and C). | Read the architecture documentation; good for the cloud-native track too. |

**Communities:** the Aya Discord server (linked from the Aya repository), the eBPF Slack (linked from ebpf.io), the
yearly eBPF Summit talks (free online), and the `bpf` mailing list of the Linux kernel for kernel-side questions.

**Phase 24** shows how to choose an issue, work with maintainers and send your first contribution.

> 💡 **Tip:** When you read eBPF code written in C, translate one small program into Aya. The concepts (maps, helpers,
> context, verifier rules) are the same; only the syntax changes. It is the fastest way to use the large amount of
> C-based eBPF material.

## Common mistakes

> ⚠️ **Common mistake:** Reading a header without a bounds check, for example
> `(ctx.data() + EthHdr::LEN) as *const Ipv4Hdr` without comparing with `data_end`. Loading fails:
>
> ```text
> invalid access to packet, off=27 size=1, R3(id=0,off=27,r=14)
> R3 offset is outside of the packet
> ```
>
> The verifier knows only that 14 bytes (`r=14`, the Ethernet header) were checked, and the program reads byte 27 (inside
> the IPv4 header). Use `ptr_at` for every header.

> ⚠️ **Common mistake:** Comparing the EtherType or addresses in the wrong byte order. Header fields are big-endian
> (network order) and your CPU is little-endian. `EtherType::Ipv4 as u16` from `network-types` is already converted;
> addresses are compared as `[u8; 4]` arrays, which have no byte-order problem.

> ⚠️ **Common mistake:** Attaching a dropping XDP program to the VM's main interface (`eth0`). You may lose the
> connection to the VM. Test on the veth pair; if it happens anyway, restart the VM with `limactl stop dev` and
> `limactl start dev`.

> ⚠️ **Common mistake:** Forgetting that the veth pair and the namespace disappear when the VM restarts. If
> `netwatch` says the interface does not exist, run `sudo ./netns.sh` again.

## Quiz

1. Why is dropping a packet in XDP cheaper than dropping it with a firewall rule later in the kernel?

<details>
<summary>Answer</summary>

XDP runs in the driver before the kernel allocates its packet structure and before routing, firewall and socket
lookups. A dropped packet costs almost nothing beyond reading its first bytes.

</details>

2. What does the verifier require before an XDP program reads an IPv4 header?

<details>
<summary>Answer</summary>

A check that the whole header lies before `data_end` (start + offset + size ≤ end). Without that comparison on the
path to the read, the program is rejected.

</details>

3. What is the difference between `XDP_PASS`, `XDP_DROP` and `XDP_TX`?

<details>
<summary>Answer</summary>

`XDP_PASS` gives the packet to the normal network stack, `XDP_DROP` discards it at once, and `XDP_TX` sends it back
out of the interface it came in on (usually after changing it).

</details>

4. Why does the project use a veth pair and a network namespace instead of the VM's real interface?

<details>
<summary>Answer</summary>

It is a private, controlled network: you know every packet (your pings), and a bug that drops packets cannot cut your
own connection to the VM.

</details>

5. Why can the blocklist change while the XDP program runs, without reloading it?

<details>
<summary>Answer</summary>

The program only looks the source address up in a map. User space can insert or remove map entries at any time; the
next packet sees the new contents.

</details>

## Summary

- XDP runs an eBPF program in the driver for each received packet and returns an action: pass, drop, transmit back or
  redirect.
- Packet data is raw bytes between `data()` and `data_end()`; every header read needs a bounds check (`ptr_at`) or the
  verifier refuses the program.
- A network namespace plus a veth pair is a safe test network inside the VM.
- The project: per-protocol counters in a per-CPU array, an IPv4 blocklist in a hash map, pure logic with tests, and
  an end-to-end test script.
- Next: the Aya book, *Learning eBPF*, *BPF Performance Tools*, the XDP tutorial, and contributions to Aya,
  bpf-linker, bpfman, bpftrace or Cilium.

## Sources and further reading

- [The Aya book](https://aya-rs.dev/book/) — XDP and the other program types in Rust.
- [`network-types` documentation](https://docs.rs/network-types/0.2.0/network_types/) — the header structs used in the
  project.
- [xdp-tutorial](https://github.com/xdp-project/xdp-tutorial) — hands-on XDP lessons from the XDP project.
- [Learning eBPF (Liz Rice)](https://isovalent.com/books/learning-ebpf/) — the book and its free download page.
- [BPF Performance Tools (Brendan Gregg)](https://www.brendangregg.com/bpf-performance-tools-book.html) — the book's
  page with its tools.
- [ip-netns(8)](https://man7.org/linux/man-pages/man8/ip-netns.8.html) and [veth(4)](https://man7.org/linux/man-pages/man4/veth.4.html) —
  the manual pages for the test network.
- [Aya](https://github.com/aya-rs/aya), [bpfman](https://github.com/bpfman/bpfman) and [Cilium](https://cilium.io/) —
  open-source projects to study and contribute to.

---

[🏠 Start](../START-HERE.md) · [← 22.8 eBPF track: maps and tracing](08-ebpf-track-maps-and-tracing.md) · [📋 Phase 22](README.md) · [22.10 Embedded track: microcontrollers and no_std →](10-embedded-track-microcontrollers-and-no-std.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
