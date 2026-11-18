<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 10.6 Аутентификация и хеширование паролей](06-authentication-and-password-hashing.md) · [📋 Этап 10](README.md) · [10.8 Управление секретами →](08-secrets-management.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/10-security-fundamentals/07-authorization-sessions-and-tokens.md) <!-- nav -->

# Урок 10.7 — Авторизация, сессии и токены

> **Этап 10 — Основы безопасности** · Урок 7 из 15 · ⏱ около 135 минут

## Что вы узнаете

- Как решать, что может делать вошедший пользователь: роли, владение и «запрет по умолчанию».
- Что такое сессия и как создавать токены сессий, которые нельзя угадать, с тайм-аутами и выходом.
- Что такое cookie и какие атрибуты делают её безопаснее.
- Что такое JSON Web Token (JWT) (JSON, JavaScript Object Notation, — распространённый текстовый формат данных), как он подписывается и какие проверки нельзя пропускать.
- Когда выбирать серверную сессию, а когда токен.

## Перед началом

- Вы прошли [Урок 10.6 — Аутентификация и хеширование паролей](06-authentication-and-password-hashing.md).
- Вы знаете перечисления и `match` с кортежами ([Урок 04.4](../04-structs-enums-collections-errors/04-option-and-match.md)), `HashMap` и derive для `serde` ([Урок 05.11](../05-rust-intermediate/11-practical-crates-serde-clap-errors.md)).
- Создайте пакет: `cargo new --lib code/p10-authz`, затем внутри него выполните `cargo add getrandom@0.4 hex@0.4 base64@0.23 sha2@0.11`, `cargo add serde@1 --features derive`, `cargo add jsonwebtoken@12 --features rust_crypto` и `mkdir src/bin`.
- Крейт `jsonwebtoken` (версия 12) требует выбрать криптографический бэкенд через фичу (`rust_crypto` написан на Rust; `aws_lc_rs` — другой). Без бэкенда подпись не работает.

## Новые слова

| Слово | Значение |
|---|---|
| **роль** (role) | Именованный набор прав, выдаваемый пользователям, например *viewer*, *editor*, *admin*. |
| **RBAC** (Role-Based Access Control, управление доступом на основе ролей) | Решение о доступе по роли пользователя. |
| **ACL** (Access Control List, список контроля доступа) | Список на объекте, который говорит, какие пользователи что могут с ним делать. |
| **ABAC** (Attribute-Based Access Control, управление доступом на основе атрибутов) | Решение о доступе по многим фактам (отдел пользователя, время суток, метка данных), а не только по роли. |
| **запрет по умолчанию** (deny by default) | Если ни одно правило не разрешает действие, оно отклоняется. |
| **сессия** (session) | Память сервера о том, что «этот браузер принадлежит Ане, которая вошла 5 минут назад». |
| **токен сессии** (session token, или session ID) | Случайный секретный текст, который браузер отправляет с каждым запросом, чтобы сервер мог найти сессию. |
| **cookie** | Небольшой текст, который сервер просит браузер сохранить и отправлять обратно с каждым следующим запросом к тому же сайту. |
| **CSPRNG** (Cryptographically Secure Pseudo-Random Number Generator, криптографически стойкий генератор псевдослучайных чисел) | Источник случайных чисел, который атакующий не может предсказать. Его предоставляет операционная система. Урок 10.9. |
| **фиксация сессии** (session fixation) | Атака, при которой атакующий даёт жертве токен сессии, который он уже знает, и ждёт, пока жертва войдёт. |
| **CSRF** (Cross-Site Request Forgery, подделка межсайтовых запросов) | Атака, при которой вредоносная страница заставляет браузер жертвы отправить запрос на другой сайт, и браузер автоматически добавляет cookie жертвы. |
| **предъявительский токен** (bearer token) | Токен, дающий доступ тому, кто его *предъявляет* (держит). Любой, кто его скопировал, может им пользоваться. |
| **JWT** (JSON Web Token) | Подписанный токен из трёх частей в Base64, `header.payload.signature`, который несёт утверждения, например кто пользователь и когда токен истекает. |
| **утверждение** (claim) | Одно высказывание внутри токена: `sub` (субъект, пользователь), `exp` (время истечения), `role`. |
| **подпись** (signature) | Значение, вычисленное из данных и секретного ключа, чтобы любой, у кого есть ключ, мог обнаружить любое изменение данных. |
| **HMAC** (Hash-based Message Authentication Code, код аутентификации сообщения на основе хеша) | Способ подписать данные общим секретным ключом. HS256 (HMAC-SHA-256) означает HMAC с хеш-функцией SHA-256 (Secure Hash Algorithm, 256-bit, безопасный алгоритм хеширования). Урок 10.9. |
| **HTTP** (HyperText Transfer Protocol, протокол передачи гипертекста) | Протокол, по которому браузеры и веб-серверы общаются друг с другом. |
| **HTTPS** (HTTP Secure) | HTTP внутри шифрованного соединения TLS (Transport Layer Security, протокол защиты транспортного уровня) (Урок 10.11). |
| **Base64** | Способ записать любые байты буквами, цифрами и несколькими знаками. Это кодирование, а не шифрование. |

