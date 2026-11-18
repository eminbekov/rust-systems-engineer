<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 04.1 Структуры](01-structs.md) · [📋 Этап 04](README.md) · [04.3 Перечисления →](03-enums.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/04-structs-enums-collections-errors/02-methods-and-associated-functions.md) <!-- nav -->

# Урок 04.2 — Методы и ассоциированные функции

> **Этап 04 — Структуры, перечисления, коллекции и ошибки** · Урок 2 из 12 · ⏱ около 4 часов (2 занятия)

## Что вы узнаете

- Как привязывать функции к типу с помощью блока `impl`.
- Чем **метод** (есть `self`) отличается от **ассоциированной функции** (нет `self`) и как вызывать каждого из них.
- Три формы `self`: `&self`, `&mut self` и `self`, и как каждая связана с владением и заимствованием.
- Полезные приёмы: конструкторы `new`, цепочки в стиле строителя, ассоциированные константы, `Default` и вывод через `Display`.
- Соглашения об именах методов (`as_`, `to_`, `into_`, `is_`, `with_`).

## Перед началом

- Вы прошли [Урок 04.1 — Структуры](01-structs.md). Вы умеете объявлять структуру, создавать экземпляры и выводить `Debug`.
- Вы помните три способа, которыми функция может получить значение: по значению, по ссылке `&` и по ссылке `&mut` ([Урок 03.4 — Ссылки и заимствование](../03-ownership-and-borrowing/04-references-and-borrowing.md) и [Урок 03.5](../03-ownership-and-borrowing/05-mutable-references-and-the-borrow-rules.md)).

## Новые слова

| Слово | Значение |
|---|---|
| **блок impl** (implementation block, блок реализации) | Блок `impl TypeName { ... }`, в котором лежат функции, принадлежащие типу. |
| **метод** (method) | Функция в блоке `impl`, у которой первый параметр — `self`. Вызывается через точку: `value.method()`. |
| **ассоциированная функция** (associated function) | Функция в блоке `impl` без `self`. Вызывается через `::`: `Type::function()`. |
| **получатель** (receiver) | Значение, у которого вызывают метод, — то, что стоит слева от точки. Внутри метода оно называется `self`. |
| **`Self`** | Короткое имя «типа, для которого написан этот блок `impl`». Внутри `impl Rectangle` `Self` означает `Rectangle`. |
| **`self`** | Значение-получатель внутри метода. |
| **конструктор** (constructor) | Функция, которая строит новый экземпляр. В Rust это лишь соглашение: ассоциированная функция, обычно с именем `new`. |
| **ассоциированная константа** (associated constant) | `const`, объявленная внутри блока `impl`. Она принадлежит типу: `Type::NAME`. |
| **паттерн «строитель»** (builder pattern) | Стиль, при котором маленькие методы по очереди задают по одной опции и возвращают значение, поэтому вызовы можно связывать в цепочку: `Config::new().with_port(80)`. |
| **цепочка методов** (method chaining) | Вызов нескольких методов подряд, каждый — для результата предыдущего. |
| **автоматическое взятие ссылки** (auto-referencing) | Компилятор сам добавляет к получателю `&`, `&mut` или `*`, чтобы `value.method()` подошёл к форме `self` метода. |
| **Display** | Трейт, на котором работает `{}`. Вы пишете его вручную, чтобы определить, как значение выглядит для пользователей. |
| **Default** | Трейт с одной функцией `default()`, которая возвращает разумное «пустое» значение типа. |
| **трейт** (trait) | Именованный набор возможностей, которые может иметь тип (объясняется в [Уроке 05.2 — Трейты](../05-rust-intermediate/02-traits.md)). В этом уроке мы только копируем небольшие рецепты для `Display` и `Default`. |
| **API** (Application Programming Interface, программный интерфейс приложения) | Набор функций и типов, которые часть кода предлагает остальным. Методы типа — это его API. |

## 1. От функций к методам

В Уроке 04.1 вы писали `fn area(rect: &Rectangle) -> u32` рядом со структурой. Это работает, но между структурой и функцией нет связи, кроме параметра. Если функций о прямоугольниках десять, они разбросаны по файлу.

Rust позволяет поместить такие функции *внутрь* **блока `impl`** типа. Тогда `area` — это «то, что умеет делать прямоугольник». Вызов меняется с `area(&rect)` на `rect.area()`. Представьте так: структура — существительное, а методы — её глаголы.

```rust
#[derive(Debug)]
struct Rectangle {
    width: u32,
    height: u32,
}

impl Rectangle {
    // ассоциированная функция: нет self, вызывается через ::
    fn new(width: u32, height: u32) -> Self {
        Self { width, height }
    }

    fn square(size: u32) -> Self {
        Self::new(size, size)
    }

    // методы: первый параметр — self
    fn area(&self) -> u32 {
        self.width * self.height
    }

    fn can_hold(&self, other: &Rectangle) -> bool {
        self.width > other.width && self.height > other.height
    }
}

fn main() {
    let big = Rectangle::new(30, 50);
    let small = Rectangle::square(10);

    println!("area of big = {}", big.area());
    println!("big can hold small? {}", big.can_hold(&small));
    println!("small can hold big? {}", small.can_hold(&big));
    println!("{:?}", small);
}
```

