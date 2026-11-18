<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 17.7 LSM-деревья](07-lsm-trees.md) · [📋 Этап 17](README.md) · [17.9 Страничный кэш и пул буферов →](09-page-cache-and-buffer-pool.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/17-storage-and-databases/08-compaction-and-bloom-filters.md) <!-- nav -->

# Урок 17.8 — Компакция и фильтры Блума

> **Этап 17 — Как устроены хранилища и базы данных** · Урок 8 из 15 · ⏱ около 3 часов (2 занятия)

## Что вы узнаете

- Почему LSM-дерево должно делать компакцию своих SSTable и что она возвращает: более быстрые чтения и свободное место
  на диске.
- Как слить много отсортированных последовательностей в одну с помощью кучи (k-путевое слияние), сохранив только самую
  новую версию каждого ключа.
- Две классические стратегии компакции, по размерам (size-tiered) и по уровням (leveled), и чего стоит каждая.
- Когда надгробие можно выбросить и почему это вопрос устойчивости к сбоям.
- Как фильтр Блума отвечает «точно нет» без чтения таблицы и как выбрать его размер.

## Перед началом

- Вы прошли [Урок 17.7 — LSM-деревья](07-lsm-trees.md): `src/sstable.rs` и `src/lsm.rs` существуют, а демо показало
  26 чтений блоков ради одного отсутствующего ключа.
- Вы знаете кучи и `BinaryHeap` из [Урока 07.12 — Кучи и очереди с приоритетом](../07-algorithms-and-data-structures/12-heaps-and-priority-queues.md),
  слияние отсортированных списков из [Урока 07.9](../07-algorithms-and-data-structures/09-merge-sort-and-quicksort.md),
  хеш-функции из [Урока 07.6](../07-algorithms-and-data-structures/06-hash-tables.md) и битовые маски из
  [Урока 12.4](../12-computer-systems/04-bitwise-operations.md).

## Новые слова

| Слово | Значение |
|---|---|
| **отсортированная последовательность** (sorted run) | Последовательность пар «ключ-значение» по порядку ключей, например содержимое одной SSTable. |
| **k-путевое слияние** (k-way merge) | Слияние `k` отсортированных последовательностей в одну за один проход, с кучей для поиска наименьшего следующего ключа. |
| **компакция по размерам** (size-tiered compaction) | Стратегия, которая сливает несколько таблиц примерно одного размера в одну большую. |
| **компакция по уровням** (leveled compaction) | Стратегия, которая держит таблицы на уровнях L0, L1, L2, …; каждый уровень примерно в десять раз больше предыдущего, а внутри уровня таблицы не пересекаются. |
| **усиление по месту** (space amplification) | Использование большего места на диске, чем нужно живым данным, из-за старых версий и надгробий. |
| **фильтр Блума** (Bloom filter) | Небольшой битовый массив, который отвечает на вопрос «есть ли этот ключ в множестве?» словами «точно нет» или «возможно». Для добавленного ключа он никогда не говорит «нет». |
| **ложное срабатывание** (false positive) | «Возможно» от фильтра Блума для ключа, который никогда не добавляли. Стоит одного бесполезного чтения, но никогда — неверного ответа. |
| **манифест** (manifest) | Небольшой файл, в котором движок хранения записывает, какие файлы таблиц сейчас действительны. Его атомарная замена — способ изменить набор файлов за один шаг. |

## 1. Два счёта к оплате

LSM-дерево из Урока 17.7 пишет быстро, но оставляет два счёта:

1. **Усиление чтения.** Каждый сброс добавляет таблицу, и поиск может прочитать блок из каждой: 26 чтений ради одного
   отсутствующего ключа в демо.
2. **Усиление по месту.** У перезаписанного ключа старые версии остаются в старых таблицах, а у удалённого — старое
   значение *и* надгробие. Файлы хранят гораздо больше, чем живые данные.

**Компакция** оплачивает первый счёт и бо́льшую часть второго: она сливает несколько таблиц в одну, оставляет только самую
новую версию каждого ключа и удаляет старые файлы. **Фильтры Блума** оплачивают остаток первого счёта: они позволяют поиску
пропустить таблицу, не читая её.

## 2. Слияние отсортированных последовательностей

Каждая SSTable отсортирована, поэтому слияние таблиц похоже на слияние отсортированных половин в сортировке слиянием
(Урок 07.9), только входов не два, а `k`. Держите следующий ключ каждого входа в **min-куче** (Урок 07.12): наверху
наименьший ключ. Снимите его, запишите в выход и положите следующий ключ того входа, откуда он пришёл. Каждый шаг стоит
`O(log k)`, а входы читаются строго по порядку, последовательно.

Новая только одна деталь: один и тот же ключ может встречаться в нескольких входах. Побеждать должна самая новая версия.
Поэтому куча упорядочивает сначала по ключу, а затем по **возрасту** (0 = самая новая последовательность), и самая новая
версия ключа всегда выходит первой. Следующие за ней старые версии того же ключа пропускаются.

```text
 run 0 (newest):  b=2   d=✝            heap pops: a=1 (run 1)
 run 1 (oldest):  a=1   b=1   d=1                 b=2 (run 0)   ← newest b
                                                  b=1 (run 1)   skip: older b
 output:          a=1   b=2   d=✝                 d=✝ (run 0)   ← tombstone wins
                                                  d=1 (run 1)   skip
```

Создайте `src/compaction.rs` и добавьте `pub mod compaction;` в `src/lib.rs`:

