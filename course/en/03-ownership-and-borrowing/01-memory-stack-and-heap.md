<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← Phase 03 — Ownership and borrowing](README.md) · [📋 Phase 03](README.md) · [03.2 The ownership rules →](02-ownership-rules.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/03-ownership-and-borrowing/01-memory-stack-and-heap.md) <!-- nav -->

# Lesson 03.1 — Memory: the stack and the heap

> **Phase 03 — Ownership and borrowing** · Lesson 1 of 8 · ⏱ about 2.5 hours (2 sittings)

## What you will learn

- How a program sees memory: bytes, addresses and pointers.
- What the **stack** is, why it is fast, and what happens when it runs out.
- What the **heap** is, why programs need it, and what it costs.
- How `String` and `Vec` are built from a small part on the stack and a large part on the heap.
- The classic memory bugs (leak, dangling pointer, double free) that Rust's ownership rules were invented to prevent.

## Before you start

- You finished [Lesson 02.11 — Checkpoint](../02-programming-basics/11-checkpoint.md) and you can write and run small Rust programs with functions.
- You remember from [Lesson 00.3 — How a computer runs a program](../00-getting-started/03-how-a-computer-runs-a-program.md) that a program is loaded into memory and run by the CPU.
- Create a project for this lesson. In a terminal, from `~/projects/rust`:

```bash
cargo new code/p03-memory
```

## New words

| Word | Meaning |
|---|---|
| **memory** | The place where a running program keeps its data. Think of a very long row of numbered boxes. |
| **RAM** (Random Access Memory) | The fast, temporary memory chips in your computer. Everything a running program uses lives here. It is emptied when the program ends or the computer is switched off. |
| **CPU** (Central Processing Unit) | The chip that runs program instructions and reads and writes memory. |
| **byte** | One memory box. It holds a number from 0 to 255. Every other value (text, big numbers) is built from several bytes. |
| **address** | The number of a memory box, like a house number on a street. |
| **pointer** | A value that contains an address. It "points" to where some other data lives. |
| **stack** | A region of memory for local variables. It works like a pile of plates: last in, first out. |
| **LIFO** (Last In, First Out) | The rule of a pile: the last item put on is the first one taken off. |
| **stack frame** | The block of the stack that belongs to one function call. It holds that call's local variables. |
| **heap** | A large region of memory for data whose size is not known in advance or that must live longer than one function call. |
| **allocate** / **free** | To allocate is to ask for a piece of heap memory. To free (also *deallocate*) is to give it back. |
| **allocator** | The part of the system that hands out and takes back pieces of heap memory. |
| **capacity** | How many bytes (or items) a growable value such as `String` or `Vec` can hold before it must ask for a bigger piece of heap. |
| **stack overflow** | A crash that happens when the stack runs out of room, usually because of a function that calls itself forever. |
| **memory leak** | Heap memory that was allocated but never freed, so the program keeps using more and more memory. |
| **dangling pointer** | A pointer to memory that has already been freed. Using it is a serious bug. |
| **cache** | A small, very fast memory inside the CPU that keeps recently used data close to the chip. |
| **OS** (Operating System) | The program (macOS on your Mac) that manages the computer and gives memory to other programs. |

## 1. Memory is a long row of numbered boxes

Your Mac has gigabytes of RAM. You can picture it as one extremely long street. Every house on the street is one **byte**, and every house has a number: its **address**.

```text
address:   ...  1000   1001   1002   1003   1004   1005  ...
content:   ...  [ 42 ] [ 7  ] [ 0  ] [ 255] [ 18 ] [ 9  ] ...
```

When your program says `let number = 42;`, Rust picks some boxes, writes the number into them, and remembers the address. A value that needs more than 255 uses several neighbouring bytes. That is why different types have different **sizes**. You can ask Rust for the size of a type with `size_of`:

```rust
fn main() {
    println!("i32:        {} bytes", size_of::<i32>());
    println!("i64:        {} bytes", size_of::<i64>());
    println!("f64:        {} bytes", size_of::<f64>());
    println!("bool:       {} byte", size_of::<bool>());
    println!("char:       {} bytes", size_of::<char>());
    println!("(i32, i32): {} bytes", size_of::<(i32, i32)>());
    println!("[i32; 4]:   {} bytes", size_of::<[i32; 4]>());
    println!("String:     {} bytes", size_of::<String>());
    println!("&str:       {} bytes", size_of::<&str>());
    println!("&i32:       {} bytes", size_of::<&i32>());
}
```

**What happens:**

1. `size_of::<i32>()` is a function that answers "how many bytes does one value of this type take?". The part `::<i32>` tells the function which type to measure. (This syntax comes from *generics*, which you will learn in Lesson 05.1. For now, just copy it.)
2. Each `println!` prints the answer for one type.
3. The size is decided by the type alone. It is the same for every value of that type.

```text
i32:        4 bytes
i64:        8 bytes
f64:        8 bytes
bool:       1 byte
char:       4 bytes
(i32, i32): 8 bytes
[i32; 4]:   16 bytes
String:     24 bytes
&str:       16 bytes
&i32:       8 bytes
```

Notice three things:

- `i32` always takes 4 bytes. A tuple of two `i32` takes 8, and an array of four takes 16. Sizes simply add up.
- A `String` takes **24 bytes** — whether it holds "hi" or the whole text of a novel. A `String` clearly cannot keep its letters inside those 24 bytes. Where are the letters? That is the main question of this lesson.
- A reference such as `&i32` takes **8 bytes**. It is just an address. (Your Mac is a *64-bit* machine, so an address is 64 bits = 8 bytes.)

> 💡 **Tip:** `size_of` is in Rust's *prelude* (the small set of names that every program can use without a `use` line). Whenever you wonder "how big is this type?", print it.

## 2. The stack: fast, tidy and limited

The **stack** is the part of memory where your local variables live. It works like a pile of plates in a canteen. You can only put a plate on the top, and you can only take a plate from the top. This rule is called **LIFO** (Last In, First Out).

Every time a function is called, the program puts a new block on top of the pile. This block is the function's **stack frame**. It holds the function's parameters and local variables. When the function returns, its frame is removed, and everything in it disappears at once.

```rust
fn main() {
    let a = 1;
    println!("main:   a is at {:p}", &a);
    first();
}

fn first() {
    let b = 2;
    println!("first:  b is at {:p}", &b);
    second();
}

fn second() {
    let c = 3;
    println!("second: c is at {:p}", &c);
}
```

**What happens:**

1. `main` starts. Its frame is put on the stack. The variable `a` lives inside it.
2. `&a` means "the address of `a`". It is a *reference*; you will learn about references in Lesson 03.4. The format `{:p}` prints an address (`p` is for *pointer*).
3. `main` calls `first`. A new frame for `first` is put on top, with `b` in it. Then `first` calls `second`, and a third frame holds `c`.
4. When `second` ends, its frame is removed. Then `first`'s frame, then `main`'s.

```text
main:   a is at 0x16bc51e64
first:  b is at 0x16bc51e14
second: c is at 0x16bc51dc4
```

Your numbers will be different. The operating system chooses where the stack starts, and the start changes from run to run. But the pattern is the same: **each deeper call has a smaller address**. On your Mac the stack grows toward smaller addresses.

```text
 high addresses
 ┌──────────────────┐
 │ frame of main    │  a = 1
 ├──────────────────┤
 │ frame of first   │  b = 2
 ├──────────────────┤
 │ frame of second  │  c = 3      ← the top of the stack (newest)
 └──────────────────┘
 low addresses
```

Why is the stack fast? Because "allocating" a frame is only moving one number (the *stack pointer*) up or down. There is nothing to search for and nothing to clean. The top of the stack is also used all the time, so the CPU keeps it in its small, very fast internal memory (called the *cache*).

