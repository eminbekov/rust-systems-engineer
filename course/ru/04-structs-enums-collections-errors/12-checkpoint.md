<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 04.11 Проект: телефонная книга](11-project-phone-book.md) · [📋 Этап 04](README.md) · [Этап 05 — Rust: средний уровень →](../05-rust-intermediate/README.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/04-structs-enums-collections-errors/12-checkpoint.md) <!-- nav -->

# Урок 04.12 — Контрольная точка

> **Этап 04 — Структуры, перечисления, коллекции и ошибки** · Урок 12 из 12 · ⏱ около 3 часов (2 занятия)

## Что вы узнаете

- Какие части Этапа 04 вы уже знаете хорошо, а какие нужно повторить.
- Как честно проверять себя: сначала напишите ответ, потом откройте скрытый.
- Как объединить структуры, перечисления, модули, коллекции и ошибки в трёх коротких программах, написанных по спецификации и протестированных.
- Как самостоятельно выполнить полный рабочий процесс Git с веткой, конфликтом слияния и тегом.

## Перед началом

- Вы прошли [Урок 04.11 — Проект: телефонная книга](11-project-phone-book.md) и все уроки этого этапа перед ним.
- Отвечайте на вопросы **по памяти**: без конспектов и без компилятора. Сначала запишите каждый ответ на бумаге или в текстовом файле и только потом открывайте скрытый ответ. Контрольная точка помогает, только если вы пытаетесь вспомнить до того, как заглянуть в ответ. (Исследователи называют это *практикой извлечения из памяти*; она работает гораздо лучше повторного чтения.)
- В практических заданиях можно пользоваться компилятором и прошлыми уроками, но не ответами.
- Создайте проект для заданий: `cd ~/projects/rust && cargo new code/p04-checkpoint && cd code/p04-checkpoint`. Для нескольких программ используйте `src/bin/`: например, положите задание 1 в `src/bin/shapes.rs` и запускайте его командой `cargo run --bin shapes`; тестируйте командой `cargo test --bin shapes`.

## Новые слова

| Слово | Значение |
|---|---|
| **контрольная точка** (checkpoint) | Пауза в конце этапа, чтобы проверить, что вы выучили, прежде чем идти дальше. |
| **практика извлечения** (retrieval practice) | Обучение через попытку вспомнить ответ по памяти вместо повторного чтения. |
| **регрессия** (regression) | Баг, при котором то, что раньше работало, перестаёт работать после изменения. Тесты нужны, чтобы ловить регрессии. |
| **конфликт слияния** (merge conflict) | Ситуация, когда Git не может автоматически соединить два изменения, потому что обе правили одни и те же строки. |
| **маркеры конфликта** (conflict markers) | Строки `<<<<<<<`, `=======` и `>>>>>>>`, которые Git пишет в файл, чтобы показать две конкурирующие версии. |

## Вопросы для самопроверки

Двадцать вопросов. Хороший результат — 16 и больше верных ответов. Для каждого неверного ответа
таблица в разделе «Если что-то не получилось» подскажет, какой урок перечитать.

### Структуры и методы

1. Чем обычная структура отличается от кортежной структуры? Приведите по одному примеру.

<details>
<summary>Ответ</summary>

У обычной структуры поля именованные: `struct User { name: String, age: u32 }`. У кортежной
структуры поля без имён, только с позициями: `struct Meters(f64);`, а поле читается через `.0`.
Кортежная структура полезна для *newtype* — нового типа вокруг существующего.

</details>

2. Что делает эта программа? Какая строка не компилируется и почему?

```rust
struct User {
    username: String,
    email: String,
    logins: u32,
}

fn main() {
    let a = User {
        username: String::from("ali"),
        email: String::from("ali@example.com"),
        logins: 3,
    };
    let b = User { email: String::from("new@example.com"), ..a };
    println!("{} {} {}", b.username, b.email, b.logins);
    println!("{}", a.email);
    println!("{}", a.logins);
    println!("{}", a.username);
}
```

<details>
<summary>Ответ</summary>

Синтаксис обновления структуры `..a` копирует поля, которые не были перечислены. `username` — это
`String`, поэтому она **перемещается** из `a` в `b`; `email` задан явно, поэтому `a.email` не тронут;
`logins` — `u32` и копируется. Первые три строки `println!` в порядке. Последняя не
компилируется:

```text
error[E0382]: borrow of moved value: `a.username`
```

Можно по-прежнему использовать `a.email` и `a.logins`, но не `a.username` и не `a` целиком.

</details>

3. Чем отличаются `&self`, `&mut self` и `self` в методе? Как вызывается *ассоциированная функция*, например `new`?

<details>
<summary>Ответ</summary>

`&self` заимствует значение, чтобы прочитать его. `&mut self` заимствует, чтобы изменить. `self`
забирает владение: значение потребляется (перемещается в метод), и вызывающий код после этого не
может им пользоваться. У ассоциированной функции нет параметра `self`; её вызывают у типа через `::`,
например `Counter::new()`, тогда как метод вызывают у значения через точку: `counter.add(5)`.

</details>

4. Что напечатает эта программа?

