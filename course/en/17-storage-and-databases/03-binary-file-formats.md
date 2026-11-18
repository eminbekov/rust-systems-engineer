<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 17.2 Durability: fsync and crash safety](02-durability-and-fsync.md) · [📋 Phase 17](README.md) · [17.4 The write-ahead log (WAL) →](04-write-ahead-log.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/17-storage-and-databases/03-binary-file-formats.md) <!-- nav -->

# Lesson 17.3 — Binary file formats and serialization

> **Phase 17 — Storage and database internals** · Lesson 3 of 15 · ⏱ about 2.5 hours (2 sittings)

## What you will learn

- Why storage engines keep data in binary form, not as text.
- How to turn integers into bytes and back, and why you must choose a byte order and write it down.
- How to design a record format with length prefixes, so that a reader knows where each record ends.
- How a CRC-32 checksum detects torn writes and damaged bytes.
- How to serialize whole structs with `serde` and `postcard`, and what a varint is.

## Before you start

- You finished [Lesson 17.2 — Durability: fsync and crash safety](02-durability-and-fsync.md): you know torn writes and
  have `src/crash.rs` in `code/p17-storage`.
- You know bits, bytes, hexadecimal and byte order (endianness) from
  [Lesson 12.1 — Bits, bytes and number systems](../12-computer-systems/01-bits-bytes-and-number-systems.md), and bit masks
  from [Lesson 12.4](../12-computer-systems/04-bitwise-operations.md).
- You used `serde` with JSON (JavaScript Object Notation, a text format for data) in [Lesson 05.11](../05-rust-intermediate/11-practical-crates-serde-clap-errors.md).
- Add the crates of this lesson:

```bash
cd ~/projects/rust/code/p17-storage
cargo add crc32fast@1
cargo add postcard@1 --features alloc
cargo add serde@1 --features derive
```

## New words

| Word | Meaning |
|---|---|
| **binary format** | A way to store data as raw bytes with a fixed meaning (for example "bytes 4–7 are the key length"), not as readable text. |
| **encode / decode** | Turn a value into bytes / turn bytes back into a value. |
| **serialization** | Encoding a whole structure (a struct, a list) into bytes so that it can be saved or sent, and decoding it later. |
| **little-endian / big-endian** | Byte orders: the lowest byte of a number comes first / the highest byte comes first. |
| **length prefix** | A number written *before* a piece of data that says how many bytes the data has. |
| **record** | One unit of data in a file, here one key and one value with their header. |
| **checksum** | A small number computed from a block of bytes. If the bytes change, the checksum (almost surely) changes too. |
| **CRC** (Cyclic Redundancy Check) | A family of checksums that is fast to compute and very good at catching damaged bytes. CRC-32 gives a 32-bit number. |
| **MAC** (Message Authentication Code) | A short tag computed from data and a secret key; only someone with the key can make a valid one (Lesson 10.9). |
| **magic number** | Fixed bytes at the start of a file that say "this is my format", so that a program does not read a wrong file. |
| **varint** (variable-length integer) | An integer encoding that uses fewer bytes for small numbers: 7 bits per byte. Also called LEB128 (Little-Endian Base 128). |

## 1. Text or binary?

You could store the number 1000 as the text `"1000"`, or as the four bytes of a `u32`. Put this into
`src/bin/int_bytes.rs`:

```rust
fn main() {
    let n: u32 = 1000;
    println!("as text:       {:?}", n.to_string().as_bytes());
    println!("little-endian: {:02x?}", n.to_le_bytes());
    println!("big-endian:    {:02x?}", n.to_be_bytes());

    let back = u32::from_le_bytes([0xe8, 0x03, 0x00, 0x00]);
    println!("decoded:       {back}");
}
```

**What happens:**

1. `n.to_string().as_bytes()` shows the text form: the ASCII (American Standard Code for Information Interchange) codes of
   `'1'`, `'0'`, `'0'`, `'0'`.
2. `to_le_bytes` returns `[u8; 4]` with the lowest byte first; `to_be_bytes` with the highest byte first.
   `{:02x?}` prints each byte as two hexadecimal digits.
3. `u32::from_le_bytes` does the opposite: four bytes in, one number out.

`cargo run --bin int_bytes`:

