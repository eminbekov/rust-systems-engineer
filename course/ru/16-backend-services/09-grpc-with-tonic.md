<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 16.8 Аутентификация в веб-сервисе](08-authentication-in-a-web-service.md) · [📋 Этап 16](README.md) · [16.10 Кэширование с Redis →](10-caching-with-redis.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/16-backend-services/09-grpc-with-tonic.md) <!-- nav -->

# Урок 16.9 — gRPC на tonic

> **Этап 16 — Бэкенд-сервисы** · Урок 9 из 16 · ⏱ около 2,5 часа (2 занятия)

## Что вы узнаете

- Что такое RPC и gRPC и когда команда выбирает gRPC вместо REST API с JSON.
- Как описать сервис в файле `.proto` с помощью Protocol Buffers и зачем у каждого поля есть номер.
- Как `tonic` и `prost` превращают файл `.proto` в код на Rust в `build.rs`, ничего не устанавливая на ваш Mac.
- Как написать gRPC-сервер и клиент, возвращать ошибки с кодами статуса и передавать поток ответов.
- Как сквозным тестом проверить gRPC-сервис на свободном порту.

## Перед началом

- Вы прошли [Урок 16.8 — Аутентификация в веб-сервисе](08-authentication-in-a-web-service.md), знаете REST API по [Уроку 16.2](02-designing-http-apis.md) и axum по [Уроку 16.3](03-web-services-with-axum.md).
- Вы знаете задачи и каналы Tokio ([Урок 14.13](../14-concurrency-and-async/13-tokio-tasks-and-channels.md)) и устройство HTTP (Hypertext Transfer Protocol, протокол передачи гипертекста) ([Урок 15.8 — HTTP с нуля](../15-networking/08-http-from-scratch.md)).
- Скрипты сборки (`build.rs`) встречались, когда вы вызывали код на C в [Уроке 06.10](../06-rust-advanced/10-ffi-calling-c.md); всё нужное этот урок объясняет заново.

## Новые слова

| Слово | Значение |
|---|---|
| **RPC** (Remote Procedure Call, удалённый вызов процедуры) | Вызов функции, которая работает в другой программе, часто на другом компьютере, так, будто это обычный вызов функции. |
| **gRPC** (gRPC Remote Procedure Calls, удалённые вызовы процедур gRPC) | Популярная система RPC, созданная в Google. Передаёт сообщения Protocol Buffers поверх HTTP/2. |
| **HTTP/2** (Hypertext Transfer Protocol version 2, протокол передачи гипертекста версии 2) | Новая версия HTTP. Она двоичная, и одно соединение TCP (Transmission Control Protocol, протокол управления передачей) несёт много запросов одновременно («потоки»). |
| **Protocol Buffers** (protobuf) | Независимый от языка способ описывать сообщения и кодировать их в компактные байты. |
| **IDL** (Interface Definition Language, язык описания интерфейсов) | Маленький язык, который только *описывает* интерфейс (функции и данные), но не то, как он работает. Язык `.proto` — это IDL. |
| **файл `.proto`** (`.proto` file) | Текстовый файл с определениями сообщений и сервисов на языке Protocol Buffers. |
| **сообщение** (message) | Тип-запись в файле `.proto`, похожий на структуру Rust. |
| **номер поля** (field number, tag) | Число после каждого поля (`text = 1`). Именно оно, а не имя, обозначает поле в закодированных байтах. |
| **генерация кода** (code generation) | Инструмент пишет исходный код за вас по описанию, здесь — типы Rust по файлу `.proto`. |
| **скрипт сборки** (build script) | Файл `build.rs`. Cargo компилирует и запускает его *до* компиляции вашего крейта. |
| **`protoc`** | Компилятор Protocol Buffers. Читает файлы `.proto`; его использует `prost-build`. |
| **унарный RPC** (unary RPC) | Один запрос, один ответ. |
| **серверный поток** (server streaming) | Один запрос, много ответов, которые приходят по одному. |
| **код статуса** (status code) | Вид ошибки, которую вернул вызов gRPC, например `NOT_FOUND` или `INVALID_ARGUMENT`. |
| **заглушка клиента** (client stub) | Сгенерированный тип клиента, методы которого отправляют запросы за вас. |

## 1. RPC: вызов функции в другой программе

В [Уроке 16.2](02-designing-http-apis.md) вы проектировали REST API. API (Application Programming Interface, программный интерфейс) — это набор запросов, которые принимает программа. REST (Representational State Transfer, передача состояния представления) — стиль, в котором URL (Uniform Resource Locator, адрес в интернете) называют *ресурсы*, а методы HTTP говорят, что с ними сделать. Клиент мыслит ресурсами: `GET /notes/7`.

