<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 22.10 Трек встраиваемых систем: микроконтроллеры и no_std](10-embedded-track-microcontrollers-and-no-std.md) · [📋 Этап 22](README.md) · [22.12 Трек встраиваемых систем: асинхронность с Embassy →](12-embedded-track-async-with-embassy.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/22-specializations/11-embedded-track-gpio-and-embedded-hal.md) <!-- nav -->

# Урок 22.11 — Трек встраиваемых систем: GPIO и embedded-hal

> **Этап 22 — Специализации** · Урок 11 из 30 · ⏱ около 2,5 часа (2 занятия)

## Что вы узнаете

- Что такое выводы GPIO (General-Purpose Input/Output, ввод-вывод общего назначения), как безопасно подключить светодиод (LED, Light-Emitting Diode) и кнопку и что делают подтягивающие резисторы.
- Что делает за вас HAL (Hardware Abstraction Layer, слой абстракции оборудования) — на примере переключения вывода одной «сырой» записью в регистр.
- Трейты `embedded-hal` 1.0 `OutputPin`, `InputPin` и `DelayNs` и почему драйверы пишут на их основе.
- Как один раз написать обобщённый драйвер и протестировать его на Mac с фейковыми выводами.
- Как владение в Rust не даёт двум частям прошивки использовать один и тот же вывод.

## Перед началом

- Вы прошли [Урок 22.10 — Трек встраиваемых систем: микроконтроллеры и no_std](10-embedded-track-microcontrollers-and-no-std.md)
  и умеете собирать `code/p22-hello-led`.
- Вы знаете трейты и обобщения ([Уроки 05.1–05.2](../05-rust-intermediate/02-traits.md)), ассоциированные
  типы ([Урок 06.2](../06-rust-advanced/02-advanced-traits-and-types.md)) и тестовые двойники
  ([Урок 09.3](../09-testing-debugging-benchmarking/03-test-doubles.md)).
- Оборудование (по желанию): Pico 2 с Debug Probe, макетная плата, кнопка и соединительные провода.

## Новые слова

| Слово | Значение |
|---|---|
| **GPIO** (General-Purpose Input/Output, ввод-вывод общего назначения) | Вывод, на который программа может подать высокий или низкий уровень (выход) или прочитать его (вход). |
| **высокий / низкий уровень** (high / low) | Два уровня цифрового вывода. На RP2350 высокий — около 3,3 вольта, низкий — около 0 вольт. |
| **LED** (Light-Emitting Diode, светодиод) | Маленькая лампочка, которая светится, когда ток идёт от длинной ножки (анода) к короткой (катоду). |
| **резистор** (resistor) | Деталь, ограничивающая ток. Её номинал измеряется в омах (Ом). |
| **подтягивающий резистор** (pull-up resistor) | Резистор, соединяющий вход с высоким уровнем, чтобы вход читал «высокий», когда его больше ничто не задаёт. В RP2350 есть встроенные. |
| **активный низкий уровень** (active low) | Сигнал, который означает «включено», когда он низкий. Кнопка на землю с подтяжкой вверх — активная по низкому уровню. |
| **дребезг** (bounce) | Быстрое случайное «включено-выключено» механического контакта кнопки в первые миллисекунды нажатия. |
| **PAC** (Peripheral Access Crate, крейт доступа к периферии) | Крейт, сгенерированный из описания регистров чипа. Даёт по типу на регистр, но без высокоуровневой логики. |
| **`embedded-hal`** | Крейт трейтов (выходной вывод, входной вывод, задержка и шины вроде I²C (Inter-Integrated Circuit) и SPI (Serial Peripheral Interface) — два распространённых способа общаться с микросхемами датчиков), которые реализуют HAL и используют драйверы. |
| **драйвер** (driver) | Код, который управляет одним устройством (узор мигания, датчик, дисплей) через трейты `embedded-hal`. |
| **volatile-доступ** (volatile access) | Чтение или запись памяти, которые компилятор не имеет права удалять, объединять или переставлять. Нужен для аппаратных регистров. |

## 1. Выводы GPIO, светодиоды и кнопки

