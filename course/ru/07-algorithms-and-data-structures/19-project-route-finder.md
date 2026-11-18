<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 07.18 Практика решения задач](18-problem-solving-practice.md) · [📋 Этап 07](README.md) · [07.20 Контрольная точка →](20-checkpoint.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/07-algorithms-and-data-structures/19-project-route-finder.md) <!-- nav -->

# Урок 07.19 — Проект: поиск маршрутов

> **Этап 07 — Алгоритмы и структуры данных** · Урок 19 из 20 · ⏱ около 6 часов (3 занятия)

## Что вы узнаете

- Как превратить текстовый файл с дорогами в граф, а алгоритм на графе — в программу командной строки.
- Как использовать BFS (Breadth-First Search, поиск в ширину: меньше всего дорог) и алгоритм Дейкстры (кратчайшее расстояние) на одной карте и почему они дают разные ответы.
- Как разделить программу на библиотеку и тонкий бинарный крейт, понятно сообщать об ошибках и тестировать и код, и программу.
- Как измерить, как два алгоритма растут с размером карты.

Это **проектный урок**. Он даёт вам цель (контракт), план с небольшими вехами, скрытые подсказки и команды, которые доказывают, что веха выполнена. Код вы пишете сами.

## Перед началом

- Вы прошли [Урок 07.18 — Практика решения задач](18-problem-solving-practice.md) и уроки о графах [07.13](13-graphs-bfs-and-dfs.md) и [07.14](14-shortest-paths-and-topological-sort.md). Вы будете использовать BFS, алгоритм Дейкстры и `BinaryHeap` ([Урок 07.12](12-heaps-and-priority-queues.md)).
- Вы знаете стиль проектов из [Урока 05.13 — Проект: minigrep](../05-rust-intermediate/13-project-minigrep.md): библиотека с логикой, тонкий бинарный крейт, `clap`, `thiserror`, интеграционные тесты и бенчмарк `criterion` ([Урок 05.11](../05-rust-intermediate/11-practical-crates-serde-clap-errors.md), [Урок 05.12](../05-rust-intermediate/12-benchmarking-basics.md)).
- Ваш проект находится в `~/projects/rust/code/p07-route-finder`, внутри вашего учебного репозитория (корневой `.gitignore` уже игнорирует `target/`). Сначала выполните `git status` в `~/projects/rust`: он должен сказать `nothing to commit, working tree clean`.
- Вы онлайн: Cargo должен скачивать крейты.

## Новые слова

| Слово | Значение |
|---|---|
| **список рёбер** (edge list) | Способ хранить граф как список его рёбер, по одному в строке: здесь `from,to,distance`. Его легко писать вручную и читать из файла. |
| **взвешенный граф** (weighted graph) | Граф, у рёбер которого есть число (*вес*), здесь длина дороги в километрах. |
| **неориентированное** (undirected) | Ребро, которое работает в обе стороны: дорога, по которой можно ехать из A в B и из B в A. |
| **маршрут** (route) | Путь от города-старта до города-цели: список городов по порядку. |
| **код возврата** (exit status) | Число, которое программа возвращает командной оболочке при завершении. `0` означает успех; любое другое число — неудачу. `echo $?` печатает его. |
| **V и E** | Число вершин (здесь городов) и рёбер (здесь дорог) графа. Время работы графовых алгоритмов записывают через них: O(V + E). |
| **приёмочный тест** (acceptance test) | Тест, который проверяет готовое поведение снаружи, например запуская программу и читая её вывод. Команды «Готово, когда» — приёмочные тесты. |

## Проект

### Что вы создадите

`route-finder` читает карту из текстового файла (список двусторонних дорог с длинами) и печатает маршрут между двумя городами с числом дорог и общим расстоянием. Алгоритм выбираете вы: `bfs` находит маршрут с **наименьшим числом дорог**; `dijkstra` (по умолчанию) находит маршрут с **наименьшим общим расстоянием**. Часто они дают разные маршруты, и вы увидите это в собственной программе.

Работать вы будете в таком порядке: настроить командную строку, прочитать и проверить карту, построить граф и BFS, добавить Дейкстру, соединить всё с понятными ошибками и кодами возврата, затем измерить и задокументировать.

```text
p07-route-finder/
├── Cargo.toml
├── README.md
├── benches/
│   └── routes.rs        ← criterion benchmark (milestone M6)
├── maps/
│   ├── small.txt        ← the sample map of this lesson
│   └── broken.txt       ← a map with a bad line (milestone M5)
├── src/
│   ├── lib.rs           ← the logic: map, graph, BFS, Dijkstra, errors
│   └── main.rs          ← the terminal: clap, printing, exit status
└── tests/
    └── cli.rs           ← integration tests (milestone M5)
```

