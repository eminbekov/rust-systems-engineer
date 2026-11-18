<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 16.6 Работа с базами данных через sqlx](06-databases-with-sqlx.md) · [📋 Этап 16](README.md) · [16.8 Аутентификация в веб-сервисе →](08-authentication-in-a-web-service.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/16-backend-services/07-migrations-and-transactions.md) <!-- nav -->

# Урок 16.7 — Миграции и транзакции

> **Этап 16 — Бэкенд-сервисы** · Урок 7 из 16 · ⏱ около 2,5 часа (2 занятия)

## Что вы узнаете

- Что такое миграция базы данных и как создавать, применять и просматривать миграции через `sqlx-cli`.
- Почему никогда нельзя менять миграцию, которая уже была применена.
- Как сервис сам применяет свои миграции при запуске и как их использует `#[sqlx::test]`.
- Что такое транзакция: несколько изменений, которые происходят полностью или не происходят вовсе.
- Как использовать транзакцию в Rust с sqlx (`begin`, `commit` и откат, когда транзакция отбрасывается) и как проверить, что она действительно откатывается.

## Перед началом

- Вы прошли [Урок 16.6 — Работа с базами данных через sqlx](06-databases-with-sqlx.md): `code/p16-tasks` использует
  PostgreSQL, а `sqlx-cli` установлен (`cargo sqlx --version` работает).
- Контейнер `p16-postgres` запущен.

## Новые слова

| Слово | Значение |
|---|---|
| **схема** (schema) | Структура базы данных: её таблицы, столбцы, типы, ограничения и индексы (а не данные в них). |
| **миграция** (migration) | Один пронумерованный SQL-файл (Structured Query Language, язык структурированных запросов), который меняет схему на один шаг, например «создать таблицу `tasks`». Миграции выполняются по порядку, и каждая — один раз для каждой базы данных. |
| **расхождение схем** (schema drift) | Когда две базы данных, у которых должна быть одинаковая схема (ваш ноутбук, тестовый сервер, продакшен), постепенно становятся разными. Миграции это предотвращают. |
| **контрольная сумма** (checksum) | Короткий «отпечаток», вычисленный из байтов файла. Если меняется один байт, меняется и контрольная сумма. |
| **транзакция** (transaction) | Группа SQL-операторов, которую база данных считает одним целым: сохраняются либо все её изменения, либо ни одного. |
| **фиксация / откат** (commit / rollback) | Завершение транзакции с сохранением всех её изменений (фиксация) или с отбрасыванием их всех (откат). |
| **атомарный** (atomic) | «Неделимый»: атомарную операцию никогда нельзя увидеть сделанной наполовину. |
| **ACID** (Atomicity, Consistency, Isolation, Durability — атомарность, согласованность, изоляция, долговечность) | Четыре гарантии транзакций базы данных. Урок 17.10 подробно объясняет каждую. |

## 1. Зачем нужны миграции?

В Уроке 16.5 вы набирали `CREATE TABLE` в `psql`, а в Уроке 16.6 тесты выполняли `schema.sql`. Один раз это
работает. Теперь представьте, что сервис в продакшене и в нём миллион задач, а вам нужен новый столбец. Нельзя удалить
базу данных и снова выполнить `schema.sql`. Нужно **изменение**: `ALTER TABLE tasks ADD COLUMN ...`. И это изменение
должно выполниться на вашем ноутбуке, на ноутбуке каждого коллеги, в тестах и в продакшене — везде ровно один раз и в
правильном порядке.

**Миграция** — одно такое изменение в своём файле. Файлы пронумерованы (sqlx использует метку времени), а база данных
помнит, какие из них уже выполнила, в таблице `_sqlx_migrations`:

```text
 migrations/                                   database: _sqlx_migrations
 ├── 20261118073832_create_tasks.sql     ──▶   20261118073832 create tasks           ✔ applied
 ├── 20261118073943_create_archived_...  ──▶   20261118073943 create archived tasks  ✔ applied
 └── 20261120090000_add_due_date.sql     ──▶   (not yet)  → `migrate run` applies only this one
```

