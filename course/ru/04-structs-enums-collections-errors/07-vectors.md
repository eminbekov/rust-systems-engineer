<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 04.6 Модули, крейты и пакеты](06-modules-crates-and-packages.md) · [📋 Этап 04](README.md) · [04.8 Строки подробно →](08-strings-in-depth.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/04-structs-enums-collections-errors/07-vectors.md) <!-- nav -->

# Урок 04.7 — Векторы

> **Этап 04 — Структуры, перечисления, коллекции и ошибки** · Урок 7 из 12 · ⏱ около 4 часов (2 занятия)

## Что вы узнаете

- Что такое *коллекция* и зачем она программе.
- Как создать вектор, добавлять в него элементы, удалять их и безопасно читать.
- Как обходить вектор циклом тремя способами: только читая, изменяя элементы и забирая владение.
- Почему правила заимствования не дают держать ссылку на элемент вектора, пока вы делаете `push`.
- Что такое *длина* и *ёмкость* (capacity) и как хранить данные разных видов в одном векторе с помощью перечисления.

## Перед началом

- Вы прошли [Урок 04.6 — Модули, крейты и пакеты](06-modules-crates-and-packages.md).
- Вы знаете, что такое стек и куча ([Урок 03.1](../03-ownership-and-borrowing/01-memory-stack-and-heap.md)), как работают владение и заимствование ([Урок 03.4](../03-ownership-and-borrowing/04-references-and-borrowing.md) и [Урок 03.5](../03-ownership-and-borrowing/05-mutable-references-and-the-borrow-rules.md)) и что такое срез ([Урок 03.6](../03-ownership-and-borrowing/06-slices-and-string-types.md)).
- Создайте проект для этого урока:

```bash
cd ~/projects/rust
cargo new code/p04-vectors
cd code/p04-vectors
```

## Новые слова

| Слово | Значение |
|---|---|
| **коллекция** | Значение, которое хранит много других значений. Данные коллекции лежат в куче, поэтому она может расти и уменьшаться, пока программа работает. |
| **вектор** | Самая используемая коллекция: растущий список значений одного типа, лежащих в памяти одно за другим. |
| **`Vec`** | Название типа вектора в Rust. Это сокращение от *vector* (вектор). Запись `Vec<i32>` читается как «вектор чисел `i32`». |
| **`<T>` (параметр типа)** | Тип внутри угловых скобок, например `i32` в `Vec<i32>`. Он говорит Rust, элементы какого вида хранит коллекция. Полное объяснение — в [Уроке 05.1 — Обобщения (generics)](../05-rust-intermediate/01-generics.md). |
| **индекс** | Номер позиции элемента в векторе. У первого элемента индекс `0`. |
| **длина** (`len`) | Сколько элементов в векторе сейчас. |
| **ёмкость** (capacity) | Сколько элементов вектор может вместить, прежде чем ему понадобится больше памяти. Ёмкость всегда не меньше длины. |
| **перераспределение памяти** (reallocation) | Перенос элементов в новый, больший участок кучи, потому что старый заполнен. |
| **итератор** | Значение, которое выдаёт элементы коллекции по одному. Его создаёт `.iter()`. Полное объяснение — в [Уроке 05.5 — Итераторы](../05-rust-intermediate/05-iterators.md). |
| **паника** | Аварийная остановка, которую Rust запускает намеренно, когда программа не может продолжать, например при чтении несуществующего индекса. |

## 1. Что такое вектор?

До сих пор каждая переменная хранила одно значение или фиксированную группу значений вроде
кортежа или массива. Настоящим программам нужно хранить *столько элементов, сколько есть в
данных*: все строки файла, всех пользователей, которые вошли в систему, все оценки класса.
Когда вы пишете код, это число вам неизвестно.

**Коллекция** — это значение, которое хранит много других значений. В стандартной библиотеке Rust
есть несколько коллекций. Самая используемая — **вектор**, он записывается как `Vec<T>`.

Представьте вектор как ряд пронумерованных коробок, который можно удлинять. В каждой коробке
лежит значение одного и того же типа. Можно добавить коробку в конец, убрать последнюю коробку и
заглянуть в любую коробку по её номеру.

У **массива** (например, `[i32; 3]`) размер фиксируется на этапе компиляции. **Вектор** может
расти и уменьшаться во время работы программы. Поэтому векторы встречаются намного чаще.

### Как вектор выглядит в памяти

Вектор — это маленькая структура на стеке с тремя полями. Сами элементы лежат в куче
(вспомните [Урок 03.1](../03-ownership-and-borrowing/01-memory-stack-and-heap.md)).

```text
 stack                           heap
┌─────────────────────┐        ┌──────┬──────┬──────┬──────┐
│ v: pointer ─────────┼──────▶ │  10  │  20  │  30  │ free │
│    length   = 3     │        └──────┴──────┴──────┴──────┘
│    capacity = 4     │          index 0   1      2     (room for 1 more)
└─────────────────────┘
```

