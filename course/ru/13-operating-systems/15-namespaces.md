<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 13.14 io_uring](14-io-uring.md) · [📋 Этап 13](README.md) · [13.16 Контрольные группы (cgroups) →](16-cgroups.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/13-operating-systems/15-namespaces.md) <!-- nav -->

# Урок 13.15 — Пространства имён (namespaces)

> **Этап 13 — Операционные системы изнутри** · Урок 15 из 18 · ⏱ около 135 минут

## Что вы узнаете

- Что такое пространство имён (namespace) в Linux и какие виды бывают.
- Как увидеть пространства имён процесса в `/proc/<pid>/ns`.
- Как создавать новые пространства имён UTS, PID и монтирования командой `unshare` и из Rust через `nix`.
- Почему первый процесс в пространстве имён PID особенный.
- Как пространства имён служат «стенами» любого контейнера и чего они *не* делают.

## Перед началом

- Вы прошли [Урок 13.14 — io_uring](14-io-uring.md).
- Вы знаете `fork` и `waitpid` ([Урок 13.5](05-fork-exec-and-wait.md)), сигналы ([Урок 13.7](07-signals-in-your-programs.md)), монтирование и `/proc` ([Урок 13.12](12-file-systems-inside.md)) и что такое контейнер снаружи ([Урок 11.12](../11-linux-shell-and-containers/12-containers-and-docker-basics.md)).
- Создайте пакет на Mac и скопируйте его в ВМ, как в Уроке 13.10:

```bash
cd ~/projects/rust
cargo new code/p13-ns
cd code/p13-ns
cargo add nix@0.32 --features sched,hostname,process,mount,fs
rm src/main.rs && mkdir src/bin
```

Для создания большинства пространств имён нужны права root, поэтому собранные программы вы запускаете через
`sudo` **в ВМ**, никогда на своём Mac. Выводы ниже получены в тестовом контейнере с именем хоста
`course-ns-demo`; ваша ВМ называется `lima-dev`.

## Новые слова

| Слово | Значение |
|---|---|
| **пространство имён** (namespace) | Возможность ядра, которая даёт группе процессов собственный приватный вид одного глобального ресурса, например имени хоста, списка процессов или таблицы монтирования. |
| **пространство имён UTS** (UNIX Time-Sharing) | Изолирует имя хоста и имя домена. Странное название пришло от старой структуры ядра `utsname`. |
| **пространство имён PID** (Process ID, идентификатор процесса) | Даёт процессам собственную нумерацию. Первый процесс внутри получает PID 1. |
| **пространство имён монтирования** (mount namespace) | Даёт процессам собственную копию таблицы монтирования, поэтому монтирования внутри не видны снаружи. |
| **сетевое пространство имён** (network namespace) | Даёт процессам собственные сетевые интерфейсы, адреса и маршруты. |
| **пространство имён пользователей** (user namespace) | Отображает идентификаторы пользователей: процесс может быть «root» внутри, оставаясь обычным пользователем снаружи. |
| **`unshare`** | Системный вызов (и команда), который переносит вызывающий процесс в новые пространства имён. |
| **процесс init** (init process) | PID 1: первый процесс системы или пространства имён PID. Он усыновляет сирот и защищён от большинства сигналов. |
| **распространение монтирования** (mount propagation) | Правило, которое решает, появится ли монтирование, сделанное в одном пространстве имён монтирования, в других. `private` означает «нет». |
| **привилегия** (capability) | Часть власти root, например `CAP_SYS_ADMIN` для монтирования. Она нужна для создания пространств имён (кроме пространства имён пользователей). |

## 1. Что такое пространство имён

В вашей ВМ одно ядро, один список процессов и одно имя хоста. **Пространство имён** оборачивает один из
таких глобальных ресурсов так, что группа процессов видит **свою собственную приватную копию**. Внутри всё
выглядит обычно; снаружи ничего не меняется.

