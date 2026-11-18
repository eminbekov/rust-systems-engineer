<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 22.23 Compilers track: a tree-walking interpreter](23-compilers-track-an-interpreter.md) · [📋 Phase 22](README.md) · [22.25 Compilers track: project and next steps →](25-compilers-track-project-and-next-steps.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/22-specializations/24-compilers-track-webassembly.md) <!-- nav -->

# Lesson 22.24 — Compilers track: WebAssembly with Rust

> **Phase 22 — Specializations** · Lesson 24 of 30 · ⏱ about 3 hours (2 sittings)

## What you will learn

- What WebAssembly is: a portable instruction format for a stack machine, with modules, imports, exports and a sandbox.
- How to read and write the WebAssembly text format (WAT) and run a module with Wasmtime.
- How to compile Rust to WebAssembly: a program for WASI and a library that exports functions.
- How to write a code generator that compiles Pebble expressions to WebAssembly.
- How to run WebAssembly from your own Rust program with the `wasmtime` crate, and stop endless loops with fuel.

## Before you start

- You finished [Lesson 22.23 — Compilers track: a tree-walking interpreter](23-compilers-track-an-interpreter.md).
- You know what machine code and a stack are from [Lesson 12.6 — How the CPU runs instructions](../12-computer-systems/06-how-the-cpu-runs-instructions.md)
  and [Lesson 12.8 — From Rust to machine code](../12-computer-systems/08-from-rust-to-machine-code.md).
- Install two Rust targets and the Wasmtime command-line tool (building it takes about ten minutes):

```bash
rustup target add wasm32-wasip1 wasm32-unknown-unknown
cargo install --locked wasmtime-cli
wasmtime --version
```

```text
wasmtime 50.0.2
```

This lesson uses three new small packages next to `code/p22-pebble`: `code/p22-wasm-hello`,
`code/p22-wasm-math` and `code/p22-wasm-host`. You create each one when the lesson gets to it.

## New words

| Word | Meaning |
|---|---|
| **WebAssembly** (Wasm) | A portable binary instruction format. A compiler produces a `.wasm` file once, and any WebAssembly runtime (a browser, Wasmtime) runs it on any CPU (Central Processing Unit) and operating system. |
| **module** | One WebAssembly file: functions, memory, and lists of **imports** (what it needs from the host) and **exports** (what it offers). |
| **host** | The program that loads and runs a WebAssembly module, for example the `wasmtime` tool or your own Rust program. |
| **stack machine** | A machine whose instructions take their operands from a stack and push their result on it, instead of naming registers. |
| **WAT** (WebAssembly Text format) | The human-readable text form of a module, with instructions such as `i64.add`. |
| **linear memory** | A module's memory: one array of bytes that the module can read and write, and nothing outside it. |
| **sandbox** | An environment where code can only do what the host explicitly allows. |
| **trap** | WebAssembly stops the running code immediately because of an error such as division by zero; the host gets an error. |
| **WASI** (WebAssembly System Interface) | A standard set of imports for system things such as files, clocks and standard output, so programs can run outside the browser. |
| **Wasmtime** | A WebAssembly runtime written in Rust, available as a command-line tool and as the `wasmtime` crate. |
| **`cdylib`** | A crate type for a library that other languages and hosts can load; for WebAssembly, it produces a `.wasm` module with exports. |
| **code generator** | The part of a compiler that turns the syntax tree into instructions of the target machine. |
| **fuel** | A budget of work: Wasmtime subtracts fuel as code runs and traps when it is gone. |

## 1. What WebAssembly is

Machine code is tied to one kind of CPU: a program compiled for your Mac's arm64 chip does not run on an
x86-64 server (Lesson 12.7). **WebAssembly** is an instruction set for a machine that does not exist as a
chip. Compilers produce it, and a **runtime** translates it into real machine code just before it runs.
The same `.wasm` file runs in every browser, on servers, on edge networks and inside other programs.

Three properties make it interesting for systems engineers:

- **Portable:** one file for every CPU and operating system.
- **Fast:** the runtime compiles it to machine code, so it runs close to native speed.
- **Sandboxed:** a module can compute and use its own linear memory. Everything else (files, network,
  printing, even the clock) must be **imported** from the host. A module that imports nothing can only
  compute. This is why WebAssembly is used for plugins: the host runs untrusted code safely.

