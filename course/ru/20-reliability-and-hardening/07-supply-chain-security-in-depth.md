<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 20.6 Основы формальной верификации с Kani](06-formal-verification-with-kani.md) · [📋 Этап 20](README.md) · [20.8 Перегрузка, лимиты и обратное давление →](08-overload-limits-and-backpressure.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/20-reliability-and-hardening/07-supply-chain-security-in-depth.md) <!-- nav -->

# Урок 20.7 — Безопасность цепочки поставок подробно

> **Этап 20 — Надёжность и защита** · Урок 7 из 11 · ⏱ около 135 минут

## Что вы узнаете

- На какие вопросы о цепочке поставок `cargo audit` и `cargo deny` не отвечают и какие инструменты отвечают на них.
- Как записать, что человек действительно прочитал код зависимости, с помощью `cargo vet`.
- Как положить список зависимостей внутрь бинарного файла (`cargo auditable`) и написать SBOM (`cargo cyclonedx`).
- Что такое воспроизводимая сборка, как её проверить и как исправить самое частое расхождение.
- Как публиковать контрольные суммы и подпись, чтобы пользователи могли проверить, что файл действительно от вас.

## Перед началом

- Вы прошли [Урок 20.6 — Основы формальной верификации с Kani](06-formal-verification-with-kani.md).
- Вы знаете `cargo tree`, `Cargo.lock`, `cargo audit` и `cargo deny` из [Урока 10.13 — Безопасность цепочки поставок](../10-security-fundamentals/13-supply-chain-security.md). Этот урок начинается там, где закончился тот.
- Вы знаете хеши ([Урок 10.9](../10-security-fundamentals/09-cryptography-hashes-macs-and-randomness.md)), подписи ([Урок 10.10](../10-security-fundamentals/10-cryptography-encryption-and-signatures.md)), Docker ([Урок 11.12](../11-linux-shell-and-containers/12-containers-and-docker-basics.md)) и GitHub Actions ([Урок 09.12](../09-testing-debugging-benchmarking/12-continuous-integration.md)).
- Установите ещё три инструмента. Компиляция займёт несколько минут:

```bash
cargo install --locked cargo-vet cargo-auditable cargo-cyclonedx
cargo vet --version
cargo cyclonedx --version
```

```text
cargo-vet 0.10.3
cargo-cyclonedx-cyclonedx 0.5.10
```

(`cargo auditable --version` печатает версию самого Cargo, потому что `cargo auditable` передаёт каждую команду в Cargo.)

- Создайте учебный пакет. Как и в Уроке 10.13, он начинается со **старого** `regex` с известной уязвимостью:

```bash
cd ~/projects/rust
cargo new code/p20-supply-demo
cd code/p20-supply-demo
cargo add regex@1.5
cargo update -p regex --precise 1.5.4
```

Замените `src/main.rs`:

```rust
use regex::Regex;

fn main() {
    let date = Regex::new(r"^\d{4}-\d{2}-\d{2}$").unwrap();
    for text in ["2026-11-18", "18.11.2026"] {
        println!("{text}: date? {}", date.is_match(text));
    }
}
```

`cargo run` печатает:

```text
2026-11-18: date? true
18.11.2026: date? false
```

## Новые слова

| Слово | Значение |
|---|---|
| **аудит** (audit, крейта) | Человек читает код одной версии крейта и записывает, что он проверил. |
| **критерии** (criteria) | В `cargo vet` — обещание, которое даёт аудит, например `safe-to-deploy`: «нет серьёзной дыры в безопасности для программ, которые получают недоверенный ввод». |
| **исключение** (exemption) | В `cargo vet` — версия крейта, которая пока принята *без* аудита. Исключения — это список отложенной работы. |
| **дельта-аудит** (delta audit) | Аудит *разницы* между двумя версиями, когда старая версия уже прошла аудит. |
| **SBOM** (Software Bill of Materials, спецификация состава ПО) | Машиночитаемый список всех компонентов программы: имена, версии, лицензии, хеши. Как список ингредиентов на упаковке еды. |
| **CycloneDX / SPDX** | Два распространённых стандартных формата SBOM (SPDX: Software Package Data Exchange, обмен данными о программных пакетах). |
| **purl** (package URL, URL пакета; URL — Uniform Resource Locator, адрес в интернете) | Стандартное текстовое имя пакета, например `pkg:cargo/regex@1.13.2`. Инструменты используют его, чтобы сопоставлять записи SBOM с бюллетенями безопасности. |
| **воспроизводимая сборка** (reproducible build) | Сборка, в которой одинаковый исходный код, инструменты и настройки всегда дают файл, одинаковый до байта. |
| **контрольная сумма** (checksum) | Хеш файла, опубликованный рядом с ним, чтобы загрузку можно было проверить. Здесь: SHA-256 (Secure Hash Algorithm, алгоритм безопасного хеширования, 256 бит). |
| **происхождение** (provenance) | Подписанные сведения о том, *как* собран файл: из какого репозитория, какого коммита, какой системой сборки. |
| **SLSA** (Supply-chain Levels for Software Artifacts, уровни цепочки поставок для программных артефактов) | Рамка, которая описывает уровни доверия к тому, как собрано ПО: от «никаких гарантий» до «собрано на защищённой системе с подписанным происхождением». |
| **артефакт** (artifact) | Файл, который производит сборка и который вы поставляете: бинарный файл, архив, SBOM. |

