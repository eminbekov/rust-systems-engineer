<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 01.4 История и просмотр изменений](04-viewing-history-and-changes.md) · [📋 Этап 01](README.md) · [01.6 Ветки и слияние →](06-branches-and-merging.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/01-git-and-version-control/05-undoing-changes-safely.md) <!-- nav -->

# Урок 01.5 — Как безопасно отменять изменения

> **Этап 01 — Git и контроль версий** · Урок 5 из 10 · ⏱ около 2,5 часа (2 занятия)

## Что вы узнаете

- Какие ошибки Git умеет отменять, а какие нет, и золотые правила, которые берегут вашу работу.
- Как отбрасывать правки (`git restore`), убирать файлы из подготовленных и исправлять последний коммит (`git commit --amend`).
- Как работает `git reset` в трёх режимах и почему `--hard` опасен.
- Как отменить коммит, который уже есть у других людей, с помощью `git revert`.
- Как спасать «потерянные» коммиты через `git reflog`, как откладывать незаконченную работу с `git stash` и как удалять неотслеживаемые файлы через `git clean`.

## Перед началом

- Вы прошли [Урок 01.4 — История и просмотр изменений](04-viewing-history-and-changes.md). Вы умеете читать `git log`, `git diff` и `git status`.
- Вы работаете в `~/projects/git-playground`.

## Новые слова

| Слово | Значение |
|---|---|
| **restore** (восстановить) | Вернуть файл в состояние, которое он имел в каком-то коммите или в области подготовки. |
| **unstage** (убрать из подготовленных) | Забрать изменение из области подготовки, но оставить его в файле. |
| **amend** (исправить) | Заменить последний коммит новым, исправленным. |
| **reset** (сброс) | Переместить текущую ветку на другой коммит (и, при желании, изменить область подготовки и файлы). |
| **revert** (откат) | Создать **новый** коммит, который отменяет изменения более старого коммита. |
| **reflog** | Личный дневник того, где побывал `HEAD`. Позволяет найти коммиты, которых уже нет ни в одной ветке. |
| **stash** (тайник) | Полка, на которую откладывают незаконченные изменения, чтобы рабочее дерево стало чистым. |
| **WIP** (Work In Progress, работа в процессе) | Незаконченная работа. |
| **отсоединённый `HEAD`** (detached `HEAD`) | Состояние, когда `HEAD` указывает прямо на коммит, а не на имя ветки. |
| **разрушительная команда** (destructive command) | Команда, которая может навсегда удалить работу, не сохранённую в коммите. |

## 1. Золотые правила

Git отлично хранит то, что вы **закоммитили**. Но он не может защитить то, что вы **не** закоммитили.

| Ваша работа… | Может ли Git её вернуть? |
|---|---|
| закоммичена | **Да**, почти всегда, даже после «плохой» команды (см. `git reflog`). |
| подготовлена (`git add`, без коммита) | В основном нет. Содержимое области подготовки теряется, если его перезаписать. |
| изменена только в рабочем дереве | **Нет.** Если команда перезапишет файл, эти правки пропали. |

Поэтому правила такие:

1. **Коммитьте рано.** Коммит — страховочная сетка. Всегда можно сделать неаккуратный коммит сейчас и привести в порядок позже.
2. **Читайте, прежде чем запускать.** Перед любой командой «отмены» посмотрите `git status` и `git log --oneline`.
3. **Знайте уровень опасности.** Команды, способные уничтожить незакоммиченную работу: `git restore <file>`, `git reset --hard`, `git clean -f`, `git checkout -- <file>`. Все остальные команды этого урока безопасны.
4. **Никогда не переписывайте историю, которая уже есть у других людей.** `amend`, `reset` и (позже) `rebase` меняют хеши коммитов. Если у кого-то уже есть старые коммиты, вы создадите путаницу. Для общих коммитов используйте `git revert`.

## 2. Карта: «Я хочу…»

