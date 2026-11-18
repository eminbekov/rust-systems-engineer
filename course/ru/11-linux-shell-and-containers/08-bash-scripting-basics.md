<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 11.7 Менеджеры пакетов](07-package-managers.md) · [📋 Этап 11](README.md) · [11.9 Надёжные скрипты на bash →](09-robust-bash-scripts.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/11-linux-shell-and-containers/08-bash-scripting-basics.md) <!-- nav -->

# Урок 11.8 — Основы скриптов на bash

> **Этап 11 — Linux, командная оболочка и контейнеры** · Урок 8 из 16 · ⏱ около 2,5 часа (2 занятия)

## Что вы узнаете

- Что такое shell-скрипт, что делает первая строка `#!` и как запускать скрипт.
- Переменные, кавычки, подстановку команд, арифметику и специальные переменные `$1`, `$#`, `$@`, `$?`.
- Условия (`if`, `[[ ]]`, `(( ))`), циклы (`for`, `while`), `case` и функции.
- Как дать скрипту хорошие коды возврата и сообщения об ошибках, как написать для него небольшой тест и как его отлаживать.

## Перед началом

- Вы прошли [Урок 11.7 — Менеджеры пакетов](07-package-managers.md).
- Вы знаете конвейеры, перенаправление, коды возврата и утилиты из [Урока 11.3](03-text-tools-pipes-and-redirection.md) и [Урока 11.5](05-processes-and-signals.md). Скрипт — это способ сохранить такие команды в файле.
- В ВМ установлен Rust (Урок 11.5); последний скрипт использует `cargo`.

**Где лежат скрипты.** Скрипты — это текстовые файлы. Пишите их на вашем Mac, в RustRover, в папке `~/projects/rust/code/p11-scripts` (сначала создайте её), а запускайте в ВМ. В ВМ эта папка видна только для чтения, под именем вашего Mac. Перейдите туда один раз и оставайтесь:

```bash
cd /Users/student/projects/rust/code/p11-scripts
```

(Замените `student` на имя, которое показывает `ls /Users`.) Чтобы прочитать и запустить скрипт, папка не обязана быть доступной для записи. Некоторые скрипты ниже создают небольшие файлы; они лежат в домашней папке ВМ `~`, но никогда в `/Users/...`. Мы запускаем скрипты как `bash NAME.sh`, это работает всегда. Фиксируйте скрипты в Git на Mac, как вы учились на [Этапе 01](../01-git-and-version-control/README.md).

## Новые слова

| Слово | Значение |
|---|---|
| **скрипт** (script) | Текстовый файл с командами оболочки, которые оболочка выполняет сверху вниз. |
| **bash** (Bourne Again SHell, «снова оболочка Борна») | Оболочка Ubuntu и язык этого урока. |
| **шебанг** (shebang) | Первая строка `#!/usr/bin/env bash`: она говорит системе, какая программа должна выполнить файл. |
| **переменная** (variable) | Именованный кусок текста: `name="Ann"`; читается как `$name`. |
| **кавычки** (quoting) | Заключение текста в `"..."` или `'...'`, чтобы оболочка не делила и не меняла его. |
| **разбиение на слова** (word splitting) | Оболочка режет значение без кавычек на отдельные слова по пробелам. Источник многих ошибок. |
| **подстановка команды** (command substitution) | `$(command)`: заменяется выводом команды. |
| **позиционные параметры** (positional parameters) | `$1`, `$2` … аргументы, переданные скрипту; `$#` — их число, `$@` — все они. |
| **функция** (function) | Именованная группа команд внутри скрипта, которую можно вызывать снова и снова. |
| **массив** (array) | Переменная, которая хранит список значений. |
| **код возврата** (exit status) | Число, которое возвращает скрипт: 0 означает успех (Урок 11.5). |

## 1. Первый скрипт

Создайте `hello.sh` в `code/p11-scripts` на вашем Mac:

```bash
#!/usr/bin/env bash
# hello.sh - the smallest useful script
name="world"
echo "Hello, $name!"
echo "This script is called $0 and was started by $USER in $PWD"
```

Запустите его в ВМ:

```bash
bash hello.sh
```

```text
Hello, world!
This script is called hello.sh and was started by student in /home/student/p11-scripts
```