WebAssembly is a **stack machine**. Instructions take their operands from a stack and push the result:

```text
 instruction        stack after it (top on the right)
 local.get $a       [2]
 local.get $b       [2, 3]
 i64.add            [5]          pops 3 and 2, pushes 2 + 3
```

Here is a whole module in the text format (WAT). Save it as `add.wat` anywhere, for example in
`code/p22-wasm-hello/`:

```text
(module
  ;; add(a, b) = a + b, for 64-bit integers
  (func (export "add") (param $a i64) (param $b i64) (result i64)
    local.get $a
    local.get $b
    i64.add))
```

**What happens:**

1. `(module ...)` contains everything. Lines starting with `;;` are comments.
2. `(func (export "add") ...)` defines a function and exports it under the name `add`. It takes two
   parameters of type `i64` (64-bit integer) and returns one.
3. The body is three instructions. The value left on the stack at the end is the result.

Run it with `wasmtime --invoke`, which calls one exported function with arguments:

```bash
wasmtime --invoke add add.wat 2 3
wasmtime --invoke add add.wat 9223372036854775807 1
```

```text
warning: using `--invoke` with a function that takes arguments is experimental and may break in the future
warning: using `--invoke` with a function that returns values is experimental and may break in the future
5
warning: using `--invoke` with a function that takes arguments is experimental and may break in the future
warning: using `--invoke` with a function that returns values is experimental and may break in the future
-9223372036854775808
```

The warnings go to standard error; add `2>/dev/null` to hide them (we do so from now on). Note the
second result: `i64.add` **wraps around** on overflow, it does not report an error. Wasmtime accepts the
text format directly and converts it to the binary format for you.

## 2. Rust programs on WASI

Rust can compile for two WebAssembly targets in this lesson:

| Target | For | Has `std`? |
|---|---|---|
| `wasm32-wasip1` | whole programs with `main`, run by a WASI runtime such as Wasmtime | yes: files, `println!`, arguments, through WASI imports |
| `wasm32-unknown-unknown` | libraries for any host; no operating system at all | the parts that need no operating system |

(`wasm32` means 32-bit addresses in linear memory; the integers can still be 64-bit.) Create the first
package and replace `src/main.rs`:

```bash
cd ~/projects/rust
cargo new code/p22-wasm-hello
cd code/p22-wasm-hello
echo "remember the milk" > notes.txt
```

```rust
use std::fs;

fn main() {
    println!("hello from WebAssembly");
    let args: Vec<String> = std::env::args().collect();
    println!("arguments: {args:?}");
    match fs::read_to_string("notes.txt") {
        Ok(text) => println!("notes.txt has {} bytes", text.len()),
        Err(e) => println!("cannot read notes.txt: {e}"),
    }
}
```

Build it for WASI and run it twice: once as it is, once with the current folder given to it:

```bash
cargo build --target wasm32-wasip1
wasmtime target/wasm32-wasip1/debug/p22-wasm-hello.wasm one two
wasmtime --dir . target/wasm32-wasip1/debug/p22-wasm-hello.wasm
```

```text
hello from WebAssembly
arguments: ["p22-wasm-hello.wasm", "one", "two"]
cannot read notes.txt: No such file or directory (os error 44)
hello from WebAssembly
arguments: ["p22-wasm-hello.wasm"]
notes.txt has 18 bytes
```

**What happens:**

1. `--target wasm32-wasip1` makes Cargo produce `target/wasm32-wasip1/debug/p22-wasm-hello.wasm` instead
   of a native program. Its first four bytes are `\0asm`, the WebAssembly "magic number".
2. `println!` and `std::env::args` work: the Rust standard library for this target calls WASI imports,
   and Wasmtime provides them.
3. The first run cannot read `notes.txt` although it is right there: the module has **no access to any
   folder** unless the host grants it. (Error 44 is the WASI code for "no such file".)
4. `--dir .` grants access to the current folder, and the same program reads the file. Access is given
   per folder, by the person who runs the program, not by the program itself.

