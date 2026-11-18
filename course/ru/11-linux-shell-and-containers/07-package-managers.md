<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 11.6 Службы и systemd](06-services-with-systemd.md) · [📋 Этап 11](README.md) · [11.8 Основы скриптов на bash →](08-bash-scripting-basics.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/11-linux-shell-and-containers/07-package-managers.md) <!-- nav -->

# Урок 11.7 — Менеджеры пакетов

> **Этап 11 — Linux, командная оболочка и контейнеры** · Урок 7 из 16 · ⏱ около 120 минут

## Что вы узнаете

- Что такое пакет, репозиторий и менеджер пакетов и почему в Linux не скачивают программы вручную.
- Как находить, устанавливать, обновлять и удалять программы с помощью `apt` и как увидеть, какие файлы установил пакет.
- Почему пакеты подписаны, как настраиваются репозитории и чем рискованно добавлять сторонний репозиторий.
- Как превратить собственную программу на Rust в пакет `.deb` и когда вместо этого использовать `cargo install`.

## Перед началом

- Вы прошли [Урок 11.6 — Службы и systemd](06-services-with-systemd.md).
- Вы уже дважды использовали `apt-get install` (Уроки 11.2 и 11.5) с краткими пояснениями. Теперь вы узнаете, что он делает.
- Все команды выполняются в ВМ (`limactl shell dev`). Раздел 6 использует Rust, который вы установили в Уроке 11.5.

## Новые слова

| Слово | Значение |
|---|---|
| **пакет** (package) | Одна программа, упакованная в архив: файлы плюс метаданные (имя, версия, описание) и список других пакетов, которые ей нужны. |
| **менеджер пакетов** (package manager) | Программа, которая устанавливает, обновляет и удаляет пакеты и ведёт базу данных того, что установлено. |
| **репозиторий** (repository, repo) | Сервер со множеством пакетов и файлом-индексом, который их перечисляет. |
| **зависимость** (dependency) | Пакет, который нужен другому пакету, чтобы работать. Менеджер пакетов устанавливает зависимости автоматически. |
| **apt** (Advanced Package Tool, усовершенствованный инструмент пакетов) | Менеджер пакетов Debian и Ubuntu. Команда `apt-get` — его более старая форма, удобная для скриптов. |
| **dpkg** (Debian package) | Низкоуровневый инструмент, который распаковывает один файл `.deb` и записывает его в базу данных. `apt` вызывает `dpkg`. |
| **.deb** | Формат файла пакета Debian/Ubuntu. |
| **индекс пакетов** (package index) | Список пакетов и версий репозитория, который `apt` скачивает командой `apt update`. |
| **подпись GPG** (GNU Privacy Guard) | Криптографическая подпись. Репозиторий подписывает свой индекс; `apt` проверяет подпись открытым ключом, который у него уже есть. |
| **связка ключей** (keyring) | Файл с открытыми ключами, которым `apt` доверяет для репозитория. |
| **набор / компонент** (suite / component) | Набор (suite) — линия выпусков дистрибутива (`resolute`, `resolute-updates`, `resolute-security`). Компонент — его часть: `main`, `universe`, `restricted`, `multiverse`. |
| **PPA** (Personal Package Archive, персональный архив пакетов) | Репозиторий, который ведёт один человек или команда на Launchpad, вне официального архива Ubuntu. |
| **версия** (version) | Текст, который идентифицирует сборку пакета, например `1.8.1-4ubuntu2`: версия автора, затем ревизия дистрибутива. |

## 1. Зачем нужен менеджер пакетов

Можно скачать программу с сайта, распаковать и скопировать куда-нибудь. Для целой системы так не получится: кто её обновит? Что ей нужно? Где её файлы? Какая это версия? **Менеджер пакетов** отвечает на эти вопросы:

```text
      repository (a server)                 your machine
  +-------------------------+        +--------------------------+
  |  index: names, versions |  apt   |  database of installed   |
  |  jq_1.8.1.deb           | -----> |  packages (dpkg)         |
  |  libjq1_1.8.1.deb ...   |        |  files in /usr, /etc ... |
  +-------------------------+        +--------------------------+
        signed by the distribution            checks the signature
```