**RPC** (Remote Procedure Call) — другой способ мыслить. Клиент вызывает *функцию*: `get_note(7)`. Функция работает в другой программе, результат возвращается по сети, а код клиента выглядит почти как обычный вызов.

```text
client program                         server program
──────────────                         ──────────────
let note = client.get_note(7) ──────►  fn get_note(id) { … }
          ◄──────────────────────────  returns Note { id: 7, … }
```

**gRPC** — самая распространённая система RPC для сервисов, которые общаются с другими сервисами. У неё три части:

1. **Protocol Buffers.** Вы описываете функции и данные в файле `.proto`.
2. **Генерация кода.** Инструмент превращает этот файл в код: типы для сообщений, *трейт* для сервера, тип *клиента* для вызывающих.
3. **Транспорт HTTP/2.** Каждый вызов — это запрос HTTP/2 с двоичными байтами protobuf в теле.

Когда команды выбирают gRPC, а когда JSON (JavaScript Object Notation, текстовый формат данных) поверх REST?

| | REST + JSON | gRPC |
|---|---|---|
| Кто вызывает | Браузеры, скрипты, другие компании | Ваши собственные сервисы внутри вашей сети |
| Контракт | Документация или файл OpenAPI (стандартный формат описания REST API) | Файл `.proto`, проверяемый компилятором |
| Размер данных | Текст, имена полей в каждом сообщении | Двоичный, маленький, только номера полей |
| Потоки | Сложно | Встроены |
| Легко попробовать через `curl` | Да | Нет, нужны специальные инструменты |

Типичная схема: REST API на границе системы для браузеров и мобильных приложений и gRPC между вашими внутренними сервисами.

## 2. Описание сервиса в файле `.proto`

Мы построим маленький сервис **заметок** с тремя функциями: добавить заметку, получить одну заметку, перечислить все. Создайте проект:

```bash
cd ~/projects/rust/code
cargo new p16-grpc-notes
cd p16-grpc-notes
mkdir proto
```

> 📏 **Соглашение:** на этом этапе каждая тема получает свой проект в `code/p16-<topic>`. Программы проекта лежат в `src/bin/` (один файл — одна программа, запуск через `cargo run --bin <name>`), а общий код — в `src/lib.rs`.

Создайте `proto/notes.proto`:

```protobuf
syntax = "proto3";

package notes;

// The service: a list of remote functions.
service Notes {
  rpc AddNote(AddNoteRequest) returns (Note);
  rpc GetNote(GetNoteRequest) returns (Note);
  rpc ListNotes(ListNotesRequest) returns (stream Note);
}

message AddNoteRequest {
  string text = 1;
}

message GetNoteRequest {
  uint64 id = 1;
}

message ListNotesRequest {}

message Note {
  uint64 id = 1;
  string text = 2;
}
```

**Что происходит:**

1. `syntax = "proto3";` говорит, какую версию языка мы используем. Версия 3 — текущая.
2. `package notes;` — пространство имён. Оно становится именем модуля Rust для сгенерированного кода.
3. `service Notes` перечисляет функции. Каждый `rpc` принимает ровно **одно** сообщение и возвращает одно сообщение.
4. `returns (stream Note)` означает *много* сообщений `Note`, отправленных одно за другим: серверный поток.
5. `message` — это запись, похожая на структуру. `string` становится `String` в Rust, а `uint64` — `u64`.
6. `ListNotesRequest {}` пустое. Каждому RPC нужно сообщение-запрос, даже если отправлять нечего. Пустое сообщение позволяет добавить поля позже.

### Зачем полям номера

Число после поля (`text = 1`) — это его **номер поля**. В передаваемых байтах protobuf пишет номер, а не имя. Это делает байты компактными и даёт правило безопасного изменения контракта:

- **Можно** добавить новое поле с новым номером. Старые программы пропускают номера, которых не знают.
- **Можно** переименовать поле. Имени нет в байтах.
- **Нельзя никогда** повторно использовать или менять номер существующего поля. Старый клиент прочитает байты как другое поле. Когда удаляете поле, напишите `reserved 3;` (его номер), чтобы никто не взял номер снова.

В proto3 поле, которое не отправили, получает **значение по умолчанию**: `0` для чисел, `""` для строк, `false` для логических значений. «Отсутствующего» значения или `null` нет, если только вы не пометите поле как `optional` (`optional string text = 1;`): тогда prost превращает его в `Option<String>`.

