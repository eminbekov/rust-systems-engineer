<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 16.1 What a backend service is](01-what-a-backend-service-is.md) · [📋 Phase 16](README.md) · [16.3 Web services with axum →](03-web-services-with-axum.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/16-backend-services/02-designing-http-apis.md) <!-- nav -->

# Lesson 16.2 — Designing HTTP APIs: REST

> **Phase 16 — Backend services** · Lesson 2 of 16 · ⏱ about 105 minutes

## What you will learn

- What REST (REpresentational State Transfer) is: resources, URLs, methods and representations.
- Which HTTP (HyperText Transfer Protocol) method to use for which action, and what *safe* and *idempotent* mean.
- The status codes a backend really uses, and how to report errors in a consistent JSON (JavaScript Object Notation) body.
- How to design the API (Application Programming Interface) of the tasks service that you build in Lessons 16.3 to 16.8.
- How to write a test that protects the JSON contract.

## Before you start

- You finished [Lesson 16.1 — What a backend service is](01-what-a-backend-service-is.md) and have `code/p16-hello`.
- You know the parts of an HTTP message (Lesson 15.8 — HTTP from scratch): request line, headers, body,
  status line.
- You know `serde` and `serde_json` from [Lesson 05.11 — Practical crates](../05-rust-intermediate/11-practical-crates-serde-clap-errors.md).

## New words

| Word | Meaning |
|---|---|
| **REST** (REpresentational State Transfer) | A style of designing HTTP APIs: the API is a set of *resources*, each with its own URL (Uniform Resource Locator), and you act on them with the standard HTTP methods. |
| **resource** | A thing the API lets you work with: a task, a user, a list of tasks. It has a URL. |
| **representation** | The form in which a resource travels in a request or response, usually JSON. |
| **URL** (Uniform Resource Locator) | The full address of a resource: `http://127.0.0.1:3000/tasks/7?done=true`. The *path* is `/tasks/7`; the *query string* is `done=true`. |
| **JSON** (JavaScript Object Notation) | The text format for data that almost every web API uses: `{"id": 7, "done": false}`. |
| **safe method** | A method that only reads and never changes anything on the server (`GET`, `HEAD`). |
| **idempotent method** | A method that has the same effect whether you send it once or ten times (`GET`, `PUT`, `DELETE`). Clients may safely repeat it after a network error. |
| **status code** | The three-digit number in the first line of every response that says how the request went (`200`, `404`, …). |
| **contract** | The exact, agreed description of the API: paths, methods, bodies, status codes. Both sides rely on it. |
| **pagination** | Returning a long list in pages (for example 50 items at a time) instead of all at once. |
| **breaking change** | A change to an API that makes existing clients fail, for example renaming a JSON field. |

## 1. What REST is

Lesson 15.8 showed that HTTP is just text: a method, a path, headers and a body. HTTP itself does not tell
you **which** paths and methods your API should have. You could design an API like this:

```text
POST /createTask        POST /getTask?id=7        POST /deleteTaskNow
```

It works, but every API designed like this is different, and clients must learn each one from zero.
**REST** is a common style that makes APIs predictable. Its main ideas, in plain words:

1. **Everything is a resource with a URL.** The tasks list is `/tasks`. Task number 7 is `/tasks/7`.
   URLs are **nouns**, never verbs.
2. **The method is the verb.** `GET /tasks/7` reads task 7. `DELETE /tasks/7` deletes it. You do not
   invent new verbs; you use the few that HTTP already has.
3. **Resources travel as representations,** usually JSON. The server sends `{"id":7,"title":"Buy milk","done":false}`;
   it does not matter how it stores the task inside.
4. **Every request stands alone.** The request carries everything the server needs (including who you are,
   Lesson 16.8). The server does not remember the "previous request" of a client. This fits the stateless
   services of Lesson 16.1.
5. **Status codes tell the result.** `200` worked, `404` not found, and so on (Section 3).

A client that knows these rules can guess most of your API without reading anything. That is the whole point.

## 2. Methods: the verbs of HTTP

