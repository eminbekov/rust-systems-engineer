<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 17.4 Журнал упреждающей записи (WAL)](04-write-ahead-log.md) · [📋 Этап 17](README.md) · [17.6 B-деревья →](06-b-trees.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/17-storage-and-databases/05-hash-indexes.md) <!-- nav -->

# Урок 17.5 — Индексы: хеш-индекс

> **Этап 17 — Как устроены хранилища и базы данных** · Урок 5 из 15 · ⏱ около 135 минут

## Что вы узнаете

- Что такое индекс и почему базе данных без него приходится читать всё, чтобы найти что-нибудь.
- Как устроено хранилище со структурой журнала и хеш-индексом в памяти (устройство Bitcask): значения на диске, в RAM (Random Access Memory, оперативная память)
  только ключи и позиции.
- Как поиск превращается в одно чтение по смещению и как удаления превращаются в *надгробия* (tombstone).
- Как измерить разницу между поиском с индексом и без него.
- Пределы хеш-индекса: все ключи в RAM, нет запросов по диапазону, медленный запуск и файл, который всё растёт.

## Перед началом

- Вы прошли [Урок 17.4 — Журнал упреждающей записи (WAL)](04-write-ahead-log.md). Этот урок использует его формат записей
  (запись + байт типа) и его восстановление.
- Вы знаете хеш-таблицы из [Урока 07.6 — Как работают хеш-таблицы](../07-algorithms-and-data-structures/06-hash-tables.md),
  чтение по смещению (`read_exact_at`) из [Урока 17.1](01-how-disks-and-files-work.md) и индексы SQL (Structured Query Language, язык структурированных запросов) из
  [Урока 16.5 — Основы SQL и PostgreSQL](../16-backend-services/05-sql-and-postgresql-basics.md).

## Новые слова

| Слово | Значение |
|---|---|
| **индекс** (index) | Дополнительная структура данных, которая говорит, *где* лежат данные, чтобы не искать по всем. Как предметный указатель в конце книги. |
| **полный просмотр** (full scan) | Поиск данных чтением их всех от начала до конца. |
| **хранение со структурой журнала** (log-structured storage) | Устройство, при котором сам файл данных — журнал, в который только дописывают: ничего никогда не меняется на месте. |
| **хеш-индекс** (hash index) | Индекс, хранящийся в хеш-таблице: ключ → позиция последней записи в файле. |
| **смещение** (offset) | Позиция байта в файле, считая от 0 (вы знаете его из Урока 13.4). |
| **надгробие** (tombstone) | Особая запись, которая говорит «этот ключ удалён». Из журнала нельзя убрать старые записи, поэтому вместо этого добавляют надгробие. |
| **устаревшие данные** (stale data) | Записи в файле, которые больше не являются последней версией своего ключа (перезаписаны или удалены). Они занимают место, но их никогда не читают. |
| **запрос по диапазону** (range query) | Вопрос о диапазоне ключей, например «все пользователи от `user:000100` до `user:000199`». |

## 1. Зачем индекс?

Хранилище из Урока 17.4 держало каждое значение в `HashMap` в RAM. Это быстро, но RAM маленькая и дорогая по сравнению
с SSD (твердотельным накопителем). Настоящее хранилище должно держать значения на диске. Тогда появляется новый вопрос: где в файле лежит значение
для данного ключа?

Без помощи единственный ответ — **полный просмотр**: читать файл с начала, декодировать каждую запись и запоминать последнюю
с этим ключом (последнюю, потому что поздние записи перекрывают ранние). Для файла в 25 МБ (мегабайт) это значит прочитать 25 МБ, чтобы
найти 100 байт, — и так при каждом поиске.

**Индекс** — это и есть помощь. Это предметный указатель в конце книги: «fsync … стр. 214». Вы находите слово в маленьком
отсортированном списке и сразу переходите на страницу. Индекс хранилища сопоставляет каждому ключу **смещение** его записи
в файле. В SQL вы встречали индексы в Уроке 16.5 как `CREATE INDEX`; теперь вы построите индекс сами.

Простейший индекс — хеш-таблица. Такое устройство называется **Bitcask** (по движку хранения базы данных Riak, который его
прославил):

