<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 22.24 Compilers track: WebAssembly with Rust](24-compilers-track-webassembly.md) · [📋 Phase 22](README.md) · [22.26 Cloud-native track: Kubernetes operators →](26-cloud-native-track-kubernetes-operators.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/22-specializations/25-compilers-track-project-and-next-steps.md) <!-- nav -->

# Lesson 22.25 — Compilers track: project and next steps

> **Phase 22 — Specializations** · Lesson 25 of 30 · ⏱ about 20 hours (10 sittings)

## What you will learn

- How to finish a small language: functions, calls, recursion and `return`, in the parser and the interpreter.
- How a static type checker finds errors before a program runs.
- How to compile whole programs, with locals, loops, branches and function calls, to WebAssembly.
- How to make two implementations of one language agree, and prove it with a differential test.
- Where to go next in compilers and WebAssembly: books, documentation, projects and communities.

## Before you start

- You finished [Lesson 22.22](22-compilers-track-lexing-and-parsing.md),
  [Lesson 22.23](23-compilers-track-an-interpreter.md) and
  [Lesson 22.24 — Compilers track: WebAssembly with Rust](24-compilers-track-webassembly.md), including
  the logic operators (exercise 3 of 22.22 and exercise 2 of 22.23).
- You continue in `code/p22-pebble`. Add the runtime to it: `cargo add wasmtime@50` and `cargo add anyhow@1`.
- Commit your current state before you start: `git commit -am "Pebble before the project"`.

## New words

| Word | Meaning |
|---|---|
| **call frame** | The variables of one running function call. Every call gets a fresh frame; it disappears when the call returns. |
| **recursion depth** | How many calls are active at the same time, one inside another. |
| **static type checker** | A compiler pass that gives every expression a type (`int` or `bool`) and reports mismatches *before* the program runs. "Static" means "from the source, without running it". |
| **compiler pass** | One walk over the syntax tree that does one job: type checking, code generation, and so on. |
| **back end** | The part of a language implementation that runs or translates the checked tree; Pebble gets two back ends, the interpreter and the WebAssembly compiler. |
| **import (host function)** | A function that a WebAssembly module calls but the host provides, such as `print_int`. |
| **`Linker`** | The `wasmtime` type that connects a module's imports to host functions before the module is instantiated. |
| **semantics** | What programs of a language *mean*: what each operation computes, and what happens on errors. |
| **differential test** | A test that runs two implementations on the same inputs and checks that the results are equal. |

**Abbreviations from Lesson 22.24:** WAT (WebAssembly Text format, the readable form of a module) and WASI
(WebAssembly System Interface, the standard imports for files, clocks and output).

## 1. The finished language

At the end of the project, Pebble has functions. Here is a program you will run both ways:

```text
// Fibonacci numbers, the slow recursive way.
fn fib(n) {
    if n < 2 {
        return n;
    }
    return fib(n - 1) + fib(n - 2);
}

print fib(10);
print fib(20);
```

The implementation becomes a pipeline with two back ends:

```text
 source ─▶ lexer ─▶ parser ─▶ type checker ─┬─▶ interpreter ──────────────────▶ output
                                            │
                                            └─▶ code generator ─▶ WAT ─▶ Wasmtime ─▶ output
                                                                         (print_int, print_bool
                                                                          are host functions)
                     the differential test checks that both outputs are equal
```

## The project

### What you will build

`p22-pebble`, a complete small language with one command-line program and four commands: `check`
(type-check only), `run` (the interpreter), `compile` (print the WebAssembly text) and `wasm` (compile and
run in an embedded Wasmtime).

### Contract

**Command line:** `p22-pebble <run|check|compile|wasm> <file.pb>`. Exit code 0 on success, 1 on any
error (syntax, type or run time), 2 on wrong usage.

**Language additions:**

```text
statement := ...
           | "fn" IDENT "(" (IDENT ("," IDENT)*)? ")" block      (only at the top level)
           | "return" expr ";"                                    (only inside a function)
expr      := ... | IDENT "(" (expr ("," expr)*)? ")"              (a call)
```

**Semantics (both back ends must follow them exactly):**

1. Function parameters and return values are `int`. A function that ends without `return` returns `0`.
2. A function sees only its parameters and its own `let` variables, not the variables of the caller or of
   the top level. Functions may call each other and themselves, in any order of definition.
