<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 13.10 Виртуальная память](10-virtual-memory.md) · [📋 Этап 13](README.md) · [13.12 Файловые системы изнутри →](12-file-systems-inside.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/13-operating-systems/11-memory-mapped-files.md) <!-- nav -->

# Урок 13.11 — Файлы, отображённые в память

> **Этап 13 — Операционные системы изнутри** · Урок 11 из 18 · ⏱ около 135 минут

## Что вы узнаете

- Что значит *отобразить* файл в память и чем это отличается от `read` и `write`.
- Как страничный кэш связывает файлы, `read` и отображения.
- Как читать и менять файл через отображение с помощью крейта `memmap2` и почему создание отображения — это `unsafe`.
- Чем различаются общие, приватные и анонимные отображения.
- Когда `mmap` быстрее `read`, когда нет, и что означает SIGBUS.

## Перед началом

- Вы прошли [Урок 13.10 — Виртуальная память](10-virtual-memory.md): страницы, страничные сбои, отображения и `/proc/self/maps`.
- Вы знаете файловые дескрипторы ([Урок 13.4](04-files-and-file-descriptors.md)), `strace` ([Урок 13.3](03-system-calls.md)) и срезы ([Урок 03.6](../03-ownership-and-borrowing/06-slices-and-string-types.md)).
- Создайте пакет для этого урока на Mac и скопируйте его в ВМ `dev` точно так же, как в Уроке 13.10 (меняется только имя):

```bash
cd ~/projects/rust
cargo new code/p13-mmap
cd code/p13-mmap
cargo add memmap2@0.9
rm src/main.rs && mkdir src/bin
```

## Новые слова

| Слово | Значение |
|---|---|
| **файл, отображённый в память** (memory-mapped file) | Файл, который ядро делает видимым как диапазон памяти в процессе. Чтение этой памяти читает файл; запись в неё может менять файл. |
| **`mmap` / `munmap`** | Системные вызовы, которые создают и удаляют отображение. `mmap` возвращает начальный адрес нового диапазона. |
| **страничный кэш** (page cache) | Оперативная память, в которой ядро держит недавно использованные данные файлов. И `read`, и отображения берут данные файла оттуда. |
| **общее отображение** (shared mapping, `MAP_SHARED`) | Изменения через отображение попадают в страничный кэш, а потом в файл; их видит каждый процесс, который отобразил этот файл. |
| **приватное отображение** (private mapping, `MAP_PRIVATE`) | Процесс получает вид файла с копированием при записи. Его изменения остаются в его собственной памяти и никогда не попадают в файл. |
| **анонимное отображение** (anonymous mapping) | Отображение без файла: новая память, заполненная нулями. Аллокаторы используют его для больших блоков. |
| **`msync` / сброс** (flush) | Просьба к ядру записать изменённые страницы отображения на диск сейчас, а не позже. |
| **SIGBUS** (bus error, ошибка шины) | Сигнал, который получает процесс, когда трогает отображённую страницу, за которой больше нет данных, например потому что файл укоротили. |
| **без копирования** (zero-copy) | Использование данных там, где они уже лежат, без предварительного копирования в другой буфер. |

## 1. Два способа получить данные файла

Обычный способ вы знаете из [Урока 13.4](04-files-and-file-descriptors.md): открыть файл через `open`, затем
вызвать `read` со своим буфером. Ядро **копирует** байты в этот буфер.

Есть второй способ. Вы просите ядро: «сделай этот файл частью моего адресного пространства». Ядро создаёт
**отображение** (Урок 13.10), страницы которого — это страницы файла. После этого файл *и есть* срез памяти.
Вы читаете его через `data[i]`, без новых системных вызовов.

Аналогия: `read` — это попросить библиотекаря сделать вам ксерокопии страниц книги. `mmap` — получить саму
книгу на стол и открывать её на любой странице.

Оба способа идут через **страничный кэш**: ядро держит недавно использованные данные файлов в RAM, чтобы при
втором обращении не трогать диск.

```text
            disk
              |
              v
   +---------------------+      (kernel memory)
   |     page cache      |  <- pages of the file, 4 KiB each
   +---------------------+
       |              |
  read() copies    mmap() maps the same frames
       |              |
       v              v
  +---------+   +-------------------+   (your process)
  | buffer  |   | mapping: &[u8]    |
  +---------+   +-------------------+
```

