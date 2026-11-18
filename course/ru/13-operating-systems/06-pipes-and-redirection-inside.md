<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 13.5 Создание процессов: fork, exec и wait](05-fork-exec-and-wait.md) · [📋 Этап 13](README.md) · [13.7 Сигналы в ваших программах →](07-signals-in-your-programs.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/13-operating-systems/06-pipes-and-redirection-inside.md) <!-- nav -->

# Урок 13.6 — Конвейеры и перенаправление изнутри

> **Этап 13 — Операционные системы изнутри** · Урок 6 из 18 · ⏱ около 135 минут

## Что вы узнаете

- Что такое конвейер (pipe) внутри ядра: маленький буфер с концом для чтения и концом для записи.
- Как работает конец файла для конвейеров и почему один незакрытый дескриптор заставляет программу висеть вечно.
- Как `dup2` перенаправляет стандартный ввод и вывод и как оболочка строит `ls | wc -l` и `> file 2>&1` из `pipe`, `fork`, `dup2` и `exec`.
- Что такое `SIGPIPE` и `EPIPE` и почему программа на Rust, выход которой передан в `head`, может запаниковать.
- Как `std::process::Stdio` безопасно делает ту же работу.

## Перед началом

- Вы прошли [Урок 13.5 — Создание процессов: fork, exec и wait](05-fork-exec-and-wait.md).
- Вы знаете три таблицы ядра, `dup` и `O_CLOEXEC` из [Урока 13.4](04-files-and-file-descriptors.md).
- Из [Урока 11.3 — Текстовые утилиты, конвейеры и перенаправление](../11-linux-shell-and-containers/03-text-tools-pipes-and-redirection.md) вы умеете *пользоваться* `|`, `>`, `<` и `2>&1` в оболочке. Теперь вы их построите.

## Новые слова

