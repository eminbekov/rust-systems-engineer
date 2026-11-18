<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 22.7 Трек eBPF: первая программа на Aya](07-ebpf-track-a-first-program-with-aya.md) · [📋 Этап 22](README.md) · [22.9 Трек eBPF: проект и следующие шаги →](09-ebpf-track-project-and-next-steps.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/22-specializations/08-ebpf-track-maps-and-tracing.md) <!-- nav -->

# Урок 22.8 — Трек eBPF: карты (maps) и трассировка

> **Этап 22 — Специализации** · Урок 8 из 30 · ⏱ около 135 минут

## Что вы узнаете

- Главные виды карт eBPF и как объявлять и использовать их на обеих сторонах с Aya.
- Как разделить тип данных между стороной ядра и стороной пользователя в общем крейте.
- Как читать аргументы tracepoint и почему память процесса нужно читать вспомогательной функцией.
- Как отправлять события в пространство пользователя через кольцевой буфер.
- Как подключить kprobe к функции ядра и почему карты по процессорам (per-CPU) делают подсчёт дешёвым и правильным.

## Перед началом

- Вы прошли [Урок 22.7 — Трек eBPF: первая программа на Aya](07-ebpf-track-a-first-program-with-aya.md); рабочее
  пространство `code/p22-tracer` собирается и запускается в ВМ.
- Вы знаете файл формата tracepoint из [Урока 22.6](06-ebpf-track-what-ebpf-is.md) и `#[repr(C)]` из
  [Урока 12.11 — Размещение типов Rust в памяти](../12-computer-systems/11-memory-layout-of-rust-types.md).

## Новые слова

| Слово | Значение |
|---|---|
| **хеш-таблица** (hash map) eBPF | Карта из ключей в значения с фиксированным максимальным числом записей, например «имя команды → счётчик». |
| **карта-массив** (array map) | Карта, ключи которой — числа `0..N`; все записи существуют с самого начала. |
| **карта по процессорам** (per-CPU map) | Карта, в которой у каждого процессора своя копия каждого значения; пространство пользователя складывает копии. |
| **кольцевой буфер** (ring buffer) eBPF | Карта, которая работает как очередь записей переменного размера из ядра в пространство пользователя. |
| **событие** (event) | Одна запись о том, что произошло, отправленная со стороны ядра на сторону пользователя. |
| **probe read** (чтение через зонд) | Копирование памяти вспомогательной функцией (`bpf_probe_read_*`), которая при плохом адресе безопасно возвращает ошибку, а не падает. |
| **память пользователя / память ядра** (user memory / kernel memory) | Память процесса (где `cat` хранит имя файла) / память самого ядра. eBPF читает их разными вспомогательными функциями. |
| **`#[repr(C)]`** | «Размести эту структуру так, как это делает C»: фиксированный порядок полей, чтобы обе стороны совпадали в каждом байте (Урок 12.11). |
| **kprobe** | Точка подключения в начале функции ядра (Урок 22.6); здесь `vfs_read`, которая обрабатывает каждый `read`. |
| **`vfs_read`** | Функция ядра за системным вызовом `read`. VFS (Virtual File System, виртуальная файловая система) — слой, общий для всех файловых систем. |

**Сокращения из прошлых уроков:** BPF (Berkeley Packet Filter, пакетный фильтр Беркли — имя, из которого вырос eBPF),
ВМ (виртуальная машина; Linux-ВМ `dev`), CPU (Central Processing Unit, центральный процессор) и IP (Internet
Protocol, межсетевой протокол; IP-адрес обозначает машину в сети, Урок 15.2).

## 1. Карты: память между ядром и пространством пользователя

Программа eBPF не может хранить данные в глобальной переменной от одного запуска до другого и не может передать
данные в пространство пользователя напрямую. Всё, что должно сохраниться или куда-то попасть, идёт в **карту**. У
каждой карты есть тип, тип ключа, тип значения и максимальный размер, и всё это фиксируется при создании. Самые
полезные типы:

| Тип карты | Тип Aya (обе стороны) | Типичное применение |
|---|---|---|
| хеш-таблица | `HashMap<K, V>` | счётчики или состояние по ключу: по процессу, по IP-адресу |
| хеш-таблица по процессорам | `PerCpuHashMap<K, V>` | то же, но без борьбы процессоров за данные |
| массив | `Array<V>` | настройки, несколько фиксированных ячеек |
| массив по процессорам | `PerCpuArray<V>` | быстрые глобальные счётчики (Урок 22.9) |
| кольцевой буфер | `RingBuf` | поток событий в пространство пользователя |

На стороне ядра карта — это `static` с атрибутом `#[map]`; макрос кладёт её описание в секцию `maps` объектного
файла, а `Ebpf::load` создаёт её в ядре. На стороне пользователя вы получаете дескриптор по имени через
`ebpf.map("NAME")` (заимствовать) или `ebpf.take_map("NAME")` (владеть, например чтобы переместить в задачу) и
превращаете его в типизированную обёртку через `try_from`.

В этом уроке трассировщик из Урока 22.7 превращается в маленький инструмент с двумя программами:

- программа **tracepoint** на `sys_enter_openat`, которая отправляет по одному событию на каждый открытый файл через
  кольцевой буфер;
- программа **kprobe** на `vfs_read`, которая считает чтения по командам в хеш-таблице по процессорам.

Сторона пользователя печатает события по мере поступления, а при нажатии Ctrl-C — самые большие счётчики.

## 2. Общий тип в общем крейте

Обе стороны должны совпадать в размещении события, байт в байт. Поместите тип в третий крейт, от которого зависят
обе. Создайте `tracer-common/Cargo.toml`:

```toml
[package]
name = "tracer-common"
version = "0.1.0"
edition.workspace = true

[dependencies]
```

И `tracer-common/src/lib.rs`:

```rust
//! Types shared by the eBPF program and the user-space program.
#![cfg_attr(not(test), no_std)]

/// One `openat` call, sent from the kernel through the ring buffer.
#[repr(C)]
#[derive(Clone, Copy)]
pub struct OpenEvent {
    pub pid: u32,
    pub comm: [u8; 16],
    pub filename: [u8; 64],
}

/// The text before the first zero byte (C strings end with a zero byte).
pub fn c_str(bytes: &[u8]) -> &str {
    let end = bytes.iter().position(|&b| b == 0).unwrap_or(bytes.len());
    core::str::from_utf8(&bytes[..end]).unwrap_or("<not utf-8>")
}

/// With no filter every command is wanted; otherwise only the named one.
pub fn wanted(filter: Option<&str>, comm: &str) -> bool {
    match filter {
        None => true,
        Some(name) => name == comm,
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn stops_at_the_first_zero_byte() {
        assert_eq!(c_str(b"cat\0\0\0"), "cat");
    }

    #[test]
    fn takes_everything_without_a_zero_byte() {
        assert_eq!(c_str(b"systemd"), "systemd");
        assert_eq!(c_str(b""), "");
    }
}
```

**Что происходит:**

1. `#![cfg_attr(not(test), no_std)]`: крейт — `no_std`, чтобы его могла использовать сторона eBPF, но его тесты
   работают со `std` на любой машине (тот же приём, что в Уроке 22.2).
2. `#[repr(C)]` фиксирует порядок и выравнивание полей. Без него компилятор может переставить поля, и две стороны,
   собранные разными тулчейнами для разных платформ, могли бы разойтись.
3. Внутри только массивы фиксированного размера: размер события должен быть известен при компиляции, потому что
   сторона ядра резервирует ровно `size_of::<OpenEvent>()` байтов (здесь 84).
4. `c_str` и `wanted` — чистые вспомогательные функции для стороны пользователя; они живут здесь, чтобы их можно было
   тестировать где угодно.

Добавьте крейт в рабочее пространство: в корневом `Cargo.toml` списки участников становятся такими

```toml
members = ["tracer", "tracer-common", "tracer-ebpf"]
default-members = ["tracer", "tracer-common"]
```

а `tracer/Cargo.toml` и `tracer-ebpf/Cargo.toml` получают строку зависимости
`tracer-common = { path = "../tracer-common" }` в `[dependencies]`. Эта версия трассировщика больше не использует
`aya-log`, поэтому удалите строки `aya-log`, `aya-log-ebpf`, `env_logger` и `log` из рабочего пространства и из обоих
крейтов, а строку Tokio замените на `tokio = { version = "1", features = ["macros", "rt-multi-thread", "signal",
"time"] }` (сторона пользователя теперь ждёт через `tokio::time`).

