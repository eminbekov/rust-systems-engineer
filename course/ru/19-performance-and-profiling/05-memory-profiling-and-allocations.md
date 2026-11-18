<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 19.4 Профилирование в Linux с perf](04-profiling-on-linux-with-perf.md) · [📋 Этап 19](README.md) · [19.6 Размещение данных для скорости →](06-data-layout-for-speed.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/19-performance-and-profiling/05-memory-profiling-and-allocations.md) <!-- nav -->

# Урок 19.5 — Профилирование памяти и меньше выделений

> **Этап 19 — Производительность и профилирование** · Урок 5 из 11 · ⏱ около 105 минут

## Что вы узнаете

- Почему выделения памяти в куче стоят времени и чем отличается то, **сколько** выделений делает программа, от того, **сколько** памяти она держит на **пике**.
- Как измерить и то и другое профилировщиком кучи `dhat`, который включается вашей собственной фичей Cargo.
- Как написать тест, который падает, когда код выделяет больше, чем должен.
- Практические способы выделять меньше: переиспользовать буфер, искать по `&str`, заимствовать вместо клонирования и читать вход по частям.

## Перед началом

- Вы прошли [Урок 19.4 — Профилирование в Linux с perf](04-profiling-on-linux-with-perf.md). Продолжаем с `code/p19-wordfreq`.
- Вы знаете, как аллокатор выдаёт блоки кучи и что делает `#[global_allocator]` ([Урок 12.13 — Как работают аллокаторы памяти](../12-computer-systems/13-how-memory-allocators-work.md)).
- Вы знаете фичи крейтов из [Урока 05.8 — Cargo: зависимости и версии](../05-rust-intermediate/08-cargo-dependencies-and-versions.md). Здесь вы определите фичу в своём собственном пакете.

## Новые слова

| Слово | Значение |
|---|---|
| **выделение памяти** (allocation) | Запрос у аллокатора блока памяти в куче (под капотом — `malloc`). Каждый `String`, `Vec` или `Box` с содержимым владеет таким блоком. |
| **блок** (block) | Один кусок памяти, выданный одним выделением. |
| **частота выделений** (allocation rate, turnover) | Сколько всего блоков программа выделяет (и освобождает). Множество короткоживущих блоков стоит времени. |
| **пиковая память** (peak memory) | Наибольший объём памяти в куче, который программа держит в один момент. От него зависит, сколько RAM (Random Access Memory, оперативная память) нужно программе. |
| **профилировщик кучи** (heap profiler) | Инструмент, который записывает каждое выделение: где в коде оно произошло, какого размера был блок и сколько он жил. |
| **dhat** | Профилировщик кучи для Rust, опубликованный как крейт. Он заменяет глобальный аллокатор на считающий. Его имя и просмотрщик пришли от **DHAT** (Dynamic Heap Analysis Tool, инструмент динамического анализа кучи) — профилировщика кучи из набора инструментов Valgrind. |
| **t-gmax** | Название в dhat для момента глобального максимума: времени, когда куча была больше всего. |
| **t-end** | Название в dhat для конца программы. Блоки, живые в t-end, так и не были освобождены. |
| **точка программы** (program point, PP) | В отчёте dhat — одно место в коде (вместе со стеком вызовов), которое выделяет память. |
| **необязательная зависимость** (optional dependency) | Зависимость, которая компилируется, только когда её включает фича вашего пакета. |
| **переиспользование буфера** (buffer reuse) | Держать один `String` или `Vec` и очищать его для каждого нового элемента вместо того, чтобы каждый раз создавать новый. |

## 1. Два вопроса о памяти

В Уроке 19.3 почти половина времени на macOS уходила в `malloc`, `free` и рост строк. Выделение памяти не бесплатно: аллокатор должен найти блок нужного размера, поддерживать свою бухгалтерию в порядке, а потом забрать блок обратно (Урок 12.13). Одно выделение стоит десятки наносекунд; 25 миллионов выделений стоят секунду.

