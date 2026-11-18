<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 22.1 Выбор специализации](01-choosing-a-specialization.md) · [📋 Этап 22](README.md) · [22.3 Трек ОС: загрузка ядра в QEMU →](03-os-track-booting-a-kernel-in-qemu.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/22-specializations/02-os-track-a-freestanding-binary.md) <!-- nav -->

# Урок 22.2 — Трек ОС: программа на Rust без операционной системы

> **Этап 22 — Специализации** · Урок 2 из 30 · ⏱ около 2,5 часа (2 занятия)

> **Не ваше направление?** Этот урок открывает направление ОС и ядра (Уроки 22.2–22.5). Если вы выбрали другое
> направление, переходите к [Уроку 22.6 — Трек eBPF: что такое eBPF](06-ebpf-track-what-ebpf-is.md) или сразу к
> [Уроку 22.30 — Контрольная точка: ваша специализация](30-checkpoint-your-specialization.md).

## Что вы узнаете

- Что такое работа над ядром, почему для неё используют Rust и какую машину использует это направление (и почему).
- Что такое автономная программа (freestanding binary) и чем отличаются `std`, `alloc` и `core`.
- Как превратить обычную программу на Rust в программу без операционной системы под ней: `#![no_std]`,
  обработчик паники, `#![no_main]` и собственная точка входа.
- Как собрать программу для целевой платформы «без ОС» и заглянуть внутрь получившегося ELF-файла.
- Как тестировать чистые части кода ядра на Mac.

## Перед началом

- Вы прошли [Урок 22.1 — Выбор специализации](01-choosing-a-specialization.md) и выбрали это направление.
- Вы знаете, что делает ядро ([Урок 13.1](../13-operating-systems/01-what-an-operating-system-does.md)), как
  программа компонуется и загружается ([Урок 12.12](../12-computer-systems/12-linking-and-loading.md)), и читали
  ассемблер arm64 ([Урок 12.7](../12-computer-systems/07-reading-assembly.md)).
- Вы знаете `unsafe` ([Урок 06.6](../06-rust-advanced/06-unsafe-rust-and-miri.md)); следующие уроки используют его часто.
- Docker Desktop работает на вашем Mac (Этап 11); он понадобится с Урока 22.3.

## Новые слова

| Слово | Значение |
|---|---|
| **автономная программа** (freestanding binary) | Программа, которой не нужна операционная система: всё, что ей нужно, она несёт с собой. Ядро — такая программа. |
| **«голое железо»** (bare metal) | Работа прямо на оборудовании, без операционной системы под вашим кодом. |
| **`std`** | Стандартная библиотека Rust. В ней файлы, потоки, `println!` и многое другое, и ей нужна ОС. |
| **`core`** | Часть стандартной библиотеки, которой ничего не нужно от ОС: `Option`, `Result`, итераторы, `fmt`, срезы. |
| **`alloc`** | Часть с типами в куче (`Box`, `Vec`, `String`). Ей нужен аллокатор памяти, но не ОС. |
| **`#![no_std]`** | Атрибут крейта: «компоновать только `core`, без `std`». |
| **обработчик паники** (panic handler) | Функция, которая выполняется при панике. С `std` она готовая; без `std` её нужно написать самому. |
| **точка входа** (entry point) | Адрес, с которого CPU (Central Processing Unit, центральный процессор) начинает выполнять программу. |
| **тройка целевой платформы** (target triple) | Имя платформы, для которой вы компилируете, например `aarch64-apple-darwin`: процессор, производитель, операционная система. |
| **кросс-компиляция** (cross-compiling) | Сборка на одной платформе (вашем Mac) программы для другой платформы («голой» машины arm64). |
| **ELF** (Executable and Linkable Format, формат исполняемых и компонуемых файлов) | Формат программ в Linux и большинства ядер (Урок 12.12). |
| **softfloat** | Вариант целевой платформы, который никогда не использует регистры для чисел с плавающей точкой и SIMD (Single Instruction, Multiple Data, «одна инструкция — много данных»). |
| **эмулятор** (emulator) | Программа, которая изображает другой компьютер. В этом направлении используется QEMU (Quick Emulator, быстрый эмулятор). |
| **llvm-tools** | Компонент rustup с инструментами LLVM (набора средств компилятора, на котором построен `rustc`) для просмотра двоичных файлов. |

