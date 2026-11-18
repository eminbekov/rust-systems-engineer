<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 19.5 Профилирование памяти и меньше выделений](05-memory-profiling-and-allocations.md) · [📋 Этап 19](README.md) · [19.7 SIMD: одна инструкция, много данных →](07-simd.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/19-performance-and-profiling/06-data-layout-for-speed.md) <!-- nav -->

# Урок 19.6 — Размещение данных для скорости

> **Этап 19 — Производительность и профилирование** · Урок 6 из 11 · ⏱ около 135 минут

## Что вы узнаете

- Почему *форма* данных часто важнее, чем код, который по ним проходит.
- Как один редкий большой вариант перечисления делает большим каждое значение и как это исправляет `Box`.
- Как хранить граф в двух плоских массивах (CSR) вместо `Vec` на каждую вершину и когда это помогает.
- Как выбирать между `HashMap`, более быстрым хешером и простым `Vec` для подсчёта по ключу.
- Как защитить решение о размещении данных проверкой на этапе компиляции и тестом.

## Перед началом

- Вы прошли [Урок 19.5 — Профилирование памяти и меньше выделений](05-memory-profiling-and-allocations.md).
- Вы знаете кэши и кэш-линии ([Урок 12.9 — Иерархия памяти и кэши](../12-computer-systems/09-memory-hierarchy-and-caches.md)), четыре правила из [Урока 12.10 — Код, дружелюбный к кэшу](../12-computer-systems/10-cache-friendly-code.md), а также размеры, заполнение и ниши из [Урока 12.11 — Размещение типов Rust в памяти](../12-computer-systems/11-memory-layout-of-rust-types.md).
- Вы знаете списки смежности и поиск в ширину ([Урок 07.13 — Графы: BFS и DFS](../07-algorithms-and-data-structures/13-graphs-bfs-and-dfs.md)) и хеш-таблицы ([Урок 07.6 — Как работают хеш-таблицы](../07-algorithms-and-data-structures/06-hash-tables.md)).

## Новые слова

| Слово | Значение |
|---|---|
| **размещение данных** (data layout) | Как значения расположены в памяти: их размеры, порядок и то, лежат ли они рядом или за указателями. |
| **горячие данные** (hot data) | Данные, которые занятый цикл программы читает снова и снова. Их размещение определяет скорость. |
| **пропускная способность памяти** (memory bandwidth) | Сколько байт в секунду может перемещаться между RAM (Random Access Memory, оперативная память) и процессором. Цикл, который читает больше байт, чем ему нужно, тратит её впустую. |
| **CSR** (Compressed Sparse Row, сжатое хранение строк) | Способ хранения графов и разреженных матриц: все списки соседей подряд в одном массиве плюс массив начальных позиций. |
| **смещение** (offset) | Позиция внутри массива, отсчитанная от его начала. В CSR `offsets[v]` говорит, где начинаются соседи `v`. |
| **плотные ключи** (dense keys) | Ключи — небольшие целые числа без больших пропусков, например `0..10_000`. Их можно прямо использовать как индексы в `Vec`. |
| **хешер** (hasher) | Алгоритм, который превращает ключ в хеш-число для `HashMap`. Стандартный, SipHash, защищён от атак, но не самый быстрый. |
| **FxHash** | Очень простая и очень быстрая хеш-функция из компилятора Rust, доступная как крейт `rustc-hash`. Не защищена от целенаправленных атак. |
| **хеш-флудинг** (hash flooding) | Атака, при которой отправляется множество ключей с одинаковым хешем, чтобы хеш-таблица стала очень медленной (Урок 04.9). |

## 1. Форма данных

Уроки 12.9–12.11 дали факты об оборудовании: на вашем Mac процессор загружает целые кэш-линии по 128 байт, проход по памяти по порядку быстрый, прыжки по указателям медленные, а у каждого типа есть размер и заполнение. Эти уроки закончились четырьмя правилами. Этот урок добавляет три изменения в размещении данных, на которые часто указывают профили реальных программ:

```text
1. Keep the hot data small           a rare big variant makes every value big
2. Flat arrays instead of many Vecs   one allocation and one scan instead of millions
3. Choose the map for your keys      dense keys -> Vec; trusted keys -> faster hasher
```

Каждое из них измерено ниже. Пакет для этого урока:

```bash
cd ~/projects/rust/code
cargo new --lib p19-layout
cd p19-layout
mkdir src/bin
cargo add rustc-hash@2.1
```

