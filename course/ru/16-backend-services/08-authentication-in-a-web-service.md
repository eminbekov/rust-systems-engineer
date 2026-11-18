<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 16.7 Миграции и транзакции](07-migrations-and-transactions.md) · [📋 Этап 16](README.md) · [16.9 gRPC на tonic →](09-grpc-with-tonic.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/16-backend-services/08-authentication-in-a-web-service.md) <!-- nav -->

# Урок 16.8 — Аутентификация в веб-сервисе

> **Этап 16 — Бэкенд-сервисы** · Урок 8 из 16 · ⏱ около 3 часов (2 занятия)

## Что вы узнаете

- Как веб-сервис позволяет пользователям зарегистрироваться и войти, используя то, чему Этап 10 научил о паролях и токенах.
- Как хранить пароли с Argon2 и почему хеширование выполняется в `spawn_blocking`.
- Как создавать случайные токены сессий, хранить только их хеши и задавать им срок действия.
- Как написать экстрактор axum `AuthUser`, который заставляет любой обработчик требовать вход, просто будучи его параметром.
- Как тестировать регистрацию, вход и защищённые эндпоинты.

## Перед началом

- Вы прошли [Урок 16.7 — Миграции и транзакции](07-migrations-and-transactions.md).
- Вы знаете идеи Урока 10.6 (Аутентификация и хеширование паролей) и Урока 10.7 (Авторизация, сессии и токены):
  никогда не храните пароли, храните медленные хеши с солью; токен сессии — случайный секрет, который заменяет вход.
  Раздел 1 повторяет главное.
- Вы знаете хеши и безопасную случайность из [Урока 10.9 — Криптография I](../10-security-fundamentals/09-cryptography-hashes-macs-and-randomness.md).

## Новые слова

| Слово | Значение |
|---|---|
| **аутентификация** (authentication) | Выяснение того, **кто** отправил запрос («это Ана»). Её неудача — `401 Unauthorized`. |
| **авторизация** (authorization) | Решение, что этому человеку **можно** делать («Ана может менять свои задачи»). Её неудача — `403 Forbidden` (или `404`). |
| **учётные данные** (credentials) | То, чем пользователь доказывает свою личность: здесь email и пароль. |
| **Argon2** | Функция хеширования паролей, которая намеренно медленная и требует много памяти, так что подбор паролей обходится дорого. Рекомендуемый вариант — `Argon2id`. |
| **соль** (salt) | Случайные байты, подмешиваемые в каждый хеш пароля, чтобы у двух пользователей с одинаковым паролем получились разные хеши. |
| **строка PHC** (PHC string, Password Hashing Competition — конкурс хеширования паролей) | Текстовый формат хеша пароля, в котором есть всё нужное для его проверки: алгоритм, параметры, соль, хеш. Начинается с `$argon2id$`. |
| **токен сессии** (session token) | Длинный случайный секрет, который сервер выдаёт после входа. Клиент отправляет его с каждым запросом вместо пароля. |
| **bearer-токен** (bearer token) | Токен, отправляемый в заголовке `Authorization: Bearer <token>`. Тот, кто его «предъявляет» (bears), считается пользователем. |
| **срок действия** (expiry) | Момент, после которого токен перестаёт работать. |
| **перебор пользователей** (user enumeration) | Выяснение, какие учётные записи существуют, по тому, как отвечает сервис (разные сообщения, разное время). |

## 1. План

Этап 10 научил правилам; этот урок встраивает их в сервис задач:

```text
 1. POST /users     {"email","password"}  → hash the password (Argon2), store the user
 2. POST /sessions  {"email","password"}  → check the password, create a random token,
                                             store SHA-256(token) with an expiry, return the token once
 3. any request     Authorization: Bearer <token>
                    → AuthUser extractor: SHA-256(token) → find a valid session → user id
```

Важные решения:

- **Пароли никогда не хранятся**, только хеши Argon2. Украденная база данных их не раскрывает.
- **Токены случайные** (32 байта от операционной системы), поэтому угадать токен никто не может.
- **Хранится только хеш каждого токена.** Тот, кто прочитает таблицу `sessions`, не сможет использовать прочитанное.
  Здесь достаточно простого быстрого хеша (SHA-256, Secure Hash Algorithm, 256-bit, безопасный алгоритм хеширования, 256 бит), потому что токен — это 256 случайных бит, а не человеческий
  пароль, который можно подобрать.
- **У токенов есть срок действия** — семь дней, и каждая проверка сравнивает его с текущим временем.
- Чтение задач в этом уроке остаётся публичным; **создание, изменение, удаление и архивация требуют входа.**
  Упражнение 3 делает задачи личными для их владельца.

## 2. Пользователи и сессии в базе данных

Новые крейты: `argon2` для паролей, `sha2` для SHA-256, `getrandom` для безопасных случайных байтов, `hex`, чтобы
записывать байты текстом:

```bash
cd ~/projects/rust/code/p16-tasks
cargo add argon2@0.6 sha2@0.11 getrandom@0.4 hex@0.4
cargo sqlx migrate add create_users_and_sessions
```

Новая миграция:

```sql
-- Учётные записи и сессии входа.
CREATE TABLE users (
    id            BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    email         TEXT NOT NULL UNIQUE,
    password_hash TEXT NOT NULL,
    created_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Храним только SHA-256-хеш каждого токена, никогда сам токен.
CREATE TABLE sessions (
    token_hash BYTEA PRIMARY KEY,
    user_id    BIGINT NOT NULL REFERENCES users (id) ON DELETE CASCADE,
    expires_at TIMESTAMPTZ NOT NULL
);

CREATE INDEX sessions_user_id_idx ON sessions (user_id);
```

`UNIQUE` на `email` заставляет базу данных отклонить вторую учётную запись с тем же адресом, даже если две
регистрации придут в один и тот же момент. `BYTEA` — тип PostgreSQL для «сырых» байтов (`Vec<u8>` в Rust).
`ON DELETE CASCADE` завершает все сессии удалённого пользователя. Выполните `cargo sqlx migrate run`.

## 3. Ошибки для аутентификации

Добавьте три варианта в `ApiError` в `src/error.rs`:

```rust
    #[error("{0}")]
    Unauthorized(&'static str),
    #[error("{0}")]
    Conflict(String),
    #[error("internal error")]
    Database(#[from] sqlx::Error),
    #[error("internal error")]
    Internal(String),
```

В `status()`: `Unauthorized` → `StatusCode::UNAUTHORIZED`, `Conflict` → `StatusCode::CONFLICT`, а
`ApiError::Database(_) | ApiError::Internal(_)` → `StatusCode::INTERNAL_SERVER_ERROR`. В `code()`: `"unauthorized"`,
`"conflict"` и `"internal"` для обоих внутренних вариантов. В `into_response` пишите в лог оба внутренних варианта:

```rust
        // Подробности идут в лог, никогда клиенту.
        match &self {
            ApiError::Database(error) => tracing::error!(%error, "database error"),
            ApiError::Internal(error) => tracing::error!(%error, "internal error"),
            _ => {}
        }
```

`Internal` — для сбоев, которые не являются ошибками базы данных, например отказа источника случайных чисел. Как и
`Database`, он показывает клиенту только `internal error`.

## 4. Пароли и токены: `src/auth.rs`

Создайте `src/auth.rs`. Он начинается с импортов, JSON-типов (JavaScript Object Notation, нотация объектов JavaScript) и правил для ввода:

```rust
use argon2::Argon2;
use argon2::password_hash::{PasswordHasher, PasswordVerifier, phc::PasswordHash};
use axum::{
    Json,
    extract::{FromRequestParts, State},
    http::{StatusCode, header, request::Parts},
};
use serde::{Deserialize, Serialize};
use sha2::{Digest, Sha256};

use crate::AppState;
use crate::error::{ApiError, AppJson};

#[derive(Debug, Deserialize)]
pub struct Credentials {
    email: String,
    password: String,
}

#[derive(Debug, Serialize)]
pub struct User {
    id: i64,
    email: String,
}

#[derive(Debug, Serialize)]
pub struct Session {
    token: String,
}

/// Строчными буквами, без пробелов по краям, ровно с одним `@`, который не в начале и не в конце.
fn parse_email(text: &str) -> Result<String, ApiError> {
    let email = text.trim().to_lowercase();
    let valid = email.len() <= 254
        && email.matches('@').count() == 1
        && !email.starts_with('@')
        && !email.ends_with('@');
    if valid {
        Ok(email)
    } else {
        Err(ApiError::Validation("email is not valid".to_string()))
    }
}

fn check_password_rules(password: &str) -> Result<(), ApiError> {
    let chars = password.chars().count();
    if !(12..=128).contains(&chars) {
        return Err(ApiError::Validation(
            "password must have 12 to 128 characters".to_string(),
        ));
    }
    Ok(())
}
```

`use crate::AppState;` работает, потому что `AppState` определён в `main.rs` — корне крейта. Проверка email
намеренно простая (настоящая проверка email очень сложна; единственное настоящее доказательство — письмо, которое
пользователь получил). Адреса хранятся строчными буквами, поэтому `Ana@Example.com` и `ana@example.com` — одна учётная
запись. У пароля есть минимальная длина (Урок 10.6) и ещё **максимальная**: хеширование «пароля» длиной в мегабайт
дорого обошлось бы серверу.

Затем функции для паролей и токенов:

```rust
/// Argon2 намеренно медленный, поэтому он выполняется в блокирующем потоке, а не в асинхронном рантайме.
async fn hash_password(password: String) -> Result<String, ApiError> {
    tokio::task::spawn_blocking(move || {
        Argon2::default()
            .hash_password(password.as_bytes())
            .map(|hash| hash.to_string())
    })
    .await
    .map_err(|error| ApiError::Internal(error.to_string()))?
    .map_err(|error| ApiError::Internal(error.to_string()))
}

async fn verify_password(password: String, stored_hash: String) -> Result<bool, ApiError> {
    tokio::task::spawn_blocking(move || {
        let parsed = PasswordHash::new(&stored_hash)?;
        Ok::<bool, argon2::password_hash::Error>(
            Argon2::default()
                .verify_password(password.as_bytes(), &parsed)
                .is_ok(),
        )
    })
    .await
    .map_err(|error| ApiError::Internal(error.to_string()))?
    .map_err(|error| ApiError::Internal(error.to_string()))
}

/// 32 случайных байта от операционной системы, записанные как 64 шестнадцатеричных символа.
fn new_token() -> Result<String, ApiError> {
    let mut bytes = [0u8; 32];
    getrandom::fill(&mut bytes).map_err(|error| ApiError::Internal(error.to_string()))?;
    Ok(hex::encode(bytes))
}

fn hash_token(token: &str) -> Vec<u8> {
    Sha256::digest(token.as_bytes()).to_vec()
}
```

