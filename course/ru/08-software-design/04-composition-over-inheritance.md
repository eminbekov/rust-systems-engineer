<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 08.3 Принципы SOLID](03-solid-principles.md) · [📋 Этап 08](README.md) · [08.5 Проектирование через типы: недопустимые состояния невозможны →](05-type-driven-design.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/08-software-design/04-composition-over-inheritance.md) <!-- nav -->

# Урок 08.4 — Композиция вместо наследования

> **Этап 08 — Проектирование ПО: принципы, паттерны и соглашения** · Урок 4 из 17 · ⏱ около 120 минут

## Что вы узнаете

- Что такое наследование и почему в Rust его нет для структур.
- Как строить большие типы из маленьких (**композиция**) и делиться поведением через трейты.
- Как расширить существующий тип, **обернув** его, например писатель, который считает байты.
- Почему использование `Deref` для имитации наследования — антипаттерн.
- Как выбирать между полем структуры, трейтом, обёрткой и перечислением.

## Перед началом

- Вы прошли [Урок 08.3 — Принципы SOLID](03-solid-principles.md).
- Вы знаете структуры и методы ([Урок 04.1](../04-structs-enums-collections-errors/01-structs.md) и [Урок 04.2](../04-structs-enums-collections-errors/02-methods-and-associated-functions.md)), трейты с методами по умолчанию ([Урок 05.2](../05-rust-intermediate/02-traits.md)) и трейт `Deref` ([Урок 05.9](../05-rust-intermediate/09-smart-pointers.md)).
- Вы помните паттерн newtype и супертрейты из [Урока 06.2](../06-rust-advanced/02-advanced-traits-and-types.md).

## Новые слова

| Слово | Значение |
|---|---|
| **наследование** (inheritance) | Возможность некоторых языков программирования: новый тип объявляется «потомком» существующего «родителя» и автоматически получает его поля и методы, которые может заменить. В Rust нет наследования полей или тел методов между структурами. |
| **является / имеет** (is-a / has-a) | Два вида отношений. «Собака *является* животным» — образ мысли наследования. «У игрока *есть* позиция» — композиция. |
| **композиция** (composition) | Построение большего типа путём помещения в него меньших типов в качестве полей. |
| **компонент** (component) | Маленький тип, который выполняет одну задачу и используется как поле больших типов (`Position`, `Health`). |
| **делегирование** (delegation, переадресация) | Метод внешнего типа, который выполняет свою работу, вызывая ту же задачу у внутреннего значения. |
| **метод по умолчанию** (default method) | Метод трейта, у которого есть тело. Реализующие типы получают его бесплатно и могут заменить. |
| **обёртка** (wrapper, декоратор) | Тип, который хранит значение другого типа, предлагает тот же интерфейс и добавляет вокруг него поведение. |
| **антипаттерн** (anti-pattern) | Распространённое решение, которое выглядит нормально, но приносит больше проблем, чем решает. Урок 08.10 собирает много таких. |

## 1. Что такое наследование и почему советы по проектированию его избегают

В некоторых языках можно написать «тип `Dog` расширяет тип `Animal`». Тогда `Dog` сам получает все поля и методы `Animal` и может заменить некоторые из них. Это моделирует отношение **является** (is-a): собака *является* животным.

Это выглядит отличным способом повторного использования кода, но опыт больших программ показал некоторые проблемы:

- Потомок зависит от *внутренностей* родителя. Изменение в родителе может сломать каждого потомка (очень жёсткое сопряжение, Урок 08.2).
- Иерархии негибки. Кто такая собака-робот? Это `Animal` или `Machine`? Родословное дерево может быть только одно.
- Утверждение «является» на практике часто неверно. Квадрат *является* прямоугольником в геометрии, но код, который меняет ширину прямоугольника независимо от высоты, ломается, когда получает квадрат.

Известная книга по паттернам проектирования (1994) дала совет, ставший лозунгом: **предпочитайте композицию наследованию**. Rust следует ему по замыслу: структура не может наследовать от другой структуры. Вместо этого Rust даёт четыре инструмента: поля (композиция), трейты (общее поведение), обёртки (расширение) и перечисления (закрытый набор вариантов). Рассмотрим каждый.

