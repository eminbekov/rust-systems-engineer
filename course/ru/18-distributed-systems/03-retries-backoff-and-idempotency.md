<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 18.2 Сбои и тайм-ауты](02-failures-and-timeouts.md) · [📋 Этап 18](README.md) · [18.4 Время и часы →](04-time-and-clocks.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/18-distributed-systems/03-retries-backoff-and-idempotency.md) <!-- nav -->

# Урок 18.3 — Повторные попытки, задержки и идемпотентность

> **Этап 18 — Распределённые системы** · Урок 3 из 15 · ⏱ около 3 часов (2 занятия)

## Что вы узнаете

- Какие сбои стоит повторять, а какие нет.
- Как экспоненциальная задержка и джиттер не дают тысячам повторяющих клиентов снова уронить восстанавливающийся сервер.
- Как написать небольшую обобщённую функцию `retry` для асинхронного кода и протестировать её с остановленными часами.
- Что такое идемпотентность и почему «повторить после тайм-аута» безопасно только для идемпотентных операций.
- Как ключи идемпотентности делают неидемпотентную операцию, например вклад, безопасной для повтора.

## Перед началом

- Вы прошли [Урок 18.2 — Сбои и тайм-ауты](02-failures-and-timeouts.md). Этот урок начинается с его главного вывода: после тайм-аута операция могла произойти, а могла и нет.
- Вы знаете замыкания и трейты `Fn`/`FnMut` с Этапа 05, обобщённые функции с `where` из [Урока 05.1](../05-rust-intermediate/01-generics.md) и тесты на основе свойств из [Урока 09.4](../09-testing-debugging-benchmarking/04-property-based-testing.md) (для Упражнения 3).
- Создайте пакет для этого урока:

```bash
cd ~/projects/rust
cargo new --lib code/p18-retries
cd code/p18-retries
cargo add tokio@1 --features full
cargo add rand@0.10
cargo add --dev tokio@1 --features test-util
cargo add --dev proptest@1
mkdir src/bin tests
```

`rand` — стандартный крейт для случайных чисел. Он нужен нам для джиттера.

## Новые слова

| Слово | Значение |
|---|---|
| **повторная попытка** (retry) | Повторная отправка того же запроса после того, как он завершился ошибкой или тайм-аутом. |
| **временный сбой** (transient failure) | Сбой, который через какое-то время проходит сам: загруженный сервер, потерянный пакет. Его стоит повторить. |
| **постоянный сбой** (permanent failure) | Сбой, который будет повторяться каждый раз: неверный пароль, отсутствующий файл. Повторять бессмысленно. |
| **задержка между попытками** (backoff) | Ожидание между повторами. *Экспоненциальная* задержка удваивает ожидание после каждого сбоя, до *потолка* (cap, максимума). |
| **джиттер** (jitter) | Случайность, добавленная к ожиданию, чтобы много клиентов не повторяли запрос в один и тот же момент. |
| **громоподобное стадо** (thundering herd) | Множество клиентов, которые действуют ровно в один момент, например все повторяют запрос вместе после аварии, и снова перегружают сервер. |
| **идемпотентный** (idempotent) | Операция идемпотентна, если выполнить её дважды — то же самое, что выполнить один раз. «Установить цену 5» — идемпотентна; «прибавить 5 к цене» — нет. |
| **ключ идемпотентности** (idempotency key) | Уникальный идентификатор, который клиент отправляет с запросом и с каждым его повтором, чтобы сервер распознал повтор и не сделал работу дважды. |
| **доставка «хотя бы один раз»** (at-least-once delivery) | Сообщение доставляется один или несколько раз, но никогда ноль раз (Урок 16.11). Повторы дают именно такую доставку. |
| **RFC** (Request for Comments, «запрос комментариев») | Документ, который определяет интернет-стандарт, например RFC 9110 для HTTP. |
| **UUID** (Universally Unique Identifier, универсальный уникальный идентификатор) | Случайный 128-битный идентификатор. Два UUID, созданные где угодно, на практике никогда не совпадают. |
| **HTTP** (Hypertext Transfer Protocol, протокол передачи гипертекста) | Протокол веба (Урок 15.8). Его ответы несут трёхзначный *код состояния*, например 200 (OK) или 503 (Service Unavailable). |

## 1. Когда повтор — хорошая идея?

Повтор помогает, только если у следующей попытки есть реальный шанс на успех. Сначала разделите сбои на две группы:

| Сбой | Вид | Повторять? |
|---|---|---|
| Тайм-аут, сброс соединения, отказ в соединении, пока узел перезапускается | временный | да, с задержкой |
| «Сервер занят», «слишком много запросов» (HTTP 429, 503) | временный | да, и ждать дольше |
| «Неверный запрос», «не найдено», «доступ запрещён» (HTTP 400, 404, 403) | постоянный | нет: тот же запрос снова не пройдёт |
| Ошибка в клиенте (неверный формат данных) | постоянный | нет: исправьте код |

Затем задайте второй вопрос, которому посвящён этот урок: **безопасно ли выполнить операцию дважды?** Урок 18.2
показал, что после тайм-аута первая попытка могла пройти. Тогда повтор сделает работу второй раз.

## 2. Экспоненциальная задержка

Если сервер перегружен, немедленный повтор делает только хуже. Поэтому мы ждём, и после каждого сбоя ждём дольше:
100 мс, 200 мс, 400 мс, 800 мс и так далее. Чтобы ожидание не становилось абсурдным, есть **потолок**. Поместите
это в `src/lib.rs`:

```rust
//! Retries, backoff and idempotency (Lesson 18.3).

use std::time::Duration;

/// Exponential backoff before retry number `attempt` (0, 1, 2, ...): base × 2^attempt, at most `cap`.
pub fn backoff(attempt: u32, base: Duration, cap: Duration) -> Duration {
    let factor = 2u32.saturating_pow(attempt);
    base.saturating_mul(factor).min(cap)
}
```

**Что происходит:**

1. `2u32.saturating_pow(attempt)` — это 2 в степени `attempt`. «Насыщающее» (saturating) значит, что вместо
   переполнения значение останавливается на наибольшем `u32` (Урок 12.2), поэтому попытка 40 не вызывает панику.
2. `base.saturating_mul(factor)` умножает длительность, снова без переполнения, а `.min(cap)` ограничивает её.

Напечатайте таблицу программой `src/bin/backoff_table.rs`:

```rust
use std::time::Duration;

use p18_retries::backoff;

fn main() {
    let base = Duration::from_millis(100);
    let cap = Duration::from_secs(5);
    for attempt in 0..8 {
        println!("retry {attempt}: wait {:?}", backoff(attempt, base, cap));
    }
}
```

`cargo run --bin backoff_table`:

```text
retry 0: wait 100ms
retry 1: wait 200ms
retry 2: wait 400ms
retry 3: wait 800ms
retry 4: wait 1.6s
retry 5: wait 3.2s
retry 6: wait 5s
retry 7: wait 5s
```

## 3. Джиттер: не повторяйте все разом

Представьте сервер, которым пользуются 1000 клиентов. Он падает и перезапускается. Все 1000 клиентов увидели сбой
запроса в один момент, и у всех одинаковая задержка, поэтому все они снова повторяют **в один и тот же момент**:
это **громоподобное стадо**. Свежий сервер получает 1000 запросов за одну миллисекунду и снова падает.

Решение — **джиттер**: каждый клиент ждёт *случайную* часть задержки. Самый простой и очень действенный вариант
называется **полным джиттером** (full jitter): ждать случайное время от нуля до задержки. Добавьте
`use rand::RngExt;` в начало `src/lib.rs`, а затем эту функцию:

```rust
/// "Full jitter": a random wait between zero and the exponential backoff.
pub fn full_jitter(attempt: u32, base: Duration, cap: Duration, rng: &mut impl RngExt) -> Duration {
    let max = backoff(attempt, base, cap).as_millis() as u64;
    Duration::from_millis(rng.random_range(0..=max))
}
```

`RngExt` — трейт `rand` 0.10 с удобными методами вроде `random_range`. Функция получает генератор случайных чисел
параметром, поэтому демонстрация или тест могут передать генератор с **зерном** (seed), который выдаёт одни и те же
числа при каждом запуске. Сравним 1000 клиентов с джиттером и без него, `src/bin/jitter.rs`:

```rust
use std::time::Duration;

use p18_retries::{backoff, full_jitter};
use rand::SeedableRng;
use rand::rngs::StdRng;

/// Prints how many of the clients retry in each 100 ms window, one `#` per 20 clients.
fn histogram(title: &str, waits: &[Duration]) {
    println!("{title}");
    for window in 0..5 {
        let from = Duration::from_millis(window * 100);
        let to = from + Duration::from_millis(100);
        let count = waits.iter().filter(|w| **w >= from && **w < to).count();
        let bar = "#".repeat(count / 20);
        println!(
            "  {:>3}-{:>3} ms {:>5} {bar}",
            window * 100,
            window * 100 + 99,
            count
        );
    }
}