```rust
struct Counter {
    value: u32,
}

impl Counter {
    fn new() -> Counter {
        Counter { value: 0 }
    }

    fn add(&mut self, n: u32) {
        self.value += n;
    }

    fn doubled(&self) -> u32 {
        self.value * 2
    }

    fn finish(self) -> u32 {
        self.value + 100
    }
}

fn main() {
    let mut c = Counter::new();
    c.add(5);
    c.add(2);
    println!("{}", c.doubled());
    println!("{}", c.finish());
}
```

<details>
<summary>Ответ</summary>

```text
14
107
```

`add` вызывается дважды, поэтому `value` равно 7. `doubled` читает его: 14. `finish` потребляет
счётчик: 7 + 100 = 107. После `c.finish()` переменную `c` больше использовать нельзя.

</details>

### Перечисления, `Option` и `match`

5. Почему перечисление с данными в вариантах часто лучше структуры с несколькими полями `Option` и флагами?

<details>
<summary>Ответ</summary>

Перечисление делает недопустимые комбинации невозможными: значение — ровно один вариант, и каждый
вариант несёт ровно те данные, которые ему нужны (`Shape::Circle { radius }`, `Shape::Square(side)`).
В структуре с необязательными полями можно построить противоречивые состояния (нет ни радиуса, ни
стороны, или есть оба), и их приходится проверять вручную.

</details>

6. Какую ошибку выдаёт этот код и от чего она вас защищает?

```rust
enum Light {
    Red,
    Yellow,
    Green,
}

fn main() {
    let l = Light::Red;
    let text = match l {
        Light::Red => "stop",
        Light::Green => "go",
    };
    println!("{text}");
    let _ = Light::Yellow;
}
```

<details>
<summary>Ответ</summary>

```text
error[E0004]: non-exhaustive patterns: `Light::Yellow` not covered
```

`match` должен быть *исчерпывающим*: он обязан обработать каждый вариант. Это защищает от
забытого случая, а когда позже вы добавите новый вариант, компилятор покажет каждый `match`,
который нужно обновить.

</details>

7. В Rust нет `null`. Что вы используете для «значение может быть», и какими тремя способами безопасно достать значение?

<details>
<summary>Ответ</summary>

`Option<T>`: `Some(value)` или `None`. Безопасные способы: `match` (обработать оба случая),
`if let Some(x) = opt { ... }` (действовать, только если значение есть) и
`unwrap_or(default)` / `unwrap_or_default()` (взять значение по умолчанию). Возможны также
`let Some(x) = opt else { return ...; };` и `?` в функции, возвращающей `Option`. `unwrap()` и
`expect()` паникуют на `None`, поэтому используйте их, только когда `None` невозможен.

</details>

8. Что напечатает эта программа?

```rust
fn describe(n: i32) -> String {
    match n {
        i32::MIN..=-1 => String::from("negative"),
        0 => String::from("zero"),
        x if x % 2 == 0 => format!("even {x}"),
        1..=9 => String::from("small odd"),
        other => format!("big odd {other}"),
    }
}

fn main() {
    for n in [-5, 0, 4, 7, 21] {
        println!("{n}: {}", describe(n));
    }
}
```

<details>
<summary>Ответ</summary>

```text
-5: negative
0: zero
4: even 4
7: small odd
21: big odd 21
```

Ветки проверяются сверху вниз, и побеждает первая подходящая. `4` не отрицательно и не ноль, а
условие `x % 2 == 0` истинно, поэтому получается «even 4». `7` не проходит условие и попадает в
`1..=9`. `21` падает в последнюю ветку, которая связывает значение с именем `other`.

</details>

### `if let`, `let else` и образцы

9. Что делает `let Some(c) = text.chars().next() else { return None; };`? Когда это лучше `match`?

<details>
<summary>Ответ</summary>

Если справа `Some(c)`, создаётся переменная `c`, и код продолжается. Иначе выполняется блок
`else`, и он обязан покинуть текущий путь (`return`, `break`, `continue` или `panic!`). Это лучше
`match`, когда вы хотите, чтобы «счастливый путь» оставался без отступа, а случай сбоя — быстрый
выход (*ранний возврат*).

</details>

10. Что напечатает эта программа?

```rust
struct Point {
    x: i32,
    y: i32,
}

fn main() {
    let data = ((1, 2), Point { x: 10, y: 20 });
    let ((a, _), Point { x, .. }) = data;
    println!("{a} {x}");

    let pair = (3, -3);
    match pair {
        (0, y) => println!("on the y axis at {y}"),
        (x, 0) => println!("on the x axis at {x}"),
        (x, y) if x + y == 0 => println!("opposites"),
        _ => println!("something else"),
    }
}
```

<details>
<summary>Ответ</summary>

```text
1 10
opposites
```

Образец берёт первое число внутреннего кортежа (`_` игнорирует второе) и `x` из `Point` (`..`
игнорирует остальные поля). `match` пробует ветки по порядку: `(3, -3)` не подходит ни к `(0, y)`, ни
к `(x, 0)`, а условие третьей ветки истинно.

</details>

### Модули, крейты и пакеты

11. Чем отличаются пакет, крейт и модуль?

<details>
<summary>Ответ</summary>

*Модуль* — пространство имён внутри крейта (`mod something`), которое группирует код и управляет
видимостью. *Крейт* — одна единица компиляции: бинарный (есть `main`) или библиотечный
(`lib.rs`). *Пакет* — то, чем управляет Cargo: один `Cargo.toml` с одним или несколькими крейтами
(не более одного библиотечного и любым числом бинарных).