## 3. Читаем аргументы tracepoint

Контекст tracepoint указывает на **запись**, описанную файлом формата. Из Урока 22.6:

```text
	field:const char * filename;	offset:24;	size:8;	signed:0;
```

Значит, в 24 байтах от начала записи лежит 8-байтовый **указатель** на имя файла. Его читают в два шага:

1. `ctx.read_at::<*const u8>(24)` копирует сам указатель из записи (память ядра).
2. Указатель ведёт в память процесса, который вызвал `openat`, — **память пользователя**, которую ядро не может
   просто разыменовать: страница может быть вытеснена, а адрес — неверным. Вспомогательная функция
   `bpf_probe_read_user_str_bytes(ptr, &mut buffer)` копирует оттуда строку, заканчивающуюся нулём, не больше
   `buffer.len()` байтов, и при плохом адресе возвращает ошибку вместо падения.

Верификатор следит за этим. Если написать `*filename` вместо вспомогательной функции, загрузка падает:

```text
; (*event).filename[0] = *filename; // read user memory directly: wrong!
124: (71) r1 = *(u8 *)(r8 +0)
R8 invalid mem access 'scalar'
```

Верификатор отслеживает, что лежит в каждом регистре. В `r8` число, пришедшее из `read_at`, — для верификатора просто
«скаляр», а не указатель, которому он доверяет, — поэтому чтение памяти через него запрещено.

## 4. Сторона ядра

Замените `tracer-ebpf/src/main.rs`:

```rust
#![no_std]
#![no_main]

use aya_ebpf::{
    EbpfContext,
    helpers::bpf_probe_read_user_str_bytes,
    macros::{kprobe, map, tracepoint},
    maps::{PerCpuHashMap, RingBuf},
    programs::{ProbeContext, TracePointContext},
};
use tracer_common::OpenEvent;

/// Events for user space: 256 KiB of space in the kernel.
#[map]
static EVENTS: RingBuf = RingBuf::with_byte_size(256 * 1024, 0);

/// How often each command called `vfs_read`, keyed by command name.
/// Every CPU has its own counters, so no two CPUs write the same value.
#[map]
static READS: PerCpuHashMap<[u8; 16], u64> = PerCpuHashMap::with_max_entries(1024, 0);

/// Offset of `filename` in the record of the `sys_enter_openat` tracepoint
/// (see /sys/kernel/tracing/events/syscalls/sys_enter_openat/format).
const FILENAME_OFFSET: usize = 24;

#[tracepoint]
pub fn openat(ctx: TracePointContext) -> u32 {
    let _ = try_openat(&ctx);
    0
}

fn try_openat(ctx: &TracePointContext) -> Result<(), i32> {
    // SAFETY: the format file says that a pointer is stored at this offset.
    let filename: *const u8 = unsafe { ctx.read_at(FILENAME_OFFSET)? };
    let Some(mut entry) = EVENTS.reserve::<OpenEvent>(0) else {
        return Ok(()); // the ring buffer is full: drop this event
    };
    let event = entry.as_mut_ptr();
    // SAFETY: `event` points to our reserved space in the ring buffer.
    unsafe {
        (*event).pid = ctx.tgid();
        (*event).comm = ctx.command().unwrap_or([0; 16]);
        if bpf_probe_read_user_str_bytes(filename, &mut (*event).filename).is_err() {
            (*event).filename[0] = 0;
        }
    }
    entry.submit(0);
    Ok(())
}

#[kprobe]
pub fn count_reads(ctx: ProbeContext) -> u32 {
    let Ok(comm) = ctx.command() else { return 0 };
    match READS.get_ptr_mut(&comm) {
        // SAFETY: the pointer points to this CPU's value inside the map.
        Some(count) => unsafe { *count += 1 },
        None => {
            let _ = READS.insert(&comm, &1, 0);
        }
    }
    0
}

#[cfg(not(test))]
#[panic_handler]
fn panic(_info: &core::panic::PanicInfo) -> ! {
    loop {}
}

#[unsafe(link_section = "license")]
#[unsafe(no_mangle)]
static LICENSE: [u8; 13] = *b"Dual MIT/GPL\0";
```

**Что происходит:**