(Вы можете разделить `lib.rs` на модули вроде `graph.rs` и `map.rs`; контракту ниже это безразлично.)

### Контракт

**Использование**

```text
route-finder --map <FILE> --from <CITY> --to <CITY> [--algo bfs|dijkstra]
```

| Параметр | Значение |
|---|---|
| `--map <FILE>` | Обязателен. Файл карты. |
| `--from <CITY>`, `--to <CITY>` | Обязательны. Названия города-старта и города-цели. Регистр имеет значение. |
| `--algo bfs\|dijkstra` | Необязателен, по умолчанию `dijkstra`. Любое другое значение — ошибка `clap`. |
| `-h`, `--help`, `-V`, `--version` | Справка и версия (делает `clap`). |

**Файл карты.** Простой текст, одна дорога в строке: `from,to,distance`.

- Пробелы вокруг трёх полей игнорируются. Пустые строки и строки, начинающиеся с `#`, пропускаются.
- Дорога работает в **обе** стороны. Город существует, как только он встретился в дороге.
- Расстояние — целое число километров не меньше 1 (оно должно помещаться в `u32`).
- Если одна и та же пара городов встречается дважды (в любом порядке), сохраняется **меньшее** расстояние.

**Ошибки в карте** (каждая останавливает программу; номер строки начинается с 1):

| Проблема | Часть сообщения после имени файла |
|---|---|
| строка не ровно с тремя полями | `line 3: expected 3 fields "from,to,distance" but found 2` |
| пустое название города | `line 3: empty city name` |
| расстояние — не целое число не меньше 1 | `line 2: invalid distance "abc" (a whole number of at least 1 is needed)` |
| дорога из города в него же | `line 4: a road cannot start and end in the same city ("A")` |

**Вывод (stdout)**, когда маршрут найден, — ровно три строки:

```text
path: Alder -> Cedar -> Fir -> Elm
roads: 3
distance: 20
```

`roads` — число дорог на маршруте, `distance` — сумма их длин в км. Если `--from` и `--to` — один и тот же город, маршрут состоит из этого одного города: `path: Elm`, `roads: 0`, `distance: 0`.

**Ошибки (stderr)**, по одной строке, всегда начинаются с `route-finder: `:

| Ситуация | Строка в stderr |
|---|---|
| файл карты не читается | `route-finder: cannot read maps/missing.txt: No such file or directory (os error 2)` |
| в карте плохая строка | `route-finder: maps/broken.txt: line 2: invalid distance "abc" (a whole number of at least 1 is needed)` |
| города нет на карте | `route-finder: unknown city "Nowhere"` |
| города не связаны | `route-finder: no route from Alder to Holly` |

В stdout не попадает ничего, кроме успешного результата.

**Код возврата:** `0` для найденного маршрута; `1` для каждой ошибки выше; `2` для неверной командной строки (это делает `clap`).

**Что должна предоставлять библиотека** (чтобы ваши тесты могли ею пользоваться; имена фиксированы, всё остальное свободно):

```rust
pub enum Algo { Bfs, Dijkstra }                       // also a clap ValueEnum
pub struct Graph { /* yours */ }                      // city_count(), road_count()
pub fn parse_map(text: &str) -> Result<Graph, MapError>;
pub struct Route { pub cities: Vec<String>, pub distance: u64 }   // roads() = cities.len() - 1
pub fn find_route(graph: &Graph, from: &str, to: &str, algo: Algo) -> Result<Route, RouteError>;
```

`MapError` и `RouteError` — перечисления `thiserror`, чьи сообщения совпадают с текстами выше (без `route-finder: ` и без имени файла).

**Правила кода**

- Логика живёт в `src/lib.rs` (и модулях). `src/main.rs` только разбирает аргументы, вызывает библиотеку, печатает и превращает ошибки в код возврата.
- Используйте собственные BFS и Дейкстру с `std::collections::BinaryHeap` (и `Reverse`) и `VecDeque`. Никаких крейтов для графов.
- Никаких `unwrap()` и `expect()` вне тестов, кроме мест, где инвариант делает сбой невозможным (добавьте комментарий, почему).
- Зависимости: `clap` (фича `derive`), `thiserror`, `anyhow`; dev-зависимости `assert_cmd`, `predicates`, `tempfile`, `criterion`. Добавляйте каждую на вехе, где она впервые используется (Cargo предупреждает о неиспользуемых зависимостях).
- `cargo fmt --check`, `cargo clippy --all-targets -- -D warnings` и `cargo test` проходят без предупреждений.

**Образец карты.** Создайте `maps/small.txt` на вехе M1. В ней восемь городов; два из них (`Gorse` и `Holly`) образуют остров без дороги к остальным.

