<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 07.4 Связные списки](04-linked-lists.md) · [📋 Этап 07](README.md) · [07.6 Как работают хеш-таблицы →](06-hash-tables.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/07-algorithms-and-data-structures/05-stacks-and-queues.md) <!-- nav -->

# Урок 07.5 — Стеки и очереди

> **Этап 07 — Алгоритмы и структуры данных** · Урок 5 из 20 · ⏱ около 2,5 часа (2 занятия)

## Что вы узнаете

- Что такое стек (последним пришёл — первым вышел) и очередь (первым пришёл — первым вышел) и для чего нужен каждый.
- Как построить стек на `Vec` и использовать его, чтобы проверять парность скобок.
- Почему очередь на `Vec::remove(0)` медленная и как кольцевой буфер делает оба конца `O(1)`.
- Как устроен `VecDeque` и когда его выбирать.

## Перед началом

- Вы прошли [Урок 07.4 — Связные списки](04-linked-lists.md).
- Вы знаете `Vec` ([Урок 07.3](03-arrays-and-vectors-under-the-hood.md)), `match` ([Урок 04.4](../04-structs-enums-collections-errors/04-option-and-match.md)) и идею амортизированной стоимости ([Урок 07.2](02-analyzing-time-and-space.md)).

## Новые слова

| Слово | Значение |
|---|---|
| **стек** (stack) | Коллекция, где добавляют и удаляют на одном и том же конце. Как стопка тарелок: берёте ту, что положили последней. |
| **LIFO** (Last In, First Out, «последним пришёл — первым вышел») | Правило стека: элемент, добавленный последним, удаляется первым. |
| **очередь** (queue) | Коллекция, где добавляют на одном конце (*в конец*) и удаляют на другом (*из начала*). Как люди в очереди. |
| **FIFO** (First In, First Out, «первым пришёл — первым вышел») | Правило очереди: элемент, добавленный первым, удаляется первым. |
| **push / pop** | Добавить элемент / удалить элемент (для стека: на вершине; для очереди это `push_back` и `pop_front`). |
| **peek** | Посмотреть на элемент, который будет удалён следующим, не удаляя его. |
| **deque** (двусторонняя очередь, читается «дек») | Очередь, где можно добавлять и удалять на обоих концах. |
| **кольцевой буфер** (ring buffer, circular buffer) | Блок ячеек фиксированного размера, используемый как круг: после последней ячейки снова идёт первая. |
| **инкапсуляция** (encapsulation) | Скрытие того, как хранятся данные, и предоставление только операций, которые сохраняют их корректность. |

## 1. Стеки

**Стек** следует правилу **LIFO**: последним пришёл — первым вышел. Представьте стопку тарелок: вы кладёте тарелку
сверху и берёте верхнюю тарелку. Основных операций всего три, и все работают на **вершине**:

```text
push(1)   push(2)   push(3)   pop() -> 3   pop() -> 2
                     ┌───┐
           ┌───┐     │ 3 │ ◀── top
 ┌───┐     │ 2 │     │ 2 │      ┌───┐
 │ 1 │     │ 1 │     │ 1 │      │ 1 │         ┌───┐
 └───┘     └───┘     └───┘      └───┘         │ 1 │
                                              └───┘
```

`Vec` уже хороший стек: `push` и `pop` работают в конце и стоят `O(1)` (для `push` — амортизированно). Мы оборачиваем его в
небольшой тип, чтобы код говорил, что имеет в виду, и чтобы никто случайно не вставил что-то в середину. Создайте
`src/stacks.rs` и добавьте `pub mod stacks;` в `src/lib.rs`:

```rust
//! Lesson 07.5: stacks (last in, first out).

/// A stack. It is a `Vec` that only allows work at one end.
pub struct Stack<T> {
    items: Vec<T>,
}

impl<T> Stack<T> {
    pub fn new() -> Stack<T> {
        Stack { items: Vec::new() }
    }

    /// Puts a value on top: amortized O(1).
    pub fn push(&mut self, value: T) {
        self.items.push(value);
    }

    /// Takes the top value off: O(1).
    pub fn pop(&mut self) -> Option<T> {
        self.items.pop()
    }

    /// Looks at the top value: O(1).
    pub fn peek(&self) -> Option<&T> {
        self.items.last()
    }

    pub fn len(&self) -> usize {
        self.items.len()
    }

    pub fn is_empty(&self) -> bool {
        self.items.is_empty()
    }
}

impl<T> Default for Stack<T> {
    fn default() -> Stack<T> {
        Stack::new()
    }
}
```

