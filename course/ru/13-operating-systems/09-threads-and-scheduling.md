<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 13.8 Проект: собственная командная оболочка](08-project-your-own-shell.md) · [📋 Этап 13](README.md) · [13.10 Виртуальная память →](10-virtual-memory.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/13-operating-systems/09-threads-and-scheduling.md) <!-- nav -->

# Урок 13.9 — Потоки и планирование

> **Этап 13 — Операционные системы изнутри** · Урок 9 из 18 · ⏱ около 135 минут

## Что вы узнаете

- Что такое поток для ядра и чем он отличается от процесса: Linux создаёт и то и другое через `clone`, с разными флагами.
- Как увидеть потоки процесса: TID, `/proc/<pid>/task` и `ps -L`.
- Что делает планировщик: очереди выполнения, кванты времени, вытеснение и переключения контекста, и как их посчитать.
- Как Linux честно делит процессор (EEVDF, значения nice и веса) и как измерить это самому.
- Как привязать поток к ядру процессора через `sched_setaffinity` и сколько стоит создать поток по сравнению с процессом.

## Перед началом

- Вы прошли [Урок 13.8 — Проект: собственная командная оболочка](08-project-your-own-shell.md) или хотя бы Уроки 13.1–13.7.
- Вы пользовались потоками через `std::thread::spawn`, `join`, `Arc` и `Mutex` в [Уроке 05.10 — Потоки, каналы и общее состояние](../05-rust-intermediate/10-threads-channels-and-shared-state.md). Здесь вы увидите, что с ними делает ядро.
- Вы знаете прерывание таймера (Урок 13.1), состояния процессов (Урок 13.2) и `clone` из вывода `strace` в Уроке 13.5.

## Новые слова

| Слово | Значение |
|---|---|
| **поток** (thread) | Одна линия выполнения внутри процесса: свои регистры и стек, но та же память, те же файлы и обработчики сигналов, что и у других потоков процесса. |
| **задача** (task) | Слово ядра Linux для всего, что оно планирует. Каждый поток — задача; однопоточный процесс — одна задача. |
| **TID** (Thread IDentifier, идентификатор потока) | Номер ядра для одного потока. TID главного потока равен PID. |
| **TGID** (Thread Group IDentifier, идентификатор группы потоков) | Название ядра для того, что вы называете PID: идентификатор, общий для всех потоков одного процесса. |
| **LWP** (Light-Weight Process, легковесный процесс) | Старое название потока; `ps -L` показывает TID в столбце `LWP`. |
| **планировщик** (scheduler) | Часть ядра, которая решает, какая задача на каком ядре процессора работает и как долго. |
| **очередь выполнения** (run queue) | Список готовых к работе задач (состояние `R`), ожидающих одно ядро процессора. У каждого ядра своя. |
| **квант времени** (time slice) | Сколько задача может работать, прежде чем планировщик проверит, не должен ли работать кто-то другой. |
| **вытеснение** (preemption) | Ядро отбирает процессор у работающей задачи с помощью прерывания таймера, не спрашивая её. |
| **переключение контекста** (context switch) | Сохранение регистров одной задачи и загрузка регистров другой, чтобы процессор продолжил другую. |
| **значение nice** (nice value) | Число от −20 до 19, которое пользователь задаёт процессу: больше — значит «вежливее к другим», то есть меньшая доля процессора. |
| **EEVDF** (Earliest Eligible Virtual Deadline First, «сначала допустимый с самым ранним виртуальным сроком») | Алгоритм обычного планировщика Linux начиная с ядра 6.6; он заменил CFS (Completely Fair Scheduler, «полностью честный планировщик»). |
| **привязка к процессору** (CPU affinity) | Набор ядер процессора, на которых потоку разрешено работать. |
| **CPU**, **ВМ**, **МиБ**, **I/O** | Известны из прошлых уроков: Central Processing Unit, центральный процессор; виртуальная машина (ваша ВМ `dev`); мебибайт (1024 × 1024 байта); Input/Output, ввод-вывод. |

## 1. Поток — это задача, которая делит ресурсы

