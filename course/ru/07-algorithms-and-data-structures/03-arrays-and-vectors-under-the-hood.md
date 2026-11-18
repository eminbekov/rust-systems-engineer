<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 07.2 Анализ времени и памяти](02-analyzing-time-and-space.md) · [📋 Этап 07](README.md) · [07.4 Связные списки →](04-linked-lists.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/07-algorithms-and-data-structures/03-arrays-and-vectors-under-the-hood.md) <!-- nav -->

# Урок 07.3 — Массивы и векторы изнутри

> **Этап 07 — Алгоритмы и структуры данных** · Урок 3 из 20 · ⏱ около 3 часов (2 занятия)

## Что вы узнаете

- Как массив лежит в памяти и почему чтение `v[i]` стоит одинаково для любого `i`.
- Чем на самом деле является `Vec`: три числа на стеке и один блок элементов в куче.
- Почему `push` — амортизированная `O(1)`, а `insert(0, x)` и `remove(0)` — `O(n)`.
- Как самим построить небольшой растущий массив, посчитать его перемещения и проверить его против `Vec` на тысячах случайных операций.

## Перед началом

- Вы прошли [Урок 07.2 — Анализ времени и памяти](02-analyzing-time-and-space.md).
- Вы знаете векторы ([Урок 04.7](../04-structs-enums-collections-errors/07-vectors.md), особенно длину и ёмкость), стек и кучу ([Урок 03.1](../03-ownership-and-borrowing/01-memory-stack-and-heap.md)) и срезы ([Урок 03.6](../03-ownership-and-borrowing/06-slices-and-string-types.md)).

## Новые слова

| Слово | Значение |
|---|---|
| **непрерывный** (contiguous) | Расположенный один за другим, без промежутков. Элементы массива лежат непрерывно. |
| **адрес** (address) | Номер байта в памяти. Как номер дома: говорит, где что находится. |
| **произвольный доступ** (random access) | Чтение любого элемента напрямую по его позиции за одно и то же время для любой позиции. |
| **перевыделение** (reallocation) | Получение более крупного блока памяти, копирование в него элементов и возврат старого блока. |
| **сдвиг** (shifting) | Перемещение группы элементов на одно место влево или вправо, чтобы открыть или закрыть промежуток. |
| **псевдослучайные числа** (pseudo-random numbers) | Числа, которые выглядят случайными, но получаются по фиксированной формуле. Одно и то же начальное значение (*seed*) даёт одни и те же числа, поэтому упавший тест можно повторить. |
| **RAM** (Random Access Memory, оперативная память) | Быстрая рабочая память компьютера. |

## 1. Память — это длинный ряд пронумерованных ящиков

Память (RAM) компьютера похожа на очень длинную улицу, где у каждого байта есть номер дома — его **адрес**.
**Массив** хранит свои элементы **непрерывно**: элемент 0, сразу за ним элемент 1, затем элемент 2, без
промежутков и с одинаковым размером каждого элемента. Благодаря этому адрес элемента `i` вычисляется просто:

```text
address of item i  =  address of item 0  +  i * (size of one item)
```

```text
address:   1000   1004   1008   1012
          ┌──────┬──────┬──────┬──────┐
 [i32; 4] │  10  │  20  │  30  │  40  │      each i32 takes 4 bytes
          └──────┴──────┴──────┴──────┘
 index:      0      1      2      3        item 3 is at 1000 + 3 * 4 = 1012
```

Одно умножение и одно сложение, каким бы ни был `i`. Поэтому `v[i]` — это `O(1)`: так называемый **произвольный
доступ**. Тот же приём объясняет, почему все элементы должны быть одного размера и почему нельзя положить в массив
напрямую `String` неизвестной длины: вы не знали бы, где начинается элемент `i`. (`Vec<String>` хранит небольшие
дескрипторы фиксированного размера, а не сам текст строк.)

Запустите `examples/e03_layout.rs`, чтобы увидеть настоящие числа:

```rust
use std::mem::size_of;

fn main() {
    let numbers: [i32; 4] = [10, 20, 30, 40];
    let first = &numbers[0] as *const i32 as usize;
    let second = &numbers[1] as *const i32 as usize;
    println!("bytes between neighbours: {}", second - first);
    println!("[i32; 4]  takes {} bytes", size_of::<[i32; 4]>());
    println!(
        "&[i32]    takes {} bytes (address + length)",
        size_of::<&[i32]>()
    );
    println!(
        "Vec<i32>  takes {} bytes (address + length + capacity)",
        size_of::<Vec<i32>>()
    );
}
```