## 3. От `.proto` к Rust: `build.rs`

Работу делают два крейта:

- **`prost`** кодирует и декодирует сообщения protobuf. `prost-build` генерирует структуры Rust из файлов `.proto`.
- **`tonic`** — библиотека gRPC: транспорт HTTP/2, сервер, клиент. `tonic-prost-build` добавляет к тому, что генерирует `prost-build`, трейт сервера и тип клиента, а `tonic-prost` соединяет их во время работы программы.

`prost-build` нужна программа-компилятор `protoc`. Вместо установки на ваш Mac мы используем крейт **`protoc-bin-vendored`**, в котором лежат готовые программы `protoc` для распространённых систем. Добавьте всё:

```bash
cargo add tonic@0.14 tonic-prost@0.14 prost@0.14
cargo add tokio@1 --features macros,rt-multi-thread
cargo add tokio-stream@0.1 --features net
cargo add --build tonic-prost-build@0.14 protoc-bin-vendored@3
```

Теперь `Cargo.toml` содержит:

```toml
[dependencies]
prost = "0.14"
tokio = { version = "1", features = ["macros", "rt-multi-thread"] }
tokio-stream = { version = "0.1", features = ["net"] }
tonic = "0.14"
tonic-prost = "0.14"

[build-dependencies]
protoc-bin-vendored = "3"
tonic-prost-build = "0.14"
```

`--build` кладёт крейт в `[build-dependencies]`: крейты, которые использует только `build.rs`. **Скрипт сборки** — маленькая программа на Rust в файле `build.rs` рядом с `Cargo.toml`. Cargo компилирует и запускает её первой, каждый раз, когда меняются её входные данные, и только потом компилирует ваш крейт. Создайте `build.rs`:

```rust
fn main() -> Result<(), Box<dyn std::error::Error>> {
    let mut config = tonic_prost_build::Config::new();
    config.protoc_executable(protoc_bin_vendored::protoc_bin_path()?);
    tonic_prost_build::configure().compile_with_config(
        config,
        &["proto/notes.proto"],
        &["proto"],
    )?;
    Ok(())
}
```

**Что происходит:**

1. `Config::new()` создаёт настройки `prost-build`. `protoc_executable(...)` говорит использовать `protoc` из `protoc-bin-vendored`, а не искать его в системе.
2. `configure()` создаёт генератор кода tonic с настройками по умолчанию: генерировать и сервер, и клиент.
3. `compile_with_config` читает `proto/notes.proto` (второй список говорит, где инструкции `import` могут искать другие файлы), запускает `protoc` и записывает файл `notes.rs` в папку, которую Cargo выдаёт скрипту сборки (`OUT_DIR`).
4. Он также просит Cargo запускать скрипт заново, когда меняется файл `.proto`.

Сгенерированный файл лежит не в `src/`. Вы подключаете его в модуль макросом `tonic::include_proto!`. Удалите `src/main.rs` (наши программы будут жить в `src/bin/`) и создайте `src/lib.rs` с таким модулем:

```rust
pub mod pb {
    tonic::include_proto!("notes");
}
```

`"notes"` — имя пакета из файла `.proto`. `pb` (сокращение от «protobuf») — частое имя для такого модуля. Вот часть того, что `prost` сгенерировал для сообщения `Note`:

```rust
#[derive(Clone, PartialEq, Eq, Hash, ::prost::Message)]
pub struct Note {
    #[prost(uint64, tag = "1")]
    pub id: u64,
    #[prost(string, tag = "2")]
    pub text: ::prost::alloc::string::String,
}
```

Это обычная структура с публичными полями. Атрибуты `#[prost(...)]` помнят номера полей, а `derive(::prost::Message)` пишет код кодирования.

### Как выглядят байты

Создайте `src/bin/encode.rs`, чтобы увидеть кодирование:

```rust
use p16_grpc_notes::pb::Note;
use prost::Message;

fn main() {
    let note = Note {
        id: 1,
        text: "hi".to_string(),
    };
    let bytes = note.encode_to_vec();
    println!("{} bytes: {:?}", bytes.len(), bytes);

    let back = Note::decode(bytes.as_slice()).unwrap();
    println!("decoded: {back:?}");
}
```

```bash
cargo run --bin encode
```

```text
6 bytes: [8, 1, 18, 2, 104, 105]
decoded: Note { id: 1, text: "hi" }
```