- Пакет знает свои **зависимости**. Установите `jq`, и `apt` установит и нужные ему библиотеки.
- Пакеты **подписаны**, поэтому изменённый или поддельный пакет будет отвергнут.
- Исправления безопасности приходят по тому же каналу, а одна команда обновляет всю систему.
- Каждый файл принадлежит какому-то пакету, поэтому программу можно **удалить** начисто.

Один менеджер пакетов вы уже знаете: **Cargo**. *Крейт* — его пакет, *crates.io* — репозиторий, `Cargo.lock` фиксирует точные версии. На вашем Mac **Homebrew** (Урок 11.1) делает то же для инструментов командной строки. В Ubuntu системный менеджер пакетов — это `apt`.

## 2. Пакеты и репозитории

Пакеты Ubuntu и Debian — это файлы `.deb`. **`dpkg`** — инструмент, который устанавливает один `.deb`; он ничего не знает ни об интернете, ни о зависимостях. **`apt`** — уровень выше: он читает список репозиториев, скачивает пакеты, выясняет зависимости и вызывает `dpkg`. В повседневной работе вы используете `apt`.

Репозитории настраиваются в `/etc/apt/sources.list.d/`. В Ubuntu 26.04 это файл `ubuntu.sources` (формат *deb822*, список строк `Key: value`). Без строк-комментариев:

```bash
grep -v '^#' /etc/apt/sources.list.d/ubuntu.sources
```

```text
Types: deb
URIs: http://archive.ubuntu.com/ubuntu/
Suites: resolute resolute-updates resolute-backports
Components: main universe restricted multiverse
Signed-By: /usr/share/keyrings/ubuntu-archive-keyring.gpg

Types: deb
URIs: http://security.ubuntu.com/ubuntu/
Suites: resolute-security
Components: main universe restricted multiverse
Signed-By: /usr/share/keyrings/ubuntu-archive-keyring.gpg
```

**Что происходит:** адрес (`URIs`) в вашей ВМ может отличаться (например, `ports.ubuntu.com` для arm64). Прочитайте поля:

| Поле | Значение |
|---|---|
| `Types: deb` | двоичные пакеты (`deb-src` означало бы исходный код) |
| `URIs` | сервер |
| `Suites` | `resolute` — Ubuntu 26.04 в момент выпуска; `-updates` — более поздние исправления ошибок; `-security` — исправления безопасности; `-backports` — более новые версии некоторых программ |
| `Components` | `main` — ПО, поддерживаемое Canonical; `universe` — поддерживаемое сообществом; `restricted` и `multiverse` содержат несвободные части |
| `Signed-By` | файл связки ключей с открытым ключом для проверки **только** этого репозитория |

> 🔒 **Безопасность:** `apt` не установит пакеты из репозитория, индекс которого не подписан ключом, названным в `Signed-By`. Именно это не даёт кому-то в сети (или взломанному зеркалу) подсунуть вам изменённый `sudo`. К этому мы вернёмся в разделе 5.

## 3. Повседневный `apt`

Индекс пакетов — это локальная копия. Обновляйте его перед установкой чего-либо:

```bash
sudo apt-get update
```

```text
Hit:4 http://archive.ubuntu.com/ubuntu resolute-backports InRelease
Get:5 http://archive.ubuntu.com/ubuntu resolute-updates/main arm64 Packages [1121 kB]
Fetched 1258 kB in 3s (470 kB/s)
Reading package lists...
```

(Показаны только последние строки; первые строки перечисляют каждый репозиторий: `Hit` означает «без изменений», `Get` — «скачано».) Теперь найдите пакет, прочитайте о нём и проверьте, какую версию вы получите:

```bash
apt search '^jq$'
apt show jq | head -n 8
apt-cache policy jq
```

