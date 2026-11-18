<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 15.1 How the internet works: layers and packets](01-how-the-internet-works.md) · [📋 Phase 15](README.md) · [15.3 TCP and UDP →](03-tcp-and-udp.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/15-networking/02-ip-addresses-and-routing.md) <!-- nav -->

# Lesson 15.2 — IP addresses, subnets and routing

> **Phase 15 — Networking** · Lesson 2 of 16 · ⏱ about 2.5 hours (2 sittings)

## What you will learn

- What an IPv4 address really is (a 32-bit number) and how IPv6 addresses are written.
- What a subnet is, how to read CIDR notation like `10.20.30.40/20`, and how to compute the network and broadcast addresses.
- Which addresses are special: loopback, private, link-local, and why NAT exists.
- How a computer decides where to send a packet: the routing table and the longest prefix match.
- How to do all of this with `std::net` in Rust, with tests.

## Before you start

- You finished [Lesson 15.1 — How the internet works: layers and packets](01-how-the-internet-works.md) and built the `p15-nettools` image there.
- You know binary numbers and the bitwise operators `&`, `|`, `!`, `<<` from [Lesson 12.4 — Bitwise operations and bit tricks](../12-computer-systems/04-bitwise-operations.md).
- You know the `FromStr` trait and `parse()` from Phase 04 and Phase 05.
- Examples go into `code/p15-net/src/bin/`, as in Lesson 15.1.

## New words

| Word | Meaning |
|---|---|
| **IPv4 / IPv6** | Versions 4 and 6 of the Internet Protocol. IPv4 addresses are 32 bits long, IPv6 addresses 128 bits. |
| **octet** | One byte (8 bits) of an IPv4 address. `192.168.1.37` has four octets. |
| **subnet** (subnetwork) | A group of addresses that share the same first bits and can reach each other directly, without a router. |
| **prefix** | The first bits of an address, which are the same for the whole subnet. `/24` means "the first 24 bits". |
| **CIDR** (Classless Inter-Domain Routing) | The notation `address/prefix-length`, for example `192.168.1.0/24`, and the rule that a subnet can have any prefix length. |
| **subnet mask** | The prefix written as an address: ones for the prefix bits, zeros for the rest. `/24` is `255.255.255.0`. |
| **network address** | The first address of a subnet: all host bits are zero. |
| **broadcast address** | The last address of an IPv4 subnet: all host bits are one. A packet to it goes to every computer in the subnet. |
| **host** | Any computer (or container) with an IP address. The *host bits* are the bits after the prefix. |
| **loopback address** | `127.0.0.1` (all of `127.0.0.0/8`) and `::1`: the computer itself. |
| **private address** | An address from ranges that are not used on the public internet: `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`. |
| **link-local address** | An address that works only on one local link, never through a router: `169.254.0.0/16` and `fe80::/10`. |
| **NAT** (Network Address Translation) | A router that rewrites private source addresses into its own public address on the way out, and back on the way in. |
| **routing table** | The list of rules a computer uses to decide where to send each packet: "this subnet goes there". |
| **route** | One rule in the routing table: a destination subnet and how to reach it. |
| **gateway** (next hop) | The router to which a packet is handed when the destination is not in a directly connected subnet. |
| **default route** | The route `0.0.0.0/0`, which matches every address. It is used when nothing more specific matches. |
| **longest prefix match** | The rule for choosing between matching routes: the one with the longest prefix (the most specific one) wins. |
| **IP, TTL, ICMP** (Internet Protocol, Time To Live, Internet Control Message Protocol) | From Lesson 15.1: the protocol that carries packets between computers; the counter in the IP header that every router lowers by one; the small control and error messages, such as "time exceeded", that routers send back. |
| **VPN** (Virtual Private Network) | An encrypted tunnel that makes a remote computer behave as if it were inside a private network. |

## 1. An IPv4 address is a 32-bit number

People write IPv4 addresses as four numbers separated by dots, like `192.168.1.37`. This is only a
friendly way to show **one 32-bit number**. Each part is one byte, called an **octet**, with a value
from 0 to 255. The computer sees the bits:

```text
   192      168       1        37
11000000 10101000 00000001 00100101
```

In Rust, `std::net::Ipv4Addr` stores an address. It can be parsed from text and converted to and
from a `u32`. Save as `src/bin/ip_bits.rs`:

```rust
use std::net::Ipv4Addr;

fn main() {
    let addr: Ipv4Addr = "192.168.1.37".parse().expect("valid IPv4 address");
    let bits: u32 = addr.to_bits();

    println!("address: {addr}");
    println!("octets:  {:?}", addr.octets());
    println!("as u32:  {bits}");
    println!("as hex:  {bits:#010x}");
    println!("as bits: {bits:032b}");
    println!("back:    {}", Ipv4Addr::from_bits(bits + 1));
}
```

**What happens:**

1. `"192.168.1.37".parse()` uses the `FromStr` implementation of `Ipv4Addr`. It fails for text like
   `"192.168.1.300"`, so we handle the `Result` with `expect`.
2. `to_bits()` returns the address as a `u32`; `octets()` returns the four bytes as `[u8; 4]`.
3. `{bits:#010x}` prints hexadecimal with the `0x` prefix, padded to 10 characters; `{bits:032b}`
   prints all 32 bits.
4. `Ipv4Addr::from_bits(bits + 1)` turns a number back into an address. Adding 1 gives the next
   address. Because an address is just a number, all subnet arithmetic is integer arithmetic.

`cargo run --bin ip_bits` prints:

```text
address: 192.168.1.37
octets:  [192, 168, 1, 37]
as u32:  3232235813
as hex:  0xc0a80125
as bits: 11000000101010000000000100100101
back:    192.168.1.38
```

32 bits give about 4.3 billion addresses. That sounded like a lot in 1981; today there are more
devices than that. Two solutions exist: NAT (section 3) and IPv6 (section 4).

## 2. Subnets and CIDR notation

Computers on the same local network share the first part of their addresses. That shared part is the
**prefix**; the rest are the **host bits**, different for each computer. A **subnet** is the set of all
addresses with the same prefix.

**CIDR** notation writes the prefix length after a slash. `192.168.1.37/24` says: "the address is
192.168.1.37, and the first 24 bits are the network part".

```text
192.168.1.37/24
11000000 10101000 00000001 | 00100101
└──── prefix: 24 bits ────┘ └ host ┘

mask      11111111 11111111 11111111 00000000   = 255.255.255.0
network   address & mask                         = 192.168.1.0     (host bits all 0)
broadcast network | !mask                        = 192.168.1.255   (host bits all 1)
usable    192.168.1.1 … 192.168.1.254            = 2^8 - 2 = 254 hosts
```

The **mask** has ones for the prefix and zeros for the host bits. With the bitwise AND (`&`) you keep
only the prefix of any address: that is the **network address**. With OR (`|`) and the inverted mask
you set all host bits to one: that is the **broadcast address**. Those two are reserved, so a subnet
with `h` host bits has 2^h − 2 usable addresses.

The prefix does not have to end at a dot. `/20` cuts the third octet in the middle. Save as
`src/bin/subnet.rs`:

```rust
use std::net::Ipv4Addr;

fn describe(addr: Ipv4Addr, prefix: u32) {
    // A mask has `prefix` ones on the left and zeros on the right.
    let mask: u32 = u32::MAX.checked_shl(32 - prefix).unwrap_or(0);
    let network = addr.to_bits() & mask;
    let broadcast = network | !mask;
    let hosts = (broadcast - network).saturating_sub(1);

    println!("{addr}/{prefix}");
    println!("  mask:      {}", Ipv4Addr::from_bits(mask));
    println!("  network:   {}", Ipv4Addr::from_bits(network));
    println!("  broadcast: {}", Ipv4Addr::from_bits(broadcast));
    println!("  hosts:     {hosts}");
}

fn main() {
    describe(Ipv4Addr::new(192, 168, 1, 37), 24);
    describe(Ipv4Addr::new(10, 20, 30, 40), 20);
}
```

**What happens:**

