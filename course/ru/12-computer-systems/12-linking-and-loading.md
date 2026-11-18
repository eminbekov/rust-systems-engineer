<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 12.11 Размещение типов Rust в памяти](11-memory-layout-of-rust-types.md) · [📋 Этап 12](README.md) · [12.13 Как работают аллокаторы памяти →](13-how-memory-allocators-work.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/12-computer-systems/12-linking-and-loading.md) <!-- nav -->

# Урок 12.12 — Компоновка и загрузка программ

> **Этап 12 — Компьютерные системы** · Урок 12 из 15 · ⏱ около 135 минут

## Что вы узнаете

- Четыре шага от исходного кода до работающего процесса: компиляция, объектные файлы, компоновка, загрузка.
- Что такое символы, как вывести их список через `nm` и зачем Rust искажает (mangle) имена.
- Что делает компоновщик (linker), что такое статическая и динамическая компоновка и как читать ошибку «undefined symbol».
- Как исследовать исполняемые файлы: Mach-O в macOS с помощью `file`, `otool` и `size`; ELF в Linux с помощью `readelf`, `ldd` и `objdump`.
- Что делает загрузчик при старте программы и почему адреса меняются при каждом запуске.

## Перед началом

- Вы прошли [Урок 12.11 — Размещение типов Rust в памяти](11-memory-layout-of-rust-types.md).
- Вы помните «от исходного кода до работающей программы» из [Урока 00.3](../00-getting-started/03-how-a-computer-runs-a-program.md), FFI (Foreign Function Interface, интерфейс внешних функций) и `extern "C"` из [Урока 06.10 — FFI: вызов кода на C](../06-rust-advanced/10-ffi-calling-c.md), а также контейнеры Docker из [Урока 11.12](../11-linux-shell-and-containers/12-containers-and-docker-basics.md).
- Создайте пакет и поместите эту программу в `src/main.rs`. Она понадобится на протяжении всего урока:

```bash
cd ~/projects/rust
cargo new code/p12-linking
cd code/p12-linking
```

```rust
static GREETING: &str = "hello from a static";

#[inline(never)]
fn add_numbers(a: u64, b: u64) -> u64 {
    a + b
}

fn main() {
    let sum = add_numbers(2, 40);
    println!("{GREETING}: {sum}");
    println!("main is at        {:p}", main as fn() as *const ());
    println!("add_numbers is at {:p}", add_numbers as fn(u64, u64) -> u64 as *const ());
    println!("GREETING is at    {:p}", &GREETING);
    let local = 7;
    println!("a local is at     {:p}", &local);
}
```

`#[inline(never)]` сохраняет `add_numbers` настоящей функцией, чтобы потом её можно было найти. `{:p}` печатает адрес; `main as fn() as *const ()` превращает функцию в простой указатель, чтобы его можно было напечатать.

## Новые слова

| Слово | Значение |
|---|---|
| **объектный файл** (object file) | Результат компиляции одной части кода: машинный код и данные плюс список имён, которые он определяет, и имён, которые ему ещё нужны. Пока не запускается. |
| **символ** (symbol) | Имя функции или статической переменной в объектном или исполняемом файле вместе с её адресом. |
| **таблица символов** (symbol table) | Список всех символов файла. Её печатает `nm`. |
| **искажение имён** (name mangling) | Кодирование полного пути Rust (крейт, модуль, обобщения) в одно уникальное имя символа, например `_RNvCs…11p12_linking11add_numbers`. |
| **компоновщик** (linker) | Программа, которая объединяет объектные файлы и библиотеки в один исполняемый файл и проставляет все недостающие адреса. |
| **перемещение** (relocation) | Запись в объектном файле: «подставь сюда итоговый адрес символа X». Её обрабатывает компоновщик (или загрузчик). |
| **статическая компоновка** (static linking) | Копирование нужного кода библиотеки в исполняемый файл во время сборки. |
| **динамическая компоновка** (dynamic linking) | Код библиотеки остаётся в отдельном файле разделяемой библиотеки, который находят и подключают при старте программы. |
| **разделяемая библиотека** (shared library) | Файл библиотеки, которым одновременно пользуются многие программы: `.dylib` в macOS, `.so` (shared object, разделяемый объект) в Linux. |
| **загрузчик** (loader, dynamic linker) | Программа, которая запускается первой, когда вы запускаете динамически скомпонованный исполняемый файл: `dyld` в macOS, `ld.so` в Linux. Она загружает разделяемые библиотеки и связывает символы. |
| **Mach-O** (Mach object, объект Mach) | Формат исполняемых файлов, библиотек и объектных файлов в macOS. |
| **ELF** (Executable and Linkable Format, формат исполняемых и компонуемых файлов) | Формат исполняемых файлов, библиотек и объектных файлов в Linux и большинстве других Unix-систем. |
| **сегмент / секция** (segment / section) | Части исполняемого файла. С секциями работает компоновщик (`.text` для кода, `.data` для данных); сегменты загрузчик отображает в память. |
| **ASLR** (Address Space Layout Randomization, рандомизация размещения адресного пространства) | Операционная система загружает программу по новому случайному адресу при каждом запуске, что усложняет атаки. |
| **PIE** (Position-Independent Executable, позиционно-независимый исполняемый файл) | Исполняемый файл, код которого работает по любому адресу, поэтому ASLR может его сдвигать. Rust по умолчанию собирает PIE. |
| **rlib** (Rust library, библиотека Rust) | Формат архива, в котором Cargo хранит скомпилированные крейты Rust; внутри объектные файлы и метаданные. |

