<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 01.6 Ветки и слияние](06-branches-and-merging.md) · [📋 Этап 01](README.md) · [01.8 Удалённые репозитории: GitHub и GitLab →](08-remotes-github-and-gitlab.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/01-git-and-version-control/07-merge-conflicts.md) <!-- nav -->

# Урок 01.7 — Конфликты слияния

> **Этап 01 — Git и контроль версий** · Урок 7 из 10 · ⏱ около 120 минут

## Что вы узнаете

- Почему возникают конфликты слияния и почему они нормальны, а не катастрофа.
- Как читать маркеры конфликта `<<<<<<<`, `|||||||`, `=======` и `>>>>>>>`.
- Точные шаги, чтобы решить конфликт и завершить слияние, и как отменить слияние.
- Как выбрать одну сторону целиком, как справиться со случаем «изменён здесь, удалён там» и что делать с конфликтами в `Cargo.lock`.
- Как сделать конфликты более редкими.

## Перед началом

- Вы прошли [Урок 01.6 — Ветки и слияние](06-branches-and-merging.md).
- Вы задали `merge.conflictStyle zdiff3` в [Уроке 01.2](02-install-and-configure-git.md). (Если нет, в маркерах ниже будет на одну часть меньше. Всё остальное то же самое.)

## Новые слова

| Слово | Значение |
|---|---|
| **конфликт слияния** (merge conflict) | Ситуация, когда Git не может объединить два изменения сам, потому что обе ветки изменили одни и те же строки (или одна изменила файл, который другая удалила). |
| **маркеры конфликта** (conflict markers) | Специальные строки (`<<<<<<<`, `|||||||`, `=======`, `>>>>>>>`), которые Git записывает в файл, чтобы показать конкурирующие версии. |
| **ours** («наша») | При слиянии: версия из ветки, на которой вы находитесь (принимающей ветки). |
| **theirs** («их») | При слиянии: версия из ветки, которую вы сливаете. |
| **base** («база») | Версия в общем предке: как выглядел текст до того, как любая из сторон его изменила. |
| **неслитый путь** (unmerged path) | Файл с нерешённым конфликтом. |
| **решить** (resolve) | Отредактировать файл до окончательного вида, убрать маркеры и сообщить Git командой `git add`. |
| **инструмент слияния** (merge tool) | Программа с визуальным представлением конфликта (в RustRover он есть). |

## 1. Почему возникают конфликты

В [Уроке 01.6](06-branches-and-merging.md) Git слил две ветки сам, потому что они меняли **разные** файлы. Git сравнивает вершины обеих веток с общим предком:

- Если строку изменила только одна сторона, Git берёт это изменение.
- Если **обе** стороны изменили **одно и то же место** по-разному, Git не может знать, что верно. Он останавливается и спрашивает вас.

Конфликт — это и есть вопрос от Git. Это не ошибка, и ничего не потеряно. Разработчики в активных командах сталкиваются с конфликтами каждую неделю.

## 2. Создайте конфликт

Сделайте репозиторий, где две ветки меняют одну и ту же строку. Вставьте блок (это просто подготовка):

```bash
cd ~/projects/git-playground
mkdir conflict-demo
cd conflict-demo
git init
printf 'fn main() {\n    println!("Welcome");\n    println!("Have a nice day");\n}\n' > main.rs
git add main.rs
git commit -m "Add main.rs"

git switch -c polite
printf 'fn main() {\n    println!("Welcome, dear guest");\n    println!("Have a nice day");\n}\n' > main.rs
git commit -am "Make the welcome polite"

git switch main
printf 'fn main() {\n    println!("Welcome to Rust");\n    println!("Have a nice day");\n}\n' > main.rs
git commit -am "Mention Rust in the welcome"
```

**Что происходит:** в первом коммите есть строка `Welcome`. В ветке `polite` она превращается в `Welcome, dear guest`. В `main` **та же строка** превращается в `Welcome to Rust`. (Код на Rust здесь — просто текст; вы изучите его на Этапе 02.)

```text
              ┌── Make the welcome polite ◀── polite
 Add main.rs ◀┤
              └── Mention Rust in the welcome ◀── main (HEAD)
```

Теперь попробуйте слить:

```bash
git merge polite
```

```text
Auto-merging main.rs
CONFLICT (content): Merge conflict in main.rs
Automatic merge failed; fix conflicts and then commit the result.
```