## 1. Четыре вопроса, которые ещё открыты

После Урока 10.13 ваш CI (Continuous Integration, непрерывная интеграция: сервер, который запускает ваши проверки при каждой отправке) знает, что в `Cargo.lock` нет *известных* уязвимостей и что лицензии и источники соответствуют вашим правилам. Открытыми остаются четыре вопроса:

| Вопрос | Почему это важно | Инструмент в этом уроке |
|---|---|---|
| Прочитал ли человек код, от которого мы зависим? | У нового вредоносного крейта ещё нет бюллетеня. | `cargo vet` |
| Что именно внутри бинарного файла, который мы выпустили в прошлом году? | Lock-файл старого релиза мог потеряться, а бинарный файл всё ещё работает на серверах. | `cargo auditable`, SBOM |
| Действительно ли этот бинарный файл собран из этого исходного кода? | Взломанная машина сборки может добавить код, которого нет в Git. | воспроизводимые сборки |
| Действительно ли этот файл от нас? | Зеркало для загрузки или атакующий посередине (man-in-the-middle) может его подменить. | контрольные суммы, подписи, происхождение |

Отвечать на них вы будете в другом порядке: сначала — что внутри бинарного файла, потому что в демо есть уязвимый `regex`, который нужно найти.

## 2. Что внутри бинарного файла: `cargo auditable`

`cargo audit` читает `Cargo.lock`. Но на сервере у вас есть только бинарный файл. Соберите демо обычным способом и попросите `cargo audit` посмотреть на **бинарный файл**:

```bash
cargo build --release
cargo audit bin target/release/p20-supply-demo
```

```text
warning: target/release/p20-supply-demo was not built with 'cargo auditable', the report will be incomplete (4 dependencies recovered)
Crate:     regex
Version:   1.5.4
Title:     Regexes with large repetitions on empty sub-expressions take a very long time to parse
ID:        RUSTSEC-2022-0013
Solution:  Upgrade to >=1.5.5 (patched)

error: 1 vulnerability found in target/release/p20-supply-demo
```

(Сокращено: строки загрузки базы бюллетеней и некоторые поля бюллетеня опущены.) Инструменту пришлось **угадывать** зависимости по тексту, оставшемуся в бинарном файле, например по путям к файлам в сообщениях о панике. Он нашёл 4 крейта; здесь это случайно все, но крейт, который не оставил такого текста в бинарном файле, остаётся невидимым, поэтому отчёт может быть неполным. Теперь соберите через `cargo auditable` (вывод сокращён):

```bash
cargo auditable build --release
cargo audit bin target/release/p20-supply-demo
```

```text
       Found 'cargo auditable' data in target/release/p20-supply-demo (5 dependencies)
Crate:     regex
Version:   1.5.4
...
error: 1 vulnerability found in target/release/p20-supply-demo
```

**Что происходит:**

1. `cargo auditable build` запускает обычный `cargo build`, но добавляет одну маленькую деталь: дерево зависимостей (имена, версии, источники) в виде сжатого JSON (JavaScript Object Notation, текстовый формат данных) в специальной секции бинарного файла. Код не меняется и работает точно так же.
2. `cargo audit bin` находит эту секцию и проверяет каждый крейт из списка по базе RustSec — так же, как проверяет `Cargo.lock`. Теперь список гарантированно полный: 5 пакетов (демо и 4 его зависимости).
3. Код возврата — `1` из-за уязвимости, так что его может использовать скрипт.

Теперь исправьте проблему, пересоберите и проверьте снова:

```bash
cargo update -p regex
cargo auditable build --release
cargo audit bin target/release/p20-supply-demo
```

```text
       Found 'cargo auditable' data in target/release/p20-supply-demo (6 dependencies)
```

Находок нет, код возврата `0`. (У нового `regex` на одну зависимость больше, `regex-automata`, поэтому пакетов 6.)

> 🔒 **Безопасность:** бюллетени для старых версий публикуются каждую неделю. Когда список зависимостей есть внутри каждого бинарного файла, вы можете проверять бинарные файлы, которые **работают в продакшене сегодня**, даже через годы после сборки и даже если никто не знает, из какого они коммита. Некоторые дистрибутивы Linux по этой причине собирают все свои пакеты на Rust через `cargo auditable`.

> ⚡ **Производительность:** дополнительные данные занимают от нескольких сотен байт до нескольких килобайт, и программа никогда их не читает. Затрат во время работы нет.

## 3. Спецификация состава: SBOM

