<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 01.5 Как безопасно отменять изменения](05-undoing-changes-safely.md) · [📋 Этап 01](README.md) · [01.7 Конфликты слияния →](07-merge-conflicts.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/01-git-and-version-control/06-branches-and-merging.md) <!-- nav -->

# Урок 01.6 — Ветки и слияние

> **Этап 01 — Git и контроль версий** · Урок 6 из 10 · ⏱ около 2,5 часа (2 занятия)

## Что вы узнаете

- Что такое ветка на самом деле (маленький перемещаемый указатель) и почему ветки в Git дёшевы.
- Как создавать, перечислять, переключать, переименовывать и удалять ветки.
- Как сливать одну ветку в другую и чем отличается слияние *fast-forward* от *коммита слияния*.
- Как читать граф истории.
- Как называть ветки, как безопасно переключаться при незаконченной работе и как восстановить удалённую ветку.

## Перед началом

- Вы прошли [Урок 01.5 — Как безопасно отменять изменения](05-undoing-changes-safely.md).
- У вас есть Rust-проект `hello-rust` из [Урока 01.3](03-your-first-repository-and-commits.md) в `~/projects/git-playground`.

## Новые слова

| Слово | Значение |
|---|---|
| **ветка** (branch) | Именованный перемещаемый указатель на коммит. Отмечает вершину одной линии работы. |
| **`main`** | Принятое имя главной ветки репозитория. |
| **`HEAD`** | Указатель на ветку (или коммит), на которой вы сейчас находитесь. Новые коммиты добавляются в неё. |
| **переключиться** (switch, check out) | Переместить `HEAD` на другую ветку и привести ваши файлы в соответствие с ней. |
| **слияние** (merge) | Объединение работы одной ветки с другой. |
| **fast-forward** («перемотка вперёд») | Слияние, которое лишь сдвигает указатель ветки вперёд, потому что другая ветка содержит всё из текущей. Новый коммит не создаётся. |
| **коммит слияния** (merge commit) | Коммит с **двумя** родителями. Он фиксирует, что две линии работы объединены. |
| **трёхстороннее слияние** (three-way merge) | Слияние, которое сравнивает вершины обеих веток с их последним общим предком. |
| **общий предок** (common ancestor, merge base) | Последний коммит, общий для обеих веток. |
| **фича-ветка** (feature branch) | Недолговечная ветка для одной задачи (новой возможности или исправления). |
| **`--no-ff`** | Опция, которая принудительно создаёт коммит слияния, даже когда возможен fast-forward. |

## 1. Какую проблему решают ветки

Вы работаете над программой, которая работает. Вы хотите попробовать большое изменение, но не хотите сломать рабочую версию, а заодно, возможно, нужно срочно исправить мелкую ошибку в рабочей версии. С единственной линией истории это сложно.

**Ветка** даёт вам параллельную линию истории. Можно пробовать идею в одной ветке, хранить стабильную версию в другой и объединить их, когда идея готова. Ветки в Git так дёшевы, что их создают на каждую задачу.

## 2. Ветка — это просто указатель

Это ключевая мысль. Ветка — не копия ваших файлов. Это крошечный файл с **одним хешем коммита** внутри. Создайте репозиторий для практики:

```bash
cd ~/projects/git-playground
mkdir branch-demo
cd branch-demo
git init
echo "# Recipes" > README.md
git add README.md
git commit -m "Start recipes"
echo "Pancakes: flour, milk, eggs" > recipes.txt
git add recipes.txt
git commit -m "Add pancakes"
```

Теперь посмотрите на ветки и на файлы, где Git их хранит:

```bash
git branch
cat .git/HEAD
cat .git/refs/heads/main
```

```text
* main
ref: refs/heads/main
bcf3362ed20ba84d4a39cc0062c3aad9f0ed1bf1
```

**Что происходит:**

1. `git branch` перечисляет локальные ветки. Звёздочка `*` отмечает ветку, на которой вы находитесь.
2. `.git/HEAD` содержит `ref: refs/heads/main`: `HEAD` указывает на ветку `main`.
3. `.git/refs/heads/main` содержит хеш последнего коммита. Это и есть вся ветка: 40 символов.

