<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 01.3 Первый репозиторий и коммиты](03-your-first-repository-and-commits.md) · [📋 Этап 01](README.md) · [01.5 Как безопасно отменять изменения →](05-undoing-changes-safely.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/01-git-and-version-control/04-viewing-history-and-changes.md) <!-- nav -->

# Урок 01.4 — История и просмотр изменений

> **Этап 01 — Git и контроль версий** · Урок 4 из 10 · ⏱ около 3 часов (2 занятия)

## Что вы узнаете

- Как читать историю командой `git log` в коротком и длинном виде и как её фильтровать.
- Как читать diff и как сравнивать рабочие файлы, подготовленные файлы и любые два коммита командой `git diff`.
- Как смотреть один коммит командой `git show` и как называть коммиты (`HEAD~2`, короткие хеши).
- Как узнать, кто изменил строку (`git blame`) и когда появился фрагмент текста (`git log -S`).
- Как найти коммит, который что-то сломал, с помощью `git bisect`.

## Перед началом

- Вы прошли [Урок 01.3 — Первый репозиторий и коммиты](03-your-first-repository-and-commits.md).
- Вы работаете в `~/projects/git-playground`.

## Новые слова

| Слово | Значение |
|---|---|
| **ревизия** (revision) | Любой способ назвать коммит: хеш, `HEAD`, `HEAD~2`, имя ветки или тег. |
| **`HEAD~N`** | Коммит, который на N шагов раньше `HEAD`. `HEAD~1` (или `HEAD^`) — родитель. |
| **ханк** (hunk) | Один непрерывный блок изменений внутри diff. |
| **унифицированный diff** (unified diff) | Стандартный формат diff со строками `-` и `+` и маркерами `@@`. |
| **blame** | Показ для каждой строки файла коммита и автора, которые изменили её последними. |
| **bisect** | Поиск коммита, внёсшего проблему, путём многократного деления истории пополам (двоичный поиск). |
| **pickaxe** («кирка») | Прозвище `git log -S`, который находит коммиты, где изменилось число вхождений текста. |
| **пейджер** (pager) | Программа (`less`), показывающая длинный вывод по одному экрану. `Пробел` — следующая страница, `q` — выход. |

## 1. Небольшая история для изучения

Вам нужен репозиторий с несколькими коммитами. Создайте его, вставив этот блок в терминал. Сначала прочитайте его: он делает пять коммитов для крошечной программы `greeter.rs`, а третий коммит «написан» другим автором, чтобы вы могли потренироваться в фильтрации.

```bash
cd ~/projects/git-playground
mkdir history-demo
cd history-demo
git init

printf 'fn main() {\n    println!("Hello");\n}\n' > greeter.rs
echo "# history-demo" > README.md
git add .
git commit -m "Add greeter and README"

printf 'fn main() {\n    println!("Hello, world");\n}\n' > greeter.rs
git commit -am "Greet the whole world"

printf 'fn greeting() -> String {\n    String::from("Hello, world")\n}\n\nfn main() {\n    println!("{}", greeting());\n}\n' > greeter.rs
git commit -am "Move the greeting into a function" --author="Bob Builder <bob@example.com>"

printf '# history-demo\n\nA tiny program that says hello.\n' > README.md
git commit -am "Describe the program in README"

printf 'fn greeting() -> String {\n    String::from("Hello, Rust world")\n}\n\nfn main() {\n    println!("{}", greeting());\n}\n' > greeter.rs
git commit -am "Mention Rust in the greeting"
```

**Что происходит:** каждый `printf` переписывает файл, а каждый `git commit -am` сохраняет все изменения отслеживаемых файлов с сообщением. Опция `--author` задаёт другого автора для одного коммита. (Код на Rust понимать пока не нужно; это просто текст для отслеживания. Язык изучается на Этапе 02.)

Выводы ниже получены в настоящем запуске. **Ваши хеши и даты будут другими.** Когда команде нужен хеш, всегда подставляйте свой.

## 2. `git log`: читаем историю

```bash
git log
```

```text
commit 996bec4df4b786faa0bf3207bd26019a7367d600
Author: Ada Learner <ada@example.com>
Date:   Mon Nov 16 16:45:00 2026 +0500

    Mention Rust in the greeting

commit a499fb029cfeab10cc396e420a4174688b9f0607
Author: Ada Learner <ada@example.com>
Date:   Fri Nov 13 14:00:00 2026 +0500

    Describe the program in README

commit 3c8a092fd0b278e71d36eb7dfba44da1cf9038f4
Author: Bob Builder <bob@example.com>
Date:   Thu Nov 12 09:15:00 2026 +0500

    Move the greeting into a function
…
```

