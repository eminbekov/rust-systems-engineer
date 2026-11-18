<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 17.3 Двоичные форматы файлов и сериализация](03-binary-file-formats.md) · [📋 Этап 17](README.md) · [17.5 Индексы: хеш-индекс →](05-hash-indexes.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/17-storage-and-databases/04-write-ahead-log.md) <!-- nav -->

# Урок 17.4 — Журнал упреждающей записи (WAL)

> **Этап 17 — Как устроены хранилища и базы данных** · Урок 4 из 15 · ⏱ около 135 минут

## Что вы узнаете

- Единственное правило, на котором стоит любая устойчивая к сбоям база данных: сначала записать изменение в журнал и
  только потом его применить.
- Как построить маленькое хранилище «ключ-значение», данные которого переживают перезапуски и сбои: `HashMap` в памяти
  плюс WAL на диске.
- Как работает восстановление: воспроизвести журнал, найти разорванный хвост и отрезать его.
- Почему, если забыть отрезать разорванный хвост, молча теряются *новые* данные.
- Почему журнал растёт бесконечно и как компакция и контрольные точки держат его маленьким.

## Перед началом

- Вы прошли [Урок 17.3 — Двоичные форматы файлов и сериализация](03-binary-file-formats.md): `src/record.rs` кодирует и
  декодирует записи с длинами и CRC (Cyclic Redundancy Check, контрольная сумма).
- Вы знаете надёжность записи, `sync_data` и разорванные записи из [Урока 17.2](02-durability-and-fsync.md), а шаблон
  атомарной замены — из [Урока 13.12](../13-operating-systems/12-file-systems-inside.md).
- Вы знаете `HashMap` ([Урок 07.6 — Как работают хеш-таблицы](../07-algorithms-and-data-structures/06-hash-tables.md)) и
  транзакции из [Урока 16.7 — Миграции и транзакции](../16-backend-services/07-migrations-and-transactions.md).
- Добавьте крейт `tempfile` для тестов. Он создаёт временный каталог, который удаляется автоматически:

```bash
cd ~/projects/rust/code/p17-storage
cargo add --dev tempfile@3
```

## Новые слова

| Слово | Значение |
|---|---|
| **WAL** (Write-Ahead Log, журнал упреждающей записи) | Файл, в который только дописывают; движок хранения записывает в него каждое изменение *до того*, как применить его где-либо ещё. |
| **запись журнала** (log entry) | Одно изменение в журнале, например «put apple = red» или «delete kiwi». |
| **воспроизведение** (replay) | Чтение журнала с начала и повторное применение каждой записи, чтобы восстановить состояние после перезапуска. |
| **восстановление** (recovery) | Всё, что движок хранения делает при запуске, чтобы после сбоя вернуться в правильное состояние: здесь — воспроизведение и отрезание разорванного хвоста. |
| **разорванный хвост** (torn tail) | Неполные или повреждённые байты в конце журнала после сбоя посреди дописывания. |
| **усечь** (truncate) | Сделать файл короче, выбросив всё после заданной длины. В Rust: `File::set_len`. |
| **компакция** (compaction) | Перезапись журнала или файла данных так, чтобы в нём остались только последние живые данные, без старых версий и удалённых ключей. |
| **контрольная точка** (checkpoint) | Момент, когда движок сохраняет полное состояние в другом месте, так что журнал до этого момента больше не нужен. |
| **dev-зависимость** (dev-dependency) | Крейт, который используют только тесты, примеры и бенчмарки, но не сама библиотека (`cargo add --dev`). |

## 1. Идея: записать заранее

Представьте лавочника, который держит количество товара в голове. Каждый раз, когда что-то продано, он сначала пишет строку
в тетрадь («продано 2 яблока») и только потом меняет число в голове. Если он уснёт и всё забудет, он может прочитать
тетрадь с первой страницы и получить количество обратно. Тетрадь растёт только в конце, и записать строку быстро.

**WAL** (журнал упреждающей записи) — это и есть такая тетрадь. Правило состоит из трёх частей:

