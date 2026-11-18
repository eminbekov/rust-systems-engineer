<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 04.2 Методы и ассоциированные функции](02-methods-and-associated-functions.md) · [📋 Этап 04](README.md) · [04.4 Option и match →](04-option-and-match.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/04-structs-enums-collections-errors/03-enums.md) <!-- nav -->

# Урок 04.3 — Перечисления

> **Этап 04 — Структуры, перечисления, коллекции и ошибки** · Урок 3 из 12 · ⏱ около 3,5 часа (2 занятия)

## Что вы узнаете

- Что такое **перечисление** (enum) и чем оно отличается от структуры: «один из нескольких вариантов» вместо «все эти части сразу».
- Как объявлять перечисления, создавать их значения и добавлять к ним методы.
- Как варианты могут нести данные (в виде кортежа и в виде структуры) и почему это так мощно.
- Первое знакомство с `match` — инструментом, который разбирает перечисление, — и почему компилятор заставляет обработать каждый вариант.
- Как перечисление хранится в памяти, как задавать вариантам числа и как безопасно превращать текст в перечисление.

## Перед началом

- Вы прошли [Урок 04.1 — Структуры](01-structs.md) и [Урок 04.2 — Методы и ассоциированные функции](02-methods-and-associated-functions.md).
- Вы знаете `if`/`else` и `bool` ([Урок 02.5 — Ветвления и циклы](../02-programming-basics/05-control-flow.md)). С `match` мы здесь знакомимся впервые по-настоящему; тему завершает [Урок 04.4 — Option и match](04-option-and-match.md).

## Новые слова

| Слово | Значение |
|---|---|
| **перечисление** (enum, enumeration) | Тип, значение которого — ровно **один** вариант из фиксированного списка именованных вариантов. |
| **вариант** (variant) | Один из вариантов перечисления. В `enum Light { Red, Green }` `Red` и `Green` — варианты. |
| **полезная нагрузка** (payload) | Данные, которые несёт вариант, например `f64` в `Circle(f64)`. |
| **вариант-кортеж** (tuple variant) | Вариант, у нагрузки которого безымянные позиции: `Circle(f64)`. |
| **вариант-структура** (struct variant) | Вариант, у нагрузки которого именованные поля: `Rectangle { width: f64, height: f64 }`. |
| **вариант-единица** (unit variant) | Вариант без нагрузки: `Quit`. |
| **match** | Выражение, которое сравнивает значение со списком **образцов** и выполняет код первого подходящего образца. |
| **ветка** (arm) | Одна строка `match`: образец, `=>` и код, который нужно выполнить. |
| **образец** (pattern) | Форма, с которой можно сравнить значение, например `Direction::North` или `Circle(r)`. Полный разбор — в [Уроке 04.5](05-if-let-let-else-and-patterns.md). |
| **исчерпывающий** (exhaustive) | `match` исчерпывающий, когда он охватывает **каждое** возможное значение. Rust это требует. |
| **шаблон-заглушка** (wildcard) | Образец `_`, который подходит к чему угодно и игнорирует это. |
| **дискриминант** (discriminant) | Скрытое число, по которому компилятор отличает варианты; его можно задать вручную (`Low = 1`). |
| **тег** (tag) | Часть значения перечисления в памяти, которая записывает, каким вариантом оно сейчас является. |
| **конечный автомат** (state machine) | Модель чего-то, что всегда находится ровно в одном состоянии из нескольких (не в сети / подключается / в сети) и переходит между состояниями по правилам. |
| **приведение типа** (cast) | Преобразование значения в другой тип через `as`, например `level as u8`. |
| **Option** | Перечисление из стандартной библиотеки для «значение или ничего»: `Some(value)` или `None`. Полностью его объясняет Урок 04.4. |

## 1. Значение — один из нескольких вариантов

Структура говорит «это **и** это **и** это». У `User` есть имя **и** почта **и** число входов.

**Перечисление** говорит «это **или** это **или** это». Светофор — красный **или** зелёный **или** жёлтый — никогда не два сразу и никогда «ни один». Направление по компасу — север **или** восток **или** юг **или** запад.

Без перечислений направление можно было бы хранить числом (`0` — север, `1` — восток …) или текстом (`"north"`). И то и другое может содержать бессмыслицу: `7` или `"nrth"`. Компилятор вам не поможет. С перечислением **существуют только допустимые варианты**, а опечатки не компилируются.

```rust
#[derive(Debug, Clone, Copy, PartialEq)]
enum Direction {
    North,
    East,
    South,
    West,
}

fn main() {
    let heading = Direction::East;
    println!("{:?}", heading);

    if heading == Direction::East {
        println!("going east");
    }

    let all = [Direction::North, Direction::East, Direction::South, Direction::West];
    for d in all {
        println!("{:?}", d);
    }
}
```

**Что происходит:**

