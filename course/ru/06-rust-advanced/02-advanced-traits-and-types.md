<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 06.1 Трейт-объекты и динамическая диспетчеризация](01-trait-objects-and-dynamic-dispatch.md) · [📋 Этап 06](README.md) · [06.3 Образцы подробно →](03-patterns-in-depth.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/06-rust-advanced/02-advanced-traits-and-types.md) <!-- nav -->

# Урок 06.2 — Продвинутые трейты и типы

> **Этап 06 — Rust: продвинутый уровень** · Урок 2 из 12 · ⏱ около 4 часов (3 занятия)

## Что вы узнаете

- Ассоциированные типы и когда выбирать их вместо обобщённого трейта.
- Параметры типа по умолчанию и перегрузку операторов (`+`, `-`, ...).
- Полный (квалифицированный) синтаксис вызова, чтобы вызвать именно тот метод, который нужен.
- Супертрейты, паттерн newtype и правило сирот (orphan rule).
- Псевдонимы типов, тип «никогда» `!` и типы без известного размера (`str`, `[T]`, `dyn Trait`).
- Указатели на функции (`fn`) в сравнении с замыканиями и способ вернуть замыкание.

## Перед началом

- Вы прошли [Урок 06.1 — Трейт-объекты и динамическая диспетчеризация](01-trait-objects-and-dynamic-dispatch.md).
- Вы знаете трейты ([Урок 05.2](../05-rust-intermediate/02-traits.md)), обобщения (generics) ([Урок 05.1](../05-rust-intermediate/01-generics.md)), замыкания ([Урок 05.4](../05-rust-intermediate/04-closures.md)) и итераторы ([Урок 05.5](../05-rust-intermediate/05-iterators.md)).
- Создайте учебный проект: `cargo new code/p06-advanced-traits`. Каждый пример ниже можно по очереди помещать в `src/main.rs`.

## Новые слова

| Слово | Значение |
|---|---|
| **ассоциированный тип** (associated type) | Имя типа, объявленное внутри трейта (`type Item;`). Каждая реализация выбирает настоящий тип. |
| **параметр типа по умолчанию** (default type parameter) | Обобщённый параметр со значением по умолчанию, записывается `<Rhs = Self>`. |
| **перегрузка операторов** (operator overloading) | Заставить оператор, например `+`, работать для вашего типа, реализовав трейт из `std::ops`. |
| **полный синтаксис вызова** (fully qualified syntax) | Длинная форма вызова `<Type as Trait>::function()`, которая точно указывает, функцию какого трейта нужно вызвать. |
| **супертрейт** (supertrait) | Трейт, который требует другой трейт: в `trait A: B` трейт `B` — супертрейт для `A`. |
| **паттерн newtype** (newtype pattern) | Кортежная структура с одним полем, которая оборачивает другой тип, чтобы дать ему новый смысл или новые реализации трейтов. |
| **правило сирот** (orphan rule) | Реализовать трейт для типа можно, только если трейт или тип определён в вашем собственном крейте. |
| **псевдоним типа** (type alias) | Новое имя для существующего типа, записывается `type Name = ...;`. Это не новый тип. |
| **тип «никогда»** (never type, `!`) | Тип выражения, которое никогда не даёт значения, например `panic!()` или `continue`. |
| **DST** (Dynamically Sized Type, тип с динамическим размером) | Тип, размер которого неизвестен на этапе компиляции: `str`, `[T]` и `dyn Trait`. |
| **указатель на функцию** (function pointer) | Значение типа `fn(A) -> B`, которое указывает на функцию. |
| **KiB / MiB** (кибибайт / мебибайт) | 1 KiB = 1024 байта, 1 MiB = 1024 KiB. Используются в Упражнении 1. |
| **JSON** (JavaScript Object Notation) | Популярный текстовый формат данных, например `{"name":"ana"}`. Используется в Упражнении 2. |
| **XML** (Extensible Markup Language, расширяемый язык разметки) | Текстовый формат данных с тегами, например `<user>ana</user>`. Используется в Упражнении 2. |

## 1. Ассоциированные типы

В [Уроке 05.5](../05-rust-intermediate/05-iterators.md) вы использовали `Iterator`, в котором есть строка `type Item;`. Это **ассоциированный тип**: тип-заполнитель внутри трейта. Тип, реализующий трейт, подставляет вместо него настоящий.

Вот наш собственный трейт — контейнер, который может отдать первый элемент:

```rust
// Трейт с ассоциированным типом: каждая реализация выбирает ОДИН Item.
trait Container {
    type Item;
    fn first(&self) -> Option<&Self::Item>;
    fn size(&self) -> usize;
}

struct Playlist {
    songs: Vec<String>,
}

struct Temperatures(Vec<f64>);

impl Container for Playlist {
    type Item = String;
    fn first(&self) -> Option<&String> {
        self.songs.first()
    }
    fn size(&self) -> usize {
        self.songs.len()
    }
}

impl Container for Temperatures {
    type Item = f64;
    fn first(&self) -> Option<&f64> {
        self.0.first()
    }
    fn size(&self) -> usize {
        self.0.len()
    }
}

// Вызывающему не нужно писать тип элемента.
fn is_empty<C: Container>(c: &C) -> bool {
    c.size() == 0
}

fn main() {
    let playlist = Playlist {
        songs: vec![String::from("Intro"), String::from("Finale")],
    };
    let temps = Temperatures(vec![]);

    println!("first song: {:?}", playlist.first());
    println!("playlist empty? {}", is_empty(&playlist));
    println!("first temperature: {:?}", temps.first());
    println!("temps empty? {}", is_empty(&temps));
}
```

```text
first song: Some("Intro")
playlist empty? false
first temperature: None
temps empty? true
```

**Что происходит:**