С `read` данные лежат в RAM дважды: в страничном кэше и в вашем буфере. С `mmap` ваша таблица страниц
указывает прямо на кадры страничного кэша. Это называется **без копирования** (zero-copy).

Как данные туда попадают? Через **страничные сбои**. Новое отображение начинается с пустой таблицы страниц,
как вектор на 256 МиБ из Урока 13.10. Первое обращение к каждой странице вызывает сбой; ядро находит страницу
в страничном кэше (малый сбой) или сначала читает её с диска (большой сбой) и отображает её.

## 2. Чтение файла через отображение

В стандартной библиотеке Rust нет `mmap`. Крейт `memmap2` оборачивает системный вызов в небольшой
аккуратный API. Создайте `src/bin/mcount.rs`:

```rust
use std::env;
use std::fs::File;

use memmap2::Mmap;

fn main() -> std::io::Result<()> {
    let path = env::args().nth(1).expect("usage: mcount <file>");
    let file = File::open(&path)?;

    // SAFETY: we assume no other process changes or shortens the file while it is mapped.
    let map = unsafe { Mmap::map(&file)? };

    let bytes: &[u8] = &map; // the whole file, as a byte slice
    let lines = bytes.iter().filter(|&&b| b == b'\n').count();
    println!("{path}: {} bytes, {lines} lines", bytes.len());
    Ok(())
}
```

**Что происходит:**

1. `File::open` открывает файл на чтение и, как обычно, даёт нам файловый дескриптор.
2. `Mmap::map(&file)` вызывает `mmap` и отображает весь файл только для чтения. Это `unsafe`-функция;
   почему — объясняет раздел 6. Комментарий `// SAFETY:` говорит, что мы обещаем ([Урок 06.6](../06-rust-advanced/06-unsafe-rust-and-miri.md)).
3. `Mmap` реализует `Deref<Target = [u8]>`, поэтому `&map` — обычный `&[u8]`. Работает любой метод среза.
4. Когда `map` выходит из области видимости, его `Drop` вызывает `munmap`. Отображение остаётся рабочим даже
   после закрытия `file`: оно держит собственную ссылку на файл.

В ВМ:

```bash
printf 'first line\nsecond line\nthird line\n' > small.txt
cargo run --bin mcount -- small.txt
```

```text
small.txt: 34 bytes, 3 lines
```

`strace` показывает системные вызовы за этим (запустите собранную программу напрямую):

```bash
strace -e trace=openat,mmap,munmap,close ./target/debug/mcount small.txt
```

Последние строки:

```text
openat(AT_FDCWD, "small.txt", O_RDONLY|O_CLOEXEC) = 3
mmap(NULL, 34, PROT_READ, MAP_SHARED, 3, 0) = 0xffffb68bc000
small.txt: 34 bytes, 3 lines
munmap(0xffffb68bc000, 34)              = 0
close(3)                                = 0
+++ exited with 0 +++
```

Прочитайте аргументы `mmap` слева направо: `NULL` = «выбери адрес сам», `34` = длина, `PROT_READ` =
страницы только для чтения, `MAP_SHARED` = общее отображение файла, `3` = файловый дескриптор, `0` = начать
со смещения 0 в файле. Вызова `read` нет вообще: подсчёт строк только обращался к памяти.

## 3. Изменение файла через отображение

Записываемое общее отображение меняет сам файл. Создайте `src/bin/mwrite.rs`:

```rust
use std::fs::OpenOptions;

use memmap2::MmapMut;

fn main() -> std::io::Result<()> {
    let file = OpenOptions::new()
        .read(true)
        .write(true)
        .create(true)
        .truncate(true)
        .open("counter.bin")?;
    file.set_len(16)?; // the file must be big enough before we map it

    // SAFETY: only this program uses counter.bin, and we never shorten it while mapped.
    let mut map = unsafe { MmapMut::map_mut(&file)? };

    map[..5].copy_from_slice(b"HELLO");
    map[15] = b'\n';
    map.flush()?; // ask the kernel to write the changed pages to the disk now

    println!("mapped {} bytes, wrote through memory", map.len());
    Ok(())
}
```

**Что происходит:**

1. Файл открывается на чтение **и** запись: записываемому отображению нужно и то и другое.
2. `set_len(16)` делает файл длиной 16 байт (вызывает `ftruncate`). Отображение не может увеличить файл:
   писать за конец файла нельзя. Размер нужно задать заранее.
