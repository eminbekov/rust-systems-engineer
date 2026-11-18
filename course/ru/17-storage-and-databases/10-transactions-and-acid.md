<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 17.9 Страничный кэш и пул буферов](09-page-cache-and-buffer-pool.md) · [📋 Этап 17](README.md) · [17.11 Уровни изоляции →](11-isolation-levels.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/17-storage-and-databases/10-transactions-and-acid.md) <!-- nav -->

# Урок 17.10 — Транзакции и ACID

> **Этап 17 — Как устроены хранилища и базы данных** · Урок 10 из 15 · ⏱ около 135 минут

## Что вы узнаете

- Что обещает каждая буква ACID и какая часть базы данных выполняет каждое обещание.
- Как работает атомарная фиксация: изменения ждут в наборе записей и попадают в журнал **одной** записью с
  контрольной суммой.
- Как восстановление выбрасывает транзакцию, прерванную сбоем, и как протестировать сбой на каждом байте.
- Что такое информация для повтора (redo) и для отката (undo) и зачем пулу буферов нужна undo-информация.
- Почему одного ACID недостаточно, чтобы два клиента не теряли обновления друг друга.

## Перед началом

- Вы пользовались транзакциями в PostgreSQL (`BEGIN`, `COMMIT`, `ROLLBACK`) и видели таблицу ACID в
  [Уроке 16.7 — Миграции и транзакции](../16-backend-services/07-migrations-and-transactions.md).
- Вы знаете `sync_data`, разорванные записи и групповую фиксацию из [Урока 17.2 — Надёжность записи: fsync и устойчивость к сбоям](02-durability-and-fsync.md),
  записи с длиной и контрольной суммой из [Урока 17.3 — Двоичные форматы файлов и сериализация](03-binary-file-formats.md)
  и WAL (write-ahead log, журнал упреждающей записи) из [Урока 17.4 — Журнал упреждающей записи (WAL)](04-write-ahead-log.md).
- Вы знаете пул буферов и грязные страницы из [Урока 17.9 — Страничный кэш и пул буферов](09-page-cache-and-buffer-pool.md).

## Новые слова

| Слово | Значение |
|---|---|
| **транзакция** (transaction) | Группа чтений и записей, которую база данных считает одним целым: происходят все её изменения или ни одного. |
| **ACID** (Atomicity, Consistency, Isolation, Durability — атомарность, согласованность, изоляция, долговечность) | Четыре обещания транзакции: всё или ничего, правила остаются верными, транзакции не мешают друг другу, зафиксированные данные переживают сбои. |
| **точка фиксации** (commit point) | Точный момент, когда транзакция становится постоянной. До него сбой отменяет транзакцию, после него — уже нет. |
| **набор записей** (write set) | Изменения, которые транзакция сделала, но ещё не зафиксировала; они отложены в памяти. |
| **запись о фиксации** (commit record) | Запись журнала, само надёжное присутствие которой означает «эта транзакция зафиксирована». |
| **откат** (rollback, abort) | Завершение транзакции с выбрасыванием её изменений. |
| **инвариант** (invariant) | Правило о ваших данных, которое всегда должно быть верным, например «общая сумма денег не меняется». |
| **информация для повтора** (redo information) | То, что нужно, чтобы *повторить* зафиксированное изменение после сбоя (новое значение). |
| **информация для отката** (undo information) | То, что нужно, чтобы *отменить* незафиксированное изменение после сбоя (старое значение). |
| **CRC** (Cyclic Redundancy Check, циклический избыточный код) | Короткая контрольная сумма, вычисленная по байтам. Если меняется любой байт, CRC почти наверняка меняется. Здесь мы используем CRC-32 из крейта `crc32fast`. |
| **ARIES** (Algorithms for Recovery and Isolation Exploiting Semantics, алгоритмы восстановления и изоляции, использующие семантику) | Классический метод восстановления дисковых баз данных; его журнал содержит и информацию для повтора, и информацию для отката. |
| **потерянное обновление** (lost update) | Две транзакции читают одно и то же значение, обе его меняют, и вторая запись молча стирает первую. |