fn main() {
    let base = Duration::from_millis(100);
    let cap = Duration::from_secs(5);
    let attempt = 2; // the third retry: the backoff is 400 ms
    let clients = 1000;
    let mut rng = StdRng::seed_from_u64(42);

    // Without jitter every client waits exactly 400 ms: all at the same moment.
    let fixed: Vec<Duration> = (0..clients).map(|_| backoff(attempt, base, cap)).collect();
    let jittered: Vec<Duration> = (0..clients)
        .map(|_| full_jitter(attempt, base, cap, &mut rng))
        .collect();

    histogram("without jitter", &fixed);
    histogram("with full jitter", &jittered);
}
```

`cargo run --bin jitter`:

```text
without jitter
    0- 99 ms     0 
  100-199 ms     0 
  200-299 ms     0 
  300-399 ms     0 
  400-499 ms  1000 ##################################################
with full jitter
    0- 99 ms   240 ############
  100-199 ms   251 ############
  200-299 ms   266 #############
  300-399 ms   241 ############
  400-499 ms     2 
```

**Что происходит:**

1. `StdRng::seed_from_u64(42)` создаёт генератор, который для зерна 42 всегда выдаёт одну и ту же
   последовательность. Настоящая программа использовала бы `rand::rng()` — генератор с зерном от операционной
   системы.
2. Без джиттера все 1000 повторов попадают в одно окно. С полным джиттером они равномерно распределены по 400 мс:
   примерно по 250 на 100 мс. (Двое в последнем окне вытянули ровно 400 мс, потому что `0..=max` включает `max`.)
3. Теперь сервер видит ровный поток вместо стены. Средний клиент ждёт вдвое меньше, чем раньше, а сервер
   восстанавливается вместо того, чтобы снова упасть.

> ⚡ **Производительность:** джиттер нужен не только для повторов. Любая периодическая задача, которая выполняется
> на многих машинах (обновление кэша, heartbeat-сообщения, cron-задачи «каждый ровный час»), должна добавлять
> немного случайности, иначе все машины ударят по общей базе данных в одну и ту же секунду.

## 4. Функция для повторов

Теперь объединим задержку и джиттер в одну вспомогательную функцию, которая может повторять **любую** асинхронную
операцию. Добавьте её в `src/lib.rs`:

```rust
/// How often and how patiently to retry.
pub struct RetryPolicy {
    pub max_attempts: u32,
    pub base: Duration,
    pub cap: Duration,
}