```text
as text:       [49, 48, 48, 48]
little-endian: [e8, 03, 00, 00]
big-endian:    [00, 00, 03, e8]
decoded:       1000
```

1000 is `0x3e8`. As text it needs 4 bytes here, but 10 bytes for 4,000,000,000, and a reader must scan for the end and
parse digits. As a `u32` it is **always 4 bytes**, and decoding is a single copy. Storage engines want exactly that:
fixed sizes, so that they can jump to "byte 8" without reading bytes 0–7, and no parsing.

The price is that the bytes mean nothing without the format. `[e8, 03, 00, 00]` is 1000 only if you know it is a
little-endian `u32`. So every binary format must **write down** its byte order. This course uses little-endian, which is
also what the processors of your Mac and of almost every server use natively.

> 📏 **Convention:** write the byte order into the name of every helper (`to_le_bytes`, `read_u32_le`). Never use
> `to_ne_bytes` ("native endian") in a file format: a file written on one machine must be readable on any other.

## 2. A record format

A key-value store needs to save pairs like `apple → red` in a file, one after another. The reader must know where one
record ends and the next begins, and it must notice a torn or damaged record (Lesson 17.2). This layout does both:

```text
 offset 0      4           8             12
 ┌────────────┬───────────┬─────────────┬─────────────┬───────────────┐
 │ crc32 (u32)│ key_len   │ value_len   │ key bytes   │ value bytes   │
 │            │ (u32, LE) │ (u32, LE)   │ (key_len)   │ (value_len)   │
 └────────────┴───────────┴─────────────┴─────────────┴───────────────┘
   checksum of everything after it ─────────────────────────────────────►
```

The two **length prefixes** tell the reader how many bytes follow. The **checksum** at the front covers the rest of the
record, including the lengths. Create `src/record.rs`, starting with the types:

```rust
use std::fmt;

/// Bytes before the key: checksum, key length and value length, each a little-endian u32.
pub const HEADER_LEN: usize = 12;
/// The largest key or value we accept (1 MiB). Anything bigger on disk is damage.
pub const MAX_LEN: usize = 1 << 20;

#[derive(Debug, Clone, PartialEq, Eq)]
pub struct Record {
    pub key: Vec<u8>,
    pub value: Vec<u8>,
}

#[derive(Debug, PartialEq, Eq)]
pub enum DecodeError {
    /// Fewer bytes than the record needs: the end of a torn write.
    Truncated,
    /// A length field larger than MAX_LEN.
    TooLarge(usize),
    /// The checksum does not match the bytes.
    BadChecksum,
}

impl fmt::Display for DecodeError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            DecodeError::Truncated => write!(f, "record is truncated"),
            DecodeError::TooLarge(n) => write!(f, "length {n} is larger than {MAX_LEN}"),
            DecodeError::BadChecksum => write!(f, "checksum mismatch"),
        }
    }
}

impl std::error::Error for DecodeError {}
```

Keys and values are `Vec<u8>`, plain bytes: the storage engine does not care whether they hold text, numbers or images.
Then the encoder, in the same file:

```rust
/// Appends one encoded record to `out`.
pub fn encode(key: &[u8], value: &[u8], out: &mut Vec<u8>) {
    assert!(
        key.len() <= MAX_LEN && value.len() <= MAX_LEN,
        "key or value too large"
    );
    let start = out.len();
    out.extend_from_slice(&[0; 4]); // room for the checksum
    out.extend_from_slice(&(key.len() as u32).to_le_bytes());
    out.extend_from_slice(&(value.len() as u32).to_le_bytes());
    out.extend_from_slice(key);
    out.extend_from_slice(value);
    let crc = crc32fast::hash(&out[start + 4..]);
    out[start..start + 4].copy_from_slice(&crc.to_le_bytes());
}
```

**What happens:**

1. The `assert!` protects the format: a length must fit into a `u32`, and we also refuse anything above `MAX_LEN`. Without
   it, `as u32` would silently cut a too-large length.
2. `encode` appends to `out` instead of returning a new `Vec`, so many records can be built in one buffer and written with
   one system call (Lesson 17.1).
3. We first push four zero bytes as a placeholder, then the lengths and the data.
4. `crc32fast::hash` computes the CRC-32 of everything after the placeholder; `copy_from_slice` puts it into the
   placeholder.

## 3. Reading it back

