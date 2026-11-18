<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 15.5 UDP-сокеты в Rust](05-udp-sockets-in-rust.md) · [📋 Этап 15](README.md) · [15.7 Проектирование протокола: фрейминг и разбор →](07-designing-a-protocol.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/15-networking/06-async-networking-with-tokio.md) <!-- nav -->

# Урок 15.6 — Асинхронная работа с сетью в Tokio

> **Этап 15 — Сети** · Урок 6 из 16 · ⏱ около 135 минут

## Что вы узнаете

- Почему схема «один поток на соединение» перестаёт работать на тысячах соединений и что делает вместо этого Tokio.
- Как писать TCP-серверы на `tokio::net`: цикл `accept` с одной задачей на соединение.
- Как одновременно читать строки и писать, разделив поток на половину для чтения и половину для записи.
- Как построить чат-сервер, который рассылает сообщения всем клиентам, с помощью `tokio::sync::broadcast` и `tokio::select!`.
- Как добавить тайм-ауты простоя и тестировать асинхронные серверы с `#[tokio::test]`.

## Перед началом

- Вы прошли [Урок 15.5 — UDP-сокеты в Rust](05-udp-sockets-in-rust.md).
- Вы знаете `async`/`.await` и основы Tokio из [Урока 06.9](../06-rust-advanced/09-async-await-introduction.md), задачи и каналы — из [Урока 14.13 — Tokio подробно: задачи и каналы](../14-concurrency-and-async/13-tokio-tasks-and-channels.md), а тайм-ауты и корректное завершение — из [Урока 14.14](../14-concurrency-and-async/14-cancellation-timeouts-and-graceful-shutdown.md).
- Вы знаете, как epoll и kqueue ждут сразу много файловых дескрипторов ([Урок 13.13 — Неблокирующий ввод-вывод и epoll](../13-operating-systems/13-non-blocking-io-and-epoll.md)).
- Добавьте Tokio в учебный пакет (версия берётся с crates.io; 2026-11-17 это была 1.54.1):

```bash
cd ~/projects/rust/code/p15-net
cargo add tokio@1 --features full
```

## Новые слова

| Слово | Значение |
|---|---|
| **задача** (task) | Единица асинхронной работы, которую выполняет Tokio; запускается через `tokio::spawn`. Многие тысячи задач делят несколько потоков. |
| **цикл accept** (accept loop) | Цикл, который принимает соединения и запускает по задаче на каждое. |
| **половина для чтения / для записи** (read half / write half) | Две части разделённого `TcpStream`: одна умеет только читать, другая — только писать. Каждую можно использовать в своём месте. |
| **широковещательный канал** (broadcast channel) | Канал, в котором каждое сообщение доставляется *каждому* получателю, а не одному. |
| **отставание** (lagging) | Получатель широковещательного канала, который так медлителен, что каналу пришлось выбросить ещё не прочитанные им сообщения. |
| **`select!`** | Макрос Tokio, который ждёт сразу несколько future и выполняет ветку того, что завершился первым. |
| **тайм-аут простоя** (idle timeout) | Закрытие соединения, которое ничего не отправляло определённое время. |
| **предел файловых дескрипторов** (file descriptor limit) | Наибольшее число открытых файлов (и сокетов) на процесс; его показывает `ulimit -n`. |
| **TCP / UDP / SYN** (Transmission Control Protocol / User Datagram Protocol / synchronise — протокол управления передачей / протокол пользовательских датаграмм / синхронизация) | Два транспортных протокола из Урока 15.3 и флаг TCP в первом сегменте рукопожатия. |
| **API** (Application Programming Interface, программный интерфейс) | Типы и функции, которые библиотека предлагает вашему коду. |

## 1. Почему потоков не хватает

Сервер «поток на соединение» из Урока 15.4 прост и правилен. Его беда — цена каждого соединения. У
каждого потока свой стек (Rust по умолчанию даёт новым потокам 2 МиБ адресного пространства), а
операционной системе приходится переключаться между потоками. При десяти соединениях это ничто. При
десяти тысячах в основном простаивающих соединений (чат, игра, балансировщик нагрузки) у вас десять
тысяч потоков, которые почти всегда просто ждут.

Асинхронный рантайм переворачивает это:

```text
 threads (Lesson 15.4)                       Tokio (this lesson)
 ┌───────────┐ ┌───────────┐                 ┌────────────────────────────────────────┐
 │ thread 1  │ │ thread 2  │  ...  10,000    │ a few worker threads (one per CPU core)│
 │ blocked   │ │ blocked   │       threads   │  run whichever task is ready           │
 │ in read() │ │ in read() │                 ├────────────────────────────────────────┤
 └───────────┘ └───────────┘                 │ 10,000 tasks: small state machines     │
 each one: a stack, a kernel thread          │ waiting cost: a few hundred bytes each │
                                             ├────────────────────────────────────────┤
                                             │ reactor: epoll (Linux) / kqueue (macOS)│
                                             │ "which sockets have data now?"         │
                                             └────────────────────────────────────────┘
```

Сокет Tokio — **неблокирующий** (Урок 13.13). Когда задача вызывает `stream.read(&mut buf).await`, а
данных нет, задача не блокирует поток: она возвращает `Pending`, реактор запоминает, что эта задача
ждёт этот сокет, а поток выполняет другую задачу. Когда ядро сообщает «на том сокете есть данные»,
задачу будят, и она продолжает с места после своего `.await`. Вы пишете код, который *выглядит*
блокирующим, а рантайм делает ожидание дешёвым.

## 2. Асинхронный эхо-сервер

API `tokio::net` повторяет `std::net`; методы асинхронные, а методы чтения и записи берутся из трейтов
`AsyncReadExt` и `AsyncWriteExt`. Сохраните как `src/bin/async_echo.rs`:

```rust
use tokio::io::{AsyncReadExt, AsyncWriteExt};
use tokio::net::{TcpListener, TcpStream};

async fn handle(mut stream: TcpStream) -> std::io::Result<()> {
    let mut buf = [0u8; 1024];
    loop {
        let n = stream.read(&mut buf).await?;
        if n == 0 {
            return Ok(()); // EOF: клиент закрыл соединение
        }
        stream.write_all(&buf[..n]).await?;
    }
}

#[tokio::main]
async fn main() -> std::io::Result<()> {
    let listener = TcpListener::bind("127.0.0.1:7200").await?;
    println!("async echo server on {}", listener.local_addr()?);
    loop {
        let (stream, peer) = listener.accept().await?;
        println!("{peer}: connected");
        tokio::spawn(async move {
            if let Err(e) = handle(stream).await {
                eprintln!("{peer}: error: {e}");
            }
            println!("{peer}: done");
        });
    }
}
```

**Что происходит:**

1. `#[tokio::main]` запускает рантайм и выполняет `main` как первую задачу.
2. `TcpListener::bind(...).await` и `listener.accept().await` работают как их версии из `std`, но
   ожидание не блокирует поток.
3. `tokio::spawn` запускает новую **задачу** для каждого соединения. Блок `async move` забирает
   владение `stream` и `peer`. Цикл сразу возвращается к `accept`.
4. `handle` — эхо-цикл из Урока 15.4 с добавленным `.await`. `read` возвращает `Ok(0)` при EOF, как и
   раньше.

Запустите его и подключитесь через `nc 127.0.0.1 7200`, введите `hello` и нажмите Ctrl+C:

```text
async echo server on 127.0.0.1:7200
127.0.0.1:51883: connected
127.0.0.1:51883: done
```

Теперь о главном. Следующая программа запускает тот же эхо-сервер, а затем открывает 5000 клиентских
соединений **одновременно**; каждое отправляет сообщение и проверяет эхо. Сохраните как
`src/bin/many_clients.rs`:

```rust
use std::sync::Arc;
use std::time::Instant;
use tokio::io::{AsyncReadExt, AsyncWriteExt};
use tokio::net::{TcpListener, TcpStream};
use tokio::sync::Semaphore;
use tokio::task::JoinSet;

async fn echo(mut stream: TcpStream) -> std::io::Result<()> {
    let mut buf = [0u8; 64];
    loop {
        let n = stream.read(&mut buf).await?;
        if n == 0 {
            return Ok(());
        }
        stream.write_all(&buf[..n]).await?;
    }
}

#[tokio::main]
async fn main() -> std::io::Result<()> {
    let listener = TcpListener::bind("127.0.0.1:0").await?;
    let addr = listener.local_addr()?;
    tokio::spawn(async move {
        while let Ok((stream, _)) = listener.accept().await {
            tokio::spawn(echo(stream));
        }
    });

    let clients = 5_000;
    let connecting = Arc::new(Semaphore::new(100)); // не больше 100 рукопожатий одновременно
    let start = Instant::now();
    let mut set = JoinSet::new();
    for i in 0..clients {
        let connecting = Arc::clone(&connecting);
        set.spawn(async move {
            let permit = connecting.acquire_owned().await.expect("semaphore open");
            let mut stream = TcpStream::connect(addr).await?;
            let msg = format!("client {i}");
            stream.write_all(msg.as_bytes()).await?;
            let mut buf = vec![0u8; msg.len()];
            stream.read_exact(&mut buf).await?;
            assert_eq!(buf, msg.as_bytes());
            drop(permit);
            Ok::<_, std::io::Error>(stream) // держим соединение открытым
        });
    }
    let mut open = Vec::new();
    while let Some(result) = set.join_next().await {
        open.push(result.expect("task panicked")?);
    }
    println!("{} connections open at the same time", open.len());
    println!("all echoed in {:?}", start.elapsed());
    Ok(())
}
```

`cargo run --bin many_clients` на Mac:

```text
5000 connections open at the same time
all echoed in 105.960416ms
```

**Что происходит:**

1. 5000 клиентских задач и 5000 серверных задач (10 000 сокетов) живут в одном процессе, на нескольких
   рабочих потоках Tokio. Каждый клиент держит соединение открытым до конца (`open` хранит потоки).
2. `JoinSet` собирает результаты многих задач; `join_next` возвращает их по мере завершения.
3. `Semaphore` (счётчик разрешений, Урок 14.13) позволяет делать рукопожатие одновременно не более чем
   100 клиентам. Без него все 5000 SYN приходят разом и переполняют очередь ожидания: macOS держит не
   больше 128 ожидающих соединений (`sysctl kern.ipc.somaxconn`), и лишние клиенты получают ошибку
   `ConnectionReset`. Настоящие клиенты никогда не приходят в одну и ту же микросекунду, так что это
   честная имитация.

Если вы видите `Error: Os { code: 24, kind: TooManyOpenFiles, message: "Too many open files" }`, ваша
командная оболочка разрешает процессу слишком мало открытых файлов. Проверьте командой `ulimit -n`;
если там `256`, поднимите предел для этого терминала командой `ulimit -n 20000` и запустите снова.

> ⚡ **Производительность:** прежде чем делать выводы, измеряйте с `--release` и помните, что
> измеряет этот тест: 5000 соединений, каждое из которых обменивается несколькими байтами через
> loopback. Асинхронность помогает больше всего, когда у вас *много соединений, которые в основном
> ждут*. Для нескольких загруженных соединений потоки ничуть не хуже.

## 3. Строки и чтение во время записи

Для построчных протоколов оберните сторону чтения в `tokio::io::BufReader` и вызовите `lines()` из
трейта `AsyncBufReadExt`. Часто задаче нужно ещё и *писать*, пока она ждёт чтения, например клиенту
чата, который получает сообщения других, пока его пользователь молчит. Для этого разделите поток
через `let (reader, mut writer) = stream.into_split();` и оберните только читающую часть:
`BufReader::new(reader).lines()`. `into_split` даёт `OwnedReadHalf` и `OwnedWriteHalf`. Это отдельные
значения, поэтому одна часть кода может ждать в `lines.next_line().await`, пока другая пишет через
`writer.write_all(...)`. (Есть ещё `stream.split()`, который заимствует, а не перемещает; используйте
его, когда обе половины остаются в одной функции.)

## 4. Чат-сервер: broadcast и `select!`

Чат-сервер должен доставлять каждое сообщение *каждому* подключённому клиенту. Именно это делает
**широковещательный канал**: каждый получатель получает каждое сообщение. Теперь каждая задача
соединения ждёт **двух** вещей сразу: строки от своего клиента или сообщения из канала.
`tokio::select!` ждёт обе и выполняет ветку той, что пришла первой. Сохраните как `src/bin/chat.rs`:

```rust
use tokio::io::{AsyncBufReadExt, AsyncWriteExt, BufReader};
use tokio::net::{TcpListener, TcpStream};
use tokio::sync::broadcast;

async fn handle(
    stream: TcpStream,
    tx: broadcast::Sender<String>,
    name: String,
) -> std::io::Result<()> {
    let mut rx = tx.subscribe();
    let (reader, mut writer) = stream.into_split();
    let mut lines = BufReader::new(reader).lines();
    let _ = tx.send(format!("* {name} joined\n"));
    loop {
        tokio::select! {
            line = lines.next_line() => match line? {
                Some(text) => { let _ = tx.send(format!("{name}: {text}\n")); }
                None => break, // EOF: клиент ушёл
            },
            msg = rx.recv() => match msg {
                Ok(text) => writer.write_all(text.as_bytes()).await?,
                Err(broadcast::error::RecvError::Lagged(n)) => {
                    writer.write_all(format!("* {n} messages skipped\n").as_bytes()).await?;
                }
                Err(broadcast::error::RecvError::Closed) => break,
            },
        }
    }
    let _ = tx.send(format!("* {name} left\n"));
    Ok(())
}

#[tokio::main]
async fn main() -> std::io::Result<()> {
    let listener = TcpListener::bind("127.0.0.1:7201").await?;
    println!("chat on {}", listener.local_addr()?);
    let (tx, _) = broadcast::channel::<String>(100);
    loop {
        let (stream, peer) = listener.accept().await?;
        let tx = tx.clone();
        tokio::spawn(async move {
            if let Err(e) = handle(stream, tx, peer.to_string()).await {
                eprintln!("{peer}: {e}");
            }
        });
    }
}
```

**Что происходит:**

1. `broadcast::channel(100)` создаёт отправителя и первого получателя (которого мы выбрасываем через
   `_`). Каждая задача получает клон отправителя и вызывает `subscribe()`, чтобы получить собственного
   получателя.
2. В `select!` обе ветки — future: `lines.next_line()` завершается, когда клиент отправил строку,
   `rx.recv()` — когда в канал пришло чьё-нибудь сообщение. Тот, кто завершился первым, выполняет свою
   ветку; другой future уничтожается и создаётся заново на следующем витке цикла. Оба этих future
   *безопасны при отмене* (cancel-safe): их уничтожение не теряет данных (почему это важно, объясняет
   Урок 14.14).
3. Строка от клиента отправляется в канал, поэтому каждая задача, включая эту, пишет её своему
   клиенту.
4. Канал хранит не больше 100 непрочитанных сообщений на получателя. Клиент, который читает слишком
   медленно, **отстаёт**: его получатель получает `Lagged(n)`, и мы сообщаем клиенту, сколько сообщений
   он пропустил, вместо того чтобы позволить памяти расти без предела. Это обратное давление
   (Урок 14.13) с ясной политикой: медленные читатели теряют сообщения, быстрых никто не тормозит.

Запустите сервер и подключите два окна `nc 127.0.0.1 7201`. Когда клиент A вводит `hello from A`, а
клиент B — `hi A, this is B`, A видит:

```text
* 127.0.0.1:52540 joined
* 127.0.0.1:52542 joined
127.0.0.1:52540: hello from A
127.0.0.1:52542: hi A, this is B
* 127.0.0.1:52542 left
```

(Последняя строка появляется, когда B нажимает Ctrl+C. Здесь клиенты названы по своим адресам;
настоящие имена добавляет Упражнение 3.)

> 🔒 **Безопасность:** этот сервер полностью доверяет своим клиентам. Клиент может прислать строку в
> гигабайт (`lines()` читает, пока не найдёт перевод строки) или молчать вечно. Прежде чем такой сервер
> встретится с интернетом, ему нужны предельная длина строки, тайм-аут простоя (Упражнение 2) и предел
> соединений (Упражнение 3 из Урока 15.4). Урок 15.7 показывает, как ограничивать размер кадров.

## 5. UDP и завершение работы в Tokio

У `tokio::net::UdpSocket` те же методы, что у версии из `std` в Уроке 15.5, только с `.await`: `bind`,
`send_to`, `recv_from`, `connect`. Тайм-ауты записываются через `tokio::time::timeout(duration,
future)` вместо `set_read_timeout`.

