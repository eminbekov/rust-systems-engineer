<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 16.8 Authentication in a web service](08-authentication-in-a-web-service.md) · [📋 Phase 16](README.md) · [16.10 Caching with Redis →](10-caching-with-redis.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/16-backend-services/09-grpc-with-tonic.md) <!-- nav -->

# Lesson 16.9 — gRPC with tonic

> **Phase 16 — Backend services** · Lesson 9 of 16 · ⏱ about 2.5 hours (2 sittings)

## What you will learn

- What RPC and gRPC are, and when a team picks gRPC instead of a JSON REST API.
- How to describe a service in a `.proto` file with Protocol Buffers, and why every field has a number.
- How `tonic` and `prost` turn the `.proto` file into Rust code in `build.rs`, without installing anything on your Mac.
- How to write a gRPC server and a client, return errors with status codes, and stream many answers.
- How to test a gRPC service end to end on a free port.

## Before you start

- You finished [Lesson 16.8 — Authentication in a web service](08-authentication-in-a-web-service.md) and know REST APIs from [Lesson 16.2](02-designing-http-apis.md) and axum from [Lesson 16.3](03-web-services-with-axum.md).
- You know Tokio tasks and channels ([Lesson 14.13](../14-concurrency-and-async/13-tokio-tasks-and-channels.md)) and how HTTP (Hypertext Transfer Protocol) works ([Lesson 15.8 — HTTP from scratch](../15-networking/08-http-from-scratch.md)).
- Build scripts (`build.rs`) were mentioned when you called C code in [Lesson 06.10](../06-rust-advanced/10-ffi-calling-c.md); this lesson explains again what you need.

## New words

| Word | Meaning |
|---|---|
| **RPC** (Remote Procedure Call) | Calling a function that runs in another program, often on another computer, as if it were a normal function call. |
| **gRPC** (gRPC Remote Procedure Calls) | A popular RPC system, first built at Google. It sends Protocol Buffers messages over HTTP/2. |
| **HTTP/2** (Hypertext Transfer Protocol version 2) | A newer version of HTTP. It is binary, and one TCP (Transmission Control Protocol) connection carries many requests at the same time ("streams"). |
| **Protocol Buffers** (protobuf) | A language-neutral way to describe messages and to encode them into compact bytes. |
| **IDL** (Interface Definition Language) | A small language that only *describes* an interface (functions and data), not how it works. The `.proto` language is an IDL. |
| **`.proto` file** | A text file with message and service definitions in the Protocol Buffers language. |
| **message** | A record type in a `.proto` file, like a Rust struct. |
| **field number** (tag) | The number after each field (`text = 1`). It, not the name, identifies the field in the encoded bytes. |
| **code generation** | A tool writes source code for you from a description, here Rust types from a `.proto` file. |
| **build script** | The file `build.rs`. Cargo compiles and runs it *before* compiling your crate. |
| **`protoc`** | The Protocol Buffers compiler. It reads `.proto` files; `prost-build` uses it. |
| **unary RPC** | One request, one response. |
| **server streaming** | One request, many responses that arrive one by one. |
| **status code** | The kind of error a gRPC call returned, such as `NOT_FOUND` or `INVALID_ARGUMENT`. |
| **stub** (client stub) | The generated client type whose methods send the requests for you. |

## 1. RPC: calling a function in another program

In [Lesson 16.2](02-designing-http-apis.md) you designed a REST API. An API (Application Programming Interface) is the set of requests a program accepts. REST (Representational State Transfer) is a style where URLs (Uniform Resource Locators, web addresses) name *resources* and HTTP methods say what to do with them. A client thinks in resources: `GET /notes/7`.

**RPC** (Remote Procedure Call) is a different way to think. The client calls a *function*: `get_note(7)`. The function runs in another program, the result travels back over the network, and the client code looks almost like a normal call.

