<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 18.3 Повторные попытки, задержки и идемпотентность](03-retries-backoff-and-idempotency.md) · [📋 Этап 18](README.md) · [18.5 Логические часы: часы Лэмпорта и векторные часы →](05-logical-clocks.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/18-distributed-systems/04-time-and-clocks.md) <!-- nav -->

# Урок 18.4 — Время и часы

> **Этап 18 — Распределённые системы** · Урок 4 из 15 · ⏱ около 2,5 часа (2 занятия)

## Что вы узнаете

- Двое часов в каждом компьютере — настенные и монотонные — и какие для чего использовать.
- Почему настенные часы могут прыгнуть назад и как `SystemTime` и `Instant` в Rust делают это видимым.
- Почему часы двух машин никогда точно не совпадают и насколько они уходят.
- Как правило «побеждает последняя запись» с метками времени незаметно теряет данные, когда часы расходятся.
- Почему блокировка с тайм-аутом (аренда) сама по себе небезопасна и как это исправляют токены ограждения.

## Перед началом

- Вы прошли [Урок 18.3 — Повторные попытки, задержки и идемпотентность](03-retries-backoff-and-idempotency.md).
- Вы знаете `Duration` и `Instant` с прошлых этапов и замёрзший узел из [Урока 18.1](01-what-a-distributed-system-is.md).
- Создайте пакет для этого урока. Зависимости ему не нужны:

```bash
cd ~/projects/rust
cargo new --lib code/p18-clocks
cd code/p18-clocks
mkdir src/bin
```

## Новые слова

| Слово | Значение |
|---|---|
| **настенные часы** (wall clock) | Часы, которые показывают время суток и дату, как часы на стене. В Rust: `SystemTime`. Их можно переставить, и они могут прыгать. |
| **монотонные часы** (monotonic clock) | Часы, которые идут только вперёд, с постоянной скоростью, от произвольной начальной точки. В Rust: `Instant`. Хороши для измерения длительностей; на другой машине бессмысленны. |
| **UTC** (Coordinated Universal Time, всемирное координированное время) | Мировое опорное время, без часовых поясов и летнего времени. |
| **эпоха Unix** (Unix epoch) | Момент 1970-01-01 00:00:00 UTC. Время по настенным часам часто хранят как «секунды с начала эпохи». |
| **NTP** (Network Time Protocol, протокол сетевого времени) | Протокол, по которому компьютеры выставляют свои настенные часы по серверам времени через сеть. |
| **уход часов** (clock drift) | Часы идут чуть быстрее или чуть медленнее. Измеряется в **ppm** (parts per million, миллионных долях): 1 ppm — это 1 мкс ошибки за секунду. |
| **рассинхронизация часов** (clock skew) | Разница между часами двух машин в один и тот же момент. |
| **LWW** (Last Write Wins, «побеждает последняя запись») | Правило для конфликтов: оставить значение с самой поздней меткой времени, остальные выбросить. |
| **аренда** (lease) | Блокировка, действующая ограниченное время, чтобы она освободилась сама, если её владелец умрёт. |
| **токен ограждения** (fencing token) | Число, которое растёт с каждой новой арендой. Хранилище отклоняет записи, несущие число меньше уже увиденного. |

## 1. Двое часов в каждом компьютере

Каждый компьютер предлагает двое совсем разных часов:

| | Настенные часы (`SystemTime`) | Монотонные часы (`Instant`) |
|---|---|---|
| Отвечают на вопрос | «Какое сейчас время и дата?» | «Сколько времени прошло с того момента?» |
| Кто выставляет | NTP, администратор, пользователь | никто: они просто считают |
| Могут прыгнуть | да, вперёд и **назад** | нет, никогда назад |
| Сравнимы между машинами | приблизительно (если обе используют NTP) | **нет**: каждая машина начинает отсчёт со своей точки |
| Для чего использовать | метки времени для людей, логи, сроки истечения | тайм-ауты, измерение длительностей, ограничение частоты |

