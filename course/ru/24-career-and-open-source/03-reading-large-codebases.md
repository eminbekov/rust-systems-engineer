<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 24.2 Ваше портфолио](02-building-your-portfolio.md) · [📋 Этап 24](README.md) · [24.4 Ваш первый вклад в open source →](04-your-first-open-source-contribution.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/24-career-and-open-source/03-reading-large-codebases.md) <!-- nav -->

# Урок 24.3 — Как читать большие кодовые базы

> **Этап 24 — Карьера и open source** · Урок 3 из 9 · ⏱ около 135 минут

## Что вы узнаете

- Почему чтение кода — ключевой навык и почему большой проект никогда не читают с первой строки до последней.
- Метод из шести шагов: вопрос, документы, карта, точки входа, один прослеженный путь и запуск.
- Как отработать метод на настоящем open-source-проекте на Rust, ripgrep, с помощью `git grep`, `git log` и `git blame`.
- Как быстро построить карту размеров любой кодовой базы на Rust собственной небольшой программой и протестировать её.

## Перед началом

- Вы прошли [Урок 24.2 — Ваше портфолио](02-building-your-portfolio.md).
- Вы знаете рабочие пространства Cargo ([Урок 08.12 — Структура проекта и рабочие пространства](../08-software-design/12-project-layout-and-workspaces.md))
  и команды истории Git ([Урок 01.4 — История и просмотр изменений](../01-git-and-version-control/04-viewing-history-and-changes.md)).
- Ваша Linux-ВМ `dev` из [Урока 11.1](../11-linux-shell-and-containers/01-linux-and-a-virtual-machine.md) работает.
  Она нужна только для необязательного шага «сборка и тесты».

## Новые слова

| Слово | Значение |
|---|---|
| **кодовая база** (codebase) | Весь исходный код одного проекта вместе с тестами, скриптами и документами. |
| **точка входа** (entry point) | Место, где начинается выполнение или куда приходит внешний код: `main` для программы, публичные элементы `lib.rs` для библиотеки. |
| **прослеживание пути** (tracing a path) | Следование за одним поведением по коду, от места, где оно начинается, до места, где оно происходит, файл за файлом. Не путайте с трассировкой в логировании из Урока 09.10. |
| **карта кода** (code map) | Ваши собственные короткие заметки: главные части кодовой базы, что делает каждая и где она находится. |
| **эвристика** (heuristic) | Правило, которое верно в большинстве случаев, но не всегда, например «файл с нулевым байтом — двоичный». |
| **байт NUL** (NUL byte) | Байт со значением 0 (`b'\x00'`). В обычном тексте его почти не бывает; в двоичных файлах — часто. |
| **частичный клон** (partial clone) | Клон Git, который загружает историю (коммиты и списки папок), а содержимое файлов — только когда оно нужно. Он гораздо меньше полного клона. |
| **`git grep`** | Команда Git, которая ищет по файлам репозитория. Работает быстро и пропускает файлы, которые Git игнорирует. |
| **`git blame`** | Команда Git, которая показывает для каждой строки файла последний коммит, изменивший её. |
| **ВМ** (VM, Virtual Machine, виртуальная машина) | Целый компьютер, смоделированный программно. Ваша Linux-ВМ `dev` из Урока 11.1 работает внутри вашего Mac. |
| **API** (Application Programming Interface, программный интерфейс) | Функции и типы, которые библиотека предлагает другому коду. |
| **I/O** (Input/Output, ввод-вывод) | Чтение и запись данных: файлы, сеть, терминал. |
| **ID процесса** (process ID; ID = identifier, идентификатор) | Число, которое операционная система присваивает каждой работающей программе (Урок 13.2). |
| **лицензия MIT** (MIT licence) | Короткая open-source-лицензия, названная по Массачусетскому технологическому институту (Massachusetts Institute of Technology) (Урок 24.2). |
| **ИИ** (AI, Artificial Intelligence, искусственный интеллект) | Программы, которые имитируют человеческие способности, например писать текст или код. У многих проектов теперь есть правила о вкладе, написанном с помощью ИИ. |

