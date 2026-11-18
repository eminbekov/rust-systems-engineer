<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 12.12 Linking and loading programs](12-linking-and-loading.md) · [📋 Phase 12](README.md) · [12.14 Project: your own memory allocator →](14-project-your-own-allocator.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/12-computer-systems/13-how-memory-allocators-work.md) <!-- nav -->

# Lesson 12.13 — How memory allocators work

> **Phase 12 — Computer systems** · Lesson 13 of 15 · ⏱ about 135 minutes

## What you will learn

- What a memory allocator is, and the path from `Box::new` down to the operating system.
- How to describe a block of memory with `Layout`, and how to allocate and free by hand.
- How the `GlobalAlloc` trait and `#[global_allocator]` let you replace Rust's allocator, with a counting allocator as the first example.
- How a bump allocator and a free-list allocator work, and what fragmentation is.
- How real allocators (the system `malloc`, mimalloc, jemalloc) stay fast, and how Miri catches allocator bugs.

## Before you start

- You finished [Lesson 12.12 — Linking and loading programs](12-linking-and-loading.md) and [Lesson 12.11 — Memory layout of Rust types](11-memory-layout-of-rust-types.md) (size and alignment).
- You know the heap and `Box` from [Lesson 03.1](../03-ownership-and-borrowing/01-memory-stack-and-heap.md) and [Lesson 05.9 — Smart pointers](../05-rust-intermediate/09-smart-pointers.md), raw pointers, `unsafe` and Miri from [Lesson 06.6](../06-rust-advanced/06-unsafe-rust-and-miri.md), and atomics from [Lesson 05.10](../05-rust-intermediate/10-threads-channels-and-shared-state.md).
- Miri needs the nightly toolchain (`rustup toolchain install nightly --component miri`, as in Lesson 06.6).
- Create the package. As before, each example goes into `src/main.rs`; the reusable code goes into `src/lib.rs`:

```bash
cd ~/projects/rust
cargo new code/p12-alloc-basics
cd code/p12-alloc-basics
```

## New words

| Word | Meaning |
|---|---|
| **allocator** | The code that hands out blocks of heap memory on request and takes them back when they are freed. |
| **allocate / deallocate (free)** | Ask the allocator for a block / give the block back. |
| **block** | One piece of memory handed out by the allocator. |
| **`Layout`** | A Rust type (`std::alloc::Layout`) that describes a block: its size and its alignment. |
| **global allocator** | The one allocator that `Box`, `Vec`, `String` and all other standard collections use in a program. |
| **`GlobalAlloc`** | The trait a type must implement to become the global allocator. |
| **`System`** | The allocator of the operating system (`malloc` and `free` of the C library). Rust uses it by default. |
| **arena** | A big region of memory that an allocator cuts into blocks. |
| **bump allocator** | An allocator that hands out blocks by moving one pointer forward and frees everything at once. |
| **free list** | A linked list of the free blocks, kept by the allocator so that freed memory can be reused. |
| **header** | A few bytes in front of (or inside) a block where the allocator stores its own information, such as the block size. |
| **splitting / coalescing** | Cutting a big free block into the part you need and a smaller free rest / joining neighbouring free blocks into one. |
| **fragmentation** | Wasted memory: inside blocks (*internal*: padding, rounding) or between blocks (*external*: free memory cut into pieces too small to use). |
| **size class** | A fixed block size (16, 32, 48, … bytes). Real allocators round every request up to a size class and keep one free list per class. |
| **OOM** (Out Of Memory) | The situation where the allocator cannot provide the memory you asked for. |

## 1. From `Box::new` to the operating system

When you write `Box::new(5_u64)`, Rust needs 8 bytes of heap memory. It does not ask the operating system directly; that would be slow, because every request to the kernel is a system call (Phase 13). Instead there is a layer in between:

```text
   your code:     Box::new, Vec::push, String::from, format!, …
                        │ alloc(Layout { size: 8, align: 8 })
                        ▼
   global allocator:  by default `System` = malloc / free of the C library
                        │ only when its own reserve runs out:
                        ▼ "give me 1 MiB more"
   operating system:  mmap (maps fresh pages into the process)
```

The allocator gets big regions from the operating system now and then, and cuts them into small blocks for you, very often and very quickly. When you free a block, it usually does **not** go back to the operating system; the allocator keeps it and reuses it for a later request.

