<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← Этап 05 — Rust: средний уровень](README.md) · [📋 Этап 05](README.md) · [05.2 Трейты →](02-traits.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/05-rust-intermediate/01-generics.md) <!-- nav -->

# Урок 05.1 — Обобщённые типы

> **Этап 05 — Rust: средний уровень** · Урок 1 из 14 · ⏱ около 4 часов (2 занятия)

## Что вы узнаете

- Почему копировать одну и ту же функцию для каждого типа — плохая идея и как это решают обобщения (generics).
- Как писать обобщённые функции, структуры, перечисления и методы с параметрами типа, например `T`.
- Что такое *ограничение трейта* (trait bound) и почему для сравнения двух значений типа `T` нужно `T: PartialOrd`.
- Почему обобщённый код не медленнее написанного вручную (мономорфизация) и чем за это приходится платить.
- Как писать *turbofish* `::<>` и как читать самые частые ошибки компилятора, связанные с обобщениями.
- Как начать практический проект этого этапа и сохранять каждый урок в Git в отдельной ветке.

## Перед началом

- Вы прошли [Урок 04.12 — Контрольная точка](../04-structs-enums-collections-errors/12-checkpoint.md).
- Вы знаете структуры ([Урок 04.1](../04-structs-enums-collections-errors/01-structs.md)), методы ([Урок 04.2](../04-structs-enums-collections-errors/02-methods-and-associated-functions.md)), `Option` ([Урок 04.4](../04-structs-enums-collections-errors/04-option-and-match.md)) и векторы ([Урок 04.7](../04-structs-enums-collections-errors/07-vectors.md)).
- У вас есть Git-репозиторий для учебных работ (из [Урока 01.10](../01-git-and-version-control/10-project-your-learning-repo-and-checkpoint.md)), а папка курса — `~/projects/rust`.

## Новые слова

| Слово | Значение |
|---|---|
| **обобщённый** (generic) | Код, написанный один раз и работающий с разными типами. Тип — это «пропуск», который компилятор заполняет позже. |
| **параметр типа** | Имя такого пропуска в угловых скобках: `T` в `fn largest<T>(…)`. |
| **трейт** (trait) | Список умений (методов), которые может иметь тип, например «можно сравнивать» или «можно печатать». Урок 05.2 разбирает трейты полностью; сегодня нужна только идея. |
| **ограничение трейта** (trait bound) | Правило, которое говорит, какими умениями должен обладать параметр типа: `T: PartialOrd` означает «`T` можно сравнивать через `<` и `>`». |
| **`PartialOrd`** | Стандартный трейт для типов, которые можно упорядочивать операторами `<`, `>`, `<=` и `>=`. |
| **мономорфизация** | Шаг, на котором компилятор делает отдельную специализированную копию обобщённого кода для каждого конкретного используемого типа. |
| **конкретный тип** | Настоящий тип, например `i32` или `String`, в отличие от пропуска вроде `T`. |
| **turbofish** | Запись `::<Тип>`, которая сообщает компилятору, какой тип взять для параметра типа: `"5".parse::<i32>()`. |
| **константный параметр** (const generic) | Обобщённый параметр, который является *значением* (числом), а не типом, например длина `N` массива `[i32; N]`. |
| **пример Cargo** (example) | Маленькая дополнительная программа в папке `examples/` пакета. Запускается командой `cargo run --example имя`. |
| **ветка** (branch) | Отдельная линия работы в Git (Урок 01.6). |

## 1. Проблема: один и тот же код для каждого типа

Допустим, нужна функция, которая возвращает наибольшее число в списке. Вот она для чисел `i32` и
ещё раз для чисел `f64`:

```rust
fn largest_i32(list: &[i32]) -> &i32 {
    let mut largest = &list[0];
    for item in list {
        if item > largest {
            largest = item;
        }
    }
    largest
}

fn largest_f64(list: &[f64]) -> &f64 {
    let mut largest = &list[0];
    for item in list {
        if item > largest {
            largest = item;
        }
    }
    largest
}

fn main() {
    let numbers = [34, 50, 25, 100, 65];
    let temperatures = [21.5, 19.0, 23.8];
    println!("largest number: {}", largest_i32(&numbers));
    println!("largest temperature: {}", largest_f64(&temperatures));
}
```

**Что происходит:**

1. `&[i32]` — это *срез* (Урок 03.6): заимствованный взгляд на список значений `i32`.
2. `largest` сначала ссылается на первый элемент. Цикл `for` сравнивает с ним каждый элемент и
   переставляет ссылку, когда находит больший.
3. `largest_f64` — **тот же самый код**. Изменился только тип.

```text
largest number: 100
largest temperature: 23.8
```