> ⚡ **Performance:** The debug build is about 2.5 MB; `cargo build --release --target wasm32-wasip1`
> gives about 100 KB. Size matters when modules are downloaded or started thousands of times. Release
> builds, `opt-level = "s"` and `strip = true` in `Cargo.toml` (Lesson 19.8) make modules smaller.

## 3. A Rust library as a WebAssembly module

For a library that a host calls, use the `cdylib` crate type and `wasm32-unknown-unknown`. Create the
package:

```bash
cd ~/projects/rust
cargo new --lib code/p22-wasm-math
```

Add this to `Cargo.toml`:

```toml
[lib]
crate-type = ["cdylib"]
```

and replace `src/lib.rs` with:

```rust
/// The greatest common divisor, exported to WebAssembly under the name `gcd`.
#[unsafe(no_mangle)]
pub extern "C" fn gcd(mut a: i64, mut b: i64) -> i64 {
    while b != 0 {
        (a, b) = (b, a % b);
    }
    a
}

/// Integer division; WebAssembly stops ("traps") when `b` is zero.
#[unsafe(no_mangle)]
pub extern "C" fn divide(a: i64, b: i64) -> i64 {
    a / b
}
```

**What happens:**

1. `extern "C"` gives the functions a simple, stable calling convention (Lesson 06.10), the one that
   WebAssembly exports use.
2. `#[unsafe(no_mangle)]` keeps the exact names `gcd` and `divide` in the module; without it, the compiler
   would invent longer, unique names. The attribute is marked `unsafe` because two functions with the
   same unmangled name in one program would be a serious bug that the compiler cannot always detect.
3. The loop is Euclid's algorithm, the same one as in `programs/loops.pb` of Lesson 22.23.

Build and call it:

```bash
cargo build --release --target wasm32-unknown-unknown
wasmtime --invoke gcd target/wasm32-unknown-unknown/release/p22_wasm_math.wasm 1071 462 2>/dev/null
wasmtime --invoke divide target/wasm32-unknown-unknown/release/p22_wasm_math.wasm 7 0
```

```text
21
Error: failed to run main module `target/wasm32-unknown-unknown/release/p22_wasm_math.wasm`

Caused by:
    0: failed to invoke `divide`
    1: error while executing at wasm backtrace:
           ...
    2: wasm trap: wasm `unreachable` instruction executed
```

(The backtrace in the middle is shortened.) The Rust code panicked ("attempt to divide by zero"). On this
target a panic cannot unwind, so it becomes the `unreachable` instruction, and WebAssembly **traps**: the
module stops at once, and the host gets an error. Wasmtime itself did not crash. The release module is
about 20 KB.

## 4. A code generator for Pebble

Now the compiler part of the track. Instead of *running* the syntax tree, we *translate* it into
WebAssembly instructions, and Wasmtime runs those. Because WebAssembly is a stack machine, the
translation of an expression is simple: **emit the code for the operands, then the operator**. For
`x * x + 1`:

```text
 tree                    instructions                stack
 (+ (* x x) 1)           local.get $x                [x]
                         local.get $x                [x, x]
                         i64.mul                     [x*x]
                         i64.const 1                 [x*x, 1]
                         i64.add                     [x*x+1]
```

This is a walk of the tree in **post-order**: children first, then the node. We compile one expression
into a module with one exported function `run`, whose parameters are the expression's variables. Every
Pebble value becomes an `i64`; `true` and `false` become 1 and 0. Save as `src/wasm.rs` in
`code/p22-pebble` and add `pub mod wasm;` to `src/lib.rs`:

```rust
//! A code generator: compiles a Pebble expression to WebAssembly text (WAT).

use crate::ast::{BinOp, Expr, UnOp};

/// Compiles `expr` into a module that exports `run`. The names in `params`
/// become the parameters of `run`; every value is a 64-bit integer.
pub fn compile_expr(expr: &Expr, params: &[&str]) -> Result<String, String> {
    let mut body = String::new();
    emit(expr, params, &mut body)?;
    let params: String = params
        .iter()
        .map(|p| format!(" (param ${p} i64)"))
        .collect();
    Ok(format!(
        "(module\n  (func (export \"run\"){params} (result i64)\n{body}  )\n)\n"
    ))
}

/// Appends the instructions that leave the value of `expr` on the stack.
fn emit(expr: &Expr, params: &[&str], out: &mut String) -> Result<(), String> {
    match expr {
        Expr::Int(n) => push(out, &format!("i64.const {n}")),
        Expr::Bool(b) => push(out, &format!("i64.const {}", *b as i64)),
        Expr::Var(name, pos) => {
            if !params.contains(&name.as_str()) {
                return Err(format!(
                    "{}:{}: unknown variable `{name}`",
                    pos.line, pos.col
                ));
            }
            push(out, &format!("local.get ${name}"));
        }
        Expr::Unary(UnOp::Neg, e, _) => {
            push(out, "i64.const 0"); // -e is computed as 0 - e
            emit(e, params, out)?;
            push(out, "i64.sub");
        }
        Expr::Binary(op, l, r, _) => {
            emit(l, params, out)?;
            emit(r, params, out)?;
            let (instr, is_comparison) = match op {
                BinOp::Add => ("i64.add", false),
                BinOp::Sub => ("i64.sub", false),
                BinOp::Mul => ("i64.mul", false),
                BinOp::Div => ("i64.div_s", false),
                BinOp::Rem => ("i64.rem_s", false),
                BinOp::Eq => ("i64.eq", true),
                BinOp::Ne => ("i64.ne", true),
                BinOp::Lt => ("i64.lt_s", true),
                BinOp::Le => ("i64.le_s", true),
                BinOp::Gt => ("i64.gt_s", true),
                BinOp::Ge => ("i64.ge_s", true),
            };
            push(out, instr);
            if is_comparison {
                // A comparison leaves a 32-bit 0 or 1; widen it to i64.
                push(out, "i64.extend_i32_u");
            }
        }
    }
    Ok(())
}

/// Appends one indented instruction line.
fn push(out: &mut String, instr: &str) {
    out.push_str("    ");
    out.push_str(instr);
    out.push('\n');
}

#[cfg(test)]
mod tests {
    use super::*;
    use crate::ast::Stmt;
    use crate::parser::parse;

    fn expr(src: &str) -> Expr {
        match parse(&format!("{src};")).unwrap().remove(0) {
            Stmt::Expr(e) => e,
            other => panic!("not an expression: {other}"),
        }
    }

    #[test]
    fn operands_first_then_the_operator() {
        let wat = compile_expr(&expr("1 + 2 * x"), &["x"]).unwrap();
        let body: Vec<&str> = wat.lines().skip(2).map(str::trim).collect();
        assert_eq!(
            body,
            [
                "i64.const 1",
                "i64.const 2",
                "local.get $x",
                "i64.mul",
                "i64.add",
                ")",
                ")"
            ]
        );
    }

    #[test]
    fn unknown_variables_are_compile_errors() {
        let err = compile_expr(&expr("x + y"), &["x"]).unwrap_err();
        assert_eq!(err, "1:5: unknown variable `y`");
    }
}
```

**What happens:**

1. `compile_expr` generates the body first, then wraps it in a module text with one `(param $name i64)`
   for each variable. The result is WAT text in a `String`.
2. `emit` is the post-order walk. Literals push a constant, variables push a parameter with `local.get`
   (or fail if the name is not a parameter: a **compile-time error**, found before anything runs).
3. WebAssembly has no "negate" instruction for integers, so `-e` is compiled as `0 - e`.
4. Each Pebble operator maps to one instruction. `div_s` and `lt_s` are the **signed** versions (`_s`);
   WebAssembly also has unsigned ones (`_u`), because the bits of an `i64` mean different numbers in the
   two readings (Lesson 12.2).
5. Comparisons produce a 32-bit value (`i32`) 0 or 1. `i64.extend_i32_u` widens it to `i64`, so every
   value on our stack has the same type. (WebAssembly checks types when it loads a module; a mismatch is
   a load error, not a crash.)
6. The tests check the instruction order and the compile error.

> ⚠️ **Common mistake:** if you kept the operators from the exercises of Lesson 22.22 (`**`, `&&`, `||`, `!`), the two
> `match` expressions in `emit` do not cover them, and the build stops with error `E0004` ("non-exhaustive patterns").
> For now add the arm `` Expr::Unary(UnOp::Not, ..) => return Err("`!` is not supported by the compiler yet".to_string()), ``
> and, as the last arm of the operator `match`,
> `` _ => return Err(format!("`{}` is not supported by the compiler yet", op.symbol())), ``.
> The project in Lesson 22.25 compiles these operators for real.

