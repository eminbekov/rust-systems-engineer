<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 06.2 Продвинутые трейты и типы](02-advanced-traits-and-types.md) · [📋 Этап 06](README.md) · [06.4 Внутренняя изменяемость →](04-interior-mutability.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/06-rust-advanced/03-patterns-in-depth.md) <!-- nav -->

# Урок 06.3 — Образцы подробно

> **Этап 06 — Rust: продвинутый уровень** · Урок 3 из 12 · ⏱ около 3 часов (2 занятия)

## Что вы узнаете

- Все места, где Rust принимает образец (pattern), а не только `match`.
- Чем отличаются опровержимые и неопровержимые образцы и почему это важно компилятору.
- Полный синтаксис образцов: литералы, диапазоны, `|`, деструктуризация, `..`, охранные условия (guards), привязки `@`.
- Образцы срезов вроде `[first, .., last]`, удобные для разбора команд.
- Как образцы работают со ссылками и что изменилось в редакции 2024.
- `let else`, цепочки `let` (let chains) и макрос `matches!`.

## Перед началом

- Вы прошли [Урок 06.2 — Продвинутые трейты и типы](02-advanced-traits-and-types.md).
- Вы знаете `match`, `Option` и перечисления по [Уроку 04.4 — Option и match](../04-structs-enums-collections-errors/04-option-and-match.md), а также `if let` и `let else` по [Уроку 04.5](../04-structs-enums-collections-errors/05-if-let-let-else-and-patterns.md). Этот урок идёт глубже, поэтому полезно быстро перечитать Урок 04.5.
- Создайте учебный проект: `cargo new code/p06-patterns`.

## Новые слова

| Слово | Значение |
|---|---|
| **образец** (pattern) | Форма, с которой сравнивается значение. Он может ещё и давать имена частям значения: `Some(x)`, `(a, b)`, `1..=5`. |
| **опровержимый образец** (refutable pattern) | Образец, который может не совпасть с некоторыми значениями. `Some(x)` не совпадает с `None`. |
| **неопровержимый образец** (irrefutable pattern) | Образец, который совпадает всегда. `x` и `(a, b)` совпадают всегда. |
| **деструктуризация** (destructuring) | Разбор значения (кортежа, структуры, перечисления, массива) на части и присвоение этим частям имён. |
| **шаблонный символ** (wildcard) | Образец `_`. Совпадает с чем угодно и не вводит имя. |
| **охранное условие** (match guard) | Дополнительное условие `if` после образца в ветке `match`. |
| **привязка `@`** (`@` binding) | `name @ pattern`: проверить значение образцом и одновременно сохранить его под именем. |
| **образец среза** (slice pattern) | Образец для массивов и срезов, например `[first, .., last]`. |
| **исчерпывающий** (exhaustive) | `match` исчерпывающий, если его ветки покрывают все возможные значения. Rust это требует. |
| **режим привязки** (binding mode) | То, как переменная образца получает значение: перемещением/копированием или как ссылка. |
| **цепочка let** (let chain) | Условие `if` или `while`, которое соединяет образцы `let` и логические проверки через `&&`. |

## 1. Где разрешены образцы

Вы уже знаете `match` и `if let`. Образцы встречаются и во многих других местах:

```rust
fn print_point(&(x, y): &(i32, i32)) {
    println!("point at {x}, {y}");
}

fn main() {
    // 1. `let` принимает образец
    let (name, age) = ("Ana", 31);
    println!("{name} is {age}");

    // 2. параметры функций принимают образцы
    print_point(&(3, 4));

    // 3. `for` принимает образец
    for (index, letter) in "abc".chars().enumerate() {
        println!("{index}: {letter}");
    }

    // 4. `while let` повторяется, пока образец совпадает
    let mut stack = vec![1, 2, 3];
    while let Some(top) = stack.pop() {
        println!("popped {top}");
    }

    // 5. `if let` и `let else`
    let maybe_number = Some(7);
    if let Some(n) = maybe_number {
        println!("got {n}");
    }
    let Some(n) = maybe_number else {
        return;
    };
    println!("still {n}");
}
```

```text
Ana is 31
point at 3, 4
0: a
1: b
2: c
popped 3
popped 2
popped 1
got 7
still 7
```

**Что происходит:**

1. `let (name, age) = ("Ana", 31);` — левая часть любого `let` является образцом. Образец-кортеж `(name, age)` разбирает кортеж. (Простое `let x = 5;` — тоже образец: просто имя.) Запись `{name}` внутри `println!` печатает переменную с этим именем.
2. `fn print_point(&(x, y): &(i32, i32))` — параметр функции является образцом. Этот убирает `&` и разбирает пару прямо в сигнатуре.
3. `for (index, letter) in ...enumerate()` — переменная цикла тоже образец.
4. `while let Some(top) = stack.pop()` повторяется, пока образец совпадает. Когда вектор пуст, `pop` возвращает `None` и цикл заканчивается.
5. `if let` выполняет блок, только если образец совпал. `let ... else` обрабатывает противоположный случай: если образец не совпал, блок `else` обязан завершиться переходом (`return`, `break`, `continue` или `panic!`).

