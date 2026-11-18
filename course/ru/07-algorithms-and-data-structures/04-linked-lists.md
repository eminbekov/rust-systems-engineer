<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 07.3 Массивы и векторы изнутри](03-arrays-and-vectors-under-the-hood.md) · [📋 Этап 07](README.md) · [07.5 Стеки и очереди →](05-stacks-and-queues.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/07-algorithms-and-data-structures/04-linked-lists.md) <!-- nav -->

# Урок 07.4 — Связные списки

> **Этап 07 — Алгоритмы и структуры данных** · Урок 4 из 20 · ⏱ около 2,5 часа (2 занятия)

## Что вы узнаете

- Что такое связный список: узлы, каждый из которых указывает на следующий узел, вместо элементов в одном блоке.
- Как построить односвязный список на безопасном Rust через `Option<Box<Node<T>>>`, включая `push_front`, `pop_front`, `push_back`, `reverse` и итератор.
- Почему освобождение очень длинного списка требует написанного вручную `Drop` и как выглядит переполнение стека.
- Почему связный список почти всегда медленнее `Vec` и в каких немногих случаях он — правильный инструмент.

## Перед началом

- Вы прошли [Урок 07.3 — Массивы и векторы изнутри](03-arrays-and-vectors-under-the-hood.md).
- Вы знаете `Box` и рекурсивные типы ([Урок 05.9](../05-rust-intermediate/09-smart-pointers.md), раздел 3), `Option` и его методы `take`, `map`, `as_ref`, `as_deref` ([Урок 04.4](../04-structs-enums-collections-errors/04-option-and-match.md)) и умеете писать итератор ([Урок 05.5](../05-rust-intermediate/05-iterators.md), раздел 8).

## Новые слова

| Слово | Значение |
|---|---|
| **связный список** (linked list) | Цепочка **узлов**. Каждый узел хранит значение и ссылку на следующий узел. |
| **узел** (node) | Один элемент цепочки: значение плюс ссылка. Каждый узел — отдельное выделение в куче. |
| **голова** (head) | Первый узел. Список хранит только его. Ссылка последнего узла пуста (`None`). |
| **односвязный** (singly linked) | Каждый узел знает только следующий узел, но не предыдущий. (*Двусвязный* список знает оба.) |
| **курсор** (cursor) | Переменная, которая указывает на место в структуре данных и шаг за шагом движется по ней. |
| **погоня за указателями** (pointer chasing) | Переход по ссылке к узлу, который может лежать где угодно в памяти. Медленно, потому что процессор не может заранее предсказать, куда смотреть дальше. |
| **кэш** (cache) | Небольшая очень быстрая память внутри процессора, хранящая недавно использованные данные и их соседей. |
| **оптимизация ниши** (niche optimization) | Приём Rust: хранить `None` как значение, которого у типа быть не может (для `Box` — адрес 0), так что `Option<Box<T>>` не требует дополнительного места. |
| **переполнение стека** (stack overflow) | Падение, когда на стеке кончается место (Урок 03.1). |

## 1. Идея

Массив хранит элементы в одном блоке, рядом друг с другом. **Связный список** делает наоборот: каждый элемент живёт
в своём маленьком кусочке памяти в куче — **узле**, а узел хранит адрес следующего.

```text
head
 │
 ▼
┌────────┬───┐   ┌────────┬───┐   ┌────────┬──────┐
│ value 7│ ●─┼──▶│ value 3│ ●─┼──▶│ value 9│ None │
└────────┴───┘   └────────┴───┘   └────────┴──────┘
   node A           node B           node C          (three separate heap allocations)
```

Список хранит только `head`. Чтобы добраться до третьего элемента, надо начать с головы и пройти по двум ссылкам:
короткой формулы `address = start + i * size` нет. Зато взамен вы получаете дешёвые изменения в начале: чтобы добавить
узел в начало, его создают, направляют на старую голову и делают новой головой. Больше ничего не двигается.

