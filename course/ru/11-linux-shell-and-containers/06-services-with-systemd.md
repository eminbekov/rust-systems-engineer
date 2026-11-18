<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 11.5 Процессы и сигналы](05-processes-and-signals.md) · [📋 Этап 11](README.md) · [11.7 Менеджеры пакетов →](07-package-managers.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/11-linux-shell-and-containers/06-services-with-systemd.md) <!-- nav -->

# Урок 11.6 — Службы и systemd

> **Этап 11 — Linux, командная оболочка и контейнеры** · Урок 6 из 16 · ⏱ около 120 минут

## Что вы узнаете

- Что такое служба и что делает `systemd`: запускает, останавливает, перезапускает программы и следит за ними.
- Как написать *юнит-файл* (unit file) для собственной программы на Rust и запустить её как службу от отдельного пользователя.
- Как управлять ею с помощью `systemctl` и читать её журналы с помощью `journalctl`.
- Как политика перезапуска возвращает упавшую службу и как несколько строк усиления защиты ограничивают ущерб от ошибки.

## Перед началом

- Вы прошли [Урок 11.5 — Процессы и сигналы](05-processes-and-signals.md). Вы собрали `p11-heartbeat` в `~/code/p11-heartbeat` внутри ВМ командой `cargo build --release`. Если нет, вернитесь: этот урок использует ту программу.
- Вы знаете системного пользователя из [Урока 11.4](04-users-groups-and-permissions.md), папки `/etc` и `/usr/local/bin` из [Урока 11.2](02-the-linux-file-system.md), а `sudo tee` — это идея `tee` из [Урока 11.3](03-text-tools-pipes-and-redirection.md).
- Все команды выполняются внутри ВМ (`limactl shell dev`). Если у вас есть только Docker, раздел 8 показывает контейнер с systemd.

## Новые слова

| Слово | Значение |
|---|---|
| **systemd** | Первый процесс (PID 1; PID, process ID, идентификатор процесса — номер процесса, Урок 11.5) в большинстве современных систем Linux. Он запускает систему, запускает службы и следит за ними и собирает их журналы. |
| **служба** (service) | Программа, которая долго работает в фоне, запускается и контролируется системой, например веб-сервер или база данных. |
| **юнит** (unit) | Всё, чем управляет systemd. Описывается *юнит-файлом*. Типы: `.service`, `.timer`, `.target`, `.socket` … |
| **юнит-файл** (unit file) | Небольшой текстовый файл в формате INI (назван по файлам *инициализации* (initialization) старых систем: `[Section]` и `Key=value`), который говорит, что запускать и как. |
| **цель** (target) | Группа юнитов, обозначающая состояние системы, например `multi-user.target` («система запущена и готова для пользователей»). |
| **включить** (enable) | Сделать так, чтобы служба запускалась автоматически при загрузке. (`start` только запускает её сейчас.) |
| **журнал** (journal) | Центральный журнал systemd. Читается командой `journalctl`. Всё, что служба пишет в stdout и stderr, попадает туда. |
| **политика перезапуска** (restart policy) | Правило, которое говорит, когда systemd запускает службу снова после её завершения: никогда, при сбое, всегда. |
| **cgroup** (control group, контрольная группа) | Возможность Linux объединять процессы в группу, чтобы система могла отслеживать их, ограничивать их ресурсы и останавливать все вместе. systemd помещает каждую службу в её собственную cgroup. |
| **усиление защиты** (hardening) | Настройки, уменьшающие то, что программе позволено делать, чтобы ошибка или злоумышленник могли нанести меньше вреда. |

## 1. Что делает systemd

Когда ядро Linux запустилось, оно выполняет одну программу как **PID 1**. В Ubuntu это `systemd`. PID 1 запускает всё остальное в правильном порядке, а также
**усыновляет сирот** и **собирает коды возврата** ([Урок 11.5](05-processes-and-signals.md)). Для долго работающих программ он даёт:

- запуск при загрузке в порядке зависимостей;
- перезапуск после сбоя;
- одно место для журналов и одну команду для просмотра состояния;
- отдельного пользователя, ограничения и защиту для каждой службы;
- аккуратную остановку: он посылает `SIGTERM`, ждёт и посылает `SIGKILL` только если программа не подчиняется.

Последний пункт — причина, по которой Урок 11.5 сначала учил сигналам: программа, которую вы написали, уже корректно завершается по `SIGTERM`.

