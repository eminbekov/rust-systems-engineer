<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 06.11 Проект: многопоточный веб-сервер](11-project-multithreaded-web-server.md) · [📋 Этап 06](README.md) · [Этап 07 — Алгоритмы и структуры данных →](../07-algorithms-and-data-structures/README.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/06-rust-advanced/12-checkpoint-stage-1-complete.md) <!-- nav -->

# Урок 06.12 — Контрольная точка: стадия 1 пройдена

> **Этап 06 — Rust: продвинутый уровень** · Урок 12 из 12 · ⏱ около 3 часов (2 занятия)

## Что вы узнаете

- Действительно ли вы владеете материалом Стадии 1 (Этапы 00–06): инструменты, Git, язык Rust от переменных до async и FFI.
- Как проверить состояние всего вашего учебного репозитория: историю, теги, игнорируемые файлы и ворота качества в каждом проекте.
- Какие темы повторить, с прямыми ссылками на уроки.
- Какие инструменты и привычки взять с собой в Стадию 2.

## Перед началом

- Вы прошли [Урок 06.11 — Проект: многопоточный веб-сервер](11-project-multithreaded-web-server.md) и слили ветку `p06-webserver`.
- Код прежних проектов лежит в `~/projects/rust/code/`.
- Запланируйте около 90 минут. Сначала пройдите тест **не заглядывая в уроки**. Запишите ответы на бумаге или в файле заметок, затем откройте скрытые ответы. Неверный ответ полезен: он подсказывает, что повторить.

## Новые слова

| Слово | Значение |
|---|---|
| **контрольная точка** (checkpoint) | Урок, который проверяет, что вы выучили, а не учит чему-то новому. |
| **Стадия** (Stage) | Одна из пяти больших групп этапов этого курса. Стадия 1 — «Фундамент: инструменты, Git и язык Rust» (Этапы 00–06). |
| **ворота качества** (quality gate) | Проверка, которую должно пройти каждое изменение, прежде чем его примут: здесь `cargo fmt --check`, `cargo clippy -- -D warnings` и `cargo test`. |
| **тег** (tag) | Постоянная именованная метка на одном коммите в Git, например `stage-1-complete`. |
| **ретроспектива** (retrospective) | Короткий взгляд назад: что получилось хорошо, что было трудно, что изменить. |
| **портфолио** (portfolio) | Собрание ваших работ, которое можно показать другим людям: ваш репозиторий с проектами, README и тестами. |
| **CI** (Continuous Integration, непрерывная интеграция) | Сервер, который автоматически запускает ваши ворота качества при каждом push. Вы настроите его на Этапе 09. |
| **интервальное повторение** (spaced repetition) | Метод учёбы: вы повторяете тему снова через день, потом через неделю, потом через месяц, чтобы помнить её долго. |

## Стадия 1 одним взглядом

| Этап | Что вы изучили | Ваш главный проект |
|---|---|---|
| 00 Начало пути | Что такое программирование, как компьютер выполняет программу, терминал, Rust и RustRover, чтение ошибок | Hello, world |
| 01 Git и контроль версий | Коммиты, история, отмена, ветки, конфликты, GitHub и GitLab, pull request, соглашения | Ваш учебный репозиторий |
| 02 Основы программирования | Переменные, типы, функции, ветвления и циклы, ввод, тесты, отладка | Игра «Угадай число» |
| 03 Владение и заимствование | Стек и куча, владение, перемещение/копирование/клонирование, ссылки, срезы | Счётчик слов |
| 04 Структуры, перечисления, коллекции, ошибки | Структуры, перечисления, `Option`, `match`, модули, `Vec`, `String`, `HashMap`, `Result` | Телефонная книга |
| 05 Rust: средний уровень | Обобщения, трейты, времена жизни, замыкания, итераторы, документация, Cargo, умные указатели, потоки, крейты, бенчмарки | minigrep |
| 06 Rust: продвинутый уровень | Трейт-объекты, продвинутые трейты, образцы, внутренняя изменяемость, unsafe и Miri, макросы, async, FFI (Foreign Function Interface, интерфейс внешних функций) | Многопоточный веб-сервер |

## Вопросы для самопроверки

Отвечайте по памяти. Каждый ответ скрыт; откройте его после того, как запишете свой.

**Этапы 00–01: инструменты и Git**

1. Чем исходный код отличается от бинарного файла и какой инструмент превращает одно в другое в проекте на Rust?

<details>
<summary>Ответ</summary>

Исходный код — это текст, который вы пишете и читаете. Бинарный файл — машинный код, который выполняет CPU (Central Processing Unit, центральный процессор — микросхема, исполняющая инструкции). Компилятор `rustc` переводит одно в другое; обычно его вызывают через Cargo: `cargo build` или `cargo run`.

</details>

2. Чем отличаются `git add` и `git commit`?

<details>
<summary>Ответ</summary>

`git add` кладёт текущую версию выбранных файлов в область индексации («то, что попадёт в следующий снимок»). `git commit` берёт всё из области индексации и записывает как новый снимок (коммит) с сообщением в истории.