Вы знаете процесс как «работающую программу со своей памятью». Поток — это вторая (третья, четвёртая…) линия выполнения
**внутри того же процесса**. Представьте кухню: процесс — это кухня с холодильником и кастрюлями; потоки — повара. У каждого
повара свои руки и своё место в рецепте, но все пользуются одним холодильником и одними кастрюлями — поэтому им нужно
договариваться, кто что берёт (Этап 14).

Что есть у каждого потока своё и что потоки одного процесса делят:

| Своё у каждого потока | Общее для всех потоков процесса |
|---|---|
| регистры, включая счётчик команд | память: код, куча, static (одно адресное пространство) |
| стек | таблица дескрипторов: открытые файлы, конвейеры, сокеты |
| TID | PID, родитель, пользователь, текущая папка |
| маска сигналов (Урок 13.7) | диспозиции сигналов (обработчики) |
| состояние планирования (работает, спит) | ограничения ресурсов |

Для ядра Linux нет глубокой разницы между потоком и процессом: оба — **задачи** с `task_struct`, оба создаются системным
вызовом `clone`. Разница во флагах: «делить память?», «делить таблицу дескрипторов?», «делить обработчики сигналов?». `fork`
(Урок 13.5) не делит ничего из этого; новый поток делит всё. Вот что запрашивает `std::thread::spawn`
(`strace -f -e trace=clone,clone3` программы из раздела 2):

```text
clone3({flags=CLONE_VM|CLONE_FS|CLONE_FILES|CLONE_SIGHAND|CLONE_THREAD|CLONE_SYSVSEM|CLONE_SETTLS|CLONE_PARENT_SETTID|CLONE_CHILD_CLEARTID, child_tid=0xffff8fa0f1f0, parent_tid=0xffff8fa0f1f0, exit_signal=0, stack=0xffff8f800000, stack_size=0x20ea00, tls=0xffff8fa0f840}, 88) = -1 ENOSYS (Function not implemented)
clone(child_stack=0xffff8fa0ea00, flags=CLONE_VM|CLONE_FS|CLONE_FILES|CLONE_SIGHAND|CLONE_THREAD|CLONE_SYSVSEM|CLONE_SETTLS|CLONE_PARENT_SETTID|CLONE_CHILD_CLEARTID ...
```

**Что происходит:**

1. `CLONE_VM` — делить память («виртуальную память»). `CLONE_FILES` — делить таблицу дескрипторов. `CLONE_FS` — делить текущую
   папку. `CLONE_SIGHAND` — делить обработчики сигналов.
2. `CLONE_THREAD` — поместить новую задачу в ту же **группу потоков**, чтобы у неё был тот же PID и она считалась частью этого
   процесса.
3. `stack=…, stack_size=0x20ea00` — библиотека C выделила для потока новый стек размером около 2 МиБ. `CLONE_SETTLS` даёт ему
   собственную локальную память потока (thread-local storage, где живут значения вроде `errno`, свои у каждого потока).
4. Как и в Уроке 13.5, `clone3` падает только потому, что его блокирует проверочный контейнер; библиотека C откатывается на
   `clone`. В вашей ВМ `clone3` срабатывает.

## 2. Как увидеть потоки: TID и `/proc/<pid>/task`

У каждого потока свой номер, **TID**. `getpid()` возвращает один и тот же PID во всех потоках; `gettid()` возвращает в каждом
свой TID. Каждый поток получает и папку `/proc/<pid>/task/<tid>/` со своими `status`, `stat` и `comm` (именем).
`std::thread::Builder::name` задаёт это имя. Создайте `src/bin/threads.rs`:

```rust
use nix::unistd::{getpid, gettid};
use std::{fs, thread, time::Duration};

fn main() {
    println!("main:     pid {} tid {}", getpid(), gettid());
    let mut handles = Vec::new();
    for i in 1..=3 {
        let handle = thread::Builder::new()
            .name(format!("worker-{i}"))
            .spawn(move || {
                println!("worker-{i}: pid {} tid {}", getpid(), gettid());
                thread::sleep(Duration::from_millis(300)); // stay alive while main looks
            })
            .unwrap();
        handles.push(handle);
    }
    thread::sleep(Duration::from_millis(100));

    // Every thread is a "task" of this process, with its own folder.
    let mut tasks: Vec<String> = fs::read_dir("/proc/self/task")
        .unwrap()
        .map(|e| e.unwrap().file_name().into_string().unwrap())
        .collect();
    tasks.sort();
    for tid in tasks {
        let comm = fs::read_to_string(format!("/proc/self/task/{tid}/comm")).unwrap();
        println!("  /proc/self/task/{tid}: {}", comm.trim());
    }
    for h in handles {
        h.join().unwrap();
    }
}
```

