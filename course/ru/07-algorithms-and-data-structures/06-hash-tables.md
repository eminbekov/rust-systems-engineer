<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 07.5 Стеки и очереди](05-stacks-and-queues.md) · [📋 Этап 07](README.md) · [07.7 Рекурсия и стек вызовов →](07-recursion.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/07-algorithms-and-data-structures/06-hash-tables.md) <!-- nav -->

# Урок 07.6 — Как работают хеш-таблицы

> **Этап 07 — Алгоритмы и структуры данных** · Урок 6 из 20 · ⏱ около 2,5 часа (2 занятия)

## Что вы узнаете

- Что делает хеш-функция, что такое `Hash` и `Hasher` в Rust и какой контракт связывает `Hash` и `Eq`.
- Что такое коллизия и два способа с ней справиться: цепочки и открытая адресация.
- Почему хеш-таблица — это `O(1)` в среднем и `O(n)` в худшем случае и при чём тут коэффициент заполнения и перехеширование.
- Как построить хеш-таблицу с цепочками, увидеть, как плохая хеш-функция её губит, и сравнить с `std::collections::HashMap`.

## Перед началом

- Вы прошли [Урок 07.5 — Стеки и очереди](05-stacks-and-queues.md).
- Вы прочитали раздел 7 [Урока 04.9 — Хеш-таблицы (HashMap)](../04-structs-enums-collections-errors/09-hash-maps.md) («Как хеш-таблица устроена внутри»): ключ, хеш-функция, корзина, коллизия. Этот урок идёт глубже и не повторяет основы.
- Вы знаете трейты и обобщения с ограничениями ([Урок 05.1](../05-rust-intermediate/01-generics.md), [Урок 05.2](../05-rust-intermediate/02-traits.md)).

## Новые слова

| Слово | Значение |
|---|---|
| **хеш-функция** (hash function) | Функция, которая превращает ключ любого размера в число фиксированного размера (*хеш*). Один и тот же ключ всегда даёт один и тот же хеш. |
| **`Hash` / `Hasher`** | Трейт Rust для «можно подать в хеш-функцию» / объект, который получает байты и выдаёт число (`finish()`). |
| **корзина** (bucket) | Одна ячейка таблицы, где лежат записи, на которые указывает их хеш. |
| **цепочки** (chaining) | Стратегия при коллизиях: в каждой корзине небольшой список (цепочка) всех записей, попавших в неё. |
| **открытая адресация** (open addressing) | Стратегия при коллизиях: все записи лежат в самой таблице; при коллизии смотрят другую ячейку (например, следующую). |
| **пробирование** (probing) | Просмотр следующей ячейки-кандидата при открытой адресации. *Линейное пробирование* перебирает ячейки одну за другой. |
| **коэффициент заполнения** (load factor) | Число записей, делённое на число корзин. У таблицы с 6 записями и 8 корзинами коэффициент 0,75. |
| **перехеширование** (rehashing) | Построение более крупной таблицы и перенос каждой записи в её новую корзину (хеш тот же, но остаток от деления другой). |
| **лавинный эффект** (avalanche) | Свойство хорошей хеш-функции: изменение одной буквы ключа меняет весь хеш, поэтому похожие ключи оказываются далеко друг от друга. |
| **надгробие** (tombstone) | Метка, оставляемая в ячейке таблицы с открытой адресацией после удаления, чтобы поиск не останавливался слишком рано. |
| **хеш-флудинг** (hash flooding) | Атака, подбирающая много ключей с одинаковым хешем, чтобы замедлить таблицу (Урок 04.9). |

## 1. Повторение и контракт

Из Урока 04.9: хеш-таблица превращает ключ в число хеш-функцией, по числу выбирает **корзину** и смотрит только в ней. Два ключа в
одной корзине — это **коллизия**. Этот урок отвечает на вопросы, которые остались открытыми: *что делает хеш-функцию хорошей, что
именно происходит при коллизии и когда хеш-таблица становится медленной?*

В Rust хеш-функция разделена на два трейта:

- `Hash` реализует **тип ключа**. Его метод `hash(&self, state: &mut H)` подаёт части ключа в `Hasher`. Почти всегда вы получаете
  его через `#[derive(Hash)]`.
- `Hasher` — это «машина», которая перемешивает полученные байты. После того как ключ подан, `finish()` возвращает хеш `u64`.

`examples/e06_hashers.rs` показывает поведение:

```rust
use std::collections::hash_map::{DefaultHasher, RandomState};
use std::hash::{BuildHasher, Hash, Hasher};

fn main() {
    // DefaultHasher::new() always starts the same way
    let mut a = DefaultHasher::new();
    "hello".hash(&mut a);
    let mut b = DefaultHasher::new();
    "hello".hash(&mut b);
    println!("same input, same hasher : {} {}", a.finish(), b.finish());

    let mut c = DefaultHasher::new();
    "hellp".hash(&mut c);
    println!("one letter changed       : {}", c.finish());

    // RandomState is what HashMap uses: every new state has its own secret keys
    let first = RandomState::new();
    let second = RandomState::new();
    println!("RandomState #1 for hello : {}", first.hash_one("hello"));
    println!("RandomState #1 again     : {}", first.hash_one("hello"));
    println!("RandomState #2 for hello : {}", second.hash_one("hello"));
}
```

```text
same input, same hasher : 16156531084128653017 16156531084128653017
one letter changed       : 15091304090336206315
RandomState #1 for hello : 13061328524403966515
RandomState #1 again     : 13061328524403966515
RandomState #2 for hello : 11387501393696546303
```

(Точные числа на вашем компьютере и в других версиях Rust могут отличаться; важно, какие числа равны.)

**Что происходит:**

1. Один и тот же текст, поданный в два новых `DefaultHasher::new()`, даёт **один и тот же хеш**. Хеш-функция должна быть чистой
   функцией ключа.
2. Изменение одной буквы даёт **совершенно другое** число. Хорошая хеш-функция разбрасывает похожие ключи по всему диапазону. Это
   называется *лавинным эффектом*.
3. `RandomState` — то, что `HashMap` использует по умолчанию. У каждого нового `RandomState` свои секретные ключи, поэтому две
   таблицы хешируют `"hello"` по-разному, а одна таблица — всегда одинаково. Атакующий, который не видит секрета, не может заранее
   подобрать много ключей с одним хешем: это защита от хеш-флудинга из Урока 04.9. Цена — немного скорости.

**Контракт.** Если `a == b`, то `hash(a)` обязан быть равен `hash(b)`. (Обратное не требуется: разные ключи могут иметь одинаковый
хеш — это коллизия.) `derive(Hash, PartialEq, Eq)` вместе сохраняет контракт. Нарушьте его — и поиск перестанет работать, как
показывает раздел 6.

## 2. Коллизии: две стратегии

Хеш — это 64-битное число, а в таблице всего несколько корзин, поэтому индекс равен `hash % bucket_count`. При 8 корзинах и 100
ключах коллизий много. Какой бы ни была стратегия, хеш-таблица должна с ними справляться.

**Цепочки** (их использует наша реализация): в каждой корзине лежит список записей.

```text
 bucket 0: ─▶ ("bob", 7)
 bucket 1: (empty)
 bucket 2: ─▶ ("alice", 1) ─▶ ("eve", 5)       <- collision: two keys, one bucket
 bucket 3: ─▶ ("dan", 9)
```

Поиск `"eve"`: хеш, остаток, переход в корзину 2, сравнение `"eve"` с каждой записью цепочки, пока одна не окажется равной.
Стоимость — длина цепочки.

**Открытая адресация**: списков нет. Каждая запись лежит в ячейке самой таблицы. Если ячейка занята, пробуем следующую
(**линейное пробирование**):

```text
 slot:    0       1        2        3       4       5
        [bob ] [      ] [alice ] [eve  ] [dan  ] [      ]
 "eve" hashes to slot 2, which holds alice  -> probe slot 3: free -> store it there.
 Lookup "eve": start at slot 2 (alice, not equal) -> slot 3 (eve, equal) -> found.
 Lookup of a missing key stops at the first empty slot.
```

Открытая адресация держит всё в одном массиве, что удобно для кэша процессора (Урок 07.4). Она требует осторожности: при удалении
записи нужно оставить *надгробие*, иначе последующий поиск остановится слишком рано на дырке. Стандартный `HashMap` использует
открытую адресацию в усовершенствованном виде, называемом *Swiss table* (библиотека `hashbrown` внутри `std`), который проверяет
много ячеек сразу специальными инструкциями процессора. Цепочки проще написать правильно, поэтому строим их.

## 3. Коэффициент заполнения и перехеширование

Какой длины цепочки? В среднем — равной **коэффициенту заполнения**: число записей, делённое на число корзин. При 1 000 записей в
2 048 корзинах он около 0,49, так что в большинстве цепочек 0 или 1 запись, и поиск делает постоянный объём работы. Если бы число
корзин никогда не росло, коэффициент рос бы вместе с `n`, цепочки удлинялись бы, и поиск стал бы `O(n)`.