Есть два разных вопроса, и исправлять их нужно по-разному:

```text
How MANY blocks? (allocation rate)        How MUCH at once? (peak memory)
  25 million tiny Strings, each freed        one 130 MB String with the whole file
  right away                                 held until the end
  -> costs CPU time in malloc/free           -> costs RAM; can get the process killed
  -> fix: reuse, borrow, avoid copies        -> fix: process data in pieces
```

Семплирующий профилировщик показывает первую проблему как широкие прямоугольники `malloc` и `free`, но не может сказать, *какая строка* выделила память и сколько памяти держится на пике. **Профилировщик кучи** может.

## 2. Настраиваем dhat

`dhat` работает так: заменяет глобальный аллокатор на тот, который записывает каждое выделение. Это сильно замедляет программу, поэтому мы включаем его только по запросу, через фичу (feature) нашего пакета. Добавьте крейт как **необязательную** зависимость:

```bash
cd ~/projects/rust/code/p19-wordfreq
cargo add dhat@0.3 --optional
```

Затем убедитесь, что в `Cargo.toml` есть такой раздел `[features]` (Cargo мог добавить фичу с именем `dhat`; переименуйте её в `dhat-heap` — это имя используется в документации `dhat`):

```toml
[dependencies]
dhat = { version = "0.3", optional = true }

[features]
dhat-heap = ["dep:dhat"]
```

**Что происходит:**

1. `optional = true`: крейт `dhat` не компилируется в обычной сборке.
2. `[features]` определяет переключатели для *вашего* пакета. `dhat-heap = ["dep:dhat"]` означает: «когда включена фича `dhat-heap`, включить зависимость `dhat`».
3. `cargo build --features dhat-heap` включает её; обычный `cargo build` — нет.

Теперь два небольших добавления в `src/main.rs`. Вверху, после строк `use`:

```rust
// With `--features dhat-heap`, every allocation goes through dhat's counting allocator.
#[cfg(feature = "dhat-heap")]
#[global_allocator]
static ALLOC: dhat::Alloc = dhat::Alloc;
```

И первыми строками `main`:

```rust
    // Starts heap profiling; the report is written when `_profiler` is dropped at the end of main.
    #[cfg(feature = "dhat-heap")]
    let _profiler = dhat::Profiler::new_heap();
```

**Что происходит:**

1. `#[cfg(feature = "dhat-heap")]` компилирует следующий элемент, только когда фича включена. Без фичи программа точно такая же, как раньше.
2. `#[global_allocator]` делает `dhat::Alloc` аллокатором для каждого `Box`, `Vec` и `String` в программе (в Уроке 12.13 вы писали свой).
3. `Profiler::new_heap()` начинает запись. У переменной должно быть имя (`_profiler`, а не `_`), чтобы она жила до конца `main`; когда она удаляется, dhat печатает сводку и пишет `dhat-heap.json`.

dhat замедляет программу во много раз, поэтому используйте небольшой вход. Сгенерируйте миллион слов (5 МБ, мегабайт):

```bash
cargo run --release --bin gen -- small.txt 1000000
echo "small.txt" >> .gitignore
echo "dhat-heap.json" >> .gitignore
cargo run --profile profiling --features dhat-heap --bin p19-wordfreq -- small.txt
```

```text
22576 different words
   52463 ka
   52370 ru
   52280 mi
   52077 ing
   52068 lo
   51825 ne
   51661 pre
   51482 sta
     958 theing
     943 ruis
dhat: Total:     13,246,059 bytes in 1,022,599 blocks
dhat: At t-gmax: 7,001,579 bytes in 14,342 blocks
dhat: At t-end:  1,088 bytes in 2 blocks
dhat: The data has been saved to dhat-heap.json, and is viewable with dhat/dh_view.html
```

Программа работала около 12,6 секунды вместо 0,2 секунды без dhat. **Прочитайте три строки dhat:**

