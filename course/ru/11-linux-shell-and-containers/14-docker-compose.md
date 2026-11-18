<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 11.13 Dockerfile для программ на Rust](13-dockerfiles-for-rust.md) · [📋 Этап 11](README.md) · [11.15 Проект: развёртывание сервиса на Rust →](15-project-deploy-a-rust-service.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/11-linux-shell-and-containers/14-docker-compose.md) <!-- nav -->

# Урок 11.14 — Docker Compose

> **Этап 11 — Linux, командная оболочка и контейнеры** · Урок 14 из 16 · ⏱ около 135 минут

## Что вы узнаете

- Что такое Docker Compose и как один файл `compose.yaml` описывает целое приложение: службу на Rust и базу данных PostgreSQL.
- Как службы находят друг друга по имени и как `depends_on` с проверкой работоспособности управляет порядком запуска.
- Как хранить данные в томе, а пароль — в секретном файле, а не в образе и не в Git.
- Как политика перезапуска возвращает упавшую службу и что происходит, когда база данных пропадает.
- Повседневные команды: `up`, `ps`, `logs`, `exec`, `down`.

## Перед началом

- Вы прошли [Урок 11.13 — Dockerfile для программ на Rust](13-dockerfiles-for-rust.md). Папка `code/p11-hello` с её `Dockerfile` и `.dockerignore` существует, а Docker Desktop запущен.
- Вы знаете контейнеры, тома, проверки работоспособности и секреты ([Урок 11.12](12-containers-and-docker-basics.md), [Урок 10.8](../10-security-fundamentals/08-secrets-management.md)).
- Все команды выполняются на **Mac**. Docker Compose входит в Docker Desktop: `docker compose version` должен напечатать версию.
- Мы используем официальный образ `postgres:18`. Знаний о базах данных, кроме одного `SELECT`, не требуется; базы данных изучаются на Этапе 17.

## Новые слова