(Папка во второй строке в вашей ВМ — это `/Users/student/projects/rust/code/p11-scripts`.) **Что происходит:**

1. `#!/usr/bin/env bash` — это **шебанг**. Когда вы запускаете файл как программу, ядро читает эти два символа `#!` и запускает названную программу, передавая ей файл. `env bash` находит `bash` в вашем `PATH`, что работает в любой системе. Сам bash видит эту строку как комментарий, как и все строки, начинающиеся с `#`.
2. `name="world"` сохраняет текст в переменной. **Никаких пробелов вокруг `=`**: `name = "world"` — ошибка (раздел 2).
3. `$name`, `$0` (имя скрипта так, как вы его набрали), `$USER` и `$PWD` (текущая папка) заменяются своими значениями внутри двойных кавычек.
4. Код возврата скрипта — это код его последней команды: здесь 0.

Чтобы запустить файл напрямую (`./hello.sh`, как в [Уроке 00.4](../00-getting-started/04-terminal-basics.md)), ему нужно **право на выполнение** ([Урок 11.4](04-users-groups-and-permissions.md)). У файла, созданного в ВМ, его пока нет:

```bash
cat hello.sh > ~/run-me.sh
~/run-me.sh
chmod +x ~/run-me.sh
~/run-me.sh
rm ~/run-me.sh
```

```text
bash: /home/student/run-me.sh: Permission denied
Hello, world!
This script is called /home/student/run-me.sh and was started by student in /home/student/p11-scripts
```

**Что происходит:** первый вызов завершается с кодом 126 («найден, но не исполняем»); после `chmod +x` всё работает, а `$0` теперь равен пути, который вы набрали. Положите скрипт в папку из `PATH`, и его можно будет вызывать по имени из любого места.

## 2. Переменные, кавычки, подстановки

```bash
name="Ann Lee"
echo "Hello, $name"
echo 'Hello, $name'
count=3
echo "count is ${count}x"
echo "sum: $((count * 4 + 1))"
echo "files: $(ls | wc -l)"
echo "${undefined:-default value}"
echo "length: ${#name}"
```

```text
Hello, Ann Lee
Hello, $name
count is 3x
sum: 13
files: 7
default value
length: 7
```

**Что происходит:**

| Запись | Значение |
|---|---|
| `"$name"` | двойные кавычки: переменная подставляется, текст остаётся **одним** словом |
| `'$name'` | одинарные кавычки: ничего не подставляется, текст буквальный |
| `${count}x` | фигурные скобки отмечают, где кончается имя |
| `$((expression))` | арифметика над целыми числами: `+ - * / %` |
| `$(command)` | подстановка команды: вывод команды |
| `${var:-default}` | значение `var` или `default`, если `var` пуста или не задана |
| `${#var}` | длина текста |

Самое важное правило скриптов оболочки: **берите переменные в кавычки.** `$name` без кавычек режется на слова по пробелам. Попробуйте на файле с пробелом в имени (в домашней папке ВМ, где можно писать):

```bash
mkdir ~/quote-lab
cd ~/quote-lab
touch "my file.txt"
f="my file.txt"
ls $f
ls "$f"
cd /Users/student/projects/rust/code/p11-scripts
rm -r ~/quote-lab
```

```text
ls: cannot access 'my': No such file or directory
ls: cannot access 'file.txt': No such file or directory
my file.txt
```

**Что происходит:** `ls $f` превратилась в `ls my file.txt` — два аргумента. `ls "$f"` осталась одним. Если вместо `ls` будет `rm`, эта ошибка удалит не те файлы ([Урок 11.2](02-the-linux-file-system.md)). Всегда пишите `"$var"`, если только вам действительно не нужно разбиение. [Урок 11.9](09-robust-bash-scripts.md) идёт дальше с надёжными скриптами.

> 🔒 **Безопасность:** переменные без кавычек и текст извне (имена файлов, аргументы, скачанные данные) — корень *внедрения команд* (command injection), как в [Уроке 10.5](../10-security-fundamentals/05-injection-attacks.md). Берите всё в кавычки и никогда не собирайте команду, склеивая пользовательский текст в строку для `eval` или `bash -c`.