В `src/lib.rs` лежат вспомогательная функция для замеров из Урока 12.10 и небольшой генератор случайных чисел, чтобы каждый пример при каждом запуске использовал одни и те же данные:

```rust
//! Helpers shared by the examples of Lesson 19.6.
use std::hint::black_box;
use std::time::{Duration, Instant};

/// Runs `work` five times and returns the fastest time and the result.
/// (The same helper as in Lesson 12.10.)
pub fn best_of_5<T>(mut work: impl FnMut() -> T) -> (Duration, T) {
    let mut best = Duration::MAX;
    let mut result = None;
    for _ in 0..5 {
        let start = Instant::now();
        let value = black_box(work());
        best = best.min(start.elapsed());
        result = Some(value);
    }
    (best, result.expect("the loop runs five times"))
}

/// A small, repeatable pseudo-random number generator (xorshift).
pub struct Rng(pub u64);

impl Rng {
    pub fn next_u64(&mut self) -> u64 {
        self.0 ^= self.0 << 13;
        self.0 ^= self.0 >> 7;
        self.0 ^= self.0 << 17;
        self.0
    }

    /// A number in `0..n`.
    pub fn below(&mut self, n: u64) -> u64 {
        self.next_u64() % n
    }
}
```

## 2. Держите горячие данные маленькими

Размер перечисления равен размеру его самого большого варианта плюс тег (Урок 12.11). Это нормально, когда все варианты похожи. Это вредит, когда один вариант редкий и огромный: тогда за него платит *каждое* значение. Представьте программу, которая обрабатывает события пользовательского интерфейса. Почти все они — клики; изредка пользователь вставляет блок текста, который хранится прямо внутри, в буфере на 256 байт. `src/bin/events.rs`:

```rust
use p19_layout::{Rng, best_of_5};
use std::hint::black_box;

/// One rare variant with a big inline buffer makes every value big.
#[allow(dead_code)] // this example reads only the clicks
enum Event {
    Click { x: u32, y: u32 },
    Key(u32),
    Paste([u8; 256]),
}

/// The same events, but the rare big payload lives on the heap.
#[allow(dead_code)]
enum SmallEvent {
    Click { x: u32, y: u32 },
    Key(u32),
    Paste(Box<[u8; 256]>),
}

fn sum_clicks(events: &[Event]) -> u64 {
    let mut sum = 0;
    for e in events {
        if let Event::Click { x, y } = e {
            sum += u64::from(*x + *y);
        }
    }
    sum
}

fn sum_clicks_small(events: &[SmallEvent]) -> u64 {
    let mut sum = 0;
    for e in events {
        if let SmallEvent::Click { x, y } = e {
            sum += u64::from(*x + *y);
        }
    }
    sum
}

fn main() {
    println!("size_of::<Event>()      = {} bytes", size_of::<Event>());
    println!(
        "size_of::<SmallEvent>() = {} bytes",
        size_of::<SmallEvent>()
    );

    let n = 1_000_000;
    let mut rng = Rng(42);
    let mut big = Vec::with_capacity(n);
    let mut small = Vec::with_capacity(n);
    for _ in 0..n {
        match rng.below(100) {
            0 => {
                big.push(Event::Paste([7; 256]));
                small.push(SmallEvent::Paste(Box::new([7; 256])));
            }
            1 => {
                big.push(Event::Key(rng.below(128) as u32));
                small.push(SmallEvent::Key(1));
            }
            _ => {
                let (x, y) = (rng.below(1920) as u32, rng.below(1080) as u32);
                big.push(Event::Click { x, y });
                small.push(SmallEvent::Click { x, y });
            }
        }
    }
    let (t, s) = best_of_5(|| sum_clicks(black_box(&big)));
    println!("Vec<Event>:      {t:>9.2?}  (sum {s})");
    let (t, s) = best_of_5(|| sum_clicks_small(black_box(&small)));
    println!("Vec<SmallEvent>: {t:>9.2?}  (sum {s})");
}
```

**Что происходит:**

1. `Event` хранит буфер вставки внутри себя: 256 байт плюс тег, с округлением до выравнивания `u32`. `SmallEvent` хранит только `Box` — 8-байтовый указатель, а сам буфер лежит в куче.
2. Цикл создаёт миллион событий: 98% кликов, 1% нажатий клавиш, 1% вставок. В оба вектора идут одни и те же случайные числа, поэтому суммы должны совпасть.
3. `sum_clicks` читает у каждого события только тег и две координаты. Всё остальное в значении в этом цикле никогда не нужно, но всё равно лежит между полезными байтами.

