<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 18.1 Что такое распределённая система](01-what-a-distributed-system-is.md) · [📋 Этап 18](README.md) · [18.3 Повторные попытки, задержки и идемпотентность →](03-retries-backoff-and-idempotency.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/18-distributed-systems/02-failures-and-timeouts.md) <!-- nav -->

# Урок 18.2 — Сбои и тайм-ауты

> **Этап 18 — Распределённые системы** · Урок 2 из 15 · ⏱ около 135 минут

## Что вы узнаете

- Типичные способы, которыми отказывают узлы и сети: от чистого падения до узла, который всего лишь медленный.
- Почему тайм-аут никогда не говорит, что произошло на другой стороне, — на примере маленького «узла»-банка.
- Как мгновенно тестировать тайм-ауты с остановленными часами Tokio.
- Как выбрать тайм-аут по измеренным задержкам и как один дедлайн защищает цепочку вызовов.
- Как работает детектор сбоев на heartbeat-сообщениях и почему он может только *подозревать* узел, но никогда не знать наверняка.

## Перед началом

- Вы прошли [Урок 18.1 — Что такое распределённая система](01-what-a-distributed-system-is.md).
- Вы знаете задачи Tokio, каналы `mpsc` и `oneshot` и паттерн «актор» из [Урока 14.13](../14-concurrency-and-async/13-tokio-tasks-and-channels.md), а также `timeout`, дедлайны и остановленные часы из [Урока 14.14](../14-concurrency-and-async/14-cancellation-timeouts-and-graceful-shutdown.md).
- Создайте пакет для этого урока:

```bash
cd ~/projects/rust
cargo new --lib code/p18-failures
cd code/p18-failures
cargo add tokio@1 --features full
cargo add --dev tokio@1 --features test-util
mkdir src/bin
```

В этом уроке «узлы» — это задачи Tokio внутри одной программы, а «сеть» — канал. Так примеры остаются маленькими,
а тесты быстрыми, и идеи при этом точно те же, что и с настоящими машинами.

## Новые слова

| Слово | Значение |
|---|---|
| **модель сбоев** (failure model) | Описание того, какими способами части системы разрешено отказывать. Алгоритмы проектируют под одну модель сбоев. |
| **падение с остановкой** (crash-stop) | Узел останавливается и больше никогда не возвращается. |
| **падение с восстановлением** (crash-recovery) | Узел останавливается и позже перезапускается, и у него есть только то, что он сохранил на диск (Урок 17.2). |
| **пропуск** (omission) | Сообщение потеряно: оно отправлено, но так и не доставлено. |
| **серый сбой** (gray failure) | Узел не мёртв, но работает плохо: очень медленно или отказывает только на части запросов. Его трудно обнаружить. |
| **византийский сбой** (Byzantine failure) | Узел отправляет неверные или даже злонамеренные сообщения. Большинство систем в этом курсе предполагают, что узлы не лгут. |
| **тайм-аут** (timeout) | Предел того, сколько мы ждём ответа. Когда он истекает, мы перестаём ждать; другую сторону он не останавливает. |
| **дедлайн** (deadline) | Один фиксированный момент, к которому должна завершиться вся операция; общий для всех её шагов. |
| **heartbeat** (сердцебиение) | Маленькое сообщение «я жив», которое узел регулярно отправляет. |
| **детектор сбоев** (failure detector) | Компонент, который следит за heartbeat-сообщениями и *подозревает* замолчавшие узлы. |
| **OOM** (Out-Of-Memory, нехватка памяти) killer | Часть ядра Linux, которая убивает процесс, когда на машине кончается память. |
| **UDP** (User Datagram Protocol, протокол пользовательских датаграмм) | Протокол, который отправляет отдельные сообщения без соединений и без гарантий доставки (Урок 15.3). |
| **CI** (Continuous Integration, непрерывная интеграция) | Сервер, который запускает сборку и тесты при каждом изменении (Урок 09.12). |
| **перцентиль** (percentile) | Значение, ниже которого лежит заданная доля измерений. 99-й перцентиль (**p99**) задержек — время, которое не превысили 99 % запросов. Подробно — в Уроке 19.1. |

## 1. Как отказывают узлы

