<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 17.8 Компакция и фильтры Блума](08-compaction-and-bloom-filters.md) · [📋 Этап 17](README.md) · [17.10 Транзакции и ACID →](10-transactions-and-acid.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/17-storage-and-databases/09-page-cache-and-buffer-pool.md) <!-- nav -->

# Урок 17.9 — Страничный кэш и пул буферов

> **Этап 17 — Как устроены хранилища и базы данных** · Урок 9 из 15 · ⏱ около 3 часов (2 занятия)

## Что вы узнаете

- Что делает страничный кэш ядра для каждой программы и как увидеть, как он заполняется и опустошается.
- Зачем база данных держит **свой** кэш страниц, пул буферов, поверх страничного кэша.
- Как пул буферов устроен внутри: фреймы, таблица страниц, «грязные» страницы и вытеснение по LRU.
- Почему LRU не справляется с большими сканированиями и что такое алгоритм «часы».
- Как наблюдать за пулом буферов PostgreSQL через `EXPLAIN (ANALYZE, BUFFERS)` и `pg_buffercache`.

## Перед началом

- Вы знаете страничный кэш из [Урока 13.11 — Файлы, отображённые в память](../13-operating-systems/11-memory-mapped-files.md)
  и разницу между `write` и `sync_all` из [Урока 13.12 — Файловые системы изнутри](../13-operating-systems/12-file-systems-inside.md).
- Вы знаете кэши, попадания и промахи из [Урока 12.9 — Иерархия памяти и кэши](../12-computer-systems/09-memory-hierarchy-and-caches.md).
- Вы знаете WAL (write-ahead log, журнал упреждающей записи) из [Урока 17.4 — Журнал упреждающей записи (WAL)](04-write-ahead-log.md) и PostgreSQL в Docker
  из [Урока 16.5 — Основы SQL и PostgreSQL](../16-backend-services/05-sql-and-postgresql-basics.md).
- Docker Desktop запущен.

## Новые слова

| Слово | Значение |
|---|---|
| **страничный кэш** (page cache) | Оперативная память, в которой ядро держит недавно использованные данные файлов. Через неё проходит каждый `read` и `write`. |
| **пул буферов** (buffer pool) | Собственный кэш страниц базы данных в её собственной памяти. База сама решает, какие страницы остаются и когда изменённые страницы записываются. |
| **фрейм** (frame) | Одна ячейка пула буферов, ровно на одну страницу. |
| **таблица страниц** (page table) пула буферов | Отображение «номер страницы → фрейм, в котором она лежит». Это не таблица страниц виртуальной памяти из Урока 13.10. |
| **грязная страница** (dirty page) | Страница, изменённая в памяти и ещё не записанная обратно в файл. |
| **вытеснение** (eviction) | Удаление страницы из кэша, чтобы освободить место для другой. Удаляемая страница — *жертва* (victim). |
| **LRU** (Least Recently Used, дольше всех не использовавшийся) | Правило вытеснения: удалить страницу, которую дольше всех не использовали. |
| **последовательное затопление** (sequential flooding) | Большое сканирование, которое выталкивает из LRU-кэша все полезные страницы, так что кэш не даёт ни одного попадания. |
| **алгоритм «часы»** (clock algorithm) | Дешёвое приближение LRU: у каждого фрейма есть бит «использовался», а «стрелка» обходит фреймы по кругу и даёт каждой использованной странице второй шанс. |
| **контрольная точка** (checkpoint) | Момент, когда база данных записывает все грязные страницы в файлы данных, чтобы восстановление после сбоя могло начинаться с этого места. |
| **двойное кэширование** (double caching) | Одна и та же страница лежит в RAM (Random Access Memory, оперативная память) дважды: в пуле буферов базы и в страничном кэше ядра. |
| **`O_DIRECT`** | Флаг Linux для `open`, который обходит страничный кэш: данные идут напрямую между диском и вашим буфером. |
| **доля попаданий** (hit ratio) | Попадания, делённые на все обращения. Доля 0,99 значит, что 99 страниц из 100 нашлись в памяти. |

## 1. Два кэша между базой данных и диском

Когда программа вызывает `read`, ядро сначала смотрит в **страничный кэш**. Если страница файла там, ядро копирует её
из RAM за микросекунды. Если нет, оно читает страницу с диска и оставляет копию на следующий раз. `write` меняет
страницу в страничном кэше и возвращает управление; ядро запишет грязную страницу на диск позже или когда вы вызовете
`sync_all` (Урок 13.12).

База данных обычно добавляет второй кэш **внутри собственного процесса** — **пул буферов**:

```text
   +----------------------------------+
   |   database process               |
   |   +--------------------------+   |
   |   |  buffer pool (frames)    |   |  <- the database decides what stays
   |   +--------------------------+   |
   +----------------|-----------------+
                    | read / write of whole pages
   +----------------v-----------------+
   |   kernel: page cache             |  <- the kernel decides what stays
   +----------------|-----------------+
                    |
                  disk
```