**Что происходит:**

1. `&numbers[0] as *const i32 as usize` превращает ссылку в обычное число-адрес (*сырой указатель*, Урок 06.6,
   преобразованный в целое). Мы его только печатаем и ничего через него не читаем, поэтому `unsafe` не нужен.
2. Два адреса отличаются на размер одного `i32`.
3. `size_of` говорит, сколько байт занимает тип на стеке.

```text
bytes between neighbours: 4
[i32; 4]  takes 16 bytes
&[i32]    takes 16 bytes (address + length)
Vec<i32>  takes 24 bytes (address + length + capacity)
```

Ссылка на срез `&[i32]` — это два числа: адрес первого элемента и длина. `Vec<i32>` — три: адрес его блока в куче,
длина и ёмкость. Сами элементы лежат в одном непрерывном блоке в куче:

```text
stack                          heap (one contiguous block of 8 slots)
┌─────────────────┐            ┌───┬───┬───┬───┬───┬───┬───┬───┐
│ ptr      ●──────┼───────────▶│ 0 │ 1 │ 2 │ 3 │ 4 │ ? │ ? │ ? │
│ len      5      │            └───┴───┴───┴───┴───┴───┴───┴───┘
│ capacity 8      │             ◀── len = 5 ──▶◀─ spare room ─▶
└─────────────────┘
```

## 2. Сколько стоит каждая операция

Посмотрите на картинку — и стоимости следуют сами:

| Операция | Стоимость | Почему |
|---|---|---|
| `v[i]`, `v.get(i)`, `v.len()` | `O(1)` | одно вычисление адреса |
| `v.push(x)` | амортизированная `O(1)` | запись после последнего элемента; иногда перевыделение |
| `v.pop()` | `O(1)` | забыть последний элемент |
| `v.insert(i, x)` | `O(n - i)` | элементы после `i` сдвигаются на одно место вправо |
| `v.remove(i)` | `O(n - i)` | элементы после `i` сдвигаются на одно место влево |
| `v.contains(&x)` | `O(n)` | просмотр элементов по одному |
| `v.swap_remove(i)` | `O(1)` | последний элемент ставится в образовавшийся промежуток (порядок меняется) |

Вставка или удаление в **начале** — худший случай: сдвигаются все `n` элементов. Вставка или удаление в **конце** —
лучший случай: ничего не сдвигается.

## 3. Строим растущий массив

Настоящий `Vec` использует неинициализированную память и `unsafe`-код. Ту же *идею* можно построить на безопасном
Rust из блока фиксированного размера из ячеек — `Box<[Option<T>]>`: ячейка хранит `Some(item)` или `None`
(пусто). Создайте `src/dyn_array.rs` и добавьте `pub mod dyn_array;` в `src/lib.rs`. Сначала структура и простые
методы:

```rust
//! Lesson 07.3: a growable array, built from a fixed-size box of slots.

/// A growable array. `slots` is a fixed-size block of memory; the first `len`
/// slots hold `Some(item)`, the rest hold `None`.
pub struct DynArray<T> {
    slots: Box<[Option<T>]>,
    len: usize,
    moves: usize, // how many items were moved so far (growing and shifting)
}

impl<T> DynArray<T> {
    pub fn new() -> DynArray<T> {
        DynArray {
            slots: Box::new([]),
            len: 0,
            moves: 0,
        }
    }

    pub fn len(&self) -> usize {
        self.len
    }

    pub fn is_empty(&self) -> bool {
        self.len == 0
    }

    pub fn capacity(&self) -> usize {
        self.slots.len()
    }

    pub fn moves(&self) -> usize {
        self.moves
    }

    /// Reading by position: one address calculation, O(1).
    pub fn get(&self, index: usize) -> Option<&T> {
        if index < self.len {
            self.slots[index].as_ref()
        } else {
            None
        }
    }
```

**Что происходит:**

1. `slots` — это блок (его длина и есть **ёмкость**); `len` считает заполненные ячейки в начале; `moves` считает,
   сколько элементов было перемещено, чтобы позже можно было измерить стоимость.
