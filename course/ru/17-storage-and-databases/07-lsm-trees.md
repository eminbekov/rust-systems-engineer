<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 17.6 B-деревья](06-b-trees.md) · [📋 Этап 17](README.md) · [17.8 Компакция и фильтры Блума →](08-compaction-and-bloom-filters.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/17-storage-and-databases/07-lsm-trees.md) <!-- nav -->

# Урок 17.7 — LSM-деревья

> **Этап 17 — Как устроены хранилища и базы данных** · Урок 7 из 15 · ⏱ около 3 часов (2 занятия)

## Что вы узнаете

- Как LSM-дерево сочетает быстрое дописывание журнала с отсортированным порядком дерева.
- Что такое memtable и SSTable и как через них проходят записи, удаления и чтения.
- Как записать и прочитать SSTable с разреженным индексом, чтобы поиск читал один маленький блок.
- Как построить на Rust маленькое LSM-хранилище, которое сохраняет данные между перезапусками.
- Почему чтения замедляются по мере накопления SSTable (усиление чтения) и чем отличаются B-деревья и LSM-деревья.

## Перед началом

- Вы прошли [Урок 17.6 — B-деревья](06-b-trees.md) и [Урок 17.5 — Индексы: хеш-индекс](05-hash-indexes.md).
  LSM-дерево берёт идею «только дописывать» из 17.5 и отсортированный порядок из 17.6.
- Вы знаете WAL из [Урока 17.4](04-write-ahead-log.md), формат записей из [Урока 17.3](03-binary-file-formats.md) и
  шаблон атомарной замены из [Урока 13.12](../13-operating-systems/12-file-systems-inside.md).
- Вы знаете внутреннюю изменяемость (`RefCell`) из [Урока 05.9 — Умные указатели](../05-rust-intermediate/09-smart-pointers.md).
  В этом уроке используется `Cell`, её более простая родственница для маленьких `Copy`-значений вроде счётчиков: `get`
  читает значение, `set` заменяет его, оба через общую ссылку.

## Новые слова

| Слово | Значение |
|---|---|
| **LSM-дерево** (LSM tree, Log-Structured Merge tree, дерево слияния со структурой журнала) | Устройство хранения, которое собирает записи в отсортированной таблице в памяти, записывает её неизменяемым отсортированным файлом, когда она заполнена, а позже сливает эти файлы. |
| **memtable** | Отсортированная таблица в памяти, которая принимает все новые записи LSM-дерева. Здесь — `BTreeMap`. |
| **SSTable** (Sorted String Table, отсортированная таблица строк) | Неизменяемый файл записей «ключ-значение», отсортированных по ключу, с индексом. «String» — историческое название: ключи и значения — просто байты. |
| **сброс** (flush) | Запись заполненной memtable на диск в виде новой SSTable и начало пустой memtable. |
| **неизменяемый** (immutable) | Никогда не меняется после записи. SSTable только создают, читают и (позже) удаляют. |
| **разреженный индекс** (sparse index) | Индекс с одной записью на *блок* записей, а не на каждую запись, поэтому он остаётся маленьким. |
| **затенение** (shadowing) | Более новое значение ключа скрывает все старые значения этого ключа в старых файлах. |
| **усиление чтения** (read amplification) | Чтение больше чем одного места (или больше байтов, чем нужно), чтобы ответить на один поиск. |

## 1. Две идеи в одной

Хеш-журнал из Урока 17.5 пишет быстро (он только дописывает), но не умеет отвечать на запросы по диапазону и требует все
ключи в RAM. B-дерево из Урока 17.6 отсортировано, но меняет страницы на месте, поэтому каждая вставка — случайная запись
целой страницы. **LSM-дерево** берёт лучшее от обоих: оно пишет только **последовательно** и держит данные
**отсортированными**.

Приём такой: сортировать в памяти и писать большими отсортированными кусками:

```text
 write path                                    read path (newest first)
 ──────────                                    ─────────
 put/delete ─► WAL (durable, Lesson 17.4)      get(key)
            └► memtable (BTreeMap, in RAM)        1. memtable
                   │ full?                        2. newest SSTable
                   ▼ flush                        3. older SSTable
               000003.sst  (newest)               4. ...
               000002.sst                         5. oldest SSTable
               000001.sst  (oldest)               the first one that knows the key wins
```

1. Каждая запись идёт в **memtable** — отсортированный словарь в памяти. (А сначала, ради надёжности, в WAL; см. раздел 6.)
2. Когда memtable достаточно велика, она **сбрасывается**: записывается за один раз, по порядку ключей, в новый файл
   **SSTable**. Затем начинается новая пустая memtable. SSTable **неизменяема**: её больше никогда не меняют.
3. Чтение смотрит сначала в memtable, затем в SSTable от новейшей к самой старой. Первое место, которое знает ключ, хранит
   его самое новое значение. Это **затенение**.
