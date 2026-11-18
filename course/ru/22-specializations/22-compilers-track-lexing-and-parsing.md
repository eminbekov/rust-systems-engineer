<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 22.21 Трек сетей: проект и следующие шаги](21-networking-track-project-and-next-steps.md) · [📋 Этап 22](README.md) · [22.23 Трек компиляторов: интерпретатор обхода дерева →](23-compilers-track-an-interpreter.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/22-specializations/22-compilers-track-lexing-and-parsing.md) <!-- nav -->

# Урок 22.22 — Трек компиляторов: лексический и синтаксический анализ

> **Этап 22 — Специализации** · Урок 22 из 30 · ⏱ около 4 часов (2 занятия)

> **Не ваше направление?** Этот урок открывает направление компиляторов и WebAssembly (Уроки
> 22.22–22.25). Если вы выбрали другое направление, переходите к [Уроку 22.26 — Трек cloud native: операторы Kubernetes](26-cloud-native-track-kubernetes-operators.md)
> или сразу к [Уроку 22.30 — Контрольная точка: ваша специализация](30-checkpoint-your-specialization.md).

## Что вы узнаете

- Что делают компиляторы и интерпретаторы — как конвейер маленьких шагов — и что строит это направление.
- Как лексер превращает текст в токены со строкой и столбцом для сообщений об ошибках.
- Как синтаксическое дерево (AST) представляет программу и как напечатать его, чтобы проверить парсер.
- Как рекурсивный спуск разбирает инструкции и как разбор Пратта справляется с приоритетом и ассоциативностью операторов.
- Как тестировать парсер через удобочитаемую печатную форму его дерева.

## Перед началом

- Вы прошли предыдущие этапы. Это направление использует перечисления и сопоставление с образцом
  ([Урок 06.3 — Образцы подробно](../06-rust-advanced/03-patterns-in-depth.md)), рекурсию
  ([Урок 07.7 — Рекурсия](../07-algorithms-and-data-structures/07-recursion.md)), деревья
  ([Урок 07.11](../07-algorithms-and-data-structures/11-trees-and-binary-search-trees.md)) и `Box`
  ([Урок 05.9 — Умные указатели](../05-rust-intermediate/09-smart-pointers.md)).
- Код направления живёт в одном пакете — библиотеке с небольшой программой:

```bash
cd ~/projects/rust
cargo new code/p22-pebble
cd code/p22-pebble
mkdir programs
```

Файлы — `src/lexer.rs`, `src/ast.rs` и `src/parser.rs`, объявленные в `src/lib.rs`:

```rust
pub mod ast;
pub mod lexer;
pub mod parser;
```

## Новые слова

| Слово | Значение |
|---|---|
| **компилятор** (compiler) | Программа, которая переводит программу с одного языка на другой, обычно в машинный код или байт-код, не выполняя её. |
| **интерпретатор** (interpreter) | Программа, которая выполняет программу напрямую, шаг за шагом, по её исходному тексту или синтаксическому дереву. |
| **лексер** (lexer, tokenizer) | Первый шаг: он режет исходный текст на **токены** — «слова» языка. |
| **токен** (token) | Один осмысленный кусок исходного текста: число, имя, ключевое слово, оператор, скобка. |
| **парсер** (parser) | Шаг, который проверяет порядок токенов по грамматике и строит синтаксическое дерево. |
| **AST** (Abstract Syntax Tree, абстрактное синтаксическое дерево) | Дерево, которое показывает структуру программы (какой оператор к чему применяется), без пунктуации вроде скобок и точек с запятой. |
| **грамматика** (grammar) | Правила, которые говорят, какие последовательности токенов образуют правильные программы. |
| **рекурсивный спуск** (recursive descent) | Метод разбора с одной функцией на каждое правило грамматики; функции вызывают друг друга (и себя), как это делают правила. |
| **приоритет** (precedence) | Какой оператор связывает сильнее: в `1 + 2 * 3` у `*` приоритет выше, чем у `+`. |
| **ассоциативность** (associativity) | Как группируются операторы одного приоритета: `10 - 3 - 2` — это `(10 - 3) - 2`, поэтому `-` *левоассоциативен*. |
| **разбор Пратта** (Pratt parsing) | Способ разбирать выражения, в котором у каждого оператора есть число — его **сила связывания** (binding power); тогда один цикл справляется со всеми приоритетами и ассоциативностью. |
| **S-выражение** (S-expression) | Запись дерева в скобках, где оператор стоит первым: `(+ 1 (* 2 3))`. |
| **ключевое слово** (keyword) | Слово с фиксированным смыслом, которое не может быть именем переменной, например `let` или `print`. |

