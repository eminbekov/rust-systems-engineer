<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 13.3 Системные вызовы](03-system-calls.md) · [📋 Этап 13](README.md) · [13.5 Создание процессов: fork, exec и wait →](05-fork-exec-and-wait.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/13-operating-systems/04-files-and-file-descriptors.md) <!-- nav -->

# Урок 13.4 — Файлы и файловые дескрипторы

> **Этап 13 — Операционные системы изнутри** · Урок 4 из 18 · ⏱ около 120 минут

## Что вы узнаете

- Что такое файловый дескриптор на самом деле: маленькое число, которое указывает в таблицу внутри ядра.
- Три таблицы ядра за каждым открытым файлом и почему они объясняют совместное использование и смещения.
- Как открывать, читать, писать, перемещаться и закрывать самими системными вызовами с помощью `nix`.
- Как Rust делает дескрипторы безопасными: `OwnedFd`, `BorrowedFd`, `AsFd` и `RawFd`.
- Что такое утечка дескрипторов, как увидеть её в `/proc/self/fd` и как Rust её предотвращает.

## Перед началом

- Вы прошли [Урок 13.3 — Системные вызовы](03-system-calls.md) и знаете `strace`, `errno` и крейт `nix`.
- Из [Урока 11.3 — Текстовые утилиты, конвейеры и перенаправление](../11-linux-shell-and-containers/03-text-tools-pipes-and-redirection.md) вы знаете три потока: стандартный ввод (0), стандартный вывод (1) и стандартный поток ошибок (2).
- Из [Урока 11.2 — Файловая система Linux](../11-linux-shell-and-containers/02-the-linux-file-system.md) вы знаете, что «всё есть файл» и что означают права вроде `644`.

## Новые слова

| Слово | Значение |
|---|---|
| **файловый дескриптор** (file descriptor, fd) | Маленькое неотрицательное число, которым процесс называет один открытый файл, конвейер, сокет или устройство. |
| **таблица дескрипторов** (fd table) | Таблица в ядре, своя у каждого процесса; файловый дескриптор — индекс в ней. |
| **описание открытого файла** (open file description) | Запись в ядре, которую создаёт один вызов `open`: в ней текущее **смещение** и флаги (только чтение, дозапись …). Её могут разделять несколько дескрипторов. |
| **inode** (index node, индексный узел) | Запись ядра об одном файле на диске: тип, размер, владелец, права и где лежат данные. Имени файла в ней нет. |
| **смещение** (offset) | Позиция в файле, с которой начнётся следующий `read` или `write`. |
| **перемещение** (seek) | Изменение смещения системным вызовом `lseek`. |
| **`dup`** (duplicate, дублировать) | Системный вызов, который создаёт второй дескриптор для того же описания открытого файла. |
| **EOF** (End Of File, конец файла) | Место, где читать больше нечего; `read` тогда возвращает 0. |
| **утечка дескрипторов** (descriptor leak) | Забытое закрытие дескрипторов, пока процесс не дойдёт до лимита и не сможет больше ничего открыть. |
| **`RawFd` / `OwnedFd` / `BorrowedFd`** | Типы Rust: простое число `i32` / дескриптор, которым вы владеете и который закрывается при drop / заимствованный дескриптор, который можно использовать, но нельзя закрыть. |
| **PID**, **КиБ**, **I/O** | Известны из прошлых уроков: Process IDentifier, идентификатор процесса; кибибайт (1024 байта); Input/Output, ввод-вывод — чтение и запись данных. |

## 1. Номерок на открытый файл

Сдавая пальто в гардероб театра, вы получаете маленький номерок. Номерок — не пальто; это лишь число, по которому
гардеробщик найдёт ваше пальто. **Файловый дескриптор** — такой номерок. Когда программа открывает файл, ядро хранит всё
об открытом файле в памяти ядра и даёт программе лишь маленькое число: 3, 4, 5 … Все последующие `read`, `write` или
`close` называют файл этим числом.

Каждый процесс начинает с тремя уже выданными номерками — их выдал родитель (обычно командная оболочка):

| fd | Название | Обычно подключён к |
|---|---|---|
| 0 | стандартный ввод | клавиатуре (терминалу) |
| 1 | стандартный вывод | терминалу |
| 2 | стандартный поток ошибок | терминалу |

Ядро всегда выдаёт **наименьший свободный номер**. Поэтому первый файл, который открывает программа, получает 3.

