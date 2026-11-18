<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← Этап 06 — Rust: продвинутый уровень](README.md) · [📋 Этап 06](README.md) · [06.2 Продвинутые трейты и типы →](02-advanced-traits-and-types.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/06-rust-advanced/01-trait-objects-and-dynamic-dispatch.md) <!-- nav -->

# Урок 06.1 — Трейт-объекты и динамическая диспетчеризация

> **Этап 06 — Rust: продвинутый уровень** · Урок 1 из 12 · ⏱ около 4 часов (2 занятия)

## Что вы узнаете

- Почему `Vec` обобщённого типа не может хранить значения разных типов и как `dyn Trait` решает эту задачу.
- Что такое трейт-объект и где он может находиться: `Box<dyn Trait>`, `&dyn Trait`, `&mut dyn Trait`.
- Как трейт-объект устроен внутри: «толстый» указатель и vtable.
- Чем статическая диспетчеризация отличается от динамической и как выбирать между ними.
- Каким правилам должен следовать трейт, чтобы его можно было использовать как `dyn Trait` (совместимость с `dyn`).
- Как читать ошибки компилятора, которые встречаются при работе с трейт-объектами.

## Перед началом

- Вы прошли [Урок 05.14 — Контрольная точка](../05-rust-intermediate/14-checkpoint.md).
- Вы знаете обобщения (generics) и ограничения трейтов по [Уроку 05.1 — Обобщённые типы](../05-rust-intermediate/01-generics.md) и трейты по [Уроку 05.2 — Трейты](../05-rust-intermediate/02-traits.md).
- Вы знаете `Box` по [Уроку 05.9 — Умные указатели](../05-rust-intermediate/09-smart-pointers.md) и замыкания по [Уроку 05.4 — Замыкания](../05-rust-intermediate/04-closures.md).
- Создайте учебный проект внутри вашего учебного репозитория: `cargo new code/p06-trait-objects`. Если позже `git status` покажет папку `target/`, добавьте строку `target/` в файл `.gitignore` (см. [Урок 01.3](../01-git-and-version-control/03-your-first-repository-and-commits.md)).

## Новые слова

| Слово | Значение |
|---|---|
| **трейт-объект** (trait object) | Значение «какого-то типа, который реализует трейт», причём точный тип известен только во время работы программы. Записывается как `dyn Trait`. |
| **`dyn`** | Ключевое слово (от *dynamic*, «динамический»), которое отмечает тип трейт-объекта, например `dyn Shape`. |
| **статическая диспетчеризация** (static dispatch) | Компилятор решает на этапе компиляции, какую функцию запустит вызов. Так работают обобщения. |
| **динамическая диспетчеризация** (dynamic dispatch) | Программа решает во время работы, какую функцию запустит вызов. Так работают трейт-объекты. |
| **vtable** (virtual table, таблица виртуальных методов) | Небольшая таблица адресов функций для одного типа и одного трейта. По ней трейт-объект находит нужный метод. |
| **«толстый» указатель** (fat pointer) | Указатель вдвое больше обычного: в нём лежит адрес и ещё одно значение (адрес vtable или длина). |
| **совместимый с `dyn`** (dyn compatible) | Трейт, который можно использовать как `dyn Trait`. Старое название — *object safe* («объектно-безопасный»). |
| **мономорфизация** (monomorphization) | Компилятор создаёт отдельную копию обобщённой функции для каждого конкретного типа, с которым её используют. Вы встречали её в [Уроке 05.1](../05-rust-intermediate/01-generics.md). |
| **`Sized`** | Маркерный трейт для типов, размер которых известен на этапе компиляции. `dyn Trait` — *не* `Sized`. |
| **API** (Application Programming Interface, программный интерфейс приложения) | Набор функций и типов, которые один фрагмент кода предлагает другому. |
| **GUI** (Graphical User Interface, графический интерфейс пользователя) | Окна, кнопки и меню программы, которыми вы пользуетесь с помощью мыши. |
| **UTF-8** (Unicode Transformation Format, 8-bit) | Стандартный способ хранить текст в виде байтов. `String::from_utf8` превращает байты в текст и возвращает ошибку, если байты — не корректный UTF-8. |

## 1. Задача: разные типы в одном списке

Представьте программу для рисования. В ней есть круги и прямоугольники. И те и другие умеют сообщать свою площадь, поэтому мы даём им общий трейт с методом `area`.

Обе фигуры — кортежные структуры: `Circle(f64)` хранит радиус, `Rect(f64, f64)` хранит ширину и высоту. Нам нужен один список, в котором лежат и круг, и прямоугольник, и мы хотим пройти по нему циклом. Сначала попробуем очевидный способ:

```rust
trait Shape {
    fn area(&self) -> f64;
}

struct Circle(f64);
struct Rect(f64, f64);

impl Shape for Circle {
    fn area(&self) -> f64 {
        std::f64::consts::PI * self.0 * self.0
    }
}

impl Shape for Rect {
    fn area(&self) -> f64 {
        self.0 * self.1
    }
}

fn main() {
    let shapes = vec![Circle(1.0), Rect(2.0, 3.0)];
    println!("{}", shapes.len());
}
```

```text
error[E0308]: mismatched types
  --> src/main.rs:21:36
   |
21 |     let shapes = vec![Circle(1.0), Rect(2.0, 3.0)];
   |                                    ^^^^^^^^^^^^^^ expected `Circle`, found `Rect`

For more information about this error, try `rustc --explain E0308`.
error: could not compile `demo` (bin "demo") due to 1 previous error
```

**Что происходит:**

1. `vec![...]` строит `Vec<T>`. У `Vec` ровно один тип элементов `T`.
2. Первый элемент — `Circle`, поэтому компилятор решает, что `T = Circle`.
3. Второй элемент — `Rect`. `Rect` — это не `Circle`, и компилятор останавливается с ошибкой `E0308` (несовпадение типов).

