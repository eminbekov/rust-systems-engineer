<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 22.6 Трек eBPF: что такое eBPF](06-ebpf-track-what-ebpf-is.md) · [📋 Этап 22](README.md) · [22.8 Трек eBPF: карты (maps) и трассировка →](08-ebpf-track-maps-and-tracing.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/22-specializations/07-ebpf-track-a-first-program-with-aya.md) <!-- nav -->

# Урок 22.7 — Трек eBPF: первая программа на Aya

> **Этап 22 — Специализации** · Урок 7 из 30 · ⏱ около 135 минут

## Что вы узнаете

- Как устроен проект Aya: крейт стороны ядра, крейт стороны пользователя и сценарий сборки, который их связывает.
- Как написать на Rust программу eBPF, которая выполняется на tracepoint и пишет сообщение в журнал.
- Как загрузить и подключить её из программы на Tokio и как её сообщения попадают в ваш терминал.
- Как собрать и запустить её в ВМ и как посмотреть на неё снаружи через `bpftool` и `readelf`.

## Перед началом

- Вы прошли [Урок 22.6 — Трек eBPF: что такое eBPF](06-ebpf-track-what-ebpf-is.md): в ВМ `dev` есть nightly Rust с
  `rust-src`, `bpf-linker` и `bpftool`.
- Вы знаете рабочие пространства Cargo ([Урок 08.12](../08-software-design/12-project-layout-and-workspaces.md)),
  сценарии сборки ([Урок 06.10](../06-rust-advanced/10-ffi-calling-c.md)) и Tokio
  ([Урок 14.13](../14-concurrency-and-async/13-tokio-tasks-and-channels.md)).
- `no_std` и обработчики паники объясняются ниже в двух предложениях; подробнее — в направлении ОС, в
  [Уроке 22.2](02-os-track-a-freestanding-binary.md), если хотите.

## Новые слова

| Слово | Значение |
|---|---|
| **сторона ядра / сторона пользователя** (kernel side / user side) | Программа eBPF, которая работает в ядре, и обычная программа, которая её загружает и читает результаты. |
| **`aya-ebpf`** | Крейт для стороны ядра: макросы вроде `#[tracepoint]`, карты, вспомогательные функции. Он `no_std`. |
| **`aya`** | Крейт для стороны пользователя: загрузить объектный файл, подключить программы, читать карты. |
| **`aya-build`** | Крейт для `build.rs`, который собирает сторону ядра через nightly и кладёт объектный файл в `OUT_DIR`. |
| **`aya-log` / `aya-log-ebpf`** | Пара для журналирования: `info!` на стороне ядра пишет записи в карту, сторона пользователя превращает их в обычные строки журнала. |
| **контекст** (context) | Значение, которое ядро передаёт вашей программе, здесь `TracePointContext`: доступ к событию и к текущему процессу. |
| **объектный файл** (object file) | Скомпилированная программа eBPF: ELF-файл с секциями кода, описаниями карт и лицензией. |
| **`OUT_DIR`** | Папка, которую Cargo даёт каждому сценарию сборки для сгенерированных файлов; `env!("OUT_DIR")` находит её во время компиляции. |
| **раннер** (runner) | Программа, которую `cargo run` запускает, передавая ей вашу программу (Урок 22.3); здесь `sudo -E`. |
| **`RUST_LOG`** | Переменная окружения, которая говорит `env_logger`, какие уровни журнала печатать, например `info`. |
| **GPL** (GNU General Public License, универсальная общественная лицензия GNU) | Лицензия на ПО. Многие вспомогательные функции ядро разрешает только программам eBPF, совместимым с GPL. |

**Сокращения из прошлых уроков:** BPF (Berkeley Packet Filter, пакетный фильтр Беркли — имя, из которого вырос eBPF),
ОС (операционная система), ВМ (виртуальная машина; Linux-ВМ `dev`), CPU (Central Processing Unit, центральный
процессор), ELF (Executable and Linkable Format, формат исполняемых и компонуемых файлов), ID (identifier,
идентификатор), UTF-8 (кодировка текста Unicode, Урок 12.5), CI (continuous integration, непрерывная интеграция,
Урок 09.12) и ПО (программное обеспечение).