| Method | Meaning | Body in request? | Safe | Idempotent | Typical success |
|---|---|---|---|---|---|
| `GET` | Read a resource or a list | no | yes | yes | `200 OK` |
| `POST` | Create a new resource in a collection (or run an action) | yes | no | **no** | `201 Created` |
| `PUT` | Replace a resource completely | yes | no | yes | `200 OK` |
| `PATCH` | Change some fields of a resource | yes | no | usually not\* | `200 OK` |
| `DELETE` | Delete a resource | no | no | yes | `204 No Content` |

\* A `PATCH` that sets fields (`{"done": true}`) is idempotent in practice; one that adds to a value is not.

**Safe** means "only reads". Browsers, caches and crawlers may send `GET` requests whenever they like, so a
`GET` must **never** change data. A link like `GET /tasks/7/delete` is a real bug: a program that checks all
links on a page would delete your tasks.

**Idempotent** means "doing it twice is the same as doing it once". This matters because networks fail.
Imagine a client sends `DELETE /tasks/7`, the task is deleted, but the answer is lost on the way back.
The client does not know what happened, so it sends the request again. The second `DELETE` finds nothing to
delete, and the end state is the same: task 7 is gone. Repeating was harmless.

Now imagine the same with `POST /tasks` (create a task). The first request created task 8, the answer was
lost, and the retry creates task 9: **two** identical tasks. `POST` is not idempotent, so clients must not
blindly repeat it. (Real APIs solve this with an *idempotency key*, a unique value the client sends with the
request so the server can recognise a repeat. Phase 18 and Phase 20 return to retries.)

You can see the method and the body that `curl` sends with `-v` ("verbose"). `-d` gives a body (and makes
`curl` use `POST`), `-H` adds a header. Start `code/p16-hello` from Lesson 16.1 and run:

```bash
curl -v -H 'content-type: application/json' -d '{"title":"Buy milk"}' http://127.0.0.1:3000/health
```

```text
*   Trying 127.0.0.1:3000...
* Connected to 127.0.0.1 (127.0.0.1) port 3000
> POST /health HTTP/1.1
> Host: 127.0.0.1:3000
> User-Agent: curl/8.7.1
> Accept: */*
> content-type: application/json
> Content-Length: 20
>
* upload completely sent off: 20 bytes
< HTTP/1.1 405 Method Not Allowed
< allow: GET,HEAD
< content-length: 0
< date: Wed, 18 Nov 2026 07:02:39 GMT
<
* Connection #0 to host 127.0.0.1 left intact
```

**What happens:**

1. Lines with `*` are `curl`'s own notes: it connects to the port.
2. Lines with `>` are what `curl` **sends**: the request line `POST /health HTTP/1.1`, the headers, and the
   20 bytes of JSON body (not printed by `-v`). `content-type: application/json` tells the server what the
   body is.
3. Lines with `<` are what the server **answers**. `/health` only allows `GET`, so `405`.

You will use exactly this form of `curl` to create tasks in the next lesson.

## 3. Status codes

The first digit gives the class:

| Class | Meaning | Whose "fault" |
|---|---|---|
| `2xx` | Success | — |
| `3xx` | Go somewhere else (redirect) | — |
| `4xx` | The request is wrong; repeating it unchanged will fail again | the client |
| `5xx` | The server failed; the same request may work later | the server |

The codes you need for almost every API:

| Code | Name | Use it when |
|---|---|---|
| `200` | OK | A read or an update worked; the body has the result. |
| `201` | Created | `POST` created a resource. Add a `location` header with its URL, for example `location: /tasks/8`. |
| `204` | No Content | It worked and there is nothing to send back (typical for `DELETE`). |
| `301` / `308` | Moved Permanently / Permanent Redirect | The resource lives at another URL forever (the `location` header). |
| `302` / `307` | Found / Temporary Redirect | Go to another URL this time. A URL shortener (Lesson 16.15) answers with one of these. |
| `400` | Bad Request | The request cannot be read: broken JSON, a wrong type in the query string. |
| `401` | Unauthorized | We do not know who you are: no login or a bad token (Lesson 16.8). |
| `403` | Forbidden | We know who you are, and you may not do this. |
| `404` | Not Found | The resource does not exist (or you may not know that it exists). |
| `409` | Conflict | The request clashes with the current state, for example "this name is already taken". |
| `415` | Unsupported Media Type | The body is not in a format the server accepts (for example JSON without `content-type: application/json`). |
| `422` | Unprocessable Content | The JSON is well formed but the values break a rule: an empty title, a negative amount. |
| `429` | Too Many Requests | The client sent too many requests; it must slow down (rate limiting). |
| `500` | Internal Server Error | A bug or an unexpected failure in the server. |
| `503` | Service Unavailable | The server cannot work right now (overloaded, database down, shutting down). |

