<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 10.11 TLS and certificates](11-tls-and-certificates.md) · [📋 Phase 10](README.md) · [10.13 Supply-chain security: cargo-audit and cargo-deny →](13-supply-chain-security.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/10-security-fundamentals/12-memory-safety-and-reviewing-unsafe.md) <!-- nav -->

# Lesson 10.12 — Memory safety and reviewing unsafe code

> **Phase 10 — Security fundamentals** · Lesson 12 of 15 · ⏱ about 135 minutes

## What you will learn

- Which memory bugs safe Rust rules out, and which security problems it does **not** prevent.
- How a panic, a huge announced size or deeply nested input turns safe code into a denial-of-service hole, and how to fix each one.
- How an unchecked integer calculation becomes a wrong size, and how `checked_*` fixes it.
- How to review `unsafe` code with a checklist, and how to catch undefined behaviour with Miri and AddressSanitizer.
- Which Clippy lints help you find risky code early.

## Before you start

- You finished [Lesson 10.11 — TLS and certificates](11-tls-and-certificates.md).
- You know ownership, the stack and the heap ([Lesson 03.1](../03-ownership-and-borrowing/01-memory-stack-and-heap.md)), integer overflow ([Lesson 02.3](../02-programming-basics/03-operators-and-expressions.md)), and error handling ([Lesson 04.10](../04-structs-enums-collections-errors/10-error-handling.md)).
- You know what `unsafe` means, `// SAFETY:` comments, undefined behaviour and Miri ([Lesson 06.6](../06-rust-advanced/06-unsafe-rust-and-miri.md)). This lesson does not teach `unsafe` again: it shows how to **review** it.
- Install the nightly toolchain with Miri once (stable stays your default), exactly as in Lesson 06.6:

```bash
rustup toolchain install nightly --component miri
```

- Create the practice package. Example programs go in `src/bin/`:

```bash
cd ~/projects/rust
cargo new code/p10-memory
cd code/p10-memory
mkdir src/bin
```

Add these lines at the end of `Cargo.toml`. They ask Clippy (the Rust linter) to warn about code that can panic or overflow, and about `unsafe` blocks without a comment:

```toml
[lints.clippy]
indexing_slicing = "warn"
unwrap_used = "warn"
arithmetic_side_effects = "warn"
undocumented_unsafe_blocks = "warn"
```

> 🔒 **Ethics:** all "attacks" here are inputs that you write yourself, against your own small programs.

## New words

| Word | Meaning |
|---|---|
| **memory safety** | A program only reads and writes memory it owns and that is still valid. Safe Rust guarantees it. |
| **buffer overflow** | Reading or writing past the end of an array or buffer. |
| **use after free** | Using memory after it was released. |
| **DoS** (Denial of Service) | An attack that makes a service unavailable: it crashes, hangs or uses all memory. |
| **resource exhaustion** | A DoS where the attacker makes your program use up memory, CPU (Central Processing Unit) time or disk. |
| **sound / unsound** | A safe function is *sound* if **no** input can cause undefined behaviour. If some input can, it is *unsound*, and that is a bug even if your own code never passes that input. |
| **sanitizer** | A tool built into the compiler that adds checks to your program while it runs, and reports memory errors. |
| **ASan** (AddressSanitizer) | A sanitizer that finds out-of-bounds accesses and use after free. |
| **stack overflow** | The call stack ran out of space (for example in endless or very deep recursion). Rust aborts the process. |
| **MiB** (mebibyte) | 1,048,576 bytes (1024 × 1024). *GiB* is 1024 MiB. |

## 1. What safe Rust prevents, and what is left

Memory bugs are behind many serious vulnerabilities: buffer overflows, use after free, double free, reading uninitialised memory and data races.
Safe Rust makes them **impossible**: the compiler checks ownership, borrowing and lifetimes (Phase 03), and arrays are bounds-checked at run time.

That is a huge win, but it is not a guarantee of "no security bugs":