</details>

3. Вы отправили (push) коммит с ошибкой. Нужно использовать `git revert` или `git reset` и почему?

<details>
<summary>Ответ</summary>

`git revert <commit>`. Он создаёт **новый** коммит, который отменяет плохой, поэтому общая история не переписывается и чужие копии остаются валидными. `git reset` двигает ветку назад и переписывает историю; используйте его только для коммитов, которых нет больше ни у кого.

</details>

4. Как конфликт слияния выглядит в файле и как завершить слияние?

<details>
<summary>Ответ</summary>

Git записывает обе версии между маркерами: `<<<<<<< HEAD`, `=======`, `>>>>>>> branch`. Вы редактируете файл до нужной версии и удаляете маркеры, выполняете `git add <file>`, а затем `git commit` (или `git merge --continue`).

</details>

5. Почему в корневом `.gitignore` вашего учебного репозитория есть `target/` и что происходит, когда вы выполняете `cargo new` внутри существующего репозитория?

<details>
<summary>Ответ</summary>

В `target/` лежит результат сборки: он большой, создан автоматически и может быть пересобран, поэтому ему не место в истории. Когда `cargo new` запускается внутри существующего репозитория, он **не** создаёт `.gitignore` для нового пакета, так что корневой `.gitignore` уже должен игнорировать `target/`. Проверить можно командой `git check-ignore -v code/<project>/target`.

</details>

**Этап 02: основы**

6. Что возвращают `checked_add`, `wrapping_add` и `saturating_add` для `255u8` плюс `1`?

<details>
<summary>Ответ</summary>

`checked_add` возвращает `None` (сообщает о переполнении), `wrapping_add` возвращает `0` (оборачивается), `saturating_add` возвращает `255` (остаётся на максимуме). Обычный `+` паникует при переполнении в отладочной сборке.

```text
None
0
255
```

</details>

**Этап 03: владение и заимствование**

7. Почему это не компилируется? В чём правило?

```rust
let mut v = vec![1, 2, 3];
let first = &v[0];
v.push(4);
println!("{first}");
```

<details>
<summary>Ответ</summary>

```text
error[E0502]: cannot borrow `v` as mutable because it is also borrowed as immutable
```

В любой момент у вас может быть **либо** любое число общих ссылок (`&T`), **либо** ровно одна изменяемая ссылка (`&mut T`), но не обе сразу. `first` — общее заимствование, которое используется позже, а `push` требует изменяемого заимствования. Это важно: `push` может перевыделить вектор, и тогда `first` указывал бы на освобождённую память. Исправление: используйте `first` до `push` или скопируйте значение (`let first = v[0];`).

</details>

8. Что здесь не так и какие есть два исправления?

```rust
let a = String::from("hi");
let b = a;
println!("{a} {b}");
```

<details>
<summary>Ответ</summary>

```text
error[E0382]: borrow of moved value: `a`
```

`String` владеет памятью в куче и не реализует `Copy`, поэтому `let b = a;` **перемещает** владение в `b`; `a` больше нельзя использовать. Исправления: `let b = a.clone();` (независимая копия) или заимствовать вместо перемещения: `let b = &a;`.

</details>

**Этап 04: структуры, перечисления, ошибки**

9. Что делает оператор `?` в `let n: i32 = text.trim().parse()?;` внутри функции, возвращающей `Result<i32, ParseIntError>`?

<details>
<summary>Ответ</summary>

Если значение `Ok(n)`, `?` извлекает его и продолжает. Если `Err(e)`, `?` досрочно возвращается из функции с этой ошибкой (при необходимости преобразованной через `From`). Тип возвращаемого значения функции должен быть совместимым `Result` (или `Option`). Настоящий вывод для `double(" 21 ")` и `double("x")`:

```text
Ok(42)
Err(ParseIntError { kind: InvalidDigit })
```

</details>

**Этап 05: средний уровень**

10. Почему это не компилируется и как исправить?

```rust
fn longest(a: &str, b: &str) -> &str {
    if a.len() > b.len() { a } else { b }
}
```

<details>
<summary>Ответ</summary>

```text
error[E0106]: missing lifetime specifier
```

Результат — ссылка, но компилятор не знает, заимствована ли она у `a` или у `b`. Это сообщает параметр времени жизни: `fn longest<'a>(a: &'a str, b: &'a str) -> &'a str`. Он означает: результат действителен, пока действительны оба входа.

</details>

11. Что напечатает эта программа и почему?

```rust
let doubled = (1..4).map(|x| {
    println!("map {x}");
    x * 2
});
println!("before");
let total: i32 = doubled.sum();
println!("total {total}");
```

<details>
<summary>Ответ</summary>

```text
before
map 1
map 2
map 3
total 12
```

Адаптеры итераторов вроде `map` **ленивы**: они ничего не делают, пока итератор что-то не потребит. `sum()` потребляет его, и только тогда выполняется замыкание.

</details>

