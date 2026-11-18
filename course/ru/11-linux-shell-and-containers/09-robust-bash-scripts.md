<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 11.8 Основы скриптов на bash](08-bash-scripting-basics.md) · [📋 Этап 11](README.md) · [11.10 SSH и удалённые серверы →](10-ssh-and-remote-servers.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/11-linux-shell-and-containers/09-robust-bash-scripts.md) <!-- nav -->

# Урок 11.9 — Надёжные скрипты на bash

> **Этап 11 — Linux, командная оболочка и контейнеры** · Урок 9 из 16 · ⏱ около 2,5 часа (2 занятия)

## Что вы узнаете

- Почему обычный скрипт на bash продолжает работу после ошибки и как «строгий режим» (`set -euo pipefail`) его останавливает.
- В каких местах строгий режим не помогает, чтобы он вас не удивлял.
- Как правильно заключать переменные в кавычки и почему одна пропущенная пара кавычек может удалить не те файлы.
- Как за собой убирать с помощью `mktemp` и `trap`.
- Как написать законченный скрипт с параметрами, справкой и безопасными значениями по умолчанию, как проверить его с помощью ShellCheck и как его протестировать.

## Перед началом

- Вы прошли [Урок 11.8 — Основы скриптов на bash](08-bash-scripting-basics.md). Вы умеете писать скрипт с первой строкой `#!/usr/bin/env bash`, переменными, `if`, `for`, функциями и аргументами (`$1`, `$#`) и знаете, что каждая команда завершается кодом возврата ([Урок 11.5](05-processes-and-signals.md)).
- Вы знаете конвейеры и перенаправление ([Урок 11.3](03-text-tools-pipes-and-redirection.md)) и права доступа ([Урок 11.4](04-users-groups-and-permissions.md)).
- Docker Desktop здесь необязателен. Он нужен только в разделе 7, чтобы запустить ShellCheck без установки.

**Куда класть файлы.** Пишите скрипты на Mac в `~/projects/rust/code/p11-scripts` (в RustRover или любом редакторе), затем копируйте их в ВМ и запускайте там. Папка Mac внутри ВМ доступна только для чтения ([Урок 11.1](01-linux-and-a-virtual-machine.md)), поэтому сначала копируйте (замените `student` на имя вашего пользователя Mac; его показывает `ls /Users`):

```bash
mkdir -p ~/scripts
cp /Users/student/projects/rust/code/p11-scripts/*.sh ~/scripts/
cd ~/scripts
```

Повторяйте `cp` после каждой правки. Все команды этого урока выполняются внутри ВМ.

## Новые слова

| Слово | Значение |
|---|---|
| **строгий режим** (strict mode) | Три настройки `set -e`, `set -u` и `set -o pipefail`, из-за которых скрипт останавливается на первой ошибке, а не продолжает работу. |
| **опция оболочки** (shell option) | Переключатель самой командной оболочки: включается через `set -o имя` (или короткий флаг вроде `set -e`) и выключается через `set +o имя` (`set +e`). |
| **неопределённая переменная** (unbound variable) | Переменная, которой никогда не присваивали значение. Обычно bash считает её пустым текстом; при `set -u` это ошибка. |
| **разбиение на слова** (word splitting) | Bash разрезает значение переменной без кавычек на отдельные слова по пробелам и табуляциям. `name="my notes.txt"` превращается в два слова. |
| **шаблон имён** (glob, globbing) | Шаблон вроде `*.txt`, который bash заменяет списком подходящих имён файлов. |
| **кавычки** (quoting) | Заключение значения в кавычки (`"$name"`), чтобы bash сохранил его как **одно** слово и не раскрывал шаблоны. |
| **`trap`** | Команда оболочки, которая говорит: выполни этот код, когда наступит определённое событие, например когда скрипт завершится или получит сигнал. |
| **временный файл** (temporary file) | Файл, который существует, только пока работает программа. `mktemp` создаёт такой файл с уникальным именем, которое никто не угадает. |
| **атомарная** (atomic) операция | Действие, которое либо выполнено полностью, либо не выполнено совсем, но никогда не выполнено наполовину. Переименование файла атомарно. |
| **массив** (array) | Пронумерованный список значений в одной переменной: `files=(a b c)`. `${files[0]}` — первое значение, `${#files[@]}` — их количество. |
| **параметр командной строки** (option) | Аргумент, который начинается с дефиса и меняет работу команды: `-k 3`, `-n`. |
| **сообщение об использовании** (usage message) | Короткая справка, которую скрипт печатает, если его вызвали неправильно или с `-h`. |
| **пробный запуск** (dry run) | Режим, в котором программа сообщает, что она **сделала бы**, и ничего не меняет. |
| **ShellCheck** | Бесплатная программа, которая читает скрипт командной оболочки и предупреждает о типичных ошибках. Программы такого рода называют *линтерами*. |
| **линтер** (linter) | Инструмент, который читает исходный код, не запуская его, и сообщает об ошибках и плохом стиле. `cargo clippy` — линтер для Rust. |