Обобщения здесь не помогают. Обобщённый `Vec<T>`, где `T: Shape`, всё равно означает *один* конкретный `T` для всего вектора. Нам нужен тип, который означает «любой тип, реализующий `Shape`». В Rust такой есть: `dyn Shape`.

## 2. `dyn Trait`: трейт-объект

`dyn Shape` — это тип «какое-то значение, чей тип реализует `Shape`; какой именно, я не знаю, пока программа не запустится». Такой тип называется **трейт-объектом**.

Есть одна загвоздка. У разных типов разные размеры: `Circle` здесь занимает 8 байт, `Rect` — 16. `Vec` хранит элементы вплотную друг к другу, поэтому у каждого элемента должен быть одинаковый размер. Решение — хранить не само значение, а **указатель** на него. Все указатели одного размера. Самый простой владеющий указатель — `Box` (вы знаете его по [Уроку 05.9](../05-rust-intermediate/09-smart-pointers.md)): `Box<dyn Shape>` владеет значением в куче, а `Vec` хранит только коробки.

Вот полная программа с двумя разными фигурами в одном списке:

```rust
trait Shape {
    fn area(&self) -> f64;
    fn name(&self) -> String;
}

struct Circle(f64); // радиус
struct Rect(f64, f64); // ширина и высота

impl Shape for Circle {
    fn area(&self) -> f64 {
        std::f64::consts::PI * self.0 * self.0
    }
    fn name(&self) -> String {
        String::from("circle")
    }
}

impl Shape for Rect {
    fn area(&self) -> f64 {
        self.0 * self.1
    }
    fn name(&self) -> String {
        String::from("rectangle")
    }
}

fn main() {
    let shapes: Vec<Box<dyn Shape>> = vec![Box::new(Circle(1.0)), Box::new(Rect(2.0, 3.0))];

    for shape in &shapes {
        println!("{} has area {:.2}", shape.name(), shape.area());
    }

    let total: f64 = shapes.iter().map(|s| s.area()).sum();
    println!("total area: {:.2}", total);
}
```

```text
circle has area 3.14
rectangle has area 6.00
total area: 9.14
```

**Что происходит:**

1. `trait Shape` перечисляет два метода. Всё, что их реализует, можно использовать как `Shape`.
2. `Circle` и `Rect` — две не связанные друг с другом структуры. У каждой своя `impl Shape`.
3. `Vec<Box<dyn Shape>>` — это «вектор коробок, каждая коробка владеет каким-то `Shape`».
4. `Box::new(Circle(1.0))` создаёт `Box<Circle>`. Rust автоматически преобразует его в `Box<dyn Shape>`, потому что так сказано в целевом типе. Это преобразование называется *приведением к безразмерному типу* (unsized coercion): компилятор забывает точный тип и помнит только «это `Shape`».
5. `for shape in &shapes` проходит по `&Box<dyn Shape>`. Вызов `shape.area()` запускает правильный `area` для каждого элемента: формулу круга для круга и произведение для прямоугольника.
6. `.iter().map(|s| s.area()).sum()` складывает площади. Итераторы работают с трейт-объектами так же, как с любыми другими типами.

> 💡 **Совет:** слово `dyn` нужно писать каждый раз, когда вы называете тип трейт-объекта: `Box<dyn Shape>`, `&dyn Shape`. В старом коде его иногда опускают (`Box<Shape>`); в редакциях 2021 и 2024 это ошибка. Всегда пишите `dyn`.

## 3. Где может находиться трейт-объект

У `dyn Trait` нет известного размера, поэтому его нельзя хранить прямо в переменной или возвращать по значению. Его всегда помещают за указатель. Вот обычные варианты:

| Тип | Что означает | Когда использовать |
|---|---|---|
| `Box<dyn Trait>` | Владеет значением в куче. | Коллекция, которая владеет элементами; функция, которая возвращает «какое-то значение». |
| `&dyn Trait` | Общее (разделяемое) заимствование. | Параметр функции, которая только читает. |
| `&mut dyn Trait` | Исключительное заимствование. | Параметр функции, которая меняет значение или вызывает методы с `&mut self`. |
| `Rc<dyn Trait>` / `Arc<dyn Trait>` | Совместное владение (один поток / много потоков). | Один и тот же объект используется во многих местах. |

Очень частый настоящий пример — `&mut dyn Write`. Трейт `std::io::Write` реализован для файлов, сетевых соединений, терминала, а также для `Vec<u8>` (запись в память). Функция, которая принимает `&mut dyn Write`, работает со всеми ними:

```rust
use std::io::Write;

fn write_report(out: &mut dyn Write, user: &str) -> std::io::Result<()> {
    writeln!(out, "report for {}", user)?;
    writeln!(out, "status: ok")?;
    Ok(())
}

fn main() {
    // 1. Запись в память.
    let mut buffer: Vec<u8> = Vec::new();
    write_report(&mut buffer, "ana").unwrap();
    println!("buffer holds {} bytes", buffer.len());

    // 2. Запись в терминал.
    let mut stdout = std::io::stdout();
    write_report(&mut stdout, "ana").unwrap();
}
```

```text
buffer holds 26 bytes
report for ana
status: ok
```

**Что происходит:**

1. `write_report` принимает `out: &mut dyn Write`: «изменяемое заимствование чего угодно, куда можно записывать».
2. `writeln!(out, ...)` похож на `println!`, но пишет в `out`. Оператор `?` передаёт ошибку наверх, если запись не удалась (`?` вы изучили в [Уроке 04.10](../04-structs-enums-collections-errors/10-error-handling.md)).
3. В первом вызове `&mut buffer` — это `&mut Vec<u8>`, который становится `&mut dyn Write`. Ничего не печатается: текст попадает в вектор. «report for ana\n» — это 15 байт, а «status: ok\n» — 11 байт, всего 26.
4. Во втором вызове `&mut stdout` пишет в терминал.
5. Одна и та же скомпилированная функция обработала два разных типа.