```text
 RAM: hash index                     disk: data.log (append-only)
 ┌──────────────┬─────────────────┐  offset
 │ "apple"      │ offset 43, 24 B │     0  [ put apple = red    ]  ← stale
 │ "kiwi"       │ offset 21, 22 B │    21  [ put kiwi  = green  ]
 └──────────────┴─────────────────┘    43  [ put apple = yellow ]
```

- **Запись** дописывает запись в конец файла (быстро, последовательно) и обновляет индекс.
- **Чтение** находит ключ в индексе и делает **одно чтение по смещению** ровно нужных байтов.
- **Значения остаются на диске.** В RAM только ключи и по два числа на ключ.

## 2. Журнал с хеш-индексом

Создайте `src/hashlog.rs` и добавьте `pub mod hashlog;` в `src/lib.rs`:

```rust
use crate::record;
use std::collections::HashMap;
use std::fs::{File, OpenOptions};
use std::io::{self, Write};
use std::os::unix::fs::FileExt;
use std::path::Path;

const PUT: u8 = 1;
const DELETE: u8 = 0;

/// Where a record lives in the data file.
#[derive(Debug, Clone, Copy)]
struct Pos {
    offset: u64,
    len: u32,
}

/// A log-structured key-value store with an in-memory hash index:
/// values live only in the file, the index maps each key to its latest record.
pub struct HashLog {
    file: File,
    index: HashMap<Vec<u8>, Pos>,
    end: u64,
}

impl HashLog {
    pub fn open(path: &Path) -> io::Result<HashLog> {
        let file = OpenOptions::new()
            .create(true)
            .read(true)
            .append(true)
            .open(path)?;
        let bytes = std::fs::read(path)?;

        let mut index = HashMap::new();
        let mut pos = 0;
        while let Ok((rec, used)) = record::decode(&bytes[pos..]) {
            let here = Pos {
                offset: pos as u64,
                len: used as u32,
            };
            match rec.value.first() {
                Some(&PUT) => {
                    index.insert(rec.key, here);
                }
                Some(&DELETE) => {
                    index.remove(&rec.key);
                }
                _ => {}
            }
            pos += used;
        }
        if pos < bytes.len() {
            file.set_len(pos as u64)?; // torn tail, exactly as in the WAL
            file.sync_all()?;
        }
        Ok(HashLog {
            file,
            index,
            end: pos as u64,
        })
    }
```

**Что происходит:**

1. `Pos` — то, что индекс хранит для каждого ключа: где начинается запись и какой она длины. Зная длину, чтению не нужно
   ничего угадывать.
2. `open` один раз просматривает файл, как воспроизведение из Урока 17.4. Но вместо значений хранит только позицию
   последней записи каждого ключа.
3. `while let Ok(...) = ...` работает, пока декодирование успешно, и останавливается на первой ошибке: на конце файла или
   на разорванном хвосте. Это более короткая запись `match` из Урока 17.4.
4. Запись удаления убирает ключ из индекса. Разорванный хвост отрезается ровно так же, как в WAL.
5. `end` помнит, где кончается файл, чтобы `put` знал смещение дописываемой записи.

Затем методы, всё ещё внутри `impl HashLog`:

```rust
    /// Number of live keys.
    pub fn len(&self) -> usize {
        self.index.len()
    }

    pub fn is_empty(&self) -> bool {
        self.index.is_empty()
    }

    pub fn get(&self, key: &[u8]) -> io::Result<Option<Vec<u8>>> {
        let Some(pos) = self.index.get(key) else {
            return Ok(None); // not in the index: no disk access at all
        };
        let mut buf = vec![0u8; pos.len as usize];
        self.file.read_exact_at(&mut buf, pos.offset)?; // one positioned read
        let (rec, _) =
            record::decode(&buf).map_err(|e| io::Error::new(io::ErrorKind::InvalidData, e))?;
        Ok(Some(rec.value[1..].to_vec()))
    }

    pub fn put(&mut self, key: &[u8], value: &[u8]) -> io::Result<()> {
        let mut entry = vec![PUT];
        entry.extend_from_slice(value);
        let pos = self.append(key, &entry)?;
        self.index.insert(key.to_vec(), pos);
        Ok(())
    }

    pub fn delete(&mut self, key: &[u8]) -> io::Result<()> {
        if self.index.contains_key(key) {
            self.append(key, &[DELETE])?; // a tombstone
            self.index.remove(key);
        }
        Ok(())
    }

    fn append(&mut self, key: &[u8], entry: &[u8]) -> io::Result<Pos> {
        let mut buf = Vec::new();
        record::encode(key, entry, &mut buf);
        self.file.write_all(&buf)?;
        self.file.sync_data()?;
        let pos = Pos {
            offset: self.end,
            len: buf.len() as u32,
        };
        self.end += buf.len() as u64;
        Ok(pos)
    }
}
```