## 1. Что делает компилятор

Компилятор или интерпретатор — это не одна большая волшебная программа. Это **конвейер** маленьких
шагов, и каждый шаг превращает одну структуру данных в следующую:

```text
 source text          tokens                  syntax tree (AST)            result
 "print 1 + 2 * 3;" → [Print, Int(1), Plus, → (print (+ 1 (* 2 3)))  ──┬─▶ interpreter: prints 7
                       Int(2), Star, Int(3),                           │    (Lesson 22.23)
                       Semicolon, Eof]                                 └─▶ code generator: a
      lexer ─────────────────▶   parser ──────────────────▶                WebAssembly module
                                                                           (Lesson 22.24)
```

**Лексер** знает о символах: где кончается число, что `>=` — один оператор, что пробелы и комментарии
ничего не значат. **Парсер** знает о структуре: что `*` связывает сильнее `+`, что `let` нужны имя и
`=`. Следующие шаги больше никогда не смотрят на символы; они работают с деревом.

В этом направлении вы за четыре урока построите маленький язык **Pebble**:

| Урок | Шаг |
|---|---|
| 22.22 (этот) | лексер и парсер выражений, `let` и `print` |
| 22.23 | интерпретатор обхода дерева, а также блоки, `if` и `while` |
| 22.24 | WebAssembly: запуск Rust в песочнице и компиляция выражений Pebble в WebAssembly |
| 22.25 | проект: функции, полный интерпретатор и компилятор, проверенные друг другом |

Программа на Pebble выглядит так. Сохраните её как `programs/first.pb`:

```text
// Our first Pebble program.
let width = 6;
let height = 7;
print width * height + 1;
print -(width - 10) >= 4 == true;
```

В Pebble есть 64-битные целые числа и логические значения (`true`, `false`), арифметические операторы
`+ - * / %`, сравнения `== != < <= > >=`, префиксный минус и комментарии, начинающиеся с `//`. `**` и
логические операторы `&& || !` вы добавите в упражнениях.

> 💡 **Совет:** бесплатная онлайн-книга *Crafting Interpreters* Роберта Нистрома (Robert Nystrom) шаг за
> шагом строит два полных интерпретатора. Это лучший спутник этого направления. Pebble меньше и
> придуман для этого курса, поэтому книгу можно читать потом, не повторяясь.

## 2. Лексер

Лексер проходит по символам один раз. В каждой позиции он решает, какой токен там начинается, берёт
все его символы и запоминает **позицию** (строку и столбец) начала токена, чтобы позже сообщения об
ошибках могли сказать, *где* что-то не так. Сохраните как `src/lexer.rs`:

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

**Что происходит:**

1. `TokenKind` перечисляет все виды токенов. Два вида несут данные: `Int(i64)` — значение числа и
   `Ident(String)` — имя переменной. `Token` — это вид плюс позиция, где он начинается.
2. Приватная структура `Lexer` хранит оставшиеся символы как `Peekable<Chars>`: итератор по символам
   исходного текста, у которого `peek()` смотрит на следующий символ, не забирая его.
3. `bump` забирает один символ и обновляет позицию: перевод строки начинает новую строку со столбца 1.
   `bump_if` забирает символ, только если это ожидаемый символ; мы используем его для двухсимвольных
   операторов.
4. В `lex` пробельные символы пропускаются. Цифра начинает число: `take_while` собирает все цифры, а
   `parse` превращает их в `i64` (число, слишком большое для 64 бит, становится `LexError`).
5. Буква или `_` начинает слово. Если слово — ключевое, мы выдаём вид ключевого слова; иначе это
   идентификатор.
6. Для остальных символов ветки `match` с условиями (`'/' if lx.bump_if('/')`) сначала обрабатывают
   двухсимвольные токены: `//` начинает комментарий, который пропускается до конца строки, `==` — один
   токен, одиночный `=` — другой. Всё остальное — ошибка «unexpected character» в его позиции.