**Другие сокращения в этом уроке:** ОС (операционная система, operating system: программа, которая управляет
оборудованием и запускает другие программы, Этап 13) и ПК (персональный компьютер, PC: здесь обычный настольный
компьютер или ноутбук x86-64).

## 1. Направление ОС и машина, которую мы используем

Инженеры ядра пишут код, который работает *под* всеми остальными программами: код, который запускает машину,
управляет устройствами, обрабатывает прерывания и распределяет память. Rust хорошо подходит для этой работы: у
него нет сборщика мусора и среды выполнения, которой нужна ОС, он даёт точный контроль над размещением в памяти, а
его проверки убирают целые классы ошибок памяти из большей части кода. `unsafe` остаётся в маленьких, проверенных
местах. Сегодня Rust используется в ядре Linux, в Android, в компонентах Windows и в исследовательских ядрах.

За четыре урока этого направления вы шаг за шагом построите маленькое ядро:

| Урок | Что ваше ядро умеет после него |
|---|---|
| 22.2 (этот) | Оно существует: автономная программа, которая собирается для машины без ОС. |
| 22.3 | Оно загружается в QEMU и печатает текст по последовательной линии. |
| 22.4 | Оно обрабатывает исключения и прерывания таймера и использует свои таблицы страниц. |
| 22.5 | Проект: маленькое интерактивное ядро и что делать дальше. |

**Какая машина?** Ядро пишется для одного вида машины. Это направление использует **машину `virt` из QEMU с
процессором arm64** (arm64 также называют AArch64). Причины:

- В вашем Mac процессор arm64. Инструкции и регистры — те самые, что вы читали в Уроке 12.7.
- `virt` — простая, хорошо документированная машина, созданная для эмуляторов. QEMU загружает наш ELF-файл прямо
  в память и переходит на его точку входа, поэтому нам **не нужны ни загрузчик, ни образ диска**.
- Нужен только стабильный Rust. Целевая платформа `aarch64-unknown-none-softfloat` готова в rustup.

Другой распространённый выбор — ПК x86-64 с крейтом `bootloader` (версия 0.11), который использует популярный блог
*Writing an OS in Rust*. Это тоже хороший путь; Урок 22.5 называет его как один из следующих шагов.

**Как устроен код.** Всё направление работает в одном пакете, `code/p22-kernel`. Каждый урок немного его
меняет, а проект Урока 22.5 его продолжает. Делайте коммит после каждого урока.

## 2. Что нужно `std` и что даёт `core`

`println!` работает в обычной программе, потому что `std` просит операционную систему записать в стандартный
вывод. С файлами, потоками, временем, сетью и даже запуском `main` то же самое: `std` — слой поверх ОС.

```text
   your program
        |
   std  (files, threads, println!, process start)   <- needs an operating system
        |
   alloc (Box, Vec, String)                        <- needs an allocator
        |
   core  (Option, Result, iterators, fmt, slices)   <- needs nothing
        |
   hardware
```

Ядро само *является* операционной системой, поэтому под ним нет никого, кто ответил бы `std`. Оно может
использовать только `core` (и позже `alloc`, когда у него появится свой аллокатор — вы писали его в Уроке 12.14).
Всё остальное вы пишете сами. Этим и занимается урок, по одной ошибке компилятора за раз.

## 3. Шаг 1: `#![no_std]`

Создайте пакет на Mac:

```bash
cd ~/projects/rust
cargo new code/p22-kernel
cd code/p22-kernel
```

Замените `src/main.rs` обычной программой, к которой добавлена одна строка в начале:

```rust
#![no_std]

fn main() {
    println!("Hello, world!");
}
```

`#![no_std]` (с `!`, поэтому он относится ко всему крейту) говорит компилятору: не компоновать `std`, только
`core`. Соберите через `cargo build`:

```text
error: cannot find macro `println` in this scope
 --> src/main.rs:4:5
  |
4 |     println!("Hello, world!");
  |     ^^^^^^^

error: `#[panic_handler]` function required, but not found

error: unwinding panics are not supported without std
  |
  = help: using nightly cargo, use -Zbuild-std with panic="abort" to avoid unwinding
  = note: since the core library is usually precompiled with panic="unwind", rebuilding your crate with panic="abort" may not be enough to fix the problem

error: could not compile `p22-kernel` (bin "p22-kernel") due to 3 previous errors
```

**Что происходит:**

1. `println!` живёт в `std`, поэтому его больше нет. Для печати нужно устройство, на которое печатать; Урок 22.3
   пишет для него драйвер.
2. Паника должна *где-то* закончиться. Обычно `std` печатает сообщение и останавливает поток. Без `std`
   компилятор просит у вас функцию с пометкой `#[panic_handler]`.
3. По умолчанию паника **раскручивает стек** (unwinding): идёт назад по стеку и вызывает `drop` для каждого
   значения. Для раскрутки нужен вспомогательный код из `std`. Ядро вместо этого просто останавливается, это
   называется **abort** (аварийное завершение).

## 4. Шаг 2: обработчик паники и `panic = "abort"`

Добавьте обработчик и уберите `println!`:

```rust
#![no_std]

use core::panic::PanicInfo;

#[panic_handler]
fn panic(_info: &PanicInfo) -> ! {
    loop {}
}

fn main() {}
```

Затем добавьте эти строки в конец `Cargo.toml`, чтобы оба профиля сборки завершались аварийно при панике:

```toml
[profile.dev]
panic = "abort"

[profile.release]
panic = "abort"
```

**Что происходит:**

1. `#[panic_handler]` отмечает единственную во всей программе функцию, которую вызывает паника.
2. Она получает `&PanicInfo` (сообщение и место в исходном коде) и возвращает `!`, тип «никогда»: эта функция
   никогда не возвращает управление. Пока она просто крутится в бесконечном цикле.
3. `panic = "abort"` убирает раскрутку стека, поэтому третья ошибка исчезает.

Теперь `cargo build` выдаёт новую ошибку:

```text
error: using `fn main` requires the standard library
  |
  = help: use `#![no_main]` to bypass the Rust generated entrypoint and declare a platform specific entrypoint yourself, usually with `#[no_mangle]`
```

`main` на самом деле не первая функция программы. В macOS и Linux сначала выполняется код запуска из библиотеки C
и `std`: он готовит аргументы и окружение и только потом вызывает ваш `main`. Без `std` этого кода нет.

## 5. Шаг 3: `#![no_main]` и собственная точка входа

Снова замените `src/main.rs`:

```rust
#![no_std]
#![no_main]

use core::panic::PanicInfo;

#[unsafe(no_mangle)]
pub extern "C" fn _start() -> ! {
    loop {}
}

#[panic_handler]
fn panic(_info: &PanicInfo) -> ! {
    loop {}
}
```

**Что происходит:**

1. `#![no_main]` говорит: нет ни `main`, ни сгенерированного кода запуска.
2. `_start` — общепринятое имя точки входа. Компоновщик (linker) ищет символ ровно с этим именем.
3. `#[unsafe(no_mangle)]` сохраняет имя `_start` в двоичном файле. Обычно компилятор **искажает** (mangles) имена,
   превращая их в длинные уникальные строки (одну такую вы увидите в разделе 7); искажённое имя компоновщик не
   нашёл бы. Атрибут помечен `unsafe`, потому что две функции с одинаковым неискажённым именем столкнулись бы.
4. `extern "C"` использует соглашение о вызовах C (Урок 06.10) — простой и стабильный способ, которым машинный код
   вызывает функции.
5. `-> !`: возвращаться некуда, поэтому функция не должна возвращать управление никогда.

`cargo build` всё ещё падает, но теперь в **компоновщике**:

```text
error: linking with `cc` failed: exit status: 1
  |
  = note: Undefined symbols for architecture arm64:
            "_main", referenced from:
                <initial-undefines>
          ld: symbol(s) not found for architecture arm64
          clang: error: linker command failed with exit code 1 (use -v to see invocation)
```