Инженеры описывают сбои несколькими стандартными названиями. Когда вы читаете о системе («она переживает падение
одного узла из трёх»), название точно говорит, что обещано.

| Сбой | Что происходит | Пример |
|---|---|---|
| **Падение с остановкой** | Узел останавливается навсегда. | Машину со сгоревшим диском выбрасывают. |
| **Падение с восстановлением** | Узел останавливается и перезапускается. Память теряется; диск сохраняется, но только данные, для которых был сделан `fsync`. | Процесс убит OOM killer (Out-Of-Memory, нехватка памяти) и перезапущен systemd (Урок 11.6). |
| **Пропуск** | Часть сообщений теряется. | Пакет UDP (User Datagram Protocol, протокол пользовательских датаграмм) отброшен загруженным маршрутизатором; соединение TCP (Transmission Control Protocol, протокол управления передачей) сброшено. |
| **Сетевое разделение** | Группы узлов не могут достучаться друг до друга, но все работают. | Сломался коммутатор между двумя серверными стойками. |
| **Серый сбой** | Узел жив, но медленен или частично сломан. | Диск, которому нужно 2 с на запись; узел, приостановленный через Ctrl+Z, как в Уроке 18.1. |
| **Византийский** | Узел отправляет неверные или злонамеренные сообщения. | Взломанная машина или перевёрнутый бит в памяти. За рамками этого этапа. |

Большинство настоящих систем, и Raft позже на этом этапе, предполагают **узлы с падением и восстановлением в сети,
которая может терять, задерживать, дублировать и переставлять сообщения**. Эту модель мы и используем дальше.

> 💡 **Совет:** медленный узел часто хуже мёртвого. Мёртвый узел замечают и заменяют. Медленный продолжает
> принимать работу, замедляет каждый запрос, который его касается, и проходит любую простую проверку
> работоспособности.

## 2. Тайм-аут не видит другую сторону

Урок 18.1 показал, что узел может сделать работу после того, как клиент сдался. Уточним это на крошечном банке.
Банк — задача-актор (Урок 14.13). Он сразу проводит каждый вклад, но его ответу нужно время, чтобы вернуться,
как ответу в медленной сети. Поместите это в `src/lib.rs`:

```rust
//! Failures and timeouts (Lesson 18.2).

use std::time::Duration;

use tokio::sync::{mpsc, oneshot};

/// A request to the bank "node".
pub enum Command {
    Deposit {
        amount: u64,
        reply: oneshot::Sender<u64>,
    },
    Balance {
        reply: oneshot::Sender<u64>,
    },
}

/// Why a call to the bank did not return a result.
#[derive(Debug, PartialEq)]
pub enum CallError {
    /// No answer within the limit: the deposit may or may not have happened.
    Timeout,
    /// The bank is not running any more.
    Gone,
}

/// Starts the bank. It does every deposit at once, but each reply needs `reply_delay`
/// to travel back, like an answer on a slow network.
pub fn spawn_bank(reply_delay: Duration) -> mpsc::Sender<Command> {
    let (tx, mut rx) = mpsc::channel(16);
    tokio::spawn(async move {
        let mut balance = 0;
        while let Some(command) = rx.recv().await {
            match command {
                Command::Deposit { amount, reply } => {
                    balance += amount; // the work is done here...
                    tokio::spawn(async move {
                        tokio::time::sleep(reply_delay).await; // ...the answer is slow
                        let _ = reply.send(balance);
                    });
                }
                Command::Balance { reply } => {
                    let _ = reply.send(balance);
                }
            }
        }
    });
    tx
}

/// Asks the bank to deposit `amount` and waits at most `limit` for the new balance.
pub async fn deposit(
    bank: &mpsc::Sender<Command>,
    amount: u64,
    limit: Duration,
) -> Result<u64, CallError> {
    let (reply, answer) = oneshot::channel();
    let command = Command::Deposit { amount, reply };
    bank.send(command).await.map_err(|_| CallError::Gone)?;
    match tokio::time::timeout(limit, answer).await {
        Ok(Ok(balance)) => Ok(balance),
        Ok(Err(_)) => Err(CallError::Gone),
        Err(_) => Err(CallError::Timeout),
    }
}

/// Asks the bank for the current balance.
pub async fn balance(bank: &mpsc::Sender<Command>) -> Result<u64, CallError> {
    let (reply, answer) = oneshot::channel();
    bank.send(Command::Balance { reply })
        .await
        .map_err(|_| CallError::Gone)?;
    answer.await.map_err(|_| CallError::Gone)
}
```