## 1. ACID изнутри

Урок 16.7 показал, как ACID выглядит снаружи. Вот какая часть базы данных выполняет каждое обещание:

| Буква | Обещание | Кто его выполняет | Где в курсе |
|---|---|---|---|
| **A** — атомарность | все изменения транзакции или ни одного | журнал: транзакция считается, только если её запись о фиксации цела | этот урок |
| **C** — согласованность | объявленные вами правила выполняются и после транзакции | ограничения в базе данных **и** ваш собственный код | этот урок, раздел 5 |
| **I** — изоляция | одновременные транзакции не видят недоделанную работу друг друга | блокировки или несколько версий | Уроки 17.11 и 17.12 |
| **D** — долговечность | после «commit OK» изменение переживает сбой | `sync_data` журнала перед ответом | Уроки 17.2 и 17.4 |

Атомарность и долговечность встречаются в одном моменте — **точке фиксации**: это момент, когда запись о фиксации
надёжно лежит в журнале. Весь этот урок о том, как поставить этот момент в правильное место.

## 2. Атомарная фиксация одной записью

Идея умещается в три правила:

1. Пока транзакция работает, её изменения складываются в **набор записей** в памяти. Ни журнал, ни данные не трогаются.
2. Чтобы зафиксировать, весь набор записей пишется в журнал **одной записью** с длиной и CRC, затем `sync_data`.
3. При запуске проигрываются только записи с правильными длиной и CRC. Запись, разорванная сбоем, не проходит проверку и
   отбрасывается вместе со всем, что после неё.

```text
 bank.log:
 +-----+-----+------------------------------+-----+-----+--------------------------
 | len | crc | alice=100, bob=50            | len | crc | alice=70, bob=80 ...  <- crash here
 +-----+-----+------------------------------+-----+-----+--------------------------
 \_______ complete: replayed ______________/ \______ torn: CRC check fails, cut off __
```

Откат бесплатен: набор записей просто никогда не записывается. Сбой до точки фиксации оставляет самое большее
разорванную запись, которую восстановление удаляет. Поэтому не бывает состояния, в котором видна только половина
транзакции.

> 💡 **Совет:** многие движки пишут транзакцию несколькими записями журнала (`BEGIN`, по записи на каждое изменение,
> `COMMIT`), а не одной большой. Правило то же: восстановление применяет транзакцию, только если её запись `COMMIT`
> цела. Одна запись на транзакцию — простейшая форма; LevelDB и RocksDB используют её для своих «пакетов записи»
> (write batches).

## 3. Транзакционное хранилище «ключ-значение» на Rust

Создайте проект:

```bash
cd ~/projects/rust
cargo new --lib code/p17-txkv
cd code/p17-txkv
cargo add crc32fast@1
cargo add tempfile@3 --dev
```

Ключи и значения — строки `String`, а данные живут в памяти в `BTreeMap` (упорядоченное отображение, Урок 07.11) и
при запуске восстанавливаются из журнала. Замените `src/lib.rs` на:

```rust
use std::collections::BTreeMap;
use std::fs::{File, OpenOptions};
use std::io::{self, Read, Write};
use std::path::Path;

/// The changes of one transaction: key -> Some(new value) or None (delete).
#[derive(Debug, Default)]
pub struct Tx {
    writes: BTreeMap<String, Option<String>>,
}

impl Tx {
    pub fn put(&mut self, key: &str, value: &str) {
        self.writes.insert(key.to_string(), Some(value.to_string()));
    }

    pub fn delete(&mut self, key: &str) {
        self.writes.insert(key.to_string(), None);
    }
}

pub struct Db {
    log: File,
    data: BTreeMap<String, String>,
}

impl Db {
    /// Opens the log, replays every complete transaction, cuts off a torn tail.
    pub fn open(path: &Path) -> io::Result<Db> {
        let mut log = OpenOptions::new()
            .read(true)
            .append(true)
            .create(true)
            .open(path)?;
        let mut bytes = Vec::new();
        log.read_to_end(&mut bytes)?;

        let mut data = BTreeMap::new();
        let mut pos = 0;
        while let Some((writes, len)) = decode_record(&bytes[pos..]) {
            apply(&mut data, writes);
            pos += len;
        }
        if pos < bytes.len() {
            log.set_len(pos as u64)?; // remove the unfinished transaction
            log.sync_all()?;
        }
        Ok(Db { log, data })
    }

    pub fn begin(&self) -> Tx {
        Tx::default()
    }

    /// Makes all changes of `tx` durable as ONE log record, then applies them.
    pub fn commit(&mut self, tx: Tx) -> io::Result<()> {
        if tx.writes.is_empty() {
            return Ok(());
        }
        self.log.write_all(&encode_record(&tx))?;
        self.log.sync_data()?; // the commit point
        apply(&mut self.data, tx.writes.into_iter().collect());
        Ok(())
    }

    pub fn get(&self, key: &str) -> Option<&str> {
        self.data.get(key).map(String::as_str)
    }
}

type Writes = Vec<(String, Option<String>)>;

fn apply(data: &mut BTreeMap<String, String>, writes: Writes) {
    for (key, value) in writes {
        match value {
            Some(v) => data.insert(key, v),
            None => data.remove(&key),
        };
    }
}

/// Record = [payload length: u32][CRC-32 of payload: u32][payload].
/// Payload = for each write: [key len: u32][key][tag: 1 = put, 0 = delete][value len: u32][value]
pub fn encode_record(tx: &Tx) -> Vec<u8> {
    let mut payload = Vec::new();
    for (key, value) in &tx.writes {
        put_bytes(&mut payload, key.as_bytes());
        match value {
            Some(v) => {
                payload.push(1);
                put_bytes(&mut payload, v.as_bytes());
            }
            None => payload.push(0),
        }
    }
    let mut record = Vec::with_capacity(8 + payload.len());
    record.extend_from_slice(&(payload.len() as u32).to_le_bytes());
    record.extend_from_slice(&crc32fast::hash(&payload).to_le_bytes());
    record.extend_from_slice(&payload);
    record
}

fn put_bytes(out: &mut Vec<u8>, bytes: &[u8]) {
    out.extend_from_slice(&(bytes.len() as u32).to_le_bytes());
    out.extend_from_slice(bytes);
}

/// Returns the writes and the record length, or None if the record is
/// incomplete or damaged.
fn decode_record(buf: &[u8]) -> Option<(Writes, usize)> {
    let len = u32::from_le_bytes(buf.get(0..4)?.try_into().ok()?) as usize;
    let crc = u32::from_le_bytes(buf.get(4..8)?.try_into().ok()?);
    let payload = buf.get(8..8 + len)?;
    if crc32fast::hash(payload) != crc {
        return None;
    }
    let mut writes = Vec::new();
    let mut p = payload;
    while !p.is_empty() {
        let key = take_bytes(&mut p)?;
        let (&tag, rest) = p.split_first()?;
        p = rest;
        let value = if tag == 1 {
            Some(take_bytes(&mut p)?)
        } else {
            None
        };
        writes.push((key, value));
    }
    Some((writes, 8 + len))
}

fn take_bytes(p: &mut &[u8]) -> Option<String> {
    let len = u32::from_le_bytes(p.get(0..4)?.try_into().ok()?) as usize;
    let bytes = p.get(4..4 + len)?;
    *p = &p[4 + len..];
    String::from_utf8(bytes.to_vec()).ok()
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn a_crash_at_any_byte_of_a_commit_loses_the_whole_transaction() {
        let dir = tempfile::tempdir().unwrap();
        let path = dir.path().join("tx.log");
        let mut db = Db::open(&path).unwrap();
        let mut t1 = db.begin();
        t1.put("alice", "100");
        t1.put("bob", "50");
        db.commit(t1).unwrap();
        drop(db);
        let good_len = std::fs::metadata(&path).unwrap().len();

        let mut t2 = Tx::default();
        t2.put("alice", "70");
        t2.put("bob", "80");
        let record = encode_record(&t2);

        for cut in 0..record.len() {
            // "crash": only the first `cut` bytes of t2's record reached the file
            let mut f = OpenOptions::new().append(true).open(&path).unwrap();
            f.write_all(&record[..cut]).unwrap();
            drop(f);

            let db = Db::open(&path).unwrap();
            assert_eq!(db.get("alice"), Some("100"), "cut at byte {cut}");
            assert_eq!(db.get("bob"), Some("50"), "cut at byte {cut}");
            assert_eq!(std::fs::metadata(&path).unwrap().len(), good_len);
        }
    }
}
```

