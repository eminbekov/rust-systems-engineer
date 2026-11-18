<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 22.12 Трек встраиваемых систем: асинхронность с Embassy](12-embedded-track-async-with-embassy.md) · [📋 Этап 22](README.md) · [22.14 Трек баз данных: как работают движки запросов →](14-databases-track-how-query-engines-work.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/22-specializations/13-embedded-track-project-and-next-steps.md) <!-- nav -->

# Урок 22.13 — Трек встраиваемых систем: проект и следующие шаги

> **Этап 22 — Специализации** · Урок 13 из 30 · ⏱ около 20 часов (10 занятий)

## Что вы узнаете

- Как спланировать прошивку как библиотеку `no_std`, протестированную на хосте, плюс тонкую прошивку на Embassy.
- Как устроены тайминги азбуки Морзе и как превратить текст в шаги «включено/выключено» без кучи.
- Как пользоваться асинхронным трейтом `DelayNs` из `embedded-hal-async` и как отменить идущий узор через `select`.
- Куда двигаться дальше во встраиваемом Rust: книги, проекты для вклада и сообщества.

## Перед началом

- Вы прошли [Урок 22.12 — Трек встраиваемых систем: асинхронность с Embassy](12-embedded-track-async-with-embassy.md)
  и умеете собирать `code/p22-embassy`.
- Вы помните фейковые выводы из [Урока 22.11](11-embedded-track-gpio-and-embedded-hal.md) и итераторы из
  [Урока 05.5](../05-rust-intermediate/05-iterators.md).
- Репозиторий Git в `code/p22-morse` (`git init`), по одному коммиту на веху.
- Оборудование не обязательно. Вехи M1–M4 проверяются на Mac; для финальной проверки M5 нужна плата.

## Новые слова

| Слово | Значение |
|---|---|
| **азбука Морзе** (Morse code) | Способ передавать буквы короткими и длинными сигналами: точкой (короткий) и тире (длинный). |
| **единица** (unit, в азбуке Морзе) | Длительность одной точки. Все остальные длительности в азбуке Морзе ей кратны. |
| **`embedded-hal-async`** | Асинхронный двойник `embedded-hal`: те же виды трейтов, но их методы — `async fn`. |
| **`Signal`** | Крошечный тип из `embassy-sync`, который хранит последнее значение; задача ждёт, пока другая задача не «просигналит» новое. |
| **маяк** (beacon) | Устройство, которое снова и снова повторяет одно и то же сообщение, как маяк на берегу. |

## 1. Азбука Морзе в пяти правилах

Азбука Морзе превращает каждую букву и цифру в точки и тире: `S` — это `...`, `O` — `---`, поэтому `SOS` —
`... --- ...`. Тайминг задаётся в **единицах**:

| Элемент | Длительность |
|---|---|
| точка | светодиод (LED, Light-Emitting Diode) горит 1 единицу |
| тире | светодиод горит 3 единицы |
| пауза между точками и тире одной буквы | выключен 1 единицу |
| пауза между буквами | выключен 3 единицы |
| пауза между словами (пробел) | выключен 7 единиц |

Пример: текст `E E` (E — одна точка) — это: горит 1, выключен 7, горит 1. `SOS` — это
`on1 off1 on1 off1 on1` (S, 5 единиц) + `off3` + `on3 off1 on3 off1 on3` (O, 11 единиц) + `off3` + S
(5 единиц) = **27 единиц**. При единице в 100 мс `SOS` длится 2,7 секунды.

```text
SOS, one character = one unit:

#_#_#___###_###_###___#_#_#
 S        O          S
```

## Проект

### Что вы создадите

`p22-morse` — **маяк азбуки Морзе**. Зелёный светодиод на Pico 2 повторяет
сообщение азбукой Морзе. Каждое нажатие кнопки переключает на следующее сообщение (`SOS` → `HELLO` →
`RUST` → `SOS` …), и новое сообщение начинается **сразу**, даже посреди буквы. `defmt` пишет в лог,
какое сообщение играет.

Устройство проекта следует правилу этого направления: почти вся логика живёт в библиотеке `no_std`,
которую вы тестируете на Mac; прошивка только связывает выводы, таймеры и задачи.

