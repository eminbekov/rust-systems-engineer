<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 18.5 Логические часы: часы Лэмпорта и векторные часы](05-logical-clocks.md) · [📋 Этап 18](README.md) · [18.7 Модели согласованности →](07-consistency-models.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/18-distributed-systems/06-replication.md) <!-- nav -->

# Урок 18.6 — Репликация

> **Этап 18 — Распределённые системы** · Урок 6 из 15 · ⏱ около 135 минут

## Что вы узнаете

- Зачем системы хранят несколько копий одних и тех же данных и чего стоит каждая копия.
- Как работает репликация с одним лидером: один лидер, журнал репликации и последователи, которые применяют его по порядку.
- Что такое отставание репликации и почему последователь может вернуть старые данные.
- Почему асинхронная репликация может потерять уже подтверждённые записи и чего стоит синхронная репликация.
- Как системы без лидера используют кворумы и почему `R + W > N` гарантирует, что каждое чтение встретит последнюю запись.

## Перед началом

- Вы прошли [Урок 18.5 — Логические часы](05-logical-clocks.md).
- Вы знаете журнал упреждающей записи из [Урока 17.4](../17-storage-and-databases/04-write-ahead-log.md): репликация пересылает журнал изменений точно так же.
- Создайте пакет для этого урока. Зависимости не нужны; весь «кластер» имитируется в памяти, чтобы каждый сбой
  случался ровно тогда, когда нам нужно:

```bash
cd ~/projects/rust
cargo new --lib code/p18-replication
cd code/p18-replication
mkdir src/bin tests
```

## Новые слова

| Слово | Значение |
|---|---|
| **реплика** (replica) | Одна копия данных, обычно на своей машине. |
| **репликация** (replication) | Поддержание нескольких реплик одних и тех же данных в актуальном состоянии. |
| **лидер** (leader) | Единственная реплика, которая принимает записи и решает их порядок. Её также называют *primary* (основной). |
| **последователь** (follower) | Реплика, которая только копирует изменения лидера и может обслуживать чтения. Её также называют *replica*, *secondary* или *standby*. |
| **журнал репликации** (replication log) | Упорядоченный список изменений, который лидер отправляет своим последователям. |
| **отставание репликации** (replication lag) | Насколько последователь отстаёт от лидера: сколько изменений он ещё не применил. |
| **синхронная / асинхронная репликация** | Лидер подтверждает запись клиенту только после того, как она есть у последователей (синхронная), или сразу (асинхронная). |
| **переключение на резерв** (failover) | Назначение последователя новым лидером, когда старый лидер отказал. |
| **разделение мозга** (split brain) | Два узла одновременно считают себя лидером и оба принимают записи. |
| **репликация без лидера** (leaderless replication) | Клиенты пишут в несколько реплик и читают из них напрямую; ни одна реплика не особенная. |
| **кворум** (quorum) | Минимальное число реплик, которые должны ответить, чтобы операция засчиталась. **N** — число реплик, **W** — кворум записи, **R** — кворум чтения. |
| **исправление при чтении** (read repair) | Когда чтение находит реплику со старым значением, оно записывает в неё самое новое значение. |

## 1. Зачем хранить копии?

Репликация означает хранение одних и тех же данных на нескольких машинах. Причин четыре:

| Причина | Пример |
|---|---|
| **Доступность**: продолжать работать при отказе машины | Лидер умер; за секунды его место занимает последователь. |
| **Надёжность хранения**: не терять данные, когда умирает диск | Каждая строка есть на трёх дисках в трёх стойках. |
| **Масштабирование чтения**: больше машин отвечают на чтения | Десять последователей обслуживают каталог товаров; один лидер принимает редкие записи. |
| **Задержка**: копия рядом с пользователем | Реплика в каждом регионе отвечает на чтения локально. |

Если бы данные никогда не менялись, репликация была бы тривиальной: скопировать один раз. Вся сложность — в
**изменениях**. Каждая запись должна дойти до каждой реплики, сеть может её потерять или задержать (Урок 18.2), а
реплики могут упасть посередине. Остаток урока — о разных способах с этим справиться.

## 2. Репликация с одним лидером

Самая распространённая схема, которую используют, например, PostgreSQL и алгоритм Raft, который вы построите позже:

```text
           writes                   replication log
client ──────────────► leader  [1: x=1] [2: x=2] [3: y=5] ...
                          │
             ┌────────────┴────────────┐
             ▼                         ▼
       follower 0                 follower 1
   applied 3 of 3 entries     applied 2 of 3 entries   ← behind
             ▲                         ▲
             └──── reads may go to followers ──┘
```

1. Записи принимает только **лидер**. Каждое изменение он кладёт в свой **журнал репликации**, по порядку. Журнал
   очень похож на журнал упреждающей записи из Урока 17.4.
2. **Последователи** получают записи журнала и применяют их **в том же порядке**. Поскольку они начинают с одного
   состояния и применяют одни и те же изменения в одном порядке, у них оказываются одинаковые данные.
3. Чтения могут идти к лидеру (всегда актуален) или к последователям (дешевле, но могут отставать).

Построим это в памяти. Поместите это в `src/lib.rs`:

```rust
//! Replication (Lesson 18.6).

use std::collections::HashMap;

/// One change in the replication log.
#[derive(Debug, Clone, PartialEq)]
pub struct Entry {
    pub key: String,
    pub value: String,
}

/// One copy of the data. It applies log entries in order and counts them.
#[derive(Debug, Default)]
pub struct Replica {
    data: HashMap<String, String>,
    applied: usize,
}

impl Replica {
    pub fn apply(&mut self, entries: &[Entry]) {
        for entry in entries {
            self.data.insert(entry.key.clone(), entry.value.clone());
        }
        self.applied += entries.len();
    }

    pub fn get(&self, key: &str) -> Option<&str> {
        self.data.get(key).map(String::as_str)
    }

    /// How many log entries this replica has applied.
    pub fn applied(&self) -> usize {
        self.applied
    }
}

/// One leader and its followers, with asynchronous replication.
#[derive(Default)]
pub struct Cluster {
    pub log: Vec<Entry>,
    pub leader: Replica,
    pub followers: Vec<Replica>,
}

impl Cluster {
    pub fn new(followers: usize) -> Self {
        let mut cluster = Cluster::default();
        cluster.followers.resize_with(followers, Replica::default);
        cluster
    }

    /// A client write. The leader appends it to the log, applies it and answers "OK"
    /// at once, without waiting for any follower. Returns the entry's position in the log.
    pub fn write(&mut self, key: &str, value: &str) -> usize {
        let entry = Entry {
            key: key.to_string(),
            value: value.to_string(),
        };
        self.leader.apply(std::slice::from_ref(&entry));
        self.log.push(entry);
        self.log.len()
    }

    /// Sends follower `i` every log entry that it has not applied yet.
    pub fn replicate(&mut self, i: usize) {
        let follower = &mut self.followers[i];
        follower.apply(&self.log[follower.applied..]);
    }

    /// The leader has crashed: follower `i` becomes the new leader with the data it has.
    /// Returns the entries that the old leader had confirmed but that are now lost.
    pub fn fail_over(&mut self, i: usize) -> Vec<Entry> {
        self.leader = self.followers.remove(i);
        self.log.split_off(self.leader.applied)
    }
}
```

**Что происходит:**

1. `Replica` — это отображение «ключ-значение» плюс число применённых записей журнала. Поскольку записи применяются по
   порядку и никогда не пропускаются, одно это число точно говорит, какие записи есть у реплики.
2. `Cluster::write` — сторона лидера при записи: добавить в журнал, применить и сразу вернуть позицию. Он не ждёт ни
   одного последователя: это **асинхронная** репликация.
3. `replicate(i)` — одна «сетевая доставка» последователю `i`: она пересылает записи от `applied` до конца журнала. В
   настоящей системе это всё время происходит в фоне; здесь **мы** решаем когда, поэтому можем устроить любую
   задержку.
4. `fail_over(i)` имитирует упавшего лидера: последователь `i` становится лидером, а журнал обрезается до того, что
   есть у нового лидера. `split_off` возвращает отрезанные записи: записи, которые старый лидер подтвердил, но которых
   больше ни у кого не было.

## 3. Отставание репликации и потерянные записи

Сначала отставание. `src/bin/lag.rs` записывает `x = 1`, реплицирует везде, затем записывает `x = 2` и реплицирует
только на последователя 0:

```rust
use p18_replication::Cluster;

fn main() {
    let mut cluster = Cluster::new(2);
    cluster.write("x", "1");
    cluster.replicate(0);
    cluster.replicate(1);

    cluster.write("x", "2");
    cluster.replicate(0); // follower 1 does not get the new entry yet

    println!("leader:     x = {:?}", cluster.leader.get("x"));
    for (i, follower) in cluster.followers.iter().enumerate() {
        let lag = cluster.log.len() - follower.applied();
        println!("follower {i}: x = {:?} (lag: {lag})", follower.get("x"));
    }
}
```

`cargo run --bin lag`:

```text
leader:     x = Some("2")
follower 0: x = Some("2") (lag: 0)
follower 1: x = Some("1") (lag: 1)
```

Клиент, читающий с последователя 1, видит **старое** значение. В здоровой системе отставание — миллисекунды, но под
нагрузкой или при сетевых проблемах оно может составлять секунды и минуты. Пользователь, который сменил имя и
перезагрузил страницу с отстающего последователя, видит старое имя и думает, что изменение пропало. Упражнение 3
исправляет именно это.

Теперь переключение на резерв. `src/bin/failover.rs` подтверждает запись клиенту, а затем роняет лидера до того, как
запись переслана:

```rust
use p18_replication::Cluster;

fn main() {
    let mut cluster = Cluster::new(2);
    cluster.write("balance", "100");
    cluster.replicate(0);
    cluster.replicate(1);

    let position = cluster.write("balance", "150");
    println!("client: write number {position} (balance = 150) confirmed");

    // The leader crashes before it sends entry 2 to anybody.
    let lost = cluster.fail_over(0);
    println!("new leader: balance = {:?}", cluster.leader.get("balance"));
    println!("lost, although confirmed: {lost:?}");
}
```

`cargo run --bin failover`:

```text
client: write number 2 (balance = 150) confirmed
new leader: balance = Some("100")
lost, although confirmed: [Entry { key: "balance", value: "150" }]
```

Клиенту сказали «OK». Записи нет. Это цена асинхронной репликации, и настоящие базы данных с асинхронными
последователями ведут себя при переключении ровно так: последние несколько подтверждённых записей могут пропасть.

Альтернативы:

| Режим | Лидер отвечает «OK», когда… | Цена |
|---|---|---|
| асинхронный | он сам записал изменение | подтверждённые записи могут пропасть при переключении |
| синхронный | изменение есть у **каждого** последователя | один медленный или мёртвый последователь останавливает все записи |
| полусинхронный | изменение есть у **одного** (или нескольких) последователей | немного медленнее; переживает потерю лидера |

Многие настоящие конфигурации выбирают середину: один синхронный последователь, остальные асинхронные. Raft
(Урок 18.11) использует **большинство**: запись подтверждена, когда она есть больше чем у половины узлов, поэтому она
переживает потерю любого меньшинства.

> ⚠️ **Частая ошибка: два лидера.** Переключение решают другие узлы, обычно потому что старый лидер перестал отвечать
> на heartbeat-сообщения (Урок 18.2). Но старый лидер может быть просто медленным или отрезанным. Если он вернётся и
> продолжит принимать записи, получится **разделение мозга**: два лидера, две расходящиеся истории. Лекарство — из
> Урока 18.4: число, растущее с каждым новым лидером (токен ограждения; в Raft это *терм*), которое проверяется при
> каждой записи.

> 🧪 **Тестирование:** поскольку симуляция детерминирована, обе истории — хорошие тесты. Добавьте в конец
> `src/lib.rs`:

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn a_follower_that_is_behind_returns_old_data() {
        let mut cluster = Cluster::new(2);
        cluster.write("x", "1");
        cluster.replicate(0);
        cluster.replicate(1);
        cluster.write("x", "2");
        cluster.replicate(0);
        assert_eq!(cluster.followers[0].get("x"), Some("2"));
        assert_eq!(cluster.followers[1].get("x"), Some("1")); // stale
    }

    #[test]
    fn asynchronous_failover_loses_confirmed_writes() {
        let mut cluster = Cluster::new(1);
        cluster.write("x", "1");
        cluster.replicate(0);
        cluster.write("x", "2"); // confirmed to the client, never replicated
        let lost = cluster.fail_over(0);
        assert_eq!(cluster.leader.get("x"), Some("1"));
        assert_eq!(lost.len(), 1);
    }
}
```

## 4. Репликация без лидера и кворумы

У некоторых баз данных (в стиле Dynamo от Amazon) лидера **нет** вообще. При N репликах клиент отправляет каждую
запись всем N и считает её успешной, когда подтвердили **W** из них. Каждое чтение он отправляет всем N, ждёт **R**
ответов и берёт значение с самой новой версией.

Почему это работает? Если каждая запись дошла хотя бы до W реплик, каждое чтение спрашивает R реплик и `R + W > N`,
то две группы **обязаны** иметь хотя бы одну общую реплику: реплик просто не хватит, чтобы группы разминулись. У этой
общей реплики и есть самое новое значение.

```text
N = 3, W = 2, R = 2:       replica 1   replica 2   replica 3
    write reached            new         new         old
    read asked                           new         old    → sees "new" through replica 2