```text
 Start recipes ◀── Add pancakes
                       ▲
                     main ◀── HEAD
```

Создайте вторую ветку:

```bash
git branch soup
git branch
git log --oneline --decorate
```

```text
* main
  soup
bcf3362 (HEAD -> main, soup) Add pancakes
d4c72b0 Start recipes
```

**Что происходит:** `git branch soup` создаёт новый указатель `soup` на тот же коммит. Ничего не копировалось. `HEAD -> main` показывает, что вы по-прежнему на `main`; `soup` указывает на тот же коммит.

```text
 Start recipes ◀── Add pancakes ◀── main ◀── HEAD
                                 ◀── soup
```

## 3. Переключение и коммиты в ветке

Используйте `git switch`, чтобы переместить `HEAD` на другую ветку:

```bash
git switch soup
cat .git/HEAD
echo "Soup: water, carrots, salt" >> recipes.txt
git commit -am "Add soup"
git log --oneline --decorate
```

```text
Switched to branch 'soup'
ref: refs/heads/soup
[soup c84d638] Add soup
 1 file changed, 1 insertion(+)
c84d638 (HEAD -> soup) Add soup
bcf3362 (main) Add pancakes
d4c72b0 Start recipes
```

**Что происходит:**

1. `git switch soup` перемещает `HEAD` на `soup` и обновляет ваши файлы в соответствии с ней.
2. Новый коммит добавлен в `soup`. Вперёд сдвинулся только указатель `soup`; `main` остался на месте.

```text
 Start recipes ◀── Add pancakes ◀── Add soup ◀── soup ◀── HEAD
                        ▲
                      main
```

Вернитесь и посмотрите, как файлы меняются на глазах:

```bash
git switch main
cat recipes.txt
git switch soup
cat recipes.txt
git switch main
```

```text
Pancakes: flour, milk, eggs
Pancakes: flour, milk, eggs
Soup: water, carrots, salt
```

**Что происходит:** на `main` в файле одна строка; на `soup` — две. Git переписывает файлы в вашем рабочем дереве при каждом переключении.

Чтобы создать ветку **и** сразу переключиться на неё, используйте `git switch -c <имя>` (`-c` = create, создать). В старых руководствах используют `git checkout -b <имя>`; это то же самое.

## 4. Слияние: случай fast-forward

Рецепт супа готов. Сольём `soup` в `main`: сначала переключитесь на ветку, которая должна **получить** работу, затем сольите в неё другую.

```bash
git switch main
git merge soup
git log --oneline --decorate
git branch -d soup
```

```text
Updating bcf3362..c84d638
Fast-forward
 recipes.txt | 1 +
 1 file changed, 1 insertion(+)
c84d638 (HEAD -> main, soup) Add soup
bcf3362 Add pancakes
d4c72b0 Start recipes
Deleted branch soup (was c84d638).
```

**Что происходит:**

1. Вы были на `main`. `git merge soup` обнаружил, что `soup` — это просто `main` плюс дополнительные коммиты.
2. Поэтому Git сделал **fast-forward**: передвинул указатель `main` вперёд на `c84d638`. Новый коммит не создавался.
3. `git branch -d soup` удаляет имя `soup`. Коммиты остаются в истории, потому что `main` до них достаёт. Удаление ветки удаляет только указатель.

```text
before:  A ◀── B ◀── main          after:  A ◀── B ◀── C ◀── main (HEAD)
                  ◀── C ◀── soup
```

## 5. Слияние: когда сдвинулись обе ветки (коммит слияния)

Теперь два человека (или две ваши задачи) одновременно меняют разные вещи. Создайте ветку `salad`, добавьте коммит, затем вернитесь на `main` и добавьте там **другой** коммит:

```bash
git switch -c salad
echo "Salad: tomato, cucumber" >> recipes.txt
git commit -am "Add salad"
git switch main
echo "All recipes are for two people." >> README.md
git commit -am "Add a note to README"
git log --oneline --graph --all --decorate
```