| Операция | Односвязный список | `Vec` |
|---|---|---|
| прочитать элемент `i` | `O(i)`: пройти `i` ссылок | `O(1)` |
| добавить или удалить в начале | `O(1)` | `O(n)` |
| добавить или удалить в конце | `O(n)`: дойти до конца (`O(1)` с указателем на последний узел) | `O(1)` амортизированно |
| вставить после узла, который у вас уже есть | `O(1)` | `O(n)` |
| найти значение | `O(n)` | `O(n)` |

## 2. Узлы в Rust

Узел содержит следующий узел, поэтому тип рекурсивный. Урок 05.9 показал, что рекурсивному типу нужна косвенность, а
её даёт `Box`. «Ссылка — это либо пусто, либо узел в коробке» — это в точности `Option<Box<Node<T>>>`. Создайте
`src/linked_list.rs`, добавьте `pub mod linked_list;` в `src/lib.rs` и начните с типов:

```rust
//! Lesson 07.4: a singly linked list.

type Link<T> = Option<Box<Node<T>>>;

struct Node<T> {
    value: T,
    next: Link<T>,
}

/// A singly linked list. `head` is the first node (or `None` for an empty list).
pub struct List<T> {
    head: Link<T>,
    len: usize,
}

impl<T> List<T> {
    pub fn new() -> List<T> {
        List { head: None, len: 0 }
    }

    pub fn len(&self) -> usize {
        self.len
    }

    pub fn is_empty(&self) -> bool {
        self.len == 0
    }
```

**Что происходит:**

1. `type Link<T> = ...` даёт короткое имя длинному типу. *Псевдоним типа* (type alias) не создаёт нового типа.
2. `Node` хранит значение и ссылку на следующий узел. `Node` приватный: пользователи списка узлов не видят.
3. `List` хранит голову и, как в Уроке 07.3, сохранённую `len`, чтобы `len()` была `O(1)`.

Бесплатна ли пустая ссылка? `examples/e04_sizes.rs`:

```rust
use std::mem::size_of;

struct Node {
    _value: u64,
    _next: Option<Box<Node>>,
}

fn main() {
    println!("Box<Node>         {} bytes", size_of::<Box<Node>>());
    println!("Option<Box<Node>> {} bytes", size_of::<Option<Box<Node>>>());
    println!("Node              {} bytes", size_of::<Node>());
}
```

```text
Box<Node>         8 bytes
Option<Box<Node>> 8 bytes
Node              16 bytes
```

`Option<Box<Node>>` занимает те же 8 байт, что и `Box<Node>`: это **оптимизация ниши**. `Box` никогда не может быть
адресом 0, поэтому Rust использует 0 для `None`. «Пустая ссылка» не стоит ничего дополнительно.

## 3. Работа с началом и обход

Продолжайте внутри того же блока `impl<T> List<T>`:

```rust
    /// Adds a value at the front: O(1).
    pub fn push_front(&mut self, value: T) {
        let old_head = self.head.take();
        self.head = Some(Box::new(Node {
            value,
            next: old_head,
        }));
        self.len += 1;
    }

    /// Removes and returns the first value: O(1).
    pub fn pop_front(&mut self) -> Option<T> {
        self.head.take().map(|node| {
            self.head = node.next;
            self.len -= 1;
            node.value
        })
    }

    /// Looks at the first value without removing it: O(1).
    pub fn peek_front(&self) -> Option<&T> {
        self.head.as_ref().map(|node| &node.value)
    }

    /// Adds a value at the back. We must walk to the end first: O(n).
    pub fn push_back(&mut self, value: T) {
        let mut cursor = &mut self.head;
        while let Some(node) = cursor {
            cursor = &mut node.next;
        }
        *cursor = Some(Box::new(Node { value, next: None }));
        self.len += 1;
    }
```

**Что происходит:**