| Я хочу… | Команда | Опасность |
|---|---|---|
| отбросить свои правки в файле | `git restore <file>` | **уничтожает** правки |
| убрать файл из следующего коммита, сохранив правки | `git restore --staged <file>` | безопасно |
| исправить сообщение последнего коммита | `git commit --amend -m "new message"` | безопасно, если коммит не общий |
| добавить забытый файл в последний коммит | `git add <file>`, затем `git commit --amend --no-edit` | безопасно, если коммит не общий |
| отменить последний коммит, оставив изменения подготовленными | `git reset --soft HEAD~1` | безопасно |
| отменить последний коммит, оставив изменения неподготовленными | `git reset HEAD~1` (mixed) | безопасно |
| отменить последний коммит и удалить изменения | `git reset --hard HEAD~1` | **уничтожает** незакоммиченную работу |
| отменить коммит, который уже общий | `git revert <commit>` | безопасно |
| найти «потерянный» коммит | `git reflog` | безопасно |
| отложить незаконченную работу | `git stash` | безопасно |
| удалить неотслеживаемые файлы | `git clean -n` (просмотр), затем `git clean -f` | **уничтожает** файлы |
| получить одну старую версию файла | `git restore --source=<commit> <file>` | перезаписывает файл |

Репозиторий для практики:

```bash
cd ~/projects/git-playground
mkdir undo-demo
cd undo-demo
git init
echo "apples" > shopping.txt
git add shopping.txt
git commit -m "Start shopping list"
echo "bread" >> shopping.txt
git commit -am "Add bread"
```

## 3. Отбросить правки: `git restore <file>`

Вы изменили файл и решили, что правка была ошибкой.

```bash
echo "socks" >> shopping.txt
git status -s
git restore shopping.txt
git status -s
cat shopping.txt
```

```text
 M shopping.txt
apples
bread
```

**Что происходит:**

1. `echo "socks" >> …` добавляет неверную строку; `git status -s` показывает ` M` (изменён, не подготовлен).
2. `git restore shopping.txt` заменяет файл версией из области подготовки (она совпадает с последним коммитом, когда ничего не подготовлено). Правка исчезла.
3. Второй `git status -s` ничего не печатает: дерево чистое. `cat` показывает файл таким, каким он был.

> ⚠️ **Частая ошибка:** у `git restore <file>` **нет отмены**. Git не может вернуть правки, которые никогда не коммитились. Если не уверены, сначала выполните `git diff <file>`, чтобы увидеть, что вы потеряете, или сохраните копию (`cp file file.bak`).

## 4. Убрать из подготовленных: `git restore --staged <file>`

Вы по ошибке подготовили файл командой `git add`.

```bash
echo "call mum" > todo.txt
git add todo.txt
git status -s
git restore --staged todo.txt
git status -s
rm todo.txt
```

```text
A  todo.txt
?? todo.txt
```

**Что происходит:** `--staged` говорит `restore` работать с областью подготовки, а не с файлом. Файл убирается из следующего коммита, а его содержимое остаётся в рабочем дереве (он снова неотслеживаемый). Эта команда **безопасна**: ничего не теряется. Последняя строка, `rm todo.txt`, просто удаляет учебный файл, чтобы он не появлялся в выводе следующих разделов.

> 📏 **Соглашение:** в старых руководствах для этого используют `git reset HEAD <file>` (убрать из подготовленных) и `git checkout -- <file>` (отбросить правки). Они всё ещё работают, но `git restore` (Git 2.23+) понятнее, потому что каждая команда делает одно дело.

## 5. Исправить последний коммит: `git commit --amend`

### Опечатка в сообщении

```bash
echo "milk" >> shopping.txt
git commit -a -m "Add milq"
git commit --amend -m "Add milk"
git log --oneline
```

```text
[main f5b7bae] Add milq
 1 file changed, 1 insertion(+)
[main af70454] Add milk
 Date: Tue Nov 17 16:43:37 2026 +0500
 1 file changed, 1 insertion(+)
af70454 Add milk
19ccf86 Add bread
16d5fb3 Start shopping list
```

**Что происходит:** `--amend` не добавляет коммит. Он **заменяет** последний новым коммитом с новым сообщением. Посмотрите на хеш: он изменился с `f5b7bae` на `af70454`. Старый коммит какое-то время ещё живёт в reflog, но в истории `main` его больше нет.

### Забытый файл

```bash
echo "eggs" >> shopping.txt
echo "tea" > drinks.txt
git add shopping.txt
git commit -m "Add eggs and drinks"
git status -s
git add drinks.txt
git commit --amend --no-edit
git show --stat --oneline HEAD
```

```text
[main 04b0399] Add eggs and drinks
 1 file changed, 1 insertion(+)
?? drinks.txt
[main c5a1577] Add eggs and drinks
 Date: Tue Nov 17 16:43:37 2026 +0500
 2 files changed, 2 insertions(+)
 create mode 100644 drinks.txt
c5a1577 Add eggs and drinks
 drinks.txt   | 1 +
 shopping.txt | 1 +
 2 files changed, 2 insertions(+)
```