**Что происходит:**

1. `impl Rectangle { ... }` говорит: «всё, что в этих скобках, принадлежит типу `Rectangle`».
2. Внутри блока `Self` — другое имя для `Rectangle`. `-> Self` значит «возвращает `Rectangle`», а `Self { width, height }` строит его (с краткой инициализацией полей из Урока 04.1). Если позже вы переименуете структуру, придётся менять меньше строк.
3. У `new` и `square` **нет параметра `self`**, поэтому это *ассоциированные функции*. Их вызывают через имя типа с двумя двоеточиями: `Rectangle::new(30, 50)`. Это обычный способ создавать значения.
4. `square` вызывает другую ассоциированную функцию: `Self::new(size, size)`.
5. У `area` и `can_hold` первый параметр — `&self`. Это *методы*. `big.area()` вызывает `area`, передавая `big` как `self`.
6. `&self` — сокращение от `self: &Self`, то есть `self: &Rectangle`: метод заимствует прямоугольник для чтения. `self.width` читает его поле.
7. `can_hold(&self, other: &Rectangle)` получает второй прямоугольник. Вызов `big.can_hold(&small)` передаёт `&small` как `other`.

```text
area of big = 1500
big can hold small? true
small can hold big? false
Rectangle { width: 10, height: 10 }
```

> 📏 **Соглашение:** `new` — не ключевое слово. Это просто имя, которого каждый программист на Rust ждёт для «основного конструктора». Если у типа есть очевидный способ его создать, предоставьте `Type::new(...)`.

## 2. Три формы `self`

Первый параметр метода определяет, что метод может делать со значением. Это те же три варианта, что и в теме владения:

| Форма | Значение | Вызывающий код после вызова | Когда использовать |
|---|---|---|---|
| `&self` | заимствует значение только для чтения | по-прежнему владеет значением и может им пользоваться | метод только смотрит (геттеры, `area`, `is_empty`) |
| `&mut self` | заимствует значение с правом изменения | по-прежнему владеет им; значение могло измениться | метод изменяет значение (`deposit`, `push`) |
| `self` | забирает владение; значение перемещается внутрь | больше не может пользоваться значением | метод поглощает значение (`close`, `into_string`) |

Полный пример со всеми тремя:

```rust
#[derive(Debug)]
struct BankAccount {
    owner: String,
    balance_cents: u64,
}

impl BankAccount {
    fn new(owner: &str) -> Self {
        Self {
            owner: String::from(owner),
            balance_cents: 0,
        }
    }

    // &self: только чтение
    fn balance_cents(&self) -> u64 {
        self.balance_cents
    }

    // &mut self: может изменить счёт
    fn deposit(&mut self, cents: u64) {
        self.balance_cents += cents;
    }

    // возвращает false, когда денег не хватает
    fn withdraw(&mut self, cents: u64) -> bool {
        if cents > self.balance_cents {
            false
        } else {
            self.balance_cents -= cents;
            true
        }
    }

    // self: забирает владение; счёт поглощается
    fn close(self) -> String {
        format!("{} closed the account with {} cents", self.owner, self.balance_cents)
    }
}

fn main() {
    let mut account = BankAccount::new("alice");
    account.deposit(10_000);
    println!("withdraw 3000: {}", account.withdraw(3_000));
    println!("withdraw 9000: {}", account.withdraw(9_000));
    println!("balance: {}", account.balance_cents());
    let receipt = account.close();
    println!("{receipt}");
}
```

**Что происходит:**

1. `BankAccount::new("alice")` создаёт счёт с нулевым балансом. Параметр имеет тип `&str`; `String::from(owner)` делает владеющую копию для поля структуры.
2. `account` объявлена как `let mut`, потому что мы вызываем методы с `&mut self`.
3. `deposit` добавляет `10_000` центов (подчёркивание нужно только для удобства чтения). `withdraw(3_000)` успешно выполняется и возвращает `true`; баланс становится 7000.
4. `withdraw(9_000)` больше баланса, поэтому ничего не меняется, а метод возвращает `false`. (Лучше было бы вернуть `Result`, который объясняет, *почему* операция не удалась. Вы узнаете об этом в [Уроке 04.10 — Обработка ошибок](10-error-handling.md).)
5. У `balance_cents()` есть `&self`, и он называется так же, как поле. В Rust геттер называют как поле, без префикса `get_`.
6. `close(self)` забирает владение. Он собирает текст через `format!` (как `println!`, но возвращает `String`) и возвращает его. После этого вызова счёта больше нет: он переместился в `close` и был уничтожен в его конце.

```text
withdraw 3000: true
withdraw 9000: false
balance: 7000
alice closed the account with 7000 cents
```

### Ошибки, которые вы встретите

**Метод, которому нужен `&mut self`, но объявлен `&self`:**

```rust
struct Counter {
    count: u32,
}

impl Counter {
    fn increment(&self) {
        self.count += 1;
    }
}
```

```text
error[E0594]: cannot assign to `self.count`, which is behind a `&` reference
 --> src/main.rs:7:9
  |
7 |         self.count += 1;
  |         ^^^^^^^^^^^^^^^ `self` is a `&` reference, so it cannot be written to
  |
help: consider changing this to be a mutable reference
  |
6 |     fn increment(&mut self) {
  |                   +++
```