Дескриптор может указывать на гораздо большее, чем файл на диске: на терминал, конвейер (Урок 13.6), сетевой сокет
(Этап 15), `/dev/null` и даже на папку. Для вашей программы все они выглядят одинаково: числа, из которых можно читать
(`read`) и в которые можно писать (`write`). Именно это на практике и означает «всё есть файл».

```bash
ls -l /proc/self/fd
```

Настоящий вывод в терминале (строка для fd 3 — это папка, которую читает сам `ls`):

```text
total 0
lrwx------ 1 root root 64 Nov 18 07:21 0 -> /dev/pts/0
lrwx------ 1 root root 64 Nov 18 07:21 1 -> /dev/pts/0
lrwx------ 1 root root 64 Nov 18 07:21 2 -> /dev/pts/0
lr-x------ 1 root root 64 Nov 18 07:21 3 -> /proc/1/fd
```

`/dev/pts/0` — ваш терминал («pseudo-terminal slave number 0», подчинённый псевдотерминал номер 0). Все три стандартных
потока указывают на него.

## 2. Три таблицы в ядре

За дескриптором стоят три слоя. Если их понять, объясняются почти все неожиданности с файлами.

```text
 process A                        kernel                                       disk
 fd table                  open file descriptions                       inodes
 +----+---------+          +--------------------------+          +-------------------+
 | 0  | --------|--------->| /dev/pts/0, offset -, rw |--------->| terminal device    |
 | 1  | --------|----/     +--------------------------+          +-------------------+
 | 2  | --------|---/
 | 3  | --------|--------->| offset 7, flags O_RDWR   |---+----->| inode 1234        |
 | 4  | --------|---+      +--------------------------+   |      | size 14, rw-r--r--|
 +----+---------+   +----->| offset 0, flags O_RDONLY |---+      | data blocks ...   |
                           +--------------------------+          +-------------------+
```

1. **Таблица дескрипторов** принадлежит одному процессу. Запись 3 говорит «иди к вот этому описанию открытого файла».
   В ней же хранится один флаг на дескриптор — `FD_CLOEXEC` (раздел 5).
2. **Описание открытого файла** создаётся каждым вызовом `open`. Оно помнит **смещение** (где будет следующее чтение или
   запись) и то, как файл открыт (чтение, запись, дозапись …).
3. **Inode** — это сам файл: размер, владелец, права, данные. Имена файлов живут в папках и указывают на inode
   (`ls -i` показывает номера inode).

На рисунке дескрипторы 3 и 4 открыли один и тот же файл дважды. У них общий inode, поэтому они видят одни и те же
данные, но **разные смещения**. Два дескриптора могут разделять и одно описание открытого файла; разницу вы увидите в
разделе 4.

## 3. Open, read, write, seek, close

`std::fs::File` делает всё это за вас, но один раз стоит сделать это самими системными вызовами. Создайте
`src/bin/fd-basics.rs` (фича `fs` крейта `nix` уже включена в Уроке 13.3):

```rust
use nix::fcntl::{OFlag, open};
use nix::sys::stat::Mode;
use nix::unistd::{Whence, lseek, read, write};
use std::os::fd::AsRawFd;

fn main() -> nix::Result<()> {
    let flags = OFlag::O_RDWR | OFlag::O_CREAT | OFlag::O_TRUNC;
    let fd = open("/tmp/notes.txt", flags, Mode::from_bits_truncate(0o644))?;
    println!("open gave us descriptor {}", fd.as_raw_fd());

    let n = write(&fd, b"hello, kernel\n")?;
    println!(
        "wrote {n} bytes; offset is now {}",
        lseek(&fd, 0, Whence::SeekCur)?
    );

    lseek(&fd, 7, Whence::SeekSet)?; // move the offset back to byte 7
    let mut buf = [0_u8; 64];
    let n = read(&fd, &mut buf)?;
    println!(
        "read {n} bytes from offset 7: {:?}",
        String::from_utf8_lossy(&buf[..n])
    );

    let n = read(&fd, &mut buf)?;
    println!("next read gave {n} bytes: end of file");
    Ok(()) // `fd` is an OwnedFd: dropping it here calls close()
}
```

```text
open gave us descriptor 3
wrote 14 bytes; offset is now 14
read 7 bytes from offset 7: "kernel\n"
next read gave 0 bytes: end of file
```

**Что происходит:**

