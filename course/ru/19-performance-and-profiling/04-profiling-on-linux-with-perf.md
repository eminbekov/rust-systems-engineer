<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 19.3 Профилировщики и флеймграфы](03-profilers-and-flame-graphs.md) · [📋 Этап 19](README.md) · [19.5 Профилирование памяти и меньше выделений →](05-memory-profiling-and-allocations.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/19-performance-and-profiling/04-profiling-on-linux-with-perf.md) <!-- nav -->

# Урок 19.4 — Профилирование в Linux с perf

> **Этап 19 — Производительность и профилирование** · Урок 4 из 11 · ⏱ около 90 минут

## Что вы узнаете

- Что такое `perf` и почему это стандартный профилировщик на Linux-серверах.
- Как безопасно запускать `perf`: в ВМ (виртуальной машине) `dev` или в привилегированном контейнере, но никогда не от root на вашем Mac.
- Как читать `perf stat` (счётчики), `perf record` и `perf report` (записанные стеки вызовов).
- Два способа пройти по стеку вызовов — указатели кадра и DWARF — и чего стоит каждый.
- Как превратить профиль из Linux во флеймграф с помощью `inferno` и почему одно и то же исправление даёт в Linux другое ускорение.

## Перед началом

- Вы прошли [Урок 19.3 — Профилировщики и флеймграфы](03-profilers-and-flame-graphs.md), и у вас есть пакет `code/p19-wordfreq` с `src/main.rs` (быстрая версия), `src/bin/slow.rs` и `src/bin/gen.rs`.
- Вы знаете системные вызовы ([Урок 13.3 — Системные вызовы](../13-operating-systems/03-system-calls.md)) и умеете запускать Rust в контейнере ([Урок 11.13 — Dockerfile для программ на Rust](../11-linux-shell-and-containers/13-dockerfiles-for-rust.md)).
- Docker Desktop запущен, или запущена ваша ВМ `dev` из Этапа 11 (`limactl shell dev`).

## Новые слова

| Слово | Значение |
|---|---|
| **perf** | Профилировщик, который поставляется вместе с ядром Linux. Умеет считать события и записывать стеки вызовов программ и самого ядра. |
| **событие** (event) | То, что ядро или процессор (CPU, Central Processing Unit) умеют считать: время на CPU (`task-clock`), переключения контекста, страничные сбои, такты процессора, инструкции. |
| **программное событие** (software event) | Событие, которое считает ядро, например `task-clock` или `page-faults`. Работает везде, в том числе в виртуальных машинах. |
| **аппаратное событие** (hardware event) | Событие, которое считает сам процессор в своём **PMU** (Performance Monitoring Unit, блок мониторинга производительности), например `cycles` или `cache-misses`. Виртуальные машины часто его не предоставляют. |
| **IPC** (Instructions Per Cycle, инструкций за такт) | Сколько инструкций процессор в среднем завершает за такт. Низкий IPC обычно значит, что процессор ждёт, например, память. |
| **привилегированный контейнер** (privileged container) | Контейнер, запущенный с `--privileged`. У него почти все права root на (виртуальной) машине, где он работает. Используйте его только для локальных экспериментов. |
| **указатель кадра** (frame pointer) | Регистр, указывающий на кадр стека текущей функции. Если каждая функция его хранит, профилировщик быстро переходит от кадра к кадру. |
| **DWARF** | Стандартный формат отладочной информации в Linux (название — игра слов, а не аббревиатура). Он также описывает, как раскрутить стек без указателей кадра. |
| **раскрутка стека** (unwinding) | Проход по стеку от текущей функции обратно к `main`, чтобы найти цепочку вызовов. |
| **DSO** (Dynamic Shared Object, динамический разделяемый объект) | Файл с кодом: ваша программа, разделяемая библиотека вроде `libc.so.6` или ядро (`[kernel.kallsyms]` в perf). |
| **libc** | Стандартная библиотека C в системе. В Linux она предоставляет `malloc` и `free`, которыми пользуется аллокатор Rust по умолчанию. |

## 1. Зачем профилировать ещё и в Linux