**Что происходит:** коммиты перечислены **сначала новые**. Для каждого показаны полный хеш, автор, дата и сообщение. Если список длиннее экрана, Git открывает пейджер: `Пробел` — следующая страница, `q` — выход.

Короткие формы, которыми вы будете пользоваться ежедневно:

```bash
git log --oneline
git log -3 --oneline
git log --oneline --graph --decorate
```

```text
996bec4 Mention Rust in the greeting
a499fb0 Describe the program in README
3c8a092 Move the greeting into a function
2c71f79 Greet the whole world
bd338f3 Add greeter and README
```

```text
996bec4 Mention Rust in the greeting
a499fb0 Describe the program in README
3c8a092 Move the greeting into a function
```

```text
* 996bec4 (HEAD -> main) Mention Rust in the greeting
* a499fb0 Describe the program in README
* 3c8a092 Move the greeting into a function
* 2c71f79 Greet the whole world
* bd338f3 Add greeter and README
```

**Что происходит:**

1. `--oneline` печатает по одной строке на коммит: короткий хеш и тему.
2. `-3` ограничивает вывод тремя последними коммитами (`-n 3` тоже работает).
3. `--graph` рисует слева форму истории (сейчас это прямая линия; ветки будут в следующих уроках), а `--decorate` показывает имена, прикреплённые к коммитам: `HEAD -> main` значит «вы на ветке `main`, и она указывает сюда».

Покажите файлы, затронутые каждым коммитом, или полные изменения:

```bash
git log --stat -2
git log -p -1
```

```text
commit 996bec4df4b786faa0bf3207bd26019a7367d600
Author: Ada Learner <ada@example.com>
Date:   Mon Nov 16 16:45:00 2026 +0500

    Mention Rust in the greeting

 greeter.rs | 2 +-
 1 file changed, 1 insertion(+), 1 deletion(-)
…
```

```text
commit 996bec4df4b786faa0bf3207bd26019a7367d600
Author: Ada Learner <ada@example.com>
Date:   Mon Nov 16 16:45:00 2026 +0500

    Mention Rust in the greeting

diff --git a/greeter.rs b/greeter.rs
index 294d8b3..dac1d51 100644
--- a/greeter.rs
+++ b/greeter.rs
@@ -1,5 +1,5 @@
 fn greeting() -> String {
-    String::from("Hello, world")
+    String::from("Hello, Rust world")
 }
 
 fn main() {
```

`--stat` добавляет сводку по изменённым файлам; `-p` (patch) добавляет полный diff каждого коммита. Этот формат diff мы разберём в разделе 4.

### Фильтрация истории

| Команда | Что показывает |
|---|---|
| `git log --author=Bob --oneline` | коммиты, в имени автора которых есть «Bob» |
| `git log --grep=README --oneline` | коммиты, в **сообщении** которых есть «README» |
| `git log --since="2 weeks ago" --oneline` | коммиты новее указанной даты (есть и `--until`) |
| `git log --oneline -- README.md` | коммиты, которые изменили файл `README.md` (`--` отделяет пути от опций) |
| `git log --pretty=format:'%h %an %ad %s' --date=short` | ваш собственный формат: `%h` короткий хеш, `%an` имя автора, `%ad` дата, `%s` тема |

```bash
git log --author=Bob --oneline
git log --grep=README --oneline
git log --oneline -- README.md
```

```text
3c8a092 Move the greeting into a function
```

```text
a499fb0 Describe the program in README
bd338f3 Add greeter and README
```

```text
a499fb0 Describe the program in README
bd338f3 Add greeter and README
```

## 3. Как называть коммиты

Многим командам нужен «коммит». Назвать его можно несколькими способами:

| Имя | Значение |
|---|---|
| `996bec4df4b786faa0bf3207bd26019a7367d600` | полный хеш |
| `996bec4` | короткий хеш: достаточно первых символов, чтобы он был уникальным (обычно 7) |
| `HEAD` | коммит, на котором вы сейчас стоите |
| `HEAD~1` или `HEAD^` | его родитель (на шаг назад) |
| `HEAD~3` | три шага назад |
| `main` | коммит, на который указывает ветка `main` |
| `HEAD~3..HEAD` | **диапазон**: коммиты после `HEAD~3` до `HEAD` |