- **указатель** (pointer) говорит, где в куче начинаются элементы;
- **длина** (length) говорит, сколько коробок занято (3);
- **ёмкость** (capacity) говорит, сколько коробок есть всего (4).

Поскольку элементы лежат в куче, вектор может попросить участок побольше, когда ему это нужно.

## 2. Создание вектора

Есть два распространённых способа.

```rust
fn main() {
    let mut scores: Vec<i32> = Vec::new();
    scores.push(90);
    scores.push(75);
    scores.push(88);

    let names = vec!["Ali", "Vali", "Sara"];

    println!("{:?}", scores);
    println!("{:?}", names);
    println!("scores has {} items", scores.len());
    println!("is empty? {}", scores.is_empty());
}
```

**Что происходит:**

1. `Vec::new()` создаёт пустой вектор. В нём нет элементов, и он ещё не просил у кучи память.
   Rust не может угадать тип элементов пустого вектора, поэтому мы пишем его сами: `Vec<i32>`.
2. `scores.push(90)` добавляет `90` в конец. `push` изменяет вектор, поэтому `scores` нужно
   объявить с `mut`.
3. `vec![...]` — это **макрос** (имя, которое заканчивается на `!`; `println!` вы уже
   использовали). Он создаёт вектор и заполняет его. Здесь Rust видит строки и понимает, что это
   `Vec<&str>`.
4. `{:?}` печатает вектор в *отладочном* формате: `[90, 75, 88]`.
5. `len()` возвращает число элементов. `is_empty()` возвращает `true`, если элементов нет.

```text
[90, 75, 88]
["Ali", "Vali", "Sara"]
scores has 3 items
is empty? false
```

Есть и третий способ — `Vec::with_capacity(n)`: он сразу резервирует место под `n` элементов.
Зачем это нужно, вы увидите в разделе 6.

> 📏 **Соглашение:** используйте `vec![...]`, когда первые элементы уже известны. Используйте
> `Vec::new()` (или `Vec::with_capacity(n)`) и `push`, когда вы собираете вектор шаг за шагом.

## 3. Добавление и удаление элементов

```rust
fn main() {
    let mut v = vec![10, 20, 30];

    v.push(40);
    println!("after push:   {:?}", v);

    let last = v.pop();
    println!("popped:       {:?}", last);
    println!("after pop:    {:?}", v);

    v.insert(1, 15);
    println!("after insert: {:?}", v);

    let removed = v.remove(0);
    println!("removed:      {}", removed);
    println!("after remove: {:?}", v);

    v.clear();
    println!("after clear:  {:?}", v);
    println!("pop on empty: {:?}", v.pop());
}
```

**Что происходит:**

1. `push(40)` добавляет `40` в конец.
2. `pop()` удаляет последний элемент и возвращает его **внутри `Option`**: `Some(40)`. Почему
   `Option`? Потому что вектор может быть пустым, и тогда отдавать нечего. `Option` — это тип для
   «значение или ничего» ([Урок 04.4 — Option и match](04-option-and-match.md)).
3. `insert(1, 15)` ставит `15` на индекс 1 и сдвигает все элементы после него на одно место вправо.
4. `remove(0)` вынимает элемент с индексом 0, возвращает его (не в `Option`) и сдвигает все
   элементы после него на одно место влево.
5. `clear()` удаляет все элементы. Последняя строка показывает, что `pop()` на пустом векторе
   даёт `None`.

```text
after push:   [10, 20, 30, 40]
popped:       Some(40)
after pop:    [10, 20, 30]
after insert: [10, 15, 20, 30]
removed:      10
after remove: [15, 20, 30]
after clear:  []
pop on empty: None
```

> ⚡ **Производительность:** `push` и `pop` работают в конце, поэтому они дешёвые. `insert` и
> `remove` в середине или в начале должны сдвинуть все элементы после этого места, поэтому чем
> длиннее вектор, тем они медленнее. Если вы часто удаляете из начала, лучше подойдёт другая
> коллекция (вы встретите её на Этапе 07 — Алгоритмы и структуры данных).

## 4. Чтение элементов: `[]` или `get`

Элемент можно прочитать двумя способами, и разница важна.

```rust
fn main() {
    let v = vec![10, 20, 30];

    let second = v[1];
    println!("v[1] = {}", second);

    match v.get(5) {
        Some(x) => println!("v[5] = {}", x),
        None => println!("there is no item at index 5"),
    }

    println!("first: {:?}", v.first());
    println!("last:  {:?}", v.last());
    println!("get(2): {:?}", v.get(2));
}
```

**Что происходит:**

1. `v[1]` читает элемент с индексом 1 (второй элемент, потому что счёт идёт с 0). Индекс должен
   быть типа `usize` (беззнаковое целое размером с адрес в памяти).
2. `v.get(5)` тоже читает по индексу, но возвращает `Option`: `Some(&элемент)`, если индекс
   существует, и `None`, если нет. Программа не падает; что делать с `None`, вы решаете сами с
   помощью `match`.
3. `first()` и `last()` — сокращения для `get(0)` и `get(len - 1)`. Они тоже возвращают `Option`.