**Что происходит:**

1. `Tx` — это только набор записей: упорядоченное отображение из ключа в `Some(value)` (запись) или `None`
   (удаление). Повторная запись того же ключа оставляет последнее значение. `Tx` не заимствует `Db`, поэтому можно
   держать несколько транзакций одновременно.
2. `commit` кодирует весь набор записей в одну запись и дописывает её. `sync_data` — **точка фиксации**: только после
   него мы меняем `data` и возвращаем `Ok`. Вызывающий код, получив `Ok`, может сказать пользователю «сохранено».
3. У отката нет метода: `Tx`, который удалён без `commit`, просто исчезает.
4. `encode_record` пишет `[длина][CRC-32][полезные данные]` в порядке little-endian, как записи из Урока 17.3. Каждая
   строка в полезных данных несёт свою длину, поэтому декодер знает, где она кончается.
5. `decode_record` возвращает `None` при **любой** проблеме: не хватает байтов на заголовок, на полезные данные,
   неверный CRC, испорченная строка. Хитрость — `buf.get(a..b)?`: `get` возвращает `None` вместо паники, когда
   диапазон выходит за буфер, а `?` передаёт этот `None` дальше.
6. `open` проигрывает записи до первой плохой, затем `set_len(pos)` обрезает файл в этом месте и синхронизирует его.
   Без обрезки следующая фиксация была бы дописана **после** мусора, и восстановление никогда бы до неё не дошло.
7. Тест записывает одну хорошую транзакцию, а затем имитирует сбой на **каждом** байте второй фиксации: дописывает
   только первые `cut` байтов, открывает базу заново и проверяет, что вторая транзакция полностью пропала, а файл
   исправлен.

```bash
cargo test
```

```text
running 1 test
test tests::a_crash_at_any_byte_of_a_commit_loses_the_whole_transaction ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.16s
```

> 🧪 **Тестирование:** «сбой на каждом байте» — дешёвый и сильный тест. Настоящий сбой — это просто «файл где-то
> кончается», поэтому дописывание каждого возможного префикса записи покрывает каждое место, где сбой может её
> оборвать. Проект в Уроке 17.14 использует ту же идею.

Теперь программа, которая этим пользуется. Создайте `src/bin/bank.rs`:

```rust
use p17_txkv::Db;
use std::path::Path;

fn main() -> std::io::Result<()> {
    let _ = std::fs::remove_file("bank.log"); // start fresh every run
    let mut db = Db::open(Path::new("bank.log"))?;

    let mut setup = db.begin();
    setup.put("alice", "100");
    setup.put("bob", "50");
    db.commit(setup)?;
    show("after setup", &db);

    // a transfer: two changes, committed together
    let mut transfer = db.begin();
    transfer.put("alice", "70");
    transfer.put("bob", "80");
    db.commit(transfer)?;
    show("after transfer", &db);

    // a transaction that is dropped instead of committed = rollback
    let mut mistake = db.begin();
    mistake.delete("alice");
    drop(mistake);
    show("after rollback", &db);

    drop(db);
    let db = Db::open(Path::new("bank.log"))?; // like a restart
    show("after reopen", &db);
    println!("log size: {} bytes", std::fs::metadata("bank.log")?.len());
    Ok(())
}

fn show(label: &str, db: &Db) {
    let alice = db.get("alice").unwrap_or("-");
    let bob = db.get("bob").unwrap_or("-");
    println!("{label:<15} alice={alice} bob={bob}");
}
```

```bash
cargo run --bin bank
```

```text
after setup     alice=100 bob=50
after transfer  alice=70 bob=80
after rollback  alice=70 bob=80
after reopen    alice=70 bob=80
log size: 77 bytes
```