7. В конце мы добавляем токен `Eof` (end of file, конец файла), чтобы у парсера всегда был токен, на
   который можно посмотреть.

Чтобы увидеть токены, сохраните эту маленькую программу как `examples/tokens.rs` (Cargo собирает файлы
из `examples/` как дополнительные программы, их запускают с `--example`):

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

Комментарий не дал ни одного токена, `>=` стал одним `GreaterEqual`, и каждый токен знает свои строку
и столбец.

## 3. Синтаксическое дерево

Результат парсера — дерево. Каждый вид узла — вариант перечисления, а дочерние узлы хранятся в `Box`,
потому что перечисление не может содержать само себя напрямую (его размер был бы бесконечным; Урок
05.9). Сохраните как `src/ast.rs`:

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

**Что происходит:**

1. `Expr` — выражение: то, у чего есть значение. `Binary(op, left, right, pos)` — это, например,
   `left + right`; позиция оператора хранится для сообщений об ошибках выполнения в Уроке 22.23.
2. `Stmt` — инструкция: то, что выполняется. В Pebble есть `let name = expr;`, `print expr;` и
   выражение, использованное как инструкция.
3. Реализации `Display` печатают дерево как **S-выражение**. `1 + 2 * 3` печатается как
   `(+ 1 (* 2 3))`: скобки точно показывают, какой оператор к чему применяется. Это самый простой
   способ проверить парсер в тесте.

## 4. Парсер: рекурсивный спуск и Пратт

**Грамматика** говорит, какие последовательности токенов являются программами. Мы записываем её в
короткой нотации, где `*` значит «повторить ноль или больше раз», а `|` значит «или»:

```text
program   := statement* EOF
statement := "let" IDENT "=" expr ";"
           | "print" expr ";"
           | expr ";"
expr      := (handled by Pratt parsing below)
```

**Рекурсивный спуск** превращает каждое правило в одну функцию: `program` вызывает `statement` в цикле;
`statement` смотрит на следующий токен и выбирает ветку. Это просто и даёт хорошие сообщения об
ошибках.

Для выражений тоже подошла бы одна функция на каждый уровень приоритета, но это длинно. **Разбор
Пратта** даёт каждому инфиксному оператору **силу связывания** и обрабатывает их все в одном цикле:

| Операторы | Сила связывания |
|---|---|
| `==` `!=` | 3 |
| `<` `<=` `>` `>=` | 4 |
| `+` `-` | 5 |
| `*` `/` `%` | 6 |
| префиксный `-` | 7 |

Правило цикла: `expr(min_power)` читает один операнд, затем продолжает брать операторы, **пока их сила
больше `min_power`**, разбирая правую часть через `expr(power)`. Посмотрите на `1 + 2 * 3`:

```text
expr(0): operand 1
         sees +, power 5 > 0  → take it, right side = expr(5)
               expr(5): operand 2
                        sees *, power 6 > 5 → take it, right side = expr(6) = 3 → (* 2 3)
                        end of input        → return (* 2 3)
         left = (+ 1 (* 2 3))
```

И на `10 - 3 - 2`:

```text
expr(0): operand 10
         sees -, power 5 > 0  → take it, right side = expr(5)
               expr(5): operand 3
                        sees -, power 5 > 5? no → return 3
         left = (- 10 3)
         sees -, power 5 > 0  → take it, right side = expr(5) = 2
         left = (- (- 10 3) 2)
```

Строгое `>` заставляет операторы одной силы группироваться влево. Сохраните парсер как
`src/parser.rs`:

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

**Что происходит:**

1. `Parser` хранит токены и индекс `at` следующего. `peek` смотрит на него, `advance` забирает его (но
   никогда не уходит дальше `Eof`), `expect` забирает токен одного точного вида или возвращает
   `ParseError`, где названо, что ожидалось и что найдено.
2. `program` и `statement` — это рекурсивный спуск: `statement` выбирает ветку по первому токену и затем
   требует завершающую `;`.
3. `expr` — цикл Пратта, описанный выше. `while let Some((op, power)) = ...` останавливается на любом
   токене, который не является инфиксным оператором, например `;` или `)`.