1. **Сначала запишите изменение в журнал**, в конец файла (дописывание, а это быстро, Урок 17.1).
2. **Сделайте запись журнала надёжной** через `sync_data` (Урок 17.2).
3. **Только потом** примените изменение к настоящим данным (в памяти или в файлах данных) и скажите вызывающему
   «сохранено».

Если программа упадёт до того, как закончится шаг 2, вызывающий так и не услышал «сохранено», поэтому потерять изменение
допустимо. Если она упадёт после шага 2, изменение есть в журнале, и восстановление применит его снова. В обоих случаях
состояние после перезапуска — такое, какое вызывающие могли бы увидеть. В этом весь приём.

```text
put(apple, red)
   │
   ├─► 1. append entry to store.wal ──► 2. sync_data ──► 3. map.insert ──► "ok"
   │
 crash here? ─ before 2: entry may be torn or missing  → caller never got "ok", fine
            ─ after 2:  entry is in the log            → replay restores it
```

Так работают PostgreSQL, SQLite (в режиме WAL), InnoDB в MySQL и LSM-деревья (LSM, Log-Structured Merge, слияние со структурой журнала) из Урока 17.7. В PostgreSQL транзакция из
Урока 16.7 «зафиксирована» ровно в тот момент, когда её запись о фиксации в WAL стала надёжной.

## 2. Формат записи журнала

Мы снова используем формат записи из Урока 17.3 (контрольная сумма, две длины, ключ, значение). Чтобы отличить put от
delete, первый байт значения записи — это **тип записи**:

```text
put apple = red    → record { key: "apple", value: [1, 'r', 'e', 'd'] }   (1 = put)
delete kiwi        → record { key: "kiwi",  value: [0] }                  (0 = delete)
```

Так слой записей по-прежнему даёт нам длины и контрольную сумму для каждой записи, а слой WAL добавляет лишь один байт.

## 3. Хранилище

Создайте `src/wal.rs`. Сначала тип и восстановление:

```rust
use crate::record::{self, DecodeError};
use std::collections::HashMap;
use std::fs::{File, OpenOptions};
use std::io::{self, Write};
use std::path::Path;

const PUT: u8 = 1;
const DELETE: u8 = 0;

/// A key-value store that keeps everything in a HashMap
/// and makes every change durable in a write-ahead log first.
pub struct WalStore {
    map: HashMap<Vec<u8>, Vec<u8>>,
    log: File,
    /// How many log entries were replayed when the store was opened.
    pub replayed: usize,
    /// How many bytes of a torn tail were cut off when the store was opened.
    pub dropped_bytes: u64,
}

impl WalStore {
    /// Opens the log at `path` (creating it if needed) and replays it.
    pub fn open(path: &Path) -> io::Result<WalStore> {
        let log = OpenOptions::new()
            .create(true)
            .read(true)
            .append(true)
            .open(path)?;
        let bytes = std::fs::read(path)?;

        let mut map = HashMap::new();
        let mut pos = 0;
        let mut replayed = 0;
        while pos < bytes.len() {
            match record::decode(&bytes[pos..]) {
                Ok((rec, used)) => {
                    apply(&mut map, rec.key, &rec.value);
                    pos += used;
                    replayed += 1;
                }
                // A torn or damaged entry: everything from here on is not trusted.
                Err(
                    DecodeError::Truncated | DecodeError::BadChecksum | DecodeError::TooLarge(_),
                ) => {
                    break;
                }
            }
        }

        let dropped_bytes = (bytes.len() - pos) as u64;
        if dropped_bytes > 0 {
            log.set_len(pos as u64)?; // cut the torn tail off
            log.sync_all()?;
        }
        Ok(WalStore {
            map,
            log,
            replayed,
            dropped_bytes,
        })
    }
```

**Что происходит:**

1. `OpenOptions` открывает журнал для чтения и **дописывания** и создаёт его, если он не существует. В режиме дописывания
   каждый `write` идёт в текущий конец файла, что бы ни происходило до этого. *Только что созданный* журнал — это новое
   имя в каталоге, поэтому полностью защищённое от сбоев хранилище один раз синхронизирует и каталог после создания
   файла (пункт 4 контрольного списка из Урока 17.2). Здесь мы это опускаем, чтобы код был коротким; в вашем проекте в
   Уроке 17.14 это обязательно.