The stack has two rules and one limit:

- **Rule 1:** the size of every value must be known when the program is compiled. Rust must know how big a frame is before the function runs.
- **Rule 2:** a frame lives exactly as long as its function call. You cannot keep a stack value after the function returns.
- **Limit:** the stack is small. The main thread on your Mac has about 8 MiB (mebibytes, roughly 8 million bytes).

What happens when you use more than that? Look at a function that calls itself with no end:

```rust
fn count_down_forever(n: u64) -> u64 {
    count_down_forever(n + 1) + 1
}

fn main() {
    println!("starting");
    let result = count_down_forever(0);
    println!("result = {}", result);
}
```

Every call adds a frame, and no call ever returns. The compiler even warns you ("function cannot return without recursing"). Run it:

```text
starting

thread 'main' (3866480) has overflowed its stack
fatal runtime error: stack overflow, aborting
```

This is a **stack overflow**. The program is stopped by the operating system, and your last `println!` never runs. The program exit code is 134 (a signal, not a normal exit).

> ⚠️ **Common mistake:** deep *recursion* (a function calling itself) is the usual cause of a stack overflow. A loop does not add frames, so a loop of ten million steps is fine, but ten million nested calls are not. You will practise this in Exercise 5.

## 3. The heap: flexible, but with a price

Some data does not fit the stack rules:

- Its size is **not known** when you write the program. A line typed by the user can have 3 letters or 3000.
- It must **grow or shrink** while the program runs.
- It must **live longer** than the function that created it.

For this, programs use the **heap**. The heap is a big pool of memory managed by an **allocator**. The conversation goes like this:

1. Your program says: "I need 100 bytes."
2. The allocator finds 100 free bytes somewhere in the heap, marks them as used, and answers with their **address**.
3. Your program keeps that address in a **pointer** and uses the memory.
4. When the program has finished with it, it must say: "I do not need these 100 bytes any more." The allocator marks them free again, so they can be reused.

Asking is called to **allocate**. Giving back is called to **free**. Between these two moments the memory belongs to your program.

Now you can answer the question from section 1. A `String` is **three numbers on the stack** — a pointer to the letters, the *length* (how many bytes are used), and the *capacity* (how many bytes were reserved) — and **the letters themselves on the heap**. 3 numbers × 8 bytes = 24 bytes. That is exactly what `size_of::<String>()` printed.

```text
  STACK                              HEAP
 ┌─────────────────────┐            ┌───┬───┬───┬───┬───┐
 │ s: String           │            │ h │ e │ l │ l │ o │
 │   ptr ──────────────┼──────────► └───┴───┴───┴───┴───┘
 │   len = 5           │             5 bytes reserved, 5 used
 │   cap = 5           │
 └─────────────────────┘
```

Let us look at these parts with real code:

```rust
fn main() {
    let number = 42;
    let text = String::from("hello");
    println!("number lives at {:p}", &number);
    println!("text (the three-part value) lives at {:p}", &text);
    println!("the letters of text live at {:p}", text.as_ptr());
}
```

**What happens:**

1. `number` is a plain `i32`, so it lives completely on the stack.
2. `String::from("hello")` asks the allocator for room and copies the letters there. The three-part value `text` is on the stack.
3. `&text` is the address of that three-part value. `text.as_ptr()` is the pointer **stored inside** it, which is the address of the letters.

```text
number lives at 0x16dab1dac
text (the three-part value) lives at 0x16dab1db0
the letters of text live at 0x102a912d0
```

The first two addresses are close to each other (they are both in the same stack frame). The third one is very different. It is in another region of memory: the heap. The exact numbers change every time you run the program, but the *difference* between the stack and the heap addresses stays.

### How a growing String uses the heap

A `String` can grow. When there is no room left, it asks the allocator for a bigger piece, copies the letters, and frees the old piece. Let us watch the capacity:

```rust
fn main() {
    let mut s = String::with_capacity(5);
    println!("start:  len = {}, capacity = {}, letters at {:p}", s.len(), s.capacity(), s.as_ptr());

    s.push_str("hello");
    println!("hello:  len = {}, capacity = {}, letters at {:p}", s.len(), s.capacity(), s.as_ptr());

    s.push('!');
    println!("hello!: len = {}, capacity = {}, letters at {:p}", s.len(), s.capacity(), s.as_ptr());

    let empty = String::new();
    println!("new():  len = {}, capacity = {}", empty.len(), empty.capacity());
}
```

**What happens:**

1. `String::with_capacity(5)` creates an empty string but already reserves 5 bytes on the heap. So `len` is 0 and `capacity` is 5.
2. `push_str("hello")` adds 5 bytes. They fit exactly. Capacity does not change.
3. `push('!')` needs a 6th byte. There is no room, so the `String` asks for more. It does not ask for just one more byte: it doubles the capacity (5 → 10), so that the next pushes are cheap.
4. `String::new()` is empty and has capacity 0: it did not touch the heap at all.

```text
start:  len = 0, capacity = 5, letters at 0x1025352d0
hello:  len = 5, capacity = 5, letters at 0x1025352d0
hello!: len = 6, capacity = 10, letters at 0x1025352d0
new():  len = 0, capacity = 0
```

(The address stayed the same here, which means the allocator was probably able to enlarge the piece in place. On other runs or machines it may move; when it moves, the old piece is freed. Rust never lets you keep using the old address by mistake — you will see why in Lesson 03.5.)

A `Vec<T>` (a growable list of items of type `T`; you will study it in Lesson 04.7) works in the same way. Watch its capacity as we push numbers:

```rust
fn main() {
    let mut numbers = Vec::new();
    println!("start: len = {}, capacity = {}", numbers.len(), numbers.capacity());
    for i in 1..=9 {
        numbers.push(i);
        println!("push {}: len = {}, capacity = {}", i, numbers.len(), numbers.capacity());
    }
}
```

```text
start: len = 0, capacity = 0
push 1: len = 1, capacity = 4
push 2: len = 2, capacity = 4
push 3: len = 3, capacity = 4
push 4: len = 4, capacity = 4
push 5: len = 5, capacity = 8
push 6: len = 6, capacity = 8
push 7: len = 7, capacity = 8
push 8: len = 8, capacity = 8
push 9: len = 9, capacity = 16
```

The pattern is 0 → 4 → 8 → 16. Each time the room is full, the capacity doubles, and the items are copied to the new piece of heap.

### Box: one value on the heap

Rust also lets you put a single value on the heap with `Box::new`. A `Box` is just a pointer to a heap value (you will learn it fully in Lesson 05.9):

```rust
fn main() {
    let on_stack = 42;
    let on_heap = Box::new(42);
    println!("on_stack = {}, on_heap = {}", on_stack, on_heap);
    println!("the Box itself is {} bytes", size_of::<Box<i32>>());
    println!("on_stack is at {:p}", &on_stack);
    println!("the heap number is at {:p}", on_heap);
}
```

```text
on_stack = 42, on_heap = 42
the Box itself is 8 bytes
on_stack is at 0x16b04dd84
the heap number is at 0x1056592d0
```

The `Box` itself (8 bytes: an address) sits on the stack. The `42` it points to sits on the heap. You use `on_heap` almost like a normal number.

## 4. What lives where

| Value | Stack part | Heap part |
|---|---|---|
| `let n: i32 = 5` | the 4 bytes of `n` | — |
| `let x: f64`, `bool`, `char` | the value itself | — |
| `let t = (1, true)` | the whole tuple | — |
| `let a = [0u8; 16]` | all 16 bytes | — |
| `let s = String::from("hi")` | pointer, length, capacity (24 bytes) | the letters |
| `let v = vec![1, 2, 3]` | pointer, length, capacity (24 bytes) | the three numbers |
| `let b = Box::new(7)` | the pointer (8 bytes) | the number `7` |
| `let r = "hello"` (a `&str`) | pointer and length (16 bytes) | — (the letters are stored inside the program file itself) |

