<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 22.2 Трек ОС: программа на Rust без операционной системы](02-os-track-a-freestanding-binary.md) · [📋 Этап 22](README.md) · [22.4 Трек ОС: прерывания и страничная память →](04-os-track-interrupts-and-paging.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/22-specializations/03-os-track-booting-a-kernel-in-qemu.md) <!-- nav -->

# Урок 22.3 — Трек ОС: загрузка ядра в QEMU

> **Этап 22 — Специализации** · Урок 3 из 30 · ⏱ около 2,5 часа (2 занятия)

## Что вы узнаете

- Как выглядит машина `virt` в QEMU: где её память и её устройства.
- Как запускать QEMU в контейнере и сделать так, чтобы `cargo run` загружал ваше ядро.
- Как скрипт компоновщика размещает ядро в памяти и что должны сделать первые инструкции на ассемблере.
- Как общаться с устройством через регистры, отображённые в память, и как построить `println!` поверх драйвера UART.
- Как печатать сообщения паники и выключать машину.

## Перед началом

- Вы прошли [Урок 22.2 — Трек ОС: программа на Rust без операционной системы](02-os-track-a-freestanding-binary.md);
  у вас есть `code/p22-kernel` с `src/lib.rs`, `.cargo/config.toml` и псевдонимом `host-test`.
- Вы знаете секции и символы ELF-файлов ([Урок 12.12](../12-computer-systems/12-linking-and-loading.md)) и стек
  ([Урок 03.1](../03-ownership-and-borrowing/01-memory-stack-and-heap.md)).
- Вы умеете собирать и запускать образ Docker ([Этап 11](../11-linux-shell-and-containers/README.md)).

## Новые слова

| Слово | Значение |
|---|---|
| **карта памяти** (memory map) | Список диапазонов адресов машины и того, что стоит за каждым: оперативная память или устройство. |
| **MMIO** (Memory-Mapped I/O, ввод-вывод, отображённый в память) | Устройства, которые выглядят как адреса: чтение или запись такого адреса обращается к устройству, а не к памяти. |
| **регистр** (register) устройства | Одна маленькая управляющая ячейка или ячейка данных устройства по фиксированному адресу. Не то же самое, что регистр процессора. |
| **UART** (Universal Asynchronous Receiver-Transmitter, универсальный асинхронный приёмопередатчик) | Простой чип, который отправляет и принимает байты по одному биту по последовательной линии. |
| **последовательная консоль** (serial console) | Текст, который машина отправляет через свой UART; QEMU показывает его в вашем терминале. |
| **PL011** | Модель UART от Arm, которая есть у машины `virt`. |
| **volatile-доступ** (volatile access) | Чтение или запись памяти, которую компилятор обязан выполнить ровно как написано, никогда не удаляя и не объединяя. Нужен для устройств. |
| **скрипт компоновщика** (linker script) | Файл, который говорит компоновщику, по каким адресам положить код и данные и какие символы определить. |
| **BSS** (Block Started by Symbol, «блок, начинающийся с символа») | Секция глобальных данных, инициализированных нулём. В файле она места не занимает; ядро должно само заполнить её нулями. |
| **указатель стека** (stack pointer, `sp`) | Регистр процессора, который указывает на вершину стека. |
| **`global_asm!`** | Макрос Rust, который вставляет ассемблерный код в программу вне какой-либо функции. |
| **PSCI** (Power State Coordination Interface, интерфейс координации состояний питания) | Стандарт Arm, позволяющий попросить прошивку включить или выключить процессоры или всю машину. |
| **раннер Cargo** (Cargo runner) | Программа, которую `cargo run` запускает вместо вашей программы, передавая ей путь к двоичному файлу. |
| **DTB** (Device Tree Blob, двоичное дерево устройств) | Файл, описывающий оборудование машины. QEMU кладёт его в начало оперативной памяти. |

**Сокращения из прошлых уроков:** ОС (операционная система), CPU (Central Processing Unit, центральный процессор),
RAM (Random Access Memory, оперативная память), ELF (Executable and Linkable Format, формат файла ядра) и QEMU (Quick
Emulator, быстрый эмулятор). GIC (Generic Interrupt Controller, универсальный контроллер прерываний) есть на карте
памяти; его использует Урок 22.4.