1. `push_front`: `self.head.take()` вынимает старую голову и оставляет на её месте `None` (`Option::take`, Урок 07.3).
   Затем создаётся новый узел, указывающий на старую голову, и он становится головой. Четыре шага при любом `n`: `O(1)`.
2. `pop_front`: `self.head.take()` убирает первый узел, если он есть. `.map(|node| ...)` выполняется только для `Some`:
   голова становится `node.next` (второй узел, вынутый из коробки), длина уменьшается, и значение возвращается.
3. `peek_front` возвращает ссылку на первое значение. `as_ref()` превращает `&Option<Box<Node>>` в
   `Option<&Box<Node>>`, так что ничего не перемещается.
4. У `push_back` короткого пути нет. `cursor` — это `&mut Link<T>`: изменяемая ссылка на *место, где хранится ссылка*.
   Цикл `while let` двигает курсор от ссылки к ссылке, пока он не укажет на пустую ссылку в конце. Запись в это место
   (`*cursor = Some(...)`) присоединяет новый узел. Это занимает `O(n)`.

```text
push_front(1) on  head -> [2] -> [3]

1. take the old head     old_head = [2] -> [3]       head = None
2. new node points to it            [1] -> [2] -> [3]
3. new node is the head       head -> [1] -> [2] -> [3]
```

Дальше: разворот списка на месте и итератор.

```rust
    /// Reverses the list in place by turning every `next` link around: O(n).
    pub fn reverse(&mut self) {
        let mut reversed: Link<T> = None;
        let mut rest = self.head.take();
        while let Some(mut node) = rest {
            rest = node.next.take();
            node.next = reversed;
            reversed = Some(node);
        }
        self.head = reversed;
    }

    /// Walks over the values from the first to the last.
    pub fn iter(&self) -> Iter<'_, T> {
        Iter {
            next: self.head.as_deref(),
        }
    }
}

pub struct Iter<'a, T> {
    next: Option<&'a Node<T>>,
}

impl<'a, T> Iterator for Iter<'a, T> {
    type Item = &'a T;

    fn next(&mut self) -> Option<&'a T> {
        self.next.map(|node| {
            self.next = node.next.as_deref();
            &node.value
        })
    }
}
```

**Что происходит:**

1. `reverse` один раз проходит по списку и разворачивает каждую ссылку. `rest` — ещё не обработанная часть, `reversed` — уже
   развёрнутая. В каждом круге мы отрезаем первый узел от `rest`, направляем его на `reversed` и делаем новым началом
   `reversed`. Ни один узел не выделяется, ни один не копируется: `O(n)` времени, `O(1)` дополнительной памяти.
2. `Iter` хранит `next: Option<&Node<T>>` — ссылку на узел, который он покажет следующим. Его метод `next` возвращает
   ссылку на значение этого узла и переходит к следующему узлу через `as_deref()`, который превращает
   `&Option<Box<Node>>` в `Option<&Node>`. Время жизни `'a` говорит: выданные ссылки живут столько же, сколько заимствование
   списка.

С итератором на нашем списке работают все стандартные инструменты: `for`, `map`, `sum`, `collect`. `examples/e04_list.rs`:

```rust
use p07_algorithms::linked_list::List;

fn main() {
    let mut list = List::new();
    list.push_front(2);
    list.push_front(1);
    list.push_back(3);
    list.push_back(4);
    println!("{:?}", list.iter().collect::<Vec<_>>());

    list.reverse();
    println!("{:?}", list.iter().collect::<Vec<_>>());

    println!("first = {:?}", list.pop_front());
    println!("peek  = {:?}", list.peek_front());
    println!("length = {}", list.len());
}
```

```text
[1, 2, 3, 4]
[4, 3, 2, 1]
first = Some(4)
peek  = Some(3)
length = 3
```

(`list.iter().collect::<Vec<_>>()` собирает ссылки в вектор для печати.)

## 4. Освобождение длинного списка

