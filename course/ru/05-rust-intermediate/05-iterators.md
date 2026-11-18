<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 05.4 Замыкания](04-closures.md) · [📋 Этап 05](README.md) · [05.6 Тестирование подробно →](06-testing-in-depth.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/05-rust-intermediate/05-iterators.md) <!-- nav -->

# Урок 05.5 — Итераторы

> **Этап 05 — Rust: средний уровень** · Урок 5 из 14 · ⏱ около 4 часов (2 занятия)

## Что вы узнаете

- Что такое итератор и как работают трейт `Iterator` и его метод `next`.
- Чем отличаются `iter()`, `iter_mut()` и `into_iter()`.
- Почему итераторы *ленивы* и чем адаптеры (`map`, `filter`, …) отличаются от потребителей (`sum`, `collect`, …).
- Как собирать результаты в `Vec`, `String`, `HashMap` или `Result`.
- Как написать собственный итератор и когда простой цикл `for` всё же лучше.

## Перед началом

- Вы прошли [Урок 05.4 — Замыкания](04-closures.md). Почти каждый метод итератора принимает замыкание.
- Вы знаете векторы ([Урок 04.7](../04-structs-enums-collections-errors/07-vectors.md)), хеш-таблицы
  ([Урок 04.9](../04-structs-enums-collections-errors/09-hash-maps.md)), `Option` и `Result`, а также трейты ([Урок 05.2](02-traits.md)).
- У вас есть пакет `code/p05-lessons` из Урока 05.1.

## Новые слова

| Слово | Значение |
|---|---|
| **итератор** (iterator) | Значение, которое выдаёт последовательность элементов по одному, когда вы просите следующий. |
| **`Iterator`** | Стандартный трейт итераторов. Его единственный обязательный метод — `next`. |
| **`Item`** | Ассоциированный тип `Iterator`: тип того, что выдаёт итератор. |
| **адаптер** (adapter) | Метод итератора, который принимает итератор и возвращает *новый* итератор (`map`, `filter`, `take`, …). Сам по себе ничего не делает. |
| **потребитель** (consumer) | Метод итератора, который *использует* итератор целиком и выдаёт итоговый результат (`sum`, `count`, `collect`, `fold`, …). |
| **ленивый** (lazy) | Не делает работу, пока результат не понадобится. Адаптеры ленивы. |
| **`collect`** | Потребитель, собирающий все элементы в коллекцию вроде `Vec`, `String` или `HashMap`. |
| **`IntoIterator`** | Трейт, который говорит «это значение можно превратить в итератор». Его используют циклы `for`. |
| **диапазон** (range) | Значения `1..5` (от 1 до 4) и `1..=5` (от 1 до 5). Диапазон сам является итератором. |
| **конвейер** (pipeline) | Цепочка адаптеров, заканчивающаяся потребителем: `data.iter().filter(…).map(…).sum()`. |
| **ассоциированный тип** | Имя типа внутри трейта, которое выбирает каждая реализация (Урок 05.2). |

## 1. Что такое итератор

**Итератор** похож на закладку в списке. Вы спрашиваете его «какой следующий элемент?», он выдаёт один
элемент и передвигает закладку вперёд. Когда элементы закончились, он говорит «больше ничего».

В Rust это стандартный трейт `Iterator`. Вот его ядро (упрощённо):

```rust
trait Iterator {
    type Item;                          // what kind of items it produces
    fn next(&mut self) -> Option<Self::Item>;
}
```

- `type Item` — **ассоциированный тип** (Урок 05.2): каждый итератор выбирает свой тип элементов.
- `next` возвращает `Some(item)`, пока элементы есть, и `None`, когда они закончились. Он принимает
  `&mut self`, потому что передвижение закладки *изменяет* итератор.

Вы можете вызвать `next` сами:

```rust
fn main() {
    let numbers = vec![10, 20, 30];
    let mut walker = numbers.iter();
    println!("{:?}", walker.next());
    println!("{:?}", walker.next());
    println!("{:?}", walker.next());
    println!("{:?}", walker.next());
}
```

**Что происходит:**

1. `numbers.iter()` создаёт итератор по **ссылкам** на элементы (`&i32`). Вектор не меняется и
   по-прежнему владеет числами.
2. `let mut walker` должна быть `mut`, потому что `next` меняет внутреннюю позицию итератора.
3. Первые три вызова возвращают `Some(10)`, `Some(20)`, `Some(30)`. Четвёртый возвращает `None`: список закончился.

```text
Some(10)
Some(20)
Some(30)
None
```

Все остальные методы итераторов в этом уроке (`map`, `sum`, `filter`, …) построены поверх `next`. Вы
получаете их бесплатно, как методы трейта по умолчанию (Урок 05.2), как только тип реализует `next`.

## 2. Циклы `for` используют итераторы

Конструкцию `for item in &list` вы используете с Этапа 02. Теперь можно увидеть, что она делает.
Цикл `for` вызывает `next`, пока не получит `None`:

```rust
fn main() {
    let names = vec!["Aziz", "Bobur"];

    for name in &names {
        println!("for: {name}");
    }

    let mut walker = names.iter();
    loop {
        match walker.next() {
            Some(name) => println!("loop: {name}"),
            None => break,
        }
    }
}
```

```text
for: Aziz
for: Bobur
loop: Aziz
loop: Bobur
```

**Что происходит:** обе половины делают одно и то же. Цикл `for` — короткая запись связки `loop` +
`match` + `next`. Он вызывает `IntoIterator::into_iter` для того, что стоит после `in`, чтобы
получить итератор.

### Три способа обойти коллекцию