A small program turns command-line arguments into a module. Save as `examples/compile.rs`:

```rust
//! Usage: cargo run --example compile -- "<expression>" [parameter ...]
use p22_pebble::ast::Stmt;
use p22_pebble::parser::parse;
use p22_pebble::wasm::compile_expr;

fn main() {
    let args: Vec<String> = std::env::args().skip(1).collect();
    let Some((src, params)) = args.split_first() else {
        eprintln!("usage: compile <expression> [parameter ...]");
        std::process::exit(2);
    };
    let params: Vec<&str> = params.iter().map(String::as_str).collect();
    let result = parse(&format!("{src};")).and_then(|mut program| match program.remove(0) {
        Stmt::Expr(e) => compile_expr(&e, &params),
        _ => Err("expected a single expression".to_string()),
    });
    match result {
        Ok(wat) => print!("{wat}"),
        Err(message) => {
            eprintln!("error: {message}");
            std::process::exit(1);
        }
    }
}
```

Compile two expressions and run them:

```bash
cargo run -q --example compile -- "x * x + 1" x > square.wat
cat square.wat
wasmtime --invoke run square.wat 7 2>/dev/null
cargo run -q --example compile -- "(a - b) * (a - b) < 10" a b > near.wat
wasmtime --invoke run near.wat 5 7 2>/dev/null
wasmtime --invoke run near.wat 5 70 2>/dev/null
```

```text
(module
  (func (export "run") (param $x i64) (result i64)
    local.get $x
    local.get $x
    i64.mul
    i64.const 1
    i64.add
  )
)
50
1
0
```

You have written a compiler: Pebble source in, WebAssembly out, and a real runtime turns it into machine
code. Compare with the interpreter of Lesson 22.23, which walks the tree *every time* it runs; here the
tree is walked once, at compile time.

The compiled code does not always behave like the interpreter, and that matters:

```bash
cargo run -q --example compile -- "a / b" a b > div.wat
wasmtime --invoke run div.wat 7 0 2>&1 | tail -1
```

```text
    2: wasm trap: integer divide by zero
```

Division by zero is a trap (the interpreter gave a run-time error with a position), and overflow wraps
around (the interpreter reported an error). Two implementations of one language must agree; in the
project (Lesson 22.25) you make them agree and test it.

> 🧪 **Testing:** A code generator is tested at two levels: the *shape* of the output (the instruction
> list, as in our unit test) and its *behaviour* (run the module and check the result). The second kind
> catches more bugs; the project runs both the interpreter and the compiled module on the same programs
> and compares the results (differential testing).

## 5. Running WebAssembly from Rust

The `wasmtime` command is itself a Rust program built on the `wasmtime` crate, and your programs can use
the same crate to load and run modules: a plugin system, a rules engine, or the test harness of your
compiler. Create the package:

```bash
cd ~/projects/rust
cargo new code/p22-wasm-host
cd code/p22-wasm-host
cargo add wasmtime@50
cargo add anyhow@1
```

Replace `src/main.rs`:

```rust
use anyhow::Result;
use wasmtime::{Config, Engine, Instance, Module, Store};

const WAT: &str = r#"
(module
  (func (export "add") (param $a i64) (param $b i64) (result i64)
    local.get $a
    local.get $b
    i64.add)
  (func (export "spin") (loop $again br $again)))
"#;

fn main() -> Result<()> {
    let mut config = Config::new();
    config.consume_fuel(true); // count work, so endless loops can be stopped
    let engine = Engine::new(&config)?;
    let module = Module::new(&engine, WAT)?; // compiles the module to machine code

    let mut store = Store::new(&engine, ());
    store.set_fuel(1_000_000)?;
    let instance = Instance::new(&mut store, &module, &[])?;

    let add = instance.get_typed_func::<(i64, i64), i64>(&mut store, "add")?;
    println!("add(40, 2) = {}", add.call(&mut store, (40, 2))?);

    let spin = instance.get_typed_func::<(), ()>(&mut store, "spin")?;
    match spin.call(&mut store, ()) {
        Ok(()) => println!("spin returned"),
        Err(e) => println!("spin stopped: {}", e.root_cause()),
    }
    println!("fuel left: {}", store.get_fuel()?);
    Ok(())
}
```