```text
Sorting...
Full Text Search...
jq/resolute-updates,resolute-security 1.8.1-4ubuntu2 arm64
  lightweight and flexible command-line JSON processor

Package: jq
Version: 1.8.1-4ubuntu2
Priority: optional
Section: utils
Origin: Ubuntu
Maintainer: Ubuntu Developers <ubuntu-devel-discuss@lists.ubuntu.com>
Original-Maintainer: ChangZhuo Chen (陳昌倬) <czchen@debian.org>
Bugs: https://bugs.launchpad.net/ubuntu/+filebug
jq:
  Installed: (none)
  Candidate: 1.8.1-4ubuntu2
  Version table:
     1.8.1-4ubuntu2 500
        500 http://archive.ubuntu.com/ubuntu resolute-updates/main arm64 Packages
        500 http://security.ubuntu.com/ubuntu resolute-security/main arm64 Packages
     1.8.1-4ubuntu1 500
        500 http://archive.ubuntu.com/ubuntu resolute/main arm64 Packages
```

**Что происходит:**

1. `apt search` ищет по именам и описаниям; шаблон `^jq$` — регулярное выражение для точного имени ([Урок 11.3](03-text-tools-pipes-and-redirection.md)). Текст после имени показывает, в каких наборах пакет есть.
2. `apt show` печатает метаданные: версию, размер, зависимости (`Depends:`), домашнюю страницу.
3. `apt-cache policy` показывает, что установлено, какой **кандидат** (версия, которую установит `apt`) и из каких репозиториев доступна каждая версия. Число `500` — приоритет.

Установите. Параметр `-s` («simulate», имитация) показывает, что произойдёт, ничего не делая:

```bash
sudo apt-get install -s jq | grep -E 'NEW|additional|newly'
sudo apt-get install -y jq
jq --version
echo '{"name":"course","n":3}' | jq .name
```

```text
The following additional packages will be installed:
The following NEW packages will be installed:
0 upgraded, 3 newly installed, 0 to remove and 0 not upgraded.
...
Setting up libonig5:arm64 (6.9.10-1build1) ...
Setting up libjq1:arm64 (1.8.1-4ubuntu2) ...
Setting up jq (1.8.1-4ubuntu2) ...
jq-1.8.1
"course"
```

**Что происходит:**

1. Имитация сообщает «3 newly installed» (3 новых пакета): вы просили `jq`, а две его библиотеки — это **зависимости**.
2. `-y` отвечает «да» на вопрос «Do you want to continue?». Без него `apt` ждёт вас. Скриптам всегда нужен `-y`.
3. Шаги такие: скачать три файла `.deb` (`Get:`), распаковать их (`Unpacking`) и настроить (`Setting up`). Длинный вывод настоящей команды выше сокращён.
4. `jq` — фильтр JSON (JavaScript Object Notation, нотация объектов JavaScript): он напечатал поле `name`.

`apt` и `apt-get` делают одно и то же. Используйте `apt`, когда печатаете вручную (у него приятнее отображение и есть индикатор прогресса), а `apt-get` — в скриптах: его вывод и параметры стабильны, а сам `apt` предупреждает *"apt does not have a stable CLI interface. Use with caution in scripts."*

Нужные команды в одной таблице:

| Команда | Что делает |
|---|---|
| `sudo apt-get update` | обновить индекс пакетов |
| `apt list --upgradable` | показать пакеты с более новой версией |
| `sudo apt-get upgrade` | установить более новые версии установленных пакетов (ничего не удаляет) |
| `sudo apt-get full-upgrade` | как `upgrade`, но может также устанавливать или удалять пакеты для разрешения изменившихся зависимостей |
| `sudo apt-get install PKG` | установить пакет (и его зависимости) |
| `sudo apt-get remove PKG` | удалить программу; оставить её файлы настроек |
| `sudo apt-get purge PKG` | удалить программу **и** её файлы настроек |
| `sudo apt-get autoremove` | удалить зависимости, которые больше никому не нужны |
| `apt-mark showmanual` | пакеты, которые вы установили сами (остальные пришли как зависимости) |

Удалите `jq` и оставшиеся библиотеки:

```bash
sudo apt-get remove -y jq
sudo apt-get autoremove -y
```

```text
Removing jq (1.8.1-4ubuntu2) ...
Removing libjq1:arm64 (1.8.1-4ubuntu2) ...
Removing libonig5:arm64 (6.9.10-1build1) ...
```

**Что происходит:** `remove` удалил `jq`; затем `autoremove` обнаружил, что `libjq1` и `libonig5` были установлены только как его зависимости, и удалил их тоже. Пакет, у которого есть файлы настроек в `/etc`, сохраняет их после `remove` (состояние `rc` в `dpkg -l`), пока вы не выполните `purge`. Поэтому при переустановке настройки часто «вспоминаются».

> 💡 **Совет:** на настоящем сервере регулярно обновляйте: `sudo apt-get update && sudo apt-get upgrade`. Ubuntu может сама устанавливать обновления безопасности с помощью пакета `unattended-upgrades`. В ВМ вы можете попробовать свободно.

## 4. Что установил пакет?

Поскольку база данных знает каждый файл, ей можно задавать вопросы:

```bash
sudo apt-get install -y jq > /dev/null
dpkg -L jq | head -n 5
dpkg -S /usr/bin/jq
dpkg -l jq | tail -n 1
```

```text
/.
/usr
/usr/bin
/usr/bin/jq
/usr/share
jq: /usr/bin/jq
ii  jq             1.8.1-4ubuntu2 arm64        lightweight and flexible command-line JSON processor
```

**Что происходит:**

1. `dpkg -L PKG` перечисляет файлы пакета (**L**ist).
2. `dpkg -S PATH` — обратное: какому пакету **принадлежит** файл. Используйте, когда находите неизвестный файл в `/usr/bin` или `/etc`.
3. `dpkg -l PKG` печатает состояние. `ii` означает «желаемое: установить, состояние: установлен». `rc` означает «удалён, настройки остались».

Пути следуют схеме [Урока 11.2](02-the-linux-file-system.md): программа в `/usr/bin`, документация в `/usr/share/doc`. Помните, что `/bin` — символическая ссылка на `/usr/bin`; `dpkg -S` знает только настоящий путь:

```bash
dpkg -S /usr/bin/ls
dpkg -S /usr/bin/ps
```

```text
coreutils-from-uutils: /usr/bin/ls
procps: /usr/bin/ps
```

**Что происходит:** в Ubuntu 26.04 файл `/usr/bin/ls` принадлежит пакету `coreutils-from-uutils`, который ведёт к реализации на Rust (Урок 11.2). Используйте `dpkg -S $(which ps)`, чтобы найти пакет команды.

## 5. Доверие: подписи и сторонние репозитории

Установить пакет — значит выполнить код его автора **от имени root** (скрипты установки пакета работают от root). Поэтому вы должны доверять тому, кто его подписывает.

- Архив Ubuntu подписан ключом Ubuntu. `apt` проверяет подпись индекса (`InRelease`), а индекс содержит контрольную сумму каждого пакета, поэтому пакет нельзя изменить незаметно.
- У **стороннего репозитория** (производителя или PPA) свой ключ. Добавив его, вы даёте этому производителю ту же власть над своей машиной, что и самой Ubuntu: каждый `apt upgrade` будет устанавливать то, что они опубликуют, от root.

Если добавлять приходится, безопасный шаблон тот же, что в файле Ubuntu, который вы видели: скачайте открытый ключ производителя в **отдельный файл связки ключей** и привяжите репозиторий к этому ключу через `Signed-By`, чтобы ключу доверяли только для этого репозитория. Вот шаблон (это **не** настоящий репозиторий, не запускайте его):

```text
# /etc/apt/sources.list.d/example-vendor.sources
Types: deb
URIs: https://packages.example.com/ubuntu
Suites: stable
Components: main
Signed-By: /etc/apt/keyrings/example-vendor.gpg
```