Тогда страница может лежать в RAM **дважды**, во фрейме и в страничном кэше: это **двойное кэширование**. Сначала
посмотрим на сам страничный кэш.

## 2. Наблюдаем за страничным кэшем

Утилита Linux `fincore` («file in core»; *core* — старое слово для RAM) показывает, какая часть файла лежит в
страничном кэше. Запустите контейнер с томом Docker, потому что том — обычная файловая система ext4 (на собственной
overlay-файловой системе контейнера `fincore` ничего не показывает):

```bash
docker run --rm -it -v p17-pagecache:/data ubuntu:26.04
apt-get update && apt-get install -y util-linux-extra
cd /data
```

`util-linux-extra` — пакет Ubuntu, в котором лежит `fincore`. Внутри контейнера:

```bash
dd if=/dev/urandom of=data.bin bs=1M count=256 status=none
fincore data.bin
sync data.bin
dd if=data.bin iflag=nocache count=0 status=none
fincore data.bin
dd if=data.bin of=/dev/null bs=1M
fincore data.bin
dd if=data.bin of=/dev/null bs=1M
```

```text
  RES PAGES  SIZE FILE
 256M 65536  256M data.bin
RES PAGES  SIZE FILE
 0B     0  256M data.bin
256+0 records in
256+0 records out
268435456 bytes (268 MB, 256 MiB) copied, 0.0969864 s, 2.8 GB/s
  RES PAGES  SIZE FILE
 256M 65536  256M data.bin
256+0 records in
256+0 records out
268435456 bytes (268 MB, 256 MiB) copied, 0.0184123 s, 15 GB/s
```

**Что происходит:**

1. `dd` записывает 256 МиБ (мебибайтов по 1 048 576 байтов) случайных байтов. Все 65 536 страниц (по 4 КиБ) **резидентны** (`RES`): только что
   записанные данные лежат в страничном кэше.
2. `sync data.bin` записывает грязные страницы этого файла на диск. Ядро может выбросить чистые страницы, но не грязные.
3. `dd ... iflag=nocache count=0` ничего не копирует, но просит ядро (системным вызовом `posix_fadvise`) выбросить
   страницы этого файла. Теперь резидентных страниц 0.
4. Первое чтение вынуждено идти на диск: 2,8 ГБ/с. Оно снова заполняет кэш.
5. Второе чтение идёт из RAM: 15 ГБ/с, примерно в пять раз быстрее. У вас числа будут другими; на медленном облачном
   диске разница бывает в сто раз.

Выйдите командой `exit` и удалите том: `docker volume rm p17-pagecache`.

> 💡 **Совет:** `free -h` в Linux показывает столбец `buff/cache`. Эта память не «занята»: ядро отдаёт её, как только
> она нужна программам. Сервер с малым «свободным» объёмом и большим страничным кэшем здоров.

## 3. Зачем базе данных свой кэш

Ядро ничего не знает о смысле страниц, а база данных должна управлять тремя вещами:

1. **Порядком записей.** Правило WAL (Урок 17.4): изменённая страница данных может попасть на диск только *после* того,
   как надёжно записаны записи WAL, описывающие изменение. Ядро пишет грязные страницы когда захочет. Пул буферов держит
   изменённые страницы в своей памяти и записывает их, только когда правило это разрешает.
2. **Тем, что хранить.** База знает, что корневая страница B-дерева (Урок 17.6) нужна каждому поиску, а страницы
   одноразового полного сканирования больше не понадобятся. Ядро видит только «страницу 7 какого-то файла».
3. **Моментом записи.** На **контрольной точке** база записывает все грязные страницы и отмечает в WAL: «всё до этого
   места уже в файлах данных». После сбоя восстановление проигрывает WAL только от последней контрольной точки.

Базы данных по-разному обходятся с двойным кэшированием:

| База данных | Пул буферов | Страничный кэш |
|---|---|---|
| PostgreSQL | `shared_buffers`, по умолчанию 128 МБ; часто ставят около четверти RAM | используется как второй уровень |
| MySQL InnoDB | получает бо́льшую часть RAM | обычно обходится через `O_DIRECT` |
| SQLite | небольшой кэш на каждое соединение | используется очень активно |

У `O_DIRECT` строгие правила (адрес буфера, смещение и длина выровнены по размеру блока диска), поэтому им пользуются
только программы со своим кэшем. В macOS похожий инструмент — `fcntl(fd, F_NOCACHE, 1)`.

## 4. Пул буферов на Rust

Соберём маленький пул буферов, чтобы слова превратились в код:

```bash
cd ~/projects/rust
cargo new --lib code/p17-bufpool
cd code/p17-bufpool
cargo add tempfile@3 --dev
```

