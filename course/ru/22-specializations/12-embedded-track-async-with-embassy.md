<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 22.11 Трек встраиваемых систем: GPIO и embedded-hal](11-embedded-track-gpio-and-embedded-hal.md) · [📋 Этап 22](README.md) · [22.13 Трек встраиваемых систем: проект и следующие шаги →](13-embedded-track-project-and-next-steps.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/22-specializations/12-embedded-track-async-with-embassy.md) <!-- nav -->

# Урок 22.12 — Трек встраиваемых систем: асинхронность с Embassy

> **Этап 22 — Специализации** · Урок 12 из 30 · ⏱ около 105 минут

## Что вы узнаете

- Почему суперцикл из Урока 22.11 плохо масштабируется и что такое прерывания.
- Как Embassy запускает задачи `async` на чипе без операционной системы и без кучи.
- Как писать и запускать задачи, ждать таймеров и фронтов сигнала на выводах и передавать сообщения через `Channel`.
- Как вести логи с чипа через `defmt` и как ждать «того, что случится раньше» с помощью `select`.
- Как тестировать асинхронную логику прошивки на Mac.

## Перед началом

- Вы прошли [Урок 22.11 — Трек встраиваемых систем: GPIO и embedded-hal](11-embedded-track-gpio-and-embedded-hal.md).
- Вы знаете, как работают future, из [Урока 14.10 — Как работают future: Future, poll и Waker](../14-concurrency-and-async/10-how-futures-work.md)
  и написали исполнитель в [Уроке 14.11 — Пишем крошечный асинхронный рантайм](../14-concurrency-and-async/11-build-a-tiny-async-runtime.md).
  Каналы между задачами — в [Уроке 14.13](../14-concurrency-and-async/13-tokio-tasks-and-channels.md).
- Оборудование (по желанию): Pico 2, Debug Probe и кнопка на выводе GPIO (General-Purpose Input/Output, ввод-вывод общего назначения) 15 из Урока 22.11.

## Новые слова

| Слово | Значение |
|---|---|
| **прерывание** (interrupt) | Аппаратный сигнал, по которому процессор (CPU, Central Processing Unit) останавливает текущую работу, выполняет короткую функцию-обработчик и затем продолжает. |
| **обработчик прерывания** (interrupt handler) | Функция, которая выполняется для одного прерывания; её ещё называют ISR (Interrupt Service Routine, процедура обслуживания прерывания). |
| **фронт** (edge) | Момент изменения цифрового сигнала: **передний фронт** (rising edge) — низкий → высокий, **задний фронт** (falling edge) — высокий → низкий. |
| **Embassy** | Набор крейтов для асинхронного встраиваемого Rust: исполнитель, таймеры, синхронизация и HAL (Hardware Abstraction Layer, слой абстракции оборудования). |
| **задача** (task, в Embassy) | `async fn` с пометкой `#[embassy_executor::task]`, которая выполняется конкурентно с другими задачами. |
| **запустить** (spawn) | Передать задачу исполнителю, чтобы он её выполнял. |
| **`Channel`** | Очередь фиксированного размера из `embassy-sync`; одна задача отправляет значения, другая получает. |
| **критическая секция** (critical section) | Короткий участок кода, во время которого прерывания не выполняются, поэтому общие данные в безопасности. |
| **`defmt`** | «Deferred formatting», отложенное форматирование: крейт логирования, который отправляет с чипа крошечные двоичные сообщения и форматирует их на вашем компьютере. |
| **RTT** (Real-Time Transfer, передача в реальном времени) | Способ передавать данные из RAM (Random Access Memory, оперативная память) чипа на компьютер через отладочный адаптер без дополнительных проводов. |

## 1. Проблема суперцикла

Прошивка из Урока 22.11 опрашивала кнопку каждые 10 мс в одном бесконечном цикле. Теперь представьте, что
нужно три вещи одновременно: светодиод-«пульс» (LED, Light-Emitting Diode), который мигает каждые полсекунды, кнопка, которую нужно
заметить сразу, и сообщение в лог на каждое нажатие. В суперцикле каждый кусок кода приходится писать
маленькими шагами, которые никогда не ждут, и самим следить за всеми их состояниями. И процессор никогда
не спит, даже когда ничего не происходит.

