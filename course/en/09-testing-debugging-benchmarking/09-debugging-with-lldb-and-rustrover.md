<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 09.8 Fuzzing](08-fuzzing.md) · [📋 Phase 09](README.md) · [09.10 Logging and tracing →](10-logging-and-tracing.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/09-testing-debugging-benchmarking/09-debugging-with-lldb-and-rustrover.md) <!-- nav -->

# Lesson 09.9 — Debugging in depth: LLDB and RustRover

> **Phase 09 — Testing, debugging and benchmarking** · Lesson 9 of 14 · ⏱ about 135 minutes

## What you will learn

- How a debugger works, and the small set of LLDB commands that covers 95% of real debugging: breakpoints, `bt`, `frame variable`, stepping and `continue`.
- How to use a **conditional breakpoint** and a **watchpoint** to find out *who* changed a value.
- How to debug a failing test, and how to stop a program exactly where it panics.
- How to read a backtrace with `RUST_BACKTRACE=1`.
- The same steps in RustRover, in words, and what a debugger cannot do (release builds).

## Before you start

- You finished [Lesson 09.8 — Fuzzing](08-fuzzing.md).
- You did a first debugger session in [Lesson 02.9 — Debugging basics](../02-programming-basics/09-debugging-basics.md): a breakpoint, stepping and watching variables, in RustRover and in LLDB. This lesson goes deeper and adds watchpoints, tests and panics.
- You know structs and methods ([Lesson 04.2](../04-structs-enums-collections-errors/02-methods-and-associated-functions.md)) and integer overflow ([Lesson 02.2](../02-programming-basics/02-data-types.md)).
- Your code goes in `~/projects/rust/code/p09-debugging`.

## New words

| Word | Meaning |
|---|---|
| **debugger** | A program that runs another program under its control, so you can pause it, look inside, and run it step by step. |
| **LLDB** | The debugger of the LLVM project. It is the debugger that RustRover and macOS use. |
| **`rust-lldb`** | A small wrapper script that starts LLDB and loads helper files, so Rust types such as `Vec` and `String` are shown in a readable way. It comes with Rust. |
| **debug information** | Extra data inside the compiled program that maps machine code back to your file names, lines and variable names. Debug builds have it. |
| **breakpoint** | A marker on a line. When the program reaches it, it stops before running the line. |
| **conditional breakpoint** | A breakpoint that stops only when an expression is true, for example `qty > 1`. |
| **watchpoint** | A marker on a *memory location*. The program stops when somebody writes to it. |
| **stack frame** | The part of the call stack that belongs to one running function: its arguments and local variables. |
| **backtrace** (`bt`) | The list of stack frames, from the function that is running now to `main`. It answers "how did we get here?". |
| **step over / step into / step out** | Run the current line completely / enter the function it calls / run until the current function returns. |
| **REPL** (Read-Eval-Print Loop) | A prompt where you type a command, the program evaluates it and prints the result. The `(lldb)` prompt works this way. |

## 1. The bug we will hunt

A debugger is for the moment when you do not know *why* the program does the wrong thing. We need a program with a real bug. Create the project:

```bash
cd ~/projects/rust/code
cargo new p09-debugging
cd p09-debugging
mkdir -p src/bin
```

Put this in `src/lib.rs`. It is a shopping cart that keeps a running total in cents:

```rust
/// One line of a shopping cart.
#[derive(Debug)]
pub struct Item {
    pub name: String,
    pub price_cents: u64,
    pub qty: u32,
}

#[derive(Debug, Default)]
pub struct Cart {
    pub items: Vec<Item>,
    pub total_cents: u64,
}

impl Cart {
    pub fn new() -> Cart {
        Cart::default()
    }

    pub fn add(&mut self, name: &str, price_cents: u64, qty: u32) {
        let line_total = price_cents * u64::from(qty);
        self.items.push(Item {
            name: name.to_string(),
            price_cents,
            qty,
        });
        self.total_cents += line_total;
    }

    /// Takes `percent` percent off the total.
    pub fn apply_discount(&mut self, percent: u64) {
        let discount = self.total_cents * (percent / 100);
        self.total_cents -= discount;
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn ten_percent_off() {
        let mut cart = Cart::new();
        cart.add("pen", 250, 4);
        cart.add("book", 1500, 1);
        cart.apply_discount(10);
        assert_eq!(cart.total_cents, 2250);
    }
}
```