## 1. Машина, которую мы загружаем

QEMU — программа, которая эмулирует целый компьютер: его процессор, память и устройства. Машина `virt` — простая
плата arm64. Её самые важные адреса:

```text
 address        what is there
 0x0000_0000    flash memory (firmware, unused by us)
 0x0800_0000    GIC: the interrupt controller (Lesson 22.4)
 0x0900_0000    UART PL011: our serial console
 0x4000_0000    RAM starts (128 MiB with -m 128M)
                0x4000_0000  the DTB that QEMU writes here
                0x4008_0000  our kernel
```

Когда вы запускаете QEMU с `-kernel <file>`, а файл — ELF, QEMU копирует каждую часть файла по адресу, записанному в
файле, ставит счётчик команд на точку входа ELF и запускает один процессор. Нет ни прошивки, ни загрузчика, ни
операционной системы: ваш `_start` — первая выполняемая инструкция.

Значит, ядро нужно скомпоновать для адреса внутри оперативной памяти. Мы используем `0x4008_0000`, оставляя первые
512 КиБ памяти под DTB.

## 2. QEMU в контейнере, запуск через `cargo run`

Вы никогда не устанавливаете QEMU на Mac. Он работает в контейнере из официального образа `ubuntu:26.04`. Создайте
файл `qemu/Dockerfile` внутри `code/p22-kernel`:

```dockerfile
FROM ubuntu:26.04
RUN apt-get update && apt-get install -y --no-install-recommends qemu-system-arm && rm -rf /var/lib/apt/lists/*
```

Соберите образ один раз (это занимает минуту-две):

```bash
docker build -t course-qemu qemu
docker image ls --filter reference='course-qemu'
```

```text
IMAGE                ID             DISK USAGE   CONTENT SIZE   EXTRA
course-qemu:latest   89f2b5e40a87        378MB         81.3MB
```

Пакет `qemu-system-arm` содержит `qemu-system-aarch64` — эмулятор 64-битных машин Arm. Дальше маленький скрипт
`qemu.sh` в папке пакета запускает его:

```sh
#!/bin/sh
# Загружает ELF-файл ядра, который Cargo передаёт как $1, в QEMU внутри контейнера.
set -eu
tty_flag=""
if [ -t 0 ]; then tty_flag="-t"; fi  # настоящий терминал: пусть Ctrl-A X доходит до QEMU
docker run --rm -i $tty_flag -v "$PWD":/work -w /work course-qemu \
  qemu-system-aarch64 -machine virt -cpu cortex-a72 -m 128M \
  -nographic -nic none -kernel "$1"
```

Сделайте его исполняемым: `chmod +x qemu.sh`. Параметры QEMU:

| Параметр | Значение |
|---|---|
| `-machine virt` | Эмулировать плату `virt`. |
| `-cpu cortex-a72` | Модель настоящего 64-битного процессора Arm. Без неё `virt` запускает 32-битный процессор. |
| `-m 128M` | 128 МиБ оперативной памяти. |
| `-nographic` | Без окна: последовательная консоль идёт в ваш терминал. Выход — **Ctrl-A**, затем **X**. |
| `-nic none` | Без сетевой карты (драйвера у нас нет, а QEMU искал бы дополнительный загрузочный файл). |
| `-kernel "$1"` | Загрузить этот ELF-файл и начать с его точки входа. |

`-v "$PWD":/work` делает текущую папку видимой внутри контейнера, поэтому путь, который передаёт Cargo
(`target/.../p22-kernel`), работает и там. Поэтому всегда запускайте `cargo run` из папки пакета.

Теперь свяжите всё в `.cargo/config.toml`:

```toml
[build]
target = "aarch64-unknown-none-softfloat"

[target.aarch64-unknown-none-softfloat]
# Использовать наш скрипт компоновщика и загружать ядро в QEMU по `cargo run`.
rustflags = ["-C", "link-arg=-Tlinker.ld"]
runner = "./qemu.sh"

[alias]
# Запускает тесты src/lib.rs на Mac, а не на платформе ядра.
host-test = "test --lib --target aarch64-apple-darwin"
```

