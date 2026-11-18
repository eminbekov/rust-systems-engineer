<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 13.11 Файлы, отображённые в память](11-memory-mapped-files.md) · [📋 Этап 13](README.md) · [13.13 Неблокирующий ввод-вывод и epoll →](13-non-blocking-io-and-epoll.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/13-operating-systems/12-file-systems-inside.md) <!-- nav -->

# Урок 13.12 — Файловые системы изнутри

> **Этап 13 — Операционные системы изнутри** · Урок 12 из 18 · ⏱ около 2,5 часа (2 занятия)

## Что вы узнаете

- Как слой ядра VFS (Virtual File System, виртуальная файловая система) позволяет одному набору системных вызовов работать со многими видами файловых систем.
- Что такое inode, что на самом деле содержит каталог и как прочитать и то и другое из Rust.
- Чем жёсткие ссылки отличаются от символических и что на самом деле делает «удаление» файла.
- Почему `write` не означает «на диске» и как `fsync` и приём атомарной замены помогают данным пережить сбой.

## Перед началом

- Вы прошли [Урок 13.11 — Файлы, отображённые в память](11-memory-mapped-files.md) и знаете страничный кэш.
- Вы знаете файлы и дескрипторы ([Урок 13.4](04-files-and-file-descriptors.md)), дерево каталогов Linux ([Урок 11.2](../11-linux-shell-and-containers/02-the-linux-file-system.md)) и права доступа ([Урок 11.4](../11-linux-shell-and-containers/04-users-groups-and-permissions.md)).
- Создайте `code/p13-fs` на Mac и скопируйте его в ВМ, как в Уроке 13.10. На этот раз это библиотека с бинарниками:

```bash
cd ~/projects/rust
cargo new --lib code/p13-fs
mkdir code/p13-fs/src/bin
```

Примеры используют только стандартную библиотеку. Они запускаются в ВМ, в маленькой файловой системе,
которую вы создадите в разделе 2. Выводы ниже получены на тестовой машине, где пользователь — `root`; в вашей
ВМ вы увидите имя своего пользователя.

## Новые слова

| Слово | Значение |
|---|---|
| **файловая система** (file system) | Способ организовать файлы на носителе: где лежат блоки данных, какие есть имена, кто может их читать. Обычная в Linux — ext4. |
| **VFS** (Virtual File System, виртуальная файловая система) | Слой в ядре, который предлагает одинаковые вызовы (`open`, `read`, `rename`…) для любой файловой системы и передаёт их нужной. |
| **монтирование** (mount) | Подключение файловой системы к каталогу (*точке монтирования*), чтобы её файлы появились там. |
| **блочное устройство** (block device) | Устройство, которое хранит данные блоками фиксированного размера, например диск. *Loop-устройство* делает обычный файл похожим на такое устройство. |
| **inode** (index node, индексный узел) | Запись об одном файле внутри файловой системы: тип, размер, владелец, права, времена и где лежат данные. У неё есть номер, но нет имени. |
| **запись каталога** (directory entry) | Одна строка каталога: имя и номер inode, на который оно указывает. |
| **жёсткая ссылка** (hard link) | Вторая запись каталога для того же inode. Все имена равноправны; ни одно не «оригинал». |
| **символическая ссылка** (symbolic link, symlink) | Маленький особый файл, в котором записан путь к другому файлу. |
| **счётчик ссылок** (link count) | Сколько записей каталогов указывает на inode. Данные освобождаются, когда он доходит до 0 и ни один процесс не держит файл открытым. |
| **unlink** | Системный вызов за «удалением»: он убирает одну запись каталога. |
| **надёжность хранения** (durability) | Обещание, что данные переживут сбой или отключение питания, после того как программе ответили «сохранено». |
| **`fsync`** | Системный вызов, который ждёт, пока данные и метаданные файла окажутся на носителе. В Rust: `File::sync_all`. |
| **журнал** (journal) | Область файловой системы, куда ext4 сначала записывает, что собирается изменить, чтобы сбой не оставил структуру изменённой наполовину. |
| **разреженный файл** (sparse file) | Файл с «дырами»: участками, в которые никогда не писали и которые не занимают блоков на диске. |

## 1. Один интерфейс, много файловых систем

