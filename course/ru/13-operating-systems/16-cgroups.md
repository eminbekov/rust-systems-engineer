<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 13.15 Пространства имён (namespaces)](15-namespaces.md) · [📋 Этап 13](README.md) · [13.17 Проект: мини-среда выполнения контейнеров →](17-project-mini-container-runtime.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/13-operating-systems/16-cgroups.md) <!-- nav -->

# Урок 13.16 — Контрольные группы (cgroups)

> **Этап 13 — Операционные системы изнутри** · Урок 16 из 18 · ⏱ около 120 минут

## Что вы узнаете

- Что такое контрольные группы и чем они отличаются от пространств имён.
- Как устроено дерево cgroup v2 в `/sys/fs/cgroup`: группы, контроллеры и интерфейсные файлы.
- Как ограничить память (`memory.max`), процессорное время (`cpu.max`) и число процессов (`pids.max`) и как увидеть, что ограничение сработало.
- Как поместить программу в cgroup из Rust — это последний кирпичик для проекта с контейнером.

## Перед началом

- Вы прошли [Урок 13.15 — Пространства имён (namespaces)](15-namespaces.md).
- Вы помните OOM killer (OOM, Out Of Memory, нехватка памяти) и RSS (Resident Set Size, память процесса, которая действительно находится в RAM) ([Урок 13.10](10-virtual-memory.md)), планировщик ([Урок 13.9](09-threads-and-scheduling.md)) и юниты systemd ([Урок 11.6](../11-linux-shell-and-containers/06-services-with-systemd.md)).
- Создайте пакет на Mac и скопируйте его в ВМ, как в Уроке 13.10:

```bash
cd ~/projects/rust
cargo new --lib code/p13-cgroup
mkdir code/p13-cgroup/src/bin
```

Запись в `/sys/fs/cgroup` требует прав root, поэтому команды используют `sudo` — **только в ВМ**. Собирайте
примеры в режиме release (`cargo build --release`), чтобы измерения CPU имели смысл. Тестовой машиной был
привилегированный контейнер, поэтому некоторые пути отличаются от вашей ВМ; выводы, зависящие от ВМ, помечены
как примеры.

## Новые слова

| Слово | Значение |
|---|---|
| **cgroup** (control group, контрольная группа) | Группа процессов, расход ресурсов которой (память, время CPU (Central Processing Unit, центральный процессор), ввод-вывод (I/O, input/output: чтение и запись диска), число процессов) ядро считает и может ограничивать. |
| **cgroup v2** | Текущая версия: одно дерево для всех ресурсов, смонтированное в `/sys/fs/cgroup`. (В версии 1 у каждого ресурса было своё дерево.) |
| **контроллер** (controller) | Часть ядра, которая управляет одним ресурсом для cgroups: `memory`, `cpu`, `io`, `pids`, … |
| **интерфейсный файл** (interface file) | Файл в каталоге cgroup, который показывает значение или задаёт ограничение, например `memory.max`. |
| **`cgroup.procs`** | Интерфейсный файл со списком процессов группы; запись PID в него переносит этот процесс в группу. |
| **троттлинг** (throttling) | Группу заставляют ждать, потому что она израсходовала своё процессорное время за текущий период. |
| **OOM kill** (Out Of Memory, нехватка памяти) | Убийство процесса, потому что его группа достигла ограничения памяти и ядро не смогло освободить достаточно. |
| **slice / scope** | Имена systemd для создаваемых им cgroup: *slice* объединяет службы или пользователей, *scope* содержит процессы, запущенные не через systemd. |

## 1. Пространства имён говорят, *что вы видите*, cgroups — *сколько вы используете*

Пространства имён (Урок 13.15) меняли то, что процесс может **видеть**. Они не мешают ему занять всю память
или весь CPU машины. Контейнер с утечкой памяти всё равно довёл бы всю машину до OOM killer.

**Контрольные группы** закрывают этот пробел. cgroup — это группа процессов с бюджетом:

- Ядро **считает**, сколько группа использует (память, время CPU, дисковый ввод-вывод, число процессов).
- Ядро **соблюдает** заданные вами ограничения: группа не может получить больше памяти, чем `memory.max`,
  больше времени CPU, чем `cpu.max`, больше процессов, чем `pids.max`.
- Потомки процесса начинают в той же cgroup, поэтому ограничение покрывает всю программу и всё, что она
  запускает.

Любая среда выполнения контейнеров использует и то и другое: пространства имён для стен, cgroups для бюджета.
Когда Docker показывает контейнер как `OOMKilled` или Kubernetes соблюдает `resources.limits`, работает
контроллер памяти cgroup.

Интерфейс — это файловая система. Никакого особого системного вызова нет: группа создаётся через `mkdir`,
ограничение задаётся через `echo`, а процесс переносится ещё одним `echo`.

## 2. Осматриваемся в дереве

Каждый процесс принадлежит ровно одной cgroup. В ВМ:

```bash
cat /proc/self/cgroup
```

Пример вывода (у вас будет похоже):

```text
0::/user.slice/user-501.slice/session-1.scope
```

`0::` означает cgroup v2, а остальное — путь внутри дерева. Его создал systemd: ваш сеанс входа — это *scope*
внутри *slice* вашего пользователя. Само дерево смонтировано в `/sys/fs/cgroup`:

```bash
ls /sys/fs/cgroup
cat /sys/fs/cgroup/cgroup.controllers
cat /sys/fs/cgroup/cgroup.subtree_control
```

Пример вывода двух последних команд (у вас будет похоже):

```text
cpuset cpu io memory hugetlb pids rdma misc
cpuset cpu io memory pids
```

- `cgroup.controllers` перечисляет контроллеры, доступные в этой группе.
- `cgroup.subtree_control` перечисляет контроллеры, **включённые для потомков** этой группы. systemd уже
  включил `cpu`, `memory` и `pids` в корне, поэтому новые группы прямо внутри `/sys/fs/cgroup` получают эти
  интерфейсные файлы.

> 💡 **Совет:** `systemd-cgls` печатает всё дерево с процессами в нём, а `systemd-cgtop` показывает расход по
> группам в реальном времени, как `top`.

## 3. Ваша первая cgroup: ограничение памяти

Создайте группу. Ядро сразу заполняет новый каталог интерфейсными файлами:

```bash
sudo mkdir /sys/fs/cgroup/course-demo
ls /sys/fs/cgroup/course-demo
```

Вывод на тестовой машине, сокращённо:

```text
cgroup.controllers cgroup.events cgroup.freeze cgroup.kill ... cgroup.procs ... cpu.max cpu.stat ...
memory.current memory.events ... memory.max memory.peak ... memory.swap.max ... pids.current pids.events pids.max ...
```

Все ограничения начинаются с «без ограничения»:

```bash
cat /sys/fs/cgroup/course-demo/memory.max /sys/fs/cgroup/course-demo/cpu.max /sys/fs/cgroup/course-demo/pids.max
```

```text
max
max 100000
max
```

Теперь программа, которая ест память. Создайте `src/bin/hog.rs`:

```rust
use std::thread;
use std::time::Duration;

fn main() {
    const STEP: usize = 10 * 1024 * 1024; // 10 MiB
    let mut chunks: Vec<Vec<u8>> = Vec::new();
    for i in 1..=20 {
        chunks.push(vec![1u8; STEP]); // vec![1; …] really writes every byte, so it uses RAM
        println!("allocated {:>3} MiB", i * 10);
        thread::sleep(Duration::from_millis(100));
    }
    println!("done, total {} MiB", chunks.len() * 10);
}
```

Она выделяет 200 МиБ шагами по 10 МиБ. Вспомните подкачку по требованию (Урок 13.10): `vec![0u8; …]` не
занимал бы RAM, пока к нему не обратились, поэтому мы заполняем единицами. Задайте ограничение в 50 МиБ,
запретите этой группе swap и запустите программу внутри группы:

```bash
echo 50M | sudo tee /sys/fs/cgroup/course-demo/memory.max
echo 0 | sudo tee /sys/fs/cgroup/course-demo/memory.swap.max
sudo sh -c 'echo $$ > /sys/fs/cgroup/course-demo/cgroup.procs; exec ./target/release/hog'
echo "exit=$?"
```

```text
50M
0
allocated  10 MiB
allocated  20 MiB
allocated  30 MiB
allocated  40 MiB
Killed
exit=137
```

**Что происходит:**

1. `tee` записывает свой ввод в файл; `sudo tee` — обычный способ записать файл, принадлежащий root, из
   обычной оболочки (`sudo echo 50M > file` не сработает, потому что перенаправление выполняет ваша собственная
   оболочка). Ядро принимает `50M` и хранит значение в байтах (`cat memory.max` теперь показывает `52428800`).
2. `memory.swap.max = 0` не даёт группе выносить страницы в swap, поэтому ограничение достигается быстро.
3. `sh -c 'echo $$ > …/cgroup.procs; exec …'` запускает маленькую оболочку, переносит **эту оболочку** в группу
   (`$$` — её PID), а затем заменяет её на `hog` через `exec` (Урок 13.5). Поэтому `hog` работает внутри
   группы с первой же инструкции.
4. Примерно на 50 МиБ группа больше не может получить память. Ядро пытается освободить память внутри группы,
   не может, и **OOM killer убивает процесс только этой группы**. Остальная ВМ не затронута.
5. Код завершения 137 = 128 + 9: SIGKILL.

Группа хранит запись о том, что случилось:

```bash
cat /sys/fs/cgroup/course-demo/memory.events
```

```text
low 0
high 0
max 25
oom 1
oom_kill 1
oom_group_kill 0
sock_throttled 0
```

`max 25` считает, сколько раз группа упиралась в ограничение (тогда ядро пыталось вернуть память), а
`oom_kill 1` показывает, что один процесс был убит. Именно этот файл читают инструменты, чтобы сообщить
«OOMKilled».

## 4. Ограничение CPU

`cpu.max` содержит два числа: **квоту** и **период**, оба в микросекундах. `20000 100000` означает «в каждом
периоде длиной 100 мс группа может работать в сумме 20 мс»: 20% одного CPU. Когда квота израсходована, потоки
группы ждут следующего периода. Это ожидание называется **троттлингом**. Создайте `src/bin/spin.rs`:

```rust
use std::time::{Duration, Instant};

fn main() {
    for second in 1..=3 {
        let start = Instant::now();
        let mut loops: u64 = 0;
        while start.elapsed() < Duration::from_secs(1) {
            loops += 1; // pure CPU work
        }
        println!("second {second}: {:>5} million loops", loops / 1_000_000);
    }
}
```

Запустите её один раз без ограничения, а затем внутри группы с ограничением в 20%:

```bash
./target/release/spin
echo "20000 100000" | sudo tee /sys/fs/cgroup/course-demo/cpu.max
sudo sh -c 'echo $$ > /sys/fs/cgroup/course-demo/cgroup.procs; exec ./target/release/spin'
cat /sys/fs/cgroup/course-demo/cpu.stat
```

```text
second 1:    50 million loops
second 2:    51 million loops
second 3:    51 million loops
20000 100000
second 1:     9 million loops
second 2:     6 million loops
second 3:     7 million loops
usage_usec 623400
user_usec 613629
system_usec 9771
nice_usec 0
nr_periods 30
nr_throttled 30
throttled_usec 2388853
nr_bursts 0
burst_usec 0
```

**Что происходит:**

1. Без ограничения цикл выполняется примерно 50 миллионов раз в секунду (на тестовой машине).
2. С ограничением в 20% он получает намного меньше: программа по-прежнему работает три секунды *реального*
   времени, но может использовать CPU только пятую часть этого времени. Числа не ровно 10 миллионов, потому что
   программа ещё и измеряет время, а троттлинг происходит кусками по 100 мс.