А теперь сюрприз. Когда значение выходит из области видимости, Rust *освобождает* его (Урок 05.9: `Drop`): по очереди
освобождает поля. Освобождение `Node` освобождает его `next`, то есть `Box<Node>`, который освобождает *свой* `next`...
Это **рекурсия**, по одному кадру стека (Урок 03.1) на узел. Списку из миллиона узлов нужен миллион кадров.
`examples/e04_overflow.rs` строит такой список **без** собственного `Drop`:

```rust
struct Node {
    value: u64,
    next: Option<Box<Node>>,
}

fn main() {
    let mut head: Option<Box<Node>> = None;
    for i in 0..1_000_000 {
        head = Some(Box::new(Node {
            value: i,
            next: head,
        }));
    }
    if let Some(node) = &head {
        println!(
            "built 1000000 nodes; first value {}, has next: {}",
            node.value,
            node.next.is_some()
        );
    }
    // `head` is dropped here
}
```

```text
built 1000000 nodes; first value 999999, has next: true

thread 'main' (4788096) has overflowed its stack
fatal runtime error: stack overflow, aborting
```

Список был построен успешно, а падение произошло на закрывающей скобке `main`, когда его освобождали: стек переполнился
(стек главного потока — около 8 МиБ, у потоков тестов меньше).

> 🔒 **Безопасность:** программу, которая строит длинный список из внешнего ввода и падает при его освобождении, можно
> нарочно обрушить, отправив длинный ввод. Переполнение стека — не паника, которую можно перехватить: процесс аварийно
> завершается.

Лекарство — освобождать узлы **в цикле**, а не рекурсией. Это последняя часть `src/linked_list.rs` (перед тестами):

```rust
impl<T> Default for List<T> {
    fn default() -> List<T> {
        List::new()
    }
}

/// Without this, dropping a very long list would use one stack frame per node.
impl<T> Drop for List<T> {
    fn drop(&mut self) {
        let mut rest = self.head.take();
        while let Some(mut node) = rest {
            rest = node.next.take();
        }
    }
}
```

**Что происходит:** `Drop::drop` вызывается, когда список умирает. Он забирает голову и идёт по цепочке; в каждом круге
`node.next.take()` отсоединяет следующий узел, поэтому когда переменная цикла `node` освобождается в конце круга, у неё
**нет** преемника, и освобождается только она сама: никакой рекурсии. (`Default` здесь потому, что Clippy просит его, когда
у типа есть `new()` без аргументов.)

## 5. Тесты

Два теста, как в Уроке 07.3: случайное сравнение с `Vec` как эталоном и тест на исправление переполнения стека. Внизу
`src/linked_list.rs`:

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use crate::testutil::Rng;

    fn contents(list: &List<u64>) -> Vec<u64> {
        list.iter().copied().collect()
    }

    #[test]
    fn behaves_like_vec_on_random_operations() {
        let mut rng = Rng::new(99);
        let mut list: List<u64> = List::new();
        let mut model: Vec<u64> = Vec::new();
        for step in 0..3000 {
            match rng.below(5) {
                0 => {
                    let x = rng.next_u64() % 100;
                    list.push_front(x);
                    model.insert(0, x);
                }
                1 => {
                    let x = rng.next_u64() % 100;
                    list.push_back(x);
                    model.push(x);
                }
                2 | 3 => {
                    let expected = if model.is_empty() {
                        None
                    } else {
                        Some(model.remove(0))
                    };
                    assert_eq!(list.pop_front(), expected);
                }
                _ => {
                    list.reverse();
                    model.reverse();
                }
            }
            assert_eq!(list.len(), model.len(), "step {step}");
            assert_eq!(contents(&list), model, "step {step}");
        }
    }

    #[test]
    fn dropping_a_very_long_list_does_not_overflow_the_stack() {
        let mut list = List::new();
        for i in 0..1_000_000u64 {
            list.push_front(i);
        }
        assert_eq!(list.len(), 1_000_000);
        // `list` is dropped here
    }
}
```

**Что происходит:**

1. Случайный тест делает 3 000 случайных шагов: `push_front` (модель делает `insert(0, ..)`), `push_back` (`push`),
   `pop_front` (вдвое чаще, `remove(0)`) и `reverse`. После каждого шага всё содержимое списка должно совпадать с моделью.
2. Второй тест строит список из миллиона узлов и даёт ему освободиться. Удалите реализацию `Drop` и запустите тест: процесс
   теста аварийно завершится из-за переполнения стека, а не пройдёт.

```text
running 2 tests
test linked_list::tests::behaves_like_vec_on_random_operations ... ok
test linked_list::tests::dropping_a_very_long_list_does_not_overflow_the_stack ... ok