2. `std::fs::read` загружает весь журнал в память. Для маленького хранилища это нормально; большое читало бы его частями.
3. Цикл — это **воспроизведение**: декодировать запись, применить её к словарю, сдвинуть `pos`. Поздние записи
   перезаписывают ранние, так что после цикла словарь содержит последнее состояние.
4. Первая запись, которая не декодируется, останавливает воспроизведение. Все три ошибки означают «эта запись и всё после
   неё — не целая правильная запись». После сбоя это разорванный хвост из Урока 17.2.
5. `set_len(pos)` **усекает** файл: выбрасывает разорванные байты. `sync_all` делает новую длину надёжной (`set_len`
   меняет метаданные, поэтому `sync_all`, а не `sync_data`). Раздел 4 показывает, почему этот шаг необходим.

Затем чтение и запись, всё ещё внутри `impl WalStore`, и после него вспомогательная `apply`:

```rust
    pub fn get(&self, key: &[u8]) -> Option<&[u8]> {
        self.map.get(key).map(|v| v.as_slice())
    }

    pub fn put(&mut self, key: &[u8], value: &[u8]) -> io::Result<()> {
        let mut entry = vec![PUT];
        entry.extend_from_slice(value);
        self.append(key, &entry)?; // 1. log first ...
        self.map.insert(key.to_vec(), value.to_vec()); // 2. ... then memory
        Ok(())
    }

    pub fn delete(&mut self, key: &[u8]) -> io::Result<()> {
        self.append(key, &[DELETE])?;
        self.map.remove(key);
        Ok(())
    }

    fn append(&mut self, key: &[u8], entry: &[u8]) -> io::Result<()> {
        let mut buf = Vec::new();
        record::encode(key, entry, &mut buf);
        self.log.write_all(&buf)?;
        self.log.sync_data() // durable before we return Ok
    }
}

/// Applies one log entry to the map. The first byte of the entry says what it is.
fn apply(map: &mut HashMap<Vec<u8>, Vec<u8>>, key: Vec<u8>, entry: &[u8]) {
    match entry.split_first() {
        Some((&PUT, value)) => {
            map.insert(key, value.to_vec());
        }
        Some((&DELETE, _)) => {
            map.remove(&key);
        }
        _ => {} // unknown entry type: ignore it
    }
}
```

**Что происходит:**

1. `get` никогда не трогает диск: все данные в словаре. `map(|v| v.as_slice())` превращает `Option<&Vec<u8>>` в
   `Option<&[u8]>`.
2. `put` и `delete` точно следуют правилу: сначала `append` (запись + `sync_data`); если он не удался, `?` возвращает
   ошибку, и словарь не меняется. Только после надёжного дописывания меняется словарь.
3. `append` кодирует запись в один буфер и пишет его одним `write_all`, чтобы запись по возможности ушла одним куском.
4. `split_first` разделяет срез на первый элемент и остаток. Образец `Some((&PUT, value))` читается как «первый байт —
   `PUT`, остальное — значение». `PUT` и `DELETE` — константы, поэтому их можно использовать как образцы.

Добавьте `pub mod wal;` в `src/lib.rs`.

> ⚡ **Производительность:** каждый `put` здесь платит за полный `sync_data`, около 5 мс на Mac (Урок 17.2), поэтому это
> хранилище успевает лишь около 200 записей в секунду. Настоящие движки используют групповую фиксацию: одна синхронизация
> на все записи, пришедшие за это время. `SyncPolicy` из Урока 17.2 — первый шаг в этом направлении.

## 4. Восстановление и почему хвост нужно отрезать

Зачем `set_len`? Представьте, что мы его пропустили. После сбоя файл заканчивается 11 разорванными байтами. Воспроизведение
правильно останавливается на них. Затем следующий `put` дописывает новую, безупречную запись **после** разорванных байтов:

```text
[ entry 1 ][ entry 2 ][ torn 11 bytes ][ new entry 3 ]
                       ▲
                       next replay stops here, and entry 3 is never read
```

