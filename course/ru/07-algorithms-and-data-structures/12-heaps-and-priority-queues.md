<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 07.11 Деревья и двоичные деревья поиска](11-trees-and-binary-search-trees.md) · [📋 Этап 07](README.md) · [07.13 Графы: BFS и DFS →](13-graphs-bfs-and-dfs.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/07-algorithms-and-data-structures/12-heaps-and-priority-queues.md) <!-- nav -->

# Урок 07.12 — Кучи и очереди с приоритетом

> **Этап 07 — Алгоритмы и структуры данных** · Урок 12 из 20 · ⏱ около 2,5 часа (2 занятия)

## Что вы узнаете

- Что такое очередь с приоритетом и как двоичная куча реализует её с `push` и `pop` за O(log n).
- Как дерево может жить внутри обычного вектора (арифметика индексов вместо указателей).
- Как написать max-кучу и пирамидальную сортировку на Rust и как протестировать их против стандартной библиотеки.
- Как использовать `BinaryHeap` и `Reverse` из стандартной библиотеки и типичные применения: планировщики, k наибольших значений, слияние отсортированных списков.

## Перед началом

- Вы прошли [Урок 07.11 — Деревья и двоичные деревья поиска](11-trees-and-binary-search-trees.md) и [Урок 07.9 — Сортировка слиянием и быстрая сортировка](09-merge-sort-and-quicksort.md).
- Вы знаете векторы ([Урок 04.7](../04-structs-enums-collections-errors/07-vectors.md)), `Option` и кортежи, а также `#[derive]` для трейтов ([Урок 05.2](../05-rust-intermediate/02-traits.md)).
- В `code/p07-algorithms` создайте `src/heap.rs` и добавьте `pub mod heap;` в `src/lib.rs`.

## Новые слова

| Слово | Значение |
|---|---|
| **очередь с приоритетом** (priority queue) | Коллекция, в которую вы добавляете элементы и всегда первым достаёте элемент с **наивысшим приоритетом**, а не самый старый. |
| **куча** (heap, структура данных) | Дерево значений, в котором каждый родитель не меньше своих потомков (*max-куча*) или не больше (*min-куча*). **К куче памяти из Этапа 03, где `Box` и `Vec` хранят свои данные, она не имеет отношения**; у них только общее название. |
| **двоичная куча** (binary heap) | Куча, в которой у каждого узла не более двух потомков, а дерево хранится в векторе. |
| **свойство кучи** (heap property) | Правило max-кучи: каждый родитель больше или равен обоим своим потомкам. |
| **полное двоичное дерево** (complete binary tree) | Двоичное дерево, в котором все уровни заполнены, кроме, возможно, последнего, который заполняется слева. |
| **просеивание вверх / вниз** (sift up / sift down) | Перемещение элемента вверх (обмен с родителем) или вниз (обмен с большим потомком), пока свойство кучи снова не выполнится. |
| **построение кучи** (heapify) | Превращение неупорядоченного вектора в кучу за O(n). |
| **пирамидальная сортировка** (heapsort) | Алгоритм сортировки, который строит кучу и снова и снова вынимает наибольший элемент. |
| **`Reverse`** | Тип-обёртка `std::cmp::Reverse<T>`, переворачивающая порядок `T`, чтобы max-куча вела себя как min-куча. |

## 1. Очереди с приоритетом

Обычная очередь возвращает элементы в порядке поступления (Урок 07.5). **Очередь с приоритетом** возвращает самый важный первым. Примеры: операционная система выбирает следующую
задачу для запуска, больница лечит самого срочного пациента, картографическая программа (Урок 07.14) всегда раскрывает самое близкое место первым.

Можно хранить отсортированный вектор, но тогда добавление стоит O(n). Можно хранить неотсортированный вектор, но тогда поиск наибольшего стоит O(n). **Куча** делает дешёвыми обе операции:

| Операция | Отсортированный `Vec` | Неотсортированный `Vec` | Куча |
|---|---|---|---|
| добавить элемент (`push`) | O(n) | O(1) | **O(log n)** |
| посмотреть наибольший (`peek`) | O(1) | O(n) | **O(1)** |
| удалить наибольший (`pop`) | O(1) | O(n) | **O(log n)** |

> ⚠️ **Не та же куча:** на Этапе 03 *кучей* называлась область памяти для данных, которые живут, пока они нужны (`Box`, `Vec`, `String`). *Куча* из этого
> урока — структура данных, вид дерева. Совпадение названий — историческая случайность. Когда мы говорим «куча» в этом уроке, мы имеем в виду структуру данных.