**Что происходит:**

1. Перевод меняет два ключа; оба изменения попадают в журнал одной записью, поэтому никакой сбой не оставит
   «alice заплатила, bob не получил».
2. `mistake` удалён: удаления не было, и ничего не записано.
3. Повторное открытие проигрывает две записи (начальную настройку и перевод): 39 + 38 = 77 байтов.

Посмотрите на первую запись через `xxd` — утилиту, которая печатает байты в шестнадцатеричной системе (по основанию 16):

```bash
xxd bank.log | head -3
```

```text
00000000: 1f00 0000 ea6a 7fbe 0500 0000 616c 6963  .....j......alic
00000010: 6501 0300 0000 3130 3003 0000 0062 6f62  e.....100....bob
00000020: 0102 0000 0035 301e 0000 006e cc3c b905  .....50....n.<..
```

`1f00 0000` — длина полезных данных 31 (0x1f) в little-endian; `ea6a 7fbe` — CRC; затем `0500 0000` и `alice`,
метка `01` (запись), `0300 0000` и `100` и так далее. Со смещения 0x27 начинается вторая запись с длиной `1e` (30).

## 4. Повтор, откат и пул буферов

Наше хранилище простое, потому что незафиксированные изменения никогда не покидают `Tx`. В журнале лежит только
**информация для повтора**: новые значения зафиксированных транзакций, которые восстановление повторяет.

Дисковой базе данных с пулом буферов (Урок 17.9) повезло меньше. Длинная транзакция может изменить больше страниц, чем
помещается в пул, и пулу иногда приходится записывать в файл данных грязную страницу с **незафиксированными**
изменениями. Если затем случится сбой, в файле данных окажутся изменения транзакции, которая так и не зафиксировалась.
Восстановление должно их отменить, поэтому журнал должен содержать и **информацию для отката**: старые значения.

| Вопрос | Если «да», базе нужна |
|---|---|
| Можно ли записать в файл данных грязную страницу с незафиксированными изменениями? | информация для отката в журнале |
| Можно ли ответить «commit OK» до того, как все изменённые страницы окажутся в файлах данных? | информация для повтора в журнале |

Почти каждая настоящая база данных отвечает «да» на оба вопроса, потому что альтернативы медленные: держать всю
транзакцию в памяти или записывать каждую изменённую страницу при фиксации. Классический метод восстановления, который
сочетает повтор и откат, называется **ARIES** (Algorithms for Recovery and Isolation Exploiting Semantics, алгоритмы восстановления и изоляции, использующие семантику); его подробности пока не нужны. PostgreSQL выбирает для отката другой
путь: обновление никогда не перезаписывает старую строку, а пишет рядом новую версию, так что «откат» означает просто
«транзакция этой версии не зафиксировалась, игнорируй её». Эти версии показывает Урок 17.12.

## 5. Согласованность — отчасти ваша работа

«C» — странная буква. База данных может проверить только те правила, которые вы **объявили**: `NOT NULL`, `UNIQUE`,
`CHECK (balance >= 0)`, внешние ключи. Если любое из них нарушено, вся транзакция откатывается (вы видели это в
Уроке 16.7).

Необъявленные правила — ваша работа. «Общая сумма денег никогда не меняется» — **инвариант**, который не выражается
никаким ограничением. Перевод сохраняет его, только если ваш код меняет оба счёта в **одной** транзакции:

```text
transfer(alice -> bob, 30):     alice 100 -> 70,  bob 50 -> 80     total 150 -> 150  OK
crash between two transactions: alice 100 -> 70,  bob 50           total 150 -> 120  money vanished
```

Атомарность гарантирует, что транзакция — всё или ничего. А что входит в одну транзакцию, решает ваш код.

> 🔒 **Безопасность:** проверка вроде «хватает ли денег на балансе?» должна происходить **внутри** той транзакции,
> которая меняет баланс, и при правильной изоляции (следующий урок). Если проверка и изменение разделены, злоумышленник
> может отправить много запросов в один момент и потратить одни и те же деньги несколько раз.

