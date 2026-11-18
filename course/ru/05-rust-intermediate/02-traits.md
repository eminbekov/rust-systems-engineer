<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 05.1 Обобщённые типы](01-generics.md) · [📋 Этап 05](README.md) · [05.3 Времена жизни →](03-lifetimes.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/05-rust-intermediate/02-traits.md) <!-- nav -->

# Урок 05.2 — Трейты

> **Этап 05 — Rust: средний уровень** · Урок 2 из 14 · ⏱ около 4 часов (3 занятия)

## Что вы узнаете

- Что такое трейт: именованный набор умений, который могут разделять разные типы.
- Как определить трейт, реализовать его для своих типов и дать методам тела по умолчанию.
- Как принять в функцию «всё, что обладает этим умением»: `impl Trait`, ограничения трейтов и `where`.
- Как пользоваться стандартными трейтами: `#[derive(...)]`, `Display`, `From`/`Into` и операторами вроде `+`.
- Что такое сквозные реализации (blanket), супертрейты и правило сирот (orphan rule), и как читать частые ошибки трейтов.

## Перед началом

- Вы прошли [Урок 05.1 — Обобщённые типы](01-generics.md). Ограничения трейтов появились там, а
  сегодня вы узнаете, что стоит справа от двоеточия.
- У вас есть учебный пакет `code/p05-lessons` из Урока 05.1 (внутри него папка `examples/`).
- Вы знаете модули ([Урок 04.6](../04-structs-enums-collections-errors/06-modules-crates-and-packages.md))
  и типы `Option`/`Result` ([Урок 04.4](../04-structs-enums-collections-errors/04-option-and-match.md)).

## Новые слова

| Слово | Значение |
|---|---|
| **трейт** (trait) | Именованный список сигнатур методов, которые тип обещает предоставить. Как должностная инструкция: говорит, *что* должно быть возможно, но не *как*. |
| **реализовать** (implement) | Написать `impl Trait for Type { … }`, дав типу методы, перечисленные в трейте. |
| **метод по умолчанию** (default method) | Метод, у которого есть тело прямо в трейте. Типы получают его бесплатно и могут заменить. |
| **`impl Trait`** | Способ записать параметр или тип результата как «какой-то тип, реализующий этот трейт», не называя его. |
| **derive** | `#[derive(Trait)]`: попросить компилятор написать стандартную реализацию трейта за вас. |
| **перегрузка операторов** | Заставить `+`, `-`, `*`, `==` и похожие операторы работать для ваших типов, реализовав трейт. |
| **сквозная реализация** (blanket implementation) | Реализация для *каждого* типа, удовлетворяющего ограничению: `impl<T: Display> Shout for T`. |
| **супертрейт** (supertrait) | Трейт, который требует другой трейт: `trait Greeter: Named` значит, что каждый `Greeter` обязан быть и `Named`. |
| **правило сирот** (orphan rule) | Реализовать трейт для типа можно, только если трейт **или** тип определён в вашем собственном крейте. |
| **ассоциированная функция** | Функция в `impl` или в трейте без `self`; вызывается как `Type::function()`. |
| **ассоциированный тип** | Имя типа внутри трейта, которое каждая реализация заполняет сама (`type Output = …`). |
| **область видимости** (scope) | Набор имён, видимых в данном месте кода. Методы трейта можно вызывать, только если трейт находится в области видимости. |
| **`Debug`, `Display`** | Трейты для печати: `{:?}` — для программиста, `{}` — для пользователя. |

## 1. Что такое трейт

Представьте должностную инструкцию: «Водитель: должен уметь `start()`, `stop()`, `turn()`». Водителями
могут быть многие люди, каждый по-своему. Инструкция не говорит, *как* водить; она лишь перечисляет,
что вы обязаны уметь. **Трейт** — такая инструкция для типов.

Вот трейт `Shape` для фигур, у которых есть площадь:

```rust
trait Shape {
    fn area(&self) -> f64;
    fn name(&self) -> String;

    fn describe(&self) -> String {
        format!("{} with area {:.2}", self.name(), self.area())
    }
}

struct Circle {
    radius: f64,
}

struct Rectangle {
    width: f64,
    height: f64,
}

impl Shape for Circle {
    fn area(&self) -> f64 {
        std::f64::consts::PI * self.radius * self.radius
    }

    fn name(&self) -> String {
        String::from("circle")
    }
}

impl Shape for Rectangle {
    fn area(&self) -> f64 {
        self.width * self.height
    }

    fn name(&self) -> String {
        String::from("rectangle")
    }

    fn describe(&self) -> String {
        format!("{}x{} rectangle", self.width, self.height)
    }
}

fn main() {
    let c = Circle { radius: 2.0 };
    let r = Rectangle { width: 3.0, height: 4.5 };
    println!("{}", c.describe());
    println!("{}", r.describe());
}
```

**Что происходит:**

1. `trait Shape { … }` объявляет трейт. `fn area(&self) -> f64;` заканчивается на `;` и не имеет
   тела: это только *сигнатура*. Каждый реализующий тип обязан написать тело.
2. У `describe` **есть** тело. Это **метод по умолчанию**: любой тип, реализующий `Shape`, получает
   `describe` автоматически. Заметьте, что тело по умолчанию может вызывать другие методы трейта
   (`self.name()`, `self.area()`).