Every allocator must answer two questions fast: "where is a free block of this size and alignment?" and "where does this freed block go?" The rest of this lesson shows several answers.

> ⚡ **Performance:** An allocation takes from a few nanoseconds to much more, depending on the allocator and the size, and every allocation may touch memory that is not in the cache. Code that allocates in a hot loop is often slower because of the allocator, not because of its own work. Section 3 shows how to count allocations.

## 2. `Layout` and allocating by hand

The allocator needs two numbers: how many bytes, and which alignment (Lesson 12.11). `std::alloc::Layout` keeps both and checks that they make sense (the alignment is a power of two, the size does not overflow). You can call the global allocator yourself with `std::alloc::alloc` and `dealloc`. This is `unsafe`, because the compiler cannot check that you free the block exactly once, with the same layout, and never use it afterwards. Put this into `src/main.rs`:

```rust
use std::alloc::{Layout, alloc, dealloc};

fn main() {
    // Ask for room for 4 u32 values, aligned for u32.
    let layout = Layout::array::<u32>(4).expect("size fits in isize");
    println!("size {} bytes, align {}", layout.size(), layout.align());

    // SAFETY: the layout has a non-zero size.
    let ptr = unsafe { alloc(layout) } as *mut u32;
    if ptr.is_null() {
        std::alloc::handle_alloc_error(layout);
    }

    // SAFETY: ptr points to room for 4 u32 values that we own.
    unsafe {
        for i in 0..4 {
            ptr.add(i).write(10 * (i as u32 + 1));
        }
        let values = std::slice::from_raw_parts(ptr, 4);
        println!("values {values:?} at {ptr:p}");
        dealloc(ptr as *mut u8, layout);
    }
}
```

```text
size 16 bytes, align 4
values [10, 20, 30, 40] at 0x104d21450
```

**What happens:**

1. `Layout::array::<u32>(4)` describes room for four `u32`: 16 bytes, alignment 4. It returns an error only if the size would be absurdly large. `Layout::new::<T>()` does the same for one value of type `T`.
2. `alloc(layout)` returns a `*mut u8`, a raw pointer to uninitialized memory. A null pointer means the allocator could not provide the memory (OOM, Out Of Memory); `handle_alloc_error` then stops the program, exactly as `Vec` would.
3. `ptr.add(i).write(…)` puts the values into the block. Plain assignment through `*ptr = …` would be fine for `u32` too, but `write` is the correct habit for memory that was never initialized.
4. `dealloc(ptr, layout)` gives the block back. It must get **the same pointer and the same layout** that `alloc` used. The allocator may rely on the layout to find the block's size.

Run it under Miri too: `cargo +nightly miri run`. It prints the same lines (with another address) and no error, because every rule was kept.

> 🔒 **Security:** Use-after-free and double-free bugs in allocators are classic security holes: an attacker who can make a program reuse a freed block can often control what is in it. Safe Rust (`Box`, `Vec`) makes them impossible; in `unsafe` allocator code you must prevent them yourself, and Miri helps you find them.

## 3. `GlobalAlloc`: replacing the allocator

Every `Box`, `Vec` and `String` calls the **global allocator**. You can install your own: write a type that implements the `GlobalAlloc` trait and mark one `static` of that type with `#[global_allocator]`. The trait has two required methods:

```text
unsafe trait GlobalAlloc {
    unsafe fn alloc(&self, layout: Layout) -> *mut u8;
    unsafe fn dealloc(&self, ptr: *mut u8, layout: Layout);
    // with default implementations: alloc_zeroed, realloc
}
```

The trait itself is `unsafe` to implement: you promise to return a block that really has the requested size and alignment and that nobody else uses. The methods are `unsafe` to call: the caller promises to pass a layout with a non-zero size, and to free only blocks from this allocator with their original layout.

The first allocator does no work of its own. It passes every call to `System` and counts the calls. Replace `src/main.rs`:

```rust
use std::alloc::{GlobalAlloc, Layout, System};
use std::sync::atomic::{AtomicUsize, Ordering};

/// Wraps the system allocator and counts what passes through it.
struct Counting;

static ALLOCATIONS: AtomicUsize = AtomicUsize::new(0);
static BYTES: AtomicUsize = AtomicUsize::new(0);

// SAFETY: every call is passed on unchanged to `System`, which is a correct allocator.
unsafe impl GlobalAlloc for Counting {
    unsafe fn alloc(&self, layout: Layout) -> *mut u8 {
        ALLOCATIONS.fetch_add(1, Ordering::Relaxed);
        BYTES.fetch_add(layout.size(), Ordering::Relaxed);
        // SAFETY: the caller of `alloc` upholds the rules of `GlobalAlloc::alloc`.
        unsafe { System.alloc(layout) }
    }

    unsafe fn dealloc(&self, ptr: *mut u8, layout: Layout) {
        // SAFETY: `ptr` was allocated by `System` with this `layout`.
        unsafe { System.dealloc(ptr, layout) }
    }
}

#[global_allocator]
static GLOBAL: Counting = Counting;

/// Runs `work` and prints how many allocations it made.
fn count(name: &str, work: impl FnOnce()) {
    let before_count = ALLOCATIONS.load(Ordering::Relaxed);
    let before_bytes = BYTES.load(Ordering::Relaxed);
    work();
    let count = ALLOCATIONS.load(Ordering::Relaxed) - before_count;
    let bytes = BYTES.load(Ordering::Relaxed) - before_bytes;
    println!("{name:<28} {count:>3} allocations, {bytes:>5} bytes");
}

fn main() {
    count("let x = 5", || {
        let x = 5;
        std::hint::black_box(x);
    });
    count("Box::new(5_u64)", || {
        std::hint::black_box(Box::new(5_u64));
    });
    count("String::from(\"hello\")", || {
        std::hint::black_box(String::from("hello"));
    });
    count("Vec: push 1000 times", || {
        let mut v = Vec::new();
        for i in 0..1000_u32 {
            v.push(i);
        }
        std::hint::black_box(v);
    });
    count("Vec::with_capacity(1000)", || {
        let mut v = Vec::with_capacity(1000);
        for i in 0..1000_u32 {
            v.push(i);
        }
        std::hint::black_box(v);
    });
    count("format!(\"{} {}\", 1, 2)", || {
        std::hint::black_box(format!("{} {}", 1, 2));
    });
}
```

```text
let x = 5                      0 allocations,     0 bytes
Box::new(5_u64)                1 allocations,     8 bytes
String::from("hello")          1 allocations,     5 bytes
Vec: push 1000 times           9 allocations,  8176 bytes
Vec::with_capacity(1000)       1 allocations,  4000 bytes
format!("{} {}", 1, 2)         1 allocations,     3 bytes
```

**What happens:**

1. `Counting` is a zero-sized type; all its state is in two atomic counters, because the global allocator is shared by all threads and `GlobalAlloc` methods only get `&self`.
2. `#[global_allocator] static GLOBAL: Counting = Counting;` installs it. From now on every heap allocation in the program goes through `Counting::alloc`, including the ones inside the standard library.
3. `let x = 5` lives on the stack: no allocation at all.
4. Pushing 1000 numbers into an empty `Vec` causes 9 allocations. The capacity grows 4, 8, 16, …, 1024 (Lesson 07.3). Each growth calls `realloc`; we did not write `realloc`, so the trait's default version is used, which calls our `alloc` for a new block, copies the data and calls `dealloc` for the old one. 4 × (4 + 8 + … + 1024) = 8176 bytes were requested in total.
5. `Vec::with_capacity(1000)` asks once, for exactly 4000 bytes. When you know the size in advance, tell the collection.

> 💡 **Tip:** A counting allocator is a cheap and precise profiling tool. Wrap one around a test or a request handler to see how many allocations it makes; Phase 19 shows full memory profilers.

## 4. A bump allocator

The simplest real allocator is a **bump allocator**. It owns an arena and one number, `next`: the first free address. To allocate, it rounds `next` up to the requested alignment, hands out that address, and moves `next` forward by the size. Freeing one block does nothing; the whole arena becomes free again only when *every* block has been freed.

```text
   arena: addresses 1000 … 1063

   start                                   alloc(1, align 1) → 1000
   [a.......................................]  next = 1001
         alloc(8, align 8): round 1001 up to 1008 → 1008
   [a|pad|bbbbbbbb..........................]  next = 1016
         alloc(4, align 4) → 1016
   [a|pad|bbbbbbbb|cccc.....................]  next = 1020
```