## 2. Композиция: «имеет»

Композиция означает, что больший тип *имеет* внутри меньшие типы. Каждый маленький тип, **компонент**, делает одну работу. Здесь игровой объект получает свои способности от компонентов:

```rust
struct Position {
    x: i32,
    y: i32,
}

impl Position {
    fn move_by(&mut self, dx: i32, dy: i32) {
        self.x += dx;
        self.y += dy;
    }
}

struct Health {
    points: u32,
}

impl Health {
    fn damage(&mut self, amount: u32) {
        self.points = self.points.saturating_sub(amount);
    }
    fn is_alive(&self) -> bool {
        self.points > 0
    }
}

struct Player {
    name: String,
    position: Position,
    health: Health,
}

struct Wall {
    position: Position, // a wall has a position but no health
}

fn main() {
    let mut player = Player {
        name: "Ana".to_string(),
        position: Position { x: 0, y: 0 },
        health: Health { points: 30 },
    };
    let mut wall = Wall {
        position: Position { x: 5, y: 5 },
    };

    player.position.move_by(2, 3);
    wall.position.move_by(-1, 0);
    player.health.damage(40);

    println!(
        "{} is at ({}, {})",
        player.name, player.position.x, player.position.y
    );
    println!("wall is at ({}, {})", wall.position.x, wall.position.y);
    println!("alive: {}", player.health.is_alive());
}
```

**Что происходит:**

1. `Position` умеет двигать себя через `move_by`. `Health` умеет получать урон, а `saturating_sub` останавливается на нуле, а не уходит ниже.
2. У `Player` *есть* `Position` и *есть* `Health`. У `Wall` только `Position`, потому что у стены нет здоровья.
3. `player.position.move_by(2, 3)` вызывает работу у внутреннего значения. Позиция игрока (0, 0) + (2, 3).
4. Стена начинает с (5, 5) и сдвигается на (-1, 0). Игрок получает 40 урона, больше, чем его 30 очков здоровья, поэтому `health.points` становится 0.

```text
Ana is at (2, 3)
wall is at (4, 5)
alive: false
```

Что мы получили?

- **Комбинируйте как хотите.** У `Wall` может быть позиция без здоровья. С родословным деревом вам понадобился бы промежуточный класс, у которого есть позиция, но нет здоровья.
- **Маленькие, тестируемые части.** `Health` и `Position` связны (Урок 08.2) и могут тестироваться по отдельности.
- **Нет скрытого сопряжения.** `Player` использует только публичные методы `Health`. Если `Health` изменит способ хранения очков, `Player` этого не заметит.

> 📏 **Соглашение:** когда хочется унаследоваться «для повторного использования кода», спросите: *что* я хочу переиспользовать? Данные и поведение на этих данных? Сделайте это компонентом и храните в поле.

## 3. Общее поведение через трейты

Трейт описывает не то, чем вещь *является*, а то, что она *умеет*. В трейте могут быть и **методы по умолчанию**, которые дают поведение каждому реализующему типу — немного похоже на то, как родитель даёт методы потомкам, но без общих данных:

```rust
trait Named {
    fn name(&self) -> &str;

    // A default method: shared behaviour that every implementer gets for free.
    fn greet(&self) -> String {
        format!("Hello, I am {}", self.name())
    }
}

struct Dog {
    name: String,
}

struct Robot {
    id: u32,
    label: String,
}

impl Named for Dog {
    fn name(&self) -> &str {
        &self.name
    }
}

impl Named for Robot {
    fn name(&self) -> &str {
        &self.label
    }

    // This type replaces the default.
    fn greet(&self) -> String {
        format!("BEEP. Unit {} ({})", self.id, self.label)
    }
}

fn main() {
    let dog = Dog {
        name: "Rex".to_string(),
    };
    let robot = Robot {
        id: 7,
        label: "R7".to_string(),
    };
    println!("{}", dog.greet());
    println!("{}", robot.greet());
}
```

**Что происходит:**