`cargo run --release --bin events`:

```text
size_of::<Event>()      = 260 bytes
size_of::<SmallEvent>() = 16 bytes
Vec<Event>:         4.94ms  (sum 1468695352)
Vec<SmallEvent>:  423.08µs  (sum 1468695352)
```

Тот же ответ — в **11 раз быстрее**. Вектор `Event` занимает 260 МБ (мегабайт): на каждый клик процессор загружает целую кэш-линию в 128 байт (события лежат на расстоянии 260 байт, поэтому никакие два не делят одну линию) и использует из неё только 12 байт. Вектор `SmallEvent` — 16 МБ, восемь событий на кэш-линию. Редкая вставка теперь стоит лишнего прыжка по указателю, но только для 1% событий, а цикл выше её вообще не трогает.

`cargo clippy` знает эту проблему и предупреждает о ней:

```text
warning: large size difference between variants
  --> src/bin/events.rs:6:1
   |
 6 | / enum Event {
 7 | |     Click { x: u32, y: u32 },
   | |     ------------------------ the second-largest variant contains at least 8 bytes
 8 | |     Key(u32),
 9 | |     Paste([u8; 256]),
   | |     ---------------- the largest variant contains at least 256 bytes
10 | | }
   | |_^ the entire enum is at least 260 bytes
   |
   = help: for further information visit https://rust-lang.github.io/rust-clippy/rust-1.100.0/index.html#large_enum_variant
   = note: `#[warn(clippy::large_enum_variant)]` on by default
```

Та же идея работает для структур и целых чисел. ID типа `usize` занимает 8 байт; если у вас никогда не будет больше четырёх миллиардов элементов, ID типа `u32` занимает 4 байта, и в кэш-линию помещается вдвое больше. Редко используемое поле `String`, которое обычно пустое, можно сделать `Option<Box<str>>`: 16 байт вместо 24, и `None` ничего не стоит дополнительно благодаря нише из Урока 12.11.

> ⚡ **Производительность:** меньший тип помогает, только если он *горячий*: его читают в занятом цикле по множеству значений. Упаковка варианта в `Box` добавляет выделение памяти при создании значения и прыжок по указателю при использовании большого варианта. Измеряйте обе стороны; для типа, который живёт в нескольких переменных, размер не важен.

## 3. Плоские массивы вместо множества маленьких векторов

В Уроке 07.13 вы хранили графы как **списки смежности**, `Vec<Vec<usize>>`: по вектору соседей на вершину. Это просто и правильно. Для большого графа это значит одно выделение памяти на вершину, 24 байта заголовка `Vec` на вершину, 8-байтовые цели `usize` и списки соседей, разбросанные по куче.

**CSR** (Compressed Sparse Row) хранит тот же граф в двух плоских массивах:

```text
adjacency lists              CSR
0: [1, 2]                    targets: [1, 2, | 0, 0, 3, | 2]     (vertex 1 has none)
1: []                        offsets: [0, 2, 2, 5, 6]
2: [0, 0, 3]                 neighbours of v = targets[offsets[v] .. offsets[v+1]]
3: [2]                       neighbours of 2 = targets[2..5] = [0, 0, 3]
```

`targets` хранит все списки соседей один за другим; `offsets[v]` говорит, где начинается список `v`, а `offsets[v + 1]` — где он заканчивается. С `u32` вместо `usize` каждое число вдвое меньше. `src/bin/graph.rs` строит случайный граф из двух миллионов вершин по восемь рёбер у каждой в обоих размещениях и выполняет две задачи: полный проход по всем рёбрам и поиск в ширину (BFS, breadth-first search) из вершины 0:

```rust
use p19_layout::{Rng, best_of_5};
use std::collections::VecDeque;
use std::hint::black_box;

/// The graph as in Lesson 07.13: one Vec of neighbours per vertex.
fn build_lists(n: usize, degree: usize, rng: &mut Rng) -> Vec<Vec<usize>> {
    (0..n)
        .map(|_| (0..degree).map(|_| rng.below(n as u64) as usize).collect())
        .collect()
}

/// The same graph in two flat arrays (CSR, "compressed sparse row"):
/// the neighbours of `v` are `targets[offsets[v]..offsets[v + 1]]`.
struct Csr {
    offsets: Vec<u32>,
    targets: Vec<u32>,
}

