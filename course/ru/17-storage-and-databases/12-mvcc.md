<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 17.11 Уровни изоляции](11-isolation-levels.md) · [📋 Этап 17](README.md) · [17.13 Как выполняется SQL-запрос →](13-how-a-sql-query-runs.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/17-storage-and-databases/12-mvcc.md) <!-- nav -->

# Урок 17.12 — MVCC: многоверсионное управление конкурентным доступом

> **Этап 17 — Как устроены хранилища и базы данных** · Урок 12 из 15 · ⏱ около 2,5 часа (2 занятия)

## Что вы узнаете

- Почему хранение нескольких версий строки позволяет читателям и писателям работать, не блокируя друг друга.
- Как PostgreSQL хранит версии на странице таблицы и как увидеть их через `xmin`, `xmax`, `ctid` и `pageinspect`.
- Из чего состоит снимок и какое правило решает, какую версию видит транзакция.
- Почему старые версии накапливаются, что делает `VACUUM` и почему одна забытая транзакция может его остановить.
- Как построить на Rust крошечное MVCC-хранилище с изоляцией снимков и правилом «побеждает первый зафиксировавший».

## Перед началом

- Вы прошли [Урок 17.11 — Уровни изоляции](11-isolation-levels.md): снимки, `REPEATABLE READ`, перекос записи.
- Вы знаете наборы записей и фиксацию из [Урока 17.10 — Транзакции и ACID](10-transactions-and-acid.md) и страницы по
  8 КиБ (кибибайтов по 1024 байта) из [Урока 17.9 — Страничный кэш и пул буферов](09-page-cache-and-buffer-pool.md).
- Контейнер `p17-postgres` запущен, и вы умеете открывать два терминала `psql` (T1 и T2), как в Уроке 17.11.

## Новые слова

| Слово | Значение |
|---|---|
| **MVCC** (Multi-Version Concurrency Control, многоверсионное управление конкурентным доступом) | Хранение нескольких версий каждой строки, чтобы каждая транзакция могла читать версию, соответствующую её снимку. |
| **версия** (version; в PostgreSQL — *tuple*, кортеж) | Одна хранимая копия строки с пометками о том, какая транзакция её создала и какая удалила. |
| **идентификатор транзакции** (transaction ID, XID) | Число, которое PostgreSQL выдаёт транзакции, когда она впервые что-то пишет. Больше — значит «получила номер позже». |
| **`xmin` / `xmax`** | Скрытые столбцы каждой версии строки в PostgreSQL: XID, который её создал, и XID, который её удалил или заменил (0 — никто). |
| **`ctid`** | Физический адрес версии строки: (номер страницы, номер слота на странице). |
| **правило видимости** (visibility rule) | Правило, которое решает, существует ли версия для данного снимка. |
| **мёртвая версия** (dead tuple) | Версия, которую не видит ни один текущий или будущий снимок. Она только занимает место. |
| **`VACUUM`** | Команда PostgreSQL (и фоновый процесс *autovacuum*), которая удаляет мёртвые версии и делает их место снова пригодным. |
| **раздувание** (bloat) | Таблица или индекс гораздо больше своих живых данных из-за мёртвых версий и пустого места. |
| **побеждает первый зафиксировавший** (first committer wins) | Правило изоляции снимков: если две одновременные транзакции пишут один ключ, побеждает та, что зафиксировалась первой, а вторая прерывается. |
| **сборка мусора** (garbage collection) версий | Удаление версий, которые больше никто не может увидеть; `VACUUM` — её вариант в PostgreSQL. |

## 1. Идея: не перезаписывать, а добавлять версию

В Уроке 17.11 транзакция на `REPEATABLE READ` продолжала видеть старый баланс alice, пока другая транзакция его меняла.
Откуда бралось старое значение? База данных **его не перезаписала**. Обновление пишет **новую версию** строки и
сохраняет старую:

```text
 alice, 100   created by tx 10456, deleted by tx 10457     <- old version
 alice,  90   created by tx 10457, deleted by nobody       <- new version
```

Каждая транзакция читает из своего снимка, а снимок решает, какая версия для неё «существует». Это и есть **MVCC**
(Multi-Version Concurrency Control). Оно даёт знаменитое свойство:

> **Читатели никогда не блокируют писателей, а писатели никогда не блокируют читателей.**