1. `open` принимает путь, **флаги** и **режим** (mode). `O_RDWR` — открыть для чтения и записи, `O_CREAT` — создать файл,
   если его нет, `O_TRUNC` — сделать его пустым, если он есть. Режим `0o644` (`rw-r--r--`, Урок 11.4) используется только
   при создании файла. Префикс `0o` записывает число в восьмеричной системе (Урок 12.1).
2. Результат — `OwnedFd` с номером 3: 0, 1 и 2 были заняты.
3. `write` записывает 14 байтов со смещения 0 и сдвигает смещение на 14. `lseek(&fd, 0, Whence::SeekCur)` означает
   «сдвинься на 0 от текущей позиции» и возвращает позицию: обычный способ спросить «где я?».
4. `lseek(&fd, 7, Whence::SeekSet)` ставит смещение на байт 7. `read` возвращает 7 байтов оттуда до конца.
5. Следующий `read` возвращает **0**. Для файла 0 означает EOF. Это не ошибка: так ядро говорит «больше ничего нет».
6. Когда `main` заканчивается, `fd` уничтожается (drop), и деструктор `OwnedFd` вызывает `close(3)`.

> ⚠️ **Частая ошибка:** ожидать, что `read` заполнит весь буфер. `read(fd, &mut buf)` возвращает **до** `buf.len()`
> байтов: меньше, если в файле осталось меньше, и часто меньше для конвейеров и сокетов. Всегда используйте возвращённое
> число, как в `&buf[..n]`, а не весь буфер.

## 4. Два open против одного `dup`

Вот эксперимент, который показывает, зачем нужна средняя таблица. Создайте `src/bin/offsets.rs`:

```rust
use std::fs::{self, File, OpenOptions};
use std::io::Write;

fn main() -> std::io::Result<()> {
    let path = "/tmp/offsets.txt";

    // Two separate open() calls: two open file descriptions, two offsets.
    let mut a = File::create(path)?;
    let mut b = OpenOptions::new().write(true).open(path)?;
    a.write_all(b"AAAA")?;
    b.write_all(b"BB")?;
    println!("two opens: {:?}", fs::read_to_string(path)?);

    // try_clone() calls dup(): two descriptors, ONE open file description, one shared offset.
    let mut c = File::create(path)?;
    let mut d = c.try_clone()?;
    c.write_all(b"AAAA")?;
    d.write_all(b"BB")?;
    println!("dup:       {:?}", fs::read_to_string(path)?);
    Ok(())
}
```

```text
two opens: "BBAA"
dup:       "AAAABB"
```

**Что происходит:**

1. `a` и `b` получены двумя вызовами `open`, поэтому у каждого своё описание открытого файла и своё смещение, оба
   начинаются с 0. `a` пишет `AAAA` в байты 0–3. `b` пишет `BB` со **своего** смещения 0, поверх первых двух `A`: `BBAA`.
2. `File::try_clone` использует системный вызов `dup` (строго говоря, `fcntl` с `F_DUPFD_CLOEXEC`, который делает то же
   самое и ставит флаг из раздела 5). `c` и `d` — два номера для **одного** описания открытого файла, поэтому смещение у
   них общее. После того как `c` записал 4 байта, общее смещение равно 4, и `d` продолжает с этого места: `AAAABB`.

Именно на это опирается командная оболочка в `command > out.txt 2>&1`: стандартный вывод и стандартный поток ошибок
становятся двумя дескрипторами одного описания открытого файла, поэтому их тексты идут друг за другом, а не перезаписывают
друг друга. Урок 13.6 строит такое перенаправление.

> 💡 **Совет:** если несколько программ дописывают в один файл журнала, открывайте его с `O_APPEND`
> (`OpenOptions::new().append(true)`). Тогда ядро переносит смещение в конец **внутри** каждого `write`, и строки разных
> процессов никогда не перезаписывают друг друга.

## 5. Дескрипторы в Rust: кто их закрывает?

На уровне системных вызовов дескриптор — просто `i32`. Ничто не мешает программе на C использовать номер после закрытия,
закрыть его дважды или забыть закрыть. Rust ставит дескрипторы под **владение** (Этап 03) с помощью четырёх типов из
`std::os::fd`:

| Тип | Что это | Закрывается при drop? |
|---|---|---|
| `RawFd` | просто `i32`, без правил | нет |
| `OwnedFd` | дескриптор, которым вы владеете | **да** |
| `BorrowedFd<'a>` | заимствованный дескриптор, действительный в течение времени жизни `'a` | нет |
| `File`, `TcpStream`, `Stdout` … | типы, содержащие `OwnedFd` (или заимствующие 0, 1, 2 процесса) | как решает их владелец |

Трейт `AsFd` означает «у меня можно позаимствовать `BorrowedFd`». Функции `nix` принимают `impl AsFd`, поэтому в примерах
можно было передавать `&fd`, `&file` или `std::io::stdout()`. Превратить сырое число в `OwnedFd` можно только с `unsafe`
(`OwnedFd::from_raw_fd`), потому что компилятор не может проверить, что вы действительно владеете этим номером.

Посмотрим, как владение закрывает дескрипторы. Создайте `src/bin/fd-table.rs`:

```rust
use std::fs::{self, File};

fn print_fds(when: &str) {
    println!("{when}:");
    let mut entries: Vec<_> = fs::read_dir("/proc/self/fd").unwrap().flatten().collect();
    entries.sort_by_key(|e| e.file_name().to_string_lossy().parse::<u32>().unwrap_or(0));
    for entry in entries {
        let target = fs::read_link(entry.path()).unwrap_or_default();
        println!(
            "  {} -> {}",
            entry.file_name().to_string_lossy(),
            target.display()
        );
    }
}

fn main() -> std::io::Result<()> {
    print_fds("at start");
    {
        let _passwd = File::open("/etc/passwd")?;
        let _hostname = File::open("/etc/hostname")?;
        print_fds("with two files open");
    } // both File values are dropped here, and their descriptors are closed
    print_fds("after the block");
    Ok(())
}
```

```text
at start:
  0 -> /dev/pts/0
  1 -> /dev/pts/0
  2 -> /dev/pts/0
  3 -> /proc/1/fd
with two files open:
  0 -> /dev/pts/0
  1 -> /dev/pts/0
  2 -> /dev/pts/0
  3 -> /etc/passwd
  4 -> /etc/hostname
  5 -> /proc/1/fd
after the block:
  0 -> /dev/pts/0
  1 -> /dev/pts/0
  2 -> /dev/pts/0
  3 -> /proc/1/fd
```

**Что происходит:**

1. Каждая запись `/proc/self/fd` — символическая ссылка от номера к тому, на что он указывает; `fs::read_link` её читает.
2. Самая старшая запись — сама папка `/proc/self/fd`, которую открыл `read_dir`, пока мы её перечисляем (здесь `self`
   раскрылось в PID 1 — программу в контейнере).
3. Внутри блока два файла получили 3 и 4, наименьшие свободные номера, а `read_dir` — 5.
4. На `}` два значения `File` выходят из области видимости; их деструкторы вызывают `close`, и номера снова свободны.

Запустите `./target/debug/fd-table < /etc/hostname 2>/dev/null`, и fd 0 будет указывать на `/etc/hostname`, а fd 2 — на
`/dev/null`: командная оболочка изменила таблицу перед запуском программы. Это перенаправление, тема Урока 13.6.

### Закрытие при exec

Когда процесс запускает другую программу (Урок 13.5), новая программа **наследует** таблицу дескрипторов. Обычно вы не
хотите отдавать дочерней программе свои открытые файлы, сокеты или секреты. Флаг `FD_CLOEXEC` («close on exec», закрыть при
exec) просит ядро автоматически закрыть этот дескриптор, когда запускается новая программа. `std` в Rust ставит его на
каждый создаваемый дескриптор: это тот самый `O_CLOEXEC`, который вы видели в каждой строке `openat` в `strace` в Уроке 13.3.

> 🔒 **Безопасность:** утёкший в дочерний процесс дескриптор — настоящий класс уязвимостей: дочерняя программа может
> прочитать секретный файл или писать в сокет, которым должен пользоваться только родитель. Когда открываете дескрипторы
> через `nix` или `libc`, добавляйте `OFlag::O_CLOEXEC`, кроме случаев, когда ребёнок действительно должен их унаследовать.

## Частые ошибки