```text
 bd338f3 ◀─ 2c71f79 ◀─ 3c8a092 ◀─ a499fb0 ◀─ 996bec4
  HEAD~4     HEAD~3     HEAD~2     HEAD~1      HEAD
```

```bash
git rev-parse HEAD
git rev-parse --short HEAD
git rev-parse HEAD~2
git log --oneline HEAD~3..HEAD
```

```text
996bec4df4b786faa0bf3207bd26019a7367d600
996bec4
3c8a092fd0b278e71d36eb7dfba44da1cf9038f4
996bec4 Mention Rust in the greeting
a499fb0 Describe the program in README
3c8a092 Move the greeting into a function
```

**Что происходит:** `git rev-parse` превращает любое имя в полный хеш (`--short` даёт короткий). Диапазон `HEAD~3..HEAD` означает «достижимо из `HEAD`, но не из `HEAD~3`»: три последних коммита.

## 4. `git show` и как читать diff

`git show <ревизия>` показывает один коммит: сообщение и его diff. Без ревизии показывает `HEAD`.

```bash
git show HEAD~2
```

```text
commit 3c8a092fd0b278e71d36eb7dfba44da1cf9038f4
Author: Bob Builder <bob@example.com>
Date:   Thu Nov 12 09:15:00 2026 +0500

    Move the greeting into a function

diff --git a/greeter.rs b/greeter.rs
index bcb94c2..294d8b3 100644
--- a/greeter.rs
+++ b/greeter.rs
@@ -1,3 +1,7 @@
+fn greeting() -> String {
+    String::from("Hello, world")
+}
+
 fn main() {
-    println!("Hello, world");
+    println!("{}", greeting());
 }
```

Как читать его строка за строкой:

| Строка | Значение |
|---|---|
| `diff --git a/greeter.rs b/greeter.rs` | сравниваемый файл; `a/` — старая версия, `b/` — новая |
| `index bcb94c2..294d8b3 100644` | короткие хеши старого и нового содержимого файла и режим файла |
| `--- a/greeter.rs` / `+++ b/greeter.rs` | старый файл / новый файл |
| `@@ -1,3 +1,7 @@` | заголовок **ханка**: в старом файле 3 строки начиная со строки 1; в новом — 7 строк начиная со строки 1 |
| строки, начинающиеся с `-` | удалены |
| строки, начинающиеся с `+` | добавлены |
| строки, начинающиеся с пробела | неизменённый контекст, показан, чтобы вам было проще найти место |

Изменённая строка выглядит как строка с `-` (старая), за которой идёт строка с `+` (новая).

Можно также посмотреть файл *таким, каким он был* в коммите, и перечислить только имена изменённых файлов:

```bash
git show 3c8a092:greeter.rs
git show --stat HEAD~1
```

```text
fn greeting() -> String {
    String::from("Hello, world")
}

fn main() {
    println!("{}", greeting());
}
```

```text
commit a499fb029cfeab10cc396e420a4174688b9f0607
Author: Ada Learner <ada@example.com>
Date:   Fri Nov 13 14:00:00 2026 +0500

    Describe the program in README

 README.md | 2 ++
 1 file changed, 2 insertions(+)
```

## 5. `git diff`: сравниваем

`git diff` сравнивает два состояния. Какие именно — зависит от опций:

| Команда | Сравнивает |
|---|---|
| `git diff` | рабочее дерево **с** областью подготовки: ваши изменения, которые *ещё не подготовлены* |
| `git diff --staged` (или `--cached`) | область подготовки **с** последним коммитом: что будет в следующем коммите |
| `git diff HEAD` | рабочее дерево **с** последним коммитом: всё, что вы изменили после последнего коммита |
| `git diff A B` | коммит `A` **с** коммитом `B` |
| `git diff A..B` | то же самое |
| `git diff A B -- file` | только один файл |
| `git diff --stat` | только сводка по изменённым файлам и строкам |
| `git diff --name-only` | только имена изменённых файлов |

Проверьте трёхуровневое поведение сами:

```bash
echo "" >> README.md
echo "More text." >> README.md
git diff
git diff --staged
git add README.md
git diff
git diff --staged
```

```text
diff --git a/README.md b/README.md
index 9c4d855..424b4fd 100644
--- a/README.md
+++ b/README.md
@@ -1,3 +1,5 @@
 # history-demo
 
 A tiny program that says hello.
+
+More text.
```