> 🔒 **Безопасность:** прежде чем добавлять репозиторий, спросите себя: кто его ведёт, почему я им доверяю и нужен ли он мне? Предпочитайте архив Ubuntu. Берите ключ со страницы производителя по HTTPS (шифрованному веб-протоколу, Урок 10.11). Никогда не кладите ключ в общую папку `trusted.gpg.d` (ему будут доверять для *каждого* репозитория) и никогда не копируйте команду `curl ... | sudo bash`, которую вы не прочитали ([Урок 00.4](../00-getting-started/04-terminal-basics.md)). Когда удаляете программу, удалите также `.sources` и ключ.

## 6. Упаковка собственной программы

Программу на Rust тоже можно распространять как `.deb`. Тогда `apt` её установит, `dpkg -S` покажет её владельца, а `apt remove` удалит. Инструмент **cargo-deb** собирает пакет по `Cargo.toml`. Установите его в ВМ (Cargo из Урока 11.5; `--locked` использует версии зависимостей, которые проверил автор, как на [Этапе 10](../10-security-fundamentals/README.md)):

```bash
cargo install --locked cargo-deb --version 3.8.0
```

```text
   Compiling ar v0.9.0
   Compiling cargo_toml v1.0.1
   Compiling xz2 v0.1.7
    Finished `release` profile [optimized] target(s) in 16.21s
  Installing /home/student/.cargo/bin/cargo-deb
   Installed package `cargo-deb v3.8.0` (executable `cargo-deb`)
```

(Вывод сокращён; компиляция занимает минуту.) Откройте копию проекта в ВМ, `~/code/p11-heartbeat/Cargo.toml`, добавьте две строки в `[package]` и новый раздел в конец:

```toml
[package]
name = "p11-heartbeat"
version = "0.1.0"
edition = "2024"
description = "A tiny demo program that prints a heartbeat line"
license = "MIT"

[dependencies]
signal-hook = "0.4"

[package.metadata.deb]
maintainer = "Course Student <student@example.com>"
copyright = "2026, Course Student"
extended-description = "A tiny demo program from the Rust systems course that prints a heartbeat line."
section = "utils"
priority = "optional"
assets = [
    ["target/release/p11-heartbeat", "usr/bin/", "755"],
]
```

**Что происходит:** `description` и `license` — стандартные поля Cargo (cargo-deb предупреждает, если их нет). Раздел `[package.metadata.deb]` читает только cargo-deb. `assets` перечисляет файлы для пакета: *исходный путь*, *папка назначения внутри пакета* и *режим*. Строка maintainer — заглушка; для настоящего пакета укажите своё имя и адрес. Соберите, осмотрите и установите:

```bash
cd ~/code/p11-heartbeat
cargo build --release
cargo deb
dpkg-deb --contents target/debian/p11-heartbeat_0.1.0-1_arm64.deb
sudo apt-get install -y ./target/debian/p11-heartbeat_0.1.0-1_arm64.deb
dpkg -S /usr/bin/p11-heartbeat
timeout 3 p11-heartbeat; echo "status: $?"
```

```text
/home/student/code/p11-heartbeat/target/debian/p11-heartbeat_0.1.0-1_arm64.deb
drwxr-xr-x 0/0               0 2026-11-18 00:00 ./usr/
drwxr-xr-x 0/0               0 2026-11-18 00:00 ./usr/share/
drwxr-xr-x 0/0               0 2026-11-18 00:00 ./usr/share/doc/
drwxr-xr-x 0/0               0 2026-11-18 00:00 ./usr/share/doc/p11-heartbeat/
-rw-r--r-- 0/0             149 2026-11-18 00:00 ./usr/share/doc/p11-heartbeat/copyright
drwxr-xr-x 0/0               0 2026-11-18 00:00 ./usr/bin/
-rwxr-xr-x 0/0          398120 2026-11-18 00:00 ./usr/bin/p11-heartbeat
Setting up p11-heartbeat (0.1.0-1) ...
p11-heartbeat: /usr/bin/p11-heartbeat
starting: pid 7179, greeting world, every 2 s
heartbeat 1 from world
heartbeat 2 from world
got signal 15: shutting down
bye
status: 124
```