**Что происходит:**

1. `rustflags` добавляет `-Tlinker.ld` к команде компоновщика, так что компоновщик использует наш скрипт (следующий
   раздел).
2. `runner` говорит Cargo: для `cargo run` не запускай двоичный файл (на Mac это невозможно), а запусти `./qemu.sh`
   с путём к двоичному файлу.
3. Обе настройки стоят внутри `[target.aarch64-unknown-none-softfloat]`, поэтому действуют только для платформы
   ядра. `cargo host-test` собирает для Mac, и они его не затрагивают.

## 3. Скрипт компоновщика

Создайте `linker.ld` в папке пакета (рядом с `Cargo.toml`):

```text
ENTRY(_start)

SECTIONS
{
    . = 0x40080000;

    .text : {
        KEEP(*(.text.boot))
        *(.text .text.*)
    }
    .rodata : { *(.rodata .rodata.*) }
    .data : { *(.data .data.*) }

    .bss (NOLOAD) : ALIGN(16) {
        __bss_start = .;
        *(.bss .bss.*)
        *(COMMON)
        . = ALIGN(16);
        __bss_end = .;
    }

    . = ALIGN(16);
    . += 0x10000;
    __stack_top = .;

    /DISCARD/ : { *(.eh_frame .eh_frame_hdr) }
}
```

**Что происходит:**

1. `ENTRY(_start)` записывает адрес `_start` в заголовок ELF как точку входа.
2. `.` — это **счётчик размещения** (location counter): «адрес, на котором мы сейчас». `. = 0x40080000;`
   начинает оттуда.
3. Выходная секция `.text` (код) собирает сначала `.text.boot`, затем весь остальной код. `KEEP` не даёт
   компоновщику её удалить, а размещение первой делает `_start` самым первым байтом по адресу `0x4008_0000`.
4. Дальше идут `.rodata` (константные данные, например строки) и `.data` (глобальные переменные с начальным
   значением).
5. `.bss` хранит глобальные переменные, которые начинаются с нуля. `NOLOAD` значит: в файле она не занимает байтов.
   Скрипт определяет два символа, `__bss_start` и `__bss_end`, чтобы код загрузки знал, что заполнять нулями.
6. После BSS мы пропускаем 64 КиБ (`0x10000`) и определяем `__stack_top`. В arm64 стек растёт **вниз**, поэтому
   область под `__stack_top` — наш стек.
7. `/DISCARD/` выбрасывает таблицы раскрутки (`.eh_frame`), которые ядро с `panic = "abort"` никогда не использует.

```text
0x4008_0000  +-------------------+  <- _start (from .text.boot)
             | .text   code      |
             | .rodata constants |
             | .data   globals   |
             | .bss    zeros     |  __bss_start .. __bss_end
             +-------------------+
             |   64 KiB stack    |  grows downwards
             +-------------------+  <- __stack_top
```

## 4. Первые инструкции на ассемблере

Коду на Rust нужен стек с самой первой инструкции (вы видели `str x30, [sp, ...]` в Уроке 22.2). Никто ещё не
установил `sp`, поэтому первые несколько инструкций должны быть на ассемблере. Создайте `src/boot.s`:

```text
.section .text.boot
.global _start
_start:
    // Only CPU core 0 continues; any other core sleeps forever.
    mrs     x1, mpidr_el1
    and     x1, x1, #0xff
    cbnz    x1, park

    // Set the stack pointer to the top of our stack area.
    ldr     x1, =__stack_top
    mov     sp, x1

    // Fill the .bss section with zeros.
    ldr     x1, =__bss_start
    ldr     x2, =__bss_end
zero_bss:
    cmp     x1, x2
    b.hs    bss_done
    str     xzr, [x1], #8
    b       zero_bss
bss_done:

    bl      kernel_main

park:
    wfe
    b       park
```

**Что происходит:**

1. `.section .text.boot` помещает этот код в секцию, которую скрипт компоновщика ставит первой.
2. `mrs x1, mpidr_el1` читает системный регистр с номером этого ядра процессора. На машине с несколькими ядрами все
   они могут начать здесь; продолжает только ядро 0 (`cbnz` прыгает на `park`, если номер не ноль). QEMU по
   умолчанию запускает одно ядро, но проверка делает код безопасным.