4. Удаление не может убрать ключ из старых неизменяемых файлов, поэтому оно пишет **надгробие** (Урок 17.5), которое
   затеняет старые значения.
5. Со временем SSTable накапливается много. **Компакция** сливает их в фоне (Урок 17.8).

LevelDB, RocksDB, Cassandra и многие другие базы данных — это LSM-деревья. Один только RocksDB работает под заметной частью
мирового хранения «ключ-значение».

## 2. Формат SSTable

SSTable — это наш формат записей из Урока 17.3, отсортированный по ключу, плюс индекс в конце:

```text
 ┌──────────────────────── data ─────────────────────────┬──── index ────┬────────┐
 │ block 0: 16 records │ block 1: 16 records │ block 2 … │ (key0, off 0) │ index  │
 │ apple … bean        │ beet … corn         │           │ (beet, off X) │ offset │
 │                     │                     │           │ …             │ u64 LE │
 └─────────────────────┴─────────────────────┴───────────┴───────────────┴────────┘
                                                          ▲                  │
                                                          └──────────────────┘
```

- **Данные** — все записи по порядку ключей. Значение каждой записи начинается с байта типа из Урока 17.4 (1 = put,
  0 = надгробие).
- Каждые 16 записей начинают новый **блок**. В **индексе** одна запись на блок: первый ключ блока и его смещение. Это
  **разреженный индекс**: в 16 раз меньше, чем по записи на каждую запись данных.
- Последние 8 байт, **подвал** (footer), хранят смещение, с которого начинается индекс. Читатель сначала читает подвал,
  затем индекс, и держит индекс в памяти.

Чтобы найти ключ, сделайте двоичный поиск по индексу — последний блок, первый ключ которого `<=` искомого, — прочитайте этот
один блок (16 маленьких записей, одно чтение по смещению) и просмотрите его. Поскольку файл отсортирован, ключ может быть
только в этом блоке.

## 3. Запись и чтение SSTable

Создайте `src/sstable.rs` и добавьте `pub mod sstable;` в `src/lib.rs`. Типы и вспомогательные функции:

```rust
use crate::record;
use std::cell::Cell;
use std::fs::{self, File};
use std::io::{self, Write};
use std::os::unix::fs::FileExt;
use std::path::Path;

const PUT: u8 = 1;
const DELETE: u8 = 0;
/// One index entry for every this many records.
const BLOCK_RECORDS: usize = 16;

/// A value, or `None` for a tombstone (the key was deleted).
pub type Value = Option<Vec<u8>>;

/// An immutable, sorted file of key-value records with a sparse index.
/// Layout: [data records][index records][index offset: u64 LE]
pub struct SsTable {
    file: File,
    index: Vec<(Vec<u8>, u64)>, // first key of each block, offset of the block
    data_end: u64,
    /// How many blocks were read from the file (for measuring).
    pub block_reads: Cell<u64>,
}

fn encode_value(value: &Value) -> Vec<u8> {
    match value {
        Some(v) => {
            let mut entry = vec![PUT];
            entry.extend_from_slice(v);
            entry
        }
        None => vec![DELETE],
    }
}

fn decode_value(entry: &[u8]) -> Value {
    match entry.split_first() {
        Some((&PUT, v)) => Some(v.to_vec()),
        _ => None,
    }
}

fn invalid(msg: &str) -> io::Error {
    io::Error::new(io::ErrorKind::InvalidData, msg.to_string())
}
```

`Value = Option<Vec<u8>>` — вся идея надгробия в одном типе: `Some(bytes)` — значение, `None` — «удалено». `block_reads`
— счётчик для измерений; это `Cell`, потому что `get` принимает `&self` и всё равно должен считать.

Запись, в `impl SsTable`:

```rust
impl SsTable {
    /// Writes `entries` (which must be sorted by key, without duplicates) to a new file at `path`.
    pub fn write<'a>(
        path: &Path,
        entries: impl Iterator<Item = (&'a Vec<u8>, &'a Value)>,
    ) -> io::Result<()> {
        let mut data = Vec::new();
        let mut index = Vec::new();
        for (i, (key, value)) in entries.enumerate() {
            if i % BLOCK_RECORDS == 0 {
                // This record starts a new block: remember its key and offset.
                record::encode(key, &(data.len() as u64).to_le_bytes(), &mut index);
            }
            record::encode(key, &encode_value(value), &mut data);
        }
        let index_offset = data.len() as u64;
        data.extend_from_slice(&index);
        data.extend_from_slice(&index_offset.to_le_bytes());

        // Atomic replace (Lesson 13.12): a crash leaves no half-written table.
        let tmp = path.with_extension("tmp");
        let mut file = File::create(&tmp)?;
        file.write_all(&data)?;
        file.sync_all()?;
        fs::rename(&tmp, path)?;
        File::open(path.parent().unwrap_or(Path::new(".")))?.sync_all()
    }
```

**Что происходит:**