Компилятор свою работу сделал. Но Cargo по умолчанию собирает для вашего Mac (целевая платформа
`aarch64-apple-darwin`), а компоновщик macOS требует программу с `main`, работающую под macOS. Программа для macOS
нам не нужна.

## 6. Шаг 4: собираем для машины без ОС

**Тройка целевой платформы** называет платформу, для которой идёт сборка. Прочитайте
`aarch64-unknown-none-softfloat` слева направо:

| Часть | Значение |
|---|---|
| `aarch64` | Процессор: 64-битный Arm, то же семейство, что и процессор вашего Mac. |
| `unknown` | Производитель: никто конкретный. |
| `none` | Операционная система: **никакой**. Именно это делает платформу «голой». |
| `softfloat` | Никогда не использовать регистры для чисел с плавающей точкой и SIMD. |

Зачем `softfloat`? Когда прерывание останавливает ядро, ядро должно сохранить каждый регистр, который может
изменить его обработчик (Урок 22.4). Если ядро никогда не трогает 32 больших регистра для чисел с плавающей
точкой, ему никогда не нужно их сохранять. Большинство ядер следуют этому правилу; плавающая точка — для
пользовательских программ.

Rust поставляет готовый `core` для этой платформы, поэтому её достаточно добавить один раз (это ваш обычный rustup;
команда больше ничего не меняет):

```bash
rustup target add aarch64-unknown-none-softfloat
rustup target list --installed
```

Пример вывода (у вас может быть больше платформ):

```text
aarch64-apple-darwin
aarch64-unknown-none-softfloat
```

Теперь соберите для неё:

```bash
cargo build --target aarch64-unknown-none-softfloat
file target/aarch64-unknown-none-softfloat/debug/p22-kernel
```

```text
   Compiling p22-kernel v0.1.0 (/Users/student/projects/rust/code/p22-kernel)
    Finished `dev` profile [unoptimized + debuginfo] target(s) in 0.23s
target/aarch64-unknown-none-softfloat/debug/p22-kernel: ELF 64-bit LSB executable, ARM aarch64, version 1 (SYSV), statically linked, with debug_info, not stripped
```

Это и есть кросс-компиляция: ваш Mac создал ELF-файл для машины arm64 без ОС. Компоновщик — `rust-lld`, он идёт
вместе с Rust, так что больше ничего не нужно. Запустить этот файл на Mac нельзя; Урок 22.3 загрузит его в QEMU.

Набирать `--target` каждый раз утомительно. Создайте в пакете `.cargo/config.toml` (файл настроек, который Cargo
читает для каждой команды в этой папке):

```toml
[build]
target = "aarch64-unknown-none-softfloat"
```

Теперь `cargo build` собирает ядро.

> 💡 **Совет:** RustRover тоже читает `.cargo/config.toml`, поэтому автодополнение и ошибки в редакторе теперь
> используют «голую» платформу. Если редактор всё ещё показывает элементы `std`, перезагрузите проект Cargo.

## 7. Заглядываем внутрь двоичного файла

Компонент `llvm-tools` содержит инструменты, которые понимают ELF-файлы на любом компьютере. Установите его и
запомните, где лежат инструменты:

```bash
rustup component add llvm-tools
LLVM_BIN="$(rustc --print sysroot)/lib/rustlib/aarch64-apple-darwin/bin"
ls "$LLVM_BIN"
```

`rustc --print sysroot` печатает папку активного тулчейна; инструменты лежат в её подпапке. Среди файлов вы
увидите `llvm-objdump`, `llvm-readobj`, `llvm-nm` и `llvm-size`.

Прежде чем смотреть, дайте `_start` какую-нибудь работу. Чистый вспомогательный код помещается в **библиотеку**
в том же пакете, чтобы позже его можно было тестировать. Создайте `src/lib.rs`:

```rust
//! Code that does not touch hardware. It builds for the kernel
//! and can be tested on your Mac.
#![cfg_attr(not(test), no_std)]

/// Rounds `addr` up to the next multiple of `align` (a power of two).
pub fn align_up(addr: usize, align: usize) -> usize {
    assert!(align.is_power_of_two(), "align must be a power of two");
    (addr + align - 1) & !(align - 1)
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn rounds_up_to_the_next_page() {
        assert_eq!(align_up(0x4008_0001, 4096), 0x4008_1000);
    }

    #[test]
    fn keeps_aligned_addresses() {
        assert_eq!(align_up(0x4008_0000, 4096), 0x4008_0000);
        assert_eq!(align_up(0, 16), 0);
    }
}
```

И замените `src/main.rs`:

```rust
#![no_std]
#![no_main]

use core::panic::PanicInfo;
use p22_kernel::align_up;

/// The first Rust code of the kernel. Nobody calls it from Rust: the machine jumps here.
#[unsafe(no_mangle)]
pub extern "C" fn _start() -> ! {
    let _first_page = align_up(0x4008_0001, 4096);
    halt()
}

/// Called by `panic!` and by failed checks such as an index out of bounds.
#[panic_handler]
fn panic(_info: &PanicInfo) -> ! {
    halt()
}

/// Stops here forever, telling the CPU that we are only waiting.
fn halt() -> ! {
    loop {
        core::hint::spin_loop();
    }
}
```

**Что происходит:**

1. `align_up` округляет адрес вверх до границы; ядрам это нужно постоянно (страницы по 4096 байт). Так как `align`
   — степень двойки, `align - 1` — маска младших битов: `+ (align - 1)` проталкивает адрес за следующую границу, а
   `& !(align - 1)` очищает младшие биты (Урок 12.4).
2. `#![cfg_attr(not(test), no_std)]` означает: «будь `no_std`, кроме сборки для тестов». Тестам нужна тестовая
   обвязка из `std`; ядро `std` не получает.
3. `main.rs` использует библиотеку как `p22_kernel::align_up` (дефис в имени пакета становится `_`).
4. `halt` заменяет пустой `loop {}`. `core::hint::spin_loop()` вставляет инструкцию, которая говорит процессору
   «я только жду»; Clippy предупреждает о пустом цикле.

Соберите в режиме release и посмотрите машинный код. `-d` дизассемблирует, `-C` превращает искажённые имена
обратно в читаемые имена Rust:

```bash
cargo build --release
"$LLVM_BIN/llvm-objdump" -d -C target/aarch64-unknown-none-softfloat/release/p22-kernel
```

```text
Disassembly of section .text:

0000000000210244 <__rustc::rust_begin_unwind>:
  210244: d5033fdf     	isb
  210248: 17ffffff     	b	0x210244 <__rustc::rust_begin_unwind>

000000000021024c <_start>:
  21024c: f81f0ffe     	str	x30, [sp, #-0x10]!
  210250: 52800020     	mov	w0, #0x1                // =1
  210254: 52820001     	mov	w1, #0x1000             // =4096
  210258: 72a80100     	movk	w0, #0x4008, lsl #16
  21025c: 94000003     	bl	0x210268 <p22_kernel::align_up>
  210260: d5033fdf     	isb
  210264: 17ffffff     	b	0x210260 <_start+0x14>

0000000000210268 <p22_kernel::align_up>:
  210268: d1000428     	sub	x8, x1, #0x1
  21026c: ca080029     	eor	x9, x1, x8
  210270: eb08013f     	cmp	x9, x8
  210274: 540000c9     	b.ls	0x21028c <p22_kernel::align_up+0x24>
  ...
```

Прочитайте его с тем, что вы знаете из Урока 12.7:

1. Наш обработчик паники получил внутреннее имя `rust_begin_unwind`. Это цикл `isb` + `b`: `spin_loop()`
   превратился в инструкцию `isb`.
2. `_start` кладёт `0x4008_0001` в `w0` (двумя половинами, `mov` и `movk`), `4096` в `w1` и вызывает `align_up`
   через `bl`. Потом бесконечно крутится в цикле.
3. `align_up` сначала проверяет «степень двойки»; если проверка не прошла, он прыгает в код, который вызывает
   механизм паники, а тот заканчивается в нашем обработчике. Даже `assert!` работает без `std`.