Аналогия: большой офисный центр (ядро) со множеством компаний. У каждой компании своя приёмная, своя нумерация
комнат и свой телефонный справочник. Сотрудники компании A видят «комнату 1», «комнату 2»…; комнат компании B
они не видят вовсе, хотя все комнаты в одном здании.

В Linux восемь видов:

| Пространство имён | Флаг | Что изолирует |
|---|---|---|
| монтирования | `CLONE_NEWNS` | таблицу монтирования: какие файловые системы где смонтированы |
| UTS | `CLONE_NEWUTS` | имя хоста и имя домена |
| IPC (Inter-Process Communication, межпроцессное взаимодействие) | `CLONE_NEWIPC` | общую память и очереди сообщений System V (старые способы Unix, которыми процессы обмениваются данными) |
| PID | `CLONE_NEWPID` | номера процессов |
| сетевое | `CLONE_NEWNET` | сетевые интерфейсы, IP-адреса (IP, Internet Protocol, межсетевой протокол), маршруты, порты |
| пользователей | `CLONE_NEWUSER` | идентификаторы пользователей и групп |
| cgroup | `CLONE_NEWCGROUP` | вид дерева cgroup (Урок 13.16) |
| времени | `CLONE_NEWTIME` | некоторые часы (время с момента загрузки) |

(Пространство имён монтирования было первым, с 2002 года, поэтому его флаг получил общее имя `NEWNS`.)

Каждый процесс принадлежит ровно одному пространству имён каждого вида. Ядро показывает их как символические
ссылки в `/proc/<pid>/ns`:

```bash
ls -l /proc/self/ns | awk 'NR>1 {print $9, $10, $11}'
```

```text
cgroup -> cgroup:[4026533114]
ipc -> ipc:[4026533110]
mnt -> mnt:[4026533108]
net -> net:[4026533115]
pid -> pid:[4026533113]
pid_for_children -> pid:[4026533113]
time -> time:[4026533243]
time_for_children -> time:[4026533243]
user -> user:[4026531837]
uts -> uts:[4026533109]
```

Число в скобках — номер inode, который определяет пространство имён. Два процесса находятся в одном
пространстве имён ровно тогда, когда числа равны. (`awk 'NR>1 {...}'` пропускает первую строку `ls -l` и
печатает три столбца.)

## 2. Пробуем командой `unshare`

Команда `unshare` (из util-linux, уже установлена) запускает программу в новых пространствах имён. В ВМ:

```bash
hostname
sudo unshare --uts sh -c 'hostname box2; hostname'
hostname
```

```text
course-ns-demo
box2
course-ns-demo
```

Оболочка внутри получила своё пространство имён UTS. Смена имени хоста там не изменила его снаружи.

Теперь пространство имён PID. `--fork` запускает программу как потомка (почему — объясняет раздел 4), а
`--mount-proc` монтирует свежий `/proc`, чтобы `ps` видел новую нумерацию:

```bash
sudo unshare --pid --fork --mount-proc sh -c 'echo "PID of this shell: $$"; ps -o pid,comm'
```

```text
PID of this shell: 1
    PID COMMAND
      1 sh
      2 ps
```

Оболочка считает себя PID 1 — первым процессом системы — и видит только себя и `ps`. Это та же картина, что и
при `docker run ... ps`.

## 3. Пространство имён UTS из Rust

Системный вызов `unshare(flags)` переносит **вызывающий процесс** в новые пространства имён указанных видов.
Создайте `src/bin/uts.rs`:

```rust
use nix::sched::{CloneFlags, unshare};
use nix::unistd::{gethostname, sethostname};

fn main() -> nix::Result<()> {
    println!("before: hostname = {:?}", gethostname()?);

    unshare(CloneFlags::CLONE_NEWUTS)?; // from now on this process has its own UTS namespace
    sethostname("inside-box")?;
    println!("after : hostname = {:?}", gethostname()?);

    let ns = std::fs::read_link("/proc/self/ns/uts").expect("read_link failed");
    println!("my UTS namespace: {ns:?}");
    Ok(())
}
```