3. `impl Shape for Circle { … }` говорит «`Circle` — это `Shape`» и даёт два обязательных метода.
   `Circle` не пишет `describe`, поэтому использует метод по умолчанию.
4. `Rectangle` пишет и собственный `describe`. Он *переопределяет* метод по умолчанию.
5. Методы трейта вызываются обычной точкой: `c.describe()`.

```text
circle with area 12.57
3x4.5 rectangle
```

> 📏 **Соглашение:** имена трейтов пишутся в `CamelCase` и обычно называют умение или роль (`Shape`,
> `Display`, `Iterator`). Имена методов — в `snake_case`. Держите трейты маленькими: два-три метода,
> которые связаны между собой. Маленький трейт легко реализовать и легко понять.

## 2. Трейты как параметры и типы результата

Трейт полезен тем, что функция может сказать: «я принимаю любой тип, который является `Shape`».
Есть три записи, и все они означают одно и то же:

```rust
fn print_area<T: Shape>(shape: &T) {
    println!("area of the {} is {:.1}", shape.name(), shape.area());
}

fn print_description(shape: &impl Shape) {
    println!("{}", shape.describe());
}

fn print_both<A, B>(first: &A, second: &B)
where
    A: Shape,
    B: Shape,
{
    println!("{} and {}", first.name(), second.name());
}
```

**Что происходит:**

1. `T: Shape` — обобщённая функция с ограничением трейта (Урок 05.1). Используйте, когда два
   аргумента должны быть *одного* типа или когда тип нужно назвать ещё раз.
2. `&impl Shape` — короткая запись. Каждый аргумент `impl Shape` может быть своего типа.
3. `where` — то же ограничение, вынесенное под сигнатуру. Используйте, когда список длинный.

С `Circle` и `Rectangle` из примера выше:

```rust
fn main() {
    let c = Circle { radius: 2.0 };
    let r = Rectangle { width: 3.0, height: 4.5 };
    print_area(&c);
    print_area(&r);
    print_description(&c);
    print_description(&r);
    print_both(&c, &r);
}
```

```text
area of the circle is 12.6
area of the rectangle is 13.5
circle with area 12.57
3x4.5 rectangle
circle and rectangle
```

Функция может и *возвращать* «какой-то тип, реализующий трейт»:

```rust
fn unit_square() -> impl Shape {
    Rectangle { width: 1.0, height: 1.0 }
}
```

Вызывающий знает только «это `Shape`» и может вызвать `unit_square().describe()`. Ограничение:
функция обязана возвращать **один** конкретный тип. Она не может вернуть `Circle` в одной ветке и
`Rectangle` в другой. (Урок 06.1 покажет трейт-объекты, которые это позволяют.)

Если передать тип, не реализующий трейт, компилятор скажет это точно:

```rust
fn main() {
    print_area(&5);
}
```

```text
error[E0277]: the trait bound `{integer}: Shape` is not satisfied
  --> examples/err_bound.rs:18:16
   |
18 |     print_area(&5);
   |     ---------- ^^ the trait `Shape` is not implemented for `{integer}`
   |     |
   |     required by a bound introduced by this call
   |
help: the trait `Shape` is implemented for `Circle`
```

**Что это значит:** `print_area` требует `T: Shape`, а у числа `5` нет `impl Shape`. Строка `help:`
перечисляет типы, у которых он есть.

> ⚡ **Производительность:** функции с `T: Shape` или `impl Shape` используют **статическую
> диспетчеризацию**. Как и в Уроке 05.1, компилятор делает по одной копии для каждого конкретного
> типа, а вызов `area` определяется на этапе компиляции, без поиска во время выполнения. Это быстрый
> способ использования трейтов по умолчанию.

## 3. Стандартные трейты и `derive`

Стандартная библиотека определяет десятки трейтов. Несколько из них встречаются почти в каждой
программе:

| Трейт | Что вы получаете | Как получить |
|---|---|---|
| `Debug` | печать через `{:?}` для программистов | `#[derive(Debug)]` |
| `Clone` | `.clone()`: явная глубокая копия | `#[derive(Clone)]` |
| `Copy` | значение копируется автоматически при присваивании (Урок 03.3) | `#[derive(Clone, Copy)]` |
| `PartialEq` / `Eq` | `==` и `!=` | `#[derive(PartialEq, Eq)]` |
| `PartialOrd` / `Ord` | `<`, `>`, `.max()`, сортировка | `#[derive(PartialOrd, Ord)]` |
| `Hash` | можно использовать как ключ `HashMap` (Урок 04.9) | `#[derive(Hash)]` |
| `Default` | `Type::default()`, разумное «пустое» значение | `#[derive(Default)]` |
| `Display` | печать через `{}` для пользователей и `.to_string()` | пишется вручную |
| `From` / `Into` | преобразования между типами | `From` пишется вручную |
| `Add`, `Mul`, … | операторы `+`, `*`, … | пишутся вручную |

Атрибут `#[derive(...)]` просит компилятор написать очевидную реализацию. Для структуры «очевидная»
значит «по полям, по очереди»:

```rust
use std::collections::HashMap;

#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash, PartialOrd, Ord, Default)]
struct Version {
    major: u32,
    minor: u32,
}

fn main() {
    let a = Version { major: 1, minor: 4 };
    let b = Version { major: 1, minor: 10 };
    let c = a; // Copy: a is still usable
    println!("{:?} {:?}", a, b);
    println!("a == c: {}", a == c);
    println!("a < b: {}", a < b);
    println!("default: {:?}", Version::default());
    println!("max: {:?}", a.max(b));

    let mut notes: HashMap<Version, &str> = HashMap::new();
    notes.insert(a, "stable");
    notes.insert(b, "beta");
    println!("note for a: {:?}", notes.get(&a));
}
```