4. Посмотрите на самую первую инструкцию `_start`: `str x30, [sp, #-0x10]!` сохраняет регистр **в стек**. Но
   стек никто не настроил! На настоящей машине в `sp` здесь случайное значение. Урок 22.3 исправит это несколькими
   инструкциями ассемблера, которые выполняются до любого кода на Rust.

Заголовок ELF говорит, где начинается выполнение:

```bash
"$LLVM_BIN/llvm-readobj" --file-headers target/aarch64-unknown-none-softfloat/release/p22-kernel
```

```text
  Type: Executable (0x2)
  Machine: EM_AARCH64 (0xB7)
  Version: 1
  Entry: 0x21024C
```

`Entry` — ровно адрес `_start` в дизассемблированном коде. Адрес `0x21024C` компоновщик выбрал сам. На машине
`virt` из QEMU память начинается с `0x4000_0000`, поэтому там этот адрес бесполезен; в Уроке 22.3 расположение
всего определит **скрипт компоновщика** (linker script).

## 8. Тестируем чистые части на Mac

На платформе ядра нет `std`, поэтому тестовая обвязка там не работает. Но `src/lib.rs` собирается для любой
платформы, поэтому его тесты можно запустить на Mac, указав платформу Mac:

```bash
cargo test --lib --target aarch64-apple-darwin
```

```text
running 2 tests
test tests::rounds_up_to_the_next_page ... ok
test tests::keeps_aligned_addresses ... ok

test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

`--lib` собирает только библиотеку (двоичный файл ядра не может быть тестом), а `--target` переопределяет
платформу из `.cargo/config.toml`. Сократите это **псевдонимом** (alias) Cargo: замените `.cargo/config.toml` на

```toml
[build]
target = "aarch64-unknown-none-softfloat"

[alias]
# Запускает тесты src/lib.rs на Mac, а не на платформе ядра.
host-test = "test --lib --target aarch64-apple-darwin"
```

Теперь `cargo host-test` запускает тесты, а `cargo build` собирает ядро.

> 🧪 **Тестирование:** это главное правило тестирования во всём направлении: код, работающий с оборудованием,
> остаётся маленьким, а все вычисления (адреса, битовые поля, разбор) живут в `src/lib.rs` с обычными модульными
> тестами. Тестировать внутри QEMU тоже можно; Урок 22.5 рассказывает как.

## Частые ошибки

> ⚠️ **Частая ошибка:** забыть `#[unsafe(no_mangle)]` на `_start`. Сборка только предупреждает, а результат
> бесполезен:
>
> ```text
> warning: linker stderr: rust-lld: cannot find entry symbol _start; not setting start address
> ```
>
> Функция получила искажённое имя, компоновщик не нашёл `_start`, поставил точку входа в `0x0` и удалил
> «неиспользуемый» код. В работе над ядром всегда читайте предупреждения компоновщика.

> ⚠️ **Частая ошибка:** запускать в пакете ядра просто `cargo test`. Он собирает тесты для платформы ядра, где
> тестовой обвязки нет:
>
> ```text
> error[E0463]: can't find crate for `std`
> error[E0463]: can't find crate for `test`
> ```
>
> Используйте `cargo host-test` (раздел 8).

> ⚠️ **Частая ошибка:** писать `std::` в коде ядра, например `std::ptr::write_volatile`. Большинство таких
> элементов есть и в `core`: пишите `core::ptr::write_volatile`.

> ⚠️ **Частая ошибка:** использовать платформу `aarch64-unknown-none` (без `softfloat`), а потом код с плавающей
> точкой. Он собирается, но первая инструкция с плавающей точкой на настоящей машине вызывает исключение, пока
> ядро не включит блок операций с плавающей точкой. Это направление обходит проблему платформой `softfloat`.

## Попробуйте сами

1. Пройдите разделы 3–6 в точности и читайте каждую ошибку, прежде чем её исправлять.
2. Запустите `cargo host-test` и `cargo clippy`; оба должны пройти чисто.
3. Сделайте коммит: `git add code/p22-kernel && git commit -m "p22-kernel: freestanding binary"`.