## 1. Авторизация: что вам можно?

После аутентификации ([Урок 10.6](06-authentication-and-password-hashing.md)) программа знает, **кто** этот
пользователь. **Авторизация** решает, **что** ему можно делать. Нарушенная авторизация стоит на первом месте в
списке OWASP (Open Worldwide Application Security Project, открытого всемирного проекта по безопасности приложений; [Урок 10.3](03-the-owasp-top-10.md)). Четыре правила предотвращают большинство проблем:

1. **Проверяйте на сервере, при каждом запросе.** Никогда не доверяйте тому, что клиент говорит о своей роли.
2. **Запрет по умолчанию.** У нового действия или новой роли нет прав, пока вы их не добавите.
3. **Проверяйте объект, а не только роль.** «Редакторы могут править заметки» недостаточно; нужно «редакторы
   могут править **свои** заметки».
4. **Наименьшие привилегии.** Давайте каждой роли минимальный набор прав, достаточный для работы.

Распространённые модели: **ACL** (список на каждом объекте), **RBAC** (права принадлежат ролям, пользователям
выдаются роли) и **ABAC** (правила по многим атрибутам). RBAC плюс проверка владения достаточно для большинства
программ.

Вот небольшой RBAC с владением. Сохраните начало `src/lib.rs`:

```rust
use std::collections::HashMap;

use sha2::{Digest, Sha256};

// ---------- Authorization: who may do what ----------

#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum Role {
    Viewer,
    Editor,
    Admin,
}

#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum Action {
    Read,
    Write,
    Delete,
    ManageUsers,
}

pub struct User {
    pub name: String,
    pub role: Role,
}

#[derive(Debug, PartialEq, Eq)]
pub struct Denied;

/// `owner` is the owner of the object (a note, for example), if the action is about one.
pub fn authorize(user: &User, action: Action, owner: Option<&str>) -> Result<(), Denied> {
    let is_owner = owner == Some(user.name.as_str());
    let allowed = match (user.role, action) {
        (Role::Admin, _) => true,
        (Role::Editor, Action::Read) => true,
        (Role::Editor, Action::Write | Action::Delete) => is_owner,
        (Role::Viewer, Action::Read) => true,
        _ => false, // deny by default: anything not listed above is refused
    };
    if allowed { Ok(()) } else { Err(Denied) }
}
```

**Что происходит:**

1. `Role` и `Action` — перечисления, поэтому каждая роль и каждое действие — именованное проверяемое значение, а не
   строка, в которой можно ошибиться.
2. `authorize` получает пользователя, действие и владельца объекта (`Option<&str>`, потому что у некоторых
   действий, например управления пользователями, владельца нет).
3. `is_owner` истинно, только когда владелец существует **и** равен имени пользователя. `owner == Some(...)`
   сравнивает значения `Option`, поэтому `None` никогда не совпадает.
4. `match` по паре `(role, action)` перечисляет каждую разрешённую комбинацию. Администратор может всё.
   Редактор может читать, а писать и удалять — только свои объекты. Наблюдатель может только читать.
5. Последняя ветка `_ => false` — это **запрет по умолчанию**: всё, что не перечислено, отклоняется. (Компромисс:
   если перечислять все случаи без `_`, компилятор сообщит, когда вы добавите новую роль и забудете её. Некоторые
   команды предпочитают так. В любом случае результат по умолчанию должен быть «нет».)
6. Функция возвращает `Result<(), Denied>`, поэтому вызывающий код не может забыть посмотреть на ответ, не получив
   предупреждения компилятора о неиспользованном `Result`.