1. Две карты объявлены как статические переменные. `RingBuf::with_byte_size(256 * 1024, 0)` просит 256 КиБ (размер
   должен быть числом страниц, равным степени двойки). Хеш-таблица по процессорам вмещает до 1024 команд.
2. `EVENTS.reserve::<OpenEvent>(0)` резервирует место для одного события **внутри самого кольцевого буфера**. Если
   буфер полон (пространство пользователя читает слишком медленно), возвращается `None`, и событие отбрасывается; ядро
   никогда не ждёт.
3. Мы заполняем зарезервированное место на месте через сырой указатель и вызываем `submit`, после чего событие
   становится видно пространству пользователя. Каждую зарезервированную запись нужно отправить или отменить на любом
   пути; верификатор проверяет и это.
4. `ctx.tgid()` — ID процесса, который вы знаете по `ps` («thread group ID», ID группы потоков); `ctx.pid()` был бы ID
   отдельного потока.
5. `count_reads` — программа kprobe: `ProbeContext` даёт доступ к регистрам функции-зонда и ту же вспомогательную
   функцию `command()`. Она выполняется на **каждый** `read` каждого процесса, поэтому должна быть крошечной.
6. В карте по процессорам `get_ptr_mut` возвращает указатель на значение **этого процессора**. Программа eBPF никогда
   не переносится на другой процессор во время выполнения, поэтому `*count += 1` не может вызвать гонку с другим
   процессором. В обычной хеш-таблице два процессора могли бы прочитать одно старое значение и оба записать
   `old + 1`, потеряв один отсчёт.
7. Когда команда встречается впервые, `insert` создаёт её запись со значением 1.

## 5. Сторона пользователя

Замените `tracer/src/main.rs`:

```rust
use std::time::Duration;

use aya::maps::{PerCpuHashMap, RingBuf};
use aya::programs::{KProbe, TracePoint};
use tracer_common::{OpenEvent, c_str, wanted};

#[tokio::main]
async fn main() -> anyhow::Result<()> {
    // `tracer cat` shows only what programs named `cat` do.
    let only = std::env::args().nth(1);

    let mut ebpf = aya::Ebpf::load(aya::include_bytes_aligned!(concat!(
        env!("OUT_DIR"),
        "/tracer"
    )))?;

    let openat: &mut TracePoint = ebpf.program_mut("openat").unwrap().try_into()?;
    openat.load()?;
    openat.attach("syscalls", "sys_enter_openat")?;

    let reads: &mut KProbe = ebpf.program_mut("count_reads").unwrap().try_into()?;
    reads.load()?;
    reads.attach("vfs_read", 0)?;

    let mut events = RingBuf::try_from(ebpf.take_map("EVENTS").unwrap())?;
    let filter = only.clone();
    tokio::spawn(async move {
        loop {
            while let Some(item) = events.next() {
                // SAFETY: the eBPF program writes exactly one OpenEvent per item.
                let event = unsafe { (item.as_ptr() as *const OpenEvent).read_unaligned() };
                let comm = c_str(&event.comm);
                if wanted(filter.as_deref(), comm) {
                    println!("{:>7} {:<12} {}", event.pid, comm, c_str(&event.filename));
                }
            }
            tokio::time::sleep(Duration::from_millis(50)).await;
        }
    });

    println!("Tracing openat and vfs_read. Press Ctrl-C to stop.");
    tokio::signal::ctrl_c().await?;

    let counts: PerCpuHashMap<_, [u8; 16], u64> =
        PerCpuHashMap::try_from(ebpf.map("READS").unwrap())?;
    let mut top: Vec<(u64, String)> = counts
        .iter()
        .filter_map(Result::ok)
        .map(|(comm, per_cpu)| (per_cpu.iter().sum(), c_str(&comm).to_string()))
        .filter(|(_, comm)| wanted(only.as_deref(), comm))
        .collect();
    top.sort_unstable_by(|a, b| b.cmp(a));
    println!("\nvfs_read calls by command:");
    for (n, comm) in top.iter().take(5) {
        println!("{n:>8} {comm}");
    }
    Ok(())
}
```

**Что происходит:**

1. Две программы загружаются и подключаются отдельно. `KProbe::attach("vfs_read", 0)` означает «функция ядра
   `vfs_read`, смещение 0» — её первая инструкция.