`cargo run` (the first build compiles Wasmtime and takes a few minutes):

```text
add(40, 2) = 42
spin stopped: wasm trap: all fuel consumed by WebAssembly
fuel left: 0
```

**What happens:**

1. An `Engine` holds the compiler settings; `consume_fuel(true)` makes the generated machine code count
   its work. A `Module` is compiled code; `Module::new` accepts WAT text (and binary `.wasm`).
2. A `Store` holds the running state of instances (their memories, their fuel); `set_fuel` gives it a
   budget. `Instance::new` creates a running instance; the empty slice `&[]` means "no imports", so this
   module can do nothing but compute.
3. `get_typed_func::<(i64, i64), i64>` looks up an export and checks its type once, so `call` is cheap
   and type-safe afterwards.
4. `spin` is an endless loop (`br $again` jumps back to the start of the `loop`). Without fuel the call
   would never return; with fuel, Wasmtime traps when the budget is used up, and the host continues. This
   is the step limit of Lesson 22.23, exercise 3, done by the runtime.

> 🔒 **Security:** WebAssembly gives memory isolation (a module cannot read the host's memory) and
> capability-based access (nothing is reachable unless imported). It does not limit time or memory by
> itself: set fuel (or epoch interruption) and memory limits when you run code you do not trust, and keep
> Wasmtime up to date, because the runtime is part of your attack surface.

## Common mistakes

**Building for the wrong target.** `cargo build` without `--target` builds a native program;
`wasmtime target/debug/p22-wasm-hello` then fails because the file is not a WebAssembly module. The
`.wasm` file is in `target/<target>/...`.

**Forgetting `#[unsafe(no_mangle)]` or `crate-type = ["cdylib"]`.** Without them the module has no
`gcd` export, and `wasmtime --invoke gcd` ends with ``no func export named `gcd` found``. Check what a module
exports by reading its text: the `wasm-tools` program (`cargo install --locked wasm-tools`, then
`wasm-tools print module.wasm`) prints any `.wasm` as WAT.

**Testing panics inside `extern "C"` functions.** A unit test such as
`#[should_panic] fn dividing_by_zero_panics() { divide(7, 0); }` in `p22-wasm-math` does not pass, it
aborts the whole test run on your Mac:

```text
thread 'tests::dividing_by_zero_panics' (6484616) panicked at src/lib.rs:13:5:
attempt to divide by zero
...
panic in a function that cannot unwind
...
thread caused non-unwinding panic. aborting.
```

An `extern "C"` function promises not to unwind, so a panic inside it aborts the process. Exported
functions should not panic at all: check inputs and return a value that means "error", or keep the logic
in a normal Rust function, test that, and let the `extern "C"` function only call it.

**Assuming compiled code behaves like the interpreter.** Overflow wraps and division by zero traps in
WebAssembly. Decide what *your language* means, then make both implementations do exactly that.

## Try it yourself

1. Run `xxd target/wasm32-wasip1/release/p22-wasm-hello.wasm | head -1` and find the magic number
   `0061 736d` (`\0asm`) and the version `0100 0000` (1).
2. Compile `-x` with `examples/compile.rs` and read the WAT: you see `i64.const 0`, `local.get $x`,
   `i64.sub`.
3. In `p22-wasm-host`, change `set_fuel(1_000_000)` to `set_fuel(3)`. `add` still works: its three
   instructions need only a few units of fuel (most simple instructions cost one unit).

## Exercises

### Exercise 1 — WAT by hand (easy)

Write `rect.wat` with two exported functions: `area(w, h) = w * h` and `perimeter(w, h) = 2 * (w + h)`.
Check: `area` of 6 and 7 is 42, `perimeter` is 26.

<details>
<summary>Hint</summary>

For `perimeter`, push `2` first, then compute `w + h`, then multiply: the stack is `[2, w+h]` before
`i64.mul`.

</details>

<details>
<summary>Answer</summary>

```text
(module
  ;; area(w, h) = w * h
  (func (export "area") (param $w i64) (param $h i64) (result i64)
    local.get $w
    local.get $h
    i64.mul)
  ;; perimeter(w, h) = 2 * (w + h)
  (func (export "perimeter") (param $w i64) (param $h i64) (result i64)
    i64.const 2
    local.get $w
    local.get $h
    i64.add
    i64.mul))
```

```bash
wasmtime --invoke area rect.wat 6 7 2>/dev/null
wasmtime --invoke perimeter rect.wat 6 7 2>/dev/null
```

```text
42
26
```

Why it works: each function's body leaves exactly one `i64` on the stack, which matches `(result i64)`.
If you forget an instruction, Wasmtime refuses to load the module with a type error, before anything runs.

</details>

### Exercise 2 — Test the library on your Mac (medium)

A `cdylib` can still have unit tests that run natively. Add a test to `p22-wasm-math` that checks
`gcd(1071, 462) == 21`, `gcd(17, 5) == 1` and `gcd(12, 0) == 12`, and run `cargo test` (without
`--target`). Why is `divide(7, 0)` not tested the same way?

<details>
<summary>Hint</summary>

`cargo test` builds for your own machine, where the `extern "C"` functions are normal functions you can
call. Read the "Common mistakes" section about panics in `extern "C"` functions.

</details>

<details>
<summary>Answer</summary>

At the end of `src/lib.rs`:

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn gcd_of_known_pairs() {
        assert_eq!(gcd(1071, 462), 21);
        assert_eq!(gcd(17, 5), 1);
        assert_eq!(gcd(12, 0), 12);
    }
}
```

```text
test tests::gcd_of_known_pairs ... ok
```

Why it works: the test calls `gcd` directly as a Rust function. `divide(7, 0)` panics inside an
`extern "C"` function, which cannot unwind, so the test process aborts instead of reporting a failed or
passed test. Move such logic into a normal function (for example one that returns `Option<i64>`) and test
that.

</details>

### Exercise 3 — A host that runs compiled Pebble (hard)

Write `src/bin/run.rs` in `p22-wasm-host`: `cargo run --bin run -- <module.wat> [numbers...]` loads the
module, calls its `run` export with the numbers as `i64` arguments (any count), prints the result, and
uses fuel. Test it with `square.wat`, `near.wat` and `div.wat` from section 4.

<details>
<summary>Hint</summary>

The number of parameters is not known at compile time, so use the untyped API (Application Programming Interface): `instance.get_func(...)`
gives a `Func`, and `func.call(&mut store, &params, &mut results)` takes slices of `wasmtime::Val`
(`Val::I64(n)`). `Module::from_file` reads `.wat` and `.wasm` files.

</details>

<details>
<summary>Answer</summary>

```rust
//! Usage: cargo run --bin run -- <module.wat> [i64 argument ...]
use anyhow::{Context, Result};
use wasmtime::{Config, Engine, Instance, Module, Store, Val};