Поэтому таблица **растёт**, когда коэффициент превышает предел (мы берём 0,75): выделяет вдвое больше корзин и **перехеширует**:
каждая запись переходит в корзину, заданную тем же хешем с новым остатком. Перехеширование стоит `O(n)`, но случается после
удвоения, так что стоимость распределяется на много вставок, как с вектором в Уроке 07.3: вставка — **амортизированная `O(1)`**.
Поэтому таблицу стоит создавать через `HashMap::with_capacity(n)`, когда вы знаете, сколько элементов добавите.

## 4. Хеш-таблица с цепочками

Создайте `src/hash_table.rs` и добавьте `pub mod hash_table;` в `src/lib.rs`. Тип, конструктор и сердце структуры: превращение
ключа в индекс корзины.

```rust
//! Lesson 07.6: a hash table with separate chaining.

use std::collections::hash_map::DefaultHasher;
use std::hash::{Hash, Hasher};

/// A hash map. `buckets[i]` is the chain (a small vector) of all entries whose
/// key hashes to bucket `i`.
pub struct ChainedMap<K, V> {
    buckets: Vec<Vec<(K, V)>>,
    len: usize,
}

impl<K: Hash + Eq, V> ChainedMap<K, V> {
    pub fn new() -> ChainedMap<K, V> {
        ChainedMap {
            buckets: Self::empty_buckets(4),
            len: 0,
        }
    }

    fn empty_buckets(count: usize) -> Vec<Vec<(K, V)>> {
        (0..count).map(|_| Vec::new()).collect()
    }

    /// The bucket for a key: hash the key to a number, then take the remainder.
    fn bucket_index(&self, key: &K) -> usize {
        let mut hasher = DefaultHasher::new();
        key.hash(&mut hasher);
        (hasher.finish() % self.buckets.len() as u64) as usize
    }
```

**Что происходит:**

1. `buckets` — вектор цепочек. Каждая цепочка — это `Vec<(K, V)>`. `new` начинает с 4 пустых корзин.
2. `bucket_index` создаёт хешер, подаёт в него ключ через `key.hash(&mut hasher)`, берёт `u64` из `finish()` и сводит его к номеру
   корзины через `%`. `DefaultHasher::new()` детерминирован, что делает наши тесты повторяемыми. (В боевой таблице использовали бы
   случайный ключ, как у `RandomState`.)
3. Ограничения `K: Hash + Eq` говорят, что ключ должен уметь.

Методы доступа, включая два для изучения таблицы:

```rust
    pub fn len(&self) -> usize {
        self.len
    }

    pub fn is_empty(&self) -> bool {
        self.len == 0
    }

    pub fn bucket_count(&self) -> usize {
        self.buckets.len()
    }

    /// Entries per bucket, on average.
    pub fn load_factor(&self) -> f64 {
        self.len as f64 / self.buckets.len() as f64
    }

    /// The length of the longest chain: the worst case of one lookup.
    pub fn longest_chain(&self) -> usize {
        self.buckets
            .iter()
            .map(|chain| chain.len())
            .max()
            .unwrap_or(0)
    }
```

Три операции:

```rust
    /// Adds or replaces an entry; returns the old value if the key was there.
    pub fn insert(&mut self, key: K, value: V) -> Option<V> {
        let index = self.bucket_index(&key);
        for entry in &mut self.buckets[index] {
            if entry.0 == key {
                return Some(std::mem::replace(&mut entry.1, value));
            }
        }
        // a new key: keep the load factor at 0.75 or less
        if (self.len + 1) * 4 > self.buckets.len() * 3 {
            self.grow();
        }
        let index = self.bucket_index(&key);
        self.buckets[index].push((key, value));
        self.len += 1;
        None
    }

    pub fn get(&self, key: &K) -> Option<&V> {
        let index = self.bucket_index(key);
        self.buckets[index]
            .iter()
            .find(|entry| entry.0 == *key)
            .map(|entry| &entry.1)
    }

    pub fn remove(&mut self, key: &K) -> Option<V> {
        let index = self.bucket_index(key);
        let position = self.buckets[index]
            .iter()
            .position(|entry| entry.0 == *key)?;
        self.len -= 1;
        Some(self.buckets[index].swap_remove(position).1)
    }
```

**Что происходит:**

1. `insert` сначала смотрит в цепочку корзины ключа. Если ключ уже там, `std::mem::replace` подставляет новое значение и
   возвращает старое (Урок 03.5). Если ключ новый и таблица превысила бы коэффициент заполнения 0,75
   (`(len + 1) * 4 > buckets * 3`, целочисленная арифметика без чисел с плавающей точкой), она сначала растёт. Затем корзина
   вычисляется заново (число корзин изменилось) и запись добавляется.