## 2. Опровержимые и неопровержимые образцы

Образец **неопровержим**, если он совпадает с любым возможным значением типа: `x`, `(a, b)`, `_`. Образец **опровержим**, если некоторые значения не совпадают: `Some(x)` не совпадает с `None`; `1..=5` не совпадает с 6.

Правило: `let`, `for` и параметры функций принимают только неопровержимые образцы, потому что для случая «не совпало» нет ветки кода. `if let`, `while let`, `let else` и ветки `match` принимают опровержимые образцы.

```rust
fn main() {
    let input: Option<i32> = Some(5);
    let Some(value) = input;
    println!("{value}");
}
```

```text
error[E0005]: refutable pattern in local binding
 --> src/main.rs:3:9
  |
3 |     let Some(value) = input;
  |         ^^^^^^^^^^^ pattern `None` not covered
  |
  = note: `let` bindings require an "irrefutable pattern", like a `struct` or an `enum` with only one variant
  = note: for more information, visit https://doc.rust-lang.org/book/ch19-02-refutability.html
  = note: the matched value is of type `Option<i32>`
help: you might want to use `let...else` to handle the variant that isn't matched
  |
3 |     let Some(value) = input else { todo!() };
  |                             ++++++++++++++++
```

**Что происходит:**

1. `input` — это `Option<i32>`, поэтому он может быть `None`.
2. У `let Some(value) = input;` нет ответа для `None`. Компилятор отказывается с ошибкой `E0005` и называет непокрытый случай.
3. Добавьте блок `else` (`let ... else { return };` или `else { panic!(...) }`) либо используйте `match` / `if let`.

Обратная ошибка — только предупреждение: неопровержимый образец в `if let` бессмыслен, потому что он совпадает всегда.

```text
warning: irrefutable `if let` pattern
 --> src/main.rs:3:8
  |
3 |     if let x = value {
  |        ^^^^^^^^^^^^^
  |
  = note: this pattern will always match, so the `if let` is useless
  = help: consider replacing the `if let` with a `let`
  = note: `#[warn(irrefutable_let_patterns)]` on by default
```

## 3. Литералы, диапазоны и альтернативы

В ветках `match` можно сочетать несколько простых образцов:

```rust
fn describe(n: i32) -> &'static str {
    match n {
        0 => "zero",
        2 | 4 | 6 | 8 => "even digit",
        1..=9 => "odd digit",
        10.. => "big",
        i32::MIN..=-1 => "negative",
    }
}

fn kind(c: char) -> &'static str {
    match c {
        'a' | 'e' | 'i' | 'o' | 'u' => "vowel",
        'a'..='z' => "consonant",
        '0'..='9' => "digit",
        _ => "other",
    }
}

fn main() {
    for n in [-5, 0, 2, 7, 10, 500] {
        println!("{n:>4} -> {}", describe(n));
    }
    for c in ['e', 'x', '7', '#'] {
        println!("{c} -> {}", kind(c));
    }
}
```

```text
  -5 -> negative
   0 -> zero
   2 -> even digit
   7 -> odd digit
  10 -> big
 500 -> big
e -> vowel
x -> consonant
7 -> digit
# -> other
```

**Что происходит:**

1. `0` совпадает с литералом 0.
2. `2 | 4 | 6 | 8` — **образец-альтернатива** (or-pattern): совпадает с 2, 4, 6 или 8.
3. `1..=9` — диапазон с включённым концом: от 1 до 9 включительно. Ветки пробуются сверху вниз, поэтому чётные цифры уже забрала ветка выше, и до этой ветки доходят только нечётные. Диапазоны работают для целых чисел и `char`. Запись `10..` означает «10 и больше», а `i32::MIN..=-1` — «все отрицательные числа».
4. Вместе ветки покрывают все `i32`, поэтому `match` **исчерпывающий** и `_` не нужен. Для `char` `_` нужен, потому что перечислить все символы нельзя.
5. Ветки проверяются сверху вниз, поэтому `'a' | 'e' | ...` должна стоять раньше диапазона `'a'..='z'`.

Если вы забудете случай, компилятор скажет, каких значений не хватает:

```text
error[E0004]: non-exhaustive patterns: `100_u8..=u8::MAX` not covered
 --> src/main.rs:3:22
  |
