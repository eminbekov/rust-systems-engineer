<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 16.4 Ошибки, проверка ввода и middleware](04-errors-validation-and-middleware.md) · [📋 Этап 16](README.md) · [16.6 Работа с базами данных через sqlx →](06-databases-with-sqlx.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/16-backend-services/05-sql-and-postgresql-basics.md) <!-- nav -->

# Урок 16.5 — Основы SQL и PostgreSQL

> **Этап 16 — Бэкенд-сервисы** · Урок 5 из 16 · ⏱ около 2,5 часа (2 занятия)

## Что вы узнаете

- Что такое реляционная база данных: таблицы, строки, столбцы, типы и ключи.
- Как запустить PostgreSQL в Docker и общаться с ним через `psql`.
- Четыре основные команды SQL (Structured Query Language, язык структурированных запросов) — `INSERT`, `SELECT`, `UPDATE` и `DELETE` — вместе с `WHERE`, `ORDER BY`, `LIMIT` и `RETURNING`.
- Как ограничения (`NOT NULL`, `CHECK`, `PRIMARY KEY`, `REFERENCES`) позволяют самой базе данных отклонять плохие данные.
- Как связать две таблицы внешним ключом и `JOIN` и зачем нужен индекс.

## Перед началом

- Вы прошли [Урок 16.4 — Ошибки, проверка ввода и middleware](04-errors-validation-and-middleware.md).
- Вы видели немного SQL с SQLite в [Уроке 10.5 — Атаки внедрения](../10-security-fundamentals/05-injection-attacks.md):
  `CREATE TABLE`, `INSERT`, `SELECT` и подстановки. Этот урок всё равно начинает с нуля.
- Вы знаете контейнеры, образы и тома (Урок 11.12 — Контейнеры и основы Docker). Docker Desktop запущен.

## Новые слова

| Слово | Значение |
|---|---|
| **SQL** (Structured Query Language, язык структурированных запросов) | Язык, на котором реляционную базу данных просят сохранить, изменить и найти данные. |
| **реляционная база данных** (relational database) | База данных, которая хранит данные в таблицах и позволяет объединять их в запросах. PostgreSQL, MySQL и SQLite — реляционные базы данных. |
| **PostgreSQL** (часто «Postgres») | Бесплатный сервер реляционной базы данных с открытым исходным кодом, очень распространённый в бэкенд-сервисах. |
| **таблица / строка / столбец** (table / row / column) | Таблица похожа на электронную таблицу: каждая **строка** — одна запись (одна задача), каждый **столбец** — одно поле с фиксированным типом (`title`, `done`). |
| **запрос** (query) | Одна команда SQL, отправленная базе данных, например `SELECT * FROM tasks`. |
| **первичный ключ** (primary key) | Столбец (или столбцы), который идентифицирует каждую строку; он должен быть уникальным и никогда не пустым. |
| **ограничение** (constraint) | Правило, которое база данных проверяет при каждом изменении: `NOT NULL`, `CHECK`, `UNIQUE`, `PRIMARY KEY`, `REFERENCES`. Изменение, которое его нарушает, не выполняется. |
| **внешний ключ** (foreign key) | Столбец, который указывает на первичный ключ другой таблицы (`comments.task_id` → `tasks.id`). База данных следит, чтобы цель существовала. |
| **`NULL`** | Отметка SQL для «нет значения / неизвестно». Это не ноль и не пустая строка. |
| **соединение** (join) | Объединение в одном результате запроса строк двух таблиц, которые относятся друг к другу. |
| **индекс** (index) | Дополнительная структура, которую база данных хранит рядом с таблицей, чтобы быстро находить строки по столбцу, не читая всю таблицу. |
| **`psql`** | Клиент командной строки PostgreSQL: вы набираете SQL, он показывает результаты. |
| **DDL / DML** (Data Definition / Data Manipulation Language, язык определения / изменения данных) | Две половины SQL: DDL меняет структуру (`CREATE TABLE`, `ALTER TABLE`), DML меняет данные (`INSERT`, `UPDATE`, `DELETE`). |

## 1. Зачем нужен сервер базы данных?