```text
 file pages.db:  | page 0 | page 1 | page 2 | page 3 | page 4 | ...   (4096 bytes each)

 page_table: {0 -> 2, 3 -> 0, 4 -> 1}

 frames:   [0] page 3, dirty    [1] page 4, clean    [2] page 0, clean
            last_used = 7        last_used = 5        last_used = 9
```

Страница `id` начинается с байта `id * 4096` файла. Чтобы читать или писать с заданной позиции (**смещения**, offset),
мы используем `read_at` и `write_all_at` из трейта `std::os::unix::fs::FileExt`. Они работают как системные вызовы
`pread` и `pwrite`: смещение передаётся параметром, а курсор файла не двигается. Замените `src/lib.rs` на:

```rust
use std::collections::HashMap;
use std::fs::File;
use std::io;
use std::os::unix::fs::FileExt;

pub const PAGE_SIZE: usize = 4096;

#[derive(Clone)]
struct Frame {
    page_id: Option<u64>, // which page is in this frame (None = empty)
    data: Box<[u8; PAGE_SIZE]>,
    dirty: bool,    // changed in memory, not yet written to the file
    last_used: u64, // "time" of the last access, for LRU
}

#[derive(Debug, Default, Clone, Copy)]
pub struct Stats {
    pub hits: u64,
    pub misses: u64,
    pub disk_writes: u64,
}

pub struct BufferPool {
    file: File,
    frames: Vec<Frame>,
    page_table: HashMap<u64, usize>, // page id -> frame index
    tick: u64,                       // counts accesses, our "time"
    pub stats: Stats,
}

impl BufferPool {
    pub fn new(file: File, capacity: usize) -> BufferPool {
        assert!(capacity > 0, "a pool needs at least one frame");
        let empty = Frame {
            page_id: None,
            data: Box::new([0; PAGE_SIZE]),
            dirty: false,
            last_used: 0,
        };
        BufferPool {
            file,
            frames: vec![empty; capacity],
            page_table: HashMap::new(),
            tick: 0,
            stats: Stats::default(),
        }
    }

    /// Gives `f` read access to the page.
    pub fn read<R>(&mut self, id: u64, f: impl FnOnce(&[u8]) -> R) -> io::Result<R> {
        let i = self.frame_for(id)?;
        Ok(f(&self.frames[i].data[..]))
    }

    /// Gives `f` write access to the page and marks the page dirty.
    pub fn write<R>(&mut self, id: u64, f: impl FnOnce(&mut [u8]) -> R) -> io::Result<R> {
        let i = self.frame_for(id)?;
        self.frames[i].dirty = true;
        Ok(f(&mut self.frames[i].data[..]))
    }

    /// Writes every dirty page to the file and makes it durable.
    pub fn flush_all(&mut self) -> io::Result<()> {
        for i in 0..self.frames.len() {
            self.write_back(i)?;
        }
        self.file.sync_data()
    }

    /// The pages that are in memory now, sorted.
    pub fn cached_pages(&self) -> Vec<u64> {
        let mut ids: Vec<u64> = self.page_table.keys().copied().collect();
        ids.sort();
        ids
    }

    /// Finds the frame that holds page `id`, loading the page if needed.
    fn frame_for(&mut self, id: u64) -> io::Result<usize> {
        self.tick += 1;
        if let Some(&i) = self.page_table.get(&id) {
            self.stats.hits += 1;
            self.frames[i].last_used = self.tick;
            return Ok(i);
        }
        self.stats.misses += 1;
        let i = self.choose_victim();
        self.write_back(i)?;
        if let Some(old) = self.frames[i].page_id.take() {
            self.page_table.remove(&old);
        }
        let frame = &mut self.frames[i];
        frame.data.fill(0); // bytes past the end of the file stay zero
        let n = self.file.read_at(&mut frame.data[..], offset(id))?;
        assert!(n == 0 || n == PAGE_SIZE, "pages are always written whole");
        frame.page_id = Some(id);
        frame.last_used = self.tick;
        self.page_table.insert(id, i);
        Ok(i)
    }

    /// An empty frame if there is one, otherwise the least recently used one.
    fn choose_victim(&self) -> usize {
        let key = |i: &usize| (self.frames[*i].page_id.is_some(), self.frames[*i].last_used);
        (0..self.frames.len()).min_by_key(key).unwrap()
    }

    /// If the frame is dirty, writes its page to the file.
    fn write_back(&mut self, i: usize) -> io::Result<()> {
        let frame = &mut self.frames[i];
        if let (true, Some(id)) = (frame.dirty, frame.page_id) {
            self.file.write_all_at(&frame.data[..], offset(id))?;
            frame.dirty = false;
            self.stats.disk_writes += 1;
        }
        Ok(())
    }
}

/// Page `id` starts at this byte of the file.
fn offset(id: u64) -> u64 {
    id * PAGE_SIZE as u64
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn least_recently_used_page_is_evicted() {
        let file = tempfile::tempfile().unwrap();
        let mut pool = BufferPool::new(file, 2);
        for id in [0, 1, 0, 2] {
            pool.read(id, |_| ()).unwrap();
        }
        assert_eq!(pool.stats.hits, 1);
        assert_eq!(pool.cached_pages(), vec![0, 2]); // page 1 was used longest ago
    }
}
```