3. `cpu.stat` это подтверждает: за 30 периодов группа подверглась троттлингу 30 раз и в сумме прождала около
   2,4 секунды (`throttled_usec`), а CPU использовала около 0,6 секунды (`usage_usec`).

Квота больше периода даёт больше одного CPU: `200000 100000` позволяет два полных CPU, если у программы два
занятых потока. Именно это записывают `docker run --cpus=2` и ограничение CPU `2` в Kubernetes.

> ⚡ **Производительность:** ограничение CPU не делает программу медленнее на каждую инструкцию; оно заставляет
> её *ждать*. Веб-сервис с жёстким ограничением CPU может отвечать долго (хвостовая задержка), даже когда машина
> простаивает. Проверьте `nr_throttled`, прежде чем винить свой код.

## 5. Ограничение числа процессов

`pids.max` не даёт группе создать слишком много процессов или потоков, например при *форк-бомбе* (программе,
которая копирует себя снова и снова, пока машина не перестанет что-либо запускать):

```bash
echo 5 | sudo tee /sys/fs/cgroup/course-demo/pids.max
sudo sh -c 'echo $$ > /sys/fs/cgroup/course-demo/cgroup.procs; for i in 1 2 3 4 5 6; do sleep 1 & done; wait'
echo "exit=$?"
cat /sys/fs/cgroup/course-demo/pids.events
```

```text
5
sh: 0: Cannot fork
exit=2
max 1
```

Сама оболочка и четыре процесса `sleep` дают пять; пятый `sleep` создать не удалось, и `fork` завершился
ошибкой, вместо того чтобы завалить машину. `pids.events` считает, сколько раз ограничение срабатывало.

Когда закончите, дождитесь, пока группа опустеет, и удалите её. Удалить cgroup через `rmdir` можно только тогда,
когда в ней нет ни одного процесса:

```bash
sleep 2
cat /sys/fs/cgroup/course-demo/pids.current
sudo rmdir /sys/fs/cgroup/course-demo
```

```text
0
```

## 6. То же самое из Rust

Ничего особенного делать не нужно: cgroups — это файлы, так что хватит `std::fs`. Программа, которая хочет
ограничить команду:

1. создаёт каталог в `/sys/fs/cgroup`,
2. записывает ограничения в `memory.max`, `cpu.max`, …,
3. записывает PID в `cgroup.procs`: либо свой (тогда внутри окажется и каждый запущенный ею потомок), либо
   потомка,
4. после работы выходит обратно и удаляет каталог.

Ровно это вы напишете в упражнении 3 и используете в проекте следующего урока.

**Кто ещё это делает?** `systemd-run --scope -p MemoryMax=50M -p CPUQuota=20% <command>` просит systemd
создать такой scope за вас. В системе под управлением systemd это вежливый путь, потому что systemd считает
дерево своим. Параметры Docker `--memory` и `--cpus` и `resources.limits` в Kubernetes в итоге превращаются в
те же файлы `memory.max` и `cpu.max`.

> 🔒 **Безопасность:** без ограничений одна ошибочная или враждебная программа может оставить без ресурсов все
> остальные (*отказ в обслуживании*, denial of service). Задавать `memory.max`, `cpu.max` и `pids.max` для каждой
> службы и контейнера — базовая гигиена, и юниты systemd умеют это через `MemoryMax=`, `CPUQuota=` и `TasksMax=`.

## Частые ошибки

1. **`sudo echo 50M > memory.max` пишет «Permission denied».** Перенаправление выполняет ваша обычная оболочка
   ещё до запуска `sudo`. *Исправление:* `echo 50M | sudo tee …/memory.max` или `sudo sh -c 'echo 50M > …'`.

2. **В новой группе нет файла `memory.max`.** Контроллер `memory` не включён в `cgroup.subtree_control`
   родителя. *Исправление:* проверьте родителя; в корне машины с systemd он обычно включён.