```rust
use crate::sstable::Value;
use std::cmp::Reverse;
use std::collections::BinaryHeap;

/// Merges sorted runs into one sorted run. `runs[0]` is the newest.
/// For a key that appears in several runs, only the newest version is kept.
/// With `drop_tombstones`, deleted keys disappear completely.
pub fn merge(runs: Vec<Vec<(Vec<u8>, Value)>>, drop_tombstones: bool) -> Vec<(Vec<u8>, Value)> {
    let mut iters: Vec<_> = runs.into_iter().map(|run| run.into_iter()).collect();
    let mut heap = BinaryHeap::new();
    for (age, iter) in iters.iter_mut().enumerate() {
        if let Some((key, value)) = iter.next() {
            heap.push(Reverse((key, age, value))); // smallest key first, then the newest run
        }
    }

    let mut out: Vec<(Vec<u8>, Value)> = Vec::new();
    while let Some(Reverse((key, age, value))) = heap.pop() {
        if let Some((next_key, next_value)) = iters[age].next() {
            heap.push(Reverse((next_key, age, next_value)));
        }
        if out.last().is_some_and(|(last, _)| *last == key) {
            continue; // an older version of a key we already have
        }
        out.push((key, value));
    }
    if drop_tombstones {
        out.retain(|(_, value)| value.is_some());
    }
    out
}

#[cfg(test)]
mod tests {
    use super::*;

    fn run(entries: &[(&str, Option<&str>)]) -> Vec<(Vec<u8>, Value)> {
        entries
            .iter()
            .map(|(k, v)| (k.as_bytes().to_vec(), v.map(|v| v.as_bytes().to_vec())))
            .collect()
    }

    #[test]
    fn newest_version_wins_and_output_is_sorted() {
        let newest = run(&[("b", Some("2")), ("d", None)]);
        let oldest = run(&[("a", Some("1")), ("b", Some("1")), ("d", Some("1"))]);
        let merged = merge(vec![newest, oldest], false);
        assert_eq!(
            merged,
            run(&[("a", Some("1")), ("b", Some("2")), ("d", None)])
        );
    }

    #[test]
    fn tombstones_can_be_dropped() {
        let newest = run(&[("d", None)]);
        let oldest = run(&[("a", Some("1")), ("d", Some("1"))]);
        assert_eq!(merge(vec![newest, oldest], true), run(&[("a", Some("1"))]));
    }
}
```

**Что происходит:**

1. `BinaryHeap` — это max-куча; обёртка элементов в `Reverse` превращает её в min-кучу (Урок 07.12).
2. Кортежи сравниваются поэлементно: сначала ключ, затем `age`. Поэтому при равных ключах первым выходит элемент из самой
   новой последовательности (с наименьшим `age`). Значение стоит третьим только для того, чтобы путешествовать вместе с
   ключом.
3. После каждого снятия в кучу кладётся следующий элемент той же последовательности. В куче никогда не больше `k`
   элементов, какой бы длины ни были последовательности.
4. `out.last()` — ключ, записанный непосредственно перед этим. Если снятый ключ тот же, это старая версия, и она
   пропускается.
5. Надгробия выбрасываются только в самом конце, через `retain`. Выбросить надгробие внутри цикла было бы ошибкой: тогда
   вместо него записалась бы следующая старая версия, и удалённый ключ вернулся бы.

> ⚡ **Производительность:** настоящая компакция работает потоком: читает каждый вход блок за блоком и пишет выход блок за
> блоком, так что ей нужна память на один блок на вход, а не на целые таблицы. Наш `merge` берёт целые `Vec`, чтобы остаться
> коротким; логика кучи та же.

## 3. Какие таблицы сливать: две стратегии

Если каждый раз сливать *все* таблицы, база данных переписывалась бы целиком снова и снова. Настоящие движки выбирают,
**какие** таблицы сливать, и классических стратегий две:

```text
 size-tiered (used by Cassandra)            leveled (LevelDB, RocksDB's default)
 ───────────────────────────────            ────────────────────────────────────
 4 small tables ──merge──► 1 medium         L0:  new flushed tables (may overlap)
 4 medium       ──merge──► 1 large          L1:  ~10 MB,  tables do not overlap
 4 large        ──merge──► 1 huge           L2:  ~100 MB, tables do not overlap
                                            L3:  ~1 GB …
 + little rewriting (low write amp.)        + a key is in at most one table per level
 − several tables of each size to search    − data is rewritten at every level
 − needs free space for the big merges        (higher write amplification)
```

- Компакция **по размерам** ждёт, пока накопится несколько таблиц примерно одного размера, и сливает их в одну побольше.
  Каждый кусок данных переписывается лишь несколько раз, поэтому запись остаётся дешёвой. Но в любой момент есть несколько
  таблиц каждого размера, так что чтение всё ещё проверяет несколько таблиц, а большое слияние временно требует столько же
  свободного места.
- Компакция **по уровням** держит фиксированную структуру. Внутри каждого уровня (кроме L0) таблицы покрывают **разные
  диапазоны ключей**, поэтому поиск проверяет не больше одной таблицы на уровень, а проверка диапазона ключей из упражнения 3
  Урока 17.7 пропускает все остальные. Цена: когда уровень `n` слишком велик, одна из его таблиц сливается с
  пересекающимися таблицами уровня `n + 1`, так что данные переписываются по разу на каждый уровень — больше **усиление
  записи**.

Снова компромисс: компакция по размерам выгоднее для записи, по уровням — для чтения и места. Наше хранилище использует
самую простую стратегию — **полную компакцию**, которая сливает все таблицы в одну; этого достаточно, чтобы увидеть эффект.