2. `get` идёт в одну корзину и просматривает её цепочку через `find`. `remove` находит позицию и использует `swap_remove`, который
   `O(1)` и здесь допустим, потому что порядок внутри цепочки не важен.
3. При коротких цепочках все три операции — `O(1)`. При цепочке длины `L` они стоят `O(L)`.

Шаг роста и реализация `Default`:

```rust
    /// Doubles the number of buckets and puts every entry in its new bucket (rehashing): O(n).
    fn grow(&mut self) {
        let new_count = self.buckets.len() * 2;
        let old = std::mem::replace(&mut self.buckets, Self::empty_buckets(new_count));
        for (key, value) in old.into_iter().flatten() {
            let index = self.bucket_index(&key);
            self.buckets[index].push((key, value));
        }
    }
}

impl<K: Hash + Eq, V> Default for ChainedMap<K, V> {
    fn default() -> ChainedMap<K, V> {
        ChainedMap::new()
    }
}
```

**Что происходит:** `grow` подставляет новый пустой набор корзин вдвое большего размера (`mem::replace` возвращает старые), затем
обходит каждую старую запись (`into_iter().flatten()` по очереди проходит все цепочки, забирая владение) и кладёт её в новую
корзину. Ни один ключ не клонируется: записи перемещаются.

### Плохая хеш-функция

Что, если все ключи хешируются в одно и то же число? `examples/e06_chains.rs` сравнивает обычный ключ `u32` с `BadKey`, чья
реализация `Hash` ничего не подаёт в хешер:

```rust
use p07_algorithms::hash_table::ChainedMap;
use std::hash::{Hash, Hasher};

/// A key whose hash is always the same.
#[derive(PartialEq, Eq)]
struct BadKey(u32);

impl Hash for BadKey {
    fn hash<H: Hasher>(&self, _state: &mut H) {}
}

fn main() {
    let mut good = ChainedMap::new();
    let mut bad = ChainedMap::new();
    for i in 0..1000u32 {
        good.insert(i, ());
        bad.insert(BadKey(i), ());
    }
    println!(
        "good hash: {} entries, {} buckets, load factor {:.2}, longest chain {}",
        good.len(),
        good.bucket_count(),
        good.load_factor(),
        good.longest_chain()
    );
    println!(
        "bad hash : {} entries, {} buckets, load factor {:.2}, longest chain {}",
        bad.len(),
        bad.bucket_count(),
        bad.load_factor(),
        bad.longest_chain()
    );
}
```

```text
good hash: 1000 entries, 2048 buckets, load factor 0.49, longest chain 4
bad hash : 1000 entries, 2048 buckets, load factor 0.49, longest chain 1000
```

При хорошем хеше 1 000 записей разложены по 2 048 корзинам, а **самая длинная цепочка — 4 записи**. При плохом хеше всё лежит в
**одной цепочке из 1 000 записей**: таблица — это медленный связный список, а каждый поиск — `O(n)`. Таблица по-прежнему корректна.
Она лишь медленная. Именно этого хочет атакующий, устраивающий хеш-флудинг.

### Тесты

Тесты сравнивают с `std::collections::HashMap` на случайных операциях, проверяют коэффициент заполнения, пока таблица растёт, и
проверяют цепочку при плохом хеше. Добавьте внизу `src/hash_table.rs`:

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use crate::testutil::Rng;
    use std::collections::HashMap;

    /// A key whose hash is always the same: every key lands in one bucket.
    #[derive(PartialEq, Eq)]
    struct BadKey(u32);

    impl Hash for BadKey {
        fn hash<H: Hasher>(&self, _state: &mut H) {}
    }

    #[test]
    fn behaves_like_hash_map_on_random_operations() {
        let mut rng = Rng::new(17);
        let mut mine: ChainedMap<u64, u64> = ChainedMap::new();
        let mut reference: HashMap<u64, u64> = HashMap::new();
        for step in 0..20_000 {
            let key = rng.next_u64() % 500;
            match rng.below(3) {
                0 => {
                    let value = rng.next_u64();
                    assert_eq!(
                        mine.insert(key, value),
                        reference.insert(key, value),
                        "step {step}"
                    );
                }
                1 => assert_eq!(mine.remove(&key), reference.remove(&key), "step {step}"),
                _ => assert_eq!(mine.get(&key), reference.get(&key), "step {step}"),
            }
            assert_eq!(mine.len(), reference.len(), "step {step}");
        }
    }

    #[test]
    fn table_grows_and_keeps_the_load_factor_low() {
        let mut map = ChainedMap::new();
        for i in 0..1000u32 {
            map.insert(i, i);
            assert!(map.load_factor() <= 0.75);
        }
        assert_eq!(map.bucket_count(), 2048);
        assert!((0..1000).all(|i| map.get(&i) == Some(&i)));
    }

    #[test]
    fn a_bad_hash_puts_everything_in_one_chain() {
        let mut map = ChainedMap::new();
        for i in 0..100 {
            map.insert(BadKey(i), i);
        }
        assert_eq!(map.longest_chain(), 100);
        assert_eq!(map.get(&BadKey(57)), Some(&57)); // still correct, only slow
    }
}
```

```text
running 3 tests
test hash_table::tests::a_bad_hash_puts_everything_in_one_chain ... ok
test hash_table::tests::table_grows_and_keeps_the_load_factor_low ... ok
test hash_table::tests::behaves_like_hash_map_on_random_operations ... ok