2. `Box::new([])` — пустой блок: ёмкость 0, ничего не выделено, как у `Vec::new()`.
3. `get` сначала проверяет `index < self.len`. Ячейки начиная с `len` пусты, и читать их нельзя.

`push` и `pop` работают в конце:

```rust
    /// Adds an item at the end: O(1), except when the array is full (then O(n)).
    pub fn push(&mut self, value: T) {
        if self.len == self.slots.len() {
            self.grow();
        }
        self.slots[self.len] = Some(value);
        self.len += 1;
    }

    /// Removes the last item: O(1).
    pub fn pop(&mut self) -> Option<T> {
        if self.len == 0 {
            return None;
        }
        self.len -= 1;
        self.slots[self.len].take()
    }
```

**Что происходит:**

1. `push` сначала проверяет, не заполнен ли блок (`len` равна ёмкости). Если заполнен, вызывает `grow` (ниже). Затем
   записывает значение в первую пустую ячейку и увеличивает `len`. Это `O(1)`.
2. `pop` уменьшает `len` и делает `take()` элемента из ячейки, оставляя там `None`. `Option::take` заменяет значение
   на `None` и возвращает старое: так элемент вынимают из ячейки, которую вы только заимствуете.

`insert` и `remove` должны сдвигать элементы:

```rust
    /// Inserts at `index`, shifting the items after it to the right: O(n - index).
    pub fn insert(&mut self, index: usize, value: T) {
        assert!(index <= self.len, "insertion index is out of bounds");
        if self.len == self.slots.len() {
            self.grow();
        }
        for i in (index..self.len).rev() {
            self.slots[i + 1] = self.slots[i].take();
            self.moves += 1;
        }
        self.slots[index] = Some(value);
        self.len += 1;
    }

    /// Removes the item at `index`, shifting the items after it to the left: O(n - index).
    pub fn remove(&mut self, index: usize) -> T {
        assert!(index < self.len, "removal index is out of bounds");
        let value = self.slots[index].take().expect("slot below len is filled");
        for i in index..(self.len - 1) {
            self.slots[i] = self.slots[i + 1].take();
            self.moves += 1;
        }
        self.len -= 1;
        value
    }
```

**Что происходит:**

1. `insert` освобождает место: цикл идёт от **последнего** элемента вниз до `index` и сдвигает каждый элемент на одну
   ячейку вправо. Идти надо с конца, чтобы ни один элемент не был затёрт до того, как его переместят. Затем
   записывается новый элемент.
2. `remove` вынимает элемент и идёт вверх от `index`, сдвигая каждый следующий элемент на одну ячейку влево, чтобы
   закрыть промежуток.
3. Оба считают каждое перемещение в `moves`. При `index = 0` они перемещают все `len` элементов: `O(n)`.

Наконец, `grow` и реализация `Default`, которую просит Clippy, когда у типа есть `new()` без аргументов:

```rust
    /// Doubles the capacity (the first capacity is 4) and moves all items: O(n).
    fn grow(&mut self) {
        let new_capacity = if self.slots.is_empty() {
            4
        } else {
            self.slots.len() * 2
        };
        let mut bigger: Box<[Option<T>]> = (0..new_capacity).map(|_| None).collect();
        for i in 0..self.len {
            bigger[i] = self.slots[i].take();
            self.moves += 1;
        }
        self.slots = bigger;
    }
}

impl<T> Default for DynArray<T> {
    fn default() -> DynArray<T> {
        DynArray::new()
    }
}
```

**Что происходит:**

1. Новая ёмкость — 4 при первом выделении, а затем удваивается: 4, 8, 16, 32...
2. Создаётся новый блок из `None`. (`collect` умеет сразу строить `Box<[T]>`.)
3. Каждый элемент перемещается в новый блок, а старый блок освобождается, когда `self.slots = bigger` его заменяет.
   Это и есть **перевыделение**, `O(n)`, и каждое перемещение посчитано.

### Почему `push` — амортизированная O(1)

Положим много элементов и посмотрим на `moves`. Пример `examples/e03_amortized.rs`:

```rust
use p07_algorithms::dyn_array::DynArray;

fn main() {
    println!(
        "{:>9} {:>10} {:>14}",
        "pushes", "item moves", "moves per push"
    );
    for n in [10, 1_000, 100_000, 1_000_000] {
        let mut a = DynArray::new();
        for i in 0..n {
            a.push(i);
        }
        println!(
            "{:>9} {:>10} {:>14.2}",
            n,
            a.moves(),
            a.moves() as f64 / n as f64
        );
    }

    println!();
    let mut b = DynArray::new();
    for i in 0..2_000 {
        b.insert(0, i); // always at the front
    }
    println!("2000 inserts at the front: {} item moves", b.moves());
}
```

```text
   pushes item moves moves per push
       10         12           1.20
     1000       1020           1.02
   100000     131068           1.31
  1000000    1048572           1.05

2000 inserts at the front: 2001044 item moves
```

Читаем вывод:

- Сколько бы элементов мы ни клали, число перемещений остаётся около **одного перемещения на `push`** (от 1,0 до
  1,3). Средняя стоимость одного `push` постоянна.
- Почему: рост происходит при размерах 4, 8, 16, ... и перемещает 4, 8, 16, ... элементов. Последний рост перемещает
  меньше `n` элементов, предыдущий — вдвое меньше и так далее, поэтому все росты вместе перемещают меньше `2n` элементов
  (в таблице — от 1,0 до 1,3 перемещения на `push`, но никогда не 2). Даже вместе с записью самих элементов это меньше
  `3n` шагов на `n` вызовов: **амортизированная O(1)**. (Если бы блок рос на фиксированное число ячеек, а не удваивался, итог был бы
  квадратичным. Ключ — удвоение.)
- Вставка 2 000 элементов в **начало** требует около **двух миллионов** перемещений: примерно `n²/2`. Это цена сдвига.

## 4. Тесты с псевдослучайными числами

У растущего массива много углов: рост, пустой, полный, вставка по краям. Лучший тест — тот, что в Уроке 07.1,
но со случайными входами: выполнить одни и те же случайные операции на `DynArray` и на `Vec` (очевидно правильный
эталон) и сравнивать после каждого шага. Для этого нужны случайные числа. Вместо подключения крейта мы используем
крошечный **псевдослучайный** генератор. Создайте `src/testutil.rs` и добавьте `#[cfg(test)] mod testutil;` в
`src/lib.rs` (`#[cfg(test)]` значит «только при тестировании»):

```rust
//! A tiny pseudo-random number generator for tests (the "xorshift64*" recipe).
//! The same seed always gives the same numbers, so a failing test can be repeated.
//! Not for security: never use it for passwords or keys.

pub struct Rng(u64);

impl Rng {
    pub fn new(seed: u64) -> Rng {
        Rng(seed.max(1)) // the state must never be 0
    }

    pub fn next_u64(&mut self) -> u64 {
        let mut x = self.0;
        x ^= x >> 12;
        x ^= x << 25;
        x ^= x >> 27;
        self.0 = x;
        x.wrapping_mul(0x2545_F491_4F6C_DD1D)
    }

    /// A number from 0 to n - 1. `n` must be greater than 0.
    pub fn below(&mut self, n: usize) -> usize {
        (self.next_u64() % n as u64) as usize
    }
}
```

**Что происходит:**

1. `Rng` хранит одно `u64` — своё состояние. Каждый вызов `next_u64` перемешивает состояние сдвигами (`>>`, `<<`) и
   исключающим «или» `^=` и умножает на большую константу через `wrapping_mul` (оно «заворачивается» при
   переполнении, а не паникует). Понимать, почему формула работает, не нужно. Нужно знать, что она даёт числа, которые
   выглядят случайными, и что **одно и то же начальное значение всегда даёт одну и ту же последовательность**, поэтому
   упавший тест можно повторить в точности.
2. `below(n)` возвращает число от 0 до `n - 1`, используя остаток `%`.

> 🔒 **Безопасность:** этот генератор предсказуем. Никогда не используйте его для паролей, токенов или ключей. Для них
> нужен криптографически стойкий генератор (Этап 10).

