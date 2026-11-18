<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 18.10 Raft: выбор лидера](10-raft-leader-election.md) · [📋 Этап 18](README.md) · [18.12 Детерминированное тестирование симуляцией →](12-deterministic-simulation-testing.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/18-distributed-systems/11-raft-log-replication.md) <!-- nav -->

# Урок 18.11 — Raft: репликация журнала

> **Этап 18 — Распределённые системы** · Урок 11 из 15 · ⏱ около 3 часов (2 занятия)

## Что вы узнаете

- Как лидер Raft копирует свой журнал последователям через `AppendEntries` и как последователь проверяет, что его журнал совпадает, прежде чем принять новые записи.
- Как лидер чинит последователя, чей журнал отстаёт или содержит записи, которые нужно выбросить.
- Когда запись становится **зафиксированной** и почему лидер может считать копии только для записей своего терма.
- Почему новый лидер первым делом пишет *пустую запись* (no-op).
- Как проверить «каждая реплика применяет одни и те же команды в одном и том же порядке» при разделениях сети и потере сообщений.

## Перед началом

- Вы прошли [Урок 18.10 — Raft: выбор лидера](10-raft-leader-election.md), и у вас есть его пакет `code/p18-raft`, все тесты которого проходят. Этот урок изменяет этот пакет.
- Вспомните картинку журнала из [Урока 18.9](09-consensus-and-raft.md): у каждой записи есть индекс (с 1) и терм; запись зафиксирована, когда она сохранена на большинстве.
- Если хотите, держите открытым рисунок 2 из статьи о Raft (см. источники). Каждое правило этого урока — одна строка того рисунка.

## Новые слова

| Слово | Значение |
|---|---|
| **`AppendEntries`** | Сообщение лидера последователю: «вот записи после индекса `prev_log_index`, чей терм `prev_log_term`; фиксируй до `leader_commit`». Без записей это пульс. |
| **проверка согласованности** (consistency check) | Проверка последователя перед добавлением: есть ли у меня запись с индексом `prev_log_index` и термом `prev_log_term`? Если нет, он отказывает. |
| **`next_index`** | Для каждого последователя — индекс следующей записи, которую лидер ему отправит. После отказа он сдвигается назад, после успеха — вперёд. |
| **`match_index`** | Для каждого последователя — наибольший индекс, о котором лидер *знает*, что у последователя там то же самое. По нему лидер считает копии. |
| **индекс фиксации** (commit index) | Наибольший индекс, о котором известно, что он зафиксирован. Узел применяет записи до него. |
| **применить** (apply) | Передать зафиксированную команду конечному автомату (Урок 18.9). Здесь конечный автомат — просто список `applied`. |
| **пустая запись** (no-op entry) | Запись без команды. Новый лидер добавляет её, чтобы суметь зафиксировать записи прежних термов (и узнать о них). |
| **совпадение журналов** (log matching) | Обещание Raft: если в двух журналах есть запись с одним и тем же индексом и термом, журналы совпадают вплоть до этого индекса. |
| **конфликт** (conflict) | У последователя на каком-то индексе запись с другим термом, чем у записи лидера на этом индексе. Запись последователя (и всё после неё) никогда не была зафиксирована и удаляется. |

## 1. Лидер отправляет, последователь проверяет

Когда клиент хочет изменить данные, он отправляет команду лидеру. Лидер добавляет её в свой журнал (со своим текущим термом) и отправляет каждому последователю в сообщении `AppendEntries`. Сообщение также говорит, **что идёт прямо перед** новыми записями:

```text
leader's log:     index  1    2    3    4
                  term  [1]  [1]  [2]  [2]
                                   ^    ^
                    prev_log_index=3    new entry
                    prev_log_term=2

AppendEntries { term: 2, prev_log_index: 3, prev_log_term: 2,
                entries: [ (2, "c=7") ], leader_commit: 3 }
```

Последователь принимает сообщение, только если в его журнале есть запись с индексом 3 и термом 2. Это и есть **проверка согласованности**. Почему достаточно одной записи? По индукции: та же проверка выполнялась, когда добавлялась запись 3, и когда добавлялась запись 2, и так далее. Если записи с индексом 3 совпадают, всё перед ними тоже совпадает. Это обещание **совпадения журналов**, и стоит оно одно сравнение на сообщение.

Если проверка не прошла, последователь отвечает «нет». Тогда лидер отступает на один шаг (`next_index - 1`) и пробует снова с более ранним `prev_log_index`, пока не найдёт место, где оба журнала совпадают. Начиная оттуда он отправляет всё, чего не хватает последователю. По пути последователь удаляет свои записи, которые **конфликтуют** с записями лидера:

```text
before:   leader   [1 a] [1 b] [2 c]
          follower [1 a] [1 b] [1 x] [1 y]     (x, y from an old leader, never committed)

try prev=3 (term 2): follower has term 1 at index 3 -> refuse
try prev=2 (term 1): match -> entries [2 c] replace [1 x] [1 y]

after:    follower [1 a] [1 b] [2 c]
```

Удалять записи звучит опасно, но это безопасно: запись, конфликтующая с журналом текущего лидера, **никогда** не могла быть зафиксирована, потому что правило выборов из Урока 18.10 гарантирует, что у лидера есть каждая зафиксированная запись.

## 2. Новые сообщения и новое состояние

Откройте `src/node.rs`. Замените варианты `Heartbeat` и `HeartbeatReply` двумя сообщениями репликации и обновите `Message::term`:

```rust
#[derive(Debug, Clone, PartialEq)]
pub enum Message {
    RequestVote {
        term: u64,
        last_log_index: u64,
        last_log_term: u64,
    },
    VoteReply {
        term: u64,
        granted: bool,
    },
    AppendEntries {
        term: u64,
        prev_log_index: u64,
        prev_log_term: u64,
        entries: Vec<Entry>,
        leader_commit: u64,
    },
    AppendReply {
        term: u64,
        success: bool,
        match_index: u64,
    },
}

impl Message {
    pub fn term(&self) -> u64 {
        match self {
            Message::RequestVote { term, .. }
            | Message::VoteReply { term, .. }
            | Message::AppendEntries { term, .. }
            | Message::AppendReply { term, .. } => *term,
        }
    }
}
```

**Что происходит:**

1. `AppendEntries` несёт проверку согласованности (`prev_log_index`, `prev_log_term`), новые записи (их может не быть: тогда это пульс) и индекс фиксации лидера.
2. `AppendReply` говорит, прошла ли проверка. При успехе `match_index` — индекс последней записи, которая теперь у последователя такая же, как у лидера, поэтому лидер точно узнаёт, докуда дошёл этот последователь.

Добавьте `BTreeMap` в импорты (`use std::collections::{BTreeMap, BTreeSet};`) и пять полей в конец `struct Node`:

```rust
    pub commit_index: u64,
    last_applied: u64,
    pub applied: Vec<String>,
    next_index: BTreeMap<NodeId, u64>,
    match_index: BTreeMap<NodeId, u64>,
```

Инициализируйте их в `Node::new`, после `outbox: Vec::new(),`:

```rust
            commit_index: 0,
            last_applied: 0,
            applied: Vec::new(),
            next_index: BTreeMap::new(),
            match_index: BTreeMap::new(),
```

`commit_index` и `last_applied` начинаются с 0 (ничего не зафиксировано и не применено). `applied` — наш конечный автомат: список уже применённых команд, без пустых записей. `next_index` и `match_index` использует только лидер.

Замените `last_log` этими двумя вспомогательными функциями; `term_at` теперь используется повсюду:

```rust
    /// Index and term of the last log entry, (0, 0) for an empty log.
    fn last_log(&self) -> (u64, u64) {
        let index = self.log.len() as u64;
        (index, self.term_at(index))
    }

    /// Term of the entry at `index`; 0 for index 0 (before the first entry).
    fn term_at(&self, index: u64) -> u64 {
        if index == 0 {
            0
        } else {
            self.log[index as usize - 1].term
        }
    }
```

Индекс 0 означает «перед первой записью»; его терм — 0. На индексе 0 любой журнал совпадает с любым другим, поэтому лидер всегда может откатиться к отправке всего журнала.

## 3. Как стать лидером, и пустая запись

В `tick` лидер больше не шлёт `Heartbeat`: замените `self.send_heartbeats();` на `self.replicate();` (её мы напишем в разделе 5). Затем измените `check_votes` и удалите `send_heartbeats`:

```rust
    fn check_votes(&mut self) {
        if self.role == Role::Candidate && self.votes.len() >= self.majority() {
            self.role = Role::Leader;
            self.elapsed = 0;
            let next = self.log.len() as u64 + 1;
            for &peer in &self.peers {
                self.next_index.insert(peer, next);
                self.match_index.insert(peer, 0);
            }
            // A no-op entry of our own term lets us commit older entries.
            self.log.push(Entry {
                term: self.term,
                command: String::new(),
            });
            self.replicate();
        }
    }
```

**Что происходит:**

1. Новый лидер не знает, докуда дошёл каждый последователь. Он начинает с оптимизмом: `next_index` — «сразу после моей последней записи», `match_index` — 0 («я пока ничего не знаю»). Отказы сдвинут `next_index` назад, где нужно.
2. Он добавляет **пустую запись** — запись с пустой командой — в своём терме и сразу реплицирует.

Зачем пустая запись? Раздел 6 объясняет правило фиксации: лидер может считать копии только записей **своего терма**. У нового лидера могут быть записи прежних термов, которые уже зафиксированы, но он не может этого знать. Пока не зафиксирована хотя бы одна запись его терма, он вообще не может продвинуть индекс фиксации, даже если клиенты больше никогда ничего не запишут. Пустая запись решает это сразу после выборов. Раздел 7 показывает тест, который без неё падает.

## 4. Обработка сообщений

В `step` замените две ветки пульса этими двумя:

```rust
            Message::AppendEntries {
                term,
                prev_log_index,
                prev_log_term,
                entries,
                leader_commit,
            } => {
                let (success, match_index) = if term < self.term {
                    (false, 0) // an old leader: refuse; the reply tells it our term
                } else {
                    self.role = Role::Follower;
                    self.reset_election_timer();
                    self.accept_entries(prev_log_index, prev_log_term, entries, leader_commit)
                };
                let reply = Message::AppendReply {
                    term: self.term,
                    success,
                    match_index,
                };
                self.send(from, reply);
            }
            Message::AppendReply {
                term,
                success,
                match_index,
            } => {
                if self.role == Role::Leader && term == self.term {
                    self.handle_append_reply(from, success, match_index);
                }
            }
```

**Что происходит:**

1. `AppendEntries` из более старого терма приходит от старого лидера: отказать. Наш ответ несёт наш более новый терм, поэтому старый лидер уходит с поста (общее правило в начале `step`).
2. Иначе отправитель — лидер нашего терма. Кандидат этого терма сдаётся; все перезапускают таймер выборов. Затем `accept_entries` (следующий раздел) выполняет основную работу и возвращает значения для ответа.
3. `AppendReply` использует только лидер и только если ответ из текущего терма. Старый ответ из прежнего терма игнорируется.

## 5. Код репликации

Добавьте четвёртый блок `impl` в конец `src/node.rs`:

```rust
// Log replication.
impl Node {
    /// A client asks the leader to add a command. Returns its log index.
    pub fn propose(&mut self, command: &str) -> Option<u64> {
        if self.role != Role::Leader {
            return None;
        }
        self.log.push(Entry {
            term: self.term,
            command: command.to_string(),
        });
        self.replicate();
        self.advance_commit(); // a 1-node cluster commits at once
        Some(self.log.len() as u64)
    }

    /// Leader: send every follower the entries it is missing (maybe none).
    fn replicate(&mut self) {
        for peer in self.peers.clone() {
            self.send_append(peer);
        }
    }

    fn send_append(&mut self, peer: NodeId) {
        let prev_log_index = self.next_index[&peer] - 1;
        let msg = Message::AppendEntries {
            term: self.term,
            prev_log_index,
            prev_log_term: self.term_at(prev_log_index),
            entries: self.log[prev_log_index as usize..].to_vec(),
            leader_commit: self.commit_index,
        };
        self.send(peer, msg);
    }

    /// Follower: check that our log matches the leader's at `prev_log_index`,
    /// then append. Returns (success, index of the last entry we now share).
    fn accept_entries(
        &mut self,
        prev_log_index: u64,
        prev_log_term: u64,
        entries: Vec<Entry>,
        leader_commit: u64,
    ) -> (bool, u64) {
        if prev_log_index > self.log.len() as u64 || self.term_at(prev_log_index) != prev_log_term {
            return (false, 0);
        }
        let last_new = prev_log_index + entries.len() as u64;
        for (i, entry) in entries.into_iter().enumerate() {
            let index = prev_log_index + 1 + i as u64;
            if index <= self.log.len() as u64 {
                if self.term_at(index) == entry.term {
                    continue; // we already have this entry
                }
                // Conflict: drop this entry and everything after it.
                self.log.truncate(index as usize - 1);
            }
            self.log.push(entry);
        }
        if leader_commit > self.commit_index {
            self.commit_index = leader_commit.min(last_new);
            self.apply();
        }
        (true, last_new)
    }

    fn handle_append_reply(&mut self, from: NodeId, success: bool, match_index: u64) {
        if success {
            let known = self.match_index[&from];
            self.match_index.insert(from, known.max(match_index));
            self.next_index.insert(from, known.max(match_index) + 1);
            self.advance_commit();
        } else {
            // The follower's log does not match yet: step back and retry.
            let next = self.next_index[&from];
            self.next_index.insert(from, (next - 1).max(1));
            self.send_append(from);
        }
    }

    /// Leader: commit the highest index stored on a majority,
    /// counting only entries from the current term.
    fn advance_commit(&mut self) {
        let last = self.log.len() as u64;
        for n in (self.commit_index + 1..=last).rev() {
            if self.term_at(n) != self.term {
                break;
            }
            let copies = 1 + self.match_index.values().filter(|&&m| m >= n).count();
            if copies >= self.majority() {
                self.commit_index = n;
                self.apply();
                return;
            }
        }
    }

    /// Give every newly committed command to the state machine, in order.
    fn apply(&mut self) {
        while self.last_applied < self.commit_index {
            self.last_applied += 1;
            let entry = &self.log[self.last_applied as usize - 1];
            if !entry.command.is_empty() {
                self.applied.push(entry.command.clone()); // skip no-op entries
            }
        }
    }
}
```

**Что происходит:**

1. `propose` — то, как команда клиента попадает в систему. Её принимает только лидер; остальные возвращают `None` (настоящий сервер сказал бы клиенту, кто лидер). Лидер добавляет запись со своим термом и реплицирует её.
2. `replicate` отправляет каждому последователю `AppendEntries`. Она выполняется и на каждом тике пульса, поэтому последователь, пропустивший сообщение, получит записи снова через несколько тиков. Потеря сообщений исправляется простой повторной отправкой.
3. `send_append` строит сообщение по `next_index`: всё от `next_index` до конца журнала плюс индекс и терм прямо перед этим. Для последователя, который ни в чём не отстаёт, `entries` пуст: это пульс.
4. `accept_entries` — сторона последователя. Сначала проверка согласованности: если наш журнал слишком короткий или на `prev_log_index` у нас другой терм, отказать. Затем каждая запись: если она у нас уже есть (тот же индекс, тот же терм), пропустить её; если там другая запись, это конфликт: `truncate` удаляет её и всё после неё; затем добавить. Пропуск важен, потому что сеть может с опозданием доставить старое, более короткое `AppendEntries`, и оно не должно отрезать уже имеющиеся записи.
5. Новый индекс фиксации последователя — это индекс лидера, но никогда не дальше последней записи, которую подтвердило это сообщение (`last_new`). Записи после неё ещё могут быть заменены.
6. `handle_append_reply` — сторона лидера. При успехе она двигает `match_index` и `next_index` вперёд (через `max`, чтобы опоздавший старый ответ не сдвинул их назад) и пытается фиксировать. При отказе она отступает `next_index` на один шаг, но не ниже 1, и сразу отправляет снова.
7. `advance_commit` ищет наибольший индекс `n`, сохранённый на большинстве (сам лидер плюс каждый последователь с `match_index >= n`). Она смотрит только на записи **текущего терма** и останавливается на первой более старой. Почему — объясняет раздел 6.
8. `apply` передаёт каждую новую зафиксированную запись конечному автомату по порядку индексов, ровно один раз. `last_applied` помнит, докуда дошли. Пустые записи пропускаются.

> ⚡ **Производительность:** отступать с `next_index` на одну запись за круг обмена медленно для последователя, отставшего на тысячи записей. Настоящие реализации позволяют последователю ответить с подсказкой («мой журнал кончается индексом 120» или «моя запись на 300 имеет терм 4, который начинается с индекса 280»), чтобы лидер отступил за один шаг. Ещё они ограничивают число записей в одном сообщении.

