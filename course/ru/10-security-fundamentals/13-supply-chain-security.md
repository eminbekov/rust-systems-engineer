<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 10.12 Безопасность памяти и проверка небезопасного кода](12-memory-safety-and-reviewing-unsafe.md) · [📋 Этап 10](README.md) · [10.14 Проект: зашифрованное хранилище секретов →](14-project-encrypted-secrets-vault.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/10-security-fundamentals/13-supply-chain-security.md) <!-- nav -->

# Урок 10.13 — Безопасность цепочки поставок: cargo-audit и cargo-deny

> **Этап 10 — Основы безопасности** · Урок 13 из 15 · ⏱ около 120 минут

## Что вы узнаете

- Что такое цепочка поставок программного обеспечения и какие атаки идут через ваши зависимости.
- Как увидеть, от чего на самом деле зависит ваш проект (`cargo tree`), и почему важен `Cargo.lock`.
- Как находить известные уязвимости и неподдерживаемые крейты с помощью `cargo-audit` и как их исправлять.
- Как проверять правила о лицензиях, запрещённых крейтах и источниках с помощью `cargo-deny`.
- Привычки при добавлении новой зависимости и как запускать оба инструмента автоматически.

## Перед началом

- Вы прошли [Урок 10.12 — Безопасность памяти и проверка небезопасного кода](12-memory-safety-and-reviewing-unsafe.md).
- Вы знаете зависимости, требования к версиям и `Cargo.lock` ([Урок 05.8](../05-rust-intermediate/08-cargo-dependencies-and-versions.md)).
- Установите два инструмента. Это обычные программы на Rust; `--locked` заставляет Cargo использовать точные версии зависимостей, которые проверили авторы. Это компилирует много кода и занимает несколько минут:

```bash
cargo install --locked cargo-audit cargo-deny
cargo audit --version
cargo deny --version
```

```text
cargo-audit-audit 0.23.0
cargo-deny 0.21.0
```

(Повторённое слово в `cargo-audit-audit` — просто то, как эта версия инструмента печатает своё имя; важен номер версии.)

- Создайте черновой пакет, который вы удалите в конце. Он намеренно использует **старую** версию крейта с опубликованной уязвимостью:

```bash
cd ~/projects/rust
cargo new code/p10-audit-demo
cd code/p10-audit-demo
cargo add regex@1.5 atty@0.2
cargo update -p regex --precise 1.5.4
```

```text
    Updating crates.io index
 Downgrading aho-corasick v1.1.5 -> v0.7.20
 Downgrading regex v1.13.2 -> v1.5.4
    Removing regex-automata v0.4.19
 Downgrading regex-syntax v0.8.12 -> v0.6.29
```

Замените `src/main.rs` крошечной программой, которая использует оба крейта:

```rust
use regex::Regex;

fn main() {
    let re = Regex::new(r"^\d{4}-\d{2}-\d{2}$").unwrap();
    println!("date? {}", re.is_match("2026-11-18"));
    println!("terminal? {}", atty::is(atty::Stream::Stdout));
}
```

```text
date? true
terminal? false
```

(`terminal? false` появляется, потому что в этом листинге вывод `cargo run` не подключён к терминалу; на вашем экране может быть `true`.)

> 🔒 **Безопасность и этика:** этот старый `regex` является уязвимостью только тогда, когда программа компилирует регулярные выражения, присланные недоверенным пользователем. Демонстрационная программа никому не доступна. Никогда не развёртывайте проект, использующий версию с известной уязвимостью.

## Новые слова