Читателю не нужна блокировка, чтобы защитить строку от изменений, потому что изменение попадает в другую версию. Друг с
другом по-прежнему должны разбираться только два **писателя** одной строки (ждать блокировку строки или прерываться с
`40001`).

## 2. Версии в PostgreSQL

У каждой версии строки в PostgreSQL есть скрытые системные столбцы. Их можно выбрать по имени. В T1 начните со свежей
таблицы:

```sql
DROP TABLE accounts;
CREATE TABLE accounts (name text PRIMARY KEY, balance int NOT NULL);
INSERT INTO accounts VALUES ('alice', 100), ('bob', 50);
SELECT ctid, xmin, xmax, name, balance FROM accounts;
UPDATE accounts SET balance = 90 WHERE name = 'alice';
SELECT ctid, xmin, xmax, name, balance FROM accounts;
```

```text
 ctid  | xmin  | xmax | name  | balance
-------+-------+------+-------+---------
 (0,1) | 10456 |    0 | alice |     100
 (0,2) | 10456 |    0 | bob   |      50
(2 rows)

UPDATE 1
 ctid  | xmin  | xmax | name  | balance
-------+-------+------+-------+---------
 (0,2) | 10456 |    0 | bob   |      50
 (0,3) | 10457 |    0 | alice |     90
(2 rows)
```

Ваши идентификаторы транзакций будут другими числами; важно, как они соотносятся.

**Что происходит:**

1. `INSERT` выполнился как транзакция 10456, поэтому у обеих строк `xmin = 10456`. `xmax = 0`: их никто не удалял.
2. `ctid` `(0,1)` означает «страница 0, слот 1». Обе строки лежат на первой 8-КиБ странице таблицы.
3. После `UPDATE` alice находится в `(0,3)`: это **новый слот**, созданный транзакцией 10457. Старая версия в слоте 1
   всё ещё на странице, но ваш снимок её больше не показывает.

Теперь посмотрите на обновление, которое ещё не зафиксировано:

```text
step  T1                                                        T2
1     BEGIN;
2     UPDATE accounts SET balance = 80 WHERE name = 'alice';
3     SELECT pg_current_xact_id();  -> 10458
4                                                               SELECT ctid, xmin, xmax, name, balance FROM accounts;
5                                                               SELECT pg_current_snapshot();
6     COMMIT;
7                                                               SELECT ctid, xmin, xmax, name, balance FROM accounts;
```

Результаты T2 на шагах 4, 5 и 7:

```text
 ctid  | xmin  | xmax  | name  | balance
-------+-------+-------+-------+---------
 (0,2) | 10456 |     0 | bob   |      50
 (0,3) | 10457 | 10458 | alice |      90
(2 rows)

 pg_current_snapshot
---------------------
 10458:10458:
(1 row)

 ctid  | xmin  | xmax | name  | balance
-------+-------+------+-------+---------
 (0,2) | 10456 |    0 | bob   |      50
 (0,4) | 10458 |    0 | alice |      80
(2 rows)
```

**Что происходит:**

1. `pg_current_xact_id()` показывает XID транзакции T1 — 10458.
2. На шаге 4 T2 видит **старую** версию `(0,3)`, но с `xmax = 10458`: «транзакция 10458 хочет её удалить». Поскольку
   10458 ещё не зафиксирована, для T2 версия по-прежнему жива. T2 не ждала: читатели не блокируются.
3. Снимок записывается как `xmin:xmax:список`. `10458:10458:` означает: каждый XID меньше 10458 завершился; каждый XID
   начиная с 10458 считается **ещё не зафиксированным** (для этого снимка); список других работающих XID пуст.
4. После фиксации следующий оператор T2 берёт новый снимок (`READ COMMITTED`) и видит новую версию `(0,4)`.

## 3. Правило видимости

По скрытым столбцам и снимку PostgreSQL для каждой версии решает, существует ли она. Упрощённо:

```text
A version is visible to a snapshot if
    its creator (xmin) committed and is "in the past" of the snapshot
AND NOT
    its deleter (xmax) committed and is "in the past" of the snapshot.

"In the past" = the XID is below the snapshot's xmax and not in its list of running XIDs.
```