**Что происходит:**

1. `Frame` хранит одну страницу (в `Box`, в куче) и три пометки: какая это страница, грязная ли она и когда её
   использовали последний раз. `vec![empty; capacity]` клонирует пустой фрейм `capacity` раз.
2. `read` и `write` принимают **замыкание**, которое получает байты страницы. Вызывающий код никогда не хранит ссылку
   внутрь пула, поэтому проверка заимствований довольна, а пул может переиспользовать фрейм при следующем вызове.
   `write` ставит `dirty`.
3. `frame_for` — сердце пула. При **попадании** таблица страниц знает фрейм, и мы только обновляем `last_used`. При
   **промахе** выбираем фрейм-жертву, записываем его, если он грязный, забываем его старую страницу и читаем в него
   новую. Страница, которой ещё нет в файле, читается как нули (`read_at` возвращает 0 в конце файла).
4. `choose_victim` берёт фрейм с наименьшим ключом `(занят ли он?, last_used)`. `false` меньше `true`, поэтому пустой
   фрейм выигрывает; среди занятых выигрывает самый старый `last_used`. Это и есть **LRU**. Перебор всех фреймов стоит
   O(n); настоящие пулы используют списки или алгоритм «часы» (раздел 6), чтобы этого избежать.
5. `write_back` вызывается **до** того, как фрейм используется снова. Забудьте его, и изменения тихо пропадут (см.
   «Частые ошибки»).
6. `flush_all` записывает все грязные страницы, а затем вызывает `sync_data`, чтобы страницы пережили сбой. В Linux это
   `fdatasync`; в macOS стандартная библиотека и для `sync_data`, и для `sync_all` вызывает `fcntl(fd, F_FULLFSYNC)`,
   который ещё и опустошает собственный кэш записи накопителя.

```bash
cargo test --lib
```

```text
running 1 test
test tests::least_recently_used_page_is_evicted ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

> 🧪 **Тестирование:** `tempfile::tempfile()` даёт настоящий файл, который удаляется автоматически, поэтому каждый тест
> начинает с чистого личного файла. Тест проверяет поведение, которое можно назвать словами («уходит страница, которую
> дольше всех не использовали»), а не приватные поля.

## 5. Попадания, промахи и вытеснение

На этом этапе маленькие программы живут в `src/bin/`. Создайте `src/bin/lru.rs`:

```rust
use p17_bufpool::BufferPool;
use std::fs::OpenOptions;

fn main() -> std::io::Result<()> {
    let file = OpenOptions::new()
        .read(true)
        .write(true)
        .create(true)
        .truncate(true)
        .open("pages.db")?;
    let mut pool = BufferPool::new(file, 3); // room for 3 pages

    // store the number id * 100 at the start of pages 0 to 5
    for id in 0..6u64 {
        let bytes = (id * 100).to_le_bytes();
        pool.write(id, |page| page[..8].copy_from_slice(&bytes))?;
    }
    pool.flush_all()?;
    println!("after writing: {:?}", pool.stats);

    pool.stats = Default::default();
    for id in [0, 1, 2, 0, 3, 0, 4] {
        let hits_before = pool.stats.hits;
        let value = pool.read(id, first_u64)?;
        let hit = pool.stats.hits > hits_before;
        let kind = if hit { "hit " } else { "miss" };
        let cached = pool.cached_pages();
        println!("page {id}: {kind} value={value:<3} in memory: {cached:?}");
    }
    println!("{:?}", pool.stats);
    Ok(())
}