```text
v[1] = 20
there is no item at index 5
first: Some(10)
last:  Some(30)
get(2): Some(30)
```

А что будет, если использовать `[]` с несуществующим индексом?

```rust
fn main() {
    let v = vec![10, 20, 30];
    let i = 7;
    println!("about to read index {}", i);
    println!("{}", v[i]);
}
```

```text
about to read index 7

thread 'main' (3849831) panicked at src/main.rs:5:21:
index out of bounds: the len is 3 but the index is 7
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
```

**Что происходит:** программа запускает **панику**: печатает сообщение с файлом и строкой и
останавливается. (Число в скобках — внутренний номер потока, а имя файла — имя тестового
проекта; у вас они будут другими.) Rust отказывается читать память, которая не входит в вектор.
Без такой проверки программа могла бы молча прочитать мусор из памяти и продолжить работу с
неверным результатом. Громкая авария намного лучше тихого неверного ответа.

> 📏 **Соглашение:** используйте `v[i]`, когда неверный индекс был бы ошибкой в вашей
> собственной программе и авария — правильная реакция. Используйте `v.get(i)`, когда индекс
> приходит оттуда, что вы не контролируете (ввод пользователя, файл, сетевое сообщение).

> 🔒 **Безопасность:** никогда не используйте данные снаружи программы как индекс с `[]`.
> Пользователь, который отправит `7` программе с 3 элементами, может её уронить. С `get`
> программа может ответить сообщением об ошибке. Паники из-за неверных индексов — частый способ
> уронить сервер.

## 5. Обход вектора циклом

Цикл `for` вы изучили в [Уроке 02.5](../02-programming-basics/05-control-flow.md). Он умеет проходить и по вектору. Есть три формы, и каждая
по-своему влияет на владение.

```rust
fn main() {
    let mut v = vec![1, 2, 3];

    // 1) read-only loop
    for x in &v {
        println!("item: {}", x);
    }

    // 2) loop that changes every item
    for x in &mut v {
        *x *= 10;
    }
    println!("after: {:?}", v);

    // 3) loop with a counter
    for (i, x) in v.iter().enumerate() {
        println!("index {} -> {}", i, x);
    }

    // 4) loop that takes ownership
    for x in v {
        println!("owned: {}", x);
    }
    // v is gone here
}
```

**Что происходит:**

1. `for x in &v` **заимствует** вектор (общая ссылка). В каждом витке `x` — это ссылка `&i32` на
   один элемент. После цикла вектор по-прежнему ваш.
2. `for x in &mut v` заимствует вектор **изменяемо**. `x` — это `&mut i32`. Чтобы изменить число
   за ссылкой, пишут `*x`; звёздочка означает «значение, на которое указывает эта ссылка»
   ([Урок 03.5](../03-ownership-and-borrowing/05-mutable-references-and-the-borrow-rules.md)). `*x *= 10` умножает элемент на 10.
3. `v.iter()` создаёт **итератор**: помощника, который выдаёт элементы по одному.
   `.enumerate()` оборачивает его так, что каждый виток даёт пару `(индекс, элемент)`. Используйте
   его, когда нужна и позиция, и значение. (Итераторам посвящён отдельный урок позже:
   [Урок 05.5](../05-rust-intermediate/05-iterators.md).)
4. `for x in v` (без `&`) **перемещает** вектор в цикл. Каждый `x` — значение, которым вы владеете.
   После цикла `v` пропадает, и использовать его больше нельзя.

```text
item: 1
item: 2
item: 3
after: [10, 20, 30]
index 0 -> 10
index 1 -> 20
index 2 -> 30
owned: 10
owned: 20
owned: 30
```

> 💡 **Совет:** по умолчанию пишите `for x in &v`. Используйте `&mut v`, когда нужно изменить
> элементы. Простое `v` применяйте только тогда, когда вектор вам больше не нужен и вы хотите
> забрать из него элементы.

## 6. Владение и векторы: классическая ошибка

Это самая известная ошибка Rust с векторами. Сначала прочитайте код и подумайте, должен ли он
быть разрешён.

```rust
fn main() {
    let mut v = vec![1, 2, 3];
    let first = &v[0];
    v.push(4);
    println!("first = {}", first);
}
```

```text
error[E0502]: cannot borrow `v` as mutable because it is also borrowed as immutable
 --> src/main.rs:4:5
  |
3 |     let first = &v[0];
  |                  - immutable borrow occurs here
4 |     v.push(4);
  |     ^^^^^^^^^ mutable borrow occurs here
5 |     println!("first = {}", first);
  |                            ----- immutable borrow later used here
```

**Что означает ошибка:** `first` — ссылка на первую коробку. Затем выполняется `push`. Если
вектор заполнен, `push` выделяет в куче участок побольше, копирует туда элементы и освобождает
старый участок (это и есть **перераспределение памяти**). Тогда `first` указывал бы в
освобождённую память — получилась бы *висячая ссылка* (dangling reference). Проверка
заимствований (borrow checker — часть компилятора, которая проверяет правила заимствования)
видит эту опасность на этапе компиляции и отвергает программу.