То есть версия видна снимку, если создавшая её транзакция (`xmin`) зафиксировалась и находится «в прошлом» снимка, и
при этом удалившая её транзакция (`xmax`) **не** зафиксировалась в прошлом снимка. «В прошлом» значит: XID меньше
`xmax` снимка и не входит в список работающих.

Зафиксировалась ли транзакция или откатилась, хранится в маленькой отдельной структуре — журнале фиксаций (`pg_xact`).
Версии откатившейся транзакции просто никогда не видны: так PostgreSQL получает **откат даром** (Урок 17.10).

> 💡 **Совет:** когда читатель впервые проверяет журнал фиксаций для версии, PostgreSQL записывает в неё маленькие
> «биты-подсказки» (hint bits: «создатель зафиксирован»), чтобы следующим читателям не приходилось смотреть снова.
> Поэтому простой `SELECT` может делать страницы грязными; после большого `INSERT` вы можете увидеть `dirtied=` в
> `EXPLAIN (ANALYZE, BUFFERS)`.

## 4. Мёртвые версии и `VACUUM`

Посмотрим на саму страницу. `pageinspect` — расширение, поставляемое с PostgreSQL, которое расшифровывает сырые
страницы. В T1:

```sql
CREATE EXTENSION pageinspect;
SELECT lp, t_xmin, t_xmax, t_ctid FROM heap_page_items(get_raw_page('accounts', 0));
```

```text
CREATE EXTENSION
 lp | t_xmin | t_xmax | t_ctid
----+--------+--------+--------
  1 |  10456 |  10457 | (0,3)
  2 |  10456 |      0 | (0,2)
  3 |  10457 |  10458 | (0,4)
  4 |  10458 |      0 | (0,4)
(4 rows)
```

Четыре версии для двух строк. `lp` — слот («указатель строки», line pointer). Слоты 1 и 3 — старые версии alice; их
`t_ctid` указывает на более новую версию, поэтому версии одной строки образуют цепочку. Обе старые версии **мертвы**:
удалившие их транзакции зафиксировались, и ни у одной работающей транзакции нет снимка, настолько старого, чтобы они ей
были нужны. Теперь приберёмся:

```sql
VACUUM accounts;
SELECT lp, lp_flags, t_xmin, t_xmax FROM heap_page_items(get_raw_page('accounts', 0));
```

```text
VACUUM
 lp | lp_flags | t_xmin | t_xmax
----+----------+--------+--------
  1 |        2 |        |
  2 |        1 |  10456 |      0
  3 |        0 |        |
  4 |        1 |  10458 |      0
(4 rows)
```

**Что происходит:**

1. `lp_flags = 1` означает «обычный»: в слотах 2 и 4 живые строки.
2. Слот 3 теперь `0`, «не используется»: его место может занять будущая версия.
3. Слот 1 — `2`, «перенаправление»: индекс первичного ключа всё ещё указывает на слот 1, поэтому слот остаётся
   крошечным указателем на живую версию в слоте 4. Сами мёртвые данные удалены.

Вручную `VACUUM` запускают редко: фоновый процесс **autovacuum** выполняет его сам, когда изменилось достаточно строк.

**Одна открытая транзакция может остановить `VACUUM`.** Версия мертва, только если она не нужна *ни одному* снимку. В
T2 откройте снимок и держите его; в T1 обновите и выполните очистку:

```text
step  T1                                                        T2
1                                                               BEGIN ISOLATION LEVEL REPEATABLE READ;
2                                                               SELECT balance FROM accounts WHERE name = 'alice';  -> 80
3     UPDATE accounts SET balance = 70 WHERE name = 'alice';
4     VACUUM (VERBOSE) accounts;
5                                                               COMMIT;
6     VACUUM (VERBOSE) accounts;
```

`VERBOSE` печатает отчёт; его строка `tuples:` на шаге 4 и на шаге 6:

```text
tuples: 0 removed, 3 remain, 1 are dead but not yet removable
tuples: 1 removed, 2 remain, 0 are dead but not yet removable
```

Пока снимок T2 был открыт, версия `alice, 80` была «мёртвой, но ещё не удаляемой»: T2 могла её прочитать. После
фиксации T2 второй `VACUUM` её удалил.