**Что происходит:** вы забыли `drinks.txt`. Вы подготавливаете его и выполняете `git commit --amend --no-edit`. `--no-edit` оставляет старое сообщение. Теперь новый коммит содержит оба файла.

> ⚠️ **Частая ошибка:** исправлять коммит, который вы уже **отправили** (push) в удалённый репозиторий ([Урок 01.8](08-remotes-github-and-gitlab.md)). Ваша локальная история теперь отличается от удалённой, и Git откажется выполнять обычный push. Исправляйте только коммиты, существующие лишь на вашем компьютере.

## 6. `git reset`: вернуть ветку назад

Имя ветки, например `main`, — это указатель на коммит. `git reset <commit>` перемещает указатель (а с ним и `HEAD`) на другой коммит. Три режима определяют, что произойдёт с областью подготовки и с вашими файлами:

| Режим | Двигает ветку | Сбрасывает область подготовки | Сбрасывает рабочие файлы |
|---|---|---|---|
| `--soft` | да | нет | нет |
| `--mixed` (по умолчанию) | да | **да** | нет |
| `--hard` | да | **да** | **да** (опасно!) |

Картина: вы по ошибке сделали коммит C и выполнили `git reset HEAD~1`.

```text
before:   A ◀── B ◀── C ◀── main (HEAD)
after:    A ◀── B ◀── main (HEAD)         C is no longer on any branch
                      └─ the changes of C are... where? That depends on the mode:
                           --soft:  in the staging area
                           --mixed: in the working files, unstaged
                           --hard:  gone
```

### `--soft`: переделать коммит

```bash
echo "cheese" >> shopping.txt
git commit -am "Add cheese too early"
git log --oneline -2
git reset --soft HEAD~1
git status -s
git log --oneline -2
```

```text
[main 3c6d189] Add cheese too early
 1 file changed, 1 insertion(+)
3c6d189 Add cheese too early
c5a1577 Add eggs and drinks
M  shopping.txt
c5a1577 Add eggs and drinks
af70454 Add milk
```

**Что происходит:** коммит исчез из истории, но `M ` показывает, что изменение всё ещё **подготовлено**. Теперь его можно закоммитить заново с лучшим сообщением или вместе с другими изменениями. `--soft` — это ещё и классический способ **склеить** (squash) несколько мелких коммитов в один: `git reset --soft HEAD~3`, затем один `git commit`.

### `--mixed` (по умолчанию): убрать из подготовленных, оставив правки

```bash
git commit -m "Add cheese"
git reset HEAD~1
git status -s
```

```text
Unstaged changes after reset:
M	shopping.txt
 M shopping.txt
```

**Что происходит:** коммит исчез, а изменение находится в вашем файле, но **не** подготовлено (` M`). Используйте это, когда хотите разделить один большой коммит на несколько.

### `--hard`: выбросить всё

```bash
git commit -am "Add cheese"
echo "rice" >> shopping.txt
git commit -am "Add rice (mistake)"
git log --oneline -3
git reset --hard HEAD~1
git status -s
cat shopping.txt
```

```text
a174f76 Add rice (mistake)
7f48478 Add cheese
c5a1577 Add eggs and drinks
HEAD is now at 7f48478 Add cheese
apples
bread
milk
eggs
cheese
```

**Что происходит:** `--hard` переместил ветку **и** вернул файл в состояние более раннего коммита. Строка `rice` из файла пропала. Поскольку она **была закоммичена**, её ещё можно вернуть (раздел 8). Если бы изменение не коммитилось, `--hard` уничтожил бы его навсегда.

> ⚠️ **Частая ошибка:** выполнять `git reset --hard`, когда у вас есть незакоммиченные правки. Эти правки не восстановить. Перед любым `--hard` выполните `git status`; если видите изменённые файлы, которые вам дороги, сначала закоммитьте их или отложите (stash).

## 7. `git revert`: отменить коммит, добавив новый

`reset` переписывает историю. Для коммита, который **уже есть у других**, историю переписывать нельзя. Вместо этого `git revert` создаёт **новый коммит** с противоположными изменениями.

```bash
echo "socks" >> shopping.txt
git commit -am "Add socks"
git revert HEAD --no-edit
git log --oneline -3
cat shopping.txt
```