```text
# A small map. Every line is a two-way road: from,to,distance (in km).
Alder,Birch,7
Alder,Cedar,9
Alder,Fir,14
Birch,Cedar,10
Birch,Dogwood,15
Cedar,Dogwood,11
Cedar,Fir,2
Dogwood,Elm,6
Elm,Fir,9

# A second, separate island of two towns: no road leads to it.
Gorse,Holly,3
```

В виде списка смежности (каждый город с соседями и длинами дорог):

```text
Alder:   Birch 7,   Cedar 9,   Fir 14
Birch:   Alder 7,   Cedar 10,  Dogwood 15
Cedar:   Alder 9,   Birch 10,  Dogwood 11, Fir 2
Dogwood: Birch 15,  Cedar 11,  Elm 6
Elm:     Dogwood 6, Fir 9
Fir:     Alder 14,  Cedar 2,   Elm 9
Gorse:   Holly 3
Holly:   Gorse 3
```

Из `Alder` в `Elm` BFS находит `Alder -> Fir -> Elm`: 2 дороги, 14 + 9 = 23 км. Дейкстра находит `Alder -> Cedar -> Fir -> Elm`: 3 дороги, 9 + 2 + 9 = 20 км. Меньше дорог — не то же самое, что меньше расстояние.

### Пошаговый план

Проходите вехи по порядку. После каждой запускайте её команды «Готово, когда», затем делайте коммит. Не открывайте подсказки, пока не попробовали сами.

> 📏 **Соглашение:** сообщения коммитов как на Этапе 05: тип, область в скобках, двоеточие и короткое предложение в повелительном наклонении: `feat(route-finder): parse the command line`. Нужные типы: `feat`, `test`, `docs`, `perf`, `chore`. Один коммит на веху. Используйте `git add .` внутри папки проекта и смотрите на `git status --short` перед коммитом.

---

#### M1 — Настроить проект и командную строку

**Цель:** проект, который собирается, находится под Git и понимает командную строку из контракта.

**Указания:**

1. Создайте ветку и пакет: `cd ~/projects/rust && git switch -c project/07-route-finder && cargo new code/p07-route-finder`. В `Cargo.toml` измените `name` пакета на `route-finder`, чтобы бинарный файл назывался `route-finder`.
2. Добавьте только то, что использует эта веха: `cargo add clap --features derive`.
3. Создайте `maps/small.txt` с текстом выше.
4. В `src/main.rs` определите структуру командной строки с `#[derive(Parser)]` и перечисление `Algo` с `#[derive(ValueEnum)]` (пока в `main.rs`; на M3 вы перенесёте его в библиотеку). Дайте каждому аргументу doc-комментарий. `main` только разбирает аргументы и печатает их через `{:?}` (добавьте и `Debug`).

<details>
<summary>Подсказка 1 — поля</summary>

`map: PathBuf`, `from: String`, `to: String`, `algo: Algo`, каждое с `#[arg(long)]`. Для поля-перечисления используйте `#[arg(long, value_enum, default_value_t = Algo::Dijkstra)]`. Перечислению нужны `Clone` и `Copy` в дополнение к `ValueEnum` и `Debug` (значение по умолчанию копируется).

</details>

<details>
<summary>Подсказка 2 — версия</summary>

`#[command(version)]` на структуре заставляет `--version` печатать имя пакета и версию из `Cargo.toml`.

</details>

**Готово, когда:**

```bash
cargo run -q -- -h
```

печатает строку использования `Usage: route-finder [OPTIONS] --map <MAP> --from <FROM> --to <TO>`, четыре параметра и `[default: dijkstra] [possible values: bfs, dijkstra]` для `--algo`. (Тексты описаний — ваши.) Например:

```text
Find a route between two cities on a map of roads

Usage: route-finder [OPTIONS] --map <MAP> --from <FROM> --to <TO>

Options:
      --map <MAP>    The map file: one road per line, as from,to,distance
      --from <FROM>  The city to start in
      --to <TO>      The city to go to
      --algo <ALGO>  bfs finds the fewest roads, dijkstra the shortest distance [default: dijkstra] [possible values: bfs, dijkstra]
  -h, --help         Print help (see more with '--help')
  -V, --version      Print version
```

и:

```bash
cargo run -q -- --map maps/small.txt --from Alder --to Elm --algo bfs
cargo run -q -- --map maps/small.txt --from Alder ; echo "exit: $?"
```

```text
Cli { map: "maps/small.txt", from: "Alder", to: "Elm", algo: Bfs }
error: the following required arguments were not provided:
  --to <TO>

Usage: route-finder --map <MAP> --from <FROM> --to <TO>

For more information, try '--help'.
exit: 2
```

Коммит: `git add . && git commit -m "feat(route-finder): parse the command line with clap"`.

---

#### M2 — Граф и разборщик карты

**Цель:** `parse_map` превращает текст карты в `Graph` или в `MapError` с номером строки. Граф может сообщить, сколько в нём городов и дорог.