1. **Total:** за весь запуск было выделено 1 022 599 блоков, всего 13 МБ. Это примерно **одно выделение на слово**: `normalize` всё ещё возвращает новый `String` для каждого слова (после исправления из Урока 19.3; исходная версия делала 2 127 791 блок, около двух на слово).
2. **At t-gmax:** на пике программа держала 7 МБ в 14 342 блоках: 5,2 МБ текста файла, `HashMap` и слова в нём.
3. **At t-end:** в самом конце были живы 1 088 байт в 2 блоках: небольшие буферы стандартной библиотеки (например, тот, что стоит за `println!`). Если бы здесь было число, растущее вместе со входом, это означало бы утечку.

Мы используем профиль `profiling` из Урока 19.3, потому что dhat нужна отладочная информация, чтобы показать в отчёте имена файлов и номера строк.

## 3. Заглядываем внутрь: DHAT viewer

Подробности лежат в `dhat-heap.json`. Откройте в браузере DHAT viewer по адресу `https://nnethercote.github.io/dh_view/dh_view.html` и загрузите файл кнопкой *Load…*. Страница читает файл внутри вашего браузера и никуда его не загружает.

Что вы видите (словами):

- Наверху — те же итоги, что в терминале, плюс время t-gmax и t-end в микросекундах.
- Меню **метрики сортировки** (sort metric). Метрика по умолчанию, «Total (blocks)», находит места, которые выделяют память *чаще всего*; «Total (bytes)», «At t-gmax (bytes)» и «At t-end (bytes)» сортируют по размеру, по памяти, удерживаемой на пике, и по памяти, которая так и не была освобождена.
- Ниже — дерево **точек программы** (PP). Для блоков, выделенных в каждой из них, показаны *Total*, *At t-gmax* и *At t-end*, а также стек вызовов «Allocated at». Точку можно раскрыть и увидеть её потомков (разные места вызова).

Для нашей программы при метрике по умолчанию список начинается так (стеки вызовов сокращены до наших функций):

```text
PP 1: 1,000,000 blocks, 4,182,316 bytes   <- p19_wordfreq::normalize (src/main.rs:23)  <- count_words
PP 2:    22,575 blocks,   205,947 bytes   <- p19_wordfreq::top::{closure#0} (src/main.rs:54)
PP 3:        14 blocks, 2,162,668 bytes   <- count_words (src/main.rs:49): HashMap growing (reserve_rehash)
PP 4:         1 block,  5,248,982 bytes   <- main (src/main.rs:70): fs::read_to_string
```

**Что происходит:**

1. PP 1 — это `String::with_capacity` внутри `normalize`: один блок на каждое слово, которое выдаёт `split_whitespace`. Это 98% всех блоков.
2. PP 2 — это `w.clone()` в `top`: каждое из 22 575 разных слов копируется один раз, только чтобы его отсортировать.
3. PP 3: `HashMap` рос 14 раз. Каждый раз он выделяет таблицу побольше и переносит всё; последняя таблица на t-gmax занимает 1,6 МБ.
4. PP 4: весь файл одним блоком — самая большая отдельная часть пика.

Значит, *частота* выделений — это `normalize`, а *пик* — это чтение всего файла сразу. Первое мы исправим в основной части урока, второе — в Упражнении 3.

> 🔒 **Безопасность:** файл профиля содержит пути и имена функций вашей программы. DHAT viewer работает локально, но не прикладывайте `dhat-heap.json` закрытого кода к публичным issue.

## 4. Тест, который считает выделения

Прежде чем менять код, запишем цель в виде теста: **подсчёт слов, которые уже есть в словаре, не должен выделять память на каждое слово.** Для этого у `dhat` есть тестовый режим. Добавьте в модуль `tests` в `src/main.rs`:

```rust
    #[cfg(feature = "dhat-heap")]
    #[test]
    fn known_words_do_not_allocate() {
        let mut counts = HashMap::new();
        count_words("x y z", &mut counts); // the first time, new words need Strings
        let text = "x Y z, ".repeat(1000); // made before profiling starts

        let _profiler = dhat::Profiler::builder().testing().build();
        count_words(&text, &mut counts);
        let stats = dhat::HeapStats::get();

        // Only the reusable `word` buffer, however many words there are.
        dhat::assert_eq!(stats.total_blocks, 1);
        assert_eq!(counts["x"], 1001);
    }
```

**Что происходит:**

1. `#[cfg(feature = "dhat-heap")]`: тест существует только при включённой фиче, потому что ему нужен глобальный аллокатор dhat.
2. Первый вызов `count_words` кладёт `x`, `y` и `z` в словарь. Тестовый текст строится *до* начала профилирования, поэтому его выделение не считается.
3. `Profiler::builder().testing().build()` запускает dhat в тестовом режиме: файл не пишется, если ни одна проверка не провалилась.
4. `HeapStats::get()` возвращает числа с момента запуска профилировщика: `total_blocks`, `total_bytes`, `curr_blocks`, `max_bytes` и другие.
5. `dhat::assert_eq!` работает как `assert_eq!`, но при провале ещё и сохраняет профиль, чтобы можно было увидеть, *откуда* взялись неожиданные блоки.
6. Мы ожидаем ровно 1 блок: тот единственный буфер, который новый код будет переиспользовать для всех 3000 слов.

Запустите только этот тест (dhat считает выделения *всех* потоков, поэтому другие тесты, работающие одновременно, исказили бы числа):

```bash
cargo test --features dhat-heap known_words
```

```text
---- tests::known_words_do_not_allocate stdout ----
dhat: Total:     4,000 bytes in 3,000 blocks
dhat: At t-gmax: 2 bytes in 1 blocks
dhat: At t-end:  0 bytes in 0 blocks
dhat: The data has been saved to dhat-heap.json, and is viewable with dhat/dh_view.html

thread 'tests::known_words_do_not_allocate' (6374792) panicked at src/main.rs:138:9:
dhat: assertion failed: `(left == right)`
  left: `3000`,
 right: `1`
```

Красный, как и ожидалось: 3000 слов — 3000 блоков.

## 5. Переиспользуем буфер, ищем по &str

Два изменения убирают выделение на каждое слово:

- `normalize` превращается в `normalize_into`: она пишет в `String`, которым владеет вызывающий код, и сначала очищает его. `count_words` создаёт этот буфер один раз.
- Поиск в `HashMap` использует `get_mut(word.as_str())`. В `HashMap<String, usize>` можно искать по `&str`, потому что `String` реализует `Borrow<str>`. В новый `String` через `clone()` копируется только слово, которого в словаре ещё нет.

Замените `normalize` и `count_words`:

```rust
/// Lowercase letters and digits only: "The," -> "the".
/// Writes into `out`, which is cleared first, so one buffer can be reused.
#[inline(never)]
fn normalize_into(word: &str, out: &mut String) {
    out.clear();
    if word.is_ascii() {
        for byte in word.bytes().filter(u8::is_ascii_alphanumeric) {
            out.push(char::from(byte.to_ascii_lowercase()));
        }
    } else {
        // Rare path: the full Unicode rules, exactly as before.
        out.extend(word.to_lowercase().chars().filter(|c| c.is_alphanumeric()));
    }
}
```

```rust
#[inline(never)]
fn count_words(text: &str, counts: &mut HashMap<String, usize>) {
    let mut word = String::new(); // one buffer for all words
    for raw in text.split_whitespace() {
        normalize_into(raw, &mut word);
        if word.is_empty() || is_stop_word(&word) {
            continue;
        }
        // Look up with a &str: no new String for a word we already know.
        if let Some(count) = counts.get_mut(word.as_str()) {
            *count += 1;
        } else {
            counts.insert(word.clone(), 1);
        }
    }
}
```

**Что происходит:**