**Что происходит:** Git попытался объединить файлы и споткнулся на `main.rs`. Слияние теперь **в процессе**: оно приостановлено и ждёт вас. Ничего не сломано. Посмотрите на ситуацию:

```bash
git status
```

```text
On branch main
You have unmerged paths.
  (fix conflicts and run "git commit")
  (use "git merge --abort" to abort the merge)

Unmerged paths:
  (use "git add <file>..." to mark resolution)
	both modified:   main.rs

no changes added to commit (use "git add" and/or "git commit -a")
```

Короткая форма показывает `UU main.rs` (`UU` = не слито с обеих сторон). `git status` всегда подсказывает два выхода: исправить и закоммитить или отменить.

## 3. Читаем маркеры

Git записал в файл обе версии. Откройте его:

```bash
cat main.rs
```

```text
fn main() {
<<<<<<< HEAD
    println!("Welcome to Rust");
||||||| 65e17d3
    println!("Welcome");
=======
    println!("Welcome, dear guest");
>>>>>>> polite
    println!("Have a nice day");
}
```

В блоке конфликта до четырёх частей:

| Маркер | Что идёт дальше |
|---|---|
| `<<<<<<< HEAD` | **Ours**: строки из вашей текущей ветки (`HEAD` — то, где вы находитесь, `main`). |
| `\|\|\|\|\|\|\| 65e17d3` | **Base**: строки в общем предке (коммит `65e17d3`). Только с `zdiff3`. |
| `=======` | Разделитель. |
| (строки после разделителя) | **Theirs**: строки из ветки, которую вы сливаете (`polite`). |
| `>>>>>>> polite` | Конец блока с именем другой ветки. |

Всё **вне** маркеров (`fn main() {`, `Have a nice day`, `}`) слилось без проблем.

Читаем: исходная строка была `Welcome`. С вашей стороны она стала `Welcome to Rust`. С их стороны — `Welcome, dear guest`. База — ключ: она позволяет увидеть, **что изменила каждая сторона**. Без `zdiff3` Git опускает базу, и вам приходится угадывать.

> 📏 **Соглашение:** средняя часть (база) делает конфликты гораздо понятнее. Поэтому Урок 01.2 задаёт `merge.conflictStyle zdiff3`.

## 4. Решаем в пять шагов

1. **Поймите**, чего хотела каждая сторона. Здесь: одна хотела «to Rust», другая — «dear guest».
2. **Решите**, каким будет итоговый текст. Это может быть одна сторона, другая, обе или что-то новое. Здесь нам нужны обе идеи: `Welcome to Rust, dear guest`.
3. **Отредактируйте файл** до итогового текста. **Удалите маркеры и отвергнутые строки.** Используйте RustRover или любой редактор. В терминале:

```bash
printf 'fn main() {\n    println!("Welcome to Rust, dear guest");\n    println!("Have a nice day");\n}\n' > main.rs
git diff --check
```

`git diff --check` ничего не печатает: маркеров не осталось. (До исправления он печатал `leftover conflict marker` для каждой строки-маркера.)

4. **Сообщите Git**, что файл решён, подготовив его:

```bash
git add main.rs
git status
```

```text
On branch main
All conflicts fixed but you are still merging.
  (use "git commit" to conclude merge)

Changes to be committed:
	modified:   main.rs

```

5. **Завершите** слияние коммитом:

```bash
git commit --no-edit
git log --oneline --graph
```

```text
[main b37cecc] Merge branch 'polite'
*   b37cecc Merge branch 'polite'
|\
| * a27ff71 Make the welcome polite
* | 8b15433 Mention Rust in the welcome
|/
* 65e17d3 Add main.rs
```

**Что происходит:** `git add` означает «этот файл теперь решён». `git commit` создаёт коммит слияния с двумя родителями. `--no-edit` оставляет сообщение по умолчанию `Merge branch 'polite'`. (`git merge --continue` делает то же самое.) Готово.

> ⚠️ **Частая ошибка:** закоммитить файл, в котором ещё остались маркеры. Git **не проверяет** содержимое: если выполнить `git add` с маркерами внутри, они попадут в коммит. Тогда компилятор Rust пожалуется, как здесь:

```text
error: encountered diff marker
 --> main.rs:2:1
  |
2 | <<<<<<< HEAD
  | ^^^^^^^ between this marker and `=======` is the code that you are merging into
3 |     println!("Welcome to Rust");
4 | =======
  | ------- between this marker and `>>>>>>>` is the incoming code
5 |     println!("Welcome, dear guest");
6 | >>>>>>> polite
  | ^^^^^^^ this marker concludes the conflict region
  |
  = note: conflict markers indicate that a merge was started but could not be completed due to merge conflicts
          to resolve a conflict, keep only the code you want and then delete the lines containing conflict markers
…
error: aborting due to 1 previous error
```

Это настоящее сообщение компилятора (`rustc`). Оно означает: вы забыли удалить маркеры. Перед `git add` выполните `git diff --check` или поищите `<<<<<<<` по проекту.

## 5. Отмена: вернуться в состояние до слияния

Если конфликт слишком запутанный или вы начали слияние по ошибке, отмените его. Всё вернётся в состояние до `git merge`.

> 💡 **Совет:** чтобы попробовать разделы 5 и 6, нужно слияние, которое *в процессе*. Если вы уже завершили слияние в разделе 4, отмените коммит слияния командой `git reset --hard HEAD~1` (здесь это безопасно, потому что коммит слияния — последний и больше ничего не осталось незакоммиченным) и снова выполните `git merge polite`.

Теперь отмените слияние:

```bash
git merge --abort
git status -s
```

(Ничего не печатается: рабочее дерево чистое.) Позже слияние можно начать снова. Это кнопка «отмена» для слияния с конфликтами.

## 6. Взять одну сторону целиком

Иногда нужна одна версия полностью. Во время слияния:

- **ours** = ветка, на которой вы находитесь (`main`);
- **theirs** = ветка, которую вы сливаете (`polite`).

```bash
git merge polite
git checkout --ours main.rs
cat main.rs
git checkout --theirs main.rs
cat main.rs
git status -s
```

```text
Updated 1 path from the index
fn main() {
    println!("Welcome to Rust");
    println!("Have a nice day");
}
Updated 1 path from the index
fn main() {
    println!("Welcome, dear guest");
    println!("Have a nice day");
}
UU main.rs
```

**Что происходит:** `git checkout --ours <file>` заменяет файл вашей стороной, `--theirs` — другой, и обе команды **убирают маркеры**. Статус всё ещё `UU`, потому что вы ещё не сообщили Git. После выбора выполните `git add main.rs` и `git commit --no-edit`, как раньше. (`git restore --ours main.rs` — новая запись той же идеи.)

> ⚠️ **Частая ошибка:** при *rebase* (Урок 01.9) значения «ours» и «theirs» **поменяны местами** по сравнению со слиянием. Если вы ими пользуетесь, перепроверьте результат через `cat`.

## 7. Другие виды конфликтов: изменён здесь, удалён там

Если одна ветка изменила файл, а другая удалила его, Git не может выбрать ни то ни другое:

```text
CONFLICT (modify/delete): main.rs deleted in cleanup and modified in HEAD.  Version HEAD of main.rs left in tree.
Automatic merge failed; fix conflicts and then commit the result.
```

```text
$ git status -s
UD main.rs
```

`UD` означает «не слито: удалён ими». Файл остаётся в вашей папке. Решаете вы:

- **Оставить файл:** `git add main.rs`.
- **Удалить файл:** `git rm main.rs`.

Затем `git commit --no-edit` завершает слияние. Похожие сообщения бывают для файла, который обе ветки **добавили** с разным содержимым (`add/add`), и для переименований. Рецепт тот же: поймите, решите, `git add` или `git rm`, коммит.

## 8. Конфликты в генерируемых файлах, например `Cargo.lock`

`Cargo.lock` — файл, который пишет Cargo. Две ветки, добавляющие зависимости, могут обе изменить его, и конфликт внутри читать трудно. **Не** правьте его вручную. Распространённый безопасный рецепт:

1. Сначала вручную решите `Cargo.toml` (файл, написанный человеком).
2. Возьмите любую версию файла блокировок (`git checkout --ours Cargo.lock`).
3. Выполните `cargo build`. Cargo обновит `Cargo.lock` в соответствии со слитым `Cargo.toml`.
4. `git add Cargo.lock Cargo.toml` и коммит.

Та же идея работает для любого генерируемого файла: решайте **исходник**, перегенерируйте генерируемый файл, никогда не сливайте его вручную.

## 9. Графический способ: RustRover