12. Что напечатает этот код и какой трейт замыкания реализует `inc`?

```rust
let mut count = 0;
let mut inc = || {
    count += 1;
    count
};
println!("{} {}", inc(), inc());
println!("{count}");
```

<details>
<summary>Ответ</summary>

```text
1 2
2
```

`inc` изменяет захваченную переменную, поэтому реализует `FnMut` (его можно вызывать много раз, и он меняет состояние), а привязка должна быть `mut`. После последнего вызова изменяемое заимствование `count` заканчивается, так что `count` можно напечатать.

</details>

13. Почему данные между потоками делят через `Arc<Mutex<T>>`, а не через `Rc<RefCell<T>>`?

<details>
<summary>Ответ</summary>

`Rc` использует неатомарный счётчик и не `Send`, а `RefCell` не `Sync`: компилятор отказывается перемещать их между потоками. У `Arc` атомарный счётчик, а `Mutex` даёт исключительный доступ с настоящей блокировкой, поэтому эта пара безопасна для потоков.

</details>

**Этап 06: продвинутый уровень**

14. Когда выбрать `fn f<T: Shape>(x: &T)`, а когда `fn f(x: &dyn Shape)`? Каков размер `&dyn Shape`?

<details>
<summary>Ответ</summary>

Обобщённая версия использует **статическую диспетчеризацию**: компилятор делает отдельную копию для каждого типа (мономорфизация), поэтому вызовы прямые и могут быть встроены, но код растёт и значения в одном вызове должны быть одного типа. `&dyn Shape` использует **динамическую диспетчеризацию**: одна копия кода, вызов через vtable во время выполнения, и значения разных типов могут жить в одной коллекции (`Vec<Box<dyn Shape>>`). `&dyn Trait` — **жирный указатель** из двух слов (указатель на данные и указатель на vtable):

```text
&u8: 8
&[u8]: 16
&dyn Debug: 16
Box<dyn Debug>: 16
```

</details>

15. Что произойдёт при выполнении этого кода и чему он учит?

```rust
let cell = RefCell::new(1);
let a = cell.borrow_mut();
let b = cell.borrow_mut();
```

<details>
<summary>Ответ</summary>

Он компилируется, но паникует во время выполнения:

```text
thread 'main' panicked at src/main.rs:6:18:
RefCell already borrowed
```

`RefCell` переносит правила заимствования из времени компиляции во время выполнения. Вы можете менять данные через общую ссылку, но расплачиваетесь возможной паникой при нарушении правил. Используйте `Cell` для небольших значений `Copy` (проверки заимствований не нужны), `RefCell`, когда нужны ссылки внутрь, и `Mutex` для потоков.

</details>

16. Назовите пять вещей, которые может только `unsafe`-код. Что такое комментарий `// SAFETY:` и что делает Miri?

<details>
<summary>Ответ</summary>

(1) Разыменовать сырой указатель, (2) вызвать `unsafe`-функцию, (3) обратиться к изменяемой `static` или изменить её, (4) реализовать `unsafe`-трейт, (5) обратиться к полю `union`. `unsafe` не отключает проверку заимствований. Комментарий `// SAFETY:` над каждым блоком `unsafe` объясняет, почему там соблюдаются правила. Miri запускает ваши тесты в интерпретаторе и сообщает о неопределённом поведении (висячие указатели, выход за границы, нарушения алиасинга); вызывать внешний код на C он не умеет.

</details>

17. Назовите два семейства макросов и три вида процедурных макросов.

<details>
<summary>Ответ</summary>

Декларативные макросы (`macro_rules!`: образцы и повторения) и процедурные макросы (функции Rust из `TokenStream` в `TokenStream`, в крейте с `proc-macro = true`). Процедурные виды: похожие на функцию (`name!(...)`), derive (`#[derive(Name)]`) и атрибутные (`#[name]`).

</details>

18. Что возвращает `async fn` при вызове? Почему нельзя вызывать `std::thread::sleep` внутри async-кода?

<details>
<summary>Ответ</summary>

Она возвращает future; тело не выполняется, пока future не опросят (например, через `.await`). `std::thread::sleep` блокирует весь поток, и рантайм не может запускать другие задачи на нём; используйте `tokio::time::sleep(...).await` или `spawn_blocking` для блокирующей работы. В эксперименте две задачи по 300 мс на одном потоке заняли около 600 мс с блокирующим сном и около 300 мс с async-сном.

</details>

19. Зачем в FFI использовать `CString` для передачи текста в C и кто должен освобождать память, которую C выделил через `malloc`?

<details>
<summary>Ответ</summary>

Строки C заканчиваются NUL-байтом (байтом со значением `0`) и не могут содержать NUL внутри; `CString::new` добавляет NUL и отвергает текст с NUL внутри. Память, выделенную в C, должна освобождать парная функция C (`free` или собственная функция библиотеки `free_...`), а не аллокатор Rust. Обычно это делает тип-обёртка в `Drop`.

</details>

20. В проекте веб-сервера: почему получатель канала обёрнут в `Arc<Mutex<...>>`, почему очередь ограничена и как останавливается пул?