## 6. Когда запись зафиксирована?

«Сохранена на большинстве» — почти всё правило. Недостающая часть тонкая. Представьте такую ситуацию в кластере из 5 узлов:

```text
          index:  1    2
node 1 (leader) [1]  [2]      leader of term 2 wrote entry 2 to itself and node 2, then crashed
node 2          [1]  [2]
node 3          [1]
node 4          [1]
node 5          [1]  [3]      leader of term 3 wrote a different entry 2, crashed
```

Теперь узел 1 снова становится лидером, в терме 4, и копирует свою запись 2 (из терма 2) на узел 3. Запись 2 есть на узлах 1, 2 и 3: это большинство. Зафиксирована ли она? **Нет.** Если узел 1 сейчас упадёт, узел 5 всё ещё может победить на выборах (его последний терм 3 побеждает 2 у остальных; узлы 2, 3 и 4 голосуют за него) и заменить запись 2 везде своей. Запись, которая была «на большинстве», исчезла.

Это знаменитый рисунок 8 из статьи о Raft, а исправление — правило в `advance_commit`: **лидер фиксирует, только считая копии записи своего терма.** Когда запись терма 4 оказалась на большинстве, ни один узел с более старым последним термом не может выиграть выборы, поэтому эта запись и всё перед ней в безопасности. Более ранние записи фиксируются *косвенно*. Поэтому же важна пустая запись из раздела 3: это та запись терма 4, которая позволяет сразу зафиксировать более старые.

## 7. Посмотрим, потом проверим

Обновите `src/cluster.rs`. Добавьте в сеть потерю сообщений: два новых поля, метод-сеттер и одну проверку в `tick`. Добавьте наверху `use rand::rngs::StdRng;` и `use rand::{RngExt, SeedableRng};`, затем:

```rust
pub struct Cluster {
    pub nodes: BTreeMap<NodeId, Node>,
    in_flight: VecDeque<Envelope>,
    cut_off: BTreeSet<NodeId>,
    loss_percent: u32,
    net_rng: StdRng,
}
```

В `Cluster::new` добавьте в литерал структуры `loss_percent: 0,` и `net_rng: StdRng::seed_from_u64(seed),`. Затем добавьте сеттер после `heal` и проверку потери в `tick`:

```rust
    /// Lose this percentage of all messages, chosen by the seeded RNG.
    pub fn set_message_loss(&mut self, percent: u32) {
        self.loss_percent = percent;
    }

    /// One tick for every node, then deliver messages until none are left.
    pub fn tick(&mut self) {
        for node in self.nodes.values_mut() {
            node.tick();
        }
        self.collect();
        while let Some(env) = self.in_flight.pop_front() {
            if self.cut_off.contains(&env.from) || self.cut_off.contains(&env.to) {
                continue; // the network loses it
            }
            if self.net_rng.random_range(0..100) < self.loss_percent {
                continue; // an unlucky message
            }
            let node = self.nodes.get_mut(&env.to).expect("unknown node");
            node.step(env.from, env.msg);
            self.collect();
        }
    }
```

И наконец две вспомогательные функции в конец `impl Cluster`, чтобы найти лидера и отправить ему команду:

```rust
    /// The leader with the highest term, if there is one.
    pub fn leader(&self) -> Option<NodeId> {
        self.leaders()
            .into_iter()
            .max_by_key(|&(_, term)| term)
            .map(|(id, _)| id)
    }

    /// Give a command to the current leader. False if there is none.
    pub fn propose(&mut self, command: &str) -> bool {
        match self.leader() {
            Some(id) => self.nodes.get_mut(&id).unwrap().propose(command).is_some(),
            None => false,
        }
    }
```

`leader` выбирает узел с наибольшим термом среди тех, кто считает себя лидером, потому что изолированный старый лидер тоже может так считать.

Теперь посмотрим, как разделение сети чинит журнал. Создайте `src/bin/replicate.rs`:

```rust
use p18_raft::Config;
use p18_raft::cluster::Cluster;

fn show(cluster: &Cluster, title: &str) {
    println!("{title}");
    for node in cluster.nodes.values() {
        let log: Vec<String> = node
            .log
            .iter()
            .map(|e| {
                let cmd = if e.command.is_empty() {
                    "-"
                } else {
                    &e.command
                };
                format!("{}:{cmd}", e.term)
            })
            .collect();
        println!(
            "  node {} {:?} term {} commit {} log [{}]",
            node.id,
            node.role,
            node.term,
            node.commit_index,
            log.join(" ")
        );
    }
}

fn run(cluster: &mut Cluster, ticks: u32) {
    for _ in 0..ticks {
        cluster.tick();
    }
}

fn main() {
    let mut cluster = Cluster::new(3, Config::default(), 7);
    run(&mut cluster, 15);
    cluster.propose("a");
    cluster.propose("b");
    run(&mut cluster, 5);
    show(&cluster, "1. two writes, replicated and committed:");

    let old = cluster.leader().unwrap();
    cluster.isolate(old);
    cluster.nodes.get_mut(&old).unwrap().propose("lost");
    run(&mut cluster, 40);
    cluster.propose("c");
    run(&mut cluster, 5);
    show(&cluster, "2. old leader cut off, a new leader wrote c:");

    cluster.heal(old);
    run(&mut cluster, 5);
    show(
        &cluster,
        "3. after healing, the old leader's log is repaired:",
    );
}
```

**Что происходит:**