Decoding must be careful: the bytes come from a disk after a possible crash. Add to `src/record.rs`:

```rust
fn read_u32(bytes: &[u8], at: usize) -> u32 {
    let four: [u8; 4] = bytes[at..at + 4].try_into().unwrap();
    u32::from_le_bytes(four)
}

/// Decodes the record at the start of `bytes`.
/// Returns the record and the number of bytes it used.
pub fn decode(bytes: &[u8]) -> Result<(Record, usize), DecodeError> {
    if bytes.len() < HEADER_LEN {
        return Err(DecodeError::Truncated);
    }
    let crc = read_u32(bytes, 0);
    let key_len = read_u32(bytes, 4) as usize;
    let value_len = read_u32(bytes, 8) as usize;
    for len in [key_len, value_len] {
        if len > MAX_LEN {
            return Err(DecodeError::TooLarge(len));
        }
    }
    let total = HEADER_LEN + key_len + value_len;
    if bytes.len() < total {
        return Err(DecodeError::Truncated);
    }
    if crc32fast::hash(&bytes[4..total]) != crc {
        return Err(DecodeError::BadChecksum);
    }
    let key = bytes[HEADER_LEN..HEADER_LEN + key_len].to_vec();
    let value = bytes[HEADER_LEN + key_len..total].to_vec();
    Ok((Record { key, value }, total))
}
```

**What happens:**

1. `try_into()` turns the 4-byte slice into an array `[u8; 4]`. It returns a `Result` because a slice can have any
   length; ours has exactly 4 bytes, so `unwrap` cannot fail.
2. The checks come in a safe order: first "is there a whole header?", then "are the lengths sane?", then "are all the
   bytes there?", and only then "is the checksum right?". No step reads past the end of `bytes`.
3. `decode` returns how many bytes the record used, so the caller knows where the next record starts.

Register the module in `src/lib.rs` (keep the lines from Lesson 17.2):

```rust
pub mod crash;
pub mod record;
pub mod synclog;
```

Now a program that writes two records into a file and reads them back, `src/bin/records.rs`:

```rust
use p17_storage::record::{self, DecodeError};
use std::fs;

fn main() -> Result<(), Box<dyn std::error::Error>> {
    // 1. Encode two records into one buffer and save it.
    let mut buf = Vec::new();
    record::encode(b"apple", b"red", &mut buf);
    record::encode(b"kiwi", b"green", &mut buf);
    fs::write("records.bin", &buf)?;
    println!("wrote {} bytes", buf.len());

    // 2. Read the file back and decode records one after another.
    let bytes = fs::read("records.bin")?;
    let mut pos = 0;
    while pos < bytes.len() {
        match record::decode(&bytes[pos..]) {
            Ok((rec, used)) => {
                println!(
                    "offset {pos:>2}: {} = {}",
                    String::from_utf8_lossy(&rec.key),
                    String::from_utf8_lossy(&rec.value)
                );
                pos += used;
            }
            Err(DecodeError::Truncated) => {
                println!("offset {pos:>2}: torn record, stop");
                break;
            }
            Err(e) => return Err(e.into()),
        }
    }
    Ok(())
}
```

**What happens:**

1. Both records go into one buffer and one `fs::write`.
2. The reader walks through the bytes with `pos`. After each record it jumps forward by `used` bytes.
3. `String::from_utf8_lossy` shows bytes as text, replacing invalid UTF-8 (the text encoding of Lesson 12.5) with `�`; good enough for printing.
4. A truncated record ends the loop (the torn tail of Lesson 17.2); any other error is returned. `e.into()` turns our
   `DecodeError` into a `Box<dyn Error>`, which works because we implemented `std::error::Error`.

`cargo run --bin records`, then look at the raw bytes with `xxd` (it prints offsets, bytes in hexadecimal, and the bytes
as text on the right):

```bash
cargo run --bin records
xxd records.bin
```

```text
wrote 41 bytes
offset  0: apple = red
offset 20: kiwi = green
00000000: 9c22 6d4f 0500 0000 0300 0000 6170 706c  ."mO........appl
00000010: 6572 6564 8368 d6a6 0400 0000 0500 0000  ered.h..........
00000020: 6b69 7769 6772 6565 6e                   kiwigreen
```

