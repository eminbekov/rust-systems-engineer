<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 24.3 Как читать большие кодовые базы](03-reading-large-codebases.md) · [📋 Этап 24](README.md) · [24.5 Работа с мейнтейнерами и сообществом →](05-working-with-maintainers-and-communities.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/24-career-and-open-source/04-your-first-open-source-contribution.md) <!-- nav -->

# Урок 24.4 — Ваш первый вклад в open source

> **Этап 24 — Карьера и open source** · Урок 4 из 9 · ⏱ около 2,5 часа (2 занятия)

## Что вы узнаете

- Что считается вкладом в open source и как выбрать проект и первую задачу (issue).
- Какие файлы проекта читать первыми: CONTRIBUTING, кодекс поведения, лицензию и правила вроде DCO,
  CLA или политики использования ИИ.
- Полный процесс работы через форк — форк, клон, `upstream`, ветка, сначала тест, коммит с sign-off, push, rebase —
  отработанный в безопасной локальной лаборатории.
- Как написать описание pull request, которое мейнтейнер может быстро проверить.

## Перед началом

- Вы прошли [Урок 24.3 — Как читать большие кодовые базы](03-reading-large-codebases.md).
- Вы знаете удалённые репозитории, pull request, rebase и форки из
  [Урока 01.8](../01-git-and-version-control/08-remotes-github-and-gitlab.md) и
  [Урока 01.9 — Командная работа: pull request, код-ревью и соглашения](../01-git-and-version-control/09-teamwork-pull-requests-and-conventions.md).
  Раздел 14 Урока 01.9 обещал этот урок.
- Ваша идентичность в Git настроена ([Урок 01.2](../01-git-and-version-control/02-install-and-configure-git.md)).

## Новые слова

| Слово | Значение |
|---|---|
| **open source** (открытое ПО) | Программы, исходный код которых опубликован под лицензией, позволяющей любому использовать, изучать, изменять и распространять его. |
| **вклад** (contribution) | Любая полезная работа для проекта: код, тесты, документация, понятный отчёт об ошибке, ответы другим пользователям. |
| **мейнтейнер** (maintainer) | Человек, который ведёт проект: проверяет и принимает изменения, выпускает версии, определяет направление. |
| **issue** (задача) | Пронумерованное обсуждение в трекере проекта об одной ошибке, идее или вопросе, например `#12`. |
| **форк** (fork) | Ваша собственная копия чужого репозитория на сайте хостинга. Вы отправляете изменения в свой форк, никогда — в оригинал. |
| **upstream** | Исходный репозиторий, от которого вы сделали форк. По соглашению ваш клон называет его `upstream`, а ваш форк — `origin`. |
| **задачи для новичков** (good first issue) | Метка, которую мейнтейнеры ставят на задачи, подходящие новичкам. В проектах встречаются и другие названия, например `E-easy` или `help wanted`. |
| **файл CONTRIBUTING** | `CONTRIBUTING.md`: правила проекта для участников: как собирать, тестировать, форматировать, писать коммиты и присылать изменения. |
| **кодекс поведения** (code of conduct) | Правила того, как люди обращаются друг с другом в пространствах проекта, и как сообщать о проблемах. |
| **DCO** (Developer Certificate of Origin, сертификат происхождения от разработчика) | Короткое заявление, что вы написали изменение (или имеете право его прислать) под лицензией проекта. Вы соглашаетесь с ним, добавляя к каждому коммиту строку `Signed-off-by:` через `git commit -s`. |
| **CLA** (Contributor License Agreement, лицензионное соглашение участника) | Юридическое соглашение, подписываемое один раз (часто онлайн, через бота), которое даёт проекту права, нужные для использования вашего вклада. |
| **sign-off** (подпись-заявление) | Строка `Signed-off-by: Имя <email>` в конце сообщения коммита. Это заявление, а не криптографическая подпись. |
| **force-with-lease** | `git push --force-with-lease`: заменить удалённую ветку вашей переписанной историей, но только если с вашего последнего fetch её никто не менял. |
| **CI** (Continuous Integration, непрерывная интеграция) | Автоматическая сборка и запуск тестов, которые проект запускает для каждого pull request (Урок 09.12). |
| **ID** (identifier, идентификатор) | Число или имя, которое обозначает что-то одно, например ID потока в сообщении паники. |
| **ИИ** (AI, Artificial Intelligence, искусственный интеллект) | Инструменты, которые генерируют текст или код. Некоторые проекты ограничивают их использование во вкладе (Урок 24.3). |