2. `take_map("EVENTS")` забирает кольцевой буфер во владение, чтобы его можно было переместить в запущенную задачу.
   `RingBuf::next()` возвращает следующую запись или `None`, когда буфер пуст; тогда задача спит 50 мс и проверяет
   снова.
3. Запись — срез байтов. `read_unaligned` копирует из неё `OpenEvent`; `unsafe` оправдан, потому что сторона ядра
   всегда пишет ровно этот тип.
4. После Ctrl-C `ebpf.map("READS")` заимствует карту счётчиков. Каждое значение — `PerCpuValues<u64>`: по числу на
   процессор, которые мы складываем через `.iter().sum()`.
5. Необязательный аргумент командной строки фильтрует оба вывода через `wanted`.

## 6. Запускаем

Скопируйте рабочее пространство в ВМ, как в Уроке 22.7, затем:

```bash
cargo build --release
cargo test -p tracer-common
cargo run --release -- cat
```

Во втором терминале ВМ выполните `cat /etc/hostname > /dev/null` и `cat /etc/os-release > /dev/null`, затем нажмите
Ctrl-C в первом:

```text
running 2 tests
test tests::stops_at_the_first_zero_byte ... ok
test tests::takes_everything_without_a_zero_byte ... ok

test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

```text
Tracing openat and vfs_read. Press Ctrl-C to stop.
  50541 cat          /etc/ld.so.cache
  50541 cat          /lib/aarch64-linux-gnu/libc.so.6
  50541 cat          /etc/hostname
  50542 cat          /etc/ld.so.cache
  50542 cat          /lib/aarch64-linux-gnu/libc.so.6
  50542 cat          /etc/os-release

vfs_read calls by command:
       6 cat
```

Этот вывод получен в контейнере Debian с классическим `cat`; `cat` в Ubuntu 26.04 открывает больше файлов (вы видели
список в Уроке 22.6). Без аргумента `cat` инструмент показывает каждый процесс на машине — попробуйте на секунду, и вы
увидите, как занята даже простаивающая система.

> ⚡ **Производительность:** `count_reads` выполняется на каждый `read` всей машины. Поэтому она почти ничего не
> делает: один вызов вспомогательной функции, один поиск в карте, одно сложение. Никогда не отправляйте событие на
> каждый `read` в пространство пользователя; считайте в ядре и читайте итоги. Карта по процессорам ещё и избавляет от
> «пинг-понга» кэш-линии между процессорами, который вызывает общий счётчик (Урок 12.10).

> 🔒 **Безопасность:** этот инструмент видит имена файлов, которые открывает каждый пользователь машины. Данные
> трассировки чувствительны: оставляйте такие инструменты только для root и не отправляйте их вывод туда, куда не
> отправили бы сами файлы.

## Частые ошибки

> ⚠️ **Частая ошибка:** разыменовывать указатель из записи tracepoint (`*filename`). Верификатор отклоняет это с
> `invalid mem access 'scalar'`. Читайте память пользователя через `bpf_probe_read_user_str_bytes` (строки) или
> `bpf_probe_read_user` (значения фиксированного размера).

> ⚠️ **Частая ошибка:** забыть `#[repr(C)]` на общем типе события. Обе стороны компилируются, но сторона пользователя
> может читать поля не по тем смещениям и печатать мусор.

> ⚠️ **Частая ошибка:** взять смещение не из того файла формата (`sys_enter_open` вместо `sys_enter_openat` или с
> другого ядра). Ничего не падает; вы просто читаете не те 8 байтов. Всегда проверяйте
> `/sys/kernel/tracing/events/.../format` на целевом ядре.

> ⚠️ **Частая ошибка:** обычная `HashMap` с `get` + `insert(count + 1)` для счётчиков. На машине со многими
> процессорами счётчики получаются слишком маленькими, потому что два процессора могут обновлять одно значение
> одновременно. Используйте карту по процессорам и складывайте значения в пространстве пользователя.

## Попробуйте сами

1. Создайте `tracer-common`, обновите манифесты и замените оба `main.rs`, как показано.
2. Запустите `cargo run --release -- cat` и создавайте события из второго терминала.
3. Запустите без аргумента на две секунды и посмотрите на пятёрку самых активных читателей.
4. Сделайте коммит: `git add -A && git commit -m "p22-tracer: ring buffer events and kprobe counters"`.