**Что происходит:**

1. `Argon2::default()` — это Argon2id с рекомендуемыми параметрами. `hash_password` сам создаёт случайную соль и
   возвращает строку PHC вида `$argon2id$v=19$m=19456,t=2,p=1$<salt>$<hash>`: алгоритм, версия, память (19 МиБ — мебибайт, 1 048 576 байт),
   итерации (2), параллелизм (1), соль и хеш — всё в одном тексте, который можно хранить в `TEXT`.
2. `verify_password` читает эти параметры обратно из сохранённой строки (`PasswordHash::new`), хеширует переданный
   пароль так же и сравнивает. Он возвращает `Ok` только для правильного пароля.
3. Хеширование занимает от десятков до сотен миллисекунд **намеренно**: именно это делает подбор дорогим. В
   асинхронном рантайме это заблокировало бы рабочий поток (Урок 16.1), поэтому работа выполняется в
   `tokio::task::spawn_blocking`, в отдельном пуле потоков, созданном для блокирующей работы. `.await` даёт `Result` от
   потока (ошибка — только если замыкание запаниковало), внутри которого `Result` хеширования; отсюда два `map_err`.
4. `getrandom::fill` заполняет массив байтами из безопасного генератора случайных чисел операционной системы
   (Урок 10.9). `hex::encode` превращает 32 байта в 64 символа `0-9a-f`.
5. `hash_token` — это SHA-256 токена, 32 байта, хранится как `BYTEA`.

## 5. Регистрация и вход

Всё ещё в `src/auth.rs`, два обработчика:

```rust
/// POST /users: создать учётную запись.
pub async fn register(
    State(state): State<AppState>,
    AppJson(credentials): AppJson<Credentials>,
) -> Result<(StatusCode, Json<User>), ApiError> {
    let email = parse_email(&credentials.email)?;
    check_password_rules(&credentials.password)?;
    let password_hash = hash_password(credentials.password).await?;

    let user = sqlx::query_as!(
        User,
        "INSERT INTO users (email, password_hash) VALUES ($1, $2) RETURNING id, email",
        email,
        password_hash
    )
    .fetch_one(&state.pool)
    .await
    .map_err(|error| match &error {
        sqlx::Error::Database(db) if db.is_unique_violation() => {
            ApiError::Conflict("email is already registered".to_string())
        }
        _ => ApiError::Database(error),
    })?;
    Ok((StatusCode::CREATED, Json(user)))
}

/// POST /sessions: войти, получить токен.
pub async fn login(
    State(state): State<AppState>,
    AppJson(credentials): AppJson<Credentials>,
) -> Result<(StatusCode, Json<Session>), ApiError> {
    let wrong = || ApiError::Unauthorized("wrong email or password");
    let email = parse_email(&credentials.email).map_err(|_| wrong())?;

    let row = sqlx::query!(
        "SELECT id, password_hash FROM users WHERE email = $1",
        email
    )
    .fetch_optional(&state.pool)
    .await?;
    let Some(row) = row else {
        return Err(wrong());
    };
    if !verify_password(credentials.password, row.password_hash).await? {
        return Err(wrong());
    }

    let token = new_token()?;
    sqlx::query!(
        "INSERT INTO sessions (token_hash, user_id, expires_at)
         VALUES ($1, $2, now() + interval '7 days')",
        hash_token(&token),
        row.id
    )
    .execute(&state.pool)
    .await?;
    Ok((StatusCode::CREATED, Json(Session { token })))
}
```

**Что происходит:**

1. `register` проверяет ввод, хеширует и вставляет. Если такой email уже есть, PostgreSQL отклоняет `INSERT` с
   *нарушением уникальности*; `is_unique_violation()` распознаёт его, и клиент получает `409 Conflict` вместо `500`.
   Позволить решать базе данных правильно даже тогда, когда два запроса соревнуются.
2. `login` даёт **один** ответ на любую неудачу — «wrong email or password»: неизвестный email, неверный пароль и
   некорректный email выглядят одинаково, поэтому ответ не говорит злоумышленнику, какие учётные записи существуют.
3. `let Some(row) = row else { ... };` — это *let-else*: если образец не совпал, блок `else` обязан выйти из функции.
4. При успехе он сохраняет **хеш** нового токена со сроком действия на семь дней вперёд (`now() + interval '7 days'`
   вычисляет PostgreSQL) и возвращает сам токен. Это единственный момент, когда токен существует вне клиента.