test result: ok. 3 passed; 0 failed; 0 ignored; 0 measured; 17 filtered out; finished in 0.01s
```

## 5. Измеряем

`benches/hash_tables.rs` ищет 2 000 существующих ключей в трёх таблицах: нашей, нашей с плохим ключом и
`std::collections::HashMap`. Зарегистрируйте его в `Cargo.toml` (как в Уроке 07.2):

```toml
[[bench]]
name = "hash_tables"
harness = false
```

```rust
use criterion::{Criterion, criterion_group, criterion_main};
use p07_algorithms::hash_table::ChainedMap;
use std::collections::HashMap;
use std::hash::{Hash, Hasher};
use std::hint::black_box;

const N: u64 = 2_000;

/// A key with a terrible hash function: every key gets the same hash.
#[derive(PartialEq, Eq)]
struct BadKey(u64);

impl Hash for BadKey {
    fn hash<H: Hasher>(&self, _state: &mut H) {}
}

fn bench_lookups(c: &mut Criterion) {
    let mut good = ChainedMap::new();
    let mut bad = ChainedMap::new();
    let mut std_map = HashMap::new();
    for i in 0..N {
        good.insert(i, i);
        bad.insert(BadKey(i), i);
        std_map.insert(i, i);
    }

    let mut group = c.benchmark_group("2_000 lookups");
    group.bench_function("our ChainedMap", |b| {
        b.iter(|| {
            (0..N)
                .map(|i| *good.get(&black_box(i)).unwrap())
                .sum::<u64>()
        })
    });
    group.bench_function("our ChainedMap, bad hash", |b| {
        b.iter(|| {
            (0..N)
                .map(|i| *bad.get(&BadKey(black_box(i))).unwrap())
                .sum::<u64>()
        })
    });
    group.bench_function("std HashMap", |b| {
        b.iter(|| {
            (0..N)
                .map(|i| *std_map.get(&black_box(i)).unwrap())
                .sum::<u64>()
        })
    });
    group.finish();
}

criterion_group!(benches, bench_lookups);
criterion_main!(benches);
```

Запустите `cargo bench --bench hash_tables -- --warm-up-time 1 --measurement-time 3`:

```text
2_000 lookups/our ChainedMap
                        time:   [15.119 µs 15.221 µs 15.347 µs]
2_000 lookups/our ChainedMap, bad hash
                        time:   [309.49 µs 314.92 µs 320.55 µs]
2_000 lookups/std HashMap
                        time:   [11.896 µs 11.940 µs 11.993 µs]
```

(Apple M4 Pro; **ваши числа будут другими**.)

**Как это читать:**

1. Наша таблица делает 2 000 поисков за 15 мкс: около **7,6 нс на поиск**. Стандартная таблица — 12 мкс, около 6 нс. Наши простые
   цепочки примерно на 25 % медленнее очень тщательно настроенной стандартной таблицы.
2. При плохом хеше те же 2 000 поисков занимают 315 мкс, то есть **в 21 раз медленнее**, и разрыв растёт вместе с `n`: каждый поиск
   просматривает цепочку до 2 000 записей (было `O(1)`, стало `O(n)`).
3. Хеш-функция — половина хеш-таблицы. Таблица с плохим хешем — это список в маскировке.

## 6. Нарушенный контракт

Вот ошибка, о которой предупреждает контракт. `Name` сравнивается без учёта регистра, но `Hash` выводит из текста как есть:

```rust
use p07_algorithms::hash_table::ChainedMap;