> 🔒 **Безопасность:** держите авторизацию в **одной** такой функции, которую вызывает каждый обработчик. Правила,
> скопированные в десять мест, не изменятся во всех десяти, когда изменится политика.

## 2. Сессии

HTTP, язык веба, **не хранит состояния**: каждый запрос приходит сам по себе, и сервер не помнит предыдущий. После
входа сервер должен узнать того же пользователя в следующем запросе. Он делает это с помощью **сессии**:

```text
1. Browser:  POST /login  (name + password)
2. Server:   checks the password, creates a session, makes a random token
             answers: Set-Cookie: session=<token>
3. Browser:  GET /notes   Cookie: session=<token>
4. Server:   finds the session by the token  ->  "this is Ana"
5. Browser:  POST /logout ...  Server: deletes the session
```

Токен — это *ключ* к сессии, поэтому он должен быть секретным и неугадываемым. Требования:

- **Непредсказуемость.** Не менее 128 бит из CSPRNG (генератора случайных чисел операционной системы). Никогда не
  стройте его из имени пользователя, времени, счётчика или генератора случайных чисел, инициализированного
  текущим временем.
- **Состояние на сервере.** Сервер сам решает, действительна ли сессия, и может немедленно её завершить (выход,
  смена пароля, «выйти со всех устройств»).
- **Тайм-ауты.** *Тайм-аут простоя* завершает сессию, которой некоторое время не пользовались (например, 15
  минут), а *абсолютный тайм-аут* завершает её по достижении максимального возраста, даже если ею пользуются
  постоянно (например, 8 часов).
- **Новый токен при входе и при смене привилегий.** Иначе возможна *фиксация сессии*.
- **Не хранится в читаемом виде.** Если таблица сессий утекла, читаемые токены становятся рабочими входами.
  Храните **хеш** токена (здесь SHA-256 подходит, потому что токен случайный и длинный, а не пароль человека).

Продолжите `src/lib.rs`:

```rust
// ---------- Sessions: remembering who is logged in ----------

pub struct Session {
    pub user: String,
    created: u64,
    last_seen: u64,
}

pub struct SessionStore {
    sessions: HashMap<String, Session>, // key: SHA-256 of the token, never the token itself
    idle_timeout: u64,
    absolute_timeout: u64,
}

fn new_token() -> String {
    let mut bytes = [0u8; 32]; // 32 random bytes = 256 bits
    getrandom::fill(&mut bytes).expect("the operating system has no random numbers");
    hex::encode(bytes)
}

fn key_of(token: &str) -> String {
    hex::encode(Sha256::digest(token.as_bytes()))
}

impl SessionStore {
    /// Times are in seconds. `now` is passed in by the caller, which makes tests easy.
    pub fn new(idle_timeout: u64, absolute_timeout: u64) -> Self {
        SessionStore {
            sessions: HashMap::new(),
            idle_timeout,
            absolute_timeout,
        }
    }

    /// Creates a session after a successful login and returns the token for the cookie.
    pub fn create(&mut self, user: &str, now: u64) -> String {
        let token = new_token();
        let session = Session {
            user: user.to_string(),
            created: now,
            last_seen: now,
        };
        self.sessions.insert(key_of(&token), session);
        token
    }

    /// Returns the user if the token is valid now, and renews the idle timer.
    pub fn check(&mut self, token: &str, now: u64) -> Option<String> {
        let key = key_of(token);
        let session = self.sessions.get_mut(&key)?;
        let idle = now.saturating_sub(session.last_seen);
        let age = now.saturating_sub(session.created);
        if idle > self.idle_timeout || age > self.absolute_timeout {
            self.sessions.remove(&key);
            return None;
        }
        session.last_seen = now;
        Some(session.user.clone())
    }

    /// Logout: the token stops working at once, on the server side.
    pub fn revoke(&mut self, token: &str) {
        self.sessions.remove(&key_of(token));
    }
}
```

**Что происходит:**

1. `new_token` заполняет 32 байта через `getrandom::fill`, который запрашивает у операционной системы
   криптографически стойкие случайные байты. `hex::encode` записывает их как 64 символа `0-9a-f`. Это 256 бит,
   более чем достаточно.
2. `key_of` хеширует токен через `Sha256::digest`. Ключом словаря служит хеш, поэтому таблица никогда не содержит
   пригодного к использованию токена.