```text
 before push                          after push (vector was full)
 first ──▶ ┌────┬────┬────┐           old area: freed ✗   (first still points here!)
           │ 1  │ 2  │ 3  │           new area: ┌────┬────┬────┬────┬────┬────┬────┬────┐
           └────┴────┴────┘                     │ 1  │ 2  │ 3  │ 4  │    │    │    │    │
```

**Как исправить.** Либо закончите пользоваться `first` до `push`, либо скопируйте число вместо
заимствования:

```rust
fn main() {
    let mut v = vec![1, 2, 3];
    let first = v[0]; // copies the number; no borrow is kept
    v.push(4);
    println!("first = {}", first);
    println!("{:?}", v);
}
```

```text
first = 1
[1, 2, 3, 4]
```

**Что происходит:** `v[0]` для `i32` делает копию (числа реализуют `Copy`,
[Урок 03.3](../03-ownership-and-borrowing/03-move-copy-and-clone.md)). Теперь `first` — независимое число, поэтому `push` вправе переносить элементы.

## 7. Длина и ёмкость

Просить у кучи память — медленно по сравнению с обычной работой. Поэтому вектор просит *больше,
чем нужно сейчас*, и использует запас для будущих вызовов `push`. Этот запас и есть разница между
**длиной** и **ёмкостью**.

```rust
fn main() {
    let mut v: Vec<i32> = Vec::new();
    println!("len {} capacity {}", v.len(), v.capacity());
    for i in 1..=9 {
        v.push(i);
        println!("len {} capacity {}", v.len(), v.capacity());
    }

    let mut w: Vec<i32> = Vec::with_capacity(100);
    for i in 0..100 {
        w.push(i);
    }
    println!("w: len {} capacity {}", w.len(), w.capacity());
}
```

**Что происходит:**

1. У нового пустого вектора ёмкость 0: он не просил никакой памяти.
2. Первый `push` резервирует место под 4 элемента. Пятый `push` обнаруживает, что вектор полон, и
   перераспределяет память до ёмкости 8; девятый `push` делает её равной 16. Ёмкость растёт
   удвоением.
3. `Vec::with_capacity(100)` сразу резервирует место под 100 элементов, поэтому 100 вызовов
   `push` ни разу не перераспределяют память.

```text
len 0 capacity 0
len 1 capacity 4
len 2 capacity 4
len 3 capacity 4
len 4 capacity 4
len 5 capacity 8
len 6 capacity 8
len 7 capacity 8
len 8 capacity 8
len 9 capacity 16
w: len 100 capacity 100
```

Точные числа (0, 4, 8, 16) — деталь текущей стандартной библиотеки, а не обещание. Ваша
программа никогда не должна на них полагаться.

> ⚡ **Производительность:** если вы знаете (или можете прикинуть), сколько элементов добавите,
> вызовите `Vec::with_capacity(n)`. Это избавляет от повторных перераспределений и копирований.
> Но не угадывайте наугад: сначала измерьте ([Урок 05.12 — Основы бенчмарков](../05-rust-intermediate/12-benchmarking-basics.md)).
> Резерв под миллион элементов, которые вы не добавите, — это напрасно потраченная память.

## 8. Полезные методы

```rust
fn main() {
    let mut v = vec![5, 3, 8, 3, 1, 8];

    println!("contains 8? {}", v.contains(&8));

    v.sort();
    println!("sorted:   {:?}", v);

    v.dedup();
    println!("dedup:    {:?}", v);

    v.reverse();
    println!("reversed: {:?}", v);

    v.extend([100, 200]);
    println!("extended: {:?}", v);

    v.truncate(3);
    println!("truncate: {:?}", v);

    v.swap(0, 2);
    println!("swap:     {:?}", v);

    let total: i32 = v.iter().sum();
    println!("sum = {}", total);
    println!("max = {:?}", v.iter().max());

    let part = &v[1..3];
    println!("slice: {:?}", part);

    let mut a = vec![1, 2];
    let mut b = vec![3, 4];
    a.append(&mut b);
    println!("a = {:?}, b = {:?}", a, b);
}
```

**Что происходит:**

1. `contains(&8)` возвращает `true`, если какой-то элемент равен 8. Метод принимает ссылку на
   искомое значение.
2. `sort()` расставляет элементы по возрастанию прямо на месте (меняет `v`).
3. `dedup()` удаляет элементы, равные предыдущему элементу. Как ожидается, он работает только на
   отсортированном векторе — поэтому мы сначала отсортировали.
4. `reverse()` переворачивает порядок. `extend([100, 200])` добавляет в конец все элементы
   другого списка. `truncate(3)` оставляет только первые 3 элемента. `swap(0, 2)` меняет местами
   элементы с двумя индексами.
5. `v.iter().sum()` складывает все элементы. `v.iter().max()` находит наибольший и возвращает
   `Option<&i32>` (`None` для пустого вектора). Оба метода — методы итераторов; вы будете
   часто их использовать после [Урока 05.5](../05-rust-intermediate/05-iterators.md).