Соберите его в `~/p13-ns`, затем запустите бинарник через `sudo` и посмотрите на имя хоста после этого:

```bash
cargo build
sudo ./target/debug/uts
hostname
readlink /proc/self/ns/uts
```

```text
before: hostname = "course-ns-demo"
after : hostname = "inside-box"
my UTS namespace: "uts:[4026533379]"
course-ns-demo
uts:[4026533109]
```

**Что происходит:**

1. `main` возвращает `nix::Result<()>`, поэтому `?` работает с каждым вызовом `nix`, а ошибка печатается как
   `Error: EPERM` или похоже.
2. `unshare(CLONE_NEWUTS)` создаёт новое пространство имён UTS — копию старого — и переносит в него этот
   процесс.
3. `sethostname` меняет имя хоста **только в новом пространстве имён**.
4. Номер пространства имён программы (`4026533379`) отличается от номера оболочки (`4026533109`). Когда
   программа завершается, последний участник пространства имён исчезает, и ядро удаляет его.

Мы запускаем бинарник напрямую, а не через `sudo cargo run`, потому что у `root` обычно нет тулчейна Rust в
`PATH`.

## 4. Пространство имён PID и особенный PID 1

`unshare(CLONE_NEWPID)` работает иначе: вызывающий процесс **остаётся** в старом пространстве имён PID (его
PID не может измениться, пока он работает). Внутри нового рождаются только **потомки**, созданные после этого,
и первый потомок становится его PID 1. Поэтому команде и понадобился `--fork`. Создайте `src/bin/pidns.rs`:

```rust
use nix::sched::{CloneFlags, unshare};
use nix::sys::wait::waitpid;
use nix::unistd::{ForkResult, fork, getpid, getppid};

fn main() -> nix::Result<()> {
    println!("parent: my PID is {}", getpid());
    unshare(CloneFlags::CLONE_NEWPID)?; // affects the CHILDREN created from now on

    // SAFETY: single-threaded program.
    match unsafe { fork() }? {
        ForkResult::Child => {
            println!(
                "child : my PID is {}, my parent's PID is {}",
                getpid(),
                getppid()
            );
            std::process::exit(0);
        }
        ForkResult::Parent { child } => {
            println!("parent: the child's PID is {child}");
            waitpid(child, None)?;
        }
    }
    Ok(())
}
```

`sudo ./target/debug/pidns`:

```text
parent: my PID is 234
parent: the child's PID is 235
child : my PID is 1, my parent's PID is 0
```

**Что происходит:**

1. Родитель — PID 234 во внешнем пространстве имён, и таким он и остаётся.
2. `fork` создаёт потомка внутри нового пространства имён PID. У **одного и того же процесса** два номера:
   235, если смотреть снаружи, и 1, если смотреть изнутри. Пространства имён PID вложены: внешнее видит всё,
   внутреннее — только свои процессы.
3. Родитель потомка живёт вне его пространства имён, поэтому внутри для него нет номера: `getppid()`
   возвращает 0.

**Почему PID 1 особенный.** В каждом пространстве имён PID процесс PID 1 играет роль *процесса init*:

- **Он усыновляет сирот.** Когда процесс умирает, его потомки получают нового родителя — PID 1, и PID 1 должен
  ждать их через `wait`, иначе они останутся зомби (Урок 13.5).
- **Сигналы фильтруются.** Процессы внутри пространства имён не могут убить PID 1 сигналом, для которого
  PID 1 не установил обработчик, даже через SIGKILL (всегда проходят только SIGKILL или SIGSTOP, отправленные
  *снаружи*, из родительского пространства имён):

  ```bash
  sudo unshare --pid --fork --mount-proc sh -c 'kill -TERM 1; echo still alive; kill -KILL 1; echo still alive after KILL'
  ```

  ```text
  still alive
  still alive after KILL
  ```

