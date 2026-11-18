<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 13.13 Неблокирующий ввод-вывод и epoll](13-non-blocking-io-and-epoll.md) · [📋 Этап 13](README.md) · [13.15 Пространства имён (namespaces) →](15-namespaces.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/13-operating-systems/14-io-uring.md) <!-- nav -->

# Урок 13.14 — io_uring

> **Этап 13 — Операционные системы изнутри** · Урок 14 из 18 · ⏱ около 120 минут

## Что вы узнаете

- Чем модель *готовности* epoll отличается от модели *завершения* io_uring.
- Как устроен io_uring: два кольцевых буфера в памяти, общей для вашей программы и ядра.
- Как отправлять чтения и записи с помощью крейта `io-uring` и почему добавление операции — это `unsafe`.
- Как много операций одновременно находятся в работе и завершаются в любом порядке.
- Где io_uring доступен, почему его иногда отключают и как он вписывается в асинхронный Rust.

## Перед началом

- Вы прошли [Урок 13.13 — Неблокирующий ввод-вывод и epoll](13-non-blocking-io-and-epoll.md).
- Вы знаете общие отображения ([Урок 13.11](11-memory-mapped-files.md)), сырые указатели и комментарии `// SAFETY:` ([Урок 06.6](../06-rust-advanced/06-unsafe-rust-and-miri.md)).
- Создайте пакет на Mac и скопируйте его в ВМ, как в Уроке 13.10:

```bash
cd ~/projects/rust
cargo new --lib code/p13-uring
cd code/p13-uring
cargo add io-uring@0.7
mkdir src/bin
```

- Проверьте в ВМ, что io_uring разрешён:

```bash
sysctl kernel.io_uring_disabled
```

Пример вывода (у вас будет похоже):

```text
kernel.io_uring_disabled = 0
```

`0` означает «разрешено всем». Если там `2`, io_uring на этой машине выключен (раздел 5).

## Новые слова

| Слово | Значение |
|---|---|
| **io_uring** | Интерфейс Linux (с ядра 5.1, 2019 год) для асинхронного ввода-вывода (I/O, input/output): вы ставите операции в очередь в общей памяти, а ядро сообщает, когда каждая выполнена. Название означает «I/O user ring», кольцо ввода-вывода пользователя. |
| **модель завершения** (completion model) | Вы просите ядро *выполнить* операцию; позже оно сообщает, что операция *завершена*, и даёт результат. |
| **модель готовности** (readiness model) | Ядро сообщает, когда операцию *можно* выполнить без ожидания; выполняете её вы сами (epoll, Урок 13.13). |
| **кольцевой буфер** (ring buffer) | Массив фиксированного размера, используемый как очередь, конец которой переходит в начало. Одна сторона пишет в *хвост*, другая читает из *головы*. |
| **SQ** (Submission Queue, очередь отправки) | Кольцо, куда ваша программа кладёт операции для ядра. |
| **SQE** (Submission Queue Entry, элемент очереди отправки) | Одна операция в SQ: что сделать (чтение, запись, открытие…), с каким дескриптором, с каким буфером. |
| **CQ** (Completion Queue, очередь завершения) | Кольцо, куда ядро кладёт результаты. |
| **CQE** (Completion Queue Entry, элемент очереди завершения) | Один результат: ваше число `user_data` и `result` (байты или отрицательный номер ошибки). |
| **`user_data`** | 64-битное число, которое вы прикрепляете к SQE; ядро копирует его в соответствующий CQE, и вы знаете, какая операция завершилась. |
| **в работе** (in flight) | Отправлено, но ещё не завершено. |
| **seccomp** (secure computing mode, режим безопасных вычислений) | Возможность Linux, которая позволяет процессу (или среде выполнения контейнеров) запретить определённые системные вызовы. |

## 1. От «готово» к «выполнено»

С epoll (Урок 13.13) разговор с ядром выглядит так:

```text
program: "tell me when fd 5 has data"       (epoll_ctl, once)
program: "anything ready?"                   (epoll_wait)
kernel:  "fd 5 is ready"
program: "read fd 5 into this buffer"        (read)  <- one more system call per operation
kernel:  "here are 100 bytes"
```