## 4. Когда надгробие может уйти?

Надгробие существует, чтобы скрывать старые значения своего ключа. Выбросить его можно только тогда, когда ниже него нигде
не может быть более старого значения. При полной компакции *всех* таблиц это так: слияние видит каждую версию, оставляет
надгробие (самую новую), и скрывать больше нечего.

Но есть подвох, связанный с устойчивостью к сбоям. Компакция делает две вещи: (1) пишет новую таблицу, (2) удаляет старые.
Сбой между ними оставляет на диске и то, и другое. Если новая таблица *выбросила* надгробие, старые таблицы всё ещё хранят
старое значение ключа, новая таблица ничего о ключе не говорит, и после перезапуска удалённый ключ **возвращается**. (Если
надгробие осталось в новой таблице, оно по-прежнему затеняет старое значение, и ничего плохого не происходит.)

Настоящие движки решают это **манифестом**: небольшим файлом со списком действительных файлов таблиц. Компакция пишет новую
таблицу, затем атомарно заменяет манифест (Урок 13.12) списком, в котором есть новая таблица и нет старых, и только потом
удаляет старые файлы. При перезапуске файлы, которых нет в манифесте, игнорируются и удаляются. Переименование манифеста —
единственный момент, когда меняется набор таблиц. В LevelDB и RocksDB этот файл называется `MANIFEST`.

У нашего хранилища нет манифеста, поэтому его компакция сохраняет надгробия (`drop_tombstones = false`). Это всегда
безопасно; цена — несколько байт на каждый удалённый ключ. Проект Урока 17.14 — хорошее место, чтобы добавить манифест и
выбрасывать их.

> 🔒 **Безопасность:** «удалённые» данные, которые возвращаются, — это не только ошибка. Если пользователь удалил личные
> данные, а они снова появились после сбоя, это может нарушить правила защиты данных. Помните также, что LSM-дерево хранит
> старые версии на диске до компакции: удалённые данные физически исчезают только после удаления файлов, в которых они были.

## 5. Фильтры Блума: «точно нет»

Компакция уменьшает число таблиц, но несколько таблиц есть всегда (по одной на уровень, плюс L0, плюс новые сбросы). Для
каждой таблицы поиску хотелось бы знать до чтения: «может ли этот ключ вообще быть в этой таблице?»

**Фильтр Блума** отвечает на это несколькими битами на ключ. Это битовый массив из `m` бит, сначала все 0, и `k`
хеш-функций. Чтобы **добавить** ключ, вычислите его `k` хешей — каждый даёт позицию в массиве — и поставьте эти `k` бит в 1.
Чтобы **спросить** о ключе, вычислите те же `k` позиций: если **хотя бы один** из этих битов 0, ключ **точно никогда не
добавляли**; если все равны 1, ключ, **возможно**, добавляли.

```text
 m = 16 bits, k = 3
 add "apple": positions 1, 6, 11          bits: 0100 0010 0001 0000
 add "kiwi":  positions 3, 6, 14          bits: 0101 0010 0001 0010
 ask "plum":  positions 3, 9, 14  → bit 9 is 0  → definitely not added
 ask "fig":   positions 1, 3, 11  → all are 1   → "maybe": a false positive
```

Ложных отрицаний не бывает никогда: добавленный ключ всегда находит свои биты установленными. Бывают **ложные
срабатывания**, когда другие ключи случайно установили все биты данного ключа. Ложное срабатывание стоит лишь одного
бесполезного чтения блока; сама таблица затем отвечает «не найдено». При лучшем выборе `k` (около `0,69 × бит на ключ`)
доля ложных срабатываний примерно равна

```text
 p ≈ (1 − e^(−k·n/m))^k        n = keys, m = bits, k = hash functions
 10 bits per key, k = 7  →  p ≈ 0.8 %
```

Итак, 10 бит (чуть больше одного байта) на ключ убирают больше 99 % бесполезных чтений, каким бы ни был размер ключей.

Создайте `src/bloom.rs` и добавьте `pub mod bloom;` в `src/lib.rs`:

```rust
/// FNV-1a, a simple 64-bit hash, followed by a "finalizer" that mixes the bits well.
/// Unlike std's `DefaultHasher`, the output is fixed forever, so a filter saved to disk
/// today can be read by tomorrow's program.
fn hash(bytes: &[u8], seed: u64) -> u64 {
    let mut h = 0xcbf2_9ce4_8422_2325 ^ seed;
    for &b in bytes {
        h ^= u64::from(b);
        h = h.wrapping_mul(0x0000_0100_0000_01b3);
    }
    // The finalizer of splitmix64: spreads every input bit over all output bits.
    h ^= h >> 30;
    h = h.wrapping_mul(0xbf58_476d_1ce4_e5b9);
    h ^= h >> 27;
    h = h.wrapping_mul(0x94d0_49bb_1331_11eb);
    h ^ (h >> 31)
}

/// A Bloom filter: "definitely not in the set" or "maybe in the set".
pub struct Bloom {
    bits: Vec<u64>, // the bit array, 64 bits per word
    m: u64,         // number of bits
    k: u32,         // number of hash functions
}

impl Bloom {
    /// A filter for about `items` keys with `bits_per_key` bits each.
    pub fn new(items: usize, bits_per_key: usize) -> Bloom {
        let m = (items * bits_per_key).max(64) as u64;
        // The best number of hash functions is bits_per_key * ln 2 (about 0.69).
        let k = ((bits_per_key as f64) * std::f64::consts::LN_2)
            .round()
            .max(1.0) as u32;
        Bloom {
            bits: vec![0; m.div_ceil(64) as usize],
            m,
            k,
        }
    }

    /// The k bit positions of a key, made from two hashes ("double hashing").
    fn positions(&self, key: &[u8]) -> impl Iterator<Item = u64> + '_ {
        let h1 = hash(key, 0);
        let h2 = hash(key, 0x9e37_79b9_7f4a_7c15) | 1;
        (0..u64::from(self.k)).map(move |i| h1.wrapping_add(i.wrapping_mul(h2)) % self.m)
    }

    pub fn insert(&mut self, key: &[u8]) {
        let positions: Vec<u64> = self.positions(key).collect();
        for bit in positions {
            self.bits[(bit / 64) as usize] |= 1 << (bit % 64);
        }
    }

    /// `false` means the key was never inserted. `true` means it probably was.
    pub fn may_contain(&self, key: &[u8]) -> bool {
        self.positions(key)
            .all(|bit| self.bits[(bit / 64) as usize] & (1 << (bit % 64)) != 0)
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn inserted_keys_are_always_found() {
        let mut filter = Bloom::new(10_000, 10);
        for i in 0..10_000 {
            filter.insert(format!("key{i}").as_bytes());
        }
        for i in 0..10_000 {
            assert!(filter.may_contain(format!("key{i}").as_bytes()), "key{i}");
        }
    }

    #[test]
    fn an_empty_filter_contains_nothing() {
        let filter = Bloom::new(100, 10);
        assert!(!filter.may_contain(b"anything"));
    }
}
```

**Что происходит:**

1. В настоящих движках фильтр сохраняется вместе с данными, поэтому его хеш никогда не должен меняться. `DefaultHasher` из
   `std` такого обещания между версиями Rust не даёт, поэтому мы пишем FNV-1a (Fowler–Noll–Vo, классический простой хеш)
   вручную. Сам по себе он плохо разводит похожие ключи вроде `user:1` и `user:2`, поэтому короткий *финализатор* из
   умножений и сдвигов перемешивает биты. `wrapping_mul` умножает и позволяет результату «завернуться», а не паниковать при
   переполнении (Урок 12.2).
2. Вычислять `k` независимых хешей было бы медленно. **Двойное хеширование** строит все `k` позиций из двух хешей:
   `h1 + i·h2` для `i = 0, 1, …, k-1`. Нечётный `h2` (`| 1`) делает позиции различными.
3. Битовый массив — это `Vec<u64>`: бит номер `b` живёт в слове `b / 64`, на бите `b % 64`. `|=` с `1 << …` его
   устанавливает, `&` проверяет (Урок 12.4).
4. `insert` сначала собирает позиции в `Vec`, потому что итератор из `positions` заимствует `self`, а цикл хочет менять
   `self.bits`; проверка заимствований не разрешила бы и то, и другое одновременно.
5. `all` останавливается на первом нулевом бите: большинство отсутствующих ключей отвергаются после одного-двух битов.

Насколько мы близки к формуле? Поместите это в `src/bin/bloom_rate.rs`:

```rust
use p17_storage::bloom::Bloom;

const N: usize = 100_000;

fn main() {
    println!("bits/key  measured  formula");
    for bits_per_key in [5, 10, 15] {
        let mut filter = Bloom::new(N, bits_per_key);
        for i in 0..N {
            filter.insert(format!("user:{i}").as_bytes());
        }
        // Ask for N keys that were never inserted and count the false "maybe" answers.
        let false_positives = (0..N)
            .filter(|i| filter.may_contain(format!("other:{i}").as_bytes()))
            .count();
        let measured = false_positives as f64 / N as f64;

        let k = (bits_per_key as f64 * std::f64::consts::LN_2).round();
        let formula = (1.0 - (-k / bits_per_key as f64).exp()).powf(k);
        println!(
            "{bits_per_key:>8}  {:>7.2}%  {:>6.2}%",
            measured * 100.0,
            formula * 100.0
        );
    }
}
```

`cargo run --release --bin bloom_rate`:

```text
bits/key  measured  formula
       5     9.25%    9.18%
      10     0.78%    0.82%
      15     0.08%    0.07%
```

Измеренные доли совпадают с формулой. Каждые 5 дополнительных бит на ключ делают ложные срабатывания примерно в десять раз
реже.

## 6. Собираем всё вместе

Теперь соединим обе идеи с `src/lsm.rs`. Сначала — фильтр для каждой таблицы. Измените импорты, структуру и добавьте
вспомогательную функцию:

```rust
use crate::bloom::Bloom;
use crate::compaction;
use crate::sstable::{SsTable, Value};
use std::collections::BTreeMap;
use std::fs;
use std::io;
use std::path::{Path, PathBuf};

/// A tiny LSM tree: a memtable in memory plus immutable SSTables on disk.
pub struct Lsm {
    dir: PathBuf,
    memtable: BTreeMap<Vec<u8>, Value>,
    memtable_bytes: usize,
    memtable_limit: usize,
    tables: Vec<SsTable>, // newest first
    filters: Vec<Bloom>,  // filters[i] belongs to tables[i]
    next_id: u64,
}

const BITS_PER_KEY: usize = 10;

/// Builds a Bloom filter with all keys of `table`.
fn build_filter(table: &SsTable) -> io::Result<Bloom> {
    let entries = table.scan()?;
    let mut filter = Bloom::new(entries.len(), BITS_PER_KEY);
    for (key, _) in &entries {
        filter.insert(key);
    }
    Ok(filter)
}
```

