<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 13.12 Файловые системы изнутри](12-file-systems-inside.md) · [📋 Этап 13](README.md) · [13.14 io_uring →](14-io-uring.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/13-operating-systems/13-non-blocking-io-and-epoll.md) <!-- nav -->

# Урок 13.13 — Неблокирующий ввод-вывод и epoll

> **Этап 13 — Операционные системы изнутри** · Урок 13 из 18 · ⏱ около 135 минут

## Что вы узнаете

- Почему обычный `read` *блокирует* и почему это проблема для программы, которая обслуживает много входов сразу.
- Как перевести файловый дескриптор в неблокирующий режим и что означает `WouldBlock` (EAGAIN).
- Как `epoll` позволяет одному потоку ждать много дескрипторов и просыпаться, только когда один из них готов.
- Чем уведомление по уровню отличается от уведомления по фронту и почему код с уведомлением по фронту должен вычитывать всё.
- Как эта модель «готовности» лежит в основе асинхронных рантаймов вроде Tokio.

## Перед началом

- Вы прошли [Урок 13.12 — Файловые системы изнутри](12-file-systems-inside.md).
- Вы знаете файловые дескрипторы ([Урок 13.4](04-files-and-file-descriptors.md)), конвейеры ([Урок 13.6](06-pipes-and-redirection-inside.md)) и потоки ([Урок 13.9](09-threads-and-scheduling.md)).
- Вы уже встречали `async`/`await` ([Урок 06.9](../06-rust-advanced/09-async-await-introduction.md)); этот урок показывает, что под ними.
- Создайте пакет на Mac и скопируйте его в ВМ, как в Уроке 13.10:

```bash
cd ~/projects/rust
cargo new --lib code/p13-epoll
cd code/p13-epoll
cargo add nix@0.32 --features event,fs
mkdir src/bin
```

## Новые слова

| Слово | Значение |
|---|---|
| **блокирующий вызов** (blocking call) | Системный вызов, который усыпляет поток, пока не сможет завершиться, например `read` из пустого конвейера. |
| **неблокирующий режим** (non-blocking mode, `O_NONBLOCK`) | Флаг файлового дескриптора: вызовы, которым пришлось бы ждать, сразу возвращают ошибку. |
| **EAGAIN / `WouldBlock`** | Ошибка, которую возвращает неблокирующий вызов, если ему пришлось бы ждать («попробуйте позже»). В Rust это `io::ErrorKind::WouldBlock`. |
| **мультиплексирование ввода-вывода** (I/O multiplexing) | I/O (input/output) — ввод-вывод. Мультиплексирование ввода-вывода — наблюдение за многими дескрипторами одним потоком и обработка того, который готов. |
| **готовность** (readiness) | Состояние «вызов для этого дескриптора сейчас не заблокирует»: есть что читать или есть место для записи. |
| **epoll** | Интерфейс Linux для мультиплексирования ввода-вывода: вы один раз регистрируете дескрипторы, а потом спрашиваете «какие готовы?». |
| **список интересов** (interest list) | Набор дескрипторов (и событий), за которыми следит экземпляр epoll. |
| **уведомление по уровню** (level-triggered, LT) | epoll сообщает о дескрипторе *всё время, пока* он готов. Режим по умолчанию. |
| **уведомление по фронту** (edge-triggered, ET) | epoll сообщает о дескрипторе, только когда он *становится* готовым (приходят новые данные). |
| **активное ожидание** (busy waiting) | Снова и снова спрашивать в цикле, готово ли что-то, сжигая время CPU (Central Processing Unit, центральный процессор). |
| **цикл событий** (event loop) | Главный цикл программы, который ждёт событий и обрабатывает их одно за другим. |

## 1. Проблема блокирования

Когда вы вызываете `read` для пустого конвейера, ядро усыпляет ваш поток до прихода данных. Это
**блокирующий вызов**. Для одного входа он идеален: пока поток ждёт, CPU не тратится.