1. `entries` — любой итератор пар `(ключ, значение)`, например `memtable.iter()` у `BTreeMap`, который выдаёт записи в
   отсортированном порядке. Время жизни `'a` говорит, что ссылки живут по крайней мере столько же, сколько вызов.
2. Записи индекса — тоже обычные записи: ключ и смещение блока в виде 8 байт little-endian как значение. Так индекс
   бесплатно получает длины и контрольные суммы.
3. Весь файл строится в памяти и пишется одним `write_all`: одна большая последовательная запись (Урок 17.1).
4. Файл пишется под временным именем, синхронизируется и переименовывается, а каталог синхронизируется. Читатель видит либо
   отсутствие таблицы, либо целую таблицу, но никогда половину.

Открытие и чтение, всё ещё в `impl SsTable`:

```rust
    pub fn open(path: &Path) -> io::Result<SsTable> {
        let file = File::open(path)?;
        let len = file.metadata()?.len();
        if len < 8 {
            return Err(invalid("file too short"));
        }
        let mut footer = [0u8; 8];
        file.read_exact_at(&mut footer, len - 8)?;
        let data_end = u64::from_le_bytes(footer);
        if data_end > len - 8 {
            return Err(invalid("bad footer"));
        }

        let mut raw = vec![0u8; (len - 8 - data_end) as usize];
        file.read_exact_at(&mut raw, data_end)?;
        let mut index = Vec::new();
        let mut pos = 0;
        while pos < raw.len() {
            let (rec, used) = record::decode(&raw[pos..]).map_err(|_| invalid("bad index"))?;
            let offset =
                u64::from_le_bytes(rec.value.try_into().map_err(|_| invalid("bad index"))?);
            index.push((rec.key, offset));
            pos += used;
        }
        Ok(SsTable {
            file,
            index,
            data_end,
            block_reads: Cell::new(0),
        })
    }

    /// `None`: this table knows nothing about the key.
    /// `Some(None)`: the key was deleted. `Some(Some(v))`: the key has value `v`.
    pub fn get(&self, key: &[u8]) -> io::Result<Option<Value>> {
        // The block to read is the last one whose first key is <= `key`.
        let n = self
            .index
            .partition_point(|(first, _)| first.as_slice() <= key);
        if n == 0 {
            return Ok(None); // smaller than the first key of the table
        }
        let start = self.index[n - 1].1;
        let end = self
            .index
            .get(n)
            .map_or(self.data_end, |(_, offset)| *offset);

        let mut block = vec![0u8; (end - start) as usize];
        self.file.read_exact_at(&mut block, start)?;
        self.block_reads.set(self.block_reads.get() + 1);

        let mut pos = 0;
        while pos < block.len() {
            let (rec, used) = record::decode(&block[pos..]).map_err(|_| invalid("bad record"))?;
            if rec.key.as_slice() == key {
                return Ok(Some(decode_value(&rec.value)));
            }
            if rec.key.as_slice() > key {
                break; // sorted: the key is not in this block
            }
            pos += used;
        }
        Ok(None)
    }

    /// Reads every record of the table, in key order.
    pub fn scan(&self) -> io::Result<Vec<(Vec<u8>, Value)>> {
        let mut data = vec![0u8; self.data_end as usize];
        self.file.read_exact_at(&mut data, 0)?;
        let mut out = Vec::new();
        let mut pos = 0;
        while pos < data.len() {
            let (rec, used) = record::decode(&data[pos..]).map_err(|_| invalid("bad record"))?;
            out.push((rec.key, decode_value(&rec.value)));
            pos += used;
        }
        Ok(out)
    }
}
```

**Что происходит:**

1. `open` читает подвал, проверяет, что он указывает внутрь файла (значение с диска — это ввод, Урок 17.3), читает область
   индекса и декодирует её в `Vec` из пар `(первый ключ, смещение)`. Сами данные не читаются.
2. `rec.value.try_into()` превращает `Vec<u8>` из 8 байт в `[u8; 8]`; при неверной длине он завершается ошибкой, о которой
   мы сообщаем как о повреждении.
3. `get` использует `partition_point` (двоичный поиск по отсортированному срезу): он возвращает, у скольких записей индекса
   первый ключ `<= key`. Читать нужно последний из этих блоков. Если таких нет, ключ меньше всего, что есть в таблице.
4. Блок кончается там, где начинается следующий, или в `data_end` для последнего блока. Один `read_exact_at` его читает.
5. Внутри блока просмотр останавливается рано, когда проходит мимо ключа, потому что записи отсортированы.
6. У `get` три ответа, поэтому он возвращает `Option<Value>` = `Option<Option<Vec<u8>>>`: «не знаю», «удалено» или
   значение. LSM нужно отличать «не знаю, спроси таблицу постарше» от «удалено, остановись».
7. `scan` читает все записи данных; Урок 17.8 использует его для слияния таблиц.

