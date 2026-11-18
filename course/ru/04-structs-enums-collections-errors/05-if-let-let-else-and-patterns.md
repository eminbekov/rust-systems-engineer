<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 04.4 Option и match](04-option-and-match.md) · [📋 Этап 04](README.md) · [04.6 Модули, крейты и пакеты →](06-modules-crates-and-packages.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/04-structs-enums-collections-errors/05-if-let-let-else-and-patterns.md) <!-- nav -->

# Урок 04.5 — if let, let else и образцы

> **Этап 04 — Структуры, перечисления, коллекции и ошибки** · Урок 5 из 12 · ⏱ около 3,5 часа (2 занятия)

## Что вы узнаете

- `if let`: короткая форма `match`, которому важен один случай.
- `while let`: цикл, который работает, пока образец продолжает подходить.
- `let else`: разобрать значение на части или досрочно выйти, если оно не подходит.
- Макрос `matches!` и «цепочки let» (несколько условий в одном `if let`).
- Образцы везде: в `let`, в параметрах функций, в циклах `for`; опровергаемые и неопровергаемые образцы; образцы срезов.

## Перед началом

- Вы прошли [Урок 04.4 — Option и match](04-option-and-match.md). Вы знаете `Option`, `match` и исчерпывающность.
- Вы знаете, как выглядит `Result` (`Ok(value)` / `Err(error)`); полную историю рассказывает [Урок 04.10 — Обработка ошибок](10-error-handling.md).

## Новые слова

| Слово | Значение |
|---|---|
| **`if let`** | `if let ОБРАЗЕЦ = ЗНАЧЕНИЕ { ... }`: выполнить блок, только если значение подходит к образцу. |
| **`while let`** | `while let ОБРАЗЕЦ = ЗНАЧЕНИЕ { ... }`: повторять блок, пока значение подходит к образцу. |
| **`let else`** | `let ОБРАЗЕЦ = ЗНАЧЕНИЕ else { ... };`: привязать переменные, если образец подходит, иначе выполнить блок `else`, который обязан покинуть текущий путь выполнения кода. |
| **опровергаемый образец** (refutable pattern) | Образец, который может не подойти, например `Some(x)` (значение может быть `None`). |
| **неопровергаемый образец** (irrefutable pattern) | Образец, который подходит всегда, например `x`, `(a, b)` или `Point { x, y }`. |
| **расходиться** (diverge) | Никогда не продолжаться обычным образом: `return`, `break`, `continue` и `panic!` расходятся. |
| **цепочка let** (let chain) | Несколько условий, соединённых через `&&` в одном `if` или `while`, где часть из них может быть образцами `let`. |
| **деструктуризация** (destructuring) | Разбор составного значения (кортежа, структуры, перечисления, массива) на части с помощью образца. |
| **образец среза** (slice pattern) | Образец, сопоставляющий элементы массива или среза, например `[first, .., last]`. |
| **`Vec`** (вектор) | Растущий список значений одного типа. У него отдельный урок, [Урок 04.7 — Векторы](07-vectors.md); здесь мы используем только `push` и `pop`. |
| **стек** (как структура данных) | Стопка, в которую добавляют и из которой убирают только сверху: последний добавленный элемент убирается первым. (Не то же самое, что стек памяти из Урока 03.1.) |
| **макрос** (macro) | Код, который пишет код. Вызывается через `!`, например `println!` или `matches!`. |
| **линт** (lint) | Проверка, находящая подозрительный или неидиоматичный код и предупреждающая о нём. `cargo clippy` запускает много линтов. |

## 1. `if let`: `match` с одной интересной ветвью

Иногда вам важен только один случай. Полный `match` заставляет писать и скучную ветку:

```rust
fn main() {
    let config_port: Option<u16> = Some(8080);

    // через match: вторая ветвь нужна лишь для того, чтобы удовлетворить компилятор
    match config_port {
        Some(port) => println!("match: port {port}"),
        _ => (),
    }

    // через if let: короче
    if let Some(port) = config_port {
        println!("if let: port {port}");
    }

    // с else
    let nothing: Option<u16> = None;
    if let Some(port) = nothing {
        println!("port {port}");
    } else {
        println!("no port configured, using the default");
    }
}
```

**Что происходит:**

1. В `match` ветка `_ => ()` значит «в остальных случаях ничего не делать». `()` — значение «unit»: отсутствие значения.
2. `if let Some(port) = config_port { ... }` читается так: «если `config_port` подходит к образцу `Some(port)`, привязать `port` и выполнить блок». Обратите внимание на одиночное `=`: это не сравнение, а сопоставление с образцом. Образец слева, значение справа.
3. Если образец не подходит, блок пропускается. Вместо него выполняется необязательный блок `else`.
4. Переменная `port` существует только внутри блока `if let`.

```text
match: port 8080
if let: port 8080
no port configured, using the default
```

`if let` работает и с вашими собственными перечислениями, и в цепочках `else if let`:

```rust
#[derive(Debug)]
enum Command {
    Quit,
    Move { x: i32, y: i32 },
    Say(String),
}

fn main() {
    let commands = [
        Command::Say(String::from("hi")),
        Command::Move { x: 4, y: 9 },
        Command::Quit,
    ];

    for command in &commands {
        if let Command::Say(text) = command {
            println!("someone said {text:?}");
        } else if let Command::Move { x, .. } = command {
            println!("moved horizontally to {x}");
        } else {
            println!("something else: {:?}", command);
        }
    }
}
```

**Что происходит:**

1. `command` — это `&Command` (мы идём по `&commands`), поэтому переменные образцов `text` и `x` — ссылки. Из массива ничего не перемещается.
2. `Command::Move { x, .. }` берёт только `x` и игнорирует остальные поля через `..`.
3. Цепочка `if let ... else if let ... else` проверяет случаи один за другим. Цикл печатает по одной строке на команду.

```text
someone said "hi"
moved horizontally to 4
something else: Quit
```

> 📏 **Соглашение:** используйте `if let`, когда обрабатываете **один** случай и игнорируете остальные. Используйте `match`, когда обрабатываете **два и более** случая или хотите, чтобы компилятор проверил, что вы охватили всё (`if let` на полноту не проверяется никогда). `cargo clippy` знает это правило. На первом примере он говорит:
>
> ```text
> warning: you seem to be trying to use `match` for destructuring a single pattern. Consider using `if let`
>  --> src/main.rs:3:5
>   |
> 3 | /     match port {
> 4 | |         Some(p) => println!("port {p}"),
> 5 | |         _ => (),
> 6 | |     }
>   | |_____^ help: try: `if let Some(p) = port { println!("port {p}") }`
>   |
>   = note: `#[warn(clippy::single_match)]` on by default
> ```

## 2. `while let`: цикл, пока образец подходит

`while let` повторяется, пока образец подходит. Классическое применение — опустошение **стека**. Здесь мы используем `Vec` — растущий список, который объясняет [Урок 04.7](07-vectors.md). Сейчас нужно знать только: `vec![1, 2, 3]` создаёт список, а `.pop()` убирает последний элемент и возвращает его как `Option` (`None`, когда список пуст).

```rust
fn main() {
    let mut stack = vec![1, 2, 3];
    while let Some(top) = stack.pop() {
        println!("popped {top}");
    }
    println!("stack is empty now: {:?}", stack);

    let mut letters = "abc".chars();
    while let Some(c) = letters.next() {
        println!("letter {c}");
    }
}
```

**Что происходит:**

1. `stack.pop()` возвращает `Some(3)`, затем `Some(2)`, затем `Some(1)`, затем `None`.
2. Пока он даёт `Some(top)`, образец подходит и тело выполняется. Когда он даёт `None`, образец не подходит и цикл заканчивается. Ручная проверка «пусто ли?» не нужна.
3. Второй цикл делает то же с `next()` по символам текста: `next()` возвращает `Some(char)`, пока текст не закончится.

```text
popped 3
popped 2
popped 1
stack is empty now: []
letter a
letter b
letter c
```

> 💡 **Совет:** для обхода списка обычно пишут `for item in &list`. `while let` нужен, когда тело цикла ещё и изменяет то, откуда берёт элементы (как извлечение из стека), или когда число шагов заранее неизвестно.

## 3. `let else`: получить или выйти

Очень часто хочется сказать: «достань значение из этого `Option`/`Result`, а если его нет — остановись здесь». С `match` или `if let` это приводит к вложенности: основной путь оказывается сдвинут всё глубже и глубже. **`let else`** сохраняет основной путь плоским.

```rust
fn describe(text: &str) -> String {
    let Ok(port) = text.parse::<u16>() else {
        return format!("{text:?} is not a port number");
    };
    format!("port {port} is {}", if port < 1024 { "privileged" } else { "free to use" })
}

fn main() {
    println!("{}", describe("80"));
    println!("{}", describe("8080"));
    println!("{}", describe("http"));
    println!("{}", describe("70000"));

    for line in ["10", "x", "20"] {
        let Ok(n) = line.parse::<i32>() else {
            println!("skipping {line:?}");
            continue;
        };
        println!("got {n}");
    }
}
```

**Что происходит:**

1. `text.parse::<u16>()` пытается прочитать число от `0` до `65535` (допустимый номер порта). Он возвращает `Result`.
2. `let Ok(port) = ... else { ... };` означает: «если результат подходит к `Ok(port)`, создать переменную `port` и идти дальше. Иначе выполнить блок `else`».
3. Блок `else` **обязан покинуть** обычный поток выполнения кода: `return`, `continue`, `break` или `panic!`. Здесь первая функция возвращает текст ошибки, а цикл в конце делает `continue` и переходит к следующей строке.
4. После оператора `let else` переменная `port` (или `n`) — обычная переменная до конца функции. Лишнего отступа нет.
5. `{text:?}` печатает текст в кавычках, поэтому пустой текст или текст из пробелов виден. `"70000"` — число, но слишком большое для `u16`, поэтому оно тоже не проходит.

```text
port 80 is privileged
port 8080 is free to use
"http" is not a port number
"70000" is not a port number
got 10
skipping "x"
got 20
```

Если блок `else` может «провалиться» дальше, компилятор откажет:

```rust
let Ok(n) = text.parse::<i32>() else {
    println!("not a number");
};
```

```text
error[E0308]: `else` clause of `let...else` does not diverge
 --> src/main.rs:3:42
  |
