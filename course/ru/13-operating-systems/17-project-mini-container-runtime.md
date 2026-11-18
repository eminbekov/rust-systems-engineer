<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 13.16 Контрольные группы (cgroups)](16-cgroups.md) · [📋 Этап 13](README.md) · [13.18 Контрольная точка →](18-checkpoint.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/13-operating-systems/17-project-mini-container-runtime.md) <!-- nav -->

# Урок 13.17 — Проект: мини-среда выполнения контейнеров

> **Этап 13 — Операционные системы изнутри** · Урок 17 из 18 · ⏱ около 8 часов (4 занятия)

## Что вы узнаете

- Как пространства имён, новая корневая файловая система и cgroups вместе складываются в контейнер.
- Почему `pivot_root` — безопасный способ сменить корень и зачем нужен каждый шаг перед ним.
- Какие флаги пространств имён действуют на вызывающий процесс, а какие только на его потомков, и почему от этого зависит, кто вызывает `unshare`.
- Как надёжно убирать за собой объекты ядра с помощью `Drop` и как тестировать программу, которая работает только от root.

## Перед началом

- Вы прошли [Урок 13.15 — Пространства имён (namespaces)](15-namespaces.md) и [Урок 13.16 — Контрольные группы (cgroups)](16-cgroups.md) вместе с упражнениями: этот проект вырастает из `parse_namespaces` и `box` из 13.15 и `limit` из 13.16. Держите эти файлы открытыми.
- Вы знаете `fork`, `execvp` и `waitpid` ([Урок 13.5](05-fork-exec-and-wait.md)), формат проекта и `assert_cmd` из [Урока 13.8 — Проект: собственная командная оболочка](08-project-your-own-shell.md), а также `clap` и `anyhow` из [Урока 05.11](../05-rust-intermediate/11-practical-crates-serde-clap-errors.md).
- На вашем Mac работает Docker Desktop ([Урок 11.12](../11-linux-shell-and-containers/12-containers-and-docker-basics.md)). Он понадобится один раз — чтобы получить файлы маленькой системы Linux.
- Вы пишете код на Mac в `~/projects/rust/code/p13-minibox`, а собираете и запускаете его в ВМ `dev`, как в Уроке 13.1 (`rsync` папки в `~/p13-minibox`). Всё, что запускает контейнер, выполняется с `sudo` **только в ВМ** и никогда на вашем Mac.

## Новые слова

| Слово | Значение |
|---|---|
| **среда выполнения контейнеров** (container runtime) | Программа, которая на самом деле создаёт контейнер: настраивает пространства имён, корневую файловую систему и cgroups, затем запускает программу внутри. Docker использует для этого шага `runc`. |
| **корневая файловая система** (root file system, rootfs) | Дерево каталогов, которое контейнер видит как `/`: свои `/bin`, `/etc`, `/lib` и так далее, обычно взятые из образа. |
| **`docker export`** | Команда Docker, которая записывает все файлы контейнера в один tar-архив. |
| **tar-архив** (tar archive) | Один файл, который содержит много файлов и папок вместе с их владельцами и правами доступа; `tar -xf` распаковывает его. |
| **`chroot`** | Системный вызов, который меняет корневой каталог одного процесса. Старая файловая система остаётся смонтированной и достижимой. |
| **`pivot_root`** | Системный вызов, который меняет корневое монтирование всего пространства имён монтирования: новый каталог становится `/`, а старый корень после этого можно отмонтировать. |
| **bind-монтирование** (bind mount) | Сделать каталог видимым ещё в одном месте (`MS_BIND`). Здесь вы монтируете каталог на самого себя, чтобы превратить его в точку монтирования. |
| **ленивое отмонтирование** (lazy unmount, `MNT_DETACH`) | Монтирование сразу убирается из дерева; ядро освобождает его, когда им больше никто не пользуется. |
| **OCI** (Open Container Initiative, Открытая инициатива по контейнерам) | Объединение, которое пишет открытые стандарты для образов и сред выполнения контейнеров; `runc` следует её спецификации среды выполнения. |
| **обёртка запуска тестов** (Cargo `runner`) | Программа, которую Cargo ставит перед каждым тестом или бинарником, который запускает, например `sudo`. |
| **PID**, **UTS**, **OOM**, **CPU**, **ВМ** | Известны из прошлых уроков: Process ID (идентификатор процесса); UNIX Time-Sharing (пространство имён для имени хоста); Out Of Memory (нехватка памяти); Central Processing Unit (центральный процессор); виртуальная машина (ваша ВМ `dev`). |

## Проект

### Что вы создадите

`minibox` — среду выполнения контейнеров примерно в 300 строк на Rust без учёта тестов. Она запускает команду в новых
пространствах имён PID, UTS и монтирования, внутри собственной корневой файловой системы, со смонтированным `/proc`, внутри
cgroup с необязательными ограничениями памяти и CPU, возвращает код возврата команды и затем удаляет cgroup. Сеанс в ВМ:

```text
$ sudo ./target/debug/minibox run --rootfs ~/minibox/alpine --hostname box1 -- /bin/sh
/ # hostname
box1
/ # ps
PID   USER     TIME  COMMAND
    1 root      0:00 /bin/sh
    3 root      0:00 ps
/ # exit 3
$ echo $?
3
```

Командная оболочка внутри — от Alpine Linux, а не от вашей ВМ; она имеет PID 1; она видит только свои процессы. (Этот вывод и
все выводы ниже получены от эталонного решения, проверенного в привилегированном Linux-контейнере, настроенном как ВМ, с
именем хоста `lima-dev`. PID и время в вашей ВМ будут другими.) Полного решения вы не получите: ваши ориентиры — контракт,
вехи, небольшие подсказки и проверки «Готово, когда».

Работа делится между двумя процессами, как в упражнении `box`, но с новым корнем и cgroup:

```text
 minibox (parent, stays outside)                  child (becomes PID 1 of the container)
 ---------------------------------                ---------------------------------------
 parse arguments, check root and rootfs
 create /sys/fs/cgroup/minibox-<pid>, limits
 move itself into the group
 unshare(CLONE_NEWPID)
 fork  ------------------------------------------> unshare(CLONE_NEWUTS | CLONE_NEWNS)
 waitpid(child)                                    sethostname
    |                                              make all mounts private
    |                                              bind-mount rootfs onto itself
    |                                              chdir(rootfs), pivot_root(".", ".")
    |                                              umount2(".", MNT_DETACH), chdir("/")
    |                                              mount proc on /proc
    |                                              execvp(command)
    v  <------------------ exit status -----------
 exit code, OOM report, move out, remove group
```

### Контракт

**Пакет и файлы.** Бинарный пакет `code/p13-minibox` с именем `minibox` и небольшой библиотекой внутри:

```text
code/p13-minibox/
├── Cargo.toml
├── README.md          ← что делает minibox, как получить rootfs, собрать, запустить и протестировать
├── src/
│   ├── main.rs        ← командная строка (clap) → Config, сообщения об ошибках, код возврата
│   ├── lib.rs         ← pub mod parse; pub mod cgroup; pub mod container;
│   ├── parse.rs       ← parse_memory, parse_cpus, cpu_max, parse_namespaces, cgroup_path + модульные тесты
│   ├── cgroup.rs      ← struct Cgroup: create, enter, oom_kills; Drop выходит из группы и удаляет её
│   └── container.rs   ← run(): unshare, fork, подготовка в ребёнке, exec, ожидание
└── tests/
    └── cli.rs         ← интеграционные тесты; те, которым нужен root, помечены #[ignore]
```

Зависимости (версии на crates.io проверены 2026-11-18):

```toml
[dependencies]
anyhow = "1.0"
clap = { version = "4.6", features = ["derive"] }
nix = { version = "0.32", features = ["sched", "hostname", "process", "mount", "fs", "user"] }

[dev-dependencies]
assert_cmd = "2.2"
predicates = "3.1"
```

**Использование.**

```text
minibox run --rootfs <dir> [--memory <bytes>] [--cpus <n>] [--hostname <name>] -- <command> [args...]
```

| Параметр | Значение | По умолчанию |
|---|---|---|
| `--rootfs <dir>` | каталог, который становится `/` внутри; он должен существовать и содержать `bin/` | обязателен |
| `--memory <bytes>` | записывается в `memory.max`; просто байты или число с `K`, `M`, `G` (степени 1024); также ставит `memory.swap.max` в 0 | без ограничения |
| `--cpus <n>` | записывается в `cpu.max` как `n × 100000` микросекунд на период 100000; не меньше `0.01` | без ограничения |
| `--hostname <name>` | имя хоста в новом пространстве имён UTS | `minibox` |
| `-- <command> [args...]` | программа, которая запускается внутри как PID 1, ищется через `PATH` | обязательна |

**Поведение.**

- `minibox` должна запускаться от root; иначе она печатает `minibox: must be run as root (try sudo)` и завершается с кодом 125.
- Команда выполняется в новых пространствах имён PID, UTS и монтирования, как PID 1, с заданным именем хоста.
- Внутри `/` — это rootfs (через `pivot_root`), старый корень отмонтирован, а на `/proc` смонтирован свежий `proc`. После
  завершения `minibox` ни одно из этих монтирований не видно в ВМ.
- `minibox` всегда создаёт cgroup `/sys/fs/cgroup/minibox-<PID of minibox>`, даже без ограничений, и в конце удаляет её.
  Если OOM killer кого-то в ней убил, `minibox` печатает `minibox: the container was killed by the OOM killer`.

**Коды возврата и ошибки** (числа 125–127 следуют соглашениям `docker run` и командных оболочек):