Тест в конце `src/sstable.rs`:

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use std::collections::BTreeMap;

    #[test]
    fn write_then_get_every_kind_of_key() -> io::Result<()> {
        let dir = tempfile::tempdir()?;
        let path = dir.path().join("000001.sst");
        let mut entries: BTreeMap<Vec<u8>, Value> = BTreeMap::new();
        for i in 0..100u32 {
            let key = format!("key{:03}", i * 2).into_bytes(); // even numbers only
            entries.insert(key, Some(i.to_string().into_bytes()));
        }
        entries.insert(b"key050".to_vec(), None); // a tombstone
        SsTable::write(&path, entries.iter())?;

        let table = SsTable::open(&path)?;
        assert_eq!(table.index.len(), 7); // 100 records, a block every 16
        assert_eq!(table.get(b"key000")?, Some(Some(b"0".to_vec())));
        assert_eq!(table.get(b"key198")?, Some(Some(b"99".to_vec())));
        assert_eq!(table.get(b"key050")?, Some(None)); // deleted
        assert_eq!(table.get(b"key051")?, None); // between two keys
        assert_eq!(table.get(b"aaa")?, None); // before the first key
        assert_eq!(table.get(b"zzz")?, None); // after the last key
        assert_eq!(table.scan()?.len(), 100);
        Ok(())
    }
}
```

`cargo test --lib sstable`:

```text
running 1 test
test sstable::tests::write_then_get_every_kind_of_key ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 27 filtered out; finished in 0.02s
```

Тест проверяет первый и последний ключ, надгробие, ключ между двумя ключами (он попал бы в блок, но его там нет) и ключи
вне таблицы. Это граничные случаи любого поиска в отсортированных данных.

## 4. LSM-хранилище

Теперь хранилище, которое связывает memtable и SSTable. Создайте `src/lsm.rs` и добавьте `pub mod lsm;` в `src/lib.rs`:

```rust
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
    next_id: u64,
}

impl Lsm {
    /// Opens (or creates) a store in `dir`. The memtable is flushed when it holds
    /// about `memtable_limit` bytes of keys and values.
    pub fn open(dir: &Path, memtable_limit: usize) -> io::Result<Lsm> {
        fs::create_dir_all(dir)?;
        let mut ids = Vec::new();
        for entry in fs::read_dir(dir)? {
            let name = entry?.file_name().to_string_lossy().into_owned();
            if let Some(id) = name
                .strip_suffix(".sst")
                .and_then(|n| n.parse::<u64>().ok())
            {
                ids.push(id);
            }
        }
        ids.sort_unstable_by(|a, b| b.cmp(a)); // newest (largest id) first
        let mut tables = Vec::new();
        for id in &ids {
            tables.push(SsTable::open(&dir.join(format!("{id:06}.sst")))?);
        }
        Ok(Lsm {
            dir: dir.to_path_buf(),
            memtable: BTreeMap::new(),
            memtable_bytes: 0,
            memtable_limit,
            tables,
            next_id: ids.first().map_or(1, |newest| newest + 1),
        })
    }

    pub fn put(&mut self, key: &[u8], value: &[u8]) -> io::Result<()> {
        self.write(key, Some(value.to_vec()))
    }

    pub fn delete(&mut self, key: &[u8]) -> io::Result<()> {
        self.write(key, None) // a tombstone
    }

    fn write(&mut self, key: &[u8], value: Value) -> io::Result<()> {
        self.memtable_bytes += key.len() + value.as_ref().map_or(0, |v| v.len());
        self.memtable.insert(key.to_vec(), value);
        if self.memtable_bytes >= self.memtable_limit {
            self.flush()?;
        }
        Ok(())
    }

    /// Writes the memtable to a new SSTable and empties it.
    pub fn flush(&mut self) -> io::Result<()> {
        if self.memtable.is_empty() {
            return Ok(());
        }
        let path = self.dir.join(format!("{:06}.sst", self.next_id));
        SsTable::write(&path, self.memtable.iter())?;
        self.tables.insert(0, SsTable::open(&path)?); // the newest table goes first
        self.next_id += 1;
        self.memtable.clear();
        self.memtable_bytes = 0;
        Ok(())
    }

    pub fn get(&self, key: &[u8]) -> io::Result<Option<Vec<u8>>> {
        if let Some(value) = self.memtable.get(key) {
            return Ok(value.clone()); // newest data; None means deleted
        }
        for table in &self.tables {
            if let Some(value) = table.get(key)? {
                return Ok(value); // the first (newest) table that knows the key wins
            }
        }
        Ok(None)
    }

    pub fn table_count(&self) -> usize {
        self.tables.len()
    }