/// Calls `op` until it succeeds, until it fails with an error that `retryable`
/// rejects, or until `max_attempts` calls were made. Waits with full jitter in between.
pub async fn retry<T, E, F, Fut>(
    policy: &RetryPolicy,
    retryable: impl Fn(&E) -> bool,
    mut op: F,
) -> Result<T, E>
where
    F: FnMut() -> Fut,
    Fut: Future<Output = Result<T, E>>,
{
    let mut attempt = 0;
    loop {
        match op().await {
            Ok(value) => return Ok(value),
            Err(error) if attempt + 1 < policy.max_attempts && retryable(&error) => {
                let wait = full_jitter(attempt, policy.base, policy.cap, &mut rand::rng());
                tokio::time::sleep(wait).await;
                attempt += 1;
            }
            Err(error) => return Err(error),
        }
    }
}
```

**Что происходит:**

1. `op` — замыкание, которое **запускает** одну попытку: каждый вызов `op()` возвращает новый future типа `Fut`,
   который мы затем ждём через `.await`. Взять один future нельзя: future можно дождаться только один раз, а
   попыток может понадобиться несколько. `FnMut` позволяет замыканию менять захваченные переменные, например
   считать попытки.
2. `retryable` решает, какие ошибки стоят ещё одной попытки (раздел 1). Любая другая ошибка возвращается сразу.
3. Условие `attempt + 1 < policy.max_attempts` останавливается после последней разрешённой попытки и возвращает
   её ошибку.
4. `&mut rand::rng()` создаёт генератор с зерном от операционной системы только на одну эту строку. Он
   уничтожается до `.await`, и это важно: этот генератор нельзя перемещать между потоками, а Tokio может перенести
   задачу в другой поток на `.await`.

## 5. Идемпотентность

Повторять теперь легко. Будет ли это **правильно**, зависит от операции. Операция **идемпотентна**, если после
двукратного выполнения система в том же состоянии, что и после однократного.

| Операция | Идемпотентна? | Почему |
|---|---|---|
| Установить баланс 100 | да | Во второй раз он уже 100. |
| Прибавить 100 к балансу | **нет** | Дважды прибавит 200. |
| Удалить заказ 17 | да | Во второй раз удалять нечего. |
| Создать новый заказ | **нет** | Дважды создаст два заказа. |
| Прочитать баланс | да | Чтение ничего не меняет. |
| HTTP `GET`, `PUT`, `DELETE` | да, по определению | Этого требует RFC 9110, стандарт HTTP. |
| HTTP `POST` | **нет** | Обычно что-то создаёт или запускает действие. |

Если операция идемпотентна, «повторить после тайм-аута» всегда безопасно: даже если первая попытка прошла, вторая
ничего не изменит. Для неидемпотентных операций, таких как вклад, нужен приём.

## 6. Ключи идемпотентности

Приём такой: **клиент** даёт каждой логической операции уникальный идентификатор — **ключ идемпотентности** — и
отправляет тот же ключ с каждым повтором. Сервер помнит уже обработанные ключи вместе с ответом, который он дал.
Когда известный ключ приходит снова, сервер не делает работу повторно, а просто возвращает запомненный ответ.

Добавьте `use std::collections::HashMap;` в начало `src/lib.rs`, а затем правила банка в виде простой структуры
без всякой сети:

```rust
/// The bank's data and rules, without any networking.
#[derive(Default)]
pub struct Ledger {
    balance: u64,
    /// Idempotency key → the balance we answered the first time.
    seen: HashMap<u64, u64>,
}

impl Ledger {
    /// Deposits `amount`. A request with a key that was seen before changes nothing
    /// and gets the same answer as the first time.
    pub fn deposit(&mut self, key: Option<u64>, amount: u64) -> u64 {
        if let Some(&earlier) = key.and_then(|key| self.seen.get(&key)) {
            return earlier;
        }
        self.balance += amount;
        if let Some(key) = key {
            self.seen.insert(key, self.balance);
        }
        self.balance
    }

    pub fn balance(&self) -> u64 {
        self.balance
    }
}
```

Ключ — это `Option`, чтобы можно было сравнить: `None` — запрос старого образца, без ключа. Правила в чистой
структуре вдобавок легко тестировать отдельно (Упражнение 3).

Теперь «узел»-банк за плохой сетью. В Уроке 18.2 мы строили сеть на каналах и настоящих тайм-аутах; здесь, чтобы
код был коротким, мы только имитируем результат: банк **выполняет** каждый вклад, но для первых нескольких вкладов
ответ теряется, клиент напрасно ждёт `limit` и получает тайм-аут. Добавьте `use std::sync::Mutex;` в начало
`src/lib.rs`, а затем:

```rust
/// The answer did not come back in time.
#[derive(Debug, PartialEq)]
pub struct Timeout;

/// A bank node behind a bad network: the answers to the first `lost_replies`
/// deposits are lost on the way back.
pub struct Bank {
    ledger: Mutex<Ledger>,
    lost_replies: Mutex<usize>,
}

impl Bank {
    pub fn new(lost_replies: usize) -> Self {
        Bank {
            ledger: Mutex::new(Ledger::default()),
            lost_replies: Mutex::new(lost_replies),
        }
    }

    /// One deposit call over the network; the client waits at most `limit`.
    pub async fn deposit(
        &self,
        key: Option<u64>,
        amount: u64,
        limit: Duration,
    ) -> Result<u64, Timeout> {
        let new_balance = self.ledger.lock().unwrap().deposit(key, amount); // the work is done...
        let answer_lost = {
            let mut lost = self.lost_replies.lock().unwrap();
            let answer_lost = *lost > 0;
            if answer_lost {
                *lost -= 1;
            }
            answer_lost
        };
        if answer_lost {
            tokio::time::sleep(limit).await; // ...but the client waits in vain
            return Err(Timeout);
        }
        Ok(new_balance)
    }