В `open` постройте фильтр для каждой таблицы и добавьте `filters,` в `Ok(Lsm { … })` после `tables,`:

```rust
        let mut tables = Vec::new();
        let mut filters = Vec::new();
        for id in &ids {
            let table = SsTable::open(&dir.join(format!("{id:06}.sst")))?;
            filters.push(build_filter(&table)?);
            tables.push(table);
        }
```

В `flush` и `get`:

```rust
    pub fn flush(&mut self) -> io::Result<()> {
        if self.memtable.is_empty() {
            return Ok(());
        }
        let path = self.dir.join(format!("{:06}.sst", self.next_id));
        SsTable::write(&path, self.memtable.iter())?;
        let table = SsTable::open(&path)?;
        self.filters.insert(0, build_filter(&table)?);
        self.tables.insert(0, table); // the newest table goes first
        self.next_id += 1;
        self.memtable.clear();
        self.memtable_bytes = 0;
        Ok(())
    }

    pub fn get(&self, key: &[u8]) -> io::Result<Option<Vec<u8>>> {
        if let Some(value) = self.memtable.get(key) {
            return Ok(value.clone()); // newest data; None means deleted
        }
        for (table, filter) in self.tables.iter().zip(&self.filters) {
            if !filter.may_contain(key) {
                continue; // definitely not in this table: skip it without reading
            }
            if let Some(value) = table.get(key)? {
                return Ok(value); // the first (newest) table that knows the key wins
            }
        }
        Ok(None)
    }
```

И компакция — новый метод в `impl Lsm`:

```rust
    pub fn compact(&mut self) -> io::Result<()> {
        if self.tables.len() < 2 {
            return Ok(());
        }
        let mut runs = Vec::new();
        for table in &self.tables {
            runs.push(table.scan()?); // newest first, as `merge` expects
        }
        // Keep tombstones: see "When can a tombstone go?" in the lesson.
        let merged = compaction::merge(runs, false);

        let id = self.next_id;
        let path = self.dir.join(format!("{id:06}.sst"));
        // 1. the new table is complete and durable before anything is deleted
        SsTable::write(&path, merged.iter().map(|(key, value)| (key, value)))?;
        let table = SsTable::open(&path)?;
        let filter = build_filter(&table)?;

        for entry in fs::read_dir(&self.dir)? {
            let entry = entry?;
            let name = entry.file_name().to_string_lossy().into_owned();
            let old = name
                .strip_suffix(".sst")
                .and_then(|n| n.parse::<u64>().ok())
                .is_some_and(|n| n < id);
            if old {
                fs::remove_file(entry.path())?; // 2. only now delete the old tables
            }
        }
        fs::File::open(&self.dir)?.sync_all()?;

        self.tables = vec![table];
        self.filters = vec![filter];
        self.next_id = id + 1;
        Ok(())
    }
```

**Что происходит:**

1. `build_filter` один раз просматривает таблицу и добавляет каждый ключ, включая удалённые: надгробие — это ответ, который
   поиск обязан найти. Настоящие движки хранят фильтр внутри файла SSTable, а не строят его заново при запуске
   (упражнение 3 готовит к этому).
2. `zip` проходит по таблицам и фильтрам вместе. Таблица, чей фильтр говорит «точно нет», пропускается без единого чтения.
3. `compact` просматривает все таблицы от новой к старой, сливает их и пишет результат как таблицу с **новым, бо́льшим**
   номером. `merged.iter().map(|(key, value)| (key, value))` превращает элементы `&(K, V)` в пары `(&K, &V)`, которых ждёт
   `write`.
4. Старые файлы удаляются только после того, как новая таблица стала надёжной, а затем синхронизируется каталог. Сбой
   посередине оставляет лишние старые таблицы, что здесь безвредно: новая таблица новее и сохранила свои надгробия
   (раздел 4).

Тест, в тестах `src/lsm.rs`:

```rust
    #[test]
    fn compaction_keeps_the_newest_data() -> io::Result<()> {
        let dir = tempfile::tempdir()?;
        let mut db = Lsm::open(dir.path(), 1_000_000)?;
        for round in 0..5 {
            for i in 0..100 {
                db.put(
                    format!("key{i:03}").as_bytes(),
                    format!("v{round}").as_bytes(),
                )?;
            }
            db.delete(format!("key{round:03}").as_bytes())?;
            db.flush()?;
        }
        assert_eq!(db.table_count(), 5);

        db.compact()?;
        assert_eq!(db.table_count(), 1);
        assert_eq!(db.get(b"key050")?, Some(b"v4".to_vec()));
        assert_eq!(db.get(b"key004")?, None); // deleted in the last round
        assert_eq!(db.get(b"key003")?, Some(b"v4".to_vec())); // deleted, then written again
        drop(db);

        let db = Lsm::open(dir.path(), 1_000_000)?;
        assert_eq!(db.table_count(), 1);
        assert_eq!(db.get(b"key050")?, Some(b"v4".to_vec()));
        Ok(())
    }
```

Запустите тесты трёх модулей вместе (после `--` можно указать несколько фильтров):

```bash
cargo test --lib -- bloom compaction lsm
```