> ⚠️ **Common mistake:** answering `200 OK` with a body like `{"success": false}`. Clients, proxies,
> monitoring tools and retry logic all look at the status code first. If the request failed, the status
> code must say so.

> 🔒 **Security:** `404` is often better than `403` for things that belong to other users. "Task 7 is
> forbidden" tells an attacker that task 7 exists; "not found" tells nothing. Lesson 10.3 called this broken
> access control.

### A consistent error body

When something fails, the status code says **what kind** of failure, and the body explains **which**.
Choose one error format and use it for **every** error of the API, so clients can write one piece of code to
handle errors. The tasks API uses:

```json
{"code": "validation_failed", "message": "title must not be empty"}
```

`code` is a short, stable word that programs can check (`not_found`, `validation_failed`, `bad_request`,
`internal`). `message` is for humans and may change. There is also a standard format for this, *Problem
Details for HTTP APIs* (RFC 9457, sent as `application/problem+json`); larger APIs often use it. RFC
(Request for Comments) is the name of the documents that define internet standards.

> 🔒 **Security:** a `500` answer must never contain internal details: no SQL (Structured Query Language) text, no file paths, no stack
> traces. Write those to your logs (Lesson 16.12) and send only `{"code":"internal","message":"internal error"}`.

## 4. Designing URLs and bodies

Rules that most well-designed APIs follow:

- **Plural nouns for collections, an identifier for one item:** `/tasks`, `/tasks/7`. Not `/task/7`, not
  `/getTask`.
- **Lowercase paths, words joined with `-`:** `/password-resets`, not `/passwordResets`.
- **Nest only when a thing truly belongs to another:** `/tasks/7/comments` lists the comments of task 7.
  Keep it to one level; deep paths like `/users/1/projects/2/tasks/7/comments` are hard to use.
- **Query strings filter, sort and page lists:** `/tasks?done=false&limit=50`. They never identify a
  single resource.
- **One naming style for JSON fields.** Rust's `serde` uses the field names of your struct, so
  `snake_case` (`created_at`) is the natural choice. Do not mix `createdAt` and `created_at`.
- **Dates and times as text in RFC 3339 format,** always with a time zone: `"2026-11-18T07:02:39Z"`.
- **Identifiers are opaque.** Clients must treat `7` as a name, not do arithmetic with it.

**Pagination.** `GET /tasks` must not return a million rows. Return a page and let the client ask for the
next one: `GET /tasks?limit=50&offset=100` (skip 100, return 50) is the simplest form. Large APIs prefer a
*cursor*: `GET /tasks?limit=50&after_id=150` ("the 50 tasks after task 150"), which stays fast and correct
even when rows are added while the client pages through them.

**Versioning.** Once clients use your API, a breaking change (renaming a field, removing an endpoint,
changing a type) breaks them. Adding a new optional field or a new endpoint is safe. When you really must
break the contract, publish the new version next to the old one, often with a prefix: `/v1/tasks` and
`/v2/tasks`, and remove `v1` only when no client uses it any more.

## 5. The tasks API

Here is the contract of the service you build in the next lessons. Write it down **before** writing code: it
is much cheaper to change a table than a running API.

A task travels as:

```json
{"id": 1, "title": "Buy milk", "done": false}
```