1. `type Item;` внутри `trait Container` говорит: «каждая реализация назовёт один тип элемента».
2. `Playlist` задаёт `type Item = String;`, а `Temperatures` — `type Item = f64;`. Внутри трейта `Self::Item` означает «тип элемента этой реализации».
3. `is_empty<C: Container>` обобщена только по контейнеру. Вызывающему не нужно писать тип элемента.
4. `first()` возвращает `Option<&Self::Item>`: для плейлиста это `Option<&String>`, для температур — `Option<&f64>`.

**Ассоциированный тип или обобщённый параметр?** Сравните с обобщённым трейтом:

```rust
// Обобщённый трейт: один тип может реализовать его много раз с разными T.
trait Convert<T> {
    fn convert(&self) -> T;
}

struct Celsius(f64);

impl Convert<f64> for Celsius {
    fn convert(&self) -> f64 {
        self.0 * 1.8 + 32.0
    }
}

impl Convert<String> for Celsius {
    fn convert(&self) -> String {
        format!("{:.1} C", self.0)
    }
}

fn main() {
    let t = Celsius(21.5);
    let fahrenheit: f64 = t.convert();
    let text: String = t.convert();
    println!("{} / {}", fahrenheit, text);
}
```

```text
70.7 / 21.5 C
```

**Что происходит:**

1. У `Convert<T>` есть параметр типа. `Celsius` реализует его дважды: для `f64` и для `String`.
2. Компилятор не может угадать, какой `convert` вы имеете в виду, поэтому в каждом `let` мы пишем нужный тип (`let fahrenheit: f64 = ...`).

Практическое правило:

| Что использовать | Когда |
|---|---|
| **Ассоциированный тип** | Для каждого реализующего типа есть ровно **один** разумный выбор (`Item` одного итератора). Вызывающим не нужно писать тип. |
| **Обобщённый параметр** | Один тип может реализовать трейт **несколько раз** с разными типами (`From<A>` и `From<B>` для одного целевого типа). |

## 2. Параметры типа по умолчанию и перегрузка операторов

Rust позволяет задать обобщённому параметру значение по умолчанию. Самый известный пример — трейт `Add`, на котором держится оператор `+`:

```rust
trait Add<Rhs = Self> {
    type Output;
    fn add(self, rhs: Rhs) -> Self::Output;
}
```

`Rhs` (right-hand side, правая часть) по умолчанию равен `Self`: «по умолчанию складывают значение того же типа». `Output` — ассоциированный тип, который сообщает, что даёт `a + b`. Реализация трейтов из `std::ops` называется **перегрузкой операторов**:

```rust
use std::ops::Add;

#[derive(Debug, Clone, Copy, PartialEq)]
struct Point {
    x: i32,
    y: i32,
}

// `Add` — это `trait Add<Rhs = Self>`: по умолчанию Rhs — тот же тип.
impl Add for Point {
    type Output = Point;
    fn add(self, other: Point) -> Point {
        Point {
            x: self.x + other.x,
            y: self.y + other.y,
        }
    }
}

#[derive(Debug)]
struct Millimeters(u32);
struct Meters(u32);

// Здесь мы выбираем другой Rhs: миллиметры плюс метры.
impl Add<Meters> for Millimeters {
    type Output = Millimeters;
    fn add(self, other: Meters) -> Millimeters {
        Millimeters(self.0 + other.0 * 1000)
    }
}

fn main() {
    let a = Point { x: 1, y: 2 };
    let b = Point { x: 10, y: 20 };
    println!("{:?}", a + b);

    let length = Millimeters(250) + Meters(2);
    println!("{:?}", length);
}
```

```text
Point { x: 11, y: 22 }
Millimeters(2250)
```

**Что происходит:**

1. `impl Add for Point` использует значение по умолчанию `Rhs = Self`, поэтому `a + b` складывает две `Point`. Запись `a + b` — то же, что `a.add(b)`.
2. `Point` — `Copy`, поэтому использование `a` и `b` в сложении не «забирает» их.
3. `impl Add<Meters> for Millimeters` выбирает другую правую часть. Теперь миллиметры плюс метры допустимы, и метры переводятся в миллиметры.
4. 250 + 2 * 1000 = 2250.

> 📏 **Соглашение:** перегружайте оператор, только когда его смысл очевиден (`+` для чисел, векторов, промежутков времени). Не используйте `+` для чего-то несвязанного, например «отправить сообщение».

## 3. Полный синтаксис вызова

У типа может быть метод с тем же именем, что у метода из трейта, и у двух трейтов могут быть методы с одинаковыми именами. Rust должен знать, какой из них вы хотите:

```rust
trait Pilot {
    fn fly(&self);
}

trait Wizard {
    fn fly(&self);
}

struct Human;

impl Pilot for Human {
    fn fly(&self) {
        println!("This is your captain speaking.");
    }
}

impl Wizard for Human {
    fn fly(&self) {
        println!("Up!");
    }
}

impl Human {
    fn fly(&self) {
        println!("*waves arms*");
    }
}

trait Animal {
    fn baby_name() -> String;
}

struct Dog;

impl Dog {
    fn baby_name() -> String {
        String::from("Spot")
    }
}

impl Animal for Dog {
    fn baby_name() -> String {
        String::from("puppy")
    }
}

fn main() {
    let person = Human;
    person.fly(); // метод самого Human
    Pilot::fly(&person); // короткая форма: имя трейта
    <Human as Wizard>::fly(&person); // полная форма: <Type as Trait>

    println!("{}", Dog::baby_name());
    println!("{}", <Dog as Animal>::baby_name());
}
```

```text
*waves arms*
This is your captain speaking.
Up!
Spot
puppy
```

**Что происходит:**