Когда ваша программа вызывает `open("/proc/uptime")`, `open("/home/student/notes.txt")` или
`open("/sys/fs/cgroup/cpu.max")`, она пользуется одним и тем же системным вызовом. При этом первый файл ядро
придумывает на лету, второй хранится на диске, а третий управляет ограничениями ресурсов (Урок 13.16). Часть
ядра, которая делает это возможным, — **VFS**:

```text
   your program:  open  read  write  rename  unlink  stat ...
                    |
           +--------v---------+
           |       VFS        |   same calls for everything
           +--+-----+-----+---+
              |     |     |
          +---v-+ +-v---+ +v-------+
          | ext4| |proc | | tmpfs  |  ... overlayfs, cgroup2, nfs
          +--+--+ +-----+ +--------+
             |     (no disk: RAM / kernel data)
     +-------v-------+
     |  block device |  a disk, or a loop device backed by a file
     +---------------+
```

Каждая файловая система **смонтирована** в какой-то каталог. `findmnt` показывает точки монтирования, а
`/proc/self/mounts` — это их «сырой» список. Контейнеры (Урок 13.15) активно этим пользуются: весь их корневой
каталог — это монтирование файловой системы *overlayfs*.

## 2. Создаём маленькую файловую систему для исследования

На основном диске ВМ тысячи файлов. Свежую крошечную файловую систему ext4 понять проще, и потом её можно
удалить. Файл может работать как диск через **loop-устройство**. В ВМ:

```bash
cd ~
truncate -s 64M disk.img          # an empty 64 MiB file
mkfs.ext4 -q disk.img             # write an empty ext4 file system into it
sudo mkdir -p /mnt/small
sudo mount -o loop disk.img /mnt/small
sudo chown "$USER" /mnt/small     # let your user create files there
findmnt -n -o TARGET,FSTYPE,SOURCE /mnt/small
cd /mnt/small
ls -la
```

```text
/mnt/small ext4 /dev/loop0
total 17
drwxr-xr-x 3 root root  1024 Nov 18 07:11 .
drwxr-xr-x 1 root root  4096 Nov 18 07:11 ..
drwx------ 2 root root 12288 Nov 18 07:11 lost+found
```

**Что происходит:**

1. `truncate -s 64M` создаёт файл на 64 МиБ, не записывая данных (подробнее — в «Попробуйте сами»).
2. `mkfs.ext4` («make file system», создать файловую систему) записывает в файл структуры ext4: таблицы
   inode, карты свободных блоков, журнал и пустой корневой каталог.
3. `mount -o loop` подключает файл к loop-устройству `/dev/loop0` и монтирует файловую систему в
   `/mnt/small`. Монтирование требует прав root, поэтому используется `sudo`.
4. В новой файловой системе уже есть `lost+found` — каталог, куда утилита восстановления `fsck` кладёт куски
   файлов, найденные после сбоя.

Посмотрите на её ёмкость:

```bash
df -h .
df -i .
```

```text
Filesystem      Size  Used Avail Use% Mounted on
/dev/loop0       55M   46K   51M   1% /mnt/small
Filesystem     Inodes IUsed IFree IUse% Mounted on
/dev/loop0      16384    12 16372    1% /mnt/small
```

`df -h` показывает место; `df -i` показывает **inode**. ext4 создаёт фиксированное число inode при создании
файловой системы, здесь 16 384. Каждому файлу нужен один. Запомните это; мы вернёмся к этому в «Попробуйте сами».

## 3. Inode и каталоги

Файл в ext4 состоит из двух отдельных частей:

- **inode**: запись с типом файла, размером, владельцем, правами, временами, счётчиком ссылок и адресами его
  блоков данных. У inode есть **номер**, но **нет имени**.
- Одна или несколько **записей каталогов**, которые дают ему имя.

**Каталог** — это просто особый файл, содержимое которого — таблица записей:

```text
directory /mnt/small (inode 2)            inode table
+-------------+-------+                   +-------+------+------+------+----------------+
| name        | inode |                   | inode | type | size | links| data blocks    |
+-------------+-------+                   +-------+------+------+------+----------------+
| .           | 2     |                   | 2     | dir  | 1024 | 4    | block 1201     |
| ..          | ...   |                   | 13    | file | 3    | 1    | block 8705     |
| lost+found  | 11    |                   | 2049  | dir  | 1024 | 2    | ...            |
| notes.txt   | 13    |  ------------->   | ...   |      |      |      |                |
| sub         | 2049  |                   +-------+------+------+------+----------------+
+-------------+-------+
```