Rounding up is a classic bit trick (Lesson 12.4): if `align` is a power of two, `(addr + align - 1) & !(align - 1)` adds just enough to reach the next multiple and then clears the low bits. To watch the algorithm without unsafe code, this model works on plain numbers instead of real memory. Put it into `src/lib.rs`:

```rust
/// Rounds `addr` up to the next multiple of `align`.
/// `align` must be a power of two.
pub fn align_up(addr: usize, align: usize) -> usize {
    debug_assert!(align.is_power_of_two());
    (addr + align - 1) & !(align - 1)
}

/// A bump allocator that works on plain numbers instead of real memory,
/// so that we can watch the algorithm. `start..end` is the arena.
pub struct BumpModel {
    start: usize,
    next: usize,
    end: usize,
    live: usize,
}

impl BumpModel {
    pub fn new(start: usize, size: usize) -> BumpModel {
        BumpModel {
            start,
            next: start,
            end: start + size,
            live: 0,
        }
    }

    /// Returns the address of a new block, or `None` if the arena is full.
    pub fn alloc(&mut self, size: usize, align: usize) -> Option<usize> {
        let start = align_up(self.next, align);
        let new_next = start.checked_add(size)?;
        if new_next > self.end {
            return None;
        }
        self.next = new_next;
        self.live += 1;
        Some(start)
    }

    /// A bump allocator cannot reuse one block. It only counts, and when
    /// the last live block is freed, it starts again from the beginning.
    pub fn dealloc(&mut self) {
        self.live -= 1;
        if self.live == 0 {
            self.next = self.start;
        }
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn align_up_rounds_to_the_next_multiple() {
        assert_eq!(align_up(0, 8), 0);
        assert_eq!(align_up(1, 8), 8);
        assert_eq!(align_up(8, 8), 8);
        assert_eq!(align_up(13, 4), 16);
        assert_eq!(align_up(13, 1), 13);
    }

    #[test]
    fn blocks_are_aligned_and_do_not_overlap() {
        let mut bump = BumpModel::new(1000, 64);
        let a = bump.alloc(1, 1).unwrap(); // 1000
        let b = bump.alloc(8, 8).unwrap(); // 1008: 1001 rounded up
        let c = bump.alloc(4, 4).unwrap(); // 1016
        assert_eq!((a, b, c), (1000, 1008, 1016));
    }

    #[test]
    fn a_full_arena_returns_none() {
        let mut bump = BumpModel::new(0, 16);
        assert!(bump.alloc(16, 1).is_some());
        assert_eq!(bump.alloc(1, 1), None);
    }

    #[test]
    fn freeing_everything_resets_the_arena() {
        let mut bump = BumpModel::new(0, 16);
        bump.alloc(10, 1).unwrap();
        bump.alloc(6, 1).unwrap();
        bump.dealloc();
        assert_eq!(bump.alloc(1, 1), None, "one block is still live");
        bump.dealloc();
        assert_eq!(bump.alloc(16, 1), Some(0), "all freed: start again");
    }
}
```

```text
running 4 tests
....
test result: ok. 4 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

(That is `cargo test -q --lib`.)

**What happens:**

1. `align_up` example: `addr = 13`, `align = 4`. `13 + 3 = 16` (binary `10000`), and `!(4 - 1)` is a mask with the lowest two bits cleared, so the result is 16. For an `addr` that is already aligned, adding `align - 1` does not reach the next multiple, so it stays.
2. `alloc` uses `checked_add`: a huge `size` must give `None`, not an overflow that wraps around to a small address (Lesson 12.2). An allocator bug like this would hand out memory that belongs to someone else.
3. `live` counts the blocks still in use. Only when it reaches zero can `next` jump back to `start`.

Bump allocation is extremely fast (a few instructions) and has no headers at all. It is perfect when many objects are created together and die together: one request in a web server, one frame in a game, one pass of a compiler. The crate `bumpalo` (version 3.20) offers such an arena for ordinary Rust code. As a *global* allocator it is a poor choice for long-running programs: memory is only reused when everything is freed. In the project (Lesson 12.14) you turn this model into a real `GlobalAlloc` over a real arena.

## 5. Free lists, splitting and coalescing

To reuse single blocks, the allocator must remember which blocks are free. The classic way is a **free list**: every free block stores, inside its own (unused) bytes, its size and a pointer to the next free block. The list costs no extra memory, because it lives in memory that nobody else uses at the moment.

```text
   heap:   [ used 16 ][ FREE 32  ][ used 16 ][ FREE 64          ]
                       │ size 32               │ size 64
   head ──────────────►│ next ─────────────────►│ next ──► (none)