В разделе 5 этого урока используется фича `sched` крейта `nix`; добавьте её сейчас: `cargo add nix@0.32 --features sched`. Затем:

```bash
cargo build -q --release
./target/release/threads
```

```text
main:     pid 1174 tid 1174
worker-2: pid 1174 tid 1176
worker-1: pid 1174 tid 1175
worker-3: pid 1174 tid 1177
  /proc/self/task/1174: threads
  /proc/self/task/1175: worker-1
  /proc/self/task/1176: worker-2
  /proc/self/task/1177: worker-3
```

**Что происходит:**

1. Во всех четырёх строках PID 1174: один процесс. TID разные; TID главного потока равен PID.
2. `worker-2` напечатал раньше `worker-1`: когда потоки существуют, порядок решает планировщик, а не ваш код.
3. В `/proc/self/task` по одной папке на поток, а в `comm` — имя, заданное через `Builder::name` (главный поток назван по
   имени программы).

Потоки показывает и `ps -L`. Пока программа работает в фоне:

```bash
./target/release/threads > /dev/null & sleep 0.15; ps -L -o pid,lwp,nlwp,stat,comm -p $!
```

```text
    PID     LWP NLWP STAT COMMAND
      6       6    4 Sl   threads
      6       8    4 Sl   worker-1
      6       9    4 Sl   worker-2
      6      10    4 Sl   worker-3
```

`LWP` — это TID, `NLWP` — число потоков, а `l` в `Sl` означает «многопоточный». В `top` нажмите `H`, чтобы переключаться между
процессами и потоками.

> 💡 **Совет:** давайте потокам имена. Когда сервер загружает процессор на 100 %, `top -H` или `ps -L` покажут, **какой** именно
> поток это делает («db-writer», «http-3»), а не 20 строк с именем программы. Имена показывают также отладчики и сообщения о
> панике.

## 3. Планировщик: кто работает и как долго

У вашей ВМ несколько ядер процессора, а задач сотни. Большинство из них спит (состояние `S`), и процессор им не нужен. Готовые
к работе (состояние `R`) ждут в **очереди выполнения**; у каждого ядра своя. **Планировщик** выбирает следующую задачу из
очереди, когда:

- работающая задача блокируется (`read` из пустого конвейера, `sleep`, ожидание блокировки) — она отдаёт процессор
  **добровольно**;
- работающая задача израсходовала свой **квант времени**, а другая ждёт — прерывание таймера (Урок 13.1) позволяет ядру
  отобрать процессор: **вытеснение**, **принудительное** переключение;
- просыпается задача, у которой больше прав на процессор.

Переход от одной задачи к другой — это **переключение контекста**: ядро сохраняет регистры старой задачи в её `task_struct`,
загружает регистры новой, переключает стек и, если новая задача принадлежит другому процессу, ещё и карту памяти
(Урок 13.10). Переключение стоит порядка микросекунд, в основном потому, что кэши процессора (Урок 12.9) теперь хранят данные
старой задачи.

Ядро считает оба вида переключений для каждой задачи в `status`. Создайте `src/bin/switches.rs`:

```rust
use std::time::{Duration, Instant};
use std::{fs, thread};

/// Reads "voluntary_ctxt_switches" and "nonvoluntary_ctxt_switches" from a status text.
fn ctxt_switches(status: &str) -> Option<(u64, u64)> {
    let field = |key: &str| -> Option<u64> {
        status
            .lines()
            .find_map(|l| l.strip_prefix(key))?
            .trim()
            .parse()
            .ok()
    };
    Some((
        field("voluntary_ctxt_switches:")?,
        field("nonvoluntary_ctxt_switches:")?,
    ))
}

fn now() -> (u64, u64) {
    let status = fs::read_to_string("/proc/thread-self/status").unwrap();
    ctxt_switches(&status).unwrap()
}

fn main() {
    let before = now();
    for _ in 0..100 {
        thread::sleep(Duration::from_millis(1)); // give the CPU away 100 times
    }
    let after_sleep = now();
    let start = Instant::now();
    while start.elapsed() < Duration::from_secs(2) {} // never give the CPU away
    let after_spin = now();
    println!(
        "sleeping 100 times: +{} voluntary, +{} involuntary",
        after_sleep.0 - before.0,
        after_sleep.1 - before.1
    );
    println!(
        "spinning for 2 s:   +{} voluntary, +{} involuntary",
        after_spin.0 - after_sleep.0,
        after_spin.1 - after_sleep.1
    );
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn parses_both_counters() {
        let text = "Name:\tx\nvoluntary_ctxt_switches:\t12\nnonvoluntary_ctxt_switches:\t3\n";
        assert_eq!(ctxt_switches(text), Some((12, 3)));
    }

    #[test]
    fn missing_counter_gives_none() {
        assert_eq!(ctxt_switches("voluntary_ctxt_switches:\t12\n"), None);
    }
}
```

```bash
cargo build -q --release
./target/release/switches
taskset -c 0 ./target/release/switches
cargo test -q --bin switches
```

```text
sleeping 100 times: +100 voluntary, +0 involuntary
spinning for 2 s:   +0 voluntary, +14 involuntary
sleeping 100 times: +100 voluntary, +0 involuntary
spinning for 2 s:   +0 voluntary, +16266 involuntary

running 2 tests
..
test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

**Что происходит:**

1. `/proc/thread-self/status` — файл `status` **вызывающего потока** (`/proc/self` — это процесс).
2. Каждый `sleep` блокирует поток, поэтому он отдаёт процессор: ровно 100 добровольных переключений.
3. Цикл ожидания никогда не блокируется (`Instant::now()` выполняется в vDSO, Урок 13.3, поэтому нет даже системного вызова).
   При 12 свободных ядрах планировщику редко нужно его останавливать: 14 принудительных переключений за 2 секунды.
4. `taskset -c 0` разрешает программе работать **только на ядре 0**. Это ядро делят другие занятые задачи машины, поэтому
   теперь планировщик тысячи раз вытесняет наш цикл, чтобы дать очередь остальным. Ваши числа зависят от загрузки ВМ, но
   картина та же.

> ⚡ **Производительность:** много принудительных переключений означает «готовых задач больше, чем ядер»: процессор
> перегружен, и каждая задача ждёт в очереди. Много добровольных переключений означает, что задача часто блокируется —
> типично для программ, которые делают маленькие чтения и записи. Оба счётчика и `/proc/loadavg` из Урока 13.1 — первые числа,
> на которые стоит смотреть, когда «машина тормозит».

## 4. Честное деление: EEVDF, значения nice и веса

Как Linux решает, **кто** будет работать следующим? Обычные задачи используют политику `SCHED_OTHER`, которую начиная с Linux
6.6 реализует **EEVDF**. Идея, упрощённо:

1. У каждой задачи есть **вес**. Со временем каждая задача должна получить долю процессора, пропорциональную своему весу.
2. Ядро следит, сколько процессорного времени получила каждая задача по сравнению с её честной долей. Задача, получившая
   меньше своей доли, «допустима» (eligible).
3. Среди допустимых задач первой работает та, у которой следующий небольшой кусок работы имеет **самый ранний срок**. Задачи с
   короткими кусками (интерактивные программы) быстро получают процессор; задачи, которые расходуют весь квант (вычисления),
   сохраняют свою честную долю, но не больше.

Вес задаётся **значением nice**. У nice 0 вес 1024; каждый шаг nice меняет вес примерно на 25 %, поэтому у nice 10 вес 110,
а у nice −10 — 9548. Обычный пользователь может делать свои процессы только **вежливее** (увеличивать значение); уменьшать его
может только root.

Измерим это. `src/bin/spin.rs` считает, сколько работы он успевает сделать за три секунды:

```rust
use std::time::{Duration, Instant};