Открыть `/mnt/small/notes.txt` значит: найти корневой каталог, найти в нём запись `mnt`, затем `small`, затем
`notes.txt` и прочитать inode 13. Создайте несколько файлов:

```bash
echo hi > notes.txt
mkdir sub
ln -s notes.txt link
```

Rust показывает данные inode через трейт `std::os::unix::fs::MetadataExt`, который добавляет к `Metadata`
методы, существующие только в Unix. Создайте `src/bin/inspect.rs`:

```rust
use std::env;
use std::fs;
use std::os::unix::fs::MetadataExt;

fn main() -> std::io::Result<()> {
    for path in env::args().skip(1) {
        let meta = fs::symlink_metadata(&path)?; // do not follow symbolic links
        let kind = if meta.is_dir() {
            "directory"
        } else if meta.file_type().is_symlink() {
            "symlink"
        } else {
            "file"
        };
        println!(
            "{path}: {kind}, inode {}, links {}, size {} bytes, mode {:o}",
            meta.ino(),
            meta.nlink(),
            meta.size(),
            meta.mode() & 0o7777
        );
    }
    Ok(())
}
```

Соберите его в `~/p13-fs` и запустите из `/mnt/small`:

```bash
~/p13-fs/target/debug/inspect notes.txt sub link .
```

```text
notes.txt: file, inode 13, links 1, size 3 bytes, mode 644
sub: directory, inode 2049, links 2, size 1024 bytes, mode 755
link: symlink, inode 14, links 1, size 9 bytes, mode 777
.: directory, inode 2, links 4, size 1024 bytes, mode 755
```

**Что происходит:**

1. `fs::symlink_metadata` вызывает `lstat`, который описывает саму ссылку. `fs::metadata` (`stat`) прошёл бы
   по `link` и описал бы `notes.txt`.
2. `ino()` — номер inode, `nlink()` — счётчик ссылок, `size()` — размер в байтах. `mode()` содержит тип и биты
   прав; `& 0o7777` оставляет права, напечатанные в восьмеричном виде (Урок 11.4).
3. У `link` размер 9: её содержимое — 9 байт текста `notes.txt`. Её режим 777 не используется; доступ
   проверяется у цели.
4. У корневого каталога inode 2 (ext4 всегда использует 2) и 4 ссылки: запись `small` в `/mnt`, его
   собственная `.` и `..` внутри `lost+found` и `sub`. Новый каталог, как `sub`, всегда начинает с 2.

`ls -i` и `stat` показывают те же данные:

```bash
ls -li
stat notes.txt
```

```text
total 14
  14 lrwxrwxrwx 1 root root     9 Nov 18 07:11 link -> notes.txt
  11 drwx------ 2 root root 12288 Nov 18 07:11 lost+found
  13 -rw-r--r-- 1 root root     3 Nov 18 07:11 notes.txt
2049 drwxr-xr-x 2 root root  1024 Nov 18 07:11 sub
  File: notes.txt
  Size: 3         	Blocks: 2          IO Block: 1024   regular file
Device: 7,0	Inode: 13          Links: 1
Access: (0644/-rw-r--r--)  Uid: (    0/    root)   Gid: (    0/    root)
Access: 2026-11-18 07:11:54.548525013 +0000
Modify: 2026-11-18 07:11:54.548525013 +0000
Change: 2026-11-18 07:11:54.548525013 +0000
 Birth: 2026-11-18 07:11:54.548525013 +0000
```

`Blocks: 2` считает единицы по 512 байт: файл из 3 байт всё равно занимает целый блок в 1 КиБ этой маленькой
файловой системы. `Device: 7,0` определяет файловую систему (loop-устройство). Номер inode уникален только
*внутри* одной файловой системы, поэтому файл определяется парой (устройство, inode).

## 4. Жёсткие ссылки, символические ссылки и удаление

Создайте `src/bin/links.rs`:

```rust
use std::fs;
use std::os::unix::fs::{MetadataExt, symlink};

fn show(name: &str) -> std::io::Result<()> {
    let meta = fs::symlink_metadata(name)?;
    println!("{name:<10} inode {:>8}  links {}", meta.ino(), meta.nlink());
    Ok(())
}

fn main() -> std::io::Result<()> {
    fs::create_dir_all("demo")?;
    std::env::set_current_dir("demo")?;
    for old in ["original", "hardlink", "softlink"] {
        let _ = fs::remove_file(old); // start clean; ignore "not found"
    }

    fs::write("original", "hello\n")?;
    fs::hard_link("original", "hardlink")?; // a second name for the same inode
    symlink("original", "softlink")?; // a new small file that stores a path
    for name in ["original", "hardlink", "softlink"] {
        show(name)?;
    }
    println!("softlink points to {:?}", fs::read_link("softlink")?);

    fs::remove_file("original")?;
    println!("after removing 'original':");
    println!("  hardlink -> {:?}", fs::read_to_string("hardlink"));
    println!(
        "  softlink -> {:?}",
        fs::read_to_string("softlink").map_err(|e| e.kind())
    );
    Ok(())
}
```

Запустите в `/mnt/small` (`~/p13-fs/target/debug/links`):

```text
original   inode       15  links 2
hardlink   inode       15  links 2
softlink   inode       16  links 1
softlink points to "original"
after removing 'original':
  hardlink -> Ok("hello\n")
  softlink -> Err(NotFound)
```

**Что происходит:**

1. `fs::hard_link` добавляет вторую запись каталога для inode 15. Оба имени показывают один и тот же inode и
   счётчик ссылок 2. Ни одно из них не «настоящее» больше другого.
2. `symlink` создаёт *новый* inode (16), содержимое которого — путь `original`. `fs::read_link` читает этот
   путь, не переходя по нему.
3. `fs::remove_file` вызывает `unlink`: он убирает запись `original` и уменьшает счётчик ссылок до 1. Данные
   остаются, потому что `hardlink` всё ещё указывает на inode.
4. Символическая ссылка теперь указывает на несуществующее имя: она *висячая* (dangling), и при открытии
   получается `NotFound`.

| | жёсткая ссылка | символическая ссылка |
|---|---|---|
| что это | ещё одна запись для того же inode | файл, в котором записан путь |
| цель удалена | продолжает работать | висячая, `NotFound` |
| между файловыми системами | нет (номера inode свои в каждой файловой системе) | да |
| на каталог | нельзя | можно |

**Удаление открытого файла.** Системного вызова «удалить» нет, есть только `unlink`. Ядро освобождает inode
и его блоки, когда **одновременно** счётчик ссылок равен 0 **и** ни один процесс не держит файл открытым.
Создайте `src/bin/unlinked.rs`:

```rust
use std::fs::{self, File};
use std::io::{Read, Seek, SeekFrom, Write};
use std::os::fd::AsRawFd;

fn main() -> std::io::Result<()> {
    let mut file = File::options()
        .read(true)
        .write(true)
        .create(true)
        .truncate(true)
        .open("secret.tmp")?;
    file.write_all(b"still here\n")?;

    fs::remove_file("secret.tmp")?; // the name is gone ...
    println!("exists by name? {}", fs::exists("secret.tmp")?);
    let link = fs::read_link(format!("/proc/self/fd/{}", file.as_raw_fd()))?;
    println!("the descriptor points to {link:?}");

    file.seek(SeekFrom::Start(0))?; // ... but the inode lives while it is open
    let mut text = String::new();
    file.read_to_string(&mut text)?;
    print!("read through the open descriptor: {text}");
    Ok(())
}
```

```text
exists by name? false
the descriptor points to "/mnt/small/secret.tmp (deleted)"
read through the open descriptor: still here
```

`/proc/self/fd/3` — символическая ссылка, которую ядро создаёт для каждого открытого дескриптора (Урок 13.4);
она помечает файл как `(deleted)`. Это объясняет классическую загадку на серверах: вы удаляете лог-файл на
10 ГБ, а `df` по-прежнему показывает, что диск заполнен. Работающая программа всё ещё держит его открытым.
Место вернётся, когда эта программа закроет файл или завершится.

