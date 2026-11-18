<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 17.1 Как работают диски и файлы](01-how-disks-and-files-work.md) · [📋 Этап 17](README.md) · [17.3 Двоичные форматы файлов и сериализация →](03-binary-file-formats.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/17-storage-and-databases/02-durability-and-fsync.md) <!-- nav -->

# Урок 17.2 — Надёжность записи: fsync и устойчивость к сбоям

> **Этап 17 — Как устроены хранилища и базы данных** · Урок 2 из 15 · ⏱ около 2,5 часа (2 занятия)

## Что вы узнаете

- Три вида сбоев и какие данные уничтожает каждый из них.
- Что на самом деле делают `File::sync_all` и `File::sync_data` на Linux и на macOS (где Rust использует `F_FULLFSYNC`).
- Сколько стоит надёжность и как *групповая фиксация* (group commit) делает её доступной.
- Почему неудавшийся `fsync` никогда нельзя просто повторить.
- Что такое *разорванная запись* (torn write) и как смоделировать сбой посреди записи в обычном тесте.

## Перед началом

- Вы прошли [Урок 17.1 — Как работают диски и файлы](01-how-disks-and-files-work.md), и у вас есть пакет
  `code/p17-storage`.
- Вы знаете `fsync` и шаблон атомарной замены (временный файл, `sync_all`, `rename`, синхронизация каталога) из
  [Урока 13.12 — Файловые системы изнутри](../13-operating-systems/12-file-systems-inside.md). Этот урок не повторяет
  шаблон, а объясняет, что лежит под ним.
- Вы знаете сигналы и `abort` из [Урока 13.7 — Сигналы в ваших программах](../13-operating-systems/07-signals-in-your-programs.md)
  и умеете реализовать трейт вроде `Write` для своего типа ([Урок 05.2](../05-rust-intermediate/02-traits.md)).

## Новые слова

| Слово | Значение |
|---|---|
| **надёжно сохранённый** (durable) | Сохранённый так, что переживает сбой или отключение питания. Надёжная запись дошла до хранилища, которое хранит данные без питания. |
| **сбой процесса** (process crash) | Умирает только ваша программа (паника с `abort`, `kill -9`, ошибка). Ядро и его страничный кэш продолжают работать. |
| **сбой ОС** (OS crash, Operating System crash, сбой операционной системы) | Останавливается само ядро или пропадает питание. Всё в RAM (Random Access Memory, оперативная память) теряется, включая страничный кэш. |
| **`fdatasync`** | Системный вызов вроде `fsync`, но он может пропустить метаданные, не нужные для чтения данных (например, время изменения). В Rust: `File::sync_data`. |
| **`F_FULLFSYNC`** | Команда `fcntl` в macOS, которая заставляет сам накопитель записать свой кэш в постоянное хранилище. Обычный `fsync` на macOS этого не делает. |
| **подтвердить** (acknowledge) | Сообщить вызывающему «ваши данные сохранены». Движок хранения не должен подтверждать запись, пока она не стала надёжной. |
| **групповая фиксация** (group commit) | Сделать надёжными много ожидающих записей **одним** `fsync`, чтобы каждая платила только часть его цены. |
| **разорванная запись** (torn write) | Запись, прерванная сбоем, так что до хранилища дошла только её первая часть. |
| **внедрение сбоев** (fault injection) | Намеренное создание отказов в тесте (полный диск, сбой после N байт), чтобы проверить, как код на них реагирует. |

## 1. Три вида сбоев

«Программа упала» может означать три совсем разные вещи. Вопрос всегда один: какие слои из Урока 17.1 теряют своё
содержимое?

| Что происходит | Ваши буферы (`BufWriter`) | Страничный кэш (RAM ядра) | Кэш записи накопителя | Флеш / пластины |
|---|---|---|---|---|
| **Сбой процесса**: паника, `abort`, `kill -9` | теряются | **сохраняется** | сохраняется | сохраняются |
| **Сбой ОС**: паника ядра, ВМ (виртуальная машина) убита | теряются | теряется | сохраняется (обычно) | сохраняются |
| **Отключение питания** | теряются | теряется | теряется, если не сброшен | сохраняются |

Хорошая новость — в первой строке. Когда системный вызов `write` вернулся, данные принадлежат ядру. Если умирает
только ваш процесс, ядро всё равно позже запишет эти страницы на устройство. `fsync` нужен только против отключения
питания и сбоев ОС.