| Problem | Safe Rust prevents it? | What you do |
|---|---|---|
| Buffer overflow, use after free, data race | **Yes** | Stay in safe code. |
| Panic on bad input (index, `unwrap`, division by zero) | No: the program *stops* | Return errors for untrusted input (Section 2). |
| Memory or CPU exhaustion from huge sizes | No | Limit every size that comes from outside (Section 3). |
| Stack overflow from deep recursion | No: the process aborts | Limit depth (Section 4). |
| Integer overflow producing a wrong size | No (wraps in release builds) | `checked_*` arithmetic (Section 3). |
| Logic bugs: missing authorization, SQL (Structured Query Language) injection, weak crypto | No | The rest of this phase. |
| Bugs inside `unsafe` blocks, in your code or in a dependency | No | Review and test them (Section 5). |

## 2. Panics are a denial of service

A panic ends the thread; if it happens on the main thread, or in a server that does not catch it, the whole program stops. If **untrusted input can cause a panic**, anyone can crash your service with one request. Create `src/bin/panic_dos.rs`:

```rust
/// Reads a 2-byte big-endian length from the start of a message.
fn read_len(message: &[u8]) -> u16 {
    u16::from_be_bytes([message[0], message[1]])
}

fn main() {
    println!("{}", read_len(&[0x01, 0x02, 0xff]));
    println!("{}", read_len(&[0x01]));
    println!("never reached");
}
```

```text
258

thread 'main' (4767495) panicked at src/bin/panic_dos.rs:3:37:
index out of bounds: the len is 1 but the index is 1
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
```

**What happens:**

1. `message[0]` and `message[1]` are bounds-checked: that is why this is *memory safe* (no out-of-bounds read happens).
2. But the check **panics** when the message has only one byte. The program stops before `never reached`. A network service reading this from a socket would be crashed by a one-byte packet.

(The thread number in the message differs on your computer.) Run `cargo clippy` and the lint `indexing_slicing` points to both places:

```text
warning: indexing may panic
 --> src/bin/panic_dos.rs:3:25
  |
3 |     u16::from_be_bytes([message[0], message[1]])
  |                         ^^^^^^^^^^
  |
  = help: consider using `.get(n)` or `.get_mut(n)` instead
```

The fix is to treat short input as a normal, expected situation and return an `Option` (or a `Result` with an error type, Lesson 04.10). Create `src/bin/panic_fixed.rs`:

```rust
/// Reads a 2-byte big-endian length from the start of a message.
/// Short input is an expected situation, so the answer is an `Option`, not a panic.
fn read_len(message: &[u8]) -> Option<u16> {
    let [a, b, ..] = message else {
        return None;
    };
    Some(u16::from_be_bytes([*a, *b]))
}

fn main() {
    println!("{:?}", read_len(&[0x01, 0x02, 0xff]));
    println!("{:?}", read_len(&[0x01]));
    println!("still running");
}
```

```text
Some(258)
None
still running
```

**What happens:** the pattern `[a, b, ..]` matches a slice with **at least** two elements and binds the first two; `let … else` runs the `else` block when it does not match. No indexing, so no panic, and Clippy is quiet.

> 📏 **Convention:** `unwrap`, `expect`, `[i]` and `panic!` are fine for things that are *your* invariants ("this cannot be `None` because I just checked") and in tests. They are not fine for anything that depends on input from a file, a network, a user or another program.

> 🧪 **Testing:** test parsers with **short, empty and absurd** inputs. A good test calls the function with every length from 0 to a few bytes and checks that it never panics (Exercise 2).

## 3. Untrusted sizes

### Integer overflow makes wrong sizes

Lesson 02.3 showed that arithmetic overflow panics in debug builds and wraps around in release builds. For a size taken from input this becomes a security bug. Create `src/bin/overflow.rs`:

```rust
/// Total bytes needed for `count` records of `size` bytes each.
fn total_bytes(count: u32, size: u32) -> u32 {
    count * size
}

fn main() {
    println!("{}", total_bytes(1000, 4096));
    // An attacker announces 2 million records of 4096 bytes: the true total is 8,192,000,000.
    let total = total_bytes(2_000_000, 4096);
    println!("allocating {total} bytes");
}
```

Compare `cargo run --bin overflow` (debug) with `cargo run --release --bin overflow`:

```text
4096000

thread 'main' (4784981) panicked at src/bin/overflow.rs:3:5:
attempt to multiply with overflow
```

```text
4096000
allocating 3897032704 bytes
```

**What happens:** in the debug build the overflow panics (a DoS, but at least it is noticed). In the release build the result silently wraps around: 8,192,000,000 does not fit in a `u32` (maximum 4,294,967,295), and the program continues with 3,897,032,704, a number that is wrong by 4.3 billion. A program that allocates `total` bytes and then copies `count × size` bytes would write far past its buffer
(in safe Rust, a bounds check stops it; in `unsafe` code it would be a classic heap overflow). The fix: `checked_mul` returns `None` on overflow. Create `src/bin/overflow_fixed.rs`:

```rust
/// Total bytes needed, or `None` if the multiplication does not fit.
fn total_bytes(count: u32, size: u32) -> Option<u32> {
    count.checked_mul(size)
}

fn main() {
    println!("{:?}", total_bytes(1000, 4096));
    println!("{:?}", total_bytes(2_000_000, 4096));
}
```

```text
Some(4096000)
None
```

> 📏 **Convention:** for arithmetic on values that come from outside, use `checked_add`, `checked_mul`, `checked_sub` (or `saturating_*` when clamping is correct), and convert with `try_from` instead of `as` (`as` silently cuts off high bits). The lint `arithmetic_side_effects` finds the plain `+`, `-` and `*` that you should look at again.

### Never trust a length

A very common format is "length, then data": the first 4 bytes say how many bytes follow. If you create a buffer of the announced size **before** the data has arrived, an attacker needs only 4 bytes to make your program ask for 4 GiB (gibibytes). Do this instead:

1. Set a **maximum** that makes sense for your protocol, and reject larger values.
2. Allocate with `try_reserve` so a failure is an error and not an abort.
3. Read with `take(len)` so you never read more than announced.

Create `src/bin/frame_alloc.rs`:

```rust
use std::io::{self, Cursor, Read};

const MAX_FRAME: usize = 1024 * 1024; // 1 MiB

/// Reads one frame: a 4-byte big-endian length, then that many bytes.
fn read_frame(input: &mut impl Read) -> io::Result<Vec<u8>> {
    let mut len_bytes = [0u8; 4];
    input.read_exact(&mut len_bytes)?;
    let len = u32::from_be_bytes(len_bytes) as usize;

    if len > MAX_FRAME {
        return Err(io::Error::new(io::ErrorKind::InvalidData, format!("frame of {len} bytes is too large")));
    }
    let mut frame = Vec::new();
    frame.try_reserve_exact(len).map_err(|_| io::Error::new(io::ErrorKind::OutOfMemory, "cannot allocate"))?;
    // `take` makes sure we never read more than `len` bytes, even if the input is endless.
    input.take(len as u64).read_to_end(&mut frame)?;
    if frame.len() != len {
        return Err(io::Error::new(io::ErrorKind::UnexpectedEof, "frame is shorter than announced"));
    }
    Ok(frame)
}

fn main() {
    // A good frame: length 5, then "hello".
    let mut good = Cursor::new(vec![0, 0, 0, 5, b'h', b'e', b'l', b'l', b'o']);
    println!("good : {:?}", read_frame(&mut good));

    // A hostile frame: only 4 bytes of input, but it announces 4 GiB.
    let mut evil = Cursor::new(vec![0xff, 0xff, 0xff, 0xff]);
    println!("evil : {:?}", read_frame(&mut evil));

    // A lying frame: announces 100 bytes, delivers 3.
    let mut short = Cursor::new(vec![0, 0, 0, 100, 1, 2, 3]);
    println!("short: {:?}", read_frame(&mut short));

    // What the allocator says to an absurd request when we ask politely with try_reserve:
    let mut v: Vec<u8> = Vec::new();
    println!("1 PiB: {:?}", v.try_reserve_exact(1 << 50).map_err(|e| e.to_string()));
}
```