| Метод | Элементы — это | Коллекция после этого |
|---|---|---|
| `.iter()` | общие ссылки `&T` | не изменена, по-прежнему ваша |
| `.iter_mut()` | изменяемые ссылки `&mut T` | по-прежнему ваша, но могла измениться |
| `.into_iter()` | значения во владении `T` | **перемещена**: пропала (для `Vec`) |

```rust
fn main() {
    let mut scores = vec![70, 85, 90];

    for s in scores.iter() {
        print!("{} ", s); // s is &i32
    }
    println!();

    for s in scores.iter_mut() {
        *s += 5; // s is &mut i32
    }
    println!("{:?}", scores);

    let total: i32 = scores.into_iter().sum(); // scores is moved here
    println!("{total}");
}
```

**Что происходит:**

1. `iter()` позволяет прочитать каждую оценку; вектор не тронут.
2. `iter_mut()` даёт `&mut i32`, поэтому `*s += 5` меняет числа внутри вектора (`*` следует по ссылке,
   Урок 03.5).
3. `into_iter()` поглощает вектор: мы получаем значения во владение, и `sum` их складывает. После этой
   строки `scores` использовать нельзя.

```text
70 85 90 
[75, 90, 95]
260
```

Форма `for x in vec` (без `&`) — это третий вид и классическая ошибка:

```rust
fn main() {
    let names = vec![String::from("Aziz"), String::from("Bobur")];
    for name in names {
        println!("{name}");
    }
    println!("{}", names.len());
}
```

```text
error[E0382]: borrow of moved value: `names`
 --> examples/into_iter_err.rs:6:20
  |
2 |     let names = vec![String::from("Aziz"), String::from("Bobur")];
  |         ----- move occurs because `names` has type `Vec<String>`, which does not implement the `Copy` trait
3 |     for name in names {
  |                 ----- `names` moved due to this implicit call to `.into_iter()`
...
6 |     println!("{}", names.len());
  |                    ^^^^^ value borrowed here after move
  |
help: consider iterating over a slice of the `Vec<String>`'s content to avoid moving into the `for` loop
  |
3 |     for name in &names {
  |                 +
```

**Что это значит:** `for name in names` вызывает `into_iter`, который принимает вектор по значению.
Напишите `for name in &names`, чтобы заимствовать его (компилятор предлагает ровно это).

## 3. Итераторы ленивы: адаптеры и потребители

Методы `Iterator` бывают двух групп:

- **Адаптеры** превращают один итератор в другой: `map`, `filter`, `take`, `enumerate`, … Они
  **ленивы**: их вызов пока не выполняет работу, а лишь описывает, что *будет* сделано.
- **Потребители** протягивают элементы через конвейер и выдают итоговое значение: `sum`, `count`,
  `collect`, `fold`, `for_each`, … Только потребитель заставляет что-либо выполняться.

```rust
fn main() {
    let numbers = vec![1, 2, 3];

    let doubled = numbers.iter().map(|x| {
        println!("  doubling {x}");
        x * 2
    });
    println!("map created, nothing printed yet");

    let result: Vec<i32> = doubled.collect();
    println!("{:?}", result);
}
```

**Что происходит:**

1. `.map(…)` создаёт новый итератор, который *будет* удваивать каждый элемент. Замыкание пока не
   выполнялось.
2. Первый `println!` это доказывает: он печатается раньше любой строки «doubling».
3. `.collect()` — потребитель. Он запрашивает каждый элемент; для каждого выполняется замыкание и
   печатает «doubling…».

```text
map created, nothing printed yet
  doubling 1
  doubling 2
  doubling 3
[2, 4, 6]
```

Если забыть потребителя, компилятор предупредит:

```rust
fn main() {
    let numbers = vec![1, 2, 3];
    numbers.iter().map(|x| println!("{x}"));
}
```

```text
warning: unused `Map` that must be used
 --> examples/lazy_warn.rs:3:5
  |
3 |     numbers.iter().map(|x| println!("{x}"));
  |     ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  |
  = note: iterators are lazy and do nothing unless consumed
help: you might have meant to use `Iterator::for_each`
```

**Что это значит:** ничего не напечаталось, потому что итератор никто не потребил. Чтобы выполнить
замыкание ради побочного эффекта (например, печати), используйте цикл `for` или `for_each`.

> 💡 **Совет:** ленивость полезна. `numbers.iter().map(…).take(3)` вычисляет только три элемента, даже
> если в `numbers` миллион. Бесконечные итераторы (например, `(1..)`, «1, 2, 3, … вечно») безопасны,
> пока что-то вроде `take` их останавливает.

## 4. Обзор адаптеров

```rust
fn main() {
    let numbers = vec![1, 2, 3, 4, 5, 6];

    let squares: Vec<i32> = numbers.iter().map(|x| x * x).collect();
    println!("map:        {:?}", squares);

    let evens: Vec<&i32> = numbers.iter().filter(|x| **x % 2 == 0).collect();
    println!("filter:     {:?}", evens);

    for (index, value) in numbers.iter().enumerate().take(2) {
        println!("enumerate:  {index} -> {value}");
    }

    let names = ["a", "b", "c"];
    let pairs: Vec<(&str, &i32)> = names.iter().copied().zip(numbers.iter()).collect();
    println!("zip:        {:?}", pairs);

    let middle: Vec<i32> = numbers.iter().copied().skip(1).take(3).collect();
    println!("skip/take:  {:?}", middle);

    let joined: Vec<i32> = numbers.iter().copied().take(2).chain(100..102).collect();
    println!("chain:      {:?}", joined);

    let backwards: Vec<i32> = numbers.iter().copied().rev().collect();
    println!("rev:        {:?}", backwards);

    let small: Vec<i32> = numbers.iter().copied().take_while(|x| *x < 4).collect();
    println!("take_while: {:?}", small);

    let every_other: Vec<i32> = (0..10).step_by(3).collect();
    println!("step_by:    {:?}", every_other);

    let nested = vec![vec![1, 2], vec![3], vec![]];
    let flat: Vec<i32> = nested.into_iter().flatten().collect();
    println!("flatten:    {:?}", flat);

    for pair in numbers.windows(2).take(2) {
        println!("windows:    {:?}", pair);
    }
    for chunk in numbers.chunks(4) {
        println!("chunks:     {:?}", chunk);
    }
}
```