The rule of thumb: **fixed-size values live on the stack; values that can grow or whose size is unknown live (at least partly) on the heap, and the stack holds a small "handle" that points to them.**

> ⚡ **Performance:** the stack is faster than the heap because there is nothing to search for and the memory is already warm in the CPU cache. An allocation costs time, and following a pointer to the heap can cost another trip to memory. Do not conclude "never use the heap" — most real programs need it. The lesson is: do not allocate more than necessary, and **measure before you optimize** (Phase 09 teaches how).

## 5. Who frees the heap? Three kinds of bugs

Heap memory must be given back exactly once. This is the hardest part of managing memory by hand. There are three classic mistakes:

```text
1. Memory leak       — you forget to free.
   The memory stays reserved for nothing. A long-running server grows
   bigger and bigger until the operating system kills it.

2. Dangling pointer  — you free, but keep using the pointer.
   It is like a hotel key card for a room that was given to another guest.
   You read or write data that now belongs to someone else.
   (This is also called "use after free".)

3. Double free       — you free the same memory twice.
   The allocator's bookkeeping is corrupted. The program may crash,
   or worse, an attacker can use it.
```

Languages solve this problem in different ways. In some, the programmer must call "free" by hand and must be perfect. In others, a runtime part called a *garbage collector* searches for unused memory from time to time, which costs speed and makes timing less predictable. Rust uses a third way:

> The compiler decides, while compiling, exactly where each piece of heap memory must be freed, using a set of rules called **ownership**. The checks happen before the program runs, and there is no garbage collector at run time.

If the rules are broken, the program does not compile. If they are kept, leaks of the usual kind, dangling pointers and double frees cannot happen in safe Rust code. You will learn the rules in the next lessons, starting with [Lesson 03.2 — The ownership rules](02-ownership-rules.md).

> 🔒 **Security:** use-after-free and double-free bugs are among the most common causes of serious security holes in programs written in languages that leave freeing memory to the programmer. A large part of the value of Rust for a systems engineer is that these bug classes are rejected by the compiler.

> 📏 **Convention:** Rust code rarely needs you to write "allocate" or "free" yourself. You create `String`, `Vec` and `Box` values, and they free their own heap memory at the right moment. This pattern is called **RAII** (Resource Acquisition Is Initialization): a value that owns a resource releases it automatically when it goes away. Lesson 03.2 shows it in action.

## Common mistakes

**1. Thinking a `String` keeps its letters on the stack.** `size_of::<String>()` is always 24 bytes. The letters are on the heap. This matters in the next lessons: copying a `String` means something different from copying an `i32`.

**2. Trying to print an address of a plain value.** The format `{:p}` needs a pointer or a reference, not the value itself:

```rust
fn main() {
    let x = 5;
    println!("{:p}", x);
}
```

```text
error[E0277]: the trait bound `{integer}: std::fmt::Pointer` is not satisfied
 --> src/main.rs:3:22
  |
3 |     println!("{:p}", x);
  |               ----   ^ the trait `std::fmt::Pointer` is not implemented for `{integer}`
  |               |
  |               required by this formatting parameter
```

It means: "an integer is not a pointer, so `{:p}` cannot print it". Fix: print the address with `&x`: `println!("{:p}", &x);`.

**3. Writing endless or very deep recursion.** The result is a stack overflow, as in section 2. Prefer a loop when the number of steps can be large.

**4. Believing the heap is "slow, so avoid it".** The heap is a normal, necessary tool. Avoid *needless* allocations in hot code, but use `String` and `Vec` freely when you need them.

## Try it yourself