> ⚡ **Производительность:** сеанс, который выполнил `BEGIN` и ждёт (упавший клиент, открытый `psql` разработчика,
> долгий отчёт), держит свой снимок, и с этого момента `VACUUM` не может удалить **ни одну** версию, умершую после него,
> во всей базе данных. Таблицы растут (**раздувание**), сканирования читают больше страниц, пул буферов держит мусор.
> Следите за сеансами в состоянии `idle in transaction` в `pg_stat_activity` и задайте
> `idle_in_transaction_session_timeout`.

## 5. Крошечное MVCC-хранилище на Rust

Теперь построим ту же идею в несколько строк. Вместо XID и журнала фиксаций у нас один счётчик — **метка времени
фиксации** (commit timestamp):

- Каждая фиксация получает следующее число: 1, 2, 3, ...
- Транзакция запоминает `start_ts` — номер последней фиксации на момент её начала. Это число *и есть* её снимок.
- Версия видна транзакции, если её `commit_ts <= start_ts`; из таких побеждает самая новая.
- Записи ждут в наборе записей (как в Уроке 17.10). При фиксации, если другая транзакция после `start_ts` зафиксировала
  более новую версию одного из наших ключей, мы прерываемся: **побеждает первый зафиксировавший**. `REPEATABLE READ` в
  PostgreSQL приходит к тому же результату чуть раньше: второй *писатель* строки ждёт блокировку строки, которую держит
  первый, и как только первый фиксируется, второй получает `could not serialize access due to concurrent update`
  («побеждает первый обновивший»).

```bash
cd ~/projects/rust
cargo new --lib code/p17-mvcc
cd code/p17-mvcc
```

Замените `src/lib.rs` на:

```rust
use std::collections::BTreeMap;

/// One version of a value. `None` means "deleted in this version".
#[derive(Debug)]
struct Version {
    commit_ts: u64,
    value: Option<String>,
}

#[derive(Debug, Default)]
pub struct Mvcc {
    versions: BTreeMap<String, Vec<Version>>, // oldest version first
    clock: u64,                               // timestamp of the last commit
}

#[derive(Debug)]
pub struct Tx {
    start_ts: u64, // the snapshot: sees commits with commit_ts <= start_ts
    writes: BTreeMap<String, Option<String>>,
}

#[derive(Debug, PartialEq)]
pub struct Conflict {
    pub key: String,
}

impl Tx {
    pub fn put(&mut self, key: &str, value: &str) {
        self.writes.insert(key.to_string(), Some(value.to_string()));
    }

    pub fn delete(&mut self, key: &str) {
        self.writes.insert(key.to_string(), None);
    }
}

impl Mvcc {
    pub fn begin(&self) -> Tx {
        Tx {
            start_ts: self.clock,
            writes: BTreeMap::new(),
        }
    }

    /// Reads `key` as `tx` sees it: its own writes first, then its snapshot.
    pub fn get<'a>(&'a self, tx: &'a Tx, key: &str) -> Option<&'a str> {
        if let Some(own) = tx.writes.get(key) {
            return own.as_deref();
        }
        let versions = self.versions.get(key)?;
        let visible = versions.iter().rev().find(|v| v.commit_ts <= tx.start_ts)?;
        visible.value.as_deref()
    }

    /// Commits `tx`, unless another transaction committed a newer version of
    /// one of its keys after `tx` started ("first committer wins").
    pub fn commit(&mut self, tx: Tx) -> Result<u64, Conflict> {
        for key in tx.writes.keys() {
            let newest = self.versions.get(key).and_then(|v| v.last());
            if newest.is_some_and(|v| v.commit_ts > tx.start_ts) {
                return Err(Conflict { key: key.clone() });
            }
        }
        self.clock += 1;
        for (key, value) in tx.writes {
            let commit_ts = self.clock;
            self.versions
                .entry(key)
                .or_default()
                .push(Version { commit_ts, value });
        }
        Ok(self.clock)
    }

    /// How many versions are stored for `key` (live and dead).
    pub fn version_count(&self, key: &str) -> usize {
        self.versions.get(key).map_or(0, Vec::len)
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    fn store_with(key: &str, value: &str) -> Mvcc {
        let mut db = Mvcc::default();
        let mut tx = db.begin();
        tx.put(key, value);
        db.commit(tx).unwrap();
        db
    }

    #[test]
    fn a_snapshot_does_not_see_later_commits() {
        let mut db = store_with("x", "1");
        let reader = db.begin();
        let mut writer = db.begin();
        writer.put("x", "2");
        db.commit(writer).unwrap();
        assert_eq!(db.get(&reader, "x"), Some("1"));
        assert_eq!(db.get(&db.begin(), "x"), Some("2"));
    }

    #[test]
    fn the_second_writer_of_a_key_is_aborted() {
        let mut db = store_with("x", "1");
        let mut t1 = db.begin();
        let mut t2 = db.begin();
        t1.put("x", "from t1");
        t2.put("x", "from t2");
        assert!(db.commit(t1).is_ok());
        assert_eq!(db.commit(t2), Err(Conflict { key: "x".into() }));
    }
}
```