test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 12 filtered out; finished in 0.03s
```

## 6. Измеряем: связный список против вектора

`benches/linked_list.rs` сравнивает наш список, `std::collections::LinkedList` (стандартный двусвязный список) и `Vec`.
Зарегистрируйте его в `Cargo.toml`, как в Уроке 07.2:

```toml
[[bench]]
name = "linked_list"
harness = false
```

```rust
use criterion::{Criterion, criterion_group, criterion_main};
use p07_algorithms::linked_list::List;
use std::collections::LinkedList;
use std::hint::black_box;

const N: u64 = 20_000;

fn bench_sum(c: &mut Criterion) {
    let vec: Vec<u64> = (0..N).collect();
    let mut list = List::new();
    let mut std_list = LinkedList::new();
    for i in 0..N {
        list.push_front(i);
        std_list.push_front(i);
    }

    let mut group = c.benchmark_group("sum 20_000 items");
    group.bench_function("Vec", |b| b.iter(|| black_box(&vec).iter().sum::<u64>()));
    group.bench_function("our List", |b| {
        b.iter(|| black_box(&list).iter().sum::<u64>())
    });
    group.bench_function("std LinkedList", |b| {
        b.iter(|| black_box(&std_list).iter().sum::<u64>())
    });
    group.finish();
}

fn bench_push_front(c: &mut Criterion) {
    let mut group = c.benchmark_group("push 20_000 items at the front");
    group.bench_function("Vec insert(0, ..)", |b| {
        b.iter(|| {
            let mut v = Vec::new();
            for i in 0..black_box(N) {
                v.insert(0, i);
            }
            v
        })
    });
    group.bench_function("our List", |b| {
        b.iter(|| {
            let mut list = List::new();
            for i in 0..black_box(N) {
                list.push_front(i);
            }
            list
        })
    });
    group.finish();
}

criterion_group!(benches, bench_sum, bench_push_front);
criterion_main!(benches);
```

Запустите `cargo bench --bench linked_list -- --warm-up-time 1 --measurement-time 3`:

```text
sum 20_000 items/Vec    time:   [1.3024 µs 1.3063 µs 1.3109 µs]
sum 20_000 items/our List
                        time:   [22.193 µs 22.245 µs 22.301 µs]
sum 20_000 items/std LinkedList
                        time:   [23.972 µs 24.045 µs 24.133 µs]
push 20_000 items at the front/Vec insert(0, ..)
                        time:   [17.680 ms 17.769 ms 17.904 ms]
push 20_000 items at the front/our List
                        time:   [276.65 µs 277.55 µs 278.51 µs]