<details>
<summary>Ответ</summary>

У канала `mpsc` один получатель, а брать задания должны многие рабочие потоки: `Arc` делит его, а `Mutex` позволяет получать одному рабочему потоку за раз (блокировку нужно освободить до выполнения задания). Очередь ограничена ради обратного давления: когда она полна, цикл приёма отвечает `503` вместо использования неограниченной памяти. Чтобы остановиться, `Drop` уничтожает отправителя; тогда `recv()` в каждом рабочем потоке возвращает `Err`, рабочие потоки выходят из циклов, а `Drop` присоединяет (join) потоки.

</details>

## Практические задания

Выполните их на своём компьютере. У каждого задания есть подсказка и проверенный ответ.

### Задание 1 — Проверьте состояние учебного репозитория

В `~/projects/rust`: (а) убедитесь, что `git status` чист, а `target/` игнорируется; (б) посмотрите последние коммиты; (в) отметьте конец Стадии 1 аннотированным тегом `stage-1-complete`; (г) если у вас есть удалённый репозиторий (Урок 01.8), отправьте ветку и теги.

<details>
<summary>Подсказка</summary>

`git status`, `git check-ignore -v code/<project>/target`, `git log --oneline --decorate -n 5`, `git tag -a <name> -m "<message>"` и `git push origin main --tags`. Если `git status` показывает изменения, сначала закоммитьте их с хорошим сообщением.

</details>

<details>
<summary>Ответ</summary>

```sh
cd ~/projects/rust
git status
git check-ignore -v code/p06-webserver/target
git log --oneline --decorate -n 5
git tag -a stage-1-complete -m "Stage 1 complete: phases 00-06"
git tag -n
git push origin main --tags     # only if you have a remote called origin
```

Ожидаемая форма вывода (ваши хеши и имена будут другими):

```text
On branch main
nothing to commit, working tree clean
.gitignore:1:target/	code/p06-webserver/target
3363654 (HEAD -> main, tag: stage-1-complete, tag: p06-webserver-v1.0) Merge branch 'p06-webserver': multithreaded web server
b675a56 chore: start learning repository
ed5a1e7 (p06-webserver) feat(p06-webserver): M3 thread pool
p06-webserver-v1.0 p06-webserver: all milestones done
stage-1-complete   Stage 1 complete: phases 00-06
```

Почему это работает: чистое дерево означает, что всё сохранено в истории. `git check-ignore -v` печатает, какое правило игнорирует путь (если ничего не напечатано, добавьте `target/` в `.gitignore`). **Аннотированный** тег (`-a`) хранит сообщение, автора и дату, и это правильный вид тега для релизов и вех. `git push` не отправляет теги, если не добавить `--tags`.

</details>

### Задание 2 — Одна команда, которая проверяет каждый проект

Напишите `scripts/check-all.sh` в корне вашего учебного репозитория. Для каждой папки `code/*/`, где есть `Cargo.toml`, он запускает `cargo fmt --check`, `cargo clippy --all-targets -- -D warnings` и `cargo test`. Он печатает `== <folder>`, а затем `OK` или `FAILED` и завершается с ненулевым кодом, если какой-то проект не прошёл. Запустите его и исправьте найденное.

<details>
<summary>Подсказка</summary>

Делайте цикл `for manifest in code/*/Cargo.toml`. Используйте `dirname`, чтобы получить папку, и подоболочку `( cd "$dir" && ... )`, чтобы `cd` не влиял на цикл. Соедините три команды через `&&` внутри `if`. Ведите переменную `status`, которую устанавливайте в `1` при сбое, и в конце делайте `exit "$status"`.

</details>

<details>
<summary>Ответ</summary>

```sh
#!/usr/bin/env bash
# Runs the quality gates in every Cargo project under code/.
# Usage, from the repository root:  bash scripts/check-all.sh
status=0
for manifest in code/*/Cargo.toml; do
    [ -f "$manifest" ] || continue
    dir=$(dirname "$manifest")
    echo "== $dir"
    if (cd "$dir" && cargo fmt --check && cargo clippy --all-targets -q -- -D warnings && cargo test -q >/dev/null); then
        echo "   OK"
    else
        echo "   FAILED"
        status=1
    fi
done
exit $status
```

Запустите `bash scripts/check-all.sh`. На трёх небольших проектах, в одном из которых есть предупреждение clippy, настоящий запуск напечатал:

```text
== code/p02-hello
   OK
== code/p03-words
   OK
== code/p04-book
error: length comparison to zero
 --> src/main.rs:3:13
  |
3 |     let n = names.len() == 0;
  |             ^^^^^^^^^^^^^^^^ help: using `is_empty` is clearer and more explicit: `names.is_empty()`
  |
  = note: `-D clippy::len-zero` implied by `-D warnings`
error: could not compile `p04-book` (bin "p04-book") due to 1 previous error
   FAILED
```