fn main() -> Result<()> {
    let args: Vec<String> = std::env::args().skip(1).collect();
    let (path, numbers) = args
        .split_first()
        .context("usage: run <module.wat> [arg ...]")?;

    let mut config = Config::new();
    config.consume_fuel(true);
    let engine = Engine::new(&config)?;
    let module = Module::from_file(&engine, path)?;
    let mut store = Store::new(&engine, ());
    store.set_fuel(1_000_000)?;
    let instance = Instance::new(&mut store, &module, &[])?;

    let run = instance
        .get_func(&mut store, "run")
        .context("the module has no `run` export")?;
    let params = numbers
        .iter()
        .map(|n| n.parse::<i64>().map(Val::I64))
        .collect::<Result<Vec<Val>, _>>()?;
    let mut results = [Val::I64(0)];
    run.call(&mut store, &params, &mut results)?;
    println!("{}", results[0].unwrap_i64());
    Ok(())
}
```

```bash
cargo run -q --bin run -- ../p22-pebble/square.wat 7
cargo run -q --bin run -- ../p22-pebble/near.wat 5 70
cargo run -q --bin run -- ../p22-pebble/div.wat 7 0
cargo run -q --bin run -- ../p22-pebble/square.wat 1 2
```

```text
50
0
Error: error while executing at wasm backtrace:
    0:     0x27 - <unknown>!<wasm function 0>

Caused by:
    wasm trap: integer divide by zero