```text
running 9 tests
test compaction::tests::tombstones_can_be_dropped ... ok
test bloom::tests::an_empty_filter_contains_nothing ... ok
test compaction::tests::newest_version_wins_and_output_is_sorted ... ok
test bloom::tests::inserted_keys_are_always_found ... ok
test lsm::tests::scan_all_merges_everything ... ok
test lsm::tests::newest_value_wins_across_memtable_and_tables ... ok
test lsm::tests::a_tombstone_survives_flush_and_reopen ... ok
test lsm::tests::compaction_keeps_the_newest_data ... ok
test lsm::tests::flushed_data_survives_a_reopen ... ok

test result: ok. 9 passed; 0 failed; 0 ignored; 0 measured; 29 filtered out; finished in 0.15s
```

Наконец, снова демо из Урока 17.7, теперь с фильтрами и компакцией. Поместите это в `src/bin/compaction_demo.rs`:

```rust
use p17_storage::lsm::Lsm;
use std::path::Path;
use std::time::Instant;

const KEYS: u64 = 100_000;

fn reads_for(db: &Lsm, key: &[u8]) -> std::io::Result<(bool, u64)> {
    let before = db.block_reads();
    let found = db.get(key)?.is_some();
    Ok((found, db.block_reads() - before))
}

fn main() -> std::io::Result<()> {
    let dir = Path::new("compaction-demo");
    let mut db = Lsm::open(dir, 256 * 1024)?;
    for i in 0..KEYS {
        let n = i * 7919 % KEYS; // the same scrambled order as in Lesson 17.7
        let value = format!("value of {n} {}", "-".repeat(40));
        db.put(format!("user:{n:06}").as_bytes(), value.as_bytes())?;
    }
    db.flush()?;

    println!("with Bloom filters, {} tables:", db.table_count());
    println!("  existing key: {:?}", reads_for(&db, b"user:050000")?);
    println!("  missing key:  {:?}", reads_for(&db, b"user:050000x")?);

    let start = Instant::now();
    db.compact()?;
    println!("compaction took {:?}", start.elapsed());

    println!("after compaction, {} table:", db.table_count());
    println!("  existing key: {:?}", reads_for(&db, b"user:050000")?);
    println!("  missing key:  {:?}", reads_for(&db, b"user:050000x")?);

    std::fs::remove_dir_all(dir)?;
    Ok(())
}
```

`cargo run --release --bin compaction_demo` (каждая пара — это `(found, blocks read)`, «найден» и «прочитано блоков»):

```text
with Bloom filters, 26 tables:
  existing key: (true, 1)
  missing key:  (false, 0)
compaction took 50.923625ms
after compaction, 1 table:
  existing key: (true, 1)
  missing key:  (false, 0)
```

Сравните с Уроком 17.7: 14 и 26 чтений блоков превратились в 1 и 0. Это сделали одни только фильтры, даже при 26 таблицах:
25 из них сказали «точно нет» для существующего ключа и все 26 — для отсутствующего. (При примерно 0,8 % ложных срабатываний
в отдельных запусках бывает одно лишнее чтение.) Затем компакция превратила 26 файлов в 1 примерно за 50 мс для 8 МБ данных.
Это демо пишет каждый ключ лишь один раз, поэтому общий размер почти не меняется; при перезаписях и удалениях компакция также
освобождает место (Попробуйте сами, шаг 1).

## Частые ошибки

1. **Выбрасывать надгробие внутри цикла слияния.** Следующая старая версия становится «самой новой», и удалённый ключ
   возвращается. Решайте судьбу надгробий после того, как выбрана самая новая версия каждого ключа.
2. **Выбрасывать надгробия без манифеста.** Сбой между записью новой таблицы и удалением старых возвращает удалённые ключи
   (раздел 4).
3. **Строить фильтр Блума на `DefaultHasher` и сохранять его.** Его алгоритм может поменяться между версиями Rust, и
   сохранённый фильтр тогда ответил бы «точно нет» для ключей, которые есть: ложные отрицания, настоящая потеря данных при
   чтении.
4. **Не включать надгробия в фильтр.** Поиск должен найти надгробие, чтобы остановиться; если фильтр пропустит эту таблицу,
   более старая таблица вернёт удалённое значение.
5. **Удалять старые таблицы до того, как новая стала надёжной.** Тогда сбой теряет данные. Запишите, синхронизируйте и только
   потом удаляйте.

## Попробуйте сами

1. В `compaction_demo.rs` напечатайте сумму размеров файлов `.sst` до и после `compact` (используйте `fs::read_dir` и
   `metadata().len()`). Затем перед компакцией перезапишите каждый ключ ещё раз и сравните снова.
2. Замените `BITS_PER_KEY` на 2 и запустите демо несколько раз. Как часто теперь отсутствующий ключ стоит чтения?
3. В `bloom_rate.rs` замените часть `hash` с финализатором просто на `h` (чистый FNV-1a) и запустите. Что происходит с
   измеренными долями? (Потом верните финализатор.)

## Упражнения

### Упражнение 1 — Сколько бит? (легко)

Для целевой доли ложных срабатываний `p` лучшему фильтру нужно `-ln(p) / (ln 2)²` бит на ключ. Добавьте
`pub fn bits_per_key_for(p: f64) -> f64` в `src/bloom.rs` и проверьте её: для 1 % нужно около 9,585 бита на ключ, для
0,1 % — около 14,378. Сколько мегабайт нужно фильтру на 1 % для 100 миллионов ключей?

<details>
<summary>Подсказка</summary>

`p.ln()` — натуральный логарифм, а `std::f64::consts::LN_2` — это ln 2. Сравнивайте числа с плавающей точкой с допуском
(Урок 12.3): `(a - b).abs() < 0.001`.

