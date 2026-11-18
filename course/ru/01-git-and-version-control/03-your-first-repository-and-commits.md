<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 01.2 Установка и настройка Git](02-install-and-configure-git.md) · [📋 Этап 01](README.md) · [01.4 История и просмотр изменений →](04-viewing-history-and-changes.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/01-git-and-version-control/03-your-first-repository-and-commits.md) <!-- nav -->

# Урок 01.3 — Первый репозиторий и коммиты

> **Этап 01 — Git и контроль версий** · Урок 3 из 10 · ⏱ около 3 часов (2 занятия)

## Что вы узнаете

- Как создать репозиторий командой `git init` и что такое скрытая папка `.git`.
- Жизнь файла: неотслеживаемый, изменённый, подготовленный, закоммиченный.
- Как читать `git status` и как пользоваться `git add` и `git commit`.
- Как писать хорошее сообщение коммита.
- Как не пускать файлы в историю с помощью `.gitignore` и как это работает с Rust-проектом, созданным через `cargo new`.

## Перед началом

- Вы прошли [Урок 01.2 — Установка и настройка Git](02-install-and-configure-git.md). Команда `git config --global --list` показывает ваше имя и email.
- Папка `~/projects/git-playground` существует (вы создали её в предыдущем уроке). Если нет, выполните `mkdir -p ~/projects/git-playground`.
- Rust и Cargo работают: `cargo --version` печатает версию ([Урок 00.5](../00-getting-started/05-install-rust-and-rustrover.md)).

## Новые слова