При следующем перезапуске воспроизведение снова останавливается на разорванных байтах, и запись 3, про которую
вызывающему сказали «сохранено», потеряна. Усечение файла по концу последней хорошей записи снова делает журнал чистым
префиксом, и новые записи его продолжают.

Этот тест стоит написать первым. Поместите тесты в конец `src/wal.rs`:

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn data_survives_a_restart() -> io::Result<()> {
        let dir = tempfile::tempdir()?;
        let path = dir.path().join("store.wal");

        let mut store = WalStore::open(&path)?;
        store.put(b"apple", b"red")?;
        store.put(b"kiwi", b"green")?;
        store.put(b"apple", b"yellow")?;
        store.delete(b"kiwi")?;
        drop(store); // "restart"

        let store = WalStore::open(&path)?;
        assert_eq!(store.replayed, 4);
        assert_eq!(store.get(b"apple"), Some(&b"yellow"[..]));
        assert_eq!(store.get(b"kiwi"), None);
        Ok(())
    }

    #[test]
    fn a_torn_tail_is_cut_off_and_new_writes_survive() -> io::Result<()> {
        let dir = tempfile::tempdir()?;
        let path = dir.path().join("store.wal");
        let mut store = WalStore::open(&path)?;
        store.put(b"apple", b"red")?;
        drop(store);

        // Simulate a crash in the middle of the next append: write half an entry.
        let mut half = Vec::new();
        record::encode(b"kiwi", &[PUT, b'g', b'r', b'e', b'e', b'n'], &mut half);
        let mut file = OpenOptions::new().append(true).open(&path)?;
        file.write_all(&half[..half.len() / 2])?;
        drop(file);

        let mut store = WalStore::open(&path)?;
        assert_eq!(store.replayed, 1);
        assert_eq!(store.dropped_bytes, (half.len() / 2) as u64);
        assert_eq!(store.get(b"kiwi"), None);

        store.put(b"plum", b"purple")?; // must not land behind the garbage
        drop(store);
        let store = WalStore::open(&path)?;
        assert_eq!(store.get(b"plum"), Some(&b"purple"[..]));
        assert_eq!(store.dropped_bytes, 0);
        Ok(())
    }
}
```

**Что происходит:**

1. `tempfile::tempdir()` создаёт новый пустой каталог; когда `dir` удаляется в конце теста, удаляются каталог и всё, что в
   нём. У каждого теста свой каталог, поэтому тесты могут идти параллельно.
2. `drop(store)` закрывает журнал; повторное открытие — это «перезапуск», который обязан воспроизвести журнал.
3. Второй тест делает разорванный хвост вручную: кодирует настоящую запись и дописывает только её первую половину — ровно
   то, что может оставить сбой посреди `write_all`.
4. После восстановления записывается `plum`, и хранилище открывается ещё раз. Только правильно усечённый журнал сохраняет
   `plum`.

`cargo test --lib wal`:

```text
running 2 tests
test wal::tests::data_survives_a_restart ... ok
test wal::tests::a_torn_tail_is_cut_off_and_new_writes_survive ... ok

test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 13 filtered out; finished in 0.11s
```

(13 отфильтрованных тестов — это тесты предыдущих уроков и их упражнений.) Теперь удалите строку с `set_len` и запустите
тесты снова. Второй тест падает ровно там, где должна быть потерянная запись:

```text
thread 'wal::tests::a_torn_tail_is_cut_off_and_new_writes_survive' (6188123) panicked at src/wal.rs:147:9:
assertion `left == right` failed
  left: None
 right: Some([112, 117, 114, 112, 108, 101])