> 🧪 **Тестирование:** поэтому `dyn Write` хорош для кода, который легко тестировать. Настоящая программа пишет в терминал, а тест передаёт `Vec<u8>` и проверяет байты. Именно это вы сделаете в Упражнении 2.

## 4. Как это устроено внутри: «толстый» указатель и vtable

Когда вы вызываете `shape.area()` у `Box<dyn Shape>`, как программа узнаёт, какой `area` запускать? Ответ несёт сам указатель.

Обычная ссылка `&Circle` — это одно число: адрес круга. Ссылка на трейт-объект `&dyn Shape` — это **два числа**: адрес значения и адрес **vtable**. Поэтому её называют «толстым» указателем (fat pointer).

```text
   Box<dyn Shape>  (16 bytes)
  +----------------+----------------+
  | data pointer   | vtable pointer |
  +-------+--------+--------+-------+
          |                 |
          v                 v
   +-------------+     +--------------------------+
   | Circle(1.0) |     | vtable for Circle: Shape |
   |   (heap)    |     |  drop function           |
   +-------------+     |  size  = 8               |
                       |  align = 8               |
                       |  area  -> Circle::area   |
                       |  name  -> Circle::name   |
                       +--------------------------+
```

Компилятор строит одну vtable для каждой пары (тип, трейт), которая превращается в трейт-объект. Вызов `shape.area()` превращается в следующее: взять указатель на vtable, найти ячейку для `area`, перейти по адресу, который в ней записан. Размеры можно проверить самим:

```rust
use std::fmt::Debug;
use std::mem::{size_of, size_of_val};

trait Shape {
    fn area(&self) -> f64;
}

struct Circle(f64);
struct Rect(f64, f64);

impl Shape for Circle {
    fn area(&self) -> f64 {
        std::f64::consts::PI * self.0 * self.0
    }
}

impl Shape for Rect {
    fn area(&self) -> f64 {
        self.0 * self.1
    }
}

fn main() {
    println!("&Circle        = {} bytes", size_of::<&Circle>());
    println!("&dyn Shape     = {} bytes", size_of::<&dyn Shape>());
    println!("Box<Circle>    = {} bytes", size_of::<Box<Circle>>());
    println!("Box<dyn Shape> = {} bytes", size_of::<Box<dyn Shape>>());
    println!("&[u8]          = {} bytes", size_of::<&[u8]>());
    println!("&dyn Debug     = {} bytes", size_of::<&dyn Debug>());

    let shapes: Vec<Box<dyn Shape>> = vec![Box::new(Circle(1.0)), Box::new(Rect(2.0, 3.0))];
    for shape in &shapes {
        println!(
            "area {:.2}, the value behind the pointer is {} bytes",
            shape.area(),
            size_of_val(&**shape)
        );
    }
}
```

```text
&Circle        = 8 bytes
&dyn Shape     = 16 bytes
Box<Circle>    = 8 bytes
Box<dyn Shape> = 16 bytes
&[u8]          = 16 bytes
&dyn Debug     = 16 bytes
area 3.14, the value behind the pointer is 8 bytes
area 6.00, the value behind the pointer is 16 bytes
```

**Что происходит:**

1. `size_of::<T>()` возвращает размер типа в байтах.
2. `&Circle` занимает 8 байт: один адрес (на этом Mac адреса 64-битные, то есть 8 байт).
3. `&dyn Shape` занимает 16 байт: адрес плюс адрес vtable. `Box<dyn Shape>` тоже занимает 16 байт.
4. `&[u8]` (срез) тоже 16 байт. Это тоже «толстый» указатель, но второе число в нём — длина, а не адрес vtable.
5. `size_of_val(&**shape)` спрашивает размер значения, на которое *указывает* указатель. Программа узнаёт его во время работы из vtable: 8 байт для круга, 16 для прямоугольника.

> ⚡ **Производительность:** вызов через vtable — это *косвенный* вызов. Компилятор обычно не может его встроить (inline — подставить тело функции в место вызова), а CPU (Central Processing Unit, центральный процессор — микросхема, выполняющая инструкции) должен угадывать, куда перейти. В большинстве программ это не заметно. В цикле, который выполняется сотни миллионов раз, это может иметь значение. Сначала измеряйте (см. [Урок 05.12 — Основы бенчмарков](../05-rust-intermediate/12-benchmarking-basics.md)), а не гадайте.

## 5. Статическая и динамическая диспетчеризация

Вы увидели два способа написать «функцию, которая работает с любым `Shape`»:

```rust
trait Shape {
    fn area(&self) -> f64;
}

struct Circle(f64);
struct Rect(f64, f64);

impl Shape for Circle {
    fn area(&self) -> f64 {
        std::f64::consts::PI * self.0 * self.0
    }
}

impl Shape for Rect {
    fn area(&self) -> f64 {
        self.0 * self.1
    }
}

// Статическая диспетчеризация: компилятор делает по копии этой функции для каждого типа.
fn print_static<T: Shape>(shape: &T) {
    println!("static: {:.2}", shape.area());
}

// Динамическая диспетчеризация: одна копия; метод находится во время работы.
fn print_dynamic(shape: &dyn Shape) {
    println!("dynamic: {:.2}", shape.area());
}

fn main() {
    let c = Circle(1.0);
    let r = Rect(2.0, 3.0);

    print_static(&c);
    print_static(&r);

    print_dynamic(&c);
    print_dynamic(&r);
}
```

```text
static: 3.14
static: 6.00
dynamic: 3.14
dynamic: 6.00
```

**Что происходит:**

