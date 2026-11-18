<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 22.22 Трек компиляторов: лексический и синтаксический анализ](22-compilers-track-lexing-and-parsing.md) · [📋 Этап 22](README.md) · [22.24 Трек компиляторов: WebAssembly и Rust →](24-compilers-track-webassembly.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/22-specializations/23-compilers-track-an-interpreter.md) <!-- nav -->

# Урок 22.23 — Трек компиляторов: интерпретатор обхода дерева

> **Этап 22 — Специализации** · Урок 23 из 30 · ⏱ около 3 часов (2 занятия)

## Что вы узнаете

- Как интерпретатор обхода дерева выполняет программу: одна рекурсивная функция для выражений, одна — для инструкций.
- Как представлять значения во время выполнения и как сообщать об ошибках выполнения с позициями.
- Как работают области видимости: блоки, затенение и присваивание со стеком хеш-таблиц.
- Как добавить в парсер из Урока 22.22 блоки, `if`/`else` и `while`.
- Как тестировать интерпретатор, запуская маленькие программы и сравнивая их вывод.

## Перед началом

- Вы прошли [Урок 22.22 — Трек компиляторов: лексический и синтаксический анализ](22-compilers-track-lexing-and-parsing.md)
  и у вас есть его пакет `code/p22-pebble`. Код ниже начинается с версии урока; если вы делали упражнения
  Урока 22.22, сохраните свои дополнения: упражнение 2 этого урока использует логические операторы.
- Вы знаете `HashMap` ([Урок 04.9](../04-structs-enums-collections-errors/09-hash-maps.md)), трейт-объекты
  (`dyn Write`, [Урок 06.1](../06-rust-advanced/01-trait-objects-and-dynamic-dispatch.md)) и арифметику с
  проверкой ([Урок 12.2 — Целые числа в памяти](../12-computer-systems/02-integers-in-memory.md)).

## Новые слова

| Слово | Значение |
|---|---|
| **интерпретатор обхода дерева** (tree-walking interpreter) | Интерпретатор, который выполняет программу, посещая узлы её синтаксического дерева и вычисляя каждый узел по его детям. |
| **вычислить** (evaluate) | Найти значение выражения. |
| **выполнить** (execute) | Запустить инструкцию ради её эффекта (печать, изменение переменной). |
| **значение** (value) | Данные во время выполнения, например целое `42` или логическое `true`. |
| **ошибка выполнения** (run-time error) | Ошибка, найденная во время работы программы (деление на ноль), в отличие от синтаксической ошибки, которую находит парсер. |
| **область видимости** (scope) | Часть программы, где видно имя переменной. В Pebble каждый блок `{ ... }` открывает новую область видимости. |
| **окружение** (environment) | Запись интерпретатора о том, какие переменные существуют и какие значения они хранят, по областям видимости. |
| **затенение** (shadowing) | `let` во внутренней области видимости создаёт новую переменную с тем же именем; внешняя скрыта, пока блок не закончится. |
| **вычисление по короткой схеме** (short-circuit evaluation) | `a && b` не вычисляет `b`, если `a` ложно, а `a \|\| b` — если `a` истинно. |

## 1. Как работает интерпретатор обхода дерева

Синтаксическое дерево уже говорит, что вычислять. Интерпретатор обходит его сверху: чтобы узнать
значение `(+ 1 (* 2 3))`, ему сначала нужны значения `1` и `(* 2 3)`; для `(* 2 3)` нужны `2` и `3` —
это листья. Значения поднимаются обратно наверх:

```text
        (+ 1 (* 2 3))      eval(+) = eval(1) + eval(*)  = 1 + 6 = 7
         /        \
        1      (* 2 3)     eval(*) = eval(2) * eval(3)  = 2 * 3 = 6
                /   \
               2     3     leaves: their value is the number itself
```

Поэтому `eval` — рекурсивная функция с одной веткой `match` на каждый вид выражения, а `exec` — ещё
одна, с одной веткой на каждый вид инструкции. Это самый простой способ выполнять язык. Он медленнее,
чем компиляция в байт-код или машинный код (Урок 22.24), потому что интерпретатор заново читает дерево
при каждом повторе цикла, но его легко писать, тестировать и менять, и многие настоящие языки
начинались именно так.

Вместе с выполнением кода приходят две новые идеи:

- **Значения.** В Pebble их два вида: `Value::Int(i64)` и `Value::Bool(bool)`. Оператор проверяет виды
  своих операндов во время выполнения: `1 + true` — ошибка.
- **Ошибки выполнения.** Деление на ноль, неопределённая переменная или переполнение — не ошибки
  интерпретатора, а ошибки *программы на Pebble*. Интерпретатор должен сообщить о них с позицией и
  аккуратно остановиться, никогда не паникуя.

## 2. Блоки, `if`, `while` и присваивание в парсере

Циклу нужна переменная, которая меняется, поэтому Pebble получает **присваивание** (`i = i + 1;`, без
`let`) и три инструкции, которые содержат другие инструкции. Грамматика растёт:

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

(`?` значит «необязательно».) Лексер получает пять новых токенов. В `src/lexer.rs` добавьте `If,`,
`Else,` и `While,` после `Print,`, а `LBrace,` и `RBrace,` — после `RParen,` в `TokenKind`, ключевые
слова — в `match name.as_str()`, а фигурные скобки — в `match` по символам:

```rust
"if" => TokenKind::If,
"else" => TokenKind::Else,
"while" => TokenKind::While,
```

```rust
'{' => TokenKind::LBrace,
'}' => TokenKind::RBrace,
```

В `src/ast.rs` у `Stmt` появляются четыре новых варианта. У `if` в дереве всегда есть ветка else; если
в программе нет `else`, эта ветка — пустой список. Реализация `Display` печатает их с помощью маленькой
вспомогательной функции `list`:

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

В `src/parser.rs` замените старую функцию `statement` этими тремя функциями:

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

**Что происходит:**

1. Теперь `statement` сопоставляет *клон* вида токена, потому что ветка `Ident(name)` забирает имя, а
   условие `if self.tokens[self.at + 1].kind == TokenKind::Equal` смотрит на токен дальше: имя, за
   которым идёт `=`, — это присваивание, имя, за которым идёт что-то другое, начинает выражение.
   (Условие безопасно: `Ident` никогда не бывает последним токеном, потому что за ним всегда идёт `Eof`.)
2. Блоки, `if` и `while` заканчиваются на `}`, поэтому они делают ранний `return` вместо ожидания `;`.
3. `block` собирает инструкции до `}`. Проверка `Eof` предотвращает бесконечный цикл, если `}` нет;
   тогда об ошибке сообщает `expect`.
4. `if_statement` обрабатывает `else if`, разбирая внутренний `if` как единственную инструкцию ветки
   else. Поэтому `if_statement` — отдельная функция: она вызывает сама себя.

Добавьте тест в модуль `tests` парсера:

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

## 3. Интерпретатор

Сохраните как `src/interp.rs` и добавьте `pub mod interp;` в `src/lib.rs`:

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

**Что происходит:**

1. `Value` реализует `Copy`: целое число или логическое значение маленькие, поэтому значения свободно
   копируются. Его `Display` — это то, что показывает `print`.
2. У `RuntimeError` необязательная позиция: у ошибок в литералах или при записи вывода её нет.
3. `Interpreter` хранит **окружение**: `scopes` — стек (`Vec`) хеш-таблиц из имён в значения. Он пишет
   вывод в `out` — `&mut dyn Write`, поэтому `main` может передать стандартный вывод, а тесты — `Vec<u8>`
   (Урок 09.3: тестовый двойник).
4. У `exec` одна ветка на каждую инструкцию. `Let` вставляет во **внутреннюю** область видимости.
   `Assign` ищет по областям видимости изнутри наружу и меняет первую переменную с таким именем; если её
   нет — это ошибка, потому что переменные создаёт только `let`.
5. `exec_block` кладёт новую область видимости, выполняет инструкции и снимает область видимости, **даже
   если инструкция завершилась ошибкой**: `try_for_each` останавливается на первой ошибке, но `pop` идёт
   после него в любом случае.
6. `condition` вычисляет условие `if` и `while` и требует bool: `if 1 { ... }` — ошибка, а не «истина».
7. У `eval` одна ветка на каждое выражение. Переменная ищется от внутренней области видимости наружу,
   поэтому внутреннее `let x` скрывает внешнее `x` (**затенение**).