Смысл: через ссылку `&` можно только читать. Исправление: `&mut self`.

**Вызов метода с `&mut self` у переменной без `mut`:**

```rust
let c = Counter { count: 0 };
c.increment();
```

```text
error[E0596]: cannot borrow `c` as mutable, as it is not declared as mutable
  --> src/main.rs:13:5
   |
13 |     c.increment();
   |     ^ cannot borrow as mutable
   |
help: consider changing this to be mutable
   |
12 |     let mut c = Counter { count: 0 };
   |         +++
```

Вызов `c.increment()` неявно берёт `&mut c`. Для этого переменная должна быть изменяемой: `let mut c`.

**Использование значения после того, как его поглотил метод с `self`:**

```rust
let account = BankAccount { balance_cents: 500 };
let money = account.close();
println!("{}", account.balance_cents);
```

```text
error[E0382]: borrow of moved value: `account`
  --> src/main.rs:15:20
   |
12 |     let account = BankAccount { balance_cents: 500 };
   |         ------- move occurs because `account` has type `BankAccount`, which does not implement the `Copy` trait
13 |     let money = account.close();
   |                         ------- `account` moved due to this method call
14 |     println!("{money}");
15 |     println!("{}", account.balance_cents);
   |                    ^^^^^^^^^^^^^^^^^^^^^ value borrowed here after move
   |
note: `BankAccount::close` takes ownership of the receiver `self`, which moves `account`
```

Компилятор даже сообщает, какой метод переместил значение. Это правило владения из Этапа 03, применённое к вызовам методов.

> 💡 **Совет:** если вы не уверены, какую форму выбрать, начните с `&self`. Переходите на `&mut self`, когда компилятор скажет, что нужно писать, и на `self` — только когда метод действительно завершает жизнь значения.

## 3. Ассоциированные функции: без `self`

Функция в блоке `impl` без `self` — это **ассоциированная функция**. Она работает не с существующим экземпляром, а «относится к типу». Главное применение — конструкторы. Вызывается она через `::`:

```text
let s = Rectangle::square(5);     // right: associated function, with ::
let s = r.square(5);              // wrong: it has no self, so no dot
```

Неправильный вариант даёт полезную ошибку:

```text
error[E0599]: no method named `square` found for struct `Rectangle` in the current scope
  --> src/main.rs:14:15
   |
 1 | struct Rectangle {
   | ---------------- method `square` not found for this struct
...
14 |     let s = r.square(5);
   |               ^^^^^^ this is an associated function, not a method
   |
   = note: found the following associated functions; to be used as methods, functions must have a `self` parameter
help: use associated function syntax instead
   |
14 -     let s = r.square(5);
14 +     let s = Rectangle::square(5);
   |
```

Обратная ошибка — вызов метода через тип без передачи получателя, `Rectangle::area()` — даёт `error[E0061]: this function takes 1 argument but 0 arguments were supplied` с примечанием `argument #1 of type &Rectangle is missing`. Это намёк на то, чем на самом деле является метод: `area(&self)` — обычная функция, первый аргумент которой — прямоугольник.

## 4. Как работает вызов через точку

Вы можете удивиться: `area` ожидает `&Rectangle`, а мы пишем `r.area()`, где `r` — `Rectangle`. Кто добавил `&`? Компилятор. Это **автоматическое взятие ссылки** (auto-referencing): когда вы пишете `value.method()`, компилятор смотрит на форму `self` у метода и сам вставляет `&`, `&mut` или `*`.

```rust
struct Rectangle {
    width: u32,
    height: u32,
}

impl Rectangle {
    fn area(&self) -> u32 {
        self.width * self.height
    }

    // у геттера то же имя, что и у поля, без префикса "get_"
    fn width(&self) -> u32 {
        self.width
    }
}

fn main() {
    let r = Rectangle { width: 6, height: 7 };
    let a = r.area();
    let b = Rectangle::area(&r);
    let c = (&r).area();
    println!("{a} {b} {c}");
    println!("field: {}, method: {}", r.width, r.width());
}
```

**Что происходит:**

1. `r.area()` — короткая форма. Компилятор превращает её в `Rectangle::area(&r)`.
2. `Rectangle::area(&r)` — длинная форма: метод вызывается как обычная ассоциированная функция, а получатель вы передаёте сами.
3. В `(&r).area()` ссылка написана вручную; компилятор принимает и это.
4. Все три дают 42.
5. `r.width` (без скобок) — это поле. `r.width()` (со скобками) — метод с тем же именем. Rust различает их по скобкам.

```text
42 42 42
field: 6, method: 6
```

Благодаря автоматическому взятию ссылки вам никогда не нужен особый оператор для «вызова метода через ссылку»: одна и та же точка работает и для значений, и для ссылок.

## 5. Ещё инструменты в блоках `impl`

### Несколько блоков `impl` и ассоциированные константы

У типа может быть много блоков `impl`. Это удобно, чтобы группировать методы. Внутри блока можно также объявлять **ассоциированные константы**:

```rust
#[derive(Debug, Default)]
struct ServerConfig {
    host: String,
    port: u16,
    verbose: bool,
}

impl ServerConfig {
    const DEFAULT_PORT: u16 = 8080;

    fn new(host: &str) -> Self {
        Self {
            host: String::from(host),
            port: Self::DEFAULT_PORT,
            verbose: false,
        }
    }

    fn with_port(mut self, port: u16) -> Self {
        self.port = port;
        self
    }

    fn with_verbose(mut self, verbose: bool) -> Self {
        self.verbose = verbose;
        self
    }
}

// второй блок impl разрешён
impl ServerConfig {
    fn address(&self) -> String {
        format!("{}:{}", self.host, self.port)
    }
}

fn main() {
    let plain = ServerConfig::new("localhost");
    println!("{}", plain.address());

    let custom = ServerConfig::new("example.org").with_port(9000).with_verbose(true);
    println!("{:?}", custom);

    let blank = ServerConfig::default();
    println!("{:?}", blank);

    let partly = ServerConfig {
        host: String::from("db"),
        ..Default::default()
    };
    println!("{:?}", partly);
}
```

**Что происходит:**

1. `const DEFAULT_PORT: u16 = 8080;` — константа, принадлежащая типу. Внутри блока к ней обращаются как `Self::DEFAULT_PORT`, снаружи — как `ServerConfig::DEFAULT_PORT`. Константы пишутся в `SCREAMING_SNAKE_CASE`. Именованная константа лучше, чем «магическое число» `8080`, повторённое в нескольких местах.
2. `with_port(mut self, port: u16) -> Self` принимает `self` **по значению** и помечает параметр `mut`. Это `mut` лишь разрешает менять локальную копию значения, которое переместилось внутрь. Метод задаёт поле и возвращает `self`.
3. Поскольку каждый метод `with_...` поглощает значение и возвращает его обратно, вызовы можно **связывать в цепочку**: `ServerConfig::new("example.org").with_port(9000).with_verbose(true)`. Каждая точка работает с результатом предыдущего вызова. Это *паттерн «строитель»*, и именно так в Rust получаются «необязательные именованные аргументы».
4. `address(&self)` лежит во втором блоке `impl`. Компилятор считает оба блока одним.
5. `#[derive(Default)]` просит компилятор написать `ServerConfig::default()`, заполняющую каждое поле его собственным значением по умолчанию: пустая `String`, `0` для чисел, `false` для `bool`.
6. `..Default::default()` — синтаксис обновления структуры из Урока 04.1, где базой служит свежее значение по умолчанию. Мы задаём только `host`, остальное берём из значения по умолчанию.

```text
localhost:8080
ServerConfig { host: "example.org", port: 9000, verbose: true }
ServerConfig { host: "", port: 0, verbose: false }
ServerConfig { host: "db", port: 0, verbose: false }
```

> 📏 **Соглашение:** используйте `Default`, когда «все поля по умолчанию» — осмысленное значение. Не выводите его, если значение по умолчанию было бы сломанным или запутывающим состоянием (например, порт 0, скорее всего, бесполезен, поэтому `new` здесь ставит 8080).

### Вывод для пользователей: `Display`

`{:?}` (`Debug`) предназначен для разработчиков. Для пользователей тип реализует **`Display`**, и тогда работает `{}`. Вы пишете его вручную, потому что только вы знаете, как значение должно выглядеть. Пока считайте следующий код рецептом; полное объяснение трейтов будет в [Уроке 05.2](../05-rust-intermediate/02-traits.md).

```rust
use std::fmt;

struct Point {
    x: i32,
    y: i32,
}

impl fmt::Display for Point {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        write!(f, "({}, {})", self.x, self.y)
    }
}

fn main() {
    let p = Point { x: 3, y: -4 };
    println!("point is {}", p);
    let text = p.to_string();
    println!("{} has {} characters", text, text.len());
}
```

**Что происходит:**

1. `use std::fmt;` позволяет писать `fmt::Display` вместо `std::fmt::Display`.
2. `impl fmt::Display for Point` означает «реализовать возможность `Display` для типа `Point`». Эта возможность требует один метод с именем `fmt`.
3. `fmt` получает `&self` (точку) и `f` — `Formatter`, то есть место, куда записывается текст.
4. `write!(f, "({}, {})", self.x, self.y)` похож на `println!`, но пишет в `f`. Его результат (`fmt::Result`) возвращается из функции.
5. Когда `Display` реализован, работает `{}`, а в придачу бесплатно появляется `.to_string()`. `text.len()` считает байты; в тексте `(3, -4)` их 7.

```text
point is (3, -4)
(3, -4) has 7 characters
```

## 6. Соглашения об именах методов

Программисты на Rust читают имя метода как обещание о стоимости и владении. Официальные API Guidelines используют такие префиксы:

| Префикс | Обещание | Принимает | Пример |
|---|---|---|---|
| `new` | создаёт значение | — | `Rectangle::new(3, 4)` |
| `with_` | строитель: возвращает значение с одной заданной опцией | `self` | `config.with_port(80)` |
| `is_` / `has_` | вопрос «да/нет» | `&self` | `name.is_empty()` |
| `as_` | дешёвое представление данных, заимствованное | `&self` | `name.as_str()` |
| `to_` | создаёт новое значение, может стоить времени или памяти; оригинал сохраняется | `&self` | `name.to_uppercase()` |
| `into_` | преобразует и **поглощает** оригинал | `self` | `name.into_string()` |
| *(имя поля)* | геттер | `&self` | `rect.width()` |