> 🔒 **Безопасность:** программы пользуются этим намеренно для временных файлов: создать, открыть и сразу
> выполнить unlink. Никакой другой процесс не найдёт файл по имени, а данные исчезнут, даже если программа
> упадёт.

## 5. Надёжность хранения: когда данные действительно сохранены?

`write` возвращает управление, как только данные оказались в **страничном кэше** (Урок 13.11). «Грязные»
страницы ядро пишет на диск позже, обычно в течение примерно 30 секунд. Если питание пропадёт раньше, данные
потеряются, хотя `write` вернул успех.

`fsync` (в Rust `File::sync_all`) ждёт, пока данные и метаданные файла окажутся на устройстве. Это медленно —
миллисекунды вместо микросекунд, — поэтому программы вызывают его только там, где это важно.

Есть и вторая проблема. Представьте, что программа сохраняет свои настройки так:

```text
1. open settings.txt with truncate   -> file is now empty
2. write the new content             -> crash here = empty or half-written file
```

Сбой между шагами уничтожает и старые настройки, *и* новые. Стандартное решение — **приём атомарной замены**
(atomic replace), который опирается на одну гарантию: `rename` внутри одной файловой системы заменяет цель за
один шаг. Любой, кто открывает это имя, видит либо старый файл, либо новый, но никогда не смесь. Положите
это в `src/lib.rs`:

```rust
use std::fs::{self, File};
use std::io::{self, Write};
use std::path::Path;

/// Replaces the file at `path` with `data` so that, even after a crash, the file holds
/// either the complete old content or the complete new content, never a mix.
pub fn atomic_write(path: &Path, data: &[u8]) -> io::Result<()> {
    let dir = path.parent().unwrap_or(Path::new("."));
    let dir = if dir.as_os_str().is_empty() {
        Path::new(".")
    } else {
        dir
    };
    let file_name = path
        .file_name()
        .ok_or_else(|| io::Error::new(io::ErrorKind::InvalidInput, "path has no file name"))?;
    let tmp = dir.join(format!(".{}.tmp", file_name.to_string_lossy()));

    let mut file = File::create(&tmp)?; // 1. write a temporary file in the same directory
    file.write_all(data)?;
    file.sync_all()?; // 2. make sure its data is on the disk
    drop(file);

    fs::rename(&tmp, path)?; // 3. atomically give it the real name
    File::open(dir)?.sync_all()?; // 4. make the new directory entry durable too
    Ok(())
}
```

**Что происходит:**

1. Временный файл лежит в **том же каталоге**, а значит, в той же файловой системе; иначе `rename` не
   сработает (см. «Частые ошибки»). Для `settings.txt` `parent()` — пустой путь, который мы превращаем в `.`.
2. `sync_all` делает новое содержимое надёжно сохранённым *до того*, как оно получит настоящее имя.
3. `fs::rename` переключает имя за один шаг. До этого момента старый файл не тронут.
4. Переименование изменило *каталог*, а это тоже данные. Открыв каталог и вызвав для него `sync_all`, мы
   делаем надёжной и новую запись.

Маленькая программа, которая этим пользуется, `src/bin/save.rs`:

```rust
use std::path::Path;

fn main() -> std::io::Result<()> {
    p13_fs::atomic_write(Path::new("settings.txt"), b"volume = 7\n")?;
    println!("saved");
    Ok(())
}
```

Посмотрите на системные вызовы в `/mnt/small`:

```bash
strace -e trace=openat,write,fsync,renameat ~/p13-fs/target/debug/save
```

Важные строки (загрузка библиотек в начале вырезана):

```text
write(3, "volume = 7\n", 11)            = 11
fsync(3)                                = 0
renameat(AT_FDCWD, "./.settings.txt.tmp", AT_FDCWD, "settings.txt") = 0
openat(AT_FDCWD, ".", O_RDONLY|O_CLOEXEC) = 3
fsync(3)                                = 0
```

ext4 защищает и *собственные* структуры с помощью **журнала**: прежде чем менять inode и каталоги, она
записывает в журнал заметку об изменении. После сбоя она доигрывает или отбрасывает эти заметки, и файловая
система остаётся согласованной. Журнал не защищает *содержимое вашего файла*; это задача `fsync` и приёма
выше. Базы данных (Этап 17) строятся ровно на этих инструментах.