fn main() {
    let name = std::env::args()
        .nth(1)
        .unwrap_or_else(|| "spin".to_string());
    let start = Instant::now();
    let mut rounds: u64 = 0;
    while start.elapsed() < Duration::from_secs(3) {
        for _ in 0..10_000 {
            rounds = std::hint::black_box(rounds + 1); // real work the compiler cannot remove
        }
    }
    println!("{name}: {} million rounds in 3 s", rounds / 1_000_000);
}
```

Запустите его один раз в одиночку, затем две копии на **одном** ядре (через `taskset -c 0`): один раз с равными значениями
nice, другой — nice 0 против nice 10:

```bash
cargo build -q --release
./target/release/spin alone
taskset -c 0 ./target/release/spin A & taskset -c 0 ./target/release/spin B & wait
taskset -c 0 ./target/release/spin nice0 & taskset -c 0 nice -n 10 ./target/release/spin nice10 & wait
```

```text
alone: 11344 million rounds in 3 s
B: 5751 million rounds in 3 s
A: 5696 million rounds in 3 s
nice0: 10377 million rounds in 3 s
nice10: 1152 million rounds in 3 s
```

**Что происходит:**

1. `std::hint::black_box` говорит компилятору «сделай вид, что не знаешь это значение», поэтому он не может заменить цикл
   умножением; `--release` делает цикл быстрым (Урок 05.12).
2. В одиночку программа получает целое ядро. Две одинаковые программы на одном ядре получают по половине: 5751 и 5696, очень
   честно.
3. При nice 0 против nice 10 доли следуют весам: 1024 : 110 — это примерно 9,3 : 1; замер показал 10377 : 1152 = 9,0 : 1.
4. Сумма немного меньше, чем «в одиночку», из-за переключений контекста между ними.

Для этого эксперимента обе программы нужно запускать из одного терминала: Linux объединяет процессы каждого терминального
сеанса в группу («autogroup») и сначала делит процессор между группами, а уже потом внутри них.

> 📏 **Соглашение:** запускайте долгие фоновые задания (резервные копии, большие сборки, пакетную обработку) с `nice -n 10`
> или выше. Они всё равно используют каждый свободный такт процессора, но интерактивные программы и службы получают процессор
> первыми, когда он им нужен.

Есть и политики **реального времени**, `SCHED_FIFO` и `SCHED_RR`: задача реального времени всегда работает раньше любой
обычной задачи. Им нужен root (`chrt -f 50 program`), а задача реального времени в бесконечном цикле может заморозить ядро,
поэтому они только для особых программ вроде обработки звука или систем управления. `chrt -p PID` показывает политику процесса;
у обычных это `SCHED_OTHER`.

## 5. Привязка к процессору: закрепляем поток за ядром

По умолчанию поток может работать на любом ядре, и планировщик переносит его между ядрами, чтобы выровнять нагрузку.
**Привязка к процессору** ограничивает его набором ядер: `taskset -c 0` делал это снаружи. Изнутри это делает
`nix::sched::sched_setaffinity`. Создайте `src/bin/affinity.rs`:

```rust
use nix::sched::{CpuSet, sched_getaffinity, sched_getcpu, sched_setaffinity};
use nix::unistd::Pid;