Все они в одном небольшом типе:

```rust
struct Name {
    text: String,
}

impl Name {
    fn new(text: &str) -> Self {
        Self { text: text.to_string() }
    }
    // as_*: дёшево, заимствует
    fn as_str(&self) -> &str {
        &self.text
    }
    // to_*: создаёт новое значение (что-то стоит), self остаётся
    fn to_uppercase(&self) -> String {
        self.text.to_uppercase()
    }
    // into_*: поглощает self
    fn into_string(self) -> String {
        self.text
    }
    // is_*: вопрос «да/нет»
    fn is_empty(&self) -> bool {
        self.text.is_empty()
    }
}

fn main() {
    let n = Name::new("ferris");
    println!("{} {} {}", n.as_str(), n.to_uppercase(), n.is_empty());
    let s: String = n.into_string();
    println!("{s}");
}
```

**Что происходит:**

1. `as_str(&self) -> &str` возвращает заимствованное представление текста. Ничего не копируется.
2. `to_uppercase(&self) -> String` создаёт новую `String` — выделяется память, поэтому имя начинается с `to_`.
3. `into_string(self) -> String` отдаёт внутреннюю `String` и поглощает `Name`. Поскольку метод принимает `self`, копирование не нужно — текст просто перемещается наружу.
4. После `n.into_string()` переменная `n` перемещена. Дальше мы используем только `s`.

```text
ferris FERRIS false
ferris
```

> ⚡ **Производительность:** префикс называет цену. `as_` бесплатен, `to_` выделяет память, `into_` перемещает. Для чтения предпочитайте методы с `&self` (заимствование), потому что они никогда не копируют данные.

## 7. Методы и проектирование

> 📏 **Соглашение:** когда использовать метод, а когда свободную функцию:
> - Если основная задача функции — работать с одним значением вашего типа, сделайте её методом.
> - Если она равноправно обращается с несколькими значениями (например, `distance(a, b)` между двумя точками), свободная функция или ассоциированная функция (`Point::distance(a, b)`) может читаться лучше.
> - Делайте методы небольшими и сфокусированными (идея *единственной ответственности*: один метод — одна задача). Метод длиннее экрана обычно делает слишком много.

> 🔒 **Безопасность:** в этом уроке все поля видны любому коду, поэтому кто угодно может написать `account.balance_cents = 999_999`. Методы `deposit` и `withdraw` призваны охранять правила (баланс никогда не уходит ниже нуля), но это работает только тогда, когда посторонние не могут трогать поля. В [Уроке 04.6 — Модули, крейты и пакеты](06-modules-crates-and-packages.md) вы научитесь делать поля приватными, чтобы единственным входом оставались методы. Эта идея называется *инкапсуляцией*, и это главный способ сохранять данные корректными.

## 8. Тестирование методов

> 🧪 **Тестирование:** тестируйте тип через его публичные методы, как это делал бы пользователь. Каждый тест создаёт значение, вызывает методы и проверяет видимый результат. Проверяйте и «плохой» случай: что будет, если снимаемая сумма слишком велика?

```rust
#[derive(Debug)]
struct BankAccount {
    balance_cents: u64,
}

impl BankAccount {
    fn new() -> Self {
        Self { balance_cents: 0 }
    }
    fn deposit(&mut self, cents: u64) {
        self.balance_cents += cents;
    }
    fn withdraw(&mut self, cents: u64) -> bool {
        if cents > self.balance_cents {
            false
        } else {
            self.balance_cents -= cents;
            true
        }
    }
}

fn main() {
    let mut a = BankAccount::new();
    a.deposit(100);
    println!("{}", a.withdraw(40));
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn deposit_adds_money() {
        let mut a = BankAccount::new();
        a.deposit(500);
        assert_eq!(a.balance_cents, 500);
    }

    #[test]
    fn cannot_withdraw_more_than_balance() {
        let mut a = BankAccount::new();
        a.deposit(100);
        assert!(!a.withdraw(101));
        assert_eq!(a.balance_cents, 100); // без изменений
        assert!(a.withdraw(100));
        assert_eq!(a.balance_cents, 0);
    }
}
```

**Что происходит:**

1. `main` — всего лишь небольшая демонстрация; `cargo run` печатает `true`.
2. `cargo test` запускает два теста. Каждый создаёт свой счёт, поэтому тесты не зависят друг от друга.
3. Во втором тесте неудачное снятие должно оставить баланс неизменным (`100`) — так проверяется, что метод не делает работу наполовину.