1. Open the terminal in RustRover (or macOS Terminal), go to your course folder and create the project (skip `cargo new` if you already created the project in "Before you start"; running it twice only prints `error: destination ... already exists`):

   ```bash
   cd ~/projects/rust
   cargo new code/p03-memory
   ```

   Because `~/projects/rust` is already a Git repository (from Phase 01), Cargo does **not** create a second repository inside the new project. It only creates `Cargo.toml` and `src/main.rs`.

2. Open `code/p03-memory/src/main.rs` and replace its content with the `size_of` program from section 1.
3. Run it:

   ```bash
   cd code/p03-memory
   cargo run
   ```

   You should see the ten lines from section 1. Check that `String` is 24 bytes.
4. Now replace the program with the "where" program from section 3 (`number`, `text`, `text.as_ptr()`). Run it two times. Are the addresses the same in both runs? (Probably not. Compare the *distance* between the stack addresses and the heap address.)
5. Make a Git commit so that your work is saved. First look at what Git sees:

   ```bash
   git status
   ```

   If `target/` shows up in the list, your repository does not ignore Cargo's build folder yet. Add a line `target/` to the `.gitignore` file in `~/projects/rust` (see [Lesson 01.3](../01-git-and-version-control/03-your-first-repository-and-commits.md)). Then:

   ```bash
   git add code/p03-memory
   git commit -m "p03: memory experiments (stack, heap, sizes)"
   ```

> 💡 **Tip:** build the habit now: every small experiment gets its own commit. In this phase you will often change code to see the compiler complain; with a commit behind you, you can always come back (Lesson 07 of this phase returns to this idea).

## Exercises

### Exercise 1 — Stack or heap? (easy)

For each declaration say what lives on the stack and what lives on the heap:

1. `let a: i32 = 7;`
2. `let b = String::from("rust");`
3. `let c = [0u8; 16];`
4. `let d = Box::new(7);`
5. `let e = vec![1, 2, 3];`
6. `let f = (1, true);`
7. `let g = "hello";`

<details>
<summary>Hint</summary>

Ask two questions: "Is the size fixed and known at compile time?" and "Can the value grow?". A growable value keeps a small handle on the stack and its data on the heap.

</details>

<details>
<summary>Answer</summary>

1. `a`: stack only (4 bytes).
2. `b`: three numbers (pointer, length, capacity) on the stack; the letters `rust` on the heap.
3. `c`: stack only (16 bytes). The array has a fixed size.
4. `d`: the pointer (8 bytes) on the stack; the number `7` on the heap.
5. `e`: pointer, length, capacity on the stack; the three numbers on the heap.
6. `f`: stack only (a tuple of two small values).
7. `g`: a `&str` is a pointer plus a length (16 bytes) on the stack. The letters are not on the heap: they are part of the program file and are only read.

Why it works: fixed-size, non-growing values fit directly in a stack frame. Everything that can grow needs the heap.

</details>

### Exercise 2 — Predict the sizes (easy)

Predict the output of this program first on paper, then run it.

```rust
fn main() {
    println!("[u8; 100]: {}", size_of::<[u8; 100]>());
    println!("(u8, u32): {}", size_of::<(u8, u32)>());
    println!("[char; 3]: {}", size_of::<[char; 3]>());
    println!("&[u8]:     {}", size_of::<&[u8]>());
    println!("Vec<i32>:  {}", size_of::<Vec<i32>>());
}
```

<details>
<summary>Hint</summary>

`u8` is 1 byte, `u32` and `char` are 4 bytes. A reference to a slice (`&[u8]`) is a pointer plus a length, like `&str`. A `Vec` is a handle like `String`. The tuple `(u8, u32)` is the surprise: the CPU likes numbers to start at an address that is a multiple of their size, so Rust adds empty padding bytes.

</details>

<details>
<summary>Answer</summary>

```text
[u8; 100]: 100
(u8, u32): 8
[char; 3]: 12
&[u8]:     16
Vec<i32>:  24
```