fn main() -> nix::Result<()> {
    let allowed = sched_getaffinity(Pid::from_raw(0))?; // 0 means "this thread"
    let cpus: Vec<usize> = (0..CpuSet::count())
        .filter(|&c| allowed.is_set(c).unwrap_or(false))
        .collect();
    println!("allowed CPUs: {cpus:?}, running on CPU {}", sched_getcpu()?);

    let mut only_cpu_2 = CpuSet::new();
    only_cpu_2.set(2)?;
    sched_setaffinity(Pid::from_raw(0), &only_cpu_2)?;
    println!("after pinning: running on CPU {}", sched_getcpu()?);
    println!(
        "available_parallelism now says {}",
        std::thread::available_parallelism().unwrap()
    );
    Ok(())
}
```

```bash
./target/release/affinity
taskset -c 0-3 ./target/release/affinity
```

```text
allowed CPUs: [0, 1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11], running on CPU 0
after pinning: running on CPU 2
available_parallelism now says 1
allowed CPUs: [0, 1, 2, 3], running on CPU 0
after pinning: running on CPU 2
available_parallelism now says 1
```

**Что происходит:**

1. `sched_getaffinity` возвращает `CpuSet` — набор битов, по одному на ядро. `Pid::from_raw(0)` означает «вызывающий поток».
2. `taskset -c 0-3` запустил программу с разрешёнными ядрами 0–3; привязка наследуется через `fork` и `exec`.
3. После `sched_setaffinity` ядро сразу переносит поток на ядро 2; `sched_getcpu` это подтверждает.
4. `available_parallelism` учитывает привязку: теперь она говорит 1. Поэтому именно она, а не `/proc/cpuinfo`, даёт правильный
   ответ на вопрос «сколько потоков мне запускать?» (Урок 13.1, упражнение 1).

Привязка помогает в особых случаях: держать поток, чувствительный к задержкам, подальше от шумных соседей или держать поток
рядом с его данными в больших многопроцессорных серверах. В большинстве программ пусть решает планировщик; он балансирует лучше
жёсткого правила.

## Частые ошибки

> ⚠️ **Частая ошибка:** ожидать, что потоки работают в том порядке, в котором запущены. Решает планировщик, и порядок меняется от
> запуска к запуску (`worker-2` раньше `worker-1` выше). Если порядок важен, используйте `join`, каналы или блокировки (Этап 14).

> ⚠️ **Частая ошибка:** запускать по потоку на каждую задачу, когда задач тысячи. У каждого потока свой стек (по умолчанию 2 МиБ
> адресного пространства у потоков `std`), и он стоит переключений контекста. Фиксированный пул примерно из
> `available_parallelism()` рабочих потоков или асинхронные задачи (Этап 14) масштабируются гораздо лучше.

> ⚠️ **Частая ошибка:** измерять скорость непривязанной программой на загруженной машине и верить одному запуску. Другие задачи
> отнимают процессорное время, и числа скачут. Повторяйте измерения и смотрите на принудительные переключения, чтобы понять, не
> вытесняли ли вас.

## Попробуйте сами

1. Запустите `threads` и, пока он работает, выполните во втором терминале `ls /proc/$(pgrep -x threads)/task`.
2. Запустите `./target/release/spin x &` в четыре раза больше раз, чем у вас ядер (`nproc`), и посмотрите `top`: каждая копия
   получает примерно `100 % × ядра / копии` процессора.
3. Выполните `nice -n 19 ./target/release/spin low & ./target/release/spin normal & wait` **без** `taskset`. Почему теперь числа
   почти равны? (Подсказка: каждая копия находит свободное ядро.)
4. Выполните `chrt -p $$` в ВМ. Какую политику использует ваша оболочка?

## Упражнения

### Упражнение 1 — Какой поток занят? (легко)

Напишите `src/bin/busy-thread.rs`, который запускает два именованных потока: `idle` спит 5 секунд, `burner` 5 секунд крутится в
цикле. Пока программа работает, найдите занятый поток через `top -H -p <pid>` или `ps -L -o lwp,pcpu,comm -p <pid>`.

<details>
<summary>Подсказка</summary>

Используйте `thread::Builder::new().name("burner".into()).spawn(...)`, а внутри цикл `while start.elapsed() <
Duration::from_secs(5) {}`. `pcpu` — процент процессора.

</details>

<details>
<summary>Ответ</summary>

```rust
use std::thread;
use std::time::{Duration, Instant};

fn main() {
    println!("pid {}", std::process::id());
    let idle = thread::Builder::new()
        .name("idle".into())
        .spawn(|| thread::sleep(Duration::from_secs(5)))
        .unwrap();
    let burner = thread::Builder::new()
        .name("burner".into())
        .spawn(|| {
            let start = Instant::now();
            while start.elapsed() < Duration::from_secs(5) {}
        })
        .unwrap();
    idle.join().unwrap();
    burner.join().unwrap();
}
```

```bash
./target/release/busy-thread & sleep 2; ps -L -o lwp,pcpu,comm -p $!; wait
```

```text
pid 105
    LWP %CPU COMMAND
    105  0.0 busy-thread
    107  0.0 idle
    108  100 burner