Ваши сервисы работают в Linux, а не в macOS. Ядро Linux, библиотека C и её аллокатор памяти — всё это не такое, как на вашем Mac, поэтому одна и та же программа может тратить время в разных местах. А ещё `perf` умеет то, чего не умеет `samply` на macOS: он видит **внутрь ядра**, так что профиль показывает время внутри системных вызовов, таких как `read`, рядом с вашими функциями.

`perf` общается с ядром через системный вызов `perf_event_open`. Поскольку он может наблюдать за другими программами и за ядром, Linux его ограничивает. Два безопасных способа запустить его для этого урока:

- **В ВМ `dev`** (Этап 11). Установите его командой `sudo apt-get install -y linux-perf` и запускайте `perf` через `sudo`, потому что Ubuntu по умолчанию разрешает `perf` только root.
- **В привилегированном контейнере** из официального образа `rust:1.100` (Debian). Все выводы в этом уроке получены именно так.

> 🔒 **Безопасность:** `--privileged` даёт контейнеру права root над виртуальной машиной Linux, которую использует Docker. Используйте его только для локальных экспериментов со своим кодом, никогда с образами, которым не доверяете, и никогда на общем или продакшен-сервере. Никогда не запускайте `perf` от root на своём Mac.

## 2. Оболочка Linux с perf

Из папки вашего пакета на Mac:

```bash
cd ~/projects/rust/code/p19-wordfreq
docker run --rm -it --privileged -v "$PWD":/work -w /work \
  -v course-cargo-registry:/usr/local/cargo/registry rust:1.100 bash
```

**Что происходит:**

1. `--rm -it` запускает интерактивный контейнер, который удаляется, когда вы из него выходите.
2. `--privileged` разрешает `perf_event_open`. Что бывает без него, показано в «Частых ошибках».
3. `-v "$PWD":/work` делает папку вашего пакета видимой внутри как `/work`. Файлы, которые вы там создаёте, появляются и на Mac.
4. Именованный том `course-cargo-registry` сохраняет скачанные крейты между контейнерами.

Внутри контейнера установите `perf` и `rustfilt` и соберите программу. Бинарники для Linux не должны смешиваться с бинарниками для macOS в `target/`, поэтому отправляем их в `target/linux`:

```bash
apt-get update && apt-get install -y linux-perf
cargo install --locked rustfilt
export CARGO_TARGET_DIR=target/linux
cargo build --profile profiling
perf --version
```

```text
perf version 6.12.117
```

`words.txt` из Урока 19.3 уже на месте, потому что папка общая. Бинарники для Linux теперь лежат в `target/linux/profiling/`: `slow` (исходная версия) и `p19-wordfreq` (быстрая).

## 3. Считаем с perf stat

`perf stat` запускает программу и считает события, пока она работает:

```bash
perf stat ./target/linux/profiling/slow words.txt > /dev/null
```

```text
 Performance counter stats for './target/linux/profiling/slow words.txt':

           1425.51 msec task-clock                       #    0.937 CPUs utilized
              1044      context-switches                 #  732.371 /sec
                 0      cpu-migrations                   #    0.000 /sec
             29742      page-faults                      #   20.864 K/sec
   <not supported>      cycles
   <not supported>      instructions
   <not supported>      branches
   <not supported>      branch-misses

       1.520992709 seconds time elapsed

       1.289571000 seconds user
       0.136636000 seconds sys
```

**Что происходит** (построчно):

1. `> /dev/null` выбрасывает собственный вывод программы; `perf` печатает свой отчёт в поток ошибок, поэтому вы его всё равно видите.
2. **task-clock:** 1425 мс процессорного времени. «0.937 CPUs utilized» означает, что единственный поток работал 94% прошедшего времени.
3. **context-switches:** сколько раз ядро забирало процессор у нашего потока (Урок 13.9). Около 1000 за 1,5 с — нормально для загруженной машины.
4. **page-faults:** 29 742 раза программа обращалась к памяти, которую ядро ещё не отобразило (Урок 13.10). Чтобы прочитать файл в 130 МБ (мегабайт) в `String`, нужно много новых страниц.
5. **cycles, instructions, branches, branch-misses: `<not supported>`.** Это **аппаратные события** из PMU процессора. Docker Desktop запускает Linux в виртуальной машине, которая не пробрасывает PMU, поэтому здесь эти счётчики недоступны (ВМ `dev` на Mac ведёт себя так же). На настоящей Linux-машине вы увидели бы числа, и `perf` напечатал бы ещё **IPC** — инструкций за такт: значение заметно ниже 1 часто означает, что процессор ждёт память, а 3 или 4 — что он занят вычислениями.
6. **user / sys:** время в нашем коде и библиотеках против времени внутри ядра (системные вызовы, страничные сбои).