> ⚠️ **Частая ошибка:** утечка дескрипторов. В долго работающем сервере дескриптор, который никогда не закрывается,
> остаётся в таблице навсегда. Ядро ограничивает, сколько их может быть у процесса (`ulimit -n`, часто 1024). Эта программа
> намеренно допускает утечку (`src/bin/leak.rs`):
>
> ```rust
> use std::fs::File;
>
> fn main() {
>     for i in 1.. {
>         match File::open("/etc/hostname") {
>             // BUG on purpose: forget() never runs the destructor, so close() is never called.
>             Ok(file) => std::mem::forget(file),
>             Err(e) => {
>                 println!("open number {i} failed: {e}");
>                 break;
>             }
>         }
>     }
> }
> ```
>
> ```bash
> (ulimit -n 10; ./target/debug/leak)
> ```
>
> ```text
> open number 8 failed: Too many open files (os error 24)
> ```
>
> Ошибка 24 — это `EMFILE`. При лимите 10 дескрипторы 0–2 заняты, а 3–9 утекли в первых семи вызовах open. Скобки
> запускают команды в подоболочке, поэтому низкий лимит не остаётся в вашей оболочке. В обычном коде на Rust закрытием
> занимается drop; утечки возникают из-за `mem::forget`, `into_raw_fd` или значений, навсегда оставленных в коллекции.

> ⚠️ **Частая ошибка:** использовать дескриптор после закрытия. В C это молча читает из того файла, который позже получил
> тот же номер. В Rust закрытого `OwnedFd` больше не существует, поэтому компилятор отвергает такое использование
> (`error[E0382]: borrow of moved value` после `drop(fd)`).

## Попробуйте сами

1. Выполните `ls -l /proc/self/fd`, а затем `ls -l /proc/self/fd < /etc/hostname > /tmp/out.txt; cat /tmp/out.txt`.
   Какие дескрипторы изменились?
2. Запустите `fd-basics`, затем `ls -i /tmp/notes.txt` и `stat /tmp/notes.txt`. Найдите номер inode и размер 14.
3. Измените `offsets.rs`, чтобы `b` открывался с `.append(true)` вместо `.write(true)`, и объясните новый результат.
4. Выполните `strace -e trace=openat,close ./target/debug/fd-table`. Найдите `close(3)` и `close(4)` в конце блока.

## Упражнения

### Упражнение 1 — Сколько дескрипторов? (легко)

Напишите `src/bin/count-fds.rs` с функцией `fn count_open_fds() -> usize`, которая считает записи `/proc/self/fd`, не
считая той, что использует сам `read_dir`. Напечатайте число в `main`.

<details>
<summary>Подсказка</summary>

`fs::read_dir("/proc/self/fd").map(|d| d.count())` даёт число записей, включая собственный дескриптор папки. Вычтите 1.

</details>

<details>
<summary>Ответ</summary>

См. упражнение 2: тот же файл, с добавленным тестом.

```text
open descriptors now: 3
```

Почему это работает: 0, 1 и 2 открыты в каждом обычном процессе; дескриптор `read_dir` вычитается.

</details>

### Упражнение 2 — Тест, который доказывает, что `drop` закрывает (средне)

Добавьте в `count-fds.rs` тест `a_file_uses_one_descriptor_until_dropped`: посчитайте дескрипторы, откройте
`/proc/self/status`, проверьте, что число стало на один больше, уничтожьте файл через drop, проверьте, что число вернулось.

<details>
<summary>Подсказка</summary>

Тесты одного бинарника работают в параллельных потоках **одного** процесса и разделяют таблицу дескрипторов. Если тест в
файле один, всё в порядке; если их несколько, другой тест может открыть файл в тот же момент, и тест будет иногда падать.

</details>

<details>
<summary>Ответ</summary>

```rust
use std::fs;

/// Counts the open file descriptors of this process.
fn count_open_fds() -> usize {
    // read_dir itself opens one descriptor for the folder; do not count it.
    fs::read_dir("/proc/self/fd")
        .map(|d| d.count())
        .unwrap_or(0)
        - 1
}

fn main() {
    println!("open descriptors now: {}", count_open_fds());
}

#[cfg(test)]
mod tests {
    use super::*;
    use std::fs::File;

    #[test]
    fn a_file_uses_one_descriptor_until_dropped() {
        let before = count_open_fds();
        let file = File::open("/proc/self/status").unwrap();
        assert_eq!(count_open_fds(), before + 1);
        drop(file);
        assert_eq!(count_open_fds(), before);
    }
}
```