Применять миграции всегда безопасно: уже применённые файлы пропускаются, новые применяются по порядку. Каждая база
данных, построенная из одной и той же папки, приходит к одной и той же схеме. Миграции — это код: они живут в Git, и их
проверяют на код-ревью, как код.

## 2. Ваши первые миграции

Начните с пустой базы данных. `sqlx-cli` читает `DATABASE_URL` из `.env`, как и макросы:

```bash
cd ~/projects/rust/code/p16-tasks
cargo sqlx database drop -y
cargo sqlx database create
cargo sqlx migrate add create_tasks
```

```text
Creating migrations/20261118073832_create_tasks.sql
```

**Что происходит:**

1. `database drop -y` удаляет базу данных `tasks` со всем содержимым (`-y` отвечает «да» на вопрос). На вашем
   ноутбуке это нормально, и это последний раз, когда вы строите эту базу вручную.
2. `database create` снова создаёт её, пустой.
3. `migrate add create_tasks` создаёт папку `migrations` и пустой файл, имя которого начинается с текущей даты и
   времени. У вас номер другой; порядок номеров — это порядок миграций.

Положите в файл таблицы из Урока 16.5 (это содержимое `schema.sql` плюс индекс):

```sql
-- Задачи и их комментарии (таблицы из Урока 16.5).
CREATE TABLE tasks (
    id         BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    title      TEXT NOT NULL CHECK (char_length(title) BETWEEN 1 AND 200),
    done       BOOLEAN NOT NULL DEFAULT false,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE comments (
    id      BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    task_id BIGINT NOT NULL REFERENCES tasks (id) ON DELETE CASCADE,
    text    TEXT NOT NULL
);

CREATE INDEX comments_task_id_idx ON comments (task_id);
```

Удалите `schema.sql`: миграция его заменяет. Теперь посмотрите на состояние, примените и посмотрите снова:

```bash
cargo sqlx migrate info
cargo sqlx migrate run
cargo sqlx migrate info
```

```text
20261118073832/pending create tasks
Applied 20261118073832/migrate create tasks (6.512875ms)
20261118073832/installed create tasks
```

`pending` означает «есть в папке, но ещё не в этой базе»; `installed` — «применена». В `psql` таблица учёта видна
рядом с вашими таблицами:

```sql
SELECT version, description, success FROM _sqlx_migrations;
```

```text
    version     | description  | success
----------------+--------------+---------
 20261118073832 | create tasks | t
(1 row)
```

Теперь вторая миграция — таблица для архива, который нужен в разделе 6:

```bash
cargo sqlx migrate add create_archived_tasks
```