## 1. Что такое вклад

Код — лишь один вид вклада, и часто не лучший для начала. Мейнтейнеры так же рады:

- **отчёту об ошибке** с точными шагами, версиями и выводом (Урок 24.5 показывает, как его писать);
- **исправлению документации**: неверный пример, недостающее предложение, устаревшая команда;
- **тесту** для поведения, у которого ещё нет теста;
- **ответам** на вопросы других пользователей в трекере задач или чате;
- **проверке исправления**, которое кто-то предложил, с сообщением, помогло ли оно вам.

Ваша первая цель — не большая новая возможность. Это **одно небольшое принятое изменение**, чтобы вы изучили процесс
проекта: как его собирать, как он тестируется, как устроено ревью. После этого каждое следующее изменение даётся легче.

## 2. Выберите проект и задачу

**Начните с того, чем пользуетесь.** ripgrep вы знаете по Уроку 24.3, а за курс вы использовали много крейтов:
`serde`, `clap`, `tokio`, `axum`, `sqlx`, `tracing` и другие. Вносить вклад гораздо проще, когда вы уже понимаете,
для чего нужен проект.

Прежде чем тратить время, проверьте, что проект **жив**:

| Признак | Где смотреть | Хороший знак |
|---|---|---|
| Недавняя активность | история коммитов | коммиты за последние несколько недель |
| Отзывчивые мейнтейнеры | недавние issue и pull request | ответы в течение дней или нескольких недель |
| Понятные правила | `CONTRIBUTING.md`, `CODE_OF_CONDUCT.md`, лицензия | они есть и актуальны |
| Рабочие тесты | статус CI основной ветки | зелёный |

**Найдите задачу.** Отфильтруйте список issue по меткам вроде `good first issue`, `help wanted`, `E-easy` или
`E-mentor` (метки `E-` используются в репозиториях rust-lang). В рассылке *This Week in Rust* также есть еженедельный
раздел «Call for Participation» с задачами, которые мейнтейнеры отобрали для новичков.

Прежде чем начать работу над задачей:

1. **Прочитайте всё обсуждение.** Решение может быть уже обсуждено или отвергнуто.
2. **Проверьте, нет ли открытого pull request**, который уже это исправляет, и не написал ли кто-то «я над этим работаю».
3. **Коротко спросите.** Например: «I would like to work on this. My plan is to add a check in `parse_pair` and a
   test for an empty key. Does that sound right?» В репозиториях rust-lang можно также написать в комментарии
   `@rustbot claim`, чтобы назначить задачу себе.
4. **Соберите проект и запустите его тесты до любых изменений.** Если они падают на неизменённом коде, спросите об этом;
   причина не в вашем изменении.

> ⚠️ **Частая ошибка:** открыть большой pull request, не спросив заранее. Если мейнтейнерам не нужно это изменение
> или они хотят сделать его иначе, работа пропадёт. Вопрос в две строки в issue экономит дни.

## 3. Сначала правила, потом код

У каждого проекта свои правила. Потратьте пятнадцать минут на эти файлы до первого коммита:

- **`CONTRIBUTING.md`** — как собирать и тестировать, правила форматирования и линтеров, стиль сообщений коммитов,
  нужно ли сначала открывать issue, нужна ли запись в CHANGELOG.
- **`CODE_OF_CONDUCT.md`** — как ожидается вести себя и к кому обращаться, если что-то пошло не так. Многие проекты
  используют *Contributor Covenant*; проекты на Rust часто следуют Rust Code of Conduct.
- **Лицензия** — ваш вклад публикуется под ней.
- **DCO или CLA.** Проект с **DCO** просит строку `Signed-off-by:` в каждом коммите; её добавляет `git commit -s`.
  Проект с **CLA** просит один раз подписать соглашение, обычно по ссылке, которую бот публикует в вашем первом
  pull request. Читайте то, что подписываете. Если вы работаете по найму, проверьте, не говорит ли что-нибудь ваш
  трудовой договор о работе над open source.
- **Политика использования ИИ**, если она есть. Урок 24.3 показал политику ripgrep: инструменты разрешены, но вы должны
  понимать изменение, объяснять его своими словами и писать комментарии сами.