## 1. Скрипт, который не останавливается

Bash создавали для ручного ввода команд. Если одна команда не сработала, вы читаете сообщение и решаете, что делать. В скрипте решать некому, но bash ведёт себя так же: **он переходит к следующей строке**. Создайте `fragile.sh`:

```bash
#!/usr/bin/env bash
# A fragile script: it keeps going after errors.
cd /no/such/folder
echo "now I am in: $(pwd)"
echo "cleaning up"
```

Запустите его и посмотрите код возврата:

```bash
bash fragile.sh
echo "exit status: $?"
```

```text
fragile.sh: line 3: cd: /no/such/folder: No such file or directory
now I am in: /home/student/scripts
cleaning up
exit status: 0
```

**Что происходит:**

1. `cd` не срабатывает и печатает ошибку, но bash это не волнует: он выполняет строку 4.
2. Скрипт продолжает работу в **неправильной папке**. Если бы следующей строкой было `rm -rf *`, он удалил бы файлы той папки, из которой вы начали.
3. Последняя команда (`echo`) прошла успешно, поэтому скрипт завершается с **кодом 0**: вызывающему сообщают «всё в порядке». Инструмент развёртывания, менеджер служб или задание CI (continuous integration, непрерывная интеграция; Урок 09.12) этому поверили бы.

Надёжный скрипт должен (а) останавливаться, когда что-то пошло не так, и (б) говорить правду об этом в своём коде возврата. Остальная часть урока — о том, как этого добиться.

## 2. Строгий режим

Поставьте эту строку сразу после первой строки каждого скрипта:

```bash
set -euo pipefail
```

Это три переключателя, записанные вместе. Проверим каждый из них.

**`-e` (выход при ошибке).** Скрипт завершается, когда простая команда не срабатывает (возвращает код, отличный от 0). Скопируйте `fragile.sh` в `strict.sh` и добавьте строку:

```bash
#!/usr/bin/env bash
set -euo pipefail
cd /no/such/folder
echo "now I am in: $(pwd)"
```

```bash
bash strict.sh
echo "exit status: $?"
```

```text
strict.sh: line 3: cd: /no/such/folder: No such file or directory
exit status: 1
```

Скрипт остановился на строке с ошибкой и передал сбой дальше (код 1).

**`-u` (неопределённая переменная — ошибка).** В `unset.sh`:

```bash
#!/usr/bin/env bash
set -u
echo "target is: $TARGET"
echo "not reached"
```

```bash
bash unset.sh
echo "exit status: $?"
```

```text
unset.sh: line 3: TARGET: unbound variable
exit status: 1
```

Без `-u` переменная `$TARGET` молча оказалась бы пустой. Так случается знаменитая авария. Посмотрите на опечатку (`targt`) в `danger.sh`:

```bash
#!/usr/bin/env bash
# DANGER demo: we only echo the command, we never run it.
target=/tmp/lab
echo rm -rf "$targt"/
```

```text
rm -rf /
```

Неправильно написанная переменная пуста, поэтому «удалить папку `$target/`» превратилось в «удалить всё». Если вверху стоит `set -u` (`danger2.sh`), та же опечатка останавливает скрипт сообщением `danger2.sh: line 4: targt: unbound variable`, и ничего не удаляется. (Эти скрипты только печатают команду. Никогда не запускайте такую команду по-настоящему.)

> 🔒 **Безопасность:** скрипты часто работают от root (из `sudo`, из systemd, из инструмента развёртывания). Скрипт, который удаляет, перемещает или меняет права, ни в коем случае не должен работать с путём, который может оказаться пустым. Кроме `set -u`, можно написать `rm -rf -- "${target:?target is not set}/"`. Форма `${name:?сообщение}` останавливает скрипт с сообщением, если переменная не задана **или пуста**.

**`-o pipefail`.** Конвейер (`a | b | c`, [Урок 11.3](03-text-tools-pipes-and-redirection.md)) обычно имеет код только своей **последней** команды. В `pipefail.sh`:

```bash
#!/usr/bin/env bash
false | cat
echo "without pipefail the pipeline status was: $?"
set -o pipefail
false | cat
echo "with pipefail the pipeline status was: $?"
```

```text
without pipefail the pipeline status was: 0
with pipefail the pipeline status was: 1
```

`false` всегда завершается ошибкой. `cat` срабатывает, поэтому конвейер выглядел благополучным. С `pipefail` конвейер завершается ошибкой, если ошибкой завершилась **любая** его команда. Представьте `curl https://example.org/file | tar -xz`: если `curl` не смог скачать, вы хотите об этом знать.

> 💡 **Совет:** строгий режим — страховочная сетка, а не замена размышлению. Коротко запомните: `-e` — стоп при ошибке, `-u` — стоп при опечатке, `pipefail` — не скрывать сбой внутри конвейера.