/// A name that ignores upper and lower case when compared...
#[derive(Hash)]
struct Name(String);

impl PartialEq for Name {
    fn eq(&self, other: &Name) -> bool {
        self.0.eq_ignore_ascii_case(&other.0)
    }
}

impl Eq for Name {}

fn main() {
    let mut map = ChainedMap::new();
    for word in [
        "Rust", "Cargo", "Crate", "Trait", "Slice", "Macro", "Enum", "Box",
    ] {
        map.insert(Name(word.to_string()), word.len());
    }
    let rust_upper = Name("Rust".to_string());
    let rust_lower = Name("rust".to_string());
    println!("Rust == rust : {}", rust_upper == rust_lower);
    println!("get(Rust)    : {:?}", map.get(&rust_upper));
    println!("get(rust)    : {:?}", map.get(&rust_lower));
}
```

```text
Rust == rust : true
get(Rust)    : Some(4)
get(rust)    : None
```

`Rust == rust` — `true`, но `get(rust)` ничего не находит: `"Rust"` и `"rust"` хешируются в разные числа, поэтому поиск идёт в другую
корзину. Таблица не упала. Она тихо дала неверный ответ. Исправление: согласовать `Hash` с `Eq`, например хешировать текст в нижнем
регистре: `self.0.to_ascii_lowercase().hash(state)` внутри написанного вручную `impl Hash`.

## 7. Что из стандартной библиотеки когда использовать

| Что нужно | Что использовать | Стоимость |
|---|---|---|
| поиск по ключу, порядок не важен | `HashMap<K, V>` | `O(1)` в среднем |
| «видел ли я это значение?» | `HashSet<T>` | `O(1)` в среднем |
| отсортированные ключи, диапазоны (`range(a..b)`), первый и последний ключ | `BTreeMap<K, V>` / `BTreeSet<T>` | `O(log n)`; Урок 07.11 знакомит с деревьями поиска |
| немного ключей (меньше 10–20) | `Vec<(K, V)>` и линейный просмотр | `O(n)`, но часто быстрее всего |
| ключи от недоверенных пользователей | `HashMap` со стандартным `RandomState` | устойчив к хеш-флудингу |

Два напоминания о настоящем `HashMap`: его `get` принимает любую заимствованную форму ключа (ключ `String` можно искать по `&str`
благодаря трейту `Borrow`), чего наш упрощённый `get(&K)` не умеет. А API `entry` (Application Programming Interface, программный интерфейс: набор функций и типов, которые предлагает библиотека) из Урока 04.9 делает «найти, затем вставить» с
одним вычислением хеша.

## Частые ошибки

**1. Нарушение контракта `Hash`/`Eq`** (раздел 6). Выводите оба вместе или пишите оба вручную по одним и тем же полям.

**2. Хеширование изменяемого поля.** Хеш ключа не должен меняться, пока ключ лежит в таблице. Ключ с внутренней изменяемостью
(`RefCell`, Урок 06.4), у которого меняется хешируемое поле, делает запись недоступной. Используйте обычные неизменяемые ключи.

**3. Ожидание порядка.** Порядок обхода `HashMap` случаен и меняется от запуска к запуску. Если нужен порядок, используйте
`BTreeMap` или сортируйте ключи.

## Попробуйте сами

1. Наберите `hash_table.rs` сами. Запустите `cargo test hash_table`, `cargo clippy` и `cargo fmt`.
2. Измените предел роста с 0,75 на 3,0 (`(self.len + 1) > self.buckets.len() * 3`). Запустите `examples/e06_chains.rs`: чему равны
   коэффициент заполнения и самая длинная цепочка теперь? Запустите бенчмарк: стала ли таблица медленнее? Верните как было.
3. Запустите `examples/e06_hashers.rs` дважды. Какие строки меняются от запуска к запуску, а какие нет? Почему?
4. Зафиксируйте: `git add code/p07-algorithms && git commit -m "feat(p07): hash map with chaining"`.

## Упражнения

### Упражнение 1 — Коэффициент заполнения (легко)

В таблице 16 корзин, и она растёт, когда коэффициент заполнения превысил бы 0,75. (а) Сколько записей она вмещает до первого роста?
(б) Сколько корзин у неё после роста и чему равен коэффициент заполнения сразу после роста? (в) Чему равно время одной `insert` в
худшем случае и амортизированно, и почему?

<details>
<summary>Подсказка</summary>

Предел равен `buckets * 0.75`. Перехеширование перемещает каждую запись, но число записей не меняется.

</details>

<details>
<summary>Ответ</summary>

(а) `16 * 0.75 = 12` записей; 13-я вставка запускает рост. (б) 32 корзины; 13 записей дают коэффициент `13 / 32 ≈ 0,41`.
(в) В худшем случае `O(n)` для той одной вставки, что перехеширует (а при ужасном хеше — тоже `O(n)` из-за длинной цепочки);
амортизированно `O(1)`, потому что таблица удваивается, и каждая запись перехешируется лишь несколько раз за много вставок.

</details>

### Упражнение 2 — Подсчёт слов (средне)

Добавьте `get_mut(&mut self, key: &K) -> Option<&mut V>` в `ChainedMap` и используйте её в
`count_words(text: &str) -> ChainedMap<String, usize>`, которая считает, сколько раз встречается каждое слово. Проверьте тестом
против `HashMap`, построенного через API `entry`. Назовите время `count_words` для текста из `n` слов.

<details>
<summary>Подсказка</summary>

`get_mut` — это `get` с `iter_mut()` вместо `iter()`. В `count_words`: `match counts.get_mut(&word)`; в ветке `Some` увеличьте
счётчик, в ветке `None` вставьте 1.

</details>

<details>
<summary>Ответ</summary>

Внутри блока `impl<K: Hash + Eq, V> ChainedMap<K, V>`:

```rust
    /// A mutable reference to the value of `key`, so it can be changed in place.
    pub fn get_mut(&mut self, key: &K) -> Option<&mut V> {
        let index = self.bucket_index(key);
        self.buckets[index]
            .iter_mut()
            .find(|entry| entry.0 == *key)
            .map(|entry| &mut entry.1)
    }