```

(Строки прогресса и выбросов опущены. Apple M4 Pro; **ваши числа будут другими**.)

**Как это читать:**

1. **Сумма 20 000 чисел**: `Vec` — 1,3 мкс, наш список — 22 мкс, `std::LinkedList` — 24 мкс. Список примерно в **17 раз
   медленнее**, хотя все три — `O(n)`. Вектор читается по памяти по порядку, и процессор загружает соседей в кэш до того,
   как они понадобятся. Список — это погоня за указателями: каждый узел нужно получить, прежде чем процессор вообще узнает,
   где лежит следующий. (Здесь узлы выделялись один за другим, так что лежат близко. В настоящей программе, которая часами
   добавляет и удаляет узлы, они разбросаны, и разрыв больше.)
2. **Добавление 20 000 чисел в начало**: `Vec::insert(0, ..)` — 17,8 мс, наш список — 0,28 мс. Это тот случай, для которого
   список и создан: `O(1)` против `O(n)`, так что список **быстрее в 64 раза**. `VecDeque` из Урока 07.5 здесь тоже `O(1)` и
   сохраняет преимущество кэша.

> ⚡ **Производительность:** «Связный список» и «быстро» на современном процессоре встречаются редко. Big-O считает шаги и не
> видит кэша. Сначала берите `Vec` или `VecDeque`.

## 7. Когда использовать связный список

- **Почти всегда: не использовать.** `Vec` (или `VecDeque` для обоих концов) проще и быстрее для стеков, очередей и
  последовательностей.
- **Использовать, когда** нужна вставка или удаление за `O(1)` в *середине* и позиция у вас уже есть (текстовый буфер
  редактора, LRU-кэш (Least Recently Used, «давно не использовавшийся»), который выбрасывает элемент, не использовавшийся дольше всех), или когда нужно соединить два списка за `O(1)`. Для этого берите `std::collections::LinkedList`.
- **Всё равно изучить.** Связанные узлы — строительный блок деревьев (Урок 07.11) и графов (Урок 07.13), а приём с `take()` и
  курсорами `&mut` встречается во всех них.

Двусвязному списку (каждый узел знает и предыдущий) нужно общее владение между соседями: `Rc`, `RefCell` и `Weak` (Уроки 05.9
и 06.4) или `unsafe`. Поэтому `std::collections::LinkedList` внутри написан с `unsafe`, и поэтому мы останавливаемся на
односвязном.

## Частые ошибки

**1. Забытый `take()`.** Запись `self.head = Some(Box::new(Node { value, next: self.head }))` в `push_front` не компилируется:

```text
error[E0507]: cannot move out of `self.head` which is behind a mutable reference
  --> src/main.rs:14:19
   |
14 |             next: self.head,
   |                   ^^^^^^^^^ move occurs because `self.head` has type `Option<Box<Node>>`, which does not implement the `Copy` trait