```

**Allocating** with *first fit*: walk the list and take the first block that is big enough (with alignment). If it is much bigger than needed, **split** it: hand out the front part and put the rest back on the list as a smaller free block.

```text
   alloc(16) from the FREE 32 block:
   [ used 16 ][ NEW 16 ][ FREE 16 ][ used 16 ][ FREE 64 ]
```

**Freeing**: put the block back on the list. If its neighbour in memory is also free, **coalesce** them into one bigger block, otherwise the heap slowly falls apart into small pieces. Coalescing is easiest when the list is kept **sorted by address**: then the neighbours of a block are its neighbours in the list.

```text
   free the 16-byte block between two free blocks:
   before: [ FREE 16 ][ used 16 ][ FREE 32 ]
   after:  [ FREE 64                       ]
```

Two kinds of waste remain:

- **Internal fragmentation**: memory wasted inside blocks, for example padding for alignment, or a request for 17 bytes rounded up to 32.
- **External fragmentation**: enough free memory in total, but cut into pieces that are each too small. Exercise 4 shows it with numbers.

There are other search strategies: *best fit* (the smallest block that fits: less waste, but it must look at the whole list) and *next fit* (continue where the last search stopped). And every free-list allocator must decide where to keep a block's size so that `dealloc` can find it. In Rust this is easier than in C: `dealloc` receives the `Layout`, so the size is known without a header.

## 6. How real allocators stay fast

A single free list protected by one lock would be slow: long searches, and all threads waiting for the same lock. The system `malloc`, jemalloc and mimalloc all use similar ideas:

1. **Size classes.** Every small request is rounded up to one of a few dozen sizes (8, 16, 32, 48, 64, …). Each class has its own free list, so "find a block" is just "take the first one": no search, no splitting. The price is some internal fragmentation.
2. **Pages per size class.** Blocks of one class are cut from the same page of memory (for example 16 KiB on your Mac), so the allocator knows a block's class from its address.
3. **Per-thread caches.** Each thread keeps a few free blocks of each class for itself. Most allocations and frees take no lock at all.
4. **Big blocks directly from the operating system.** Requests larger than some limit (often hundreds of KiB) get their own pages via `mmap`, and are returned with `munmap` when freed.

Changing Rust's global allocator to one of them takes three lines and one dependency (`cargo add mimalloc@0.1`):

```rust
use mimalloc::MiMalloc;

#[global_allocator]
static GLOBAL: MiMalloc = MiMalloc;

fn main() {
    let words: Vec<String> = (0..5).map(|i| format!("word{i}")).collect();
    println!("{}", words.join(" "));
}
```

```text
word0 word1 word2 word3 word4
```

Nothing else in the program changes: `Vec`, `String` and `format!` now use mimalloc. (`tikv-jemallocator`, version 0.7, works the same way for jemalloc.) Whether it is faster for *your* program, only a benchmark can say (Lesson 05.12).

> 📏 **Convention:** Choose the global allocator only in the final binary (`main.rs`), never in a library: a library that sets `#[global_allocator]` would force its choice on every program that uses it, and a program may have only one.

## Common mistakes

**1. Freeing with a different layout.** The layout passed to `dealloc` must be the one used for `alloc`. The program may seem to work; Miri does not accept it:

```rust
use std::alloc::{Layout, alloc, dealloc};

fn main() {
    let layout = Layout::array::<u32>(4).expect("size fits in isize");
    // SAFETY: the layout has a non-zero size.
    let ptr = unsafe { alloc(layout) };
    if ptr.is_null() {
        std::alloc::handle_alloc_error(layout);
    }
    let wrong = Layout::array::<u32>(8).expect("size fits in isize");
    // BUG: the layout must be the same one that was used for `alloc`.
    unsafe { dealloc(ptr, wrong) };
    println!("freed");
}
```