Данные внутри бинарного файла предназначены для инструментов, которые понимают `cargo auditable`. Клиенты, команды безопасности и сканеры для других языков ждут **стандартный** документ — SBOM. Создайте его:

```bash
cargo cyclonedx --format json
```

Команда ничего не печатает и записывает `p20-supply-demo.cdx.json` рядом с `Cargo.toml`. Файл длинный (около 250 строк). Вот один компонент из его списка `components`, в сокращённом виде:

```json
{
  "type": "library",
  "name": "regex",
  "version": "1.13.2",
  "scope": "required",
  "hashes": [
    {
      "alg": "SHA-256",
      "content": "9807af0e638c12e753aa1b4e601791bf2a8f71274023c8bbeec12fc149950857"
    }
  ],
  "licenses": [
    {
      "expression": "MIT OR Apache-2.0"
    }
  ],
  "purl": "pkg:cargo/regex@1.13.2"
}
```

**Что происходит:**

1. `cargo cyclonedx` читает те же сведения, которые показывает `cargo tree`, и записывает их в формате **CycloneDX** (другой распространённый формат — **SPDX**; многие инструменты читают оба).
2. У каждого компонента есть имя, версия, выражение лицензии и хеш SHA-256 загруженного файла крейта (та же контрольная сумма, что и в `Cargo.lock`).
3. `purl` — имя, которое работает для любых языков: сканер, который ничего не знает о Cargo, всё равно может сопоставить `pkg:cargo/regex@1.13.2` с базой уязвимостей.
4. Секция `dependencies` в конце файла записывает, кто от кого зависит, так что SBOM хранит дерево, а не только плоский список.
5. Dev-зависимости (только для тестов) не указываются: они не входят в то, что вы поставляете.

Когда объявляют новую уязвимость, организация, у которой есть SBOM для каждого релиза, может за минуты ответить на вопрос «затронуты ли мы и где?», просто поискав по документам. SBOM публикуется рядом с релизом (раздел 6), и для каждого релиза создаётся новый.

> 📏 **Соглашение:** создавайте SBOM в CI из того же коммита и lock-файла, что и бинарный файл, никогда вручную. SBOM, который старше бинарного файла, неверен.

## 4. Читал ли кто-нибудь этот код? `cargo vet`

`cargo audit` знает только о проблемах, которые кто-то уже нашёл и сообщил. У совсем новой вредоносной версии бюллетеня нет. Единственная защита — чтобы **человек** посмотрел на код до того, как он попадёт в ваш продукт. `cargo vet` не читает код за вас; он ведёт запись о том, **кто что прочитал**, и проваливает CI, когда приходит что-то непрочитанное.

Запустите его в демо (`regex` уже обновлён):

```bash
cargo vet init
cargo vet
```

```text
Vetting Succeeded (5 exempted)
```

`init` создал папку `supply-chain/` с тремя файлами:

| Файл | Содержимое |
|---|---|
| `config.toml` | Настройки и **исключения**: по одной записи для каждой зависимости, которая есть у вас сегодня. |
| `audits.toml` | Аудиты, которые пишет ваша команда. Сначала пустой. |
| `imports.lock` | Аудиты, импортированные из других организаций, зафиксированные так же, как в `Cargo.lock`. |

Одно исключение из `config.toml` выглядит так: `[[exemptions.regex]]`, затем `version = "1.13.2"` и `criteria = "safe-to-deploy"`.

**Что происходит:**

1. В первый день вы не можете прочитать все существующие зависимости. Поэтому `init` принимает всё, что у вас есть **сейчас**, как исключения, и проверка проходит.
2. С этого момента каждый **новый** крейт или новая версия требует аудита. Исключения — это отложенная работа, которую вы постепенно сокращаете.
3. `criteria = "safe-to-deploy"` — более строгий из двух встроенных критериев. Он означает: «этот код не добавит серьёзной дыры в безопасности в программу, которая обрабатывает недоверенный ввод». Более слабый, `safe-to-run`, говорит только, что код не вредит машине, на которой вы его запускаете (этого достаточно для инструмента на вашем компьютере, но не для кода в продакшене).

Теперь добавьте новую зависимость и снова запустите проверку:

```bash
cargo add itoa@1
cargo vet
```

```text
Vetting Failed!

1 unvetted dependencies:
  itoa:1.0.19 missing ["safe-to-deploy"]

recommended audits for safe-to-deploy:
    Command                        Publisher  Used By          Audit Size
    cargo vet inspect itoa 1.0.19  dtolnay    p20-supply-demo  1074 lines
      NOTE: cargo vet import google would reduce this to a 733-line diff

estimated audit backlog: 1074 lines

Use |cargo vet certify| to record the audits.
```

Код возврата не `0`, поэтому CI останавливается. Инструмент говорит, сколько нужно прочитать (1074 строки), и даёт подсказку: Google публикует свои аудиты, и они покрывают более старую версию `itoa`.

**Импортируйте аудиты команды, которой вы доверяете:**