1. `enum Direction { ... }` объявляет новый тип с четырьмя **вариантами**. Имена вариантов пишутся в `UpperCamelCase`, как и имена типов.
2. Значение создаётся так: имя перечисления, два двоеточия и вариант — `Direction::East`. Имя перечисления играет роль *пространства имён* (префикса, который разделяет имена), поэтому у другого перечисления тоже может быть вариант `North`.
3. `#[derive(Debug)]` позволяет вывести значение. `PartialEq` включает `==`, так что мы можем сравнивать два направления. `Clone, Copy` делают передачу дешёвой: перечисление без нагрузки — это просто маленькое число (см. раздел 6).
4. Массив хранит четыре направления; цикл `for` выводит каждое.

```text
East
going east
North
East
South
West
```

## 2. Первое знакомство с `match`

Чтобы *использовать* перечисление, обычно задают вопрос «какой это вариант?» и действуют по ответу. Для этого служит **`match`**.

```rust
#[derive(Debug, Clone, Copy, PartialEq)]
enum Direction {
    North,
    East,
    South,
    West,
}

impl Direction {
    fn turn_right(self) -> Direction {
        match self {
            Direction::North => Direction::East,
            Direction::East => Direction::South,
            Direction::South => Direction::West,
            Direction::West => Direction::North,
        }
    }

    fn is_vertical(self) -> bool {
        match self {
            Direction::North | Direction::South => true,
            Direction::East | Direction::West => false,
        }
    }
}

fn main() {
    let mut d = Direction::North;
    for _ in 0..5 {
        println!("{:?} (vertical: {})", d, d.is_vertical());
        d = d.turn_right();
    }
}
```

**Что происходит:**

1. `impl Direction { ... }` даёт перечислению методы точно так же, как структуре ([Урок 04.2](02-methods-and-associated-functions.md)). `self` принимается по значению; это нормально, потому что `Direction` — `Copy`.
2. `match self { ... }` смотрит на значение `self`. У каждой **ветки** есть образец, затем `=>`, затем выражение. Компилятор проверяет ветки сверху вниз и выполняет первую подходящую.
3. В `turn_right` каждая ветка возвращает `Direction`. Весь `match` — выражение, поэтому его значение — это возвращаемое значение функции (после него нет точки с запятой).
4. `Direction::North | Direction::South` значит «любой из этих двух» (`|` читается как «или»). Оба дают `true`.
5. В `main` цикл выполняется пять раз и каждый раз поворачивает направо: после `West` снова идёт `North`.

```text
North (vertical: true)
East (vertical: false)
South (vertical: true)
West (vertical: false)
North (vertical: true)
```

### Компилятор проверяет, что вы охватили всё

Уберите одну ветку и посмотрите, что будет:

```rust
fn name(d: Direction) -> &'static str {
    match d {
        Direction::North => "north",
        Direction::East => "east",
        Direction::South => "south",
    }
}
```

```text
error[E0004]: non-exhaustive patterns: `Direction::West` not covered
  --> src/main.rs:10:11
   |
10 |     match d {
   |           ^ pattern `Direction::West` not covered
   |
note: `Direction` defined here
  --> src/main.rs:2:6
   |
 2 | enum Direction {
   |      ^^^^^^^^^
...
 6 |     West,
   |     ---- not covered
   = note: the matched value is of type `Direction`
help: ensure that all possible cases are being handled by adding a match arm with a wildcard pattern or an explicit pattern as shown
   |
13 ~         Direction::South => "south",
14 ~         Direction::West => todo!(),
   |
```

`match` должен быть **исчерпывающим**: для каждого возможного значения нужна ветка. Компилятор даже называет пропущенный вариант. Это одно из лучших средств безопасности Rust: если позже вы добавите в перечисление пятый вариант, компилятор покажет **каждый `match` в вашей программе**, который нужно обновить. Во многих других языках такая ошибка обнаруживается только тогда, когда программа уже работает.

Когда вам действительно нужно «всё остальное», используйте заглушку `_`:

```text
match self {
    Day::Saturday | Day::Sunday => true,
    _ => false,
}
```

> 📏 **Соглашение:** используйте `_` только тогда, когда «всё остальное» действительно один случай. Если вы перечисляете каждый вариант явно, компилятор предупредит вас, когда кто-то добавит новый; с `_` он промолчит, и новый вариант молча попадёт в ветку по умолчанию.

## 3. Варианты, которые несут данные

До сих пор каждый вариант был только именем. Перечисления Rust умеют гораздо больше: **каждый вариант может нести собственные данные** собственной формы.

```rust
#[derive(Debug)]
enum Shape {
    Circle(f64),
    Square(f64),
    Rectangle { width: f64, height: f64 },
}

impl Shape {
    fn area(&self) -> f64 {
        match self {
            Shape::Circle(radius) => std::f64::consts::PI * radius * radius,
            Shape::Square(side) => side * side,
            Shape::Rectangle { width, height } => width * height,
        }
    }
}

fn main() {
    let shapes = [
        Shape::Circle(1.0),
        Shape::Square(3.0),
        Shape::Rectangle { width: 2.0, height: 4.5 },
    ];
    for shape in &shapes {
        println!("{:?} has area {:.2}", shape, shape.area());
    }
}
```