`cargo run` prints `freed`. `cargo +nightly miri run` stops:

```text
error: Undefined Behavior: incorrect layout on deallocation: alloc294 has size 16 and alignment 4, but gave size 32 and alignment 4
  --> src/main.rs:12:14
   |
12 |     unsafe { dealloc(ptr, wrong) };
   |              ^^^^^^^^^^^^^^^^^^^ Undefined Behavior occurred here
```

Keep the layout next to the pointer (as `Vec` keeps its capacity) so that you always free with the original one.

**2. Forgetting to free.** A block that is never freed is a **leak**. Rust does not count a leak as UB (Undefined Behaviour), but Miri reports it at the end of the run:

```rust
use std::alloc::{Layout, alloc};

fn main() {
    let layout = Layout::new::<u64>();
    // SAFETY: the layout has a non-zero size.
    let ptr = unsafe { alloc(layout) };
    println!("allocated at {ptr:p}, never freed");
}
```

```text
allocated at 0x812b0, never freed
error: memory leaked: alloc283 (Rust heap, size: 8, align: 8), allocated here:
 --> src/main.rs:6:24
  |
6 |     let ptr = unsafe { alloc(layout) };
  |                        ^^^^^^^^^^^^^
```

**3. Allocating inside the allocator.** Code inside `GlobalAlloc::alloc` must not allocate, because that calls `alloc` again. A tempting debugging line like `println!("alloc {} bytes", layout.size());` inside `alloc` made the test program print nothing and hang forever (it had to be stopped from outside after 10 seconds): `println!` itself needs a heap buffer and the lock on standard output. Count with atomics instead, as in section 3, and print the numbers later.

**4. Forgetting that the global allocator is shared.** `GlobalAlloc` methods take `&self` and may be called from many threads at the same time. Shared state needs atomics or a lock. A `Cell` inside the allocator does not even compile (a `static` must be `Sync`, and `Cell` is not), and changing a `static mut` from several threads is a data race, which is UB.

## Try it yourself

1. Run the example from section 2 with `cargo run` and with `cargo +nightly miri run`.
2. Run the counting allocator from section 3. Add a line that counts `vec![0_u8; 100]` and one that counts `"a".repeat(10)`. Predict the numbers first.
3. In the counting allocator, change `Vec::new()` in the "push 1000 times" case to `Vec::with_capacity(500)`. How many allocations now, and why?
4. Put the bump model from section 4 into `src/lib.rs` and run `cargo test` and `cargo +nightly miri test`.

## Exercises

### Exercise 1 — Round up by hand (easy)

Compute `align_up(17, 8)`, `align_up(32, 16)`, `align_up(5, 1)` and `align_up(1, 4096)` on paper. Then explain in one sentence why the trick only works when `align` is a power of two.

<details>
<summary>Hint</summary>

Write the numbers in binary. `!(align - 1)` for `align = 8` is a mask that ends in `…11111000`.

</details>

<details>
<summary>Answer</summary>

`align_up(17, 8)` = 24 (17 + 7 = 24, already a multiple of 8). `align_up(32, 16)` = 32 (already aligned). `align_up(5, 1)` = 5 (alignment 1 changes nothing). `align_up(1, 4096)` = 4096.

Why only powers of two: for a power of two, `align - 1` is a run of 1 bits (8 − 1 = `0b111`), so `& !(align - 1)` clears exactly the bits below the alignment. For 6, `6 - 1 = 0b101` is not such a run, and the mask gives wrong results; `Layout` refuses alignments that are not powers of two for this reason.

</details>

### Exercise 2 — How much is left? (medium)

Add `pub fn remaining(&self) -> usize` to `BumpModel`: the free bytes at the end of the arena. Write two tests: padding counts as used (in a 32-byte arena, `alloc(1, 1)` and then `alloc(8, 8)` leave 16 bytes), and a zero-sized block takes no space (`alloc(0, 1)` in an 8-byte arena returns the start and leaves 8).

<details>
<summary>Hint</summary>

`remaining` is one subtraction. Work out the padding in the first test by hand: the second block starts at 8, not 1.

</details>

<details>
<summary>Answer</summary>

Add to `impl BumpModel`:

```rust
    /// How many bytes are still free at the end of the arena.
    pub fn remaining(&self) -> usize {
        self.end - self.next
    }
```

Add to `mod tests`:

```rust
    #[test]
    fn remaining_counts_padding_as_used() {
        let mut bump = BumpModel::new(0, 32);
        assert_eq!(bump.remaining(), 32);
        bump.alloc(1, 1).unwrap();
        bump.alloc(8, 8).unwrap(); // 7 bytes of padding before it
        assert_eq!(bump.remaining(), 16);
    }

    #[test]
    fn zero_sized_blocks_take_no_space() {
        let mut bump = BumpModel::new(0, 8);
        assert_eq!(bump.alloc(0, 1), Some(0));
        assert_eq!(bump.remaining(), 8);
    }
```

```text
running 6 tests
......
test result: ok. 6 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

Why it works: after the first block `next` is 1; the second block is aligned to 8, so it occupies 8..16 and `next` becomes 16; 32 − 16 = 16. The 7 padding bytes (1..8) are lost until the arena is reset: internal fragmentation. A real `GlobalAlloc` never sees size 0 (the rules forbid it), but the model should still behave sensibly.

</details>

### Exercise 3 — Iterators and allocations (medium)

With the counting allocator from section 3, count two lines: `let v: Vec<u32> = (0..1000).collect();` and `let v: Vec<u32> = (0..1000).filter(|x| x % 2 == 0).collect();`. The second vector is half as long. Why does it make more allocations?

<details>
<summary>Hint</summary>

`collect` asks the iterator for a size hint (`size_hint`, Lesson 05.5) and reserves that much at once. How exact can a `filter` be about its length?

</details>

<details>
<summary>Answer</summary>

Replace `main` in the counting program with:

```rust
fn main() {
    count("collect 1000 from a range", || {
        let v: Vec<u32> = (0..1000).collect();
        std::hint::black_box(v);
    });
    count("collect 500 after filter", || {
        let v: Vec<u32> = (0..1000).filter(|x| x % 2 == 0).collect();
        std::hint::black_box(v);
    });
}
```

```text
collect 1000 from a range      1 allocations,  4000 bytes
collect 500 after filter       8 allocations,  4080 bytes
```

Why it works: a range knows its exact length, so `collect` allocates 4000 bytes once. A `filter` cannot know how many items will pass; its size hint says "between 0 and 1000", so `collect` starts small and grows: capacities 4, 8, …, 512, eight allocations and 4080 bytes in total. If you know a good estimate, use `Vec::with_capacity(n)` and `extend`.

</details>

### Exercise 4 — A free list on paper (hard)

A free-list allocator manages 64 bytes at offsets 0..64. It uses first fit, splits blocks, keeps the list sorted by address and coalesces neighbours. Ignore headers and alignment. Write down the free list after each step, and say what happens at step 5:

1. `a = alloc(16)` 2. `b = alloc(16)` 3. `c = alloc(16)` 4. `free(b)` 5. `d = alloc(24)` 6. `free(a)` 7. `d = alloc(24)` 8. `free(c)`

<details>
<summary>Hint</summary>

Write free blocks as ranges, like `[16..32)`. At step 5, count the free bytes in total, then look at the size of each free block.

</details>

<details>
<summary>Answer</summary>

| Step | Result | Free list after the step |
|---|---|---|
| start | | `[0..64)` |
| 1. `a = alloc(16)` | a = 0 | `[16..64)` |
| 2. `b = alloc(16)` | b = 16 | `[32..64)` |
| 3. `c = alloc(16)` | c = 32 | `[48..64)` |
| 4. `free(b)` | | `[16..32)`, `[48..64)` |
| 5. `d = alloc(24)` | **fails** | `[16..32)`, `[48..64)` |
| 6. `free(a)` | coalesce with `[16..32)` | `[0..32)`, `[48..64)` |
| 7. `d = alloc(24)` | d = 0, split | `[24..32)`, `[48..64)` |
| 8. `free(c)` | coalesce with both neighbours | `[24..64)` |

At step 5, 32 bytes are free, but in two pieces of 16; no single block holds 24 bytes. That is **external fragmentation**. Coalescing in step 6 repairs it. In step 8, `c` (32..48) touches both free blocks, so all three become one block of 40 bytes. You build exactly this logic in the project.

</details>

## Quiz

1. Why does Rust not ask the operating system for memory on every `Box::new`?

<details>
<summary>Answer</summary>

Every request to the kernel is a system call, which is slow compared to a few instructions in the program. The allocator gets big regions now and then and hands out small blocks from them quickly, reusing freed blocks.

</details>

2. What two numbers does a `Layout` hold, and why does `dealloc` need it?

<details>
<summary>Answer</summary>

The size and the alignment of the block. `dealloc` needs them because the allocator may use them to find the block's size class or the size to give back; the rules require exactly the layout that was used for `alloc`.

</details>

3. How do you replace the global allocator, and why must its state be thread-safe?

<details>
<summary>Answer</summary>

Implement `GlobalAlloc` for a type and declare `#[global_allocator] static NAME: Type = …;`. All threads of the program allocate through this one value at the same time, via `&self`, so its state needs atomics or locks.