3. `MmapMut::map_mut` создаёт отображение `PROT_READ|PROT_WRITE`, `MAP_SHARED`. `MmapMut` разыменовывается
   в `&mut [u8]`.
4. Запись в срез меняет страницы в страничном кэше. Любой другой процесс, читающий файл, теперь видит
   `HELLO` — ещё до того, как что-то попало на диск.
5. `flush()` вызывает `msync`, который ждёт, пока изменённые страницы запишутся на диск. Без него ядро
   запишет их позже само (обычно примерно в течение 30 секунд) или при удалении отображения.

```bash
cargo run --bin mwrite
od -c counter.bin
```

```text
mapped 16 bytes, wrote through memory
0000000   H   E   L   L   O  \0  \0  \0  \0  \0  \0  \0  \0  \0  \0  \n
0000020
```

`od -c` (octal dump, восьмеричный дамп, режим символов) печатает каждый байт; `\0` — нулевой байт. Новые байты
файла были нулями, и мы изменили шесть из них.

> ⚡ **Производительность:** `flush()` ждёт диск, а это может занять миллисекунды. Вызывайте его, когда данные
> должны пережить сбой (о надёжности хранения — Урок 13.12), а не после каждой записи.

## 4. Приватные и анонимные отображения

**Приватные отображения.** Иногда нужно поменять данные в памяти, но оставить файл как есть. Приватное
отображение даёт ровно это: страницы сначала — страницы файла и копируются при первой записи, тот же трюк
с копированием при записи, что и после `fork`. Создайте `src/bin/private.rs`:

```rust
use std::fs::{self, File};

use memmap2::MmapOptions;

fn main() -> std::io::Result<()> {
    fs::write("note.txt", "original text\n")?;
    let file = File::open("note.txt")?;

    // SAFETY: only this program uses note.txt.
    let mut copy = unsafe { MmapOptions::new().map_copy(&file)? };
    copy[..8].copy_from_slice(b"CHANGED ");

    println!("in memory: {:?}", String::from_utf8_lossy(&copy));
    println!("on disk  : {:?}", fs::read_to_string("note.txt")?);
    Ok(())
}
```

```text
in memory: "CHANGED  text\n"
on disk  : "original text\n"
```

`map_copy` использует `MAP_PRIVATE`. Файл открыт только на чтение, и всё же мы смогли писать в отображение:
записанная страница — наша собственная копия. Так же загрузчик отображает глобальные переменные программы
(строки `rw-p` в `/proc/self/maps`; `p` означает private, приватное).

**Анонимные отображения.** Отображение без файла — это просто новая память, заполненная нулями. Одно такое вы
уже видели в Уроке 13.10: `vec![0u8; 256 MiB]` превратился в
`mmap(NULL, 268439552, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0)`. `-1` означает «без файла».
Аллокаторы памяти ([Урок 12.13](../12-computer-systems/13-how-memory-allocators-work.md)) так просят у ядра
большие блоки. В `memmap2` это `MmapMut::map_anon(len)`.

| Вид | Флаги | Изменения попадают в файл? | Типичное применение |
|---|---|---|---|
| общее отображение файла | `MAP_SHARED` + файл | да | базы данных, правка больших файлов, обмен данными между процессами |
| приватное отображение файла | `MAP_PRIVATE` + файл | нет (копирование при записи) | загрузка программ и библиотек |
| анонимное отображение | `MAP_ANONYMOUS` | файла нет | аллокаторы, большие буферы |

## 5. Быстрее ли `mmap`, чем `read`?

Часто говорят: «mmap быстрый». Честный ответ: *иногда*. Давайте измерим. Создайте `src/bin/compare.rs`:

```rust
use std::env;
use std::fs::File;
use std::io::Read;
use std::time::Instant;

use memmap2::Mmap;

fn count_with_read(path: &str) -> std::io::Result<usize> {
    let mut file = File::open(path)?;
    let mut buf = vec![0u8; 64 * 1024];
    let mut lines = 0;
    loop {
        let n = file.read(&mut buf)?; // the kernel copies up to 64 KiB into buf
        if n == 0 {
            return Ok(lines);
        }
        lines += buf[..n].iter().filter(|&&b| b == b'\n').count();
    }
}

fn count_with_mmap(path: &str) -> std::io::Result<usize> {
    let file = File::open(path)?;
    // SAFETY: nobody changes the file during the measurement.
    let map = unsafe { Mmap::map(&file)? };
    Ok(map.iter().filter(|&&b| b == b'\n').count())
}

fn main() -> std::io::Result<()> {
    let path = env::args().nth(1).expect("usage: compare <file>");
    for round in 1..=3 {
        let t = Instant::now();
        let a = count_with_read(&path)?;
        let read_ms = t.elapsed().as_millis();
        let t = Instant::now();
        let b = count_with_mmap(&path)?;
        let mmap_ms = t.elapsed().as_millis();
        assert_eq!(a, b);
        println!("round {round}: {a} lines | read: {read_ms:>4} ms | mmap: {mmap_ms:>4} ms");
    }
    Ok(())
}
```