Посмотрите, что работает в вашей ВМ (в вашей ВМ список длиннее):

```bash
ps -o pid,cmd -p 1
systemctl list-units --type=service --state=running --no-pager
```

```text
    PID CMD
      1 /sbin/init
  UNIT                     LOAD   ACTIVE SUB     DESCRIPTION
  dbus.service             loaded active running D-Bus System Message Bus
  getty@tty1.service       loaded active running Getty on tty1
  systemd-journald.service loaded active running Journal Service
  systemd-logind.service   loaded active running User Login Management
```

**Что происходит:** `/sbin/init` — ссылка на `systemd`. `systemctl` («system control») — команда, которая общается с systemd. `list-units` показывает загруженные юниты; `--type=service` оставляет службы; `--state=running` оставляет работающие.
`LOAD` говорит, прочитан ли юнит-файл, `ACTIVE` — общее состояние, `SUB` — подробное состояние.

## 2. Подготовка: пользователь и место для программы

Служба должна лежать в предсказуемом месте и работать от собственного пользователя. Создайте системного пользователя из Урока 11.4 и *установите* бинарный файл, который вы собрали в Уроке 11.5:

```bash
sudo useradd --system --no-create-home --shell /usr/sbin/nologin heartbeat
sudo install -m 755 ~/code/p11-heartbeat/target/release/p11-heartbeat /usr/local/bin/p11-heartbeat
ls -l /usr/local/bin/p11-heartbeat
```

```text
-rwxr-xr-x 1 root root 571376 Nov 18 07:06 /usr/local/bin/p11-heartbeat
```

**Что происходит:**

1. `useradd --system ...` создаёт пользователя `heartbeat`, под которым никто не может войти.
2. `install -m 755 source target` копирует файл и задаёт его режим за один шаг (безопасная замена `cp` плюс `chmod`).
3. Файл принадлежит **root**, а не `heartbeat`: пользователь службы может запускать программу, но не может её менять. `/usr/local/bin` — место по FHS (Filesystem Hierarchy Standard, стандарт иерархии файловой системы) для программ, которые вы устанавливаете сами ([Урок 11.2](02-the-linux-file-system.md)). Размер зависит от вашей сборки.

> 🔒 **Безопасность:** программа принадлежит root, а процесс работает от `heartbeat`. Если злоумышленник захватит службу, он не сможет перезаписать программу, чтобы закрепиться. Держите **код во владении администратора, а данные — во владении пользователя службы**.

## 3. Юнит-файл

Юнит-файлы, которые пишете вы, лежат в `/etc/systemd/system/`. (Файлы в `/usr/lib/systemd/system/` приходят из пакетов; их не редактируют.) Создайте `heartbeat.service`:

```bash
sudo tee /etc/systemd/system/heartbeat.service > /dev/null <<'EOF'
[Unit]
Description=Heartbeat demo service from the course
After=network.target

[Service]
User=heartbeat
Group=heartbeat
Environment=HEARTBEAT_NAME=systemd
Environment=HEARTBEAT_SECONDS=2
ExecStart=/usr/local/bin/p11-heartbeat
ExecReload=/bin/kill -HUP $MAINPID
Restart=on-failure
RestartSec=2
NoNewPrivileges=yes
ProtectSystem=strict
ProtectHome=yes
PrivateTmp=yes

[Install]
WantedBy=multi-user.target
EOF
systemd-analyze verify /etc/systemd/system/heartbeat.service
echo "verify exit status: $?"
```

```text
verify exit status: 0
```

**Что происходит:**

1. `sudo tee file > /dev/null <<'EOF' ... EOF` записывает текст в файл, который может создать только root. Мы используем `tee`, а не `sudo cat > file`, потому что перенаправление `>` выполняет **ваша** оболочка *до* запуска `sudo`, а у вашей оболочки нет права писать туда: получилось бы `Permission denied`. С `tee` программа, которая пишет файл, работает от root. `> /dev/null` скрывает копию, которую `tee` печатает на экран.
2. `systemd-analyze verify` проверяет юнит-файл и ничего не печатает, если всё в порядке.

В файле три раздела:

| Раздел | Ключ | Значение |
|---|---|---|
| `[Unit]` | `Description=` | текст, показываемый в `status` и в журналах |
| | `After=network.target` | запускать после настройки сети (только порядок, не требование) |
| `[Service]` | `User=`, `Group=` | запускать от этого пользователя и группы |
| | `Environment=` | задать переменную окружения процесса (программа читает `HEARTBEAT_NAME`) |
| | `ExecStart=` | команда для запуска: **абсолютный путь**, без оболочки, без конвейеров |
| | `ExecReload=` | команда для `systemctl reload`: здесь она посылает `SIGHUP` главному процессу (`$MAINPID`) |
| | `Restart=on-failure`, `RestartSec=2` | перезапускать после сбоя, подождать 2 секунды |
| | `NoNewPrivileges=`, `ProtectSystem=strict`, `ProtectHome=`, `PrivateTmp=` | усиление защиты (раздел 7) |
| `[Install]` | `WantedBy=multi-user.target` | когда служба *включена*, запускать её как часть обычного запуска системы |

По умолчанию `Type=simple`: systemd считает службу запущенной, как только выполнил `ExecStart`, а программа должна оставаться на **переднем плане** (без демонизации). Наша программа делает именно так.

> 💡 **Совет:** поскольку systemd собирает stdout и stderr, службе **не нужно** вести собственные файлы журналов. Она просто печатает строки через `println!` / `eprintln!`, а остальное делает journal ([Урок 09.10](../09-testing-debugging-benchmarking/10-logging-and-tracing.md)). Это же правило 12-факторного приложения ([Урок 08.15](../08-software-design/15-the-twelve-factor-app.md)).

## 4. Запуск, остановка, состояние

systemd читает юнит-файлы при запуске и когда вы ему скажете. После каждого изменения юнит-файла выполняйте `daemon-reload`:

```bash
sudo systemctl daemon-reload
systemctl status heartbeat --no-pager
sudo systemctl start heartbeat
systemctl status heartbeat --no-pager
```

```text
○ heartbeat.service - Heartbeat demo service from the course
     Loaded: loaded (/etc/systemd/system/heartbeat.service; disabled; preset: enabled)
     Active: inactive (dead)
● heartbeat.service - Heartbeat demo service from the course
     Loaded: loaded (/etc/systemd/system/heartbeat.service; disabled; preset: enabled)
     Active: active (running) since Wed 2026-11-18 07:06:43 UTC; 15s ago
 Invocation: f9db13e9be7a457ba0d53b32e9a178b4
   Main PID: 148 (p11-heartbeat)
      Tasks: 2 (limit: 9563)
     Memory: 2.8M (peak: 2.8M)
        CPU: 5ms
     CGroup: /system.slice/heartbeat.service
             └─148 /usr/local/bin/p11-heartbeat

Nov 18 07:06:43 lima-dev systemd[1]: Started heartbeat.service - Heartbeat demo service from the course.
Nov 18 07:06:43 lima-dev p11-heartbeat[148]: starting: pid 148, greeting systemd, every 2 s
Nov 18 07:06:43 lima-dev p11-heartbeat[148]: heartbeat 1 from systemd
Nov 18 07:06:45 lima-dev p11-heartbeat[148]: heartbeat 2 from systemd
```

(Время, PID и длинный идентификатор в вашей ВМ будут другими.) **Как читать `status`:**

1. Первый символ: `○` неактивна, `●` активна, `×` сбой (красный/зелёный в цветном терминале).
2. `Loaded:` показывает, где лежит юнит-файл и `enabled` ли он (запускается при загрузке) или `disabled`.
3. `Active:` — текущее состояние и с какого времени. `active (running)` — здоровое состояние службы.
4. `Main PID`, `Tasks` (потоки и процессы), `Memory`, `CPU` — живые числа из **cgroup** службы. `CGroup:` перечисляет все процессы службы. У нашей программы две задачи: главный поток и рабочий поток.
5. Последние строки — свежие записи журнала.

Команды, которыми вы будете пользоваться ежедневно:

| Команда | Что делает |
|---|---|
| `sudo systemctl start NAME` | запустить сейчас |
| `sudo systemctl stop NAME` | остановить сейчас (посылает `SIGTERM`, затем `SIGKILL` после тайм-аута) |
| `sudo systemctl restart NAME` | остановить и запустить |
| `sudo systemctl reload NAME` | выполнить `ExecReload` (здесь: `SIGHUP`, без перезапуска) |
| `sudo systemctl enable NAME` | запускать при загрузке (`enable --now` ещё и запускает сейчас) |
| `sudo systemctl disable NAME` | не запускать при загрузке |
| `systemctl status NAME` | состояние плюс последние строки журнала |
| `systemctl is-active NAME` | печатает `active` или `inactive`; код возврата 0 только для `active` |
| `systemctl is-enabled NAME` | печатает `enabled` или `disabled` |
| `systemctl cat NAME` | показать юнит-файл, который использует systemd |
| `systemctl --failed` | перечислить юниты со сбоем |