## 2. Дерево внутри вектора

**Двоичная куча** — это *полное* двоичное дерево: все уровни заполнены, кроме последнего, который заполняется слева. Так как пропусков нет, дерево можно хранить в обычном `Vec` уровень за уровнем, без `Box` и без
указателей. Позиция в векторе говорит, где узел находится в дереве:

```text
index:   0   1   2   3   4   5   6
value:  90  70  80  30  50  60  20

               90   (index 0)
            /        \
         70            80          (indexes 1 and 2)
        /  \          /  \
      30    50      60    20       (indexes 3, 4, 5, 6)

for the node at index i:
    parent         = (i - 1) / 2       (whole-number division)
    left child     = 2 * i + 1
    right child    = 2 * i + 2
```

Проверка: у узла `80` индекс 2. Его родитель — `(2 - 1) / 2 = 0` (`90`), его потомки — индексы 5 и 6 (`60` и `20`). Это **max-куча**: каждый родитель не меньше своих потомков. Упорядочена только связь родитель–потомок;
между `70` и `80` порядка нет. Это более слабое правило, чем правило BST (binary search tree, двоичного дерева поиска) из Урока 07.11, поэтому кучу дешевле поддерживать, и поэтому она может выдавать только наибольший элемент, а не искать произвольный.

Наибольший элемент всегда находится на индексе 0 (корень), поэтому `peek` — O(1). Так как дерево полное, его высота около log₂ n, и оба описанных ниже восстановления идут по одному пути: O(log n).

### Push: добавить в конец, просеять вверх

Добавьте новый элемент в конец (следующее свободное место), где форма остаётся полной. Затем, пока он больше родителя, меняйте их местами. Добавляем `85`:

```text
start:        [90, 70, 80, 30, 50, 60, 20]
append 85:    [90, 70, 80, 30, 50, 60, 20, 85]       85 is the child of index 3 (30)
85 > 30, swap:[90, 70, 80, 85, 50, 60, 20, 30]       now the child of index 1 (70)
85 > 70, swap:[90, 85, 80, 70, 50, 60, 20, 30]       now the child of index 0 (90)
85 < 90: stop.
```

### Pop: взять корень, поставить на его место последний элемент, просеять вниз

Наибольший элемент лежит на индексе 0. Его удаление оставляет дыру. Заполните дыру **последним** элементом вектора (форма остаётся полной), затем, пока он меньше потомка, меняйте его с **большим** потомком.
Извлекаем из кучи выше:

```text
start:           [90, 70, 80, 30, 50, 60, 20]
swap root/last:  [20, 70, 80, 30, 50, 60 | 90]       take 90 out: that is the result
sift down 20:    children 70 and 80, the bigger is 80:
                 [80, 70, 20, 30, 50, 60]
                 children of index 2: only 60 (index 5):
                 [80, 70, 60, 30, 50, 20]            20 has no children: stop.
```

## 3. Max-куча на Rust

Поместите это в `src/heap.rs`:

```rust
//! A binary max-heap in a vector, and heapsort.

/// Moves the item at index `i` up until its parent is not smaller.
fn sift_up<T: Ord>(v: &mut [T], mut i: usize) {
    while i > 0 {
        let parent = (i - 1) / 2;
        if v[i] <= v[parent] {
            break;
        }
        v.swap(i, parent);
        i = parent;
    }
}

/// Moves the item at index `i` down until it is not smaller than its children.
/// Only the first `len` items of `v` belong to the heap.
fn sift_down<T: Ord>(v: &mut [T], mut i: usize, len: usize) {
    loop {
        let (left, right) = (2 * i + 1, 2 * i + 2);
        let mut biggest = i;
        if left < len && v[left] > v[biggest] {
            biggest = left;
        }
        if right < len && v[right] > v[biggest] {
            biggest = right;
        }
        if biggest == i {
            break;
        }
        v.swap(i, biggest);
        i = biggest;
    }
}

/// A priority queue that always gives back the biggest item first.
#[derive(Debug)]
pub struct MaxHeap<T> {
    items: Vec<T>,
}

impl<T: Ord> MaxHeap<T> {
    pub fn new() -> Self {
        MaxHeap { items: Vec::new() }
    }

    /// Builds a heap from a vector in O(n).
    pub fn from_vec(mut items: Vec<T>) -> Self {
        let len = items.len();
        for i in (0..len / 2).rev() {
            sift_down(&mut items, i, len);
        }
        MaxHeap { items }
    }

    pub fn len(&self) -> usize {
        self.items.len()
    }

    pub fn is_empty(&self) -> bool {
        self.items.is_empty()
    }

    /// The biggest item, without removing it.
    pub fn peek(&self) -> Option<&T> {
        self.items.first()
    }

    pub fn push(&mut self, value: T) {
        self.items.push(value);
        let last = self.items.len() - 1;
        sift_up(&mut self.items, last);
    }

    /// Removes and returns the biggest item.
    pub fn pop(&mut self) -> Option<T> {
        if self.items.is_empty() {
            return None;
        }
        let last = self.items.len() - 1;
        self.items.swap(0, last); // the last item goes to the root ...
        let top = self.items.pop(); // ... and the old root leaves
        let len = self.items.len();
        sift_down(&mut self.items, 0, len); // repair the heap
        top
    }
}

impl<T: Ord> Default for MaxHeap<T> {
    fn default() -> Self {
        Self::new()
    }
}
```