Плохая новость — в последней строке. Накопитель подтверждает запись, когда данные оказались в его собственном быстром
кэше в RAM, а не в микросхемах флеш-памяти. При отключении питания этот кэш тоже может пропасть, если никто не попросил
накопитель его сбросить.

## 2. Сбой процесса своими глазами

Поместите это в `src/bin/crash.rs`:

```rust
use std::fs::File;
use std::io::{BufWriter, Write};

fn main() -> std::io::Result<()> {
    let mode = std::env::args().nth(1).unwrap_or_default();
    let file = File::create("crash.log")?;

    if mode == "buffered" {
        let mut out = BufWriter::new(file);
        out.write_all(b"line 1\nline 2\nline 3\n")?;
        // The three lines are still in the BufWriter's buffer, in this process's memory.
        std::process::abort();
    } else {
        let mut out = file;
        out.write_all(b"line 1\nline 2\nline 3\n")?;
        // The write system call has finished: the lines are in the kernel's page cache.
        std::process::abort();
    }
}
```

**Что происходит:**

1. `std::process::abort()` сразу убивает процесс сигналом `SIGABRT`. Деструкторы не выполняются, поэтому `BufWriter`
   не получает шанса записать свой буфер. Это то же самое, что сбой.
2. В режиме `buffered` 21 байт только копируется в буфер `BufWriter`; системного вызова не было.
3. В другом режиме `write_all` на `File` делает системный вызов `write`, который возвращается до `abort`.

Запустите оба режима. Командная оболочка сообщает о каждом аварийном завершении (zsh печатает строку с `abort`); затем
посмотрите на файл:

```bash
cargo run --release --bin crash -- buffered
wc -c crash.log
cargo run --release --bin crash -- direct
cat crash.log
```

```text
       0 crash.log
line 1
line 2
line 3
```

Буферизованные строки пропали; прямые пережили сбой, хотя никто не вызывал `fsync`. Страничный кэш находится вне
вашего процесса. Отсюда первое правило устойчивости к сбоям: данные, которые есть только в памяти вашего процесса,
теряются при любом сбое, даже самом мягком.

## 3. Что на самом деле обещает `fsync`

Из Урока 13.12: `File::sync_all` ждёт, пока данные и метаданные файла окажутся на устройстве хранения. Вот детали,
которые важны для базы данных.

**На Linux** `sync_all` — это системный вызов `fsync`, а `sync_data` — `fdatasync`. Оба записывают «грязные» страницы этого
файла на устройство, а затем просят устройство сбросить его кэш записи, чтобы данные действительно оказались во
флеш-памяти. Разница: `fdatasync` может пропустить метаданные, которые не нужны для чтения данных, например время
изменения. Размер файла, если файл вырос, он записать всё равно обязан. Для растущего журнала оба стоят примерно
одинаково.

Это можно увидеть через `strace` в вашей ВМ `dev` (или в контейнере из образа `rust:1.100`). Программа
`src/bin/sync_calls.rs`:

```rust
use std::fs::File;
use std::io::Write;

fn main() -> std::io::Result<()> {
    let mut file = File::create("sync.log")?;
    file.write_all(b"hello\n")?;
    file.sync_data()?; // Linux: fdatasync
    file.sync_all()?; // Linux: fsync
    std::fs::remove_file("sync.log")?;
    println!("done");
    Ok(())
}
```

Соберите её в ВМ и выполните `strace -e trace=openat,write,fsync,fdatasync ./target/release/sync_calls`. Важные строки
(загрузка библиотек в начале вырезана):

```text
openat(AT_FDCWD, "sync.log", O_WRONLY|O_CREAT|O_TRUNC|O_CLOEXEC, 0666) = 3
write(3, "hello\n", 6)                  = 6
fdatasync(3)                            = 0
fsync(3)                                = 0
```

**На macOS** системный вызов `fsync` слабее. Справочная страница Apple для `fsync` говорит, что он переносит данные из
компьютера *в накопитель*, но накопитель может держать их в своём кэше и записать позже, даже в другом порядке. Чтобы
попросить накопитель записать всё в постоянное хранилище, в macOS есть отдельная команда: `fcntl(fd, F_FULLFSYNC)`.