Оболочка задаёт для вас специальные переменные:

| Переменная | Значение |
|---|---|
| `$0` | имя скрипта |
| `$1`, `$2` … | первый, второй … аргумент |
| `$#` | число аргументов |
| `$@` | все аргументы, каждый отдельным словом, если писать `"$@"` |
| `$?` | код возврата последней команды |
| `$$` | PID (process ID, идентификатор процесса) оболочки, выполняющей скрипт |
| `$USER`, `$HOME`, `$PWD` | переменные окружения: пользователь, домашняя папка, текущая папка |

## 3. Условия

`if` выполняет команду и смотрит на её **код возврата**: 0 — истина, всё остальное — ложь. Команда обычно является проверкой:

```bash
[[ -f greet.sh ]] && echo "greet.sh is a file"
[[ -d /etc ]] && echo "/etc is a directory"
[[ "abc" == a* ]] && echo "glob match"
[[ 5 -gt 3 ]] && echo "5 > 3"
[[ "5" == "05" ]] || echo "strings differ"
x=7
if (( x % 2 == 0 )); then echo even; else echo odd; fi
```

```text
greet.sh is a file
/etc is a directory
glob match
5 > 3
strings differ
odd
```

(`greet.sh` появится после раздела 5; создайте его сначала, иначе первая строка не появится.) **Проверки:**

| Проверка | Истинна, когда |
|---|---|
| `-f PATH`, `-d PATH`, `-e PATH` | существует обычный файл / каталог / что угодно |
| `-z STR`, `-n STR` | строка пуста / не пуста |
| `A == B`, `A != B` | строки равны / различны (в `[[ ]]` справа `B` может быть шаблоном вроде `a*`) |
| `A -eq B`, `-ne`, `-lt`, `-le`, `-gt`, `-ge` | числа: равны, не равны, меньше, меньше или равно, больше, больше или равно |
| `! TEST` | проверка ложна |

`[[ ... ]]` — проверка bash (её же пишут как `[ ... ]` в более старой и хрупкой форме, которую вы встречали в Уроке 11.2). `(( ... ))` служит для **арифметики**: внутри пишут `x % 2 == 0`, `n > 0` без `$`. Полная форма:

```bash
if [[ -f "$file" ]]; then
    echo "found"
elif [[ -d "$file" ]]; then
    echo "directory"
else
    echo "missing"
fi
```

`cmd1 && cmd2` выполняет `cmd2`, только если `cmd1` успешна, `cmd1 || cmd2` — только если неуспешна ([Урок 11.2](02-the-linux-file-system.md)). Пробелы внутри `[[ ]]` обязательны: `[[-f x]]` — ошибка.

## 4. Циклы, `case`, функции, массивы

```bash
for i in 1 2 3; do echo "i=$i"; done
for f in *.sh; do echo "script: $f"; done
for ((i = 0; i < 3; i++)); do echo -n "$i "; done; echo
n=3; while (( n > 0 )); do echo "countdown $n"; n=$((n - 1)); done
```

```text
i=1
i=2
i=3
script: check-project.sh
script: greet.sh
script: hello.sh
script: log-report.sh
script: test-greet.sh
0 1 2 
countdown 3
countdown 2
countdown 1
```

(Список скриптов — это то, что лежит в вашей папке в конце урока.) **Что происходит:**

1. `for x in LIST; do ...; done` повторяется для каждого слова. Список может быть фиксированными словами, **шаблоном** (glob) вроде `*.sh` (оболочка раскрывает его в подходящие имена файлов, и имена с пробелами остаются одним словом) или `"$@"`.
2. `for ((i = 0; i < 3; i++))` — форма со счётчиком.
3. `while CONDITION; do ...; done` повторяется, пока условие успешно. Чтобы пройти по строкам файла, используйте `while read -r line; do ...; done < file`: `read` берёт по одной строке за круг из stdin (`-r` не даёт ему особым образом обрабатывать обратные косые черты).

`case` сравнивает одно значение с несколькими шаблонами:

```bash
for ext in rs toml md; do
  case "$ext" in
    rs) echo "$ext: Rust source" ;;
    toml|yaml) echo "$ext: config" ;;
    *) echo "$ext: something else" ;;
  esac
done
```