Вывод **GPIO** — это крошечный переключатель под управлением программы. Как **выход** программа
соединяет его либо с 3,3 вольта (**высокий уровень**), либо с 0 вольт (**низкий уровень**). Как **вход**
программа читает, какой из двух уровней сейчас на проводе.

**Светодиоду нужен резистор.** Как только напряжение на светодиоде достигает его порога (около 2 вольт
для красного), он пропускает почти неограниченный ток. Слишком большой ток сжигает светодиод или вывод.
Последовательный резистор его ограничивает. Номинал даёт закон Ома: *ток = напряжение / сопротивление*.
С 3,3 В на выводе и 2 В на светодиоде на резистор остаётся 1,3 В; 1,3 В / 330 Ом ≈ 4 мА — безопасный ток
для вывода RP2350.

```text
Pico 2 pin 21 (GPIO 16) ---[ 330 Ω ]---(long leg) LED (short leg)--- pin 23 (GND)

Pico 2 pin 20 (GPIO 15) ---[ button ]--- pin 18 (GND)
```

GND (ground) — это земля, сторона цепи с 0 вольт. У зелёного светодиода платы на GPIO 25 резистор уже
есть, поэтому урок использует его, и подключить нужно только кнопку.

**Кнопке нужен определённый уровень.** Когда кнопка на схеме выше разомкнута, GPIO 15 ни к чему не
подключён. Вход, подключённый к пустоте, «плавает»: ловит электрические помехи и читает высокий или
низкий уровень случайно. Решение — **подтягивающий резистор** внутри чипа, который слабо соединяет вывод с 3,3 В:

```text
          3.3 V
            |
         [pull-up]        button open:    pin reads HIGH
            |             button pressed: pin is tied to GND, reads LOW
GPIO 15 ----+----[ button ]---- GND
```

Значит, кнопка **активна по низкому уровню**: нажата — значит низкий. Подтяжку вверх вы выбираете в коде через `Pull::Up`.

> ⚠️ **Частая ошибка:** никогда не соединяйте выходной вывод напрямую с землёй или с 3,3 В и никогда не
> соединяйте два выходных вывода между собой. Если один выдаёт высокий уровень, а другой низкий, течёт
> большой ток, который может повредить чип. Используйте резистор или настройте вывод как вход.

## 2. Что делает за вас HAL: одна «сырая» запись в регистр

В Уроке 22.10 вы видели, что периферия — это регистры по фиксированным адресам. В RP2350 есть блок
**SIO** (Single-cycle I/O, однотактный ввод-вывод; I/O — input/output, ввод-вывод) по адресу
`0xd000_0000`. Один из его регистров, `GPIO_OUT_XOR` по смещению `0x28`, переключает каждый выход, бит
которого вы записали единицей. Чтобы переключить GPIO 25, запишите `1 << 25`.

Сделаем это один раз вручную. Прошивка живёт в пакете `code/p22-gpio/firmware`; раздел 4 создаёт его и
объясняет его файлы, поэтому пока просто прочитайте код. Он кладётся в
`code/p22-gpio/firmware/src/bin/raw.rs` — второй бинарник рядом с `src/main.rs`:

```rust
#![no_std]
#![no_main]

use core::panic::PanicInfo;
use cortex_m_rt::entry;
use embassy_rp::gpio::{Level, Output};

// RP2350 datasheet: the SIO block starts at 0xd000_0000, and the
// GPIO_OUT_XOR register at offset 0x28 flips every output whose bit is 1.
const SIO_GPIO_OUT_XOR: *mut u32 = 0xd000_0028 as *mut u32;

#[entry]
fn main() -> ! {
    let p = embassy_rp::init(Default::default());
    // Let the HAL do the setup: pad, pin function, output enable.
    let _led = Output::new(p.PIN_25, Level::Low);

    loop {
        // SAFETY: SIO_GPIO_OUT_XOR is a valid register of the RP2350 that is
        // always mapped, and writing a bit mask only flips those output pins.
        unsafe { core::ptr::write_volatile(SIO_GPIO_OUT_XOR, 1 << 25) };
        cortex_m::asm::delay(75_000_000); // about 0.5 s at 150 MHz
    }
}

#[panic_handler]
fn panic(_info: &PanicInfo) -> ! {
    loop {}
}
```

**Что происходит:**