```text
map:        [1, 4, 9, 16, 25, 36]
filter:     [2, 4, 6]
enumerate:  0 -> 1
enumerate:  1 -> 2
zip:        [("a", 1), ("b", 2), ("c", 3)]
skip/take:  [2, 3, 4]
chain:      [1, 2, 100, 101]
rev:        [6, 5, 4, 3, 2, 1]
take_while: [1, 2, 3]
step_by:    [0, 3, 6, 9]
flatten:    [1, 2, 3]
windows:    [1, 2]
windows:    [2, 3]
chunks:     [1, 2, 3, 4]
chunks:     [5, 6]
```

**Что делает каждый адаптер:**

| Адаптер | Смысл |
|---|---|
| `map(f)` | преобразовать каждый элемент через `f` |
| `filter(p)` | оставить только элементы, для которых предикат `p` (замыкание, возвращающее `bool`) истинен |
| `enumerate()` | сопоставить каждому элементу его индекс: `(0, a), (1, b), …` |
| `zip(other)` | идти по двум итераторам вместе, выдавая пары; останавливается на более коротком |
| `skip(n)` / `take(n)` | пропустить первые `n` элементов / остановиться после `n` элементов |
| `chain(other)` | все элементы первого итератора, затем все элементы второго |
| `rev()` | идти в обратном порядке (работает для итераторов, которые знают свой конец, как у векторов и диапазонов) |
| `take_while(p)` | брать элементы до первого, для которого `p` ложен |
| `step_by(n)` | каждый `n`-й элемент |
| `flatten()` | превратить итератор коллекций в один итератор их элементов |
| `copied()` / `cloned()` | превратить элементы `&T` в `T` копированием / клонированием (см. Раздел 7) |

`windows(2)` и `chunks(4)` — методы **срезов**, а не `Iterator`, но они возвращают итераторы:
`windows(n)` даёт перекрывающиеся группы по `n`, `chunks(n)` — отдельные группы до `n` элементов.

## 5. Обзор потребителей

```rust
fn main() {
    let numbers = vec![3, 8, 1, 9, 4];

    let sum: i32 = numbers.iter().sum();
    let product: i64 = numbers.iter().map(|&x| x as i64).product();
    println!("sum={sum} product={product} count={}", numbers.iter().count());

    println!("min={:?} max={:?}", numbers.iter().min(), numbers.iter().max());
    println!("last={:?}", numbers.iter().last());

    let total = numbers.iter().fold(0, |acc, x| acc + x);
    println!("fold={total}");

    println!("any > 8: {}", numbers.iter().any(|&x| x > 8));
    println!("all > 0: {}", numbers.iter().all(|&x| x > 0));
    println!("find > 5: {:?}", numbers.iter().find(|&&x| x > 5));
    println!("position of 9: {:?}", numbers.iter().position(|&x| x == 9));

    let words = ["kiwi", "banana", "fig"];
    println!("longest: {:?}", words.iter().max_by_key(|w| w.len()));
    println!("shortest: {:?}", words.iter().min_by_key(|w| w.len()));

    let empty: Vec<i32> = Vec::new();
    println!("max of empty: {:?}", empty.iter().max());
    println!("sum of empty: {}", empty.iter().sum::<i32>());
}
```

```text
sum=25 product=864 count=5
min=Some(1) max=Some(9)
last=Some(4)
fold=25
any > 8: true
all > 0: true
find > 5: Some(8)
position of 9: Some(3)
longest: Some("banana")
shortest: Some("fig")
max of empty: None
sum of empty: 0
```

**Что происходит:**

1. `sum` и `product` должны знать тип результата: `let sum: i32` или `sum::<i32>()` (turbofish из
   Урока 05.1). Для `product` мы преобразуем числа в `i64` через `x as i64` (Этап 02), чтобы
   надёжнее избежать переполнения.
2. `min`, `max`, `last`, `find` и `position` возвращают `Option`, потому что итератор может быть
   пуст или не содержать подходящего элемента. Поэтому `max` пустого списка — `None`, а не падение.
3. `fold(start, f)` — универсальный потребитель. Он держит аккумулятор: начинает со `start`, а для
   каждого элемента делает `acc = f(acc, item)`. `sum` — это `fold` со сложением.
4. `any` и `all` останавливаются, как только ответ известен (они *вычисляются по короткой схеме*):
   `any` останавливается на первом `true`.
5. `max_by_key(|w| w.len())` выбирает элемент с наибольшим ключом.

## 6. `collect`: сбор результатов

`collect` строит коллекцию из итератора. Какую? **Решает тип, который вы запросили.**