```text
good : Ok([104, 101, 108, 108, 111])
evil : Err(Custom { kind: InvalidData, error: "frame of 4294967295 bytes is too large" })
short: Err(Custom { kind: UnexpectedEof, error: "frame is shorter than announced" })
1 PiB: Err("memory allocation failed because the memory allocator returned an error")
```

**What happens:**

1. `read_exact` reads exactly 4 bytes or fails, and `u32::from_be_bytes` turns them into the length. `as usize` is safe here: `u32` always fits in `usize` on 64-bit systems.
2. The limit check rejects the 4 GiB announcement immediately, using only 4 bytes of input. `MAX_FRAME` is a design decision: pick the largest value your protocol really needs.
3. `try_reserve_exact` asks for the memory and returns `Err` instead of aborting when the system cannot give it (the last line shows an absurd request of 1 PiB, pebibyte, which fails cleanly).
4. `input.take(len as u64)` is a reader that stops after `len` bytes. The check afterwards catches an input that ends early.

> ⚡ **Performance:** `Vec::with_capacity(len)` and `vec![0; len]` with an untrusted `len` are the usual mistakes. The same rule applies to decompression ("zip bombs": a tiny file that expands to terabytes), JSON (JavaScript Object Notation) arrays and `read_to_end` on a socket. Always put a limit on the **output** size, with `Read::take` or a counter.

## 4. Untrusted depth

Recursion uses the call stack, which is small (about 8 MiB for the main thread on macOS; check with `ulimit -s`). A parser for nested data (brackets, JSON, XML: two common text formats for structured data, JavaScript Object Notation and Extensible Markup Language) that calls itself for each level can be driven off the end of the stack by input that is only one megabyte long. Create `src/bin/nesting.rs`:

```rust
/// A tree made of nested brackets: "((()))" is Group(Group(Group(Empty))).
#[allow(dead_code)]
enum Node {
    Empty,
    Group(Box<Node>),
}

/// Parses `(` ... `)` recursively. Looks innocent, right?
fn parse(input: &[u8], pos: &mut usize) -> Option<Node> {
    if input.get(*pos) == Some(&b'(') {
        *pos += 1;
        let inner = parse(input, pos)?;
        if input.get(*pos) != Some(&b')') {
            return None;
        }
        *pos += 1;
        Some(Node::Group(Box::new(inner)))
    } else {
        Some(Node::Empty)
    }
}

fn main() {
    let mut pos = 0;
    let small = parse(b"((()))", &mut pos);
    println!("small input parsed: {}", small.is_some());

    // An attacker sends one million opening brackets: only one megabyte of input.
    let evil = vec![b'('; 1_000_000];
    let mut pos = 0;
    let result = parse(&evil, &mut pos);
    println!("evil input parsed: {}", result.is_some());
}
```

```text
small input parsed: true

thread 'main' (4773359) has overflowed its stack
fatal runtime error: stack overflow, aborting
```

**What happens:** every `(` makes one more nested call to `parse`, each with its own stack frame. One million levels need far more than 8 MiB, so the process **aborts**. Unlike a panic, a stack overflow cannot be caught. The fix is a depth limit, passed down as a parameter. Create `src/bin/nesting_fixed.rs` (the new parts are marked):