5. `POST /sessions` создаёт ресурс-сессию, поэтому подходит `201 Created` (Урок 16.2).

> 🔒 **Безопасность:** одинаковое сообщение — ещё не всё. Неизвестный email возвращается сразу, неверный пароль — только
> после работы Argon2, поэтому внимательный злоумышленник всё ещё может различить их по **времени**. Обычное решение —
> при неизвестном email проверять пароль против фиксированного фиктивного хеша, чтобы оба пути занимали одинаковое
> время. Кроме того, ограничивайте число попыток входа с одного адреса и для одной учётной записи (ограничение частоты
> запросов, Урок 16.15) и отправляйте токены только через TLS (Урок 15.12): bearer-токен в простом HTTP (HyperText Transfer Protocol, протокол передачи гипертекста) может прочитать
> любой в сети.

## 6. Экстрактор `AuthUser`

Каждый обработчик, которому нужен вход, должен прочитать заголовок `Authorization`, захешировать токен и найти
действующую сессию. Писать это в каждом обработчике было бы однообразно, и это легко забыть. **Экстрактор** делает это
один раз. Остаток `src/auth.rs`:

```rust
/// Пользователь, отправивший запрос. Обработчики, принимающие этот параметр, требуют входа.
pub struct AuthUser {
    pub id: i64,
}

impl FromRequestParts<AppState> for AuthUser {
    type Rejection = ApiError;

    async fn from_request_parts(parts: &mut Parts, state: &AppState) -> Result<Self, ApiError> {
        let missing = || ApiError::Unauthorized("missing or invalid token");
        let token = parts
            .headers
            .get(header::AUTHORIZATION)
            .and_then(|value| value.to_str().ok())
            .and_then(|value| value.strip_prefix("Bearer "))
            .ok_or_else(missing)?;

        let user_id = sqlx::query_scalar!(
            "SELECT user_id FROM sessions WHERE token_hash = $1 AND expires_at > now()",
            hash_token(token)
        )
        .fetch_optional(&state.pool)
        .await?
        .ok_or_else(missing)?;
        Ok(AuthUser { id: user_id })
    }
}

/// GET /me: кто я?
pub async fn me(user: AuthUser, State(state): State<AppState>) -> Result<Json<User>, ApiError> {
    let user = sqlx::query_as!(User, "SELECT id, email FROM users WHERE id = $1", user.id)
        .fetch_one(&state.pool)
        .await?;
    Ok(Json(user))
}
```

**Что происходит:**

1. `FromRequestParts` — трейт, стоящий за `Path`, `Query` и `State` (в Уроке 16.4 мы получали его через derive для
   `AppPath`). Здесь мы реализуем его вручную: axum вызывает `from_request_parts` до обработчика, передавая заголовки
   запроса и состояние.
2. `type Rejection = ApiError;`: если экстрактор не справился, его `ApiError` становится ответом (`401` с нашим JSON), и
   обработчик не выполняется.
3. Заголовок должен выглядеть как `Bearer <token>`; всё остальное — «missing or invalid».
4. Запрос ищет сессию с этим хешем токена, **срок которой не истёк**. Нет строки — значит, токен неизвестный, уже
   вышедший или просроченный: все три получают одинаковый `401`.
5. `me` показывает образец: обработчик, принимающий `user: AuthUser`, может выполниться только для вошедшего
   пользователя и знает, кто это.

Теперь защитите эндпоинты задач. В `src/main.rs` добавьте `mod auth;` рядом с другими модулями и `use auth::AuthUser;`
рядом с другими строками `use`. Затем дайте `create_task`, `update_task`, `delete_task` и `archive_done` первый
параметр `_user: AuthUser`:

```rust
async fn create_task(
    _user: AuthUser,
    State(state): State<AppState>,
    AppJson(new_task): AppJson<NewTask>,
) -> Result<impl IntoResponse, ApiError> {
```

(`_` говорит «нам нужна проверка, а не значение», поэтому компилятор не предупреждает.) Тело каждого обработчика
остаётся тем же. Наконец, три новых маршрута в `app()` после маршрутов задач:

```rust
        .route("/users", post(auth::register))
        .route("/sessions", post(auth::login))
        .route("/me", get(auth::me))
```

Снова выполните `cargo sqlx prepare` (новые запросы), затем `cargo run`.

## 7. Попробуйте весь путь

Во втором терминале (`H` — просто переменная командной оболочки, чтобы меньше печатать):

```bash
H='content-type: application/json'
curl -s -H "$H" -d '{"email":"Ana@Example.com","password":"correct horse battery"}' http://127.0.0.1:3000/users; echo
curl -s -H "$H" -d '{"email":"ana@example.com","password":"another long password"}' http://127.0.0.1:3000/users; echo
curl -s -H "$H" -d '{"email":"bob@example.com","password":"short"}' http://127.0.0.1:3000/users; echo
curl -s -H "$H" -d '{"email":"ana@example.com","password":"correct horse battery"}' http://127.0.0.1:3000/sessions; echo
curl -s -H "$H" -d '{"email":"ana@example.com","password":"wrong password"}' http://127.0.0.1:3000/sessions; echo
curl -s -H "$H" -d '{"email":"nobody@example.com","password":"correct horse battery"}' http://127.0.0.1:3000/sessions; echo
```

