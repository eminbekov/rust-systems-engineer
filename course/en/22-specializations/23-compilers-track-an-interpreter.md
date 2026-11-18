<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 22.22 Compilers track: lexing and parsing](22-compilers-track-lexing-and-parsing.md) · [📋 Phase 22](README.md) · [22.24 Compilers track: WebAssembly with Rust →](24-compilers-track-webassembly.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/22-specializations/23-compilers-track-an-interpreter.md) <!-- nav -->

# Lesson 22.23 — Compilers track: a tree-walking interpreter

> **Phase 22 — Specializations** · Lesson 23 of 30 · ⏱ about 3 hours (2 sittings)

## What you will learn

- How a tree-walking interpreter runs a program: one recursive function for expressions, one for statements.
- How to represent run-time values, and how to report run-time errors with positions.
- How scopes work: blocks, shadowing and assignment, with a stack of hash maps.
- How to add blocks, `if`/`else` and `while` to the parser from Lesson 22.22.
- How to test an interpreter by running small programs and comparing their output.

## Before you start

- You finished [Lesson 22.22 — Compilers track: lexing and parsing](22-compilers-track-lexing-and-parsing.md)
  and have its `code/p22-pebble` package. The code below starts from the lesson's version; if you did the
  exercises of Lesson 22.22, keep your additions: exercise 2 of this lesson uses the logic operators.
- You know `HashMap` ([Lesson 04.9](../04-structs-enums-collections-errors/09-hash-maps.md)), trait objects
  (`dyn Write`, [Lesson 06.1](../06-rust-advanced/01-trait-objects-and-dynamic-dispatch.md)) and checked
  arithmetic ([Lesson 12.2 — Integers in memory](../12-computer-systems/02-integers-in-memory.md)).

## New words

| Word | Meaning |
|---|---|
| **tree-walking interpreter** | An interpreter that runs a program by visiting the nodes of its syntax tree, computing each node from its children. |
| **evaluate** | Compute the value of an expression. |
| **execute** | Run a statement for its effect (printing, changing a variable). |
| **value** | A piece of data at run time, such as the integer `42` or the boolean `true`. |
| **run-time error** | An error found while the program runs (division by zero), as opposed to a syntax error found by the parser. |
| **scope** | The part of a program where a variable name is visible. In Pebble every block `{ ... }` opens a new scope. |
| **environment** | The interpreter's record of which variables exist and what values they hold, scope by scope. |
| **shadowing** | A `let` in an inner scope creates a new variable with the same name; the outer one is hidden until the block ends. |
| **short-circuit evaluation** | `a && b` does not evaluate `b` if `a` is false, and `a \|\| b` does not if `a` is true. |

## 1. How a tree-walking interpreter works

The syntax tree already says what to compute. The interpreter walks it from the top: to know the value
of `(+ 1 (* 2 3))`, it first needs the values of `1` and `(* 2 3)`; for `(* 2 3)` it needs `2` and `3`,
which are leaves. The values flow back up:

```text
        (+ 1 (* 2 3))      eval(+) = eval(1) + eval(*)  = 1 + 6 = 7
         /        \
        1      (* 2 3)     eval(*) = eval(2) * eval(3)  = 2 * 3 = 6
                /   \
               2     3     leaves: their value is the number itself
```

So `eval` is a recursive function with one `match` arm per kind of expression, and `exec` is another
one with one arm per kind of statement. This is the simplest way to run a language. It is slower than
compiling to bytecode or machine code (Lesson 22.24), because the interpreter re-reads the tree every
time a loop repeats, but it is easy to write, test and change, and many real languages started this way.

Two new ideas come with running code:

- **Values.** Pebble has two kinds: `Value::Int(i64)` and `Value::Bool(bool)`. An operator checks the
  kinds of its operands at run time: `1 + true` is an error.
- **Run-time errors.** Division by zero, an undefined variable or an overflow are not bugs of the
  interpreter, they are errors of the *Pebble program*. The interpreter must report them with a position
  and stop cleanly, never panic.

## 2. Blocks, `if`, `while` and assignment in the parser

A loop needs a variable that changes, so Pebble gets **assignment** (`i = i + 1;`, without `let`), and
three statements that contain other statements. The grammar grows:

```text
statement := "let" IDENT "=" expr ";"
           | IDENT "=" expr ";"
           | "print" expr ";"
           | block
           | "if" expr block ("else" (block | if))?
           | "while" expr block
           | expr ";"
block     := "{" statement* "}"
```