**Указания:**

1. Добавьте `cargo add thiserror`. Создайте `src/lib.rs` (и модули, если хотите) и определите `MapError` с одним вариантом на каждую проблему из контракта; сообщения берутся из `#[error("...")]`.
2. Храните граф как **список смежности** ([Урок 07.13](13-graphs-bfs-and-dfs.md)): дайте каждому городу номер (его *id*), храните названия в `Vec<String>` и `HashMap<String, usize>` из названия в id, а для каждого id — `Vec<(usize, u32)>` из (id соседа, длина дороги).
3. `add_road(a, b, km)` создаёт города, если они новые, и добавляет дорогу в **обе** стороны. Если дорога уже есть, оставьте меньшую длину (в обе стороны!).
4. `parse_map` идёт по `text.lines().enumerate()`, пропускает пустые строки и строки с `#`, делит по `,`, обрезает пробелы, проверяет четыре проблемы контракта в разумном порядке и вызывает `add_road`.
5. Сначала тесты (см. [Урок 05.6](../05-rust-intermediate/06-testing-in-depth.md)): у образца карты 8 городов и 10 дорог; для каждой ошибки есть тест, проверяющий точный вариант, включая номер строки; `"A,B,10\nB,A,4\n"` даёт одну дорогу длиной 4.
6. Пока `main` читает файл, вызывает `parse_map` и печатает `loaded 8 cities and 10 roads` (эта строка временная; на M5 её заменят). Сделайте библиотечную цель: когда `src/lib.rs` лежит рядом с `src/main.rs`, `main.rs` может использовать библиотеку по имени пакета с подчёркиваниями: `route_finder::parse_map`.

<details>
<summary>Подсказка 1 — разбор строки</summary>

`let fields: Vec<&str> = line.split(',').map(str::trim).collect();` затем сначала проверьте `fields.len() == 3`. Расстояние вроде `-5` или `3.5` не проходит `parse::<u32>()`; `0` разбирается, но контракт требует не меньше 1. Проверяйте пустое название и дорогу в себя до расстояния или после: контракт порядок не фиксирует, но будьте последовательны и протестируйте свой порядок.

</details>

<details>
<summary>Подсказка 2 — меньшая дорога</summary>

Найдите `b` в списке соседей `a`. Если он там, замените длину на `min(old, km)` и сделайте то же в списке `b`. Если нет, добавьте новую пару в оба списка. Следите, чтобы `road_count` был честным: это половина суммы длин всех списков соседей.

</details>

**Готово, когда:**

```bash
cargo test
```

показывает, что все ваши тесты проходят (как минимум: подсчёты, четыре ошибки, меньшая дорога, пропуск комментариев и пустых строк), и:

```bash
cargo run -q -- --map maps/small.txt --from Alder --to Elm
```

```text
loaded 8 cities and 10 roads
```

Коммит: `feat(route-finder): read a map into a graph`.

---

#### M3 — BFS: маршрут с наименьшим числом дорог

**Цель:** `find_route(&graph, from, to, Algo::Bfs)` возвращает маршрут с наименьшим числом дорог или нужную ошибку.

**Указания:**

1. Определите `Route`, `RouteError` (через `thiserror`: `UnknownCity` и `NoRoute`) и перенесите `Algo` в библиотеку (здесь выведите `clap::ValueEnum`; `main.rs` теперь импортирует его оттуда).
2. Напишите BFS на id: `VecDeque` городов для посещения, вектор `seen` и `previous: Vec<Option<usize>>`, который помнит, из какого города вы попали в каждый город ([Урок 07.13](13-graphs-bfs-and-dfs.md)). Дойдя до цели, идите по `previous` назад до старта и разверните.
3. `find_route` превращает названия в id (неизвестное название → `UnknownCity`), запускает BFS (ничего не найдено → `NoRoute { from, to }`), превращает id обратно в названия и суммирует длины дорог вдоль пути в `distance` (используйте `u64`).
4. Для `Algo::Dijkstra` пока поставьте `todo!("M4")`.
5. Тесты: из `Alder` в `Elm` путь `Alder, Fir, Elm` с 2 дорогами и 23 км; из города в него же маршрут — этот город с 0 дорог; `Alder` в `Holly` — `NoRoute`; `Alder` в `Nowhere` — `UnknownCity("Nowhere")`; самодельный линейный граф `A-B-C-D` даёт путь `A, B, C, D`.

<details>
<summary>Подсказка 1 — расстояние пути</summary>

`path.windows(2)` даёт каждую пару соседних городов пути ([Урок 07.15](15-two-pointers-and-sliding-window.md)). Нужен способ спросить у графа длину дороги между двумя соседями; хватит небольшого метода `road_length(a, b) -> Option<u32>`, который ищет в списке соседей. Путь получен из графа, поэтому отсутствие дороги невозможно; скажите это в комментарии там, где делаете unwrap.