```rust
use std::collections::HashMap;

fn main() {
    let letters: String = vec!['r', 'u', 's', 't'].into_iter().collect();
    println!("{letters}");

    let upper: Vec<String> = ["a", "b"].iter().map(|s| s.to_uppercase()).collect();
    println!("{:?}", upper);

    let keys = ["one", "two", "three"];
    let values = [1, 2, 3];
    let map: HashMap<&str, i32> = keys.iter().copied().zip(values.iter().copied()).collect();
    println!("two -> {:?}", map.get("two"));

    let good: Result<Vec<i32>, _> = "1 2 3".split(' ').map(|s| s.parse::<i32>()).collect();
    println!("{:?}", good);
    let bad: Result<Vec<i32>, _> = "1 x 3".split(' ').map(|s| s.parse::<i32>()).collect();
    println!("{:?}", bad);

    let (even, odd): (Vec<i32>, Vec<i32>) = (1..=6).partition(|n| n % 2 == 0);
    println!("{:?} {:?}", even, odd);
}
```

**Что происходит:**

1. Сбор `char` в `String` склеивает их.
2. Сбор пар `(ключ, значение)` в `HashMap` строит таблицу. Здесь пары делает `zip`.
3. Сбор итератора из `Result<i32, _>` в `Result<Vec<i32>, _>` — очень полезный приём: если **все**
   элементы `Ok`, вы получаете `Ok(vec)`; на **первом** `Err` сбор останавливается и возвращает эту
   ошибку. То же работает с `Option`.
4. `partition(p)` делит один итератор на две коллекции: элементы, где `p` истинен, и остальные.

```text
rust
["A", "B"]
two -> Some(2)
Ok([1, 2, 3])
Err(ParseIntError { kind: InvalidDigit })
[2, 4, 6] [1, 3, 5]
```

Если тип не указан, компилятор не может угадать, какую коллекцию вы хотите:

```rust
fn main() {
    let doubled = (1..4).map(|x| x * 2).collect();
    println!("{:?}", doubled);
}
```

```text
error[E0283]: type annotations needed
 --> examples/collect_err.rs:2:9
  |
2 |     let doubled = (1..4).map(|x| x * 2).collect();
  |         ^^^^^^^                         ------- type must be known at this point
  |
  = note: the type must implement `FromIterator<i32>`
help: consider giving `doubled` an explicit type
  |
2 |     let doubled: Vec<_> = (1..4).map(|x| x * 2).collect();
  |                ++++++++
```

**Исправление:** `let doubled: Vec<_> = …` (`_` означает «выведи тип элементов сам») или
`.collect::<Vec<_>>()`.

## 7. Ловушка ссылок: `&i32`, `&&i32` и компания

Поскольку `iter()` выдаёт **ссылки**, замыкания часто получают ссылку, а `filter` передаёт своему
замыканию ссылку на элемент, так что получается ссылка на ссылку:

```rust
fn main() {
    let numbers = vec![1, 2, 3, 4];

    let a: Vec<&i32> = numbers.iter().filter(|x| **x > 1).collect();
    let b: Vec<i32> = numbers.iter().copied().filter(|x| *x > 1).collect();
    let c: Vec<i32> = numbers.iter().filter(|&&x| x > 1).cloned().collect();
    let d: Vec<i32> = numbers.clone().into_iter().filter(|x| *x > 1).collect();
    println!("{:?} {:?} {:?} {:?}", a, b, c, d);
}
```

```text
[2, 3, 4] [2, 3, 4] [2, 3, 4] [2, 3, 4]
```

**Что происходит:** четыре способа сказать одно и то же.

- `a`: элементы `iter()` — это `&i32`; `filter` передаёт замыканию `&&i32`, поэтому `**x` добирается до
  числа. Результат хранит ссылки (`Vec<&i32>`).
- `b`: `.copied()` сначала превращает `&i32` в `i32`. Теперь `filter` получает `&i32`, и хватает `*x`.
  `copied` работает для типов с `Copy`; `cloned()` — для любого типа с `Clone` (он клонирует каждый элемент).
- `c`: образец `|&&x|` разбирает ссылки прямо в параметре (образцы, Урок 04.5).
- `d`: `into_iter()` для вектора во владении выдаёт значения `i32` (мы сначала клонировали вектор,
  чтобы сохранить оригинал).

Если забыть про ссылку, возникнет такая ошибка:

```rust
fn main() {
    let numbers = vec![1, 2, 3, 4];
    let big: Vec<&i32> = numbers.iter().filter(|x| x > 1).collect();
    println!("{:?}", big);
}
```

```text
error[E0308]: mismatched types
 --> examples/refs_err.rs:3:56
  |
3 |     let big: Vec<&i32> = numbers.iter().filter(|x| x > 1).collect();
  |                                                        ^ expected `&&_`, found integer
  |
  = note: expected reference `&&_`
                  found type `{integer}`
```

**Что это значит:** `x` — это `&&i32`, а вы сравниваете её с числом `1`. Разыменуйте через `**x` или
сначала используйте `.copied()`. Приём для новичков: **если запутались, добавьте `.copied()`
сразу после `.iter()`** (для чисел и других типов с `Copy`), и ссылки исчезнут.

Последняя частая ошибка — вызов `next` без `mut`:

```text
error[E0596]: cannot borrow `walker` as mutable, as it is not declared as mutable
 --> examples/mut_err.rs:4:22
  |
4 |     println!("{:?}", walker.next());
  |                      ^^^^^^ cannot borrow as mutable
  |
help: consider changing this to be mutable
  |
3 |     let mut walker = numbers.iter();
  |         +++
```

## 8. Пишем собственный итератор

Чтобы сделать свой тип итерируемым, реализуйте `Iterator`: выберите `type Item` и напишите `next`.