```text
running 2 tests
..
test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

## Частые ошибки

> ⚠️ **Частая ошибка:** `&self` там, где нужен `&mut self` (`E0594`), или переменная без `mut` при вызове метода с `&mut self` (`E0596`). См. раздел 2.

> ⚠️ **Частая ошибка:** использовать значение после вызова метода, который принимает `self` (`E0382`). Посмотрите на сигнатуру метода: голый `self` означает «поглощает». Если значение всё ещё нужно, сделайте так, чтобы метод принимал `&self`, или вызовите сначала `.clone()`.

> ⚠️ **Частая ошибка:** вызов ассоциированной функции через точку (`E0599`) или метода без получателя (`E0061`). Запомните: нет `self` — значит `Type::function()`, есть `self` — значит `value.method()`.

> ⚠️ **Частая ошибка:** передать одно и то же значение дважды как `&mut`:

```rust
BankAccount::transfer(&mut a, &mut a, 10);
```

```text
error[E0499]: cannot borrow `a` as mutable more than once at a time
  --> src/main.rs:14:35
   |
14 |     BankAccount::transfer(&mut a, &mut a, 10);
   |     --------------------- ------  ^^^^^^ second mutable borrow occurs here
   |     |                     |
   |     |                     first mutable borrow occurs here
   |     first borrow later used by call
```

Это правило заимствования «одна `&mut` за раз» из [Урока 03.5](../03-ownership-and-borrowing/05-mutable-references-and-the-borrow-rules.md). Здесь оно вас защищает: перевод денег со счёта на него же был бы ошибкой.

## Попробуйте сами

1. Создайте проект:

   ```text
   cd ~/projects/rust
   cargo new code/p04-methods
   cd code/p04-methods
   ```

2. Поместите в `src/main.rs`:

   ```rust
   #[derive(Debug)]
   struct Task {
       title: String,
       done: bool,
   }

   impl Task {
       fn new(title: &str) -> Self {
           Self {
               title: String::from(title),
               done: false,
           }
       }

       fn complete(&mut self) {
           self.done = true;
       }

       fn label(&self) -> String {
           let mark = if self.done { "x" } else { " " };
           format!("[{}] {}", mark, self.title)
       }
   }

   fn main() {
       let mut task = Task::new("write lesson");
       println!("{}", task.label());
       task.complete();
       println!("{}", task.label());
   }
   ```

3. Запустите `cargo run`. Ожидаемый вывод:

   ```text
   [ ] write lesson
   [x] write lesson
   ```

4. Расширьте программу: добавьте метод `fn rename(&mut self, new_title: &str)`, добавьте модуль `#[cfg(test)]` с тестом, что `complete` устанавливает `done`, и запустите `cargo test`.
5. Сделайте коммит:

   ```text
   cd ~/projects/rust
   git add code/p04-methods
   git commit -m "feat(p04): task type with methods and tests"
   ```

## Упражнения

### Упражнение 1 — Круг (легко)

Напишите `struct Circle { radius: f64 }` с ассоциированной функцией `new(radius: f64) -> Self` и методами `area(&self) -> f64` и `circumference(&self) -> f64`. Используйте `std::f64::consts::PI`. Напишите тест для единичного круга (радиус 1.0) с допуском.

<details>
<summary>Подсказка</summary>

`use std::f64::consts::PI;` в начале файла. Площадь — `PI * r * r`; длина окружности — `2.0 * PI * r`. Числа с плавающей точкой сравнивайте так: `(a - b).abs() < 1e-9`.

</details>

<details>
<summary>Ответ</summary>

```rust
use std::f64::consts::PI;

struct Circle {
    radius: f64,
}

impl Circle {
    fn new(radius: f64) -> Self {
        Self { radius }
    }

    fn area(&self) -> f64 {
        PI * self.radius * self.radius
    }

    fn circumference(&self) -> f64 {
        2.0 * PI * self.radius
    }
}

fn main() {
    let c = Circle::new(3.0);
    println!("area = {:.3}", c.area());
    println!("circumference = {:.3}", c.circumference());
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn unit_circle() {
        let c = Circle::new(1.0);
        assert!((c.area() - PI).abs() < 1e-9);
        assert!((c.circumference() - 2.0 * PI).abs() < 1e-9);
    }
}
```

Вывод `cargo run`:

```text
area = 28.274
circumference = 18.850
```

Почему это работает: оба метода только читают радиус, поэтому достаточно `&self`. `{:.3}` печатает три знака после запятой.

</details>

### Упражнение 2 — Счётчик (средне)

Напишите `struct Counter { count: u32 }` с методами `new()`, `increment(&mut self)`, `reset(&mut self)` и `value(&self) -> u32`. Напишите тесты: новый счётчик равен 0; после двух `increment` он равен 2; после `reset` снова 0.

<details>
<summary>Подсказка</summary>

Только `value` может использовать `&self`. Остальные два изменяют счётчик, поэтому им нужен `&mut self`, а переменная в тесте должна быть `let mut`.

</details>

<details>
<summary>Ответ</summary>

```rust
#[derive(Debug)]
struct Counter {
    count: u32,
}

impl Counter {
    fn new() -> Self {
        Self { count: 0 }
    }

    fn increment(&mut self) {
        self.count += 1;
    }

    fn reset(&mut self) {
        self.count = 0;
    }

    fn value(&self) -> u32 {
        self.count
    }
}

fn main() {
    let mut c = Counter::new();
    c.increment();
    c.increment();
    println!("{}", c.value());
    c.reset();
    println!("{}", c.value());
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn starts_at_zero() {
        assert_eq!(Counter::new().value(), 0);
    }

    #[test]
    fn increment_and_reset() {
        let mut c = Counter::new();
        c.increment();
        c.increment();
        assert_eq!(c.value(), 2);
        c.reset();
        assert_eq!(c.value(), 0);
    }
}
```