    /// Total number of blocks read from all SSTables so far.
    pub fn block_reads(&self) -> u64 {
        self.tables.iter().map(|t| t.block_reads.get()).sum()
    }
}
```

**Что происходит:**

1. SSTable называются растущим шестизначным номером (`000001.sst`), поэтому больший номер означает более новые данные.
   `open` собирает номера из каталога и открывает таблицы от новейшей. Оставшиеся после сбоя во время сброса файлы `.tmp`
   игнорируются.
2. И `put`, и `delete` идут через `write`: удаление — это просто запись `None`. `memtable_bytes` — грубая оценка размера,
   по которой решается, когда сбрасывать.
3. `flush` записывает memtable как новую таблицу, ставит её в начало `tables` и начинает новую memtable. Запись memtable —
   одна последовательная запись отсортированных данных, в каком бы случайном порядке ни шли вызовы put.
4. `get` в точности следует пути чтения из раздела 1. `table.get` возвращает `Some(...)`, как только таблица знает ключ, и
   этот ответ — значение или надгробие — окончателен.

Два теста в конце `src/lsm.rs`:

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn newest_value_wins_across_memtable_and_tables() -> io::Result<()> {
        let dir = tempfile::tempdir()?;
        let mut db = Lsm::open(dir.path(), 1_000_000)?;
        db.put(b"apple", b"red")?;
        db.put(b"kiwi", b"green")?;
        db.flush()?; // table 1
        db.put(b"apple", b"yellow")?;
        db.delete(b"kiwi")?;
        db.flush()?; // table 2
        db.put(b"plum", b"purple")?; // still in the memtable

        assert_eq!(db.table_count(), 2);
        assert_eq!(db.get(b"apple")?, Some(b"yellow".to_vec())); // table 2 shadows table 1
        assert_eq!(db.get(b"kiwi")?, None); // the tombstone in table 2 hides table 1
        assert_eq!(db.get(b"plum")?, Some(b"purple".to_vec()));
        Ok(())
    }

    #[test]
    fn flushed_data_survives_a_reopen() -> io::Result<()> {
        let dir = tempfile::tempdir()?;
        let mut db = Lsm::open(dir.path(), 100)?; // tiny limit: many flushes
        for i in 0..50 {
            db.put(format!("key{i:02}").as_bytes(), b"0123456789")?;
        }
        db.flush()?;
        let tables = db.table_count();
        drop(db);

        let db = Lsm::open(dir.path(), 100)?;
        assert_eq!(db.table_count(), tables);
        assert_eq!(db.get(b"key07")?, Some(b"0123456789".to_vec()));
        assert_eq!(db.get(b"key99")?, None);
        Ok(())
    }
}
```

`cargo test --lib lsm`:

```text
running 2 tests
test lsm::tests::newest_value_wins_across_memtable_and_tables ... ok
test lsm::tests::flushed_data_survives_a_reopen ... ok

test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 28 filtered out; finished in 0.09s
```

## 5. Цена: усиление чтения

Каждый сброс добавляет таблицу, и поиску, возможно, придётся заглянуть во все. Поместите это в `src/bin/lsm_demo.rs`:

```rust
use p17_storage::lsm::Lsm;
use std::path::Path;

const KEYS: u64 = 100_000;

fn main() -> std::io::Result<()> {
    let dir = Path::new("lsm-demo");
    let mut db = Lsm::open(dir, 256 * 1024)?; // flush every ~256 KiB

    // Write 100,000 keys in a scrambled order (7919 shares no factor with KEYS, so every key comes once).
    for i in 0..KEYS {
        let n = i * 7919 % KEYS;
        let value = format!("value of {n} {}", "-".repeat(40));
        db.put(format!("user:{n:06}").as_bytes(), value.as_bytes())?;
    }
    db.flush()?;
    println!("SSTables on disk: {}", db.table_count());

    let before = db.block_reads();
    let found = db.get(b"user:050000")?.is_some();
    println!(
        "get existing key: found = {found}, blocks read = {}",
        db.block_reads() - before
    );

    let before = db.block_reads();
    let found = db.get(b"user:050000x")?.is_some();
    println!(
        "get missing key:  found = {found}, blocks read = {}",
        db.block_reads() - before
    );

    std::fs::remove_dir_all(dir)?;
    Ok(())
}
```

**Что происходит:**

1. `i * 7919 % KEYS` проходит каждое число от 0 до 99 999 ровно один раз, в перемешанном порядке, потому что у 7919 и
   100 000 нет общих делителей. Поэтому каждая таблица содержит ключи из всего диапазона.
2. `user:050000x` не существует, но при сортировке оказывается между `user:050000` и `user:050001`, то есть внутри
   диапазона ключей каждой таблицы.
3. `block_reads` до и после `get` показывает, сколько блоков прочитал этот один поиск.

`cargo run --release --bin lsm_demo`:

```text
SSTables on disk: 26
get existing key: found = true, blocks read = 14
get missing key:  found = false, blocks read = 26
```

Запись была дешёвой: 26 больших последовательных записей. Но чтению `user:050000` пришлось прочитать по блоку из
14 таблиц, прежде чем нашлась нужная, а отсутствующий ключ стоил чтения блока в **каждой** из 26 таблиц. Это и есть
**усиление чтения**, и с каждым сбросом оно растёт. Два лекарства — тема Урока 17.8: **компакция** сливает много таблиц в
несколько, а **фильтры Блума** позволяют поиску пропускать таблицы, в которых ключа точно нет, не читая их.