```text
rs: Rust source
toml: config
md: something else
```

**Функции** группируют команды. Функция получает аргументы как `$1`, `$2`; она «возвращает» текст через `echo` (получите его через `$(...)`), а статус — через `return`:

```bash
add() { local a="$1" b="$2"; echo $((a + b)); }
result=$(add 2 3)
echo "2+3=$result"
is_even() { (( $1 % 2 == 0 )); }
is_even 4 && echo "4 is even"
is_even 5 || echo "5 is odd"
```

```text
2+3=5
4 is even
5 is odd
```

**Что происходит:** `local` оставляет переменные внутри функции (без него они глобальные). Код возврата функции — код её последней команды, поэтому `is_even` можно использовать в `&&` / `||` и в `if`, как любую команду. **Массивы** хранят списки:

```bash
files=(a.txt b.txt "c d.txt")
echo "${#files[@]} items; second is ${files[1]}"
for f in "${files[@]}"; do echo "[$f]"; done
```

```text
3 items; second is b.txt
[a.txt]
[b.txt]
[c d.txt]
```

`"${files[@]}"` раскрывается в каждый элемент отдельным словом, пробелы сохраняются. Индексы начинаются с 0.

## 5. Настоящие скрипты с хорошими манерами

Скрипт, который вызывают другие (а также задания cron, CI и юниты systemd), должен вести себя как программы, которые вы уже знаете: **аргументы на входе, результаты в stdout, ошибки в stderr, код возврата, говорящий правду.**

Создайте `greet.sh`:

```bash
#!/usr/bin/env bash
# greet.sh [NAME]... - greet every name given, or the world
if [[ $# -eq 0 ]]; then
    echo "Hello, world!"
    exit 0
fi

for name in "$@"; do
    echo "Hello, $name!"
done
echo "(greeted $# name(s))"
```

```bash
bash greet.sh
bash greet.sh Ann Bob
bash greet.sh "Ann Lee"
```

```text
Hello, world!
Hello, Ann!
Hello, Bob!
(greeted 2 name(s))
Hello, Ann Lee!
(greeted 1 name(s))
```

**Что происходит:** `$#` — число аргументов; если их нет, скрипт приветствует мир и завершается через `exit 0`. `"$@"` сохраняет каждый аргумент одним словом, поэтому `"Ann Lee"` — одно имя.

Теперь скрипт, полезный для файла журнала из Урока 11.3. Создайте `access.log` в домашней папке ВМ (12 строк из того урока, если вы их удалили) и напишите `log-report.sh`:

```bash
#!/usr/bin/env bash
# log-report.sh FILE - count requests per status code in a web server log
# Each line of FILE looks like: 203.0.113.5 GET /index.html 200 512

usage() {
    echo "usage: $0 LOGFILE" >&2
}

if [[ $# -ne 1 ]]; then
    usage
    exit 2
fi

log="$1"
if [[ ! -f "$log" ]]; then
    echo "error: '$log' is not a file" >&2
    exit 1
fi

total=$(wc -l < "$log")
echo "total requests: $total"

for code in $(cut -d' ' -f4 "$log" | sort -u); do
    count=$(grep -c " $code " "$log")
    printf '%s: %s\n' "$code" "$count"
done

errors=$(awk '$4 >= 400' "$log" | wc -l)
if (( errors * 100 / total >= 30 )); then
    echo "WARNING: $errors of $total requests failed"
    exit 3
fi
echo "ok: $errors of $total requests failed"
```

```bash
cd ~
bash /Users/student/projects/rust/code/p11-scripts/log-report.sh access.log
echo "exit=$?"
```

```text
total requests: 12
200: 5
201: 1
204: 1
304: 1
403: 1
404: 2
500: 1
WARNING: 4 of 12 requests failed
exit=3
```

**Что происходит:**

1. `usage` — это **функция**; она пишет в **stderr** (`>&2`), потому что это сообщение для человека, а не результат. Скрипт завершается с кодом 2 при неверном использовании, 1 при плохом файле, 3 при «слишком много сбоев», 0 при успехе. Документируйте свои коды в комментарии.
2. `for code in $(...)` проходит по кодам статуса, которые встречаются в файле (каждый — просто число, поэтому подстановка без кавычек здесь допустима).
3. `total=$(wc -l < "$log")` сохраняет число; `(( errors * 100 / total >= 30 ))` вычисляет процент сбоев (целочисленная арифметика) и сравнивает с 30.
4. Те же части из Урока 11.3 (`cut`, `sort`, `grep -c`, `awk`) теперь составляют один инструмент, который можно вызывать снова.