**Что происходит:**

1. `sift_up` повторяет: вычислить индекс родителя, остановиться, если элемент не больше родителя (`<=`), иначе поменять и продолжить с места родителя. Не больше одного шага на уровень: O(log n).
2. `sift_down` смотрит на элемент и двух его потомков (если они есть: `left < len`), находит наибольший из трёх и останавливается, если это сам элемент. Иначе меняет его с большим потомком и продолжает там.
   Параметр `len` говорит, сколько элементов в начале среза принадлежат куче; это нужно `heap_sort` ниже.
3. Обе функции работают со срезом `&mut [T]`, поэтому подходят и для `Vec`, и для любого другого среза (следующий раздел это использует).
4. `push` добавляет в конец и просеивает новый элемент вверх. `pop` меняет корень с последним элементом, удаляет последний элемент (старый корень — это результат) через `Vec::pop` и просеивает новый корень вниз. Пустая куча возвращает `None`.
5. `from_vec` **строит кучу**: вызывает `sift_down` для каждого узла, у которого есть потомки, от последнего родителя (`len / 2 - 1`) назад к корню. Элементам у основания нужно мало работы, а их много; в сумме выходит **O(n)**, лучше,
   чем `n` вызовов `push`, которые стоили бы O(n log n).
6. `peek` — это `items.first()`: максимум лежит на индексе 0.
7. Реализация `Default` нужна, потому что Clippy требует её, когда у типа есть `new()` без аргументов (Урок 07.11).

## 4. Пирамидальная сортировка

Куча даёт изящный алгоритм сортировки. Постройте max-кучу прямо внутри среза. Наибольший элемент стоит в начале: обменяйте его с последним элементом среза, это его окончательное место, и уменьшите кучу на один. Новый корень,
скорее всего, неверен, поэтому просейте его вниз. Повторите. Сортируем `[4, 1, 3, 2]`:

```text
start:                 4 1 3 2
build a max-heap:      4 2 3 1               (sift down at index 1: 1 and 2 swap)
swap root with end:    1 2 3 | 4             the part after | is final; repair the heap of 3 items:
                       3 2 1 | 4
swap root with end:    1 2 | 3 4             repair:
                       2 1 | 3 4
swap root with end:    1 | 2 3 4
done:                  1 2 3 4
```

Добавьте функцию ниже `MaxHeap`:

```rust
/// Heapsort: sorts in place in O(n log n), not stable.
pub fn heap_sort<T: Ord>(v: &mut [T]) {
    let n = v.len();
    for i in (0..n / 2).rev() {
        sift_down(v, i, n); // build a max-heap
    }
    for end in (1..n).rev() {
        v.swap(0, end); // the biggest item goes to its final place
        sift_down(v, 0, end); // repair the heap of the first `end` items
    }
}
```

**Что происходит:** первый цикл — то же построение кучи, что и в `from_vec`. Второй цикл делает `n − 1` раундов, в каждом один обмен и один `sift_down` стоимостью O(log n), поэтому всего **O(n log n) в лучшем, среднем и худшем случае**.
Дополнительная память: **O(1)**, сортировка *на месте*. Она **неустойчива**. Пирамидальная сортировка — надёжный запасной вариант быстрых сортировок: её худший случай гарантирован (это то, что `sort_unstable` использует как аварийный выход, Урок 07.9), но на типичных данных она
медленнее быстрой сортировки, потому что прыгает по вектору далеко, и процессор не может угадать, откуда читать дальше.

## 5. Тестирование