impl Csr {
    fn from_lists(lists: &[Vec<usize>]) -> Csr {
        let mut offsets = Vec::with_capacity(lists.len() + 1);
        let mut targets = Vec::new();
        offsets.push(0);
        for list in lists {
            targets.extend(list.iter().map(|&t| t as u32));
            offsets.push(targets.len() as u32);
        }
        Csr { offsets, targets }
    }

    fn neighbours(&self, v: usize) -> &[u32] {
        &self.targets[self.offsets[v] as usize..self.offsets[v + 1] as usize]
    }
}

/// Breadth-first search from vertex 0; returns how many vertices were reached.
fn bfs_lists(lists: &[Vec<usize>]) -> usize {
    let mut seen = vec![false; lists.len()];
    let mut queue = VecDeque::from([0]);
    seen[0] = true;
    let mut reached = 0;
    while let Some(v) = queue.pop_front() {
        reached += 1;
        for &u in &lists[v] {
            if !seen[u] {
                seen[u] = true;
                queue.push_back(u);
            }
        }
    }
    reached
}

fn bfs_csr(g: &Csr) -> usize {
    let mut seen = vec![false; g.offsets.len() - 1];
    let mut queue = VecDeque::from([0u32]);
    seen[0] = true;
    let mut reached = 0;
    while let Some(v) = queue.pop_front() {
        reached += 1;
        for &u in g.neighbours(v as usize) {
            if !seen[u as usize] {
                seen[u as usize] = true;
                queue.push_back(u);
            }
        }
    }
    reached
}

/// Sum of all edge targets: a pure scan, no random access.
fn sum_lists(lists: &[Vec<usize>]) -> u64 {
    lists.iter().flatten().map(|&u| u as u64).sum()
}

fn sum_csr(g: &Csr) -> u64 {
    g.targets.iter().map(|&u| u64::from(u)).sum()
}

fn main() {
    let (n, degree) = (2_000_000, 8);
    let mut rng = Rng(7);
    let lists = build_lists(n, degree, &mut rng);
    let csr = Csr::from_lists(&lists);

    let list_bytes = n * size_of::<Vec<usize>>() + n * degree * size_of::<usize>();
    let csr_bytes = (csr.offsets.len() + csr.targets.len()) * size_of::<u32>();
    println!("Vec<Vec<usize>>: {} MB", list_bytes / 1_000_000);
    println!("CSR with u32:    {} MB", csr_bytes / 1_000_000);

    let (t, r) = best_of_5(|| sum_lists(black_box(&lists)));
    println!("sum Vec<Vec<usize>>: {t:>9.2?}  ({r})");
    let (t, r) = best_of_5(|| sum_csr(black_box(&csr)));
    println!("sum CSR:             {t:>9.2?}  ({r})");
    let (t, r) = best_of_5(|| bfs_lists(black_box(&lists)));
    println!("BFS Vec<Vec<usize>>: {t:>9.2?}  ({r} reached)");
    let (t, r) = best_of_5(|| bfs_csr(black_box(&csr)));
    println!("BFS CSR:             {t:>9.2?}  ({r} reached)");
}
```

**Что происходит:**

1. `from_lists` копирует каждый список в `targets` и записывает, где каждый из них заканчивается. В `offsets` на один элемент больше, чем вершин, поэтому `offsets[v + 1]` существует всегда.
2. `neighbours(v)` возвращает срез `targets`: без копирования и без выделения памяти.
3. Две функции BFS — это алгоритм из Урока 07.13, по одной на каждое размещение. Оценка памяти учитывает заголовки `Vec` (по 24 байта) и цели; у настоящего `Vec<Vec<usize>>` есть ещё накладные расходы аллокатора на каждый список.

`cargo run --release --bin graph`:

```text
Vec<Vec<usize>>: 176 MB
CSR with u32:    72 MB
sum Vec<Vec<usize>>:   10.17ms  (15999321722498)
sum CSR:              785.50µs  (15999321722498)
BFS Vec<Vec<usize>>:  110.10ms  (1999337 reached)
BFS CSR:               97.02ms  (1999337 reached)
```

Читайте результат внимательно, потому что это не «CSR всегда в 13 раз быстрее»:

- **Память:** 72 МБ вместо 176 МБ и 2 выделения памяти вместо 2 миллионов (построение и удаление списков тоже занимает время).
- **Проход по всем рёбрам:** в 13 раз быстрее. CSR — один непрерывный массив 4-байтовых чисел, который процессор читает по порядку (а компилятор может складывать несколько чисел за раз); спискам нужен прыжок по указателю на каждую вершину и вдвое больше байт.
- **BFS:** быстрее всего примерно на 10%. BFS прыгает по случайным вершинам: `seen[u]` и список соседей каждой новой вершины каждый раз лежат в другом месте памяти. Эти случайные обращения преобладают, и оба размещения за них платят.

Вот честное правило: компактное размещение больше всего помогает там, где цикл идёт по памяти по порядку, и даёт меньший выигрыш там, где сам алгоритм прыгает. Графовые базы данных, движки маршрутизации и код для разреженных матриц используют CSR, потому что он выигрывает по памяти и проходам и никогда не проигрывает.

> 🧪 **Тестирование:** два размещения одних и тех же данных должны давать одинаковые ответы. Упражнение 2 просит проверить тестом, что `Csr` возвращает ровно тех же соседей, что и списки, включая вершину без соседей и повторяющегося соседа.

## 4. Выбирайте словарь под ключи

Подсчёт по ключу встречается повсюду: запросы на пользователя, слова в тексте, пакеты на адрес. Обычный инструмент — `HashMap`. Стандартный использует **SipHash** со случайным ключом (Урок 07.6), который защищает от **хеш-флудинга**, но стоит времени при каждом поиске. Если ключи — небольшие целые числа без больших пропусков, хеширование вообще не нужно. `src/bin/maps.rs` считает десять миллионов случайных ID от 0 до 9 999 тремя способами:

```rust
use p19_layout::{Rng, best_of_5};
use rustc_hash::FxHashMap;
use std::collections::HashMap;
use std::hint::black_box;