Теперь представьте программу, которая обслуживает много входов: чат-сервер с 10 000 подключённых
пользователей или утилиту, которая следит за выводом нескольких дочерних процессов. Заблокированный поток
может ждать только **один** дескриптор. Пока он спит на пользователе A, сообщение пользователя B не
обрабатывается.

Выхода три:

| Способ | Идея | Проблема |
|---|---|---|
| поток на каждый вход | каждый поток блокируется на своём дескрипторе | 10 000 потоков стоят памяти (у каждого стек) и работы планировщика (Урок 13.9) |
| неблокирующий режим + цикл | по очереди спрашивать каждый дескриптор «есть что-нибудь для меня?» | **активное ожидание**: когда ничего не происходит, цикл крутится на 100% CPU |
| **мультиплексирование** | попросить ядро «разбуди меня, когда *любой* из них будет готов» | ничего из перечисленного; это тема урока |

Третьему способу нужны два кирпичика: неблокирующие дескрипторы и `epoll`.

## 2. Неблокирующий режим

У каждого открытого файла есть флаги. Флаг `O_NONBLOCK` меняет правило: вызов, которому пришлось бы ждать,
сразу возвращает ошибку **EAGAIN** («ресурс временно недоступен, попробуйте снова»). Rust превращает её в
`io::ErrorKind::WouldBlock`. В стандартной библиотеке нет функции, чтобы поставить этот флаг на конвейер,
поэтому мы используем `fcntl` («file control», управление файлом) из `nix`. Создайте `src/bin/nonblock.rs`:

```rust
use std::fs::File;
use std::io::{ErrorKind, Read, Write};

use nix::fcntl::{FcntlArg, OFlag, fcntl};
use nix::unistd::pipe;

fn main() -> std::io::Result<()> {
    let (read_end, write_end) = pipe()?;
    fcntl(&read_end, FcntlArg::F_SETFL(OFlag::O_NONBLOCK))?; // switch the read end to non-blocking

    let mut reader = File::from(read_end);
    let mut writer = File::from(write_end);
    let mut buf = [0u8; 16];

    match reader.read(&mut buf) {
        Err(e) if e.kind() == ErrorKind::WouldBlock => println!("empty pipe: {e}"),
        other => println!("unexpected: {other:?}"),
    }

    writer.write_all(b"hi")?;
    let n = reader.read(&mut buf)?;
    println!(
        "after a write: {n} bytes, {:?}",
        String::from_utf8_lossy(&buf[..n])
    );
    Ok(())
}
```

**Что происходит:**

1. `pipe()` возвращает два значения `OwnedFd` (Урок 13.6). Ошибки `nix` превращаются в `io::Error`, поэтому
   `?` работает в функции, которая возвращает `io::Result`.
2. `fcntl(..., F_SETFL(O_NONBLOCK))` ставит флаг на конец для чтения.
3. `File::from(fd)` оборачивает дескриптор в `File`, чтобы пользоваться обычными трейтами `Read` и `Write`.
4. Первый `read` находит конвейер пустым. Вместо того чтобы уснуть, он возвращает `WouldBlock`
   (os error 11 = EAGAIN).
5. После записи `read` сразу возвращает два байта.

```text
empty pipe: Resource temporarily unavailable (os error 11)
after a write: 2 bytes, "hi"
```

Сам по себе неблокирующий режим нашу проблему не решает. Он только говорит «не сейчас». Нам всё ещё нужен
способ уснуть до тех пор, пока не наступит «сейчас». Это `epoll`.

## 3. epoll: ожидание многих дескрипторов сразу

`epoll` работает в три шага, каждый — системный вызов:

```text
1. epoll_create1()           -> a new epoll instance (itself a file descriptor)
2. epoll_ctl(ADD, fd, events) -> put fd on the interest list, once per descriptor
3. epoll_wait(timeout)        -> sleep until at least one fd is ready; get the list of ready ones

           interest list                 ready list
   epoll  +------------------+  data   +-------------+
   fd 3   | fd 4  EPOLLIN  0 | ------> | fd 6 (id 1) |  -> returned by epoll_wait
          | fd 6  EPOLLIN  1 | arrives +-------------+
          | fd 8  EPOLLIN  2 |
          +------------------+
```