`NAME` можно писать с `.service` или без. Чтение (`status`, `is-active`) не требует `sudo`; изменение состояния требует.

Попробуйте `reload`, `stop` и `enable`:

```bash
sudo systemctl reload heartbeat
sudo systemctl stop heartbeat
systemctl is-active heartbeat
sudo systemctl enable heartbeat
ls -l /etc/systemd/system/multi-user.target.wants/heartbeat.service
systemctl is-enabled heartbeat
sudo systemctl disable heartbeat
```

```text
Reloaded heartbeat.service - Heartbeat demo service from the course.
inactive
Created symlink '/etc/systemd/system/multi-user.target.wants/heartbeat.service' → '/etc/systemd/system/heartbeat.service'.
lrwxrwxrwx 1 root root 37 Nov 18 07:07 /etc/systemd/system/multi-user.target.wants/heartbeat.service -> /etc/systemd/system/heartbeat.service
enabled
Removed '/etc/systemd/system/multi-user.target.wants/heartbeat.service'.
```

**Что происходит:** `enable` — это всего лишь **символическая ссылка** ([Урок 11.2](02-the-linux-file-system.md)) из папки «wants» цели, названной в `WantedBy=`, на ваш юнит-файл. При загрузке systemd запускает всё, что связано в `multi-user.target.wants/`. `disable` удаляет ссылку. `start`/`stop` и `enable`/`disable` независимы: служба может работать и быть выключенной (disabled) или быть остановленной и включённой (enabled).

## 5. Журналы с `journalctl`

Снова запустите службу, подождите около двадцати секунд и посмотрите её журнал (числа у вас зависят от того, сколько вы ждали):

```bash
sudo systemctl start heartbeat
journalctl -u heartbeat -n 5 --no-pager
journalctl -u heartbeat --since "1 minute ago" --no-pager -o cat | tail -n 2
```

```text
Nov 18 07:06:49 lima-dev p11-heartbeat[148]: heartbeat 4 from systemd
Nov 18 07:06:51 lima-dev p11-heartbeat[148]: heartbeat 5 from systemd
Nov 18 07:06:53 lima-dev p11-heartbeat[148]: heartbeat 6 from systemd
Nov 18 07:06:55 lima-dev p11-heartbeat[148]: heartbeat 7 from systemd
Nov 18 07:06:57 lima-dev p11-heartbeat[148]: heartbeat 8 from systemd
heartbeat 7 from systemd
heartbeat 8 from systemd
```

**Что происходит:**

1. `-u heartbeat` выбирает юнит. В каждой строке время, имя компьютера, имя программы с её PID и текст, который напечатала программа.
2. `-n 5` показывает последние пять строк; `--no-pager` печатает напрямую, а не открывает `less`.
3. `--since "1 minute ago"` фильтрует по времени (есть также `--until` и даты вроде `"2026-11-18 07:00"`). `-o cat` печатает только сообщение, без времени и имени компьютера.

Полезные формы: `journalctl -u heartbeat -f` следит за новыми строками вживую (остановка по `Ctrl + C`, как у `tail -f`); `journalctl -b` показывает только текущую загрузку; `journalctl -p err` показывает только ошибки и серьёзнее; `journalctl -u heartbeat -o json` печатает структурированные поля.

> 💡 **Совет:** если `journalctl` сообщает, что не может читать журнал, ваш пользователь не входит в группу `adm` или `systemd-journal`; используйте `sudo journalctl ...`. В ВМ Lima вашему пользователю читать журнал разрешено.

## 6. Политика перезапуска: упавшая служба возвращается

Теперь — причина, ради которой нужен надзиратель. Убейте службу сигналом `SIGKILL`, как выглядит сбой для systemd:

```bash
PID=$(systemctl show -p MainPID --value heartbeat)
echo "main pid: $PID"
sudo kill -9 $PID
sleep 4
systemctl show -p NRestarts -p MainPID heartbeat
journalctl -u heartbeat --since "-10s" --no-pager | grep -v 'heartbeat [0-9]'
```