У оборудования есть инструмент получше — **прерывания**. Периферийное устройство может поднять прерывание,
когда что-то произошло, например «на выводе 15 был задний фронт» или «таймер дошёл до этого значения».
Процессор останавливается, выполняет **обработчик прерывания** и продолжает. Между прерываниями
процессор может спать.

Писать всё вручную в виде обработчиков прерываний трудно: общее состояние, маленькие обработчики,
аккуратный порядок действий. Здесь помогает `async`. Идею вы уже знаете из Этапа 14:

- `async fn` превращается в **future** — конечный автомат, который компилятор строит за вас;
- **исполнитель** (executor) вызывает `poll` у future; future, который не может продолжить, возвращает `Pending`;
- **waker** говорит исполнителю «опроси этот future ещё раз».

Embassy связывает это с оборудованием: future, который ждёт вывод, регистрирует свой waker, обработчик
прерывания вывода вызывает `wake`, и исполнитель опрашивает только эту задачу. Когда ни одна задача не
может продолжить, исполнитель усыпляет процессор инструкцией `wfe` («wait for event», ждать события).

```text
  task A: heartbeat         task B: watch_button          main task
  ---------------           --------------------          ---------
  led.toggle()              wait_for_falling_edge()       PRESSES.receive()
  Timer 500 ms  -- Pending  -- Pending                    -- Pending
        |                         |                              |
        +------------ executor: nothing ready -> CPU sleeps -----+
        |                         |                              |
  timer interrupt -> wake A   pin interrupt -> wake B            |
  poll A: toggle again        poll B: send(1) ------------> wake main, log
```

**Embassy не нужна куча.** Каждая задача хранится в собственной `static`-памяти, размер которой известен
при компиляции. Если задачи не помещаются в RAM, об этом скажет компоновщик; у работающей программы память
для задач закончиться не может.

## 2. Прошивка с тремя задачами

Сделайте для этого урока копию проекта со светодиодом и добавьте новые крейты:

```bash
cd ~/projects/rust
cp -R code/p22-hello-led code/p22-embassy
cd code/p22-embassy
rm -rf target
```

`Cargo.toml` (измените имя и зависимости):

```toml
[package]
name = "p22-embassy"
version = "0.1.0"
edition = "2024"

[dependencies]
cortex-m = "0.7"
cortex-m-rt = "0.7"
defmt = "1"
defmt-rtt = "1"
embassy-executor = "0.10"
embassy-rp = { version = "0.10", features = ["rp235xa", "critical-section-impl", "time-driver", "executor-thread", "defmt"] }
embassy-sync = "0.8"
embassy-time = "0.5"
panic-probe = { version = "1", features = ["print-defmt"] }

[profile.release]
debug = true
```

`executor-thread` заставляет `embassy-rp` предоставить исполнитель для чипа, `defmt` включает поддержку
логов в HAL, а `panic-probe` — готовый обработчик паники, который печатает сообщение паники через адаптер
и останавливает программу. `defmt` нужен собственный скрипт компоновщика, поэтому добавьте одну строку в
конец `main` в `build.rs`:

```rust
    // defmt needs its own linker script for the log strings.
    println!("cargo:rustc-link-arg-bins=-Tdefmt.x");
```

и выберите уровень логов в `.cargo/config.toml` (добавьте в конец):

```toml
[env]
DEFMT_LOG = "info"
```

Теперь `src/main.rs`:

```rust
#![no_std]
#![no_main]

use defmt::info;
use embassy_executor::Spawner;
use embassy_rp::gpio::{Input, Level, Output, Pull};
use embassy_sync::blocking_mutex::raw::CriticalSectionRawMutex;
use embassy_sync::channel::Channel;
use embassy_time::Timer;
use {defmt_rtt as _, panic_probe as _};

/// Button presses travel from the button task to `main` through this channel.
static PRESSES: Channel<CriticalSectionRawMutex, u32, 4> = Channel::new();

#[embassy_executor::task]
async fn heartbeat(mut led: Output<'static>) {
    loop {
        led.toggle();
        Timer::after_millis(500).await;
    }
}

#[embassy_executor::task]
async fn watch_button(mut button: Input<'static>) {
    let mut count = 0;
    loop {
        button.wait_for_falling_edge().await;
        Timer::after_millis(20).await; // let the contact stop bouncing
        if button.is_low() {
            count += 1;
            PRESSES.send(count).await;
        }
        button.wait_for_high().await;
    }
}

#[embassy_executor::main(
    executor = "embassy_rp::executor::Executor",
    entry = "cortex_m_rt::entry"
)]
async fn main(spawner: Spawner) {
    let p = embassy_rp::init(Default::default());
    info!("started");

    spawner.spawn(heartbeat(Output::new(p.PIN_25, Level::Low)).unwrap());
    spawner.spawn(watch_button(Input::new(p.PIN_15, Pull::Up)).unwrap());

    loop {
        let n = PRESSES.receive().await;
        info!("button pressed {} times", n);
    }
}
```