а скрипт завершился с кодом `1`. Исправьте каждую найденную проблему (`cargo fmt` сам исправляет форматирование), запускайте снова, пока каждый проект не покажет `OK`, затем закоммитьте: `git add scripts/check-all.sh && git commit -m "chore: add check-all quality gate script"`.

Почему это работает: три команды — те же ворота, что вы использовали в каждом проекте. Скрипт даёт одну команду для всех, а код выхода делает его пригодным для CI (Этап 09), который запускает этот же скрипт при каждом push.

</details>

### Задание 3 — Исполнитель заданий: трейт-объекты, макрос и потоки

Напишите программу с трейтом `Job: Send` (методы `name(&self) -> String` и `run(&self) -> u64`), тремя типами заданий (`Square`, `SumTo`, `Fib`), макросом `jobs![...]`, который строит `Vec<Box<dyn Job>>`, и функцией `run_all`, которая запускает каждое задание в отдельном потоке, отправляет `(name, result)` через канал и возвращает отсортированные результаты. Добавьте тесты для обычного случая, пустого списка и `Fib`.

<details>
<summary>Подсказка</summary>

Макрос: `($($job:expr),* $(,)?) => {{ let list: Vec<Box<dyn Job>> = vec![$(Box::new($job)),*]; list }}`. Аннотация типа позволяет компилятору превратить каждый `Box<Square>` в `Box<dyn Job>`. `Job: Send` позволяет переместить `Box<dyn Job>` в `thread::spawn`. Клонируйте отправителя для каждого потока и уничтожьте (`drop`) оригинал, чтобы `rx.iter()` закончился.

</details>

<details>
<summary>Ответ</summary>

```rust
use std::sync::mpsc;
use std::thread;

trait Job: Send {
    fn name(&self) -> String;
    fn run(&self) -> u64;
}

struct Square(u64);
struct SumTo(u64);
struct Fib(u32);

impl Job for Square {
    fn name(&self) -> String {
        format!("square({})", self.0)
    }
    fn run(&self) -> u64 {
        self.0 * self.0
    }
}

impl Job for SumTo {
    fn name(&self) -> String {
        format!("sum_to({})", self.0)
    }
    fn run(&self) -> u64 {
        (1..=self.0).sum()
    }
}

impl Job for Fib {
    fn name(&self) -> String {
        format!("fib({})", self.0)
    }
    fn run(&self) -> u64 {
        let (mut a, mut b) = (0u64, 1u64);
        for _ in 0..self.0 {
            (a, b) = (b, a + b);
        }
        a
    }
}

macro_rules! jobs {
    ($($job:expr),* $(,)?) => {{
        let list: Vec<Box<dyn Job>> = vec![$(Box::new($job)),*];
        list
    }};
}

fn run_all(jobs: Vec<Box<dyn Job>>) -> Vec<(String, u64)> {
    let (tx, rx) = mpsc::channel();
    let mut handles = Vec::new();
    for job in jobs {
        let tx = tx.clone();
        handles.push(thread::spawn(move || {
            tx.send((job.name(), job.run())).unwrap();
        }));
    }
    drop(tx);
    let mut results: Vec<(String, u64)> = rx.iter().collect();
    for handle in handles {
        handle.join().unwrap();
    }
    results.sort();
    results
}

fn main() {
    let results = run_all(jobs![Square(12), SumTo(100), Fib(10), Fib(50)]);
    for (name, value) in results {
        println!("{name} = {value}");
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn every_job_reports_once() {
        let results = run_all(jobs![Square(3), SumTo(4)]);
        assert_eq!(
            results,
            vec![("square(3)".to_string(), 9), ("sum_to(4)".to_string(), 10)]
        );
    }

    #[test]
    fn empty_list_gives_empty_results() {
        assert!(run_all(jobs![]).is_empty());
    }

    #[test]
    fn fib_values() {
        assert_eq!(Fib(0).run(), 0);
        assert_eq!(Fib(1).run(), 1);
        assert_eq!(Fib(10).run(), 55);
    }
}
```

```text
fib(10) = 55
fib(50) = 12586269025
square(12) = 144
sum_to(100) = 5050
```

`cargo test` покажет `test result: ok. 3 passed`.

Почему это работает: трейт-объект позволяет трём разным типам жить в одном `Vec`. Супертрейт `Send` позволяет перемещать коробки в другие потоки. Канал собирает результаты в том порядке, в котором потоки заканчивают, поэтому сортировка делает вывод стабильным и тестируемым. Уничтожение исходного отправителя обязательно: `rx.iter()` заканчивается, только когда исчезли **все** отправители.

</details>

### Задание 4 — Безопасная обёртка над `qsort` из C

Используйте функцию библиотеки C `qsort`, чтобы отсортировать срез `i32`. Объявите её в блоке `unsafe extern "C"`, напишите обратный вызов `extern "C" fn compare_i32` и безопасную функцию `sort_ints(values: &mut [i32])` с комментариями `// SAFETY:`. Добавьте тесты для обычного среза, пустого среза и среза из одного элемента.