</details>

<details>
<summary>Подсказка 2 — путь</summary>

Начните с `vec![goal]`, затем повторяйте `current = previous[current]` и добавляйте, пока `current == start`. Вектор получится задом наперёд, поэтому вызовите `reverse()`. Когда `from == to`, цикл не выполняется, и путь — `[goal]`.

</details>

**Готово, когда:** `cargo test` проходит с вашими тестами BFS (как минимум пять выше), `cargo clippy --all-targets -- -D warnings` молчит, а `cargo run -q -- --map maps/small.txt --from Alder --to Elm` по-прежнему печатает `loaded 8 cities and 10 roads` (main ещё не подключён).

Коммит: `feat(route-finder): find the route with the fewest roads (BFS)`.

---

#### M4 — Дейкстра: кратчайшее расстояние

**Цель:** `Algo::Dijkstra` возвращает маршрут с наименьшим общим расстоянием.

**Указания:**

1. Ведите `best: Vec<u64>` (кратчайшее известное расстояние до каждого города, `u64::MAX` в начале, 0 для города-старта), `previous` как раньше и **min-кучу** из (расстояние, город). `BinaryHeap` — max-куча, поэтому храните `Reverse((distance, city))` ([Урок 07.12](12-heaps-and-priority-queues.md)). Тот же алгоритм вы писали в [Уроке 07.14](14-shortest-paths-and-topological-sort.md); здесь он работает с id городов и может остановиться, как только достанет из кучи цель.
2. Достаньте ближайший город. Если его извлечённое расстояние больше `best[city]`, это старая запись: пропустите её. Если это цель, остановитесь и восстановите путь. Иначе для каждого соседа, если `distance + road` меньше `best[neighbour]`, обновите `best` и `previous` и добавьте в кучу.
3. Тот же код `find_route` строит `Route`; расстояние теперь берётся прямо из алгоритма.
4. Тесты: `Alder` в `Elm` — `Alder, Cedar, Fir, Elm` с 3 дорогами и 20 км (а BFS дал 23 км: напишите тест, который это утверждает); `Alder` в `Holly` — `NoRoute`; используется меньшая из двух параллельных дорог. Затем напишите тест с **судьёй**: сравните Дейкстру с медленной заведомо верной функцией на многих маленьких случайных картах (подсказка ниже).

<details>
<summary>Подсказка 1 — судья для Дейкстры</summary>

Медленная функция: храните вектор лучших расстояний (`Option<u64>`), поставьте для старта `Some(0)` и повторяйте «для каждой дороги, в обе стороны, попробуй улучшить другой конец», пока целый проход ничего не меняет. Это O(города × дороги), но ошибиться в ней очень трудно. Создавайте карты генератором псевдослучайных чисел с фиксированным seed: скопируйте `src/testutil.rs` с его `Rng` из `code/p07-algorithms` ([Урок 07.3](03-arrays-and-vectors-under-the-hood.md), метод `range` — из [Урока 07.15](15-two-pointers-and-sliding-window.md)) в этот пакет и объявите его строкой `#[cfg(test)] mod testutil;` в `src/lib.rs`. Затем создавайте карты, например, до 8 городов и 14 дорог длиной от 1 до 9, и сравнивайте расстояния от города `c0` до всех городов.

</details>

<details>
<summary>Подсказка 2 — зачем пропускать старые записи?</summary>

Город может попасть в кучу несколько раз, каждый раз с более коротким расстоянием. Куча не умеет обновлять запись на месте, поэтому старые, более длинные записи остаются внутри. Когда одна из них выходит, `best[city]` уже меньше. Без проверки вы зря повторяете работу над соседями.

</details>

**Готово, когда:** `cargo test` проходит (тесты BFS и Дейкстры, включая судью), clippy молчит.

Коммит: `feat(route-finder): find the shortest route (Dijkstra)`.

---

#### M5 — Программа: вывод, ошибки, коды возврата

**Цель:** программа ведёт себя в точности как в контракте, а интеграционные тесты это доказывают.

**Указания:**