3. `ldr x1, =__stack_top` загружает 64-битный адрес символа компоновщика; `mov sp, x1` делает его указателем стека.
4. Цикл записывает нулевой регистр `xzr` в каждые 8 байтов от `__bss_start` до `__bss_end`. `str xzr, [x1], #8`
   сохраняет, а затем прибавляет 8 к `x1`. Rust ожидает, что `static` со значением `0` действительно равен нулю.
5. `bl kernel_main` вызывает нашу функцию на Rust. Она никогда не возвращается, но если бы вернулась, ядро
   процессора ушло бы в `park`.
6. `wfe` («wait for event», «жди события») усыпляет ядро процессора, пока что-нибудь не случится; цикл отправляет
   его спать снова.

`_start` из Урока 22.2 больше нет: точка входа теперь на ассемблере, а Rust начинается с `kernel_main`.

## 5. Общаемся с UART

Устройство на `virt` — набор **регистров по фиксированным адресам**. Для UART PL011 по адресу `0x0900_0000` нужны
два из них:

| Смещение | Регистр | Для чего |
|---|---|---|
| `0x00` | регистр данных | Запись байта сюда отправляет его по последовательной линии. |
| `0x18` | регистр флагов | Бит 5 (`TXFF`) равен 1, пока очередь передачи полна; тогда нужно ждать. |

Запись по такому адресу должна быть **volatile**-записью. Обычную запись в память оптимизатор может удалить, если
значение потом никто не читает, или объединить со следующей записью. Для устройства каждое обращение — действие
(«отправь этот байт»), поэтому оно должно произойти ровно как написано: это гарантируют `core::ptr::write_volatile`
и `read_volatile`.

Создайте `src/uart.rs`:

```rust
//! A tiny driver for the PL011 UART of the QEMU `virt` machine.

use core::fmt;
use core::ptr::{read_volatile, write_volatile};

const UART_BASE: usize = 0x0900_0000;
const DATA: usize = 0x00; // data register: write a byte here to send it
const FLAGS: usize = 0x18; // flag register
const TX_FULL: u32 = 1 << 5; // flag bit: the transmit queue is full

pub struct Uart;

impl Uart {
    pub fn write_byte(&mut self, byte: u8) {
        let flags = (UART_BASE + FLAGS) as *const u32;
        let data = (UART_BASE + DATA) as *mut u32;
        // SAFETY: on the QEMU `virt` machine these addresses are the registers
        // of the PL011 UART, and volatile access is how device registers are used.
        unsafe {
            while read_volatile(flags) & TX_FULL != 0 {}
            write_volatile(data, byte as u32);
        }
    }
}

impl fmt::Write for Uart {
    fn write_str(&mut self, s: &str) -> fmt::Result {
        for byte in s.bytes() {
            if byte == b'\n' {
                self.write_byte(b'\r');
            }
            self.write_byte(byte);
        }
        Ok(())
    }
}

#[macro_export]
macro_rules! print {
    ($($arg:tt)*) => {{
        use core::fmt::Write as _;
        let _ = write!($crate::uart::Uart, $($arg)*);
    }};
}

#[macro_export]
macro_rules! println {
    () => { $crate::print!("\n") };
    ($($arg:tt)*) => {{
        $crate::print!($($arg)*);
        $crate::print!("\n");
    }};
}
```

**Что происходит:**

1. `Uart` — структура без полей: устройство существует в одном экземпляре, и всё его состояние — в оборудовании.
2. `write_byte` ждёт, пока установлен бит «очередь передачи полна», затем записывает байт. Превратить число в
   указатель (`as *mut u32`) безопасно; использовать его — `unsafe`, и комментарий `// SAFETY:` объясняет, почему
   здесь это допустимо (Урок 06.6).
3. `fmt::Write` — трейт из `core`, который нужен форматированию: реализуйте `write_str`, и вы получите
   `write!(uart, "...{}", x)` со всем форматированием `println!`. Куча не нужна.