**Что происходит:**

1. Одна строка `derive` даёт `Version` девять умений.
2. `let c = a;` копирует `a` (благодаря `Copy`), поэтому `a` остаётся доступной в следующей строке.
3. `a < b` сравнивает поле за полем в том порядке, в котором они записаны: сначала `major` (1 и 1:
   равны), затем `minor` (4 < 10), поэтому `a < b` истинно. **Порядок полей важен** для
   выведенного `Ord`.
4. `Version::default()` создаёт `Version { major: 0, minor: 0 }`: значение по умолчанию для каждого
   поля.
5. Благодаря `Hash` и `Eq` `Version` может быть ключом `HashMap`.

```text
Version { major: 1, minor: 4 } Version { major: 1, minor: 10 }
a == c: true
a < b: true
default: Version { major: 0, minor: 0 }
max: Version { major: 1, minor: 10 }
note for a: Some("stable")
```

Забыли `derive` — компилятор попросит его:

```rust
#[derive(Debug)]
struct Money {
    cents: i64,
}

fn main() {
    let a = Money { cents: 100 };
    let b = Money { cents: 100 };
    println!("{}", a == b);
}
```

```text
error[E0369]: binary operation `==` cannot be applied to type `Money`
 --> examples/err_compare.rs:9:22
  |
9 |     println!("{}", a == b);
  |                    - ^^ - Money
  |                    |
  |                    Money
  |
note: an implementation of `PartialEq` might be missing for `Money`
help: consider annotating `Money` with `#[derive(PartialEq)]`
```

> 📏 **Соглашение:** выводите `Debug` почти у каждого своего типа, чтобы его можно было печатать при
> отладке. Добавляйте `Clone`, `PartialEq` и `Default`, когда это имеет смысл. `Copy` выводите
> только для небольших типов, которые ведут себя как простые значения.

## 4. Ручная реализация стандартных трейтов

### `Display`: печать для пользователей

`{:?}` (`Debug`) — для вас, программиста. `{}` (`Display`) — для пользователя вашей программы, и
Rust не может угадать его формат, поэтому вы пишете его сами:

```rust
use std::fmt;

struct Temperature {
    celsius: f64,
}

impl fmt::Display for Temperature {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        write!(f, "{:.1} °C", self.celsius)
    }
}

fn main() {
    let t = Temperature { celsius: 21.456 };
    println!("now: {}", t);
    let text: String = t.to_string();
    println!("as text: {text} (length {})", text.len());
}
```

**Что происходит:**

1. `impl fmt::Display for Temperature` реализует трейт из `std::fmt`.
2. Единственный метод — `fmt`. Он получает `f` — `Formatter`, «перо вывода». `fmt::Result` — это
   `Result<(), fmt::Error>`, как в Уроке 04.10.
3. `write!(f, …)` работает как `println!`, но пишет в `f`, а не на экран. Его результат —
   возвращаемое значение.
4. После этого `{}` работает с `Temperature`, и `.to_string()` вы получаете бесплатно (Раздел 5
   объясняет почему).
5. `len()` — это число **байтов**, а не символов: `°` занимает 2 байта в UTF-8 (Unicode Transformation Format, 8-bit: стандартный способ хранения текста в виде байтов, Урок 04.8), поэтому
   текст `21.5 °C` имеет длину 8 байт.

```text
now: 21.5 °C
as text: 21.5 °C (length 8)
```

Без такой реализации печать через `{}` не компилируется. Здесь `p` — значение небольшой структуры `Point { x: i32, y: i32 }`, у которой нет реализации `Display`:

```text
error[E0277]: `Point` doesn't implement `std::fmt::Display`
 --> examples/err_nodisplay.rs:8:20
  |
8 |     println!("{}", p);
  |               --   ^ `Point` cannot be formatted with the default formatter
  |
  = note: in format strings you may be able to use `{:?}` (or `{:#?}` for pretty-print) instead
```

### `From` и `Into`: преобразования

`From<A> for B` говорит «`B` можно получить из `A`». Если вы написали его, `Into` вы получаете даром:

```rust
struct Celsius(f64);
struct Fahrenheit(f64);

impl From<Celsius> for Fahrenheit {
    fn from(c: Celsius) -> Self {
        Fahrenheit(c.0 * 9.0 / 5.0 + 32.0)
    }
}

fn show(t: impl Into<Fahrenheit>) {
    let f: Fahrenheit = t.into();
    println!("{:.1} °F", f.0);
}