1. `cargo add anyhow`. В `src/main.rs`: прочитайте файл (`std::fs::read_to_string`), `parse_map`, `find_route`, напечатайте три строки. Поместите это в функцию `run(&Cli) -> anyhow::Result<()>` и вызывайте её из `main`, при `Err` печатайте `route-finder: {error:#}` в stderr и возвращайте `ExitCode::FAILURE` (код 1) — [Урок 05.13](../05-rust-intermediate/13-project-minigrep.md) делал то же.
2. Используйте `.with_context(...)` ([Урок 05.11](../05-rust-intermediate/11-practical-crates-serde-clap-errors.md)), чтобы сообщения совпадали с контрактом: `cannot read <file>` для ошибки чтения и имя файла перед `MapError`. Формат `{error:#}` печатает всю цепочку через `: `.
3. Создайте `maps/broken.txt` из двух строк: `A,B,5` и `B,C,abc` (например, `printf 'A,B,5\nB,C,abc\n' > maps/broken.txt`).
4. Интеграционные тесты в `tests/cli.rs` с `assert_cmd` ([Урок 05.13](../05-rust-intermediate/13-project-minigrep.md)): `cargo add --dev assert_cmd predicates tempfile`. Проверьте как минимум: маршрут по умолчанию (Дейкстра); `--algo bfs`; отсутствие маршрута (код 1, пустой stdout, точная строка stderr); неизвестный город; плохая карта во временном файле через `tempfile` (сообщение содержит имя файла и `line 2`); отсутствующий файл; неверная командная строка (код 2).

<details>
<summary>Подсказка 1 — каркас теста</summary>

```rust
use assert_cmd::Command;

#[test]
fn finds_the_shortest_route_by_default() {
    Command::cargo_bin("route-finder")
        .unwrap()
        .args(["--map", "maps/small.txt", "--from", "Alder", "--to", "Elm"])
        .assert()
        .success()
        .stdout("path: Alder -> Cedar -> Fir -> Elm\nroads: 3\ndistance: 20\n");
}
```

Интеграционные тесты запускаются с папкой пакета как текущей, поэтому относительный путь `maps/small.txt` работает. `.stderr(predicates::str::contains("line 2"))` проверяет часть сообщения; `.code(1)` проверяет код возврата.

</details>

<details>
<summary>Подсказка 2 — цепочка ошибок</summary>

`parse_map(&text).with_context(|| cli.map.display().to_string())?` оборачивает `MapError` в контекст — имя файла; с `{error:#}` печатается `maps/broken.txt: line 2: invalid distance ...`. `find_route(...)?` сам превращает `RouteError` в `anyhow::Error`, потому что он реализует `std::error::Error`.

</details>

**Готово, когда:**

```bash
cargo run -q -- --map maps/small.txt --from Alder --to Elm
cargo run -q -- --map maps/small.txt --from Alder --to Elm --algo bfs
cargo run -q -- --map maps/small.txt --from Elm --to Elm
```

```text
path: Alder -> Cedar -> Fir -> Elm
roads: 3
distance: 20
path: Alder -> Fir -> Elm
roads: 2
distance: 23
path: Elm
roads: 0
distance: 0
```

и каждая из этих команд печатает одну строку в stderr и заканчивается `exit: 1`:

```bash
cargo run -q -- --map maps/small.txt --from Alder --to Holly ; echo "exit: $?"
cargo run -q -- --map maps/small.txt --from Alder --to Nowhere ; echo "exit: $?"
cargo run -q -- --map maps/broken.txt --from A --to C ; echo "exit: $?"
cargo run -q -- --map maps/missing.txt --from A --to C ; echo "exit: $?"
```

```text
route-finder: no route from Alder to Holly
exit: 1
route-finder: unknown city "Nowhere"
exit: 1
route-finder: maps/broken.txt: line 2: invalid distance "abc" (a whole number of at least 1 is needed)
exit: 1
route-finder: cannot read maps/missing.txt: No such file or directory (os error 2)
exit: 1
```

`cargo test` проходит (модульные и интеграционные тесты), `cargo fmt --check` и `cargo clippy --all-targets -- -D warnings` молчат.

Коммит: `feat(route-finder): print routes and report errors with exit statuses`.

---

#### M6 — Измерить, задокументировать, выпустить

**Цель:** бенчмарк, который показывает, как BFS и Дейкстра растут с размером карты, README и тег.

**Указания:**

1. `cargo add --dev criterion@0.8`, затем объявите бенчмарк в `Cargo.toml` (**после** того, как создали файл, см. «Частые ошибки»):

```toml
[[bench]]
name = "routes"
harness = false
```

2. В `benches/routes.rs` стройте карты в форме **сетки** из `size × size` городов с дорогами между правым и нижним соседом. Разбирайте каждую карту **один раз** (вне измеряемого кода), затем измеряйте `find_route` от первого угла до противоположного, для BFS и Дейкстры, для размеров 10, 30 и 100 (100, 900 и 10 000 городов). Используйте `benchmark_group` и `BenchmarkId`, чтобы отчёт показывал оба алгоритма рядом. Используйте `std::hint::black_box` ([Урок 05.12](../05-rust-intermediate/12-benchmarking-basics.md)).
3. Запустите `cargo bench --bench routes -- --warm-up-time 1 --measurement-time 3` и поместите шесть чисел в README таблицей с предложением «измерено на моём компьютере; на разных компьютерах числа разные».
4. Напишите `README.md`: что делает программа, строка использования, формат карты, пример запуска каждого алгоритма на `maps/small.txt`, сложность обоих алгоритмов (BFS: O(V + E); Дейкстра с двоичной кучей: O((V + E) log V)), таблица ваших измерений и как запускать тесты.
5. Итоговые проверки и тег.