Урок 16.1 закончился правилом: данные, которые должны пережить перезапуск и быть общими для всех экземпляров,
живут вне сервиса. Можно писать их в файл, но тогда придётся самим решать много трудных задач: два экземпляра
пишут в один и тот же момент, сбой посреди записи, поиск одной задачи среди миллионов без чтения их всех,
гарантия, что ни у одной задачи нет пустого заголовка. **Сервер базы данных** решает эти задачи один раз и очень
хорошо:

- он хранит данные на диске так, что они переживают сбои (Этап 17 показывает как);
- он позволяет многим клиентам читать и писать одновременно, не портя данные;
- он быстро находит данные с помощью индексов;
- он проверяет правила (ограничения) при каждом изменении, кто бы его ни отправил.

```text
  service instance 1 ──┐
  service instance 2 ──┼── TCP, port 5432 ──▶  PostgreSQL server ──▶ files on disk (a Docker volume)
  psql (you)         ──┘
```

PostgreSQL — отдельная программа, сервер, точно как ваш сервис. Клиенты подключаются к нему по TCP (по умолчанию
порт `5432`) и отправляют текст SQL; он отвечает строками.

## 2. PostgreSQL в Docker

Официальный образ `postgres:18` даёт готовый сервер одной командой:

```bash
docker run -d --name p16-postgres \
  -e POSTGRES_PASSWORD=dev-password \
  -p 127.0.0.1:5432:5432 \
  -v p16-pgdata:/var/lib/postgresql \
  postgres:18
```

**Что происходит:**

1. `-d` запускает контейнер в фоне и печатает его длинный идентификатор.
2. `--name p16-postgres` даёт ему имя, чтобы последующие команды могли писать `p16-postgres` вместо
   идентификатора.
3. `-e POSTGRES_PASSWORD=dev-password` задаёт пароль суперпользователя `postgres`. Без него образ отказывается
   запускаться. Это пароль **для разработки** — для базы данных на вашем ноутбуке, не более того.
4. `-p 127.0.0.1:5432:5432` делает порт 5432 контейнера доступным на порту 5432 вашего Mac, но **только** с вашего
   Mac (`127.0.0.1`).
5. `-v p16-pgdata:/var/lib/postgresql` хранит файлы базы данных в томе Docker с именем `p16-pgdata`. Контейнер
   можно удалить и создать заново; данные остаются в томе.

Проверьте, что он готов (`docker logs` показывает, что напечатал сервер):

```bash
docker logs p16-postgres 2>&1 | tail -1
```

```text
2026-11-18 07:23:07.761 UTC [1] LOG:  database system is ready to accept connections
```

> 🔒 **Безопасность:** привязка к `127.0.0.1` важна. С `-p 5432:5432` база данных слушала бы в каждой сети, к
> которой подключён ваш Mac, например на Wi-Fi в кафе, защищённая лишь паролем `dev-password`. Настоящие пароли
> никогда не попадают в команды или в Git; Урок 16.13 и Урок 10.8 показывают, как обращаться с секретами.

`psql` уже есть внутри контейнера. Запустите его там (`-it` подключает к нему вашу клавиатуру и терминал):

```bash
docker exec -it p16-postgres psql -U postgres
```

```text
psql (18.7 (Debian 18.7-1.pgdg13+2))
Type "help" for help.

postgres=#
```

`postgres=#` — приглашение: вы подключены к базе данных `postgres` как суперпользователь. Сервер хранит много
**баз данных**; создайте одну для сервиса задач и переключитесь на неё. Каждая команда SQL заканчивается на `;`.
Команды, которые начинаются с обратного слеша, как `\c` («connect», подключиться), принадлежат самому `psql` и не
требуют `;`:

```sql
CREATE DATABASE tasks;
\c tasks
```

```text
CREATE DATABASE
You are now connected to database "tasks" as user "postgres".
```

Теперь приглашение — `tasks=#`. Полезные команды `psql`: `\dt` перечисляет таблицы, `\d name` описывает одну
таблицу, `\q` выходит. Весь SQL в остальной части урока набирается в этом приглашении.

> 💡 **Совет:** ключевые слова SQL не чувствительны к регистру; `select` работает как `SELECT`. Писать ключевые слова
> заглавными, а имена строчными — распространённое соглашение, которое облегчает чтение запросов.

## 3. Таблица