```

`self.head` заимствуется через `&mut self`, поэтому значение нельзя вынуть и оставить дыру. `take()` вынимает значение *и*
оставляет на его месте `None`, так что список корректен в каждый момент.

**2. Рекурсия по длинному списку.** Любая функция, которая вызывает себя один раз на узел (рекурсивная `len`, рекурсивная печать),
переполнит стек на длинном списке — точно так же, как освобождение по умолчанию. Пишите циклы или используйте итератор.

## Попробуйте сами

1. Наберите `linked_list.rs` сами и запустите `cargo test linked_list`, `cargo clippy` и `cargo fmt`.
2. Удалите реализацию `Drop`, запустите тесты и прочитайте сбой. Верните её на место.
3. Запустите `examples/e04_sizes.rs`, затем в копии замените `next: Option<Box<Node>>` на `next: Box<Node>` (не скомпилируется).
   Почему?
4. Запустите бенчмарк и запишите отношение «погони за указателями» на вашем компьютере.
5. Зафиксируйте: `git add code/p07-algorithms && git commit -m "feat(p07): singly linked list"`.

## Упражнения

### Упражнение 1 — Сколько это стоит? (легко)

Для нашего односвязного списка только с указателем `head` назовите временную сложность: (a) `push_front`; (b) `push_back`;
(c) удаление *последнего* узла; (d) `len()`; (e) поиск значения 42; (f) `reverse`. Что из этого изменилось бы, если бы список
хранил ещё и указатель на последний узел?

<details>
<summary>Подсказка</summary>

Посчитайте, сколько ссылок надо пройти. Для (d) посмотрите на структуру.

</details>

<details>
<summary>Ответ</summary>

(a) `O(1)`. (b) `O(n)`: дойти до конца. (c) `O(n)`: нужно найти предпоследний узел, чтобы разорвать ссылку. (d) `O(1)`:
сохранённая `len`. (e) `O(n)`. (f) `O(n)`. С указателем на последний узел (b) стала бы `O(1)`; (c) осталась бы `O(n)`, потому
что и тогда до предпоследнего узла не добраться без обхода (двусвязный список решает и это).

</details>

### Упражнение 2 — `contains` и `pop_back` (средне)

Добавьте `contains(&self, target: &T) -> bool where T: PartialEq` (используйте итератор) и `pop_back(&mut self) -> Option<T>`
(используйте курсор `&mut Link`, остановитесь на последнем узле). Напишите тест, который сравнивает обе с `Vec`. Назовите
время обеих.

<details>
<summary>Подсказка</summary>

Для `pop_back` двигайте курсор, пока у узла, на который он указывает, есть преемник: `cursor.as_ref()?.next.is_some()`. (`?`
сразу возвращает `None`, если список пуст.) Когда цикл остановится, `cursor` указывает на ссылку последнего узла, и
`cursor.take()` его удаляет.

</details>

<details>
<summary>Ответ</summary>

Добавьте оба метода внутрь блока `impl<T> List<T>`:

```rust
    /// Is `target` in the list? O(n).
    pub fn contains(&self, target: &T) -> bool
    where
        T: PartialEq,
    {
        self.iter().any(|value| value == target)
    }

    /// Removes and returns the last value: O(n), because we must find the second-to-last node.
    pub fn pop_back(&mut self) -> Option<T> {
        let mut cursor = &mut self.head;
        // walk until `cursor` points at the link of the last node
        while cursor.as_ref()?.next.is_some() {
            cursor = &mut cursor.as_mut()?.next;
        }
        let last = cursor.take()?;
        self.len -= 1;
        Some(last.value)
    }
```

Тест (в том же стиле, что в разделе 5; он заодно проверяет `nth_from_end` из следующего упражнения):

```rust
    #[test]
    fn contains_pop_back_and_nth_from_end_match_vec() {
        let mut rng = Rng::new(5);
        for _ in 0..200 {
            let mut list: List<u64> = List::new();
            let mut model: Vec<u64> = Vec::new();
            for _ in 0..rng.below(8) {
                let x = rng.next_u64() % 5;
                list.push_back(x);
                model.push(x);
            }
            assert_eq!(list.contains(&3), model.contains(&3));
            for k in 0..10 {
                let expected = model.len().checked_sub(k + 1).map(|i| &model[i]);
                assert_eq!(list.nth_from_end(k), expected, "k = {k}");
            }
            assert_eq!(list.pop_back(), model.pop());
            assert_eq!(list.len(), model.len());
            assert_eq!(contents(&list), model);
        }
    }
```

Оба метода — `O(n)` по времени и `O(1)` по дополнительной памяти.

</details>

### Упражнение 3 — k-й с конца за один проход (сложно)

Добавьте `nth_from_end(&self, k: usize) -> Option<&T>`, которая возвращает значение на `k` мест раньше последнего (`k = 0` —
последнее значение), проходя по списку **один раз** и не вычисляя длину заранее. Назовите время и память.

<details>
<summary>Подсказка</summary>

Используйте два итератора. Продвиньте первый на `k` шагов вперёд. Затем двигайте оба вместе, пока первый не дойдёт до конца:
второй окажется на `k` мест позади него.

</details>

<details>
<summary>Ответ</summary>

Внутри блока `impl<T> List<T>`:

```rust
    /// The value `k` places before the last one (`k = 0` is the last value), in one pass: O(n).
    pub fn nth_from_end(&self, k: usize) -> Option<&T> {
        let mut lead = self.iter();
        for _ in 0..k {
            lead.next()?; // the list is too short
        }
        let mut lag = self.iter();
        let mut found = None;
        while lead.next().is_some() {
            found = lag.next();
        }
        found
    }