## Упражнения

### Упражнение 1 — Насколько велико ядро? (легко)

Запустите `"$LLVM_BIN/llvm-size"` для отладочной и release-сборки. Сколько байтов кода (`text`) в каждой? Затем
сравните с размером файлов на диске (`ls -l`). Почему файлы намного больше?

<details>
<summary>Подсказка</summary>

Столбец `text` — это машинный код и константные данные. Отладочные сборки содержат ещё и отладочную информацию,
которая в память не загружается.

</details>

<details>
<summary>Ответ</summary>

```text
   text	   data	    bss	    dec	    hex	filename
    917	      0	      0	    917	    395	target/aarch64-unknown-none-softfloat/debug/p22-kernel
    372	      0	      0	    372	    174	target/aarch64-unknown-none-softfloat/release/p22-kernel
```

`ls -l` показывает около 713 КБ для отладочного файла и около 8 КБ для release-файла. Лишние байты — отладочная
информация и таблица символов: они полезны отладчикам, но никогда не загружаются в память машины. Само ядро —
несколько сотен байтов. (Ваши числа могут немного отличаться.)

</details>

### Упражнение 2 — `is_aligned` с тестом (средне)

Добавьте `pub fn is_aligned(addr: usize, align: usize) -> bool` в `src/lib.rs` и тест `checks_alignment` минимум
с тремя проверками. Запустите его через `cargo host-test`.

<details>
<summary>Подсказка</summary>

Адрес выровнен, когда его младшие биты (биты внутри `align - 1`) все равны нулю.

</details>

<details>
<summary>Ответ</summary>

```rust
/// Is `addr` a multiple of `align` (a power of two)?
pub fn is_aligned(addr: usize, align: usize) -> bool {
    addr & (align - 1) == 0
}
```

И внутри `mod tests`:

```rust
    #[test]
    fn checks_alignment() {
        assert!(is_aligned(0x4008_0000, 4096));
        assert!(!is_aligned(0x4008_0008, 4096));
        assert!(is_aligned(0x4008_0008, 8));
    }
```

Почему это работает: `0x4008_0008 & 0xfff` равно `8`, а не нулю, поэтому адрес не выровнен по странице, но
`8 & 7` равно `0`, поэтому он выровнен по 8 байтам.

</details>

### Упражнение 3 — `align_down`, проверенный на десяти тысячах адресов (сложно)

Добавьте `align_down` (округление вниз до кратного `align`, с той же проверкой степени двойки). Затем напишите два
теста: один проходит по адресам `0..10_000` и проверяет, что `align_down(a, 64) <= a <= align_up(a, 64)`, что оба
результата выровнены и что они либо равны, либо отличаются на 64; второй — тест `#[should_panic]` для `align = 3`.

<details>
<summary>Подсказка</summary>

Округление вниз только очищает младшие биты: `addr & !(align - 1)`. `#[should_panic(expected = "power of two")]`
проверяет, что сообщение паники содержит этот текст (Урок 05.6).

</details>

<details>
<summary>Ответ</summary>

```rust
/// Rounds `addr` down to a multiple of `align` (a power of two).
pub fn align_down(addr: usize, align: usize) -> usize {
    assert!(align.is_power_of_two(), "align must be a power of two");
    addr & !(align - 1)
}
```

```rust
    #[test]
    fn align_down_and_up_surround_every_address() {
        for addr in 0..10_000 {
            let down = align_down(addr, 64);
            let up = align_up(addr, 64);
            assert!(down <= addr && addr <= up, "addr {addr}");
            assert!(is_aligned(down, 64) && is_aligned(up, 64));
            assert!(up - down == 0 || up - down == 64);
        }
    }

    #[test]
    #[should_panic(expected = "power of two")]
    fn rejects_an_alignment_that_is_not_a_power_of_two() {
        align_down(100, 3);
    }
```