1. `SIO_GPIO_OUT_XOR` — сырой указатель на регистр. Создать сырой указатель безопасно, использовать — нет.
2. `Output::new` по-прежнему делает настройку: подключает вывод к блоку SIO, включает выходной драйвер и
   настраивает электрическую «площадку» (pad). Вручную это ещё три регистра в двух других блоках периферии.
3. `write_volatile` записывает битовую маску. **Volatile** говорит компилятору: «у этой записи есть эффект
   вне программы, не удаляй и не объединяй её». Обычную запись в память, которую потом никто не читает,
   компилятор мог бы выбросить при оптимизации.
4. `cortex_m::asm::delay(75_000_000)` сжигает не меньше 75 миллионов тактов процессора (CPU, Central Processing Unit) — около половины
   секунды на 150 МГц.

```bash
cargo build --release --bin raw
```

```text
    Finished `release` profile [optimized + debuginfo] target(s) in 0.81s
```

На плате (`cargo run --release --bin raw`) зелёный светодиод мигает раз в секунду. Это работает, но
посмотрите, что вам пришлось знать: адрес, смещение, номер бита и то, что два других блока уже настроены.
Одна неверная цифра — и запись уйдёт в другое периферийное устройство, а компилятор ничем не поможет.

Мир встраиваемого Rust решает это слоями:

```text
+------------------------------------------------------+
| your application (firmware main)                     |
+------------------------------------------------------+
| drivers: LED patterns, sensors, displays             |  use only embedded-hal traits
+------------------------------------------------------+
| embedded-hal traits: OutputPin, InputPin, DelayNs... |  the shared contract
+------------------------------------------------------+
| HAL: embassy-rp (Output, Input, Delay...)            |  safe types, implements the traits
+------------------------------------------------------+
| PAC: rp-pac (one type per register)                  |  generated from the chip's description
+------------------------------------------------------+
| hardware registers at fixed addresses                |
+------------------------------------------------------+
```

> 🔒 **Безопасность:** каждая сырая запись в регистр — `unsafe` не просто так: компилятор не может
> проверить адрес. Держите такой код в одном маленьком проверенном модуле с комментарием `// SAFETY:`,
> точно как в [Уроке 10.12](../10-security-fundamentals/12-memory-safety-and-reviewing-unsafe.md).
> Код приложения должен пользоваться только HAL.

## 3. Трейты `embedded-hal`

`embedded-hal` 1.0 — маленький крейт, в котором только **трейты**. Каждый HAL реализует их для своих
типов, а каждый **драйвер** обобщён по ним. Три трейта, которые нужны сейчас:

```rust
// From embedded-hal 1.0 (simplified)
pub trait ErrorType {
    type Error: core::fmt::Debug;
}

pub trait OutputPin: ErrorType {
    fn set_low(&mut self) -> Result<(), Self::Error>;
    fn set_high(&mut self) -> Result<(), Self::Error>;
}

pub trait InputPin: ErrorType {
    fn is_high(&mut self) -> Result<bool, Self::Error>;
    fn is_low(&mut self) -> Result<bool, Self::Error>;
}

pub trait DelayNs {
    fn delay_ns(&mut self, ns: u32);
    fn delay_us(&mut self, us: u32) { /* default: calls delay_ns */ }
    fn delay_ms(&mut self, ms: u32) { /* default: calls delay_us */ }
}
```

**Что происходит:**

1. Каждая операция с выводом возвращает `Result`, потому что некоторые выводы могут дать сбой: вывод
   микросхемы-расширителя ввода-вывода (I/O expander) задаётся по шине, а шина может отказать. Для
   собственных выводов RP2350 тип ошибки — `Infallible`, перечисление без значений, поэтому операция
   никогда не завершается ошибкой.
2. `ErrorType` один раз хранит ассоциированный тип `Error`, и `OutputPin` с `InputPin` используют его вместе.
3. `DelayNs` требует только `delay_ns`; у `delay_us` и `delay_ms` есть методы по умолчанию на его основе.

Теперь драйвер. Создайте библиотечный пакет рядом с прошивкой:

```bash
cd ~/projects/rust
mkdir code/p22-gpio
cargo new --lib code/p22-gpio/driver --name p22-gpio-driver
cd code/p22-gpio/driver
cargo add embedded-hal@1
```