Попробуйте ошибочные случаи:

```bash
bash /Users/student/projects/rust/code/p11-scripts/log-report.sh; echo "exit=$?"
bash /Users/student/projects/rust/code/p11-scripts/log-report.sh nofile.log; echo "exit=$?"
```

```text
usage: /Users/student/projects/rust/code/p11-scripts/log-report.sh LOGFILE
exit=2
error: 'nofile.log' is not a file
exit=1
```

(`$0` — это путь, который вы набрали, поэтому он виден в строке usage.) Третий скрипт запускает контроль качества Rust (`fmt`, `clippy`, `test` с Этапа 05) в каждой указанной вами папке. Создайте `check-project.sh`:

```bash
#!/usr/bin/env bash
# check-project.sh DIR... - run the Rust quality gate in each project folder

failed=0

for dir in "$@"; do
    echo "== $dir"
    if [[ ! -f "$dir/Cargo.toml" ]]; then
        echo "   skipped: no Cargo.toml"
        continue
    fi
    if (cd "$dir" && cargo fmt --check && cargo clippy -q -- -D warnings && cargo test -q); then
        echo "   PASS"
    else
        echo "   FAIL"
        failed=$((failed + 1))
    fi
done

echo "failed projects: $failed"
if [[ $failed -gt 0 ]]; then
    exit 1
fi
```

**Что происходит:** круглые скобки `( cd "$dir" && ... )` выполняют команды в **подоболочке** (subshell), поэтому `cd` не меняет папку самого скрипта. `if (command...)` использует код возврата всей цепочки: она истинна, только если успешны все три команды cargo. `-D warnings` заставляет Clippy считать предупреждения ошибками. Счётчик `failed` становится кодом возврата: сам скрипт можно использовать в цепочках `&&` и в CI ([Урок 09.12](../09-testing-debugging-benchmarking/12-continuous-integration.md)). Чтобы попробовать, создайте в домашней папке ВМ хороший проект, проект с падающим тестом и папку без `Cargo.toml` и выполните:

```bash
cd /Users/student/projects/rust/code/p11-scripts
bash check-project.sh ~/proj-lab/good ~/proj-lab/bad ~/proj-lab/notes 2>&1 | grep -E '^(==|   (PASS|FAIL|skipped)|failed)'
```

```text
== /home/student/proj-lab/good
   PASS
== /home/student/proj-lab/bad
   FAIL
== /home/student/proj-lab/notes
   skipped: no Cargo.toml
failed projects: 1
```

(`grep` оставляет только итоговые строки вывода. Чтобы попробовать скрипт, создайте проект `good` командой `cargo new --vcs none good`, проект `bad` с тестом, содержащим `assert_eq!(1 + 1, 3)`, и папку `notes` без `Cargo.toml`, всё в `~/proj-lab`.)

## 6. Тестирование и отладка скрипта

Скрипт — это код: ему нужны тесты. Тест оболочки запускает скрипт и сравнивает то, что он печатает (и его код возврата), с ожидаемым. Создайте `test-greet.sh`:

```bash
#!/usr/bin/env bash
# test-greet.sh - a tiny test for greet.sh
cd "$(dirname "$0")" || exit 1

check() {
    local description="$1" expected="$2" actual="$3"
    if [[ "$actual" == "$expected" ]]; then
        echo "PASS: $description"
    else
        echo "FAIL: $description"
        echo "  expected: $expected"
        echo "  actual:   $actual"
        failures=$((failures + 1))
    fi
}

failures=0
check "one name" $'Hello, Ann!\n(greeted 1 name(s))' "$(bash greet.sh Ann)"
check "no name" "Hello, world!" "$(bash greet.sh)"
check "two names" $'Hello, Ann!\nHello, Bob!\n(greeted 2 name(s))' "$(bash greet.sh Ann Bob)"

if (( failures > 0 )); then
    exit 1
fi
```