3 |       let Ok(n) = text.parse::<i32>() else {
  |  __________________________________________^
4 | |         println!("not a number");
5 | |     };
  | |_____^ expected `!`, found `()`
  |
  = note:   expected type `!`
          found unit type `()`
  = help: try adding a diverging expression, such as `return` or `panic!(..)`
  = help: ...or use `match` instead of `let...else`
```

Если бы блок не выходил, у `n` после него не было бы значения. Сообщение называет выход «расхождением» (diverging); тип `!` (читается «never», «никогда») — это тип выражения, которое никогда не завершается обычным образом, например `return` или `panic!()`.

> 📏 **Соглашение:** стиль «раннего возврата»: сначала проверяйте необычные случаи и выходите, чтобы основная логика не тонула во вложенных `if`. `let else` — инструмент для этого.

> 💡 **Совет:** если ваш блок `else` делает только `return None` (для `Option`) или `return Err(e)`, есть более короткий путь — оператор `?`. `cargo clippy` предложит его (`clippy::question_mark`). Оператор `?` вы изучите в [Уроке 04.10](10-error-handling.md). Используйте `let else`, когда блок `else` делает настоящую работу: строит сообщение, пишет лог или продолжает цикл.

## 4. `matches!` и цепочки let

### `matches!`

Когда нужен только ответ «да/нет» — «подходит ли это значение к этому образцу?» — используйте макрос `matches!`:

```rust
fn is_vowel(c: char) -> bool {
    matches!(c, 'a' | 'e' | 'i' | 'o' | 'u')
}

fn main() {
    println!("{} {}", is_vowel('e'), is_vowel('z'));

    let score = Some(7);
    println!("{}", matches!(score, Some(n) if n > 5));
    println!("{}", matches!(score, None));

    let word = "education";
    let vowels = word.chars().filter(|c| is_vowel(*c)).count();
    println!("{word} has {vowels} vowels");
}
```

**Что происходит:**

1. `matches!(value, pattern)` даёт `true`, если значение подходит к образцу, иначе `false`. Это короткий `match` с ветками `true`/`false`.
2. В образце можно использовать `|` (или) и охранное условие (`if n > 5`), точно как в ветке `match`.
3. `word.chars().filter(...).count()` считает символы, для которых замыкание даёт `true` (итераторы: [Урок 05.5](../05-rust-intermediate/05-iterators.md)).

```text
true false
true
false
education has 5 vowels
```

### Цепочки let

Начиная с Rust 1.88 (редакция 2024, то есть редакция этого курса) `if` или `while` могут соединять несколько условий и образцов `let` через `&&`. Без этого понадобились бы вложенные `if let`:

```rust
fn main() {
    let width: Option<u32> = Some(12);
    let height: Option<u32> = Some(5);

    if let Some(w) = width
        && let Some(h) = height
        && w > h
    {
        println!("wide: {w} x {h}");
    }

    let age: Option<u32> = Some(17);
    if let Some(a) = age && a >= 18 {
        println!("adult");
    } else {
        println!("not an adult, or unknown");
    }
}
```

**Что происходит:**

1. В первом `if` три части, соединённые `&&`: два образца и обычное условие. Они проверяются слева направо; если одна не проходит, остальные пропускаются. Переменные `w` и `h` доступны в следующих частях и в блоке.
2. Второй `if` сочетает образец с условием. Если `age` равно `None` или меньше 18, выполняется блок `else`.

```text
wide: 12 x 5
not an adult, or unknown
```

## 5. Образцы повсюду

Вы использовали образцы в `match`. На самом деле образцы встречаются во многих местах Rust. Даже обычный `let` — уже образец!

```rust
struct Point {
    x: i32,
    y: i32,
}

fn sum_pair(&(a, b): &(i32, i32)) -> i32 {
    a + b
}