3. `SessionStore::new` принимает два тайм-аута. Время — это обычные числа секунд, а `now` — **параметр**. Функцию,
   которая сама читает часы, трудно тестировать; эту легко.
4. `create` создаёт токен и сохраняет сессию. Вызывающий код отправляет токен браузеру.
5. `check` ищет сессию по хешу переданного токена. `?` возвращает `None` для неизвестного токена.
   `saturating_sub` избегает отрицательного результата, если часы идут назад. Если сессия простаивала слишком
   долго или слишком стара, она **удаляется** и возвращается `None`. Иначе `last_seen` обновляется: таймер простоя
   начинается заново.
6. `revoke` — это выход. Поскольку состояние на сервере, он работает мгновенно.

Посмотрим, как это работает. Сохраните `src/bin/session.rs` и запустите `cargo run --bin session`:

```rust
use p10_authz::SessionStore;

fn main() {
    // idle timeout 15 minutes, absolute timeout 8 hours (times in seconds)
    let mut store = SessionStore::new(15 * 60, 8 * 3600);

    let token = store.create("ana", 0);
    println!(
        "token: {} hex characters = {} bits",
        token.len(),
        token.len() * 4
    );

    println!("after 10 minutes:       {:?}", store.check(&token, 10 * 60));
    println!("after 20 minutes:       {:?}", store.check(&token, 20 * 60));
    println!("after 40 minutes:       {:?}", store.check(&token, 40 * 60));

    let token = store.create("ana", 100);
    store.revoke(&token);
    println!("after logout:           {:?}", store.check(&token, 101));
    println!(
        "a guessed token:        {:?}",
        store.check(&"0".repeat(64), 101)
    );
}
```

```text
token: 64 hex characters = 256 bits
after 10 minutes:       Some("ana")
after 20 minutes:       Some("ana")
after 40 minutes:       None
after logout:           None
a guessed token:        None
```

Сессия переживает 10 минут, а проверка на 20-й минуте всё ещё проходит, потому что таймер простоя был обновлён на
10-й минуте (простоя только 10 минут). После ещё 20 минут простоя (40-я минута) она истекла. После `revoke` токен
мёртв, а угаданный токен просто неизвестен.

Добавьте тесты в конец `src/lib.rs`:

```rust
#[cfg(test)]
mod tests {
    use super::*;

    fn user(name: &str, role: Role) -> User {
        User {
            name: name.to_string(),
            role,
        }
    }

    #[test]
    fn roles_and_ownership() {
        let ana = user("ana", Role::Editor);
        assert_eq!(authorize(&ana, Action::Write, Some("ana")), Ok(()));
        assert_eq!(authorize(&ana, Action::Write, Some("bob")), Err(Denied));
        assert_eq!(authorize(&ana, Action::Read, Some("bob")), Ok(()));
        assert_eq!(authorize(&ana, Action::ManageUsers, None), Err(Denied));

        let vera = user("vera", Role::Viewer);
        assert_eq!(authorize(&vera, Action::Read, None), Ok(()));
        assert_eq!(authorize(&vera, Action::Delete, Some("vera")), Err(Denied));

        let root = user("root", Role::Admin);
        assert_eq!(authorize(&root, Action::ManageUsers, None), Ok(()));
    }

    #[test]
    fn a_session_works_until_it_expires() {
        let mut store = SessionStore::new(60, 600);
        let token = store.create("ana", 1_000);
        assert_eq!(store.check(&token, 1_030), Some("ana".to_string()));
        // 30 s later it is still fine, because the last check renewed the idle timer:
        assert_eq!(store.check(&token, 1_060), Some("ana".to_string()));
        // 61 s of silence: the idle timeout (60 s) has passed.
        assert_eq!(store.check(&token, 1_121), None);
        assert_eq!(store.check(&token, 1_122), None);
    }

    #[test]
    fn logout_and_guessing() {
        let mut store = SessionStore::new(60, 600);
        let token = store.create("ana", 0);
        assert_eq!(store.check("not-a-real-token", 1), None);
        store.revoke(&token);
        assert_eq!(store.check(&token, 2), None);
    }
}
```

```bash
cargo test
```

```text
running 3 tests
test tests::a_session_works_until_it_expires ... ok
test tests::logout_and_guessing ... ok
test tests::roles_and_ownership ... ok

test result: ok. 3 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.01s
```