Сделайте файл примерно на 250 МБ с помощью `seq` (она печатает числа от 1 до 30 000 000, по одному в строке)
и измерьте в режиме release:

```bash
seq 1 30000000 > big.txt
cargo run --release --bin compare -- big.txt
```

```text
round 1: 30000000 lines | read:  179 ms | mmap:   98 ms
round 2: 30000000 lines | read:  102 ms | mmap:   88 ms
round 3: 30000000 lines | read:   96 ms | mmap:   91 ms
```

**Что происходит:**

1. В раунде 1 `read` медленнее: часть страниц файла ещё не была в страничном кэше. Если измерить только первый
   запуск, вы обманете себя ([Урок 05.12](../05-rust-intermediate/12-benchmarking-basics.md)).
2. В раундах 2 и 3 разница мала. `read` копирует 250 МБ, но копировать память, которая уже в кэше, быстро.
   `mmap` избегает копирования, но взамен платит примерно за 63 000 страничных сбоев.
3. Ваши числа будут другими. Когда закончите, удалите `big.txt` (`rm big.txt`).

Практические правила:

- **`mmap` помогает**, когда вы прыгаете по большому файлу (произвольный доступ, как в индексе базы данных),
  когда много процессов читают один и тот же файл (они делят кадры страничного кэша) или когда хочется
  работать с файлом как с `&[u8]`, не управляя буферами.
- **`read` так же хорош или лучше**, когда файл читается один раз от начала до конца, для маленьких файлов
  (создание отображения тоже стоит времени) и для конвейеров, сокетов и терминалов, которые вообще нельзя
  отобразить.
- **`read` безопаснее:** он сообщает об ошибках через `Err`, а ошибка диска внутри отображения приходит
  сигналом.

## 6. Почему `Mmap::map` — это `unsafe`: SIGBUS

`&[u8]` в Rust обещает, что байты не меняются, пока вы держите ссылку. Отображение само по себе не может
сдержать это обещание: **другой процесс может в любой момент изменить или укоротить файл**, и изменение
появится в вашем срезе. Компилятор не может проверить другие процессы, поэтому `memmap2` заставляет вас
написать `unsafe` и пообещать это самому.

Укорачивание — худший случай. За страницами после нового конца файла нет данных. Обращение к ним вызывает
**SIGBUS**, и процесс умирает. Создайте `src/bin/sigbus.rs`, который делает это намеренно:

```rust
use std::fs::OpenOptions;

use memmap2::Mmap;

fn main() -> std::io::Result<()> {
    let file = OpenOptions::new()
        .read(true)
        .write(true)
        .create(true)
        .truncate(true)
        .open("shrinking.bin")?;
    file.set_len(8192)?;

    // SAFETY: deliberately broken! We shorten the file below while it is mapped.
    let map = unsafe { Mmap::map(&file)? };
    println!("byte 5000 before: {}", map[5000]);

    file.set_len(0)?; // the file is now empty, but the mapping is still 8192 bytes long
    println!("file shortened, reading byte 5000 again ...");
    println!("byte 5000 after: {}", map[5000]);
    Ok(())
}
```

```bash
cargo run --bin sigbus
echo "exit=$?"
```

```text
byte 5000 before: 0
file shortened, reading byte 5000 again ...
Bus error
exit=135
```

Индекс `5000` внутри среза (его длина по-прежнему 8192), поэтому проверка границ в Rust проходит. Падение
вызывает ядро: 135 = 128 + 7, а 7 — это SIGBUS. В настоящей программе «другим процессом» может оказаться
утилита ротации логов или пользователь, который запустил `truncate` для этого файла.

