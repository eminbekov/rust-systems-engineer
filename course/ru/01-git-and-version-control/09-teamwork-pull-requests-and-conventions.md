<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 01.8 Удалённые репозитории: GitHub и GitLab](08-remotes-github-and-gitlab.md) · [📋 Этап 01](README.md) · [01.10 Проект: ваш учебный репозиторий и контрольная точка →](10-project-your-learning-repo-and-checkpoint.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/01-git-and-version-control/09-teamwork-pull-requests-and-conventions.md) <!-- nav -->

# Урок 01.9 — Командная работа: pull request, код-ревью и соглашения

> **Этап 01 — Git и контроль версий** · Урок 9 из 10 · ⏱ около 3,5 часа (2 занятия)

## Что вы узнаете

- Рабочий процесс с pull request, которым пользуется почти каждая команда, шаг за шагом.
- Как написать хороший pull request, как давать и получать код-ревью.
- Как обновлять свою ветку через `rebase`, как приводить коммиты в порядок интерактивным rebase и одно правило, которое нельзя нарушать.
- Соглашения о сообщениях коммитов (Conventional Commits), номера версий (SemVer) и теги.
- Что такое непрерывная интеграция (CI) и как автоматически проверять проект на Rust при каждом push.
- Полезные дополнения: `cherry-pick`, хуки Git, защищённые ветки и файлы, которые обычно есть в командном репозитории.

## Перед началом

- Вы прошли [Урок 01.8 — Удалённые репозитории: GitHub и GitLab](08-remotes-github-and-gitlab.md).
- Вы знаете ветки, слияние и конфликты ([Урок 01.6](06-branches-and-merging.md), [Урок 01.7](07-merge-conflicts.md)).
- Аккаунт GitHub полезен, но для чтения этого урока не обязателен.

## Новые слова