1. `u32::MAX` is 32 ones. Shifting it left by `32 - prefix` pushes zeros in from the right, leaving
   `prefix` ones. For `/24` the shift is 8.
2. A shift by 32 or more is not allowed for a `u32` (it would panic in a debug build).
   `checked_shl` returns `None` in that case, and `unwrap_or(0)` gives the right mask for `/0`: no
   prefix bits at all.
3. `network | !mask` sets all host bits. `!` on an integer inverts every bit.
4. `saturating_sub(1)` subtracts 1 but stops at 0, so tiny subnets (`/31`, `/32`) do not underflow.

```text
192.168.1.37/24
  mask:      255.255.255.0
  network:   192.168.1.0
  broadcast: 192.168.1.255
  hosts:     254
10.20.30.40/20
  mask:      255.255.240.0
  network:   10.20.16.0
  broadcast: 10.20.31.255
  hosts:     4094
```

Look at the second result. The third octet 30 is `00011110` in binary. A `/20` keeps only its upper
four bits, `0001`, which is 16. So the subnet runs from `10.20.16.0` to `10.20.31.255`.

> 💡 **Tip:** you will meet CIDR everywhere in infrastructure work: Docker networks
> (`172.17.0.0/16`), Kubernetes pod and service ranges, firewall rules, cloud network settings.
> A shorter prefix means a *bigger* subnet: `/8` has 16 million addresses, `/32` is one single address.

## 3. Special addresses and NAT

Some ranges have fixed meanings. `Ipv4Addr` and `Ipv6Addr` have methods to check them. Save as
`src/bin/classify.rs`:

```rust
use std::net::{IpAddr, Ipv6Addr};

fn kind(ip: IpAddr) -> &'static str {
    match ip {
        IpAddr::V4(v4) if v4.is_loopback() => "loopback",
        IpAddr::V4(v4) if v4.is_private() => "private",
        IpAddr::V4(v4) if v4.is_link_local() => "link-local",
        IpAddr::V4(v4) if v4.is_unspecified() => "unspecified (any)",
        IpAddr::V4(v4) if v4.is_broadcast() => "broadcast",
        IpAddr::V6(v6) if v6.is_loopback() => "loopback",
        IpAddr::V6(v6) if v6.is_unicast_link_local() => "link-local",
        IpAddr::V6(v6) if v6.is_unique_local() => "private (unique local)",
        _ => "public (probably)",
    }
}

fn main() {
    let inputs = [
        "127.0.0.1",
        "172.17.0.4",
        "169.254.10.20",
        "0.0.0.0",
        "255.255.255.255",
        "1.1.1.1",
        "::1",
        "fe80::1",
        "2606:4700:4700:0000:0000:0000:0000:1111",
    ];
    for text in inputs {
        let ip: IpAddr = text.parse().expect("valid address");
        println!("{:<20} {}", ip.to_string(), kind(ip));
    }
    let v6: Ipv6Addr = "2001:db8::1".parse().unwrap();
    println!("segments of {v6}: {:x?}", v6.segments());
}
```

**What happens:**

1. `IpAddr` is an enum with two variants, `V4(Ipv4Addr)` and `V6(Ipv6Addr)`. Parsing into `IpAddr`
   accepts both forms.
2. Each match arm has a *guard* (`if ...`): the arm is taken only when the method returns `true`.
   The order matters; the first matching arm wins.
3. `"public (probably)"` is honest: some other special ranges exist (documentation, multicast), and
   the methods for all of them are not in stable Rust yet.
4. `{:<20}` pads the text to 20 characters, so the second column lines up. `{:x?}` prints the
   debug form with numbers in hexadecimal.

```text
127.0.0.1            loopback
172.17.0.4           private
169.254.10.20        link-local
0.0.0.0              unspecified (any)
255.255.255.255      broadcast
1.1.1.1              public (probably)
::1                  loopback
fe80::1              link-local
2606:4700:4700::1111 public (probably)
segments of 2001:db8::1: [2001, db8, 0, 0, 0, 0, 0, 1]
```

What the ranges mean:

- **Loopback** `127.0.0.0/8`: the computer itself. Servers that bind to `127.0.0.1` can be reached
  only from the same machine.
- **Unspecified** `0.0.0.0`: "no address yet" or, when a server binds to it, "every interface of this
  computer". You will use it in Lesson 15.4.
- **Private** `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`: free for anybody to use inside their own
  network. Your home network and the Docker container (`172.17.0.4`) use them. Routers on the public
  internet drop packets to these addresses.
- **Link-local** `169.254.0.0/16`: addresses a computer gives itself when nobody hands out addresses.
  Cloud providers also use `169.254.169.254` for their metadata service.

How can a computer with a private address load `example.com`? Through **NAT**. Your home router has
one public address. When a packet from `192.168.1.37:51000` goes out, the router rewrites the source
to `<its public address>:62001` and remembers that pair in a table. When the answer comes back to
port 62001, it rewrites the destination back to `192.168.1.37:51000`. Docker does the same for
containers. NAT saved IPv4 from running out, but it has a price: computers behind NAT cannot easily
receive new connections from outside.

> 🔒 **Security:** never decide that a request is "internal and safe" only because its address looks
> private. Addresses can be spoofed in some networks, and proxies hide the real client. Lesson 10.7
> (authorization) applies to internal services too.

## 4. IPv6 in five minutes

IPv6 addresses have 128 bits, written as eight groups of four hexadecimal digits, separated by
colons: `2606:4700:4700:0000:0000:0000:0000:1111`. Two rules make them shorter:

1. Leading zeros inside a group can be dropped: `0db8` → `db8`, `0000` → `0`.
2. **One** run of zero groups can be replaced with `::`. So the address above becomes
   `2606:4700:4700::1111`, and `0000:…:0001` becomes `::1`.

Rust's `Ipv6Addr` applies these rules when it prints an address, as the output above shows, and
`segments()` gives back the eight 16-bit groups. Important IPv6 addresses: `::1` (loopback),
`fe80::/10` (link-local, every interface has one), `fd00::/8` (private "unique local" addresses),
and `2001:db8::/32` (reserved for documentation and examples). The CIDR idea is the same as in IPv4,
just with 128 bits: a home network typically gets a `/64` and the computers pick their own host bits.

## 5. Routing: where does this packet go?

Every computer, not only routers, has a **routing table**. When your program sends a packet, the
kernel looks up the destination address in it and decides which interface to use and whether to hand
the packet to a **gateway**. Look at the table inside the container:

```bash
docker run --rm -it --cap-add NET_RAW --cap-add NET_ADMIN p15-nettools bash
ip -4 addr show eth0
ip route
```

```text
11: eth0@if2939: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 65535 qdisc noqueue state UP group default  link-netnsid 0
    inet 172.17.0.4/16 brd 172.17.255.255 scope global eth0
       valid_lft forever preferred_lft forever
default via 172.17.0.1 dev eth0 
172.17.0.0/16 dev eth0 proto kernel scope link src 172.17.0.4 
```

1. The container's address is `172.17.0.4/16`, and `brd` shows the broadcast address
   `172.17.255.255` that you can now compute yourself.
2. The route `172.17.0.0/16 dev eth0` says: "addresses in this subnet are directly reachable on
   `eth0`". The kernel added it automatically (`proto kernel`) when the address was set.
3. `default via 172.17.0.1` is the **default route**: everything else goes to the gateway
   `172.17.0.1`, Docker's virtual router.

`ip route get` asks the kernel which route it would use for one address:

```bash
ip route get 1.1.1.1
ip route get 172.17.0.9
```

```text
1.1.1.1 via 172.17.0.1 dev eth0 src 172.17.0.4 uid 0 
    cache 
172.17.0.9 dev eth0 src 172.17.0.4 uid 0 
    cache 
```

Both addresses match the default route (`0.0.0.0/0` matches everything), and `172.17.0.9` also
matches `172.17.0.0/16`. The kernel picks the **longest prefix**: `/16` is more specific than `/0`,
so `172.17.0.9` is reached directly. This rule is the heart of routing. Here it is in Rust. Save as
`src/bin/route.rs`:

```rust
use std::net::Ipv4Addr;

fn mask(prefix: u32) -> u32 {
    u32::MAX.checked_shl(32 - prefix).unwrap_or(0)
}

/// Longest prefix match: of all routes that match `dst`, pick the most specific one.
fn lookup<'a>(table: &'a [(Ipv4Addr, u32, &'a str)], dst: Ipv4Addr) -> Option<&'a str> {
    table
        .iter()
        .filter(|(network, prefix, _)| dst.to_bits() & mask(*prefix) == network.to_bits())
        .max_by_key(|(_, prefix, _)| *prefix)
        .map(|(_, _, via)| *via)
}

fn main() {
    let table = [
        (Ipv4Addr::new(0, 0, 0, 0), 0, "0.0.0.0/0 via 172.17.0.1"),
        (Ipv4Addr::new(172, 17, 0, 0), 16, "172.17.0.0/16 on eth0"),
        (Ipv4Addr::new(10, 8, 0, 0), 16, "10.8.0.0/16 via VPN tunnel"),
        (Ipv4Addr::new(10, 8, 5, 0), 24, "10.8.5.0/24 via lab router"),
    ];
    for text in ["1.1.1.1", "172.17.0.9", "10.8.1.1", "10.8.5.7"] {
        let dst: Ipv4Addr = text.parse().unwrap();
        let via = lookup(&table, dst).expect("the default route matches everything");
        println!("{dst:<11} -> {via}");
    }
}
```

**What happens:**

1. The table is an array of tuples: network address, prefix length, and a description. The first two
   rows copy the container's table; the last two are invented to show nested subnets.
2. `filter` keeps the routes whose network equals the destination with its host bits cleared.
3. `max_by_key` picks the route with the largest prefix: the longest prefix match.
4. The lifetime `'a` says that the returned `&str` lives as long as the table (Lesson 06.5).

```text
1.1.1.1     -> 0.0.0.0/0 via 172.17.0.1
172.17.0.9  -> 172.17.0.0/16 on eth0
10.8.1.1    -> 10.8.0.0/16 via VPN tunnel
10.8.5.7    -> 10.8.5.0/24 via lab router
```

`10.8.5.7` matches three routes (`/0`, `/16` and `/24`); the `/24` wins. Real routers do the same
lookup millions of times per second with special tree structures instead of a loop.

Every router on the way repeats this decision with its own table and lowers the TTL by one. You can
watch that happen. `ping -t 1` sends a packet with TTL 1, so the very first router must drop it and
answer with an ICMP "time exceeded" message:

```bash
ping -c 1 -t 1 1.1.1.1
```

```text
PING 1.1.1.1 (1.1.1.1) 56(84) bytes of data.
From 172.17.0.1 icmp_seq=1 Time to live exceeded

--- 1.1.1.1 ping statistics ---
1 packets transmitted, 0 received, +1 errors, 100% packet loss, time 0ms
```

The answer comes from `172.17.0.1`, the gateway from the routing table. Tools like `traceroute` and
`tracepath` send packets with TTL 1, 2, 3, … and list who answers, which shows every router on the
path. Leave the container with `exit`.

> 🧪 **Testing:** address arithmetic is full of off-by-one traps (`/0`, `/32`, the last address of a
> range). Test exactly these borders, as the exercises below do.

## Common mistakes

> ⚠️ **Common mistake: shifting by 32.** Writing the mask as `u32::MAX << (32 - prefix)` works for
> every prefix except `/0`. Then the shift amount is 32, and a debug build panics:
>
> ```text
> thread 'main' (5532244) panicked at src/bin/subnet.rs:5:21:
> attempt to shift left with overflow
> ```
>
> (The number after `'main'` is the thread ID and differs on every run.) Use `checked_shl(...).unwrap_or(0)`, or handle `/0` separately.

> ⚠️ **Common mistake: forgetting to clear the host bits.** `10.1.2.3/8` and `10.0.0.0/8` describe the
> same subnet. If you store the address unchanged and later compare it with a masked address, the
> comparison fails. Normalise the network address (`addr & mask`) once, when you parse it.