1. У `Human` есть собственный `fly`, а также `fly` из `Pilot` и `fly` из `Wizard`.
2. `person.fly()` выбирает метод, написанный прямо на типе.
3. `Pilot::fly(&person)` называет трейт и передаёт значение первым аргументом (`self`).
4. `<Human as Wizard>::fly(&person)` — это **полный синтаксис вызова**: `<Type as Trait>::function(...)`. Он работает всегда.
5. `Dog::baby_name()` выбирает ассоциированную функцию, написанную на `Dog`. У ассоциированных функций без `self` нет значения, которое подсказало бы выбор, поэтому для версии из трейта нужно писать `<Dog as Animal>::baby_name()`.

А что будет, если попробовать короткую форму для функции трейта?

```rust
trait Animal {
    fn baby_name() -> String;
}

struct Dog;

impl Animal for Dog {
    fn baby_name() -> String {
        String::from("puppy")
    }
}

fn main() {
    println!("{}", Animal::baby_name());
}
```

```text
error[E0790]: cannot call associated function on trait without specifying the corresponding `impl` type
  --> src/main.rs:14:20
   |
 2 |     fn baby_name() -> String;
   |     ------------------------- `Animal::baby_name` defined here
...
14 |     println!("{}", Animal::baby_name());
   |                    ^^^^^^^^^^^^^^^^^^^ cannot call associated function of trait
   |
help: use the fully-qualified path to the only available implementation
   |
14 |     println!("{}", <Dog as Animal>::baby_name());
   |                    +++++++       +
```

Компилятор предлагает полный путь. У `Animal::baby_name()` нет `self`, поэтому Rust не может знать, версию какого типа вы имеете в виду.

## 4. Супертрейты

Иногда трейту для работы нужен другой трейт. **Супертрейт** говорит: «чтобы реализовать меня, тип должен реализовать ещё и тот трейт». Тогда метод трейта по умолчанию может пользоваться методами супертрейта:

```rust
use std::fmt::Display;

// `OutlinePrint: Display` означает: каждый тип с OutlinePrint обязан быть и Display.
trait OutlinePrint: Display {
    fn outline(&self) {
        let text = self.to_string(); // разрешено благодаря супертрейту
        let line = "*".repeat(text.len() + 4);
        println!("{}", line);
        println!("* {} *", text);
        println!("{}", line);
    }
}

struct Point {
    x: i32,
    y: i32,
}

impl Display for Point {
    fn fmt(&self, f: &mut std::fmt::Formatter) -> std::fmt::Result {
        write!(f, "({}, {})", self.x, self.y)
    }
}

impl OutlinePrint for Point {}

fn main() {
    Point { x: 3, y: -7 }.outline();
}
```

```text
***********
* (3, -7) *
***********
```

**Что происходит:**

1. `trait OutlinePrint: Display` означает, что у каждого типа с `OutlinePrint` обязан быть и `Display`.
2. Благодаря этому метод по умолчанию `outline` может вызвать `self.to_string()` (метод, который есть у каждого типа с `Display`).
3. `impl OutlinePrint for Point {}` пустая: она использует только метод по умолчанию. Она компилируется лишь потому, что `Point` реализует `Display` выше.
4. В тексте `(3, -7)` 7 символов, поэтому строка из звёздочек содержит 7 + 4 = 11 символов: текст плюс `* ` слева и ` *` справа.

> 💡 **Совет:** супертрейт вы уже встречали в [Уроке 06.1](01-trait-objects-and-dynamic-dispatch.md): `trait Plugin: Debug`. Супертрейтами часто бывают `Debug`, `Display`, `Send` или `Clone`.

## 5. Паттерн newtype и правило сирот

Реализовать трейт для типа можно, только если **ваш крейт определяет трейт или тип**. Это **правило сирот** (orphan rule). Оно вас защищает: если бы два крейта могли оба реализовать `Display` для `Vec<String>`, компилятор не смог бы выбрать, и ваша программа ломалась бы при обновлении зависимости.

```rust
use std::fmt;

impl fmt::Display for Vec<String> {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        write!(f, "[{}]", self.join(", "))
    }
}

fn main() {
    let v = vec![String::from("a"), String::from("b")];
    println!("{}", v);
}
```

```text
error[E0117]: only traits defined in the current crate can be implemented for types defined outside of the crate
 --> src/main.rs:3:1
  |
3 | impl fmt::Display for Vec<String> {
  | ^^^^^^^^^^^^^^^^^^^^^^-----------
  |                       |
  |                       `Vec` is not defined in the current crate
  |
  = note: impl doesn't have any local type before any uncovered type parameters
  = note: for more information see https://doc.rust-lang.org/reference/items/implementations.html#orphan-rules
  = note: define and implement a trait or new type instead

For more information about this error, try `rustc --explain E0117`.
error: could not compile `demo` (bin "demo") due to 1 previous error
```

Последняя строка подсказки даёт стандартное решение — **паттерн newtype**. Вы оборачиваете чужой тип в собственную кортежную структуру. Обёртка — ваш тип, поэтому вы можете реализовать для неё любой трейт:

```rust
use std::fmt;
use std::ops::Deref;

// Newtype: кортежная структура ровно с одним полем.
struct Names(Vec<String>);

impl fmt::Display for Names {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        write!(f, "[{}]", self.0.join(", "))
    }
}

// Deref позволяет вызывать методы Vec (len, iter, ...) прямо у Names.
impl Deref for Names {
    type Target = Vec<String>;
    fn deref(&self) -> &Vec<String> {
        &self.0
    }
}

// Newtype также не даёт перепутать значения с одним и тем же внутренним типом.
struct UserId(u64);
struct OrderId(u64);

fn load_user(id: UserId) -> String {
    format!("user #{}", id.0)
}

fn load_order(id: OrderId) -> String {
    format!("order #{}", id.0)
}

fn main() {
    let names = Names(vec![String::from("Ana"), String::from("Bo")]);
    println!("{}", names);
    println!("{} names", names.len());

    println!("{}", load_user(UserId(7)));
    println!("{}", load_order(OrderId(7)));
    // load_user(OrderId(7)); // не скомпилируется: ожидался `UserId`, найден `OrderId`
}
```