</details>

<details>
<summary>Ответ</summary>

```rust
/// Exercise 1: bits per key needed for a false-positive rate `p` (with the best k).
pub fn bits_per_key_for(p: f64) -> f64 {
    -p.ln() / (std::f64::consts::LN_2 * std::f64::consts::LN_2)
}
```

```rust
    #[test]
    fn bits_for_a_rate() {
        assert!((bits_per_key_for(0.01) - 9.585).abs() < 0.001);
        assert!((bits_per_key_for(0.001) - 14.378).abs() < 0.001);
    }
```

100 миллионов ключей × 9,585 бита ≈ 958,5 миллиона бит ≈ 120 МБ. Почему это работает: каждое уменьшение доли в десять раз
стоит всего около 4,8 дополнительного бита на ключ, поэтому настоящие движки могут позволить себе фильтры для каждой таблицы.
120 МБ — немного рядом с данными 100 миллионов ключей и гораздо меньше хеш-индекса по ним (Урок 17.5).

</details>

### Упражнение 2 — Проверьте долю ложных срабатываний (средне)

Напишите тест `false_positive_rate_is_about_one_percent`: добавьте 10 000 ключей `in:0` … `in:9999` в фильтр с 10 битами на
ключ, спросите про 10 000 других ключей `out:0` … `out:9999` и проверьте, что «возможно» получили меньше 150 из них (1,5 %).

<details>
<summary>Подсказка</summary>

Считайте через `(0..10_000).filter(|i| filter.may_contain(...)).count()`. Почему 1,5 %, а не ровно 0,82 %? Результат одного
запуска немного колеблется; тесту нужен запас, иначе он станет нестабильным.

</details>

<details>
<summary>Ответ</summary>

```rust
    #[test]
    fn false_positive_rate_is_about_one_percent() {
        let mut filter = Bloom::new(10_000, 10);
        for i in 0..10_000 {
            filter.insert(format!("in:{i}").as_bytes());
        }
        let false_positives = (0..10_000)
            .filter(|i| filter.may_contain(format!("out:{i}").as_bytes()))
            .count();
        assert!(false_positives < 150, "{false_positives} false positives");
    }
```

Почему это работает: хеш фиксирован, поэтому тест даёт одно и то же число при каждом запуске и на любой машине; он не
нестабилен. Запас всё равно важен: небольшое изменение хеша изменило бы число, а тест должен падать, только когда фильтр
действительно стал хуже. При неудаче сообщение показывает число.

</details>

### Упражнение 3 — Сохраните фильтр (сложно)

Настоящие движки хранят фильтр каждой таблицы на диске, чтобы при открытии таблицы не приходилось её просматривать. Добавьте
в `Bloom` методы `pub fn to_bytes(&self) -> Vec<u8>` и `pub fn from_bytes(bytes: &[u8]) -> Option<Bloom>`. Формат: `k` как
`u32` LE, `m` как `u64` LE, затем слова как `u64` LE. `from_bytes` возвращает `None` для слишком короткого ввода, для `k` или
`m`, равных 0, и для неправильного числа байтов слов при данном `m`. Проверьте круговой путь и два сломанных ввода.

<details>
<summary>Подсказка</summary>

Число слов — `m.div_ceil(64)`, поэтому байтов слов должно быть ровно `m.div_ceil(64) * 8`. `bytes.get(0..4)` возвращает
`None`, а не паникует, если ввод слишком короткий, а `?` передаёт этот `None` дальше. `as_chunks::<8>()` делит срез на
массивы `[u8; 8]`.

</details>

<details>
<summary>Ответ</summary>

```rust
impl Bloom {
    /// Exercise 3: the filter as bytes: k (u32 LE), m (u64 LE), then the words (u64 LE).
    pub fn to_bytes(&self) -> Vec<u8> {
        let mut out = Vec::with_capacity(12 + self.bits.len() * 8);
        out.extend_from_slice(&self.k.to_le_bytes());
        out.extend_from_slice(&self.m.to_le_bytes());
        for word in &self.bits {
            out.extend_from_slice(&word.to_le_bytes());
        }
        out
    }

    /// Exercise 3: the opposite of `to_bytes`; `None` if the bytes do not fit together.
    pub fn from_bytes(bytes: &[u8]) -> Option<Bloom> {
        let k = u32::from_le_bytes(bytes.get(0..4)?.try_into().ok()?);
        let m = u64::from_le_bytes(bytes.get(4..12)?.try_into().ok()?);
        let words = &bytes[12..];
        if k == 0 || m == 0 || words.len() as u64 != m.div_ceil(64) * 8 {
            return None;
        }
        let (chunks, _) = words.as_chunks::<8>(); // the length check above leaves no rest
        let bits = chunks.iter().map(|w| u64::from_le_bytes(*w)).collect();
        Some(Bloom { bits, m, k })
    }
}
```

```rust
    #[test]
    fn a_filter_survives_bytes() {
        let mut filter = Bloom::new(1_000, 10);
        for i in 0..1_000 {
            filter.insert(format!("key{i}").as_bytes());
        }
        let bytes = filter.to_bytes();
        let back = Bloom::from_bytes(&bytes).unwrap();
        assert_eq!(back.to_bytes(), bytes);
        assert!(back.may_contain(b"key7"));
        assert!(Bloom::from_bytes(&bytes[..bytes.len() - 1]).is_none());
        assert!(Bloom::from_bytes(&[1, 2, 3]).is_none());
    }
```