3. Integer `+ - *` and prefix `-` **wrap around** on overflow (two's complement, Lesson 12.2).
4. Division or remainder by zero is the run-time error `division by zero`; `i64::MIN / -1` is
   `integer overflow`.
5. More than 200 active calls is the run-time error `call stack too deep`.
6. `print` writes an `int` as a decimal number and a `bool` as `true` or `false`, one value per line.
7. Errors go to standard error: `<file>:<line>:<col>: type error: <message>` from `check` (all of them,
   one per line) and `<file>: runtime error: <message>` from `run` and `wasm`. Output printed before a
   run-time error stays printed.

**Out of scope** (stretch goals): types other than `int` and `bool`, strings, arrays, closures.

### Step-by-step plan

#### M1 — Functions in the parser

**Goal:** parse function definitions, `return` and calls.

**Guidance:** three new tokens (`Fn`, `Return`, `Comma`), `Stmt::Fn(name, params, body, pos)`,
`Stmt::Return(expr, pos)` and `Expr::Call(name, args, pos)`. A call is a name followed by `(`: check the
next token in `prefix`, just as `statement` does for assignment. Print them as S-expressions:
`(fn fib (n) (block ...))`, `(return ...)`, `(call fib (- n 1))`.

<details>
<summary>Hint</summary>

Write a helper `fn ident(&mut self, what: &str) -> Result<String, ParseError>` that takes one identifier
or fails with "expected <what>". Parameter lists and argument lists are "zero or more, separated by
commas": parse one, then loop while the next token is a comma.

</details>

**Done when:** a parser test checks that `fn add(a, b) { return a + b; } print add(1, 2);` prints as
`(fn add (a b) (block (return (+ a b))))` and `(print (call add 1 2))`, and the old tests still pass.
Commit.

#### M2 — Functions in the interpreter

**Goal:** calls with fresh frames, recursion, `return`, the depth limit.

**Guidance:** before running, collect all `Stmt::Fn` into a `HashMap<String, Rc<...>>` (so calls can come
before definitions). `return` must leave every enclosing block and loop at once: make `exec` return
`Result<Flow, RuntimeError>` with `enum Flow { Normal, Return(Value) }`, and stop executing a block as soon
as a statement returns `Flow::Return`. For a call, evaluate the arguments in the caller, then swap
`self.scopes` with a new stack that holds only the parameters, run the body, and swap back. Switch
`+ - *` to `wrapping_*` (semantics 3).

<details>
<summary>Hint</summary>

`std::mem::replace(&mut self.scopes, vec![frame])` gives you the caller's scopes back as a value to
restore later. Count the depth in a field and check it *before* the call. Keep the limit modest: the
interpreter recurses in Rust for every Pebble call, and test threads have smaller stacks than `main`
(2 MiB by default); with a limit of 1,000 the differential test of M5 overflowed its stack in a debug
build.

</details>

**Done when:**

```bash
cargo run -- run programs/fib.pb
```

```text
55
6765
```

and `programs/deep.pb`

```text
fn down(n) {
    return down(n + 1);
}
print down(0);
```

gives, on standard error, `programs/deep.pb: runtime error: call stack too deep` with exit code 1. Commit.

#### M3 — A static type checker

**Goal:** `check` reports type errors with positions before anything runs; `run`, `compile` and `wasm`
refuse programs with type errors.

**Guidance:** a new module `src/check.rs` with `pub fn check(program: &[Stmt]) -> Result<(), Vec<String>>`.
Walk the tree with a stack of scopes that maps names to `Type::Int` or `Type::Bool`. Rules: arithmetic and
`< <= > >=` need two ints; `&&`, `||`, `!` need bools; `==` and `!=` need two operands of the same type;
conditions are bools; arguments and `return` values are ints; calls need a known function with the right
number of arguments. Collect *all* errors instead of stopping at the first one.

<details>
<summary>Hint</summary>

Let the method for expressions return `Option<Type>`: `None` means "an error was already reported inside
this expression". Then one mistake does not produce a cascade of follow-up errors. Also check the rules
"`fn` only at the top level" and "`return` only inside a function".

</details>

**Done when:** with `programs/bad_types.pb`

```text
let ready = true;
print ready + 1;
let n = 5;
if n {
    print n;
}
```

`cargo run -- check programs/bad_types.pb` prints

```text
programs/bad_types.pb:2:13: type error: `+` cannot combine bool and int
programs/bad_types.pb:4:4: type error: expected bool, found int
```

and exits with 1, while `cargo run -- check programs/fib.pb` prints `programs/fib.pb: ok`. Commit.

#### M4 — Compile whole programs to WebAssembly

**Goal:** `compile` prints a WAT module; `wasm` runs it in Wasmtime with two host functions.

**Guidance:** extend the code generator of Lesson 22.24 into `src/compile.rs`:

- Every Pebble function becomes `(func $f_<name> (param $<p> i64)... (result i64) ...)`, ending with
  `i64.const 0` for the "no `return`" case. The top-level statements become `(func (export "main") ...)`.
- Every `let` becomes a new local with a unique name (`$x_0`, `$x_1`, ...), declared at the start of its
  function as `(local $x_0 i64)`. A stack of scopes maps Pebble names to WAT locals, so shadowing works.
- `print` calls the import `$print_int` or `$print_bool`, depending on the expression's type; declare them
  at the top: `(import "env" "print_int" (func $print_int (param i64)))`.
- `if` and `while` need an `i32` condition: `i32.wrap_i64` before `if`; for `while`, use
  `block $exit loop $again <cond> i64.eqz br_if $exit <body> br $again end end`.
- `&&` and `||` short-circuit with `if (result i64) ... else ... end`.
- An expression statement ends with `drop`; `return` is `<expr> return`.

For `wasm`, write `src/runwasm.rs`: a `wasmtime::Linker<String>` whose two host functions append to the
`String` in the store (`func_wrap("env", "print_int", |mut caller: Caller<'_, String>, v: i64| ...)`),
fuel, and a call of `main`. Translate traps into the messages of the contract with
`error.downcast_ref::<wasmtime::Trap>()`: `IntegerDivisionByZero`, `IntegerOverflow`, `StackOverflow`.

<details>
<summary>Hint</summary>

Generate a function's body first (into a `String`) while you collect its locals, and only then write the
`(func ...)` header with the `(local ...)` list. To know the type of an expression for `print`, the code
generator can compute it on the way: literals, comparisons and logic give bool, arithmetic and calls give
int, variables have the type stored in the scope map.

</details>

**Done when:**

```bash
cargo run -- compile programs/fib.pb | head -12
cargo run -- wasm programs/fib.pb
```

```text
(module
  (import "env" "print_int" (func $print_int (param i64)))
  (import "env" "print_bool" (func $print_bool (param i64)))
  (func $f_fib (param $n i64) (result i64)
    local.get $n
    i64.const 2
    i64.lt_s
    i64.extend_i32_u
    i32.wrap_i64
    if
    local.get $n
    return
55
6765
```

(Your instructions may differ in details; the output of `wasm` must not.) Commit.

#### M5 — Make the back ends agree, and prove it

**Goal:** both back ends follow the semantics of the contract, checked by a differential test.

**Guidance:** add programs to `programs/` that exercise each rule: loops, recursion, short-circuit logic,
wrapping (`9223372036854775807 + 1`), division by zero after some output, the depth limit. Then write
`tests/differential.rs`: for every file in `programs/` that passes the type checker, run the interpreter
and the compiled module and `assert_eq!` their results, including the output printed before an error and
the error message.

<details>
<summary>Hint</summary>

Give both back ends the same result type, for example `Result<String, (String, String)>` (the output, or
the output so far plus the message). Then the test is a single `assert_eq!(interpreted, compiled, "{}",
path.display())`. The trap for exhausted Wasm stack plays the role of "call stack too deep" in the
compiled back end.

</details>

**Done when:** `cargo test --test differential` passes, for example

```text
test interpreter_and_compiler_agree_on_every_program ... ok
```

and `programs/zero.pb`

```text
fn ratio(a, b) {
    return a / b;
}
print ratio(10, 3);
print ratio(1, 0);
print 99;
```

prints `3` and then `programs/zero.pb: runtime error: division by zero` with **both** `run` and `wasm`.
Commit.

#### M6 — Quality, measurements and a README

**Goal:** a project you can show.

**Guidance:** `cargo clippy --all-targets -- -D warnings` and `cargo fmt --check` clean; unit tests for
the checker's messages and for the code generator; a `README.md` with the language reference (syntax and
the semantics list), the four commands, and a measurement. Measure with `hyperfine` (Lesson 09.11) on a
release build, with `fib(27)` in a file:

```bash
cargo build --release
hyperfine -w 1 -r 5 './target/release/p22-pebble run programs/fib27.pb' './target/release/p22-pebble wasm programs/fib27.pb'
```

<details>
<summary>Hint</summary>

The `wasm` command includes compiling the module with Wasmtime, so very short programs can be slower
than with the interpreter. Measure both a short and a long program, and explain the difference in the
README.

</details>

**Done when:** clippy and fmt print nothing, all tests pass, and your README contains a table like this
one (measured on an Apple M4 Mac; your numbers will differ):

```text
Benchmark 1: ./target/release/p22-pebble run programs/fib27.pb (5 runs)
  Wall Time    110.3 ms  ±     2.1 ms     108.1 ms  …   113.7 ms
Benchmark 2: ./target/release/p22-pebble wasm programs/fib27.pb (5 runs)
  Wall Time      4.9 ms  ±     0.3 ms       4.6 ms  …     5.4 ms           -95.5%  (22.4x faster)
```

Commit.

### Stretch goals

- **A bytecode virtual machine:** a third back end that compiles the tree into a flat list of instructions
  for your own stack machine and runs it in a loop; *Crafting Interpreters* part III shows the idea.
  Add it to the differential test.
- **The `**` operator** from exercise 2 of Lesson 22.22 in all back ends (WebAssembly has no power
  instruction: generate a loop, or call a helper function you emit into every module).
