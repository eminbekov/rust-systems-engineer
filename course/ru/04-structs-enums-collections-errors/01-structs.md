<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← Этап 04 — Структуры, перечисления, коллекции и ошибки](README.md) · [📋 Этап 04](README.md) · [04.2 Методы и ассоциированные функции →](02-methods-and-associated-functions.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/04-structs-enums-collections-errors/01-structs.md) <!-- nav -->

# Урок 04.1 — Структуры

> **Этап 04 — Структуры, перечисления, коллекции и ошибки** · Урок 1 из 12 · ⏱ около 4 часов (2 занятия)

## Что вы узнаете

- Что такое **структура** (struct) и зачем программам объединять связанные значения под одним именем.
- Как объявить структуру, создать её значение, прочитать и изменить поля.
- Три сокращения: краткая инициализация полей, синтаксис обновления структуры и два «лёгких» вида структур (кортежные структуры и структуры-единицы).
- Как вывести структуру для отладки с помощью `#[derive(Debug)]`, `{:?}`, `{:#?}` и `dbg!`.
- Как структуры связаны с владением, заимствованием и `Copy`, и как структура выглядит в памяти.

## Перед началом

- Вы прошли [Урок 03.5 — Изменяемые ссылки и правила заимствования](../03-ownership-and-borrowing/05-mutable-references-and-the-borrow-rules.md) и [Урок 03.3 — Перемещение, копирование и клонирование](../03-ownership-and-borrowing/03-move-copy-and-clone.md). Структуры подчиняются тем же правилам владения, что и любые другие значения.
- Вы умеете писать тесты ([Урок 02.8 — Ваши первые тесты](../02-programming-basics/08-your-first-tests.md)) и выводить текст через `println!` ([Урок 02.6 — Вывод, форматирование и комментарии](../02-programming-basics/06-printing-formatting-and-comments.md)).
- У вас есть учебный репозиторий из [Урока 01.10 — Проект: ваш учебный репозиторий и контрольная точка](../01-git-and-version-control/10-project-your-learning-repo-and-checkpoint.md). В конце урока мы сделаем небольшой коммит.

## Новые слова

| Слово | Значение |
|---|---|
| **структура** (struct) | Тип, который вы проектируете сами. Он объединяет несколько значений, у каждого из которых есть своё имя и тип, в одно значение. |
| **поле** (field) | Одна именованная часть структуры. У структуры `User` могут быть поля `username` и `email`. |
| **экземпляр** (instance) | Одно конкретное значение типа-структуры. `User` — это тип, а один конкретный пользователь — экземпляр. |
| **краткая инициализация полей** (field init shorthand) | Запись `username` вместо `username: username`, когда переменная называется так же, как поле. |
| **синтаксис обновления структуры** (struct update syntax) | Запись `..other` в конце значения структуры, чтобы взять оставшиеся поля из другого экземпляра. |
| **кортежная структура** (tuple struct) | Структура, у полей которой нет имён, только позиции: `Color(255, 0, 0)`. |
| **структура-единица** (unit-like struct) | Структура без единого поля: `struct Marker;`. Она занимает ноль байт. |
| **newtype** («новый тип») | Кортежная структура с одним полем, которая придаёт существующему типу новый смысл: `struct Meters(f64);`. |
| **атрибут** (attribute) | Пометка вида `#[...]` перед элементом программы. Она сообщает компилятору что-то об этом элементе. |
| **derive** | Атрибут (`#[derive(Debug)]`), который просит компилятор автоматически написать стандартный код для вашего типа. |
| **трейт** (trait) | Именованный набор возможностей, которые может иметь тип, например «можно вывести через `{:?}`». Определять трейты вы научитесь в [Уроке 05.2 — Трейты](../05-rust-intermediate/02-traits.md). Здесь мы лишь просим компилятор дать их нашим типам. |
| **Debug** | Трейт, на котором работает `{:?}`. Он выводит значение в удобной для разработчика форме. |
| **PartialEq** | Трейт, благодаря которому для вашего типа работает `==`. |
| **dbg!** | Макрос (особый вид вызова, который пишется с `!`), выводящий значение вместе с файлом и строкой и затем возвращающий это значение. |
| **выравнивающие байты** (padding) | Неиспользуемые байты, которые компилятор добавляет внутрь структуры, чтобы каждое поле лежало по удобному ему адресу. |
| **FFI** (Foreign Function Interface, интерфейс внешних функций) | Способ, которым код на Rust вызывает код на другом языке, например на C. См. [Урок 06.10 — FFI: вызов кода на C](../06-rust-advanced/10-ffi-calling-c.md). |
| **UpperCamelCase** | Стиль имён, где каждое слово начинается с заглавной буквы и нет подчёркиваний: `UserAccount`. В Rust так называют типы. |
| **std** (standard library, стандартная библиотека) | Библиотека, которая поставляется вместе с Rust. `std::mem::size_of` означает «функция `size_of` в модуле `mem` стандартной библиотеки». |
| **stderr** (standard error, стандартный поток ошибок) | Канал вывода, которым программы пользуются для диагностики и сообщений об ошибках, отдельный от обычного вывода. |

## 1. Зачем нужны структуры

Представьте, что вы пишете программу, которая работает с пользователями. У каждого пользователя есть имя, адрес электронной почты, признак «учётная запись активна» и число входов. Без структур вам пришлось бы таскать за собой четыре отдельные переменные:

```rust
let username = String::from("alice");
let email = String::from("alice@example.com");
let active = true;
let login_count = 1;
```