**Что происходит:** `Stack<T>` хранит приватный `Vec<T>`. `push`, `pop` и `peek` используют конец вектора (`last()` — это
конец), поэтому каждая операция — `O(1)`. Тип не даёт добраться до других позиций: пользователь не может нарушить правило
LIFO. Это называется **инкапсуляцией**: вы скрываете, как хранятся данные, и предлагаете только операции, которые сохраняют
их корректность.

### Применение 1: парность скобок

Правильно ли `f(a[1], {b: 2})`: каждая скобка закрыта скобкой нужного вида и в нужном порядке? Идея: пройти по тексту; когда
встречаем *открывающую* скобку — кладём её в стек. Когда встречаем *закрывающую* — ближайшая незакрытая открывающая должна
быть её парой: достаём и сравниваем. В конце стек должен быть пуст.

```rust
/// Are all brackets `()`, `[]`, `{}` closed in the right order? O(n) time, O(n) space.
pub fn is_balanced(text: &str) -> bool {
    let mut open = Stack::new();
    for ch in text.chars() {
        match ch {
            '(' | '[' | '{' => open.push(ch),
            ')' | ']' | '}' => {
                let expected = match ch {
                    ')' => '(',
                    ']' => '[',
                    _ => '{',
                };
                if open.pop() != Some(expected) {
                    return false;
                }
            }
            _ => {} // other characters are ignored
        }
    }
    open.is_empty()
}
```

**Что происходит:**

1. Открывающие скобки кладутся в стек. Остальные символы игнорируются.
2. Для закрывающей скобки мы выясняем, какая открывающая ей нужна (`expected`). `open.pop()` даёт ближайшую незакрытую (или
   `None`, если её нет). Если она не равна `Some(expected)`, текст неверен: `"a)b"` (нечего закрывать), `"([)]"` (неверный
   порядок).
3. В конце `open.is_empty()`: оставшаяся скобка, как в `"(()"`, означает, что что-то так и не закрыли.

```text
text:  ( [ ] { } )
step 1  (        stack: (
step 2  [        stack: ( [
step 3  ]  pop [ stack: (
step 4  {        stack: ( {
step 5  }  pop { stack: (
step 6  )  pop ( stack: (empty)   -> balanced
```

Время `O(n)` (каждый символ обрабатывается один раз, каждая операция стека — `O(1)`); память `O(n)` в худшем случае (текст только
из открывающих скобок).

`examples/e05_brackets.rs`:

```rust
use p07_algorithms::stacks::is_balanced;

fn main() {
    for text in ["f(a[1], {b: 2})", "f(a[1}, 2)", "((", "a)b"] {
        println!("{text:>16}  balanced: {}", is_balanced(text));
    }
}
```

```text
 f(a[1], {b: 2})  balanced: true
      f(a[1}, 2)  balanced: false
              ((  balanced: false
             a)b  balanced: false
```

Добавьте тесты внизу `src/stacks.rs`:

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn stack_is_last_in_first_out() {
        let mut s = Stack::new();
        s.push(1);
        s.push(2);
        s.push(3);
        assert_eq!(s.peek(), Some(&3));
        assert_eq!(s.pop(), Some(3));
        assert_eq!(s.pop(), Some(2));
        assert_eq!(s.len(), 1);
        assert_eq!(s.pop(), Some(1));
        assert_eq!(s.pop(), None);
        assert!(s.is_empty());
    }

    #[test]
    fn brackets() {
        for good in ["", "()", "([]{})", "a(b[c]{d})e", "{[()()]}"] {
            assert!(is_balanced(good), "{good:?} should be balanced");
        }
        for bad in ["(", ")", ")(", "([)]", "(()", "{[}]", "a)b"] {
            assert!(!is_balanced(bad), "{bad:?} should not be balanced");
        }
    }
}
```

```text
running 2 tests
test stacks::tests::stack_is_last_in_first_out ... ok
test stacks::tests::brackets ... ok