Прочитаем и те и другие, `src/bin/two_clocks.rs`:

```rust
use std::thread::sleep;
use std::time::{Duration, Instant, SystemTime, UNIX_EPOCH};

fn main() {
    let wall_start = SystemTime::now();
    let mono_start = Instant::now();
    sleep(Duration::from_millis(250));
    println!("wall clock:      {:?}", wall_start.elapsed());
    println!("monotonic clock: {:?}", mono_start.elapsed());

    let since_epoch = wall_start.duration_since(UNIX_EPOCH).unwrap();
    println!("seconds since 1970-01-01 00:00 UTC: {}", since_epoch.as_secs());
}
```

`cargo run --bin two_clocks` (ваши числа будут другими):

```text
wall clock:      Ok(255.015ms)
monotonic clock: 255.109292ms
seconds since 1970-01-01 00:00 UTC: 1794990293
```

**Что происходит:**

1. Оба вида часов намерили около 250 мс (плюс несколько миллисекунд, пока программа просыпалась). На спокойной
   машине они согласны.
2. Но посмотрите на типы. `SystemTime::elapsed()` возвращает `Result`: Rust напоминает, что настенные часы за это
   время могли уйти назад, и тогда положительной длительности не существует. `Instant::elapsed()` возвращает просто
   `Duration`, потому что монотонные часы назад не идут.

## 2. Когда настенные часы прыгают

Настенные часы прыгают чаще, чем можно подумать:

- NTP обнаруживает, что часы спешат на 2 секунды, и **переставляет** (step) их назад. (При маленьких ошибках NTP
  предпочитает **подстраивать** (slew) часы: немного замедлить их ход, пока они не станут верными. При больших —
  переставляет.)
- Администратор или пользователь меняет время вручную.
- Виртуальную машину приостановили, перенесли на другой хост и возобновили; её часы вдруг сильно отстают.
- **Високосная секунда** (leap second): время от времени к UTC добавляют лишнюю секунду, чтобы оно не расходилось с
  вращением Земли, и системы обрабатывают её по-разному, некоторые — повторяя секунду.

Rust позволяет увидеть такой прыжок, не меняя часы вашего Mac. Мы построим два показания настенных часов вручную,
`src/bin/backwards.rs`:

```rust
use std::time::{Duration, Instant, UNIX_EPOCH};

fn main() {
    // Pretend the time service moved the wall clock 2 seconds back between two readings.
    let first = UNIX_EPOCH + Duration::from_secs(1_800_000_000);
    let second = first - Duration::from_secs(2);
    println!("wall: second - first = {:?}", second.duration_since(first));

    // An Instant can never be earlier than one read before it. If you subtract the
    // wrong way round, you get zero instead of a negative duration.
    let a = Instant::now();
    let b = Instant::now();
    println!("monotonic: b - a = {:?}", b.duration_since(a));
    println!("monotonic: a - b = {:?}", a.duration_since(b));
}
```

`cargo run --bin backwards`:

```text
wall: second - first = Err(SystemTimeError(2s))
monotonic: b - a = 42ns
monotonic: a - b = 0ns
```

**Что происходит:**

1. `UNIX_EPOCH + Duration::from_secs(1_800_000_000)` — это `SystemTime` в 2027 году; `second` на 2 секунды
   *раньше*, как будто часы переставили назад между двумя показаниями.
2. `second.duration_since(first)` не может дать отрицательную `Duration` (`Duration` никогда не бывает
   отрицательной), поэтому возвращает `Err(SystemTimeError(2s))`: «другое время на 2 с позже этого».
3. Для `Instant` вызов `duration_since` с аргументами наоборот возвращает ноль, а не ошибку. Два показания
   монотонных часов подряд отличаются на несколько десятков наносекунд.

Отсюда правило: **измеряйте длительности, тайм-ауты и дедлайны через `Instant`, никогда через `SystemTime`.**
Таймеры Tokio (Уроки 18.2 и 18.3) уже по этой причине используют монотонные часы.

## 3. Часы уходят, и машины не согласны