> 💡 **Совет:** `perf stat -r 5 <команда>` запускает команду пять раз и печатает среднее и разброс (`+- 3.7%`), как маленький `hyperfine`.

## 4. Записываем замеры: perf record и perf report

`perf record` снимает замеры с программы так же, как `samply`; `-g` дополнительно записывает стек вызовов каждого замера:

```bash
perf record -g ./target/linux/profiling/slow words.txt > /dev/null
```

```text
[ perf record: Woken up 5 times to write data ]
[ perf record: Captured and wrote 1.092 MB perf.data (6889 samples) ]
```

Замеры лежат в файле `perf.data` (по умолчанию около 4000 замеров в секунду). `perf report` его читает. Без параметров он открывает интерактивный текстовый экран: стрелки перемещают, `Enter` открывает меню, чтобы приблизить выбранную функцию или посмотреть её аннотированный код, `+` раскрывает её цепочку вызовов, `/` фильтрует по имени, `h` показывает все клавиши, а `q` выходит. Для печатного отчёта используйте `--stdio`. Имена Rust в отчёте искажены, поэтому пропускаем его через `rustfilt`:

```bash
perf report --stdio --no-children --sort symbol --percent-limit 2 -g none | rustfilt
```

```text
# Samples: 6K of event 'task-clock:ppp'
# Event count (approx.): 1722250000
#
# Overhead  Symbol
# ........  ..........................................................
#
    20.31%  [.] slow::count_words
    13.05%  [.] slow::is_stop_word
    11.31%  [.] <alloc::string::String as core::iter::traits::collect::FromIterator<char>>::from_iter::<...>
    10.70%  [.] cfree
     7.71%  [.] malloc
     6.60%  [.] <std::hash::random::RandomState as core::hash::BuildHasher>::hash_one::<&alloc::string::String>
     5.17%  [.] <hashbrown::map::HashMap<alloc::string::String, usize, std::hash::random::RandomState>>::rustc_entry
     4.59%  [.] <str>::to_lowercase
     4.22%  [.] <core::hash::sip::Hasher<core::hash::sip::Sip13Rounds> as core::hash::Hasher>::write
     2.28%  [.] __rustc::__rdl_alloc
     2.02%  [.] <alloc::raw_vec::RawVecInner>::finish_grow
```

(Заголовок сокращён, одно длинное имя обрезано.) **Что происходит:**

1. `--no-children` сортирует по **собственному времени** («Overhead» здесь означает собственные замеры). `--children` сортировал бы по общему времени.
2. `--sort symbol` группирует по функциям; `--percent-limit 2` скрывает всё ниже 2%; `-g none` скрывает цепочки вызовов под каждой строкой.
3. `[.]` отмечает код, выполняемый в пространстве пользователя; `[k]` отмечал бы код ядра.
4. `cfree` и `malloc` — функции аллокатора из glibc, библиотеки C в Debian. `cfree` — это имя, под которым glibc экспортирует `free`.

Теперь по общему времени (`--children`), только строки выше 10%:

```bash
perf report --stdio --children --sort symbol --percent-limit 10 -g none | rustfilt
```

```text
    99.99%     0.00%  [.] slow::main
    ...
    93.90%    20.31%  [.] slow::count_words
    30.21%     0.17%  [.] slow::normalize
    20.13%    11.31%  [.] <alloc::string::String as core::iter::traits::collect::FromIterator<char>>::from_iter::<...>
    16.36%     5.17%  [.] <hashbrown::map::HashMap<alloc::string::String, usize, std::hash::random::RandomState>>::rustc_entry
    13.12%    13.05%  [.] slow::is_stop_word
    11.31%    10.70%  [.] cfree
    10.86%     6.60%  [.] <std::hash::random::RandomState as core::hash::BuildHasher>::hash_one::<&alloc::string::String>
```

Первый столбец — общее время («Children»), второй — собственное. Сравните с профилем на macOS из Урока 19.3:

```text
                          macOS (samply)     Linux (perf)
normalize, total             ~50%               30%
malloc + free + growing      ~45%               ~23%
HashMap (entry + hashing)     ~4%               16%
is_stop_word                  <1%               13%
```

Та же программа, тот же вход — и совсем разные картины. `malloc` и `free` из glibc для таких крошечных строк гораздо дешевле, чем аллокатор macOS, поэтому в Linux проблема выделения памяти меньше, и становятся видны другие части.

## 5. Указатели кадра или DWARF

Чтобы найти цепочку вызовов замера, `perf` должен **раскрутить** стек. `-g` использует **указатели кадра**: идёт по цепочке сохранённых адресов кадров — это быстро и даёт маленькие файлы. Работает это только там, где код хранит указатели кадра. Rust в Linux их не включает принудительно, как и многие системные библиотеки, поэтому некоторые цепочки обрываются. В одном из наших запусков у нескольких замеров цепочка заканчивалась сырым адресом вроде `0x9aaaadeb5c4eb` вместо имени функции.

Альтернатива — `--call-graph dwarf`. Для каждого замера `perf` копирует в файл кусок стека (по умолчанию 8 КБ, килобайт), а `perf report` потом раскручивает его по отладочной информации DWARF. Попробуйте:

```bash
perf record --call-graph dwarf -o perf-dwarf.data ./target/linux/profiling/slow words.txt > /dev/null
perf report -i perf-dwarf.data --stdio --children --sort symbol --percent-limit 15 -g none | rustfilt
```

```text
[ perf record: Captured and wrote 60.797 MB perf-dwarf.data (7450 samples) ]
    79.89%     0.01%  [.] slow::main
    ...
    79.06%    17.68%  [.] slow::count_words
    26.55%     0.21%  [.] slow::normalize
    19.28%    13.05%  [.] <alloc::string::String as core::iter::traits::collect::FromIterator<char>>::from_iter::<...>
    17.70%     0.00%  [.] <std::collections::hash::map::HashMap<alloc::string::String, usize>>::entry (inlined)
    ...
    15.57%     0.00%  [.] <core::str::iter::SplitWhitespace as core::iter::traits::iterator::Iterator>::next (inlined)
```

(Сокращено: в полном списке много строк с `(inlined)`.) Компромисс очевиден:

| | `-g` (указатели кадра) | `--call-graph dwarf` |
|---|---|---|
| Размер файла для нашего запуска | 1,1 МБ | 61 МБ |
| Встроенные функции | не показаны | показаны как `(inlined)` |
| Оборванные цепочки | там, где у кода нет указателей кадра | там, где стек глубже скопированных 8 КБ |

С DWARF внезапно видно, что `split_whitespace` (встроенная в `count_words`) стоит около 15% — с `-g` этого не видно. Но только 80% замеров доходят до `main`; для остальных 20% скопированного куска стека не хватило. Начинайте с `-g`, потому что это дёшево, и переходите на DWARF, когда нужны подробности о встроенных функциях.

> 💡 **Совет:** третий способ — скомпилировать свой код с указателями кадра: `RUSTFLAGS="-C force-frame-pointers=yes" cargo build --profile profiling`. Тогда `-g` хорошо работает для ваших функций при маленьких файлах. Стандартная библиотека, которую вы подключаете, уже скомпилирована заранее, поэтому её функции сохраняют свои исходные настройки.

## 6. Флеймграф из perf

`perf script` печатает каждый замер с его стеком в виде текста. Этот текст — вход для `inferno-collapse-perf`. Можно установить `inferno` и в контейнер, но он уже есть на вашем Mac (Урок 19.3), а папка общая. В контейнере:

```bash
perf script > perf.txt
```

Затем на Mac, во втором терминале в той же папке:

```bash
inferno-collapse-perf perf.txt | rustfilt | inferno-flamegraph > flame-linux.svg
open -a Safari flame-linux.svg
```

**Что происходит:**