Настоящему серверу нужно ещё и **корректное завершение** (graceful shutdown): перестать принимать
соединения, дать текущим закончиться, затем выйти. Вы строили его в Уроке 14.14. Для сетевого сервера
цикл `accept` использует `select!` с двумя ветками: `listener.accept()` запускает задачу, как раньше, а
`tokio::signal::ctrl_c()` выходит из цикла, так что новые соединения больше не принимаются. Работающие
задачи отслеживаются (например, с помощью `JoinSet` или `CancellationToken` из `tokio-util`), чтобы
`main` мог дождаться их, прежде чем вернуться. Этот шаблон использует проект балансировщика нагрузки
(Урок 15.15).

> 🧪 **Тестирование:** асинхронные серверы тестируются так же, как многопоточные: привяжитесь к
> `127.0.0.1:0`, запустите сервер через `tokio::spawn`, подключитесь через `tokio::net::TcpStream`.
> Пометьте тест `#[tokio::test]` вместо `#[test]`; он создаёт рантайм для каждого теста. Оборачивайте
> шаги ожидания в `tokio::time::timeout`, чтобы ошибка роняла тест через секунду, а не вешала его
> навсегда.

## Частые ошибки

> ⚠️ **Частая ошибка: блокировка внутри задачи.** `std::thread::sleep`, `std::net::TcpStream` или
> долгое вычисление внутри асинхронной задачи блокирует целый рабочий поток, и все остальные задачи на
> нём ждут. Используйте `tokio::time::sleep`, `tokio::net` и `tokio::task::spawn_blocking` для тяжёлой
> работы (Урок 06.9).

> ⚠️ **Частая ошибка: трейты `std::io` для потоков Tokio.** У `tokio::net::TcpStream` нет метода
> `read`, пока вы не импортируете `tokio::io::AsyncReadExt`. Компилятор говорит:
>
> ```text
> error[E0599]: no method named `read` found for struct `tokio::net::TcpStream` in the current scope
> ```
>
> и предлагает импорт. Используйте `AsyncReadExt`, `AsyncWriteExt` и `AsyncBufReadExt` из `tokio::io`,
> а не `std::io::Read` и `Write`.

## Попробуйте сами

1. Запустите `chat` и подключите три окна `nc`. Закройте одно через Ctrl+C и посмотрите, как остальные
   получают `* ... left`.
2. Замените `broadcast::channel::<String>(100)` на `(2)`, подключите двух клиентов и быстро вставьте
   много строк в одного из них. Иногда другой клиент видит `* N messages skipped`.
3. В `many_clients.rs` замените `Semaphore::new(100)` на `Semaphore::new(5000)` и запустите несколько
   раз. Скорее всего, вы увидите `ConnectionReset` из-за переполненной очереди ожидания.
4. Запустите `many_clients` командой `cargo run --release --bin many_clients` и сравните время.

## Упражнения

### Упражнение 1 — Кто в сети? (легко)

Измените `async_echo.rs`, чтобы он печатал, сколько клиентов подключено, каждый раз, когда кто-то
подключается или уходит, например `127.0.0.1:51879: connected (1 online)` и
`127.0.0.1:51879: done (0 online)`.

<details>
<summary>Подсказка</summary>

Разделите `Arc<AtomicUsize>` между циклом `accept` и задачами (Урок 14.5). `fetch_add` и `fetch_sub`
возвращают *старое* значение.

</details>

<details>
<summary>Ответ</summary>

`handle` остаётся как в разделе 2; `main` становится таким:

```rust
#[tokio::main]
async fn main() -> std::io::Result<()> {
    let listener = TcpListener::bind("127.0.0.1:7200").await?;
    let online = Arc::new(AtomicUsize::new(0));
    loop {
        let (stream, peer) = listener.accept().await?;
        let now = online.fetch_add(1, Ordering::SeqCst) + 1;
        println!("{peer}: connected ({now} online)");
        let online = Arc::clone(&online);
        tokio::spawn(async move {
            let _ = handle(stream).await;
            let now = online.fetch_sub(1, Ordering::SeqCst) - 1;
            println!("{peer}: done ({now} online)");
        });
    }
}
```

с `use std::sync::Arc;` и `use std::sync::atomic::{AtomicUsize, Ordering};` в начале файла. С двумя
клиентами `nc`, из которых первый остаётся дольше:

```text
127.0.0.1:51879: connected (1 online)
127.0.0.1:51880: connected (2 online)
127.0.0.1:51880: done (1 online)
127.0.0.1:51879: done (0 online)
```

Почему это работает: атомарный счётчик можно менять из многих задач без блокировки; прибавив или
вычтя 1 из возвращённого старого значения, получаем новое число.

</details>

### Упражнение 2 — Отключайте молчащих клиентов, с тестами (средне)

Напишите `src/bin/idle.rs`: эхо-сервер на `127.0.0.1:7202`, чей обработчик принимает
`idle: Duration`. Если клиент ничего не отправляет столько времени, сервер пишет `bye: idle\n` и
закрывает соединение. Напишите два теста `#[tokio::test]`: молчащий клиент получает ровно
`bye: idle\n`, а затем EOF; активный клиент получает своё эхо.

<details>
<summary>Подсказка</summary>

Оберните каждое чтение: `tokio::time::timeout(idle, stream.read(&mut buf)).await` возвращает
`Err(Elapsed)`, когда время вышло, и `Ok(результат_чтения)` в противном случае. В тесте используйте
короткий `idle`, например 100 мс, и `read_to_string` на клиенте, чтобы дождаться EOF.

</details>

<details>
<summary>Ответ</summary>

```rust
use std::time::Duration;
use tokio::io::{AsyncReadExt, AsyncWriteExt};
use tokio::net::{TcpListener, TcpStream};
use tokio::time::timeout;

/// An echo handler that hangs up on clients that stay silent for longer than `idle`.
async fn handle(mut stream: TcpStream, idle: Duration) -> std::io::Result<()> {
    let mut buf = [0u8; 1024];
    loop {
        let n = match timeout(idle, stream.read(&mut buf)).await {
            Ok(result) => result?,
            Err(_elapsed) => {
                stream.write_all(b"bye: idle\n").await?;
                return Ok(());
            }
        };
        if n == 0 {
            return Ok(());
        }
        stream.write_all(&buf[..n]).await?;
    }
}

#[tokio::main]
async fn main() -> std::io::Result<()> {
    let listener = TcpListener::bind("127.0.0.1:7202").await?;
    loop {
        let (stream, _) = listener.accept().await?;
        tokio::spawn(handle(stream, Duration::from_secs(10)));
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    async fn start(idle: Duration) -> std::net::SocketAddr {
        let listener = TcpListener::bind("127.0.0.1:0").await.unwrap();
        let addr = listener.local_addr().unwrap();
        tokio::spawn(async move {
            let (stream, _) = listener.accept().await.unwrap();
            handle(stream, idle).await.unwrap();
        });
        addr
    }

    #[tokio::test]
    async fn silent_client_is_disconnected() {
        let addr = start(Duration::from_millis(100)).await;
        let mut client = TcpStream::connect(addr).await.unwrap();
        let mut answer = String::new();
        client.read_to_string(&mut answer).await.unwrap(); // ждёт EOF
        assert_eq!(answer, "bye: idle\n");
    }

    #[tokio::test]
    async fn active_client_is_served() {
        let addr = start(Duration::from_millis(500)).await;
        let mut client = TcpStream::connect(addr).await.unwrap();
        client.write_all(b"ping").await.unwrap();
        let mut buf = [0u8; 4];
        client.read_exact(&mut buf).await.unwrap();
        assert_eq!(&buf, b"ping");
    }
}
```

`cargo test --bin idle` сообщает `2 passed; ... finished in 0.10s`.

Почему это работает: `timeout` устраивает гонку между чтением и таймером. Когда побеждает таймер,
future чтения уничтожается (отменяется), и для `read` это безопасно, потому что ни одного байта из
сокета не было взято. Возврат из `handle` уничтожает поток, что закрывает соединение, поэтому
`read_to_string` на клиенте видит EOF.

</details>

### Упражнение 3 — Чат с именами и тестом (сложно)

Напишите `src/bin/named_chat.rs`: как `chat.rs`, но сервер сначала пишет `name? `, и первая строка,
которую отправит клиент, становится его именем. Сообщения выглядят как `ann: hello`. Вынесите цикл
`accept` в `async fn serve(listener: TcpListener)`, чтобы тест мог запустить его на порту 0. Напишите
тест с двумя клиентами, `ann` и `bob`: когда `bob` отправляет `hi everyone`, оба получают
`bob: hi everyone`.