Ядро ведёт оба списка. Когда на наблюдаемый дескриптор приходят данные, ядро кладёт его в список готовых и
будит ждущий поток. Ожидание ничего не стоит, а работа при каждом пробуждении зависит от того, сколько
дескрипторов *готово*, а не от того, сколько наблюдается. Поэтому один поток может следить за десятками
тысяч соединений.

Каждая регистрация несёт число на ваш выбор (`data`), которое `epoll_wait` возвращает обратно. Мы используем
его как индекс в нашей собственной таблице читателей. Создайте `src/bin/watch3.rs`:

```rust
use std::fs::File;
use std::io::{Read, Write};
use std::thread;
use std::time::{Duration, Instant};

use nix::sys::epoll::{Epoll, EpollCreateFlags, EpollEvent, EpollFlags, EpollTimeout};
use nix::unistd::pipe;

fn main() -> std::io::Result<()> {
    let start = Instant::now();
    let epoll = Epoll::new(EpollCreateFlags::empty())?;
    let mut readers = Vec::new();

    // Three "workers": each writes one message into its own pipe after a delay, then closes it.
    for (id, delay_ms) in [(0u64, 300u64), (1, 100), (2, 200)] {
        let (read_end, write_end) = pipe()?;
        epoll.add(&read_end, EpollEvent::new(EpollFlags::EPOLLIN, id))?;
        readers.push(File::from(read_end));
        thread::spawn(move || {
            let mut writer = File::from(write_end);
            thread::sleep(Duration::from_millis(delay_ms));
            writer
                .write_all(format!("hello from worker {id}").as_bytes())
                .unwrap();
        }); // `writer` is dropped here: the write end is closed
    }

    let mut open = readers.len();
    let mut events = [EpollEvent::empty(); 8];
    while open > 0 {
        let ready = epoll.wait(&mut events, EpollTimeout::NONE)?; // sleep until something is ready
        for event in &events[..ready] {
            let id = event.data() as usize;
            let mut buf = [0u8; 64];
            let n = readers[id].read(&mut buf)?;
            let t = start.elapsed().as_millis();
            if n == 0 {
                println!("{t:>4} ms  pipe {id}: closed");
                epoll.delete(&readers[id])?;
                open -= 1;
            } else {
                println!(
                    "{t:>4} ms  pipe {id}: {}",
                    String::from_utf8_lossy(&buf[..n])
                );
            }
        }
    }
    Ok(())
}
```

**Что происходит:**

1. `Epoll::new` создаёт экземпляр (`epoll_create1`). `Epoll` владеет своим дескриптором и закрывает его при
   уничтожении.
2. Для каждого «работника» мы делаем конвейер и регистрируем конец для чтения с `EPOLLIN` («скажи, когда будет
   что читать») и числом `id`. Потоки здесь только изображают медленные источники; в настоящем сервере это были
   бы сетевые соединения (Этап 15).
3. `epoll.wait(&mut events, EpollTimeout::NONE)` спит без ограничения времени и заполняет `events` максимум
   восемью готовыми дескрипторами. Возвращает, сколько заполнил.
4. `event.data()` возвращает наш `id`. Мы читаем из этого конвейера. Раз epoll сказал «готов», этот `read`
   не заблокирует.
5. `read`, вернувший 0, означает конец файла: работник закрыл свой конец. Мы убираем дескриптор из списка
   интересов через `epoll.delete` и уменьшаем счётчик.

```text
 100 ms  pipe 1: hello from worker 1
 100 ms  pipe 1: closed
 202 ms  pipe 2: hello from worker 2
 202 ms  pipe 2: closed
 303 ms  pipe 0: hello from worker 0
 303 ms  pipe 0: closed
```

Один поток обработал три источника в том порядке, в каком они стали готовы, а не в том, в каком были
зарегистрированы. Миллисекунды немного меняются от запуска к запуску.

