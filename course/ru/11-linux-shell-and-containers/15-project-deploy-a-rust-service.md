<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 11.14 Docker Compose](14-docker-compose.md) · [📋 Этап 11](README.md) · [11.16 Контрольная точка: стадия 2 пройдена →](16-checkpoint-stage-2-complete.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/11-linux-shell-and-containers/15-project-deploy-a-rust-service.md) <!-- nav -->

# Урок 11.15 — Проект: развёртывание сервиса на Rust

> **Этап 11 — Linux, командная оболочка и контейнеры** · Урок 15 из 16 · ⏱ около 6 часов (3 занятия)

## Что вы узнаете

- Как ввести в эксплуатацию одну небольшую службу на Rust двумя способами: как **службу systemd в ВМ (виртуальной машине) `dev`** и как **образ Docker, запускаемый через Compose**.
- Как написать скрипт развёртывания, который собирает, копирует, перезапускает, проверяет работоспособность и откатывается, если проверка не прошла.
- Как сравнить эти два способа: журналы, перезапуск после падения, корректная остановка, от какого пользователя работает программа, кто может до неё достучаться.
- Как проверить развёртывание одним приёмочным скриптом, а не на глаз.

## Перед началом

- Вы прошли [Урок 11.14 — Docker Compose](14-docker-compose.md) и всё, что идёт перед ним на этом этапе. В проекте используются ВМ `dev` ([Урок 11.1](01-linux-and-a-virtual-machine.md)) с установленным в ней Rust ([Урок 11.5](05-processes-and-signals.md)), службы ([Урок 11.6](06-services-with-systemd.md)), скрипты на bash ([Уроки 11.8](08-bash-scripting-basics.md) и [11.9](09-robust-bash-scripts.md)), SSH (Secure Shell, защищённая оболочка) с псевдонимом `dev` ([Урок 11.10](10-ssh-and-remote-servers.md)) и Docker ([Уроки 11.12–11.14](12-containers-and-docker-basics.md)).
- У вас есть служба `code/p11-hello` из [Урока 11.13](13-dockerfiles-for-rust.md). Вместо неё можно использовать ваш собственный веб-сервер из [Урока 06.11](../06-rust-advanced/11-project-multithreaded-web-server.md), если он соответствует контракту службы ниже.
- ВМ `dev` запущена, `ssh dev hostname` работает, и в ней установлен `rsync` (`ssh dev 'sudo apt-get install -y rsync build-essential'`).
- Docker Desktop запущен. Отведите несколько часов на саму практику. Полного решения здесь **нет**; вы пишете всё сами, а команды «Готово, когда» подскажут, работает ли это.

## Новые слова

| Слово | Значение |
|---|---|
| **развёртывание** (deploy) | Размещение программы на машине так, чтобы она работала для пользователей: собрать или скопировать её, настроить, запустить и проверить. |
| **цель развёртывания** (deployment target) | Место, где работает программа: здесь ВМ `dev` и контейнер Docker. |
| **релиз** (release) | Одна версия программы, которую вы развёртываете. |
| **откат** (rollback) | Возврат к предыдущему рабочему релизу после неудачного развёртывания. |
| **дымовой тест** (smoke test) | Очень быстрая проверка того, что самое важное работает (например, `GET /health` отвечает `200`), сразу после развёртывания. |
| **идемпотентный** (idempotent) | О действии, которое даёт тот же результат при повторении. Двойной запуск скрипта развёртывания должен быть безопасным. |
| **файл окружения** (environment file) | Файл со строками `ИМЯ=значение`, который systemd читает (`EnvironmentFile=`), чтобы задать окружение службы. |
| **ротация журналов** (log rotation) | Ограничение того, сколько журнала хранится: когда файл журнала достигает размера, самая старая часть отбрасывается. |
| **приёмочный скрипт** (acceptance script) | Скрипт, который командами проверяет, что система делает то, что сказано в контракте. |

## 1. Два способа запустить одну и ту же программу

Вы знаете два способа запустить службу. Прежде чем строить оба, посмотрите, что даёт каждый:

| Вопрос | systemd в ВМ | Docker с Compose |
|---|---|---|
| Что развёртывается? | бинарный файл в `/usr/local/bin`, собранный в ВМ | образ с бинарным файлом и его базовой системой |
| Кто запускает и следит? | `systemd` (юнит-файл) | Docker Engine (политика перезапуска) |
| Где журналы? | журнал: `journalctl -u p11-hello` | `docker compose logs`, хранятся драйвером журналов |
| Как она получает настройки? | `/etc/p11-hello.env` | `environment:` в `compose.yaml` |
| От какого пользователя работает? | системный пользователь `p11hello` | пользователь `app` (uid 10001) в образе |
| Перезапуск после падения? | `Restart=on-failure` | `restart: unless-stopped` |
| Как её обновляют? | `deploy.sh`: синхронизация, сборка, установка, перезапуск | собрать новый образ, `docker compose up -d` |
| Что её изолирует? | параметры ужесточения службы | контейнер: собственная файловая система, `--read-only`, нет capabilities |

Ни один не «лучше». Серверы с несколькими долгоживущими программами используют systemd; команды, выпускающие много служб, используют контейнеры. Системный инженер знает оба и умеет проверить, что любой из них здоров. Проект заставляет вас написать оба и сравнить их.

**Git в этом уроке.** Делайте **по одному коммиту на веху** (маленький шаг из Урока 01.3): `git add code/p11-deploy` и `git commit -m "p11-deploy: M3 deploy script"`. Сначала создайте ветку: `git switch -c lesson/11-15-deploy`.

## Проект

### Что вы создадите

Папку `code/p11-deploy` со следующим:

```text
code/p11-deploy/
  Cargo.toml  Cargo.lock  src/main.rs   <- the service (a copy of p11-hello)
  Dockerfile  .dockerignore             <- deployment 2: the image
  compose.yaml                          <- deployment 2: the stack
  deploy/p11-hello.service              <- deployment 1: the systemd unit
  deploy/deploy.sh                      <- deployment 1: the deploy script
  deploy/check.sh                       <- the acceptance script for both
  README.md                             <- how to run, and the comparison table
```

### Контракт

**Служба** (ваша копия `p11-hello`) должна:

- читать `BIND` (по умолчанию `127.0.0.1:8080`) и `GREETING` из окружения;
- отвечать на `GET /` строкой `<GREETING> from p11-hello`, а на `GET /health` — `200` и `ok`;
- писать по одной строке журнала на запрос в стандартный вывод, никогда не в файл;
- корректно останавливаться по `SIGTERM` (записать `got a stop signal, bye`, код возврата `0`);
- проходить `cargo test`, `cargo clippy` и `cargo fmt --check`.

**Развёртывание 1: ВМ `dev`.**