## 1. Форма проекта Aya

Проект Aya — это **рабочее пространство** Cargo с (как минимум) двумя крейтами:

```text
code/p22-tracer/
├── Cargo.toml              ← рабочее пространство: участники и общие версии
├── .cargo/config.toml      ← `cargo run` использует `sudo -E` как раннер
├── tracer-ebpf/            ← сторона ядра: no_std, сборка для bpfel-unknown-none
│   ├── Cargo.toml
│   ├── build.rs
│   └── src/{lib.rs, main.rs}
└── tracer/                 ← сторона пользователя: обычный Rust, Tokio
    ├── Cargo.toml
    ├── build.rs            ← собирает tracer-ebpf через nightly (aya-build)
    └── src/main.rs         ← встраивает объектный файл, загружает, подключает
```

Хитрость — в `tracer/build.rs`: когда вы собираете сторону пользователя обычным стабильным тулчейном, её сценарий
сборки запускает **вторую** сборку Cargo для `tracer-ebpf` — через nightly, для платформы `bpfel-unknown-none`, с
`bpf-linker`. Получившийся объектный файл попадает в `OUT_DIR`, а `tracer/src/main.rs` включает его байты во время
компиляции. Вы выполняете одну команду и получаете одну программу, которая несёт свою программу eBPF внутри.

Программа этого урока подключается к tracepoint `sched:sched_process_exec`, который срабатывает каждый раз, когда
процесс запускает новую программу (вы трассировали его через bpftrace в Упражнении 2 Урока 22.6). Для каждого события
она пишет в журнал ID процесса и имя программы.

## 2. Рабочее пространство

На Mac создайте папки:

```bash
cd ~/projects/rust
mkdir -p code/p22-tracer/.cargo code/p22-tracer/tracer/src code/p22-tracer/tracer-ebpf/src
cd code/p22-tracer
```

`Cargo.toml` (рабочее пространство):

```toml
[workspace]
resolver = "3"
members = ["tracer", "tracer-ebpf"]
default-members = ["tracer"]

[workspace.package]
edition = "2024"

[workspace.dependencies]
aya = "0.16"
aya-build = "0.2"
aya-ebpf = "0.3"
aya-log = "0.4"
aya-log-ebpf = "0.3"
anyhow = "1"
cargo_metadata = "0.23"
env_logger = "0.11"
log = "0.4"
tokio = { version = "1", features = ["macros", "rt-multi-thread", "net", "signal"] }
which = "8"

[profile.release.package.tracer-ebpf]
debug = 2
codegen-units = 1
```

**Что происходит:**

1. `default-members = ["tracer"]`: простой `cargo build` собирает только сторону пользователя. Сторона ядра никогда
   не собирается для вашей обычной платформы; для eBPF её собирает сценарий сборки.
2. `[workspace.dependencies]` перечисляет каждую версию один раз; крейты-участники пишут `aya.workspace = true`.
3. Настройки профиля для `tracer-ebpf` сохраняют отладочную информацию (загрузчик использует её, чтобы понимать типы)
   и компилируют крейт одной единицей, с чем `bpf-linker` справляется лучше всего.

`.cargo/config.toml`:

```toml
[target."cfg(all())"]
runner = "sudo -E"
```

`cfg(all())` — условие, которое всегда истинно, поэтому это относится к любой платформе: `cargo run` запускает
`sudo -E target/release/tracer`. `-E` сохраняет ваши переменные окружения (например, `RUST_LOG`) под `sudo`.

## 3. Сторона ядра

`tracer-ebpf/Cargo.toml`:

```toml
[package]
name = "tracer-ebpf"
version = "0.1.0"
edition.workspace = true

[dependencies]
aya-ebpf.workspace = true
aya-log-ebpf.workspace = true

[build-dependencies]
which.workspace = true

[[bin]]
name = "tracer"
path = "src/main.rs"
```

`tracer-ebpf/build.rs` заставляет Cargo пересобирать код eBPF, когда меняется `bpf-linker`:

```rust
fn main() {
    // Rebuild this crate when the bpf-linker program changes.
    let linker = which::which("bpf-linker").expect("bpf-linker is not installed");
    println!("cargo:rerun-if-changed={}", linker.display());
}
```