8. `binary` сопоставляет оператор *и оба значения* одним кортежем. Каждая арифметическая операция
   использует методы `checked_*`, которые возвращают `None` при переполнении, вместо паники (в
   отладочной сборке) или переноса по модулю (в релизной). У деления на ноль своя ветка, перед общей
   веткой деления. Последняя ветка ловит любую неверную комбинацию, например `1 + true`.
9. `run_source` разбирает и выполняет всю программу и возвращает её вывод как `String`. Тесты используют
   её, чтобы проверить вывод программы в одну строку.

> ⚠️ **Частая ошибка:** если вы оставили `UnOp::Not` из упражнения 3 Урока 22.22, `match` в `eval` не покрывает
> `Expr::Unary(UnOp::Not, ..)`, и сборка останавливается с ошибкой `E0004` («non-exhaustive patterns», не все варианты
> разобраны). Настоящую ветку добавит упражнение 2 ниже; до тех пор добавьте временную:
> `` Expr::Unary(UnOp::Not, _, pos) => Err(error("`!` is not supported yet".into(), *pos)), ``
> (`&&` и `||` пока ничего не требуют: они доходят до последней ветки `binary` и дают ошибку времени выполнения.)

Области видимости в действии для `let x = 1; { let x = 2; print x; } print x;`:

```text
 before the block      inside the block           after the block
 ┌─────────┐           ┌─────────┐ ┌─────────┐    ┌─────────┐
 │ x = 1   │           │ x = 1   │ │ x = 2   │    │ x = 1   │
 └─────────┘           └─────────┘ └─────────┘    └─────────┘
  scopes[0]             scopes[0]   scopes[1]      scopes[0]
                        print x → finds scopes[1] first → 2
```

Наконец, `src/main.rs` выполняет файл, а не печатает его дерево:

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

Сохраните эту программу как `programs/loops.pb`:

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

И программа с ошибкой, `programs/oops.pb`:

```text
let total = 10;
let count = 0;
print total / count;
```

`cargo run -- programs/oops.pb` печатает ошибку с позицией `/` и завершается с кодом 1:

```text
programs/oops.pb:3:13: runtime error: division by zero
```

`cargo test` запускает тесты парсера и тесты интерпретатора:

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

> 🧪 **Тестирование:** лучшие тесты для интерпретатора — маленькие программы с ожидаемым выводом,
> например `run_source("let x = 6; print x * 7;") == "42\n"`. Напишите такой тест для каждой возможности
> и каждого сообщения об ошибке. Позже заведите папку программ `.pb` с файлами `.out` и тест, который
> запускает их все: это называется набором **эталонных тестов** (golden tests), и он есть у каждого
> настоящего языка.

> 🔒 **Безопасность:** интерпретатор часто выполняет код, написанный кем-то другим: плагины, скрипты
> конфигурации, правила, введённые пользователями. Такой код не должен ронять хост (никаких паник:
> используйте арифметику с проверкой и возвращайте ошибки), не должен работать вечно и использовать
> неограниченную память. Упражнение 3 добавляет ограничение числа шагов — первую линию защиты.
> WebAssembly (Урок 22.24) даёт такую же песочницу для скомпилированного кода.

## Частые ошибки

**Использовать `+` из Rust для `+` языка.** Замените ветку `Add` в `binary` на
`(BinOp::Add, Int(a), Int(b)) => Ok(Int(a + b)),` и выполните `print 9223372036854775807 + 1;`:

```text
thread 'main' (6413393) panicked at src/interp.rs:152:48:
attempt to add with overflow
```

*Интерпретатор* упал из-за ошибки в программе на *Pebble*. С `checked_add` та же программа даёт
нормальную ошибку: `runtime error: integer overflow` с позицией `1:27`.

**`Assign`, который вставляет во внутреннюю область видимости.** Если бы присваивание просто делало
`self.scopes.last_mut().unwrap().insert(...)`, то `i = i + 1;` внутри тела `while` создавало бы новую `i`
в области видимости тела, которая исчезает в конце каждого прохода. `i` цикла никогда не меняется, и
цикл никогда не заканчивается. Присваивание должно найти область видимости, где переменная уже живёт.

