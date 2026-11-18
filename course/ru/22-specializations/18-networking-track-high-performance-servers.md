<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 22.17 Трек баз данных: проект и следующие шаги](17-databases-track-project-and-next-steps.md) · [📋 Этап 22](README.md) · [22.19 Трек сетей: разбор протоколов без копирования →](19-networking-track-zero-copy-parsing.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/22-specializations/18-networking-track-high-performance-servers.md) <!-- nav -->

# Урок 22.18 — Трек сетей: высокопроизводительные серверы

> **Этап 22 — Специализации** · Урок 18 из 30 · ⏱ около 2,5 часа (2 занятия)

> **Не ваше направление?** Этот урок открывает сетевое направление (Уроки 22.18–22.21). Если вы
> выбрали другое направление, переходите к [Уроку 22.22 — Трек компиляторов: лексический и синтаксический анализ](22-compilers-track-lexing-and-parsing.md)
> или сразу к [Уроку 22.30 — Контрольная точка: ваша специализация](30-checkpoint-your-specialization.md).

## Что вы узнаете

- Что строят сетевые инженеры на Rust и что значит «высокая производительность» для сервера.
- Как самостоятельно создать слушающий сокет с крейтом `socket2` и задать ему параметры сокета.
- Что делает `TCP_NODELAY` и почему, забыв о нём, можно добавить 40 миллисекунд к каждому запросу.
- Как `SO_REUSEPORT` позволяет нескольким потокам принимать соединения на одном порту и как это даёт сервер «поток на ядро».
- Какие ограничения (очередь backlog, файловые дескрипторы, лимит соединений) защищают сервер под нагрузкой.

## Перед началом

- Вы прошли предыдущие этапы. Больше всего это направление опирается на [Урок 15.4 — TCP-сокеты в Rust](../15-networking/04-tcp-sockets-in-rust.md),
  [Урок 15.6 — Асинхронная работа с сетью в Tokio](../15-networking/06-async-networking-with-tokio.md),
  [Урок 13.13 — Неблокирующий ввод-вывод и epoll](../13-operating-systems/13-non-blocking-io-and-epoll.md) и
  [Урок 19.2 — Нагрузочное тестирование сервисов](../19-performance-and-profiling/02-load-testing.md).
- Код направления живёт в одном пакете. Создайте его; каждый пример кладётся в `src/bin/`:

```bash
cd ~/projects/rust
cargo new code/p22-net
cd code/p22-net
cargo add tokio@1 --features full
cargo add socket2@0.6
```

- Некоторые примеры ведут себя по-разному в Linux и macOS. Серверы в продакшене работают в Linux,
  поэтому запускайте примеры в вашей Linux-ВМ `dev` из Этапа 11 (`limactl shell dev`, затем те же
  команды). Выводы ниже получены в Linux; различия с macOS названы там, где они важны.

## Новые слова