> 🧪 **Тестирование:** поскольку часы — параметр, тесты могут «путешествовать во времени» за микросекунду. Когда
> вашему коду нужно время, передавайте его, а не вызывайте `SystemTime::now()` глубоко внутри.

### Cookie

Сервер передаёт токен браузеру в заголовке, а браузер отправляет его обратно. Заголовок выглядит так:

```text
Set-Cookie: session=<token>; HttpOnly; Secure; SameSite=Lax; Path=/; Max-Age=900
```

| Атрибут | Что делает |
|---|---|
| `HttpOnly` | JavaScript на странице не может прочитать cookie, поэтому ошибка XSS (cross-site scripting, межсайтовый скриптинг; [Урок 10.5](05-injection-attacks.md)) не может её украсть. |
| `Secure` | Браузер отправляет cookie только по HTTPS ([Урок 10.11](11-tls-and-certificates.md)). |
| `SameSite=Lax` (или `Strict`) | Браузер не добавляет cookie к большинству запросов, начатых другими сайтами. Это блокирует большинство **CSRF**. |
| `Path`, `Max-Age` | Где применяется cookie и как долго браузер её хранит. |

Для действий, меняющих данные, добавьте дополнительную защиту от CSRF (секретный токен внутри формы), если
используемый вами фреймворк ещё не делает этого за вас.

## 3. Токены и JWT

Сессии нужна таблица на сервере. **Токен**, например **JWT**, сам несёт факты и **подписан**, поэтому серверу
таблица не нужна: он проверяет подпись. JWT выглядит как `xxxxx.yyyyy.zzzzz`:

```text
header.payload.signature        each part is Base64URL (letters, digits, - and _)
header:    {"typ":"JWT","alg":"HS256"}              which algorithm signed it
payload:   {"sub":"ana","role":"viewer","exp":...}  the claims
signature: HMAC-SHA256(secret, header + "." + payload)
```

Сохраните `src/bin/jwt.rs`. Он создаёт токен, читает его и затем атакует тремя способами:

```rust
use base64::Engine;
use base64::engine::general_purpose::URL_SAFE_NO_PAD;
use jsonwebtoken::{Algorithm, DecodingKey, EncodingKey, Header, Validation, decode, encode};
use serde::{Deserialize, Serialize};

#[derive(Debug, Serialize, Deserialize)]
struct Claims {
    sub: String,  // subject: who the token is about
    role: String, // our own claim
    exp: u64,     // expiry: seconds since 1970-01-01 (the "Unix time")
}

fn validation() -> Validation {
    let mut validation = Validation::new(Algorithm::HS256); // ONLY this algorithm is accepted
    validation.leeway = 0; // the default allows 60 seconds of clock difference
    validation
}

fn main() {
    // A demo secret. A real one is long, random and kept outside the code (Lesson 10.8).
    let secret = b"demo-secret-do-not-use-in-real-life";
    let claims = Claims {
        sub: "ana".into(),
        role: "viewer".into(),
        exp: 4_000_000_000,
    };

    let token = encode(
        &Header::new(Algorithm::HS256),
        &claims,
        &EncodingKey::from_secret(secret).expect("bad key"),
    )
    .expect("encoding failed");
    println!("token: {token}");

    // The first two parts are only Base64: anybody can read them.
    let parts: Vec<&str> = token.split('.').collect();
    for (name, part) in ["header ", "payload"].iter().zip(&parts) {
        let bytes = URL_SAFE_NO_PAD.decode(part).expect("not base64");
        println!("{name}: {}", String::from_utf8_lossy(&bytes));
    }

    let key = DecodingKey::from_secret(secret).expect("bad key");
    let ok = decode::<Claims>(&token, &key, &validation());
    println!("valid token:    {:?}", ok.map(|data| data.claims.role));

    // Attack 1: change "viewer" to "admin" in the payload, keep the old signature.
    let payload = String::from_utf8(URL_SAFE_NO_PAD.decode(parts[1]).unwrap()).unwrap();
    let forged_payload = URL_SAFE_NO_PAD.encode(payload.replace("viewer", "admin"));
    let forged = format!("{}.{}.{}", parts[0], forged_payload, parts[2]);
    let result = decode::<Claims>(&forged, &key, &validation());
    println!(
        "forged payload: {:?}",
        result
            .map(|data| data.claims.role)
            .map_err(|e| e.into_kind())
    );

    // Attack 2: a header that says "no signature needed".
    let none_header = URL_SAFE_NO_PAD.encode(r#"{"alg":"none","typ":"JWT"}"#);
    let unsigned = format!("{none_header}.{}.", parts[1]);
    let result = decode::<Claims>(&unsigned, &key, &validation());
    println!(
        "alg none:       {:?}",
        result
            .map(|data| data.claims.role)
            .map_err(|e| e.into_kind())
    );

    // Attack 3: an old token (expired in the year 2001).
    let old = Claims {
        sub: "ana".into(),
        role: "viewer".into(),
        exp: 1_000_000_000,
    };
    let old_token = encode(
        &Header::new(Algorithm::HS256),
        &old,
        &EncodingKey::from_secret(secret).expect("bad key"),
    )
    .expect("encoding failed");
    let result = decode::<Claims>(&old_token, &key, &validation());
    println!(
        "expired token:  {:?}",
        result
            .map(|data| data.claims.role)
            .map_err(|e| e.into_kind())
    );
}
```