3 |     let text = match n {
  |                      ^ pattern `100_u8..=u8::MAX` not covered
  |
  = note: the matched value is of type `u8`
help: ensure that all possible cases are being handled by adding a match arm with a wildcard pattern or an explicit pattern as shown
  |
5 ~         10..=99 => "two digits",
6 ~         100_u8..=u8::MAX => todo!(),
  |
```

> 💡 **Совет:** простое имя в образце — всегда *новая переменная*, которая совпадает со всем. Это никогда не сравнение с существующей переменной с таким же именем. Чтобы сравнить со значением, используйте константу, литерал или охранное условие (раздел 5).

## 4. Деструктуризация структур, перечислений и кортежей

Деструктуризация разбирает значение прямо в образце. Можно спускаться сколь угодно глубоко:

```rust
#[derive(Debug)]
struct Point {
    x: i32,
    y: i32,
}

#[derive(Debug)]
enum Shape {
    Circle {
        center: Point,
        radius: u32,
    },
    Rect {
        top_left: Point,
        bottom_right: Point,
    },
    Dot(Point),
}

fn describe(shape: &Shape) -> String {
    match shape {
        Shape::Circle {
            center: Point { x: 0, y: 0 },
            radius,
        } => {
            format!("circle at the origin, radius {radius}")
        }
        Shape::Circle { radius, .. } => format!("circle, radius {radius}"),
        Shape::Rect {
            top_left: Point { x: x1, y: y1 },
            bottom_right: Point { x: x2, y: y2 },
        } => format!("rectangle {}x{}", x2 - x1, y1 - y2),
        Shape::Dot(Point { x, .. }) => format!("dot with x = {x}"),
    }
}

fn main() {
    let shapes = [
        Shape::Circle {
            center: Point { x: 0, y: 0 },
            radius: 5,
        },
        Shape::Circle {
            center: Point { x: 2, y: 3 },
            radius: 1,
        },
        Shape::Rect {
            top_left: Point { x: 1, y: 6 },
            bottom_right: Point { x: 5, y: 2 },
        },
        Shape::Dot(Point { x: 9, y: 9 }),
    ];
    for shape in &shapes {
        println!("{}", describe(shape));
    }

    // Игнорирование частей значения
    let (first, _, third) = (1, 2, 3);
    let _unused = first + third; // ведущее подчёркивание убирает предупреждение «не используется»
    let numbers = (10, 20, 30, 40, 50);
    let (head, .., last) = numbers;
    println!("{head} {last}");
}
```

```text
circle at the origin, radius 5
circle, radius 1
rectangle 4x4
dot with x = 9
10 50
```

**Что происходит:**

1. `Shape::Circle { center: Point { x: 0, y: 0 }, radius }` проверяет, что фигура — круг с центром точно в `(0, 0)`, и привязывает `radius`. Литерал внутри образца — это проверка.
2. `Shape::Circle { radius, .. }` — `..` означает «остальные поля меня не интересуют». Короткая запись `radius` означает `radius: radius`.
3. `Point { x: x1, y: y1 }` переименовывает поля при привязке. Четыре числа привязываются одним образцом и используются в `x2 - x1` (ширина) и `y1 - y2` (высота).
4. `Shape::Dot(Point { x, .. })` достаёт только `x` из кортежного варианта.
5. `match shape`, где `shape` — это `&Shape`: привязки автоматически становятся ссылками (раздел 7).
6. `_` игнорирует одно значение, `..` игнорирует несколько подряд: `(head, .., last)` берёт первый и последний элементы кортежа из пяти. Имя, начинающееся с подчёркивания (`_unused`), — настоящая переменная, но Rust не предупреждает, если вы её не используете.

## 5. Охранные условия и привязки `@`

**Охранное условие** (match guard) — это `if` после образца. Ветка срабатывает, только если образец совпал и условие истинно. Оператор `@` сохраняет совпавшее значение под именем, одновременно проверяя его образцом:

```rust
fn classify(pair: (i32, i32)) -> String {
    match pair {
        (x, y) if x == y => format!("{x} equals {y}"),
        (x, y) if x + y == 0 => String::from("opposites"),
        (x, _) if x % 2 == 0 => format!("{x} is even"),
        _ => String::from("nothing special"),
    }
}

fn age_group(age: u32) -> String {
    match age {
        n @ 0..=12 => format!("child, {n} years"),
        n @ 13..=19 => format!("teenager, {n} years"),
        n => format!("adult, {n} years"),
    }
}

fn main() {
    for pair in [(3, 3), (4, -4), (6, 1), (5, 2)] {
        println!("{pair:?}: {}", classify(pair));
    }
    for age in [7, 15, 40] {
        println!("{}", age_group(age));
    }

    // Охранное условие относится ко ВСЕМУ образцу с `|`
    let n = 4;
    match n {
        2 | 4 | 6 if n > 4 => println!("even digit above 4"),
        2 | 4 | 6 => println!("even digit up to 4"),
        _ => println!("something else"),
    }
}
```

```text
(3, 3): 3 equals 3
(4, -4): opposites
(6, 1): 6 is even
(5, 2): nothing special
child, 7 years
teenager, 15 years
adult, 40 years
even digit up to 4
```

**Что происходит:**

1. `(x, y) if x == y` — сам образец совпадает с любой парой. Охранное условие его сужает. Так сравнивают две части одного значения.
2. Ветки пробуются сверху. Пара `(4, -4)` не проходит первое условие (4 не равно -4), но проходит второе (4 + -4 == 0).
3. `n @ 0..=12` проверяет, что значение лежит в диапазоне от 0 до 12, *и* привязывает число к `n`. Без `@` пришлось бы выбирать между диапазоном и именем.
4. `2 | 4 | 6 if n > 4` — охранное условие относится ко **всему** образцу-альтернативе, а не только к `6`. Для `n = 4` образец совпадает, но условие ложно, поэтому выполняется следующая ветка.
5. Компилятор не учитывает охранные условия при проверке исчерпываемости. `match`, который заканчивается веткой с условием, всё равно требует последнюю ветку без условия.

> 📏 **Соглашение:** делайте охранные условия короткими. Если условию нужно несколько строк, вынесите логику в функцию (`if is_valid(&x)`) или в `if` внутри ветки.

## 6. Образцы срезов

**Образец среза** совпадает с массивами и срезами по их длине и элементам. Он идеален для разборщиков команд:

```rust
fn run(command: &[&str]) -> String {
    match command {
        [] => String::from("empty command"),
        ["quit"] => String::from("bye"),
        ["add", name, phone] => format!("adding {name} with {phone}"),
        ["remove", name] => format!("removing {name}"),
        ["echo", words @ ..] => words.join(" "),
        [first, .., last] => format!("unknown command from {first} to {last}"),
        [single] => format!("unknown command {single}"),
    }
}

fn main() {
    let lines = [
        "",
        "quit",
        "add ana 555-0101",
        "remove bo",
        "echo hello brave new world",
        "move left up high",
        "jump",
    ];
    for line in lines {
        let parts: Vec<&str> = line.split_whitespace().collect();
        println!("{:<28} => {}", format!("{line:?}"), run(&parts));
    }

    let numbers = [1, 2, 3, 4, 5];
    let [a, b, rest @ ..] = numbers;
    println!("{a} {b} {rest:?}");
}
```

```text
""                           => empty command
"quit"                       => bye
"add ana 555-0101"           => adding ana with 555-0101
"remove bo"                  => removing bo
"echo hello brave new world" => hello brave new world
"move left up high"          => unknown command from move to high
"jump"                       => unknown command jump
1 2 [3, 4, 5]
```

**Что происходит:**

1. `command` — это `&[&str]`: срез слов.
2. `[]` совпадает с пустым срезом. `["quit"]` совпадает ровно с одним элементом, равным `"quit"`.
3. `["add", name, phone]` совпадает ровно с тремя элементами, первый из которых равен `"add"`, и привязывает остальные два.
4. `["echo", words @ ..]` совпадает с `"echo"`, за которым следует любое число элементов. `words @ ..` привязывает остаток как подсрез. Его тип — `&[&str]`, поэтому работает `join(" ")`.
5. `[first, .., last]` требует не менее двух элементов и игнорирует всё между ними.
6. `[single]` обрабатывает единственный оставшийся случай: ровно одно слово, не равное `quit`. Вместе ветки покрывают все возможные длины, поэтому `match` исчерпывающий, и компилятор принимает его без `_`.
7. Последние строки показывают тот же синтаксис с `let` для массива: `let [a, b, rest @ ..] = numbers;`. Для массива с известной длиной такой образец неопровержим.

> 🧪 **Тестирование:** образцы срезов упрощают тестирование функций-разборщиков: передайте срез вроде `&["add", "ana", "555"]` и сравните результат. Вы сделаете это в Упражнении 2.

## 7. Образцы и ссылки

Когда вы сопоставляете ссылку, Rust вам помогает. Если значение — `&Option<String>`, а образец — `Some(nick)`, переменная `nick` становится `&String`: *ссылкой* на содержимое. Это называется **режимом привязки по умолчанию** (default binding mode). Ничего не перемещается:

```rust
#[derive(Debug)]
struct User {
    name: String,
    nickname: Option<String>,
}

fn main() {
    let user = User {
        name: String::from("Alexandra"),
        nickname: Some(String::from("Sasha")),
    };

    // Сопоставление ССЫЛКИ: `nick` становится ссылкой, ничего не перемещается
    if let Some(nick) = &user.nickname {
        println!("{} is called {nick}", user.name);
    }
    println!("user is still usable: {user:?}");

    // `&` в образце снимает один уровень ссылки
    let numbers = [1, 2, 3];
    let total: i32 = numbers.iter().map(|&n| n * 10).sum();
    println!("{total}");

    // `ref mut` берёт изменяемое заимствование при деструктуризации
    let mut pair = (String::from("a"), String::from("b"));
    let (ref mut first, _) = pair;
    first.push('!');
    println!("{pair:?}");

    // `matches!` возвращает bool для образца
    let c = 'k';
    println!("{}", matches!(c, 'a'..='m'));
}
```

```text
Alexandra is called Sasha
user is still usable: User { name: "Alexandra", nickname: Some("Sasha") }
60
("a!", "b")
true
```

**Что происходит:**

1. `if let Some(nick) = &user.nickname` сопоставляет ссылку, поэтому `nick` — это `&String`. После этого `user` по-прежнему можно использовать.
2. `.map(|&n| n * 10)` — итератор отдаёт `&i32`. Образец `&n` снимает ссылку, и `n` становится `i32`.
3. `let (ref mut first, _) = pair;` использует `ref mut`, чтобы взять *изменяемое заимствование* поля вместо перемещения. `first.push('!')` меняет строку внутри `pair`.
4. `matches!(c, 'a'..='m')` разворачивается в `match`, который возвращает `true`, если образец совпал, и `false` в остальных случаях. Это самый короткий способ проверить образец в условии.

А что, если забыть `&`? Значение будет перемещено из `user`:

```text
error[E0382]: borrow of partially moved value: `user`
  --> src/main.rs:13:22
   |
10 |     if let Some(name) = user.name {
   |                 ---- value partially moved here
...
13 |     println!("{:?}", user);
   |                      ^^^^ value borrowed here after partial move
   |
   = note: partial move occurs because value has type `String`, which does not implement the `Copy` trait
help: borrow this binding in the pattern to avoid moving the value
   |
10 |     if let Some(ref name) = user.name {
   |                 +++
```

Компилятор предлагает `ref name`. В современном коде лучше сопоставлять `&user.name`.

**Изменение в редакции 2024.** В редакции 2024, если режим привязки по умолчанию уже делает `n` ссылкой, нельзя явно писать `ref`, `ref mut`, `mut` или `&` в том же образце:

```text
error: cannot explicitly borrow within an implicitly-borrowing pattern
 --> src/main.rs:4:14
  |
4 |         Some(ref n) => println!("{n}"),
  |              ^^^ explicit `ref` binding modifier not allowed when implicitly borrowing
  |
  = note: for more information, see <https://doc.rust-lang.org/reference/patterns.html#binding-modes>
note: matching on a reference type with a non-reference pattern implicitly borrows the contents
 --> src/main.rs:4:9
  |
4 |         Some(ref n) => println!("{n}"),
  |         ^^^^^^^^^^^ this non-reference pattern matches on a reference type `&_`
help: remove the unnecessary binding modifier
  |
4 -         Some(ref n) => println!("{n}"),
4 +         Some(n) => println!("{n}"),
  |
```

Исправление — то, что показывает компилятор: уберите `ref`. Явный `ref` по-прежнему допустим, когда значение сопоставляется по значению (как в `let (ref mut first, _) = pair;`), но он вам редко нужен.

## 8. Цепочки let

**Цепочка let** соединяет образцы `let` и обычные условия через `&&` в одном `if` (или `while`). Эта возможность доступна в редакции 2024:

```rust
fn main() {
    let width: Option<u32> = Some(8);
    let height: Option<u32> = Some(5);

    // Цепочка let соединяет образцы и логические условия через `&&`
    if let Some(w) = width
        && let Some(h) = height
        && w > h
    {
        println!("wide: {w} x {h}");
    }

    let words = ["apple", "bob"];
    if let Some(first) = words.first()
        && first.len() > 3
    {
        println!("first word is long: {first}");
    }
}
```

```text
wide: 8 x 5
first word is long: apple
```

**Что происходит:**

1. Условия проверяются слева направо. Каждый успешный `let` делает свою переменную доступной для следующих условий и внутри блока.
2. Первому `if` нужно, чтобы оба значения были `Some` *и* `w > h`. Если любая часть не выполнилась, блок пропускается.
3. Без цепочек let вы писали бы вложенные блоки `if let` или `match` по кортежу.
4. `words.first()` возвращает `Option<&&str>`. `first.len()` работает, потому что вызов метода автоматически проходит сквозь ссылки.

## Частые ошибки

### 1. Опровержимый образец в `let`

`let Some(x) = opt;` даёт `E0005`. Используйте `let Some(x) = opt else { ... };`, `if let` или `match`.

### 2. Случайное перемещение значения

`if let Some(name) = user.name` перемещает `String` из `user` (`E0382 borrow of partially moved value`). Сопоставляйте ссылку: `if let Some(name) = &user.name`.

### 3. Простое имя ничего не сравнивает

```rust
let limit = 10;
match n {
    limit => println!("n equals limit"), // неверно: `limit` — НОВАЯ переменная, она совпадает с любым n
    _ => println!("n is different"),
}
```

Первая ветка совпадает со всем. Компилятор предупреждает, что ветка `_` недостижима. Используйте охранное условие: `x if x == limit`.

### 4. Запись `ref` там, где действуют правила редакции 2024

`cannot explicitly borrow within an implicitly-borrowing pattern`: удалите `ref`.

### 5. Забыли про порядок веток

Широкая ветка (`n`, `_`, широкий диапазон) перед узкой скрывает её. Ставьте конкретные образцы первыми.

## Попробуйте сами

1. Откройте `~/projects/rust/code/p06-patterns`.
2. Скопируйте пример `slices.rs` (раздел 6) в `src/main.rs` и запустите `cargo run`.
3. Добавьте новую команду `["mv", from, to]`, которая возвращает `moving {from} to {to}`. Запустите снова со строкой `mv a.txt b.txt`.
4. Добавьте команду `["help", ..]`, которая печатает `help is coming soon` при любом количестве следующих слов. Поставьте её раньше «ловящих всё» веток.
5. Выполните `cargo fmt` и `cargo clippy`.
6. Сделайте коммит: `git add code/p06-patterns` и `git commit -m "feat(p06): slice-pattern command parser"`.

## Упражнения

### Упражнение 1 — Буквенные оценки (легко)

Напишите `fn grade(score: u32) -> char`: от 90 до 100 — `'A'`, от 80 до 89 — `'B'`, от 70 до 79 — `'C'`, от 0 до 69 — `'F'`, а всё выше 100 — `'?'`. Напишите тест для границ (0, 69, 70, 89, 90, 100) и тест для 101.

<details>
<summary>Подсказка</summary>

Используйте образцы-диапазоны с `..=` в `match`. Последняя ветка — `_`. Тест — это функция с `#[test]` внутри `mod tests`.

</details>

<details>
<summary>Ответ</summary>

```rust
fn grade(score: u32) -> char {
    match score {
        90..=100 => 'A',
        80..=89 => 'B',
        70..=79 => 'C',
        0..=69 => 'F',
        _ => '?', // оценки выше 100 недопустимы
    }
}

fn main() {
    for score in [95, 85, 72, 10, 101] {
        println!("{score} -> {}", grade(score));
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn boundaries() {
        assert_eq!(grade(100), 'A');
        assert_eq!(grade(90), 'A');
        assert_eq!(grade(89), 'B');
        assert_eq!(grade(70), 'C');
        assert_eq!(grade(69), 'F');
        assert_eq!(grade(0), 'F');
    }

    #[test]
    fn invalid_score() {
        assert_eq!(grade(101), '?');
    }
}
```

```text
95 -> A
85 -> B
72 -> C
10 -> F
101 -> ?
```

```text

running 2 tests
..
test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

Почему это работает: диапазоны проверяются по порядку; поскольку они не пересекаются, порядок здесь не важен. Ветка `_` ловит всё, что выше 100.

</details>

### Упражнение 2 — Разборщик команд с тестом (средне)

Определите `enum Command<'a>` с вариантами `Add { name, phone }`, `Remove(name)`, `List` и `Say(words)`. Напишите `fn parse<'a>(parts: &[&'a str]) -> Option<Command<'a>>` с образцами срезов. Для неизвестных или неправильных команд возвращайте `None`. Напишите тесты для корректной `add`, для `say` с несколькими словами и для неверного ввода.

<details>
<summary>Подсказка</summary>

Одна ветка на команду: `["add", name, phone]`, `["remove", name]`, `["list"]`, `["say", words @ ..]`. Последняя ветка — `_ => None`. Для `Say` превратите подсрез в вектор через `words.to_vec()`. Время жизни `'a` относится к кускам текста; зачем оно нужно, вы узнаете в [Уроке 06.5](05-lifetimes-in-depth.md).

</details>

<details>
<summary>Ответ</summary>

```rust
#[derive(Debug, PartialEq)]
enum Command<'a> {
    Add { name: &'a str, phone: &'a str },
    Remove(&'a str),
    List,
    Say(Vec<&'a str>),
}

fn parse<'a>(parts: &[&'a str]) -> Option<Command<'a>> {
    match parts {
        ["add", name, phone] => Some(Command::Add { name, phone }),
        ["remove", name] => Some(Command::Remove(name)),
        ["list"] => Some(Command::List),
        ["say", words @ ..] => Some(Command::Say(words.to_vec())),
        _ => None,
    }
}

fn main() {
    let line = "add ana 555-0101";
    let parts: Vec<&str> = line.split_whitespace().collect();
    println!("{:?}", parse(&parts));
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn parses_add() {
        assert_eq!(
            parse(&["add", "ana", "555"]),
            Some(Command::Add {
                name: "ana",
                phone: "555"
            })
        );
    }

    #[test]
    fn parses_say_with_many_words() {
        assert_eq!(
            parse(&["say", "hi", "there"]),
            Some(Command::Say(vec!["hi", "there"]))
        );
    }

    #[test]
    fn rejects_wrong_arity_and_unknown() {
        assert_eq!(parse(&["add", "ana"]), None);
        assert_eq!(parse(&["fly"]), None);
        assert_eq!(parse(&[]), None);
    }
}
```

```text
Some(Add { name: "ana", phone: "555-0101" })
```

```text

running 3 tests
...
test result: ok. 3 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

Почему это работает: каждая ветка проверяет и длину, и первое слово, поэтому `add ana` из двух слов проваливается в `_ => None`. Короткая запись `Command::Add { name, phone }` использует переменные `name` и `phone`, привязанные образцом.

</details>

### Упражнение 3 — Четверти с охранными условиями (средне)

Напишите `fn quadrant(point: (i32, i32)) -> &'static str`, которая возвращает `"origin"`, `"on the x axis"`, `"on the y axis"` или от `"first quadrant"` до `"fourth quadrant"` (первая: x и y положительны; вторая: x отрицателен, y положителен; и так далее). Напечатайте ответ для семи точек.

<details>
<summary>Подсказка</summary>

Особые случаи поставьте первыми: `(0, 0)`, `(_, 0)`, `(0, _)`. Для четвертей используйте охранные условия: `(x, y) if x > 0 && y > 0`. Последняя ветка может быть `_` для четвёртой четверти.

</details>

<details>
<summary>Ответ</summary>

```rust
fn quadrant(point: (i32, i32)) -> &'static str {
    match point {
        (0, 0) => "origin",
        (_, 0) => "on the x axis",
        (0, _) => "on the y axis",
        (x, y) if x > 0 && y > 0 => "first quadrant",
        (x, y) if x < 0 && y > 0 => "second quadrant",
        (x, y) if x < 0 && y < 0 => "third quadrant",
        _ => "fourth quadrant",
    }
}

fn main() {
    for p in [(0, 0), (5, 0), (0, -2), (1, 1), (-1, 1), (-1, -1), (1, -1)] {
        println!("{p:?} -> {}", quadrant(p));
    }
}
```

```text
(0, 0) -> origin
(5, 0) -> on the x axis
(0, -2) -> on the y axis
(1, 1) -> first quadrant
(-1, 1) -> second quadrant
(-1, -1) -> third quadrant
(1, -1) -> fourth quadrant
```

Почему это работает: ветки проверяются по порядку. Когда мы добираемся до охранных условий, ни одна координата не равна нулю, остаются четыре сочетания знаков, и последняя ветка обрабатывает оставшееся.

</details>

### Упражнение 4 — Разбор `key = value` (сложно)

Напишите `fn parse_setting(line: &str) -> Option<(String, u32)>`, которая делит строку по `=`, убирает пробелы, разбирает правую часть как `u32` и возвращает `Some((key, number))`. Возвращайте `None`, если нет `=`, если число неверное или если ключ пустой. Используйте `let else`. Протестируйте одну корректную и три некорректные строки.

<details>
<summary>Подсказка</summary>

`line.split_once('=')` возвращает `Option<(&str, &str)>`. Для `Option` оператор `?` досрочно возвращает `None`. Для числа используйте `let Ok(number) = value.trim().parse::<u32>() else { return None; };`.

</details>

<details>
<summary>Ответ</summary>

```rust
fn parse_setting(line: &str) -> Option<(String, u32)> {
    let (key, value) = line.split_once('=')?;
    let Ok(number) = value.trim().parse::<u32>() else {
        return None;
    };
    if key.trim().is_empty() {
        return None;
    }
    Some((key.trim().to_string(), number))
}

fn main() {
    for line in ["port = 8080", "port=abc", "no equals sign", " = 5"] {
        println!("{line:?} -> {:?}", parse_setting(line));
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn valid_line() {
        assert_eq!(
            parse_setting("port = 8080"),
            Some((String::from("port"), 8080))
        );
    }

    #[test]
    fn invalid_lines() {
        assert_eq!(parse_setting("port=abc"), None);
        assert_eq!(parse_setting("no equals sign"), None);
        assert_eq!(parse_setting(" = 5"), None);
    }
}
```

```text
"port = 8080" -> Some(("port", 8080))
"port=abc" -> None
"no equals sign" -> None
" = 5" -> None
```

```text

running 2 tests
..
test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

Почему это работает: `?` для `Option` возвращает `None` из функции, если значение — `None`. `let else` делает то же самое для любого опровержимого образца, здесь `Ok(number)`. В первом случае Clippy предлагает `?`, поэтому там использован он.

</details>

## Вопросы для самопроверки

1. Назовите пять мест, где образец может появиться, кроме `match`.

<details>
<summary>Ответ</summary>

Операторы `let`, параметры функций (и замыканий), циклы `for`, `if let` / `while let` / `let else` и цепочки let.

</details>

2. Почему `let Some(x) = opt;` не компилируется?

<details>
<summary>Ответ</summary>

`Some(x)` опровержим (не совпадает с `None`), а `let` требует неопровержимый образец, потому что нет кода, который выполнялся бы при неудаче. Используйте `let ... else`, `if let` или `match`.

</details>

3. Что делает `n @ 13..=19`?

<details>
<summary>Ответ</summary>

Совпадает со значением из диапазона от 13 до 19 и одновременно привязывает это значение к имени `n`.

</details>

4. Как охранное условие взаимодействует с образцом-альтернативой `2 | 4 | 6 if cond`?

<details>
<summary>Ответ</summary>

Условие относится ко всему образцу-альтернативе. Если значение совпало с какой-то альтернативой, но условие ложно, ветка пропускается (другие альтернативы не пробуются).

</details>

5. С чем совпадает `["echo", words @ ..]`?

<details>
<summary>Ответ</summary>

Со срезом хотя бы из одного элемента, где первый элемент равен `"echo"`. Остальные элементы (возможно, их нет) привязываются как подсрез `words`.

</details>

6. Какой тип у `nick` в `if let Some(nick) = &user.nickname`, если `nickname` — `Option<String>`?

<details>
<summary>Ответ</summary>

`&String`. Сопоставление ссылки делает привязку ссылкой (режим привязки по умолчанию), поэтому ничего не перемещается из `user`.

</details>

7. Что такое цепочка let и какое условие выражает `if let Some(w) = a && w > 3`?

<details>
<summary>Ответ</summary>

Цепочка let соединяет образцы `let` и логические условия через `&&`. Пример истинен, только если `a` — `Some` и значение внутри больше 3; `w` можно использовать в блоке.

</details>

## Итоги

- Образцы встречаются в `let`, параметрах функций, `for`, `match`, `if let`, `while let`, `let else` и цепочках let.
- Неопровержимые образцы совпадают всегда (их требуют `let`, `for` и параметры); опровержимые могут не совпасть (`if let`, `while let`, `match`, `let else`).
- Образцы могут использовать литералы, диапазоны (`..=`), альтернативы (`|`), шаблонные символы (`_`, `..`) и разбирать кортежи, структуры, перечисления и вложенные значения.
- Охранные условия (`if`) добавляют проверки; `name @ pattern` сохраняет значение, проверяя его.
- Образцы срезов (`[first, .., last]`, `[cmd, rest @ ..]`) идеальны для разбора.
- Сопоставление ссылки даёт привязки-ссылки; в редакции 2024 нельзя писать `ref` там, где это уже происходит автоматически.
- `matches!` проверяет образец как `bool`. `match` должен быть исчерпывающим, а ветки пробуются сверху вниз.

## Источники и что почитать

- [The Rust Programming Language — 19 Patterns and Matching](https://doc.rust-lang.org/book/ch19-00-patterns.html) — обзор главы.
- [19.1 All the Places Patterns Can Be Used](https://doc.rust-lang.org/book/ch19-01-all-the-places-for-patterns.html) — все места, где разрешён образец.
- [19.2 Refutability](https://doc.rust-lang.org/book/ch19-02-refutability.html) — опровержимые и неопровержимые образцы.
- [19.3 Pattern Syntax](https://doc.rust-lang.org/book/ch19-03-pattern-syntax.html) — полный синтаксис с примерами.
- [The Rust Reference — Patterns](https://doc.rust-lang.org/reference/patterns.html) — точные правила, включая режимы привязки и изменения редакции 2024.
- [std::matches! macro](https://doc.rust-lang.org/std/macro.matches.html) — документация `matches!`.

---

[🏠 Начало](../START-HERE.md) · [← 06.2 Продвинутые трейты и типы](02-advanced-traits-and-types.md) · [📋 Этап 06](README.md) · [06.4 Внутренняя изменяемость →](04-interior-mutability.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