| Слово | Значение |
|---|---|
| **плоскость данных** (data plane) | Часть сетевой системы, которая перемещает сами байты (прокси, балансировщики нагрузки, маршрутизаторы). *Плоскость управления* (control plane) решает, куда байты должны идти. |
| **параметр сокета** (socket option) | Настройка сокета, которую читают и записывают системными вызовами `getsockopt` и `setsockopt`, например размер его буферов. |
| **`socket2`** | Крейт, который даёт безопасный доступ почти ко всем параметрам сокета и позволяет создавать сокет пошагово. |
| **TCP** (Transmission Control Protocol, протокол управления передачей) | Протокол, который даёт надёжный упорядоченный поток байтов между двумя программами (Урок 15.3). |
| **ACK** (acknowledgement, подтверждение) | Маленькое сообщение TCP, которое говорит: «я получил ваши байты вот до этого места». |
| **алгоритм Нейгла** (Nagle's algorithm) | Правило TCP, которое придерживает маленькие записи, пока предыдущие данные ещё не подтверждены, чтобы много крошечных записей ушли одним пакетом. |
| **отложенный ACK** (delayed ACK) | Получатель ждёт немного (в Linux примерно до 40 мс), прежде чем отправить ACK, надеясь приложить ACK к ответу. |
| **`TCP_NODELAY`** | Параметр сокета, который выключает алгоритм Нейгла. |
| **`SO_REUSEPORT`** | Параметр сокета, который позволяет нескольким сокетам слушать один и тот же адрес и порт. В Linux ядро распределяет новые соединения между ними. |
| **поток на ядро** (thread-per-core) | Архитектура сервера с одним потоком на каждое ядро CPU (Central Processing Unit, центральный процессор); у каждого потока свой цикл событий и свой слушающий сокет, общего между ними как можно меньше. |
| **backlog** | Длина очереди соединений, которые ядро уже установило, но программа ещё не приняла. |
| **очередь приёма** (accept queue) | Очередь ядра, длину которой ограничивает backlog. `accept` берёт соединения из неё. |
| **4-кортеж** (4-tuple) | Четыре числа, которые определяют TCP-соединение: IP-адрес (Internet Protocol, межсетевой протокол) источника, порт источника, IP-адрес назначения, порт назначения. |

**Другие сокращения в этом уроке:** ВМ (виртуальная машина; Linux-ВМ `dev` из Этапа 11), IPv4 и IPv6 (версии 4 и 6
протокола IP, Internet Protocol), МиБ (мебибайт, 1 048 576 байтов) и RFC (Request for Comments, «запрос комментариев» —
официальный документ интернет-стандартов).

## 1. О чём это направление

Сетевые инженеры, которые пишут на Rust, создают программы в середине сети: прокси, балансировщики
нагрузки, API-шлюзы (API — Application Programming Interface, программный интерфейс), серверы DNS (Domain Name System, система доменных имён), программы VPN (Virtual
Private Network, виртуальная частная сеть) и обработчики пакетов. Rust подходит для этой работы, потому
что такие программы должны быть быстрыми *и* никогда не падать на странном вводе из интернета. Крупные
компании ставят прокси на Rust перед значительной частью веба; с одним из них, Pingora, вы встретитесь
в Уроке 22.20.

В направлении четыре урока:

| Урок | Навык |
|---|---|
| 22.18 (этот) | Сервер вокруг сокетов: параметры сокетов, сколько потоков, сколько слушающих сокетов, ограничения |
| 22.19 | Чтение протоколов из байтов без их копирования |
| 22.20 | Pingora, фреймворк для прокси в продакшене |
| 22.21 | Проект: собственный HTTP-прокси (HTTP — Hypertext Transfer Protocol, протокол передачи гипертекста) и куда двигаться дальше |

«Высокая производительность» для сервера — это три числа, которые вы измеряли на Этапе 19:

- **пропускная способность** — сколько запросов (или байтов) в секунду он обрабатывает;
- **задержка** — сколько длится один запрос, особенно медленные (99-й перцентиль, p99);
- **конкурентность** — сколько соединений он может держать открытыми одновременно.

Основные инструменты у вас уже есть с Этапа 15: цикл приёма на Tokio с одной задачей на соединение.
Этот урок добавляет рычаги, лежащие под этим циклом. Большинство из них — *параметры сокетов*, с них
и начнём.

## 2. Слушающий сокет с `socket2`

`tokio::net::TcpListener::bind` делает за вас четыре системных вызова: `socket`,
`setsockopt(SO_REUSEADDR)`, `bind` и `listen`. Он скрывает момент *между* ними, а некоторые параметры
нужно задавать именно там, до `bind`. Крейт `socket2` позволяет сделать каждый вызов самостоятельно.
Сохраните как `src/bin/listener.rs`:

```rust
use socket2::{Domain, Protocol, SockRef, Socket, Type};
use std::net::SocketAddr;
use tokio::net::TcpListener;

fn make_listener(addr: SocketAddr, backlog: i32) -> std::io::Result<TcpListener> {
    let socket = Socket::new(Domain::for_address(addr), Type::STREAM, Some(Protocol::TCP))?;
    socket.set_reuse_address(true)?;
    socket.set_nonblocking(true)?;
    socket.bind(&addr.into())?;
    socket.listen(backlog)?;
    TcpListener::from_std(socket.into())
}

#[tokio::main]
async fn main() -> std::io::Result<()> {
    let listener = make_listener("127.0.0.1:7300".parse().unwrap(), 1024)?;
    let sock = SockRef::from(&listener);
    println!("listening on {}", listener.local_addr()?);
    println!("SO_REUSEADDR = {}", sock.reuse_address()?);
    println!("SO_RCVBUF    = {} bytes", sock.recv_buffer_size()?);

    let client = tokio::net::TcpStream::connect("127.0.0.1:7300").await?;
    println!("new connection: TCP_NODELAY = {}", client.nodelay()?);
    Ok(())
}
```

**Что происходит:**

1. `Socket::new` — это системный вызов `socket`. `Domain::for_address(addr)` выбирает IPv4 или IPv6 в
   соответствии с адресом, `Type::STREAM` с `Protocol::TCP` запрашивает TCP-сокет.
2. `set_reuse_address(true)` задаёт `SO_REUSEADDR`: сервер можно перезапустить сразу, даже пока старые
   соединения предыдущего запуска ещё закрываются на этом порту.
3. `set_nonblocking(true)` делает сокет неблокирующим (Урок 13.13). Tokio этого требует, потому что его
   реактор никогда не должен застревать в вызове.
4. `bind` и `listen(backlog)` — системные вызовы с теми же именами. `socket.into()` превращает
   `socket2::Socket` в `std::net::TcpListener`, а `TcpListener::from_std` регистрирует его в Tokio.
5. `SockRef::from(&listener)` заимствует любой сокет как представление `socket2`, поэтому можно читать
   и задавать параметры и у сокетов, созданных Tokio.
6. В конце программа подключается сама к себе. Ядро устанавливает соединение ещё до того, как кто-то
   вызвал `accept`, и `nodelay()` показывает, что `TCP_NODELAY` у нового соединения выключен.

Запустите через `cargo run --bin listener`:

```text
listening on 127.0.0.1:7300
SO_REUSEADDR = true
SO_RCVBUF    = 131072 bytes
new connection: TCP_NODELAY = false
```

> 💡 **Совет:** буфер приёма (`SO_RCVBUF`) — это память ядра, в которой лежат байты, которые уже
> пришли, но ещё не прочитаны вашей программой. Большие буферы помогают быстрым передачам на дальние
> расстояния; для множества маленьких запросов хватает значения по умолчанию. Измерьте, прежде чем
> менять.

## 3. `TCP_NODELAY` и загадка 40 миллисекунд

В TCP есть два старых и разумных правила, которые вместе вызывают известную проблему:

- **Алгоритм Нейгла (отправитель):** «если я уже отправил маленький кусок, который ещё не подтверждён,
  придержу следующий маленький кусок до прихода ACK, потом отправлю всё вместе». Это экономит пакеты,
  когда программа пишет по одному байту.
- **Отложенный ACK (получатель):** «не отправлять ACK сразу; подождать немного, вдруг я отправлю ответ
  и смогу приложить ACK к нему». Это тоже экономит пакеты.

Теперь представьте сервер, который отвечает на запрос **двумя маленькими записями**: заголовком и
затем телом:

```text
 client                                   server (Nagle on)
   │ ── "GET\n" ───────────────────────────▶ │
   │ ◀──────────────────────── "LEN 5\n" ─── │  first small write: sent at once
   │   (client waits for the body,           │  second write "hello": Nagle holds it,
   │    so it has no reply to put an ACK on; │  because "LEN 5\n" is not acknowledged
   │    delayed ACK timer: about 40 ms)      │
   │ ── ACK (after ~40 ms) ─────────────────▶ │
   │ ◀────────────────────────── "hello" ─── │  now the body goes
```

Каждая сторона ждёт другую, и ожидание прерывает только таймер. Измерим это. Сохраните как
`src/bin/nagle.rs`:

```rust
use std::time::{Duration, Instant};
use tokio::io::{AsyncBufReadExt, AsyncReadExt, AsyncWriteExt, BufReader};
use tokio::net::{TcpListener, TcpStream};

// The server answers every request line with TWO small writes:
// first a header line, then the body.
async fn serve(listener: TcpListener, nodelay: bool) -> std::io::Result<()> {
    let (stream, _) = listener.accept().await?;
    stream.set_nodelay(nodelay)?;
    let (read_half, mut write_half) = stream.into_split();
    let mut lines = BufReader::new(read_half).lines();
    while let Some(_request) = lines.next_line().await? {
        write_half.write_all(b"LEN 5\n").await?;
        write_half.write_all(b"hello").await?;
    }
    Ok(())
}

async fn measure(nodelay: bool) -> std::io::Result<Duration> {
    let listener = TcpListener::bind("127.0.0.1:0").await?;
    let addr = listener.local_addr()?;
    tokio::spawn(serve(listener, nodelay));

    let mut client = TcpStream::connect(addr).await?;
    client.set_nodelay(true)?;
    let mut reply = [0u8; 11]; // "LEN 5\n" + "hello"
    let rounds = 20;
    let start = Instant::now();
    for _ in 0..rounds {
        client.write_all(b"GET\n").await?;
        client.read_exact(&mut reply).await?;
    }
    Ok(start.elapsed() / rounds)
}

#[tokio::main]
async fn main() -> std::io::Result<()> {
    for nodelay in [false, true] {
        let avg = measure(nodelay).await?;
        println!("server TCP_NODELAY = {nodelay:5}: {avg:?} per request");
    }
    Ok(())
}
```

**Что происходит:**

1. `measure` запускает сервер на свободном порту (`:0` означает «ядро, выбери порт») и подключается к
   нему.
2. Сервер ставит `TCP_NODELAY` в переданное значение и отвечает на каждую строку запроса двумя
   вызовами `write_all`.
3. Клиент отправляет 20 запросов один за другим и каждый раз читает ровно 11 байтов. Среднее время
   на запрос печатается для обеих настроек.

Вывод в Linux (ваши числа будут немного другими):

```text
server TCP_NODELAY = false: 39.039229ms per request
server TCP_NODELAY = true : 63.277µs per request
```

Та же работа примерно в **600 раз медленнее** с включённым алгоритмом Нейгла. В macOS таймер
отложенного ACK короче, поэтому вы увидите несколько миллисекунд вместо 40, но разница всё равно
большая.

Есть два исправления, и хорошие серверы применяют оба:

1. **Задайте `TCP_NODELAY`** на каждом принятом соединении сервера типа «запрос — ответ». Так делают
   почти все HTTP-серверы и прокси.
2. **Пишите один раз.** Сложите заголовок и тело в один буфер и вызовите `write_all` один раз (или
   используйте `write_vectored`, который передаёт ядру несколько буферов одним системным вызовом).
   Меньше системных вызовов — быстрее в любом случае.

> ⚡ **Производительность:** `TCP_NODELAY` не делает сеть быстрее. Он убирает *искусственное*
> ожидание. Если ваш сервер делает много крошечных записей, теперь он отправляет много крошечных
> пакетов, а это стоит времени CPU. Настоящее исправление — объединять записи в программе, где вы
> знаете, где кончается сообщение.

## 4. Много слушающих сокетов на одном порту: `SO_REUSEPORT`

Рантайм Tokio по умолчанию — с **перехватом работы** (work-stealing): несколько рабочих потоков делят
все задачи, и свободный поток забирает задачи у занятого (Урок 14.13). Это отличный выбор по
умолчанию. Некоторые очень нагруженные серверы используют другую архитектуру — **поток на ядро**:
один поток на ядро CPU, у каждого *свой* однопоточный рантайм и *свой* слушающий сокет. Между потоками
ничего не разделяется, поэтому нет блокировок и кэш-линии не перемещаются между ядрами (Урок 12.10).

Для этого несколько сокетов должны слушать один порт. Обычно второй `bind` падает с `AddrInUse`.
`SO_REUSEPORT`, заданный **до** `bind`, это разрешает. Сохраните как `src/bin/reuseport.rs`:

```rust
use socket2::{Domain, Protocol, Socket, Type};
use std::net::SocketAddr;
use std::sync::Arc;
use std::sync::atomic::{AtomicUsize, Ordering};
use std::time::Duration;

const WORKERS: usize = 4;
const CLIENTS: usize = 1000;

fn reuseport_listener(addr: SocketAddr) -> std::io::Result<std::net::TcpListener> {
    let socket = Socket::new(Domain::for_address(addr), Type::STREAM, Some(Protocol::TCP))?;
    socket.set_reuse_address(true)?;
    socket.set_reuse_port(true)?;
    socket.set_nonblocking(true)?;
    socket.bind(&addr.into())?;
    socket.listen(1024)?;
    Ok(socket.into())
}

// One worker = one thread with its own single-threaded Tokio runtime
// and its own listening socket on the SAME port.
fn worker(addr: SocketAddr, counter: Arc<AtomicUsize>) -> std::io::Result<()> {
    let std_listener = reuseport_listener(addr)?;
    let rt = tokio::runtime::Builder::new_current_thread()
        .enable_all()
        .build()?;
    rt.block_on(async move {
        let listener = tokio::net::TcpListener::from_std(std_listener)?;
        loop {
            let (stream, _) = listener.accept().await?;
            counter.fetch_add(1, Ordering::Relaxed);
            drop(stream); // a real server would spawn a task here
        }
    })
}

fn main() -> std::io::Result<()> {
    let addr: SocketAddr = "127.0.0.1:7301".parse().unwrap();
    let counters: Vec<Arc<AtomicUsize>> = (0..WORKERS)
        .map(|_| Arc::new(AtomicUsize::new(0)))
        .collect();
    for (id, counter) in counters.iter().enumerate() {
        let counter = Arc::clone(counter);
        std::thread::spawn(move || {
            if let Err(e) = worker(addr, counter) {
                eprintln!("worker {id}: {e}");
            }
        });
    }
    std::thread::sleep(Duration::from_millis(200)); // let all workers bind

    for _ in 0..CLIENTS {
        let _conn = std::net::TcpStream::connect(addr)?;
    }
    std::thread::sleep(Duration::from_millis(200)); // let workers count the last ones

    for (id, counter) in counters.iter().enumerate() {
        println!(
            "worker {id}: {} connections",
            counter.load(Ordering::Relaxed)
        );
    }
    Ok(())
}
```

**Что происходит:**

1. `main` запускает четыре обычных потока. Каждый вызывает `worker`, который создаёт свой слушающий
   сокет с `SO_REUSEPORT` на порту 7301 и свой рантайм `current_thread` (рантайм Tokio, который
   выполняет все задачи в одном потоке — в том, что вызвал `block_on`).
2. Все четыре вызова `bind` проходят успешно, потому что каждый сокет задал `SO_REUSEPORT` до `bind`.
3. Затем `main` открывает 1000 клиентских соединений одно за другим и сразу закрывает каждое.
4. Каждый рабочий поток считает принятые соединения в своём `AtomicUsize` (Урок 14.5). `Relaxed`
   достаточно, потому что нам нужны только итоговые числа.

Вывод в Linux (точные числа меняются от запуска к запуску):

```text
worker 0: 240 connections
worker 1: 243 connections
worker 2: 262 connections
worker 3: 255 connections
```

Ядро Linux вычисляет хеш **4-кортежа** каждого соединения и по нему выбирает один из слушающих
сокетов. Разные клиентские порты дают разные хеши, поэтому соединения распределяются равномерно. У
каждого рабочего потока своя очередь приёма, поэтому потоки никогда не борются за одну блокировку.

В macOS та же программа печатает `worker 3: 1000 connections` и ноль для остальных: macOS принимает
параметр, но не распределяет соединения, и один сокет получает их все. Это одна из причин, почему
направление работает в Linux.

> ⚠️ **Частая ошибка:** «поток на ядро» не становится быстрее автоматически. Если одно соединение
> намного загруженнее других, его поток перегружен, а остальные простаивают, потому что ничто не
> переносит работу между ними. Перехват работы лучше справляется с неравномерной нагрузкой. Измерьте
> обе архитектуры под *вашей* нагрузкой (Этап 19), прежде чем выбирать.

## 5. Ограничения, которые защищают сервер

Сервер под сильной нагрузкой ломается на своих пределах, поэтому нужно знать, где они:

| Предел | Что это | Что происходит на пределе | Как изменить |
|---|---|---|---|
| backlog | длина очереди приёма | новые соединения ждут или отбрасываются ядром | `listen(backlog)`; Linux ещё ограничивает её через `net.core.somaxconn` |
| файловые дескрипторы | каждый сокет — файловый дескриптор | `accept` падает с «Too many open files» | `ulimit -n` (на процесс), `LimitNOFILE=` в юнит-файле systemd (Урок 11.6) |
| буферы | память ядра на сокет | отправитель ждёт (управление потоком TCP) | `SO_RCVBUF` / `SO_SNDBUF`, ограничены `net.core.rmem_max` / `wmem_max` |
| ваш собственный лимит | число соединений, которое вы решили обслуживать | решаете вы: ждать или отказать | `Semaphore` в цикле приёма (упражнение 3) |

Ваш собственный лимит — самый важный, потому что только его *вы* контролируете в коде. Без него поток
соединений делает медленным каждого клиента; с ним сервер хорошо обслуживает фиксированное число
соединений, а остальные ждут в очереди приёма ядра. Это ожидание — **обратное давление**
(backpressure): медленная часть системы сдерживает быструю, вместо того чтобы рухнуть.

> 🔒 **Безопасность:** каждое открытое соединение стоит памяти и файлового дескриптора.
> Злоумышленник, который откроет тысячи соединений и ничего не будет отправлять, может их исчерпать.
> Лимит соединений плюс тайм-аут простоя (Урок 15.6) — базовая защита от этого.

> 🧪 **Тестирование:** код с сокетами тестируют на настоящих сокетах на `127.0.0.1` и порту `0`, чтобы
> тесты никогда не спорили за фиксированный порт и могли выполняться параллельно. Упражнения 2 и 3
> делают именно так.

## Частые ошибки

**Превращение блокирующего сокета в сокет Tokio.** Удалите строку `socket.set_nonblocking(true)?;`
из `make_listener` в `src/bin/listener.rs` и запустите снова. Tokio остановит программу:

```text
thread 'main' (75) panicked at src/bin/listener.rs:10:5:
Registering a blocking socket with the tokio runtime is unsupported. If you wish to do anyways, please add `--cfg tokio_allow_from_blocking_fd` to your RUSTFLAGS. See github.com/tokio-rs/tokio/issues/7172 for details.
```

Исправление: вызовите `socket.set_nonblocking(true)?` до `TcpListener::from_std` (или
`TcpStream::from_std`).

**Два слушающих сокета на одном порту без `SO_REUSEPORT`.** Сохраните это как `src/bin/twice.rs`:

```rust
use std::net::TcpListener;

fn main() -> std::io::Result<()> {
    let _first = TcpListener::bind("127.0.0.1:7304")?;
    let _second = TcpListener::bind("127.0.0.1:7304")?;
    Ok(())
}
```

`cargo run --bin twice` показывает, что второй `bind` падает:

```text
Error: Os { code: 98, kind: AddrInUse, message: "Address already in use" }
```

(В macOS код 48; вид ошибки тот же.) Исправление: задайте `SO_REUSEPORT` на *каждом* сокете и задайте
его *до* `bind`. Если задать после `bind`, ничего не изменится.

**Измерения в macOS и развёртывание в Linux.** Как показал раздел 4, поведение сокетов различается
между операционными системами. Измеряйте производительность в той операционной системе, в которую
развёртываете.

**Изменение размера буферов без измерений.** В Linux `set_recv_buffer_size(n)` даёт вам `2 * n` (ядро
удваивает значение для собственного учёта), но не больше предела `net.core.rmem_max`. Прочитайте
значение обратно и измерьте эффект; не стройте догадок.

## Попробуйте сами

1. В вашей ВМ `dev` запустите `cargo run --bin nagle` три раза. Разница в 40 мс стабильна?
2. Измените сервер в `nagle.rs` так, чтобы он собирал один буфер (`b"LEN 5\nhello"`) и вызывал
   `write_all` один раз. Запустите снова с `TCP_NODELAY = false`. Медленный случай исчезнет, потому что
   нет второй маленькой записи, которую алгоритм Нейгла мог бы придержать.
3. Измените `WORKERS` в `reuseport.rs` на 8, а `CLIENTS` на 4000. Соединения по-прежнему распределяются
   равномерно.

## Упражнения

### Упражнение 1 — Что вам дало ядро? (легко)

Напишите `src/bin/bufsize.rs`: создайте TCP-сокет с `socket2`, напечатайте размеры буферов приёма и
отправки по умолчанию, запросите буфер приёма в 1 МиБ (1 048 576 байтов) и напечатайте полученное
значение. Запустите в вашей ВМ.

<details>
<summary>Подсказка</summary>

Методы чтения — `recv_buffer_size()` и `send_buffer_size()`; метод записи —
`set_recv_buffer_size(bytes)`. Tokio здесь не нужен.

</details>

<details>
<summary>Ответ</summary>

```rust
use socket2::{Domain, Protocol, Socket, Type};

fn main() -> std::io::Result<()> {
    let socket = Socket::new(Domain::IPV4, Type::STREAM, Some(Protocol::TCP))?;
    println!("default SO_RCVBUF = {}", socket.recv_buffer_size()?);
    println!("default SO_SNDBUF = {}", socket.send_buffer_size()?);
    socket.set_recv_buffer_size(1024 * 1024)?;
    println!(
        "asked for 1048576, got SO_RCVBUF = {}",
        socket.recv_buffer_size()?
    );
    Ok(())
}
```

Вывод в Linux (у вас может отличаться):

```text
default SO_RCVBUF = 131072
default SO_SNDBUF = 16384
asked for 1048576, got SO_RCVBUF = 2097152
```

Почему это работает: Linux удваивает запрошенное значение. Если `net.core.rmem_max` меньше вашего
запроса (во многих системах Ubuntu он равен 212992), вы получите `2 * rmem_max`. Проверьте своё
значение командой `sysctl net.core.rmem_max`.

</details>

### Упражнение 2 — Протестируйте правило общего порта (средне)

Перенесите код слушающего сокета в `src/lib.rs` как
`pub fn listener(addr: SocketAddr, reuse_port: bool) -> std::io::Result<std::net::TcpListener>`.
Напишите модульный тест, который доказывает: два слушающих сокета с `reuse_port = true` могут занять
один порт, а второй сокет с `reuse_port = false` падает с `ErrorKind::AddrInUse`.

<details>
<summary>Подсказка</summary>

Привяжите первый сокет к `127.0.0.1:0`, затем прочитайте его настоящий адрес через `local_addr()` и
привяжите второй к этому адресу. `unwrap_err()` отдаёт ошибку неудачного `Result`.

</details>

<details>
<summary>Ответ</summary>

```rust
use socket2::{Domain, Protocol, Socket, Type};
use std::net::SocketAddr;

pub fn listener(addr: SocketAddr, reuse_port: bool) -> std::io::Result<std::net::TcpListener> {
    let socket = Socket::new(Domain::for_address(addr), Type::STREAM, Some(Protocol::TCP))?;
    socket.set_reuse_address(true)?;
    socket.set_reuse_port(reuse_port)?;
    socket.bind(&addr.into())?;
    socket.listen(1024)?;
    socket.set_nonblocking(true)?;
    Ok(socket.into())
}

#[cfg(test)]
mod tests {
    use super::*;
    use std::io::ErrorKind;

    #[test]
    fn two_listeners_share_a_port_only_with_reuse_port() {
        let first = listener("127.0.0.1:0".parse().unwrap(), true).unwrap();
        let addr = first.local_addr().unwrap();
        assert!(listener(addr, true).is_ok());

        let lonely = listener("127.0.0.1:0".parse().unwrap(), false).unwrap();
        let err = listener(lonely.local_addr().unwrap(), false).unwrap_err();
        assert_eq!(err.kind(), ErrorKind::AddrInUse);
    }
}
```

```text
test tests::two_listeners_share_a_port_only_with_reuse_port ... ok
```

Почему это работает: тест использует порт `0`, поэтому ядро выбирает свободный порт, и тест никогда не
столкнётся с другой программой. Оба правила проверяются на настоящих сокетах — это единственный
честный способ тестировать параметры сокетов.

</details>

### Упражнение 3 — Лимит соединений с обратным давлением (сложно)

Добавьте в `src/lib.rs`
`pub async fn serve_limited(listener: tokio::net::TcpListener, max: usize) -> std::io::Result<()>`:
эхо-сервер, который одновременно обслуживает не больше `max` соединений. Лишним соединениям не
отказывают; они ждут, пока освободится место. Напишите `#[tokio::test]` с `max = 2`, который
показывает, что третий клиент не получает эхо, пока подключены два клиента, и получает его после
того, как один из них отключится.

<details>
<summary>Подсказка</summary>

Используйте `Arc<tokio::sync::Semaphore>` с `max` разрешениями. Берите разрешение через
`acquire_owned().await` *до* `accept` и перемещайте его в задачу соединения: когда задача
заканчивается, разрешение освобождается, и место снова свободно. В тесте оберните третье эхо в
`tokio::time::timeout`.

</details>

<details>
<summary>Ответ</summary>

Добавьте это в `src/lib.rs` под кодом упражнения 2:

```rust
use std::sync::Arc;
use tokio::io::{AsyncReadExt, AsyncWriteExt};
use tokio::net::TcpListener;
use tokio::sync::Semaphore;

/// Echo server that serves at most `max` connections at the same time.
pub async fn serve_limited(listener: TcpListener, max: usize) -> std::io::Result<()> {
    let limit = Arc::new(Semaphore::new(max));
    loop {
        // Wait for a free place BEFORE accepting the next connection.
        let permit = Arc::clone(&limit).acquire_owned().await.unwrap();
        let (mut stream, _) = listener.accept().await?;
        tokio::spawn(async move {
            let _permit = permit; // released when this task ends
            let mut buf = [0u8; 1024];
            loop {
                match stream.read(&mut buf).await {
                    Ok(0) | Err(_) => return,
                    Ok(n) => {
                        if stream.write_all(&buf[..n]).await.is_err() {
                            return;
                        }
                    }
                }
            }
        });
    }
}

#[cfg(test)]
mod limit_tests {
    use super::*;
    use std::time::Duration;
    use tokio::net::TcpStream;
    use tokio::time::timeout;

    async fn echo_once(stream: &mut TcpStream) -> std::io::Result<()> {
        stream.write_all(b"hi").await?;
        let mut buf = [0u8; 2];
        stream.read_exact(&mut buf).await?;
        Ok(())
    }

    #[tokio::test]
    async fn third_client_waits_until_a_place_is_free() {
        let l = TcpListener::bind("127.0.0.1:0").await.unwrap();
        let addr = l.local_addr().unwrap();
        tokio::spawn(serve_limited(l, 2));

        let mut a = TcpStream::connect(addr).await.unwrap();
        let mut b = TcpStream::connect(addr).await.unwrap();
        echo_once(&mut a).await.unwrap();
        echo_once(&mut b).await.unwrap();

        let mut c = TcpStream::connect(addr).await.unwrap();
        let waited = timeout(Duration::from_millis(200), echo_once(&mut c)).await;
        assert!(waited.is_err(), "the third client must wait");

        drop(a); // free one place
        timeout(Duration::from_secs(2), echo_once(&mut c))
            .await
            .expect("the third client is served after a place is free")
            .unwrap();
    }
}
```

```text
test limit_tests::third_client_waits_until_a_place_is_free ... ok
```

Почему это работает: третий `connect` проходит, потому что ядро завершает TCP-рукопожатие и кладёт
соединение в очередь приёма. Но сервер не вызывает `accept`, пока нет свободного разрешения, поэтому
байты третьего клиента никто не читает. Когда `a` удаляется, его задача видит конец файла,
завершается и освобождает разрешение; цикл приёма просыпается и обслуживает `c`. Это обратное давление
примерно в десять строк.

</details>

## Вопросы для самопроверки

1. Почему некоторые параметры сокета можно задать только через `socket2`, а не после
   `TcpListener::bind`?

<details>
<summary>Ответ</summary>

Параметры вроде `SO_REUSEPORT` нужно задавать *до* `bind`. `TcpListener::bind` создаёт сокет,
привязывает его и начинает слушать одним вызовом, поэтому момента для их установки нет. `socket2`
позволяет сделать каждый системный вызов самостоятельно.

</details>

2. Какие два правила TCP вместе вызывают задержку в 40 мс и чего ждёт каждое из них?

<details>
<summary>Ответ</summary>

Алгоритм Нейгла у отправителя придерживает маленькую запись, пока предыдущие данные не подтверждены.
Отложенный ACK у получателя придерживает подтверждение, надеясь приложить его к ответу. Каждый ждёт
другого, пока не сработает таймер отложенного ACK (около 40 мс в Linux).

</details>

3. Назовите два способа избежать задержки.

<details>
<summary>Ответ</summary>

Задать `TCP_NODELAY` на соединении и записывать весь ответ одним `write_all` (или одной векторной
записью) вместо нескольких маленьких.

</details>

4. Как Linux выбирает, какой из нескольких слушающих сокетов с `SO_REUSEPORT` получит новое
   соединение?

<details>
<summary>Ответ</summary>

Ядро хеширует 4-кортеж соединения (адрес и порт источника, адрес и порт назначения) и по хешу выбирает
один из слушающих сокетов. Разные соединения распределяются примерно равномерно.

</details>

5. Когда рантайм с перехватом работы лучше, чем «поток на ядро»?

<details>
<summary>Ответ</summary>

Когда нагрузка неравномерна: некоторые соединения намного загруженнее других. Перехват работы
переносит задачи на свободные потоки; «поток на ядро» держит каждое соединение в своём потоке, поэтому
одно горячее соединение может перегрузить одно ядро, пока другие простаивают.

</details>

## Итоги

- Сетевое направление посвящено программам плоскости данных: прокси, балансировщикам нагрузки,
  шлюзам. Они должны быть быстрыми и выдерживать любой ввод.
- `socket2` позволяет создавать сокет пошагово и задавать параметры до `bind`; `SockRef` читает и
  задаёт параметры у любого существующего сокета.
- Tokio нужны неблокирующие сокеты: вызывайте `set_nonblocking(true)` до `from_std`.
- Алгоритм Нейгла вместе с отложенным ACK может добавить около 40 мс к каждому запросу; задавайте
  `TCP_NODELAY` и записывайте каждый ответ за один раз.
- `SO_REUSEPORT` позволяет нескольким сокетам слушать один порт; в Linux ядро распределяет соединения
  по хешу 4-кортежа, что делает возможными серверы «поток на ядро». macOS их не распределяет.
- Знайте свои пределы: backlog, файловые дескрипторы, буферы и ваш собственный лимит соединений,
  который даёт обратное давление вместо обрушения.

## Источники и что почитать

- [socket2 documentation](https://docs.rs/socket2/0.6/socket2/) — все параметры сокетов, которые предлагает крейт, с платформами, где работает каждый.
- [tokio::net::TcpListener](https://docs.rs/tokio/1/tokio/net/struct.TcpListener.html) — `from_std`, `bind` и `accept` в Tokio.
- [tokio::runtime::Builder](https://docs.rs/tokio/1/tokio/runtime/struct.Builder.html) — многопоточный рантайм и рантайм текущего потока.
- [socket(7) — Linux manual page](https://man7.org/linux/man-pages/man7/socket.7.html) — `SO_REUSEADDR`, `SO_REUSEPORT`, `SO_RCVBUF` и другие общие параметры сокетов.
- [tcp(7) — Linux manual page](https://man7.org/linux/man-pages/man7/tcp.7.html) — `TCP_NODELAY`, `TCP_QUICKACK` и настройки TCP в `/proc/sys/net/ipv4/`.
- [RFC 896 — Congestion Control in IP/TCP Internetworks](https://www.rfc-editor.org/rfc/rfc896) — первоначальное описание алгоритма Нейгла (RFC, Request for Comments, «запрос комментариев» — документ из официальной серии интернет-стандартов).

---

[🏠 Начало](../START-HERE.md) · [← 22.17 Трек баз данных: проект и следующие шаги](17-databases-track-project-and-next-steps.md) · [📋 Этап 22](README.md) · [22.19 Трек сетей: разбор протоколов без копирования →](19-networking-track-zero-copy-parsing.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
