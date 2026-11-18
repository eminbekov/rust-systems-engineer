<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 01.1 Что такое контроль версий](01-what-is-version-control.md) · [📋 Этап 01](README.md) · [01.3 Первый репозиторий и коммиты →](03-your-first-repository-and-commits.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/01-git-and-version-control/02-install-and-configure-git.md) <!-- nav -->

# Урок 01.2 — Установка и настройка Git

> **Этап 01 — Git и контроль версий** · Урок 2 из 10 · ⏱ около 120 минут

## Что вы узнаете

- Как проверить, установлен ли Git на вашем Mac, и как его установить или обновить.
- Где Git хранит настройки и три уровня конфигурации: системный, глобальный и локальный.
- Как сообщить Git, кто вы, и как задать небольшой набор безопасных настроек по умолчанию, который нужен каждому новичку.
- Как игнорировать файлы вроде `.DS_Store` во всех репозиториях с помощью одного глобального файла игнорирования.
- Как найти справку, когда вы забыли команду.

## Перед началом

- Вы прошли [Урок 01.1 — Что такое контроль версий](01-what-is-version-control.md).
- Вы умеете открыть терминал и запускать команды ([Урок 00.4](../00-getting-started/04-terminal-basics.md)).
- Вы установили Rust и RustRover ([Урок 00.5](../00-getting-started/05-install-rust-and-rustrover.md)).

## Новые слова

| Слово | Значение |
|---|---|
| **конфигурация** (configuration, config) | Набор настроек, которые меняют поведение программы. |
| **уровень конфигурации** (config level) | Где хранится настройка и на что она распространяется: *системный* (все пользователи), *глобальный* (вы), *локальный* (один репозиторий). |
| **`~`** (тильда) | Короткое имя вашей домашней папки, например `/Users/ada`. |
| **dotfile** («точечный файл») | Файл или папка, имя которых начинается с точки (`.gitconfig`). По умолчанию скрыты в Finder и в `ls`. |
| **Xcode Command Line Tools** | Бесплатный пакет от Apple с программами разработчика (инструменты компилятора, `git`, `make`). Rust на macOS тоже его требует. |
| **Homebrew** | Популярный *менеджер пакетов* для macOS: программа, которая устанавливает другие программы одной командой, `brew install …`. |
| **менеджер пакетов** (package manager) | Программа, которая скачивает, устанавливает и обновляет другие программы. |
| **PATH** | Список папок, в которых терминал ищет программы, когда вы вводите имя команды. |
| **редактор** (для Git) | Текстовый редактор, который Git открывает, когда нужно ввести сообщение. |
| **псевдоним** (alias) | Короткое собственное имя для длинной команды. |
| **пейджер** (pager) | Программа, которая показывает длинный вывод по одному экрану (Git использует `less`). Нажмите `q`, чтобы выйти. |
| **глобальный файл игнорирования** (global ignore file) | Файл с шаблонами файлов, которые Git должен игнорировать в *каждом* репозитории на вашем компьютере. |
| **GUI** (Graphical User Interface, графический интерфейс) | Способ работать с программой через окна и мышь. |

## 1. Git уже установлен?

На macOS Git часто устанавливается вместе с **Xcode Command Line Tools**, которые нужны и Rust. Проверьте:

```bash
git --version
which git
```

```text
git version 2.54.0 (Apple Git-157)
/usr/bin/git
```

**Что происходит:**

1. `git --version` просит Git напечатать свою версию. Ваши числа могут отличаться. Для этого курса подойдёт любая версия **2.40 или новее**. В этом уроке мы используем настройки, которым нужна версия не ниже 2.37.
2. `which git` печатает полный путь к программе, которая запускается, когда вы вводите `git`. `/usr/bin/git` — маленький «запускатель» от Apple, который стартует Git из инструментов Xcode.

Если терминал пишет `command not found` (или macOS открывает окно с предложением установить инструменты разработчика), установите Git одним из двух способов.

**Способ A — Xcode Command Line Tools (просто, от Apple):**

```bash
xcode-select --install
```

Откроется окно. Нажмите *Install* и подождите. Это установит и инструменты компилятора, которые Rust нужны для компоновки программ. Git от Apple обновляется вместе с обновлениями macOS, поэтому может отставать от новейшего на несколько версий.

**Способ B — Homebrew (более новый Git, легко обновлять):**

```bash
brew install git
```