**Что происходит:**

1. `versions` отображает ключ в его версии, от старой к новой. Удаление — это версия с `value: None`, как `xmax` в
   PostgreSQL: ключ перестаёт существовать *начиная с этой версии*, а более старые снимки по-прежнему видят старое
   значение.
2. `begin` копирует часы: с этого момента транзакция видит ровно фиксации до `start_ts`. Взять снимок ничего не стоит.
3. `get` сначала смотрит в собственные записи транзакции (читать свои записи), затем идёт по версиям от новой к старой
   и берёт первую с `commit_ts <= start_ts`. `as_deref` превращает `&Option<String>` в `Option<&str>`.
4. `commit` сначала проверяет каждый записанный ключ. Если у самой новой версии ключа `commit_ts` больше нашего
   `start_ts`, кто-то зафиксировал её после нашего снимка, и наша запись основана на старых данных: `Conflict`.
5. Иначе часы идут дальше, и каждая запись становится новой версией с новой меткой. Старые версии остаются.
6. Два теста — это два обещания изоляции снимков: снимки стабильны, и два писателя одного ключа не могут оба победить.

```bash
cargo test
```

```text
running 2 tests
test tests::the_second_writer_of_a_key_is_aborted ... ok
test tests::a_snapshot_does_not_see_later_commits ... ok

test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

Снова потерянное обновление из Урока 17.10, теперь поверх MVCC. Создайте `src/bin/timeline.rs`:

```rust
use p17_mvcc::Mvcc;