4. Последовательному терминалу нужен `\r` (возврат каретки: перейти в начало строки) перед `\n` (новая строка).
5. `print!` и `println!` — наши собственные макросы (Урок 06.7) с теми же именами, что и в `std`. `#[macro_export]`
   делает их доступными во всём крейте; `$crate::uart::Uart` находит драйвер из любого модуля.

> ⚡ **Производительность:** UART в QEMU никогда не заполняется, поэтому цикл ожидания заканчивается сразу.
> Настоящий UART на скорости 115200 бод отправляет около 11 500 байтов в секунду: ядро, которое много печатает,
> проводит в этом цикле реальное время. Настоящие ядра используют буфер и прерывание «готов принять ещё».

## 6. Выключение и первая загрузка

Когда ядро закончило, оно должно выключить машину, чтобы QEMU завершился. Машины Arm делают это через **PSCI** —
маленький стандартный интерфейс к прошивке. На `virt` сам QEMU отвечает на вызовы PSCI, которые ядро делает
инструкцией `hvc` (hypervisor call, вызов гипервизора). Создайте `src/power.rs`:

```rust
//! Turning the machine off with PSCI (Power State Coordination Interface).

const PSCI_SYSTEM_OFF: u64 = 0x8400_0008;

pub fn shutdown() -> ! {
    // SAFETY: on QEMU `virt` the firmware interface answers `hvc #0`;
    // SYSTEM_OFF never returns.
    unsafe {
        core::arch::asm!("hvc #0", in("x0") PSCI_SYSTEM_OFF, options(noreturn));
    }
}
```

`asm!` вставляет в функцию одну инструкцию (Урок 06.6 показывал `unsafe`; Rust Reference подробно описывает `asm!`).
`in("x0") PSCI_SYSTEM_OFF` кладёт номер функции `0x8400_0008` («выключить систему») в регистр `x0` перед
инструкцией. `options(noreturn)` обещает, что выполнение после неё никогда не продолжится.

Теперь замените `src/main.rs`:

```rust
#![no_std]
#![no_main]

mod power;
mod uart;

use core::panic::PanicInfo;

core::arch::global_asm!(include_str!("boot.s"));

#[unsafe(no_mangle)]
pub extern "C" fn kernel_main() -> ! {
    println!("Hello from p22-kernel!");
    let answer = 6 * 7;
    println!("6 * 7 = {answer}");
    println!("kernel_main is at {:#x}", kernel_main as *const () as usize);
    power::shutdown();
}

#[panic_handler]
fn panic(info: &PanicInfo) -> ! {
    println!("KERNEL PANIC: {info}");
    power::shutdown();
}
```

**Что происходит:**

1. `global_asm!(include_str!("boot.s"))` читает `src/boot.s` во время компиляции и ассемблирует его в программу.
   Теперь символ `_start` берётся оттуда.
2. `kernel_main` вызывается из ассемблера, поэтому ему нужно фиксированное имя (`no_mangle`) и соглашение о вызовах
   C. Он печатает три строки нашим `println!`, включая собственный адрес, и выключает машину.
3. Обработчик паники теперь печатает сообщение: `PanicInfo` реализует `Display`, поэтому `{info}` показывает текст
   и место в исходном коде.

Загрузите:

```bash
cargo run
```

```text
   Compiling p22-kernel v0.1.0 (/Users/student/projects/rust/code/p22-kernel)
    Finished `dev` profile [unoptimized + debuginfo] target(s) in 0.83s
     Running `/Users/student/projects/rust/code/p22-kernel/./qemu.sh target/aarch64-unknown-none-softfloat/debug/p22-kernel`
Hello from p22-kernel!
6 * 7 = 42
kernel_main is at 0x40080c88
```

Ваше собственное ядро загрузилось на (эмулированной) машине и напечатало текст. QEMU завершился сам, потому что ядро
выключило машину. Проверьте, что куда попало, инструментами из Урока 22.2:

```bash
"$LLVM_BIN/llvm-readobj" --file-headers target/aarch64-unknown-none-softfloat/debug/p22-kernel | grep Entry
"$LLVM_BIN/llvm-nm" -n target/aarch64-unknown-none-softfloat/debug/p22-kernel | grep -E '_start|kernel_main$|__bss|__stack'
```

```text
  Entry: 0x40080000