**Что происходит:**

1. `cargo deb` напечатал путь нового пакета. `dpkg-deb --contents` перечисляет, что внутри: программа и файл copyright. (cargo-deb также сам нашёл зависимость `libc6`; `dpkg-deb --info FILE` показывает метаданные.)
2. `apt-get install ./file.deb` (`./` обязательно, это говорит `apt`, что перед ним файл) устанавливает локальный пакет и его зависимости.
3. `dpkg -S` показывает, что программа теперь принадлежит пакету `p11-heartbeat`.
4. `timeout 3 PROGRAM` запускает программу и через 3 секунды останавливает её, посылая `SIGTERM`. Наша программа на Rust обработала сигнал, напечатала `bye`, а `timeout` сообщил свой код `124` («лимит времени достигнут»). Это обработка сигналов из [Урока 11.5](05-processes-and-signals.md) в действии.

Когда закончите, удалите пакет: `sudo apt-get remove -y p11-heartbeat`.

**Другие менеджеры пакетов.** `cargo install --locked NAME` скачивает *программу на Rust* с crates.io, компилирует её и кладёт в `~/.cargo/bin` для вашего пользователя. Используйте для инструментов разработчика (`cargo-deb`, `ripgrep`). Для системного ПО и служб используйте `apt`. Держите их раздельно: программа, установленная через `apt`, лежит в `/usr/bin` и обновляется вместе с системой; программа от `cargo install` лежит в вашем домашнем каталоге и обновляется вами. `which NAME` подскажет, какую из них вы запускаете. В Ubuntu есть и пакеты `snap`, которые приносят собственные библиотеки в песочнице; ими управляет `snap`, а не `apt`.

## Частые ошибки

1. **`E: Unable to locate package NAME`.** Индекс устарел, имя неверно или компонент не включён. *Исправление:* `sudo apt-get update`, затем `apt search NAME`.
2. **`Could not get lock /var/lib/dpkg/lock-frontend` / `Permission denied`.** Работает другой `apt` (часто автоматическое обновление) или вы забыли `sudo`. *Исправление:* подождите и повторите; никогда не удаляйте файлы блокировки вручную.
3. **Забыли `-y` в скрипте.** Скрипт зависает на «Do you want to continue? [Y/n]». *Исправление:* `sudo apt-get install -y ...`; в автоматизации также `DEBIAN_FRONTEND=noninteractive`, что отключает вопросы по настройке.
4. **`apt install ./file.deb` без `./`.** `apt` ищет в репозитории пакет с именем `file.deb`. *Исправление:* укажите путь.
5. **`sudo apt-get upgrade` без предварительного `update`.** Обновление идёт по старому индексу. *Исправление:* сначала `update`.
6. **Ключ репозитория добавлен в `trusted.gpg.d`.** Теперь ему доверяют для всех репозиториев. *Исправление:* отдельный файл связки ключей для каждого репозитория и `Signed-By`.
7. **Смешение источников одного инструмента.** `cargo` из `apt` и из `rustup` одновременно: вы запускаете не ту версию, что думаете. *Исправление:* проверьте через `which -a cargo`; оставьте один.

## Попробуйте сами

1. Выполните `sudo apt-get update`. С каким числом репозиториев (строки, начинающиеся с `Hit:` или `Get:`) он связывается?
2. Найдите пакет `htop` командой `apt search '^htop$'`, покажите его зависимости командой `apt-cache depends htop`, затем установите командой `sudo apt-get install -y htop`. Запустите `htop`, нажмите `q` и найдите его файлы командой `dpkg -L htop | grep bin`.
3. Выполните `dpkg -S $(readlink -f $(which sudo))` — из какого пакета `sudo`? (`which sudo` даёт символическую ссылку; `readlink -f` идёт по ней к настоящему файлу, Урок 11.2.)
4. Удалите `htop` командой `remove`, затем проверьте `dpkg -l htop`. После этого выполните `autoremove`.
5. Раздел 6: соберите и установите `.deb`, запустите его с `timeout 3` и удалите.