Read the first record in the dump: `9c22 6d4f` is the checksum, `0500 0000` is the key length 5 (little-endian),
`0300 0000` is the value length 3, then `apple` and `red`. The second record starts at offset 20 (`0x14`) with its own
checksum `8368 d6a6`.

## 4. Checksums: catching damage

A **checksum** is a short fingerprint of a block of bytes. CRC-32 is computed with a fixed formula that is very fast
(`crc32fast` uses special processor instructions). Two facts make it useful for storage:

- If random damage changes the bytes, the chance that the checksum still matches is about 1 in 4 billion (2³²).
- Some damage is caught **always**: any single changed bit, and any burst of changed bits no longer than 32.

So after a crash, a reader can tell "a complete, correct record" from "a torn record" or "garbage that happens to be in
the file". A torn write usually shows up as `Truncated` (the file ends too early). But a file system may also leave old
garbage or zero bytes at the end of a file after a crash; then the lengths look valid and only the checksum shows the
problem.

> 🔒 **Security:** a CRC protects against *accidents*, not against an attacker. Anybody can change the bytes and compute
> a new CRC. To detect deliberate changes you need a MAC (Message Authentication Code) with a secret key, from
> [Lesson 10.9](../10-security-fundamentals/09-cryptography-hashes-macs-and-randomness.md). Also note `MAX_LEN`: a
> length read from disk is **input**, and without the limit a damaged length of four billion would make the reader try to
> allocate 4 GiB (gibibytes, 4 × 1,073,741,824 bytes).

The tests for the format go at the end of `src/record.rs`:

```rust
#[cfg(test)]
mod tests {
    use super::*;

    fn encoded(key: &[u8], value: &[u8]) -> Vec<u8> {
        let mut buf = Vec::new();
        encode(key, value, &mut buf);
        buf
    }

    #[test]
    fn round_trip() {
        let buf = encoded(b"apple", b"red");
        let (record, used) = decode(&buf).unwrap();
        assert_eq!(record.key, b"apple");
        assert_eq!(record.value, b"red");
        assert_eq!(used, HEADER_LEN + 5 + 3);
    }

    #[test]
    fn a_flipped_bit_is_detected() {
        let mut buf = encoded(b"apple", b"red");
        let last = buf.len() - 1;
        buf[last] ^= 0b0000_0001; // "red" becomes "ree"
        assert_eq!(decode(&buf), Err(DecodeError::BadChecksum));
    }

    #[test]
    fn a_cut_record_is_truncated() {
        let buf = encoded(b"apple", b"red");
        assert_eq!(decode(&buf[..15]), Err(DecodeError::Truncated));
    }

    #[test]
    fn a_huge_length_is_rejected() {
        let mut buf = encoded(b"apple", b"red");
        buf[4..8].copy_from_slice(&u32::MAX.to_le_bytes());
        assert_eq!(decode(&buf), Err(DecodeError::TooLarge(u32::MAX as usize)));
    }
}
```

**What happens:**

1. `round_trip` is the most important test of any format: encode, decode, and get the same thing back.
2. `^= 0b0000_0001` flips the lowest bit of the last byte (`d` = `0x64` becomes `e` = `0x65`), like a bit flip on a bad
   device.
3. `&buf[..15]` keeps 15 of 20 bytes, like a torn write.
4. The last test writes `u32::MAX` into the key length, like garbage after a crash.

`cargo test --lib record::`:

```text
running 4 tests
test record::tests::round_trip ... ok
test record::tests::a_flipped_bit_is_detected ... ok
test record::tests::a_cut_record_is_truncated ... ok
test record::tests::a_huge_length_is_rejected ... ok

test result: ok. 4 passed; 0 failed; 0 ignored; 0 measured; 3 filtered out; finished in 0.00s
```

> 🧪 **Testing:** for every binary format, test the round trip, and then test damage of every kind you can think of:
> cut, flipped, too long, all zeros. The decoder must return an error for each, and never panic.

## 5. Serializing structs with serde and postcard

Writing every field by hand is fine for a small record header. For bigger structures, Rust has **serde**: you derive
`Serialize` and `Deserialize`, and a format crate turns the value into bytes. You used it with JSON in Lesson 05.11.
For compact binary data, this course uses **postcard**. (Older tutorials use the `bincode` crate; it is no longer
maintained, so do not add it to new projects.) Put this into `src/bin/postcard_demo.rs`:

```rust
use serde::{Deserialize, Serialize};

#[derive(Debug, PartialEq, Serialize, Deserialize)]
struct User {
    id: u32,
    name: String,
    admin: bool,
}

fn main() -> Result<(), postcard::Error> {
    let user = User {
        id: 1000,
        name: "ada".to_string(),
        admin: true,
    };
    let bytes: Vec<u8> = postcard::to_allocvec(&user)?;
    println!("{} bytes: {:02x?}", bytes.len(), bytes);

    let back: User = postcard::from_bytes(&bytes)?;
    println!("{back:?}");
    assert_eq!(back, user);
    Ok(())
}
```

**What happens:**

1. `#[derive(Serialize, Deserialize)]` generates the code that walks through the fields in order.
2. `postcard::to_allocvec` serializes into a new `Vec<u8>` (the `alloc` feature you enabled provides it).
3. `postcard::from_bytes` decodes. It returns an error, not a panic, if the bytes do not fit the type.

`cargo run --bin postcard_demo`:

```text
7 bytes: [e8, 07, 03, 61, 64, 61, 01]
User { id: 1000, name: "ada", admin: true }
```

Only 7 bytes. There are no field names in the output (unlike JSON): the reader must know the struct. `61 64 61` is
`"ada"`, preceded by its length `03`, and `01` is `true`. But why is 1000 `e8 07` and not `e8 03 00 00`?

postcard writes integers as **varints**. Each byte carries 7 bits of the number; the highest bit (`0x80`) says "another
byte follows". 1000 is `0b111_1101000` in binary:

```text
lowest 7 bits:  1101000 = 0x68, more follows → 0x68 | 0x80 = 0xe8
next 7 bits:    0000111 = 0x07, last byte    → 0x07
```

Small numbers, which are the most common, take one byte; a `u32` takes at most 5. Exercise 3 asks you to write this
encoding yourself.

When to use which? Write the **hot, fixed parts** of a storage format by hand (headers, lengths, checksums), because their
exact bytes must never change and they are read millions of times. Use **serde + postcard** for the rest: values that
are structs, metadata files, messages.

## 6. Formats live for years: magic numbers and versions

A database file written today may be opened by the program of next year. Real formats therefore start with a small
**file header**:

```text
 ┌──────────────────┬──────────────┬─────────────── records … ───
 │ magic: "P17LOG\0"│ version: u16 │
 │ (7 bytes)        │ (2 bytes LE) │
 └──────────────────┴──────────────┴──────────────────────────────
```

The **magic number** lets a program refuse a wrong file at once ("this is not a log file") instead of decoding garbage.
The **version** lets new code read old files: if the version is 1, use the old decoder. SQLite files start with the text
`SQLite format 3` followed by a zero byte, for example, and PNG (Portable Network Graphics) images start with the bytes `89 50 4e 47`.

> ⚠️ **Common mistake:** changing a field in a struct that is serialized with postcard, and then reading old files. postcard
> has no field names, so the old bytes are decoded with the new layout. Change formats only together with a new version
> number.

## Common mistakes

1. **Using native endian (`to_ne_bytes`) in a file.** The file is then readable only on machines with the same byte order.
2. **Trusting a length from disk.** Without a maximum, a damaged length makes the reader allocate gigabytes or read past
   the end. Check lengths before you use them.
3. **Indexing slices without checking the length first.** `bytes[4..8]` panics if `bytes` is shorter. A decoder should
   return `Truncated`, never panic.
4. **Checksumming only the data, not the lengths.** If a length is damaged, the reader looks for the data in the wrong
   place. Our checksum covers everything after it.

## Try it yourself

1. Change `"kiwi"` in `records.rs` to `"banana"` and predict where the second record starts before you run it.
2. Run `xxd records.bin` and find the two length fields of the second record.
3. In `postcard_demo.rs`, change `id` to 5 and to 300. Predict the bytes, then check.

## Exercises

### Exercise 1 — Byte order in a test (easy)

Add a test `integers_are_little_endian` to the tests in `src/record.rs`: for `n: u32 = 0x0102_0304`, check the results of
`to_le_bytes` and `to_be_bytes`, and that `from_le_bytes` gives `n` back.

<details>
<summary>Hint</summary>