> ⚡ **Производительность:** в LSM-дереве поиск отсутствующих ключей — самый дорогой, потому что ответ «не найдено» верен
> только после последней таблицы. Многие нагрузки спрашивают отсутствующие ключи постоянно («есть ли уже такой
> пользователь?»).

## 6. Надёжность, и B-дерево против LSM-дерева

Наша memtable живёт только в RAM: сбой теряет всё, что записано после последнего сброса. Настоящие LSM-деревья ставят перед
ней WAL из Урока 17.4: каждая запись дописывается в WAL и синхронизируется, затем кладётся в memtable. После сброса данные в
безопасности в SSTable, поэтому WAL до этой точки можно удалить; при перезапуске WAL воспроизводится в свежую memtable. Если
для проекта Урока 17.14 вы выберете LSM (вариант B), то соедините их там.

Как это соотносится с B-деревом?

| | B-дерево (Урок 17.6) | LSM-дерево |
|---|---|---|
| Шаблон записи | случайный, страница на вставку, на месте | последовательный: WAL + большие отсортированные файлы |
| Чтение одного ключа | один путь по страницам, около 3–4 чтений | memtable + возможно каждая таблица (пока не помогут компакция и фильтры Блума) |
| Запрос по диапазону | пройти по листьям | слить диапазоны всех таблиц |
| Усиление записи | целая страница ради маленького изменения | данные снова переписываются каждой компакцией |
| Типичные пользователи | PostgreSQL, SQLite, MySQL InnoDB | RocksDB, LevelDB, Cassandra |

Ни одно не «лучше». B-деревья выгоднее для чтения; LSM-деревья — для записи, особенно для множества мелких случайных записей.

## Частые ошибки

1. **Искать в таблицах начиная со старой.** Первый ответ должен давать самые новые данные; иначе старое значение затенит
   более новое.
2. **Путать «удалено» с «не найдено».** Если таблица говорит «удалено», поиск должен остановиться. Вопрос к более старым
   таблицам вернул бы старое значение. Поэтому `SsTable::get` возвращает `Option<Option<…>>`.
3. **Сбрасывать неотсортированные данные.** Разреженный индекс работает, только если каждый блок отсортирован и блоки идут
   друг за другом по порядку ключей. Memtable на `BTreeMap` это гарантирует.
4. **Открывать недописанную таблицу после сбоя.** Пишите таблицы под временным именем и переименовывайте, чтобы файл
   таблицы был либо целым, либо отсутствовал.

## Попробуйте сами

1. В `lsm_demo.rs` измените предел memtable на 1 МиБ (мебибайт, 1 048 576 байтов), затем на 64 КиБ (кибибайта по 1024 байта). Как меняются число таблиц и число прочитанных
   блоков для отсутствующего ключа?
2. Запишите ключи по порядку (`let n = i;`). Сколько блоков теперь стоит отсутствующий ключ и почему?
3. Запустив демо без последнего `remove_dir_all`, посмотрите каталог через `ls -l lsm-demo`. Все ли таблицы одного
   размера?

## Упражнения

### Упражнение 1 — Надгробие после перезапуска (легко)

Напишите тест `a_tombstone_survives_flush_and_reopen`: запишите `kiwi = green`, сбросьте, удалите `kiwi`, сбросьте,
закройте, откройте заново и проверьте, что `kiwi` не найден, хотя старая таблица всё ещё содержит его значение.

<details>
<summary>Подсказка</summary>

Возьмите большой предел memtable (`1_000_000`), чтобы таблицы создавали только ваши вызовы `flush`.

</details>

<details>
<summary>Ответ</summary>

```rust
    #[test]
    fn a_tombstone_survives_flush_and_reopen() -> io::Result<()> {
        let dir = tempfile::tempdir()?;
        let mut db = Lsm::open(dir.path(), 1_000_000)?;
        db.put(b"kiwi", b"green")?;
        db.flush()?; // old table has the value
        db.delete(b"kiwi")?;
        db.flush()?; // new table has the tombstone
        drop(db);

        let db = Lsm::open(dir.path(), 1_000_000)?;
        assert_eq!(db.get(b"kiwi")?, None);
        Ok(())
    }
```

Почему это работает: после повторного открытия `open` сортирует таблицы по номеру от новейшей, поэтому таблицу с надгробием
спрашивают первой, и она отвечает «удалено». Тест упал бы, если бы `open` сортировал таблицы в обратную сторону.

</details>

### Упражнение 2 — Прочитать всё (средне)

Добавьте в `Lsm` метод `pub fn scan_all(&self) -> io::Result<BTreeMap<Vec<u8>, Vec<u8>>>`: все живые ключи с их
самыми новыми значениями, слитые из всех таблиц и memtable, без удалённых ключей. Проверьте так: put `a = 1`, `b = 1`,
сброс, put `b = 2`, delete `a`, сброс, put `c = 3`. Результат должен быть `{b: 2, c: 3}`.