`src/main.rs`:

```rust
use p09_debugging::Cart;

fn main() {
    let mut cart = Cart::new();
    cart.add("pen", 250, 4);
    cart.add("book", 1500, 1);
    cart.apply_discount(10);
    println!("total: {} cents", cart.total_cents);
}
```

(`src/main.rs` uses the library by its crate name `p09_debugging`, with an underscore. Extra programs go in `src/bin/`; we will use that in Section 6.)

```bash
cargo run
cargo test
```

```text
total: 2500 cents
```

```text
---- tests::ten_percent_off stdout ----

thread 'tests::ten_percent_off' (4658211) panicked at src/lib.rs:47:9:
assertion `left == right` failed
  left: 2500
 right: 2250
```

Four pens at 250 plus one book at 1500 is 2500. Ten percent off must give 2250, but the total is still 2500: the discount did nothing. Where is the mistake? You may already see it, but pretend you do not, and let the debugger show it.

## 2. Starting LLDB

Cargo's normal `cargo build` (the `dev` profile) already includes debug information, as you saw in Lesson 02.9. Start LLDB on the compiled program with `rust-lldb`:

```bash
cargo build
rust-lldb target/debug/p09-debugging
```

You get the prompt `(lldb)`. It is a **REPL** (Read-Eval-Print Loop): you type a command, LLDB runs it and prints the result. The commands you need:

| Command (short form) | What it does |
|---|---|
| `breakpoint set --file lib.rs --line 32` (`b lib.rs:32`) | Stop when line 32 of `lib.rs` is reached. |
| `breakpoint set --name apply_discount` (`b apply_discount`) | Stop at the start of a function. |
| `run` (`r`) | Start the program. |
| `continue` (`c`) | Run until the next stop. |
| `next` (`n`) | Step over: run the current line, stop at the next line. |
| `step` (`s`) | Step into: enter the function called on this line. |
| `finish` | Step out: run until the current function returns. |
| `bt` | Print the backtrace. `bt 3` prints only the three innermost frames. |
| `frame variable` (`fr v`) | Show the variables of the current frame. |
| `frame select 2` / `up` / `down` | Move to another frame of the backtrace. |
| `print EXPR` (`p`) | Evaluate an expression and print it. |
| `breakpoint list` / `breakpoint delete 1` | Show / remove breakpoints. |
| `quit` (`q`) | Leave LLDB. |

A debugger session is hard to show as one block, because you type and LLDB answers. In this lesson each block shows the commands after the `(lldb)` prompt, and the answers below them. (We shortened the output: LLDB prints long `Process …` lines and paths that are different on every computer. You can also run a list of commands without typing: `rust-lldb -b -o 'command 1' -o 'command 2' -- target/debug/p09-debugging`, where `-b` means "batch" and each `-o` is one command. Handy for repeating a session.)

> 💡 **Tip:** Use `rust-lldb`, not plain `lldb`. Plain `lldb` knows nothing about Rust types and shows only a raw address for `self`:
>
> ```text
> (p09_debugging::Cart *) self = 0x000000016fdfe0c0
> ```
>
> `rust-lldb` shows what is inside: `self = 0x000000016fdfe030 {items:size=2, total_cents:2500}`.

## 3. Breakpoints, the stack and variables

Set a breakpoint inside `apply_discount`, run, and look around:

```text
(lldb) breakpoint set --file lib.rs --line 32
Breakpoint 1: where = p09-debugging`<p09_debugging::Cart>::apply_discount + 28 at lib.rs:32:24, address = 0x0000000100000bd4
(lldb) run
Process 34145 stopped
* thread #1, name = 'main', queue = 'com.apple.main-thread', stop reason = breakpoint 1.1
      frame #0: 0x0000000100000bd4 p09-debugging`<p09_debugging::Cart>::apply_discount(self={items:size=2, total_cents:2500}, percent=10) at lib.rs:32:24
   29  	
   30  	    /// Takes `percent` percent off the total.
   31  	    pub fn apply_discount(&mut self, percent: u64) {
-> 32  	        let discount = self.total_cents * (percent / 100);
    	                       ^
   33  	        self.total_cents -= discount;
   34  	    }
   35  	}