После первого `git diff` команда `git diff --staged` ничего не печатает (ничего не подготовлено). После `git add` картина переворачивается: `git diff` ничего не печатает, а `git diff --staged` показывает тот же ханк.

> 💡 **Совет:** перед каждым коммитом выполняйте `git diff --staged`, чтобы просмотреть, что именно вы собираетесь сохранить. Это самое дешёвое код-ревью. Отмените эксперимент командами `git restore --staged README.md`, а затем `git restore README.md` (вы изучите их в [Уроке 01.5](05-undoing-changes-safely.md)).

Сравните два коммита:

```bash
git diff HEAD~1 HEAD
git diff --stat HEAD~3 HEAD
```

```text
diff --git a/greeter.rs b/greeter.rs
index 294d8b3..dac1d51 100644
--- a/greeter.rs
+++ b/greeter.rs
@@ -1,5 +1,5 @@
 fn greeting() -> String {
-    String::from("Hello, world")
+    String::from("Hello, Rust world")
 }
 
 fn main() {
```

```text
 README.md  | 2 ++
 greeter.rs | 6 +++++-
 2 files changed, 7 insertions(+), 1 deletion(-)
```

## 6. `git blame`: кто написал эту строку?

```bash
git blame greeter.rs
```

```text
3c8a092f (Bob Builder 2026-11-12 09:15:00 +0500 1) fn greeting() -> String {
996bec4d (Ada Learner 2026-11-16 16:45:00 +0500 2)     String::from("Hello, Rust world")
3c8a092f (Bob Builder 2026-11-12 09:15:00 +0500 3) }
3c8a092f (Bob Builder 2026-11-12 09:15:00 +0500 4) 
^bd338f3 (Ada Learner 2026-11-10 10:00:00 +0500 5) fn main() {
3c8a092f (Bob Builder 2026-11-12 09:15:00 +0500 6)     println!("{}", greeting());
^bd338f3 (Ada Learner 2026-11-10 10:00:00 +0500 7) }
```

**Что происходит:** перед каждой строкой стоит коммит, который **изменил её последним**, автор, дата и номер строки. Ведущий `^` помечает первый коммит репозитория. Чтобы перейти к коммиту, выполните `git show 3c8a092`. Опция `-L 6,6` проверяет только одну строку.

`blame` не про то, чтобы кого-то обвинять. Он отвечает на вопрос «почему эта строка выглядит так?», приводя вас к сообщению коммита. В RustRover такой же вид есть: щёлкните правой кнопкой по левому полю редактора и выберите *Annotate with Git Blame*.

## 7. Поиск по истории

Какой коммит добавил слово «Rust» в приветствие? `git log -S` (*pickaxe*) перечисляет коммиты, в которых изменилось число вхождений текста:

```bash
git log -S"Rust world" --oneline
git log -S"greeting()" --oneline
```

```text
996bec4 Mention Rust in the greeting
```

```text
3c8a092 Move the greeting into a function
```

**Что происходит:** первая команда находит единственный коммит, добавивший `Rust world`. Вторая находит коммит, в котором появилось имя функции.

Искать по **текущим** файлам (а не по истории) можно командой `git grep`:

```bash
git grep -n "Hello"
```

```text
greeter.rs:2:    String::from("Hello, Rust world")
```

`git grep` ищет только в файлах, отслеживаемых Git, поэтому пропускает `target/` и другие игнорируемые файлы; так он быстрее и тише обычного `grep`.

## 8. `git bisect`: найти коммит, который что-то сломал

Представьте: тест проходил на прошлой неделе, а сегодня падает, и между этими моментами 100 коммитов. Проверять каждый долго. `git bisect` находит виновный коммит **двоичным поиском**: просит вас проверить средний коммит, отбрасывает половину истории и повторяет. На 100 коммитов нужно всего около 7 проверок.

Вы сообщаете Git **плохой** коммит (где ошибка есть) и **хороший** (где её не было). Git переключается на средний, а вы отвечаете `good` или `bad`.

Подготовьте небольшой репозиторий со спрятанной «ошибкой» на шаге 5 (скопируйте и вставьте):

```bash
cd ~/projects/git-playground
mkdir bisect-demo
cd bisect-demo
git init
for i in 1 2 3 4 5 6 7 8; do
  echo "line $i" >> app.txt
  if [ $i = 5 ]; then echo "BUG" >> app.txt; fi
  git add app.txt
  git commit -m "Step $i"
done
git log --oneline
```