Теперь тесты — внизу `src/dyn_array.rs`:

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use crate::testutil::Rng;

    #[test]
    fn capacity_doubles() {
        let mut a = DynArray::new();
        assert!(a.is_empty());
        let mut seen = Vec::new();
        for i in 0..20 {
            a.push(i);
            if seen.last() != Some(&a.capacity()) {
                seen.push(a.capacity());
            }
        }
        assert_eq!(seen, vec![4, 8, 16, 32]);
    }

    #[test]
    fn pushes_move_fewer_than_two_items_each() {
        for n in 1..=300 {
            let mut a = DynArray::new();
            for i in 0..n {
                a.push(i);
            }
            assert!(a.moves() < 2 * n, "n = {n}, moves = {}", a.moves());
        }
    }

    #[test]
    fn behaves_like_vec_on_random_operations() {
        let mut rng = Rng::new(2024);
        let mut mine: DynArray<u64> = DynArray::new();
        let mut reference: Vec<u64> = Vec::new();
        for step in 0..5000 {
            match rng.below(5) {
                0 | 1 => {
                    let x = rng.next_u64() % 100;
                    mine.push(x);
                    reference.push(x);
                }
                2 => assert_eq!(mine.pop(), reference.pop()),
                3 => {
                    let i = rng.below(reference.len() + 1);
                    let x = rng.next_u64() % 100;
                    mine.insert(i, x);
                    reference.insert(i, x);
                }
                _ => {
                    if !reference.is_empty() {
                        let i = rng.below(reference.len());
                        assert_eq!(mine.remove(i), reference.remove(i));
                    }
                }
            }
            assert_eq!(mine.len(), reference.len(), "step {step}");
            for (i, expected) in reference.iter().enumerate() {
                assert_eq!(mine.get(i), Some(expected), "step {step}, index {i}");
            }
        }
    }
}
```

**Что происходит:**

1. `capacity_doubles` кладёт 20 элементов и записывает каждую новую ёмкость: 4, 8, 16, 32.
2. `pushes_move_fewer_than_two_items_each` превращает амортизированное рассуждение в тест: для каждого `n` от 1 до 300
   число перемещений остаётся меньше `2n`.
3. `behaves_like_vec_on_random_operations` — главный тест. Он выполняет 5 000 случайных шагов (`match rng.below(5)`
   выбирает push, push, pop, insert или remove), делает каждый и на `DynArray`, и на `Vec`, а после каждого шага
   сравнивает всё содержимое. Если вы сломаете `insert` или `grow`, этот тест найдёт это за несколько шагов, а
   сообщение покажет номер шага.

```text
running 3 tests
test dyn_array::tests::capacity_doubles ... ok
test dyn_array::tests::pushes_move_fewer_than_two_items_each ... ok
test dyn_array::tests::behaves_like_vec_on_random_operations ... ok

test result: ok. 3 passed; 0 failed; 0 ignored; 0 measured; 9 filtered out; finished in 0.04s
```

(Порядок строк на вашем компьютере может отличаться.)

## 5. Измеряем: push против вставки в начало

Теперь настоящий `Vec`. Мы строим вектор через `push` и через `insert(0, ..)` для трёх размеров, каждый в четыре раза
больше предыдущего. Добавьте программу бенчмарка в `Cargo.toml` (`criterion` уже есть с Урока 07.2):

```toml
[[bench]]
name = "vectors"
harness = false
```

`benches/vectors.rs`:

```rust
use criterion::{BenchmarkId, Criterion, criterion_group, criterion_main};
use std::hint::black_box;

fn push_back(n: usize) -> Vec<usize> {
    let mut v = Vec::new();
    for i in 0..n {
        v.push(i);
    }
    v
}

fn insert_front(n: usize) -> Vec<usize> {
    let mut v = Vec::new();
    for i in 0..n {
        v.insert(0, i);
    }
    v
}

fn bench_building(c: &mut Criterion) {
    let mut group = c.benchmark_group("build a vector");
    for n in [2_500usize, 10_000, 40_000] {
        group.bench_with_input(BenchmarkId::new("push at the end", n), &n, |b, &n| {
            b.iter(|| push_back(black_box(n)))
        });
        group.bench_with_input(BenchmarkId::new("insert at the front", n), &n, |b, &n| {
            b.iter(|| insert_front(black_box(n)))
        });
    }
    group.finish();
}

criterion_group!(benches, bench_building);
criterion_main!(benches);
```

Запустите `cargo bench --bench vectors -- --warm-up-time 1 --measurement-time 3`:

```text
build a vector/push at the end/2500
                        time:   [1.5042 µs 1.5097 µs 1.5156 µs]
build a vector/insert at the front/2500
                        time:   [231.56 µs 233.57 µs 236.28 µs]