Часы в компьютере — маленький кварцевый кристалл, который колеблется с известной частотой. Двух одинаковых
кристаллов не бывает, а их скорость меняется с температурой. Типичные часы компьютера ошибаются на 10–100 ppm. При
50 ppm часы убегают или отстают примерно на 4,3 секунды в сутки (это считает Упражнение 1).

NTP исправляет это, опрашивая серверы времени и регулярно подстраивая часы. Но NTP сам работает через сеть
(Урок 18.2: сообщения задерживаются на неизвестное время), поэтому результат никогда не идеален. Типичные числа:

| Условия | Типичная рассинхронизация двух машин |
|---|---|
| NTP через интернет | от нескольких до десятков миллисекунд |
| NTP внутри одного дата-центра | около миллисекунды или меньше |
| NTP сломан или заблокирован файрволом | растёт без предела, секунды в сутки |
| Специальное оборудование (приёмники GPS, Global Positioning System, глобальной системы позиционирования, и атомные часы) | микросекунды |

Важное следствие: **метку времени машины A и метку времени машины B нельзя точно сравнить.** Если A говорит
12:00:00.005, а B — 12:00:00.003, событие B вполне могло случиться *после* события A.

## 4. «Побеждает последняя запись» теряет записи

Многие реплицируемые базы данных разрешают конфликты по правилу **LWW** (Last Write Wins, «побеждает последняя
запись»): каждая запись несёт метку времени по настенным часам автора, и когда две реплики расходятся, побеждает
значение с большей меткой. Это просто — и незаметно теряет данные, когда часы расходятся. Посмотрим. Поместите это
в `src/lib.rs`:

```rust
//! Time and clocks (Lesson 18.4).

/// A register that keeps the value with the highest timestamp: "last write wins".
#[derive(Debug, Default)]
pub struct LwwRegister {
    value: Option<String>,
    timestamp_ms: u64,
}

impl LwwRegister {
    /// Stores `value` if its timestamp is newer than the stored one. Returns whether it was stored.
    pub fn write(&mut self, value: &str, timestamp_ms: u64) -> bool {
        if self.value.is_some() && timestamp_ms <= self.timestamp_ms {
            return false;
        }
        self.value = Some(value.to_string());
        self.timestamp_ms = timestamp_ms;
        true
    }

    pub fn read(&self) -> Option<&str> {
        self.value.as_deref()
    }
}

/// The clock of one machine: the real time plus this machine's error.
pub struct NodeClock {
    pub offset_ms: i64,
}

impl NodeClock {
    pub fn now_ms(&self, real_ms: u64) -> u64 {
        real_ms.saturating_add_signed(self.offset_ms)
    }
}
```

`NodeClock` имитирует машину, чьи часы на `offset_ms` спешат (положительное число) или отстают (отрицательное).
`saturating_add_signed` прибавляет `i64` к `u64` без переполнения. Теперь два узла пишут в один ключ,
`src/bin/lost_update.rs`:

```rust
use p18_clocks::{LwwRegister, NodeClock};

fn main() {
    let a = NodeClock { offset_ms: 100 }; // node a's clock is 100 ms fast
    let b = NodeClock { offset_ms: 0 }; // node b's clock is right
    let mut x = LwwRegister::default();

    let ts = a.now_ms(1000);
    println!(
        "real 1000 ms: a writes \"1\" with timestamp {ts}: stored {}",
        x.write("1", ts)
    );
    let ts = b.now_ms(1050);
    println!(
        "real 1050 ms: b writes \"2\" with timestamp {ts}: stored {}",
        x.write("2", ts)
    );
    println!("x = {:?}", x.read());
}
```

`cargo run --bin lost_update`:

```text
real 1000 ms: a writes "1" with timestamp 1100: stored true
real 1050 ms: b writes "2" with timestamp 1050: stored false
x = Some("1")
```