> 🔒 **Безопасность:** никогда не отображайте файлы, в которые могут писать другие, менее доверенные
> пользователи, если вы сначала не копируете данные и не проверяете их. Злоумышленник, который может писать в файл,
> способен изменить байты после вашей проверки или укоротить файл, чтобы уронить ваш сервис.

## Частые ошибки

1. **Отображение до задания размера.** Отображение нового пустого файла через `MmapMut::map_mut` даёт
   отображение длины 0, и `map[0] = 1` паникует с `index out of bounds: the len is 0 but the index is 0`.
   *Исправление:* сначала вызовите `file.set_len(n)`.

2. **Файл открыт только на чтение, а отображается на запись.** `MmapMut::map_mut` для файла, открытого
   через `File::open`, завершается ошибкой:
   `Error: Os { code: 13, kind: PermissionDenied, message: "Permission denied" }`.
   *Исправление:* используйте `OpenOptions` с `.read(true)` и `.write(true)` или `map_copy`, если менять
   файл не нужно.

3. **Ожидание, что `flush` после каждого изменения бесплатен.** Каждый `flush` ждёт диск. *Исправление:*
   группируйте изменения и сбрасывайте один раз — там, где данные должны стать надёжно сохранёнными.

4. **Попытка отобразить конвейер.** `cat small.txt | ./target/debug/mcount /dev/stdin` завершается ошибкой
   `Error: Os { code: 19, kind: Uncategorized, message: "No such device" }`, потому что у конвейера нет
   страниц, которые можно отобразить. *Исправление:* для потоков используйте `read`; `mmap` — только для
   обычных файлов.

## Попробуйте сами

1. Запустите `mwrite`, затем посмотрите на тот же файл двумя способами: `od -c counter.bin` и `cat counter.bin`.
   `cat` показывает `HELLO`, затем невидимые нулевые байты, затем перевод строки.
2. Посмотрите на отображения работающей программы. Добавьте
   `std::thread::sleep(std::time::Duration::from_secs(30));` в конец `mcount`, запустите её командой
   `./target/debug/mcount big.txt &`, затем выполните `grep big.txt /proc/$!/maps` (`$!` — PID последней
   фоновой программы). Вы увидите строку вроде
   `ffffa4dab000-ffffa6380000 r--s 00000000 00:34 320408  /home/.../big.txt`: только чтение и **s** — shared, общее.
3. Выполните `cargo run --bin mcount -- /dev/stdin < small.txt`. Перенаправление обычного файла работает,
   потому что `/dev/stdin` тогда указывает на сам файл. Теперь попробуйте
   `cat small.txt | cargo run --bin mcount -- /dev/stdin` и сравните ошибку с частой ошибкой 4.

## Упражнения

### Упражнение 1 — Подсчёт через протестированную функцию (легко)

Вынесите подсчёт из `main` в `pub fn count_byte(data: &[u8], needle: u8) -> usize` в `src/lib.rs` и
протестируйте её на `b"a\nb\nc\n"`, пустом срезе и тексте без переводов строк. Используйте её в `mcount`.

<details>
<summary>Подсказка</summary>

Функции вообще не нужно знать об отображениях: отображение *и есть* `&[u8]`. В `mcount` вызовите
`p13_mmap::count_byte(&map, b'\n')`.

</details>

<details>
<summary>Ответ</summary>

```rust
/// Counts how many times `needle` occurs in `data`.
pub fn count_byte(data: &[u8], needle: u8) -> usize {
    data.iter().filter(|&&b| b == needle).count()
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn counts_newlines() {
        assert_eq!(count_byte(b"a\nb\nc\n", b'\n'), 3);
        assert_eq!(count_byte(b"", b'\n'), 0);
        assert_eq!(count_byte(b"no newline", b'\n'), 0);
    }
}
```

Почему это работает: раз `Mmap` разыменовывается в `&[u8]`, всю логику можно писать и тестировать на обычных
срезах. Только `main` знает, что байты приходят из отображения.

</details>

### Упражнение 2 — `mgrep` (средне)

Напишите `src/bin/mgrep.rs`: `mgrep <word> <file>` отображает файл и печатает номера (начиная с 1) строк,
в которых есть это слово. Поиск вынесите в `pub fn matching_lines(data: &[u8], word: &[u8]) -> Vec<usize>`
с тестом. Пустое слово не совпадает ни с чем. Для файла со строками `apple pie`, `banana`, `pineapple` и
слова `apple` вывод — `1` и `3`.