**Что происходит:**

1. `spawn_bank` запускает задачу, которая владеет `balance` и обрабатывает по одной команде `Command`. Никто
   другой не может тронуть баланс: как до настоящего удалённого узла, до него можно добраться только сообщениями.
2. При вкладе банк **сначала** прибавляет сумму. Затем он запускает маленькую задачу, которая ждёт `reply_delay`
   и только потом отправляет ответ. Так мы имитируем ответ, который медленно идёт по сети.
3. `deposit` — это клиент. Он отправляет команду и ждёт ответа через `tokio::time::timeout`. У результата два
   слоя: внешний `Err` означает «время вышло»; внутренний `Err` означает, что банк уничтожил отправитель ответа,
   например потому что остановился.
4. `CallError::Timeout` задокументирован честно: «могло произойти, а могло и нет». Лучшего комментария написать
   невозможно.

Теперь эксперимент, `src/bin/timeout_lies.rs`:

```rust
use std::time::Duration;

use p18_failures::{balance, deposit, spawn_bank};

#[tokio::main]
async fn main() {
    // The bank does the work at once, but its answer needs 300 ms to come back.
    let bank = spawn_bank(Duration::from_millis(300));

    match deposit(&bank, 100, Duration::from_millis(100)).await {
        Ok(new_balance) => println!("deposit done, balance {new_balance}"),
        Err(error) => println!("deposit failed: {error:?}"),
    }
    println!(
        "balance after the 'failed' deposit: {:?}",
        balance(&bank).await
    );

    match deposit(&bank, 100, Duration::from_millis(500)).await {
        Ok(new_balance) => println!("deposit done, balance {new_balance}"),
        Err(error) => println!("deposit failed: {error:?}"),
    }
}
```

`cargo run --bin timeout_lies`:

```text
deposit failed: Timeout
balance after the 'failed' deposit: Ok(100)
deposit done, balance 200
```

**Что происходит:**

1. Первый вклад ждёт только 100 мс, а ответу нужно 300 мс. Клиент сообщает `Timeout`.
2. Но баланс равен 100. «Неудавшийся» вклад произошёл. Если бы клиент показал пользователю «Платёж не прошёл,
   попробуйте ещё раз», и пользователь нажал бы снова, деньги были бы учтены дважды.
3. Второй вклад ждёт 500 мс — этого достаточно — и видит баланс 200.

У любого удалённого вызова три возможных исхода, а не два:

```text
            ┌── Ok(answer)         → it happened, here is the result
 a call ────┼── definite error     → it surely did NOT happen (e.g. Connection refused)
            └── timeout / unknown  → it may have happened, may happen later, or never
```

Каждый проект в следующих уроках должен обрабатывать третью ветку. Урок 18.3 показывает стандартный ответ:
сделать операции безопасными для повторения, а затем повторять их.

## 3. Тестируем без ожидания

Тест с настоящими паузами по 300 мс медленный, а на загруженной машине он может случайно упасть: это
**нестабильный** (flaky) тест. Остановленные часы Tokio (Урок 14.14) решают обе проблемы. Добавьте модуль тестов
в конец `src/lib.rs`:

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[tokio::test(start_paused = true)]
    async fn a_timed_out_deposit_can_still_happen() {
        let bank = spawn_bank(Duration::from_secs(10));
        let result = deposit(&bank, 100, Duration::from_secs(1)).await;
        assert_eq!(result, Err(CallError::Timeout));
        // The client gave up, but the money is there.
        assert_eq!(balance(&bank).await, Ok(100));
    }
}
```

`cargo test --lib`:

```text
test tests::a_timed_out_deposit_can_still_happen ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

Ответ через 10 секунд и тайм-аут в 1 секунду проверены за 0,00 секунды. Когда часы остановлены, Tokio сразу
перескакивает к следующему таймеру, как только все задачи ждут, поэтому порядок событий всегда одинаков.