| Слово | Значение |
|---|---|
| **цепочка поставок** (supply chain) | Всё, что попадает в вашу программу и что вы не писали сами: зависимости, инструменты сборки, реестр, компилятор. |
| **дерево зависимостей** (dependency tree) | Все крейты, нужные проекту, включая зависимости ваших зависимостей (*транзитивные* зависимости). |
| **бюллетень безопасности** (advisory) | Опубликованное уведомление о безопасности версии крейта: что не так, какие версии затронуты и какая версия исправляет проблему. |
| **RustSec** | База данных бюллетеней безопасности Rust для крейтов, опубликованных на crates.io. У записей есть идентификаторы вроде `RUSTSEC-2022-0013`. |
| **CVE** (Common Vulnerabilities and Exposures, общеизвестные уязвимости и риски) | Всемирная система нумерации уязвимостей, например `CVE-2022-24713`. |
| **не поддерживается** (unmaintained) | Крейт, авторы которого заявили, что прекратили работу над ним. Он может продолжать работать, но исправлений безопасности не получит. |
| **некорректный** (unsound) | Крейт с безопасной функцией, которая может вызвать неопределённое поведение (Урок 10.12). |
| **отозванная версия** (yanked) | Версия, которую автор изъял из реестра, потому что она сломана или опасна. Проекты, уже зафиксировавшие её, продолжают работать, а новые проекты её не выберут. |
| **лицензия** (license) | Юридические условия использования кода. Некоторые лицензии (например, GPL, General Public License, универсальная общественная лицензия) накладывают условия на ваш собственный код. |
| **SPDX** (Software Package Data Exchange) | Стандартный список коротких названий лицензий, таких как `MIT` и `Apache-2.0`. |
| **тайпсквоттинг** (typosquatting) | Публикация вредоносного крейта с именем, близким к популярному (`serde_jsonn`), в расчёте на опечатки. |
| **`build.rs` / процедурный макрос** | Код, который выполняется на вашем компьютере **во время компиляции**. Зависимость может его использовать. |

## 1. Что такое цепочка поставок и как её атакуют?

В типичной программе на Rust ваши 500 строк и код десятков или сотен крейтов, написанных другими людьми. Вы доверяете всему этому данные своих пользователей. Атаки идут через это доверие:

| Атака | Пример | Защита |
|---|---|---|
| **Известная уязвимость** в зависимости | Старая версия `regex`, которую можно заставить долго обрабатывать специально составленный шаблон | `cargo audit`, `cargo deny check advisories`, регулярные обновления |
| **Неподдерживаемый или некорректный** крейт | `atty`: он работает, но проблемы никто не исправляет | Те же инструменты, затем замена крейта |
| **Вредоносный крейт** | Тайпсквоттинг; украденная учётная запись автора публикует плохое обновление | Тщательная проверка новых зависимостей, `Cargo.lock`, короткий список зависимостей |
| **Код, выполняемый при сборке** | `build.rs` или процедурный макрос читает ваши файлы или секреты | Проверка, сборка в чистом окружении CI (Continuous Integration, непрерывная интеграция: сервер, запускающий проверки при каждой отправке) без секретов |
| **Проблема с лицензией** | Зависимость с условиями, которые ваша компания принять не может | `cargo deny check licenses` |
| **Неожиданный источник** | Зависимость, взятая из случайного Git-адреса (URL, Uniform Resource Locator, веб-адрес) | `cargo deny check sources` |

Полностью устранить риск нельзя, но можно сделать его малым и **видимым**.

## 2. Знайте, от чего вы зависите

```bash
cargo tree
```

```text
p10-audit-demo v0.1.0 (/Users/ada/projects/rust/code/p10-audit-demo)
├── atty v0.2.14
│   └── libc v0.2.191
└── regex v1.5.4
    ├── aho-corasick v0.7.20
    │   └── memchr v2.8.3
    ├── memchr v2.8.3
    └── regex-syntax v0.6.29
```

Две прямые зависимости превратились в **шесть крейтов** в дереве. (В `Cargo.lock` всего 11 пакетов: эти шесть, сама демонстрационная программа и ещё четыре, нужные другим системам: `hermit-abi` и три крейта `winapi`, которые `atty` использует в Windows и на других платформах.) Две дополнительные команды `cargo tree` отвечают на частые вопросы:

```bash
cargo tree -i libc    # inverse: who depends on libc?
cargo tree -d         # duplicates: the same crate in two versions
```

```text
libc v0.2.191
└── atty v0.2.14
    └── p10-audit-demo v0.1.0 (/Users/ada/projects/rust/code/p10-audit-demo)
```

`cargo tree -d` напечатала `warning: nothing to print.`: дубликатов здесь нет. Дубликаты увеличивают сборку и поверхность атаки.

`Cargo.lock` записывает точную версию (и контрольную сумму) каждого крейта в дереве. Для **программ** его коммитят в Git, чтобы все собирали один и тот же код. В CI `cargo build --locked` завершается ошибкой, если файл блокировки изменился бы, поэтому зависимость не может измениться незаметно. Реестр также хранит хеш каждой загрузки, а Cargo сверяет его с файлом блокировки (Урок 10.9: хеш обнаруживает случайное или враждебное изменение файла).

## 3. cargo-audit: известные уязвимости