<details>
<summary>Подсказка</summary>

`data.split(|&b| b == b'\n')` даёт строки как `&[u8]`. `line.windows(word.len())` даёт каждый кусок строки
длиной со слово; сравните каждый с `word`.

</details>

<details>
<summary>Ответ</summary>

В `src/lib.rs`:

```rust
/// Returns the numbers (starting at 1) of the lines that contain `word`.
pub fn matching_lines(data: &[u8], word: &[u8]) -> Vec<usize> {
    data.split(|&b| b == b'\n')
        .enumerate()
        .filter(|(_, line)| !word.is_empty() && line.windows(word.len()).any(|w| w == word))
        .map(|(i, _)| i + 1)
        .collect()
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn finds_line_numbers() {
        let text = b"apple pie\nbanana\npineapple\n";
        assert_eq!(matching_lines(text, b"apple"), vec![1, 3]);
        assert_eq!(matching_lines(text, b"cherry"), Vec::<usize>::new());
        assert_eq!(matching_lines(text, b""), Vec::<usize>::new());
    }
}
```

`src/bin/mgrep.rs`:

```rust
use std::env;
use std::fs::File;

use memmap2::Mmap;
use p13_mmap::matching_lines;

fn main() -> std::io::Result<()> {
    let args: Vec<String> = env::args().collect();
    if args.len() != 3 {
        eprintln!("usage: mgrep <word> <file>");
        std::process::exit(2);
    }
    let file = File::open(&args[2])?;
    // SAFETY: we assume nobody shortens the file while we read it.
    let map = unsafe { Mmap::map(&file)? };
    for n in matching_lines(&map, args[1].as_bytes()) {
        println!("{n}");
    }
    Ok(())
}
```

```bash
printf 'apple pie\nbanana\npineapple\n' > fruit.txt
cargo run --bin mgrep -- apple fruit.txt
```

```text
1
3
```

Почему это работает: `windows(n)` сдвигает окно из `n` байт по строке, поэтому совпадение находится в любом
месте строки. Проверка на пустое слово идёт первой, потому что `windows(0)` паникует.

</details>

### Упражнение 3 — Перевести файл в верхний регистр на месте (сложно)

Напишите `pub fn upcase_file(path: &Path) -> io::Result<()>`, которая превращает каждую ASCII-букву (ASCII, American Standard Code for Information Interchange, Урок 12.5) файла в
заглавную **на месте**, через записываемое общее отображение, и маленький бинарник `upcase <file>`. Пустой
файл должен работать. Напишите тест, который создаёт временный файл с `hello, mmap!\n`, вызывает функцию и
проверяет результат `HELLO, MMAP!\n`.

<details>
<summary>Подсказка</summary>

`u8::make_ascii_uppercase()` меняет один байт. Для тестового файла используйте
`std::env::temp_dir().join(format!("upcase-test-{}.txt", std::process::id()))`, чтобы параллельные запуски
тестов не мешали друг другу, и удалите файл в конце.

</details>

<details>
<summary>Ответ</summary>

В `src/lib.rs`:

```rust
use std::fs::OpenOptions;
use std::io;
use std::path::Path;

use memmap2::MmapMut;

/// Turns ASCII letters a-z into A-Z, in place.
pub fn upcase_in_place(bytes: &mut [u8]) {
    for b in bytes.iter_mut() {
        b.make_ascii_uppercase();
    }
}

/// Upper-cases a whole file in place through a shared, writable mapping.
pub fn upcase_file(path: &Path) -> io::Result<()> {
    let file = OpenOptions::new().read(true).write(true).open(path)?;
    if file.metadata()?.len() == 0 {
        return Ok(()); // nothing to change
    }
    // SAFETY: we assume no other process shortens the file while we work on it.
    let mut map = unsafe { MmapMut::map_mut(&file)? };
    upcase_in_place(&mut map);
    map.flush()
}

#[cfg(test)]
mod tests {
    use super::*;
    use std::fs;

    #[test]
    fn upcases_a_real_file() -> io::Result<()> {
        let path = std::env::temp_dir().join(format!("upcase-test-{}.txt", std::process::id()));
        fs::write(&path, "hello, mmap!\n")?;
        upcase_file(&path)?;
        let result = fs::read_to_string(&path)?;
        fs::remove_file(&path)?;
        assert_eq!(result, "HELLO, MMAP!\n");
        Ok(())
    }
}
```