`tracer-ebpf/src/lib.rs` состоит из одной строки, чтобы у крейта была и (пустая) библиотечная часть:

```rust
#![no_std]
```

А сама программа — `tracer-ebpf/src/main.rs`:

```rust
#![no_std]
#![no_main]

use aya_ebpf::{EbpfContext, macros::tracepoint, programs::TracePointContext};
use aya_log_ebpf::info;

#[tracepoint]
pub fn tracer(ctx: TracePointContext) -> u32 {
    match try_tracer(&ctx) {
        Ok(()) => 0,
        Err(_) => 1,
    }
}

fn try_tracer(ctx: &TracePointContext) -> Result<(), i32> {
    let comm = ctx.command()?;
    let len = comm.iter().position(|&b| b == 0).unwrap_or(comm.len());
    // SAFETY: the kernel writes the command name as ASCII text.
    let name = unsafe { core::str::from_utf8_unchecked(&comm[..len]) };
    info!(ctx, "exec: pid {} runs {}", ctx.pid(), name);
    Ok(())
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

1. `#![no_std]` и `#![no_main]`: внутри ядра нет библиотеки операционной системы и нет `main`. Программа — это только
   функции, помеченные как программы eBPF.
2. `#[tracepoint]` превращает `tracer` в программу eBPF типа «tracepoint» и кладёт её в секцию `tracepoint`
   объектного файла. Ядро вызывает её с `TracePointContext` и ждёт обратно `u32` (для tracepoint значение
   игнорируется; по соглашению возвращают 0).
3. Схема `tracer` → `try_tracer` позволяет использовать `?` внутри и превращать любую ошибку в возвращаемое значение в
   одном месте.
4. `ctx.command()` вызывает вспомогательную функцию, которая копирует имя команды текущего процесса (`comm`, до 16
   байтов, дополненных нулевыми байтами). Мы обрезаем по первому нулевому байту. `from_utf8_unchecked` безопасен,
   потому что имена команд — простой текст; проверка UTF-8 внутри eBPF стоила бы многих инструкций.
5. `info!` из `aya-log-ebpf` **не** форматирует текст в ядре. Он пишет маленькую двоичную запись (шаблон сообщения и
   два значения) в карту с именем `AYA_LOGS`; сторона пользователя форматирует её позже.
6. Обработчик паники требуется для `no_std` (Урок 22.2). Программа eBPF на самом деле никогда не может запаниковать:
   если бы компилятор оставил достижимую панику, верификатор отклонил бы цикл в ней. Пишите код без `unwrap` и без
   индексации, которая может не удаться.
7. `LICENSE` идёт в секцию с именем `license`. Ядро её читает; вспомогательные функции с пометкой «только GPL» не
   выдаются программам с несовместимой лицензией.

## 4. Сторона пользователя

`tracer/Cargo.toml`:

```toml
[package]
name = "tracer"
version = "0.1.0"
edition.workspace = true

[dependencies]
anyhow.workspace = true
aya.workspace = true
aya-log.workspace = true
env_logger.workspace = true
log.workspace = true
tokio.workspace = true

[build-dependencies]
anyhow.workspace = true
aya-build.workspace = true
cargo_metadata.workspace = true
tracer-ebpf = { path = "../tracer-ebpf" }

[lints.cargo]
unused_dependencies = "allow"
```

Зависимость сборки от `tracer-ebpf` говорит Cargo, что сторона пользователя зависит от стороны ядра, поэтому изменение
там пересобирает всё. Код на Rust напрямую её не использует, поэтому проверка `unused_dependencies` отключена.

`tracer/build.rs`:

```rust
use anyhow::{Context as _, anyhow};

fn main() -> anyhow::Result<()> {
    let cargo_metadata::Metadata { packages, .. } = cargo_metadata::MetadataCommand::new()
        .no_deps()
        .exec()
        .context("MetadataCommand::exec")?;
    let ebpf_package = packages
        .into_iter()
        .find(|package| package.name.as_str() == "tracer-ebpf")
        .ok_or_else(|| anyhow!("tracer-ebpf package not found"))?;
    let root_dir = ebpf_package
        .manifest_path
        .parent()
        .ok_or_else(|| anyhow!("no parent for {}", ebpf_package.manifest_path))?;
    let package = aya_build::Package {
        name: ebpf_package.name.as_str(),
        root_dir: root_dir.as_str(),
        ..Default::default()
    };
    aya_build::build_ebpf([package], aya_build::Toolchain::default())
}
```