В RustRover есть инструмент слияния. Когда у слияния есть конфликты, диалог перечисляет файлы (или откройте *Git → Resolve Conflicts*). Для каждого файла выберите *Merge…* и увидите три колонки: ваша версия (слева), результат (посередине) и их версия (справа). Маленькие стрелки `»` и `«` принимают изменение одной из сторон; `✕` игнорирует его. Когда результат верен, нажмите *Apply*. RustRover сам подготовит файл. Завершить всё равно нужно коммитом.

Он делает то же самое, что и текстовые шаги выше. Сначала выучите текстовый способ: он работает везде, в том числе по SSH (Secure Shell, способ работать в терминале удалённого компьютера) на сервере.

## 10. Как сделать конфликты более редкими

Убрать конфликты нельзя, но можно уменьшить их число:

- **Маленькие недолговечные ветки.** Чем меньше две ветки живут порознь, тем меньше шанс, что они заденут одни и те же строки.
- **Интегрируйте часто.** Регулярно сливайте `main` в свою ветку, чтобы различия оставались небольшими.
- **Маленькие целенаправленные коммиты и файлы.** Не смешивайте переформатирование с настоящими изменениями в одном коммите. Если без переформатирования не обойтись, делайте его отдельно и быстро.
- **Пользуйтесь форматтером.** `cargo fmt` даёт всем одинаковую раскладку, поэтому конфликты из-за пробелов и переносов строк исчезают.
- **Общайтесь с командой.** Если двое должны править одну функцию, договоритесь, кто идёт первым.

## Частые ошибки

> ⚠️ **Частая ошибка:** удалить не те строки и потерять чужую работу. Прочитайте блок: **обе** стороны могут содержать нужный вам код. Если не уверены, оставьте обе, соберите и протестируйте.

> ⚠️ **Частая ошибка:** забыть выполнить `git add` после исправления файла. Тогда `git commit` говорит `error: Committing is not possible because you have unmerged files.` Сначала подготовьте файл.

> ⚠️ **Частая ошибка:** запаниковать и выполнить `git reset --hard`. Для слияния в процессе это сработало бы, но `git merge --abort` — безопасный инструмент для этой задачи.

> ⚠️ **Частая ошибка:** решить конфликт текстово, но программа перестала компилироваться или тесты падают. Решённый файл всё равно должен быть корректным. Соберите и протестируйте (`cargo build`, `cargo test`) **до** того, как закоммитите слияние.

> 🧪 **Тестирование:** после решения конфликта запустите тесты проекта. Слияние, которое применилось «чисто», всё равно может быть логически неверным; у решённого конфликта такой шанс ещё выше.

## Попробуйте сами

1. Выполните блок подготовки из раздела 2 в новой папке. Используйте имя `conflict-demo2` вместо `conflict-demo`, чтобы старая папка для практики осталась нетронутой.
2. Выполните `git merge polite` и посмотрите `git status`.
3. Откройте `main.rs` и найдите четыре строки-маркера.
4. Выполните `git merge --abort` и проверьте `cat main.rs` (файл вернулся к версии из `main`).
5. Слейте снова. На этот раз решите конфликт до `Welcome to Rust, dear guest`, выполните `git diff --check`, `git add`, `git commit --no-edit`.
6. Выполните `git log --oneline --graph` и найдите коммит слияния.

## Упражнения

### Упражнение 1 — Прочитайте блок (легко)

Этот блок оказался в `README.md` после слияния. Что было в исходном тексте, что написали «ours» и что написали «theirs»? Напишите итоговый файл, если хотите сохранить **оба** предложения.

```text
# Project
<<<<<<< HEAD
This tool is fast.
||||||| a1b2c3d
This tool works.
=======
This tool is safe.
>>>>>>> feature/safety
```

<details>
<summary>Подсказка</summary>

Часть между `|||||||` и `=======` — это база: текст до того, как любая сторона его изменила.

</details>

<details>
<summary>Ответ</summary>

- Исходный текст (base): `This tool works.`
- Ours (`HEAD`): `This tool is fast.`
- Theirs (`feature/safety`): `This tool is safe.`

Итоговый файл, сохраняющий обе идеи:

```text
# Project
This tool is fast and safe.
```

(Две строки, `This tool is fast.` и `This tool is safe.`, тоже верный вариант. Важно, чтобы маркеров и базы не осталось, а текст был тем, который вы хотите.)