В стандартной библиотеке есть `BinaryHeap`, так что эталон готов. Мы также пишем помощник, который проверяет свойство кучи напрямую, и вызываем его после каждой операции. Добавьте в конец `src/heap.rs`:

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use crate::testutil::Rng;
    use std::collections::BinaryHeap;

    /// Is every parent at least as big as its children?
    fn is_heap<T: Ord>(v: &[T]) -> bool {
        (1..v.len()).all(|i| v[(i - 1) / 2] >= v[i])
    }

    /// `len` pseudo-random numbers from 0 to `max - 1` (as in Lesson 07.8).
    fn random_vec(seed: u64, len: usize, max: u64) -> Vec<i32> {
        let mut rng = Rng::new(seed);
        (0..len).map(|_| (rng.next_u64() % max) as i32).collect()
    }

    #[test]
    fn empty_heap() {
        let mut heap: MaxHeap<i32> = MaxHeap::new();
        assert!(heap.is_empty());
        assert_eq!(heap.peek(), None);
        assert_eq!(heap.pop(), None);
    }

    #[test]
    fn pops_in_decreasing_order() {
        let mut heap = MaxHeap::new();
        for value in [5, 3, 8, 1, 9, 2, 8] {
            heap.push(value);
            assert!(is_heap(&heap.items));
        }
        assert_eq!(heap.len(), 7);
        assert_eq!(heap.peek(), Some(&9));
        let mut out = Vec::new();
        while let Some(value) = heap.pop() {
            assert!(is_heap(&heap.items));
            out.push(value);
        }
        assert_eq!(out, [9, 8, 8, 5, 3, 2, 1]);
    }

    #[test]
    fn behaves_like_std_binary_heap() {
        for seed in 1..=5u64 {
            let mut rng = Rng::new(seed);
            let mut ours = MaxHeap::new();
            let mut model = BinaryHeap::new();
            for _ in 0..1000 {
                if rng.below(3) == 0 {
                    assert_eq!(ours.pop(), model.pop());
                } else {
                    let value = rng.below(100) as i32;
                    ours.push(value);
                    model.push(value);
                }
                assert_eq!(ours.peek(), model.peek());
                assert_eq!(ours.len(), model.len());
            }
        }
    }

    #[test]
    fn from_vec_builds_a_heap() {
        for len in 0..50 {
            let heap = MaxHeap::from_vec(random_vec(len as u64 + 1, len, 30));
            assert!(is_heap(&heap.items));
            assert_eq!(heap.len(), len);
        }
    }

    #[test]
    fn heap_sort_matches_std() {
        for len in 0..60 {
            for seed in 1..=4 {
                for max in [3, 1000] {
                    let mut actual = random_vec(seed, len, max);
                    let mut expected = actual.clone();
                    expected.sort();
                    heap_sort(&mut actual);
                    assert_eq!(actual, expected, "len {len}, seed {seed}, max {max}");
                }
            }
        }
    }
}
```

**Что происходит:**

1. `is_heap` проверяет определение для каждого индекса `i ≥ 1`: его родитель `(i - 1) / 2` должен быть не меньше. Тесты вызывают её после **каждого** push и pop, поэтому ошибка находится на той операции, которая её вызвала.
2. `pops_in_decreasing_order` добавляет семь чисел (с дубликатом `8`), затем извлекает их все. Куча, которая всегда извлекает максимум, должна выдать числа по убыванию: она сортирует. Именно поэтому `pop` — шаг сортировки в пирамидальной сортировке.
3. `behaves_like_std_binary_heap` выполняет 1000 случайных операций на каждое зерно, примерно треть — извлечения и две трети — добавления чисел меньше 100 (много дубликатов), на нашей куче и на `std::collections::BinaryHeap` и каждый раз сравнивает `pop`, `peek` и `len`
   (дифференциальное тестирование, Урок 07.8).
4. `from_vec_builds_a_heap` проверяет построение кучи для каждой длины от 0 до 49. `heap_sort_matches_std` сравнивает пирамидальную сортировку с `sort`, как в предыдущих уроках о сортировке.

```bash
cd ~/projects/rust/code/p07-algorithms
cargo test heap
```

```text
running 5 tests
test heap::tests::empty_heap ... ok
test heap::tests::pops_in_decreasing_order ... ok
test heap::tests::from_vec_builds_a_heap ... ok
test heap::tests::behaves_like_std_binary_heap ... ok
test heap::tests::heap_sort_matches_std ... ok