**Что происходит:**

1. `use {defmt_rtt as _, panic_probe as _};` подключает два крейта, ничего из них не называя:
   `defmt-rtt` отправляет логи через RTT, а `panic-probe` даёт `#[panic_handler]`. Поэтому своего
   обработчика паники в файле больше нет.
2. `#[embassy_executor::main(...)]` превращает `async fn main` в настоящую точку входа: создаёт
   исполнитель RP2350, запускается через `cortex_m_rt::entry` и выполняет `main` как первую задачу.
   `Spawner` — это дескриптор для запуска других задач.
3. `#[embassy_executor::task]` превращает `async fn` в задачу. Вызов `heartbeat(led)` её не запускает;
   он возвращает `Result` с **токеном запуска** (spawn token) или ошибку, если эта задача уже работает
   (по умолчанию у каждой задачи место для одного экземпляра). `unwrap()` достаёт токен, а
   `spawner.spawn(...)` передаёт его исполнителю.
4. Аргументы задач должны быть `'static`: задача живёт вечно в своей статической памяти, поэтому не
   может заимствовать ничего со стека `main`. `Output<'static>` — драйвер вывода, который владеет своим
   выводом всё время работы программы.
5. `Timer::after_millis(500).await` возвращает `Pending`, пока не сработает аппаратный таймер. Тем временем
   выполняются другие задачи или процессор спит.
6. `button.wait_for_falling_edge().await` настраивает прерывание вывода на изменение высокий → низкий и
   спит, пока оно не придёт. Таймер на 20 мс после этого — простое подавление дребезга: короткий дребезг к
   тому времени заканчивается. `wait_for_high` ждёт отпускания, поэтому удержание кнопки считается одним нажатием.
7. `PRESSES` — `static`-канал `Channel`, вмещающий до 4 значений `u32`. `CriticalSectionRawMutex` защищает
   его **критической секцией** (прерывания ненадолго отключаются), поэтому задачи и обработчики
   прерываний могут пользоваться им безопасно. `send(...).await` ждёт, если канал полон; `receive().await`
   ждёт, если он пуст.

```bash
cargo build --release
cargo size --release
```

```text
    Finished `release` profile [optimized + debuginfo] target(s) in 5.50s
   text	   data	    bss	    dec	    hex	filename
  19052	     56	   1568	  20676	   50c4	p22-embassy
```

Около 19 КБ флеш-памяти и 1,6 КБ RAM на исполнитель, очередь таймеров, три задачи, канал и логирование.

> ⚡ **Производительность:** сравните с суперциклом. Там процессор работал всё время. Здесь после каждого
> шага исполнитель не находит работы и выполняет `wfe`: ядро спит до следующего прерывания. Для устройства
> на батарейке это разница между днями и месяцами.

## 3. Логи через defmt

`println!` на чипе нет, а форматирование текста медленное и занимает много флеш-памяти. `defmt`
(«deferred formatting», отложенное форматирование) решает это: строки формата остаются в ELF-файле (Executable and Linkable Format, формат исполняемых и компонуемых файлов) на Mac,
а чип отправляет только маленький номер (какая строка) и сырые аргументы. `probe-rs` читает их через
**RTT** (через отладочный адаптер) и форматирует на вашем компьютере.

Пример вывода `cargo run --release` с платой (нужно оборудование; у вас будет похожий). Нажмите кнопку два раза:

```text
      Erasing ✔ 100% [####################]  20.00 KiB @  60.12 KiB/s (took 0s)
  Programming ✔ 100% [####################]  20.00 KiB @  41.05 KiB/s (took 0s)
     Finished in 0.83s
INFO  started
└─ p22_embassy::____embassy_main_task::{async_fn#0} @ src/main.rs:43
INFO  button pressed 1 times
└─ p22_embassy::____embassy_main_task::{async_fn#0} @ src/main.rs:50
INFO  button pressed 2 times
└─ p22_embassy::____embassy_main_task::{async_fn#0} @ src/main.rs:50
```