    pub fn balance(&self) -> u64 {
        self.ledger.lock().unwrap().balance()
    }
}
```

**Что происходит:**

1. `Bank` оборачивает учётную книгу и счётчик ответов, которые нужно потерять. Оба под `Mutex`, потому что
   `deposit` принимает `&self` и много вызовов могут идти одновременно (Урок 14.2).
2. `deposit` **сначала** меняет учётную книгу. Затем проверяет, не из потерянных ли этот ответ. Внутренний блок
   `{ ... }` заканчивается до `.await`, поэтому блокировка снимается до сна: мьютекс из `std` никогда нельзя
   держать через `.await`.
3. Для потерянного ответа клиент ждёт весь `limit` и получает `Err(Timeout)`, хотя деньги уже зачислены.

Теперь эксперимент, `src/bin/double_charge.rs`. Он вносит 100 с повторами — один раз без ключа и один раз с ключом
7, каждый раз в новый банк, который теряет два ответа. Все ошибки здесь — тайм-ауты, а они временные, поэтому
замыкание `retryable` — просто `|_| true`:

```rust
use std::time::Duration;

use p18_retries::{Bank, RetryPolicy, retry};

#[tokio::main]
async fn main() {
    let policy = RetryPolicy {
        max_attempts: 5,
        base: Duration::from_millis(50),
        cap: Duration::from_secs(1),
    };
    let limit = Duration::from_millis(200);

    for key in [None, Some(7)] {
        // A fresh bank that loses the answers to the first two deposits.
        let bank = Bank::new(2);
        let mut attempt = 0;
        let result = retry(
            &policy,
            |_| true,
            || {
                attempt += 1;
                println!("key {key:?}: attempt {attempt}");
                bank.deposit(key, 100, limit)
            },
        )
        .await;
        println!(
            "key {key:?}: result {result:?}, balance {}\n",
            bank.balance()
        );
    }
}
```

`cargo run --bin double_charge`:

```text
key None: attempt 1
key None: attempt 2
key None: attempt 3
key None: result Ok(300), balance 300

key Some(7): attempt 1
key Some(7): attempt 2
key Some(7): attempt 3
key Some(7): result Ok(100), balance 100
```

**Что происходит:**

1. Без ключа попытки 1 и 2 банк **выполнил**, но их ответы потерялись; клиент увидел два тайм-аута и повторил.
   Попытка 3 получила ответ. Пользователь хотел внести 100, а в банке теперь 300.
2. С ключом 7 попытка 1 сделала вклад и запомнила «ключ 7 → 100». Попытки 2 и 3 несли тот же ключ; учётная книга
   узнала его и ответила 100, ничего не прибавив.
3. Функция повторов и сеть в обоих запусках одинаковы. Разницу делает только ключ.

Это стандартный рецепт распределённых систем: **доставка «хотя бы один раз» + идемпотентная обработка = эффект
«ровно один раз»**. Настоящая доставка «ровно один раз» по ненадёжной сети невозможна, но *эффект* «ровно один раз»
достижим.

> 🧪 **Тестирование:** превратите эксперимент в тесты, которым не нужно настоящее время. Добавьте в конец
> `src/lib.rs`:

```rust
#[cfg(test)]
mod tests {
    use super::*;

    const POLICY: RetryPolicy = RetryPolicy {
        max_attempts: 5,
        base: Duration::from_millis(50),
        cap: Duration::from_secs(1),
    };
    const LIMIT: Duration = Duration::from_millis(200);

    #[tokio::test(start_paused = true)]
    async fn retrying_without_a_key_counts_the_money_three_times() {
        let bank = Bank::new(2);
        let result = retry(&POLICY, |_| true, || bank.deposit(None, 100, LIMIT)).await;
        assert_eq!(result, Ok(300));
        assert_eq!(bank.balance(), 300);
    }

    #[tokio::test(start_paused = true)]
    async fn retrying_with_a_key_counts_the_money_once() {
        let bank = Bank::new(2);
        let result = retry(&POLICY, |_| true, || bank.deposit(Some(7), 100, LIMIT)).await;
        assert_eq!(result, Ok(100));
        assert_eq!(bank.balance(), 100);
    }
}
```

`cargo test --lib`:

```text
test tests::retrying_with_a_key_counts_the_money_once ... ok
test tests::retrying_without_a_key_counts_the_money_three_times ... ok

test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

Первый тест намеренно документирует ошибку: он предупредит, если кто-то «исправит» банк так, что проблема
спрячется, а не решится.