**Что происходит:**

1. `cargo_metadata` спрашивает у Cargo список пакетов рабочего пространства, и сценарий находит `tracer-ebpf` и его
   папку.
2. `aya_build::build_ebpf` выполняет `cargo +nightly build --target bpfel-unknown-none -Z build-std=core --release`
   для этого пакета и копирует объектный файл в `OUT_DIR`. `Toolchain::default()` означает «тулчейн с именем
   `nightly`».

`tracer/src/main.rs`:

```rust
use std::time::Duration;

use aya::programs::TracePoint;
use log::warn;
use tokio::signal;

#[tokio::main]
async fn main() -> anyhow::Result<()> {
    env_logger::init();

    // The eBPF object file was built by build.rs and is embedded here.
    let mut ebpf = aya::Ebpf::load(aya::include_bytes_aligned!(concat!(
        env!("OUT_DIR"),
        "/tracer"
    )))?;

    // Move log records from the kernel to env_logger, ten times per second.
    match aya_log::EbpfLogger::init(&mut ebpf) {
        Ok(mut logger) => {
            std::thread::spawn(move || {
                loop {
                    logger.flush();
                    std::thread::sleep(Duration::from_millis(100));
                }
            });
        }
        Err(e) => warn!("failed to initialize eBPF logger: {e}"),
    }

    let program: &mut TracePoint = ebpf.program_mut("tracer").unwrap().try_into()?;
    program.load()?;
    program.attach("sched", "sched_process_exec")?;

    println!("Tracing exec calls. Press Ctrl-C to stop.");
    signal::ctrl_c().await?;
    println!("Exiting...");
    Ok(())
}
```

**Что происходит:**

1. `include_bytes_aligned!` помещает объектный файл в программу во время компиляции, выровненным в памяти так, как
   нужно разборщику ELF. `Ebpf::load` разбирает его и создаёт его карты в ядре (здесь карту `AYA_LOGS` из
   `aya-log`).
2. `EbpfLogger::init` подключается к этой карте. Журнал владеет картой; мы перемещаем его в поток, который каждые
   100 мс опустошает её и передаёт каждую запись в `env_logger`. (Документация `aya-log` показывает вариант, который
   ждёт данных через Tokio вместо таймера; простого потока здесь достаточно.)
3. `program_mut("tracer")` находит программу по имени её функции; `try_into()` проверяет, что это действительно
   программа tracepoint.
4. `load()` отправляет байт-код в ядро: именно здесь работает **верификатор**. `attach("sched",
   "sched_process_exec")` подключает её к tracepoint (категория и имя, как в tracefs).
5. Программа остаётся подключённой, пока существует `ebpf`. Когда `main` возвращается (после Ctrl-C), `ebpf`
   уничтожается, файловые дескрипторы закрываются, и ядро удаляет программу. Ничего не остаётся.

## 5. Сборка и запуск в ВМ

Скопируйте рабочее пространство в ВМ и соберите там (замените `student` на имя пользователя вашего Mac, как на
Этапе 13):

```bash
limactl shell dev
rsync -a --delete --exclude target /Users/student/projects/rust/code/p22-tracer/ ~/p22-tracer/
cd ~/p22-tracer
cargo build --release
```

Первая сборка скачивает крейты и собирает `core` для eBPF; это занимает минуту. Конец вывода (из проверочного
контейнера, где рабочее пространство лежало в `/work`; у вас будет ваша папка в ВМ):

```text
warning: tracer@0.1.0:    Compiling tracer-ebpf v0.1.0 (/work/tracer-ebpf)
warning: tracer@0.1.0:    Compiling aya-log-ebpf v0.3.0
warning: tracer@0.1.0:     Finished `release` profile [optimized] target(s) in 12.72s
    Finished `release` profile [optimized] target(s) in 36.95s
```

