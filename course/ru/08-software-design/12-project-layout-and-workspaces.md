<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 08.11 Соглашения: имена, Clippy и Rust API Guidelines](11-conventions-naming-clippy-and-api-guidelines.md) · [📋 Этап 08](README.md) · [08.13 Рефакторинг и код-ревью →](13-refactoring-and-code-review.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/08-software-design/12-project-layout-and-workspaces.md) <!-- nav -->

# Урок 08.12 — Структура проекта и рабочие пространства

> **Этап 08 — Проектирование ПО: принципы, паттерны и соглашения** · Урок 12 из 17 · ⏱ около 3,5 часа (2 занятия)

## Что вы узнаете

- Стандартную структуру пакета Cargo: где лежат программа, библиотека, дополнительные программы, тесты и примеры.
- Как организовать модули внутри `src` и как `pub use` и `pub(crate)` формируют публичную сторону крейта.
- Что такое рабочее пространство Cargo (workspace), как собрать его из трёх крейтов и как участники делят зависимости, версии и настройки lint-ов.
- Как компилятор защищает ваш дизайн (никаких циклов зависимостей) и когда проект *не* нужно делить.

## Перед началом

- Вы прошли [Урок 08.11 — Соглашения: имена, Clippy и Rust API Guidelines](11-conventions-naming-clippy-and-api-guidelines.md).
- Вы знаете модули и пакеты ([Урок 04.6](../04-structs-enums-collections-errors/06-modules-crates-and-packages.md)), зависимости Cargo ([Урок 05.8](../05-rust-intermediate/08-cargo-dependencies-and-versions.md)) и идею «тонкий `main`, всю работу делает библиотека» ([Урок 05.13](../05-rust-intermediate/13-project-minigrep.md)).
- Ваша папка курса уже является репозиторием Git, поэтому `cargo new` не запускает `git init` повторно. (Если вы работаете вне репозитория, добавляйте `--vcs none` к `cargo new` в этом уроке.)

## Новые слова

| Слово | Значение |
|---|---|
| **цель сборки** (target, Cargo) | Одна вещь, которую Cargo может собрать из пакета: библиотека, бинарник, пример, тест или бенчмарк. |
| **рабочее пространство** (workspace) | Группа пакетов в одном дереве папок, у которых общие `Cargo.lock` и папка `target`. |
| **участник** (member) | Пакет, входящий в рабочее пространство. |
| **виртуальный манифест** (virtual manifest) | Корневой `Cargo.toml` с таблицей `[workspace]`, но без `[package]`: сам корень ничего не собирает. |
| **реэкспорт** (`pub use`) | Предоставление элемента из одного модуля под другим путём, чтобы пользователям не нужно было знать вашу внутреннюю структуру. |
| **резолвер** (resolver) | Часть Cargo, которая выбирает версии и фичи зависимостей. `resolver = "3"` — резолвер для редакции 2024. |
| **зависимость по пути** (path dependency) | Зависимость от пакета в папке на вашем диске: `shop-core = { path = "crates/shop-core" }`. |
| **цикл зависимостей** (dependency cycle) | Пакету A нужен B, а B (напрямую или через другие) нужен A. Cargo этого не допускает. |

## 1. Один пакет, несколько целей сборки

**Пакет** описывается одним `Cargo.toml`. Он может порождать много **целей сборки**. Cargo находит их по именам
папок и файлов, поэтому ничего настраивать не нужно:

```text
p08-layout/
├── Cargo.toml
├── src/
│   ├── lib.rs          ← библиотека (настоящая работа)
│   ├── main.rs         ← основная программа, тонкая обёртка
│   └── bin/
│       └── shout.rs    ← ещё одна программа
├── tests/
│   └── greeting.rs     ← интеграционный тест (использует только публичный API)
├── examples/
│   └── basic.rs        ← запускаемый пример для пользователей библиотеки
└── benches/            ← бенчмарки (Этап 09)
```