```text
client program                         server program
──────────────                         ──────────────
let note = client.get_note(7) ──────►  fn get_note(id) { … }
          ◄──────────────────────────  returns Note { id: 7, … }
```

**gRPC** is the most widely used RPC system for services that talk to other services. It has three parts:

1. **Protocol Buffers.** You describe the functions and the data in a `.proto` file.
2. **Code generation.** A tool turns that file into code: types for the messages, a *trait* for the server, a *client* type for callers.
3. **HTTP/2 transport.** Every call is an HTTP/2 request with binary protobuf bytes in the body.

When do teams choose gRPC, and when JSON (JavaScript Object Notation) over REST?

| | REST + JSON | gRPC |
|---|---|---|
| Who calls it | Browsers, scripts, other companies | Your own services, inside your network |
| Contract | Written docs, or an OpenAPI file (a standard format that describes REST APIs) | The `.proto` file, checked by the compiler |
| Data size | Text, field names in every message | Binary, small, field numbers only |
| Streaming | Hard | Built in |
| Easy to try with `curl` | Yes | No, you need special tools |

A typical design: a REST API at the edge for browsers and mobile apps, and gRPC between your internal services.

## 2. Describing a service in a `.proto` file

We build a tiny **notes** service with three functions: add a note, get one note, list all notes. Create the project:

```bash
cd ~/projects/rust/code
cargo new p16-grpc-notes
cd p16-grpc-notes
mkdir proto
```

> 📏 **Convention:** In this phase each topic gets its own project in `code/p16-<topic>`. Programs that a project runs go in `src/bin/` (one file is one program, run with `cargo run --bin <name>`), and shared code goes in `src/lib.rs`.

Create `proto/notes.proto`:

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

**What happens:**

1. `syntax = "proto3";` says which version of the language we use. Version 3 is current.
2. `package notes;` is a namespace. It becomes the Rust module name of the generated code.
3. `service Notes` lists the functions. Each `rpc` takes exactly **one** message and returns one message.
4. `returns (stream Note)` means *many* `Note` messages, sent one after another: server streaming.
5. A `message` is a record, like a struct. `string` becomes `String` in Rust and `uint64` becomes `u64`.
6. `ListNotesRequest {}` is empty. Every RPC needs a request message, even when there is nothing to send. An empty message lets you add fields later.

### Why fields have numbers

The number after a field (`text = 1`) is its **field number**. On the wire, protobuf writes the number, not the name. This makes the bytes small and gives you a rule for changing the contract safely:

- You **may** add a new field with a new number. Old programs skip numbers they do not know.
- You **may** rename a field. The name is not in the bytes.
- You **must never** reuse or change the number of an existing field. An old client would read the bytes as the wrong field. When you delete a field, write `reserved 3;` (its number) so nobody reuses it.

In proto3, a field that was not sent gets its **default value**: `0` for numbers, `""` for strings, `false` for booleans. There is no "missing" or `null`, unless you mark the field `optional` (`optional string text = 1;`), which prost turns into an `Option<String>`.

## 3. From `.proto` to Rust: `build.rs`

Two crates do the work:

- **`prost`** encodes and decodes protobuf messages. `prost-build` generates Rust structs from `.proto` files.
- **`tonic`** is the gRPC library: HTTP/2 transport, server, client. `tonic-prost-build` adds the server trait and the client type to what `prost-build` generates, and `tonic-prost` connects the two at run time.

`prost-build` needs the `protoc` compiler program. Instead of installing it on your Mac, we use the crate **`protoc-bin-vendored`**, which contains ready `protoc` programs for common systems. Add everything:

```bash
cargo add tonic@0.14 tonic-prost@0.14 prost@0.14
cargo add tokio@1 --features macros,rt-multi-thread
cargo add tokio-stream@0.1 --features net
cargo add --build tonic-prost-build@0.14 protoc-bin-vendored@3
```

`Cargo.toml` now contains:

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