**Что происходит:**

1. `get` использует `let … else` (Урок 04.5): если ключа нет в индексе, ответ «не найдено» получается без обращения к
   диску. Иначе `read_exact_at` читает ровно `pos.len` байт по смещению `pos.offset` — один системный вызов.
2. Запись снова декодируется, поэтому контрольная сумма проверяется при каждом чтении: повреждённый байт на диске становится
   ошибкой, а не неверным значением. `map_err` превращает наш `DecodeError` в `io::Error` вида `InvalidData`.
3. `rec.value[1..]` пропускает байт типа.
4. `put` следует правилу упреждающей записи: сначала дописать и синхронизировать, потом обновить индекс.
5. `delete` не может стереть старую запись, потому что в файл только дописывают. Он дописывает **надгробие** — запись,
   которая говорит «удалено». Без неё следующий `open` нашёл бы старый put и вернул бы ключ к жизни. Удаление
   несуществующего ключа ничего не пишет.
6. Clippy требует `is_empty` всякий раз, когда у типа есть публичный `len`; это соглашение Rust.

> 📏 **Соглашение:** тип с `pub fn len(&self)` получает и `pub fn is_empty(&self)`. Об этом напоминает lint Clippy
> `len_without_is_empty`.

## 3. Тестируем

Добавьте тесты в конец `src/hashlog.rs`:

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn get_reads_the_latest_value_from_disk() -> io::Result<()> {
        let dir = tempfile::tempdir()?;
        let path = dir.path().join("data.log");
        let mut db = HashLog::open(&path)?;
        db.put(b"apple", b"red")?;
        db.put(b"kiwi", b"green")?;
        db.put(b"apple", b"yellow")?;
        assert_eq!(db.get(b"apple")?, Some(b"yellow".to_vec()));
        assert_eq!(db.get(b"plum")?, None);
        assert_eq!(db.len(), 2);
        Ok(())
    }

    #[test]
    fn the_index_is_rebuilt_on_open() -> io::Result<()> {
        let dir = tempfile::tempdir()?;
        let path = dir.path().join("data.log");
        let mut db = HashLog::open(&path)?;
        db.put(b"apple", b"red")?;
        db.put(b"kiwi", b"green")?;
        db.delete(b"kiwi")?;
        db.put(b"apple", b"yellow")?;
        drop(db);

        let db = HashLog::open(&path)?;
        assert_eq!(db.get(b"apple")?, Some(b"yellow".to_vec()));
        assert_eq!(db.get(b"kiwi")?, None);
        assert_eq!(db.len(), 1);
        Ok(())
    }
}
```

`cargo test --lib hashlog`:

```text
running 2 tests
test hashlog::tests::get_reads_the_latest_value_from_disk ... ok
test hashlog::tests::the_index_is_rebuilt_on_open ... ok

test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 17 filtered out; finished in 0.03s
```

Главный здесь второй тест: после перезапуска `kiwi` остаётся удалённым только благодаря надгробию.

## 4. Измеряем индекс

Насколько помогает индекс? Поместите это в `src/bin/hash_index.rs`. Программа строит файл с 200 000 записей, а затем
сравнивает поиск через индекс с полным просмотром:

```rust
use p17_storage::{hashlog::HashLog, record};
use std::fs::{self, File};
use std::io::{BufWriter, Write};
use std::path::Path;
use std::time::Instant;

const KEYS: usize = 200_000;

fn key(i: usize) -> String {
    format!("user:{i:06}")
}

/// Finds a key the slow way: decode every record of the file and keep the last match.
fn scan_for(bytes: &[u8], wanted: &[u8]) -> Option<Vec<u8>> {
    let mut pos = 0;
    let mut found = None;
    while let Ok((rec, used)) = record::decode(&bytes[pos..]) {
        if rec.key == wanted {
            found = Some(rec.value[1..].to_vec());
        }
        pos += used;
    }
    found
}