Теперь вы хотите передать «пользователя» в функцию. Понадобятся четыре параметра. Второй пользователь — ещё четыре переменные. Очень легко перепутать: имя `alice` и почту `bob`.

**Структура** решает эту проблему. Она похожа на бумажную анкету с подписанными клетками: одна анкета — много именованных клеток. Вы один раз придумываете бланк (это *тип*), а затем заполняете сколько угодно его копий (это *экземпляры*).

## 2. Объявление и создание структуры

```rust
struct User {
    username: String,
    email: String,
    active: bool,
    login_count: u32,
}

fn main() {
    let user = User {
        username: String::from("alice"),
        email: String::from("alice@example.com"),
        active: true,
        login_count: 1,
    };

    println!("{} <{}>", user.username, user.email);
    println!("active: {}, logins: {}", user.active, user.login_count);
}
```

**Что происходит:**

1. `struct User { ... }` объявляет новый тип `User`. В фигурных скобках перечислены поля: имя, двоеточие, тип и запятая после каждого.
2. В `main` мы создаём экземпляр. Пишем имя типа, а в скобках — значение для **каждого** поля в виде `имя: значение`. Порядок не важен, но пропускать поля нельзя.
3. `user.username` читает поле `username` экземпляра `user`. Точка — это способ добраться до содержимого структуры.
4. Последние две строки выводят четыре разных поля одного значения.

```text
alice <alice@example.com>
active: true, logins: 1
```

> 📏 **Соглашение:** имена типов (`User`) пишутся в `UpperCamelCase`, имена полей (`login_count`) — в `snake_case`. Если нарушить это правило, компилятор и `cargo clippy` выдадут предупреждение.

### Изменение полей

Экземпляр структуры либо целиком изменяемый, либо целиком неизменяемый. Пометить `mut` только одно поле нельзя. Решает привязка переменной целиком:

```rust
struct User {
    username: String,
    email: String,
    active: bool,
    login_count: u32,
}

fn main() {
    let mut user = User {
        username: String::from("alice"),
        email: String::from("alice@example.com"),
        active: true,
        login_count: 1,
    };

    user.login_count += 1;
    user.email = String::from("alice@new-mail.example");

    println!("{} <{}>", user.username, user.email);
    println!("active: {}, logins: {}", user.active, user.login_count);
}
```

**Что происходит:**

1. `let mut user` делает изменяемым весь экземпляр.
2. `user.login_count += 1;` прибавляет единицу к одному полю.
3. `user.email = ...` заменяет поле типа `String`. Старая `String` в этот момент уничтожается (её память освобождается) — об этом рассказывал [Урок 03.2 — Правила владения](../03-ownership-and-borrowing/02-ownership-rules.md).

```text
alice <alice@new-mail.example>
active: true, logins: 2
```

Если забыть `mut`, компилятор вас остановит:

```text
error[E0594]: cannot assign to `user.login_count`, as `user` is not declared as mutable
  --> src/main.rs:11:5
   |
11 |     user.login_count += 1;
   |     ^^^^^^^^^^^^^^^^^^^^^ cannot assign
   |
help: consider changing this to be mutable
   |
 7 |     let mut user = User {
   |         +++
```

Сообщение говорит: вы присваиваете значение полю, но переменная `user` не изменяемая. Исправление — то, что показывает компилятор: `let mut user`.

## 3. Краткая инициализация полей

Функции, которые строят структуру, часто имеют параметры с теми же именами, что и поля. Повторять имя дважды — это шум:

```rust
User { username: username, email: email, active: true, login_count: 0 }
```

Если переменная и поле называются одинаково, имя можно написать один раз:

```rust
struct User {
    username: String,
    email: String,
    active: bool,
    login_count: u32,
}

fn build_user(username: String, email: String) -> User {
    User {
        username,
        email,
        active: true,
        login_count: 0,
    }
}

fn main() {
    let user = build_user(String::from("bob"), String::from("bob@example.com"));
    println!("{} <{}> active={} logins={}", user.username, user.email, user.active, user.login_count);
}
```

**Что происходит:**

1. `build_user` принимает две `String`. Владение обеими строками переходит в функцию.
2. `User { username, email, ... }` — сокращение для `username: username, email: email`. Две `String` перемещаются ещё раз, теперь в новую структуру.
3. Последнее выражение функции (без точки с запятой) — это возвращаемый `User`.

```text
bob <bob@example.com> active=true logins=0
```

> 💡 **Совет:** такая функция, как `build_user`, называется *конструктором*. В [Уроке 04.2 — Методы и ассоциированные функции](02-methods-and-associated-functions.md) вы узнаете стандартный для Rust способ писать конструкторы (`User::new(...)`).

## 4. Синтаксис обновления структуры

Часто нужен новый экземпляр, почти такой же, как старый. Напишите поля, которые отличаются, а затем `..old`, чтобы скопировать остальные:

```rust
struct User {
    username: String,
    email: String,
    active: bool,
    login_count: u32,
}

fn main() {
    let user1 = User {
        username: String::from("alice"),
        email: String::from("alice@example.com"),
        active: true,
        login_count: 5,
    };

    let user2 = User {
        email: String::from("alice2@example.com"),
        ..user1
    };

    println!("{} <{}> logins={}", user2.username, user2.email, user2.login_count);
    // user1.active — это bool (Copy), поэтому его по-прежнему можно использовать:
    println!("user1 still active? {}", user1.active);
}
```

**Что происходит:**