fn main() {
    // кортежи
    let (name, age) = ("Zed", 41);
    println!("{name} is {age}");

    // вложенные образцы
    let ((a, b), c) = ((1, 2), 3);
    println!("{}", a + b + c);

    // структура с литералом внутри
    let p = Point { x: 0, y: 5 };
    match p {
        Point { x: 0, y } => println!("on the y axis at {y}"),
        Point { x, y: 0 } => println!("on the x axis at {x}"),
        Point { x, y } => println!("at ({x}, {y})"),
    }

    // параметры функций
    println!("{}", sum_pair(&(3, 4)));

    // циклы for
    for (i, ch) in "hey".chars().enumerate() {
        println!("{i}: {ch}");
    }
    for &(from, to) in &[(1, 2), (2, 3)] {
        println!("{from} -> {to}");
    }

    // игнорирование частей
    let (first, _, third) = (10, 20, 30);
    println!("{first} {third}");
    let (head, ..) = (1, 2, 3, 4);
    println!("{head}");
}
```

**Что происходит:**

1. `let (name, age) = ("Zed", 41);` **деструктурирует** кортеж на две переменные.
2. Образцы можно вкладывать на любую глубину: `((a, b), c)` разбирает кортеж, содержащий кортеж.
3. В образце структуры можно поставить литерал для поля (`x: 0`), чтобы подходили только определённые значения, и имя (`y`), чтобы захватить остальное. Как и в `match`, ветки пробуются по порядку.
4. Параметр функции может быть образцом: `&(a, b): &(i32, i32)` принимает ссылку на пару и деструктурирует её прямо в списке параметров. `&` в образце означает «значение — ссылка; посмотри на то, на что она указывает».
5. В цикле `for` `(i, ch)` разбирает каждую пару; `.enumerate()` выдаёт пары `(индекс, элемент)`. В `for &(from, to) in &[...]` образец `&` копирует числа из ссылки. (Эта форма `&n` — то, что вы видели в `for &n in numbers` в Уроке 04.4.)
6. `_` игнорирует одно значение; `..` игнорирует «всё остальное» (в кортеже — остальные позиции, в структуре — остальные поля).

```text
Zed is 41
6
on the y axis at 5
7
0: h
1: e
2: y
1 -> 2
2 -> 3
10 30
1
```

### Опровергаемые и неопровергаемые образцы

Некоторые образцы подходят всегда (**неопровергаемые**): `x`, `(a, b)`, `Point { x, y }`. Другие могут не подойти (**опровергаемые**): `Some(x)` не подходит к `None`, `Ok(n)` — к `Err`.

- Обычный `let`, параметры функций и циклы `for` требуют **неопровергаемых** образцов: что им делать, если значение не подошло?
- `if let`, `while let`, `let else` и ветки `match` принимают **опровергаемые** образцы: у них есть план на случай несовпадения.

```rust
let maybe: Option<i32> = Some(3);
let Some(x) = maybe;
```

```text
error[E0005]: refutable pattern in local binding
 --> src/main.rs:3:9
  |
3 |     let Some(x) = maybe;
  |         ^^^^^^^ pattern `None` not covered
  |
  = note: `let` bindings require an "irrefutable pattern", like a `struct` or an `enum` with only one variant
  = note: for more information, visit https://doc.rust-lang.org/book/ch19-02-refutability.html
  = note: the matched value is of type `Option<i32>`
help: you might want to use `let...else` to handle the variant that isn't matched
  |
3 |     let Some(x) = maybe else { todo!() };
  |                         ++++++++++++++++
```

Компилятор даже предлагает лекарство: `let else`.

### Образцы срезов

Образец может также разбирать массив или срез ([Урок 03.6 — Срезы и строковые типы](../03-ownership-and-borrowing/06-slices-and-string-types.md)):

```rust
fn describe(slice: &[i32]) -> String {
    match slice {
        [] => String::from("empty"),
        [x] => format!("one element: {x}"),
        [first, second] => format!("two elements: {first} and {second}"),
        [first, .., last] => format!("starts with {first}, ends with {last}"),
    }
}

fn sum(slice: &[i32]) -> i32 {
    match slice {
        [] => 0,
        [first, rest @ ..] => first + sum(rest),
    }
}