fn main() -> std::io::Result<()> {
    let path = Path::new("bench.log");

    // 1. Build a data file quickly: one BufWriter, no sync per record.
    let mut out = BufWriter::new(File::create(path)?);
    let mut buf = Vec::new();
    for i in 0..KEYS {
        buf.clear();
        let mut entry = vec![1u8]; // put
        entry.extend_from_slice(format!("profile of user {i}, {}", "x".repeat(80)).as_bytes());
        record::encode(key(i).as_bytes(), &entry, &mut buf);
        out.write_all(&buf)?;
    }
    out.into_inner()?.sync_all()?;
    println!("data file: {} MB", fs::metadata(path)?.len() / 1_000_000);

    // 2. Open = scan the whole file once and build the index.
    let start = Instant::now();
    let db = HashLog::open(path)?;
    println!(
        "open (rebuild index of {} keys): {:?}",
        db.len(),
        start.elapsed()
    );

    // 3. Look up 10,000 keys through the index.
    let start = Instant::now();
    for i in (0..KEYS).step_by(KEYS / 10_000) {
        assert!(db.get(key(i).as_bytes())?.is_some());
    }
    println!("get with index:    {:?} per key", start.elapsed() / 10_000);

    // 4. One lookup without an index: read and decode the whole file.
    let start = Instant::now();
    let bytes = fs::read(path)?;
    assert!(scan_for(&bytes, key(KEYS - 1).as_bytes()).is_some());
    println!("get without index: {:?} per key", start.elapsed());

    fs::remove_file(path)?;
    Ok(())
}
```

**Что происходит:**

1. Чтобы быстро построить тестовый файл, программа пишет записи через `BufWriter` с одной синхронизацией в конце, а не
   200 000 вызовов `put` с 200 000 синхронизаций (по 5 мс это заняло бы около 17 минут).
2. `{i:06}` дополняет число нулями до шести цифр: `user:000042`.
3. `step_by(KEYS / 10_000)` берёт каждый 20-й ключ, всего 10 000 поисков.
4. `scan_for` — это хранилище без индекса: оно должно декодировать каждую запись, потому что последняя версия ключа может
   быть где угодно.

`cargo run --release --bin hash_index`:

```text
data file: 25 MB
open (rebuild index of 200000 keys): 23.47975ms
get with index:    731ns per key
get without index: 9.314375ms per key
```

С индексом поиск занимает меньше микросекунды: один поиск в хеш-таблице и один `pread` примерно 125 байт из страничного
кэша. Без индекса — 9 мс: более чем в десять тысяч раз медленнее, и это время растёт вместе с файлом. Цена индекса — 23 мс
при запуске, когда `open` один раз просматривает файл.

> ⚡ **Производительность:** файл был всё ещё в страничном кэше (его только что записали), поэтому `get` не ждал SSD. С
> холодным кэшем каждый `get` стоил бы одного чтения с SSD (десятки микросекунд, Урок 17.1) — но всё равно одного чтения,
> а не просмотра.

## 5. Пределы хеш-индекса

Bitcask прост и очень быстр, и настоящие системы им пользуются. Но посмотрите, чего он не умеет:

1. **Все ключи должны помещаться в RAM.** Индекс хранит каждый ключ плюс около 16 байт `Pos` и собственные накладные
   расходы хеш-таблицы. При 100 миллионах ключей по 20 байт это несколько гигабайт RAM.
2. **Нет запросов по диапазону.** У хеш-таблицы нет порядка (Урок 07.6). «Все ключи от `user:000100` до `user:000199`»
   означает проверку каждого ключа в индексе.
3. **Медленный запуск.** `open` должен просмотреть весь файл. Для 25 МБ это заняло 23 мс; для 100 ГБ (гигабайт) заняло бы минуты.
   Bitcask решает это *hint-файлами*: небольшим файлом только с индексом, который пишется при закрытии файла данных.
4. **Файл только растёт.** Каждая перезапись и каждое удаление оставляют **устаревшие данные**. Ключ, обновлённый миллион
   раз, занимает место миллиона записей. Ответ — компакция, как в упражнении 3 Урока 17.4; Урок 17.8 делает её сразу для
   многих файлов.

Следующие два урока снимают пределы 1 и 2 двумя разными способами. **B-дерево** (Урок 17.6) держит ключи отсортированными
в страницах на диске, так что сам индекс живёт на диске и поддерживает диапазоны. **LSM-дерево** (Log-Structured Merge tree, Урок 17.7) сохраняет
идею «только дописывать», но пишет **отсортированные** файлы.

## Частые ошибки

1. **Забыть надгробие.** Удаление ключа только из индекса работает до следующего перезапуска; потом старая запись
   возвращает ключ.
2. **Начинать `end` с 0 после повторного открытия.** Новые записи дописываются в настоящий конец файла, а индекс указывал бы
   на смещение 0. Упражнение 2 проверяет именно это.
3. **Доверять чтению без контрольной суммы.** Чтение по смещению может вернуть повреждённые байты; декодируйте и проверяйте
   их каждый раз.
4. **Мерить только с тёплым кэшем.** 731 нс — это страничный кэш; говорите об этом, когда сообщаете такие числа.

## Попробуйте сами

1. Замените `KEYS` на 1 000 000 в `hash_index.rs`. Как меняется время открытия и время просмотра? Сильно ли меняется
   `get` с индексом?
2. В `hash_index.rs` после построения файла перезапишите каждый ключ ещё раз, затем измерьте размер файла. Какая его часть
   устарела?
3. Напишите тест, который удаляет ключ, снова его записывает, открывает хранилище заново и проверяет, что новое значение на
   месте.

## Упражнения

### Упражнение 1 — Сколько RAM? (легко)

В хранилище 50 миллионов ключей, в среднем по 24 байта. Каждой записи индекса нужны сам ключ, 16 байт на `Pos` и ещё
примерно 32 байта накладных расходов (заголовок `Vec` и пустые ячейки хеш-таблицы). Оцените RAM для индекса. Поместится
ли он на сервере с 4 ГБ RAM?

<details>
<summary>Подсказка</summary>

Байты на запись × число ключей. 1 ГБ — это около 10⁹ байт.

</details>

<details>
<summary>Ответ</summary>

24 + 16 + 32 = 72 байта на ключ; 72 × 50 000 000 = 3,6 × 10⁹ байт, около 3,6 ГБ. Поместится с трудом, почти не оставив
места страничному кэшу и программе. Это предел 1 из раздела 5: хеш-индексу нужна RAM пропорционально числу ключей,
каким бы ни был размер значений.

</details>

### Упражнение 2 — Запись после повторного открытия (средне)

Напишите тест `writes_after_reopen_land_at_the_end`: запишите `apple = red`, закройте, откройте заново, запишите
`kiwi = green`, удалите отсутствующий ключ `plum`. Проверьте, что оба значения читаются и что файл занимает ровно
21 + 22 байта (никакого надгробия для `plum`).

<details>
<summary>Подсказка</summary>

`std::fs::metadata(&path)?.len()` даёт размер файла. Запись apple — это 12 + 5 + 1 + 3 байта.

</details>

<details>
<summary>Ответ</summary>

```rust
    #[test]
    fn writes_after_reopen_land_at_the_end() -> io::Result<()> {
        let dir = tempfile::tempdir()?;
        let path = dir.path().join("data.log");
        let mut db = HashLog::open(&path)?;
        db.put(b"apple", b"red")?;
        drop(db);

        let mut db = HashLog::open(&path)?;
        db.put(b"kiwi", b"green")?;
        db.delete(b"plum")?; // not there: must write nothing
        assert_eq!(db.get(b"apple")?, Some(b"red".to_vec()));
        assert_eq!(db.get(b"kiwi")?, Some(b"green".to_vec()));
        assert_eq!(std::fs::metadata(&path)?.len(), 21 + 22);
        Ok(())
    }