## 3. Где `set -e` не помогает

У `set -e` есть исключения. Знание о них экономит часы. Запустите `esurprise.sh` с тремя случаями (комментарий над каждым — это правило):

```bash
#!/usr/bin/env bash
set -e
# 1. grep returns 1 when nothing matches. That is not a crash!
echo "hello" | grep -q xyz || echo "1: grep found nothing, fine"

# 2. Arithmetic: the value 0 counts as 'false' and gives status 1.
count=0
((count++)) || echo "2: ((count++)) returned status 1 when count was 0"
echo "count is now $count"

# 3. 'local x=$(cmd)' hides the status of cmd.
f() { local out=$(false); echo "3: local hid the failure, out=[$out]"; }
f
g() { local out; out=$(false); echo "not reached"; }
g
echo "also not reached"
```

```bash
bash esurprise.sh
echo "exit status: $?"
```

```text
1: grep found nothing, fine
2: ((count++)) returned status 1 when count was 0
count is now 1
3: local hid the failure, out=[]
exit status: 1
```

**Что происходит:**

1. `set -e` также не останавливает команду, которая используется как **условие** `if` или стоит слева от `||` и `&&`: вы и так обрабатываете сбой. Поэтому `grep ... || echo` выше безопасно. `grep` использует код 1 для «нет совпадений», это информация, а не ошибка.
2. `((count++))` вычисляется как **старое значение** `count`, то есть 0, а арифметика считает 0 ложью, поэтому код возврата равен 1. При `set -e` одна такая строка остановила бы скрипт. Вместо этого пишите `count=$((count + 1))`.
3. В `local out=$(false)` две команды: `local` (она срабатывает) и подстановка. Побеждает код `local`, и сбой исчезает. Исправление — объявлять и присваивать в **двух строках** (`local out; out=$(false)`), как делает функция `g`: там скрипт останавливается.

> ⚠️ **Частая ошибка:** считать, что `set -e` делает скрипт безопасным. Он ловит не все ошибки. Используйте его вместе с явными проверками того, что можно предсказать («существует ли эта папка?»), и вместе с ShellCheck (раздел 7).

## 4. Кавычки

Bash обрабатывает командную строку по шагам. Два из них удивляют новичков: **разбиение на слова** (`$name` без кавычек режется по пробелам) и **раскрытие шаблонов** (`*` в значении без кавычек заменяется именами файлов). Лекарство простое: **заключайте в двойные кавычки каждую переменную и каждое `$(...)`**, если только вы не хотите именно разбиения. Запустите `quoting.sh`:

```bash
#!/usr/bin/env bash
set -euo pipefail
dir=$(mktemp -d)
cd "$dir"
touch "my notes.txt"
echo "--- unquoted"
for f in $(ls); do echo "item: [$f]"; done
echo "--- glob and quotes"
for f in *; do echo "item: [$f]"; done
echo "--- unquoted variable in a command"
name="my notes.txt"
rm $name || echo "rm failed"
rm "$name"
find . -type f | wc -l
cd /
rmdir "$dir"
```

```text
--- unquoted
item: [my]
item: [notes.txt]
--- glob and quotes
item: [my notes.txt]
--- unquoted variable in a command
rm: cannot remove 'my': No such file or directory
rm: cannot remove 'notes.txt': No such file or directory
rm failed
0
```

**Что происходит:**

1. `mktemp -d` создаёт новую пустую папку со случайным именем в `/tmp` (подробнее в разделе 5). Скрипт работает там, поэтому не может навредить вашим файлам.
2. `$(ls)` возвращает текст `my notes.txt`. Без кавычек он разбивается на два слова, и цикл выполняется дважды с несуществующими именами. Никогда не перебирайте в цикле вывод `ls`. Шаблон (`*`) отдаёт каждое имя файла как **одно** слово, пробелы включая.
3. `rm $name` превратилось в `rm my notes.txt`: два неправильных имени. `rm "$name"` — правильно.

Правила в порядке важности:

| Правило | Пример |
|---|---|
| Заключайте в кавычки каждую переменную | `"$file"`, `"${dir}/out"` |
| Заключайте в кавычки каждую подстановку команды | `"$(date +%F)"` |
| Передавайте все аргументы дальше через `"$@"` | `cp -- "$@" /backup` |
| Заканчивайте опции через `--` перед именами | `rm -- "$file"` |
| Для проверок используйте `[[ ... ]]` | `[[ -d $dir ]]` (внутри `[[ ]]` кавычки не нужны) |
| Для текста, который должен остаться буквальным, используйте одинарные кавычки | `'$HOME is not expanded here'` |