| Слово | Значение |
|---|---|
| **PR** (Pull Request) | Запрос на GitHub слить вашу ветку в другую (обычно `main`) с местом для ревью и обсуждения. |
| **MR** (Merge Request) | То же самое на GitLab. |
| **код-ревью** (code review) | Другие люди читают ваши изменения и комментируют их до слияния. |
| **ревьюер** (reviewer) / **автор** (author) | Человек, проверяющий изменение / человек, который его написал. |
| **LGTM** («Looks Good To Me», «по-моему, всё хорошо») | Распространённое короткое одобрение в ревью. |
| **nit** | Крошечный необязательный комментарий в ревью («придирка»), например про именование. |
| **rebase** | Взять ваши коммиты и воспроизвести их поверх другого базового коммита. При этом создаются новые коммиты. |
| **squash** | Объединить несколько коммитов в один. |
| **cherry-pick** | Скопировать один коммит из другой ветки в текущую. |
| **fork** (форк) | Ваша собственная копия чужого репозитория на GitHub/GitLab. |
| **upstream** (во форках) | Исходный репозиторий, от которого вы сделали форк. |
| **защищённая ветка** (protected branch) | Ветка (например, `main`), в которую сервер не разрешает отправлять напрямую или делать force-push. |
| **Conventional Commits** | Соглашение о сообщениях коммитов: `type(scope): description`. |
| **SemVer** (Semantic Versioning, семантическое версионирование) | Правило номеров версий: `MAJOR.MINOR.PATCH`. |
| **тег** (tag) | Постоянное имя одного коммита, используется для релизов вроде `v1.2.0`. |
| **CI** (Continuous Integration, непрерывная интеграция) | Сервис, который автоматически собирает и тестирует ваш код при каждом push. |
| **CD** (Continuous Delivery / Deployment, непрерывная доставка / развёртывание) | Автоматическая доставка или развёртывание результата CI. Здесь вы только знакомитесь с идеей. |
| **YAML** (YAML Ain't Markup Language) | Текстовый формат конфигурации, основанный на отступах. Файлы CI написаны на нём. |
| **workflow** / **job** / **step** | В GitHub Actions: весь файл автоматизации / группа шагов, выполняемых на одной машине / одна команда или действие. |
| **runner** (исполнитель) | Машина (часто виртуальная, VM), которая запускает задачи CI. |
| **VM** (Virtual Machine, виртуальная машина) | Компьютер, имитируемый программно. |
| **хук** (hook) | Скрипт, который Git запускает автоматически в определённые моменты, например перед коммитом. |
| **Clippy** | Линтер Rust: инструмент, который находит подозрительный или неидиоматичный код. |
| **CLI** (Command-Line Interface, интерфейс командной строки) | Способ работать с программой, вводя команды. |

## 1. Зачем нужны pull request?

Представьте десять разработчиков, которые отправляют изменения прямо в `main`. Рано или поздно кто-то отправит код, который не компилируется, и все окажутся заблокированы. **Pull request** добавляет контрольную точку между «моей работой» и «общей веткой»:

- кто-то другой **читает** изменение (код-ревью): меньше ошибок, общее знание;
- на предлагаемом изменении запускаются **автоматические проверки** (CI);
- есть **обсуждение** и запись о том, зачем сделано изменение;
- `main` **всегда остаётся рабочей**.

(Название означает: «пожалуйста, *втяни* (pull) мои изменения в свою ветку».) На GitLab это *merge request*; идея та же.

## 2. Рабочий процесс шаг за шагом

Это «GitHub flow». Он простой и работает и для маленьких, и для больших команд.

```text
 main:   ●────────────●───────────────●
          \          /               ▲
 branch:   ●──●──●──●  ── review ── merge ── delete branch
           (commits)   (open PR)
```

| № | Шаг | Команды |
|---|---|---|
| 1 | Начните с актуальной `main` | `git switch main`, затем `git pull` |
| 2 | Создайте ветку для **одной** задачи | `git switch -c feature/short-name` |
| 3 | Делайте небольшие осмысленные коммиты | `git add …` и `git commit` |
| 4 | Отправьте ветку | `git push` (в первый раз `git push -u origin feature/short-name`; с нашей настройкой `-u` ставится автоматически) |
| 5 | Откройте pull request на сайте (или `gh pr create`) | |
| 6 | Запускается CI; ревьюеры комментируют; вы отправляете новые коммиты | снова `git push`; PR обновляется сам |
| 7 | PR одобрен и **слит** (кнопкой) | |
| 8 | Уборка: удалите ветку, обновите свою `main` | `git switch main`, `git pull`, `git branch -d feature/short-name` |

С GitHub CLI средние шаги выглядят так (этим командам нужен настоящий репозиторий на GitHub):

```bash
git push -u origin feature/short-name
gh pr create --fill
gh pr list
gh pr checkout 12
gh pr merge --squash --delete-branch
```

**Что происходит:** `gh pr create --fill` открывает PR, взяв заголовок и описание из ваших коммитов. `gh pr list` перечисляет открытые PR. `gh pr checkout 12` скачивает ветку PR номер 12, чтобы вы могли проверить работу коллеги. `gh pr merge --squash --delete-branch` сливает со стратегией squash (раздел 5) и удаляет ветку.

> 💡 **Совет:** для незаконченной работы открывайте **черновой** PR сразу (`gh pr create --draft`). Отметьте его «готовым к ревью», когда закончите.

## 3. Как написать хороший pull request

Ревьюер — занятый человек. Облегчите ему работу.

1. **Одна цель.** Один PR = одно изменение. Если PR делает две вещи, сделайте два PR.
2. **Небольшой.** Стремитесь к менее чем примерно 400 изменённым строкам. Большие PR просматривают по диагонали, а не ревьюят.
3. **Понятный заголовок** в том же стиле, что и тема коммита: «Fix crash when the config file is empty».
4. **Полезное описание.** Шаблон, который работает везде:

```text
## What
Short summary of the change.

## Why
The problem it solves, or a link to the issue (for example: Closes #12).

## How
Anything non-obvious about the approach.

## Testing
How you checked it: commands run, tests added.
```

5. **Сначала проверьте свой PR сами.** Откройте вкладку «Files changed» и прочитайте её как посторонний. Вы заметите отладочные выводы и лишние файлы.
6. **Зелёные проверки.** Не просите ревью, пока CI красный.

В описании PR `Closes #12` (или `Fixes #12`) связывает PR с задачей 12 и автоматически закрывает задачу при слиянии PR.

Команда может положить готовый шаблон в `.github/pull_request_template.md`; GitHub подставит его в описание каждого нового PR.

## 4. Код-ревью

### Когда вы автор

- Ревью — о **коде**, а не о вас. Поблагодарите людей за их время.
- Отвечайте на каждый комментарий, даже просто «Done». Если не согласны, спокойно объясните фактами.
- Во время ревью отправляйте исправления **новыми коммитами**, чтобы ревьюеры видели, что изменилось. (В конце можно склеить.)
- Если комментарий непонятен, задайте вопрос.

### Когда вы ревьюер

Читайте в таком порядке: *решает ли это задачу?* → *правильно ли и протестировано ли?* → *читаемо ли?* → *детали стиля*. Какие вопросы задавать:

| Область | Спросите |
|---|---|
| Корректность | Делает ли это то, что сказано в PR? Что с пустым вводом, ошибками, большим вводом? |
| Тесты | Есть ли тест, который упал бы без этого изменения? |
| Дизайн | Просто ли это? Вписывается ли в существующий код? |
| Читаемость | Понятны ли имена? Поймёт ли новый коллега? |
| Безопасность | Не принимается ли какой-нибудь ввод на веру, без проверки? Нет ли секрета в коде? |
| Rust | Проходит ли `cargo fmt --check`? `cargo clippy` без предупреждений? Нет ли `unwrap()`, который может вызвать панику в реальном использовании? |

Доброжелательные и понятные стили комментариев:

- `nit: rename x to count (optional)` — небольшое необязательное предложение.
- `question: why is this loop needed?` — вы спрашиваете, а не требуете.
- `suggestion: this could use …` — идея.
- `blocking: this panics on empty input` — нужно исправить до слияния.
- `LGTM` или *Approve*, когда вы довольны.

📏 **Соглашение:** комментируйте код, а не человека («эту функцию трудно понять», а не «вы пишете запутанный код»). Хвалите и хорошую работу.

## 5. Как сливается PR: три стратегии

Когда вы нажимаете *Merge*, GitHub и GitLab предлагают такие варианты:

| Стратегия | Что получает `main` | Форма истории |
|---|---|---|
| **Merge commit** | все коммиты ветки **плюс** один коммит слияния | ветвления и слияния видны (граф с развилками) |
| **Squash and merge** | **один** новый коммит, содержащий весь PR | прямая линия; один коммит на PR; неряшливые «wip»-коммиты исчезают |
| **Rebase and merge** | коммиты ветки воспроизводятся по одному поверх `main`, без коммита слияния | прямая линия; каждый коммит сохраняется |

```text
 merge commit:         squash:              rebase and merge:
 ●─●───────●  main      ●─●─S  main          ●─●─a'─b'─c'  main
  \       /              (S = a+b+c)
   a──b──c
```

Команды выбирают одну стратегию и используют её везде. **Squash and merge** очень популярна для небольших PR, потому что `main` получает по одному аккуратному коммиту на PR (его сообщение должно следовать соглашениям ниже). После squash-слияния `git branch -d` может сказать, что ветка «not fully merged», потому что склеенный коммит — это другой коммит; убедившись, что PR слит, можно смело использовать `git branch -D`.

## 6. Как держать ветку актуальной: `rebase`

Пока ваш PR открыт, `main` уходит вперёд. Нужно забрать эти изменения в свою ветку. Есть два способа: **слить** `main` в вашу ветку (вы это знаете) или сделать **rebase** вашей ветки на `main`.

Rebase берёт ваши коммиты, временно откладывает их, переносит вашу ветку на самую новую `main` и **воспроизводит** ваши коммиты сверху. Потренируйтесь на небольшом репозитории:

```bash
cd ~/projects/git-playground
mkdir rebase-demo
cd rebase-demo
git init
echo "one" > a.txt
git add a.txt
git commit -m "A: start"
git switch -c feature
echo "feature work" > feature.txt
git add feature.txt
git commit -m "F1: add feature file"
echo "more feature work" >> feature.txt
git commit -am "F2: extend feature file"
git switch main
echo "two" >> a.txt
git commit -am "B: update a.txt on main"
echo "three" >> a.txt
git commit -am "C: update a.txt again"
git log --oneline --graph --all
```

```text
* 5b4cd87 F2: extend feature file
* 5a1c6b0 F1: add feature file
| * 78f9dc8 C: update a.txt again
| * 867371c B: update a.txt on main
|/
* 42d4976 A: start
```

Теперь сделайте rebase `feature` на `main`:

```bash
git switch feature
git rebase main
git log --oneline --graph --all
```

```text
Rebasing (1/2)Rebasing (2/2)Successfully rebased and updated refs/heads/feature.
* 40e6f13 F2: extend feature file
* 4c229ba F1: add feature file
* 78f9dc8 C: update a.txt again
* 867371c B: update a.txt on main
* 42d4976 A: start
```

```text
before:  A ── B ── C            (main)         after:  A ── B ── C ── F1' ── F2'   (feature)
          \                                                    (main is at C)
           F1 ── F2   (feature)
```

**Что происходит:** история теперь — **прямая линия**. Посмотрите на хеши: `F1` был `5a1c6b0`, а теперь `4c229ba`. Коммиты `F1'` и `F2'` — **новые коммиты** с теми же изменениями и сообщениями, но другим родителем, поэтому у них другие хеши. Слияние `feature` в `main` теперь — чистый fast-forward:

```bash
git switch main
git merge feature
```

```text
Updating 78f9dc8..40e6f13
Fast-forward
 feature.txt | 2 ++
 1 file changed, 2 insertions(+)
 create mode 100644 feature.txt
```

### Золотое правило rebase

> ⚠️ **Частая ошибка:** **никогда не делайте rebase коммитов, которые уже есть у других людей** (коммитов, отправленных в общую ветку вроде `main`). Rebase создаёт новые коммиты с новыми хешами. Если кто-то построил работу на старых, вы создадите дубликаты и конфликты для всех. Делайте rebase **только своей, ещё не опубликованной работы** или ветки PR, которой пользуетесь только вы.

Если вы делаете rebase ветки, которую **уже отправили** (например, вашей собственной ветки PR), удалённый репозиторий откажет в обычном push, потому что истории различаются. Только для своей ветки используйте осторожный force:

```bash
git push --force-with-lease
```

`--force-with-lease` перезаписывает удалённую ветку **только если она всё ещё такая, какой вы видели её в последний раз**. Если за это время коллега отправил что-то в вашу ветку, команда откажется:

```text
To /Users/ada/projects/git-playground/fwl/remote.git
 ! [rejected]        feature -> feature (stale info)
error: failed to push some refs to '/Users/ada/projects/git-playground/fwl/remote.git'
```

(Заметьте: если сначала выполнить `git fetch`, Git решит, что вы видели новые коммиты, и force пройдёт, перезаписав их. Поэтому **смотрите**, что собираетесь перезаписать.) У обычного `--force` такой защиты нет. Никогда не используйте его на `main`.

### Конфликты во время rebase

Rebase воспроизводит коммиты по одному, поэтому конфликты могут появиться посередине. Рецепт почти тот же, что в [Уроке 01.7](07-merge-conflicts.md):

```text
Auto-merging doc.txt
CONFLICT (content): Merge conflict in doc.txt
error: could not apply 4d71444... Change title on topic
hint: Resolve all conflicts manually, mark them as resolved with
hint: "git add/rm <conflicted_files>", then run "git rebase --continue".
hint: You can instead skip this commit: run "git rebase --skip".
hint: To abort and get back to the state before "git rebase", run "git rebase --abort".
```

1. Откройте файл, уберите маркеры, составьте итоговый текст.
2. `git add <file>`.
3. `git rebase --continue` (Git может открыть редактор для сообщения; сохраните и закройте).
4. Повторите, если конфликтует следующий коммит.
5. Чтобы сдаться: `git rebase --abort` вернёт всё, как было до этого.

Помните из Урока 01.7: во время rebase «ours» — это ветка, **на которую** вы делаете rebase, а «theirs» — **ваш собственный коммит**, который воспроизводится. Это кажется наоборот, поэтому проверяйте результат.

## 7. Наведение порядка в коммитах: интерактивный rebase

Перед открытием PR вы можете захотеть привести в порядок неаккуратные коммиты («wip», «fix typo», «oops»). `git rebase -i` (interactive) позволяет редактировать список коммитов. Сделайте три небольших коммита:

```bash
git switch -c wip
echo "line 1" >> notes.txt
git add notes.txt
git commit -m "wip 1"
echo "line 2" >> notes.txt
git commit -am "wip 2"
echo "line 3" >> notes.txt
git commit -am "wip 3"
git log --oneline -4
git rebase -i HEAD~3
```

```text
c91c95e wip 3
bef0851 wip 2
b66cc6b wip 1
40e6f13 F2: extend feature file
```

`git rebase -i HEAD~3` открывает ваш редактор (nano) со списком, сначала самые старые:

```text
pick b66cc6b # wip 1
pick bef0851 # wip 2
pick c91c95e # wip 3

# Rebase 40e6f13..c91c95e onto 40e6f13 (3 commands)
#
# Commands:
# p, pick <commit> = use commit
# r, reword <commit> = use commit, but edit the commit message
# e, edit <commit> = use commit, but stop for amending
# s, squash <commit> = use commit, but meld into previous commit
# f, fixup [-C | -c] <commit> = like "squash" but keep only the previous
#                    commit's log message, unless -C is used, in which case
#                    keep only this commit's message; -c is same as -C but
#                    opens the editor
# x, exec <command> = run command (the rest of the line) using shell
# b, break = stop here (continue rebase later with 'git rebase --continue')
# d, drop <commit> = remove commit
…
```

Измените слово в начале строки, чтобы сказать Git, что делать с этим коммитом:

| Команда | Эффект |
|---|---|
| `pick` | оставить коммит как есть |
| `reword` | оставить коммит, отредактировать его сообщение |
| `squash` | влить в предыдущий коммит и позволить объединить сообщения |
| `fixup` | влить в предыдущий коммит и **отбросить** это сообщение |
| `drop` | удалить коммит |
| (поменять строки местами) | изменить порядок коммитов |

Чтобы влить `wip 2` и `wip 3` в `wip 1`, измените файл так:

```text
pick b66cc6b # wip 1
fixup bef0851 # wip 2
fixup c91c95e # wip 3
```

Сохраните и закройте редактор (в `nano`: `Ctrl+O`, `Enter`, `Ctrl+X`). Git воспроизведёт список:

```text
Rebasing (2/3)Rebasing (3/3)Successfully rebased and updated refs/heads/wip.
```

```bash
git log --oneline -3
```

```text
9317beb wip 1
40e6f13 F2: extend feature file
4c229ba F1: add feature file
```

**Что происходит:** три коммита стали одним (его сообщение — `wip 1`; используйте `reword`, если хотите лучшее). То же правило, что и раньше: переписывайте только коммиты, которые не опубликованы.

> 📏 **Соглашение:** в типичном PR несколько чистых логических коммитов, либо он склеивается при слиянии. Не тратьте часы на полировку; ясно и мелко — достаточно.

## 8. `cherry-pick`: взять один коммит

Иногда нужен ровно один коммит из другой ветки, например исправление ошибки, сделанное в боковой ветке, а `main` должна получить его прямо сейчас.

```bash
git switch -c hotfix
echo "fix" > fix.txt
git add fix.txt
git commit -m "Fix the crash"
git switch main
git cherry-pick hotfix
git log --oneline -3
```

```text
[main 59835de] Fix the crash
 Date: Tue Nov 17 16:58:53 2026 +0500
 1 file changed, 1 insertion(+)
 create mode 100644 fix.txt
59835de Fix the crash
40e6f13 F2: extend feature file
4c229ba F1: add feature file
```

**Что происходит:** `git cherry-pick hotfix` скопировал изменения **верхнего коммита** `hotfix` в `main` как новый коммит (новый хеш). Вместо имени ветки можно указать хеш коммита. Если возникнет конфликт, решите его как при слиянии и выполните `git cherry-pick --continue`. Применяйте редко; он дублирует коммиты.

## 9. Соглашения о сообщениях коммитов

Базовые правила вы встретили в [Уроке 01.3](03-your-first-repository-and-commits.md): короткая тема в повелительном наклонении, пустая строка, тело с объяснением *зачем*. Многие команды добавляют формат **Conventional Commits**, который читают и машины, и люди:

```text
<type>(<scope>): <description>

[optional body]

[optional footer, e.g. "Closes #12" or "BREAKING CHANGE: …"]
```

| Тип | Для чего |
|---|---|
| `feat` | новая возможность (попадает в журнал изменений; обычно версия MINOR) |
| `fix` | исправление ошибки (PATCH) |
| `docs` | только документация |
| `style` | только форматирование, без изменения смысла кода |
| `refactor` | изменение кода, которое не исправляет ошибку и не добавляет новую возможность |
| `perf` | улучшение производительности |
| `test` | добавление или исправление тестов |
| `build` | система сборки, зависимости (`Cargo.toml`) |
| `ci` | настройка CI |
| `chore` | прочее обслуживание |

Примеры:

```text
feat(parser): support quoted strings
fix: avoid panic on empty input
docs(readme): add build instructions
refactor(config)!: rename Settings to Config

BREAKING CHANGE: the public type Settings no longer exists.
```

Знак `!` после типа (или подвал `BREAKING CHANGE:`) отмечает изменение, ломающее код пользователей вашего кода (версия MAJOR). `(scope)` необязателен и называет часть проекта.

Зачем это нужно? Инструменты могут по таким коммитам сгенерировать журнал изменений и номер следующей версии, а `git log --oneline` становится легко просматривать. Если в вашей команде формат не принят, не навязывайте его; используйте базовые правила. Что бы вы ни выбрали, **будьте последовательны**.

## 10. Версии и теги: SemVer

Для **релиза** нужен номер версии. В мире Rust используют **семантическое версионирование** (SemVer): `MAJOR.MINOR.PATCH`, например `1.4.2`.

| Часть | Увеличивайте, когда вы… | Пример |
|---|---|---|
| MAJOR | делаете изменение, которое **ломает** код пользователей | `1.4.2` → `2.0.0` |
| MINOR | добавляете новую возможность с обратной совместимостью | `1.4.2` → `1.5.0` |
| PATCH | исправляете ошибку с обратной совместимостью | `1.4.2` → `1.4.3` |

До `1.0.0` (версии вроде `0.3.1`) API (Application Programming Interface, программный интерфейс: публичные функции и типы, которыми пользуется другой код) считается нестабильным, и изменение MINOR может что-то ломать. В Rust версия живёт в `Cargo.toml` (`version = "0.1.0"`). Сам Cargo следует SemVer для зависимостей (вы изучите это в Уроке 05.8).

**Тег** даёт коммиту постоянное имя:

```bash
git tag v0.1.0
git tag -a v0.2.0 -m "Second release"
git tag
git show v0.2.0 --stat
git log --oneline --decorate -3
git tag -d v0.1.0
```

```text
v0.1.0
v0.2.0
tag v0.2.0
Tagger: Ada Learner <ada@example.com>
Date:   Tue Nov 17 16:58:53 2026 +0500

Second release

commit 59835de1f808395ae8d48978f9c43ed161315d51
Author: Ada Learner <ada@example.com>
Date:   Tue Nov 17 16:58:53 2026 +0500

    Fix the crash

 fix.txt | 1 +
 1 file changed, 1 insertion(+)
59835de (HEAD -> main, tag: v0.2.0, tag: v0.1.0) Fix the crash
40e6f13 (feature) F2: extend feature file
4c229ba F1: add feature file
Deleted tag 'v0.1.0' (was 59835de)
```

**Что происходит:**

1. `git tag v0.1.0` создаёт **облегчённый** (lightweight) тег: просто имя текущего коммита.
2. `git tag -a v0.2.0 -m "…"` создаёт **аннотированный** (annotated) тег: настоящий объект с автором тега, датой и сообщением. Для релизов используйте аннотированные теги.
3. `git tag` перечисляет теги; `git show` показывает один; `git tag -d` удаляет (локально).

Теги **не** отправляются автоматически:

```bash
git push origin v0.2.0
git push origin --tags
```

Первая команда отправляет один тег, вторая — все теги. На GitHub отправленный тег может стать страницей **Release** с заметками и загрузками. По соглашению имена тегов начинаются с `v`: `v1.2.0`.

## 11. Защита `main` и командные файлы

Команды не полагаются только на дисциплину. В *Settings → Branches* (GitHub) администратор репозитория может добавить **правило защиты ветки** для `main`:

- требовать pull request (никаких прямых push);
- требовать определённое число одобрений;
- требовать успешного прохождения проверок CI;
- запретить force-push и удаление.

Типичные файлы в командном репозитории:

| Файл | Назначение |
|---|---|
| `README.md` | что за проект, как его собрать и использовать |
| `CONTRIBUTING.md` | как предлагать изменения (ветки, стиль коммитов, как запускать тесты) |
| `LICENSE` | что другие могут делать с кодом |
| `CHANGELOG.md` | что изменилось в каждой версии (см. формат «Keep a Changelog») |
| `.gitignore` | файлы, которые Git не должен отслеживать |
| `.github/pull_request_template.md` | описание PR по умолчанию |
| `.github/CODEOWNERS` | кто обязан ревьюить изменения каких файлов |
| `.github/workflows/*.yml` | GitHub Actions (CI) |

## 12. Непрерывная интеграция

**CI** означает, что робот собирает и тестирует каждое изменение. Результат появляется рядом с PR как зелёная галочка или красный крест. Для проекта на Rust стандартны три проверки:

| Проверка | Команда | Падает, когда |
|---|---|---|
| Форматирование | `cargo fmt --check` | код отформатирован не по стандарту |
| Линты | `cargo clippy --all-targets -- -D warnings` | Clippy находит проблему (`-D warnings` превращает предупреждения в ошибки) |
| Тесты | `cargo test` | падает любой тест |

Сначала запускайте их на своём компьютере. Вот как они выглядят для небольшой библиотеки (проект, созданный через `cargo new ci-demo --lib`):

```bash
cargo fmt --check
cargo clippy --all-targets -- -D warnings
cargo test
```

Все три проходят молча или с `ok`. Теперь посмотрите, как они падают. Плохо отформатированный код:

```text
Diff in /Users/ada/projects/git-playground/ci-demo/src/lib.rs:1:
-pub fn add(a: i32,b: i32) -> i32 {
-        a+b
+pub fn add(a: i32, b: i32) -> i32 {
+    a + b
 }
 
 #[cfg(test)]
```

(Исправьте командой `cargo fmt`, которая переписывает файлы.) Проблема линтера (лишний `return`):

```text
error: unneeded `return` statement
 --> src/lib.rs:2:5
  |
2 |     return a + b;
  |     ^^^^^^^^^^^^
  |
  = help: for further information visit https://rust-lang.github.io/rust-clippy/rust-1.100.0/index.html#needless_return
  = note: `-D clippy::needless-return` implied by `-D warnings`
  = help: to override `-D warnings` add `#[allow(clippy::needless_return)]`
help: remove `return`
  |
2 -     return a + b;
2 +     a + b
  |

error: could not compile `ci-demo` (lib) due to 1 previous error
```

(Код на Rust пока понимать не нужно; важно, как инструменты сообщают о проблемах. `fn`, `return` и тесты будут на Этапе 02.)

### Рабочий процесс GitHub Actions

Положите этот файл в `.github/workflows/ci.yml` вашего репозитория:

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:

jobs:
  check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v7
      - run: rustup toolchain install stable --profile minimal --component rustfmt,clippy
      - run: cargo fmt --check
      - run: cargo clippy --all-targets -- -D warnings
      - run: cargo test
```

**Что происходит:**

1. `name: CI` — название, которое показывает GitHub.
2. `on:` говорит, когда запускать: при каждом push в `main` и при каждом pull request.
3. `jobs:` перечисляет задачи; здесь одна задача `check`.
4. `runs-on: ubuntu-latest` выбирает исполнитель: свежую виртуальную машину Linux, которую предоставляет GitHub.
5. `steps:` выполняются по порядку на этой машине. `uses: actions/checkout@v7` — готовое действие, которое скачивает ваш репозиторий на машину. Каждая строка `run:` — команда оболочки. Если команда падает, задача красная.
6. Строка `rustup … --component rustfmt,clippy` гарантирует, что форматтер и Clippy установлены.

YAML использует **отступы пробелами** (никогда не табуляцией) для обозначения структуры; один неверный пробел ломает файл.

Та же идея на GitLab — файл `.gitlab-ci.yml` в корне репозитория:

```yaml
stages: [check]

rust-checks:
  stage: check
  image: rust:1
  script:
    - rustup component add rustfmt clippy
    - cargo fmt --check
    - cargo clippy --all-targets -- -D warnings
    - cargo test
```

(`image: rust:1` — официальный образ контейнера Rust; контейнеры вы изучите гораздо позже.)

> 🧪 **Тестирование:** CI — страховочная сетка для тестов: он запускает их для всех, на чистой машине, каждый раз. В вашем PR красный крест означает «пока не сливать».

## 13. Хуки Git: проверки на вашем компьютере

**Хук** (hook) — это скрипт в `.git/hooks/`, который Git запускает в определённый момент. Хук `pre-commit` запускается перед каждым коммитом и может его остановить:

```bash
cat > .git/hooks/pre-commit <<'EOF'
#!/bin/sh
# Stop the commit when the code is not formatted.
if ! cargo fmt --check > /dev/null 2>&1; then
    echo "pre-commit: run 'cargo fmt' before committing"
    exit 1
fi
EOF
chmod +x .git/hooks/pre-commit
```

Теперь коммит с плохо отформатированным кодом отклоняется:

```text
pre-commit: run 'cargo fmt' before committing
```

(В проекте на Rust выполняйте это в папке проекта. Первая строка `#!/bin/sh` говорит, какая программа запускает скрипт, а `chmod +x` делает файл исполняемым.)

В экстренном случае хуки можно пропустить через `git commit --no-verify`. Помните, что `.git/hooks/` **не** входит в репозиторий: ваши коллеги не получают ваши хуки. Поэтому **настоящий** сторож — это CI, а хуки — лишь удобство для вас.

## 14. Работа над чужим проектом: форки

В проект, которым вы не владеете, push сделать нельзя. В открытом исходном коде вы:

1. Делаете **fork** репозитория (кнопка на GitHub): вы получаете свою копию в своём аккаунте.
2. **Клонируете** свой форк. Добавьте оригинал как второй удалённый репозиторий с именем `upstream`: `git remote add upstream git@github.com:owner/project.git`.
3. Создаёте ветку, коммитите, делаете `git push` в **ваш** форк (`origin`).
4. Открываете pull request из ветки вашего форка в исходный репозиторий.
5. Поддерживаете форк актуальным: `git fetch upstream` и `git rebase upstream/main` (или слияние).

Вклад в открытый исходный код вы отработаете на более позднем этапе; идеи те же, что выше.

> 🔒 **Безопасность:** в коммите можно указать любое имя автора. Чтобы доказать, что коммит действительно ваш, коммиты можно **подписывать** своим SSH-ключом (SSH — Secure Shell); GitHub тогда показывает значок «Verified». Если интересно, посмотрите страницу GitHub «About commit signature verification» в источниках. Пока это необязательно.

## Частые ошибки

> ⚠️ **Частая ошибка:** PR с 40 файлами и тремя несвязанными темами. Разделите его. Небольшие PR ревьюят быстрее, и они ломают меньше.

> ⚠️ **Частая ошибка:** отправлять изменения прямо в `main` «только в этот раз». Вот так `main` и ломается. Используйте ветку и PR, даже когда работаете в одиночку, чтобы выработать привычку.

> ⚠️ **Частая ошибка:** сделать rebase общей ветки, а затем force-push. Вы переписываете чужую историю. Делайте rebase только своих личных веток.

> ⚠️ **Частая ошибка:** `git add .` перед открытием PR, и в «Files changed» появляется `.env` или отладочный файл. Всегда читайте список файлов своего PR, прежде чем просить ревью.

> ⚠️ **Частая ошибка:** игнорировать красный CI. Красная проверка — это отчёт об ошибке. Откройте лог, найдите упавшую команду, воспроизведите её локально той же командой, исправьте.

> ⚠️ **Частая ошибка:** расплывчатый коммит вроде `update` в PR, который сливается способом «merge commit». В `main` он останется навсегда. Используйте squash-слияние или сначала приведите всё в порядок.

> ⚠️ **Частая ошибка:** непоследовательные отступы YAML в файлах CI. Если рабочий процесс не появляется на GitHub, сначала проверьте отступы.

## Попробуйте сами

1. В `rebase-demo` (из раздела 6) создайте новую ветку `docs`, добавьте коммит, передвиньте `main` вперёд на один коммит, затем сделайте rebase `docs` на `main` и слейте её fast-forward.
2. Сделайте четыре коммита `wip a`, `wip b`, `wip c`, `wip d`. Используйте `git rebase -i HEAD~4` и `fixup`, чтобы остался один коммит. Затем выполните reword.
3. Поставьте на последний коммит тег `v0.1.0` (аннотированный) и покажите тег через `git show v0.1.0`.
4. В собственном проекте на Rust (например, `hello-rust` из Урока 01.3) выполните `cargo fmt --check`, `cargo clippy --all-targets -- -D warnings` и `cargo test`. Намеренно испортите форматирование, прочитайте вывод первой команды, затем выполните `cargo fmt` и проверьте снова.
5. Напишите в этом проекте файл `.github/workflows/ci.yml` из раздела 12. Отправлять его на сервер вы будете в проекте из [Урока 01.10](10-project-your-learning-repo-and-checkpoint.md).

## Упражнения

### Упражнение 1 — Сообщения Conventional Commits (легко)

Напишите сообщение Conventional Commits для каждого изменения: (а) вы добавили опцию `--verbose` в инструмент; (б) вы исправили падение на пустом вводе; (в) вы обновили README; (г) вы переименовали публичную функцию, что ломает чужой код.

<details>
<summary>Подсказка</summary>

Типы: `feat`, `fix`, `docs`, `refactor`. У ломающего изменения есть `!`.

</details>

<details>
<summary>Ответ</summary>

```text
feat: add --verbose option
fix: avoid crash on empty input
docs(readme): update build instructions
refactor!: rename parse_line to parse

BREAKING CHANGE: parse_line was renamed to parse.
```

Почему это работает: каждое сообщение называет тип и краткое описание в повелительном наклонении. Знак `!` и подвал предупреждают, что пользователям придётся изменить свой код.

</details>

### Упражнение 2 — Напишите описание PR (легко)

Вы исправили задачу (#12), в которой программа паникует на пустом входном файле. Вы добавили тест. Напишите заголовок и описание PR по шаблону из раздела 3.

<details>
<summary>Подсказка</summary>

Используйте четыре заголовка What / Why / How / Testing и `Closes #12`.

</details>

<details>
<summary>Ответ</summary>

```text
Title: Fix panic on empty input file

## What
The program now prints a clear message and exits with code 1 when the input file is empty.

## Why
Closes #12. Before this change the program panicked on an empty file.

## How
Added an early check after reading the file.

## Testing
Added the test `empty_file_is_rejected`. `cargo test` passes.
```

Почему это работает: ревьюер видит цель, ссылку на задачу, подход и доказательства в одном месте, а GitHub закрывает задачу 12 при слиянии.

</details>

### Упражнение 3 — Сделайте rebase ветки (средне)

Создайте репозиторий с `main` (два коммита) и веткой `feature`, начатой от первого коммита, с одним коммитом, затрагивающим **другой** файл. Сделайте rebase `feature` на `main`, покажите линейную историю и слейте в `main` как fast-forward.

<details>
<summary>Подсказка</summary>

Создайте `feature` после первого коммита, затем вернитесь в `main` и добавьте ещё один коммит. Потом `git switch feature`, `git rebase main`, `git switch main`, `git merge feature`.

</details>

<details>
<summary>Ответ</summary>

```bash
git init -b main rebase-ex
cd rebase-ex
echo "one" > a.txt
git add a.txt
git commit -m "A"
git switch -c feature
echo "f" > f.txt
git add f.txt
git commit -m "F"
git switch main
echo "two" >> a.txt
git commit -am "B"
git switch feature
git rebase main
git log --oneline --graph --all
git switch main
git merge feature
```

Граф после rebase — одна прямая линия `F`, `B`, `A`; итоговое слияние говорит `Fast-forward`.

Почему это работает: rebase воспроизводит `F` поверх `B`, поэтому `feature` содержит всю `main`. Тогда `main` может просто сдвинуться вперёд.

</details>

### Упражнение 4 — Склейте три коммита через `fixup` (средне)

В ветке сделайте коммиты `wip 1`, `wip 2`, `wip 3`, каждый добавляет по строке в `notes.txt`. Объедините их в один коммит интерактивным rebase. Покажите лог до и после.

<details>
<summary>Подсказка</summary>

`git rebase -i HEAD~3`, затем замените второй и третий `pick` на `fixup`. Чтобы сделать это без редактора, задайте `GIT_SEQUENCE_EDITOR` командой, которая правит список.

</details>

<details>
<summary>Ответ</summary>

```bash
git switch -c wip
echo "line 1" >> notes.txt
git add notes.txt
git commit -m "wip 1"
echo "line 2" >> notes.txt
git commit -am "wip 2"
echo "line 3" >> notes.txt
git commit -am "wip 3"
git log --oneline -3
git rebase -i HEAD~3
git log --oneline -3
```

В редакторе измените список на `pick` / `fixup` / `fixup`, сохраните и закройте. До: три коммита `wip`. После: один коммит `wip 1`.

Без редактора (macOS): `GIT_SEQUENCE_EDITOR="sed -i '' -e '2,3s/^pick/fixup/'" git rebase -i HEAD~3`.

Почему это работает: `fixup` вливает коммит в предыдущий и отбрасывает его сообщение. Три изменения остаются в одном коммите.

</details>

### Упражнение 5 — Запустите CI на своём компьютере (сложно)

В проекте, созданном через `cargo new ci-exercise --lib`: замените `src/lib.rs` кодом ниже, запустите три команды CI, прочитайте ошибки, исправьте их и закоммитьте, когда пройдут все три. Также создайте файл `.github/workflows/ci.yml` и закоммитьте его отдельным коммитом с сообщением в стиле Conventional Commits.

```rust
pub fn add(a: i32,b: i32) -> i32 {
    return a+b;
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

<details>
<summary>Подсказка</summary>

Первой падает `cargo fmt --check` (пробелы). Выполните `cargo fmt`, чтобы исправить. Затем `cargo clippy` жалуется на `return`; уберите слово `return` и точку с запятой.

</details>

<details>
<summary>Ответ</summary>

```bash
cargo new ci-exercise --lib
cd ci-exercise
# put the code above into src/lib.rs
cargo fmt --check
cargo fmt
cargo clippy --all-targets -- -D warnings
```

`cargo fmt --check` печатает diff и падает; после `cargo fmt` файл отформатирован. Затем Clippy сообщает `unneeded return statement`. Исправьте `src/lib.rs`:

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

```bash
cargo fmt --check
cargo clippy --all-targets -- -D warnings
cargo test
git add .
git commit -m "feat: add the add function with a test"
mkdir -p .github/workflows
# put the workflow from section 12 into .github/workflows/ci.yml
git add .github
git commit -m "ci: check format, lints and tests"
```

`cargo test` заканчивается строкой `test result: ok. 1 passed`.

Почему это работает: три команды те же, что запустит CI, поэтому если они проходят локально, то пройдут и на GitHub. `cargo fmt` исправляет форматирование автоматически, но советы Clippy должны применить вы.

</details>

## Вопросы для самопроверки

1. Какова цель pull request?

<details>
<summary>Ответ</summary>

Предложить слить ветку в другую (обычно `main`) с местом для ревью, обсуждения и автоматических проверок, чтобы общая ветка оставалась рабочей.

</details>

2. Чем отличаются «squash and merge» и «rebase and merge»?

<details>
<summary>Ответ</summary>

Squash объединяет все коммиты PR в один новый коммит в `main`. Rebase-and-merge воспроизводит каждый коммит отдельно поверх `main`. Оба дают прямую историю.

</details>

3. В чём золотое правило rebase?

<details>
<summary>Ответ</summary>

Никогда не делайте rebase коммитов, которые уже есть у других людей (общая история). Делайте rebase только своей неопубликованной работы.

</details>

4. Почему `git push --force-with-lease` безопаснее `--force`?

<details>
<summary>Ответ</summary>

Он отказывается перезаписывать удалённую ветку, если она изменилась с тех пор, как вы видели её в последний раз, так что вы случайно не уничтожите новые коммиты коллеги. (Он не защитит вас, если вы выполнили `git fetch` и проигнорировали то, что пришло.)

</details>

5. Версия изменилась с `1.4.2` на `1.5.0`. Какого рода было изменение?

<details>
<summary>Ответ</summary>

Изменение MINOR: новая возможность с обратной совместимостью.

</details>

6. Каковы три стандартные проверки CI для проекта на Rust?

<details>
<summary>Ответ</summary>

`cargo fmt --check` (форматирование), `cargo clippy --all-targets -- -D warnings` (линты) и `cargo test` (тесты).

</details>

7. Почему хука `pre-commit` недостаточно, чтобы гарантировать качество в команде?

<details>
<summary>Ответ</summary>

Хуки живут в `.git/hooks/`, который не входит в репозиторий. Коллеги их не получают и могут пропустить через `--no-verify`. CI работает для всех и не может быть пропущен.

</details>

## Итоги

- Процесс с pull request: ветка → коммиты → push → PR → ревью и CI → слияние → уборка.
- Хорошие PR небольшие, с одной целью, понятным заголовком и полезным описанием; в ревью оценивайте код, а не человека.
- Стратегии слияния: merge commit, squash, rebase-and-merge. Команда выбирает одну.
- `git rebase` воспроизводит ваши коммиты на новой базе. Никогда не делайте rebase общей истории; для своей отправленной ветки используйте `--force-with-lease`.
- `git rebase -i` наводит порядок в коммитах (`reword`, `squash`, `fixup`, `drop`); `git cherry-pick` копирует один коммит.
- Conventional Commits (`feat:`, `fix:` …), SemVer (`MAJOR.MINOR.PATCH`) и аннотированные теги (`v1.2.0`) делают историю и релизы читаемыми.
- CI запускает `cargo fmt --check`, `cargo clippy -- -D warnings` и `cargo test` при каждом push; защитите `main`, чтобы она их требовала.
- Хуки помогают локально, но настоящий сторож — CI.

## Источники и что почитать

- [GitHub Docs — About pull requests](https://docs.github.com/en/pull-requests) — pull request, ревью и слияние на GitHub.
- [GitLab Docs — Merge requests](https://docs.gitlab.com/user/project/merge_requests/) — те же идеи на GitLab.
- [Pro Git — 3.6 Rebasing](https://git-scm.com/book/en/v2/Git-Branching-Rebasing) — официальное объяснение с «опасностями rebase».
- [Pro Git — 7.6 Rewriting History](https://git-scm.com/book/en/v2/Git-Tools-Rewriting-History) — amend, интерактивный rebase, squash, перестановка.
- [Pro Git — 2.6 Tagging](https://git-scm.com/book/en/v2/Git-Basics-Tagging) — облегчённые и аннотированные теги.
- [Conventional Commits 1.0.0](https://www.conventionalcommits.org/en/v1.0.0/) — полная спецификация.
- [Semantic Versioning 2.0.0](https://semver.org/) — правила для `MAJOR.MINOR.PATCH`.
- [Keep a Changelog](https://keepachangelog.com/en/1.1.0/) — формат для `CHANGELOG.md`.
- [Google Engineering Practices — Code Review](https://google.github.io/eng-practices/review/) — как ревьюить и как проходить ревью.
- [GitHub Docs — Understanding GitHub Actions](https://docs.github.com/en/actions) — workflow, job и step.
- [GitLab Docs — GitLab CI/CD](https://docs.gitlab.com/ci/) — пайплайны и `.gitlab-ci.yml`.
- [GitHub Docs — About protected branches](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-protected-branches/about-protected-branches) — правила защиты веток.
- [GitHub Docs — About commit signature verification](https://docs.github.com/en/authentication/managing-commit-signature-verification/about-commit-signature-verification) — подписанные коммиты.
- [The Cargo Book — SemVer compatibility](https://doc.rust-lang.org/cargo/reference/semver.html) — что считается ломающим изменением в Rust.

---

[🏠 Начало](../START-HERE.md) · [← 01.8 Удалённые репозитории: GitHub и GitLab](08-remotes-github-and-gitlab.md) · [📋 Этап 01](README.md) · [01.10 Проект: ваш учебный репозиторий и контрольная точка →](10-project-your-learning-repo-and-checkpoint.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
