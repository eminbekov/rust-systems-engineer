<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 05.6 Тестирование подробно](06-testing-in-depth.md) · [📋 Этап 05](README.md) · [05.8 Cargo: зависимости и версии →](08-cargo-dependencies-and-versions.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/05-rust-intermediate/07-documentation.md) <!-- nav -->

# Урок 05.7 — Документация

> **Этап 05 — Rust: средний уровень** · Урок 7 из 14 · ⏱ около 3 часов (2 занятия)

## Что вы узнаете

- Три слоя документации: комментарии для сопровождающих, документационные комментарии для пользователей и файлы проекта (README, CHANGELOG).
- Как писать документационные комментарии `///` и `//!` с Markdown и стандартными разделами (`# Examples`, `# Errors`, `# Panics`).
- Как документационные тесты превращают ваши примеры в тесты, которые не могут устареть.
- Как создавать и читать HTML-документацию (HTML, HyperText Markup Language — язык веб-страниц) через `cargo doc` и как находить недостающую документацию и битые ссылки.
- Как документировать проект в Git: README, журнал изменений, теги версий и чем помогают `git log` и `git blame`.

## Перед началом

- Вы прошли [Урок 05.6 — Тестирование подробно](06-testing-in-depth.md) и у вас есть библиотечный пакет `code/p05-textkit` с `slugify` и `parse_duration`.
- Вы знаете модули и `pub` ([Урок 04.6](../04-structs-enums-collections-errors/06-modules-crates-and-packages.md)), `Result` ([Урок 04.10](../04-structs-enums-collections-errors/10-error-handling.md)) и Markdown ([Урок 00.8](../00-getting-started/08-study-routine-notes-and-checkpoint.md)).
- Git: коммиты, ветки и удалённый репозиторий ([Этап 01](../01-git-and-version-control/03-your-first-repository-and-commits.md)).

## Новые слова

| Слово | Значение |
|---|---|
| **документация** (documentation) | Текст, объясняющий, для чего нужен код и как им пользоваться. |
| **документационный комментарий** (doc comment) | Комментарий, который становится частью сгенерированной документации. Начинается с `///` (для следующего элемента) или `//!` (для объемлющего элемента). |
| **rustdoc** | Инструмент из поставки Rust, превращающий документационные комментарии в HTML-страницы. Его запускает `cargo doc`. |
| **документационный тест** (doc test) | Блок кода на Rust внутри документационного комментария. `cargo test` компилирует и запускает его. |
| **ссылка внутри документации** (intra-doc link) | Ссылка вида ``[`Name`]``, указывающая на другой элемент вашего кода; rustdoc проверяет, что он существует. |
| **README** | Первый файл, который читают в проекте: что это, как пользоваться, как собрать. |
| **журнал изменений** (changelog) | Файл со списком изменений в каждой версии, предназначенный для пользователей. |
| **SemVer** (Semantic Versioning, семантическое версионирование) | Правило версий `MAJOR.MINOR.PATCH`: MAJOR увеличивают при несовместимых изменениях, MINOR — для новых возможностей, PATCH — для исправлений ошибок. |
| **тег Git** (Git tag) | Постоянное имя одного коммита, обычно релиза, например `v0.1.0`. |
| **публичный API** (Application Programming Interface, программный интерфейс) | Элементы с пометкой `pub`, которые может использовать другой код. Именно их нужно документировать. |
| **docs.rs** | Сайт, который собирает и хранит документацию каждого крейта, опубликованного на crates.io. |

## 1. Три слоя документации

Код отвечает на вопрос «как это работает?» для компьютера. Людям нужны ответы на другие вопросы, и у
каждого ответа своё место:

| Слой | Читатель | Вопрос | Где |
|---|---|---|---|
| комментарии `//` | следующий разработчик, который **меняет** код | *Почему* сделано именно так? | внутри тел функций |
| документационные комментарии `///` и `//!` | разработчик, который **использует** код | Что это делает, как вызвать, что может пойти не так? | над элементами `pub`, в начале крейта |
| файлы проекта | все, включая пользователей, которые не читают код | Что это? Как установить? Что изменилось? | `README.md`, `CHANGELOG.md`, история Git |

Чем больше из этого вы пишете, тем меньше зависите от умения помнить. Через полгода вы станете чужим
человеком для собственного кода.

## 2. Документационные комментарии

Строка, которая начинается с трёх косых черт `///`, — это **документационный комментарий**. Он документирует
элемент **прямо под ним**. Текст — Markdown (Урок 00.8): `` `code` ``, списки, полужирный шрифт, ссылки и так далее.