1. `out.clear()` делает длину равной 0, но сохраняет ёмкость. После первых нескольких слов буфер достаточно велик для любого слова, и `push` больше никогда не выделяет память.
2. `String::new()` вообще не выделяет память; это сделает первый `push`, один раз.
3. `get_mut(word.as_str())` хеширует текст и находит запись, ничего не создавая. `insert(word.clone(), 1)` выполняется всего 22 576 раз — по одному на каждое разное слово.
4. Тест из Урока 19.3 всё ещё вызывает `normalize`. Дайте модулю `tests` маленькую вспомогательную функцию со старым интерфейсом:

```rust
    /// Test helper: the old interface, built on `normalize_into`.
    fn normalize(word: &str) -> String {
        let mut out = String::new();
        normalize_into(word, &mut out);
        out
    }
```

Теперь тесты:

```bash
cargo test --features dhat-heap known_words
cargo test
```

```text
test tests::known_words_do_not_allocate ... ok
test tests::counts_and_top_on_a_small_text ... ok
test tests::fast_normalize_matches_the_reference ... ok
```

Зелёный. Профиль кучи всей программы:

```bash
cargo run --profile profiling --features dhat-heap --bin p19-wordfreq -- small.txt
```

```text
dhat: Total:     9,269,722 bytes in 45,177 blocks
dhat: At t-gmax: 7,000,205 bytes in 14,343 blocks
dhat: At t-end:  1,088 bytes in 2 blocks
```

С 1 022 599 блоков до 45 177: 22 576 для новых слов в словаре, 22 575 для копий в `top` и немного других. Пик не изменился (7 МБ), потому что мы не меняли способ чтения файла. А скорость на большом файле — после проверки, что вывод тот же (`before-buffer` — копия бинарника до этого изменения):

```text
Benchmark 1: ./before-buffer words.txt (10 runs)
                mean     ±       σ          min     …     max
  Wall Time    1.187 s   ±   0.202 s      1.077 s   …   1.753 s
  Memory       131.1 MiB ±     0.6 MiB    129.6 MiB …   131.5 MiB
 
Benchmark 2: ./target/release/p19-wordfreq words.txt (10 runs)
                mean     ±       σ          min     …     max
  Wall Time    0.768 s   ±   0.084 s      0.714 s   …   0.953 s            -35.3%  (1.5x faster)
  Memory       131.0 MiB ±     0.6 MiB    129.6 MiB …   131.5 MiB            0.0%
```

Снова в 1,5 раза быстрее. Вместе с Уроком 19.3 программа на этом Mac ускорилась примерно с 1,75 с до 0,77 с при том же выводе. Обратите внимание на строку *Memory*: 131 МиБ в обеих версиях. Пик — следующее, что нужно исправить (Упражнение 3).

## 6. Набор приёмов для меньшего числа выделений

| Приём | Вместо | Делайте так |
|---|---|---|
| Переиспользуйте буфер | `let s = make_string(x);` в цикле | один `String`/`Vec` вне цикла, `clear()` и заполнение |
| Ищите по заимствованному ключу | `map.entry(key.to_string())` для каждого элемента | сначала `map.get_mut(key)`, выделяйте память только для новых ключей |
| Заимствуйте вместо клонирования | `Vec<(String, usize)>`, скопированный из словаря | `Vec<(&str, usize)>`, указывающий в словарь (Упражнение 2) |
| Резервируйте один раз | `Vec::new()` и много `push` | `Vec::with_capacity(n)`, когда `n` известно |
| Форматируйте в буфер | `format!(...)` или `to_string()` для каждого элемента | `write!(buffer, ...)` в переиспользуемый `String` (`use std::fmt::Write`) |
| Читайте вход потоком | `fs::read_to_string` огромного файла | `BufReader` и `read_line` в переиспользуемый `String` (Упражнение 3) |

> ⚡ **Производительность:** не применяйте это везде «на всякий случай». Такой код немного труднее читать. Используйте приёмы там, где профилировщик показывает `malloc`/`free` или где dhat показывает точку программы с миллионами блоков, и держите тест, который защищает результат.

