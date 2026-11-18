<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 01.9 Командная работа: pull request, код-ревью и соглашения](09-teamwork-pull-requests-and-conventions.md) · [📋 Этап 01](README.md) · [Этап 02 — Основы программирования на Rust →](../02-programming-basics/README.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/01-git-and-version-control/10-project-your-learning-repo-and-checkpoint.md) <!-- nav -->

# Урок 01.10 — Проект: ваш учебный репозиторий и контрольная точка

> **Этап 01 — Git и контроль версий** · Урок 10 из 10 · ⏱ около 3 часов (2 занятия)

## Что вы узнаете

- Как поместить всё, что вы делаете в этом курсе, в один аккуратный репозиторий Git с хорошей структурой и хорошими привычками.
- Как отработать полный профессиональный цикл на собственном проекте: ветки, конфликт, удалённый репозиторий, pull request с автоматическими проверками и релиз с тегом.
- Как избежать одной ловушки Cargo и Git, которая создаёт скрытый репозиторий внутри вашего репозитория.
- Как проверить себя по всему Этапу 01 и где повторить, если что-то непонятно.

## Перед началом

- Вы прошли Уроки [01.1](01-what-is-version-control.md) — [01.9](09-teamwork-pull-requests-and-conventions.md). Git у вас настроен ([Урок 01.2](02-install-and-configure-git.md)).
- Для вехи M5 нужен аккаунт GitHub или GitLab с SSH-ключом (SSH — Secure Shell, защищённая оболочка) ([Урок 01.8](08-remotes-github-and-gitlab.md)). Если сейчас создать его нельзя, у M5 и M6 есть вариант без сети.
- Ваша папка курса — `~/projects/rust`. Отныне **каждый проект этого курса лежит в `~/projects/rust/code/pNN-<name>`** (NN — номер этапа) и коммитится в этот репозиторий.

## Новые слова

| Слово | Значение |
|---|---|
| **учебный репозиторий** (learning repository) | Единый репозиторий Git для всего вашего кода и заметок в этом курсе. |
| **веха** (milestone, M1, M2, …) | Шаг проекта с ясной целью и способом проверки. |
| **приёмочный тест** (acceptance test) | Команда (или несколько), ожидаемый вывод которой доказывает, что веха выполнена. Строки «Готово, когда» — это приёмочные тесты. |
| **вложенный репозиторий** (nested repository) | Репозиторий внутри папки другого репозитория. Почти всегда это ошибка. |
| **журнал изменений** (changelog) | Файл (`CHANGELOG.md`), в котором для людей перечислено, что изменилось в каждой версии. |
| **релиз** (release) | Именованная версия проекта с тегом, например `v0.1.0`. |
| **`core.hooksPath`** | Настройка Git, которая говорит искать хуки в другой папке, которую можно закоммитить в репозиторий. |
| **контрольная точка** (checkpoint) | Урок-повторение в конце этапа: вопросы и практические задания. |

## 1. Что это за проект

В этом этапе вы учили Git на небольших папках для практики. Теперь вы строите настоящее: **один репозиторий на весь курс**. Начиная с Этапа 02 каждый проект попадает в него, а ваша история Git становится дневником вашего обучения. Тот же процесс под другими названиями вы будете использовать на работе.

Ещё вы напишете свой первый *библиотечный крейт* Rust (пакет Rust, который может вызывать другой код) как объект, который нужно поставить под контроль версий. Пока не нужно понимать код на Rust: вы его копируете, а на Этапе 02 его объяснят. Смысл этого проекта — **Git**, а не Rust.

## 2. Ловушка: `cargo new` может создать скрытый репозиторий

Прочитайте это перед началом. Когда вы запускаете `cargo new` **вне** репозитория Git, Cargo любезно сам выполняет `git init`. Внутри существующего репозитория он этого делать не должен. Но есть нюанс: Cargo решает, глядя на **родительскую папку** нового проекта, и если этой родительской папки ещё нет, Cargo не видит ваш репозиторий. Сравните:

```bash
# the folder code/ does NOT exist yet
cargo new code/p01-git-practice --lib
find . -maxdepth 4 -name .git
```

```text
./code/p01-git-practice/.git
./.git
```

Внутри проекта появился второй `.git`: **вложенный репозиторий**. Внешний репозиторий теперь видит `code/p01-git-practice` как один странный объект, а ваша история не содержит его файлов. Правильно — сначала создать родительскую папку:

```bash
mkdir -p code
cargo new code/p01-git-practice --lib
find . -maxdepth 4 -name .git
```

```text
./.git
```

(Только один `.git`.) В этом случае Cargo также не создаёт `.gitignore` внутри проекта, поэтому **корневой** `.gitignore` должен игнорировать `target/` (без косой черты в начале, чтобы шаблон совпадал в каждой папке). Сделайте `mkdir -p code` привычкой перед каждым `cargo new code/…` до конца курса.

## Частые ошибки

> ⚠️ **Частая ошибка:** вложенная папка `.git` (см. выше). Проверьте командой `find . -maxdepth 4 -name .git`. Должен появиться только `./.git`. Если у папки проекта есть собственный `.git` и в нём нет истории, которую вы хотите сохранить, удалите эту внутреннюю папку (`rm -rf` для **именно этого пути**, после проверки через `git -C <папка> log --oneline`).