```rust
struct Countdown {
    remaining: u32,
}

impl Iterator for Countdown {
    type Item = u32;

    fn next(&mut self) -> Option<u32> {
        if self.remaining == 0 {
            None
        } else {
            let current = self.remaining;
            self.remaining -= 1;
            Some(current)
        }
    }
}

fn evens_up_to(limit: u32) -> impl Iterator<Item = u32> {
    (0..=limit).filter(|n| n % 2 == 0)
}

fn main() {
    for n in (Countdown { remaining: 3 }) {
        println!("{n}...");
    }
    println!("liftoff");

    let total: u32 = Countdown { remaining: 4 }.sum();
    println!("sum = {total}");

    let doubled: Vec<u32> = Countdown { remaining: 3 }.map(|n| n * 2).collect();
    println!("{:?}", doubled);

    println!("{:?}", evens_up_to(9).collect::<Vec<_>>());
}
```

**Что происходит:**

1. `Countdown` хранит состояние в `remaining`. Каждый `next` возвращает текущее значение и делает
   шаг к концу; на нуле он возвращает `None`.
2. Мы написали только `next`. Поскольку `Countdown` реализует `Iterator`, его можно использовать в
   цикле `for`, а `sum`, `map`, `collect` и все остальные методы работают бесплатно. (Скобки вокруг
   литерала структуры в `for n in (Countdown { … })` нужны, потому что `{` сразу после `in` был бы
   прочитан как начало тела цикла.)
3. `evens_up_to` возвращает `impl Iterator<Item = u32>` (Урок 05.2): «какой-то итератор, выдающий
   `u32`». Вызывающему не нужно знать, что это `Filter<RangeInclusive<u32>, …>`.

```text
3...
2...
1...
liftoff
sum = 10
[6, 4, 2]
[0, 2, 4, 6, 8]
```

> 📏 **Соглашение:** функции, принимающие «что-то, по чему можно пройтись циклом», часто используют
> `impl IntoIterator<Item = T>` как тип параметра, чтобы вызывающий мог передать вектор, срез или
> итератор. Вы увидите это в библиотеках. Для ваших собственных простых функций `&[T]` вполне
> достаточно.

## 9. Итераторы или циклы?

Правильны оба. Сравните одну и ту же задачу, написанную двумя способами: сложить числа на корректных
строках текста.

```rust
fn main() {
    let text = "10\nabc\n32\n\n8";

    let mut total_loop = 0;
    for line in text.lines() {
        if let Ok(n) = line.parse::<i32>() {
            total_loop += n;
        }
    }

    let total_iter: i32 = text.lines().filter_map(|l| l.parse::<i32>().ok()).sum();
    println!("{total_loop} {total_iter}");

    let first_bad = text.lines().find(|l| l.parse::<i32>().is_err());
    println!("{:?}", first_bad);
}
```

```text
50 50
Some("abc")
```

**Что происходит:** `filter_map(f)` — это `map` и `filter` в один шаг: замыкание возвращает `Option`,
`Some(x)` сохраняется как `x`, а `None` отбрасывается. `.ok()` превращает `Result` в `Option`. Версия
с итераторами говорит, *что* нам нужно (сумма чисел, которые разобрались); версия с циклом говорит, *как*.

> ⚡ **Производительность:** конвейеры итераторов — **абстракции с нулевой стоимостью** (Урок 05.1):
> компилятор генерирует тот же машинный код, что и для цикла, написанного вручную, а часто и лучший,
> потому что знает, что индекс не выйдет за границы. За чистый стиль вы не платите. Измеряйте
> бенчмарком, прежде чем верить любому утверждению об обратном (Урок 05.12).

Что когда предпочитать:

- Используйте **конвейеры итераторов** для задач «преобразовать / отфильтровать / свернуть последовательность».
- Используйте **цикл `for`**, когда нужны `break`/`continue` со сложными условиями, несколько изменяемых переменных или ранние возвраты из середины тела. Понятный цикл лучше хитрого конвейера.

Последний пример, который объединяет много частей, — подсчёт частоты слов:

```rust
use std::collections::HashMap;

fn main() {
    let text = "the cat and the dog and the bird";
    let mut counts: HashMap<&str, usize> = HashMap::new();
    for word in text.split_whitespace() {
        *counts.entry(word).or_insert(0) += 1;
    }

    let mut pairs: Vec<(&str, usize)> = counts.into_iter().collect();
    pairs.sort_by(|a, b| b.1.cmp(&a.1).then(a.0.cmp(b.0)));
    for (word, n) in pairs.iter().take(3) {
        println!("{word}: {n}");
    }
}
```

```text
the: 3
and: 2
bird: 1
```

**Что происходит:** у `HashMap` нет фиксированного порядка, поэтому мы собираем его пары в `Vec` и
сортируем замыканием: сначала большие счётчики (`b.1.cmp(&a.1)`), при равенстве — по алфавиту
(`.then(a.0.cmp(b.0))`, Урок 05.4). Затем `take(3)` оставляет тройку лидеров. Без сортировки порядок
вывода менялся бы от запуска к запуску.

> 💡 **Совет:** крейт `itertools` (версия 0.15) добавляет десятки дополнительных адаптеров, таких как
> `join`, `sorted`, `dedup` и `group_by`. Урок 05.8 учит, как добавить крейт в проект.

## Частые ошибки

**1. Забыть, что адаптеры ленивы** (предупреждение `unused Map that must be used`, раздел 3). Добавьте потребителя или используйте `for`/`for_each`.

**2. Использовать `for x in vec`, а потом сам вектор** (`E0382`, раздел 2). Обходите `&vec`.

**3. Забыть тип для `collect`** (`E0283`, раздел 6). Пишите `let v: Vec<_> = …`.

**4. Сравнивать ссылку со значением внутри `filter`** (`E0308`, раздел 7). Используйте `**x`, `|&&x|` или `.copied()`.

**5. Вызывать `next` у итератора без `mut`** (`E0596`, раздел 7).