**Что происходит:**

1. `Circle(f64)` — **вариант-кортеж**: он хранит один безымянный `f64`. Создаётся как вызов функции: `Shape::Circle(1.0)`.
2. `Rectangle { width: f64, height: f64 }` — **вариант-структура**: он хранит именованные поля. Создаётся как структура: `Shape::Rectangle { width: 2.0, height: 4.5 }`.
3. В `area(&self)` мы применяем `match` к *ссылке* на фигуру. В образце `Shape::Circle(radius)` имя `radius` — новая переменная, которая **достаёт нагрузку** из варианта. Поскольку мы сопоставляем `&self`, `radius` — это `&f64`, то есть ссылка внутрь фигуры; арифметика `PI * radius * radius` работает со ссылками на числа.
4. В образце варианта-структуры `Shape::Rectangle { width, height }` имена `width` и `height` подхватывают поля (та же краткая инициализация полей, что вы знаете по структурам).
5. Поскольку данные лежат внутри варианта, добраться до них можно, только проверив, какой это вариант. У `Square` нет `width`, поэтому спросить его по ошибке невозможно.
6. `{:.2}` печатает два знака после запятой.

```text
Circle(1.0) has area 3.14
Square(3.0) has area 9.00
Rectangle { width: 2.0, height: 4.5 } has area 9.00
```

Вариант без данных — **вариант-единица** (как `Quit` ниже). В одном перечислении можно смешивать все три вида:

```rust
#[derive(Debug)]
enum Command {
    Quit,
    Move { x: i32, y: i32 },
    Say(String),
    SetColor(u8, u8, u8),
}

fn run(command: &Command) {
    match command {
        Command::Quit => println!("bye"),
        Command::Move { x, y } => println!("moving by ({x}, {y})"),
        Command::Say(text) => println!("saying: {text}"),
        Command::SetColor(r, g, b) => println!("color is now #{r:02x}{g:02x}{b:02x}"),
    }
}

fn main() {
    let script = vec![
        Command::Say(String::from("hello")),
        Command::Move { x: 3, y: -1 },
        Command::SetColor(255, 128, 0),
        Command::Quit,
    ];
    for command in &script {
        run(command);
    }
}
```

**Что происходит:**

1. У `Command` четыре варианта с четырьмя разными формами: ничего, два именованных числа, одна `String` и три `u8`.
2. `run` принимает `&Command`, поэтому вызывающий код сохраняет команду. Каждая ветка достаёт ровно те данные, которые есть у её варианта.
3. `{r:02x}` форматирует число как **шестнадцатеричное** (по основанию 16, цифры `0-9a-f`), шириной не менее 2 символов, с дополнением нулями. Поэтому `255, 128, 0` превращается в `ff8000` — общепринятую запись цвета.
4. `vec![...]` создаёт растущий список. Он объясняется в [Уроке 04.7 — Векторы](07-vectors.md); здесь это просто контейнер, по которому можно пройти циклом.

```text
saying: hello
moving by (3, -1)
color is now #ff8000
bye
```

Без перечислений вам понадобились бы четыре отдельные структуры и способ передавать «любую из них». С перечислением один тип охватывает все команды, а `match` следит, чтобы `run` обработал каждую.

> 💡 **Совет:** перечисление — способ моделировать **состояния**. Сравните два способа описать сетевое соединение:
>
> - Структура с флагами: `connected: bool`, `attempt: u32`, `session_id: u64`. Она допускает бессмыслицу вроде «не подключено, но есть id сессии».
> - Перечисление, где у каждого состояния есть только те данные, которые в нём имеют смысл.
>
> ```rust
> #[derive(Debug)]
> enum ConnectionState {
>     Disconnected,
>     Connecting { attempt: u32 },
>     Connected { session_id: u64 },
> }
> ```
>
> Теперь недопустимое сочетание нельзя даже записать. Программисты на Rust называют это **«сделать недопустимые состояния непредставимыми»** (making illegal states unrepresentable), и это одна из самых полезных привычек проектирования.

```rust
#[derive(Debug)]
enum ConnectionState {
    Disconnected,
    Connecting { attempt: u32 },
    Connected { session_id: u64 },
}

fn describe(state: &ConnectionState) -> String {
    match state {
        ConnectionState::Disconnected => String::from("offline"),
        ConnectionState::Connecting { attempt } => format!("trying, attempt {attempt}"),
        ConnectionState::Connected { session_id } => format!("online, session {session_id}"),
    }
}

fn main() {
    let states = [
        ConnectionState::Disconnected,
        ConnectionState::Connecting { attempt: 2 },
        ConnectionState::Connected { session_id: 90210 },
    ];
    for s in &states {
        println!("{}", describe(s));
    }
}
```

```text
offline
trying, attempt 2
online, session 90210
```

## 4. Числа для вариантов

Перечисление только из вариантов-единиц («C-подобное перечисление») может дать каждому варианту число — **дискриминант**. Это удобно, когда число имеет смысл за пределами вашей программы, например код в файле или в сетевом сообщении.