| Слово | Значение |
|---|---|
| **Docker Compose** | Инструмент (`docker compose …`), который запускает несколько контейнеров вместе по одному файлу описания. |
| **`compose.yaml`** | Файл описания. В нём перечислены **службы**, а также могут быть тома, сети и секреты. |
| **YAML** (YAML Ain't Markup Language, «YAML — не язык разметки») | Текстовый формат данных: пары `ключ: значение`, списки с `-` и **вложенность пробелами** (никогда не табуляцией). |
| **служба** (service) | Одна часть приложения, запускаемая как один или несколько контейнеров из одного образа, например `app` или `db`. |
| **проект** (project) | Один запущенный экземпляр файла Compose. Его имя (здесь `course-counter`) — префикс всех создаваемых контейнеров, сетей и томов. |
| **сеть по умолчанию** (default network) | Compose создаёт для проекта частную сеть. Каждая служба может обратиться к остальным по **имени службы** (например, `db`) как по имени хоста. |
| **`depends_on`** | Говорит, что службе сначала нужна другая. С `condition: service_healthy` Compose ждёт, пока пройдёт проверка работоспособности другой службы. |
| **политика перезапуска** (restart policy) | Что делает Docker, когда контейнер завершился: `no`, `on-failure`, `always`, `unless-stopped`. |
| **секрет** (secret) | Чувствительные данные (пароль), которые Compose передаёт контейнеру как файл в `/run/secrets/`, а не как переменную окружения. |
| **PostgreSQL** | Популярный сервер баз данных с открытым исходным кодом. Клиентская программа — `psql`. |
| **SQL** (Structured Query Language, язык структурированных запросов) | Язык реляционных баз данных: `INSERT`, `SELECT`, … |

## 1. Что делает Compose

Настоящее приложение редко состоит из одной программы. В нашем примере их две: веб-служба на Rust и база данных. С обычным `docker run` вы набирали бы две длинные команды в нужном порядке и создавали сеть и том вручную. **Compose** записывает всё это в файл, который вы храните в Git, и запускает всё одной командой:

```text
   your Mac
   127.0.0.1:18080 ---> +------------------ project "course-counter" -------------------+
                        |  app  (our Rust service)  --- network "default" ---  db      |
                        |       listens on 8080          (names resolve)   (postgres)  |
                        +-----------------------------------------------------|--------+
                                                                       volume "pgdata"
```

Файл использует **YAML**. Хватит трёх правил: `ключ: значение` создаёт запись словаря; `- элемент` создаёт запись списка; а **отступ** (пробелы, обычно по два на уровень) показывает, что к чему относится. Неправильный отступ меняет смысл или ломает файл, а табуляция — ошибка.

## 2. Служба на Rust с базой данных

Служба `p11-counter` — это `p11-hello` плюс один маршрут: `GET /hits` сохраняет посещение в PostgreSQL и отвечает общим числом посещений. Начните с копии:

```bash
cd ~/projects/rust/code
cp -r p11-hello p11-counter
cd p11-counter
rm -rf target Dockerfile.naive Dockerfile.distroless
sed -i '' 's/p11-hello/p11-counter/g' Cargo.toml Dockerfile src/main.rs
cargo add postgres@0.19
```

(`sed -i '' 's/a/b/g' файлы` заменяет текст внутри файлов; на Mac пустые `''` после `-i` обязательны. Не запускайте это на файлах, которые вы не копировали.) Крейт `postgres` — синхронный клиент PostgreSQL. Служба читает **пароль из файла**, как это делается в настоящих развёртываниях. Правьте `src/main.rs` по шагам.

**Шаг 1.** В начале добавьте две строки `use`: `use std::error::Error;` перед остальными строками `std` и `use postgres::{Client, NoTls};` перед `use signal_hook…`.

**Шаг 2.** Замените `Config` и его `impl` на:

```rust
/// Settings of the service, read from environment variables.
#[derive(Debug, PartialEq)]
struct Config {
    bind: String,
    greeting: String,
    db_host: String,
    db_user: String,
    db_name: String,
    db_password_file: String,
}

impl Config {
    /// `get` looks up a variable by name (so tests need no real environment).
    fn from_vars(get: impl Fn(&str) -> Option<String>) -> Config {
        let var = |name: &str, default: &str| get(name).unwrap_or_else(|| default.to_string());
        Config {
            bind: var("BIND", "127.0.0.1:8080"),
            greeting: var("GREETING", "Hello"),
            db_host: var("DB_HOST", "localhost"),
            db_user: var("DB_USER", "app"),
            db_name: var("DB_NAME", "app"),
            db_password_file: var("DB_PASSWORD_FILE", "/run/secrets/db_password"),
        }
    }
}
```

**Шаг 3.** В `reason` добавьте ветку `503 => "Service Unavailable",` перед веткой `_`.

**Шаг 4.** После `reason` добавьте код работы с базой данных:

```rust
/// Opens a connection and makes sure the table exists. The password is read from a file.
fn connect(config: &Config) -> Result<Client, Box<dyn Error>> {
    let password = std::fs::read_to_string(&config.db_password_file)?;
    let mut client = postgres::Config::new()
        .host(&config.db_host)
        .user(&config.db_user)
        .password(password.trim())
        .dbname(&config.db_name)
        .connect_timeout(Duration::from_secs(3))
        .connect(NoTls)?;
    client.batch_execute(
        "CREATE TABLE IF NOT EXISTS hits (id BIGSERIAL PRIMARY KEY, at TIMESTAMPTZ NOT NULL DEFAULT now())",
    )?;
    Ok(client)
}

/// Stores one visit and returns how many visits there are in total.
fn record_hit(client: &mut Client) -> Result<i64, postgres::Error> {
    client.execute("INSERT INTO hits DEFAULT VALUES", &[])?;
    let row = client.query_one("SELECT count(*) FROM hits", &[])?;
    Ok(row.get(0))
}

/// Turns an error and its causes into one line, for example `db error: FATAL: ...`.
fn describe(error: &dyn Error) -> String {
    let mut text = error.to_string();
    let mut source = error.source();
    while let Some(cause) = source {
        text = format!("{text}: {cause}");
        source = cause.source();
    }
    text
}

/// A database connection that is opened when needed and opened again after a failure.
struct Database<'a> {
    config: &'a Config,
    client: Option<Client>,
}

impl Database<'_> {
    fn hit(&mut self) -> Result<i64, Box<dyn Error>> {
        // Try the open connection first; if it is broken, forget it and connect again.
        if let Some(client) = self.client.as_mut() {
            match record_hit(client) {
                Ok(total) => return Ok(total),
                Err(_) => self.client = None,
            }
        }
        let client = self.client.insert(connect(self.config)?);
        let result = record_hit(client);
        if result.is_err() {
            self.client = None;
        }
        Ok(result?)
    }
}
```

**Шаг 5.** Замените `handle` версией, которая знает `/hits`:

```rust
fn handle(mut stream: TcpStream, greeting: &str, database: &mut Database) -> std::io::Result<()> {
    stream.set_read_timeout(Some(Duration::from_secs(2)))?;
    let mut buffer = [0u8; 2048];
    let size = stream.read(&mut buffer)?;
    let request = String::from_utf8_lossy(&buffer[..size]);
    let request_line = request.lines().next().unwrap_or("");

    let (status, body) = if request_line.starts_with("GET /hits ") {
        match database.hit() {
            Ok(total) => (200, format!("hits: {total}\n")),
            Err(error) => {
                eprintln!("database error: {}", describe(error.as_ref()));
                (503, "database unavailable\n".to_string())
            }
        }
    } else {
        route(request_line, greeting)
    };
    println!("{request_line} -> {status}");
    write!(
        stream,
        "HTTP/1.1 {status} {}\r\nContent-Length: {}\r\nConnection: close\r\n\r\n{body}",
        reason(status),
        body.len()
    )
}
```

**Шаг 6.** В `main` непосредственно перед циклом `while` добавьте `let mut database = Database { config: &config, client: None };` (`cargo fmt` разобьёт это на четыре строки) и измените вызов на `.and_then(|()| handle(stream, &config.greeting, &mut database))`. В тест `config_has_defaults` добавьте `assert_eq!(config.db_host, "localhost");`.

Отформатируйте, проверьте и протестируйте:

```bash
cargo fmt && cargo clippy --all-targets && cargo test
```

```text
running 3 tests
test tests::config_reads_variables ... ok
test tests::config_has_defaults ... ok
test tests::routes_requests ... ok

test result: ok. 3 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

**Что происходит:**

1. Соединение открывается **не** при запуске, а при первом `/hits`. Если база данных не готова или пропадает, служба остаётся жива, на `/hits` отвечает `503 database unavailable`, на `/health` — `200`, и подключается снова при следующем запросе. Веб-служба должна переживать потерю зависимости.
2. `hit()` сначала пробует открытое соединение. Если оно не сработало (базу данных перезапустили), функция забывает это соединение, открывает новое и пробует ещё раз.
3. Пароль берётся из файла, названного в `DB_PASSWORD_FILE`, а не из переменной окружения. Compose положит туда секрет ([Урок 10.8](../10-security-fundamentals/08-secrets-management.md)).
4. `describe` проходит по цепочке причин ошибки (`Error::source`), потому что одна ошибка `postgres` печатает лишь `db error`.
5. Модульные тесты покрывают только то, что не требует базы данных. Настоящий тест — работающий стек (Упражнение 2).

`Dockerfile` из Урока 11.13 менять не нужно (`sed` уже переименовал в нём программу). Создайте файл с паролем. Это секрет, поэтому его нельзя коммитить в Git; об этом заботится `.gitignore`:

```bash
mkdir -p secrets
head -c 18 /dev/urandom | base64 > secrets/db_password.txt
chmod 600 secrets/db_password.txt
printf 'secrets/\n.env\ntarget\n' > .gitignore
```

## 3. Файл Compose

Создайте `compose.yaml` в `code/p11-counter`:

```yaml
name: course-counter

services:
  db:
    image: postgres:18
    environment:
      POSTGRES_USER: app
      POSTGRES_DB: app
      POSTGRES_PASSWORD_FILE: /run/secrets/db_password
    secrets:
      - db_password
    volumes:
      - pgdata:/var/lib/postgresql
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U app -d app"]
      interval: 5s
      timeout: 3s
      retries: 10
    restart: unless-stopped

  app:
    build: .
    image: course-counter:1
    environment:
      BIND: 0.0.0.0:8080
      DB_HOST: db
      DB_USER: app
      DB_NAME: app
    secrets:
      - db_password
    ports:
      - "127.0.0.1:${HOST_PORT:-18080}:8080"
    depends_on:
      db:
        condition: service_healthy
    restart: on-failure

volumes:
  pgdata:

secrets:
  db_password:
    file: ./secrets/db_password.txt
```

| Ключ | Значение |
|---|---|
| `name:` | Имя проекта. Все объекты получают этот префикс (`course-counter-db-1`, сеть `course-counter_default`, том `course-counter_pgdata`), поэтому убирать их легко. |
| `services:` | По одной записи на службу. |
| `image: postgres:18` | Использовать этот официальный образ. Зафиксированная основная версия, не `latest`. |
| `environment:` | Переменные окружения контейнера. Официальный образ базы данных читает имя пользователя и базы данных, а пароль — **из файла**, если имя заканчивается на `_FILE`. |
| `secrets:` (внутри службы) | Смонтировать секрет `db_password` в `/run/secrets/db_password` в этом контейнере. |
| `volumes: - pgdata:/var/lib/postgresql` | Смонтировать именованный том `pgdata` туда, где PostgreSQL 18 хранит данные. |
| `healthcheck:` | Команда проверки здоровья (`pg_isready` входит в образ) и как часто её выполнять. |
| `build: .` и `image:` | Собрать образ этой службы из `Dockerfile` текущей папки и назвать его `course-counter:1`. |
| `DB_HOST: db` | **Имя службы** — это имя хоста внутри сети проекта. |
| `ports: - "127.0.0.1:…:8080"` | Опубликовать службу на Mac только на петле. `${HOST_PORT:-18080}` значит «переменная `HOST_PORT` или 18080, если она не задана». |
| `depends_on: … service_healthy` | Запускать `app` только после того, как пройдёт проверка работоспособности `db`. |
| `restart:` | Политика (раздел 5). |
| верхнеуровневые `volumes:` и `secrets:` | Объявляют том и секрет: здесь секрет — это файл `secrets/db_password.txt`. |

У службы базы данных **нет** записи `ports:`. Снаружи проекта до PostgreSQL никто не достучится; только `app` может это сделать по сети проекта. Проверьте файл, прежде чем что-либо запускать:

```bash
docker compose config --quiet && echo "config ok"
```

```text
config ok
```

(`docker compose config` без `--quiet` печатает файл так, как его понимает Compose, с подставленными переменными. Это первое, что нужно выполнить, когда что-то выглядит неправильно.)

## 4. Запуск, осмотр, остановка

```bash
docker compose up -d --build --wait
docker compose ps
```

`up` создаёт сеть, том и контейнеры по порядку, `-d` работает в фоне, `--build` собирает образ `app`, а `--wait` возвращает управление, когда все службы здоровы. Последние строки вывода:

```text
 Volume course-counter_pgdata Created
 Network course-counter_default Created
 Container course-counter-db-1 Created
 Container course-counter-app-1 Created
 Container course-counter-db-1 Started
 Container course-counter-db-1 Healthy
 Container course-counter-app-1 Started
 Container course-counter-app-1 Healthy
```

```text
NAME                   IMAGE              COMMAND                  SERVICE   CREATED          STATUS                    PORTS
course-counter-app-1   course-counter:1   "/usr/local/bin/p11-…"   app       16 seconds ago   Up 10 seconds (healthy)   127.0.0.1:18080->8080/tcp
course-counter-db-1    postgres:18        "docker-entrypoint.s…"   db        16 seconds ago   Up 16 seconds (healthy)   5432/tcp
```

(Первая сборка скачивает `postgres:18` и компилирует крейты: около минуты.) `app` запустился **после** того, как `db` стала здоровой, а у `db` указано `5432/tcp` без стрелки: порт существует только внутри сети. Воспользуйтесь службой:

```bash
curl -s http://localhost:18080/hits
curl -s http://localhost:18080/hits
curl -s http://localhost:18080/
docker compose logs --no-log-prefix app
docker compose exec db psql -U app -d app -c 'SELECT count(*) FROM hits;'
```

```text
hits: 1
hits: 2
Hello from p11-counter
listening on 0.0.0.0:8080
GET /hits HTTP/1.1 -> 200
GET /hits HTTP/1.1 -> 200
GET / HTTP/1.1 -> 200
 count 
-------
     2
(1 row)
```

Служба насчитала два посещения, и таблица базы данных содержит те же две строки. `docker compose exec db …` выполняет команду в работающем контейнере `db` (`psql` — клиент PostgreSQL). Изнутри сети проекта имя службы — это адрес:

```bash
docker compose exec db getent hosts app
```

```text
172.31.0.3      app
```

| Команда | Что делает |
|---|---|
| `docker compose up -d [--build] [--wait]` | создать и запустить (пересобрать, дождаться здоровья) |
| `docker compose ps` | показать контейнеры проекта |
| `docker compose logs [-f] [служба]` | показать журналы всех служб или одной (`-f` следит за новыми строками) |
| `docker compose exec служба команда` | выполнить команду в работающей службе |
| `docker compose stop` / `start` / `restart [служба]` | остановить, запустить, перезапустить без удаления |
| `docker compose down` | остановить и удалить контейнеры и сеть, **тома оставить** |
| `docker compose down -v` | то же, но удалить и тома (данные пропадут) |

**Данные переживают `down`, но не `down -v`.** Выполните `docker compose down`, затем `docker compose up -d --wait` и `curl -s http://localhost:18080/hits`: ответ — `hits: 3`, потому что том `course-counter_pgdata` сохранился. После `docker compose down -v` и нового `up` первый ответ снова `hits: 1`.

> ⚠️ **Частая ошибка:** выполнять `docker compose down -v` по привычке в проекте, в томе которого лежат настоящие данные. Опция `-v` удаляет их без вопросов. В боевой среде сначала делайте резервную копию (Упражнение 3).

## 5. Когда что-то ломается

**База данных пропала.** Служба выживает:

```bash
docker compose stop db
curl -s -w "[%{http_code}]\n" http://localhost:18080/hits
docker compose logs --no-log-prefix app | tail -1
docker compose start db
sleep 6
curl -s -w "[%{http_code}]\n" http://localhost:18080/hits
```

```text
database unavailable
[503]
database error: error connecting to server: failed to lookup address information: Name or service not known
hits: 4
[200]
```

Пока `db` остановлена, `/hits` отвечает `503`, а журнал объясняет причину; `/health` по-прежнему отвечал бы `200`. Когда база данных вернулась, следующий запрос сам подключается заново, без перезапуска `app`.

**Служба падает.** Политика перезапуска решает, что делает Docker:

| `restart:` | Значение |
|---|---|
| `no` (по умолчанию) | никогда не перезапускать |
| `on-failure` | перезапускать, когда процесс завершился с ненулевым кодом |
| `always` | перезапускать при любом коде возврата, в том числе после перезапуска Docker |
| `unless-stopped` | как `always`, но не если вы сами остановили контейнер |

Чтобы увидеть это, добавьте намеренно плохой маршрут в начало `handle`, сразу после `let request_line = …;`. Это **только для обучения**; потом удалите его, потому что у службы никогда не должно быть кнопки, которая её убивает:

```rust
    if request_line.starts_with("GET /crash ") {
        eprintln!("crashing on purpose");
        std::process::exit(1);
    }
```

Пересоберите через `docker compose up -d --build --wait`, затем:

```bash
curl -sS -m 3 http://localhost:18080/crash
sleep 4
docker compose ps --format 'table {{.Service}}\t{{.Status}}'
docker inspect course-counter-app-1 --format 'restarts={{.RestartCount}}'
docker compose logs --no-log-prefix app | tail -2
```

```text
curl: (52) Empty reply from server
SERVICE   STATUS
app       Up 3 seconds (health: starting)
db        Up 20 seconds (healthy)
restarts=1
crashing on purpose
listening on 0.0.0.0:8080
```

Процесс завершился с кодом 1, Docker запустил его снова (`restarts=1`), и служба вернулась за секунды. При `on-failure` обычная остановка (`docker compose stop`, код 0 после `SIGTERM`) **не** перезапускается. И ещё: `docker kill` и `docker compose kill` считаются *ручной* остановкой, поэтому политика к ним **не** применяется; вот почему мы использовали настоящее падение.

> 💡 **Совет:** политика перезапуска — страховочная сетка, а не исправление. Прочитайте `docker compose logs`, чтобы выяснить, почему служба упала. Добавьте ограничение (`on-failure:5`), если цикл падений расходует ресурсы.

**Неверный или изменённый пароль.** `POSTGRES_PASSWORD_FILE` читается только тогда, когда PostgreSQL создаёт **новый** каталог данных. Если вы меняете `secrets/db_password.txt`, пока том уже существует, база данных сохраняет старый пароль:

```text
database unavailable
app-1  | database error: db error: FATAL: password authentication failed for user "app"
```

Либо верните старый пароль, либо смените пароль внутри базы командой `ALTER USER`, либо (только если данные одноразовые) выполните `docker compose down -v`, чтобы начать с нуля. Функция `describe` здесь окупилась: без неё в журнале было бы только `db error`.

> 🔒 **Безопасность:** секреты в Compose — это файлы, а не переменные окружения, потому что переменные видны в `docker inspect` и в списках процессов. Держите файл вне Git (`.gitignore`) и вне образа (он монтируется при запуске, а не копируется). В Docker Desktop для Mac пользователь контейнера может читать файл как есть. На сервере Linux файл сохраняет права хоста, поэтому файл с режимом `600`, принадлежащий вашему пользователю, может оказаться нечитаемым для пользователя контейнера; проверьте через `docker compose exec` и задайте файлу правильного владельца или группу, а не делайте его читаемым для всех. Настоящие боевые системы используют хранилище секретов платформы ([Урок 10.8](../10-security-fundamentals/08-secrets-management.md)).

Уберите за собой, когда закончите: `docker compose down -v` и `docker image rm course-counter:1`.

## Частые ошибки

1. **Неправильный отступ или табуляция в YAML.** Compose сообщает `yaml: line 12: …` или молча игнорирует ключ. *Исправление:* только пробелы, по два на уровень; `docker compose config` показывает, что было понято.
2. **`depends_on` без проверки работоспособности** ждёт лишь *запуска* контейнера, а не готовности базы данных. *Исправление:* `condition: service_healthy` с `healthcheck`.
3. **`localhost` как хост базы данных.** Внутри `app` `localhost` — это контейнер app. *Исправление:* используйте имя службы: `DB_HOST: db`.
4. **Публикация порта базы данных** (`5432:5432`) на всех интерфейсах. *Исправление:* публикуйте только то, что нужно пользователям; `db` не нужны `ports:`.
5. **Пароль в `compose.yaml` или в Git.** *Исправление:* секретный файл, `.gitignore`; смените пароль, который когда-либо попадал в коммит.
6. **`docker compose down -v`, который удаляет данные.** *Исправление:* используйте `down`; перед `-v` делайте резервную копию.
7. **Ожидание, что `restart:` сработает после `docker kill`.** *Исправление:* проверяйте настоящим падением.
8. **Смена секрета на существующем томе** (раздел 5). *Исправление:* знайте, что пароль задаётся один раз.

## Попробуйте сами

1. Соберите стек, обратитесь к `/hits` три раза и посмотрите на таблицу через `psql`.
2. Выполните `docker compose down`, запустите снова и проверьте, что счётчик продолжается. Затем используйте `down -v` и посмотрите, как он начинается с 1.
3. Остановите только `db`, обратитесь к `/hits`, запустите `db` и обратитесь снова.
4. Запустите стек с другим портом хоста, не редактируя файл: `HOST_PORT=18090 docker compose up -d --wait`.
5. Удалите всё: `docker compose down -v` и образ.

## Упражнения

### Упражнение 1 — Измените настройку, не редактируя файл (легко)

Запустите стек на порту хоста 18090 через переменную, проверьте через `docker compose ps`, какой порт опубликован, и обратитесь к `/hits` на новом порту.

<details>
<summary>Подсказка</summary>

В файле есть `${HOST_PORT:-18080}`. Задайте переменную перед командой (`HOST_PORT=… docker compose …`). Контейнер пересоздаётся, потому что настройка порта изменилась.

</details>

<details>
<summary>Ответ</summary>

```bash
HOST_PORT=18090 docker compose up -d --wait
docker compose ps --format 'table {{.Service}}\t{{.Ports}}'
curl -s http://localhost:18090/hits
```

```text
SERVICE   PORTS
app       127.0.0.1:18090->8080/tcp
db        5432/tcp
hits: 1
```

(Счётчик показывает `1` после свежего `down -v`; иначе он продолжает с сохранённого значения.)

Почему это работает: Compose заменяет `${HOST_PORT:-18080}` переменной оболочки или значением по умолчанию после `:-`. Поэтому один и тот же файл работает на машинах, где порт 18080 занят.

</details>

### Упражнение 2 — Тест всего стека (средне)

Напишите `test-compose.sh`, который запускает стек, проверяет, что первые два посещения дают `hits: 1` и `hits: 2`, перезапускает базу данных и проверяет, что данные сохранились (`hits: 3`), и всегда удаляет стек через `trap`.

<details>
<summary>Подсказка</summary>

`docker compose up -d --build --wait` возвращается, когда службы здоровы. `trap '…down -v…' EXIT` убирает за собой на любом пути выхода. После `docker compose restart db` подождите несколько секунд, пока база данных начнёт принимать соединения.

</details>

<details>
<summary>Ответ</summary>

```bash
#!/usr/bin/env bash
# test-compose.sh - starts the stack, checks the counter, and always cleans up.
set -u
cd "$(dirname "$0")" || exit 1

failures=0
check() {   # check "description" expected actual
    if [[ $2 == "$3" ]]; then
        echo "ok   - $1"
    else
        echo "FAIL - $1 (expected [$2], got [$3])"
        failures=$((failures + 1))
    fi
}

trap 'docker compose down -v > /dev/null 2>&1' EXIT

docker compose up -d --build --wait > /dev/null 2>&1
check "stack starts and is healthy" 0 "$?"

first=$(curl -s http://localhost:18080/hits)
second=$(curl -s http://localhost:18080/hits)
check "first visit" "hits: 1" "$first"
check "second visit" "hits: 2" "$second"

docker compose restart db > /dev/null 2>&1
sleep 5
check "data survives a database restart" "hits: 3" "$(curl -s http://localhost:18080/hits)"

(( failures == 0 ))
```

```text
ok   - stack starts and is healthy
ok   - first visit
ok   - second visit
ok   - data survives a database restart
```

Почему это работает: trap удаляет контейнеры, сеть и том, даже если проверка не прошла, поэтому следующий запуск начинается с `hits: 1`. Тест опирается на повторную попытку в `Database::hit`: первый запрос после перезапуска находит старое соединение разорванным, открывает новое и всё равно отвечает правильно.

</details>

### Упражнение 3 — Резервная копия и восстановление базы данных (сложно)

При работающем стеке и нескольких записанных посещениях запишите содержимое базы данных в файл `backup.sql` с помощью `pg_dump` (запуск внутри контейнера `db`, вывод на ваш Mac). Затем удалите всё через `down -v`, запустите снова, восстановите файл через `psql` и покажите, что `/hits` продолжает со старого числа.

<details>
<summary>Подсказка</summary>

`docker compose exec -T db pg_dump -U app app > backup.sql` (`-T` отключает терминал, он нужен для перенаправления). Восстановление: `docker compose exec -T db psql -U app -d app < backup.sql`. Новая база данных пуста, а приложение создаёт таблицу только при первом `/hits`, поэтому восстанавливайте **до** первого посещения или сначала удалите таблицу.

</details>

<details>
<summary>Ответ</summary>

```bash
curl -s http://localhost:18080/hits               # hits: 4, for example
docker compose exec -T db pg_dump -U app app > backup.sql
docker compose down -v
docker compose up -d --wait
docker compose exec -T db psql -q -U app -d app < backup.sql > /dev/null
docker compose exec db psql -U app -d app -t -c 'SELECT count(*) FROM hits;'
curl -s http://localhost:18080/hits
```

```text
     4
hits: 5
```

Почему это работает: `pg_dump` пишет операторы SQL, которые воссоздают таблицу и её строки (`CREATE TABLE`, `COPY` и значение последовательности). Восстановление выполняет их в новой пустой базе данных, поэтому счётчик продолжается. Не кладите `backup.sql` в Git, если данные приватные. Резервная копия, которую вы ни разу не восстанавливали, — не резервная копия: это упражнение и есть проверка.

</details>

## Вопросы для самопроверки

1. В чём преимущество файла Compose по сравнению с несколькими командами `docker run`?

<details>
<summary>Ответ</summary>

Он описывает всё приложение одним файлом, который можно хранить в Git, запускает и останавливает всё одной командой, создаёт сеть и тома и каждый раз применяет одни и те же настройки.

</details>

2. Как `app` находит базу данных и почему не использует `localhost`?

<details>
<summary>Ответ</summary>

Compose создаёт сеть проекта, где имя каждой службы — имя хоста, поэтому работает `DB_HOST=db`. `localhost` внутри контейнера `app` — это сам контейнер `app`, а не база данных.

</details>

3. Что добавляет `depends_on` с `condition: service_healthy` по сравнению с простым `depends_on`?

<details>
<summary>Ответ</summary>

Простой `depends_on` лишь упорядочивает запуск контейнеров. С `service_healthy` Compose ждёт, пока пройдёт проверка работоспособности другой службы, так что база данных принимает соединения до запуска `app`.

</details>

4. В чём разница между `docker compose down` и `docker compose down -v`?

<details>
<summary>Ответ</summary>

`down` удаляет контейнеры и сеть, но сохраняет именованные тома (данные переживают). `down -v` удаляет и тома, поэтому данные теряются.

</details>

5. Почему пароль передаётся файлом, а не переменной окружения?

<details>
<summary>Ответ</summary>

Переменные окружения видны через `docker inspect` и в информации о процессах и легко попадают в журналы. Файл в `/run/secrets/` виден только внутри контейнера, а образ и репозиторий Git его никогда не содержат.

</details>

6. Контейнер убит командой `docker kill` и имеет `restart: on-failure`. Почему он не перезапускается?

<details>
<summary>Ответ</summary>

Docker считает `docker kill` (и `stop`) ручной остановкой и не применяет политику перезапуска. Настоящее падение, когда процесс сам завершается с ненулевым кодом, перезапускается.

</details>

## Итоги

- Compose описывает службы, сети, тома и секреты в `compose.yaml` и запускает их через `docker compose up -d`; префикс `name:` держит все объекты проекта вместе и облегчает их удаление.
- Службы обращаются друг к другу по имени службы в сети проекта. Публикуйте только те порты, которые нужны пользователям, и привязывайте их к `127.0.0.1`, если они нужны только вам.
- `depends_on` с `condition: service_healthy` и `healthcheck` ждёт готовности, а не просто запуска.
- Данные живут в именованных томах: `down` их сохраняет, `down -v` удаляет. Перед удалением делайте резервную копию и проверяйте восстановление.
- Пароли хранятся в секретных файлах, и программа читает их по пути (`*_FILE`); их не коммитят и не вшивают в образы.
- Политика перезапуска (`on-failure`, `unless-stopped`) перезапускает упавшие контейнеры, но не остановленные вручную. Хорошая служба переживает отсутствие зависимости и подключается заново.

## Источники и что почитать

- [Docker Compose overview](https://docs.docker.com/compose/) — что такое Compose и как им пользоваться.
- [Compose file reference](https://docs.docker.com/reference/compose-file/) — каждый ключ `compose.yaml`.
- [Control startup and shutdown order in Compose](https://docs.docker.com/compose/how-tos/startup-order/) — условия `depends_on` и проверки работоспособности.
- [Use secrets in Compose](https://docs.docker.com/compose/how-tos/use-secrets/) — как монтируются секреты.
- [Docker Official Image: postgres](https://hub.docker.com/_/postgres) — переменные окружения вроде `POSTGRES_PASSWORD_FILE` и место хранения данных.
- [Start containers automatically](https://docs.docker.com/engine/containers/start-containers-automatically/) — политики перезапуска подробно.
- [`pg_dump` и `pg_isready`](https://www.postgresql.org/docs/18/app-pgdump.html) — страницы руководства PostgreSQL 18 об инструменте резервного копирования (страница `pg_isready` — [здесь](https://www.postgresql.org/docs/18/app-pg-isready.html)).

---

[🏠 Начало](../START-HERE.md) · [← 11.13 Dockerfile для программ на Rust](13-dockerfiles-for-rust.md) · [📋 Этап 11](README.md) · [11.15 Проект: развёртывание сервиса на Rust →](15-project-deploy-a-rust-service.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