Если Homebrew не установлен, не ставьте его сейчас. Способа A достаточно. Если вы уже пользуетесь Homebrew, он даст самый новый Git, а позже `brew upgrade git` его обновит.

После способа B на вашем Mac может быть два Git. Запускается тот, который стоит первым в `PATH`:

```bash
which -a git
```

```text
/usr/bin/git
```

(С Homebrew вы увидели бы две строки, например сначала `/opt/homebrew/bin/git`, потом `/usr/bin/git`. Запускается тот, что в первой строке.)

> 💡 **Совет:** RustRover находит Git сам. Проверить можно в *Settings → Version Control → Git*: поле «Path to Git executable» показывает, какой Git используется. IDE использует те же файлы конфигурации, что и терминал, поэтому всё, что вы настроите в этом уроке, работает и в IDE (Integrated Development Environment, интегрированной среде разработки — программе для написания кода).

## 2. Где Git хранит настройки

Git читает настройки из трёх мест. Более конкретное место побеждает более общее.

```text
 system   (all users of the computer)   e.g. /etc/gitconfig        weakest
    ↓ overridden by
 global   (you, in every repo)          ~/.gitconfig               ← most of your settings
    ↓ overridden by
 local    (one repository only)         <repo>/.git/config         strongest
```

Настройки меняются командой `git config`. Флаг указывает Git, какой уровень использовать:

| Команда | Уровень |
|---|---|
| `git config --system …` | системный (используется редко; нужны права администратора) |
| `git config --global …` | глобальный: файл `~/.gitconfig` |
| `git config …` (без флага, внутри репозитория) | локальный: файл `.git/config` этого репозитория |

Имена настроек состоят из двух частей: `раздел.ключ`, например `user.name` (раздел `user`, ключ `name`).

## 3. Скажите Git, кто вы

Каждый коммит хранит автора: имя и email. Git не угадывает их надёжно, поэтому задайте их один раз:

```bash
git config --global user.name "Ada Learner"
git config --global user.email "ada@example.com"
```

Используйте **свои** имя и email вместо примера. Кавычки вокруг имени оставьте: в нём есть пробел.

**Что происходит:**

1. `git config` меняет настройку. `--global` значит «для меня, во всех репозиториях».
2. `user.name` и `user.email` — ключи. Последнее слово — значение.
3. Git записывает значения в файл `~/.gitconfig`. Если всё получилось, ничего не печатается.

Прочитайте значения обратно:

```bash
git config user.name
git config user.email
```

```text
Ada Learner
ada@example.com
```

> 🔒 **Безопасность:** email из коммита становится **публичным**, когда вы отправляете коммит на GitHub или GitLab. Если вы не хотите видеть свой настоящий адрес в публичной истории, используйте приватный адрес «noreply», который выдаёт GitHub (он выглядит как `12345678+yourname@users.noreply.github.com`; его можно найти в аккаунте GitHub, *Settings → Emails*, после включения «Keep my email addresses private»). Аккаунт вы создадите в [Уроке 01.8](08-remotes-github-and-gitlab.md). Email можно изменить позже той же командой.

## 4. Небольшой набор безопасных настроек по умолчанию

Выполните эти команды один раз. Каждая из них объяснена ниже.

```bash
git config --global init.defaultBranch main
git config --global core.editor nano
git config --global pull.ff only
git config --global push.autoSetupRemote true
git config --global merge.conflictStyle zdiff3
git config --global fetch.prune true
```

| Настройка | Что делает | Зачем она нужна |
|---|---|---|
| `init.defaultBranch main` | Первая ветка каждого нового репозитория называется `main`. | Прежнее имя по умолчанию в Git — `master`. В Git 3.0 по умолчанию станет `main`; GitHub и GitLab уже используют `main`. Без этой настройки Git каждый раз при создании репозитория печатает длинную подсказку. |
| `core.editor nano` | Текстовый редактор, который Git открывает для сообщений коммитов. | Без неё Git может открыть `vim`, который сильно путает новичков (см. ошибку ниже). `nano` показывает клавиши внизу экрана. |
| `pull.ff only` | `git pull` может только «перемотать вперёд» (fast-forward, просто добавить новые коммиты других). Если изменились и ваша работа, и чужая, команда останавливается. | Она никогда не создаёт неожиданное слияние. Что такое «fast-forward», вы узнаете в [Уроке 01.6](06-branches-and-merging.md), а эту настройку увидите в действии в [Уроке 01.8](08-remotes-github-and-gitlab.md). |
| `push.autoSetupRemote true` | Первый `git push` новой ветки связывает её с веткой с тем же именем в удалённом репозитории. | Не нужно каждый раз вводить `--set-upstream`. |
| `merge.conflictStyle zdiff3` | При конфликте слияния Git показывает ещё и *исходный* текст между двумя конфликтующими версиями. | С этой третьей порцией информации конфликты решать гораздо проще. См. [Урок 01.7](07-merge-conflicts.md). Нужен Git 2.35 или новее. |
| `fetch.prune true` | При загрузке обновлений Git удаляет вашу локальную пометку об удалённых ветках, которые были удалены. | Список веток остаётся чистым. |