```text
[main b3046a1] Revert "Add socks"
 Date: Tue Nov 17 16:43:37 2026 +0500
 1 file changed, 1 deletion(-)
b3046a1 Revert "Add socks"
733b11b Add socks
7f48478 Add cheese
apples
bread
milk
eggs
cheese
```

**Что происходит:** `git revert HEAD` создал новый коммит «Revert "Add socks"», который убирает строку `socks`. В истории остаются оба коммита, а файл вернулся в прежнее состояние. Ничья история не изменилась, поэтому это безопасно отправлять. `--no-edit` принимает автоматическое сообщение; без него Git откроет редактор. Можно откатить любой старый коммит: `git revert <hash>`.

```text
  A ◀── B ◀── C(socks) ◀── C'(revert socks)  ◀── main
```

## 8. `git reflog`: страховка под страховкой

Каждый раз, когда `HEAD` перемещается (коммит, reset, switch…), Git записывает строку в **reflog**. Это личный дневник на вашем компьютере. Представьте, что вы сбросили слишком далеко:

```bash
git reset --hard HEAD~3
git log --oneline
```

```text
HEAD is now at c5a1577 Add eggs and drinks
c5a1577 Add eggs and drinks
af70454 Add milk
19ccf86 Add bread
16d5fb3 Start shopping list
```

Коммиты `Add socks`, его откат и `Add cheese` исчезли из `git log`. Не паникуйте:

```bash
git reflog
```

```text
c5a1577 HEAD@{0}: reset: moving to HEAD~3
b3046a1 HEAD@{1}: revert: Revert "Add socks"
733b11b HEAD@{2}: commit: Add socks
7f48478 HEAD@{3}: reset: moving to HEAD~1
a174f76 HEAD@{4}: commit: Add rice (mistake)
7f48478 HEAD@{5}: commit: Add cheese
…
```

**Что происходит:** в каждой строке: хеш, `HEAD@{n}` (позиция в дневнике, 0 — сейчас) и что произошло. Строка `HEAD@{1}` — состояние **до** сброса. Вернитесь к нему:

```bash
git reset --hard 'HEAD@{1}'
git log --oneline -3
```

```text
HEAD is now at b3046a1 Revert "Add socks"
b3046a1 Revert "Add socks"
733b11b Add socks
7f48478 Add cheese
```

Всё вернулось. (Кавычки вокруг `HEAD@{1}` защищают фигурные скобки от оболочки.)

> 💡 **Совет:** по умолчанию Git хранит недостижимые коммиты в reflog не менее примерно 30 дней. Если вам кажется, что вы «потеряли коммит», выполните `git reflog` и поищите его сообщение. Поэтому *закоммиченную работу почти невозможно потерять*.

## 9. `git stash`: отложить работу

Вы посреди незаконченного изменения и должны переключиться на другое. Коммитить полуготовую работу не хочется. `git stash` сохраняет ваши незакоммиченные изменения на полку и очищает рабочее дерево.

```bash
echo "half-written idea" >> shopping.txt
git stash
git status -s
git stash list
git stash pop
git status -s
```

```text
Saved working directory and index state WIP on main: b3046a1 Revert "Add socks"
stash@{0}: WIP on main: b3046a1 Revert "Add socks"
On branch main
Changes not staged for commit:
  (use "git add <file>..." to update what will be committed)
  (use "git restore <file>..." to discard changes in working directory)
	modified:   shopping.txt

no changes added to commit (use "git add" and/or "git commit -a")
Dropped refs/stash@{0} (dd0bc310092fafbd34f43a33a59fa3e56b7bc33f)
 M shopping.txt
```

**Что происходит:**

1. `git stash` сохранил правку и вернул файл к состоянию последнего коммита. Первый `git status -s` ничего не напечатал: дерево чистое.
2. `git stash list` показывает полку: одна запись и коммит, на котором она сделана.
3. `git stash pop` возвращает изменения и удаляет запись с полки. Он также печатает полный отчёт `git status` (строки от `On branch main` до `no changes added…`) и строку `Dropped refs/stash@{0}`, которая подтверждает, что запись удалена. `git stash apply` возвращает изменения, но оставляет запись.
4. Последний `git status -s` показывает ` M shopping.txt`: недописанная правка снова в файле. (Раздел 11 ниже выбрасывает её с помощью `git restore`.)