`code/p22-gpio/driver/src/lib.rs` (тесты — в разделе 4):

```rust
#![no_std]

use embedded_hal::delay::DelayNs;
use embedded_hal::digital::OutputPin;

/// Blinks `led` `times` times: on for `ms` milliseconds, then off for `ms`.
pub fn blink<P, D>(led: &mut P, delay: &mut D, times: u32, ms: u32) -> Result<(), P::Error>
where
    P: OutputPin,
    D: DelayNs,
{
    for _ in 0..times {
        led.set_high()?;
        delay.delay_ms(ms);
        led.set_low()?;
        delay.delay_ms(ms);
    }
    Ok(())
}
```

> 📏 **Соглашение:** драйвер принимает нужные ему выводы и задержки как **обобщённые параметры** с
> ограничениями трейтов `embedded-hal` и никогда не называет типы HAL. Только `main` прошивки знает, на каком
> чипе она работает. Опубликованные крейты драйверов обычно называют по имени устройства, например крейт для
> микросхемы датчика.

`blink` ничего не знает об RP2350. Функция работает с любым типом, который является выходным выводом, и с
любым типом, который умеет ждать: с выводом RP2350, выводом ESP32, выводом микросхемы-расширителя или
фейковым выводом в тесте.

## 4. Тестируем драйвер с фейковыми выводами

**Фейк** (Урок 09.3) — простая работающая замена. Фейковый вывод запоминает уровни, которые ему
выставляли; фейковая задержка складывает время вместо ожидания. Добавьте этот модуль тестов в конец `src/lib.rs`:

```rust
#[cfg(test)]
mod tests {
    extern crate std;

    use super::*;
    use core::convert::Infallible;
    use embedded_hal::digital::ErrorType;
    use std::vec::Vec;

    /// A fake pin that remembers every level it was set to.
    #[derive(Default)]
    struct FakePin {
        levels: Vec<bool>,
    }

    impl ErrorType for FakePin {
        type Error = Infallible;
    }

    impl OutputPin for FakePin {
        fn set_high(&mut self) -> Result<(), Infallible> {
            self.levels.push(true);
            Ok(())
        }
        fn set_low(&mut self) -> Result<(), Infallible> {
            self.levels.push(false);
            Ok(())
        }
    }

    /// A fake delay that only adds up the time instead of waiting.
    #[derive(Default)]
    struct FakeDelay {
        total_ns: u64,
    }

    impl DelayNs for FakeDelay {
        fn delay_ns(&mut self, ns: u32) {
            self.total_ns += ns as u64;
        }
    }

    #[test]
    fn blinks_three_times() {
        let mut pin = FakePin::default();
        let mut delay = FakeDelay::default();
        blink(&mut pin, &mut delay, 3, 200).unwrap();
        assert_eq!(pin.levels, [true, false, true, false, true, false]);
        assert_eq!(delay.total_ns, 6 * 200 * 1_000_000);
    }
}
```

**Что происходит:**

1. `extern crate std;` внутри модуля тестов возвращает `std` **только для тестов**. Сама библиотека
   остаётся `no_std`; тесты выполняются на Mac, где `std` есть, поэтому в них можно использовать `Vec`.
2. `FakePin` реализует те же два трейта, что и настоящий вывод. `blink` не видит разницы.
3. `FakeDelay` реализует только `delay_ns`. Метод по умолчанию `delay_ms` вызывает его, поэтому 200 мс
   превращаются в сумме в 200 000 000 наносекунд, а тест проходит мгновенно.

```bash
cargo test
```

```text
running 1 test
test tests::blinks_three_times ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

> 🧪 **Тестирование:** это самый важный приём тестирования во встраиваемых системах. Пишите драйверы на
> трейтах `embedded-hal`, тестируйте их с фейками на хосте и оставляйте плате только короткую финальную
> проверку. Если нужно больше, крейт `embedded-hal-mock` предлагает готовые моки с ожиданиями.

**Используем драйвер на чипе.** Создайте пакет прошивки — второй пакет рядом с драйвером. Это копия
`code/p22-hello-led` под новым именем:

```bash
cd ~/projects/rust
cp -R code/p22-hello-led code/p22-gpio/firmware
cd code/p22-gpio/firmware
rm -rf target
```

```text
code/p22-gpio/
├── driver/      ← библиотека no_std, тестируется на Mac через `cargo test`
│   ├── Cargo.toml
│   └── src/lib.rs
└── firmware/    ← бинарник для чипа, собирается через `cargo build --release`
    ├── .cargo/config.toml
    ├── build.rs
    ├── memory.x
    ├── Cargo.toml
    └── src/
        ├── main.rs
        └── bin/raw.rs