> 📏 **Соглашение:** это *безопасные* настройки: они не меняют ваши файлы, а лишь делают поведение Git предсказуемее. Не копируйте длинные «идеальные .gitconfig» из интернета, пока не понимаете каждую строку.

### Если вы предпочитаете другой редактор

`core.editor` — любая команда, которая открывает файл и ждёт, пока вы его закроете. Например, при установленном Visual Studio Code: `git config --global core.editor "code --wait"`. Часть `--wait` важна: она заставляет Git ждать, пока вы закроете вкладку с файлом. Пока достаточно `nano`.

## 5. Посмотрите на результат

```bash
git config --global --list
```

```text
user.name=Ada Learner
user.email=ada@example.com
init.defaultbranch=main
core.editor=nano
pull.ff=only
push.autosetupremote=true
merge.conflictstyle=zdiff3
fetch.prune=true
```

(Имена ключей показаны строчными буквами. Это нормально: Git не различает регистр в названиях разделов и ключей.)

Чтобы увидеть, **какой файл** задаёт каждую настройку, добавьте `--show-origin`:

```bash
git config --global --list --show-origin
```

```text
file:/Users/ada/.gitconfig	user.name=Ada Learner
file:/Users/ada/.gitconfig	user.email=ada@example.com
file:/Users/ada/.gitconfig	init.defaultbranch=main
…
```

Файл — простой текст. Откройте его командой `cat ~/.gitconfig` (или отредактируйте через `git config --global --edit`):

```ini
[user]
	name = Ada Learner
	email = ada@example.com
[init]
	defaultBranch = main
[core]
	editor = nano
[pull]
	ff = only
[push]
	autoSetupRemote = true
[merge]
	conflictStyle = zdiff3
[fetch]
	prune = true
```

**Что происходит:** в файле есть *разделы* в квадратных скобках и строки `ключ = значение`. Этот формат называется INI (от «initialization», очень старый простой формат файлов настроек). `git config` просто редактирует файл за вас. На Mac с Git от Apple может существовать и *системный* файл (Apple кладёт туда небольшую настройку для связки ключей macOS). Его можно не трогать.

## 6. Локальные настройки перекрывают глобальные

Иногда проекту нужна другая личность, например рабочему репозиторию — рабочая почта. Внутри такого репозитория выполните ту же команду **без** `--global`:

```bash
git init -b main demo-local
cd demo-local
git config user.email "work@example.org"
git config --show-scope --get-all user.email
```

```text
global	ada@example.com
local	work@example.org
```

**Что происходит:**

1. `git init -b main demo-local` создаёт новый пустой репозиторий (команду `git init` вы изучите в следующем уроке).
2. Без `--global` команда `git config` записывает значение в `demo-local/.git/config`, поэтому оно действует только здесь.
3. `--show-scope` печатает, откуда взято каждое значение. У `user.email` два значения. Git использует **последнее и самое конкретное**: локальное.

Проверьте, какое из них действует:

```bash
git config user.email
```

```text
work@example.org
```

## 7. Глобальный файл игнорирования

Некоторые файлы появляются в каждом проекте, и их никогда не нужно коммитить: служебный файл Finder в macOS `.DS_Store`, папка RustRover `.idea/`, временные файлы редакторов. Сказать об этом Git можно один раз для всех репозиториев на вашем компьютере:

```bash
printf '.DS_Store\n.idea/\n*.swp\n' > ~/.gitignore_global
git config --global core.excludesFile '~/.gitignore_global'
cat ~/.gitignore_global
```

```text
.DS_Store
.idea/
*.swp
```