4. `prefix` разбирает то, с чего может начинаться выражение: число, `true`/`false`, переменную,
   префиксный минус (его операнд разбирается с `PREFIX_POWER`, поэтому `-x * 2` — это `(* (- x) 2)`) или
   группу в скобках, где `expr(0)` снова начинает с самой низкой силы. (Силы 1 и 2 специально оставлены
   свободными: их использует упражнение 3.)
5. `parse` запускает лексер и парсер и превращает оба вида ошибок в одно сообщение с позицией.
6. Тесты проверяют приоритет, ассоциативность, префиксный минус и сообщение об ошибке — всё через форму
   S-выражений.

Наконец, `src/main.rs` читает файл и печатает его дерево:

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

`cargo test` и `cargo run -- programs/first.pb`:

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

Теперь сломанная программа. Сохраните `let x = (1 + 2;` как `programs/broken.pb` и выполните
`cargo run -- programs/broken.pb`:

```text
programs/broken.pb: error at 1:15: expected `)`, found Semicolon
```

> 🧪 **Тестирование:** тесты парсера сравнивают S-выражения, а не структуры Rust: `"(+ 1 (* 2 3))"`
> гораздо легче читать и писать, чем вложенные вызовы `Box::new(...)`. Добавляйте тест на каждую
> найденную в парсере ошибку и тестируйте сообщения об ошибках тоже: они часть пользовательского
> интерфейса языка.

> 🔒 **Безопасность:** парсер — первый код, который касается недоверенного ввода (например, языка
> конфигурации или запроса, присланного по сети). Он никогда не должен паниковать, а очень глубокая
> вложенность, например 100 000 открывающих скобок, может переполнить стек рекурсивного парсера.
> Настоящие парсеры ограничивают глубину вложенности.

## Частые ошибки

**Левая рекурсия в рекурсивном спуске.** Правило, записанное как `sum := sum "+" NUMBER | NUMBER`,
превращается в функцию, которая вызывает себя, ещё не взяв ни одного токена. Сохраните как
`examples/left_recursion.rs` и выполните `cargo run --example left_recursion`:

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

Компилятор предупреждает, а запуск падает:

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

Исправление: сначала разберите один операнд, затем в цикле обходите операторы, как это делает `expr`.
После этого удалите файл.

**`<` вместо `<=` в цикле Пратта.** Замените `if power <= min_power` на `if power < min_power` и
запустите `cargo test`:

```text
---- parser::tests::same_power_operators_group_to_the_left stdout ----

thread 'parser::tests::same_power_operators_group_to_the_left' (6365722) panicked at src/parser.rs:171:9:
assertion `left == right` failed
  left: "(- 10 (- 3 2))"
 right: "(- (- 10 3) 2)"
```

Теперь `10 - 3 - 2` значит `10 - (3 - 2)`, то есть 9 вместо 5. Разница в один символ решает
ассоциативность — именно поэтому существует этот тест.

**Забыть позицию.** Без строки и столбца сообщение «expected `)`» в программе из 500 строк почти
бесполезно. Храните позицию в каждом токене и в каждом узле AST, который может дать ошибку при
выполнении.

## Попробуйте сами

1. Добавьте `print 1 +;` в `programs/first.pb`. Какую позицию показывает ошибка и почему именно эту?
2. Попробуйте `print 1 2;`. Парсер читает `1`, не видит оператора, а затем `expect` хочет `;`, но
   находит `Int(2)`.
3. Напишите `print 2 * -3;`. Проверьте, что дерево — `(print (* 2 (- 3)))`.

## Упражнения

### Упражнение 1 — Удобочитаемые числа (легко)

Разрешите подчёркивания внутри чисел, чтобы `1_000_000` было целым числом миллион. Число по-прежнему
должно начинаться с цифры. Напишите модульный тест в `src/lexer.rs`, который разбирает `"1_000_000 7"`
и проверяет виды токенов.

<details>
<summary>Подсказка</summary>

В ветке цифр позвольте `take_while` принимать также `'_'`, затем уберите подчёркивания через
`text.replace('_', "")` перед `parse`.

</details>

<details>
<summary>Ответ</summary>

Ветка цифр становится такой:

```rust
let text = lx.take_while(|c| c.is_ascii_digit() || c == '_');
let value = text.replace('_', "").parse().map_err(|_| LexError {
    message: format!("number too large: {text}"),
    pos: start,
})?;
```

и в конце `src/lexer.rs`:

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