fn main() {
    println!("{}", describe(&[]));
    println!("{}", describe(&[7]));
    println!("{}", describe(&[7, 8]));
    println!("{}", describe(&[7, 8, 9, 10]));
    println!("{}", sum(&[1, 2, 3, 4]));
}
```

**Что происходит:**

1. `[]` подходит к пустому срезу; `[x]` — ровно к одному элементу; `[first, second]` — ровно к двум.
2. `[first, .., last]` подходит к любому срезу не короче двух элементов и берёт первый и последний. `..` означает «любое число элементов между ними».
3. В `sum` образец `[first, rest @ ..]` делит срез на первый элемент и *остаток как более короткий срез*. Функция вызывает сама себя с `rest` — это **рекурсия** (функция, вызывающая саму себя). Она останавливается на пустом срезе, сумма которого равна 0.
4. Компилятор проверяет, что четыре ветки `describe` охватывают любую длину, так что `match` исчерпывающий.

```text
empty
one element: 7
two elements: 7 and 8
starts with 7, ends with 10
10
```

### `_` и `_name` — не одно и то же

Одиночное подчёркивание игнорирует значение и **ничего не привязывает**. Имя, начинающееся с подчёркивания (`_b`), — **настоящая** переменная (подчёркивание лишь гасит предупреждение «неиспользуемая переменная»). Разница важна для владения:

```rust
fn main() {
    let a = String::from("kept");
    let _ = a; // `_` ничего не привязывает, поэтому ничего не перемещается
    println!("{a} is still here");

    let b = String::from("moved");
    let _b = b; // `_b` — настоящая переменная, поэтому b перемещается в неё
    println!("{_b}");
}
```

```text
kept is still here
moved
```

Если вывести `b` после `let _b = b;`, вы получите привычное `error[E0382]: borrow of moved value: `b``.

## 6. Какой инструмент выбрать?

| Вы хотите … | Используйте |
|---|---|
| обработать каждый вариант или случай и чтобы компилятор проверил полноту | `match` |
| выполнить действие для **одного** образца и игнорировать остальное | `if let` (с `else` для противоположного случая) |
| повторять, пока образец подходит (`pop`, `next`) | `while let` |
| достать значение или выйти из функции/цикла, если его нет | `let else` |
| получить ответ «да/нет» об образце | `matches!` |
| сочетать образец с дополнительными условиями | цепочка let (`if let ... && cond`) |

## 7. Тестирование

> 🧪 **Тестирование:** у кода с `let else` два выхода — обычный и ранний — и **каждому нужен тест**. `matches!` удобен в проверках, когда важен *вид* значения, а не каждая деталь: `assert!(matches!(value, Some(Shape::Circle(_))))`.

```rust
#[derive(Debug)]
enum Shape {
    Circle(f64),
    Square(f64),
}

fn describe(text: &str) -> String {
    let Ok(port) = text.parse::<u16>() else {
        return format!("{text:?} is not a port number");
    };
    format!("port {port} is {}", if port < 1024 { "privileged" } else { "free to use" })
}

fn make_shape(kind: &str, size: f64) -> Option<Shape> {
    match kind {
        "circle" => Some(Shape::Circle(size)),
        "square" => Some(Shape::Square(size)),
        _ => None,
    }
}

fn main() {
    println!("{}", describe("80"));
    println!("{:?}", make_shape("circle", 2.0));
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn valid_ports() {
        assert_eq!(describe("80"), "port 80 is privileged");
        assert_eq!(describe("8080"), "port 8080 is free to use");
    }

    #[test]
    fn invalid_ports() {
        assert_eq!(describe("http"), "\"http\" is not a port number");
        assert_eq!(describe("70000"), "\"70000\" is not a port number");
        assert_eq!(describe(""), "\"\" is not a port number");
    }

    #[test]
    fn matches_checks_the_variant_only() {
        assert!(matches!(make_shape("circle", 2.0), Some(Shape::Circle(_))));
        assert!(matches!(make_shape("square", 1.0), Some(Shape::Square(s)) if s > 0.5));
        assert!(matches!(make_shape("hexagon", 1.0), None));
    }
}
```

**Что происходит:**

1. `valid_ports` идёт обычным выходом `describe`; `invalid_ports` три раза идёт ранним выходом, включая пустой текст и число вне диапазона (граничные случаи).
2. `matches_checks_the_variant_only` проверяет, какой вариант вернулся, не сравнивая нагрузку, — а третья проверка показывает, что `matches!(x, None)` тоже работает.
3. (Cargo печатает безвредные предупреждения «field is never read» для этой демонстрации; здесь они опущены.)

```text
running 3 tests
...
test result: ok. 3 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

> 🔒 **Безопасность:** `let else` делает проверку недоверенного ввода простой для написания и чтения: каждая проверка — одна строка, а всё, что после неё, может полагаться на проверенное значение. Предпочитайте список таких проверок одному глубоко вложенному блоку.

## Частые ошибки

> ⚠️ **Частая ошибка:** писать `==`, когда нужен образец: `if Some(x) == value` не привязывает `x`. Используйте `if let Some(x) = value`.

> ⚠️ **Частая ошибка:** блок `let else`, который не выходит (`E0308`, раздел 3). Завершайте его через `return`, `continue`, `break` или `panic!`.

> ⚠️ **Частая ошибка:** опровергаемый образец в обычном `let` (`E0005`, раздел 5). Используйте `let else`, `if let` или `match`.

> ⚠️ **Частая ошибка:** ждать, что компилятор проверит на полноту цепочку `if let ... else if let ...`. Он этого не делает; проверяется только `match`. Если позже вы добавите новый вариант перечисления, такая цепочка молча его проигнорирует.

> ⚠️ **Частая ошибка:** `let _x = value;`, когда вы хотели проигнорировать значение, не перемещая его. Используйте `let _ = value;` или просто не привязывайте.

> ⚠️ **Частая ошибка:** бесконечный `while let`, потому что значение никогда не обновляется, например `while let Some(x) = list.first() { ... }` с телом, которое никогда не убирает элемент. Следите, чтобы каждая итерация меняла то, что увидит следующая проверка (как `pop`).

## Попробуйте сами

1. Создайте проект:

   ```text
   cd ~/projects/rust
   cargo new code/p04-patterns
   cd code/p04-patterns
   ```