> 🔒 **Безопасность:** sign-off (`-s`, строчная) — это *заявление*. Оно отличается от *криптографической подписи*
> коммита (`-S`, заглавная), которая с помощью вашего ключа доказывает, что коммит ваш (Урок 01.9). Проект может
> просить одно, другое или оба.

## 4. Процесс с форком в локальной лаборатории

На GitHub или GitLab вы сделали бы форк настоящего проекта. Здесь вы строите ту же ситуацию из папок на своём компьютере,
чтобы отработать каждый шаг, никого не беспокоя. Вы играете две роли: **Max**, мейнтейнера небольшой библиотеки
`kvparse`, и **Ada**, впервые вносящую вклад (это вы).

```text
          upstream.git  (Max's original project)
            │      ▲
  "Fork"    │      │ pull request (on a real site)
            ▼      │
          fork.git  (Ada's copy on the server)
            │      ▲
   clone    │      │ git push
            ▼      │
          ada/  (Ada's working folder on her computer)
```

### Шаг 1 — Max публикует проект

```bash
mkdir -p ~/projects/git-playground/oss-lab
cd ~/projects/git-playground/oss-lab
git init -q --bare upstream.git
git clone -q upstream.git maintainer
cd maintainer
git config user.name "Max Maintainer"
git config user.email "max@example.com"
cargo init -q --lib --name kvparse
```

```text
warning: You appear to have cloned an empty repository.
```

Предупреждение ожидаемо: «сервер» пока пуст. Замените `src/lib.rs` первой версией от Max:

```rust
//! Parses `key=value` pairs, as in simple configuration files.

/// Splits `"key=value"` into its two parts, without spaces around them.
/// Returns `None` if there is no `=`.
pub fn parse_pair(line: &str) -> Option<(&str, &str)> {
    let (key, value) = line.split_once('=')?;
    Some((key.trim(), value.trim()))
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn splits_and_trims() {
        assert_eq!(parse_pair(" port = 8080 "), Some(("port", "8080")));
    }

    #[test]
    fn needs_an_equals_sign() {
        assert_eq!(parse_pair("port"), None);
    }
}
```

и создайте `CONTRIBUTING.md`:

```markdown
# Contributing to kvparse

- Open an issue before a large change.
- Every change needs a test. Run `cargo fmt`, `cargo clippy -- -D warnings`
  and `cargo test` before you push.
- Sign off every commit (`git commit -s`): this project uses the DCO.
```

Max тестирует, коммитит и публикует:

```bash
cargo test -q
git add .
git commit -q -m "Add parse_pair"
git push -q origin main
cd ..
```

```text

running 2 tests
..
test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s


running 0 tests

test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s

```

(Второй блок «running 0 tests» относится к документационным тестам; в библиотеке их нет.) Затем пользователь открывает
issue. Она придумана для лаборатории, но выглядит как настоящая:

```text
#12  parse_pair accepts an empty key

parse_pair(" = 8080") returns Some(("", "8080")). A line without a key
is a mistake in the configuration file and should be rejected (None).
```

### Шаг 2 — Ada делает форк и клонирует

На настоящем сайте кнопка **Fork** создаёт копию репозитория под вашей учётной записью. Здесь эту роль играет «голая»
копия. Затем Ada клонирует **свой форк**, задаёт свою идентичность для лаборатории и добавляет оригинал как `upstream`:

```bash
git clone -q --bare upstream.git fork.git
git clone -q fork.git ada
cd ada
git config user.name "Ada Learner"
git config user.email "ada@example.com"
git remote add upstream ../upstream.git
git remote -v
```

```text
origin	/Users/ada/projects/git-playground/oss-lab/fork.git (fetch)
origin	/Users/ada/projects/git-playground/oss-lab/fork.git (push)
upstream	../upstream.git (fetch)
upstream	../upstream.git (push)
```

Вместо `/Users/ada` будет ваша домашняя папка. Два имени важны: **`origin`** — ваш форк, куда вы можете отправлять
изменения; **`upstream`** — оригинал, из которого вы только получаете изменения. (В настоящем проекте вы использовали бы
свою глобальную идентичность из Урока 01.2; строки `git config` здесь лишь делают двух людей лаборатории разными.)

### Шаг 3 — Ветка и падающий тест

Никогда не работайте в `main`. Создайте ветку, названную по изменению, затем **сначала** напишите тест из issue
(Урок 09.6). Добавьте его внутрь `mod tests` в `src/lib.rs`:

```bash
git switch -c fix-empty-key
```

```rust
    #[test]
    fn rejects_an_empty_key() {
        assert_eq!(parse_pair(" = 8080"), None);
    }
```

```bash
cargo test
```

```text
running 3 tests
test tests::needs_an_equals_sign ... ok
test tests::splits_and_trims ... ok
test tests::rejects_an_empty_key ... FAILED

failures:

---- tests::rejects_an_empty_key stdout ----

thread 'tests::rejects_an_empty_key' (6636791) panicked at src/lib.rs:26:9:
assertion `left == right` failed
  left: Some(("", "8080"))
 right: None
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace


failures:
    tests::rejects_an_empty_key

test result: FAILED. 2 passed; 1 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s

error: test failed, to rerun pass `--lib`
```

Тест точно воспроизводит issue: `left` — то, что возвращает код, `right` — то, что просит issue. (Число после имени потока —
ID потока, его идентификатор; оно меняется при каждом запуске.)

### Шаг 4 — Исправление и проверки проекта

Измените `parse_pair` и её документацию:

```rust
/// Splits `"key=value"` into its two parts, without spaces around them.
/// Returns `None` if there is no `=` or if the key is empty.
pub fn parse_pair(line: &str) -> Option<(&str, &str)> {
    let (key, value) = line.split_once('=')?;
    let key = key.trim();
    if key.is_empty() {
        return None;
    }
    Some((key, value.trim()))
}
```

Затем запустите ровно те проверки, которых требует `CONTRIBUTING.md`:

```bash
cargo test
cargo fmt --check && cargo clippy -q -- -D warnings && echo "all checks passed"
```

```text
running 3 tests
test tests::needs_an_equals_sign ... ok
test tests::rejects_an_empty_key ... ok
test tests::splits_and_trims ... ok

test result: ok. 3 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

```text
all checks passed
```

### Шаг 5 — Коммит с sign-off и push в форк

```bash
git add src/lib.rs
git commit -s -m "Reject empty keys in parse_pair" -m "A line like \"= 8080\" returned an empty key. Return None instead.

Fixes #12"
git log -1 --format='%h %s%n%n%b'
```

```text
[fix-empty-key 12cc175] Reject empty keys in parse_pair
 1 file changed, 11 insertions(+), 2 deletions(-)
12cc175 Reject empty keys in parse_pair

A line like "= 8080" returned an empty key. Return None instead.

Fixes #12

Signed-off-by: Ada Learner <ada@example.com>
```

**Что происходит:**

1. Первый `-m` — строка темы, второй `-m` — тело сообщения (Урок 01.3). Тело объясняет *почему*.
2. `Fixes #12` — ключевое слово, которое понимают GitHub и GitLab: когда изменение принимают в основную ветку,
   issue #12 закрывается автоматически.
3. `-s` добавил строку `Signed-off-by:` из ваших `user.name` и `user.email`, как требует DCO.
4. Хеши коммитов (здесь `12cc175`) на вашем компьютере будут другими.

Отправьте ветку в **свой форк**:

```bash
git push
```

```text
To /Users/ada/projects/git-playground/oss-lab/fork.git
 * [new branch]      fix-empty-key -> fix-empty-key
branch 'fix-empty-key' set up to track 'origin/fix-empty-key'.
```

Простой `git push` пошёл в `origin` и создал там ветку благодаря настройке `push.autoSetupRemote` из Урока 01.2.
На настоящем сайте вы бы теперь открыли pull request из `your-account:fix-empty-key` в `upstream:main`.

### Шаг 6 — Upstream ушёл вперёд: rebase и обновление

Пока Ada ждёт ревью, Max принимает другое изменение. Сыграйте ненадолго за Max:

```bash
cd ../maintainer
printf '# kvparse\n\nParses `key=value` lines.\n' > README.md
git add README.md
git commit -q -m "Add README"
git push -q
cd ../ada
```

Ветка Ada теперь основана на старом `main`. Многие проекты просят сделать **rebase** на новейший код upstream, чтобы
история оставалась прямой, а CI проверял ваше изменение вместе с последним кодом:

```bash
git fetch upstream
git rebase upstream/main
cargo test -q
git log --oneline --graph -3
```

```text
From ../upstream
 * [new branch]      main       -> upstream/main
Successfully rebased and updated refs/heads/fix-empty-key.
```