```text
{"id":1,"email":"ana@example.com"}
{"code":"conflict","message":"email is already registered"}
{"code":"validation_failed","message":"password must have 12 to 128 characters"}
{"token":"155e63c7c5629273e9c94777105fecbb98e295537822ee39e89982194c84316e"}
{"code":"unauthorized","message":"wrong email or password"}
{"code":"unauthorized","message":"wrong email or password"}
```

Email сохранён строчными буквами, поэтому вторая регистрация с `ana@example.com` — это конфликт. Ваш токен, конечно,
другой. Теперь запись без токена и с ним. Команда оболочки `$(...)` выполняет команду и кладёт её вывод в переменную;
`sed` вырезает токен из JSON:

```bash
curl -s -H "$H" -d '{"title":"Buy milk"}' http://127.0.0.1:3000/tasks; echo
TOKEN=$(curl -s -H "$H" -d '{"email":"ana@example.com","password":"correct horse battery"}' \
  http://127.0.0.1:3000/sessions | sed 's/.*"token":"\([0-9a-f]*\)".*/\1/')
curl -s -H "$H" -H "authorization: Bearer $TOKEN" -d '{"title":"Buy milk"}' http://127.0.0.1:3000/tasks; echo
curl -s -H "authorization: Bearer $TOKEN" http://127.0.0.1:3000/me; echo
curl -s -H "authorization: Bearer 0000" http://127.0.0.1:3000/me; echo
```

```text
{"code":"unauthorized","message":"missing or invalid token"}
{"id":1,"title":"Buy milk","done":false}
{"id":1,"email":"ana@example.com"}
{"code":"unauthorized","message":"missing or invalid token"}
```

Посмотрите, что знает база данных (в `psql`):

```sql
SELECT id, email, password_hash FROM users;
SELECT encode(token_hash, 'hex') AS token_hash, user_id, expires_at > now() AS valid FROM sessions;
```

```text
 id |      email      |                                           password_hash
----+-----------------+---------------------------------------------------------------------------------------------------
  1 | ana@example.com | $argon2id$v=19$m=19456,t=2,p=1$TmnAISn+Stbdb/Njn9u+iQ$2CqEm5IPbGeZ3QlkNtF/vtHhp7Zl57gRGTUdXDelMJU
(1 row)

                            token_hash                            | user_id | valid
------------------------------------------------------------------+---------+-------
 408ff18aeaea528da6a07d1883ee82a3ba07c54ae4183cd135896cf969d7da0f |       1 | t
 2c0bd48eb942127d9725a743ec1c62ae73415ce8c4e41f38b64e5051f5c81b16 |       1 | t
(2 rows)
```

Нигде нет ни пароля, ни токена: только строка PHC (с солью `TmnAISn+...`) и два хеша токенов, по одному на вход.
Сравните полученный токен с хешами: они никак не связаны, как и должно быть.

> ⚡ **Производительность:** лог сервиса показывает `latency=239 ms` для `POST /users`, почти всё — Argon2 в
> отладочной сборке. Эта цена и есть защита, но она же означает, что вход — дорогой эндпоинт, которым злоумышленники
> могут нагружать ваш сервер: ещё одна причина для ограничения частоты запросов. Релизные сборки
> (`cargo run --release`) гораздо быстрее.

## 8. Тесты с вошедшим пользователем

Старые тесты, которые создают задачи, теперь получают `401`. Дайте помощникам тестов токен: `post_json` добавляет
заголовок `Authorization`, а `login` регистрирует тестового пользователя и входит:

```rust
    fn post_json(uri: &str, token: &str, json: &'static str) -> Request<Body> {
        Request::post(uri)
            .header(header::CONTENT_TYPE, "application/json")
            .header(header::AUTHORIZATION, format!("Bearer {token}"))
            .body(Body::from(json))
            .unwrap()
    }

    /// Регистрирует пользователя, входит и возвращает токен.
    async fn login(app: &Router) -> String {
        let credentials = r#"{"email":"ana@example.com","password":"correct horse battery"}"#;
        let (status, _) = send(app.clone(), post_json("/users", "", credentials)).await;
        assert_eq!(status, StatusCode::CREATED);
        let (status, body) = send(app.clone(), post_json("/sessions", "", credentials)).await;
        assert_eq!(status, StatusCode::CREATED);
        // Тело — это {"token":"<64 шестнадцатеричных символа>"}.
        body[10..74].to_string()
    }
```

`body[10..74]` вырезает токен из `{"token":"` (10 символов) плюс 64 шестнадцатеричных символа; для анонимных вызовов
токен — `""`, который сервер для `/users` и `/sessions` просто игнорирует. Обновите тесты, которые пишут задачи: пусть
сначала вызывают `let token = login(&app).await;` и передают `&token`, например:

```rust
    #[sqlx::test]
    async fn created_task_can_be_read_back(pool: PgPool) {
        let app = test_app(pool).await;
        let token = login(&app).await;

        let (status, body) = send(
            app.clone(),
            post_json("/tasks", &token, r#"{"title":"Buy milk"}"#),
        )
        .await;
        assert_eq!(status, StatusCode::CREATED);
        assert_eq!(body, r#"{"id":1,"title":"Buy milk","done":false}"#);

        let request = Request::get("/tasks/1").body(Body::empty()).unwrap();
        let (status, body) = send(app, request).await;
        assert_eq!(status, StatusCode::OK);
        assert_eq!(body, r#"{"id":1,"title":"Buy milk","done":false}"#);
    }
```

Сделайте то же в `empty_title_is_rejected` и `archive_is_all_or_nothing` (там `post_empty` тоже принимает токен и
добавляет тот же заголовок). Затем тест самой аутентификации:

```rust
    #[sqlx::test]
    async fn the_token_identifies_the_user(pool: PgPool) {
        let app = test_app(pool).await;
        let token = login(&app).await;

        let request = Request::get("/me")
            .header(header::AUTHORIZATION, format!("Bearer {token}"))
            .body(Body::empty())
            .unwrap();
        let (status, body) = send(app.clone(), request).await;
        assert_eq!(status, StatusCode::OK);
        assert_eq!(body, r#"{"id":1,"email":"ana@example.com"}"#);

        let wrong = r#"{"email":"ana@example.com","password":"wrong password!!"}"#;
        let (status, body) = send(app, post_json("/sessions", "", wrong)).await;
        assert_eq!(status, StatusCode::UNAUTHORIZED);
        assert_eq!(
            body,
            r#"{"code":"unauthorized","message":"wrong email or password"}"#
        );
    }
```

```text
test result: ok. 9 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 1.76s
```

Теперь тесты идут дольше (Argon2 в отладочной сборке); так и должно быть.

> 🧪 **Тестирование:** проверяйте **отказы** системы аутентификации не менее тщательно, чем успех: нет токена, неверный
> токен, просроченный, неверный пароль. Вход, который принимает всё подряд, проходит каждый тест удачного пути.

## Частые ошибки

**1. Хранение токенов или паролей открытым текстом.** Тогда утёкшая резервная копия, файл лога или одно SQL-внедрение (Structured Query Language, язык структурированных запросов)
выдают все учётные записи. Храните хеши Argon2 для паролей и хеши SHA-256 для токенов.

**2. Запись секретов в лог.** `tracing::debug!(?credentials)` пишет пароль в лог; записанный заголовок запроса пишет
токен. Никогда не логируйте `Credentials`, заголовки `Authorization` и токены (Урок 09.10, раздел 8).

**3. Хеширование в асинхронном рантайме.** Прямой вызов `Argon2::default().hash_password(...)` в обработчике блокирует
поток рантайма на каждый вход. Используйте `spawn_blocking`.

**4. Разные ответы для «нет такого пользователя» и «неверный пароль».** Это превращает форму входа в инструмент для
перебора пользователей. Используйте одно сообщение (и, в идеале, одинаковое время).

**5. Забытая проверка срока действия.** `WHERE token_hash = $1` без `AND expires_at > now()` делает каждый токен
вечным.

**6. Проверка только во фронтенде.** Спрятанная от невошедших пользователей кнопка «удалить» ничего не защищает
(Урок 16.1). Настоящая проверка — параметр `AuthUser` у обработчика.

## Попробуйте сами

1. Войдите дважды и посмотрите на две строки в `sessions`. Каждое устройство пользователя получает свою сессию.
2. В `psql` сделайте свою сессию просроченной: `UPDATE sessions SET expires_at = now() - interval '1 second';`, затем
   вызовите `/me` со своим токеном: `401`.
3. Зарегистрируйтесь с `"   Bob@Example.COM  "` и проверьте, что сохранилось.
4. Сделайте коммит: `git add code/p16-tasks && git commit -m "p16-tasks: users, sessions and protected writes"`.

## Упражнения

### Упражнение 1 — Запись требует токена (легко)

Напишите `#[sqlx::test]` `writes_need_a_token`: `POST /tasks` без токена должен вернуть `401` с
`{"code":"unauthorized","message":"missing or invalid token"}`, а `DELETE /tasks/1` без токена должен вернуть `401`.

<details>
<summary>Подсказка</summary>

`post_json("/tasks", "", ...)` отправляет `Authorization: Bearer ` с пустым токеном, которому не соответствует ни одна
сессия.

</details>

<details>
<summary>Ответ</summary>

```rust
    #[sqlx::test]
    async fn writes_need_a_token(pool: PgPool) {
        let app = test_app(pool).await;
        let (status, body) = send(app.clone(), post_json("/tasks", "", r#"{"title":"x"}"#)).await;
        assert_eq!(status, StatusCode::UNAUTHORIZED);
        assert_eq!(
            body,
            r#"{"code":"unauthorized","message":"missing or invalid token"}"#
        );

        let request = Request::delete("/tasks/1").body(Body::empty()).unwrap();
        let (status, _) = send(app, request).await;
        assert_eq!(status, StatusCode::UNAUTHORIZED);
    }
```