`cargo test --lib bloom` (вместе с упражнениями 1 и 2):

```text
running 5 tests
test bloom::tests::bits_for_a_rate ... ok
test bloom::tests::an_empty_filter_contains_nothing ... ok
test bloom::tests::a_filter_survives_bytes ... ok
test bloom::tests::false_positive_rate_is_about_one_percent ... ok
test bloom::tests::inserted_keys_are_always_found ... ok

test result: ok. 5 passed; 0 failed; 0 ignored; 0 measured; 36 filtered out; finished in 0.01s
```

Почему это работает: все длины проверяются до декодирования слов, поэтому повреждённые байты дают `None`, а не панику или
фильтр не того размера (правило из Урока 17.3). Неверное `m` отображало бы ключи на не те биты и вызывало бы ложные
отрицания — единственную ошибку, которую фильтр Блума не имеет права совершать. В настоящей SSTable эти байты лежали бы в
отдельном блоке под защитой CRC (Cyclic Redundancy Check, контрольной суммы из Урока 17.3).

</details>

## Вопросы для самопроверки

1. Какие две проблемы решает компакция?

<details>
<summary>Ответ</summary>

Усиление чтения (меньше таблиц для проверки) и усиление по месту (убираются старые версии, а при наличии манифеста — и
надгробия).

</details>

2. Сколько элементов самое большее находится в куче при k-путевом слиянии 10 последовательностей и сколько стоит каждый шаг?

<details>
<summary>Ответ</summary>

Не больше 10, по одному на последовательность. Каждое снятие и добавление стоит O(log 10), какой бы длины ни были
последовательности.

</details>

3. Почему при равных ключах слияние должно предпочитать самую новую последовательность?

<details>
<summary>Ответ</summary>

Самая новая версия — текущее значение ключа (или его надгробие). Взяв более старую версию, мы вернули бы старое значение
или удалённый ключ.

</details>

4. Сравните компакцию по размерам и по уровням, по одному предложению на каждую.

<details>
<summary>Ответ</summary>

Компакция по размерам сливает таблицы похожего размера в бо́льшие: мало переписывания, но несколько таблиц для поиска и
большие временные потребности в месте. Компакция по уровням держит непересекающиеся таблицы на уровнях, каждый из которых в
десять раз больше предыдущего: не больше одной таблицы на уровень для поиска, но данные переписываются на каждом уровне.

</details>

5. Фильтр Блума говорит «возможно». Что делает поиск и каков худший исход?

<details>
<summary>Ответ</summary>

Он читает блок таблицы. Худший исход — ложное срабатывание: одно бесполезное чтение, после которого таблица говорит «не
найдено». Ответ никогда не бывает неверным.

</details>

6. Почему наша компакция сохраняет надгробия?

<details>
<summary>Ответ</summary>

Без манифеста сбой между записью новой таблицы и удалением старых оставил бы старые значения на диске; новая таблица без
надгробия позволила бы тогда удалённому ключу вернуться. Сохранённое надгробие продолжает их затенять.

</details>

## Итоги

- Компакция сливает SSTable в меньшее число таблиц, оставляет только самую новую версию каждого ключа и освобождает место;
  она устраняет бо́льшую часть усиления чтения и по месту в LSM.
- K-путевое слияние с min-кучей сливает отсортированные последовательности за один последовательный проход; при равных ключах
  побеждает самая новая последовательность.
- Компакция по размерам выгоднее для записи; компакция по уровням выгоднее для чтения и места ценой большего переписывания.
- Надгробие может уйти только тогда, когда ничего более старого существовать не может и никакой сбой не вернёт старые
  таблицы; настоящие движки используют манифест.
- Фильтр Блума (около 10 бит на ключ, `k` ≈ 7) отвечает «точно нет» или «возможно» примерно с 0,8 % ложных срабатываний и без
  ложных отрицаний.
- Его хеш должен быть зафиксирован навсегда; мы используем FNV-1a с финализатором и двойным хешированием.
- Вместе фильтры и компакция превратили 26 чтений блоков ради отсутствующего ключа в 0.

## Источники и что почитать

- [RocksDB wiki — Compaction](https://github.com/facebook/rocksdb/wiki/Compaction) — компакция по уровням, universal (по
  размерам) и FIFO (first in, first out, «первым пришёл — первым ушёл») в самом распространённом LSM-движке.
- [RocksDB wiki — RocksDB Bloom Filter](https://github.com/facebook/rocksdb/wiki/RocksDB-Bloom-Filter) — как настоящий движок
  хранит фильтры для каждой таблицы и выбирает число бит на ключ.
- [LevelDB implementation notes](https://github.com/google/leveldb/blob/main/doc/impl.md) — уровни, компакции и файл
  `MANIFEST`.
- [Less Hashing, Same Performance: Building a Better Bloom Filter](https://www.eecs.harvard.edu/~michaelm/postscripts/rsa2008.pdf)
  — статья Кирша и Митценмахера, на которой основано двойное хеширование (продвинутый уровень).
- [std::collections::BinaryHeap](https://doc.rust-lang.org/std/collections/struct.BinaryHeap.html) и
  [std::cmp::Reverse](https://doc.rust-lang.org/std/cmp/struct.Reverse.html) — куча, которую использует слияние.

---

[🏠 Начало](../START-HERE.md) · [← 17.7 LSM-деревья](07-lsm-trees.md) · [📋 Этап 17](README.md) · [17.9 Страничный кэш и пул буферов →](09-page-cache-and-buffer-pool.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