| Ситуация | Сообщение в стандартный поток ошибок | Код |
|---|---|---|
| команда завершается с кодом N | — | N |
| команду убивает сигнал S | — | 128 + S (137 для SIGKILL) |
| команда не найдена | `minibox: NAME: command not found` | 127 |
| команда найдена, но не может быть запущена | `minibox: NAME: <error text>` | 126 |
| ошибка самой `minibox` (не root, плохой rootfs, неудачный системный вызов) | `minibox: <what failed>: <error>` | 125 |
| неверное использование (нет `--rootfs`, нет команды, плохой `--cpus`) | сообщение об ошибке от clap | 2 |

**Качество.** `cargo test` проходит от обычного пользователя, интеграционные тесты, которым нужен root, проходят при запуске
через `sudo`, `cargo clippy --all-targets` не выдаёт предупреждений, `cargo fmt --check` чист, и на каждую веху есть один
коммит Git.

### Пошаговый план

**Получите корневую файловую систему.** На **Mac** попросите Docker создать (но не запускать) контейнер из официального
образа Alpine Linux и выгрузить его файлы:

```bash
docker create --name course-alpine alpine:3.24
docker export -o ~/projects/rust/code/alpine-3.24-rootfs.tar course-alpine
docker rm course-alpine
```

Пример вывода (ваш ID другой; при первом запуске печатаются ещё сообщения о скачивании образа):

```text
4511a81975d6132c06bd6f2a878d472c46a52de2e2684dec99f5350608f8541c
course-alpine
```

В **ВМ** распакуйте архив с `sudo`, чтобы файлы принадлежали root, как в образе (передать файлы другому владельцу может только
root), а затем удалите архив на Mac:

```bash
mkdir -p ~/minibox/alpine
sudo tar -xf /Users/student/projects/rust/code/alpine-3.24-rootfs.tar -C ~/minibox/alpine
cat ~/minibox/alpine/etc/alpine-release
ls ~/minibox/alpine
```

```text
3.24.4
bin  etc   lib    mnt  proc  run   srv  tmp  var
dev  home  media  opt  root  sbin  sys  usr
```

(Замените `student` на имя пользователя вашего Mac, как в Уроке 13.1. Версия Alpine — та, на которую указывает `3.24` в день
скачивания.) Теперь создайте пакет на Mac:

```bash
cd ~/projects/rust
cargo new code/p13-minibox --name minibox
cd code/p13-minibox
cargo add nix@0.32 --features sched,hostname,process,mount,fs,user
cargo add clap@4.6 --features derive
cargo add anyhow@1.0
cargo add --dev assert_cmd@2.2 predicates@3.1
```

#### M1 — Командная строка, сначала тесты

**Цель.** `minibox run ...` разбирается в `Config`, и каждое правило из таблицы параметров проверено модульным тестом. Пока
`main` только печатает `Config`.

**Указания.**

1. В `parse.rs` напишите чистые функции и их тесты **до** кода `clap`. Предлагаемые сигнатуры:

   ```rust
   pub fn parse_memory(text: &str) -> Result<u64, String> {
       todo!()
   }

   pub fn parse_cpus(text: &str) -> Result<f64, String> {
       todo!()
   }

   pub fn cpu_max(cpus: f64) -> String {
       todo!()
   }
   ```

   `parse_memory("50M")` — это `Ok(52428800)`; `""`, `"M"`, `"-5M"`, `"0"` и число, которое не помещается в `u64`, — ошибки.
   `parse_cpus` отвергает всё меньше `0.01`, потому что ядро не принимает квоту меньше 1000 микросекунд;
   `cpu_max(0.5)` — это `"50000 100000"`.
2. Скопируйте `parse_namespaces` (Урок 13.15, упражнение 2) и `cgroup_path` (Урок 13.16, упражнение 2) в `parse.rs` вместе
   с их тестами. Обе функции понадобятся позже.
3. В `main.rs` опишите командную строку в derive-стиле `clap`: структура `Cli` с перечислением подкоманд, в котором один
   вариант — `Run(RunArgs)`. В `RunArgs` атрибут `#[arg(long, value_parser = parse_memory)]` заставляет `clap` вызвать вашу
   функцию, а `#[arg(last = true, required = true)] command: Vec<String>` собирает всё после `--`.
4. Поместите `Config` (rootfs, memory, cpus, hostname, command) в `container.rs` с `#[derive(Debug)]`.

<details>
<summary>Подсказка</summary>

Когда функция из `value_parser` возвращает `Err(text)`, `clap` печатает `error: invalid value '...' for '--cpus <CPUS>': text`
и завершается с кодом 2 — ровно как требует контракт. Для суффикса посмотрите на последний символ через `text.chars().last()`
и отрежьте его; умножайте через `checked_mul`, который при переполнении возвращает `None` вместо заворачивания (Урок 12.2).

</details>

**Готово, когда:**

