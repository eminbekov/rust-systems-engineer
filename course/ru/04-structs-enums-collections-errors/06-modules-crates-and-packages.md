<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 04.5 if let, let else и образцы](05-if-let-let-else-and-patterns.md) · [📋 Этап 04](README.md) · [04.7 Векторы →](07-vectors.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/04-structs-enums-collections-errors/06-modules-crates-and-packages.md) <!-- nav -->

# Урок 04.6 — Модули, крейты и пакеты

> **Этап 04 — Структуры, перечисления, коллекции и ошибки** · Урок 6 из 12 · ⏱ около 4 часов (3 занятия)

## Что вы узнаете

- Три уровня организации кода в Rust: **пакет** (package), **крейт** (crate) и **модуль** (module), и как они сочетаются.
- Как писать модули в одном файле и в отдельных файлах и как пути (`crate::`, `self::`, `super::`) находят элементы.
- Приватность: всё приватно по умолчанию, `pub` открывает доступ, и почему это защищает ваши данные.
- Как `use` приносит имена в область видимости, как переименовывать через `as` и реэкспортировать через `pub use`.
- Как собрать пакет, в котором есть и библиотека, и программа, и где находятся модульные и интеграционные тесты.

## Перед началом

- Вы прошли [Урок 04.5 — if let, let else и образцы](05-if-let-let-else-and-patterns.md) и предыдущие уроки этого этапа (структуры, методы, перечисления, `Option`).
- Вы умеете создавать проект через `cargo new` и запускать `cargo run`, `cargo test` ([Урок 00.6 — Ваша первая программа на Rust](../00-getting-started/06-your-first-rust-program.md), [Урок 02.8 — Ваши первые тесты](../02-programming-basics/08-your-first-tests.md)).
- Ваш учебный репозиторий из [Урока 01.10](../01-git-and-version-control/10-project-your-learning-repo-and-checkpoint.md) готов; в конце мы сделаем коммит.

## Новые слова