</details>

12. Какое сообщение выдаёт компилятор для этого кода и как это исправить?

```rust
mod bank {
    pub fn deposit() -> u32 {
        secret() + 1
    }

    fn secret() -> u32 {
        41
    }
}

fn main() {
    println!("{}", bank::deposit());
    println!("{}", bank::secret());
}
```

<details>
<summary>Ответ</summary>

```text
error[E0603]: function `secret` is private
```

Элементы модуля по умолчанию приватны: ими может пользоваться только код внутри модуля (и его
дочерних модулей). `bank::deposit` объявлена как `pub`, поэтому работает, и она может вызывать
`secret`, потому что находится внутри `bank`. Чтобы вызвать `secret` из `main`, пришлось бы
написать `pub fn secret()`, но лучшее исправление обычно — оставить её приватной и использовать
публичную функцию.

</details>

### Векторы, строки и хеш-таблицы

13. Чем `v[10]` отличается от `v.get(10)` для вектора из 3 элементов?

<details>
<summary>Ответ</summary>

`v[10]` паникует с сообщением «index out of bounds». `v.get(10)` возвращает `None`, поэтому
отсутствующий элемент можно обработать. Используйте `get` для индексов, пришедших снаружи
программы.

</details>

14. Почему нельзя вызвать `v.push(4)`, пока у вас есть ссылка `let first = &v[0];`, которой вы пользуетесь позже?

<details>
<summary>Ответ</summary>

`push` может перераспределить память вектора (перенести элементы в больший участок кучи), и
`first` указывала бы на освобождённую память. Правила заимствования запрещают изменяемое
заимствование (`push`), пока живо общее (`first`): ошибка E0502.

</details>

15. Что напечатает эта программа и какая строка паникует?

```rust
fn main() {
    let s = "héllo";
    println!("{} {}", s.len(), s.chars().count());
    println!("{:?}", s.get(0..2));
    println!("{}", &s[0..3]);
    println!("{}", &s[0..2]);
}
```

<details>
<summary>Ответ</summary>

```text
6 5
None
hé
```

а затем паника на последней строке: `byte index 2 is not a char boundary; it is inside 'é' (bytes 1..3 of string)`.
`h` занимает 1 байт, а `é` — 2, поэтому длина 6 байт при 5 символах. Байты 0..2 разрезают `é`
пополам: `get` безопасно отвечает `None`, а `&s[0..2]` паникует. Байты 0..3 — это ровно `hé`.

</details>

16. Напишите одну строку, которая прибавляет 1 к счётчику `word` в `HashMap<String, u32>` с именем `counts`, создавая запись при необходимости.

<details>
<summary>Ответ</summary>

```rust
*counts.entry(word.to_string()).or_insert(0) += 1;
```

`entry` находит место для ключа, `or_insert(0)` сохраняет 0, если ключ новый, и возвращает
`&mut u32`, а `*` ... `+= 1` меняет число за этой ссылкой. Всё делает один поиск.

</details>

17. Почему `HashMap` может печатать записи в другом порядке при каждом запуске? Что использовать, когда нужен отсортированный вывод?

<details>
<summary>Ответ</summary>

Порядок зависит от значений хешей, а Rust намеренно инициализирует хеширование случайным
секретом (чтобы противостоять атакам hash flooding), поэтому порядок не определён. Для
отсортированного вывода используйте `BTreeMap` или соберите ключи в `Vec` и вызовите `sort()`.

</details>

### Ошибки

18. Когда использовать `panic!`, а когда возвращать `Result`?

<details>
<summary>Ответ</summary>

`panic!` — для багов и нарушенных обещаний (ситуации, которой не должно быть в корректном коде).
`Result` — для сбоев, возможных в корректной программе (плохой ввод пользователя, нет файла,
ошибка сети), чтобы вызывающий код мог решить, как реагировать. Никогда не вызывайте `unwrap()`
для данных, пришедших снаружи программы.

</details>

19. Что делает оператор `?`, где его можно использовать и что нужно, чтобы он мог преобразовать один тип ошибки в другой?

<details>
<summary>Ответ</summary>

Для `Ok(v)` он вычисляется в `v`; для `Err(e)` он сразу возвращает `Err(e)` из текущей функции.
Использовать его можно только в функции, у которой возвращаемый тип — `Result` (или `Option`),
например `fn main() -> Result<(), Box<dyn Error>>`. Если типы ошибок различаются, тип ошибки
функции должен реализовывать `From<ДругаяОшибка>` (или быть `Box<dyn Error>`).

</details>

### Git и контроль версий

20. Какую папку Rust-проекта нельзя коммитить никогда и как это гарантировать? Какие команды создают ветку, делают коммит в ней, возвращают на `main` и сливают ветку с коммитом слияния?

<details>
<summary>Ответ</summary>

Никогда не коммитьте `target/` (результаты сборки). Добавьте строку `/target` в `.gitignore`.
(Заметьте: `cargo new` внутри существующего репозитория не создаёт этот файл за вас.) Команды:

```bash
git switch -c feature/name       # create and switch to the branch
git add . && git commit -m "feat: ..."
git switch main
git merge --no-ff feature/name   # always makes a merge commit
git branch -d feature/name       # delete the merged branch
```