3. **`rmdir` завершается ошибкой «Device or resource busy».** В группе ещё есть процессы (возможно, ваша
   собственная оболочка, если вы записали в `cgroup.procs` значение `$$` своей оболочки входа).
   *Исправление:* перенесите их (запишите их PID в `cgroup.procs` другой группы) или дождитесь их завершения.

4. **Ограничение памяти «не работает».** Программа только зарезервировала память (подкачка по требованию) и
   ни разу к ней не обратилась, или группа уходит в swap вместо отказа. *Исправление:* обращайтесь к памяти и в
   экспериментах ставьте `memory.swap.max` в 0.

## Попробуйте сами

1. Посмотрите, сколько использует ваш сеанс входа:
   `cat /sys/fs/cgroup$(cut -d: -f3 /proc/self/cgroup)/memory.current`. (`cut -d: -f3` берёт третье поле,
   разделённое `:`, из `0::/…` — это путь.)
2. Выполните `systemd-cgls --no-pager | head -30` и найдите свою оболочку в дереве.
3. Запустите `hog` через `systemd-run --scope -p MemoryMax=50M -p MemorySwapMax=0 ./target/release/hog`
   (с `sudo`). Совпадает ли результат с разделом 3?

## Упражнения

### Упражнение 1 — Сколько использует моя оболочка? (легко)

Напечатайте текущий и пиковый расход памяти cgroup вашей оболочки, используя только `/proc/self/cgroup` и
`/sys/fs/cgroup`. Почему число намного больше, чем RSS оболочки?

<details>
<summary>Подсказка</summary>

Постройте путь из третьего поля `/proc/self/cgroup`, затем прочитайте `memory.current` и `memory.peak`.

</details>

<details>
<summary>Ответ</summary>

```bash
cat /sys/fs/cgroup$(cut -d: -f3 /proc/self/cgroup)/memory.current
cat /sys/fs/cgroup$(cut -d: -f3 /proc/self/cgroup)/memory.peak
```

На тестовой машине (маленький контейнер) вывод был таким:

```text
856064
1425408
```

Почему это работает: cgroup считает память **всех** своих процессов, и не только их RSS, но и память ядра,
использованную для них, и страницы файлов в страничном кэше, которые они подгрузили. В группе вашего сеанса
входа — все программы, которые вы из него запускали.

</details>

### Упражнение 2 — Протестированные вспомогательные функции (средне)

Напишите две функции в `src/lib.rs` с тестами:

- `pub fn cpu_max(percent: u32) -> String`: текст для `cpu.max` для доли одного CPU с периодом 100 мс;
  `20` → `"20000 100000"`, `250` → `"250000 100000"`.
- `pub fn cgroup_path(proc_cgroup: &str) -> Option<&str>`: путь из текста `/proc/<pid>/cgroup` (строка,
  начинающаяся с `0::`), или `None`, если строки v2 нет.

<details>
<summary>Подсказка</summary>

`line.strip_prefix("0::")` возвращает `Some(rest)` только для нужной строки; `find_map` останавливается на
первом `Some`.

</details>

<details>
<summary>Ответ</summary>

```rust
/// The value for `cpu.max` that allows `percent` % of one CPU per 100 ms period.
/// 100 means one whole CPU, 250 means two and a half CPUs.
pub fn cpu_max(percent: u32) -> String {
    const PERIOD_US: u32 = 100_000;
    format!("{} {PERIOD_US}", percent * PERIOD_US / 100)
}

/// Finds the cgroup v2 path in the text of /proc/<pid>/cgroup, for example "/user.slice".
pub fn cgroup_path(proc_cgroup: &str) -> Option<&str> {
    proc_cgroup
        .lines()
        .find_map(|line| line.strip_prefix("0::"))
        .map(str::trim)
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn cpu_max_for_common_values() {
        assert_eq!(cpu_max(20), "20000 100000");
        assert_eq!(cpu_max(100), "100000 100000");
        assert_eq!(cpu_max(250), "250000 100000");
    }

    #[test]
    fn finds_the_v2_line() {
        let text = "0::/user.slice/user-1000.slice/session-3.scope\n";
        assert_eq!(
            cgroup_path(text),
            Some("/user.slice/user-1000.slice/session-3.scope")
        );
        assert_eq!(cgroup_path("12:pids:/old-v1-line\n"), None);
    }
}
```