> 📏 **Соглашение:** функции, которые пишут в буфер, переданный вызывающим кодом, часто называют `..._into` (`normalize_into`) — по аналогии с `read_to_string(&mut buf)` и `extend_from_slice` в стандартной библиотеке. Буфером владеет вызывающий код, и он решает, когда его переиспользовать.

## Частые ошибки

1. **Запуск без фичи.** `cargo run --profile profiling --bin p19-wordfreq -- small.txt` не печатает строк dhat: фича `dhat-heap` выключена, поэтому нет ни аллокатора, ни профилировщика. Добавьте `--features dhat-heap`.
2. **Назвать профилировщик `_`.** `let _ = dhat::Profiler::new_heap();` сразу же удаляет профилировщик, и он ничего полезного не записывает. Используйте настоящее имя, например `_profiler`.
3. **Поставлять программу с включённым dhat.** Аллокатор dhat замедлил нашу программу примерно в 60 раз. Держите его за фичей и никогда не включайте эту фичу в продакшен-сборках.
4. **Тесты выделений рядом с другими тестами.** dhat считает выделения всех потоков. Если другие тесты идут параллельно, `total_blocks` включает их блоки, и тест падает случайным образом. Запускайте тесты выделений отдельно — с фильтром по имени, как выше, или через `cargo test --features dhat-heap -- --test-threads=1`.
5. **Огромный `with_capacity`.** Резервирование «с запасом на всё» (`String::with_capacity(1 << 20)` на каждый элемент) меняет число выделений на пиковую память. Резервируйте столько, сколько нужно.

## Попробуйте сами

1. Откройте `dhat-heap.json` новой версии в DHAT viewer (сортировка «Total (blocks)»). Какие две точки программы остались наверху? (`clone()` в `count_words` и в `top`.)
2. Отсортируйте по «At t-gmax (bytes)». Какая точка программы держит больше всего памяти на пике? (Чтение файла.)
3. Поменяйте в тесте строку на `"x Y z, ".repeat(100_000)`. Проходит ли он по-прежнему с `total_blocks == 1`?

## Упражнения

### Упражнение 1 — Прочитайте числа (легко)

Исходная версия (Урок 19.3) дала на `small.txt` `Total: 22,929,129 bytes in 2,127,791 blocks`; версия после Урока 19.3 — `1,022,599 blocks`; после этого урока — `45,177 blocks`. В файле 1 000 000 слов. (а) Сколько блоков на слово выделяла каждая версия? (б) Почему у исходной версии больше 2?

<details>
<summary>Подсказка</summary>

Поделите на миллион. Затем вспомните, что делала исходная `normalize`: `to_lowercase`, а потом `collect` в `String`, который начинался пустым.

</details>

<details>
<summary>Ответ</summary>

(а) 2,13, 1,02 и 0,045 блока на слово. (б) Исходная версия делала один блок в `to_lowercase` и минимум один в `collect`. `collect` начинает с пустого `String` (`filter` не может знать, сколько символов пройдёт), поэтому для длинных слов строке приходилось расти, а рост — это новый, больший блок (dhat считает `realloc` новым блоком). Это дало ещё около 0,1 блока на слово.

</details>

### Упражнение 2 — Заимствуйте вместо клонирования в top (средне)

Измените `top`, чтобы она возвращала `Vec<(&str, usize)>` со срезами строк, указывающими в словарь, вместо клонированных `String`. Обновите тест `counts_and_top_on_a_small_text` и измерьте через dhat. Сколько блоков осталось?

<details>
<summary>Подсказка</summary>

`counts.iter().map(|(w, c)| (w.as_str(), *c))`. При сортировке `a.0` и `b.0` уже `&str`, поэтому сравнивайте `a.0.cmp(b.0)`. В тесте ожидаемое значение станет `vec![("x", 2), ("y", 2)]`.

</details>

<details>
<summary>Ответ</summary>