Видны две слабости. Каждой операции по-прежнему нужен свой системный вызов, а каждый системный вызов тратит
время на вход в ядро и выход из него. И обычные файлы не работают вовсе, потому что файл на диске всегда
«готов», даже если его чтение означает ожидание диска.

io_uring меняет разговор:

```text
program: "read fd 5 into this buffer, read fd 7 into that buffer, write fd 9 ..."  (queued in memory)
program: "go" (one system call for the whole batch, or even none)
kernel:  ... does the work, maybe in parallel ...
kernel:  "operation 1 done: 100 bytes; operation 3 done: 4096 bytes; ..."         (in memory)
```

Вы описываете, *что* вам нужно; ядро это делает и сообщает, когда работа *выполнена*. Это и есть **модель
завершения**. Она одинаково работает для файлов, сокетов и конвейеров.

## 2. Два кольца в общей памяти

io_uring построен вокруг двух **кольцевых буферов**, которые живут в памяти, отображённой и в ядро, и в ваш
процесс (общие отображения, Урок 13.11). Их никому не нужно копировать, а чтобы заглянуть в них, часто не
нужен даже системный вызов.

```text
          your program                               kernel
   +------------------------------------------------------------------+
   | SQ (submission queue)                                            |
   |   tail -> you write SQEs here      head -> kernel takes them here|
   |   [ read fd4 buf0 off0 ud=0 ][ read fd4 buf1 off4096 ud=1 ][ ]   |
   +------------------------------------------------------------------+
   | CQ (completion queue)                                            |
   |   head -> you take CQEs here       tail -> kernel writes here    |
   |   [ ud=1 res=4096 ][ ud=0 res=4096 ][ ]                          |
   +------------------------------------------------------------------+
        system call io_uring_enter: "I added N entries" / "wait for M"
```

1. Ваша программа пишет **SQE** в хвост **SQ** и сдвигает хвост.
2. Она вызывает `io_uring_enter`, чтобы сказать ядру «есть новые элементы» (и, если хочет, «подожди, пока
   завершатся M»).
3. Ядро берёт элементы из головы, выполняет их и для каждого пишет **CQE** в хвост **CQ**.
4. Ваша программа читает CQE из головы CQ. `user_data` в каждом CQE говорит, к какому SQE он относится.

Кольцевой буфер — это просто массив плюс два счётчика: пишущий увеличивает хвост, читающий увеличивает
голову, а позиции берутся по модулю размера. Поскольку каждый счётчик одна сторона только пишет, а другая
только читает, блокировка не нужна.

Крейт `io-uring` всё это оборачивает: `IoUring::new` создаёт кольца, `submission()` даёт вам SQ,
`completion()` — CQ, а `opcode::*` строит SQE.

## 3. Первое чтение

Создайте `src/bin/first.rs`:

```rust
use std::fs::File;
use std::os::fd::AsRawFd;

use io_uring::{IoUring, opcode, types};

fn main() -> std::io::Result<()> {
    let mut ring = IoUring::new(8)?; // room for 8 submissions
    let file = File::open("/etc/os-release")?;
    let mut buf = vec![0u8; 64];

    // Describe the operation: "read up to 64 bytes from this fd into buf, at offset 0".
    let read = opcode::Read::new(
        types::Fd(file.as_raw_fd()),
        buf.as_mut_ptr(),
        buf.len() as u32,
    )
    .offset(0)
    .build()
    .user_data(42);

    // SAFETY: `buf` and `file` stay alive and are not moved until the completion arrives.
    unsafe {
        ring.submission()
            .push(&read)
            .expect("submission queue is full");
    }
    ring.submit_and_wait(1)?; // one system call: hand over the work and wait for 1 completion

    let cqe = ring.completion().next().expect("no completion");
    println!("user_data = {}, result = {}", cqe.user_data(), cqe.result());
    let n = cqe.result() as usize;
    println!("{:?}", String::from_utf8_lossy(&buf[..n]));
    Ok(())
}
```

**Что происходит:**

1. `IoUring::new(8)` вызывает `io_uring_setup` и отображает кольца. В SQ 8 элементов (CQ по умолчанию
   получает вдвое больше).