```text
[Ana, Bo]
2 names
user #7
order #7
```

**Что происходит:**

1. `struct Names(Vec<String>);` — это newtype: структура с одним полем. Во время работы она ничего не стоит; у обёртки та же раскладка в памяти, что у внутреннего `Vec`.
2. `impl fmt::Display for Names` разрешена, потому что `Names` определён здесь. `self.0` — внутренний вектор.
3. Обёртка скрывает методы внутреннего типа. `impl Deref for Names` возвращает их: `names.len()` работает, потому что `Names` можно разыменовать до `Vec<String>`.
4. `UserId(u64)` и `OrderId(u64)` оба оборачивают `u64`, но это разные типы. Передать `OrderId` в `load_user` — ошибка компиляции. Newtype бесплатно превращает путаницу в ошибки компиляции.

> 📏 **Соглашение:** используйте newtype для идентификаторов и единиц измерения (`UserId`, `Meters`). Функцию `fn charge(user: u64, amount: u64)` легко вызвать с перепутанными аргументами; `fn charge(user: UserId, amount: Cents)` — нет.

## 6. Псевдонимы типов

**Псевдоним типа** даёт существующему типу другое имя. Он не создаёт новый тип, поэтому преобразование не нужно:

```rust
use std::fmt;

#[derive(Debug)]
struct AppError(String);

impl fmt::Display for AppError {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        write!(f, "app error: {}", self.0)
    }
}

// Псевдоним даёт типу новое ИМЯ. Это не новый тип.
type Result<T> = std::result::Result<T, AppError>;
type Callback = Box<dyn Fn(i32) -> i32 + Send + 'static>;
type Meters = u32;

fn parse_age(text: &str) -> Result<u8> {
    text.trim()
        .parse::<u8>()
        .map_err(|e| AppError(format!("bad age {:?}: {}", text, e)))
}

fn run(callbacks: &[Callback], start: i32) -> i32 {
    callbacks.iter().fold(start, |acc, cb| cb(acc))
}

fn main() {
    println!("{:?}", parse_age(" 41 "));
    match parse_age("abc") {
        Ok(age) => println!("age {}", age),
        Err(e) => println!("{}", e),
    }

    let callbacks: Vec<Callback> = vec![Box::new(|x| x + 1), Box::new(|x| x * 2)];
    println!("{}", run(&callbacks, 5));

    let distance: Meters = 100;
    let other: u32 = distance; // тот же тип, преобразование не нужно
    println!("{}", distance + other);
}
```

```text
Ok(41)
app error: bad age "abc": invalid digit found in string
12
200
```

**Что происходит:**

1. `type Result<T> = std::result::Result<T, AppError>;` фиксирует тип ошибки. Все функции программы могут писать `Result<u8>` вместо `Result<u8, AppError>`. Стандартная библиотека делает так же с `std::io::Result<T>`.
2. `type Callback = Box<dyn Fn(i32) -> i32 + Send + 'static>;` сокращает длинный тип, который используется в нескольких местах (Clippy предупреждает о «слишком сложных типах»).
3. `type Meters = u32;` — просто другое имя. `distance` можно напрямую присвоить переменной `u32`. Поэтому псевдоним **не** добавляет безопасности; newtype добавляет.

## 7. Тип «никогда» `!`

Некоторые выражения никогда не завершаются значением: `panic!()`, `continue`, `break`, `return`, `std::process::exit()`, бесконечный `loop {}`. Их тип — `!`, он называется **типом «никогда»** (never type, также «пустой тип»). Поскольку значения такого типа не существует, компилятор может превратить `!` в любой другой тип. Поэтому `continue` может стоять в ветке `match`, где ожидается `u32`:

```rust
fn main() {
    let inputs = ["10", "x", "30", "", "50"];
    let mut total: u32 = 0;

    for text in inputs {
        // У `continue` тип `!`, поэтому он подходит там, где ожидается u32.
        let number: u32 = match text.parse() {
            Ok(n) => n,
            Err(_) => continue,
        };
        total += number;
    }
    println!("total = {}", total);

    let value: u32 = match "7".parse::<u32>() {
        Ok(n) => n,
        Err(_) => panic!("not a number"), // у panic! тоже тип `!`
    };
    println!("value = {}", value);
}
```

```text
total = 90
value = 7
```

**Что происходит:**

1. `text.parse()` пытается превратить каждый текст в `u32` ([Урок 02.7](../02-programming-basics/07-reading-input-and-parsing.md)). `"x"` и `""` не получаются.
2. В ветке `Ok(n)` значение — `u32`. В ветке `Err(_)` у `continue` тип `!`, который подходит как `u32`, потому что он никогда не даёт значения: цикл переходит к следующему элементу.
3. Поэтому складывается только `10 + 30 + 50 = 90`.
4. У `panic!` тоже тип `!`, поэтому `Err(_) => panic!(...)` компилируется. Функции вроде `Option::unwrap` опираются на ту же идею: ветка с ошибкой вызывает панику и никогда не возвращает значение.

Можно написать функцию, которая никогда не возвращается, с `-> !`:

```rust
fn fail(message: &str) -> ! {
    println!("fatal: {}", message);
    std::process::exit(3)
}

fn main() {
    let value = 5;
    let x: i32 = if value > 0 {
        10
    } else {
        fail("math is broken")
    };
    println!("x = {}", x);
    fail("stopping here");
}
```

```text
x = 10
fatal: stopping here
```

**Что происходит:**