Полезные дополнения: `git stash -u` откладывает и неотслеживаемые файлы; `git stash push -m "message"` даёт записи имя; `git stash drop` удаляет запись.

## 10. `git clean`: удалить неотслеживаемые файлы

`restore` не трогает неотслеживаемые файлы. `git clean` трогает, и он **разрушителен**, поэтому всегда сначала смотрите, что будет.

```bash
echo x > junk1.tmp
mkdir junkdir
echo y > junkdir/junk2.tmp
git clean -n
git clean -nd
git clean -fd
git status -s
```

```text
Would remove junk1.tmp
Would remove junk1.tmp
Would remove junkdir/
Removing junk1.tmp
Removing junkdir/
```

**Что происходит:**

1. `-n` означает *dry run* (пробный запуск): Git перечисляет, что удалил бы, и ничего не удаляет. По умолчанию папки он пропускает.
2. `-d` включает неотслеживаемые папки.
3. `-f` (force) реально удаляет. Git требует `-f` намеренно, чтобы избежать случайностей.

Файлы, подходящие под `.gitignore`, сохраняются, если не добавить `-x`. Не добавляйте `-x`, пока не поймёте, зачем.

## 11. Получить старую версию файла

Не обязательно возвращать во времени весь проект. Можно взять один файл из любого коммита.

```bash
git restore --source=HEAD~3 shopping.txt
cat shopping.txt
git diff
git restore shopping.txt
```

```text
apples
bread
milk
eggs
diff --git a/shopping.txt b/shopping.txt
index 541ae17..41fe824 100644
--- a/shopping.txt
+++ b/shopping.txt
@@ -2,4 +2,3 @@ apples
 bread
 milk
 eggs
-cheese
```

**Что происходит:** `--source=HEAD~3` берёт файл из коммита трёхшаговой давности и кладёт его в рабочее дерево как правку. `git diff` показывает отличие от текущего коммита. Сохраните старую версию, закоммитив её, или отбросьте командой `git restore shopping.txt`.

Чтобы только **посмотреть** на старый коммит, можно выполнить `git switch --detach HEAD~3`. Это переводит вас в состояние *отсоединённого `HEAD`*: вы видите старые файлы, но находитесь ни на какой ветке. Всё, что вы там закоммитите, легко потерять. Вернуться можно командой `git switch main`. С ветками по-настоящему вы познакомитесь в следующем уроке.

## Частые ошибки

> ⚠️ **Частая ошибка:** `error: pathspec 'nosuchfile.txt' did not match any file(s) known to git`. Опечатка в имени файла или файл не отслеживается. Проверьте через `git status` или `git ls-files`.

> ⚠️ **Частая ошибка:** `fatal: ambiguous argument 'HEAD~50': unknown revision or path not in the working tree.` Вы попросили уйти дальше назад, чем есть история. Посчитайте через `git log --oneline`.

> ⚠️ **Частая ошибка:** использовать `git reset --hard`, чтобы «исправить» конфликт или непонятную ситуацию. Сначала выполните `git status` и прочитайте вывод. У большинства ситуаций есть более безопасное решение.

> ⚠️ **Частая ошибка:** пытаться спрятать закоммиченный секрет, удалив файл и сделав новый коммит. Старый коммит по-прежнему его содержит. Считайте секрет украденным: замените его новым и отзовите старый. Переписывание истории для его удаления (инструментами вроде `git filter-repo`) — отдельный продвинутый шаг; если коммит уже на GitHub, надёжный способ — заменить секрет.

> 🔒 **Безопасность:** удалённый файл — это не удалённый секрет. История хранит всё. Используйте `.gitignore` для файлов с секретами с самого начала ([Урок 01.3](03-your-first-repository-and-commits.md)).

## Попробуйте сами

Используйте репозиторий `undo-demo`.

1. Добавьте в `shopping.txt` строку `lemons`. Выполните `git status -s`, затем отбросьте правку командой `git restore shopping.txt`. Проверьте через `cat`.
2. Создайте `scratch.txt`, подготовьте его, затем уберите из подготовленных. Удалите его.
3. Закоммитьте строку `lemons` с сообщением с опечаткой `Add lemmons`. Исправьте сообщение через `git commit --amend`. Сравните хеши до и после.
4. Сделайте коммит `Add jam`. Выполните `git reset --soft HEAD~1`, посмотрите `git status -s`, затем закоммитьте снова как `Add jam to the list`.
5. Сделайте коммит `Add oil`, затем отмените его командой `git revert HEAD --no-edit`. Выполните `git log --oneline` и объясните три коммита.
6. Выполните `git reset --hard HEAD~2`, затем восстановитесь через `git reflog` и `git reset --hard 'HEAD@{1}'`.
7. Внесите правку, отложите её (stash), убедитесь, что дерево чистое, затем выполните `git stash pop`.