(`?` means "optional".) The lexer gets five new tokens. In `src/lexer.rs`, add `If,`, `Else,` and
`While,` after `Print,` and `LBrace,` and `RBrace,` after `RParen,` in `TokenKind`, the keywords to the
`match name.as_str()`, and the braces to the character `match`:

```rust
"if" => TokenKind::If,
"else" => TokenKind::Else,
"while" => TokenKind::While,
```

```rust
'{' => TokenKind::LBrace,
'}' => TokenKind::RBrace,
```

In `src/ast.rs`, `Stmt` gets four new variants. An `if` always has an else branch in the tree; when the
program has no `else`, the branch is an empty list. The `Display` implementation prints them, using a
small helper `list`:

```rust
#[derive(Debug, Clone, PartialEq)]
pub enum Stmt {
    Let(String, Expr),
    Assign(String, Expr, Pos),
    Print(Expr),
    Expr(Expr),
    Block(Vec<Stmt>),
    If(Expr, Vec<Stmt>, Vec<Stmt>),
    While(Expr, Vec<Stmt>),
}
```

```rust
impl fmt::Display for Stmt {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        match self {
            Stmt::Let(name, e) => write!(f, "(let {name} {e})"),
            Stmt::Assign(name, e, _) => write!(f, "(set {name} {e})"),
            Stmt::Print(e) => write!(f, "(print {e})"),
            Stmt::Expr(e) => write!(f, "{e}"),
            Stmt::Block(body) => write!(f, "(block{})", list(body)),
            Stmt::If(cond, then, otherwise) => {
                write!(
                    f,
                    "(if {cond} (block{}) (block{}))",
                    list(then),
                    list(otherwise)
                )
            }
            Stmt::While(cond, body) => write!(f, "(while {cond} (block{}))", list(body)),
        }
    }
}

/// Each statement with a space in front: " (print 1) (print 2)".
fn list(stmts: &[Stmt]) -> String {
    stmts.iter().map(|s| format!(" {s}")).collect()
}
```

In `src/parser.rs`, replace the old `statement` function with these three functions:

```rust
/// statement := "let" IDENT "=" expr ";" | IDENT "=" expr ";" | "print" expr ";"
///            | block | "if" expr block ("else" (block | if))? | "while" expr block
///            | expr ";"
fn statement(&mut self) -> Result<Stmt, ParseError> {
    let stmt = match self.peek().kind.clone() {
        TokenKind::Let => {
            self.advance();
            let name = match self.advance().kind {
                TokenKind::Ident(name) => name,
                _ => return Err(self.error("expected a variable name after `let`")),
            };
            self.expect(TokenKind::Equal, "`=`")?;
            Stmt::Let(name, self.expr(0)?)
        }
        TokenKind::Ident(name) if self.tokens[self.at + 1].kind == TokenKind::Equal => {
            let pos = self.advance().pos;
            self.advance(); // the `=`
            Stmt::Assign(name, self.expr(0)?, pos)
        }
        TokenKind::Print => {
            self.advance();
            Stmt::Print(self.expr(0)?)
        }
        // These three end with `}`, not with `;`.
        TokenKind::LBrace => return Ok(Stmt::Block(self.block()?)),
        TokenKind::If => return self.if_statement(),
        TokenKind::While => {
            self.advance();
            let cond = self.expr(0)?;
            return Ok(Stmt::While(cond, self.block()?));
        }
        _ => Stmt::Expr(self.expr(0)?),
    };
    self.expect(TokenKind::Semicolon, "`;`")?;
    Ok(stmt)
}

/// block := "{" statement* "}"
fn block(&mut self) -> Result<Vec<Stmt>, ParseError> {
    self.expect(TokenKind::LBrace, "`{`")?;
    let mut stmts = Vec::new();
    while !matches!(self.peek().kind, TokenKind::RBrace | TokenKind::Eof) {
        stmts.push(self.statement()?);
    }
    self.expect(TokenKind::RBrace, "`}`")?;
    Ok(stmts)
}

fn if_statement(&mut self) -> Result<Stmt, ParseError> {
    self.expect(TokenKind::If, "`if`")?;
    let cond = self.expr(0)?;
    let then = self.block()?;
    let otherwise = if self.peek().kind != TokenKind::Else {
        Vec::new()
    } else {
        self.advance();
        if self.peek().kind == TokenKind::If {
            vec![self.if_statement()?] // `else if` is an `if` inside the else branch
        } else {
            self.block()?
        }
    };
    Ok(Stmt::If(cond, then, otherwise))
}
```

**What happens:**