> 🧪 **Тестирование:** напишите случай «тайм-аут, но всё произошло» в виде теста для каждой удалённой операции,
> которая меняет данные. Тест документирует опасность, а Урок 18.3 превратит его в тест, доказывающий, что
> повтор безопасен.

## 4. Сколько ждать?

Идеального тайм-аута нет. У каждого выбора своя цена:

| Тайм-аут | Что идёт не так |
|---|---|
| Слишком короткий | Здоровые, но чуть медленные узлы считаются отказавшими. Клиенты повторяют запросы, это добавляет нагрузку, от которой узлы становятся ещё медленнее: порочный круг. |
| Слишком длинный | Пользователи ждут мёртвый узел. Потоки, соединения и память остаются занятыми, и клиент тем временем не может сделать ничего полезного. |

Практическое правило — **измерить** задержки настоящего сервиса и выбрать тайм-аут заметно выше медленного края,
например в несколько раз больше 99-го перцентиля. Если 99 % вызовов укладываются в 40 мс, тайм-аут от 100 до
200 мс редко обрывает здоровый вызов, но всё ещё быстро замечает зависший. Это строят Упражнения 1 и 3.

Когда один запрос делает несколько вызовов, дайте всей цепочке **один дедлайн**, и пусть каждый шаг использует
только оставшееся время (Урок 14.14). Если бы у каждого шага был свой фиксированный тайм-аут, цепочка из пяти
шагов могла бы длиться в пять раз дольше, чем пользователь готов ждать. Упражнение 2 строит такую цепочку и
тестирует её.

> 🔒 **Безопасность:** тайм-ауты защищают и от атак. Клиент, который подключился и отправляет по одному байту в
> минуту, может вечно держать соединение, буфер и, возможно, поток. Тайм-ауты чтения и записи на соединениях нужны
> каждому серверу, а не только каждому клиенту.

## 5. Детекторы сбоев: heartbeat

Чтобы не узнавать о мёртвом узле только по тайм-ауту запроса, узлы часто отправляют **heartbeat-сообщения**:
маленькие «я жив», например каждые 100 мс. **Детектор сбоев** помнит, когда последний раз слышал каждый узел, и
**подозревает** каждый узел, который молчит дольше предела.

Добавьте эти импорты в начало `src/lib.rs`, рядом с остальными:

```rust
use std::collections::HashMap;
use std::time::Instant;
```

Затем добавьте детектор (перед модулем тестов):

```rust
/// Suspects a node when no heartbeat came from it for longer than `timeout`.
pub struct FailureDetector {
    timeout: Duration,
    last_seen: HashMap<String, Instant>,
}

impl FailureDetector {
    pub fn new(timeout: Duration) -> Self {
        FailureDetector {
            timeout,
            last_seen: HashMap::new(),
        }
    }

    /// Records that `node` said "I am alive" at the moment `now`.
    pub fn heartbeat(&mut self, node: &str, now: Instant) {
        self.last_seen.insert(node.to_string(), now);
    }

    /// The nodes that have been silent for too long at the moment `now`, sorted by name.
    pub fn suspects(&self, now: Instant) -> Vec<String> {
        let mut silent: Vec<String> = self
            .last_seen
            .iter()
            .filter(|(_, seen)| now.duration_since(**seen) > self.timeout)
            .map(|(node, _)| node.clone())
            .collect();
        silent.sort();
        silent
    }
}
```

Детектор никогда сам не читает часы: вызывающий код передаёт `now`. Это маленькое решение делает его **чистым**
куском логики, которым тест может управлять с любыми моментами времени. Добавьте этот тест внутрь `mod tests`:

```rust
    #[test]
    fn a_silent_node_becomes_a_suspect() {
        let start = Instant::now();
        let ms = Duration::from_millis;
        let mut detector = FailureDetector::new(ms(250));
        detector.heartbeat("a", start);
        detector.heartbeat("b", start);
        detector.heartbeat("a", start + ms(200));

        assert!(detector.suspects(start + ms(250)).is_empty());
        assert_eq!(detector.suspects(start + ms(300)), vec!["b"]);
        assert_eq!(detector.suspects(start + ms(500)), vec!["a", "b"]);
    }
```

