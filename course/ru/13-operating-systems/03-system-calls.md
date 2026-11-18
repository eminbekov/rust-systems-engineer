<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 13.2 Процессы изнутри](02-processes-from-the-inside.md) · [📋 Этап 13](README.md) · [13.4 Файлы и файловые дескрипторы →](04-files-and-file-descriptors.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/13-operating-systems/03-system-calls.md) <!-- nav -->

# Урок 13.3 — Системные вызовы

> **Этап 13 — Операционные системы изнутри** · Урок 3 из 18 · ⏱ около 120 минут

## Что вы узнаете

- Что именно происходит, когда программа делает системный вызов, вплоть до инструкции процессора.
- Как один системный вызов выглядит на четырёх уровнях: ассемблер, библиотека C, крейт `nix` и стандартная библиотека Rust.
- Как ядро сообщает об ошибках (`errno`) и как `libc`, `nix` и `std` показывают их вам.
- Как увидеть каждый системный вызов программы с помощью `strace`.
- Почему системные вызовы стоят времени и как буферизация экономит большую их часть.

## Перед началом

- Вы прошли [Урок 13.2 — Процессы изнутри](02-processes-from-the-inside.md).
- Вы немного читаете ассемблер arm64: регистры `x0`…`x30` и понятие инструкции ([Урок 12.7 — Читаем ассемблер](../12-computer-systems/07-reading-assembly.md)).
- Вы знаете `unsafe` и сырые указатели ([Урок 06.6](../06-rust-advanced/06-unsafe-rust-and-miri.md)) и умеете вызывать функции C через FFI ([Урок 06.10 — FFI: вызов C](../06-rust-advanced/10-ffi-calling-c.md)).
- В вашей ВМ установлен `strace` (Урок 13.1, раздел 5).

## Новые слова