```

```rust
/// Counts how often each word appears.
pub fn count_words(text: &str) -> ChainedMap<String, usize> {
    let mut counts = ChainedMap::new();
    for word in text.split_whitespace() {
        match counts.get_mut(&word.to_string()) {
            Some(count) => *count += 1,
            None => {
                counts.insert(word.to_string(), 1);
            }
        }
    }
    counts
}
```

Тест (внутри `mod tests`):

```rust
    #[test]
    fn word_counts_match_hash_map() {
        let text = "the cat and the dog and the bird";
        let mine = count_words(text);
        let mut reference: HashMap<String, usize> = HashMap::new();
        for word in text.split_whitespace() {
            *reference.entry(word.to_string()).or_insert(0) += 1;
        }
        assert_eq!(mine.len(), reference.len());
        for (word, count) in &reference {
            assert_eq!(mine.get(word), Some(count));
        }
    }
```

Время `O(n)` в среднем для `n` слов (один поиск и не более одной вставки на слово, каждая `O(1)` в среднем) плюс рост, который
амортизируется. Дополнительная память — `O(d)` для `d` различных слов.

</details>

### Упражнение 3 — Множество из таблицы (сложно)

Постройте `ChainedSet<K>` поверх `ChainedMap<K, ()>` с `insert(key) -> bool` (`true`, если ключ был новым), `contains`,
`remove -> bool`, `len` и `is_empty`. Проверьте тестом против `std::collections::HashSet` на 5 000 случайных операций. Какова
стоимость операций?

<details>
<summary>Подсказка</summary>

`ChainedMap::insert` возвращает старое значение (`Option<()>`): ключ был новым ровно тогда, когда это `None`. Множество — это
таблица, значения которой не несут информации.

</details>

<details>
<summary>Ответ</summary>

```rust
/// A set of keys, built on the map with empty values.
pub struct ChainedSet<K> {
    map: ChainedMap<K, ()>,
}

impl<K: Hash + Eq> ChainedSet<K> {
    pub fn new() -> ChainedSet<K> {
        ChainedSet {
            map: ChainedMap::new(),
        }
    }

    /// Returns `true` if the key was not in the set before.
    pub fn insert(&mut self, key: K) -> bool {
        self.map.insert(key, ()).is_none()
    }

    pub fn contains(&self, key: &K) -> bool {
        self.map.get(key).is_some()
    }

    pub fn remove(&mut self, key: &K) -> bool {
        self.map.remove(key).is_some()
    }

    pub fn len(&self) -> usize {
        self.map.len()
    }

    pub fn is_empty(&self) -> bool {
        self.map.is_empty()
    }
}