> ⚠️ **Частая ошибка:** корневой `.gitignore` содержит `/target`, что совпадает только с `target` верхнего уровня. Используйте `target/`.

> ⚠️ **Частая ошибка:** создать удалённый репозиторий на GitHub с README и лицензией, а потом не суметь сделать push. Создавайте его **пустым**.

> ⚠️ **Частая ошибка:** коммитить `Cargo.lock` поздно или никогда. Для программ и для этого учебного репозитория коммитьте его.

> ⚠️ **Частая ошибка:** работать прямо в `main` «потому что это только я». Цель здесь — отработать командный процесс, поэтому используйте ветки для каждой вехи.

> 🔒 **Безопасность:** в репозитории никогда не должно быть пароля, токена, файла `.env` или закрытого ключа. Проверьте `git ls-files` и `.gitignore` перед первым push.

## Попробуйте сами

Проверка готовности. Перед началом проверьте инструменты. Все команды должны работать:

```bash
git --version
git config --global user.name
git config --global user.email
git config --global init.defaultBranch
cargo --version
ssh -T git@github.com
```

Первые пять печатают значения (версию, ваше имя, ваш email, `main`, версию Cargo). Последняя печатает сообщение «successfully authenticated» (GitHub) или `Permission denied (publickey)` (исправьте по [Уроку 01.8](08-remotes-github-and-gitlab.md) или используйте вариант без сети).

## Проект

### Что вы создадите

Один репозиторий Git, `~/projects/rust`, в котором есть:

- понятная структура и README;
- ваша собственная **шпаргалка по Git** в `notes/`;
- первый библиотечный крейт Rust `code/p01-git-practice` с тестами;
- история с фича-ветками, слияниями и одним решённым конфликтом;
- приватный удалённый репозиторий на GitHub или GitLab (или bare-репозиторий без сети);
- pull request с автоматическими проверками (CI, Continuous Integration), слитый через squash;
- релиз с тегом `v0.1.0` и журнал изменений.

### Контракт

**Структура** (итоговый результат):

```text
~/projects/rust/
├── .gitignore                  ← target/, .idea/, .DS_Store, *.swp, .env
├── README.md                   ← что это за репозиторий, как запускать тесты, ваш стиль коммитов
├── CHANGELOG.md                ← пишется в M7
├── notes/
│   └── git-cheatsheet.md       ← ваша шпаргалка, не меньше 15 команд
├── code/
│   └── p01-git-practice/       ← библиотечный крейт (cargo new --lib)
│       ├── Cargo.toml
│       ├── Cargo.lock
│       └── src/lib.rs
└── .github/workflows/ci.yml    ← или .gitlab-ci.yml (пишется в M6)
```

Если в вашей папке есть и другие материалы (например, текст курса, если он хранится там), решите для каждого элемента: **отслеживать**, **игнорировать** (добавить в `.gitignore`) или **убрать в другое место**. Запишите решение в README.

**Правила:**

1. Ветка по умолчанию — `main`. Работу над каждой вехой ведите в ветке с именем `type/short-name` (`feature/…`, `docs/…`, `ci/…`, `chore/…`).
2. Сообщения коммитов: короткая тема в повелительном наклонении (около 50 символов, без точки в конце). По желанию используйте префиксы Conventional Commits (`feat:`, `docs:`, `ci:`). Выберите один стиль, запишите его в README и используйте везде.
3. Одно логическое изменение на коммит. К концу — не менее **10 коммитов** всего.
4. Никогда не коммитьте `target/`, `.idea/`, `.DS_Store`, `.env` и любые секреты.
5. Во всём дереве только **одна** папка `.git`.
6. Перед каждым коммитом выполняйте `git status` и `git diff --staged`.

**Контракт библиотеки** для `code/p01-git-practice` (код дан; Этап 02 его объяснит):

| Функция | Поведение |
|---|---|
| `add(a: i32, b: i32) -> i32` | возвращает `a + b` |
| `subtract(a: i32, b: i32) -> i32` | возвращает `a - b` |
| `multiply(a: i32, b: i32) -> i32` | возвращает `a * b` |

У каждой функции один тест. Команды `cargo test`, `cargo fmt --check` и `cargo clippy --all-targets -- -D warnings` должны завершаться успешно.

**Итоговая приёмка** (весь проект):

```bash
cd ~/projects/rust
git status -sb                       # first line: ## main...origin/main
git log --oneline | wc -l            # 10 or more
git log --merges --oneline | wc -l   # 2 or more
git ls-files | grep -c 'target/'     # 0
find . -maxdepth 4 -name .git        # only ./.git
git tag -n1                          # v0.1.0 and your message
cargo test --manifest-path code/p01-git-practice/Cargo.toml   # 3 passed
```

### Пошаговый план

Полного решения здесь нет. У каждой вехи есть цель, указания, подсказка (скрытая) и команды **«Готово, когда»** с ожидаемым выводом. Коммитьте в конце каждой вехи, в её ветке.

#### M1 — Создайте репозиторий

**Цель:** `~/projects/rust` — репозиторий Git с хорошим `.gitignore`, README и папкой для заметок и одним первым коммитом в `main`.

**Указания:**

