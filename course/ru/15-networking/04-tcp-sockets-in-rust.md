<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 15.3 TCP и UDP](03-tcp-and-udp.md) · [📋 Этап 15](README.md) · [15.5 UDP-сокеты в Rust →](05-udp-sockets-in-rust.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/15-networking/04-tcp-sockets-in-rust.md) <!-- nav -->

# Урок 15.4 — TCP-сокеты в Rust

> **Этап 15 — Сети** · Урок 4 из 16 · ⏱ около 135 минут

## Что вы узнаете

- Какие системные вызовы стоят за `TcpListener` и `TcpStream` и чем слушающий сокет отличается от подключённого.
- Как написать сервер, который одновременно обслуживает много клиентов, по одному потоку на соединение.
- Почему одна запись `write` не означает одно чтение `read`, как распознать конец потока и как закрыть только одно направление с помощью `shutdown`.
- Как использовать тайм-ауты, порт `0` и правильный адрес привязки и что означают частые ошибки сокетов.
- Как тестировать сетевой код на настоящих сокетах по адресу `127.0.0.1:0`.

## Перед началом

- Вы прошли [Урок 15.3 — TCP и UDP](03-tcp-and-udp.md).
- Вы знаете системные вызовы и файловые дескрипторы из [Урока 13.3](../13-operating-systems/03-system-calls.md) и [Урока 13.4](../13-operating-systems/04-files-and-file-descriptors.md), потоки — из Этапов 06 и 14, а атомарные операции — из [Урока 14.5](../14-concurrency-and-async/05-atomics.md) (только для последнего упражнения).
- Примеры кладутся в `code/p15-net/src/bin/`. Вам понадобятся два терминала: один для сервера, другой для клиента.

## Новые слова