`let ms = Duration::from_millis;` сохраняет саму функцию в переменной, так что `ms(250)` — короткая запись
`Duration::from_millis(250)`. В 250 мс никто не подозревается (правило — «дольше чем», а `b` молчал ровно 250 мс).
В 300 мс подозревается `b`; в 500 мс — ещё и `a`.

Теперь живая демонстрация с тремя узлами. Узел `a` здоров, узел `b` **замирает** на 400 мс и затем продолжает,
узел `c` **падает** через 200 мс. Создайте `src/bin/heartbeats.rs`:

```rust
use std::time::Duration;

use p18_failures::FailureDetector;
use tokio::sync::mpsc;
use tokio::time::{Instant, sleep_until};

/// A node that sends a heartbeat at each of the given moments (in ms after `start`).
async fn node(name: &'static str, beats: Vec<u64>, tx: mpsc::Sender<&'static str>, start: Instant) {
    for ms in beats {
        sleep_until(start + Duration::from_millis(ms)).await;
        let _ = tx.send(name).await;
    }
}

#[tokio::main]
async fn main() {
    let start = Instant::now();
    let (tx, mut rx) = mpsc::channel(64);
    let schedules = [
        ("a", (0..10).map(|i| i * 100).collect()), // always healthy
        ("b", vec![0, 100, 200, 300, 700, 800, 900]), // frozen from 300 to 700 ms
        ("c", vec![0, 100, 200]),                  // crashes after 200 ms
    ];
    for (name, beats) in schedules {
        tokio::spawn(node(name, beats, tx.clone(), start));
    }

    let mut detector = FailureDetector::new(Duration::from_millis(220));
    let mut suspected: Vec<String> = Vec::new();
    // The detector looks at the heartbeats every 100 ms, at 50, 150, 250, ... ms.
    let first_check = start + Duration::from_millis(50);
    let mut check = tokio::time::interval_at(first_check, Duration::from_millis(100));
    while start.elapsed() < Duration::from_millis(1000) {
        tokio::select! {
            Some(name) = rx.recv() => detector.heartbeat(name, Instant::now().into_std()),
            _ = check.tick() => {
                let now_suspected = detector.suspects(Instant::now().into_std());
                if now_suspected != suspected {
                    let t = start.elapsed().as_millis() / 10 * 10;
                    println!("t={t}ms: suspects {now_suspected:?}");
                    suspected = now_suspected;
                }
            }
        }
    }
}
```

`cargo run --bin heartbeats`:

```text
t=450ms: suspects ["c"]
t=550ms: suspects ["b", "c"]
t=750ms: suspects ["c"]
```

**Что происходит:**

1. Задача каждого узла отправляет своё имя в канал в моменты из своего расписания. `sleep_until` ждёт до момента,
   а не в течение длительности, поэтому маленькие задержки не накапливаются.
2. Главный цикл использует `select!` (Урок 14.14), чтобы записывать каждый пришедший heartbeat и искать
   подозреваемых на каждом тике. `into_std()` превращает `Instant` из Tokio в `Instant` из `std`, который
   использует детектор. Время округляется вниз до 10 мс, чтобы крошечные задержки планировщика не меняли
   напечатанные числа.
3. Узел `c` последний раз говорил в 200 мс. На проверке в 450 мс он молчал 250 мс, больше 220 мс: подозревается.
   Он больше не возвращается.
4. Узел `b` последний раз говорил в 300 мс и подозревается в 550 мс. В 700 мс он снова говорит, и на следующей
   проверке он уже не подозреваемый. Детектор **ошибся** насчёт `b`: тот только замер.

Это фундаментальный предел обнаружения сбоев. Снаружи упавший узел и очень медленный узел выглядят **одинаково**,
пока медленный снова не заговорит. Детектор с коротким тайм-аутом быстро замечает падения, но чаще подозревает
напрасно; длинный тайм-аут ошибается реже, но замечает падения позже. Настоящие системы выбирают баланс, а
алгоритмы поверх них (как Raft в Уроке 18.10) устроены так, чтобы оставаться правильными, даже когда детектор
ошибается.