## Try it yourself

1. In the container, run `ip route get 127.0.0.1`. The answer starts with `local`: such addresses are
   handled by a separate table that you can see with `ip route show table local`.
2. Change `main` in `subnet.rs` to describe your own home network (on macOS, `ipconfig getifaddr en0`
   prints your Wi-Fi address; the prefix is usually `/24`).
3. Add `/31` and `/32` cases to `subnet.rs` and check that `hosts` becomes 0 instead of panicking.

## Exercises

### Exercise 1 — Compute a subnet by hand (easy)

Without running code, find the mask, the network address, the broadcast address and the number of
usable host addresses for `172.16.45.200/22`. Then check your answer with `describe` from
section 2.

<details>
<summary>Hint</summary>

`/22` leaves 10 host bits: the last two bits of the third octet and the whole fourth octet. Write
45 in binary (`00101101`) and clear its last two bits.

</details>

<details>
<summary>Answer</summary>

Mask `255.255.252.0`, network `172.16.44.0`, broadcast `172.16.47.255`, and 2^10 − 2 = 1022 usable
hosts. Clearing the last two bits of `00101101` gives `00101100` = 44; setting them gives
`00101111` = 47. `describe(Ipv4Addr::new(172, 16, 45, 200), 22)` prints the same values.

</details>

### Exercise 2 — A `Cidr` type with tests (medium)

Write `src/bin/cidr.rs` with `struct Cidr { network: Ipv4Addr, prefix: u32 }`, a method
`contains(&self, ip: Ipv4Addr) -> bool`, and `impl FromStr for Cidr` that parses text like
`"192.168.1.0/24"`. Parsing must reject a missing `/`, a bad address, a prefix that is not a number,
and a prefix larger than 32, and it must clear the host bits (`"10.1.2.3/8"` becomes `10.0.0.0/8`).
Write tests for normal cases, for `/0`, and for every error. Run `cargo test --bin cidr`.

<details>
<summary>Hint</summary>

`s.split_once('/')` returns `Option<(&str, &str)>`. `.ok_or("missing '/'")?` turns `None` into an
error; with `type Err = String`, the `?` converts the `&str` into a `String` for you. Check
`prefix > 32` **before** you compute the mask.

</details>

<details>
<summary>Answer</summary>

```rust
use std::net::Ipv4Addr;
use std::str::FromStr;

#[derive(Debug, PartialEq, Clone, Copy)]
struct Cidr {
    network: Ipv4Addr,
    prefix: u32,
}

impl Cidr {
    fn mask(&self) -> u32 {
        u32::MAX.checked_shl(32 - self.prefix).unwrap_or(0)
    }

    fn contains(&self, ip: Ipv4Addr) -> bool {
        ip.to_bits() & self.mask() == self.network.to_bits()
    }
}

impl FromStr for Cidr {
    type Err = String;

    fn from_str(s: &str) -> Result<Self, Self::Err> {
        let (addr, prefix) = s.split_once('/').ok_or("missing '/'")?;
        let addr: Ipv4Addr = addr.parse().map_err(|_| format!("bad address {addr:?}"))?;
        let prefix: u32 = prefix
            .parse()
            .map_err(|_| format!("bad prefix {prefix:?}"))?;
        if prefix > 32 {
            return Err(format!("prefix {prefix} is larger than 32"));
        }
        let mut cidr = Cidr {
            network: addr,
            prefix,
        };
        // Keep only the network part: "10.1.2.3/8" becomes 10.0.0.0/8.
        cidr.network = Ipv4Addr::from_bits(addr.to_bits() & cidr.mask());
        Ok(cidr)
    }
}

fn main() {
    let lan: Cidr = "192.168.1.0/24".parse().unwrap();
    println!("{lan:?}");
    println!(
        "contains 192.168.1.37? {}",
        lan.contains("192.168.1.37".parse().unwrap())
    );
    println!(
        "contains 192.168.2.1?  {}",
        lan.contains("192.168.2.1".parse().unwrap())
    );
    println!("{:?}", "10.0.0.0".parse::<Cidr>());
    println!("{:?}", "10.0.0.0/33".parse::<Cidr>());
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn parses_and_normalises() {
        let c: Cidr = "10.1.2.3/8".parse().unwrap();
        assert_eq!(c.network, Ipv4Addr::new(10, 0, 0, 0));
        assert_eq!(c.prefix, 8);
    }

    #[test]
    fn contains_checks_the_network_part() {
        let c: Cidr = "172.16.0.0/12".parse().unwrap();
        assert!(c.contains(Ipv4Addr::new(172, 31, 255, 1)));
        assert!(!c.contains(Ipv4Addr::new(172, 32, 0, 1)));
    }

    #[test]
    fn slash_zero_contains_everything() {
        let all: Cidr = "0.0.0.0/0".parse().unwrap();
        assert!(all.contains(Ipv4Addr::new(8, 8, 8, 8)));
    }

    #[test]
    fn rejects_bad_input() {
        assert!("10.0.0.0".parse::<Cidr>().is_err());
        assert!("10.0.0/8".parse::<Cidr>().is_err());
        assert!("10.0.0.0/x".parse::<Cidr>().is_err());
        assert!("10.0.0.0/33".parse::<Cidr>().is_err());
    }
}
```