1. `Named` требует одно: `name`. У `greet` есть тело, использующее `name`, поэтому это метод по умолчанию.
2. `Dog` реализует только `name` и бесплатно получает `greet`.
3. `Robot` реализует `name` и ещё заменяет `greet` собственным текстом.
4. `main` вызывает `greet` у обоих. Каждый использует свою версию.

```text
Hello, I am Rex
BEEP. Unit 7 (R7)
```

Трейт несёт *поведение* (то, что общее); каждая структура хранит свои *данные*. Функция, принимающая `&impl Named`, принимает обоих. Если трейту нужны методы другого трейта, используйте супертрейт (Урок 06.2): `trait Pet: Named { ... }`.

## 4. Расширяем тип, оборачивая его

Наследование часто используют, чтобы «взять класс и добавить одну вещь». В Rust вы оборачиваете значение в новую структуру, предлагаете тот же трейт и делегируете настоящую работу внутреннему значению. Это **обёртка** (её также называют декоратором). Вот писатель, который считает, сколько байтов через него прошло:

```rust
use std::io::{self, Write};

struct CountingWriter<W: Write> {
    inner: W,
    bytes: usize,
}

impl<W: Write> CountingWriter<W> {
    fn new(inner: W) -> Self {
        CountingWriter { inner, bytes: 0 }
    }
}

impl<W: Write> Write for CountingWriter<W> {
    fn write(&mut self, buf: &[u8]) -> io::Result<usize> {
        let written = self.inner.write(buf)?; // delegate to the inner writer
        self.bytes += written;
        Ok(written)
    }

    fn flush(&mut self) -> io::Result<()> {
        self.inner.flush()
    }
}

fn main() -> io::Result<()> {
    let mut out = CountingWriter::new(io::stdout());
    writeln!(out, "hello")?;
    writeln!(out, "composition")?;
    println!("bytes written: {}", out.bytes);
    Ok(())
}
```

**Что происходит:**

1. `CountingWriter<W: Write>` хранит любого писателя `inner` (трейт `Write` объяснялся в Уроке 08.3) и счётчик.
2. `impl Write for CountingWriter<W>` делает саму обёртку писателем, так что её можно использовать везде, где ожидается писатель.
3. В `write` настоящая запись **делегируется** через `self.inner.write(buf)?`; обёртка добавляет только подсчёт. `flush` делегируется как есть.
4. `main` оборачивает стандартный вывод. Две строки дают 6 + 12 байтов (каждая включает перевод строки `\n`).

```text
hello
composition
bytes written: 18
```

Стандартная библиотека полна обёрток: `BufWriter` и `BufReader` добавляют буферизацию любому писателю или читателю, а адаптеры итераторов вроде `map` и `filter` оборачивают другой итератор. Их можно складывать друг на друга: считающий писатель вокруг буферизованного вокруг файла. Обёртка не знает, что именно она оборачивает, знает только трейт, и это держит сопряжение низким.

## 5. `Deref` — это не наследование

Новички иногда замечают, что `Deref` позволяет `Dog` использовать методы внутреннего `Animal` (Урок 05.9), и применяют его для имитации наследования. Кажется, что работает:

```rust
use std::ops::Deref;

struct Animal {
    name: String,
}

impl Animal {
    fn name(&self) -> &str {
        &self.name
    }
}

trait Speak {
    fn speak(&self) -> String;
}

impl Speak for Animal {
    fn speak(&self) -> String {
        format!("{} makes a sound", self.name)
    }
}

struct Dog(Animal);

impl Deref for Dog {
    type Target = Animal;
    fn deref(&self) -> &Animal {
        &self.0
    }
}

fn announce(speaker: &impl Speak) {
    println!("{}", speaker.speak());
}

fn main() {
    let dog = Dog(Animal {
        name: "Rex".to_string(),
    });
    println!("{}", dog.name()); // works through Deref
    announce(&dog); // does not work: Dog is not an Animal
}
```

```text
error[E0277]: the trait bound `Dog: Speak` is not satisfied
  --> src/main.rs:41:14
   |
41 |     announce(&dog); // does not work: Dog is not an Animal
   |     -------- ^^^^ unsatisfied trait bound
   |     |
   |     required by a bound introduced by this call
   |
help: the trait `Speak` is not implemented for `Dog`
   ...

For more information about this error, try `rustc --explain E0277`.
error: could not compile `demo` (bin "demo") due to 1 previous error
```