fn main() {
    let mut db = Mvcc::default();
    let mut setup = db.begin();
    setup.put("alice", "100");
    println!("setup committed at ts {}", db.commit(setup).unwrap());

    let reader = db.begin(); // snapshot at ts 1
    let mut t1 = db.begin();
    let mut t2 = db.begin();
    t1.put("alice", "90");
    t2.put("alice", "80");
    println!("t1 committed at ts {}", db.commit(t1).unwrap());
    match db.commit(t2) {
        Ok(ts) => println!("t2 committed at ts {ts}"),
        Err(conflict) => println!("t2 aborted: {conflict:?}"),
    }

    println!(
        "reader (started at ts 1) sees alice = {:?}",
        db.get(&reader, "alice")
    );
    println!(
        "a new transaction sees alice = {:?}",
        db.get(&db.begin(), "alice")
    );
    println!("versions of alice stored: {}", db.version_count("alice"));
}
```

```bash
cargo run --bin timeline
```

```text
setup committed at ts 1
t1 committed at ts 2
t2 aborted: Conflict { key: "alice" }
reader (started at ts 1) sees alice = Some("100")
a new transaction sees alice = Some("90")
versions of alice stored: 2
```

t1 и t2 обе начались при ts 1 и обе записали alice. t1 зафиксировалась первой; t2 нашла версию новее своего снимка и
была прервана, вместо того чтобы молча потерять обновление t1. Старый читатель всё ещё видит 100, и хранятся две версии:
цена MVCC — место.

> 🧪 **Тестирование:** ошибки MVCC выглядят как «транзакция увидела то, чего не должна». Тестируйте явными временными
> линиями: запускайте транзакции в фиксированном порядке, фиксируйте в фиксированном порядке и проверяйте, что видит
> каждая. Упражнение 2 так доказывает *слабость* изоляции снимков.

## Частые ошибки

> ⚠️ **Частая ошибка:** думать, что `UPDATE` меняет строку на месте. В PostgreSQL он пишет новую версию, поэтому часто
> обновляемая таблица растёт, пока `VACUUM` не вернёт место. Обновление всех строк большой таблицы за раз может удвоить
> её размер.

> ⚠️ **Частая ошибка:** считать изоляцию снимков сериализуемой. Она останавливает потерянные обновления (побеждает
> первый зафиксировавший), но две транзакции, пишущие **разные** ключи, никогда не конфликтуют, поэтому перекос записи
> проходит (Упражнение 2). Именно поэтому `SERIALIZABLE` в PostgreSQL добавляет поверх SSI (Serializable Snapshot Isolation, сериализуемая изоляция снимков, Урок 17.11).

> ⚠️ **Частая ошибка:** держать транзакции открытыми, пока ждёте чего-то медленного (пользователя, HTTP-вызова — HTTP, Hypertext Transfer Protocol, протокол передачи гипертекста). Кроме
> удержания блокировок, открытый снимок останавливает очистку везде (раздел 4).

## Попробуйте сами

1. В T1 выполните `BEGIN; DELETE FROM accounts WHERE name = 'bob';` и не фиксируйте. В T2 выберите
   `ctid, xmin, xmax, name` из `accounts`: bob всё ещё там, а `xmax` равен XID транзакции T1. Затем выполните
   `ROLLBACK` в T1 и снова выберите в T2: `xmax` всё ещё показывает этот XID, но журнал фиксаций говорит, что она
   откатилась, поэтому версия жива.
2. Выполните `SELECT n_live_tup, n_dead_tup FROM pg_stat_user_tables WHERE relname = 'accounts';` после нескольких
   обновлений и ещё раз после `VACUUM`. (Эта статистика собирается в фоне и может отставать на секунду.)

## Упражнения

### Упражнение 1 — Какая версия видна? (легко)

У ключа `x` три версии: `"a"`, зафиксированная при ts 1, удаление, зафиксированное при ts 4, и `"c"`, зафиксированная
при ts 6. Что вернёт `get` для транзакций с `start_ts` 0, 1, 3, 4, 5 и 6? Ответьте на бумаге, затем напишите тест.

<details>
<summary>Подсказка</summary>

Берите самую новую версию с `commit_ts <= start_ts`. Версия-удаление означает «значения нет». Внутри модуля `tests` можно
напрямую построить `Tx { start_ts, writes: BTreeMap::new() }`, потому что приватные поля видны дочерним модулям.

</details>

<details>
<summary>Ответ</summary>

0 → ничего (x ещё не существовал), 1 → `"a"`, 3 → `"a"`, 4 → ничего (удаление), 5 → ничего, 6 → `"c"`.

```rust
    #[test]
    fn which_version_does_each_snapshot_see() {
        let mut db = Mvcc::default();
        let mut t = db.begin();
        t.put("x", "a");
        db.commit(t).unwrap(); // ts 1
        for _ in 0..2 {
            let mut t = db.begin();
            t.put("other", "-");
            db.commit(t).unwrap(); // ts 2 and 3
        }
        let mut t = db.begin();
        t.delete("x");
        db.commit(t).unwrap(); // ts 4
        let mut t = db.begin();
        t.put("other", "-");
        db.commit(t).unwrap(); // ts 5
        let mut t = db.begin();
        t.put("x", "c");
        db.commit(t).unwrap(); // ts 6

        let read_at = |start_ts| {
            let snapshot = Tx {
                start_ts,
                writes: BTreeMap::new(),
            };
            db.get(&snapshot, "x").map(String::from)
        };
        assert_eq!(read_at(0), None); // before x existed
        assert_eq!(read_at(1), Some("a".to_string()));
        assert_eq!(read_at(3), Some("a".to_string()));
        assert_eq!(read_at(4), None); // the delete is visible
        assert_eq!(read_at(5), None);
        assert_eq!(read_at(6), Some("c".to_string()));
    }