| Слово | Значение |
|---|---|
| **модуль** (module) | Именованный контейнер кода внутри крейта (функции, типы, другие модули). Это пространство имён и граница приватности. |
| **крейт** (crate) | Единица, которую компилятор собирает за один раз. Крейт — это либо **бинарник** (программа с `main`), либо **библиотека** (код, которым пользуются другие). |
| **пакет** (package) | Одна папка с `Cargo.toml`. В ней один или несколько крейтов. `cargo new` создаёт пакет. |
| **корень крейта** (crate root) | Исходный файл, с которого компилятор начинает читать крейт: `src/main.rs` для бинарника, `src/lib.rs` для библиотеки. Его содержимое образует модуль `crate`. |
| **дерево модулей** (module tree) | Все модули крейта в виде дерева: корень крейта наверху, дочерние модули под ним. |
| **путь** (path) | «Адрес» элемента в дереве модулей, записанный через `::`, например `crate::geometry::circle::Circle`. |
| **абсолютный путь** (absolute path) | Путь, который начинается от корня крейта с `crate::` (или с имени внешнего крейта). |
| **относительный путь** (relative path) | Путь, который начинается в текущем модуле либо с `self::` или `super::`. |
| **`super`** | В пути: «родительский модуль». |
| **`pub`** | Делает элемент (функцию, структуру, модуль, поле …) видимым за пределами его модуля. Без `pub` он приватный. |
| **приватность** (privacy) | Правило о том, какой код что может видеть. |
| **инкапсуляция** (encapsulation) | Сокрытие внутренностей типа (приватные поля) и предложение небольшого набора публичных функций, чтобы внешний код не мог нарушить правила типа. |
| **`use`** | Объявление, которое приносит путь в область видимости под коротким именем. |
| **реэкспорт** (re-export) | `pub use path;`: делает элемент доступным по новому, обычно более короткому публичному пути. |
| **глоб-импорт** (glob import) | `use some::path::*;` импортирует все публичные элементы модуля. |
| **модульный тест** (unit test) | Тест в исходном файле тестируемого кода, в `#[cfg(test)] mod tests`. |
| **интеграционный тест** (integration test) | Тест в папке `tests/`. Он компилируется как отдельный крейт и может пользоваться только публичным API. |
| **API** (Application Programming Interface, программный интерфейс приложения) | Публичные функции и типы, которые крейт предлагает другим. |
| **пространство имён** (namespace) | Именованная область, в которой имена не могут конфликтовать с именами из других областей. |
| **TOML** (Tom's Obvious Minimal Language) | Простой формат файла `Cargo.toml`. |

## 1. Зачем вообще делить код?

До сих пор каждая программа была одним файлом. Для 50 строк это работает. В настоящей программе тысячи строк. Если всё лежит в одном файле и в одном пространстве имён:

- ничего не найти;
- двум вещам нужно одно и то же имя (`new`, `parse`, `Error`);
- кто угодно может менять что угодно, поэтому правила вроде «баланс никогда не бывает отрицательным» не защитить;
- компилятору приходится пересобирать всё после каждого изменения.

Rust даёт три инструмента: **модули**, чтобы организовывать и защищать код внутри крейта, **крейты** как единицы компиляции и распространения, и **пакеты** как единицы сборки и поставки через Cargo.

Представьте здание. **Пакет** — всё здание с планом адресов (`Cargo.toml`). **Крейт** — крыло, которое можно строить и открывать отдельно: вестибюль для посетителей (библиотека) или машинный зал, который всё запускает (бинарник). **Модули** — этажи и комнаты внутри крыла. Двери бывают запертыми (приватное) или открытыми (`pub`).

```text
package  p04-modules                        (folder with Cargo.toml)
├── library crate  p04_modules              (root file: src/lib.rs)
│     └── module tree:   crate
│                          ├── geometry
│                          │     ├── circle
│                          │     └── rectangle
│                          └── percent
└── binary crate   p04-modules              (root file: src/main.rs)
        uses the library crate like any other crate
```

Правила пакета (от Cargo): в пакете **хотя бы один** крейт, **не более одного** библиотечного крейта и **сколько угодно** бинарных.

## 2. Модули в одном файле

Сначала модули, записанные прямо в файле через `mod name { ... }` — так проще всего увидеть правила.

```rust
mod restaurant {
    pub mod front {
        pub fn seat_guest() {
            println!("front: guest seated");
        }

        fn clean_tables() {
            println!("front: tables cleaned");
        }

        pub fn close() {
            clean_tables();
            super::kitchen::wash_dishes();
        }
    }

    pub mod kitchen {
        pub fn wash_dishes() {
            println!("kitchen: dishes washed");
        }

        pub fn cook(order: &str) {
            self::prepare(order);
            crate::restaurant::front::seat_guest();
        }

        fn prepare(order: &str) {
            println!("kitchen: preparing {order}");
        }
    }
}

fn main() {
    // относительный путь от корня крейта (где находится main)
    restaurant::front::seat_guest();
    // абсолютный путь, начинающийся от корня крейта
    crate::restaurant::kitchen::cook("soup");
    restaurant::front::close();
}
```

**Что происходит:**

1. `mod restaurant { ... }` объявляет модуль. Внутри него `pub mod front` и `pub mod kitchen` — два дочерних модуля. Верхний уровень файла — это **корень крейта**, модуль с именем `crate`.
2. Дерево модулей такое: `crate` → `restaurant` → `front`, `kitchen`. Функция `main` лежит в `crate`.
3. В `main` путь `restaurant::front::seat_guest()` — **относительный**: он начинается от текущего модуля (корня крейта) и спускается вниз по именам. `crate::restaurant::kitchen::cook("soup")` — **абсолютный** путь для такого же спуска: он начинается со слова `crate`.
4. Внутри `front::close` выражение `super::kitchen::wash_dishes()` поднимается **на один уровень вверх** (`super` = родительский модуль, `restaurant`), а затем спускается в `kitchen`.
5. Внутри `kitchen::cook` выражение `self::prepare(order)` означает «элемент `prepare` в этом самом модуле» (`self` здесь — текущий модуль, а не тот `self`, что у методов). `crate::restaurant::front::seat_guest()` показывает абсолютный путь изнутри.
6. У `clean_tables` и `prepare` нет `pub`: они **приватные** и доступны только коду того же модуля (и его потомкам). `close` и `cook` публичные, поэтому могут их вызывать.

```text
front: guest seated
kitchen: preparing soup
front: guest seated
front: tables cleaned
kitchen: dishes washed
```

> 📏 **Соглашение:** для близких элементов используйте относительные пути, для далёких — пути с `crate::`. Если позже вы переместите модуль, абсолютные пути от `crate::` останутся верными для кода, который сам не перемещали. Выбирайте стиль, который уже принят в вашем проекте.

### Правила приватности

Правил всего два:

1. **Всё приватно по умолчанию.** Приватным элементом может пользоваться код того же модуля и модулей, вложенных в него (детей и внуков). Родители и посторонние — не могут.
2. **`pub` открывает элемент для внешнего мира.** Чтобы добраться до элемента по пути, **каждый** шаг пути должен быть вам виден: и модули по дороге, и сам элемент.

Дети могут смотреть на приватные элементы родителей; родители не могут заглядывать в приватные элементы детей. Ребёнок «внутри дома»; родитель — сосед.

Вызов приватной функции снаружи:

```rust
mod restaurant {
    pub mod front {
        fn clean_tables() {
            println!("front: tables cleaned");
        }
    }
}

fn main() {
    restaurant::front::clean_tables();
}
```

```text
error[E0603]: function `clean_tables` is private
  --> src/main.rs:10:24
   |
10 |     restaurant::front::clean_tables();
   |                        ^^^^^^^^^^^^ private function
   |
note: the function `clean_tables` is defined here
  --> src/main.rs:3:9
   |
 3 |         fn clean_tables() {
   |         ^^^^^^^^^^^^^^^^^
```

Модуль `front` публичный, а функция внутри — нет. Исправление: добавить `pub` или — лучше — спросить себя, нужна ли она посторонним на самом деле.

Приватный **модуль** блокирует весь путь:

```text
error[E0603]: module `front` is private
  --> src/main.rs:10:17
   |
10 |     restaurant::front::seat_guest();
   |                 ^^^^^  ---------- function `seat_guest` is not publicly re-exported
   |                 |
   |                 private module
```

(Так бывает, когда в коде написано `mod front` вместо `pub mod front`, даже если сама `seat_guest` помечена `pub`.)

### Приватность структур и перечислений

`pub struct` делает публичным *тип*, но у каждого **поля** своя приватность. Поля приватны, если не помечены `pub`. **Варианты перечислений** устроены иначе: если перечисление публичное, все его варианты публичны.

```rust
mod bank {
    pub struct Account {
        pub owner: String,
        balance_cents: u64,
    }

    impl Account {
        pub fn new(owner: &str) -> Account {
            Account {
                owner: String::from(owner),
                balance_cents: 0,
            }
        }

        pub fn deposit(&mut self, cents: u64) {
            self.balance_cents += cents;
        }

        pub fn balance_cents(&self) -> u64 {
            self.balance_cents
        }
    }

    #[derive(Debug)]
    pub enum Kind {
        Checking,
        Savings,
    }
}

fn main() {
    let mut account = bank::Account::new("alice");
    account.deposit(500);
    println!("{} has {} cents", account.owner, account.balance_cents());
    println!("{:?}", bank::Kind::Savings);
}
```

**Что происходит:**

1. `owner` помечено `pub`: прочитать его может кто угодно. `balance_cents` приватно: трогать это поле может только код внутри `mod bank`.
2. Посторонние получают баланс через **метод** `balance_cents()` и меняют его только через `deposit`. Это **инкапсуляция**: правила типа живут в одном месте.
3. `bank::Kind::Savings` работает без пометки каждого варианта `pub`: варианты публичного перечисления публичны.

```text
alice has 500 cents
Savings
```

Попытка тронуть приватное поле снаружи:

```text
error[E0616]: field `balance_cents` of struct `Account` is private
  --> src/main.rs:16:13
   |
16 |     account.balance_cents = 1_000_000;
   |             ^^^^^^^^^^^^^ private field
```

И попытка создать `Account`, самостоятельно перечислив поля (в обход `new`):

```text
error[E0451]: field `balance_cents` of struct `Account` is private
  --> src/main.rs:11:9
   |
 9 |     let account = bank::Account {
   |                   ------------- in this type
10 |         owner: String::from("mallory"),
11 |         balance_cents: 1_000_000,
   |         ^^^^^^^^^^^^^ private field
```

> 🔒 **Безопасность:** вот почему важны приватные поля. Если у структуры есть приватное поле, **единственный** способ создать или изменить экземпляр — через ваш код, поэтому ваши проверки выполняются всегда. Тип вроде `Percent` (раздел 5) может гарантировать «значение от 0 до 100», потому что снаружи никто не построит `Percent(250)`. Делайте публичную поверхность каждого модуля как можно меньше: каждый `pub`-элемент — обещание, которое придётся выполнять, и дверь, которой кто-то может злоупотребить.

## 3. `use`: более короткие имена

Писать везде `restaurant::front::seat_guest()` утомительно. Объявление **`use`** приносит путь в область видимости:

```rust
mod shapes {
    pub mod round {
        pub fn describe() -> &'static str {
            "round"
        }
    }

    pub mod flat {
        pub fn describe() -> &'static str {
            "flat"
        }
    }

    pub fn secret_sauce() -> u32 {
        42
    }
}

// принести модуль в область видимости: вызываем функции как `round::describe()`
use shapes::round;
// принести функцию напрямую
use shapes::secret_sauce;
// переименовать через `as`, удобно при столкновении имён
use shapes::flat::describe as describe_flat;
// несколько путей с общим префиксом
use std::fmt::{self, Display};
// два разных типа `Result`, различаемые через `as`
use std::fmt::Result as FmtResult;
use std::io::Result as IoResult;

struct Tag(u32);

impl Display for Tag {
    fn fmt(&self, f: &mut fmt::Formatter) -> FmtResult {
        write!(f, "tag-{}", self.0)
    }
}

fn pretend_to_read() -> IoResult<u32> {
    Ok(7)
}

fn main() {
    println!("{}", round::describe());
    println!("{}", describe_flat());
    println!("{}", secret_sauce());
    println!("{}", Tag(9));
    println!("{:?}", pretend_to_read().ok());
}
```

**Что происходит:**

1. `use shapes::round;` делает доступным имя `round`, поэтому мы пишем `round::describe()`. Имя модуля перед функцией сообщает читателю, откуда она взялась.
2. `use shapes::secret_sauce;` импортирует саму функцию, поэтому мы вызываем `secret_sauce()` напрямую.
3. `use ... describe as describe_flat;` переименовывает при импорте. Две сущности с именем `describe` конфликтовали бы; `as` этого избегает.
4. `use std::fmt::{self, Display};` — **вложенный импорт**: `self` означает сам `std::fmt`, так что становятся доступны и `fmt`, и `Display`. Это избавляет от повторения префикса.
5. В стандартной библиотеке больше одного типа с именем `Result` (один для форматирования, другой для ввода-вывода). Переименование через `as` позволяет им жить в одном файле.
6. `use` лишь создаёт короткое имя. Он ничего не копирует и ничего не стоит во время выполнения.

```text
round
flat
42
tag-9
Some(7)
```

> 📏 **Соглашение (идиоматичный `use`):**
> - Для **функций** импортируйте *родительский модуль* (`use shapes::round;`, затем `round::describe()`), чтобы вызов показывал, где живёт функция.
> - Для **типов** (структур, перечислений, трейтов) импортируйте *сам тип* (`use std::collections::HashMap;`).
> - Избегайте глоба `use module::*;` в обычном коде: непонятно, откуда берутся имена. Два допустимых применения: `use super::*;` в `mod tests` и модули в стиле prelude.
> - Располагайте строки `use` вверху файла в таком порядке: сначала стандартная библиотека, затем другие крейты, затем ваши собственные модули. `cargo fmt` их сортирует.

### Реэкспорт через `pub use`

Путь может быть длинным и глубоким, потому что так вы организовали файлы, но пользователям вашей библиотеки знать об этом не хочется. `pub use` делает элемент доступным по более короткому публичному пути:

```text
pub use geometry::circle::Circle;   // in lib.rs: now `my_crate::Circle` works too
```

Мы воспользуемся этим в разделе 5.

## 4. Уровни видимости

Помимо приватного и `pub` есть более узкие формы:

| Запись | Кому видно |
|---|---|
| *(ничего)* | текущему модулю и модулям внутри него |
| `pub(super)` | родительскому модулю (и всему внутри него) |
| `pub(crate)` | всему крейту, но не другим крейтам |
| `pub` | всем, кто может дойти до пути, включая другие крейты |

`pub(crate)` — правильный выбор для вспомогательных функций, которые используют несколько модулей вашего крейта, но которые не должны входить в публичный API.

## 5. Модули в отдельных файлах

В настоящих проектах каждый модуль лежит в собственном файле. Правило простое: запись `mod name;` (с точкой с запятой вместо блока) говорит компилятору «модуль `name` лежит в другом файле; загрузи его». Компилятор ищет его в таком порядке:

1. `name.rs` рядом с текущим файлом, либо
2. `name/mod.rs` (более старый стиль; работают оба).

Для модуля, вложенного в `name`, файл лежит в папке `name/`: `name/child.rs`.

Мы построим пакет с библиотечным и бинарным крейтами. Создайте его так:

```text
cd ~/projects/rust
cargo new code/p04-modules
cd code/p04-modules
mkdir tests src/geometry
```

Итоговая раскладка:

```text
p04-modules/
├── Cargo.toml
├── src/
│   ├── main.rs              ← корень бинарного крейта (программа)
│   ├── lib.rs               ← корень библиотечного крейта
│   ├── geometry.rs          ← модуль `geometry`  (объявляет дочерние модули)
│   ├── geometry/
│   │   ├── circle.rs        ← модуль `geometry::circle`
│   │   └── rectangle.rs     ← модуль `geometry::rectangle`
│   └── percent.rs           ← модуль `percent`
└── tests/
    └── geometry_api.rs      ← интеграционные тесты
```

`cargo new` создал только `main.rs`. Библиотека появляется, как только вы добавляете `src/lib.rs`: Cargo видит файл и собирает библиотечный крейт, названный по имени пакета, где дефисы заменены подчёркиваниями: `p04_modules`.

**`src/lib.rs`** — корень библиотеки:

```rust
pub mod geometry;
pub mod percent;

// реэкспорт: вызывающий код может писать `p04_modules::Circle`
pub use geometry::circle::Circle;
```

**`src/geometry.rs`** — модуль `geometry`, объявляющий два своих дочерних модуля:

```rust
pub mod circle;
pub mod rectangle;

// видна везде внутри этого крейта, но не другим крейтам
pub(crate) fn round2(value: f64) -> f64 {
    (value * 100.0).round() / 100.0
}
```

**`src/geometry/circle.rs`**:

```rust
use std::f64::consts::PI;

use super::round2;

#[derive(Debug, Clone, Copy, PartialEq)]
pub struct Circle {
    radius: f64,
}

impl Circle {
    /// Returns `None` unless the radius is a positive, finite number.
    pub fn new(radius: f64) -> Option<Circle> {
        if radius > 0.0 && radius.is_finite() {
            Some(Circle { radius })
        } else {
            None
        }
    }

    pub fn radius(&self) -> f64 {
        self.radius
    }

    pub fn area(&self) -> f64 {
        round2(PI * self.radius * self.radius)
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn area_of_unit_circle() {
        let c = Circle::new(1.0).unwrap();
        assert_eq!(c.area(), 3.14);
    }

    #[test]
    fn rejects_bad_radius() {
        assert_eq!(Circle::new(0.0), None);
        assert_eq!(Circle::new(-2.0), None);
        assert_eq!(Circle::new(f64::NAN), None);
    }

    #[test]
    fn unit_tests_may_touch_private_fields() {
        let c = Circle { radius: 2.0 };
        assert_eq!(c.radius, 2.0);
    }
}
```

**`src/geometry/rectangle.rs`**:

```rust
use super::round2;

#[derive(Debug, Clone, Copy, PartialEq)]
pub struct Rectangle {
    pub width: f64,
    pub height: f64,
}

impl Rectangle {
    pub fn area(&self) -> f64 {
        round2(self.width * self.height)
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn area_is_rounded() {
        let r = Rectangle { width: 1.234, height: 2.0 };
        assert_eq!(r.area(), 2.47);
    }
}
```

**`src/percent.rs`**:

```rust
/// A percentage from 0 to 100. The field is private,
/// so no value outside that range can ever exist.
#[derive(Debug, Clone, Copy, PartialEq)]
pub struct Percent(u8);

impl Percent {
    pub fn new(value: u8) -> Option<Percent> {
        if value <= 100 {
            Some(Percent(value))
        } else {
            None
        }
    }

    pub fn value(&self) -> u8 {
        self.0
    }

    pub fn of(&self, amount: u32) -> u32 {
        amount * u32::from(self.0) / 100
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn accepts_0_to_100() {
        assert!(Percent::new(0).is_some());
        assert!(Percent::new(100).is_some());
    }

    #[test]
    fn rejects_101() {
        assert_eq!(Percent::new(101), None);
    }

    #[test]
    fn computes_share() {
        assert_eq!(Percent::new(25).unwrap().of(200), 50);
    }
}
```

**`src/main.rs`** — программа пользуется библиотекой как любым внешним крейтом:

```rust
use p04_modules::geometry::rectangle::Rectangle;
use p04_modules::percent::Percent;
use p04_modules::Circle;

fn main() {
    let Some(circle) = Circle::new(2.0) else {
        println!("bad radius");
        return;
    };
    let rect = Rectangle {
        width: 3.5,
        height: 2.0,
    };
    println!("circle area    = {}", circle.area());
    println!("rectangle area = {}", rect.area());

    if let Some(discount) = Percent::new(15) {
        println!("15% of 200 = {}", discount.of(200));
    }
    println!("{:?}", Percent::new(150));
}
```

**`tests/geometry_api.rs`** — интеграционные тесты:

```rust
use p04_modules::geometry::rectangle::Rectangle;
use p04_modules::{Circle, percent::Percent};

#[test]
fn circle_through_the_public_api() {
    let circle = Circle::new(1.0).expect("valid radius");
    assert_eq!(circle.area(), 3.14);
    assert_eq!(circle.radius(), 1.0);
}

#[test]
fn rectangle_through_the_public_api() {
    let rect = Rectangle { width: 2.0, height: 3.0 };
    assert_eq!(rect.area(), 6.0);
}

#[test]
fn percent_cannot_exceed_100() {
    assert!(Percent::new(100).is_some());
    assert!(Percent::new(101).is_none());
}
```

**Что происходит:**

1. `lib.rs` — корень библиотечного крейта. `pub mod geometry;` говорит компилятору загрузить `src/geometry.rs` и сделать модуль публичным. `pub mod percent;` загружает `src/percent.rs`.
2. `geometry.rs` сам является модулем с детьми: `pub mod circle;` загружает `src/geometry/circle.rs`. Имя папки `geometry/` совпадает с именем модуля.
3. `round2` помечена `pub(crate)`: `circle` и `rectangle` вызывают её как `super::round2` (`super` — родительский модуль `geometry`), но код других крейтов, например `main.rs`, не может.
4. В `circle.rs` поле `radius` приватное, поэтому единственный способ получить `Circle` — `Circle::new`, которая отвергает нулевой, отрицательный, бесконечный радиус и `NaN` и возвращает `None`. `NaN` («не число») — особое значение `f64`, которое появляется при недопустимых результатах вроде `0.0 / 0.0`.
5. `Percent(u8)` — кортежная структура с приватным полем, применение идеи «сделать недопустимые состояния непредставимыми» ([Урок 04.3](03-enums.md)). Её метод `of` превращает значение в долю от суммы.
6. Строки, начинающиеся с `///`, — **документирующие комментарии**. Они описывают следующий за ними элемент и попадают в сгенерированную документацию; их разбирает [Урок 05.7 — Документация](../05-rust-intermediate/07-documentation.md).
7. `pub use geometry::circle::Circle;` в `lib.rs` — реэкспорт: посторонние могут писать `p04_modules::Circle` наравне с длинным путём.
8. `main.rs` — *другой крейт*, зависящий от библиотеки в том же пакете. Он обращается к ней по имени крейта `p04_modules` (дефис превращён в подчёркивание), а не через `crate::`. Он видит только `pub`-элементы.
9. Модульные тесты лежат в `mod tests` внутри каждого файла. Дочерний модуль видит приватные элементы родителя, поэтому последний тест в `circle.rs` может напрямую построить `Circle { radius: 2.0 }`. Тесты в `tests/` компилируются как внешний крейт и могут использовать только публичный API.

Запустите:

```text
cargo run
```

```text
circle area    = 12.57
rectangle area = 7
15% of 200 = 30
None
```

`rectangle area = 7`, а не `7.0`: `println!` с `{}` печатает целый `f64` без `.0`.

И тесты:

```text
cargo test
```

```text
running 7 tests
test geometry::circle::tests::unit_tests_may_touch_private_fields ... ok
test geometry::circle::tests::rejects_bad_radius ... ok
test geometry::circle::tests::area_of_unit_circle ... ok
test percent::tests::accepts_0_to_100 ... ok
test geometry::rectangle::tests::area_is_rounded ... ok
test percent::tests::computes_share ... ok
test percent::tests::rejects_101 ... ok
test result: ok. 7 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s

running 0 tests
test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s

running 3 tests
test circle_through_the_public_api ... ok
test percent_cannot_exceed_100 ... ok
test rectangle_through_the_public_api ... ok
test result: ok. 3 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

(Cargo печатает строку «Running ...» перед каждой группой и последнюю пустую группу для тестов документации; здесь они опущены. Порядок тестов внутри группы на вашей машине может отличаться, потому что тесты выполняются параллельно.) Три группы — это: модульные тесты библиотеки (7), модульные тесты `main.rs` (их нет) и интеграционные тесты (3). Имя теста показывает путь по модулям, например `geometry::circle::tests::rejects_bad_radius` — дерево модулей видно прямо в выводе.

> 📏 **Соглашение:** держите `main.rs` тонким: он читает аргументы, вызывает библиотеку и печатает результаты. Настоящую логику кладите в библиотечный крейт (`lib.rs` и модули). Тогда библиотеку можно тестировать без запуска программы, переиспользовать в других программах и позже разбивать дальше. Отныне такая раскладка будет во всех проектах курса.

> 📏 **Соглашение:** один модуль — одна идея, имена файлов в `snake_case`, совпадающие с именем модуля, и небольшие модули: если файл вырос больше нескольких сотен строк, разделите его.

### Дополнительные бинарники и другие особые папки

| Путь | Что это |
|---|---|
| `src/main.rs` | бинарный крейт по умолчанию (назван по имени пакета) |
| `src/lib.rs` | библиотечный крейт |
| `src/bin/tool.rs` | дополнительный бинарник; запускается через `cargo run --bin tool` |
| `tests/*.rs` | интеграционные тесты; каждый файл — отдельный крейт |
| `examples/*.rs` | небольшие примеры для пользователей вашей библиотеки, запускаются через `cargo run --example name` |
| `benches/*.rs` | бенчмарки (измерение скорости, [Урок 05.12](../05-rust-intermediate/12-benchmarking-basics.md)) |

## 6. Подводные камни с файлами

### Необъявленный файл не компилируется

Cargo **не** просматривает вашу папку в поисках файлов `.rs`. В крейт входят только файлы, достижимые через объявления `mod name;` от корня крейта. Создайте `src/extra.rs` с тестом, который заведомо падает:

```rust
#[test]
fn this_test_fails() {
    assert_eq!(1 + 1, 3);
}
```

и запустите `cargo test`: все тесты проходят, а новый тест даже не упоминается. Файл просто игнорируется, потому что никто не написал `mod extra;`. Если вы добавили файл, а «ничего не происходит», проверьте, что он объявлен.

### Файл объявленного модуля должен существовать

```text
error[E0583]: file not found for module `missing`
 --> src/lib.rs:6:1
  |
6 | pub mod missing;
  | ^^^^^^^^^^^^^^^^
  |
  = help: to create the module `missing`, create file "src/missing.rs" or "src/missing/mod.rs"
  = note: if there is a `mod missing` elsewhere in the crate already, import it with `use crate::...` instead
```

Сообщение называет два места, где может лежать файл.

### Бинарный крейт — это не библиотека

В `main.rs` `crate::` означает *бинарный* крейт, в котором здесь нет модуля `geometry`:

```text
error[E0433]: cannot find `geometry` in the crate root
 --> src/main.rs:2:12
  |
2 | use crate::geometry::rectangle::Rectangle;
  |            ^^^^^^^^ could not find `geometry` in the crate root
```

Используйте имя библиотечного крейта: `use p04_modules::geometry::rectangle::Rectangle;`.

### Забытый `pub` у модуля

```text
error[E0603]: module `geometry` is private
 --> src/main.rs:2:18
  |
2 | use p04_modules::geometry::rectangle::Rectangle;
  |                  ^^^^^^^^             --------- struct `Rectangle` is not publicly re-exported
  |                  |
  |                  private module
  |
note: the module `geometry` is defined here
 --> src/lib.rs:1:1
  |
1 | mod geometry;
  | ^^^^^^^^^^^^
```

В `lib.rs` напишите `pub mod geometry;`.

### Вызов элемента `pub(crate)` из другого крейта

```text
error[E0603]: function `round2` is private
 --> src/main.rs:6:43
  |
6 |     println!("{}", p04_modules::geometry::round2(1.234));
  |                                           ^^^^^^ private function
```

`pub(crate)` заканчивается на границе крейта. Сообщение просто говорит «private», потому что для такого вызывающего кода так оно и есть.

### Опечатки и отсутствующие импорты

```text
error[E0433]: cannot find `geometri` in `p04_modules`
 --> src/main.rs:2:18
  |
2 | use p04_modules::geometri::rectangle::Rectangle;
  |                  ^^^^^^^^ could not find `geometri` in `p04_modules`
```

```text
error[E0433]: cannot find module or crate `geometry` in this scope
 --> src/main.rs:9:16
  |
9 |     let rect = geometry::rectangle::Rectangle {
  |                ^^^^^^^^ use of unresolved module or unlinked crate `geometry`
  |
help: to make use of source file src/geometry.rs, use `mod geometry` in this file to declare the module
```

Вторая ошибка — отсутствующий `use` (подсказка компилятора про `mod geometry` в этом случае неверна, потому что модуль принадлежит библиотечному крейту, а не `main.rs`). Сначала читайте первые строки: *что* не найдено и *где*.

## 7. Чужие крейты и большие проекты

Всё сказанное выше работает так же для крейтов, написанных другими. Вы перечисляете их в разделе `[dependencies]` файла `Cargo.toml` (этот файл написан на **TOML** — простом формате из строк `key = value` и `[секций]`), Cargo скачивает их с crates.io (публичного реестра крейтов), а вы подключаете их через `use` по имени:

```text
[dependencies]
rand = "0.10"
```

[Урок 05.8 — Cargo: зависимости и версии](../05-rust-intermediate/08-cargo-dependencies-and-versions.md) полностью рассказывает об этом, включая версии и фичи. Когда один проект вырастает в несколько связанных пакетов, Cargo может собирать их вместе как **рабочее пространство** (workspace); это тоже в Уроке 05.8.

> 🔒 **Безопасность:** каждая зависимость — это код, который вы запускаете на своём компьютере и поставляете своим пользователям. Добавляйте их осознанно; [Урок 05.8](../05-rust-intermediate/08-cargo-dependencies-and-versions.md) показывает, как их проверять.

> ⚡ **Производительность:** модули и файлы не меняют скорость работы вашей программы: компилятор видит весь крейт как одну единицу. Они меняют **время сборки** и **читаемость**. Разделение большой программы на несколько крейтов позволяет Cargo пересобирать только изменившиеся крейты.

## 8. Тестирование на нескольких модулях

> 🧪 **Тестирование:**
> - **Модульные тесты** лежат в `#[cfg(test)] mod tests` внизу тестируемого файла. Они могут использовать приватные элементы этого файла (через `use super::*;`), поэтому вы можете тестировать небольшие вспомогательные функции.
> - **Интеграционные тесты** лежат в `tests/`. Они пользуются библиотекой так, как это делал бы посторонний (только `pub`-элементы), поэтому проверяют, что ваш публичный API полон и пригоден для использования, включая реэкспорты.
> - Тест не запустится, если его файл не объявлен через `mod`. Если новый тест никогда не появляется в выводе, проверьте строку `mod`.
> - Запустить только часть тестов можно с фильтром по имени: `cargo test percent` запускает тесты, в пути которых есть `percent`.

## Частые ошибки

> ⚠️ **Частая ошибка:** создать `src/something.rs` и забыть `mod something;` — файл молча игнорируется (раздел 6).

> ⚠️ **Частая ошибка:** ставить `pub` на всё, «чтобы ошибки ушли». Каждый `pub` расширяет ваш публичный API. Начинайте с приватного и открывайте только то, что нужно вызывающему коду.

> ⚠️ **Частая ошибка:** использовать `crate::` в `main.rs`, чтобы добраться до модулей из `lib.rs` (`E0433`, раздел 6). Бинарник и библиотека — разные крейты.

> ⚠️ **Частая ошибка:** `pub struct` с приватными полями, а затем попытка построить её литералом структуры из другого модуля (`E0451`). Предоставьте функцию `new`.

> ⚠️ **Частая ошибка:** путать имя пакета (`p04-modules`, с дефисом) и имя крейта в коде (`p04_modules`, с подчёркиванием).

> ⚠️ **Частая ошибка:** забыть, что `mod tests` нужен `use super::*;`, чтобы видеть код выше.

> ⚠️ **Частая ошибка:** закоммитить папку `target/` в Git. Проверьте, что в `.gitignore` есть `target/`. В пакете с бинарником `Cargo.lock` **нужно** коммитить (он фиксирует точные версии зависимостей); для библиотечного крейта, который вы публикуете, практика различается, но хранить его не вредно.

## Попробуйте сами

1. Постройте пакет `p04-modules` из раздела 5: создайте папки, наберите каждый файл, затем запустите `cargo fmt`, `cargo run`, `cargo test`, `cargo clippy`. Всё должно пройти без предупреждений.
2. Намеренно сломайте его, по одному изменению за раз, прочитайте ошибку и отмените изменение:
   - уберите `pub` из `pub mod geometry;` в `lib.rs`;
   - добавьте `pub mod missing;` в `lib.rs`;
   - в `main.rs` напишите `Circle { radius: 2.0 }`;
   - создайте `src/extra.rs` с падающим тестом и убедитесь, что `cargo test` его игнорирует; затем добавьте `mod extra;` в `lib.rs` и посмотрите, как он падает.
3. Добавьте новый модуль `temperature` в собственном файле с публичной функцией `c_to_f(c: f64) -> f64` и модульным тестом. Объявите его в `lib.rs`, реэкспортируйте функцию через `pub use temperature::c_to_f;`, вызовите её из `main.rs` и добавьте интеграционный тест.
4. Делайте коммиты по шагам, как в [Уроке 01.9](../01-git-and-version-control/09-teamwork-pull-requests-and-conventions.md):

   ```text
   cd ~/projects/rust
   git status
   git add code/p04-modules
   git commit -m "feat(p04): geometry and percent modules with tests"
   ```

   Проверьте через `git status`, что папка `target/` не добавлена в индекс. Если `git status` перечисляет `target/`, сначала добавьте `target/` в файл `.gitignore`.

## Упражнения

### Упражнение 1 — Модуль с приватной вспомогательной функцией (легко)

В одном файле создайте `mod calc` с публичными функциями `square(x: i32)` и `cube(x: i32)`, которые обе используют **приватную** вспомогательную функцию `multiply(a, b)`. Вызовите публичные функции из `main`. Напишите внутри `calc` модуль с модульными тестами; покажите, что тест может вызвать приватную вспомогательную функцию.

<details>
<summary>Подсказка</summary>

Только `square` и `cube` получают `pub`. Внутри `calc` добавьте `#[cfg(test)] mod tests { use super::*; ... }`; поскольку `tests` — потомок `calc`, он видит `multiply`.

</details>

<details>
<summary>Ответ</summary>

```rust
mod calc {
    pub fn square(x: i32) -> i32 {
        multiply(x, x)
    }

    pub fn cube(x: i32) -> i32 {
        multiply(multiply(x, x), x)
    }

    // приватная вспомогательная функция: вызвать её может только код внутри `calc`
    fn multiply(a: i32, b: i32) -> i32 {
        a * b
    }

    #[cfg(test)]
    mod tests {
        use super::*;

        #[test]
        fn square_and_cube() {
            assert_eq!(square(4), 16);
            assert_eq!(cube(-3), -27);
        }

        #[test]
        fn tests_can_call_private_helpers() {
            assert_eq!(multiply(6, 7), 42);
        }
    }
}

fn main() {
    println!("{}", calc::square(5));
    println!("{}", calc::cube(2));
}
```

Вывод `cargo run`:

```text
25
8
```

`cargo test` сообщает о 2 пройденных тестах. Почему это работает: дочерний модуль (`tests`) может использовать приватные элементы родителя (`calc`), а `main` не может вызвать `calc::multiply`.

</details>

### Упражнение 2 — Тип, который не может быть неверным (средне)

Внутри `mod weather` объявите `pub struct Temperature { celsius: f64 }` (приватное поле) с `pub fn new(celsius: f64) -> Option<Temperature>`, которая отвергает значения ниже абсолютного нуля (−273.15), а также методы `celsius()` и `fahrenheit()`. Напишите тесты для обычного значения, граничного значения −273.15 и −300. В `main` покажите, что `weather::Temperature { celsius: -500.0 }` не компилируется.

<details>
<summary>Подсказка</summary>

У поля нет `pub`, поэтому внешний код обязан идти через `new`. Проверка — `celsius >= -273.15`. Вспомните формулу Фаренгейта из Урока 04.1: `c * 9.0 / 5.0 + 32.0`.

</details>

<details>
<summary>Ответ</summary>

```rust
mod weather {
    #[derive(Debug, Clone, Copy, PartialEq)]
    pub struct Temperature {
        celsius: f64,
    }

    impl Temperature {
        pub fn new(celsius: f64) -> Option<Temperature> {
            if celsius >= -273.15 {
                Some(Temperature { celsius })
            } else {
                None
            }
        }

        pub fn celsius(&self) -> f64 {
            self.celsius
        }

        pub fn fahrenheit(&self) -> f64 {
            self.celsius * 9.0 / 5.0 + 32.0
        }
    }

    #[cfg(test)]
    mod tests {
        use super::*;

        #[test]
        fn normal_values_are_accepted() {
            let t = Temperature::new(100.0).unwrap();
            assert_eq!(t.fahrenheit(), 212.0);
        }

        #[test]
        fn below_absolute_zero_is_rejected() {
            assert_eq!(Temperature::new(-300.0), None);
            assert!(Temperature::new(-273.15).is_some());
        }
    }
}

use weather::Temperature;

fn main() {
    match Temperature::new(36.6) {
        Some(t) => println!("{} C = {:.1} F", t.celsius(), t.fahrenheit()),
        None => println!("impossible temperature"),
    }
    println!("{:?}", Temperature::new(-500.0));
}
```

Вывод `cargo run`:

```text
36.6 C = 97.9 F
None
```

Попытка `weather::Temperature { celsius: -500.0 }` в `main` не компилируется с `error[E0451]: field `celsius` of struct `Temperature` is private`.

Почему это работает: поскольку поле приватное, каждая `Temperature` в программе создана через `new`, а значит, каждая физически возможна. Правило обеспечивает компилятор, а не дисциплина.

</details>

### Упражнение 3 — Перенесите модуль в отдельный файл (средне)

Начните с `cargo new code/p04-greeter`. В `src/main.rs` модуль `greeter` с `pub fn hello(name: &str) -> String` (возвращает `"Hello, NAME!"`) записан прямо в файле. Перенесите его в `src/greeter.rs`, чтобы в `main.rs` остались только `mod greeter;` и `main`. Оставьте модульный тест рядом с функцией. Что печатает `cargo run` для `hello("world")`?

<details>
<summary>Подсказка</summary>

Файл `src/greeter.rs` содержит *содержимое* модуля без обёртки `mod greeter { }`. В `main.rs` замените весь блок на `mod greeter;`.

</details>

<details>
<summary>Ответ</summary>

`src/greeter.rs`:

```rust
pub fn hello(name: &str) -> String {
    format!("Hello, {name}!")
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn greets_by_name() {
        assert_eq!(hello("Ferris"), "Hello, Ferris!");
    }
}
```

`src/main.rs`:

```rust
mod greeter;

fn main() {
    println!("{}", greeter::hello("world"));
}
```

Вывод `cargo run`:

```text
Hello, world!
```

`cargo test` сообщает об 1 пройденном тесте.

Почему это работает: `mod greeter;` говорит компилятору загрузить `src/greeter.rs` и считать его содержимое телом модуля `greeter`. Функция должна остаться `pub`, потому что `main` находится вне модуля.

</details>

### Упражнение 4 — Библиотека и программа в одном пакете (сложно)

Создайте пакет `code/p04-text-tools` (`cargo new`). Напишите библиотеку с двумя модулями в двух файлах: `counting` с `pub fn word_count(text: &str) -> usize` и `shouting` с `pub fn shout(text: &str) -> String` (текст в верхнем регистре плюс `!`). Реэкспортируйте обе функции в корне крейта через `pub use`. Программа в `main.rs` должна вызывать их по корневым путям. Добавьте модульные тесты в каждом файле модуля и один интеграционный тест в `tests/api.rs`. Всё должно проходить `cargo test`, `cargo clippy` и `cargo fmt`.

(В ответе пакет назван `ex4`; с вашим именем пакета `p04-text-tools` имя крейта в строках `use` станет `p04_text_tools`.)

<details>
<summary>Подсказка</summary>

`src/lib.rs`: `pub mod counting; pub mod shouting; pub use counting::word_count; pub use shouting::shout;`. В `main.rs` и в `tests/api.rs` начните с `use p04_text_tools::{shout, word_count};`. `text.split_whitespace().count()` считает слова; `text.to_uppercase()` переводит в верхний регистр.

</details>

<details>
<summary>Ответ</summary>

`src/lib.rs`:

```rust
pub mod counting;
pub mod shouting;

pub use counting::word_count;
pub use shouting::shout;
```

`src/counting.rs`:

```rust
/// Counts the words separated by whitespace.
pub fn word_count(text: &str) -> usize {
    text.split_whitespace().count()
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn counts_words() {
        assert_eq!(word_count("one two  three"), 3);
    }

    #[test]
    fn empty_text_has_no_words() {
        assert_eq!(word_count(""), 0);
        assert_eq!(word_count("   "), 0);
    }
}
```

`src/shouting.rs`:

```rust
/// Upper-cases the text and adds an exclamation mark.
pub fn shout(text: &str) -> String {
    format!("{}!", text.to_uppercase())
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn shouts() {
        assert_eq!(shout("hi there"), "HI THERE!");
    }
}
```

`src/main.rs` (для пакета с именем `ex4`; используйте имя своего крейта):

```rust
use ex4::{shout, word_count};

fn main() {
    let text = "modules keep code tidy";
    println!("{} words", word_count(text));
    println!("{}", shout(text));
}
```

`tests/api.rs`:

```rust
use ex4::{shout, word_count};

#[test]
fn both_functions_are_reachable_from_the_crate_root() {
    assert_eq!(word_count("a b c"), 3);
    assert_eq!(shout("ok"), "OK!");
}
```

Вывод `cargo run`:

```text
4 words
MODULES KEEP CODE TIDY!
```

`cargo test` сообщает о 3 модульных тестах (2 + 1) и 1 интеграционном, все пройдены, а `cargo clippy` ничего не печатает.

Почему это работает: библиотечный крейт создаётся, как только появляется `src/lib.rs`. Две строки `pub mod` загружают два файла, а две строки `pub use` позволяют вызывающему коду писать `crate_name::word_count` вместо `crate_name::counting::word_count`. Тест в `tests/` компилируется как внешний пользователь и доказывает, что реэкспорты действительно работают.

</details>

## Вопросы для самопроверки

1. Чем различаются пакет, крейт и модуль?

<details>
<summary>Ответ</summary>

Пакет — папка с `Cargo.toml`, содержащая один или несколько крейтов. Крейт — одна единица компиляции, бинарник или библиотека, начинающаяся с корневого файла. Модуль — пространство имён и граница приватности внутри крейта.

</details>

2. Что означают `crate::`, `self::` и `super::` в пути?

<details>
<summary>Ответ</summary>

`crate::` начинается от корня текущего крейта. `self::` начинается от текущего модуля. `super::` начинается от родительского модуля.

</details>

3. Кто может пользоваться приватным элементом?

<details>
<summary>Ответ</summary>

Код того же модуля и модулей, вложенных в него. Родительские модули и посторонние — не могут.

</details>

4. Если `pub mod a { fn b() {} }`, может ли внешний код вызвать `a::b()`? Почему?

<details>
<summary>Ответ</summary>

Нет. Модуль `a` публичный, но функция `b` приватна (нет `pub`), поэтому путь блокируется на последнем шаге (`E0603`).

</details>

5. Зачем структуре с приватным полем нужна функция `new`?

<details>
<summary>Ответ</summary>

Внешний код не может написать литерал структуры, потому что не может назвать приватное поле (`E0451`). `new` — контролируемый вход, где вы можете проверить значения, так что неверного экземпляра существовать не может.

</details>

6. Что делает `mod geometry;` и где компилятор ищет код?

<details>
<summary>Ответ</summary>

Объявляет модуль `geometry`, тело которого находится в другом файле. Компилятор загружает `geometry.rs` рядом с текущим файлом или `geometry/mod.rs`. Файл вложенного модуля лежит в папке `geometry/`.

</details>

7. Почему `main.rs` не может написать `crate::geometry::...` для модуля, объявленного в `lib.rs`?

<details>
<summary>Ответ</summary>

`main.rs` и `lib.rs` — два отдельных крейта одного пакета. В `main.rs` `crate` означает бинарный крейт. Библиотекой пользуются по имени её крейта, например `use p04_modules::geometry::...`.

</details>

8. Где живут модульные и интеграционные тесты и что может видеть каждый из них?

<details>
<summary>Ответ</summary>

Модульные тесты живут в `#[cfg(test)] mod tests` внутри исходного файла и могут использовать приватные элементы этого модуля. Интеграционные тесты живут в `tests/*.rs`, это отдельные крейты, и они могут использовать только публичный API.

</details>

## Итоги

- **Пакет** (один `Cargo.toml`) содержит крейты: не более одной библиотеки и сколько угодно бинарников. **Крейт** компилируется как одна единица. **Модули** организуют код внутри крейта в дерево.
- Пути используют `::`. Они начинаются с `crate::` (абсолютные), с имени модуля или `self::` (относительные) либо с `super::` (родитель).
- Всё приватно по умолчанию; `pub` открывает элемент; `pub(crate)` открывает его всему крейту. Поля структур приватны, если не помечены `pub`; варианты публичного перечисления публичны.
- Приватные поля плюс проверяющий `new` дают **инкапсуляцию** — тип, который не может хранить неверные данные.
- `use` сокращает пути; `as` переименовывает, `{a, b}` группирует, `pub use` реэкспортирует более короткий публичный путь. Идиома: для функций импортируйте родительский модуль, для типов — сам элемент.
- `mod name;` загружает `name.rs` (или `name/mod.rs`); вложенные модули живут в папке `name/`. Файл, который никогда не объявлен через `mod`, никогда не компилируется.
- Логику кладите в `lib.rs` и модули; `main.rs` держите тонким. Бинарник пользуется библиотекой по имени крейта (дефисы становятся подчёркиваниями).
- Модульные тесты лежат в каждом файле; интеграционные лежат в `tests/` и видят только публичный API.

## Источники и что почитать

- [The Rust Programming Language — Ch. 7 Managing Growing Projects with Packages, Crates, and Modules](https://doc.rust-lang.org/book/ch07-00-managing-growing-projects-with-packages-crates-and-modules.html) — глава официальной книги по этой теме (её подглавы 7.1–7.5 разбирают каждую часть).
- [The Rust Programming Language — 7.2 Defining Modules to Control Scope and Privacy](https://doc.rust-lang.org/book/ch07-02-defining-modules-to-control-scope-and-privacy.html) — модули и `pub`.
- [The Rust Programming Language — 7.5 Separating Modules into Different Files](https://doc.rust-lang.org/book/ch07-05-separating-modules-into-different-files.html) — правила для файлов из раздела 5.
- [The Cargo Book — Package Layout](https://doc.rust-lang.org/cargo/guide/project-layout.html) — стандартные папки (`src/bin`, `tests`, `examples`, `benches`).
- [The Rust Reference — Visibility and Privacy](https://doc.rust-lang.org/reference/visibility-and-privacy.html) — точные правила приватности, включая `pub(crate)` и `pub(in path)`.
- [Rust by Example — Modules](https://doc.rust-lang.org/rust-by-example/mod.html) — запускаемые примеры модулей, видимости и `use`.

---

[🏠 Начало](../START-HERE.md) · [← 04.5 if let, let else и образцы](05-if-let-let-else-and-patterns.md) · [📋 Этап 04](README.md) · [04.7 Векторы →](07-vectors.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