```rust
const MAX_DEPTH: usize = 128; // new

#[allow(dead_code)]
enum Node {
    Empty,
    Group(Box<Node>),
}

/// Same parser, but it refuses to go deeper than MAX_DEPTH.
fn parse(input: &[u8], pos: &mut usize, depth: usize) -> Option<Node> { // new parameter
    if depth > MAX_DEPTH {
        return None; // too deep: reject the input instead of crashing
    }
    if input.get(*pos) == Some(&b'(') {
        *pos += 1;
        let inner = parse(input, pos, depth + 1)?;
        if input.get(*pos) != Some(&b')') {
            return None;
        }
        *pos += 1;
        Some(Node::Group(Box::new(inner)))
    } else {
        Some(Node::Empty)
    }
}

fn main() {
    let mut pos = 0;
    println!("small input parsed: {}", parse(b"((()))", &mut pos, 0).is_some());

    let evil = vec![b'('; 1_000_000];
    let mut pos = 0;
    println!("evil input parsed: {}", parse(&evil, &mut pos, 0).is_some());
}
```

```text
small input parsed: true
evil input parsed: false
```

Well-known parsers do exactly this: `serde_json`, for example, stops at a nesting depth of 128 by default.

## 5. Reviewing `unsafe` code

Inside an `unsafe` block, the compiler trusts **you**. A review has one goal: to be convinced that *no input* from outside, however hostile, can break the rules. A checklist:

| Question | Why |
|---|---|
| Is `unsafe` needed at all? | Most "I need `unsafe` for speed" cases disappear with `get`, iterators or slices. The best review result is deleting the block. |
| Does every `unsafe` block have a `// SAFETY:` comment that explains **why** the rules hold here (not what the code does)? | If you cannot write the reason, the code is not safe. |
| Can a **safe** function reach the `unsafe` block with any input? | Then the function must check everything the block needs. Otherwise it is *unsound*, or it must become `unsafe fn` with a `# Safety` section. |
| Where do indexes, lengths and pointers come from? | Anything that comes from input (a header field, a file) must be checked against the real size first. |
| Which calls are the classic traps? | `get_unchecked`, `from_raw_parts`, `ptr.add`/`offset`, `set_len`, `assume_init`, `transmute`, `from_utf8_unchecked`. |
| Are there tests that run under Miri, with hostile inputs? | Miri checks the actual execution against the rules. |

Find all `unsafe` in your own code with `grep -rn "unsafe" src/`. If a crate needs none, forbid it with `#![forbid(unsafe_code)]` at the top of `lib.rs` or `main.rs`: the compiler then rejects any `unsafe`. Dependencies can contain `unsafe` too (Lesson 10.13).

Here is a function that looks safe from the outside, but is not. Create `src/bin/unsound.rs`:

```rust
/// Returns the byte at `index`. Safe to call... or is it?
pub fn byte_at(data: &[u8], index: usize) -> u8 {
    // SAFETY: the caller passes a valid index.
    unsafe { *data.as_ptr().add(index) }
}

fn main() {
    let data = vec![10u8, 20, 30];
    println!("{}", byte_at(&data, 1));
    println!("{}", byte_at(&data, 5)); // index 5 of a 3-byte vector
}
```

**What happens:** `byte_at` is a *safe* function, so any caller may pass any `index`. The comment "the caller passes a valid index" is a wish, not a proof: nothing makes it true. Index 5 is outside the 3-byte allocation, so the read is **undefined behaviour**. Look at what four different ways of running it do:

```bash
cargo run --quiet --bin unsound
cargo run --quiet --release --bin unsound
```

Output of the debug build, then of the release build:

```text
20
0
```

```text
20
20
```

The debug build printed `0` and the release build printed `20` for index 5: no error, no warning, and **different wrong answers**. This is what undefined behaviour looks like: it "works". Now run it under Miri (Lesson 06.6):

```bash
cargo +nightly miri run --quiet --bin unsound
```

```text
20
error: Undefined Behavior: in-bounds pointer arithmetic failed: attempting to offset pointer by 5 bytes, but got alloc290 which is only 3 bytes from the end of the allocation
 --> src/bin/unsound.rs:4:15
  |
4 |     unsafe { *data.as_ptr().add(index) }
  |               ^^^^^^^^^^^^^^^^^^^^^^^^ Undefined Behavior occurred here
  |
help: alloc290 was allocated here:
 --> src/bin/unsound.rs:8:16
  |
8 |     let data = vec![10u8, 20, 30];
  |                ^^^^^^^^^^^^^^^^^^
```