`DEFMT_LOG = "info"` в `.cargo/config.toml` задаёт, какие уровни попадут в сборку (`trace`, `debug`,
`info`, `warn`, `error`). Уровни ниже него удаляются при компиляции и ничего не стоят.

> 💡 **Совет:** с `panic-probe` и `print-defmt` паника — например, `unwrap()` у `Err` — печатает сообщение,
> файл и строку, а затем останавливает программу. Это самый быстрый способ находить ошибки на плате.

## 4. Ждём то, что случится раньше: `select`

Часто задача должна ждать одного из нескольких событий: нажатия кнопки **или** тайм-аута. В крейте
`embassy-futures` есть `select`: он опрашивает два future и возвращается, как только один из них готов.
Добавьте его командой `cargo add embassy-futures@0.1` и измените конец `main`:

```rust
    loop {
        match select(PRESSES.receive(), Timer::after_secs(5)).await {
            Either::First(n) => info!("button pressed {} times", n),
            Either::Second(()) => info!("no press for 5 seconds"),
        }
    }
```

и добавьте `use embassy_futures::select::{Either, select};` в начало файла.

**Что происходит:**

1. `select(a, b)` сам является future. Его опрос опрашивает `a` и `b`.
2. Если канал первым выдал значение, результат — `Either::First(n)`; если раньше сработал таймер —
   `Either::Second(())`.
3. Проигравший future **уничтожается** (drop). Уничтожение `Timer` отменяет его; уничтожение `receive()`
   просто прекращает ожидание, и значение не теряется. Это отмена, как в
   [Уроке 14.14](../14-concurrency-and-async/14-cancellation-timeouts-and-graceful-shutdown.md).

Пример вывода после 5 секунд без нажатий: `INFO  no press for 5 seconds`.

## 5. Тестируем асинхронную логику на Mac

Логику задачи часто можно записать как обычную `async fn`, которая получает `Channel` (или трейт)
параметром. Тогда тест на Mac может выполнить её крошечным исполнителем: `embassy_futures::block_on`
опрашивает future в цикле, пока он не будет готов. `embassy-sync` нужна реализация критической секции; на
Mac её даёт крейт `critical-section` с фичей `std`. Упражнение 3 показывает всю настройку целиком.

> 🧪 **Тестирование:** держите «что» (логику: подсчёт, смены состояний, протоколы) в `async fn` с
> параметрами, а «как» (какой вывод, какой таймер) — в `main`. «Что» тестируйте на Mac с `block_on` и
> каналами; «как» проверяйте на плате по логам `defmt`.

## Частые ошибки

> ⚠️ **Частая ошибка: забыли `.await`.** `Timer::after_millis(500);` без `.await` создаёт future и
> выбрасывает его. Ничто не ждёт, и светодиод переключается так быстро, как может процессор. Компилятор предупреждает:
>
> ```text
> warning: unused `Timer` that must be used
>   --> src/main.rs:19:9
>    |
> 19 |         Timer::after_millis(500);
>    |         ^^^^^^^^^^^^^^^^^^^^^^^^
>    |
>    = note: futures do nothing unless you `.await` or poll them
> ```