```

`[112, 117, 114, 112, 108, 101]` — это `purple` в виде байтов. Верните строку на место.

> 🧪 **Тестирование:** тест восстановления всегда состоит из трёх шагов: построить состояние, повредить файл так, как это
> сделал бы сбой, открыть заново и проверить и старые данные, *и* то, что новые записи по-прежнему работают. Последнюю
> часть обычно забывают.

Наше восстановление останавливается на **первой** плохой записи, где бы она ни была. Плохая запись в середине журнала
(сбой бита на устройстве) тоже выбросила бы все последующие хорошие записи. Настоящие движки различают два случая: плохой
хвост — это нормально после сбоя, а плохая запись, за которой идут хорошие, — это настоящее повреждение, о котором они
сообщают, а не молча выбрасывают данные.

## 5. Попробуйте с настоящим сбоем

Маленькая программа командной строки делает хранилище видимым. Поместите это в `src/bin/wal_demo.rs`:

```rust
use p17_storage::{record, wal::WalStore};
use std::fs::OpenOptions;
use std::io::Write;
use std::path::Path;

const LOG: &str = "demo.wal";

/// Writes only the first half of a put entry and then dies, like a power cut during a write.
fn crash_put(key: &str, value: &str) -> std::io::Result<()> {
    let mut entry = vec![1u8]; // 1 = put
    entry.extend_from_slice(value.as_bytes());
    let mut buf = Vec::new();
    record::encode(key.as_bytes(), &entry, &mut buf);
    let mut file = OpenOptions::new().append(true).open(LOG)?;
    file.write_all(&buf[..buf.len() / 2])?;
    std::process::abort();
}