The lowest byte of `0x0102_0304` is `0x04`. Little-endian puts it first.

</details>

<details>
<summary>Answer</summary>

```rust
    #[test]
    fn integers_are_little_endian() {
        let n: u32 = 0x0102_0304;
        assert_eq!(n.to_le_bytes(), [0x04, 0x03, 0x02, 0x01]);
        assert_eq!(n.to_be_bytes(), [0x01, 0x02, 0x03, 0x04]);
        assert_eq!(u32::from_le_bytes([0x04, 0x03, 0x02, 0x01]), n);
    }
```

Why it works: hexadecimal digits come in pairs, one pair per byte, so `0x0102_0304` is the bytes `01 02 03 04` from the
highest to the lowest. Little-endian reverses that order.

</details>

### Exercise 2 — Every cut is a torn record (medium)

In Lesson 17.2 you learned that a crash leaves a prefix of what was written. Write a test `every_cut_is_truncated`: encode
one record, then for every length `cut` from 0 to the record's length minus 1, check that decoding `&buf[..cut]` returns
`Err(DecodeError::Truncated)`. At the end, check that the whole buffer decodes.

<details>
<summary>Hint</summary>

A `for cut in 0..buf.len()` loop. Add `"cut at {cut}"` as the message of `assert_eq!`, so a failure tells you where.

</details>

<details>
<summary>Answer</summary>

```rust
    #[test]
    fn every_cut_is_truncated() {
        let buf = encoded(b"apple", b"red");
        for cut in 0..buf.len() {
            assert_eq!(
                decode(&buf[..cut]),
                Err(DecodeError::Truncated),
                "cut at {cut}"
            );
        }
        assert!(decode(&buf).is_ok());
    }
```

Why it works: for cuts shorter than 12 bytes the header is incomplete; for longer cuts the lengths are valid but the data
is missing. In both cases `decode` notices it **before** it reads the missing bytes. This is the property the WAL (write-ahead log) in
Lesson 17.4 relies on: a torn last record is reported as `Truncated`, not as corruption or a panic.

</details>

### Exercise 3 — Your own varint (hard)

Create `src/varint.rs` (and add `pub mod varint;` to `src/lib.rs`) with
`pub fn encode_varint(n: u64, out: &mut Vec<u8>)` and `pub fn decode_varint(bytes: &[u8]) -> Option<(u64, usize)>`
(the number and the bytes used, or `None` if the input ends too early). Test: 5 encodes as `[0x05]`; 1000 encodes as
`[0xe8, 0x07]`, exactly like `postcard::to_allocvec(&1000u32)`; a round trip works for 0, 1, 127, 128, 300, `1 << 35`
and `u64::MAX`; and `[0xe8]` alone decodes as `None`.

<details>
<summary>Hint</summary>

Encoding: while `n >= 0x80`, push `(n as u8 & 0x7f) | 0x80` and shift `n` right by 7; then push the last byte. Decoding:
for byte number `i`, add `(byte & 0x7f) << (7 * i)`; stop when the top bit is 0. A `u64` needs at most 10 bytes.

</details>

<details>
<summary>Answer</summary>

```rust
/// Appends `n` as a varint (LEB128): 7 bits per byte, lowest bits first,
/// the top bit of each byte says "more bytes follow".
pub fn encode_varint(mut n: u64, out: &mut Vec<u8>) {
    while n >= 0x80 {
        out.push((n as u8 & 0x7f) | 0x80);
        n >>= 7;
    }
    out.push(n as u8);
}

/// Decodes a varint from the start of `bytes`.
/// Returns the number and how many bytes it used, or `None` if the input ends too early or is too long.
pub fn decode_varint(bytes: &[u8]) -> Option<(u64, usize)> {
    let mut n: u64 = 0;
    for (i, &byte) in bytes.iter().enumerate().take(10) {
        n |= u64::from(byte & 0x7f) << (7 * i);
        if byte & 0x80 == 0 {
            return Some((n, i + 1));
        }
    }
    None
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn small_numbers_take_one_byte() {
        let mut out = Vec::new();
        encode_varint(5, &mut out);
        assert_eq!(out, [0x05]);
    }

    #[test]
    fn matches_postcard() {
        let mut out = Vec::new();
        encode_varint(1000, &mut out);
        assert_eq!(out, [0xe8, 0x07]);
        assert_eq!(out, postcard::to_allocvec(&1000u32).unwrap());
    }

    #[test]
    fn round_trip() {
        for n in [0, 1, 127, 128, 300, 1 << 35, u64::MAX] {
            let mut out = Vec::new();
            encode_varint(n, &mut out);
            assert_eq!(decode_varint(&out), Some((n, out.len())), "n = {n}");
        }
    }

    #[test]
    fn missing_last_byte_is_none() {
        assert_eq!(decode_varint(&[0xe8]), None);
    }
}
```