```bash
cargo vet import google
cargo vet
```

```text
Vetting Failed!
...
    cargo vet diff itoa 1.0.15 1.0.19  dtolnay    p20-supply-demo  9 files changed, 441 insertions(+), 292 deletions(-)

estimated audit backlog: 733 lines
```

Аудиторы Google уже прочитали `itoa` до версии `1.0.15`. Вам нужно прочитать только **разницу** до `1.0.19`. Теперь у вас два честных варианта:

1. **Использовать проверенную версию.** `cargo update -p itoa --precise 1.0.15`, и `cargo vet` печатает `Vetting Succeeded (1 fully audited, 5 exempted)`.
2. **Прочитать разницу и записать это.** `cargo vet diff itoa 1.0.15 1.0.19` показывает разницу (в браузере или, с `--mode local`, в терминале). Ищите новый `unsafe`-код, новый `build.rs`, новые зависимости, доступ к файлам, сети или процессам. Когда вы убедились, запишите аудит:

```bash
cargo vet certify itoa 1.0.15 1.0.19 --criteria safe-to-deploy
cargo vet
```

`certify` показывает критерии и просит их подтвердить, берёт ваше имя из настроек Git и записывает в `audits.toml` вот это:

```toml
[[audits.itoa]]
who = "Ada Lovelace <ada@example.com>"
criteria = "safe-to-deploy"
delta = "1.0.15 -> 1.0.19"
notes = "Read the whole diff. The unsafe blocks moved into one helper; no build script, no new dependencies, no file or network access."
```

```text
Vetting Succeeded (1 fully audited, 5 exempted)
```

(Заметку добавляют через `--notes "..."`; для следующего читателя это самая полезная часть.) Папка `supply-chain/` идёт в Git. Тогда аудит проходит ревью в pull request, как код, а другие команды могут импортировать и **ваши** аудиты.

> ⚠️ **Частая ошибка:** запускать `cargo vet certify`, не читая код, только чтобы CI стал зелёным. Тогда запись говорит «человек это проверил», что неправда, а другие команды могут на неё положиться. Если времени нет, оставьте исключение: это честно.

Снова удалите `itoa` (`cargo remove itoa`); демо его не использует.

## 5. Воспроизводимые сборки

Бинарный файл **воспроизводим**, если любой, кто возьмёт тот же исходный код, тот же тулчейн и те же настройки, получит файл, одинаковый до байта. Тогда вторая, независимая машина может пересобрать релиз и сравнить хеши. Если атакующий изменил официальную машину сборки, хеши различаются, и атака становится видна.

Rust по умолчанию близок к воспроизводимости, но мелкие детали попадают в бинарный файл. Самая частая — **пути к файлам**: сообщения о панике содержат путь к исходному файлу, а для зависимостей этот путь лежит внутри `CARGO_HOME` (папки, где Cargo хранит загруженные крейты, обычно `~/.cargo`).

Проверьте это в чистом контейнере Linux. Сохраните этот скрипт как `repro-check.sh` в папке проекта и сделайте его исполняемым (`chmod +x repro-check.sh`):

```bash
#!/usr/bin/env bash
# Builds the project in different places and prints a short hash of each binary.
set -euo pipefail

build() {
    local dir="$1" cargo_home="$2" flags="$3"
    mkdir -p "$dir"
    cp -r /src/Cargo.toml /src/Cargo.lock /src/src "$dir"/
    cd "$dir"
    CARGO_HOME="$cargo_home" RUSTFLAGS="$flags" cargo build --release --locked --quiet
    echo "$(sha256sum target/release/p20-supply-demo | cut -c1-16)  $dir (CARGO_HOME=$cargo_home)"
}

echo "== same tools, two folders"
build /build/one /usr/local/cargo ""
build /build/two /usr/local/cargo ""
echo "== another CARGO_HOME"
build /build/three /opt/other-cargo ""
echo "== another CARGO_HOME, paths remapped"
build /build/four /usr/local/cargo "--remap-path-prefix=/usr/local/cargo=/cargo"
build /build/five /opt/other-cargo "--remap-path-prefix=/opt/other-cargo=/cargo"
```

Запустите его в официальном образе Rust. Папка проекта подключена только для чтения (`:ro`), так что контейнер не может её изменить:

```bash
docker run --rm -v "$PWD":/src:ro rust:1.100 /src/repro-check.sh
```

```text
== same tools, two folders
2ed3df37d40119d4  /build/one (CARGO_HOME=/usr/local/cargo)
2ed3df37d40119d4  /build/two (CARGO_HOME=/usr/local/cargo)
== another CARGO_HOME
cfa62cdcf4bef64a  /build/three (CARGO_HOME=/opt/other-cargo)
== another CARGO_HOME, paths remapped
8e54709e3a080b86  /build/four (CARGO_HOME=/usr/local/cargo)
8e54709e3a080b86  /build/five (CARGO_HOME=/opt/other-cargo)
```

**Что происходит:**