`cargo run` печатает `2` и `0`; `cargo test` сообщает о двух пройденных тестах.

Почему это работает: методы, изменяющие данные, принимают `&mut self`; методы, которые только читают, принимают `&self`.

</details>

### Упражнение 3 — Исправьте ошибки компиляции (средне)

В этой программе три ошибки. Найдите их по сообщениям компилятора и исправьте. Ожидаемый вывод после исправления: `77 32`.

```rust
struct Temperature {
    celsius: f64,
}

impl Temperature {
    fn from_celsius(celsius: f64) -> Self {
        Self { celsius }
    }

    fn warm_up(&self, degrees: f64) {
        self.celsius += degrees;
    }

    fn fahrenheit(&self) -> f64 {
        self.celsius * 9.0 / 5.0 + 32.0
    }
}

fn main() {
    let t = Temperature::from_celsius(20.0);
    t.warm_up(5.0);
    let other = t.from_celsius(0.0);
    println!("{} {}", t.fahrenheit(), other.fahrenheit());
}
```

<details>
<summary>Подсказка</summary>

Первый запуск сообщает о двух ошибках: `E0599` (`from_celsius` — ассоциированная функция, её нельзя вызывать через точку) и `E0594` (`warm_up` пишет через `&self`). После их исправления компилятор попросит ещё сделать `t` изменяемой.

</details>

<details>
<summary>Ответ</summary>

```rust
struct Temperature {
    celsius: f64,
}

impl Temperature {
    fn from_celsius(celsius: f64) -> Self {
        Self { celsius }
    }

    fn warm_up(&mut self, degrees: f64) {
        self.celsius += degrees;
    }

    fn fahrenheit(&self) -> f64 {
        self.celsius * 9.0 / 5.0 + 32.0
    }
}

fn main() {
    let mut t = Temperature::from_celsius(20.0);
    t.warm_up(5.0);
    let other = Temperature::from_celsius(0.0);
    println!("{} {}", t.fahrenheit(), other.fahrenheit());
}
```

Три исправления: (1) `warm_up` принимает `&mut self`; (2) `let mut t`; (3) `Temperature::from_celsius(0.0)` вместо `t.from_celsius(0.0)`. Программа печатает `77 32` (25 °C — это 77 °F, 0 °C — это 32 °F).

Почему это работает: каждое исправление соответствует правилу этого урока: для записи нужен `&mut self`, для вызова метода с `&mut self` нужна переменная с `mut`, а ассоциированная функция вызывается через тип.

</details>

### Упражнение 4 — Перевод между счетами (сложно)

Возьмите `BankAccount { balance_cents: u64 }` с методами `new`, `deposit` и `withdraw` (возвращает `bool`). Добавьте ассоциированную функцию `transfer(from: &mut BankAccount, to: &mut BankAccount, cents: u64) -> bool`, которая переводит деньги, только если у отправителя их достаточно. Напишите два теста: успешный перевод и неудачный, который должен оставить **оба** баланса без изменений.

<details>
<summary>Подсказка</summary>

Используйте другие методы: `if from.withdraw(cents) { to.deposit(cents); true } else { false }`. В `main` вы вызываете её как `BankAccount::transfer(&mut a, &mut b, 400)` с двумя **разными** счетами.

</details>

<details>
<summary>Ответ</summary>

```rust
#[derive(Debug)]
struct BankAccount {
    balance_cents: u64,
}

impl BankAccount {
    fn new(balance_cents: u64) -> Self {
        Self { balance_cents }
    }

    fn withdraw(&mut self, cents: u64) -> bool {
        if cents > self.balance_cents {
            false
        } else {
            self.balance_cents -= cents;
            true
        }
    }

    fn deposit(&mut self, cents: u64) {
        self.balance_cents += cents;
    }

    fn transfer(from: &mut BankAccount, to: &mut BankAccount, cents: u64) -> bool {
        if from.withdraw(cents) {
            to.deposit(cents);
            true
        } else {
            false
        }
    }
}

fn main() {
    let mut a = BankAccount::new(1_000);
    let mut b = BankAccount::new(0);
    println!("{}", BankAccount::transfer(&mut a, &mut b, 400));
    println!("{}", BankAccount::transfer(&mut a, &mut b, 700));
    println!("{:?} {:?}", a, b);
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn transfer_moves_money() {
        let mut a = BankAccount::new(1_000);
        let mut b = BankAccount::new(0);
        assert!(BankAccount::transfer(&mut a, &mut b, 400));
        assert_eq!(a.balance_cents, 600);
        assert_eq!(b.balance_cents, 400);
    }

    #[test]
    fn failed_transfer_changes_nothing() {
        let mut a = BankAccount::new(100);
        let mut b = BankAccount::new(5);
        assert!(!BankAccount::transfer(&mut a, &mut b, 101));
        assert_eq!(a.balance_cents, 100);
        assert_eq!(b.balance_cents, 5);
    }
}
```

Вывод `cargo run`:

```text
true
false
BankAccount { balance_cents: 600 } BankAccount { balance_cents: 400 }
```