## Упражнения

### Упражнение 1 — Считаем записи (легко)

Подключите `count_reads` к `vfs_write` вместо `vfs_read` (и поменяйте заголовок итогов). Запустите
`cargo run --release -- cat`, а во втором терминале — `cat /etc/os-release > /dev/null`. Сколько записей делает `cat`?

<details>
<summary>Подсказка</summary>

Меняется только сторона пользователя: имя, переданное в `KProbe::attach`. Программа eBPF не знает, к какой функции
она подключена.

</details>

<details>
<summary>Ответ</summary>

В `tracer/src/main.rs`:

```rust
    reads.attach("vfs_write", 0)?;
```

```rust
    println!("\nvfs_write calls by command:");
```

```text
vfs_write calls by command:
       1 cat
```

Почему это работает: программу kprobe можно подключить к любой функции ядра; тот же код счётчика теперь считает
записи. `cat` читает маленький файл за один раз и пишет его одним вызовом `write`.

</details>

### Упражнение 2 — Тестируем фильтр (средне)

`wanted` решает, что печатает инструмент, но теста у неё пока нет. Напишите тест `wanted_filters_by_name` в
`tracer-common`, который проверяет три случая: без фильтра подходит любая команда, фильтр принимает то же имя и фильтр
отклоняет другое имя. Запустите `cargo test -p tracer-common`.

<details>
<summary>Подсказка</summary>

`wanted(None, "systemd")` должно быть `true`, `wanted(Some("cat"), "cat")` — `true`, `wanted(Some("cat"), "catalog")` —
`false`.

</details>

<details>
<summary>Ответ</summary>

```rust
    #[test]
    fn wanted_filters_by_name() {
        assert!(wanted(None, "systemd"));
        assert!(wanted(Some("cat"), "cat"));
        assert!(!wanted(Some("cat"), "catalog"));
    }
```

```text
test result: ok. 3 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

Почему это работает: тест выполняется на любой машине, без root и без ядра, потому что общий крейт — обычный Rust.
Случай `catalog` гарантирует, что фильтр сравнивает имена целиком, а не префиксы.

</details>

### Упражнение 3 — Чтение или запись? (сложно)

Добавьте в событие аргумент `flags` вызова `openat` и печатайте `read ` или `write` перед каждым именем файла. Файл
формата показывает `flags` по смещению 32 (8 байтов). Два младших бита флагов — режим доступа: `0` — только чтение,
`1` — только запись, `2` — чтение и запись. Проверьте через `cargo run --release -- touch` и
`touch /tmp/p22-new-file`.

<details>
<summary>Подсказка</summary>

Добавьте `pub flags: u32` в `OpenEvent` (после `pid`, чтобы структура осталась аккуратно выровненной), прочитайте
значение через `ctx.read_at::<u64>(32)` и сохраните его через `as u32`. В пространстве пользователя
`event.flags & 0b11 == 0` означает «только чтение».

</details>

<details>
<summary>Ответ</summary>

`tracer-common/src/lib.rs`:

```rust
pub struct OpenEvent {
    pub pid: u32,
    pub flags: u32,
    pub comm: [u8; 16],
    pub filename: [u8; 64],
}
```

`tracer-ebpf/src/main.rs` (новая константа и две новые строки в `try_openat`):

```rust
/// Offset of `flags` in the same record.
const FLAGS_OFFSET: usize = 32;
```

```rust
    // SAFETY: the format file says that `flags` is stored at this offset.
    let flags: u64 = unsafe { ctx.read_at(FLAGS_OFFSET)? };
```

```rust
        (*event).flags = flags as u32;
```

`tracer/src/main.rs`, внутри `if wanted(...)`:

```rust
                    let mode = if event.flags & 0b11 == 0 {
                        "read "
                    } else {
                        "write"
                    };
                    println!(
                        "{:>7} {:<12} {mode} {}",
                        event.pid,
                        comm,
                        c_str(&event.filename)
                    );
```

```text
Tracing openat and vfs_read. Press Ctrl-C to stop.
  60866 touch        read  /etc/ld.so.cache
  60866 touch        read  /lib/aarch64-linux-gnu/libc.so.6
  60866 touch        write /tmp/p22-new-file