1. `fail` печатает сообщение и вызывает `std::process::exit(3)`, которая завершает программу с кодом выхода 3. Её тип — `!`, поэтому `fail` можно объявить как `-> !`.
2. В `let x: i32 = if ... { 10 } else { fail(...) }` ветка `else` имеет тип `!` и подходит там, где нужен `i32`.
3. `value` равно 5, поэтому условие `value > 0` истинно, и первая строка печатает `x = 10`.
4. Затем `fail("stopping here")` завершает программу. В терминале ([Урок 00.4](../00-getting-started/04-terminal-basics.md)) команда `echo $?` печатает код выхода последней программы; выполните её после `cargo run` или после запуска собранного бинарного файла, и вы увидите `3`.

## 8. Типы без известного размера (DST) и `?Sized`

У большинства типов размер известен при компиляции (`i32` — 4 байта). У некоторых нет. Их называют **DST** (Dynamically Sized Types, типы с динамическим размером):

- `str` (текст любой длины). Используется как `&str` или `Box<str>`.
- `[T]` (срез любой длины). Используется как `&[T]`.
- `dyn Trait` (любой тип, реализующий трейт, [Урок 06.1](01-trait-objects-and-dynamic-dispatch.md)).

DST всегда находится **за указателем**, и это «толстый» указатель, который несёт недостающую информацию (длину или vtable). Компилятору нужны размеры, чтобы разместить локальные переменные в стеке, поэтому он отказывается создавать DST по значению:

```text
error[E0277]: the size for values of type `str` cannot be known at compilation time
 --> src/main.rs:2:9
  |
2 |     let text: str = *"hello";
  |         ^^^^ doesn't have a size known at compile-time
  |
  = help: the trait `Sized` is not implemented for `str`
  = note: all local variables must have a statically known size
help: consider borrowing here
  |
2 |     let text: &str = *"hello";
  |               +

For more information about this error, try `rustc --explain E0277`.
error: could not compile `demo` (bin "demo") due to 1 previous error
```

Есть маркерный трейт `Sized`, он означает «размер известен на этапе компиляции». В каждой обобщённой функции у `T` автоматически есть ограничение `T: Sized`. Чтобы принимать DST, нужно отказаться от него с помощью `?Sized` («возможно, безразмерный»):

```rust
use std::fmt::Display;

// `?Sized` означает «T может быть типом без известного размера».
fn show<T: Display + ?Sized>(value: &T) {
    println!("show: {}", value);
}

fn main() {
    show(&42); // T = i32, у него известный размер (Sized)
    show("hello"); // T = str, у него НЕТ известного размера: работает только благодаря ?Sized

    let boxed: Box<str> = "boxed text".into(); // владеющий str в куче
    println!("{} ({} bytes)", boxed, boxed.len());
}
```

```text
show: 42
show: hello
boxed text (10 bytes)
```

**Что происходит:**

1. `show<T: Display + ?Sized>(value: &T)` принимает ссылку на любой `T`, имеющий размер или нет. У самой ссылки размер известен, так что всё в порядке.
2. `show(&42)` использует `T = i32`. `show("hello")` использует `T = str`, и это возможно только благодаря `?Sized`.
3. `Box<str>` — это владеющий текст в куче без запаса ёмкости. `"boxed text".into()` строит его из `&str`. Его длина — 10 байт.

## 9. Указатели на функции, замыкания и возврат замыканий

В [Уроке 05.4](../05-rust-intermediate/04-closures.md) вы передавали замыкания в функции через `F: Fn(...)`. Можно передать и обычную функцию. В Rust есть тип «указатель на функцию»: `fn(i32) -> i32` (со строчной `fn`; не путайте его с трейтом `Fn`).

```rust
fn add_one(x: i32) -> i32 {
    x + 1
}

// `fn(i32) -> i32` — тип УКАЗАТЕЛЯ на функцию (строчная fn).
fn apply_twice(f: fn(i32) -> i32, x: i32) -> i32 {
    f(f(x))
}

// `F: Fn(i32) -> i32` принимает функции И замыкания, захватывающие переменные.
fn apply_generic<F: Fn(i32) -> i32>(f: F, x: i32) -> i32 {
    f(x)
}

fn make_adder(n: i32) -> impl Fn(i32) -> i32 {
    move |x| x + n
}

fn make_op(kind: &str) -> Box<dyn Fn(i32) -> i32> {
    if kind == "double" {
        Box::new(|x| x * 2)
    } else {
        Box::new(|x| x - 1)
    }
}

fn main() {
    println!("{}", apply_twice(add_one, 5));
    println!("{}", apply_twice(|x| x * 3, 5)); // замыкание без захвата тоже подходит

    let offset = 100;
    println!("{}", apply_generic(|x| x + offset, 5)); // захватывает `offset`

    let words: Vec<String> = [1, 2, 3].iter().map(ToString::to_string).collect();
    println!("{:?}", words);

    // Вариант перечисления с данными — тоже функция: Some(1), Some(2).
    let options: Vec<Option<i32>> = [1, 2].into_iter().map(Some).collect();
    println!("{:?}", options);

    let add5 = make_adder(5);
    println!("{}", add5(10));
    println!("{} {}", make_op("double")(8), make_op("other")(8));
}
```

```text
7
45
105
["1", "2", "3"]
[Some(1), Some(2)]
15
16 7
```

**Что происходит:**

1. `apply_twice(f: fn(i32) -> i32, x)` принимает обычную функцию. `apply_twice(add_one, 5)` — это `add_one(add_one(5))` = 7.
2. Замыкание, которое **ничего не захватывает** (`|x| x * 3`), можно преобразовать в указатель на функцию, поэтому оно тоже принимается: (5 * 3) * 3 = 45.
3. `apply_generic<F: Fn(i32) -> i32>` принимает любое замыкание, в том числе захватывающее `offset`.
4. `ToString::to_string` — функция, поэтому её можно передать в `map` напрямую. Так же `Some` — функция из `T` в `Option<T>`, поэтому `.map(Some)` работает. То же верно для конструкторов кортежных структур и вариантов перечислений.
5. `make_adder` возвращает `impl Fn(i32) -> i32`: одно конкретное замыкание со скрытым типом. `move` копирует `n` внутрь замыкания.
6. `make_op` может возвращать разные замыкания из разных веток, поэтому каждое замыкание упаковывается в `Box`, а тип — `Box<dyn Fn(i32) -> i32>` ([Урок 06.1](01-trait-objects-and-dynamic-dispatch.md)). `impl Fn` здесь не скомпилировался бы.