```text
test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

Почему это работает: `File::open` добавляет одну запись в таблицу дескрипторов, а `drop` вызывает `close`, который её убирает.

> 🧪 **Тестирование:** тесты, которые смотрят на общее состояние процесса (таблицу дескрипторов, текущую папку, переменные
> окружения), могут мешать друг другу, потому что `cargo test` запускает тесты в параллельных потоках. Держите такие тесты в
> отдельном бинарнике или запускайте их командой `cargo test -- --test-threads=1`.

</details>

### Упражнение 3 — `cp` на системных вызовах (сложно)

Напишите `src/bin/copy.rs`: `copy <from> <to>` копирует файл с помощью `nix::fcntl::open`, `nix::unistd::read` и `write`
через буфер 64 КиБ. Вынесите цикл в `fn copy_fd<S: AsFd, D: AsFd>(src: S, dst: D) -> nix::Result<(u64, u32)>`, которая
возвращает число байтов и число вызовов `read`. Напишите тест, который копирует 200 000 байтов и проверяет содержимое и то,
что понадобилось ровно 5 вызовов `read`. Затем сравните через `strace` с `std::fs::copy`.

<details>
<summary>Подсказка</summary>

Читайте, пока `read` не вернёт 0. `write` может записать меньше, чем просили, поэтому записывайте каждый кусок во
внутреннем цикле (как в Уроке 13.3). 200 000 / 65 536 = 3,05, значит, будет четыре чтения с данными и одно, которое
вернёт 0.

</details>

<details>
<summary>Ответ</summary>

```rust
use nix::fcntl::{OFlag, open};
use nix::sys::stat::Mode;
use nix::unistd::{read, write};
use std::os::fd::AsFd;

/// Copies everything from `src` to `dst` with a fixed buffer. Returns (bytes, read calls).
fn copy_fd<S: AsFd, D: AsFd>(src: S, dst: D) -> nix::Result<(u64, u32)> {
    let mut buf = vec![0_u8; 64 * 1024];
    let (mut total, mut reads) = (0_u64, 0_u32);
    loop {
        let n = read(src.as_fd(), &mut buf)?;
        reads += 1;
        if n == 0 {
            return Ok((total, reads)); // 0 means end of file
        }
        let mut chunk = &buf[..n];
        while !chunk.is_empty() {
            let written = write(dst.as_fd(), chunk)?;
            chunk = &chunk[written..];
        }
        total += n as u64;
    }
}

fn main() -> nix::Result<()> {
    let args: Vec<String> = std::env::args().collect();
    if args.len() != 3 {
        eprintln!("usage: copy <from> <to>");
        std::process::exit(2);
    }
    let src = open(args[1].as_str(), OFlag::O_RDONLY, Mode::empty())?;
    let flags = OFlag::O_WRONLY | OFlag::O_CREAT | OFlag::O_TRUNC;
    let dst = open(args[2].as_str(), flags, Mode::from_bits_truncate(0o644))?;
    let (bytes, reads) = copy_fd(&src, &dst)?;
    println!("copied {bytes} bytes with {reads} read calls");
    Ok(())
}

#[cfg(test)]
mod tests {
    use super::*;
    use std::fs;

    #[test]
    fn copies_a_file_larger_than_the_buffer() {
        let dir = std::env::temp_dir();
        let (from, to) = (dir.join("p13-copy-from"), dir.join("p13-copy-to"));
        let data: Vec<u8> = (0..200_000_u32).map(|i| (i % 251) as u8).collect();
        fs::write(&from, &data).unwrap();
        let src = fs::File::open(&from).unwrap();
        let dst = fs::File::create(&to).unwrap();
        let (bytes, reads) = copy_fd(&src, &dst).unwrap();
        assert_eq!(bytes, 200_000);
        assert_eq!(reads, 5); // 4 full or partial buffers + the final 0
        assert_eq!(fs::read(&to).unwrap(), data);
    }
}
```

```bash
head -c 1000000 /dev/urandom > /tmp/big
./target/debug/copy /tmp/big /tmp/big2
cmp /tmp/big /tmp/big2 && echo same
```

```text
copied 1000000 bytes with 17 read calls
same
```

Под `strace` маленькая программа, вызывающая `std::fs::copy("/tmp/big", "/tmp/big-copy")`, вообще не показывает `read` и
`write` для данных:

```text
copy_file_range(3, NULL, 4, NULL, 1073741824, 0) = 1000000
copy_file_range(3, NULL, 4, NULL, 1073741824, 0) = 0
```

Почему это работает: ваш цикл проводит каждый байт через ваш буфер, за 17 раундов `read` плюс `write`. `std::fs::copy`
использует системный вызов Linux `copy_file_range`, который говорит ядру «скопируй из дескриптора 3 в дескриптор 4» и держит
данные внутри ядра: один вызов вместо 34 и никакого копирования в пространство пользователя.

</details>

## Вопросы для самопроверки

1. Ваша программа открывает два файла. Какие номера дескрипторов она скорее всего получит и почему?

<details>
<summary>Ответ</summary>

3 и 4. 0, 1 и 2 уже заняты стандартными потоками, а ядро всегда выдаёт наименьший свободный номер.

</details>

2. Что означает, что `read` вернул 0 для файла?

<details>
<summary>Ответ</summary>

Конец файла: читать больше нечего. Это не ошибка.

</details>

3. Два дескриптора указывают на один файл. Когда у них общее смещение, а когда нет?

<details>
<summary>Ответ</summary>

Общее — когда они указывают на одно описание открытого файла; так бывает после `dup` (или `try_clone`, или когда ребёнок
его наследует). Два отдельных вызова `open` создают два описания открытого файла, каждое со своим смещением.

</details>

4. Чем `OwnedFd` отличается от `BorrowedFd`?

<details>
<summary>Ответ</summary>

`OwnedFd` владеет дескриптором и закрывает его при уничтожении. `BorrowedFd<'a>` лишь заимствует его на время жизни; им
можно пользоваться, но он никогда не закрывает дескриптор.