| Слово | Значение |
|---|---|
| **`git init`** | Команда, которая превращает папку в репозиторий. |
| **папка `.git`** | Скрытая папка внутри репозитория, где Git хранит всю историю. Если её удалить, история пропадёт. |
| **рабочее дерево** (working tree) | Файлы и папки, которые вы видите и редактируете. |
| **отслеживаемый файл** (tracked file) | Файл, который Git уже знает (он есть в последнем коммите или в области подготовки). |
| **неотслеживаемый файл** (untracked file) | Новый файл, за которым Git пока не велено следить. |
| **область подготовки** (staging area, также *индекс*) | Место, где вы собираете изменения, которые попадут в следующий коммит. |
| **подготовить** (stage, глагол) | Поместить изменение в область подготовки командой `git add`. |
| **`HEAD`** | Указатель, означающий «коммит, на котором я сейчас стою», обычно последний коммит текущей ветки. |
| **сообщение коммита** (commit message) | Короткий текст, объясняющий, зачем сделан коммит. |
| **автор** (author) | Имя и email, сохранённые в коммите. |
| **`.gitignore`** | Текстовый файл с шаблонами файлов, которые Git не должен отслеживать. |
| **шаблон** (pattern) | Правило вроде `*.tmp`, подходящее ко многим именам файлов. `*` означает «что угодно». |
| **Cargo.lock** | Файл, который создаёт Cargo; в нём записаны точные версии библиотек, использованных проектом. |
| **TOML** (Tom's Obvious Minimal Language) | Простой текстовый формат для настроек. На нём написан `Cargo.toml`. |
| **SHA-1** (Secure Hash Algorithm 1) | Хеш-функция, которой Git называет коммиты. |

## 1. Создайте репозиторий

**Репозиторий** — это обычная папка плюс скрытая папка `.git`. Создадим один.

```bash
cd ~/projects/git-playground
mkdir hello-git
cd hello-git
git init
```

```text
Initialized empty Git repository in /Users/ada/projects/git-playground/hello-git/.git/
```

**Что происходит:**

1. Вы переходите в папку для практики, создаёте `hello-git` и заходите в неё.
2. `git init` создаёт внутри скрытую папку `.git`. С этого момента эта папка — репозиторий.
3. Git печатает, где находится репозиторий. В вашем пути будет ваше имя.

Спросите Git, что он видит:

```bash
git status
```

```text
On branch main

No commits yet

nothing to commit (create/copy files and use "git add" to track)
```

**Что происходит:** `git status` — команда, которую вы будете запускать чаще всего. Она сообщает три вещи: на какой вы ветке (`main`), что коммитов ещё нет и что сохранять нечего.

> ⚠️ **Частая ошибка:** не создавайте репозиторий внутри другого репозитория. Если выполнить `git init` в папке, которая уже находится внутри репозитория, получится «вложенный» репозиторий, и внешний запутается. Сначала выполните `git status`: если команда пишет `fatal: not a git repository`, можно спокойно делать `git init`.

### Что внутри `.git`?

```bash
ls -a
ls .git
```

```text
.
..
.git
```

```text
HEAD
config
description
hooks
info
objects
refs
```

(После первого коммита появится ещё несколько элементов. Они не важны.) Вы никогда не редактируете эти файлы вручную. Запомните два факта: **вся история живёт в `.git`** и **удаление `.git` удаляет историю** (обычные файлы остаются).

## 2. Жизнь файла

Файл в репозитории всегда находится в одном из четырёх состояний. Выучите эту картинку; на ней держится весь Git.

```text
 untracked ──git add──▶ staged ──git commit──▶ committed (clean)
                           ▲                        │
                           │                        │ you edit the file
                           └───git add──── modified ◀┘
```

| Состояние | Значение | Как выглядит в `git status` |
|---|---|---|
| **неотслеживаемый** (untracked) | Новый файл. Git за ним не следит. | указан в «Untracked files» |
| **подготовленный** (staged) | Его текущее содержимое выбрано для следующего коммита. | указан в «Changes to be committed» |
| **закоммиченный** (committed) | Его текущее содержимое сохранено в истории. Делать ничего не нужно. | не указан (дерево «чистое») |
| **изменённый** (modified) | Он отслеживается, и вы изменили его после последнего коммита. | указан в «Changes not staged for commit» |

Зачем в Git лишний шаг `git add`? Потому что вы сами решаете, **что попадёт в каждый коммит**. Допустим, за час вы исправили ошибку и начали работу над новой возможностью. С областью подготовки можно положить исправление в один коммит, а новую возможность — в другой. Маленькие целенаправленные коммиты гораздо легче читать и отменять.

## 3. Ваш первый коммит

Создайте файл. Используйте RustRover (*File → New → File*) или создайте его из терминала:

```bash
echo "# hello-git" > README.md
echo "" >> README.md
echo "My first repository." >> README.md
git status
```

```text
On branch main

No commits yet

Untracked files:
  (use "git add <file>..." to include in what will be committed)
	README.md

nothing added to commit but untracked files present (use "git add" to track)
```

**Что происходит:**

1. Три команды `echo` создают `README.md` из трёх строк: заголовок, пустая строка и предложение. (`>` создаёт или заменяет файл, `>>` добавляет в конец.)
2. `git status` показывает `README.md` как **неотслеживаемый**. Git его видит, но не следит за ним. Он даже подсказывает следующую команду.

Подготовьте файл:

```bash
git add README.md
git status
```

```text
On branch main

No commits yet

Changes to be committed:
  (use "git rm --cached <file>..." to unstage)
	new file:   README.md

```

**Что происходит:** `git add README.md` копирует текущее содержимое файла в область подготовки. Теперь файл **подготовлен** (staged). Статус говорит «Changes to be committed» (изменения, которые будут закоммичены): это в точности список того, что войдёт в следующий коммит.

Сделайте коммит:

```bash
git commit -m "Add README"
```

```text
[main (root-commit) 4fb5aaf] Add README
 1 file changed, 3 insertions(+)
 create mode 100644 README.md
```

**Что происходит:**

1. `git commit` берёт всё, что есть в области подготовки, и сохраняет как новый коммит.
2. `-m "Add README"` передаёт сообщение прямо в командной строке. Без `-m` Git откроет ваш редактор (`nano`), чтобы вы ввели более длинное сообщение: введите его, нажмите `Ctrl+O`, затем `Enter` для сохранения и `Ctrl+X` для выхода.
3. Вывод означает: ветка `main`; `root-commit` — «это самый первый коммит, у него нет родителя»; `4fb5aaf` — короткий хеш (первые 7 символов длинного); затем сообщение; затем сводка (1 файл, 3 добавленные строки). `100644` — режим файла: обычный, не исполняемый файл.

Ваш хеш будет другим. Это нормально. Он зависит от вашего имени, времени и содержимого.

```bash
git status
git log
```

```text
On branch main
nothing to commit, working tree clean
```

```text
commit 4fb5aafe1d1ebe8ae283f983329fbd147d8307bb
Author: Ada Learner <ada@example.com>
Date:   Tue Nov 17 16:22:18 2026 +0500

    Add README
```

**Что происходит:** `working tree clean` («рабочее дерево чистое») значит, что каждый отслеживаемый файл в точности такой же, как в последнем коммите. `git log` показывает историю: полный 40-символьный хеш (SHA-1), автора, дату и сообщение. Если вывод не помещается на экран, Git показывает его в пейджере: нажмите `q`, чтобы выйти.

## 4. Ритм «изменить → подготовить → закоммитить»

Это вы будете делать тысячи раз. Измените файл:

```bash
echo "It tracks my first steps with Git." >> README.md
git status
```

```text
On branch main
Changes not staged for commit:
  (use "git add <file>..." to update what will be committed)
  (use "git restore <file>..." to discard changes in working directory)
	modified:   README.md

no changes added to commit (use "git add" and/or "git commit -a")
```

Посмотрите, что именно изменилось, с помощью `git diff` (подробно эту команду вы изучите в следующем уроке):

```bash
git diff
```

```text
diff --git a/README.md b/README.md
index 6aafd58..c4d9981 100644
--- a/README.md
+++ b/README.md
@@ -1,3 +1,4 @@
 # hello-git
 
 My first repository.
+It tracks my first steps with Git.
```

Теперь подготовьте и закоммитьте:

```bash
git add README.md
git commit -m "Describe the project in README"
git log --oneline
```

```text
[main ee34cdb] Describe the project in README
 1 file changed, 1 insertion(+)
ee34cdb Describe the project in README
4fb5aaf Add README
```

**Что происходит:** `git log --oneline` печатает по одной строке на коммит: короткий хеш и сообщение, сначала самые новые. В вашей истории теперь два коммита.

### Краткий статус

`git status --short` (или `-s`) — компактная версия. Добавьте два файла и посмотрите:

```bash
echo 'fn main() {}' > main.rs
echo 'notes' > notes.txt
git status -s
git add main.rs
git status -s
git commit -m "Add empty main.rs"
git status -s
```

```text
?? main.rs
?? notes.txt
A  main.rs
?? notes.txt
[main fa488a1] Add empty main.rs
 1 file changed, 1 insertion(+)
 create mode 100644 main.rs
?? notes.txt
```

**Что происходит:** каждая строка начинается с двух букв. **Первая** буква — состояние в области подготовки, **вторая** — состояние в рабочем дереве.

| Код | Значение |
|---|---|
| `??` | неотслеживаемый |
| `A ` | добавлен (подготовлен как новый файл) |
| `M ` | изменён и подготовлен |
| ` M` | изменён, **не** подготовлен |
| `D ` / ` D` | удалён (подготовлено / не подготовлено) |
| `R ` | переименован (подготовлено) |

Вы закоммитили `main.rs`, но не `notes.txt`. Это **выборочная подготовка** в действии: `git commit` сохраняет только то, что подготовлено.

## 5. Что такое коммит на самом деле?

Коммит — это небольшой объект из четырёх частей. Можно посмотреть на последний:

```bash
git cat-file -p HEAD
```

```text
tree d819d0582756f690ade00d55cf1786e4d3879d2d
parent ee34cdb1ae84df7eccb449400b58d350947f965e
author Ada Learner <ada@example.com> 1794914538 +0500
committer Ada Learner <ada@example.com> 1794914538 +0500

Add empty main.rs
```

**Что происходит:** `git cat-file -p` печатает объект Git в читаемом виде. `HEAD` означает «текущий коммит». У объекта есть:

- **tree** — хеш снимка: список всех файлов и их содержимого в тот момент;
- **parent** — хеш коммита перед этим (у первого коммита родителя нет);
- **author / committer** — кто написал и когда (число — время в секундах с 1970 года, затем часовой пояс);
- **сообщение**.

Поскольку хеш родителя входит в коммит, а собственный хеш коммита вычисляется из всего перечисленного, изменение старого коммита меняет все последующие хеши. Поэтому историю нельзя незаметно изменить.

```text
 ee34cdb  ◀──────  fa488a1  ◀── HEAD
 (parent)           (this commit)
```

## 6. Хорошие сообщения коммитов

Сообщение пишется для вашего будущего «я» и для коллег. Сравните:

| Плохо | Хорошо |
|---|---|
| `fix` | `Fix crash when the input file is empty` |
| `changes` | `Add README with build instructions` |
| `asdf` | `Rename Config to Settings` |
| `final version 2` | `Print the total at the end of the report` |

📏 **Соглашение:** правила, которым следует большинство команд:

1. Первая строка (**тема**, subject) — до примерно 50 символов, без точки в конце.
2. Пишите её в **повелительном наклонении**, как приказ: «Add», «Fix», «Remove» — а не «Added» или «Fixing». Хороший тест: фраза *«Если применить, этот коммит … <тема>»* должна иметь смысл.
3. Если нужно больше слов, оставьте **пустую строку** после темы и напишите тело: объясните, **зачем** сделано изменение, а не что именно сделано (что — показывает diff). Переносите строки примерно на 72 символах.
4. Один коммит = одно логическое изменение. Не смешивайте исправление ошибки и изменение форматирования.

Для более длинного сообщения выполните `git commit` без `-m`. Откроется редактор с пустой первой строкой; введите тему, пустую строку, затем тело. Другие соглашения о сообщениях вы увидите в [Уроке 01.9](09-teamwork-pull-requests-and-conventions.md).

> 💡 **Совет:** коммитьте часто. Хороший ритм — один коммит каждый раз, когда заработал небольшой кусок. Коммиты дёшевы, и это ваши точки сохранения.

## 7. `.gitignore`: файлы, которые Git не должен отслеживать

Некоторые файлы никогда не должны попадать в историю: результат сборки, временные файлы, секреты. Их шаблоны перечисляются в файле `.gitignore` в корне репозитория.

```bash
printf '*.tmp\nbuild/\n' > .gitignore
mkdir build
touch build/out.bin cache.tmp
git status -s
```

```text
?? .gitignore
?? notes.txt
```

**Что происходит:** два файла, `build/out.bin` и `cache.tmp`, существуют, но Git их не показывает. Они подходят под шаблоны, поэтому Git ведёт себя так, будто их нет. Сам новый `.gitignore` — неотслеживаемый: его **нужно** закоммитить, чтобы у всех были одинаковые правила.

```bash
git add .gitignore
git commit -m "Ignore temporary files and build output"
```

Правила шаблонов, которые нужны сейчас:

| Шаблон | Совпадает с |
|---|---|
| `*.tmp` | любым файлом, имя которого заканчивается на `.tmp`, в любой папке |
| `build/` | папкой с именем `build` (косая черта в конце означает «папка»), где угодно |
| `/target` | папкой `target` **только** на верхнем уровне репозитория (косая черта в начале привязывает шаблон) |
| `!keep.tmp` | исключением: **не** игнорировать этот файл, даже если подходит другой шаблон |
| `# text` | комментарием |

> ⚠️ **Частая ошибка:** `.gitignore` действует только на файлы, которые **ещё не отслеживаются**. Если вы уже закоммитили `old.tmp`, то добавление `*.tmp` позже ему не поможет. Чтобы перестать отслеживать файл, но оставить его на диске, выполните `git rm --cached old.tmp` и закоммитьте:

```bash
git rm --cached old.tmp
git commit -m "Stop tracking old.tmp"
```

```text
rm 'old.tmp'
[main 4e0cbfc] Stop tracking old.tmp
 1 file changed, 1 deletion(-)
 delete mode 100644 old.tmp
```

(Файл остаётся в вашей папке; Git лишь забывает о нём. Старое содержимое остаётся в прежних коммитах.)

> 🔒 **Безопасность:** добавляйте файлы с секретами в `.gitignore` **до** первого коммита: `.env`, `*.pem`, `*.key`, `secrets.toml`. Секрет, который хоть раз закоммитили, остаётся в истории даже после удаления файла, и любой, кто может читать репозиторий, найдёт его. Если это случилось, считайте секрет украденным: создайте новый и отмените старый. Безопасный способ вы отработаете в [Уроке 01.5](05-undoing-changes-safely.md) и [Уроке 01.8](08-remotes-github-and-gitlab.md).

## 8. Git и проект на Rust

Теперь поместим настоящий проект на Rust под Git. Cargo помогает: когда вы создаёте проект **вне** какого-либо репозитория, `cargo new` сам выполняет `git init`.

```bash
cd ~/projects/git-playground
cargo new hello-rust
cd hello-rust
ls -a
cat .gitignore
```

```text
    Creating binary (application) `hello-rust` package
note: see more `Cargo.toml` keys and their definitions at https://doc.rust-lang.org/cargo/reference/manifest.html
.
..
.git
.gitignore
Cargo.toml
src
```

```text
/target
```

**Что происходит:**

1. `cargo new hello-rust` создаёт проект с `Cargo.toml` (файл настроек в формате TOML), папкой `src` с `main.rs` **и** папкой `.git` с файлом `.gitignore`.
2. В `.gitignore` один шаблон, `/target`. Папка `target` — место, куда Cargo складывает всё, что собирает. Она может занимать сотни мегабайт и в любой момент пересобирается из вашего кода, поэтому её никогда не нужно коммитить.

Соберите проект и посмотрите статус:

```bash
cargo build
git status
```

```text
   Compiling hello-rust v0.1.0 (/Users/ada/projects/git-playground/hello-rust)
    Finished `dev` profile [unoptimized + debuginfo] target(s) in 0.17s
On branch main

No commits yet

Untracked files:
  (use "git add <file>..." to include in what will be committed)
	.gitignore
	Cargo.lock
	Cargo.toml
	src/

nothing added to commit but untracked files present (use "git add" to track)
```

**Что происходит:** сборка создала папку `target` и новый файл `Cargo.lock`. Git показывает `Cargo.lock`, но не `target`, потому что `target` игнорируется. В `Cargo.lock` записаны точные версии использованных библиотек. **Коммитьте его** для программ, чтобы у всех собиралось ровно одно и то же (подробнее вы узнаете в Уроке 05.8).

Закоммитьте проект:

```bash
git add .
git commit -m "Create the project with cargo new"
git ls-files
git check-ignore -v target
```

```text
[main (root-commit) a43950e] Create the project with cargo new
 4 files changed, 17 insertions(+)
 create mode 100644 .gitignore
 create mode 100644 Cargo.lock
 create mode 100644 Cargo.toml
 create mode 100644 src/main.rs
.gitignore
Cargo.lock
Cargo.toml
src/main.rs
.gitignore:1:/target	target
```

**Что происходит:** `git add .` подготавливает всё в текущей папке (точка означает «здесь»). `git ls-files` перечисляет файлы, которые отслеживает Git: `target` в списке нет. `git check-ignore -v target` объясняет почему: строка 1 файла `.gitignore`.

> ⚠️ **Частая ошибка:** `git add .` подготавливает *всё, что не игнорируется*, в том числе файлы, которые вы не собирались коммитить (файл `.env`, большой тестовый файл). **Всегда выполняйте `git status` перед `git commit`** и читайте список.

> 📏 **Соглашение:** коммитьте `Cargo.toml`, `Cargo.lock` (для программ), `src/` и `.gitignore` проекта. Никогда не коммитьте `target/`.

Если создать проект **внутри** папки, которая уже является репозиторием, `cargo new` не создаёт ещё один репозиторий. С этим связана одна ловушка; вы встретитесь с ней в [Уроке 01.10](10-project-your-learning-repo-and-checkpoint.md).

## 9. Удаление и переименование файлов

Если удалить или переименовать файл обычными командами, Git увидит изменение, и вы его подготовите. У Git есть и собственные команды, которые делают два шага сразу.

```bash
echo "a" > old-name.txt
git add old-name.txt
git commit -m "Add old-name.txt"
git mv old-name.txt new-name.txt
git status -s
git commit -m "Rename old-name.txt to new-name.txt"
git rm new-name.txt
git commit -m "Remove new-name.txt"
```

```text
[main 2bc8f1b] Add old-name.txt
 1 file changed, 1 insertion(+)
 create mode 100644 old-name.txt
R  old-name.txt -> new-name.txt
[main ac4f9ef] Rename old-name.txt to new-name.txt
 1 file changed, 0 insertions(+), 0 deletions(-)
 rename old-name.txt => new-name.txt (100%)
rm 'new-name.txt'
[main eb766c8] Remove new-name.txt
 1 file changed, 1 deletion(-)
 delete mode 100644 new-name.txt
```

**Что происходит:**

1. `git mv a b` переименовывает (или перемещает) файл и подготавливает изменение. `R` в статусе значит «переименован». Git не хранит переименование; он *обнаруживает* его, потому что содержимое то же самое (`100%`).
2. `git rm file` удаляет файл из вашей папки **и** подготавливает удаление.
3. В старых коммитах файл всё ещё есть, поэтому удаление файла никогда не теряет историю.

Git не отслеживает **пустые папки**, только файлы. Если нужна пустая папка в репозитории, положите в неё пустой файл; по соглашению его называют `.gitkeep`.

## Частые ошибки

> ⚠️ **Частая ошибка:** вы изменили файл, выполнили `git commit -m "..."`, а Git говорит «no changes added to commit». Вы забыли `git add`. Выполните `git status`: файл будет в разделе «Changes not staged for commit». Подготовьте его и закоммитьте снова. (Или используйте `git commit -a -m "..."`: она подготавливает все изменения *уже отслеживаемых* файлов перед коммитом. Новые файлы она не включает.)

> ⚠️ **Частая ошибка:** `nothing to commit, working tree clean`. Нового для сохранения нет. Это не ошибка; Git завершается с кодом 1 лишь затем, чтобы сообщить, что коммит не сделан.

> ⚠️ **Частая ошибка:** очень длинное сообщение коммита, набранное через `-m` с несколькими кавычками. Используйте `git commit` без `-m` и пишите в редакторе.

> ⚠️ **Частая ошибка:** гигантский коммит с работой за неделю и сообщением «work». Когда что-то ломается, вы не можете найти, какая часть виновата. Коммитьте мелко и часто.

> ⚠️ **Частая ошибка:** коммит `target/`, `.DS_Store` или `.idea/`. Создавайте правила игнорирования в начале проекта (для последних двух вы уже создали глобальное правило в [Уроке 01.2](02-install-and-configure-git.md)).

## Попробуйте сами

Выполните каждый шаг и сравните с ожидаемым результатом.

1. `cd ~/projects/git-playground && mkdir recipes && cd recipes && git init`.
2. Создайте файл `pancakes.txt` из двух строк (по ингредиенту в каждой) с помощью `echo` и `>>`.
3. Выполните `git status -s`. Ожидайте `?? pancakes.txt`.
4. Выполните `git add pancakes.txt`, затем `git status -s`. Ожидайте `A  pancakes.txt`.
5. Сделайте коммит с сообщением `Add pancakes recipe`.
6. Добавьте в файл третий ингредиент. Выполните `git status -s`. Ожидайте ` M pancakes.txt` (пробел, затем `M`).
7. Выполните `git diff` и найдите строку, начинающуюся с `+`.
8. Сделайте коммит с сообщением `Add a third ingredient to pancakes`.
9. Выполните `git log --oneline`. Ожидайте два коммита, сначала новый.
10. Создайте `scratch.tmp` и `.gitignore` со строкой `*.tmp`. Проверьте, что `git status -s` показывает только `?? .gitignore`. Закоммитьте его с хорошим сообщением.

## Упражнения

### Упражнение 1 — Первый коммит (легко)

Создайте репозиторий `todo-list` в `~/projects/git-playground`. Сделайте файл `todo.txt` из двух строк (`buy milk` и `write lesson`) и закоммитьте его с сообщением `Add the first todo list`. Покажите историю.

<details>
<summary>Подсказка</summary>

Используйте `git init`, файл, `git add`, `git commit -m`, `git log --oneline`.

</details>

<details>
<summary>Ответ</summary>

```bash
cd ~/projects/git-playground
mkdir todo-list
cd todo-list
git init
echo "buy milk" > todo.txt
echo "write lesson" >> todo.txt
git add todo.txt
git commit -m "Add the first todo list"
git log --oneline
```

```text
[main (root-commit) e0aba58] Add the first todo list
 1 file changed, 2 insertions(+)
 create mode 100644 todo.txt
e0aba58 Add the first todo list
```

Почему это работает: `git add` помещает файл в область подготовки, а `git commit` сохраняет его как первый коммит. Хеш на вашем компьютере будет другим.

</details>

### Упражнение 2 — Прочитайте статус (легко)

В `todo-list` вы закоммитили `README.md` и `todo.txt`. Затем добавили строку в `README.md` и создали новый файл `notes.txt`. Что напечатает `git status -s`? Проверьте на своём компьютере.

<details>
<summary>Подсказка</summary>

Один файл отслеживается и изменён, другой новый. Вспомните двухбуквенные коды.

</details>

<details>
<summary>Ответ</summary>

```text
 M README.md
?? notes.txt
```

` M` (пробел, потом M) означает «изменён, но не подготовлен» (первая колонка — область подготовки — пуста). `??` означает «неотслеживаемый».

Почему это работает: первая буква описывает область подготовки, а вторая — рабочее дерево.

</details>

### Упражнение 3 — Закоммитьте только одно из двух изменений (средне)

В `todo-list` измените и `README.md`, и `todo.txt`. Сделайте **два отдельных коммита**: сначала только изменение README (сообщение `Describe the project in README`), затем изменение todo (сообщение `Add a todo item`). Покажите `git status -s` после первого коммита и итоговый `git log --oneline`.

<details>
<summary>Подсказка</summary>

Подготовьте только один файл через `git add README.md`. Для второго коммита используйте `git commit -a` или `git add todo.txt`.

</details>

<details>
<summary>Ответ</summary>

```bash
echo "A small project." >> README.md
echo "call the doctor" >> todo.txt
git add README.md
git commit -m "Describe the project in README"
git status -s
git commit -am "Add a todo item"
git log --oneline
```

```text
[main 25df0e4] Describe the project in README
 1 file changed, 1 insertion(+)
 M todo.txt
[main c61e27a] Add a todo item
 1 file changed, 1 insertion(+)
c61e27a Add a todo item
25df0e4 Describe the project in README
6dc73b4 Add README
e0aba58 Add the first todo list
```

Почему это работает: в первый коммит попадает только подготовленный файл (`README.md`). `todo.txt` остаётся изменённым (` M`) до второго коммита. `-a` подготавливает каждый изменённый отслеживаемый файл; `-am` объединяет `-a` и `-m`.

</details>

### Упражнение 4 — Игнорируйте и перестаньте отслеживать (средне)

В `todo-list` создайте `.gitignore` с шаблонами `*.tmp` и `build/`. Докажите, что `build/out.bin` и `cache.tmp` игнорируются. Затем имитируйте ошибку: принудительно добавьте `old.tmp` (`git add -f old.tmp`) и закоммитьте его, а потом сделайте так, чтобы Git перестал его отслеживать, **не удаляя** файл.

<details>
<summary>Подсказка</summary>

`git status -s` показывает, что видит Git. Чтобы перестать отслеживать файл, но сохранить его, используйте `git rm --cached`.

</details>

<details>
<summary>Ответ</summary>

```bash
printf '*.tmp\nbuild/\n' > .gitignore
mkdir build
touch build/out.bin cache.tmp
git status -s
git add .gitignore
git commit -m "Ignore temporary files and build output"
echo data > old.tmp
git add -f old.tmp
git commit -m "Oops, tracked a tmp file"
git rm --cached old.tmp
git commit -m "Stop tracking old.tmp"
ls old.tmp
git ls-files
```

Первый `git status -s` показывает только `?? .gitignore` (и другие новые файлы, например `notes.txt`); `build/out.bin` и `cache.tmp` в списке нет. В конце `ls old.tmp` всё ещё находит файл, а `git ls-files` его не перечисляет.

Почему это работает: `-f` (force) добавляет игнорируемый файл. `git rm --cached` убирает файл только из области подготовки, поэтому следующий коммит фиксирует его удаление с точки зрения истории, а копия на диске остаётся.

</details>

### Упражнение 5 — Проект на Rust с чистой историей (сложно)

Создайте проект на Rust `calc` командой `cargo new calc` внутри `~/projects/git-playground`. Сделайте **три коммита** с хорошими сообщениями: (1) проект, созданный Cargo; (2) изменённое приветствие в `src/main.rs` (измените текст внутри `println!`); (3) `README.md`. Затем покажите три коммита через `git log --oneline` и докажите, что `target/` не отслеживается.

<details>
<summary>Подсказка</summary>

`cargo new` уже выполняет `git init`. Соберите или запустите программу один раз, чтобы появилась `target/`, затем используйте `git ls-files`.

</details>

<details>
<summary>Ответ</summary>

```bash
cd ~/projects/git-playground
cargo new calc
cd calc
cargo build
git add .
git commit -m "Create the project with cargo new"
sed -i '' 's/Hello, world!/Hello from calc!/' src/main.rs
cargo run
git commit -am "Change the greeting"
echo "# calc" > README.md
git add README.md
git commit -m "Add README"
git log --oneline
git ls-files
```

```text
Hello from calc!
```

(Вывод `git log --oneline` — три строки с вашими хешами: `Add README`, `Change the greeting`, `Create the project with cargo new`. `git ls-files` перечисляет `.gitignore`, `Cargo.lock`, `Cargo.toml`, `README.md` и `src/main.rs` — без `target`.)

Почему это работает: `cargo new` создал репозиторий и `.gitignore` с `/target`. `cargo build` и `cargo run` собирают в `target/`, которую Git игнорирует. Сборка до первого `git add .` создаёт `Cargo.lock`, поэтому он тоже попадает в коммит. Команда `sed` заменяет текст в файле; можно также отредактировать файл в RustRover. В macOS `sed -i ''` правит файл на месте.

</details>

## Вопросы для самопроверки

1. Что создаёт `git init`?

<details>
<summary>Ответ</summary>

Скрытую папку `.git` в текущей папке, которая превращает её в репозиторий.

</details>

2. Чем отличаются `git add` и `git commit`?

<details>
<summary>Ответ</summary>

`git add` помещает изменения в область подготовки (выбирает, что будет сохранено). `git commit` сохраняет подготовленные изменения как новый коммит в истории.

</details>

3. Что означает `?? notes.txt` в `git status -s`?

<details>
<summary>Ответ</summary>

`notes.txt` неотслеживаемый: это новый файл, за которым Git пока не следит.

</details>

4. Почему сообщение коммита должно быть в повелительном наклонении и объяснять *зачем*?

<details>
<summary>Ответ</summary>

Повелительное наклонение («Fix …», «Add …») — общепринятое соглашение, и сообщение читается как задача, выполненная коммитом. *Зачем* важно, потому что diff уже показывает, *что* изменилось, но не причину.

</details>

5. Вы добавили `*.tmp` в `.gitignore`, но Git по-прежнему отслеживает `old.tmp`. Почему и как это исправить?

<details>
<summary>Ответ</summary>

`.gitignore` действует только на неотслеживаемые файлы. Выполните `git rm --cached old.tmp` и закоммитьте; файл останется на диске, но отслеживаться перестанет.

</details>

6. Какие файлы и папки Cargo вы коммитите, а какую — никогда?

<details>
<summary>Ответ</summary>

Коммитьте `Cargo.toml`, `Cargo.lock` (для программ), `src/` и `.gitignore`. Никогда не коммитьте `target/`.

</details>

7. Что говорит `git status`, когда всё закоммичено?

<details>
<summary>Ответ</summary>

`nothing to commit, working tree clean`.

</details>

## Итоги

- `git init` создаёт репозиторий; история живёт в скрытой папке `.git`.
- Файл бывает неотслеживаемым, изменённым, подготовленным или закоммиченным. `git add` подготавливает, `git commit` сохраняет.
- `git status` (и `git status -s`) — ваша карта; выполняйте его перед каждым коммитом.
- В коммите есть снимок, родитель, автор и сообщение; его хеш его идентифицирует.
- Хорошие сообщения: короткая тема в повелительном наклонении, необязательное тело с объяснением *зачем*, одно логическое изменение на коммит.
- `.gitignore` не пускает в историю результат сборки, временные файлы и секреты, но только для неотслеживаемых файлов.
- Для Rust коммитьте `Cargo.toml`, `Cargo.lock` и `src/`; никогда не коммитьте `target/`.

## Источники и что почитать

- [Pro Git — 2.1 Git Basics: Getting a Git Repository](https://git-scm.com/book/en/v2/Git-Basics-Getting-a-Git-Repository) — `git init` и `git clone`.
- [Pro Git — 2.2 Recording Changes to the Repository](https://git-scm.com/book/en/v2/Git-Basics-Recording-Changes-to-the-Repository) — жизненный цикл файла подробно, `.gitignore`, `git rm`, `git mv`.
- [gitignore manual](https://git-scm.com/docs/gitignore) — точные правила шаблонов.
- [The Cargo Book — Creating a New Package](https://doc.rust-lang.org/cargo/guide/creating-a-new-project.html) — что создаёт `cargo new`, включая опцию VCS (Version Control System, система контроля версий).
- [The Cargo Book — Cargo.toml vs Cargo.lock](https://doc.rust-lang.org/cargo/guide/cargo-toml-vs-cargo-lock.html) — когда коммитить `Cargo.lock`.

---

[🏠 Начало](../START-HERE.md) · [← 01.2 Установка и настройка Git](02-install-and-configure-git.md) · [📋 Этап 01](README.md) · [01.4 История и просмотр изменений →](04-viewing-history-and-changes.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