```

Верить правилу на слово не нужно: для маленьких N его можно проверить, перебрав **все** возможные группы записи и
чтения. Группа реплик — это множество; запишем его числом, биты которого говорят, какие реплики в него входят (бит 0 —
реплика 1 и так далее, как в Уроке 12.4). У двух групп есть общая реплика, если их побитовое И не равно нулю. Добавьте
в `src/lib.rs` (перед тестами):

```rust
/// Checks every possible write set of size `w` and read set of size `r` among `n` replicas:
/// true if each pair shares at least one replica.
pub fn quorums_always_overlap(n: u32, w: u32, r: u32) -> bool {
    let sets_of = |size: u32| (0u32..1 << n).filter(move |set| set.count_ones() == size);
    sets_of(w).all(|write_set| sets_of(r).all(|read_set| write_set & read_set != 0))
}
```

**Что происходит:**

1. `0u32..1 << n` — все числа от 0 до 2ⁿ − 1, то есть все подмножества из n реплик.
2. `set.count_ones() == size` оставляет только подмножества с нужным числом членов.
3. `all(... all(...))` проверяет каждую пару из группы записи и группы чтения. `move` копирует `size` во внутреннее
   замыкание, чтобы `sets_of` можно было вызвать дважды.

И таблица, `src/bin/quorums.rs`:

```rust
use p18_replication::quorums_always_overlap;

fn main() {
    let n = 3;
    for (w, r) in [(1, 1), (2, 1), (2, 2), (3, 1), (1, 3)] {
        let overlap = quorums_always_overlap(n, w, r);
        println!(
            "N={n} W={w} R={r}: R+W > N is {:5}  sets always overlap: {overlap}",
            r + w > n
        );
    }
}
```

`cargo run --bin quorums`:

```text
N=3 W=1 R=1: R+W > N is false  sets always overlap: false
N=3 W=2 R=1: R+W > N is false  sets always overlap: false
N=3 W=2 R=2: R+W > N is true   sets always overlap: true
N=3 W=3 R=1: R+W > N is true   sets always overlap: true
N=3 W=1 R=3: R+W > N is true   sets always overlap: true
```

Полный перебор совпадает с формулой в каждой строке. Превратите его в тест для N от 1 до 5 и всех W и R; добавьте
внутрь `mod tests`:

```rust
    #[test]
    fn overlapping_quorums_follow_r_plus_w_greater_than_n() {
        for n in 1..=5 {
            for w in 1..=n {
                for r in 1..=n {
                    assert_eq!(
                        quorums_always_overlap(n, w, r),
                        r + w > n,
                        "n={n} w={w} r={r}"
                    );
                }
            }
        }
    }
```

`cargo test --lib`:

```text
test tests::asynchronous_failover_loses_confirmed_writes ... ok
test tests::a_follower_that_is_behind_returns_old_data ... ok
test tests::overlapping_quorums_follow_r_plus_w_greater_than_n ... ok

test result: ok. 3 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

Частые варианты: N = 3 с W = 2 и R = 2 (переживает отказ одной реплики и для чтения, и для записи) или W = N и R = 1
(быстрые чтения, но одна мёртвая реплика блокирует записи). Реплики, пропустившие запись, исправляют двумя способами:
**исправление при чтении** (чтение, увидевшее старую версию, записывает новую обратно; Упражнение 2) и фоновый
процесс, который сравнивает реплики.