```

Почему это работает: ядро учитывает процессорное время для каждой задачи, поэтому числа по потокам получаются даром. Имена
делают ответ очевидным.

</details>

### Упражнение 2 — Тест для счётчиков (средне)

У функции `ctxt_switches` в `switches.rs` уже есть два теста. Добавьте третий тест, который проверяет текст status, где у значений
лишние пробелы, а две строки идут в обратном порядке, и четвёртый — для значения, которое не является числом. Добейтесь, чтобы
проходили все четыре.

<details>
<summary>Подсказка</summary>

`find_map` просматривает каждую строку, поэтому порядок строк не важен. `trim()` убирает пробелы и табуляции вокруг числа. Для
значения вроде `"abc"` `parse()` не удаётся, `.ok()` даёт `None`, а `?` заставляет всю функцию вернуть `None`.

</details>

<details>
<summary>Ответ</summary>

```rust
    #[test]
    fn order_and_spaces_do_not_matter() {
        let text = "nonvoluntary_ctxt_switches:   7  \nvoluntary_ctxt_switches:\t  40\n";
        assert_eq!(ctxt_switches(text), Some((40, 7)));
    }

    #[test]
    fn a_broken_value_gives_none() {
        let text = "voluntary_ctxt_switches:\tabc\nnonvoluntary_ctxt_switches:\t1\n";
        assert_eq!(ctxt_switches(text), None);
    }
```

```text
running 4 tests
....
test result: ok. 4 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

Почему это работает: существующую функцию менять не пришлось — тесты доказывают, что она уже справляется с этими случаями. Такие
тесты — дешёвая страховка перед тем, как вы позже измените парсер.

</details>

### Упражнение 3 — Сколько стоит поток? (сложно)

Напишите `src/bin/spawn-cost.rs`, который с помощью `std::time::Instant` измеряет среднее время `thread::spawn(|| {}).join()` и
`fork` + `waitpid` ребёнка, который сразу завершается, по 1000 раз каждое. Соберите с `--release`, запустите дважды и объясните
разницу.

<details>
<summary>Подсказка</summary>

`Duration` можно делить на `u32`: `start.elapsed() / 1000`. В ребёнке вызывайте `libc::_exit(0)` (Урок 13.5). Сначала
измерьте потоки и дождитесь (join) каждого потока до fork, чтобы при каждом `fork` у программы был только один поток.

</details>

<details>
<summary>Ответ</summary>

```rust
use nix::sys::wait::waitpid;
use nix::unistd::{ForkResult, fork};
use std::thread;
use std::time::Instant;

const N: u32 = 1000;

fn main() {
    let start = Instant::now();
    for _ in 0..N {
        thread::spawn(|| {}).join().unwrap();
    }
    let per_thread = start.elapsed() / N;

    let start = Instant::now();
    for _ in 0..N {
        // SAFETY: when we fork, the only other thread has already been joined.
        match unsafe { fork() }.unwrap() {
            ForkResult::Child => unsafe { libc::_exit(0) },
            ForkResult::Parent { child } => {
                waitpid(child, None).unwrap();
            }
        }
    }
    let per_process = start.elapsed() / N;

    println!("thread  spawn + join: {per_thread:?} each");
    println!("process fork + wait:  {per_process:?} each");
}
```

Два запуска:

```text
thread  spawn + join: 97.789µs each
process fork + wait:  262.615µs each
thread  spawn + join: 47.034µs each
process fork + wait:  149.053µs each
```

Почему это работает: поток делит карту памяти и таблицу дескрипторов, поэтому ядро создаёт лишь `task_struct`, а библиотека C —
стек. `fork` должен ещё скопировать таблицы страниц и таблицу дескрипторов родителя (Урок 13.10), а завершение ребёнка — их
разобрать. Числа меняются от запуска к запуску (первый запуск ещё и «прогревал» кэши), поэтому всегда измеряйте больше одного
раза. И то и другое слишком медленно, чтобы делать это на каждый запрос в быстром сервере, — поэтому серверы переиспользуют
потоки в пуле.

</details>

## Вопросы для самопроверки

1. Что потоки одного процесса делят, а что у каждого своё?

<details>
<summary>Ответ</summary>

Они делят память (код, кучу, static), таблицу дескрипторов, PID, текущую папку и обработчики сигналов. У каждого свои регистры,
стек, TID и маска сигналов.