build a vector/push at the end/10000
                        time:   [5.3047 µs 5.4668 µs 5.6396 µs]
build a vector/insert at the front/10000
                        time:   [3.8321 ms 3.8632 ms 3.9034 ms]
build a vector/push at the end/40000
                        time:   [40.976 µs 41.214 µs 41.469 µs]
build a vector/insert at the front/40000
                        time:   [92.547 ms 93.003 ms 93.683 ms]
```

(Часть строк отчёта criterion опущена. Числа получены на Apple M4 Pro; **у вас они будут другими**.)

**Как это читать:**

1. `push` растёт примерно пропорционально `n`: 1,5 мкс, 5,5 мкс, 41 мкс. (Последний скачок немного больше чем в 4 раза;
   большие блоки памяти обходятся чуть дороже, а точная причина зависит от устройства памяти компьютера.)
2. `insert(0, ..)` растёт намного быстрее: 0,23 мс, 3,9 мс, 93 мс. Каждый раз, когда `n` растёт в четыре раза, время
   растёт примерно в 16 раз или больше: это `O(n²)` для `n` вставок.
3. При 40 000 элементов построение через `insert(0, ..)` **более чем в 2 000 раз медленнее**, чем через `push`.

> ⚡ **Производительность:** сдвиг делается очень быстрым копированием памяти, поэтому `insert(0, ..)` нормален для 100
> элементов. Проблемой он становится при большом `n`. Если нужно добавлять или удалять с **обоих концов**, используйте
> `VecDeque` (Урок 07.5), где обе операции — `O(1)`.

## 6. Когда вместо этого взять стандартную библиотеку

Используйте `Vec`, если нет причины не делать этого; в нём есть всё, что вы построили здесь, и больше.

- Знаете размер? `Vec::with_capacity(n)` избавляет от всех перевыделений (Урок 04.7).
- Порядок не важен, а вы удаляете из середины? `v.swap_remove(i)` — это `O(1)`.
- Удаляете много элементов по условию? `v.retain(|x| ...)` делает это за один проход `O(n)`. Вызов `remove` в цикле — `O(n²)`.
- Добавляете или удаляете в начале? `std::collections::VecDeque`.
- Нужно быстро читать по позиции, а длина известна на этапе компиляции? Обычный массив `[T; N]`: он лежит на стеке и не
  требует выделения памяти.

## Частые ошибки

**1. Вставка за концом.** `insert` принимает позиции `0..=len`, и не больше. Положите этот код в
`examples/e03_insert_panic.rs` и запустите `cargo run --example e03_insert_panic`:

```rust
fn main() {
    let mut v = vec![10, 20, 30];
    v.insert(5, 99);
    println!("{v:?}");
}
```

```text