```sql
-- Сюда выполненные задачи переносит POST /tasks/archive-done.
CREATE TABLE archived_tasks (
    id          BIGINT PRIMARY KEY,
    title       TEXT NOT NULL,
    archived_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

```bash
cargo sqlx migrate run
```

```text
Applied 20261118073943/migrate create archived tasks (5.881666ms)
```

Выполнился только новый файл. `archived_tasks.id` — простой `BIGINT PRIMARY KEY`, не `GENERATED`: архивная задача
сохраняет идентификатор, который у неё был.

> 📏 **Соглашение:** одна миграция на одно изменение, с именем, которое говорит, что она делает
> (`add_due_date_to_tasks`, `create_users`). Держите изменение схемы и код, которому оно нужно, в одном коммите.

## 3. Никогда не правьте применённую миграцию

Что, если вы заметили ошибку в миграции, которая уже выполнилась? Допустим, вы меняете `BETWEEN 1 AND 200` на `300` в
первом файле и снова запускаете:

```bash
cargo sqlx migrate run
cargo sqlx migrate info
```

```text
error: migration 20261118073832 was previously applied but has been modified
20261118073832/installed (different checksum) create tasks
applied migration had checksum b7e8b06a9283d75509f4e7e3a47c6d36fd7e2d257d77c1b51368bed55593f42ef53a9fa365ff18ab43f7b67f7a4e180c
local migration has checksum 9ad74d7a90b957610d322c84e431900cbb89a963873d01b1bf99ef8dfe9b49cd34aaa7de419ace0c0548c22d8ee832b4
```

sqlx сохранил **контрольную сумму** каждого применённого файла. Файл изменился, значит, и сумма другая, и sqlx
отказывается продолжать. Это защита: изменённый файл больше никогда не выполнился бы на базах, где уже была старая
версия, и ваш ноутбук и продакшен молча разошлись бы.

Правило: **применённая миграция — это история; никогда её не меняйте.** Чтобы изменить схему, добавьте **новую**
миграцию (`ALTER TABLE tasks DROP CONSTRAINT ...; ALTER TABLE tasks ADD CONSTRAINT ...`). Сейчас отмените свою правку
(обратно на `200`), и `migrate run` снова доволен.

> 💡 **Совет:** миграцию, которую вы **ещё не** отправили и которая выполнялась только на вашем ноутбуке, ещё можно
> исправить: после правки откатите всё через `cargo sqlx database reset -y` (удалить, создать, выполнить все миграции).
> Если её могли выполнить другие, пишите новую миграцию.

## 4. Миграции при запуске сервиса

Запускать `cargo sqlx migrate run` вручную перед каждым стартом легко забыть. sqlx может положить файлы миграций
**в саму программу**: макрос `sqlx::migrate!()` читает папку `migrations` при компиляции. Добавьте три строки в
`main` сразу после создания пула:

```rust
    let pool = PgPoolOptions::new()
        .max_connections(5)
        .acquire_timeout(Duration::from_secs(3))
        .connect(&database_url)
        .await
        .expect("cannot connect to the database");
    sqlx::migrate!()
        .run(&pool)
        .await
        .expect("cannot run the migrations");
```

При каждом запуске сервис применяет недостающие миграции и пропускает остальные. Если что-то не так (изменённый файл,
падающий SQL-оператор), сервис отказывается запускаться, и это гораздо лучше, чем работать с неправильной схемой.

`#[sqlx::test]` использует ту же папку: когда `migrations` существует, каждая тестовая база получает все миграции до
начала теста. Поэтому помощник в тестах становится тривиальным:

```rust
    /// `#[sqlx::test]` уже выполнил миграции в свежей тестовой базе данных.
    async fn test_app(pool: PgPool) -> Router {
        app(AppState { pool })
    }
```

Теперь тесты используют ровно ту схему, что и продакшен, и никому не нужно поддерживать `schema.sql` в актуальном
состоянии.

> ⚠️ **Частая ошибка:** при нескольких экземплярах все они в момент старта одновременно выполняют `migrate!`. sqlx
> берёт блокировку в PostgreSQL, так что миграции применяет только один, а остальные ждут. И всё же миграция, которая
> меняет огромную таблицу, может заблокировать её на минуты; большие команды выполняют такие миграции отдельным шагом
> перед развёртыванием.

## 5. Транзакции

Некоторым изменениям нужны **несколько** операторов. Хороший пример — архив: «перенести каждую выполненную задачу в
`archived_tasks`» — это `DELETE` из одной таблицы **и** `INSERT` в другую. Если между ними программа упадёт или
оператор завершится ошибкой, задачи потеряются (удалены, но не заархивированы) или задвоятся. Изменение должно быть
**атомарным**: всё или ничего.

Ровно это даёт **транзакция**. Попробуйте в `psql`. `BEGIN` начинает транзакцию, `ROLLBACK` отбрасывает её изменения:

```sql
BEGIN;
INSERT INTO tasks (title) VALUES ('Inside a transaction') RETURNING id;
SELECT count(*) FROM tasks;
ROLLBACK;
SELECT count(*) FROM tasks;
```

```text
BEGIN
 id
----
  1
(1 row)

INSERT 0 1
 count
-------
     1
(1 row)

ROLLBACK
 count
-------
     0