```text
test result: ok. 3 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

```text
* ebfde23 Reject empty keys in parse_pair
* 47044c0 Add README
* a642456 Add parse_pair
```

Rebase дал коммиту Ada новый хеш (`ebfde23`), а в форке остался старый. Обычный push был бы отклонён. Замените ветку
в форке, безопасно:

```bash
git push --force-with-lease
```

```text
To /Users/ada/projects/git-playground/oss-lab/fork.git
 + 12cc175...ebfde23 fix-empty-key -> fix-empty-key (forced update)
```

`--force-with-lease` перезаписывает ветку, только если она всё ещё там, где вы видели её в последний раз. Если кто-то
(или вы сами с другого компьютера) за это время отправил в неё изменения, push остановится вместо того, чтобы уничтожить
эту работу. Используйте его только на **своих** ветках в **своём** форке, никогда на `main` и никогда на общей ветке
(Урок 01.9, «золотое правило rebase»).

## 5. Описание pull request

Описание — первое, что читает мейнтейнер. Делайте его коротким и полным:

```markdown
Fixes #12.

## What
`parse_pair` now returns `None` when the key is empty, for example for `" = 8080"`.

## Why
An empty key is a mistake in the configuration file; accepting it hid the mistake.

## How I tested it
- Added the test `rejects_an_empty_key` (it failed before the fix).
- `cargo fmt --check`, `cargo clippy -- -D warnings` and `cargo test` pass locally.
```

Хороший первый pull request **маленький** (одна issue, одна идея), **протестированный**, **следует правилам**
`CONTRIBUTING.md` и **не трогает** посторонний код. Если по ходу работы вы заметили другую проблему, откройте для неё
отдельную issue.

> 🧪 **Тестирование:** всегда показывайте, что ваш тест падал до исправления. Тест, который никогда не был красным, может
> ничего не проверять (Урок 05.6). Многие ревьюеры смотрят на это в первую очередь.

После открытия запускается CI. Если он упал, прочитайте лог (Урок 09.12), исправьте проблему в той же ветке и снова
отправьте изменения; pull request обновится сам. Затем ждите. Мейнтейнеры часто волонтёры; Урок 24.5 объясняет, как
пережить ожидание и ревью.

## Частые ошибки

**Ваша новая ветка отслеживает `upstream`.** Если создать ветку прямо из `upstream/main`, Git настроит её на отслеживание
`upstream`, и простой `git push` откажет:

```bash
git switch -c try-tracking upstream/main
git push
```

```text
Switched to a new branch 'try-tracking'
branch 'try-tracking' set up to track 'upstream/main'.
fatal: The upstream branch of your current branch does not match
the name of your current branch.  To push to the upstream branch
on the remote, use

    git push upstream HEAD:main

To push to the branch of the same name on the remote, use

    git push upstream HEAD

To choose either option permanently, see push.default in 'git help config'.

To avoid automatically configuring an upstream branch when its name
won't match the local branch, see option 'simple' of branch.autoSetupMerge
in 'git help config'.
```

**Не** следуйте подсказкам: обе отправили бы изменения в исходный проект (на настоящем сайте у вас обычно и нет таких прав).
Создавайте такие ветки без отслеживания, `git switch -c <name> --no-track upstream/main`, или явно отправляйте их
в свой форк через `git push -u origin <name>`. Уберите эту пробную ветку командами `git switch fix-empty-key`
и `git branch -D try-tracking`.

**Забытый sign-off.** В проектах с DCO проверка pull request упадёт. Исправьте последний коммит через
`git commit --amend -s --no-edit` и отправьте с `--force-with-lease` (Упражнение 2).

## Попробуйте сами

1. Постройте лабораторию из раздела 4 и сами выполните каждый шаг, включая rebase.
2. Напишите описание pull request для своего изменения в `notes/oss-log.md`.
3. Выберите один настоящий проект, которым пользуетесь. Проверьте четыре признака из раздела 2 и прочитайте его
   `CONTRIBUTING.md`. Запишите: DCO, CLA или ни то ни другое; обязательные проверки; метку для задач новичков.
4. Найдите одну issue с меткой для новичков и напишите (в заметках, пока не на сайте) короткий комментарий, которым вы
   спросили бы, можно ли над ней поработать.

## Упражнения

### Упражнение 1 — Issue #13 (средне)

Приходит вторая issue: «`parse_pair("my port = 8080")` возвращает ключ с пробелом внутри. Ключи должны быть одним словом».
В лаборатории начните новую ветку `reject-spaces` от новейшего кода upstream (без отслеживания), сначала напишите тест,
исправьте код и сделайте коммит. Пробелы **вокруг** ключа по-прежнему должны быть разрешены.

<details>
<summary>Подсказка</summary>

`str::contains` принимает функцию: `key.contains(char::is_whitespace)` равно `true`, если какой-нибудь символ — пробел,
табуляция или другой пробельный символ. Сначала обрежьте пробелы, потом проверяйте.

</details>

<details>
<summary>Ответ</summary>

```bash
git switch -c reject-spaces --no-track upstream/main
```

Функция и новый тест (в `src/lib.rs`; эта ветка начинается от upstream, поэтому в ней ещё нет первого исправления Ada):

```rust
/// Splits `"key=value"` into its two parts, without spaces around them.
/// Returns `None` if there is no `=` or if the key contains a space.
pub fn parse_pair(line: &str) -> Option<(&str, &str)> {
    let (key, value) = line.split_once('=')?;
    let key = key.trim();
    if key.contains(char::is_whitespace) {
        return None;
    }
    Some((key, value.trim()))
}