1. `perf script` пишет около 7 МБ текста: для каждого замера строку-заголовок (программа, ID потока, время, событие) и по строке на каждый кадр стека, включая кадры ядра, такие как `do_page_fault`.
2. `inferno-collapse-perf` превращает это в одну строку на каждый отдельный стек с количеством; `rustfilt` выполняет деманглинг; `inferno-flamegraph` рисует.
3. На картинке `count_words` — широкая башня посередине; прямоугольники `is_stop_word` и `HashMap` заметно шире, чем на графе из macOS; у левого края тонкая башня показывает загрузку программы и то, как ядро обрабатывает **страничные сбои**.

Добавьте `perf.data*`, `perf.txt` и `*.svg` в `.gitignore`: профили — это результат работы, а не исходный код.

## 7. То же исправление — другое ускорение

В Уроке 19.3 быстрый путь для слов в ASCII (American Standard Code for Information Interchange: базовые латинские буквы, цифры и знаки) ускорил программу в 1,5 раза на macOS. В Linux, с `perf stat -r 5`:

```bash
perf stat -r 5 ./target/linux/profiling/slow words.txt > /dev/null
perf stat -r 5 ./target/linux/profiling/p19-wordfreq words.txt > /dev/null
```

```text
           1347.59 msec task-clock                       #    0.979 CPUs utilized               ( +-  3.71% )
            1.3768 +- 0.0514 seconds time elapsed  ( +-  3.74% )
           1139.72 msec task-clock                       #    0.978 CPUs utilized               ( +-  3.08% )
            1.1658 +- 0.0357 seconds time elapsed  ( +-  3.06% )
```

(Только по две ключевые строки каждого отчёта.) 1,38 с → 1,17 с: примерно в **1,18 раза** быстрее, а не в 1,5. Исправление убрало выделения памяти, а в Linux они просто дешевле. Ни одно из чисел не ошибочно. Вывод: **измеряйте на той платформе, где будете развёртывать**, а ускорения, измеренные на ноутбуке, считайте первой подсказкой.

> 🧪 **Тестирование:** профиль подсказывает, что менять дальше (в Linux: `is_stop_word`, хеширование, `split_whitespace`). Прежде чем трогать `count_words` или `top`, закрепите их поведение маленьким тестом (Упражнение 2). Код подсчёта легко сломать незаметно, например изменив порядок слов с одинаковым количеством.

## Частые ошибки

1. **Забыть `--privileged`** (или `sudo` в ВМ). Без него `perf` не может открыть свои события:

   ```text
   Error:
   No permission to enable task-clock event.
   ```

2. **Смешивать сборки для macOS и Linux в одном `target/`.** Если запустить `cargo build` внутри контейнера без `CARGO_TARGET_DIR`, файлы для Linux попадут в тот же `target/release`, которым пользуется ваш Mac, и обе стороны будут постоянно пересобирать или путаться. Внутри контейнера используйте `target/linux` (или другую папку).
3. **Профилировать бинарник без символов.** `strip` удаляет таблицу символов. Профиль тогда показывает только адреса:

   ```text
    11.98%  stripped           [.] 0x00000000000099e0
    10.48%  libc.so.6          [.] cfree
     9.46%  stripped           [.] 0x0000000000009d78
   ```

   Профилируйте сборку `profiling` и не ставьте `strip = true` в профиле, который профилируете.
4. **Читать искажённые имена.** Без `rustfilt` `perf` 6.12 показывает имена Rust вида `_RNvCs5bwcD7iyBfD_4slow11count_words`. Пропускайте `perf report` и свёрнутые стеки через `rustfilt`.
5. **Считать, что `<not supported>` означает ноль.** Это значит «эта машина не умеет это считать», а не «этого не было». Аппаратным счётчикам нужно настоящее железо (или ВМ, которая пробрасывает PMU).

## Попробуйте сами

1. Запустите `perf report` без `--stdio`, выберите `slow::count_words` и нажмите `e`, чтобы раскрыть её цепочки вызовов. Какие пути самые большие? Нажмите `h`, чтобы увидеть остальные клавиши.
2. Запишите быструю версию: `perf record -g ./target/linux/profiling/p19-wordfreq words.txt > /dev/null` — и выведите список по собственному времени. Какая функция теперь наверху?
3. Выполните `perf stat -e task-clock,page-faults ./target/linux/profiling/slow words.txt > /dev/null`. `-e` выбирает, какие события считать.

## Упражнения

### Упражнение 1 — perf stat как секундомер (легко)