```sql
CREATE TABLE tasks (
    id         BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    title      TEXT NOT NULL CHECK (char_length(title) BETWEEN 1 AND 200),
    done       BOOLEAN NOT NULL DEFAULT false,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

**Что происходит:**

1. Каждая строка внутри скобок — один столбец: имя, **тип** и правила.
2. `BIGINT` — 64-битное целое со знаком (как `i64`). `GENERATED ALWAYS AS IDENTITY` значит: номер — 1, 2, 3, … —
   для каждой новой строки выбирает база данных. `PRIMARY KEY` делает `id` уникальным идентификатором строки.
3. `TEXT` — строка любой длины. `NOT NULL` запрещает «нет значения». `CHECK (...)` — условие, которому должна
   отвечать каждая строка: то же правило «от 1 до 200 символов», что и в `Title::parse` из Урока 16.4.
4. `BOOLEAN` — `true`/`false`. `DEFAULT false`: если `INSERT` не упоминает `done`, он равен `false`.
5. `TIMESTAMPTZ` — момент времени с часовым поясом. `DEFAULT now()` подставляет момент вставки.

`\d tasks` показывает, что поняла база данных:

```text
                                    Table "public.tasks"
   Column   |           Type           | Collation | Nullable |           Default
------------+--------------------------+-----------+----------+------------------------------
 id         | bigint                   |           | not null | generated always as identity
 title      | text                     |           | not null |
 done       | boolean                  |           | not null | false
 created_at | timestamp with time zone |           | not null | now()
Indexes:
    "tasks_pkey" PRIMARY KEY, btree (id)
Check constraints:
    "tasks_title_check" CHECK (char_length(title) >= 1 AND char_length(title) <= 200)
```

Первичный ключ автоматически получил индекс (`tasks_pkey`, раздел 7), а `CHECK` получил имя `tasks_title_check`.
`public` — схема по умолчанию, «папка» для таблиц внутри базы данных.

| Rust | PostgreSQL |
|---|---|
| `i32` / `i64` | `INTEGER` / `BIGINT` |
| `String` | `TEXT` (или `VARCHAR(n)` с максимальной длиной) |
| `bool` | `BOOLEAN` |
| `f64` | `DOUBLE PRECISION` (никогда для денег: используйте `NUMERIC`) |
| момент времени | `TIMESTAMPTZ` |
| `Option<T>` | столбец **без** `NOT NULL` |

## 4. Вставка, чтение, изменение, удаление

**`INSERT`** добавляет строки. Вы называете столбцы, которые передаёте; остальные получают значение по умолчанию:

```sql
INSERT INTO tasks (title) VALUES ('Buy milk');
INSERT INTO tasks (title) VALUES ('Write lesson'), ('Call mom') RETURNING id, title, done;
```

```text
INSERT 0 1
 id |    title     | done
----+--------------+------
  2 | Write lesson | f
  3 | Call mom     | f
(2 rows)

INSERT 0 2
```

`INSERT 0 1` означает «вставлена одна строка» (`0` — исторический пережиток). `RETURNING` возвращает столбцы новых
строк: так сервис узнаёт `id`, выбранный базой данных. Текстовые значения пишутся в **одинарных** кавычках. `f` и
`t` — так `psql` печатает `false` и `true`.

**`SELECT`** читает. `*` означает «все столбцы»:

```sql
SELECT * FROM tasks;
```

```text
 id |    title     | done |          created_at
----+--------------+------+------------------------------
  1 | Buy milk     | f    | 2026-11-18 07:23:12.18793+00
  2 | Write lesson | f    | 2026-11-18 07:23:12.18828+00
  3 | Call mom     | f    | 2026-11-18 07:23:12.18828+00
(3 rows)
```

`WHERE` оставляет только подходящие строки, `ORDER BY` сортирует (`DESC` — по убыванию), `LIMIT` возвращает не
больше заданного числа строк:

```sql
SELECT id, title FROM tasks WHERE done = false ORDER BY id DESC LIMIT 2;
```

```text
 id |    title
----+--------------
  3 | Call mom
  2 | Write lesson