```rust
#[derive(Debug, Clone, Copy)]
enum Level {
    Low = 1,
    Medium = 5,
    High = 10,
}

impl Level {
    fn from_number(n: u8) -> Option<Level> {
        match n {
            1 => Some(Level::Low),
            5 => Some(Level::Medium),
            10 => Some(Level::High),
            _ => None,
        }
    }
}

fn main() {
    let level = Level::High;
    println!("{:?} as number = {}", level, level as u8);
    println!("{:?}", Level::from_number(5));
    println!("{:?}", Level::from_number(7));
}
```

**Что происходит:**

1. `Low = 1` задаёт число этого варианта. Без `= ...` нумерация начинается с 0 и идёт по возрастанию.
2. `level as u8` — это **приведение типа** (cast): оно превращает перечисление в его число. От перечисления к числу `as` разрешено.
3. Обратное направление **не** автоматическое. Не каждое число — допустимый уровень (что такое `7`?). Поэтому мы пишем `from_number`, которая применяет `match` к числу и возвращает `Option<Level>`. `Option` — перечисление из стандартной библиотеки: `Some(value)` значит «вот значение», `None` — «ничего». Это честный способ сказать «здесь возможна неудача»; полностью его объясняет [Урок 04.4](04-option-and-match.md).
4. `_ => None` — ветка-заглушка: любое другое число даёт `None`.

```text
High as number = 10
Some(Medium)
None
```

> 🔒 **Безопасность:** никогда не превращайте недоверенное число в перечисление «силой» (например, с помощью приёмов `unsafe`). Значение вне диапазона создало бы недопустимое перечисление, а это **неопределённое поведение** (поведение, на которое язык не даёт никаких гарантий; программа может упасть или повести себя непредсказуемо). Безопасное преобразование через `match`, показанное выше, — правильный путь: неизвестный ввод становится `None`.

## 5. Превращаем текст в перечисление

Программы читают текст из файлов, с клавиатуры и из сети. Частая задача — один раз, на границе программы, преобразовать текст в перечисление. После этого остальной код работает с безопасным типом.

```rust
#[derive(Debug, PartialEq)]
enum Direction {
    North,
    East,
    South,
    West,
}

fn parse_direction(text: &str) -> Option<Direction> {
    match text {
        "n" | "north" => Some(Direction::North),
        "e" | "east" => Some(Direction::East),
        "s" | "south" => Some(Direction::South),
        "w" | "west" => Some(Direction::West),
        _ => None,
    }
}

fn main() {
    println!("{:?}", parse_direction("east"));
    println!("{:?}", parse_direction("up"));
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn parses_known_words() {
        assert_eq!(parse_direction("north"), Some(Direction::North));
        assert_eq!(parse_direction("w"), Some(Direction::West));
    }

    #[test]
    fn rejects_unknown_words() {
        assert_eq!(parse_direction("up"), None);
        assert_eq!(parse_direction(""), None);
    }
}
```

**Что происходит:**

1. `match text` сравнивает `&str` со строковыми литералами. `"n" | "north"` принимает любое из двух написаний.
2. Известные слова дают `Some(Direction::...)`; всё остальное — `None`. Вызывающий код обязан обработать `None`, поэтому неизвестный ввод никогда не превратится молча в направление.
3. Тесты проверяют хорошие и плохие слова, в том числе пустую строку — классический граничный случай.

```text
Some(East)
None
```

```text
running 2 tests
..
test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

> 🔒 **Безопасность:** считайте любой ввод недоверенным. Разбирайте его в закрытое множество допустимых значений (перечисление) и отвергайте остальное. Тогда `match` по перечислению полный, и никакая «неожиданная строка» не доберётся до глубоких частей программы.

## 6. Как хранится перечисление

Значение перечисления должно уметь вместить **самый большой** из своих вариантов плюс **тег**, который записывает, каким вариантом оно сейчас является:

```text
enum Number { Small(u8), Big(u64) }

 Small(7):  ┌─────┬─────────┬──────────────────────┐
            │ tag │ padding │ 7 (u8), rest unused  │
            │  0  │         │                      │
            └─────┴─────────┴──────────────────────┘
 Big(1000): ┌─────┬─────────┬──────────────────────┐
            │ tag │ padding │ 1000 (u64)           │
            │  1  │         │                      │
            └─────┴─────────┴──────────────────────┘
            same size for every value: 16 bytes
```

Проверьте это через `size_of` (из [Урока 04.1](01-structs.md)):

```rust
use std::mem::size_of;

#[allow(dead_code)]
enum Direction {
    North,
    East,
    South,
    West,
}

#[allow(dead_code)]
enum Number {
    Small(u8),
    Big(u64),
}

#[allow(dead_code)]
enum Shape {
    Circle(f64),
    Square(f64),
    Rectangle { width: f64, height: f64 },
}