- **Когда PID 1 завершается, ядро убивает все остальные процессы пространства имён.**

Это объясняет классическую проблему контейнеров: `docker stop` посылает SIGTERM процессу PID 1 контейнера.
Если ваша программа работает как PID 1 и не обрабатывает SIGTERM ([Урок 13.7](07-signals-in-your-programs.md)),
сигнал игнорируется, и через 10 секунд Docker убивает её через SIGKILL. Именно поэтому существуют маленькие
программы init вроде `tini` (параметр Docker `--init`).

## 5. Пространство имён монтирования

Пространство имён монтирования даёт процессу собственную копию таблицы монтирования. Важна одна деталь: в
большинстве систем корневое монтирование **общее** (shared), то есть монтирования распространяются между
пространствами имён (так его настраивает systemd). Новое пространство имён монтирования начинается как копия,
которая всё ещё связана с оригиналом. Поэтому первым делом внутри нужно сделать все монтирования
**приватными**. Создайте `src/bin/mntns.rs`:

```rust
use std::fs;
use std::process::Command;

use nix::mount::{MsFlags, mount};
use nix::sched::{CloneFlags, unshare};

fn main() -> nix::Result<()> {
    unshare(CloneFlags::CLONE_NEWNS)?; // our own copy of the mount table

    // Stop mount events from travelling back to the parent namespace.
    mount(
        None::<&str>,
        "/",
        None::<&str>,
        MsFlags::MS_REC | MsFlags::MS_PRIVATE,
        None::<&str>,
    )?;

    fs::create_dir_all("/mnt/private").expect("mkdir failed");
    mount(
        Some("tmpfs"),
        "/mnt/private",
        Some("tmpfs"),
        MsFlags::empty(),
        Some("size=1m"),
    )?;
    fs::write("/mnt/private/note.txt", "only visible in here\n").expect("write failed");

    let out = Command::new("findmnt")
        .args(["-n", "/mnt/private"])
        .output()
        .expect("findmnt");
    print!("inside : {}", String::from_utf8_lossy(&out.stdout));
    print!(
        "inside : note.txt says {}",
        fs::read_to_string("/mnt/private/note.txt").unwrap()
    );
    Ok(())
}
```

```bash
sudo ./target/debug/mntns
findmnt -n /mnt/private || echo 'not mounted'
ls /mnt/private
```

```text
inside : /mnt/private tmpfs tmpfs rw,relatime,size=1024k
inside : note.txt says only visible in here
not mounted
```

**Что происходит:**

1. `unshare(CLONE_NEWNS)` копирует таблицу монтирования для этого процесса.
2. `mount(None, "/", None, MS_REC | MS_PRIVATE, None)` ничего нового не монтирует. Он меняет
   *распространение* для `/` и всего, что ниже (`MS_REC`, рекурсивно), на приватное.
3. Второй `mount` монтирует **tmpfs** (файловую систему в RAM) размером 1 МиБ в `/mnt/private`, и мы пишем туда
   файл. Дочерний процесс `findmnt` наследует наше пространство имён и видит это монтирование.
4. После завершения программы в пространстве имён оболочки на `/mnt/private` ничего не смонтировано, и `ls`
   показывает пустой каталог. tmpfs и её файл исчезли вместе с пространством имён.

Так контейнер получает свою корневую файловую систему: новое пространство имён монтирования, затем другой
корень (`pivot_root` или `chroot`), затем свежий `/proc`. Всё это вы сделаете в проекте,
[Урок 13.17](17-project-mini-container-runtime.md).

## 6. Сетевое пространство имён и пространство имён пользователей, кратко

**Сетевое пространство имён.** В новом есть только интерфейс обратной петли (loopback), и он выключен:

```bash
sudo unshare --net ip -brief link
```

Первая строка вывода:

```text
lo               DOWN           00:00:00:00:00:00 <LOOPBACK>
```