1. `show` печатает журнал каждого узла как `терм:команда`, с `-` для пустой записи.
2. Шаг 1: две записи в здоровом кластере. Шаг 2: лидер отрезан, но всё ещё считает себя лидером, поэтому мы даём ему запись (`lost`) напрямую. Большинство выбирает нового лидера, и тот записывает `c`. Шаг 3: старый лидер вернулся.

Запустите `cargo run --bin replicate`:

```text
1. two writes, replicated and committed:
  node 1 Leader term 1 commit 3 log [1:- 1:a 1:b]
  node 2 Follower term 1 commit 3 log [1:- 1:a 1:b]
  node 3 Follower term 1 commit 3 log [1:- 1:a 1:b]
2. old leader cut off, a new leader wrote c:
  node 1 Leader term 1 commit 3 log [1:- 1:a 1:b 1:lost]
  node 2 Leader term 2 commit 5 log [1:- 1:a 1:b 2:- 2:c]
  node 3 Follower term 2 commit 5 log [1:- 1:a 1:b 2:- 2:c]
3. after healing, the old leader's log is repaired:
  node 1 Follower term 2 commit 5 log [1:- 1:a 1:b 2:- 2:c]
  node 2 Leader term 2 commit 5 log [1:- 1:a 1:b 2:- 2:c]
  node 3 Follower term 2 commit 5 log [1:- 1:a 1:b 2:- 2:c]
```

На шаге 1 каждый журнал начинается с пустой записи терма 1, затем `a` и `b`; все три зафиксированы. На шаге 2 у узла 1 на индексе 4 стоит `1:lost`, но его индекс фиксации остаётся 3: он никогда не дотянется до большинства. Узел 2 выиграл терм 2, записал свою пустую запись и `c` и зафиксировал их вместе с узлом 3. На шаге 3 узел 1 получает `AppendEntries` с термом 2, уходит с поста, проваливает проверку согласованности на индексе 4, и после одного шага назад его `1:lost` заменяется на `2:- 2:c`. Запись `lost` так и не была зафиксирована, поэтому ни одному клиенту не сказали, что она удалась. Потерять её — правильно.

Теперь тесты. Создайте `tests/replication.rs`:

```rust
use p18_raft::Config;
use p18_raft::cluster::Cluster;
use rand::rngs::StdRng;
use rand::{RngExt, SeedableRng};

fn run(cluster: &mut Cluster, ticks: u32) {
    for _ in 0..ticks {
        cluster.tick();
    }
}

#[test]
fn every_node_applies_the_same_commands() {
    let mut cluster = Cluster::new(3, Config::default(), 1);
    run(&mut cluster, 50);
    for cmd in ["a=1", "b=2", "a=3"] {
        assert!(cluster.propose(cmd));
    }
    run(&mut cluster, 10);
    for node in cluster.nodes.values() {
        assert_eq!(node.applied, ["a=1", "b=2", "a=3"], "node {}", node.id);
    }
}

#[test]
fn a_lagging_follower_catches_up() {
    let mut cluster = Cluster::new(3, Config::default(), 2);
    run(&mut cluster, 50);
    let leader = cluster.leader().unwrap();
    let lagging = if leader == 1 { 2 } else { 1 };

    cluster.isolate(lagging);
    for i in 0..5 {
        cluster.propose(&format!("x={i}"));
    }
    run(&mut cluster, 10);
    // Two of three nodes are a majority: the leader commits without it.
    assert_eq!(cluster.nodes[&leader].applied.len(), 5);
    assert_eq!(cluster.nodes[&lagging].applied.len(), 0);

    cluster.heal(lagging);
    run(&mut cluster, 10);
    assert_eq!(
        cluster.nodes[&lagging].applied,
        cluster.nodes[&leader].applied
    );
}

#[test]
fn an_uncommitted_entry_of_a_cut_off_leader_is_replaced() {
    let mut cluster = Cluster::new(5, Config::default(), 3);
    run(&mut cluster, 50);
    assert!(cluster.propose("first"));
    run(&mut cluster, 10);

    // The leader is cut off, but still believes it leads: it accepts a write.
    let old = cluster.leader().unwrap();
    cluster.isolate(old);
    assert!(
        cluster
            .nodes
            .get_mut(&old)
            .unwrap()
            .propose("lost")
            .is_some()
    );
    run(&mut cluster, 60);
    assert_eq!(cluster.nodes[&old].applied, ["first"], "never committed");

    // The majority elects a new leader and commits a different write.
    assert!(cluster.propose("second"));
    run(&mut cluster, 10);

    cluster.heal(old);
    run(&mut cluster, 20);
    for node in cluster.nodes.values() {
        assert_eq!(node.applied, ["first", "second"], "node {}", node.id);
    }
}

/// One of two lists must be the beginning of the other.
fn is_prefix(a: &[String], b: &[String]) -> bool {
    let n = a.len().min(b.len());
    a[..n] == b[..n]
}

#[test]
fn state_machine_safety_under_chaos() {
    for seed in 0..200 {
        let mut cluster = Cluster::new(5, Config::default(), seed);
        cluster.set_message_loss(20);
        let mut chaos = StdRng::seed_from_u64(seed + 10_000);
        let mut next_cmd = 0;

        for _ in 0..400 {
            match chaos.random_range(0..20) {
                0 => cluster.isolate(chaos.random_range(1..=5)),
                1 => cluster.heal(chaos.random_range(1..=5)),
                2..=5 => {
                    cluster.propose(&format!("cmd{next_cmd}"));
                    next_cmd += 1;
                }
                _ => {}
            }
            cluster.tick();

            let applied: Vec<&Vec<String>> = cluster.nodes.values().map(|n| &n.applied).collect();
            for a in &applied {
                for b in &applied {
                    assert!(is_prefix(a, b), "seed {seed}: replicas disagree");
                }
            }
        }

        // Liveness: after the network is repaired, all nodes catch up...
        for id in 1..=5 {
            cluster.heal(id);
        }
        cluster.set_message_loss(0);
        run(&mut cluster, 100);
        let applied = &cluster.nodes[&1].applied;
        for node in cluster.nodes.values() {
            assert_eq!(&node.applied, applied, "seed {seed}: node {}", node.id);
        }
        // ...and a new write still commits everywhere.
        assert!(cluster.propose("final"), "seed {seed}: no leader");
        run(&mut cluster, 20);
        for node in cluster.nodes.values() {
            assert_eq!(node.applied.last().unwrap(), "final", "seed {seed}");
        }
    }
}
```