fn main() -> std::io::Result<()> {
    let args: Vec<String> = std::env::args().skip(1).collect();
    let args: Vec<&str> = args.iter().map(String::as_str).collect();

    let mut store = WalStore::open(Path::new(LOG))?;
    if store.dropped_bytes > 0 {
        println!(
            "recovery: cut off a torn tail of {} bytes",
            store.dropped_bytes
        );
    }

    match args.as_slice() {
        ["put", key, value] => {
            store.put(key.as_bytes(), value.as_bytes())?;
            println!("ok");
        }
        ["get", key] => match store.get(key.as_bytes()) {
            Some(value) => println!("{}", String::from_utf8_lossy(value)),
            None => println!("(not found)"),
        },
        ["delete", key] => {
            store.delete(key.as_bytes())?;
            println!("ok");
        }
        ["crash-put", key, value] => crash_put(key, value)?,
        _ => eprintln!("usage: wal_demo put K V | get K | delete K | crash-put K V"),
    }
    Ok(())
}
```

**Что происходит:**

1. Каждый запуск — новый процесс: он открывает (и воспроизводит) `demo.wal`, выполняет одну команду и завершается. Поэтому
   каждая команда после первой проверяет восстановление.
2. `args.as_slice()` позволяет `match` смотреть сразу на весь список аргументов: `["put", key, value]` подходит ровно к
   трём аргументам с `put` первым и связывает остальные два.
3. `crash-put` строит настоящую запись, дописывает только её половину и аварийно завершается — отключение питания в
   миниатюре. Её тип возврата — `io::Result<()>`, но она никогда не возвращается: `abort` завершает процесс.

Выполните команды одну за другой:

```bash
cargo run -q --bin wal_demo -- put apple red
cargo run -q --bin wal_demo -- put kiwi green
cargo run -q --bin wal_demo -- get apple
cargo run -q --bin wal_demo -- delete kiwi
cargo run -q --bin wal_demo -- get kiwi
cargo run -q --bin wal_demo -- crash-put plum purple
cargo run -q --bin wal_demo -- get plum
cargo run -q --bin wal_demo -- get apple
```

```text
ok
ok
red
ok
(not found)
recovery: cut off a torn tail of 11 bytes
(not found)
red
```

(Командная оболочка также сообщит об аварийном завершении `crash-put`.) После `delete kiwi` журнал занимал 60 байт:
21 + 22 + 17 байт на три записи. `crash-put` добавил 11 из своих 23 байт, и следующий запуск их отрезал. `plum` не
найден, и это правильно: `crash-put` так и не сказал «ok». Когда закончите, удалите `demo.wal`.

## 6. Журнал растёт бесконечно

Каждый `put` добавляет запись, даже когда перезаписывает ключ. После миллиона обновлений одного счётчика в журнале миллион
записей, из которых важна одна. Воспроизведение при запуске становится всё медленнее. Ответов два, и настоящие системы
используют оба:

- **Компакция**: перезаписать журнал так, чтобы в нём было только текущее состояние, по одной записи на живой ключ.
  Упражнение 3 делает это с шаблоном атомарной замены из Урока 13.12.
- **Контрольные точки**: данные живут и в другом месте (страницы данных в PostgreSQL, отсортированные файлы в LSM-дереве).
  Время от времени движок убеждается, что они надёжны до позиции X в журнале; после этого журнал до X не нужен, и его можно
  удалить или использовать повторно. В PostgreSQL это называется *checkpoint*.

В Уроках 17.5–17.8 данные переезжают из `HashMap` в файлы на диске, и WAL становится тем, чем он является в настоящей
базе данных: недолговечной страховочной сеткой между «изменение пришло» и «изменение в файлах данных».

## Частые ошибки

1. **Менять память раньше журнала.** Если словарь меняется первым, а затем дописывание не удаётся, читатели видели данные,
   которые не надёжны и исчезнут после перезапуска.
2. **Не усекать разорванный хвост.** Новые записи оказываются за мусором и теряются при следующем перезапуске (раздел 4).
3. **Считать любую ошибку декодирования фатальной.** Разорванная последняя запись после сбоя — это *нормально*; хранилище,
   которое тогда отказывается запускаться, не устойчиво к сбоям. Считайте хвост «не записанным».
4. **Подтверждать до того, как вернулся `sync_data`.** Тогда WAL ничего не защищает; см. Урок 17.2.
5. **Тестировать восстановление без записи после него.** Многие ошибки проявляются только тогда, когда к восстановленному
   журналу дописываются новые данные.

## Попробуйте сами

1. Выполните команды демо выше, затем `xxd demo.wal`, и найдите байт типа (`01` или `00`) у каждой записи.
2. Выполните `crash-put` дважды подряд, без ничего между ними. Что сообщит следующий запуск и почему разорванный хвост
   только один?
3. Измерьте время 100 вызовов `put` в цикле в небольшом тесте или программе. Сравните с ценой `sync_data`, которую вы
   измерили в Уроке 17.2.

## Упражнения

### Упражнение 1 — Какой порядок правильный? (легко)

Коллега пишет `put` так: `self.map.insert(...)`, затем `println!("ok")` клиенту, затем `self.append(...)`. Опишите один
сбой, при котором клиент увидит неправильное поведение, и назовите правильный порядок.

<details>
<summary>Подсказка</summary>

Во что верит клиент после того, как увидел «ok», и что содержит журнал, если программа падает именно тогда?

</details>

<details>
<summary>Ответ</summary>

Программа печатает «ok» и падает до `append`. Клиент считает, что значение сохранено, но в журнале нет записи, поэтому
после перезапуска значение пропало. Даже без сбоя другой читатель мог увидеть новое значение в словаре, а потом потерять его
после перезапуска. Правильный порядок: `append` (запись + `sync_data`), затем изменить словарь, затем ответить «ok».

</details>

### Упражнение 2 — Повреждённая последняя запись (средне)

Напишите тест `a_damaged_last_entry_is_dropped`: запишите `apple = red` и `kiwi = green`, закройте хранилище, переверните
все биты **последнего байта** файла и откройте заново. Проверьте, что воспроизведена одна запись, что выброшено 22 байта,
что `apple` на месте, а `kiwi` нет.

<details>
<summary>Подсказка</summary>

Прочитайте файл через `std::fs::read`, измените `bytes[bytes.len() - 1] ^= 0xff` и запишите обратно через
`std::fs::write`. Запись kiwi — это 12 + 4 + 6 = 22 байта (заголовок, ключ, байт типа + значение).

</details>

<details>
<summary>Ответ</summary>

```rust
    #[test]
    fn a_damaged_last_entry_is_dropped() -> io::Result<()> {
        let dir = tempfile::tempdir()?;
        let path = dir.path().join("store.wal");
        let mut store = WalStore::open(&path)?;
        store.put(b"apple", b"red")?;
        store.put(b"kiwi", b"green")?;
        drop(store);

        let mut bytes = std::fs::read(&path)?;
        let last = bytes.len() - 1;
        bytes[last] ^= 0xff; // damage the last byte of the kiwi entry
        std::fs::write(&path, &bytes)?;

        let store = WalStore::open(&path)?;
        assert_eq!(store.replayed, 1);
        assert_eq!(store.dropped_bytes, 22); // the whole kiwi entry
        assert_eq!(store.get(b"apple"), Some(&b"red"[..]));
        assert_eq!(store.get(b"kiwi"), None);
        Ok(())
    }