```bash
bash test-greet.sh; echo "exit=$?"
```

```text
PASS: one name
PASS: no name
PASS: two names
exit=0
```

**Что происходит:** `$(dirname "$0")` — папка тестового скрипта, поэтому он работает из любого каталога. `check` сравнивает настоящий вывод (`$(...)` захватывает stdout) с ожидаемым текстом; `$'...\n...'` — способ bash записать перевод строки внутри строки. Тест завершается с кодом 1, когда что-то отличается, так что сервер сборки может это заметить. Тесту можно доверять, только если вы хоть раз видели, как он падает: замените `Hello,` на `Hi,` в *копии* `greet.sh`, и подходящая строка напечатает `FAIL`.

Для поиска ошибок у bash есть два помощника:

```bash
bash -n greet.sh; echo "syntax ok? $?"
bash -x greet.sh Ann
```

```text
syntax ok? 0
+ [[ 1 -eq 0 ]]
+ for name in "$@"
+ echo 'Hello, Ann!'
Hello, Ann!
+ echo '(greeted 1 name(s))'
(greeted 1 name(s))
```

**Что происходит:** `bash -n` только проверяет синтаксис, ничего не выполняя. `bash -x` («trace», трассировка) печатает каждую команду с подставленными переменными после `+` до её выполнения: самый быстрый способ увидеть, что скрипт делает на самом деле. Можно также написать `set -x` внутри скрипта вокруг подозрительного места. Эти строки идут в stderr.

## Частые ошибки

1. **Пробелы вокруг `=`.** `name = "Ann"` → `name: command not found`. *Исправление:* `name="Ann"`.
2. **Переменные без кавычек.** `rm $file` с пробелом в имени удаляет не то. *Исправление:* `rm "$file"`.
3. **Нет пробелов в проверках.** `[[$x -eq 1]]` → `command not found`. *Исправление:* пробелы внутри скобок: `[[ $x -eq 1 ]]`.
4. **`=` вместо `-eq` для чисел, `-eq` вместо `==` для текста.** `[[ "5" == "05" ]]` ложно, `[[ 5 -eq 05 ]]` истинно. *Исправление:* для чисел используйте `-eq`, `-lt`, …, для текста — `==`.
5. **Деление на ноль в `(( ))`.** `(( errors * 100 / total ))` при `total=0` печатает `division by 0`, но скрипт **продолжает работу** (см. Упражнение 2). *Исправление:* сначала проверьте на ноль.
6. **Слишком поздно читать `$?`.** Другая команда между ними его перезаписывает. *Исправление:* `status=$?` сразу.
7. **Забыли, что скрипты работают в собственном процессе.** `cd` или `export` в скрипте не меняют ваш терминал. *Исправление:* так задумано; `source script.sh` используйте только для скриптов, которые настраивают вашу оболочку.
8. **Разбор вывода `ls`.** Имена с пробелами его ломают. *Исправление:* используйте шаблон (`for f in *.sh`) или `find ... -print0`.

## Попробуйте сами

1. Создайте `hello.sh`, запустите его командой `bash hello.sh`, затем запустите напрямую, сделав копию в домашней папке ВМ с `chmod +x`.
2. Напишите `greet.sh` и запустите его без аргументов, с одним и с именем с пробелом.
3. Напишите `log-report.sh` и запустите его на `access.log` и на файле только из первых 6 строк (`head -n 6 access.log > ~/small.log`). Ожидаемый код возврата: 3 для полного файла, 0 для маленького (`ok: 1 of 6 requests failed`).
4. Добавьте `set -x` второй строкой в копию `greet.sh` и прочитайте трассировку. Затем удалите копию.
5. Напишите `test-greet.sh` и запустите его. Сломайте `greet.sh` в копии и посмотрите, как тест падает.

## Упражнения

### Упражнение 1 — Пронумеруйте аргументы (легко)

Напишите `args.sh ARG...`, который печатает `got N argument(s)`, а затем каждый аргумент на отдельной строке с номером: `1: first`. Аргумент с пробелом должен остаться одним аргументом.

<details>
<summary>Подсказка</summary>

