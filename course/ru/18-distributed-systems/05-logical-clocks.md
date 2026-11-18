<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 18.4 Время и часы](04-time-and-clocks.md) · [📋 Этап 18](README.md) · [18.6 Репликация →](06-replication.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/18-distributed-systems/05-logical-clocks.md) <!-- nav -->

# Урок 18.5 — Логические часы: часы Лэмпорта и векторные часы

> **Этап 18 — Распределённые системы** · Урок 5 из 15 · ⏱ около 135 минут

## Что вы узнаете

- Что значит «произошло раньше», когда общих часов нет, и что значит, что два события *конкурентны*.
- Как часы Лэмпорта дают каждому событию номер, который уважает причину и следствие, двумя строками кода.
- Чего часы Лэмпорта сказать не могут и как это исправляют векторные часы.
- Как сравнивать двое векторных часов: раньше, позже, равны или конкурентны.
- Как реплицируемые базы данных с помощью векторных часов обнаруживают конфликтующие записи, вместо того чтобы терять одну из них.

## Перед началом

- Вы прошли [Урок 18.4 — Время и часы](04-time-and-clocks.md). Его вывод — отправная точка здесь: метки настенных часов разных машин не могут упорядочить события.
- Вы знаете `BTreeMap` с Этапа 04, реализацию `Display` с Этапа 05 и `proptest` из [Урока 09.4](../09-testing-debugging-benchmarking/04-property-based-testing.md).
- Создайте пакет для этого урока:

```bash
cd ~/projects/rust
cargo new --lib code/p18-logical-clocks
cd code/p18-logical-clocks
cargo add --dev proptest@1
mkdir src/bin tests
```

## Новые слова

| Слово | Значение |
|---|---|
| **событие** (event) | Всё, что происходит на одном узле: локальный шаг, отправка сообщения или его получение. |
| **отношение «произошло раньше»** (happens-before, →) | «a → b» значит, что событие a могло повлиять на событие b: либо a было раньше на том же узле, либо сообщение донесло влияние a до b, возможно, через другие события. |
| **конкурентные** события (concurrent) | Два события, ни одно из которых не произошло раньше другого. Информация не могла пройти от одного к другому. О времени по настенным часам это ничего не говорит. |
| **причинность** (causality) | Связь причины и следствия между событиями; в распределённых системах — то же, что «произошло раньше». |
| **логические часы** (logical clock) | Счётчик, который упорядочивает события по отношению «произошло раньше», а не по физическому времени. |
| **часы Лэмпорта** (Lamport clock) | Логические часы с одним числом на узел, придуманные Лесли Лэмпортом (Leslie Lamport) в 1978 году. |
| **векторные часы** (vector clock) | Логические часы, в каждой метке которых есть по счётчику **на каждый узел**. Они отличают конкурентные события от упорядоченных. |
| **полный порядок** (total order) | Порядок, в котором любые два разных элемента сравнимы: один всегда первый. |
| **частичный порядок** (partial order) | Порядок, в котором некоторые пары несравнимы. Конкурентные события — такие пары. |
| **сиблинг** (sibling) | Одна из нескольких конкурентных версий одного значения, которые база данных хранит, потому что не может знать, какая новее. |

## 1. Порядок без часов

Урок 18.4 показал, что настенные часы не могут решить, какое из двух событий на разных машинах было первым. Но
часто настоящее время нам вообще не нужно. Нужно нечто более слабое и гораздо более полезное: **могло ли событие a
повлиять на событие b?** Если узел b прочитал значение, которое записал узел a, следующая запись b должна считаться
«после» записи a, что бы ни говорили часы.

Это отношение **«произошло раньше»**, записывается a → b. Его определяют три правила:

1. Если a и b происходят на **одном узле** и a идёт первым, то a → b.
2. Если a — **отправка** сообщения, а b — **получение** этого сообщения, то a → b.
3. Если a → b и b → c, то a → c (отношение *транзитивно*).

Если ни a → b, ни b → a, события **конкурентны**: информация между ними пройти не могла. Вот пример, который мы
используем весь урок. Время течёт вправо; стрелки между линиями — сообщения:

```text
node a:  a1 ──── a2 ─────────────────────────────── a3
                  \ m1
node b:      b1 ── b2 ── b3
                          \ m2
node c:         c1 ─────── c2
```

- a1 → a2 → b2 → b3 → c2: цепочка шагов и сообщений. Значит, a1 → c2: то, что узел a знал в a1, могло дойти до c2.
- a3 и c2 конкурентны: ни одно сообщение не шло от a3 к узлу c или от c2 к узлу a, поэтому ни одно не могло
  повлиять на другое — даже если a3 случилось «позже» по настенным часам.
- b1 и a1 тоже конкурентны.

## 2. Часы Лэмпорта

В 1978 году Лесли Лэмпорт предложил очень маленькое правило, которое даёт каждому событию номер так, что
**если a → b, то L(a) < L(b)**:

- Каждый узел хранит счётчик, начиная с 0.
- Перед каждым локальным событием или отправкой узел прибавляет 1. Сообщение несёт счётчик отправителя.
- Получив сообщение со счётчиком `remote`, узел ставит свой счётчик в `max(own, remote) + 1`.

Поместите это в `src/lib.rs`:

```rust
//! Logical clocks (Lesson 18.5).

/// A Lamport clock: one counter that every node keeps for itself.
#[derive(Debug, Default)]
pub struct LamportClock {
    time: u64,
}

impl LamportClock {
    /// A local event or sending a message: count one up.
    pub fn tick(&mut self) -> u64 {
        self.time += 1;
        self.time
    }

    /// Receiving a message that carries the sender's time `remote`: jump past it.
    pub fn receive(&mut self, remote: u64) -> u64 {
        self.time = self.time.max(remote) + 1;
        self.time
    }
}
```

Теперь воспроизведём пример с диаграммы, `src/bin/lamport.rs`:

```rust
use p18_logical_clocks::LamportClock;

fn main() {
    let mut a = LamportClock::default();
    let mut b = LamportClock::default();
    let mut c = LamportClock::default();

    println!("a: local event           L={}", a.tick());
    let m1 = a.tick();
    println!("a: sends m1 to b         L={m1}");
    println!("b: local event           L={}", b.tick());
    println!("b: receives m1           L={}", b.receive(m1));
    let m2 = b.tick();
    println!("b: sends m2 to c         L={m2}");
    println!("c: local event           L={}", c.tick());
    println!("c: receives m2           L={}", c.receive(m2));
    println!("a: local event           L={}", a.tick());
}
```

`cargo run --bin lamport`:

```text
a: local event           L=1
a: sends m1 to b         L=2
b: local event           L=1
b: receives m1           L=3
b: sends m2 to c         L=4
c: local event           L=1
c: receives m2           L=5
a: local event           L=3
```

**Что происходит:**

1. Собственный счётчик узла b был 1, когда пришло m1 со значением 2. `max(1, 2) + 1 = 3`: b перепрыгивает **через**
   всё, что знал a в момент отправки. Узел c делает то же с m2: `max(1, 4) + 1 = 5`.
2. Пройдите по цепочке a1 → a2 → b2 → b3 → c2: номера 1, 2, 3, 4, 5. Они растут вдоль каждой цепочки причин и
   следствий. Это и есть гарантия.
3. Теперь посмотрите на a3 (L=3) и c2 (L=5). 3 < 5, но в разделе 1 мы видели, что эти события **конкурентны**.
   Правило работает только в одну сторону: из a → b следует L(a) < L(b), но из L(a) < L(b) **не** следует a → b.

Часы Лэмпорта всё равно очень полезны. Пары `(L, id узла)` дают **полный порядок** всех событий, который каждый узел
вычисляет одинаково и который никогда не противоречит причинности: сравнивайте сначала L, а id узла используйте для
разрешения ничьих. Многим алгоритмам нужно именно это, например чтобы честно решить, какой из двух запросов
блокировки пришёл «первым». Номера термов Raft (Урок 18.10) работают похоже на время Лэмпорта.

## 3. Векторные часы

Чтобы отличать конкурентные события от упорядоченных, каждая метка должна помнить, **что событие знало о каждом
узле**. **Векторные часы** — это отображение `узел → счётчик`:

- Перед каждым локальным событием или отправкой узел n прибавляет 1 к **своей** записи.
- Сообщение несёт весь вектор отправителя.
- При получении узел берёт максимум каждой записи (узнаёт всё, что знал отправитель), затем прибавляет 1 к своей
  записи.

Двое векторных часов V и W сравниваются так:

| Условие | Результат |
|---|---|
| каждая запись V ≤ записи W, и хотя бы одна < | V **раньше** W |
| каждая запись V ≥ записи W, и хотя бы одна > | V **позже** W |
| все записи равны | **равны** |
| одни записи меньше, другие больше | **конкурентны** |

Отсутствующая запись считается равной 0. Добавьте `use std::collections::BTreeMap;` и `use std::fmt;` в начало
`src/lib.rs`, затем:

```rust
/// How two events are related.
#[derive(Debug, PartialEq)]
pub enum Order {
    Before,
    After,
    Equal,
    Concurrent,
}

/// A vector clock: one counter for every node, by node name.
#[derive(Debug, Clone, Default, PartialEq)]
pub struct VectorClock {
    counts: BTreeMap<String, u64>,
}

impl VectorClock {
    /// A local event on `node` (also used before sending a message).
    pub fn tick(&mut self, node: &str) {
        *self.counts.entry(node.to_string()).or_insert(0) += 1;
    }

    /// Takes in everything the other clock knows: the maximum of each counter.
    pub fn merge(&mut self, other: &VectorClock) {
        for (node, &count) in &other.counts {
            let mine = self.counts.entry(node.clone()).or_insert(0);
            *mine = (*mine).max(count);
        }
    }

    /// Receiving a message on `node` that carries the sender's clock.
    pub fn receive(&mut self, node: &str, message: &VectorClock) {
        self.merge(message);
        self.tick(node);
    }

    fn get(&self, node: &str) -> u64 {
        self.counts.get(node).copied().unwrap_or(0)
    }

    pub fn compare(&self, other: &VectorClock) -> Order {
        let mut smaller = false; // some counter of self is smaller than in other
        let mut larger = false; // some counter of self is larger than in other
        for node in self.counts.keys().chain(other.counts.keys()) {
            let (a, b) = (self.get(node), other.get(node));
            smaller |= a < b;
            larger |= a > b;
        }
        match (smaller, larger) {
            (false, false) => Order::Equal,
            (true, false) => Order::Before,
            (false, true) => Order::After,
            (true, true) => Order::Concurrent,
        }
    }
}

impl fmt::Display for VectorClock {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        let parts: Vec<String> = self
            .counts
            .iter()
            .map(|(n, c)| format!("{n}:{c}"))
            .collect();
        write!(f, "[{}]", parts.join(" "))
    }
}
```

**Что происходит:**

1. `tick` прибавляет 1 к записи своего узла, при необходимости сначала создавая её с 0 (`entry(...).or_insert(0)`).
2. `merge` берёт максимум каждой записи; `receive` — это merge плюс tick, ровно по правилу выше.
3. `compare` обходит узлы обоих часов и запоминает, была ли **хоть одна** запись меньше и была ли **хоть одна**
   больше. `match` превращает два флага в четыре результата из таблицы. (Узел, который есть в обоих отображениях,
   проверяется дважды; это не вредит.)
4. `Display` печатает часы как `[a:2 b:3]`. `BTreeMap` хранит ключи отсортированными, поэтому вывод стабилен.

Снова наш пример, `src/bin/vector.rs`:

```rust
use p18_logical_clocks::VectorClock;

fn main() {
    let mut a = VectorClock::default();
    let mut b = VectorClock::default();
    let mut c = VectorClock::default();

    a.tick("a");
    let a1 = a.clone();
    println!("a1 a: local event    {a}");
    a.tick("a");
    let m1 = a.clone();
    println!("a2 a: sends m1       {a}");
    b.tick("b");
    println!("b1 b: local event    {b}");
    b.receive("b", &m1);
    println!("b2 b: receives m1    {b}");
    b.tick("b");
    let m2 = b.clone();
    println!("b3 b: sends m2       {b}");
    c.tick("c");
    let c1 = c.clone();
    println!("c1 c: local event    {c}");
    c.receive("c", &m2);
    let c2 = c.clone();
    println!("c2 c: receives m2    {c}");
    a.tick("a");
    let a3 = a.clone();
    println!("a3 a: local event    {a}");

    println!();
    println!("a1 vs c2: {:?}", a1.compare(&c2));
    println!("c2 vs a1: {:?}", c2.compare(&a1));
    println!("a3 vs c1: {:?}", a3.compare(&c1));
    println!("a3 vs c2: {:?}", a3.compare(&c2));
}
```