`cargo test --lib` with all exercises of this phase so far:

```text
test result: ok. 13 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.02s
```

Why it works: `n as u8 & 0x7f` keeps the lowest 7 bits; `| 0x80` marks "more follows". The decoder puts each group of
7 bits back in its place with a shift. `take(10)` stops a damaged input with all top bits set from looping forever or
shifting past 64 bits.

</details>

## Quiz

1. Why do storage engines prefer `u32` bytes to the text `"1000"`?

<details>
<summary>Answer</summary>

Fixed size (always 4 bytes, so offsets can be computed without reading), and decoding is a copy instead of parsing digits
and searching for the end.

</details>

2. What are the bytes of the `u16` value 258 (`0x0102`) in little-endian?

<details>
<summary>Answer</summary>

`[0x02, 0x01]`: the lowest byte first.

</details>

3. What two problems does a length prefix solve that a separator character (like `\n`) does not?

<details>
<summary>Answer</summary>

The data may contain the separator itself (a value with a newline), and with a length the reader knows the size before
reading, so it can check it, allocate once and detect a cut record.

</details>

4. A record's last byte was flipped on disk. Which error does `decode` return, and why not `Truncated`?

<details>
<summary>Answer</summary>

`BadChecksum`. All bytes are there, so the lengths are fine; but the CRC computed from the bytes no longer matches the
stored one.

</details>

5. Can a CRC-32 protect a file against an attacker who edits it?

<details>
<summary>Answer</summary>

No. The attacker can simply compute the CRC of the new bytes. You need a MAC with a secret key (Lesson 10.9).

</details>

6. Why is 300 encoded by postcard in two bytes, and how many bytes does 100 take?

<details>
<summary>Answer</summary>

postcard uses varints with 7 bits per byte. 300 needs 9 bits, so two bytes. 100 fits into 7 bits, so it takes one byte.

</details>

## Summary

- Storage engines keep data in binary form: fixed-size fields, no parsing, exact bytes.
- Choose a byte order for every file format and write it down; this course uses little-endian (`to_le_bytes`,
  `from_le_bytes`).
- Length prefixes tell the reader where each record ends; a maximum length protects against damaged lengths.
- A CRC-32 checksum catches torn and damaged records; it is not protection against attackers.
- A decoder checks in a safe order and returns errors (`Truncated`, `TooLarge`, `BadChecksum`), never panics.
- `serde` + `postcard` serialize whole structs compactly, with varints for integers; `bincode` is no longer maintained.
- Real formats start with a magic number and a version, so that old files stay readable.

## Sources and further reading

- [Rust primitive type u32](https://doc.rust-lang.org/std/primitive.u32.html) — `to_le_bytes`, `from_le_bytes` and the
  other byte-order methods.
- [crc32fast documentation](https://docs.rs/crc32fast/latest/crc32fast/) — `hash` and the incremental `Hasher`.
- [postcard documentation](https://docs.rs/postcard/latest/postcard/) and
  [the postcard wire format](https://postcard.jamesmunns.com/wire-format) — exactly which bytes each Rust type becomes.
- [serde.rs](https://serde.rs/) — the serde book: derives, attributes and data formats.
- [LevelDB log format](https://github.com/google/leveldb/blob/main/doc/log_format.md) — a real storage engine's record
  format with checksums and lengths, close to ours.
- [SQLite database file format](https://www.sqlite.org/fileformat.html) — a complete, well-documented binary file format,
  starting with its magic header.

---

[🏠 Start](../START-HERE.md) · [← 17.2 Durability: fsync and crash safety](02-durability-and-fsync.md) · [📋 Phase 17](README.md) · [17.4 The write-ahead log (WAL) →](04-write-ahead-log.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