```text
test tests::rejects_an_alignment_that_is_not_a_power_of_two - should panic ... ok

test result: ok. 5 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

Почему это работает: тест, который проверяет *свойство* на многих входах, находит ошибки, которые пропускают два-три
подобранных вручную примера (Урок 09.4 делает это с библиотекой). Он использует `is_aligned` из Упражнения 2.

</details>

## Вопросы для самопроверки

1. Чем отличаются `core`, `alloc` и `std`?

<details>
<summary>Ответ</summary>

`core` ничего не требует (типы, трейты, итераторы, форматирование). `alloc` добавляет типы в куче и требует
аллокатор памяти. `std` добавляет файлы, потоки, печать и запуск процесса и требует операционную систему. Ядро может
использовать `core` всегда, а `alloc` — когда у него есть аллокатор.

</details>

2. Зачем программе с `#![no_std]` нужен `#[panic_handler]`?

<details>
<summary>Ответ</summary>

Каждая паника должна закончиться в какой-то функции. Обычно её даёт `std` (напечатать и остановить поток). Без
`std` такой функции нет, поэтому вы пишете её сами; она не должна возвращать управление (`-> !`).

</details>

3. Почему мы пишем `#![no_main]` и `_start` вместо `fn main`?

<details>
<summary>Ответ</summary>

`main` вызывается кодом запуска из библиотеки C и `std`, которому нужна операционная система. У ядра такого кода
нет; машина переходит прямо на точку входа, которую мы задаём как `_start`.

</details>

4. Что делает `#[unsafe(no_mangle)]` и что будет без него на `_start`?

<details>
<summary>Ответ</summary>

Он сохраняет точное имя символа `_start` вместо искажённого. Без него компоновщик не находит `_start`, ставит точку
входа в `0x0` и удаляет код, который теперь считает неиспользуемым.

</details>

5. Дизассемблированный `_start` начинается с `str x30, [sp, #-0x10]!`. Почему это проблема на настоящем оборудовании?

<details>
<summary>Ответ</summary>

Инструкция пишет в стек, но в точке входа никто не настроил указатель стека `sp`, поэтому запись идёт по случайному
адресу. Ядро должно настроить стек на ассемблере до запуска любого кода на Rust (Урок 22.3).

</details>

## Итоги

- Ядро — автономная программа: под ним нет операционной системы, поэтому оно использует только `core` (и позже
  `alloc`).
- `#![no_std]` убирает `std`; после этого нужны `#[panic_handler]` и `panic = "abort"`.
- `#![no_main]` убирает код запуска; `#[unsafe(no_mangle)] pub extern "C" fn _start() -> !` — точка входа.
- Платформа `aarch64-unknown-none-softfloat` означает arm64, без ОС, без регистров с плавающей точкой; задайте её в
  `.cargo/config.toml`.
- `llvm-objdump -d -C` и `llvm-readobj --file-headers` из `llvm-tools` показывают код и точку входа.
- Чистая логика живёт в `src/lib.rs` с `#![cfg_attr(not(test), no_std)]` и тестируется на Mac через
  `cargo host-test`.

## Источники и что почитать

- [The Embedded Rust Book — A `no_std` Rust Environment](https://docs.rust-embedded.org/book/intro/no-std.html) — что
  дают `core`, `alloc` и `std`.
- [The Rust Reference — the `panic_handler` attribute](https://doc.rust-lang.org/reference/panic.html) — точные правила
  для обработчиков паники.
- [rustc book — aarch64-unknown-none](https://doc.rust-lang.org/rustc/platform-support/aarch64-unknown-none.html) —
  официальная страница «голых» платформ arm64.
- [Документация библиотеки `core`](https://doc.rust-lang.org/core/) — всё, что ядро может использовать без `std`.
- [Writing an OS in Rust — A Freestanding Rust Binary](https://os.phil-opp.com/freestanding-rust-binary/) — тот же
  первый шаг для x86-64 в известной серии статей.

---

[🏠 Начало](../START-HERE.md) · [← 22.1 Выбор специализации](01-choosing-a-specialization.md) · [📋 Этап 22](README.md) · [22.3 Трек ОС: загрузка ядра в QEMU →](03-os-track-booting-a-kernel-in-qemu.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