(2 rows)
```

> ⚠️ **Частая ошибка:** без `ORDER BY` база данных может вернуть строки в **любом** порядке, и порядок может меняться
> от запуска к запуску. Если порядок важен (а в API — Application Programming Interface, программном интерфейсе — он важен всегда), укажите его.

**`UPDATE`** меняет столбцы строк, подходящих под `WHERE`. **`DELETE`** удаляет их:

```sql
UPDATE tasks SET done = true WHERE id = 1 RETURNING id, title, done;
SELECT count(*) FROM tasks WHERE NOT done;
DELETE FROM tasks WHERE id = 3;
SELECT id, title, done FROM tasks ORDER BY id;
```

```text
 id |  title   | done
----+----------+------
  1 | Buy milk | t
(1 row)

UPDATE 1
 count
-------
     2
(1 row)

DELETE 1
 id |    title     | done
----+--------------+------
  1 | Buy milk     | t
  2 | Write lesson | f
(2 rows)
```

`count(*)` — **агрегатная функция**: она превращает много строк в одно число. `UPDATE 1` и `DELETE 1` сообщают,
сколько строк затронуто; сервис использует это число, чтобы ответить `404`, когда оно равно `0`.

> ⚠️ **Частая ошибка:** `UPDATE tasks SET done = true;` или `DELETE FROM tasks;` **без** `WHERE` меняет или удаляет
> **каждую** строку. `psql` не спрашивает «вы уверены?». Пишите `WHERE` первым.

## 5. Ограничения: база данных говорит «нет»

Теперь нарушьте правила нарочно. Каждая команда завершается понятной ошибкой, и ничего не сохраняется:

```sql
INSERT INTO tasks (title) VALUES ('');
INSERT INTO tasks (title, done) VALUES ('Read', NULL);
INSERT INTO tasks (id, title) VALUES (10, 'Pick an id');
UPDATE tasks SET done = 'maybe' WHERE id = 2;
SELECT * FROM taks;
```

```text
ERROR:  new row for relation "tasks" violates check constraint "tasks_title_check"
DETAIL:  Failing row contains (4, , f, 2026-11-18 07:23:12.189411+00).
ERROR:  null value in column "done" of relation "tasks" violates not-null constraint
DETAIL:  Failing row contains (5, Read, null, 2026-11-18 07:23:12.189521+00).
ERROR:  cannot insert a non-DEFAULT value into column "id"
DETAIL:  Column "id" is an identity column defined as GENERATED ALWAYS.
HINT:  Use OVERRIDING SYSTEM VALUE to override.
ERROR:  invalid input syntax for type boolean: "maybe"
LINE 1: UPDATE tasks SET done = 'maybe' WHERE id = 2;
                                ^
ERROR:  relation "taks" does not exist
LINE 1: SELECT * FROM taks;
                      ^
```

**Что происходит:**

1. Пустой заголовок нарушает `tasks_title_check`.
2. `NULL` в столбце `NOT NULL` отклоняется.
3. С `GENERATED ALWAYS` идентификаторы выбирает только база данных; клиент не может выбрать `10`.
4. Каждое значение должно подходить к типу столбца: `'maybe'` — не логическое значение.
5. Опечатка в имени таблицы («relation» — так PostgreSQL называет таблицу) обнаруживается до того, как что-то
   выполнится.

Посмотрите на строки `DETAIL`: у неудачных вставок уже были идентификаторы `4` и `5`. База данных берёт следующий
номер **до** проверки строки, а неудачная вставка номер не возвращает. Вставьте ещё одну задачу:

```sql
INSERT INTO tasks (title) VALUES ('Read a book') RETURNING id;
```

```text
 id
----
  6
(1 row)

INSERT 0 1
```

В идентификаторах бывают **пропуски**. Это нормально и правильно: относитесь к идентификаторам как к именам,
никогда как к «количеству задач».

> 🔒 **Безопасность:** сервис (Урок 16.4) уже проверяет заголовки, так зачем ещё и в базе данных? *Эшелонированная
> защита* (defense in depth, Урок 10.1): в ту же таблицу может писать другая программа, скрипт или будущая ошибка.
> База данных — последний рубеж, и она защищает данные, кто бы их ни писал.

**`NULL` в одном абзаце.** `NULL` означает «неизвестно», поэтому сравнение с ним тоже даёт «неизвестно»:
`NULL = NULL` — не `true`, а `NULL`, и `WHERE` с `NULL` ничего не оставляет. Для проверки используйте
`IS NULL` / `IS NOT NULL`:

```sql
SELECT NULL = NULL AS equal, NULL IS NULL AS is_null;
```

```text
 equal | is_null