6. `&v[1..3]` — это **срез** (заимствованный взгляд на часть вектора, [Урок 03.6](../03-ownership-and-borrowing/06-slices-and-string-types.md)): элементы с
   индексами 1 и 2. Конец диапазона не включается.
7. `a.append(&mut b)` перемещает все элементы `b` в `a` и оставляет `b` пустым.

```text
contains 8? true
sorted:   [1, 3, 3, 5, 8, 8]
dedup:    [1, 3, 5, 8]
reversed: [8, 5, 3, 1]
extended: [8, 5, 3, 1, 100, 200]
truncate: [8, 5, 3]
swap:     [3, 5, 8]
sum = 16
max = Some(8)
slice: [5, 8]
a = [1, 2, 3, 4], b = []
```

> 📏 **Соглашение:** функция, которая только читает список, должна принимать срез `&[T]`, а не
> `&Vec<T>`. Срез принимает вектор, массив и часть любого из них, поэтому функция получается
> гибче. Сравните: `fn average(numbers: &[f64])`.

## 9. Элементы разных видов: вектор перечислений или структур

Вектор хранит элементы **одного** типа. А если в ваших данных есть несколько видов? Оберните их в
**перечисление** ([Урок 04.3](03-enums.md)): перечисление — это один тип, а его варианты — виды. Вектор **структур**
([Урок 04.1](01-structs.md)) тоже очень распространён.

```rust
#[derive(Debug)]
enum Reading {
    Temperature(f64),
    Missing,
}

#[derive(Debug)]
struct Student {
    name: String,
    score: u32,
}

fn main() {
    let readings = vec![
        Reading::Temperature(21.5),
        Reading::Missing,
        Reading::Temperature(19.0),
    ];
    for r in &readings {
        match r {
            Reading::Temperature(t) => println!("temperature {t}"),
            Reading::Missing => println!("no data"),
        }
    }

    let mut class = Vec::new();
    class.push(Student { name: String::from("Ali"), score: 90 });
    class.push(Student { name: String::from("Sara"), score: 72 });
    for s in &class {
        println!("{} has {}", s.name, s.score);
    }
    println!("{:?}", class[0]);
}
```

**Что происходит:**

1. У `Reading` два варианта: измеренная температура или отсутствующее значение. Все три элемента
   `readings` имеют один тип, `Reading`, хотя выглядят по-разному.
2. `match` внутри цикла обрабатывает каждый вариант. Поскольку `r` — ссылка, `t` тоже ссылка;
   `{t}` печатает число за ней.
3. Для `class` тип не записан. Rust выводит `Vec<Student>` из первого `push`.
4. `class[0]` заимствует первого студента; `{:?}` печатает его благодаря `#[derive(Debug)]`.

```text
temperature 21.5
no data
temperature 19
Ali has 90
Sara has 72
Student { name: "Ali", score: 90 }
```

Обратите внимание: `19.0` печатается как `19` — формат `Display` для числа с плавающей точкой
отбрасывает лишний `.0`.

## 10. Тестирование кода с векторами

> 🧪 **Тестирование:** проверяйте обычный случай, граничный случай (один элемент) и пустой
> случай. Именно в пустых входах и входах из одного элемента прячется большинство ошибок. Вот
> функция, которая для пустого списка возвращает `None` вместо падения из-за деления на ноль, и
> два теста к ней.

```rust
/// Returns the average of the numbers, or None for an empty slice.
fn average(numbers: &[f64]) -> Option<f64> {
    if numbers.is_empty() {
        return None;
    }
    let mut total = 0.0;
    for n in numbers {
        total += n;
    }
    Some(total / numbers.len() as f64)
}

fn main() {
    let v = vec![2.0, 4.0, 9.0];
    println!("{:?}", average(&v));
    println!("{:?}", average(&[]));
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn average_of_three() {
        assert_eq!(average(&[2.0, 4.0, 9.0]), Some(5.0));
    }

    #[test]
    fn average_of_empty_is_none() {
        assert_eq!(average(&[]), None);
    }
}
```

**Что происходит:**

1. `numbers.len() as f64` превращает длину (`usize`) в число с плавающей точкой, чтобы можно было
   делить.
2. `average(&v)` передаёт `&Vec<f64>`; Rust автоматически превращает его в срез `&[f64]`.
3. `cargo test` запускает две функции с пометкой `#[test]` ([Урок 02.8](../02-programming-basics/08-your-first-tests.md)).

```text
Some(5.0)
None
```

```text
running 2 tests
test tests::average_of_empty_is_none ... ok
test tests::average_of_three ... ok

test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

## Частые ошибки

**1. Забыли `mut`.**

```rust
fn main() {
    let v = Vec::new();
    v.push(1);
    println!("{:?}", v);
}
```

```text
error[E0596]: cannot borrow `v` as mutable, as it is not declared as mutable
 --> src/main.rs:3:5
  |
3 |     v.push(1);
  |     ^ cannot borrow as mutable
  |