```text
code/p22-morse/
├── morse/        ← библиотека no_std: кодирование, тайминги, play(); тестируется через `cargo test`
│   ├── Cargo.toml
│   └── src/lib.rs
├── firmware/     ← прошивка на Embassy для Pico 2; собирается через `cargo build --release`
│   ├── .cargo/config.toml
│   ├── build.rs
│   ├── memory.x
│   ├── Cargo.toml
│   └── src/main.rs
└── README.md
```

### Контракт

**Библиотека `morse`** (`code/p22-morse/morse`, имя пакета `morse`, `#![no_std]`, без кучи):

```rust
/// Morse code of one letter or digit, for example 'A' -> ".-". Case does not matter.
/// Any other character (space, punctuation, letters outside A-Z) gives None.
pub fn encode_char(c: char) -> Option<&'static str>;

/// One step of a pattern: LED on or off for `units` units.
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub struct Step {
    pub on: bool,
    pub units: u8,
}

/// All steps for `text`, using the five rules. Unknown characters are skipped.
/// Spaces at the start or end produce no steps; there is no gap after the last symbol.
pub fn timings(text: &str) -> Timings<'_>;   // Timings implements Iterator<Item = Step>

/// Sum of the units of all steps of `text`.
pub fn total_units(text: &str) -> u32;

/// Plays `text` on `led`, one unit = `unit_ms` milliseconds. The LED is off at the end.
pub async fn play<P, D>(led: &mut P, delay: &mut D, text: &str, unit_ms: u32) -> Result<(), P::Error>
where
    P: embedded_hal::digital::OutputPin,
    D: embedded_hal_async::delay::DelayNs;
```

Обязательные результаты (это ваши приёмочные тесты):

| Вызов | Результат |
|---|---|
| `encode_char('A')`, `encode_char('s')`, `encode_char('5')`, `encode_char('?')` | `Some(".-")`, `Some("...")`, `Some(".....")`, `None` |
| `timings("E E")`, собранный в коллекцию | `[on 1, off 7, on 1]` |
| `total_units("SOS")`, `total_units("PARIS")`, `total_units("")` | `27`, `43`, `0` |
| `total_units("A?A")` | то же, что `total_units("AA")` |
| `total_units("  E  ")` | `1` |
| `play` для `"SOS"` с `unit_ms = 100` | общая задержка 2 700 мс, последний уровень вывода низкий |

**Прошивка** (`code/p22-morse/firmware`, пакет `p22-morse-firmware`): светодиод на выводе GPIO
(General-Purpose Input/Output, ввод-вывод общего назначения) 25, кнопка на GPIO 15 к земле (подтяжка
вверх), сообщения `["SOS", "HELLO", "RUST"]`, единица 150 мс, пауза в 7 единиц между повторами,
`info!("playing {}", message)` при каждом старте сообщения.

### Пошаговый план

#### M1 — Кодирование символов

Создайте библиотеку командой `cargo new --lib morse --name morse` внутри `code/p22-morse`, добавьте
`#![no_std]` и напишите `encode_char` с таблицей из 26 букв и 10 цифр (по правилам ITU, International
Telecommunication Union, Международного союза электросвязи; см. «Источники»). Напишите тесты из первой
строки таблицы контракта.

<details>
<summary>Подсказка</summary>

Достаточно `const TABLE: [(char, &str); 36]` и `c.to_ascii_uppercase()`. `iter().find(...)` возвращает
`Option<&(char, &str)>`; `.map(|(_, code)| *code)` превращает его в `Option<&'static str>`.

</details>

**Готово, когда:**

```bash
cd ~/projects/rust/code/p22-morse/morse
cargo test
```

```text
test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

(Число — это количество ваших тестов; их должно быть не меньше 1, и все должны проходить.) Коммит:
`M1: encode characters`.

#### M2 — Тайминги как итератор

Напишите `Step`, итератор `Timings` и `total_units`. Без `Vec`: итератор хранит несколько полей
состояния и выдаёт по одному `Step` за вызов `next`. Добавьте тесты для строк 2–5 таблицы контракта.

<details>
<summary>Подсказка</summary>

Полезное состояние: `Chars` текста, оставшиеся символы текущей буквы (`&'static [u8]`), пауза, которая
должна идти перед следующим символом (0, 1, 3 или 7), и признак, выдано ли уже что-нибудь. Пауза и
следующий за ней символ — это два шага; один из способов — запомнить символ в поле `Option<u8>` и вернуть
его при следующем вызове.

</details>