> ⚡ **Производительность:** операция с кворумом ждёт W-й или R-й по скорости ответ, а не самый медленный. При N = 3 и
> R = 2 одна медленная реплика вообще не замедляет чтения. Это одна из главных причин выбирать схемы без лидера.

Но кворумы — не волшебство. Две одновременные записи могут дойти до разных реплик в разном порядке (чтобы это
обнаружить, номерам версий нужны векторные часы из Урока 18.5), а запись, не удавшаяся на части реплик, может
остаться на других. Урок 18.7 показывает, какие гарантии такая система может и не может дать.

| | Один лидер | Несколько лидеров | Без лидера |
|---|---|---|---|
| Кто принимает записи | один лидер | по лидеру на регион или устройство | любая реплика |
| Порядок записей | решает лидер | конфликты нужно объединять | конфликты нужно обнаруживать и объединять |
| Когда лидер отказывает | нужно переключение | другие лидеры продолжают | ничего особенного |
| Типичное применение | большинство реляционных баз данных, Raft | несколько дата-центров, офлайн-приложения | хранилища с очень высокой доступностью |

## Частые ошибки

> ⚠️ **Частая ошибка: читать свою запись с последователя.** После записи чтение с отстающего последователя может её не
> увидеть. Какое-то время после записи читайте с лидера или используйте позицию записи, как в Упражнении 3.

## Попробуйте сами

1. В `failover.rs` вызовите `cluster.replicate(0);` сразу после второй записи и запустите. Ничего не потеряно, потому
   что у нового лидера запись была.
2. Вместо этого реплицируйте на последователя 1 и переключитесь на последователя 0. Запись снова потеряна: важно,
   была ли она у **нового лидера**.

## Упражнения

### Упражнение 1 — Синхронные записи (легко)

Добавьте в `Cluster` метод `pub fn write_sync(&mut self, key: &str, value: &str) -> usize`: как `write`, но перед
возвратом он реплицирует на каждого последователя. Напишите интеграционный тест `tests/sync_and_sessions.rs`, который
делает две синхронные записи, переключается на последователя 0 и проверяет, что ничего не потеряно, а у нового лидера
второе значение.

<details>
<summary>Подсказка</summary>

`write_sync` может вызвать `self.write(...)`, а затем `self.replicate(i)` для каждого `i` из `0..self.followers.len()`.

</details>

<details>
<summary>Ответ</summary>

В `impl Cluster`:

```rust
    /// A write that is confirmed only after every follower has applied it.
    pub fn write_sync(&mut self, key: &str, value: &str) -> usize {
        let position = self.write(key, value);
        for i in 0..self.followers.len() {
            self.replicate(i);
        }
        position
    }
```

В `tests/sync_and_sessions.rs`:

```rust
use p18_replication::Cluster;

#[test]
fn synchronous_writes_survive_a_failover() {
    let mut cluster = Cluster::new(2);
    cluster.write_sync("balance", "100");
    cluster.write_sync("balance", "150");
    let lost = cluster.fail_over(0);
    assert!(lost.is_empty());
    assert_eq!(cluster.leader.get("balance"), Some("150"));
}
```

Почему это работает: каждая подтверждённая запись есть у каждого последователя до того, как клиент услышит «OK»,
поэтому какой бы последователь ни стал лидером, у него будут все записи. Цена: если бы один последователь не работал,
`write_sync` пришлось бы его ждать, и в настоящей системе записи остановились бы.

</details>

### Упражнение 2 — Исправление при чтении (средне)

Создайте `src/leaderless.rs` (и `pub mod leaderless;`) с `VersionedReplica`, которая хранит `(version, value)` для
каждого ключа и при `put` оставляет только более высокую версию, и функцию
`pub fn quorum_read(replicas: &mut [VersionedReplica], key: &str, ask: &[usize]) -> Option<String>`, которая читает из
реплик, перечисленных в `ask`, возвращает самое новое значение и записывает его обратно в опрошенные реплики со старым
значением. Тест: у трёх реплик версия 1; версия 2 доходит до реплик 0 и 1 (W = 2); чтение с реплик 1 и 2 (R = 2)
возвращает новое значение и исправляет реплику 2.

<details>
<summary>Подсказка</summary>