fn main() {
    let boiling = Fahrenheit::from(Celsius(100.0));
    println!("{:.1}", boiling.0);
    let body: Fahrenheit = Celsius(36.6).into();
    println!("{:.1}", body.0);
    show(Celsius(0.0));
    show(Fahrenheit(70.0));
}
```

**Что происходит:**

1. `struct Celsius(f64);` — *кортежная структура* (Урок 04.1): одно безымянное поле, читается как
   `.0`.
2. `impl From<Celsius> for Fahrenheit` определяет преобразование. `Self` — это `Fahrenheit`.
3. `Fahrenheit::from(Celsius(100.0))` и `Celsius(36.6).into()` делают одно и то же. Для `into()`
   нужна аннотация типа (`let body: Fahrenheit`), чтобы компилятор знал целевой тип.
4. `show` принимает `impl Into<Fahrenheit>`: `Celsius` или `Fahrenheit` (каждый тип преобразуется
   сам в себя). Это делает функции гибкими для вызывающего.

```text
212.0
97.9
32.0 °F
70.0 °F
```

> 💡 **Совет:** оператор `?` (Урок 04.10) использует `From` для преобразования ошибок. Поэтому `?`
> можно писать в функции, возвращающей другой тип ошибки, как только реализован `From`.

### `Add`: операторы — это трейты

`a + b` — просто вызов метода `add` трейта `std::ops::Add`. Реализуйте трейт — и оператор заработает
для вашего типа. Это называется **перегрузкой операторов**:

```rust
use std::ops::{Add, Mul};

#[derive(Debug, Clone, Copy, PartialEq, Default)]
struct Vec2 {
    x: f64,
    y: f64,
}

impl Add for Vec2 {
    type Output = Vec2;

    fn add(self, other: Vec2) -> Vec2 {
        Vec2 { x: self.x + other.x, y: self.y + other.y }
    }
}

impl Mul<f64> for Vec2 {
    type Output = Vec2;

    fn mul(self, k: f64) -> Vec2 {
        Vec2 { x: self.x * k, y: self.y * k }
    }
}

fn main() {
    let a = Vec2 { x: 1.0, y: 2.0 };
    let b = Vec2 { x: 0.5, y: -1.0 };
    println!("{:?}", a + b);
    println!("{:?}", a * 3.0);
    println!("{:?}", Vec2::default() + a);
}
```

**Что происходит:**

1. `type Output = Vec2;` — это **ассоциированный тип**: каждая реализация сама указывает, какой тип
   возвращает оператор. (`Iterator` использует ту же идею; Урок 05.5.)
2. `impl Add for Vec2` означает `Vec2 + Vec2`. `impl Mul<f64> for Vec2` означает `Vec2 * f64`.
   `<f64>` — тип справа от `*`.
3. `add` принимает `self` по значению. Здесь это нормально, потому что `Vec2` — `Copy`, и `a`
   остаётся доступным.
4. `Vec2::default()` берётся из `derive(Default)` и является нулевым вектором.

```text
Vec2 { x: 1.5, y: 1.0 }
Vec2 { x: 3.0, y: 6.0 }
Vec2 { x: 1.0, y: 2.0 }
```

> 📏 **Соглашение:** перегружайте оператор, только когда его смысл очевиден (сложение векторов,
> денег, промежутков времени). `+`, делающий что-то неожиданное, затрудняет чтение кода.

## 5. Другие возможности трейтов

### Ограничения на блоках `impl` и трейты с `Self`

Реализовать трейт или добавить методы можно только для типов, удовлетворяющих ограничению:

```rust
use std::fmt::{self, Display};

struct Wrapper<T> {
    value: T,
}

impl<T: Display> Display for Wrapper<T> {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        write!(f, "[{}]", self.value)
    }
}

impl<T: PartialOrd> Wrapper<T> {
    fn bigger_than(&self, other: &T) -> bool {
        self.value > *other
    }
}

fn main() {
    let w = Wrapper { value: 7 };
    println!("{} {}", w, w.bigger_than(&5));
    let s = Wrapper { value: "text" };
    println!("{}", s);
}
```

**Что происходит:** `Wrapper<T>` является `Display` **только когда** `T` — `Display`. Метод
`bigger_than` существует, только когда `T` — `PartialOrd`. Именно так `Vec<T>` является `Debug`,
только если `T` — `Debug`.

```text
[7] true
[text]
```

В трейте могут быть и функции без `self`, возвращающие `Self`. В сочетании с ограничением можно
писать обобщённый код, который создаёт значения:

```rust
trait Zero {
    fn zero() -> Self;
}

impl Zero for i32 {
    fn zero() -> Self {
        0
    }
}

impl Zero for f64 {
    fn zero() -> Self {
        0.0
    }
}

fn sum_all<T: Zero + std::ops::Add<Output = T> + Copy>(list: &[T]) -> T {
    let mut total = T::zero();
    for &x in list {
        total = total + x;
    }
    total
}

fn main() {
    println!("{}", sum_all(&[1, 2, 3]));
    println!("{}", sum_all(&[0.5, 0.25]));
    println!("{}", sum_all::<i32>(&[]));
}
```

**Что происходит:** `T::zero()` вызывает ассоциированную функцию того типа, которым является `T`.
Мы даже реализовали собственный трейт для встроенных типов `i32` и `f64` (это разрешено: трейт наш,
см. правило сирот ниже). `Add<Output = T>` говорит, что сумма двух значений `T` снова даёт `T`.

```text
6
0.75
0
```

### Сквозные реализации (blanket)

Реализация для *всех* типов, удовлетворяющих ограничению, называется **сквозной (blanket)**:

```rust
use std::fmt::Display;

trait Shout {
    fn shout(&self) -> String;
}

impl<T: Display> Shout for T {
    fn shout(&self) -> String {
        format!("{}!", self.to_string().to_uppercase())
    }
}