> ⚠️ **Подводный камень:** ошибка в состоянии итератора часто означает, что `next` никогда не возвращает
> `None`. Тогда `sum()` крутится «вечно»; в отладочной сборке это может через долгое время закончиться
> `attempt to add with overflow`. Если тест как будто завис, сначала проверьте
> `timings(...).take(20).collect::<Vec<_>>()` и напечатайте результат.

**Готово, когда:** `cargo test` проходит, включая `total_units("PARIS") == 43`. (PARIS — классическое
тестовое слово для скорости азбуки Морзе: 43 единицы плюс пауза между словами в 7 единиц — ровно 50.)
Коммит: `M2: timings`.

#### M3 — Воспроизведение с асинхронными фейками

Добавьте `embedded-hal-async` (`cargo add embedded-hal-async@1`) и напишите `play`. Для тестов добавьте
`cargo add --dev embassy-futures@0.1` и напишите фейковую асинхронную задержку:

```rust
struct FakeDelay(u64); // total nanoseconds

impl embedded_hal_async::delay::DelayNs for FakeDelay {
    async fn delay_ns(&mut self, ns: u32) {
        self.0 += ns as u64;
    }
}
```

Выполните `play` в тесте через `embassy_futures::block_on(...)` с фейковым выводом из Урока 22.11.

<details>
<summary>Подсказка</summary>

`play` — это цикл `for step in timings(text)`: выставить вывод, затем `delay.delay_ms(step.units as u32 * unit_ms).await`.
После цикла `led.set_low()` выполняет правило «в конце светодиод выключен» даже для пустого текста.

</details>

**Готово, когда:** тест проверяет, что воспроизведение `"SOS"` с `unit_ms = 100` даёт в сумме
`2_700 * 1_000_000` наносекунд и что последний уровень — `false`; `cargo test`,
`cargo clippy --all-targets` и `cargo fmt --check` чистые. Коммит: `M3: play`.

#### M4 — Прошивка

Скопируйте `code/p22-embassy` в `code/p22-morse/firmware`, переименуйте пакет в `p22-morse-firmware` и
добавьте `morse = { path = "../morse" }` и `embassy-futures = "0.1"`. В `main`:

1. `static NEXT: Signal<CriticalSectionRawMutex, usize> = Signal::new();` (из
   `embassy_sync::signal::Signal`);
2. задача `button`, которая после нажатия с подавлением дребезга вычисляет следующий индекс и вызывает
   `NEXT.signal(i)`;
3. цикл в `main`, который пишет сообщение в лог, а затем ждёт через
   `select(play_and_pause, NEXT.wait())`, где `play_and_pause` — блок `async`, который играет сообщение и
   затем ждёт 7 единиц. Если побеждает `NEXT`, переключитесь на новый индекс.

<details>
<summary>Подсказка</summary>

`embassy_time::Delay` реализует и асинхронный `DelayNs`, поэтому `morse::play(&mut led, &mut Delay, ...)`
работает на чипе без изменений. Поскольку `select` уничтожает проигравший future, нажатие сразу
останавливает текущее сообщение; следующий проход цикла запускает новое. Помните: в момент уничтожения
future светодиод может гореть. Выключите его перед новым воспроизведением или смиритесь с одной короткой вспышкой.

</details>

**Готово, когда:**

```bash
cd ~/projects/rust/code/p22-morse/firmware
cargo build --release
cargo size --release
```

```text
    Finished `release` profile [optimized + debuginfo] target(s) in 1.42s
   text	   data	    bss	    dec	    hex	filename
  20140	     56	   1568	  21764	   5504	p22-morse-firmware
```

Ваши числа будут немного отличаться; `text` должен оставаться меньше 32768 (32 КиБ). `cargo clippy --release`
чистый. Коммит: `M4: firmware`.

#### M5 — На плате и README

Прошейте командой `cargo run --release` (нужны плата и адаптер). Напишите `code/p22-morse/README.md`: что
делает маяк, подключение (кнопка GPIO 15 ↔ GND, вывод земли), как запускать тесты (`cd morse && cargo test`)
и как прошивать (`cd firmware && cargo run --release`).

**Готово, когда:** светодиод повторяет `SOS`, каждое нажатие сразу переключает сообщение, а лог показывает
(пример; точные строки зависят от вашего кода):