Два из этих правил требуют примера. `"$@"` сохраняет каждый аргумент как одно слово. При аргументах `"a b"` и `c` `"$@"` даёт два слова (`a b` и `c`), а `$@` или `$*` без кавычек дают три (`a`, `b`, `c`), и `"$*"` склеивает всё в одно. `--` защищает от имён файлов, которые начинаются с дефиса:

```bash
touch ./-rf
rm -- -rf && echo "removed a file called -rf"
```

```text
removed a file called -rf
```

Без `--` команда `rm -rf` была бы прочитана как опции `-r` и `-f`. Любой, кто может создавать файлы в папке (загрузка, распакованный архив), может назвать их как опции (идея Урока 10.5).

## 5. Уборка с помощью `mktemp` и `trap`

Скрипту часто нужен временный файл или папка. Не придумывайте имя вроде `/tmp/work`: другой запуск или другой пользователь может взять то же имя, а злоумышленник может подготовить его заранее. `mktemp` создаёт файл (или, с `-d`, папку) со **случайным** именем и безопасными правами (читать может только вы) и печатает это имя.

Когда скрипт завершается, временные файлы должны исчезнуть, **в том числе когда он завершился с ошибкой**. Это делает `trap`. Запустите `trap.sh`:

```bash
#!/usr/bin/env bash
set -euo pipefail
tmp=$(mktemp -d)
cleanup() {
    echo "cleanup: removing $tmp"
    rm -rf "$tmp"
}
trap cleanup EXIT
echo "working in $tmp"
echo data > "$tmp/work.txt"
false   # an error: the script stops here
echo "not reached"
```

```bash
bash trap.sh
echo "exit status: $?"
```

```text
working in /tmp/tmp.LL1HIzxbjs
cleanup: removing /tmp/tmp.LL1HIzxbjs
exit status: 1
```

**Что происходит:**

1. `trap cleanup EXIT` регистрирует функцию `cleanup`. `EXIT` — особое событие: оно наступает, **когда бы ни завершился скрипт**: при обычном конце, при `exit` и после ошибки, пойманной `set -e`.
2. `false` завершается ошибкой, `set -e` завершает скрипт, и trap срабатывает до возврата bash. Код возврата по-прежнему 1: trap не скрывает сбой.
3. Имя папки случайное; у вас оно будет другим.

А если кто-то нажмёт `Ctrl + C` или менеджер служб пошлёт `SIGTERM` ([Урок 11.5](05-processes-and-signals.md))? Назовите сигналы во втором trap и завершайтесь с условленным кодом 128 + номер сигнала (`SIGINT` — это 2, значит 130). Trap на `EXIT` после этого всё равно сработает:

```bash
#!/usr/bin/env bash
set -euo pipefail
tmp=$(mktemp -d)
trap 'echo "cleanup $tmp"; rm -rf "$tmp"' EXIT
trap 'echo "interrupted"; exit 130' INT TERM
echo "sleeping, pid $$"
sleep 30 &
wait $!
```

Запустите его в фоне, отправьте `SIGTERM` командой `kill -TERM $!` и прочитайте код возврата через `wait $!` и `echo $?`. Результат:

```text
sleeping, pid 14
interrupted
cleanup /tmp/tmp.s0MXYBEICj
exit status: 130
```

**Что происходит:** `$$` — номер процесса самого скрипта. Скрипт ждёт `sleep` через `wait`, потому что bash выполняет trap только между командами, а не посреди долгой команды переднего плана; `wait` можно прервать. (Сам `sleep 30` продолжает работать после ухода скрипта. В настоящем скрипте вы бы также останавливали запущенные вами дочерние процессы.)

## 6. Законченный скрипт: `backup.sh`

Теперь соберём привычки в одном инструменте: он делает сжатую резервную копию папки и оставляет только несколько самых новых. В нём есть **параметры**, **сообщение об использовании**, **проверка входных данных**, **пробный запуск**, **временный файл** и **атомарное** переименование. Сохраните его как `code/p11-scripts/backup.sh` на Mac (и сделайте исполняемым там командой `chmod +x`):