А если замыкание захватывает переменную, а параметр — указатель на функцию?

```text
error[E0308]: mismatched types
 --> src/main.rs:7:26
  |
7 |     println!("{}", apply(|x| x + offset, 5));
  |                    ----- ^^^^^^^^^^^^^^ expected fn pointer, found closure
  |                    |
  |                    arguments to this function are incorrect
  |
  = note: expected fn pointer `fn(i32) -> i32`
                found closure `{closure@src/main.rs:7:26: 7:29}`
note: closures can only be coerced to `fn` types if they do not capture any variables
 --> src/main.rs:7:34
  |
7 |     println!("{}", apply(|x| x + offset, 5));
  |                                  ^^^^^^ `offset` captured here
note: function defined here
 --> src/main.rs:1:4
  |
1 | fn apply(f: fn(i32) -> i32, x: i32) -> i32 {
  |    ^^^^^ -----------------

For more information about this error, try `rustc --explain E0308`.
error: could not compile `demo` (bin "demo") due to 1 previous error
```

| Что принимать в параметре | Что подходит | Когда использовать |
|---|---|---|
| `fn(i32) -> i32` | Функции и замыкания без захвата | Вы работаете с кодом на C (Урок 06.10) или нужно простое значение фиксированного размера. |
| `F: Fn(i32) -> i32` | Всё, что можно вызвать, с захватом и без | Почти всегда. Это вариант по умолчанию. |
| `Box<dyn Fn(i32) -> i32>` | Любой вызываемый объект, хранящийся со скрытым типом | Список разных вызываемых объектов или поле структуры. |

## Частые ошибки

### 1. Вызов ассоциированной функции трейта без указания типа

`Animal::baby_name()` даёт `E0790`. Пишите `<Dog as Animal>::baby_name()`.

### 2. Реализация чужого трейта для чужого типа

`impl Display for Vec<String>` даёт `E0117` (правило сирот). Оберните тип в newtype.

### 3. Ожидание, что псевдоним типа вас защитит

`type UserId = u64; type OrderId = u64;` — это один и тот же тип. Передача `OrderId` туда, где ждут `UserId`, компилируется. Используйте структуру newtype.

### 4. Забыли `?Sized`

`fn show<T: Display>(v: &T)` отвергает `show("hello")` с сообщением «the size for values of type `str` cannot be known at compilation time». Добавьте `+ ?Sized`.

### 5. Передача замыкания с захватом как `fn(...)`

`E0308 ... expected fn pointer, found closure`, см. раздел 9. Замените параметр на `impl Fn(...)`.

## Попробуйте сами

1. Откройте `~/projects/rust/code/p06-advanced-traits`.
2. Скопируйте пример с `Point` и `Add` (раздел 2) в `src/main.rs` и запустите `cargo run`.
3. Добавьте `impl Sub for Point` (трейт `Sub` из `std::ops`, метод `sub`). Напечатайте `b - a`. Должно получиться `Point { x: 9, y: 18 }`.
4. Добавьте `impl std::ops::Neg for Point` (унарный минус: `-a`). У `Neg` есть только ассоциированный тип `Output` и метод `neg(self)`. Напечатайте `-a`; ожидается `Point { x: -1, y: -2 }`.
5. Выполните `cargo fmt` и `cargo clippy`.
6. Сделайте коммит: `git add code/p06-advanced-traits`, затем `git commit -m "feat(p06): operator overloading for Point"`.

## Упражнения

### Упражнение 1 — Байты с единицами измерения (легко)

Создайте `struct Bytes(u64)`. Реализуйте `Add` и `Sub`, а также `Display` так, чтобы `Bytes(1536)` печатался как `1.5 KiB`, `Bytes(1024 * 1024)` — как `1.0 MiB`, а `Bytes(512)` — как `512 B`. Напишите тесты для сложения и для трёх вариантов вывода.

<details>
<summary>Подсказка</summary>

Внутри `fmt` используйте `if self.0 >= 1024 * 1024 { ... } else if self.0 >= 1024 { ... } else { ... }`, а `{:.1}` печатает один знак после запятой. Преобразуйте через `self.0 as f64`.

</details>

<details>
<summary>Ответ</summary>

```rust
use std::fmt;
use std::ops::{Add, Sub};

#[derive(Debug, Clone, Copy, PartialEq)]
struct Bytes(u64);

impl Add for Bytes {
    type Output = Bytes;
    fn add(self, other: Bytes) -> Bytes {
        Bytes(self.0 + other.0)
    }
}

impl Sub for Bytes {
    type Output = Bytes;
    fn sub(self, other: Bytes) -> Bytes {
        Bytes(self.0 - other.0)
    }
}

impl fmt::Display for Bytes {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        if self.0 >= 1024 * 1024 {
            write!(f, "{:.1} MiB", self.0 as f64 / (1024.0 * 1024.0))
        } else if self.0 >= 1024 {
            write!(f, "{:.1} KiB", self.0 as f64 / 1024.0)
        } else {
            write!(f, "{} B", self.0)
        }
    }
}

fn main() {
    let total = Bytes(1000) + Bytes(536);
    println!("{}", total);
    println!("{}", Bytes(3 * 1024 * 1024 / 2));
    println!("{}", Bytes(900) - Bytes(100));
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn adds_sizes() {
        assert_eq!(Bytes(1) + Bytes(2), Bytes(3));
    }

    #[test]
    fn prints_human_units() {
        assert_eq!(Bytes(512).to_string(), "512 B");
        assert_eq!(Bytes(1536).to_string(), "1.5 KiB");
        assert_eq!(Bytes(1024 * 1024).to_string(), "1.0 MiB");
    }
}
```