```text
main pid: 148
MainPID=188
NRestarts=1
Nov 18 07:06:59 lima-dev systemd[1]: heartbeat.service: Main process exited, code=killed, status=9/KILL
Nov 18 07:06:59 lima-dev systemd[1]: heartbeat.service: Failed with result 'signal'.
Nov 18 07:07:02 lima-dev systemd[1]: heartbeat.service: Scheduled restart job, restart counter is at 1.
Nov 18 07:07:02 lima-dev systemd[1]: Started heartbeat.service - Heartbeat demo service from the course.
Nov 18 07:07:02 lima-dev p11-heartbeat[188]: starting: pid 188, greeting systemd, every 2 s
```

**Что происходит:**

1. `systemctl show -p MainPID --value NAME` печатает только PID главного процесса; `kill -9` его завершает. `grep -v` скрывает обычные строки heartbeat. У службы не было шанса прибраться: в журнале нет `bye`.
2. systemd заметил: `code=killed, status=9/KILL`, «Failed with result 'signal'». Из-за `Restart=on-failure` он запланировал перезапуск через `RestartSec=2` и запустил **новый процесс с новым PID** (здесь 188). `NRestarts=1` это считает.
3. Обычный `systemctl stop` не является сбоем, поэтому перезапуска нет. Чистое завершение (код 0) тоже не сбой.

| `Restart=` | Когда перезапускать |
|---|---|
| `no` (по умолчанию) | никогда |
| `on-failure` | код возврата не 0, убит «нечистым» сигналом (`SIGHUP`, `SIGINT`, `SIGTERM` и `SIGPIPE` считаются чистыми) или тайм-аут |
| `always` | всегда, даже после чистого завершения, если только вы не выполнили `systemctl stop` |

systemd отказывается перезапускать службу бесконечно, если она снова и снова падает сразу (по умолчанию: пять запусков за десять секунд, затем он сдаётся и показывает `failed` с «start-limit-hit»). Это защищает машину от цикла перезапусков. Вы исправляете ошибку; затем `sudo systemctl reset-failed NAME` и запускаете снова.

## 7. Усиление защиты: дайте службе меньше

`NoNewPrivileges=yes` не даёт процессу и его потомкам получить дополнительные права (например, через программу с setuid). `ProtectSystem=strict` делает всю файловую систему доступной службе только для чтения (кроме `/dev`, `/proc`, `/sys`), `ProtectHome=yes` скрывает `/home`, `/root` и `/run/user`, а `PrivateTmp=yes` даёт службе собственный пустой `/tmp`. systemd умеет оценить юнит:

```bash
systemd-analyze security heartbeat.service --no-pager | tail -n 1
```

```text
→ Overall exposure level for heartbeat.service: 8.3 EXPOSED :-(
```

**Что происходит:** команда проверяет около 80 настроек; чем ниже число, тем меньше служба может сделать. 8.3 — «EXPOSED» (уязвима) даже с четырьмя директивами, потому что остальные двери открыты. Добавьте эти строки в раздел `[Service]` (например, через `sudo nano /etc/systemd/system/heartbeat.service` или записав файл заново), затем перезагрузите и перезапустите:

```text
PrivateDevices=yes
ProtectKernelTunables=yes
ProtectKernelModules=yes
ProtectControlGroups=yes
RestrictAddressFamilies=AF_UNIX
RestrictNamespaces=yes
SystemCallFilter=@system-service
UMask=0077
```

```bash
sudo systemctl daemon-reload
sudo systemctl restart heartbeat
systemctl is-active heartbeat
systemd-analyze security heartbeat.service --no-pager | tail -n 1
```

```text
active
→ Overall exposure level for heartbeat.service: 4.2 OK :-)
```

**Что происходит:** служба по-прежнему работает, а уровень уязвимости упал с 8.3 до 4.2. Что делает каждая строка: нет доступа к файлам устройств (`PrivateDevices`), настройки ядра, модули и cgroups только для чтения, разрешены только сокеты Unix (`RestrictAddressFamilies=AF_UNIX`; нашей программе сеть не нужна), нет новых пространств имён, разрешены только системные вызовы, которые нужны обычным службам (`SystemCallFilter=@system-service`), а новые файлы приватны (`UMask=0077`).
Если вы добавите ещё и пустой `CapabilityBoundingSet=` (отказ от всех особых прав), оценка упадёт до 2.1, и эта программа по-прежнему работает.