```rust
fn top(counts: &HashMap<String, usize>, n: usize) -> Vec<(&str, usize)> {
    let mut all: Vec<(&str, usize)> = counts.iter().map(|(w, c)| (w.as_str(), *c)).collect();
    // Most frequent first; equal counts in alphabetical order.
    all.sort_by(|a, b| b.1.cmp(&a.1).then(a.0.cmp(b.0)));
    all.truncate(n);
    all
}
```

В тесте: `let expected = vec![("x", 2), ("y", 2)];`. Цикл с `println!` в `main` работает без изменений. dhat теперь сообщает:

```text
dhat: Total:     8,702,549 bytes in 22,601 blocks
dhat: At t-gmax: 7,000,205 bytes in 14,343 blocks
dhat: At t-end:  1,088 bytes in 2 blocks
```

Почему это работает: результат заимствует из `counts` (компилятор связывает время жизни возвращаемых `&str` с параметром `&HashMap` по правилам опускания времён жизни, Урок 05.3), поэтому текст не копируется. На 22 575 блоков меньше; остаётся по сути один блок на каждое разное слово — он действительно нужен словарю.

</details>

### Упражнение 3 — Снизьте пик: читайте по строкам (сложно)

Пик — это весь файл в одном `String`. Измените `main`, чтобы она читала каждый файл через `BufReader` и `read_line` в один переиспользуемый `String`, вызывая `count_words` для каждой строки. Проверьте, что вывод на `words.txt` тот же, затем измерьте через dhat (блоки и t-gmax) и через `hyperfine` (время и память). Объясните компромисс.

<details>
<summary>Подсказка</summary>

`use std::fs::File; use std::io::{BufRead, BufReader};`. `read_line` *дописывает* в строку и возвращает число прочитанных байт, 0 в конце файла. Очищайте строку после каждого вызова.

</details>

<details>
<summary>Ответ</summary>

```rust
    let mut line = String::new(); // one buffer for all lines
    for path in &paths {
        let mut reader = BufReader::new(File::open(path).expect("cannot open the file"));
        while reader.read_line(&mut line).expect("cannot read the file") > 0 {
            count_words(&line, &mut counts);
            line.clear();
        }
    }
```

dhat на `small.txt` (66 666 строк):

```text
dhat: Total:     4,860,756 bytes in 143,357 blocks
dhat: At t-gmax: 2,372,324 bytes in 22,584 blocks
dhat: At t-end:  1,088 bytes in 2 blocks
```

`hyperfine` на `words.txt` (`lines-wordfreq` — новый бинарник):

```text
Benchmark 1: ./target/release/p19-wordfreq words.txt (5 runs)
                mean     ±       σ          min     …     max
  Wall Time    791.1 ms  ±    89.2 ms     727.0 ms  …   944.1 ms
  Memory       130.0 MiB ±     0.6 MiB    129.2 MiB …   130.8 MiB
 
Benchmark 2: ./lines-wordfreq words.txt (5 runs)
                mean     ±       σ          min     …     max
  Wall Time    892.7 ms  ±    55.3 ms     842.4 ms  …   971.3 ms           +12.8%
  Memory         4.9 MiB ±     0.6 MiB      4.5 MiB …     5.6 MiB          -96.2%  (26.3x less)
```

Почему это работает: файла больше нет в пике, поэтому пик кучи упал с 7,0 до 2,4 МБ на маленьком файле, а весь процесс на большом требует 4,9 МиБ вместо 130 МиБ; память больше не растёт вместе с размером файла. Цена: `count_words` создаёт свой буфер `word` один раз на *вызов*, то есть теперь на каждую строку, поэтому блоков стало 143 357 вместо 22 601, а программа примерно на 13% медленнее. Меньше выделений и меньше пиковой памяти — разные цели. (Чтобы получить и то и другое, вынесите буфер `word` из `count_words` и передавайте его снаружи или храните его в структуре вместе со словарём.)

</details>

## Вопросы для самопроверки