1. У `user2` свой собственный `email`. Все остальные поля (`username`, `active`, `login_count`) берутся из `user1`. `..user1` должно стоять последним, и запятой после него нет.
2. Взять поле — значит **переместить или скопировать** его, как в `let a = b;`. `username` — это `String`, поэтому она *перемещается* из `user1`. `active` и `login_count` — простые числа и булевы значения с трейтом `Copy`, поэтому они копируются.
3. Поскольку `user1.username` перемещено, `user1` целиком больше использовать нельзя. Но поля, которые только копировались, остаются доступны, поэтому `user1.active` работает.

```text
alice <alice2@example.com> logins=5
user1 still active? true
```

Если попытаться прочитать перемещённое поле, вы получите знакомую ошибку владения:

```text
error[E0382]: borrow of moved value: `user1.username`
  --> src/main.rs:17:20
   |
13 |       let user2 = User {
   |  _________________-
14 | |         email: String::from("other@example.com"),
15 | |         ..user1
16 | |     };
   | |_____- value moved here
17 |       println!("{}", user1.username);
   |                      ^^^^^^^^^^^^^^ value borrowed here after move
   |
   = note: move occurs because `user1.username` has type `String`, which does not implement the `Copy` trait
```

Примечание внизу объясняет всё: `String` не реализует `Copy`, поэтому `..user1` её переместил. Чтобы сохранить `user1.username`, задайте `user2` собственный `username` или воспользуйтесь `.clone()` (см. [Урок 03.3](../03-ownership-and-borrowing/03-move-copy-and-clone.md)).

> ⚡ **Производительность:** *перемещение* `String` копирует только её маленький заголовок (указатель, длину и ёмкость), а не текст в куче. Поэтому синтаксис обновления структуры дёшев. `.clone()` копирует текст из кучи и стоит дороже.

## 5. Кортежные структуры, newtype и структуры-единицы

Иногда имена полей ничего не добавляют. В Rust есть два лёгких вида структур.

У **кортежной структуры** поля определяются позицией:

```rust
struct Color(u8, u8, u8);
struct Meters(f64);
struct Marker;

fn main() {
    let red = Color(255, 0, 0);
    let length = Meters(12.5);
    let _marker = Marker;

    println!("red = ({}, {}, {})", red.0, red.1, red.2);
    println!("length = {} m", length.0);

    let Color(r, g, b) = red;
    println!("r={r} g={g} b={b}");
}
```

**Что происходит:**

1. `Color(u8, u8, u8)` объявляет тип с тремя безымянными полями. Создаётся он как вызов функции: `Color(255, 0, 0)`.
2. Поле читается через `.0`, `.1`, `.2` (позиция, счёт с нуля).
3. У `Meters(f64)` одно поле. Такой приём называется **newtype**. `Meters(12.5)` и обычный `f64` — разные типы, поэтому вы не сложите метры с секундами по ошибке.
4. `struct Marker;` не имеет полей. Это **структура-единица**: она не несёт данных, но остаётся отдельным типом. Пригодится позже, когда тип нужен только для того, чтобы привязать к нему поведение.
5. `let Color(r, g, b) = red;` разбирает кортежную структуру на три переменные. Это *деструктуризация*; в разделе 9 будет больше примеров.

```text
red = (255, 0, 0)
length = 12.5 m
r=255 g=0 b=0
```

> 💡 **Совет:** используйте newtype, когда два значения имеют один и тот же «сырой» тип, но разный смысл: `UserId(u64)` и `OrderId(u64)`. Компилятор не даст передать номер заказа туда, где ждут номер пользователя. Такую ошибку трудно найти без помощи.

## 6. Вывод структуры: `Debug`

Попробуйте вывести только что объявленную структуру:

```rust
struct Point {
    x: i32,
    y: i32,
}

fn main() {
    let p = Point { x: 1, y: 2 };
    println!("{:?}", p);
}
```

```text
error[E0277]: `Point` doesn't implement `Debug`
 --> src/main.rs:8:22
  |
8 |     println!("{:?}", p);
  |               ----   ^ `Point` cannot be formatted using `{:?}` because it doesn't implement `Debug`
  |               |
  |               required by this formatting parameter
  |
  = help: the trait `Debug` is not implemented for `Point`
help: consider annotating `Point` with `#[derive(Debug)]`
  |