-------+---------
       | t
(1 row)
```

`AS` даёт столбцу результата имя. Пустая ячейка — это `NULL`. Большинство ошибок с `NULL` исчезает, если столбцы
объявлены `NOT NULL`, когда «нет значения» действительно не нужно.

## 6. Две таблицы: внешние ключи и соединения

В Уроке 16.2 мы спроектировали комментарии к задачам. Каждый комментарий принадлежит одной задаче, поэтому таблица
комментариев **указывает** на таблицу задач:

```sql
CREATE TABLE comments (
    id      BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    task_id BIGINT NOT NULL REFERENCES tasks (id) ON DELETE CASCADE,
    text    TEXT NOT NULL
);
INSERT INTO comments (task_id, text) VALUES (1, 'Use the oat milk'), (1, 'Two bottles');
INSERT INTO comments (task_id, text) VALUES (99, 'Lost comment');
```

```text
CREATE TABLE
INSERT 0 2
ERROR:  insert or update on table "comments" violates foreign key constraint "comments_task_id_fkey"
DETAIL:  Key (task_id)=(99) is not present in table "tasks".
```

`REFERENCES tasks (id)` делает `task_id` **внешним ключом**: база данных отклоняет комментарий к несуществующей
задаче. `ON DELETE CASCADE` говорит, что происходит при удалении задачи: её комментарии тоже удаляются.

Чтобы увидеть задачи и их комментарии **вместе**, соедините таблицы. `JOIN ... ON` ставит в пару каждую строку
задачи с каждой строкой комментария, у которой `task_id` равен `id` задачи:

```sql
SELECT tasks.title, comments.text
FROM tasks
JOIN comments ON comments.task_id = tasks.id;
```

```text
  title   |       text
----------+------------------
 Buy milk | Use the oat milk
 Buy milk | Two bottles
(2 rows)
```

Задач без комментариев нет: простой `JOIN` оставляет только существующие пары. `LEFT JOIN` оставляет **каждую**
строку левой таблицы, с `NULL` там, где партнёра нет. Вместе с `GROUP BY` (одна строка результата на задачу) и
`count` он отвечает на вопрос «сколько комментариев у каждой задачи?»:

```sql
SELECT tasks.id, tasks.title, count(comments.id) AS comments
FROM tasks
LEFT JOIN comments ON comments.task_id = tasks.id
GROUP BY tasks.id
ORDER BY tasks.id;
```

```text
 id |    title     | comments
----+--------------+----------
  1 | Buy milk     |        2
  2 | Write lesson |        0
  6 | Read a book  |        0
(3 rows)
```

`count(comments.id)` считает только значения, не равные `NULL`, поэтому задачи без комментариев получают `0`.

```sql
DELETE FROM tasks WHERE id = 1;
SELECT count(*) FROM comments;
```

```text
DELETE 1
 count
-------
     0
(1 row)
```

`ON DELETE CASCADE` удалил оба комментария задачи 1. Без него `DELETE` не выполнился бы, потому что комментарии
указывали бы в никуда.

## 7. Индексы

Чтобы выполнить `WHERE comments.task_id = 6`, база данных должна найти подходящие строки. Без помощи она читает
**каждую** строку таблицы (*последовательное сканирование*): для сотни строк это нормально, для сотни миллионов —
медленно. **Индекс** — отсортированная структура (обычно B-дерево, Урок 17.6) от значений столбца к строкам, как
предметный указатель в конце книги.

PostgreSQL создаёт индекс для каждого первичного ключа и ограничения `UNIQUE`, но **не** для внешних ключей.
Создайте его, потому что «комментарии задачи N» — запрос, который сервис выполняет постоянно:

```sql
CREATE INDEX comments_task_id_idx ON comments (task_id);
```

`EXPLAIN` показывает **план**: как база данных выполнит запрос, не выполняя его:

```sql
EXPLAIN SELECT * FROM tasks WHERE id = 2;
```

```text
                               QUERY PLAN