```text
30f26db Step 8
be02adb Step 7
7e63b37 Step 6
491dfe9 Step 5
e1a05de Step 4
38ae93a Step 3
f0432b8 Step 2
4669310 Step 1
```

(Цикл повторяет шаги для чисел от 1 до 8. Условие `if` добавляет строку `BUG` на шаге 5.) Теперь выполните bisect вручную:

```bash
git bisect start
git bisect bad
git bisect good HEAD~7
```

```text
status: waiting for both good and bad commits
status: waiting for good commit(s), bad commit known
Bisecting: 3 revisions left to test after this (roughly 2 steps)
[e1a05ded1cffb1d3f583dd3549403081d4e41ed0] Step 4
```

**Что происходит:**

1. `git bisect start` начинает сеанс.
2. `git bisect bad` помечает текущий коммит (`HEAD`, шаг 8) как плохой.
3. `git bisect good HEAD~7` помечает шаг 1 как хороший.
4. Git переключает вас на средний коммит (шаг 4), чтобы вы его проверили. Ваши файлы теперь выглядят как на шаге 4.

Проверьте его (здесь: есть ли в `app.txt` слово `BUG`?) и ответьте:

```bash
grep BUG app.txt
git bisect good
```

`grep` ничего не печатает, значит, шаг 4 хороший. Git переходит к шагу 6, затем к шагу 5. Каждый раз вы проверяете и отвечаете `git bisect good` или `git bisect bad`. Когда ответ найден, Git печатает:

```text
491dfe96ef4781af3dd52b3af4888eb7b644fa27 is the first bad commit
```

Завершите командой `git bisect reset`, которая вернёт вас туда, откуда вы начали:

```bash
git bisect reset
```

```text
Previous HEAD position was 491dfe9 Step 5
Switched to branch 'main'
```

Проверку можно поручить и скрипту. Скрипт, завершившийся с кодом 0, означает «хорошо», а ненулевым — «плохо»:

```bash
printf '#!/bin/sh\n! grep -q BUG app.txt\n' > check.sh
chmod +x check.sh
git bisect start HEAD HEAD~7
git bisect run ./check.sh
git bisect reset
```

```text
Bisecting: 3 revisions left to test after this (roughly 2 steps)
[e1a05ded1cffb1d3f583dd3549403081d4e41ed0] Step 4
running './check.sh'
Bisecting: 1 revision left to test after this (roughly 1 step)
[7e63b373500d785f2995587315b13582541f088e] Step 6
running './check.sh'
Bisecting: 0 revisions left to test after this (roughly 0 steps)
[491dfe96ef4781af3dd52b3af4888eb7b644fa27] Step 5
running './check.sh'
491dfe96ef4781af3dd52b3af4888eb7b644fa27 is the first bad commit
bisect found first bad commit
```

**Что происходит:** `git bisect start HEAD HEAD~7` начинает сразу с заданными плохим и хорошим коммитами. `git bisect run ./check.sh` запускает скрипт на каждом среднем коммите. Скрипт `check.sh` выполняет `grep -q BUG app.txt` (`-q` = тихо), а `!` инвертирует результат: успех, если `BUG` **не** найден. В проекте на Rust скриптом обычно будет `cargo test`, потому что он тоже завершается с 0 при успехе и с ненулевым кодом при ошибке.

> 🧪 **Тестирование:** `git bisect run cargo test` — мощная комбинация: падающего теста и длинной истории достаточно, чтобы найти точный коммит. Собственные тесты вы напишете начиная с [Урока 02.8](../02-programming-basics/08-your-first-tests.md).

## Частые ошибки

> ⚠️ **Частая ошибка:** `git diff` ничего не печатает, и вы думаете, что ничего не изменилось. Если вы уже выполнили `git add`, изменения в области подготовки: используйте `git diff --staged`.

> ⚠️ **Частая ошибка:** вы застряли в пейджере. Внизу экрана `:` или `(END)`. Нажмите `q`.

> ⚠️ **Частая ошибка:** `fatal: ambiguous argument 'HEAD~10': unknown revision`. История короче, чем вы просили. Выполните `git log --oneline` и посчитайте.

> ⚠️ **Частая ошибка:** оставить сеанс bisect открытым и забыть о нём. В строке статуса будет `(no branch, bisect started on main)`. Всегда завершайте командой `git bisect reset`.