```bash
cargo test -q
cargo build -q
./target/debug/minibox run --rootfs /tmp --memory 50M --cpus 0.5 -- /bin/echo hi
./target/debug/minibox run --rootfs /tmp --cpus 0 -- /bin/echo hi; echo "exit=$?"
```

показывает `test result: ok` для ваших модульных тестов (в эталонном решении их 6), а затем:

```text
Config { rootfs: "/tmp", memory: Some(52428800), cpus: Some(0.5), hostname: "minibox", command: ["/bin/echo", "hi"] }
error: invalid value '0' for '--cpus <CPUS>': --cpus must be at least 0.01

For more information, try '--help'.
exit=2
```

Коммит: `git commit -m "feat(p13-minibox): M1 command line and parsers"`.

#### M2 — Новые пространства имён и код возврата

**Цель.** `run` запускает команду в новых пространствах имён PID, UTS и монтирования и возвращает её код возврата. rootfs
пока только проверяется; команда всё ещё видит файлы ВМ.

**Указания.**

1. Сначала проверьте `nix::unistd::geteuid().is_root()`. Затем примените `fs::canonicalize` к rootfs (получится абсолютный
   путь и понятная ошибка, если каталога нет) и проверьте, что в нём есть каталог `bin`.
2. Соберите `CString` для `execvp` **до** `fork`, как в Уроке 13.5.
3. Получите флаги через `parse_namespaces("pid,uts,mnt")`. Родитель вызывает `unshare` **только** для `CLONE_NEWPID`
   (`flags & CloneFlags::CLONE_NEWPID`); ребёнок сразу после `fork` вызывает `unshare` для остальных
   (`flags - CloneFlags::CLONE_NEWPID`), затем `sethostname` и `execvp`.
4. Если `execvp` не удался, ребёнок печатает сообщение из контракта и завершается с кодом 127 для `ENOENT` или 126 для любой
   другой ошибки. Ребёнок должен уходить через `std::process::exit` и никогда — возвратом из `run`.
5. Родитель превращает `WaitStatus` в число: `Exited(_, code)` даёт `code`, `Signaled(_, signal, _)` даёт
   `128 + signal as i32`.
6. `main` возвращает `std::process::ExitCode`: `ExitCode::from(code as u8)` при успехе; при ошибке печатает
   `minibox: {e:#}` (`#` заставляет `anyhow` напечатать всю цепочку контекста в одну строку) и возвращает 125.

<details>
<summary>Подсказка</summary>

Зачем делить флаги? `CLONE_NEWPID` меняет пространство имён только для **потомков**, созданных после вызова (Урок 13.15,
раздел 4), а `CLONE_NEWUTS` и `CLONE_NEWNS` сразу переносят **вызывающий** процесс. Если бы родитель тоже взял себе новое
пространство имён монтирования, родитель и ребёнок делили бы его, и в M3 `pivot_root` сменил бы корень и родителю: тогда
родитель уже не нашёл бы `/sys/fs/cgroup`, чтобы убрать за собой. Почему ребёнку нельзя возвращаться? Он копия родителя,
включая значения на его стеке; возврат запустил бы их код `Drop` в ребёнке (в M4 это удалило бы cgroup).

</details>

**Готово, когда** (в ВМ):

```bash
cargo build -q
sudo ./target/debug/minibox run --rootfs ~/minibox/alpine --hostname box1 -- /bin/sh -c 'hostname; echo "my PID is $$"; exit 7'
echo "exit=$?"
hostname
sudo ./target/debug/minibox run --rootfs ~/minibox/alpine -- no-such-cmd; echo "exit=$?"
./target/debug/minibox run --rootfs ~/minibox/alpine -- true; echo "exit=$?"
```

```text
box1
my PID is 1
exit=7
lima-dev
minibox: no-such-cmd: command not found
exit=127
minibox: must be run as root (try sudo)
exit=125
```

Имя хоста изменилось только внутри, командная оболочка была PID 1, а её код возврата вышел из `minibox`. Коммит:
`feat(p13-minibox): M2 namespaces and exit code`.

#### M3 — Новая корневая файловая система и `/proc`

**Цель.** Внутри `/` — это rootfs Alpine, до файлов ВМ больше не добраться, а `ps` видит только контейнер.

**Указания.** В ребёнке, после `sethostname` и до `execvp`, в таком порядке:

1. **Сделайте все монтирования частными:** `mount(None, "/", None, MS_REC | MS_PRIVATE, None)`, как в Уроке 13.15. На
   машине с systemd `/` *общий* (shared), и без этого шага ваши монтирования появлялись бы в ВМ.
2. **Смонтируйте rootfs на самого себя (bind):** `mount(Some(rootfs), rootfs, None, MS_BIND | MS_REC, None)`. `pivot_root`
   принимает только новый корень, который является точкой монтирования; обычный каталог ею не является, а каталог,
   смонтированный на самого себя, — является.