> 🔒 **Безопасность:** усиливайте защиту шаг за шагом: добавьте одну-две строки, перезапустите, прочитайте журнал. Слишком строгая настройка проявляется как сбой при запуске (`status=...`) или в журнале, и вы убираете эту одну строку. Для службы, которая открывает сетевой порт (проект в Уроке 11.15 и службы следующих этапов), разрешите `AF_INET` и `AF_INET6`.

## 8. Если у вас есть только Docker

Контейнер обычно не запускает systemd. Чтобы пройти этот урок без ВМ Lima, соберите одноразовый образ. Имена начинаются с `course-`; в конце всё удалите. Сохраните это как `Dockerfile` в пустой папке:

```text
FROM ubuntu:26.04
RUN apt-get update \
 && apt-get install -y --no-install-recommends systemd systemd-sysv dbus sudo procps \
 && rm -rf /var/lib/apt/lists/*
STOPSIGNAL SIGRTMIN+3
CMD ["/sbin/init"]
```

```bash
docker build -t course-systemd .
docker run -d --name course-systemd-lab --privileged course-systemd
docker exec -it course-systemd-lab bash
```

Внутри контейнера вы `root`, поэтому `sudo` опускайте. Чтобы получить Linux-бинарник `p11-heartbeat`, соберите его официальным образом Rust из папки проекта на вашем Mac и скопируйте внутрь:

```bash
cd ~/projects/rust/code/p11-heartbeat
docker run --rm -v "$PWD":/w -w /w -e CARGO_TARGET_DIR=/w/target-linux rust:1.100-slim cargo build --release
docker cp target-linux/release/p11-heartbeat course-systemd-lab:/tmp/p11-heartbeat
```

Внутри контейнера используйте `/tmp/p11-heartbeat` как файл для `install` в разделе 2. Когда закончите: `docker rm -f course-systemd-lab` и `docker rmi course-systemd`. `--privileged` даёт контейнеру широкие права в ВМ Docker, поэтому используйте его только для такой лаборатории и только с этим образом.

## 9. Уборка

```bash
sudo systemctl stop heartbeat
sudo rm /etc/systemd/system/heartbeat.service
sudo rm /usr/local/bin/p11-heartbeat
sudo systemctl daemon-reload
sudo userdel heartbeat
```

Оставьте `~/code/p11-heartbeat` — один из следующих уроков снова использует исходный код. (Не забудьте удалить пользователя службы после удаления юнита, иначе `useradd` в следующем уроке завершится ошибкой «already exists».)

## Частые ошибки

1. **Нет `daemon-reload` после правки юнита.** `status` показывает `Warning: The unit file, source configuration file or drop-ins of heartbeat.service changed on disk. Run 'systemctl daemon-reload' to reload units.` *Исправление:* выполните `sudo systemctl daemon-reload`, затем перезапустите.
2. **`status=203/EXEC`.** Путь в `ExecStart=` неверен или файл не исполняем: `Failed at step EXEC spawning /usr/local/bin/does-not-exist: No such file or directory`. *Исправление:* используйте абсолютный путь, проверьте через `ls -l`.
3. **`status=217/USER`.** Пользователя из `User=` не существует: `Failed to determine credentials for user 'heartbeat': Unknown user`. *Исправление:* сначала создайте пользователя (Урок 11.4).
4. **`sudo echo ... > /etc/...`.** `Permission denied`, потому что перенаправление делает ваша оболочка. *Исправление:* `... | sudo tee file`.
5. **Программа, которая демонизируется сама** (делает fork и выходит) при `Type=simple`: systemd решает, что она завершилась. *Исправление:* держите программу на переднем плане; пусть следит systemd.
6. **Запись файлов журнала в самой программе.** Вы теряете ротацию, поиск по времени и единое место для просмотра. *Исправление:* печатайте в stdout/stderr и используйте `journalctl`.
7. **Запуск службы от root, «чтобы заработало».** *Исправление:* найдите недостающее право через `namei -l` (Урок 11.4) и дайте именно его.

## Попробуйте сами

1. Пройдите разделы 2–4: создайте пользователя, установите бинарный файл, напишите юнит-файл, запустите службу и прочитайте `status`.
2. Выполните `journalctl -u heartbeat -f` в одной вкладке терминала (вторую откройте через `limactl shell dev`), а в другой выполните `sudo systemctl reload heartbeat` и `sudo systemctl restart heartbeat`. Какие строки появляются в каждом случае?
3. Измените `HEARTBEAT_SECONDS=2` на `5` в юнит-файле. Что нужно выполнить, чтобы изменение вступило в силу?
4. Убейте службу через `kill -9` и через `sudo systemctl stop`. Сравните строки журнала и поведение `Restart=`.
5. Раздел 9: уберите за собой и убедитесь, что `systemctl status heartbeat` сообщает, что юнит не найден.