test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 15 filtered out; finished in 0.00s
```

> 🔒 **Безопасность:** `is_balanced` читает текст, который может прийти снаружи. Она никогда не паникует (использует `pop()` и
> сравнивает `Option`, без `unwrap`) и обрабатывает любой вход за один проход, поэтому враждебный текст не может ни уронить её, ни
> заставить работать долго. Сохраните эту привычку: обрабатывайте `None`, тестируйте на сломанном вводе.

Где ещё встречаются стеки? **Стек вызовов** вашей программы (Урок 03.1) — это стек вызовов функций, и рекурсия (Урок 07.7) им
пользуется. «Отмена» в редакторе — это стек изменений. Компилятор использует стек, чтобы проверять вложенность скобок, точно как
здесь.

## 2. Очереди

**Очередь** следует **FIFO**: первым пришёл — первым вышел, как очередь в магазине. Элементы встают в **конец** и уходят из
**начала**.

```text
 front                    back
   ▼                       ▼
  ┌───┬───┬───┬───┬───┐
  │ 3 │ 4 │ 5 │ 6 │ 7 │   pop_front() -> 3        push_back(8) joins after 7
  └───┴───┴───┴───┴───┘
```

В очередях лежит работа, которую надо выполнять в порядке поступления: задания печати, сетевые пакеты и список «что посетить
дальше» при поиске в ширину (Урок 07.13).

Очевидная реализация — `Vec`, где `push` ставит в очередь, а `remove(0)` выводит из неё, — стоит `O(n)` на каждое извлечение:
все остальные элементы сдвигаются (Урок 07.3). Связный список с указателем на хвост подошёл бы, но на практике он медленный
(Урок 07.4). Хорошее решение — **кольцевой буфер**.

### Кольцевой буфер

Держите блок ячеек и два числа: `head` — индекс первого элемента и `len` — число элементов. Элементы занимают `len` ячеек,
начиная с `head`; когда они доходят до конца блока, они **заворачиваются** в ячейку 0. Извлечение сдвигает `head` на одну
ячейку вперёд: **ничего не сдвигается**.

```text
capacity 4, head = 2, len = 3:       slot:   0     1     2     3
                                           [ 5 ] [   ] [ 3 ] [ 4 ]
                                             ▲           ▲
                                      (wrapped: 5 is   head: the front is 3
                                       the last item)
index of the back slot = (head + len) % capacity = (2 + 3) % 4 = 1   <- the next push goes here
```

Остаток `%` заставляет индекс заворачиваться: `(2 + 3) % 4 = 1`. Когда блок заполнен, мы выделяем блок вдвое большего размера и
копируем элементы **по порядку** в его начало (`head` становится 0), как в `DynArray`. Создайте `src/queues.rs` и добавьте
`pub mod queues;` в `src/lib.rs`:

```rust
//! Lesson 07.5: a queue (first in, first out) on a ring buffer.

/// A queue stored in a ring: `head` is the index of the front item, and the
/// items continue after it, wrapping around to index 0 at the end of `slots`.
pub struct RingQueue<T> {
    slots: Vec<Option<T>>,
    head: usize,
    len: usize,
}