3. **Поменяйте корни местами:** `chdir(rootfs)`, затем `pivot_root(".", ".")`, затем `umount2(".", MntFlags::MNT_DETACH)`,
   затем `chdir("/")`. Когда путь дважды один и тот же, старый корень кладётся *поверх* нового в том же месте, поэтому ленивое
   отмонтирование `.` убирает ровно старый корень. Вспомогательный каталог не нужен; этот приём показан в man-странице
   `pivot_root(2)`.
4. **Смонтируйте свежий `/proc`:** `mount(Some("proc"), "/proc", Some("proc"), MS_NOSUID | MS_NODEV | MS_NOEXEC, None)`.
   Три флага запрещают программы с set-user-ID, файлы устройств и запуск программ оттуда — так же делает Docker.

Почему не `chroot(rootfs)`? Это одна строка, и программа увидит файлы Alpine. Но `chroot` меняет только то, откуда начинаются
пути, начинающиеся с `/`, и только для **этого процесса**. Вся файловая система ВМ остаётся смонтированной в пространстве
имён, просто вне поля зрения, и процесс root внутри может вернуться к ней известным приёмом, описанным в man-странице
`chroot(2)` (второй `chroot` в подпапку, а затем `cd ..` за пределы нового корня). `pivot_root` вместе с отмонтированием
действительно убирает старый корень из таблицы монтирования контейнера.

<details>
<summary>Подсказка</summary>

Дайте каждому шагу свой `.context("...")` (например, `.context("pivot_root")`): когда что-то упадёт, сообщение скажет, какой
это был шаг. `EINVAL` от `pivot_root` обычно означает забытый шаг 1, `EBUSY` — забытый шаг 2. Чтобы увидеть то, что видит
контейнер, выполните внутри `cat /proc/self/mountinfo`.

</details>

**Готово, когда:**

```bash
sudo ./target/debug/minibox run --rootfs ~/minibox/alpine -- /bin/sh -c 'cat /etc/alpine-release; echo /*; ps; wc -l < /proc/self/mountinfo'
findmnt | grep alpine || echo "no container mounts in the VM"
```

```text
3.24.4
/bin /dev /etc /home /lib /media /mnt /opt /proc /root /run /sbin /srv /sys /tmp /usr /var
PID   USER     TIME  COMMAND
    1 root      0:00 /bin/sh -c cat /etc/alpine-release; echo /*; ps; wc -l < /proc/self/mountinfo
    3 root      0:00 ps
2
no container mounts in the VM
```

У контейнера ровно два монтирования: его корень и его `/proc`. PID 2 был у `cat`, который уже завершился. Коммит:
`feat(p13-minibox): M3 root file system and /proc`.

#### M4 — Ограничения cgroup и уборка

**Цель.** Программа `limit` из Урока 13.16 становится типом: `Cgroup` создаёт группу с ограничениями, а её `Drop` выводит
`minibox` обратно из группы и удаляет её — на любом пути выхода из `run`.

**Указания.**

1. `Cgroup::create(memory, cpus)`: **сначала** прочитайте `/proc/self/cgroup` и сохраните путь исходной группы (с помощью
   вашей `cgroup_path`); создайте `/sys/fs/cgroup/minibox-<process::id()>`; запишите `memory.max` и `memory.swap.max` = `0`,
   если есть ограничение памяти, и `cpu.max` (с помощью вашей `cpu_max`), если есть ограничение CPU.
2. `enter()` записывает собственный PID `minibox` в `cgroup.procs`. Вызывайте её до `unshare` и `fork`: тогда ребёнок рождается
   в группе, и каждый процесс, который он запускает, остаётся там же.
3. После `waitpid` прочитайте `memory.events`; если `oom_kill` больше 0, напечатайте сообщение из контракта.
4. `impl Drop for Cgroup`: запишите PID в `cgroup.procs` **исходной** группы, затем удалите группу через `fs::remove_dir`.
   `Drop` не может вернуть ошибку и не должен паниковать, поэтому, если удаление не удалось, напечатайте сообщение.

<details>
<summary>Подсказка</summary>

Создайте `Cgroup` как локальную переменную в `run` и явно уничтожьте её после отчёта (`drop(cgroup)`). Благодаря `Drop` ранний
возврат через `?` между `create` и `fork` тоже убирает за собой. После `waitpid` группа пуста: когда PID 1 пространства имён
завершается, ядро убивает все остальные процессы в нём и ждёт, пока они исчезнут (Урок 13.15).

</details>

**Готово, когда:**

```bash
R=~/minibox/alpine
sudo ./target/debug/minibox run --rootfs $R --memory 20M -- awk 'BEGIN { s = "x"; for (i = 0; i < 27; i++) s = s s; print length(s) }'; echo "exit=$?"
sudo ./target/debug/minibox run --rootfs $R -- awk 'BEGIN { s = "x"; for (i = 0; i < 27; i++) s = s s; print length(s) }'; echo "exit=$?"
sudo ./target/debug/minibox run --rootfs $R -- time awk 'BEGIN { for (i = 0; i < 5000000; i++) x++ }'
sudo ./target/debug/minibox run --rootfs $R --cpus 0.5 -- time awk 'BEGIN { for (i = 0; i < 5000000; i++) x++ }'
sudo ./target/debug/minibox run --rootfs $R -- cat /proc/self/cgroup
ls /sys/fs/cgroup | grep minibox || echo "no minibox groups left"
```