1. Откройте папку и посмотрите, что в ней уже есть: `cd ~/projects/rust && ls -a`. Убедитесь, что она ещё не репозиторий (`git status` должен сказать `fatal: not a git repository`) и что ни у одной подпапки нет собственного `.git` (`find . -maxdepth 3 -name .git`). Если папка уже репозиторий (например, потому что вы опубликовали курс вместе со своей работой), пропустите `git init` в шаге 2 и вместо создания `.gitignore` и `README.md` проверьте, что существующие файлы соответствуют контракту.
2. Выполните `git init`. (`init.defaultBranch` сделает ветку `main`.)
3. Создайте `.gitignore` с такими строками: `target/`, `.idea/`, `.DS_Store`, `*.swp`, `.env`. Добавьте другие папки из вашего решения (см. контракт).
4. Создайте `README.md` с заголовком, одним абзацем и строкой «Commit style».
5. Создайте `notes/git-cheatsheet.md` пока только с заголовком.
6. Прочитайте `git status`, подготовьте файлы **по именам**, проверьте `git diff --staged` и закоммитьте: `Start the learning repository`.

<details>
<summary>Подсказка</summary>

Не используйте `git add .`, пока не прочитали `git status`: так можно захватить текст курса или старый проект. Подготовьте явно `.gitignore README.md notes`. Если `find` показывает внутреннюю папку `.git`, сначала разберитесь с ней (см. «Частые ошибки»).

</details>

**Готово, когда:**

```bash
git status -sb
git log --oneline | wc -l
find . -maxdepth 4 -name .git
```

```text
## main
1
./.git
```

(`wc -l` может печатать пробелы перед числом. Если существуют другие неотслеживаемые папки, `git status -sb` перечисляет их под первой строкой; это нормально, если вы решили, что с ними делать.)

#### M2 — Первый крейт Rust без скрытого репозитория

**Цель:** крейт `code/p01-git-practice` создан и закоммичен как часть **вашего** репозитория.

**Указания:**

1. `git switch -c feature/crate`.
2. Сначала создайте родительскую папку: `mkdir -p code`. Затем `cargo new code/p01-git-practice --lib`.
3. Проверьте, что вложенный репозиторий не появился (команда `find` выше).
4. Замените `code/p01-git-practice/src/lib.rs` этим кодом (всё в нём вы изучите на Этапе 02):

```rust
pub fn add(a: i32, b: i32) -> i32 {
    a + b
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn adds_two_numbers() {
        assert_eq!(add(2, 3), 5);
    }
}
```

5. Выполните `cargo test` в папке крейта (это также создаст `Cargo.lock`). Подготовьте и закоммитьте: `Add p01-git-practice with an add function`.
6. Слейте ветку в `main` (подойдёт fast-forward) и удалите её.

<details>
<summary>Подсказка</summary>

Если `find` показывает `./code/p01-git-practice/.git`, вы забыли `mkdir -p code`. Удалите внутренний `.git` (`rm -rf code/p01-git-practice/.git`) и продолжайте. Убедитесь, что `Cargo.lock` подготовлен.

</details>

**Готово, когда:**

```bash
find . -maxdepth 4 -name .git
git ls-files code
cd code/p01-git-practice && cargo test && cd ../..
git ls-files | grep -c 'target/'
```

```text
./.git
code/p01-git-practice/Cargo.lock
code/p01-git-practice/Cargo.toml
code/p01-git-practice/src/lib.rs
```

а `cargo test` заканчивается строкой `test result: ok. 1 passed`, и последняя команда печатает `0`.

#### M3 — Две ветки, один конфликт

**Цель:** добавить `subtract` и `multiply` в двух **отдельных** ветках, которые обе начинаются с одного коммита, слить их в `main` и решить возникающий при этом конфликт.

**Указания:**

1. От `main` создайте `feature/subtract`. Добавьте функцию `subtract` (`a - b`) **над** строкой `#[cfg(test)]` и тест `subtracts_two_numbers` (`assert_eq!(subtract(5, 3), 2);`). Закоммитьте: `Add subtract with a test`.
2. Вернитесь в `main` и создайте `feature/multiply` **от `main`** (а не от `feature/subtract`). Добавьте `multiply` (`a * b`) в то же место и тест `multiplies_two_numbers` (`assert_eq!(multiply(4, 3), 12);`). Закоммитьте: `Add multiply with a test`.
3. Переключитесь на `main`. Слейте `feature/subtract` командой `git merge --no-ff feature/subtract` (дайте хорошее сообщение). Удалите ветку.
4. Слейте `feature/multiply`. Git сообщит о конфликте в `src/lib.rs`. Прочитайте маркеры (секция base от `zdiff3` поможет). Решите итоговый текст: **обе функции и оба теста должны остаться**, а файл — сохранить свою структуру.
5. `cargo fmt`, `cargo test`, `git diff --check`. Затем `git add` и `git commit --no-edit`.
6. Удалите `feature/multiply`.

<details>
<summary>Подсказка</summary>

Конфликт выглядит странно, потому что обе ветки добавили код в одном месте, а закрывающие строки `}` общие. Не пытайтесь понять его построчно: напишите итоговый файл вручную, с `add`, `subtract`, `multiply` и тремя тестами, и сравните с тем, что вам нужно. `cargo test` должен показать `3 passed`. Если запутались: `git merge --abort` и попробуйте снова.