> 🧪 **Тестирование:** «сбой посередине» протестировать нелегко, но можно проверить видимый контракт: новое
> содержимое на месте, временного файла не осталось, ошибки сообщаются. Это делает упражнение 3.

## Частые ошибки

1. **Переименование между файловыми системами.** Если записать временный файл в `/tmp` и переименовать его в
   другую файловую систему, будет ошибка:

   ```rust
   use std::fs;

   fn main() -> std::io::Result<()> {
       fs::write("/tmp/draft.txt", "text\n")?;
       fs::rename("/tmp/draft.txt", "/mnt/small/draft.txt")?;
       Ok(())
   }
   ```

   ```text
   Error: Os { code: 18, kind: CrossesDevices, message: "Invalid cross-device link" }
   ```

   Переименование меняет только записи каталогов, а inode не может переехать в другую файловую систему.
   Команда `mv` скрывает это, копируя и удаляя, а это **не** атомарно. *Исправление:* создавайте временный
   файл рядом с целью.

2. **«Я вызвал `write`, значит, сохранено».** Это обещает только `sync_all` (или `sync_data`).
   *Исправление:* вызывайте его там, где важна надёжность, и не забывайте про каталог после переименования.

3. **Случайный переход по символической ссылке.** `fs::metadata` проходит по символической ссылке и описывает
   цель, поэтому проверка вроде «это каталог?» отвечает про цель, а не про ссылку. *Исправление:*
   используйте `fs::symlink_metadata`, когда нужно узнать о самой ссылке.

4. **Диск «заполнен», хотя место есть.** `No space left on device` может означать «нет свободных inode».
   *Исправление:* проверяйте `df -i` вместе с `df -h`.

## Попробуйте сами

1. **Израсходуйте все inode.** В `/mnt/small`:

   ```bash
   mkdir many && cd many
   for i in $(seq 1 17000); do touch f$i || break; done
   df -h /mnt/small; df -i /mnt/small
   ```

   ```text
   touch: cannot touch 'f16365': No space left on device
   Filesystem      Size  Used Avail Use% Mounted on
   /dev/loop0       55M  420K   50M   1% /mnt/small
   Filesystem     Inodes IUsed IFree IUse% Mounted on
   /dev/loop0      16384 16384     0  100% /mnt/small
   ```

   Диск заполнен на 1%, но новый файл создать нельзя. Уберите за собой: `cd .. && rm -r many`.

2. **Посмотрите на разреженный файл.** `truncate` не записал данных, поэтому в файле-образе есть дыры:

   ```bash
   cd ~
   ls -lh disk.img
   du -h disk.img
   ```

   ```text
   -rw-r--r-- 1 root root 64M Nov 18 07:14 disk.img
   8.5M	disk.img
   ```

   `ls` показывает видимый размер; `du` — реально занятые блоки.

3. **Уберите за собой** только после того, как выполните упражнения ниже:

   ```bash
   cd ~
   sudo umount /mnt/small
   rm disk.img
   ```

## Упражнения

Выполняйте их, пока `/mnt/small` ещё смонтирован.

### Упражнение 1 — Посчитайте ссылки каталога (легко)

После шага 1 из «Попробуйте сами» (когда `many` уже снова удалён) в `/mnt/small` выполните
`ls -lid /mnt/small /mnt/small/. /mnt/small/sub/..`. Почему все три строки показывают один и тот же inode и
почему счётчик ссылок теперь 5 (в разделе 3 он был 4)?

<details>
<summary>Подсказка</summary>

Какие каталоги вы создали после раздела 3? В каждом каталоге есть запись `..`, которая указывает на
родителя.

</details>

<details>
<summary>Ответ</summary>

```text
2 drwxr-xr-x 5 root root 1024 Nov 18 07:14 /mnt/small
2 drwxr-xr-x 5 root root 1024 Nov 18 07:14 /mnt/small/.
2 drwxr-xr-x 5 root root 1024 Nov 18 07:14 /mnt/small/sub/..
```

Почему это работает: три пути — это три записи каталогов, указывающие на inode 2. Счётчик ссылок равен 5:
`small` в `/mnt`, `.` в самом каталоге и `..` в каждом из трёх подкаталогов: `lost+found`, `sub` и `demo`
(его создала `links`). Каталог `many` из «Попробуйте сами» добавлял ещё одну, пока существовал.