Почему это работает: в ветку попадают, только если первый символ — цифра, поэтому `_x` остаётся
идентификатором. Сообщение об ошибке сохраняет исходный текст с подчёркиваниями — то, что написал
пользователь.

</details>

### Упражнение 2 — Оператор возведения в степень (средне)

Добавьте `**` для степеней. Он связывает сильнее `*` и **правоассоциативен**: `2 ** 3 ** 2` значит
`2 ** (3 ** 2)`. Префиксный минус применяется ко всей степени: `-2 ** 2` значит `-(2 ** 2)`. Напишите
тесты:

- `2 ** 3 ** 2` → `(** 2 (** 3 2))`
- `2 * 3 ** 2` → `(* 2 (** 3 2))`
- `-2 ** 2` → `(- (** 2 2))`

<details>
<summary>Подсказка</summary>

Нужны токен `StarStar` (`'*' if lx.bump_if('*')`, поставленный перед веткой `'*'`), `BinOp::Pow` с
символом `"**"` и сила связывания 8 (больше `PREFIX_POWER`, чтобы работало `-2 ** 2`). Для правой
ассоциативности разбирайте правую часть с `power - 1` вместо `power`, чтобы оператор той же силы
забирался справа.

</details>

<details>
<summary>Ответ</summary>

`src/lexer.rs`: добавьте `StarStar,` после `Star,` в `TokenKind` и перед веткой `'*'`:

```rust
'*' if lx.bump_if('*') => TokenKind::StarStar,
```

`src/ast.rs`: добавьте `Pow,` в `BinOp` и `BinOp::Pow => "**",` в `symbol`. `src/parser.rs`: добавьте
`TokenKind::StarStar => (BinOp::Pow, 8),` в `infix_binding_power`, а в `expr`:

```rust
let pos = self.advance().pos;
// `**` groups to the right: allow the same power again on the right side.
let right_min = if op == BinOp::Pow { power - 1 } else { power };
let right = self.expr(right_min)?;
```

Тест, в модуле `tests` файла `src/parser.rs`:

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

Почему это работает: внутри `expr(7)` (правая часть первого `**`, с `power - 1 = 7`) у второго `**` сила
8 > 7, поэтому он забирается там, справа. Для `-2 ** 2` операнд префикса разбирается через `expr(7)`, и
`**` (8) забирается внутри него, поэтому минус применяется к степени.

</details>

### Упражнение 3 — Логические операторы (сложно)

Добавьте логические операторы `&&` (и), `||` (или) и префиксный `!` (не). Они связывают слабее любого
сравнения, а `&&` связывает сильнее `||`, поэтому `!a || b && c` значит `(!a) || (b && c)`. Напишите
тест:

- `!a || b && c` → `(|| (! a) (&& b c))`
- `x < 1 || x > 9` → `(|| (< x 1) (> x 9))`

<details>
<summary>Подсказка</summary>

Три новых токена (`Bang`, `AndAnd`, `OrOr`; одиночные `&` и `|` остаются ошибками), три новых случая
AST (`BinOp::And`, `BinOp::Or`, `UnOp::Not`) с их символами и две свободные силы связывания 1 и 2. В
лексере ветка `'!' => TokenKind::Bang` должна идти *после* `'!' if lx.bump_if('=')`.

</details>

<details>
<summary>Ответ</summary>

`src/lexer.rs`: добавьте `Bang,`, `AndAnd,` и `OrOr,` в `TokenKind` и эти ветки (ветку `!` — сразу
после ветки `!=`, остальные — после `'>' => TokenKind::Greater,`):

```rust
'!' => TokenKind::Bang,
'&' if lx.bump_if('&') => TokenKind::AndAnd,
'|' if lx.bump_if('|') => TokenKind::OrOr,
```

`src/ast.rs`: добавьте `And,` и `Or,` в `BinOp` с `BinOp::And => "&&",` и `BinOp::Or => "||",` в
`symbol`; добавьте `Not,` в `UnOp` и в `Display`:

```rust
Expr::Unary(UnOp::Not, e, _) => write!(f, "(! {e})"),
```

`src/parser.rs`: две силы связывания, и ветка префикса принимает оба оператора:

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

и тест:

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