test result: ok. 5 passed; 0 failed; 0 ignored; 0 measured; 41 filtered out; finished in 0.00s
```

(Отфильтрован 41 тест из основного текста Уроков 07.1–07.11; с тестами из упражнений ваше число будет больше.)

## 6. Стандартная библиотека: `BinaryHeap` и `Reverse`

В настоящем коде используйте `std::collections::BinaryHeap`. Это max-куча с теми же методами: `push`, `pop`, `peek`, `len`, `is_empty`, `BinaryHeap::from(vec)` (построение кучи) и `into_sorted_vec()`.
Её элементы должны реализовывать `Ord`. Для **min-кучи** оберните элементы в `Reverse`. Структура получает свой порядок из `#[derive(PartialOrd, Ord)]`, который сравнивает поля **в том порядке, в каком они записаны**.
Создайте `examples/e12_scheduler.rs`:

```rust
use std::cmp::Reverse;
use std::collections::BinaryHeap;

/// The derived `Ord` compares `priority` first, then `name`.
#[derive(Debug, PartialEq, Eq, PartialOrd, Ord)]
struct Task {
    priority: u8,
    name: String,
}

fn main() {
    let mut queue = BinaryHeap::new();
    for (priority, name) in [
        (2, "write report"),
        (9, "fix outage"),
        (5, "review code"),
        (9, "call customer"),
    ] {
        queue.push(Task {
            priority,
            name: name.to_string(),
        });
    }

    println!("next: {:?}", queue.peek().map(|task| &task.name));
    while let Some(task) = queue.pop() {
        println!("{} {}", task.priority, task.name);
    }

    // A min-heap: wrap the items in `Reverse`, which flips the order.
    let mut smallest_first = BinaryHeap::new();
    for n in [5, 1, 8, 3] {
        smallest_first.push(Reverse(n));
    }
    while let Some(Reverse(n)) = smallest_first.pop() {
        print!("{n} ");
    }
    println!();
}
```

```bash
cargo run --example e12_scheduler
```

```text
next: Some("fix outage")
9 fix outage
9 call customer
5 review code
2 write report
1 3 5 8 
```

**Что происходит:**

1. `Task` сравнивается сначала по `priority`. Две задачи с приоритетом 9 сравниваются по `name`: `"fix outage"` больше, чем `"call customer"`, потому что `f` стоит после `c`, поэтому она выходит первой.
   Если среди равных нужно сначала более старую задачу, добавьте поле-счётчик и сравнивайте его через `Reverse`.
2. `queue.peek()` смотрит на лучшую задачу, не забирая её; `pop` удаляет задачи по одной в порядке приоритета.
3. `Reverse(n)` — кортежная структура, переворачивающая сравнение. Max-куча теперь первым извлекает наименьшее число, а образец `Some(Reverse(n))` достаёт число из обёртки.

Вы будете использовать `BinaryHeap` с `Reverse` для алгоритма кратчайшего пути Дейкстры в Уроке 07.14, где куча хранит «ближайшее непосещённое место».

## Частые ошибки

**1. Забыли, что `BinaryHeap` — это max-куча.** Вы хотели наименьшее число, а получаете наибольшее:

```rust
use std::collections::BinaryHeap;

fn main() {
    let mut heap = BinaryHeap::new();
    for n in [5, 1, 8, 3] {
        heap.push(n);
    }
    println!("first pop: {:?}", heap.pop());
}
```

```text
first pop: Some(8)
```

Оберните числа в `Reverse` (раздел 6), чтобы первым получать `1`.

**2. Чтение кучи по порядку через `iter()`.** Итератор обходит внутренний вектор, который **не отсортирован**; гарантированно наибольшим является только первый элемент:

```rust
use std::collections::BinaryHeap;

fn main() {
    let heap = BinaryHeap::from(vec![1, 5, 2, 8, 3]);
    println!("iter:   {:?}", heap.iter().collect::<Vec<_>>());
    println!("sorted: {:?}", heap.into_sorted_vec());
}
```

```text
iter:   [8, 5, 2, 1, 3]
sorted: [1, 2, 3, 5, 8]
```

Чтобы получить элементы по порядку, извлекайте их по одному через `pop` (это опустошает кучу) или вызовите `into_sorted_vec()` (по возрастанию). Не полагайтесь на порядок `iter()`.

**3. `f64` в куче.** У чисел с плавающей точкой нет полного порядка (`NaN`, Урок 07.8), поэтому они не реализуют `Ord`:

```rust
use std::collections::BinaryHeap;

fn main() {
    let mut heap = BinaryHeap::new();
    heap.push(2.5_f64);
    heap.push(0.5);
}
```