> ⚠️ **Частая ошибка:** использовать `git blame`, чтобы найти виноватого. Смысл в том, чтобы прочитать сообщение коммита и контекст. Будьте доброжелательны в ревью.

## Попробуйте сами

В `history-demo`:

1. Выполните `git log --oneline` и запишите хеш второго коммита.
2. Выполните `git show <этот хеш>`. Какая строка изменилась? Прочитайте строки `-` и `+`.
3. Выполните `git log --stat` и выясните, сколько строк добавил третий коммит.
4. Выполните `git diff HEAD~4 HEAD --stat`. Какие файлы изменились в сумме?
5. Отредактируйте `README.md`: добавьте строку, затем сравните `git diff` и `git diff --staged` до и после `git add`.
6. Выполните `git blame README.md`. Какой коммит написал последнюю строку?
7. Выполните `git restore --staged README.md && git restore README.md`, чтобы оставить репозиторий чистым.

## Упражнения

### Упражнение 1 — Посчитайте и назовите (легко)

Выведите только число коммитов в `history-demo` и только тему (сообщение) последнего коммита.

<details>
<summary>Подсказка</summary>

`git log --oneline` печатает по строке на коммит; `wc -l` считает строки. Для темы используйте `git log -1 --format=%s` (`%s` — тема).

</details>

<details>
<summary>Ответ</summary>

```bash
git log --oneline | wc -l
git log -1 --format=%s
```

```text
       5
Mention Rust in the greeting
```

Почему это работает: `|` передаёт вывод `git log` в `wc -l`, который считает строки (пробелы перед числом в macOS нормальны). `-1` ограничивает одним коммитом, а `--format=%s` печатает только его тему.

</details>

### Упражнение 2 — Что изменилось в коммите? (легко)

Покажите, какие файлы изменил коммит «Describe the program in README», **не** показывая diff.

<details>
<summary>Подсказка</summary>

У `git show` есть опция `--stat`, а также `--name-only`.

</details>

<details>
<summary>Ответ</summary>

```bash
git show --stat HEAD~1
```

или только имена:

```bash
git show --name-only --format=%h HEAD~1
```

```text
a499fb0

README.md
```

Почему это работает: `HEAD~1` — предпоследний коммит. `--stat` или `--name-only` заменяют полный diff сводкой.

</details>

### Упражнение 3 — Кто написал функцию? (средне)

Выясните, кто написал строку `fn greeting() -> String {` и в каком коммите. Одной командой, затем подтвердите, показав тему этого коммита.

<details>
<summary>Подсказка</summary>

`git blame -L 1,1 greeter.rs` проверяет только строку 1. Затем используйте `git show -s --format=%s <хеш>`.

</details>

<details>
<summary>Ответ</summary>

```bash
git blame -L 1,1 greeter.rs
git show -s --format='%an: %s' 3c8a092
```

```text
3c8a092f (Bob Builder 2026-11-12 09:15:00 +0500 1) fn greeting() -> String {
Bob Builder: Move the greeting into a function
```

Используйте свой хеш из `git blame`. `-s` подавляет diff, а `%an` — имя автора.

Почему это работает: `blame` показывает последний коммит, затронувший строку. `show -s` с форматом печатает нужные вам факты.

</details>

### Упражнение 4 — Сравните первую и последнюю версию (средне)

Покажите полную разницу `greeter.rs` между самым первым коммитом и последним, а затем только сводку по всем изменённым файлам.

<details>
<summary>Подсказка</summary>

Используйте `git diff <старый> <новый> -- greeter.rs`, где старый — `HEAD~4`. Для сводки добавьте `--stat`.

</details>

<details>
<summary>Ответ</summary>

```bash
git diff HEAD~4 HEAD -- greeter.rs
git diff --stat HEAD~4 HEAD
```

```text
diff --git a/greeter.rs b/greeter.rs
index 244f744..dac1d51 100644
--- a/greeter.rs
+++ b/greeter.rs
@@ -1,3 +1,7 @@
+fn greeting() -> String {
+    String::from("Hello, Rust world")
+}
+
 fn main() {
-    println!("Hello");
+    println!("{}", greeting());
 }
```

```text
 README.md  | 2 ++
 greeter.rs | 6 +++++-
 2 files changed, 7 insertions(+), 1 deletion(-)
```