#[test]
fn rejects_a_key_with_spaces_inside() {
    assert_eq!(parse_pair("my port = 8080"), None);
    assert_eq!(parse_pair("  port = 8080"), Some(("port", "8080")));
}
```

```text
running 3 tests
test tests::needs_an_equals_sign ... ok
test tests::splits_and_trims ... ok
test tests::rejects_a_key_with_spaces_inside ... ok

test result: ok. 3 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

Затем `git add src/lib.rs` и `git commit -m "Reject keys with spaces inside"` (без `-s`, намеренно: Упражнение 2
это исправит). Почему это работает: `trim` сначала убирает пробелы вокруг ключа, поэтому отклоняются только пробелы
*внутри* него. Две отдельные ветки — это два отдельных pull request, каждый легко проверить; мейнтейнер может принять
их в любом порядке.

</details>

### Упражнение 2 — Забытый sign-off (легко)

Вы закоммитили в `reject-spaces` без `-s`. Добавьте sign-off к последнему коммиту и отправьте ветку в свой форк.
Покажите итоговое сообщение коммита.

<details>
<summary>Подсказка</summary>

`--amend` переписывает последний коммит; `--no-edit` сохраняет его сообщение. В форке у ветки ещё нет пары, поэтому
первый push её создаст.

</details>

<details>
<summary>Ответ</summary>

```bash
git commit --amend -s --no-edit
git push
git log -1 --format=%B
```

```text
[reject-spaces 330f9c9] Reject keys with spaces inside
 Date: Wed Nov 18 09:20:03 2026 +0000
 1 file changed, 12 insertions(+), 2 deletions(-)
To /Users/ada/projects/git-playground/oss-lab/fork.git
 * [new branch]      reject-spaces -> reject-spaces
branch 'reject-spaces' set up to track 'origin/reject-spaces'.
Reject keys with spaces inside

Signed-off-by: Ada Learner <ada@example.com>
```

Почему это работает: `--amend` создаёт новый коммит с теми же изменениями и дополнительной строкой. Поскольку ветка
создана с `--no-track`, простой `git push` идёт в `origin`, ваш форк. Если бы вы уже отправили коммит без подписи,
понадобился бы `git push --force-with-lease`.

</details>

### Упражнение 3 — Хороший ли это первый проект? (легко, без кода)

Возьмите один проект из ваших файлов `Cargo.toml` этого курса. Заполните таблицу из раздела 2 тем, что найдёте, и решите:
хороший ли это первый проект для вас? Приведите две причины.

<details>
<summary>Подсказка</summary>

Страница крейта на crates.io ссылается на его репозиторий. Посмотрите на даты последних коммитов и последних issue,
на которые ответили.

</details>

<details>
<summary>Ответ</summary>

Единственного ответа нет. Хороший ответ заполняет все четыре строки фактами (даты, имена файлов, статус CI) и приводит
причины вроде «я пользовался им на Этапе 16, мейнтейнеры отвечают в течение недели, и в нём пять открытых задач
с меткой `good first issue`». «Слишком большой для первого вклада» — тоже правильный вывод; тогда выберите проект поменьше из тех, которыми
пользуетесь.