Why it works: 100 × 1 = 100; `(u8, u32)` is 1 byte + 3 bytes of padding + 4 bytes = 8, not 5; 3 × 4 = 12; a slice reference is pointer (8) + length (8) = 16; a `Vec` is pointer + length + capacity = 24.

</details>

### Exercise 3 — Test the sizes (medium)

Write three tests (use `#[test]`, see [Lesson 02.8](../02-programming-basics/08-your-first-tests.md)) that check: a `String` is three machine words, a `&str` is two machine words, and `[i32; 4]` is four times `i32`. A "machine word" here is `size_of::<usize>()`. Run `cargo test`.

<details>
<summary>Hint</summary>

Put the tests in a `#[cfg(test)] mod tests { ... }` block. Use `assert_eq!(size_of::<String>(), 3 * size_of::<usize>());`. No `use` line is needed for `size_of`.

</details>

<details>
<summary>Answer</summary>

```rust
fn main() {
    println!("run `cargo test`");
}

#[cfg(test)]
mod tests {
    #[test]
    fn string_is_three_machine_words() {
        assert_eq!(size_of::<String>(), 3 * size_of::<usize>());
    }

    #[test]
    fn str_reference_is_pointer_plus_length() {
        assert_eq!(size_of::<&str>(), 2 * size_of::<usize>());
    }

    #[test]
    fn array_size_is_count_times_item_size() {
        assert_eq!(size_of::<[i32; 4]>(), 4 * size_of::<i32>());
    }
}
```

```text
running 3 tests
...
test result: ok. 3 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

Why it works: the test says the same thing as section 1, but now the computer re-checks it every time you run `cargo test`.

</details>

### Exercise 4 — Watch the capacity grow (medium)

Write a program that starts with `String::new()`, pushes the character `'x'` one hundred times, and prints a line **only** when the capacity changes.

<details>
<summary>Hint</summary>

Keep a `let mut last_capacity = s.capacity();`. After each `push`, compare `s.capacity()` with `last_capacity`. If they differ, print and update `last_capacity`.

</details>

<details>
<summary>Answer</summary>

```rust
fn main() {
    let mut s = String::new();
    let mut last_capacity = s.capacity();
    println!("len = 0, capacity = {last_capacity}");
    for i in 1..=100 {
        s.push('x');
        if s.capacity() != last_capacity {
            last_capacity = s.capacity();
            println!("len = {i}, capacity grew to {last_capacity}");
        }
    }
}
```

```text
len = 0, capacity = 0
len = 1, capacity grew to 8
len = 9, capacity grew to 16
len = 17, capacity grew to 32
len = 33, capacity grew to 64
len = 65, capacity grew to 128
```

Why it works: the first push allocates 8 bytes, and then the capacity doubles whenever it is full. The exact numbers are an implementation detail of the standard library, so never write code that depends on them.

</details>

### Exercise 5 — Recursion against a loop (hard)

Write two functions that compute `1 + 2 + … + n`: `sum_to_recursive(n: u64) -> u64` (the function calls itself with `n - 1`) and `sum_to_loop(n: u64) -> u64` (a `for` loop). Write a test that checks both give 55 for `n = 10`. Then call both with `n = 10_000_000` in `main` and watch what happens.

<details>
<summary>Hint</summary>

The recursive version: if `n == 0` return `0`, otherwise return `n + sum_to_recursive(n - 1)`. Ten million nested calls need ten million stack frames.

</details>

<details>
<summary>Answer</summary>

```rust
fn sum_to_recursive(n: u64) -> u64 {
    if n == 0 { 0 } else { n + sum_to_recursive(n - 1) }
}

fn sum_to_loop(n: u64) -> u64 {
    let mut total = 0;
    for i in 1..=n {
        total += i;
    }
    total
}

fn main() {
    println!("recursive 100 = {}", sum_to_recursive(100));
    println!("loop 10_000_000 = {}", sum_to_loop(10_000_000));
    println!("recursive 10_000_000 = {}", sum_to_recursive(10_000_000));
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn both_versions_agree() {
        assert_eq!(sum_to_recursive(10), 55);
        assert_eq!(sum_to_loop(10), 55);
    }
}
```

Output of `cargo run`:

```text
recursive 100 = 5050
loop 10_000_000 = 50000005000000