`cargo-audit` читает `Cargo.lock` и ищет каждую пару `имя + версия` в **базе бюллетеней RustSec**.

```bash
cargo audit
```

При первом запуске загружается база (первые строки сообщают, откуда и сколько бюллетеней загружено; оба значения со временем меняются). Остальное выглядит так:

```text
    Scanning Cargo.lock for vulnerabilities (11 crate dependencies)
Crate:     regex
Version:   1.5.4
Title:     Regexes with large repetitions on empty sub-expressions take a very long time to parse
Date:      2022-03-08
ID:        RUSTSEC-2022-0013
URL:       https://rustsec.org/advisories/RUSTSEC-2022-0013
Severity:  7.5 (high)
Solution:  Upgrade to >=1.5.5 (patched)

Crate:     atty
Version:   0.2.14
Warning:   unmaintained
Title:     `atty` is unmaintained
Date:      2024-09-25
ID:        RUSTSEC-2024-0375
URL:       https://rustsec.org/advisories/RUSTSEC-2024-0375
Solution:  No fixed upgrade is available!

Crate:     atty
Version:   0.2.14
Warning:   unsound
Title:     Potential unaligned read
Date:      2021-07-04
ID:        RUSTSEC-2021-0145
URL:       https://rustsec.org/advisories/RUSTSEC-2021-0145
Solution:  No fixed upgrade is available!

error: 1 vulnerability found!
warning: 2 allowed warnings found
```

**Что происходит:**

1. Инструмент нашёл три проблемы. Первая — настоящая **уязвимость** (атакующий, который может отправить вашей программе регулярное выражение, способен заставить её очень долго его компилировать; отказ в обслуживании, Урок 10.12). Поле solution говорит, какая версия это исправляет.
2. Два других — *предупреждения* об `atty`: не поддерживается и некорректен (в Windows может неправильно читать память). Обновляться некуда, поэтому исправление — перестать использовать этот крейт.
3. Команда завершается ненулевым кодом возврата (`1`), потому что нашла уязвимость. Так CI замечает проблему. Одни предупреждения её не проваливают, если вы не попросите: `cargo audit --deny warnings`.

**Исправление 1: обновление** (у уязвимости есть исправленная версия). `cargo update -p` обновляет один пакет в границах `Cargo.toml` (`"1.5"` допускает любую `1.x` начиная с `1.5`):

```bash
cargo update -p regex
```

```text
    Updating crates.io index
     Locking 4 packages to highest Rust 1.100.0 compatible versions
    Updating aho-corasick v0.7.20 -> v1.1.5
    Updating regex v1.5.4 -> v1.13.2
      Adding regex-automata v0.4.19
    Updating regex-syntax v0.6.29 -> v0.8.12
```

**Исправление 2: замена** крейта, для которого исправления нет. В стандартной библиотеке есть `IsTerminal` начиная с Rust 1.70, поэтому `atty` не нужен. Измените `main.rs`, затем удалите зависимость:

```rust
use regex::Regex;
use std::io::IsTerminal;

fn main() {
    let re = Regex::new(r"^\d{4}-\d{2}-\d{2}$").unwrap();
    println!("date? {}", re.is_match("2026-11-18"));
    println!("terminal? {}", std::io::stdout().is_terminal());
}
```

```bash
cargo remove atty
cargo audit
```

```text
    Scanning Cargo.lock for vulnerabilities (6 crate dependencies)
```

Ничего не найдено, код возврата `0`. Дерево зависимостей сократилось с 11 пакетов до 6.

> 📏 **Соглашение:** запускайте `cargo audit` в CI при каждой отправке **и по расписанию**. Новые бюллетени публикуются каждую неделю для кода, который не менялся. Если находку нельзя исправить сразу, её можно игнорировать через `cargo audit --ignore RUSTSEC-xxxx-xxxx` или в `.cargo/audit.toml`, но запишите причину и назначьте дату повторной проверки.

## 4. cargo-deny: правила для всего дерева

`cargo-audit` отвечает на один вопрос. `cargo-deny` проверяет четыре вида правил, чтобы команда один раз приняла решение, а инструмент его обеспечивал:

| Проверка | Что обеспечивает |
|---|---|
| `advisories` | Снова проверка RustSec (уязвимости, неподдерживаемые, некорректные, **отозванные** версии). |
| `licenses` | Только те названия лицензий, которые вы разрешили. |
| `bans` | Крейты, которые вы запрещаете, и может ли дерево содержать один крейт в нескольких версиях. |
| `sources` | Только те реестры и Git-репозитории, которые вы перечислили. |