## Упражнения

### Упражнение 1 — Безопасно отбросьте правку (легко)

Вы изменили `shopping.txt` и хотите выбросить изменение. Напишите команды, которые (а) показывают, что вы потеряете, затем (б) отбрасывают это, затем (в) доказывают, что дерево чистое.

<details>
<summary>Подсказка</summary>

Используйте `git diff`, `git restore` и `git status`.

</details>

<details>
<summary>Ответ</summary>

```bash
git diff shopping.txt
git restore shopping.txt
git status
```

Последняя команда печатает `nothing to commit, working tree clean`.

Почему это работает: `git diff` заранее показывает теряемые строки, `git restore` заменяет файл закоммиченной версией, а `git status` подтверждает результат.

</details>

### Упражнение 2 — Уберите из подготовленных, не потеряв правку (легко)

Вы выполнили `git add .` и заметили, что `.env` попал в подготовленные. Уберите из следующего коммита только `.env`, но оставьте файл на диске.

<details>
<summary>Подсказка</summary>

Используйте `restore` с опцией, которая работает с областью подготовки.

</details>

<details>
<summary>Ответ</summary>

```bash
git restore --staged .env
git status -s
```

```text
?? .env
```

Ещё стоит добавить `.env` в `.gitignore`.

Почему это работает: `--staged` затрагивает только область подготовки. Файл остаётся в вашей папке.

</details>

### Упражнение 3 — Дважды исправьте последний коммит (средне)

Сделайте коммит с сообщением `Add jelly`, содержащий только `shopping.txt`. Затем (а) исправьте сообщение на `Add jam` и (б) добавьте забытый файл `recipes.txt` в тот же коммит, не создавая нового. Покажите, что в журнале только один новый коммит.

<details>
<summary>Подсказка</summary>

Нужно два раза выполнить `git commit --amend`: один раз с `-m`, другой — с `--no-edit` после `git add`.

</details>

<details>
<summary>Ответ</summary>

```bash
echo "jelly" >> shopping.txt
echo "pancakes" > recipes.txt
git add shopping.txt
git commit -m "Add jelly"
git commit --amend -m "Add jam"
git add recipes.txt
git commit --amend --no-edit
git show --stat --oneline HEAD
git log --oneline -3
```

`git show --stat` перечисляет и `recipes.txt`, и `shopping.txt`, а `git log` показывает один коммит `Add jam` над прежней историей.

Почему это работает: `--amend` каждый раз заменяет последний коммит, а не добавляет новый.

</details>

### Упражнение 4 — Склейте два коммита в один (средне)

Сделайте два коммита `wip` и `wip 2`, каждый из которых добавляет строку в `notes.txt`, затем объедините их в **один** коммит `Add two lines to notes`. Использовать `rebase` нельзя.

<details>
<summary>Подсказка</summary>

`git reset --soft` возвращает ветку назад, но оставляет все изменения подготовленными.

</details>

<details>
<summary>Ответ</summary>

```bash
echo "one" > notes.txt
git add notes.txt
git commit -m "Add notes"
echo "two" >> notes.txt
git commit -am "wip"
echo "three" >> notes.txt
git commit -am "wip 2"
git reset --soft HEAD~2
git commit -m "Add two lines to notes"
git log --oneline
```

```text
7537e84 Add two lines to notes
f1904d8 Add notes
```

Почему это работает: `reset --soft HEAD~2` возвращает ветку на два коммита назад, но оставляет изменения обоих коммитов подготовленными, так что один коммит может сохранить их вместе. (В [Уроке 01.9](09-teamwork-pull-requests-and-conventions.md) вы сделаете то же самое с помощью интерактивного rebase.)

</details>

### Упражнение 5 — Восстановите «потерянный» коммит (сложно)

Продолжайте с упражнения 4: добавьте два коммита, `Add four` и `Add five`. Затем по ошибке выполните `git reset --hard HEAD~2`. Восстановите оба коммита и докажите это через `git log` и `cat notes.txt`.