<details>
<summary>Подсказка 1 — текст карты-сетки</summary>

Назовите города `t{row}_{col}` и запишите дороги текстом, затем вызовите `parse_map` для него:

```rust
fn grid_map(size: usize) -> String {
    let mut text = String::new();
    for row in 0..size {
        for col in 0..size {
            let km = 1 + (row * 7 + col * 13) % 9; // lengths from 1 to 9
            if col + 1 < size {
                text.push_str(&format!("t{row}_{col},t{row}_{},{km}\n", col + 1));
            }
            if row + 1 < size {
                text.push_str(&format!("t{row}_{col},t{}_{col},{km}\n", row + 1));
            }
        }
    }
    text
}
```

Город-цель — `format!("t{}_{}", size - 1, size - 1)`.

</details>

<details>
<summary>Подсказка 2 — бенчмарк нескольких размеров</summary>

`group.bench_with_input(BenchmarkId::new("bfs", size * size), &size, |b, _| b.iter(|| ...))` создаёт одну запись на алгоритм и размер. Цикл по размерам — снаружи, цикл по двум алгоритмам — внутри.

</details>

**Готово, когда:**

```bash
cargo bench --bench routes -- --warm-up-time 1 --measurement-time 3
```

печатает шесть измерений, `bfs` и `dijkstra` для 100, 900 и 10000 городов. Пример вывода (у вас он будет другим; важна только форма: обе кривые растут примерно пропорционально числу городов, а Дейкстра в несколько раз медленнее BFS из-за кучи):

```text
corner to corner/bfs/100
                        time:   [1.0759 µs 1.0829 µs 1.0898 µs]
corner to corner/dijkstra/100
                        time:   [1.9750 µs 1.9903 µs 2.0097 µs]
corner to corner/bfs/900
                        time:   [5.1621 µs 5.1828 µs 5.2069 µs]
corner to corner/dijkstra/900
                        time:   [15.430 µs 15.502 µs 15.576 µs]
corner to corner/bfs/10000
                        time:   [53.554 µs 53.865 µs 54.138 µs]
corner to corner/dijkstra/10000
                        time:   [217.96 µs 224.54 µs 232.52 µs]
```

(Criterion печатает ещё строки о ходе работы и отчёты о выбросах; они опущены. Эти числа получены на Mac с Apple M4.) Затем:

```bash
cargo fmt --check && cargo clippy --all-targets -- -D warnings && cargo test
git add . && git commit -m "perf(route-finder): benchmark both algorithms, add the README"
git tag -a route-finder-v0.1.0 -m "route-finder 0.1.0"
git switch main && git merge project/07-route-finder
```

`git tag` показывает `route-finder-v0.1.0`, а `git log --oneline -6` — по одному коммиту на веху.

### Дополнительные цели

- **`--all`:** печатать расстояние от города-старта до каждого достижимого города, отсортированное по названию (Дейкстра без цели).
- **Карта из стандартного ввода:** `--map -` читает карту из stdin, чтобы работало `cat maps/small.txt | route-finder --map - ...`.
- **Односторонние дороги:** строка `from>to,distance` — дорога только в одну сторону. Что должно измениться в `add_road` и в подсчёте дорог?
- **Равные маршруты:** когда два маршрута имеют одно расстояние, всегда печатайте тот, который первый по алфавиту. Сначала напишите тест.
- **Поиск A\*:** добавьте городам координаты и используйте расстояние по прямой как подсказку, чтобы ускорить Дейкстру. Сравните бенчмарком.
- **Вторая проверка:** постройте карту, где каждая дорога длиной 1 км. BFS и Дейкстра тогда должны находить маршруты с одинаковым числом дорог. Напишите этот тест.

## Частые ошибки

**1. Объявить бенчмарк до того, как есть файл.** Если в `Cargo.toml` есть `[[bench]] name = "routes"`, а `benches/routes.rs` нет, **каждая** команда Cargo падает, а не только `cargo bench`:

```text
error: Cargo.toml: can't find `routes` bench at `benches/routes.rs` or `benches/routes/main.rs`. Please specify bench.path if you want to use a non-default path.
error: could not parse `route-finder` (manifest) due to 1 previous error
```

**Исправление:** сначала создайте `benches/routes.rs`, а запись `[[bench]]` добавьте потом.