Вызов `dog.name()` скомпилировался, потому что компилятор идёт по `Deref`, чтобы найти метод. Но `Dog` не является `Animal`: трейт `Speak`, реализованный для `Animal`, не реализован для `Dog`, и обобщённый код, требующий `Speak`, его отвергает.

Это **антипаттерн полиморфизма через Deref**. `Deref` предназначен для *умных указателей* (`Box`, `Rc`, `String` в `str`) — типов, вся цель которых вести себя как то, что внутри. Использование его для «является» даёт неожиданный поиск методов и скрывает, какой тип на самом деле имеет метод. Честное исправление — явное делегирование: реализуйте трейт для `Dog` и переадресуйте, `impl Speak for Dog { fn speak(&self) -> String { self.0.speak() } }`, и напишите обычный метод `fn name(&self) -> &str { self.0.name() }`, если он нужен.

## 6. Какой инструмент? И когда не нужно композировать

| Вам нужно... | Используйте | Пример |
|---|---|---|
| Переиспользовать данные и операции над ними | Компонент в **поле** | `Player { health: Health }` |
| Сказать «эти типы умеют X» и поделиться кодом для X | **Трейт**, возможно, с методами по умолчанию | `Named::greet` |
| Добавить поведение вокруг существующего типа | **Обёртку**, которая делегирует | `CountingWriter<W>` |
| Смоделировать «ровно один из этих случаев» | **Перечисление** | `Shape::Circle`, `Shape::Rect` |

**Когда НЕ композировать (или не слишком):**

- Оборачивание типа и ручная переадресация 20 методов — много шаблонного кода. Подумайте, не лучше ли подходит трейт с методами по умолчанию, или достаточно ли метода обёртки, возвращающего `&Inner`.
- Не дробите структуру на десяток «компонентов» из одного поля на всякий случай. Делайте компонент, когда у него есть собственное поведение и больше одного пользователя (снова правило трёх).
- Очень глубокая вложенность (`a.b.c.d.value`) — замаскированное сопряжение (Урок 08.2). Дайте внешнему типу метод для того, что нужно вызывающим.

## Частые ошибки

**Ошибка 1: имитация наследования через `Deref`.** См. раздел 5. Если вы заметили, что написали `impl Deref for MyType`, а цель — не что-то похожее на указатель, остановитесь и используйте делегирование.

**Ошибка 2: родословное дерево в структурах.** `struct Dog { animal: Animal }` внутри `struct Puppy { dog: Dog }` — это наследование с лишним набором текста. Спросите, в чём настоящее повторное использование, и дайте ему собственное имя.