impl<K: Hash + Eq> Default for ChainedSet<K> {
    fn default() -> ChainedSet<K> {
        ChainedSet::new()
    }
}
```

```rust
    #[test]
    fn set_matches_hash_set() {
        use std::collections::HashSet;
        let mut rng = Rng::new(21);
        let mut mine: ChainedSet<u64> = ChainedSet::new();
        let mut reference: HashSet<u64> = HashSet::new();
        for step in 0..5000 {
            let key = rng.next_u64() % 100;
            match rng.below(3) {
                0 => assert_eq!(mine.insert(key), reference.insert(key), "step {step}"),
                1 => assert_eq!(mine.remove(&key), reference.remove(&key), "step {step}"),
                _ => assert_eq!(mine.contains(&key), reference.contains(&key), "step {step}"),
            }
            assert_eq!(mine.len(), reference.len());
        }
        assert_eq!(mine.is_empty(), reference.is_empty());
    }
```

Каждая операция делегируется таблице: `O(1)` в среднем, `insert` — амортизированно. Так же стандартный `HashSet<T>` построен
поверх `HashMap<T, ()>`.

</details>

## Вопросы для самопроверки

1. Что говорит контракт между `Hash` и `Eq`?

<details>
<summary>Ответ</summary>

Если два ключа равны, их хеши обязаны быть равны. (Разные ключи могут иметь общий хеш — это коллизия.)

</details>

2. Почему поиск в хеш-таблице — `O(1)` в среднем, но `O(n)` в худшем случае?

<details>
<summary>Ответ</summary>

При хорошем хеше и ограниченном коэффициенте заполнения цепочки короткие (в среднем постоянной длины). Если много ключей попадает в
одну корзину (плохой хеш или атакующий), одна цепочка хранит почти все записи, и поиск просматривает их все.

</details>

3. Что такое коэффициент заполнения и зачем таблица растёт, когда он превышает предел?

<details>
<summary>Ответ</summary>

Записи, делённые на корзины. Это средняя длина цепочки. Рост (и перехеширование) держит его ниже предела, поэтому цепочки остаются
короткими, а операции — `O(1)`.

</details>

4. Почему `HashMap<String, V>` можно искать по `&str`, а наш `ChainedMap::get` — нельзя?

<details>
<summary>Ответ</summary>

Стандартный `get` обобщён по заимствованной форме `Q` ключа (`K: Borrow<Q>`), а `str` хешируется и сравнивается так же, как
`String`. Наш `get` принимает только `&K`, поэтому ему нужен `&String`.

</details>

5. Когда `BTreeMap` может быть лучшим выбором, чем `HashMap`?

<details>
<summary>Ответ</summary>

Когда нужны ключи в отсортированном порядке, запросы по диапазону или гарантированный худший случай `O(log n)`, не зависящий от
хеширования.

</details>

## Итоги

- Хеш-функция отображает ключ в число; таблица отображает число в корзину через `%`. Равные ключи обязаны иметь равные хеши
  (контракт `Hash`/`Eq`).
- Коллизии неизбежны. Цепочки хранят список в каждой корзине; открытая адресация ищет другую ячейку (так делает стандартный
  `HashMap` в виде Swiss table).
- Коэффициент заполнения — средняя длина цепочки. Удержание его ниже предела удвоением числа корзин и перехешированием делает
  `insert` амортизированной `O(1)`, а `get` — `O(1)` в среднем.
- Плохой хеш или хеш, подобранный атакующим, превращает таблицу в список: `O(n)` на операцию.
- Проверяйте свою таблицу против `HashMap` на случайных операциях и проверяйте саму структуру (коэффициент заполнения, самая
  длинная цепочка).
- В реальном коде используйте `HashMap`/`HashSet`; `BTreeMap` — когда нужен порядок.

## Источники и что почитать

- [std::collections::HashMap](https://doc.rust-lang.org/std/collections/struct.HashMap.html) — стандартная хеш-таблица, с кратким описанием её устройства, хеширования и API `entry`.
- [std::hash — Hash, Hasher and BuildHasher](https://doc.rust-lang.org/std/hash/index.html) — трейты хеширования и контракт между `Hash` и `Eq`.
- [hashbrown](https://github.com/rust-lang/hashbrown) — реализация Swiss table, на которой построен `std::collections::HashMap`.
- [std::borrow::Borrow](https://doc.rust-lang.org/std/borrow/trait.Borrow.html) — почему ключ `String` можно искать по `&str`.
- [std::collections::BTreeMap](https://doc.rust-lang.org/std/collections/struct.BTreeMap.html) — упорядоченная таблица.

---

[🏠 Начало](../START-HERE.md) · [← 07.5 Стеки и очереди](05-stacks-and-queues.md) · [📋 Этап 07](README.md) · [07.7 Рекурсия и стек вызовов →](07-recursion.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