<details>
<summary>Подсказка</summary>

Читайте имя через `lines.next_line().await?` *до* подписки. В тесте напишите помощника, который
читает строки, пока одна не закончится ожидаемым текстом, внутри `tokio::time::timeout`, потому что
между ними приходят другие строки (`name? `, сообщения о входе). Пусть `ann` увидит своё
`* ann joined` до того, как подключится `bob`: тогда вы знаете, что задача ann уже подписалась и не
пропустит сообщения bob.

</details>

<details>
<summary>Ответ</summary>

```rust
use tokio::io::{AsyncBufReadExt, AsyncWriteExt, BufReader};
use tokio::net::{TcpListener, TcpStream};
use tokio::sync::broadcast;

async fn handle(stream: TcpStream, tx: broadcast::Sender<String>) -> std::io::Result<()> {
    let (reader, mut writer) = stream.into_split();
    let mut lines = BufReader::new(reader).lines();
    writer.write_all(b"name? ").await?;
    let Some(name) = lines.next_line().await? else {
        return Ok(()); // ушёл, не назвав имени
    };
    let name = name.trim().to_string();
    let mut rx = tx.subscribe();
    let _ = tx.send(format!("* {name} joined\n"));
    loop {
        tokio::select! {
            line = lines.next_line() => match line? {
                Some(text) => { let _ = tx.send(format!("{name}: {text}\n")); }
                None => break,
            },
            msg = rx.recv() => match msg {
                Ok(text) => writer.write_all(text.as_bytes()).await?,
                Err(broadcast::error::RecvError::Lagged(_)) => continue,
                Err(broadcast::error::RecvError::Closed) => break,
            },
        }
    }
    let _ = tx.send(format!("* {name} left\n"));
    Ok(())
}

async fn serve(listener: TcpListener) -> std::io::Result<()> {
    let (tx, _) = broadcast::channel::<String>(100);
    loop {
        let (stream, _) = listener.accept().await?;
        tokio::spawn(handle(stream, tx.clone()));
    }
}

#[tokio::main]
async fn main() -> std::io::Result<()> {
    serve(TcpListener::bind("127.0.0.1:7203").await?).await
}

#[cfg(test)]
mod tests {
    use super::*;
    use std::time::Duration;
    use tokio::io::Lines;
    use tokio::net::tcp::{OwnedReadHalf, OwnedWriteHalf};
    use tokio::time::timeout;

    type Incoming = Lines<BufReader<OwnedReadHalf>>;

    async fn join(addr: std::net::SocketAddr, name: &str) -> (Incoming, OwnedWriteHalf) {
        let (r, mut w) = TcpStream::connect(addr).await.unwrap().into_split();
        w.write_all(format!("{name}\n").as_bytes()).await.unwrap();
        (BufReader::new(r).lines(), w)
    }

    /// Reads lines until one ends with `expected`; fails after one second.
    async fn wait_for(lines: &mut Incoming, expected: &str) {
        let found = timeout(Duration::from_secs(1), async {
            while let Some(line) = lines.next_line().await.unwrap() {
                if line.ends_with(expected) {
                    return true;
                }
            }
            false // соединение закрылось
        });
        assert!(found.await.unwrap_or(false), "never got {expected:?}");
    }

    #[tokio::test]
    async fn messages_carry_the_sender_name() {
        let listener = TcpListener::bind("127.0.0.1:0").await.unwrap();
        let addr = listener.local_addr().unwrap();
        tokio::spawn(serve(listener));

        let (mut ann_in, _ann_out) = join(addr, "ann").await;
        wait_for(&mut ann_in, "* ann joined").await; // теперь ann подписана
        let (mut bob_in, mut bob_out) = join(addr, "bob").await;
        wait_for(&mut ann_in, "* bob joined").await;

        bob_out.write_all(b"hi everyone\n").await.unwrap();
        wait_for(&mut ann_in, "bob: hi everyone").await;
        wait_for(&mut bob_in, "bob: hi everyone").await;
    }
}
```

`cargo test --bin named_chat` сообщает `1 passed` (пять запусков подряд прошли).