```text
error[E0277]: the trait bound `f64: Ord` is not satisfied
 --> src/main.rs:5:10
  |
5 |     heap.push(2.5_f64);
  |          ^^^^ ------- `f64` doesn't satisfy the trait bound
  |          |
  |          the trait `Ord` is not implemented for `f64`
```

Компилятор говорит, что `f64` не `Ord`. Исправление: оберните число в небольшую структуру, реализующую `Ord` через `f64::total_cmp`, или используйте целочисленные приоритеты (например, миллисекунды вместо секунд в виде `f64`).

## Попробуйте сами

1. Добавьте `heap.rs` шаг за шагом: две функции просеивания, `MaxHeap`, `heap_sort` и тесты. Запустите `cargo test heap`.
2. В `sift_up` замените `<=` на `>=`. Запустите тесты. Какие тесты падают (их два) и что говорит сообщение `pops_in_decreasing_order`? Верните как было.
3. Запустите `cargo run --example e12_scheduler`. Измените `Task` так, чтобы в структуре `name` было записано раньше `priority`. Каков теперь порядок и почему?
4. Запустите `cargo fmt`, `cargo clippy --all-targets` и сделайте коммит: `git add code/p07-algorithms && git commit -m "feat(p07): binary heap and heapsort"`.

## Упражнения

### Упражнение 1 — Трассировка кучи (легко)

Добавьте `5, 3, 8, 1, 9` по очереди в пустую max-кучу и запишите вектор после каждого добавления. Затем сделайте один `pop` и запишите вектор.

<details>
<summary>Подсказка</summary>

После каждого добавления сравнивайте новый последний элемент с его родителем `(i - 1) / 2` и меняйте, пока он больше. Для `pop` обменяйте первый и последний элементы, удалите последний и просейте новый первый вниз по большему потомку.

</details>

<details>
<summary>Ответ</summary>

```text
push 5:  [5]
push 3:  [5, 3]
push 8:  [5, 3, 8] -> 8 > 5, swap:           [8, 3, 5]
push 1:  [8, 3, 5, 1]                        1 < 3, stays
push 9:  [8, 3, 5, 1, 9] -> 9 > 3, swap:     [8, 9, 5, 1, 3]
                         -> 9 > 8, swap:     [9, 8, 5, 1, 3]
pop:     swap first and last:                [3, 8, 5, 1 | 9]  -> 9 is the result
         sift down 3: bigger child is 8:     [8, 3, 5, 1]
         children of 3 (index 1): only 1:    1 < 3, stop
```

Результат: после извлечения вектор равен `[8, 3, 5, 1]`.

</details>

### Упражнение 2 — Тест свойства (легко)

Напишите тест, который смешивает добавления и извлечения 300 псевдослучайных чисел (извлечение после каждого третьего добавления) и проверяет `is_heap` после каждой операции.

<details>
<summary>Подсказка</summary>

Используйте `random_vec(5, 300, 50)` и `enumerate()`. После каждого `push`, если `i % 3 == 2`, вызывайте `heap.pop()`. Затем `assert!(is_heap(&heap.items))`.

</details>

<details>
<summary>Ответ</summary>

```rust
    #[test]
    fn is_heap_holds_for_random_pushes_and_pops() {
        let mut heap = MaxHeap::new();
        for (i, value) in random_vec(5, 300, 50).into_iter().enumerate() {
            heap.push(value);
            if i % 3 == 2 {
                heap.pop();
            }
            assert!(is_heap(&heap.items));
        }
    }
```

Почему это работает: свойство вроде «каждый родитель не меньше своих потомков» проверяется на каждом шаге при малом диапазоне значений (от 0 до 49), поэтому дубликатов много. Если бы `sift_down` когда-нибудь выбрал не того потомка, `is_heap` упала бы именно на этой операции.

</details>

### Упражнение 3 — Назовите сложность (средне)

1. Сколько стоят `n` вызовов `push` по сравнению с `from_vec` для `n` элементов?
2. Сколько стоит получение элементов в отсортированном порядке через `n` вызовов `pop`?
3. У вас есть поток из `n` чисел, и нужны `k` наибольших. Это решает min-куча, хранящая не более `k` элементов (когда она вырастает до `k + 1`, извлеките наименьший): каковы время и дополнительная память через `n` и `k`?

<details>
<summary>Подсказка</summary>

Для (3): каждое из `n` чисел добавляется один раз и извлекается не более одного раза, а в куче не более `k + 1` элементов, поэтому каждая операция стоит O(log k).

</details>

<details>
<summary>Ответ</summary>