</details>

**Готово, когда:**

```bash
git grep -n '<<<<<<<' ; echo "exit=$?"
git log --merges --oneline
cd code/p01-git-practice && cargo test 2>&1 | grep 'test result' | head -1 ; cd ../..
```

```text
exit=1
… Merge branch 'feature/multiply'
… Merge branch 'feature/subtract'
test result: ok. 3 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

(`exit=1` значит, что `git grep` не нашёл маркеров конфликта. Хеши перед двумя строками слияний — ваши.)

#### M4 — Ваша шпаргалка по Git и README

**Цель:** заметки и README описывают репозиторий и то, чему вы научились, вашими словами.

**Указания:**

1. `git switch -c docs/cheatsheet`.
2. В `notes/git-cheatsheet.md` запишите **не менее 15 команд**, которые вы изучили в этом этапе, каждую в строке, начинающейся с ``- `git ``, с короткими пояснениями, сгруппировав по темам (настройка, снимки, история, отмена, ветки, удалённые репозитории, командная работа). Используйте только то, что понимаете.
3. В `README.md` опишите: что это за репозиторий, структуру, как запускать тесты (`cargo test` в папке крейта) и ваш стиль коммитов.
4. Закоммитьте двумя коммитами, по одному на файл (`docs: add Git cheat sheet`, `docs: describe the repository`), слейте через **fast-forward** или коммит слияния по своему желанию и удалите ветку.

<details>
<summary>Подсказка</summary>

Пишите шпаргалку, глядя на историю терминала (`history | grep git`), а не копируя. Ваши собственные слова — вот что делает её полезной потом.

</details>

**Готово, когда:**

```bash
grep -c '^- `git' notes/git-cheatsheet.md
git log --oneline | wc -l
```

Первая команда печатает `15` или больше; вторая сейчас печатает `8` или больше (точное число зависит от ваших решений).

#### M5 — Удалённый репозиторий

**Цель:** ваш репозиторий существует в удалённом хранилище, а `main` отправлена.

**Указания:**

1. На GitHub (или GitLab) создайте **пустой приватный** репозиторий, например `learning-rust`. Не добавляйте README, `.gitignore` или лицензию.
2. Скопируйте SSH-адрес. В своём репозитории: `git remote add origin git@github.com:YOUR-NAME/learning-rust.git`.
3. `git push -u origin main`. Если вы пользуетесь GitHub CLI (Command-Line Interface, интерфейс командной строки: собственная терминальная программа GitHub, команда `gh`): `gh repo create learning-rust --private --source=. --remote=origin --push` делает шаги 1–3.
4. Теги вы отправите позже (M7). Посмотрите на репозиторий на сайте.

*Вариант без сети:* создайте bare-репозиторий **вне** учебного репозитория, например `git init --bare ~/projects/rust-origin.git`, и используйте `git remote add origin ~/projects/rust-origin.git`. Всё остальное в M5–M7 одинаково.

<details>
<summary>Подсказка</summary>

Если `git push` говорит `Permission denied (publickey)`, выполните `ssh -T git@github.com`, чтобы проверить ключ ([Урок 01.8](08-remotes-github-and-gitlab.md)). Если он говорит, что в удалённом репозитории есть работа, которой у вас нет, вы создали удалённый репозиторий с README: удалите его и создайте пустой.

</details>

**Готово, когда:**

```bash
git remote -v
git status -sb
git ls-remote origin
```

```text
origin	git@github.com:YOUR-NAME/learning-rust.git (fetch)
origin	git@github.com:YOUR-NAME/learning-rust.git (push)
## main...origin/main
<hash>	HEAD
<hash>	refs/heads/main
```

(Два хеша в последней команде одинаковы и равны вашему последнему коммиту в `main`.)

#### M6 — Pull request с CI

**Цель:** вы добавляете рабочий процесс CI через **pull request**, сами его ревьюите и сливаете со squash.

**Указания:**

1. `git switch -c ci/add-checks`.
2. Создайте `.github/workflows/ci.yml` (GitHub). Поскольку крейт лежит внутри `code/`, скажите задаче, где запускаться:

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:

jobs:
  check:
    runs-on: ubuntu-latest
    defaults:
      run:
        working-directory: code/p01-git-practice
    steps:
      - uses: actions/checkout@v7
      - run: rustup toolchain install stable --profile minimal --component rustfmt,clippy
      - run: cargo fmt --check
      - run: cargo clippy --all-targets -- -D warnings
      - run: cargo test
```

   На GitLab используйте вместо этого такой `.gitlab-ci.yml`:

```yaml
stages: [check]

rust-checks:
  stage: check
  image: rust:1
  before_script:
    - cd code/p01-git-practice
  script:
    - rustup component add rustfmt clippy
    - cargo fmt --check
    - cargo clippy --all-targets -- -D warnings
    - cargo test
```