0000000040080000 T _start
0000000040080c88 T kernel_main
0000000040082a90 B __bss_end
0000000040082a90 B __bss_start
0000000040092a90 B __stack_top
```

Точка входа — `0x4008_0000`, как и сказано в скрипте компоновщика. BSS пока пустая (начало = конец), а вершина стека
на 64 КиБ (`0x10000`) выше неё.

## 7. Паника в ядре

Добавьте две строки перед `power::shutdown();` в `kernel_main`, чтобы прочитать за концом массива:

```rust
    let values = [1, 2, 3];
    let index = answer / 10;
    println!("value: {}", values[index]);
```

```text
Hello from p22-kernel!
6 * 7 = 42
kernel_main is at 0x40080cd0
KERNEL PANIC: panicked at src/main.rs:19:27:
index out of bounds: the len is 3 but the index is 4
```

Проверка границ Rust работает в ядре точно так же, как в обычных программах, и теперь вы видите сообщение и строку.
Без неё чтение `values[4]` молча вернуло бы то, что лежит после массива. Уберите эти две строки.

> 🔒 **Безопасность:** в ядре тихое чтение за границей может выдать данные другой программы, а запись — захватить
> машину. Паника, которая останавливает ядро, гораздо лучше, чем продолжение работы с испорченной памятью.

> 🧪 **Тестирование:** вывод паники — простой тестовый оракул: скрипт может запустить `cargo run`, перехватить
> вывод и поискать `KERNEL PANIC`. Урок 22.5 показывает, как превратить это в автоматические тесты в QEMU.

## Частые ошибки

> ⚠️ **Частая ошибка:** компоновщик не получает скрипт (опечатка в `rustflags` или `linker.ld` не в той папке).
> Тогда код загрузки ссылается на символы, которые никто не определяет:
>
> ```text
> rust-lld: error: undefined symbol: __stack_top
> rust-lld: error: undefined symbol: __bss_end
> ```
>
> `-Tlinker.ld` читается относительно папки пакета; держите `linker.ld` рядом с `Cargo.toml`.

> ⚠️ **Частая ошибка:** забыть `-nic none`. Тогда QEMU добавляет сетевую карту и ищет её загрузочный файл:
>
> ```text
> qemu-system-aarch64: failed to find romfile "efi-virtio.rom"
> ```

> ⚠️ **Частая ошибка:** ядро, которое никогда не вызывает `shutdown` (например, бесконечный цикл), оставляет QEMU
> работать. Оно не зависло: нажмите **Ctrl-A**, затем **X**, чтобы выйти из QEMU.

> ⚠️ **Частая ошибка:** писать в устройство обычным `*ptr = value`. Это может работать в отладочной сборке и
> исчезнуть в release-сборке, потому что оптимизатору разрешено удалить запись, которую никто не читает. Для
> регистров устройств всегда используйте `read_volatile` и `write_volatile`.

## Попробуйте сами

1. Соберите образ, создайте `qemu.sh`, `linker.ld`, `src/boot.s`, `src/uart.rs`, `src/power.rs` и загрузите ядро
   через `cargo run`.
2. Измените текст первого `println!` и загрузите снова.
3. Уберите `-cpu cortex-a72` из `qemu.sh` и загрузите. QEMU откажется с сообщением «The image is from incompatible
   architecture»: процессор `virt` по умолчанию 32-битный. Верните параметр на место.
4. Запустите `cargo host-test` и `cargo clippy`, затем сделайте коммит: `git commit -am "p22-kernel: boot in QEMU"`
   (новые файлы сначала добавьте через `git add`).

## Упражнения

### Упражнение 1 — Сколько ядро занимает в памяти? (легко)

Напечатайте из `kernel_main` адрес `__bss_end` и `__stack_top`, а также сколько КиБ ядро занимает от `0x4008_0000`
до вершины стека.

<details>
<summary>Подсказка</summary>

Объявите символы компоновщика в Rust как `unsafe extern "C" { static __bss_end: u8; }`. Их значение вы никогда не
читаете; вы только берёте их адрес через `&raw const __bss_end`.

</details>

<details>
<summary>Ответ</summary>

Добавьте это в `kernel_main` перед `power::shutdown();`:

```rust
    unsafe extern "C" {
        static __bss_end: u8;
        static __stack_top: u8;
    }
    let bss_end = &raw const __bss_end as usize;
    let stack_top = &raw const __stack_top as usize;
    println!("kernel data ends at {bss_end:#x}, stack top is {stack_top:#x}");
    println!("the kernel uses {} KiB", (stack_top - 0x4008_0000) / 1024);