Почему это работает: экстрактор `AuthUser` выполняется до обработчика, поэтому `DELETE` отклоняется, хотя задачи 1 не
существует: анонимный клиент ничего не узнаёт — даже то, существует ли задача.

</details>

### Упражнение 2 — Выход (средне)

Добавьте `DELETE /sessions/current`: он удаляет сессию отправленного токена и отвечает `204`. После этого токен не
должен работать. Напишите тест, который входит, выходит и проверяет, что `/me` затем отвечает `401`.

<details>
<summary>Подсказка</summary>

Обработчику нужен хеш токена, а не только идентификатор пользователя. Добавьте `pub token_hash: Vec<u8>` в `AuthUser`
и заполните его в экстракторе. Зарегистрируйте маршрут через `axum::routing::delete(auth::logout)`.

</details>

<details>
<summary>Ответ</summary>

```rust
// в AuthUser:               pub token_hash: Vec<u8>,
// в from_request_parts:     let token_hash = hash_token(token);
//                           ... используйте `token_hash` в запросе ...
//                           Ok(AuthUser { id: user_id, token_hash })

/// DELETE /sessions/current: выйти. Токен сразу перестаёт работать.
pub async fn logout(user: AuthUser, State(state): State<AppState>) -> Result<StatusCode, ApiError> {
    sqlx::query!(
        "DELETE FROM sessions WHERE token_hash = $1",
        user.token_hash
    )
    .execute(&state.pool)
    .await?;
    Ok(StatusCode::NO_CONTENT)
}

// в app():   .route("/sessions/current", axum::routing::delete(auth::logout))

    #[sqlx::test]
    async fn logout_ends_the_session(pool: PgPool) {
        let app = test_app(pool).await;
        let token = login(&app).await;
        let with_token = |request: axum::http::request::Builder| {
            request
                .header(header::AUTHORIZATION, format!("Bearer {token}"))
                .body(Body::empty())
                .unwrap()
        };

        let (status, _) = send(
            app.clone(),
            with_token(Request::delete("/sessions/current")),
        )
        .await;
        assert_eq!(status, StatusCode::NO_CONTENT);

        let (status, _) = send(app, with_token(Request::get("/me"))).await;
        assert_eq!(status, StatusCode::UNAUTHORIZED);
    }
```

Почему это работает: поскольку сессии живут в базе данных, выход — это один `DELETE`, и он сразу действует на
**каждом** экземпляре сервиса. В этом большое преимущество серверных сессий перед самодостаточными токенами
(Урок 10.7), которые нельзя отменить до истечения срока.

</details>

### Упражнение 3 — Личные задачи (сложно)

Сделайте так, чтобы задачи принадлежали своему создателю. Добавьте миграцию `add_owner_to_tasks` с
`ALTER TABLE tasks ADD COLUMN owner_id BIGINT REFERENCES users (id) ON DELETE CASCADE;`. Сохраняйте `user.id` при
создании задачи и сделайте так, чтобы `GET /tasks/{id}` требовал входа и возвращал только собственные задачи
вызывающего (`404` для чужих, Урок 16.2). Напишите тест, в котором Боб не может прочитать задачу Аны. (Для списка,
`PATCH` и `DELETE` то же делается по тому же образцу.)

<details>
<summary>Подсказка</summary>

Столбец должен допускать `NULL`, потому что у старых задач нет владельца. В SQL добавьте `AND owner_id = $2` в
`WHERE`. Для двух пользователей разделите `login` на `login_as(app, credentials)`. Старые тесты, которые читают
`/tasks/1` без токена, теперь получают `401`: дайте токен и им.

</details>

<details>
<summary>Ответ</summary>

```sql
-- Каждая новая задача принадлежит пользователю, который её создал.
-- У старых задач нет владельца (NULL), поэтому через API их никто не прочитает.
ALTER TABLE tasks ADD COLUMN owner_id BIGINT REFERENCES users (id) ON DELETE CASCADE;
```

```rust
// create_task принимает `user: AuthUser` и вставляет владельца:
//     "INSERT INTO tasks (title, owner_id) VALUES ($1, $2) RETURNING id, title, done",
//     title,
//     user.id

async fn get_task(
    user: AuthUser,
    State(state): State<AppState>,
    Path(id): Path<i64>,
) -> Result<Json<Task>, ApiError> {
    let task = sqlx::query_as!(
        Task,
        "SELECT id, title, done FROM tasks WHERE id = $1 AND owner_id = $2",
        id,
        user.id
    )
    .fetch_optional(&state.pool)
    .await?;
    task.map(Json).ok_or(ApiError::NotFound(id))
}

    #[sqlx::test]
    async fn users_cannot_read_each_others_tasks(pool: PgPool) {
        let app = test_app(pool).await;
        let ana = login(&app).await;
        let bob = login_as(
            &app,
            r#"{"email":"bob@example.com","password":"another long password"}"#,
        )
        .await;
        send(
            app.clone(),
            post_json("/tasks", &ana, r#"{"title":"Ana's task"}"#),
        )
        .await;

        let get_as = |token: &str| {
            Request::get("/tasks/1")
                .header(header::AUTHORIZATION, format!("Bearer {token}"))
                .body(Body::empty())
                .unwrap()
        };
        let (status, _) = send(app.clone(), get_as(&ana)).await;
        assert_eq!(status, StatusCode::OK);
        let (status, _) = send(app, get_as(&bob)).await;
        assert_eq!(status, StatusCode::NOT_FOUND);
    }
```