help: consider changing this to be mutable
  |
2 |     let mut v = Vec::new();
  |         +++
```

`push` изменяет вектор, поэтому переменная должна быть `mut`. Исправление: `let mut v = Vec::new();`.

**2. Добавили значение не того типа.**

```rust
fn main() {
    let mut v = vec![1, 2, 3];
    v.push("four");
    println!("{:?}", v);
}
```

```text
error[E0308]: mismatched types
 --> src/main.rs:3:12
  |
3 |     v.push("four");
  |       ---- ^^^^^^ expected integer, found `&str`
  |       |
  |       arguments to this method are incorrect
```

Вектор хранит один тип. `v` — вектор целых чисел, поэтому строка в него не войдёт. Исправление:
добавьте число или используйте перечисление, если вам действительно нужны два вида данных.

**3. Использовали вектор после `for x in v`.**

```rust
fn main() {
    let names = vec![String::from("Ali"), String::from("Vali")];
    for n in names {
        println!("{}", n);
    }
    println!("{}", names.len());
}
```

```text
error[E0382]: borrow of moved value: `names`
 --> src/main.rs:6:20
  |
2 |     let names = vec![String::from("Ali"), String::from("Vali")];
  |         ----- move occurs because `names` has type `Vec<String>`, which does not implement the `Copy` trait