</details>

## Практические задания

Сначала напишите каждую программу сами. В каждом задании **подсказка** даёт план, а **ответ** — одно
из правильных решений. Каждое задание должно проходить `cargo test`,
`cargo clippy --all-targets -- -D warnings` и `cargo fmt --check`. Делайте коммит после каждого
задания (по одному коммиту на задание, в стиле Conventional Commits).

### Задание 1 — Фигуры (структуры, перечисления, методы, векторы)

Создайте `src/bin/shapes.rs`. Определите перечисление `Shape` с тремя вариантами: `Circle { radius: f64 }`, `Rectangle { width: f64, height: f64 }` и `Square(f64)`.
Добавьте методы `area(&self) -> f64` и `name(&self) -> &'static str`. Напишите `fn largest(shapes: &[Shape]) -> Option<&Shape>`.
В `main` постройте вектор из круга радиуса 1.5, прямоугольника 3 × 4 и квадрата со стороной 2.5. Напечатайте по строке на фигуру, общую площадь и
имя самой большой фигуры. Напишите тесты: площадь прямоугольника, площадь и имя квадрата, площадь круга, близкая к π (сравнивайте с допуском),
самая большая из нескольких, самая большая из пустого набора. Ожидаемый вывод:

```text
circle    area   7.07
rectangle area  12.00
square    area   6.25
total area 25.32
largest: rectangle
```

<details>
<summary>Подсказка</summary>

Используйте `match self` с одной веткой на вариант; `Shape::Circle { radius }` связывает поле, а `Shape::Circle { .. }` игнорирует все поля. Площадь круга — `PI * radius * radius`
с `use std::f64::consts::PI;`. Для `largest` держите `best: Option<&Shape>` и обновляйте его в цикле. Для вывода используйте `{:<9}` (дополнить до 9, по левому краю) и `{:>6.2}` (ширина 6, два знака после запятой, по правому краю).
Числа с плавающей точкой после неточных вычислений не следует сравнивать через `==`; используйте `(a - b).abs() < 0.001`. (`&'static str` — тип строкового литерала, который живёт столько же, сколько программа; временам жизни посвящён отдельный урок, [Урок 05.3](../05-rust-intermediate/03-lifetimes.md).)

</details>

<details>
<summary>Ответ</summary>

```rust
use std::f64::consts::PI;

#[derive(Debug)]
enum Shape {
    Circle { radius: f64 },
    Rectangle { width: f64, height: f64 },
    Square(f64),
}

impl Shape {
    fn area(&self) -> f64 {
        match self {
            Shape::Circle { radius } => PI * radius * radius,
            Shape::Rectangle { width, height } => width * height,
            Shape::Square(side) => side * side,
        }
    }

    fn name(&self) -> &'static str {
        match self {
            Shape::Circle { .. } => "circle",
            Shape::Rectangle { .. } => "rectangle",
            Shape::Square(_) => "square",
        }
    }
}

/// The shape with the biggest area, or None for an empty slice.
fn largest(shapes: &[Shape]) -> Option<&Shape> {
    let mut best: Option<&Shape> = None;
    for shape in shapes {
        match best {
            Some(current) if current.area() >= shape.area() => {}
            _ => best = Some(shape),
        }
    }
    best
}

fn main() {
    let shapes = vec![
        Shape::Circle { radius: 1.5 },
        Shape::Rectangle { width: 3.0, height: 4.0 },
        Shape::Square(2.5),
    ];

    let mut total = 0.0;
    for shape in &shapes {
        println!("{:<9} area {:>6.2}", shape.name(), shape.area());
        total += shape.area();
    }
    println!("total area {:.2}", total);

    match largest(&shapes) {
        Some(shape) => println!("largest: {}", shape.name()),
        None => println!("no shapes"),
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn rectangle_area() {
        let r = Shape::Rectangle { width: 3.0, height: 4.0 };
        assert_eq!(r.area(), 12.0);
    }

    #[test]
    fn square_area_and_name() {
        let s = Shape::Square(2.5);
        assert_eq!(s.area(), 6.25);
        assert_eq!(s.name(), "square");
    }

    #[test]
    fn circle_area_is_close_to_pi() {
        let c = Shape::Circle { radius: 1.0 };
        assert!((c.area() - 3.14159).abs() < 0.001);
    }

    #[test]
    fn largest_of_several() {
        let shapes = vec![Shape::Square(1.0), Shape::Square(3.0), Shape::Square(2.0)];
        assert_eq!(largest(&shapes).unwrap().area(), 9.0);
    }

    #[test]
    fn largest_of_none() {
        assert!(largest(&[]).is_none());
    }
}
```

Почему это работает: каждый вариант несёт только нужные ему данные, а каждый `match` покрывает все
три варианта, поэтому добавление четвёртой фигуры позже заставит компилятор указать на каждое
место, которое нужно обновить. Условие `Some(current) if current.area() >= shape.area() => {}`
означает «оставить текущего лучшего»; любой другой случай (лучшего ещё нет или фигура больше)
заменяет его. (Площади квадратов 6.25 и 9.0 точно представимы в числах с плавающей точкой,
поэтому здесь безопасно использовать `assert_eq!`.)

</details>

### Задание 2 — Склад (модули, хеш-таблицы, ошибки)