<details>
<summary>Подсказка</summary>

`void qsort(void *base, size_t count, size_t size, int (*compare)(const void *, const void *))`. В Rust типы указателей — `*mut c_void` и `*const c_void`. Базовый указатель получите через `values.as_mut_ptr().cast()`. В обратном вызове приведите два указателя обратно через `*(a as *const i32)` и верните `a.cmp(&b) as c_int`: `Ordering` преобразуется в `-1`, `0`, `1`.

</details>

<details>
<summary>Ответ</summary>

```rust
use std::ffi::{c_int, c_void};

unsafe extern "C" {
    fn qsort(
        base: *mut c_void,
        count: usize,
        size: usize,
        compare: extern "C" fn(*const c_void, *const c_void) -> c_int,
    );
}

extern "C" fn compare_i32(a: *const c_void, b: *const c_void) -> c_int {
    // SAFETY: `qsort` calls us with pointers to two elements of the array,
    // and the array holds `i32` values.
    let (a, b) = unsafe { (*(a as *const i32), *(b as *const i32)) };
    a.cmp(&b) as c_int
}

/// Sorts the numbers in place using the C library's `qsort`.
pub fn sort_ints(values: &mut [i32]) {
    // SAFETY: the pointer and the length describe exactly the slice, the
    // element size is the size of `i32`, and `compare_i32` matches the type.
    unsafe {
        qsort(
            values.as_mut_ptr().cast(),
            values.len(),
            std::mem::size_of::<i32>(),
            compare_i32,
        );
    }
}

fn main() {
    let mut numbers = [42, -7, 19, 0, 3, -7];
    sort_ints(&mut numbers);
    println!("{:?}", numbers);
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn sorts_numbers() {
        let mut values = [5, 3, 9, -1, 3];
        sort_ints(&mut values);
        assert_eq!(values, [-1, 3, 3, 5, 9]);
    }

    #[test]
    fn empty_and_single_are_fine() {
        let mut empty: [i32; 0] = [];
        sort_ints(&mut empty);
        let mut one = [7];
        sort_ints(&mut one);
        assert_eq!(one, [7]);
    }
}
```

```text
[-7, -7, 0, 3, 19, 42]
```

Почему это работает: срез — это «указатель + длина», ровно то, что хочет `qsort`. Размер элемента берётся из `size_of::<i32>()`, поэтому C перемещает целые числа. Обратный вызов — `extern "C"` и не паникует. Безопасная функция прячет весь `unsafe`: вызывающий код не может передать неверную длину или размер.

</details>

### Задание 5 — Конкурентные загрузки с ограничением времени

Напишите async-программу: `fetch(id)` спит `id * 100` мс и возвращает `page-<id>`. `fetch_with_limit(id, limit)` возвращает `Ok(page)` или `Err("page-<id> timed out")`, если лимит превышен. В `main` запустите загрузки для id от 1 до 5 **конкурентно** с лимитом 350 мс и выведите результаты в порядке id. Добавьте два теста `#[tokio::test]`.

<details>
<summary>Подсказка</summary>

Запускайте каждый `fetch_with_limit` через `tokio::spawn` и храните `JoinHandle` в `Vec`; ждите их по порядку. `tokio::time::timeout(limit, fetch(id)).await` даёт `Result<String, Elapsed>`. Добавьте Tokio командой `cargo add tokio --features rt-multi-thread,macros,time`.

</details>

<details>
<summary>Ответ</summary>

```rust
use std::time::Duration;
use tokio::time::{sleep, timeout};

async fn fetch(id: u64) -> String {
    sleep(Duration::from_millis(id * 100)).await;
    format!("page-{id}")
}

async fn fetch_with_limit(id: u64, limit: Duration) -> Result<String, String> {
    match timeout(limit, fetch(id)).await {
        Ok(page) => Ok(page),
        Err(_) => Err(format!("page-{id} timed out")),
    }
}

#[tokio::main]
async fn main() {
    let limit = Duration::from_millis(350);
    let mut handles = Vec::new();
    for id in 1..=5 {
        handles.push(tokio::spawn(fetch_with_limit(id, limit)));
    }
    for handle in handles {
        match handle.await.unwrap() {
            Ok(page) => println!("ok:      {page}"),
            Err(message) => println!("failed:  {message}"),
        }
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[tokio::test]
    async fn fast_fetch_succeeds() {
        let result = fetch_with_limit(1, Duration::from_millis(200)).await;
        assert_eq!(result, Ok("page-1".to_string()));
    }

    #[tokio::test]
    async fn slow_fetch_times_out() {
        let result = fetch_with_limit(10, Duration::from_millis(50)).await;
        assert_eq!(result, Err("page-10 timed out".to_string()));
    }
}
```

```text
ok:      page-1
ok:      page-2
ok:      page-3
failed:  page-4 timed out
failed:  page-5 timed out
```

`cargo test` покажет `test result: ok. 2 passed`.