```text
INFO  playing SOS
└─ p22_morse_firmware::____embassy_main_task::{async_fn#0} @ src/main.rs:41
INFO  playing HELLO
└─ p22_morse_firmware::____embassy_main_task::{async_fn#0} @ src/main.rs:41
```

Без оборудования M5 готова, когда README написан и `cargo build --release` работает. Коммит: `M5: README`.

### Дополнительные цели

- **Управление скоростью:** долгое нажатие (больше 1 секунды, измеренное через `embassy_time::Instant`)
  переключает единицу между 100, 150 и 250 мс.
- **Свой текст:** читайте сообщение через последовательный порт USB (Universal Serial Bus, универсальная
  последовательная шина) с `embassy-usb` (см. примеры Embassy для RP2350).
- **Декодер:** обратное направление. Декодер `no_std`, который получает длительности «нажато/отпущено» от
  кнопки и выдаёт буквы; протестируйте его на Mac на записанных длительностях.
- **Проверка энергопотребления:** считайте, как часто просыпается исполнитель, `static`-атомарным
  счётчиком и пишите значение в лог каждые 10 секунд.

## Частые ошибки

> ⚠️ **Частая ошибка: `std` в библиотеке.** Если `morse` теряет строку `#![no_std]`, `cargo test` на Mac
> по-прежнему проходит, но `cargo build --release` в `firmware` падает, потому что библиотека компилируется
> и для чипа:
>
> ```text
> error[E0463]: can't find crate for `std`
>   |
>   = note: the `thumbv8m.main-none-eabihf` target may not support the standard library
>   = note: `std` is required by `morse` because it does not declare `#![no_std]`
> ```
>
> Собирайте прошивку после каждой вехи, а не только в конце.

> ⚠️ **Частая ошибка: одно рабочее пространство (workspace) для обоих пакетов.** Если сделать `code/p22-morse`
> рабочим пространством Cargo и положить `.cargo/config.toml` с платформой чипа на верхний уровень,
> `cargo test` соберёт библиотеку для чипа и упадёт (`can't find crate for test`). Держите настройки внутри
> `firmware/`, как в структуре выше.

> ⚠️ **Частая ошибка: не тот `DelayNs`.** У `embedded_hal::delay::DelayNs` (блокирующего) и
> `embedded_hal_async::delay::DelayNs` (асинхронного) одинаковое имя. `play` должна использовать
> асинхронный, иначе `.await` у `delay_ms` не скомпилируется.

## Следующие шаги во встраиваемом Rust

Теперь вы умеете собирать, тестировать и запускать асинхронные прошивки. Хорошие направления отсюда:

**Учиться дальше**

- *The Embedded Rust Book* и *The Embedonomicon* (ссылки ниже): основы, стартовый код, скрипты компоновщика,
  прерывания и конкурентность.
- *Discovery* (издание для micro:bit v2): вторая плата с датчиками, другое семейство чипов и много
  практических глав.
- *The Embassy Book*: сеть через `embassy-net`, USB через `embassy-usb`, Wi-Fi на Pico 2 W.
- *Comprehensive Rust — Bare-metal Rust*: компактный курс по `no_std`, MMIO (memory-mapped I/O, ввод-вывод, отображённый в память) и написанию драйверов.
- Даташит RP2350: когда освоитесь, прочитайте главы про PIO (Programmable Input/Output,
  программируемый ввод-вывод: маленькие автоматы, которые сами управляют выводами) и DMA (Direct Memory
  Access, прямой доступ к памяти: копирование данных без процессора, CPU — Central Processing Unit).

**Вносить вклад** (Урок 24.4 подробно разбирает процесс)

- **Напишите драйвер.** Возьмите дешёвый датчик (температуры, расстояния, освещённости), для которого ещё
  нет драйвера на Rust, напишите его на трейтах `embedded-hal` 1.0, протестируйте с фейками и опубликуйте.
  Это самый частый первый вклад во встраиваемый Rust.
- **Embassy** (`github.com/embassy-rs/embassy`): примеры, документация и возможности HAL (Hardware
  Abstraction Layer, слой абстракции оборудования) для многих чипов.
- **probe-rs** (`github.com/probe-rs/probe-rs`): поддержка новых чипов и адаптеров, отладчик.
- **rp-hal** (`github.com/rp-rs/rp-hal`): другое семейство HAL для RP2040 и RP2350.
- Ищите задачи с метками «good first issue» или «help wanted»; сначала прочитайте руководство для участников.