Создайте начальную конфигурацию и посмотрите на неё:

```bash
cargo deny init
```

Она записывает `deny.toml` — длинный файл, где каждая опция объяснена в комментариях. Замените его этой короткой рабочей версией:

```toml
[graph]
all-features = false

[advisories]
yanked = "deny"

[licenses]
allow = ["MIT", "Apache-2.0"]

[licenses.private]
ignore = true # our own, unpublished crate needs no license

[bans]
multiple-versions = "warn"
wildcards = "deny"

[sources]
unknown-registry = "deny"
unknown-git = "deny"
allow-registry = ["https://github.com/rust-lang/crates.io-index"]
```

Добавьте также `publish = false` в `[package]` файла `Cargo.toml`, чтобы `cargo-deny` знал, что демонстрационный крейт приватный.

**Что происходит:**

1. `yanked = "deny"` делает отозванную версию ошибкой. Уязвимости по умолчанию — ошибки.
2. `licenses.allow` — это **список разрешённого**: крейт принимается, если его выражение лицензии можно выполнить разрешёнными названиями. `"MIT OR Apache-2.0"` (очень распространённое) принимается, потому что нужна лишь одна из них.
3. `multiple-versions = "warn"` сообщает об одном крейте дважды в дереве; `wildcards = "deny"` запрещает зависимости, записанные как `version = "*"` (любая версия), которые превращают каждое обновление в сюрприз.
4. `sources` разрешает только официальный индекс crates.io; зависимость из любого Git-адреса — ошибка.
5. Таблица `bans` может также запретить крейт по имени, с указанием причины. Это правило вы добавите через минуту.

Чтобы увидеть, как проверки не проходят, воспроизведите старое состояние: верните `atty` (`cargo add atty@0.2` и строку `atty::is` в `main.rs`) и понизьте `regex` (`cargo update -p regex --precise 1.5.4`). Затем выполните:

```bash
cargo deny check
```

Полный вывод длинный: к каждой находке прилагается текст бюллетеня. Вот сокращённая версия, по одному сообщению на проблему:

```text
error[unsound]: Potential unaligned read
  ├ ID: RUSTSEC-2021-0145
  ├ Solution: No safe upgrade is available!
  ├ atty v0.2.14
    └── p10-audit-demo v0.1.0

error[vulnerability]: Regexes with large repetitions on empty sub-expressions take a very long time to parse
  ├ ID: RUSTSEC-2022-0013
  ├ Solution: Upgrade to >=1.5.5 (try `cargo update -p regex`)
  ├ regex v1.5.4
    └── p10-audit-demo v0.1.0

error[unmaintained]: `atty` is unmaintained
  ├ ID: RUSTSEC-2024-0375
  ├ Solution: No safe upgrade is available!
  ├ atty v0.2.14
    └── p10-audit-demo v0.1.0

advisories FAILED, bans ok, licenses ok, sources ok
```

Последняя строка — итог четырёх проверок. Теперь намеренно проваливаем остальные проверки. Измените `allow` на `["Apache-2.0"]` и выполните `cargo deny check licenses`:

```text
error[rejected]: failed to satisfy license requirements
   ┌─ ~/.cargo/registry/src/index.crates.io-…/atty-0.2.14/Cargo.toml:23:12
   │
23 │ license = "MIT"
   │            ━━━
   │            │
   │            rejected: license is not explicitly allowed
   │
   ├ MIT - MIT License:
   ├   - OSI approved
   ├   - FSF Free/Libre
   ├ atty v0.2.14
     └── p10-audit-demo v0.1.0
```

(Пути сокращены; будет по одной такой ошибке на каждый крейт, чью лицензию вы не разрешили.) Верните `allow = ["MIT", "Apache-2.0"]`. Теперь запретите крейт по имени: добавьте эту строку в таблицу `[bans]`, сразу после `wildcards = "deny"`, и выполните `cargo deny check bans`:

```toml
deny = [{ crate = "atty", reason = "unmaintained, use std::io::IsTerminal" }]
```

Запрет даёт собственную ошибку с вашей причиной:

```text
error[banned]: crate 'atty = 0.2.14' is explicitly banned
   ┌─ deny.toml:16:20
   │
16 │ deny = [{ crate = "atty", reason = "unmaintained, use std::io::IsTerminal" }]
   │                    ━━━━             ───────────────────────────────────── reason
   │                    │                 
   │                    banned here
   │
   ├ atty v0.2.14
     └── p10-audit-demo v0.1.0

bans FAILED
```

Уберите строку `deny` снова (или оставьте: когда `atty` исчезнет, она не помешает), удалите `atty` из проекта и обновите `regex`. Теперь все четыре проверки проходят:

```text
advisories ok, bans ok, licenses ok, sources ok
```

> 💡 **Совет:** `cargo deny check advisories` может игнорировать находку с **причиной**, записанной в файле, при код-ревью, где её видят все:
> `ignore = [{ id = "RUSTSEC-2021-0145", reason = "atty only checks for a terminal; replace by 2027-01" }]`. Используйте это как крайнюю меру и всегда с причиной и датой.

## 5. Автоматизируйте

Проверка, которую никто не запускает, ничего не защищает. Создайте `.github/workflows/security.yml` в корне репозитория:

```yaml
name: security

on:
  push:
  pull_request:
  schedule:
    - cron: "17 4 * * 1" # every Monday: new advisories appear even when your code does not change

permissions:
  contents: read

jobs:
  audit:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v7
      - name: Install the tools
        run: cargo install --locked cargo-audit cargo-deny
      - name: Known vulnerabilities (RustSec)
        run: cargo audit
      - name: Licenses, bans, sources and advisories
        run: cargo deny check
```

Настоящие раннеры GitHub Actions с вашего Mac не запустить, но синтаксис можно проверить локально с помощью `actionlint` (Урок 09.12): команда `actionlint .github/workflows/security.yml` ничего не печатает и завершается с кодом `0`. Три триггера означают: при каждой отправке, при каждом pull request и еженедельно. `permissions: contents: read` даёт токену задания только права на чтение, как Урок 09.12 советует для каждого рабочего процесса. (Есть и более быстрые варианты: действия `EmbarkStudios/cargo-deny-action` и `rustsec/audit-check` устанавливают готовые бинарные файлы вместо компиляции инструментов.)

## 6. Прежде чем добавить зависимость

Короткий чек-лист. Он занимает пять минут и предотвращает большинство проблем:

| Вопрос | Где смотреть |
|---|---|
| Нужна ли она мне? Могу ли я написать десять строк или использовать `std`? | Ваш собственный код. Каждый крейт — это поверхность атаки. |
| Имя точно верное? | Сравните имя и ссылку на репозиторий с документацией, которой вы пользовались (тайпсквоттинг). |
| Поддерживается ли и используется ли? | Страница на crates.io: свежий релиз, число загрузок, активность репозитория. |
| Кто её поддерживает? | Один человек или команда? Это крейт организации (например, `rust-lang`, `tokio-rs`)? |
| Что она тянет за собой? | `cargo tree -p <crate>` после добавления; чем меньше транзитивных крейтов, тем лучше. |
| Использует ли она `unsafe`, `build.rs` или процедурный макрос? | Просмотрите репозиторий. Скрипты сборки и макросы выполняются при компиляции. |
| Минимальны ли фичи? | `default-features = false` и включайте только нужное. |

Для критичных проектов есть инструменты, которые записывают «доверенный человек проверил эту версию этого крейта», например `cargo vet`; они опираются на те же идеи. Также обновляйтесь **регулярно**: проект, который понемногу обновляется каждую неделю, исправить проще, чем замороженный два года. После `cargo update` запускайте тесты.

## Частые ошибки

**1. Не коммитить `Cargo.lock` для программы.** Тогда две сборки одного коммита могут содержать разные версии зависимостей, а у `cargo audit` нет точного списка для проверки.

**2. Запускать инструменты один раз.** База бюллетеней меняется каждую неделю. Поставьте их в CI по расписанию.

**3. Игнорировать предупреждения.** «Не поддерживается» — сегодня предупреждение, а завтра уязвимость, которую никто не исправит. Запланируйте замену.

**4. Использовать `cargo update` без тестов.** Обновления могут изменить поведение даже внутри допустимого диапазона версий. Обновляйтесь в ветке, запускайте тесты и читайте список изменений важных крейтов.

**5. Зависимости, записанные как `*`.** `serde = "*"` значит «любая версия, даже будущая несовместимая или враждебная». `cargo-deny` запрещает это через `wildcards = "deny"`.