## 1. От исходного кода до работающего процесса

Сборка и запуск программы на Rust проходят четыре шага:

```text
  src/main.rs ──► rustc ──► object files (.o) ─┐
  std, core, …  (already compiled, .rlib)  ────┼──► linker (cc / ld) ──► executable file
  system C library (libSystem, libc)  ─────────┘                          │
                                                               you run it ▼
                                                    loader (dyld / ld.so) ──► process
```

1. **Компиляция.** `rustc` превращает каждый крейт в машинный код, разрезанный на один или несколько объектных файлов. На этом этапе вызов `println!` — всё ещё только имя: код `std` лежит в другом месте.
2. **Объектные файлы и библиотеки.** Стандартная библиотека была скомпилирована при установке Rust и лежит в файлах `.rlib`. Библиотека C операционной системы — разделяемая библиотека.
3. **Компоновка.** Компоновщик берёт все объектные файлы и библиотеки, собирает нужные части вместе, назначает итоговый адрес каждой функции и статической переменной и исправляет каждое место, которое на них ссылается. Результат — исполняемый файл.
4. **Загрузка.** Когда вы запускаете программу, операционная система и загрузчик читают этот файл, отображают его в память, подключают разделяемые библиотеки и переходят к его первой инструкции.

Шаги 1–3 Cargo выполняет за вас при каждом `cargo build`. В macOS и в Linux на arm64 Rust вызывает системный драйвер компилятора C (`cc`), а тот — системный компоновщик (`ld`). (В Linux на x86-64 свежие версии Rust просят `cc` использовать вместо него `rust-lld` — более быстрый компоновщик, который устанавливается вместе с Rust.)

## 2. Объектные файлы и символы

Попросите `rustc` остановиться после шага 1 и записать объектный файл `src/main.rs`:

```bash
mkdir obj
rustc --edition 2024 -C opt-level=3 --emit obj -o obj/main.o src/main.rs
file obj/main.o
nm --demangle obj/main.o | grep -E ' (T|t|U|s) ' | grep -v -E 'l_anon|ltmp'
```

```text
obj/main.o: Mach-O 64-bit object arm64
0000000000000000 T std::rt::lang_start::<()>
0000000000000038 t std::sys::backtrace::__rust_begin_short_backtrace::<fn(), ()>
000000000000004c t std::rt::lang_start::<()>::{closure#0}
0000000000000068 t <std::rt::lang_start<()>::{closure#0} as core::ops::function::FnOnce<()>>::call_once::{shim:vtable#0}
0000000000000084 t main::add_numbers
000000000000008c T main::main
0000000000000228 s main::GREETING
                 U std::rt::lang_start_internal
                 U core::fmt::pointer_fmt_inner
                 U std::io::stdio::_print
0000000000000180 t <&str as core::fmt::Display>::fmt
                 U <u64 as core::fmt::Display>::fmt
                 U <str as core::fmt::Display>::fmt
0000000000000190 t <*const () as core::fmt::Pointer>::fmt
0000000000000198 t <&&str as core::fmt::Pointer>::fmt
00000000000001a0 t <&i32 as core::fmt::Pointer>::fmt
00000000000001a8 T _main
```

**Что происходит:**

1. `file` сообщает, что это за файл: объектный файл (не исполняемый) для arm64 в формате Mach-O. При компиляции отдельно от Cargo крейт называется `main`, по имени файла.
2. `nm` выводит **таблицу символов**. В каждой строке адрес (внутри этого объектного файла, начиная с 0), буква вида символа и имя. `--demangle` превращает искажённые имена обратно в читаемые пути Rust. Две команды `grep` лишь прячут внутренние метки компилятора.
3. `T` / `t`: код (в секции «text»), определённый здесь; заглавная буква — виден другим файлам, строчная — локальный. `s`: данные в какой-то другой секции, здесь наш `GREETING`.
4. `U`: **не определён** (undefined). Этот файл вызывает `std::io::stdio::_print`, но не содержит её. Заполнять такие дыры — работа компоновщика.
5. `_main` — точка входа, которую вызывает среда выполнения C; её генерирует `rustc`, и она вызывает `std::rt::lang_start`, которая подготавливает стандартную библиотеку и затем вызывает наш `main::main`.

Без `--demangle` имена выглядят так (из `nm obj/main.o`):

```text
                 U __RNvNtCsi4LIHK2nfKK_3std2rt19lang_start_internal
                 U __RNvNtNtCsi4LIHK2nfKK_3std2io5stdio6__print
```

Это и есть **искажение имён**. В двух крейтах может быть по функции `add`, обобщённая функция существует в одном экземпляре на каждый тип, а компоновщик понимает только плоские имена. Поэтому `rustc` кодирует полный путь в одно уникальное имя: `_R` обозначает схему искажения Rust (она называется v0), `Cs…` — хеш крейта, а числа — длины (`3std`, `2rt`, `19lang_start_internal`). macOS добавляет ещё один `_` перед каждым именем уровня C. Часть с хешем отличается между сборками и компьютерами.