**6. Слишком хитрая цепочка.** Если конвейеру нужно больше пяти шагов или комментарий для объяснения, разбейте его на именованные
переменные или используйте цикл. Читаемость важнее хитрости.

## Попробуйте сами

На этот раз практика с Git — это **рефакторинг** (изменение способа записи кода без изменения того,
что он делает) и исправление последнего коммита через `--amend`.

1. Начните ветку урока:

   ```bash
   cd ~/projects/rust
   git switch main
   git switch -c lesson/05-05-iterators
   ```

2. Создайте `code/p05-lessons/examples/iter_sum_loop.rs` с функцией на цикле и тестом и сделайте коммит:

   ```rust
   fn sum_even_squares(limit: u32) -> u32 {
       let mut total = 0;
       for n in 1..=limit {
           if n % 2 == 0 {
               total += n * n;
           }
       }
       total
   }

   fn main() {
       println!("{}", sum_even_squares(10));
   }

   #[cfg(test)]
   mod tests {
       use super::*;

       #[test]
       fn sums_squares_of_even_numbers() {
           assert_eq!(sum_even_squares(10), 220);
       }
   }
   ```

   ```bash
   cd code/p05-lessons
   cargo test --example iter_sum_loop
   cd ~/projects/rust
   git add .
   git commit -m "feat(p05): sum of even squares with a loop"
   ```

3. **Рефакторинг:** перепишите тело функции в конвейер `(1..=limit).filter(|n| n % 2 == 0).map(|n| n * n).sum()`.
   Запустите тот же тест. Он по-прежнему должен проходить: тест — ваша страховка того, что поведение не изменилось.

   ```bash
   cd code/p05-lessons
   cargo test --example iter_sum_loop
   cd ~/projects/rust
   git commit -am "refactor(p05): use an iterator pipeline"
   ```

   Правило: коммит `refactor:` никогда не меняет поведение, а тесты проходят **до и после** него. Если
   пришлось менять тест, это не рефакторинг.

4. Вы заметили, что сообщение первого коммита неточное или вы забыли файл. **`git commit --amend`**
   заменяет **последний** коммит (новое сообщение плюс всё, что вы сейчас подготовили к коммиту):

   ```bash
   git commit --amend -m "refactor(p05): sum of even squares as an iterator pipeline"
   git log --oneline
   ```

   Посмотрите на хеш: он изменился (например, с `b356577` на `b83f345`), потому что Git создал *новый*
   коммит, который заменил старый. Поэтому **никогда не используйте amend для коммита, который вы уже
   отправили** (Урок 01.8) и который другие могли скачать: это переписывает историю, от которой зависят
   другие люди. Для локальных неопубликованных коммитов это безопасно и обычно.

5. Завершите: слейте в `main`, как в предыдущих уроках.

## Упражнения

### Упражнение 1 — Сумма квадратов чётных чисел (легко)

Напишите `fn sum_even_squares(limit: u32) -> u32` одним конвейером итераторов: сумма квадратов чётных
чисел от 1 до `limit`. Проверьте `limit = 10` (ответ 220) и `limit = 0`.

<details>
<summary>Подсказка</summary>

`(1..=limit)` — итератор-диапазон. Добавьте `.filter(|n| n % 2 == 0)`, `.map(|n| n * n)` и потребителя `.sum()`.
`sum` нужно знать тип: это тип результата функции `u32`, поэтому turbofish не нужен.

</details>

<details>
<summary>Ответ</summary>

```rust
fn sum_even_squares(limit: u32) -> u32 {
    (1..=limit).filter(|n| n % 2 == 0).map(|n| n * n).sum()
}

fn main() {
    println!("{}", sum_even_squares(10));
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn sums_squares_of_even_numbers() {
        assert_eq!(sum_even_squares(10), 4 + 16 + 36 + 64 + 100);
    }

    #[test]
    fn limit_zero_gives_zero() {
        assert_eq!(sum_even_squares(0), 0);
    }
}
```

```text
220
```

Почему это работает: `1..=0` — пустой диапазон, поэтому `sum` возвращает 0. `n` в `filter` — это `&u32`,
и `n % 2` работает и для ссылок, потому что арифметические операторы реализованы и для них.

</details>

### Упражнение 2 — Посчитать длинные слова (легко)

Напишите `fn count_long_words(text: &str, min_len: usize) -> usize`, которая считает слова длиннее
`min_len` букв. Проверьте на `"the quick brown fox jumps"` и `min_len = 3` (ответ 3), а также на пустом тексте.

<details>
<summary>Подсказка</summary>

`text.split_whitespace()` даёт итератор по словам. Используйте `filter` с `w.len() > min_len` и `count()`.

</details>

<details>
<summary>Ответ</summary>

```rust
fn count_long_words(text: &str, min_len: usize) -> usize {
    text.split_whitespace().filter(|w| w.len() > min_len).count()
}

fn main() {
    println!("{}", count_long_words("the quick brown fox jumps", 3));
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn counts_words_longer_than_the_limit() {
        assert_eq!(count_long_words("the quick brown fox jumps", 3), 3);
    }

    #[test]
    fn empty_text_has_no_words() {
        assert_eq!(count_long_words("", 3), 0);
    }
}
```

```text
3
```

Почему это работает: в «quick», «brown» и «jumps» по пять букв; в «the» и «fox» — по три. Обратите внимание,
что `len()` считает байты, что для английского текста равно числу букв (Урок 04.8 объясняет разницу).

</details>

### Упражнение 3 — Итератор Фибоначчи (средне)

Реализуйте `struct Fibonacci`, реализующую `Iterator<Item = u64>` и выдающую 0, 1, 1, 2, 3, 5, … Она должна
**останавливаться (возвращать `None`), а не переполняться**, когда следующее число не поместится в `u64`.
Тесты: первые десять чисел; сумма чётных среди них; и то, что итератор заканчивается.