**6. `cargo install` без `--locked`** для инструментов: инструмент собирается с самыми новыми версиями собственных зависимостей, а не с проверенными.

## Попробуйте сами

1. Выполните `cargo tree` в одном из ваших прежних пакетов (например, `code/p10-hashing`). Сколько крейтов принесли несколько зависимостей?
2. Выполните там `cargo audit`. Находит ли он что-нибудь?
3. В демонстрационном пакете замените `regex = "1.5"` на `regex = "*"` в `Cargo.toml` и выполните `cargo deny check bans`. Что происходит? Потом верните как было.

Закончив, удалите демонстрацию: `rm -rf ~/projects/rust/code/p10-audit-demo`.

## Упражнения

### Упражнение 1 — Кто это использует? (легко)

После исправления демонстрации (`regex` обновлён, `atty` удалён) выясните, **какие крейты** зависят от `memchr`.

<details>
<summary>Подсказка</summary>

Обратное дерево: `cargo tree -i <crate>`.

</details>

<details>
<summary>Ответ</summary>

```bash
cargo tree -i memchr
```

```text
memchr v2.8.3
├── aho-corasick v1.1.5
│   ├── regex v1.13.2
│   │   └── p10-audit-demo v0.1.0 (/Users/ada/projects/rust/code/p10-audit-demo)
│   └── regex-automata v0.4.19
│       └── regex v1.13.2 (*)
├── regex v1.13.2 (*)
└── regex-automata v0.4.19 (*)
```

Почему это работает: `-i` (inverse, обратное) рисует дерево вверх ногами от выбранного крейта, поэтому вы видите каждый путь от своего пакета до него. `(*)` помечает ветку, уже показанную выше.

</details>

### Упражнение 2 — Игнорировать с причиной (средне)

Верните `atty` (`regex` можно оставить обновлённым) и сделайте так, чтобы `cargo deny check advisories` проходила **без** удаления `atty`: проигнорируйте два бюллетеня об `atty` с указанием причины. Затем объясните, почему это крайняя мера.

<details>
<summary>Подсказка</summary>

В таблице `[advisories]` используйте `ignore = [{ id = "RUSTSEC-…", reason = "…" }]`. Идентификаторы есть в выводе выше: `RUSTSEC-2021-0145` и `RUSTSEC-2024-0375`. Помните, что правило `[bans]` тоже запрещает `atty`; это упражнение только про `advisories`.

</details>

<details>
<summary>Ответ</summary>

```toml
[advisories]
yanked = "deny"
ignore = [
    { id = "RUSTSEC-2021-0145", reason = "atty is only used to check for a terminal; replace with IsTerminal by 2027-01" },
    { id = "RUSTSEC-2024-0375", reason = "same: atty, tracked in the backlog" },
]
```

```bash
cargo deny check advisories
```

```text
advisories ok
```

Почему это работает: игнорируемые бюллетени больше не сообщаются. Это крайняя мера, потому что проблема остаётся: крейт по-прежнему не поддерживается и некорректен. `reason` и дата делают решение видимым при код-ревью, чтобы оно не было забыто.

</details>

### Упражнение 3 — Тест политики (сложно)

Напишите интеграционный тест `tests/policy.rs`, который падает, если в `Cargo.lock` появляется крейт, который вы решили никогда не использовать. Список запрещённых имён — константа в тесте (начните с `atty` и `time`). Убедитесь, что тест проходит на исправленной демонстрации и падает, когда в проекте есть `atty`.

<details>
<summary>Подсказка</summary>

В файле блокировки на каждый пакет есть строка `name = "crate"`. `env!("CARGO_MANIFEST_DIR")` — путь к папке с `Cargo.toml`, известный на этапе компиляции. Прочитайте файл через `std::fs::read_to_string`.

</details>

<details>
<summary>Ответ</summary>

```rust
//! A policy test: fails if a crate we decided never to use appears in Cargo.lock.

const BANNED: &[&str] = &["atty", "time"];

#[test]
fn no_banned_crates_in_cargo_lock() {
    let lock = std::fs::read_to_string(concat!(env!("CARGO_MANIFEST_DIR"), "/Cargo.lock")).unwrap();
    for banned in BANNED {
        let line = format!("name = \"{banned}\"");
        assert!(!lock.lines().any(|l| l == line), "banned crate `{banned}` is in Cargo.lock");
    }
}
```