/// Reads the first 8 bytes of a page as a little-endian number.
fn first_u64(page: &[u8]) -> u64 {
    u64::from_le_bytes(page[..8].try_into().unwrap())
}
```

```bash
cargo run --bin lru
```

```text
after writing: Stats { hits: 0, misses: 6, disk_writes: 6 }
page 0: miss value=0   in memory: [0, 4, 5]
page 1: miss value=100 in memory: [0, 1, 5]
page 2: miss value=200 in memory: [0, 1, 2]
page 0: hit  value=0   in memory: [0, 1, 2]
page 3: miss value=300 in memory: [0, 2, 3]
page 0: hit  value=0   in memory: [0, 2, 3]
page 4: miss value=400 in memory: [0, 3, 4]
Stats { hits: 2, misses: 5, disk_writes: 0 }
```

**Что происходит:**

1. Запись 6 страниц через 3 фрейма: 6 промахов (пул сначала читает каждую страницу: он не может знать, что она новая).
   Страницы 0, 1 и 2 были вытеснены грязными (3 записи), а `flush_all` записал страницы 3, 4 и 5: всего 6 записей.
2. Чтение страницы 0 вытесняет страницу 3, которую дольше всех не использовали; страница 1 вытесняет 4; страница 2 вытесняет 5.
3. Снова страница 0: **попадание**. Когда приходит страница 3, самая старая — страница 1 (страницу 0 только что
   использовали), поэтому уходит страница 1.
4. Значения правильно возвращаются из файла, и `disk_writes: 0`: чтение никогда не делает страницу грязной, поэтому
   вытеснение чистой страницы ничего не стоит.

> ⚡ **Производительность:** доля попаданий здесь 2/7. Настоящие базы стремятся к 0,99 и выше, потому что один промах
> может стоить столько же, сколько тысячи попаданий. Когда база данных тормозит, доля попаданий пула буферов — одно из
> первых чисел, которые стоит проверить.

## 6. Когда LRU не справляется: большое сканирование

Создайте `src/bin/scan.rs`: пул из 9 фреймов снова и снова читает одни и те же 10 страниц.

```rust
use p17_bufpool::BufferPool;
use std::fs::OpenOptions;

fn main() -> std::io::Result<()> {
    let file = OpenOptions::new()
        .read(true)
        .write(true)
        .create(true)
        .truncate(true)
        .open("scan.db")?;
    let mut pool = BufferPool::new(file, 9); // 9 frames, but we scan 10 pages
    for round in 1..=3 {
        pool.stats = Default::default();
        for id in 0..10 {
            pool.read(id, |_| ())?;
        }
        let s = pool.stats;
        println!("scan {round}: {} hits, {} misses", s.hits, s.misses);
    }
    Ok(())
}
```

```bash
cargo run --bin scan
```

```text
scan 1: 0 hits, 10 misses
scan 2: 0 hits, 10 misses
scan 3: 0 hits, 10 misses
```

После страниц 0–8 пул полон, и страница 9 вытесняет страницу 0. Следующий круг начинается со страницы 0: промах, и
она вытесняет страницу 1 — ту самую, которая нужна следующей. Каждое обращение выбрасывает страницу, которая
понадобится раньше всех: девять фреймов и **ноль** попаданий. Это **последовательное затопление**. В настоящей базе
один большой отчётный запрос по огромной таблице вытолкнул бы маленькие «горячие» страницы, нужные всем остальным
запросам.

Базы данных борются с этим двумя способами:

- **Кольца для сканирования.** PostgreSQL даёт большому последовательному сканированию маленькое личное **кольцо**
  фреймов (256 КиБ), которое оно переиспользует, так что сканирование не может затопить основной пул.
- **Алгоритм «часы».** Точный LRU требует обновления при каждом обращении. PostgreSQL использует вариант «часов»: у
  каждого фрейма есть счётчик использования; «стрелка» обходит фреймы по кругу, как стрелка часов, уменьшает каждый
  счётчик, мимо которого проходит, и вытесняет первый фрейм со счётчиком ноль. Часто используемые страницы переживают
  несколько кругов. Упражнение 3 строит простейшую версию с одним битом на фрейм.

## 7. Пул буферов PostgreSQL снаружи

Запустите одноразовый PostgreSQL 18 на остаток этапа (без тома):

```bash
docker run -d --name p17-postgres -e POSTGRES_PASSWORD=dev-password \
  -p 127.0.0.1:5432:5432 postgres:18
docker exec -it p17-postgres psql -U postgres
```

Подождите несколько секунд после `docker run`: серверу нужно мгновение, прежде чем он начнёт принимать соединения.
Если порт 5432 занят, потому что ещё работает `p16-postgres`, сначала остановите его: `docker stop p16-postgres`. В
приглашении `postgres=#`:

```sql
SHOW shared_buffers;
CREATE TABLE readings (id bigint PRIMARY KEY, sensor int, value double precision);
INSERT INTO readings SELECT i, i % 100, random() FROM generate_series(1, 200000) AS i;
SELECT pg_relation_size('readings') / 8192 AS pages;
VACUUM readings;
```

```text
 shared_buffers
----------------
 128MB
(1 row)

CREATE TABLE
INSERT 0 200000
 pages
-------
  1274
(1 row)

VACUUM
```

Страницы PostgreSQL — 8 КиБ (8192 байта); в таблице их 1274. (`VACUUM` приводит новую таблицу в порядок; Урок 17.12
объясняет, что он делает.) Выйдите через `\q`, перезапустите сервер, чтобы его пул буферов опустел, подключитесь снова
и прочитайте всю таблицу дважды:

```bash
docker restart p17-postgres
docker exec -it p17-postgres psql -U postgres
```

```sql
SET max_parallel_workers_per_gather = 0;
EXPLAIN (ANALYZE, BUFFERS, COSTS OFF) SELECT count(*) FROM readings;
EXPLAIN (ANALYZE, BUFFERS, COSTS OFF) SELECT count(*) FROM readings;
```