-------------------------------------------------------------------------
 Index Scan using tasks_pkey on tasks  (cost=0.15..8.17 rows=1 width=49)
   Index Cond: (id = 2)
(2 rows)
```

«Index Scan using tasks_pkey»: база данных перейдёт к строке через индекс первичного ключа, а не будет читать
таблицу. Урок 17.13 подробно объясняет планы.

> ⚡ **Производительность:** индекс делает чтение по этому столбцу быстрым, но каждый `INSERT`, `UPDATE` и `DELETE`
> должен обновлять и индекс, а ещё он занимает место на диске. Создавайте индексы для запросов, которые вы
> действительно выполняете (внешние ключи, столбцы в частых `WHERE` и `ORDER BY`), а не для каждого столбца.

## 8. Данные сохраняются

Выйдите из `psql` через `\q` и перезапустите контейнер, как это сделал бы сбой или перезагрузка:

```bash
docker restart p16-postgres
docker exec -it p16-postgres psql -U postgres -d tasks -c 'SELECT id, title, done FROM tasks ORDER BY id;'
```

```text
 id |    title     | done
----+--------------+------
  2 | Write lesson | f
  6 | Read a book  | f
(2 rows)
```

`-d tasks` подключается сразу к базе данных `tasks`; `-c '...'` выполняет одну команду и выходит. Строки на месте:
они живут в томе `p16-pgdata`, а не в памяти процесса. Именно этого не умело хранилище в памяти из Урока 16.3.

Оставьте контейнер работать для следующих уроков или остановите его через `docker stop p16-postgres` и снова
запустите через `docker start p16-postgres`. Когда закончите весь этап, удалите его через `docker rm -f p16-postgres`
и `docker volume rm p16-pgdata` (это удаляет данные).

## Частые ошибки

**1. Двойные кавычки для текста.** `WHERE title = "Buy milk"` не работает: в SQL двойные кавычки — для **имён**
(столбцов, таблиц), поэтому PostgreSQL ищет столбец с именем `Buy milk`:

```text
ERROR:  column "Buy milk" does not exist
```

Для текста используйте одинарные кавычки: `'Buy milk'`. Одинарная кавычка внутри текста пишется дважды:
`'Mom''s birthday'`.

**2. Забытый `WHERE`** в `UPDATE` или `DELETE` (раздел 4). Меняется каждая строка.

**3. Сравнение через `= NULL`.** `WHERE done = NULL` никогда ничему не соответствует. Используйте `IS NULL`.

**4. Сборка SQL вставкой значений в текст.** Это SQL-внедрение (Урок 10.5). В `psql` вы набираете значения вручную;
в коде на Rust вы всегда будете передавать их отдельно, как параметры (`$1`, `$2`), как показывает Урок 16.6.

**5. Ожидание идентификаторов без пропусков.** Неудачные вставки (а в Уроке 16.7 — откаченные транзакции) оставляют
пропуски.

## Попробуйте сами

1. Выполните `\dt` и `\d comments`. Найдите внешний ключ в выводе.
2. Сложите обе команды `CREATE TABLE` в файл `code/p16-tasks/schema.sql` (они снова понадобятся в Уроке 16.6), затем
   проверьте, что файл корректен, выполнив его на новой пустой базе данных:

   ```bash
   docker exec -it p16-postgres psql -U postgres -c 'CREATE DATABASE scratch;'
   docker exec -i p16-postgres psql -U postgres -d scratch < code/p16-tasks/schema.sql
   ```

   (`-i` без `-t`, потому что ввод идёт из файла, а не с клавиатуры.) Вы должны увидеть `CREATE TABLE` дважды. Затем
   удалите тестовую базу: `docker exec -it p16-postgres psql -U postgres -c 'DROP DATABASE scratch;'`.
3. В `psql` выполните `SELECT now();` дважды и посмотрите, как меняется время.

## Упражнения

Выполняйте их в `psql`, подключившись к базе данных `tasks`. Сначала добавьте две задачи:
`INSERT INTO tasks (title) VALUES ('Buy oat milk'), ('Milk the cow');`.

### Упражнение 1 — Открытые задачи, сначала новые (легко)

Выведите `id`, `title` и `created_at` всех невыполненных задач, сначала самые новые. У задач, вставленных одной
командой, одинаковый `created_at`; для них ставьте первой задачу с большим `id`.

<details>
<summary>Подсказка</summary>

`ORDER BY` принимает несколько столбцов через запятую; второй решает только тогда, когда первые равны.

</details>

<details>
<summary>Ответ</summary>

```sql
SELECT id, title, created_at FROM tasks WHERE NOT done ORDER BY created_at DESC, id DESC;
```

```text
 id |    title     |          created_at