2. Поместите в `src/main.rs`:

   ```rust
   fn main() {
       let inputs = ["42", "abc", "7", "", "-3"];
       let mut total = 0;

       for text in inputs {
           let Ok(number) = text.parse::<i32>() else {
               println!("skipping {text:?}");
               continue;
           };
           total += number;
       }

       println!("total = {total}");
   }
   ```

3. Запустите `cargo run`. Ожидаемый вывод:

   ```text
   skipping "abc"
   skipping ""
   total = 46
   ```

4. Измените программу так, чтобы числа меньше нуля тоже пропускались, используя цепочку let: `if let Ok(n) = text.parse::<i32>() && n >= 0 { total += n; } else { ... }`. Какой вариант читается лучше, `let else` или цепочка? (Оба нормальны; важно выработать привычку сравнивать.)
5. Запустите `cargo clippy` и `cargo fmt`; исправьте всё, о чём они сообщат. Затем сделайте коммит:

   ```text
   cd ~/projects/rust
   git add code/p04-patterns
   git commit -m "feat(p04): let else and patterns practice"
   ```

## Упражнения

### Упражнение 1 — Сообщать только о чётных числах (легко)

Напишите `fn describe_even(value: Option<i32>) -> String`, которая возвращает `"N is even"`, когда значение — `Some(N)` с чётным `N`, и `"nothing to report"` во всех остальных случаях. Используйте `if let` с цепочкой let. Протестируйте чётное, нечётное и `None`.

<details>
<summary>Подсказка</summary>

`if let Some(n) = value && n % 2 == 0 { return format!(...); }`, затем верните запасной текст после `if`.

</details>

<details>
<summary>Ответ</summary>

```rust
fn describe_even(value: Option<i32>) -> String {
    if let Some(n) = value
        && n % 2 == 0
    {
        return format!("{n} is even");
    }
    String::from("nothing to report")
}

fn main() {
    println!("{}", describe_even(Some(4)));
    println!("{}", describe_even(Some(5)));
    println!("{}", describe_even(None));
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn even_number_is_reported() {
        assert_eq!(describe_even(Some(4)), "4 is even");
    }

    #[test]
    fn odd_or_missing_is_not() {
        assert_eq!(describe_even(Some(5)), "nothing to report");
        assert_eq!(describe_even(None), "nothing to report");
    }
}
```

Вывод `cargo run`:

```text
4 is even
nothing to report
nothing to report
```

Почему это работает: тело `if` выполняется, только когда образец подходит **и** условие истинно. Во всех остальных случаях функция проваливается на последнюю строку. (Без цепочек let вы написали бы вложенный `if` внутри `if let`, а `cargo clippy` предложил бы его объединить.)

</details>

### Упражнение 2 — Сумма через образцы срезов (средне)

Напишите `fn sum(slice: &[i32]) -> i32` с помощью `match` с образцами срезов `[]` и `[first, rest @ ..]` и рекурсии. Протестируйте обычный список, список с нулевой суммой и пустой список.

<details>
<summary>Подсказка</summary>

Для `[]` сумма равна `0`. Для `[first, rest @ ..]` сумма равна `first + sum(rest)`; `first` — ссылка на число, а прибавление числа к ссылке на число работает.

</details>

<details>
<summary>Ответ</summary>

```rust
fn sum(slice: &[i32]) -> i32 {
    match slice {
        [] => 0,
        [first, rest @ ..] => first + sum(rest),
    }
}

fn main() {
    println!("{}", sum(&[1, 2, 3, 4]));
    println!("{}", sum(&[]));
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn sums_numbers() {
        assert_eq!(sum(&[1, 2, 3, 4]), 10);
        assert_eq!(sum(&[-5, 5]), 0);
    }

    #[test]
    fn empty_sum_is_zero() {
        assert_eq!(sum(&[]), 0);
    }
}
```

Вывод `cargo run`:

```text
10
0
```

Почему это работает: каждый вызов отделяет первый элемент и обрабатывает более короткий срез, пока срез не станет пустым. Два образца вместе охватывают любой возможный срез, поэтому `match` исчерпывающий. (Для очень длинных срезов цикл или `.iter().sum()` лучше рекурсии, потому что каждый вызов занимает память стека.)

</details>

### Упражнение 3 — Разбор строк `key=value` (средне)

Напишите `fn parse_kv(line: &str) -> Result<(String, i32), String>` для строк вроде `"retries = 3"`. Используйте `split_once('=')` и `let else` дважды: если нет `=`, верните `Err` с сообщением, а если значение не число — `Err` с другим сообщением. Протестируйте одну хорошую строку и оба вида плохих.

`Result` объясняется в Уроке 04.10; пока достаточно знать: `Ok(value)` — успех, `Err(message)` — неудача с причиной.

<details>
<summary>Подсказка</summary>

`let Some((key, value)) = line.split_once('=') else { return Err(...); };` Затем `let Ok(number) = value.trim().parse::<i32>() else { return Err(...); };` В конце `Ok((key.trim().to_string(), number))`.