Когда у функции должно быть простое предсказуемое имя, например потому что её вызывает код на C, отключите искажение. Создайте `obj/names.rs`:

```rust
#[unsafe(no_mangle)]
pub extern "C" fn course_add(a: u64, b: u64) -> u64 {
    a + b
}

#[inline(never)]
pub fn rust_add(a: u64, b: u64) -> u64 {
    a + b
}
```

```bash
rustc --edition 2024 --crate-type lib --emit obj -C opt-level=3 -o obj/names.o obj/names.rs
nm obj/names.o | grep -v ltmp
```

```text
0000000000000000 T __RNvCskb9wMNpQEXg_5names8rust_add
0000000000000008 T _course_add
```

**Что происходит:**

1. `#[unsafe(no_mangle)]` сохраняет имя ровно таким, как оно написано. Атрибут помечен `unsafe` (редакция 2024), потому что два элемента с одинаковым простым именем где угодно в итоговой программе конфликтовали бы, а компилятор не может проверить это за вас.
2. `extern "C"` заставляет функцию использовать соглашение о вызовах C, чтобы код на C мог её вызвать (Урок 06.10).
3. `rust_add` получает искажённое имя. Без `#[inline(never)]` её бы вообще не было: маленькие публичные функции часто компилируются только в том крейте, который их вызывает.

## 3. Компоновщик: статическая и динамическая компоновка

Соберите программу обычным образом и посмотрите на результат:

```bash
cargo build --release
./target/release/p12-linking
file target/release/p12-linking
otool -L target/release/p12-linking
```

```text
hello from a static: 42
main is at        0x100e2c914
add_numbers is at 0x100e2c90c
GREETING is at    0x100e70040
a local is at     0x16efd22e4
target/release/p12-linking: Mach-O 64-bit executable arm64
target/release/p12-linking:
	/usr/lib/libSystem.B.dylib (compatibility version 1.0.0, current version 1359.40.3)
```

**Что происходит:**

1. Компоновщик объединил наши объектные файлы с теми частями `std`, `core` и `alloc`, которые использует программа. Эти библиотеки Rust скомпонованы **статически**: их код теперь внутри исполняемого файла (здесь 436 384 байта).
2. `otool -L` выводит список разделяемых библиотек, которые нужны исполняемому файлу во время работы. Она одна: `libSystem`, библиотека C в macOS. Она скомпонована **динамически**: исполняемый файл содержит только её имя и список нужных из неё символов.
3. Этот список можно увидеть через `nm -u` (неопределённые символы). Среди его 70 строк есть `_malloc`, `_free`, `_mmap` и `_write`: стандартная библиотека получает память в куче и пишет в терминал через библиотеку C.

| | Статическая компоновка | Динамическая компоновка |
|---|---|---|
| Где код библиотеки | Скопирован в исполняемый файл | В отдельном файле `.dylib` / `.so` |
| Размер файла | Больше | Меньше |
| Нужна ли библиотека во время работы | Нет | Да, должна быть установлена нужная версия |
| Обновления библиотеки (исправления безопасности) | Пересобрать программу | Обновить библиотеку один раз — выигрывают все программы |
| Что Rust компонует так | Крейты Rust (`std`, ваши зависимости) | Системную библиотеку C |

> ⚠️ **Частая ошибка:** отсутствующий символ обнаруживается во время **компоновки**, уже после успешной компиляции. Если вы объявите функцию C, которой нет ни в одной библиотеке, ошибку выдаст компоновщик, а не `rustc`. Раздел «Частые ошибки» показывает настоящее сообщение.

## 4. Внутри исполняемого файла Mach-O (macOS)

Исполняемый файл — это не только машинный код. Он начинается с **заголовка** и **команд загрузки** (load commands), которые говорят операционной системе, как его загружать. `otool -h` печатает заголовок, `otool -l` — команды загрузки, а `size -m` — сегменты:

```bash
otool -h target/release/p12-linking
otool -l target/release/p12-linking | grep -A2 LC_LOAD_DYLINKER
size -m target/release/p12-linking | grep -E '^Segment'
```

```text
target/release/p12-linking:
Mach header
      magic  cputype cpusubtype  caps    filetype ncmds sizeofcmds      flags
 0xfeedfacf 16777228          0  0x00           2    17       2104 0x00a00085
          cmd LC_LOAD_DYLINKER
      cmdsize 32
         name /usr/lib/dyld (offset 12)
Segment __PAGEZERO: 4294967296 (zero fill) 
Segment __TEXT: 278528
Segment __DATA_CONST: 16384
Segment __DATA: 16384
Segment __LINKEDIT: 131072
```

**Что происходит:**