1. `print_static<T: Shape>` — обобщённая функция. Для каждого типа, с которым её вызывают, компилятор пишет отдельную копию (мономорфизация): одну для `Circle` и одну для `Rect`. Каждый вызов — прямой переход по известному адресу. Это **статическая диспетчеризация**.
2. `print_dynamic` принимает `&dyn Shape`. Копия функции одна. Внутри `shape.area()` идёт через vtable. Это **динамическая диспетчеризация**.
3. Вывод одинаковый. Различается лишь то, как машина до него добирается.

Сравним:

| | Статическая диспетчеризация (`T: Trait`, `impl Trait`) | Динамическая диспетчеризация (`dyn Trait`) |
|---|---|---|
| Когда выбирается метод? | На этапе компиляции | Во время работы |
| Скорость вызова | Прямой, можно встроить | Косвенный, через vtable |
| Размер машинного кода | Одна копия на тип (может сильно вырасти) | Одна копия для всех типов |
| Разные типы в одном `Vec`? | Нет | Да |
| Тип должен быть известен при компиляции? | Да | Нет |
| Вернуть разные типы из одной функции? | Нет | Да, через `Box<dyn Trait>` |
| Можно добавить новый тип, не меняя код, который использует трейт? | Да, но код пользователя компилируется заново | Да, даже из другого крейта |

Практическое правило:

- Начинайте с обобщений или `impl Trait`. Они быстрые, а компилятор проверяет всё за вас.
- Используйте `dyn Trait`, когда нужны **разные типы за одним интерфейсом во время работы**: список плагинов, обработчики, выбранные по файлу настроек, виджеты GUI (графического интерфейса), «стратегия», которую выбирает пользователь.
- Используйте `dyn Trait` и тогда, когда хотите сократить время компиляции и размер программы, а вызов не критичен по скорости.

> 📏 **Соглашение:** в **параметре** функции по возможности предпочитайте `impl Trait` или обобщённый `T: Trait`; используйте `&dyn Trait`, когда нужна одна скомпилированная функция или значение уже является трейт-объектом. В **поле структуры**, которое должно хранить много разных типов, используйте `Box<dyn Trait>`.

## 6. Совместимость с dyn: какие трейты могут быть трейт-объектами

Не каждый трейт можно использовать как `dyn Trait`. Компилятор должен суметь построить vtable с фиксированными ячейками. Трейт, который следует правилам, называется **совместимым с dyn** (раньше говорили «object safe»). Главные правила для методов трейта:

- Метод не должен возвращать `Self`. Vtable не знает конкретного типа, поэтому не может знать, что возвращается.
- У метода не должно быть собственных обобщённых параметров типа (`fn log<T>(...)`). У обобщённого метода бесконечно много версий, а в vtable нужна одна фиксированная ячейка.
- У метода должен быть получатель (`&self`, `&mut self`, `self: Box<Self>` и т. д.). Функцию без `self`, например `fn new() -> Self`, нельзя вызвать у трейт-объекта.
- Трейт не должен требовать `Self: Sized`.

Посмотрите, что скажет компилятор, если нарушить первое правило:

```rust
trait Duplicate {
    fn duplicate(&self) -> Self;
}

struct Note(String);

impl Duplicate for Note {
    fn duplicate(&self) -> Self {
        Note(self.0.clone())
    }
}

fn main() {
    let items: Vec<Box<dyn Duplicate>> = vec![Box::new(Note(String::from("hi")))];
    println!("{}", items.len());
}
```

```text
error[E0038]: the trait `Duplicate` is not dyn compatible
  --> src/main.rs:14:28
   |
14 |     let items: Vec<Box<dyn Duplicate>> = vec![Box::new(Note(String::from("hi")))];
   |                            ^^^^^^^^^ `Duplicate` is not dyn compatible
   |
note: for a trait to be dyn compatible it needs to allow building a vtable
      for more information, visit <https://doc.rust-lang.org/reference/items/traits.html#dyn-compatibility>
  --> src/main.rs:2:28
   |
 1 | trait Duplicate {
   |       --------- this trait is not dyn compatible...
 2 |     fn duplicate(&self) -> Self;
   |                            ^^^^ ...because method `duplicate` references the `Self` type in its return type
   = help: consider moving `duplicate` to another trait
   = help: only type `Note` implements `Duplicate`; consider using it directly instead.
```

И второе правило:

```text
error[E0038]: the trait `Logger` is not dyn compatible
  --> src/main.rs:20:24
   |
20 |     let boxed: Box<dyn Logger> = Box::new(ConsoleLogger);
   |                        ^^^^^^ `Logger` is not dyn compatible
   |
note: for a trait to be dyn compatible it needs to allow building a vtable
      for more information, visit <https://doc.rust-lang.org/reference/items/traits.html#dyn-compatibility>
  --> src/main.rs:5:8
   |
 3 | trait Logger {
   |       ------ this trait is not dyn compatible...
 4 |     fn log(&self, message: &str);
 5 |     fn log_value<T: Debug>(&self, value: T);
   |        ^^^^^^^^^ ...because method `log_value` has generic type parameters
   = help: consider moving `log_value` to another trait
   = help: only type `ConsoleLogger` implements `Logger`; consider using it directly instead.
```

**Что происходит:**

1. `Box<dyn Duplicate>` требует vtable для `Duplicate`.
2. Метод `duplicate` возвращает `Self`. За указателем `dyn Duplicate` никто не знает, что такое `Self`, поэтому ячейку для `duplicate` заполнить нельзя.
3. Компилятор отказывается с ошибкой `E0038` и точно сообщает, какой метод создаёт проблему и почему.

Есть два выхода. Можно переделать трейт (например, возвращать `Box<dyn Duplicate>`). Или оставить метод, но исключить его из vtable с помощью `where Self: Sized`. Тогда метод существует только для конкретных типов:

```rust
use std::fmt::Debug;

trait Logger {
    fn log(&self, message: &str);

    // Обобщённый метод нельзя вызвать через vtable,
    // поэтому мы исключаем его из трейт-объекта через `where Self: Sized`.
    fn log_value<T: Debug>(&self, value: T)
    where
        Self: Sized,
    {
        self.log(&format!("{:?}", value));
    }
}

struct ConsoleLogger;

impl Logger for ConsoleLogger {
    fn log(&self, message: &str) {
        println!("[console] {}", message);
    }
}

fn main() {
    let logger = ConsoleLogger;
    logger.log_value(42); // работает: вызов у конкретного типа

    let boxed: Box<dyn Logger> = Box::new(ConsoleLogger);
    boxed.log("hello from a trait object"); // работает: log есть в vtable
}
```

```text
[console] 42
[console] hello from a trait object
```

**Что происходит:**

1. `log` — обычный метод, поэтому он попадает в vtable.
2. `log_value<T>` обобщённый, но строка `where Self: Sized` говорит: «этот метод только для типов с известным размером». У `dyn Logger` известного размера нет, поэтому метод не попадает в vtable, и трейт снова совместим с dyn.
3. `logger.log_value(42)` работает на конкретном `ConsoleLogger`.
4. `boxed.log(...)` работает на трейт-объекте. Вызов `boxed.log_value(1)` не скомпилировался бы.

## 7. Трейт-объекты и времена жизни

`Box<dyn Shape>` может хранить значение, внутри которого есть ссылки (структуру с полем `&'a str`). Времена жизни вы изучили в [Уроке 05.3](../05-rust-intermediate/03-lifetimes.md); вот единственное новое правило. В сигнатуре функции `Box<dyn Shape>` означает `Box<dyn Shape + 'static>`: «внутри нет недолговечных заимствований». Если значение что-то заимствует, нужно указать, как долго длится заимствование:

```rust
trait Shape {
    fn label(&self) -> String;
}

struct Tag<'a> {
    text: &'a str,
}

impl Shape for Tag<'_> {
    fn label(&self) -> String {
        self.text.to_string()
    }
}

fn make_tag(text: &str) -> Box<dyn Shape> {
    Box::new(Tag { text })
}

fn main() {
    let text = String::from("local");
    let shape = make_tag(&text);
    println!("{}", shape.label());
}
```

```text
error: lifetime may not live long enough
  --> src/main.rs:16:5
   |
15 | fn make_tag(text: &str) -> Box<dyn Shape> {
   |                   - let's call the lifetime of this reference `'1`
16 |     Box::new(Tag { text })
   |     ^^^^^^^^^^^^^^^^^^^^^^ returning this value requires that `'1` must outlive `'static`
   |
help: to declare that the trait object captures data from argument `text`, you can add an explicit `'_` lifetime bound
   |
15 | fn make_tag(text: &str) -> Box<dyn Shape + '_> {
   |                                          ++++
```

Исправление — время жизни заимствования, записанное после имени трейта: `Box<dyn Shape + 'a>`. В сигнатуре с единственной входной ссылкой можно написать `'_` (анонимное время жизни):

```rust
trait Shape {
    fn label(&self) -> String;
}

struct Tag<'a> {
    text: &'a str,
}

impl Shape for Tag<'_> {
    fn label(&self) -> String {
        self.text.to_string()
    }
}

fn make_tag<'a>(text: &'a str) -> Box<dyn Shape + 'a> {
    Box::new(Tag { text })
}

fn main() {
    let text = String::from("local");
    let shape = make_tag(&text);
    println!("{}", shape.label());
}
```

```text
local
```

**Что происходит:**

1. `Tag<'a>` хранит заимствованную `&'a str`.
2. `Box<dyn Shape + 'a>` говорит: «фигура, которая может хранить заимствования, живущие не меньше, чем `'a`».
3. Возвращённая коробка не может пережить `text`. Компилятор проверяет это в `main`.

После `dyn` можно добавлять и другие маркеры, например `Box<dyn Shape + Send>`, чтобы сказать, что значение можно передать в другой поток ([Урок 05.10](../05-rust-intermediate/10-threads-channels-and-shared-state.md)). Глубже во времена жизни мы погрузимся в [Уроке 06.5](05-lifetimes-in-depth.md).

## 8. Замыкания и плагины как трейт-объекты

У каждого замыкания свой уникальный тип, поэтому два замыкания нельзя положить в один `Vec` как есть. Трейт-объект решает эту проблему: замыкания реализуют `Fn`, `FnMut` или `FnOnce`, а это тоже трейты.

```rust
fn main() {
    let steps: Vec<Box<dyn Fn(i32) -> i32>> = vec![
        Box::new(|x| x + 1),
        Box::new(|x| x * 10),
        Box::new(|x| x - 3),
    ];

    let mut value = 4;
    for step in &steps {
        value = step(value);
    }
    println!("result: {}", value);
}
```

```text
result: 47
```

**Что происходит:**

1. `Box<dyn Fn(i32) -> i32>` означает «коробка с любым замыканием, которое принимает `i32` и возвращает `i32`».
2. У трёх замыканий три разных типа, но все они хранятся в одном `Vec`.
3. Цикл применяет их по порядку: 4 + 1 = 5, 5 * 10 = 50, 50 - 3 = 47.

Пример побольше — список плагинов. У каждого плагина есть имя и метод `run`, который может менять собственное состояние плагина (`&mut self`). `Debug` здесь — *супертрейт*: каждый `Plugin` обязан реализовать ещё и `Debug`. Благодаря этому мы можем напечатать весь список через `{:?}`.

```rust
use std::fmt::Debug;

trait Plugin: Debug {
    fn name(&self) -> &str;
    fn run(&mut self, input: &str) -> String;
}

#[derive(Debug)]
struct Upper;

#[derive(Debug)]
struct Counter {
    calls: u32,
}