1. `statement` now matches on a *clone* of the token kind, because the `Ident(name)` arm takes the name
   out, and the guard `if self.tokens[self.at + 1].kind == TokenKind::Equal` looks one token further: a
   name followed by `=` is an assignment, a name followed by anything else starts an expression. (The
   guard is safe: an `Ident` is never the last token, because `Eof` always follows.)
2. Blocks, `if` and `while` end with `}`, so they `return` early instead of expecting `;`.
3. `block` collects statements until `}`. Checking `Eof` too prevents an endless loop when the `}` is
   missing; `expect` then reports the error.
4. `if_statement` handles `else if` by parsing the inner `if` as the only statement of the else branch.
   That is why `if_statement` is its own function: it calls itself.

Add a test to the parser's `tests` module:

```rust
#[test]
fn parses_blocks_if_and_while() {
    let program = parse("while i < 3 { i = i + 1; } if x { print 1; } else if y { print 2; }");
    let shown: Vec<String> = program.unwrap().iter().map(|s| s.to_string()).collect();
    assert_eq!(
        shown,
        [
            "(while (< i 3) (block (set i (+ i 1))))",
            "(if x (block (print 1)) (block (if y (block (print 2)) (block))))",
        ]
    );
}
```

## 3. The interpreter

Save as `src/interp.rs` and add `pub mod interp;` to `src/lib.rs`:

```rust
//! A tree-walking interpreter: runs a program by walking its syntax tree.

use crate::ast::{BinOp, Expr, Stmt, UnOp};
use crate::lexer::Pos;
use std::collections::HashMap;
use std::fmt;
use std::io::Write;

#[derive(Debug, Clone, Copy, PartialEq)]
pub enum Value {
    Int(i64),
    Bool(bool),
}

impl fmt::Display for Value {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        match self {
            Value::Int(n) => write!(f, "{n}"),
            Value::Bool(b) => write!(f, "{b}"),
        }
    }
}

#[derive(Debug, PartialEq)]
pub struct RuntimeError {
    pub message: String,
    pub pos: Option<Pos>,
}

fn error(message: String, pos: Pos) -> RuntimeError {
    RuntimeError {
        message,
        pos: Some(pos),
    }
}

pub struct Interpreter<'w> {
    /// One map per open block; the last one is the innermost scope.
    scopes: Vec<HashMap<String, Value>>,
    out: &'w mut dyn Write,
}

impl<'w> Interpreter<'w> {
    pub fn new(out: &'w mut dyn Write) -> Interpreter<'w> {
        Interpreter {
            scopes: vec![HashMap::new()],
            out,
        }
    }

    pub fn run(&mut self, program: &[Stmt]) -> Result<(), RuntimeError> {
        for stmt in program {
            self.exec(stmt)?;
        }
        Ok(())
    }

    fn exec(&mut self, stmt: &Stmt) -> Result<(), RuntimeError> {
        match stmt {
            Stmt::Let(name, value) => {
                let value = self.eval(value)?;
                self.scopes.last_mut().unwrap().insert(name.clone(), value);
            }
            Stmt::Assign(name, value, pos) => {
                let value = self.eval(value)?;
                // Find the innermost scope that already has this variable.
                match self.scopes.iter_mut().rev().find_map(|s| s.get_mut(name)) {
                    Some(slot) => *slot = value,
                    None => {
                        return Err(error(
                            format!("assignment to undefined variable `{name}`"),
                            *pos,
                        ));
                    }
                }
            }
            Stmt::Print(e) => {
                let value = self.eval(e)?;
                writeln!(self.out, "{value}").map_err(|e| RuntimeError {
                    message: format!("cannot write output: {e}"),
                    pos: None,
                })?;
            }
            Stmt::Expr(e) => {
                self.eval(e)?;
            }
            Stmt::Block(body) => self.exec_block(body)?,
            Stmt::If(cond, then, otherwise) => {
                if self.condition(cond)? {
                    self.exec_block(then)?;
                } else {
                    self.exec_block(otherwise)?;
                }
            }
            Stmt::While(cond, body) => {
                while self.condition(cond)? {
                    self.exec_block(body)?;
                }
            }
        }
        Ok(())
    }

    /// Runs statements in a new scope, which is removed afterwards (even after an error).
    fn exec_block(&mut self, body: &[Stmt]) -> Result<(), RuntimeError> {
        self.scopes.push(HashMap::new());
        let result = body.iter().try_for_each(|stmt| self.exec(stmt));
        self.scopes.pop();
        result
    }

    fn condition(&mut self, cond: &Expr) -> Result<bool, RuntimeError> {
        match self.eval(cond)? {
            Value::Bool(b) => Ok(b),
            Value::Int(_) => Err(RuntimeError {
                message: "condition must be a bool".into(),
                pos: pos_of(cond),
            }),
        }
    }

    pub fn eval(&mut self, expr: &Expr) -> Result<Value, RuntimeError> {
        match expr {
            Expr::Int(n) => Ok(Value::Int(*n)),
            Expr::Bool(b) => Ok(Value::Bool(*b)),
            Expr::Var(name, pos) => self
                .scopes
                .iter()
                .rev()
                .find_map(|s| s.get(name).copied())
                .ok_or_else(|| error(format!("undefined variable `{name}`"), *pos)),
            Expr::Unary(UnOp::Neg, e, pos) => match self.eval(e)? {
                Value::Int(n) => n
                    .checked_neg()
                    .map(Value::Int)
                    .ok_or_else(|| error("integer overflow".into(), *pos)),
                Value::Bool(_) => Err(error("cannot negate a bool".into(), *pos)),
            },
            Expr::Binary(op, left, right, pos) => {
                let l = self.eval(left)?;
                let r = self.eval(right)?;
                binary(*op, l, r, *pos)
            }
        }
    }
}

fn binary(op: BinOp, l: Value, r: Value, pos: Pos) -> Result<Value, RuntimeError> {
    use Value::{Bool, Int};
    let overflow = || error("integer overflow".into(), pos);
    match (op, l, r) {
        (BinOp::Add, Int(a), Int(b)) => a.checked_add(b).map(Int).ok_or_else(overflow),
        (BinOp::Sub, Int(a), Int(b)) => a.checked_sub(b).map(Int).ok_or_else(overflow),
        (BinOp::Mul, Int(a), Int(b)) => a.checked_mul(b).map(Int).ok_or_else(overflow),
        (BinOp::Div | BinOp::Rem, Int(_), Int(0)) => Err(error("division by zero".into(), pos)),
        (BinOp::Div, Int(a), Int(b)) => a.checked_div(b).map(Int).ok_or_else(overflow),
        (BinOp::Rem, Int(a), Int(b)) => a.checked_rem(b).map(Int).ok_or_else(overflow),
        (BinOp::Lt, Int(a), Int(b)) => Ok(Bool(a < b)),
        (BinOp::Le, Int(a), Int(b)) => Ok(Bool(a <= b)),
        (BinOp::Gt, Int(a), Int(b)) => Ok(Bool(a > b)),
        (BinOp::Ge, Int(a), Int(b)) => Ok(Bool(a >= b)),
        (BinOp::Eq, Int(a), Int(b)) => Ok(Bool(a == b)),
        (BinOp::Ne, Int(a), Int(b)) => Ok(Bool(a != b)),
        (BinOp::Eq, Bool(a), Bool(b)) => Ok(Bool(a == b)),
        (BinOp::Ne, Bool(a), Bool(b)) => Ok(Bool(a != b)),
        (op, l, r) => Err(error(
            format!(
                "cannot apply `{}` to {} and {}",
                op.symbol(),
                type_name(l),
                type_name(r)
            ),
            pos,
        )),
    }
}

fn type_name(v: Value) -> &'static str {
    match v {
        Value::Int(_) => "int",
        Value::Bool(_) => "bool",
    }
}

/// The position to blame for an expression (literals do not store one).
fn pos_of(expr: &Expr) -> Option<Pos> {
    match expr {
        Expr::Var(_, pos) | Expr::Unary(_, _, pos) | Expr::Binary(_, _, _, pos) => Some(*pos),
        Expr::Int(_) | Expr::Bool(_) => None,
    }
}

/// Parses and runs `src`, and returns everything it printed, or the first error as text.
pub fn run_source(src: &str) -> Result<String, String> {
    let program = crate::parser::parse(src)?;
    let mut out = Vec::new();
    Interpreter::new(&mut out)
        .run(&program)
        .map_err(|e| match e.pos {
            Some(p) => format!("runtime error at {}:{}: {}", p.line, p.col, e.message),
            None => format!("runtime error: {}", e.message),
        })?;
    Ok(String::from_utf8(out).expect("the interpreter prints UTF-8"))
}

#[cfg(test)]
mod tests {
    use super::run_source;

    #[test]
    fn arithmetic_and_variables() {
        assert_eq!(run_source("let x = 6; print x * 7;").unwrap(), "42\n");
    }

    #[test]
    fn while_loop_and_assignment() {
        let src = "let i = 0; let sum = 0; while i < 5 { i = i + 1; sum = sum + i; } print sum;";
        assert_eq!(run_source(src).unwrap(), "15\n");
    }

    #[test]
    fn inner_let_shadows_and_disappears() {
        let src = "let x = 1; { let x = 2; print x; } print x;";
        assert_eq!(run_source(src).unwrap(), "2\n1\n");
    }

    #[test]
    fn runtime_errors_have_positions() {
        assert_eq!(
            run_source("print 1 / 0;"),
            Err("runtime error at 1:9: division by zero".into())
        );
        assert_eq!(
            run_source("print 1 + true;"),
            Err("runtime error at 1:9: cannot apply `+` to int and bool".into())
        );
        assert_eq!(
            run_source("y = 3;"),
            Err("runtime error at 1:1: assignment to undefined variable `y`".into())
        );
    }
}
```