- Бинарный файл — `/usr/local/bin/p11-hello`, владелец root. Служба работает от системного пользователя `p11hello`, без оболочки входа и без домашней папки.
- Настройки лежат в `/etc/p11-hello.env` (`BIND=127.0.0.1:8081` и `GREETING=Hello (VM)`), доступном для чтения root и группе `p11hello`, но не всем.
- Юнит `p11-hello.service` перезапускает службу после сбоя, включён при загрузке, пишет только в журнал и имеет не менее трёх параметров ужесточения (Урок 11.6).
- Lima перенаправляет порты петли ВМ на ваш Mac, поэтому `http://localhost:8081/` на Mac достигает её (см. документацию Lima про [перенаправление портов](https://lima-vm.io/docs/config/port/)).

**`deploy/deploy.sh [-n] [HOST]`** запускается на **Mac**; `HOST` — псевдоним SSH (по умолчанию `dev`).

| Поведение | Правило |
|---|---|
| шаги | печатать по одной строке на шаг: `[1/5] sync …`, `[2/5] build …`, `[3/5] install …`, `[4/5] restart …`, `[5/5] health check` |
| синхронизация | `rsync` исходников в `~/deploy/p11-hello` на `HOST` без `target/`, `.git` и `deploy/` |
| сборка | `cargo build --release` **на `HOST`** |
| установка | сохранить старый бинарный файл как `/usr/local/bin/p11-hello.prev`, затем установить новый через `sudo install -m 755` |
| перезапуск | `sudo systemctl restart p11-hello` |
| проверка работоспособности | до 10 попыток с интервалом в секунду: `curl -fsS -m 2 http://127.0.0.1:8081/health` **на `HOST`**; при успехе печатает `health check ok …` и завершается с кодом `0` |
| откат | если ни одна попытка не удалась: напечатать `health check failed, rolling back` в стандартный поток ошибок, вернуть `.prev`, перезапустить, завершиться с кодом `1` |
| `-n` | пробный запуск: показать, что отправил бы `rsync`, больше ничего не менять, код `0` |
| использование | `-h` печатает справку (код `0`); неизвестная опция печатает сообщение в стандартный поток ошибок (код `2`) |
| качество | строгий режим (`set -euo pipefail`), переменные в кавычках, ShellCheck без замечаний |

**Развёртывание 2: Docker с Compose.**

- `docker build` создаёт образ менее 150 МБ (мегабайт), собранный по этапам, работающий от пользователя без прав root, с `HEALTHCHECK`.
- `compose.yaml` (имя проекта `course-deploy`) публикует службу как `127.0.0.1:8082`, задаёт `BIND=0.0.0.0:8080` и `GREETING` (по умолчанию `Hello (Docker)`), перезапускает контейнер, если вы его не останавливали, делает файловую систему доступной только для чтения, отбрасывает все capabilities, запрещает новые привилегии и ограничивает журнал тремя файлами по 1 МБ.

**`deploy/check.sh [HOST]`** проверяет оба развёртывания, печатает `ok   - …` или `FAIL - …` для каждой проверки и завершается кодом `0`, только если все проверки прошли.

### Пошаговый план

#### M0 — Подготовьте проект

**Цель:** новая папка с рабочей, протестированной копией службы.

Скопируйте `code/p11-hello` в `code/p11-deploy` (без `target/`), создайте ветку, запустите проверки качества и сделайте коммит. Сохраните имя программы `p11-hello` в `Cargo.toml`.

<details>
<summary>Подсказка</summary>

`cp -r code/p11-hello code/p11-deploy && rm -rf code/p11-deploy/target code/p11-deploy/Dockerfile.*`. `Dockerfile` из Урока 11.13 можно оставить, а остальные удалить.

</details>

**Готово, когда:**

```bash
cd ~/projects/rust/code/p11-deploy
cargo fmt --check && cargo clippy --all-targets && cargo test
git log --oneline | head -2
```

показывает `test result: ok. 3 passed` и коммит для M0.

#### M1 — Запустите службу вручную в ВМ

**Цель:** доказать, что программа собирается и работает на Linux, до всякой автоматизации. Каждый автоматический шаг в M3 — это то, что вы здесь сделали вручную.

Скопируйте исходники в ВМ через `rsync` (исключив `target`), соберите их там, запустите с `BIND=127.0.0.1:8081` в фоне, обратитесь к ней и остановите по `SIGTERM`.

<details>
<summary>Подсказка</summary>

`rsync -a --exclude target ./ dev:~/manual/`, затем `ssh dev 'cd ~/manual && PATH=$HOME/.cargo/bin:$PATH cargo build --release'`. Неинтерактивная оболочка `ssh` не загружает `~/.cargo/env`, поэтому `cargo` не в `PATH`, пока вы его не добавите. Запускайте программу через `BIND=127.0.0.1:8081 ./target/release/p11-hello &` внутри той же команды `ssh`, а останавливайте через `pkill p11-hello`.

</details>

**Готово, когда:** одна команда `ssh dev '…'`, которая запускает программу, выполняет `curl -s http://127.0.0.1:8081/` и `curl -s http://127.0.0.1:8081/health` и останавливает её, печатает:

```text
Hello from p11-hello
ok
```

а журнал программы (запишите его в файл и выполните `cat`) заканчивается строкой `got a stop signal, bye`. После этого удалите `~/manual` в ВМ.

#### M2 — Запустите её как службу systemd

**Цель:** программа работает от собственного пользователя, запускается при загрузке и возвращается после падения.

Сделайте это один раз вручную: создайте пользователя, установите бинарный файл из сборки M1, напишите `/etc/p11-hello.env`, напишите `deploy/p11-hello.service` в проекте, скопируйте его в `/etc/systemd/system/`, перезагрузите конфигурацию, включите и запустите.

<details>
<summary>Подсказка</summary>

Здесь применимы команды Урока 11.6: `useradd --system --no-create-home --shell /usr/sbin/nologin`, `install -m 755`, `systemd-analyze verify`, `daemon-reload`, `enable --now`. Используйте `EnvironmentFile=/etc/p11-hello.env` и задайте файлу режим `640`, владельца `root:p11hello`. В юните нужны `User=`, `Group=`, `ExecStart=`, `Restart=on-failure`, `RestartSec=2` и параметры вроде `NoNewPrivileges=yes`, `ProtectSystem=strict`, `ProtectHome=yes`, `PrivateTmp=yes`. Службе, слушающей порт, не нужен доступ на запись ни к чему. Проверяйте юнит через `systemd-analyze verify` **после** установки бинарного файла, иначе она пожалуется, что программы нет.

</details>

**Готово, когда:**

```bash
ssh dev 'systemctl is-active p11-hello; systemctl is-enabled p11-hello'
ssh dev 'ps -o user=,comm= -C p11-hello'
ssh dev 'curl -s http://127.0.0.1:8081/'
ssh dev 'ls -l /etc/p11-hello.env'
```

```text
active
enabled
p11hello p11-hello
Hello (VM) from p11-hello
-rw-r----- 1 root p11hello 40 Nov 18 08:13 /etc/p11-hello.env
```

(Размер и дата отличаются.) Затем докажите политику перезапуска и корректную остановку:

```bash
ssh dev 'systemctl show -p MainPID --value p11-hello'
ssh dev 'sudo systemctl kill -s SIGKILL p11-hello; sleep 3.5; systemctl show -p MainPID --value p11-hello; systemctl is-active p11-hello'
ssh dev 'sudo systemctl stop p11-hello; sudo journalctl -u p11-hello -n 3 -o cat; sudo systemctl start p11-hello'
```

Первая команда печатает номер процесса, например `676`. Вторая печатает **другой** номер и `active`: systemd перезапустил убитый вами процесс. Третья показывает в журнале `got a stop signal, bye` и `Stopped p11-hello.service - …`: чистая остановка, а не сбой.

#### M3 — Скрипт развёртывания

**Цель:** одна команда обновляет ВМ, а сломанный релиз не остаётся.

Напишите `deploy/deploy.sh` по контракту. Разрабатывайте в таком порядке: опции и справка; пробный запуск `-n`; синхронизация и сборка; установка; перезапуск; цикл проверки работоспособности; откат. Запускайте `shellcheck` почаще (Урок 11.9).

<details>
<summary>Подсказка 1 — структура</summary>

```text
getopts for -n and -h  ->  host=${1:-dev}
src_dir=$(cd "$(dirname "$0")/.." && pwd)        # the project folder, wherever you run the script from
step() { printf '[%s] %s\n' "$1" "$2"; }
rsync flags in an array, plus -n -v when dry-running
ssh "$host" '…' for build, install, restart
for attempt in 1 … 10: if answer=$(ssh "$host" "curl …"); then success
rollback: only reached when the loop ended without success
```

</details>

<details>
<summary>Подсказка 2 — ловушки</summary>

`rsync dir/ host:path/` (со слэшами) копирует **содержимое**; `rsync` не создаёт недостающие родительские папки, поэтому сначала выполните `ssh "$host" mkdir -p …` (для `-n` пропустите это, если хотите пробный запуск, который вообще ничего не меняет). Старый бинарный файл нужно скопировать в `.prev` **до** установки нового. `ssh` выполняет команду в домашней папке пользователя, поэтому относительный удалённый путь вроде `deploy/p11-hello` подходит. Переменные внутри команды `ssh "…"` в двойных кавычках раскрываются **на Mac** (ShellCheck говорит SC2029; здесь это именно то, что нужно, поэтому заглушите его для файла комментарием с причиной). Для `$` удалённой стороны нужна обратная косая черта: `\$HOME`. При `set -e` неудавшийся `ssh` в проверке работоспособности остановил бы скрипт, поэтому поместите его в `if` или в условие `answer=$(…)`. Откат должен работать и тогда, когда `.prev` нет (первое развёртывание).

</details>

**Готово, когда:**

```bash
deploy/deploy.sh -n dev; echo "exit=$?"
```

печатает строку `[1/5]`, список файлов, которые отправил бы `rsync` (`Cargo.toml`, `src/main.rs`, … но **не** `target/` и не `deploy/`), заканчивается строкой о том, что ничего не собиралось и не менялось, и `exit=0`. У службы тот же `MainPID` до и после.

```bash
deploy/deploy.sh dev; echo "exit=$?"
```

```text
[1/5] sync source to dev:deploy/p11-hello
[2/5] build on dev
[3/5] install the binary (the old one is kept as p11-hello.prev)
[4/5] restart the service
[5/5] health check
health check ok after 1 attempt(s): ok
exit=0
```

(Ваши формулировки могут отличаться, но пять пронумерованных строк и `health check ok` должны быть.) Запустите его **второй** раз: он должен снова завершиться успешно, с новым `MainPID`. Затем намеренно сломайте релиз: заставьте `/health` отвечать статусом `500`, запустите скрипт и потом восстановите исходники:

```text
[5/5] health check
deploy.sh: health check failed, rolling back
exit=1
```

После этого `ssh dev 'curl -s -w "[%{http_code}]\n" http://127.0.0.1:8081/health'` по-прежнему печатает `ok` и `[200]`: старый релиз снова работает. Выполните также `docker run --rm -v "$PWD/deploy":/mnt koalaman/shellcheck:stable deploy.sh`: вывода нет.

#### M4 — Образ Docker

**Цель:** маленький образ без прав root, который сам проверяет своё здоровье.

Используйте многоэтапный `Dockerfile` из Урока 11.13 (`debian:trixie-slim` или distroless, если хотите), со слоем зависимостей и `.dockerignore`.

<details>
<summary>Подсказка</summary>

`COPY Cargo.toml Cargo.lock ./`, сборка с фиктивным `main.rs`, `COPY src ./src`, `touch src/main.rs`, затем этап времени выполнения с `useradd`, `USER`, `ENV BIND=0.0.0.0:8080`, `HEALTHCHECK … ["/usr/local/bin/p11-hello", "--check"]` и `CMD` в **exec-форме**. Опция `--check` в `p11-hello` уже есть; если вы используете свой сервер, дайте ему способ проверять себя или установите инструмент для проверки работоспособности.

</details>

**Готово, когда:**

```bash
docker build -t course-deploy:1 .
docker image ls course-deploy
docker run -d --name course-deploy-test -p 127.0.0.1:8082:8080 course-deploy:1
sleep 13; docker ps --filter name=course-deploy-test --format '{{.Status}}'
docker inspect course-deploy-test --format 'user={{.Config.User}}'
curl -s http://localhost:8082/
SECONDS=0; docker stop course-deploy-test; echo "stop took ${SECONDS}s"
docker inspect course-deploy-test --format 'exit={{.State.ExitCode}}'
docker rm course-deploy-test
```

```text
IMAGE             ID             DISK USAGE   CONTENT SIZE   EXTRA
course-deploy:1   c14d811a7bf7        141MB         30.4MB
Up 13 seconds (healthy)
user=app
Hello from p11-hello
stop took 0s
exit=0
```

(Строка `IMAGE`/`ID` — настоящий вывод `docker image ls`; идентификатор будет другим. Размер должен быть меньше 150 МБ, статус — `healthy`, пользователь — не пустой и не `root`, остановка — менее 3 секунд с кодом `0`. Заметьте, что служба отвечает `Hello from p11-hello`, потому что без `GREETING` по умолчанию используется `Hello`.)

#### M5 — Compose, перезапуск и журналы

**Цель:** второе развёртывание в виде файла, с политикой перезапуска, ужесточением и ограниченными журналами.

Напишите `compose.yaml` по контракту и поднимите его. Чтобы проверить политику перезапуска, нужно **настоящее падение**, потому что `docker kill` — ручная остановка. Добавьте маршрут только для обучения из Урока 11.14 во **временную** копию службы или в рабочее дерево и удалите его перед коммитом.

<details>
<summary>Подсказка</summary>

Нужные ключи: `name: course-deploy`, `services: app: build: .`, `image: course-deploy:1`, `environment:` (используйте `${GREETING:-Hello (Docker)}`), `ports: - "127.0.0.1:8082:8080"`, `restart: unless-stopped`, `read_only: true`, `cap_drop: [ALL]`, `security_opt: ["no-new-privileges:true"]` и `logging: driver: json-file` с `options: max-size: "1m"` и `max-file: "3"`. Сначала проверьте файл через `docker compose config --quiet`. Маршрут падения: `if request_line.starts_with("GET /crash ") { std::process::exit(1); }` перед маршрутизацией.

</details>

**Готово, когда:**

```bash
docker compose up -d --build --wait
curl -s http://localhost:8082/
docker compose ps --format 'table {{.Service}}\t{{.Status}}\t{{.Ports}}'
docker inspect course-deploy-app-1 --format '{{.HostConfig.LogConfig}} restart={{.HostConfig.RestartPolicy.Name}} ro={{.HostConfig.ReadonlyRootfs}} user={{.Config.User}}'
```

```text
Hello (Docker) from p11-hello
SERVICE   STATUS                    PORTS
app       Up 10 seconds (healthy)   127.0.0.1:8082->8080/tcp
{json-file map[max-file:3 max-size:1m]} restart=unless-stopped ro=true user=app
```

Затем, с временным маршрутом падения в образе:

```bash
curl -sS -m 3 http://localhost:8082/crash
sleep 3
docker compose ps --format 'table {{.Service}}\t{{.Status}}'
docker inspect course-deploy-app-1 --format 'restarts={{.RestartCount}}'
docker compose logs --no-log-prefix app | tail -3
SECONDS=0; docker compose stop app; echo "stop took ${SECONDS}s"
docker compose logs --no-log-prefix app | tail -1
```

```text
curl: (52) Empty reply from server
SERVICE   STATUS
app       Up 2 seconds (health: starting)
restarts=1
GET / HTTP/1.1 -> 200
crashing on purpose
listening on 0.0.0.0:8080
stop took 0s
got a stop signal, bye
```

Удалите маршрут падения, пересоберите и сделайте коммит.

#### M6 — Приёмочный скрипт и сравнение

**Цель:** один скрипт, который проверяет оба развёртывания, и README, который их объясняет.

Напишите `deploy/check.sh`. Он запускается на Mac, спрашивает ВМ через `ssh` и Docker локально. Проверьте как минимум: юнит `active` и `enabled`; `/health` отвечает `200` в ВМ; процесс работает от `p11hello`; контейнер `healthy`; `/health` отвечает `200` на `localhost:8082`; пользователь контейнера — не root; файловая система контейнера доступна только для чтения. Затем напишите `README.md` с командами запуска каждого развёртывания и таблицей сравнения из раздела 1, заполненной тем, **что вы наблюдали** (например, как долго длился перезапуск в каждом случае).

<details>
<summary>Подсказка</summary>

Используйте функцию `check` из тестов `backup.sh` (Урок 11.9). Полезные кирпичики: `ssh "$host" systemctl is-active p11-hello`, `ssh "$host" curl -s -o /dev/null -w '%{http_code}' http://127.0.0.1:8081/health`, `ssh "$host" ps -o user= -C p11-hello | tr -d ' '`, `docker inspect ИМЯ --format '{{.State.Health.Status}}'`, `docker inspect ИМЯ --format '{{.Config.User}}'`, `docker inspect ИМЯ --format '{{.HostConfig.ReadonlyRootfs}}'`. Заметьте, что `docker top` требует в формате столбец `pid`, поэтому один `-eo user` не работает.

</details>

**Готово, когда:** при работающих обоих развёртываниях `deploy/check.sh dev; echo "exit=$?"` печатает (четыре строки части для ВМ и часть для Docker в вашем порядке и ваших формулировках):

```text
ok   - VM: service is active
ok   - VM: service is enabled
ok   - VM: /health answers 200
ok   - VM: runs as p11hello
ok   - Docker: container is healthy
ok   - Docker: /health answers 200
ok   - Docker: runs as user app
ok   - Docker: root file system is read-only
exit=0
```

После `sudo systemctl stop p11-hello` в ВМ (или `docker compose stop`) соответствующие строки становятся `FAIL`, а код возврата — `1`. Потом запустите их снова. ShellCheck должен быть чист и на `check.sh`.

### Дополнительные цели

- **Версия в ответе.** Добавьте `GET /version` с `env!("CARGO_PKG_VERSION")` (Урок 11.13) и пусть `deploy.sh` печатает развёрнутую версию.
- **Без сборки в ВМ.** Соберите релиз внутри образа `rust:1.100` на Mac, скопируйте только бинарный файл через `scp` и сравните время с M3. Проверьте, что glibc в ВМ не старее, чем в образе сборки.
- **`systemd-analyze security p11-hello`** в ВМ показывает «уровень уязвимости». Добавляйте параметры ужесточения, пока уровень не станет ниже 5, и записывайте, какие из них сломали службу.
- **Разверните образ в ВМ** (нужен Docker в ВМ, `limactl start template:docker` во втором экземпляре): `docker save course-deploy:1 | ssh dev docker load`.
- **Таймер для проверки.** Юнит systemd `.timer` в ВМ, который каждую минуту выполняет проверку работоспособности и пишет результат в журнал (таймер — один из типов юнитов из Урока 11.6; прочитайте `systemd.timer(5)`).
- **Без простоя.** Запустите новый релиз на порту 8083, проверьте его работоспособность, затем переключитесь; подумайте, что дал бы обратный прокси.

## Частые ошибки

1. **`cargo: command not found` в скрипте, который работает в вашем терминале.** Неинтерактивный `ssh` не загружает `~/.cargo/env`. *Исправление:* добавьте `$HOME/.cargo/bin` в `PATH` в удалённой команде.
2. **Скрипт развёртывания успешен, но по-прежнему работает старая версия.** Сборка скопирована не туда или службу не перезапустили. *Исправление:* печатайте версию или идентификатор сборки в ответе и проверяйте его в рамках проверки работоспособности.
3. **Скрипт развёртывания, который нельзя запустить дважды.** `mkdir` без `-p`, копия в `.prev`, которая падает, когда файла нет. *Исправление:* делайте каждый шаг идемпотентным.
4. **`rsync --delete` с неверной целью** удаляет настоящие файлы в ВМ. *Исправление:* цель всегда внутри `~/deploy/p11-hello`; сначала `-n`.
5. **Проверка работоспособности проходит на старом процессе.** Проверка выполнилась до окончания перезапуска. *Исправление:* повторяйте с паузой и перезапускайте **до** проверки; сравнивайте `MainPID`.
6. **Секреты в `/etc/p11-hello.env`, читаемом всеми**, или закоммиченные в Git. *Исправление:* режим `640`, группа службы; никогда не коммитьте настоящие значения (используйте файл `.example`).
7. **Работа контейнера от root или с привязкой к петле.** *Исправление:* `USER app`, `BIND=0.0.0.0:8080` внутри, `127.0.0.1:8082` на хосте.
8. **Проверка политики перезапуска через `docker kill`.** Это ручная остановка. *Исправление:* используйте настоящее падение.
9. **Оставленный маршрут `/crash`.** *Исправление:* удалите его перед коммитом и проверьте `git diff`.
10. **Забытая уборка.** *Исправление:* `docker compose down`, `docker image rm course-deploy:1`, а в ВМ, закончив урок, `sudo systemctl disable --now p11-hello`, удалите юнит, файл окружения, бинарные файлы и пользователя (`sudo userdel p11hello`), если не оставляете их как площадку для экспериментов.

## Вопросы для самопроверки

1. Зачем `deploy.sh` хранит старый бинарный файл как `.prev`?

<details>
<summary>Ответ</summary>

Чтобы после неудачной проверки работоспособности сразу вернуть последний рабочий релиз без пересборки, и служба продолжала работать на старой версии.

</details>

2. Что доказывает проверка работоспособности и чего она доказать не может?

<details>
<summary>Ответ</summary>

Она доказывает, что процесс отвечает на своём порту после перезапуска. Она не доказывает, что всё работает (базы данных, другие службы), а проверка изнутри ВМ или контейнера не доказывает, что пользователи могут достучаться до опубликованного порта.

</details>

3. Почему скрипт развёртывания должен быть безопасен при двойном запуске?

<details>
<summary>Ответ</summary>

Развёртывания повторяют, перезапускают после ошибок, их запускают уставшие люди. Скрипт, который при втором запуске падает или что-то ломает (нет папки, копирование без `-p`), ухудшает следующий инцидент. Идемпотентные шаги (`mkdir -p`, `rsync`, `install`) дают один и тот же результат каждый раз.

</details>

4. Какие настройки политики перезапуска используют два развёртывания и почему `docker kill` не проверяет политику Docker?

<details>
<summary>Ответ</summary>

systemd использует `Restart=on-failure`; Compose — `restart: unless-stopped`. `docker kill` и `docker stop` считаются ручными остановками, которые политика по замыслу уважает, поэтому правильная проверка — настоящее падение (процесс сам завершается). (Для systemd подходит `systemctl kill -s SIGKILL`, потому что systemd видит неожиданное завершение.)

</details>

5. Назовите три различия в изоляции двух развёртываний.

<details>
<summary>Ответ</summary>

Служба systemd использует файловую систему и ядро ВМ и ограничена пользователем и параметрами юнита (`ProtectSystem`, `NoNewPrivileges`, …). У контейнера своя файловая система из образа (здесь только для чтения), свой вид процессов и сети, отброшенные capabilities и ограничения ресурсов. Обе работают от пользователя без прав root.

</details>

## Итоги

- У развёртывания пять задач: доставить код или образ, настроить, запустить, проверить и иметь возможность вернуться.
- Путь systemd: бинарный файл в `/usr/local/bin`, пользователь и файл окружения, юнит с `Restart=` и ужесточением, журналы в journal, скрипт развёртывания по SSH с синхронизацией, сборкой, установкой, перезапуском, проверкой работоспособности и откатом.
- Путь Docker: многоэтапный образ без прав root с проверкой работоспособности, файл Compose с политикой перезапуска, файловой системой только для чтения, отброшенными capabilities и ротируемыми журналами.
- Повторяемые шаги (`rsync`, `install`, `mkdir -p`) делают скрипт развёртывания идемпотентным. Копия `.prev` даёт дешёвый откат.
- Проверяйте политики перезапуска настоящим падением, остановки — через `SIGTERM`, а оба — скриптом, а не на глаз.
- Сравнение двух способов выполнить одну задачу показывает, чего стоит и что даёт каждый.

## Источники и что почитать

- [systemd.service(5)](https://www.freedesktop.org/software/systemd/man/latest/systemd.service.html) — все настройки юнита службы, включая `Restart=` и `EnvironmentFile=`.
- [systemd.exec(5)](https://www.freedesktop.org/software/systemd/man/latest/systemd.exec.html) — параметры ужесточения (`ProtectSystem=`, `NoNewPrivileges=`, …).
- [rsync manual page](https://man7.org/linux/man-pages/man1/rsync.1.html) — `-a`, `--delete`, `--exclude`, `-n`.
- [Compose file reference: services](https://docs.docker.com/reference/compose-file/services/) — `restart`, `read_only`, `cap_drop`, `logging`, `healthcheck`.
- [Docker logging drivers: json-file](https://docs.docker.com/engine/logging/drivers/json-file/) — параметры ротации журналов.
- [Lima: port forwarding](https://lima-vm.io/docs/config/port/) — как порты ВМ попадают на ваш Mac.
- [The Twelve-Factor App](https://12factor.net/) — конфигурация, журналы и «одноразовость» как принципы развёртывания.

---

[🏠 Начало](../START-HERE.md) · [← 11.14 Docker Compose](14-docker-compose.md) · [📋 Этап 11](README.md) · [11.16 Контрольная точка: стадия 2 пройдена →](16-checkpoint-stage-2-complete.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