| Путь | Цель | Как запустить |
|---|---|---|
| `src/lib.rs` | библиотека; другие пакеты и ваши программы подключают её через `use` | (не запускается) |
| `src/main.rs` | основная программа, названная как пакет | `cargo run` |
| `src/bin/NAME.rs` | дополнительная программа | `cargo run --bin NAME` |
| `tests/*.rs` | интеграционные тесты; каждый файл — отдельный крейт | `cargo test` |
| `examples/*.rs` | примеры; `cargo test` тоже их компилирует, поэтому они не устаревают | `cargo run --example NAME` |
| `benches/*.rs` | бенчмарки | `cargo bench` |
| `build.rs` | скрипт сборки, который запускается до компиляции крейта (нужен редко) | автоматически |

Создайте пакет и файлы (имя крейта в коде — `p08_layout`, с подчёркиванием):

```bash
cargo new code/p08-layout
mkdir code/p08-layout/{src/bin,tests,examples}
```

`src/lib.rs`:

```rust
//! Greeting helpers.

/// Builds a greeting.
#[must_use]
pub fn greeting(name: &str) -> String {
    format!("Hello, {name}!")
}
```

`src/main.rs`:

```rust
fn main() {
    println!("{}", p08_layout::greeting("world"));
}
```

`src/bin/shout.rs`:

```rust
fn main() {
    println!("{}", p08_layout::greeting("world").to_uppercase());
}
```

`examples/basic.rs`:

```rust
fn main() {
    println!("{}", p08_layout::greeting("example"));
}
```

`tests/greeting.rs`:

```rust
#[test]
fn greets_by_name() {
    assert_eq!(p08_layout::greeting("Ana"), "Hello, Ana!");
}
```

Теперь `cargo run` жалуется, потому что в пакете две программы:

```text
error: `cargo run` could not determine which binary to run. Use the `--bin` option to specify a binary, or the `default-run` manifest key.
available binaries: p08-layout, shout
```

Добавьте `default-run = "p08-layout"` в таблицу `[package]` файла `Cargo.toml`. Теперь работают все три команды:

```bash
cargo run
cargo run --bin shout
cargo run --example basic
```

```text
Hello, world!
HELLO, WORLD!
Hello, example!
```

**Что происходит:**

1. `main.rs` и `shout.rs` — отдельные программы. Обе вызывают одну библиотеку, поэтому логика существует в одном экземпляре.
2. `cargo test` запускает модульные тесты каждой цели, файлы из `tests/` и документационные тесты. Интеграционный тест видит только то, что `pub`, как и пользователь вашего крейта.
3. Логика в `lib.rs` и тонкие программы (аргументы, вывод, код возврата) — это структура из Урока 05.13. Она делает логику тестируемой.

## 2. Организация `src`

Урок 04.6 показал, как модуль становится файлом (`mod parser;` находит `parser.rs` или `parser/mod.rs`). Вот решения
проектирования поверх этого:

- **`lib.rs` — это оглавление.** Держите его коротким: строки `mod`, несколько строк `pub use` и документация крейта. Читатель должен
  понять, что предлагает крейт, за один экран.
- **Предпочитайте `name.rs` плюс папку `name/`** для дочерних модулей, а не `name/mod.rs`. С `mod.rs` в каждой папке лежит файл с
  одинаковым именем, и вкладки в редакторе теряют смысл.
- **Держите файлы маленькими и целенаправленными.** Файл в 200–400 строк об одном деле легко удержать в голове. Файл `utils.rs` или `helpers.rs`,
  собирающий несвязанные функции, — это «запах» «божественного модуля» из Урока 08.10: называйте модули по тому, *о чём* они.
- **Сначала делайте всё приватным.** Всё приватно по умолчанию. Используйте `pub(crate)` для помощников, которыми пользуются несколько ваших модулей, и `pub` только для того,
  на что должны полагаться пользователи. Элемент `pub` — это обещание.