<details>
<summary>Подсказка</summary>

Вставляйте всё в один `BTreeMap<Vec<u8>, Value>`, начиная с **самой старой** таблицы, чтобы новые данные перезаписывали
старые; memtable идёт последней. В конце оставьте только записи, у которых значение `Some`.

</details>

<details>
<summary>Ответ</summary>

В `impl Lsm`:

```rust
    /// Exercise 2: every live key and value, merged from all tables and the memtable.
    pub fn scan_all(&self) -> io::Result<BTreeMap<Vec<u8>, Vec<u8>>> {
        let mut merged: BTreeMap<Vec<u8>, Value> = BTreeMap::new();
        for table in self.tables.iter().rev() {
            // oldest first, so that newer tables overwrite older ones
            for (key, value) in table.scan()? {
                merged.insert(key, value);
            }
        }
        for (key, value) in &self.memtable {
            merged.insert(key.clone(), value.clone());
        }
        Ok(merged
            .into_iter()
            .filter_map(|(key, value)| value.map(|v| (key, v)))
            .collect())
    }
```

Тест:

```rust
    #[test]
    fn scan_all_merges_everything() -> io::Result<()> {
        let dir = tempfile::tempdir()?;
        let mut db = Lsm::open(dir.path(), 1_000_000)?;
        db.put(b"a", b"1")?;
        db.put(b"b", b"1")?;
        db.flush()?;
        db.put(b"b", b"2")?;
        db.delete(b"a")?;
        db.flush()?;
        db.put(b"c", b"3")?; // memtable only

        let all = db.scan_all()?;
        let expected = BTreeMap::from([
            (b"b".to_vec(), b"2".to_vec()),
            (b"c".to_vec(), b"3".to_vec()),
        ]);
        assert_eq!(all, expected);
        Ok(())
    }
```

`cargo test --lib lsm` (вместе с упражнением 1):

```text
running 4 tests
test lsm::tests::a_tombstone_survives_flush_and_reopen ... ok
test lsm::tests::newest_value_wins_across_memtable_and_tables ... ok
test lsm::tests::scan_all_merges_everything ... ok
test lsm::tests::flushed_data_survives_a_reopen ... ok

test result: ok. 4 passed; 0 failed; 0 ignored; 0 measured; 28 filtered out; finished in 0.12s
```

Почему это работает: вставка от старого к новому — это затенение, записанное циклом. Надгробие (`None`) перезаписывает
старое значение и затем отбрасывается `filter_map`. Это слияние читает все данные в память; Урок 17.8 сливает
отсортированные таблицы, не делая этого.

</details>

### Упражнение 3 — Пропускать таблицы по последнему ключу (сложно)

`SsTable::get` уже возвращается без чтения, когда ключ меньше первого ключа таблицы. Сделайте так, чтобы он возвращался без
чтения и тогда, когда ключ **больше последнего ключа таблицы**. Найдите последний ключ один раз в `open` (сохраните его в
новом поле `last_key`). Проверьте на таблице из `b` и `d`: `get(b"z")` и `get(b"a")` не читают ни одного блока,
`get(b"c")` читает один.

<details>
<summary>Подсказка</summary>

Последний ключ — это последняя запись последнего блока. В `open` прочитайте блок от последнего смещения индекса до
`data_end` и декодируйте его записи; ключ последней из них и есть последний ключ.

</details>

<details>
<summary>Ответ</summary>

Добавьте поле `last_key: Vec<u8>,` после `data_end` в `struct SsTable`. В `open` замените последний `Ok(SsTable { … })`
на:

```rust
        // Exercise 3: find the last key by decoding the last block once.
        let mut last_key = Vec::new();
        if let Some((_, start)) = index.last() {
            let mut block = vec![0u8; (data_end - start) as usize];
            file.read_exact_at(&mut block, *start)?;
            let mut pos = 0;
            while pos < block.len() {
                let (rec, used) =
                    record::decode(&block[pos..]).map_err(|_| invalid("bad record"))?;
                last_key = rec.key;
                pos += used;
            }
        }
        Ok(SsTable {
            file,
            index,
            data_end,
            last_key,
            block_reads: Cell::new(0),
        })
```

В `get` измените первую проверку на:

```rust
        if n == 0 || key > self.last_key.as_slice() {
            return Ok(None); // outside the key range of the table: no read at all
        }
```

Тест:

```rust
    #[test]
    fn keys_after_the_last_key_need_no_read() -> io::Result<()> {
        let dir = tempfile::tempdir()?;
        let path = dir.path().join("000001.sst");
        let entries = BTreeMap::from([
            (b"b".to_vec(), Some(b"1".to_vec())),
            (b"d".to_vec(), Some(b"2".to_vec())),
        ]);
        SsTable::write(&path, entries.iter())?;
        let table = SsTable::open(&path)?;
        assert_eq!(table.get(b"z")?, None);
        assert_eq!(table.get(b"a")?, None);
        assert_eq!(table.block_reads.get(), 0);
        assert_eq!(table.get(b"c")?, None); // inside the range: one block read
        assert_eq!(table.block_reads.get(), 1);
        Ok(())
    }
```