<details>
<summary>Подсказка</summary>

Выполните `git reflog`. Запись прямо перед `reset: moving to HEAD~2` — это нужное вам состояние, `HEAD@{1}`.

</details>

<details>
<summary>Ответ</summary>

```bash
echo "four" >> notes.txt
git commit -am "Add four"
echo "five" >> notes.txt
git commit -am "Add five"
git reset --hard HEAD~2
git reflog -4
git reset --hard 'HEAD@{1}'
git log --oneline
cat notes.txt
```

```text
e9a1d23 Add five
4126c8f Add four
7537e84 Add two lines to notes
f1904d8 Add notes
one
two
three
four
five
```

Почему это работает: reflog помнит, где был `HEAD` до сброса. `git reset --hard 'HEAD@{1}'` возвращает ветку на тот коммит и восстанавливает файлы.

</details>

## Вопросы для самопроверки

1. Какую работу Git **не** сможет вернуть после `git restore <file>`?

<details>
<summary>Ответ</summary>

Правки, которые никогда не подготавливались и не коммитились. У Git нет их копии.

</details>

2. Чем отличаются `git reset --soft`, `--mixed` и `--hard`?

<details>
<summary>Ответ</summary>

Все три двигают ветку. `--soft` оставляет область подготовки и файлы как есть. `--mixed` дополнительно сбрасывает область подготовки, но оставляет файлы. `--hard` сбрасывает область подготовки **и** файлы.

</details>

3. Когда использовать `git revert` вместо `git reset`?

<details>
<summary>Ответ</summary>

Когда коммит уже общий (отправлен). `revert` добавляет новый коммит и не переписывает историю.

</details>

4. Вы выполнили `git reset --hard` и потеряли коммит. Как его найти?

<details>
<summary>Ответ</summary>

Выполните `git reflog`, найдите строку этого коммита и используйте `git reset --hard <hash>` или `git switch -c rescue <hash>`, чтобы вернуть его.

</details>

5. Что `git commit --amend` делает с хешем коммита?

<details>
<summary>Ответ</summary>

Создаёт новый коммит, поэтому хеш меняется. Вот почему нельзя исправлять коммиты, которые уже есть у других людей.

</details>

6. Как безопасно посмотреть, что сделает `git clean`?

<details>
<summary>Ответ</summary>

`git clean -n` (с `-d` для папок). Он перечисляет, что было бы удалено, и ничего не удаляет.

</details>

## Итоги

- Закоммиченная работа почти никогда не теряется; незакоммиченная может. Коммитьте рано и сначала читайте `git status`.
- `git restore <file>` отбрасывает правки (разрушительно); `git restore --staged <file>` убирает из подготовленных (безопасно).
- `git commit --amend` исправляет последний коммит; используйте его только до push.
- `git reset` двигает ветку: `--soft` оставляет изменения подготовленными, `--mixed` — неподготовленными, `--hard` удаляет их.
- `git revert` отменяет коммит, добавляя противоположный; это правильный инструмент для общей истории.
- `git reflog` показывает, где побывал `HEAD`, и спасает «потерянные» коммиты.
- `git stash` откладывает незаконченную работу; `git clean -n` показывает, `git clean -f` удаляет неотслеживаемые файлы.

## Источники и что почитать

- [Pro Git — 2.4 Undoing Things](https://git-scm.com/book/en/v2/Git-Basics-Undoing-Things) — amend, unstage и restore в официальной книге.
- [Pro Git — 7.7 Reset Demystified](https://git-scm.com/book/en/v2/Git-Tools-Reset-Demystified) — глубокое объяснение трёх деревьев и трёх режимов reset.
- [Pro Git — 7.3 Stashing and Cleaning](https://git-scm.com/book/en/v2/Git-Tools-Stashing-and-Cleaning) — `stash` и `clean` подробно.
- [git-restore manual](https://git-scm.com/docs/git-restore) — точные опции `restore`.
- [git-reflog manual](https://git-scm.com/docs/git-reflog) — как работает reflog и как долго он хранит записи.
- [GitHub Docs — Removing sensitive data from a repository](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/removing-sensitive-data-from-a-repository) — что делать, если секрет был закоммичен.

---

[🏠 Начало](../START-HERE.md) · [← 01.4 История и просмотр изменений](04-viewing-history-and-changes.md) · [📋 Этап 01](README.md) · [01.6 Ветки и слияние →](06-branches-and-merging.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