fn main() {
    println!("Direction = {} byte", size_of::<Direction>());
    println!("Number    = {} bytes", size_of::<Number>());
    println!("Shape     = {} bytes", size_of::<Shape>());
}
```

**Что происходит:**

1. `#[allow(dead_code)]` говорит компилятору не предупреждать, что варианты в этой демонстрации нигде не создаются.
2. У `Direction` нет нагрузки: нужен только тег, а один байт умеет считать до 255, поэтому размер равен 1.
3. `Number` занимает 16 байт: `u64` требует 8 байт и должен начинаться по адресу, кратному 8, поэтому сначала идут 1-байтовый тег и 7 байт выравнивания. `Small(7)` тратит неиспользуемое место впустую.
4. `Shape` занимает 24 байта: самый большой вариант, `Rectangle`, содержит два `f64` (16 байт) плюс тег, дополненный до 8.

```text
Direction = 1 byte
Number    = 16 bytes
Shape     = 24 bytes
```

> ⚡ **Производительность:** перечисление так же велико, как его самый большой вариант. Если один редкий вариант несёт огромную нагрузку, за неё платит каждое значение перечисления. Обычное лекарство — хранить большую нагрузку в куче (через `Box`, [Урок 05.9 — Умные указатели](../05-rust-intermediate/09-smart-pointers.md)). Прежде чем беспокоиться, измерьте с помощью `size_of`.

## 7. Перечисления, которыми вы уже пользуетесь и будете пользоваться

Два перечисления из стандартной библиотеки встречаются почти в каждой программе на Rust:

- `Option<T>` — `Some(value)` или `None`. Это способ Rust сказать «значения может не быть». Следующий урок.
- `Result<T, E>` — `Ok(value)` или `Err(error)`. С `Result` вы встречались при разборе текста на Этапе 02; [Урок 04.10 — Обработка ошибок](10-error-handling.md) рассматривает его подробно.

Оба — обычные перечисления с нагрузкой, определённые в стандартной библиотеке теми же средствами, что в этом уроке. Никакой магии.

## 8. Тестирование перечислений

> 🧪 **Тестирование:** для каждого метода, который делает `match`, пишите хотя бы один тест на каждый вариант — варианты и есть ваши тестовые случаи. Добавьте перечислению `PartialEq` и `Debug`, чтобы `assert_eq!` мог сравнивать значения.

```rust
#[derive(Debug, Clone, Copy, PartialEq)]
enum Day {
    Monday,
    Tuesday,
    Wednesday,
    Thursday,
    Friday,
    Saturday,
    Sunday,
}

impl Day {
    fn is_weekend(self) -> bool {
        match self {
            Day::Saturday | Day::Sunday => true,
            _ => false,
        }
    }

    fn next(self) -> Day {
        match self {
            Day::Monday => Day::Tuesday,
            Day::Tuesday => Day::Wednesday,
            Day::Wednesday => Day::Thursday,
            Day::Thursday => Day::Friday,
            Day::Friday => Day::Saturday,
            Day::Saturday => Day::Sunday,
            Day::Sunday => Day::Monday,
        }
    }
}

fn main() {
    let mut day = Day::Friday;
    for _ in 0..4 {
        println!("{:?}: weekend? {}", day, day.is_weekend());
        day = day.next();
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn weekend_days() {
        assert!(Day::Saturday.is_weekend());
        assert!(Day::Sunday.is_weekend());
        assert!(!Day::Monday.is_weekend());
    }

    #[test]
    fn week_wraps_around() {
        assert_eq!(Day::Sunday.next(), Day::Monday);
        assert_eq!(Day::Monday.next(), Day::Tuesday);
    }
}
```

**Что происходит:**

1. `is_weekend` использует заглушку, потому что «все остальные дни» действительно один случай.
2. `next` намеренно перечисляет все семь вариантов: если вы добавите восьмой, компилятор заставит вас обновить функцию.
3. Тест `week_wraps_around` проверяет границу: последний день ведёт обратно к первому.

```text
Friday: weekend? false
Saturday: weekend? true
Sunday: weekend? true
Monday: weekend? false
```

```text
running 2 tests
..
test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

## Частые ошибки

> ⚠️ **Частая ошибка:** опечатка в имени варианта.

```text
error[E0599]: no variant, associated function, or constant named `Nort` found for enum `Direction` in the current scope
 --> src/main.rs:8:24
  |
8 |     let d = Direction::Nort;
  |                        ^^^^ variant, associated function, or constant not found in `Direction`
  |
help: there is a variant with a similar name
  |
8 |     let d = Direction::North;
  |                            +
```

Компилятор предлагает правильное написание. В этом и состоит преимущество перед текстовыми значениями.

> ⚠️ **Частая ошибка:** забыть имя перечисления перед вариантом: `let d = North;`.

```text
error[E0425]: cannot find value `North` in this scope
 --> src/main.rs:8:13
  |
8 |     let d = North;
  |             ^^^^^ not found in this scope
  |
help: consider importing this unit variant
  |
1 + use crate::Direction::North;
  |