1. `magic 0xfeedfacf` отмечает 64-битный файл Mach-O. `cputype 16777228` — номер arm64, а `filetype 2` означает «исполняемый файл» (у объектного файла тип 1).
2. `LC_LOAD_DYLINKER` называет **загрузчик**, который ядро должно запустить для этой программы: `/usr/lib/dyld`.
3. `__TEXT` содержит машинный код и данные только для чтения, например байты текста "hello from a static". `__DATA_CONST` содержит данные, которые загрузчик заполняет один раз и затем делает доступными только для чтения: здесь живёт сам `GREETING`, потому что это указатель плюс длина, и указатель нужно подправить под адрес, по которому загружена программа. `__DATA` содержит данные, которые программа может менять, а `__LINKEDIT` — таблицу символов и информацию для загрузчика.
4. `__PAGEZERO` — это 4 GiB (гибибайта) адресного пространства, отображённого вообще без доступа. Любое обращение по маленькому адресу, например по нулевому указателю, сразу вызывает падение, а не чтение мусора.

## 5. Внутри исполняемого файла ELF (Linux)

Linux использует ELF. Соберите и исследуйте ту же программу в Linux-контейнере (в официальном образе `rust:1.100`, где установлены инструменты для двоичных файлов). `CARGO_TARGET_DIR` держит сборку для Linux отдельно от сборки для macOS:

```bash
docker run --rm -it -v "$PWD":/work -w /work -e CARGO_TARGET_DIR=/work/target-linux rust:1.100 bash
```

Внутри контейнера:

```bash
cargo build -q --release
B=target-linux/release/p12-linking
file $B
readelf -d $B | grep NEEDED
readelf -l $B | grep interpreter
ldd $B
```

```text
target-linux/release/p12-linking: ELF 64-bit LSB pie executable, ARM aarch64, version 1 (SYSV), dynamically linked, interpreter /lib/ld-linux-aarch64.so.1, BuildID[sha1]=67070b5c417c5727d84dc8a67139276832f71a2a, for GNU/Linux 3.7.0, not stripped
 0x0000000000000001 (NEEDED)             Shared library: [libgcc_s.so.1]
 0x0000000000000001 (NEEDED)             Shared library: [libc.so.6]
      [Requesting program interpreter: /lib/ld-linux-aarch64.so.1]
	linux-vdso.so.1 (0x0000ffff8808b000)
	libgcc_s.so.1 => /lib/aarch64-linux-gnu/libgcc_s.so.1 (0x0000ffff87f90000)
	libc.so.6 => /lib/aarch64-linux-gnu/libc.so.6 (0x0000ffff87dd0000)
	/lib/ld-linux-aarch64.so.1 (0x0000ffff88040000)
```

**Что происходит:**

1. `file`: 64-битный исполняемый файл ELF типа **PIE** для AArch64 (так Linux называет arm64), скомпонованный динамически. LSB (Least Significant Byte first, сначала младший байт) означает little-endian (Урок 12.1).
2. `readelf -d` показывает динамическую секцию. Две записи `NEEDED`: библиотека C `libc.so.6` и `libgcc_s.so.1` — маленькая вспомогательная библиотека, которую стандартная библиотека Rust использует для раскрутки стека после паники.
3. **Интерпретатор** — это загрузчик `/lib/ld-linux-aarch64.so.1`. В x86-64 Linux это `/lib64/ld-linux-x86-64.so.2`.
4. `ldd` просит загрузчик всё разрешить и печатает, где найдена каждая библиотека. `linux-vdso.so.1` — крошечная библиотека, которую ядро само кладёт в каждый процесс, чтобы ускорить некоторые системные вызовы (Этап 13); файла у неё нет.

`readelf -h` печатает заголовок ELF, а `objdump -d` дизассемблирует код (Урок 12.7). По-прежнему внутри контейнера:

```bash
readelf -h $B | grep -E 'Class|Type|Machine|Entry'
nm -C $B | grep -E 'add_numbers|p12_linking::main'
objdump -d -C --no-show-raw-insn $B | grep -A3 '<p12_linking::add_numbers>:'
```

```text
  Class:                             ELF64
  Type:                              DYN (Position-Independent Executable file)
  Machine:                           AArch64
  Entry point address:               0x8700
0000000000008874 t p12_linking::add_numbers
000000000000887c T p12_linking::main
0000000000008874 <p12_linking::add_numbers>:
    8874:	add	x0, x1, x0
    8878:	ret
```

Вся функция — две инструкции arm64: сложить два аргумента в `x0` (регистр возвращаемого значения) и вернуться. **Точка входа** (0x8700) — это не `main`: это `_start` из среды выполнения C, которая подготавливает процесс и затем вызывает `main`.

**Статический двоичный файл для Linux.** В Linux можно попросить скомпоновать статически и библиотеку C — параметром генерации кода `rustc`. По-прежнему внутри контейнера:

```bash
RUSTFLAGS='-C target-feature=+crt-static' CARGO_TARGET_DIR=target-static cargo build -q --release
file target-static/release/p12-linking
ldd target-static/release/p12-linking
ls -l target-linux/release/p12-linking target-static/release/p12-linking | awk '{print $5, $9}'
exit
```