Три правила заставляют ключи идемпотентности работать в настоящем сервисе:

- **Один ключ на логическую операцию, а не на попытку.** Клиент создаёт ключ *до* первой попытки (например UUID,
  Universally Unique Identifier, универсальный уникальный идентификатор: случайный 128-битный id) и повторно
  использует его в каждом повторе.
- **Храните ключ и изменение вместе.** Если сервер зачислил деньги и упал, не успев сохранить ключ, повтор зачислит
  их снова. Записывайте и то и другое в одной транзакции базы данных ([Урок 16.7](../16-backend-services/07-migrations-and-transactions.md)).
- **Решите, как долго хранить ключи.** Таблица увиденных ключей растёт вечно, если старые ключи не истекают,
  например через 24 часа; повторы должны укладываться в это время.

> 🔒 **Безопасность:** сервер не должен доверять одному лишь ключу. Если по ключу пользователя A можно получить
> сохранённый ответ пользователя B, угаданный ключ приведёт к утечке данных. Храните ключ вместе с пользователем и
> проверяйте оба.

## Частые ошибки

**Новый ключ на каждую попытку.** Это выглядит как идемпотентность, но ничего не даёт. Если в `double_charge.rs`
создавать ключ *внутри* замыкания, каждая попытка получает новый случайный ключ:

```rust
            || {
                attempt += 1;
                println!("key {key:?}: attempt {attempt}");
                // WRONG: a new key for every attempt
                bank.deposit(Some(rand::random::<u64>()), 100, limit)
            },
```

```text
key None: result Ok(300), balance 300
key Some(7): result Ok(300), balance 300
```

Сервер видит три разных ключа, а значит, три разных вклада. Исправление: создайте ключ один раз, до вызова `retry`,
и пусть замыкание его только копирует.

> ⚠️ **Частая ошибка: повторы на каждом уровне.** Если веб-сервис повторяет вызов базы данных 3 раза, его клиент
> повторяет вызов веб-сервиса 3 раза, и приложение пользователя повторяет 3 раза, то во время аварии один клик может
> превратиться в 3 × 3 × 3 = 27 вызовов базы данных. Повторяйте на **одном** уровне, обычно ближайшем к отказавшему
> вызову, а остальные пусть быстро сообщают об ошибке.

> ⚠️ **Частая ошибка: повтор постоянных ошибок.** Пять повторов «404 Not Found» лишь заставят пользователя ждать в
> пять раз дольше тот же ответ. Упражнение 2 сортирует коды состояния.

## Попробуйте сами

1. В `double_charge.rs` замените `Bank::new(2)` на `Bank::new(4)`. Без ключа в банке окажется 500. С ключом — всё
   ещё 100.
2. Замените на `Bank::new(10)`. Все пять попыток завершаются тайм-аутом, и `retry` возвращает `Err(Timeout)`.
   Посмотрите на баланс: он **не** ноль. «Неудавшийся» вызов всё равно внёс деньги (с ключом — ровно один раз).

## Упражнения

### Упражнение 1 — Тест задержки (легко)

Напишите интеграционный тест `tests/backoff.rs`, который проверяет две вещи о `backoff`: при базе 100 мс и потолке
1 с первые шесть ожиданий равны 100, 200, 400, 800, 1000 и 1000 мс; а огромные номера попыток (40 и `u32::MAX`)
возвращают потолок, а не вызывают панику.

<details>
<summary>Подсказка</summary>

Соберите ожидания в вектор миллисекунд через `(0..6).map(|a| backoff(a, base, cap).as_millis())` и сравните весь
вектор одним `assert_eq!`.

</details>

<details>
<summary>Ответ</summary>

```rust
use std::time::Duration;

use p18_retries::backoff;

#[test]
fn doubles_until_the_cap() {
    let base = Duration::from_millis(100);
    let cap = Duration::from_secs(1);
    let waits: Vec<u128> = (0..6).map(|a| backoff(a, base, cap).as_millis()).collect();
    assert_eq!(waits, vec![100, 200, 400, 800, 1000, 1000]);
}

#[test]
fn a_huge_attempt_number_does_not_overflow() {
    let cap = Duration::from_secs(30);
    assert_eq!(backoff(40, Duration::from_millis(100), cap), cap);
    assert_eq!(backoff(u32::MAX, Duration::from_secs(1), cap), cap);
}
```