## Упражнения

### Упражнение 1 — Прочитайте сбой (легко)

Создайте `/etc/systemd/system/broken.service` с `ExecStart=/usr/local/bin/does-not-exist`, выполните `daemon-reload` и `start` и прочитайте состояние. Что означает `status=203/EXEC`? Потом удалите юнит.

<details>
<summary>Подсказка</summary>

Запишите файл через `sudo tee`, как в разделе 3. Минимальному юниту нужен только раздел `[Service]` с `ExecStart=`.

</details>

<details>
<summary>Ответ</summary>

```bash
sudo tee /etc/systemd/system/broken.service > /dev/null <<'EOF'
[Service]
ExecStart=/usr/local/bin/does-not-exist
EOF
sudo systemctl daemon-reload
sudo systemctl start broken
systemctl status broken --no-pager -n 3
sudo rm /etc/systemd/system/broken.service
sudo systemctl daemon-reload
sudo systemctl reset-failed
```

```text
× broken.service
     Loaded: loaded (/etc/systemd/system/broken.service; static)
     Active: failed (Result: exit-code) since Wed 2026-11-18 07:07:26 UTC; 1s ago
    Process: 481 ExecStart=/usr/local/bin/does-not-exist (code=exited, status=203/EXEC)
   Main PID: 481 (code=exited, status=203/EXEC)

Nov 18 07:07:26 lima-dev (does-not-exist)[481]: broken.service: Failed at step EXEC spawning /usr/local/bin/does-not-exist: No such file or directory
```

Почему это работает: `203/EXEC` означает, что systemd не смог выполнить файл из `ExecStart=`: здесь его нет. `reset-failed` снимает пометку «сбой».

</details>

### Упражнение 2 — Настройка без перекомпиляции (средне)

Измените приветствие работающей службы на `Rust`, а интервал — на 1 секунду, **не трогая** код на Rust, и докажите это по журналу.

<details>
<summary>Подсказка</summary>

Программа читает две переменные окружения. Измените строки `Environment=` в юнит-файле, затем `daemon-reload` и `restart`.

</details>

<details>
<summary>Ответ</summary>

```bash
sudo sed -i 's/HEARTBEAT_NAME=systemd/HEARTBEAT_NAME=Rust/; s/HEARTBEAT_SECONDS=2/HEARTBEAT_SECONDS=1/' /etc/systemd/system/heartbeat.service
sudo systemctl daemon-reload
sudo systemctl restart heartbeat
sleep 3
journalctl -u heartbeat -n 3 --no-pager -o cat
```

```text
heartbeat 1 from Rust
heartbeat 2 from Rust
heartbeat 3 from Rust
```

(Вы можете увидеть другие числа: они зависят от того, сколько вы ждали.)

Почему это работает: настройки живут в окружении, вне бинарного файла ([Урок 08.15](../08-software-design/15-the-twelve-factor-app.md)). `sed -i 's/old/new/'` правит файл на месте ([Урок 11.3](03-text-tools-pipes-and-redirection.md)).

</details>

### Упражнение 3 — Дымовой тест для службы (сложно)

**Дымовой тест** (smoke test) — быстрая проверка, что базовые вещи работают. Напишите короткий shell-тест, который завершается с кодом 0, только если (а) служба `heartbeat` активна и (б) в журнале за последнюю минуту есть строка `heartbeat`, за которой следует число. Печатайте `PASS` или `FAIL`. Запустите его при запущенной и остановленной службе.

<details>
<summary>Подсказка</summary>

`systemctl is-active --quiet NAME` только задаёт код возврата. `grep -q` делает то же для шаблона. Соедините проверки через `&&`.

</details>

<details>
<summary>Ответ</summary>

```bash
sudo systemctl start heartbeat
sleep 3
systemctl is-active --quiet heartbeat \
  && journalctl -u heartbeat --since "1 minute ago" --no-pager -o cat | grep -q 'heartbeat [0-9]' \
  && echo PASS || echo FAIL
sudo systemctl stop heartbeat
systemctl is-active --quiet heartbeat \
  && journalctl -u heartbeat --since "1 minute ago" --no-pager -o cat | grep -q 'heartbeat [0-9]' \
  && echo PASS || echo FAIL
```