```

Пишите `Direction::North`. (Варианты можно импортировать через `use Direction::*;`, но тогда трудно понять, откуда берутся имена; избегайте этого, кроме небольших `match`.)

> ⚠️ **Частая ошибка:** неверная нагрузка.

```rust
let a = Command::Say("hello");
let b = Command::Move { x: 1 };
```

```text
error[E0308]: mismatched types
 --> src/main.rs:7:26
  |
7 |     let a = Command::Say("hello");
  |             ------------ ^^^^^^^ expected `String`, found `&str`
  |             |
  |             arguments to this enum variant are incorrect
  |
help: try using a conversion method
  |
7 |     let a = Command::Say("hello".to_string());
  |                                 ++++++++++++

error[E0063]: missing field `y` in initializer of `Command`
 --> src/main.rs:8:13
  |
8 |     let b = Command::Move { x: 1 };
  |             ^^^^^^^^^^^^^ missing `y`
```

Нагрузка подчиняется тем же правилам, что функции и структуры: точные типы и все поля. `"hello"` — это `&str`, а вариант хочет владеющую `String`.

> ⚠️ **Частая ошибка:** сравнение через `==` или вывод через `{}` без нужного трейта.

```text
error[E0369]: binary operation `==` cannot be applied to type `Direction`
 --> src/main.rs:8:10
  |