`cargo run --bin cidr` prints:

```text
Cidr { network: 192.168.1.0, prefix: 24 }
contains 192.168.1.37? true
contains 192.168.2.1?  false
Err("missing '/'")
Err("prefix 33 is larger than 32")
```

and `cargo test --bin cidr` reports `test result: ok. 4 passed`.

Why it works: the parser checks every piece before it uses it and normalises the network once, so
`contains` can be a single AND and compare.

</details>

### Exercise 3 — Do two subnets overlap? (hard)

When you plan networks (a VPN, Docker networks, Kubernetes pod and service ranges), two ranges must
not overlap, or routing becomes ambiguous. Add a method `overlaps(&self, other: &Cidr) -> bool` to
your `Cidr` and test it: `10.0.0.0/8` and `10.1.0.0/16` overlap (in both orders), `10.0.0.0/16` and
`10.1.0.0/16` do not, and `0.0.0.0/0` overlaps everything. Print the results for `10.244.0.0/16`
against `10.244.3.0/24`, `10.96.0.0/12` and `10.0.0.0/8`.

<details>
<summary>Hint</summary>

Two CIDR blocks either do not overlap at all, or one lies completely inside the other. So it is
enough to ask the bigger block (the one with the *shorter* prefix) whether it contains the network
address of the smaller one.

</details>

<details>
<summary>Answer</summary>

Add this method to the `impl Cidr` block of Exercise 2:

```rust
    /// Two blocks overlap when the bigger one (shorter prefix) contains the other's network.
    fn overlaps(&self, other: &Cidr) -> bool {
        if self.prefix <= other.prefix {
            self.contains(other.network)
        } else {
            other.contains(self.network)
        }
    }
```

Inside the existing `mod tests`, add a small helper `c` (it keeps the tests short) and three tests:

```rust
    fn c(s: &str) -> Cidr {
        s.parse().unwrap()
    }

    #[test]
    fn a_block_inside_another_overlaps() {
        assert!(c("10.0.0.0/8").overlaps(&c("10.1.0.0/16")));
        assert!(c("10.1.0.0/16").overlaps(&c("10.0.0.0/8")));
    }

    #[test]
    fn neighbours_do_not_overlap() {
        assert!(!c("10.0.0.0/16").overlaps(&c("10.1.0.0/16")));
    }

    #[test]
    fn the_whole_internet_overlaps_everything() {
        assert!(c("0.0.0.0/0").overlaps(&c("192.168.1.0/24")));
    }
```

With a `main` that checks `10.244.0.0/16` against the three other blocks, the program prints:

```text
10.244.0.0/16 overlaps 10.244.3.0/24: true
10.244.0.0/16 overlaps 10.96.0.0/12: false
10.244.0.0/16 overlaps 10.0.0.0/8: true
```