3 |     for n in names {
  |              ----- `names` moved due to this implicit call to `.into_iter()`
...
6 |     println!("{}", names.len());
  |                    ^^^^^ value borrowed here after move
  |
help: consider iterating over a slice of the `Vec<String>`'s content to avoid moving into the `for` loop
  |
3 |     for n in &names {
  |              +
```

`for n in names` перемещает вектор в цикл (правило из [Урока 03.2](../03-ownership-and-borrowing/02-ownership-rules.md)). Компилятор даже подсказывает
исправление: напишите `for n in &names`.

**4. Держите ссылку, пока делаете `push`.** Это ошибка E0502 из раздела 6. Сначала закончите
пользоваться ссылкой или скопируйте значение.

**5. Используете `v[i]` с непроверенным индексом.** раздел 4: это паника. Используйте `get`.

## Попробуйте сами

В `code/p04-vectors` замените `src/main.rs` небольшим «списком дел»:

```rust
fn main() {
    let mut todo: Vec<String> = Vec::new();
    todo.push(String::from("read lesson 04.7"));
    todo.push(String::from("do the exercises"));
    todo.push(String::from("commit to Git"));

    for (i, task) in todo.iter().enumerate() {
        println!("{}. {}", i + 1, task);
    }

    let done = todo.remove(0);
    println!("done: {}", done);
    println!("left: {}", todo.len());
}
```

1. Запустите командой `cargo run`. Ожидаемый вывод:

```text
1. read lesson 04.7
2. do the exercises
3. commit to Git
done: read lesson 04.7
left: 2
```

2. Добавьте четвёртую задачу с помощью `push` и запустите снова. Какой номер она получит?
3. Замените `todo.remove(0)` на `todo.remove(10)` и запустите. Прочитайте сообщение о панике.
   Какие слова указывают на проблему? (Ответ: *index out of bounds*, длина и индекс.)
4. Верните строку к `remove(0)`, затем сохраните работу в Git так, как вы учились на Этапе 01:
   `git add .` и `git commit -m "Add vector practice"`.

## Упражнения

### Упражнение 1 — Список покупок (легко)

Создайте вектор из `"milk"`, `"bread"`, `"eggs"`. Добавьте `"apples"`. Напечатайте каждый
элемент с номером, начиная с 1, затем напечатайте, сколько всего элементов. Ожидаемый вывод:

```text
1. milk
2. bread
3. eggs
4. apples
4 items
```

<details>
<summary>Подсказка</summary>

Используйте `push` для нового элемента и `.iter().enumerate()` для номеров. `enumerate`
начинает с 0, поэтому печатайте `i + 1`.

</details>

<details>
<summary>Ответ</summary>

```rust
fn main() {
    let mut list = vec!["milk", "bread", "eggs"];
    list.push("apples");
    for (i, item) in list.iter().enumerate() {
        println!("{}. {}", i + 1, item);
    }
    println!("{} items", list.len());
}
```

Почему это работает: `vec!` создаёт первые три элемента, `push` добавляет четвёртый, а
`enumerate` выдаёт пары `(индекс, элемент)`.

</details>

### Упражнение 2 — Третий элемент, безопасно (легко)

Напишите `fn third(v: &[i32]) -> Option<i32>`, которая возвращает третий элемент или `None`, если
элементов меньше трёх. Она никогда не должна паниковать. Вызовите её с `&[1, 2, 3, 4]`,
`&[1, 2]` и `&[]`.

<details>
<summary>Подсказка</summary>

Используйте `v.get(2)`. Он возвращает `Option<&i32>`. Метод `.copied()` превращает
`Option<&i32>` в `Option<i32>`.

</details>

<details>
<summary>Ответ</summary>

```rust
fn third(v: &[i32]) -> Option<i32> {
    v.get(2).copied()
}

fn main() {
    println!("{:?}", third(&[1, 2, 3, 4]));
    println!("{:?}", third(&[1, 2]));
    println!("{:?}", third(&[]));
}
```

```text
Some(3)
None
None
```

Почему это работает: `get(2)` проверяет индекс, поэтому короткий вектор даёт `None`, а не
панику. `.copied()` копирует `i32` из ссылки.

</details>

### Упражнение 3 — Наименьший и наибольший, с тестами (средне)

Напишите `fn min_max(v: &[i32]) -> Option<(i32, i32)>`, которая возвращает наименьший и
наибольший элементы в виде кортежа или `None` для пустого среза. Используйте цикл (без
`.iter().max()`). Напишите **три теста**: обычный список, список из одного элемента и пустой
список.

<details>
<summary>Подсказка</summary>

Сначала верните `None`, если `v.is_empty()`. Иначе начните с `min = v[0]` и `max = v[0]`, затем
пройдите циклом и обновляйте их. В цикле `for &n in v` даёт каждое число по значению.

</details>

<details>
<summary>Ответ</summary>

```rust
/// Returns (smallest, largest) or None for an empty slice.
fn min_max(v: &[i32]) -> Option<(i32, i32)> {
    if v.is_empty() {
        return None;
    }
    let mut min = v[0];
    let mut max = v[0];
    for &n in v {
        if n < min {
            min = n;
        }
        if n > max {
            max = n;
        }
    }
    Some((min, max))
}

fn main() {
    println!("{:?}", min_max(&[4, -2, 9, 0]));
    println!("{:?}", min_max(&[]));
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn finds_both_ends() {
        assert_eq!(min_max(&[4, -2, 9, 0]), Some((-2, 9)));
    }

    #[test]
    fn one_item_is_both() {
        assert_eq!(min_max(&[7]), Some((7, 7)));
    }

    #[test]
    fn empty_gives_none() {
        assert_eq!(min_max(&[]), None);
    }
}
```

```text
running 3 tests
test tests::empty_gives_none ... ok
test tests::one_item_is_both ... ok
test tests::finds_both_ends ... ok

test result: ok. 3 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

Почему это работает: проверка на пустоту делает `v[0]` безопасным. Образец `&n` в
`for &n in v` копирует каждое число из ссылки.

</details>

### Упражнение 4 — Чётные и нечётные (средне)

Напишите `fn split_even_odd(v: &[i32]) -> (Vec<i32>, Vec<i32>)`, которая возвращает два новых
вектора: чётные числа и нечётные числа, каждый в исходном порядке. Для `[1, 2, 3, 4, 5, 6]`
напечатайте:

```text
even: [2, 4, 6]
odd:  [1, 3, 5]
```

<details>
<summary>Подсказка</summary>

Создайте два пустых вектора через `Vec::new()`, пройдите циклом по `v` и с помощью
`n % 2 == 0` выберите, в какой из них делать `push`. Верните их как кортеж `(even, odd)`.

</details>

<details>
<summary>Ответ</summary>

```rust
fn split_even_odd(v: &[i32]) -> (Vec<i32>, Vec<i32>) {
    let mut even = Vec::new();
    let mut odd = Vec::new();
    for &n in v {
        if n % 2 == 0 {
            even.push(n);
        } else {
            odd.push(n);
        }
    }
    (even, odd)
}

fn main() {
    let (e, o) = split_even_odd(&[1, 2, 3, 4, 5, 6]);
    println!("even: {:?}", e);
    println!("odd:  {:?}", o);
}
```

Почему это работает: функция владеет двумя новыми векторами и возвращает их, поэтому владение
переходит вызывающему коду. `let (e, o) = ...` раскладывает кортеж на две переменные.

</details>

### Упражнение 5 — Лучший студент (сложно)

Используйте структуру `Student` из раздела 9. Напишите
`fn best(students: &[Student]) -> Option<&Student>`, которая возвращает ссылку на студента с
наибольшим баллом или `None` для пустого среза. Для класса Ali 90, Sara 97, Vali 61 она должна
найти Sara.

<details>
<summary>Подсказка</summary>

Заведите переменную `best: Option<&Student> = None`. Для каждого студента: если `best` равна
`None`, запишите в неё этого студента; если это `Some(current)` и новый балл больше, замените.
В конце верните `best`. Возвращаемая ссылка корректна, потому что указывает в срез, которым
владеет вызывающий код.

</details>

<details>
<summary>Ответ</summary>

```rust
#[derive(Debug)]
struct Student {
    name: String,
    score: u32,
}

fn best(students: &[Student]) -> Option<&Student> {
    let mut best: Option<&Student> = None;
    for s in students {
        match best {
            None => best = Some(s),
            Some(current) => {
                if s.score > current.score {
                    best = Some(s);
                }
            }
        }
    }
    best
}

fn main() {
    let class = vec![
        Student { name: String::from("Ali"), score: 90 },
        Student { name: String::from("Sara"), score: 97 },
        Student { name: String::from("Vali"), score: 61 },
    ];
    match best(&class) {
        Some(s) => println!("best: {} ({})", s.name, s.score),
        None => println!("no students"),
    }
    println!("{:?}", best(&[]).is_none());
}
```

```text
best: Sara (97)
true
```

Почему это работает: функция только заимствует срез и возвращает заимствованную `&Student`;
ничего не копируется и не перемещается. (Как Rust узнаёт, что возвращаемая ссылка действительна,
объясняется в [Уроке 05.3 — Времена жизни](../05-rust-intermediate/03-lifetimes.md).)

</details>

## Вопросы для самопроверки

1. Чем массив отличается от вектора?

<details>
<summary>Ответ</summary>

У массива размер фиксирован и известен на этапе компиляции. Вектор может расти и уменьшаться,
пока программа работает, потому что его элементы хранятся в куче.

</details>

2. Что возвращает `v.pop()` и почему?

<details>
<summary>Ответ</summary>

`Option`: `Some(элемент)` с удалённым последним элементом или `None`, если вектор пуст.
`Option` нужен потому, что удалять может быть нечего.

</details>

3. Что произойдёт при выполнении `let x = v[10];` для вектора из 3 элементов? Как этого избежать?

<details>
<summary>Ответ</summary>

Программа запаникует с сообщением «index out of bounds». Чтобы избежать аварии, используйте
`v.get(10)`, который для отсутствующего индекса возвращает `None`.

</details>

4. Почему Rust отвергает код, который держит `&v[0]`, а затем вызывает `v.push(...)`?

<details>
<summary>Ответ</summary>

`push` может перераспределить память (перенести элементы в больший участок кучи). Старая ссылка
тогда указывала бы в освобождённую память. Правила заимствования (нельзя иметь изменяемое
заимствование, пока живо общее) останавливают это на этапе компиляции.

</details>

5. Чем длина отличается от ёмкости?

<details>
<summary>Ответ</summary>

Длина — это число элементов в векторе сейчас. Ёмкость — число элементов, которое он может
вместить, прежде чем перераспределит память. Ёмкость всегда больше длины или равна ей.

</details>

6. Чем отличаются `for x in &v`, `for x in &mut v` и `for x in v`?

<details>
<summary>Ответ</summary>

`&v` заимствует вектор и даёт общие ссылки (только чтение). `&mut v` заимствует его изменяемо и
даёт изменяемые ссылки (элементы можно менять через `*x`). `v` перемещает вектор в цикл и даёт
элементы во владение; после этого вектор использовать нельзя.

</details>

7. Вектор хранит один тип. Как положить в один вектор число, слово и «ничего»?

<details>
<summary>Ответ</summary>

Определите перечисление с вариантом для каждого вида, например `enum Item { Number(i32),
Word(String), Nothing }`, и создайте `Vec<Item>`.

</details>

## Итоги

- Вектор `Vec<T>` — растущий список элементов одного типа; элементы лежат в куче.
- Создавайте его через `Vec::new()`, `Vec::with_capacity(n)` или `vec![...]`; меняйте через
  `push`, `pop`, `insert`, `remove` и `clear`.
- `v[i]` паникует при неверном индексе; `v.get(i)` возвращает `Option` и является безопасным
  выбором для недоверенных индексов.
- `for x in &v` читает, `for x in &mut v` изменяет, `for x in v` поглощает вектор.
- Нельзя держать ссылку на элемент вектора, пока вы делаете `push`, потому что `push` может
  перенести элементы.
- Длина — число элементов; ёмкость — зарезервированное место. Используйте `with_capacity`, когда
  знаете размер.
- Используйте перечисление, чтобы положить в один вектор данные разных видов; в функциях,
  которые только читают, принимайте `&[T]`.
- Всегда тестируйте пустой случай и случай из одного элемента.

## Источники и что почитать

- [The Rust Programming Language — 8.1 Storing Lists of Values with Vectors](https://doc.rust-lang.org/book/ch08-01-vectors.html) — глава официальной книги о векторах.
- [std::vec::Vec](https://doc.rust-lang.org/std/vec/struct.Vec.html) — полный список методов вектора с примерами.
- [Rust by Example — Vectors](https://doc.rust-lang.org/rust-by-example/std/vec.html) — короткие примеры, которые можно запустить.
- [The Rustonomicon — Implementing Vec](https://doc.rust-lang.org/nomicon/vec/vec.html) — продвинутая глава, где вектор строится с нуля (читайте намного позже, после Этапа 06).
- [The Rust Programming Language — 9.1 Unrecoverable Errors with panic!](https://doc.rust-lang.org/book/ch09-01-unrecoverable-errors-with-panic.html) — подробнее о паниках, включая пример с выходом за границы.

---

[🏠 Начало](../START-HERE.md) · [← 04.6 Модули, крейты и пакеты](06-modules-crates-and-packages.md) · [📋 Этап 04](README.md) · [04.8 Строки подробно →](08-strings-in-depth.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