```

Почему это работает: все байты записи kiwi на месте, поэтому `decode` не сообщает `Truncated`, но CRC больше не совпадает,
и он сообщает `BadChecksum`. Воспроизведение останавливается перед записью, а восстановление отрезает все 22 байта.
Разорванная и повреждённая записи обрабатываются одинаково.

</details>

### Упражнение 3 — Компакция (сложно)

Добавьте в `WalStore` метод `pub fn compact(&mut self, path: &Path) -> io::Result<()>` (передавайте тот же путь, что и в
`open`). Он записывает по одной записи put на каждый живой ключ во временный файл рядом с журналом, делает его надёжным,
переименовывает поверх журнала, синхронизирует каталог и продолжает дописывать в новый файл. Тест: 50 вызовов put для
`counter`, put и delete для `gone`; после `compact` и ещё одного put файл меньше, чем был, повторное открытие
воспроизводит ровно 2 записи, и все значения правильные.

<details>
<summary>Подсказка</summary>

Соберите все записи в один `Vec<u8>` через `record::encode`, как это делает `append`. Для временного имени используйте
`path.with_extension("compacting")`. После `fs::rename` откройте каталог (`path.parent()`) и вызовите для него
`sync_all`, затем замените `self.log` новым файлом, открытым в режиме дописывания.

</details>

<details>
<summary>Ответ</summary>

Внутри `impl WalStore`:

```rust
    /// Rewrites the log at `path` (the same path as in `open`) so that it holds
    /// one put entry per live key, and nothing else.
    pub fn compact(&mut self, path: &Path) -> io::Result<()> {
        let tmp = path.with_extension("compacting");
        let mut buf = Vec::new();
        for (key, value) in &self.map {
            let mut entry = vec![PUT];
            entry.extend_from_slice(value);
            record::encode(key, &entry, &mut buf);
        }
        let mut file = File::create(&tmp)?;
        file.write_all(&buf)?;
        file.sync_all()?; // 1. the new log is durable
        drop(file);
        std::fs::rename(&tmp, path)?; // 2. atomically replace the old log
        let dir = path.parent().unwrap_or(Path::new("."));
        File::open(dir)?.sync_all()?; // 3. make the rename durable
        self.log = OpenOptions::new().append(true).open(path)?; // 4. append to the new file
        Ok(())
    }
```

В тестах:

```rust
    #[test]
    fn compact_shrinks_the_log_and_keeps_the_data() -> io::Result<()> {
        let dir = tempfile::tempdir()?;
        let path = dir.path().join("store.wal");
        let mut store = WalStore::open(&path)?;
        for i in 0..50 {
            store.put(b"counter", i.to_string().as_bytes())?;
        }
        store.put(b"gone", b"soon")?;
        store.delete(b"gone")?;
        let before = std::fs::metadata(&path)?.len();

        store.compact(&path)?;
        store.put(b"after", b"compaction")?;
        let after = std::fs::metadata(&path)?.len();
        drop(store);

        let store = WalStore::open(&path)?;
        assert!(after < before, "before {before}, after {after}");
        assert_eq!(store.replayed, 2);
        assert_eq!(store.get(b"counter"), Some(&b"49"[..]));
        assert_eq!(store.get(b"after"), Some(&b"compaction"[..]));
        assert_eq!(store.get(b"gone"), None);
        Ok(())
    }