**Что происходит:** `use prost::Message;` делает видимым трейт с методами `encode_to_vec` и `decode`. В байтах `8` значит «поле 1, дальше число» (1 × 8 + тип кодирования 0), а `1` — это `id`; `18` значит «поле 2, дальше длина» (2 × 8 + 2), затем длина `2` и байты строки `"hi"`. Та же заметка в JSON, `{"id":1,"text":"hi"}`, занимает 20 байт, потому что JSON повторяет имена полей в каждом сообщении.

Теперь напишем сервер.

## 4. Сервер

`tonic-prost-build` сгенерировал **трейт** `pb::notes_server::Notes` с одним асинхронным методом на каждый `rpc`. Ваша задача — реализовать его. Вот полный `src/lib.rs`:

```rust
use std::sync::Mutex;

use tokio_stream::wrappers::ReceiverStream;
use tonic::{Request, Response, Status};

pub mod pb {
    tonic::include_proto!("notes");
}

use pb::notes_server::Notes;
use pb::{AddNoteRequest, GetNoteRequest, ListNotesRequest, Note};

#[derive(Default)]
struct Store {
    next_id: u64,
    notes: Vec<Note>,
}

#[derive(Default)]
pub struct NotesService {
    store: Mutex<Store>,
}

#[tonic::async_trait]
impl Notes for NotesService {
    async fn add_note(&self, request: Request<AddNoteRequest>) -> Result<Response<Note>, Status> {
        let text = request.into_inner().text;
        if text.trim().is_empty() {
            return Err(Status::invalid_argument("text must not be empty"));
        }
        let mut store = self.store.lock().unwrap();
        store.next_id += 1;
        let note = Note {
            id: store.next_id,
            text,
        };
        store.notes.push(note.clone());
        Ok(Response::new(note))
    }

    async fn get_note(&self, request: Request<GetNoteRequest>) -> Result<Response<Note>, Status> {
        let id = request.into_inner().id;
        let store = self.store.lock().unwrap();
        match store.notes.iter().find(|n| n.id == id) {
            Some(note) => Ok(Response::new(note.clone())),
            None => Err(Status::not_found(format!("no note with id {id}"))),
        }
    }

    type ListNotesStream = ReceiverStream<Result<Note, Status>>;

    async fn list_notes(
        &self,
        _request: Request<ListNotesRequest>,
    ) -> Result<Response<Self::ListNotesStream>, Status> {
        let notes = self.store.lock().unwrap().notes.clone();
        let (tx, rx) = tokio::sync::mpsc::channel(4);
        tokio::spawn(async move {
            for note in notes {
                if tx.send(Ok(note)).await.is_err() {
                    break; // клиент ушёл
                }
            }
        });
        Ok(Response::new(ReceiverStream::new(rx)))
    }
}
```

**Что происходит:**

1. `NotesService` — состояние нашего сервера: список заметок и следующий свободный id за `Mutex`, потому что tonic вызывает методы из многих задач одновременно через `&self`. Мьютекс из `std` здесь подходит, потому что мы никогда не держим блокировку через `.await` (почему это важно, объясняет Урок 14.13).
2. `#[tonic::async_trait]` должен стоять на `impl`, потому что сгенерированный трейт тоже его использует. Он превращает методы `async fn` в форму, которая работает в трейте, используемом многими задачами.
3. Каждый метод получает `Request<T>` (сообщение плюс метаданные, например заголовки) и возвращает `Result<Response<T>, Status>`. `into_inner()` достаёт сообщение из запроса.
4. `Status::invalid_argument(...)` и `Status::not_found(...)` — это **ошибки с кодом статуса**. Клиент получает код и сообщение. Так gRPC сообщает об ошибках вместо кодов статуса HTTP вроде 404.
5. Для потокового метода трейт требует **ассоциированный тип** `ListNotesStream`: тип потока, который мы возвращаем. Мы создаём канал Tokio `mpsc` (multi-producer, single-consumer, много отправителей и один получатель), запускаем задачу, которая отправляет в него заметки, и возвращаем принимающую сторону, обёрнутую в поток (`ReceiverStream`). Tonic отправляет клиенту каждый элемент, как только он появляется.
6. Если клиент отключился, `tx.send` возвращает ошибку и задача останавливается, так что лишняя работа не делается.

Самые полезные коды статуса:

| Код | Когда использовать |
|---|---|
| `OK` | Вызов удался (сами вы его не создаёте, это и означает `Ok(...)`). |
| `INVALID_ARGUMENT` | Неверен сам запрос: пустой текст, плохой формат. Повтор не поможет. |
| `NOT_FOUND` | Такого объекта нет. |
| `ALREADY_EXISTS` | Создают то, что уже есть. |
| `PERMISSION_DENIED` / `UNAUTHENTICATED` | Вызывающему нельзя / он не доказал, кто он (Урок 16.8). |
| `UNAVAILABLE` | Сервер на время недоступен или перегружен. Клиент может повторить позже. |
| `DEADLINE_EXCEEDED` | Вызов длился дольше, чем позволил клиент. |
| `INTERNAL` | Ошибка в коде или неожиданный сбой на сервере. |

Теперь программа сервера, `src/bin/server.rs`:

```rust
use p16_grpc_notes::NotesService;
use p16_grpc_notes::pb::notes_server::NotesServer;
use tonic::transport::Server;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let addr = "127.0.0.1:50051".parse()?;
    println!("notes server listening on {addr}");
    Server::builder()
        .add_service(NotesServer::new(NotesService::default()))
        .serve(addr)
        .await?;
    Ok(())
}
```

**Что происходит:**

1. `NotesServer::new(...)` — сгенерированный код. Он оборачивает нашу реализацию и превращает запросы HTTP/2 в вызовы наших методов.
2. `Server::builder()` создаёт сервер tonic. `add_service` можно вызвать несколько раз: один сервер может обслуживать много сервисов.
3. `serve(addr)` слушает порт 50051 (обычный порт для примеров gRPC) и работает, пока программу не остановят.

> 🔒 **Безопасность:** у этого сервера нет TLS (Transport Layer Security, протокол защиты транспортного уровня, Урок 10.11) и аутентификации, поэтому он слушает только `127.0.0.1`. Настоящий gRPC-сервис использует TLS (tonic поддерживает его с фичей `tls-ring`) и проверяет токен в метаданных запроса, так же как заголовки HTTP в Уроке 16.8.

## 5. Клиент

Сгенерированная **заглушка клиента** `NotesClient` имеет по одному асинхронному методу на каждый `rpc`. Создайте `src/bin/client.rs`:

```rust
use p16_grpc_notes::pb::notes_client::NotesClient;
use p16_grpc_notes::pb::{AddNoteRequest, GetNoteRequest, ListNotesRequest};

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let mut client = NotesClient::connect("http://127.0.0.1:50051").await?;

    let note = client
        .add_note(AddNoteRequest {
            text: "buy milk".to_string(),
        })
        .await?
        .into_inner();
    println!("added: {note:?}");

    client
        .add_note(AddNoteRequest {
            text: "call mom".to_string(),
        })
        .await?;

    match client.get_note(GetNoteRequest { id: 42 }).await {
        Ok(response) => println!("found: {:?}", response.into_inner()),
        Err(status) => println!(
            "error: code={:?} message={}",
            status.code(),
            status.message()
        ),
    }

    let mut stream = client.list_notes(ListNotesRequest {}).await?.into_inner();
    while let Some(note) = stream.message().await? {
        println!("listed: {} {}", note.id, note.text);
    }
    Ok(())
}
```

Запустите сервер в одном терминале и оставьте его работать:

```bash
cargo run --bin server
```

```text
notes server listening on 127.0.0.1:50051
```

Запустите клиент во втором терминале дважды:

```bash
cargo run --bin client
cargo run --bin client
```

```text
added: Note { id: 1, text: "buy milk" }
error: code=NotFound message=no note with id 42
listed: 1 buy milk
listed: 2 call mom
added: Note { id: 3, text: "buy milk" }
error: code=NotFound message=no note with id 42
listed: 1 buy milk
listed: 2 call mom
listed: 3 buy milk
listed: 4 call mom
```

**Что происходит:**

1. `NotesClient::connect` открывает одно соединение HTTP/2. Адрес должен начинаться с `http://` (или `https://` с TLS).
2. `client.add_note(...)` выглядит как локальный асинхронный вызов. За ним tonic кодирует сообщение, отправляет запрос HTTP/2 по пути `/notes.Notes/AddNote`, ждёт и декодирует ответ. `.into_inner()` достаёт `Note` из `Response`.
3. Вызову нужен `&mut self`, поэтому `client` объявлен как `mut`. Клиент дёшево клонировать (`clone()`), и клоны делят одно соединение, так что у каждой задачи может быть своя копия.
4. `get_note(42)` возвращает `Err(Status)`. `status.code()` равен `Code::NotFound` — ровно тому, что выбрал сервер.
5. `list_notes` возвращает `Streaming<Note>`. `stream.message().await?` даёт `Some(note)` для каждого сообщения и `None`, когда сервер закончил.
6. Второй запуск показывает id 3 и 4: заметки живут в памяти сервера, а он продолжает работать между запусками клиента.