## 1. Почему чтение кода — это навык

Инженеры тратят больше времени на **чтение** кода, чем на его написание. Исправление ошибки часто означает
прочитать тысячу строк, чтобы изменить три. Первый вклад в open source (Урок 24.4) начинается с понимания кода,
который другие люди писали годами. Новая работа начинается с кодовой базы, которая гораздо больше всего, что было
в этом курсе.

Ошибка, которую делает каждый новичок, — читать большой проект как книгу: открыть первый файл, прочитать его
сверху вниз, потом следующий. Через час вы ничего не помните, потому что ничто не связало строки с целью.

Опытные инженеры читают с **вопросом**. «Как эта программа решает, что файл двоичный?» — хороший вопрос: у него
есть ответ, и ответ живёт в небольшой части кода. Вы следуете за вопросом по коду и игнорируете всё остальное.
После нескольких вопросов весь проект начинает складываться в картину.

## 2. Метод из шести шагов

| Шаг | Что вы делаете | Инструменты |
|---|---|---|
| 1. Задать вопрос | Запишите один конкретный вопрос в заметки. | ваш журнал |
| 2. Прочитать документы | README, руководства, CONTRIBUTING, заметки об архитектуре, CHANGELOG. | текстовый редактор |
| 3. Нарисовать карту | Перечислите крейты или модули, что делает каждый и какого он размера. | `Cargo.toml`, ваша программа `loc` |
| 4. Найти точки входа | `main` для программы; публичный API каждого библиотечного крейта. | `cargo doc`, RustRover |
| 5. Проследить один путь | Ищите имена, переходите к определениям, читайте тесты этой части. | `git grep`, «Go to Definition» |
| 6. Запустить и изменить | Соберите, запустите тесты, добавьте печать или тест, посмотрите, что изменится. | `cargo test`, ВМ |

Метод работает благодаря двум привычкам. **Пишите, пока читаете**: каждый понятый файл получает одну строку в карте
кода. И **останавливайтесь, когда вопрос решён**: за следующим вопросом вы вернётесь.

> 💡 **Совет:** в RustRover «Go to Definition» (Cmd+B или Cmd+щелчок) и «Find Usages» (Option+F7) делают одним
> щелчком то, что `git grep` делает в терминале. Пользуйтесь и терминальным способом: он работает везде, в том
> числе на сервере или на странице код-ревью.

## 3. Настоящий проект: ripgrep

ripgrep (команда `rg`) — инструмент поиска, написанный на Rust. Он ищет текстовый шаблон в папках и пропускает файлы,
которые игнорирует Git. Это хорошая первая кодовая база: средний размер, отличная документация, разбиение на небольшие
крейты и много пользователей.

Клонируйте его рядом с папкой курса, а не внутри неё (репозиторий внутри другого репозитория сбивает Git с толку).
**Частичный клон** с `--filter=blob:none` сохраняет всю историю, но загружает содержимое файлов только по необходимости,
а `git switch --detach 15.2.0` переключает на выпуск, который использует этот урок:

```bash
mkdir -p ~/projects/oss
cd ~/projects/oss
git clone --filter=blob:none https://github.com/BurntSushi/ripgrep.git
cd ripgrep
git switch --detach 15.2.0
git log --oneline -1
```

```text
e89fff8 15.2.0
```

Когда вы будете это читать, может выйти более новый выпуск. Для урока оставайтесь на `15.2.0`, чтобы имена файлов
и номера строк совпадали; потом переключитесь на `master` командой `git switch master`.

> 🔒 **Безопасность:** сборка проекта на Rust запускает его код на вашей машине: скрипты `build.rs` и процедурные
> макросы выполняются во время `cargo build`, а тесты — во время `cargo test`. Читать файлы безопасно; **собирать
> и запускать** незнакомый проект нужно в вашей ВМ `dev` или в контейнере и никогда от имени root.

## 4. Сначала прочитайте документы

Выполните `ls` в верхней папке. Рядом с `Cargo.toml` и папками `crates`, `tests`, `fuzz` и `scripts` вы увидите
несколько Markdown-файлов и три файла лицензий.