Miri interprets the program and checks every pointer operation, so it finds the exact line. The second tool is **AddressSanitizer**: the compiler adds checks around every memory access, and the program runs at near normal speed. It needs the nightly compiler and the name of your target:

```bash
RUSTFLAGS="-Zsanitizer=address" cargo +nightly run --quiet --target aarch64-apple-darwin --bin unsound 2>&1 | grep -E "^[0-9]|ERROR|READ of|#0 |#1 |located"
```

```text
20
==54022==ERROR: AddressSanitizer: heap-buffer-overflow on address 0x602000000935 at pc 0x0001006bd71c bp 0x00016f741820 sp 0x00016f741818
READ of size 1 at 0x602000000935 thread T0
    #0 0x0001006bd718 in _RNvCskCVANAyUgAU_7unsound7byte_at unsound.rs:4
    #1 0x0001006bd470 in _RNvCskCVANAyUgAU_7unsound4main unsound.rs:10
0x602000000935 is located 2 bytes after 3-byte region [0x602000000930,0x602000000933)
```

**What happens:** `-Zsanitizer=address` turns on ASan (the `-Z` flags exist only on nightly). ASan reports a *heap-buffer-overflow*: a read of 1 byte, 2 bytes after a 3-byte region, in `byte_at` on line 4, called from line 10. (Addresses and numbers differ on every run.)

Which tool when? Miri is the most precise about Rust's rules but slow and it cannot run code that talks to the operating system or C libraries. ASan is fast and works with ordinary programs and C code, but it only sees accesses that hit memory outside allocations. Use both on tests of `unsafe` code. On Linux, more sanitizers exist (for example ThreadSanitizer for data races).

The fix is to make the function **sound**. Best: delete the `unsafe`.

```rust
pub fn byte_at(data: &[u8], index: usize) -> Option<u8> {
    data.get(index).copied()
}
```

If you must keep `unsafe` (a measured hot path, Lesson 09.11), the safe function must check what the block relies on, and the comment must name that check:

```rust
pub fn byte_at(data: &[u8], index: usize) -> Option<u8> {
    if index < data.len() {
        // SAFETY: `index < data.len()` was just checked, so the pointer stays inside the slice.
        Some(unsafe { *data.as_ptr().add(index) })
    } else {
        None
    }
}
```

Or make the contract the caller's job: `pub unsafe fn byte_at_unchecked(...)` with a `# Safety` section ("`index` must be less than `data.len()`"). Now nobody can call it from safe code by accident.

> 🔒 **Security:** "it works in my tests" is not a review result for `unsafe` code. Run the tests under Miri, use hostile inputs (empty, huge, off by one), and keep each `unsafe` block as small as possible.

## Common mistakes

**1. `unwrap()` on input.** `let n: u32 = text.parse().unwrap();` lets anyone who sends `abc` crash the service. Use `?` or `match` and return an error.

**2. `as` for sizes.** `len as u32` silently cuts a `usize` to 32 bits, so a 4,294,967,297-byte input looks like 1 byte long. Use `u32::try_from(len)?`.

**3. Allocating from an announced size.** `vec![0; announced_len]` and `Vec::with_capacity(announced_len)` before you check `announced_len`.

**4. Trusting the comment above `unsafe`.** A `// SAFETY:` comment is a claim; review it like code. "The caller guarantees this" is only valid for an `unsafe fn`.

**5. Testing only the happy path.** Parsers need tests with empty, one-byte, truncated, over-long and deeply nested inputs.

## Try it yourself

