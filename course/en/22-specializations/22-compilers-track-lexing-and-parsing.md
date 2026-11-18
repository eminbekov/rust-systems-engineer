<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 22.21 Networking track: project and next steps](21-networking-track-project-and-next-steps.md) · [📋 Phase 22](README.md) · [22.23 Compilers track: a tree-walking interpreter →](23-compilers-track-an-interpreter.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/22-specializations/22-compilers-track-lexing-and-parsing.md) <!-- nav -->

# Lesson 22.22 — Compilers track: lexing and parsing

> **Phase 22 — Specializations** · Lesson 22 of 30 · ⏱ about 4 hours (2 sittings)

> **Not your track?** This lesson starts the compilers and WebAssembly track (Lessons 22.22–22.25). If
> you chose another track, skip to [Lesson 22.26 — Cloud-native track: Kubernetes operators](26-cloud-native-track-kubernetes-operators.md)
> or go straight to [Lesson 22.30 — Checkpoint: your specialization](30-checkpoint-your-specialization.md).

## What you will learn

- What compilers and interpreters do, as a pipeline of small steps, and what this track builds.
- How a lexer turns text into tokens, with line and column positions for error messages.
- How a syntax tree (AST) represents a program, and how to print it to check your parser.
- How recursive descent parses statements and how Pratt parsing handles operator precedence and associativity.
- How to test a parser through a readable printed form of its tree.

## Before you start

- You finished the earlier phases. This track uses enums and pattern matching
  ([Lesson 06.3 — Patterns in depth](../06-rust-advanced/03-patterns-in-depth.md)), recursion
  ([Lesson 07.7 — Recursion](../07-algorithms-and-data-structures/07-recursion.md)), trees
  ([Lesson 07.11](../07-algorithms-and-data-structures/11-trees-and-binary-search-trees.md)) and `Box`
  ([Lesson 05.9 — Smart pointers](../05-rust-intermediate/09-smart-pointers.md)).
- The track's code lives in one package, a library with a small program:

```bash
cd ~/projects/rust
cargo new code/p22-pebble
cd code/p22-pebble
mkdir programs
```

The files are `src/lexer.rs`, `src/ast.rs` and `src/parser.rs`, declared in `src/lib.rs`:

```rust
pub mod ast;
pub mod lexer;
pub mod parser;
```

## New words

| Word | Meaning |
|---|---|
| **compiler** | A program that translates a program from one language into another, usually into machine code or bytecode, without running it. |
| **interpreter** | A program that runs a program directly, step by step, from its source or its syntax tree. |
| **lexer** (tokenizer) | The first step: it cuts the source text into **tokens**, the "words" of the language. |
| **token** | One meaningful piece of source text: a number, a name, a keyword, an operator, a bracket. |
| **parser** | The step that checks the order of tokens against the grammar and builds a syntax tree. |
| **AST** (Abstract Syntax Tree) | A tree that shows the structure of a program (which operator applies to what), without punctuation such as brackets and semicolons. |
| **grammar** | The rules that say which sequences of tokens form valid programs. |
| **recursive descent** | A parsing method with one function per grammar rule; the functions call each other (and themselves) as the rules do. |
| **precedence** | Which operator binds tighter: in `1 + 2 * 3`, `*` has higher precedence than `+`. |
| **associativity** | How operators of the same precedence group: `10 - 3 - 2` is `(10 - 3) - 2`, so `-` is *left-associative*. |
| **Pratt parsing** | A way to parse expressions in which every operator has a number, its **binding power**; one loop then handles all precedence and associativity. |
| **S-expression** | A written form of a tree with brackets and the operator first: `(+ 1 (* 2 3))`. |
| **keyword** | A word with a fixed meaning that cannot be a variable name, such as `let` or `print`. |

## 1. What a compiler does

A compiler or an interpreter is not one big magic program. It is a **pipeline** of small steps, and each
step turns one data structure into the next:

```text
 source text          tokens                  syntax tree (AST)            result
 "print 1 + 2 * 3;" → [Print, Int(1), Plus, → (print (+ 1 (* 2 3)))  ──┬─▶ interpreter: prints 7
                       Int(2), Star, Int(3),                           │    (Lesson 22.23)
                       Semicolon, Eof]                                 └─▶ code generator: a
      lexer ─────────────────▶   parser ──────────────────▶                WebAssembly module
                                                                           (Lesson 22.24)
```

The **lexer** knows about characters: where a number ends, that `>=` is one operator, that spaces and
comments mean nothing. The **parser** knows about structure: that `*` binds tighter than `+`, that a
`let` needs a name and an `=`. Later steps never look at characters again; they work on the tree.

In this track you build a small language called **Pebble** in four lessons:

| Lesson | Step |
|---|---|
| 22.22 (this one) | lexer and parser for expressions, `let` and `print` |
| 22.23 | a tree-walking interpreter, plus blocks, `if` and `while` |
| 22.24 | WebAssembly: run Rust in a sandbox, and compile Pebble expressions to WebAssembly |
| 22.25 | project: functions, a complete interpreter and a compiler, tested against each other |

A Pebble program looks like this. Save it as `programs/first.pb`:

```text
// Our first Pebble program.
let width = 6;
let height = 7;
print width * height + 1;
print -(width - 10) >= 4 == true;
```

Pebble has 64-bit integers and booleans (`true`, `false`), the arithmetic operators `+ - * / %`, the
comparisons `== != < <= > >=`, a prefix minus, and comments that start with `//`. You will add `**` and
the logic operators `&& || !` in the exercises.

> 💡 **Tip:** The free online book *Crafting Interpreters* by Robert Nystrom builds two complete
> interpreters step by step. It is the best companion to this track. Pebble is smaller and designed for
> this course, so you can read the book afterwards without repeating yourself.

## 2. The lexer

The lexer walks through the characters once. At each position it decides what kind of token starts
there, takes all its characters, and remembers the **position** (line and column) where the token
started, so that later error messages can say *where* something is wrong. Save as `src/lexer.rs`:

```rust
//! The lexer: turns source text into a list of tokens.

use std::iter::Peekable;
use std::str::Chars;

/// A position in the source: line and column, both starting at 1.
#[derive(Debug, Clone, Copy, PartialEq)]
pub struct Pos {
    pub line: u32,
    pub col: u32,
}

#[derive(Debug, Clone, PartialEq)]
pub enum TokenKind {
    Int(i64),
    Ident(String),
    Let,
    Print,
    True,
    False,
    Plus,
    Minus,
    Star,
    Slash,
    Percent,
    Equal,
    EqualEqual,
    BangEqual,
    Less,
    LessEqual,
    Greater,
    GreaterEqual,
    LParen,
    RParen,
    Semicolon,
    Eof,
}

#[derive(Debug, Clone, PartialEq)]
pub struct Token {
    pub kind: TokenKind,
    pub pos: Pos,
}

#[derive(Debug, PartialEq)]
pub struct LexError {
    pub message: String,
    pub pos: Pos,
}

struct Lexer<'a> {
    chars: Peekable<Chars<'a>>,
    pos: Pos,
}

impl Lexer<'_> {
    fn peek(&mut self) -> Option<char> {
        self.chars.peek().copied()
    }

    /// Takes one character and moves the position forward.
    fn bump(&mut self) -> Option<char> {
        let c = self.chars.next()?;
        if c == '\n' {
            self.pos.line += 1;
            self.pos.col = 1;
        } else {
            self.pos.col += 1;
        }
        Some(c)
    }

    /// Takes the next character only if it is `expected`.
    fn bump_if(&mut self, expected: char) -> bool {
        if self.peek() == Some(expected) {
            self.bump();
            true
        } else {
            false
        }
    }

    /// Takes characters while `keep` is true and returns them.
    fn take_while(&mut self, keep: impl Fn(char) -> bool) -> String {
        let mut text = String::new();
        while let Some(c) = self.peek().filter(|&c| keep(c)) {
            text.push(c);
            self.bump();
        }
        text
    }
}

pub fn lex(src: &str) -> Result<Vec<Token>, LexError> {
    let mut lx = Lexer {
        chars: src.chars().peekable(),
        pos: Pos { line: 1, col: 1 },
    };
    let mut tokens = Vec::new();
    while let Some(c) = lx.peek() {
        let start = lx.pos;
        let kind = if c.is_whitespace() {
            lx.bump();
            continue;
        } else if c.is_ascii_digit() {
            let text = lx.take_while(|c| c.is_ascii_digit());
            let value = text.parse().map_err(|_| LexError {
                message: format!("number too large: {text}"),
                pos: start,
            })?;
            TokenKind::Int(value)
        } else if c.is_ascii_alphabetic() || c == '_' {
            let name = lx.take_while(|c| c.is_ascii_alphanumeric() || c == '_');
            match name.as_str() {
                "let" => TokenKind::Let,
                "print" => TokenKind::Print,
                "true" => TokenKind::True,
                "false" => TokenKind::False,
                _ => TokenKind::Ident(name),
            }
        } else {
            lx.bump();
            match c {
                '+' => TokenKind::Plus,
                '-' => TokenKind::Minus,
                '*' => TokenKind::Star,
                '%' => TokenKind::Percent,
                '(' => TokenKind::LParen,
                ')' => TokenKind::RParen,
                ';' => TokenKind::Semicolon,
                '/' if lx.bump_if('/') => {
                    lx.take_while(|c| c != '\n'); // a comment: skip the rest of the line
                    continue;
                }
                '/' => TokenKind::Slash,
                '=' if lx.bump_if('=') => TokenKind::EqualEqual,
                '=' => TokenKind::Equal,
                '!' if lx.bump_if('=') => TokenKind::BangEqual,
                '<' if lx.bump_if('=') => TokenKind::LessEqual,
                '<' => TokenKind::Less,
                '>' if lx.bump_if('=') => TokenKind::GreaterEqual,
                '>' => TokenKind::Greater,
                other => {
                    return Err(LexError {
                        message: format!("unexpected character {other:?}"),
                        pos: start,
                    });
                }
            }
        };
        tokens.push(Token { kind, pos: start });
    }
    tokens.push(Token {
        kind: TokenKind::Eof,
        pos: lx.pos,
    });
    Ok(tokens)
}
```