Узел `b` записал на 50 мс **позже** узла `a` в реальном времени — возможно, прочитав значение `a` и решив его
изменить. Но часы `a` спешат на 100 мс, поэтому у его старой записи метка больше, и более новая запись `b`
выбрасывается. Ни ошибки, ни предупреждения: данные просто пропали. При рассинхронизации в 100 мс может потеряться
каждая запись, сделанная меньше чем через 100 мс после записи спешащего узла.

> 🧪 **Тестирование:** такие ошибки никогда не проявляются на одной машине, где все записи используют одни часы.
> Имитируйте часы, как это делает `NodeClock`, и тестируйте с рассинхронизацией. Добавьте в `src/lib.rs`:

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn a_fast_clock_hides_a_later_write() {
        let fast = NodeClock { offset_ms: 100 };
        let correct = NodeClock { offset_ms: 0 };
        let mut x = LwwRegister::default();
        assert!(x.write("1", fast.now_ms(1000))); // real time 1000 ms
        assert!(!x.write("2", correct.now_ms(1050))); // real time 1050 ms: later!
        assert_eq!(x.read(), Some("1"));
    }
}
```

LWW всё ещё используют там, где потеря одновременной записи допустима, например для поля «последний раз был в сети».
Для всего важного системы решают порядок событий **логическими часами** (следующий урок) или **консенсусом**
(Уроки 18.9–18.11), а не настенными часами.

## 5. Аренда и токены ограждения

Распределённую блокировку часто строят как **аренду**: «клиент 1 держит блокировку следующие 10 секунд». Если
клиент 1 упадёт, аренда просто истечёт, и никто не будет ждать вечно. Опасность — клиент, который **приостановлен**, а
не упал (Урок 18.1): он не знает, что прошло время.

```text
time   lock service             client 1                      client 2
 0s    lease to 1 until 10s     "I have the lock"
 1s                             ...frozen (overloaded machine)...
12s    lease ran out;
       lease to 2 until 22s                                   "I have the lock", writes
15s                             wakes up: "I still have the lock", writes  ← two writers!
```

Клиент 1 не может себя защитить: с его точки зрения между «у меня блокировка» и «я пишу» время не прошло. Проверка
часов прямо перед записью тоже не помогает, потому что пауза может случиться между проверкой и записью.

Исправление должно быть в **хранилище**: каждая аренда приходит с **токеном ограждения** — числом, которое растёт с
каждой новой арендой, — а хранилище отклоняет любую запись с токеном старше самого нового из уже увиденных.
Добавьте в `src/lib.rs` (перед модулем тестов):

```rust
/// A lock service that gives out leases. Every new lease gets a larger fencing token.
#[derive(Default)]
pub struct LockService {
    last_token: u64,
    lease_ends_at: u64,
}

impl LockService {
    /// Gives a lease of `lease` seconds if nobody holds a valid one at time `now`.
    pub fn acquire(&mut self, now: u64, lease: u64) -> Option<u64> {
        if now < self.lease_ends_at {
            return None; // somebody else still holds the lock
        }
        self.last_token += 1;
        self.lease_ends_at = now + lease;
        Some(self.last_token)
    }
}

/// Storage that refuses writes with an older token than the newest it has seen.
#[derive(Default)]
pub struct FencedStorage {
    highest_token: u64,
    pub log: Vec<String>,
}

impl FencedStorage {
    pub fn write(&mut self, token: u64, text: &str) -> Result<(), String> {
        if token < self.highest_token {
            return Err(format!(
                "token {token} is older than {}",
                self.highest_token
            ));
        }
        self.highest_token = token;
        self.log.push(text.to_string());
        Ok(())
    }
}
```

И история с диаграммы, `src/bin/fencing.rs`:

```rust
use p18_clocks::{FencedStorage, LockService};