```

```text
kernel_main is at 0x40080ce4
kernel data ends at 0x40082cf0, stack top is 0x40092cf0
the kernel uses 75 KiB
```

Почему это работает: символ компоновщика — только адрес; за ним нет переменной. `&raw const` берёт адрес, не создавая
ссылку на память, которая на самом деле не `u8`. 64 КиБ из 75 КиБ — это стек.

</details>

### Упражнение 2 — Тестируем правило перевода строки на Mac (средне)

Вынесите правило «отправлять `\r` перед каждым `\n`» из драйвера в `src/lib.rs` как чистую функцию
`pub fn serial_bytes(text: &str, emit: impl FnMut(u8))` и напишите для неё два теста. Драйвер вызывает её.

<details>
<summary>Подсказка</summary>

В тесте `emit` может быть замыканием, которое добавляет байты в `Vec<u8>` (у тестов есть `std`). `b"a\r\nb"` —
строка байтов, которую можно сравнить с вектором.

</details>

<details>
<summary>Ответ</summary>

В `src/lib.rs`:

```rust
/// Calls `emit` for every byte of `text`, sending "\r\n" for each "\n",
/// because a serial terminal needs both to start a new line.
pub fn serial_bytes(text: &str, mut emit: impl FnMut(u8)) {
    for byte in text.bytes() {
        if byte == b'\n' {
            emit(b'\r');
        }
        emit(byte);
    }
}
```

Внутри `mod tests`:

```rust
    #[test]
    fn newline_becomes_carriage_return_and_newline() {
        let mut sent = Vec::new();
        serial_bytes("a\nb", |byte| sent.push(byte));
        assert_eq!(sent, b"a\r\nb");
    }

    #[test]
    fn text_without_newline_is_unchanged() {
        let mut sent = Vec::new();
        serial_bytes("ok", |byte| sent.push(byte));
        assert_eq!(sent, b"ok");
    }
```

А в `src/uart.rs` тело `write_str` становится таким:

```rust
        p22_kernel::serial_bytes(s, |byte| self.write_byte(byte));
        Ok(())
```

```text
test result: ok. 4 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

Почему это работает: замыкание решает, куда идут байты, — в UART в ядре, в `Vec` в тесте. Логика проверяется один
раз, на Mac, а ядро печатает в точности как раньше.

</details>

### Упражнение 3 — Сколько длилась загрузка? (сложно)

В каждом процессоре arm64 есть системный счётчик: `CNTPCT_EL0` считает такты с момента включения, а `CNTFRQ_EL0`
говорит, сколько тактов в секунду. Прочитайте оба через `asm!` и напечатайте время с момента включения в
микросекундах.

<details>
<summary>Подсказка</summary>

`core::arch::asm!("mrs {}, cntpct_el0", out(reg) ticks);` читает системный регистр в переменную Rust. Сначала
умножайте, потом делите, иначе получится 0.

</details>

<details>
<summary>Ответ</summary>

```rust
    let (ticks, freq): (u64, u64);
    // SAFETY: reading the counter and its frequency has no side effects.
    unsafe {
        core::arch::asm!("mrs {}, cntpct_el0", out(reg) ticks);
        core::arch::asm!("mrs {}, cntfrq_el0", out(reg) freq);
    }
    let micros = ticks * 1_000_000 / freq;
    println!("counter: {ticks} ticks at {freq} Hz = {micros} us since power-on");
```

```text
counter: 142455 ticks at 62500000 Hz = 2279 us since power-on
```