Строки `warning: tracer@0.1.0:` — не проблемы: это вывод внутренней сборки nightly, который сценарий сборки может
показать только как предупреждения. Теперь запустите с уровнем журнала `info`:

```bash
RUST_LOG=info cargo run --release
```

Откройте второй терминал, снова войдите в ВМ (`limactl shell dev`) и выполните там `ls` и `date`. Первый терминал
показывает (среди строк о других программах, которые запускаются на машине):

```text
     Running `sudo -E target/release/tracer`
Tracing exec calls. Press Ctrl-C to stop.
[2026-11-18T08:57:04Z INFO  tracer] exec: pid 6573 runs ls
[2026-11-18T08:57:04Z INFO  tracer] exec: pid 6574 runs date
```

Нажмите Ctrl-C, чтобы остановить; программа печатает `Exiting...`. Ваш первый код на Rust выполнился внутри ядра
Linux.

> 🧪 **Тестирование:** запустите трассировщик, затем выполните известную команду во втором терминале и проверьте,
> что её строка появилась. Эта ручная сквозная проверка — первый тест любой программы eBPF. Упражнение 2 добавляет
> автоматический тест, который проверяет объектный файл без root.

## 6. Смотрим снаружи

Пока трассировщик работает, используйте `bpftool` во втором терминале:

```bash
sudo bpftool prog list name tracer
sudo bpftool map list name AYA_LOGS
```

```text
5664: tracepoint  name tracer  tag 0526e951d91946cd  gpl
	loaded_at 2026-11-18T08:37:24+0000  uid 0
	xlated 4064B  jited 2568B  memlock 8192B  map_ids 626,625
	btf_id 258
625: ringbuf  name AYA_LOGS  flags 0x0
	key 0B  value 0B  max_entries 131072  memlock 144200B
```

У вашей программы тип `tracepoint`, лицензия совместима с GPL, проверенный верификатором байт-код (`xlated`) занимает
4064 байта, а машинный код, созданный JIT (`jited`), — 2568 байтов. Карта журнала — **кольцевой буфер** на 128 КиБ.
После остановки трассировщика обе команды ничего не печатают: ядро их удалило.

Сам объектный файл — ELF-файл для машины «Linux BPF»:

```bash
f=$(find target/release/build -path '*out/tracer' -type f | head -1)
file "$f"
readelf -S -W "$f" | grep -E 'tracepoint|license|maps'
```

```text
target/release/build/tracer/dd119d04b2095d25/out/tracer: ELF 64-bit LSB relocatable, eBPF, version 1 (SYSV), with debug_info, not stripped
  [ 3] tracepoint        PROGBITS        0000000000000000 0002a0 000f48 00  AX  0   0  8
  [ 5] license           PROGBITS        0000000000000000 0011e8 00000d 00   A  0   0  1
  [ 7] maps              PROGBITS        0000000000000000 001224 00001c 00  WA  0   0  4
```

Имена секций — те, что выбрали макросы: программа в `tracepoint`, ваш `LICENSE` в `license`.

## Частые ошибки

> ⚠️ **Частая ошибка:** выбросить журнал. Если написать `let _ = aya_log::EbpfLogger::init(&mut ebpf);`, журнал и его
> карта сразу закрываются, и загрузка программы падает с `fd 10 is not pointing to valid bpf_map`. Держите журнал
> живым (здесь — внутри потока).

> ⚠️ **Частая ошибка:** запускать без `RUST_LOG=info`. `env_logger` по умолчанию печатает только ошибки, поэтому
> программа работает, но не показывает строк `exec:`.

> ⚠️ **Частая ошибка:** запускать программу напрямую обычным пользователем (`target/release/tracer`). Загрузка падает
> с `the BPF_PROG_LOAD syscall returned Operation not permitted (os error 1)`. Используйте `cargo run --release`
> (раннер добавляет `sudo -E`) или `sudo -E target/release/tracer`.

> ⚠️ **Частая ошибка:** собирать на Mac. Сторона пользователя использует системные вызовы Linux и не собирается для
> macOS. Редактируйте на Mac, собирайте и запускайте в ВМ.

## Попробуйте сами