(lldb) bt 2
* thread #1, name = 'main', queue = 'com.apple.main-thread', stop reason = breakpoint 1.1
  * frame #0: 0x0000000100000bd4 p09-debugging`<p09_debugging::Cart>::apply_discount(self={items:size=2, total_cents:2500}, percent=10) at lib.rs:32:24
    frame #1: 0x0000000100000ad0 p09-debugging`p09_debugging::main at main.rs:7:10
(lldb) frame variable
(p09_debugging::Cart *) self = 0x000000016fdfe030 {items:size=2, total_cents:2500}
(unsigned long) percent = 10
```

**How to read it:**

1. The breakpoint is "resolved" to the real place: function `apply_discount`, line 32. `+ 28` is the number of bytes from the start of the function.
2. The program **stopped before** running line 32. The arrow `->` marks the next line to run. LLDB shows the lines around it.
3. `bt 2` shows two frames. Frame 0 is the function that is running; frame 1 is its caller: `main`, at `main.rs:7`, the line `cart.apply_discount(10);`. The `*` marks the selected frame.
4. `frame variable` shows the variables: `self` is a pointer to a `Cart` that has 2 items and a total of 2500, and `percent` is 10. Everything is as we expect. No `discount` yet: line 32 has not run.

Now run line 32 with `next` and look again:

```text
(lldb) next
Process 34145 stopped
* thread #1, name = 'main', queue = 'com.apple.main-thread', stop reason = step over
      frame #0: 0x0000000100000c00 p09-debugging`<p09_debugging::Cart>::apply_discount(self={items:size=2, total_cents:2500}, percent=10) at lib.rs:33:9
   32  	        let discount = self.total_cents * (percent / 100);
-> 33  	        self.total_cents -= discount;
(lldb) frame variable
(p09_debugging::Cart *) self = 0x000000016fdfe030 {items:size=2, total_cents:2500}
(unsigned long) percent = 10
(unsigned long) discount = 0
(lldb) p percent / 100
(unsigned long) 0
```

There it is: `discount = 0`. And the expression `percent / 100` is `0` because both numbers are integers: `10 / 100` is `0` (integer division throws away the fraction, Lesson 02.3). We wanted `total * percent / 100`. The multiplication must happen before the division.

> ⚠️ **Common mistake:** `print` (`p`) in LLDB does not understand Rust. Its own messages say so: *"Expression evaluation in Rust not supported. Falling back to default language."* It evaluates simple things like `percent / 100`, field access with `->` (`p self->total_cents`) and variables, but not method calls: `p self->items.len()` fails with `error: called object type 'unsigned long' is not a function or function pointer`. Use `frame variable` (it always works), and for `Vec` look at the `size=` in its summary.

Fix the line, then check with the test:

```rust
        let discount = self.total_cents * percent.min(100) / 100;
```

`percent.min(100)` also stops the percent from going above 100 (Section 6 shows why we need that). Run `cargo test`: the test passes. For the next sections, keep the **buggy** version of the line (`self.total_cents * (percent / 100)`) in your file, so that your outputs match the lesson; fix it at the end.

## 4. Conditional breakpoints and watchpoints

A breakpoint at a line inside `add` stops twice: for the pen and for the book. In a loop that runs a million times, stopping every time is impossible. A **conditional breakpoint** stops only when a condition is true. Stop in `add` only when `qty > 1`:

```text
(lldb) breakpoint set --file lib.rs --line 27 --condition "qty > 1"
Breakpoint 1: where = p09-debugging`<p09_debugging::Cart>::add + 160 at lib.rs:27:9, address = 0x0000000100000ce8
(lldb) run
Process 36375 stopped
* thread #1, name = 'main', queue = 'com.apple.main-thread', stop reason = breakpoint 1.1
      frame #0: ... p09-debugging`<p09_debugging::Cart>::add(self={items:size=1, total_cents:0}, name="pen", price_cents=250, qty=4) at lib.rs:27:9