Запустите `cargo run --bin jwt`:

```text
token: eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJzdWIiOiJhbmEiLCJyb2xlIjoidmlld2VyIiwiZXhwIjo0MDAwMDAwMDAwfQ.crrEcjh-LPhucxLldtUAxBdYYhcCnwlsdsjBUcLKdzA
header : {"typ":"JWT","alg":"HS256"}
payload: {"sub":"ana","role":"viewer","exp":4000000000}
valid token:    Ok("viewer")
forged payload: Err(InvalidSignature)
alg none:       Err(Json(Error("unknown variant `none`, expected one of `HS256`, `HS384`, `HS512`, `ES256`, `ES384`, `RS256`, `RS384`, `RS512`, `PS256`, `PS384`, `PS512`, `EdDSA`", line: 1, column: 13)))
expired token:  Err(ExpiredSignature)
```

**Что происходит:**

1. `Claims` — обычная структура с `Serialize` и `Deserialize`. `exp` — **время истечения** в секундах с 1 января
   1970 года («Unix-время»). `4_000_000_000` приходится на 2096 год, поэтому токен действителен.
2. `Validation::new(Algorithm::HS256)` говорит: **принимать только HS256**. По умолчанию `leeway` — 60 секунд
   допустимой разницы часов; мы ставим 0, чтобы тест был точным. `exp` проверяется автоматически.
3. `encode` подписывает утверждения секретом. В версии 12 `EncodingKey::from_secret` возвращает `Result`, поэтому
   мы вызываем `.expect(...)`.
4. Программа делит токен по точкам и декодирует первые две части. **Полезную нагрузку может прочитать любой**
   (это всего лишь Base64). Подпись защищает от *изменения*, а не от *чтения*. Никогда не кладите секреты в JWT.
5. **Атака 1.** Заменить `viewer` на `admin` в нагрузке и оставить подпись. Подпись была вычислена для старой
   нагрузки, поэтому больше не совпадает: `InvalidSignature`. Без секрета атакующий не может сделать новую
   действительную подпись.
6. **Атака 2** — печально известный `"alg":"none"`: заголовок, утверждающий, что подпись не нужна. Некоторые старые
   библиотеки это принимали. Здесь алгоритм в заголовке должен быть одним из разрешённых нами, а `none` даже не
   является известным алгоритмом, поэтому декодирование завершается ошибкой.
7. **Атака 3**: токен, истёкший в 2001 году, даёт `ExpiredSignature`.

**Проверки, которые нельзя пропускать:**

- Выбирайте **алгоритм** на сервере (`Validation::new(Algorithm::HS256)`); никогда не верьте `alg` из токена.
- Всегда проверяйте **`exp`**, а также `iss` (издатель) и `aud` (аудитория), когда токены разделяют несколько сервисов.
- Используйте **длинный случайный секрет** (не менее 32 случайных байт), хранимый вне кода ([Урок 10.8](08-secrets-management.md)).
- Делайте токены **недолговечными** (минуты). JWT нельзя отменить до истечения срока, если не вести чёрный список,
  что возвращает состояние на сервер.
- Не храните **секреты и чувствительные данные** в нагрузке.
- Отправляйте токены только по HTTPS и обращайтесь с ними как с паролями: предъявительский токен работает для
  каждого, кто его держит.