3. Сначала запустите три команды локально в папке крейта. Закоммитьте: `ci: check format, lints and tests`. Отправьте ветку.
4. Откройте pull request (`gh pr create --fill` или через сайт). Напишите описание с What / Why / Testing. Откройте вкладку «Files changed» и отревьюьте свои изменения. Дождитесь зелёной проверки.
5. Слейте через **Squash and merge**. Удалите удалённую ветку (кнопкой или `git push origin --delete ci/add-checks`).
6. Обновите локальную `main` (`git switch main`, `git pull`) и удалите локальную ветку (`git branch -d ci/add-checks` или `-D`, если после squash Git скажет «not fully merged»).

*Вариант без сети:* слейте локально через `git merge --squash ci/add-checks` и `git commit -m "ci: check format, lints and tests (#1)"` и отправьте `main`. Запишите в журнал, что добавил бы настоящий PR (ревью, проверки).

<details>
<summary>Подсказка</summary>

Если рабочий процесс не появляется во вкладке *Actions*, проверьте имя папки (`.github/workflows`, с точкой) и отступы (пробелы, никогда не табуляция). Если CI красный, откройте лог, найдите упавшую команду и воспроизведите её локально.

</details>

**Готово, когда:**

```bash
git log --oneline -1
git branch -a
ls .github/workflows
```

```text
<hash> ci: check format, lints and tests (#1)
* main
  remotes/origin/main
ci.yml
```

(Номер в заголовке PR берётся из вашего сервиса, например `(#1)`.) На сайте во вкладке *Actions* виден зелёный значок последнего запуска на `main`.

#### M7 — Журнал изменений и релиз

**Цель:** первый релиз `v0.1.0` описан, помечен тегом и отправлен.

**Указания:**

1. `git switch -c docs/changelog`. Создайте `CHANGELOG.md` в стиле «Keep a Changelog»: заголовок `## [0.1.0] - 2026-11-17` (используйте сегодняшнюю дату) и пункты в `### Added` (крейт, три функции, CI, заметки).
2. Закоммитьте (`docs: add changelog for 0.1.0`), откройте PR или слейте локально, как в M6, и обновите `main`.
3. Создайте **аннотированный тег** на последнем коммите `main`: `git tag -a v0.1.0 -m "First release of the learning repository"`.
4. Отправьте тег: `git push origin v0.1.0`.
5. (GitHub) Откройте *Releases → Draft a new release*, выберите тег `v0.1.0` и вставьте раздел из журнала изменений. Необязательно.

<details>
<summary>Подсказка</summary>

`git tag` без `-a` создаёт облегчённый тег; используйте `-a` и `-m`. Теги не отправляются просто командой `git push`. Если вы поставили тег не на тот коммит, удалите его (`git tag -d v0.1.0`; а в удалённом репозитории `git push origin --delete v0.1.0`) и создайте заново.

</details>

**Готово, когда:**

```bash
git tag -n1
git ls-remote --tags origin
```

```text
v0.1.0          First release of the learning repository
<hash>	refs/tags/v0.1.0
<hash>	refs/tags/v0.1.0^{}
```

(Для аннотированного тега появляются две строки: объект тега и коммит, на который он указывает, помеченный `^{}`.)

Теперь выполните команды **итоговой приёмки** из контракта. Все они должны совпасть.

### Дополнительные цели

1. **Хук в репозитории.** Создайте `scripts/hooks/pre-commit`, который выполняет `cargo fmt --check` в крейте, сделайте его исполняемым и закоммитьте. Скажите Git использовать эту папку: `git config core.hooksPath scripts/hooks`. Теперь хук — часть репозитория (коллеги один раз выполняют `git config core.hooksPath scripts/hooks`).
2. **Защитите `main`.** На сайте добавьте правило защиты ветки: требовать pull request и успешную проверку CI. Попробуйте отправить прямо в `main` и прочитайте ошибку.
3. **Шаблон PR.** Добавьте `.github/pull_request_template.md` с заголовками What / Why / How / Testing.
4. **Детектив.** Добавьте намеренно сломанный коммит в середину новой ветки, затем найдите его с помощью `git bisect run cargo test`.
5. **Подписанные коммиты.** Следуйте руководству GitHub, чтобы подписывать коммиты SSH-ключом и увидеть значок «Verified».
6. **Второй удалённый репозиторий.** Добавьте удалённый репозиторий GitLab с именем `gitlab` и отправьте `main` в оба (`git push gitlab main`).

## Вопросы для самопроверки

Ответьте, не заглядывая в уроки, затем откройте ответы.

1. Какую проблему решает контроль версий и что значит слово «распределённая» для Git?

<details>
<summary>Ответ</summary>

Он записывает каждую версию ваших файлов с сообщением и автором, чтобы можно было сравнивать, находить, кто что и зачем изменил, возвращаться назад и работать параллельно. «Распределённая» значит, что у каждого разработчика есть полная копия всей истории, поэтому большинство команд работает без сети, а любая копия может восстановить проект.

</details>

2. Чем отличаются Git и GitHub?

<details>
<summary>Ответ</summary>

Git — программа, которая записывает историю на вашем компьютере. GitHub — сайт и сервис, который хостит репозитории Git и добавляет pull request, задачи и CI. GitLab — похожий сервис.

</details>

3. Назовите по порядку три места, где может жить изменение до отправки.

<details>
<summary>Ответ</summary>

Рабочее дерево (ваши отредактированные файлы), область подготовки (то, что `git add` собрал для следующего коммита) и репозиторий (коммиты, в `.git`).