</details>

### Упражнение 2 — Это один и тот же файл? (средне)

Напишите `pub fn same_file(a: &Path, b: &Path) -> io::Result<bool>` в `src/lib.rs`. Два пути — один и тот же
файл, когда у них одинаковое устройство **и** одинаковый inode. Напишите тест, который создаёт в новом
временном каталоге файл, жёсткую ссылку, символическую ссылку и копию и проверяет, что первые две — тот же
файл, а копия — нет. Что должна дать символическая ссылка?

<details>
<summary>Подсказка</summary>

`MetadataExt` даёт `dev()` и `ino()`. Используйте `fs::metadata` (она проходит по символическим ссылкам).
Для тестового каталога возьмите `std::env::temp_dir()` плюс уникальное имя и удалите его в конце.

</details>

<details>
<summary>Ответ</summary>

```rust
/// True when both paths lead to the same inode on the same file system.
pub fn same_file(a: &Path, b: &Path) -> io::Result<bool> {
    use std::os::unix::fs::MetadataExt;
    let (ma, mb) = (fs::metadata(a)?, fs::metadata(b)?);
    Ok(ma.dev() == mb.dev() && ma.ino() == mb.ino())
}

#[cfg(test)]
mod tests {
    use super::*;
    use std::os::unix::fs::symlink;
    use std::path::PathBuf;

    /// A fresh, empty directory for one test.
    fn test_dir(name: &str) -> PathBuf {
        let dir = std::env::temp_dir().join(format!("p13-fs-{name}-{}", std::process::id()));
        let _ = fs::remove_dir_all(&dir);
        fs::create_dir_all(&dir).unwrap();
        dir
    }

    #[test]
    fn hard_link_and_symlink_are_the_same_file_but_a_copy_is_not() -> io::Result<()> {
        let dir = test_dir("same");
        let a = dir.join("a");
        fs::write(&a, "data")?;
        fs::hard_link(&a, dir.join("hard"))?;
        symlink(&a, dir.join("soft"))?;
        fs::copy(&a, dir.join("copy"))?;

        assert!(same_file(&a, &dir.join("hard"))?);
        assert!(same_file(&a, &dir.join("soft"))?); // metadata() follows the symlink
        assert!(!same_file(&a, &dir.join("copy"))?);
        fs::remove_dir_all(&dir)
    }
}
```

Почему это работает: жёсткая ссылка — по определению тот же inode. `fs::metadata` проходит по символической
ссылке к цели и поэтому тоже сообщает тот же inode. У копии свой inode с таким же содержимым. Сравнивать
только номера inode было бы неправильно: в двух разных файловых системах может быть по inode 13.

</details>

### Упражнение 3 — Протестируйте атомарную запись (сложно)

Напишите тесты для `atomic_write`: (а) после замены существующего файла в нём новое содержимое, а в каталоге
нет других файлов (не остался `.tmp`); (б) запись в несуществующий каталог возвращает `Err`. Затем подумайте:
какую часть гарантии ваши тесты проверить *не могут*?

<details>
<summary>Подсказка</summary>

Используйте `test_dir` из упражнения 2. Соберите имена через `fs::read_dir(&dir)?.map(|e| e.unwrap().file_name())`.

</details>

<details>
<summary>Ответ</summary>

Добавьте в модуль `tests`:

```rust
#[test]
fn replaces_content_and_leaves_no_temp_file() -> io::Result<()> {
    let dir = test_dir("atomic");
    let path = dir.join("config.toml");
    fs::write(&path, "old = true\n")?;

    atomic_write(&path, b"new = true\n")?;

    assert_eq!(fs::read_to_string(&path)?, "new = true\n");
    let names: Vec<_> = fs::read_dir(&dir)?
        .map(|e| e.unwrap().file_name())
        .collect();
    assert_eq!(names, vec!["config.toml"]);
    fs::remove_dir_all(&dir)
}

#[test]
fn missing_directory_is_an_error() {
    let path = Path::new("/no/such/dir/file.txt");
    assert!(atomic_write(path, b"x").is_err());
}
```

`cargo test` (вместе с тестом из упражнения 2):