1. `build` копирует только исходный код (`Cargo.toml`, `Cargo.lock`, `src`) в новую папку, собирает релиз с `--locked` (ровно те версии, что в lock-файле) и печатает первые 16 шестнадцатеричных цифр хеша SHA-256.
2. **Две папки, одни инструменты: одинаково.** Папка проекта не попадает в релизный бинарный файл, потому что Cargo записывает пути к вашим исходникам относительно проекта (`src/main.rs`).
3. **Другой `CARGO_HOME`: по-разному.** Бинарный файл содержит тексты вроде `/usr/local/cargo/registry/src/index.crates.io-…/regex-1.13.2/src/…` (их видно через `strings`). Другая папка загрузок даёт другие байты.
4. **С подменой путей: снова одинаково.** `--remap-path-prefix=FROM=TO` велит компилятору писать `TO` везде, где путь начинается с `FROM`. Обе сборки теперь содержат `/cargo/registry/…`, поэтому совпадают друг с другом (но не с первыми двумя, где остались старые пути).
5. Ваши хеши не совпадут с этими: они зависят от точного тулчейна и целевой платформы. В этом и смысл: воспроизводимость означает, что **одинаковые входные данные** дают одинаковый результат.

Входные данные, которые нужно зафиксировать для воспроизводимого релиза:

| Входные данные | Как зафиксировать |
|---|---|
| Версия тулчейна | Файл `rust-toolchain.toml` с `channel = "1.100.0"` или закреплённый тег образа Docker. |
| Версии зависимостей | Закоммитить `Cargo.lock`, собирать с `--locked`. |
| Целевая платформа и настройки | Тот же `--target`, тот же профиль, те же `RUSTFLAGS`. |
| Пути | `--remap-path-prefix` для `CARGO_HOME` и папки проекта. |
| Метки времени | Сам Rust их не добавляет; для инструментов, которые добавляют, задайте `SOURCE_DATE_EPOCH` (фиксированное время в секундах с 1970 года). |

Отладочные сборки (`cargo build` без `--release`) не воспроизводимы между папками: их отладочная информация содержит полные пути. Сравнивайте релизные сборки.

## 6. Контрольные суммы и подписи

Пользователи скачивают ваш бинарный файл со страницы релиза или с зеркала. Их защищают две вещи:

- **Файл контрольных сумм** (`SHA256SUMS`) показывает, что загрузка полная и не изменена.
- **Подпись** файла контрольных сумм показывает, что суммы опубликовал владелец ключа, а не тот, кто управляет зеркалом.

Подписи вы знаете из Урока 10.10. Практичный инструмент, который уже есть на каждом Mac и в каждом Linux, — `ssh-keygen`: он умеет подписывать любые файлы, а не только входить на серверы. Создайте ключ **только для релизов**, внутри проекта, и не пускайте его в Git:

```bash
mkdir -p dist keys
echo "keys/" >> .gitignore
cp target/release/p20-supply-demo dist/
ssh-keygen -t ed25519 -N "" -C "p20-supply-demo release key" -f keys/release_key
cd dist
shasum -a 256 p20-supply-demo > SHA256SUMS
ssh-keygen -Y sign -f ../keys/release_key -n file SHA256SUMS
```

```text
Signing file SHA256SUMS
Write signature to SHA256SUMS.sig
```

**Что происходит:**

1. `ssh-keygen -t ed25519 -f keys/release_key` создаёт пару ключей: закрытый ключ `release_key` и открытый ключ `release_key.pub`. `-N ""` означает «без парольной фразы», что допустимо только в этом упражнении.
2. `shasum -a 256` (в Linux: `sha256sum`) пишет по одной строке на файл: хеш и имя файла.
3. `ssh-keygen -Y sign -n file` подписывает `SHA256SUMS` и записывает `SHA256SUMS.sig`. `-n file` — это **пространство имён** подписи: метка внутри подписи, чтобы подпись, сделанную для файлов, нельзя было использовать повторно, например как подпись коммитов Git.

Пользователю нужен ваш открытый ключ, который вы публикуете (на сайте, в README). Он записывает его с именем в файл `allowed_signers`, а затем проверяет подпись и контрольные суммы:

```bash
echo "release@p20-supply-demo $(cat ../keys/release_key.pub)" > allowed_signers
ssh-keygen -Y verify -f allowed_signers -I release@p20-supply-demo -n file -s SHA256SUMS.sig < SHA256SUMS
shasum -a 256 -c SHA256SUMS
```

```text
Good "file" signature for release@p20-supply-demo with ED25519 key SHA256:h3w2qWMyzIHM51bP6ydIqxdpAhWCs9vuqS8+y8aSNTQ
p20-supply-demo: OK
```

Измените один символ в `SHA256SUMS` и снова проверьте подпись:

```text
Signature verification failed: incorrect signature
Could not verify signature.
```

Код возврата — `255`. Атакующий, который подменяет бинарный файл, должен изменить и контрольную сумму, а тогда подпись больше не совпадает.