`ask.iter().filter_map(|&i| replicas[i].get(key)).max_by_key(|(version, _)| *version)` находит самый новый ответ.
Возвращайте хранимую пару по значению (`cloned()`), чтобы потом можно было заимствовать `replicas` изменяемо для
исправления.

</details>

<details>
<summary>Ответ</summary>

```rust
//! Exercise 2: a leaderless store with read repair.

use std::collections::HashMap;

/// A replica in a leaderless store: every value carries a version number.
#[derive(Debug, Default)]
pub struct VersionedReplica {
    data: HashMap<String, (u64, String)>,
}

impl VersionedReplica {
    /// Stores the value unless the replica already has the same or a newer version.
    pub fn put(&mut self, key: &str, version: u64, value: &str) {
        let newer = match self.data.get(key) {
            Some((old, _)) => version > *old,
            None => true,
        };
        if newer {
            self.data
                .insert(key.to_string(), (version, value.to_string()));
        }
    }

    pub fn get(&self, key: &str) -> Option<(u64, String)> {
        self.data.get(key).cloned()
    }
}

/// Reads `key` from the replicas listed in `ask`, returns the newest value, and writes it
/// back to the asked replicas that had an older version or none (read repair).
pub fn quorum_read(replicas: &mut [VersionedReplica], key: &str, ask: &[usize]) -> Option<String> {
    let newest = ask
        .iter()
        .filter_map(|&i| replicas[i].get(key))
        .max_by_key(|(version, _)| *version)?;
    for &i in ask {
        replicas[i].put(key, newest.0, &newest.1);
    }
    Some(newest.1)
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn a_read_finds_the_newest_value_and_repairs_the_stale_replica() {
        let mut replicas: Vec<VersionedReplica> =
            (0..3).map(|_| VersionedReplica::default()).collect();
        for replica in replicas.iter_mut() {
            replica.put("x", 1, "old");
        }
        // A write with W = 2 reached replicas 0 and 1 only.
        replicas[0].put("x", 2, "new");
        replicas[1].put("x", 2, "new");

        // A read with R = 2 asks replicas 1 and 2.
        assert_eq!(
            quorum_read(&mut replicas, "x", &[1, 2]),
            Some("new".to_string())
        );
        assert_eq!(replicas[2].get("x"), Some((2, "new".to_string()))); // repaired
    }
}
```

`cargo test --lib`:

```text
test result: ok. 4 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

Почему это работает: поскольку R + W > N, хотя бы у одной опрошенной реплики (здесь реплики 1) есть версия 2.
Записать её обратно в остальные ничего не стоит читающему и повышает шанс, что следующее чтение найдёт новое значение
везде.

</details>

### Упражнение 3 — Читать свои записи (сложно)

Напишите в `src/lib.rs` функцию
`pub fn read_your_writes(cluster: &Cluster, i: usize, key: &str, min_position: usize) -> Result<Option<String>, String>`.
Клиент передаёт позицию, которую `write` вернул для его последней записи; функция читает с последователя `i`, только
если тот применил как минимум столько записей, и иначе возвращает ошибку (клиент может тогда попробовать другую реплику
или лидера). Добавьте тест в `tests/sync_and_sessions.rs`: после записи, которая есть только у последователя 0,
последователь 0 возвращает новое значение, а последователь 1 — ошибку.

<details>
<summary>Подсказка</summary>

Сравните `follower.applied()` с `min_position`. Возвращайте значение как владеющую `String` (`map(str::to_string)`),
потому что результат не должен заимствовать из кластера.

</details>

<details>
<summary>Ответ</summary>

```rust
/// Reads `key` from follower `i`, but only if it has applied at least `min_position`
/// entries, the position of the client's own last write.
pub fn read_your_writes(
    cluster: &Cluster,
    i: usize,
    key: &str,
    min_position: usize,
) -> Result<Option<String>, String> {
    let follower = &cluster.followers[i];
    if follower.applied() < min_position {
        return Err(format!(
            "follower {i} has applied {} of {min_position} entries",
            follower.applied()
        ));
    }
    Ok(follower.get(key).map(str::to_string))
}
```

Тест, добавленный в `tests/sync_and_sessions.rs` (и измените его первую строку на
`use p18_replication::{Cluster, read_your_writes};`):

```rust
#[test]
fn a_client_never_reads_older_data_than_its_own_write() {
    let mut cluster = Cluster::new(2);
    cluster.write("name", "Ann");
    cluster.replicate(0);
    cluster.replicate(1);
    let mine = cluster.write("name", "Anna"); // the client remembers the position: 2
    cluster.replicate(0);

    assert_eq!(
        read_your_writes(&cluster, 0, "name", mine),
        Ok(Some("Anna".to_string()))
    );
    assert!(read_your_writes(&cluster, 1, "name", mine).is_err()); // follower 1 is behind
}
```

`cargo test --test sync_and_sessions`:

```text
test synchronous_writes_survive_a_failover ... ok
test a_client_never_reads_older_data_than_its_own_write ... ok