Создайте `src/bin/inventory.rs`. Внутри напишите встроенный модуль `mod inventory { ... }` с публичной структурой `Inventory` (она хранит `HashMap<String, u32>`; поле приватное), публичным перечислением
`InventoryError { UnknownItem(String), NotEnough { item: String, have: u32, wanted: u32 } }` и публичными методами `new()`, `add(&mut self, item: &str, quantity: u32)`,
`remove(&mut self, item: &str, quantity: u32) -> Result<(), InventoryError>` и `quantity(&self, item: &str) -> u32`. Названия товаров не зависят от регистра (храните их в нижнем регистре).
Неудачный `remove` должен оставлять склад неизменным. Вне модуля реализуйте `Display` для `InventoryError` и выполните в `main` такой сценарий: добавить 10 Apples, 5 apples и 2 Pears, затем напечатать
количество яблок (поиск без учёта регистра), затем попробовать убрать 4 яблока, 3 груши и 1 сливу, затем напечатать, сколько яблок осталось. Напишите четыре теста (слияние вариантов регистра, remove уменьшает запас, слишком много оставляет запас, неизвестный товар). Ожидаемый вывод:

```text
apples: 15
sold 4 apples
error: only 2 of 'pears' left, but 3 wanted
error: we do not sell 'plums'
apples left: 11
```

<details>
<summary>Подсказка</summary>

В `add` используйте `*self.stock.entry(item.to_lowercase()).or_insert(0) += quantity;`. В `remove` сделайте `match self.stock.get_mut(&item)`: `None` даёт `UnknownItem`; `Some(have)` при `*have < quantity` (охранное условие) даёт `NotEnough`; последняя ветка вычитает. Чтобы типы модуля были видны в `main` и в тестах, добавьте `use inventory::{Inventory, InventoryError};` после модуля. `impl fmt::Display for InventoryError` разрешён вне модуля (это тот же крейт), но перечисление должно быть `pub`.

</details>

<details>
<summary>Ответ</summary>

```rust
use std::fmt;

mod inventory {
    use std::collections::HashMap;

    #[derive(Debug, PartialEq)]
    pub enum InventoryError {
        UnknownItem(String),
        NotEnough { item: String, have: u32, wanted: u32 },
    }

    #[derive(Debug, Default)]
    pub struct Inventory {
        stock: HashMap<String, u32>,
    }

    impl Inventory {
        pub fn new() -> Inventory {
            Inventory::default()
        }

        pub fn add(&mut self, item: &str, quantity: u32) {
            *self.stock.entry(item.to_lowercase()).or_insert(0) += quantity;
        }

        pub fn remove(&mut self, item: &str, quantity: u32) -> Result<(), InventoryError> {
            let item = item.to_lowercase();
            match self.stock.get_mut(&item) {
                None => Err(InventoryError::UnknownItem(item)),
                Some(have) if *have < quantity => Err(InventoryError::NotEnough {
                    item,
                    have: *have,
                    wanted: quantity,
                }),
                Some(have) => {
                    *have -= quantity;
                    Ok(())
                }
            }
        }

        pub fn quantity(&self, item: &str) -> u32 {
            self.stock.get(&item.to_lowercase()).copied().unwrap_or(0)
        }
    }
}

use inventory::{Inventory, InventoryError};

impl fmt::Display for InventoryError {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        match self {
            InventoryError::UnknownItem(item) => write!(f, "we do not sell '{item}'"),
            InventoryError::NotEnough { item, have, wanted } => {
                write!(f, "only {have} of '{item}' left, but {wanted} wanted")
            }
        }
    }
}

fn main() {
    let mut shop = Inventory::new();
    shop.add("Apples", 10);
    shop.add("apples", 5);
    shop.add("Pears", 2);

    println!("apples: {}", shop.quantity("APPLES"));
    for (item, qty) in [("apples", 4), ("pears", 3), ("plums", 1)] {
        match shop.remove(item, qty) {
            Ok(()) => println!("sold {qty} {item}"),
            Err(e) => println!("error: {e}"),
        }
    }
    println!("apples left: {}", shop.quantity("apples"));
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn add_merges_case_variants() {
        let mut shop = Inventory::new();
        shop.add("Apples", 10);
        shop.add("apples", 5);
        assert_eq!(shop.quantity("APPLES"), 15);
    }

    #[test]
    fn remove_reduces_stock() {
        let mut shop = Inventory::new();
        shop.add("tea", 5);
        assert_eq!(shop.remove("tea", 2), Ok(()));
        assert_eq!(shop.quantity("tea"), 3);
    }

    #[test]
    fn remove_too_many_fails_and_keeps_stock() {
        let mut shop = Inventory::new();
        shop.add("tea", 1);
        let err = shop.remove("tea", 3).unwrap_err();
        assert_eq!(err, InventoryError::NotEnough { item: "tea".into(), have: 1, wanted: 3 });
        assert_eq!(shop.quantity("tea"), 1);
    }

    #[test]
    fn unknown_item() {
        let mut shop = Inventory::new();
        assert_eq!(shop.remove("milk", 1), Err(InventoryError::UnknownItem("milk".into())));
    }
}
```