- **Better errors:** show the source line and a `^` under the position, as the Rust compiler does.
- **Generated programs:** with `proptest`, generate random expressions and check that both back ends
  agree (Lesson 09.4). This finds bugs that hand-written programs miss.
- **Binary output:** write `.wasm` instead of WAT with the `wasm-encoder` crate, and run it with the
  `wasmtime` command-line tool.

## Common mistakes

**`return` that only leaves the innermost block.** If `exec` returns `()`, a `return` inside a `while`
inside an `if` has no way to stop the loop and the function. Return a `Flow` value and check it after every
statement.

**Functions that see the caller's variables.** Pushing the parameters as one more scope on the caller's
stack makes the caller's variables visible inside the function (dynamic scoping). The contract says they
are not; the WebAssembly back end cannot see them either, so the differential test fails. Swap in a new
stack for each call.

**Different semantics, hidden by tests that agree with themselves.** Interpreter tests check the
interpreter; they cannot notice that `9223372036854775807 + 1` gives an error in one back end and wraps
in the other. Only the differential test, with a program for every rule of the contract, catches that.

**Locals named after the Pebble variable.** Two `let x` in different blocks of one function would both
become `$x`, and an inner block would overwrite the outer variable. Give each `let` its own local.

## Next steps in compilers and WebAssembly

You have built the whole path from text to machine code. These resources lead further.

**Books**

- *Crafting Interpreters* by Robert Nystrom (free online) — part II is a tree-walking interpreter like
  yours, part III a fast bytecode virtual machine; read it now and compare the designs.
- *Engineering a Compiler* by Keith Cooper and Linda Torczon — a university textbook on optimization,
  intermediate representations and code generation.
- *Types and Programming Languages* by Benjamin Pierce — the theory behind type checkers, for when you want
  more than `int` and `bool`.

**Documentation**

- The *Rust Compiler Development Guide* (rustc-dev-guide) — how `rustc` is organised and how to contribute.
- The WebAssembly specification and the Wasmtime documentation (both from Lesson 22.24).

**Open-source projects to read and contribute to**

| Project | What it is | A good first step |
|---|---|---|
| rustc | the Rust compiler | issues labelled `E-easy`, after reading the rustc-dev-guide |
| rust-analyzer | the Rust language server used by editors | its issues labelled for newcomers; read its architecture document |
| Wasmtime and Cranelift | the runtime from Lesson 22.24 and its code generator | documentation and test improvements |
| wasm-tools | tools to parse, print and validate WebAssembly | add a test or improve an error message |
| Boa | a JavaScript engine written in Rust | the conformance test results show what is missing |
| Rhai | an embedded scripting language for Rust programs | compare its interpreter with yours |
| Gleam | a programming language whose compiler is written in Rust | issues for newcomers in the compiler |
| oxc and Ruff | fast JavaScript and Python tooling (parsers, linters) written in Rust | add a lint rule |

**Communities**