`login` становится `login_as(app, ana_credentials)`, а в `login_as` переезжает старое тело. Когда старые тесты,
читающие `/tasks/1` и `/tasks/42`, получат токен, все двенадцать тестов (с упражнениями 1 и 2) проходят:

```text
test result: ok. 12 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 1.26s
```

Почему это работает: авторизация теперь — часть самого запроса, поэтому её невозможно забыть между «загрузить» и
«проверить». Боб получает `404`, а не `403`, так что он даже не узнаёт, что задача 1 существует. Это закрывает дыру
«число в адресе» из Урока 10.3.

</details>

## Вопросы для самопроверки

1. Чем аутентификация отличается от авторизации и какие коды статуса относятся к каждой?

<details>
<summary>Ответ</summary>

Аутентификация выясняет, кто обращается (неудача: `401`). Авторизация решает, что этому клиенту можно делать
(неудача: `403` или `404`, чтобы скрыть существование ресурса).

</details>

2. Почему сервис хранит `SHA-256(token)`, а не сам токен?

<details>
<summary>Ответ</summary>

Тот, кто прочитает базу данных (утечка, резервная копия, SQL-внедрение), получит только хеши и не сможет использовать
их как токены. Быстрого хеша достаточно, потому что 256-битный случайный токен нельзя угадать.

</details>

3. Почему Argon2 выполняется внутри `spawn_blocking`?

<details>
<summary>Ответ</summary>

Argon2 намеренно медленный (от десятков до сотен миллисекунд работы процессора). В асинхронном рантайме он
заблокировал бы рабочий поток и задержал другие запросы; `spawn_blocking` переносит его в пул потоков для
блокирующей работы.

</details>

4. Почему `login` отвечает «wrong email or password» и на неизвестный email, и на неверный пароль?

<details>
<summary>Ответ</summary>

Чтобы не допустить перебора пользователей: разные сообщения сказали бы злоумышленнику, у каких адресов есть учётные
записи.

</details>

5. Как обработчик требует входа и что происходит с запросом без действительного токена?

<details>
<summary>Ответ</summary>

Он принимает параметр `AuthUser`. Экстрактор выполняется до обработчика; без действующей непросроченной сессии он
возвращает `ApiError::Unauthorized`, клиент получает `401`, а обработчик не выполняется.

</details>

6. Почему `email` объявлен в базе данных как `UNIQUE`, хотя `register` мог бы сначала проверить через `SELECT`?

<details>
<summary>Ответ</summary>

Две регистрации могут прийти одновременно; оба `SELECT` ничего не найдут, и оба `INSERT` пройдут. Ограничение `UNIQUE`
заставляет базу данных отклонить второй, а обработчик превращает это в `409`.

</details>

## Итоги

- Регистрация сохраняет хеш Argon2id (строку PHC с солью и параметрами); вход его проверяет. Хеширование выполняется в
  `spawn_blocking`.
- Вход создаёт 32-байтный случайный токен; база данных хранит только его хеш SHA-256 со сроком действия; клиент
  отправляет его как `Authorization: Bearer <token>`.
- Одно сообщение об ошибке для любой неудачи входа; гонки решают ограничения уникальности; у пароля есть пределы длины.
- Собственный экстрактор `FromRequestParts` (`AuthUser`) превращает «требует входа» в параметр обработчика; его отказ —
  JSON-ответ `401`.
- Серверные сессии можно завершить сразу (выход = `DELETE`); авторизации место в запросе (`owner_id = $2`).
- Никогда не логируйте и не храните пароли и токены открытым текстом; в продакшене используйте TLS и ограничение
  частоты запросов.

## Источники и что почитать

- [OWASP Password Storage Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Password_Storage_Cheat_Sheet.html) — параметры Argon2id, соли и «перец».
- [OWASP Session Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Session_Management_Cheat_Sheet.html) — длина токена, срок действия, выход и хранение.
- [OWASP Authentication Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html) — сообщения об ошибках, перебор пользователей и ограничение попыток входа.
- [argon2 crate](https://docs.rs/argon2/0.6) — используемый здесь API (Application Programming Interface, программный интерфейс) с примерами хеширования и проверки.
- [axum `FromRequestParts`](https://docs.rs/axum/0.8/axum/extract/trait.FromRequestParts.html) — как писать собственные экстракторы.
- [RFC 6750 — Bearer Token Usage](https://www.rfc-editor.org/rfc/rfc6750) — стандарт для `Authorization: Bearer`.

---

[🏠 Начало](../START-HERE.md) · [← 16.7 Миграции и транзакции](07-migrations-and-transactions.md) · [📋 Этап 16](README.md) · [16.9 gRPC на tonic →](09-grpc-with-tonic.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