## 4. Сессия или токен?

Для классического веб-сайта с браузером **серверная сессия** с cookie — более простой и безопасный выбор: её можно
мгновенно отозвать (выход, смена пароля, украденное устройство), а cookie небольшая. **Подписанный токен**
хорош, когда один вход должны доверять многие сервисы, потому что каждый сервис может проверить подпись без общей
таблицы; цена — его нельзя отменить до истечения срока, и он добавляет больше правил, которым нужно следовать.

## Частые ошибки

> ⚠️ **Частая ошибка:** проверять роль только в пользовательском интерфейсе. Сервер должен проверять каждый запрос.

> ⚠️ **Частая ошибка:** ID сессии, сделанный из счётчика, времени или генератора, инициализированного временем.
> Их можно угадать. Используйте байты из операционной системы.

> ⚠️ **Частая ошибка:** вызывать `decode` без фиксированного алгоритма, без `exp` или с секретом `"secret"`.

## Попробуйте сами

1. Наберите библиотеку, обе программы и тесты. Запустите `cargo test`, `cargo run --bin session` и `cargo run --bin jwt`.
2. В `jwt.rs` верните `validation.leeway` к значению по умолчанию (удалите строку) и дайте старому токену `exp`,
   равный текущему времени минус 30 секунд. Токен всё ещё проходит? Почему?
3. В `session.rs` сделайте тайм-аут простоя 5 минут. На какой из трёх проверок сессия закончится?

## Упражнения

### Упражнение 1 — Найдите недостающую проверку (легко)

Чего не хватает? (a) `if user.role == Role::Editor { delete(note) }`. (b) Обработчик читает роль из JSON-поля
в теле запроса. (c) Используется `Validation::default()`, но токены подписаны RS256
(подпись RSA с SHA-256: закрытый ключ подписывает, открытый проверяет; RSA назван по фамилиям изобретателей — Rivest,
Shamir и Adleman, а подписи объясняет Урок 10.10). (d) Токен сессии — это id
пользователя.

<details>
<summary>Подсказка</summary>

Подумайте, какое правило из раздела 1 или раздела 3 нарушает каждая строка.

</details>

<details>
<summary>Ответ</summary>

(a) Проверки владения: любой редактор может удалить любую заметку. (b) Клиент сам выбирает свою роль: роль должна
приходить от сервера (из сессии или проверенного токена). (c) Сервер не назвал алгоритм, который принимает. По
умолчанию это HS256, поэтому токены RS256 отклоняются; всегда задавайте его явно, например
`Validation::new(Algorithm::RS256)`. (d) Токен можно угадать: id — маленькие числа; используйте 256 случайных бит.

</details>

### Упражнение 2 — Абсолютный тайм-аут (средне)

Напишите **тест**, доказывающий, что сессия, которой пользуются каждые 30 секунд, всё равно заканчивается, когда
становится старше абсолютного тайм-аута (600 секунд).

<details>
<summary>Подсказка</summary>

Создайте хранилище `SessionStore::new(60, 600)`, вызывайте `check` для `t` из `(30..=600).step_by(30)` и ожидайте
`Some`, затем проверьте на 630.

</details>

<details>
<summary>Ответ</summary>

```rust
    #[test]
    fn the_absolute_timeout_ends_even_a_busy_session() {
        let mut store = SessionStore::new(60, 600);
        let token = store.create("ana", 0);
        for t in (30..=600).step_by(30) {
            assert!(store.check(&token, t).is_some());
        }
        assert_eq!(store.check(&token, 630), None);
    }
```

Поместите его внутрь `mod tests`. Почему это работает: каждая проверка обновляет таймер простоя, поэтому лимит простоя
никогда не срабатывает; на 630 секундах возраст больше 600, и сессию может завершить только абсолютный тайм-аут.

</details>

### Упражнение 3 — Смените токен (сложно)

Добавьте в `SessionStore` метод `rotate(&mut self, old_token: &str, now: u64) -> Option<String>`. Он должен выдать
тому же пользователю **новый** токен и сделать старый бесполезным. Напишите **тест**.

<details>
<summary>Подсказка</summary>

`check` возвращает имя пользователя (или `None` для недействительного токена), затем `revoke` старого токена и
`create` нового.

</details>

<details>
<summary>Ответ</summary>