**Что происходит:**

1. `every_node_applies_the_same_commands`: три записи в здоровом кластере доходят до каждого конечного автомата по порядку.
2. `a_lagging_follower_catches_up`: когда один из трёх узлов отрезан, два других составляют большинство и фиксируют; когда он возвращается, он догоняет.
3. `an_uncommitted_entry_of_a_cut_off_leader_is_replaced`: та же история, что и в примере, в виде теста, на 5 узлах.
4. `state_machine_safety_under_chaos`: для 200 зёрен, по 400 тиков каждое, при потере 20 % всех сообщений, когда случайные узлы отрезаются и возвращаются, а запись предлагается примерно на каждом пятом тике. После **каждого** тика он проверяет главное обещание: для любых двух узлов один список `applied` — начало (префикс) другого. Узлы могут отставать, но никогда не различаются. В конце он чинит сеть и проверяет живучесть: все догоняют, и последняя запись применяется везде.

```text
test every_node_applies_the_same_commands ... ok
test a_lagging_follower_catches_up ... ok
test an_uncommitted_entry_of_a_cut_off_leader_is_replaced ... ok
test state_machine_safety_under_chaos ... ok
test result: ok. 4 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.65s
```

Запустите весь набор через `cargo test`: тесты выборов из Урока 18.10 тоже по-прежнему проходят.

Теперь посмотрим, чего стоит пустая запись. Удалите строки `self.log.push(Entry { … })` (пустую запись) из `check_votes` и снова запустите `cargo test --test replication`:

```text
test state_machine_safety_under_chaos ... FAILED

thread 'state_machine_safety_under_chaos' (6268561) panicked at tests/replication.rs:123:13:
assertion `left == right` failed: seed 16: node 4
  left: ["cmd9", "cmd10", "cmd11", "cmd12", "cmd13", "cmd18", "cmd19", "cmd20", "cmd21", "cmd22", "cmd23", "cmd24", "cmd25", "cmd26", "cmd27", "cmd28", "cmd29", "cmd38", "cmd39", "cmd40", "cmd41", "cmd42", "cmd43", "cmd44", "cmd45", "cmd46", "cmd47", "cmd48", "cmd59", "cmd60", "cmd61"]
 right: ["cmd9", "cmd10", "cmd11", "cmd12", "cmd13", "cmd18", "cmd19", "cmd20", "cmd21", "cmd22", "cmd23", "cmd24", "cmd25", "cmd26", "cmd27", "cmd28", "cmd29", "cmd38", "cmd39", "cmd40", "cmd41", "cmd42", "cmd43", "cmd44", "cmd45"]
```

Безопасность всё ещё соблюдается (один список — префикс другого), но после починки узел 4 применил 31 команду, а узел 1 — только 25. Старый лидер зафиксировал `cmd46` … `cmd61`, а у нового лидера без записи своего терма нет способа зафиксировать их снова. Он ждал бы следующей записи клиента. С пустой записью он фиксирует их сразу после выборов. Верните пустую запись.

Вспомните обещание Урока 18.10: сломайте правило актуальности на выборах (`let log_ok = true;`) и запустите тесты. Теперь, когда журналы не пусты, тест хаоса падает на самом первом зерне (`seed 0: node 2`): был выбран лидер без некоторых зафиксированных записей, и он их перезаписал. Верните правило.

> 🧪 **Тестирование:** тест хаоса проверяет свойства («один список — префикс другого», «все догоняют»), а не точный вывод. Это тестирование на основе свойств из [Урока 09.4](../09-testing-debugging-benchmarking/04-property-based-testing.md), применённое к целому кластеру. Зерно играет роль сгенерированного входа, и каждый сбой воспроизводим.

## Частые ошибки

> ⚠️ **Частая ошибка:** обрезать журнал последователя на каждом `AppendEntries`. Тогда опоздавшее более старое сообщение с меньшим числом записей удалило бы записи, которые уже сохранены и, может быть, уже посчитаны лидером. Обрезайте только при настоящем конфликте (тот же индекс, другой терм).

> ⚠️ **Частая ошибка:** ставить индекс фиксации последователя в `leader_commit` без `min(last_new)`. Последователь может пометить зафиксированными записи, которые он ещё не получил от этого лидера, и применить устаревшие записи старого лидера.

> ⚠️ **Частая ошибка:** фиксировать записи старых термов подсчётом копий (раздел 6). Код выглядит разумно и проходит большинство тестов; а потом, при редкой последовательности падений, зафиксированная запись исчезает. Наш тест хаоса эту ошибку не поймал (попробуйте: удалите строки `if self.term_at(n) != self.term { break; }`): ситуация с рисунка 8 требует, чтобы лидеры падали в очень определённые моменты.

> ⚠️ **Частая ошибка:** применять запись до того, как она зафиксирована, например сразу, как только она пришла на последователь. Незафиксированные записи ещё могут быть заменены, а применённую команду «отменить» нельзя.