-> 27  	        self.total_cents += line_total;
(lldb) frame variable
(p09_debugging::Cart *) self = 0x000000016fdfe030 {items:size=1, total_cents:0}
(&str) name = "pen" {
  [0] = 'p'
  [1] = 'e'
  [2] = 'n'
}
(unsigned long) price_cents = 250
(unsigned int) qty = 4
(unsigned long) line_total = 1000
(lldb) breakpoint list
1: file = 'lib.rs', line = 27, exact_match = 0, locations = 1, resolved = 1, hit count = 1
Condition: qty > 1
```

The book (`qty = 1`) did not stop the program; only the pen did. `hit count = 1` says the breakpoint stopped once.

Now a different question: *who changes `total_cents`?* In a big program many functions could write to it. A **watchpoint** tells LLDB: "stop whenever this memory is written". The CPU (Central Processing Unit, the chip that runs the instructions) has special registers for it, so the program does not become slow. Stop in `main` after `Cart::new()` and watch the field:

```text
(lldb) breakpoint set --file main.rs --line 5
(lldb) run
(lldb) watchpoint set variable cart.total_cents
Watchpoint created: Watchpoint 1: addr = 0x16fdfe048 size = 8 state = enabled type = m
    watchpoint spec = 'cart.total_cents'
(lldb) continue

Watchpoint 1 hit:
old value: 0
new value: 1000
* thread #1, name = 'main', queue = 'com.apple.main-thread', stop reason = watchpoint 1
      frame #0: ... p09-debugging`<p09_debugging::Cart>::add(self={items:size=1, total_cents:1000}, name="pen", price_cents=250, qty=4) at lib.rs:28:6
(lldb) continue

Watchpoint 1 hit:
old value: 1000
new value: 2500
      frame #0: ... p09-debugging`<p09_debugging::Cart>::add(self={items:size=2, total_cents:2500}, name="book", price_cents=1500, qty=1) at lib.rs:28:6
(lldb) continue
total: 2500 cents
```

**How to read it:**

1. `size = 8` is the number of bytes watched: a `u64`.
2. The first stop shows the old and the new value, and the function that wrote it: `add`, for the pen. The second stop: `add` again, for the book. The frame is shown at line 28, the line after the write, because the CPU reports the write after it finished.
3. After the second `continue` the program printed `total: 2500 cents` and **there was no third stop in `apply_discount`**. `apply_discount` did run `self.total_cents -= discount`, but it subtracted 0, so the stored value did not change, and LLDB reports a watchpoint only when the value changes. That confirms what the breakpoint showed: the discount is zero.

(If you keep typing `continue`, you get one more stop: a watchpoint on a local variable of `main` is still active when `main` ends, and the stack space is reused by other code, for example by `fwrite`, the library function behind `println!`. The report shows `libsystem_c.dylib` as the frame. This is not a bug in your program. Delete the watchpoint with `watchpoint delete` when you are done.)

> 🔒 **Security:** Watchpoints are also how people investigate memory corruption: "who overwrote this buffer?". The tool stops at the exact write. Combined with `bt`, it gives you the full path of the bad write.

## 5. Debugging a failing test

The same tools work for tests. A test binary is a program too, but you must find its file. `cargo test --no-run` builds the tests without running them and prints the path:

```bash
cargo test --no-run
```

```text
  Executable unittests src/lib.rs (target/debug/build/p09-debugging/c14c00238fa9a8c4/out/p09_debugging-c14c00238fa9a8c4)
  Executable unittests src/main.rs (target/debug/build/p09-debugging/0366ed2c2e057707/out/p09_debugging-0366ed2c2e057707)
```

(The hash in the name, `c14c00238fa9a8c4`, is different on your computer. Copy the path of the line `unittests src/lib.rs`.) Start LLDB on that file. Everything after `--` is given to the test program as its arguments: the test name and `--exact` (run only a test with exactly this name):

```bash
rust-lldb -- target/debug/build/p09-debugging/c14c00238fa9a8c4/out/p09_debugging-c14c00238fa9a8c4 tests::ten_percent_off --exact
```

```text
(lldb) breakpoint set --file lib.rs --line 46
(lldb) run
running 1 test
* thread #2, name = 'tests::ten_percent_off', stop reason = breakpoint 1.1
-> 46  	        cart.apply_discount(10);
(lldb) step
* thread #2, name = 'tests::ten_percent_off', stop reason = step in
      frame #0: ... <p09_debugging::Cart>::apply_discount(self={items:size=2, total_cents:2500}, percent=10) at lib.rs:32:24