**What happens:**

1. `TokenKind` lists every kind of token. Two kinds carry data: `Int(i64)` the value of a number, and
   `Ident(String)` the name of a variable. A `Token` is a kind plus the position where it starts.
2. The private struct `Lexer` holds the remaining characters as a `Peekable<Chars>`: an iterator over
   the characters of the source on which `peek()` looks at the next character without taking it.
3. `bump` takes one character and updates the position: a newline starts a new line at column 1.
   `bump_if` takes a character only if it is the expected one; we use it for two-character operators.
4. In `lex`, whitespace is skipped. A digit starts a number: `take_while` collects all digits, and
   `parse` turns them into an `i64` (a number too large for 64 bits becomes a `LexError`).
5. A letter or `_` starts a word. If the word is a keyword we produce the keyword's kind; otherwise it is
   an identifier.
6. For other characters, match arms with guards (`'/' if lx.bump_if('/')`) handle two-character tokens
   first: `//` starts a comment that is skipped to the end of the line, `==` is one token, a single `=`
   is another. Anything else is an "unexpected character" error at its position.
7. At the end we add an `Eof` (end of file) token, so the parser always has a token to look at.

To see the tokens, save this small program as `examples/tokens.rs` (Cargo builds files in `examples/`
as extra programs, run with `--example`):

```rust
use p22_pebble::lexer::lex;

fn main() {
    let src = "let n = 10 * (k + 1); // a comment\nprint n >= 7;";
    for token in lex(src).unwrap() {
        println!("{}:{} {:?}", token.pos.line, token.pos.col, token.kind);
    }
}
```

`cargo run --example tokens`:

```text
1:1 Let
1:5 Ident("n")
1:7 Equal
1:9 Int(10)
1:12 Star
1:14 LParen
1:15 Ident("k")
1:17 Plus
1:19 Int(1)
1:20 RParen
1:21 Semicolon
2:1 Print
2:7 Ident("n")
2:9 GreaterEqual
2:12 Int(7)
2:13 Semicolon
2:14 Eof
```

The comment produced no token, `>=` became one `GreaterEqual`, and every token knows its line and column.

## 3. The syntax tree

The parser's output is a tree. Each kind of node is an enum variant, and child nodes are stored in `Box`,
because an enum cannot contain itself directly (its size would be infinite; Lesson 05.9). Save as
`src/ast.rs`:

```rust
//! The syntax tree: what a program means, without the punctuation.

use crate::lexer::Pos;
use std::fmt;

#[derive(Debug, Clone, Copy, PartialEq)]
pub enum BinOp {
    Add,
    Sub,
    Mul,
    Div,
    Rem,
    Eq,
    Ne,
    Lt,
    Le,
    Gt,
    Ge,
}

#[derive(Debug, Clone, Copy, PartialEq)]
pub enum UnOp {
    Neg,
}

#[derive(Debug, Clone, PartialEq)]
pub enum Expr {
    Int(i64),
    Bool(bool),
    Var(String, Pos),
    Unary(UnOp, Box<Expr>, Pos),
    Binary(BinOp, Box<Expr>, Box<Expr>, Pos),
}

#[derive(Debug, Clone, PartialEq)]
pub enum Stmt {
    Let(String, Expr),
    Print(Expr),
    Expr(Expr),
}

impl BinOp {
    pub fn symbol(self) -> &'static str {
        match self {
            BinOp::Add => "+",
            BinOp::Sub => "-",
            BinOp::Mul => "*",
            BinOp::Div => "/",
            BinOp::Rem => "%",
            BinOp::Eq => "==",
            BinOp::Ne => "!=",
            BinOp::Lt => "<",
            BinOp::Le => "<=",
            BinOp::Gt => ">",
            BinOp::Ge => ">=",
        }
    }
}

/// Prints an expression as an S-expression: `(+ 1 (* 2 3))`.
impl fmt::Display for Expr {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        match self {
            Expr::Int(n) => write!(f, "{n}"),
            Expr::Bool(b) => write!(f, "{b}"),
            Expr::Var(name, _) => write!(f, "{name}"),
            Expr::Unary(UnOp::Neg, e, _) => write!(f, "(- {e})"),
            Expr::Binary(op, l, r, _) => write!(f, "({} {l} {r})", op.symbol()),
        }
    }
}

impl fmt::Display for Stmt {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        match self {
            Stmt::Let(name, e) => write!(f, "(let {name} {e})"),
            Stmt::Print(e) => write!(f, "(print {e})"),
            Stmt::Expr(e) => write!(f, "{e}"),
        }
    }
}
```