`cargo test`:

```text
test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

Почему это работает: обе функции работают только с текстом, поэтому их можно тестировать без root. Весь код,
который трогает `/sys/fs/cgroup`, остаётся маленькой тонкой обёрткой вокруг них.

</details>

### Упражнение 3 — `limit`: запуск команды с ограничениями (сложно)

Напишите `src/bin/limit.rs`: `limit <memory> <cpu-percent> <command> [args...]`. Программа создаёт
`/sys/fs/cgroup/course-limit-<свой PID>`, записывает `memory.max`, `memory.swap.max = 0` и `cpu.max`
(используйте ваши вспомогательные функции), переносит себя внутрь, запускает команду и ждёт её, затем печатает
статус команды и строку `oom_kill` из `memory.events`, возвращает себя в исходную группу и удаляет группу.
Проверьте: `sudo ./target/release/limit 50M 100 ./target/release/hog` сообщает об OOM kill, `300M` — нет, и
после этого не остаётся ни одной группы `course-limit-*`.

<details>
<summary>Подсказка</summary>

Прочитайте `/proc/self/cgroup` **до** переноса, чтобы знать, куда возвращаться.
`Command::new(&args[2]).args(&args[3..]).status()` запускает команду и ждёт её.

</details>

<details>
<summary>Ответ</summary>

```rust
use std::env;
use std::fs;
use std::path::PathBuf;
use std::process::{self, Command};

use p13_cgroup::{cgroup_path, cpu_max};