(На тестовой машине появились ещё несколько туннельных устройств вроде `tunl0`, созданных модулями ядра; ваша
ВМ может показать только `lo`.) Затем среды выполнения контейнеров создают пару виртуальных интерфейсов: один
конец внутри, другой — на мосту снаружи, чтобы дать контейнеру сеть. В Уроке 15.2 (Этап 15)
вы узнаете об интерфейсах, адресах и маршрутах, из которых строится такая схема.

**Пространство имён пользователей.** Внутри пространства имён пользователей процесс может иметь
идентификатор пользователя 0, root, оставаясь снаружи обычным пользователем. Это единственное пространство
имён, которое обычный пользователь может создать без `sudo`:

```bash
unshare --user --map-root-user sh -c id
```

На тестовой машине, при запуске от обычного пользователя, вывод был таким:

```text
uid=0(root) gid=0(root) groups=0(root)
```

«Root» внутри пространства имён пользователей имеет власть только над тем, что принадлежит этому пространству
имён; он не может, например, прочитать чужие файлы. На этом построены контейнеры без root (Podman, rootless
Docker). Некоторые дистрибутивы это ограничивают: новые версии Ubuntu используют правила AppArmor, из-за
которых эта команда может завершиться ошибкой или новый «root» получит меньше прав.

> 🔒 **Безопасность:** пространства имён — это стены, а не полная граница безопасности. Все контейнеры делят
> **одно ядро**, поэтому ошибка в ядре может пробить стены. Настоящие среды выполнения добавляют слои:
> отбрасывают привилегии, фильтруют системные вызовы через seccomp (Урок 13.14) и ограничивают ресурсы через
> cgroups (Урок 13.16). Процесс, который является root во внешнем пространстве имён и работает в контейнере с
> `--privileged`, почти root на хосте.

## Частые ошибки

1. **Запуск без root.** От обычного пользователя `./target/debug/uts` печатает

   ```text
   before: hostname = "613981bb33ae"
   Error: EPERM
   ```

   потому что для создания пространства имён UTS нужна привилегия `CAP_SYS_ADMIN`. *Исправление:* используйте
   `sudo` в ВМ (или сочетайте с пространством имён пользователей — это сложнее).

2. **Ожидание, что `unshare(CLONE_NEWPID)` изменит ваш собственный PID.** `getpid()` остаётся прежним; внутри
   только потомки. *Исправление:* после `unshare` вызовите `fork` и работайте в потомке.

3. **Забыли `MS_PRIVATE`.** Без него монтирование, сделанное внутри нового пространства имён монтирования,
   может появиться и в собственном пространстве имён ВМ (при общем распространении). *Исправление:* сразу
   после `unshare(CLONE_NEWNS)` сделайте `/` рекурсивно приватным.

4. **`ps` показывает старые процессы внутри нового пространства имён PID.** `ps` читает `/proc`, а старый
   `/proc` принадлежит старому пространству имён. *Исправление:* смонтируйте внутри новую файловую систему
   `proc` (`--mount-proc` или `mount("proc", "/proc", "proc", …)` в новом пространстве имён монтирования).

## Попробуйте сами

1. Найдите два процесса в одних и тех же пространствах имён: сравните `sudo ls -l /proc/1/ns` с
   `ls -l /proc/self/ns`. Какие номера отличаются внутри контейнера Docker
   (`docker run --rm ubuntu:26.04 ls -l /proc/self/ns`, если Docker работает в вашей ВМ)?
2. Запустите `sudo unshare --uts --pid --fork --mount-proc bash`. Внутри задайте имя хоста, выполните `ps aux`
   и `echo $$`. Выйдите через `exit`. Вы только что вручную собрали ядро контейнера.
3. Во втором терминале, пока работает оболочка из шага 2, найдите её через `ps -ef | grep unshare` и
   сравните `sudo readlink /proc/<pid>/ns/pid` для `unshare` и для его потомка `bash`.