Markdown-файлы написаны для вас. В этом проекте `README.md` объясняет, что делает ripgrep, `GUIDE.md` — руководство
пользователя, `FAQ.md` отвечает на частые вопросы, а `CHANGELOG.md` перечисляет изменения каждого выпуска.
`CONTRIBUTING.md` очень короткий: он отсылает к `AI_POLICY.md`. Эта политика разрешает инструменты ИИ для написания
кода, но участники должны понимать и объяснять свои изменения **своими словами**, комментарии для мейнтейнеров должен
писать человек, а полностью автоматическим ИИ-агентам участвовать нельзя. Таких правил становится всё больше; Урок 24.4
вернётся к ним.

`COPYING`, `LICENSE-MIT` и `UNLICENSE` — лицензии: ripgrep предлагается под «Unlicense OR MIT», та же идея, что
и двойная лицензия из Урока 24.2.

## 5. Нарисуйте карту

Верхний `Cargo.toml` описывает рабочее пространство. Важнее всего две части. Таблица `[[bin]]` называет программу
и её точку входа:

```toml
[[bin]]
bench = false
path = "crates/core/main.rs"
name = "rg"
```

а `[workspace] members = [...]` перечисляет девять папок внутри `crates/`.

Значит, программа `rg` начинается в `crates/core/main.rs`, а работа разделена на девять библиотечных крейтов. Строка
`name = ...` в собственном `Cargo.toml` каждого крейта даёт имя пакета; вместе с `README.md` каждого крейта получается
такая карта:

| Папка | Пакет | Что делает |
|---|---|---|
| `crates/core` | `ripgrep` (программа `rg`) | Флаги командной строки и «клей», который соединяет остальные крейты. |
| `crates/ignore` | `ignore` | Параллельно обходит каталоги и применяет правила `.gitignore`. |
| `crates/globset` | `globset` | Сопоставляет имена файлов с glob-шаблонами вроде `*.rs`. |
| `crates/searcher` | `grep-searcher` | Читает файлы, делит их на строки, обнаруживает двоичные данные. |
| `crates/matcher` | `grep-matcher` | Трейт, который реализует каждый сопоставитель шаблонов. |
| `crates/regex` / `crates/pcre2` | `grep-regex` / `grep-pcre2` | Две реализации этого трейта. |
| `crates/printer` | `grep-printer` | Форматирует результаты: обычный вывод, машиночитаемый формат, сводки. |
| `crates/cli` | `grep-cli` | Помощники для терминалов и программ командной строки. |
| `crates/grep` | `grep` | Один крейт, который реэкспортирует крейты `grep-*`. |

Насколько велики части? Подсчёт строк показывает, где живёт основная логика. Добавьте небольшую программу в учебный
пакет, `code/p24-career/src/bin/loc.rs`:

```rust
// Lists the biggest Rust files under a folder: a quick map of a large codebase.
use std::fs;
use std::path::{Path, PathBuf};

/// Counts lines that are not blank and are not only a comment (`//`, `///` or `//!`).
fn code_lines(text: &str) -> usize {
    text.lines()
        .map(str::trim)
        .filter(|line| !line.is_empty() && !line.starts_with("//"))
        .count()
}

/// Collects every `.rs` file under `dir`. Skips `target` and hidden folders such as `.git`.
fn rust_files(dir: &Path, out: &mut Vec<PathBuf>) -> std::io::Result<()> {
    for entry in fs::read_dir(dir)? {
        let path = entry?.path();
        let name = path.file_name().and_then(|n| n.to_str()).unwrap_or("");
        if path.is_dir() {
            if name != "target" && !name.starts_with('.') {
                rust_files(&path, out)?;
            }
        } else if name.ends_with(".rs") {
            out.push(path);
        }
    }
    Ok(())
}