</details>

4. Вы изменили `a.rs` и `b.rs`, но хотите закоммитить только `a.rs`. Какие команды?

<details>
<summary>Ответ</summary>

`git add a.rs` и `git commit -m "…"`. В коммит попадает только подготовленный файл; `b.rs` остаётся изменённым.

</details>

5. Что показывает `git status -s` для нового неотслеживаемого файла и для изменённого неподготовленного файла?

<details>
<summary>Ответ</summary>

`?? file` для неотслеживаемого файла и ` M file` (пробел, затем `M`) для изменённого неподготовленного файла.

</details>

6. Зачем в проекте на Rust добавляют `target/` в `.gitignore`? Помогает ли `.gitignore` для файла, который уже закоммичен?

<details>
<summary>Ответ</summary>

В `target/` лежит результат сборки, который можно воссоздать и который может быть огромным; в истории его быть не должно. `.gitignore` не влияет на уже отслеживаемые файлы: используйте `git rm --cached <file>` и коммит.

</details>

7. Что означают `HEAD`, `HEAD~1` и `HEAD~3`?

<details>
<summary>Ответ</summary>

`HEAD` — коммит, на котором вы сейчас; `HEAD~1` — его родитель; `HEAD~3` — три шага назад.

</details>

8. Как увидеть, что именно будет в следующем коммите?

<details>
<summary>Ответ</summary>

`git diff --staged` (и `git status`).

</details>

9. Что из перечисленного может уничтожить незакоммиченную работу: `git restore <file>`, `git restore --staged <file>`, `git reset --hard`, `git stash`, `git clean -f`?

<details>
<summary>Ответ</summary>

`git restore <file>`, `git reset --hard` и `git clean -f`. `git restore --staged` и `git stash` безопасны.

</details>

10. Когда использовать `git revert` вместо `git reset`?

<details>
<summary>Ответ</summary>

Когда коммит уже общий (отправлен). `revert` добавляет новый коммит, отменяющий старый, и не переписывает историю.

</details>

11. Вы по ошибке выполнили `git reset --hard HEAD~3`. Как вернуть три коммита?

<details>
<summary>Ответ</summary>

Выполните `git reflog`, найдите запись до сброса (например, `HEAD@{1}`) и выполните `git reset --hard 'HEAD@{1}'` (или используйте хеш).

</details>

12. Что такое ветка и что делает `git switch -c fix/typo`?

<details>
<summary>Ответ</summary>

Ветка — небольшой перемещаемый указатель на коммит. `git switch -c fix/typo` создаёт новую ветку на текущем коммите и переключается на неё.

</details>

13. Когда слияние является fast-forward, а когда Git создаёт коммит слияния?

<details>
<summary>Ответ</summary>

Fast-forward: в текущей ветке нет коммитов, которых не хватает в другой, поэтому Git лишь двигает указатель. Коммит слияния: в обеих ветках есть новые коммиты, поэтому Git соединяет их коммитом с двумя родителями (или вы принуждаете к этому через `--no-ff`).

</details>

14. Что отмечают в блоке конфликта `<<<<<<< HEAD`, `|||||||`, `=======` и `>>>>>>> other`? Каковы шаги завершения?

<details>
<summary>Ответ</summary>

Вашу версию (ours), версию общего предка (base, с `zdiff3`), разделитель и конец с версией другой ветки (theirs). Отредактируйте файл до итогового текста без маркеров, выполните `git add` и `git commit` (или `git merge --continue`). `git merge --abort` отменяет.

</details>

15. Чем отличаются `git fetch` и `git pull`? Почему push был отклонён с «fetch first»?

<details>
<summary>Ответ</summary>

`fetch` скачивает и обновляет `origin/*`, не трогая вашу работу; `pull` — это `fetch` плюс слияние. Push был отклонён, потому что в удалённом репозитории были коммиты, которых у вас нет; сначала их нужно интегрировать.

</details>

16. Какой SSH-ключ отдают GitHub и что будет, если утечёт другой?

<details>
<summary>Ответ</summary>

Открытый (`.pub`). Если утечёт закрытый ключ, любой может действовать от вашего имени: удалите открытый ключ из сервисов и создайте новую пару ключей.

</details>

17. Почему `git push --force` опасен и что безопаснее для своей ветки?

<details>
<summary>Ответ</summary>

Он перезаписывает историю удалённого репозитория и может уничтожить чужие коммиты. `--force-with-lease` откажет, если удалённая ветка изменилась с тех пор, как вы видели её в последний раз. Никогда не делайте force-push в `main`.

</details>

18. В чём золотое правило `rebase`?

<details>
<summary>Ответ</summary>

Не делайте rebase коммитов, которые уже есть у других людей; делайте rebase только своей неопубликованной работы.

</details>

19. Версия меняется с `0.4.2` на `0.5.0`. Что SemVer говорит об изменении? Какое хорошее сообщение коммита для исправления ошибки в стиле Conventional Commits?

<details>
<summary>Ответ</summary>

Изменение MINOR (новая возможность с обратной совместимостью; до `1.0.0` MINOR может и ломать). Сообщение для исправления ошибки: `fix: avoid panic on empty input`.

</details>

20. Какие три команды обычно запускает CI для проекта на Rust?