**What happens:**

1. `Expr` is an expression: something that has a value. `Binary(op, left, right, pos)` is, for example,
   `left + right`; the position of the operator is kept for runtime error messages in Lesson 22.23.
2. `Stmt` is a statement: something that is executed. Pebble has `let name = expr;`, `print expr;` and an
   expression used as a statement.
3. The `Display` implementations print the tree as an **S-expression**. `1 + 2 * 3` prints as
   `(+ 1 (* 2 3))`: the brackets show exactly which operator applies to what. This is the easiest way to
   check a parser in a test.

## 4. The parser: recursive descent and Pratt

A **grammar** says which token sequences are programs. We write it in a short notation, where `*` means
"repeat zero or more times" and `|` means "or":

```text
program   := statement* EOF
statement := "let" IDENT "=" expr ";"
           | "print" expr ";"
           | expr ";"
expr      := (handled by Pratt parsing below)
```

**Recursive descent** turns each rule into one function: `program` calls `statement` in a loop;
`statement` looks at the next token and chooses a branch. This is simple and gives good error messages.

For expressions, one function per precedence level would work too, but it gets long. **Pratt parsing**
gives every infix operator a **binding power** and handles all of them in one loop:

| Operators | Binding power |
|---|---|
| `==` `!=` | 3 |
| `<` `<=` `>` `>=` | 4 |
| `+` `-` | 5 |
| `*` `/` `%` | 6 |
| prefix `-` | 7 |

The rule of the loop: `expr(min_power)` reads one operand, then keeps taking operators **as long as
their power is greater than `min_power`**, parsing the right side with `expr(power)`. See it on
`1 + 2 * 3`:

```text
expr(0): operand 1
         sees +, power 5 > 0  → take it, right side = expr(5)
               expr(5): operand 2
                        sees *, power 6 > 5 → take it, right side = expr(6) = 3 → (* 2 3)
                        end of input        → return (* 2 3)
         left = (+ 1 (* 2 3))
```

And on `10 - 3 - 2`:

```text
expr(0): operand 10
         sees -, power 5 > 0  → take it, right side = expr(5)
               expr(5): operand 3
                        sees -, power 5 > 5? no → return 3
         left = (- 10 3)
         sees -, power 5 > 0  → take it, right side = expr(5) = 2
         left = (- (- 10 3) 2)
```

The strict `>` makes operators of the same power group to the left. Save the parser as `src/parser.rs`:

```rust
//! The parser: turns tokens into a syntax tree.
//! Statements: recursive descent. Expressions: Pratt parsing (binding powers).

use crate::ast::{BinOp, Expr, Stmt, UnOp};
use crate::lexer::{Pos, Token, TokenKind};

#[derive(Debug, PartialEq)]
pub struct ParseError {
    pub message: String,
    pub pos: Pos,
}

pub struct Parser {
    tokens: Vec<Token>,
    at: usize,
}

/// How strongly an infix operator binds: higher binds tighter.
fn infix_binding_power(kind: &TokenKind) -> Option<(BinOp, u8)> {
    let pair = match kind {
        TokenKind::EqualEqual => (BinOp::Eq, 3),
        TokenKind::BangEqual => (BinOp::Ne, 3),
        TokenKind::Less => (BinOp::Lt, 4),
        TokenKind::LessEqual => (BinOp::Le, 4),
        TokenKind::Greater => (BinOp::Gt, 4),
        TokenKind::GreaterEqual => (BinOp::Ge, 4),
        TokenKind::Plus => (BinOp::Add, 5),
        TokenKind::Minus => (BinOp::Sub, 5),
        TokenKind::Star => (BinOp::Mul, 6),
        TokenKind::Slash => (BinOp::Div, 6),
        TokenKind::Percent => (BinOp::Rem, 6),
        _ => return None,
    };
    Some(pair)
}

/// Prefix operators bind tighter than every infix operator.
const PREFIX_POWER: u8 = 7;

impl Parser {
    pub fn new(tokens: Vec<Token>) -> Parser {
        Parser { tokens, at: 0 }
    }

    fn peek(&self) -> &Token {
        &self.tokens[self.at]
    }

    fn advance(&mut self) -> Token {
        let token = self.tokens[self.at].clone();
        if token.kind != TokenKind::Eof {
            self.at += 1;
        }
        token
    }

    fn expect(&mut self, kind: TokenKind, what: &str) -> Result<Token, ParseError> {
        if self.peek().kind == kind {
            Ok(self.advance())
        } else {
            Err(self.error(&format!("expected {what}")))
        }
    }

    fn error(&self, message: &str) -> ParseError {
        let token = self.peek();
        ParseError {
            message: format!("{message}, found {:?}", token.kind),
            pos: token.pos,
        }
    }

    /// program := statement* EOF
    pub fn program(&mut self) -> Result<Vec<Stmt>, ParseError> {
        let mut stmts = Vec::new();
        while self.peek().kind != TokenKind::Eof {
            stmts.push(self.statement()?);
        }
        Ok(stmts)
    }

    /// statement := "let" IDENT "=" expr ";" | "print" expr ";" | expr ";"
    fn statement(&mut self) -> Result<Stmt, ParseError> {
        let stmt = match self.peek().kind {
            TokenKind::Let => {
                self.advance();
                let name = match self.advance().kind {
                    TokenKind::Ident(name) => name,
                    _ => return Err(self.error("expected a variable name after `let`")),
                };
                self.expect(TokenKind::Equal, "`=`")?;
                Stmt::Let(name, self.expr(0)?)
            }
            TokenKind::Print => {
                self.advance();
                Stmt::Print(self.expr(0)?)
            }
            _ => Stmt::Expr(self.expr(0)?),
        };
        self.expect(TokenKind::Semicolon, "`;`")?;
        Ok(stmt)
    }

    /// Parses an expression whose operators all bind tighter than `min_power`.
    pub fn expr(&mut self, min_power: u8) -> Result<Expr, ParseError> {
        let mut left = self.prefix()?;
        while let Some((op, power)) = infix_binding_power(&self.peek().kind) {
            if power <= min_power {
                break;
            }
            let pos = self.advance().pos;
            let right = self.expr(power)?;
            left = Expr::Binary(op, Box::new(left), Box::new(right), pos);
        }
        Ok(left)
    }

    /// A literal, a variable, a group in parentheses, or a prefix operator.
    fn prefix(&mut self) -> Result<Expr, ParseError> {
        let token = self.peek().clone();
        let expr = match token.kind {
            TokenKind::Int(n) => Expr::Int(n),
            TokenKind::True => Expr::Bool(true),
            TokenKind::False => Expr::Bool(false),
            TokenKind::Ident(name) => Expr::Var(name, token.pos),
            TokenKind::Minus => {
                self.advance();
                let operand = self.expr(PREFIX_POWER)?;
                return Ok(Expr::Unary(UnOp::Neg, Box::new(operand), token.pos));
            }
            TokenKind::LParen => {
                self.advance();
                let inner = self.expr(0)?;
                self.expect(TokenKind::RParen, "`)`")?;
                return Ok(inner);
            }
            _ => return Err(self.error("expected an expression")),
        };
        self.advance();
        Ok(expr)
    }
}

/// Lexes and parses a whole program.
pub fn parse(src: &str) -> Result<Vec<Stmt>, String> {
    let tokens = crate::lexer::lex(src)
        .map_err(|e| format!("error at {}:{}: {}", e.pos.line, e.pos.col, e.message))?;
    Parser::new(tokens)
        .program()
        .map_err(|e| format!("error at {}:{}: {}", e.pos.line, e.pos.col, e.message))
}

#[cfg(test)]
mod tests {
    use super::*;

    /// Parses one expression statement and prints it as an S-expression.
    fn show(src: &str) -> String {
        let stmts = parse(&format!("{src};")).unwrap();
        stmts[0].to_string()
    }

    #[test]
    fn multiplication_binds_tighter_than_addition() {
        assert_eq!(show("1 + 2 * 3"), "(+ 1 (* 2 3))");
        assert_eq!(show("(1 + 2) * 3"), "(* (+ 1 2) 3)");
    }

    #[test]
    fn same_power_operators_group_to_the_left() {
        assert_eq!(show("10 - 3 - 2"), "(- (- 10 3) 2)");
    }

    #[test]
    fn prefix_minus_and_comparisons() {
        assert_eq!(show("-x * 2 < 7 == true"), "(== (< (* (- x) 2) 7) true)");
    }

    #[test]
    fn reports_the_position_of_errors() {
        assert_eq!(
            parse("let x = ;"),
            Err("error at 1:9: expected an expression, found Semicolon".into())
        );
    }
}
```

**What happens:**

1. `Parser` holds the tokens and the index `at` of the next one. `peek` looks at it, `advance` takes it
   (but never moves past `Eof`), `expect` takes a token of one exact kind or returns a `ParseError` that
   names what was expected and what was found.
2. `program` and `statement` are recursive descent: `statement` chooses a branch by the first token and
   then requires the final `;`.
3. `expr` is the Pratt loop described above. `while let Some((op, power)) = ...` stops at any token that
   is not an infix operator, such as `;` or `)`.