fn main() {
    let root = std::env::args().nth(1).unwrap_or_else(|| ".".to_string());
    let mut files = Vec::new();
    if let Err(err) = rust_files(Path::new(&root), &mut files) {
        eprintln!("cannot read {root}: {err}");
        std::process::exit(2);
    }

    let mut sizes: Vec<(usize, PathBuf)> = Vec::new();
    for path in files {
        if let Ok(text) = fs::read_to_string(&path) {
            sizes.push((code_lines(&text), path));
        }
    }
    // Biggest first; equal sizes in path order, so the output is stable.
    sizes.sort_by(|a, b| b.0.cmp(&a.0).then(a.1.cmp(&b.1)));

    let total: usize = sizes.iter().map(|(lines, _)| lines).sum();
    println!("{} Rust files, {total} code lines", sizes.len());
    for (lines, path) in sizes.iter().take(10) {
        let shown = path.strip_prefix(&root).unwrap_or(path);
        println!("{lines:>6}  {}", shown.display());
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn blank_and_comment_lines_do_not_count() {
        let text = "//! crate docs\n\n/// docs\nfn main() {\n    // note\n    run();\n}\n";
        assert_eq!(code_lines(text), 3);
    }

    #[test]
    fn finds_rust_files_and_skips_target() {
        let dir = std::env::temp_dir().join(format!("loc-test-{}", std::process::id()));
        fs::create_dir_all(dir.join("src")).unwrap();
        fs::create_dir_all(dir.join("target")).unwrap();
        fs::write(dir.join("src/lib.rs"), "fn a() {}\n").unwrap();
        fs::write(dir.join("target/gen.rs"), "fn b() {}\n").unwrap();
        fs::write(dir.join("notes.txt"), "not code\n").unwrap();

        let mut files = Vec::new();
        rust_files(&dir, &mut files).unwrap();
        fs::remove_dir_all(&dir).unwrap();

        assert_eq!(files, vec![dir.join("src/lib.rs")]);
    }
}
```

**Что происходит:**

1. `code_lines` обрезает пробелы в каждой строке и оставляет её, только если она не пустая и не начинается с `//`.
   Эта одна проверка убирает обычные комментарии, doc-комментарии (`///`) и документацию крейта (`//!`). Подсчёт
   грубый, но для карты его достаточно: строка с кодом и комментарием всё равно считается кодом.
2. `rust_files` рекурсивна (Урок 07.7): для папки она вызывает саму себя, для файла `.rs` сохраняет путь.
   `fs::read_dir` перечисляет одну папку; `?` передаёт любую ошибку I/O вызывающему коду.
3. Папки с именем `target` (результаты сборки) или начинающиеся с `.` (например, `.git`) пропускаются; это не исходный код.
4. `main` берёт папку из первого аргумента (или `.`), считает каждый файл, сортирует от больших к меньшим и печатает
   общий итог и первую десятку. `strip_prefix` укорачивает пути, убирая корневую папку.
5. Второй тест строит крошечное дерево папок во временной папке системы, с ID процесса в имени, чтобы параллельные
   запуски тестов не мешали друг другу, и удаляет его до проверки результата.

Запустите тесты, затем направьте программу на крейты ripgrep:

```bash
cd ~/projects/rust/code/p24-career
cargo test --bin loc
cargo run -q --bin loc -- ~/projects/oss/ripgrep/crates
```

```text
running 2 tests
test tests::blank_and_comment_lines_do_not_count ... ok
test tests::finds_rust_files_and_skips_target ... ok

test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

```text
88 Rust files, 34332 code lines
  6703  core/flags/defs.rs
  3220  printer/src/standard.rs
  1817  ignore/src/walk.rs
  1340  globset/src/glob.rs
  1306  searcher/src/searcher/glue.rs
  1184  ignore/src/dir.rs
  1075  core/flags/hiargs.rs
   995  globset/src/lib.rs
   779  matcher/src/lib.rs
   764  printer/src/hyperlink/mod.rs
```

Около 34 000 строк кода (плюс много строк комментариев) в 88 файлах: слишком много, чтобы прочитать сразу, но достаточно
мало, чтобы нанести на карту. Самый большой файл, `core/flags/defs.rs`, определяет все флаги командной строки; он длинный,
потому что флагов много, а не потому, что он сложный. Теперь можно нарисовать поток данных:

```text
 rg (crates/core) ── parses flags ──> HiArgs (the settings for one run)
        │
        ├── ignore::WalkBuilder ──> walks folders, applies .gitignore ──> files
        │
        └── for each file: grep-searcher ──> lines ──> grep-regex (match?) ──> grep-printer ──> terminal
```

> ⚡ **Производительность:** карта размеров указывает и на горячий код. В инструменте поиска работа на каждый байт идёт
> в searcher и matcher; медленный разбор флагов не имел бы значения, а медленное деление на строки — имел бы.

## 6. Проследите один вопрос: как ripgrep обнаруживает двоичные файлы?

Вы могли заметить, что `rg` обычно не печатает строки из картинок или скомпилированных программ. Давайте выясним, как это
устроено. Начните там, где начинается программа, в `crates/core`, и поищите слово «binary» в именах типов:

```bash
cd ~/projects/oss/ripgrep
git grep -n "BinaryDetection::quit" crates/core
```

```text
crates/core/flags/hiargs.rs:1172:            grep::searcher::BinaryDetection::quit(b'\x00')
crates/core/search.rs:135:    /// `grep::searcher::BinaryDetection::quit` if we want to skip binary files
crates/core/search.rs:151:    /// `grep::searcher::BinaryDetection::quit`, since we never want to
```

Откройте `crates/core/flags/hiargs.rs` около строки 1160. Функция `BinaryDetection::from_low_args` выбирает две настройки:
**явную** (explicit) для файлов, которые вы назвали в командной строке, и **неявную** (implicit) для файлов, которые `rg`
находит при обходе папок. По умолчанию неявная — `quit(b'\x00')`, а явная — `convert(b'\x00')`. Doc-комментарии в
`crates/core/search.rs` (строки 130–160) объясняют почему: найденные при обходе файлы можно пропустить, если они выглядят
двоичными, но файл, который вы назвали сами, никогда не пропускается молча.

Теперь спуститесь на уровень ниже, в библиотеку, которая делает работу:

```bash
git grep -n "pub fn quit" crates/searcher
git grep -n "BinaryDetection::Quit(byte)" crates/searcher
```

```text
crates/searcher/src/searcher/mod.rs:79:    pub fn quit(binary_byte: u8) -> BinaryDetection {
crates/searcher/src/searcher/mod.rs:102:    pub fn quit_byte(&self) -> Option<u8> {
crates/searcher/src/line_buffer.rs:437:                BinaryDetection::Quit(byte) => {
```

Длинный doc-комментарий над строкой 79 файла `searcher/mod.rs` описывает **эвристику**: данные, содержащие заданный байт,
считаются двоичными. В `line_buffer.rs` около строки 437 находится место, где это происходит: каждый раз, когда буфер
заполняется из файла, новые байты проверяются на этот байт; если он найден, буфер заканчивается прямо перед ним и поиск
останавливается, как будто файл закончился там.

Наконец, прочитайте модульный тест этого кода; тесты — самая точная документация. Вот один из того же файла (около
строки 795):

```rust
    #[test]
    fn buffer_binary_quit1() {
        let bytes = "homer\nli\x00sa\nmaggie\n";
        let mut linebuf = LineBufferBuilder::new()
            .binary_detection(BinaryDetection::Quit(b'\x00'))
            .build();
        let mut rdr = LineBufferReader::new(bytes.as_bytes(), &mut linebuf);

        assert!(rdr.buffer().is_empty());

        assert!(rdr.fill().unwrap());
        assert_eq!(rdr.bstr(), "homer\nli");
        rdr.consume_all();

        assert!(!rdr.fill().unwrap());
        assert_eq!(rdr.absolute_byte_offset(), 8);
        assert_eq!(rdr.binary_byte_offset(), Some(8));
    }
```

Во входных данных после `li` стоит байт NUL. Тест показывает, что читатель возвращает только `"homer\nli"` и сообщает
о двоичном байте на смещении 8. **Ответ на вопрос:** ripgrep считает данные двоичными, если в них есть байт NUL. Для файлов,
найденных при обходе папок, он прекращает чтение на первом NUL; для файлов, которые вы назвали сами, он заменяет байты NUL
и продолжает поиск. Запишите это в карту кода вместе с именами четырёх файлов.

## 7. Спросите историю «почему»

Код говорит, *что* происходит; история говорит, *почему*. Поскольку у частичного клона есть вся история, можно спросить
Git о любом файле: `git log --oneline -- crates/searcher/src/line_buffer.rs` перечисляет коммиты, которые его меняли,
начиная с самого нового.

`git blame` отвечает на вопрос «какой коммит последним изменил эту строку?». Параметр `-s` скрывает имена авторов и даты,
а `-L 6,6` ограничивает вывод строкой 6:

```bash
git blame -s -L 6,6 crates/searcher/src/line_buffer.rs
```

```text
1c3eebef 6) pub(crate) const DEFAULT_BUFFER_CAPACITY: usize = 64 * (1 << 10); // 64 KB
```

Затем `git show 1c3eebef` печатает сообщение коммита и само изменение. В хороших проектах сообщения объясняют причину,
а многие сообщения коммитов ссылаются на issue с полным обсуждением.

## 8. Запустите и измените (в ВМ)

Чтение даёт теорию; запуск кода её проверяет. В вашей ВМ `dev` (или в Linux-контейнере) клонируйте тот же тег и запустите
тесты того крейта, который вы изучали. Имена тестов работают как фильтры, поэтому эта команда запускает только тесты
обнаружения двоичных данных в буфере строк:

```bash
cargo test -p grep-searcher buffer_binary_quit
```

Пример вывода (у вас будет похожий; первая сборка загружает и компилирует зависимости):

```text
running 5 tests
test line_buffer::tests::buffer_binary_quit1 ... ok
test line_buffer::tests::buffer_binary_quit2 ... ok
test line_buffer::tests::buffer_binary_quit3 ... ok
test line_buffer::tests::buffer_binary_quit4 ... ok
test line_buffer::tests::buffer_binary_quit5 ... ok
```

Затем **намеренно что-нибудь измените** в своей локальной копии: замените `b'\x00'` в `buffer_binary_quit1` на `b'\x01'`,
снова запустите тест и прочитайте, как он падает. Когда закончите, отмените изменение командой `git restore .`. Так вы
убеждаетесь, что действительно поняли кусок кода: предсказываете, что сделает изменение, и проверяете.

> 🧪 **Тестирование:** тест, который вы сломали намеренно, учит сразу двум вещам: что делает код и как выглядит вывод
> тестов проекта. Оба знания понадобятся, когда вы отправите первый pull request.

## Частые ошибки

**Чтение без вопроса.** Открыть `main.rs` и читать «всё подряд» — путь в никуда. Сначала запишите вопрос; остановитесь,
когда он решён.

**Попытка понять каждую строку.** Вам нужны только строки на вашем пути. Остальное пометьте в карте кода как «позже».

**Мелкий клон, когда нужна история.** `git clone --depth 1` загружает только самый новый коммит. Тогда `git log` показывает
один коммит, а `git blame` приписывает ему всё. Используйте вместо этого `--filter=blob:none`: клон маленький и с историей.

**Бездумная сборка незнакомого проекта на Mac от вашего обычного пользователя.** Скрипты сборки и тесты запускают
настоящий код. Для всего, что вы не прочитали, используйте ВМ.

## Попробуйте сами

1. Создайте `notes/code-map-ripgrep.md` с таблицей-картой из раздела 5, своими словами.
2. Добавьте ответ на вопрос о двоичных файлах (раздел 6) с четырьмя файлами и номерами строк.
3. Выберите новый вопрос, например «Где ripgrep читает файлы `.gitignore`?», и ответьте на него по методу. Подсказка:
   начните с `git grep -n "gitignore" crates/ignore/src | head`.
4. В ВМ запустите тесты изученного крейта и намеренно сломайте один тест, как в разделе 8.

## Упражнения

### Упражнение 1 — Размер по крейтам (средне)

После списка самых больших файлов напечатайте число строк кода по каждой папке верхнего уровня (для `ripgrep/crates` это
по крейтам), отсортированное по имени. Сначала напишите `top_folder(root: &Path, path: &Path) -> String` и тест для неё.
Файл, лежащий прямо в `root`, относится к папке `"."`.

<details>
<summary>Подсказка</summary>

`path.strip_prefix(root)` даёт относительный путь; `.components()` делит его на части. Если частей хотя бы две, первая —
это папка. `BTreeMap<String, usize>` хранит папки отсортированными.

</details>

<details>
<summary>Ответ</summary>

```rust
/// Returns the first folder of `path` below `root`, or "." for files directly in `root`.
fn top_folder(root: &Path, path: &Path) -> String {
    let relative = path.strip_prefix(root).unwrap_or(path);
    let mut parts = relative.components();
    match (parts.next(), parts.next()) {
        (Some(first), Some(_)) => first.as_os_str().to_string_lossy().into_owned(),
        _ => ".".to_string(),
    }
}

#[test]
fn top_folder_is_the_first_part_below_root() {
    let root = Path::new("/r");
    assert_eq!(top_folder(root, Path::new("/r/searcher/src/lib.rs")), "searcher");
    assert_eq!(top_folder(root, Path::new("/r/build.rs")), ".");
}
```

В конце `main`:

```rust
    let mut per_folder: std::collections::BTreeMap<String, usize> = Default::default();
    for (lines, path) in &sizes {
        *per_folder
            .entry(top_folder(Path::new(&root), path))
            .or_default() += lines;
    }
    println!("per folder:");
    for (folder, lines) in per_folder {
        println!("{lines:>6}  {folder}");
    }
```

Для `~/projects/oss/ripgrep/crates` новая часть вывода такая:

```text
per folder:
  1121  cli
 10352  core
  2656  globset
    81  grep
  5339  ignore
  1329  matcher
   320  pcre2
  6631  printer
  1865  regex
  4638  searcher
```

Почему это работает: `entry(...).or_default()` вставляет 0 для новой папки и возвращает изменяемую ссылку на счётчик.
Числа показывают форму проекта: флаги и «клей» (`core`), вывод, обход каталогов и searcher — крупные части; `grep` — лишь
фасад.

</details>

### Упражнение 2 — Скрытые файлы (средне, чтение)

По умолчанию `rg` не ищет в скрытых файлах. Как в macOS и Linux ripgrep решает, что файл скрытый, и где этот код?
Используйте метод; запишите команды, которые вы выполняли.

<details>
<summary>Подсказка</summary>

Обход каталогов живёт в `crates/ignore`. Ищите функцию, имя которой начинается с `is_hidden`.

</details>

<details>
<summary>Ответ</summary>

```bash
git grep -n "fn is_hidden" crates/ignore/src
```

```text
crates/ignore/src/pathutil.rs:11:pub(crate) fn is_hidden(dent: &DirEntry) -> bool {
crates/ignore/src/pathutil.rs:28:pub(crate) fn is_hidden(dent: &DirEntry) -> bool {
crates/ignore/src/pathutil.rs:51:pub(crate) fn is_hidden(dent: &DirEntry) -> bool {
```

Есть три версии, выбираемые при компиляции через `#[cfg(unix)]`, `#[cfg(windows)]` и `#[cfg(not(any(unix, windows)))]`.
Версия для Unix (macOS и Linux) возвращает true, если имя файла начинается с `.`. Версия для Windows дополнительно проверяет
атрибут файловой системы «скрытый».

Почему это работает: `#[cfg(...)]` — тот же атрибут, что и `#[cfg(test)]` из Урока 02.8. `#[cfg(unix)]` оставляет элемент,
только когда программа собирается для Unix-подобной системы, поэтому в каждой сборке ровно одна из трёх функций, и у одного
имени может быть своё тело для каждой операционной системы.

</details>

### Упражнение 3 — Карта вашего итогового проекта (легко)

Запустите `loc` на своём итоговом проекте (или любом проекте курса) и напишите карту кода из трёх строк: самая большая часть,
часть с самой важной логикой и одна часть, которую вы прочитали бы первой как новичок.

<details>
<summary>Подсказка</summary>

Самый большой файл не всегда самый важный; смотрите на имена и на то, что вызывает `main`.

</details>

<details>
<summary>Ответ</summary>

Единственного ответа нет. Хорошая карта называет файлы или модули, в нескольких словах говорит, что делает каждый,
и приводит причину, например: «`src/wal.rs` — самый важный: каждая запись сначала проходит через него».

</details>

## Вопросы для самопроверки

1. Почему чтение кодовой базы начинают с вопроса?

<details>
<summary>Ответ</summary>

Вопрос даёт путь по коду и точку, где можно остановиться. Без него вы читаете строки, которые ни с чем не связаны,
и забываете их.

</details>

2. Чем `git clone --depth 1` отличается от `git clone --filter=blob:none`?

<details>
<summary>Ответ</summary>

`--depth 1` загружает только самый новый коммит, поэтому истории нет. `--filter=blob:none` загружает все коммиты и списки
папок, а содержимое файлов — по требованию, поэтому `git log` и `git blame` работают, а клон остаётся маленьким.

</details>

3. Почему незнакомый проект нужно собирать и тестировать в ВМ или контейнере?

<details>
<summary>Ответ</summary>

`cargo build` запускает скрипты сборки и процедурные макросы, а `cargo test` — код тестов. Всё это код незнакомых вам
людей, который работает с правами вашего пользователя.

</details>

4. Почему модульные тесты — хорошая документация при чтении кода?

<details>
<summary>Ответ</summary>

Они показывают точные входные данные и точные ожидаемые результаты и проверяются при каждой сборке, поэтому не могут
незаметно устареть, как комментарии.

</details>

## Итоги

- Читайте большие кодовые базы с вопросом и останавливайтесь, когда он решён.
- Метод: вопрос, документы, карта, точки входа, прослеживание одного пути, запуск и изменение.
- Члены рабочего пространства в `Cargo.toml` и подсчёт размеров (`loc`) дают первую карту; README каждого крейта объясняет
  его часть.
- `git grep` находит имена, RustRover переходит к определениям, модульные тесты показывают точное поведение.
- `git log -- <file>` и `git blame` объясняют, почему код выглядит так, как выглядит; чтобы сохранить историю, используйте
  частичный клон.
- Собирайте и запускайте незнакомый код только в ВМ или контейнере и держите свои эксперименты локально.

## Источники и что почитать

- [ripgrep on GitHub](https://github.com/BurntSushi/ripgrep) — проект из этого урока с руководством пользователя, ответами
  на частые вопросы и README крейтов.
- [Git — git grep](https://git-scm.com/docs/git-grep), [git blame](https://git-scm.com/docs/git-blame) и
  [git clone](https://git-scm.com/docs/git-clone) — справочные страницы, включая `--filter` для частичных клонов.
- [The Cargo Book — Workspaces](https://doc.rust-lang.org/cargo/reference/workspaces.html) — как объявляются члены
  рабочего пространства.
- [The Cargo Book — cargo doc](https://doc.rust-lang.org/cargo/commands/cargo-doc.html) — локальная сборка документации
  API любого проекта.
- [RustRover — Source code navigation](https://www.jetbrains.com/help/rust/navigating-through-the-source-code.html) —
  «Go to Definition», «Find Usages» и другие инструменты навигации.
- [Rust Compiler Development Guide](https://rustc-dev-guide.rust-lang.org/) — руководство по одной из крупнейших кодовых
  баз на Rust и хороший пример того, как большие проекты описывают себя.

---

[🏠 Начало](../START-HERE.md) · [← 24.2 Ваше портфолио](02-building-your-portfolio.md) · [📋 Этап 24](README.md) · [24.4 Ваш первый вклад в open source →](04-your-first-open-source-contribution.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