```

Положите «сырой» пример из раздела 2 в `src/bin/raw.rs` (папку создайте командой `mkdir src/bin`).
Замените `code/p22-gpio/firmware/Cargo.toml`:

```toml
[package]
name = "p22-gpio-firmware"
version = "0.1.0"
edition = "2024"
default-run = "p22-gpio-firmware"

[dependencies]
p22-gpio-driver = { path = "../driver" }
cortex-m = "0.7"
cortex-m-rt = "0.7"
embassy-rp = { version = "0.10", features = ["rp235xa", "critical-section-impl", "time-driver"] }
embassy-time = "0.5"
embedded-hal = "1"

[profile.release]
debug = true
```

Новое здесь: `default-run` говорит `cargo run`, какой из двух бинарников запускать, зависимость по пути
указывает на драйвер, а фича `time-driver` позволяет `embassy-time` читать аппаратный таймер чипа.
`code/p22-gpio/firmware/src/main.rs`:

```rust
#![no_std]
#![no_main]

use core::panic::PanicInfo;
use cortex_m_rt::entry;
use embassy_rp::gpio::{Input, Level, Output, Pull};
use embassy_time::Delay;
use embedded_hal::delay::DelayNs;
use p22_gpio_driver::blink;

#[entry]
fn main() -> ! {
    let p = embassy_rp::init(Default::default());
    let mut led = Output::new(p.PIN_25, Level::Low);
    // The button connects GPIO 15 to ground, so pressed = low.
    let button = Input::new(p.PIN_15, Pull::Up);
    let mut delay = Delay;

    loop {
        if button.is_low() {
            blink(&mut led, &mut delay, 3, 100).unwrap();
        } else {
            delay.delay_ms(10);
        }
    }
}

#[panic_handler]
fn panic(_info: &PanicInfo) -> ! {
    loop {}
}
```

**Что происходит:**

1. `Input::new(p.PIN_15, Pull::Up)` делает GPIO 15 входом с включённой внутренней подтяжкой вверх.
2. `embassy_time::Delay` реализует `DelayNs` через аппаратный таймер, а `Output` реализует `OutputPin`,
   поэтому оба подходят для `blink`.
3. Каждые 10 мс цикл **опрашивает** (polls) кнопку. Пока она нажата, светодиод быстро мигает три раза.
   Это называется **суперцикл** (super loop): один бесконечный цикл, который по очереди проверяет всё.
4. `.unwrap()` здесь безопасен, потому что тип ошибки — `Infallible`.

```bash
cargo build --release
cargo size --release
```

```text
    Finished `release` profile [optimized + debuginfo] target(s) in 7.44s
   text	   data	    bss	    dec	    hex	filename
  10932	      0	    304	  11236	   2be4	p22-gpio-firmware
```

Пример поведения на плате (`cargo run --release`): ничего не происходит, пока вы не нажмёте кнопку;
пока вы её держите, зелёный светодиод мигает сериями по три.

> ⚡ **Производительность:** суперцикл никогда не спит; процессор работает на полной скорости, даже когда
> ничего не происходит, а долгий `blink` блокирует проверку кнопки. Урок 22.12 заменяет это задачами
> `async`, которые спят, пока их не разбудит кнопка или таймер.

## 5. Владение выводами

`embassy_rp::init` возвращает каждый вывод ровно один раз, как значение собственного типа (`PIN_15`,
`PIN_25`, …). Передача вывода в `Output::new` его перемещает. А что, если вторая часть программы
попробует использовать тот же вывод как вход? Замените строку кнопки на `p.PIN_25`:

```text
error[E0382]: use of moved value: `p.PIN_25`
  --> src/main.rs:16:29
   |
14 |     let mut led = Output::new(p.PIN_25, Level::Low);
   |                               -------- value moved here