Почему это работает: пять загрузок идут одновременно, поэтому программе нужно около 400 мс (самое долгое допустимое ожидание — 350 мс), а не 1500 мс. `timeout` уничтожает медленный future, что его отменяет. В тестах оставлены щедрые запасы (200 мс против 100 мс; 50 мс против 1000 мс), потому что слишком тугие тесты на время падают случайно.

</details>

## Если что-то не получилось

| Тема | Повторите этот урок |
|---|---|
| Исходный код, компилятор, бинарный файл | [Урок 00.3 — Как компьютер выполняет программу](../00-getting-started/03-how-a-computer-runs-a-program.md) |
| Чтение ошибок компилятора | [Урок 00.7 — Как читать ошибки и где искать помощь](../00-getting-started/07-reading-errors-and-getting-help.md) |
| `git add`, `git commit`, `.gitignore` | [Урок 01.3 — Первый репозиторий и коммиты](../01-git-and-version-control/03-your-first-repository-and-commits.md) |
| `git log`, `git diff` | [Урок 01.4 — История и просмотр изменений](../01-git-and-version-control/04-viewing-history-and-changes.md) |
| `git revert`, `git reset`, `git restore` | [Урок 01.5 — Как безопасно отменять изменения](../01-git-and-version-control/05-undoing-changes-safely.md) |
| Ветки и `git merge` | [Урок 01.6 — Ветки и слияние](../01-git-and-version-control/06-branches-and-merging.md) |
| Конфликты слияния | [Урок 01.7 — Конфликты слияния](../01-git-and-version-control/07-merge-conflicts.md) |
| Удалённые репозитории, `git push` | [Урок 01.8 — Удалённые репозитории: GitHub и GitLab](../01-git-and-version-control/08-remotes-github-and-gitlab.md) |
| Сообщения коммитов, pull request, теги | [Урок 01.9 — Командная работа: pull request, код-ревью и соглашения](../01-git-and-version-control/09-teamwork-pull-requests-and-conventions.md) |
| Целочисленные типы и переполнение | [Урок 02.2 — Типы данных](../02-programming-basics/02-data-types.md) |
| Тесты и отладка | [Урок 02.8 — Ваши первые тесты](../02-programming-basics/08-your-first-tests.md), [Урок 02.9 — Основы отладки](../02-programming-basics/09-debugging-basics.md) |
| Владение и перемещение | [Урок 03.2 — Правила владения](../03-ownership-and-borrowing/02-ownership-rules.md), [Урок 03.3 — Перемещение, копирование и клонирование](../03-ownership-and-borrowing/03-move-copy-and-clone.md) |
| Заимствование и `&mut` | [Урок 03.4 — Ссылки и заимствование](../03-ownership-and-borrowing/04-references-and-borrowing.md), [Урок 03.5 — Изменяемые ссылки и правила заимствования](../03-ownership-and-borrowing/05-mutable-references-and-the-borrow-rules.md) |
| `Result` и оператор `?` | [Урок 04.10 — Обработка ошибок](../04-structs-enums-collections-errors/10-error-handling.md) |
| Времена жизни | [Урок 05.3 — Времена жизни](../05-rust-intermediate/03-lifetimes.md), [Урок 06.5 — Времена жизни подробно](05-lifetimes-in-depth.md) |
| Замыкания и итераторы | [Урок 05.4 — Замыкания](../05-rust-intermediate/04-closures.md), [Урок 05.5 — Итераторы](../05-rust-intermediate/05-iterators.md) |
| `Arc`, `Mutex`, потоки, каналы | [Урок 05.10 — Потоки, каналы и общее состояние](../05-rust-intermediate/10-threads-channels-and-shared-state.md) |
| Обобщения и трейт-объекты, vtable | [Урок 06.1 — Трейт-объекты и динамическая диспетчеризация](01-trait-objects-and-dynamic-dispatch.md) |
| `Cell`, `RefCell`, `Rc<RefCell<T>>` | [Урок 06.4 — Внутренняя изменяемость](04-interior-mutability.md) |
| `unsafe`, сырые указатели, Miri | [Урок 06.6 — Небезопасный Rust и Miri](06-unsafe-rust-and-miri.md) |
| `macro_rules!` | [Урок 06.7 — Декларативные макросы](07-declarative-macros.md) |
| Процедурные макросы | [Урок 06.8 — Процедурные макросы](08-procedural-macros.md) |
| `async`/`await`, Tokio | [Урок 06.9 — Введение в async/await](09-async-await-introduction.md) |
| FFI, `CString`, `build.rs` | [Урок 06.10 — FFI: вызов кода на C](10-ffi-calling-c.md) |
| Пул потоков, основы HTTP (HyperText Transfer Protocol, протокола передачи гипертекста), корректное завершение | [Урок 06.11 — Проект: многопоточный веб-сервер](11-project-multithreaded-web-server.md) |

Если вы ошиблись больше чем в пяти вопросах, не спешите в Стадию 2. Повторите уроки из таблицы, переделайте по одному упражнению из каждого и пройдите тест снова через день. Повторять через день, потом через неделю — это **интервальное повторение**, и оно работает намного лучше, чем дважды перечитать всё за один вечер.