Важные строки двух планов:

```text
   ->  Seq Scan on readings (actual time=0.073..6.171 rows=200000.00 loops=1)
         Buffers: shared read=1274
 Execution Time: 11.252 ms

   ->  Seq Scan on readings (actual time=0.005..5.440 rows=200000.00 loops=1)
         Buffers: shared hit=1274
 Execution Time: 10.512 ms
```

**Что происходит:**

1. `SET max_parallel_workers_per_gather = 0` просит обойтись одним процессом, так план короче.
2. `EXPLAIN (ANALYZE, BUFFERS)` выполняет запрос и считает страницы: `shared read` значит «нет в пуле буферов, спросили
   ядро», `shared hit` — «нашлось в пуле буферов». Остальное в плане разбирает Урок 17.13.
3. Первый запуск: 1274 промаха. Второй: 1274 попадания.
4. Первый запуск почти не медленнее. Перезапуск контейнера опустошил пул буферов PostgreSQL, но **не** страничный кэш
   ядра, поэтому каждое «чтение» ядро обслужило из RAM: двойное кэширование сработало нам на пользу.

Теперь посмотрим на грязные страницы и контрольную точку. `pg_buffercache` — расширение, которое поставляется вместе с
PostgreSQL и показывает фреймы пула буферов:

```sql
CREATE EXTENSION pg_buffercache;
UPDATE readings SET value = 0 WHERE id <= 1000;
SELECT count(*) FILTER (WHERE isdirty) AS dirty
FROM pg_buffercache WHERE relfilenode = pg_relation_filenode('readings');
CHECKPOINT;
SELECT count(*) FILTER (WHERE isdirty) AS dirty
FROM pg_buffercache WHERE relfilenode = pg_relation_filenode('readings');
```

```text
CREATE EXTENSION
UPDATE 1000
 dirty
-------
    19
(1 row)

CHECKPOINT
 dirty
-------
     0
(1 row)
```

`UPDATE` сделал грязными 19 страниц таблицы и вернул управление задолго до того, как они были записаны: изменение уже
было в безопасности, потому что его записи WAL были надёжно сохранены. `CHECKPOINT` записал грязные страницы в файлы
данных — как `flush_all` в вашем пуле. Оставьте контейнер для следующих уроков; выйдите из `psql` через `\q`.

> 🔒 **Безопасность:** пул буферов держит в RAM настоящие строки — с персональными данными, иногда с хешами паролей.
> Дамп памяти процесса базы данных (для отладки или core-файл после падения) содержит их. Защищайте такие дампы так же,
> как саму базу.

## Частые ошибки

> ⚠️ **Частая ошибка:** забыть записать грязную страницу перед тем, как фрейм используется снова. Закомментируйте
> `self.write_back(i)?;` в `frame_for` и запустите тест из Упражнения 2:
>
> ```text
> thread 'tests::dirty_page_survives_eviction' (6049315) panicked at src/lib.rs:134:9:
> assertion `left == right` failed
>   left: 0
>  right: 1
> ```
>
> Ни ошибки, ни падения: изменение просто теряется. Такую ошибку ловит только тест.

> ⚠️ **Частая ошибка:** измерить запрос или чтение файла один раз и сделать выводы. Говорите, что вы измеряли —
> «холодный» или «тёплый» запуск: страничный кэш и пул буферов могут сделать второй запуск во много раз быстрее.

> ⚠️ **Частая ошибка:** думать, что вытеснение страницы её «сохраняет». Вытеснение лишь переносит её в страничный кэш.
> Надёжность дают `sync_data` / `sync_all`, а в базе данных — WAL.

## Попробуйте сами

1. Измените размер пула в `lru.rs` с 3 на 2, предскажите попадания, запустите. Затем попробуйте 4. Должно получиться
   `hits: 1, misses: 6` для 2 фреймов и `hits: 2, misses: 5` для 4: лишний фрейм помогает не всегда.
2. В `scan.rs` замените 9 фреймов на 10. Круги 2 и 3 теперь печатают `10 hits, 0 misses`: когда всё помещается, LRU
   идеален; на один фрейм меньше — и он бесполезен.
3. В `psql` дважды выполните `EXPLAIN (ANALYZE, BUFFERS, COSTS OFF) SELECT * FROM readings WHERE id = 5;`. Поиск по
   индексу затрагивает несколько страниц (во втором запуске у нас было `Buffers: shared hit=6`), а не 1274.

## Упражнения

### Упражнение 1 — Предскажите попадания (легко)

Пул из 2 фреймов использует LRU. Страницы читаются в порядке `1, 2, 1, 3, 2, 1`. Какие обращения — попадания и какие
страницы останутся в памяти в конце? Решите на бумаге, затем проверьте маленькой программой в `src/bin/`.