8 |     if d == Direction::South {
  |        - ^^ ---------------- Direction
  |        |
  |        Direction
  |
help: consider annotating `Direction` with `#[derive(PartialEq)]`
```

```text
error[E0277]: `Direction` doesn't implement `std::fmt::Display`
 --> src/main.rs:6:20
  |
6 |     println!("{}", Direction::North);
  |               --   ^^^^^^^^^^^^^^^^ `Direction` cannot be formatted with the default formatter
  |
  = note: in format strings you may be able to use `{:?}` (or `{:#?}` for pretty-print) instead
```

Выведите `PartialEq` для `==`, выведите `Debug` для `{:?}` или напишите `Display` вручную для `{}` ([Урок 04.2](02-methods-and-associated-functions.md)).

## Попробуйте сами

1. Создайте проект:

   ```text
   cd ~/projects/rust
   cargo new code/p04-enums
   cd code/p04-enums
   ```

2. Поместите в `src/main.rs`:

   ```rust
   #[derive(Debug, Clone, Copy, PartialEq)]
   enum Light {
       Red,
       Green,
       Yellow,
   }

   impl Light {
       fn next(self) -> Light {
           match self {
               Light::Red => Light::Green,
               Light::Green => Light::Yellow,
               Light::Yellow => Light::Red,
           }
       }

       fn seconds(self) -> u32 {
           match self {
               Light::Red => 30,
               Light::Green => 25,
               Light::Yellow => 5,
           }
       }
   }

   fn main() {
       let mut light = Light::Red;
       for _ in 0..4 {
           println!("{:?} for {} seconds", light, light.seconds());
           light = light.next();
       }
   }
   ```

3. Запустите `cargo run`. Ожидаемый вывод:

   ```text
   Red for 30 seconds
   Green for 25 seconds
   Yellow for 5 seconds
   Red for 30 seconds
   ```

4. Добавьте четвёртый вариант `RedYellow` (в некоторых странах его показывают перед зелёным). Пока **не** правьте `match` — запустите `cargo build` и прочитайте две ошибки `E0004`. Затем исправьте обе.
5. Сделайте коммит с сообщением в стиле Урока 01.9:

   ```text
   cd ~/projects/rust
   git add code/p04-enums
   git commit -m "feat(p04): traffic light enum"
   ```

## Упражнения

### Упражнение 1 — Монеты (легко)

Объявите `enum Coin { Penny, Nickel, Dime, Quarter }` (выведите `Debug` и `PartialEq`). Напишите `fn cents(coin: &Coin) -> u32`, возвращающую 1, 5, 10 и 25. В `main` сложите стоимость небольшого массива монет. Напишите тест для двух монет.

<details>
<summary>Подсказка</summary>

В `match coin { Coin::Penny => 1, ... }` по одной ветке на вариант. Поскольку `coin` — ссылка, образцы выглядят так же: `Coin::Penny`. Идите циклом: `for coin in &purse { total += cents(coin); }`.

</details>

<details>
<summary>Ответ</summary>

```rust
#[derive(Debug, PartialEq)]
enum Coin {
    Penny,
    Nickel,
    Dime,
    Quarter,
}

fn cents(coin: &Coin) -> u32 {
    match coin {
        Coin::Penny => 1,
        Coin::Nickel => 5,
        Coin::Dime => 10,
        Coin::Quarter => 25,
    }
}

fn main() {
    let purse = [Coin::Quarter, Coin::Dime, Coin::Dime, Coin::Penny];
    let mut total = 0;
    for coin in &purse {
        total += cents(coin);
    }
    println!("{total} cents");
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn values() {
        assert_eq!(cents(&Coin::Penny), 1);
        assert_eq!(cents(&Coin::Quarter), 25);
    }
}
```

Вывод: `46 cents`.

Почему это работает: `match` — выражение; каждая ветка даёт `u32`, а проверка на исчерпывающность гарантирует, что у каждой монеты есть значение. (Компилятор может предупредить, что `Nickel` в `main` не создаётся; здесь предупреждение безвредно.)

</details>

### Упражнение 2 — Фигуры с тремя видами данных (средне)

Объявите `enum Shape` с вариантами-структурами `Circle { radius }`, `Rectangle { width, height }` и `Triangle { base, height }` (все `f64`). Добавьте методы `area(&self) -> f64` и `name(&self) -> &'static str`. Протестируйте площадь прямоугольника (12.0 для 3 × 4), площадь треугольника (7.5 для основания 6 и высоты 2.5) и одно имя.

<details>
<summary>Подсказка</summary>

Площадь треугольника — `0.5 * base * height`. В `name` нагрузка не нужна, поэтому пишите `Shape::Circle { .. }` — `..` значит «поля меня не интересуют». Числа с плавающей точкой сравнивайте с допуском.

</details>

<details>
<summary>Ответ</summary>

```rust
#[derive(Debug, Clone, PartialEq)]
enum Shape {
    Circle { radius: f64 },
    Rectangle { width: f64, height: f64 },
    Triangle { base: f64, height: f64 },
}

impl Shape {
    fn area(&self) -> f64 {
        match self {
            Shape::Circle { radius } => std::f64::consts::PI * radius * radius,
            Shape::Rectangle { width, height } => width * height,
            Shape::Triangle { base, height } => 0.5 * base * height,
        }
    }

    fn name(&self) -> &'static str {
        match self {
            Shape::Circle { .. } => "circle",
            Shape::Rectangle { .. } => "rectangle",
            Shape::Triangle { .. } => "triangle",
        }
    }
}

fn main() {
    let shapes = [
        Shape::Circle { radius: 1.0 },
        Shape::Rectangle { width: 3.0, height: 4.0 },
        Shape::Triangle { base: 6.0, height: 2.5 },
    ];
    for s in &shapes {
        println!("{}: {:.2}", s.name(), s.area());
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn rectangle_area() {
        let r = Shape::Rectangle { width: 3.0, height: 4.0 };
        assert!((r.area() - 12.0).abs() < 1e-9);
    }

    #[test]
    fn triangle_area() {
        let t = Shape::Triangle { base: 6.0, height: 2.5 };
        assert!((t.area() - 7.5).abs() < 1e-9);
    }

    #[test]
    fn names() {
        assert_eq!(Shape::Circle { radius: 1.0 }.name(), "circle");
    }
}
```

Вывод `cargo run`:

```text
circle: 3.14
rectangle: 12.00
triangle: 7.50
```

Почему это работает: каждый вариант несёт ровно те числа, которые нужны его формуле. `'static` в возвращаемом типе означает, что текст живёт всю программу (строковые литералы такие); это объясняется в [Уроке 05.3](../05-rust-intermediate/03-lifetimes.md).

</details>

### Упражнение 3 — Крошечный токенизатор (сложно)

Объявите `enum Token { Number(i64), Plus, Minus, Star }`. Напишите `fn tokenize_word(word: &str) -> Option<Token>`: `"+"`, `"-"`, `"*"` дают операторы; слово, которое разбирается как `i64`, даёт `Number`; всё остальное даёт `None`. Протестируйте `"12"`, `"-7"`, `"+"`, `"x"` и `""`.

<details>
<summary>Подсказка</summary>

Используйте `match word` с ветками-литералами для трёх операторов, а в последней ветке привяжите слово к имени (`other => ...`). Внутри неё используйте `match other.parse::<i64>() { Ok(n) => ..., Err(_) => ... }`. `"-7"` — это число, а не оператор «минус», потому что слово целиком — `-7`, а не `-`.

</details>

<details>
<summary>Ответ</summary>

```rust
#[derive(Debug, PartialEq)]
enum Token {
    Number(i64),
    Plus,
    Minus,
    Star,
}

fn tokenize_word(word: &str) -> Option<Token> {
    match word {
        "+" => Some(Token::Plus),
        "-" => Some(Token::Minus),
        "*" => Some(Token::Star),
        other => match other.parse::<i64>() {
            Ok(n) => Some(Token::Number(n)),
            Err(_) => None,
        },
    }
}

fn main() {
    for word in ["12", "+", "x", "-7", "*"] {
        println!("{:>3} -> {:?}", word, tokenize_word(word));
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn numbers_and_operators() {
        assert_eq!(tokenize_word("12"), Some(Token::Number(12)));
        assert_eq!(tokenize_word("-7"), Some(Token::Number(-7)));
        assert_eq!(tokenize_word("+"), Some(Token::Plus));
    }

    #[test]
    fn garbage_is_rejected() {
        assert_eq!(tokenize_word("x"), None);
        assert_eq!(tokenize_word(""), None);
    }
}
```

Вывод `cargo run`:

```text
 12 -> Some(Number(12))
  + -> Some(Plus)
  x -> None
 -7 -> Some(Number(-7))
  * -> Some(Star)
```

Почему это работает: ветки проверяются по порядку, поэтому точные слова `+`, `-`, `*` перехватываются первыми; последняя ветка `other` подходит к любому слову и привязывает его к переменной. `parse::<i64>()` возвращает `Result`, а его вариант `Err` превращается в `None`. Это первый шаг любого калькулятора или интерпретатора: превратить текст в список токенов с чёткими типами.

</details>

## Вопросы для самопроверки

1. Структура или перечисление: пользователь с именем и возрастом; состояние двери (открыта, закрыта, заперта)?

<details>
<summary>Ответ</summary>

Пользователь — структура (имя **и** возраст). Состояние двери — перечисление (открыта **или** закрыта **или** заперта).

</details>

2. Что такое нагрузка варианта?

<details>
<summary>Ответ</summary>

Данные, которые несёт этот вариант, например `f64` в `Circle(f64)`. У каждого варианта может быть своя нагрузка или никакой.

</details>

3. Что значит «`match` должен быть исчерпывающим» и что делает компилятор, если это не так?

<details>
<summary>Ответ</summary>

Каждое возможное значение должно быть охвачено какой-то веткой. Если ветки не хватает, компиляция завершается ошибкой `E0004` с названием непокрытого случая.

</details>

4. Почему сообщение компилятора при добавлении нового варианта в перечисление — это *хорошо*?

<details>
<summary>Ответ</summary>

Каждый `match` без ветки-заглушки перестаёт компилироваться, поэтому компилятор показывает все места, где нужно обработать новый случай. Ничего не забывается молча.

</details>

5. Какого размера значение перечисления в памяти?

<details>
<summary>Ответ</summary>

Размером с самый большой вариант плюс тег, записывающий текущий вариант (с округлением вверх для выравнивания). Все значения одного перечисления имеют одинаковый размер.

</details>

6. Как преобразовать перечисление с дискриминантами в число, а число обратно в перечисление?

<details>
<summary>Ответ</summary>

Перечисление в число: `value as u8`. Число в перечисление: автоматического способа нет, потому что не каждое число допустимо. Напишите функцию с `match`, возвращающую `Option<Enum>` (`None` для неизвестных чисел).

</details>

7. Зачем разбирать текст в перечисление на границе программы, а не передавать строки по всему коду?

<details>
<summary>Ответ</summary>

После разбора внутри программы существуют только допустимые значения, а каждый `match` по перечислению полон. Неизвестный или вредоносный ввод отвергается один раз, на границе, как `None` (или позже как ошибка).

</details>

8. Когда `_` — хорошая ветка в `match` по перечислению, а когда плохая идея?

<details>
<summary>Ответ</summary>

Хорошая — когда «всё остальное» действительно один случай (как выходные дни). Плохая — когда каждый вариант нужно обрабатывать осознанно, потому что новый вариант молча попадёт в `_`, а компилятор не предупредит.

</details>

## Итоги

- **Структура** — это «и то, и то»; **перечисление** — «или то, или то». Значение перечисления всегда ровно один вариант.
- Значения создаются как `Enum::Variant`; варианты бывают единицами, кортежами (`Circle(f64)`) или структурами (`Rectangle { width, height }`), и каждый может нести свои данные.
- `match` разбирает перечисление, привязывает нагрузку к именам и должен быть **исчерпывающим** — компилятор находит все места, которые нужно обновить, когда вы добавляете вариант.
- Используйте перечисления для моделирования состояний и чтобы сделать недопустимые состояния непредставимыми.
- У перечислений могут быть методы в блоках `impl`, и они выводят `Debug`, `Clone`, `Copy` и `PartialEq` так же, как структуры.
- У C-подобного перечисления могут быть числа (`Low = 1`); `as` превращает перечисление в число, но число в перечисление — только через проверяющий `match`.
- В памяти перечисление — это тег плюс место под самый большой вариант.
- Разбирайте недоверенный текст в перечисление один раз и отвергайте всё остальное.

## Источники и что почитать

- [The Rust Programming Language — 6.1 Defining an Enum](https://doc.rust-lang.org/book/ch06-01-defining-an-enum.html) — глава официальной книги по этой теме.
- [The Rust Programming Language — 6.2 The match Control Flow Construct](https://doc.rust-lang.org/book/ch06-02-match.html) — больше о `match`; Урок 04.4 продолжает эту тему.
- [Rust by Example — Enums](https://doc.rust-lang.org/rust-by-example/custom_types/enum.html) — короткие запускаемые примеры, включая C-подобные перечисления.
- [The Rust Reference — Enumerations](https://doc.rust-lang.org/reference/items/enumerations.html) — точные правила, включая дискриминанты и приведения.
- [std::mem::size_of](https://doc.rust-lang.org/std/mem/fn.size_of.html) — документация функции из раздела 6.

---

[🏠 Начало](../START-HERE.md) · [← 04.2 Методы и ассоциированные функции](02-methods-and-associated-functions.md) · [📋 Этап 04](README.md) · [04.4 Option и match →](04-option-and-match.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