Почему это работает: циклу Пратта всё равно, какие операторы существуют; добавление оператора — это
одна строка в таблице сил связывания. `!` разбирается как префиксный минус, с `PREFIX_POWER`, поэтому
он применяется только к операнду сразу после него.

</details>

## Вопросы для самопроверки

1. Что выдаёт лексер и что выдаёт парсер?

<details>
<summary>Ответ</summary>

Лексер выдаёт список токенов (с позициями). Парсер выдаёт из этих токенов синтаксическое дерево (AST)
или ошибку.

</details>

2. Зачем каждый токен несёт строку и столбец?

<details>
<summary>Ответ</summary>

Чтобы парсер (а позже интерпретатор) мог сообщить, *где* ошибка. Сообщения об ошибках без позиции почти
бесполезны в длинной программе.

</details>

3. Что решают приоритет и ассоциативность? Приведите пример каждого.

<details>
<summary>Ответ</summary>

Приоритет решает, какой оператор связывает сильнее: `1 + 2 * 3` — это `1 + (2 * 3)`. Ассоциативность
решает, как группируются операторы одного приоритета: `10 - 3 - 2` — это `(10 - 3) - 2`.

</details>

4. Почему в цикле Пратта сравнение для остановки — `power <= min_power`, а не `power < min_power`?

<details>
<summary>Ответ</summary>

С `<=` оператор той же силы, что и слева, останавливает внутренний вызов, поэтому его забирает внешний
цикл: операторы группируются влево. С `<` его забирает внутренний вызов, и они группируются вправо.

</details>

5. Почему дочерние выражения в `Expr` хранятся в `Box<Expr>`?

<details>
<summary>Ответ</summary>

Перечисление, которое напрямую содержит само себя, имело бы бесконечный размер. `Box` — указатель
фиксированного размера на значение в куче, поэтому у перечисления известный размер.

</details>

6. Что такое левая рекурсия и почему она ломает рекурсивный спуск?

<details>
<summary>Ответ</summary>

Правило, которое начинается с самого себя, например `sum := sum "+" NUMBER`. Его функция вызывает себя,
ещё не взяв ни одного токена, поэтому рекурсия бесконечна и переполняет стек. Сначала разберите
операнд, затем — цикл.

</details>

## Итоги

- Компилятор или интерпретатор — это конвейер: исходный текст → лексер → токены → парсер →
  синтаксическое дерево → интерпретатор или генератор кода.
- Лексер работает с символами, пропускает пробелы и комментарии и записывает позицию каждого токена.
- AST — дерево значений перечислений с дочерними узлами в `Box`; печать в виде S-выражений делает тесты
  парсера удобочитаемыми.
- Рекурсивный спуск — одна функция на правило грамматики, идеален для инструкций.
- Разбор Пратта даёт каждому оператору силу связывания; `expr(min_power)` со строгим сравнением
  обрабатывает приоритет и левую ассоциативность одним циклом, а `power - 1` даёт правую
  ассоциативность.
- Избегайте левой рекурсии, храните позиции для сообщений об ошибках и тестируйте каждое правило
  приоритета.

## Источники и что почитать

- [Crafting Interpreters](https://craftinginterpreters.com/) — бесплатная книга Robert Nystrom; главы 4–6 о лексическом анализе, синтаксических деревьях и разборе.
- [Simple but Powerful Pratt Parsing](https://matklad.github.io/2020/04/13/simple-but-powerful-pratt-parsing.html) — понятная статья о разборе Пратта на Rust от автора rust-analyzer.
- [logos documentation](https://docs.rs/logos/0.16/logos/) — генератор лексеров: вы описываете токены шаблонами, а он пишет быстрый лексер; альтернатива написанному вручную.
- [The Rust Reference — Tokens](https://doc.rust-lang.org/reference/tokens.html) — как собственный лексер Rust определяет токены; хороший пример точной лексической грамматики.
- [The Rust Reference — Expression precedence](https://doc.rust-lang.org/reference/expressions.html) — таблица приоритетов Rust для сравнения с Pebble.

---

[🏠 Начало](../START-HERE.md) · [← 22.21 Трек сетей: проект и следующие шаги](21-networking-track-project-and-next-steps.md) · [📋 Этап 22](README.md) · [22.23 Трек компиляторов: интерпретатор обхода дерева →](23-compilers-track-an-interpreter.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