**Ошибка 3: трейт, который лишь оборачивает одну структуру.** Если у трейта один реализующий тип и никто не планирует другого, достаточно самой структуры (YAGNI, You Aren't Gonna Need It — «тебе это не понадобится», Урок 08.1).

## Попробуйте сами

1. Создайте пакет:

```bash
cd ~/projects/rust
cargo new code/p08-composition
cd code/p08-composition
```

2. Скопируйте компоненты и `main` из раздела 2 в `src/main.rs` и запустите `cargo run`. Вы увидите три строки из раздела 2.

3. Добавьте новый тип `Enemy` с `Position` и `Health` и `Trap` (ловушку), у которой только `Position`. Копировать методы нельзя: всё перемещение и весь урон должны приходить из компонентов. Напечатайте, где каждый из них находится после перемещения.

4. Добавьте трейт `Describe` с двумя обязательными методами, `label(&self) -> String` и `is_alive(&self) -> bool` (он может спросить компонент `Health`), и методом по умолчанию `describe(&self) -> String`, который возвращает метку, за которой следует `" (alive)"` или `" (gone)"`. Реализуйте его только для `Player` и `Enemy` (у ловушки нет здоровья, поэтому она не может быть «живой»). Заметьте, что наследование вам не понадобилось: хватило трейта и компонентов.

5. Запустите проверки и сохраните:

```bash
cargo fmt
cargo clippy
git add .
git commit -m "feat(p08): components and a describe trait"
```

**Что происходит:** каждый новый объект собирается из частей, которые уже существуют и протестированы. Добавление `Trap` не потребовало изменений в `Player`.

## Упражнения

### Упражнение 1 — Поле, трейт, обёртка или перечисление? (легко)

Какой инструмент подходит лучше всего?

1. Нескольким сервисам нужно писать сообщения в журнал, и у всех один и тот же логгер.
2. `Invoice` и `Customer` оба можно напечатать в отчёте.
3. Заказ всегда находится ровно в одном из состояний: открыт, оплачен или отправлен.

<details>
<summary>Подсказка</summary>

Это «то, что у меня есть», «то, что я умею» или «один из нескольких случаев»?

</details>

<details>
<summary>Ответ</summary>

1. Композиция: каждый сервис хранит (или получает) логгер как поле.
2. Трейт (например, `Report`), который реализуют оба типа.
3. Перечисление с тремя вариантами (Урок 08.8 глубже разбирает состояния).

Почему это работает: поле = «имеет», трейт = «умеет», перечисление = «один из».

</details>

### Упражнение 2 — Выделите компонент и протестируйте его (средне)

`Car` и `Boat` повторяют одну и ту же логику заправки с пределом 50 литров. Выделите компонент `FuelTank` с методом `refuel`, сделайте так, чтобы оба типа хранили его, и **напишите тест** для `refuel` (один раз, когда есть место, один раз, когда бак почти полон).

```rust
struct Car {
    fuel: u32,
}

struct Boat {
    fuel: u32,
}

impl Car {
    fn refuel(&mut self, liters: u32) {
        self.fuel = (self.fuel + liters).min(50);
    }
}

impl Boat {
    fn refuel(&mut self, liters: u32) {
        self.fuel = (self.fuel + liters).min(50);
    }
}

fn main() {
    let mut car = Car { fuel: 45 };
    let mut boat = Boat { fuel: 10 };
    car.refuel(20);
    boat.refuel(20);
    println!("{} {}", car.fuel, boat.fuel);
}
```

<details>
<summary>Подсказка</summary>

Повторяющееся знание — «в бак помещается не больше 50 литров». Поместите число в ассоциированную константу `FuelTank`.

</details>

<details>
<summary>Ответ</summary>

```rust
struct FuelTank {
    liters: u32,
}

impl FuelTank {
    const CAPACITY: u32 = 50;

    fn refuel(&mut self, liters: u32) {
        self.liters = (self.liters + liters).min(Self::CAPACITY);
    }
}

struct Car {
    tank: FuelTank,
}

struct Boat {
    tank: FuelTank,
}

fn main() {
    let mut car = Car {
        tank: FuelTank { liters: 45 },
    };
    let mut boat = Boat {
        tank: FuelTank { liters: 10 },
    };
    car.tank.refuel(20);
    boat.tank.refuel(20);
    println!("{} {}", car.tank.liters, boat.tank.liters);
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn refuel_stops_at_capacity() {
        let mut tank = FuelTank { liters: 45 };
        tank.refuel(20);
        assert_eq!(tank.liters, 50);
    }

    #[test]
    fn refuel_adds_when_there_is_room() {
        let mut tank = FuelTank { liters: 10 };
        tank.refuel(20);
        assert_eq!(tank.liters, 30);
    }
}
```

Почему это работает: предел теперь живёт в одном месте (DRY, Don't Repeat Yourself — «не повторяйся», Урок 08.1). У `Car` и `Boat` *есть* бак, а тесты покрывают общую логику один раз.

</details>

### Упражнение 3 — Напишите обёртку (сложно)

Напишите `UppercaseWriter<W: Write>`, который превращает каждую ASCII-букву (базовую английскую букву) в заглавную перед передачей байтов внутреннему писателю. **Напишите тест**, который оборачивает `Vec<u8>` и проверяет байты.

<details>
<summary>Подсказка</summary>

Реализуйте `write` и `flush` как в `CountingWriter`. В `write` постройте копию в верхнем регистре через `to_ascii_uppercase`, отправьте её через `write_all` и верните `buf.len()` (число принятых вами байтов, а не длину копии). `Vec<u8>` реализует `Write`.

</details>

<details>
<summary>Ответ</summary>

```rust
use std::io::{self, Write};

struct UppercaseWriter<W: Write> {
    inner: W,
}

impl<W: Write> Write for UppercaseWriter<W> {
    fn write(&mut self, buf: &[u8]) -> io::Result<usize> {
        let upper: Vec<u8> = buf.iter().map(|b| b.to_ascii_uppercase()).collect();
        self.inner.write_all(&upper)?;
        Ok(buf.len())
    }

    fn flush(&mut self) -> io::Result<()> {
        self.inner.flush()
    }
}

fn main() -> io::Result<()> {
    let mut out = UppercaseWriter {
        inner: io::stdout(),
    };
    writeln!(out, "quiet text")
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn text_is_uppercased_before_it_reaches_the_inner_writer() {
        let mut out = UppercaseWriter { inner: Vec::new() };
        write!(out, "abc 123").unwrap();
        assert_eq!(out.inner, b"ABC 123");
    }
}
```

Почему это работает: обёртка изменяет данные и делегирует остальное. Тест читает внутреннее поле `out.inner` напрямую, что возможно, потому что модуль тестов находится в том же файле. Возврат `buf.len()` сообщает вызывающему, что все его байты приняты.

</details>

## Вопросы для самопроверки

1. Что означает «имеет» (has-a) в проектировании?

<details>
<summary>Ответ</summary>

Один тип содержит другой как поле и использует его: композиция.

</details>

2. Как трейт Rust делится кодом, не деля данные?

<details>
<summary>Ответ</summary>

Через методы по умолчанию: они пишутся один раз в трейте и используют только обязательные методы трейта.

</details>

3. Почему `announce(&dog)` не компилируется в примере с `Deref`, хотя `dog.name()` работает?

<details>
<summary>Ответ</summary>

Поиск метода идёт через `Deref`, но ограничение трейта требует, чтобы сам тип реализовывал трейт. `Dog` не реализует `Speak`; его реализует только `Animal`.

</details>

4. Назовите две обёртки из стандартной библиотеки.

<details>
<summary>Ответ</summary>

`BufWriter` и `BufReader`; также адаптеры итераторов вроде `Map` и `Filter`.

</details>

## Итоги

- В Rust нет наследования между структурами; вместо него есть композиция, трейты, обёртки и перечисления.
- **Композиция** (has-a): помещайте небольшие связные компоненты в поля. Комбинировать легко, а каждая часть тестируется отдельно.
- **Трейты** делятся поведением: обязательные методы плюс методы по умолчанию; данные остаются в каждой структуре.
- **Обёртка** расширяет существующий тип: храните его, предлагайте тот же трейт, делегируйте и добавляйте своё поведение.
- `Deref` предназначен для умных указателей; использовать его для имитации наследования — антипаттерн. Делегируйте явно.
- Выбирайте по вопросу: имеет (поле), умеет (трейт), вокруг (обёртка), один из (перечисление).
- Не переусердствуйте с композицией: избегайте «компонентов» из одного поля, ручной переадресации десятков методов и длинных цепочек полей.

## Источники и что почитать

- [The Rust Programming Language — 18.1 Characteristics of Object-Oriented Languages](https://doc.rust-lang.org/book/ch18-01-what-is-oo.html) — официальное обсуждение наследования и того, что предлагает Rust взамен.
- [Rust Design Patterns — Deref polymorphism](https://rust-unofficial.github.io/patterns/anti_patterns/deref.html) — почему `Deref` не стоит использовать для имитации наследования.
- [Rust Design Patterns — Newtype](https://rust-unofficial.github.io/patterns/patterns/behavioural/newtype.html) — оборачивание типа, чтобы дать ему новое поведение.
- [std::io::BufWriter](https://doc.rust-lang.org/std/io/struct.BufWriter.html) — настоящая обёртка из стандартной библиотеки.

---

[🏠 Начало](../START-HERE.md) · [← 08.3 Принципы SOLID](03-solid-principles.md) · [📋 Этап 08](README.md) · [08.5 Проектирование через типы: недопустимые состояния невозможны →](05-type-driven-design.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