Почему это работает: `mrs` копирует системный регистр в регистр общего назначения, а `out(reg) ticks` говорит
компилятору выбрать регистр и потом сохранить его в `ticks`. Счётчик QEMU идёт с частотой 62,5 МГц; загрузка до этой
строки заняла около 2 мс (у вас число будет другим при каждом запуске).

</details>

## Вопросы для самопроверки

1. Почему ядро компонуется по адресу `0x4008_0000`, а не `0x0`?

<details>
<summary>Ответ</summary>

На `virt` оперативная память начинается с `0x4000_0000`; ниже — флеш-память и устройства. QEMU кладёт DTB в начало
памяти, поэтому ядро идёт чуть выше.

</details>

2. Какие две задачи должен выполнить ассемблерный код до первой функции на Rust?

<details>
<summary>Ответ</summary>

Установить указатель стека (код на Rust сразу использует стек) и заполнить BSS нулями (Rust ожидает, что статические
переменные с нулевой инициализацией равны нулю). Ещё он усыпляет все ядра процессора, кроме ядра 0.

</details>

3. Почему к регистрам устройств нужно обращаться через `read_volatile` / `write_volatile`?

<details>
<summary>Ответ</summary>

Каждое обращение к устройству — действие. Компилятор может удалить, повторить или объединить обычные обращения к
памяти; volatile-обращения происходят ровно как написано.

</details>

4. Что делает настройка `runner` в `.cargo/config.toml`?

<details>
<summary>Ответ</summary>

`cargo run` запускает программу-раннер (`./qemu.sh`), передавая ей путь к собранному двоичному файлу, вместо того
чтобы запускать сам двоичный файл.

</details>

5. Как ядро заставляет QEMU завершиться?

<details>
<summary>Ответ</summary>

Оно делает вызов PSCI `SYSTEM_OFF`: номер функции `0x8400_0008` в `x0`, затем `hvc #0`. QEMU играет роль прошивки,
выключает машину и завершается.

</details>

## Итоги

- У `virt` оперативная память начинается с `0x4000_0000`, а устройства — ниже (UART по `0x0900_0000`); QEMU загружает
  ELF-ядро и переходит на его точку входа.
- QEMU работает в контейнере; `qemu.sh` и `runner` Cargo заставляют `cargo run` загружать ядро.
- Скрипт компоновщика ставит `.text.boot` первой по адресу `0x4008_0000` и определяет `__bss_start`, `__bss_end` и
  `__stack_top`.
- Ассемблерный код загрузки усыпляет другие ядра, устанавливает `sp`, очищает BSS и вызывает `kernel_main`.
- Устройства — это регистры, отображённые в память; используйте volatile-обращения. `fmt::Write` на крошечном
  драйвере UART даёт вам `println!`.
- Обработчик паники печатает `PanicInfo`; PSCI `SYSTEM_OFF` через `hvc #0` завершает запуск.

## Источники и что почитать

- [QEMU — 'virt' generic virtual platform](https://www.qemu.org/docs/master/system/arm/virt.html) — машина этого
  направления, её устройства и правила загрузки.
- [Arm PrimeCell UART (PL011) Technical Reference Manual](https://developer.arm.com/documentation/ddi0183/latest/) —
  все регистры UART.
- [Arm Power State Coordination Interface (PSCI)](https://developer.arm.com/documentation/den0022/latest/) —
  спецификация `SYSTEM_OFF` и других вызовов.
- [GNU ld — Linker Scripts](https://sourceware.org/binutils/docs/ld/Scripts.html) — язык скриптов компоновщика,
  который понимает и `rust-lld`.
- [The Rust Reference — Inline assembly](https://doc.rust-lang.org/reference/inline-assembly.html) — `asm!` и
  `global_asm!` подробно.
- [`core::ptr::write_volatile`](https://doc.rust-lang.org/core/ptr/fn.write_volatile.html) — точные правила
  volatile-обращений.

---

[🏠 Начало](../START-HERE.md) · [← 22.2 Трек ОС: программа на Rust без операционной системы](02-os-track-a-freestanding-binary.md) · [📋 Этап 22](README.md) · [22.4 Трек ОС: прерывания и страничная память →](04-os-track-interrupts-and-paging.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