(1 row)
```

Внутри транзакции новая строка существует; после `ROLLBACK` её никогда не было. (Её `id`, 1, всё равно израсходован:
ещё один пропуск.) `COMMIT` вместо `ROLLBACK` сохранил бы её.

**Ошибка внутри транзакции** портит всю транзакцию:

```sql
BEGIN;
INSERT INTO tasks (title) VALUES ('First');
INSERT INTO tasks (title) VALUES ('');
INSERT INTO tasks (title) VALUES ('After the error');
COMMIT;
SELECT count(*) FROM tasks;
```

```text
BEGIN
INSERT 0 1
ERROR:  new row for relation "tasks" violates check constraint "tasks_title_check"
DETAIL:  Failing row contains (3, , f, 2026-11-18 07:39:07.064407+00).
ERROR:  current transaction is aborted, commands ignored until end of transaction block
ROLLBACK
 count
-------
     0
(1 row)
```

После неудачного `INSERT` PostgreSQL игнорирует все команды до конца транзакции, а на `COMMIT` отвечает `ROLLBACK`:
пропало даже успешное `'First'`. Всё или ничего.

**Другие клиенты не видят незаконченную работу.** Откройте второй терминал с `psql`
(`docker exec -it p16-postgres psql -U postgres -d tasks`). В **первом** начните медленную транзакцию:

```sql
BEGIN; INSERT INTO tasks (title) VALUES ('Slow writer'); SELECT pg_sleep(10); COMMIT;
```

В течение этих десяти секунд во **втором** терминале, а затем ещё раз после них:

```sql
SELECT id, title FROM tasks;
```

```text
 id | title
----+-------
(0 rows)

 id |    title
----+-------------
  4 | Slow writer
(1 row)
```

Пока транзакция идёт, второй клиент ничего не видит; после `COMMIT` он видит весь результат сразу. Это
*изоляция*, буква «I» в **ACID**:

| Буква | Гарантия | В этом уроке |
|---|---|---|
| A — атомарность | всё или ничего | неудачный `INSERT` отменил `'First'` |
| C — согласованность | ограничения выполняются после каждой транзакции | `CHECK` отклонил `''` |
| I — изоляция | незаконченные транзакции невидимы для других | второй клиент увидел `0 rows` |
| D — долговечность | зафиксированные данные переживают сбой | Урок 16.5, раздел 8 |

Существует несколько **уровней изоляции** с разными компромиссами; уровень по умолчанию в PostgreSQL («read
committed») — тот, что вы только что видели. Уроки 17.10–17.12 показывают, как базы данных реализуют все четыре
гарантии.

## 6. Транзакция в Rust

Теперь архив как эндпоинт. Добавьте `post` в импорт `routing` (`routing::{get, post}`), затем этот обработчик:

```rust
#[derive(Debug, Serialize)]
struct Archived {
    archived: usize,
}

/// Переносит каждую выполненную задачу в `archived_tasks`: все или ни одной.
async fn archive_done(State(state): State<AppState>) -> Result<Json<Archived>, ApiError> {
    let mut tx = state.pool.begin().await?;

    let moved = sqlx::query_as!(
        Task,
        "DELETE FROM tasks WHERE done RETURNING id, title, done"
    )
    .fetch_all(&mut *tx)
    .await?;

    for task in &moved {
        sqlx::query!(
            "INSERT INTO archived_tasks (id, title) VALUES ($1, $2)",
            task.id,
            task.title
        )
        .execute(&mut *tx)
        .await?;
    }

    tx.commit().await?;
    Ok(Json(Archived {
        archived: moved.len(),
    }))
}
```

и его маршрут в `app()`, рядом с `/tasks`:

```rust
        .route("/tasks/archive-done", post(archive_done))