```text
Switched to a new branch 'salad'
[salad 6481c09] Add salad
 1 file changed, 1 insertion(+)
Switched to branch 'main'
[main de30e3f] Add a note to README
 1 file changed, 1 insertion(+)
* de30e3f (HEAD -> main) Add a note to README
| * 6481c09 (salad) Add salad
|/
* c84d638 Add soup
* bcf3362 Add pancakes
* d4c72b0 Start recipes
```

**Что происходит:** история **раздвоилась**. И у `main`, и у `salad` есть коммит, которого нет у другой. `--graph` рисует развилку, `--all` показывает все ветки, а не только текущую.

```text
                    ┌── Add salad ◀── salad
 … ◀── Add soup ◀───┤
                    └── Add a note to README ◀── main (HEAD)
```

Fast-forward теперь невозможен. Сливайте:

```bash
git merge salad --no-edit
git log --oneline --graph --all --decorate
```

```text
Merge made by the 'ort' strategy.
 recipes.txt | 1 +
 1 file changed, 1 insertion(+)
*   4a42176 (HEAD -> main) Merge branch 'salad'
|\
| * 6481c09 (salad) Add salad
* | de30e3f Add a note to README
|/
* c84d638 Add soup
* bcf3362 Add pancakes
* d4c72b0 Start recipes
```

**Что происходит:**

1. Git нашёл **общего предка** (`Add soup`) и сравнил его с обеими вершинами. Два изменения затрагивают разные файлы, поэтому Git объединил их сам. Это называется **трёхсторонним слиянием**.
2. Он создал **коммит слияния** `4a42176` с двумя родителями. `--no-edit` принимает сообщение по умолчанию «Merge branch 'salad'»; без него Git откроет редактор.
3. (`ort` — название текущего алгоритма слияния в Git. Сейчас он вам не нужен.)

Загляните внутрь коммита слияния:

```bash
git cat-file -p HEAD
```

```text
tree cdd1d66f2dba6e65632a193271076c8d2a41df37
parent de30e3f6e144c13e802e88369512d86fef9867d4
parent 6481c095ae50a818cb9e1d6d06ea01bb55a41527
author Ada Learner <ada@example.com> 1794915965 +0500
committer Ada Learner <ada@example.com> 1794915965 +0500

Merge branch 'salad'
```

Две строки `parent`: вершина `main` и вершина `salad`. Именно это и есть коммит слияния.

> 💡 **Совет:** а что, если одна и та же строка изменена в обеих ветках? Тогда Git не может решить и сообщает о **конфликте слияния**. Это нормально, и решать конфликты вы научитесь в [Уроке 01.7](07-merge-conflicts.md).

### Всегда создавать коммит слияния: `--no-ff`

Иногда команда хочет получать коммит слияния, даже когда возможен fast-forward, чтобы по истории было видно: «эта группа коммитов была одной новой возможностью».

```bash
git switch -c rice
echo "Rice: rice, water" >> recipes.txt
git commit -am "Add rice"
git switch main
git merge --no-ff rice -m "Merge branch 'rice'"
git log --oneline --graph -4
```

```text
Merge made by the 'ort' strategy.
 recipes.txt | 1 +
 1 file changed, 1 insertion(+)
*   e5be826 Merge branch 'rice'
|\
| * 8ecd99c Add rice
|/
*   4a42176 Merge branch 'salad'
|\
| * 6481c09 Add salad
```

В [Уроке 01.9](09-teamwork-pull-requests-and-conventions.md) вы увидите, как команды выбирают между коммитами слияния, squash и rebase.

## 6. Просмотр и удаление веток

| Команда | Что делает |
|---|---|
| `git branch` | перечисляет локальные ветки |
| `git branch -v` | добавляет последний коммит каждой ветки |
| `git branch --merged` | ветки, уже слитые в текущую (их безопасно удалять) |
| `git branch --no-merged` | ветки с работой, которая ещё не слита |
| `git branch -d <name>` | удаляет ветку, но **только если она слита** |
| `git branch -D <name>` | удаляет ветку **даже если она не слита** |
| `git branch -m <old> <new>` | переименовывает ветку |