----+--------------+-------------------------------
  8 | Milk the cow | 2026-11-18 07:23:26.631474+00
  7 | Buy oat milk | 2026-11-18 07:23:26.631474+00
  6 | Read a book  | 2026-11-18 07:23:12.189823+00
  2 | Write lesson | 2026-11-18 07:23:12.18828+00
(4 rows)
```

Почему это работает: `now()` — время **транзакции**, поэтому обе строки одного `INSERT` получают одинаковое
значение, и второй ключ сортировки разрешает ничью. Ваше время, конечно, будет другим.

</details>

### Упражнение 2 — Всё с молоком (средне)

Отметьте выполненными **одной** командой все задачи, в заголовке которых есть слово «milk» в любом регистре («Buy oat
milk», «Milk the cow»), и покажите `id` и `title` изменённых строк.

<details>
<summary>Подсказка</summary>

`LIKE` сравнивает с шаблоном, в котором `%` означает «любой текст»; `ILIKE` — то же самое, но без учёта регистра.

</details>

<details>
<summary>Ответ</summary>

```sql
UPDATE tasks SET done = true WHERE title ILIKE '%milk%' RETURNING id, title;
```

```text
 id |    title
----+--------------
  7 | Buy oat milk
  8 | Milk the cow
(2 rows)

UPDATE 2
```

Почему это работает: `'%milk%'` подходит к любому заголовку с «milk» в любом месте; `ILIKE` позволяет подойти и
«Milk». `RETURNING` показывает, какие именно строки затронул `UPDATE`, так что рискованное изменение можно сразу
проверить.

</details>

### Упражнение 3 — Тест внутри базы данных (сложно)

Напишите **тест** для ограничения `CHECK` на самом SQL: он должен попытаться вставить пустой заголовок, напечатать
`test passed: the empty title was rejected`, если PostgreSQL его отклоняет, и завершиться ошибкой, если PostgreSQL
его принимает. Затем докажите, что тест может упасть: удалите ограничение через
`ALTER TABLE tasks DROP CONSTRAINT tasks_title_check;`, снова запустите тест и наконец восстановите ограничение через
`ALTER TABLE tasks ADD CONSTRAINT tasks_title_check CHECK (char_length(title) BETWEEN 1 AND 200);`.

<details>
<summary>Подсказка</summary>

PostgreSQL выполняет маленькие программы через `DO $$ BEGIN ... END $$;` (язык — PL/pgSQL, Procedural Language/PostgreSQL, процедурный язык PostgreSQL). Внутри
`RAISE EXCEPTION '...'` завершается ошибкой, `RAISE NOTICE '...'` печатает сообщение, а блок может перехватить ошибку
по имени через `EXCEPTION WHEN check_violation THEN ...`.

</details>

<details>
<summary>Ответ</summary>

```sql
DO $$
BEGIN
    INSERT INTO tasks (title) VALUES ('');
    RAISE EXCEPTION 'test failed: an empty title was accepted';
EXCEPTION WHEN check_violation THEN
    RAISE NOTICE 'test passed: the empty title was rejected';