1. `n` вызовов `push` стоят **O(n log n)**; `from_vec` стоит **O(n)**, потому что большинство элементов находится у основания и сдвигается всего на несколько шагов.
2. **O(n log n)**: `n` извлечений по O(log n). Это пирамидальная сортировка с дополнительной копией.
3. Время **O(n log k)**, дополнительная память **O(k)**. Если `k` намного меньше `n`, это лучше, чем сортировать всё (O(n log n), O(n) памяти), и работает на потоке, который нельзя держать в памяти.

</details>

### Упражнение 4 — k наибольших (средне)

Напишите `k_largest(values: &[i32], k: usize) -> Vec<i32>`, возвращающую `k` наибольших значений, наибольшее первым (меньше, если значений меньше `k`), используя `BinaryHeap` из значений `Reverse`, как в Упражнении 3. Напишите тест, сравнивающий её с сортировкой.

<details>
<summary>Подсказка</summary>

Добавляйте `Reverse(value)` для каждого значения; когда `heap.len() > k`, делайте `pop` (это отбрасывает наименьшее). В конце превратите кучу в вектор обычных чисел через `into_iter().map(|Reverse(v)| v)`, затем отсортируйте его по убыванию. Добавьте `use std::cmp::Reverse; use std::collections::BinaryHeap;` в начало файла.

</details>

<details>
<summary>Ответ</summary>

```rust
/// The `k` biggest values of `values`, biggest first.
pub fn k_largest(values: &[i32], k: usize) -> Vec<i32> {
    // A min-heap holding the k biggest values seen so far; its root is the smallest of them.
    let mut heap = BinaryHeap::with_capacity(k + 1);
    for &value in values {
        heap.push(Reverse(value));
        if heap.len() > k {
            heap.pop(); // drop the smallest: it cannot be among the k biggest
        }
    }
    let mut result: Vec<i32> = heap.into_iter().map(|Reverse(v)| v).collect();
    result.sort_by(|a, b| b.cmp(a));
    result
}
```

Тест внутри модуля `tests`:

```rust
    #[test]
    fn k_largest_matches_sorting() {
        for len in 0..40 {
            let values = random_vec(len as u64 + 3, len, 20);
            let mut sorted = values.clone();
            sorted.sort_by(|a, b| b.cmp(a));
            for k in 0..=len + 2 {
                let expected: Vec<i32> = sorted.iter().take(k).copied().collect();
                assert_eq!(k_largest(&values, k), expected, "len {len}, k {k}");
            }
        }
    }
```

Почему это работает: куча всегда содержит `k` наибольших из виденных значений, а наименьшее из них лежит в корне, готовое вылететь, когда придёт большее значение. Тест охватывает `k = 0` и `k` больше длины, где `take(k)` даёт меньше элементов.

</details>

### Упражнение 5 — Слияние отсортированных списков (сложно)

Напишите `merge_sorted(lists: &[Vec<i32>]) -> Vec<i32>`, которая сливает `k` отсортированных списков в один отсортированный вектор с помощью min-кучи, хранящей не более одной записи на список. Какова временная сложность для `N` значений всего? Проверьте её против `concat` плюс `sort`.

<details>
<summary>Подсказка</summary>

Положите `Reverse((value, list_index, position))` в кучу для первого элемента каждого непустого списка. Извлеките наименьшую запись, добавьте её значение в результат и добавьте в кучу следующий элемент того же списка (`lists[list_index].get(position + 1)`), если он есть. Кортежи сравниваются поле за полем, поэтому наименьшее значение идёт первым.

</details>

<details>
<summary>Ответ</summary>

```rust
/// Merges sorted lists into one sorted vector in O(N log k) for k lists with N values in total.
pub fn merge_sorted(lists: &[Vec<i32>]) -> Vec<i32> {
    // Each entry is (value, which list, position in that list); the smallest comes out first.
    let mut heap = BinaryHeap::new();
    for (list_index, list) in lists.iter().enumerate() {
        if let Some(&first) = list.first() {
            heap.push(Reverse((first, list_index, 0)));
        }
    }
    let mut merged = Vec::new();
    while let Some(Reverse((value, list_index, position))) = heap.pop() {
        merged.push(value);
        if let Some(&next) = lists[list_index].get(position + 1) {
            heap.push(Reverse((next, list_index, position + 1)));
        }
    }
    merged
}
```

Тест внутри модуля `tests`:

```rust
    #[test]
    fn merge_sorted_matches_sorting() {
        let lists = vec![vec![1, 4, 9], vec![], vec![2, 2, 8, 10], vec![3]];
        assert_eq!(merge_sorted(&lists), [1, 2, 2, 3, 4, 8, 9, 10]);
        for seed in 1..=5u64 {
            let lists: Vec<Vec<i32>> = (0..6)
                .map(|i| {
                    let mut v = random_vec(seed * 10 + i, (i * 3) as usize, 100);
                    v.sort();
                    v
                })
                .collect();
            let mut expected: Vec<i32> = lists.concat();
            expected.sort();
            assert_eq!(merge_sorted(&lists), expected);
        }
    }
```

Почему это работает: куча хранит передний элемент каждого списка, поэтому её наименьшая запись — наименьшее ещё не использованное значение. Каждое значение добавляется один раз и извлекается один раз, в куче не более `k` записей: **O(N log k)**. Это шаг слияния из Урока 07.9 для `k` списков вместо двух, и именно так внешняя сортировка сливает большие отсортированные файлы.

</details>

## Вопросы для самопроверки

1. Что такое свойство кучи для max-кучи?

<details>
<summary>Ответ</summary>

Каждый родитель больше или равен обоим своим потомкам. Поэтому наибольший элемент находится в корне.

</details>

2. Где находятся родитель и потомки узла с индексом `i` в двоичной куче на основе вектора?

<details>
<summary>Ответ</summary>

Родитель — на `(i - 1) / 2`, потомки — на `2 * i + 1` и `2 * i + 2`.

</details>

3. Почему `pop` работает за O(log n)?

<details>
<summary>Ответ</summary>

Он меняет корень с последним элементом и просеивает его вниз. Просеивание на каждом шаге спускается на один уровень, а у полного двоичного дерева из `n` элементов около log₂ n уровней.

</details>

4. Как получить min-кучу из `BinaryHeap`?

<details>
<summary>Ответ</summary>

Оберните элементы в `std::cmp::Reverse`: `heap.push(Reverse(x))` и читайте обратно через `Some(Reverse(x)) = heap.pop()`.

</details>

5. Это та же куча, что и куча памяти из Этапа 03?

<details>
<summary>Ответ</summary>

Нет. Эта куча — структура данных (дерево, хранящееся в векторе, где родители больше потомков). Куча памяти — область, где программа хранит данные, переживающие вызов функции. Общее только название.

</details>

## Итоги

- Очередь с приоритетом первым возвращает элемент с наивысшим приоритетом. Двоичная куча делает `push` и `pop` за O(log n), а `peek` — за O(1).
- Куча — это полное двоичное дерево, хранящееся в векторе: родитель `i` находится на `(i - 1) / 2`, потомки — на `2i + 1` и `2i + 2`. Указатели не нужны.
- `push` добавляет в конец и просеивает вверх; `pop` переносит последний элемент в корень и просеивает вниз; построение кучи занимает O(n).
- Пирамидальная сортировка строит кучу в срезе и многократно переносит корень в конец: всегда O(n log n), O(1) дополнительной памяти, неустойчива.
- Куча упорядочивает только родителей относительно потомков, поэтому не умеет искать и обходить по порядку; `iter()` у `BinaryHeap` не отсортирован.
- Используйте `std::collections::BinaryHeap` (max-куча), `Reverse` для min-кучи и `#[derive(Ord)]` с самым важным полем первым.
- Типичные применения: планировщики, k наибольших значений за O(n log k), слияние k отсортированных списков за O(N log k) и алгоритм Дейкстры (Урок 07.14).

## Источники и что почитать

- [Rust standard library — std::collections::BinaryHeap](https://doc.rust-lang.org/std/collections/struct.BinaryHeap.html) — max-куча: методы, пример с Дейкстрой и стоимость каждой операции.
- [Rust standard library — std::cmp::Reverse](https://doc.rust-lang.org/std/cmp/struct.Reverse.html) — обёртка, переворачивающая порядок.
- [Rust standard library — Ord and derive](https://doc.rust-lang.org/std/cmp/trait.Ord.html) — как порядок полей определяет производный `Ord`.
- [Rust standard library — slice::select_nth_unstable](https://doc.rust-lang.org/std/primitive.slice.html#method.select_nth_unstable) — когда нужен только k-й элемент, а не куча или полная сортировка.

---

[🏠 Начало](../START-HERE.md) · [← 07.11 Деревья и двоичные деревья поиска](11-trees-and-binary-search-trees.md) · [📋 Этап 07](README.md) · [07.13 Графы: BFS и DFS →](13-graphs-bfs-and-dfs.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