2. `opcode::Read::new(fd, pointer, length)` строит описание чтения. Он принимает **сырой указатель** на буфер,
   а не `&mut [u8]`: ядро будет писать туда позже, когда эта строка уже завершится. `.offset(0)` говорит, с
   какого места файла читать; `.user_data(42)` прикрепляет нашу метку.
3. `push` — это `unsafe`, потому что компилятор не может проверить, что ядро сделает с этим указателем. Мы
   обещаем, что `buf` (и дескриптор) живут и не перемещаются и не используются повторно, пока не придёт
   завершение. Если бы `buf` уничтожили слишком рано, ядро писало бы в освобождённую память: неопределённое
   поведение.
4. `submit_and_wait(1)` один раз вызывает `io_uring_enter`: «вот новая работа, вернись, когда завершится хотя
   бы 1 операция».
5. `completion().next()` берёт CQE. `result()` — число прочитанных байтов, как возвращаемое значение `read`;
   отрицательное число означало бы ошибку (упражнение 1).

В ВМ, `cargo run --bin first`:

```text
user_data = 42, result = 64
"PRETTY_NAME=\"Debian GNU/Linux 13 (trixie)\"\nNAME=\"Debian GNU/Linu"
```

(В вашей ВМ Ubuntu, поэтому текст будет другим.) `strace` показывает системные вызовы. Ниже строки запуска
вырезаны, а длинная строка `io_uring_setup` сокращена через `...`:

```bash
strace -e trace=io_uring_setup,io_uring_enter,mmap,openat ./target/debug/first
```

```text
io_uring_setup(8, {flags=0, sq_thread_cpu=0, sq_thread_idle=0, sq_entries=8, cq_entries=16, ...}) = 3
mmap(NULL, 512, PROT_READ|PROT_WRITE, MAP_SHARED|MAP_POPULATE, 3, 0x10000000) = 0xffff9e1d2000
mmap(NULL, 352, PROT_READ|PROT_WRITE, MAP_SHARED|MAP_POPULATE, 3, 0) = 0xffff9e1d1000
openat(AT_FDCWD, "/etc/os-release", O_RDONLY|O_CLOEXEC) = 4
io_uring_enter(3, 1, 1, IORING_ENTER_GETEVENTS, NULL, 128) = 1
user_data = 42, result = 64
```

**Что происходит** в трассировке:

1. `io_uring_setup` создаёт кольцо и возвращает дескриптор (3). Ядро отвечает настоящими размерами:
   `sq_entries=8, cq_entries=16`.
2. Два вызова `mmap` с `MAP_SHARED` для дескриптора 3 отображают кольца в наше адресное пространство: тот, что
   со смещением 0, содержит оба кольца (SQ и CQ), а тот, что со смещением `0x10000000`, — массив SQE. Это то же
   общее отображение, что в Уроке 13.11, только за ним стоит память ядра, а не файл.
3. `io_uring_enter(3, 1, 1, IORING_ENTER_GETEVENTS, …)`: «1 новый элемент, жди 1 завершения». Системного
   вызова `read` нет: чтение ядро выполнило само.

## 4. Много операций в работе

Настоящая сила проявляется на многих операциях. Создайте `src/bin/many.rs`, который читает 16 блоков файла
**одним** системным вызовом:

```rust
use std::fs::{self, File};
use std::os::fd::AsRawFd;

use io_uring::{IoUring, opcode, types};

const BLOCK: usize = 4096;
const COUNT: usize = 16;

fn main() -> std::io::Result<()> {
    // A test file: block 0 is full of 'A', block 1 of 'B', ... block 15 of 'P'.
    let data: Vec<u8> = (0..COUNT)
        .flat_map(|i| vec![b'A' + i as u8; BLOCK])
        .collect();
    fs::write("blocks.bin", &data)?;

    let file = File::open("blocks.bin")?;
    let mut ring = IoUring::new(COUNT as u32)?;
    let mut buffers = vec![vec![0u8; BLOCK]; COUNT];

    for (i, buf) in buffers.iter_mut().enumerate() {
        let read = opcode::Read::new(types::Fd(file.as_raw_fd()), buf.as_mut_ptr(), BLOCK as u32)
            .offset((i * BLOCK) as u64)
            .build()
            .user_data(i as u64); // remember which block this is
        // SAFETY: `buffers` and `file` live until all completions have been collected below.
        unsafe { ring.submission().push(&read).expect("queue full") };
    }
    let submitted = ring.submit_and_wait(COUNT)?; // ONE system call for all 16 reads
    println!("submitted {submitted} reads with one io_uring_enter");

    let mut order = Vec::new();
    for cqe in ring.completion() {
        assert_eq!(cqe.result(), BLOCK as i32, "a read failed or was short");
        order.push(cqe.user_data());
    }
    println!("completion order: {order:?}");
    let firsts: String = buffers.iter().map(|b| b[0] as char).collect();
    println!("first byte of each block: {firsts}");
    Ok(())
}
```