- **Реэкспортируйте, чтобы скрыть структуру.** Ваши внутренние папки изменятся; код пользователей не должен ломаться. В `lib.rs`:

```rust
mod model;
mod parser;

pub use model::Item;
pub use parser::parse_line;
```

  Пользователи пишут `my_crate::Item`, а не `my_crate::model::Item`. Вы можете перенести `Item` в другой файл без ломающего изменения.
- **Группируйте по возможностям, а не по виду вещей.** Структура `src/orders/`, `src/customers/`, `src/payments/` держит код, который меняется вместе, в одном
  месте. Структура `src/models/`, `src/handlers/`, `src/services/` размазывает одну возможность по трём папкам («запах» «стрельба дробью»). Слои — тема
  [Урока 08.14](14-layers-and-hexagonal-architecture.md); и там обычный совет такой: сначала папки по возможностям, слои внутри.

> 📏 **Соглашение:** в `lib.rs` используйте `//!` для документации крейта, затем объявления `mod`, затем `pub use`. В любом другом файле сначала строки `use`,
> затем типы, затем блоки `impl`, а в конце `#[cfg(test)] mod tests`.

## 3. Рабочие пространства Cargo

Когда проект растёт, вам могут понадобиться несколько пакетов, которые собираются, тестируются и версионируются вместе. Это делает **рабочее пространство**.
Вы построите `p08-shop` — крошечный магазин из трёх крейтов:

```text
shop-cli ──► shop-report ──► shop-core
   └──────────────────────────►
```

- `shop-core` хранит предметную область: товары и цены. Он не зависит ни от чего нашего.
- `shop-report` превращает товары в текст. Он зависит от `shop-core`.
- `shop-cli` — программа. Она зависит от обоих.

Стрелки показывают **направление зависимостей**: от внешнего (программа) к внутреннему (предметная область). Внутреннее ничего не знает о внешнем. Это
правило вы встретите снова в Уроке 08.14.

### 3.1 Создайте рабочее пространство

```bash
mkdir code/p08-shop
cd code/p08-shop
mkdir crates
cargo new --lib crates/shop-core
cargo new --lib crates/shop-report
cargo new crates/shop-cli
```

Поместите это в **корневой** `Cargo.toml` (создайте его; в нём нет `[package]`, так что это *виртуальный манифест*):

```toml
[workspace]
resolver = "3"
members = ["crates/*"]

[workspace.package]
version = "0.1.0"
edition = "2024"
license = "MIT"

[workspace.dependencies]
shop-core = { path = "crates/shop-core" }
shop-report = { path = "crates/shop-report" }
thiserror = "2"

[workspace.lints.clippy]
pedantic = { level = "warn", priority = -1 }
```

**Что происходит:**

1. `members = ["crates/*"]` делает участником каждую папку внутри `crates/`. Новый крейт входит в рабочее пространство, как только вы создаёте его папку.
2. `[workspace.package]` хранит значения, общие для всех участников (версия, редакция, лицензия), так что вы меняете их в одном месте.
3. `[workspace.dependencies]` перечисляет зависимости **один раз**: внешние крейты с версией (`thiserror`, Урок 05.11) и наши собственные крейты по пути. Участники затем пишут
   `thiserror.workspace = true` и наследуют строку. Гарантируется, что все участники используют одну версию.
4. `[workspace.lints]` — политика lint-ов из Урока 08.11 для всего рабочего пространства. Каждый участник подключается через `[lints] workspace = true`.
5. `resolver = "3"` — резолвер зависимостей для редакции 2024. (Рабочему пространству его нужно указать явно, потому что в корне нет `edition`, из которого его можно взять.)

Теперь три манифеста участников. `crates/shop-core/Cargo.toml`:

```toml
[package]
name = "shop-core"
version.workspace = true
edition.workspace = true
license.workspace = true

[dependencies]
thiserror.workspace = true

[lints]
workspace = true
```