## Упражнения

### Упражнение 1 — То же пространство имён? (легко)

Напишите `src/bin/samens.rs`, который принимает два PID и вид пространства имён (`uts`, `pid`, `mnt`…) и
печатает `same` или `different`, сравнивая цели `/proc/<pid>/ns/<kind>`. Попробуйте с `1` и `$$`.

<details>
<summary>Подсказка</summary>

`std::fs::read_link(format!("/proc/{pid}/ns/{kind}"))` возвращает путь вроде `uts:[4026533109]`. Чтение
процесса другого пользователя требует `sudo`.

</details>

<details>
<summary>Ответ</summary>

```rust
use std::env;
use std::fs;

fn main() -> std::io::Result<()> {
    let args: Vec<String> = env::args().collect();
    if args.len() != 4 {
        eprintln!("usage: samens <pid1> <pid2> <kind>");
        std::process::exit(2);
    }
    let kind = &args[3];
    let a = fs::read_link(format!("/proc/{}/ns/{kind}", args[1]))?;
    let b = fs::read_link(format!("/proc/{}/ns/{kind}", args[2]))?;
    let verdict = if a == b { "same" } else { "different" };
    println!("{verdict}: {} vs {}", a.display(), b.display());
    Ok(())
}
```

`sudo ./target/debug/samens 1 $$ uts` на тестовой машине:

```text
same: uts:[4026533109] vs uts:[4026533109]
```

Почему это работает: цель ссылки — это идентичность пространства имён. Одинаковые цели означают одно и то же
пространство имён; так же группируют процессы инструменты вроде `lsns`.

</details>

### Упражнение 2 — Протестированный разбор флагов (средне)

Напишите `pub fn parse_namespaces(list: &str) -> Result<CloneFlags, String>` в `src/lib.rs`, которая
превращает текст вроде `"uts,pid,mnt"` в соответствующие `CloneFlags`. Неизвестные имена дают
`Err("unknown namespace: xyz")`. Напишите тесты для сочетания, для пробелов вокруг имён и для неизвестного имени.
(Эта идея пригодится в проекте.)

<details>
<summary>Подсказка</summary>

Разделите через `split(',')`, примените `trim()` к каждой части, сопоставьте имя через `match` и объединяйте
флаги через `|=`. Начните с `CloneFlags::empty()`.

</details>

<details>
<summary>Ответ</summary>

```rust
use nix::sched::CloneFlags;

/// Turns "uts,pid,mnt" into the matching clone flags.
pub fn parse_namespaces(list: &str) -> Result<CloneFlags, String> {
    let mut flags = CloneFlags::empty();
    for name in list.split(',').map(str::trim).filter(|s| !s.is_empty()) {
        flags |= match name {
            "mnt" => CloneFlags::CLONE_NEWNS,
            "uts" => CloneFlags::CLONE_NEWUTS,
            "ipc" => CloneFlags::CLONE_NEWIPC,
            "pid" => CloneFlags::CLONE_NEWPID,
            "net" => CloneFlags::CLONE_NEWNET,
            "user" => CloneFlags::CLONE_NEWUSER,
            "cgroup" => CloneFlags::CLONE_NEWCGROUP,
            other => return Err(format!("unknown namespace: {other}")),
        };
    }
    Ok(flags)
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn combines_several_names() {
        let flags = parse_namespaces("uts,pid,mnt").unwrap();
        assert_eq!(
            flags,
            CloneFlags::CLONE_NEWUTS | CloneFlags::CLONE_NEWPID | CloneFlags::CLONE_NEWNS
        );
    }

    #[test]
    fn ignores_spaces_and_empty_parts() {
        assert_eq!(
            parse_namespaces(" uts , ,net").unwrap(),
            CloneFlags::CLONE_NEWUTS | CloneFlags::CLONE_NEWNET
        );
    }

    #[test]
    fn rejects_unknown_names() {
        assert_eq!(
            parse_namespaces("uts,xyz"),
            Err("unknown namespace: xyz".to_string())
        );
    }
}
```