<details>
<summary>Подсказка</summary>

Храните два поля, `current` и `next`. В `next()` запомните `current` как результат, вычислите новый
`next` через `self.current.checked_add(self.next)?` (`checked_add` возвращает `None` при переполнении,
а `?` возвращает этот `None` из вашей функции), затем сдвиньте два поля.

</details>

<details>
<summary>Ответ</summary>

```rust
struct Fibonacci {
    current: u64,
    next: u64,
}

impl Fibonacci {
    fn new() -> Self {
        Fibonacci { current: 0, next: 1 }
    }
}

impl Iterator for Fibonacci {
    type Item = u64;

    fn next(&mut self) -> Option<u64> {
        let result = self.current;
        let new_next = self.current.checked_add(self.next)?;
        self.current = self.next;
        self.next = new_next;
        Some(result)
    }
}

fn main() {
    let first: Vec<u64> = Fibonacci::new().take(10).collect();
    println!("{:?}", first);
    println!("count until overflow: {}", Fibonacci::new().count());
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn first_ten_numbers() {
        let v: Vec<u64> = Fibonacci::new().take(10).collect();
        assert_eq!(v, vec![0, 1, 1, 2, 3, 5, 8, 13, 21, 34]);
    }

    #[test]
    fn works_with_adapters() {
        let even_sum: u64 = Fibonacci::new().take(10).filter(|n| n % 2 == 0).sum();
        assert_eq!(even_sum, 2 + 8 + 34);
    }

    #[test]
    fn stops_instead_of_overflowing() {
        assert!(Fibonacci::new().count() < 100);
    }
}
```

```text
[0, 1, 1, 2, 3, 5, 8, 13, 21, 34]
count until overflow: 92
```

Почему это работает: эта простая версия останавливается чуть раньше. Она возвращает `None`, когда не
может вычислить число на два шага вперёд, поэтому выдаёт 92 числа, хотя ещё пара чисел поместилась бы.
Для нас это приемлемо, и переполнения не происходит.
🔒 **Безопасность:** во многих языках целочисленное переполнение молча даёт неверные числа; `checked_add` делает
граничный случай явным (переполнение целых чисел объяснялось на Этапе 02).

</details>

### Упражнение 4 — Разобрать строки `key=value` в таблицу (средне)

Напишите `fn parse_pairs(text: &str) -> HashMap<&str, &str>`, которая читает по одной паре `key = value` в
строке, обрезает пробелы, пропускает строки без `=` и собирает результат **одним конвейером итераторов**
(без цикла `for`). Тесты: корректные и «сломанные» строки; повторяющиеся ключи (побеждает более поздняя строка).

<details>
<summary>Подсказка</summary>

`text.lines().filter_map(|line| line.split_once('='))` даёт пары `(&str, &str)` и отбрасывает строки без `=`.
Затем `.map(|(k, v)| (k.trim(), v.trim()))` и `.collect()` в `HashMap`, который назван в типе результата.
Для повторяющихся ключей `collect` в таблицу вставляет элементы по порядку, поэтому побеждает последний.

</details>

<details>
<summary>Ответ</summary>

```rust
use std::collections::HashMap;

fn parse_pairs(text: &str) -> HashMap<&str, &str> {
    text.lines()
        .filter_map(|line| line.split_once('='))
        .map(|(k, v)| (k.trim(), v.trim()))
        .collect()
}

fn main() {
    let pairs = parse_pairs("host = example.org\nbroken\nport=8080");
    let mut keys: Vec<_> = pairs.keys().collect();
    keys.sort();
    println!("{:?}", keys);
    println!("{:?}", pairs.get("port"));
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn parses_valid_lines_and_skips_broken_ones() {
        let map = parse_pairs("host = a\nbroken\nport=1");
        assert_eq!(map.len(), 2);
        assert_eq!(map.get("host"), Some(&"a"));
        assert_eq!(map.get("port"), Some(&"1"));
    }

    #[test]
    fn later_line_wins_for_duplicate_keys() {
        let map = parse_pairs("a=1\na=2");
        assert_eq!(map.get("a"), Some(&"2"));
    }
}
```

```text
["host", "port"]
Some("8080")
```

Почему это работает: ключи и значения — срезы `text` (пропуск времён жизни, Урок 05.3), поэтому ничего не
копируется. Ключи мы сортируем перед печатью, потому что у `HashMap` нет фиксированного порядка.

</details>

### Упражнение 5 — Самые частые слова (сложно)

Напишите `fn top_words(text: &str, n: usize) -> Vec<(String, usize)>`, возвращающую `n` самых частых слов без учёта
регистра (`"The"` и `"the"` — одно слово), сначала самые частые, при равенстве — по алфавиту. Тесты: подсчёт и порядок;
равенства; слов меньше, чем `n`; пустой текст.

<details>
<summary>Подсказка</summary>

Считайте в `HashMap<String, usize>`, используя `word.to_lowercase()` как ключ. Соберите `counts.into_iter()` в `Vec`, отсортируйте через
`sort_by(|a, b| b.1.cmp(&a.1).then(a.0.cmp(&b.0)))` и закончите `.into_iter().take(n).collect()`.

</details>

<details>
<summary>Ответ</summary>