```

`cargo test --lib wal` (вместе с упражнением 2):

```text
test result: ok. 4 passed; 0 failed; 0 ignored; 0 measured; 13 filtered out; finished in 0.27s
```

Почему это работает: сбой в любой момент оставляет либо целый старый журнал (до переименования), либо целый новый (после
него); оба воспроизводятся в одно и то же состояние. В новом журнале по одной записи на живой ключ, поэтому `gone`
(удалённый) и 49 старых значений счётчика исчезают. Шаг 4 важен: старый `File` всё ещё указывает на старый, теперь
безымянный файл, и дописывания в него были бы потеряны.

</details>

## Вопросы для самопроверки

1. Сформулируйте правило упреждающей записи одним предложением.

<details>
<summary>Ответ</summary>

Сделайте изменение надёжным в журнале, прежде чем применять его где-либо ещё или подтверждать.

</details>

2. Программа падает после того, как `sync_data` вернулся, но до `map.insert`. Потеряно ли изменение?

<details>
<summary>Ответ</summary>

Нет. Запись надёжно лежит в журнале, поэтому воспроизведение при следующем запуске применит её к словарю.

</details>

3. Почему воспроизведение останавливается на первой записи, которая не декодируется, а не пропускает её?

<details>
<summary>Ответ</summary>

После повреждённой записи читатель больше не знает, где начинается следующая (длины могут быть неверными), а разорванная
запись по определению — последнее, что записывалось. Всё начиная с неё считается «не записанным».

</details>

4. Что пойдёт не так, если восстановление не усечёт разорванный хвост?

<details>
<summary>Ответ</summary>

Новые записи дописываются после разорванных байтов. При следующем перезапуске воспроизведение снова остановится на
разорванных байтах и никогда не прочитает новые записи, так что подтверждённые данные потеряются.

</details>

5. Почему `open` вызывает после `set_len` именно `sync_all`, а не `sync_data`?

<details>
<summary>Ответ</summary>

`set_len` меняет размер файла, а это метаданные. `sync_all` делает надёжными и данные, *и* метаданные, поэтому меньшая
длина гарантированно переживёт ещё один сбой.

</details>

6. Назовите два способа, которыми настоящие базы данных не дают WAL расти бесконечно.

<details>
<summary>Ответ</summary>

Компакция (перезаписать журнал только с живыми данными) и контрольные точки (сделать файлы данных надёжными до некоторой
позиции журнала, а затем удалить или переиспользовать журнал до неё).

</details>

## Итоги

- Правило упреждающей записи: дописать изменение в журнал, сделать его надёжным, затем применить и подтвердить.
- Крошечное, но устойчивое к сбоям хранилище «ключ-значение» — это `HashMap` плюс WAL; `open` воспроизводит журнал, чтобы
  восстановить словарь.
- Журнал использует формат записей из Урока 17.3 с одним дополнительным байтом типа записи (put или delete).
- Восстановление останавливается на первой записи, которая не декодируется, и усекает файл там; без усечения теряются новые
  данные.
- Тесты восстановления строят состояние, повреждают файл как сбой, открывают заново и проверяют и старые данные, и новые
  записи.
- Журнал растёт с каждым изменением; компакция и контрольные точки держат его маленьким.

## Источники и что почитать

- [PostgreSQL documentation — Write-Ahead Logging (WAL)](https://www.postgresql.org/docs/current/wal-intro.html) — то же
  правило в настоящей базе данных и то, как оно делает фиксации надёжными.
- [SQLite — Write-Ahead Logging](https://www.sqlite.org/wal.html) — режим WAL в SQLite, контрольные точки и их
  компромиссы.
- [LevelDB log format](https://github.com/google/leveldb/blob/main/doc/log_format.md) — WAL настоящего хранилища
  «ключ-значение».
- [std::fs::OpenOptions](https://doc.rust-lang.org/std/fs/struct.OpenOptions.html) — `append`, `create` и другие
  параметры и что гарантирует режим дописывания.
- [std::fs::File](https://doc.rust-lang.org/std/fs/struct.File.html) — `set_len`, `sync_all` и `sync_data`.
- [Документация tempfile](https://docs.rs/tempfile/latest/tempfile/) — временные файлы и каталоги для тестов.

---

[🏠 Начало](../START-HERE.md) · [← 17.3 Двоичные форматы файлов и сериализация](03-binary-file-formats.md) · [📋 Этап 17](README.md) · [17.5 Индексы: хеш-индекс →](05-hash-indexes.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