`cargo test`:

```text
test result: ok. 3 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

Почему это работает: `CloneFlags` — набор битов, поэтому `|=` добавляет вид в набор. Разбору не нужны права
root, его можно тестировать где угодно; привилегированное окружение нужно только коду, который вызывает
`unshare`.

</details>

### Упражнение 3 — Крошечный «контейнер» (сложно)

Объедините разделы 3–5 в `src/bin/box.rs`: вызовите `unshare` для пространств имён UTS, PID и монтирования;
затем `fork`. В потомке: сделайте `/` приватным, смонтируйте новый `proc` в `/proc`, задайте имя хоста `box` и
запустите `/bin/sh` через `std::process::Command`, дождавшись его. Родитель ждёт потомка. Внутри оболочки
`hostname` должна печатать `box`, а `ps` — показывать всего несколько процессов, причём PID 1 — это ваша
программа.

<details>
<summary>Подсказка</summary>

Смонтируйте `proc` через `mount(Some("proc"), "/proc", Some("proc"), MsFlags::empty(), None::<&str>)`. Это
должно происходить в потомке, потому что только потомок находится внутри нового пространства имён PID.
Потомок — PID 1, а оболочка, которую он запускает, — PID 2.

</details>

<details>
<summary>Ответ</summary>

```rust
use std::process::Command;

use nix::mount::{MsFlags, mount};
use nix::sched::{CloneFlags, unshare};
use nix::sys::wait::waitpid;
use nix::unistd::{ForkResult, fork, sethostname};

fn child() -> nix::Result<()> {
    mount(
        None::<&str>,
        "/",
        None::<&str>,
        MsFlags::MS_REC | MsFlags::MS_PRIVATE,
        None::<&str>,
    )?;
    mount(
        Some("proc"),
        "/proc",
        Some("proc"),
        MsFlags::empty(),
        None::<&str>,
    )?;
    sethostname("box")?;
    let status = Command::new("/bin/sh")
        .status()
        .expect("cannot start /bin/sh");
    println!("shell exited with {status}");
    Ok(())
}

fn main() -> nix::Result<()> {
    unshare(CloneFlags::CLONE_NEWUTS | CloneFlags::CLONE_NEWPID | CloneFlags::CLONE_NEWNS)?;
    // SAFETY: single-threaded program.
    match unsafe { fork() }? {
        ForkResult::Child => {
            child()?;
            std::process::exit(0);
        }
        ForkResult::Parent { child } => {
            waitpid(child, None)?;
        }
    }
    Ok(())
}
```

Сеанс на тестовой машине (введённые команды — после `#`):

```text
# hostname
box
# ps -o pid,comm
    PID COMMAND
      1 box
      2 sh
      4 ps
# exit
/bin/sh: 3: Cannot set tty process group (No such process)
shell exited with exit status: 2
```

PID 3 была команда `hostname`, которая уже завершилась. Сообщение в конце выдаёт `sh`: при выходе он пытается
вернуть терминал той группе процессов, которая была на переднем плане раньше. Эта группа живёт вне нового
пространства имён PID, поэтому внутри у неё нет номера («No such process»), и `sh` сообщает код завершения 2.
Настоящие среды выполнения избегают этого, давая контейнеру собственный сеанс и терминал; на то, что вы здесь
изучаете, это не влияет.

Почему это работает: `unshare` подготовил три новых пространства имён; `fork` поместил потомка в новое
пространство имён PID как PID 1. Свежее монтирование `proc` показывает процессы этого пространства имён.
Пространство имён монтирования не выпускает это монтирование `/proc` в ВМ. Чего ещё не хватает настоящему
контейнеру: своей корневой файловой системы и ограничений ресурсов. Это следующие два урока.

</details>

## Вопросы для самопроверки