## Упражнения

### Упражнение 1 — Кому принадлежит этот файл? (легко)

Выясните, какой пакет предоставляет команду `ps` и какому пакету принадлежит файл `/etc/hostname`. Объясните второй ответ.

<details>
<summary>Подсказка</summary>

`dpkg -S` принимает путь; используйте `which ps` или `command -v ps`, чтобы найти его. Файл, созданный при установке скриптом, а не поставляемый в пакете, владельца не имеет.

</details>

<details>
<summary>Ответ</summary>

```bash
dpkg -S $(which ps)
dpkg -S /etc/hostname
```

```text
procps: /usr/bin/ps
dpkg-query: no path found matching pattern /etc/hostname
```

Почему это работает: `ps` входит в пакет `procps`. `/etc/hostname` записывается при настройке системы (он принадлежит этой машине, а не пакету), поэтому владельца-пакета у него нет.

</details>

### Упражнение 2 — Посмотрите план до изменений (средне)

Используя только имитацию (ничего не должно быть установлено), выясните, сколько новых пакетов установил бы `apt` для `htop` и как они называются.

<details>
<summary>Подсказка</summary>

Используйте `-s`. Имитация печатает по одной строке `Inst NAME (...)` для каждого пакета, который она установила бы, и итоговую строку с `newly installed`.

</details>

<details>
<summary>Ответ</summary>

```bash
apt-get install -s htop | grep -E '^Inst|newly'
```

```text
Inst htop (3.4.1-5build2 Ubuntu:26.04/resolute [arm64])
0 upgraded, 1 newly installed, 0 to remove and 0 not upgraded.
```

(Версии и число пакетов зависят от того, что установлено в вашей ВМ, а `htop` пока не должен быть установлен.)

Почему это работает: `-s` (simulate) разрешает зависимости и печатает план, не меняя систему. Всегда читайте план большого обновления, прежде чем подтверждать его.

</details>

### Упражнение 3 — Напишите проверку установки (средне)

Напишите однострочную **проверку**, которая завершается с кодом 0, если пакет установлен, и с ненулевым кодом, если нет, и ничего не печатает. Проверьте её на `jq` до и после установки.

<details>
<summary>Подсказка</summary>

`dpkg -s PKG` возвращает ненулевой код для неустановленного пакета. Скройте его вывод перенаправлениями ([Урок 11.3](03-text-tools-pipes-and-redirection.md)).

</details>

<details>
<summary>Ответ</summary>

```bash
sudo apt-get remove -y -qq jq > /dev/null
dpkg -s jq > /dev/null 2>&1; echo "installed? $?"
sudo apt-get install -y -qq jq > /dev/null
dpkg -s jq > /dev/null 2>&1; echo "installed? $?"
```

```text
installed? 1
installed? 0
```

Почему это работает: ответ — код возврата (0 = да), а `> /dev/null 2>&1` отбрасывает оба потока вывода. Скрипты используют это, чтобы устанавливать пакет только когда его нет ([Урок 11.8](08-bash-scripting-basics.md)).

</details>

### Упражнение 4 — Протестируйте пакет (сложно)

Напишите **тест пакета** для `p11-heartbeat_0.1.0-1_arm64.deb`: короткую последовательность команд shell, которая завершается с кодом 0, только если в пакете есть `./usr/bin/p11-heartbeat`, печатает `PASS` или `FAIL` и ничего не устанавливает.

<details>
<summary>Подсказка</summary>

`dpkg-deb --contents FILE` перечисляет файлы. Ищите в списке через `grep -q`.

</details>

<details>
<summary>Ответ</summary>

```bash
DEB=~/code/p11-heartbeat/target/debian/p11-heartbeat_0.1.0-1_arm64.deb
dpkg-deb --contents "$DEB" | grep -q ' \./usr/bin/p11-heartbeat$' && echo PASS || echo FAIL
dpkg-deb --contents "$DEB" | grep -q ' \./usr/bin/missing$' && echo PASS || echo FAIL
```