`cargo run --bin vector`:

```text
a1 a: local event    [a:1]
a2 a: sends m1       [a:2]
b1 b: local event    [b:1]
b2 b: receives m1    [a:2 b:2]
b3 b: sends m2       [a:2 b:3]
c1 c: local event    [c:1]
c2 c: receives m2    [a:2 b:3 c:2]
a3 a: local event    [a:3]

a1 vs c2: Before
c2 vs a1: After
a3 vs c1: Concurrent
a3 vs c2: Concurrent
```

Читайте вектор c2, `[a:2 b:3 c:2]`, так: «c2 видело 2 события узла a, 3 — узла b и 2 своих». a3 — это `[a:3]`: оно
впереди c2 по записи a (3 > 2), но позади по записям b и c. Ни одно не знает всего, что знает другое, поэтому они
конкурентны — тот самый случай, который часы Лэмпорта увидеть не могли.

> 🧪 **Тестирование:** правила векторных часов идеально подходят для тестов на основе свойств: «слияние в любом
> порядке даёт одни и те же часы», «часы никогда не позже слияния самих себя с другими». Упражнение 2 их пишет. А
> пока добавьте один тест-пример в конец `src/lib.rs`:

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn a_message_orders_the_events() {
        let mut a = VectorClock::default();
        let mut b = VectorClock::default();
        a.tick("a"); // a sends a message
        let message = a.clone();
        b.tick("b"); // b does something before the message arrives
        let b_before = b.clone();
        b.receive("b", &message);
        assert_eq!(message.compare(&b), Order::Before);
        assert_eq!(b.compare(&message), Order::After);
        assert_eq!(message.compare(&b_before), Order::Concurrent);
    }
```

`cargo test --lib`:

```text
test tests::a_message_orders_the_events ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

## 4. Для чего используют векторные часы

**Обнаружение конфликтов.** Некоторые реплицируемые базы данных позволяют клиентам писать в любую реплику (Урок 18.6
называет это репликацией *без лидера*). Два клиента могут одновременно изменить одну корзину покупок на двух
репликах. Если бы база использовала «побеждает последняя запись» (Урок 18.4), одно изменение незаметно исчезло бы. С
векторными часами на каждой версии база видит, что две версии **конкурентны**, хранит **обе** как *сиблинги* и даёт
приложению их объединить (для корзины — объединение товаров). Если одна версия *позже* другой, старую можно спокойно
выбросить. Упражнение 3 строит это решение.

**Причинная доставка.** Система чатов может придержать ответ, пока не показано сообщение, на которое он отвечает,
сравнивая векторные часы.

**Цена.** У векторных часов по записи на каждый узел, который когда-либо писал, поэтому они растут вместе с размером
системы и путешествуют с каждым сообщением и каждой хранимой версией. Настоящие системы их ограничивают, например
считая по репликам (а не по клиентам) и удаляя старые записи. Другой популярный компромисс — **гибридные логические
часы** (hybrid logical clock): время по настенным часам плюс маленький логический счётчик. Они остаются близки к
реальному времени, но никогда не идут назад и уважают причинность, как часы Лэмпорта. Их используют несколько
современных баз данных.

| Часы | Размер метки | Если a → b | Видят конкурентность |
|---|---|---|---|
| Настенные часы | одно число | **не** гарантировано: рассинхронизация может перевернуть порядок | нет |
| Часы Лэмпорта | одно число | L(a) < L(b) | нет |
| Векторные часы | по числу на узел | V(a) раньше V(b) | **да** |

> 💡 **Совет:** в стандартной библиотеке Rust есть трейт, созданный для частичных порядков: `PartialOrd`, у которого
> `partial_cmp` возвращает `Option<Ordering>`. Векторные часы могли бы реализовать его и возвращать `None` для
> конкурентных часов. Мы используем явный `compare` с перечислением `Order`, потому что «конкурентны» здесь — важный
> ответ, а не отсутствие ответа.