Почему это работает: `git diff A B` напрямую сравнивает два коммита, игнорируя те, что между ними.

</details>

### Упражнение 5 — Найдите виновника скриптом (сложно)

В `bisect-demo` найдите первый коммит, в котором `app.txt` содержит `BUG`, с помощью `git bisect run`, а затем вернитесь в обычное состояние. Запишите все команды.

<details>
<summary>Подсказка</summary>

Нужен скрипт, который завершается с ненулевым кодом, если файл содержит `BUG`. Начните с `git bisect start <плохой> <хороший>`.

</details>

<details>
<summary>Ответ</summary>

```bash
printf '#!/bin/sh\n! grep -q BUG app.txt\n' > check.sh
chmod +x check.sh
git bisect start HEAD HEAD~7
git bisect run ./check.sh
git bisect reset
```

Вывод заканчивается строкой `… is the first bad commit` для коммита `Step 5`.

Почему это работает: скрипт возвращает 0 (хорошо), когда `BUG` отсутствует, и ненулевой код (плохо), когда он есть, поэтому двоичный поиск находит первый коммит с `BUG` примерно за три шага. `git bisect reset` завершает сеанс.

</details>

## Вопросы для самопроверки

1. Чем отличаются `git diff` и `git diff --staged`?

<details>
<summary>Ответ</summary>

`git diff` показывает неподготовленные изменения (рабочее дерево и область подготовки). `git diff --staged` показывает то, что подготовлено для следующего коммита (область подготовки и последний коммит).

</details>

2. Что означает `HEAD~2`?

<details>
<summary>Ответ</summary>

Коммит на два шага раньше `HEAD` (родитель родителя текущего коммита).

</details>

3. Что означают в diff строки, начинающиеся с `-` и `+`?

<details>
<summary>Ответ</summary>

Строки с `-` удалены, строки с `+` добавлены. Изменённая строка показывается как по одной строке каждого вида.

</details>

4. Какая команда перечисляет только коммиты, изменившие `README.md`?

<details>
<summary>Ответ</summary>

`git log --oneline -- README.md`.

</details>

5. Как `git bisect` находит плохой коммит среди 100 примерно за 7 проверок?

<details>
<summary>Ответ</summary>

Он использует двоичный поиск: проверяет средний коммит, отбрасывает половину, где не может быть первого плохого коммита, и повторяет.

</details>

6. Какая команда находит коммит, добавивший определённый фрагмент текста?

<details>
<summary>Ответ</summary>

`git log -S"text"` (pickaxe).

</details>

## Итоги

- `git log` читает историю; `--oneline`, `--graph`, `--stat`, `-p`, `-n` и фильтры вроде `--author`, `--grep` и `-- путь` делают её полезной.
- Коммит можно назвать хешем, коротким хешем, `HEAD`, `HEAD~N` или именем ветки; `A..B` — диапазон.
- `git show` показывает один коммит; `git show <коммит>:<файл>` показывает файл в тот момент.
- В diff есть строки `-` (удалено) и `+` (добавлено), строки контекста и заголовки ханков `@@`.
- `git diff` (неподготовленное), `git diff --staged` (подготовленное), `git diff HEAD` (всё) и `git diff A B` (два коммита).
- `git blame` находит коммит каждой строки; `git log -S` и `git grep` ищут по истории и по файлам.
- `git bisect` (с `run`) находит коммит, внёсший ошибку, двоичным поиском.

## Источники и что почитать

- [Pro Git — 2.3 Viewing the Commit History](https://git-scm.com/book/en/v2/Git-Basics-Viewing-the-Commit-History) — все опции `git log`.
- [git-log manual](https://git-scm.com/docs/git-log) — полный справочник, включая метки формата.
- [git-diff manual](https://git-scm.com/docs/git-diff) — все способы сравнения.
- [Pro Git — 7.1 Revision Selection](https://git-scm.com/book/en/v2/Git-Tools-Revision-Selection) — все способы назвать коммиты (`~`, `^`, диапазоны).
- [Pro Git — 7.10 Debugging with Git](https://git-scm.com/book/en/v2/Git-Tools-Debugging-with-Git) — `blame` и `bisect` подробно.

---

[🏠 Начало](../START-HERE.md) · [← 01.3 Первый репозиторий и коммиты](03-your-first-repository-and-commits.md) · [📋 Этап 01](README.md) · [01.5 Как безопасно отменять изменения →](05-undoing-changes-safely.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