thread 'main' (4733772) panicked at examples/e03_insert_panic.rs:3:7:
insertion index (is 5) should be <= len (is 3)
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
```

(Число после `thread 'main'` на вашем компьютере будет другим.) Сообщение точное: индекс (5) должен быть не больше
длины (3). Наш `DynArray::insert` проверяет то же правило через `assert!`.

**2. Удаление элементов в цикле.** `while v.contains(&x) { let i = v.iter().position(...); v.remove(i); }` снова и
снова сдвигает вектор. Используйте `v.retain(|item| *item != x)`.

## Попробуйте сами

1. Наберите `testutil.rs` и `dyn_array.rs` сами и запустите `cargo test dyn_array`.
2. Сломайте `grow` нарочно (например, начните его цикл с `1` вместо `0`), посмотрите, какой тест падает и что он печатает,
   и почините.
3. Измените первую ёмкость с 4 на 1 и запустите `examples/e03_amortized.rs`. Меняется ли число перемещений на `push`?
4. Запустите бенчмарк. Затем измените `insert_front`, чтобы вставка шла в **конец** (`v.insert(v.len(), i)`). Что
   произойдёт с числами?
5. Зафиксируйте: `git add code/p07-algorithms && git commit -m "feat(p07): DynArray, a growable array"`.

## Упражнения

### Упражнение 1 — Сколько это стоит? (легко)

Назовите временную сложность для `Vec` из `n` элементов: (a) прочитать средний элемент; (b) `push`, когда вектор полон;
(c) `remove(0)`; (d) `pop()`; (e) `contains`. (f) Какая из этих операций является `O(1)` только в среднем по многим
вызовам?

<details>
<summary>Подсказка</summary>

Вспомните картинку из раздела 1: какие операции трогают одну ячейку, а какие перемещают много?

</details>

<details>
<summary>Ответ</summary>

(a) `O(1)`. (b) `O(n)`: перевыделение копирует все элементы. (c) `O(n)`: сдвигает все остальные элементы. (d) `O(1)`.
(e) `O(n)`. (f) `push`: он `O(1)` *амортизированно*, но один `push`, запускающий перевыделение, стоит `O(n)`.

</details>

### Упражнение 2 — Swap remove (средне)

Добавьте `pub fn swap_remove(&mut self, index: usize) -> T` в `DynArray`. Она удаляет элемент в позиции `index`,
ставя на его место последний элемент, за `O(1)`. Напишите тест, который сравнивает её с `Vec::swap_remove` на
случайных операциях.

<details>
<summary>Подсказка</summary>

Сначала уменьшите `len`. Затем поменяйте местами ячейку `index` и ячейку `len` (`self.slots.swap(a, b)` меняет две
ячейки среза) и сделайте `take` элемента, который теперь лежит в ячейке `len`. Это работает и когда `index` — последний
элемент (тогда ячейка меняется сама с собой).

</details>

<details>
<summary>Ответ</summary>

Добавьте внутрь блока `impl<T> DynArray<T>`:

```rust
    /// Removes the item at `index` by swapping it with the last item: O(1).
    /// The order of the items changes.
    pub fn swap_remove(&mut self, index: usize) -> T {
        assert!(index < self.len, "removal index is out of bounds");
        self.len -= 1;
        self.slots.swap(index, self.len);
        self.slots[self.len]
            .take()
            .expect("slot below len is filled")
    }
```

И тест:

```rust
    #[test]
    fn swap_remove_matches_vec() {
        let mut rng = Rng::new(7);
        let mut mine: DynArray<u64> = DynArray::new();
        let mut reference: Vec<u64> = Vec::new();
        for _ in 0..2000 {
            if reference.is_empty() || rng.below(3) > 0 {
                let x = rng.next_u64() % 50;
                mine.push(x);
                reference.push(x);
            } else {
                let i = rng.below(reference.len());
                assert_eq!(mine.swap_remove(i), reference.swap_remove(i));
            }
            assert_eq!(mine.len(), reference.len());
            for (i, expected) in reference.iter().enumerate() {
                assert_eq!(mine.get(i), Some(expected));
            }
        }
    }
```

Почему это работает: затрагиваются только две ячейки, поэтому стоимость не зависит от `n`. Цена — порядок оставшихся
элементов меняется, а `Vec::swap_remove` делает ровно то же самое, поэтому сравнение корректно.

</details>

### Упражнение 3 — Удалить все за один проход (сложно)

Добавьте `pub fn remove_all(&mut self, value: &T) where T: PartialEq`, которая удаляет каждый элемент, равный `value`,
за **один** проход, `O(n)` (а не по одному `remove` на элемент). Проверьте её против `Vec::retain` на 300 случайных
векторах с маленькими значениями (чтобы было много равных элементов). Назовите время и дополнительную память.

<details>
<summary>Подсказка</summary>

Используйте две позиции: `read` проходит по каждому элементу, `write` — куда класть следующий сохраняемый элемент.
Сохраняемые элементы переносятся из `read` в `write` (если они различаются), и `write` продвигается. В конце
`len = write`. Не забудьте записать `None` в удалённые ячейки.

</details>

<details>
<summary>Ответ</summary>

```rust
    /// Removes every item equal to `value` in one pass: O(n).
    pub fn remove_all(&mut self, value: &T)
    where
        T: PartialEq,
    {
        let mut write = 0;
        for read in 0..self.len {
            if self.slots[read].as_ref() == Some(value) {
                self.slots[read] = None;
            } else {
                if read != write {
                    self.slots[write] = self.slots[read].take();
                    self.moves += 1;
                }
                write += 1;
            }
        }
        self.len = write;
    }