Стандартная библиотека Rust выбирает безопасный вариант. В её исходном коде (`library/std/src/sys/fs/unix.rs`) на
системах Apple **и** `sync_all`, **и** `sync_data` вызывают `fcntl(fd, F_FULLFSYNC)`; на Linux они вызывают `fsync` и
`fdatasync`. Документация `File::sync_all` говорит только, что метод «attempts to sync all OS-internal file content and
metadata to disk»; деталь для платформ есть в исходниках. Поэтому на вашем Mac `sync_all` и `sync_data` одинаково
надёжны и одинаково медленны.

> 🔒 **Безопасность:** надёжность записи — часть *целостности*, одной из трёх целей из
> [Урока 10.1](../10-security-fundamentals/01-security-mindset-and-the-cia-triad.md). Банк, который сказал «платёж
> сохранён» и потерял его после отключения света, нарушил обещание так же, как банк, который взломали.

## 4. Цена надёжности

Насколько «медленно» — это медленно? Поместите это в `src/bin/fsync_cost.rs`. Программа дописывает 1000 записей по
100 байт и синхронизирует разными способами:

```rust
use std::fs::{self, File, OpenOptions};
use std::io::Write;
use std::os::fd::AsRawFd;
use std::time::{Duration, Instant};

const RECORDS: usize = 1000;
const RECORD: [u8; 100] = [b'r'; 100];

/// A function that makes a file durable, such as `File::sync_data`.
type SyncFn = fn(&File) -> std::io::Result<()>;

/// Appends RECORDS records to a new file and calls `sync` after every `every` records.
fn run(every: Option<usize>, sync: SyncFn) -> std::io::Result<Duration> {
    let mut file = OpenOptions::new()
        .create(true)
        .truncate(true)
        .write(true)
        .open("cost.log")?;
    let start = Instant::now();
    for i in 1..=RECORDS {
        file.write_all(&RECORD)?;
        if every.is_some_and(|n| i % n == 0) {
            sync(&file)?;
        }
    }
    let elapsed = start.elapsed();
    fs::remove_file("cost.log")?;
    Ok(elapsed)
}

/// The plain fsync() system call, without F_FULLFSYNC.
fn plain_fsync(file: &File) -> std::io::Result<()> {
    // SAFETY: fsync only reads the descriptor number of a file that is open.
    if unsafe { libc::fsync(file.as_raw_fd()) } == 0 {
        Ok(())
    } else {
        Err(std::io::Error::last_os_error())
    }
}

fn main() -> std::io::Result<()> {
    let cases: [(&str, Option<usize>, SyncFn); 4] = [
        ("no sync", None, File::sync_data),
        ("sync_data every record", Some(1), File::sync_data),
        ("sync_data every 100", Some(100), File::sync_data),
        ("plain fsync every record", Some(1), plain_fsync),
    ];
    for (name, every, sync) in cases {
        println!("{name:<26} {:?}", run(every, sync)?);
    }
    Ok(())
}
```

**Что происходит:**

1. `type SyncFn = fn(&File) -> …` даёт короткое имя типу *указателя на функцию*: любой функции, которая принимает `&File`
   и возвращает `io::Result<()>`. Подходят и `File::sync_data`, и наша `plain_fsync`, поэтому `run` принимает любую.
2. `every.is_some_and(|n| i % n == 0)` истинно, когда интервал синхронизации задан и номер записи `i` кратен ему.
3. `plain_fsync` вызывает функцию C `fsync` напрямую через `libc`, чтобы сравнить слабый `fsync` macOS с `F_FULLFSYNC`
   из Rust. При ошибке `last_os_error` превращает номер ошибки C в `io::Error`.
4. Каждый случай использует новый файл, поэтому случаи не мешают друг другу.

`cargo run --release --bin fsync_cost` на Mac:

```text
no sync                    2.482958ms
sync_data every record     5.211412959s
sync_data every 100        46.045ms
plain fsync every record   44.335167ms
```

Прочитайте эту таблицу медленно: она многое объясняет в базах данных.

- Без синхронизации 1000 дописываний занимают 2,5 мс: они идут только в страничный кэш.
- С настоящей синхронизацией после каждой записи — **5,2 секунды**: около 5 мс на один `F_FULLFSYNC`, в две тысячи раз
  медленнее. Программа, которая так делает, может сохранить лишь около 200 записей в секунду.