**2. Использовать BFS и называть результат «кратчайшим маршрутом».** BFS считает дороги, а не километры. На образце карты он печатает 23 км там, где Дейкстра находит 20. Слову «кратчайший» нужно определение; контракт определяет два.

**3. Забыть, что один и тот же город может попасть в кучу много раз.** Программа всё равно даёт верный ответ, но делает лишнюю работу. Проверка «эта запись кучи старая?» — маленькая строка, которая сохраняет Дейкстре O((V + E) log V).

**4. Позволить `main` делать работу.** Если логика в `main.rs`, её можно проверить только запуском программы. Держите `main` тонким, тестируйте библиотеку быстрыми модульными тестами, а программу — несколькими интеграционными.

## Вопросы для самопроверки

1. Почему BFS даёт 23 км, а Дейкстра 20 км от `Alder` до `Elm`?

<details>
<summary>Ответ</summary>

BFS находит маршрут с наименьшим числом дорог (2 дороги, 14 + 9 = 23 км) и игнорирует длины. Дейкстра находит наименьшую сумму длин (3 дороги, 9 + 2 + 9 = 20 км).

</details>

2. Почему алгоритм Дейкстры нельзя использовать с отрицательными длинами дорог?

<details>
<summary>Ответ</summary>

Он берёт ближайший непосещённый город и считает его расстояние окончательным. Это верно, только если последующая дорога никогда не может сделать путь короче. С отрицательной дорогой может. (Расстояния в этом проекте не меньше 1.)

</details>

3. Каковы временные сложности BFS и Дейкстры (с двоичной кучей) на графе с V городами и E дорогами?

<details>
<summary>Ответ</summary>

BFS: O(V + E). Дейкстра: O((V + E) log V), потому что каждая операция кучи стоит O(log V). Наши измерения показывают, что на сетке оба растут примерно пропорционально числу городов, причём Дейкстра в несколько раз медленнее.

</details>

4. Зачем библиотека отделена от `main.rs`?

<details>
<summary>Ответ</summary>

Чтобы логику можно было тестировать напрямую и быстро модульными тестами (и бенчмарком), не запуская программу. `main.rs` только соединяет терминал (аргументы, печать, код возврата) с библиотекой.

</details>

5. В карте есть строка `A,B,10`, а позже `B,A,4`. Какова дорога между A и B?

<details>
<summary>Ответ</summary>

Одна двусторонняя дорога в 4 км: контракт оставляет меньшую длину, в обе стороны.

</details>

## Итоги

- Текстовый файл с дорогами — это список рёбер; прочитав его в список смежности, вы получаете граф, по которому могут ходить BFS и Дейкстра.
- BFS находит маршрут с наименьшим числом дорог за O(V + E); Дейкстра находит наименьшее общее расстояние за O((V + E) log V) с min-кучей (`BinaryHeap<Reverse<_>>`). Слову «кратчайший» всегда нужно определение.
- Библиотека плюс тонкий бинарный крейт позволяют тестировать логику модульными тестами, а программу — интеграционными; `thiserror` даёт точные ошибки в библиотеке, а `anyhow` добавляет контекст в `main`.
- У понятных ошибок фиксированная форма (`route-finder: ...` в stderr) и фиксированные коды возврата (0, 1, 2); приёмочные тесты проверяют и то и другое.
- Медленная заведомо верная функция — лучший судья для хитрого алгоритма; сравнивайте на многих маленьких случайных картах.
- Измеряйте на нескольких размерах входа, чтобы видеть рост, а не одно число; записывайте числа в README и пишите, что на разных компьютерах они разные.

## Источники и что почитать

- [The Rust Standard Library — BinaryHeap](https://doc.rust-lang.org/std/collections/struct.BinaryHeap.html) — куча и пример Дейкстры в её документации (прочитайте его, когда закончите свою версию).
- [The Rust Standard Library — VecDeque](https://doc.rust-lang.org/std/collections/struct.VecDeque.html) — очередь для BFS.
- [clap documentation (derive tutorial)](https://docs.rs/clap/latest/clap/_derive/_tutorial/index.html) — аргументы, `ValueEnum` и значения по умолчанию.
- [thiserror](https://docs.rs/thiserror/latest/thiserror/) и [anyhow](https://docs.rs/anyhow/latest/anyhow/) — типы ошибок и контекст ошибок.
- [assert_cmd](https://docs.rs/assert_cmd/latest/assert_cmd/) и [predicates](https://docs.rs/predicates/latest/predicates/) — тестирование программы командной строки.
- [Criterion.rs User Guide](https://bheisler.github.io/criterion.rs/book/) — группы бенчмарков и `BenchmarkId`.

---

[🏠 Начало](../START-HERE.md) · [← 07.18 Практика решения задач](18-problem-solving-practice.md) · [📋 Этап 07](README.md) · [07.20 Контрольная точка →](20-checkpoint.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