</details>

<details>
<summary>Ответ</summary>

```rust
fn parse_kv(line: &str) -> Result<(String, i32), String> {
    let Some((key, value)) = line.split_once('=') else {
        return Err(format!("missing '=' in {line:?}"));
    };
    let Ok(number) = value.trim().parse::<i32>() else {
        return Err(format!("{:?} is not a number", value.trim()));
    };
    Ok((key.trim().to_string(), number))
}

fn main() {
    println!("{:?}", parse_kv("retries = 3"));
    println!("{:?}", parse_kv("broken"));
    println!("{:?}", parse_kv("a=b"));
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn valid_line() {
        assert_eq!(parse_kv("retries = 3"), Ok((String::from("retries"), 3)));
    }

    #[test]
    fn invalid_lines() {
        assert_eq!(parse_kv("broken"), Err(String::from("missing '=' in \"broken\"")));
        assert_eq!(parse_kv("a=b"), Err(String::from("\"b\" is not a number")));
    }
}
```

Вывод `cargo run`:

```text
Ok(("retries", 3))
Err("missing '=' in \"broken\"")
Err("\"b\" is not a number")
```

Почему это работает: каждый `let else` проверяет одну вещь и выходит с конкретным сообщением, поэтому путь успеха читается строго сверху вниз без вложенности. Поскольку блоки `else` строят сообщения, `clippy` здесь не предлагает `?`.

</details>

### Упражнение 4 — Крошечный интерпретатор команд (сложно)

Объявите `enum Command { Add(i32), Reset, Print }` (выведите `Debug` и `PartialEq`). Напишите `fn parse_command(line: &str) -> Option<Command>`:

- `"add 5"` даёт `Add(5)`, `"reset"` даёт `Reset`, `"print"` даёт `Print`;
- всё остальное (пустой текст, `"add"`, `"add x"`, `"reset now"`, `"bogus"`) даёт `None`.

Затем напишите `fn apply(total: i32, command: &Command) -> i32` (`Add` прибавляет, `Reset` возвращает 0, `Print` печатает `total = N` и возвращает сумму без изменений). Протестируйте корректные и некорректные строки, а также `apply`.

<details>
<summary>Подсказка</summary>

Разбейте строку через `let mut parts = line.split_whitespace();` и сопоставьте кортеж из трёх вызовов: `match (parts.next(), parts.next(), parts.next()) { (Some("add"), Some(arg), None) => ..., (Some("reset"), None, None) => ..., ... , _ => None }`. Третий `None` отвергает лишние слова, как в `"reset now"`.

</details>

<details>
<summary>Ответ</summary>

```rust
#[derive(Debug, PartialEq)]
enum Command {
    Add(i32),
    Reset,
    Print,
}

fn parse_command(line: &str) -> Option<Command> {
    let mut parts = line.split_whitespace();
    match (parts.next(), parts.next(), parts.next()) {
        (Some("add"), Some(arg), None) => match arg.parse::<i32>() {
            Ok(n) => Some(Command::Add(n)),
            Err(_) => None,
        },
        (Some("reset"), None, None) => Some(Command::Reset),
        (Some("print"), None, None) => Some(Command::Print),
        _ => None,
    }
}

fn apply(total: i32, command: &Command) -> i32 {
    match command {
        Command::Add(n) => total + n,
        Command::Reset => 0,
        Command::Print => {
            println!("total = {total}");
            total
        }
    }
}

fn main() {
    let mut total = 0;
    for line in ["add 5", "add 7", "print", "bogus", "add x", "reset", "print"] {
        match parse_command(line) {
            Some(command) => total = apply(total, &command),
            None => println!("cannot understand {line:?}"),
        }
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn parses_valid_commands() {
        assert_eq!(parse_command("add 5"), Some(Command::Add(5)));
        assert_eq!(parse_command("add -2"), Some(Command::Add(-2)));
        assert_eq!(parse_command("reset"), Some(Command::Reset));
        assert_eq!(parse_command("print"), Some(Command::Print));
    }

    #[test]
    fn rejects_invalid_commands() {
        assert_eq!(parse_command(""), None);
        assert_eq!(parse_command("add"), None);
        assert_eq!(parse_command("add x"), None);
        assert_eq!(parse_command("reset now"), None);
        assert_eq!(parse_command("bogus"), None);
    }

    #[test]
    fn apply_changes_total() {
        assert_eq!(apply(10, &Command::Add(5)), 15);
        assert_eq!(apply(10, &Command::Reset), 0);
    }
}
```

Вывод `cargo run`:

```text
total = 12
cannot understand "bogus"
cannot understand "add x"
total = 0
```

Почему это работает: сопоставление кортежа из `Option` позволяет одному `match` описать всю форму команды, включая «без лишних слов». Текст разбирается в перечисление `Command` один раз; после этого `apply` — полный, проверяемый компилятором `match` по перечислению. Такая структура «разобрать, затем применить» лежит в основе любого консольного инструмента и интерпретатора, и вы воспользуетесь ею в проекте «телефонная книга» ([Урок 04.11](11-project-phone-book.md)).