## Попробуйте сами

1. Запустите `cargo run --bin replicate`. Затем дайте старому лидеру три записи вместо одной и проверьте, что после починки исчезают все три.
2. В `tests/replication.rs` поднимите потерю сообщений до 50 %. Все четыре теста по-прежнему проходят. Почему Raft выдерживает потерю каждого второго сообщения? (Каждый пульс снова отправляет всё, чего не хватает последователю.)
3. Сломайте проверку согласованности: замените условие в `accept_entries` на `if prev_log_index > self.log.len() as u64 {` и запустите тесты. Посмотрите, какое зерно и какой узел падают, затем верните строку.
4. Добавьте `println!` в `handle_append_reply` на случай отказа последователя и запустите пример `replicate`, чтобы посмотреть, как лидер отступает.

## Упражнения

### Упражнение 1 — Зачем нужен `min(last_new)`? (легко)

В `accept_entries` последователь ставит `commit_index = leader_commit.min(last_new)`. Опишите в двух-трёх предложениях ситуацию, в которой просто `leader_commit` привёл бы к применению неверной команды.

<details>
<summary>Подсказка</summary>

Подумайте о последователе, у которого ещё остались записи старого лидера **после** записей, подтверждённых этим сообщением.

</details>

<details>
<summary>Ответ</summary>

В нашем коде каждое `AppendEntries` несёт все записи до конца журнала лидера, поэтому `last_new` никогда не меньше `leader_commit`. Настоящие реализации ограничивают число записей в сообщении (см. заметку ⚡ в разделе 5), и тогда это важно. У последователя устаревшие записи `[1 a] [1 b] [1 x]`. У лидера терма 2 журнал `[1 a] [1 b] [2 c]`, и индекс 3 у него зафиксирован. Из-за ограничения его сообщение несёт только одну запись: `prev_log_index: 1`, записи `[1 b]`, `leader_commit: 3`. С простым `leader_commit` последователь зафиксировал бы индекс 3 и применил бы свою устаревшую `x` вместо `c`. С `min(last_new)` он фиксирует только до индекса 2 — последней записи, совпадение которой с лидером доказало это сообщение.

</details>

### Упражнение 2 — Модульный тест починки журнала (средне)

Напишите `tests/log.rs` с одним тестом. Создайте последователя (`Node::new(1, vec![2, 3], …)`), задайте ему журнал `[(1, "a"), (1, "b"), (2, "stale")]` и доставьте два `AppendEntries` от узла 2 в терме 3, каждое с одной новой записью `(3, "c")`:

1. с `prev_log_index: 3, prev_log_term: 3`: последователь должен отказать (`success: false, match_index: 0`);
2. с `prev_log_index: 2, prev_log_term: 1`: он должен принять с `match_index: 3`, а его журнал должен стать `[(1, "a"), (1, "b"), (3, "c")]`.

<details>
<summary>Подсказка</summary>

`node.log` публичный, поэтому можно присвоить целый `Vec<Entry>`. Маленькие вспомогательные функции `entry(term, command)` и `append(node, prev_index, prev_term, entries) -> Message` делают тест читаемым.

</details>

<details>
<summary>Ответ</summary>

```rust
use p18_raft::{Config, Entry, Message, Node};

fn entry(term: u64, command: &str) -> Entry {
    Entry {
        term,
        command: command.to_string(),
    }
}

/// Deliver an AppendEntries from leader 2 in term 3 and return the reply.
fn append(
    node: &mut Node,
    prev_log_index: u64,
    prev_log_term: u64,
    entries: Vec<Entry>,
) -> Message {
    let msg = Message::AppendEntries {
        term: 3,
        prev_log_index,
        prev_log_term,
        entries,
        leader_commit: 0,
    };
    node.step(2, msg);
    node.take_outbox().remove(0).msg
}

#[test]
fn a_follower_repairs_a_conflicting_log() {
    let mut node = Node::new(1, vec![2, 3], Config::default(), 1);
    node.log = vec![entry(1, "a"), entry(1, "b"), entry(2, "stale")];

    // The leader's entry before the new one does not match: refuse.
    let reply = append(&mut node, 3, 3, vec![entry(3, "c")]);
    assert_eq!(
        reply,
        Message::AppendReply {
            term: 3,
            success: false,
            match_index: 0
        }
    );

    // One step back, index 2 matches: drop "stale", append "c".
    let reply = append(&mut node, 2, 1, vec![entry(3, "c")]);
    assert_eq!(
        reply,
        Message::AppendReply {
            term: 3,
            success: true,
            match_index: 3
        }
    );
    assert_eq!(node.log, vec![entry(1, "a"), entry(1, "b"), entry(3, "c")]);
}
```

```text
test a_follower_repairs_a_conflicting_log ... ok
```

Почему это работает: в первом сообщении у записи последователя с индексом 3 терм 2, а не 3: проверка не проходит. Во втором у индекса 2, как и ожидалось, терм 1; у новой записи с индексом 3 терм 3, а у записи последователя — 2: конфликт, поэтому `stale` обрезается, и добавляется `c`.

</details>

### Упражнение 3 — Ошибка, которую тест хаоса пропускает (сложно)

Удалите три строки `if self.term_at(n) != self.term { break; }` из `advance_commit` и запустите `cargo test`. Все тесты проходят. Объясните (а) почему код всё равно неверен, на примере ситуации из раздела 6, и (б) два изменения симулированного `Cluster`, которые дали бы тесту хаоса больше шансов дойти до этой ситуации.

<details>
<summary>Подсказка</summary>

В разделе 6 запись 2 терма 2 оказалась на большинстве, только когда узел 1 **снова** стал лидером, в более позднем терме, а затем упал. Чего наша сеть никогда не делает? Посмотрите, как `tick` доставляет сообщения.