-> 32  	        let discount = self.total_cents * (percent / 100);
(lldb) step
-> 33  	        self.total_cents -= discount;
(lldb) frame variable
(p09_debugging::Cart *) self = 0x0000000170006628 {items:size=2, total_cents:2500}
(unsigned long) percent = 10
(unsigned long) discount = 0
(lldb) finish
* thread #2, name = 'tests::ten_percent_off', stop reason = step out
-> 46  	        cart.apply_discount(10);
(lldb) next
-> 47  	        assert_eq!(cart.total_cents, 2250);
(lldb) frame variable cart
(p09_debugging::Cart) cart = {items:size=2, total_cents:2500}
(lldb) continue
test tests::ten_percent_off ... FAILED
```

**What happens:**

1. The breakpoint is on the line `cart.apply_discount(10);` inside the test (line 46 in the file).
2. The test runs on **thread #2**, not on the main thread: the test harness (the program that `cargo test` builds) starts one thread per test. LLDB follows it.
3. `step` goes **into** `apply_discount`. A second `step` moves from line 32 to line 33, and `discount = 0` is visible.
4. `finish` runs until `apply_discount` returns, and we are back in the test, on the line of the call (the call just finished). `next` goes to the `assert_eq!` line, and `frame variable cart` shows the wrong total.
5. `continue` lets the test run to its end, and the harness prints `FAILED`.

> 🧪 **Testing:** A debugger does not replace a test. First make the failure **repeatable** in a small test, then use the debugger to understand it, then fix it, and keep the test. The test is what you keep.

## 6. Stopping at a panic, and the backtrace

Add a second program that makes a different mistake: a discount of 250 percent. Create `src/bin/overflow.rs`:

```rust
use p09_debugging::Cart;

fn main() {
    let mut cart = Cart::new();
    cart.add("pen", 250, 4);
    cart.apply_discount(250);
    println!("total: {} cents", cart.total_cents);
}
```

(With the buggy line, `250 / 100` is `2`, so the discount is twice the total, and `total -= discount` would go below zero. An unsigned number cannot be negative.)

```bash
cargo run --bin overflow
```

```text
thread 'main' (4664184) panicked at src/lib.rs:33:9:
attempt to subtract with overflow
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
```

The panic message already gives the file and the line. But who called `apply_discount` with the bad number? The environment variable `RUST_BACKTRACE=1` makes Rust print the backtrace of the panic:

```bash
RUST_BACKTRACE=1 cargo run --bin overflow
```

```text
thread 'main' (4664334) panicked at src/lib.rs:33:9:
attempt to subtract with overflow
stack backtrace:
   0: __rustc::rust_begin_unwind
             at /rustc/0ef1bc386059596dada8fb65b50425aab2ea637a/library/std/src/panicking.rs:676:5
   1: core::panicking::panic_fmt
             at /rustc/0ef1bc386059596dada8fb65b50425aab2ea637a/library/core/src/panicking.rs:80:14
   2: core::panicking::panic_const::panic_const_sub_overflow
             at /rustc/0ef1bc386059596dada8fb65b50425aab2ea637a/library/core/src/panicking.rs:175:17
   3: <p09_debugging::Cart>::apply_discount
             at ./src/lib.rs:33:9
   4: overflow::main
             at ./src/bin/overflow.rs:6:10
   5: <fn() as core::ops::function::FnOnce<()>>::call_once
             at /rustc/0ef1bc386059596dada8fb65b50425aab2ea637a/library/core/src/ops/function.rs:250:5