```text
PASS
FAIL
```

Почему это работает: первая проверка находит путь в конце строки списка содержимого; вторая — намеренная неудача, доказывающая, что тест может падать. Тест, который не может упасть, ничего не доказывает.

</details>

## Вопросы для самопроверки

1. Чем отличаются `dpkg` и `apt`?

<details>
<summary>Ответ</summary>

`dpkg` устанавливает один файл `.deb` и записывает его в базу, без репозиториев и без разрешения зависимостей. `apt` читает репозитории, скачивает пакеты, разрешает зависимости и вызывает `dpkg`.

</details>

2. Зачем выполнять `apt-get update` перед `install`?

<details>
<summary>Ответ</summary>

`update` обновляет локальный индекс пакетов. Без него `apt` может не знать пакет или выбрать старую версию.

</details>

3. Чем отличаются `remove` и `purge`?

<details>
<summary>Ответ</summary>

`remove` удаляет программу, но оставляет файлы настроек; `purge` удаляет и настройки.

</details>

4. Какая команда говорит, какому пакету принадлежит файл?

<details>
<summary>Ответ</summary>

`dpkg -S /path/to/file`.

</details>

5. Почему добавление стороннего репозитория — решение в области безопасности?

<details>
<summary>Ответ</summary>

Его пакеты устанавливаются и обновляются от root, поэтому производитель получает над вашей машиной ту же власть, что и дистрибутив. Вы должны доверять ему и привязывать его ключ только к его репозиторию.

</details>

6. Когда использовать `cargo install`, а когда `apt`?

<details>
<summary>Ответ</summary>

`cargo install --locked` — для инструментов разработчика на Rust в домашней папке; `apt` — для системного ПО и служб, которые ставятся на всю систему и обновляются вместе с ней.

</details>

## Итоги

- Менеджер пакетов устанавливает программы вместе с зависимостями, проверяет подписи и записывает каждый файл, так что удаление получается чистым.
- В Ubuntu: `dpkg` работает с одним `.deb`; `apt` — с репозиториями и зависимостями. Репозитории описаны в `/etc/apt/sources.list.d/` с наборами, компонентами и `Signed-By`.
- Порядок работы: `apt-get update`, `apt search`, `apt show`, `apt-cache policy`, `apt-get install -y`, `remove` / `purge` / `autoremove`.
- `dpkg -L` перечисляет файлы пакета, `dpkg -S` находит владельца файла, `dpkg -l` показывает состояние.
- Репозиторию доверяют с правами root: предпочитайте официальный архив, привязывайте ключи сторонних через `Signed-By`.
- `cargo deb` упаковывает программу на Rust; `apt-get install ./file.deb` устанавливает её. `cargo install --locked` — для инструментов разработчика в домашней папке.
- Проверяйте пакет через `dpkg -s` и его код возврата.

## Источники и что почитать

- [Ubuntu Server documentation — Package management](https://documentation.ubuntu.com/server/how-to/software/package-management/) — `apt` и `dpkg` в Ubuntu.
- [apt(8) manual](https://manpages.ubuntu.com/manpages/noble/man8/apt.8.html) — все команды `apt`.
- [sources.list(5) — deb822 format](https://manpages.ubuntu.com/manpages/noble/man5/sources.list.5.html) — поля файлов `.sources`, включая `Signed-By`.
- [dpkg(1) manual](https://manpages.ubuntu.com/manpages/noble/man1/dpkg.1.html) — `-L`, `-S`, `-l`, `-s` и другое.
- [cargo-deb on GitHub](https://github.com/kornelski/cargo-deb) — параметры `[package.metadata.deb]`.
- [The Cargo Book — cargo install](https://doc.rust-lang.org/cargo/commands/cargo-install.html) — что делает `cargo install` и куда он устанавливает.

---

[🏠 Начало](../START-HERE.md) · [← 11.6 Службы и systemd](06-services-with-systemd.md) · [📋 Этап 11](README.md) · [11.8 Основы скриптов на bash →](08-bash-scripting-basics.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