**Что происходит:**

1. `printf '...\n' > file` записывает три строки в файл `~/.gitignore_global`. (`\n` означает «новая строка».)
2. `core.excludesFile` говорит Git: «читай шаблоны игнорирования и из этого файла в каждом репозитории».
3. `cat` показывает файл.

Каждая строка — это **шаблон**: `.DS_Store` совпадает с файлом с таким именем, `.idea/` — с папкой, а `*.swp` — с любым файлом, имя которого заканчивается на `.swp` (`*` означает «что угодно»). Файлы игнорирования вы примените в проекте в [Уроке 01.3](03-your-first-repository-and-commits.md).

> 📏 **Соглашение:** то, что относится к **вашему** компьютеру и инструментам (macOS, ваш редактор), кладите в *глобальный* файл игнорирования. То, что относится к **проекту** (например, папка `target/` в Rust), кладите в собственный `.gitignore` проекта, чтобы он достался каждому участнику команды.

## 8. Псевдонимы и справка

**Псевдоним** (alias) — это сокращение. Например:

```bash
git config --global alias.st "status --short --branch"
git st
```

Это удобно, но с псевдонимами лучше подождать, пока вы не выучите полные команды; иначе вы растеряетесь на любом другом компьютере. Удалить псевдоним можно командой `git config --global --unset alias.st`.

Когда вы что-то забыли, у Git есть встроенная справка:

| Команда | Что вы получите |
|---|---|
| `git help commit` | Полную страницу руководства по `commit`. Двигайтесь стрелками или пробелом, выходите по `q`. |
| `git commit -h` | Только краткий список опций. Удобно и быстро. |
| `git help -a` | Все команды. |
| `git help -g` | Список справочных руководств по понятиям, например `tutorial`, `everyday`, `glossary`. Попробуйте `git help everyday`. |
| `git help config` | Все настройки (очень длинно). |

## Частые ошибки

> ⚠️ **Частая ошибка:** забыть кавычки вокруг имени с пробелом.

```bash
git config --global user.name Ada Learner
git config --global user.name
```

```text
Ada
```

Git не пожаловался, но сохранил только `Ada`. Оболочка разделила текст на два слова, а Git взял первое. **Исправление:** поставьте кавычки вокруг значения: `git config --global user.name "Ada Learner"`. Всегда читайте настройку обратно, чтобы убедиться.

> ⚠️ **Частая ошибка:** опечатка в имени ключа — **не** ошибка.

```bash
git config --global user.mial "oops@example.com"
git config --global --get user.mial
```

```text
oops@example.com
```

Git охотно сохраняет любой ключ. Настоящий `user.email` не изменился, поэтому ваши коммиты будут использовать старый (или неверный) адрес. **Исправление:** после настройки проверяйте командой `git config --global --list` и удаляйте неверный ключ: `git config --global --unset user.mial`.

> ⚠️ **Частая ошибка:** «застрявший» редактор. Если Git открыл `vim` и ничего из того, что вы вводите, не работает, сделайте так: нажмите `Esc`, введите `:q!` и нажмите `Enter`. Это выход без сохранения. (Чтобы сохранить и выйти из `vim`: `Esc`, введите `:wq`, нажмите `Enter`.) Если выставить `core.editor` в `nano`, проблемы не будет.

> ⚠️ **Частая ошибка:** задать личность только после первых коммитов или использовать `--global`, когда нужен один репозиторий. У старых коммитов остаётся старый автор. Проверяйте настройки **до** первого коммита.

> 🔒 **Безопасность:** никогда не кладите пароли, токены или ключи в `~/.gitconfig`. Не вставляйте чужую конфигурацию, не прочитав её.

## Попробуйте сами

1. Проверьте Git: выполните `git --version` и `which git`. Запишите версию в заметки.
2. Задайте имя и email (раздел 3) со **своими** значениями. Прочитайте их обратно.
3. Выполните шесть настроек по умолчанию из раздела 4.
4. Выполните `git config --global --list` и сравните со списком в разделе 5.
5. Создайте глобальный файл игнорирования (раздел 7).
6. Создайте тестовую папку и репозиторий и проверьте, что глобальное игнорирование работает:

```bash
mkdir -p ~/projects/git-playground/ignore-test
cd ~/projects/git-playground/ignore-test
git init
touch .DS_Store notes.txt
git status --short
git check-ignore -v .DS_Store
```