Две функции с одинаковым телом — это проблема. Если вы нашли ошибку, её придётся исправлять
дважды. Если понадобятся ещё `char`, `u8`, `String` — копировать придётся снова. Инженерный принцип
**DRY** (Don't Repeat Yourself, «не повторяйся») гласит: одна идея должна жить в одном месте.
Обобщения — это способ, которым Rust следует DRY для типов.

## 2. Обобщённые функции

У **обобщённой** функции есть *параметр типа* — имя для «какого-то типа, который выберем позже».
Его записывают в угловых скобках после имени функции:

```rust
fn largest<T: PartialOrd>(list: &[T]) -> &T {
    let mut largest = &list[0];
    for item in list {
        if item > largest {
            largest = item;
        }
    }
    largest
}

fn main() {
    let numbers = [34, 50, 25, 100, 65];
    let temperatures = [21.5, 19.0, 23.8];
    let letters = ['q', 'z', 'a'];
    let words = ["pear", "apple", "zebra"];
    println!("{}", largest(&numbers));
    println!("{}", largest(&temperatures));
    println!("{}", largest(&letters));
    println!("{}", largest(&words));
}
```

**Что происходит:**

1. `<T: PartialOrd>` объявляет параметр типа с именем `T`. `T` — просто имя; обычно берут `T`
   от слова «type».
2. `: PartialOrd` — это **ограничение трейта**. Оно говорит: «`T` может быть любым типом, *но только
   если* значения этого типа можно сравнивать через `<` и `>`». Целые числа, дробные числа, `char` и
   `&str` — можно.
3. `list: &[T]` — срез значений `T`, а `-> &T` возвращает ссылку на одно из них.
4. Тело — тот же код, что и раньше. Внутри него компилятор знает только то, что разрешает
   ограничение; именно поэтому сравнение `>` принимается.
5. В `main` мы вызываем `largest` с четырьмя разными типами. Rust сам определяет `T` по аргументу
   (это называется *выводом типов*).

```text
100
23.8
z
zebra
```

> 💡 **Совет:** читайте `fn largest<T: PartialOrd>(list: &[T]) -> &T` как предложение: «для любого
> типа `T`, который можно упорядочить, возьми срез из `T` и верни ссылку на `T`».

### Зачем нужно ограничение?

Попробуйте убрать `: PartialOrd`:

```rust
fn largest<T>(list: &[T]) -> &T {
    let mut largest = &list[0];
    for item in list {
        if item > largest {
            largest = item;
        }
    }
    largest
}
```

```text
error[E0369]: binary operation `>` cannot be applied to type `&T`
 --> examples/nobound.rs:4:17
  |
4 |         if item > largest {
  |            ---- ^ ------- &T
  |            |
  |            &T
  |
help: consider restricting type parameter `T` with trait `PartialOrd`
  |
1 | fn largest<T: std::cmp::PartialOrd>(list: &[T]) -> &T {
  |             ++++++++++++++++++++++
```

**Что это значит:** `T` может быть *любым* типом, в том числе таким, который нельзя сравнивать
(например, структурой без упорядочивания). Компилятор не хочет гадать. Он проверяет обобщённую
функцию один раз — для всех возможных типов, — поэтому разрешает только то, что обещают ограничения.
Строка `help:` подсказывает исправление: добавить ограничение.

Это важное свойство Rust: он проверяет обобщённую функцию, когда вы её
**пишете**, а не когда вызываете. Если она компилируется, то верна для любого типа, удовлетворяющего
ограничениям.

## 3. Обобщённые структуры

У структуры тоже могут быть параметры типа. `Point`, который работает и с целыми, *и* с дробными
числами:

```rust
#[derive(Debug)]
struct Point<T> {
    x: T,
    y: T,
}

impl<T> Point<T> {
    fn x(&self) -> &T {
        &self.x
    }
}

impl Point<f64> {
    fn distance_from_origin(&self) -> f64 {
        (self.x * self.x + self.y * self.y).sqrt()
    }
}

fn main() {
    let a = Point { x: 3, y: 4 };
    let b = Point { x: 3.0, y: 4.0 };
    let c = Point { x: "left", y: "right" };
    println!("{:?} {:?} {:?}", a, b, c);
    println!("a.x = {}", a.x());
    println!("distance of b = {}", b.distance_from_origin());
}
```

**Что происходит:**

1. `#[derive(Debug)]` (Урок 04.1) позволяет печатать структуру через `{:?}`.
2. `struct Point<T>` объявляет один параметр типа. Оба поля используют его, поэтому `x` и `y`
   должны быть **одного** типа.
3. `impl<T> Point<T>` значит: «для любого типа `T` вот методы `Point<T>`». `<T>` после `impl`
   обязательно. Оно *объявляет* `T`, а `Point<T>` после него — *использует*.
4. У `impl Point<f64>` нет `<T>`: эти методы существуют **только** для `Point<f64>`. У `Point<i32>`
   метода `distance_from_origin` нет.
5. `a` — это `Point<i32>`, `b` — `Point<f64>`, `c` — `Point<&str>`. Компилятор подставляет `T` для
   каждого из них.

```text
Point { x: 3, y: 4 } Point { x: 3.0, y: 4.0 } Point { x: "left", y: "right" }
a.x = 3
distance of b = 5
```

### Больше одного параметра типа

Если `x` и `y` могут быть разных типов, используйте два параметра:

```rust
#[derive(Debug)]
struct Point<X, Y> {
    x: X,
    y: Y,
}

impl<X, Y> Point<X, Y> {
    fn swap(self) -> Point<Y, X> {
        Point { x: self.y, y: self.x }
    }
}

fn main() {
    let p = Point { x: 5, y: "five" };
    println!("{:?}", p);
    let q = p.swap();
    println!("{:?}", q);
}
```

**Что происходит:** `p` — это `Point<i32, &str>`. `swap` поглощает `p` (`self`, а не `&self`,
см. Урок 04.2) и строит `Point<&str, i32>`: типы меняются местами вместе со значениями.

```text
Point { x: 5, y: "five" }
Point { x: "five", y: 5 }
```

## 4. Обобщённые перечисления и стандартная библиотека

Обобщённые типы вы использовали ещё на Этапе 04, не зная этого названия:

| Тип | Определение (упрощённо) | Смысл |
|---|---|---|
| `Option<T>` | `enum Option<T> { Some(T), None }` | может быть `T`, а может ничего |
| `Result<T, E>` | `enum Result<T, E> { Ok(T), Err(E) }` | `T` при успехе, `E` при ошибке |
| `Vec<T>` | растущий список | список из `T` |

`Vec<i32>` и `Vec<String>` — два разных типа, сделанных из одного обобщённого определения.

Теперь построим собственный обобщённый контейнер: **стек**. Стек — это стопка, в которую добавляют
(`push`) и из которой забирают (`pop`) только сверху, как тарелки. Он построен на `Vec<T>`:

```rust
#[derive(Debug)]
struct Stack<T> {
    items: Vec<T>,
}

impl<T> Stack<T> {
    fn new() -> Self {
        Stack { items: Vec::new() }
    }

    fn push(&mut self, item: T) {
        self.items.push(item);
    }

    fn pop(&mut self) -> Option<T> {
        self.items.pop()
    }

    fn peek(&self) -> Option<&T> {
        self.items.last()
    }

    fn len(&self) -> usize {
        self.items.len()
    }

    fn is_empty(&self) -> bool {
        self.items.is_empty()
    }
}

fn main() {
    let mut numbers: Stack<i32> = Stack::new();
    numbers.push(10);
    numbers.push(20);
    println!("peek = {:?}", numbers.peek());
    println!("pop = {:?}", numbers.pop());
    println!("len = {}", numbers.len());

    let mut words = Stack::new();
    words.push(String::from("hello"));
    println!("{:?}", words);
    println!("empty? {}", words.is_empty());
    words.pop();
    println!("pop on empty = {:?}", words.pop());
}
```

**Что происходит:**

1. `Stack<T>` хранит `Vec<T>`. `Self` внутри `impl<T> Stack<T>` означает `Stack<T>`.
2. `pop` возвращает `Option<T>`: `None`, когда стек пуст. Ни падения, ни специальных значений.
   Это безопасный стиль из Урока 04.4.
3. `peek` возвращает `Option<&T>` — *ссылку* на верхний элемент, чтобы можно было посмотреть, не
   забирая.
4. `let mut numbers: Stack<i32> = Stack::new();` сообщает компилятору `T = i32`. Для `words` мы
   ничего не писали; Rust видит `push(String::from(…))` и выводит `T = String`.
5. Последняя строка достаёт элемент из пустого стека и получает `None`.

```text
peek = Some(20)
pop = Some(20)
len = 1
Stack { items: ["hello"] }
empty? false
pop on empty = None
```

> 📏 **Соглашение:** функция `new`, которая не принимает `self` и возвращает `Self`, — стандартный
> конструктор в Rust. Clippy (программа, которая читает ваш код и предлагает улучшения; вы будете
> пользоваться ею начиная с Урока 05.6) рекомендует для публичных типов добавлять метод `is_empty`
> рядом с `len`, как в стандартной библиотеке. Поэтому мы написали оба.

## 5. Ещё об ограничениях трейтов

Несколько умений можно потребовать через `+`:

```rust
use std::fmt::Display;

fn show_largest<T: PartialOrd + Display>(a: T, b: T) {
    if a > b {
        println!("the larger value is {a}");
    } else {
        println!("the larger value is {b}");
    }
}

fn show_pair<T, U>(first: T, second: U)
where
    T: Display,
    U: Display + Clone,
{
    println!("({first}, {second})");
}

fn describe(item: impl Display) {
    println!("item: {item}");
}

fn main() {
    show_largest(3, 9);
    show_largest("apple", "banana");
    show_pair(1, "one");
    describe(2.5);
    describe('x');
}
```

**Что происходит:**

1. `Display` — стандартный трейт «можно печатать через `{}`». `T: PartialOrd + Display` значит, что
   тип должен быть и упорядочиваемым, **и** печатаемым.
2. Когда ограничений становится много, вынесите их в **блок `where`** под сигнатурой. Смысл тот же,
   но имя функции и аргументы читаются легко.
3. `item: impl Display` — короткая запись (она называется *`impl Trait` в позиции аргумента*) для
   `fn describe<T: Display>(item: T)`. Используйте её, когда имя типа не нужно.
4. `show_largest(3, 9)` и `show_largest("apple", "banana")` используют два разных типа. Оба
   аргумента должны быть **одним и тем же** `T`: `show_largest(3, "a")` не скомпилируется.

```text
the larger value is 9
the larger value is banana
(1, one)
item: 2.5
item: x
```

### Ограничение `Copy`

А если `largest` должна вернуть само значение, а не ссылку? Первая идея не работает:

```rust
fn largest<T: PartialOrd>(list: &[T]) -> T {
    let mut largest = list[0];
    for &item in list {
        if item > largest {
            largest = item;
        }
    }
    largest
}
```

```text
error[E0508]: cannot move out of type `[T]`, a non-copy slice
 --> examples/nocopy.rs:2:23
  |
2 |     let mut largest = list[0];
  |                       ^^^^^^^
  |                       |
  |                       cannot move out of here
  |                       move occurs because `list[_]` has type `T`, which does not implement the `Copy` trait
```

**Что это значит:** это правило владения из Урока 03.3. `list[0]` *переместило бы* значение из
заимствованного среза, а для типов без `Copy` это запрещено. Решение — ограничение `Copy`: принимаются
только типы, которые дёшево копируются (числа, `char`, …).

```rust
fn largest<T: PartialOrd + Copy>(list: &[T]) -> T {
    let mut largest = list[0];
    for &item in list {
        if item > largest {
            largest = item;
        }
    }
    largest
}

fn main() {
    println!("{}", largest(&[1, 5, 2]));
    println!("{}", largest(&[0.5, -1.0]));
}
```

```text
5
0.5
```

Цена: `largest` больше не работает со `String`, потому что `String` не `Copy`. Версия со ссылкой
(`-> &T`) универсальнее. Выбор между ними — обычное проектное решение: *что я требую от вызывающего
и что отдаю обратно?*

### А если список пуст?

`&list[0]` вызывает панику, если срез пуст. Более аккуратная версия возвращает `Option`:

```rust
fn largest<T: PartialOrd>(list: &[T]) -> Option<&T> {
    let mut iter = list.iter();
    let mut largest = iter.next()?;
    for item in iter {
        if item > largest {
            largest = item;
        }
    }
    Some(largest)
}

fn main() {
    println!("{:?}", largest(&[3, 8, 1]));
    let empty: [i32; 0] = [];
    println!("{:?}", largest(&empty));
}
```

**Что происходит:** `list.iter()` даёт *итератор* (Урок 05.5 объясняет их полностью; здесь это
«ходок» по списку). `iter.next()` возвращает первый элемент как `Option<&T>`, а `?` возвращает
`None` из нашей функции, если список был пуст (оператор `?` из Урока 04.10 работает и с `Option`).
Затем цикл `for` проходит по оставшимся элементам.

```text
Some(8)
None
```

> 🧪 **Тестирование:** обобщённую функцию нужно проверять **больше чем на одном типе** и на
> граничных случаях: пустой список, один элемент, равные элементы. Именно это делает Упражнение 4
> ниже.

## 6. Мономорфизация: обобщения ничего не стоят во время выполнения

CPU (Central Processing Unit, центральный процессор — чип, выполняющий инструкции) не знает, что
такое «параметр типа». Так как же выполняется `largest<T>`? Компилятор использует
**мономорфизацию** (от греческого: «создание единой формы»). Для каждого конкретного типа, который
вы используете, он пишет отдельную копию функции, где `T` заменён:

```text
 your code                         what the compiler generates
 ─────────                         ───────────────────────────
 fn largest<T>(...)  ──┬── largest_for_i32(&[i32]) -> &i32
                       ├── largest_for_f64(&[f64]) -> &f64
                       └── largest_for_char(&[char]) -> &char
 (one definition)                  (one copy for each type you call it with)
```

(Настоящие внутренние имена выглядят иначе; это лишь идея.)

Следствия:

- ⚡ **Производительность:** сгенерированные копии работают так же быстро, как написанные вручную
  `largest_i32` и `largest_f64` из начала урока. Во время выполнения нет ни проверки типов, ни
  поиска, ни обращения через посредника. В Rust это называют **абстракцией с нулевой стоимостью**
  (zero-cost abstraction): за удобство во время выполнения вы ничего не платите.
- Стоимость переезжает во **время компиляции и размер файла**: много типов — много копий, поэтому
  компиляция идёт дольше, а файл программы может вырасти. Для большинства программ это не проблема.
  Урок 06.1 покажет альтернативу (трейт-объекты), которая немного жертвует скоростью ради меньшего
  размера программы.

Размер обобщённого типа тоже зависит от его параметра типа. Его можно измерить через
`std::mem::size_of::<T>()`, которая возвращает число байт, занимаемое значением типа:

```rust
use std::mem::size_of;

#[allow(dead_code)]
struct Point<T> {
    x: T,
    y: T,
}

fn main() {
    println!("Point<u8>  = {} bytes", size_of::<Point<u8>>());
    println!("Point<i32> = {} bytes", size_of::<Point<i32>>());
    println!("Point<f64> = {} bytes", size_of::<Point<f64>>());
    println!("Option<u8>  = {} bytes", size_of::<Option<u8>>());
    println!("Option<i32> = {} bytes", size_of::<Option<i32>>());
}
```

**Что происходит:** у `Point<u8>` два однобайтовых поля (2 байта); у `Point<i32>` два
четырёхбайтовых (8); у `Point<f64>` два восьмибайтовых (16). Это действительно три разных типа.
`Option<u8>` требует ещё один байт, чтобы помнить «Some или None» (2 байта), а `Option<i32>`
требует выравнивающих байтов, чтобы число лежало ровно (8 байт). `#[allow(dead_code)]` убирает
предупреждение «поле никогда не читается», потому что эта программа только измеряет размеры.

```text
Point<u8>  = 2 bytes
Point<i32> = 8 bytes
Point<f64> = 16 bytes
Option<u8>  = 2 bytes
Option<i32> = 8 bytes
```

(Точные размеры таких типов, как `Option<i32>`, выбирает компилятор; на вашем Mac они именно такие.
Расположение в памяти подробно разбирает Этап 12.)

## 7. Turbofish и вывод типов

В большинстве случаев Rust выводит параметр типа сам. Иногда не может, и тогда его нужно назвать.
Способов два: указать тип у переменной или использовать **turbofish** `::<>` (похоже на рыбку):

```rust
fn main() {
    let a = "42".parse::<i32>().unwrap();
    let b: f64 = "2.5".parse().unwrap();
    let v = Vec::<u8>::new();
    let c = std::mem::size_of::<u64>();
    println!("{a} {b} {} {c}", v.len());
}
```

**Что происходит:**

1. `"42".parse::<i32>()` — `parse` обобщена по типу результата. Turbofish говорит `i32`.
2. `let b: f64 = …parse()` — указанный у `b` тип позволяет компилятору вывести всё остальное.
3. `Vec::<u8>::new()` — пустой вектор байтов.
4. `size_of::<u64>()` — здесь нет аргумента, из которого можно вывести тип, поэтому turbofish —
   единственный способ.

```text
42 2.5 0 8
```

Без всякой подсказки компилятор останавливается:

```rust
fn main() {
    let n = "42".parse().unwrap();
    println!("{}", n);
}
```

```text
error[E0284]: type annotations needed
 --> examples/turbofish_err.rs:2:9
  |
2 |     let n = "42".parse().unwrap();
  |         ^        ----- type must be known at this point
  |
help: consider giving `n` an explicit type
  |
2 |     let n: /* Type */ = "42".parse().unwrap();
  |          ++++++++++++
```

**Исправление:** напишите `let n: i32 = …` или `parse::<i32>()`.

## 8. Константные параметры: числа как параметры

Параметр типа обозначает тип. **Константный параметр (const generic)** обозначает *значение*, чаще
всего длину массива. Он объявляется как `const N: usize`:

```rust
fn sum_array<const N: usize>(values: [i32; N]) -> i32 {
    let mut total = 0;
    for v in values {
        total += v;
    }
    total
}

fn first_and_len<T: Copy, const N: usize>(values: [T; N]) -> (T, usize) {
    (values[0], N)
}

fn main() {
    println!("{}", sum_array([1, 2, 3]));
    println!("{}", sum_array([10, 20, 30, 40, 50]));
    println!("{:?}", first_and_len(['a', 'b']));
}
```

**Что происходит:** `[i32; N]` — массив ровно из `N` чисел. Компилятор узнаёт `N` из аргумента
(сначала 3, потом 5), а внутри функции `N` можно использовать как обычное значение `usize`. Массивы
разной длины — это разные типы, и константные параметры позволяют одной функции обслуживать все
они. Чаще всего вы встретите это в коде библиотек; просто помните, что такое бывает.

```text
6
150
('a', 2)
```

> 📏 **Соглашение:** однобуквенные имена нормальны для коротких обобщённых функций: `T` — «тип»,
> `E` — «ошибка», `K` и `V` — «ключ» и «значение», `N` — число. Если у функции несколько
> параметров с разными ролями, берите описательные имена (`Item`, `Key`, `Value`), чтобы сигнатура
> сама себя объясняла.

## Частые ошибки

**1. Забытое ограничение** (Раздел 2): `error[E0369]: binary operation '>' cannot be applied to type '&T'`.
Добавьте трейт, который нужен телу: `T: PartialOrd`.

**2. Два разных типа там, где ожидается один `T`.**

```rust
struct Point<T> {
    x: T,
    y: T,
}

fn main() {
    let p = Point { x: 5, y: 4.0 };
    println!("{}", p.x);
}
```

```text
error[E0308]: mismatched types
 --> examples/point_mixed.rs:7:30
  |
7 |     let p = Point { x: 5, y: 4.0 };
  |                              ^^^ expected integer, found floating-point number
```

**Что это значит:** `x: 5` зафиксировало `T` как целое число, значит, и `y` должно быть целым. Если
нужны разные типы, используйте два параметра: `Point<X, Y>`.

**3. Забытое `<T>` после `impl`.**

```rust
struct Wrapper<T> {
    value: T,
}

impl Wrapper<T> {
    fn get(&self) -> &T {
        &self.value
    }
}
```

```text
error[E0425]: cannot find type `T` in this scope
 --> examples/noimpl.rs:5:14
  |
5 | impl Wrapper<T> {
  |              ^ not found in this scope
  |
help: you might be missing a type parameter
  |
5 | impl<T> Wrapper<T> {
  |     +++
```

**Что это значит:** `T` нужно объявить, прежде чем использовать. `impl<T>` объявляет его, а
`Wrapper<T>` использует. Строка `help:` показывает исправление.

**4. Перемещение значения без `Copy` из среза** (Раздел 5): `E0508`. Верните ссылку, добавьте
ограничение `Copy` или вызовите `clone()` (с ограничением `Clone`).

**5. Компилятор не может вывести тип** (Раздел 7): `E0284: type annotations needed`. Добавьте
аннотацию типа или turbofish.

## Попробуйте сами

В этом этапе все маленькие эксперименты живут в одном учебном пакете, `code/p05-lessons`, а каждый
урок сохраняется в отдельной ветке Git. Если пакета ещё нет, создайте его.

1. Откройте терминал и перейдите в учебный репозиторий:

   ```bash
   cd ~/projects/rust
   git status
   ```

   `git status` должен сказать `nothing to commit, working tree clean`. Если это не так, сначала
   сделайте коммит или отмените старые изменения (Уроки 01.3 и 01.5).

2. Создайте пакет и папку для примеров:

   ```bash
   cargo new code/p05-lessons
   mkdir code/p05-lessons/examples
   ```

   Каждый файл в `examples/` — отдельная маленькая программа. Запускается она командой
   `cargo run --example <имя файла без .rs>`. Так целый этап экспериментов живёт в одном пакете, а
   у каждого эксперимента свой `main`.

3. Убедитесь, что результаты сборки никогда не попадают в коммиты. Папка `target/` содержит
   скомпилированные файлы и очень велика. Посмотрите, что видит Git:

   ```bash
   cd code/p05-lessons
   cargo build
   git status --short -uall | head -5
   ```

   Если вы видите строки вроде `?? code/p05-lessons/target/...`, значит, в вашем репозитории нет
   `.gitignore` для неё. Из корня репозитория создайте или дополните файл:

   ```bash
   cd ~/projects/rust
   echo "target/" >> .gitignore
   git status --short
   ```

   Теперь файлы `target/` должны исчезнуть из списка. Останутся только ваши исходники и
   `.gitignore`.

4. Начните **ветку для этого урока**. Одна ветка на урок — хорошая привычка: в ветке `main` всегда
   лежит законченная работа, а неудачный эксперимент можно выбросить.

   ```bash
   git switch -c lesson/05-01-generics
   ```

   ```text
   Switched to a new branch 'lesson/05-01-generics'
   ```

5. Создайте `code/p05-lessons/examples/generics_largest.rs` с обобщённой программой `largest` из
   Раздела 2 и запустите её:

   ```bash
   cd code/p05-lessons
   cargo run --example generics_largest
   ```

   Вы должны увидеть `100`, `23.8`, `z`, `zebra`.

6. Сохраните работу коммитом. Формат сообщения — из Урока 01.9 (короткий тип, двоеточие, краткое
   описание):

   ```bash
   cd ~/projects/rust
   git add .
   git commit -m "feat(p05): generic largest function"
   ```

7. Повторите шаги 5–6 для программы `Stack<T>` (`examples/generics_stack.rs`), затем верните
   законченный урок в `main`:

   ```bash
   git switch main
   git merge lesson/05-01-generics
   git branch -d lesson/05-01-generics
   git log --oneline --graph
   ```

   Поскольку `main` за это время не менялась, Git делает *fast-forward* (просто передвигает `main`
   вперёд), и коммит слияния не появляется. История получается прямой линией маленьких коммитов.

> 🔒 **Безопасность:** `.gitignore` — это ещё и место, где секреты не попадают в историю. Позже в
> курсе (Этап 10) вы добавите в него такие файлы, как `.env`. Секрет, один раз попавший в коммит,
> остаётся в истории навсегда, даже если следующим коммитом удалить файл.

## Упражнения

### Упражнение 1 — Поменять местами пару (легко)

Напишите обобщённую функцию `swap_pair<A, B>(pair: (A, B)) -> (B, A)`. Для `swap_pair((1, "one"))`
результат должен быть `("one", 1)`.

<details>
<summary>Подсказка</summary>

Поля кортежа — это `pair.0` и `pair.1`. Верните новый кортеж с ними в другом порядке. Нужны два
параметра типа, потому что у двух частей могут быть разные типы.

</details>

<details>
<summary>Ответ</summary>

```rust
fn swap_pair<A, B>(pair: (A, B)) -> (B, A) {
    (pair.1, pair.0)
}

fn main() {
    println!("{:?}", swap_pair((1, "one")));
}
```

```text
("one", 1)
```

Почему это работает: функция забирает владение кортежем и перемещает две его части в новый кортеж.
Ограничения не нужны, потому что мы не сравниваем, не печатаем и не копируем значения, а только
перемещаем их.

</details>

### Упражнение 2 — Наименьшее значение и обобщённая пара (средне)

1. Напишите `fn smallest<T: PartialOrd>(list: &[T]) -> &T`.
2. Напишите структуру `Pair<T>` с полями `first` и `second` и методом `larger(&self) -> &T`,
   который возвращает ссылку на большее из двух. Подумайте, какое ограничение нужно блоку `impl`.

<details>
<summary>Подсказка</summary>

Для `smallest` скопируйте `largest` и переверните сравнение. Для `Pair` сравнение `>=` двух
значений `T` требует `T: PartialOrd`. Поставьте ограничение на блок `impl`: `impl<T: PartialOrd> Pair<T>`.

</details>

<details>
<summary>Ответ</summary>

```rust
fn smallest<T: PartialOrd>(list: &[T]) -> &T {
    let mut smallest = &list[0];
    for item in list {
        if item < smallest {
            smallest = item;
        }
    }
    smallest
}

#[derive(Debug)]
struct Pair<T> {
    first: T,
    second: T,
}

impl<T: PartialOrd> Pair<T> {
    fn larger(&self) -> &T {
        if self.first >= self.second {
            &self.first
        } else {
            &self.second
        }
    }
}

fn main() {
    println!("{}", smallest(&[7, 3, 9]));
    println!("{}", smallest(&["pear", "apple"]));
    let p = Pair { first: 2.5, second: 1.5 };
    println!("{}", p.larger());
}
```

```text
3
apple
2.5
```

Почему это работает: ограничение на блоке `impl` означает, что `larger` существует только для
`Pair<T>`, где `T` можно сравнивать. `Pair` из несравнимого типа создать всё же можно, но метода
`larger` у него просто не будет.

</details>

### Упражнение 3 — Обобщённая очередь (средне)

Напишите обобщённую структуру `Queue<T>` (первым пришёл — первым вышел) на основе `Vec<T>` с
методами `new`, `enqueue(item)`, `dequeue() -> Option<T>` и `len`. Затем **напишите тесты**: один
проверяет порядок элементов, другой использует `Queue<String>`.

<details>
<summary>Подсказка</summary>

`enqueue` — это `push` у вектора. `dequeue` должен брать *первый* элемент: `self.items.remove(0)`
удаляет и возвращает элемент с индексом 0. Сначала проверьте `is_empty()`, потому что `remove(0)`
паникует на пустом векторе. Тесты поместите в блок `#[cfg(test)] mod tests` и запускайте командой
`cargo test --example <имя>`.

</details>

<details>
<summary>Ответ</summary>

```rust
#[derive(Debug)]
struct Queue<T> {
    items: Vec<T>,
}

impl<T> Queue<T> {
    fn new() -> Self {
        Queue { items: Vec::new() }
    }

    fn enqueue(&mut self, item: T) {
        self.items.push(item);
    }

    fn dequeue(&mut self) -> Option<T> {
        if self.items.is_empty() {
            None
        } else {
            Some(self.items.remove(0))
        }
    }

    fn len(&self) -> usize {
        self.items.len()
    }
}

fn main() {
    let mut q = Queue::new();
    q.enqueue("a");
    q.enqueue("b");
    println!("{:?} {:?} {:?} len={}", q.dequeue(), q.dequeue(), q.dequeue(), q.len());
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn queue_is_first_in_first_out() {
        let mut q = Queue::new();
        q.enqueue(1);
        q.enqueue(2);
        assert_eq!(q.dequeue(), Some(1));
        assert_eq!(q.dequeue(), Some(2));
        assert_eq!(q.dequeue(), None);
    }

    #[test]
    fn queue_works_with_strings() {
        let mut q: Queue<String> = Queue::new();
        q.enqueue(String::from("x"));
        assert_eq!(q.len(), 1);
    }
}
```

```text
Some("a") Some("b") None len=0
```

Запуск `cargo test --example answers` (если файл — `examples/answers.rs`) выведет
`test result: ok. 2 passed` для двух тестов.

Почему это работает: один и тот же код `Queue` обслуживает `&str`, `i32` и `String`. `remove(0)`
сдвигает все остальные элементы на одну позицию влево, поэтому для длинных очередей он медленный
(время пропорционально длине). Настоящая очередь использует `VecDeque`, о котором можно прочитать в
документации стандартной библиотеки; для обучения простой версии достаточно.

</details>

### Упражнение 4 — Минимум и максимум за один проход (сложно)

Напишите `fn min_max<T: PartialOrd + Copy>(list: &[T]) -> Option<(T, T)>`, которая возвращает
наименьшее и наибольшее значение или `None` для пустого среза. Напишите тесты: несколько чисел, одно
число `f64` и пустой срез.

<details>
<summary>Подсказка</summary>

Сразу верните `None`, если `list.is_empty()`. Иначе начните и `min`, и `max` с `list[0]` и
обновляйте их в одном цикле. Для теста с пустым срезом компилятору нужно знать `T`; напишите
`min_max::<u8>(&[])` с turbofish.

</details>

<details>
<summary>Ответ</summary>

```rust
fn min_max<T: PartialOrd + Copy>(list: &[T]) -> Option<(T, T)> {
    if list.is_empty() {
        return None;
    }
    let mut min = list[0];
    let mut max = list[0];
    for &item in list {
        if item < min {
            min = item;
        }
        if item > max {
            max = item;
        }
    }
    Some((min, max))
}

fn main() {
    println!("{:?}", min_max(&[4, -2, 9]));
    println!("{:?}", min_max::<i32>(&[]));
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn min_max_finds_both_ends() {
        assert_eq!(min_max(&[4, -2, 9]), Some((-2, 9)));
        assert_eq!(min_max(&[1.5]), Some((1.5, 1.5)));
    }

    #[test]
    fn min_max_of_empty_is_none() {
        assert_eq!(min_max::<u8>(&[]), None);
    }
}
```

```text
Some((-2, 9))
None
```

Почему это работает: `Copy` позволяет копировать значения из среза в `min` и `max`. Для
`assert_eq!` сравниваемые значения должны реализовывать `Debug` и `PartialEq`; `Option<(T, T)>`
реализует оба, если их реализует `T`, а целые и дробные типы, использованные здесь, их реализуют.

</details>

## Вопросы для самопроверки

1. Что простыми словами означает `fn first<T>(list: &[T]) -> &T`?

<details>
<summary>Ответ</summary>

«Для любого типа `T` возьми заимствованный срез значений `T` и верни заимствованную ссылку на одно `T`». `T` — параметр типа: пропуск, который компилятор заполняет при каждом вызове.

</details>

2. Почему `fn largest<T>(list: &[T]) -> &T` не компилируется, если в теле есть `>`?

<details>
<summary>Ответ</summary>

Потому что `T` может быть любым типом, в том числе таким, который нельзя сравнивать. Тело может использовать только умения, обещанные ограничениями. Добавление `T: PartialOrd` делает сравнение допустимым.

</details>

3. Чем отличаются `impl<T> Point<T>` и `impl Point<f64>`?

<details>
<summary>Ответ</summary>

`impl<T> Point<T>` определяет методы для любого `Point`, каким бы ни был `T`. `impl Point<f64>` определяет методы только для `Point<f64>`. У `Point<i32>` их нет.

</details>

4. Замедляют ли обобщения программу во время выполнения? Чем вы платите вместо этого?

<details>
<summary>Ответ</summary>

Нет. Мономорфизация создаёт специализированную копию обобщённого кода для каждого конкретного типа, поэтому скорость во время выполнения такая же, как у кода, написанного вручную. Платить приходится временем компиляции (больше кода для компиляции) и, возможно, размером программы (больше копий).

</details>

5. Когда нужен turbofish `::<>`?

<details>
<summary>Ответ</summary>

Когда компилятор не может вывести параметр типа ни из аргументов, ни из аннотации типа, например `"42".parse::<i32>()` или `size_of::<u64>()`.

</details>

6. Что такое `N` в `fn f<const N: usize>(a: [i32; N])`?

<details>
<summary>Ответ</summary>

Константный параметр (const generic): число (значение, а не тип), которое входит в тип массива. Компилятор берёт его из длины переданного массива.

</details>

## Итоги

- Обобщённая функция или тип работает с разными типами; вы пишете её один раз (DRY), а компилятор подставляет тип.
- Параметр типа (`T`) объявляется в `<>` после имени, а при написании методов — ещё и после `impl`.
- Компилятор проверяет обобщённый код один раз, используя только умения из ограничений трейтов (`T: PartialOrd + Copy` или блок `where`).
- `Option<T>`, `Result<T, E>` и `Vec<T>` — обобщённые типы стандартной библиотеки.
- Мономорфизация делает одну копию кода на каждый конкретный тип: нет затрат во время выполнения, но компиляция дольше, а файлы больше.
- Используйте turbofish `::<Тип>` или аннотацию типа, когда вывода типов недостаточно.
- Константные параметры (`const N: usize`) делают числа, например длины массивов, параметром.
- Привычка с Git: одна ветка на урок, маленькие коммиты, слияние в `main`, когда урок закончен.

## Источники и что почитать

- [The Rust Programming Language — 10.1 Generic Data Types](https://doc.rust-lang.org/book/ch10-01-syntax.html) — официальная глава об обобщениях.
- [Rust by Example — Generics](https://doc.rust-lang.org/rust-by-example/generics.html) — много коротких запускаемых примеров.
- [The Rust Reference — Generic parameters](https://doc.rust-lang.org/reference/items/generics.html) — точные правила, включая константные параметры.
- [std::mem::size_of](https://doc.rust-lang.org/std/mem/fn.size_of.html) — функция из раздела 6.
- [Документация Git — git switch](https://git-scm.com/docs/git-switch) — ветки через `git switch -c`.

---

[🏠 Начало](../START-HERE.md) · [← Этап 05 — Rust: средний уровень](README.md) · [📋 Этап 05](README.md) · [05.2 Трейты →](02-traits.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