`crates/shop-report/Cargo.toml`:

```toml
[package]
name = "shop-report"
version.workspace = true
edition.workspace = true
license.workspace = true

[dependencies]
shop-core.workspace = true

[lints]
workspace = true
```

`crates/shop-cli/Cargo.toml` такой же, но с `name = "shop-cli"` и двумя зависимостями:

```toml
[package]
name = "shop-cli"
version.workspace = true
edition.workspace = true
license.workspace = true

[dependencies]
shop-core.workspace = true
shop-report.workspace = true

[lints]
workspace = true
```

### 3.2 Код

`crates/shop-core/src/lib.rs`:

```rust
//! The domain: items and prices.

use thiserror::Error;

/// Something that can be bought.
#[derive(Debug, Clone, PartialEq)]
pub struct Item {
    name: String,
    price_cents: u32,
}

/// Why an item could not be created.
#[derive(Debug, Error, PartialEq)]
pub enum ItemError {
    /// The name is empty.
    #[error("the name must not be empty")]
    EmptyName,
}

impl Item {
    /// Creates an item.
    ///
    /// # Errors
    ///
    /// Returns [`ItemError::EmptyName`] when `name` is empty.
    pub fn new(name: &str, price_cents: u32) -> Result<Item, ItemError> {
        if name.is_empty() {
            return Err(ItemError::EmptyName);
        }
        Ok(Item {
            name: name.to_string(),
            price_cents,
        })
    }

    /// The item name.
    #[must_use]
    pub fn name(&self) -> &str {
        &self.name
    }

    /// The price in cents.
    #[must_use]
    pub fn price_cents(&self) -> u32 {
        self.price_cents
    }
}

/// The sum of all prices, in cents.
#[must_use]
pub fn total_cents(items: &[Item]) -> u32 {
    items.iter().map(Item::price_cents).sum()
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn total_adds_prices() {
        let items = [
            Item::new("tea", 250).unwrap(),
            Item::new("cake", 400).unwrap(),
        ];
        assert_eq!(total_cents(&items), 650);
    }

    #[test]
    fn empty_name_is_rejected() {
        assert_eq!(Item::new("", 100), Err(ItemError::EmptyName));
    }
}
```

`crates/shop-report/src/lib.rs`:

```rust
//! Text reports about items.

use shop_core::{Item, total_cents};

fn line(label: &str, cents: u32) -> String {
    format!("{label:<8}{cents:>6}\n")
}

/// Renders one line per item and a total line.
#[must_use]
pub fn render(items: &[Item]) -> String {
    let mut lines: Vec<String> = items
        .iter()
        .map(|item| line(item.name(), item.price_cents()))
        .collect();
    lines.push(line("total", total_cents(items)));
    lines.concat()
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn report_has_a_total_line() {
        let items = [Item::new("tea", 250).unwrap()];
        assert_eq!(render(&items), "tea        250\ntotal      250\n");
    }
}
```

`crates/shop-cli/src/main.rs`:

```rust
use shop_core::Item;
use shop_report::render;

fn main() {
    let items = [
        Item::new("tea", 250).expect("valid item"),
        Item::new("cake", 400).expect("valid item"),
    ];
    print!("{}", render(&items));
}
```

**Что происходит:**

1. У `Item` приватные поля и проверяющий конструктор (Уроки 08.5 и 08.11). Другой крейт может прочитать `Item` только через `name()` и `price_cents()`.
   Между крейтами `pub` — единственный способ поделиться: `pub(crate)` заканчивается на границе крейта. Поэтому рабочее пространство заставляет вас решить, какова публичная сторона.
2. `shop_report` использует `shop_core::{Item, total_cents}`. Имя крейта в коде — это имя пакета с подчёркиваниями.
3. `render` собирает строки в векторе и склеивает их через `concat`. (Первую версию с `text.push_str(&format!(...))` отклонила настройка `pedantic` рабочего пространства
   с сообщением `` `format!(..)` appended to existing `String` `` — политика lint-ов действует на всех участников.)