## Частые ошибки

> ⚠️ **Частая ошибка: считать тайм-аут признаком «не произошло».** Как показал раздел 2, операция с тайм-аутом
> может быть выполнена. Никогда не повторяйте неповторяемую операцию только потому, что истёк тайм-аут. Урок 18.3
> показывает, как сделать повторение безопасным.

> ⚠️ **Частая ошибка: настоящие паузы в тестах.** Тест, который спит 300 мс и ждёт срабатывания тайм-аута в
> 100 мс, иногда будет падать на загруженном сервере CI (Continuous Integration, непрерывная интеграция).
> Используйте `#[tokio::test(start_paused = true)]` и, где можно, держите логику чистой (передавайте `now`), как
> это делает детектор сбоев.

## Попробуйте сами

1. В `timeout_lies.rs` измените первый предел со 100 мс на 250 мс, а затем на 350 мс. Найдите границу, на которой
   вклад меняется с `Timeout` на успех.
2. В `heartbeats.rs` измените тайм-аут детектора с 220 мс на 450 мс. Запустите: теперь `c` подозревается гораздо
   позже, а `b` не подозревается вовсе. Вы обменяли скорость обнаружения на меньшее число ошибок.

## Упражнения

### Упражнение 1 — 99-й перцентиль, с тестами (легко)

Создайте `src/latency.rs` (и добавьте `pub mod latency;` в `src/lib.rs`) с функцией
`pub fn percentile(samples: &[Duration], p: f64) -> Duration` по методу **ближайшего ранга** (nearest-rank):
отсортируйте измерения и возьмите ранг `ceil(p / 100 × n)` (считая с 1). Напишите тесты: для измерений 1 мс … 100 мс
в любом порядке p50 равен 50 мс, p99 — 99 мс, p100 — 100 мс; для девяти измерений по 10 мс и одного в 400 мс p50
равен 10 мс, а p99 — 400 мс.

<details>
<summary>Подсказка</summary>

`let mut sorted = samples.to_vec(); sorted.sort();`, а затем `sorted[rank - 1]`, потому что в Rust индексы
начинаются с 0. Преобразуйте через `as f64` и обратно через `as usize` после `.ceil()`.

</details>

<details>
<summary>Ответ</summary>

```rust
//! Exercises 1 and 3: percentiles and an adaptive timeout.

use std::time::Duration;

/// The `p`-th percentile (0 < p <= 100) of `samples`, by the nearest-rank method.
pub fn percentile(samples: &[Duration], p: f64) -> Duration {
    assert!(!samples.is_empty(), "no samples");
    assert!(p > 0.0 && p <= 100.0, "p must be in (0, 100]");
    let mut sorted = samples.to_vec();
    sorted.sort();
    let rank = (p / 100.0 * sorted.len() as f64).ceil() as usize;
    sorted[rank - 1]
}

#[cfg(test)]
mod tests {
    use super::*;

    fn ms(n: u64) -> Duration {
        Duration::from_millis(n)
    }

    #[test]
    fn percentiles_of_one_to_a_hundred() {
        let samples: Vec<Duration> = (1..=100).rev().map(ms).collect();
        assert_eq!(percentile(&samples, 50.0), ms(50));
        assert_eq!(percentile(&samples, 99.0), ms(99));
        assert_eq!(percentile(&samples, 100.0), ms(100));
    }

    #[test]
    fn one_slow_sample_out_of_ten_shows_in_p99() {
        let mut samples = vec![ms(10); 9];
        samples.push(ms(400));
        assert_eq!(percentile(&samples, 50.0), ms(10));
        assert_eq!(percentile(&samples, 99.0), ms(400));
    }
}
```

Почему это работает: второй тест показывает, почему инженеры смотрят на p99, а не на среднее или медиану: один
медленный вызов из десяти не виден в p50, но определяет тайм-аут.

</details>

### Упражнение 2 — Один дедлайн на два вызова (средне)