`--build` puts a crate into `[build-dependencies]`: crates that only `build.rs` uses. A **build script** is a small Rust program in `build.rs`, next to `Cargo.toml`. Cargo compiles and runs it first, every time its inputs change, and then compiles your crate. Create `build.rs`:

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

**What happens:**

1. `Config::new()` creates the settings of `prost-build`. `protoc_executable(...)` tells it to use the `protoc` from `protoc-bin-vendored` instead of searching your system.
2. `configure()` creates the tonic code generator with its defaults: generate both a server and a client.
3. `compile_with_config` reads `proto/notes.proto` (the second list says where `import` statements may look for other files), runs `protoc`, and writes the file `notes.rs` into a folder that Cargo gives to the build script (`OUT_DIR`).
4. It also tells Cargo to run the script again when the `.proto` file changes.

The generated file is not in `src/`. You include it into a module with the macro `tonic::include_proto!`. Delete `src/main.rs` (our programs will live in `src/bin/`) and create `src/lib.rs` with this module:

```rust
pub mod pb {
    tonic::include_proto!("notes");
}
```

`"notes"` is the package name from the `.proto` file. `pb` (short for "protobuf") is a common name for the module. Here is part of what `prost` generated for the `Note` message:

```rust
#[derive(Clone, PartialEq, Eq, Hash, ::prost::Message)]
pub struct Note {
    #[prost(uint64, tag = "1")]
    pub id: u64,
    #[prost(string, tag = "2")]
    pub text: ::prost::alloc::string::String,
}
```

It is a normal struct with public fields. The `#[prost(...)]` attributes remember the field numbers, and `derive(::prost::Message)` writes the encoding code.

### What the bytes look like

Create `src/bin/encode.rs` to see the encoding:

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

**What happens:** `use prost::Message;` brings the trait with `encode_to_vec` and `decode` into scope. In the bytes, `8` means "field 1, a number follows" (1 × 8 + wire type 0) and `1` is the `id`; `18` means "field 2, a length follows" (2 × 8 + 2), then the length `2` and the bytes of `"hi"`. The same note as JSON, `{"id":1,"text":"hi"}`, takes 20 bytes, because JSON repeats the field names in every message.

Now let us write the server.

## 4. The server

`tonic-prost-build` generated a **trait** `pb::notes_server::Notes` with one async method per `rpc`. Your job is to implement it. Here is the complete `src/lib.rs`:

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
                    break; // the client went away
                }
            }
        });
        Ok(Response::new(ReceiverStream::new(rx)))
    }
}
```

**What happens:**

1. `NotesService` is our server state: a list of notes and the next free id, behind a `Mutex`, because tonic calls the methods from many tasks at the same time through `&self`. The `std` mutex is fine here because we never hold the lock across an `.await` (Lesson 14.13 explains why that matters).
2. `#[tonic::async_trait]` must be on the `impl`, because the generated trait uses it too. It turns the `async fn` methods into a form that works in a trait used by many tasks.
3. Each method gets a `Request<T>` (the message plus metadata, such as headers) and returns `Result<Response<T>, Status>`. `into_inner()` takes the message out of the request.
4. `Status::invalid_argument(...)` and `Status::not_found(...)` are **errors with a status code**. The client receives the code and the message. This is how gRPC reports errors, instead of HTTP status codes like 404.
5. For the streaming method, the trait asks for an **associated type** `ListNotesStream`: the type of the stream we return. We create a Tokio `mpsc` (multi-producer, single-consumer) channel, spawn a task that sends the notes into it, and return the receiving side wrapped as a stream (`ReceiverStream`). Tonic sends each item to the client as soon as it arrives.
6. If the client disconnects, `tx.send` fails and the task stops, so no work is wasted.

The most useful status codes:

| Code | Use it when |
|---|---|
| `OK` | The call worked (you never create it yourself, `Ok(...)` means this). |
| `INVALID_ARGUMENT` | The request itself is wrong: empty text, bad format. Retrying will not help. |
| `NOT_FOUND` | The thing does not exist. |
| `ALREADY_EXISTS` | Creating something that is already there. |
| `PERMISSION_DENIED` / `UNAUTHENTICATED` | The caller is not allowed / did not prove who they are (Lesson 16.8). |
| `UNAVAILABLE` | The server is down or overloaded for a moment. The client may retry later. |
| `DEADLINE_EXCEEDED` | The call took longer than the client allowed. |
| `INTERNAL` | A bug or an unexpected failure on the server. |

Now the server program, `src/bin/server.rs`:

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

**What happens:**

1. `NotesServer::new(...)` is generated code. It wraps our implementation and turns HTTP/2 requests into calls of our methods.
2. `Server::builder()` creates the tonic server. `add_service` can be called several times: one server can host many services.
3. `serve(addr)` listens on port 50051 (the usual example port for gRPC) and runs until the program is stopped.

> 🔒 **Security:** This server has no TLS (Transport Layer Security, Lesson 10.11) and no authentication, so it listens only on `127.0.0.1`. A real gRPC service uses TLS (tonic supports it with the feature `tls-ring`) and checks a token in the request metadata, just like the HTTP headers in Lesson 16.8.

## 5. The client

The generated **client stub** `NotesClient` has one async method per `rpc`. Create `src/bin/client.rs`:

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

Start the server in one terminal and keep it running:

```bash
cargo run --bin server
```

```text
notes server listening on 127.0.0.1:50051
```

Run the client in a second terminal, twice:

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

**What happens:**

1. `NotesClient::connect` opens one HTTP/2 connection. The address must start with `http://` (or `https://` with TLS).
2. `client.add_note(...)` looks like a local async call. Behind it, tonic encodes the message, sends an HTTP/2 request to the path `/notes.Notes/AddNote`, waits, and decodes the answer. `.into_inner()` takes the `Note` out of the `Response`.
3. The call needs `&mut self`, which is why `client` is `mut`. The client is cheap to `clone()`, and clones share the same connection, so each task can have its own copy.
4. `get_note(42)` returns `Err(Status)`. `status.code()` is `Code::NotFound`, exactly what the server chose.
5. `list_notes` returns a `Streaming<Note>`. `stream.message().await?` gives `Some(note)` for each message and `None` when the server has finished.
6. The second run shows ids 3 and 4: the notes live in the server's memory, which keeps running between client runs.

Stop the server with Ctrl+C.

## 6. Testing the service end to end

A gRPC test can start the real server inside the test, on a **free port**, and talk to it with the real client. Port `0` asks the operating system to choose a free port, so tests can run in parallel. Create `tests/notes_api.rs`:

```rust
use p16_grpc_notes::NotesService;
use p16_grpc_notes::pb::notes_client::NotesClient;
use p16_grpc_notes::pb::notes_server::NotesServer;
use p16_grpc_notes::pb::{AddNoteRequest, GetNoteRequest};
use tokio::net::TcpListener;
use tokio_stream::wrappers::TcpListenerStream;
use tonic::transport::{Channel, Server};

/// Starts the server on a free port and returns a connected client.
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

**What happens:**

1. We bind the listener **first** and read its address. Only then do we start the server in a task. When `connect` runs, the port is already open, so there is no race.
2. `serve_with_incoming` serves connections from a stream of accepted sockets instead of an address. `TcpListenerStream` (the `net` feature of `tokio-stream`) turns the listener into such a stream.
3. `#[tokio::test]` gives each test its own runtime. When a test ends, its runtime stops, and the server task stops with it.
4. Each test gets a fresh server, so the tests do not share notes.

> 🧪 **Testing:** You can also call the methods of `NotesService` directly in a unit test (`service.add_note(Request::new(...)).await`), with no network at all. The end-to-end test above checks more: the generated code, the encoding and the status codes as the client sees them.

## Common mistakes

> ⚠️ **Common mistake:** No `protoc` on the build machine. If you use `tonic_prost_build::compile_protos(...)` without `protoc-bin-vendored` and there is no `protoc` program, the build fails:

```text
error: failed to run custom build command for `p16-grpc-notes v0.1.0 (...)`

Caused by:
  --- stderr
  Error: Custom { kind: NotFound, error: "Could not find `protoc`. If `protoc` is installed, try setting the `PROTOC` environment variable to the path of the `protoc` binary. ..." }
```

Fix: use the vendored `protoc` as in Section 3, or install `protoc` with your system's package manager.

> ⚠️ **Common mistake:** You added an `rpc` to the `.proto` file but not to your `impl`. The generated trait now has one more method:

```text
error[E0046]: not all trait items implemented, missing: `delete_note`
   --> src/lib.rs:27:1
    |
 27 |   impl Notes for NotesService {
    |   ^^^^^^^^^^^^^^^^^^^^^^^^^^^ missing `delete_note` in implementation
```

Fix: implement the new method. This error is a good thing: the compiler checks that the server matches the contract.

> ⚠️ **Common mistake:** The client address without a scheme, `NotesClient::connect("127.0.0.1:50051")`:

```text
Error: tonic::transport::Error(Transport, ConnectError(ConnectError("invalid URL, scheme is missing")))
```

Fix: write `"http://127.0.0.1:50051"`. If the server is not running, you get `ConnectionRefused` instead.

## Try it yourself

1. Build the project from Sections 2–6 and run `cargo test`.
2. Start the server, then run the client three times. Predict the ids before each run.
3. In the client, change the id `42` to `1`. Run it: you get `found: Note { id: 1, text: "buy milk" }`.
4. Run `cargo run --bin encode` with `id: 300` instead of `1`. The bytes become `[8, 172, 2, 18, 2, 104, 105]`: numbers use a variable number of bytes (a *varint*), small numbers take one byte and 300 takes two.

## Exercises

### Exercise 1 — A test for `NOT_FOUND` (easy)

Add a test `unknown_id_is_not_found` to `tests/notes_api.rs`. It asks for id `999` on a fresh server and checks both the code and the message.

<details>
<summary>Hint</summary>

Use `start_server()`, call `get_note`, and `unwrap_err()` to get the `Status`. Compare `status.code()` with `tonic::Code::NotFound` and `status.message()` with the text the server builds.

</details>

<details>
<summary>Answer</summary>

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

Why it works: a new server has no notes, so `get_note` takes the `None` branch and returns `Status::not_found`. The client receives the same code and message.

</details>

### Exercise 2 — Delete a note (medium)

Add `rpc DeleteNote(DeleteNoteRequest) returns (DeleteNoteResponse);` with `message DeleteNoteRequest { uint64 id = 1; }` and an empty `message DeleteNoteResponse {}`. Deleting an id that does not exist returns `NOT_FOUND`. Check it with a small test in the style of Exercise 1: add a note, delete it, and expect `NOT_FOUND` from `get_note`.

<details>
<summary>Hint</summary>

After you change the `.proto` file, the compiler tells you which method is missing. In the method, remember the length of the list, call `retain`, and compare the lengths.

</details>

<details>
<summary>Answer</summary>

Add `DeleteNoteRequest` and `DeleteNoteResponse` to the `use pb::{...}` line, then add this method inside `impl Notes for NotesService`:

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

Why it works: `retain` keeps only the notes with another id. If nothing was removed, the length did not change, so the id did not exist. Ids come from the `next_id` counter, so a deleted id is never given to a new note.

</details>

### Exercise 3 — A filter that old clients do not notice (hard)

Add a field `string contains = 1;` to `ListNotesRequest`. The server streams only notes whose text contains that string. Explain why a client that was compiled with the *old* `.proto` file still gets all notes.

<details>
<summary>Hint</summary>

What value does a proto3 string field have when the sender did not set it? Does every string contain that value?

</details>

<details>
<summary>Answer</summary>