1. Run `cargo clippy` in the package and read each warning. Which lints fire on `overflow.rs`? (`arithmetic_side_effects` points at `count * size`.)
2. Change `MAX_DEPTH` in `nesting_fixed.rs` to `10_000_000` and run it. What happens, and why?
3. Replace `byte_at` in `unsound.rs` with the fixed version that returns an `Option` (change the `println!` calls to `{:?}`) and run it under Miri again.

## Exercises

### Exercise 1 — Two tools, one bug (easy)

Run `unsound.rs` under Miri and under AddressSanitizer. For each tool write down: the kind of error, and the source line it points to.

<details>
<summary>Hint</summary>

Use the two commands from Section 5. Look for the first line that names a file and a line number.

</details>

<details>
<summary>Answer</summary>

Miri: `Undefined Behavior: in-bounds pointer arithmetic failed: attempting to offset pointer by 5 bytes, but got alloc290 which is only 3 bytes from the end of the allocation`, at `src/bin/unsound.rs:4:15`, with a note that the allocation was made at line 8.

ASan: `heap-buffer-overflow`, `READ of size 1`, `2 bytes after 3-byte region`, in `byte_at` at `unsound.rs:4`, called from `main` at `unsound.rs:10`.

Why it works: both report the same bug (a read outside the 3-byte allocation) in different words. Miri checks Rust's rules in an interpreter; ASan checks real memory accesses in the compiled program.

</details>

### Exercise 2 — A parser that never panics (medium)

In a new library package write `pub fn read_len(message: &[u8]) -> Option<u16>` (the fixed function from Section 2). Write a test that calls it with inputs of every length from 0 to 3 and checks that it returns `Some` only when there are at least two bytes, and one test for the value `258`.

<details>
<summary>Hint</summary>

`vec![0xff; size]` makes an input of any length. `result.is_some() == (size >= 2)` is the property.

</details>

<details>
<summary>Answer</summary>

```rust
/// Reads a 2-byte big-endian length. Never panics: short input gives `None`.
pub fn read_len(message: &[u8]) -> Option<u16> {
    let [a, b, ..] = message else {
        return None;
    };
    Some(u16::from_be_bytes([*a, *b]))
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn read_len_never_panics_on_short_input() {
        for size in 0..4 {
            let input = vec![0xff; size];
            let result = read_len(&input);
            assert_eq!(result.is_some(), size >= 2, "size {size}");
        }
        assert_eq!(read_len(&[0x01, 0x02, 0xff]), Some(258));
    }
}
```

`cargo test` shows the test as `ok`.

Why it works: the test covers the boundary (sizes 1, 2) and the empty input, exactly where an indexing version would panic.

</details>

### Exercise 3 — A frame reader with a limit (hard)

Add `read_frame` (Section 3) with `MAX_FRAME` to your library as a `pub fn`. Write tests with `std::io::Cursor`: a good frame, an announced length of 4 GiB (must be `InvalidData`, and must not allocate), a frame shorter than announced and an empty input (both `UnexpectedEof`).

<details>
<summary>Hint</summary>

`io::Error::kind()` returns the `ErrorKind`. `Cursor::new(vec![...])` is a `Read` that reads from memory. Reuse the function from `frame_alloc.rs`.

</details>

<details>
<summary>Answer</summary>

The function is the one from Section 3 (make it `pub`). The tests, inside `mod tests` with `use std::io::Cursor;`:

```rust
    #[test]
    fn good_frame_is_read() {
        let mut input = Cursor::new(vec![0, 0, 0, 5, b'h', b'e', b'l', b'l', b'o']);
        assert_eq!(read_frame(&mut input).unwrap(), b"hello");
    }

    #[test]
    fn huge_announced_length_is_rejected_without_allocating() {
        let mut input = Cursor::new(vec![0xff, 0xff, 0xff, 0xff]);
        let err = read_frame(&mut input).unwrap_err();
        assert_eq!(err.kind(), io::ErrorKind::InvalidData);
    }

    #[test]
    fn short_and_empty_inputs_are_errors() {
        let mut short = Cursor::new(vec![0, 0, 0, 100, 1, 2, 3]);
        assert_eq!(read_frame(&mut short).unwrap_err().kind(), io::ErrorKind::UnexpectedEof);
        let mut empty = Cursor::new(Vec::new());
        assert_eq!(read_frame(&mut empty).unwrap_err().kind(), io::ErrorKind::UnexpectedEof);
    }
```