Напишите интеграционный тест `tests/deadline.rs` со вспомогательной функцией
`async fn deposit_twice(bank, amount, total: Duration) -> Vec<Result<u64, CallError>>`, которая делает два вклада
и даёт **обоим вместе** не больше `total`. Проверьте её с остановленными часами: при задержке ответа 600 мс и общем
пределе 1 с первый вклад проходит (баланс 10), второй завершается тайм-аутом — а баланс после этого равен 20.

<details>
<summary>Подсказка</summary>

Вычислите `let deadline = Instant::now() + total;` один раз (используйте `tokio::time::Instant`, который следует за
остановленными часами). Перед каждым вызовом `deadline.saturating_duration_since(Instant::now())` — оставшееся
время; передайте его как `limit`.

</details>

<details>
<summary>Ответ</summary>

```rust
use std::time::Duration;

use p18_failures::{CallError, Command, balance, deposit, spawn_bank};
use tokio::sync::mpsc;
use tokio::time::Instant;

/// Deposits `amount` twice, with one deadline for both calls together.
async fn deposit_twice(
    bank: &mpsc::Sender<Command>,
    amount: u64,
    total: Duration,
) -> Vec<Result<u64, CallError>> {
    let deadline = Instant::now() + total;
    let mut results = Vec::new();
    for _ in 0..2 {
        // Each call gets only the time that is left before the deadline.
        let left = deadline.saturating_duration_since(Instant::now());
        results.push(deposit(bank, amount, left).await);
    }
    results
}

#[tokio::test(start_paused = true)]
async fn the_second_call_runs_out_of_time() {
    let bank = spawn_bank(Duration::from_millis(600));
    let results = deposit_twice(&bank, 10, Duration::from_secs(1)).await;
    assert_eq!(results, vec![Ok(10), Err(CallError::Timeout)]);
    // The second deposit timed out, but it was still applied.
    assert_eq!(balance(&bank).await, Ok(20));
}
```

`cargo test --test deadline`:

```text
test the_second_call_runs_out_of_time ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

Почему это работает: первый вызов тратит 600 мс, поэтому второму, которому нужно 600 мс, остаётся только 400 мс.
С отдельным тайм-аутом в 1 секунду на каждый вызов оба прошли бы, но цепочка заняла бы 1,2 с.
`saturating_duration_since` возвращает ноль, а не паникует, если дедлайн уже прошёл.

</details>

### Упражнение 3 — Тайм-аут, который учится (сложно)

Добавьте в `src/latency.rs` функцию `pub fn adaptive_timeout(recent: &[Duration], minimum: Duration) -> Duration`:
удвоенный p99 недавних задержек, но не меньше `minimum`, и ровно `minimum`, если измерений ещё нет. Напишите тест
с тремя случаями: нет измерений; измерения 10 и 12 мс (результат — минимум 50 мс); измерения 10 и 100 мс
(результат — 200 мс). Затем подумайте: откуда в настоящем клиенте брались бы измерения `recent`?

<details>
<summary>Подсказка</summary>

Используйте `percentile` из Упражнения 1. `Duration` можно умножать на целое число (`d * 2`) и сравнивать через
`.max(other)`.

</details>

<details>
<summary>Ответ</summary>

```rust
/// Twice the p99 of the recent latencies, but never below `minimum`.
pub fn adaptive_timeout(recent: &[Duration], minimum: Duration) -> Duration {
    if recent.is_empty() {
        return minimum;
    }
    (percentile(recent, 99.0) * 2).max(minimum)
}
```

И внутри `mod tests`:

```rust
    #[test]
    fn the_timeout_follows_the_latencies() {
        let minimum = ms(50);
        assert_eq!(adaptive_timeout(&[], minimum), ms(50));
        assert_eq!(adaptive_timeout(&[ms(10), ms(12)], minimum), ms(50));
        assert_eq!(adaptive_timeout(&[ms(10), ms(100)], minimum), ms(200));
    }
```

`cargo test --lib`:

```text
test result: ok. 5 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

Почему это работает: тайм-аут растёт, когда сервис замедляется, и уменьшается, когда он приходит в себя; минимум
защищает от настолько короткого тайм-аута, что его вызывал бы обычный разброс задержек (джиттер). В настоящем
клиенте `recent` был бы окном из последних нескольких сотен измеренных длительностей вызовов, например в
`VecDeque` (двусторонней очереди, [Урок 07.5](../07-algorithms-and-data-structures/05-stacks-and-queues.md)),
которая выбрасывает самое старое измерение, когда заполнена.