4. `prefix` parses what can start an expression: a number, `true`/`false`, a variable, a prefix minus
   (whose operand is parsed with `PREFIX_POWER`, so `-x * 2` is `(* (- x) 2)`), or a group in brackets,
   where `expr(0)` starts again at the lowest power. (Powers 1 and 2 are free on purpose: exercise 3
   uses them.)
5. `parse` runs the lexer and the parser and turns both kinds of errors into one message with a position.
6. The tests check precedence, associativity, prefix operators and an error message, all through the
   S-expression form.

Finally, `src/main.rs` reads a file and prints its tree:

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
    match p22_pebble::parser::parse(&src) {
        Ok(program) => {
            for stmt in &program {
                println!("{stmt}");
            }
            ExitCode::SUCCESS
        }
        Err(message) => {
            eprintln!("{path}: {message}");
            ExitCode::FAILURE
        }
    }
}
```

`cargo test` and `cargo run -- programs/first.pb`:

```text
test parser::tests::reports_the_position_of_errors ... ok
test parser::tests::multiplication_binds_tighter_than_addition ... ok
test parser::tests::same_power_operators_group_to_the_left ... ok
test parser::tests::prefix_minus_and_comparisons ... ok
test result: ok. 4 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

```text
(let width 6)
(let height 7)
(print (+ (* width height) 1))
(print (== (>= (- (- width 10)) 4) true))
```

Now a broken program. Save `let x = (1 + 2;` as `programs/broken.pb` and run
`cargo run -- programs/broken.pb`:

```text
programs/broken.pb: error at 1:15: expected `)`, found Semicolon
```

> 🧪 **Testing:** Parser tests compare S-expressions, not Rust structures: `"(+ 1 (* 2 3))"` is much easier
> to read and write than nested `Box::new(...)` calls. Add a test for every bug you find in the parser,
> and test error messages too: they are part of the language's user interface.

> 🔒 **Security:** A parser is the first code that touches untrusted input (for example, a configuration
> language or a query sent over the network). It must never panic, and very deep nesting such as
> 100,000 opening brackets can overflow the stack of a recursive parser. Real parsers limit the
> nesting depth.

## Common mistakes

**Left recursion in recursive descent.** A rule written as `sum := sum "+" NUMBER | NUMBER` turns into a
function that calls itself before it takes any token. Save as `examples/left_recursion.rs` and run
`cargo run --example left_recursion`:

```rust
/// A broken rule: sum := sum "+" NUMBER | NUMBER
fn sum(tokens: &[&str], at: &mut usize) -> i64 {
    let left = sum(tokens, at); // calls itself before taking any token
    if tokens.get(*at) == Some(&"+") {
        *at += 1;
        let right: i64 = tokens[*at].parse().unwrap();
        *at += 1;
        return left + right;
    }
    left
}

fn main() {
    let tokens = ["1", "+", "2"];
    println!("{}", sum(&tokens, &mut 0));
}
```

The compiler warns, and running it crashes:

```text
warning: function cannot return without recursing
 --> examples/left_recursion.rs:2:1
  |
2 | fn sum(tokens: &[&str], at: &mut usize) -> i64 {
  | ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ cannot return without recursing
3 |     let left = sum(tokens, at); // calls itself before taking any token
  |                --------------- recursive call site

thread 'main' (6315492) has overflowed its stack
fatal runtime error: stack overflow, aborting
```

Fix: parse one operand first, then loop over the operators, as `expr` does. Delete the file afterwards.

**`<` instead of `<=` in the Pratt loop.** Change `if power <= min_power` to `if power < min_power` and
run `cargo test`:

```text
---- parser::tests::same_power_operators_group_to_the_left stdout ----

thread 'parser::tests::same_power_operators_group_to_the_left' (6365722) panicked at src/parser.rs:171:9:
assertion `left == right` failed
  left: "(- 10 (- 3 2))"
 right: "(- (- 10 3) 2)"
```

Now `10 - 3 - 2` means `10 - (3 - 2)`, which is 9 instead of 5. The one-character difference decides
the associativity, which is exactly why this test exists.

**Forgetting the position.** Without line and column, "expected `)`" in a 500-line program is almost
useless. Keep a position in every token, and in every AST node that can fail at run time.

## Try it yourself

1. Add `print 1 +;` to `programs/first.pb`. Which position does the error show, and why that one?
2. Try `print 1 2;`. The parser reads `1`, sees no operator, and then `expect` wants `;` but finds
   `Int(2)`.
3. Write `print 2 * -3;`. Check that the tree is `(print (* 2 (- 3)))`.

## Exercises

### Exercise 1 — Readable numbers (easy)

Allow underscores inside numbers, so `1_000_000` is the integer one million. A number must still start
with a digit. Write a unit test in `src/lexer.rs` that lexes `"1_000_000 7"` and checks the token kinds.