note: Some details are omitted, run with `RUST_BACKTRACE=full` for a verbose backtrace.
```

**How to read a backtrace:**

1. It lists the functions that were running, **innermost first**. Numbers 0 to 2 are the panic machinery inside Rust's standard library; ignore them.
2. **Frame 3 is the first of your own functions**: `Cart::apply_discount`, `src/lib.rs:33:9`. That is where the subtraction overflowed.
3. Frame 4 is the caller: `overflow::main` at `src/bin/overflow.rs:6:10`. That is the line that passed 250.
4. The lines after frame 5 are omitted: they are the code that starts `main`. The rule: **find the first frame whose file is yours.**

A backtrace shows *where*, but not the *values*. For values, stop the program at the panic and look at the frame. Panics go through a function named `rust_panic`; put a breakpoint on it:

```text
(lldb) breakpoint set --name rust_panic
(lldb) run
thread 'main' panicked at src/lib.rs:33:9:
attempt to subtract with overflow
* thread #1, name = 'main', queue = 'com.apple.main-thread', stop reason = breakpoint 1.1
      frame #0: 0x0000000100035374 overflow`__rustc::rust_panic at panicking.rs:883:16 [opt]
(lldb) bt 8
  * frame #0: ... overflow`__rustc::rust_panic at panicking.rs:883:16 [opt]
    frame #1: ... overflow`std::panicking::panic_with_hook at panicking.rs:836:5 [opt]
    ...
    frame #6: ... overflow`core::panicking::panic_const::panic_const_sub_overflow at panicking.rs:175:17 [opt]
    frame #7: ... overflow`<p09_debugging::Cart>::apply_discount(self={items:size=1, total_cents:1000}, percent=250) at lib.rs:33:9
(lldb) frame select 7
frame #7: ... overflow`<p09_debugging::Cart>::apply_discount(self={items:size=1, total_cents:1000}, percent=250) at lib.rs:33:9
-> 33  	        self.total_cents -= discount;
(lldb) frame variable
(p09_debugging::Cart *) self = 0x000000016fdfe030 {items:size=1, total_cents:1000}
(unsigned long) percent = 250
(unsigned long) discount = 2000
```

Now the picture is complete: `discount = 2000` is larger than `total_cents = 1000`. The frames of the standard library are marked `[opt]` (optimized) and have no variable information (`frame variable` on them says "no variable information is available"), so select the frame that is yours with `frame select 7`.

With `percent.min(100)` in the fix, a percent above 100 is cut to 100 and the total stops at zero.

> 📏 **Convention:** Look at the **first frame in your code**, not at the top of the backtrace. Do not set `RUST_BACKTRACE=1` in production by default if the output goes to users; it is a tool for developers.

## 7. The same in RustRover

RustRover uses LLDB too, so everything above exists as buttons. Use whichever is faster.

1. **Breakpoint:** click in the grey gutter next to the line number. A red dot appears. **Conditional breakpoint:** right-click the red dot (or press `Ctrl+Shift+F8`) and write the expression, such as `qty > 1`, into the **Condition** field. **Logging breakpoint:** hold `Shift` and click the gutter; it prints a value without stopping.
2. **Start:** click the green bug icon next to `main` or next to a `#[test]` function and choose **Debug**. RustRover builds with debug information and starts LLDB for you. To debug a single test, use the bug icon beside that test.
3. **Look around:** the **Debug** tool window has a **Threads & Variables** area. The frame list is your `bt`; click a frame to select it (like `frame select`). The variables of that frame are listed, and you can expand `self` and `items`. This view uses the same Rust-aware printers as `rust-lldb`.
4. **Step:** the toolbar buttons **Step Over** (like `next`), **Step Into** (like `step`), **Step Out** (like `finish`), **Run to Cursor** and **Resume Program** (like `continue`).
5. **Evaluate:** the **Evaluate Expression** action lets you type an expression and see its value, and **Watches** keeps expressions on screen while you step.
6. **Stop at a panic:** open the **LLDB** console tab of the **Debug** tool window (see the next item) and type `breakpoint set --name rust_panic`, as in Section 6. When the program stops, select your own frame in the frame list to see its variables.
7. **Watchpoint:** the way to add one differs between RustRover versions, so check the menu of a variable in the Variables list (right-click it). If you cannot find it, the **Debug** tool window also has an **LLDB** console tab, where you can type `watchpoint set variable cart.total_cents` exactly as in Section 4.

## 8. What a debugger cannot do

A debugger needs the debug information and a program that still looks like your source. Build the same program in release mode (with the debug information added by an environment variable) and try the same breakpoint:

```bash
CARGO_PROFILE_RELEASE_DEBUG=true cargo build --release --bin p09-debugging
rust-lldb target/release/p09-debugging
```

```text
(lldb) breakpoint set --file lib.rs --line 33
Breakpoint 1: no locations (pending).
WARNING:  Unable to resolve breakpoint to any actual locations.
(lldb) run
total: 2500 cents
Process 38249 exited with status = 0 (0x00000000)
```

**What happens:** the optimizer turned the small function into a few instructions inside `main` (Lesson 05.12 showed that the compiler removes work). There is no "line 33" in the machine code, so the breakpoint stays *pending* and never stops. The rule: **debug with the debug build** (`cargo build`, `cargo test`), and when a bug exists only in release mode, use logging (Lesson 09.10).

More limits:

- A debugger **changes timing**. A bug in threads (a race) can disappear when you stop one thread. Logging usually changes the timing less.
- It is **interactive**. A bug that happens on a server at 3 a.m. must be understood from logs, not from a debugger.
- It is slower to repeat than a test. Once you understand a bug, **write the test**.

## Common mistakes

**1. "Breakpoint 1: no locations (pending)".** The file name or line is wrong, or the build is optimized. Check `breakpoint set --file lib.rs --line 32` against the real file, and build with the plain `cargo build`.

**2. Using plain `lldb` and seeing only raw addresses.** Start `rust-lldb` instead.

**3. Debugging an old binary.** LLDB runs the file you gave it. After editing the code, run `cargo build` (or `cargo test --no-run`) again, otherwise you debug yesterday's program. (A changed hash in a test file name is also a sign of a new build.)

**4. Trying `p some_vec.len()`.** Method calls are not available in LLDB for Rust. Use `frame variable some_vec` and read `size=`.

**5. Stepping through standard library code.** If `step` enters unknown functions such as `core::fmt`, use `finish` to leave them, and prefer `next` on lines you do not care about.

## Try it yourself

1. Create `code/p09-debugging` with the three files of Section 1 (keep the buggy line). Confirm that `cargo test` fails.
2. Use `rust-lldb -b -o … -- target/debug/p09-debugging` or an interactive session to stop in `apply_discount`, run `frame variable`, `next`, `frame variable`, and find `discount = 0`.
3. Set the watchpoint on `cart.total_cents` and count the stops. How many are there before the program prints its line?
4. Run `RUST_BACKTRACE=1 cargo run --bin overflow` and find the first frame in your own code.
5. Fix the line with `percent.min(100)`, run `cargo test`, and also `cargo clippy --all-targets -- -D warnings`.

## Exercises

### Exercise 1 — A conditional breakpoint (easy)

In LLDB, make the program stop in `add` only for the line whose `price_cents` is greater than 1000. Which item stops, and what is its `qty`?

<details>
<summary>Hint</summary>

Use `--condition "price_cents > 1000"` on the line `self.total_cents += line_total;` (line 27), then `run` and `frame variable`.

</details>

<details>
<summary>Answer</summary>

```text
(lldb) breakpoint set --file lib.rs --line 27 --condition "price_cents > 1000"
(lldb) run
(lldb) frame variable name qty
```

The stop happens for `"book"`, with `qty = 1` (price 1500). The pen costs 250, so its condition is false.

Why it works: LLDB evaluates the condition each time the line is reached and resumes the program silently when the answer is false.

</details>

### Exercise 2 — A regression test (medium)

The 250 percent bug crashed `overflow`. Write a test `discount_over_100_percent_stops_at_zero` that adds a pen (250 × 4) and applies a 250 percent discount, and checks that the total is `0`. Run it **with** the fix `percent.min(100)` and **without** it. What does it print without the fix?

<details>
<summary>Hint</summary>

Put the test inside `mod tests` next to `ten_percent_off`. Without the fix, the test panics inside `apply_discount`, so `cargo test` reports it as failed.

</details>

<details>
<summary>Answer</summary>

```rust
    #[test]
    fn discount_over_100_percent_stops_at_zero() {
        let mut cart = Cart::new();
        cart.add("pen", 250, 4);
        cart.apply_discount(250);
        assert_eq!(cart.total_cents, 0);
    }
```