| Слово | Значение |
|---|---|
| **слушающий сокет** (listening socket) | Сокет, который только принимает новые соединения. В Rust это `TcpListener`. |
| **подключённый сокет** (connected socket) | Сокет одного соединения, через который идут данные. В Rust это `TcpStream`. |
| **привязать** (bind) | Прикрепить сокет к локальному адресу и порту. |
| **принять** (accept) | Взять следующее готовое соединение из очереди ожидания слушающего сокета. |
| **очередь ожидания** (backlog) | Очередь ядра с соединениями, которые завершили рукопожатие, но ещё не приняты. |
| **EOF** (End Of File, конец файла) | Конец потока. Для сокета: другая сторона закрыла своё направление отправки. Тогда `read` возвращает `Ok(0)`. |
| **полузакрытие** (half-close) | Закрытие только одного направления соединения через `shutdown(Shutdown::Write)`: «я больше ничего не отправлю, но ещё читаю». |
| **эхо-сервер** (echo server) | Сервер, который отправляет обратно каждый полученный байт. «Hello world» сетевого программирования. |
| **тайм-аут** (timeout) | Предел времени, после которого ожидающая операция сдаётся с ошибкой. |
| **алгоритм Нейгла** (Nagle's algorithm) | Возможность TCP ненадолго придерживать маленькие записи, чтобы отправить их вместе меньшим числом пакетов. `set_nodelay(true)` его отключает. |
| **TCP / UDP** (Transmission Control Protocol / User Datagram Protocol — протокол управления передачей / протокол пользовательских датаграмм) | Протоколы потока байтов и датаграмм из Урока 15.3. Этот урок использует TCP, Урок 15.5 — UDP. |
| **FIN / RST** (finish / reset, завершение / сброс) | Флаги TCP из Урока 15.3: FIN говорит «я больше ничего не отправляю» (обычное закрытие), RST немедленно обрывает соединение. |

## 1. Что `std::net` делает внутри

Из Урока 06.11 вы знаете `TcpListener` и `TcpStream`. Каждый их метод — тонкая обёртка над одним или
двумя системными вызовами (Урок 13.3). Сокет — это файловый дескриптор (Урок 13.4), поэтому после
создания с ним работают обычными `read`, `write` и `close`:

| Rust | Системные вызовы | Что делает ядро |
|---|---|---|
| `TcpListener::bind(addr)` | `socket`, `bind`, `listen` | создаёт сокет, прикрепляет его к адресу, начинает принимать рукопожатия в очередь ожидания |
| `listener.accept()` / `incoming()` | `accept` | берёт одно готовое соединение и возвращает для него **новый** сокет |
| `TcpStream::connect(addr)` | `socket`, `connect` | создаёт сокет и выполняет тройное рукопожатие |
| `stream.read(&mut buf)` | `read` (или `recv`) | копирует пришедшие байты из буфера ядра в `buf`; ждёт, если их нет |
| `stream.write(&buf)` | `write` (или `send`) | копирует байты в буфер отправки ядра; TCP отправляет их, когда может |
| `stream.shutdown(Write)` | `shutdown` | отправляет FIN, но оставляет сокет открытым для чтения |
| `drop(stream)` | `close` | закрывает сокет: FIN (или RST, если остались непрочитанные данные) |

Самая важная строка — `accept`. У сервера **один** слушающий сокет, а `accept` создаёт **отдельный
подключённый сокет для каждого клиента**:

```text
                   ┌────────────────────┐
  clients ──SYN──► │ listening socket   │  127.0.0.1:7000, state LISTEN
                   │ backlog: [c3] [c4] │  (handshakes done, not accepted yet)
                   └─────────┬──────────┘
                     accept()│ returns a new socket each time
            ┌────────────────┼────────────────┐
            ▼                ▼                ▼
     TcpStream c1      TcpStream c2      (c3, c4 wait in the backlog)
     ESTABLISHED       ESTABLISHED
```

Каждый вызов здесь по умолчанию *блокирует* (ждёт): `accept` ждёт клиента, `read` ждёт данных, а
`write` ждёт, когда буфер отправки полон. С потоками заблокированный поток — это нормально: другие
потоки продолжают работать. Урок 15.6 показывает другой путь — асинхронный ввод-вывод с Tokio.

## 2. Эхо-сервер для многих клиентов

Сохраните как `src/bin/echo_server.rs`:

```rust
use std::io::{Read, Write};
use std::net::{TcpListener, TcpStream};
use std::thread;

/// Sends every byte it receives back to the client, until the client closes.
fn handle(mut stream: TcpStream) -> std::io::Result<()> {
    let peer = stream.peer_addr()?;
    let mut buf = [0u8; 1024];
    loop {
        let n = stream.read(&mut buf)?;
        if n == 0 {
            println!("{peer}: closed");
            return Ok(());
        }
        println!("{peer}: {n} bytes");
        stream.write_all(&buf[..n])?;
    }
}

fn main() -> std::io::Result<()> {
    let listener = TcpListener::bind("127.0.0.1:7000")?;
    println!("echo server on {}", listener.local_addr()?);
    for stream in listener.incoming() {
        let stream = stream?;
        thread::spawn(move || {
            if let Err(e) = handle(stream) {
                eprintln!("connection error: {e}");
            }
        });
    }
    Ok(())
}

#[cfg(test)]
mod tests {
    use super::*;
    use std::net::Shutdown;

    #[test]
    fn echoes_bytes_back() {
        let listener = TcpListener::bind("127.0.0.1:0").unwrap();
        let addr = listener.local_addr().unwrap();
        thread::spawn(move || {
            let (stream, _) = listener.accept().unwrap();
            handle(stream).unwrap();
        });

        let mut client = TcpStream::connect(addr).unwrap();
        client.write_all(b"ping").unwrap();
        client.shutdown(Shutdown::Write).unwrap(); // «я закончил отправку»
        let mut reply = Vec::new();
        client.read_to_end(&mut reply).unwrap();
        assert_eq!(reply, b"ping");
    }
}
```

**Что происходит:**

1. `main` привязывает слушающий сокет и в цикле проходит по `incoming()`, который снова и снова
   вызывает `accept`. Каждый принятый `TcpStream` перемещается в собственный поток, поэтому медленный
   клиент не блокирует остальных.
2. `handle` читает до 1024 байтов. `read` возвращает, сколько байтов он положил в `buf`, и их может
   быть меньше 1024. **`Ok(0)` означает EOF**: клиент закрыл своё направление отправки, и данных больше
   никогда не будет. Это обычный способ завершить цикл по сокету.
3. `write_all` отправляет те же байты обратно. Внутри он повторяет запись, пока не запишет каждый
   байт, потому что отдельный `write` может принять только часть буфера.
4. Ошибки одного соединения печатаются через `eprintln!` и завершают только этот поток, а не сервер.

Запустите сервер в одном терминале командой `cargo run --bin echo_server`. Во втором терминале
подключитесь командой `nc 127.0.0.1 7000`, введите `hello` и Enter, затем `how are you?` и Enter
(каждая строка возвращается обратно) и нажмите Ctrl+C. Сервер печатает (номер порта у вас будет
другим):

```text
echo server on 127.0.0.1:7000
127.0.0.1:53912: 6 bytes
127.0.0.1:53912: 13 bytes
127.0.0.1:53912: closed
```

Откройте два окна `nc` одновременно: обслуживаются оба, каждое своим потоком.

> 🧪 **Тестирование:** тест привязывается к `127.0.0.1:0`, поэтому операционная система выбирает
> свободный порт, и тесты могут идти параллельно, не споря за порт 7000. Тест запускает настоящий
> `handle` в потоке и общается с ним через настоящий сокет. `shutdown(Shutdown::Write)` отправляет FIN,
> поэтому `handle` видит EOF и завершается; затем и `read_to_end` на клиенте видит EOF. Запустите
> командой `cargo test --bin echo_server`: `test result: ok. 1 passed`.

## 3. У потока нет границ сообщений

Урок 15.3 сказал, что TCP сохраняет байты, но не границы сообщений. Вот это в коде. Программа
самодостаточна: поток-клиент пишет, а главный поток читает. Сохраните как `src/bin/chunks.rs`:

```rust
use std::io::{Read, Write};
use std::net::{TcpListener, TcpStream};
use std::thread;
use std::time::Duration;

fn main() -> std::io::Result<()> {
    let listener = TcpListener::bind("127.0.0.1:0")?;
    let addr = listener.local_addr()?;

    let client = thread::spawn(move || -> std::io::Result<()> {
        let mut stream = TcpStream::connect(addr)?;
        stream.write_all(b"hel")?;
        stream.write_all(b"lo\nwor")?;
        thread::sleep(Duration::from_millis(100));
        stream.write_all(b"ld\n")?;
        thread::sleep(Duration::from_millis(100));
        stream.write_all(&vec![b'x'; 300_000])?; // одна большая запись
        Ok(()) // здесь поток уничтожается: клиент закрывает соединение
    });

    let (mut stream, _) = listener.accept()?;
    thread::sleep(Duration::from_millis(50)); // даём первым двум записям дойти
    let mut buf = [0u8; 65536];
    let mut total = 0;
    loop {
        let n = stream.read(&mut buf)?;
        if n == 0 {
            break;
        }
        total += n;
        if n < 16 {
            println!(
                "read {n:>5} bytes: {:?}",
                String::from_utf8_lossy(&buf[..n])
            );
        } else {
            println!("read {n:>5} bytes");
        }
    }
    println!("end of stream after {total} bytes");
    client.join().unwrap()
}
```

`cargo run --bin chunks` напечатал на Mac вот что (размеры больших чтений меняются от запуска к
запуску):

```text
read     9 bytes: "hello\nwor"
read     3 bytes: "ld\n"
read 65328 bytes
read 65536 bytes
read 32456 bytes
read 65536 bytes
read 65536 bytes
read  5608 bytes
end of stream after 300012 bytes
```

**Что происходит:**

1. Записи `hel` и `lo\nwor` пришли до первого `read`, поэтому **две записи вышли одним чтением**, и
   это чтение обрывается посреди второй строки (`wor`).
2. Один `write_all` на 300 000 байтов вышел **шестью чтениями** разного размера — в зависимости от
   того, как быстро наполнялись и освобождались буферы ядра.
3. Итог точен: 3 + 6 + 3 + 300 000 = 300 012 байтов, по порядку. TCP сохраняет байты; границы
   сообщений ваша программа должна находить сама. Именно об этом Урок 15.7.

Для текстовых протоколов с одним сообщением на строку работу делает `BufReader`: `read_line` и
`lines()` читают столько кусков, сколько нужно, и отдают вам целые строки. Его использует
Упражнение 2.

## 4. Закрытие, тайм-ауты и ошибки

**Закрытие.** Уничтожение `TcpStream` закрывает оба направления. Иногда клиент должен сказать «мой
запрос закончен» и *после этого* прочитать ответ. Это делает `stream.shutdown(Shutdown::Write)`: он
отправляет FIN (у сервера `read` возвращает 0), но клиент всё ещё может читать. Его использует тест
из раздела 2.

**Тайм-ауты.** По умолчанию `read` ждёт бесконечно. Клиент, который никогда ничего не отправит,
навсегда занял бы поток сервера. Установите предел. Сохраните как `src/bin/timeouts.rs`:

```rust
use std::io::Read;
use std::net::{TcpListener, TcpStream};
use std::time::{Duration, Instant};

fn main() -> std::io::Result<()> {
    let listener = TcpListener::bind("127.0.0.1:0")?;
    let mut stream = TcpStream::connect(listener.local_addr()?)?;
    stream.set_read_timeout(Some(Duration::from_millis(200)))?;

    let start = Instant::now();
    let mut buf = [0u8; 16];
    match stream.read(&mut buf) {
        Ok(n) => println!("read {n} bytes"),
        Err(e) => println!("after {:?}: {e} (kind: {:?})", start.elapsed(), e.kind()),
    }

    // Второй слушающий сокет на том же адресе не создаётся.
    match TcpListener::bind(listener.local_addr()?) {
        Ok(_) => println!("bound twice?"),
        Err(e) => println!("second bind: {e} (kind: {:?})", e.kind()),
    }
    Ok(())
}
```

```text
after 201.028042ms: Resource temporarily unavailable (os error 35) (kind: WouldBlock)
second bind: Address already in use (os error 48) (kind: AddrInUse)
```

**Что происходит:**

1. В поток никто не пишет, поэтому `read` ждёт и через 200 мс возвращает ошибку. В Unix её вид —
   `WouldBlock` (в Windows был бы `TimedOut`), поэтому проверяйте оба.
2. `set_write_timeout` делает то же для записи, а `TcpStream::connect_timeout(&addr, duration)`
   ограничивает рукопожатие.
3. Слушать адрес и порт может только один сокет, поэтому второй `bind` завершается ошибкой
   `AddrInUse`. Числа в `os error 35` и `os error 48` — коды ошибок macOS; в Linux те же ошибки —
   `os error 11` и `os error 98`.

Ошибки, которые встречаются чаще всего:

| `ErrorKind` | Обычное значение |
|---|---|
| `ConnectionRefused` | этот адрес и порт никто не слушает (RST, Урок 15.3) |
| `AddrInUse` | другая программа (или старая копия вашей) уже слушает этот порт |
| `WouldBlock` / `TimedOut` | истёк установленный вами тайм-аут |
| `ConnectionReset` | другая сторона прервала соединение пакетом RST |
| `BrokenPipe` | вы пишете в соединение, которое другая сторона уже закрыла |
| `UnexpectedEof` | `read_exact` нужно было больше байтов, но поток закончился |

**Адрес привязки.** `127.0.0.1:7000` принимает клиентов только с той же машины. `0.0.0.0:7000`
принимает соединения на всех интерфейсах, так что подключиться могут и другие компьютеры сети. Внутри
контейнера сервер должен привязываться к `0.0.0.0`, иначе опубликованный порт до него не достанет.

> 🔒 **Безопасность:** сервер на `0.0.0.0` доступен всем, кто может достучаться до вашей машины. Во
> время разработки привязывайтесь к `127.0.0.1`. Всегда устанавливайте тайм-ауты чтения и предел числа
> соединений: без них несколько клиентов, которые подключились и ничего не отправляют, могут занять
> все ваши потоки. Такой предел строит Упражнение 3.

> ⚡ **Производительность:** **алгоритм Нейгла** в TCP немного задерживает маленькие записи, чтобы
> отправить их вместе. Для протоколов «запрос — ответ» с крошечными сообщениями это может добавить
> десятки миллисекунд задержки. `stream.set_nodelay(true)` его отключает. Ещё лучше собрать всё
> сообщение в буфер (или обернуть поток в `BufWriter`) и записать его за один раз.

## Частые ошибки

> ⚠️ **Частая ошибка: игнорировать значение, которое вернул `read`.** `stream.read(&mut buf)?;`, после
> которого используется весь `buf`, принимает старые или нулевые байты за данные. Всегда используйте
> `&buf[..n]` и считайте `n == 0` концом потока. `cargo clippy` сообщает о `read`, результат которого
> выброшен, как об ошибке:
>
> ```text
> error: read amount is not handled
>   = help: use `Read::read_exact` instead, or handle partial reads
> ```

> ⚠️ **Частая ошибка: `write` вместо `write_all`.** `write` может записать только часть буфера и
> возвращает, сколько записал. Код, который это игнорирует, теряет данные под нагрузкой. Используйте
> `write_all`.

> ⚠️ **Частая ошибка: обе стороны ждут друг друга.** Клиент отправляет запрос и вызывает
> `read_to_end`, чтобы получить весь ответ, а сервер ждёт EOF, чтобы понять, что запрос закончен. Оба
> ждут вечно. Либо клиент вызывает `shutdown(Shutdown::Write)` после отправки, либо протокол отмечает,
> где кончается сообщение (перевод строки, длина; Урок 15.7).

> ⚠️ **Частая ошибка: забыть сбросить `BufWriter`.** Данные, записанные в `BufWriter<TcpStream>`,
> остаются в памяти, пока буфер не заполнится, не будет вызван `flush()` или писатель не будет
> уничтожен. Клиент, который ждёт ответа на несброшенный запрос, ждёт вечно.

## Попробуйте сами

1. Запустите `echo_server` и подключитесь двумя окнами `nc` одновременно. Пишите в оба и смотрите,
   как в журнале сервера появляются два разных клиентских порта.
2. Остановите сервер через Ctrl+C и сразу запустите снова. Он запускается без `AddrInUse`, потому что
   `TcpListener::bind` в Rust в Unix устанавливает параметр `SO_REUSEADDR`, который разрешает привязку,
   пока старые соединения ещё в TIME-WAIT.
3. Запустите сервер дважды в двух терминалах. Второй завершится ошибкой
   `Error: Os { code: 48, kind: AddrInUse, message: "Address already in use" }`.
4. Запустите `cargo run --bin chunks` три раза и сравните размеры больших чтений.

## Упражнения

### Упражнение 1 — Посчитайте байты (легко)

Измените `handle` в `echo_server.rs` так, чтобы при закрытии клиентом он печатал, сколько всего
байтов прошло через соединение, например `127.0.0.1:53912: closed after 19 bytes`.

<details>
<summary>Подсказка</summary>

Заведите `let mut total = 0;` перед циклом и прибавляйте `n` после каждого чтения.

</details>

<details>
<summary>Ответ</summary>

Остальная часть `echo_server.rs` не меняется:

```rust
fn handle(mut stream: TcpStream) -> std::io::Result<()> {
    let peer = stream.peer_addr()?;
    let mut buf = [0u8; 1024];
    let mut total = 0;
    loop {
        let n = stream.read(&mut buf)?;
        if n == 0 {
            println!("{peer}: closed after {total} bytes");
            return Ok(());
        }
        total += n;
        stream.write_all(&buf[..n])?;
    }
}
```

С двумя строками из раздела 2 (`hello` с переводом строки — 6 байтов, `how are you?` с переводом
строки — 13 байтов) сервер печатает `closed after 19 bytes`.

</details>

### Упражнение 2 — Построчный сервер команд с тестами (средне)

Напишите `src/bin/cmd_server.rs` — сервер на `127.0.0.1:7001`, который читает по одной команде в
строке и отвечает одной строкой: `PING` → `PONG`, `ADD 2 40` → `42` (используйте `i64`, а
переполнение сообщайте как `ERR overflow`), плохое число → `ERR ADD needs two integers`, всё
остальное → `ERR unknown command`. Вынесите протокол в чистую функцию
`fn respond(line: &str) -> String` и тестируйте её напрямую. Добавьте один тест, который общается с
`handle` через настоящий сокет на порту 0.

<details>
<summary>Подсказка</summary>

Для чтения строк оберните поток в `BufReader`; для записи нужен второй дескриптор того же сокета:
`stream.try_clone()`. `reader.lines()` заканчивается, когда клиент закрывает соединение. В тесте с
сокетом отправьте обе команды, вызовите `shutdown(Shutdown::Write)` и соберите строки ответа.

</details>

<details>
<summary>Ответ</summary>

```rust
use std::io::{BufRead, BufReader, Write};
use std::net::{TcpListener, TcpStream};
use std::thread;

/// The protocol: one command per line, one answer per line.
fn respond(line: &str) -> String {
    let mut parts = line.split_whitespace();
    match (parts.next(), parts.next(), parts.next()) {
        (Some("PING"), None, None) => "PONG".to_string(),
        (Some("ADD"), Some(a), Some(b)) => match (a.parse::<i64>(), b.parse::<i64>()) {
            (Ok(a), Ok(b)) => a
                .checked_add(b)
                .map_or("ERR overflow".into(), |s| s.to_string()),
            _ => "ERR ADD needs two integers".to_string(),
        },
        _ => "ERR unknown command".to_string(),
    }
}

fn handle(stream: TcpStream) -> std::io::Result<()> {
    let mut writer = stream.try_clone()?; // второй дескриптор того же сокета
    let reader = BufReader::new(stream);
    for line in reader.lines() {
        let answer = respond(&line?);
        writer.write_all(answer.as_bytes())?;
        writer.write_all(b"\n")?;
    }
    Ok(()) // lines() закончился: клиент закрыл соединение
}

fn main() -> std::io::Result<()> {
    let listener = TcpListener::bind("127.0.0.1:7001")?;
    println!("command server on {}", listener.local_addr()?);
    for stream in listener.incoming() {
        let stream = stream?;
        thread::spawn(move || handle(stream));
    }
    Ok(())
}

#[cfg(test)]
mod tests {
    use super::*;
    use std::net::Shutdown;

    #[test]
    fn answers_commands() {
        assert_eq!(respond("PING"), "PONG");
        assert_eq!(respond("ADD 2 40"), "42");
        assert_eq!(respond("ADD 2 x"), "ERR ADD needs two integers");
        assert_eq!(respond("ADD 9223372036854775807 1"), "ERR overflow");
        assert_eq!(respond("DELETE everything"), "ERR unknown command");
    }

    #[test]
    fn works_over_a_real_socket() {
        let listener = TcpListener::bind("127.0.0.1:0").unwrap();
        let addr = listener.local_addr().unwrap();
        thread::spawn(move || {
            let (stream, _) = listener.accept().unwrap();
            handle(stream).unwrap();
        });

        let mut client = TcpStream::connect(addr).unwrap();
        client.write_all(b"PING\nADD 1 2\n").unwrap();
        client.shutdown(Shutdown::Write).unwrap();
        let answers: Vec<String> = BufReader::new(client).lines().map(|l| l.unwrap()).collect();
        assert_eq!(answers, ["PONG", "3"]);
    }
}
```

`cargo test --bin cmd_server` сообщает `2 passed`. Если сервер запущен, ввод `PING`, `ADD 2 40` и
`HELLO` в `nc 127.0.0.1 7001` возвращает `PONG`, `42` и `ERR unknown command`.

Почему это работает: `BufReader` собирает куски, пока у него не будет целой строки, поэтому код
протокола никогда не видит половину команды. Протокол в чистой функции делает бо́льшую часть тестов
быстрыми и независимыми от сети; один тест с сокетом доказывает, что части стыкуются.

</details>

### Упражнение 3 — Не больше N клиентов (сложно)

Напишите `src/bin/limited.rs`: эхо-сервер, который одновременно обслуживает не больше 2 клиентов.
Третий клиент получает строку `BUSY\n` и сразу отключается. Когда клиент уходит, его место
освобождается. Напишите тест: двух клиентов обслуживают, третий читает `BUSY\n`, а после отключения
одного из первых двух новый клиент снова обслуживается.

<details>
<summary>Подсказка</summary>

Считайте активные соединения с помощью `Arc<AtomicUsize>`. Увеличивайте счётчик в цикле `accept` (не
в потоке, иначе несколько потоков могут пройти проверку одновременно). Чтобы надёжно уменьшать его,
даже когда `handle` возвращает ошибку, поместите уменьшение в `Drop` маленькой структуры-охранника,
которой владеет поток.

</details>

<details>
<summary>Ответ</summary>

```rust
use std::io::{Read, Write};
use std::net::{TcpListener, TcpStream};
use std::sync::Arc;
use std::sync::atomic::{AtomicUsize, Ordering};
use std::thread;

/// Lowers the counter when it is dropped, even if the handler returns early with an error.
struct Slot(Arc<AtomicUsize>);

impl Drop for Slot {
    fn drop(&mut self) {
        self.0.fetch_sub(1, Ordering::SeqCst);
    }
}

fn handle(mut stream: TcpStream) -> std::io::Result<()> {
    let mut buf = [0u8; 1024];
    loop {
        let n = stream.read(&mut buf)?;
        if n == 0 {
            return Ok(());
        }
        stream.write_all(&buf[..n])?;
    }
}

fn serve(listener: TcpListener, max: usize) -> std::io::Result<()> {
    let active = Arc::new(AtomicUsize::new(0));
    for stream in listener.incoming() {
        let mut stream = stream?;
        if active.fetch_add(1, Ordering::SeqCst) >= max {
            active.fetch_sub(1, Ordering::SeqCst);
            let _ = stream.write_all(b"BUSY\n");
            continue; // поток уничтожается: соединение закрывается
        }
        let slot = Slot(Arc::clone(&active));
        thread::spawn(move || {
            let _slot = slot; // живёт, пока обработчик не закончит
            let _ = handle(stream);
        });
    }
    Ok(())
}

fn main() -> std::io::Result<()> {
    let listener = TcpListener::bind("127.0.0.1:7002")?;
    println!(
        "echo server (at most 2 clients) on {}",
        listener.local_addr()?
    );
    serve(listener, 2)
}

#[cfg(test)]
mod tests {
    use super::*;

    fn echo_once(stream: &mut TcpStream) -> Vec<u8> {
        stream.write_all(b"hi").unwrap();
        let mut buf = [0u8; 2];
        stream.read_exact(&mut buf).unwrap();
        buf.to_vec()
    }

    #[test]
    fn third_client_is_turned_away() {
        let listener = TcpListener::bind("127.0.0.1:0").unwrap();
        let addr = listener.local_addr().unwrap();
        thread::spawn(move || serve(listener, 2));

        let mut a = TcpStream::connect(addr).unwrap();
        let mut b = TcpStream::connect(addr).unwrap();
        assert_eq!(echo_once(&mut a), b"hi"); // обоих действительно обслуживают
        assert_eq!(echo_once(&mut b), b"hi");

        let mut c = TcpStream::connect(addr).unwrap();
        let mut answer = String::new();
        c.read_to_string(&mut answer).unwrap();
        assert_eq!(answer, "BUSY\n");

        drop(a); // место освобождается ...
        thread::sleep(std::time::Duration::from_millis(100));
        let mut d = TcpStream::connect(addr).unwrap();
        assert_eq!(echo_once(&mut d), b"hi"); // ... и его получает новый клиент
    }
}
```

`cargo test --bin limited` сообщает `1 passed` (тест прошёл пять запусков подряд).

Почему это работает: `fetch_add` возвращает старое значение, поэтому проверка и увеличение — один
атомарный шаг, и делает его только цикл `accept`. Охранник `Slot` освобождает место в `Drop`, который
выполняется, как бы ни закончился поток. Короткая пауза в тесте даёт потоку сервера время увидеть EOF
от `a`. Проекту балансировщика нагрузки (Урок 15.15) нужен именно такой предел соединений.

</details>

## Вопросы для самопроверки

1. Какие системные вызовы делает `TcpListener::bind` и что возвращает `accept`?

<details>
<summary>Ответ</summary>

`socket`, `bind` и `listen`. `accept` возвращает **новый** подключённый сокет (`TcpStream`) для одного
клиента и адрес клиента; слушающий сокет остаётся открытым для следующих клиентов.

</details>

2. Что означает `Ok(0)` от `stream.read`?

<details>
<summary>Ответ</summary>

Конец потока (EOF): другая сторона закрыла своё направление отправки, и байтов больше не будет.

</details>

3. Клиент записывает 100 байтов одним `write_all`. Сколько вызовов `read` понадобится серверу?

<details>
<summary>Ответ</summary>

Любое число от одного и больше. TCP может доставить байты одним куском или несколькими, а одно
чтение может захватить и байты следующего сообщения. Сервер не должен на это полагаться.

</details>

4. Почему тесты привязываются к `127.0.0.1:0`, а не к фиксированному порту?

<details>
<summary>Ответ</summary>

Порт 0 заставляет операционную систему выбрать свободный порт. Параллельные тесты (или другая
программа) не могут столкнуться на одном порту, а `local_addr()` говорит тесту, куда подключаться.

</details>

5. Чем уничтожение `TcpStream` отличается от вызова `shutdown(Shutdown::Write)`?

<details>
<summary>Ответ</summary>

Уничтожение закрывает весь сокет: ни чтения, ни записи. `shutdown(Write)` отправляет FIN только для
направления отправки; программа всё ещё может прочитать ответ.

</details>

6. Ваш сервер работает в контейнере Docker и привязан к `127.0.0.1:8080`. Порт опубликован через
   `-p 8080:8080`, но `curl` с Mac не может подключиться. Почему?

<details>
<summary>Ответ</summary>

Внутри контейнера `127.0.0.1` — собственный петлевой интерфейс контейнера. Соединения с
опубликованного порта приходят на адрес `eth0` контейнера, где никто не слушает. Сервер должен
привязываться к `0.0.0.0:8080`.

</details>

## Итоги

- `TcpListener::bind` = `socket` + `bind` + `listen`; `accept` возвращает новый `TcpStream` на каждого
  клиента; `TcpStream::connect` выполняет рукопожатие.
- Один поток на соединение — самый простой способ обслуживать многих клиентов; всегда обрабатывайте
  ошибки отдельно для каждого соединения.
- `read` возвращает число полученных байтов; `Ok(0)` — это EOF. Используйте `&buf[..n]`, `write_all` и
  `BufReader` для строк. Записи и чтения не соответствуют друг другу один к одному.
- `shutdown(Shutdown::Write)` выполняет полузакрытие; `set_read_timeout`, `set_write_timeout` и
  `connect_timeout` прекращают бесконечное ожидание.
- Привязывайтесь к `127.0.0.1` для локальной работы и к `0.0.0.0` для всех интерфейсов (и внутри
  контейнеров); порт `0` выбирает свободный порт.
- Тестируйте на настоящих сокетах по адресу `127.0.0.1:0`, а логику протокола держите в чистых,
  отдельно протестированных функциях.

## Источники и что почитать

- [`std::net::TcpListener`](https://doc.rust-lang.org/std/net/struct.TcpListener.html) и [`std::net::TcpStream`](https://doc.rust-lang.org/std/net/struct.TcpStream.html) — все методы, которые используются в этом уроке.
- [The Rust Programming Language — 21.1 Building a Single-Threaded Web Server](https://doc.rust-lang.org/book/ch21-01-single-threaded.html) — версия первых шагов из книги.
- [Beej's Guide to Network Programming](https://beej.us/guide/bgnet/) — подробное объяснение системных вызовов сокетов (с примерами на C).
- [Страница руководства socket(7)](https://man7.org/linux/man-pages/man7/socket.7.html) — параметры сокетов в Linux, включая `SO_REUSEADDR`.
- [`std::io::ErrorKind`](https://doc.rust-lang.org/std/io/enum.ErrorKind.html) — список видов ошибок ввода-вывода.

---

[🏠 Начало](../START-HERE.md) · [← 15.3 TCP и UDP](03-tcp-and-udp.md) · [📋 Этап 15](README.md) · [15.5 UDP-сокеты в Rust →](05-udp-sockets-in-rust.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