`strace -f` (`-f` следит и за потоками) показывает системные вызовы; вот первые строки:

```bash
strace -f -e trace=epoll_create1,epoll_ctl,epoll_pwait ./target/debug/watch3
```

```text
epoll_create1(0)                        = 3
epoll_ctl(3, EPOLL_CTL_ADD, 4, {events=EPOLLIN, data=0}) = 0
strace: Process 284 attached
[pid   283] epoll_ctl(3, EPOLL_CTL_ADD, 6, {events=EPOLLIN, data=0x1}) = 0
strace: Process 285 attached
[pid   283] epoll_ctl(3, EPOLL_CTL_ADD, 8, {events=EPOLLIN, data=0x2}) = 0
strace: Process 286 attached
[pid   283] epoll_pwait(3, [{events=EPOLLIN, data=0x1}], 8, -1, NULL, 8) = 1
 102 ms  pipe 1: hello from worker 1
[pid   283] epoll_pwait(3, [{events=EPOLLHUP, data=0x1}], 8, -1, NULL, 8) = 1
 102 ms  pipe 1: closed
[pid   283] epoll_ctl(3, EPOLL_CTL_DEL, 6, NULL) = 0
```

Две детали:

- В Linux на arm64 нет системного вызова `epoll_wait`; библиотека C реализует его через `epoll_pwait` (без
  маски сигналов, `NULL`). `-1` означает «без тайм-аута».
- Второе пробуждение сообщает `EPOLLHUP` («hang up», другой конец закрыт), хотя мы просили только `EPOLLIN`.
  epoll всегда сообщает о закрытии и об ошибках.

## 4. Уведомление по уровню и по фронту

Когда epoll должен сообщать о дескрипторе? Есть два ответа:

- **По уровню** (по умолчанию): *пока* есть непрочитанные данные. Если прочитать только часть, следующий
  `wait` снова сообщит о дескрипторе. Как дверной звонок, который звонит, пока кто-то стоит у двери.
- **По фронту** (`EPOLLET`): только когда состояние *меняется*, то есть приходят новые данные. Если оставить
  данные непрочитанными, epoll будет молчать, пока не придут ещё. Как звонок, который звонит один раз.

Посмотрим на разницу. Создайте `src/bin/triggers.rs`:

```rust
use std::fs::File;
use std::io::{Read, Write};

use nix::fcntl::{FcntlArg, OFlag, fcntl};
use nix::sys::epoll::{Epoll, EpollCreateFlags, EpollEvent, EpollFlags};
use nix::unistd::pipe;

fn run(name: &str, flags: EpollFlags) -> std::io::Result<()> {
    let (read_end, write_end) = pipe()?;
    fcntl(&read_end, FcntlArg::F_SETFL(OFlag::O_NONBLOCK))?;
    let epoll = Epoll::new(EpollCreateFlags::empty())?;
    epoll.add(&read_end, EpollEvent::new(flags, 0))?;
    let mut reader = File::from(read_end);
    File::from(write_end).write_all(b"0123456789")?; // 10 bytes, write end closed at once

    println!("{name}:");
    let mut events = [EpollEvent::empty(); 1];
    loop {
        let ready = epoll.wait(&mut events, 100u16)?; // wait at most 100 ms
        if ready == 0 {
            println!("  no event within 100 ms");
            return Ok(());
        }
        let mut buf = [0u8; 4]; // read only 4 bytes per wake-up
        let n = reader.read(&mut buf)?;
        println!(
            "  woke up, read {n} bytes: {:?}",
            String::from_utf8_lossy(&buf[..n])
        );
        if n == 0 {
            return Ok(());
        }
    }
}

fn main() -> std::io::Result<()> {
    run("level-triggered", EpollFlags::EPOLLIN)?;
    run("edge-triggered", EpollFlags::EPOLLIN | EpollFlags::EPOLLET)?;
    Ok(())
}
```

**Что происходит:**