```bash
git branch --merged
git branch -d salad
```

```text
* main
  salad
Deleted branch salad (was 6481c09).
```

Git защищает вас от потери неслитой работы:

```bash
git switch -c dessert
echo "Pie: apples, flour" >> recipes.txt
git commit -am "Add pie"
git switch main
git branch -d dessert
git branch --no-merged
git branch -D dessert
```

```text
error: the branch 'dessert' is not fully merged
hint: If you are sure you want to delete it, run 'git branch -D dessert'
  dessert
Deleted branch dessert (was 8b94729).
```

**Что происходит:** `-d` отказывается, потому что коммит `Add pie` недостижим из любой другой ветки: удаление указателя сделало бы этот коммит «потерянным». `-D` принуждает. Обратите внимание, что Git печатает хеш `8b94729`. Сохраните его: с этим хешем (или через `git reflog`) вы можете вернуть ветку командой `git branch dessert 8b94729`.

## 7. Переключение при незаконченной работе

Git меняет ваши файлы при переключении. Если у вас есть незакоммиченные правки в файле, который **отличается** в двух ветках, Git откажется, чтобы защитить ваши правки:

```bash
git switch -c tea
echo "Tea: water, tea leaves" >> recipes.txt
git commit -am "Add tea"
git switch main
echo "local edit" >> recipes.txt
git switch tea
```

```text
error: Your local changes to the following files would be overwritten by checkout:
	recipes.txt
Please commit your changes or stash them before you switch branches.
Aborting
```

**Что происходит:** у `recipes.txt` в `tea` другое содержимое, а у вас в нём несохранённая правка. Переключение перезаписало бы вашу правку. Есть два безопасных варианта:

1. **Закоммитить** правку (даже как «WIP»-коммит; WIP означает Work In Progress, работа в процессе), затем переключиться.
2. **Отложить** её через stash ([Урок 01.5](05-undoing-changes-safely.md)):

```bash
git stash
git switch tea
git switch main
git stash pop
```

```text
Saved working directory and index state WIP on main: e5be826 Merge branch 'rice'
Switched to branch 'tea'
Switched to branch 'main'
On branch main
Changes not staged for commit:
  (use "git add <file>..." to update what will be committed)
  (use "git restore <file>..." to discard changes in working directory)
	modified:   recipes.txt

no changes added to commit (use "git add" and/or "git commit -a")
Dropped refs/stash@{0} (85cba267b09db8662ee4be1c2d3a0b7186a9ae61)
```

Если изменённый файл в обеих ветках **одинаков**, Git позволит переключиться и перенесёт вашу правку с собой. Неотслеживаемые файлы тоже переносятся.

## 8. Переименование, быстрый возврат, правила имён

```bash
git restore recipes.txt
git branch -m tea drinks
git branch -vv
git switch drinks
git switch -
```

```text
  drinks 498de6a Add tea
* main   e5be826 Merge branch 'rice'
Switched to branch 'drinks'
Switched to branch 'main'
```

**Что происходит:** `git branch -m tea drinks` переименовывает ветку. `git switch -` возвращает на ветку, на которой вы были до этого (как «назад» в браузере).

В имени ветки нельзя использовать пробелы и некоторые специальные символы:

```bash
git branch "bad name"
```

```text
fatal: 'bad name' is not a valid branch name
hint: See 'git help check-ref-format'
```

📏 **Соглашение:** команды обычно называют ветки так:

- строчными буквами, слова через дефис: `fix-empty-input`;
- с префиксом, указывающим тип работы, через косую черту: `feature/login-page`, `fix/crash-on-start`, `docs/readme`, `chore/update-deps`;
- коротко, но осмысленно; номер задачи, если ваша команда ведёт задачи: `fix/1234-null-pointer`;
- без пробелов, без заглавных букв, без личных имён для общей работы.

Главная ветка называется `main`. В некоторых старых проектах всё ещё используют `master`.

## 9. Ветки в проекте на Rust