```

Почему это работает: если бы `open` ставил `end` в 0 вместо длины файла, запись kiwi всё равно дописалась бы по смещению 21
(режим дописывания), но индекс сказал бы «смещение 0», и `get(b"kiwi")` прочитал бы запись apple. Тест ловит эту ошибку, а
проверка размера — лишнее надгробие.

</details>

### Упражнение 3 — Посчитайте устаревшие байты (сложно)

Хранилище должно делать компакцию, когда, скажем, половина его файла устарела. Добавьте поле `stale: u64` и метод
`pub fn stale_bytes(&self) -> u64`, который возвращает, сколько байтов принадлежит перезаписанным или удалённым записям
(надгробие тоже считайте устаревшим: это не живые данные). Поддерживайте счётчик правильным в `put`, `delete` **и** `open`.
Проверьте такой последовательностью: put `apple = red` (21 байт), put `apple = green` (23 байта), put `kiwi = x`
(18 байт), delete `kiwi` (надгробие в 17 байт); результат должен быть 21 + 18 + 17 и до, и после повторного открытия.

<details>
<summary>Подсказка</summary>

`HashMap::insert` возвращает старое значение, `Some(old)`, если ключ был; `remove` тоже. В этих случаях прибавляйте
`old.len` к `stale`. В `delete` сначала скопируйте старый `Pos` через `self.index.get(key).copied()`.

</details>

<details>
<summary>Ответ</summary>

Изменённые части `src/hashlog.rs`:

```rust
pub struct HashLog {
    file: File,
    index: HashMap<Vec<u8>, Pos>,
    end: u64,
    stale: u64,
}
```

В `open` — цикл и результат:

```rust
        let mut index: HashMap<Vec<u8>, Pos> = HashMap::new();
        let mut stale = 0;
        let mut pos = 0;
        while let Ok((rec, used)) = record::decode(&bytes[pos..]) {
            let here = Pos {
                offset: pos as u64,
                len: used as u32,
            };
            match rec.value.first() {
                Some(&PUT) => {
                    if let Some(old) = index.insert(rec.key, here) {
                        stale += old.len as u64;
                    }
                }
                Some(&DELETE) => {
                    if let Some(old) = index.remove(&rec.key) {
                        stale += old.len as u64;
                    }
                    stale += here.len as u64; // the tombstone itself is not live data
                }
                _ => {}
            }
            pos += used;
        }