**What happens:**

1. `Value` is `Copy`: an integer or a boolean is small, so values are copied freely. Its `Display` is what
   `print` shows.
2. `RuntimeError` has an optional position: errors in literals or in writing the output have none.
3. The `Interpreter` holds the **environment**: `scopes`, a stack (a `Vec`) of hash maps from names to
   values. It writes output to `out`, a `&mut dyn Write`, so `main` can pass standard output and tests can
   pass a `Vec<u8>` (Lesson 09.3: a test double).
4. `exec` has one arm per statement. `Let` inserts into the **innermost** scope. `Assign` searches the
   scopes from the innermost outwards and changes the first variable with that name; if there is none, it
   is an error, because only `let` creates variables.
5. `exec_block` pushes a new scope, runs the statements, and pops the scope **even if a statement failed**:
   `try_for_each` stops at the first error, but the `pop` comes after it in any case.
6. `condition` evaluates the condition of `if` and `while` and insists on a bool: `if 1 { ... }` is an
   error, not "true".
7. `eval` has one arm per expression. A variable is looked up from the innermost scope outwards, so an
   inner `let x` hides an outer `x` (**shadowing**).
8. `binary` matches the operator *and both values* in one tuple. Every arithmetic operation uses the
   `checked_*` methods, which return `None` on overflow instead of panicking (in debug builds) or wrapping
   around (in release builds). Division by zero has its own arm, before the general division arm. The last
   arm catches every wrong combination, such as `1 + true`.
9. `run_source` parses and runs a whole program and returns its output as a `String`. The tests use it to
   check a program's output in one line.

> ⚠️ **Common mistake:** if you kept `UnOp::Not` from exercise 3 of Lesson 22.22, the `match` in `eval` does not cover
> `Expr::Unary(UnOp::Not, ..)`, and the build stops with error `E0004` ("non-exhaustive patterns"). Exercise 2 below
> adds the real arm; until then add a temporary one:
> `` Expr::Unary(UnOp::Not, _, pos) => Err(error("`!` is not supported yet".into(), *pos)), ``
> (`&&` and `||` need nothing yet: they reach the last arm of `binary` and give a run-time error.)

The scopes in action, for `let x = 1; { let x = 2; print x; } print x;`:

```text
 before the block      inside the block           after the block
 ┌─────────┐           ┌─────────┐ ┌─────────┐    ┌─────────┐
 │ x = 1   │           │ x = 1   │ │ x = 2   │    │ x = 1   │
 └─────────┘           └─────────┘ └─────────┘    └─────────┘
  scopes[0]             scopes[0]   scopes[1]      scopes[0]
                        print x → finds scopes[1] first → 2
```

Finally, `src/main.rs` runs a file instead of printing its tree:

```rust
use std::process::ExitCode;

fn main() -> ExitCode {
    let Some(path) = std::env::args().nth(1) else {
        eprintln!("usage: p22-pebble <file.pb>");
        return ExitCode::FAILURE;
    };
    let src = match std::fs::read_to_string(&path) {
        Ok(src) => src,
        Err(e) => {
            eprintln!("cannot read {path}: {e}");
            return ExitCode::FAILURE;
        }
    };
    let program = match p22_pebble::parser::parse(&src) {
        Ok(program) => program,
        Err(message) => {
            eprintln!("{path}: {message}");
            return ExitCode::FAILURE;
        }
    };
    let mut stdout = std::io::stdout();
    let mut interpreter = p22_pebble::interp::Interpreter::new(&mut stdout);
    match interpreter.run(&program) {
        Ok(()) => ExitCode::SUCCESS,
        Err(e) => {
            match e.pos {
                Some(p) => eprintln!("{path}:{}:{}: runtime error: {}", p.line, p.col, e.message),
                None => eprintln!("{path}: runtime error: {}", e.message),
            }
            ExitCode::FAILURE
        }
    }
}
```