Почему это работает: оба автора заменили одну и ту же строку, поэтому Git не может выбрать. Вы объединяете их намерения вручную.

</details>

### Упражнение 2 — Решите и завершите (легко)

Создайте конфликт из раздела 2 и решите его, оставив текст `Welcome to Rust, dear guest`. Закончите чистым `git status`.

<details>
<summary>Подсказка</summary>

Отредактируйте файл (или перепишите его через `printf`), затем `git add`, затем `git commit --no-edit`.

</details>

<details>
<summary>Ответ</summary>

```bash
git merge polite
printf 'fn main() {\n    println!("Welcome to Rust, dear guest");\n    println!("Have a nice day");\n}\n' > main.rs
git diff --check
git add main.rs
git commit --no-edit
git status
```

```text
On branch main
nothing to commit, working tree clean
```

Почему это работает: после команды слияния репозиторий находится в состоянии «слияние». Редактирование файла, `git add` и `git commit` завершают слияние.

</details>

### Упражнение 3 — Отмените и повторите (средне)

Начните конфликтное слияние, затем передумайте и отмените его. Докажите, что ничего не изменилось: ветка, файл и `git status`. Затем снова выполните слияние. Перед началом убедитесь, что `polite` ещё не слита в `main` и что никакое слияние не идёт: если `git log --oneline -1` показывает `Merge branch 'polite'`, выполните `git reset --hard HEAD~1`; если `git status` пишет `You have unmerged paths`, выполните `git merge --abort`.

<details>
<summary>Подсказка</summary>

`git merge --abort`. Сравните `cat main.rs` и `git log --oneline` до и после.

</details>

<details>
<summary>Ответ</summary>

```bash
git merge polite
git merge --abort
git status
cat main.rs
git merge polite
```

```text
On branch main
nothing to commit, working tree clean
```

`cat main.rs` показывает версию `Welcome to Rust`, в точности как до первого слияния.

Почему это работает: `--abort` восстанавливает состояние до начала слияния, поэтому можно повторять сколько угодно.

</details>

### Упражнение 4 — Возьмите другую сторону целиком (средне)

Снова слейте `polite` в `main`, но вместо редактирования файла оставьте **их** версию всего файла. Завершите слияние и выведите результат. Перед началом убедитесь, что `polite` ещё не слита в `main` и что никакое слияние не идёт: если `git log --oneline -1` показывает `Merge branch 'polite'`, выполните `git reset --hard HEAD~1`; если `git status` пишет `You have unmerged paths`, выполните `git merge --abort`.

<details>
<summary>Подсказка</summary>

`git checkout --theirs <file>`, затем `git add`, затем `git commit --no-edit`.

</details>

<details>
<summary>Ответ</summary>

```bash
git merge polite
git checkout --theirs main.rs
git add main.rs
git commit --no-edit
cat main.rs
```

```text
fn main() {
    println!("Welcome, dear guest");
    println!("Have a nice day");
}
```

Почему это работает: `--theirs` берёт файл таким, каким он в ветке, которую вы сливали. Поскольку файл целый и без маркеров, его можно сразу подготовить.

</details>

### Упражнение 5 — Изменён здесь, удалён там (сложно)

В новом репозитории с `main.rs` создайте ветку `cleanup`, которая удаляет `main.rs`, и коммит в `main`, который его редактирует. Слейте и решите двумя способами: сначала оставьте файл, затем (после отмены и повтора) удалите его. Создайте новый репозиторий рядом с `conflict-demo`, а не внутри него: репозиторий внутри другого репозитория только запутывает.

<details>
<summary>Подсказка</summary>

Статус будет `UD`. Используйте `git add main.rs`, чтобы оставить файл, или `git rm main.rs`, чтобы удалить.

</details>

<details>
<summary>Ответ</summary>

```bash
cd ~/projects/git-playground
git init -b main modify-delete
cd modify-delete
printf 'fn main() {}\n' > main.rs
git add main.rs
git commit -m "Add main.rs"
git switch -c cleanup
git rm main.rs
git commit -m "Remove main.rs"
git switch main
echo "// edited" >> main.rs
git commit -am "Edit main.rs"
git merge cleanup
git status -s
git add main.rs
git commit --no-edit
ls
```

Слияние печатает `CONFLICT (modify/delete)`, а `git status -s` показывает `UD main.rs`. После `git add` коммит слияния сохраняет файл, и `ls` показывает `main.rs`.