**Что происходит:**

1. Цикл кладёт 16 SQE на чтение, у каждого свой буфер, смещение и `user_data` (номер блока). Ядру пока ничего
   не отправлено: мы только записали в общую память.
2. `submit_and_wait(COUNT)` передаёт все 16 одним `io_uring_enter` и ждёт, пока 16 завершатся.
3. `ring.completion()` — итератор по всем готовым CQE. Мы собираем их `user_data`, чтобы увидеть порядок.
4. Каждый буфер получил свой блок: первые байты составляют буквы от `A` до `P`.

```text
submitted 16 reads with one io_uring_enter
completion order: [0, 2, 3, 1, 6, 4, 7, 5, 9, 8, 10, 11, 13, 12, 14, 15]
first byte of each block: ABCDEFGHIJKLMNOP
```

Завершения пришли **не по порядку**, и порядок меняется от запуска к запуску. Ядро работало над несколькими
чтениями одновременно. Именно для этого нужен `user_data`: никогда не считайте, что завершение номер *n*
относится к отправке номер *n*.

Посчитайте системные вызовы через `strace -c` (сводная таблица вместо списка):

```bash
strace -c -e trace=io_uring_enter,read,pread64 ./target/debug/many
```

```text
% time     seconds  usecs/call     calls    errors syscall
------ ----------- ----------- --------- --------- ----------------
 89.05    0.000187         187         1           io_uring_enter
 10.95    0.000023           4         5           read
------ ----------- ----------- --------- --------- ----------------
100.00    0.000210          35         6           total
```

Пять вызовов `read` делает загрузчик, читая библиотеки программы при запуске. Все 16 чтений файла прошли через
один `io_uring_enter`. С `pread` (`read` по смещению) это было бы 16 системных вызовов.

> ⚡ **Производительность:** меньшее число системных вызовов важно, когда программа выполняет сотни тысяч
> мелких операций в секунду, как базы данных и прокси. Для программы, которая читает несколько файлов, обычный
> `read` проще и так же быстр. Как всегда, сначала измеряйте.

## 5. Доступность и безопасность

io_uring мощный, и у этого есть цена. Это большая новая часть ядра, которая умеет почти всё, что умеют
обычные системные вызовы, и с 2019 года в ней нашли много серьёзных ошибок безопасности. Поэтому:

- Некоторые системы его отключают. Настройка `kernel.io_uring_disabled` равна `0` (разрешено), `1` (только для root и
  членов одной группы, заданной в `kernel.io_uring_group`) или `2` (запрещено всем).
- **Профиль seccomp Docker по умолчанию его блокирует.** **seccomp** позволяет среде выполнения контейнеров
  запрещать системные вызовы, и `io_uring_setup` по умолчанию запрещён. В обычном контейнере `first` сразу
  завершается ошибкой:

  ```text
  Error: Os { code: 1, kind: PermissionDenied, message: "Operation not permitted" }
  ```

  Примеры этого урока проверялись в контейнере, запущенном с `--security-opt seccomp=unconfined`. В вашей ВМ
  они работают без всяких параметров.

> 🔒 **Безопасность:** относитесь к io_uring как к любой мощной возможности: используйте его там, где он даёт
> реальную пользу, обновляйте ядро и ожидайте, что в защищённых средах он будет отключён. Пишите код, который
> может вернуться к обычному вводу-выводу.