> 🔒 **Безопасность:** закрытый ключ теперь — самый ценный секрет проекта (Урок 10.8). В настоящих проектах он хранится в хранилище секретов CI или в аппаратном устройстве, защищён парольной фразой и никогда не лежит в папке проекта разработчика. Многие проекты совсем обходятся без долгоживущих ключей благодаря подписи **без ключа** (keyless, проект Sigstore): система сборки на несколько минут подтверждает свою личность и получает короткоживущий сертификат.

**Происхождение.** Подпись говорит, *кто* опубликовал файл. **Происхождение** (provenance) говорит, *как* он собран: репозиторий, коммит, workflow, машина сборки — и всё это подписано системой сборки. Рамка **SLSA** описывает уровни таких гарантий. В GitHub действие `actions/attest-build-provenance` создаёт подписанное происхождение для артефакта, а пользователи проверяют его командой `gh attestation verify <file> --repo <owner>/<repo>`. crates.io в том же духе поддерживает **доверенную публикацию** (trusted publishing): workflow в GitHub публикует крейт без долгоживущего токена API (Application Programming Interface, программный интерфейс), который можно украсть.

## 7. Всё вместе в одном workflow релиза

Эти части должны жить в CI, чтобы выполняться при каждом релизе, и никому не нужно было о них помнить. Сохраните это как `.github/workflows/release.yml`:

```yaml
name: release

on:
  push:
    tags: ["v*"]

permissions:
  contents: read

jobs:
  build:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      id-token: write      # lets the job prove its identity for the attestation
      attestations: write  # lets the job store the attestation
    steps:
      - uses: actions/checkout@v7
      - name: Install the tools
        run: cargo install --locked cargo-vet cargo-auditable cargo-cyclonedx
      - name: Every dependency is vetted
        run: cargo vet --locked
      - name: Build with the dependency list inside
        run: cargo auditable build --release --locked
        env:
          RUSTFLAGS: --remap-path-prefix=${{ github.workspace }}=/src --remap-path-prefix=/home/runner/.cargo=/cargo
      - name: Bill of materials
        run: cargo cyclonedx --format json
      - name: Checksums
        run: |
          cp target/release/p20-supply-demo .
          sha256sum p20-supply-demo p20-supply-demo.cdx.json > SHA256SUMS
      - name: Signed provenance
        uses: actions/attest-build-provenance@v4
        with:
          subject-path: p20-supply-demo
      - uses: actions/upload-artifact@v7
        with:
          name: release
          path: |
            p20-supply-demo
            p20-supply-demo.cdx.json
            SHA256SUMS
```

`actionlint .github/workflows/release.yml` ничего не печатает и завершается с `0`. Раннеры GitHub нельзя запустить с вашего Mac, поэтому сам запуск здесь не показан.

**Что происходит:**

1. Workflow запускается, когда вы отправляете тег вроде `v0.2.0` (теги: Урок 01.9). Только это задание получает дополнительные права `id-token: write` (короткоживущий токен личности для подписи без ключа) и `attestations: write`.
2. `cargo vet --locked` останавливает релиз, если какая-то зависимость не прошла аудит; `--locked` означает, что во время сборки он не загружает новые импортированные аудиты.
3. Сборка встраивает список зависимостей и подменяет оба пути: папку проекта (`github.workspace`) и папку Cargo на раннере. SBOM, контрольные суммы и происхождение получаются из одного и того же checkout.

## Частые ошибки

**1. SBOM, сделанный один раз вручную.** Он описывает старое состояние. Создавайте его в CI для каждого релиза.

**2. Подтверждение без чтения.** `cargo vet` записывает обещание человека. Ложная запись хуже исключения.

**3. Сравнение отладочных сборок** при проверке воспроизводимости. Используйте релизные сборки с `--locked`, тем же тулчейном и подменёнными путями.

**4. Контрольные суммы без подписи.** Если атакующий управляет страницей загрузки, он подменит и бинарный файл, **и** `SHA256SUMS`. Контрольная сумма помогает только против испорченных загрузок; против атакующих помогает подпись.

**5. Ключ подписи в репозитории.** Любой, кто может читать репозиторий, сможет подписывать релизы. Храните ключ в хранилище секретов или используйте подпись без ключа.

## Попробуйте сами

1. Запустите `cargo audit bin` для релизного бинарного файла одного из ваших прежних сервисов (например, из Этапа 16): сначала собранного обычным способом, потом через `cargo auditable build --release`. Сколько зависимостей он находит в каждом случае?
2. Запустите `cargo cyclonedx --format json` в том же проекте и посчитайте компоненты: `jq '.components | length' <name>.cdx.json`.
3. Запустите там `cargo vet init` и посмотрите, со скольких исключений вы начинаете. Это число — ваш список отложенных аудитов.

## Упражнения

### Упражнение 1 — Прочитать SBOM (легко)

Напечатайте по одной строке на каждый компонент SBOM демо: `name@version`, два пробела и выражение лицензии.