```text
1.5 KiB
1.5 MiB
800 B
```

```text

running 2 tests
..
test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

Почему это работает: трейты `Add` и `Sub` придают смысл `+` и `-`. `Display` бесплатно даёт `to_string()`, которым пользуются тесты.

</details>

### Упражнение 2 — Два трейта, одно имя метода (средне)

У двух трейтов `Json` и `Xml` есть метод `fn render(&self) -> String`. `User` реализует оба. Напечатайте для одного и того же пользователя JSON-строку `{"name":"ana"}` и XML-строку `<user>ana</user>`.

<details>
<summary>Подсказка</summary>

`user.render()` неоднозначен. Используйте `<User as Json>::render(&user)` или `Xml::render(&user)`.

</details>

<details>
<summary>Ответ</summary>

```rust
trait Json {
    fn render(&self) -> String;
}

trait Xml {
    fn render(&self) -> String;
}

struct User {
    name: String,
}

impl Json for User {
    fn render(&self) -> String {
        format!("{{\"name\":\"{}\"}}", self.name)
    }
}

impl Xml for User {
    fn render(&self) -> String {
        format!("<user>{}</user>", self.name)
    }
}

fn main() {
    let user = User {
        name: String::from("ana"),
    };
    println!("{}", <User as Json>::render(&user));
    println!("{}", Xml::render(&user));
}
```

```text
{"name":"ana"}
<user>ana</user>
```

Почему это работает: путь называет трейт, и компилятор знает, какой `render` вызвать. Если у метода есть аргумент `self`, достаточно `Trait::method(&value)`, потому что реализацию выбирает тип `value`.

</details>

### Упражнение 3 — Newtype для температур (средне)

Создайте `Celsius(f64)` и `Fahrenheit(f64)`. Реализуйте `From<Celsius> for Fahrenheit` и `From<Fahrenheit> for Celsius`. Напишите тест, что 100 градусов Цельсия — это 212 градусов Фаренгейта, и тест, что преобразование туда и обратно возвращает исходное значение.

<details>
<summary>Подсказка</summary>

F = C * 9 / 5 + 32 и C = (F - 32) * 5 / 9. Числа с плавающей точкой сравнивайте с небольшим допуском: `(a - b).abs() < 1e-9`, но никогда через `==`.

</details>

<details>
<summary>Ответ</summary>

```rust
#[derive(Debug, Clone, Copy)]
struct Celsius(f64);

#[derive(Debug, Clone, Copy)]
struct Fahrenheit(f64);

impl From<Celsius> for Fahrenheit {
    fn from(c: Celsius) -> Fahrenheit {
        Fahrenheit(c.0 * 9.0 / 5.0 + 32.0)
    }
}

impl From<Fahrenheit> for Celsius {
    fn from(f: Fahrenheit) -> Celsius {
        Celsius((f.0 - 32.0) * 5.0 / 9.0)
    }
}

fn main() {
    let boiling: Fahrenheit = Celsius(100.0).into();
    println!("{:?}", boiling);
    let freezing = Celsius::from(Fahrenheit(32.0));
    println!("{:?}", freezing);
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn celsius_to_fahrenheit() {
        let f: Fahrenheit = Celsius(100.0).into();
        assert!((f.0 - 212.0).abs() < 1e-9);
    }

    #[test]
    fn round_trip() {
        let c = Celsius::from(Fahrenheit::from(Celsius(36.6)));
        assert!((c.0 - 36.6).abs() < 1e-9);
    }
}
```

```text
Fahrenheit(212.0)
Celsius(0.0)
```

```text

running 2 tests
..
test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

Почему это работает: `From` автоматически даёт `.into()`. Два newtype нельзя случайно перепутать, хотя оба хранят `f64`.

</details>

### Упражнение 4 — Трейт с ассоциированным типом (сложно)

Определите `trait Source { type Item; fn next_item(&mut self) -> Option<Self::Item>; }`. Реализуйте его для `Countdown(u32)` (элементы `u32`, отсчёт вниз до 1) и для `Letters` (элементы `char`). Напишите обобщённую функцию `collect_all<S: Source>(source: S) -> Vec<S::Item>`. Протестируйте её на `Countdown(3)` и `Countdown(0)`.

<details>
<summary>Подсказка</summary>

Тип результата использует ассоциированный тип обобщённого параметра: `Vec<S::Item>`. Проходите циклом `while let Some(item) = source.next_item()`. Параметр должен быть `mut source`.

</details>

<details>
<summary>Ответ</summary>

```rust
trait Source {
    type Item;
    fn next_item(&mut self) -> Option<Self::Item>;
}

struct Countdown(u32);

impl Source for Countdown {
    type Item = u32;
    fn next_item(&mut self) -> Option<u32> {
        if self.0 == 0 {
            None
        } else {
            self.0 -= 1;
            Some(self.0 + 1)
        }
    }
}

struct Letters {
    rest: Vec<char>,
}

impl Source for Letters {
    type Item = char;
    fn next_item(&mut self) -> Option<char> {
        if self.rest.is_empty() {
            None
        } else {
            Some(self.rest.remove(0))
        }
    }
}

// `S::Item` — ассоциированный тип того источника, который нам передали.
fn collect_all<S: Source>(mut source: S) -> Vec<S::Item> {
    let mut items = Vec::new();
    while let Some(item) = source.next_item() {
        items.push(item);
    }
    items
}

fn main() {
    println!("{:?}", collect_all(Countdown(3)));
    println!(
        "{:?}",
        collect_all(Letters {
            rest: "hey".chars().collect()
        })
    );
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn countdown_collects_in_order() {
        assert_eq!(collect_all(Countdown(3)), vec![3, 2, 1]);
    }

    #[test]
    fn empty_source_gives_empty_vec() {
        assert!(collect_all(Countdown(0)).is_empty());
    }
}
```