fn main() {
    let mut locks = LockService::default();
    let mut storage = FencedStorage::default();

    let token1 = locks.acquire(0, 10).unwrap();
    println!("t=0s:  client 1 gets the lock for 10 s, token {token1}");
    println!("t=1s:  client 1 freezes (its machine is overloaded)");
    let token2 = locks.acquire(12, 10).unwrap();
    println!("t=12s: the lease has run out; client 2 gets the lock, token {token2}");
    println!(
        "t=13s: client 2 writes: {:?}",
        storage.write(token2, "from client 2")
    );
    println!("t=15s: client 1 wakes up, still thinks it holds the lock, and writes:");
    println!("       {:?}", storage.write(token1, "from client 1"));
    println!("stored: {:?}", storage.log);
}
```

`cargo run --bin fencing`:

```text
t=0s:  client 1 gets the lock for 10 s, token 1
t=1s:  client 1 freezes (its machine is overloaded)
t=12s: the lease has run out; client 2 gets the lock, token 2
t=13s: client 2 writes: Ok(())
t=15s: client 1 wakes up, still thinks it holds the lock, and writes:
       Err("token 1 is older than 2")
stored: ["from client 2"]
```

**Что происходит:**

1. Служба блокировок и хранилище решают, кто может писать, только по токену, а не по часам. Время аренды нужно лишь
   для того, чтобы решить, когда можно выдать **новую** аренду.
2. Запись клиента 2 подняла `highest_token` хранилища до 2. Когда проснувшийся клиент 1 приходит с токеном 1,
   хранилище знает, что существует более новый владелец блокировки, и отказывает.

Та же идея снова появится в Raft: его номер **терма** (term, Урок 18.10) — это токен ограждения для лидеров.

Добавьте этот тест в `mod tests`:

```rust
    #[test]
    fn a_paused_client_cannot_overwrite_newer_data() {
        let mut locks = LockService::default();
        let mut storage = FencedStorage::default();
        let token1 = locks.acquire(0, 10).unwrap();
        assert_eq!(locks.acquire(5, 10), None); // the lease of client 1 is still valid
        let token2 = locks.acquire(12, 10).unwrap(); // client 1's lease ran out
        assert_eq!(storage.write(token2, "from client 2"), Ok(()));
        assert!(storage.write(token1, "from client 1").is_err());
        assert_eq!(storage.log, vec!["from client 2"]);
    }
```

`cargo test --lib`:

```text
test tests::a_fast_clock_hides_a_later_write ... ok
test tests::a_paused_client_cannot_overwrite_newer_data ... ok

test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

## 6. Правила обращения со временем в распределённых системах

| Задача | Что использовать |
|---|---|
| Тайм-ауты, дедлайны, измерение того, сколько что-то длилось | монотонные часы (`Instant`, `tokio::time::Instant`) |
| Метка времени, которую читает человек (логи, «создано в») | настенные часы в UTC (`SystemTime`) |
| Решить, какое из двух событий на **разных** машинах было первым | не настенные часы: логические часы (Урок 18.5) или консенсус (Уроки 18.9–18.11) |
| Гарантировать, что только один узел выступает владельцем чего-то | аренда **плюс** токены ограждения, которые проверяет хранилище |
| Истечение срока, которое может слегка опоздать или поспешить (записи кэша, сессии) | настенные часы с запасом больше ожидаемой рассинхронизации |

## Частые ошибки

**`unwrap` разницы настенных часов.** `later.duration_since(earlier).unwrap()` выглядит безобидно и работает месяцами
— пока NTP не переставит часы назад (`src/bin/unwrap_jump.rs`):

```rust
use std::time::{Duration, UNIX_EPOCH};

fn main() {
    let first = UNIX_EPOCH + Duration::from_secs(1_800_000_000);
    let second = first - Duration::from_secs(2); // the wall clock jumped back
    let took = second.duration_since(first).unwrap();
    println!("took {took:?}");
}
```

```text
thread 'main' (6295495) panicked at src/bin/unwrap_jump.rs:6:45:
called `Result::unwrap()` on an `Err` value: SystemTimeError(2s)
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
```

(Число после `'main'` обозначает поток и меняется при каждом запуске.) Исправление: измеряйте длительности через
`Instant`. Если действительно нужно вычесть два показания настенных часов, обработайте ошибку, например через
`.unwrap_or(Duration::ZERO)`.