```

Почему это работает: снимок — это всего лишь число; правило видимости «самая новая версия не позже моей метки» отвечает
на каждый случай, включая удаления.

</details>

### Упражнение 2 — Докажите, что изоляция снимков допускает перекос записи (средне)

Напишите тест `snapshot_isolation_allows_write_skew` с врачами из Урока 17.11: ann и ben «дежурят»; две транзакции
каждая читают другого врача, видят «дежурит» и ставят себе «не дежурит». Проверьте, что **обе** фиксации успешны и что
после этого никто не дежурит.

<details>
<summary>Подсказка</summary>

Записываются ключи `"ann"` (транзакцией t1) и `"ben"` (транзакцией t2). `commit` проверяет только те ключи, которые
транзакция **пишет**.

</details>

<details>
<summary>Ответ</summary>

```rust
    #[test]
    fn snapshot_isolation_allows_write_skew() {
        let mut db = Mvcc::default();
        let mut setup = db.begin();
        setup.put("ann", "on call");
        setup.put("ben", "on call");
        db.commit(setup).unwrap();

        let mut t1 = db.begin();
        let mut t2 = db.begin();
        // both see two doctors on call, so each one may leave
        assert_eq!(db.get(&t1, "ben"), Some("on call"));
        assert_eq!(db.get(&t2, "ann"), Some("on call"));
        t1.put("ann", "off");
        t2.put("ben", "off");
        assert!(db.commit(t1).is_ok());
        assert!(db.commit(t2).is_ok()); // different keys: no conflict detected

        let now = db.begin();
        assert_eq!(db.get(&now, "ann"), Some("off"));
        assert_eq!(db.get(&now, "ben"), Some("off")); // nobody is on call
    }
```

Почему это работает: правило «побеждает первый зафиксировавший» сравнивает только наборы **записей**. Нарушенное
правило зависит от того, что транзакции **читали**, а это хранилище чтения не отслеживает. Тест, документирующий
известное ограничение, ценен: если кто-то потом добавит отслеживание чтений (как в SSI), этот тест точно скажет, какое
поведение изменилось.

</details>

### Упражнение 3 — Очистка (сложно)

Добавьте `pub fn vacuum(&mut self, oldest_start_ts: u64) -> usize`: удалите каждую версию, которую не может увидеть ни
одна транзакция с `start_ts >= oldest_start_ts`, и верните, сколько удалено. Для каждого ключа оставьте самую новую
версию с `commit_ts <= oldest_start_ts` и всё, что новее. Если осталась единственная версия-удаление, которую видят
все, удалите ключ полностью. Протестируйте со старым читателем, который держит старую версию живой.

<details>
<summary>Подсказка</summary>

`versions.iter().rposition(|v| v.commit_ts <= oldest_start_ts)` находит индекс версии, которую видит самый старый
снимок; всё до этого индекса можно удалить (`versions.drain(..i)`). В конце
`self.versions.retain(|_, v| !v.is_empty())` убирает пустые ключи.

</details>

<details>
<summary>Ответ</summary>

```rust
    /// Removes versions that no snapshot at or after `oldest_start_ts` can see.
    /// Returns how many versions were removed.
    pub fn vacuum(&mut self, oldest_start_ts: u64) -> usize {
        let mut removed = 0;
        for versions in self.versions.values_mut() {
            // the newest version that the oldest snapshot can see
            let keep_from = versions
                .iter()
                .rposition(|v| v.commit_ts <= oldest_start_ts)
                .unwrap_or(0);
            removed += keep_from;
            versions.drain(..keep_from);
            // a delete that every snapshot sees can go, too
            if versions.len() == 1
                && versions[0].value.is_none()
                && versions[0].commit_ts <= oldest_start_ts
            {
                versions.clear();
                removed += 1;
            }
        }
        self.versions.retain(|_, versions| !versions.is_empty());
        removed
    }
```

```rust
    #[test]
    fn vacuum_keeps_what_the_oldest_snapshot_needs() {
        let mut db = store_with("x", "1"); // ts 1
        let old_reader = db.begin(); // start_ts 1
        for value in ["2", "3"] {
            let mut tx = db.begin();
            tx.put("x", value);
            db.commit(tx).unwrap(); // ts 2, 3
        }
        assert_eq!(db.vacuum(old_reader.start_ts), 0); // "1" is still needed
        assert_eq!(db.get(&old_reader, "x"), Some("1"));

        drop(old_reader);
        assert_eq!(db.vacuum(db.begin().start_ts), 2); // only "3" is needed now
        assert_eq!(db.version_count("x"), 1);

        let mut tx = db.begin();
        tx.delete("x");
        db.commit(tx).unwrap(); // ts 4
        assert_eq!(db.vacuum(db.begin().start_ts), 2); // "3" and the delete
        assert_eq!(db.version_count("x"), 0);
    }