/// Counts how often each id appears, with the standard HashMap (SipHash).
fn count_std(ids: &[u32]) -> HashMap<u32, u32> {
    let mut counts = HashMap::new();
    for &id in ids {
        *counts.entry(id).or_insert(0) += 1;
    }
    counts
}

/// The same with FxHashMap: a much simpler and faster hash function.
fn count_fx(ids: &[u32]) -> FxHashMap<u32, u32> {
    let mut counts = FxHashMap::default();
    for &id in ids {
        *counts.entry(id).or_insert(0) += 1;
    }
    counts
}

/// Ids are small and dense (0..max), so a plain Vec indexed by id is enough.
fn count_vec(ids: &[u32], max: u32) -> Vec<u32> {
    let mut counts = vec![0; max as usize];
    for &id in ids {
        counts[id as usize] += 1;
    }
    counts
}

fn main() {
    let max = 10_000;
    let mut rng = Rng(1);
    let ids: Vec<u32> = (0..10_000_000).map(|_| rng.below(max as u64) as u32).collect();

    let (t, c) = best_of_5(|| count_std(black_box(&ids)));
    println!("HashMap (SipHash): {t:>9.2?}  (id 7 seen {} times)", c[&7]);
    let (t, c) = best_of_5(|| count_fx(black_box(&ids)));
    println!("FxHashMap:         {t:>9.2?}  (id 7 seen {} times)", c[&7]);
    let (t, c) = best_of_5(|| count_vec(black_box(&ids), max));
    println!("Vec by index:      {t:>9.2?}  (id 7 seen {} times)", c[7]);
}
```

**Что происходит:**

1. `FxHashMap<K, V>` — это стандартный `HashMap` с другим хешером: `HashMap<K, V, FxBuildHasher>`. Он создаётся через `default()`, а не `new()`, потому что `new()` существует только для хешера по умолчанию. Всё остальное (`entry`, `get`, `insert`) то же самое.
2. `count_vec` использует сам ID как индекс. Нет ни хеширования, ни сравнения, ни поиска свободного места: одно обращение к памяти на ID в массив размером 40 КБ (килобайт), который остаётся в кэше.
3. `c[&7]` читает словарь по ключу; `c[7]` читает вектор по индексу. Все три печатают одно и то же количество — это показывает, что три версии согласны.

`cargo run --release --bin maps`:

```text
HashMap (SipHash):   48.51ms  (id 7 seen 1049 times)
FxHashMap:           10.96ms  (id 7 seen 1049 times)
Vec by index:         2.82ms  (id 7 seen 1049 times)
```

Здесь FxHash в 4,4 раза быстрее SipHash, а `Vec` — в 17 раз. Короткая памятка:

| Ключи | Выбор |
|---|---|
| небольшие целые числа в известном диапазоне (`0..n`, с малым числом пропусков) | `Vec<V>`, индексированный ключом |
| ваши собственные данные, которые не выбирает посторонний (ID из вашей базы данных, слова из ваших собственных файлов) | `FxHashMap` (или другой быстрый хешер) |
| всё, что приходит от пользователей или из сети (заголовки HTTP, HyperText Transfer Protocol, протокола передачи гипертекста; имена пользователей; ключи JSON, JavaScript Object Notation) | стандартный `HashMap` |

> 🔒 **Безопасность:** у FxHash нет секретного ключа. Атакующий, который может выбирать ключи, легко создаст тысячи ключей, попадающих в одну корзину, и каждый поиск превратится в медленный перебор: это DoS-атака (Denial of Service, отказ в обслуживании), называемая хеш-флудингом (Урок 04.9). Никогда не используйте быстрый хешер без ключа для ключей из недоверенного ввода. Если сомневаетесь — оставляйте стандартный.

## Частые ошибки

1. **Менять размещение без профиля.** Изменение размещения делает код сложнее для чтения. Делайте его для данных, которые профиль показывает горячими (Уроки 19.3 и 19.4), и измеряйте до и после.
2. **Упаковывать в `Box` всё подряд.** `Box` помогает *редкому* большому варианту. Упаковка частого варианта добавляет выделение памяти и прыжок по указателю большинству значений и замедляет программу.
3. **ID типа `u32`, которые могут переполниться.** Преобразование через `as u32` молча отрезает большие значения (Урок 12.2). Если возможно больше четырёх миллиардов элементов, используйте `u64` или преобразуйте через `u32::try_from(x)` и обрабатывайте ошибку.
4. **Быстрый хешер на сетевом вводе.** См. замечание о безопасности выше. `FxHashMap` — для внутренних, доверенных данных.
5. **Сравнивать размещения на объёме, который помещается в кэш.** При 1000 событий оба размещения перечисления помещаются в кэши и работают почти одинаково. Эффекты размещения появляются, когда горячие данные больше кэшей (Урок 12.10). Проверяйте на реалистичных объёмах.

## Попробуйте сами

1. В `events.rs` поменяйте смесь событий на 50% кликов и 49% нажатий клавиш (`1..=49 =>`). Разница между размещениями уменьшается. Почему? (Подсказка: Урок 12.6. `if let` теперь — ветвление, которое процессор не может предсказать, и цена неверных догадок скрывает цену памяти.)
2. В `graph.rs` поменяйте `degree` на 2, а потом на 32. Как меняется разрыв между двумя проходами?
3. В `maps.rs` поменяйте `max` на 10 000 000. Теперь `Vec` занимает 40 МБ и больше не помещается в кэши. Какая версия выигрывает теперь?

## Упражнения

### Упражнение 1 — Охраняйте размер на этапе компиляции (легко)

Добавьте в `events.rs` проверку, из-за которой сборка упадёт, если `SmallEvent` когда-нибудь станет больше 16 байт. Затем, чтобы увидеть её в деле, верните в `SmallEvent` `Paste([u8; 256])` вместо `Paste(Box<[u8; 256]>)` (и поправьте строку, которая его создаёт) и соберите.

<details>
<summary>Подсказка</summary>

В Уроке 12.11 использовалось `const _: () = assert!(...);`. `const` вычисляет компилятор, поэтому проваленный `assert!` — это ошибка компиляции.

</details>

<details>
<summary>Ответ</summary>

После определения `SmallEvent`:

```rust
// Fails to compile if someone makes SmallEvent big again.
const _: () = assert!(size_of::<SmallEvent>() <= 16);
```

С буфером внутри `cargo build --release --bin events` падает:

```text
error[E0080]: evaluation panicked: assertion failed: size_of::<SmallEvent>() <= 16
  --> src/bin/events.rs:21:15
   |