1. Что делает пространство имён?

<details>
<summary>Ответ</summary>

Оно даёт группе процессов собственный приватный вид одного глобального ресурса ядра, например имени хоста
(UTS), номеров процессов (PID), таблицы монтирования (mount) или сети (net). Внутри всё выглядит обычно;
снаружи ничего не меняется.

</details>

2. Как узнать, делят ли два процесса одно пространство имён?

<details>
<summary>Ответ</summary>

Сравнить цели `/proc/<pid>/ns/<kind>`, например `uts:[4026533109]`. Одинаковые числа означают одно и то же
пространство имён.

</details>

3. Какой PID у вызывающего процесса после `unshare(CLONE_NEWPID)` и какой PID у его следующего потомка?

<details>
<summary>Ответ</summary>

Вызывающий процесс сохраняет старый PID; он остаётся в старом пространстве имён PID. Его следующий потомок —
первый процесс нового пространства имён, и внутри у него PID 1 (а снаружи — обычный PID).

</details>

4. Почему `docker stop` иногда занимает 10 секунд?

<details>
<summary>Ответ</summary>

`docker stop` посылает SIGTERM процессу PID 1 контейнера. Ядро не доставляет сигналы процессу PID 1
пространства имён, если тот не установил обработчик. Программа, которая не обрабатывает SIGTERM, его
игнорирует, и через 10 секунд Docker применяет SIGKILL.

</details>

5. Зачем делать `/` приватным сразу после создания пространства имён монтирования?

<details>
<summary>Ответ</summary>

При общем распространении монтирования, сделанные внутри нового пространства имён, могли бы появиться и в
родительском. `MS_REC | MS_PRIVATE` разрывает эту связь, и монтирования контейнера остаются внутри.

</details>

## Итоги

- Пространство имён даёт процессам приватный вид одного глобального ресурса; в Linux есть пространства имён монтирования, UTS, IPC, PID, сетевые, пользователей, cgroup и времени.
- `/proc/<pid>/ns/*` показывает пространства имён процесса; одинаковые числа означают одно и то же пространство имён.
- `unshare` (команда и системный вызов) переносит процесс в новые пространства имён; для их создания нужны права root, кроме пространства имён пользователей.
- Новое пространство имён PID действует на потомков: первый потомок — PID 1, он усыновляет сирот, игнорирует сигналы без обработчиков и при выходе забирает с собой всех остальных.
- Новое пространство имён монтирования сначала нужно сделать приватным; тогда монтирования внутри остаются внутри.
- Пространства имён — стены контейнера, но ядро у всех одно; настоящие среды выполнения добавляют привилегии, seccomp и cgroups.

## Источники и что почитать

- [namespaces(7) — Linux manual page](https://man7.org/linux/man-pages/man7/namespaces.7.html) — все виды пространств имён и файлы `/proc/<pid>/ns`.
- [pid_namespaces(7) — Linux manual page](https://man7.org/linux/man-pages/man7/pid_namespaces.7.html) — PID 1, сигналы и вложенность.
- [mount_namespaces(7) — Linux manual page](https://man7.org/linux/man-pages/man7/mount_namespaces.7.html) — распространение монтирования (`shared`, `private`).
- [unshare(1) — Linux manual page](https://man7.org/linux/man-pages/man1/unshare.1.html) — команда из этого урока.
- [nix::sched на docs.rs](https://docs.rs/nix/0.32.0/nix/sched/index.html) — `unshare`, `setns` и `CloneFlags` в Rust.
- [Namespaces in operation, part 1](https://lwn.net/Articles/531114/) — начало известной серии статей на LWN.net (изначально «Linux Weekly News», давний сайт о ядре Linux).

---

[🏠 Начало](../START-HERE.md) · [← 13.14 io_uring](14-io-uring.md) · [📋 Этап 13](README.md) · [13.16 Контрольные группы (cgroups) →](16-cgroups.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