С помощью `perf stat -r 5` сравните `slow` и `p19-wordfreq` в Linux. Какие строки отчёта вы сравниваете и каково ускорение?

<details>
<summary>Подсказка</summary>

Смотрите на строку `seconds time elapsed` с её частью `+-` и на `task-clock`. Поделите одно прошедшее время на другое.

</details>

<details>
<summary>Ответ</summary>

Два запуска из раздела 7: `1.3768 +- 0.0514` и `1.1658 +- 0.0357` секунды, значит 1,3768 ÷ 1,1658 ≈ 1,18. Значения `+-` (около 3–4%) гораздо меньше разницы в 18%, так что разница реальная. `task-clock` (1348 мс против 1140 мс) рассказывает то же самое в процессорном времени. Ваши числа будут другими; отношение должно быть похожим.

</details>

### Упражнение 2 — Закрепите поведение перед следующим изменением (средне)

Профиль в Linux указывает на `count_words` и `HashMap`. Прежде чем кто-то начнёт их оптимизировать, напишите в `src/main.rs` тест, который вызывает `count_words` для текста `"z y, Z! the Y x X. The w"` и проверяет, что (а) разных слов 4 (`"the"` — стоп-слово), (б) `"w"` посчитано один раз, (в) `top(&counts, 2)` возвращает `x` и `y` с количеством 2 именно в этом порядке.

<details>
<summary>Подсказка</summary>

`counts["w"]` читает значение из `HashMap` по ключу (паникует, если ключа нет, — в тесте это нормально). У `x`, `y` и `z` у всех количество 2; `top` сортирует одинаковые количества по алфавиту.

</details>

<details>
<summary>Ответ</summary>

Добавьте этот тест в модуль `tests` из Урока 19.3:

```rust
    #[test]
    fn counts_and_top_on_a_small_text() {
        let mut counts = HashMap::new();
        count_words("z y, Z! the Y x X. The w", &mut counts);
        assert_eq!(counts.len(), 4); // "the" is a stop word
        assert_eq!(counts["w"], 1);
        // Equal counts come out in alphabetical order.
        let expected = vec![("x".to_string(), 2), ("y".to_string(), 2)];
        assert_eq!(top(&counts, 2), expected);
    }
```

```text
test tests::counts_and_top_on_a_small_text ... ok
test tests::fast_normalize_matches_the_reference ... ok
test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

Почему это работает: в тексте `z`, `y`, `x` встречаются по два раза (в разном регистре и с пунктуацией), `w` — один раз, а `the` — дважды, и оно пропускается. Ничья между `x`, `y` и `z` проверяет алфавитное правило в `top`, которое оптимизированная версия легко может потерять.

</details>

### Упражнение 3 — Кто тратит время: наш код, libc или ядро? (сложно)

С помощью `perf report --sort dso` разделите время в `perf.data` медленной версии по DSO. Затем выведите самые большие функции внутри libc и внутри ядра (`--dsos libc.so.6` и `--dsos '[kernel.kallsyms]'`). Объясните каждую часть.

<details>
<summary>Подсказка</summary>

```bash
perf report --stdio --no-children --sort dso -g none
perf report --stdio --no-children --sort symbol --dsos libc.so.6 -g none --percent-limit 0.3
```

</details>

<details>
<summary>Ответ</summary>

```text
    75.18%  slow
    20.03%  libc.so.6
     4.79%  [kernel.kallsyms]