`$#` — это количество. Пройдите циклом по `"$@"` (с кавычками) и ведите счётчик через `n=$((n + 1))`.

</details>

<details>
<summary>Ответ</summary>

```bash
#!/usr/bin/env bash
# args.sh ARG... - number the arguments
echo "got $# argument(s)"
n=1
for arg in "$@"; do
    echo "$n: $arg"
    n=$((n + 1))
done
```

```bash
bash args.sh a "b c" d
bash args.sh
```

```text
got 3 argument(s)
1: a
2: b c
3: d
got 0 argument(s)
```

Почему это работает: `"$@"` сохраняет `b c` одним словом, поэтому цикл выполняется три раза; без аргументов тело цикла не выполняется вообще.

</details>

### Упражнение 2 — Исправьте настоящую ошибку (средне)

Запустите `log-report.sh` на **пустом** файле (`: > empty.log`). Что он печатает и что не так? Исправьте скрипт так, чтобы пустой журнал сообщался как `ok: no requests` с кодом 0.

<details>
<summary>Подсказка</summary>

Посмотрите на строку с `(( errors * 100 / total >= 30 ))` и подумайте о `total=0`. Проверьте `total` до деления.

</details>

<details>
<summary>Ответ</summary>

До исправления:

```text
total requests: 0
./log-report.sh: line 29: ((: errors * 100 / total >= 30 : division by 0 (error token is "total >= 30 ")
ok: 0 of 0 requests failed
exit=0
```

Скрипт печатает ошибку, но всё равно говорит `ok` и завершается с кодом 0: неудавшаяся `(( ))` считается «ложью», поэтому `if` пропускается. Добавьте проверку сразу после `echo "total requests: $total"`:

```bash
if (( total == 0 )); then
    echo "ok: no requests"
    exit 0
fi
```

```text
total requests: 0
ok: no requests
exit=0
```

Почему это работает: проверка обрабатывает особый случай до деления. Вывод: сообщение об ошибке в скрипте его не останавливает, пока вы не остановите его сами. Следующий урок, о надёжных скриптах, показывает, как сделать так, чтобы скрипт останавливался на первой неудаче.

</details>

### Упражнение 3 — Напишите тест для кодов возврата (сложно)

Напишите `test-log-report.sh`, который проверяет четыре кода возврата `log-report.sh`: нет аргумента (2), нет файла (1), журнал, где все запросы завершились сбоем (3), и журнал, где сбоев нет (0). Тест сам создаёт небольшие файлы журналов во временной папке и в конце их удаляет. Он печатает `PASS`/`FAIL` для каждого случая и завершается с кодом 0, только если все прошли.

<details>
<summary>Подсказка</summary>

`mktemp -d` создаёт новую пустую временную папку и печатает её имя. Напишите функцию `expect_status EXPECTED LABEL ARGS...`: запустите скрипт с `> /dev/null 2>&1`, чтобы скрыть его вывод, сразу сохраните `$?` в переменную и сравните. Используйте `shift 2`, чтобы убрать первые два аргумента, прежде чем передавать остальные.

</details>

<details>
<summary>Ответ</summary>

```bash
#!/usr/bin/env bash
# test-log-report.sh - check the exit status of log-report.sh
cd "$(dirname "$0")" || exit 1

data=$(mktemp -d)
printf '203.0.113.5 GET /a 200 1\n203.0.113.5 GET /a 200 1\n' > "$data/ok.log"
printf '203.0.113.5 GET /a 500 0\n203.0.113.5 GET /a 500 0\n' > "$data/bad.log"

failures=0

# expect_status EXPECTED LABEL [ARGS...]
expect_status() {
    local expected="$1" label="$2"
    shift 2
    bash log-report.sh "$@" > /dev/null 2>&1
    local actual=$?
    if [[ $actual -eq $expected ]]; then
        echo "PASS: $label -> $actual"
    else
        echo "FAIL: $label -> $actual (expected $expected)"
        failures=$((failures + 1))
    fi
}

expect_status 2 "no argument"
expect_status 1 "missing file" "$data/missing.log"
expect_status 3 "all requests failed" "$data/bad.log"
expect_status 0 "no request failed" "$data/ok.log"

rm -r "$data"
[[ $failures -eq 0 ]]
```