Остановите сервер через Ctrl+C.

## 6. Сквозное тестирование сервиса

Тест gRPC может запустить настоящий сервер прямо внутри теста, на **свободном порту**, и общаться с ним настоящим клиентом. Порт `0` просит операционную систему выбрать свободный порт, поэтому тесты могут идти параллельно. Создайте `tests/notes_api.rs`:

```rust
use p16_grpc_notes::NotesService;
use p16_grpc_notes::pb::notes_client::NotesClient;
use p16_grpc_notes::pb::notes_server::NotesServer;
use p16_grpc_notes::pb::{AddNoteRequest, GetNoteRequest};
use tokio::net::TcpListener;
use tokio_stream::wrappers::TcpListenerStream;
use tonic::transport::{Channel, Server};

/// Запускает сервер на свободном порту и возвращает подключённого клиента.
async fn start_server() -> NotesClient<Channel> {
    let listener = TcpListener::bind("127.0.0.1:0").await.unwrap();
    let addr = listener.local_addr().unwrap();
    tokio::spawn(async move {
        Server::builder()
            .add_service(NotesServer::new(NotesService::default()))
            .serve_with_incoming(TcpListenerStream::new(listener))
            .await
            .unwrap();
    });
    NotesClient::connect(format!("http://{addr}")).await.unwrap()
}

#[tokio::test]
async fn added_note_can_be_read_back() {
    let mut client = start_server().await;
    let added = client
        .add_note(AddNoteRequest {
            text: "test".to_string(),
        })
        .await
        .unwrap()
        .into_inner();
    let found = client
        .get_note(GetNoteRequest { id: added.id })
        .await
        .unwrap()
        .into_inner();
    assert_eq!(found, added);
}

#[tokio::test]
async fn empty_text_is_invalid_argument() {
    let mut client = start_server().await;
    let status = client
        .add_note(AddNoteRequest {
            text: "  ".to_string(),
        })
        .await
        .unwrap_err();
    assert_eq!(status.code(), tonic::Code::InvalidArgument);
}
```

```bash
cargo test --test notes_api
```

```text
running 2 tests
test empty_text_is_invalid_argument ... ok
test added_note_can_be_read_back ... ok

test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

**Что происходит:**

1. Мы **сначала** привязываем слушатель и читаем его адрес. Только потом запускаем сервер в задаче. Когда выполняется `connect`, порт уже открыт, поэтому гонки нет.
2. `serve_with_incoming` обслуживает соединения из потока принятых сокетов вместо адреса. `TcpListenerStream` (фича `net` крейта `tokio-stream`) превращает слушатель в такой поток.
3. `#[tokio::test]` даёт каждому тесту свой рантайм. Когда тест заканчивается, рантайм останавливается, а с ним и задача сервера.
4. Каждый тест получает свежий сервер, поэтому тесты не делят заметки.

> 🧪 **Тестирование:** методы `NotesService` можно вызвать и напрямую в модульном тесте (`service.add_note(Request::new(...)).await`), совсем без сети. Сквозной тест выше проверяет больше: сгенерированный код, кодирование и коды статуса так, как их видит клиент.

## Частые ошибки

> ⚠️ **Частая ошибка:** на машине сборки нет `protoc`. Если вы используете `tonic_prost_build::compile_protos(...)` без `protoc-bin-vendored`, а программы `protoc` нет, сборка падает:

```text
error: failed to run custom build command for `p16-grpc-notes v0.1.0 (...)`

Caused by:
  --- stderr
  Error: Custom { kind: NotFound, error: "Could not find `protoc`. If `protoc` is installed, try setting the `PROTOC` environment variable to the path of the `protoc` binary. ..." }
```

Исправление: используйте встроенный `protoc`, как в разделе 3, или установите `protoc` менеджером пакетов вашей системы.

> ⚠️ **Частая ошибка:** вы добавили `rpc` в файл `.proto`, но не в свой `impl`. У сгенерированного трейта теперь на один метод больше:

```text
error[E0046]: not all trait items implemented, missing: `delete_note`
   --> src/lib.rs:27:1
    |
 27 |   impl Notes for NotesService {
    |   ^^^^^^^^^^^^^^^^^^^^^^^^^^^ missing `delete_note` in implementation
```

Исправление: реализуйте новый метод. Эта ошибка — благо: компилятор проверяет, что сервер соответствует контракту.

> ⚠️ **Частая ошибка:** адрес клиента без схемы, `NotesClient::connect("127.0.0.1:50051")`:

```text
Error: tonic::transport::Error(Transport, ConnectError(ConnectError("invalid URL, scheme is missing")))
```

Исправление: пишите `"http://127.0.0.1:50051"`. Если сервер не запущен, вместо этого вы получите `ConnectionRefused`.

## Попробуйте сами

1. Соберите проект из разделов 2–6 и запустите `cargo test`.
2. Запустите сервер, затем клиент три раза. Перед каждым запуском предскажите id.
3. В клиенте замените id `42` на `1`. Запустите: вы получите `found: Note { id: 1, text: "buy milk" }`.
4. Запустите `cargo run --bin encode` с `id: 300` вместо `1`. Байты станут `[8, 172, 2, 18, 2, 104, 105]`: числа занимают переменное число байт (*varint*), маленькие числа — один байт, а 300 — два.

## Упражнения

### Упражнение 1 — Тест на `NOT_FOUND` (легко)

Добавьте в `tests/notes_api.rs` тест `unknown_id_is_not_found`. Он запрашивает id `999` у свежего сервера и проверяет и код, и сообщение.

<details>
<summary>Подсказка</summary>

Используйте `start_server()`, вызовите `get_note` и `unwrap_err()`, чтобы получить `Status`. Сравните `status.code()` с `tonic::Code::NotFound`, а `status.message()` — с текстом, который строит сервер.

</details>

<details>
<summary>Ответ</summary>

```rust
#[tokio::test]
async fn unknown_id_is_not_found() {
    let mut client = start_server().await;
    let status = client
        .get_note(GetNoteRequest { id: 999 })
        .await
        .unwrap_err();
    assert_eq!(status.code(), tonic::Code::NotFound);
    assert_eq!(status.message(), "no note with id 999");
}
```

Почему это работает: у нового сервера нет заметок, поэтому `get_note` идёт в ветку `None` и возвращает `Status::not_found`. Клиент получает те же код и сообщение.

</details>

### Упражнение 2 — Удаление заметки (средне)

Добавьте `rpc DeleteNote(DeleteNoteRequest) returns (DeleteNoteResponse);` с `message DeleteNoteRequest { uint64 id = 1; }` и пустым `message DeleteNoteResponse {}`. Удаление несуществующего id возвращает `NOT_FOUND`. Проверьте это небольшим тестом в стиле Упражнения 1: добавьте заметку, удалите её и ожидайте `NOT_FOUND` от `get_note`.

<details>
<summary>Подсказка</summary>

После изменения файла `.proto` компилятор подскажет, какого метода не хватает. В методе запомните длину списка, вызовите `retain` и сравните длины.

</details>

<details>
<summary>Ответ</summary>

Добавьте `DeleteNoteRequest` и `DeleteNoteResponse` в строку `use pb::{...}`, затем добавьте этот метод внутрь `impl Notes for NotesService`:

```rust
    async fn delete_note(
        &self,
        request: Request<DeleteNoteRequest>,
    ) -> Result<Response<DeleteNoteResponse>, Status> {
        let id = request.into_inner().id;
        let mut store = self.store.lock().unwrap();
        let before = store.notes.len();
        store.notes.retain(|n| n.id != id);
        if store.notes.len() == before {
            return Err(Status::not_found(format!("no note with id {id}")));
        }
        Ok(Response::new(DeleteNoteResponse {}))
    }
```

Почему это работает: `retain` оставляет только заметки с другим id. Если ничего не удалилось, длина не изменилась, значит, такого id не было. Id берутся из счётчика `next_id`, поэтому удалённый id никогда не достанется новой заметке.

</details>

### Упражнение 3 — Фильтр, которого не замечают старые клиенты (сложно)

Добавьте в `ListNotesRequest` поле `string contains = 1;`. Сервер передаёт в потоке только заметки, текст которых содержит эту строку. Объясните, почему клиент, собранный со *старым* файлом `.proto`, по-прежнему получает все заметки.

<details>
<summary>Подсказка</summary>

Какое значение у строкового поля proto3, если отправитель его не задал? Содержит ли каждая строка это значение?

</details>

<details>
<summary>Ответ</summary>

```protobuf
message ListNotesRequest {
  string contains = 1;
}
```

В `list_notes` переименуйте `_request` в `request` и замените строку, которая клонирует все заметки, на:

```rust
        let contains = request.into_inner().contains;
        let notes: Vec<Note> = self
            .store
            .lock()
            .unwrap()
            .notes
            .iter()
            .filter(|n| n.text.contains(&contains))
            .cloned()
            .collect();
```