15 |     // The button connects GPIO 15 to ground, so pressed = low.
16 |     let button = Input::new(p.PIN_25, Pull::Up);
   |                             ^^^^^^^^ value used here after move
   |
   = note: move occurs because `p.PIN_25` has type `Peri<'_, PIN_25>`, which does not implement the `Copy` trait
```

Это правило владения из [Урока 03.2](../03-ownership-and-borrowing/02-ownership-rules.md), работающее на
оборудовании: два драйвера, которые дерутся за один вывод, — настоящий класс ошибок в прошивках, а здесь
это ошибка компиляции. Тип `Peri<'_, PIN_25>` — обёртка Embassy для «права пользоваться выводом 25».

## Частые ошибки

> ⚠️ **Частая ошибка: нет драйвера времени.** Без фичи `time-driver` у `embassy-rp` типу
> `embassy_time::Delay` неоткуда читать время, и компоновка падает:
>
> ```text
>   = note: rust-lld: error: undefined symbol: _embassy_time_now
>           >>> referenced by lib.rs:149 (.../embassy-time-driver-0.2.2/src/lib.rs:149)
> ```
>
> Исправление: добавьте `"time-driver"` в фичи `embassy-rp`.

> ⚠️ **Частая ошибка: `cargo test` в папке прошивки.** `.cargo/config.toml` прошивки заставляет каждую
> сборку использовать платформу чипа, а у чипа нет тестовой обвязки:
>
> ```text
> error[E0463]: can't find crate for `test`
> ```
>
> Исправление: запускайте тесты в `code/p22-gpio/driver`, который собирается для Mac. Держите логику в драйвере.

> ⚠️ **Частая ошибка: «плавающий» вход.** `Input::new(p.PIN_15, Pull::None)` с разомкнутой кнопкой читает
> случайные значения; светодиод мигает «сам», когда вы касаетесь проводов. Исправление: `Pull::Up`
> (кнопка на землю) или `Pull::Down` (кнопка на 3,3 В).

## Попробуйте сами

1. В `code/p22-gpio/driver` измените тест так, чтобы `blink` вызывалась с `times = 0`. Что вы ожидаете в
   `pin.levels`? Запустите `cargo test` и проверьте (пустой список, и `total_ns` равно 0).
2. В прошивке замените `blink(&mut led, &mut delay, 3, 100)` на пять медленных миганий по 300 мс и
   соберите снова.

## Упражнения

### Упражнение 1 — Подберите резистор (легко)

Зелёному светодиоду нужно около 2,2 В. Вы хотите не больше 3 мА от вывода на 3,3 В. Какой номинал
резистора нужен как минимум? Подойдёт ли 330 Ом?

<details>
<summary>Подсказка</summary>

На резистор приходится остаток напряжения: 3,3 − 2,2 вольта. Затем *R = V / I*.

</details>

<details>
<summary>Ответ</summary>

1,1 В / 0,003 А ≈ 367 Ом, значит нужно не меньше примерно 370 Ом; ближайший распространённый номинал —
390 Ом. С 330 Ом ток будет 1,1 / 330 ≈ 3,3 мА — чуть выше вашего предела, но для вывода всё ещё
безопасно. Резистор побольше лишь делает светодиод немного тусклее.

</details>

### Упражнение 2 — Обобщённое переключение с тестом (средне)

В драйвере напишите `pub fn toggle_n<P: StatefulOutputPin>(led: &mut P, n: u32) -> Result<(), P::Error>`,
которая вызывает `toggle` `n` раз. **Напишите тест** с фейковым выводом, который начинает с низкого
уровня, и проверьте, что после 3 переключений он высокий. (`StatefulOutputPin` требует `is_set_high` и
`is_set_low`; у `toggle` есть реализация по умолчанию.)

<details>
<summary>Подсказка</summary>

Дайте фейковому выводу поле `state: bool`. `set_high` ставит его в `true`, `is_set_high` возвращает
`Ok(self.state)`. Импортируйте `embedded_hal::digital::StatefulOutputPin`.

</details>

<details>
<summary>Ответ</summary>