<details>
<summary>Подсказка</summary>

`jq -r` печатает «сырой» текст. Внутри строки `\(.name)` вставляет поле. Лицензия лежит в `.licenses[0].expression`.

</details>

<details>
<summary>Ответ</summary>

```bash
jq -r '.components[] | "\(.name)@\(.version)  \(.licenses[0].expression)"' p20-supply-demo.cdx.json
```

```text
aho-corasick@1.1.5  Unlicense OR MIT
memchr@2.8.3  Unlicense OR MIT
regex-automata@0.4.19  MIT OR Apache-2.0
regex-syntax@0.8.12  MIT OR Apache-2.0
regex@1.13.2  MIT OR Apache-2.0
```

Почему это работает: `.components[]` проходит по массиву по одному элементу, а шаблон строки собирает из каждого элемента одну строку. Тот же однострочный запрос работает с любым SBOM в формате CycloneDX, для любого языка.

</details>

### Упражнение 2 — Изменённый бинарный файл (средне)

В `dist/` допишите в бинарный файл один байт (`printf 'x' >> p20-supply-demo`). Затем выполните обе проверки из раздела 6. Какая из них падает и почему вторая всё ещё проходит?

<details>
<summary>Подсказка</summary>

Подпись покрывает `SHA256SUMS`, а не сам бинарный файл.

</details>

<details>
<summary>Ответ</summary>

```text
p20-supply-demo: FAILED
shasum: WARNING: 1 computed checksum did NOT match
```

Проверка подписи по-прежнему печатает `Good "file" signature …`.

Почему это работает: подпись доказывает, что `SHA256SUMS` — ровно то, что подписал владелец ключа, и это так. Контрольная сумма доказывает, что бинарный файл совпадает с подписанным хешем, а это не так. Нужны **обе** проверки: сначала подпись (настоящий ли список?), потом контрольные суммы (совпадают ли файлы со списком?). После упражнения снова соберите или скопируйте бинарный файл.

</details>

### Упражнение 3 — Тест политики для SBOM (сложно)

Напишите интеграционный тест `tests/sbom_policy.rs`, который читает `p20-supply-demo.cdx.json` и падает, если у компонента нет хеша SHA-256, нет выражения лицензии или его имя есть в списке запрещённых (`atty`, `openssl`). Проверьте, что он падает, когда вы добавляете `memchr` в список запрещённых.

<details>
<summary>Подсказка</summary>

Добавьте `serde_json` как dev-зависимость (`cargo add --dev serde_json@1`) и разберите файл в `serde_json::Value`. `value["components"].as_array()` даёт список; обращение по отсутствующему ключу возвращает `Value::Null` вместо паники, поэтому `c["licenses"][0]["expression"].as_str()` равно `None`, если чего-то не хватает.

</details>

<details>
<summary>Ответ</summary>

```rust
//! Checks the bill of materials that `cargo cyclonedx` wrote next to Cargo.toml.

use serde_json::Value;

const BANNED: &[&str] = &["atty", "openssl"];

#[test]
fn every_component_has_a_hash_a_license_and_is_allowed() {
    let path = concat!(env!("CARGO_MANIFEST_DIR"), "/p20-supply-demo.cdx.json");
    let text = std::fs::read_to_string(path).expect("run `cargo cyclonedx --format json` first");
    let sbom: Value = serde_json::from_str(&text).unwrap();

    let components = sbom["components"].as_array().expect("no components list");
    assert!(!components.is_empty(), "the SBOM lists no components");

    for c in components {
        let name = c["name"].as_str().unwrap();
        assert!(!BANNED.contains(&name), "banned crate `{name}` is in the SBOM");

        let has_sha256 = c["hashes"]
            .as_array()
            .is_some_and(|hashes| hashes.iter().any(|h| h["alg"] == "SHA-256"));
        assert!(has_sha256, "`{name}` has no SHA-256 hash");

        let license = c["licenses"][0]["expression"].as_str();
        assert!(license.is_some(), "`{name}` has no license expression");
    }
}
```

`cargo test --test sbom_policy` проходит. С `"memchr"` в `BANNED`:

```text
thread 'every_component_has_a_hash_a_license_and_is_allowed' (5994724) panicked at tests/sbom_policy.rs:18:9:
banned crate `memchr` is in the SBOM
```

Почему это работает: SBOM — это обычный JSON, поэтому обычный тест может проверять для него правила. `serde_json` не появляется в SBOM, хотя тест его использует: dev-зависимости не входят в поставляемую программу. В CI запускайте `cargo cyclonedx` перед `cargo test`.

</details>

## Вопросы для самопроверки

1. `cargo audit` зелёный. Почему в вашей сборке всё равно может быть вредоносный крейт?

<details>
<summary>Ответ</summary>

`cargo audit` знает только о проблемах, о которых **сообщили**. У новой вредоносной версии бюллетеня ещё нет. Против неё помогает только ревью людьми (записанное через `cargo vet`) или очень короткий список зависимостей.