```rust
/// Turns a title into lowercase, dash-separated text that is safe to use in a URL.
///
/// Letters and digits are kept (letters become lowercase). Every run of other
/// characters becomes a single dash, and dashes at the start or the end are removed.
///
/// # Examples
///
/// ```
/// use p05_textkit::slugify;
///
/// assert_eq!(slugify("Hello World"), "hello-world");
/// assert_eq!(slugify("  Rust -- is... fun!  "), "rust-is-fun");
/// assert_eq!(slugify("!!!"), "");
/// ```
pub fn slugify(input: &str) -> String {
    // (body as in Lesson 05.6)
```

**Что происходит:**

1. **Первая строка** — краткое описание в одно предложение. Оно появляется отдельно в списках функций, поэтому
   сделайте его законченным предложением, начинающимся с глагола («Turns…», «Returns…», «Parses…») и
   заканчивающимся точкой.
2. После пустой строки `///` идёт более длинное объяснение: правила, особые случаи, единицы измерения.
3. Заголовок `# Examples` вводит блок кода. Заголовки Markdown в Rust пишутся через `#`, а **стандартные имена
   разделов — соглашение, известное каждому, кто читает Rust** (таблица ниже).
4. Блок в тройных обратных апострофах без указания языка компилируется как Rust **и запускается**: подробнее в разделе 3.

### Стандартные разделы

| Раздел | Когда писать | Содержание |
|---|---|---|
| `# Examples` | всегда, для каждой публичной функции | хотя бы один короткий запускаемый пример |
| `# Errors` | функция возвращает `Result` | когда и какие ошибки возвращаются |
| `# Panics` | функция может паниковать | точные условия |
| `# Safety` | функция `unsafe` (Урок 06.6) | что обязан гарантировать вызывающий |

Функция, возвращающая `Result`, документирует свои ошибки:

```rust
/// Reads a duration such as `"1h30m15s"` and returns the number of seconds.
///
/// The text is a list of parts. Each part is a whole number followed by one of
/// the units `h` (hours), `m` (minutes) or `s` (seconds). Spaces around the
/// whole text are ignored.
///
/// # Errors
///
/// Returns a [`ParseError`] if the text is empty, if a part has no number or no
/// unit, if the unit is unknown, or if the total does not fit into a `u64`.
///
/// # Examples
///
/// ```
/// use p05_textkit::{parse_duration, ParseError};
///
/// assert_eq!(parse_duration("1h30m15s"), Ok(5415));
/// assert_eq!(parse_duration("5x"), Err(ParseError::UnknownUnit('x')));
/// ```
pub fn parse_duration(input: &str) -> Result<u64, ParseError> {
```

а функция, которая может паниковать, документирует это (и тестирует):

```rust
/// Returns `part` as a whole-number percentage of `whole`, rounded down.
///
/// # Panics
///
/// Panics if `whole` is zero, because a percentage of nothing is undefined.
///
/// # Examples
///
/// ```
/// assert_eq!(p05_textkit::percent(1, 4), 25);
/// ```
///
/// The panic is part of the contract, so it is tested too:
///
/// ```should_panic
/// p05_textkit::percent(1, 0);
/// ```
pub fn percent(part: u32, whole: u32) -> u32 {
    assert!(whole != 0, "whole must not be zero");
    ((part as u64 * 100) / whole as u64) as u32
}
```

**Что происходит:** `assert!(cond, "message")` паникует, если `cond` ложно (Урок 05.6). Раздел `# Panics` превращает
это поведение в задокументированное обещание. Преобразования в `u64` предотвращают переполнение при вычислении
`part * 100` для больших `part`.

Документируйте также типы и каждый вариант перечисления, потому что компилятор предупредит вас, если этого не делать (Раздел 4):

```rust
/// The ways in which [`parse_duration`] can fail.
#[derive(Debug, PartialEq)]
pub enum ParseError {
    /// The text is empty or contains only spaces.
    Empty,
    /// A unit letter has no number in front of it, as in `"h"`.
    MissingNumber,
    /// The text ends with digits that have no unit, as in `"10"`.
    MissingUnit,
    /// The unit letter is not one of `h`, `m` or `s`.
    UnknownUnit(char),
    /// The result does not fit into a `u64`.
    TooLarge,
}
```

### `//!` документирует контейнер

`//!` документирует то, что **содержит** комментарий, а не следующий элемент. Его используют в самом начале
`src/lib.rs`, чтобы описать весь крейт (и так же в начале файла модуля):

```rust
//! # p05-textkit
//!
//! Small helpers for turning titles into URL-friendly text and for reading
//! durations such as `1h30m`.
//!
//! ```
//! use p05_textkit::{parse_duration, slugify};
//!
//! assert_eq!(slugify("Hello World"), "hello-world");
//! assert_eq!(parse_duration("1m30s"), Ok(90));
//! ```

#![warn(missing_docs)]
```

Это титульная страница вашей документации: для чего нужен крейт и первый пример. `#![warn(missing_docs)]` (с `!`) — атрибут
для всего крейта: он просит компилятор предупреждать о публичных элементах без документации.

> 📏 **Соглашение:** пишите документацию для **пользователей**: они знают Rust, но не знают ваш код. Говорите, что функция
> *делает*, а не как она устроена внутри. Описывайте единицы измерения (секунды? байты?), диапазоны и особые случаи
> (пустой ввод, ноль). Избегайте слов «просто» и «очевидно»: то, что очевидно вам, не очевидно им.

## 3. Документационные тесты: примеры, которые не могут устареть

У документации с примерами кода есть давняя проблема: код меняется, а примеры молча перестают работать. Rust решает
её так: **каждый блок кода Rust в документационном комментарии — это тест**. `cargo test` компилирует и запускает его в
разделе `Doc-tests`:

```text
   Doc-tests p05_textkit

running 6 tests
test src/lib.rs - (line 6) ... ok
test src/lib.rs - parse_duration (line 87) ... ok
test src/lib.rs - percent (line 134) ... ok
test src/lib.rs - parse_duration (line 77) ... ok
test src/lib.rs - slugify (line 22) ... ok
test src/lib.rs - percent (line 140) - should panic ... ok

test result: ok. 6 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

**Что происходит:**

1. Каждый блок кода становится маленькой программой. Rustdoc оборачивает ваши строки в `fn main() { … }`, компилирует её
   как *внешнего* пользователя вашего крейта (поэтому нужна строка `use p05_textkit::…`, как в интеграционных тестах из
   Урока 05.6), и запускает.
2. Упавший `assert_eq!` или паника проваливают документационный тест. Имена показывают, где находится каждый тест: файл,
   элемент и строка блока кода.
3. Запустить только документационные тесты можно через `cargo test --doc`.

Если пример неверен, падение указывает на него:

```rust
/// Doubles a number.
///
/// ```
/// assert_eq!(d2::double(2), 5);
/// ```
pub fn double(x: i32) -> i32 {
    x * 2
}
```

```text
---- src/lib.rs - double (line 6) stdout ----
Test executable failed (exit status: 101).

stderr:

thread 'main' panicked at .../doctest_bundle_2024.rs:6:1:
assertion `left == right` failed
  left: 4
 right: 5
```

**Что это значит:** документация обещала `5`, а код даёт `4`. Документационный тест поймал неправду раньше, чем
пользователь. (Здесь неверен пример; в реальной жизни, когда меняется поведение, упавший документационный тест
напоминает обновить документацию.)

### Скрытые строки и `?`

Строки, начинающиеся с `# ` (решётка и пробел), компилируются, но **не показываются** в готовой документации. Они прячут
шаблонный код, чтобы пример оставался читаемым. Типичное применение — функция `main`, возвращающая `Result`, что позволяет
использовать `?`:

```rust
/// ```
/// # fn main() -> Result<(), p05_textkit::ParseError> {
/// let seconds = p05_textkit::parse_duration("2m")?;
/// assert_eq!(seconds, 120);
/// # Ok(())
/// # }
/// ```
```

Читатель HTML-страницы видит только две средние строки; компилятор видит полную программу. (Если в примере нет собственной
`fn main`, rustdoc добавит её сам.)

### Атрибуты блока кода

Слово после открывающих обратных апострофов меняет то, что происходит с блоком:

| Пометка | Значение |
|---|---|
| ` ```no_run ` | скомпилировать, но не запускать (для кода, который запускает сервер, обращается к сети, работает бесконечно) |
| ` ```ignore ` | пропустить совсем; показывается как `ignored`. Применяйте редко, потому что ничто не проверяет такой код |
| ` ```should_panic ` | пример обязан паниковать (как `percent(1, 0)` выше) |
| ` ```compile_fail ` | пример обязан **не скомпилироваться** (чтобы показать, что компилятор отвергает) |
| ` ```text ` | вообще не Rust: обычный текст, вывод, команда оболочки |

```text
test src/lib.rs - serve_forever (line 17) - compile ... ok          <- no_run
test src/lib.rs - serve_forever (line 21) ... ignored                <- ignore
test src/lib.rs - serve_forever (line 25) - compile fail ... ok     <- compile_fail
```

> 🧪 **Тестирование:** документационный тест — это **тест вашего публичного API с точки зрения пользователя**, как
> интеграционный тест (Урок 05.6), но это ещё и пример, который читает пользователь. Делайте его коротким, показывайте
> самое частое использование первым и не повторяйте здесь каждый граничный случай: граничные случаи принадлежат обычным
> модульным тестам.

## 4. `cargo doc`: создание и проверка

`cargo doc` собирает HTML-документацию вашего крейта **и всех его зависимостей** в `target/doc/`. Самая полезная форма:

```bash
cargo doc --no-deps --open
```

- `--no-deps`: документировать только ваш крейт (намного быстрее).
- `--open`: открыть результат в браузере. Начальная страница — `target/doc/p05_textkit/index.html`.

```text
 Documenting p05-textkit v0.1.0 (...)
    Finished `dev` profile [unoptimized + debuginfo] target(s) in 0.07s
   Generated .../target/doc/p05_textkit/index.html
```

Вы увидите свой текст `//!` на титульной странице, затем список функций и типов, у каждого — краткая строка; страницы
подробностей показывают полный текст, разделы и примеры (с кнопкой «Run»). Поиск вверху работает без интернета. Это тот
же сайт, который docs.rs показывает для крейтов на crates.io. Документация стандартной библиотеки на doc.rust-lang.org
собрана тем же инструментом из таких же комментариев.

> 💡 **Совет:** `cargo doc --open` — хорошая привычка, прежде чем считать библиотеку «готовой»: прочитайте собственную
> документацию как посторонний. В RustRover можно также навести курсор на имя или выбрать *View → Quick Documentation*,
> чтобы увидеть его документационный комментарий, не выходя из редактора.

### Ссылки внутри документации

Ссылайтесь на другие элементы, записывая их имя в квадратных скобках и обратных апострофах, как в ``[`ParseError`]``. Rustdoc находит элемент,
превращает его в ссылку и **предупреждает, если элемента нет**. Можно использовать и пути: ``[`crate::slugify`]``, ``[`Vec::push`]``.

### Находим недостающую документацию и битые ссылки

С `#![warn(missing_docs)]` компилятор предупреждает о каждом публичном элементе без документационного комментария, а rustdoc
предупреждает о ссылках, которые никуда не ведут. Вот небольшой крейт с обеими проблемами:

```rust
#![warn(missing_docs)]
//! A tiny crate.

/// Doubles a number. See also [`triple_it`] and [`Missing`].
pub fn double(x: i32) -> i32 {
    x * 2
}

pub fn undocumented() {}
```

```text
warning: missing documentation for a function
  --> src/lib.rs:13:1
   |
13 | pub fn undocumented() {}
   | ^^^^^^^^^^^^^^^^^^^^^
   |
note: the lint level is defined here
  --> src/lib.rs:1:9
   |
 1 | #![warn(missing_docs)]
   |         ^^^^^^^^^^^^

warning: unresolved link to `triple_it`
 --> src/lib.rs:4:34
  |
4 | /// Doubles a number. See also [`triple_it`] and [`Missing`].
  |                                  ^^^^^^^^^ no item named `triple_it` in scope
  |
  = note: `#[warn(rustdoc::broken_intra_doc_links)]` on by default
```

**Что это значит:** первое предупреждение перечисляет публичную функцию, которую никто не задокументировал; остальные
показывают несуществующие имена (возможно, переименованные или с опечаткой). Исправьте их и запускайте `cargo doc`, пока он не
замолчит. Позже задача CI может запускать `RUSTDOCFLAGS="-D warnings" cargo doc --no-deps`, чтобы такие предупреждения
проваливали сборку.

## 5. Комментарии внутри кода: объясняйте *почему*

Документационные комментарии — для пользователей. Обычные комментарии `//` — для человека, который будет **менять** код.
Правило: **не повторяйте то, что говорит код; объясняйте то, что он сказать не может.**

```rust
// Bad: says what the code obviously does.
// add one to counter
counter += 1;

// Bad: the comment has become false.
// retry three times
for _ in 0..5 { /* ... */ }

// Good: says WHY.
// The server closes idle connections after 30 s, so we retry once with a fresh one.
```

Рекомендации:

- Предпочитайте **ясные имена** комментариям. Если для `let d = 86400;` нужен комментарий, назовите это `let seconds_per_day = 86_400;`.
- Объясняйте *почему*: причину неожиданного выбора, обход ошибки, правило из спецификации, единицы измерения.
- Отмечайте незавершённую работу через `TODO:` или `FIXME:` с причиной. Макрос Rust `todo!()` — скомпилированная версия той же идеи (Урок 05.6).
- **Не оставляйте закомментированный код.** Git помнит все старые версии (Этап 01): удалите код и сделайте коммит.
- 🔒 **Безопасность:** никогда не кладите пароли, токены или закрытые адреса в комментарии или примеры документации: документация публикуется, а история Git вечна.

## 6. Документация проекта и Git как документация

Проект — это больше, чем код. Картину дополняют три файла и история Git.

### README.md

README — первое, что читает посетитель (GitHub и GitLab показывают его на главной странице проекта). В минимальном полезном README есть:

1. название и **одно предложение** о том, для чего проект;
2. короткий пример использования;
3. как собрать, протестировать и запустить;
4. лицензия и куда обращаться с вопросами (когда проект публичный).

```markdown
# p05-textkit

Small helpers for turning titles into URL-friendly text and for reading
durations such as `1h30m`.

## Usage

(a short code example)

## Development

(cargo fmt --check && cargo clippy -- -D warnings && cargo test)
```

### CHANGELOG.md и SemVer

Журнал изменений рассказывает **пользователям**, что изменилось между версиями, человеческими словами (а не выгрузкой коммитов).
Распространённый формат — *Keep a Changelog*: сначала новейшая версия, раздел `Unreleased` сверху и группы `Added`, `Changed`, `Fixed`,
`Removed`.

```markdown
# Changelog

## [Unreleased]

### Added
- `percent`: a whole-number percentage.

## [0.1.0] - 2026-11-17

### Added
- `slugify`: turn a title into URL-friendly text.
- `parse_duration`: read durations such as `1h30m15s`.
```

Номера версий следуют **семантическому версионированию** `MAJOR.MINOR.PATCH`: `0.1.0` → `0.1.1` для исправления ошибки, `0.2.0` для новых
возможностей, `1.0.0`, когда вы обещаете стабильность API. До `1.0` может измениться что угодно; после неё *несовместимое изменение*
(удаление или изменение публичной функции) требует нового номера MAJOR. Урок 05.8 объясняет, как Cargo использует эти номера для выбора
версий зависимостей.

### Git: теги, `git log` и `git blame`

Когда вы публикуете версию, пометьте точный коммит аннотированным **тегом**:

```bash
git tag -a v0.1.0 -m "First release of p05-textkit"
git tag
git show v0.1.0 --stat
```

```text
v0.1.0
tag v0.1.0
Tagger: Student <s@example.com>
Date:   Tue Nov 17 16:51:03 2026 +0500

First release of p05-textkit
```

`-a` создаёт *аннотированный* тег (он хранит, кто его сделал, когда и сообщение). `git tag` показывает список тегов; `git show` показывает тег и коммит.
Чтобы отправить теги на удалённый репозиторий, нужна команда `git push --tags` (Урок 01.8). Теперь Git может ответить на вопрос «что изменилось
с релиза?» — это сырьё для следующей записи в журнале изменений:

```bash
git log --oneline v0.1.0..HEAD
git log --oneline --decorate | head -4
```

```text
903adfc docs(p05): explain panics in percent
48ab9f0 feat(p05): add percent
903adfc (HEAD -> main) docs(p05): explain panics in percent
48ab9f0 feat(p05): add percent
ab9558f (tag: v0.1.0) feat(p05): slugify and parse_duration with documentation
```

`v0.1.0..HEAD` означает «коммиты после `v0.1.0` и до текущего». Поскольку сообщения следуют формату Conventional Commits (Урок 05.2: `feat`,
`fix`, `docs`…), по ним можно писать журнал изменений: каждый `feat:` — это *Added*, каждый `fix:` — *Fixed*.

**Сообщения коммитов — тоже документация.** Хорошее сообщение объясняет, *почему* сделано изменение: в первой строке до примерно 70 символов, затем
(после пустой строки) подробности. Через полгода `git log` и `git blame` — единственное объяснение, почему строка существует. `git blame`
показывает, кто последним менял каждую строку и в каком коммите:

```bash
git blame -L 1,3 code/p05-textkit/README.md
```

```text
^ab9558f (Student 2026-11-17 16:51:03 +0500 1) # p05-textkit
^ab9558f (Student 2026-11-17 16:51:03 +0500 2) 
^ab9558f (Student 2026-11-17 16:51:03 +0500 3) Small helpers for turning titles into URL-f
```

(`-L 1,3` ограничивает вывод строками с 1 по 3. Хеш коммита в начале строки ведёт к `git show <hash>`, который показывает полное сообщение коммита
и его изменение. Несмотря на название («blame» — «винить»), команда нужна, чтобы понять, а не чтобы искать, кого отругать.)

## Частые ошибки

**1. Документирование «что» вместо «почему» или «как пользоваться».** «Увеличивает счётчик» для `fn increment()` никому не помогает. Скажите, когда
вызывать функцию и что она гарантирует.

**2. Примеры, которые никогда не запускаются.** Повсеместное ` ```ignore ` лишает смысла всю затею. Предпочитайте ` ```no_run ` для кода, который нельзя запустить в тестах.

**3. Забытый `use` в документационном тесте.** Тест — внешний пользователь вашего крейта, поэтому одно `slugify("x")` падает с `cannot find function`.
Напишите `use p05_textkit::slugify;` (или спрячьте его через `# use …;`).

**4. Использование `?` в документационном тесте без `main`, возвращающего `Result`.** Добавьте скрытые строки `# fn main() -> Result<(), …> { … # Ok(()) # }`.

**5. Битые ссылки внутри документации** после переименования элемента. `cargo doc` предупреждает: читайте предупреждения.

**6. Комментарии, которые лгут.** Неверный комментарий хуже, чем отсутствие комментария. Меняя код, меняйте его комментарии в том же коммите.

**7. README, который может понять только автор.** Проверьте его: попросите друга (или себя, в чистой папке) выполнить его в точности.

## Попробуйте сами

В новой ветке полностью задокументируйте `code/p05-textkit` и выпустите версию 0.1.0.

1. Создайте ветку:

   ```bash
   cd ~/projects/rust
   git switch main
   git switch -c lesson/05-07-docs
   ```

2. В `code/p05-textkit/src/lib.rs` добавьте текст `//!` для крейта вместе с `#![warn(missing_docs)]`, комментарий `///` с `# Examples` для
   `slugify`, `# Errors` и `# Examples` для `parse_duration`, документацию для каждого варианта `ParseError` и функцию `percent` с
   `# Panics`. Пишите каждое так, как показано в разделе 2.

3. Запускайте проверки, пока все не станут чистыми:

   ```bash
   cd code/p05-textkit
   cargo test --doc
   cargo doc --no-deps --open
   cargo clippy -- -D warnings
   ```

   Браузер откроет документацию. Пройдитесь по ней: титульная страница, `slugify`, `ParseError` с вариантами и кнопка «Run» у примера. Затем
   **нарочно что-нибудь сломайте**: измените ожидаемое значение в примере, запустите `cargo test --doc` и прочитайте падение; добавьте
   недокументированную `pub fn` и прочитайте предупреждение `missing_docs`; исправьте обе проблемы.

4. Закоммитьте код, затем добавьте `README.md` и `CHANGELOG.md` в `code/p05-textkit`, как в разделе 6, и закоммитьте их отдельно:

   ```bash
   cd ~/projects/rust
   git add code/p05-textkit/src
   git commit -m "docs(p05): document the public API of textkit"
   git add code/p05-textkit/README.md code/p05-textkit/CHANGELOG.md
   git commit -m "docs(p05): add README and CHANGELOG"
   ```

5. Слейте в `main` и пометьте релиз тегом (тег отмечает коммит, который теперь является `HEAD` в `main`):

   ```bash
   git switch main
   git merge lesson/05-07-docs
   git tag -a v0.1.0 -m "First release of p05-textkit"
   git tag
   ```

6. Внесите ещё одно изменение (например, функцию `percent`), затем посмотрите, что произошло после релиза:
   `git log --oneline v0.1.0..HEAD`. Запишите это изменение в раздел `## [Unreleased]` файла `CHANGELOG.md` и закоммитьте.

Если вы пользуетесь GitHub или GitLab, отправьте и тег: `git push --tags`. Платформы показывают теги как релизы.

## Упражнения

### Упражнение 1 — Задокументируйте функцию (легко)

Напишите полный документационный комментарий (краткое описание в одну строку, короткое объяснение, `# Examples` с тремя проверками) для этой функции
и проверьте его через `cargo test --doc`. Замените `my_crate` на имя вашего крейта (с подчёркиваниями).

```rust
pub fn clamp_percent(value: i32) -> u8 {
    value.clamp(0, 100) as u8
}
```

<details>
<summary>Подсказка</summary>

Начните с глагола («Limits…»). Объясните, что происходит ниже 0 и выше 100. В примере `use my_crate::clamp_percent;` и проверьте значение внутри
диапазона, одно ниже и одно выше.

</details>

<details>
<summary>Ответ</summary>

```rust
/// Limits a number to the range 0 to 100 and returns it as a percentage.
///
/// Values below 0 become 0, and values above 100 become 100.
///
/// # Examples
///
/// ```
/// use my_crate::clamp_percent;
///
/// assert_eq!(clamp_percent(42), 42);
/// assert_eq!(clamp_percent(-5), 0);
/// assert_eq!(clamp_percent(150), 100);
/// ```
pub fn clamp_percent(value: i32) -> u8 {
    value.clamp(0, 100) as u8
}
```

`cargo test --doc` сообщает об одном успешном документационном тесте.

Почему это работает: краткое описание говорит, для чего функция, вторая строка называет особые случаи, а три проверки показывают обычный случай и обе границы.

</details>

### Упражнение 2 — Исправьте сломанный документационный тест (легко)

Этот документационный тест падает. Запустите его, прочитайте падение и решите, что неверно: **пример** или **код**. Исправьте.

```rust
/// Doubles a number.
///
/// ```
/// assert_eq!(my_crate::double(2), 5);
/// ```
pub fn double(x: i32) -> i32 {
    x * 2
}
```

<details>
<summary>Подсказка</summary>

Падение показывает `left: 4` и `right: 5`. `left` — реальный результат кода. Что верно: имя функции и тело или число 5 в примере?

</details>

<details>
<summary>Ответ</summary>

Код верен (удвоение 2 даёт 4); в примере опечатка:

```rust
/// Doubles a number.
///
/// ```
/// assert_eq!(my_crate::double(2), 4);
/// ```
pub fn double(x: i32) -> i32 {
    x * 2
}
```

Почему это работает: документационный тест падает всякий раз, когда пример и код расходятся; вы должны решить, что из них — истина. Если вы не уверены,
решают имя функции, её другие тесты и текст документации.

</details>

### Упражнение 3 — `# Errors` и `# Panics` (средне)

Напишите документацию, включая правильный стандартный раздел и документационный тест, который её подтверждает, для обеих функций:

```rust
pub fn divide(a: i32, b: i32) -> Result<i32, String> {
    if b == 0 {
        Err(String::from("division by zero"))
    } else {
        Ok(a / b)
    }
}

pub fn item_at(list: &[i32], index: usize) -> i32 {
    list[index]
}
```

<details>
<summary>Подсказка</summary>

`divide` возвращает `Result`, поэтому ей нужен `# Errors`, а пример может проверить `divide(7, 2) == Ok(3)` и `divide(1, 0).is_err()`. `item_at` паникует, когда
`index >= list.len()`, поэтому ей нужен `# Panics`, один обычный пример и второй блок кода с пометкой `should_panic`.

</details>

<details>
<summary>Ответ</summary>

```rust
/// Divides `a` by `b` and rounds toward zero.
///
/// # Errors
///
/// Returns an error message if `b` is zero.
///
/// # Examples
///
/// ```
/// use my_crate::divide;
///
/// assert_eq!(divide(7, 2), Ok(3));
/// assert!(divide(1, 0).is_err());
/// ```
pub fn divide(a: i32, b: i32) -> Result<i32, String> {
    if b == 0 {
        Err(String::from("division by zero"))
    } else {
        Ok(a / b)
    }
}

/// Returns the item at `index`.
///
/// # Panics
///
/// Panics if `index` is not smaller than `list.len()`.
///
/// # Examples
///
/// ```
/// assert_eq!(my_crate::item_at(&[10, 20, 30], 1), 20);
/// ```
///
/// ```should_panic
/// my_crate::item_at(&[10, 20, 30], 3);
/// ```
pub fn item_at(list: &[i32], index: usize) -> i32 {
    list[index]
}
```

В проверенном проекте `cargo test --doc` запустил пять документационных тестов (включая два для этих функций и один с пометкой `should panic`), и все прошли.

Почему это работает: `# Errors` говорит вызывающим, что нужно обработать, `# Panics` — чего избегать, а у каждого утверждения есть тест, поэтому оно не может незаметно стать неправдой.

</details>

### Упражнение 4 — Подготовьте релиз (средне)

Начиная с ветки из раздела «Попробуйте сами»: напишите `CHANGELOG.md` с разделом `Unreleased` и разделом `0.1.0`, пометьте релиз тегом, сделайте новый
коммит (`feat: …`) и покажите одной командой, какие коммиты появились после тега. Запишите команды.

<details>
<summary>Подсказка</summary>

Тег ставится через `git tag -a v0.1.0 -m "…"`. Коммиты после тега показывает `git log --oneline v0.1.0..HEAD`. Новая возможность записывается в `## [Unreleased]` / `### Added`.

</details>

<details>
<summary>Ответ</summary>

```markdown
# Changelog

All notable changes to this project are documented in this file.
The format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

## [Unreleased]

### Added
- `percent`: a whole-number percentage.

## [0.1.0] - 2026-11-17

### Added
- `slugify`: turn a title into URL-friendly text.
- `parse_duration`: read durations such as `1h30m15s`.
```

```bash
git add code/p05-textkit/CHANGELOG.md
git commit -m "docs(p05): add CHANGELOG"
git tag -a v0.1.0 -m "First release of p05-textkit"
# ...implement and commit a new feature...
git commit -am "feat(p05): add percent"
git log --oneline v0.1.0..HEAD
```

```text
48ab9f0 feat(p05): add percent
```

(Ваш хеш будет другим.) Почему это работает: тег фиксирует релиз в истории, `v0.1.0..HEAD` показывает только более поздние коммиты, а журнал изменений держит
раздел `Unreleased` готовым к следующей версии.

</details>

## Вопросы для самопроверки

1. Чем отличаются `//`, `///` и `//!`?

<details>
<summary>Ответ</summary>

`//` — обычный комментарий для тех, кто читает код. `///` — документационный комментарий для элемента прямо под ним. `//!` — документационный комментарий для объемлющего элемента (крейта или модуля), обычно в начале `lib.rs`.

</details>

2. Почему первая строка документационного комментария особенная?

<details>
<summary>Ответ</summary>

Она используется как краткое описание в списках элементов. Это должно быть описание в одно предложение, начинающееся с глагола и заканчивающееся точкой.

</details>

3. Что происходит с блоком кода ` ``` ` в документационном комментарии при запуске `cargo test`?

<details>
<summary>Ответ</summary>

Он компилируется и запускается как документационный тест — маленькая программа, использующая ваш крейт снаружи. Если он не компилируется или паникует, документационный тест падает.

</details>

4. Как показать в документации короткий пример, которому для компиляции на самом деле нужны `fn main` и `Ok(())`?

<details>
<summary>Ответ</summary>

Поместите эти строки в блок кода, но начните их с `# ` (решётка и пробел). Они компилируются, но скрыты в готовой документации.

</details>

5. Какой стандартный раздел писать для функции, возвращающей `Result`, а какой — для функции, которая может паниковать?

<details>
<summary>Ответ</summary>

`# Errors` для функции с `Result` и `# Panics` для функции, которая может паниковать (плюс `# Safety` для `unsafe`-функций).

</details>

6. Почему комментарии в коде должны объяснять *почему*, а не *что*?

<details>
<summary>Ответ</summary>

Код уже показывает, что происходит, а комментарий, повторяющий это, станет неверным при изменении кода. Причина решения (обход ошибки, правило, единица измерения) в коде не видна, поэтому сохранить её может только комментарий.

</details>

7. Что даёт тег `v0.1.0` и что показывает `git log --oneline v0.1.0..HEAD`?

<details>
<summary>Ответ</summary>

Тег — постоянное имя коммита релиза. Команда показывает коммиты, сделанные после этого релиза, то есть сырьё для следующей записи в журнале изменений.

</details>

## Итоги

- Три слоя документации: комментарии `//` объясняют *почему* сопровождающим, `///` и `//!` объясняют *что и как* пользователям, а README, CHANGELOG и история Git объясняют проект.
- Документационный комментарий начинается с краткого описания в одно предложение и использует Markdown и стандартные разделы `# Examples`, `# Errors`, `# Panics`, `# Safety`.
- Каждый блок кода Rust в документационном комментарии — это документационный тест, который запускает `cargo test`; прячьте шаблонный код через `# `, а при необходимости используйте `no_run`, `should_panic`, `compile_fail`.
- `cargo doc --no-deps --open` собирает и показывает HTML-документацию; `#![warn(missing_docs)]` и ссылки внутри документации `` [`Name`] `` поддерживают её полноту и правильность.
- Не комментируйте то, что говорит код; объясняйте почему. Никогда не оставляйте закомментированный код и не кладите секреты в документацию.
- Ведите README (что это, как пользоваться, как собрать) и CHANGELOG (формат Keep a Changelog), а версии нумеруйте по SemVer.
- `git tag -a`, `git log v0.1.0..HEAD` и `git blame` превращают Git в документацию: пишите сообщения коммитов для читателя из будущего.

## Источники и что почитать

- [The rustdoc book](https://doc.rust-lang.org/rustdoc/) — полное руководство по `rustdoc`, `cargo doc` и документационным комментариям.
- [The rustdoc book — How to write documentation](https://doc.rust-lang.org/rustdoc/how-to-write-documentation.html) — разделы, Markdown, скрытые строки и ссылки.
- [The rustdoc book — Documentation tests](https://doc.rust-lang.org/rustdoc/write-documentation/documentation-tests.html) — все атрибуты блоков кода.
- [The rustdoc book — Lints](https://doc.rust-lang.org/rustdoc/lints.html) — `broken_intra_doc_links` и другие предупреждения rustdoc.
- [Rust API Guidelines — Documentation](https://rust-lang.github.io/api-guidelines/documentation.html) — что содержит хорошая документация на Rust.
- [The Cargo Book — cargo doc](https://doc.rust-lang.org/cargo/commands/cargo-doc.html) — параметры `cargo doc`.
- [Keep a Changelog 1.1.0](https://keepachangelog.com/en/1.1.0/) и [Semantic Versioning 2.0.0](https://semver.org/) — формат журнала изменений и номера версий.
- [Документация Git — git tag](https://git-scm.com/docs/git-tag) и [git blame](https://git-scm.com/docs/git-blame).

---

[🏠 Начало](../START-HERE.md) · [← 05.6 Тестирование подробно](06-testing-in-depth.md) · [📋 Этап 05](README.md) · [05.8 Cargo: зависимости и версии →](08-cargo-dependencies-and-versions.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