fn main() {
    println!("{}", "hello".shout());
    println!("{}", 42.shout());
    println!("{}", 2.5.shout());
}
```

**Что происходит:** каждый тип с `Display` — сейчас и в будущем — получает `shout`. Стандартная
библиотека делает то же самое: `impl<T: Display> ToString for T` — причина, по которой у любого типа
с `Display` есть `.to_string()`.

```text
HELLO!
42!
2.5!
```

### Супертрейты

Трейт может требовать другой трейт. `Greeter: Named` означает «чтобы быть `Greeter`, тип должен
быть ещё и `Named`», поэтому методы `Greeter` по умолчанию могут использовать `name()`:

```rust
trait Named {
    fn name(&self) -> String;
}

trait Greeter: Named {
    fn greet(&self) -> String {
        format!("Hello, {}!", self.name())
    }
}

struct Student {
    name: String,
}

impl Named for Student {
    fn name(&self) -> String {
        self.name.clone()
    }
}

impl Greeter for Student {}

fn main() {
    let s = Student { name: String::from("Dilnoza") };
    println!("{}", s.greet());
}
```

**Что происходит:** у `impl Greeter for Student {}` пустое тело: всё берётся из методов по
умолчанию, а `Named` реализован отдельно. Если забыть `impl Named`, компилятор откажется.

```text
Hello, Dilnoza!
```

### Правило сирот (orphan rule)

Реализовать трейт для типа можно, только если **трейт или тип принадлежит вашему крейту**.
Например, нельзя реализовать стандартный `Display` для стандартного `Vec<i32>`:

```rust
use std::fmt;

impl fmt::Display for Vec<i32> {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        write!(f, "a vector")
    }
}
```

```text
error[E0117]: only traits defined in the current crate can be implemented for types defined outside of the crate
 --> examples/err_orphan.rs:3:1
  |