</details>

5. Зачем `std` открывает каждый файл с `O_CLOEXEC`?

<details>
<summary>Ответ</summary>

Чтобы дескриптор автоматически закрылся, когда процесс запускает другую программу. Иначе дочерняя программа унаследует
открытые файлы и сокеты, которых у неё быть не должно.

</details>

6. Сервер через несколько дней падает с `Too many open files (os error 24)`. Какая ошибка вероятна и как её подтвердить?

<details>
<summary>Ответ</summary>

Утечка дескрипторов: какие-то дескрипторы открываются, но никогда не закрываются. Подтвердить можно, наблюдая, как растёт
`ls /proc/<pid>/fd | wc -l`, и посмотрев, на что указывают утёкшие записи.

</details>

## Итоги

- Файловый дескриптор — маленькое число, индекс в таблице дескрипторов процесса; 0, 1 и 2 — стандартные потоки, а новые дескрипторы получают наименьший свободный номер.
- За ним три слоя: таблица дескрипторов (своя у процесса), описание открытого файла (смещение и флаги, создаётся `open`) и inode (сам файл).
- `open`, `read`, `write`, `lseek` и `close` — основные системные вызовы; `read` может вернуть меньше байтов, чем просили, а 0 означает EOF.
- Два вызова `open` дают отдельные смещения; `dup` даёт два дескриптора с одним общим смещением.
- `OwnedFd` в Rust закрывается при drop, `BorrowedFd` заимствует, а `AsFd` позволяет функциям принимать любой тип, у которого есть дескриптор.
- `FD_CLOEXEC` закрывает дескриптор при запуске новой программы; `std` ставит его на всё, что открывает.
- `/proc/<pid>/fd` показывает каждый открытый дескриптор; это первое место, где стоит искать утечку.

## Источники и что почитать

- [open(2) — Linux manual page](https://man7.org/linux/man-pages/man2/open.2.html) — все флаги и понятное объяснение описаний открытых файлов.
- [dup(2) — Linux manual page](https://man7.org/linux/man-pages/man2/dup.2.html) — `dup`, `dup2` и `dup3`.
- [std::os::fd — Rust documentation](https://doc.rust-lang.org/std/os/fd/index.html) — `OwnedFd`, `BorrowedFd`, `AsFd` и правила безопасного ввода-вывода (I/O safety).
- [RFC 3128 — I/O safety](https://rust-lang.github.io/rfcs/3128-io-safety.html) — RFC (Request for Comments, письменное предложение об изменении Rust), которое объясняет, почему Rust обращается с дескрипторами как с ресурсами, которыми владеют.
- [copy_file_range(2) — Linux manual page](https://man7.org/linux/man-pages/man2/copy_file_range.2.html) — копирование внутри ядра.

---

[🏠 Начало](../START-HERE.md) · [← 13.3 Системные вызовы](03-system-calls.md) · [📋 Этап 13](README.md) · [13.5 Создание процессов: fork, exec и wait →](05-fork-exec-and-wait.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