<details>
<summary>Ответ</summary>

`cargo fmt --check`, `cargo clippy --all-targets -- -D warnings` и `cargo test`.

</details>

## Практические задания

Выполняйте их в новой папке `~/projects/git-playground/checkpoint`. Каждое задание начинается с блока подготовки.

### Задание 1 — Спасение

Подготовка:

```bash
mkdir -p ~/projects/git-playground/checkpoint
cd ~/projects/git-playground/checkpoint
git init -b main rescue-task
cd rescue-task
for n in 1 2 3 4; do echo "line $n" >> story.txt; git add story.txt; git commit -m "Add line $n"; done
git switch -c idea
echo "special idea" >> story.txt
git commit -am "Add the special idea"
git switch main
git branch -D idea
git reset --hard HEAD~2
```

Катастрофа: ветки `idea` больше нет, а `main` потеряла два коммита. Верните **оба**: `main` должна снова показывать четыре строки, а в ветке `idea` должна быть особая идея.

<details>
<summary>Подсказка</summary>

`git reflog` помнит каждую позицию `HEAD`. Сообщение `git branch -D` тоже напечатало хеш.

</details>

<details>
<summary>Ответ</summary>

```bash
git reflog
git reset --hard 'HEAD@{1}'
git branch idea 28622a0
git log --oneline --all --graph --decorate
```

Используйте хеш, который **вы** видите в строке reflog `commit: Add the special idea` (здесь `28622a0`). Результат:

```text
* 28622a0 (idea) Add the special idea
* c3802a1 (HEAD -> main) Add line 4
* a17aec2 Add line 3
* c5b0e53 Add line 2
* 99a925a Add line 1
```

Почему это работает: `HEAD@{1}` — то место, где был `HEAD` до сброса (вершина `main`), а ветку можно создать на любом коммите, в том числе на таком, на который не указывает ни одна ветка.

</details>

### Задание 2 — Решите конфликт

Подготовка:

```bash
cd ~/projects/git-playground/checkpoint
git init -b main conflict-task
cd conflict-task
printf 'name = "demo"\nversion = "0.1.0"\nauthor = "ada"\n' > app.conf
git add app.conf
git commit -m "Add app.conf"
git switch -c release
printf 'name = "demo"\nversion = "0.2.0"\nauthor = "ada"\n' > app.conf
git commit -am "Bump version to 0.2.0"
git switch main
printf 'name = "demo-tool"\nversion = "0.1.5"\nauthor = "ada"\n' > app.conf
git commit -am "Rename the tool and bump to 0.1.5"
```

Слейте `release` в `main`. В итоговом файле должно быть новое имя **и** версия `0.2.0`.

<details>
<summary>Подсказка</summary>

Слейте, прочитайте блок, напишите итоговые три строки, `git add`, коммит.

</details>

<details>
<summary>Ответ</summary>

```bash
git merge release
printf 'name = "demo-tool"\nversion = "0.2.0"\nauthor = "ada"\n' > app.conf
git diff --check
git add app.conf
git commit --no-edit
git log --oneline --graph
cat app.conf
```

```text
*   1635a42 Merge branch 'release'
|\
| * 85c32fa Bump version to 0.2.0
* | e5e6963 Rename the tool and bump to 0.1.5
|/
* 9067988 Add app.conf
name = "demo-tool"
version = "0.2.0"
author = "ada"
```

Почему это работает: две ветки изменили соседние строки, поэтому Git показал один блок конфликта; вы вручную записали комбинацию и пометили её решённой.

</details>

### Задание 3 — Отклонённый push

Подготовка:

```bash
cd ~/projects/git-playground/checkpoint
git init --bare hub.git
git clone hub.git alice
cd alice
git config user.name Alice
git config user.email alice@example.com
echo base > shared.txt
git add shared.txt
git commit -m "Base"
git push -u origin main
cd ..
git clone hub.git bob
cd bob
git config user.name Bob
git config user.email bob@example.com
cd ../alice
echo "from alice" > alice.txt
git add alice.txt
git commit -m "Alice work"
git push
cd ../bob
echo "from bob" > bob.txt
git add bob.txt
git commit -m "Bob work"
git push
```

Последний `git push` Боба отклонён. Исправьте это и покажите общую историю.

<details>
<summary>Подсказка</summary>

Fetch, слияние `origin/main` (ваша настройка `pull.ff only` останавливает обычный `git pull`), push.

</details>

<details>
<summary>Ответ</summary>

```bash
git fetch
git merge origin/main --no-edit
git push
git log --oneline --graph
```

```text
*   a9575b9 Merge remote-tracking branch 'origin/main'
|\
| * eccd231 Alice work
* | 3d713ec Bob work
|/
* ca9e9ff Base
```

Почему это работает: `main` Боба и `main` в удалённом репозитории разошлись. Слияние вносит коммит Алисы в историю Боба, поэтому push только добавляет коммиты в удалённый репозиторий.

</details>

### Задание 4 — Найдите плохой коммит

Подготовка:

```bash
cd ~/projects/git-playground/checkpoint
git init -b main detective
cd detective
for i in 1 2 3 4 5 6 7 8 9 10; do
  echo "ok $i" >> status.txt
  if [ $i = 7 ]; then echo "FAIL" >> status.txt; fi
  git add status.txt
  git commit -m "Step $i"
done
printf '#!/bin/sh\n! grep -q FAIL status.txt\n' > check.sh
chmod +x check.sh
```