```text
?? notes.txt
/Users/ada/.gitignore_global:1:.DS_Store	.DS_Store
```

**Что происходит:**

1. `mkdir -p …` создаёт папку для практики `~/projects/git-playground` (специально вне папки курса) и подпапку.
2. `git init` превращает папку в репозиторий.
3. `touch` создаёт два пустых файла.
4. `git status --short` перечисляет файлы, которые видит Git. Появляется только `notes.txt` (`??` значит «новый, ещё не отслеживается»). `.DS_Store` в списке нет, потому что он игнорируется.
5. `git check-ignore -v` объясняет, *почему* файл игнорируется: показывает файл, номер строки и совпавший шаблон.

Сохраните папку `~/projects/git-playground`: она понадобится в следующих уроках.

## Упражнения

### Упражнение 1 — Где Git и какая версия? (легко)

Напишите одну команду, которая показывает версию Git, и одну, которая показывает, где лежит программа Git. Затем выясните, поддерживает ли эта версия настройку `merge.conflictStyle zdiff3` (нужна версия 2.35 или новее).

<details>
<summary>Подсказка</summary>

Первая — `git --version`. Вторая — команда из раздела 1, которая «находит» программы.

</details>

<details>
<summary>Ответ</summary>

```bash
git --version
which git
```

Если номер версии 2.35 или выше (например, 2.54), `zdiff3` поддерживается.

Почему это работает: `git --version` печатает версию, а `which` показывает первую программу с таким именем, найденную в `PATH`.

</details>

### Упражнение 2 — Проверка личности (легко)

Задайте имя и email, затем напишите одну команду, которая печатает **оба** значения вместе с файлом, откуда они взяты.

<details>
<summary>Подсказка</summary>

`git config --global --list --show-origin` печатает всё. Отфильтровать можно через `--get-regexp '^user\.'`.

</details>

<details>
<summary>Ответ</summary>

```bash
git config --global user.name "Ada Learner"
git config --global user.email "ada@example.com"
git config --global --show-origin --get-regexp '^user\.'
```

```text
file:/Users/ada/.gitconfig	user.name Ada Learner
file:/Users/ada/.gitconfig	user.email ada@example.com
```

Почему это работает: `--get-regexp` печатает все ключи, подходящие под шаблон (здесь — ключи, которые начинаются с `user.`), а `--show-origin` добавляет имя файла.

</details>

### Упражнение 3 — Другой email для одного репозитория (средне)

Создайте репозиторий `~/projects/git-playground/work-demo`. Задайте email `work@example.org` только для этого репозитория. Докажите, что глобальный email **не** изменился.

<details>
<summary>Подсказка</summary>

Выполните `git config user.email …` без `--global`, внутри репозитория. Сравните `git config --local user.email` и `git config --global user.email`.

</details>

<details>
<summary>Ответ</summary>

```bash
mkdir -p ~/projects/git-playground/work-demo
cd ~/projects/git-playground/work-demo
git init
git config user.email "work@example.org"
git config --local user.email
git config --global user.email
```

```text
work@example.org
ada@example.com
```

Почему это работает: без `--global` значение попадает в `.git/config` этого репозитория. Глобальный файл `~/.gitconfig` не тронут, а локальное значение побеждает только внутри этого репозитория.

</details>

### Упражнение 4 — Игнорируйте файлы везде (средне)

Добавьте шаблон `*.log` в глобальный файл игнорирования. Затем докажите, что файл `debug.log` в новом репозитории игнорируется.

<details>
<summary>Подсказка</summary>

Допишите строку через `>>`. Затем создайте репозиторий и используйте `git check-ignore -v`.

</details>

<details>
<summary>Ответ</summary>

```bash
echo '*.log' >> ~/.gitignore_global
mkdir -p ~/projects/git-playground/ignore-test2
cd ~/projects/git-playground/ignore-test2
git init
touch debug.log
git check-ignore -v debug.log
```

```text
/Users/ada/.gitignore_global:4:*.log	debug.log
```

Почему это работает: `>>` дописывает строку в конец файла (здесь это строка 4). Git читает файл в каждом репозитории, а `check-ignore -v` показывает строку, которая совпала.

</details>

### Упражнение 5 — Прочитайте руководство (сложно)