1. Записываются десять байт, и конец для записи закрывается в той же строке (временный `File` уничтожается).
2. Тайм-аут `100u16` означает «ждать не дольше 100 мс»; если ничего не случилось, `wait` возвращает 0.
3. Каждое пробуждение намеренно читает только 4 байта.

```text
level-triggered:
  woke up, read 4 bytes: "0123"
  woke up, read 4 bytes: "4567"
  woke up, read 2 bytes: "89"
  woke up, read 0 bytes: ""
edge-triggered:
  woke up, read 4 bytes: "0123"
  no event within 100 ms
```

В режиме по уровню epoll сообщал о конвейере, пока всё, включая конец файла, не было прочитано. В режиме по
фронту он сообщил один раз; шесть байт всё ещё ждут, и нового события не будет никогда, потому что новых
данных не приходит. Программа, написанная так, зависла бы навсегда.

Правило для режима по фронту: **после каждого события читайте, пока не получите `WouldBlock`** (это
называется *вычитыванием*, draining), а значит, всегда используйте с ним неблокирующие дескрипторы. Зачем
вообще его выбирать? Когда много потоков ждут на одном epoll или дескрипторы очень загружены, он избавляет от
повторных пробуждений из-за одних и тех же данных. Его используют высокопроизводительные библиотеки, например
`mio`. В своём коде начинайте с режима по уровню.

> ⚡ **Производительность:** стоимость epoll растёт с числом *готовых* дескрипторов за одно пробуждение, а не
> с числом наблюдаемых. Старые вызовы `select` и `poll` передают ядру весь список при каждом вызове, поэтому
> замедляются по мере роста списка.

## 5. От epoll к асинхронному Rust

Посмотрите ещё раз на цикл в `watch3`: дождаться событий, понять, чьё это событие, выполнить для него
небольшой кусок кода, вернуться к ожиданию. Такая форма — это **цикл событий**, и ровно это делает
асинхронный рантайм:

```text
   your async tasks:   task A        task B        task C
                     .await read   .await read   .await sleep
                          \             |             /
                    +------------------------------------+
                    | runtime (Tokio): event loop        |
                    |   mio -> epoll_wait (Linux)        |
                    |         kqueue    (macOS)          |
                    +------------------------------------+
```

- Крейт `mio` («metal I/O») оборачивает epoll в Linux и kqueue (аналог) в macOS одним API (Application Programming Interface, программный интерфейс: набор функций, который предлагает библиотека).
- Tokio построен на `mio`. Когда `async`-функция делает `.await` на сокете, который не готов, Tokio
  регистрирует сокет в epoll и выполняет другие задачи. Когда epoll сообщает о готовности, Tokio продолжает
  задачу.

Крошечный рантайм вы напишете сами на Этапе 14
([Урок 14.11](../14-concurrency-and-async/11-build-a-tiny-async-runtime.md)). Обратите внимание на одно
ограничение модели готовности: epoll говорит, *когда* можно читать, но сам `read` вы делаете сами. Следующий
урок, [13.14](14-io-uring.md), показывает другую модель, где работу делает ядро и сообщает, когда она
*выполнена*.

## Частые ошибки

1. **Активное ожидание на `WouldBlock`.** Цикл вроде `loop { match file.read(..) { Err(WouldBlock) => continue, … } }`
   работает, но, пока ждёт, держит одно ядро CPU на 100%. *Исправление:* получив `WouldBlock`, возвращайтесь
   к `epoll.wait`.

2. **Режим по фронту без вычитывания.** Как показала `triggers`, об остатке данных никто не сообщит.
   *Исправление:* после каждого события читайте до `WouldBlock` (упражнение 3) или используйте режим по уровню.

3. **Регистрация обычного файла.** epoll работает с конвейерами, сокетами, терминалами и похожими объектами,
   но не с обычными файлами, которые всегда «готовы». Программа из упражнения 2, запущенная как
   `tick < input.txt`, завершается ошибкой:

   ```text
   Error: Os { code: 1, kind: PermissionDenied, message: "Operation not permitted" }
   ```

   Это EPERM («operation not permitted», операция не разрешена) от `epoll_ctl`. *Исправление:* читайте обычные файлы обычным способом (или через io_uring,
   Урок 13.14).