Почему это работает: поле `stock` приватное, поэтому изменять склад можно только через `add` и
`remove`, которые соблюдают правила (названия в нижнем регистре, запас никогда не уходит в минус).
Ветка с охранным условием стоит перед общей веткой `Some(have)`, поэтому слишком большое
`quantity` отклоняется до того, как что-либо вычитается. `"tea".into()` превращает `&str` в
`String` (в тестах, где приветствуется короткая форма).

</details>

### Задание 3 — Разбор конфигурации (строки, таблицы, ошибки с номерами строк)

Создайте `src/bin/config.rs`. Напишите `fn parse_config(text: &str) -> Result<BTreeMap<String, String>, ConfigError>` для текста вроде

```text
# server settings
port = 8080

name = demo
```

Правила: пустые строки и строки, начинающиеся с `#`, пропускаются; каждая другая строка должна иметь вид `key = value` (делить по **первому** `=`); пробелы вокруг ключа и значения убираются; значение может быть пустым или содержать `=`.
Ошибки (все несут номер строки, счёт с 1): `MissingEquals { line }`, `EmptyKey { line }`, `DuplicateKey { line, key }`. Напишите шесть тестов: пары и комментарии, пустое значение и `=` внутри значения, по одному тесту на каждую ошибку и на пустой текст. В `main` напечатайте результаты
`parse_config` для четырёх текстов, показанных в ожидаемом выводе:

```text
Ok({"name": "demo", "port": "8080"})
Err(MissingEquals { line: 2 })
Err(EmptyKey { line: 2 })
Err(DuplicateKey { line: 3, key: "a" })
```

(Тексты такие: пример выше вместе со строкой-комментарием; `"a=1\nbroken line\n"`; `"a=1\n = 2\n"`; `"a=1\nb=2\na=3\n"`.)

<details>
<summary>Подсказка</summary>

Используйте `for (index, raw) in text.lines().enumerate()` и `let line_number = index + 1;`. После `trim()` пропускайте, если `line.is_empty() || line.starts_with('#')`.
`let Some((key, value)) = line.split_once('=') else { return Err(...) };` обрабатывает отсутствие `=`. `BTreeMap` держит ключи отсортированными, что делает отладочный вывод и тесты предсказуемыми.
Проверяйте дубликаты через `map.contains_key(key)` до `insert`. Выведите `Debug` и `PartialEq` для ошибки, чтобы пользоваться `assert_eq!`.

</details>

<details>
<summary>Ответ</summary>

```rust
use std::collections::BTreeMap;

#[derive(Debug, PartialEq)]
enum ConfigError {
    MissingEquals { line: usize },
    EmptyKey { line: usize },
    DuplicateKey { line: usize, key: String },
}

/// Parses `key = value` lines. Blank lines and lines starting with '#' are skipped.
fn parse_config(text: &str) -> Result<BTreeMap<String, String>, ConfigError> {
    let mut map = BTreeMap::new();
    for (index, raw) in text.lines().enumerate() {
        let line_number = index + 1;
        let line = raw.trim();
        if line.is_empty() || line.starts_with('#') {
            continue;
        }
        let Some((key, value)) = line.split_once('=') else {
            return Err(ConfigError::MissingEquals { line: line_number });
        };
        let key = key.trim();
        if key.is_empty() {
            return Err(ConfigError::EmptyKey { line: line_number });
        }
        if map.contains_key(key) {
            return Err(ConfigError::DuplicateKey { line: line_number, key: key.to_string() });
        }
        map.insert(key.to_string(), value.trim().to_string());
    }
    Ok(map)
}

fn main() {
    let good = "# server settings\nport = 8080\n\nname = demo\n";
    println!("{:?}", parse_config(good));
    println!("{:?}", parse_config("a=1\nbroken line\n"));
    println!("{:?}", parse_config("a=1\n = 2\n"));
    println!("{:?}", parse_config("a=1\nb=2\na=3\n"));
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn parses_pairs_and_skips_comments() {
        let map = parse_config("# note\nport = 8080\n\nname=demo\n").unwrap();
        assert_eq!(map.len(), 2);
        assert_eq!(map["port"], "8080");
        assert_eq!(map["name"], "demo");
    }

    #[test]
    fn value_may_be_empty_and_contain_equals() {
        let map = parse_config("a =\nb = x=y\n").unwrap();
        assert_eq!(map["a"], "");
        assert_eq!(map["b"], "x=y");
    }

    #[test]
    fn missing_equals_reports_line() {
        assert_eq!(parse_config("a=1\nbroken\n"), Err(ConfigError::MissingEquals { line: 2 }));
    }

    #[test]
    fn empty_key_reports_line() {
        assert_eq!(parse_config("=1\n"), Err(ConfigError::EmptyKey { line: 1 }));
    }

    #[test]
    fn duplicate_key() {
        assert_eq!(
            parse_config("a=1\nb=2\na=3\n"),
            Err(ConfigError::DuplicateKey { line: 3, key: "a".into() })
        );
    }

    #[test]
    fn empty_text_is_an_empty_map() {
        assert!(parse_config("").unwrap().is_empty());
    }
}
```

Почему это работает: `split_once` режет только по первому `=`, поэтому `x=y` остаётся в значении.
Функция останавливается на первой проблеме и сообщает, где она. Поскольку функция возвращает и
разобранные данные, и ошибки как значения, её можно тестировать без файлов и терминала.

</details>

### Задание 4 — Полный рабочий процесс Git с конфликтом (контроль версий)