```bash
#!/usr/bin/env bash
# backup.sh - make a compressed backup of a directory and keep only the newest few.
set -euo pipefail

prog=${0##*/}
keep=3
dry_run=0

usage() {
    cat <<USAGE
Usage: $prog [-k COUNT] [-n] SOURCE_DIR DEST_DIR

Writes DEST_DIR/NAME-YYYYmmdd-HHMMSS.tar.gz, where NAME is the name of SOURCE_DIR,
then deletes the oldest backups so that only COUNT remain.

  -k COUNT  how many backups to keep (default: $keep)
  -n        dry run: print what would happen, change nothing
  -h        show this help
USAGE
}

log() { printf '%s: %s\n' "$prog" "$*" >&2; }
die() { log "error: $*"; exit 2; }

while getopts ':k:nh' opt; do
    case $opt in
        k) keep=$OPTARG ;;
        n) dry_run=1 ;;
        h) usage; exit 0 ;;
        :) die "option -$OPTARG needs a value" ;;
        \?) die "unknown option -$OPTARG" ;;
    esac
done
shift $((OPTIND - 1))

[[ $# -eq 2 ]] || { usage >&2; exit 2; }
[[ $keep =~ ^[1-9][0-9]*$ ]] || die "COUNT must be a positive number, got '$keep'"
source_dir=$1
dest_dir=$2
[[ -d $source_dir ]] || die "not a directory: $source_dir"

name=$(basename -- "$(realpath -- "$source_dir")")
stamp=${BACKUP_STAMP:-$(date +%Y%m%d-%H%M%S)}
target=$dest_dir/$name-$stamp.tar.gz

if (( dry_run )); then
    log "dry run: would create $target"
else
    mkdir -p -- "$dest_dir"
    tmp=$(mktemp -- "$dest_dir/.$name.XXXXXX")
    trap 'rm -f -- "$tmp"' EXIT
    tar -czf "$tmp" -C "$source_dir" .
    mv -- "$tmp" "$target"
    log "created $target"
fi

shopt -s nullglob
backups=("$dest_dir/$name"-*.tar.gz)
count=${#backups[@]}
if (( dry_run )); then
    count=$(( count + 1 ))   # the backup that a real run would have created
fi
excess=$(( count - keep ))
for (( i = 0; i < excess; i++ )); do
    if (( dry_run )); then
        log "dry run: would delete ${backups[i]}"
    else
        rm -- "${backups[i]}"
        log "deleted ${backups[i]}"
    fi
done
```

**Что происходит:**

1. `prog=${0##*/}` берёт `$0` (имя скрипта, как его набрали) и отрезает всё до последнего `/`, поэтому сообщения говорят `backup.sh`. `${name##шаблон}` удаляет самое длинное совпадение с началом.
2. `usage` печатает here-документ (`<<USAGE ... USAGE`): строки до слова `USAGE` — это текст. `log` и `die` печатают в **стандартный поток ошибок** (`>&2`, [Урок 11.3](03-text-tools-pipes-and-redirection.md)), поэтому сообщения никогда не смешиваются с настоящим выводом. `die` завершается с кодом 2 — обычный код «неправильное использование».
3. `getopts ':k:nh' opt` — встроенный разборщик параметров bash. `k:` означает «параметру `-k` нужно значение» (оно приходит в `$OPTARG`); `n` и `h` — простые переключатели; ведущее `:` позволяет нам самим сообщать об ошибках. `case` обрабатывает каждую букву.
4. `shift $((OPTIND - 1))` убирает обработанные параметры, так что `$1` и `$2` — два оставшихся аргумента. Мы требуем ровно два.
5. `[[ $keep =~ ^[1-9][0-9]*$ ]]` проверяет регулярным выражением, что `-k` — положительное целое число. **Проверяйте входные данные, прежде чем их использовать** (Урок 10.4).
6. Новый архив пишется во временный файл `mktemp` в **той же папке**, и только когда `tar` отработал успешно, `mv` переименовывает его. Переименование внутри одной папки атомарно, поэтому никто никогда не увидит недописанный `.tar.gz`; а если `tar` упадёт, trap на `EXIT` удалит временный файл.
7. `shopt -s nullglob` заставляет шаблон без совпадений раскрыться в **ничто** (а не в текст шаблона). `backups=(...)` сохраняет подходящие имена в **массиве**. В именах есть метка времени вида год-месяц-день-время, поэтому сортировка по алфавиту одновременно сортирует их по возрасту: самые старые идут первыми.
8. `excess` — сколько лишних копий. Цикл удаляет `backups[0]`, `backups[1]`, ... (самые старые). В пробном запуске нового файла ещё нет, поэтому мы прибавляем 1 к количеству.
9. `BACKUP_STAMP` позволяет тесту зафиксировать метку времени; обычно её даёт `date`.

Подготовьте маленькую лабораторию и сначала попробуйте пробный запуск (метка времени в выводе — время вашего запуска):

```bash
mkdir -p "/tmp/lab/my project" && echo hi > "/tmp/lab/my project/a.txt"
cd /tmp/lab
bash ~/scripts/backup.sh -n "my project" out
ls out
```

```text
backup.sh: dry run: would create out/my project-20261118-071250.tar.gz
ls: cannot access 'out': No such file or directory
```

Пробный запуск ничего не создал, даже папку `out`. Неправильный ввод отклоняется с кодом 2 (попробуйте `-k 0` или несуществующую папку). Теперь сделаем пять копий с фиксированными метками и оставим три:

```bash
for day in 1 2 3 4 5; do
  BACKUP_STAMP=2026111$day-100000 bash ~/scripts/backup.sh -k 3 "my project" out
done
ls out
```