## 6. Чего ACID в нашем хранилище не даёт: изоляции

Два одновременно работающих клиента всё равно могут нарушить инвариант. Создайте `src/bin/lost.rs`:

```rust
use p17_txkv::Db;
use std::path::Path;

fn main() -> std::io::Result<()> {
    let _ = std::fs::remove_file("lost.log");
    let mut db = Db::open(Path::new("lost.log"))?;
    let mut setup = db.begin();
    setup.put("alice", "100");
    db.commit(setup)?;

    // two "clients" both take 10 from alice at the same time
    let mut t1 = db.begin();
    let mut t2 = db.begin();
    let seen1: i64 = db.get("alice").unwrap().parse().unwrap();
    let seen2: i64 = db.get("alice").unwrap().parse().unwrap();
    t1.put("alice", &(seen1 - 10).to_string());
    t2.put("alice", &(seen2 - 10).to_string());
    db.commit(t1)?;
    db.commit(t2)?;

    println!("alice = {} (expected 80)", db.get("alice").unwrap());
    Ok(())
}
```

```bash
cargo run --bin lost
```

```text
alice = 90 (expected 80)
```

**Что происходит:**

1. Обе транзакции читают `alice = 100` до того, как какая-либо из них зафиксируется.
2. Каждая пишет `100 - 10 = 90`. Обе фиксации атомарны и долговечны, и всё равно 10 потеряны: вторая запись стёрла
   первую. Это **потерянное обновление**.
3. Атомарность и долговечность — о сбоях. В этой ошибке сбоя нет; она о двух транзакциях, которые **пересекаются по
   времени**. Не допускать такого — задача **изоляции**, темы Уроков 17.11 и 17.12.

## Частые ошибки

> ⚠️ **Частая ошибка:** писать транзакцию отдельной записью журнала на каждый ключ без отметки о фиксации. Мы это
> попробовали: если `commit` дописывает по записи на ключ, тест сбоя падает, потому что сбой между двумя записями
> сохраняет новое значение alice и теряет значение bob:
>
> ```text
> thread 'tests::a_crash_at_any_byte_of_a_commit_loses_the_whole_transaction' (6118007) panicked at src/lib.rs:177:13:
> assertion `left == right` failed: cut at byte 24
>   left: Some("70")
>  right: Some("100")
> ```

> ⚠️ **Частая ошибка:** менять данные в памяти (или отвечать «OK») **до** того, как вернулся `sync_data`. Если
> синхронизация не удалась или машина упала, пользователю сказали «сохранено» о данных, которых больше нет. Порядок
> такой: записать, синхронизировать, затем применить и ответить.

> ⚠️ **Частая ошибка:** не обрезать разорванный хвост при запуске. Следующая фиксация дописывается после мусора,
> восстановление останавливается на мусоре, и при следующем перезапуске теряется каждая более поздняя транзакция.

## Попробуйте сами

1. Выполните `cargo run --bin bank`, затем обрежьте журнал посередине второй записи и откройте его снова. В macOS:
   `head -c 60 bank.log > cut.log && mv cut.log bank.log`, затем добавьте маленькую программу `src/bin/show.rs`, которая
   открывает `bank.log` и печатает alice и bob (не удаляя файл перед этим). Вы должны увидеть `alice=100 bob=50`, а
   `ls -l bank.log` покажет 39 байтов: разорванная запись отрезана.
2. В `bank.rs` запишите один ключ дважды в одной транзакции (`put("alice", "1")`, затем `put("alice", "2")`). Какое
   значение победит и сколько записей окажется в журнальной записи?

## Упражнения

### Упражнение 1 — Посчитайте байты (легко)

Сколько байтов выдаёт `encode_record` для транзакции с единственной записью `put("k", "v")`? Вычислите по формату,
затем подтвердите тестом.

<details>
<summary>Подсказка</summary>

Заголовок: 4 + 4 байта. Полезные данные: длина ключа (4) + ключ + метка (1) + длина значения (4) + значение.

</details>

<details>
<summary>Ответ</summary>

8 + (4 + 1 + 1 + 4 + 1) = 19 байтов.