Пользуясь только встроенной справкой Git, выясните, что делает опция `--amend` команды `git commit` (вы воспользуетесь ею в [Уроке 01.5](05-undoing-changes-safely.md)) и что делает `git config --unset`. Интернетом не пользуйтесь.

<details>
<summary>Подсказка</summary>

`git commit -h` показывает только краткий список. `git help commit` открывает руководство: нажмите `/`, чтобы искать внутри, введите `amend`, нажмите `Enter`, а `q` — чтобы выйти.

</details>

<details>
<summary>Ответ</summary>

```bash
git help commit      # search for "--amend" with /--amend
git help config      # search for "--unset"
```

`--amend` заменяет вершину текущей ветки новым коммитом (позволяет исправить сообщение или содержимое последнего коммита). `--unset` удаляет ключ из файла конфигурации.

Почему это работает: страницы руководства устанавливаются вместе с Git, поэтому работают без сети. `less` (пейджер) позволяет искать по `/`.

</details>

## Вопросы для самопроверки

1. В каком файле хранятся ваши глобальные настройки Git?

<details>
<summary>Ответ</summary>

В `~/.gitconfig` (файл `.gitconfig` в вашей домашней папке).

</details>

2. Вы задали `user.email` и глобально, и локально в одном репозитории. Какое значение Git использует в этом репозитории?

<details>
<summary>Ответ</summary>

Локальное. Более конкретный уровень побеждает.

</details>

3. Вы ввели `git config --global user.name Ada Learner`, и Git ничего не напечатал. Что сохранилось?

<details>
<summary>Ответ</summary>

Только `Ada`. Имени нужны кавычки, потому что в нём есть пробел.

</details>

4. Зачем мы задаём `init.defaultBranch main`?

<details>
<summary>Ответ</summary>

Чтобы каждый новый репозиторий начинался с ветки `main` (это имя используют GitHub, GitLab и будущие версии Git) и чтобы Git не печатал длинную подсказку про `master`.

</details>

5. Чем отличается `~/.gitignore_global` от `.gitignore` проекта?

<details>
<summary>Ответ</summary>

Глобальный файл есть только на вашем компьютере и действует во всех репозиториях (для ваших инструментов, вроде `.DS_Store`). `.gitignore` проекта коммитится в репозиторий и действует для каждого, кто над ним работает.

</details>

6. Как выйти из `vim` без сохранения?

<details>
<summary>Ответ</summary>

Нажмите `Esc`, введите `:q!` и нажмите `Enter`.

</details>

## Итоги

- Проверьте Git командой `git --version`; установите его через `xcode-select --install` или `brew install git`.
- Настройки живут на трёх уровнях: системном, глобальном (`~/.gitconfig`) и локальном (`.git/config`). Побеждает самый конкретный уровень.
- Задайте `user.name` и `user.email` один раз, с кавычками, и прочитайте их обратно.
- Несколько безопасных настроек по умолчанию: `init.defaultBranch main`, простой редактор, `pull.ff only`, `push.autoSetupRemote true`, `merge.conflictStyle zdiff3`, `fetch.prune true`.
- Глобальный файл игнорирования не пускает `.DS_Store` и `.idea/` ни в один репозиторий.
- `git config --list --show-origin` показывает каждую настройку и её файл; `git help <команда>` и `git <команда> -h` — ваша справка без сети.
- Опечатки в именах настроек не считаются ошибками. Всегда читайте результат обратно.

## Источники и что почитать

- [Pro Git — 1.6 First-Time Git Setup](https://git-scm.com/book/en/v2/Getting-Started-First-Time-Git-Setup) — официальная первая настройка.
- [Pro Git — 1.5 Installing Git](https://git-scm.com/book/en/v2/Getting-Started-Installing-Git) — варианты установки для каждой системы.
- [git-config manual](https://git-scm.com/docs/git-config) — все настройки с точным значением.
- [gitignore manual](https://git-scm.com/docs/gitignore) — точные правила шаблонов игнорирования.
- [GitHub Docs — Setting your commit email address](https://docs.github.com/en/account-and-profile/setting-up-and-managing-your-personal-account-on-github/managing-email-preferences/setting-your-commit-email-address) — как использовать приватный адрес «noreply».

---

[🏠 Начало](../START-HERE.md) · [← 01.1 Что такое контроль версий](01-what-is-version-control.md) · [📋 Этап 01](README.md) · [01.3 Первый репозиторий и коммиты →](03-your-first-repository-and-commits.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