```

```rust
        Ok(HashLog {
            file,
            index,
            end: pos as u64,
            stale,
        })
```

Новый метод и изменённые `put` и `delete`:

```rust
    /// Bytes in the file that belong to overwritten or deleted records.
    pub fn stale_bytes(&self) -> u64 {
        self.stale
    }
```

```rust
    pub fn put(&mut self, key: &[u8], value: &[u8]) -> io::Result<()> {
        let mut entry = vec![PUT];
        entry.extend_from_slice(value);
        let pos = self.append(key, &entry)?;
        if let Some(old) = self.index.insert(key.to_vec(), pos) {
            self.stale += old.len as u64;
        }
        Ok(())
    }

    pub fn delete(&mut self, key: &[u8]) -> io::Result<()> {
        if let Some(old) = self.index.get(key).copied() {
            let tombstone = self.append(key, &[DELETE])?; // a tombstone
            self.index.remove(key);
            self.stale += old.len as u64 + tombstone.len as u64;
        }
        Ok(())
    }
```

Тест:

```rust
    #[test]
    fn stale_bytes_count_overwrites_and_deletes() -> io::Result<()> {
        let dir = tempfile::tempdir()?;
        let path = dir.path().join("data.log");
        let mut db = HashLog::open(&path)?;
        db.put(b"apple", b"red")?; // 21 bytes, live
        db.put(b"apple", b"green")?; // 23 bytes, live; the 21 become stale
        db.put(b"kiwi", b"x")?; // 18 bytes
        db.delete(b"kiwi")?; // 17-byte tombstone; 18 + 17 stale
        assert_eq!(db.stale_bytes(), 21 + 18 + 17);
        drop(db);

        let db = HashLog::open(&path)?;
        assert_eq!(db.stale_bytes(), 21 + 18 + 17);
        Ok(())
    }
```

`cargo test --lib hashlog` (вместе с упражнением 2):

```text
running 4 tests
test hashlog::tests::writes_after_reopen_land_at_the_end ... ok
test hashlog::tests::get_reads_the_latest_value_from_disk ... ok
test hashlog::tests::the_index_is_rebuilt_on_open ... ok
test hashlog::tests::stale_bytes_count_overwrites_and_deletes ... ok