В отдельной папке (не в вашем учебном репозитории) сделайте следующее и записывайте, что видите после каждого шага:

1. `mkdir ~/git-practice && cd ~/git-practice && git init -b main`, затем `cargo new demo` и `cd demo && cargo build && cd ..`.
2. Выполните `git status --short`, затем `git status --short -uall | wc -l`. В чём проблема? Исправьте её, создав `demo/.gitignore` со строкой `/target`, и проверьте, что `git status --short -uall` теперь перечисляет только 4 файла.
3. Закоммитьте всё с сообщением `chore: create the demo project`.
4. Создайте ветку `fix/greeting` и переключитесь на неё. В `demo/src/main.rs` замените `Hello, world!` на `Hello, systems!`. Закоммитьте как `fix: change the greeting to systems`.
5. Вернитесь на `main`. На `main` измените ту же строку на `Hello, Rust!` и закоммитьте как `feat: greet Rust`.
6. Слейте `fix/greeting` в `main`. Git сообщит о конфликте. Откройте файл, уберите маркеры и сделайте строку `Hello, Rust systems!`. Завершите слияние.
7. Удалите влитую ветку, поставьте результату тег `v0.1.0` и покажите историю в виде графа. Запустите программу.

Ожидаемые ключевые строки вывода (хеши будут другими):

```text
?? demo/
...
Auto-merging demo/src/main.rs
CONFLICT (content): Merge conflict in demo/src/main.rs
Automatic merge failed; fix conflicts and then commit the result.
...
*   c4e4821 Merge branch 'fix/greeting'
|\
| * 676b350 fix: change the greeting to systems
* | f829c3a feat: greet Rust
|/
* 704cb66 chore: create the demo project
```

и программа печатает `Hello, Rust systems!`.

<details>
<summary>Подсказка</summary>

Без `.gitignore` список `-uall` показывает каждый файл в `demo/target/` (десятки строк; мы насчитали 33). Ветка: `git switch -c fix/greeting`; правка; `git commit -am "..."` (параметр `-a` добавляет в коммит изменения файлов, которые Git уже знает).
После конфликта `git status` показывает `both modified: demo/src/main.rs`. В файле часть между `<<<<<<< HEAD` и `=======` — ваша ветка (`main`), а часть между `=======` и `>>>>>>> fix/greeting` пришла из другой ветки.
Оставьте нужный текст, удалите три строки маркеров, затем `git add demo/src/main.rs` и `git commit --no-edit`. Вспомните [Урок 01.7 — Конфликты слияния](../01-git-and-version-control/07-merge-conflicts.md).

</details>

<details>
<summary>Ответ</summary>

```bash
mkdir ~/git-practice && cd ~/git-practice
git init -b main
cargo new demo
(cd demo && cargo build)
git status --short                    # ?? demo/
git status --short -uall | wc -l      # dozens of lines: target/ is not ignored!
printf '/target\n' > demo/.gitignore
git status --short -uall              # only .gitignore, Cargo.lock, Cargo.toml, src/main.rs
git add . && git commit -m "chore: create the demo project"

git switch -c fix/greeting
sed -i '' 's/Hello, world!/Hello, systems!/' demo/src/main.rs     # on Linux: sed -i without ''
git commit -am "fix: change the greeting to systems"

git switch main
sed -i '' 's/Hello, world!/Hello, Rust!/' demo/src/main.rs
git commit -am "feat: greet Rust"

git merge fix/greeting                # CONFLICT (content): Merge conflict in demo/src/main.rs
```

Отредактируйте `demo/src/main.rs` так, чтобы в нём не осталось маркеров:

```rust
fn main() {
    println!("Hello, Rust systems!");
}
```

```bash
git add demo/src/main.rs
git commit --no-edit
git branch -d fix/greeting
git tag -a v0.1.0 -m "first release"
git log --oneline --graph
(cd demo && cargo run -q)
```

Можно также открыть файл в RustRover и воспользоваться его инструментом слияния. Почему это
работает: Git не смог выбрать между двумя правками одной строки и спросил вас. После того как вы
решили и сделали `git add` для файла, коммит слияния соединяет две истории; граф показывает обе
ветки. Шаг с `.gitignore` важен, потому что `cargo new` не создаёт его, когда папка уже находится
внутри репозитория Git.

</details>

## Если что-то не получилось