```text
warning: linker stderr: /usr/bin/ld: … in function `std::sys::net::connection::socket::lookup_host::{closure#0}':
         … Using 'getaddrinfo' in statically linked applications requires at runtime the shared libraries from the glibc version used for linking
target-static/release/p12-linking: ELF 64-bit LSB pie executable, ARM aarch64, version 1 (GNU/Linux), static-pie linked, BuildID[sha1]=f3fc44f638e2af7c69362c71f1d522f5f4d76722, for GNU/Linux 3.7.0, not stripped
	statically linked
469448 target-linux/release/p12-linking
1258976 target-static/release/p12-linking
```

**Что происходит:**

1. `crt-static` (C runtime, static — статическая среда выполнения C) встраивает в программу glibc — библиотеку C проекта GNU (большого проекта свободного ПО; GNU расшифровывается как «GNU's Not Unix»). Двоичному файлу больше не нужны разделяемые библиотеки (`ldd`: «statically linked»), и он почти втрое больше.
2. Компоновщик предупреждает (здесь сокращено): некоторые функции glibc, например поиск имён хостов, даже в статическом двоичном файле загружают разделяемые библиотеки во время работы. Это известное ограничение glibc. Для действительно самодостаточных двоичных файлов Linux проекты на Rust обычно собирают для целевых платформ `…-unknown-linux-musl`, которые используют маленькую библиотеку C musl вместо glibc.

> 💡 **Совет:** проводите такие эксперименты с Linux в контейнере или в вашей ВМ (VM, virtual machine, виртуальная машина) `dev` из Этапа 11. Инструменты macOS (`otool`) не читают ELF, а инструменты Linux (`readelf`, `ldd`) в macOS не установлены.

## 6. Загрузка: что происходит при старте программы

Когда вы набираете `./target/release/p12-linking`, ядро и загрузчик делают следующее:

1. Ядро открывает файл, читает заголовок и видит, какой загрузчик просит программа (`dyld` или `ld.so`).
2. Оно отображает сегменты программы в новое адресное пространство: код — только для чтения и исполнения, данные — для чтения и записи. Оно выбирает **случайный базовый адрес** (ASLR).
3. Оно отображает и загрузчик и запускает его.
4. Загрузчик читает список нужных разделяемых библиотек, находит их и тоже отображает по случайным адресам.
5. Он обрабатывает **перемещения**: каждое место, которому нужен адрес `malloc` или `write`, получает настоящий адрес в этом запуске.
6. Он переходит к точке входа. Стартует среда выполнения C, затем `lang_start` из Rust, затем ваш `main`.

Запустите программу на Mac дважды:

```text
main is at        0x100e2c914
add_numbers is at 0x100e2c90c
GREETING is at    0x100e70040
a local is at     0x16efd22e4
```

```text
main is at        0x100df0914
add_numbers is at 0x100df090c
GREETING is at    0x100e34040
a local is at     0x16f00e2e4
```

Все адреса между запусками меняются, но **расстояния остаются прежними**: `main` всегда на 8 байт после `add_numbers`, а последние цифры `…914`, `…90c`, `…040` никогда не меняются. Вся программа сдвигается одним блоком. Стек (`a local`) размещается по отдельному случайному адресу. Это работает только потому, что код позиционно-независимый: он находит свои данные относительно собственного положения, а не по фиксированным адресам.

> 🔒 **Безопасность:** ASLR защищает от атак, которым нужно знать, где в памяти лежат код или данные, например от перехода в существующий код после переполнения буфера. Он работает только для исполняемых файлов PIE; никогда не отключайте PIE у программы, которая обрабатывает недоверенный ввод. В Linux `file` показывает `pie executable`, когда он включён.

## Частые ошибки

**1. Вызов функции, которой нет ни в одной библиотеке.** Это компилируется, но не компонуется:

```rust
unsafe extern "C" {
    fn answer_from_c() -> i32;
}

fn main() {
    // SAFETY: we promise that a C function with this signature exists.
    let answer = unsafe { answer_from_c() };
    println!("the answer is {answer}");
}
```

```text
error: linking with `cc` failed: exit status: 1
  |
  = note:  "cc" … (a very long command line, shortened here)
  = note: some arguments are omitted. use `--verbose` to show all linker arguments
  = note: Undefined symbols for architecture arm64:
            "_answer_from_c", referenced from:
                __RNvCsbDkgxTN66NU_11p12_linking4main in p12_linking.3naovqe6q21388ozee9hq60ud.1tu0v35.rcgu.o
          ld: symbol(s) not found for architecture arm64
          clang: error: linker command failed with exit code 1 (use -v to see invocation)
```

Читайте снизу вверх: компоновщик (`ld`) не нашёл символ `_answer_from_c`, который использует `p12_linking::main` (искажённое имя). Исправление — предоставить функцию: скомпилировать код на C с помощью крейта `cc` (Урок 06.10) или скомпоновать библиотеку, в которой она есть. В Linux та же ошибка звучит как ``undefined reference to `answer_from_c'`` (GNU `ld`) или `undefined symbol: answer_from_c` (`rust-lld`).

**2. Разделяемая библиотека, которую не находит загрузчик.** Программа может успешно скомпоноваться и всё равно упасть при старте: `error while loading shared libraries: libgreet.so: cannot open shared object file`. Компоновщик нашёл библиотеку в папке сборки, но загрузчик ищет только в системных папках и в путях, которые вы ему укажете. Упражнение 4 показывает это и исправление.

**3. Думать, что `main` — первый выполняемый код.** До `main` работают загрузчик, среда выполнения C и `lang_start` из Rust. Они готовят стек, аргументы, стандартные потоки и обработку паники.

## Попробуйте сами

1. Соберите `p12-linking` в режиме release, запустите три раза и проверьте, что расстояние между `main` и `add_numbers` не меняется.
2. Запустите `nm -u target/release/p12-linking | wc -l`, а затем `nm -u target/release/p12-linking | grep write`. Какие функции C нужны даже программе «hello»?
3. Создайте `obj/main.o`, как в разделе 2, и сравните `nm obj/main.o` с `nm --demangle obj/main.o`.
4. Выполните раздел 5 в контейнере и сравните размеры динамического и статического двоичного файла.
5. Уберите за собой: `rm -rf obj target-linux target-static` и `cargo clean`.

## Упражнения

### Упражнение 1 — Прочитайте таблицу символов (легко)

Вот три строки из `nm --demangle` объектного файла. Для каждой скажите, определён ли символ в этом файле, могут ли его использовать другие файлы и что это, скорее всего, такое.

```text
0000000000000084 t main::add_numbers
                 U std::io::stdio::_print
0000000000000228 s main::GREETING
```

<details>
<summary>Подсказка</summary>

Заглавная буква — виден другим файлам; строчная — локальный. У `U` нет адреса, потому что кода нет в этом файле.

</details>

<details>
<summary>Ответ</summary>

1. `t main::add_numbers`: код, определённый в этом файле по смещению 0x84, локальный (строчная `t`): приватная функция.
2. `U std::io::stdio::_print`: здесь не определён; файл его использует, и компоновщик должен его найти (в rlib `std`).
3. `s main::GREETING`: данные, определённые в этом файле, локальные: наш `static`.

</details>

### Упражнение 2 — Функция для вызова из C с тестами (средне)

Добавьте в `p12-linking` файл `src/lib.rs` с функцией `course_checksum(data: *const u8, len: usize) -> u32`, которую может вызывать код на C: она складывает все байты с переносом при переполнении и возвращает 0 при `len == 0`, не читая `data`. Напишите два теста (`[1, 2, 3, 250]` даёт 256; нулевой указатель с длиной 0 даёт 0), запустите их и под Miri и проверьте через `nm`, что имя символа не искажено.

<details>
<summary>Подсказка</summary>

Функция должна быть `pub unsafe extern "C" fn` с `#[unsafe(no_mangle)]`: вызывающая сторона должна обещать, что указатель корректен, поэтому вызов самой функции — `unsafe`. Внутри `std::slice::from_raw_parts(data, len)` превращает указатель в срез. После `cargo build --release --lib` запустите `nm target/release/libp12_linking.rlib`.

</details>

<details>
<summary>Ответ</summary>

```rust
/// Adds up all bytes of a buffer, wrapping on overflow. Callable from C.
///
/// # Safety
///
/// `data` must point to `len` readable bytes (or `len` must be 0).
#[unsafe(no_mangle)]
pub unsafe extern "C" fn course_checksum(data: *const u8, len: usize) -> u32 {
    if len == 0 {
        return 0;
    }
    // SAFETY: the caller promises that `data` points to `len` readable bytes.
    let bytes = unsafe { std::slice::from_raw_parts(data, len) };
    bytes.iter().fold(0u32, |sum, &b| sum.wrapping_add(u32::from(b)))
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn sums_the_bytes() {
        let data = [1u8, 2, 3, 250];
        // SAFETY: `data` is a live array of exactly `data.len()` bytes.
        let sum = unsafe { course_checksum(data.as_ptr(), data.len()) };
        assert_eq!(sum, 256);
    }

    #[test]
    fn empty_input_gives_zero() {
        // SAFETY: with len 0 the pointer is never read.
        let sum = unsafe { course_checksum(std::ptr::null(), 0) };
        assert_eq!(sum, 0);
    }
}
```

```bash
cargo test -q --lib
cargo +nightly miri test -q --lib
cargo build -q --release --lib
nm target/release/libp12_linking.rlib 2>/dev/null | grep course
```

```text
..
test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
..
test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.24s
0000000000000000 T _course_checksum
```

Почему это работает: `from_raw_parts` с нулевым указателем — это неопределённое поведение (UB) даже для длины 0, поэтому ранний `return` важен, и Miri поймал бы его отсутствие. Раздел `# Safety` документирует обещание, которое должна сдержать вызывающая сторона (Урок 06.6). `.rlib` — архив объектных файлов, поэтому `nm` может его прочитать; `2>/dev/null` прячет его замечания о членах архива с метаданными, в которых нет символов.

</details>

### Упражнение 3 — Прямые и косвенные зависимости (средне)

Сравните `otool -L /bin/ls` на вашем Mac с `ldd /bin/ls` и `readelf -d /bin/ls | grep NEEDED` в контейнере `rust:1.100`. Почему `ldd` показывает библиотеку, которой нет в выводе `readelf -d`?

<details>
<summary>Подсказка</summary>

`readelf` читает только тот файл, который вы ему дали. `ldd` просит загрузчик загрузить всё, включая библиотеки, нужные библиотекам.

</details>

<details>
<summary>Ответ</summary>

```text
/bin/ls:
	/usr/lib/libutil.dylib (compatibility version 1.0.0, current version 1.0.0)
	/usr/lib/libncurses.5.4.dylib (compatibility version 5.4.0, current version 5.4.0)
	/usr/lib/libSystem.B.dylib (compatibility version 1.0.0, current version 1359.40.3)
```

В контейнере:

```text
	linux-vdso.so.1 (0x0000ffff8522c000)
	libselinux.so.1 => /lib/aarch64-linux-gnu/libselinux.so.1 (0x0000ffff85140000)
	libcap.so.2 => /lib/aarch64-linux-gnu/libcap.so.2 (0x0000ffff85110000)
	libc.so.6 => /lib/aarch64-linux-gnu/libc.so.6 (0x0000ffff84f50000)
	/lib/ld-linux-aarch64.so.1 (0x0000ffff851f0000)
	libpcre2-8.so.0 => /lib/aarch64-linux-gnu/libpcre2-8.so.0 (0x0000ffff84e90000)
 0x0000000000000001 (NEEDED)             Shared library: [libselinux.so.1]
 0x0000000000000001 (NEEDED)             Shared library: [libcap.so.2]
 0x0000000000000001 (NEEDED)             Shared library: [libc.so.6]
 0x0000000000000001 (NEEDED)             Shared library: [ld-linux-aarch64.so.1]
```

Почему это работает: самой `ls` нужны `libselinux`, `libcap` и `libc` (её записи `NEEDED`). `libselinux`, в свою очередь, нужна `libpcre2-8`, поэтому загрузчик загружает и её, а `ldd` показывает всё дерево. `otool -L` в macOS, как и `readelf -d`, показывает только прямые зависимости.

</details>

### Упражнение 4 — Ваша собственная разделяемая библиотека (сложно)

В контейнере соберите **разделяемую библиотеку** на Rust с функцией `course_add(a: u64, b: u64) -> u64` и маленькую программу на C, которая её вызывает. Скомпонуйте их с помощью `gcc`, затем запустите программу: сначала как есть, потом с `LD_LIBRARY_PATH`, указывающим на папку библиотеки. Объясните оба результата.

Используйте эту программу на C (`main.c`, рядом с `Cargo.toml`):

```c
#include <stdint.h>
#include <stdio.h>

uint64_t course_add(uint64_t a, uint64_t b);

int main(void) {
    printf("2 + 40 = %llu\n", (unsigned long long)course_add(2, 40));
    return 0;
}
```

<details>
<summary>Подсказка</summary>

`cargo new --lib greet`, затем добавьте в `Cargo.toml` раздел `[lib]` с `crate-type = ["cdylib"]`: динамическая библиотека, совместимая с C, `libgreet.so`. Компонуйте командой `gcc main.c -o main -L <folder> -lgreet`: `-L` говорит, где искать во время компоновки, `-lgreet` означает `libgreet.so`.

</details>

<details>
<summary>Ответ</summary>

`Cargo.toml` крейта `greet` заканчивается так:

```toml
[lib]
crate-type = ["cdylib"]
```

`src/lib.rs`:

```rust
/// Returns a + b. Callable from C as `uint64_t course_add(uint64_t, uint64_t)`.
#[unsafe(no_mangle)]
pub extern "C" fn course_add(a: u64, b: u64) -> u64 {
    a + b
}
```

Внутри контейнера `rust:1.100`, запущенного в папке `greet` (как в разделе 5):

```bash
cargo build -q --release
nm -D --defined-only target-linux/release/libgreet.so | grep course
gcc main.c -o main -L target-linux/release -lgreet
./main; echo "exit $?"
LD_LIBRARY_PATH=target-linux/release ldd ./main
LD_LIBRARY_PATH=target-linux/release ./main
```

```text
0000000000000648 T course_add
./main: error while loading shared libraries: libgreet.so: cannot open shared object file: No such file or directory
exit 127
	linux-vdso.so.1 (0x0000ffffb2d57000)
	libgreet.so => target-linux/release/libgreet.so (0x0000ffffb2cb0000)
	libc.so.6 => /lib/aarch64-linux-gnu/libc.so.6 (0x0000ffffb2af0000)
	libgcc_s.so.1 => /lib/aarch64-linux-gnu/libgcc_s.so.1 (0x0000ffffb2ab0000)
	/lib/ld-linux-aarch64.so.1 (0x0000ffffb2d10000)
2 + 40 = 42
```

Почему это работает: `nm -D` показывает **динамическую** таблицу символов — имена, которые разделяемая библиотека предлагает другим; `course_add` там с простым именем. Во время компоновки `-L` сказал компоновщику, где лежит `libgreet.so`, поэтому компоновка прошла, но исполняемый файл хранит только имя `libgreet.so`. При старте загрузчик ищет в системных папках, не находит библиотеку и останавливается с кодом возврата 127. `LD_LIBRARY_PATH` добавляет папку к поиску загрузчика. Настоящие программы устанавливают библиотеку в системную папку или записывают путь поиска в исполняемый файл.

</details>

## Вопросы для самопроверки

1. Что находится внутри объектного файла и почему его нельзя запустить?

<details>
<summary>Ответ</summary>

Машинный код и данные одной части программы плюс таблица символов с именами, которые она определяет, и именами, которые ей ещё нужны (`U`), а также перемещения для каждого места, которому нужен итоговый адрес. Запустить его нельзя, потому что недостающие части и итоговые адреса заполняет только компоновщик.

</details>

2. Зачем `rustc` искажает имена и как отключить это для одной функции?

<details>
<summary>Ответ</summary>

Компоновщик знает только плоские имена, а в Rust есть модули, крейты с одинаковыми именами функций и обобщённые функции, которые существуют по одной на каждый тип. Искажение кодирует полный путь в одно уникальное имя. `#[unsafe(no_mangle)]` сохраняет простое имя функции, обычно вместе с `extern "C"`.

</details>

3. Какие библиотеки обычная программа на Rust в macOS компонует статически, а какие — динамически?

<details>
<summary>Ответ</summary>

Статически: все крейты Rust, включая `std`, `core`, `alloc` и ваши зависимости. Динамически: системную библиотеку C `libSystem` (в Linux — `libc.so.6` и `libgcc_s.so.1`).

</details>

4. Сборка падает с «Undefined symbols … referenced from …». Это ошибка компилятора или компоновщика и что она значит?

<details>
<summary>Ответ</summary>

Ошибка компоновщика. Код скомпилировался, но какой-то используемый им символ не определён ни в одном объектном файле или библиотеке, переданных компоновщику. Предоставьте код (например, скомпилируйте файл на C) или скомпонуйте нужную библиотеку.

</details>

5. Что делают `dyld` и `ld.so`?

<details>
<summary>Ответ</summary>

Это загрузчики (динамические компоновщики) macOS и Linux. Они работают первыми при старте динамически скомпонованной программы: отображают нужные разделяемые библиотеки в память, проставляют адреса их символов (перемещения) и затем переходят к точке входа программы.

</details>

6. Почему у `main` при каждом запуске другой адрес и почему программа всё равно работает?

<details>
<summary>Ответ</summary>

ASLR каждый раз загружает программу по случайному базовому адресу. Программа позиционно-независимая (PIE): её код обращается к собственным функциям и данным относительно своего положения, поэтому работает по любому адресу, а расстояния между её частями остаются прежними.

</details>

## Итоги

- Исходный код становится процессом за четыре шага: `rustc` компилирует в **объектные файлы**, **компоновщик** объединяет их с библиотеками в исполняемый файл, а **загрузчик** отображает его в память и подключает разделяемые библиотеки.
- **Символы** — это имена функций и статических переменных; `nm` выводит их (`T` — определён, `U` — нужен). Rust **искажает** имена; `nm --demangle` делает их читаемыми, а `#[unsafe(no_mangle)]` вместе с `extern "C"` даёт простое имя для C.
- Крейты Rust компонуются **статически**, системная библиотека C — **динамически**. `otool -L` (macOS) и `readelf -d` / `ldd` (Linux) показывают разделяемые библиотеки.
- macOS использует **Mach-O** (заголовок, команды загрузки, сегменты вроде `__TEXT`, загрузчик `/usr/lib/dyld`); Linux использует **ELF** (`readelf -h`, `NEEDED`, интерпретатор `ld-linux…so`).
- `-C target-feature=+crt-static` даёт в Linux статический двоичный файл с glibc, с ограничениями; обычный путь к полностью статическим двоичным файлам — целевые платформы musl.
- Отсутствующие символы — ошибки **компоновки**; отсутствующие разделяемые библиотеки — ошибки **загрузки** при старте.
- **ASLR** и **PIE** сдвигают всю программу по случайному адресу при каждом запуске; расстояния внутри неё остаются прежними.

## Источники и что почитать

- [The Rust Reference — Linkage](https://doc.rust-lang.org/reference/linkage.html) — типы крейтов (`bin`, `rlib`, `cdylib`, …), статическая и динамическая среда выполнения C.
- [The rustc book — Symbol mangling](https://doc.rust-lang.org/rustc/symbol-mangling/index.html) — как Rust строит имена символов, включая схему v0.
- [The Rust Reference — Application binary interface](https://doc.rust-lang.org/reference/abi.html) — `no_mangle`, `extern` и связанные атрибуты.
- [The rustc book — Codegen options](https://doc.rust-lang.org/rustc/codegen-options/index.html) — `target-feature`, `link-arg` и другие параметры, связанные с компоновщиком.
- [The Cargo Book — Cargo targets](https://doc.rust-lang.org/cargo/reference/cargo-targets.html) — где настраивается `crate-type`.
- [elf(5)](https://man7.org/linux/man-pages/man5/elf.5.html), [ld.so(8)](https://man7.org/linux/man-pages/man8/ld.so.8.html) и [ldd(1)](https://man7.org/linux/man-pages/man1/ldd.1.html) — формат ELF и загрузчик Linux в страницах руководства Linux.
- [David Drysdale — Beginner's Guide to Linkers](https://www.lurklurk.org/linkers/linkers.html) — мягкое и более длинное введение в символы, компоновщики и разделяемые библиотеки.
- [Computer Systems: A Programmer's Perspective (CS:APP)](https://csapp.cs.cmu.edu/) — глава 7 — классическое объяснение компоновки.

---

[🏠 Начало](../START-HERE.md) · [← 12.11 Размещение типов Rust в памяти](11-memory-layout-of-rust-types.md) · [📋 Этап 12](README.md) · [12.13 Как работают аллокаторы памяти →](13-how-memory-allocators-work.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