```

Внутри libc: `cfree` 10.70%, `malloc` 7.71%, `memcmp` 1.00%, `realloc` 0.58%. Внутри ядра: `__pi_clear_page` 1.29%, `__arch_copy_to_user` 0.62%, `charge_memcg` 0.36% и много мелких.

Почему это работает: три четверти времени — наш собственный код на Rust (включая встроенную стандартную библиотеку). Почти все 20% libc — это аллокатор: цена короткоживущих строк из `normalize`. 5% ядра — плата за ввод: `clear_page` заполняет нулями новые страницы памяти, когда растёт `String` на 130 МБ (страничные сбои), `copy_to_user` копирует данные файла в наш буфер во время `read`, а `charge_memcg` учитывает память в cgroup контейнера (Урок 13.16). Часть ядра имела бы значение только для программы, которая читает гораздо больше, чем вычисляет.

</details>

## Вопросы для самопроверки

1. Почему профили одной и той же программы в macOS и в Linux могут выглядеть по-разному?

<details>
<summary>Ответ</summary>

Операционная система, библиотека C и особенно аллокатор памяти разные, поэтому одни и те же операции стоят по-разному. В нашей программе выделение памяти занимало около половины времени на macOS и около четверти в Linux.

</details>

2. Что означает `<not supported>` рядом с `cycles` в `perf stat`?

<details>
<summary>Ответ</summary>

Машина не умеет считать это аппаратное событие — здесь потому, что виртуальная машина Docker не открывает счётчики производительности процессора. Это не значит, что значение равно нулю.

</details>

3. Чем в `perf report` отличаются `--children` и `--no-children`?

<details>
<summary>Ответ</summary>

`--children` сортирует по общему времени (функция плюс всё, что она вызвала) и показывает оба столбца; `--no-children` сортирует только по собственному времени.

</details>

4. Назовите одно преимущество и один недостаток `--call-graph dwarf` по сравнению с `-g`.

<details>
<summary>Ответ</summary>

Преимущество: работает без указателей кадра и показывает встроенные функции. Недостаток: гораздо более крупные файлы (здесь 61 МБ вместо 1,1 МБ) и больше работы при записи; очень глубокие стеки всё равно могут обрезаться.

</details>

5. Почему мы запускаем `perf` в ВМ или в привилегированном контейнере, а не прямо на Mac?

<details>
<summary>Ответ</summary>

`perf` — инструмент Linux, и ему нужен `perf_event_open` ядра Linux. Ещё ему нужны повышенные права, а их мы даём только в одноразовом окружении Linux, но никогда — от root на своём Mac.

</details>

6. Исправление дало 1,5× на вашем Mac и 1,18× на Linux-сервере. Какое число вы сообщите?

<details>
<summary>Ответ</summary>

Число для Linux, если программа работает в продакшене в Linux, — вместе с условиями, в которых оно измерено. Число для Mac полезно как подсказка, но пользователи получат скорость Linux.

</details>

## Итоги

- `perf` — стандартный профилировщик Linux; он видит ваш код, библиотеки **и ядро**.
- Запускайте его в ВМ `dev` (через `sudo`) или в контейнере с `--privileged`; задайте `CARGO_TARGET_DIR=target/linux`, чтобы сборки для Linux и macOS не смешивались.
- `perf stat` считает события: `task-clock`, переключения контекста, страничные сбои; аппаратным счётчикам вроде тактов и IPC нужно настоящее железо.
- `perf record -g` + `perf report --stdio | rustfilt` дают собственное и общее время по функциям; `[k]` отмечает код ядра.
- `-g` (указатели кадра) — дёшево; `--call-graph dwarf` показывает встроенные функции, но даёт большие файлы.
- `perf script | inferno-collapse-perf | rustfilt | inferno-flamegraph` строит флеймграф.
- Одно и то же изменение может давать разное ускорение на разных платформах. Измеряйте там, где развёртываете.

## Источники и что почитать

- [perf wiki: Tutorial](https://perfwiki.github.io/main/tutorial/) — официальное руководство по `perf stat`, `perf record` и `perf report`.
- [Страница руководства perf-record(1)](https://man7.org/linux/man-pages/man1/perf-record.1.html) — все параметры, включая `-g` и `--call-graph dwarf`.
- [Brendan Gregg — perf Examples](https://www.brendangregg.com/perf.html) — много практичных однострочников и объяснений.
- [The Rust Performance Book — Profiling](https://nnethercote.github.io/perf-book/profiling.html) — заметки о `perf` с Rust, отладочной информации и указателях кадра.
- [inferno на GitHub](https://github.com/jonhoo/inferno) — `inferno-collapse-perf` и отрисовщик флеймграфов.
- [Параметры кодогенерации rustc: force-frame-pointers](https://doc.rust-lang.org/rustc/codegen-options/index.html) — флаг, который сохраняет указатели кадра в вашем коде.

---

[🏠 Начало](../START-HERE.md) · [← 19.3 Профилировщики и флеймграфы](03-profilers-and-flame-graphs.md) · [📋 Этап 19](README.md) · [19.5 Профилирование памяти и меньше выделений →](05-memory-profiling-and-allocations.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