```text
test result: ok. 3 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

Почему это работает: тесты проверяют видимый контракт. Чего они проверить не могут, так это саму гарантию при
сбое: что отключение питания посередине оставит старый или новый файл. Для этого нужна машина, которую можно
выключить, или особые инструменты, имитирующие потерянные записи. Поэтому порядок четырёх шагов должен быть
правильным по рассуждению, а не только по тестам.

</details>

## Вопросы для самопроверки

1. Что содержит каталог?

<details>
<summary>Ответ</summary>

Список записей: имена и номера inode, на которые они указывают. Имена живут в каталоге; у самого inode имени
нет.

</details>

2. У файла две жёсткие ссылки. Вы удаляете одно имя. Что происходит с данными?

<details>
<summary>Ответ</summary>

Ничего: счётчик ссылок падает с 2 до 1, и данные остаются доступны по другому имени. Они освобождаются
только тогда, когда счётчик равен 0 и ни один процесс не держит файл открытым.

</details>

3. Вы удалили огромный лог-файл, а `df -h` не показывает больше свободного места. В чём вероятная причина?

<details>
<summary>Ответ</summary>

Работающий процесс всё ещё держит файл открытым. Inode и его блоки освобождаются, только когда закрыт
последний дескриптор. `ls -l /proc/<pid>/fd` показывает его как `(deleted)`.

</details>

4. Почему в приёме атомарной замены временный файл должен лежать в том же каталоге, что и цель?

<details>
<summary>Ответ</summary>

`rename` атомарен только внутри одной файловой системы; между файловыми системами он завершается ошибкой
«Invalid cross-device link». Тот же каталог гарантированно находится в той же файловой системе.

</details>

5. Чем `Ok` от `write` отличается от `Ok` от `sync_all`?

<details>
<summary>Ответ</summary>

После `write` данные в страничном кэше, в RAM; отключение питания всё ещё может их уничтожить. После
`sync_all` данные и метаданные на носителе.

</details>

## Итоги

- VFS даёт каждой файловой системе (ext4, proc, tmpfs, overlayfs, cgroup2…) одни и те же системные вызовы; каждая смонтирована в свой каталог.
- Файл — это inode (тип, размер, владелец, права, времена, счётчик ссылок, блоки данных) плюс одна или несколько записей каталогов, которые дают ему имя; файл определяется парой (устройство, inode).
- Жёсткие ссылки — дополнительные имена одного inode; символические ссылки — маленькие файлы с путём, которые могут стать висячими.
- «Удаление» — это `unlink`: данные освобождаются, когда не осталось ни имён, ни открытых дескрипторов.
- `write` доходит только до страничного кэша; `sync_all` делает данные надёжно сохранёнными. Безопасно заменяйте файлы: временный файл → `sync_all` → `rename` → синхронизация каталога.
- Файловая система может закончиться по inode раньше, чем по данным: проверяйте `df -i`.

## Источники и что почитать

- [inode(7) — Linux manual page](https://man7.org/linux/man-pages/man7/inode.7.html) — каждое поле inode и как его показывает `stat`.
- [rename(2) — Linux manual page](https://man7.org/linux/man-pages/man2/rename.2.html) — гарантия атомарной замены и ошибка `EXDEV`.
- [fsync(2) — Linux manual page](https://man7.org/linux/man-pages/man2/fsync.2.html) — что обещает `fsync`, включая замечание о синхронизации каталога.
- [std::os::unix::fs::MetadataExt](https://doc.rust-lang.org/std/os/unix/fs/trait.MetadataExt.html) — номер inode, счётчик ссылок, режим и другое из Rust.
- [The Linux kernel documentation — Overview of the Linux Virtual File System](https://docs.kernel.org/filesystems/vfs.html) — как устроена VFS (для продвинутых).
- [Operating Systems: Three Easy Pieces](https://pages.cs.wisc.edu/~remzi/OSTEP/) — главы «Files and Directories», «File System Implementation» и «Crash Consistency: FSCK and Journaling».

---

[🏠 Начало](../START-HERE.md) · [← 13.11 Файлы, отображённые в память](11-memory-mapped-files.md) · [📋 Этап 13](README.md) · [13.13 Неблокирующий ввод-вывод и epoll →](13-non-blocking-io-and-epoll.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