test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

Почему это работает: позиция в журнале — это обещание «всё до этого места». Если последователь применил столько
записей, у него есть запись клиента. Эта гарантия называется *read-your-writes* («читай свои записи»); Урок 18.7
перечисляет её вместе с другими гарантиями сессии.

</details>

## Вопросы для самопроверки

1. Почему последователи применяют записи журнала лидера точно в том же порядке?

<details>
<summary>Ответ</summary>

Потому что, начав с одного состояния, одни и те же изменения в одном порядке дают один результат. В другом порядке
`x = 1` и `x = 2` могли бы закончиться значением 1 на одной реплике и 2 на другой.

</details>

2. Запись подтвердили клиенту, а затем лидер упал. При каком режиме репликации запись может потеряться?

<details>
<summary>Ответ</summary>

При асинхронном: лидер подтвердил запись раньше, чем она появилась хоть у одного последователя, и у нового лидера её
может не быть. При синхронной репликации (или подтверждённом большинстве) она у нового лидера есть.

</details>

3. Что такое разделение мозга и что его предотвращает?

<details>
<summary>Ответ</summary>

Два узла одновременно ведут себя как лидер и принимают записи, и данные расходятся. Номер лидера (терм или токен
ограждения), растущий с каждым новым лидером и проверяемый при каждой записи, позволяет остальной системе отвергнуть
старого лидера.

</details>

4. N = 5. Какие из пар гарантируют, что чтение встретит последнюю запись: W = 3, R = 3; W = 2, R = 3; W = 5, R = 1?

<details>
<summary>Ответ</summary>

W = 3, R = 3 (6 > 5) и W = 5, R = 1 (6 > 5). W = 2, R = 3 даёт 5, а это не больше 5, поэтому чтение может разминуться
с записью.

</details>

## Итоги

- Репликация хранит копии ради доступности, надёжности хранения, масштабирования чтения и задержки; вся сложность — в
  изменениях.
- Репликация с одним лидером: лидер упорядочивает все записи в журнале, последователи применяют его в том же порядке.
- Последователи отстают, поэтому чтения с них могут вернуть старые данные.
- Асинхронная репликация может потерять подтверждённые записи при переключении; синхронная останавливается на
  медленном последователе; большинство (Raft) — середина.
- Переключение грозит разделением мозга; растущий номер лидера (терм, токен ограждения) его предотвращает.
- Системы без лидера читают и пишут кворумами; `R + W > N` заставляет группы чтения и записи пересекаться, а
  исправление при чтении чинит устаревшие реплики.

## Источники и что почитать

- [Designing Data-Intensive Applications](https://dataintensive.net/) — глава о репликации подробно разбирает лидеров и последователей, отставание репликации, схемы с несколькими лидерами и без лидера.
- [PostgreSQL: Log-Shipping Standby Servers](https://www.postgresql.org/docs/current/warm-standby.html) — как настоящая база данных пересылает журнал упреждающей записи последователям и что там означает синхронная репликация.
- [Dynamo: Amazon's Highly Available Key-value Store](https://www.allthingsdistributed.com/files/amazon-dynamo-sosp2007.pdf) — статья, на которой основана репликация без лидера с кворумами N, W и R и исправлением при чтении.
- [Redis replication](https://redis.io/docs/latest/operate/oss_and_stack/management/replication/) — короткое практическое описание асинхронной репликации «лидер — последователь» и её компромиссов.

---

[🏠 Начало](../START-HERE.md) · [← 18.5 Логические часы: часы Лэмпорта и векторные часы](05-logical-clocks.md) · [📋 Этап 18](README.md) · [18.7 Модели согласованности →](07-consistency-models.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