В проекте Cargo ветки работают так же. Попробуйте в `hello-rust`:

```bash
cd ~/projects/git-playground/hello-rust
git switch -c feature/greeting
sed -i '' 's/Hello, world!/Hello from a branch!/' src/main.rs
git commit -am "Change the greeting"
cargo run -q
git switch main
cargo run -q
git merge feature/greeting
cargo run -q
git branch -d feature/greeting
git log --oneline --graph --decorate
```

```text
Switched to a new branch 'feature/greeting'
[feature/greeting 9781128] Change the greeting
 1 file changed, 1 insertion(+), 1 deletion(-)
Hello from a branch!
Switched to branch 'main'
Hello, world!
Updating cf73604..9781128
Fast-forward
 src/main.rs | 2 +-
 1 file changed, 1 insertion(+), 1 deletion(-)
Hello from a branch!
Deleted branch feature/greeting (was 9781128).
* 9781128 (HEAD -> main) Change the greeting
* cf73604 Create the project with cargo new
```

**Что происходит:** программа печатает разный текст в каждой ветке, потому что `src/main.rs` различается. (`-q` делает Cargo тихим. `sed -i '' 's/old/new/' file` заменяет текст в файле на macOS; можно просто отредактировать файл в RustRover.) Папка `target/` игнорируется Git, поэтому она не переключается: Cargo пересобирает, когда исходники меняются.

> 💡 **Совет:** RustRover показывает текущую ветку справа внизу окна (и в виджете *Git*). Нажмите на неё, чтобы создавать ветки или переключаться без терминала.

## Частые ошибки

> ⚠️ **Частая ошибка:** коммит не в той ветке. Перед коммитом смотрите `git status` (первая строка) или приглашение командной строки или виджет IDE (Integrated Development Environment, интегрированной среды разработки). Если это уже случилось и вы не делали push: создайте новую ветку на текущем коммите (`git branch feature/x`), затем верните `main` назад (`git switch main`, `git reset --hard HEAD~1`). Используйте это осторожно ([Урок 01.5](05-undoing-changes-safely.md)).

> ⚠️ **Частая ошибка:** слияние не в ту сторону. `git merge A` сливает `A` **в ветку, на которой вы находитесь**. Всегда сначала переключайтесь на принимающую ветку (обычно `main`).

> ⚠️ **Частая ошибка:** `error: the branch 'x' is not fully merged`. В ветке есть коммиты, которых нет ни в одной другой. Сначала слейте её или используйте `-D`, только если действительно хотите выбросить работу.

> ⚠️ **Частая ошибка:** держать ветку живой неделями. Чем дольше живёт ветка, тем сильнее она отличается от `main` и тем сложнее слияние. Делайте ветки маленькими и короткоживущими и сливайте часто.

> ⚠️ **Частая ошибка:** коммитить в состоянии «отсоединённого `HEAD`» (после `git switch --detach` или после checkout хеша коммита). Такой коммит не принадлежит ни одной ветке, и его легко потерять. Если это случилось, сразу создайте ветку: `git switch -c rescue`.

## Попробуйте сами

1. В `branch-demo` создайте ветку `drinks`, добавьте в `recipes.txt` строку `Lemonade: water, lemon` и закоммитьте.
2. Переключитесь на `main`. Есть ли там строка про лимонад? Почему нет?
3. Выполните `git log --oneline --graph --all`. Куда указывает `drinks`?
4. Слейте `drinks` в `main`. Был ли это fast-forward? Проверьте по сообщению.
5. Удалите `drinks` с `-d`. Получилось? Почему?
6. Создайте `fix/typo` и `feature/menu` и закоммитьте в каждой разные файлы. Слейте обе в `main` и посмотрите `git log --oneline --graph`.
7. Откройте проект в RustRover и найдите виджет ветки справа внизу.

## Упражнения

### Упражнение 1 — Первая ветка (легко)

В `branch-demo` создайте ветку `notes`, добавьте файл `notes.txt` с одной строкой и закоммитьте его там. Докажите, что на `main` этого файла **нет**.

<details>
<summary>Подсказка</summary>