**io_uring и асинхронный Rust.** Tokio по умолчанию использует epoll, а не io_uring. Рантаймы вроде `tokio-uring` построены
на io_uring. Они показывают ещё один урок о владении: поскольку ядро пользуется буфером *после* вызова,
который его отправил, API на модели завершения не может просто заимствовать `&mut [u8]`, как это делает
`read`. Операция должна **владеть** буфером до своего завершения и потом вернуть его. Поэтому такие API
принимают `Vec<u8>` по значению и возвращают его вместе с результатом.

## Частые ошибки

1. **Ожидание, что завершения приходят в порядке отправки.** Это не так, как показал `many`.
   *Исправление:* всегда сопоставляйте CQE с его SQE через `user_data`.

2. **Буфер умирает слишком рано.** Если буфер — локальная переменная функции, которая возвращается до прихода
   завершения, ядро пишет в память, которая принадлежит чему-то другому. Компилятор не предупредит; именно
   об этом блок `unsafe`. *Исправление:* держите буферы там, где они живут дольше операции (как `buffers` в
   `many`), и соберите все завершения, прежде чем уничтожать их.

3. **Отрицательные результаты игнорируются.** `result()` равен `-2` для «нет такого файла», `-9` для «плохой
   файловый дескриптор» и так далее. Если считать это числом байтов (`as usize`), получится гигантское число.
   *Исправление:* сначала преобразуйте его (упражнение 1).

4. **Проверка в контейнере по умолчанию.** `Operation not permitted` от `IoUring::new` обычно означает seccomp
   или `kernel.io_uring_disabled`, а не ошибку в вашем коде.

## Попробуйте сами

1. Поменяйте `COUNT` в `many` на 64 (размер кольца следует за ним). Запустите несколько раз и посмотрите на
   порядок.
2. Сделайте файл на один блок короче: замените `(0..COUNT)` в строке `let data` на `(0..COUNT - 1)`, но
   по-прежнему отправляйте 16 чтений. Проверка падает с `left: 0` и `right: 4096`: чтение, которое начинается
   в конце файла, завершается с результатом 0, точно как `read` в конце файла.
3. Добавьте `std::thread::sleep(std::time::Duration::from_secs(10));` перед `Ok(())` в `first`, запустите его
   в фоне командой `./target/debug/first &` и выполните `ls -l /proc/$!/fd`. Дескриптор 3 показан как
   `anon_inode:[io_uring]`: само кольцо — это файловый дескриптор, как экземпляр epoll в Уроке 13.13.

## Упражнения

### Упражнение 1 — Превратить результат CQE в `io::Result` (легко)

Напишите `pub fn cqe_result(result: i32) -> io::Result<usize>` в `src/lib.rs`. Значение от нуля и выше — число
байтов. Отрицательное значение — номер ошибки (errno) со знаком минус, например `-2` для ENOENT («нет такого
файла»). Напишите тесты для `5`, `0` и `-2` (его `kind()` должен быть `NotFound`).

<details>
<summary>Подсказка</summary>

`io::Error::from_raw_os_error(code)` строит ошибку из положительного errno.

</details>

<details>
<summary>Ответ</summary>

```rust
use std::io;

/// Turns the `result` field of a completion into an `io::Result`:
/// a negative value is a negated error number (errno).
pub fn cqe_result(result: i32) -> io::Result<usize> {
    if result < 0 {
        Err(io::Error::from_raw_os_error(-result))
    } else {
        Ok(result as usize)
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn positive_result_is_a_byte_count() {
        assert_eq!(cqe_result(5).unwrap(), 5);
        assert_eq!(cqe_result(0).unwrap(), 0);
    }

    #[test]
    fn negative_result_is_an_errno() {
        let err = cqe_result(-2).unwrap_err(); // 2 = ENOENT
        assert_eq!(err.kind(), io::ErrorKind::NotFound);
    }
}
```

Почему это работает: обычные системные вызовы возвращают -1 и кладут номер ошибки в `errno`; у io_uring нет
`errno`, поэтому он кладёт номер ошибки со знаком минус прямо в `result`. `from_raw_os_error` даёт вам тот же
`io::Error`, который создала бы `std`.

</details>

### Упражнение 2 — Запись файла через io_uring (средне)

Напишите `src/bin/uwrite.rs`, который создаёт `out.txt`, записывает в него `hello from io_uring\n` через
`opcode::Write`, печатает число записанных байтов (используйте `cqe_result`), а затем печатает содержимое
файла, прочитанное обратно через `fs::read_to_string`.