| Тема | Вопросы / задания | Повторить |
|---|---|---|
| Структуры, кортежные структуры, синтаксис обновления, владение полями | 1, 2 | [Урок 04.1 — Структуры](01-structs.md) |
| Методы, виды `self`, ассоциированные функции | 3, 4 | [Урок 04.2 — Методы и ассоциированные функции](02-methods-and-associated-functions.md) |
| Перечисления и варианты с данными | 5, задание 1 | [Урок 04.3 — Перечисления](03-enums.md) |
| `Option`, `match`, исчерпывающие ветки, охранные условия | 6, 7, 8, задание 1 | [Урок 04.4 — Option и match](04-option-and-match.md) |
| `if let`, `let else`, образцы-деструктуризация | 9, 10 | [Урок 04.5 — if let, let else и образцы](05-if-let-let-else-and-patterns.md) |
| Модули, `pub`, крейты и пакеты | 11, 12, задание 2 | [Урок 04.6 — Модули, крейты и пакеты](06-modules-crates-and-packages.md) |
| Векторы, индексация, заимствование и рост | 13, 14 | [Урок 04.7 — Векторы](07-vectors.md) |
| Строки, UTF-8, срезы текста, разбиение | 15, задание 3 | [Урок 04.8 — Строки подробно](08-strings-in-depth.md) |
| Хеш-таблицы, API `entry`, порядок | 16, 17, задание 2 | [Урок 04.9 — Хеш-таблицы (HashMap)](09-hash-maps.md) |
| `Result`, `?`, собственные ошибки, `panic!` | 18, 19, задания 2 и 3 | [Урок 04.10 — Обработка ошибок](10-error-handling.md) |
| Рабочий процесс проекта: вехи, тесты, clippy, fmt | все задания | [Урок 04.11 — Проект: телефонная книга](11-project-phone-book.md) |
| Git: `.gitignore`, ветки, слияние, конфликты, теги | 20, задание 4 | [Урок 01.6 — Ветки и слияние](../01-git-and-version-control/06-branches-and-merging.md), [Урок 01.7 — Конфликты слияния](../01-git-and-version-control/07-merge-conflicts.md), [Урок 01.9 — Командная работа: pull request, код-ревью и соглашения](../01-git-and-version-control/09-teamwork-pull-requests-and-conventions.md) |
| Написание тестов | все задания | [Урок 02.8 — Ваши первые тесты](../02-programming-basics/08-your-first-tests.md) |

Если вы верно ответили на **16 или больше** вопросов и выполнили все четыре задания, вы готовы к
следующему этапу. Если верных ответов **меньше 12**, вернитесь к урокам из таблицы, прежде чем
продолжать; Этап 05 напрямую опирается на эти идеи. От 12 до 15: повторите только строки тех
вопросов, где ошиблись, а через день решите эти вопросы заново.

## Что дальше

Этап 05 — *Rust: средний уровень* — превращает код, который вы писали здесь, в *переиспользуемый* код:

- [Урок 05.1 — Обобщения (generics)](../05-rust-intermediate/01-generics.md): `<T>`, которое вы видели в `Vec<T>` и `Result<T, E>`, и как писать свои.
- [Урок 05.2 — Трейты](../05-rust-intermediate/02-traits.md): чем на самом деле являются `Display`, `Error`, `From` и `Hash` и как определять свои.
- Позже: времена жизни, замыкания, итераторы, углублённое тестирование, крейты (`serde`, `clap`, `thiserror`, `anyhow`), потоки и проект `minigrep`.

Перед тем как идти дальше:

1. Убедитесь, что проект телефонной книги и задания контрольной точки закоммичены и помечены тегами. Если вы используете GitHub или GitLab, отправьте их туда.
2. Продолжайте запускать тесты: с этого момента запускайте `cargo test`, `cargo fmt` и `cargo clippy` перед каждым коммитом.
3. Сделайте перерыв. Этап 05 проще начинать со свежей головой.

## Итоги

- Этап 04 дал вам инструменты для моделирования данных: структуры для «и то, и другое», перечисления для «или то, или другое» и методы, чтобы привязать поведение.
- `Option` и `Result` делают отсутствие и сбой частью типа; `match`, `if let`, `let else` и `?` — способы с ними справляться.
- Модули, `pub` и крейты организуют код и определяют, что видно.
- `Vec`, `String`, `HashMap` и `BTreeMap` хранят данные любого размера; правила владения и заимствования действуют для них так же, как для любого другого значения.
- У хороших программ есть чёткий контракт, маленькие протестированные функции, ошибки как значения, чистые `fmt` и `clippy` и по одному коммиту Git на каждый законченный шаг.

## Источники и что почитать

- [The Rust Programming Language — 5. Using Structs to Structure Related Data](https://doc.rust-lang.org/book/ch05-00-structs.html) — структуры и методы, для повторения.
- [The Rust Programming Language — 6. Enums and Pattern Matching](https://doc.rust-lang.org/book/ch06-00-enums.html) — перечисления, `Option`, `match`, `if let`, `let else`.
- [The Rust Programming Language — 7. Managing Growing Projects with Packages, Crates, and Modules](https://doc.rust-lang.org/book/ch07-00-managing-growing-projects-with-packages-crates-and-modules.html) — пакеты, крейты и модули.
- [The Rust Programming Language — 8. Common Collections](https://doc.rust-lang.org/book/ch08-00-common-collections.html) — векторы, строки и хеш-таблицы.
- [The Rust Programming Language — 9. Error Handling](https://doc.rust-lang.org/book/ch09-00-error-handling.html) — `panic!`, `Result` и `?`.
- [Rustlings](https://github.com/rust-lang/rustlings) — небольшие упражнения, которые запускаются локально; разделы про структуры, перечисления, модули, векторы, строки, хеш-таблицы и обработку ошибок соответствуют этому этапу.
- [Pro Git — Git Branching: Basic Branching and Merging](https://git-scm.com/book/en/v2/Git-Branching-Basic-Branching-and-Merging) — ветки, слияния и конфликты подробно.

---

[🏠 Начало](../START-HERE.md) · [← 04.11 Проект: телефонная книга](11-project-phone-book.md) · [📋 Этап 04](README.md) · [Этап 05 — Rust: средний уровень →](../05-rust-intermediate/README.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