</details>

2. Как Linux создаёт поток и чем это отличается от `fork`?

<details>
<summary>Ответ</summary>

Через `clone` (или `clone3`) с флагами вроде `CLONE_VM`, `CLONE_FILES`, `CLONE_SIGHAND` и `CLONE_THREAD`, благодаря которым новая
задача делит память, файлы и обработчики и входит в ту же группу потоков. `fork` — это `clone` без этих флагов, поэтому ребёнок
получает копии.

</details>

3. Чем добровольное переключение контекста отличается от принудительного?

<details>
<summary>Ответ</summary>

Добровольное: задача заблокировалась (sleep, ввод-вывод, ожидание блокировки) и сама отдала процессор. Принудительное:
планировщик её вытеснил, потому что её квант времени кончился, а другая задача ждала.

</details>

4. Две программы, нагружающие процессор, делят одно ядро; одна работает с nice 0, другая с nice 10. Как примерно делится процессор?

<details>
<summary>Ответ</summary>

По весам: 1024 против 110, то есть примерно 9 : 1. Программа с nice 0 получает около 90 % ядра.

</details>

5. Почему `available_parallelism()` лучше подходит для выбора размера пула потоков, чем число строк в `/proc/cpuinfo`?

<details>
<summary>Ответ</summary>

Потому что она учитывает привязку к процессору (и ограничения процессора у контейнера): если процессу разрешено работать только
на двух ядрах, она скажет 2, а `/proc/cpuinfo` всё равно перечислит все ядра машины.

</details>

6. Что меняет `CLONE_THREAD` по сравнению с `clone` только с `CLONE_VM`?

<details>
<summary>Ответ</summary>

Он помещает новую задачу в группу потоков вызывающего: она делит PID (TGID), не видна в `ps` как отдельный процесс, и когда один
поток вызывает `exit_group` (или процесс получает смертельный сигнал), все потоки завершаются вместе.

</details>

## Итоги

- Поток — задача, которая делит память, файлы и обработчики сигналов с другими потоками своего процесса; у каждого потока свои регистры, стек, TID и маска сигналов.
- Linux создаёт потоки и процессы одним системным вызовом `clone`; флаги решают, что делится.
- `/proc/<pid>/task/<tid>/`, `ps -L` и `top -H` показывают потоки; давайте им имена через `thread::Builder::name`.
- Планировщик держит готовые задачи в очередях выполнения своих ядер и переключается между ними при блокировке (добровольно) или при вытеснении таймером (принудительно).
- Обычные задачи планирует EEVDF, честно по весам; значение nice (от −20 до 19) задаёт вес.
- Привязка к процессору (`taskset`, `sched_setaffinity`) ограничивает поток некоторыми ядрами; `available_parallelism` её учитывает.
- Создать поток дешевле, чем процесс, но и то и другое не бесплатно: переиспользуйте потоки.

## Источники и что почитать

- [clone(2) — Linux manual page](https://man7.org/linux/man-pages/man2/clone.2.html) — каждый флаг, определяющий, что делит новая задача.
- [sched(7) — Linux manual page](https://man7.org/linux/man-pages/man7/sched.7.html) — политики планирования, значения nice, планирование реального времени и autogroup.
- [EEVDF Scheduler — The Linux Kernel documentation](https://docs.kernel.org/scheduler/sched-eevdf.html) — собственное описание EEVDF от ядра.
- [sched_setaffinity(2) — Linux manual page](https://man7.org/linux/man-pages/man2/sched_setaffinity.2.html) — привязка к процессору.
- [std::thread — Rust documentation](https://doc.rust-lang.org/std/thread/index.html) — `spawn`, `Builder`, имена, размеры стека и `available_parallelism`.
- [Operating Systems: Three Easy Pieces — Scheduling: Proportional Share](https://pages.cs.wisc.edu/~remzi/OSTEP/cpu-sched-lottery.pdf) — идея деления процессора по весам.

---

[🏠 Начало](../START-HERE.md) · [← 13.8 Проект: собственная командная оболочка](08-project-your-own-shell.md) · [📋 Этап 13](README.md) · [13.10 Виртуальная память →](10-virtual-memory.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