<details>
<summary>Подсказка</summary>

`opcode::Write::new(fd, pointer, length)` принимает `*const u8`: `text.as_ptr()`. Всё остальное — как в
`first`.

</details>

<details>
<summary>Ответ</summary>

```rust
use std::fs::{self, File};
use std::os::fd::AsRawFd;

use io_uring::{IoUring, opcode, types};
use p13_uring::cqe_result;

fn main() -> std::io::Result<()> {
    let text = b"hello from io_uring\n";
    let file = File::create("out.txt")?;
    let mut ring = IoUring::new(2)?;

    let write = opcode::Write::new(
        types::Fd(file.as_raw_fd()),
        text.as_ptr(),
        text.len() as u32,
    )
    .offset(0)
    .build()
    .user_data(1);
    // SAFETY: `text` is a constant and `file` lives until the completion is read.
    unsafe { ring.submission().push(&write).expect("queue full") };
    ring.submit_and_wait(1)?;

    let cqe = ring.completion().next().expect("no completion");
    let written = cqe_result(cqe.result())?;
    println!("wrote {written} bytes");
    print!("file now contains: {}", fs::read_to_string("out.txt")?);
    Ok(())
}
```

```text
wrote 20 bytes
file now contains: hello from io_uring
```

Почему это работает: запись описывается так же, как чтение, — указателем на данные. `text` — байтовый
строковый литерал, который живёт всю программу, поэтому обещание безопасности легко сдержать.

</details>

### Упражнение 3 — Читать блоки в запрошенном порядке (сложно)

Напишите `pub fn read_blocks(file: &File, offsets: &[u64], len: usize) -> io::Result<Vec<Vec<u8>>>`. Функция
отправляет по одному чтению `len` байт на каждое смещение, все сразу, и возвращает буферы **в порядке
`offsets`**, хотя завершения приходят в любом порядке. Чтение в конце файла может оказаться коротким;
обрежьте такой буфер до настоящей длины. Проверьте её на файле с `aaaabbbbccccdd`, смещениях `[8, 0, 12]` и
длине 4: результат — `cccc`, `aaaa`, `dd`.

<details>
<summary>Подсказка</summary>

Используйте индекс смещения как `user_data` и `buffers[cqe.user_data() as usize].truncate(n)`. В кольце должно
хватить места на все чтения и хотя бы на один элемент; ядро работает со степенями двойки, поэтому просите
`offsets.len().max(1).next_power_of_two()`.

</details>

<details>
<summary>Ответ</summary>

Добавьте в `src/lib.rs` (рядом с `cqe_result`):

```rust
use std::fs::File;
use std::os::fd::AsRawFd;

use io_uring::{IoUring, opcode, types};

/// Reads `len` bytes at each offset, all submitted at once.
/// The results come back in the order of `offsets`, whatever order the kernel finishes in.
pub fn read_blocks(file: &File, offsets: &[u64], len: usize) -> io::Result<Vec<Vec<u8>>> {
    let mut ring = IoUring::new(offsets.len().max(1).next_power_of_two() as u32)?;
    let mut buffers = vec![vec![0u8; len]; offsets.len()];
    for (i, (buf, &offset)) in buffers.iter_mut().zip(offsets).enumerate() {
        let read = opcode::Read::new(types::Fd(file.as_raw_fd()), buf.as_mut_ptr(), len as u32)
            .offset(offset)
            .build()
            .user_data(i as u64);
        // SAFETY: `buffers` is not touched or moved until every completion is collected.
        unsafe { ring.submission().push(&read).expect("queue full") };
    }
    ring.submit_and_wait(offsets.len())?;
    for cqe in ring.completion() {
        let n = cqe_result(cqe.result())?;
        buffers[cqe.user_data() as usize].truncate(n); // a read at the end may be short
    }
    Ok(buffers)
}
```

И тест, в модуле `tests`:

```rust
#[test]
fn reads_blocks_in_the_order_asked() -> io::Result<()> {
    let path = std::env::temp_dir().join(format!("blocks-{}.bin", std::process::id()));
    fs::write(&path, b"aaaabbbbccccdd")?;
    let file = File::open(&path)?;
    let blocks = read_blocks(&file, &[8, 0, 12], 4)?;
    fs::remove_file(&path)?;
    assert_eq!(
        blocks,
        vec![b"cccc".to_vec(), b"aaaa".to_vec(), b"dd".to_vec()]
    );
    Ok(())
}
```

(Добавьте `use std::fs;` в начало модуля `tests`.) `cargo test`:

```text
test result: ok. 3 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

Почему это работает: `user_data` несёт позицию в `offsets`, поэтому каждое завершение находит свой буфер. Пока
ядро работает, программа не трогает `buffers`; ядро пишет в них через сырые указатели. `truncate` выполняется
только после возврата из `submit_and_wait`, когда все чтения завершены.

</details>

## Вопросы для самопроверки

1. В чём главное различие между epoll и io_uring?

<details>
<summary>Ответ</summary>

epoll сообщает о *готовности*: каждую операцию вы всё равно выполняете своим системным вызовом. io_uring
использует *завершение*: вы описываете операции, ядро их выполняет и сообщает результаты.

</details>

2. Что такое SQ и CQ и кто пишет в каждую?

<details>
<summary>Ответ</summary>

Очередь отправки, куда программа пишет операции (SQE) для ядра, и очередь завершения, куда ядро пишет
результаты (CQE) для программы. Обе — кольцевые буферы в памяти, общей для обеих сторон.

</details>

3. Почему добавление SQE в крейте `io-uring` — это `unsafe`?

<details>
<summary>Ответ</summary>

SQE содержит сырые указатели на буферы, которыми ядро воспользуется позже. Компилятор не может проверить, что
буферы живут и не перемещаются до завершения операции; это должен гарантировать программист.

</details>

4. Программа отправляет 10 чтений и получает 10 CQE. Как она узнаёт, какой результат к какому чтению
   относится?

<details>
<summary>Ответ</summary>

По `user_data`: каждый SQE несёт число, выбранное программой, и ядро копирует его в соответствующий CQE.
Порядок CQE не совпадает с порядком отправки.

</details>

5. `IoUring::new` в контейнере Docker завершается ошибкой «Operation not permitted». Почему?

<details>
<summary>Ответ</summary>

Профиль seccomp Docker по умолчанию запрещает системные вызовы io_uring. (На обычной машине его также может
выключить sysctl `kernel.io_uring_disabled`.)

</details>

## Итоги

- io_uring — интерфейс ввода-вывода Linux на модели завершения: опишите операции, пусть ядро их выполнит, соберите результаты.
- Два кольцевых буфера в общей памяти: SQ (программа → ядро, SQE) и CQ (ядро → программа, CQE); `io_uring_enter` отправляет и ждёт.
- Много операций может быть в работе одновременно и завершаться в любом порядке; `user_data` сопоставляет результаты с запросами.
- `result` — число байтов или отрицательный errno; буферы должны жить до своего завершения, поэтому добавление — `unsafe`.
- Один системный вызов может отправить целую пачку, что важно при очень большом числе операций.
- io_uring может быть отключён (sysctl, seccomp в Docker); асинхронные API на модели завершения должны владеть своими буферами.

## Источники и что почитать

- [io_uring(7) — Linux manual page](https://man7.org/linux/man-pages/man7/io_uring.7.html) — обзор интерфейса, колец и элементов.
- [io_uring_enter(2) — Linux manual page](https://man7.org/linux/man-pages/man2/io_uring_enter.2.html) — отправка и ожидание и все коды операций.
- [io-uring на docs.rs](https://docs.rs/io-uring/0.7.16/io_uring/) — крейт Rust из этого урока.
- [Efficient IO with io_uring](https://kernel.dk/io_uring.pdf) — статья о дизайне от автора io_uring, Jens Axboe.
- [Lord of the io_uring](https://unixism.net/loti/) — длинное руководство (примеры на C), чтобы разобраться глубже.

---

[🏠 Начало](../START-HERE.md) · [← 13.13 Неблокирующий ввод-вывод и epoll](13-non-blocking-io-and-epoll.md) · [📋 Этап 13](README.md) · [13.15 Пространства имён (namespaces) →](15-namespaces.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