</details>

## Вопросы для самопроверки

1. Что такое `origin` и `upstream` в процессе с форком?

<details>
<summary>Ответ</summary>

`origin` — ваш форк, куда вы отправляете свои ветки. `upstream` — исходный проект, из которого вы получаете изменения,
чтобы не отставать. В `upstream` вы никогда не отправляете изменения.

</details>

2. Чем DCO отличается от CLA?

<details>
<summary>Ответ</summary>

DCO — заявление, которое вы делаете в каждом коммите строкой `Signed-off-by:` (`git commit -s`). CLA — юридическое
соглашение, которое вы подписываете один раз, обычно онлайн, и которое даёт проекту права на ваш вклад.

</details>

3. Почему нужно спросить в issue, прежде чем начинать большое изменение?

<details>
<summary>Ответ</summary>

Мейнтейнерам может быть не нужно это изменение, они могут хотеть сделать его иначе, или над ним уже кто-то работает.
Короткий вопрос избавляет обе стороны от напрасной работы.

</details>

4. Почему после rebase нужен `git push --force-with-lease`, а не `git push --force`?

<details>
<summary>Ответ</summary>

`--force-with-lease` отказывается перезаписывать удалённую ветку, если она изменилась с вашего последнего fetch, поэтому
вы не уничтожите работу, которую не видели. `--force` перезаписывает в любом случае.

</details>

5. Что делает `Fixes #12` в сообщении коммита или pull request?

<details>
<summary>Ответ</summary>

Связывает изменение с issue 12, а на GitHub и GitLab автоматически закрывает issue, когда изменение принимают в ветку
по умолчанию.

</details>

## Итоги

- Вклад — это и отчёты об ошибках, и документация, и тесты, и ответы, а не только код. Сначала добейтесь одного
  небольшого принятого изменения.
- Выбирайте проект, которым пользуетесь и который жив; берите issue с меткой для новичков, читайте обсуждение и спрашивайте
  до начала работы.
- Читайте `CONTRIBUTING.md`, кодекс поведения, лицензию и правила вроде DCO, CLA или политики использования ИИ.
- Процесс с форком: форк, клон своего форка (`origin`), добавить `upstream`, ветка, сначала тест, исправление, проверки,
  коммит с `-s`, push в `origin`, открыть pull request.
- Не отставайте: `git fetch upstream` и `git rebase upstream/main`, затем `git push --force-with-lease` в свою ветку.
- Хороший pull request маленький, протестированный, объяснённый и ограничен одной идеей.

## Источники и что почитать

- [Open Source Guides — How to Contribute to Open Source](https://opensource.guide/how-to-contribute/) — дружелюбный
  обзор всего процесса.
- [GitHub Docs — Contributing to a project](https://docs.github.com/en/get-started/exploring-projects-on-github/contributing-to-a-project) и
  [GitLab Docs — Forking workflow](https://docs.gitlab.com/user/project/repository/forking_workflow/) — процесс с форком
  на двух крупных сайтах.
- [GitHub Docs — Linking a pull request to an issue](https://docs.github.com/en/issues/tracking-your-work-with-issues/using-issues/linking-a-pull-request-to-an-issue) —
  ключевые слова вроде `Fixes #12`.
- [Developer Certificate of Origin](https://developercertificate.org/) — полный текст, с которым вы соглашаетесь через `git commit -s`.
- [Contributor Covenant](https://www.contributor-covenant.org/) и [Rust Code of Conduct](https://www.rust-lang.org/policies/code-of-conduct) —
  два широко используемых кодекса поведения.
- [Rust Forge — Issue assignment](https://forge.rust-lang.org/triagebot/issue-assignment.html) и
  [Rust Compiler Development Guide — Contributing](https://rustc-dev-guide.rust-lang.org/contributing.html) — как устроен
  вклад в репозитории rust-lang, включая `@rustbot claim`.
- [This Week in Rust](https://this-week-in-rust.org/) — еженедельный раздел «Call for Participation» с задачами для новых
  участников.

---

[🏠 Начало](../START-HERE.md) · [← 24.3 Как читать большие кодовые базы](03-reading-large-codebases.md) · [📋 Этап 24](README.md) · [24.5 Работа с мейнтейнерами и сообществом →](05-working-with-maintainers-and-communities.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