`src/bin/upcase.rs`:

```rust
use std::env;
use std::path::Path;

fn main() -> std::io::Result<()> {
    let path = env::args().nth(1).expect("usage: upcase <file>");
    p13_mmap::upcase_file(Path::new(&path))
}
```

`cargo test` печатает `test result: ok.` для тестов библиотеки, а `upcase fruit.txt` превращает файл с
фруктами в `APPLE PIE`, `BANANA`, `PINEAPPLE`.

Почему это работает: отображение `MAP_SHARED`, поэтому изменение байтов меняет файл в страничном кэше, а
`flush` записывает его на диск. Ничего не читалось в буфер и не записывалось обратно через `write`.

</details>

## Вопросы для самопроверки

1. В чём главное различие между `read` и `mmap` для данных файла?

<details>
<summary>Ответ</summary>

`read` копирует данные из страничного кэша в буфер программы. `mmap` отображает страницы страничного кэша
прямо в адресное пространство, и программа читает их на месте, без копирования и без новых системных
вызовов.

</details>

2. Почему перед созданием записываемого отображения нового файла нужно вызвать `set_len`?

<details>
<summary>Ответ</summary>

Отображение не может увеличить файл. У нового файла длина 0, значит, и его отображение было бы длины 0. Файл
должен уже иметь тот размер, в который вы хотите писать.

</details>

3. Вы меняете приватное отображение (`map_copy`). Что происходит с файлом?

<details>
<summary>Ответ</summary>

Ничего. Первая запись в каждую страницу создаёт приватную копию (копирование при записи); ни файл, ни другие
процессы изменений не увидят.

</details>

4. Почему `Mmap::map` — это `unsafe`-функция?

<details>
<summary>Ответ</summary>

Потому что полученный `&[u8]` может измениться или исчезнуть за спиной у Rust: другой процесс может записать
в файл или укоротить его. Rust не может это проверить, поэтому пообещать должен программист. Укороченный
файл даже приводит к SIGBUS.

</details>

5. Назовите одну ситуацию, где `mmap` — хороший выбор, и одну, где лучше `read`.

<details>
<summary>Ответ</summary>

`mmap`: произвольный доступ к большому файлу или много процессов, которые делят одни и те же данные файла.
`read`: однократное чтение файла от начала до конца, маленькие файлы, а также конвейеры и сокеты, которые
нельзя отобразить.

</details>

## Итоги

- `mmap` делает файл частью адресного пространства; его страницы приходят из страничного кэша через страничные сбои, без копирования в буфер.
- В Rust `memmap2::Mmap` даёт `&[u8]`, а `MmapMut` — `&mut [u8]`; `Drop` снимает отображение.
- Общие отображения меняют файл (`flush` = `msync` делает изменения надёжными); приватные работают с копированием при записи; анонимные — это новая память с нулями, которую используют аллокаторы.
- Записываемому отображению нужен файл, открытый на чтение и запись и уже нужного размера.
- `mmap` не быстрее автоматически: измеряйте. Он хорош для произвольного доступа и совместного использования; для последовательного чтения подходит `read`.
- Создание отображения — `unsafe`, потому что другие процессы могут изменить или укоротить файл; обращение к страницам за концом даёт SIGBUS.

## Источники и что почитать

- [mmap(2) — Linux manual page](https://man7.org/linux/man-pages/man2/mmap.2.html) — все флаги `mmap`, в том числе `MAP_SHARED`, `MAP_PRIVATE` и случай SIGBUS.
- [msync(2) — Linux manual page](https://man7.org/linux/man-pages/man2/msync.2.html) — что значит сбросить отображение.
- [memmap2 на docs.rs](https://docs.rs/memmap2/0.9.12/memmap2/) — крейт из этого урока с замечаниями о безопасности для `Mmap::map`.
- [The Linux kernel documentation — Page Cache](https://docs.kernel.org/mm/page_cache.html) — как ядро держит данные файлов в RAM.
- [Operating Systems: Three Easy Pieces](https://pages.cs.wisc.edu/~remzi/OSTEP/) — глава «Files and Directories» и главы о виртуальной памяти дают общую картину.

---

[🏠 Начало](../START-HERE.md) · [← 13.10 Виртуальная память](10-virtual-memory.md) · [📋 Этап 13](README.md) · [13.12 Файловые системы изнутри →](12-file-systems-inside.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