1. Чем частота выделений отличается от пиковой памяти? Назовите по одному исправлению для каждой.

<details>
<summary>Ответ</summary>

Частота выделений — сколько всего блоков выделяется и освобождается; она стоит процессорного времени. Пиковая память — наибольший объём памяти, удерживаемый в один момент; она стоит RAM. Переиспользование буферов снижает частоту; обработка входа по частям снижает пик.

</details>

2. Почему `dhat` спрятан за необязательной фичей, а не включён всегда?

<details>
<summary>Ответ</summary>

Он заменяет глобальный аллокатор на медленный, записывающий (здесь примерно в 60 раз медленнее). Обычные сборки и продакшен не должны платить эту цену или зависеть от крейта.

</details>

3. Что говорят «At t-gmax» и «At t-end»?

<details>
<summary>Ответ</summary>

At t-gmax: сколько памяти удерживалось в момент наибольшей кучи (пик). At t-end: что ещё оставалось выделенным, когда программа завершилась; значение, которое растёт вместе со входом, указывает на утечку.

</details>

4. Почему в `HashMap<String, usize>` можно искать по `&str`?

<details>
<summary>Ответ</summary>

Потому что `String` реализует `Borrow<str>`, а `HashMap::get` и `get_mut` принимают любую заимствованную форму ключа, у которой совпадают хеш и равенство. Поэтому для поиска не нужен временный `String`.

</details>

5. Почему тест, считающий выделения, должен запускаться отдельно?

<details>
<summary>Ответ</summary>

dhat считает выделения всего процесса, из всех потоков. По умолчанию тесты идут параллельно, поэтому их выделения тоже были бы посчитаны, и числа менялись бы от запуска к запуску.

</details>

## Итоги

- Выделения памяти стоят процессорного времени (частота выделений) и памяти (пик); измерьте и то и другое, прежде чем что-то исправлять.
- `dhat` за необязательной фичей (`dhat-heap = ["dep:dhat"]`) плюс `#[global_allocator]` и `Profiler::new_heap()` дают Total, At t-gmax и At t-end, а также `dhat-heap.json` для DHAT viewer.
- Точки программы в просмотрщике, отсортированные по блокам или байтам, показывают, какая именно строка выделяет память.
- `Profiler::builder().testing()` и `HeapStats` превращают «этот код не должен выделять память» в тест.
- Переиспользуйте буферы (функции `..._into`), ищите по `&str` и заимствуйте вместо клонирования: наша программа перешла с 2,1 миллиона до 23 тысяч блоков и стала быстрее.
- Потоковое чтение входа снижает пик с размера файла до нескольких мегабайт ценой небольшой потери во времени.

## Источники и что почитать

- [Документация крейта dhat](https://docs.rs/dhat/latest/dhat/) — настройка, шаблон с фичей `dhat-heap`, тестовый режим и `HeapStats`.
- [DHAT viewer](https://nnethercote.github.io/dh_view/dh_view.html) — веб-страница, которая открывает `dhat-heap.json`.
- [Руководство Valgrind DHAT](https://valgrind.org/docs/manual/dh-manual.html) — исходный инструмент; объясняет t-gmax, t-end, точки программы и метрики сортировки.
- [The Rust Performance Book — Heap Allocations](https://nnethercote.github.io/perf-book/heap-allocations.html) — ещё много приёмов, как выделять меньше памяти в Rust.
- [The Cargo Book — Features](https://doc.rust-lang.org/cargo/reference/features.html) — `[features]`, необязательные зависимости и `dep:`.
- [`std::io::BufRead`](https://doc.rust-lang.org/std/io/trait.BufRead.html) — `read_line` и чтение входа по частям.

---

[🏠 Начало](../START-HERE.md) · [← 19.4 Профилирование в Linux с perf](04-profiling-on-linux-with-perf.md) · [📋 Этап 19](README.md) · [19.6 Размещение данных для скорости →](06-data-layout-for-speed.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