```rust
use embedded_hal::digital::StatefulOutputPin;

pub fn toggle_n<P: StatefulOutputPin>(led: &mut P, n: u32) -> Result<(), P::Error> {
    for _ in 0..n {
        led.toggle()?;
    }
    Ok(())
}

#[cfg(test)]
mod toggle_tests {
    use super::*;
    use core::convert::Infallible;
    use embedded_hal::digital::ErrorType;

    struct StatePin {
        state: bool,
    }

    impl ErrorType for StatePin {
        type Error = Infallible;
    }

    impl OutputPin for StatePin {
        fn set_high(&mut self) -> Result<(), Infallible> {
            self.state = true;
            Ok(())
        }
        fn set_low(&mut self) -> Result<(), Infallible> {
            self.state = false;
            Ok(())
        }
    }

    impl StatefulOutputPin for StatePin {
        fn is_set_high(&mut self) -> Result<bool, Infallible> {
            Ok(self.state)
        }
        fn is_set_low(&mut self) -> Result<bool, Infallible> {
            Ok(!self.state)
        }
    }

    #[test]
    fn three_toggles_from_low_end_high() {
        let mut pin = StatePin { state: false };
        toggle_n(&mut pin, 3).unwrap();
        assert!(pin.state);
    }
}
```

Почему это работает: `toggle` по умолчанию читает `is_set_low` и вызывает `set_high` или `set_low`.
Низкий → высокий → низкий → высокий после трёх вызовов. Этому фейку не нужен `Vec`, поэтому модулю тестов
не нужен `extern crate std`.

</details>

### Упражнение 3 — Подавитель дребезга (сложно)

Нажатая кнопка дребезжит несколько миллисекунд. Напишите в драйвере `no_std`-тип `Debouncer`, которому вы
каждую миллисекунду передаёте одно сырое значение через `update(pressed: bool) -> Option<bool>`. Он
возвращает `Some(new_state)` только тогда, когда сырое значение было одинаковым 5 отсчётов подряд **и**
отличается от последнего сообщённого состояния; иначе `None`. **Напишите тесты**: чистое нажатие,
дребезжащее нажатие (`true, false, true, true, true, true, true`) и короткую помеху (два значения
`true`), которую нужно проигнорировать.

<details>
<summary>Подсказка</summary>

Храните три поля: последнее сообщённое состояние `state`, последнее сырое значение `candidate` и счётчик
`count` — сколько отсчётов подряд равны `candidate`. Сбрасывайте `count` в 1, когда сырое значение меняется.

</details>

<details>
<summary>Ответ</summary>

```rust
pub struct Debouncer {
    state: bool,
    candidate: bool,
    count: u8,
}

impl Debouncer {
    pub const STABLE: u8 = 5;

    pub const fn new() -> Self {
        Self {
            state: false,
            candidate: false,
            count: 0,
        }
    }

    pub fn update(&mut self, pressed: bool) -> Option<bool> {
        if pressed == self.candidate {
            self.count = self.count.saturating_add(1);
        } else {
            self.candidate = pressed;
            self.count = 1;
        }
        if self.count >= Self::STABLE && self.candidate != self.state {
            self.state = self.candidate;
            return Some(self.state);
        }
        None
    }
}

impl Default for Debouncer {
    fn default() -> Self {
        Self::new()
    }
}

#[cfg(test)]
mod debounce_tests {
    use super::*;

    fn feed(d: &mut Debouncer, samples: &[bool]) -> Option<bool> {
        let mut last = None;
        for &s in samples {
            if let Some(v) = d.update(s) {
                last = Some(v);
            }
        }
        last
    }

    #[test]
    fn clean_press_is_reported_once() {
        let mut d = Debouncer::new();
        assert_eq!(feed(&mut d, &[true; 5]), Some(true));
        assert_eq!(feed(&mut d, &[true; 5]), None);
    }

    #[test]
    fn bouncing_press_is_reported_after_it_settles() {
        let mut d = Debouncer::new();
        assert_eq!(
            feed(&mut d, &[true, false, true, true, true, true, true]),
            Some(true)
        );
    }

    #[test]
    fn short_glitch_is_ignored() {
        let mut d = Debouncer::new();
        assert_eq!(feed(&mut d, &[true, true, false, false, false]), None);
    }
}
```