21 | const _: () = assert!(size_of::<SmallEvent>() <= 16);
   |               ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ evaluation of `_` failed here
```

Почему это работает: проверка выполняется во время компиляции и ничего не стоит при работе программы. Она превращает решение о производительности в то, за чем следит компилятор: коллега, который добавит большое поле, получит ошибку с понятной причиной, а не замедление, которого никто не заметит. Потом верните `Box` на место.

</details>

### Упражнение 2 — Протестируйте размещение CSR (средне)

Напишите в `graph.rs` два теста: (а) для маленького графа `[[1, 2], [], [0, 0, 3], [2]]` `Csr::from_lists` даёт `offsets` `[0, 2, 2, 5, 6]`, а `neighbours(v)` совпадает с каждым списком; (б) для случайного графа из 1000 вершин со степенью 2 обе функции BFS достигают одинакового числа вершин. Запустите их командой `cargo test --bin graph`.

<details>
<summary>Подсказка</summary>

Поля `Csr` приватные, но тестовый модуль находится в том же файле, поэтому может их читать. Чтобы сравнить `&[u32]` с `Vec<usize>`, сначала преобразуйте список: `list.iter().map(|&u| u as u32).collect::<Vec<u32>>()`.

</details>

<details>
<summary>Ответ</summary>

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn csr_has_the_same_neighbours() {
        // Vertex 1 has no neighbours; vertex 2 has a repeated one.
        let lists = vec![vec![1, 2], vec![], vec![0, 0, 3], vec![2]];
        let csr = Csr::from_lists(&lists);
        assert_eq!(csr.offsets, [0, 2, 2, 5, 6]);
        for (v, list) in lists.iter().enumerate() {
            let expected: Vec<u32> = list.iter().map(|&u| u as u32).collect();
            assert_eq!(csr.neighbours(v), expected, "vertex {v}");
        }
    }

    #[test]
    fn both_searches_reach_the_same_vertices() {
        let lists = build_lists(1000, 2, &mut Rng(3));
        assert_eq!(bfs_lists(&lists), bfs_csr(&Csr::from_lists(&lists)));
    }
}
```