## Частые ошибки

**Сравнение векторных часов через `<`.** Поскольку `VectorClock` не реализует `PartialOrd`, `a1 < c2` не
компилируется:

```text
error[E0369]: binary operation `<` cannot be applied to type `VectorClock`
  --> src/bin/vector.rs:32:37
   |
32 |     println!("a1 before c2: {}", a1 < c2);
   |                                  -- ^ -- VectorClock
   |                                  |
   |                                  VectorClock
   |
note: `VectorClock` does not implement `PartialOrd`
```

И это хорошо: простое «меньше» спрятало бы четвёртый ответ. Используйте `a1.compare(&c2)` и обрабатывайте все четыре
варианта `Order` в `match`.

> ⚠️ **Частая ошибка: читать порядок Лэмпорта как причину и следствие.** L(a) < L(b) не значит, что a вызвало b или
> хотя бы было раньше. Это значит только, что b не произошло раньше a. Чтобы знать, конкурентны ли два события, нужны
> векторные часы.

> ⚠️ **Частая ошибка: забыть tick при получении.** Если `receive` только сливает часы, событие получения получает тот
> же вектор, что и сообщение, и два события выглядят равными. Каждое событие, включая получение, прибавляет 1 к
> собственной записи узла.

## Попробуйте сами

1. В `vector.rs` добавьте сравнения `b1 vs a1` и `b1 vs c2` (сохраните `b1` через `let b1 = b.clone();`). Предскажите
   результаты по диаграмме, прежде чем запускать.
2. Пусть узел c после c2 отправит сообщение m3 обратно узлу a, а a получит его как новое событие a4. Напечатайте
   вектор a4: теперь он больше всех остальных событий, а `a3 vs a4` — это `Before`.
3. Сделайте то же с часами Лэмпорта в `lamport.rs`. Чему равно L(a4)?

## Упражнения

### Упражнение 1 — Прыжки Лэмпорта (легко)

Напишите интеграционный тест `tests/lamport.rs`: новые часы делают tick до 1; получение сообщения с меткой 10 даёт 11;
следующий tick даёт 12; получение **старого** сообщения с меткой 3 даёт 13, а не 4.

<details>
<summary>Подсказка</summary>

Каждый вызов возвращает новое время, поэтому каждый шаг — один `assert_eq!`.

</details>

<details>
<summary>Ответ</summary>

```rust
use p18_logical_clocks::LamportClock;

#[test]
fn receiving_jumps_past_the_sender() {
    let mut b = LamportClock::default();
    assert_eq!(b.tick(), 1);
    assert_eq!(b.receive(10), 11); // the sender was far ahead
    assert_eq!(b.tick(), 12);
    assert_eq!(b.receive(3), 13); // an old message never moves the clock back
}
```

`cargo test --test lamport`:

```text
test receiving_jumps_past_the_sender ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

Почему это работает: `max(own, remote) + 1` никогда не двигает часы назад, поэтому старое или задержанное сообщение не
может заставить узел «забыть» то, что он уже знал.

</details>

### Упражнение 2 — Свойства векторных часов (средне)

Напишите тесты на основе свойств в `tests/vector_props.rs`. Создавайте часы, делая tick узлов "a", "b" и "c" в
случайном порядке (список до 10 индексов из `0..3`). Свойство 1: слияние x в y даёт те же часы, что слияние y в x, и
x, и y относительно слитых часов — `Before` или `Equal`. Свойство 2: если x по сравнению с y — `Before`, то y по
сравнению с x — `After` (и так же для остальных трёх ответов).

<details>
<summary>Подсказка</summary>

Напишите вспомогательную функцию `fn clock(ticks: &[usize]) -> VectorClock`, которая делает tick `["a", "b", "c"][i]`
для каждого индекса, и стратегию `prop::collection::vec(0usize..3, 0..10)`. `matches!(value, Order::Before | Order::Equal)`
проверяет один из двух вариантов.

</details>

<details>
<summary>Ответ</summary>

```rust
use p18_logical_clocks::{Order, VectorClock};
use proptest::prelude::*;