```rust
    #[test]
    fn record_size_of_one_small_put() {
        let mut tx = Tx::default();
        tx.put("k", "v");
        assert_eq!(encode_record(&tx).len(), 19);
    }
```

Почему это работает: в формате нет скрытого выравнивания; у каждого поля фиксированный размер, кроме строк, которые
несут свои длины.

</details>

### Упражнение 2 — Тест: откат не оставляет следов (средне)

Напишите тест `rollback_leaves_no_trace`: зафиксируйте `alice = 100`, запомните размер журнала, затем начните
транзакцию, которая ставит `alice = 0` и `eve = 1000`, и удалите её. Ещё зафиксируйте пустую транзакцию. Проверьте, что
журнал не вырос и что после повторного открытия alice по-прежнему 100, а eve не существует.

<details>
<summary>Подсказка</summary>

`std::fs::metadata(&path)?.len()` даёт размер файла. `db.commit(db.begin())` фиксирует пустую транзакцию.

</details>

<details>
<summary>Ответ</summary>

```rust
    #[test]
    fn rollback_leaves_no_trace() {
        let dir = tempfile::tempdir().unwrap();
        let path = dir.path().join("tx.log");
        let mut db = Db::open(&path).unwrap();
        let mut t1 = db.begin();
        t1.put("alice", "100");
        db.commit(t1).unwrap();
        let len_before = std::fs::metadata(&path).unwrap().len();

        let mut t2 = db.begin();
        t2.put("alice", "0");
        t2.put("eve", "1000");
        drop(t2); // rollback
        db.commit(db.begin()).unwrap(); // an empty commit writes nothing either

        assert_eq!(std::fs::metadata(&path).unwrap().len(), len_before);
        drop(db);
        let db = Db::open(&path).unwrap();
        assert_eq!(db.get("alice"), Some("100"));
        assert_eq!(db.get("eve"), None);
    }
```

Почему это работает: `Tx` никогда не трогает журнал; пишет только `commit`, а для пустого набора записей он сразу
возвращается.

</details>

### Упражнение 3 — Читать свои записи (сложно)

Внутри транзакции программа ожидает видеть свои изменения: после `tx.put("a", "10")` чтение `a` *в этой транзакции*
должно давать 10, а все остальные по-прежнему видят старое значение. Добавьте в `Db`
`pub fn get_in<'a>(&'a self, tx: &'a Tx, key: &str) -> Option<&'a str>` и протестируйте его с записью и удалением.

<details>
<summary>Подсказка</summary>

Сначала смотрите в `tx.writes`. Случаев три: `Some(Some(v))` (записано), `Some(None)` (удалено в этой транзакции),
`None` (не трогали: спросить зафиксированные данные). Метод находится в том же модуле, что и `Tx`, поэтому может читать
приватное поле `writes`.

</details>

<details>
<summary>Ответ</summary>

```rust
    /// Reads a key as transaction `tx` sees it: its own writes first.
    pub fn get_in<'a>(&'a self, tx: &'a Tx, key: &str) -> Option<&'a str> {
        match tx.writes.get(key) {
            Some(Some(value)) => Some(value),
            Some(None) => None, // deleted in this transaction
            None => self.get(key),
        }
    }
```

```rust
    #[test]
    fn a_transaction_reads_its_own_writes() {
        let dir = tempfile::tempdir().unwrap();
        let mut db = Db::open(&dir.path().join("tx.log")).unwrap();
        let mut setup = db.begin();
        setup.put("a", "1");
        setup.put("b", "2");
        db.commit(setup).unwrap();

        let mut tx = db.begin();
        tx.put("a", "10");
        tx.delete("b");
        assert_eq!(db.get_in(&tx, "a"), Some("10"));
        assert_eq!(db.get_in(&tx, "b"), None);
        assert_eq!(db.get("a"), Some("1")); // others still see the old value
        assert_eq!(db.get("b"), Some("2"));
    }
```

Почему это работает: набор записей «накрывает» зафиксированные данные, как прозрачная плёнка, на которой закрашены
некоторые клетки. Время жизни `'a` говорит, что возвращённый `&str` может прийти и из `Db`, и из `Tx`, поэтому оба
должны жить достаточно долго.