```text
test tests::csr_has_the_same_neighbours ... ok
test tests::both_searches_reach_the_same_vertices ... ok

test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

Почему это работает: первый тест покрывает граничные случаи, на которых обычно ломается код со смещениями (пустой список даёт два одинаковых смещения подряд; повторяющегося соседа нужно сохранить). Второй сравнивает два размещения на данных побольше через алгоритм, который их использует.

</details>

### Упражнение 3 — Быстрый хешер для счётчика слов (сложно)

В `code/p19-wordfreq` (Урок 19.5) `HashMap<String, usize>` был одной из самых больших частей профиля в Linux. Замените его на `FxHashMap` везде (в программе, `count_words`, `top` и тестах), проверьте, что вывод на `words.txt` тот же, и измерьте через `hyperfine`. Безопасно ли это для этой программы?

<details>
<summary>Подсказка</summary>

`cargo add rustc-hash@2.1`, затем `use rustc_hash::FxHashMap;` и замена `HashMap::new()` на `FxHashMap::default()`. Сохраните копию старого бинарника для сравнения.

</details>

<details>
<summary>Ответ</summary>

Изменённые строки в `src/main.rs`:

```text
use rustc_hash::FxHashMap;

fn count_words(text: &str, counts: &mut FxHashMap<String, usize>) {
fn top(counts: &FxHashMap<String, usize>, n: usize) -> Vec<(String, usize)> {
    let mut counts = FxHashMap::default();
```

(Строки с `count_words` и `top` показывают только новые сигнатуры; тела остаются прежними. Замените каждый `HashMap::new()` в `main` и в тестах на `FxHashMap::default()`.) `std-wordfreq` — копия бинарника до изменения:

```text
identical output
Benchmark 1: ./std-wordfreq words.txt (10 runs)
                mean     ±       σ          min     …     max
  Wall Time    714.8 ms  ±     6.5 ms     706.9 ms  …   725.1 ms
  Memory       130.9 MiB ±     0.4 MiB    130.4 MiB …   131.4 MiB
 
Benchmark 2: ./target/release/p19-wordfreq words.txt (10 runs)
                mean     ±       σ          min     …     max
  Wall Time    629.3 ms  ±    13.8 ms     613.1 ms  …   663.5 ms           -12.0%
  Memory       130.8 MiB ±     0.4 MiB    130.4 MiB …   131.4 MiB           -0.1%
```

Почему это работает: хеширование короткого слова через FxHash занимает несколько инструкций вместо раундов SipHash, которые были видны в профилях. Вывод не меняется, потому что `top` сортирует по количеству, а затем по слову, так что порядок обхода словаря не важен. Безопасно ли это? Для инструмента командной строки, который считает слова в ваших собственных файлах, — да. Если бы этот код когда-нибудь работал в сервисе на тексте от посторонних, атакующий мог бы прислать слова, подобранные для коллизий, и сильно его замедлить; там оставьте стандартный `HashMap`.

</details>

## Вопросы для самопроверки

1. У перечисления есть варианты размером 8, 4 и 256 байт. Сколько примерно занимает `Vec` из миллиона таких значений и почему?

<details>
<summary>Ответ</summary>

Около 260 МБ: каждое значение занимает размер наибольшего варианта плюс тег (здесь 260 байт), какой бы вариант оно ни содержало.

</details>

2. Когда упаковка большого варианта в `Box` делает программу *медленнее*?

<details>
<summary>Ответ</summary>

Когда большой вариант частый или читается в горячем цикле: тогда большинству значений нужно лишнее выделение памяти при создании и лишний прыжок по указателю при чтении.

</details>

3. Из каких двух массивов состоит граф в CSR и как найти соседей вершины `v`?

<details>
<summary>Ответ</summary>

`targets` (все списки соседей подряд) и `offsets` (где начинается каждый список, плюс один лишний элемент в конце). Соседи `v` — это `targets[offsets[v]..offsets[v + 1]]`.

</details>

4. Почему CSR ускорил полный проход в 13 раз, а BFS — всего примерно на 10%?

<details>
<summary>Ответ</summary>

Проход читает один непрерывный массив по порядку, поэтому компактное размещение (меньше байт, нет указателя на вершину) окупается полностью. BFS прыгает по случайным вершинам и случайным позициям в `seen`; эти случайные обращения к памяти занимают основную часть времени и стоят одинаково в обоих размещениях.

</details>

5. Когда *не* стоит использовать `FxHashMap`?

<details>
<summary>Ответ</summary>

Когда ключи может выбирать кто-то вне вашего контроля (пользователи, сетевой ввод). Без секретного ключа атакующий может создать множество ключей с коллизиями и замедлить словарь (хеш-флудинг).

</details>

6. Вы считаете события по часам суток. Какой словарь вы выберете?

<details>
<summary>Ответ</summary>

Простой массив или `Vec` из 24 элементов с индексом-часом. Ключи маленькие и плотные; хеширование не нужно.

</details>

## Итоги

- Размещение горячих данных часто определяет скорость цикла. Меняйте его там, куда указывает профиль, и измеряйте.
- Перечисление занимает столько же, сколько его самый большой вариант: упаковывайте редкий большой вариант в `Box` (об этом предупреждает `large_enum_variant` в clippy); используйте ID типа `u32`, где позволяет диапазон.
- Каждый из множества маленьких `Vec` стоит одного выделения памяти и одного указателя; граф в CSR хранит всё в двух плоских массивах: здесь в 2,4 раза меньше памяти и в 13 раз более быстрые проходы, но для BFS — всего около 10%.
- Плотным целочисленным ключам место в `Vec`; для доверенных ключей подходит быстрый хешер вроде `FxHashMap`; ключи из недоверенного ввода остаются в стандартном `HashMap`.
- Защищайте решения о размещении проверками размера через `const` и тестами, которые сравнивают старое и новое размещения.

## Источники и что почитать

- [The Rust Performance Book — Type Sizes](https://nnethercote.github.io/perf-book/type-sizes.html) — как измерять и уменьшать типы, включая упаковку больших вариантов.
- [The Rust Performance Book — Hashing](https://nnethercote.github.io/perf-book/hashing.html) — альтернативные хешеры и когда они уместны.
- [Линты Clippy](https://rust-lang.github.io/rust-clippy/stable/index.html) — найдите `large_enum_variant`, чтобы прочитать объяснение этого линта.
- [Документация rustc-hash](https://docs.rs/rustc-hash/latest/rustc_hash/) — `FxHashMap`, `FxHashSet` и `FxBuildHasher`.
- [`std::collections::HashMap`](https://doc.rust-lang.org/std/collections/struct.HashMap.html) — хешер по умолчанию, `with_hasher` и замечание о хеш-флудинге.
- [Wikipedia — Sparse matrix](https://en.wikipedia.org/wiki/Sparse_matrix) — формат CSR (compressed sparse row), объяснённый с картинками.

---

[🏠 Начало](../START-HERE.md) · [← 19.5 Профилирование памяти и меньше выделений](05-memory-profiling-and-allocations.md) · [📋 Этап 19](README.md) · [19.7 SIMD: одна инструкция, много данных →](07-simd.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