4. Формат `{label:<8}{cents:>6}` дополняет название до 8 символов, выравнивая по левому краю, а число — до 6, выравнивая по правому.

### 3.3 Сборка, запуск, тесты

Из корня рабочего пространства (`code/p08-shop`):

```bash
cargo run
```

```text
tea        250
cake       400
total      650
```

`cargo run` в корне работает, потому что во всём рабочем пространстве только одна программа. Затем:

```bash
cargo test --workspace
cargo clippy --workspace --all-targets -- -D warnings
cargo test -p shop-core
```

`cargo test --workspace` запускает тесты каждого участника (сокращено):

```text
test tests::empty_name_is_rejected ... ok
test tests::total_adds_prices ... ok
test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
test tests::report_has_a_total_line ... ok
test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

(Cargo также печатает строки `0 passed` для целей без тестов.) Команда `clippy` печатает только `Checking` и `Finished`. `-p NAME` (`--package`) ограничивает команду одним участником.

Посмотрите на папку. У всех участников **один** `Cargo.lock` и **одна** папка `target`, обе в корне, поэтому зависимость компилируется один раз на всё рабочее пространство:

```text
p08-shop/
├── Cargo.toml          ← виртуальный манифест
├── Cargo.lock          ← один на всех участников
├── target/             ← одна папка сборки
└── crates/
    ├── shop-core/      (Cargo.toml, src/lib.rs)
    ├── shop-report/    (Cargo.toml, src/lib.rs)
    └── shop-cli/       (Cargo.toml, src/main.rs)
```

> 📏 **Соглашение:** коммитьте `Cargo.lock` для рабочих пространств, содержащих программу (Урок 05.8), и добавьте `target/` в `.gitignore`.

### 3.4 Компилятор охраняет ваш дизайн

Допустим, разработчик в спешке заставляет `shop-core` использовать код отчётов: он добавляет `shop-report.workspace = true` в `[dependencies]` крейта `shop-core`. Cargo отказывается собирать:

```text
error: cyclic package dependency: package `shop-core v0.1.0 (...)` depends on itself. Cycle:
package `shop-core v0.1.0 (...)`
    ... which satisfies path dependency `shop-core` (locked to 0.1.0) of package `shop-report v0.1.0 (...)`
    ... which satisfies path dependency `shop-report` (locked to 0.1.0) of package `shop-core v0.1.0 (...)`