```

Почему это работает: версия может понадобиться только снимкам, у которых `start_ts` не меньше `oldest_start_ts`, а все
они видят версию с индексом `keep_from` или более новую. Вызывающий код должен передать `start_ts` самой старой
**работающей** транзакции; одна забытая транзакция держит `oldest_start_ts` маленьким, и ничего не удаляется — ровно
как сеанс `idle in transaction` в разделе 4.

</details>

## Вопросы для самопроверки

1. Почему при MVCC читатели не блокируют писателей?

<details>
<summary>Ответ</summary>

Писатель создаёт новую версию вместо изменения старой, поэтому читатель может продолжать читать версию, которую видит
его снимок. Для чтения ничего блокировать не нужно.

</details>

2. Что означают `xmin` и `xmax` версии строки в PostgreSQL?

<details>
<summary>Ответ</summary>

`xmin` — идентификатор транзакции, создавшей версию; `xmax` — идентификатор транзакции, которая её удалила или заменила
(0, если такой нет). Зафиксировались ли эти транзакции, смотрится в журнале фиксаций.

</details>

3. Снимок — `500:510:503,507`. Видна ли версия с `xmin = 505` (зафиксирована)? А с `xmin = 507`?

<details>
<summary>Ответ</summary>

505: да. Он меньше 510, не входит в список работающих транзакций и зафиксирован. 507: нет. Он в списке, то есть ещё
работал, когда снимок был взят.

</details>

4. Что удаляет `VACUUM` и что может его остановить?

<details>
<summary>Ответ</summary>

Мёртвые версии: версии, у которых удалившая их транзакция зафиксировалась и которые уже не видит ни один снимок. Его
останавливает долго открытая транзакция (или снимок), потому что версии, умершие после взятия этого снимка, могут ещё
понадобиться.

</details>

5. Как PostgreSQL «откатывает» транзакцию без информации для отката?

<details>
<summary>Ответ</summary>

Он ничего не меняет обратно. Журнал фиксаций помечает транзакцию как прерванную, поэтому её новые версии никогда не
видны, а «удалённые» ею версии остаются видимыми. Остатки потом убирает `VACUUM`.

</details>

## Итоги

- MVCC хранит несколько версий каждой строки; транзакция читает версию, которую видит её снимок, поэтому читатели и
  писатели не блокируют друг друга.
- Версии в PostgreSQL несут `xmin` (создатель) и `xmax` (удаливший); `ctid` — физический слот; `pageinspect` показывает
  каждую версию на странице.
- Снимок — это `xmin:xmax:список работающих`; версия видна, если её создатель зафиксировался в прошлом снимка, а
  удаливший — нет.
- Старые версии становятся мёртвыми; `VACUUM` (и autovacuum) возвращает их место, но только если ни один открытый снимок
  их больше не требует.
- Изоляция снимков = снимки + «побеждает первый зафиксировавший»: она останавливает потерянные обновления, но допускает
  перекос записи.
- В игрушечном хранилище снимок может быть одним числом — меткой последней фиксации.

## Источники и что почитать

- [PostgreSQL: Concurrency Control (introduction)](https://www.postgresql.org/docs/current/mvcc-intro.html) — MVCC словами самого PostgreSQL.
- [PostgreSQL: System Columns](https://www.postgresql.org/docs/current/ddl-system-columns.html) — `xmin`, `xmax`, `ctid`.
- [PostgreSQL: System Information Functions](https://www.postgresql.org/docs/current/functions-info.html) — `pg_current_xact_id` и `pg_current_snapshot` с форматом снимка.
- [PostgreSQL: Routine Vacuuming](https://www.postgresql.org/docs/current/routine-vacuuming.html) — зачем нужна очистка, autovacuum, переполнение счётчика транзакций (wraparound).
- [PostgreSQL: pageinspect](https://www.postgresql.org/docs/current/pageinspect.html) и [Database Page Layout](https://www.postgresql.org/docs/current/storage-page-layout.html) — инструменты и формат страницы из раздела 4.
- [PostgreSQL: Heap-Only Tuples (HOT)](https://www.postgresql.org/docs/current/storage-hot.html) — почему слот 1 стал перенаправлением.

---

[🏠 Начало](../START-HERE.md) · [← 17.11 Уровни изоляции](11-isolation-levels.md) · [📋 Этап 17](README.md) · [17.13 Как выполняется SQL-запрос →](13-how-a-sql-query-runs.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