```

```rust
    #[test]
    fn remove_all_matches_retain() {
        let mut rng = Rng::new(11);
        for _ in 0..300 {
            let mut mine: DynArray<u64> = DynArray::new();
            let mut reference: Vec<u64> = Vec::new();
            for _ in 0..rng.below(30) {
                let x = rng.next_u64() % 4;
                mine.push(x);
                reference.push(x);
            }
            let target = rng.next_u64() % 4;
            mine.remove_all(&target);
            reference.retain(|x| *x != target);
            assert_eq!(mine.len(), reference.len());
            for (i, expected) in reference.iter().enumerate() {
                assert_eq!(mine.get(i), Some(expected));
            }
        }
    }
```

Время `O(n)`: один проход, каждый элемент перемещается не более одного раза. Дополнительная память `O(1)`: только два
счётчика. Этот приём с «индексом чтения и индексом записи» — та же идея, что в `Vec::retain`, и вы встретите её снова как
*два указателя* в Уроке 07.15.

</details>

## Вопросы для самопроверки

1. Почему `v[i]` — это `O(1)` для любого `i`?

<details>
<summary>Ответ</summary>

Элементы лежат непрерывно и имеют одинаковый размер, поэтому адрес элемента `i` равен `start + i * size`: одно
умножение и одно сложение, каким бы ни был `i`.

</details>

2. Какие три значения `Vec<T>` хранит на стеке и где лежат элементы?

<details>
<summary>Ответ</summary>

Указатель на блок, длину и ёмкость. Элементы лежат в одном непрерывном блоке в куче.

</details>

3. Почему удвоение ёмкости делает `push` амортизированной `O(1)`, а рост по 10 ячеек за раз — нет?

<details>
<summary>Ответ</summary>

При удвоении перевыделения происходят при размерах 4, 8, 16, ... и копирования в сумме дают меньше `2n`, так что в среднем
на один `push` приходится константа. При фиксированном шаге 10 перевыделение происходит каждые 10 вызовов `push` и
каждый раз копирует около `n` элементов, поэтому итог растёт как `n²/20`: средняя стоимость одного `push` растёт вместе с `n`.

</details>

4. Вам надо снова и снова удалять первый элемент огромного вектора. Какой тип взять?

<details>
<summary>Ответ</summary>

`VecDeque`, где удаление из начала — `O(1)` (Урок 07.5). `Vec::remove(0)` — `O(n)`.

</details>

## Итоги

- Массив хранит элементы одного размера непрерывно, поэтому `address = start + i * size`, а `v[i]` — это `O(1)`.
- `Vec` — это указатель, длина и ёмкость на стеке плюс один непрерывный блок в куче.
- `push` и `pop` в конце дёшевы (`push` — амортизированная `O(1)`, потому что ёмкость удваивается); `insert` и
  `remove` в середине или в начале сдвигают элементы и стоят `O(n)`.
- `DynArray` показывает механизм на безопасном Rust: блок ячеек, длина, шаг роста с удвоением и счётчик перемещений.
- Проверяйте структуру данных против стандартной на тысячах случайных операций; используйте генератор псевдослучайных
  чисел с фиксированным начальным значением, чтобы сбой можно было повторить.
- В реальном коде используйте `Vec` (`with_capacity`, `swap_remove`, `retain`) или `VecDeque` для начала.

## Источники и что почитать

- [std::vec::Vec — Capacity and reallocation](https://doc.rust-lang.org/std/vec/struct.Vec.html#capacity-and-reallocation) — официальное описание длины, ёмкости и роста.
- [The Rustonomicon — Implementing Vec](https://doc.rust-lang.org/nomicon/vec/vec.html) — как настоящий `Vec` построен на `unsafe`-коде (продвинутый материал; читайте, когда Этап 06 станет лёгким).
- [Vec::swap_remove](https://doc.rust-lang.org/std/vec/struct.Vec.html#method.swap_remove) и [Vec::retain](https://doc.rust-lang.org/std/vec/struct.Vec.html#method.retain) — версии из стандартной библиотеки для Упражнений 2 и 3.
- [Option::take](https://doc.rust-lang.org/std/option/enum.Option.html#method.take) — как вынуть значение из `Option`, который вы только заимствуете.
- [std::mem::size_of](https://doc.rust-lang.org/std/mem/fn.size_of.html) — размер типа в байтах.

---

[🏠 Начало](../START-HERE.md) · [← 07.2 Анализ времени и памяти](02-analyzing-time-and-space.md) · [📋 Этап 07](README.md) · [07.4 Связные списки →](04-linked-lists.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