/// Builds a clock by ticking the nodes "a", "b" and "c" in the given order.
fn clock(ticks: &[usize]) -> VectorClock {
    let mut clock = VectorClock::default();
    for &i in ticks {
        clock.tick(["a", "b", "c"][i]);
    }
    clock
}

fn ticks() -> impl Strategy<Value = Vec<usize>> {
    prop::collection::vec(0usize..3, 0..10)
}

proptest! {
    #[test]
    fn merging_gives_a_clock_that_knows_both(x in ticks(), y in ticks()) {
        let (x, y) = (clock(&x), clock(&y));
        let mut xy = x.clone();
        xy.merge(&y);
        let mut yx = y.clone();
        yx.merge(&x);
        prop_assert_eq!(&xy, &yx); // the order of merging does not matter
        prop_assert!(matches!(x.compare(&xy), Order::Before | Order::Equal));
        prop_assert!(matches!(y.compare(&xy), Order::Before | Order::Equal));
    }

    #[test]
    fn before_and_after_are_mirror_images(x in ticks(), y in ticks()) {
        let (x, y) = (clock(&x), clock(&y));
        let expected = match x.compare(&y) {
            Order::Before => Order::After,
            Order::After => Order::Before,
            Order::Equal => Order::Equal,
            Order::Concurrent => Order::Concurrent,
        };
        prop_assert_eq!(y.compare(&x), expected);
    }
}
```

`cargo test --test vector_props`:

```text
test before_and_after_are_mirror_images ... ok
test merging_gives_a_clock_that_knows_both ... ok

test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.01s
```

Почему это работает: максимум коммутативен, поэтому порядок слияния не может иметь значения, а максимум никогда не
меньше своих аргументов. Свойство 2 проверяет, что `compare` согласован сам с собой; опечатка в одной ветке его
`match` сразу его нарушила бы.

</details>

### Упражнение 3 — Сомневаешься — храни обе версии (сложно)

Создайте `src/siblings.rs` (и `pub mod siblings;`) с `Version { clock: VectorClock, value: String }` и функцией
`pub fn resolve(a: Version, b: Version) -> Vec<Version>`, которая возвращает только более новую версию, когда две
версии упорядочены (только одну, когда они равны), и **обе**, когда они конкурентны. Проверьте её на списке покупок:
`[r1:1] "milk"` заменяется на `[r1:2] "milk, eggs"`, а `[r1:2] "milk, eggs"` и `[r1:1 r2:1] "milk, bread"` (записанные
на двух репликах после того, как обе видели "milk") сохраняются обе.

<details>
<summary>Подсказка</summary>

Реализуйте `resolve` как `match` по `a.clock.compare(&b.clock)`. В тесте маленькая вспомогательная функция, которая
строит `Version` из списка имён узлов для tick, делает случаи читаемыми.

</details>

<details>
<summary>Ответ</summary>

```rust
//! Exercise 3: keep both values when two writes were concurrent.

use crate::{Order, VectorClock};

/// One stored version of a value, with the vector clock of the write that made it.
#[derive(Debug, Clone, PartialEq)]
pub struct Version {
    pub clock: VectorClock,
    pub value: String,
}