1 + #[derive(Debug)]
2 | struct Point {
  |
```

Компилятор не знает, как вы хотите видеть `Point`. Для `{:?}` нужна возможность **`Debug`** (это *трейт*). Попросить компилятор написать её за вас можно с помощью **атрибута** `derive`:

```rust
#[derive(Debug)]
struct Point {
    x: i32,
    y: i32,
}

#[derive(Debug)]
struct Player {
    name: String,
    position: Point,
    health: u8,
}

fn main() {
    let p = Player {
        name: String::from("Zed"),
        position: Point { x: 3, y: -4 },
        health: 100,
    };
    println!("{:?}", p);
    println!("{:#?}", p);
}
```

**Что происходит:**

1. `#[derive(Debug)]` стоит прямо над структурой. Он говорит компилятору: «сгенерируй код, который выводит этот тип в отладочном виде». Тип каждого поля тоже должен поддерживать `Debug`, поэтому он есть и у `Point`.
2. `Player` содержит `Point`. Структуры могут хранить другие структуры — так из маленьких частей строятся большие модели.
3. `{:?}` выводит всё в одну строку.
4. `{:#?}` («красивый debug») выводит значение в несколько строк с отступами. Большие значения так читать легче.

```text
Player { name: "Zed", position: Point { x: 3, y: -4 }, health: 100 }
Player {
    name: "Zed",
    position: Point {
        x: 3,
        y: -4,
    },
    health: 100,
}
```

> 💡 **Совет:** пока вы пишете программу, добавляйте `Debug` почти ко всем структурам. Это ничего не стоит во время выполнения и сильно экономит время. Если в учебной программе cargo выводит `warning: fields ... are never read`, это значит лишь, что программа пока не использует эти поля; в выводах этого урока такие предупреждения опущены.

`{:?}` предназначен для разработчика. Для текста, который видят пользователи, вы пишете форматирование сами; это мы сделаем в [Уроке 04.2](02-methods-and-associated-functions.md).

### `dbg!`

`dbg!(выражение)` выводит выражение, его значение, а также файл и строку в **поток ошибок** (`stderr` — канал, который программы используют для диагностики, отдельно от обычного вывода). Затем он возвращает значение, поэтому его можно обернуть вокруг любого выражения:

```rust
#[derive(Debug)]
struct Rectangle {
    width: u32,
    height: u32,
}

fn main() {
    let scale = 3;
    let rect = Rectangle {
        width: dbg!(10 * scale),
        height: 20,
    };
    dbg!(&rect);
}
```

**Что происходит:**

1. `dbg!(10 * scale)` вычисляет `30`, печатает строку в `stderr` и возвращает `30`, так что `width` становится `30`.
2. `dbg!(&rect)` получает *ссылку*. Если бы мы написали `dbg!(rect)`, макрос забрал бы владение, и `rect` переместился бы внутрь него.

```text
[src/main.rs:10:16] 10 * scale = 30
[src/main.rs:13:5] &rect = Rectangle {
    width: 30,
    height: 20,
}
```

Числа после имени файла — это строка и столбец.

> 📏 **Соглашение:** `dbg!` — временный инструмент. Удалите его перед коммитом. `println!` нужен для вывода, который является частью программы.

## 7. Структуры, владение и заимствование

Структура **владеет** своими полями. Когда структура выходит из области видимости, все её поля уничтожаются. Поэтому для текстовых полей мы брали `String`, а не `&str`: сама структура удерживает текст в живых.

А если попробовать заимствованное поле?

```rust
struct Article {
    title: &str,
    words: u32,
}
```

```text
error[E0106]: missing lifetime specifier
 --> src/main.rs:2:12
  |
2 |     title: &str,
  |            ^ expected named lifetime parameter
  |
help: consider introducing a named lifetime parameter
  |
1 ~ struct Article<'a> {
2 ~     title: &'a str,
  |
```

Ссылка внутри структуры должна гарантированно оставаться действительной всё время жизни структуры. Чтобы это проверить, компилятору нужна аннотация *времени жизни* (`'a`). Времена жизни — отдельная большая тема в [Уроке 05.3 — Времена жизни](../05-rust-intermediate/03-lifetimes.md). А пока держитесь простого правила:

> 📏 **Соглашение:** в структурах используйте **владеющие** типы (`String`, `Vec<T>`, числа). Поля `&str` появятся после того, как вы изучите времена жизни.

### Передача структур в функции

Передача структуры по значению *перемещает* её, как и любое другое значение:

```rust
#[derive(Debug)]
struct Point {
    x: i32,
    y: i32,
}

fn show(p: Point) {
    println!("({}, {})", p.x, p.y);
}

fn main() {
    let a = Point { x: 1, y: 2 };
    show(a);
    println!("{:?}", a);
}
```

```text
error[E0382]: borrow of moved value: `a`
  --> src/main.rs:14:22
   |
12 |     let a = Point { x: 1, y: 2 };
   |         - move occurs because `a` has type `Point`, which does not implement the `Copy` trait
13 |     show(a);
   |          - value moved here
14 |     println!("{:?}", a);
   |                      ^ value borrowed here after move
   |
note: consider changing this parameter type in function `show` to borrow instead if owning the value isn't necessary
```

Есть два способа исправить. Обычный — **заимствовать**: принимать в функции `&Point` и вызывать `show(&a)`. Ссылка позволяет функции посмотреть на значение, не забирая владение. Если функции нужно изменить структуру, принимайте `&mut Point`:

```rust
struct Rectangle {
    width: u32,
    height: u32,
}

fn area(rect: &Rectangle) -> u32 {
    rect.width * rect.height
}

fn grow(rect: &mut Rectangle, by: u32) {
    rect.width += by;
    rect.height += by;
}

fn main() {
    let mut rect = Rectangle { width: 30, height: 50 };
    println!("area = {}", area(&rect));
    grow(&mut rect, 10);
    println!("area after grow = {}", area(&rect));
}
```

**Что происходит:**

1. `area` заимствует прямоугольник только для чтения (`&Rectangle`). После вызова `main` по-прежнему владеет `rect`.
2. `grow` заимствует его изменяемо (`&mut Rectangle`) и меняет оба поля через ссылку. Это следует правилам заимствования из [Урока 03.5](../03-ownership-and-borrowing/05-mutable-references-and-the-borrow-rules.md): пока существует `&mut`, больше никто не может использовать `rect`.
3. В месте вызова `main` обязан написать `&mut rect`, поэтому изменение видно прямо в вызывающем коде.

```text
area = 1500
area after grow = 2400
```

Второй способ — сделать структуру **`Copy`**. Небольшая структура, состоящая только из `Copy`-полей, может на это согласиться:

```rust
#[derive(Debug, Clone, Copy)]
struct Point {
    x: i32,
    y: i32,
}

fn show(p: Point) {
    println!("({}, {})", p.x, p.y);
}

fn main() {
    let a = Point { x: 1, y: 2 };
    let b = a; // скопировано, а не перемещено
    show(a);
    show(b);
    println!("{:?}", a);
}
```

**Что происходит:**

1. `#[derive(Clone, Copy)]` просит компилятор сделать тип копируемым. Это разрешено, только когда все поля — `Copy` (здесь два `i32`). Структура со `String` никогда не может быть `Copy`.
2. `let b = a;` и `show(a)` теперь копируют 8 байт вместо перемещения. `a` остаётся действительной.
3. `Copy` требует `Clone`, поэтому оба трейта всегда выводятся вместе.

```text
(1, 2)
(1, 2)
Point { x: 1, y: 2 }
```

> 📏 **Соглашение:** выводите `Copy` только для небольших типов-«простых данных»: точек, цветов, идентификаторов. Для всего, что владеет данными в куче, оставляйте семантику перемещения.

## 8. Как структура выглядит в памяти

Структура — это просто её поля, лежащие одно за другим в одном блоке памяти. Никакого скрытого заголовка и никакого указателя на поля:

```text
Rectangle { width: 30, height: 50 }      (on the stack)

 address:   +0        +4
          ┌────────┬────────┐
          │   30   │   50   │
          └────────┴────────┘
           width     height      total: 8 bytes (two u32 of 4 bytes)
```

Структура со `String` содержит заголовок `String` внутри себя, а текст лежит в куче ([Урок 03.1 — Память: стек и куча](../03-ownership-and-borrowing/01-memory-stack-and-heap.md)):

```text
User { username: String, login_count: u32 }

 stack                                         heap
┌─────────────────────────────┐
│ username.ptr  ──────────────┼──────────►  a l i c e
│ username.len  = 5           │
│ username.cap  = 5           │
│ login_count   = 1           │
└─────────────────────────────┘
```

Компилятор может добавить **выравнивающие байты** (padding) и переставить поля, чтобы каждое было выровнено. Размер можно измерить с помощью `std::mem::size_of`:

```rust
use std::mem::size_of;

struct Mixed {
    a: u8,
    b: u32,
    c: u8,
}

#[repr(C)]
struct MixedC {
    a: u8,
    b: u32,
    c: u8,
}

struct Pair(u32, u32);
struct Marker;

fn main() {
    println!("Pair   = {} bytes", size_of::<Pair>());
    println!("Marker = {} bytes", size_of::<Marker>());
    println!("Mixed  = {} bytes", size_of::<Mixed>());
    println!("MixedC = {} bytes", size_of::<MixedC>());
}
```

**Что происходит:**

1. `size_of::<T>()` возвращает, сколько байт занимает значение типа `T`. Часть `::<...>` называет тип.
2. `Pair` — это два 4-байтовых числа: 8 байт. У `Marker` нет полей: 0 байт.
3. В `Mixed` 1 + 4 + 1 = 6 байт данных. По умолчанию Rust вправе переставлять поля. Он ставит `u32` первым, а два `u8` — за ним, поэтому всего получается 8 (6 байт данных и 2 байта выравнивания).
4. `#[repr(C)]` запрещает перестановку и использует раскладку языка C в том порядке, в котором поля записаны. `u32` должен начинаться по адресу, кратному 4, поэтому после `a` идут 3 байта выравнивания и ещё 3 после `c`: итого 12 байт. `repr(C)` нужен, когда вы делите структуру с кодом на C (FFI) — см. [Урок 06.10](../06-rust-advanced/10-ffi-calling-c.md).

```text
Pair   = 8 bytes
Marker = 0 bytes
Mixed  = 8 bytes
MixedC = 12 bytes
```

> ⚡ **Производительность:** о раскладке редко приходится думать. Rust и так переставляет поля, чтобы тратить меньше места. Когда структуру создают миллионы раз (например, в большом списке), её размер имеет значение, и `size_of` покажет правду.

## 9. Деструктуризация структуры

Структуру можно разобрать на переменные в той же форме, в которой вы её строили:

```rust
struct Point {
    x: i32,
    y: i32,
}

fn main() {
    let p = Point { x: 7, y: 9 };
    let Point { x, y } = p;
    println!("x={x}, y={y}");

    let Point { x: horizontal, .. } = p;
    println!("horizontal={horizontal}");
}
```

**Что происходит:**

1. `let Point { x, y } = p;` создаёт переменные `x` и `y` со значениями одноимённых полей.
2. `x: horizontal` значит «возьми поле `x` и назови переменную `horizontal`». `..` говорит «остальные поля игнорируй».
3. В конце `p` по-прежнему можно использовать, потому что оба поля — числа с `Copy`.

```text
x=7, y=9
horizontal=7
```

Это малая часть более широкого инструмента, который называется *образцами* (patterns). Вы изучите их в [Уроке 04.5 — if let, let else и образцы](05-if-let-let-else-and-patterns.md).

## 10. Тестирование структур

> 🧪 **Тестирование:** чтобы сравнивать две структуры в `assert_eq!`, тип должен реализовывать два трейта: `PartialEq` (чтобы работал `==`) и `Debug` (чтобы тест мог показать оба значения, когда они различаются). Выведите оба.

```rust
#[derive(Debug, PartialEq)]
struct Point {
    x: i32,
    y: i32,
}

fn origin() -> Point {
    Point { x: 0, y: 0 }
}

fn main() {
    println!("{:?}", origin());
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn origin_is_zero_zero() {
        assert_eq!(origin(), Point { x: 0, y: 0 });
    }
}
```

**Что происходит:**

1. `#[derive(Debug, PartialEq)]` даёт `Point` обе возможности. `PartialEq` сравнивает поле за полем.
2. `#[cfg(test)] mod tests` — модуль, который существует только при запуске `cargo test` ([Урок 02.8](../02-programming-basics/08-your-first-tests.md)). `use super::*;` подключает в него элементы родителя.
3. `assert_eq!` вызывает `==` для двух `Point` и проваливает тест, если они различаются.

```text
running 1 test
test tests::origin_is_zero_zero ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

Без `PartialEq` тест даже не скомпилируется:

```text
error[E0369]: binary operation `==` cannot be applied to type `Point`
  --> src/main.rs:21:9
   |
21 |         assert_eq!(origin(), Point { x: 0, y: 0 });
   |         ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
   |         |
   |         Point
   |         Point
   |
note: an implementation of `PartialEq` might be missing for `Point`
help: consider annotating `Point` with `#[derive(PartialEq)]`
```

## Частые ошибки

> ⚠️ **Частая ошибка:** забыть поле при создании структуры.

```rust
let p = Point { x: 7 };
```

```text
error[E0063]: missing field `y` in initializer of `Point`
 --> src/main.rs:7:13
  |
7 |     let p = Point { x: 7 };
  |             ^^^^^ missing `y`
```

Каждому полю нужно значение. Исправление: `Point { x: 7, y: 0 }`. (Или синтаксис обновления структуры с базовым значением.)

> ⚠️ **Частая ошибка:** пытаться сделать изменяемым одно поле через `let user = ...; user.field = ...;`. Ошибка — `E0594` (см. раздел 2). Сделайте `mut` всю переменную.

> ⚠️ **Частая ошибка:** использовать структуру после того, как вы передали её в функцию по значению (`E0382`, раздел 7). Заимствуйте через `&` или выведите `Copy` для крошечных типов-«простых данных».

> ⚠️ **Частая ошибка:** `&str` в поле (`E0106`). Используйте `String`, пока не изучите времена жизни.

> ⚠️ **Частая ошибка:** вывод через `{}` или `{:?}` без реализации трейта. Для `{}` нужен `Display`, который нужно написать вручную ([Урок 04.2](02-methods-and-associated-functions.md)); для `{:?}` нужен `Debug`, который можно вывести через derive.

## Попробуйте сами

1. Создайте проект в папке курса:

   ```text
   cd ~/projects/rust
   cargo new code/p04-structs
   cd code/p04-structs
   ```

2. Замените `src/main.rs` на:

   ```rust
   #[derive(Debug)]
   struct Item {
       name: String,
       quantity: u32,
       price_cents: u32,
   }

   fn total_cents(item: &Item) -> u32 {
       item.quantity * item.price_cents
   }

   fn main() {
       let pen = Item {
           name: String::from("pen"),
           quantity: 12,
           price_cents: 150,
       };
       let notebook = Item {
           name: String::from("notebook"),
           quantity: 3,
           price_cents: 899,
       };

       for item in [&pen, &notebook] {
           println!(
               "{}: {} x {} = {} cents",
               item.name,
               item.quantity,
               item.price_cents,
               total_cents(item)
           );
       }
       println!("{:#?}", pen);
   }
   ```

3. Запустите `cargo run`. Ожидаемый вывод:

   ```text
   pen: 12 x 150 = 1800 cents
   notebook: 3 x 899 = 2697 cents
   Item {
       name: "pen",
       quantity: 12,
       price_cents: 150,
   }
   ```

4. Измените цену ручки и запустите снова. Добавьте третий `Item`, созданный с помощью синтаксиса обновления структуры (`..pen`).

> 📏 **Соглашение:** деньги хранят целым числом копеек или центов (`price_cents: u32`), а не десятичным числом вроде `f64`. Числа с плавающей точкой не могут точно представить значения вроде 0.10, и маленькие погрешности накапливаются.

5. Сохраните работу в Git ([Уроки 01.3](../01-git-and-version-control/03-your-first-repository-and-commits.md) и [01.9](../01-git-and-version-control/09-teamwork-pull-requests-and-conventions.md)):

   ```text
   cd ~/projects/rust
   git status
   git add code/p04-structs
   git commit -m "feat(p04): add Item struct demo"
   ```

   `git status` должен показать только `code/p04-structs`. Если вы видите папку `target/`, сначала добавьте строку `target/` в файл `.gitignore`: результат сборки никогда не должен попадать в Git. Сообщение коммита следует соглашению из Урока 01.9 (тип, область, короткое описание).

## Упражнения

### Упражнение 1 — Книга (легко)

Объявите структуру `Book` с полями `title: String`, `author: String`, `pages: u32` и `available: bool`. Выведите для неё `Debug`. В `main` создайте одну книгу и напечатайте её через `{:#?}`.

<details>
<summary>Подсказка</summary>

Поставьте `#[derive(Debug)]` прямо над `struct Book`. Для текстовых полей используйте `String::from("...")`.

</details>

<details>
<summary>Ответ</summary>

```rust
#[derive(Debug)]
struct Book {
    title: String,
    author: String,
    pages: u32,
    available: bool,
}

fn main() {
    let book = Book {
        title: String::from("The Rust Programming Language"),
        author: String::from("Steve Klabnik and Carol Nichols"),
        pages: 560,
        available: true,
    };
    println!("{:#?}", book);
}
```

Вывод:

```text
Book {
    title: "The Rust Programming Language",
    author: "Steve Klabnik and Carol Nichols",
    pages: 560,
    available: true,
}
```

Почему это работает: derive генерирует код вывода, а `{:#?}` использует его в многострочной форме.

</details>

### Упражнение 2 — Помощники для прямоугольника с тестами (средне)

Объявите `Rectangle { width: u32, height: u32 }` (выведите `Debug` и `PartialEq`). Напишите `fn is_square(rect: &Rectangle) -> bool` и `fn scale(rect: &Rectangle, factor: u32) -> Rectangle`. Напишите два теста: один для `is_square` (случаи «да» и «нет») и один для `scale`.

<details>
<summary>Подсказка</summary>

Обе функции принимают ссылку, поэтому вызывающий код сохраняет прямоугольник. `scale` строит и возвращает **новый** `Rectangle`. В тесте сравнивайте структуры целиком через `assert_eq!` (поэтому и нужен `PartialEq`).

</details>

<details>
<summary>Ответ</summary>

```rust
#[derive(Debug, PartialEq)]
struct Rectangle {
    width: u32,
    height: u32,
}

fn is_square(rect: &Rectangle) -> bool {
    rect.width == rect.height
}

fn scale(rect: &Rectangle, factor: u32) -> Rectangle {
    Rectangle {
        width: rect.width * factor,
        height: rect.height * factor,
    }
}

fn main() {
    let r = Rectangle { width: 4, height: 4 };
    println!("{} {:?}", is_square(&r), scale(&r, 3));
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn square_is_detected() {
        assert!(is_square(&Rectangle { width: 5, height: 5 }));
        assert!(!is_square(&Rectangle { width: 5, height: 6 }));
    }

    #[test]
    fn scale_multiplies_both_sides() {
        let r = Rectangle { width: 2, height: 3 };
        assert_eq!(scale(&r, 4), Rectangle { width: 8, height: 12 });
    }
}
```

`cargo run` печатает `true Rectangle { width: 12, height: 12 }`, а `cargo test` сообщает `2 passed`.

Почему это работает: функции только читают прямоугольник, поэтому достаточно `&Rectangle`; тест сравнивает возвращённую структуру с ожидаемой поле за полем.

</details>

### Упражнение 3 — Два newtype (средне)

Создайте newtype `Celsius(f64)` и `Fahrenheit(f64)` (выведите `Debug` и `PartialEq`). Напишите `fn to_fahrenheit(c: Celsius) -> Fahrenheit` по формуле `F = C × 9 / 5 + 32`. Проверьте тестом значения 0 → 32, 100 → 212 и −40 → −40.

<details>
<summary>Подсказка</summary>

Внутреннее число читайте как `c.0`. Результат стройте через `Fahrenheit( ... )`. Пишите `9.0` и `5.0` (десятичные литералы), потому что `c.0` — это `f64`.

</details>

<details>
<summary>Ответ</summary>

```rust
#[derive(Debug, PartialEq)]
struct Celsius(f64);

#[derive(Debug, PartialEq)]
struct Fahrenheit(f64);

fn to_fahrenheit(c: Celsius) -> Fahrenheit {
    Fahrenheit(c.0 * 9.0 / 5.0 + 32.0)
}

fn main() {
    println!("{:?}", to_fahrenheit(Celsius(100.0)));
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn known_points() {
        assert_eq!(to_fahrenheit(Celsius(0.0)), Fahrenheit(32.0));
        assert_eq!(to_fahrenheit(Celsius(100.0)), Fahrenheit(212.0));
        assert_eq!(to_fahrenheit(Celsius(-40.0)), Fahrenheit(-40.0));
    }
}
```

Программа печатает `Fahrenheit(212.0)`. Почему это работает: оба типа хранят один и тот же «сырой» `f64`, но компилятор их различает, поэтому вы не сможете передать `Fahrenheit` туда, где ждут `Celsius`. (Сравнивать `f64` через `==` здесь безопасно только потому, что именно эти результаты точны; общий подход показан в следующем упражнении.)

</details>

### Упражнение 4 — Расстояние между точками (сложно)

Объявите `Point { x: f64, y: f64 }` (выведите `Debug`, `Clone`, `Copy`). Напишите `fn distance(a: Point, b: Point) -> f64` по теореме Пифагора: квадратный корень из `dx² + dy²`. Напишите тест для треугольника 3-4-5 и тест для расстояния от точки до самой себя. Числа с плавающей точкой сравнивайте с небольшим допуском, а не через `==`.

<details>
<summary>Подсказка</summary>

У `f64` есть метод `.sqrt()`. Для сравнения пишите `(d - 5.0).abs() < 1e-9`, где `1e-9` — это 0.000000001. Поскольку `Point` — `Copy`, вызов `distance(p, p)` компилируется, хотя `p` используется дважды.

</details>

<details>
<summary>Ответ</summary>

```rust
#[derive(Debug, Clone, Copy)]
struct Point {
    x: f64,
    y: f64,
}

fn distance(a: Point, b: Point) -> f64 {
    let dx = a.x - b.x;
    let dy = a.y - b.y;
    (dx * dx + dy * dy).sqrt()
}

fn main() {
    let a = Point { x: 0.0, y: 0.0 };
    let b = Point { x: 3.0, y: 4.0 };
    println!("{}", distance(a, b));
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn three_four_five() {
        let d = distance(Point { x: 0.0, y: 0.0 }, Point { x: 3.0, y: 4.0 });
        assert!((d - 5.0).abs() < 1e-9);
    }

    #[test]
    fn distance_to_itself_is_zero() {
        let p = Point { x: 1.5, y: -2.0 };
        assert!(distance(p, p).abs() < 1e-9);
    }
}
```

Программа печатает `5`, оба теста проходят.

Почему это работает: компьютер хранит `f64` с крошечными ошибками округления, поэтому два результата, равных в математике, могут отличаться в последних битах. Проверка того, что разность меньше допуска, — стандартное решение.

</details>

## Вопросы для самопроверки

1. Чем отличается тип-структура от её экземпляра?

<details>
<summary>Ответ</summary>

Тип (`struct User { ... }`) — это проект: он перечисляет поля и их типы. Экземпляр — одно конкретное значение, созданное по этому проекту, например один конкретный пользователь с реальными данными.

</details>

2. Можно ли сделать изменяемым только одно поле экземпляра структуры?

<details>
<summary>Ответ</summary>

Нет. Изменяемость принадлежит привязке: `let mut user` делает изменяемыми все поля, `let user` — ни одного.

</details>

3. Что делает `..user1` в `User { email: new_email, ..user1 }` и что может пойти не так потом?

<details>
<summary>Ответ</summary>

Оно заполняет все явно не записанные поля значениями из `user1`. Каждое поле перемещается или копируется. Поля типа без `Copy` (например, `String`) перемещаются, поэтому `user1` целиком больше использовать нельзя, и перемещённые поля читать тоже нельзя.

</details>

4. В коде упражнения `let custom = Config { name: String::from("custom"), ..base };` компилируется, и `base` всё ещё можно вывести, если у `base` поля `name: String`, `verbose: bool`, `retries: u32`. Почему?

<details>
<summary>Ответ</summary>

`name` задано явно, поэтому из `base` оно не берётся. Остальные поля (`verbose` и `retries`) — `Copy`, поэтому они копируются, а не перемещаются. Из `base` ничего не было перемещено, и оно остаётся полностью пригодным. (Если бы из `base` взяли поле `String`, перемещение сделало бы `base` непригодной.)

</details>

5. Почему пока мы используем в полях структур `String`, а не `&str`?

<details>
<summary>Ответ</summary>

Структура должна владеть тем, что хранит, иначе компилятору нужно сообщить, как долго живут заимствованные данные (время жизни, `E0106`). `String` владеющая, поэтому время жизни не нужно. Времена жизни будут в Уроке 05.3.

</details>

6. Что такое newtype и чем он полезен?

<details>
<summary>Ответ</summary>

Newtype — это кортежная структура с одним полем, например `struct UserId(u64);`. Она делает новый тип из существующего, поэтому компилятор различает значения с разным смыслом (`UserId` — это не `OrderId`), и это ничего не стоит во время выполнения.

</details>

7. Какие два derive нужны, чтобы сравнивать структуры в `assert_eq!`?

<details>
<summary>Ответ</summary>

`PartialEq` (чтобы работал `==`) и `Debug` (чтобы сообщение об ошибке могло показать оба значения).

</details>

8. Каков размер `struct Marker;` и зачем такой тип нужен?

<details>
<summary>Ответ</summary>

0 байт. Он не несёт данных, но остаётся отдельным типом, что полезно, когда тип нужен лишь для того, чтобы привязать к нему поведение или смысл.

</details>

## Итоги

- **Структура** объединяет именованные поля разных типов в один тип. Экземпляры создаются как `Name { field: value, ... }`, поля читаются через точку.
- Изменяемость принадлежит экземпляру целиком: `let mut` либо ничего.
- **Краткая инициализация полей** (`User { username, email }`) и **синтаксис обновления структуры** (`..other`) уменьшают повторы; синтаксис обновления перемещает поля без `Copy`.
- **Кортежные структуры**, **newtype** и **структуры-единицы** — лёгкие формы; newtype придаёт один и тот же «сырой» тип разным смыслам.
- `#[derive(Debug)]` вместе с `{:?}`, `{:#?}` и `dbg!` — ваш первый инструмент отладки; `#[derive(PartialEq)]` включает `==` и `assert_eq!`.
- Структура владеет своими полями; пока используйте в полях владеющие типы, передавайте в функции `&Struct` / `&mut Struct` и выводите `Clone, Copy` только для небольших типов-«простых данных».
- В памяти структура — это её поля одно за другим; Rust может переставить их и добавить выравнивающие байты, а `#[repr(C)]` фиксирует порядок языка C.

## Источники и что почитать

- [The Rust Programming Language — 5.1 Defining and Instantiating Structs](https://doc.rust-lang.org/book/ch05-01-defining-structs.html) — глава официальной книги по этой теме.
- [The Rust Programming Language — 5.2 An Example Program Using Structs](https://doc.rust-lang.org/book/ch05-02-example-structs.html) — пошаговый пример с прямоугольником, в котором добавляются `Debug` и `dbg!`.
- [Rust by Example — Structures](https://doc.rust-lang.org/rust-by-example/custom_types/structs.html) — короткие запускаемые примеры всех трёх видов структур.
- [The Rust Reference — Struct types](https://doc.rust-lang.org/reference/types/struct.html) — точные правила, включая раскладку и `repr`.
- [Rust API Guidelines — Naming](https://rust-lang.github.io/api-guidelines/naming.html) — официальные соглашения об именах, которые используются в этом курсе.
- [std::mem::size_of](https://doc.rust-lang.org/std/mem/fn.size_of.html) — документация функции из раздела 8.

---

[🏠 Начало](../START-HERE.md) · [← Этап 04 — Структуры, перечисления, коллекции и ошибки](README.md) · [📋 Этап 04](README.md) · [04.2 Методы и ассоциированные функции →](02-methods-and-associated-functions.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