```text
minibox: the container was killed by the OOM killer
exit=137
134217728
exit=0
real	0m 1.07s
user	0m 1.07s
sys	0m 0.00s
real	0m 2.24s
user	0m 1.13s
sys	0m 0.00s
0::/minibox-3077
no minibox groups left
```

**Что происходит:**

1. Первая программа `awk` (из Alpine, из rootfs) удваивает строку 27 раз, до 128 MiB (мебибайт; 1 MiB =
   1024 × 1024 байт). С ограничением 20 MiB OOM killer убивает её внутри группы: код 137, и `minibox` сообщает об этом.
   Без ограничения она печатает длину.
2. `time` (тоже из Alpine) запускает цикл, который просто занимает процессор. Без ограничения `real` (время по часам) равно
   `user` (времени CPU). С `--cpus 0.5` та же работа CPU занимает вдвое больше времени, потому что группу притормаживают
   (троттлинг) на половину каждого периода в 100 мс.
3. `/proc/self/cgroup` внутри показывает группу, которую создала `minibox`. После этого группы уже нет.

Коммит: `feat(p13-minibox): M4 cgroup limits and clean-up`.

#### M5 — Интеграционные тесты, линтер и README

**Цель.** Проект, который можно показать: тесты для каждой строки таблицы кодов возврата, никаких предупреждений линтера,
README.

**Указания.** Большинству тестов нужны root и rootfs. Пометьте их `#[ignore = "..."]`, чтобы обычный `cargo test` их
пропускал, а путь к rootfs берите из переменной окружения. Образец для `tests/cli.rs`:

```rust
use assert_cmd::cargo::cargo_bin_cmd;

/// Распакованная корневая файловая система Alpine из переменной окружения MINIBOX_ROOTFS.
fn rootfs() -> String {
    std::env::var("MINIBOX_ROOTFS").expect("set MINIBOX_ROOTFS to an extracted root file system")
}

#[test]
fn a_command_is_required() {
    cargo_bin_cmd!("minibox")
        .args(["run", "--rootfs", "/tmp"])
        .assert()
        .code(2);
}

#[test]
#[ignore = "needs root and MINIBOX_ROOTFS"]
fn passes_on_the_exit_code() {
    cargo_bin_cmd!("minibox")
        .args(["run", "--rootfs", &rootfs(), "--"])
        .args(["/bin/sh", "-c", "exit 7"])
        .assert()
        .code(7);
}
```

Добавьте как минимум: `--cpus 0` даёт 2; запуск не от root даёт 125 с точным сообщением (выходите из этого теста сразу, если
`geteuid().is_root()`); имя хоста, PID 1 и `/etc/alpine-release` внутри; неизвестная команда даёт 127; ограничение памяти
даёт 137, **и** группа после этого удалена.

Как тесты для root получают права root? `cargo test` нельзя запускать от root (у root нет тулчейна Rust, а файлы сборки
принадлежали бы root). Вместо этого Cargo умеет ставить **обёртку запуска** (runner) перед каждым тестовым бинарником, который
запускает. Это настройка `target.<target triple>.runner`, доступная и как переменная окружения; целевая платформа (target
triple) вашей ВМ — `aarch64-unknown-linux-gnu` (её показывает `rustc -vV | grep host`). `sudo --preserve-env=MINIBOX_ROOTFS`
сохраняет эту одну переменную, которую `sudo` иначе выбросил бы. Запишите обе команды в свой README.

<details>
<summary>Подсказка</summary>

Для теста уборки пусть контейнер напечатает свою группу (`cat /proc/self/cgroup` перед пожирателем памяти, с `exec` перед
`awk`), отрежьте `0::/` и проверьте, что `/sys/fs/cgroup/<that name>` не существует после возврата из `minibox`. Тесты
выполняются в параллельных потоках с одним общим rootfs: это полезно, потому что так находятся ошибки, которые проявляются,
только когда два контейнера стартуют одновременно (см. «Частые ошибки»).

</details>

**Готово, когда:**

```bash
cargo fmt --check && cargo clippy -q --all-targets && cargo test -q
export MINIBOX_ROOTFS=~/minibox/alpine
CARGO_TARGET_AARCH64_UNKNOWN_LINUX_GNU_RUNNER='sudo --preserve-env=MINIBOX_ROOTFS' cargo test -q -- --include-ignored
ls /sys/fs/cgroup | grep minibox || echo "no minibox groups left"
```