```text
PASS: no argument -> 2
PASS: missing file -> 1
PASS: all requests failed -> 3
PASS: no request failed -> 0
```

Чтобы увидеть падение теста, замените `exit 3` на `exit 4` в **копии** `log-report.sh`: третья строка станет `FAIL: all requests failed -> 4 (expected 3)`, а код возврата теста будет 1.

Почему это работает: `shift 2` отбрасывает первые два аргумента, `"$@"` передаёт остальные, а `local actual=$?` читает код сразу. Самая последняя строка — это код возврата теста: 0, только если ни один случай не упал. Это та же идея, что и `assert_eq!` в Rust, только на языке оболочки. Тесту не нужны файлы от вас, поэтому он работает и в папке только для чтения.

</details>

## Вопросы для самопроверки

1. Что делает `#!/usr/bin/env bash` в первой строке?

<details>
<summary>Ответ</summary>

Когда файл запускают напрямую, ядро запускает названную программу (`bash`, найденный через `env` в `PATH`) и передаёт ей файл. Сам bash считает эту строку комментарием.

</details>

2. Чем отличаются `"$@"` и `$@`?

<details>
<summary>Ответ</summary>

`"$@"` сохраняет каждый аргумент одним словом (даже с пробелами). `$@` без кавычек снова делит аргументы по пробелам.

</details>

3. Когда условие `if` «истинно»?

<details>
<summary>Ответ</summary>

Когда команда в нём завершилась с кодом возврата 0.

</details>

4. Куда должны идти сообщения об ошибках скрипта и почему?

<details>
<summary>Ответ</summary>

В stderr (`>&2`), чтобы в stdout оставались только результаты, которые другие программы могут использовать в конвейерах.

</details>

5. Что делают `bash -n` и `bash -x`?

<details>
<summary>Ответ</summary>

`-n` проверяет синтаксис, ничего не запуская; `-x` печатает каждую команду с подставленными переменными перед её выполнением.

</details>

## Итоги

- Скрипт — текстовый файл с командами оболочки; шебанг `#!/usr/bin/env bash` называет интерпретатор; запускайте как `bash NAME.sh` или после `chmod +x` как `./NAME.sh`.
- Переменные: `name="value"` (без пробелов), `"$name"` (в кавычках!), `$(command)`, `$((arithmetic))`, `${var:-default}`.
- Специальные переменные: `$0`, `$1…`, `$#`, `"$@"`, `$?`, `$$`.
- Условия используют коды возврата: `if`, `[[ ]]` для файлов, текста и чисел, `(( ))` для арифметики, `&&` и `||`.
- Циклы `for` и `while`, `case`, функции с `local`, массивы.
- Хороший скрипт: аргументы на входе, результаты в stdout, ошибки в stderr, честные коды возврата, сообщение об использовании.
- Тестируйте скрипты небольшим тестовым скриптом; отлаживайте с `bash -n` и `bash -x`.
- Дальше: [Урок 11.9 — Надёжные скрипты на bash](09-robust-bash-scripts.md) продолжает тему скриптов, которым можно доверять в рабочей среде.

## Источники и что почитать

- [bash(1) manual page](https://man7.org/linux/man-pages/man1/bash.1.html) — полный справочник по языку и по кодам возврата.
- [GNU Bash Manual — Shell Expansions](https://www.gnu.org/software/bash/manual/html_node/Shell-Expansions.html) — кавычки, раскрытие параметров, подстановка команд, арифметика.
- [GNU Bash Manual — Conditional Constructs](https://www.gnu.org/software/bash/manual/html_node/Conditional-Constructs.html) — `if`, `case`, `[[ ]]`, `(( ))`.
- [GNU Bash Manual — Bash Conditional Expressions](https://www.gnu.org/software/bash/manual/html_node/Bash-Conditional-Expressions.html) — список проверок файлов и строк.
- [ShellCheck](https://www.shellcheck.net/) — инструмент, который находит типичные ошибки в shell-скриптах.

---

[🏠 Начало](../START-HERE.md) · [← 11.7 Менеджеры пакетов](07-package-managers.md) · [📋 Этап 11](README.md) · [11.9 Надёжные скрипты на bash →](09-robust-bash-scripts.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