- Синхронизация раз в 100 записей — это 10 синхронизаций вместо 1000, примерно в 100 раз быстрее.
- Обычный `fsync` на macOS в сто раз быстрее `F_FULLFSYNC`, потому что не ждёт кэша накопителя. Именно этой скоростью
  вы платите за настоящую надёжность.

Движок хранения не может просто синхронизировать реже: каждая подтверждённая запись должна быть надёжной. Приём
называется **групповой фиксацией**. Когда много клиентов пишут одновременно, движок дописывает все их записи, вызывает
`fsync` **один раз** и затем подтверждает всем:

```text
client A ─ write ─┐
client B ─ write ─┼─► append A, B, C ─► one fsync (5 ms) ─► "saved" to A, B and C
client C ─ write ─┘
```

Каждый клиент по-прежнему ждёт полной синхронизации, но движок делает за одну синхронизацию в 100 раз больше работы.
Эту идею используют PostgreSQL, SQLite и движки хранения, которые вы напишете. В упражнении 3 вы построите маленькую
версию политики, которая за этим стоит.

> ⚡ **Производительность:** скорость надёжной записи задаёт синхронизация, а не байты. Измеряйте с синхронизацией,
> иначе ваш бенчмарк измеряет RAM.

## 5. Когда `fsync` завершается ошибкой

`sync_all` возвращает `Result`, и это может быть ошибка: у устройства аппаратная неисправность, закончилось место или
сетевая файловая система потеряла соединение. Что программе делать тогда?

Естественная мысль — попробовать ещё раз. Это неправильно. В 2018 году разработчики PostgreSQL обнаружили (в обсуждении,
которое теперь называют «fsyncgate»), что на Linux после неудачного `fsync` ядро может пометить «грязные» страницы как
чистые или даже выбросить их. Второй `fsync` тогда возвращает успех, хотя данные так и не дошли до устройства. Об ошибке
сообщается только один раз.

С тех пор PostgreSQL останавливается (*паникует*), когда `fsync` завершается ошибкой, а при перезапуске восстанавливает
своё состояние из журнала упреждающей записи, с которым вы познакомитесь в Уроке 17.4. Правило для вашего собственного
кода, в `src/bin/save.rs`:

```rust
use std::fs::File;
use std::io::Write;

fn save(file: &mut File, record: &[u8]) -> std::io::Result<()> {
    file.write_all(record)?;
    file.sync_data()?; // on error: return it; never loop and retry
    Ok(())
}

fn main() {
    let mut file = File::create("save.log").expect("cannot create save.log");
    if let Err(e) = save(&mut file, b"put a=1\n") {
        // We do not know what is on the disk now. Stop, and let recovery decide on restart.
        eprintln!("fatal: could not make the write durable: {e}");
        std::process::exit(1);
    }
    println!("saved");
    std::fs::remove_file("save.log").expect("cannot remove save.log");
}
```

**Что происходит:**

1. `save` возвращает ошибку `sync_data` через `?`, а не прячет её. Функция никогда не заявляет об успехе ложно.
2. `main` считает ошибку фатальной: ничего не подтверждает и завершается. При следующем запуске восстановление прочитает
   файл и выяснит, что там на самом деле.
3. Когда всё работает, программа печатает:

```text
saved
```

Помните также из Урока 17.1, что удаление (drop) `File` или `BufWriter` игнорирует ошибки. Надёжный код всегда
заканчивается явными `flush` и синхронизацией, чей `Result` он проверяет.

## 6. Разорванные записи и сбой, который можно протестировать

Запись 100 байт — не один шаг для оборудования. Если питание пропадёт посередине, устройство может сохранить первую
часть и потерять остальное: это **разорванная запись**. Устройства обещают в лучшем случае, что не разорвётся один сектор
(512 байт или 4 КиБ — кибибайта по 1024 байта), а многие не обещают и этого. И, как сказано на странице Apple, без полного сброса накопитель может
даже записать более поздние данные раньше более ранних.

Поэтому после сбоя в конце журнала может оказаться половина записи. Код, который читает файл при перезапуске, должен её
распознать. Урок 17.3 даст для этого инструменты (префиксы длины и контрольные суммы). Сначала создадим такой файл
намеренно.

Выдернуть шнур питания в тесте нельзя, но эффект можно смоделировать **внедрением сбоев**: писателем, который «падает»
после заданного числа байт. Создайте `src/crash.rs`:

```rust
use std::io::{self, Write};

/// A writer that pretends the machine crashes after `budget` bytes.
/// It passes at most `budget` bytes to `inner`; after that every write fails.
pub struct CrashAfter<W> {
    inner: W,
    budget: usize,
}

impl<W: Write> CrashAfter<W> {
    pub fn new(inner: W, budget: usize) -> Self {
        CrashAfter { inner, budget }
    }

    pub fn into_inner(self) -> W {
        self.inner
    }
}

impl<W: Write> Write for CrashAfter<W> {
    fn write(&mut self, buf: &[u8]) -> io::Result<usize> {
        if self.budget == 0 {
            return Err(io::Error::other("simulated crash"));
        }
        let n = buf.len().min(self.budget);
        let written = self.inner.write(&buf[..n])?;
        self.budget -= written;
        Ok(written)
    }

    fn flush(&mut self) -> io::Result<()> {
        self.inner.flush()
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn crash_in_the_middle_of_a_record_leaves_half_of_it() {
        let mut disk = CrashAfter::new(Vec::new(), 20);
        disk.write_all(b"put a=1\n").unwrap(); // 8 bytes
        disk.write_all(b"put b=2\n").unwrap(); // 16 bytes
        let result = disk.write_all(b"put c=3\n"); // crash after 4 more bytes
        assert!(result.is_err());
        assert_eq!(disk.into_inner(), b"put a=1\nput b=2\nput ");
    }
}
```

и объявите модуль, заменив содержимое `src/lib.rs` (упражнение Урока 17.1 выполнено; отныне `lib.rs` перечисляет
модули этапа):

```rust
pub mod crash;
```

**Что происходит:**

1. `CrashAfter<W>` оборачивает любого писателя: `Vec<u8>` в тестах («диск в памяти») или настоящий `File`.
2. `write` может записать меньше байтов, чем просили; трейт `Write` это разрешает. `write_all` вызывает `write`, пока всё
   не записано или не пришла ошибка. Когда бюджет исчерпан, `write` возвращает ошибку: это и есть «сбой».
3. `io::Error::other` создаёт ошибку вида `Other` с сообщением.
4. Тест пишет две целые записи и начинает третью. После «сбоя» на диске первые 20 байт: две хорошие записи и разорванное
   начало третьей, `put `.

`cargo test --lib crash`:

```text
running 1 test
test crash::tests::crash_in_the_middle_of_a_record_leaves_half_of_it ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

Этот писатель — маленькая модель сбоя: он показывает, как выглядит файл, и код восстановления можно проверить на каждой
возможной точке сбоя (упражнение 2). Он не моделирует накопитель, который переставляет записи; для этого понадобился бы
фейковый диск, который хранит несинхронизированные записи отдельно и теряет часть из них. Настоящие проекты также
запускают тесты с убийством процесса: программа запускается как дочерний процесс, в случайный момент её убивают сигналом
`SIGKILL`, перезапускают и проверяют данные. Дополнительная цель 2 проекта в Уроке 17.14 — именно такой тест.

> 🧪 **Тестирование:** устойчивость к сбоям тестируют, делая сбои дешёвыми. Оберните файл писателем, внедряющим сбои,
> переберите в цикле все точки сбоя и проверьте одно свойство: после восстановления каждая подтверждённая запись на
> месте, а ничего недописанного не видно.

## 7. Чек-лист надёжности

Всё из Уроков 13.12, 17.1 и этого урока одним списком:

1. Данные в вашем процессе (`Vec`, `BufWriter`) теряются при любом сбое. Делайте `flush`, прежде чем на них полагаться.
2. Данные после `write` переживают сбой процесса, но не сбой ОС и не отключение питания.
3. Вызывайте `sync_data` (или `sync_all`) до того, как подтверждать запись. На macOS вызовы синхронизации в Rust
   используют `F_FULLFSYNC`.
4. Новый файл, переименование или удаление меняют **каталог**: синхронизируйте и каталог (Урок 13.12).
5. Если синхронизация не удалась, не повторяйте и не продолжайте. Остановитесь и восстановитесь при перезапуске.
6. Ожидайте разорванную последнюю запись после сбоя; пусть читатель её распознаёт (Уроки 17.3 и 17.4).
7. Группируйте: одна синхронизация на много записей (групповая фиксация).

## Частые ошибки

1. **Мерить без синхронизации, а потом обещать надёжность.** Числа «no sync» — это не числа базы данных.
2. **Вызывать обычный `libc::fsync` на macOS ради надёжности.** Он не сбрасывает кэш накопителя. Используйте
   `File::sync_data` или `File::sync_all`, которые используют `F_FULLFSYNC`.
3. **Повторять неудавшуюся синхронизацию.** Второй вызов может вернуть `Ok` для потерянных данных. Считайте первую
   ошибку фатальной.
4. **Подтверждать до синхронизации.** «Сохранено» должно прийти *после* того, как `sync_data` вернул `Ok`, — никогда не
   раньше и не параллельно.
5. **Думать, что тест со сбоем процесса доказывает устойчивость к отключению питания.** `kill -9` сохраняет страничный
   кэш, поэтому такой тест проходит даже вовсе без синхронизации. Он проверяет ваш код восстановления, а не надёжность.

## Попробуйте сами

1. Запустите `fsync_cost` дважды. Вычислите по своим числам время одного `F_FULLFSYNC` и одного обычного `fsync`.
2. Замените `Some(100)` на `Some(10)` и предскажите новое время до запуска.
3. В `crash.rs` вызовите `out.flush()?` перед `abort()` в буферизованной ветке. Запустите снова: теперь строки
   сохраняются. Почему?

## Упражнения

### Упражнение 1 — Какие данные выживут? (легко)

Программа дописывает запись через `write_all` на `File`, **не** вызывает `sync_data` и печатает «saved». Для каждого
события скажите, сохранится ли запись: (а) программа через мгновение паникует и аварийно завершается; (б) кто-то
выполняет для неё `kill -9`; (в) садится батарея ноутбука; (г) программа вызывает `sync_data`, получает `Ok`, и затем
садится батарея.

<details>
<summary>Подсказка</summary>

Воспользуйтесь таблицей из раздела 1. После `write` запись находится в страничном кэше.

</details>

<details>
<summary>Ответ</summary>

(а) Сохранится: сбой процесса не трогает страничный кэш. (б) Сохранится по той же причине. (в) Скорее всего, потеряется:
страничный кэш — это RAM, и без синхронизации ядро могло ещё не записать её. (г) Сохранится: `sync_data` вернул `Ok`,
значит, данные и размер файла в постоянном хранилище (на macOS — через `F_FULLFSYNC`).

</details>

### Упражнение 2 — Проверьте каждую точку сбоя (средне)

Добавьте в `src/crash.rs` тест, который пишет 24 байта `b"put a=1\nput b=2\nput c=3\n"` через `CrashAfter` с каждым
бюджетом от 0 до 24. Для каждого бюджета проверьте, что на диске ровно `budget` байт и что это начало данных.

<details>
<summary>Подсказка</summary>

`write_all` возвращает ошибку, когда бюджета не хватает; проигнорируйте её через `let _ = …`. У среза есть метод
`starts_with`.

</details>

<details>
<summary>Ответ</summary>

```rust
    #[test]
    fn every_crash_point_leaves_a_prefix() {
        let data = b"put a=1\nput b=2\nput c=3\n";
        for budget in 0..=data.len() {
            let mut disk = CrashAfter::new(Vec::new(), budget);
            let _ = disk.write_all(data); // may fail: that is the crash
            let left = disk.into_inner();
            assert_eq!(left.len(), budget);
            assert!(data.starts_with(&left), "budget {budget}: {left:?}");
        }
    }
```

`cargo test --lib crash` теперь показывает `2 passed`. Почему это работает: файл, в который только дописывают, при сбое
во время записи всегда содержит **префикс** того, что записывалось. На это свойство опирается код восстановления, и тест
проверяет его для всех 25 точек сбоя, а не для одной.

</details>

### Упражнение 3 — Политика синхронизации (сложно)

Создайте `src/synclog.rs` (и `pub mod synclog;` в `src/lib.rs`) с:

- `pub enum SyncPolicy { Always, EveryN(usize), Never }`;
- `pub struct SyncLog`, которая владеет `File`, политикой, счётчиком несинхронизированных записей и публичным счётчиком
  `syncs`;
- `append(&mut self, record: &[u8])`, который пишет запись и синхронизирует, когда это требует политика;
- `sync(&mut self)`, который вызывает `sync_data`, только если есть несинхронизированные записи.

Тест: с `EveryN(10)` 25 дописываний дают 2 синхронизации; один вызов `sync()` делает их 3; ещё один вызов `sync()`
оставляет 3.
(Проверить так же `Always` и `Never` — хорошее дополнительное упражнение.)

<details>
<summary>Подсказка</summary>

В `append` вычислите `due` через `match` по политике: `Always => true`, `EveryN(n) => self.unsynced >= n`,
`Never => false`. Создайте тестовый файл в `std::env::temp_dir()` и удалите его в конце.

</details>

<details>
<summary>Ответ</summary>

```rust
use std::fs::File;
use std::io::{self, Write};