| Endpoint | Request body | Success | Errors |
|---|---|---|---|
| `GET /tasks` | — | `200`, a JSON array of tasks, oldest first | — |
| `GET /tasks?done=false` | — | `200`, only tasks with this `done` value | `400` if `done` is not `true`/`false` |
| `POST /tasks` | `{"title": "Buy milk"}` | `201`, the new task, header `location: /tasks/{id}` | `400` broken JSON, `415` wrong content type, `422` missing field or invalid title |
| `GET /tasks/{id}` | — | `200`, the task | `404` |
| `PATCH /tasks/{id}` | `{"title": "…"}` and/or `{"done": true}` | `200`, the changed task | `404`, `422` |
| `DELETE /tasks/{id}` | — | `204`, no body | `404` |
| `GET /health` | — | `200`, `ok` | — |

Rules: a title is trimmed (spaces at both ends are removed) and must have 1 to 200 characters. `{id}` is a
positive whole number. Every error has the body `{"code": "...", "message": "..."}`.

The `{id}` in the table is a **path parameter**: a part of the path that changes. In the next lesson you will
see that axum uses exactly this notation.

A full conversation with the API will look like this (you will produce it yourself in Lesson 16.3):

```text
POST /tasks  {"title":"Buy milk"}      →  201  location: /tasks/1   {"id":1,"title":"Buy milk","done":false}
PATCH /tasks/1  {"done":true}          →  200  {"id":1,"title":"Buy milk","done":true}
GET /tasks?done=false                  →  200  []
DELETE /tasks/1                        →  204
GET /tasks/1                           →  404  {"code":"not_found","message":"task 1 not found"}
```

> 📏 **Convention:** keep the contract in the repository next to the code (for example in `API.md`) and
> update it in the same commit as the code. Bigger teams write it in the OpenAPI format, a standard
> machine-readable description of REST APIs; tools can then generate documentation and client code from it.

> 🧪 **Testing:** every row of the contract table is a test case: one request, the expected status, the
> expected body. Lesson 16.3 starts writing such tests, and Lesson 16.14 runs them against a real database.

## Common mistakes

**1. Verbs in URLs.** `POST /tasks/7/delete` or `GET /deleteTask?id=7`. Use `DELETE /tasks/7`. The
`GET` version is also dangerous (Section 2).

**2. `200` for everything.** An error with status `200` looks like success to every tool. Use the right
`4xx` or `5xx` code.

**3. Different error shapes.** One endpoint answers `{"error":"..."}`, another plain text, a third
`{"msg":"..."}`. Pick one format and use it everywhere.

**4. Changing the JSON without noticing.** You rename the struct field `title` to `name` during a
refactoring. `serde` happily follows, and every client breaks. A test that compares the exact JSON
(Exercise 3) catches this.

**5. Returning everything.** `GET /tasks` with no limit is fine with ten tasks and falls over with ten
million. Plan pagination from the start.

## Try it yourself

1. Write the contract of Section 5 into `code/p16-hello/API.md` (you will move it to `code/p16-tasks` in the
   next lesson). Add one more row: `GET /tasks/count` that returns `{"count": 3}`. Is that a good REST
   design, or should it be `GET /tasks?count_only=true`? Both exist in real APIs; write one sentence about
   why you chose yours.
2. With `p16-hello` running, send a `DELETE` with `curl -i -X DELETE http://127.0.0.1:3000/health` and read
   the `allow` header.
3. Run `curl -v http://127.0.0.1:3000/health` and find the request line, the `Host` header and the status
   line in the output.

## Exercises

### Exercise 1 — Choose method, path and status (easy)

For each action on the tasks API, give the method, the path and the success status code:
(a) list the tasks that are done; (b) mark task 4 as done; (c) delete task 9; (d) create a task "Call mom";
(e) read task 12, which does not exist.

<details>
<summary>Hint</summary>

Collections are `/tasks`, one item is `/tasks/{id}`. Changing one field is `PATCH`.

</details>

<details>
<summary>Answer</summary>