Когда в проекте есть `atty`, `cargo test --test policy` показывает:

```text
test no_banned_crates_in_cargo_lock ... FAILED
thread 'no_banned_crates_in_cargo_lock' (5319383) panicked at tests/policy.rs:10:9:
banned crate `atty` is in Cargo.lock
```

а после `cargo remove atty` показывает `test no_banned_crates_in_cargo_lock ... ok`.

Почему это работает: точное сравнение строки исключает ложные совпадения, например с крейтом `atty-extra`. Это маленькая самодельная версия проверки `bans`, и она показывает, что политика может быть обычным тестом. В настоящих проектах предпочитайте `cargo deny`, который к тому же понимает граф зависимостей.

</details>

## Вопросы для самопроверки

1. Что такое транзитивная зависимость?

<details>
<summary>Ответ</summary>

Зависимость одной из ваших зависимостей. Вы её не выбирали, но она часть вашей программы и вашей поверхности атаки. `cargo tree` показывает все.

</details>

2. `cargo audit` сообщает `Warning: unmaintained` и «No fixed upgrade is available!». Какие у вас варианты?

<details>
<summary>Ответ</summary>

Заменить крейт (лучше всего — на `std` или поддерживаемый крейт) либо проигнорировать предупреждение с записанной причиной и датой пересмотра. Версии, на которую можно обновиться, нет.

</details>

3. Почему запускать `cargo audit` нужно по расписанию, а не только при изменении кода?

<details>
<summary>Ответ</summary>

Потому что новые бюллетени публикуются для старого кода. Проект, который был чистым в прошлом месяце, на этой неделе может содержать известную уязвимость без единого изменения в вашем репозитории.

</details>

4. Назовите четыре проверки `cargo-deny`.

<details>
<summary>Ответ</summary>

`advisories`, `licenses`, `bans` и `sources`.

</details>

5. Что такое тайпсквоттинг и какая есть одна защита?

<details>
<summary>Ответ</summary>

Публикация вредоносного крейта с именем, очень близким к популярному, в расчёте на опечатки. Защита: копируйте строку зависимости из официальной документации, проверяйте репозиторий и число загрузок на crates.io и просматривайте diff `Cargo.toml` и `Cargo.lock` в каждом pull request.

</details>

## Итоги

- В вашей программе много кода других людей; цепочка поставок — часть вашей поверхности атаки.
- `cargo tree` показывает, от чего вы зависите; `Cargo.lock` фиксирует точные версии, и для программ его нужно коммитить.
- `cargo audit` находит в `Cargo.lock` известные уязвимости, неподдерживаемые и некорректные крейты; исправление — `cargo update -p` или замена крейта.
- `cargo deny check` обеспечивает политику: бюллетени (включая отозванные версии), лицензии, запреты и источники, настраиваемые в `deny.toml`.
- Запускайте оба в CI при каждой отправке и по еженедельному расписанию; игнорируйте находки только с причиной и датой.
- Перед добавлением крейта спросите: нужен ли он, верно ли имя, поддерживается ли, что он тянет за собой и выполняет ли код при сборке.

## Источники и что почитать

- [База бюллетеней RustSec](https://rustsec.org/) — бюллетени, на которых работает `cargo-audit`, с поиском по крейту.
- [cargo-audit на GitHub](https://github.com/rustsec/rustsec/tree/main/cargo-audit) — параметры, конфигурация (`audit.toml`) и использование в CI.
- [Книга cargo-deny](https://embarkstudios.github.io/cargo-deny/) — каждая опция `deny.toml` и четыре проверки.
- [The Cargo Book: Cargo.toml vs Cargo.lock](https://doc.rust-lang.org/cargo/guide/cargo-toml-vs-cargo-lock.html) — зачем и когда коммитить `Cargo.lock`.
- [The Cargo Book: cargo tree](https://doc.rust-lang.org/cargo/commands/cargo-tree.html) — все параметры `cargo tree`.
- [Книга cargo-vet](https://mozilla.github.io/cargo-vet/) — как записывать, кто проверил какую версию зависимости.

---

[🏠 Начало](../START-HERE.md) · [← 10.12 Безопасность памяти и проверка небезопасного кода](12-memory-safety-and-reviewing-unsafe.md) · [📋 Этап 10](README.md) · [10.14 Проект: зашифрованное хранилище секретов →](14-project-encrypted-secrets-vault.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