печатает только результаты тестов, все `ok`, и ни одной группы не осталось. В эталонном решении 6 модульных и 7
интеграционных тестов; пустые блоки для `main.rs` и doc-тестов здесь опущены:

```text
running 6 tests
......
test result: ok. 6 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s

running 7 tests
iiii...
test result: ok. 3 passed; 0 failed; 4 ignored; 0 measured; 0 filtered out; finished in 0.00s

running 6 tests
......
test result: ok. 6 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s

running 7 tests
.......
test result: ok. 7 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.02s
no minibox groups left
```

Каждая `i` — пропущенный (ignored) тест. Коммит: `feat(p13-minibox): M5 tests, lint and README`.

### Дополнительные цели

- **Минимальный `/dev`.** В выгруженном rootfs каталог `/dev` почти пуст, поэтому `sleep 30 &` внутри печатает
  `can't open '/dev/null'`. До `pivot_root` смонтируйте `tmpfs` на `rootfs/dev` и bind-смонтируйте туда `null`, `zero`,
  `random`, `urandom` и `tty` из `/dev` ВМ поверх пустых файлов.
- **`--pids <n>`** для `pids.max` (Урок 13.16), и проверьте, что fork-бомба внутри останавливается на ограничении.
- **`--namespaces <list>`** с вашей `parse_namespaces`, например с добавлением `net` (тогда `ip link` внутри показывает только
  `lo`) или `cgroup` (тогда `/proc/self/cgroup` внутри показывает `0::/`).
- **Сигналы и остатки.** Передавайте контейнеру `SIGINT` и `SIGTERM`, пришедшие к `minibox`, а при старте удаляйте группы
  `minibox-*`, у которых процесса `minibox` уже не существует.
- **Пространство имён пользователей** (`CLONE_NEWUSER`): родитель записывает `/proc/<child>/uid_map` и `gid_map` (например,
  `0 1000 1` — «root внутри — это пользователь 1000 снаружи»), пока ребёнок ждёт на конвейере; после этого root внутри не имеет
  власти над файлами ВМ. На этом построены контейнеры без root (rootless).
- **Чистое окружение** через `execvpe` (только `PATH`, `HOME`, `TERM`) и монтирование `overlay`, чтобы сам rootfs никогда не
  менялся контейнером.

## Частые ошибки

> ⚠️ **Частая ошибка:** родитель вызывает `unshare` для всех трёх пространств имён. Контейнер работает, но в конце:
>
> ```text
> minibox: cannot remove /sys/fs/cgroup/minibox-3919: No such file or directory (os error 2)
> ```
>
> Родитель и ребёнок делили пространство имён монтирования, поэтому `pivot_root` сменил корень и родителю, и родитель искал
> `/sys/fs/cgroup` внутри Alpine. Группа остаётся (удалите её через `sudo rmdir`). *Исправление:* родитель вызывает `unshare`
> только для `CLONE_NEWPID`.

> ⚠️ **Частая ошибка:** `minibox: pivot_root: EBUSY: Device or resource busy`. rootfs — обычный каталог, а не точка
> монтирования. *Исправление:* сначала смонтируйте его на самого себя (bind).

> ⚠️ **Частая ошибка:** `minibox: pivot_root: EINVAL: Invalid argument`, а после этого `findmnt | grep alpine` в ВМ
> показывает монтирование, которое там никто не делал. Без `MS_PRIVATE` bind-монтирование попало в пространство имён ВМ через
> общее распространение (shared propagation), а `pivot_root` отказывается работать с общими монтированиями.
> *Исправление:* сначала сделайте `/` частным; остаток уберите через `sudo umount ~/minibox/alpine`.

> ⚠️ **Частая ошибка:** использовать для старого корня постоянный вспомогательный каталог, например `rootfs/.oldroot`, и
> удалять его в конце. Поодиночке это работает, но параллельные интеграционные тесты иногда падают с
> `minibox: unmount old root: EINVAL: Invalid argument`: все контейнеры делят один rootfs, и один контейнер удаляет каталог,
> пока другой ещё им пользуется. *Исправление:* `pivot_root(".", ".")` не нужен никакой каталог.

> ⚠️ **Частая ошибка:** `sudo cargo run ...` печатает `sudo: cargo: command not found`, потому что в `PATH` пользователя root
> нет тулчейна Rust. *Исправление:* собирайте от своего имени и запускайте `sudo ./target/debug/minibox ...` или используйте
> обёртку запуска из M5.

## Вопросы для самопроверки

1. Почему родитель вызывает `unshare` только для `CLONE_NEWPID`, а ребёнок — для UTS и монтирования?

<details>
<summary>Ответ</summary>

`CLONE_NEWPID` действует только на потомков, созданных после вызова, поэтому его должен сделать родитель до `fork`.
Пространства имён UTS и монтирования сразу переносят вызывающий процесс; если бы это сделал родитель, он делил бы с ребёнком
пространство имён монтирования и потерял бы свой корень при `pivot_root`.