impl<T> RingQueue<T> {
    pub fn new() -> RingQueue<T> {
        RingQueue {
            slots: Vec::new(),
            head: 0,
            len: 0,
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

    /// Adds a value at the back: amortized O(1).
    pub fn push_back(&mut self, value: T) {
        if self.len == self.slots.len() {
            self.grow();
        }
        let tail = (self.head + self.len) % self.slots.len();
        self.slots[tail] = Some(value);
        self.len += 1;
    }

    /// Removes the value at the front: O(1). Nothing is shifted.
    pub fn pop_front(&mut self) -> Option<T> {
        if self.len == 0 {
            return None;
        }
        let value = self.slots[self.head].take();
        self.head = (self.head + 1) % self.slots.len();
        self.len -= 1;
        value
    }

    /// Looks at the front value: O(1).
    pub fn front(&self) -> Option<&T> {
        if self.len == 0 {
            None
        } else {
            self.slots[self.head].as_ref()
        }
    }
```

```rust
    /// Doubles the capacity and lays the items out again from index 0: O(n).
    fn grow(&mut self) {
        let new_capacity = if self.slots.is_empty() {
            4
        } else {
            self.slots.len() * 2
        };
        let mut bigger: Vec<Option<T>> = Vec::new();
        bigger.resize_with(new_capacity, || None);
        for (i, slot) in bigger.iter_mut().enumerate().take(self.len) {
            let from = (self.head + i) % self.slots.len();
            *slot = self.slots[from].take();
        }
        self.slots = bigger;
        self.head = 0;
    }
}

impl<T> Default for RingQueue<T> {
    fn default() -> RingQueue<T> {
        RingQueue::new()
    }
}
```

**Что происходит:**

1. `push_back` увеличивает кольцо, если оно полно, вычисляет ячейку после последнего элемента как `(head + len) % capacity` и
   записывает туда значение.
2. `pop_front` вынимает значение из ячейки `head`, сдвигает `head` вперёд на единицу (заворачивая через `%`) и уменьшает `len`.
   Это `O(1)`, и никакой другой элемент не перемещается.
3. `grow` удваивает ёмкость и копирует элементы в порядке очереди в новый блок. Благодаря удвоению `push_back` — амортизированная
   `O(1)`, тот же довод, что в Уроке 07.3. (Метод `take(self.len)` у итератора ограничивает цикл `len` настоящими элементами.)

`examples/e05_queue.rs` повторяет картинку выше (4 добавления, 2 извлечения, 2 добавления с заворотом, 1 добавление с ростом):

```rust
use p07_algorithms::queues::RingQueue;

fn main() {
    let mut q = RingQueue::new();
    for i in 1..=4 {
        q.push_back(i);
    }
    println!("after 4 pushes: len {}, capacity {}", q.len(), q.capacity());

    println!("pop {:?}, pop {:?}", q.pop_front(), q.pop_front());
    q.push_back(5);
    q.push_back(6);
    println!(
        "after wrap-around: len {}, capacity {}",
        q.len(),
        q.capacity()
    );

    q.push_back(7);
    println!(
        "after one more push: len {}, capacity {}",
        q.len(),
        q.capacity()
    );

    let mut order = Vec::new();
    while let Some(x) = q.pop_front() {
        order.push(x);
    }
    println!("order out: {order:?}");
}
```

```text
after 4 pushes: len 4, capacity 4
pop Some(1), pop Some(2)
after wrap-around: len 4, capacity 4
after one more push: len 5, capacity 8
order out: [3, 4, 5, 6, 7]
```

Два дополнительных добавления использовали освободившиеся ячейки, поэтому ёмкость осталась 4. Только следующее добавление, которое
нашло кольцо полным, увеличило его до 8. Элементы по-прежнему выходили в том порядке, в каком вошли.

Тест сравнивает очередь со стандартным `VecDeque` на 10 000 случайных добавлений и извлечений:

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use crate::testutil::Rng;
    use std::collections::VecDeque;

    #[test]
    fn behaves_like_vec_deque_on_random_operations() {
        let mut rng = Rng::new(31);
        let mut mine: RingQueue<u64> = RingQueue::new();
        let mut reference: VecDeque<u64> = VecDeque::new();
        for step in 0..10_000 {
            if rng.below(5) < 3 {
                let x = rng.next_u64() % 1000;
                mine.push_back(x);
                reference.push_back(x);
            } else {
                assert_eq!(mine.pop_front(), reference.pop_front(), "step {step}");
            }
            assert_eq!(mine.len(), reference.len(), "step {step}");
            assert_eq!(mine.front(), reference.front(), "step {step}");
        }
    }
}
```

```text
running 1 test
test queues::tests::behaves_like_vec_deque_on_random_operations ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 17 filtered out; finished in 0.00s
```

## 3. Измеряем: почему не `Vec::remove(0)`

`benches/queues.rs` заполняет очередь 20 000 числами и полностью опустошает её — один раз через `Vec::remove(0)`, другой раз через
стандартный `VecDeque`. Добавьте в `Cargo.toml` (как в Уроке 07.2):

```toml
[[bench]]
name = "queues"
harness = false
```

```rust
use criterion::{Criterion, criterion_group, criterion_main};
use std::collections::VecDeque;
use std::hint::black_box;

const N: u64 = 20_000;

fn bench_drain(c: &mut Criterion) {
    let mut group = c.benchmark_group("fill and empty a queue of 20_000");
    group.bench_function("Vec, remove(0)", |b| {
        b.iter(|| {
            let mut q: Vec<u64> = (0..black_box(N)).collect();
            let mut sum = 0;
            while !q.is_empty() {
                sum += q.remove(0);
            }
            sum
        })
    });
    group.bench_function("VecDeque", |b| {
        b.iter(|| {
            let mut q: VecDeque<u64> = (0..black_box(N)).collect();
            let mut sum = 0;
            while let Some(x) = q.pop_front() {
                sum += x;
            }
            sum
        })
    });
    group.finish();
}

criterion_group!(benches, bench_drain);
criterion_main!(benches);
```

Запустите `cargo bench --bench queues -- --warm-up-time 1 --measurement-time 3`:

```text
fill and empty a queue of 20_000/Vec, remove(0)
                        time:   [17.874 ms 17.927 ms 17.982 ms]
fill and empty a queue of 20_000/VecDeque
                        time:   [23.609 µs 24.009 µs 24.578 µs]
```

(Apple M4 Pro; **ваши числа будут другими**.)

**Как это читать:** `Vec::remove(0)` занимает 17,9 мс: это `O(n)` на извлечение и `O(n²)` в сумме. `VecDeque` занимает 24 мкс —
**примерно в 750 раз быстрее** для той же работы, потому что это кольцевой буфер, как наш: извлечение из начала двигает индекс
`head` и ничего не сдвигает.

## 4. Когда вместо этого взять стандартную библиотеку

| Что нужно | Что использовать | Примечания |
|---|---|---|
| стек | `Vec<T>`: `push`, `pop`, `last` | в повседневном коде обёртка не нужна |
| очередь или оба конца | `std::collections::VecDeque<T>`: `push_back`, `pop_front`, `push_front`, `pop_back` | внутри кольцевой буфер, как у нас |
| «следующий наименьший или наибольший элемент» | `std::collections::BinaryHeap<T>` | очередь с приоритетом (Урок 07.12) |
| потоки, которые передают друг другу работу | каналы (Урок 05.10) | очередь между потоками |

`VecDeque` также поддерживает индексирование (`q[3]`) за `O(1)` и обход от начала к концу.

## Частые ошибки

**1. `Vec::remove(0)` в роли очереди.** Это работает и проходит маленькие тесты, а потом становится медленным, когда в очереди
тысячи элементов (см. бенчмарк). Используйте `VecDeque`.

**2. Извлечение из пустого стека и вызов `unwrap`.** `stack.pop().unwrap()` паникует, когда стек пуст. В `is_balanced`
закрывающая скобка `)` в `"a)b"` встречает пустой стек, и это нормальный ответ (`false`), а не падение, поэтому мы сравниваем
`Option`, а не разворачиваем его.

**3. Забытый заворот.** Запись `self.head + 1` вместо `(self.head + 1) % self.slots.len()` работает, пока голова не дойдёт до
конца блока, а потом паникует с выходом индекса за границы. Случайный тест быстро это ловит, поэтому такой тест стоит писать.

## Попробуйте сами

1. Наберите `stacks.rs` и `queues.rs` сами. Запустите `cargo test stacks`, `cargo test queues`, `cargo clippy` и `cargo fmt`.
2. Добавьте в тест свои случаи со скобками, например `"<>"` (пока не поддерживается: чего вы ждёте и что происходит?).
3. Научите `is_balanced` паре `<` `>` и добавьте для неё тесты.
4. Запустите бенчмарк и сравните два числа с приведёнными выше.
5. Зафиксируйте: `git add code/p07-algorithms && git commit -m "feat(p07): stacks and ring-buffer queue"`.

## Упражнения

### Упражнение 1 — Проследите стек (легко)

Покажите стек после каждого символа, пока работает `is_balanced("{[(a)]")`, и назовите результат. Затем скажите, какая это ошибка
(неверный порядок, пропущенная закрывающая скобка, лишняя закрывающая скобка) для каждой из строк `"([)]"`, `"(()"` и `"a)b"`.

<details>
<summary>Подсказка</summary>

Открывающая скобка кладётся в стек; закрывающая достаёт и сравнивает. Записывайте стек снизу вверх.

</details>

<details>
<summary>Ответ</summary>

```text
{   -> [ { ]
[   -> [ {, [ ]
(   -> [ {, [, ( ]
a   -> (ignored)
)   -> pop ( matches  -> [ {, [ ]
]   -> pop [ matches  -> [ { ]
end -> the stack still holds { -> false
```

Текст не сбалансирован: `{` так и не закрыли. `"([)]"` — неверный порядок (`)` встречает на вершине `[`), в `"(()"` пропущена
закрывающая скобка (в конце стек не пуст), а в `"a)b"` лишняя закрывающая скобка (когда приходит `)`, стек пуст).

</details>

### Упражнение 2 — Стек, который знает свой минимум (средне)

Напишите `MinStack<T: Ord + Copy>` с `push`, `pop` и `min() -> Option<T>`, где **все три операции — `O(1)`** (а не перебор
элементов). Напишите тест, который сравнивает его с перебором по `Vec` на 2 000 случайных операций.

<details>
<summary>Подсказка</summary>

Храните рядом с каждым значением наименьшее значение в стеке до этой записи включительно. Минимум всего стека — второе число
верхней записи.

</details>

<details>
<summary>Ответ</summary>

```rust
pub struct MinStack<T> {
    // each entry: the value, and the smallest value in the stack up to and including it
    items: Vec<(T, T)>,
}

impl<T: Ord + Copy> MinStack<T> {
    pub fn new() -> MinStack<T> {
        MinStack { items: Vec::new() }
    }

    pub fn push(&mut self, value: T) {
        let smallest = match self.items.last() {
            Some(&(_, m)) if m < value => m,
            _ => value,
        };
        self.items.push((value, smallest));
    }

    pub fn pop(&mut self) -> Option<T> {
        self.items.pop().map(|(value, _)| value)
    }

    /// The smallest value in the stack: O(1).
    pub fn min(&self) -> Option<T> {
        self.items.last().map(|&(_, m)| m)
    }
}
```

```rust
    #[test]
    fn min_stack_matches_a_scan() {
        let mut rng = Rng::new(3);
        let mut s: MinStack<u64> = MinStack::new();
        let mut model: Vec<u64> = Vec::new();
        for _ in 0..2000 {
            if model.is_empty() || rng.below(3) > 0 {
                let x = rng.next_u64() % 50;
                s.push(x);
                model.push(x);
            } else {
                assert_eq!(s.pop(), model.pop());
            }
            assert_eq!(s.min(), model.iter().copied().min());
        }
    }
```

Почему это работает: когда запись кладётся в стек, её второе число — `min(value, второе число записи под ней)`. Когда запись
извлекается, запись под ней всё ещё хранит верный минимум для меньшего стека. Стоимость: `O(1)` времени для каждой операции и
`O(n)` дополнительной памяти (по одному лишнему числу на элемент).

</details>

### Упражнение 3 — Очередь из двух стеков (сложно)

Постройте `QueueFromStacks<T>` с `push_back` и `pop_front`, используя только два `Vec` как стеки: `inbox`, куда кладутся новые
элементы, и `outbox`, откуда они извлекаются. Когда `outbox` пуст, переложите все элементы из `inbox` в `outbox` (это меняет их
порядок на обратный). Проверьте тестом против `VecDeque`. Почему `pop_front` амортизированно `O(1)`, хотя один вызов может
стоить `O(n)`?

<details>
<summary>Подсказка</summary>

Если достать все элементы стека и положить их в другой стек, порядок станет обратным. Обращённый порядок LIFO — это порядок FIFO.

</details>

<details>
<summary>Ответ</summary>

```rust
pub struct QueueFromStacks<T> {
    inbox: Vec<T>,  // new items are pushed here
    outbox: Vec<T>, // items leave from here, oldest on top
}

impl<T> QueueFromStacks<T> {
    pub fn new() -> QueueFromStacks<T> {
        QueueFromStacks {
            inbox: Vec::new(),
            outbox: Vec::new(),
        }
    }

    pub fn push_back(&mut self, value: T) {
        self.inbox.push(value);
    }

    pub fn pop_front(&mut self) -> Option<T> {
        if self.outbox.is_empty() {
            while let Some(x) = self.inbox.pop() {
                self.outbox.push(x); // reverses the order
            }
        }
        self.outbox.pop()
    }

    pub fn len(&self) -> usize {
        self.inbox.len() + self.outbox.len()
    }

    pub fn is_empty(&self) -> bool {
        self.len() == 0
    }
}
```

```rust
    #[test]
    fn queue_from_stacks_matches_vec_deque() {
        let mut rng = Rng::new(8);
        let mut q: QueueFromStacks<u64> = QueueFromStacks::new();
        let mut model: VecDeque<u64> = VecDeque::new();
        for _ in 0..5000 {
            if rng.below(5) < 3 {
                let x = rng.next_u64() % 100;
                q.push_back(x);
                model.push_back(x);
            } else {
                assert_eq!(q.pop_front(), model.pop_front());
            }
            assert_eq!(q.len(), model.len());
        }
        assert_eq!(q.is_empty(), model.is_empty());
    }
```

Каждый элемент перекладывается из `inbox` в `outbox` **не более одного раза** за свою жизнь, поэтому за `n` операций
перекладывания стоят в сумме не более `n` перемещений. Один вызов `pop_front`, нашедший `outbox` пустым, может стоить `O(n)`, но в
среднем — `O(1)`: **амортизированно** (Урок 07.2).

</details>

## Вопросы для самопроверки

1. Что значат LIFO и FIFO и какая структура какому правилу следует?

<details>
<summary>Ответ</summary>

LIFO (последним пришёл — первым вышел) — это стек; FIFO (первым пришёл — первым вышел) — это очередь.

</details>

2. Почему `Vec::remove(0)` — плохой способ брать следующий элемент очереди?

<details>
<summary>Ответ</summary>

Он сдвигает все остальные элементы на одно место влево, `O(n)` на извлечение, поэтому опустошение очереди из `n` элементов стоит
`O(n²)`. `VecDeque::pop_front` — это `O(1)`.

</details>

3. В кольцевом буфере с ёмкостью 8 `head = 6` и `len = 3`. В какую ячейку пишет следующий `push_back` и какие ячейки заняты?

<details>
<summary>Ответ</summary>

`(6 + 3) % 8 = 1`, то есть ячейка 1. Элементы лежат в ячейках 6, 7 и 0.

</details>

4. Какая структура нужна команде «отмена» в редакторе — стек или очередь?

<details>
<summary>Ответ</summary>

Стек: сначала отменяется самое последнее изменение (LIFO).

</details>

## Итоги

- Стек — это LIFO: push и pop на одном конце. `Vec` — стек с операциями `O(1)`. Используйте его для проверки скобок, отмены и (позже)
  рекурсии.
- Очередь — это FIFO: встать в конец, уйти из начала. `Vec::remove(0)` — `O(n)`; кольцевой буфер делает оба конца `O(1)` с помощью
  индекса `head` и заворота через `%`.
- `std::collections::VecDeque` — стандартный кольцевой буфер. Используйте `Vec` как стек и `VecDeque` как очередь.
- Очередь можно построить из двух стеков; стоимость `O(1)` амортизированно, потому что каждый элемент перемещается только один раз.
- Обрабатывайте `None`, а не разворачивайте его, когда читаете внешний ввод; тестируйте на сломанном вводе.
- Сравнивайте свою структуру со стандартной на случайных операциях.

## Источники и что почитать

- [std::collections::VecDeque](https://doc.rust-lang.org/std/collections/struct.VecDeque.html) — стандартная двусторонняя очередь, растущий кольцевой буфер.
- [std::collections — When Should You Use Which Collection?](https://doc.rust-lang.org/std/collections/index.html#when-should-you-use-which-collection) — официальное руководство по выбору коллекции.
- [Vec::push](https://doc.rust-lang.org/std/vec/struct.Vec.html#method.push) и [Vec::pop](https://doc.rust-lang.org/std/vec/struct.Vec.html#method.pop) — операции стека у `Vec`.

---

[🏠 Начало](../START-HERE.md) · [← 07.4 Связные списки](04-linked-lists.md) · [📋 Этап 07](README.md) · [07.6 Как работают хеш-таблицы →](06-hash-tables.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