1. Создайте рабочее пространство точно как в разделах 2–4, скопируйте его в ВМ и запустите.
2. Пока оно работает, запустите во втором терминале несколько программ (`ls`, `date`, `uname -a`) и найдите их строки.
3. Посмотрите на программу и карту через `bpftool`, пока оно работает, и снова после Ctrl-C.
4. Сделайте коммит на Mac: `git add code/p22-tracer && git commit -m "p22-tracer: first eBPF program"`.

## Упражнения

### Упражнение 1 — Какой процессор? (легко)

Добавьте вторую строку журнала на уровне `debug`, которая показывает номер процессора, на котором выполняется
программа. Используйте вспомогательную функцию `bpf_get_smp_processor_id()` из `aya_ebpf::helpers` и запустите с
`RUST_LOG=debug`.

<details>
<summary>Подсказка</summary>

Вспомогательные функции — `unsafe`. Импортируйте `debug` рядом с `info` из `aya_log_ebpf`.

</details>

<details>
<summary>Ответ</summary>

Измените импорты и добавьте три строки после строки `info!` в `tracer-ebpf/src/main.rs`:

```rust
use aya_ebpf::{
    EbpfContext, helpers::bpf_get_smp_processor_id, macros::tracepoint,
    programs::TracePointContext,
};
use aya_log_ebpf::{debug, info};
```

```rust
    // SAFETY: this helper only returns the number of the current CPU.
    let cpu = unsafe { bpf_get_smp_processor_id() };
    debug!(ctx, "  (on CPU {})", cpu);
```

```text
[2026-11-18T08:59:10Z INFO  tracer] exec: pid 21198 runs ls
[2026-11-18T08:59:10Z DEBUG tracer]   (on CPU 8)
[2026-11-18T08:59:10Z INFO  tracer] exec: pid 21199 runs date
[2026-11-18T08:59:10Z DEBUG tracer]   (on CPU 7)
```

Почему это работает: программа eBPF выполняется на том процессоре, где произошло событие, поэтому две команды могут
трассироваться на двух разных процессорах одновременно. Урок 22.8 использует этот факт для карт по процессорам
(per-CPU).

</details>

### Упражнение 2 — Тестируем объектный файл без root (средне)

Напишите модульный тест в `tracer/src/main.rs`, который разбирает встроенный объектный файл крейтом `aya-obj` (добавьте
`aya-obj = "0.6"` в `[dev-dependencies]` в `tracer/Cargo.toml`) и проверяет, что в нём есть программа с именем
`tracer` и лицензия `Dual MIT/GPL`. Запустите его через `cargo test --release` — `sudo` не нужен.

<details>
<summary>Подсказка</summary>

`aya_obj::Object::parse(bytes)` читает объектный файл в пространстве пользователя, не обращаясь к ядру. У результата
есть хеш-таблица `programs` и поле `license` типа `CString`.

</details>

<details>
<summary>Ответ</summary>

```rust
#[cfg(test)]
mod tests {
    #[test]
    fn object_file_has_the_program_and_a_license() {
        let bytes = aya::include_bytes_aligned!(concat!(env!("OUT_DIR"), "/tracer"));
        let object = aya_obj::Object::parse(bytes).expect("not a valid eBPF object");
        assert!(object.programs.contains_key("tracer"));
        assert_eq!(object.license.to_str(), Ok("Dual MIT/GPL"));
    }
}
```

```text
test tests::object_file_has_the_program_and_a_license ... ok
test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

Почему это работает: сам `aya` использует `aya-obj`, чтобы читать объектные файлы перед загрузкой. Тест за секунды
ловит сломанную сборку (переименованную функцию, пропавшую лицензию) и работает в CI (Урок 09.12) без всяких
привилегий.

</details>

### Упражнение 3 — Фильтруем в ядре (сложно)

Измените программу eBPF так, чтобы она писала в журнал **только** процессы, имя команды которых — `ls`. Сравнение
должно происходить в ядре, чтобы для других программ запись не создавалась вовсе.

<details>
<summary>Подсказка</summary>

`comm` — массив `[u8; 16]`. Сравните часть фиксированного размера, включая завершающий нулевой байт, со строкой
байтов: `comm[..3] == *b"ls\0"`. Сравнение фиксированного размера легко даётся верификатору.

</details>

<details>
<summary>Ответ</summary>

В `try_tracer`, сразу после `let comm = ctx.command()?;`:

```rust
    if comm[..3] != *b"ls\0" {
        return Ok(()); // not `ls`: do nothing
    }