```

**Что происходит:**

1. `state.pool.begin()` берёт соединение из пула и отправляет `BEGIN`. `tx` — это `Transaction`: все запросы,
   выполненные через неё, принадлежат транзакции.
2. `.fetch_all(&mut *tx)` выполняет запрос **в транзакции**, а не через пул. (`*tx` даёт соединение внутри
   транзакции; `&mut` — потому что соединение может выполнять только один запрос за раз.) Если бы вы по ошибке написали
   здесь `&state.pool`, запрос выполнился бы на другом соединении, вне транзакции.
3. `DELETE ... RETURNING` удаляет выполненные задачи и возвращает именно эти строки, поэтому `INSERT` архивируют ровно
   то, что было удалено, даже если другой запрос тем временем отметит задачу выполненной.
4. `tx.commit()` отправляет `COMMIT`. Только теперь другие клиенты видят изменение.
5. Если какой-нибудь `?` досрочно возвращается, `tx` **отбрасывается** без `commit`. Тогда sqlx откатывает
   транзакцию. Вы никогда не пишете `ROLLBACK` сами: досрочный возврат, ошибка или паника отменяют работу. Это RAII (Resource Acquisition Is Initialization, получение ресурса есть инициализация) из
   Урока 08.7.

Путь `/tasks/archive-done` называет действие, а не ресурс. Урок 16.2 разрешал именно это для `POST`, когда операция не
укладывается в «создать/прочитать/изменить/удалить».

Пересоздайте базу данных, чтобы увидеть, как миграции выполняются при запуске (`cargo sqlx database drop -y` и
`cargo sqlx database create`), запустите сервис через `cargo run` и во втором терминале:

```bash
curl -s -H 'content-type: application/json' -d '{"title":"Buy milk"}' http://127.0.0.1:3000/tasks; echo
curl -s -H 'content-type: application/json' -d '{"title":"Write lesson"}' http://127.0.0.1:3000/tasks; echo
curl -s -H 'content-type: application/json' -d '{"title":"Call mom"}' http://127.0.0.1:3000/tasks; echo
curl -s -X PATCH -H 'content-type: application/json' -d '{"done":true}' http://127.0.0.1:3000/tasks/1 > /dev/null
curl -s -X PATCH -H 'content-type: application/json' -d '{"done":true}' http://127.0.0.1:3000/tasks/3 > /dev/null
curl -s -X POST http://127.0.0.1:3000/tasks/archive-done; echo
curl -s http://127.0.0.1:3000/tasks; echo
```

```text
{"id":1,"title":"Buy milk","done":false}
{"id":2,"title":"Write lesson","done":false}
{"id":3,"title":"Call mom","done":false}
{"archived":2}
[{"id":2,"title":"Write lesson","done":false}]
```

Новая база данных началась с идентификатора 1, и в ней обе миграции: сервис применил их, прежде чем начать слушать.
В `psql` запрос `SELECT id, title FROM archived_tasks ORDER BY id;` показывает задачи 1 и 3.

> ⚡ **Производительность:** цикл отправляет по одному `INSERT` на задачу. Для нескольких строк это нормально; для
> тысяч сделайте это одним оператором (PostgreSQL умеет вставлять из списка через `UNNEST` или из `DELETE ... RETURNING`
> в запросе `WITH`). И делайте транзакции **короткими**: транзакция держит соединение из пула и блокирует строки, пока
> не закончится, поэтому никогда не ждите внутри неё медленного сетевого вызова.

## 7. Тест отката

Самое интересное свойство транзакции — что происходит, когда что-то ломается **посередине**. Заставьте второй
оператор нарочно упасть: положите в `archived_tasks` строку с `id` 1 до архивации задачи 1, чтобы `INSERT` нарушил
первичный ключ. Добавьте в модуль `tests` помощник для пустых запросов `POST` и тест:

```rust
    fn post_empty(uri: &str) -> Request<Body> {
        Request::post(uri).body(Body::empty()).unwrap()
    }

    #[sqlx::test]
    async fn archive_is_all_or_nothing(pool: PgPool) {
        let app = test_app(pool.clone()).await;
        send(app.clone(), post_json("/tasks", r#"{"title":"Buy milk"}"#)).await;
        sqlx::query!("UPDATE tasks SET done = true WHERE id = 1")
            .execute(&pool)
            .await
            .unwrap();
        // Заставляем INSERT в archived_tasks упасть: id 1 там уже есть.
        sqlx::query!("INSERT INTO archived_tasks (id, title) VALUES (1, 'old')")
            .execute(&pool)
            .await
            .unwrap();

        let (status, _) = send(app.clone(), post_empty("/tasks/archive-done")).await;
        assert_eq!(status, StatusCode::INTERNAL_SERVER_ERROR);

        // DELETE откатился: задача 1 по-прежнему в `tasks`.
        let request = Request::get("/tasks/1").body(Body::empty()).unwrap();
        let (status, _) = send(app, request).await;
        assert_eq!(status, StatusCode::OK);
    }
```

```text
test tests::archive_is_all_or_nothing ... ok
```

`DELETE` выполнился, `INSERT` упал, `?` вернулся, `tx` отбросился, и PostgreSQL откатил `DELETE`: задача 1 на месте.
Уберите `tx.commit()` из обработчика или замените `&mut *tx` на `&state.pool` в `DELETE` — и этот тест упадёт. Снова
выполните `cargo sqlx prepare`, потому что в обработчике появились новые запросы.

> 🧪 **Тестирование:** чтобы проверить обработку ошибок, вызывайте сбой по-настоящему (здесь — повторяющийся ключ), а
> не только представляйте его. Тест, который заставляет средний шаг упасть, — единственное доказательство того, что
> «всё или ничего» работает.

## Частые ошибки

**1. Правка применённой миграции.** Раздел 3: `was previously applied but has been modified`. Пишите новую миграцию.

**2. Удаление файла миграции, которую применила база данных.** Тогда сервис отказывается запускаться:

```text
thread 'main' (5828436) panicked at src/main.rs:224:10:
cannot run the migrations: VersionMissing(20261118074133)
```

а `cargo sqlx migrate run` говорит `migration 20261118074133 was previously applied but is missing in the resolved
migrations`. Восстановите файл из Git. Только на своём ноутбуке можно начать с нуля через `cargo sqlx database reset -y`.

**3. Запрос через пул внутри транзакции.** `.execute(&state.pool)` между `begin` и `commit` выполняется на другом
соединении, вне транзакции. Он не откатывается и не видит незаконченных изменений транзакции. Внутри транзакции всегда
передавайте `&mut *tx`.

**4. Забытый `commit`.** Без `tx.commit().await?` работа молча откатывается, когда `tx` отбрасывается. Эндпоинт
отвечает успехом, а ничего не изменилось. Тест в стиле раздела 7 это ловит.

**5. Длинные транзакции.** Транзакция, которая ждёт пользователя, загрузку файла или другой сервис, держит соединение
и блокировки. Сначала сделайте медленную работу, а затем откройте транзакцию для быстрой части с базой данных.

## Попробуйте сами

1. Выполните `cargo sqlx migrate info` и `\d archived_tasks` в `psql`.
2. Выполните `cargo sqlx database reset -y`, затем `cargo sqlx migrate info`: всё снова применено по порядку.
3. В `psql` попробуйте `BEGIN; DELETE FROM tasks; SELECT count(*) FROM tasks; ROLLBACK; SELECT count(*) FROM tasks;` и
   посмотрите, как задачи возвращаются.
4. Сделайте коммит: `git add code/p16-tasks && git commit -m "p16-tasks: migrations and an atomic archive"`.

## Упражнения

### Упражнение 1 — Третья миграция (легко)

Большинство запросов спрашивает об открытых задачах. Добавьте миграцию `index_open_tasks`, которая создаёт
**частичный индекс** — индекс, содержащий только строки, подходящие под условие:
`CREATE INDEX tasks_open_idx ON tasks (id) WHERE NOT done;`. Примените её и покажите, что она установлена и видна в
`\d tasks`.

<details>
<summary>Подсказка</summary>

`cargo sqlx migrate add index_open_tasks`, запишите SQL в новый файл, `cargo sqlx migrate run`.

</details>

<details>
<summary>Ответ</summary>

```sql
-- Большинство запросов спрашивает об открытых задачах.
CREATE INDEX tasks_open_idx ON tasks (id) WHERE NOT done;
```

```text
Applied 20261118074133/migrate index open tasks (129.933667ms)
20261118073832/installed create tasks
20261118073943/installed create archived tasks
20261118074133/installed index open tasks
```

`\d tasks` теперь показывает `"tasks_open_idx" btree (id) WHERE NOT done` в разделе `Indexes`.

Почему это работает: в индексе только открытые задачи, поэтому он остаётся маленьким, пока выполненные задачи
накапливаются. Поскольку это миграция, каждая база данных (и каждый тест) получает его автоматически. Не забудьте
закоммитить новый файл вместе с кодом.

</details>

### Упражнение 2 — Тест удачного пути (средне)

Напишите `#[sqlx::test]` `archive_moves_only_done_tasks`: создайте `a`, `b`, `c`, отметьте задачи 1 и 3 выполненными
через `PATCH`, вызовите `POST /tasks/archive-done` и проверьте ответ `{"archived":2}` и то, что `GET /tasks` возвращает
только `b`.

<details>
<summary>Подсказка</summary>

`Request::patch(format!("/tasks/{id}"))` принимает и `String`. Используйте `post_empty` из раздела 7.

</details>

<details>
<summary>Ответ</summary>

```rust
    #[sqlx::test]
    async fn archive_moves_only_done_tasks(pool: PgPool) {
        let app = test_app(pool).await;
        for title in ["a", "b", "c"] {
            let body = format!(r#"{{"title":"{title}"}}"#);
            let request = Request::post("/tasks")
                .header(header::CONTENT_TYPE, "application/json")
                .body(Body::from(body))
                .unwrap();
            send(app.clone(), request).await;
        }
        for id in [1, 3] {
            let request = Request::patch(format!("/tasks/{id}"))
                .header(header::CONTENT_TYPE, "application/json")
                .body(Body::from(r#"{"done":true}"#))
                .unwrap();
            send(app.clone(), request).await;
        }

        let (status, body) = send(app.clone(), post_empty("/tasks/archive-done")).await;
        assert_eq!(status, StatusCode::OK);
        assert_eq!(body, r#"{"archived":2}"#);

        let request = Request::get("/tasks").body(Body::empty()).unwrap();
        let (_, body) = send(app, request).await;
        assert_eq!(body, r#"[{"id":2,"title":"b","done":false}]"#);
    }
```

Почему это работает: тест идёт только через публичный API (Application Programming Interface, программный интерфейс), как клиент, поэтому остаётся верным, даже если обработчик
перепишут (например, одним SQL-оператором). Вместе с `archive_is_all_or_nothing` он покрывает оба исхода.

</details>

### Упражнение 3 — Отбросить значит откатить (сложно)

Докажите правило «отброшенная транзакция откатывается» без всякого HTTP (HyperText Transfer Protocol, протокол передачи гипертекста): напишите `#[sqlx::test]`
`dropped_transaction_is_rolled_back`, который начинает транзакцию, вставляет через неё задачу, позволяет транзакции
выйти из области видимости **без** `commit`, а затем проверяет через `query_scalar!`, что `tasks` пуста.

<details>
<summary>Подсказка</summary>

Поместите транзакцию в отдельный блок `{ ... }`; она отбрасывается на закрывающей скобке. С точки зрения макроса
`count(*)` может быть `NULL`, поэтому назовите столбец `AS "count!"`, чтобы получить `i64`.

</details>

<details>
<summary>Ответ</summary>

```rust
    #[sqlx::test]
    async fn dropped_transaction_is_rolled_back(pool: PgPool) {
        {
            let mut tx = pool.begin().await.unwrap();
            sqlx::query!("INSERT INTO tasks (title) VALUES ('never saved')")
                .execute(&mut *tx)
                .await
                .unwrap();
            // Без commit: `tx` отбрасывается в конце этого блока.
        }
        let count = sqlx::query_scalar!(r#"SELECT count(*) AS "count!" FROM tasks"#)
            .fetch_one(&pool)
            .await
            .unwrap();
        assert_eq!(count, 0);
    }
```

С двумя упражнениями этого урока (и без тестов из упражнений Урока 16.6) `cargo test` запускает десять тестов:

```text
test result: ok. 10 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 1.99s
```

Почему это работает: `Transaction` реализует `Drop`; если она не была зафиксирована, она откатывается (sqlx отправляет
`ROLLBACK`, когда соединение используется в следующий раз или возвращается в пул). Именно поэтому `?` в
`archive_done` достаточно для безопасности. Тест документирует поведение, на которое опирается ваш код.

</details>

## Вопросы для самопроверки

1. Что хранит таблица `_sqlx_migrations` и зачем?

<details>
<summary>Ответ</summary>

Какие миграции применены к этой базе данных (версия, описание, успех) и контрольную сумму каждого файла. Благодаря ей
`migrate run` применяет только новые миграции и обнаруживает изменённые.

</details>

2. Вы нашли ошибку в миграции, которая уже в продакшене. Что делать?

<details>
<summary>Ответ</summary>

Оставить её как есть и написать новую миграцию, которая исправит схему (например, `ALTER TABLE ...`). Правка старого
файла изменила бы его контрольную сумму, и он никогда больше не выполнился бы на базах, где уже применён.

</details>

3. Что делает `sqlx::migrate!()`?

<details>
<summary>Ответ</summary>

При компиляции он встраивает файлы папки `migrations` в программу; `.run(&pool)` применяет недостающие при запуске
сервиса.

</details>

4. Что происходит с `'First'` в транзакции, где более поздний оператор завершился ошибкой, и почему?

<details>
<summary>Ответ</summary>

Оно тоже отменяется. Транзакция атомарна: после ошибки PostgreSQL отклоняет дальнейшие команды, и транзакция может
закончиться только откатом.

</details>

5. Что происходит в sqlx, если функция досрочно возвращается через `?` после `begin()`, но до `commit()`?

<details>
<summary>Ответ</summary>

Значение `Transaction` отбрасывается без фиксации, и sqlx откатывает транзакцию. Ни одно из её изменений не
сохраняется.

</details>

6. Почему транзакции должны быть короткими?

<details>
<summary>Ответ</summary>

Пока транзакция открыта, она держит соединение из пула и блокирует изменённые строки, так что другим запросам,
возможно, придётся ждать. Медленную работу делайте вне её.

</details>

## Итоги

- Миграция — пронумерованный SQL-файл, который меняет схему один раз для каждой базы данных; `_sqlx_migrations`
  записывает, какие выполнены.
- `cargo sqlx migrate add`, `migrate run` и `migrate info` создают, применяют и показывают миграции; `database reset`
  пересобирает базу данных для разработки.
- Никогда не правьте и не удаляйте применённую миграцию; добавьте новую. sqlx проверяет контрольные суммы и не
  допускает расхождения.
- `sqlx::migrate!().run(&pool)` применяет недостающие миграции при запуске; `#[sqlx::test]` применяет их к каждой
  тестовой базе.
- Транзакция — это всё или ничего (ACID). В sqlx: `pool.begin()`, запросы через `&mut *tx`, `tx.commit()`;
  отброшенная без фиксации транзакция откатывается.
- Проверяйте сбой посередине, а не только удачный путь.

## Источники и что почитать

- [sqlx `migrate!` macro](https://docs.rs/sqlx/0.9/sqlx/macro.migrate.html) — встраивание и выполнение миграций из программы.
- [sqlx-cli README](https://github.com/launchbadge/sqlx/blob/main/sqlx-cli/README.md) — `database create/drop/reset` и `migrate add/run/info/revert`, включая обратимые миграции.
- [sqlx `Transaction`](https://docs.rs/sqlx/0.9/sqlx/struct.Transaction.html) — `begin`, `commit`, `rollback` и откат при отбрасывании.
- [PostgreSQL tutorial — Transactions](https://www.postgresql.org/docs/18/tutorial-transactions.html) — `BEGIN`, `COMMIT`, `ROLLBACK` и точки сохранения на примере.
- [PostgreSQL — Transaction isolation](https://www.postgresql.org/docs/18/transaction-iso.html) — подробно об уровнях изоляции (подготовка к Этапу 17).

---

[🏠 Начало](../START-HERE.md) · [← 16.6 Работа с базами данных через sqlx](06-databases-with-sqlx.md) · [📋 Этап 16](README.md) · [16.8 Аутентификация в веб-сервисе →](08-authentication-in-a-web-service.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