> ⚠️ **Частая ошибка: заимствованный аргумент задачи.** Задача не может заимствовать у `main`:
>
> ```text
> error: Arguments for tasks must live forever. Try using the `'static` lifetime.
>   --> src/main.rs:16:36
>    |
> 16 | async fn heartbeat(mut led: Output<'_>) {
>    |                                    ^^
> ```
>
> Исправление: перемещайте в задачу значения, которыми она владеет (например `Output<'static>`), или
> делитесь данными через `static`-элементы, например `Channel`.

> ⚠️ **Частая ошибка: нет `defmt.x`.** Без строки `-Tdefmt.x` в `build.rs` компоновка падает:
>
> ```text
>   = note: rust-lld: error: undefined symbol: _defmt_timestamp
>           rust-lld: error: undefined symbol: _defmt_panic
> ```

> ⚠️ **Частая ошибка: блокировка внутри задачи.** `cortex_m::asm::delay(...)` или долгое вычисление
> внутри задачи никогда не возвращают `Pending`, поэтому **ни одна другая задача не выполняется**, пока они
> не закончатся, и кнопка остаётся незамеченной. В задачах всегда ждите через `.await` (например,
> `Timer::after_millis`).

## Попробуйте сами

1. Измените период «пульса» с 500 мс на 100 мс и соберите. На плате светодиод быстро мерцает, а кнопка
   по-прежнему срабатывает сразу: задачи не замедляют друг друга.
2. Поставьте `DEFMT_LOG = "warn"` и соберите снова. Строки `info!` удаляются, и `cargo size --release`
   показывает, что `text` уменьшился с 19052 до 18956 байт. Верните `"info"`.

## Упражнения

### Упражнение 1 — Двойное мигание (легко)

Измените `heartbeat` так, чтобы светодиод дважды быстро мигал (горит 100 мс, не горит 100 мс, два раза),
а затем 600 мс оставался выключенным, как сердцебиение. Используйте `set_high` и `set_low`.

<details>
<summary>Подсказка</summary>

Цикл `for _ in 0..2` внутри `loop` и ещё один `Timer` после него.

</details>

<details>
<summary>Ответ</summary>

```rust
#[embassy_executor::task]
async fn heartbeat(mut led: Output<'static>) {
    loop {
        for _ in 0..2 {
            led.set_high();
            Timer::after_millis(100).await;
            led.set_low();
            Timer::after_millis(100).await;
        }
        Timer::after_millis(600).await;
    }
}
```

Почему это работает: каждый `.await` возвращает процессор исполнителю, поэтому задача кнопки продолжает
работать во время узора. Один цикл длится 1 секунду.

</details>

### Упражнение 2 — Второй светодиод (средне)

Подключите светодиод с резистором 330 Ом к GPIO 16 (Урок 22.11, раздел 1). В `main` создайте
`Output::new(p.PIN_16, Level::Low)` и зажигайте его на 200 мс после каждого нажатия кнопки, пока «пульс»
продолжается на GPIO 25. Соберите командой `cargo build --release`.

<details>
<summary>Подсказка</summary>

`main` — тоже задача. После `PRESSES.receive().await` поставьте высокий уровень, подождите через `Timer`,
поставьте низкий.

</details>

<details>
<summary>Ответ</summary>

```rust
    let mut press_led = Output::new(p.PIN_16, Level::Low);
    loop {
        let n = PRESSES.receive().await;
        info!("button pressed {} times", n);
        press_led.set_high();
        Timer::after_millis(200).await;
        press_led.set_low();
    }
```

Почему это работает: `p.PIN_16` ещё не перемещён, поэтому `main` может им владеть. Пока `main` ждёт
200 мс, задачи «пульса» и кнопки продолжают работать. Нажатие в эти 200 мс подождёт в канале (он вмещает
4 значения) и будет записано в лог потом.

</details>

### Упражнение 3 — Тест логики задачи на Mac (сложно)

Создайте библиотеку `code/p22-presses` (`no_std`) с функцией
`pub async fn wait_for_presses<M: RawMutex, const N: usize>(presses: &Channel<M, u32, N>, goal: u32) -> u32`,
которая получает счётчики, пока один из них не станет не меньше `goal`, и возвращает его. **Напишите
тест**, который кладёт в канал 1, 2, 3, 4 через `try_send`, выполняет функцию через `block_on` с целью 3 и
проверяет, что она вернула 3, а 4 всё ещё лежит в канале.

<details>
<summary>Подсказка</summary>

Зависимости: `cargo add embassy-sync@0.8`, затем `cargo add --dev embassy-futures@0.1` и
`cargo add --dev critical-section@1 -F std` (фича `std` даёт критическую секцию для Mac; `-F` означает
`--features`). Благодаря обобщению по `M: RawMutex` функция работает с любым видом мьютекса.

</details>

<details>
<summary>Ответ</summary>

```rust
#![no_std]

use embassy_sync::blocking_mutex::raw::RawMutex;
use embassy_sync::channel::Channel;

/// Waits until the press counter reaches at least `goal` and returns that counter.
pub async fn wait_for_presses<M: RawMutex, const N: usize>(
    presses: &Channel<M, u32, N>,
    goal: u32,
) -> u32 {
    loop {
        let count = presses.receive().await;
        if count >= goal {
            return count;
        }
    }
}

#[cfg(test)]
mod tests {
    use super::*;
    use embassy_futures::block_on;
    use embassy_sync::blocking_mutex::raw::CriticalSectionRawMutex;

    #[test]
    fn stops_at_the_goal() {
        let ch: Channel<CriticalSectionRawMutex, u32, 4> = Channel::new();
        for n in 1..=4 {
            ch.try_send(n).unwrap();
        }
        let got = block_on(wait_for_presses(&ch, 3));
        assert_eq!(got, 3);
        // The fourth press is still waiting in the channel.
        assert_eq!(ch.try_receive().unwrap(), 4);
    }
}
```

```text
running 1 test
test tests::stops_at_the_goal ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

Почему это работает: `block_on` — минимальный исполнитель, который опрашивает future, пока тот не станет
`Ready`. Значения уже лежат в канале, поэтому `receive()` ждать не приходится. Ту же функцию без изменений
можно вызвать из задачи на чипе со `static`-каналом `PRESSES`.

</details>

## Вопросы для самопроверки

1. Что делает исполнитель Embassy, когда ни одна задача не может продолжить?

<details>
<summary>Ответ</summary>

Усыпляет процессор (`wfe`), пока прерывание не разбудит задачу. Затем опрашивает только разбуженную задачу.

</details>

2. Как `button.wait_for_falling_edge().await` узнаёт, когда продолжить?

<details>
<summary>Ответ</summary>

Future включает прерывание вывода по заднему фронту и сохраняет свой waker. Когда фронт происходит,
обработчик прерывания будит задачу, исполнитель опрашивает её снова, и теперь future готов (`Ready`).

</details>

3. Почему аргументы задач должны быть `'static`?

<details>
<summary>Ответ</summary>

Задача живёт в своей статической памяти и может работать вечно, дольше любого кадра стека функции.
Заимствованные у `main` данные могли бы исчезнуть, пока задача ещё ими пользуется.

</details>

4. Чем `defmt` лучше форматирования текста на чипе?

<details>
<summary>Ответ</summary>

Чип отправляет только номер строки формата и сырые аргументы; сами строки остаются на компьютере. Это
требует гораздо меньше флеш-памяти и процессорного времени, а неиспользуемые уровни логов удаляются при компиляции.

</details>

5. Что происходит с таймером в `select(PRESSES.receive(), Timer::after_secs(5))`, если нажатие пришло раньше?

<details>
<summary>Ответ</summary>

`select` возвращает `Either::First(n)` и уничтожает другой future. Уничтожение future таймера отменяет его.

</details>

## Итоги

- Суперцикл тратит энергию и усложняется; прерывания вместе с `async` позволяют задачам спать, пока их не
  разбудит оборудование.
- Исполнитель Embassy опрашивает задачи, спит через `wfe`, когда работы нет, и хранит каждую задачу
  статически: куча не нужна.
- Задачи — это `async fn` с `#[embassy_executor::task]`, запускаемые через
  `spawner.spawn(task(...).unwrap())`; их аргументы должны быть `'static`.
- `Timer::after_millis`, `wait_for_falling_edge` и `Channel::receive` — future, которые завершаются по
  аппаратным прерываниям.
- `defmt` ведёт логи через RTT и адаптер; `select` ждёт первый из двух future и отменяет второй.
- Тестируйте логику задач как обычные `async fn` на Mac с `block_on`.

## Источники и что почитать

- [The Embassy Book](https://embassy.dev/book/) — официальное руководство: исполнитель, задачи, время, HAL.
- [`embassy-executor` на docs.rs](https://docs.rs/embassy-executor/0.10.0/embassy_executor/) — задачи, `Spawner` и макрос `main`.
- [`embassy-sync` на docs.rs](https://docs.rs/embassy-sync/0.8.0/embassy_sync/) — `Channel`, `Signal`, `Mutex` и другое.
- [`embassy-time` на docs.rs](https://docs.rs/embassy-time/0.5.1/embassy_time/) — `Timer`, `Duration`, `Instant`.
- [The defmt book](https://defmt.ferrous-systems.com/) — как устроены отложенное форматирование и уровни логов.

---

[🏠 Начало](../START-HERE.md) · [← 22.11 Трек встраиваемых систем: GPIO и embedded-hal](11-embedded-track-gpio-and-embedded-hal.md) · [📋 Этап 22](README.md) · [22.13 Трек встраиваемых систем: проект и следующие шаги →](13-embedded-track-project-and-next-steps.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