</details>

## Вопросы для самопроверки

1. Чем падение с остановкой отличается от падения с восстановлением?

<details>
<summary>Ответ</summary>

Узел с падением и остановкой никогда не возвращается. Узел с падением и восстановлением позже перезапускается; он
теряет память, но сохраняет то, что надёжно записал на диск.

</details>

2. Вклад клиента завершился тайм-аутом. Назовите три группы возможных исходов любого удалённого вызова.

<details>
<summary>Ответ</summary>

Успех с ответом; определённая ошибка (точно не произошло, например соединение отклонено); и неизвестный исход
(тайм-аут): могло произойти, может произойти позже или не произойти никогда.

</details>

3. Почему тест с задержкой в 10 секунд закончился за 0,00 секунды?

<details>
<summary>Ответ</summary>

`start_paused = true` останавливает часы Tokio. Когда все задачи ждут таймера, Tokio сразу переводит часы к
следующему таймеру, поэтому настоящее время не проходит.

</details>

4. Запрос вызывает три сервиса подряд, у каждого тайм-аут 1 секунда. Каково худшее время ожидания и как помогает
   дедлайн?

<details>
<summary>Ответ</summary>

До 3 секунд (плюс обработка). С одним дедлайном, скажем, в 1 секунду на весь запрос каждый вызов получает только
оставшееся время, поэтому весь запрос никогда не длится дольше 1 секунды.

</details>

5. Детектор сбоев заподозрил узел `b`, но `b` был жив. Детектор сломан?

<details>
<summary>Ответ</summary>

Нет. Детектор сбоев может только подозревать: снаружи молчащий и медленный узлы выглядят одинаково. Каждый
детектор обменивает быстрое обнаружение на ложные подозрения, а алгоритмы поверх него должны оставаться
правильными, когда он ошибается.

</details>

## Итоги

- Называйте сбои точно: падение с остановкой, падение с восстановлением, пропуск, разделение, серый сбой и
  византийский сбой. Этот этап предполагает узлы с падением и восстановлением в сети, которая теряет, задерживает и
  переставляет сообщения.
- У удалённого вызова три исхода: успех, определённая ошибка и **неизвестность**. Тайм-аут всегда означает
  неизвестность.
- Операция с тайм-аутом всё ещё может завершиться; обработка этого случая — работа следующего урока.
- Тестируйте код, зависящий от времени, с остановленными часами Tokio и держите такую логику чистой, передавая `now`.
- Выбирайте тайм-ауты по измеренным задержкам (в несколько раз больше p99) и давайте цепочке вызовов один дедлайн.
- Heartbeat-сообщения и детектор сбоев находят замолчавшие узлы, но детектор может только подозревать: медленный и
  мёртвый выглядят одинаково.

## Источники и что почитать

- [Timeouts, retries, and backoff with jitter](https://aws.amazon.com/builders-library/timeouts-retries-and-backoff-with-jitter/) — Amazon Builders' Library о том, как в настоящих сервисах выбирают тайм-ауты по перцентилям задержки.
- [Designing Data-Intensive Applications](https://dataintensive.net/) — глава «The Trouble with Distributed Systems» разбирает ненадёжные сети, тайм-ауты и обнаружение сбоев.
- [`tokio::time::timeout`](https://docs.rs/tokio/latest/tokio/time/fn.timeout.html) — API (Application Programming Interface, программный интерфейс), использованный в этом уроке.
- [`tokio::time::pause`](https://docs.rs/tokio/latest/tokio/time/fn.pause.html) — как работают остановленные часы в тестах.
- [Implementing health checks](https://aws.amazon.com/builders-library/implementing-health-checks/) — Amazon Builders' Library о heartbeat-сообщениях, проверках работоспособности и серых сбоях.

---

[🏠 Начало](../START-HERE.md) · [← 18.1 Что такое распределённая система](01-what-a-distributed-system-is.md) · [📋 Этап 18](README.md) · [18.3 Повторные попытки, задержки и идемпотентность →](03-retries-backoff-and-idempotency.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