3 | impl fmt::Display for Vec<i32> {
  | ^^^^^^^^^^^^^^^^^^^^^^--------
  |                       |
  |                       `Vec` is not defined in the current crate
  |
  = note: define and implement a trait or new type instead
```

**Зачем нужно это правило:** если бы две разные библиотеки обе реализовали `Display` для `Vec<i32>`,
компилятор не смог бы выбрать между ними. **Исправление** — подсказка в последней строке: оберните
чужой тип в свою структуру (`struct Numbers(Vec<i32>);`, паттерн «newtype», его разбирает Этап 08)
и реализуйте трейт для обёртки.

### Чего трейты пока не умеют

У всего кода выше есть ограничение: `Vec<Shape>` невозможен, потому что вектор хранит значения
**одного** типа, а `Circle` и `Rectangle` — два типа. Список «разных фигур» требует *трейт-объектов*
(`dyn Shape`), которым посвящён [Урок 06.1](../06-rust-advanced/01-trait-objects-and-dynamic-dispatch.md).
Сегодня вы используете трейты через обобщения, а это более быстрый вариант.

## Частые ошибки

**1. Не хватает обязательного метода** — `error[E0046]: not all trait items implemented, missing: 'name'`:

```rust
trait Shape {
    fn area(&self) -> f64;
    fn name(&self) -> String;
}

struct Square(f64);

impl Shape for Square {
    fn area(&self) -> f64 {
        self.0 * self.0
    }
}
```

```text
error[E0046]: not all trait items implemented, missing: `name`
 --> examples/err_missing.rs:8:1
  |
3 |     fn name(&self) -> String;
  |     ------------------------- `name` from trait
...
8 | impl Shape for Square {
  | ^^^^^^^^^^^^^^^^^^^^^ missing `name` in implementation
```

**Что это значит:** трейт перечисляет `name` без тела, поэтому каждая реализация обязана его
предоставить. Напишите метод или дайте ему тело по умолчанию в трейте.

**2. Вызов метода трейта, когда трейт не в области видимости.**

```rust
mod animals {
    pub trait Speak {
        fn speak(&self) -> String;
    }

    pub struct Dog;

    impl Speak for Dog {
        fn speak(&self) -> String {
            String::from("woof")
        }
    }
}

fn main() {
    let d = animals::Dog;
    println!("{}", d.speak());
}
```

```text
error[E0599]: no method named `speak` found for struct `Dog` in the current scope
  --> examples/err_scope.rs:17:22
   |
17 |     println!("{}", d.speak());
   |                      ^^^^^ method not found in `Dog`
   |
   = help: items from traits can only be used if the trait is in scope
help: trait `Speak` which provides `speak` is implemented but not in scope; perhaps you want to import it
   |
 1 + use crate::animals::Speak;
   |
```

**Что это значит:** метод принадлежит *трейту*, а не `Dog`. Чтобы вызвать его, импортируйте трейт
через `use`. Компилятор даже даёт точную строку, которую нужно добавить.

**3. Забытый `derive`** (Раздел 3): `E0369` для `==`, `E0277` для `{:?}` без `Debug`. Добавьте
derive.

**4. Реализация чужого трейта для чужого типа** (Раздел 5): `E0117`. Используйте обёртку newtype.

**5. Возврат двух разных типов из `-> impl Trait`.** Функция должна вернуть один конкретный тип;
используйте перечисление или дождитесь трейт-объектов (Урок 06.1).

## Попробуйте сами

1. Перейдите в репозиторий и начните ветку для этого урока:

   ```bash
   cd ~/projects/rust
   git switch main
   git switch -c lesson/05-02-traits
   ```

2. Сохраните программу `Shape` из раздела 1 как `code/p05-lessons/examples/traits_shape.rs` и
   запустите её:

   ```bash
   cd code/p05-lessons
   cargo run --example traits_shape
   ```

   Добавьте `print_area` и `print_both` из раздела 2 и запустите ещё раз.

3. Сделайте коммит. Коммитьте **мелко и часто**: после каждого работающего примера, а не один раз в
   конце дня. Указывайте в сообщении *тип*, чтобы журнал читался с одного взгляда: `feat` (новое),
   `fix`, `test`, `docs`, `refactor`.

   ```bash
   cd ~/projects/rust
   git add .
   git commit -m "feat(p05): Shape trait with a default method"
   ```

4. Добавьте пример с `Display` (`traits_display.rs`). Перед коммитом проверьте, что именно вы
   собираетесь сохранить:

   ```bash
   git add .
   git diff --staged --stat
   ```

   ```text
    code/p05-lessons/examples/traits_display.rs | 18 ++++++++++++++++++
    1 file changed, 18 insertions(+)
   ```

   `--staged` показывает то, что находится в «области подготовки» (Урок 01.3), а `--stat` — только
   сводку по файлам. Это хороший последний взгляд перед каждым коммитом: *лежит ли там только то,
   что я хотел?*

   ```bash
   git commit -m "feat(p05): Display for Temperature"
   ```

5. Выполните Упражнение 5 (`Display` для стека, с тестами) и сделайте коммит `test(p05): …`. Затем
   посмотрите на историю, которую вы написали:

   ```bash
   git log --oneline
   ```

   ```text
   56700dc test(p05): tests for Display of Stack
   475649f feat(p05): Display for Temperature
   474f069 feat(p05): Shape trait with a default method
   8f26f07 chore: start
   ```

   (Ваши хеши будут другими, а ниже этих строк вы увидите и более старые коммиты из прошлых уроков.) Читатель может понять ход работы только по сообщениям.

6. Слейте урок в `main` (`git switch main`, `git merge lesson/05-02-traits`) и удалите ветку, как в
   Уроке 05.1.

## Упражнения

### Упражнение 1 — Животные (легко)

Определите трейт `Animal` с обязательными методами `name(&self) -> String` и `sound(&self) -> String`
и методом по умолчанию `speak(&self) -> String`, который возвращает `"<name> says <sound>"`.
Реализуйте его для `Dog` (структура-единица) и для `Cat` (структура с полем `name`), причём `Cat`
переопределяет `speak` и возвращает `"<name> purrs"`. Напишите тесты для обоих.

<details>
<summary>Подсказка</summary>

Метод по умолчанию может вызывать `self.name()` и `self.sound()` внутри `format!`. Структура-единица
объявляется как `struct Dog;` и используется как значение `Dog`. Переопределите метод, написав
`fn speak` внутри `impl Animal for Cat`.

</details>

<details>
<summary>Ответ</summary>

```rust
trait Animal {
    fn name(&self) -> String;
    fn sound(&self) -> String;

    fn speak(&self) -> String {
        format!("{} says {}", self.name(), self.sound())
    }
}

struct Dog;
struct Cat {
    name: String,
}

impl Animal for Dog {
    fn name(&self) -> String {
        String::from("Dog")
    }

    fn sound(&self) -> String {
        String::from("woof")
    }
}

impl Animal for Cat {
    fn name(&self) -> String {
        self.name.clone()
    }

    fn sound(&self) -> String {
        String::from("meow")
    }

    fn speak(&self) -> String {
        format!("{} purrs", self.name)
    }
}

fn main() {
    println!("{}", Dog.speak());
    println!("{}", Cat { name: String::from("Murka") }.speak());
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn default_speak_uses_name_and_sound() {
        assert_eq!(Dog.speak(), "Dog says woof");
    }

    #[test]
    fn cat_overrides_speak() {
        let c = Cat { name: String::from("Tom") };
        assert_eq!(c.speak(), "Tom purrs");
    }
}
```

```text
Dog says woof
Murka purrs
```

Почему это работает: `Dog` использует `speak` по умолчанию, `Cat` заменяет его. `cargo test --example <имя>`
сообщает `2 passed`.

</details>

### Упражнение 2 — Display с тестом (легко)

Реализуйте `Display` для `struct Temperature { celsius: f64 }` так, чтобы `21.456` печаталось как
`21.5 °C`. Напишите тесты, вызывающие `.to_string()` для положительного и отрицательного значения.

<details>
<summary>Подсказка</summary>

Используйте `write!(f, "{:.1} °C", self.celsius)`. `{:.1}` означает «одна цифра после запятой».
`to_string()` доступен, как только реализован `Display`.

</details>

<details>
<summary>Ответ</summary>

```rust
use std::fmt;

struct Temperature {
    celsius: f64,
}

impl fmt::Display for Temperature {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        write!(f, "{:.1} °C", self.celsius)
    }
}

fn main() {
    println!("{}", Temperature { celsius: -3.04 });
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn displays_one_decimal_and_unit() {
        let t = Temperature { celsius: 21.456 };
        assert_eq!(t.to_string(), "21.5 °C");
    }

    #[test]
    fn displays_negative_values() {
        assert_eq!(Temperature { celsius: -3.04 }.to_string(), "-3.0 °C");
    }
}
```

```text
-3.0 °C
```

Почему это работает: проверка *текста*, который производит объект, — самый простой способ
протестировать `Display`.

</details>

### Упражнение 3 — Сложить два вектора (средне)

Реализуйте `Add` для `Vec2 { x: f64, y: f64 }` (выведите `Debug`, `Clone`, `Copy`, `PartialEq`,
`Default`). Напишите один тест на сложение двух векторов и один тест, что прибавление
`Vec2::default()` ничего не меняет.

<details>
<summary>Подсказка</summary>

`impl Add for Vec2 { type Output = Vec2; fn add(self, other: Vec2) -> Vec2 { … } }` и
`use std::ops::Add;`. Чтобы сравнивать векторы в `assert_eq!`, нужны `PartialEq` и `Debug`.

</details>

<details>
<summary>Ответ</summary>

```rust
use std::ops::Add;

#[derive(Debug, Clone, Copy, PartialEq, Default)]
struct Vec2 {
    x: f64,
    y: f64,
}

impl Add for Vec2 {
    type Output = Vec2;

    fn add(self, other: Vec2) -> Vec2 {
        Vec2 { x: self.x + other.x, y: self.y + other.y }
    }
}

fn main() {
    let a = Vec2 { x: 1.0, y: 2.0 };
    println!("{:?}", a + Vec2::default());
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn adds_component_by_component() {
        let a = Vec2 { x: 1.0, y: 2.0 };
        let b = Vec2 { x: 0.5, y: -1.0 };
        assert_eq!(a + b, Vec2 { x: 1.5, y: 1.0 });
    }

    #[test]
    fn default_is_the_zero_vector() {
        let a = Vec2 { x: 4.0, y: 5.0 };
        assert_eq!(a + Vec2::default(), a);
    }
}
```

```text
Vec2 { x: 1.0, y: 2.0 }
```

Почему это работает: числа в этих тестах (1.0, 0.5, …) точно представимы в двоичном виде, поэтому
сравнивать дробные числа через `==` здесь безопасно. В общем случае сравнивайте дробные числа с
небольшим допуском (Урок 05.6).

</details>

### Упражнение 4 — Преобразование температуры в обе стороны (средне)

Для `Celsius(f64)` и `Fahrenheit(f64)` реализуйте `From<Celsius> for Fahrenheit` **и**
`From<Fahrenheit> for Celsius`. Напишите обобщённую функцию `print_in_fahrenheit<T: Into<Fahrenheit>>(t: T)`,
которая печатает значение с одним десятичным знаком и текстом ` °F`. Проверьте тестами, что 100 °C
это 212 °F и что круговое преобразование 36.6 °C возвращает (почти) то же число.

<details>
<summary>Подсказка</summary>

°F = °C × 9 / 5 + 32, а °C = (°F − 32) × 5 / 9. В тесте кругового преобразования сравнивайте с
допуском: `(back.0 - 36.6).abs() < 1e-9`. Выведите `Debug` и `PartialEq` у структур, чтобы работал
`assert_eq!`.

</details>

<details>
<summary>Ответ</summary>

```rust
#[derive(Debug, PartialEq)]
struct Celsius(f64);
#[derive(Debug, PartialEq)]
struct Fahrenheit(f64);

impl From<Celsius> for Fahrenheit {
    fn from(c: Celsius) -> Self {
        Fahrenheit(c.0 * 9.0 / 5.0 + 32.0)
    }
}

impl From<Fahrenheit> for Celsius {
    fn from(f: Fahrenheit) -> Self {
        Celsius((f.0 - 32.0) * 5.0 / 9.0)
    }
}

fn print_in_fahrenheit<T: Into<Fahrenheit>>(t: T) {
    let f: Fahrenheit = t.into();
    println!("{:.1} °F", f.0);
}

fn main() {
    print_in_fahrenheit(Celsius(100.0));
    print_in_fahrenheit(Fahrenheit(70.0));
    let c: Celsius = Fahrenheit(212.0).into();
    println!("{:?}", c);
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn boiling_point() {
        assert_eq!(Fahrenheit::from(Celsius(100.0)), Fahrenheit(212.0));
    }

    #[test]
    fn round_trip_is_close() {
        let back: Celsius = Fahrenheit::from(Celsius(36.6)).into();
        assert!((back.0 - 36.6).abs() < 1e-9);
    }
}
```

```text
212.0 °F
70.0 °F
Celsius(100.0)
```

Почему это работает: каждый `From` автоматически даёт соответствующий `Into`. Допуск во втором тесте
нужен потому, что арифметика с плавающей точкой редко бывает точной.

</details>

### Упражнение 5 — Display для обобщённого стека (сложно)

Возьмите `Stack<T>` из Урока 05.1 и реализуйте `Display` для `Stack<T>` **только когда** `T: Display`.
Формат — `[1, 2, 3]`, а для пустого стека — `[]`. Напишите тесты для обоих случаев.

<details>
<summary>Подсказка</summary>

`impl<T: Display> Display for Stack<T>`. Начните с `write!(f, "[")?;`. Пройдите циклом по
`&self.items` с переменной `first` (типа `bool`), чтобы писать `", "` перед каждым элементом, кроме
первого. `?` после каждого `write!` пробрасывает ошибку. Закончите `write!(f, "]")`.

</details>

<details>
<summary>Ответ</summary>

```rust
use std::fmt::{self, Display};

struct Stack<T> {
    items: Vec<T>,
}

impl<T: Display> Display for Stack<T> {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        write!(f, "[")?;
        let mut first = true;
        for item in &self.items {
            if !first {
                write!(f, ", ")?;
            }
            write!(f, "{}", item)?;
            first = false;
        }
        write!(f, "]")
    }
}