**Сообщества**

- Rust Embedded Working Group, которая поддерживает `embedded-hal`, `cortex-m` и книги
  (`github.com/rust-embedded/wg`), и её чат в Matrix `#rust-embedded:matrix.org`.
- Комната Embassy в Matrix `#embassy-rs:matrix.org`.
- Список *Awesome Embedded Rust*: HAL, драйверы, инструменты и блоги.

> 🧭 **Конец направления встраиваемых систем.** Переходите к [Уроку 22.14 — Трек баз данных: как работают движки запросов](14-databases-track-how-query-engines-work.md),
> если хотите второе направление, или к [контрольной точке, Уроку 22.30](30-checkpoint-your-specialization.md).

## Вопросы для самопроверки

1. Почему почти вся логика маяка находится в библиотеке `morse`, а не в прошивке?

<details>
<summary>Ответ</summary>

Библиотека — это `no_std` без аппаратных типов, поэтому её можно тестировать на Mac за миллисекунды с
фейками. Прошивка только связывает выводы, таймеры и задачи, и эту маленькую часть проверяют на плате.

</details>

2. Сколько единиц возвращает `total_units("SOS")` и почему?

<details>
<summary>Ответ</summary>

27: S — 5 единиц (три точки с двумя паузами по 1 единице), O — 11 (три тире с двумя паузами), плюс две
паузы между буквами по 3 единицы: 5 + 3 + 11 + 3 + 5 = 27.

</details>

3. Как нажатие кнопки останавливает сообщение посередине?

<details>
<summary>Ответ</summary>

`main` ждёт через `select(play_and_pause, NEXT.wait())`. Когда задача кнопки сигналит, `NEXT.wait()`
готов первым, поэтому `select` возвращается и уничтожает играющий future, что отменяет его таймер. Затем
цикл запускает новое сообщение.

</details>

4. Почему одна и та же функция `play` работает и в тесте на Mac, и на чипе?

<details>
<summary>Ответ</summary>

Она обобщена по трейтам `OutputPin` и асинхронному `DelayNs`. Тест передаёт фейковый вывод и фейковую
задержку; прошивка передаёт `Output` и `embassy_time::Delay`, которые реализуют те же трейты.

</details>

## Итоги

- Планируйте прошивку как библиотеку логики `no_std` (тестируется на хосте) плюс тонкую прошивку на Embassy.
- Тайминг азбуки Морзе задаётся в единицах: точка 1, тире 3, паузы 1 / 3 / 7.
- Итератор с несколькими полями состояния выдаёт целый узор без кучи.
- `embedded-hal-async` позволяет использовать задержки и выводы в обобщённых `async fn`, которые
  тестируются через `block_on`.
- `select` вместе с `Signal` позволяет новому событию сразу отменить идущий узор.
- Следующие шаги: драйверы, Embassy, probe-rs, rp-hal и сообщество Rust Embedded.

## Источники и что почитать

- [ITU-R M.1677-1 — International Morse code](https://www.itu.int/rec/R-REC-M.1677-1-200910-I/) — официальное определение кода и его таймингов.
- [`embedded-hal-async` 1.0](https://docs.rs/embedded-hal-async/1.0.0/embedded_hal_async/) — асинхронный `DelayNs` и другие трейты.
- [`Signal` из `embassy-sync`](https://docs.rs/embassy-sync/0.8.0/embassy_sync/signal/struct.Signal.html) — как работают `signal` и `wait`.
- [Discovery, micro:bit v2 edition](https://docs.rust-embedded.org/discovery-mb2/) — следующая книга для чтения.
- [Comprehensive Rust — Bare-metal Rust](https://google.github.io/comprehensive-rust/bare-metal.html) — компактный курс по Rust на «голом железе» и драйверам.
- [Awesome Embedded Rust](https://github.com/rust-embedded/awesome-embedded-rust) — подборка HAL, драйверов и инструментов.
- [Rust Embedded Working Group](https://github.com/rust-embedded/wg) — кто поддерживает основные крейты и как присоединиться.

---

[🏠 Начало](../START-HERE.md) · [← 22.12 Трек встраиваемых систем: асинхронность с Embassy](12-embedded-track-async-with-embassy.md) · [📋 Этап 22](README.md) · [22.14 Трек баз данных: как работают движки запросов →](14-databases-track-how-query-engines-work.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