<details>
<summary>Hint</summary>

In the digit branch, let `take_while` also accept `'_'`, then remove the underscores with
`text.replace('_', "")` before `parse`.

</details>

<details>
<summary>Answer</summary>

The digit branch becomes:

```rust
let text = lx.take_while(|c| c.is_ascii_digit() || c == '_');
let value = text.replace('_', "").parse().map_err(|_| LexError {
    message: format!("number too large: {text}"),
    pos: start,
})?;
```

and at the end of `src/lexer.rs`:

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn numbers_may_contain_underscores() {
        let kinds: Vec<TokenKind> = lex("1_000_000 7")
            .unwrap()
            .into_iter()
            .map(|t| t.kind)
            .collect();
        assert_eq!(
            kinds,
            vec![TokenKind::Int(1_000_000), TokenKind::Int(7), TokenKind::Eof]
        );
    }
}
```

```text
test lexer::tests::numbers_may_contain_underscores ... ok
```

Why it works: the branch is entered only when the first character is a digit, so `_x` is still an
identifier. The error message keeps the original text with underscores, which is what the user wrote.

</details>

### Exercise 2 — A power operator (medium)

Add `**` for powers. It binds tighter than `*`, and it is **right-associative**: `2 ** 3 ** 2` means
`2 ** (3 ** 2)`. A prefix minus applies to the whole power: `-2 ** 2` means `-(2 ** 2)`. Write tests:

- `2 ** 3 ** 2` → `(** 2 (** 3 2))`
- `2 * 3 ** 2` → `(* 2 (** 3 2))`
- `-2 ** 2` → `(- (** 2 2))`

<details>
<summary>Hint</summary>

You need a token `StarStar` (`'*' if lx.bump_if('*')`, placed before the `'*'` arm), a `BinOp::Pow` with
the symbol `"**"`, and binding power 8 (higher than `PREFIX_POWER`, so `-2 ** 2` works). For right
associativity, parse the right side with `power - 1` instead of `power`, so an operator of the same power
is taken on the right.

</details>

<details>
<summary>Answer</summary>

`src/lexer.rs`: add `StarStar,` after `Star,` in `TokenKind`, and before the `'*'` arm:

```rust
'*' if lx.bump_if('*') => TokenKind::StarStar,
```

`src/ast.rs`: add `Pow,` to `BinOp` and `BinOp::Pow => "**",` to `symbol`. `src/parser.rs`: add
`TokenKind::StarStar => (BinOp::Pow, 8),` to `infix_binding_power`, and in `expr`:

```rust
let pos = self.advance().pos;
// `**` groups to the right: allow the same power again on the right side.
let right_min = if op == BinOp::Pow { power - 1 } else { power };
let right = self.expr(right_min)?;
```

The test, in the `tests` module of `src/parser.rs`:

```rust
#[test]
fn power_groups_to_the_right_and_binds_tightest() {
    assert_eq!(show("2 ** 3 ** 2"), "(** 2 (** 3 2))");
    assert_eq!(show("2 * 3 ** 2"), "(* 2 (** 3 2))");
    assert_eq!(show("-2 ** 2"), "(- (** 2 2))");
}
```

```text
test parser::tests::power_groups_to_the_right_and_binds_tightest ... ok
```

Why it works: inside `expr(7)` (the right side of the first `**`, with `power - 1 = 7`), the second `**`
has power 8 > 7, so it is taken there, on the right. For `-2 ** 2`, the prefix operand is parsed with
`expr(7)`, and `**` (8) is taken inside it, so the minus applies to the power.

</details>

### Exercise 3 — Logic operators (hard)

Add the logic operators `&&` (and), `||` (or) and the prefix `!` (not). They bind looser than every
comparison, and `&&` binds tighter than `||`, so `!a || b && c` means `(!a) || (b && c)`. Write a test:

- `!a || b && c` → `(|| (! a) (&& b c))`
- `x < 1 || x > 9` → `(|| (< x 1) (> x 9))`

<details>
<summary>Hint</summary>

Three new tokens (`Bang`, `AndAnd`, `OrOr`; `&` and `|` alone stay errors), three new AST cases
(`BinOp::And`, `BinOp::Or`, `UnOp::Not`) with their symbols, and the two free binding powers 1 and 2.
In the lexer, the arm `'!' => TokenKind::Bang` must come *after* `'!' if lx.bump_if('=')`.

</details>

<details>
<summary>Answer</summary>

`src/lexer.rs`: add `Bang,`, `AndAnd,` and `OrOr,` to `TokenKind`, and these arms (the `!` one right
after the `!=` arm, the others after `'>' => TokenKind::Greater,`):

```rust
'!' => TokenKind::Bang,
'&' if lx.bump_if('&') => TokenKind::AndAnd,
'|' if lx.bump_if('|') => TokenKind::OrOr,
```

`src/ast.rs`: add `And,` and `Or,` to `BinOp` with `BinOp::And => "&&",` and `BinOp::Or => "||",` in
`symbol`; add `Not,` to `UnOp` and to `Display`:

```rust
Expr::Unary(UnOp::Not, e, _) => write!(f, "(! {e})"),
```

`src/parser.rs`: two binding powers, and the prefix branch accepts both operators:

```rust
TokenKind::OrOr => (BinOp::Or, 1),
TokenKind::AndAnd => (BinOp::And, 2),
```

```rust
TokenKind::Minus | TokenKind::Bang => {
    self.advance();
    let op = if token.kind == TokenKind::Bang {
        UnOp::Not
    } else {
        UnOp::Neg
    };
    let operand = self.expr(PREFIX_POWER)?;
    return Ok(Expr::Unary(op, Box::new(operand), token.pos));
}
```

and the test:

```rust
#[test]
fn logic_operators_bind_loosest() {
    assert_eq!(show("!a || b && c"), "(|| (! a) (&& b c))");
    assert_eq!(show("x < 1 || x > 9"), "(|| (< x 1) (> x 9))");
}
```

```text
test parser::tests::logic_operators_bind_loosest ... ok
```

Why it works: the Pratt loop does not care which operators exist; adding an operator is one line in the
binding-power table. `!` is parsed like the prefix minus, with `PREFIX_POWER`, so it applies only to the
operand right after it.

</details>

## Quiz

1. What does the lexer produce, and what does the parser produce?

<details>
<summary>Answer</summary>

The lexer produces a list of tokens (with positions). The parser produces a syntax tree (AST) from those
tokens, or an error.

</details>

2. Why does every token carry a line and a column?

<details>
<summary>Answer</summary>

So that the parser (and later the interpreter) can report *where* an error is. Error messages without a
position are almost useless in a long program.

</details>

3. What do precedence and associativity decide? Give an example of each.

<details>
<summary>Answer</summary>

Precedence decides which operator binds tighter: `1 + 2 * 3` is `1 + (2 * 3)`. Associativity decides how
operators of the same precedence group: `10 - 3 - 2` is `(10 - 3) - 2`.

</details>

4. In the Pratt loop, why is the comparison `power <= min_power` (stop) and not `power < min_power`?

<details>
<summary>Answer</summary>

With `<=`, an operator of the same power as the one on the left stops the inner call, so the outer loop
takes it: the operators group to the left. With `<`, the inner call takes it, and they group to the right.

</details>

5. Why are the child expressions in `Expr` stored in `Box<Expr>`?

<details>
<summary>Answer</summary>

An enum that contains itself directly would have an infinite size. A `Box` is a pointer of fixed size to
a value on the heap, so the enum has a known size.

</details>

6. What is left recursion, and why does it break recursive descent?

<details>
<summary>Answer</summary>

A rule that starts with itself, like `sum := sum "+" NUMBER`. Its function calls itself before taking any
token, so it recurses forever and overflows the stack. Parse an operand first and then loop.

</details>

## Summary

- A compiler or interpreter is a pipeline: source text → lexer → tokens → parser → syntax tree → an
  interpreter or a code generator.
- The lexer works on characters, skips spaces and comments, and records a position for every token.
- The AST is a tree of enum values with `Box` children; printing it as S-expressions makes parser tests
  easy to read.
- Recursive descent has one function per grammar rule and is ideal for statements.
- Pratt parsing gives each operator a binding power; `expr(min_power)` with a strict comparison handles
  precedence and left associativity in one loop, and `power - 1` gives right associativity.
- Avoid left recursion, keep positions for error messages, and test every precedence rule.

## Sources and further reading

- [Crafting Interpreters](https://craftinginterpreters.com/) — Robert Nystrom's free book; chapters 4–6 cover scanning, syntax trees and parsing.
- [Simple but Powerful Pratt Parsing](https://matklad.github.io/2020/04/13/simple-but-powerful-pratt-parsing.html) — a clear article on Pratt parsing in Rust by the author of rust-analyzer.
- [logos documentation](https://docs.rs/logos/0.16/logos/) — a lexer generator: you describe tokens with patterns and it writes a fast lexer; an alternative to the hand-written one.
- [The Rust Reference — Tokens](https://doc.rust-lang.org/reference/tokens.html) — how Rust's own lexer defines its tokens; a good example of a precise lexical grammar.
- [The Rust Reference — Expression precedence](https://doc.rust-lang.org/reference/expressions.html) — the precedence table of Rust, for comparison with Pebble's.

---

[🏠 Start](../START-HERE.md) · [← 22.21 Networking track: project and next steps](21-networking-track-project-and-next-steps.md) · [📋 Phase 22](README.md) · [22.23 Compilers track: a tree-walking interpreter →](23-compilers-track-an-interpreter.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