</details>

<details>
<summary>Ответ</summary>

(а) Без этого правила лидер терма 4 считает копии записи из терма 2 и фиксирует её, как только она оказалась на большинстве. Узел, чья последняя запись имеет терм 3, всё ещё может выиграть следующие выборы (его журнал «актуальнее» последнего терма 2 у большинства) и перезаписать эту запись. Зафиксированная и, возможно, уже применённая команда заменяется: безопасность конечного автомата нарушена.

(б) Наша сеть доставляет каждое сообщение мгновенно и по порядку в пределах тика, а узлы никогда не падают и не перезапускаются. Два полезных изменения: **задерживать и переставлять сообщения** (держать конверты «в пути» случайное число тиков, чтобы лидер мог записать к себе, потерять лидерство и снова выиграть, пока старые сообщения ещё в дороге) и **ронять узлы**: выбрасывать исходящие лидера и приостанавливать его на несколько тиков сразу после того, как он добавил запись. Урок 18.12 делает и то и другое с `turmoil`, а средства проверки моделей вроде `stateright` систематически перебирают все порядки событий.

</details>

## Вопросы для самопроверки

1. Что сравнивает проверка согласованности в `AppendEntries` и почему достаточно одной записи?

<details>
<summary>Ответ</summary>

Она проверяет, что у последователя есть запись с индексом `prev_log_index` и термом `prev_log_term`. Одной записи достаточно, потому что та же проверка выполнялась для каждой более ранней записи при её добавлении, поэтому совпадение на одном индексе означает совпадение везде перед ним (совпадение журналов).

</details>

2. Последователь отказал в `AppendEntries`. Что делает лидер?

<details>
<summary>Ответ</summary>

Уменьшает `next_index` этого последователя на один (не ниже 1) и отправляет снова, с более ранним `prev_log_index`, пока журналы не совпадут. Затем отправляет всё начиная с этого места.

</details>

3. Почему последователю безопасно удалять записи, конфликтующие с записями лидера?

<details>
<summary>Ответ</summary>

Конфликтующая запись не может быть зафиксирована: правило выборов гарантирует, что у текущего лидера есть каждая зафиксированная запись, поэтому любая запись, отличающаяся от журнала лидера, никогда не была на большинстве так, чтобы это засчитывалось.

</details>

4. Когда лидер может продвинуть свой индекс фиксации до `n`?

<details>
<summary>Ответ</summary>

Когда запись на `n` — из **текущего терма** лидера и сохранена на большинстве (лидер плюс последователи с `match_index >= n`). Тогда всё перед `n` тоже зафиксировано.

</details>

5. Зачем новый лидер добавляет пустую запись?

<details>
<summary>Ответ</summary>

Потому что фиксировать он может, только считая записи своего терма. Пустая запись — такая запись; как только она на большинстве, все более старые записи перед ней тоже зафиксированы, без ожидания записи от клиента.

</details>

6. Что проверяет тест хаоса после каждого тика и что в конце?

<details>
<summary>Ответ</summary>

После каждого тика: для любых двух узлов один список `applied` — префикс другого (безопасность). В конце, после починки сети: у всех узлов одинаковый список `applied`, и новая запись применяется везде (живучесть).

</details>

## Итоги

- Лидер реплицирует через `AppendEntries`, которое несёт записи плюс `prev_log_index` и `prev_log_term`. Последователь добавляет записи, только если его журнал совпадает в этой точке: одна проверка гарантирует совпадение всего префикса.
- При отказе лидер сдвигает `next_index` назад и повторяет; последователь удаляет конфликтующие, никогда не зафиксированные записи и берёт записи лидера.
- `match_index` говорит лидеру, докуда дошёл каждый последователь. Запись зафиксирована, когда на большинстве оказалась запись **текущего терма**; более старые фиксируются вместе с ней.
- Новый лидер добавляет пустую запись, чтобы сразу зафиксировать более старые.
- Последователи фиксируют до `min(leader_commit, last_new)` и применяют зафиксированные записи по порядку, ровно один раз.
- Тест хаоса с зерном, разделениями сети и потерей сообщений проверяет свойство префикса после каждого тика и быстро находит сломанные правила, но не каждую редкую ошибку: Урок 18.12 делает симуляцию жёстче.

## Источники и что почитать

- [In Search of an Understandable Consensus Algorithm (статья о Raft)](https://raft.github.io/raft.pdf) — разделы 5.3 (репликация журнала) и 5.4 (безопасность, с рисунком 8); раздел 8 объясняет пустую запись.
- [The Raft Consensus Algorithm](https://raft.github.io/) — визуализация позволяет записывать значения и отрезать узлы, как в примере этого урока.
- [The Secret Lives of Data: Raft](https://thesecretlivesofdata.com/raft/) — вторая половина анимирует репликацию журнала и разделение сети.
- [Consensus: Bridging Theory and Practice (диссертация Ongaro)](https://web.stanford.edu/~ouster/cgi-bin/papers/OngaroPhD.pdf) — глава 3 — полное описание Raft, включая то, как последователи могут помочь лидеру быстрее найти совпадающий индекс.
- [MIT 6.5840 Distributed Systems](https://pdos.csail.mit.edu/6.824/) — университетский курс, лабораторные работы которого просят реализовать Raft и проверить его при сбоях, как в этом уроке.

---

[🏠 Начало](../START-HERE.md) · [← 18.10 Raft: выбор лидера](10-raft-leader-election.md) · [📋 Этап 18](README.md) · [18.12 Детерминированное тестирование симуляцией →](12-deterministic-simulation-testing.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