Найдите автоматически первый коммит, содержащий `FAIL`, а затем вернитесь в обычное состояние.

<details>
<summary>Подсказка</summary>

`git bisect start <плохой> <хороший>` и `git bisect run ./check.sh`. Первый коммит хороший.

</details>

<details>
<summary>Ответ</summary>

```bash
git bisect start HEAD HEAD~9
git bisect run ./check.sh
git bisect reset
```

Вывод второй команды заканчивается строкой `… is the first bad commit` для `Step 7` примерно после трёх запусков скрипта.

Почему это работает: скрипт успешен, когда `FAIL` отсутствует. Bisect каждый раз делит диапазон пополам (10 коммитов требуют около 3 проверок) и печатает первый коммит, на котором скрипт падает.

</details>

## Если что-то не получилось

| Тема | Повторите |
|---|---|
| Что такое контроль версий, хеши, Git и GitHub | [Урок 01.1](01-what-is-version-control.md) |
| Установка Git, уровни конфигурации, личность, глобальное игнорирование | [Урок 01.2](02-install-and-configure-git.md) |
| `init`, `add`, `commit`, `status`, `.gitignore`, Cargo и Git | [Урок 01.3](03-your-first-repository-and-commits.md) |
| `log`, `show`, `diff`, `blame`, `bisect`, названия коммитов | [Урок 01.4](04-viewing-history-and-changes.md) |
| `restore`, `amend`, `reset`, `revert`, `reflog`, `stash`, `clean` | [Урок 01.5](05-undoing-changes-safely.md) |
| Ветки, `switch`, fast-forward и коммиты слияния | [Урок 01.6](06-branches-and-merging.md) |
| Маркеры конфликта, решение, отмена, `Cargo.lock` | [Урок 01.7](07-merge-conflicts.md) |
| Удалённые репозитории, `clone`, `fetch`, `pull`, `push`, SSH-ключи, секреты | [Урок 01.8](08-remotes-github-and-gitlab.md) |
| Pull request, ревью, `rebase`, теги, SemVer, CI, хуки | [Урок 01.9](09-teamwork-pull-requests-and-conventions.md) |
| Команды терминала, которыми пользуются везде | [Урок 00.4](../00-getting-started/04-terminal-basics.md) |

## Что дальше

Теперь у вас есть профессиональная страховочная сетка. С этого момента:

- каждый проект попадает в `~/projects/rust/code/pNN-<name>` (выполните `mkdir -p code` один раз и всегда создавайте проекты через `cargo new code/pNN-<name>`);
- работайте в ветке, коммитьте мелко и часто и сливайте в `main` в конце каждой вехи;
- поддерживайте `notes/` в актуальном состоянии.

На Этапе 02 вы начнёте программировать на Rust. Первый урок там — [Урок 02.1 — Переменные и изменяемость](../02-programming-basics/01-variables-and-mutability.md). Слова из этого этапа объяснены в [глоссарии](../GLOSSARY.md), а [стартовая страница](../START-HERE.md) показывает весь путь.

## Итоги

- Один учебный репозиторий с понятной структурой, хорошим `.gitignore` и README — основа всей вашей дальнейшей работы.
- Создавайте родительскую папку (`mkdir -p code`) **до** `cargo new code/…`, иначе Cargo создаст вложенный репозиторий.
- Проект готов, когда приёмочные команды печатают то, что сказано в плане; делайте один коммит (или несколько) на веху.
- Профессиональный цикл: ветка → коммит → push → pull request с CI → squash-слияние → обновление `main` → тег релиза.
- Конфликты решаются вручную, тестируются и затем коммитятся; удалённые репозитории отклоняют push, когда вы отстаёте.
- Не пускайте секреты в историю; защищайте `main`; подписывайте и ставьте теги аккуратно.

## Источники и что почитать

- [Pro Git](https://git-scm.com/book/en/v2) — полная бесплатная книга; перечитайте главы 2, 3 и 7, теперь, когда вы знаете основы.
- [The Cargo Book — Creating a New Package](https://doc.rust-lang.org/cargo/guide/creating-a-new-project.html) — опция VCS у `cargo new`.
- [GitHub Docs — Quickstart for repositories](https://docs.github.com/en/repositories/creating-and-managing-repositories/quickstart-for-repositories) — создание репозитория и управление им.
- [GitHub Docs — About protected branches](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-protected-branches/about-protected-branches) — для дополнительной цели.
- [Keep a Changelog](https://keepachangelog.com/en/1.1.0/) — формат журнала изменений для M7.
- [git-hooks manual](https://git-scm.com/docs/githooks) — все хуки и `core.hooksPath`.
- [Oh Shit, Git!?!](https://ohshitgit.com/) — короткий дружелюбный список типичных катастроф Git и способов их исправить.

---

[🏠 Начало](../START-HERE.md) · [← 01.9 Командная работа: pull request, код-ревью и соглашения](09-teamwork-pull-requests-and-conventions.md) · [📋 Этап 01](README.md) · [Этап 02 — Основы программирования на Rust →](../02-programming-basics/README.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