```

Почему это работает: аргументы tracepoint — просто поля записи по фиксированным смещениям, а `flags` — обычное число в
памяти ядра, поэтому достаточно `read_at`. Так как обе стороны используют один `OpenEvent`, добавление поля меняет обе
сразу.

</details>

## Вопросы для самопроверки

1. Почему программа eBPF не может просто держать счётчик в обычной переменной `static mut`, чтобы его читало
   пространство пользователя?

<details>
<summary>Ответ</summary>

Данные, которые должны сохраняться между запусками или читаться пространством пользователя, живут в картах; загрузчик
создаёт их в ядре, а пространство пользователя читает их системным вызовом `bpf`. Карты ещё и дают верификатору
известный размер и тип для проверки.

</details>

2. Что делают `reserve` и `submit` у кольцевого буфера и что происходит, когда буфер полон?

<details>
<summary>Ответ</summary>

`reserve` занимает место для одной записи внутри кольцевого буфера; программа заполняет его на месте; `submit` делает
его видимым пространству пользователя. Когда буфер полон, `reserve` возвращает `None`, и программа отбрасывает событие,
а не ждёт.

</details>

3. Почему имя файла читается через `bpf_probe_read_user_str_bytes`, а не через `*ptr`?

<details>
<summary>Ответ</summary>

Указатель ведёт в память вызывающего процесса, которая может быть неверной или отсутствовать. Вспомогательная функция
копирует безопасно и при плохом адресе возвращает ошибку; прямое разыменование такого значения верификатор запрещает.

</details>

4. Почему хеш-таблица по процессорам — правильный выбор для подсчёта вызовов `vfs_read`?

<details>
<summary>Ответ</summary>

Каждый процессор обновляет только свою копию, поэтому увеличения с разных процессоров никогда не затирают друг друга, и
нет борьбы за одно общее значение. Пространство пользователя складывает значения процессоров.

</details>

5. Зачем общему типу события `#[repr(C)]`?

<details>
<summary>Ответ</summary>

Обе стороны должны размещать каждое поле по одному и тому же смещению. `#[repr(C)]` фиксирует порядок и выравнивание;
размещение Rust по умолчанию может отличаться между сборками и платформами.

</details>

## Итоги

- Карты — единственный способ хранить состояние и общаться с пространством пользователя: хеш-таблицы, массивы,
  варианты по процессорам и кольцевые буферы, объявленные через `#[map]` и открываемые по имени на стороне
  пользователя.
- Тип с `#[repr(C)]` в общем крейте `no_std` разделяют обе стороны; его чистые вспомогательные функции проверяются
  через `cargo test`.
- Аргументы tracepoint читаются по смещениям из файла формата через `read_at`; память пользователя копируется
  вспомогательными функциями `bpf_probe_read_user_*`, иначе верификатор отклоняет программу.
- Кольцевой буфер: `reserve` → заполнить → `submit` в ядре; `next()` в пространстве пользователя.
- Kprobe подключается к функции ядра (`vfs_read`); считайте в ядре картой по процессорам и складывайте в пространстве
  пользователя.

## Источники и что почитать

- [Linux kernel documentation — BPF maps](https://docs.kernel.org/bpf/maps.html) — все типы карт и их правила.
- [Linux kernel documentation — BPF ring buffer](https://docs.kernel.org/bpf/ringbuf.html) — как работают `reserve` и
  `submit`.
- [`aya::maps`](https://docs.rs/aya/0.16.0/aya/maps/index.html) — типы карт на стороне пользователя.
- [`aya_ebpf::maps`](https://docs.rs/aya-ebpf/0.3.0/aya_ebpf/maps/index.html) — типы карт на стороне ядра.
- [bpf-helpers(7)](https://man7.org/linux/man-pages/man7/bpf-helpers.7.html) — страница руководства со всеми
  вспомогательными функциями, включая `bpf_probe_read_user_str`.

---

[🏠 Начало](../START-HERE.md) · [← 22.7 Трек eBPF: первая программа на Aya](07-ebpf-track-a-first-program-with-aya.md) · [📋 Этап 22](README.md) · [22.9 Трек eBPF: проект и следующие шаги →](09-ebpf-track-project-and-next-steps.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