</details>

4. What are the strong and the weak sides of a bump allocator?

<details>
<summary>Answer</summary>

Strong: allocation is a few instructions, there are no headers, and freeing a whole arena at once is instant. Weak: single blocks cannot be reused; memory comes back only when every block in the arena is freed, so a long-running program with mixed lifetimes runs out of memory.

</details>

5. What is the difference between internal and external fragmentation?

<details>
<summary>Answer</summary>

Internal: wasted bytes inside handed-out blocks (alignment padding, rounding up to a size class). External: enough free memory in total, but split into pieces that are each too small for the request.

</details>

6. Name two techniques that make allocators like mimalloc fast with many threads.

<details>
<summary>Answer</summary>

Size classes (one free list per block size, so no search and no splitting) and per-thread caches (most allocations and frees take no lock). Large requests go straight to the operating system with `mmap`.

</details>

## Summary

- An **allocator** hands out and takes back heap blocks; Rust's default global allocator is `System` (`malloc`/`free`), which gets big regions from the operating system and cuts them up.
- A `Layout` (size + alignment) describes every block; `std::alloc::alloc` and `dealloc` are the raw, `unsafe` interface, and `dealloc` must get the same layout as `alloc`.
- Implementing `GlobalAlloc` and declaring a `#[global_allocator]` static replaces the allocator for the whole program; its state is shared by all threads.
- A counting allocator shows real numbers: `Vec` growth, `with_capacity`, and iterator size hints change how often you allocate.
- A **bump allocator** moves a pointer forward with `align_up` and frees everything at once; a **free-list allocator** reuses blocks with first fit, splitting and coalescing, and suffers from **fragmentation**.
- Real allocators use **size classes**, **per-thread caches** and `mmap` for big blocks; swapping in mimalloc or jemalloc takes three lines in `main.rs`.
- Miri catches wrong layouts, double frees, use-after-free and leaks: run it on every allocator you write.

## Sources and further reading

- [std::alloc](https://doc.rust-lang.org/std/alloc/index.html) — the allocation module: `alloc`, `dealloc`, `System`, `#[global_allocator]` with an example.
- [std::alloc::GlobalAlloc](https://doc.rust-lang.org/std/alloc/trait.GlobalAlloc.html) — every rule an implementation and its callers must follow.
- [std::alloc::Layout](https://doc.rust-lang.org/std/alloc/struct.Layout.html) — constructing and combining layouts.
- [Philipp Oppermann — Allocator Designs](https://os.phil-opp.com/allocator-designs/) — bump, linked-list and fixed-size-block allocators in Rust, explained step by step (written for a kernel, but the designs are the same).
- [Computer Systems: A Programmer's Perspective (CS:APP)](https://csapp.cs.cmu.edu/) — section 9.9, dynamic memory allocation, with free lists and coalescing.
- [mimalloc on docs.rs](https://docs.rs/mimalloc) and [bumpalo on docs.rs](https://docs.rs/bumpalo) — a general-purpose allocator and an arena for Rust programs.
- [Miri on GitHub](https://github.com/rust-lang/miri) — what Miri checks and how to configure it.

---

[🏠 Start](../START-HERE.md) · [← 12.12 Linking and loading programs](12-linking-and-loading.md) · [📋 Phase 12](README.md) · [12.14 Project: your own memory allocator →](14-project-your-own-allocator.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