```

Предметная область не может зависеть от кода, который зависит от неё. Это достоинство: стрелки на схеме раздела 3 *принудительно соблюдаются* компилятором. Внутри одного крейта ничто не мешает
`domain.rs` вызвать `cli.rs`; между крейтами сборка падает. Если вы увидите эту ошибку, исправление почти никогда не состоит в слиянии крейтов. Спросите, кто
из двух не должен знать о другом; обычно ответ — перенести общую часть «вниз» или позволить нижнему крейту определить трейт, который реализует верхний (Уроки 08.3 и 08.9).

## 4. Когда делить, а когда нет

Начинайте с **одного пакета** (библиотека + тонкая программа). Делите на рабочее пространство, когда у вас есть причина, которую можно назвать:

- **Граница, которую вы хотите принудительно соблюдать.** Предметная область не должна знать о командной строке или базе данных (раздел 3.4).
- **Разные пользователи.** Одну часть используют программа, сервер и тестовый инструмент; другую — только сервер.
- **Разные зависимости.** Предметная область не должна тянуть веб-фреймворк. Отдельный крейт держит тяжёлые зависимости подальше от крейтов, которым они не нужны.
- **Время сборки.** Cargo компилирует крейты параллельно и пересобирает только изменённые, так что независимые крейты могут собираться быстрее одного огромного (сначала измерьте, Урок 05.12).
- **Публикация.** Часть, которую вы хотите опубликовать как библиотеку в реестре.

*Не* делите потому, что «у больших проектов много крейтов». Рабочее пространство из пятнадцати крейтов по пятьдесят строк — это «запах» «микрокрейтов»: каждый общий элемент приходится
делать `pub`, каждое изменение затрагивает три манифеста, и никто не видит программу целиком. Модуль — правильный инструмент для порядка; крейт — правильный инструмент для границы.

## 5. Файлы вокруг кода

В корне проекта обычно лежит больше, чем файлы Cargo:

| Файл | Назначение |
|---|---|
| `README.md` | Что за проект, как его собрать и запустить (Урок 05.7). |
| `LICENSE` | Лицензия. Без неё другие юридически не вправе использовать ваш код. |
| `.gitignore` | Должен содержать `target/`. `cargo new` создаёт его, только когда заодно создаёт новый Git-репозиторий; внутри существующего репозитория (как ваша папка курса) это правило должно быть в корневом `.gitignore`. |
| `rust-toolchain.toml` | Фиксирует версию Rust для проекта (см. ниже). |
| `rustfmt.toml`, `clippy.toml` | Настройки формата и lint-ов (Урок 08.11); большинству проектов не нужны оба. |
| `CHANGELOG.md` | Что изменилось в каждом релизе (Урок 05.7). |
| `deny.toml` | Политика `cargo-deny` для лицензий и бюллетеней безопасности (Этап 10). |
| `.github/workflows/` | Конвейеры CI (Continuous Integration, непрерывная интеграция; Этап 09). |

`rust-toolchain.toml` заставляет всех (и CI) использовать один и тот же компилятор:

```toml
[toolchain]
channel = "1.100.0"
components = ["rustfmt", "clippy"]
```

Когда rustup видит этот файл, а тулчейн не установлен, он скачивает его перед запуском любой команды `cargo`; первая строка, которую вы увидите, —
`info: syncing channel updates for 1.100.0-aarch64-apple-darwin`. (Добавляйте файл только когда вам это нужно. Черновому проекту в вашем учебном репозитории
он не нужен.)

## Частые ошибки

1. **Вся логика в `main.rs`.** Интеграционные тесты не могут импортировать бинарный крейт. Перенесите логику в `lib.rs`.
2. **Забыли `default-run`** в пакете с несколькими бинарниками; `cargo run` тогда просит выбрать.
3. **Имя пакета с дефисом в коде** (`use shop-core::Item;`). В коде это `shop_core`.
4. **Разные версии одной зависимости у разных участников.** Помещайте общие зависимости в `[workspace.dependencies]` и используйте `dep.workspace = true`.
5. **Забыли `[lints] workspace = true`** у участника. Таблица `[workspace.lints]` ничего не делает, пока каждый участник к ней не подключится.
6. **Циклические зависимости.** Исправьте направление (раздел 3.4); не сливайте крейты, чтобы скрыть проблему.

## Попробуйте сами

1. Постройте `p08-shop` в точности как в разделе 3 и запустите `cargo run`, `cargo test --workspace` и `cargo clippy --workspace --all-targets -- -D warnings`.
2. Запустите `cargo tree` в корне. Найдите `shop-cli` и убедитесь, что `shop-core` и `shop-report` расположены под ним, а `thiserror` — под `shop-core`.
3. Вызовите ошибку цикла из раздела 3.4 (добавьте `shop-report.workspace = true` в `[dependencies]` файла `crates/shop-core/Cargo.toml`), прочитайте сообщение и удалите строку.
4. Сделайте коммит: `git add code/p08-shop && git commit -m "feat(p08): shop workspace with three crates"`.

## Упражнения

### Упражнение 1 — Какой файл? (легко)

В каком файле или папке пакета находится: (а) вторая программа, которая преобразует файл; (б) тест, использующий только публичные функции; (в) короткая программа,
показывающая пользователю, как вызывать вашу библиотеку; (г) бенчмарк; (д) вспомогательный код, общий для двух модулей, но не предназначенный для пользователей.

<details>
<summary>Подсказка</summary>

Раздел 1, таблица целей. Для (д) думайте о видимости, а не о папках.

</details>

<details>
<summary>Ответ</summary>

(а) `src/bin/convert.rs`, запуск `cargo run --bin convert`. (б) `tests/some_name.rs`. (в) `examples/some_name.rs`, запуск `cargo run --example some_name`.
(г) `benches/some_name.rs`. (д) модуль внутри `src` с элементами `pub(crate)`.

</details>

### Упражнение 2 — Добавьте крейт с тестом (средне)

Добавьте в рабочее пространство четвёртого участника `shop-discount` с функцией `pub fn apply_percent(cents: u64, percent: u64) -> u64`, которая снимает `percent` процентов, округляя скидку вниз
и считая процент больше 100 равным 100. Напишите тесты: 10% от 1000, 50% от 999 и 150% от 1000. Новый крейт должен наследовать версию, редакцию, лицензию и lint-ы из рабочего пространства. Нужно ли править корневой `Cargo.toml`?

<details>
<summary>Подсказка</summary>

`members = ["crates/*"]` подхватывает новые папки. Скопируйте таблицы `[package]` и `[lints]` другого участника. Скидка 50% от 999 — это `999 * 50 / 100 = 499` (целочисленное деление), так что результат 500.

</details>

<details>
<summary>Ответ</summary>

Править корневой файл не нужно. `cargo new --lib crates/shop-discount`, затем `crates/shop-discount/Cargo.toml`:

```toml
[package]
name = "shop-discount"
version.workspace = true
edition.workspace = true
license.workspace = true