fn main() {
    let s = Stack { items: vec![1, 2, 3] };
    println!("{}", s);
    let empty: Stack<String> = Stack { items: vec![] };
    println!("{}", empty);
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn shows_items_separated_by_commas() {
        let s = Stack { items: vec!["a", "b"] };
        assert_eq!(s.to_string(), "[a, b]");
    }

    #[test]
    fn empty_stack_is_two_brackets() {
        let s: Stack<i32> = Stack { items: Vec::new() };
        assert_eq!(s.to_string(), "[]");
    }
}
```

```text
[1, 2, 3]
[]
```

Почему это работает: ограничение `T: Display` на `impl` делает `Stack<T>` печатаемым ровно тогда,
когда печатаемы его элементы, и один и тот же код обслуживает числа, `&str` и `String`.

</details>

## Вопросы для самопроверки

1. Чем *обязательный* метод трейта отличается от метода *по умолчанию*?

<details>
<summary>Ответ</summary>

У обязательного метода есть только сигнатура (заканчивается на `;`); каждый реализующий тип обязан написать его тело. У метода по умолчанию тело есть в трейте; реализующие типы получают его бесплатно и могут переопределить.

</details>

2. Чем отличаются `fn f<T: Shape>(x: &T)` и `fn f(x: &impl Shape)`?

<details>
<summary>Ответ</summary>

Для одного параметра они означают одно и то же. Форма с `T` позволяет назвать тип ещё раз (например, чтобы два аргумента были одного типа или чтобы использовать `T` в теле); форма с `impl` короче, когда это не нужно.

</details>

3. Почему `println!("{}", p)` не работает для структуры и как это исправить двумя способами?

<details>
<summary>Ответ</summary>

Потому что структура не реализует `Display`, а Rust не может угадать пользовательский формат. Либо реализуйте `std::fmt::Display` вручную, либо печатайте через `{:?}`, добавив `#[derive(Debug)]`.