</details>

## Вопросы для самопроверки

1. Когда вы предпочтёте `if let` вместо `match`?

<details>
<summary>Ответ</summary>

Когда важен только один образец, а остальное нужно игнорировать (или обработать единственным `else`). Если нужно обработать несколько случаев или хочется, чтобы компилятор проверил полноту, используйте `match`.

</details>

2. Чем `if let Some(x) = value` отличается от `if Some(x) == value`?

<details>
<summary>Ответ</summary>

`if let` — сопоставление с образцом: он привязывает `x` к внутреннему значению, когда образец подходит. `==` сравнивает два уже существующих значения; здесь `x` должен был бы уже существовать, и ничего не привязывается.

</details>

3. Что обязан делать блок `else` у `let else`?

<details>
<summary>Ответ</summary>

Он обязан расходиться — покидать обычный поток выполнения через `return`, `break`, `continue` или `panic!` (всё, что имеет тип `!`). Иначе переменные из образца были бы неопределены после оператора (`E0308`).

</details>

4. Что делает `while let Some(top) = stack.pop()`?

<details>
<summary>Ответ</summary>

Убирает и обрабатывает элементы с конца списка по одному и останавливается, когда `pop()` возвращает `None` (список пуст).

</details>

5. Чем опровергаемый образец отличается от неопровергаемого? Где разрешён каждый?

<details>
<summary>Ответ</summary>

Неопровергаемый образец подходит всегда (`x`, `(a, b)`); опровергаемый может не подойти (`Some(x)`). Обычный `let`, параметры функций и `for` требуют неопровергаемых образцов. `match`, `if let`, `while let` и `let else` принимают опровергаемые.

</details>

6. Чему соответствует `[first, .., last]`?

<details>
<summary>Ответ</summary>

Срезу или массиву не короче двух элементов; он привязывает первый и последний элементы и игнорирует всё между ними.

</details>

7. Чем `let _ = value;` отличается от `let _value = value;`?

<details>
<summary>Ответ</summary>

`_` ничего не привязывает, поэтому `value` не перемещается и им можно пользоваться позже. `_value` — настоящая переменная: значение перемещается в неё.

</details>

8. Что предлагает `cargo clippy`, когда `let else` только возвращает `None`?

<details>
<summary>Ответ</summary>

Использовать вместо него оператор `?` (линт `question_mark`). Оператор `?` вы изучите в Уроке 04.10.

</details>

## Итоги

- `if let ОБРАЗЕЦ = ЗНАЧЕНИЕ { ... } else { ... }` обрабатывает один случай `match`; полнота компилятором не проверяется.
- `while let` повторяется, пока образец подходит: естественный инструмент для `pop()` и `next()`.
- `let else` привязывает переменные или выходит (`return`, `continue`, `break`, `panic!`); он сохраняет основной путь плоским.
- `matches!(value, pattern)` возвращает `true` или `false`; цепочка let (`if let ... && cond`) сочетает образцы и условия (редакция 2024).
- Образцы работают в `let`, параметрах функций, циклах `for` и замыканиях; `_` игнорирует одно значение, `..` — остальное, а `_name` всё равно перемещает.
- Неопровергаемые образцы обязательны в обычном `let`, параметрах и `for`; опровергаемые идут в `match`, `if let`, `while let` и `let else`.
- Образцы срезов (`[first, .., last]`, `[first, rest @ ..]`) разбирают массивы и срезы; [Урок 06.3](../06-rust-advanced/03-patterns-in-depth.md) углубляется в образцы.
- Тестируйте оба выхода каждого `let else`.

## Источники и что почитать

- [The Rust Programming Language — 6.3 Concise Control Flow with if let and let else](https://doc.rust-lang.org/book/ch06-03-if-let.html) — глава официальной книги по этой теме.
- [The Rust Programming Language — 19.1 All the Places Patterns Can Be Used](https://doc.rust-lang.org/book/ch19-01-all-the-places-for-patterns.html) — полный список мест, где встречаются образцы.
- [The Rust Programming Language — 19.2 Refutability](https://doc.rust-lang.org/book/ch19-02-refutability.html) — опровергаемые и неопровергаемые образцы.
- [The Rust Programming Language — 19.3 Pattern Syntax](https://doc.rust-lang.org/book/ch19-03-pattern-syntax.html) — все формы образцов в одном месте.
- [Rust by Example — let-else](https://doc.rust-lang.org/rust-by-example/flow_control/let_else.html) — запускаемые примеры `let else`.
- [std::matches!](https://doc.rust-lang.org/std/macro.matches.html) — документация макроса `matches!`.

---

[🏠 Начало](../START-HERE.md) · [← 04.4 Option и match](04-option-and-match.md) · [📋 Этап 04](README.md) · [04.6 Модули, крейты и пакеты →](06-modules-crates-and-packages.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