[lints]
workspace = true
```

и `crates/shop-discount/src/lib.rs`:

```rust
//! Discount rules.

/// Takes `percent` percent off `cents`, rounding down.
///
/// A `percent` above 100 is treated as 100.
#[must_use]
pub fn apply_percent(cents: u64, percent: u64) -> u64 {
    let percent = percent.min(100);
    cents - cents * percent / 100
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn takes_a_percentage_off() {
        assert_eq!(apply_percent(1000, 10), 900);
        assert_eq!(apply_percent(999, 50), 500);
    }

    #[test]
    fn percent_is_capped_at_100() {
        assert_eq!(apply_percent(1000, 150), 0);
    }
}
```

`cargo test -p shop-discount` печатает `test result: ok. 2 passed`. Чтобы использовать крейт в `shop-cli`, добавьте `shop-discount = { path = "crates/shop-discount" }` в `[workspace.dependencies]`
и `shop-discount.workspace = true` в `[dependencies]` крейта `shop-cli`.

</details>

### Упражнение 3 — Разорвите цикл (сложно)

Коллега хочет, чтобы в `shop-core` была функция `print_report(items)`, которая вызывает `shop_report::render`. Cargo сообщает о циклической зависимости. Опишите два решения, которые избегают цикла
без слияния крейтов.

<details>
<summary>Подсказка</summary>

Кому разрешено знать о ком? Посмотрите на стрелки в разделе 3. Подумайте о варианте «нижний крейт определяет трейт» (Урок 08.9).

</details>

<details>
<summary>Ответ</summary>

Решение 1: не помещайте `print_report` в `shop-core`. Поместите её в `shop-report` (которому разрешено зависеть от `shop-core`) или в `shop-cli`. Предметная область остаётся свободной от кода вывода.

Решение 2: если предметная область действительно должна запускать вывод, пусть она определит трейт, например `pub trait Reporter { fn report(&self, items: &[Item]); }`, и принимает `&dyn Reporter` или обобщённый
`R: Reporter`. Крейт `shop-report` реализует `Reporter`, а `shop-cli` передаёт реализацию. Стрелка зависимости по-прежнему направлена от `shop-report` к `shop-core`.

</details>

## Вопросы для самопроверки

1. Чем отличаются пакет, крейт и рабочее пространство?

<details>
<summary>Ответ</summary>

У пакета один `Cargo.toml`, и он может содержать библиотечный крейт и бинарные крейты. Крейт — одна единица компиляции (библиотека или программа). Рабочее пространство — группа пакетов
с общими `Cargo.lock` и папкой `target`.

</details>

2. Где живёт логика программы и что делает `main.rs`?

<details>
<summary>Ответ</summary>

В `lib.rs` и его модулях, где до неё могут добраться тесты. `main.rs` тонкий: читает аргументы, вызывает библиотеку, печатает и выбирает код возврата.

</details>

3. Что дают `[workspace.dependencies]` и `dep.workspace = true`?

<details>
<summary>Ответ</summary>

Одно место, где объявлена каждая зависимость и её версия, так что все участники используют одну и ту же, а обновление — изменение одной строки.

</details>

4. Почему цикл зависимостей не может возникнуть случайно внутри одного крейта и что делает Cargo между крейтами?

<details>
<summary>Ответ</summary>

Внутри одного крейта модули могут свободно использовать друг друга, поэтому циклы возможны, и никто им не мешает. Между крейтами Cargo отказывается собирать («cyclic package dependency»), что
принудительно задаёт направление зависимостей.

</details>

## Итоги

- Пакет собирает несколько *целей* из стандартных путей: `src/lib.rs`, `src/main.rs`, `src/bin/*.rs`, `tests/`, `examples/`, `benches/`. Логику держите в библиотеке, а программы — тонкими.
- Используйте `default-run`, когда в пакете несколько программ.
- Внутри `src`: `lib.rs` как оглавление, небольшие целенаправленные модули, названные по тому, о чём они, приватность по умолчанию, `pub(crate)` для внутреннего обмена, `pub use`, чтобы скрыть вашу структуру,
  и папки по возможностям раньше папок по слоям.
- Рабочее пространство делит `Cargo.lock`, `target`, версии (`[workspace.package]`), зависимости (`[workspace.dependencies]`) и политику lint-ов (`[workspace.lints]`).
- Зависимости направлены от внешнего (программы) к внутреннему (предметная область); Cargo отклоняет циклы между крейтами.
- Начинайте с одного пакета; делите, когда можете назвать причину (граница, разные пользователи или зависимости, публикация). Избегайте микрокрейтов.

## Источники и что почитать

- [The Cargo Book — Cargo Targets](https://doc.rust-lang.org/cargo/reference/cargo-targets.html) — все виды целей и как Cargo их находит.
- [The Cargo Book — Package layout](https://doc.rust-lang.org/cargo/guide/project-layout.html) — стандартная структура пакета.
- [The Cargo Book — Workspaces](https://doc.rust-lang.org/cargo/reference/workspaces.html) — `[workspace]`, наследование полей пакета, зависимостей и lint-ов.
- [The Rust Programming Language — Chapter 7: Managing Growing Projects with Packages, Crates, and Modules](https://doc.rust-lang.org/book/ch07-00-managing-growing-projects-with-packages-crates-and-modules.html) — модули, пути и видимость.
- [The Rust Programming Language — Chapter 14.3: Cargo Workspaces](https://doc.rust-lang.org/book/ch14-03-cargo-workspaces.html) — первое рабочее пространство шаг за шагом.
- [The rustup book — Overrides](https://rust-lang.github.io/rustup/overrides.html) — файл `rust-toolchain.toml`.

---

[🏠 Начало](../START-HERE.md) · [← 08.11 Соглашения: имена, Clippy и Rust API Guidelines](11-conventions-naming-clippy-and-api-guidelines.md) · [📋 Этап 08](README.md) · [08.13 Рефакторинг и код-ревью →](13-refactoring-and-code-review.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