END
$$;
```

С ограничением:

```text
NOTICE:  test passed: the empty title was rejected
DO
```

Без ограничения:

```text
ERROR:  test failed: an empty title was accepted
CONTEXT:  PL/pgSQL function inline_code_block line 4 at RAISE
```

Почему это работает: если `INSERT` завершается ошибкой `check_violation`, выполняется часть `EXCEPTION` и печатает
сообщение. Если `INSERT` проходит, следующая строка выбрасывает нашу собственную ошибку, и тест падает. Увидеть
падение теста один раз — значит доказать, что он действительно что-то проверяет (шаг «красный» из Урока 05.6).
Поскольку весь блок `DO` завершается ошибкой, его `INSERT` тоже отменяется:
`SELECT count(*) FROM tasks WHERE title = '';` возвращает `0`. Урок 16.7 объясняет почему (транзакции).

</details>

## Вопросы для самопроверки

1. Чем первичный ключ отличается от внешнего?

<details>
<summary>Ответ</summary>

Первичный ключ идентифицирует каждую строку **своей** таблицы (уникален, не `NULL`). Внешний ключ — столбец, который
указывает на первичный ключ **другой** таблицы, и база данных следит, чтобы целевая строка существовала.

</details>

2. Почему хорошо, что база данных проверяет заголовок, хотя сервис это уже делает?

<details>
<summary>Ответ</summary>

Эшелонированная защита: в таблицу могут писать другие программы, скрипты или будущая ошибка в сервисе. Ограничение
защищает данные, кто бы их ни писал.

</details>

3. Что делает `RETURNING id` и зачем это сервису?

<details>
<summary>Ответ</summary>

Он заставляет `INSERT` (или `UPDATE`/`DELETE`) вернуть столбцы затронутых строк. Сервис узнаёт идентификатор,
выбранный базой данных, и может ответить `201` с `location: /tasks/{id}`.

</details>

4. Почему `SELECT * FROM tasks WHERE done = NULL` ничего не возвращает?

<details>
<summary>Ответ</summary>

Любое сравнение с `NULL` — «неизвестно» (`NULL`), а `WHERE` оставляет только строки, где условие равно `true`.
Используйте `IS NULL`.

</details>

5. Чем `JOIN` отличается от `LEFT JOIN`?

<details>
<summary>Ответ</summary>

`JOIN` возвращает только совпавшие пары строк. `LEFT JOIN` возвращает каждую строку левой таблицы со столбцами
`NULL`, если справа нет подходящей строки.

</details>

6. Чего стоит индекс?

<details>
<summary>Ответ</summary>

Места на диске и дополнительной работы при каждом `INSERT`, `UPDATE` и `DELETE`, потому что индекс тоже нужно
обновлять. Взамен чтение по индексированному столбцу становится быстрым.

</details>

## Итоги

- Реляционная база данных хранит строки в таблицах с типизированными столбцами; PostgreSQL — сервер, к которому
  клиенты обращаются по TCP.
- `docker run ... postgres:18` с томом даёт вам локальный сервер; `psql` позволяет набирать SQL.
- `INSERT ... RETURNING`, `SELECT ... WHERE ... ORDER BY ... LIMIT`, `UPDATE ... WHERE`, `DELETE ... WHERE` —
  повседневные команды. Всегда пишите `ORDER BY`, когда важен порядок, и `WHERE` — до изменения данных.
- Ограничения (`NOT NULL`, `CHECK`, `PRIMARY KEY`, `REFERENCES`) заставляют базу данных отклонять плохие данные; в
  идентификаторах бывают пропуски.
- `NULL` означает «неизвестно»: проверяйте его через `IS NULL`.
- Внешние ключи связывают таблицы; `JOIN`/`LEFT JOIN` с `GROUP BY` объединяют их; индексы ускоряют поиск.

## Источники и что почитать

- [PostgreSQL 18 tutorial](https://www.postgresql.org/docs/18/tutorial.html) — официальное, понятное новичку введение в SQL с PostgreSQL.
- [psql reference](https://www.postgresql.org/docs/18/app-psql.html) — все команды `psql` с обратным слешем.
- [PostgreSQL — Constraints](https://www.postgresql.org/docs/18/ddl-constraints.html) — `CHECK`, `NOT NULL`, `UNIQUE`, первичные и внешние ключи.
- [PostgreSQL — Queries](https://www.postgresql.org/docs/18/queries.html) — `SELECT`, соединения, `GROUP BY`, `ORDER BY` и `LIMIT` подробно.
- [PostgreSQL — Data types](https://www.postgresql.org/docs/18/datatype.html) — все типы столбцов.
- [The postgres Docker image](https://hub.docker.com/_/postgres) — переменные окружения и пути томов официального образа.

---

[🏠 Начало](../START-HERE.md) · [← 16.4 Ошибки, проверка ввода и middleware](04-errors-validation-and-middleware.md) · [📋 Этап 16](README.md) · [16.6 Работа с базами данных через sqlx →](06-databases-with-sqlx.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