```text
PASS
FAIL
```

Почему это работает: каждая команда завершается успехом или неудачей через код возврата. `&&` выполняет следующую часть только после успеха, а `||` выполняет `echo FAIL` после любой неудачи. Такой же тест можно запускать после каждого развёртывания; [Урок 11.8](08-bash-scripting-basics.md) собирает такие проверки в скрипт, а проект [Урока 11.15](15-project-deploy-a-rust-service.md) их использует.

</details>

## Вопросы для самопроверки

1. Что такое PID 1 в Ubuntu и что он делает для служб?

<details>
<summary>Ответ</summary>

`systemd`. Он запускает службы по порядку, следит за ними, перезапускает по политике, собирает их журналы и корректно останавливает.

</details>

2. Чем отличаются `start` и `enable`?

<details>
<summary>Ответ</summary>

`start` запускает службу сейчас. `enable` делает так, что она запускается при загрузке (символическая ссылка в `multi-user.target.wants/`). Они независимы.

</details>

3. Почему юнит-файлы мы пишем через `sudo tee`, а не `sudo cat > file`?

<details>
<summary>Ответ</summary>

Перенаправление `>` выполняет ваша обычная оболочка до запуска `sudo`, поэтому у неё нет права писать в `/etc`. С `tee` пишущая программа работает от root.

</details>

4. Какая команда показывает журнал одной службы и как следить за ним вживую?

<details>
<summary>Ответ</summary>

`journalctl -u NAME`; добавьте `-f`, чтобы следить.

</details>

5. Служба упала от `SIGKILL` и вернулась с новым PID. Какая строка юнит-файла это сделала?

<details>
<summary>Ответ</summary>

`Restart=on-failure` (с `RestartSec=` как задержкой).

</details>

6. Почему бинарный файл службы принадлежит root, а процесс работает от `heartbeat`?

<details>
<summary>Ответ</summary>

Чтобы взломанная служба не могла заменить собственную программу. Наименьшие привилегии: пользователь службы может запускать файл, но не может его менять.

</details>

## Итоги

- `systemd` — это PID 1. Он запускает, контролирует, перезапускает и останавливает службы и собирает их журналы в journal.
- Юнит-файл в `/etc/systemd/system/` имеет разделы `[Unit]`, `[Service]` и `[Install]`. `ExecStart=` требует абсолютный путь и программу, остающуюся на переднем плане.
- Выполняйте `sudo systemctl daemon-reload` после каждого изменения. Затем `start`, `stop`, `restart`, `reload`, `enable`, `status`, `is-active`.
- `journalctl -u NAME` (с `-n`, `-f`, `--since`, `-o cat`) читает то, что программа напечатала в stdout и stderr.
- `Restart=on-failure` и `RestartSec=` возвращают упавшую службу; чистая остановка по `SIGTERM` не перезапускается.
- Запускайте службу от собственного системного пользователя, держите бинарный файл во владении root и усиливайте защиту, ориентируясь на `systemd-analyze security`.
- Служба, обрабатывающая `SIGTERM` (Урок 11.5), корректно останавливается, когда systemd её об этом просит.

## Источники и что почитать

- [systemd.service(5)](https://www.freedesktop.org/software/systemd/man/latest/systemd.service.html) — все параметры раздела `[Service]`.
- [systemd.unit(5)](https://www.freedesktop.org/software/systemd/man/latest/systemd.unit.html) — общий формат юнит-файлов, а также `[Unit]` и `[Install]`.
- [systemctl(1)](https://www.freedesktop.org/software/systemd/man/latest/systemctl.html) — каждая команда `systemctl`.
- [journalctl(1)](https://www.freedesktop.org/software/systemd/man/latest/journalctl.html) — фильтры и форматы вывода журнала.
- [systemd.exec(5)](https://www.freedesktop.org/software/systemd/man/latest/systemd.exec.html) — параметры песочницы (`ProtectSystem`, `PrivateTmp`, `SystemCallFilter` …).
- [systemd-analyze(1)](https://www.freedesktop.org/software/systemd/man/latest/systemd-analyze.html) — `verify` и `security`.

---

[🏠 Начало](../START-HERE.md) · [← 11.5 Процессы и сигналы](05-processes-and-signals.md) · [📋 Этап 11](README.md) · [11.7 Менеджеры пакетов →](07-package-managers.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