- The Rust Zulip chat (`#t-compiler` and the rust-analyzer streams), where compiler contributors talk.
- The Bytecode Alliance Zulip, for Wasmtime, Cranelift and WASI.
- The r/ProgrammingLanguages community, for language design questions and projects like Pebble.

> **End of the compilers and WebAssembly track.** When you finish the project, continue with
> [Lesson 22.30 — Checkpoint: your specialization](30-checkpoint-your-specialization.md). If you want a second track
> later, the next one starts at [Lesson 22.26 — Cloud-native track: Kubernetes operators](26-cloud-native-track-kubernetes-operators.md).

## Quiz

1. Why does `exec` return a `Flow` value once the language has `return`?

<details>
<summary>Answer</summary>

`return` must stop the current block, every enclosing block and loop, and the function, all at once.
Returning `Flow::Return(value)` lets each level see that and stop, until the call itself receives the value.

</details>

2. What does a static type checker give you that the interpreter's run-time checks do not?

<details>
<summary>Answer</summary>

It finds type errors in *every* part of the program before anything runs, including branches that a
particular run never reaches. It also lets the code generator know the type of each expression (for
`print`).

</details>

3. Why are WebAssembly locals given unique names such as `$x_0` and `$x_1`?

<details>
<summary>Answer</summary>

A WebAssembly function has one flat list of locals, but Pebble allows several variables with the same name
in different scopes (shadowing). Unique names keep them apart.

</details>

4. Why did the contract choose wrapping arithmetic instead of an overflow error?

<details>
<summary>Answer</summary>

Because WebAssembly's `i64.add` wraps, so the compiled back end gets that behaviour for free. An overflow
error would need extra checks around every operation in the generated code. It is a design decision;
what matters is that both back ends do the same thing.

</details>

5. What does a differential test catch that ordinary unit tests miss?

<details>
<summary>Answer</summary>

Disagreements between implementations: cases where each back end passes its own tests but they compute
different results, such as different overflow or error behaviour.

</details>

## Summary

- Functions need call frames (a fresh scope stack per call), a way for `return` to leave nested
  statements (`Flow`), and a depth limit.
- A static type checker walks the tree once, gives each expression a type, and reports all mismatches
  before the program runs.
- Compiling whole programs to WebAssembly means locals with unique names, `block`/`loop`/`br_if` for
  loops, `if` for branches, and host functions for output.
- Write the semantics down; then make every back end follow them and check it with a differential test.
- Next: *Crafting Interpreters* part III, the rustc-dev-guide, and a first contribution to rustc,
  rust-analyzer, Wasmtime or another language project.

## Sources and further reading

- [Crafting Interpreters — A Bytecode Virtual Machine](https://craftinginterpreters.com/a-bytecode-virtual-machine.html) — part III of the book, the natural next step after this project.
- [Rust Compiler Development Guide](https://rustc-dev-guide.rust-lang.org/) — how `rustc` works and how to contribute to it.
- [wasmtime::Linker](https://docs.rs/wasmtime/50/wasmtime/struct.Linker.html) — defining host functions such as `print_int` for a module's imports.
- [WebAssembly Core Specification](https://webassembly.github.io/spec/core/) — the `block`, `loop`, `br_if` and `if` instructions in detail.
- [rust-lang/rust issues labelled E-easy](https://github.com/rust-lang/rust/labels/E-easy), [rust-analyzer](https://github.com/rust-lang/rust-analyzer), [Wasmtime](https://github.com/bytecodealliance/wasmtime), [Cranelift](https://cranelift.dev/), [wasm-tools](https://github.com/bytecodealliance/wasm-tools), [Boa](https://github.com/boa-dev/boa), [Rhai](https://github.com/rhaiscript/rhai), [Gleam](https://github.com/gleam-lang/gleam), [oxc](https://github.com/oxc-project/oxc), [Ruff](https://github.com/astral-sh/ruff) — the projects from the table above.
- [Rust Zulip](https://rust-lang.zulipchat.com/), [Bytecode Alliance Zulip](https://bytecodealliance.zulipchat.com/), [r/ProgrammingLanguages](https://www.reddit.com/r/ProgrammingLanguages/) — the communities.

---

[🏠 Start](../START-HERE.md) · [← 22.24 Compilers track: WebAssembly with Rust](24-compilers-track-webassembly.md) · [📋 Phase 22](README.md) · [22.26 Cloud-native track: Kubernetes operators →](26-cloud-native-track-kubernetes-operators.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