</details>

4. Что даёт `impl From<A> for B`, кроме `B::from(a)`?

<details>
<summary>Ответ</summary>

Бесплатную реализацию `Into<B> for A`, поэтому можно писать `a.into()` (если целевой тип известен), а оператор `?` может использовать её для преобразования типов ошибок.

</details>

5. Сформулируйте правило сирот и назовите одну вещь, которую оно запрещает.

<details>
<summary>Ответ</summary>

Реализовать трейт для типа можно, только если трейт или тип определён в вашем крейте. Поэтому нельзя реализовать стандартный `Display` для стандартного `Vec<i32>`. Вместо этого оберните тип в собственную структуру (newtype).

</details>

6. Почему нельзя вызвать `d.speak()`, если `Speak` определён в модуле, который вы не импортировали?

<details>
<summary>Ответ</summary>

Методы трейта можно вызывать только когда трейт находится в области видимости. Метод принадлежит трейту, поэтому нужно написать `use path::Speak;`. Компилятор (`E0599`) подсказывает, какую строку `use` добавить.

</details>

## Итоги

- Трейт — именованный набор сигнатур методов. `impl Trait for Type` даёт типу эти умения.
- У обязательных методов есть только сигнатуры; у методов по умолчанию есть тела, и их можно переопределять.
- Принимайте «всё с этим умением» через `T: Trait`, `&impl Trait` или `where`; это статическая диспетчеризация без затрат во время выполнения.
- `#[derive(Debug, Clone, PartialEq, …)]` пишет стандартные реализации за вас. Порядок полей влияет на выведенное упорядочивание.
- `Display` пишите вручную для текста пользователю, `From` — для преобразований (`Into` получаете бесплатно), `Add`/`Mul` — для операторов.
- Сквозные реализации, супертрейты и ограничения на блоках `impl` позволяют одному определению трейта покрыть много типов.
- Правило сирот запрещает чужой трейт для чужого типа; используйте обёртку newtype.
- Привычка с Git: маленькие коммиты с типом (`feat`, `test`, `fix`) и `git diff --staged --stat` перед каждым коммитом.

## Источники и что почитать

- [The Rust Programming Language — 10.2 Traits](https://doc.rust-lang.org/book/ch10-02-traits.html) — официальная глава об определении и использовании трейтов.
- [Rust by Example — Traits](https://doc.rust-lang.org/rust-by-example/trait.html) — короткие запускаемые примеры, включая `derive` и перегрузку операторов.
- [std::fmt::Display](https://doc.rust-lang.org/std/fmt/trait.Display.html) — трейт и примеры его реализации.
- [std::convert::From](https://doc.rust-lang.org/std/convert/trait.From.html) — преобразования и связь с `Into`.
- [std::ops::Add](https://doc.rust-lang.org/std/ops/trait.Add.html) — операторные трейты.
- [std::default::Default](https://doc.rust-lang.org/std/default/trait.Default.html) — `Default` и `derive(Default)`.
- [Conventional Commits 1.0.0](https://www.conventionalcommits.org/en/v1.0.0/) — формат сообщений, на котором держатся `feat:`, `fix:`, `test:`.

---

[🏠 Начало](../START-HERE.md) · [← 05.1 Обобщённые типы](01-generics.md) · [📋 Этап 05](README.md) · [05.3 Времена жизни →](03-lifetimes.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