</details>

2. Какие три условия нужны `pivot_root` и какой шаг `minibox` обеспечивает каждое?

<details>
<summary>Ответ</summary>

Новый корень должен быть точкой монтирования (смонтировать rootfs на самого себя), участвующие монтирования не должны быть
общими (сделать `/` частным через `MS_REC | MS_PRIVATE`), а старый корень должен оказаться где-то под новым корнем (с `"."`
дважды он кладётся поверх нового корня и затем лениво отмонтируется).

</details>

3. Почему `chroot` недостаточно для контейнера?

<details>
<summary>Ответ</summary>

Он меняет только то, откуда один процесс начинает пути, начинающиеся с `/`. Старая файловая система остаётся смонтированной в
пространстве имён, и процесс root может сбежать через второй `chroot` и `cd ..`. `pivot_root` с отмонтированием убирает
старый корень из пространства имён монтирования контейнера.

</details>

4. Контейнер завершился с кодом 137. Что произошло и как `minibox` может узнать, что виновато ограничение памяти?

<details>
<summary>Ответ</summary>

137 = 128 + 9: команду убил SIGKILL. Если в `memory.events` группы `oom_kill` больше 0, это сделал OOM killer из-за
`memory.max`.

</details>

5. Почему уборка находится в `Drop` и почему порождённому через `fork` ребёнку нельзя возвращаться из `run`?

<details>
<summary>Ответ</summary>

`Drop` выполняется на любом пути выхода из функции, включая ранний возврат через `?`, поэтому группа удаляется всегда. Ребёнок —
копия родителя с теми же значениями на стеке; возврат запустил бы их `Drop` и в ребёнке. Он завершается через
`process::exit` или `exec`, которые не запускают деструкторы.

</details>

6. Почему большинство интеграционных тестов помечены `#[ignore]` и как их запустить?

<details>
<summary>Ответ</summary>

Им нужны root и распакованный rootfs, которых у обычного `cargo test` нет. Запускайте их с обёрткой запуска Cargo:
`CARGO_TARGET_AARCH64_UNKNOWN_LINUX_GNU_RUNNER='sudo --preserve-env=MINIBOX_ROOTFS' cargo test -- --include-ignored`. Cargo
собирает от вашего имени и запускает каждый тестовый бинарник через `sudo`.

</details>

## Итоги

- Контейнер — обычный процесс: новые пространства имён для стен, новая корневая файловая система для файлов, cgroup для бюджета.
- `CLONE_NEWPID` действует на потомков; пространства имён UTS и монтирования — на вызывающий процесс. Поэтому родитель делает `unshare` для PID, а ребёнок — всё остальное.
- Безопасный путь к новому корню: сделать монтирования частными, смонтировать rootfs на самого себя, `pivot_root(".", ".")`, лениво отмонтировать старый корень, смонтировать `/proc`.
- `chroot` меняет взгляд только одного процесса; старая файловая система остаётся достижимой.
- guard-объект для cgroup с `Drop` удаляет группу на любом пути; порождённый ребёнок уходит через `exec` или `process::exit`, а не возвратом.
- Коды возврата рассказывают историю: N от команды, 128 + сигнал, 127 — не найдена, 126 — не запускается, 125 — собственная ошибка среды выполнения.
- Тесты, которым нужен root, помечены `#[ignore]` и запускаются через обёртку запуска Cargo с `sudo`; параллельный запуск находит гонки.

## Источники и что почитать

- [pivot_root(2) — Linux manual page](https://man7.org/linux/man-pages/man2/pivot_root.2.html) — правила для нового корня и последовательность `pivot_root(".", ".")`.
- [chroot(2) — Linux manual page](https://man7.org/linux/man-pages/man2/chroot.2.html) — что `chroot` меняет и чего не меняет.
- [mount_namespaces(7) — Linux manual page](https://man7.org/linux/man-pages/man7/mount_namespaces.7.html) — общее и частное распространение монтирования, причина для `MS_PRIVATE`.
- [The Linux kernel documentation — Control Group v2](https://docs.kernel.org/admin-guide/cgroup-v2.html) — `memory.max`, `cpu.max`, `memory.events` и `cgroup.procs`.
- [OCI Runtime Specification](https://github.com/opencontainers/runtime-spec) — что должна делать настоящая среда выполнения вроде `runc`; сравните с `minibox`.
- [The Cargo Book — Configuration](https://doc.rust-lang.org/cargo/reference/config.html) — см. `target.<triple>.runner`, настройку для тестов, которым нужен root.
- [docker container export](https://docs.docker.com/reference/cli/docker/container/export/) — команда, которая сделала rootfs.

---

[🏠 Начало](../START-HERE.md) · [← 13.16 Контрольные группы (cgroups)](16-cgroups.md) · [📋 Этап 13](README.md) · [13.18 Контрольная точка →](18-checkpoint.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