impl Plugin for Upper {
    fn name(&self) -> &str {
        "upper"
    }
    fn run(&mut self, input: &str) -> String {
        input.to_uppercase()
    }
}

impl Plugin for Counter {
    fn name(&self) -> &str {
        "counter"
    }
    fn run(&mut self, input: &str) -> String {
        self.calls += 1;
        format!("{} (call #{})", input, self.calls)
    }
}

fn main() {
    let mut plugins: Vec<Box<dyn Plugin>> = vec![Box::new(Upper), Box::new(Counter { calls: 0 })];

    for round in 1..=2 {
        for plugin in plugins.iter_mut() {
            let out = plugin.run("hello");
            println!("round {} {:>8}: {}", round, plugin.name(), out);
        }
    }
    println!("{:?}", plugins);
}
```

```text
round 1    upper: HELLO
round 1  counter: hello (call #1)
round 2    upper: HELLO
round 2  counter: hello (call #2)
[Upper, Counter { calls: 2 }]
```

**Что происходит:**

1. `trait Plugin: Debug` означает «чтобы быть `Plugin`, тип должен ещё и быть `Debug`». Поэтому `Box<dyn Plugin>` тоже `Debug`.
2. `plugins.iter_mut()` даёт `&mut Box<dyn Plugin>`. Мы можем вызвать `run`, потому что он принимает `&mut self`.
3. `Counter` хранит состояние: `calls` растёт с каждым вызовом. У `Upper` состояния нет.
4. `{:?}` для вектора печатает настоящую структуру каждого плагина: `Upper` и `Counter { calls: 2 }`.

> 📏 **Соглашение:** делайте трейт для трейт-объектов небольшим. Два-три метода — нормально. Большой трейт трудно реализовывать и трудно сохранять совместимым с dyn.

## Частые ошибки

### 1. Разные типы в `Vec` без `Box<dyn ...>`

Вы видели это в разделе 1: `E0308 mismatched types`. Объявите вектор как `Vec<Box<dyn Shape>>` и оберните каждый элемент в `Box::new(...)`.

### 2. Возврат `dyn Trait` по значению

```text
error[E0746]: return type cannot be a trait object without pointer indirection
  --> src/main.rs:13:20
   |
13 | fn make_shape() -> dyn Shape {
   |                    ^^^^^^^^^ doesn't have a size known at compile-time
   |
help: consider returning an `impl Trait` instead of a `dyn Trait`
   |
13 - fn make_shape() -> dyn Shape {
13 + fn make_shape() -> impl Shape {
   |
help: alternatively, box the return type, and wrap all of the returned values in `Box::new`
   |
13 ~ fn make_shape() -> Box<dyn Shape> {
14 ~     Box::new(Circle(1.0))
   |
```

Компилятор даже предлагает оба исправления. Функция возвращает `dyn Shape`, у которого нет известного размера, а у результата функции он должен быть. Верните `Box<dyn Shape>` (и напишите `Box::new(Circle(1.0))`) или `impl Shape`, если конкретный тип только один.

### 3. `impl Trait` — это не трейт-объект

`impl Shape` в возвращаемом типе означает «один конкретный тип, который я не хочу называть». Это всё равно **один** тип:

```text
error[E0308]: `if` and `else` have incompatible types
  --> src/main.rs:21:37
   |
21 |     if round { Circle(1.0) } else { Rect(2.0, 3.0) }
   |                -----------          ^^^^^^^^^^^^^^ expected `Circle`, found `Rect`
   |                |
   |                expected because of this
   |
help: you could change the return type to be a boxed trait object
   |
20 - fn make_shape(round: bool) -> impl Shape {
20 + fn make_shape(round: bool) -> Box<dyn Shape> {
   |
help: if you change the return type to expect trait objects, box the returned expressions
   |
21 |     if round { Box::new(Circle(1.0)) } else { Box::new(Rect(2.0, 3.0)) }
   |                +++++++++           +          +++++++++              +
```

Если функция должна вернуть круг в одной ветке и прямоугольник в другой, верните `Box<dyn Shape>` и упакуйте каждое значение в коробку.

### 4. Трейт, не совместимый с dyn

`E0038`, см. раздел 6. Прочитайте строку `...because method ... references the Self type`: в ней указан метод, который нужно изменить.

### 5. Забыли про время жизни заимствованного значения

`lifetime may not live long enough ... must outlive 'static`, см. раздел 7. Добавьте `+ 'a` или `+ '_` к типу трейт-объекта.

## Попробуйте сами

Соберите небольшой реестр плагинов и сохраните его в коммите.

1. Откройте проект: `cd ~/projects/rust/code/p06-trait-objects`.
2. Поместите пример с `Plugin` из раздела 8 в `src/main.rs`.
3. Запустите: `cargo run`. Вы должны увидеть четыре строки, как в выводе выше, а затем список `Debug`.
4. Добавьте третий плагин, `Reverse`, который возвращает входной текст задом наперёд (подсказка: `input.chars().rev().collect()`). Добавьте его в вектор. Запустите снова.
5. Проверьте код: `cargo fmt`, затем `cargo clippy`. Ни одна команда не должна ругаться.
6. Сохраните работу в Git (см. [Урок 01.3](../01-git-and-version-control/03-your-first-repository-and-commits.md) и стиль сообщений из [Урока 01.9](../01-git-and-version-control/09-teamwork-pull-requests-and-conventions.md)):

```sh
git add code/p06-trait-objects
git commit -m "feat(p06): plugin registry with trait objects"
```

> 💡 **Совет:** один небольшой коммит на каждое выполненное упражнение даёт безопасную точку, к которой можно вернуться. Если позже что-то сломается, `git diff` покажет, что изменилось (Урок 01.4).

## Упражнения

### Упражнение 1 — Зоопарк (легко)

Напишите трейт `Animal` с методами `name(&self) -> String` и `sound(&self) -> String`. Реализуйте его для `Dog`, `Cat` и `Cow` с полем `loud: bool` (громкая корова говорит `MOO`, тихая — `moo`). Положите по одному экземпляру в `Vec<Box<dyn Animal>>` и напечатайте `Dog says woof` и так далее.

<details>
<summary>Подсказка</summary>

Тип вектора — `Vec<Box<dyn Animal>>`. Элементы создавайте через `Box::new(Dog)`. Проходите циклом `for animal in &zoo`.

</details>

<details>
<summary>Ответ</summary>

```rust
trait Animal {
    fn name(&self) -> String;
    fn sound(&self) -> String;
}

struct Dog;
struct Cat;
struct Cow {
    loud: bool,
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
        String::from("Cat")
    }
    fn sound(&self) -> String {
        String::from("meow")
    }
}

impl Animal for Cow {
    fn name(&self) -> String {
        String::from("Cow")
    }
    fn sound(&self) -> String {
        if self.loud {
            String::from("MOO")
        } else {
            String::from("moo")
        }
    }
}

fn main() {
    let zoo: Vec<Box<dyn Animal>> =
        vec![Box::new(Dog), Box::new(Cat), Box::new(Cow { loud: true })];
    for animal in &zoo {
        println!("{} says {}", animal.name(), animal.sound());
    }
}
```

```text
Dog says woof
Cat says meow
Cow says MOO
```

Почему это работает: у каждой структуры своя `impl Animal`. Коробки скрывают настоящие типы, а каждый вызов `animal.sound()` идёт через vtable настоящего типа.

</details>

### Упражнение 2 — Тест для писателя отчёта (средне)

Возьмите `write_report` из раздела 3. Напишите тест, который вызывает её с `Vec<u8>` и проверяет, что текст в точности равен `"report for bob\nstatus: ok\n"`.

<details>
<summary>Подсказка</summary>

Создайте `let mut buffer: Vec<u8> = Vec::new();`, передайте `&mut buffer`, затем превратите байты в `String` через `String::from_utf8(buffer).unwrap()` и сравните с помощью `assert_eq!`.

</details>

<details>
<summary>Ответ</summary>

```rust
use std::io::Write;

fn write_report(out: &mut dyn Write, user: &str) -> std::io::Result<()> {
    writeln!(out, "report for {}", user)?;
    writeln!(out, "status: ok")?;
    Ok(())
}

fn main() {
    let mut stdout = std::io::stdout();
    write_report(&mut stdout, "ana").unwrap();
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn report_has_two_lines() {
        let mut buffer: Vec<u8> = Vec::new();
        write_report(&mut buffer, "bob").unwrap();
        let text = String::from_utf8(buffer).unwrap();
        assert_eq!(text, "report for bob\nstatus: ok\n");
    }
}
```

```text

running 1 test
.
test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

Почему это работает: `Vec<u8>` реализует `Write`, поэтому его можно передать как `&mut dyn Write`. Функция не знает и не интересуется, что «файл» — это просто память.

</details>

### Упражнение 3 — Сделать трейт совместимым с dyn (средне)

У этого трейта есть конструктор `unit() -> Self`, поэтому его нельзя использовать как `dyn Shape`. Измените **только трейт** так, чтобы `Vec<Box<dyn Shape>>` работал и `Square::unit()` по-прежнему работал. Программа должна напечатать `total: 10`.

```rust
trait Shape {
    fn area(&self) -> f64;
    fn unit() -> Self;
}
```

<details>
<summary>Подсказка</summary>

Добавьте `where Self: Sized` к методу, который возвращает `Self` и не имеет получателя.

</details>

<details>
<summary>Ответ</summary>

```rust
trait Shape {
    fn area(&self) -> f64;

    // Не входит в vtable: только для конкретных типов.
    fn unit() -> Self
    where
        Self: Sized;
}

struct Square(f64);

impl Shape for Square {
    fn area(&self) -> f64 {
        self.0 * self.0
    }
    fn unit() -> Self {
        Square(1.0)
    }
}

fn main() {
    let shapes: Vec<Box<dyn Shape>> = vec![Box::new(Square::unit()), Box::new(Square(3.0))];
    let total: f64 = shapes.iter().map(|s| s.area()).sum();
    println!("total: {}", total);
}
```

```text
total: 10
```

Почему это работает: `where Self: Sized` убирает `unit` из vtable. Вызов `Square::unit()` у конкретного типа по-прежнему допустим. Площади: 1 * 1 + 3 * 3 = 10.

</details>

### Упражнение 4 — Шина событий (сложно)

Напишите структуру `EventBus`, которая хранит список обработчиков. Обработчик — любое замыкание `Fn(&str) -> String`. Нужны три функции: `new()`, `subscribe(&mut self, handler)` и `publish(&self, event: &str) -> Vec<String>`, которая запускает все обработчики по порядку и возвращает их результаты. Напишите два теста: с двумя обработчиками (проверьте порядок результатов) и без обработчиков (результат пуст).

<details>
<summary>Подсказка</summary>

Храните `Vec<Box<dyn Fn(&str) -> String>>`. Clippy говорит, что этот тип очень сложный; дайте ему имя: `type Handler = Box<dyn Fn(&str) -> String>;` (псевдонимы типов объясняются в [Уроке 06.2](02-advanced-traits-and-types.md)). В `publish` используйте `self.handlers.iter().map(|h| h(event)).collect()`. Замыканию, которое использует `String` снаружи, нужно ключевое слово `move`.

</details>

<details>
<summary>Ответ</summary>

```rust
// Имя для длинного типа (Clippy предупреждает о слишком сложных типах).
type Handler = Box<dyn Fn(&str) -> String>;

struct EventBus {
    handlers: Vec<Handler>,
}

impl EventBus {
    fn new() -> Self {
        EventBus {
            handlers: Vec::new(),
        }
    }

    fn subscribe(&mut self, handler: Handler) {
        self.handlers.push(handler);
    }

    fn publish(&self, event: &str) -> Vec<String> {
        self.handlers.iter().map(|h| h(event)).collect()
    }
}

fn main() {
    let mut bus = EventBus::new();
    let prefix = String::from("log");
    bus.subscribe(Box::new(move |e| format!("{}: {}", prefix, e)));
    bus.subscribe(Box::new(|e| e.to_uppercase()));

    for line in bus.publish("started") {
        println!("{}", line);
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn every_handler_runs_in_order() {
        let mut bus = EventBus::new();
        bus.subscribe(Box::new(|e| format!("a:{}", e)));
        bus.subscribe(Box::new(|e| format!("b:{}", e)));
        assert_eq!(bus.publish("x"), vec!["a:x", "b:x"]);
    }

    #[test]
    fn no_handlers_means_no_output() {
        let bus = EventBus::new();
        assert!(bus.publish("x").is_empty());
    }
}
```

```text
log: started
STARTED
```

```text

running 2 tests
..
test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

Почему это работает: у каждого замыкания свой тип, но `Box<dyn Fn(&str) -> String>` его скрывает. `move` даёт первому замыканию собственную копию `prefix`, так что замыкание не зависит от локальной переменной, которая может исчезнуть.

</details>

## Вопросы для самопроверки

1. Почему `vec![Circle(..), Rect(..)]` не компилируется, а `Vec<Box<dyn Shape>>` — компилируется?

<details>
<summary>Ответ</summary>

У `Vec<T>` один тип элементов, а `Circle` и `Rect` — разные типы. `Box<dyn Shape>` — это один тип («толстый» указатель на «какой-то `Shape`»), поэтому коробки с разными конкретными типами помещаются в один вектор.

</details>

2. Сколько байт занимает `&dyn Trait` на 64-битной машине и что внутри?

<details>
<summary>Ответ</summary>

16 байт: 8 — адрес значения и 8 — адрес vtable.

</details>

3. Что содержит vtable?

<details>
<summary>Ответ</summary>

Для одного типа и одного трейта: функцию, которая уничтожает значение (drop), размер, выравнивание и адреса методов трейта.

</details>

4. Чем отличается статическая диспетчеризация от динамической?

<details>
<summary>Ответ</summary>

При статической диспетчеризации (обобщения, `impl Trait`) компилятор знает конкретный тип и выбирает метод на этапе компиляции, создавая по одной копии кода на тип. При динамической (`dyn Trait`) метод ищется в vtable во время работы, а копия кода одна.

</details>

5. Почему `trait Duplicate { fn duplicate(&self) -> Self; }` не работает как `dyn Duplicate`?

<details>
<summary>Ответ</summary>

Метод возвращает `Self` — конкретный тип, который трейт-объект «забыл». В vtable не может быть ячейки с неизвестным возвращаемым типом, поэтому трейт не совместим с dyn (ошибка `E0038`).

</details>

6. У трейта есть обобщённый метод `fn log_value<T: Debug>(&self, v: T)`. Как сохранить совместимость трейта с dyn?

<details>
<summary>Ответ</summary>

Добавить к этому методу `where Self: Sized`. Тогда он существует только для конкретных типов и не попадает в vtable.

</details>

7. Что означает `Box<dyn Shape>` в сигнатуре функции, если вы не пишете никакого времени жизни?

<details>
<summary>Ответ</summary>

`Box<dyn Shape + 'static>`: внутри значения не должно быть недолговечных заимствований. Если они есть, пишите `Box<dyn Shape + 'a>` (или `'_`).

</details>

## Итоги

- Трейт-объект `dyn Trait` — это значение «какого-то типа, реализующего `Trait`», причём тип известен только во время работы.
- У трейт-объекта нет известного размера, поэтому он живёт за указателем: `Box<dyn T>`, `&dyn T`, `&mut dyn T`, `Rc<dyn T>`.
- Указатель на трейт-объект — «толстый»: адрес данных плюс адрес vtable (16 байт на 64-битных системах).
- Динамическая диспетчеризация ищет метод в vtable во время работы. Статическая (обобщения) решается при компиляции и допускает встраивание.
- По умолчанию используйте обобщения. `dyn Trait` — для списков разных типов, плагинов и выбора во время работы.
- Трейт должен быть совместим с dyn: без `Self` в возвращаемых типах, без обобщённых методов, у методов есть получатель. `where Self: Sized` прячет метод от vtable.
- `Box<dyn T>` означает `Box<dyn T + 'static>`. Добавляйте `+ 'a`, когда значение заимствует данные.

## Источники и что почитать

- [The Rust Programming Language — 18.2 Using Trait Objects](https://doc.rust-lang.org/book/ch18-02-trait-objects.html) — официальная глава о трейт-объектах и динамической диспетчеризации.
- [The Rust Reference — Trait objects](https://doc.rust-lang.org/reference/types/trait-object.html) — точные правила, включая границы времени жизни по умолчанию.
- [The Rust Reference — Traits (dyn compatibility)](https://doc.rust-lang.org/reference/items/traits.html) — полный список правил совместимости с dyn.
- [Rust by Example — Returning Traits with dyn](https://doc.rust-lang.org/rust-by-example/trait/dyn.html) — короткий пример `Box<dyn Trait>` как возвращаемого типа.
- [std::mem::size_of_val](https://doc.rust-lang.org/std/mem/fn.size_of_val.html) — как измерить размер значения за указателем.

---

[🏠 Начало](../START-HERE.md) · [← Этап 06 — Rust: продвинутый уровень](README.md) · [📋 Этап 06](README.md) · [06.2 Продвинутые трейты и типы →](02-advanced-traits-and-types.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