</details>

## Вопросы для самопроверки

1. Что именно является точкой фиксации хранилища из этого урока?

<details>
<summary>Ответ</summary>

Момент, когда возвращается `sync_data` после дописывания записи транзакции. До него сбой может оставить самое большее
разорванную запись, которую восстановление удалит; после него запись цела, надёжно сохранена и будет проиграна.

</details>

2. Сбой случился, когда на диск попали 20 из 38 байтов записи о фиксации. Что делает `Db::open`?

<details>
<summary>Ответ</summary>

Проигрывает предыдущие целые записи, не может декодировать разорванную (не хватает байтов или неверный CRC), обрезает
файл по концу последней хорошей записи через `set_len` и синхронизирует его. Оборванной транзакции как будто не было.

</details>

3. Почему базе данных с пулом буферов нужна информация для отката, а нашему хранилищу — нет?

<details>
<summary>Ответ</summary>

Пул буферов может записать в файл данных грязную страницу с незафиксированными изменениями. После сбоя их нужно
отменить, а для этого нужны старые значения. Наше хранилище никуда не пишет незафиксированные изменения.

</details>

4. За какую букву ACID отчасти отвечает приложение и почему?

<details>
<summary>Ответ</summary>

За согласованность. База данных может проверить только объявленные правила (ограничения). Необъявленные инварианты,
например «общая сумма денег постоянна», выполняются, только если приложение помещает все связанные изменения в одну
транзакцию.

</details>

5. Обе транзакции в `lost.rs` зафиксировались атомарно и долговечно. Почему результат всё равно неверен?

<details>
<summary>Ответ</summary>

Они пересеклись по времени: обе прочитали 100 до того, как какая-либо записала. Это проблема изоляции (потерянное
обновление), а не сбоя, поэтому атомарность и долговечность помочь не могут.

</details>

## Итоги

- ACID: атомарность и долговечность дают журнал и `sync_data`, изоляцию — блокировки или версии, согласованность —
  ограничения плюс ваш собственный код.
- Транзакция собирает изменения в набор записей; фиксация пишет их одной записью с контрольной суммой и
  синхронизирует. Эта синхронизация — точка фиксации.
- Восстановление проигрывает целые записи и обрезает разорванный хвост; прерванная транзакция исчезает полностью.
- Устойчивость к сбоям проверяется так: записать каждый префикс записи и открыть базу заново.
- Информация для повтора повторяет зафиксированные изменения; информация для отката отменяет незафиксированные — она
  нужна пулу буферов, который рано пишет грязные страницы.
- Транзакция группирует изменения; решать, *какие* изменения должны быть вместе, — работа приложения.
- Атомарные и долговечные транзакции всё равно могут терять обновления, если пересекаются: для этого и нужна изоляция.

## Источники и что почитать

- [PostgreSQL: Transactions (tutorial)](https://www.postgresql.org/docs/current/tutorial-transactions.html) — пример банковского перевода словами самого PostgreSQL.
- [PostgreSQL: Write-Ahead Logging (WAL)](https://www.postgresql.org/docs/current/wal-intro.html) — как PostgreSQL делает фиксации долговечными.
- [SQLite: Atomic Commit In SQLite](https://www.sqlite.org/atomiccommit.html) — подробный и понятный разбор атомарной фиксации с журналом отката.
- [LevelDB implementation notes](https://github.com/google/leveldb/blob/main/doc/impl.md) — журнал и пакеты записи настоящего хранилища «ключ-значение».
- [Документация crc32fast](https://docs.rs/crc32fast/latest/crc32fast/) — крейт контрольной суммы, который используется здесь.
- [Designing Data-Intensive Applications](https://dataintensive.net/) — глава о транзакциях очень понятно объясняет ACID и изоляцию.

---

[🏠 Начало](../START-HERE.md) · [← 17.9 Страничный кэш и пул буферов](09-page-cache-and-buffer-pool.md) · [📋 Этап 17](README.md) · [17.11 Уровни изоляции →](11-isolation-levels.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