(a) `GET /tasks?done=true` → `200`. (b) `PATCH /tasks/4` with `{"done": true}` → `200`.
(c) `DELETE /tasks/9` → `204`. (d) `POST /tasks` with `{"title": "Call mom"}` → `201` and a `location`
header. (e) `GET /tasks/12` → `404` with `{"code":"not_found",…}`.

Why it works: the method says what to do, the path says to what, and the status code says how it went.
(e) is not a "success" at all: the right answer to a missing resource is a `4xx`, not a `200` with an empty body.

</details>

### Exercise 2 — Comments on tasks (medium)

Users want to write comments on a task. Design the endpoints to list the comments of a task, add a comment,
and delete one comment. Give the JSON of a comment, the status codes, and say what happens when the task
does not exist.

<details>
<summary>Hint</summary>

A comment belongs to exactly one task, so one level of nesting is reasonable. Does deleting a comment
need the task id in the path?

</details>

<details>
<summary>Answer</summary>

A comment: `{"id": 3, "task_id": 7, "text": "Use the oat milk", "created_at": "2026-11-18T07:02:39Z"}`.

| Endpoint | Success | Errors |
|---|---|---|
| `GET /tasks/{id}/comments` | `200`, array | `404` if the task does not exist |
| `POST /tasks/{id}/comments` with `{"text": "…"}` | `201`, `location: /comments/{comment_id}` | `404` no task, `422` empty text |
| `DELETE /comments/{comment_id}` | `204` | `404` |

Why it works: listing and creating comments only make sense *inside* a task, so they are nested. A comment
has its own identifier, so it can be addressed directly as `/comments/{comment_id}` without repeating the
task. (`DELETE /tasks/{id}/comments/{comment_id}` is also acceptable; choose one style and keep it.) A
missing task gives `404`, not an empty list, so the client can tell "no comments" from "no such task".

</details>

### Exercise 3 — A contract test (medium)

Create a library `cargo new --lib code/p16-contract`, add `serde` (with the `derive` feature) and
`serde_json`. Write the struct `Task` (`id: u64`, `title: String`, `done: bool`) that serializes to the JSON of
Section 5, and the struct `NewTask` for the body of `POST /tasks`. Write three tests:

1. a `Task` serializes to exactly `{"id":1,"title":"Buy milk","done":false}`;
2. the body `{"title":"Buy milk"}` deserializes into `NewTask`;
3. a misspelled field, `{"titel":"Buy milk"}`, is **rejected**.

<details>
<summary>Hint</summary>

By default `serde` ignores unknown fields, and then test 3 fails differently than you expect (it reports a
missing `title`, but would silently accept `{"title":"a","titel":"b"}`). The attribute
`#[serde(deny_unknown_fields)]` on the struct makes unknown fields an error.

</details>

<details>
<summary>Answer</summary>

```rust
use serde::{Deserialize, Serialize};

/// A task as the API sends it to clients.
#[derive(Debug, Serialize, Deserialize, PartialEq)]
pub struct Task {
    pub id: u64,
    pub title: String,
    pub done: bool,
}

/// The body of `POST /tasks`.
#[derive(Debug, Deserialize, PartialEq)]
#[serde(deny_unknown_fields)]
pub struct NewTask {
    pub title: String,
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn task_json_matches_the_contract() {
        let task = Task {
            id: 1,
            title: "Buy milk".to_string(),
            done: false,
        };
        let json = serde_json::to_string(&task).unwrap();
        assert_eq!(json, r#"{"id":1,"title":"Buy milk","done":false}"#);
    }

    #[test]
    fn new_task_reads_the_documented_body() {
        let body: NewTask = serde_json::from_str(r#"{"title":"Buy milk"}"#).unwrap();
        assert_eq!(
            body,
            NewTask {
                title: "Buy milk".to_string()
            }
        );
    }

    #[test]
    fn misspelled_field_is_rejected() {
        let error = serde_json::from_str::<NewTask>(r#"{"titel":"Buy milk"}"#).unwrap_err();
        println!("{error}");
        assert!(error.to_string().contains("unknown field `titel`"));
    }
}
```

`cargo test -- --nocapture` (which shows the `println!` output) prints:

```text
running 3 tests
unknown field `titel`, expected `title` at line 1 column 8
test tests::new_task_reads_the_documented_body ... ok
test tests::misspelled_field_is_rejected ... ok
test tests::task_json_matches_the_contract ... ok

test result: ok. 3 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

Why it works: test 1 compares the **exact** text, so renaming or reordering a field fails the test before
any client sees it. `r#"…"#` is a raw string, so the quotes inside need no backslashes.
`deny_unknown_fields` turns a typo in a client's request into a clear `422` later, instead of a silently
ignored field.

</details>

## Quiz

1. In REST, what should a URL contain: nouns or verbs? Where does the verb go?

<details>
<summary>Answer</summary>

Nouns (resources: `/tasks/7`). The verb is the HTTP method (`GET`, `POST`, `PATCH`, `DELETE`).

</details>

2. What is the difference between a *safe* and an *idempotent* method? Give one method that is idempotent
   but not safe.

<details>
<summary>Answer</summary>

Safe: it never changes anything (`GET`, `HEAD`). Idempotent: sending it many times has the same effect as
sending it once. `DELETE` (and `PUT`) are idempotent but not safe: they change data, but repeating them
changes nothing more.

</details>

3. Why is it dangerous to retry a `POST /tasks` automatically after a timeout?

<details>
<summary>Answer</summary>

The first request may have succeeded and only the answer was lost. `POST` is not idempotent, so the retry
creates a second, duplicate task.

</details>

4. Which status code fits: (a) the JSON body is cut in the middle; (b) the JSON is fine but the title is
   empty; (c) a database query fails because of a bug?

<details>
<summary>Answer</summary>

(a) `400 Bad Request`. (b) `422 Unprocessable Content`. (c) `500 Internal Server Error`, with no internal
details in the body.

</details>

5. What is the difference between `401` and `403`?

<details>
<summary>Answer</summary>

`401`: the server does not know who you are (no or invalid credentials). `403`: it knows who you are, and
you are not allowed to do this.

</details>

6. Which of these changes break existing clients: (a) adding an optional field `due_date` to the task JSON;
   (b) renaming `done` to `completed`; (c) adding the endpoint `GET /tasks/{id}/comments`?

<details>
<summary>Answer</summary>

Only (b). Old clients still read `done` and now find nothing. Adding a field or an endpoint is safe,
because old clients simply ignore what they do not know.

</details>

## Summary

- REST designs an API as resources with URLs (nouns) and acts on them with HTTP methods (verbs).
- `GET` reads (safe), `POST` creates (not idempotent), `PUT` replaces, `PATCH` changes some fields,
  `DELETE` removes (idempotent).
- Status codes tell the result: `2xx` success, `4xx` the client's mistake, `5xx` the server's failure. Never
  answer an error with `200`.
- Use one JSON error format everywhere; never put internal details into a `500` answer.
- Plural nouns, lowercase paths, query strings for filters and pages, one naming style for JSON fields,
  RFC 3339 dates.
- Write the contract before the code and protect it with tests that compare the exact JSON.

## Sources and further reading

- [RFC 9110 — HTTP Semantics](https://www.rfc-editor.org/rfc/rfc9110) — the official definition of methods (section 9), safe and idempotent methods, and status codes (section 15).
- [RFC 9457 — Problem Details for HTTP APIs](https://www.rfc-editor.org/rfc/rfc9457) — the standard JSON error format.
- [MDN — HTTP response status codes](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Status) — a readable list of every status code.
- [Microsoft REST API Guidelines](https://github.com/microsoft/api-guidelines) — a long, practical set of API design rules used by a large company.
- [OpenAPI Specification](https://spec.openapis.org/oas/latest.html) — the standard format for describing REST APIs.
- [serde — container attributes](https://serde.rs/container-attrs.html) — `deny_unknown_fields`, `rename_all` and the other attributes for structs.

---

[🏠 Start](../START-HERE.md) · [← 16.1 What a backend service is](01-what-a-backend-service-is.md) · [📋 Phase 16](README.md) · [16.3 Web services with axum →](03-web-services-with-axum.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