4. **Закрытые дескрипторы не удаляются.** Если оставить закрытый конвейер в списке интересов (или продолжать
   его читать), epoll в режиме по уровню снова и снова сообщает о конце файла. *Исправление:* когда `read`
   вернул 0, вызовите `epoll.delete` и уничтожьте читателя.

## Попробуйте сами

1. Запустите `watch3` и поменяйте задержки так, чтобы два работника заканчивали одновременно, например
   `[(0, 200), (1, 200), (2, 100)]`. Возвращает ли один `wait` оба? Напечатайте `ready`, чтобы увидеть.
2. Выполните `ls -l /proc/$(pgrep watch3)/fd` во втором терминале, пока `watch3` ждёт (сначала сделайте
   задержки длиннее, например 10 секунд). Один дескриптор показан как `anon_inode:[eventpoll]`: это сам
   экземпляр epoll.
3. Уберите из `nonblock` строку с `fcntl` и запустите снова. Что происходит при первом `read`? (Остановите
   программу через Ctrl-C.)

## Упражнения

### Упражнение 1 — Протестированный `try_read` (легко)

Напишите `pub fn try_read(source: &mut impl Read, buf: &mut [u8]) -> io::Result<Option<usize>>` в
`src/lib.rs`: `Ok(None)` для `WouldBlock`, `Ok(Some(n))` для данных или конца файла, остальные ошибки без
изменений. Проверьте её неблокирующим конвейером: пусто → `None`; после записи `abc` → `Some(3)`; после
уничтожения писателя → `Some(0)`.

<details>
<summary>Подсказка</summary>

Напишите тестовую функцию `fn nonblocking_pipe() -> (File, File)`, которая создаёт конвейер, ставит
`O_NONBLOCK` на конец для чтения и оборачивает оба конца в `File`.

</details>

<details>
<summary>Ответ</summary>

```rust
use std::io::{self, ErrorKind, Read};

/// Reads without waiting: `Ok(None)` means "no data right now, try later".
pub fn try_read(source: &mut impl Read, buf: &mut [u8]) -> io::Result<Option<usize>> {
    match source.read(buf) {
        Ok(n) => Ok(Some(n)),
        Err(e) if e.kind() == ErrorKind::WouldBlock => Ok(None),
        Err(e) => Err(e),
    }
}

#[cfg(test)]
mod tests {
    use super::*;
    use nix::fcntl::{FcntlArg, OFlag, fcntl};
    use nix::unistd::pipe;
    use std::fs::File;
    use std::io::Write;

    /// A pipe whose read end is non-blocking: (reader, writer).
    fn nonblocking_pipe() -> (File, File) {
        let (r, w) = pipe().unwrap();
        fcntl(&r, FcntlArg::F_SETFL(OFlag::O_NONBLOCK)).unwrap();
        (File::from(r), File::from(w))
    }

    #[test]
    fn empty_pipe_gives_none_then_data_gives_some() {
        let (mut reader, mut writer) = nonblocking_pipe();
        let mut buf = [0u8; 8];
        assert_eq!(try_read(&mut reader, &mut buf).unwrap(), None);
        writer.write_all(b"abc").unwrap();
        assert_eq!(try_read(&mut reader, &mut buf).unwrap(), Some(3));
        assert_eq!(&buf[..3], b"abc");
    }

    #[test]
    fn closed_pipe_gives_end_of_file() {
        let (mut reader, writer) = nonblocking_pipe();
        drop(writer);
        let mut buf = [0u8; 8];
        assert_eq!(try_read(&mut reader, &mut buf).unwrap(), Some(0));
    }
}
```

Почему это работает: `Option` отделяет две нормальные ситуации («пока ничего» и «получено n байт») от
настоящих ошибок. `impl Read` позволяет функции работать с любым читателем, и её легко тестировать.

</details>

### Упражнение 2 — `tick`: ввод с тайм-аутом (средне)