```protobuf
message ListNotesRequest {
  string contains = 1;
}
```

In `list_notes`, rename `_request` to `request` and replace the line that clones all notes with:

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

In the new client, write `ListNotesRequest { contains: "milk".to_string() }`; it prints only `listed: 1 buy milk`.

Why it works: an old client does not know field 1, so it sends an empty message. The server decodes the missing field as the default value `""`, and every string contains the empty string, so the filter keeps all notes. Adding a field with a new number is a backward-compatible change.

</details>

## Quiz

1. Why does protobuf identify fields by number and not by name?

<details>
<summary>Answer</summary>

Numbers are short in the encoded bytes, so messages are small. And it makes renaming a field safe. The price: you must never change or reuse a field number.

</details>

2. Where does the generated Rust code live, and how does your code see it?

<details>
<summary>Answer</summary>

`build.rs` writes it into Cargo's `OUT_DIR` folder inside `target/`, not into `src/`. `tonic::include_proto!("notes")` includes that file into a module, for example `pb`.

</details>

3. A client asks for a note that does not exist. Which status code should the server return? And for an empty text in `AddNote`?

<details>
<summary>Answer</summary>

`NOT_FOUND` (`Status::not_found`) for the missing note, `INVALID_ARGUMENT` (`Status::invalid_argument`) for the empty text.

</details>

4. How does the server send many notes for one `ListNotes` call?

<details>
<summary>Answer</summary>

The RPC is declared with `returns (stream Note)`. The server returns a stream (here the receiving side of a Tokio channel, wrapped in `ReceiverStream`), and a spawned task sends notes into the channel. Tonic sends each one to the client as it arrives; the client reads them with `stream.message().await`.

</details>

5. Your public API is used by web browsers and other companies. Your internal services call each other thousands of times per second. Which style fits each case?

<details>
<summary>Answer</summary>

REST with JSON for the public API: every client can use it, even with `curl`. gRPC between internal services: a strict contract checked at compile time, small binary messages and streaming.

</details>

## Summary

- RPC lets a program call a function in another program. gRPC is RPC with Protocol Buffers messages over HTTP/2.
- The `.proto` file is the contract: messages with numbered fields and a service with `rpc` functions. Add fields with new numbers; never reuse a number.
- `build.rs` with `tonic-prost-build` generates Rust structs, a server trait and a client stub; `protoc-bin-vendored` supplies `protoc` without installing anything.
- The server implements the generated trait with `#[tonic::async_trait]` and returns `Result<Response<T>, Status>`; errors carry status codes like `NOT_FOUND` and `INVALID_ARGUMENT`.
- Server streaming returns a stream, often a `ReceiverStream` fed by a spawned task.
- End-to-end tests start the server on port `0` with `serve_with_incoming` and call it with the real client.

## Sources and further reading

- [gRPC — Introduction to gRPC](https://grpc.io/docs/what-is-grpc/introduction/) — what gRPC is and how services and messages fit together.
- [gRPC — Status codes](https://grpc.io/docs/guides/status-codes/) — the full list of codes and when to use each one.
- [Protocol Buffers — Language Guide (proto3)](https://protobuf.dev/programming-guides/proto3/) — every part of the `.proto` language, including `reserved` and default values.
- [Protocol Buffers — Encoding](https://protobuf.dev/programming-guides/encoding/) — exactly how messages become bytes (field numbers, wire types, varints).
- [tonic on docs.rs](https://docs.rs/tonic) and [tonic-prost-build on docs.rs](https://docs.rs/tonic-prost-build) — the API of the server, the client and the code generator.
- [tonic examples on GitHub](https://github.com/hyperium/tonic/tree/master/examples) — official examples for streaming, TLS, authentication and health checks.

---

[🏠 Start](../START-HERE.md) · [← 16.8 Authentication in a web service](08-authentication-in-a-web-service.md) · [📋 Phase 16](README.md) · [16.10 Caching with Redis →](10-caching-with-redis.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