<details>
<summary>Подсказка</summary>

Запишите фреймы списком от «дольше всех не использовавшейся» к «использованной последней». Попадание переносит страницу в
конец; промах при полном списке удаляет первую страницу.

</details>

<details>
<summary>Ответ</summary>

```text
1: miss  [1]
2: miss  [1, 2]
1: hit   [2, 1]
3: miss  [1, 3]     (2 goes)
2: miss  [3, 2]     (1 goes)
1: miss  [2, 1]     (3 goes)
```

Одно попадание, пять промахов; остаются страницы 1 и 2. Копия `lru.rs` с `BufferPool::new(file, 2)` и циклом по
`[1, 2, 1, 3, 2, 1]` печатает `Stats { hits: 1, misses: 5, disk_writes: 0 }` и `[1, 2]`.

Почему это работает: LRU всегда вытесняет страницу, чьё последнее использование самое давнее; попадание лишь обновляет
это время.

</details>

### Упражнение 2 — Тест: грязная страница переживает вытеснение (средне)

Напишите тест `dirty_page_survives_eviction`: пул из **одного** фрейма записывает байт 42 в страницу 0, затем читает
страницу 1, что вынуждает вытеснить страницу 0. Проверьте, что пул насчитал одну запись на диск, что байт 0 *файла*
равен 42 и что чтение страницы 0 через пул снова даёт 42.

<details>
<summary>Подсказка</summary>

`file.try_clone()` даёт второй дескриптор того же файла: один отдайте пулу, другой оставьте в тесте. Читайте файл через
`read_exact_at` из `FileExt` (он уже импортирован через `use super::*`).

</details>

<details>
<summary>Ответ</summary>

```rust
    #[test]
    fn dirty_page_survives_eviction() {
        let file = tempfile::tempfile().unwrap();
        let mut pool = BufferPool::new(file.try_clone().unwrap(), 1);
        pool.write(0, |page| page[0] = 42).unwrap();
        pool.read(1, |_| ()).unwrap(); // the only frame is needed: page 0 must go
        assert_eq!(pool.stats.disk_writes, 1);

        let mut byte = [0u8; 1];
        file.read_exact_at(&mut byte, 0).unwrap();
        assert_eq!(byte[0], 42, "page 0 was not written to the file");
        assert_eq!(pool.read(0, |page| page[0]).unwrap(), 42);
    }
```

Почему это работает: с одним фреймом каждый промах вытесняет. Тест читает файл в обход пула, поэтому доказывает, что
вытеснение действительно записало страницу. Без `write_back` он падает (см. «Частые ошибки»).

</details>

### Упражнение 3 — Алгоритм «часы» (сложно)

Замените LRU алгоритмом «часы». Добавьте во `Frame` поле `referenced: bool` (ставится в `true` при каждом попадании и
каждой загрузке), а в пул — поле `hand: usize`. Выбор жертвы: если есть пустой фрейм, взять его; иначе посмотреть на
фрейм под стрелкой и сдвинуть стрелку на шаг. Если у этого фрейма стоит бит, сбросить его (второй шанс) и продолжить;
если нет — это жертва. Тест: пул из 3 фреймов читает `0, 1, 2, 3`, затем страницу 1, затем страницу 4; страница 1
должна остаться, а страница 2 — уйти.

<details>
<summary>Подсказка</summary>

`choose_victim` теперь меняет пул (стрелку и биты), поэтому принимает `&mut self`. Цикл всегда заканчивается: не позже
чем через один полный круг все биты сброшены.

</details>

<details>
<summary>Ответ</summary>

```rust
    /// An empty frame if there is one; otherwise the clock algorithm picks one.
    fn choose_victim(&mut self) -> usize {
        if let Some(i) = self.frames.iter().position(|f| f.page_id.is_none()) {
            return i;
        }
        loop {
            let i = self.hand;
            self.hand = (self.hand + 1) % self.frames.len();
            if self.frames[i].referenced {
                self.frames[i].referenced = false; // second chance
            } else {
                return i;
            }
        }
    }
```

Добавьте `referenced: false` в пустой фрейм и `hand: 0` в `new`, и ставьте `referenced = true` рядом с обеими строками
`last_used = self.tick` во `frame_for`. Тест:

```rust
    #[test]
    fn clock_gives_a_second_chance() {
        let file = tempfile::tempfile().unwrap();
        let mut pool = BufferPool::new(file, 3);
        for id in [0, 1, 2, 3] {
            pool.read(id, |_| ()).unwrap();
        }
        assert_eq!(pool.cached_pages(), vec![1, 2, 3]);
        pool.read(1, |_| ()).unwrap(); // page 1 is used again
        pool.read(4, |_| ()).unwrap(); // page 1 survives, page 2 goes
        assert_eq!(pool.cached_pages(), vec![1, 3, 4]);
    }
```