```text
[3, 2, 1]
['h', 'e', 'y']
```

```text

running 2 tests
..
test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

Почему это работает: каждая реализация выбирает свой `Item`, а `S::Item` в сигнатуре функции следует этому выбору. Так устроен и `Iterator`.

</details>

## Вопросы для самопроверки

1. Когда ассоциированный тип лучше обобщённого параметра трейта?

<details>
<summary>Ответ</summary>

Когда у каждого реализующего типа есть ровно один естественный выбор для этого типа (как `Item` у итератора). Тогда вызывающие никогда не пишут тип. Обобщённый параметр нужен, когда один и тот же тип должен реализовать трейт несколько раз с разными аргументами типа (как `From<A>` и `From<B>`).

</details>

2. Что является значением по умолчанию для `Rhs` в `trait Add<Rhs = Self>` и что это означает?

<details>
<summary>Ответ</summary>

`Self`: если не сказано иное, складывается значение того же типа, поэтому `impl Add for Point` складывает две точки.

</details>

3. Что такое полный синтаксис вызова и когда он нужен?

<details>
<summary>Ответ</summary>

`<Type as Trait>::function(...)`. Он нужен, когда короткая форма неоднозначна: у двух трейтов одинаковое имя метода или это ассоциированная функция без `self`.

</details>

4. Что запрещает правило сирот и как его обойти?

<details>
<summary>Ответ</summary>

Оно запрещает реализовывать трейт для типа, если и трейт, и тип пришли из других крейтов. Обходят его паттерном newtype: оборачивают тип в собственную кортежную структуру и реализуют трейт для обёртки.

</details>

5. Почему псевдоним типа не защищает от путаницы между `UserId` и `OrderId`?

<details>
<summary>Ответ</summary>

Псевдоним — просто другое имя того же типа. `type UserId = u64;` и `type OrderId = u64;` — оба `u64`, и компилятор считает их равными. Структура newtype создаёт новый, другой тип.

</details>

6. Какой тип у `continue` и почему он может стоять в ветке `match`, которая должна вернуть `u32`?

<details>
<summary>Ответ</summary>

Его тип — `!` (never). Он никогда не даёт значения, поэтому компилятор разрешает преобразовать его в любой тип, в том числе в `u32`.

</details>

7. Почему нельзя написать `let s: str = ...` и что писать вместо этого?

<details>
<summary>Ответ</summary>

`str` — тип с динамическим размером: его размер неизвестен при компиляции, поэтому он не может быть локальной переменной. Используйте его за указателем: `&str`, `Box<str>` или `String` для владеющего растущего текста.

</details>

8. Чем отличается `fn(i32) -> i32` от `F: Fn(i32) -> i32`?

<details>
<summary>Ответ</summary>

`fn(i32) -> i32` — тип указателя на функцию: он принимает обычные функции и замыкания, которые ничего не захватывают. `Fn(i32) -> i32` — трейт: он принимает любой вызываемый объект, в том числе замыкания с захватом переменных. Для параметров предпочитайте трейт.

</details>

## Итоги

- Ассоциированный тип (`type Item;`) позволяет каждой реализации выбрать один тип; вызывающие его не пишут. Обобщённый параметр трейта допускает много реализаций для одного типа.
- Параметры типа по умолчанию (`Add<Rhs = Self>`) делают частые случаи короткими. Перегрузка операторов — это реализация трейтов из `std::ops`.
- `<Type as Trait>::function()` снимает неоднозначность между методами с одинаковыми именами и вызывает функции трейта без `self`.
- Супертрейт (`trait A: B`) требует `B` и позволяет методам `A` по умолчанию им пользоваться.
- Правило сирот требует, чтобы вам принадлежал трейт или тип. Паттерн newtype — обходной путь, а ещё он не даёт перепутать значения.
- Псевдоним типа — только новое имя; newtype — новый тип.
- `!` — тип выражений, которые никогда не завершаются; `str`, `[T]` и `dyn Trait` — DST, которые живут за указателями; `?Sized` принимает их в обобщённых функциях.
- `fn(...)` — тип указателя на функцию; `impl Fn` и `Box<dyn Fn>` возвращают замыкания.

## Источники и что почитать

- [The Rust Programming Language — 20.2 Advanced Traits](https://doc.rust-lang.org/book/ch20-02-advanced-traits.html) — ассоциированные типы, параметры по умолчанию, полный синтаксис вызова, супертрейты и паттерн newtype.
- [The Rust Programming Language — 20.3 Advanced Types](https://doc.rust-lang.org/book/ch20-03-advanced-types.html) — newtype, псевдонимы типов, тип «никогда» и типы с динамическим размером.
- [The Rust Programming Language — 20.4 Advanced Functions and Closures](https://doc.rust-lang.org/book/ch20-04-advanced-functions-and-closures.html) — указатели на функции и возврат замыканий.
- [std::ops::Add](https://doc.rust-lang.org/std/ops/trait.Add.html) — трейт `Add` с примерами; в модуле перечислены все остальные трейты операторов.
- [Rust by Example — New Type Idiom](https://doc.rust-lang.org/rust-by-example/generics/new_types.html) — короткий пример newtype.
- [The Rust Reference — Special types and traits](https://doc.rust-lang.org/reference/special-types-and-traits.html) — что именно означают `Sized` и `Copy`.

---

[🏠 Начало](../START-HERE.md) · [← 06.1 Трейт-объекты и динамическая диспетчеризация](01-trait-objects-and-dynamic-dispatch.md) · [📋 Этап 06](README.md) · [06.3 Образцы подробно →](03-patterns-in-depth.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