fn main() -> std::io::Result<()> {
    let args: Vec<String> = env::args().skip(1).collect();
    if args.len() < 3 {
        eprintln!("usage: limit <memory, e.g. 50M> <cpu percent> <command> [args...]");
        process::exit(2);
    }
    let root = PathBuf::from("/sys/fs/cgroup");
    let me = process::id().to_string();

    // Remember where we are now, so we can go back.
    let original = fs::read_to_string("/proc/self/cgroup")?;
    let original = root.join(
        cgroup_path(&original)
            .unwrap_or("/")
            .trim_start_matches('/'),
    );

    // 1. Create the group and set the limits.
    let group = root.join(format!("course-limit-{me}"));
    fs::create_dir(&group)?;
    fs::write(group.join("memory.max"), &args[0])?;
    fs::write(group.join("memory.swap.max"), "0")?;
    fs::write(
        group.join("cpu.max"),
        cpu_max(args[1].parse().expect("cpu percent")),
    )?;

    // 2. Move ourselves in; the child we start inherits the group.
    fs::write(group.join("cgroup.procs"), &me)?;
    let status = Command::new(&args[2]).args(&args[3..]).status()?;

    // 3. Report, move back out, remove the group.
    let events = fs::read_to_string(group.join("memory.events"))?;
    let oom_kills = events
        .lines()
        .find(|l| l.starts_with("oom_kill "))
        .unwrap_or("?");
    fs::write(original.join("cgroup.procs"), &me)?;
    fs::remove_dir(&group)?;
    println!("limit: command finished with {status}; {oom_kills}");
    Ok(())
}
```

```bash
sudo ./target/release/limit 50M 100 ./target/release/hog
sudo ./target/release/limit 300M 100 ./target/release/hog | tail -2
ls /sys/fs/cgroup | grep course || echo "no course groups left"
```

```text
allocated  10 MiB
allocated  20 MiB
allocated  30 MiB
allocated  40 MiB
limit: command finished with signal: 9 (SIGKILL); oom_kill 1
done, total 200 MiB
limit: command finished with exit status: 0; oom_kill 0
no course groups left
```

Почему это работает: дочерний процесс начинает в cgroup родителя, поэтому `hog` ограничен с первой
инструкции. OOM killer выбрал `hog` — самый большой процесс в группе, — и `limit` выжил, чтобы об этом
сообщить. Сначала выйти из группы необходимо, потому что группу с процессом внутри удалить нельзя. Остаётся
небольшая слабость: если сам `limit` упадёт посередине, группа останется; проект следующего урока должен
убирать такие остатки.

</details>

## Вопросы для самопроверки

1. Чем пространство имён отличается от cgroup?

<details>
<summary>Ответ</summary>

Пространство имён меняет то, что процессы могут *видеть* (своё имя хоста, список процессов, монтирования…).
cgroup управляет тем, сколько они могут *использовать* (память, время CPU, число процессов), и считает их
расход.

</details>

2. Как перенести работающий процесс в cgroup?

<details>
<summary>Ответ</summary>

Записать его PID в файл `cgroup.procs` группы. Его будущие потомки начнут в той же группе.

</details>

3. Что означает `cpu.max` = `50000 100000`?

<details>
<summary>Ответ</summary>

В каждом периоде длиной 100 мс группа может использовать в сумме 50 мс времени CPU: половину одного CPU.
Когда квота израсходована, потоки группы подвергаются троттлингу до следующего периода.

</details>

4. Процесс в группе с `memory.max = 100M` обращается к 150 МиБ памяти и не может уйти в swap. Что
   произойдёт и где это увидеть?

<details>
<summary>Ответ</summary>

Ядро попытается вернуть память в группе; когда не сможет, OOM killer убьёт процесс этой группы (код
завершения 137, SIGKILL). `memory.events` покажет `oom_kill 1`. Процессы вне группы не затронуты.

</details>

5. Почему `rmdir` для cgroup может завершиться ошибкой «Device or resource busy»?

<details>
<summary>Ответ</summary>

В группе ещё есть процессы. Перед удалением она должна быть пустой.

</details>

## Итоги

- cgroup — это группа процессов, расход ресурсов которой ядро считает и ограничивает; потомки наследуют группу родителя.
- cgroup v2 — одно дерево в `/sys/fs/cgroup`; группы — это каталоги, настройки — интерфейсные файлы, а контроллеры нужно включить в `cgroup.subtree_control` родителя.
- `memory.max` ограничивает память (OOM killer действует внутри группы; см. `memory.events`), `cpu.max` задаёт квоту и период (см. `cpu.stat`), `pids.max` ограничивает число процессов.
- `cgroup.procs` переносит процессы; группу можно удалить, только когда она пуста.
- Из Rust cgroups — это обычные операции с файлами; systemd, Docker и Kubernetes пишут в те же файлы.

## Источники и что почитать

- [The Linux kernel documentation — Control Group v2](https://docs.kernel.org/admin-guide/cgroup-v2.html) — полный справочник по каждому интерфейсному файлу из урока.
- [cgroups(7) — Linux manual page](https://man7.org/linux/man-pages/man7/cgroups.7.html) — обзор cgroups v1 и v2.
- [systemd.resource-control(5)](https://www.freedesktop.org/software/systemd/man/latest/systemd.resource-control.html) — `MemoryMax=`, `CPUQuota=`, `TasksMax=` и как systemd отображает их на cgroups.
- [Docker documentation — Resource constraints](https://docs.docker.com/engine/containers/resource_constraints/) — как `--memory` и `--cpus` связаны с cgroups.
- [Kubernetes documentation — Resource Management for Pods and Containers](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/) — requests и limits, для следующих этапов.

---

[🏠 Начало](../START-HERE.md) · [← 13.15 Пространства имён (namespaces)](15-namespaces.md) · [📋 Этап 13](README.md) · [13.17 Проект: мини-среда выполнения контейнеров →](17-project-mini-container-runtime.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