`cargo test --test backoff`:

```text
test a_huge_attempt_number_does_not_overflow ... ok
test doubles_until_the_cap ... ok

test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

Почему это работает: с `pow` вместо `saturating_pow` второй тест в отладочной сборке паниковал бы с сообщением
«attempt to exponentiate with overflow». Тест охраняет эту деталь.

</details>

### Упражнение 2 — Какие ответы HTTP заслуживают повтора? (средне)

Создайте `src/http.rs` (и добавьте `pub mod http;` в `src/lib.rs`) с функцией `pub fn is_retryable(status: u16) -> bool`.
Повторяйте 408 (Request Timeout), 429 (Too Many Requests) и серверные ошибки 5xx, кроме 501 (Not Implemented) и 505
(HTTP Version Not Supported), которые никогда не сработают. Больше ничего не повторяйте. Напишите табличный тест
минимум с шестью кодами для повтора и восемью без повтора.

<details>
<summary>Подсказка</summary>

`match` с диапазонами читается хорошо: `500..=599 => true`. Поставьте два исключения в ветку **до** диапазона,
потому что `match` берёт первую подходящую ветку.

</details>

<details>
<summary>Ответ</summary>

```rust
//! Exercise 2: which HTTP answers are worth a retry.

/// `true` if a request that got this HTTP status may succeed when it is sent again.
pub fn is_retryable(status: u16) -> bool {
    match status {
        408 | 429 => true,  // Request Timeout, Too Many Requests
        501 | 505 => false, // Not Implemented, HTTP Version Not Supported: will never work
        500..=599 => true,  // server-side trouble, often temporary
        _ => false,         // success, redirects and the other 4xx client errors
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn retries_only_temporary_problems() {
        for status in [408, 429, 500, 502, 503, 504] {
            assert!(is_retryable(status), "{status} should be retried");
        }
        for status in [200, 201, 301, 400, 401, 403, 404, 409, 422, 501] {
            assert!(!is_retryable(status), "{status} should not be retried");
        }
    }
}
```

Почему это работает: сообщение в каждом `assert!` называет код, на котором тест упал, поэтому и табличный тест
точно говорит, какой случай неверен. Помните, что «можно повторить» — только половина решения: `POST`, получивший
503, безопасно повторять только с ключом идемпотентности.

</details>

### Упражнение 3 — Докажите, что повторы ничего не меняют (сложно)

Напишите тесты на основе свойств (Урок 09.4) в `tests/ledger_props.rs` для `Ledger`. Генерируйте списки до 20
запросов `(key, amount)` с ключами из `0..5` (чтобы повторы были частыми) и суммами из `1..100`. Свойство 1:
применить весь список дважды — то же, что применить его один раз. Свойство 2: баланс равен сумме **первых** сумм,
увиденных для каждого ключа.

<details>
<summary>Подсказка</summary>

`prop::collection::vec((0u64..5, 1u64..100), 0..20)` генерирует такие списки. Для свойства 2 постройте ожидаемое
значение через `HashMap` и `entry(key).or_insert(amount)`, который сохраняет только первую сумму для каждого ключа.

</details>

<details>
<summary>Ответ</summary>

```rust
use std::collections::HashMap;

use p18_retries::Ledger;
use proptest::prelude::*;

proptest! {
    #[test]
    fn repeating_every_request_changes_nothing(
        requests in prop::collection::vec((0u64..5, 1u64..100), 0..20)
    ) {
        let mut once = Ledger::default();
        for &(key, amount) in &requests {
            once.deposit(Some(key), amount);
        }
        let mut twice = Ledger::default();
        for &(key, amount) in requests.iter().chain(requests.iter()) {
            twice.deposit(Some(key), amount);
        }
        prop_assert_eq!(once.balance(), twice.balance());
    }

    #[test]
    fn only_the_first_request_with_a_key_counts(
        requests in prop::collection::vec((0u64..5, 1u64..100), 0..20)
    ) {
        let mut ledger = Ledger::default();
        let mut first: HashMap<u64, u64> = HashMap::new();
        for &(key, amount) in &requests {
            ledger.deposit(Some(key), amount);
            first.entry(key).or_insert(amount);
        }
        prop_assert_eq!(ledger.balance(), first.values().sum::<u64>());
    }
}
```

`cargo test --test ledger_props`:

```text
test repeating_every_request_changes_nothing ... ok
test only_the_first_request_with_a_key_counts ... ok

test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.01s
```

Почему это работает: свойство 1 — это определение идемпотентности, проверенное на 256 случайных историях.
Свойство 2 говорит ещё и то, что повтор с **другой** суммой игнорируется: сервер снова отвечает на первый запрос, он
не смешивает их. Попробуйте вклады с `None` вместо `Some(key)`: свойство 1 сразу падает, и proptest сжимает
контрпример до одного запроса.

</details>

## Вопросы для самопроверки

1. Что из этого клиенту стоит повторить: тайм-аут, «403 Forbidden», «503 Service Unavailable»?

<details>
<summary>Ответ</summary>

Тайм-аут и 503 — временные, их стоит повторить с задержкой. 403 — постоянная ошибка: тот же запрос снова
отклонят.

</details>

2. Какую проблему решает джиттер?

<details>
<summary>Ответ</summary>

Громоподобное стадо: клиенты, у которых сбой случился в один момент, иначе повторили бы запрос тоже в один момент и
снова перегрузили бы восстанавливающийся сервер. Случайные ожидания распределяют повторы во времени.

</details>

3. Идемпотентна ли операция «установить email пользователя 5 равным a@b.c»? А «отправить пользователю 5
   приветственное письмо»?

<details>
<summary>Ответ</summary>

Первая идемпотентна: после двух выполнений email тот же. Вторая — нет: дважды уйдут два письма. Ей нужен ключ
идемпотентности (или флаг «приветствие отправлено», который проверяется и ставится вместе с отправкой).

</details>

4. Почему клиент должен создать ключ идемпотентности до первой попытки?

<details>
<summary>Ответ</summary>

Ключ должен быть одинаковым для всех попыток одной операции. Если каждая попытка создаёт свой ключ, сервер видит в
каждом повторе новую операцию, и работа выполняется снова.

</details>

5. Что даёт «доставка „хотя бы один раз“ + идемпотентная обработка»?

<details>
<summary>Ответ</summary>

Эффект «ровно один раз»: сообщение может прийти несколько раз, но повторная обработка ничего не меняет. Настоящая
доставка «ровно один раз» по ненадёжной сети невозможна.

</details>

## Итоги

- Повторяйте только временные сбои и только операции, которые безопасно повторить.
- Между повторами ждите с экспоненциальной задержкой и потолком и добавляйте полный джиттер, чтобы клиенты не
  повторяли в ногу.
- Небольшая обобщённая функция `retry` принимает замыкание, которое запускает одну попытку; тестируйте её с
  остановленными часами.
- Операция идемпотентна, если выполнить её дважды — то же, что один раз. Чтение, «установить» и «удалить» обычно
  идемпотентны; «прибавить», «создать» и «отправить» — нет.
- Ключи идемпотентности делают любую операцию безопасной для повтора: один ключ на логическую операцию, хранится
  вместе с изменением, ограниченное время.
- Доставка «хотя бы один раз» плюс идемпотентная обработка дают эффект «ровно один раз».

## Источники и что почитать

- [Timeouts, retries, and backoff with jitter](https://aws.amazon.com/builders-library/timeouts-retries-and-backoff-with-jitter/) — Amazon Builders' Library о повторах в больших системах, включая умножение повторов на разных уровнях.
- [Exponential Backoff And Jitter](https://aws.amazon.com/blogs/architecture/exponential-backoff-and-jitter/) — статья в блоге AWS (Amazon Web Services) Architecture Blog, которая сравнивает стратегии джиттера на симуляциях; «полный джиттер» пришёл отсюда.
- [Making retries safe with idempotent APIs](https://aws.amazon.com/builders-library/making-retries-safe-with-idempotent-APIs/) — как устроены ключи идемпотентности (client request tokens) в настоящем облачном API (Application Programming Interface, программный интерфейс).
- [RFC 9110 — HTTP Semantics](https://www.rfc-editor.org/rfc/rfc9110.html) — стандарт HTTP; его раздел об идемпотентных методах определяет, какие методы безопасно повторять.
- [Документация `rand`](https://docs.rs/rand/latest/rand/) — генераторы случайных чисел, зёрна и `random_range`.

---

[🏠 Начало](../START-HERE.md) · [← 18.2 Сбои и тайм-ауты](02-failures-and-timeouts.md) · [📋 Этап 18](README.md) · [18.4 Время и часы →](04-time-and-clocks.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