```text
test tests::discount_over_100_percent_stops_at_zero ... ok
test tests::ten_percent_off ... ok
test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

Without the fix, the test fails with `attempt to subtract with overflow`, the same panic as in Section 6. Why it works: with `percent.min(100)` the discount can be at most the whole total, so `total - discount` can reach 0 but never go below.

</details>

### Exercise 3 — The value at the panic (hard)

Using only LLDB and the `overflow` program (with the buggy line), find the value of `discount` at the moment of the panic **without** typing `bt`.

<details>
<summary>Hint</summary>

Break on `rust_panic`. The frame that belongs to `apply_discount` is a few frames up. You can move there with `up` (several times) or with `frame select N`.

</details>

<details>
<summary>Answer</summary>

```text
(lldb) breakpoint set --name rust_panic
(lldb) run
(lldb) frame select 7
(lldb) frame variable discount
(unsigned long) discount = 2000
```

Why it works: the panic starts in `apply_discount`, which called into the panic machinery of the standard library; 7 frames up is the frame with your variables. (The number 7 matches this build; if it differs on your computer, use `bt` once to find the frame number, or press `up` until you see `apply_discount`.) The value 2000 is `1000 * (250 / 100)`, which is larger than the total 1000.

</details>

## Quiz

1. What is the difference between `next` and `step`?

<details>
<summary>Answer</summary>

`next` (step over) runs the whole current line, including any function it calls, and stops on the next line. `step` (step into) enters the function called on the line.

</details>

2. Why does a watchpoint not stop when the code executes `self.total_cents -= 0`?

<details>
<summary>Answer</summary>

The value did not change (2500 stayed 2500), and LLDB reports a watchpoint only when the stored value changes.

</details>

3. In a backtrace, which frame do you read first?

<details>
<summary>Answer</summary>

The first frame that points to a file of your own project. Frames above it belong to the standard library's panic code.

</details>

4. Why does a breakpoint in a release build often stay "pending"?

<details>
<summary>Answer</summary>

The optimizer can inline, reorder or delete code, so some source lines no longer exist in the machine code. There is no address for LLDB to stop at.

</details>

5. Why use `rust-lldb` and not `lldb`?

<details>
<summary>Answer</summary>

`rust-lldb` loads helper files that print Rust types (`Vec`, `String`, `&str`) in a readable way. Plain `lldb` prints raw pointers and internal fields.

</details>

6. You understand a bug using the debugger. What should you do before you fix it?

<details>
<summary>Answer</summary>

Write a test that fails because of this bug (a regression test), then fix the code and see the test pass. The test keeps the bug from coming back.

</details>

## Summary

- A debugger pauses a running program so you can inspect it. Use the debug build and `rust-lldb`.
- The core commands: `breakpoint set`, `run`, `continue`, `next`, `step`, `finish`, `bt`, `frame variable`, `frame select`.
- A **conditional breakpoint** stops only for the case you care about. A **watchpoint** stops when a value is written, so it answers "who changes this?".
- A failing test can be debugged: get the binary path with `cargo test --no-run` and pass the test name after `--`.
- For a panic, read the backtrace (`RUST_BACKTRACE=1`) from the first frame in your own code, or break on `rust_panic` and use `frame select`.
- Optimized builds and timing bugs are weak points of a debugger; then use logs (next lesson). Always turn what you learn into a test.

## Sources and further reading

- [LLDB tutorial](https://lldb.llvm.org/use/tutorial.html) — the official introduction to LLDB commands, with the short forms.
- [LLDB command map](https://lldb.llvm.org/use/map.html) — a table of the commands of other debuggers and their LLDB equivalents, useful as a reference of all `breakpoint`, `watchpoint` and `frame` options.
- [RustRover: Debugging code](https://www.jetbrains.com/help/rust/debugging-code.html) — the official guide to the graphical debugger, breakpoints and stepping.
- [The Rust Programming Language — 9.1 Unrecoverable Errors with panic!](https://doc.rust-lang.org/book/ch09-01-unrecoverable-errors-with-panic.html) — panics and how to read a backtrace.

---

[🏠 Start](../START-HERE.md) · [← 09.8 Fuzzing](08-fuzzing.md) · [📋 Phase 09](README.md) · [09.10 Logging and tracing →](10-logging-and-tracing.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