Почему это работает: имя читается до подписки задачи, поэтому клиент без имени никогда не получает
сообщений чата. Тест избегает гонки: получатель широковещательного канала получает только сообщения,
отправленные *после* его подписки, поэтому тест сначала ждёт, пока собственное сообщение ann о входе не
вернётся к ней. Первая строка, которую получает ann, — `name? * ann joined` (у приглашения нет
перевода строки), поэтому помощник проверяет `ends_with`, а не равенство.

</details>

## Вопросы для самопроверки

1. Почему ждущая задача Tokio дешевле ждущего потока?

<details>
<summary>Ответ</summary>

Ждущая задача — это лишь маленький конечный автомат в памяти; ей не нужен собственный стек и поток
ядра. Несколько рабочих потоков выполняют все задачи, а реактор (epoll/kqueue) сообщает рантайму,
какие сокеты готовы.

</details>

2. Что произойдёт, если вызвать `std::thread::sleep(Duration::from_secs(5))` внутри задачи соединения?

<details>
<summary>Ответ</summary>

Рабочий поток заблокируется на 5 секунд. Все остальные задачи на этом потоке в это время стоят.
Используйте вместо этого `tokio::time::sleep(...).await`.

</details>

3. Зачем в чат-сервере мы делим `TcpStream` на половины для чтения и для записи?

<details>
<summary>Ответ</summary>

Задача должна ждать строку от своего клиента и одновременно иметь возможность писать этому клиенту
сообщения других. Две отдельные половины можно использовать в двух ветках `select!`, не споря за одно
заимствование `&mut`.

</details>

4. Чем канал `mpsc` отличается от канала `broadcast`?

<details>
<summary>Ответ</summary>

В `mpsc` каждое сообщение получает ровно один получатель (он там один). В `broadcast` каждый
подписанный получатель получает копию каждого сообщения.

</details>

5. Клиент чата перестал читать, но остаётся подключённым. Что происходит с его получателем
   широковещательного канала и чем это хорошо для сервера?

<details>
<summary>Ответ</summary>

Его получатель отстаёт больше чем на 100 сообщений, и канал выбрасывает самые старые; следующий
`recv` возвращает `Lagged(n)`. Память остаётся ограниченной, а остальные клиенты не замедляются.

</details>

## Итоги

- «Поток на соединение» платит стеком и потоком ядра за каждое соединение; Tokio выполняет много задач
  на нескольких потоках и с помощью epoll/kqueue будит только готовые.
- `tokio::net::TcpListener` + `accept().await` + `tokio::spawn` на каждое соединение — базовый
  асинхронный сервер; чтение и запись берутся из `AsyncReadExt` и `AsyncWriteExt`.
- `into_split` даёт независимые половины для чтения и записи; `BufReader::lines()` читает целые
  строки.
- `broadcast` доставляет каждое сообщение каждому подписчику; `select!` ждёт несколько событий и
  выполняет первое; `Lagged` — это способ, которым ограниченный широковещательный канал обходится с
  медленными читателями.
- Добавляйте тайм-ауты простоя через `tokio::time::timeout`, пределы на строки и соединения и
  корректное завершение.
- Тестируйте с `#[tokio::test]`, портом `0` и тайм-аутами вокруг каждого ожидания. Следите за
  `ulimit -n` и очередью ожидания, когда открываете тысячи соединений.

## Источники и что почитать

- [Tokio tutorial — I/O](https://tokio.rs/tokio/tutorial/io) — чтение, запись, разделение и эхо-сервер от авторов Tokio.
- [Tokio tutorial — Select](https://tokio.rs/tokio/tutorial/select) — как работает `select!` и что значит безопасность при отмене.
- [`tokio::net`](https://docs.rs/tokio/latest/tokio/net/index.html) — API `TcpListener`, `TcpStream` и `UdpSocket`.
- [`tokio::sync::broadcast`](https://docs.rs/tokio/latest/tokio/sync/broadcast/index.html) — широковещательный канал и его правила отставания.
- [Tokio — graceful shutdown](https://tokio.rs/tokio/topics/shutdown) — официальное руководство по корректному завершению сервера.

---

[🏠 Начало](../START-HERE.md) · [← 15.5 UDP-сокеты в Rust](05-udp-sockets-in-rust.md) · [📋 Этап 15](README.md) · [15.7 Проектирование протокола: фрейминг и разбор →](07-designing-a-protocol.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