/// When the log calls `sync_data`.
#[derive(Debug, Clone, Copy)]
pub enum SyncPolicy {
    /// After every record: slow, but nothing acknowledged is ever lost.
    Always,
    /// After every `n` records: up to `n - 1` acknowledged records can be lost.
    EveryN(usize),
    /// Never: fast, everything since the last OS flush can be lost.
    Never,
}

pub struct SyncLog {
    file: File,
    policy: SyncPolicy,
    unsynced: usize,
    pub syncs: usize,
}

impl SyncLog {
    pub fn new(file: File, policy: SyncPolicy) -> Self {
        SyncLog {
            file,
            policy,
            unsynced: 0,
            syncs: 0,
        }
    }

    pub fn append(&mut self, record: &[u8]) -> io::Result<()> {
        self.file.write_all(record)?;
        self.unsynced += 1;
        let due = match self.policy {
            SyncPolicy::Always => true,
            SyncPolicy::EveryN(n) => self.unsynced >= n,
            SyncPolicy::Never => false,
        };
        if due {
            self.sync()?;
        }
        Ok(())
    }

    /// Forces everything written so far to the device.
    pub fn sync(&mut self) -> io::Result<()> {
        if self.unsynced > 0 {
            self.file.sync_data()?;
            self.syncs += 1;
            self.unsynced = 0;
        }
        Ok(())
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    fn log_in_temp(name: &str, policy: SyncPolicy) -> (SyncLog, std::path::PathBuf) {
        let path = std::env::temp_dir().join(name);
        let file = File::create(&path).unwrap();
        (SyncLog::new(file, policy), path)
    }

    #[test]
    fn every_n_syncs_once_per_n_records() {
        let (mut log, path) = log_in_temp("p17-every-n.log", SyncPolicy::EveryN(10));
        for _ in 0..25 {
            log.append(b"record\n").unwrap();
        }
        assert_eq!(log.syncs, 2); // after record 10 and record 20
        log.sync().unwrap(); // the last 5 records
        assert_eq!(log.syncs, 3);
        log.sync().unwrap(); // nothing new: no extra sync
        assert_eq!(log.syncs, 3);
        assert_eq!(std::fs::read(&path).unwrap().len(), 25 * 7);
        std::fs::remove_file(path).unwrap();
    }
}
```

`cargo test --lib` (вместе с тестами `crash.rs`):

```text
running 3 tests
test crash::tests::crash_in_the_middle_of_a_record_leaves_half_of_it ... ok
test crash::tests::every_crash_point_leaves_a_prefix ... ok
test synclog::tests::every_n_syncs_once_per_n_records ... ok

test result: ok. 3 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.02s
```

Почему это работает: счётчик `unsynced` помнит, сколько записей ещё не надёжны, а `sync` его сбрасывает. `EveryN` — это
политика, на которой держится групповая фиксация; комментарии к каждому варианту честно говорят, что может потерять сбой.
Настоящий движок подтверждал бы запись только после синхронизации, которая её покрывает.

</details>

## Вопросы для самопроверки

1. Процесс убит через `kill -9` сразу после того, как `write` вернулся. Потеряны ли данные?

<details>
<summary>Ответ</summary>

Нет. После `write` данные в страничном кэше ядра, который переживает сбой процесса. Ядро позже запишет их на устройство.

</details>

2. Что вызывает `File::sync_data` на Linux, а что на macOS?

<details>
<summary>Ответ</summary>

На Linux — `fdatasync`. На macOS (на всех системах Apple) — `fcntl(fd, F_FULLFSYNC)`, то же, что и `sync_all`.

</details>

3. Почему обычного `fsync` на macOS недостаточно для базы данных?

<details>
<summary>Ответ</summary>

Он только переносит данные в накопитель. Накопитель может держать их в своём энергозависимом кэше и записать позже или в
другом порядке, так что отключение питания всё ещё может их уничтожить. `F_FULLFSYNC` просит накопитель записать свой
кэш в постоянное хранилище.

</details>

4. Синхронизация занимает 5 мс. Примерно сколько надёжных записей в секунду может сделать программа, если
   синхронизирует после каждой записи, и сколько — с групповой фиксацией по 100 записей на синхронизацию?

<details>
<summary>Ответ</summary>

Около 200 в секунду с одной синхронизацией на запись и около 20 000 в секунду со 100 записями на синхронизацию (без учёта
цены самих записей).

</details>

5. `sync_data` вернул ошибку. Коллега предлагает вызывать его снова, пока не получится. Что в этом не так?

<details>
<summary>Ответ</summary>

После неудачной синхронизации ядро могло выбросить «грязные» страницы или пометить их чистыми. Второй вызов может вернуть
`Ok`, хотя данные так и не дошли до устройства. Единственная безопасная реакция — остановиться и восстановиться из
журнала при перезапуске.

</details>

6. Что такое разорванная запись и как выглядит конец файла, в который только дописывают, после неё?

<details>
<summary>Ответ</summary>

Запись, прерванная сбоем, так что до хранилища дошла только её первая часть. Тогда файл заканчивается префиксом последней
записи: целые ранние записи и за ними половина записи.

</details>

## Итоги

- Сбой процесса теряет только ваши собственные буферы; сбой ОС теряет ещё и страничный кэш; отключение питания может
  потерять и кэш записи накопителя.
- `sync_all` / `sync_data` — это `fsync` / `fdatasync` на Linux. На macOS Rust вызывает для обоих `fcntl(F_FULLFSYNC)`,
  потому что обычный `fsync` там не сбрасывает накопитель.
- Настоящая синхронизация стоит миллисекунды (здесь около 5 мс), в тысячи раз больше буферизованной записи.
- Групповая фиксация делает надёжными много записей одной синхронизацией, и надёжность становится доступной.
- Никогда не повторяйте неудавшуюся синхронизацию; остановитесь и восстановитесь при перезапуске.
- После сбоя файл, в который только дописывают, содержит префикс записанного, возможно с разорванной записью в конце.
- Писатель с внедрением сбоев вроде `CrashAfter` позволяет обычным тестам перебрать каждую точку сбоя.

## Источники и что почитать

- [std::fs::File](https://doc.rust-lang.org/std/fs/struct.File.html) — `sync_all`, `sync_data` и замечание о том, что
  ошибки при удалении игнорируются.
- [Исходный код стандартной библиотеки Rust: library/std/src/sys/fs/unix.rs](https://github.com/rust-lang/rust/blob/main/library/std/src/sys/fs/unix.rs)
  — найдите `F_FULLFSYNC`, чтобы увидеть, как `fsync` и `datasync` реализованы на системах Apple и на Linux.
- [fsync(2) — справочная страница Linux](https://man7.org/linux/man-pages/man2/fsync.2.html) — `fsync` и `fdatasync`,
  включая сброс кэша диска.
- [fsync(2) — справочная страница macOS (Apple)](https://developer.apple.com/library/archive/documentation/System/Conceptual/ManPages_iPhoneOS/man2/fsync.2.html)
  — почему одного `fsync` недостаточно, чтобы дойти до постоянного хранилища, и что делает `F_FULLFSYNC`.
- [PostgreSQL wiki — Fsync Errors](https://wiki.postgresql.org/wiki/Fsync_Errors) — история «fsyncgate 2018» и то, как
  каждая операционная система обрабатывает ошибки `fsync`.
- [Operating Systems: Three Easy Pieces — Crash Consistency: FSCK and Journaling](https://pages.cs.wisc.edu/~remzi/OSTEP/file-journaling.pdf)
  — как сами файловые системы переживают сбои; основа для WAL (журнала упреждающей записи) из Урока 17.4.

---

[🏠 Начало](../START-HERE.md) · [← 17.1 Как работают диски и файлы](01-how-disks-and-files-work.md) · [📋 Этап 17](README.md) · [17.3 Двоичные форматы файлов и сериализация →](03-binary-file-formats.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