```

Если во втором терминале запустить `ls` и `date`, весь вывод теперь такой:

```text
Tracing exec calls. Press Ctrl-C to stop.
[2026-11-18T09:00:45Z INFO  tracer] exec: pid 29580 runs ls
Exiting...
```

Почему это работает: нулевой байт по индексу 2 гарантирует, что `lsblk` (который начинается с `ls`) не совпадёт.
Фильтровать в ядре дешевле, чем в пространстве пользователя: отклонённые события вообще не попадают в пространство
пользователя.

</details>

## Вопросы для самопроверки

1. Зачем крейту стороны пользователя `build.rs`?

<details>
<summary>Ответ</summary>

Он собирает крейт стороны ядра через nightly для `bpfel-unknown-none` (с помощью `aya-build`) и кладёт объектный
файл в `OUT_DIR`, чтобы `include_bytes_aligned!` мог встроить его в программу стороны пользователя.

</details>

2. Где в этой программе работает верификатор?

<details>
<summary>Ответ</summary>

В `program.load()`, когда байт-код отправляется в ядро системным вызовом `bpf`. Если программа отклонена, `load()`
возвращает ошибку с журналом верификатора.

</details>

3. Форматирует ли `info!` в программе eBPF текст внутри ядра?

<details>
<summary>Ответ</summary>

Нет. Он пишет компактную двоичную запись в карту-кольцевой буфер `AYA_LOGS`; `aya-log` в пространстве пользователя
читает запись и форматирует сообщение.

</details>

4. Что происходит с программой eBPF, когда программа стороны пользователя завершается?

<details>
<summary>Ответ</summary>

Её файловые дескрипторы закрываются, поэтому ядро отключает и удаляет программу и её карты. `bpftool prog list`
больше её не показывает.

</details>

5. Почему ядру важна секция `license`?

<details>
<summary>Ответ</summary>

Многие вспомогательные функции доступны только программам, совместимым с GPL; ядро проверяет строку лицензии при
загрузке программы.

</details>

## Итоги

- У проекта Aya есть крейт стороны ядра с `no_std`, обычный крейт стороны пользователя и `build.rs`, который собирает
  сторону ядра через nightly и встраивает объектный файл.
- `#[tracepoint]` помечает программу; контекст даёт текущий процесс (`pid()`, `command()`); `info!` пишет записи
  журнала в карту.
- Сторона пользователя: `Ebpf::load` → держать живым `EbpfLogger` → `program_mut(..).try_into()` → `load()`
  (верификатор) → `attach(category, name)`.
- `cargo run --release` с раннером `sudo -E` и `RUST_LOG=info` запускает всё в ВМ.
- `bpftool prog list`, `bpftool map list` и `readelf -S` показывают программу, её карту и её объектный файл.

## Источники и что почитать

- [The Aya book](https://aya-rs.dev/book/) — официальное руководство; его первые главы строят похожую программу.
- [Документация `aya`](https://docs.rs/aya/0.16.0/aya/) — загрузка, программы и карты на стороне пользователя.
- [Документация `aya-ebpf`](https://docs.rs/aya-ebpf/0.3.0/aya_ebpf/) — макросы, контексты и вспомогательные функции
  на стороне ядра.
- [Документация `aya-log`](https://docs.rs/aya-log/0.4.0/aya_log/) — как записи журнала доходят до пространства
  пользователя.
- [aya-template](https://github.com/aya-rs/aya-template) — официальный шаблон проекта (для `cargo generate`), которому
  следует эта структура.
- [Linux kernel documentation — Event Tracing](https://docs.kernel.org/trace/events.html) — события tracepoint и их
  форматы.

---

[🏠 Начало](../START-HERE.md) · [← 22.6 Трек eBPF: что такое eBPF](06-ebpf-track-what-ebpf-is.md) · [📋 Этап 22](README.md) · [22.8 Трек eBPF: карты (maps) и трассировка →](08-ebpf-track-maps-and-tracing.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