test result: ok. 4 passed; 0 failed; 0 ignored; 0 measured; 17 filtered out; finished in 0.05s
```

Почему это работает: запись становится устаревшей ровно тогда, когда индекс перестаёт на неё указывать, а `insert` и
`remove` сообщают вам этот момент. `open` повторяет тот же учёт во время просмотра, поэтому число переживает перезапуск.
По условию `stale_bytes() * 2 > длина файла` хранилище могло бы решить сделать компакцию.

</details>

## Вопросы для самопроверки

1. Что сопоставляет хеш-индекс и что остаётся вне RAM?

<details>
<summary>Ответ</summary>

Он сопоставляет каждому ключу позицию (смещение и длину) последней записи этого ключа в файле данных. Значения остаются на
диске.

</details>

2. Сколько чтений с диска нужно `get` для существующего ключа и сколько для отсутствующего?

<details>
<summary>Ответ</summary>

Одно чтение по смещению для существующего ключа; ноль для отсутствующего, потому что индекс уже говорит, что его нет.

</details>

3. Почему `delete` должен писать надгробие, а не только убирать ключ из индекса?

<details>
<summary>Ответ</summary>

Старая запись put всё ещё в файле. При следующем `open` просмотр нашёл бы её и вернул бы ключ. Надгробие, стоящее в файле
позже, сообщает просмотру, что ключ удалён.

</details>

4. Почему хеш-индекс не может эффективно ответить на «все ключи между `a` и `c`»?

<details>
<summary>Ответ</summary>

Хеш-таблица хранит ключи в порядке, который зависит от их хешей, а не от самих ключей. Чтобы найти диапазон, надо проверить
каждый ключ.

</details>

5. Измеренный `get` занял 731 нс. Почему это не скорость SSD?

<details>
<summary>Ответ</summary>

Файл только что записали, и он был в страничном кэше, поэтому `read_exact_at` копировал из RAM. Чтение с SSD заняло бы
десятки микросекунд.

</details>

6. Назовите четыре предела устройства Bitcask.

<details>
<summary>Ответ</summary>

Все ключи должны помещаться в RAM; нет запросов по диапазону; при запуске нужно просмотреть файл (если нет hint-файлов); и
файл растёт за счёт устаревших данных, пока не сделана компакция.

</details>

## Итоги

- Индекс говорит, где лежат данные; без него каждый поиск — полный просмотр.
- Хранилище в стиле Bitcask дописывает записи в журнал и держит в RAM хеш-индекс ключ → (смещение, длина).
- `get` — один поиск в хеш-таблице плюс одно чтение по смещению; отсутствующему ключу диск вообще не нужен.
- Удаления — это надгробия, потому что записи из файла, в который только дописывают, никогда не убираются.
- При открытии индекс строится заново просмотром журнала, с той же обработкой разорванного хвоста, что и в WAL.
- Измерено: меньше 1 мкс на поиск по индексу (из страничного кэша) против 9 мс на полный просмотр 25 МБ.
- Пределы: ключи в RAM, нет диапазонов, медленный запуск, устаревшие данные. B-деревья и LSM-деревья решают эти проблемы.

## Источники и что почитать

- [Bitcask: A Log-Structured Hash Table for Fast Key/Value Data](https://riak.com/assets/bitcask-intro.pdf) — короткая
  исходная статья об устройстве из этого урока, включая hint-файлы и слияние.
- [std::collections::HashMap](https://doc.rust-lang.org/std/collections/struct.HashMap.html) — `insert` и `remove`
  возвращают старое значение, чем пользуется упражнение 3.
- [std::os::unix::fs::FileExt](https://doc.rust-lang.org/std/os/unix/fs/trait.FileExt.html) — `read_exact_at`, чтение по
  смещению, на котором стоит `get`.
- [PostgreSQL documentation — Index Types](https://www.postgresql.org/docs/current/indexes-types.html) — хеш- и
  B-tree-индексы настоящей базы данных и когда используется каждый.
- [Lint-проверки Clippy](https://rust-lang.github.io/rust-clippy/stable/index.html) — найдите `len_without_is_empty`,
  соглашение из раздела 2.

---

[🏠 Начало](../START-HERE.md) · [← 17.4 Журнал упреждающей записи (WAL)](04-write-ahead-log.md) · [📋 Этап 17](README.md) · [17.6 B-деревья →](06-b-trees.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