Почему это работает: у `transfer` нет получателя, потому что два счёта играют равные роли, поэтому ассоциированная функция читается лучше метода. Компилятор гарантирует, что `from` и `to` — разные счета: передача одной и той же переменной дважды отвергается с `E0499`. А поскольку `u64` не может уйти ниже нуля, проверка в `withdraw` ещё и защищает от арифметической паники (вычитание большего числа, чем есть).

</details>

## Вопросы для самопроверки

1. Чем отличается метод от ассоциированной функции?

<details>
<summary>Ответ</summary>

У метода первым параметром стоит `self` (в какой-либо форме), и вызывается он через точку у значения: `rect.area()`. У ассоциированной функции нет `self`, и вызывается она через тип с `::`: `Rectangle::new(3, 4)`.

</details>

2. Что означает `Self` внутри `impl Rectangle`?

<details>
<summary>Ответ</summary>

Это короткое имя типа блока `impl`, здесь — `Rectangle`. `-> Self` возвращает `Rectangle`.

</details>

3. Какую форму `self` вы выберете для метода, который изменяет поле? А для метода, который уничтожает значение?

<details>
<summary>Ответ</summary>

`&mut self` — для изменения поля (значение остаётся у вызывающего кода). `self` (по значению) — для поглощения: после этого вызывающий код больше не может пользоваться значением.

</details>

4. Почему можно писать `rect.area()`, когда `area` ожидает `&Rectangle`?

<details>
<summary>Ответ</summary>

Из-за автоматического взятия ссылки: компилятор добавляет нужный `&`, `&mut` или `*` к получателю в соответствии с сигнатурой метода. `rect.area()` означает `Rectangle::area(&rect)`.

</details>

5. Что обещают префиксы `as_`, `to_` и `into_`?

<details>
<summary>Ответ</summary>

`as_`: дешёвое заимствованное представление (принимает `&self`). `to_`: создаёт новое значение, что может стоить памяти или времени, а оригинал сохраняется (принимает `&self`). `into_`: преобразует и поглощает оригинал (принимает `self`).

</details>

6. На чём основан вызов `ServerConfig::new("x").with_port(1).with_verbose(true)`?

<details>
<summary>Ответ</summary>

Каждый метод `with_` принимает `self` по значению и возвращает `Self`, поэтому результат одного вызова становится получателем следующего. Это паттерн «строитель» с цепочкой методов.

</details>

7. Почему `Counter::new()` не требует `mut`, а `c.increment()` (с `&mut self`) требует `let mut c`?

<details>
<summary>Ответ</summary>

`Counter::new()` просто возвращает новое значение; ничего не заимствуется изменяемо. `c.increment()` неявно заимствует `c` как `&mut`, а для этого переменная должна быть объявлена изменяемой.

</details>

8. Как сделать так, чтобы `println!("{}", my_value)` работал для вашего собственного типа?

<details>
<summary>Ответ</summary>

Реализуйте для типа трейт `Display`: `impl fmt::Display for MyType { fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result { write!(f, "...") } }`. Для `{:?}` нужен `Debug`, который можно вывести через derive.

</details>

## Итоги

- Блок `impl` привязывает функции к типу. Внутри него `Self` — собственное имя типа.
- У **метода** первым стоит `self`, и вызывается он через точку; у **ассоциированной функции** нет `self`, и вызывается она как `Type::name()`.
- `&self` читает, `&mut self` изменяет, `self` поглощает. Это правила владения и заимствования, применённые к получателю.
- `new` — соглашение для конструкторов; методы `with_`, принимающие и возвращающие `self`, дают цепочки в стиле строителя; `#[derive(Default)]` и `..Default::default()` уменьшают шаблонный код.
- Ассоциированные константы (`Self::DEFAULT_PORT`) заменяют магические числа.
- `impl fmt::Display` включает `{}` и `.to_string()`; `Debug` остаётся выводимым для разработчиков.
- Называйте методы по их обещанию: `as_` (дешёвое представление), `to_` (новое значение), `into_` (поглощение), `is_` (вопрос), геттеры — по имени поля.
- Тестируйте типы через их методы, включая случаи отказа.

## Источники и что почитать

- [The Rust Programming Language — 5.3 Method Syntax](https://doc.rust-lang.org/book/ch05-03-method-syntax.html) — глава официальной книги по этой теме.
- [Rust by Example — Associated functions & Methods](https://doc.rust-lang.org/rust-by-example/fn/methods.html) — короткие запускаемые примеры.
- [The Rust Reference — Associated Items](https://doc.rust-lang.org/reference/items/associated-items.html) — точные правила для ассоциированных функций, методов и констант.
- [Rust API Guidelines — Naming (conversions)](https://rust-lang.github.io/api-guidelines/naming.html) — официальные правила, стоящие за `as_`, `to_` и `into_`.
- [std::fmt::Display](https://doc.rust-lang.org/std/fmt/trait.Display.html) — документация трейта, который используется в разделе 5.
- [std::default::Default](https://doc.rust-lang.org/std/default/trait.Default.html) — документация трейта `Default`.

---

[🏠 Начало](../START-HERE.md) · [← 04.1 Структуры](01-structs.md) · [📋 Этап 04](README.md) · [04.3 Перечисления →](03-enums.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