/// The versions that must be kept after seeing `a` and `b`: one if they are ordered, both if not.
pub fn resolve(a: Version, b: Version) -> Vec<Version> {
    match a.clock.compare(&b.clock) {
        Order::Before => vec![b],
        Order::After | Order::Equal => vec![a],
        Order::Concurrent => vec![a, b],
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    fn version(ticks: &[&str], value: &str) -> Version {
        let mut clock = VectorClock::default();
        for node in ticks {
            clock.tick(node);
        }
        Version {
            clock,
            value: value.to_string(),
        }
    }

    #[test]
    fn a_newer_version_replaces_an_older_one() {
        let old = version(&["r1"], "milk");
        let new = version(&["r1", "r1"], "milk, eggs");
        assert_eq!(resolve(old.clone(), new.clone()), vec![new.clone()]);
        assert_eq!(resolve(new.clone(), old), vec![new]);
    }

    #[test]
    fn concurrent_versions_are_both_kept() {
        let on_r1 = version(&["r1", "r1"], "milk, eggs"); // [r1:2]
        let on_r2 = version(&["r1", "r2"], "milk, bread"); // [r1:1 r2:1]
        let kept = resolve(on_r1.clone(), on_r2.clone());
        assert_eq!(kept, vec![on_r1, on_r2]);
    }
}
```

`cargo test --lib`:

```text
test result: ok. 3 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

Почему это работает: заменить другую версию может только та, что *позже* неё, потому что она видела всё, что было в
другой. Для конкурентных версий никто не может знать, какое изменение пользователь хотел сохранить, поэтому база
хранит обе, а приложение их объединяет — здесь оно показало бы "milk, eggs, bread".

</details>

## Вопросы для самопроверки

1. Событие a — отправка, а событие b — получение этого сообщения на другом узле. Что вы знаете об их временах
   Лэмпорта?

<details>
<summary>Ответ</summary>

L(a) < L(b), потому что получатель ставит свои часы как минимум в L(a) + 1. В общем виде: из a → b следует
L(a) < L(b).

</details>

2. L(x) = 4, а L(y) = 9. Произошло ли x раньше y?

<details>
<summary>Ответ</summary>

Не обязательно. Мы знаем только, что y не произошло раньше x. x и y могут быть конкурентны.

</details>

3. Сравните `[a:2 b:1]` с `[a:1 b:3]`, а также `[a:1 b:3]` с `[a:2 b:3]`.

<details>
<summary>Ответ</summary>

Первая пара конкурентна (a: 2 > 1, но b: 1 < 3). Во второй паре `[a:1 b:3]` раньше `[a:2 b:3]` (одна запись меньше,
ни одной больше).

</details>

4. Почему реплицируемая база данных хранит два сиблинга, а не выбирает один?

<details>
<summary>Ответ</summary>

Потому что их векторные часы конкурентны: ни одна запись не знала о другой, поэтому «более новой» нет. Выбор одной
незаметно потерял бы изменение другого пользователя; хранение обеих позволяет приложению их объединить.

</details>

5. Какова главная цена векторных часов по сравнению с часами Лэмпорта?

<details>
<summary>Ответ</summary>

Размер: у векторных часов по записи на каждый узел, и их нужно отправлять с каждым сообщением и хранить с каждой
версией. Часы Лэмпорта — одно число.

</details>

## Итоги

- «Произошло раньше» (a → b) значит, что a могло повлиять на b: тот же узел по порядку или связь сообщениями.
  События без такого пути конкурентны.
- Часы Лэмпорта — один счётчик на узел: tick на каждое событие, `max(own, remote) + 1` при получении. Из a → b
  следует L(a) < L(b), но не наоборот.
- Пары `(L, id узла)` дают полный порядок, уважающий причинность.
- Векторные часы хранят по счётчику на узел и различают «раньше», «позже», «равны» и «конкурентны».
- Базы данных без лидера используют векторные часы, чтобы хранить конкурентные версии как сиблинги, а не терять одну
  из них.
- Векторные часы растут с числом узлов; гибридные логические часы — частый компромисс.

## Источники и что почитать

- [Time, Clocks, and the Ordering of Events in a Distributed System](https://lamport.azurewebsites.net/pubs/time-clocks.pdf) — оригинальная статья Лесли Лэмпорта 1978 года; короткая и читаемая, с отношением «произошло раньше» и правилом часов.
- [Vector clock](https://en.wikipedia.org/wiki/Vector_clock) — обзор векторных часов с правилами обновления и сравнения и пространственно-временной диаграммой.
- [Designing Data-Intensive Applications](https://dataintensive.net/) — главы о репликации и согласованности показывают, как векторы версий обнаруживают конкурентные записи в базах данных без лидера.
- [Dynamo: Amazon's Highly Available Key-value Store](https://www.allthingsdistributed.com/files/amazon-dynamo-sosp2007.pdf) — статья, сделавшая векторные часы и сиблинги известными в настоящих базах данных (пример с корзиной покупок — оттуда).
- [Документация `proptest`](https://docs.rs/proptest/latest/proptest/) — стратегии и генераторы коллекций из Упражнения 2.

---

[🏠 Начало](../START-HERE.md) · [← 18.4 Время и часы](04-time-and-clocks.md) · [📋 Этап 18](README.md) · [18.6 Репликация →](06-replication.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