Чтобы вместо этого удалить файл, выполните `git merge --abort`, повторите `git merge cleanup`, затем используйте `git rm main.rs` вместо `git add main.rs` и `git commit --no-edit`. После этого `ls` файл не показывает.

Почему это работает: Git не может определить, что должно победить — правка или удаление, поэтому вы выбираете: подготовкой (`git add`, оставить) или удалением (`git rm`, удалить).

</details>

## Вопросы для самопроверки

1. Когда Git сообщает о конфликте слияния?

<details>
<summary>Ответ</summary>

Когда обе ветки по-разному изменили одни и те же строки или когда одна ветка изменила файл, который другая удалила. Git не может выбрать, поэтому спрашивает вас.

</details>

2. Что отмечают `<<<<<<<`, `=======` и `>>>>>>>`?

<details>
<summary>Ответ</summary>

`<<<<<<<` начинает блок и показывает вашу версию (ours), `=======` разделяет вашу версию и их, `>>>>>>>` заканчивает блок и называет другую ветку.

</details>

3. Что за дополнительная секция `|||||||`?

<details>
<summary>Ответ</summary>

База: текст в общем предке до того, как любая сторона его изменила. Она помогает увидеть, что на самом деле изменила каждая сторона. Появляется при `merge.conflictStyle zdiff3`.

</details>

4. Какие две команды завершают слияние после правки файла с конфликтом?

<details>
<summary>Ответ</summary>

`git add <file>` (пометить как решённый) и `git commit` (или `git merge --continue`).

</details>

5. Как отменить слияние с конфликтами?

<details>
<summary>Ответ</summary>

`git merge --abort`.

</details>

6. Как поступить с конфликтом в `Cargo.lock`?

<details>
<summary>Ответ</summary>

Не править его вручную. Решите `Cargo.toml`, возьмите любую версию `Cargo.lock`, выполните `cargo build`, чтобы перегенерировать его, затем подготовьте оба файла.

</details>

7. Назовите два способа уменьшить число конфликтов.

<details>
<summary>Ответ</summary>

Короткоживущие небольшие ветки; частое слияние `main` в свою ветку; использование `cargo fmt`; не смешивать переформатирование с другими изменениями; общение с командой.

</details>

## Итоги

- Конфликт — это просьба Git сделать выбор, когда две стороны изменили одно место; он нормален и полностью обратим.
- Маркеры: `<<<<<<< HEAD` (ours), `|||||||` (base, с `zdiff3`), `=======`, `>>>>>>> branch` (theirs).
- Чтобы решить: поймите, решите, уберите маркеры, `git add`, `git commit`. Проверьте через `git diff --check`.
- `git merge --abort` возвращает в состояние до слияния.
- `git checkout --ours/--theirs <file>` берёт одну сторону целиком. При rebase значения меняются местами.
- Конфликты modify/delete решаются через `git add` (оставить) или `git rm` (удалить).
- Никогда не сливайте вручную генерируемые файлы вроде `Cargo.lock`: решайте исходник и перегенерируйте.
- После решения соберите и протестируйте, прежде чем завершать.

## Источники и что почитать

- [Pro Git — 3.2 Basic Branching and Merging (Basic Merge Conflicts)](https://git-scm.com/book/en/v2/Git-Branching-Basic-Branching-and-Merging) — часть «Basic Merge Conflicts» официальной главы.
- [Pro Git — 7.8 Advanced Merging](https://git-scm.com/book/en/v2/Git-Tools-Advanced-Merging) — `--ours`/`--theirs`, вид с базой, `git diff` во время слияния.
- [git-merge manual](https://git-scm.com/docs/git-merge) — раздел «How conflicts are presented».
- [RustRover help — Resolve Git conflicts](https://www.jetbrains.com/help/rust/resolving-conflicts.html) — визуальный инструмент слияния.
- [The Cargo Book — Cargo.toml vs Cargo.lock](https://doc.rust-lang.org/cargo/guide/cargo-toml-vs-cargo-lock.html) — что такое `Cargo.lock`.

---

[🏠 Начало](../START-HERE.md) · [← 01.6 Ветки и слияние](06-branches-and-merging.md) · [📋 Этап 01](README.md) · [01.8 Удалённые репозитории: GitHub и GitLab →](08-remotes-github-and-gitlab.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