В новом клиенте напишите `ListNotesRequest { contains: "milk".to_string() }`; он печатает только `listed: 1 buy milk`.

Почему это работает: старый клиент не знает поля 1, поэтому отправляет пустое сообщение. Сервер декодирует отсутствующее поле как значение по умолчанию `""`, а любая строка содержит пустую строку, так что фильтр оставляет все заметки. Добавление поля с новым номером — обратно совместимое изменение.

</details>

## Вопросы для самопроверки

1. Почему protobuf обозначает поля номерами, а не именами?

<details>
<summary>Ответ</summary>

Номера коротки в закодированных байтах, поэтому сообщения маленькие. И переименование поля становится безопасным. Цена: номер поля нельзя менять или использовать повторно.

</details>

2. Где лежит сгенерированный код на Rust и как ваш код его видит?

<details>
<summary>Ответ</summary>

`build.rs` записывает его в папку Cargo `OUT_DIR` внутри `target/`, а не в `src/`. `tonic::include_proto!("notes")` подключает этот файл в модуль, например `pb`.

</details>

3. Клиент запрашивает несуществующую заметку. Какой код статуса должен вернуть сервер? А для пустого текста в `AddNote`?

<details>
<summary>Ответ</summary>

`NOT_FOUND` (`Status::not_found`) для отсутствующей заметки, `INVALID_ARGUMENT` (`Status::invalid_argument`) для пустого текста.

</details>

4. Как сервер отправляет много заметок на один вызов `ListNotes`?

<details>
<summary>Ответ</summary>

RPC объявлен с `returns (stream Note)`. Сервер возвращает поток (здесь принимающую сторону канала Tokio в обёртке `ReceiverStream`), а запущенная задача отправляет заметки в канал. Tonic отправляет клиенту каждую по мере появления; клиент читает их через `stream.message().await`.

</details>

5. Вашим публичным API пользуются веб-браузеры и другие компании. Ваши внутренние сервисы вызывают друг друга тысячи раз в секунду. Какой стиль подходит для каждого случая?

<details>
<summary>Ответ</summary>

REST с JSON для публичного API: им может пользоваться любой клиент, даже `curl`. gRPC между внутренними сервисами: строгий контракт, проверяемый при компиляции, маленькие двоичные сообщения и потоки.

</details>

## Итоги

- RPC позволяет программе вызвать функцию в другой программе. gRPC — это RPC с сообщениями Protocol Buffers поверх HTTP/2.
- Файл `.proto` — это контракт: сообщения с пронумерованными полями и сервис с функциями `rpc`. Добавляйте поля с новыми номерами; никогда не используйте номер повторно.
- `build.rs` с `tonic-prost-build` генерирует структуры Rust, трейт сервера и заглушку клиента; `protoc-bin-vendored` даёт `protoc` без установки чего-либо.
- Сервер реализует сгенерированный трейт с `#[tonic::async_trait]` и возвращает `Result<Response<T>, Status>`; ошибки несут коды статуса вроде `NOT_FOUND` и `INVALID_ARGUMENT`.
- Серверный поток возвращает поток, часто `ReceiverStream`, который наполняет запущенная задача.
- Сквозные тесты запускают сервер на порту `0` через `serve_with_incoming` и вызывают его настоящим клиентом.

## Источники и что почитать

- [gRPC — Introduction to gRPC](https://grpc.io/docs/what-is-grpc/introduction/) — что такое gRPC и как сочетаются сервисы и сообщения.
- [gRPC — Status codes](https://grpc.io/docs/guides/status-codes/) — полный список кодов и когда использовать каждый.
- [Protocol Buffers — Language Guide (proto3)](https://protobuf.dev/programming-guides/proto3/) — все части языка `.proto`, включая `reserved` и значения по умолчанию.
- [Protocol Buffers — Encoding](https://protobuf.dev/programming-guides/encoding/) — как именно сообщения превращаются в байты (номера полей, типы кодирования, varint).
- [tonic на docs.rs](https://docs.rs/tonic) и [tonic-prost-build на docs.rs](https://docs.rs/tonic-prost-build) — API сервера, клиента и генератора кода.
- [Примеры tonic на GitHub](https://github.com/hyperium/tonic/tree/master/examples) — официальные примеры для потоков, TLS, аутентификации и проверок работоспособности.

---

[🏠 Начало](../START-HERE.md) · [← 16.8 Аутентификация в веб-сервисе](08-authentication-in-a-web-service.md) · [📋 Этап 16](README.md) · [16.10 Кэширование с Redis →](10-caching-with-redis.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