```text
backup.sh: created out/my project-20261111-100000.tar.gz
backup.sh: created out/my project-20261112-100000.tar.gz
backup.sh: created out/my project-20261113-100000.tar.gz
backup.sh: created out/my project-20261114-100000.tar.gz
backup.sh: deleted out/my project-20261111-100000.tar.gz
backup.sh: created out/my project-20261115-100000.tar.gz
backup.sh: deleted out/my project-20261112-100000.tar.gz
```

а `ls out` показывает три самые новые:

```text
my project-20261113-100000.tar.gz
my project-20261114-100000.tar.gz
my project-20261115-100000.tar.gz
```

Уберите лабораторию: `rm -r /tmp/lab`.

## 7. Проверка и тестирование скриптов

**Проверка синтаксиса.** `bash -n script.sh` читает скрипт, не выполняя его, и сообщает о синтаксических ошибках. Для скрипта, у которого у `if` нет `fi`:

```text
syn.sh: line 4: syntax error: unexpected end of file from `if' command on line 2
```

**Трассировка.** `set -x` заставляет bash печатать каждую команду после раскрытия с `+` в начале, пока не будет `set +x`. Включайте её вокруг строк, в которых сомневаетесь.

**ShellCheck** — самый полезный помощник. Запуск на небезопасных `quoting.sh` и `esurprise.sh` даёт (сокращённо):

```text
In quoting.sh line 7:
for f in $(ls); do echo "item: [$f]"; done
         ^---^ SC2045 (error): Iterating over ls output is fragile. Use globs.

In esurprise.sh line 12:
f() { local out=$(false); echo "3: local hid the failure, out=[$out]"; }
            ^-^ SC2155 (warning): Declare and assign separately to avoid masking return values.
```

Он нашёл ошибки из разделов 3 и 4, ничего не запуская. Устанавливать его не нужно: авторы публикуют образ на Docker Hub ([Урок 11.12](12-containers-and-docker-basics.md) объясняет Docker; пока просто выполните строку). Запустите это на вашем **Mac** в папке со скриптами:

```bash
docker run --rm -v "$PWD":/mnt koalaman/shellcheck:stable backup.sh
```

`backup.sh` не даёт никакого вывода: он чист. (В ВМ его можно установить командой `sudo apt-get install -y shellcheck`; `apt` объясняется в [Уроке 11.7](07-package-managers.md).) Исправьте каждое предупреждение или заглушите одно осознанно комментарием `# shellcheck disable=SC2086` с указанием причины.

> 🧪 **Тестирование:** скрипт заслуживает теста, как любая программа. Тест ниже запускает `backup.sh` во временной папке и сравнивает результаты. Сохраните его как `test-backup.sh` рядом со скриптом:

```bash
#!/usr/bin/env bash
# test-backup.sh - tests for backup.sh. Run it with: bash test-backup.sh
set -u

here=$(cd "$(dirname "$0")" && pwd)
script=$here/backup.sh
failures=0

check() {   # check "description" expected actual
    if [[ $2 == "$3" ]]; then
        echo "ok   - $1"
    else
        echo "FAIL - $1 (expected [$2], got [$3])"
        failures=$((failures + 1))
    fi
}

count_files() {   # count_files DIR: how many entries are in DIR
    local entries=("$1"/*)
    echo "${#entries[@]}"
}

work=$(mktemp -d)
trap 'rm -rf "$work"' EXIT
mkdir -p "$work/data dir"
echo hello > "$work/data dir/a.txt"

# 1. Five runs with "keep 2" leave exactly two archives.
for n in 1 2 3 4 5; do
    BACKUP_STAMP=20260101-00000$n "$script" -k 2 "$work/data dir" "$work/out" 2> /dev/null
done
check "keeps only 2" 2 "$(count_files "$work/out")"

# 2. A dry run changes nothing.
"$script" -n "$work/data dir" "$work/out" 2> /dev/null
check "dry run changes nothing" 2 "$(count_files "$work/out")"

# 3. Bad input gives status 2.
"$script" -k abc "$work/data dir" "$work/out" 2> /dev/null
check "bad count gives status 2" 2 "$?"

if (( failures > 0 )); then
    echo "$failures test(s) failed"
    exit 1
fi
echo "all tests passed"
```

```bash
bash test-backup.sh
```

```text
ok   - keeps only 2
ok   - dry run changes nothing
ok   - bad count gives status 2
all tests passed
```

**Что происходит:** `check` сравнивает ожидаемое и фактическое значение и считает сбои. В этом тестовом скрипте **нет** `set -e` (неудачный вызов `backup.sh` — часть теста, а не повод остановиться), но есть `set -u`. Завершающий `exit 1` делает тест пригодным для задания CI. Тест, который никогда не падает, ничего не доказывает: чтобы убедиться, что он работает, нарочно сломайте удаление старых копий (например, поставьте `excess=0` в копии `backup.sh`), и тест сообщит `FAIL - keeps only 2 (expected [2], got [5])`.