## Попробуйте сами

1. В `lost_update.rs` измените смещение узла `a` на `-100` (отстаёт на 100 мс). Теперь, как и должно быть, побеждает
   запись `b`. Затем замените ещё реальное время записи `b` с `1050` на `950` в обоих местах, чтобы `b` писал на 50 мс
   **раньше** `a` (просто его сообщение пришло позже). Запустите: `x` всё ещё `"2"`. На этот раз отстающие часы
   заставили победить **более старую** запись, а потерялась более новая запись `a`.

## Упражнения

### Упражнение 1 — Насколько уходят часы? (легко)

Создайте `src/drift.rs` (и добавьте `pub mod drift;` в `src/lib.rs`) с функциями
`pub fn drift_per_day(ppm: f64) -> Duration` и `pub fn max_sync_interval(ppm: f64, allowed: Duration) -> Duration`:
как часто нужно поправлять часы, чтобы их ошибка никогда не превышала `allowed`. Напишите тесты: 50 ppm уходят на
4320 мс в сутки, 1 ppm — на 86 400 мкс в сутки, а при 50 ppm и допустимых 10 мс часы нужно поправлять как минимум
каждые 200 с.

<details>
<summary>Подсказка</summary>

В сутках 24 × 60 × 60 секунд; ошибка равна `секунды × ppm / 1 000 000`. Для интервала переверните формулу:
`допустимые_секунды × 1 000 000 / ppm`. `Duration::from_secs_f64` строит `Duration` из `f64`.

</details>

<details>
<summary>Ответ</summary>

```rust
//! Exercises 1 and 3.

use std::time::Duration;

/// How far a clock that is wrong by `ppm` parts per million drifts in one day.
pub fn drift_per_day(ppm: f64) -> Duration {
    Duration::from_secs_f64(24.0 * 60.0 * 60.0 * ppm / 1_000_000.0)
}

/// How often a clock must be corrected so that it never drifts more than `allowed`.
pub fn max_sync_interval(ppm: f64, allowed: Duration) -> Duration {
    Duration::from_secs_f64(allowed.as_secs_f64() * 1_000_000.0 / ppm)
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn fifty_ppm_is_more_than_four_seconds_a_day() {
        assert_eq!(drift_per_day(50.0).as_millis(), 4320);
        assert_eq!(drift_per_day(1.0).as_micros(), 86_400);
    }

    #[test]
    fn sync_often_enough_for_ten_milliseconds() {
        let interval = max_sync_interval(50.0, Duration::from_millis(10));
        assert_eq!(interval.as_secs(), 200);
    }
}
```

Почему это работает: 86 400 с × 50 / 1 000 000 = 4,32 с. Вторая функция объясняет, почему клиенты NTP сверяют время
каждые несколько минут, а не раз в сутки: дешёвым часам это нужно, чтобы оставаться в пределах миллисекунд.

</details>

### Упражнение 2 — Последняя запись побеждает, без ничьих (средне)

Две записи могут нести **одну и ту же** миллисекунду. Тогда `LwwRegister` оставляет ту, что пришла первой, и две
реплики, получившие записи в разном порядке, приходят к разным значениям. Создайте `src/stamp.rs` (и
`pub mod stamp;`) со структурой `Stamp { ms: u64, node: u32 }`, которая сравнивается сначала по `ms`, а при равенстве —
по `node`, и с `Register`, который хранит значение с наибольшей меткой. Напишите тест: две записи с одинаковым `ms`
от узлов 1 и 2, применённые в обоих порядках, дают равные регистры со значением от узла 2.

<details>
<summary>Подсказка</summary>

`#[derive(PartialEq, Eq, PartialOrd, Ord)]` на структуре сравнивает поля в том порядке, в котором они записаны, как
словарь сравнивает слова буква за буквой. Поэтому ставьте `ms` первым.

</details>

<details>
<summary>Ответ</summary>