Error: expected 1 arguments, got 2
```

Why it works: `Func::call` checks the number and types of the arguments against the function's type and
returns an error instead of calling with the wrong arguments. Traps come back as ordinary `Err` values,
so `?` passes them to `main`, which prints them and exits with code 1.

</details>

## Quiz

1. Why can one `.wasm` file run on both an arm64 Mac and an x86-64 server?

<details>
<summary>Answer</summary>

WebAssembly is an instruction set for a virtual machine, not for a real CPU. The runtime on each machine
compiles the module into that machine's own code just before running it.

</details>

2. What can a WebAssembly module that imports nothing do?

<details>
<summary>Answer</summary>

Only compute: use its own linear memory and its functions, and return results to the host. It cannot
read files, use the network, print or read the clock, because all of these must be imported.

</details>

3. What is the difference between `wasm32-wasip1` and `wasm32-unknown-unknown`?

<details>
<summary>Answer</summary>

`wasm32-wasip1` is for whole programs that use WASI imports (files, standard output, arguments) and run
in a WASI runtime. `wasm32-unknown-unknown` assumes no operating system at all; it is used for libraries
whose host provides everything.

</details>

4. In which order does a stack-machine code generator emit the parts of `a - b`, and why?

<details>
<summary>Answer</summary>

First the code for `a`, then for `b`, then `i64.sub`. The operator takes its operands from the stack, so
they must be there before it runs (post-order traversal).

</details>

5. What is a trap, and what happens to the host when a module traps?

<details>
<summary>Answer</summary>

A trap is an immediate stop of the WebAssembly code because of an error such as division by zero, an
`unreachable` instruction or exhausted fuel. The host receives an error value and keeps running.

</details>

6. Name two differences between the compiled Pebble expressions and the interpreter of Lesson 22.23.

<details>
<summary>Answer</summary>

Overflow wraps around in WebAssembly but is an error in the interpreter; division by zero traps in
WebAssembly but is a run-time error with a position in the interpreter. (Also: booleans become the
integers 0 and 1.)

</details>

## Summary

- WebAssembly is a portable, sandboxed instruction format for a stack machine; a runtime such as Wasmtime
  compiles it to machine code.
- A module has functions, linear memory, imports and exports; without imports it can only compute.
- `wasm32-wasip1` builds whole Rust programs that use WASI; folders must be granted with `--dir`.
- `wasm32-unknown-unknown` with `crate-type = ["cdylib"]` and `#[unsafe(no_mangle)] extern "C"` builds
  libraries that export functions; a panic there becomes a trap.
- A code generator for a stack machine emits operands first, then the operator: a post-order walk of the
  syntax tree.
- The `wasmtime` crate embeds the runtime in your programs; fuel stops endless loops.

## Sources and further reading

- [WebAssembly](https://webassembly.org/) — the official site, with the list of features and runtimes.
- [WebAssembly Core Specification](https://webassembly.github.io/spec/core/) — the exact definition of every instruction and of module validation.
- [MDN: Understanding WebAssembly text format](https://developer.mozilla.org/en-US/docs/WebAssembly/Guides/Understanding_the_text_format) — a friendly guide to WAT (MDN, the MDN Web Docs, is Mozilla's web documentation).
- [Wasmtime documentation](https://docs.wasmtime.dev/) and [installing the CLI](https://docs.wasmtime.dev/cli-install.html) — the command-line tool (CLI, command-line interface), fuel, and embedding.
- [wasmtime crate on docs.rs](https://docs.rs/wasmtime/50/wasmtime/) — `Engine`, `Store`, `Module`, `Instance`, `Func`.
- [WASI](https://wasi.dev/) — the WebAssembly System Interface and its versions.
- [rustc book: wasm32-wasip1](https://doc.rust-lang.org/rustc/platform-support/wasm32-wasip1.html) and [wasm32-unknown-unknown](https://doc.rust-lang.org/rustc/platform-support/wasm32-unknown-unknown.html) — what each Rust target supports.

---

[🏠 Start](../START-HERE.md) · [← 22.23 Compilers track: a tree-walking interpreter](23-compilers-track-an-interpreter.md) · [📋 Phase 22](README.md) · [22.25 Compilers track: project and next steps →](25-compilers-track-project-and-next-steps.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