## Частые ошибки

1. **Нет кавычек вокруг переменной.** `rm $file` ломается на пробелах и шаблонах. *Исправление:* `rm -- "$file"`; запускайте ShellCheck.
2. **`cd` без проверки.** `cd "$dir"; rm -rf *` удаляет в неправильной папке, если `cd` не сработал. *Исправление:* `set -e` или `cd "$dir" || exit 1`, а лучше `rm -rf -- "${dir:?}/"*` или пути, для которых `cd` не нужен.
3. **Фиксированное имя временного файла** вроде `/tmp/out.txt`. *Исправление:* `mktemp` и `trap ... EXIT`, который его удаляет.
4. **`set -e` в скрипте с `grep`**, когда код 1 принимают за сбой. *Исправление:* `if grep -q ...; then` или `grep ... || true`.
5. **Сообщения в стандартный вывод.** Они попадают внутрь вывода, который читает другая программа. *Исправление:* ошибки и журнал — в стандартный поток ошибок (`>&2`).

## Попробуйте сами

Делайте это из `~/scripts` внутри ВМ.

1. Напишите `hello.sh` со строгим режимом, который печатает `Hello, NAME!`, где NAME — `$1`, а при отсутствии аргумента печатает строку об использовании в стандартный поток ошибок и завершается с кодом 2. Проверьте оба случая через `echo $?`.
2. Выполните `bash -n hello.sh`, затем `bash -x hello.sh Ann`.
3. Уберите кавычки вокруг `"$1"`, вызовите скрипт с `"Ann Lee"` и спросите у ShellCheck, что он об этом думает.

## Упражнения

### Упражнение 1 — Найдите три ошибки (легко)

Этот скрипт должен печатать количество строк каждого файла `.txt` в папке, заданной как `$1`. В нём три проблемы, которые исправили бы строгий режим или кавычки. Назовите их и напишите исправленный скрипт.

```bash
#!/bin/bash
for f in $(ls $1/*.txt); do
    n=$(wc -l < $f)
    echo $f $n
done
```

<details>
<summary>Подсказка</summary>

Подумайте о: имени папки или файла с пробелом; отсутствующем аргументе; папке без файлов `.txt`.

</details>

<details>
<summary>Ответ</summary>

```bash
#!/usr/bin/env bash
set -euo pipefail

dir=${1:?usage: count-lines.sh DIR}
shopt -s nullglob
for f in "$dir"/*.txt; do
    n=$(wc -l < "$f")
    printf '%s %s\n' "$f" "$n"
done
```

Почему это работает: `${1:?...}` останавливает скрипт с сообщением, если аргумента нет (без этого `$1/*.txt` стало бы `/*.txt`). Шаблон `"$dir"/*.txt` заменяет `ls`, сохраняет пробелы в именах, а `nullglob` заставляет цикл выполниться ноль раз, если совпадений нет (иначе цикл выполнился бы один раз с текстом `*.txt`). `"$f"` взято в кавычки в `wc` и `printf`.

</details>

### Упражнение 2 — Безопасный `cleanup-old.sh` и его тест (средне)

Напишите `cleanup-old.sh DIR DAYS`, который **печатает** (не удаляет) файлы в `DIR`, изменённые более `DAYS` дней назад, используя `find`. Он должен отклонять отсутствующую `DIR` и нечисловое `DAYS` с кодом 2. Затем напишите `test-cleanup-old.sh`: во временной папке `mktemp -d` создайте один старый файл (`touch -d '10 days ago'`) и один новый и проверьте, что `DAYS=5` печатает только старый файл, `DAYS=20` не печатает ничего, а `DAYS=abc` даёт код 2.

<details>
<summary>Подсказка</summary>

`find "$dir" -type f -mtime +"$days" -print`. Проверяйте через `[[ $days =~ ^[0-9]+$ ]]`. В тесте перехватывайте вывод через `out=$(bash cleanup-old.sh "$work" 5)` и используйте `check` из `test-backup.sh`.

</details>

<details>
<summary>Ответ</summary>

```bash
#!/usr/bin/env bash
set -euo pipefail

die() { echo "cleanup-old.sh: $*" >&2; exit 2; }

[[ $# -eq 2 ]] || die "usage: cleanup-old.sh DIR DAYS"
dir=$1
days=$2
[[ -d $dir ]] || die "not a directory: $dir"
[[ $days =~ ^[0-9]+$ ]] || die "DAYS must be a number, got '$days'"

find "$dir" -type f -mtime +"$days" -print
```

и тест:

```bash
#!/usr/bin/env bash
set -u

here=$(cd "$(dirname "$0")" && pwd)
failures=0
check() {
    if [[ $2 == "$3" ]]; then echo "ok   - $1"; else echo "FAIL - $1 (expected [$2], got [$3])"; failures=$((failures + 1)); fi
}

work=$(mktemp -d)
trap 'rm -rf "$work"' EXIT
touch -d '10 days ago' "$work/old.txt"
touch "$work/new.txt"

check "5 days lists the old file" "$work/old.txt" "$(bash "$here/cleanup-old.sh" "$work" 5)"
check "20 days lists nothing" "" "$(bash "$here/cleanup-old.sh" "$work" 20)"
bash "$here/cleanup-old.sh" "$work" abc 2> /dev/null
check "bad DAYS gives status 2" 2 "$?"

(( failures == 0 )) || { echo "$failures test(s) failed"; exit 1; }
echo "all tests passed"
```

Запуск теста печатает три строки `ok` и `all tests passed`.

Почему это работает: каждый вход проверяется перед использованием, ошибки идут в стандартный поток ошибок с кодом 2, а скрипт только печатает, поэтому ошибка не может ничего удалить. `-mtime +N` значит «изменён более N дней назад». Временная папка приватна и удаляется trap-ом, даже если проверка упала, а `touch -d` делает файл «старым» без ожидания.

</details>

## Вопросы для самопроверки

1. Что делает `set -e` и назовите одну ситуацию, в которой он не останавливает скрипт.

<details>
<summary>Ответ</summary>

Он завершает скрипт, когда простая команда не срабатывает. Он не останавливает команду, использованную как условие `if`, слева от `||` или `&&`, или в неудачном `local x=$(cmd)` (используется код `local`).

</details>

2. Какой код возврата у конвейера `false | cat` с `pipefail` и без него?

<details>
<summary>Ответ</summary>

Без `pipefail` — 0 (код последней команды, `cat`). С `pipefail` — 1.

</details>

3. В чём разница между `$@`, `"$@"` и `"$*"`?

<details>
<summary>Ответ</summary>

`"$@"` раскрывается во все аргументы, каждый как отдельное слово (правильно для передачи аргументов дальше). `"$*"` склеивает их в одно слово. `$@` и `$*` без кавычек снова режутся по пробелам, что ломает аргументы с пробелами.

</details>

4. Когда срабатывает trap на `EXIT`?

<details>
<summary>Ответ</summary>

Всякий раз, когда скрипт завершается: на последней строке, при `exit`, после ошибки, остановившей его через `set -e`, и после сигнала, превращённого другим trap в `exit`. Он не срабатывает, если скрипт убит сигналом `SIGKILL`.

</details>

5. Почему `backup.sh` пишет во временный файл `mktemp`, а затем делает `mv`?

<details>
<summary>Ответ</summary>

Переименование внутри одной папки атомарно, поэтому читатель никогда не увидит недописанный архив. Если `tar` упадёт, временный файл удалит trap, и под итоговым именем не останется испорченной копии.

</details>

## Итоги

- Без строгого режима скрипт на bash продолжает работу после ошибок и может сообщить об успехе. Начинайте с `set -euo pipefail`.
- У `set -e` есть исключения (условия, `||`, `local x=$(...)`, `((i++))` при нуле); ожидаемые сбои обрабатывайте сами.
- Берите в кавычки каждую переменную и `$(...)`, передавайте аргументы через `"$@"`, заканчивайте опции через `--` и используйте `[[ ]]` для проверок.
- Для временных файлов используйте `mktemp`, а `trap ... EXIT` гарантирует уборку; превращайте `INT` и `TERM` в `exit`.
- У хорошего скрипта есть `-h`, он проверяет входные данные, пишет сообщения в стандартный поток ошибок, завершается с кодом 2 при неправильном использовании и предлагает пробный запуск, если что-то удаляет.
- Проверяйте через `bash -n`, `set -x` и ShellCheck и пишите для скрипта тестовый скрипт.

## Источники и что почитать

- [GNU Bash Reference Manual — The Set Builtin](https://www.gnu.org/software/bash/manual/html_node/The-Set-Builtin.html) — официальное описание `set -e`, `-u` и `pipefail`.
- [GNU Bash Reference Manual — Shell Expansions](https://www.gnu.org/software/bash/manual/html_node/Shell-Expansions.html) — разбиение на слова, раскрытие имён файлов и кавычки подробно.
- [GNU Bash Reference Manual — Bourne Shell Builtins (trap, getopts)](https://www.gnu.org/software/bash/manual/html_node/Bourne-Shell-Builtins.html) — `trap`, `getopts`, `shift` и остальные.
- [ShellCheck](https://www.shellcheck.net/) — онлайн-проверка и страница вики для каждого кода предупреждения, например SC2086.
- [mktemp(1) manual page](https://man7.org/linux/man-pages/man1/mktemp.1.html) — как создаются временные файлы и папки.

---

[🏠 Начало](../START-HERE.md) · [← 11.8 Основы скриптов на bash](08-bash-scripting-basics.md) · [📋 Этап 11](README.md) · [11.10 SSH и удалённые серверы →](10-ssh-and-remote-servers.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