```rust
use std::collections::HashMap;

fn top_words(text: &str, n: usize) -> Vec<(String, usize)> {
    let mut counts: HashMap<String, usize> = HashMap::new();
    for word in text.split_whitespace() {
        *counts.entry(word.to_lowercase()).or_insert(0) += 1;
    }
    let mut pairs: Vec<(String, usize)> = counts.into_iter().collect();
    pairs.sort_by(|a, b| b.1.cmp(&a.1).then(a.0.cmp(&b.0)));
    pairs.into_iter().take(n).collect()
}

fn main() {
    for (word, count) in top_words("The cat and the dog and THE bird", 3) {
        println!("{word}: {count}");
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn counts_ignoring_case_and_sorts_by_count() {
        let top = top_words("The cat and the dog and THE bird", 2);
        assert_eq!(top, vec![("the".to_string(), 3), ("and".to_string(), 2)]);
    }

    #[test]
    fn ties_are_sorted_alphabetically() {
        let top = top_words("b a c", 3);
        assert_eq!(top[0].0, "a");
        assert_eq!(top[2].0, "c");
    }

    #[test]
    fn fewer_words_than_n_is_fine() {
        assert_eq!(top_words("one", 5).len(), 1);
        assert!(top_words("", 5).is_empty());
    }
}
```

```text
the: 3
and: 2
bird: 1
```

Почему это работает: сортировка делает результат детерминированным (одинаковым при каждом запуске), а именно это
делает его проверяемым. `take(n)` просто останавливается раньше, когда пар меньше `n`. Знаки препинания здесь не
обрабатываются: `"dog,"` и `"dog"` будут разными словами. Улучшить это — хорошее дополнительное задание.

</details>

## Вопросы для самопроверки

1. Какой единственный метод нужно написать, чтобы реализовать `Iterator`, и что он возвращает?

<details>
<summary>Ответ</summary>

`next(&mut self) -> Option<Self::Item>`. Он возвращает `Some(item)` для следующего элемента и `None`, когда итератор закончился. Вы также указываете тип `Item`.

</details>

2. Чем отличаются `iter()`, `iter_mut()` и `into_iter()` для `Vec<String>`?

<details>
<summary>Ответ</summary>

`iter()` даёт `&String` (заимствование только для чтения), `iter_mut()` даёт `&mut String` (можно менять), `into_iter()` даёт `String` во владение и поглощает вектор.

</details>

3. Что значит «адаптеры ленивы»? Что запускает конвейер?

<details>
<summary>Ответ</summary>

Адаптеры вроде `map` и `filter` лишь строят новый итератор и не делают работу. Конвейер запускается, когда потребитель (`collect`, `sum`, `count`, `for_each`, цикл `for`, …) запрашивает элементы.

</details>

4. Как превратить итератор из `Result` в `Result<Vec<i32>, E>`?

<details>
<summary>Ответ</summary>

Вызвать `collect()` в тип `Result<Vec<i32>, E>`. Он вернёт `Ok(vec)`, если каждый элемент — `Ok`, иначе первую `Err`.

</details>

5. Почему `numbers.iter().filter(|x| x > 1)` не компилируется?

<details>
<summary>Ответ</summary>

`iter()` выдаёт `&i32`, а `filter` передаёт ссылку на этот элемент, поэтому `x` — это `&&i32`, которую нельзя сравнивать с целым числом `1`. Используйте `**x > 1`, `|&&x| x > 1` или `.copied()` перед `filter`.

</details>

6. Медленнее ли конвейеры итераторов, чем циклы, написанные вручную?

<details>
<summary>Ответ</summary>

Обычно нет. Это абстракции с нулевой стоимостью: компилятор производит эквивалентный машинный код. Если производительность действительно важна, измеряйте бенчмарком (Урок 05.12).

</details>

## Итоги

- Итератор выдаёт элементы по одному через `next`, возвращая `Some(item)` и в конце `None`.
- Цикл `for` построен на итераторах. `iter()` заимствует элементы, `iter_mut()` заимствует их изменяемо, `into_iter()` забирает во владение.
- Адаптеры (`map`, `filter`, `enumerate`, `zip`, `take`, `skip`, `chain`, `rev`, …) ленивы; потребители (`sum`, `count`, `fold`, `collect`, `find`, `any`, …) запускают конвейер.
- `collect` строит `Vec`, `String`, `HashMap` или даже `Result<Vec<_>, _>`; решает целевой тип, поэтому указывайте его.
- Элементы `iter()` — это ссылки; справляйтесь с ними через `.copied()` / `.cloned()` или образцы вроде `|&&x|`.
- Вы можете реализовать `Iterator` для собственных типов, написав `next`; все адаптеры тогда работают бесплатно.
- Конвейеры так же быстры, как циклы; выбирайте более понятный. Рефакторьте при зелёных тестах до и после, а `git commit --amend` применяйте только к неопубликованным коммитам.

## Источники и что почитать

- [The Rust Programming Language — 13.2 Processing a Series of Items with Iterators](https://doc.rust-lang.org/book/ch13-02-iterators.html) — официальная глава.
- [std::iter::Iterator](https://doc.rust-lang.org/std/iter/trait.Iterator.html) — полный список адаптеров и потребителей с примерами.
- [std::iter (обзор модуля)](https://doc.rust-lang.org/std/iter/index.html) — как соотносятся итераторы, `IntoIterator` и `FromIterator`.
- [Rust by Example — Iterators](https://doc.rust-lang.org/rust-by-example/trait/iter.html) — короткие примеры.
- [itertools на docs.rs](https://docs.rs/itertools) — дополнительные методы итераторов (крейт объясняется в Уроке 05.8).
- [Документация Git — git commit (--amend)](https://git-scm.com/docs/git-commit) — переписывание последнего коммита.

---

[🏠 Начало](../START-HERE.md) · [← 05.4 Замыкания](04-closures.md) · [📋 Этап 05](README.md) · [05.6 Тестирование подробно →](06-testing-in-depth.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