Используйте `git switch -c`, затем `git switch main` и `ls`.

</details>

<details>
<summary>Ответ</summary>

```bash
git switch -c notes
echo "Try the soup with bread." > notes.txt
git add notes.txt
git commit -m "Add notes.txt"
git switch main
ls
```

`ls` на `main` не показывает `notes.txt`.

Почему это работает: коммит существует только в ветке `notes`. Когда вы переключаетесь на `main`, Git переписывает рабочее дерево в соответствии с `main`, поэтому файл пропадает из папки (в коммите он в безопасности).

</details>

### Упражнение 2 — Предскажите слияние (легко)

Вы на `main` на коммите B. У ветки `feature` есть коммиты C и D поверх B. В `main` ничего не добавляли. Какое слияние получится при `git merge feature`? Сколько родителей у результата?

<details>
<summary>Подсказка</summary>

Содержит ли `feature` всё, что есть в `main`?

</details>

<details>
<summary>Ответ</summary>

**Fast-forward**. `main` просто сдвигается на D. Коммит слияния не создаётся, поэтому новых родителей нет: у D остаётся единственный родитель C.

Почему это работает: весь `main` (до B) уже содержится в `feature`, поэтому Git достаточно передвинуть указатель.

</details>

### Упражнение 3 — Создайте коммит слияния (средне)

Создайте две ветки `a` и `b` от `main`, в каждой — коммит с **разным** файлом (`a.txt`, `b.txt`). Слейте обе в `main`. Подсчитайте родителей последнего коммита слияния одной командой.

<details>
<summary>Подсказка</summary>

После слияния `a` (fast-forward) у `b` общий предок старше, чем `main`, поэтому нужен коммит слияния. `git cat-file -p HEAD` перечисляет родителей; `grep -c parent` считает строки.

</details>

<details>
<summary>Ответ</summary>

```bash
git switch -c a
echo "a" > a.txt
git add a.txt
git commit -m "Add a.txt"
git switch main
git switch -c b
echo "b" > b.txt
git add b.txt
git commit -m "Add b.txt"
git switch main
git merge a
git merge b --no-edit
git cat-file -p HEAD | grep -c parent
```

```text
2
```

Почему это работает: слияние `a` — fast-forward (в main не было новых коммитов). После этого `main` сдвинулся, а `b` всё ещё стартует от старого коммита, поэтому история раздвоилась, и Git делает коммит слияния с двумя родителями.

</details>

### Упражнение 4 — Верните удалённую ветку (средне)

Создайте ветку `dessert` с одним коммитом, удалите её с `-D` и верните **вместе с её коммитом**.

<details>
<summary>Подсказка</summary>

Git напечатал хеш при удалении ветки. Его также можно найти через `git reflog`. Затем `git branch <имя> <хеш>` создаёт ветку на этом коммите.

</details>

<details>
<summary>Ответ</summary>

```bash
git switch -c dessert
echo "Pie: apples, flour" >> recipes.txt
git commit -am "Add pie"
git switch main
git branch -D dessert
git reflog -3
git branch dessert 'HEAD@{1}'
git branch -v
```

Последняя команда снова показывает `dessert` на коммите `Add pie`.

Почему это работает: коммит никогда не удалялся; удалили только указатель. Reflog помнит, где был `HEAD`, поэтому `HEAD@{1}` (коммит `Add pie`) — допустимая стартовая точка для новой ветки.

</details>

### Упражнение 5 — Срочное исправление посреди работы (сложно)

Подготовка: выполните `git switch -c feature/long-work`, затем `echo "idea" >> recipes.txt` (не коммитьте). Вы посреди незаконченной правки в ветке `feature/long-work`. Опечатку нужно исправить в `main` **прямо сейчас**. Опишите и выполните команды: отложите свою работу, исправьте опечатку в ветке `fix/typo` от `main`, слейте её, вернитесь в свою фича-ветку с нетронутой правкой.

<details>
<summary>Подсказка</summary>

Используйте `git stash`, затем `git switch main`, `git switch -c fix/typo`, коммит, слияние, `git switch feature/long-work`, `git stash pop`.