**Забыть снять область видимости после ошибки.** Если написать `for stmt in body { self.exec(stmt)?; }`,
а затем `self.scopes.pop();`, область видимости остаётся на стеке, когда `?` делает ранний возврат. В
этом интерпретаторе ошибка и так завершает программу, но как только вы добавите обработку ошибок или
функции (Урок 22.25), устаревшая область видимости сделает переменные видимыми там, где их не должно
быть. Снимайте её в любом случае, как это делает `exec_block`.

**Истинность по случайности.** Считать любое ненулевое целое число `true` — решение при проектировании
языка, а не то, что должно получиться случайно. Pebble требует `bool` в условиях, и `condition` об этом
говорит.

## Попробуйте сами

1. Запустите программу, которая печатает переменную, определённую внутри блока, после конца блока:
   `let x = 5; { let y = x + 1; } print y;`. Ошибка — `undefined variable `y`` в позиции `y` в последнем
   `print`.
2. Напишите `programs/countdown.pb`, которая печатает 5, 4, 3, 2, 1 циклом `while`.
3. Попробуйте `if 1 { print 2; }`. Ошибка — `condition must be a bool` без позиции, потому что литерал
   `1` её не хранит. Как бы вы добавили позиции литералам?

## Упражнения

### Упражнение 1 — Протестируйте цепочку `else if` (легко)

Напишите тест, который запускает одну программу три раза с `n`, равным 30, 10 и 7. Программа печатает
`15`, если `n` делится на 15, иначе `5`, если делится на 5, иначе `0`.

<details>
<summary>Подсказка</summary>

Соберите исходный текст через `format!` внутри замыкания, которое принимает `n`. В `format!` пишите `{{`
и `}}` для фигурных скобок блоков Pebble, потому что одиночные скобки — это места подстановки.

</details>

<details>
<summary>Ответ</summary>

В модуле `tests` файла `src/interp.rs`:

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

Почему это работает: 30 делится на 15, поэтому выполняется первая ветка, а часть `else` пропускается;
10 не проходит первую проверку и проходит вторую; 7 доходит до последнего `else`.

</details>

### Упражнение 2 — `&&`, `||` и `!` по короткой схеме (средне)

Если вы добавили логические операторы в парсер (упражнение 3 Урока 22.22), заставьте их работать в
интерпретаторе. `&&` и `||` должны вычисляться **по короткой схеме**: `false && 1 / 0 == 0` — это
`false`, и деления на ноль не происходит. Оба операнда и операнд `!` должны быть bool. Протестируйте.

<details>
<summary>Подсказка</summary>

`&&` и `||` не могут идти через `binary`, потому что `binary` получает два уже вычисленных значения.
Добавьте ветку перед общей веткой `Expr::Binary` с образцом
`Expr::Binary(op @ (BinOp::And | BinOp::Or), left, right, pos)`: вычислите `left`, примите решение и
вычисляйте `right` только при необходимости.

</details>

<details>
<summary>Ответ</summary>

В `eval`, после ветки `UnOp::Neg`:

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

и тест:

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

Почему это работает: ветка с привязкой `@` (Урок 06.3) совпадает только с `And` и `Or` и стоит перед
общей веткой, поэтому эти операторы никогда не доходят до `binary`. `let ... else` достаёт bool из
левого значения или возвращает ошибку.

</details>

### Упражнение 3 — Ограничение числа шагов (сложно)

Добавьте `pub fn with_step_limit(out: &'w mut dyn Write, limit: u64) -> Interpreter<'w>`. Каждая
выполненная инструкция стоит один шаг; когда бюджет исчерпан, `run` возвращает `RuntimeError` с
сообщением `step limit reached`. `new` продолжает работать без ограничения. Убедитесь, что
`while true { }` с **пустым** телом тоже останавливается. Напишите тест с бесконечным циклом и
ограничением 10 000.

<details>
<summary>Подсказка</summary>

Добавьте поле `steps_left: u64` и метод `tick(&mut self) -> Result<(), RuntimeError>`, который падает на
нуле, а иначе вычитает единицу. Вызывайте его в начале `exec`. Пустое тело цикла не выполняет
инструкций, поэтому вызывайте `tick` ещё и один раз за проход цикла `while`.

</details>

<details>
<summary>Ответ</summary>

Добавьте `steps_left: u64,` в структуру и замените `new` на:

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

Вызовите `self.tick()?;` первой строкой `exec`, а в ветке `While`:

```rust
while self.condition(cond)? {
    self.tick()?; // also counts loops with an empty body
    self.exec_block(body)?;
}
```

Тест:

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

Почему это работает: каждый путь через программу, который может повторяться, проходит через `tick`,
поэтому число шагов ограничено, и хост получает управление обратно с ошибкой, а не зависает. Настоящие
песочницы используют ту же идею и называют её **топливом** (fuel; у Wasmtime есть настройка fuel для
WebAssembly).

</details>

## Вопросы для самопроверки

1. Чем `eval` отличается от `exec`?

<details>
<summary>Ответ</summary>

`eval` вычисляет значение выражения и возвращает `Value`. `exec` выполняет инструкцию ради её эффекта
(печать, создание или изменение переменной, выполнение цикла) и не возвращает ничего, кроме возможной
ошибки.

</details>

2. Почему интерпретатор использует `checked_add`, а не `+`?

<details>
<summary>Ответ</summary>

Переполнение в программе на Pebble — ошибка этой программы. С `+` сам интерпретатор паниковал бы
(отладочная сборка) или молча переносил бы значение по модулю (релизная сборка). `checked_add`
возвращает `None`, который становится ошибкой выполнения с позицией.

</details>

3. Как окружение реализует области видимости и что происходит, когда блок заканчивается?

<details>
<summary>Ответ</summary>

Это стек хеш-таблиц. Блок кладёт новую пустую таблицу; `let` вставляет в верхнюю таблицу; поиск идёт
сверху вниз. Когда блок заканчивается, его таблица снимается, и его переменные исчезают.

</details>

4. Чем `let x = 2;` отличается от `x = 2;` внутри блока, если `x` существует снаружи?

<details>
<summary>Ответ</summary>

`let x = 2;` создаёт новую `x` в области видимости блока, которая затеняет внешнюю; после блока внешняя
`x` по-прежнему имеет старое значение. `x = 2;` меняет внешнюю `x`.

</details>

5. Почему `&&` и `||` нужна особая обработка, а не обычная ветка в `binary`?

<details>
<summary>Ответ</summary>

`binary` получает оба операнда уже вычисленными. Вычисление по короткой схеме должно решить, *нужно ли*
вычислять правый операнд, увидев левый, поэтому его приходится делать в `eval`.

</details>

## Итоги

- Интерпретатор обхода дерева вычисляет выражения одним рекурсивным `match`, а выполняет инструкции
  другим; значения поднимаются вверх по дереву.
- Значения во время выполнения несут свой вид (`Int`, `Bool`); операторы проверяют виды и возвращают
  ошибки, никогда не паникуя.
- Ошибки выполнения принадлежат программе пользователя: сообщайте о них с позицией и аккуратно
  останавливайтесь.
- Окружение — стек областей видимости: `let` добавляет во внутреннюю, поиск и присваивание идут наружу,
  а блок снимает свою область видимости в любом случае.
- Блоки, `if`/`else if`/`else` и `while` — это ещё несколько функций рекурсивного спуска в парсере.
- Тестируйте интерпретатор маленькими программами с ожидаемым выводом; защищайте хосты арифметикой с
  проверкой и ограничением числа шагов.

## Источники и что почитать

- [Crafting Interpreters — Evaluating Expressions](https://craftinginterpreters.com/evaluating-expressions.html) — та же идея в книге Robert Nystrom, на другом языке.
- [Crafting Interpreters — Statements and State](https://craftinginterpreters.com/statements-and-state.html) — переменные, присваивание и окружения.
- [Crafting Interpreters — Control Flow](https://craftinginterpreters.com/control-flow.html) — `if`, `while` и логика по короткой схеме.
- [std::primitive::i64 — checked_add and friends](https://doc.rust-lang.org/std/primitive.i64.html) — методы арифметики с проверкой, с переносом и с насыщением.
- [The Rust Programming Language — 8.3 Hash Maps](https://doc.rust-lang.org/book/ch08-03-hash-maps.html) — повторение о таблице, на которой стоит каждая область видимости.

---

[🏠 Начало](../START-HERE.md) · [← 22.22 Трек компиляторов: лексический и синтаксический анализ](22-compilers-track-lexing-and-parsing.md) · [📋 Этап 22](README.md) · [22.24 Трек компиляторов: WebAssembly и Rust →](24-compilers-track-webassembly.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