`cargo test --lib sstable` показывает `2 passed`. Почему это работает: отсортированная таблица точно знает свой диапазон
ключей, поэтому ключи вне `[first, last]` не могут в ней быть. Запустите `lsm_demo` снова: он по-прежнему печатает 26 блоков
для отсутствующего ключа, потому что `user:050000x` лежит *внутри* диапазона каждой таблицы. Диапазоны ключей помогают,
когда таблицы покрывают разные диапазоны (например, после компакции по уровням); для ключей внутри диапазона нужны фильтры
Блума из Урока 17.8.

</details>

## Вопросы для самопроверки

1. Куда в LSM-дереве сначала идёт новая запись и когда она попадает в SSTable?

<details>
<summary>Ответ</summary>

В WAL (ради надёжности) и в memtable. В SSTable она попадает, когда memtable заполнена и сбрасывается.

</details>

2. Почему LSM-дерево пишет только последовательно, даже когда ключи приходят в случайном порядке?

<details>
<summary>Ответ</summary>

Memtable сортирует ключи в памяти. Записи на диск — это только дописывания в WAL и целые SSTable, записанные за один раз;
ничего на диске не меняется на месте.

</details>

3. У ключа есть значение в таблице 1 (старой) и надгробие в таблице 3 (новой). Что вернёт `get` и почему он должен
   остановиться на таблице 3?

<details>
<summary>Ответ</summary>

«Не найдено». Надгробие в более новой таблице затеняет старое значение; если бы `get` пошёл дальше к таблице 1, удалённое
значение вернулось бы.

</details>

4. Что такое разреженный индекс и сколько он стоит на один поиск?

<details>
<summary>Ответ</summary>

Индекс с одной записью на блок (здесь каждые 16 записей), а не на каждую запись. Он в 16 раз меньше, а поиск должен прочитать
и просмотреть целый блок, а не ровно одну запись.

</details>

5. Почему поиск отсутствующего ключа — самый дорогой в LSM-дереве?

<details>
<summary>Ответ</summary>

«Не найдено» верно только после проверки memtable и каждой таблицы, поэтому поиск может прочитать блок в каждой таблице.

</details>

6. Назовите одно преимущество B-дерева перед LSM-деревом и одно в обратную сторону.

<details>
<summary>Ответ</summary>

B-дерево читает ключ по одному пути страниц, не спрашивая много файлов. LSM-дерево пишет только последовательно и гораздо
быстрее справляется с множеством мелких случайных записей.

</details>

## Итоги

- LSM-дерево собирает записи в отсортированной memtable (за WAL) и сбрасывает её в неизменяемую отсортированную SSTable.
- Чтение спрашивает memtable, затем SSTable от новой к старой; побеждает первый ответ (затенение).
- Удаления — надгробия; «удалено» должно останавливать поиск, поэтому у поиска в таблице три ответа.
- Наша SSTable — это записи по порядку ключей, разреженный индекс каждого 16-го ключа и 8-байтовый подвал; поиск читает один
  блок.
- Таблицы пишутся под временным именем и переименовываются, поэтому сбой никогда не оставляет половину таблицы.
- Усиление чтения растёт с каждым сбросом: 26 таблиц дали 26 чтений блоков ради одного отсутствующего ключа.
- B-деревья выгоднее для чтения, LSM-деревья — для записи; компакция и фильтры Блума (Урок 17.8) снижают цену чтения в LSM.

## Источники и что почитать

- [The Log-Structured Merge-Tree (LSM-Tree)](https://www.cs.umb.edu/~poneil/lsmtree.pdf) — исходная статья 1996 года
  О'Нила и соавторов, которая ввела эту идею.
- [LevelDB implementation notes](https://github.com/google/leveldb/blob/main/doc/impl.md) — memtable, журнал, файлы SSTable
  и компакция в настоящем LSM-дереве на нескольких страницах.
- [RocksDB Overview](https://github.com/facebook/rocksdb/wiki/RocksDB-Overview) — архитектура самого распространённого
  LSM-движка.
- [Mini-LSM](https://skyzh.github.io/mini-lsm/) — бесплатный курс, который шаг за шагом строит полный движок хранения на
  LSM-дереве на Rust: естественный следующий шаг после этого этапа.
- [Методы срезов Rust](https://doc.rust-lang.org/std/primitive.slice.html) — `partition_point` и `binary_search`, которыми
  пользуется поиск в SSTable.

---

[🏠 Начало](../START-HERE.md) · [← 17.6 B-деревья](06-b-trees.md) · [📋 Этап 17](README.md) · [17.8 Компакция и фильтры Блума →](08-compaction-and-bloom-filters.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