</details>

2. Что `cargo vet init` кладёт в `config.toml` и зачем?

<details>
<summary>Ответ</summary>

Исключение для каждой зависимости, которая есть у вас сейчас. В первый день нельзя провести аудит всего, поэтому сегодняшнее состояние принимается, а каждый новый крейт или версия с этого момента требует аудита. Исключения — отложенная работа, которую нужно сокращать.

</details>

3. Чем `cargo auditable` отличается от SBOM?

<details>
<summary>Ответ</summary>

`cargo auditable` хранит список зависимостей **внутри** бинарного файла, для инструментов, знающих Rust, таких как `cargo audit bin`. SBOM — отдельный документ в стандартном формате (CycloneDX, SPDX), который могут читать инструменты для любого языка. Оба отвечают на вопрос «что внутри этой программы?».

</details>

4. Две релизные сборки одного коммита на двух машинах дают разные хеши. Назовите две частые причины.

<details>
<summary>Ответ</summary>

Другой `CARGO_HOME` или путь к проекту, попавший в бинарный файл (исправляется через `--remap-path-prefix`); другая версия тулчейна (исправляется через `rust-toolchain.toml` или закреплённый образ); другие версии зависимостей (закоммитить `Cargo.lock`, собирать с `--locked`); другая целевая платформа или другие `RUSTFLAGS`.

</details>

5. Зачем подписывать `SHA256SUMS`, а не просто публиковать его?

<details>
<summary>Ответ</summary>

Тот, кто может подменить бинарный файл на странице загрузки, может подменить и файл контрольных сумм. Подпись можно сделать только закрытым ключом, поэтому изменённый файл контрольных сумм будет обнаружен.

</details>

6. Что происхождение добавляет к подписи?

<details>
<summary>Ответ</summary>

Подпись говорит, кто опубликовал файл. Происхождение говорит, как он собран: из какого репозитория и коммита, каким workflow, на какой системе сборки, — и система сборки подписывает это утверждение.

</details>

## Итоги

- `cargo audit` и `cargo deny` проверяют известные проблемы и правила; они не могут сказать, читал ли кто-нибудь код.
- `cargo vet` записывает аудиты людей (ваши и импортированные) и падает на новых крейтах без аудита; исключения — честный список отложенной работы.
- `cargo auditable build` кладёт список зависимостей в бинарный файл, поэтому `cargo audit bin` может проверять уже развёрнутые бинарные файлы.
- `cargo cyclonedx` пишет SBOM — стандартный список компонентов, версий, лицензий и хешей, который создаётся в CI для каждого релиза.
- Воспроизводимая сборка даёт одинаковые байты из одинаковых входных данных: закрепите тулчейн, используйте `--locked`, подменяйте пути, сравнивайте релизные сборки.
- Публикуйте `SHA256SUMS` и подпись к нему; сначала проверяйте подпись, потом контрольные суммы. Происхождение и подпись без ключа добавляют ответ на вопрос «как и где это собрано».

## Источники и что почитать

- [cargo-vet book](https://mozilla.github.io/cargo-vet/) — понятия, критерии, импорт аудитов и все команды.
- [cargo-auditable на GitHub](https://github.com/rust-secure-code/cargo-auditable) — как встраивается список зависимостей и какие инструменты его читают.
- [cargo-cyclonedx на GitHub](https://github.com/CycloneDX/cyclonedx-rust-cargo) — параметры `cargo cyclonedx`.
- [CycloneDX](https://cyclonedx.org/) и [SPDX](https://spdx.dev/) — два стандарта SBOM.
- [Reproducible Builds](https://reproducible-builds.org/) — почему важна воспроизводимость, и `SOURCE_DATE_EPOCH`.
- [rustc command-line arguments](https://doc.rust-lang.org/rustc/command-line-arguments.html) — `--remap-path-prefix` и другие флаги компилятора.
- [Руководство ssh-keygen](https://man.openbsd.org/ssh-keygen) — `-Y sign`, `-Y verify` и формат `allowed_signers`.
- [SLSA](https://slsa.dev/) и [Sigstore](https://www.sigstore.dev/) — уровни происхождения сборок и подпись без ключа.
- [GitHub: artifact attestations](https://docs.github.com/en/actions/security-for-github-actions/using-artifact-attestations/using-artifact-attestations-to-establish-provenance-for-builds) — подписанное происхождение в GitHub Actions.
- [RFC 3691: trusted publishing for crates.io](https://rust-lang.github.io/rfcs/3691-trusted-publishing-cratesio.html) — проектное предложение (RFC: Request for Comments, «запрос комментариев») о публикации крейтов без долгоживущих токенов.

---

[🏠 Начало](../START-HERE.md) · [← 20.6 Основы формальной верификации с Kani](06-formal-verification-with-kani.md) · [📋 Этап 20](README.md) · [20.8 Перегрузка, лимиты и обратное давление →](08-overload-limits-and-backpressure.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