Why it works: CIDR blocks are aligned. A block with prefix `p` starts at a multiple of 2^(32−p), so
a smaller block can never stick out of a bigger one halfway; checking one address is enough. (These
are the default pod and service ranges of many Kubernetes setups, which is why they must not
overlap.)

</details>

## Quiz

1. What is `10.0.0.0/8` in plain words, and how many addresses does it contain?

<details>
<summary>Answer</summary>

All addresses whose first 8 bits are `00001010` (the first octet is 10): `10.0.0.0` to
`10.255.255.255`, which is 2^24 = 16,777,216 addresses.

</details>

2. Which of these can be reached from the public internet: `192.168.0.10`, `172.20.1.1`, `8.8.8.8`,
   `172.32.0.1`?

<details>
<summary>Answer</summary>

`8.8.8.8` and `172.32.0.1`. `192.168.0.10` is in `192.168.0.0/16` and `172.20.1.1` is in
`172.16.0.0/12` (which ends at `172.31.255.255`); both are private.

</details>

3. A table has routes `0.0.0.0/0`, `10.0.0.0/8` and `10.1.0.0/16`. Which one is used for
   `10.1.2.3`, and why?

<details>
<summary>Answer</summary>

`10.1.0.0/16`. All three match, and the longest prefix match picks the most specific route.

</details>

4. What does NAT do, and why did it become common?

<details>
<summary>Answer</summary>

A NAT router rewrites the private source address (and port) of outgoing packets into its own public
address and reverses the change for the answers. It lets many devices share one public IPv4
address, because there are not enough IPv4 addresses for every device.

</details>

5. Write `2001:0db8:0000:0000:0000:ff00:0042:8329` in the short form.

<details>
<summary>Answer</summary>

`2001:db8::ff00:42:8329`. Leading zeros are dropped in each group, and the run of three zero groups
becomes `::`.

</details>

## Summary

- An IPv4 address is a 32-bit number shown as four octets; `Ipv4Addr` converts between text, bytes and
  `u32` (`parse`, `octets`, `to_bits`, `from_bits`).
- CIDR `address/prefix` describes a subnet. The mask has `prefix` ones; `address & mask` is the network
  address, `network | !mask` the broadcast address; `2^h − 2` addresses are usable.
- Special ranges: loopback `127.0.0.0/8` and `::1`, private `10/8`, `172.16/12`, `192.168/16`,
  link-local `169.254/16` and `fe80::/10`, unspecified `0.0.0.0`.
- NAT lets many private addresses share one public address; IPv6 (128 bits) removes the shortage.
- The routing table maps subnets to interfaces and gateways; the **longest prefix match** chooses the
  route; `0.0.0.0/0` is the default route.
- `ip addr`, `ip route` and `ip route get` show what the kernel does; test address code at the
  borders (`/0`, `/32`).

## Sources and further reading

- [RFC 4632 — Classless Inter-domain Routing (CIDR)](https://datatracker.ietf.org/doc/html/rfc4632) — the CIDR notation and the idea of prefixes.
- [RFC 1918 — Address Allocation for Private Internets](https://datatracker.ietf.org/doc/html/rfc1918) — the three private IPv4 ranges.
- [RFC 4291 — IP Version 6 Addressing Architecture](https://datatracker.ietf.org/doc/html/rfc4291) — how IPv6 addresses are written and which ranges are special.
- [RFC 5952 — A Recommendation for IPv6 Address Text Representation](https://datatracker.ietf.org/doc/html/rfc5952) — the rules for the short IPv6 form that Rust follows.
- [`std::net` module](https://doc.rust-lang.org/std/net/index.html) — `Ipv4Addr`, `Ipv6Addr`, `IpAddr` and their methods.
- [ip-route(8) manual page](https://man7.org/linux/man-pages/man8/ip-route.8.html) — everything about Linux routing tables.

---

[🏠 Start](../START-HERE.md) · [← 15.1 How the internet works: layers and packets](01-how-the-internet-works.md) · [📋 Phase 15](README.md) · [15.3 TCP and UDP →](03-tcp-and-udp.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