`cargo test` shows all tests `ok`.

Why it works: `read_exact` on an empty input fails with `UnexpectedEof` before any length is known, and the size limit stops the 4 GiB request after reading only 4 bytes.

</details>

## Quiz

1. Which of these does safe Rust **not** prevent: use after free, a panic from `data[10]`, a data race?

<details>
<summary>Answer</summary>

A panic from `data[10]`. Safe Rust makes the access memory safe by stopping the program; that is still a denial of service if the index comes from untrusted input.

</details>

2. A message starts with a 4-byte length. What do you do before allocating a buffer?

<details>
<summary>Answer</summary>

Compare the length with a sensible maximum and reject larger values; then allocate with `try_reserve` and read with `take`. Never allocate from a number that came from outside without a limit.

</details>

3. Why does `count * size` need `checked_mul` for values from input?

<details>
<summary>Answer</summary>

In release builds the multiplication wraps around silently and gives a wrong, smaller size. `checked_mul` returns `None` on overflow, so you can reject the input.

</details>

4. What makes a safe function that contains `unsafe` *unsound*?

<details>
<summary>Answer</summary>

If some input that a safe caller can pass causes undefined behaviour, for example an unchecked index. A safe function must be correct for **every** input.

</details>

5. When would you use Miri, and when AddressSanitizer?

<details>
<summary>Answer</summary>

Miri for small Rust tests of `unsafe` code: it is precise but slow and cannot call C code. AddressSanitizer for the compiled program, including larger programs and C libraries, to find out-of-bounds accesses and use after free. Use both on `unsafe` code.

</details>

## Summary

- Safe Rust removes buffer overflows, use after free and data races, but not panics, resource exhaustion, stack overflow, integer overflow or logic bugs.
- Untrusted input must never cause a panic: return `Option` or `Result`, avoid `[i]` and `unwrap` on input, and test short, empty and absurd inputs.
- Put a limit on every size and depth that comes from outside: allocation sizes, output sizes, recursion depth.
- Use `checked_*` arithmetic and `try_from` for sizes from input.
- A safe function with `unsafe` inside must be sound for every possible input; a `// SAFETY:` comment must give a reason that is actually true.
- Undefined behaviour often "works"; use Miri and AddressSanitizer to find it, and Clippy lints (`indexing_slicing`, `unwrap_used`, `arithmetic_side_effects`, `undocumented_unsafe_blocks`) to find risky code early.

## Sources and further reading

- [The Rustonomicon](https://doc.rust-lang.org/nomicon/) — the official guide to writing and reviewing `unsafe` code, including what soundness means.
- [Miri on GitHub](https://github.com/rust-lang/miri) — installation, flags and the list of what Miri detects.
- [The Rust Unstable Book: sanitizer](https://doc.rust-lang.org/beta/unstable-book/compiler-flags/sanitizer.html) — which sanitizers exist and how to enable them.
- [Clippy lint list](https://rust-lang.github.io/rust-clippy/master/index.html) — documentation of `indexing_slicing`, `unwrap_used`, `arithmetic_side_effects` and `undocumented_unsafe_blocks`.
- [Rust Secure Code Working Group](https://github.com/rust-secure-code/wg) — guidelines and tools for writing secure Rust.
- [std::vec::Vec::try_reserve](https://doc.rust-lang.org/std/vec/struct.Vec.html#method.try_reserve) — fallible allocation.

---

[🏠 Start](../START-HERE.md) · [← 10.11 TLS and certificates](11-tls-and-certificates.md) · [📋 Phase 10](README.md) · [10.13 Supply-chain security: cargo-audit and cargo-deny →](13-supply-chain-security.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