Почему это работает: когда приходит страница 3, все три бита стоят, поэтому стрелка сбрасывает их все, возвращается к
фрейму 0 и вытесняет страницу 0; теперь стрелка указывает на фрейм 1. Чтение страницы 1 снова ставит её бит, поэтому
для страницы 4 стрелка сбрасывает его, идёт дальше и вытесняет страницу 2. С этой версией `lru.rs` на шаге «page 3»
вытесняет страницу 0, которую LRU сохранил: «часы» лишь приближают LRU, зато стоят один бит на обращение и их гораздо
проще сделать быстрыми при многих потоках.

</details>

## Вопросы для самопроверки

1. Программа читает файл дважды, и второе чтение гораздо быстрее. Откуда во второй раз пришли данные?

<details>
<summary>Ответ</summary>

Из страничного кэша ядра в RAM: первое чтение загрузило страницы файла с диска, и ядро их сохранило.

</details>

2. Назовите две причины, по которым база данных не оставляет всё кэширование ядру.

<details>
<summary>Ответ</summary>

Любые две из: она должна управлять **порядком** записей (страница данных — только после надёжной записи её записей
WAL); она знает, какие страницы ценны (корни B-деревьев), а какие нет (страницы одноразового сканирования); она решает,
**когда** писать грязные страницы (контрольные точки), и это делает восстановление коротким.

</details>

3. Что такое грязная страница и что должно произойти, прежде чем её фрейм сможет принять другую страницу?

<details>
<summary>Ответ</summary>

Страница, изменённая в памяти, но ещё не записанная в файл. Сначала её нужно записать обратно (в настоящей базе — после
надёжной записи её записей WAL); иначе изменение потеряется.

</details>

4. Почему 10 страниц, сканируемых через 9 фреймов, не дали с LRU ни одного попадания?

<details>
<summary>Ответ</summary>

Каждый промах вытеснял страницу, которую дольше всех не использовали, а при повторяющемся сканировании это всегда страница,
нужная следующей: последовательное затопление.

</details>

5. Чем в `EXPLAIN (ANALYZE, BUFFERS)` отличаются `shared hit` и `shared read`? Значит ли `read`, что использовался диск?

<details>
<summary>Ответ</summary>

`hit`: страница была в пуле буферов PostgreSQL. `read`: её не было, и PostgreSQL спросил ядро. Не обязательно диск:
страница могла лежать в страничном кэше ядра.

</details>

## Итоги

- Каждое обращение к файлу проходит через страничный кэш ядра; `fincore` его показывает, а «тёплое» чтение во много
  раз быстрее «холодного».
- База данных держит свой пул буферов, чтобы управлять порядком и моментом записей и выбирать, что хранить; у
  PostgreSQL это `shared_buffers`, по умолчанию 128 МБ.
- Пул буферов — это фреймы, таблица страниц и флаги «грязная»; промах выбирает жертву, записывает её, если она грязная,
  и загружает новую страницу.
- LRU вытесняет страницу, которую дольше всех не использовали, и не справляется с большими сканированиями; базы
  добавляют кольца для сканирования и алгоритм «часы».
- Вытеснение — не надёжность: для неё нужны `sync_data` / `sync_all` и WAL.
- `EXPLAIN (ANALYZE, BUFFERS)` показывает попадания и чтения; `pg_buffercache` — грязные страницы; `CHECKPOINT`
  записывает их.

## Источники и что почитать

- [PostgreSQL: Resource Consumption](https://www.postgresql.org/docs/current/runtime-config-resource.html) — `shared_buffers` и как выбрать его размер.
- [PostgreSQL: pg_buffercache](https://www.postgresql.org/docs/current/pgbuffercache.html) — расширение из раздела 7.
- [PostgreSQL: Using EXPLAIN](https://www.postgresql.org/docs/current/using-explain.html) — в том числе параметр `BUFFERS`.
- [PostgreSQL: WAL Configuration](https://www.postgresql.org/docs/current/wal-configuration.html) — контрольные точки и когда записываются грязные страницы.
- [fincore(1)](https://man7.org/linux/man-pages/man1/fincore.1.html) и [open(2)](https://man7.org/linux/man-pages/man2/open.2.html) — утилита из раздела 2 и `O_DIRECT` с правилами выравнивания.
- [Rust std: FileExt](https://doc.rust-lang.org/std/os/unix/fs/trait.FileExt.html) — `read_at` и `write_all_at`.
- [CMU 15-445/645 Database Systems](https://15445.courses.cs.cmu.edu/) — CMU — Carnegie Mellon University (Университет Карнеги — Меллона); бесплатный университетский курс; его лекции о пулах буферов идут гораздо глубже.

---

[🏠 Начало](../START-HERE.md) · [← 17.8 Компакция и фильтры Блума](08-compaction-and-bloom-filters.md) · [📋 Этап 17](README.md) · [17.10 Транзакции и ACID →](10-transactions-and-acid.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