```rust
    /// Gives the user a new token and kills the old one (use it after login or a role change).
    pub fn rotate(&mut self, old_token: &str, now: u64) -> Option<String> {
        let user = self.check(old_token, now)?;
        self.revoke(old_token);
        Some(self.create(&user, now))
    }
```

```rust
    #[test]
    fn rotate_replaces_the_token() {
        let mut store = SessionStore::new(60, 600);
        let old = store.create("ana", 0);
        let new = store.rotate(&old, 10).unwrap();
        assert_ne!(old, new);
        assert_eq!(store.check(&old, 11), None);
        assert_eq!(store.check(&new, 11), Some("ana".to_string()));
        assert_eq!(store.rotate("bogus", 12), None);
    }
```

Почему это работает: после `rotate` атакующий, знавший старый токен (фиксация сессии), держит токен, которого
больше не существует, а недействительный токен нельзя сменить на действительный.

</details>

## Вопросы для самопроверки

1. Назовите четыре правила авторизации из раздела 1.

<details>
<summary>Ответ</summary>

Проверять на сервере при каждом запросе; запрет по умолчанию; проверять объект (владение), а не только роль;
наименьшие привилегии.

</details>

2. Почему хранилище сессий хранит хеш токена, а не сам токен?

<details>
<summary>Ответ</summary>

Если таблица утечёт, хеши нельзя использовать как токены, поэтому утечка не даёт атакующим рабочих сессий.

</details>

3. Секретна ли нагрузка JWT? Что её защищает?

<details>
<summary>Ответ</summary>

Нет, это всего лишь Base64, и её может прочитать любой. Подпись защищает её от изменения, но не от чтения.

</details>

4. Почему алгоритм должен выбирать сервер, а не токен?

<details>
<summary>Ответ</summary>

Токеном управляет атакующий. Если бы сервер верил `alg` в заголовке, атакующий мог бы запросить `none` или более
слабый алгоритм.

</details>

## Итоги

- Авторизация: проверка на сервере при каждом запросе, запрет по умолчанию, проверка владения наряду с ролью, наименьшие привилегии.
- Держите политику в одной функции и возвращайте `Result`, чтобы её нельзя было проигнорировать.
- Токен сессии — это 128 и более случайных бит из операционной системы; сервер хранит состояние, хеш токена, тайм-ауты простоя и абсолютный.
- Меняйте токен при входе; выход удаляет сессию на сервере.
- Cookie: `HttpOnly`, `Secure`, `SameSite`.
- JWT подписан, но не зашифрован: фиксируйте алгоритм, проверяйте `exp`, делайте его недолговечным, без секретов внутри.
- Передавайте время в функции, чтобы правила, зависящие от времени, можно было тестировать.

## Источники и что почитать

- [OWASP Authorization Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authorization_Cheat_Sheet.html) — запрет по умолчанию, наименьшие привилегии и проверки при каждом запросе.
- [OWASP Session Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Session_Management_Cheat_Sheet.html) — длина токена, cookie, тайм-ауты и фиксация сессии.
- [OWASP CSRF Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Cross-Site_Request_Forgery_Prevention_Cheat_Sheet.html) — как работает подделка межсайтовых запросов и как её остановить.
- [MDN — Set-Cookie header](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Set-Cookie) — все атрибуты cookie с объяснениями, на MDN (Mozilla Developer Network, справочный сайт по веб-технологиям).
- [RFC 7519 — JSON Web Token (JWT)](https://www.rfc-editor.org/rfc/rfc7519) — стандарт, определяющий токен и его утверждения (RFC, Request for Comments, «запрос комментариев», — документ официальной серии интернет-стандартов).
- [RFC 8725 — JWT Best Current Practices](https://www.rfc-editor.org/rfc/rfc8725) — список известных атак и проверок, которые их останавливают.
- [Документация крейта `jsonwebtoken`](https://docs.rs/jsonwebtoken/12.0.0/jsonwebtoken/) — `encode`, `decode`, `Validation` и фичи бэкенда.
- [Документация крейта `getrandom`](https://docs.rs/getrandom/0.4.3/getrandom/) — безопасные случайные байты от операционной системы.

---

[🏠 Начало](../START-HERE.md) · [← 10.6 Аутентификация и хеширование паролей](06-authentication-and-password-hashing.md) · [📋 Этап 10](README.md) · [10.8 Управление секретами →](08-secrets-management.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