| Слово | Значение |
|---|---|
| **номер системного вызова** (system call number) | Число, которое говорит ядру, какая услуга нужна: в Linux на arm64 64 — это `write`, а 172 — `getpid`. |
| **`svc`** (SuperVisor Call, вызов супервизора) | Инструкция arm64, которая входит в ядро. На x86-64 инструкция называется `syscall`. |
| **ABI** (Application Binary Interface, двоичный интерфейс приложений) | Правила на уровне машины: в каком регистре какой аргумент, куда кладётся результат. |
| **libc** (C library, библиотека C) | Библиотека маленьких функций на C (`write`, `open`, `fork` …), через которые каждая программа в Linux делает системные вызовы. Ubuntu и Debian используют glibc (библиотеку C проекта GNU). |
| **обёртка** (wrapper) | Небольшая функция вокруг другой, которая делает её удобнее или безопаснее. |
| **`errno`** (error number, номер ошибки) | Переменная (своя у каждого потока), куда libc записывает причину неудачи последнего вызова, например 2 = `ENOENT`. |
| **`strace`** (system call trace, трассировка системных вызовов) | Инструмент, который запускает программу и печатает каждый её системный вызов с аргументами и результатом. |
| **буфер** / **буферизация** (buffer / buffering) | Кусок памяти, где собираются данные / сбор многих маленьких записей в одну большую. |
| **vDSO** (virtual Dynamic Shared Object, виртуальный динамический разделяемый объект) | Крошечная библиотека, которую ядро кладёт в каждый процесс, чтобы некоторым вызовам (например, чтению часов) не нужно было входить в ядро. |
| **CPU**, **ОС**, **PID**, **ELF**, **КиБ**, **GNU** | Известны из прошлых уроков: Central Processing Unit, центральный процессор; операционная система; Process IDentifier, идентификатор процесса; Executable and Linkable Format (Урок 12.12); кибибайт (1024 байта); GNU («GNU's Not Unix», «GNU — не Unix»), проект свободного ПО, который делает glibc. |

## 1. Вызов функции, пересекающий границу

Ваша программа не может просто вызвать функцию внутри ядра: память ядра защищена (Урок 13.1), и переход туда вызвал бы тот
же `SIGSEGV`, что и чтение. Вместо этого у процессора есть **одна особая инструкция**, которая переключает уровень
привилегий на уровень ядра и переходит в единственную точку входа, выбранную ядром при загрузке. На arm64 это `svc #0`,
на x86-64 — `syscall`.

Раз дверь одна, программа должна сказать, какая услуга ей нужна. Перед инструкцией она кладёт в регистры **номер
системного вызова** и аргументы. Эти правила — часть **ABI** Linux:

| | arm64 (ваша ВМ) | x86-64 |
|---|---|---|
| инструкция | `svc #0` | `syscall` |
| номер системного вызова | `x8` | `rax` |
| аргументы 1–6 | `x0` … `x5` | `rdi`, `rsi`, `rdx`, `r10`, `r8`, `r9` |
| результат | `x0` | `rax` |
| `write` / `getpid` | 64 / 172 | 1 / 39 |

Ядро находит номер в своей **таблице системных вызовов**, выполняет соответствующую функцию ядра (например,
`ksys_write`), кладёт результат в `x0` и возвращается к инструкции после `svc`. Если что-то пошло не так, результат —
небольшое **отрицательное** число: −2 означает «ошибка 2», то есть `ENOENT`.

Обратите внимание: номера отличаются у разных типов процессоров. Это одна из причин, почему программы почти никогда не
пишут `svc` сами.

## 2. Один системный вызов на четырёх уровнях

Выведем одну строку текста на экран (файловый дескриптор 1, стандартный вывод, Урок 11.3) четырьмя способами, от самого
низкого уровня к самому высокому. Сначала добавьте крейт `libc` (объявления Rust для библиотеки C, Урок 06.10) и фичу
`fs` крейта `nix`, которая понадобится в разделе 3:

```bash
cargo add libc@0.2
cargo add nix@0.32 --features fs
```

Создайте `src/bin/four-writes.rs`:

```rust
use std::io::Write;

/// Level 0: the raw system call, written in assembly (arm64 only).
#[cfg(target_arch = "aarch64")]
fn raw_write(fd: usize, buf: &[u8]) -> isize {
    let ret: isize;
    // SAFETY: `write` only reads `buf.len()` bytes from `buf`, which is a valid slice.
    unsafe {
        std::arch::asm!(
            "svc #0",                  // "supervisor call": enter the kernel
            in("x8") 64_usize,         // system call number: 64 = write on arm64
            inlateout("x0") fd => ret, // argument 1 in, result out
            in("x1") buf.as_ptr(),     // argument 2: address of the bytes
            in("x2") buf.len(),        // argument 3: how many bytes
            options(nostack),
        );
    }
    ret
}

fn main() {
    // Level 0: assembly.
    #[cfg(target_arch = "aarch64")]
    raw_write(1, b"0: raw svc instruction\n");

    // Level 1: the C library function write(), through FFI.
    let text = b"1: libc::write\n";
    // SAFETY: the pointer and the length describe the valid array `text`.
    unsafe { libc::write(1, text.as_ptr().cast(), text.len()) };

    // Level 2: nix, a safe wrapper around the libc function.
    nix::unistd::write(std::io::stdout(), b"2: nix::unistd::write\n").unwrap();

    // Level 3: the standard library.
    std::io::stdout().write_all(b"3: std::io::Write\n").unwrap();
    println!("4: println!");
}
```

```text
0: raw svc instruction
1: libc::write
2: nix::unistd::write
3: std::io::Write
4: println!
```

**Что происходит:**

1. **Уровень 0.** `std::arch::asm!` вставляет ассемблер прямо в функцию Rust. `in("x8") 64_usize` загружает номер `write`
   в `x8`; `x0`, `x1`, `x2` получают файловый дескриптор, адрес и длину. `inlateout("x0") fd => ret` означает «`x0` —
   вход, а после инструкции прочитай из него результат». `svc #0` входит в ядро. Встроенный ассемблер всегда `unsafe`:
   компилятор не может его проверить. `#[cfg(target_arch = "aarch64")]` компилирует эту часть только на arm64.
2. **Уровень 1.** `libc::write` — функция библиотеки C. Внутри она делает то же, что уровень 0 (на любом типе процессора),
   а при неудаче записывает номер ошибки в `errno` и возвращает −1. Она принимает сырой указатель, поэтому вызов `unsafe`.
3. **Уровень 2.** `nix::unistd::write` вызывает `libc::write` за вас, принимает срез Rust и всё, у чего есть файловый
   дескриптор (у `std::io::stdout()` он есть), и возвращает `Result`. В вашем коде нет `unsafe`.
4. **Уровень 3.** `std::io::Write` — то, чем вы обычно пользуетесь. `println!` заканчивается тем же системным вызовом `write`.

Все пять строк прошли в ядро одной и той же дорогой. `strace` это доказывает (подробнее об этом инструменте — в разделе 4):

```bash
strace -e trace=write ./target/debug/four-writes > /dev/null
```

```text
write(1, "0: raw svc instruction\n", 23) = 23
write(1, "1: libc::write\n", 15)        = 15
write(1, "2: nix::unistd::write\n", 22) = 22
write(1, "3: std::io::Write\n", 18)     = 18
write(1, "4: println!\n", 12)           = 12
+++ exited with 0 +++
```

Ядро не может отличить, с какого уровня пришёл вызов. `= 23` — результат: число записанных байтов.

> 📏 **Соглашение:** используйте самый высокий уровень, который справляется с задачей. Сначала `std`; `nix` — для
> возможностей Unix, которых нет в `std`; `libc` — только если чего-то нет в `nix`; ассемблер — никогда в обычном коде.
> Здесь вы увидели его один раз, чтобы знать, что лежит на самом дне.

## 3. Когда системный вызов не удаётся: `errno`

Многие системные вызовы могут не удаться: файла нет, не хватает прав, диск заполнен. Ядро возвращает отрицательное число,
и каждый уровень превращает его в своё:

- **libc** возвращает −1 и кладёт положительный номер ошибки в `errno`. Читать `errno` нужно сразу, пока другой вызов его
  не изменил.
- **nix** возвращает `Err(Errno::ENOENT)` — перечисление (enum) с отдельным именем для каждой ошибки.
- **std** возвращает `Err(io::Error)`, который хранит номер (`raw_os_error()`) и переносимый вид ошибки `kind()`.

Создайте `src/bin/errors.rs`:

```rust
use nix::fcntl::{OFlag, open};
use nix::sys::stat::Mode;

fn main() {
    // libc: returns -1 and leaves the error number in `errno`.
    let fd = unsafe { libc::open(c"/no/such/file".as_ptr(), libc::O_RDONLY) };
    let err = std::io::Error::last_os_error();
    println!("libc: fd = {fd}, errno = {:?}, {err}", err.raw_os_error());

    // nix: returns Result<OwnedFd, Errno>.
    match open("/no/such/file", OFlag::O_RDONLY, Mode::empty()) {
        Ok(_) => println!("nix:  opened?!"),
        Err(e) => println!("nix:  {e:?} = {}", e.desc()),
    }

    // std: returns io::Result<File>; the OS error number is inside.
    match std::fs::File::open("/no/such/file") {
        Ok(_) => println!("std:  opened?!"),
        Err(e) => println!("std:  {e} (kind {:?})", e.kind()),
    }
}
```

```text
libc: fd = -1, errno = Some(2), No such file or directory (os error 2)
nix:  ENOENT = No such file or directory
std:  No such file or directory (os error 2) (kind NotFound)
```

**Что происходит:**

1. `c"/no/such/file"` — это **литерал C-строки**: `&CStr` с NUL-байтом в конце, созданный во время компиляции. Это то же,
   что делает `CString::new` из Урока 06.10, только без выделения памяти.
2. `libc::open` возвращает −1. `std::io::Error::last_os_error()` читает `errno` и оборачивает его: ошибка 2.
3. `nix::fcntl::open` возвращает `Err(Errno::ENOENT)`; `desc()` даёт текст. При успехе результатом был бы `OwnedFd` —
   файловый дескриптор, которым владеют (Урок 13.4).
4. `std::fs::File::open` показывает ту же ошибку 2 и вдобавок переносимый вид `NotFound`.

Частые имена ошибок, которые вы встретите на этом этапе: `ENOENT` (нет такого файла), `EACCES` (доступ запрещён), `EBADF`
(плохой файловый дескриптор), `EINTR` (прервано сигналом, Урок 13.7), `EAGAIN` (попробуйте позже), `EPIPE` (разорванный
конвейер, Урок 13.6). Полный список — на странице справки `errno(3)`.

> ⚠️ **Частая ошибка:** читать `errno` слишком поздно. Любой вызов между неудачным вызовом и `last_os_error()` — даже
> `println!` — может изменить `errno`, и вы сообщите не ту ошибку. В `nix` и `std` ошибка — часть возвращаемого значения,
> поэтому такая ошибка невозможна.

## 4. `strace`: смотрим, как программа разговаривает с ядром

`strace` запускает программу и печатает каждый системный вызов: имя, аргументы и после `=` результат. Он использует
возможность ядра под названием `ptrace` (process trace, трассировка процессов) — ту же, что и отладчики. Начнём с самой
маленькой программы, `src/bin/hello.rs`:

```rust
fn main() {
    println!("hello");
}
```

```bash
cargo build -q
strace ./target/debug/hello
```

Вывод занимает около 65 строк. Вот начало и конец (настоящий вывод, середина сокращена):

```text
execve("./target/debug/hello", ["./target/debug/hello"], 0xfffff220e130 /* 11 vars */) = 0
brk(NULL)                               = 0xaaaaea1eb000
openat(AT_FDCWD, "/etc/ld.so.cache", O_RDONLY|O_CLOEXEC) = 3
openat(AT_FDCWD, "/lib/aarch64-linux-gnu/libc.so.6", O_RDONLY|O_CLOEXEC) = 3
read(3, "\177ELF\2\1\1\3\0\0\0\0\0\0\0\0\3\0\267\0\1\0\0\0\200$\2\0\0\0\0\0"..., 832) = 832
mmap(0xffff975b0000, 1826832, PROT_READ|PROT_EXEC, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0) = 0xffff975b0000
close(3)                                = 0
...
rt_sigaction(SIGPIPE, {sa_handler=SIG_IGN, sa_mask=[PIPE], sa_flags=SA_RESTART}, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=0}, 8) = 0
...
write(1, "hello\n", 6hello
)                  = 6
exit_group(0)                           = ?
+++ exited with 0 +++
```

**Что происходит:**

1. `execve` — ядро заменяет программу процесса на `hello` (Урок 13.5).
2. `openat`, `read`, `mmap`, `close` — динамический загрузчик (Урок 12.12) открывает `libc.so.6`, читает его заголовок
   ELF и отображает его в память. Результат `= 3` — новый файловый дескриптор.
3. `rt_sigaction(SIGPIPE, … SIG_IGN …)` — рантайм Rust ещё до запуска `main` говорит ядру **игнорировать** `SIGPIPE`.
   Урок 13.6 покажет, почему это важно.
4. `write(1, "hello\n", 6)` — ваш `println!`. Собственный вывод программы `hello` появляется посреди строки, потому что
   `strace` пишет в стандартный поток ошибок, а программа — в стандартный вывод, на один и тот же терминал.
5. `exit_group(0)` — процесс завершается с кодом 0. У этого вызова нет результата, поэтому `strace` печатает `= ?`.

Ваша программа из одной строки делает около 60 системных вызовов, почти все — для загрузки и настройки. Полезные параметры:

| Параметр | Что делает |
|---|---|
| `-e trace=openat,read` | показывать только эти вызовы |
| `-c` | считать вызовы и время вместо печати каждого |
| `-f` | следить и за дочерними процессами (Урок 13.5) |
| `-p PID` | подключиться к уже работающему процессу |
| `-o file` | записать трассировку в файл |

```bash
strace -e trace=openat ./target/debug/os-info 2>&1 | grep proc
```

```text
openat(AT_FDCWD, "/proc/self/maps", O_RDONLY|O_CLOEXEC) = 3
openat(AT_FDCWD, "/proc/version", O_RDONLY|O_CLOEXEC) = 3
openat(AT_FDCWD, "/proc/uptime", O_RDONLY|O_CLOEXEC) = 3
openat(AT_FDCWD, "/proc/meminfo", O_RDONLY|O_CLOEXEC) = 3
```

Последние три строки — файлы вашего `os-info` из Урока 13.1. Первую читает рантайм Rust при старте (он узнаёт там границы
стека). Каждый файл снова получает дескриптор 3, потому что предыдущий был закрыт.

> 💡 **Совет:** `strace` — первый инструмент, за который стоит браться, когда программа «ничего не делает» или падает без
> внятного сообщения. Ищите последние вызовы перед остановкой и результаты вида `-1 E…`: они назовут файл, который не
> удалось открыть, или вызов, который не удался.

> 🔒 **Безопасность:** `strace` показывает всё, что программа отправляет и получает, включая пароли, которые она читает.
> Поэтому обычный пользователь может трассировать только свои процессы, и поэтому некоторым конфигурациям контейнеров нужен
> параметр `--cap-add SYS_PTRACE`, прежде чем `strace` заработает внутри них.

## 5. Системные вызовы стоят времени

Системный вызов намного медленнее обычного вызова функции: процессор должен сохранить состояние, сменить уровень привилегий,
проверить аргументы и переключиться обратно. Простой вызов стоит примерно от 100 наносекунд до нескольких микросекунд, то
есть сотни обычных вызовов функций. Программы, которые делают миллионы крошечных вызовов, теряют много времени на границе.

`stdout` в Rust собирает текст до конца строки, поэтому каждый `println!` — это один вызов `write`. Сравните с `BufWriter`,
который собирает 8 КиБ, прежде чем записать. Создайте `src/bin/many-lines.rs`:

```rust
use std::io::{BufWriter, Write};

fn main() -> std::io::Result<()> {
    let buffered = std::env::args().any(|a| a == "--buffered");
    if buffered {
        let mut out = BufWriter::new(std::io::stdout().lock());
        for i in 0..1000 {
            writeln!(out, "line {i}")?;
        }
        out.flush()?;
    } else {
        for i in 0..1000 {
            println!("line {i}");
        }
    }
    Ok(())
}
```

```bash
cargo build -q
strace -c -e trace=write ./target/debug/many-lines > /dev/null
strace -c -e trace=write ./target/debug/many-lines --buffered > /dev/null
```

```text
% time     seconds  usecs/call     calls    errors syscall
------ ----------- ----------- --------- --------- ----------------
100.00    0.003099           3      1000           write
------ ----------- ----------- --------- --------- ----------------
100.00    0.003099           3      1000           total
% time     seconds  usecs/call     calls    errors syscall
------ ----------- ----------- --------- --------- ----------------
100.00    0.000008           4         2           write
------ ----------- ----------- --------- --------- ----------------
100.00    0.000008           4         2           total
```

**Что происходит:**

1. Без буфера: 1000 строк, 1000 вызовов `write`, около 3 мс внутри ядра (`strace` сам замедляет каждый вызов, поэтому
   настоящие числа меньше, но соотношение сохраняется).
2. С `BufWriter`: те же 8890 байтов уходят за 2 вызова, потому что 8 КиБ текста поместились в первый.
3. `stdout().lock()` берёт блокировку стандартного вывода один раз, а не на каждую строку.
4. `out.flush()?` записывает остаток. `BufWriter` сбрасывает буфер и при уничтожении (drop), но тогда не может сообщить об
   ошибке, поэтому вызывайте `flush` сами.

> ⚡ **Производительность:** считайте системные вызовы, прежде чем оптимизировать: `strace -c` показывает, где программа
> пересекает границу. Объединение мелких чтений и записей с помощью `BufReader` и `BufWriter` — самое простое ускорение в
> системном коде.

Некоторые вызовы настолько часты, что Linux вообще обходится без границы. Ядро отображает в каждый процесс крошечную
библиотеку, **vDSO**; чтение часов (`clock_gettime`, которым пользуется `std::time::Instant::now()`) выполняется там, в
пространстве пользователя. Поэтому `strace` никогда не показывает эти вызовы.

## Частые ошибки

> ⚠️ **Частая ошибка:** подключаться через `strace -p PID` к процессу, который запустил не `strace`. Модуль безопасности
> Ubuntu Yama по умолчанию разрешает это только root (файл `/proc/sys/kernel/yama/ptrace_scope` содержит 1), поэтому даже
> для своей программы вы получите:
>
> ```text
> strace: attach: ptrace(PTRACE_SEIZE, 22): Operation not permitted
> ```
>
> Запускайте программу под `strace` или используйте в ВМ `sudo strace -p PID`.

> ⚠️ **Частая ошибка:** использовать номера системных вызовов другого типа процессора. `write` — это 64 на arm64, но 1 на
> x86-64. В Rust берите константы крейта `libc` (`libc::SYS_write`): они верны для той целевой платформы (target), под
> которую вы компилируете.

> ⚠️ **Частая ошибка:** забывать, что `write` может записать **меньше**, чем вы просили. Результат — число действительно
> записанных байтов, остаток нужно записать следующим вызовом. `write_all` из `std` делает цикл за вас; в упражнении 3 вы
> напишете такой цикл.

## Попробуйте сами

1. Выполните `strace -c ./target/debug/hello > /dev/null` и найдите в последней строке общее число вызовов.
2. Выполните `strace -e trace=openat cat /etc/hostname`. Какие файлы `cat` открывает кроме `/etc/hostname`?
3. Выполните `strace -e trace=write echo hi`. Сравните с вашей программой `hello`.
4. Измените `many-lines`, чтобы она печатала 100 000 строк, и сравните две версии с помощью `time` (Урок 11.3).

## Упражнения

### Упражнение 1 — Что открывает `ls`? (легко)

Выполните `strace -e trace=openat,getdents64 ls /tmp`. Найдите вызов, который открывает папку `/tmp`, и вызов, который
читает её записи. Какой результат у `getdents64`?

<details>
<summary>Подсказка</summary>

Ищите `openat(AT_FDCWD, "/tmp", O_RDONLY|O_NONBLOCK|O_CLOEXEC|O_DIRECTORY) = 3`. `getdents64` означает «get directory
entries», получить записи каталога.

</details>

<details>
<summary>Ответ</summary>

`ls` открывает `/tmp` с флагом `O_DIRECTORY` и получает дескриптор 3. Затем вызывает `getdents64(3, …)`, пока результат не
станет 0. Первый результат — число байтов записей каталога, которые ядро скопировало в буфер; последний 0 означает «записей
больше нет». До этого `ls` открывает библиотеки и файлы локали, так же как `hello` открывал `libc.so.6`.

</details>

### Упражнение 2 — `getpid` по номеру, с тестом (средне)

Напишите `src/bin/raw-getpid.rs` с функцией `fn raw_getpid() -> i64`, которая использует `libc::syscall(libc::SYS_getpid)`:
универсальную функцию библиотеки C, делающую любой системный вызов по номеру. Напечатайте номер `SYS_getpid` и оба PID.
Напишите тест `raw_getpid_matches_std`, который сравнивает результат с `std::process::id()`.

<details>
<summary>Подсказка</summary>

`libc::syscall` — `unsafe` и возвращает `c_long` (`i64` в 64-битном Linux). Сравнивайте с `i64::from(std::process::id())`.

</details>

<details>
<summary>Ответ</summary>

```rust
/// Asks the kernel for our PID with the generic `syscall` function of libc.
fn raw_getpid() -> i64 {
    // SAFETY: getpid takes no arguments and cannot fail.
    unsafe { libc::syscall(libc::SYS_getpid) }
}

fn main() {
    println!("SYS_getpid is number {} on this CPU", libc::SYS_getpid);
    println!("raw: {}, std: {}", raw_getpid(), std::process::id());
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn raw_getpid_matches_std() {
        assert_eq!(raw_getpid(), i64::from(std::process::id()));
    }
}
```

```text
SYS_getpid is number 172 on this CPU
raw: 63, std: 63
test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

Почему это работает: оба задают одному ядру один и тот же вопрос. `strace -e trace=getpid` показывает два вызова `getpid()`
с одинаковым результатом. Тест — дешёвая страховка: если бы вы взяли неверный номер, он бы упал.

</details>

### Упражнение 3 — `write_all`, который справляется с частичной записью (сложно)

Напишите `fn write_all_fd<F: AsFd>(fd: F, buf: &[u8]) -> nix::Result<usize>` с `nix::unistd::write` в цикле. Функция
должна записать каждый байт, вернуть, сколько вызовов `write` понадобилось, повторять попытку при `Errno::EINTR` и
возвращать `Err(Errno::EIO)`, если вызов записал 0 байтов. Напишите тест, который записывает 300 000 байтов в файл в
`std::env::temp_dir()` и читает их обратно.

<details>
<summary>Подсказка</summary>

Держите `buf` как сжимающийся срез: после `Ok(n)` делайте `buf = &buf[n..]`. `AsFd` находится в `std::os::fd`; `&File`
его реализует, поэтому можно передать `&file`. Урок 13.4 объясняет `AsFd` подробно.

</details>

<details>
<summary>Ответ</summary>

```rust
use nix::errno::Errno;
use std::os::fd::AsFd;

/// Writes every byte of `buf`, even if the kernel accepts only a part per call.
fn write_all_fd<F: AsFd>(fd: F, mut buf: &[u8]) -> nix::Result<usize> {
    let mut calls = 0;
    while !buf.is_empty() {
        match nix::unistd::write(fd.as_fd(), buf) {
            Ok(0) => return Err(Errno::EIO), // the kernel accepted nothing: give up
            Ok(n) => {
                buf = &buf[n..]; // drop the part that was written
                calls += 1;
            }
            Err(Errno::EINTR) => continue, // interrupted by a signal: just try again
            Err(e) => return Err(e),
        }
    }
    Ok(calls)
}

fn main() -> nix::Result<()> {
    let calls = write_all_fd(std::io::stdout(), b"all bytes arrived\n")?;
    eprintln!("({calls} write call)");
    Ok(())
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn writes_a_large_buffer_completely() {
        let path = std::env::temp_dir().join("p13-write-all-test.bin");
        let file = std::fs::File::create(&path).unwrap();
        let data = vec![7_u8; 300_000];
        write_all_fd(&file, &data).unwrap();
        drop(file);
        assert_eq!(std::fs::read(&path).unwrap(), data);
        std::fs::remove_file(&path).unwrap();
    }
}
```

```text
all bytes arrived
(1 write call)
test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

Почему это работает: цикл заканчивается только когда срез пуст, поэтому частичная запись просто приводит к следующему
вызову с оставшимися байтами. Файлы на локальном диске обычно принимают всё сразу; конвейеры и сетевые сокеты (Урок 13.6
и Этап 15) часто нет, и там этот цикл важен.

</details>

## Вопросы для самопроверки

1. Почему программа не может вызвать функцию ядра как обычную функцию?

<details>
<summary>Ответ</summary>

Память ядра защищена, а процессор выполняет программу на низком уровне привилегий. Единственный вход — особая инструкция
(`svc` на arm64, `syscall` на x86-64), которая меняет уровень и переходит в фиксированную точку входа ядра.

</details>

2. Куда в Linux на arm64 кладутся номер системного вызова и первый аргумент и откуда берётся результат?

<details>
<summary>Ответ</summary>

Номер кладётся в `x8`, первый аргумент — в `x0`, а результат возвращается в `x0`.

</details>

3. Функция libc вернула −1. Где найти причину и когда её нужно прочитать?

<details>
<summary>Ответ</summary>

В `errno` (в Rust: `std::io::Error::last_os_error()`), сразу после неудачного вызова, пока другой вызов её не перезаписал.

</details>

4. Что показывает `strace -c` и когда это полезно?

<details>
<summary>Ответ</summary>

Таблицу: сколько раз был сделан каждый системный вызов и сколько времени он занял. Это полезно, чтобы увидеть, не делает ли
программа слишком много вызовов, например по одному `write` на строку.

</details>

5. Почему `BufWriter` ускоряет программу, которая печатает много коротких строк?

<details>
<summary>Ответ</summary>

Он собирает строки в памяти и отправляет их несколькими большими вызовами `write` вместо одного вызова на строку. У каждого
системного вызова есть постоянная цена, поэтому меньше вызовов — меньше времени на границе.

</details>

6. Почему `strace` не показывает системный вызов, когда Rust читает часы через `Instant::now()`?

<details>
<summary>Ответ</summary>

Потому что чтение часов выполняется в vDSO — маленькой библиотеке, которую ядро отображает в процесс. Этот код работает в
пространстве пользователя и вообще не входит в ядро.

</details>

## Итоги

- Системный вызов — это запрос к ядру через одну особую инструкцию (`svc #0` на arm64, `syscall` на x86-64), с номером в `x8` и аргументами в `x0`–`x5`.
- Один и тот же вызов можно сделать из ассемблера, `libc`, `nix` или `std`; используйте самый высокий уровень, который справляется с задачей.
- Ошибки возвращаются отрицательными числами; libc кладёт их в `errno`, `nix` возвращает `Errno`, `std` — `io::Error`.
- `strace` печатает каждый системный вызов программы; самые полезные параметры — `-e trace=`, `-c`, `-f` и `-p`.
- Даже «hello» делает около 60 системных вызовов, большинство — для загрузки библиотек.
- Системные вызовы дороги; буферизация через `BufWriter` и `BufReader` превращает много мелких вызовов в несколько крупных.

## Источники и что почитать

- [syscall(2) — Linux manual page](https://man7.org/linux/man-pages/man2/syscall.2.html) — правила регистров для каждого типа процессора.
- [errno(3) — Linux manual page](https://man7.org/linux/man-pages/man3/errno.3.html) — все имена ошибок с их значениями.
- [strace(1) — Linux manual page](https://man7.org/linux/man-pages/man1/strace.1.html) — все параметры `strace`.
- [The Rust Reference — Inline assembly](https://doc.rust-lang.org/reference/inline-assembly.html) — полные правила `asm!`.
- [vdso(7) — Linux manual page](https://man7.org/linux/man-pages/man7/vdso.7.html) — какие вызовы предоставляет vDSO.
- [Документация крейта nix](https://docs.rs/nix/0.32.0/nix/) — безопасные обёртки, сгруппированные по фичам.

---

[🏠 Начало](../START-HERE.md) · [← 13.2 Процессы изнутри](02-processes-from-the-inside.md) · [📋 Этап 13](README.md) · [13.4 Файлы и файловые дескрипторы →](04-files-and-file-descriptors.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