## Что дальше

**Стадия 2 — Основы инженерии** начинается с [Этапа 07 — Алгоритмы и структуры данных](../07-algorithms-and-data-structures/README.md). Он использует всё из Стадии 1: тесты, бенчмарки, обобщения и итераторы. Затем Стадии 3–5 уходят вглубь — к тому, как на самом деле работают компьютеры и операционные системы, и вширь — к сетям, базам данных и распределённым системам.

Перед началом Этапа 07:

1. **Напишите короткую ретроспективу** в журнале обучения (10 минут): какие три темы были самыми трудными, какими тремя проектами вы гордитесь, что одно вы сделаете иначе в Стадии 2. Добавьте дату.
2. **Запланируйте повторения:** поставьте темы, в которых вы ошиблись в тесте, в календарь на завтра, на неделю вперёд и на месяц вперёд.
3. **Посмотрите на своё портфолио.** В вашем репозитории теперь проекты с тестами, README и историей. Откройте его на GitHub или GitLab как посторонний: понятен ли `README.md` верхнего уровня? Говорит ли каждый проект, как его запускать и тестировать? Исправьте самую слабую страницу.
4. **Используйте привычки Git** этой стадии в каждом проекте с этого момента:

| Привычка | Команда |
|---|---|
| Смотрите перед коммитом | `git status`, `git diff` |
| Маленькие коммиты с понятными сообщениями | `git add <files>`, `git commit -m "feat(scope): what changed"` |
| Работайте в ветке, сливайте с сохранением истории | `git switch -c name`, `git merge --no-ff name` |
| Отмечайте вехи | `git tag -a name -m "message"`, `git push --tags` |
| Безопасно отменяйте | `git restore <file>`, `git revert <commit>` |
| Смотрите историю | `git log --oneline --graph --decorate` |
| Никогда не коммитьте результат сборки и секреты | `.gitignore` с `target/` и проверка `git status` |

5. **Знайте свой набор инструментов.** Это инструменты следующих стадий. Не устанавливайте их все сейчас; каждый урок скажет, когда ставить. Версии проверены 2026-11-17.

| Инструмент | Версия | Для чего | Где используется |
|---|---|---|---|
| `cargo fmt`, `cargo clippy`, `cargo test` | идут с Rust | форматирование, предупреждения, тесты: ворота качества | каждый проект |
| `cargo-expand` | 1.0 | показывает раскрытие макросов | Уроки 06.7, 06.8 |
| Miri (`cargo +nightly miri`) | компонент nightly | находит неопределённое поведение в коде Rust | Урок 06.6, Этап 20 |
| `cargo-nextest` | 0.9 | более быстрый запуск тестов | Этап 09 |
| `criterion` | 0.8 | статистические бенчмарки на Rust | Этап 05, Этап 09 |
| `hyperfine` | 2.0 | бенчмарки целых команд | Этап 09 |
| `cargo-llvm-cov` | 0.9 | покрытие тестами | Этап 09 |
| `cargo-audit`, `cargo-deny` | 0.23, 0.21 | проверки безопасности и лицензий зависимостей | Этап 10 |
| `oha` | 1.17 | нагрузочное тестирование HTTP | Этап 19 |
| `just` | 1.59 | маленький запускатель команд для задач проекта | по желанию |
| `bacon` | 3.26 | перезапускает `cargo check` или тесты при изменении файлов | по желанию |

Откройте [START-HERE.md](../START-HERE.md), если хотите заново прочитать, как устроен курс, и держите [глоссарий](../GLOSSARY.md) открытым, пока читаете: Стадия 2 вводит много новых слов.

## Источники и что почитать

- [The Rust Programming Language](https://doc.rust-lang.org/book/) — официальная книга; у каждой темы Стадии 1 есть глава, которую можно перечитать.
- [Rust by Example](https://doc.rust-lang.org/rust-by-example/) — короткие запускаемые примеры по тем же темам.
- [Rustlings](https://github.com/rust-lang/rustlings) — небольшие упражнения, которые вы решаете в редакторе; хороший способ потренироваться перед Стадией 2.
- [Exercism — Rust track](https://exercism.org/tracks/rust) — практические задачи с обратной связью.
- [The Cargo Book](https://doc.rust-lang.org/cargo/) — всё о Cargo, рабочих пространствах, скриптах сборки и профилях.
- [Clippy lints](https://rust-lang.github.io/rust-clippy/master/index.html) — список предупреждений с объяснениями, чтобы понимать, что говорит `clippy`.
- [Pro Git](https://git-scm.com/book/en/v2) — бесплатная книга о Git для тех частей Этапа 01, которые вы хотите углубить.

---

[🏠 Начало](../START-HERE.md) · [← 06.11 Проект: многопоточный веб-сервер](11-project-multithreaded-web-server.md) · [📋 Этап 06](README.md) · [Этап 07 — Алгоритмы и структуры данных →](../07-algorithms-and-data-structures/README.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