Почему это работает: значение сообщается только после того, как оно продержалось `STABLE` отсчётов, поэтому
дребезг (короткие изменения) заново запускает счёт. `saturating_add` не даёт `count` переполниться, когда
кнопку держат долго. `const fn new` позволяет сделать `static`-подавитель в прошивке, а `Default` — то,
что Clippy ожидает рядом с `new` без аргументов.

</details>

## Вопросы для самопроверки

1. Зачем входу кнопки подтягивающий (вверх или вниз) резистор?

<details>
<summary>Ответ</summary>

Без него разомкнутая кнопка оставляет вывод ни к чему не подключённым, он «плавает» и читает случайные
уровни. Подтяжка вверх даёт определённый высокий уровень, когда кнопка разомкнута; нажатие соединяет
вывод с землёй (низкий уровень).

</details>

2. Что гарантирует `write_volatile` в отличие от обычной записи?

<details>
<summary>Ответ</summary>

Компилятор обязан выполнить запись ровно так, как написано: не удалять её, не объединять с другими
записями и не переносить за другие volatile-обращения. У аппаратных регистров есть эффекты вне программы,
поэтому обычные оптимизации их бы сломали.

</details>

3. Почему `OutputPin::set_high` возвращает `Result` и почему `blink` может использовать `?`?

<details>
<summary>Ответ</summary>

Некоторые выводы могут дать сбой (например, выводы микросхемы-расширителя за шиной). `blink` возвращает
`Result<(), P::Error>` — тот же тип ошибки, что у вывода, поэтому `?` передаёт ошибку вывода вызывающему коду.

</details>

4. Почему драйверы пишут на трейтах `embedded-hal`, а не на `embassy_rp::gpio::Output`?

<details>
<summary>Ответ</summary>

Чтобы один драйвер работал с любым HAL и на любом чипе и чтобы его можно было тестировать на хосте с
фейковыми выводами и задержками.

</details>

5. От чего защищает ошибка `E0382: use of moved value: p.PIN_25`?

<details>
<summary>Ответ</summary>

От того, что два драйвера одновременно используют один вывод (например, один как выход, другой как вход).
Каждый вывод — значение, которое можно переместить ровно в один драйвер.

</details>

## Итоги

- На выводе GPIO бывает высокий (3,3 В) или низкий (0 В) уровень; выходы его задают, входы читают.
- Светодиоду нужен последовательный резистор; кнопке — подтяжка вверх или вниз. Кнопка на землю с
  подтяжкой вверх активна по низкому уровню.
- Под HAL изменение вывода — это volatile-запись в регистр; HAL прячет адреса, биты и настройку.
- `embedded-hal` 1.0 определяет трейты `OutputPin`, `InputPin`, `StatefulOutputPin` и `DelayNs`;
  драйверы обобщены по ним.
- Тестируйте драйверы на Mac с фейками (`extern crate std;` в модуле тестов); прошивку держите тонкой.
- Каждый вывод — значение: перемещение в драйвер предотвращает двойное использование ещё при компиляции.

## Источники и что почитать

- [Документация `embedded-hal` 1.0](https://docs.rs/embedded-hal/1.0.0/embedded_hal/) — трейты и правила для драйверов и HAL.
- [The Embedded Rust Book — Static guarantees](https://docs.rust-embedded.org/book/static-guarantees/index.html) — как типы и владение моделируют периферию.
- [Модуль gpio в `embassy-rp`](https://docs.rs/embassy-rp/0.10.0/embassy_rp/gpio/index.html) — `Output`, `Input`, `Pull` и другое.
- [RP2350 datasheet](https://datasheets.raspberrypi.com/rp2350/rp2350-datasheet.pdf) — блок SIO и регистры GPIO (глава «Processor subsystem»).
- [`core::ptr::write_volatile`](https://doc.rust-lang.org/core/ptr/fn.write_volatile.html) — точные правила volatile-доступа.

---

[🏠 Начало](../START-HERE.md) · [← 22.10 Трек встраиваемых систем: микроконтроллеры и no_std](10-embedded-track-microcontrollers-and-no-std.md) · [📋 Этап 22](README.md) · [22.12 Трек встраиваемых систем: асинхронность с Embassy →](12-embedded-track-async-with-embassy.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