```

Это приём *двух указателей* (Урок 07.15). Время `O(n)`, один проход; память `O(1)`. Если в списке меньше `k + 1` элементов,
первый цикл возвращает `None`.

</details>

## Вопросы для самопроверки

1. Почему чтение элемента `i` связного списка — это `O(i)`, а не `O(1)`?

<details>
<summary>Ответ</summary>

Узлы — отдельные выделения; единственный способ найти узел `i` — пройти `i` ссылок от головы. Формулы адреса нет.

</details>

2. Что делает `Option::take` и зачем он нужен `push_front` и `pop_front`?

<details>
<summary>Ответ</summary>

Он вынимает значение из `Option` и оставляет `None`. Без него мы не смогли бы вынуть старую голову из `self.head`
(заимствованного через `&mut self`), не оставив список в некорректном состоянии.

</details>

3. Почему освобождение списка из миллиона узлов может уронить программу и как это предотвращает собственный `Drop`?

<details>
<summary>Ответ</summary>

Освобождение по умолчанию рекурсивно: каждый узел освобождает своего преемника, поэтому глубина вызовов равна длине списка,
и стек переполняется. Собственный `Drop` отсоединяет узлы по одному в цикле, так что каждый узел освобождается без
преемника, и глубина остаётся постоянной.

</details>

4. И `Vec`, и связный список суммируют элементы за `O(n)`. Почему в нашем бенчмарке список оказался в 17 раз медленнее?

<details>
<summary>Ответ</summary>

Big-O не учитывает поведение памяти. Вектор читается по порядку, поэтому процессор заранее подгружает данные в кэш.
Списку нужен один переход по указателю на элемент, и каждый переход может промахнуться мимо кэша.

</details>

## Итоги

- Связный список — это цепочка отдельно выделенных узлов; список хранит только голову. На безопасном Rust:
  `Option<Box<Node<T>>>`.
- `push_front` и `pop_front` — `O(1)`; чтение по позиции, `push_back` (без указателя на хвост) и удаление последнего узла — `O(n)`.
- `Option::take` и курсоры `&mut` — инструменты для изменения ссылок в безопасном Rust.
- Рекурсивное освобождение длинной цепочки переполняет стек; написанный вручную `Drop` с циклом это исправляет.
- Проверяйте список против `Vec` на случайных операциях и держите тест с миллионом узлов.
- На настоящем оборудовании связный список обычно намного медленнее `Vec`. Используйте `Vec` или `VecDeque`, а
  `std::collections::LinkedList` — только ради его немногих особых сильных сторон.

## Источники и что почитать

- [Learn Rust With Entirely Too Many Linked Lists](https://rust-unofficial.github.io/too-many-lists/) — бесплатная книга, в которой строятся разные виды связных списков на Rust, от дизайна этого урока до двусвязных и небезопасных версий.
- [std::collections::LinkedList](https://doc.rust-lang.org/std/collections/struct.LinkedList.html) — стандартный двусвязный список и его собственный совет о том, когда его использовать.
- [Option::take](https://doc.rust-lang.org/std/option/enum.Option.html#method.take) и [Option::as_deref](https://doc.rust-lang.org/std/option/enum.Option.html#method.as_deref) — два метода `Option`, на которых держится этот урок.
- [std::ops::Drop](https://doc.rust-lang.org/std/ops/trait.Drop.html) — как и когда выполняется `drop`.
- [std::option — Representation](https://doc.rust-lang.org/std/option/index.html#representation) — какие типы `Option<T>` не занимают дополнительного места (оптимизация ниши).

---

[🏠 Начало](../START-HERE.md) · [← 07.3 Массивы и векторы изнутри](03-arrays-and-vectors-under-the-hood.md) · [📋 Этап 07](README.md) · [07.5 Стеки и очереди →](05-stacks-and-queues.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