| Слово | Значение |
|---|---|
| **конвейер** (pipe) | Односторонний канал внутри ядра: байты, записанные в его конец для записи, выходят из конца для чтения в том же порядке. |
| **конец для чтения / конец для записи** (read end / write end) | Два файловых дескриптора, которые возвращает `pipe`: один только для чтения, другой только для записи. |
| **буфер конвейера** (pipe buffer) | Память ядра, где хранятся записанные, но ещё не прочитанные байты; в Linux по умолчанию 64 КиБ. |
| **блокирующий вызов** (blocking) | Вызов, который усыпляет процесс, пока не сможет завершиться: `read` из пустого конвейера, `write` в полный. |
| **`dup2`** | Системный вызов, который делает так, что дескриптор номер B указывает на то же описание открытого файла, что и A, при необходимости сначала закрыв B. |
| **перенаправление** (redirection) | Изменение того, на что указывают дескрипторы 0, 1 или 2 программы, до её запуска. |
| **`EPIPE`** / **`SIGPIPE`** | Ошибка / сигнал, которые вы получаете при записи в конвейер, у которого закрыты все концы для чтения («broken pipe», разорванный конвейер). |
| **`EAGAIN`** | «Попробуйте снова»: ошибка, которую неблокирующий вызов возвращает вместо ожидания. |
| **PID**, **КиБ**/**МиБ**, **I/O** | Известны из прошлых уроков: Process IDentifier, идентификатор процесса; кибибайт и мебибайт (1024 байта, 1024 КиБ); Input/Output, ввод-вывод. |

## 1. Конвейер — это буфер с двумя концами

Представьте конвейер как узкую трубку между двумя комнатами: записки, брошенные в один конец, выходят из другого в том же
порядке. Если никто не забирает записки, трубка заполняется, и пишущему приходится ждать.

Системный вызов `pipe` создаёт такую трубку внутри ядра и возвращает **два** дескриптора. Создайте `src/bin/pipe-basic.rs`:

```rust
use nix::unistd::{pipe, read, write};
use std::os::fd::AsRawFd;

fn main() -> nix::Result<()> {
    let (reader, writer) = pipe()?;
    println!(
        "read end: fd {}, write end: fd {}",
        reader.as_raw_fd(),
        writer.as_raw_fd()
    );

    write(&writer, b"through the kernel\n")?;
    let mut buf = [0_u8; 64];
    let n = read(&reader, &mut buf)?;
    print!("got {n} bytes: {}", String::from_utf8_lossy(&buf[..n]));

    drop(writer); // close the only write end
    let n = read(&reader, &mut buf)?;
    println!("after closing the write end, read returns {n}: end of file");
    Ok(())
}
```

```text
read end: fd 3, write end: fd 4
got 19 bytes: through the kernel
after closing the write end, read returns 0: end of file
```

**Что происходит:**

1. `pipe()` возвращает два `OwnedFd`: из fd 3 можно только читать, в fd 4 можно только писать.
2. `write` копирует 19 байтов в буфер конвейера внутри ядра; `read` копирует их обратно. Байты выходят в том порядке, в
   котором вошли, и каждый байт читается один раз.
3. После `drop(writer)` концов для записи больше нет. `read` из пустого конвейера без пишущих возвращает **0**: конец файла,
   тот же сигнал, что и в конце файла на диске.

Правила конвейера, которые объясняют всё в этом уроке:

| Ситуация | Что делает ядро |
|---|---|
| `read`, в конвейере есть данные | сразу возвращает до запрошенного числа байтов |
| `read`, конвейер пуст, конец для записи ещё открыт **где-нибудь** | читающий **блокируется** (спит), пока не придут данные |
| `read`, конвейер пуст, **все** концы для записи закрыты | возвращает 0 (конец файла) |
| `write`, в буфере есть место | копирует байты и возвращается |
| `write`, буфер полон (64 КиБ) | пишущий **блокируется**, пока читающий не освободит место |
| `write`, **все** концы для чтения закрыты | ошибка: `SIGPIPE` и `EPIPE` (раздел 5) |

Ключевое слово в третьей строке — «где-нибудь». Ядро считает концы для записи **во всех процессах**. Пока хотя бы один
процесс где угодно держит конец для записи, читающий ждёт.

> ⚡ **Производительность:** правило блокировки даёт конвейерам автоматическое управление потоком. Если `wc` медленнее `ls`,
> буфер заполняется, `ls` засыпает, и расход памяти остаётся на уровне 64 КиБ, сколько бы данных ни шло. Это простая форма
> **обратного давления** (backpressure): медленный потребитель замедляет производителя, а не захлёбывается.

## 2. Конвейер между двумя процессами

Конвейер внутри одного процесса не очень полезен. Весь смысл в том, что `fork` копирует таблицу дескрипторов, поэтому после
`fork` и родитель, и ребёнок держат **оба** конца. Каждая сторона закрывает ненужный ей конец. Создайте
`src/bin/pipe-fork.rs`:

```rust
use nix::sys::wait::waitpid;
use nix::unistd::{ForkResult, fork, pipe, write};
use std::fs::File;
use std::io::{BufRead, BufReader};

fn main() -> nix::Result<()> {
    let (reader, writer) = pipe()?;

    // SAFETY: single-threaded program.
    match unsafe { fork() }? {
        ForkResult::Child => {
            drop(reader); // the child only writes
            for i in 1..=3 {
                write(&writer, format!("message {i} from the child\n").as_bytes())?;
            }
            std::process::exit(0); // exiting closes the child's write end
        }
        ForkResult::Parent { child } => {
            drop(writer); // the parent only reads: close its copy of the write end
            let lines = BufReader::new(File::from(reader)).lines();
            for line in lines {
                println!("parent got: {}", line.unwrap());
            }
            println!("parent: end of file, the child is done");
            waitpid(child, None)?;
        }
    }
    Ok(())
}
```

```text
parent got: message 1 from the child
parent got: message 2 from the child
parent got: message 3 from the child
parent: end of file, the child is done
```

**Что происходит:**

1. После `fork` существуют два конца для записи: у родителя и у ребёнка. Оба указывают на один конвейер.
2. Родитель сразу закрывает **свой** конец для записи. Теперь он есть только у ребёнка.
3. `File::from(reader)` превращает `OwnedFd` в `File`, чтобы можно было использовать `BufReader::lines()` с Этапа 04.
4. Когда ребёнок завершается, ядро закрывает его дескрипторы. Последний конец для записи исчез, `read` родителя возвращает 0,
   и цикл `lines()` заканчивается.

> ⚠️ **Частая ошибка:** не закрыть конец для записи у родителя. Удалите `drop(writer);` в ветке родителя и запустите программу:
> она напечатает три сообщения и зависнет навсегда. Родитель ждёт конца файла, но сам держит конец для записи, поэтому конец
> файла никогда не наступит. Остановите программу через Ctrl-C. У каждой зависшей программы с конвейером, которую вам
> придётся отлаживать, одна и та же причина: какой-то процесс всё ещё держит конец для записи.

## 3. `dup2`: перенаправление дескрипторов 0, 1 и 2

Программы вроде `ls` и `wc` ничего не знают о конвейерах. `ls` пишет в дескриптор 1, `wc` читает из дескриптора 0. Чтобы их
соединить, ребёнок должен сделать так, чтобы **его дескриптор 1 указывал на конвейер**, прежде чем вызвать `exec`. Это и
делает `dup2`.

`dup2(old, new)` делает дескриптор `new` копией `old`: после вызова оба номера указывают на одно описание открытого файла
(Урок 13.4). Если `new` был открыт, он сначала закрывается. В `nix` для трёх стандартных номеров есть `dup2_stdin(fd)`,
`dup2_stdout(fd)` и `dup2_stderr(fd)`.

```text
 child of "ls", before dup2_stdout(writer)     after dup2_stdout(writer)
   fd 1 -> terminal                              fd 1 -> pipe (write end)
   fd 4 -> pipe (write end)                      fd 4 -> pipe (write end)  (closed by O_CLOEXEC at exec)
```

Теперь `ls /etc | wc -l` из оболочки, собранный вручную. Создайте `src/bin/pipeline.rs`:

```rust
use nix::fcntl::OFlag;
use nix::sys::wait::waitpid;
use nix::unistd::{ForkResult, Pid, dup2_stdin, dup2_stdout, execvp, fork, pipe2};
use std::ffi::CString;
use std::os::fd::OwnedFd;

/// Forks a child that connects the given descriptors to its stdin/stdout and runs `argv`.
fn spawn(argv: &[&str], stdin: Option<&OwnedFd>, stdout: Option<&OwnedFd>) -> nix::Result<Pid> {
    let args: Vec<CString> = argv.iter().map(|a| CString::new(*a).unwrap()).collect();
    // SAFETY: single-threaded program.
    match unsafe { fork() }? {
        ForkResult::Child => {
            if let Some(fd) = stdin {
                dup2_stdin(fd)?; // fd 0 now points to the pipe's read end
            }
            if let Some(fd) = stdout {
                dup2_stdout(fd)?; // fd 1 now points to the pipe's write end
            }
            let err = execvp(&args[0], &args).unwrap_err();
            eprintln!("{}: {err}", argv[0]);
            std::process::exit(127);
        }
        ForkResult::Parent { child } => Ok(child),
    }
}

fn main() -> nix::Result<()> {
    // ls /etc | wc -l
    // O_CLOEXEC: the original pipe descriptors close by themselves when a child calls exec.
    // Only the copies made by dup2 (fd 0 and fd 1) survive into `ls` and `wc`.
    let (reader, writer) = pipe2(OFlag::O_CLOEXEC)?;
    let left = spawn(&["ls", "/etc"], None, Some(&writer))?;
    let right = spawn(&["wc", "-l"], Some(&reader), None)?;
    drop(reader); // the parent must close both ends,
    drop(writer); // or `wc` never sees end of file
    println!("ls: {:?}", waitpid(left, None)?);
    println!("wc: {:?}", waitpid(right, None)?);
    Ok(())
}
```

```bash
./target/debug/pipeline
ls /etc | wc -l
```

```text
94
ls: Exited(Pid(379), 0)
wc: Exited(Pid(380), 0)
94
```

**Что происходит:**

1. `pipe2(OFlag::O_CLOEXEC)` — это `pipe` с флагами: оба конца получают `FD_CLOEXEC`.
2. Первый ребёнок подключает fd 1 к концу для записи и становится `ls`. Его вывод идёт в конвейер, а не в терминал.
3. Второй ребёнок подключает fd 0 к концу для чтения и становится `wc -l`, который считает строки, приходящие по конвейеру.
4. `dup2` **не** копирует флаг `FD_CLOEXEC`. Поэтому при `exec` исходные дескрипторы конвейера (3 и 4) закрываются
   автоматически, а копии на 0 и 1 остаются открытыми. У каждой программы оказывается ровно тот конец, который ей нужен.
5. Родитель закрывает оба конца и ждёт обоих детей. Наша программа и настоящая оболочка печатают одно и то же 94.

Под `strace` вы увидите `dup3(4, 1, 0)` вместо `dup2`: в Linux на arm64 нет отдельного системного вызова `dup2`, и библиотека C использует `dup3` с флагами 0, который делает то же самое.

> ⚠️ **Частая ошибка:** забыть здесь `O_CLOEXEC`. Замените `pipe2(OFlag::O_CLOEXEC)` на `pipe()` и запустите с
> ограничением по времени:
>
> ```bash
> timeout 3 ./target/debug/pipeline; echo "exit code $?"
> ```
>
> ```text
> ls: Exited(Pid(385), 0)
> exit code 124
> ```
>
> `ls` завершается, а `wc` — никогда, и `timeout` убивает программу через 3 секунды (код возврата 124). Причина: когда
> родитель делал fork для `wc`, он ещё держал `writer`, поэтому `wc` унаследовал копию конца для записи на fd 4, а без
> `FD_CLOEXEC` эта копия переживает `exec`. Теперь сам `wc` держит конец для записи конвейера, из которого читает, и конец
> файла никогда не наступит.

## 4. Перенаправление: `>`, `<` и `2>&1`

Перенаправление в файл — тот же приём, только с файлом вместо конвейера: открыть файл в ребёнке, сделать `dup2` на 0, 1 или 2,
затем `exec`. Команда оболочки `ls -l /etc/hostname /nope > /tmp/out.txt 2>&1` превращается в `src/bin/redirect.rs`:

```rust
use nix::fcntl::{OFlag, open};
use nix::sys::stat::Mode;
use nix::sys::wait::waitpid;
use nix::unistd::{ForkResult, dup2_stderr, dup2_stdout, execvp, fork};
use std::ffi::CString;

fn main() -> nix::Result<()> {
    // The shell command:  ls -l /etc/hostname /nope > /tmp/out.txt 2>&1
    let args: Vec<CString> = ["ls", "-l", "/etc/hostname", "/nope"]
        .iter()
        .map(|a| CString::new(*a).unwrap())
        .collect();

    // SAFETY: single-threaded program.
    match unsafe { fork() }? {
        ForkResult::Child => {
            let flags = OFlag::O_WRONLY | OFlag::O_CREAT | OFlag::O_TRUNC | OFlag::O_CLOEXEC;
            let file = open("/tmp/out.txt", flags, Mode::from_bits_truncate(0o644))?;
            dup2_stdout(&file)?; // > /tmp/out.txt : fd 1 -> the file
            dup2_stderr(&file)?; // 2>&1           : fd 2 -> the same open file description
            let err = execvp(&args[0], &args).unwrap_err();
            eprintln!("exec failed: {err}");
            std::process::exit(127);
        }
        ForkResult::Parent { child } => {
            println!("child status: {:?}", waitpid(child, None)?);
            print!("{}", std::fs::read_to_string("/tmp/out.txt").unwrap());
        }
    }
    Ok(())
}
```

```text
child status: Exited(Pid(245), 2)
ls: cannot access '/nope': No such file or directory
-rw-r--r-- 1 root root 13 Nov 18 07:35 /etc/hostname
```

**Что происходит:**

1. Ребёнок открывает файл **после** `fork`, поэтому собственные дескрипторы родителя не трогаются. Флаги — ровно те, что
   оболочка использует для `>`: только запись, создать, обрезать.
2. `dup2_stdout(&file)` направляет fd 1 на файл; `dup2_stderr(&file)` направляет fd 2 на **то же описание открытого файла**,
   поэтому у обоих потоков одно смещение (Урок 13.4), и их строки идут друг за другом.
3. `ls` пишет ошибку в fd 2, а список — в fd 1; оба попадают в `/tmp/out.txt`. `ls` завершается с кодом 2, потому что одного
   пути не существовало.

Остальные перенаправления устроены по той же схеме:

| Оболочка | В ребёнке, до `exec` |
|---|---|
| `> file` | `open(file, O_WRONLY \| O_CREAT \| O_TRUNC)`, затем `dup2` на 1 |
| `>> file` | `open(file, O_WRONLY \| O_CREAT \| O_APPEND)`, затем `dup2` на 1 |
| `< file` | `open(file, O_RDONLY)`, затем `dup2` на 0 |
| `2>&1` | `dup2(1, 2)`: fd 2 становится копией того, чем fd 1 является **сейчас** |
| `2> /dev/null` | `open("/dev/null", O_WRONLY)`, затем `dup2` на 2 |

> 💡 **Совет:** для `2>&1` важен порядок, потому что он копирует fd 1 **в этот момент**. `cmd > file 2>&1` отправляет оба
> потока в файл. `cmd 2>&1 > file` сначала копирует fd 1 (терминал) в fd 2, а потом направляет fd 1 на файл: ошибки
> по-прежнему идут в терминал.

## 5. Когда читающий уходит: `SIGPIPE` и `EPIPE`

Что будет, если программа пишет в конвейер, а читающий уже завершился? Попробуйте с бесконечной программой на Rust
(`src/bin/endless.rs`), вывод которой передан в `head`:

```rust
fn main() {
    for i in 1..=u64::MAX {
        println!("line {i}");
    }
}
```

```bash
./target/debug/endless | head -2
echo "status: ${PIPESTATUS[0]}"
```

```text
line 1
line 2

thread 'main' (248) panicked at /rustc/0ef1bc386059596dada8fb65b50425aab2ea637a/library/std/src/io/stdio.rs:1209:9:
failed printing to stdout: Broken pipe (os error 32)
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
status: 101
```

**Что происходит:**

1. `head -2` печатает две строки и завершается. Его конец для чтения закрыт; читающих не осталось.
2. Следующий `write` нашей программы попадает под правило «все концы для чтения закрыты». Ядро посылает процессу сигнал
   `SIGPIPE`, а `write` завершается ошибкой `EPIPE` («broken pipe», ошибка 32).
3. Действие `SIGPIPE` по умолчанию — молча убить процесс. Так происходит с программами на C вроде `cat`. Но рантайм Rust ещё
   до `main` ставит для `SIGPIPE` «игнорировать» (вы видели строку `rt_sigaction` в Уроке 13.3), поэтому программа выживает и
   вместо сигнала получает ошибку `EPIPE`.
4. `println!` паникует, когда запись не удаётся, поэтому программа умирает с сообщением о панике и кодом возврата 101.
   `${PIPESTATUS[0]}` — переменная bash с кодом возврата первой команды последнего конвейера.

Решение для утилиты командной строки: писать через `writeln!`, который возвращает `Result`, и считать `BrokenPipe` нормальным
завершением (`src/bin/endless-quiet.rs`):

```rust
use std::io::{ErrorKind, Write};

fn main() {
    let mut out = std::io::stdout().lock();
    for i in 1.. {
        if let Err(e) = writeln!(out, "line {i}") {
            if e.kind() == ErrorKind::BrokenPipe {
                return; // the reader has gone away: stop quietly, like `yes` or `cat`
            }
            eprintln!("write error: {e}");
            std::process::exit(1);
        }
    }
}
```

```text
line 1
line 2
status: 0
```

> 🔒 **Безопасность:** игнорировать `SIGPIPE` — более безопасный выбор по умолчанию для серверов: сетевой клиент, который
> отключился, не должен убивать сигналом весь сервер. Урок 13.7 покажет, как самому менять поведение сигналов.

## 6. Способ стандартной библиотеки: `Stdio`

`std::process::Command` делает всё это за вас. `Stdio::piped()` создаёт конвейер, а `Stdio::from(...)` передаёт один его конец
следующей программе:

```rust
use std::process::{Command, Stdio};

fn main() -> std::io::Result<()> {
    // ls /etc | wc -l, the standard library way
    let mut ls = Command::new("ls")
        .arg("/etc")
        .stdout(Stdio::piped())
        .spawn()?;
    let ls_out = ls.stdout.take().expect("stdout was piped");
    let wc = Command::new("wc")
        .arg("-l")
        .stdin(Stdio::from(ls_out))
        .output()?;
    ls.wait()?;
    print!("wc says: {}", String::from_utf8_lossy(&wc.stdout));
    Ok(())
}
```

```text
wc says: 94
```

`spawn` запускает `ls`, не дожидаясь его; `take()` забирает конвейер его stdout из `Child`, поэтому в нашем процессе он
закрывается, как только его получил `wc`; `output()` запускает `wc`, собирает его вывод через ещё один конвейер и ждёт. `std`
создаёт каждый конвейер с `O_CLOEXEC`, поэтому зависание из раздела 3 невозможно. Сохраните этот файл как
`src/bin/std-pipeline.rs`.

## Частые ошибки

> ⚠️ **Частая ошибка:** ждать первую команду, прежде чем начать читать. Если родитель вызывает `waitpid` для ребёнка, который
> пишет в конвейер больше 64 КиБ, и только потом читает, оба ждут вечно: ребёнок — места в буфере, родитель — завершения
> ребёнка. Сначала читайте (или читайте в другом потоке), потом ждите.

## Попробуйте сами

1. В `pipe-fork` удалите `drop(writer);` родителя и посмотрите, как программа зависает. Нажмите Ctrl-C. Верните строку.
2. Выполните `ls -l /proc/self/fd | cat`. Какой дескриптор `ls` — конвейер? (`1 -> pipe:[...]`.)
3. Измените `redirect.rs`, чтобы он реализовал `>>` (дозапись), и запустите дважды. В файле должны оказаться оба вывода.
4. Выполните `./target/debug/endless-quiet | head -3` и `yes | head -3`. Обе программы останавливаются тихо.

## Упражнения

### Упражнение 1 — Какого размера конвейер? (легко)

Создайте конвейер через `pipe2(OFlag::O_NONBLOCK)` и пишите в него кусками по 1 КиБ, не читая, пока `write` не вернёт
`Errno::EAGAIN`. Напечатайте, сколько байтов поместилось.

<details>
<summary>Подсказка</summary>

С `O_NONBLOCK` `write` в полный конвейер не засыпает, а возвращает `Err(EAGAIN)`. Держите конец для чтения живым
(`let (_reader, writer) = ...`), иначе получите `EPIPE`.

</details>

<details>
<summary>Ответ</summary>

```rust
use nix::errno::Errno;
use nix::fcntl::OFlag;
use nix::unistd::{pipe2, write};

fn main() -> nix::Result<()> {
    // O_NONBLOCK: when the pipe is full, write fails with EAGAIN instead of waiting.
    let (_reader, writer) = pipe2(OFlag::O_NONBLOCK)?;
    let chunk = [b'x'; 1024];
    let mut total = 0;
    loop {
        match write(&writer, &chunk) {
            Ok(n) => total += n,
            Err(Errno::EAGAIN) => break,
            Err(e) => return Err(e),
        }
    }
    println!(
        "the pipe was full after {total} bytes ({} KiB)",
        total / 1024
    );
    Ok(())
}
```

```text
the pipe was full after 65536 bytes (64 KiB)
```

Почему это работает: буфер конвейера в Linux по умолчанию равен 64 КиБ. Программа может увеличить его через
`fcntl(F_SETPIPE_SZ)`, до предела из `/proc/sys/fs/pipe-max-size` (по умолчанию 1 МиБ). К неблокирующему вводу-выводу вы
вернётесь в Уроке 13.13.

</details>

### Упражнение 2 — `capture`, безопасный для тестов (средне)

Напишите `fn capture(argv: &[&str]) -> nix::Result<(String, i32)>`, которая запускает программу с подключённым к конвейеру
стандартным выводом и возвращает всё, что та напечатала, вместе с кодом возврата (маленький `Command::output`). Используйте
аккуратного ребёнка из упражнения 3 Урока 13.5: всё подготовьте до `fork`, а в ребёнке вызывайте только `libc::dup2`,
`libc::execvp` и `libc::_exit`. Напишите три теста: `echo hi there` даёт `("hi there\n", 0)`, `false` даёт `("", 1)`, а
`seq 1 100000` (больше 64 КиБ вывода) даёт 100 000 строк.

<details>
<summary>Подсказка</summary>

В родителе: закройте конец для записи, прочитайте конец для чтения до конца (`File::from(reader).read_to_string`), **потом**
вызовите `waitpid`. Именно чтение первым позволяет пройти тесту на 100 000 строк. Используйте `pipe2(OFlag::O_CLOEXEC)`:
тесты идут в потоках, и дети других тестов иначе могли бы унаследовать ваш конец для записи и держать конвейер открытым.

</details>

<details>
<summary>Ответ</summary>

```rust
use nix::fcntl::OFlag;
use nix::sys::wait::{WaitStatus, waitpid};
use nix::unistd::{ForkResult, fork, pipe2};
use std::ffi::{CString, c_char};
use std::io::Read;
use std::os::fd::AsRawFd;

/// Runs `argv` and returns everything it wrote to standard output, plus its exit code.
fn capture(argv: &[&str]) -> nix::Result<(String, i32)> {
    let c_args: Vec<CString> = argv.iter().map(|a| CString::new(*a).unwrap()).collect();
    let mut ptrs: Vec<*const c_char> = c_args.iter().map(|a| a.as_ptr()).collect();
    ptrs.push(std::ptr::null());
    let (reader, writer) = pipe2(OFlag::O_CLOEXEC)?;
    let write_fd = writer.as_raw_fd();

    // SAFETY: the child only calls dup2, execvp and _exit, which are async-signal-safe.
    match unsafe { fork() }? {
        ForkResult::Child => unsafe {
            libc::dup2(write_fd, 1);
            libc::execvp(ptrs[0], ptrs.as_ptr());
            libc::_exit(127);
        },
        ForkResult::Parent { child } => {
            drop(writer); // keep only the child's copy of the write end
            let mut output = String::new();
            std::fs::File::from(reader)
                .read_to_string(&mut output)
                .map_err(|_| nix::errno::Errno::EIO)?;
            let code = match waitpid(child, None)? {
                WaitStatus::Exited(_, code) => code,
                _ => -1,
            };
            Ok((output, code))
        }
    }
}

fn main() -> nix::Result<()> {
    let (out, code) = capture(&["uname", "-s"])?;
    println!("uname printed {out:?} and exited with {code}");
    Ok(())
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn captures_standard_output() {
        assert_eq!(
            capture(&["echo", "hi", "there"]).unwrap(),
            ("hi there\n".to_string(), 0)
        );
    }

    #[test]
    fn reports_the_exit_code_and_empty_output() {
        assert_eq!(capture(&["false"]).unwrap(), (String::new(), 1));
    }

    #[test]
    fn output_larger_than_the_pipe_buffer_arrives_completely() {
        let (out, code) = capture(&["seq", "1", "100000"]).unwrap();
        assert_eq!(code, 0);
        assert_eq!(out.lines().count(), 100_000);
    }
}
```

```text
uname printed "Linux\n" and exited with 0
test result: ok. 3 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.01s
```

Почему это работает: fd 1 ребёнка — копия конца для записи, а его исходный конец для записи закрывается при `exec`. Родитель
закрыл свою копию, поэтому, когда ребёнок завершается, концов для записи не остаётся, и `read_to_string` заканчивается.
Поскольку родитель читает, пока ребёнок пишет, буфер в 64 КиБ надолго ребёнка не блокирует.

</details>

### Упражнение 3 — Читаем конвейер самой оболочки (сложно)

Выполните `strace -f -o /tmp/t -e trace=pipe2,clone,dup3,close,execve bash -c 'ls /etc | wc -l'` и прочитайте `/tmp/t`
(`-o` записывает трассировку в файл; каждая строка начинается с PID). Ответьте: использует ли bash `O_CLOEXEC` для своего
конвейера? Как он добивается того, чтобы `wc` увидел конец файла? В каком процессе и в каком порядке закрываются 3 и 4?

<details>
<summary>Подсказка</summary>

Посмотрите на флаги `pipe2`, затем проследите за каждым PID отдельно: `grep -E '^[0-9]+ +(pipe2|clone|dup3|close\((3|4)\)|execve)' /tmp/t`.
Строки `close(3)` в начале, связанные с загрузкой библиотек, относятся к самому bash; не обращайте на них внимания.

</details>

<details>
<summary>Ответ</summary>

Интересные строки, отсортированные по процессам и сокращённые:

```text
10    pipe2([3, 4], 0)                  = 0
10    clone(...)                        = 11
10    close(4)                          = 0
10    clone(...)                        = 12
10    close(3)                          = 0
11    close(3)                          = 0
11    dup3(4, 1, 0)                     = 1
11    close(4)                          = 0
11    execve("/usr/bin/ls", ["ls", "/etc"], ...) = 0
12    dup3(3, 0, 0)                     = 0
12    close(3)                          = 0
12    execve("/usr/bin/wc", ["wc", "-l"], ...) = 0
```

Bash (PID 10) создаёт конвейер **без** `O_CLOEXEC` (флаги `0`) и вместо этого закрывает каждый лишний дескриптор вручную:

1. Сразу после fork левого ребёнка (11) bash закрывает свой конец для записи 4. Поэтому, когда он делает fork правого ребёнка
   (12), наследовать уже нечего — зависание из раздела 3 невозможно.
2. После fork правого ребёнка bash закрывает конец для чтения 3. Сам bash больше не держит ни одного конца конвейера.
3. Левый ребёнок закрывает ненужный ему конец для чтения, делает fd 1 копией 4, закрывает 4 и становится `ls`.
4. Правый ребёнок делает fd 0 копией 3, закрывает 3 и становится `wc`.

Работают оба подхода: закрывать вручную в правильном порядке или позволить `O_CLOEXEC` закрыть исходные дескрипторы при `exec`.
Командная оболочка из Урока 13.8 должна справляться с конвейерами любой длины, поэтому выберите один подход и применяйте его к
каждому конвейеру.

</details>

## Вопросы для самопроверки

1. Конвейер пуст. Когда `read` блокируется, а когда возвращает 0?

<details>
<summary>Ответ</summary>

Блокируется, пока хотя бы один конец для записи открыт в каком-либо процессе. Возвращает 0 (конец файла), когда все концы для
записи закрыты.

</details>

2. Что делает `dup2(4, 1)`?

<details>
<summary>Ответ</summary>

Делает так, что дескриптор 1 указывает на то же описание открытого файла, что и дескриптор 4, закрыв то, на что fd 1 указывал
раньше. После этого всё, что программа пишет в стандартный вывод, идёт туда же, куда fd 4.

</details>

3. Почему оболочка должна вызывать `dup2` в ребёнке, а не в себе?

<details>
<summary>Ответ</summary>

Потому что `dup2` меняет таблицу дескрипторов вызвавшего процесса. В ребёнке он готовит дескрипторы только для новой программы;
в оболочке он перенаправил бы собственный ввод или вывод оболочки.

</details>

4. Программа читает из конвейера и зависает после того, как пришли все данные. Какова самая вероятная причина?

<details>
<summary>Ответ</summary>

Какой-то процесс всё ещё держит конец для записи конвейера — часто сам читающий или «брат», унаследовавший его без
`FD_CLOEXEC`. Пока открыт хоть один конец для записи, конец файла наступить не может.

</details>

5. Почему программа на Rust получает `EPIPE`, а не погибает от `SIGPIPE`, как программа на C?

<details>
<summary>Ответ</summary>

Рантайм Rust ставит для `SIGPIPE` «игнорировать» ещё до запуска `main`. Игнорируемый `SIGPIPE` не убивает процесс, поэтому
`write` просто возвращает ошибку `EPIPE`.

</details>

## Итоги

- Конвейер — буфер ядра (64 КиБ) с концом для чтения и концом для записи; `read` блокируется, когда он пуст, `write` — когда он полон.
- Конец файла наступает, только когда закрыты **все** концы для записи во всех процессах; один забытый дескриптор подвешивает читающего.
- После `fork` оба процесса держат оба конца; каждый закрывает тот конец, которым не пользуется.
- `dup2` направляет 0, 1 или 2 на конвейер или файл; `dup2` не копирует `FD_CLOEXEC`, поэтому создавайте конвейеры с `O_CLOEXEC`, и исходные дескрипторы закроются при `exec`.
- Перенаправление `>`, `>>`, `<`, `2>&1` — это `open` плюс `dup2` в ребёнке до `exec`.
- Запись в конвейер без читающих даёт `SIGPIPE` и `EPIPE`; Rust игнорирует `SIGPIPE`, поэтому в утилитах командной строки обрабатывайте `ErrorKind::BrokenPipe`.
- `Stdio::piped()` и `Stdio::from` безопасно делают то же самое в `std`.

## Источники и что почитать

- [pipe(7) — Linux manual page](https://man7.org/linux/man-pages/man7/pipe.7.html) — ёмкость, блокировка, `SIGPIPE` и атомарная запись.
- [pipe(2) — Linux manual page](https://man7.org/linux/man-pages/man2/pipe.2.html) — `pipe` и `pipe2`.
- [dup(2) — Linux manual page](https://man7.org/linux/man-pages/man2/dup.2.html) — `dup2` и `dup3`.
- [GNU Bash Reference Manual — Redirections](https://www.gnu.org/software/bash/manual/html_node/Redirections.html) — все перенаправления, которые знает оболочка.
- [std::process::Stdio — Rust documentation](https://doc.rust-lang.org/std/process/struct.Stdio.html) — конвейеры в стандартной библиотеке.

---

[🏠 Начало](../START-HERE.md) · [← 13.5 Создание процессов: fork, exec и wait](05-fork-exec-and-wait.md) · [📋 Этап 13](README.md) · [13.7 Сигналы в ваших программах →](07-signals-in-your-programs.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