Save this program as `programs/loops.pb`:

```text
// The greatest common divisor of two numbers (Euclid's algorithm).
let a = 1071;
let b = 462;
while b != 0 {
    let t = b;
    b = a % b;
    a = t;
}
print a;

// The first five powers of two.
let n = 1;
let i = 0;
while i < 5 {
    print n;
    n = n * 2;
    i = i + 1;
}

if a > 20 {
    print true;
} else {
    print false;
}
```

`cargo run -- programs/loops.pb`:

```text
21
1
2
4
8
16
true
```

And a program with a bug, `programs/oops.pb`:

```text
let total = 10;
let count = 0;
print total / count;
```

`cargo run -- programs/oops.pb` prints the error with the position of the `/` and exits with code 1:

```text
programs/oops.pb:3:13: runtime error: division by zero
```

`cargo test` runs the parser tests and the interpreter tests:

```text
test interp::tests::inner_let_shadows_and_disappears ... ok
test interp::tests::arithmetic_and_variables ... ok
test parser::tests::multiplication_binds_tighter_than_addition ... ok
test interp::tests::runtime_errors_have_positions ... ok
test interp::tests::while_loop_and_assignment ... ok
test parser::tests::reports_the_position_of_errors ... ok
test parser::tests::prefix_minus_and_comparisons ... ok
test parser::tests::same_power_operators_group_to_the_left ... ok
test parser::tests::parses_blocks_if_and_while ... ok

test result: ok. 9 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

> 🧪 **Testing:** The best tests for an interpreter are small programs with their expected output, like
> `run_source("let x = 6; print x * 7;") == "42\n"`. Write one for every feature and every error message.
> Later, keep a folder of `.pb` programs with `.out` files and a test that runs them all: this is called a
> **golden test** suite, and every real language has one.

> 🔒 **Security:** An interpreter often runs code that someone else wrote: plugins, configuration
> scripts, rules typed by users. Such code must not crash the host (no panics: use checked arithmetic and
> return errors), and it must not run forever or use unlimited memory. Exercise 3 adds a step limit, the
> first defence. WebAssembly (Lesson 22.24) offers the same kind of sandbox for compiled code.

## Common mistakes

**Using Rust's `+` for the language's `+`.** Change the `Add` arm in `binary` to
`(BinOp::Add, Int(a), Int(b)) => Ok(Int(a + b)),` and run `print 9223372036854775807 + 1;`:

```text
thread 'main' (6413393) panicked at src/interp.rs:152:48:
attempt to add with overflow
```

The *interpreter* crashed because of a mistake in the *Pebble* program. With `checked_add` the same program
gives a proper error: `runtime error: integer overflow` with the position `1:27`.

**`Assign` that inserts into the innermost scope.** If assignment simply did
`self.scopes.last_mut().unwrap().insert(...)`, then `i = i + 1;` inside a `while` body would create a new
`i` in the body's scope, which disappears at the end of each round. The loop's `i` never changes and the
loop never ends. Assignment must find the scope where the variable already lives.

**Forgetting to pop a scope after an error.** Writing `for stmt in body { self.exec(stmt)?; }` and then
`self.scopes.pop();` leaves the scope on the stack when `?` returns early. In this interpreter an error
ends the program anyway, but once you add error handling or functions (Lesson 22.25), the stale scope
makes variables visible where they should not be. Pop in every case, as `exec_block` does.

**Truthiness by accident.** Treating any non-zero integer as `true` is a language design decision, not
something to get by accident. Pebble requires a `bool` in conditions, and `condition` says so.

## Try it yourself

1. Run a program that prints a variable defined inside a block after the block ends:
   `let x = 5; { let y = x + 1; } print y;`. The error is `undefined variable `y`` at the position of `y`
   in the last `print`.
2. Write `programs/countdown.pb` that prints 5, 4, 3, 2, 1 with a `while` loop.
3. Try `if 1 { print 2; }`. The error is `condition must be a bool`, without a position, because the
   literal `1` stores none. How would you add positions to literals?

## Exercises

### Exercise 1 — Test the `else if` chain (easy)

Write a test that runs one program three times with `n` set to 30, 10 and 7. The program prints `15` if
`n` is divisible by 15, else `5` if it is divisible by 5, else `0`.

<details>
<summary>Hint</summary>

Build the source with `format!` inside a closure that takes `n`. In `format!`, write `{{` and `}}` for the
braces of Pebble blocks, because single braces are placeholders.

</details>

<details>
<summary>Answer</summary>

In the `tests` module of `src/interp.rs`:

```rust
#[test]
fn else_if_chooses_the_first_true_branch() {
    let classify = |n: i64| {
        let src = format!(
            "let n = {n}; if n % 15 == 0 {{ print 15; }} else if n % 5 == 0 {{ print 5; }} else {{ print 0; }}"
        );
        run_source(&src).unwrap()
    };
    assert_eq!(classify(30), "15\n");
    assert_eq!(classify(10), "5\n");
    assert_eq!(classify(7), "0\n");
}
```

```text
test interp::tests::else_if_chooses_the_first_true_branch ... ok
```

Why it works: 30 is divisible by 15, so the first branch runs and the `else` part is skipped; 10 fails the
first test and passes the second; 7 reaches the final `else`.

</details>

### Exercise 2 — `&&`, `||` and `!` with short-circuit (medium)

If you added the logic operators to the parser (exercise 3 of Lesson 22.22), make them work in the
interpreter. `&&` and `||` must **short-circuit**: `false && 1 / 0 == 0` is `false` and does not divide by
zero. Both operands and the operand of `!` must be bools. Test it.

<details>
<summary>Hint</summary>

`&&` and `||` cannot go through `binary`, because `binary` receives two values that are already
evaluated. Add an arm before the general `Expr::Binary` arm with the pattern
`Expr::Binary(op @ (BinOp::And | BinOp::Or), left, right, pos)`: evaluate `left`, decide, and evaluate
`right` only when needed.

</details>

<details>
<summary>Answer</summary>

In `eval`, after the `UnOp::Neg` arm:

```rust
Expr::Unary(UnOp::Not, e, pos) => match self.eval(e)? {
    Value::Bool(b) => Ok(Value::Bool(!b)),
    Value::Int(_) => Err(error("`!` needs a bool".into(), *pos)),
},
Expr::Binary(op @ (BinOp::And | BinOp::Or), left, right, pos) => {
    let needs_bool = || error(format!("`{}` needs bools", op.symbol()), *pos);
    let Value::Bool(l) = self.eval(left)? else {
        return Err(needs_bool());
    };
    // Short-circuit: the right side runs only if it can change the answer.
    if (*op == BinOp::And && !l) || (*op == BinOp::Or && l) {
        return Ok(Value::Bool(l));
    }
    match self.eval(right)? {
        Value::Bool(r) => Ok(Value::Bool(r)),
        Value::Int(_) => Err(needs_bool()),
    }
}
```

and the test:

```rust
#[test]
fn logic_operators_short_circuit() {
    let src = "print false && 1 / 0 == 0; print true || 1 / 0 == 0; print !(1 < 2);";
    assert_eq!(run_source(src).unwrap(), "false\ntrue\nfalse\n");
    assert_eq!(
        run_source("print 1 && true;"),
        Err("runtime error at 1:9: `&&` needs bools".into())
    );
}
```

```text
test interp::tests::logic_operators_short_circuit ... ok
```

Why it works: the arm with the `@` binding (Lesson 06.3) matches only `And` and `Or` and comes before the
general arm, so these operators never reach `binary`. The `let ... else` takes the bool out of the left
value or returns the error.

</details>

### Exercise 3 — A step limit (hard)

Add `pub fn with_step_limit(out: &'w mut dyn Write, limit: u64) -> Interpreter<'w>`. Every executed
statement costs one step; when the budget is used up, `run` returns a `RuntimeError` with the message
`step limit reached`. `new` keeps working without a limit. Make sure that `while true { }`, with an
**empty** body, is also stopped. Write a test with an endless loop and a limit of 10,000.

<details>
<summary>Hint</summary>

Add a field `steps_left: u64` and a method `tick(&mut self) -> Result<(), RuntimeError>` that fails at
zero and otherwise subtracts one. Call it at the start of `exec`. An empty loop body executes no
statements, so also call `tick` once per round of the `while` loop.

</details>

<details>
<summary>Answer</summary>

Add `steps_left: u64,` to the struct, and replace `new` with:

```rust
pub fn new(out: &'w mut dyn Write) -> Interpreter<'w> {
    Interpreter::with_step_limit(out, u64::MAX)
}

/// An interpreter that stops with an error after `limit` steps.
pub fn with_step_limit(out: &'w mut dyn Write, limit: u64) -> Interpreter<'w> {
    Interpreter {
        scopes: vec![HashMap::new()],
        out,
        steps_left: limit,
    }
}

/// Counts one step; fails when the budget is used up.
fn tick(&mut self) -> Result<(), RuntimeError> {
    if self.steps_left == 0 {
        return Err(RuntimeError {
            message: "step limit reached".into(),
            pos: None,
        });
    }
    self.steps_left -= 1;
    Ok(())
}
```

Call `self.tick()?;` as the first line of `exec`, and in the `While` arm:

```rust
while self.condition(cond)? {
    self.tick()?; // also counts loops with an empty body
    self.exec_block(body)?;
}
```

The test:

```rust
#[test]
fn an_endless_loop_hits_the_step_limit() {
    let program = crate::parser::parse("let i = 0; while true { i = i + 1; }").unwrap();
    let mut out = Vec::new();
    let result = super::Interpreter::with_step_limit(&mut out, 10_000).run(&program);
    assert_eq!(result.unwrap_err().message, "step limit reached");
}
```

```text
test interp::tests::an_endless_loop_hits_the_step_limit ... ok
```

Why it works: every path through the program that can repeat passes through `tick`, so the number of
steps is bounded, and the host gets control back with an error instead of hanging. Real sandboxes use the
same idea and call it **fuel** (Wasmtime has a fuel setting for WebAssembly).

</details>

## Quiz

1. What is the difference between `eval` and `exec`?

<details>
<summary>Answer</summary>

`eval` computes the value of an expression and returns a `Value`. `exec` runs a statement for its effect
(print, create or change a variable, run a loop) and returns nothing but a possible error.

</details>

2. Why does the interpreter use `checked_add` instead of `+`?

<details>
<summary>Answer</summary>

An overflow in a Pebble program is an error of that program. With `+`, the interpreter itself would panic
(debug build) or silently wrap around (release build). `checked_add` returns `None`, which becomes a
run-time error with a position.

</details>

3. How does the environment implement scopes, and what happens when a block ends?

<details>
<summary>Answer</summary>

It is a stack of hash maps. A block pushes a new empty map; `let` inserts into the top map; lookups search
from the top down. When the block ends, its map is popped, so its variables disappear.

</details>

4. What is the difference between `let x = 2;` and `x = 2;` inside a block, if `x` exists outside?

<details>
<summary>Answer</summary>

`let x = 2;` creates a new `x` in the block's scope that shadows the outer one; after the block the outer
`x` still has its old value. `x = 2;` changes the outer `x`.

</details>

5. Why do `&&` and `||` need special handling instead of a normal arm in `binary`?

<details>
<summary>Answer</summary>

`binary` receives both operands already evaluated. Short-circuit evaluation must decide *whether* to
evaluate the right operand after seeing the left one, so it has to be done in `eval`.

</details>

## Summary

- A tree-walking interpreter evaluates expressions with one recursive `match` and executes statements with
  another; values flow up the tree.
- Run-time values carry their kind (`Int`, `Bool`); operators check the kinds and return errors, never
  panic.
- Run-time errors belong to the user's program: report them with a position and stop cleanly.
- The environment is a stack of scopes: `let` adds to the innermost, lookups and assignment search outward,
  and a block pops its scope in every case.
- Blocks, `if`/`else if`/`else` and `while` are a few more recursive-descent functions in the parser.
- Test an interpreter with small programs and their expected output; protect hosts with checked arithmetic
  and step limits.

## Sources and further reading

- [Crafting Interpreters — Evaluating Expressions](https://craftinginterpreters.com/evaluating-expressions.html) — the same idea in Robert Nystrom's book, with a different language.
- [Crafting Interpreters — Statements and State](https://craftinginterpreters.com/statements-and-state.html) — variables, assignment and environments.
- [Crafting Interpreters — Control Flow](https://craftinginterpreters.com/control-flow.html) — `if`, `while` and short-circuit logic.
- [std::primitive::i64 — checked_add and friends](https://doc.rust-lang.org/std/primitive.i64.html) — the checked, wrapping and saturating arithmetic methods.
- [The Rust Programming Language — 8.3 Hash Maps](https://doc.rust-lang.org/book/ch08-03-hash-maps.html) — a refresher on the map behind every scope.

---

[🏠 Start](../START-HERE.md) · [← 22.22 Compilers track: lexing and parsing](22-compilers-track-lexing-and-parsing.md) · [📋 Phase 22](README.md) · [22.24 Compilers track: WebAssembly with Rust →](24-compilers-track-webassembly.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