Напишите `src/bin/tick.rs`. Программа регистрирует стандартный ввод в epoll и ждёт не дольше секунды за раз.
Если за секунду не пришло ни одной строки, она печатает `tick`. Каждая строка повторяется как
`you typed: <line>`. В конце ввода программа печатает `bye` и завершается. Проверьте её конвейером, который
выдаёт строки медленно:

```bash
(sleep 2.5; echo hello; sleep 0.3; echo world) | ./target/debug/tick
```

<details>
<summary>Подсказка</summary>

`io::stdin()` реализует `AsFd`, поэтому `&stdin` можно передать в `epoll.add`. Тайм-аут в 1000 мс — это
`1000u16`. Читайте строки через `stdin.lock().lines()`.

</details>

<details>
<summary>Ответ</summary>

```rust
use std::io::{self, BufRead};

use nix::sys::epoll::{Epoll, EpollCreateFlags, EpollEvent, EpollFlags};

fn main() -> io::Result<()> {
    let stdin = io::stdin();
    let epoll = Epoll::new(EpollCreateFlags::empty())?;
    epoll.add(&stdin, EpollEvent::new(EpollFlags::EPOLLIN, 0))?;

    let mut events = [EpollEvent::empty(); 1];
    let mut lines = stdin.lock().lines();
    loop {
        if epoll.wait(&mut events, 1000u16)? == 0 {
            println!("tick"); // a whole second without input
            continue;
        }
        match lines.next() {
            Some(line) => println!("you typed: {}", line?),
            None => {
                println!("bye");
                return Ok(());
            }
        }
    }
}
```

```text
tick
tick
you typed: hello
you typed: world
bye
```

Почему это работает: тайм-аут превращает «ничего не произошло» в обычный результат (0 событий). Одно
предостережение: `lines()` читает через буфер. Если две строки пришли одним куском, вторая лежит в *вашем*
буфере, а epoll, который знает только про буфер конвейера в ядре, о ней не сообщит. Здесь строки приходят с
интервалом 0,3 с, поэтому этого не происходит; настоящие циклы событий держат собственные буферы и проверяют
их перед новым ожиданием.

</details>

### Упражнение 3 — Вычитывание для режима по фронту (сложно)

Напишите `pub fn drain(source: &mut impl Read) -> io::Result<Vec<u8>>`, которая читает всё, что доступно
сейчас (через ваш `try_read` и маленький буфер на 4 байта), и останавливается на `WouldBlock` или конце файла.
Проверьте, что после записи `0123456789` один `drain` возвращает все десять байт, а второй `drain` ничего не
возвращает, пока писатель ещё открыт. Затем измените `triggers`, чтобы прогон в режиме по фронту использовал
`drain` и читал всё.

<details>
<summary>Подсказка</summary>

Цикл: `match try_read(source, &mut buf)? { Some(0) | None => return Ok(all), Some(n) => … }`.

</details>

<details>
<summary>Ответ</summary>

```rust
/// Reads everything that is available now from a non-blocking source.
/// With edge-triggered epoll you must drain like this after every event.
pub fn drain(source: &mut impl Read) -> io::Result<Vec<u8>> {
    let mut all = Vec::new();
    let mut buf = [0u8; 4];
    loop {
        match try_read(source, &mut buf)? {
            Some(0) | None => return Ok(all), // end of file, or nothing more for now
            Some(n) => all.extend_from_slice(&buf[..n]),
        }
    }
}
```

Тест, в том же модуле `tests`, что и в упражнении 1:

```rust
#[test]
fn drain_reads_everything_available() {
    let (mut reader, mut writer) = nonblocking_pipe();
    writer.write_all(b"0123456789").unwrap();
    assert_eq!(drain(&mut reader).unwrap(), b"0123456789");
    assert_eq!(drain(&mut reader).unwrap(), b""); // nothing left, writer still open
}
```

`cargo test`:

```text
test result: ok. 3 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

Почему это работает: цикл останавливается, только когда ядро говорит «сейчас больше ничего нет»
(`WouldBlock`) или «больше не будет никогда» (0). После этого любые новые данные — это новый *фронт*, и epoll
о них сообщит. Блокирующий дескриптор заставил бы второй `drain` уснуть навсегда; поэтому режиму по фронту
нужен неблокирующий режим.

</details>

## Вопросы для самопроверки

1. Что возвращает неблокирующий `read`, когда данных ещё нет, и как это показывает Rust?

<details>
<summary>Ответ</summary>

Он сразу возвращает ошибку EAGAIN («попробуйте снова»). В Rust это `io::Error`, у которого `kind()` равен
`ErrorKind::WouldBlock`.

</details>

2. Почему «поток на каждое соединение» — плохая схема для 10 000 соединений?

<details>
<summary>Ответ</summary>

Каждому потоку нужен свой стек и учётные структуры ядра, а планировщик должен переключаться между ними.
Большинство из них только спит. Один поток с epoll может ждать все соединения почти бесплатно.

</details>

3. Какие три системных вызова у epoll и что делает каждый?

<details>
<summary>Ответ</summary>

`epoll_create1` создаёт экземпляр; `epoll_ctl` добавляет, меняет или удаляет дескриптор в списке интересов;
`epoll_wait` (на arm64 — `epoll_pwait`) спит, пока какие-то дескрипторы не станут готовы, и возвращает их.

</details>

4. Вы прочитали только часть доступных данных. Что будет дальше в режиме по уровню и в режиме по фронту?

<details>
<summary>Ответ</summary>

По уровню: следующий `wait` снова сообщит о дескрипторе, потому что данные ещё есть. По фронту: ничего не
сообщается, пока не придут новые данные, и остаток может ждать вечно. Код с режимом по фронту должен
вычитывать всё.

</details>

5. Как epoll связан с Tokio?

<details>
<summary>Ответ</summary>

Цикл событий Tokio использует `mio`, который в Linux использует epoll. Задача, которая ждёт неготовый сокет,
откладывается; за её сокетом следит epoll, и задача продолжается, когда epoll сообщает о готовности.

</details>

## Итоги

- Блокирующий вызов спит, пока не сможет завершиться; хорошо для одного входа, плохо для многих.
- `O_NONBLOCK` заставляет вызовы возвращать `WouldBlock` (EAGAIN) вместо сна; сам по себе он ведёт к активному ожиданию.
- epoll: создать экземпляр, зарегистрировать дескрипторы с числом на ваш выбор, затем ждать готовых через `wait`; стоимость зависит от готовых дескрипторов, а не от наблюдаемых.
- О закрытии (`EPOLLHUP`) epoll сообщает всегда; `read`, вернувший 0, означает конец файла — удалите дескриптор.
- Режим по уровню повторяет уведомление, пока есть данные; режим по фронту срабатывает один раз на изменение и требует вычитывать всё до `WouldBlock`.
- epoll — механизм готовности под `mio` и Tokio; обычные файлы через него наблюдать нельзя.

## Источники и что почитать

- [epoll(7) — Linux manual page](https://man7.org/linux/man-pages/man7/epoll.7.html) — полное описание, включая режимы по уровню и по фронту и их подводные камни.
- [epoll_ctl(2) — Linux manual page](https://man7.org/linux/man-pages/man2/epoll_ctl.2.html) — все флаги событий и ошибки, например EPERM для обычных файлов.
- [nix::sys::epoll на docs.rs](https://docs.rs/nix/0.32.0/nix/sys/epoll/index.html) — безопасная обёртка из этого урока.
- [mio на docs.rs](https://docs.rs/mio/1.2.5/mio/) — кроссплатформенная библиотека событий под Tokio.
- [Operating Systems: Three Easy Pieces](https://pages.cs.wisc.edu/~remzi/OSTEP/) — глава «Event-based Concurrency (Advanced)».

---

[🏠 Начало](../START-HERE.md) · [← 13.12 Файловые системы изнутри](12-file-systems-inside.md) · [📋 Этап 13](README.md) · [13.14 io_uring →](14-io-uring.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