thread 'main' (3895331) has overflowed its stack
fatal runtime error: stack overflow, aborting
```

Why it works: 100 nested calls fit easily in the 8 MiB stack, and the loop uses one frame for any `n`. Ten million frames do not fit, so the last line crashes the program. The test passes because it uses only `n = 10`.

</details>

## Quiz

1. What is an address, and what is a pointer?

<details>
<summary>Answer</summary>

An address is the number of one byte in memory. A pointer is a value that stores an address, so it "points to" the data that lives there.

</details>

2. Why is the stack fast?

<details>
<summary>Answer</summary>

Putting a frame on the stack or removing it only moves one number (the stack pointer). There is no search for free space and no cleanup of individual values.

</details>

3. A `String` is 24 bytes on the stack. What are those 24 bytes?

<details>
<summary>Answer</summary>

Three 8-byte numbers: a pointer to the letters on the heap, the length (bytes in use), and the capacity (bytes reserved).

</details>

4. Name two reasons to use the heap instead of the stack.

<details>
<summary>Answer</summary>

The size is not known when the program is written, or the data must grow or shrink at run time, or the data must live longer than the function that created it. (Any two of these.)

</details>

5. What is a dangling pointer?

<details>
<summary>Answer</summary>

A pointer to memory that has already been freed. Reading or writing through it touches memory that may now belong to something else.

</details>

6. What causes a stack overflow, and how can you avoid it?

<details>
<summary>Answer</summary>

Using more stack than is available, most often by a function that calls itself too many times. Avoid it by using a loop instead of deep recursion, or by keeping the depth small.

</details>

7. How does Rust avoid leaks, dangling pointers and double frees without a garbage collector?

<details>
<summary>Answer</summary>

The compiler applies the ownership rules to decide exactly where each value is freed, and it rejects programs that break the rules. The rules are covered in Lesson 03.2.

</details>

## Summary

- Memory is a row of numbered bytes; an address is a byte's number; a pointer holds an address.
- The stack holds local variables in frames that appear and disappear with function calls (LIFO). It is fast but small, and values on it must have a size known at compile time.
- The heap is a flexible pool of memory handed out by an allocator. Data on the heap needs a pointer, must be freed exactly once, and costs more than stack data.
- A `String` or `Vec` is a small stack handle (pointer, length, capacity) plus data on the heap. `Box::new(x)` puts one value on the heap.
- Growing a `String` or `Vec` past its capacity allocates a bigger piece, copies the data and frees the old piece.
- The classic bugs — leak, dangling pointer, double free — come from freeing memory at the wrong time. Rust's ownership rules, checked at compile time, prevent them.

## Sources and further reading

- [The Rust Programming Language — 4.1 What Is Ownership?](https://doc.rust-lang.org/book/ch04-01-what-is-ownership.html) — the official explanation of stack, heap and ownership (the part "The Stack and the Heap").
- [Rust By Example — RAII](https://doc.rust-lang.org/rust-by-example/scope/raii.html) — how values free their resources when they go out of scope.
- [std::mem::size_of](https://doc.rust-lang.org/std/mem/fn.size_of.html) — the function used in this lesson, with notes about padding.
- [std::string::String](https://doc.rust-lang.org/std/string/struct.String.html) — the reference page of `String`, including `capacity` and `with_capacity`.
- [The Rustonomicon — Ownership and Lifetimes](https://doc.rust-lang.org/nomicon/ownership.html) — a deeper look for later; read it after Phase 06.

---

[🏠 Start](../START-HERE.md) · [← Phase 03 — Ownership and borrowing](README.md) · [📋 Phase 03](README.md) · [03.2 The ownership rules →](02-ownership-rules.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