```rust
//! Exercise 2: last write wins, with a tie-breaker.

/// A timestamp with the writer's node id as a tie-breaker.
/// Deriving `Ord` compares `ms` first and `node` only when `ms` is equal.
#[derive(Debug, Clone, Copy, PartialEq, Eq, PartialOrd, Ord)]
pub struct Stamp {
    pub ms: u64,
    pub node: u32,
}

#[derive(Debug, Default, PartialEq)]
pub struct Register {
    current: Option<(Stamp, String)>,
}

impl Register {
    pub fn write(&mut self, value: &str, stamp: Stamp) {
        let newer = match &self.current {
            None => true,
            Some((old, _)) => stamp > *old,
        };
        if newer {
            self.current = Some((stamp, value.to_string()));
        }
    }

    pub fn read(&self) -> Option<&str> {
        self.current.as_ref().map(|(_, value)| value.as_str())
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn the_order_of_arrival_does_not_matter() {
        let w1 = ("from node 1", Stamp { ms: 500, node: 1 });
        let w2 = ("from node 2", Stamp { ms: 500, node: 2 }); // the same millisecond

        let mut one_way = Register::default();
        one_way.write(w1.0, w1.1);
        one_way.write(w2.0, w2.1);

        let mut other_way = Register::default();
        other_way.write(w2.0, w2.1);
        other_way.write(w1.0, w1.1);

        assert_eq!(one_way, other_way);
        assert_eq!(one_way.read(), Some("from node 2"));
    }
}
```

Почему это работает: с правилом разрешения ничьих любые две метки от разных узлов различаются, поэтому все реплики
выбирают одного и того же победителя при любом порядке прихода. Теперь реплики **согласны** — но проблема
рассинхронизации часов из раздела 4 никуда не делась: они могут согласиться на неверном значении.

</details>

### Упражнение 3 — Время с погрешностью (сложно)

Если известно, что часы ошибаются не больше чем на `max_error` мс, показание `t` на самом деле означает «где-то между
`t - max_error` и `t + max_error`». Добавьте в `src/drift.rs` тип `Interval { earliest_ms, latest_ms }` с
`Interval::around(reading_ms, max_error_ms)` и `surely_before(&self, other: &Interval) -> bool`, который истинен только
тогда, когда `self` произошло раньше `other` **при любых** настоящих ошибках. Проверьте три показания с ошибкой 5 мс:
1000, 1008 и 1020. Какие пары имеют определённый порядок?

<details>
<summary>Подсказка</summary>

Событие A точно раньше B, только если даже самое позднее возможное время A раньше самого раннего возможного времени
B. Если два интервала перекрываются, порядок неизвестен в обе стороны.

</details>

<details>
<summary>Ответ</summary>

```rust
/// A reading of a clock with a known maximum error: the real time is somewhere in between.
#[derive(Debug, Clone, Copy)]
pub struct Interval {
    pub earliest_ms: u64,
    pub latest_ms: u64,
}

impl Interval {
    /// A reading `reading_ms` from a clock that may be wrong by up to `max_error_ms`.
    pub fn around(reading_ms: u64, max_error_ms: u64) -> Self {
        Interval {
            earliest_ms: reading_ms.saturating_sub(max_error_ms),
            latest_ms: reading_ms + max_error_ms,
        }
    }

    /// True only if `self` certainly happened before `other`, whatever the real errors were.
    pub fn surely_before(&self, other: &Interval) -> bool {
        self.latest_ms < other.earliest_ms
    }
}
```

И в модуле тестов `src/drift.rs`:

```rust
    #[test]
    fn overlapping_readings_have_no_known_order() {
        let a = Interval::around(1000, 5); // 995..=1005
        let b = Interval::around(1008, 5); // 1003..=1013: overlaps with a
        let c = Interval::around(1020, 5); // 1015..=1025
        assert!(!a.surely_before(&b));
        assert!(!b.surely_before(&a));
        assert!(a.surely_before(&c));
        assert!(!c.surely_before(&a));
    }
```

`cargo test --lib`:

```text
test result: ok. 6 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

Почему это работает: 1000 и 1008 отстоят всего на 8 мс, меньше двух погрешностей, поэтому первым может быть любое.
1000 и 1020 отстоят на 20 мс, больше двух погрешностей, поэтому порядок определён. База данных Google Spanner
построена на этой идее: перед тем как сообщить о фиксации транзакции, она выжидает интервал погрешности.

</details>

## Вопросы для самопроверки

1. Нужно остановить операцию через 5 секунд. Какие часы вы используете и почему?

<details>
<summary>Ответ</summary>

Монотонные (`Instant` или таймеры Tokio). Настенные часы во время ожидания могут переставить вперёд или назад, и
тайм-аут сработает слишком рано или гораздо позже.

</details>

2. Часы узла A спешат на 30 мс. Узел A пишет в реальное время 0 мс, узел B (верные часы) — в реальное время 20 мс,
   правило — «побеждает последняя запись». Какое значение останется?

<details>
<summary>Ответ</summary>

Значение A: его метка 30 мс, у B — 20 мс. Более поздняя запись B потеряна.

</details>

3. Почему клиент не может защититься от записи с истёкшей арендой, проверив время прямо перед записью?

<details>
<summary>Ответ</summary>

Пауза может случиться сразу после проверки и до того, как запись дойдёт до хранилища (или запись может задержаться в
сети). Клиент не видит собственную паузу. Только хранилище, сравнивающее токены ограждения, может отклонить
устаревшую запись.

</details>

4. Что такое токен ограждения и кто его проверяет?

<details>
<summary>Ответ</summary>

Число, которое растёт с каждой новой арендой. Владелец блокировки отправляет его с каждой записью, а **хранилище**
отклоняет записи, токен которых старше самого нового уже увиденного.

</details>

## Итоги

- У каждого компьютера есть настенные часы (`SystemTime`: время суток, могут прыгать) и монотонные часы (`Instant`:
  только вперёд, только на одной машине).
- Используйте `Instant` для тайм-аутов и длительностей, `SystemTime` в UTC — для меток времени, понятных людям.
- Часы уходят на десятки ppm, а NTP в лучшем случае держит машины в пределах миллисекунд друг от друга. Метки времени
  разных машин нельзя точно сравнить.
- «Побеждает последняя запись» с метками настенных часов незаметно теряет записи, когда часы рассинхронизированы.
- Аренда — это обещание о времени, которое приостановленный процесс может нарушить, сам того не заметив. Токены
  ограждения, которые проверяет хранилище, делают аренду безопасной.
- Чтобы упорядочить события на разных машинах, используйте логические часы (следующий урок) или консенсус, а не
  настенные часы.

## Источники и что почитать

- [How to do distributed locking](https://martin.kleppmann.com/2016/02/08/how-to-do-distributed-locking.html) — эссе Martin Kleppmann об аренде, паузах процессов и токенах ограждения; хронология в разделе 5 следует его рассуждению.
- [Designing Data-Intensive Applications](https://dataintensive.net/) — глава «The Trouble with Distributed Systems» разбирает ненадёжные часы, «побеждает последняя запись» и паузы процессов.
- [`std::time::Instant`](https://doc.rust-lang.org/std/time/struct.Instant.html) и [`std::time::SystemTime`](https://doc.rust-lang.org/std/time/struct.SystemTime.html) — документация точно объясняет, что гарантирует каждый вид часов в каждой операционной системе.
- [RFC 5905 — Network Time Protocol Version 4](https://datatracker.ietf.org/doc/html/rfc5905) — стандарт NTP (RFC, Request for Comments, — документ, определяющий интернет-стандарт) для тех, кто хочет увидеть, как оцениваются смещение часов и сетевая задержка.

---

[🏠 Начало](../START-HERE.md) · [← 18.3 Повторные попытки, задержки и идемпотентность](03-retries-backoff-and-idempotency.md) · [📋 Этап 18](README.md) · [18.5 Логические часы: часы Лэмпорта и векторные часы →](05-logical-clocks.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