</details>

<details>
<summary>Ответ</summary>

```bash
git stash
git switch main
git switch -c fix/typo
echo "typo fixed" >> README.md
git commit -am "Fix a typo"
git switch main
git merge fix/typo
git switch feature/long-work
git stash pop
git status -s
```

Итоговый `git status -s` показывает вашу незаконченную правку как изменённую, как и раньше.

Почему это работает: `stash` откладывает незаконченную правку, рабочее дерево становится чистым, и переключение разрешено. После слияния исправления `stash pop` возвращает правку поверх вашей фича-ветки.

</details>

## Вопросы для самопроверки

1. Что именно такое ветка в Git?

<details>
<summary>Ответ</summary>

Небольшой перемещаемый указатель (имя) на коммит. Он отмечает вершину линии работы. Это не копия файлов.

</details>

2. Что означает `HEAD -> main` в `git log --decorate`?

<details>
<summary>Ответ</summary>

`HEAD` (где вы находитесь) указывает на ветку `main`, то есть вы на `main`, и новые коммиты будут сдвигать `main`.

</details>

3. Когда слияние является fast-forward?

<details>
<summary>Ответ</summary>

Когда в текущей ветке нет коммитов, которых не хватает в другой, и Git достаточно сдвинуть текущий указатель вперёд. Коммит слияния не создаётся.

</details>

4. Сколько родителей у коммита слияния?

<details>
<summary>Ответ</summary>

Два (вершины двух слитых линий работы).

</details>

5. Почему `git branch -d` иногда отказывается и что делает `-D`?

<details>
<summary>Ответ</summary>

`-d` отказывается, если в ветке есть коммиты, не слитые больше нигде, потому что удаление указателя их потеряло бы. `-D` принудительно удаляет.

</details>

6. У вас есть несохранённая правка, и `git switch` отказывается. Какие два безопасных выхода?

<details>
<summary>Ответ</summary>

Закоммитить правку или отложить её (`git stash`), затем переключиться. Позже `git stash pop` вернёт правку.

</details>

## Итоги

- Ветка — это указатель на коммит; создание и удаление веток почти ничего не стоит.
- `HEAD` указывает на текущую ветку; коммиты двигают эту ветку вперёд.
- `git switch <имя>` переключает между ветками; `git switch -c <имя>` создаёт и переключает.
- `git merge <имя>` вносит `<имя>` в ветку, на которой вы находитесь: fast-forward, если возможно, иначе — коммит слияния с двумя родителями.
- `--no-ff` принуждает к коммиту слияния. `git log --oneline --graph --all` рисует историю.
- `git branch -d` удаляет только слитые ветки; `-D` принуждает. Удалённую ветку можно восстановить по хешу или через reflog.
- Используйте короткие имена строчными буквами с префиксами (`feature/…`, `fix/…`) и держите ветки маленькими и короткоживущими.

## Источники и что почитать

- [Pro Git — 3.1 Branches in a Nutshell](https://git-scm.com/book/en/v2/Git-Branching-Branches-in-a-Nutshell) — модель указателей с картинками.
- [Pro Git — 3.2 Basic Branching and Merging](https://git-scm.com/book/en/v2/Git-Branching-Basic-Branching-and-Merging) — тот же процесс в официальной книге.
- [Pro Git — 3.4 Branching Workflows](https://git-scm.com/book/en/v2/Git-Branching-Branching-Workflows) — долгоживущие и тематические ветки.
- [git-switch manual](https://git-scm.com/docs/git-switch) — все опции `switch`.
- [git-merge manual](https://git-scm.com/docs/git-merge) — стратегии и опции слияния.
- [Learn Git Branching](https://learngitbranching.js.org/) — интерактивный визуальный учебник в браузере (нужен интернет, по желанию).

---

[🏠 Начало](../START-HERE.md) · [← 01.5 Как безопасно отменять изменения](05-undoing-changes-safely.md) · [📋 Этап 01](README.md) · [01.7 Конфликты слияния →](07-merge-conflicts.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
