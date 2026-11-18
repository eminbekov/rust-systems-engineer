<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 00.5 Установка Rust и RustRover](05-install-rust-and-rustrover.md) · [📋 Этап 00](README.md) · [00.7 Как читать ошибки и где искать помощь →](07-reading-errors-and-getting-help.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/00-getting-started/06-your-first-rust-program.md) <!-- nav -->

# Урок 00.6 — Ваша первая программа на Rust

> **Этап 00 — Начало пути** · Урок 6 из 8 · ⏱ около 3 часов (2 занятия)

## Что вы узнаете

- Как создать проект на Rust с помощью Cargo и для чего нужен каждый файл в нём.
- Как собрать и запустить программу и чем отличаются `cargo run`, `cargo build` и `cargo check`.
- Что означает каждая часть `fn main() { println!("Hello, world!"); }`.
- Как изменить программу, отформатировать её командой `cargo fmt` и проверить командой `cargo clippy`.
- Как скомпилировать один файл напрямую через `rustc` и как запустить проект в RustRover.
- Почему Cargo уже создал для вас репозиторий Git.

## Перед началом

- Вы прошли [Урок 00.5 — Установка Rust и RustRover](05-install-rust-and-rustrover.md). Команды `rustc --version` и `cargo --version` работают в **новой** вкладке терминала.
- Папка `~/projects/rust/code` существует (вы создали её в [Уроке 00.4](04-terminal-basics.md)). Если её нет, выполните `mkdir -p ~/projects/rust/code`.
- Откройте терминал и перейдите в папку курса: `cd ~/projects/rust`.

## Новые слова

| Слово | Значение |
|---|---|
| **проект** (project, package) | Папка с файлом `Cargo.toml` и исходным кодом, которую Cargo умеет собирать. Cargo называет её *пакетом* (package). |
| **крейт** (crate) | Единица, которую собирает компилятор: либо программа (*бинарный крейт*), либо библиотека. Подробно объясняется на Этапе 04. |
| **`Cargo.toml`** | Файл настроек проекта: его имя, версия, редакция и библиотеки. |
| **TOML** (Tom's Obvious, Minimal Language, «очевидный минимальный язык Тома») | Простой текстовый формат для настроек: строки `name = "value"`, сгруппированные под `[заголовками]`. |
| **`Cargo.lock`** | Файл, который Cargo пишет сам. В нём записаны точные версии всего, что использовалось при последней сборке. |
| **функция** (function) | Именованный блок кода. Функция с именем `main` — то место, откуда начинается каждая программа на Rust. |
| **макрос** (macro) | Особый вид помощника, который разворачивается в больший код. Его имя оканчивается на `!`, как у `println!`. Как они работают, учит Этап 05. |
| **строка** (string) | Фрагмент текста в кавычках: `"Hello"`. |
| **точка с запятой** (semicolon) `;` | Знак, который заканчивает оператор (один шаг) в Rust. |
| **комментарий** (comment) | Текст для людей, который компилятор игнорирует. В Rust начинается с `//`. |
| **сборка** (build) | Скомпилировать код в исполняемый файл. |
| **бинарный файл** (binary) | Исполняемый файл. Cargo также использует это слово для крейта-программы. |
| **профиль** (profile) | Набор настроек компилятора. **debug** (по умолчанию) собирается быстро и удобен для отладки; **release** собирается медленно, но делает программу быстрее. |
| **`target`** | Папка, куда Cargo складывает всё, что собирает. |
| **репозиторий Git** (Git repository) | Папка, за которой следит Git. Содержит скрытую папку `.git` с историей. Его изучает Этап 01. |
| **`.gitignore`** | Файл со списком того, за чем Git не должен следить, например результатов сборки. |
| **IDE** (Integrated Development Environment, интегрированная среда разработки) | Программа для написания кода: редактор с кнопками запуска, отметками ошибок и другими помощниками. Мы используем **RustRover**. |

## 1. Создайте проект

В терминале, из `~/projects/rust`, выполните:

```text
% cargo new code/p00-hello
    Creating binary (application) `p00-hello` package
note: see more `Cargo.toml` keys and their definitions at https://doc.rust-lang.org/cargo/reference/manifest.html
```

Что происходит:

1. `cargo new` создаёт новый проект (**пакет**).
2. `code/p00-hello` — путь новой папки. *Последняя* часть пути, `p00-hello`, становится именем проекта.
3. `binary (application)` означает: программа, которую можно запускать (а не библиотека для других программ).
4. Строка `note:` — просто ссылка на документацию.

> 📏 **Соглашение:** имена проектов в этом курсе выглядят как `p00-hello`: `p`, номер этапа, дефис и
> короткое имя строчными буквами с дефисами между словами. Сам Cargo требует лишь, чтобы имя начиналось с
> буквы и состояло из букв, цифр, `-` или `_`.

Зайдите в проект и посмотрите на него. Опция `-a` показывает и скрытые файлы:

```text
% cd code/p00-hello
% ls -a
.
..
.git
.gitignore
Cargo.toml
src
```

Дерево папок:

```text
p00-hello/
├── .git/            ← скрытая: история Git (пока пустая)
├── .gitignore       ← список файлов, которые Git должен игнорировать
├── Cargo.toml       ← настройки проекта
└── src/
    └── main.rs      ← ВАШ исходный код
```

## 2. Файлы, которые создал Cargo

### `Cargo.toml`

```text
% cat Cargo.toml
[package]
name = "p00-hello"
version = "0.1.0"
edition = "2024"

[dependencies]
```

Что происходит:

1. `[package]` — заголовок: следующие строки описывают пакет.
2. `name = "p00-hello"` — имя проекта (и программы, которую он производит).
3. `version = "0.1.0"` — версия *вашей* программы в виде старшая.младшая.патч. `0.1.0` значит «только начали».
4. `edition = "2024"` — редакция языка из Урока 00.5.
5. `[dependencies]` — здесь вы будете перечислять библиотеки (называемые *крейтами*), которые использует ваша программа. Пока пусто: мы используем только то, что поставляется с Rust.

### `src/main.rs`

```text
% cat src/main.rs
fn main() {
    println!("Hello, world!");
}
```

Это полная программа. `src` означает «source» (исходники). Файлы исходного кода Rust оканчиваются на `.rs`. `main.rs` — это
начальный файл программы.

## 3. Запустите её

```text
% cargo run
   Compiling p00-hello v0.1.0 (/Users/student/projects/rust/code/p00-hello)
    Finished `dev` profile [unoptimized + debuginfo] target(s) in 0.16s
     Running `target/debug/p00-hello`
Hello, world!
```

Что происходит:

1. `cargo run` сначала собирает проект, затем запускает программу. Это два шага из Урока 00.3: компиляция, потом запуск.
2. `Compiling ...` — Cargo вызвал `rustc`, чтобы превратить `src/main.rs` в машинный код. Время (`0.16s`) на вашем компьютере будет другим.
3. `Finished `dev` profile [unoptimized + debuginfo]` — сборка готова. «dev» — профиль *debug*: без оптимизации, с дополнительной информацией для инструментов отладки.
4. `Running `target/debug/p00-hello`` — Cargo запускает исполняемый файл, который создал. Показан его путь.
5. `Hello, world!` — вывод **вашей** программы.

Запустите ещё раз:

```text
% cargo run
    Finished `dev` profile [unoptimized + debuginfo] target(s) in 0.00s
     Running `target/debug/p00-hello`
Hello, world!
```

Код не изменился, поэтому Cargo не стал компилировать снова. Он только запустил программу.

Теперь посмотрите, что появилось:

```text
% ls
Cargo.lock
Cargo.toml
src
target
```

- `target/` — новая папка. Здесь все результаты сборки. Её можно удалить в любой момент (`cargo clean`); следующая сборка создаст её заново.
- `Cargo.lock` — новый файл. Его пишет Cargo; никогда не правьте его вручную.

Сама программа — это `target/debug/p00-hello`. Это обычный исполняемый файл (Урок 00.3), поэтому его можно
запустить напрямую, без Cargo:

```text
% ./target/debug/p00-hello
Hello, world!
```

## 4. Что означает каждая часть программы

```rust
fn main() {
    println!("Hello, world!");
}
```

Что происходит, часть за частью:

1. `fn` — ключевое слово (зарезервированное слово языка), которое говорит: «здесь начинается функция». `fn` — сокращение от *function* (функция).
2. `main` — имя функции. Rust начинает каждую программу с выполнения функции с именем `main`.
3. `()` — список входных данных функции. Пустые скобки означают, что `main` ничего не принимает.
4. `{` и `}` — фигурные скобки охватывают тело функции: шаги, которые она выполняет.
5. `println!` — **макрос**, который печатает текст и затем завершает строку (`ln` = line, строка). `!` говорит, что это макрос, а не обычная функция.
6. `("Hello, world!")` — аргумент: **строка**. Текст в Rust пишется в **двойных** кавычках.
7. `;` — точка с запятой заканчивает шаг. Почти каждая строка в функции ею заканчивается.

Два правила стиля, которые стоит принять с первого дня:

> 📏 **Соглашение:** делайте отступ в **4 пробела** (а не клавишей Tab). Считать их не нужно: RustRover и
> `cargo fmt` делают это за вас.

> 📏 **Соглашение:** имена функций и переменных пишутся в `snake_case`: строчные слова,
> разделённые подчёркиваниями (`print_report`). Больше правил именования вы увидите на Этапе 02.

## 5. Измените программу

Откройте `src/main.rs` в RustRover (или любом редакторе) и замените содержимое этим:

```rust
fn main() {
    // This program prints several lines.
    println!("Hello, Rust!");
    println!("I am learning to program.");
    print!("This line has no newline, ");
    println!("so this text continues it.");
    println!("Two plus three is {}.", 2 + 3);
    let name = "Ada";
    println!("My name is {}.", name);
}
```

Сохраните файл (`Cmd + S`) и выполните `cargo run`:

```text
% cargo run
   Compiling p00-hello v0.1.0 (/Users/student/projects/rust/code/p00-hello)
    Finished `dev` profile [unoptimized + debuginfo] target(s) in 0.13s
     Running `target/debug/p00-hello`
Hello, Rust!
I am learning to program.
This line has no newline, so this text continues it.
Two plus three is 5.
My name is Ada.
```

Что происходит:

1. `// This program prints several lines.` — **комментарий**. Компилятор игнорирует всё от `//` до конца строки. Используйте комментарии, чтобы объяснять, *почему*, а не повторять, *что* делает код.
2. Следующие две строки `println!` печатают текст и переходят на новую строку.
3. `print!` (без `ln`) печатает текст и **не** переходит на новую строку. Поэтому следующий текст продолжается в той же строке.
4. В `println!("Two plus three is {}.", 2 + 3)` символы `{}` — это дырка. Rust подставляет в неё значение после запятой. Значение сначала вычисляется: `2 + 3` — это `5`.
5. `let name = "Ada";` — создаёт именованную коробку `name`, в которой лежит текст `"Ada"`. Это **переменная**, из Урока 00.2; Урок 02.1 учит ей как следует. `{}` в последней строке заполняется её значением.
6. В выводе нет ни одного `{}`: дырки заменены значениями.

Печать и форматирование во всех подробностях: Урок 02.6.

## 6. Команды Cargo, которые нужны каждый день

| Команда | Что делает | Когда использовать |
|---|---|---|
| `cargo run` | собрать, затем запустить | чаще всего |
| `cargo build` | только собрать (профиль debug) | чтобы увидеть, компилируется ли код, и получить файл |
| `cargo check` | проверить код **без** создания исполняемого файла | самая быстрая обратная связь, пока вы пишете |
| `cargo build --release` | собрать с оптимизациями (профиль release) | чтобы измерить скорость или отдать программу кому-то |
| `cargo fmt` | отформатировать код | перед каждым коммитом |
| `cargo clippy` | советы об ошибках и стиле | перед каждым коммитом |
| `cargo clean` | удалить папку `target` | чтобы освободить место или начать с чистого листа |

Посмотрите на `check` и `--release`:

```text
% cargo check
    Checking p00-hello v0.1.0 (/Users/student/projects/rust/code/p00-hello)
    Finished `dev` profile [unoptimized + debuginfo] target(s) in 0.04s
% cargo build --release
   Compiling p00-hello v0.1.0 (/Users/student/projects/rust/code/p00-hello)
    Finished `release` profile [optimized] target(s) in 0.08s
% ls -lh target/debug/p00-hello target/release/p00-hello
-rwxr-xr-x  1 student  wheel   465K Nov 17 16:25 target/debug/p00-hello
-rwxr-xr-x  1 student  wheel   426K Nov 17 16:25 target/release/p00-hello
```

Что происходит:

1. `cargo check` пишет «Checking» вместо «Compiling»: он проанализировал код и сообщил об ошибках, но не создал исполняемый файл. В больших проектах он намного быстрее.
2. `cargo build --release` создал второй исполняемый файл в `target/release/`, с оптимизациями. Для программы из одной строки разница крошечная; для настоящих программ релизная сборка может быть быстрее во много раз.
3. У двух файлов разный размер, потому что debug-сборка содержит дополнительную информацию. (Ваши размеры будут другими.)

> ⚡ **Производительность:** всегда измеряйте скорость с `--release`. Debug-сборка предназначена для разработки; в тяжёлых вычислениях она может
> быть в 10–100 раз медленнее release-сборки. Никогда не судите о скорости
> Rust по debug-сборке.

### Форматирование и линтер

```text
% cargo fmt --check
```

Если команда ничего не печатает, ваш код уже отформатирован. Вот что бывает, когда нет. Представьте
неаккуратный файл:

```rust
fn main(){
println!("Hello, Rust!"   );
        println!("Second line");
}
```

`cargo fmt --check` показывает различия и завершается ошибкой (код выхода 1):

```text
Diff in /Users/student/projects/rust/code/p00-hello/src/main.rs:1:
-fn main(){
-println!("Hello, Rust!"   );
-        println!("Second line");
+fn main() {
+    println!("Hello, Rust!");
+    println!("Second line");
 }
```

Строки, начинающиеся с `-`, были бы удалены, строки с `+` — добавлены. Выполните `cargo fmt` (без
`--check`), и он перепишет файл за вас:

```text
% cargo fmt
% cat src/main.rs
fn main() {
    println!("Hello, Rust!");
    println!("Second line");
}
```

`cargo clippy` запускает линтер. Для правильного, чистого кода он печатает только строки `Checking` и `Finished`. Clippy
подаёт голос, когда ваш код работает, но мог бы быть лучше. Например, пусть последняя строка нашей программы была бы
`println!("My name is {} and I like {}.", "Ada", "Rust")`. Она работает, но тексты уже известны, поэтому их можно
написать прямо в первой строке. Clippy говорит:

```text
warning: literal with an empty format string
 --> src/main.rs:8:46
  |
8 |     println!("My name is {} and I like {}.", "Ada", "Rust");
  |                                              ^^^^^^^^^^^^^
  = note: `#[warn(clippy::print_literal)]` on by default
help: try
8 -     println!("My name is {} and I like {}.", "Ada", "Rust");
8 +     println!("My name is Ada and I like Rust.");
```

> 💡 **Совет:** сделайте `cargo fmt` и `cargo clippy` привычкой. Каждый проект в этом курсе должен проходить обе проверки.
> RustRover делает это за вас, если вы включили «run rustfmt on save» и Clippy в Уроке 00.5.

## 7. Компиляция одного файла через `rustc`

Cargo вызывает `rustc` за вас. Вы можете вызвать его сами для одного файла. Сделайте временную папку:

```text
% mkdir -p /tmp/rust-try && cd /tmp/rust-try
% cat > hello.rs <<'XX'
fn main() {
    println!("Hello, Rust");
}
XX
% rustc hello.rs
% ./hello
Hello, Rust
```

Что происходит:

1. `rustc hello.rs` скомпилировал файл и записал исполняемый файл `hello` рядом с ним. Ни папки `target`, ни `Cargo.toml` нет.
2. `./hello` запустил его (Урок 00.4: `./` значит «в этой папке»).

Для всего, что больше одного файла, всегда используйте Cargo: он управляет настройками, папками и (позже) библиотеками.
По умолчанию `rustc` использует более старую редакцию, чем Cargo; ещё одна причина пользоваться Cargo.

Удалите временную папку: `cd ~ && rm -r /tmp/rust-try`.

## 8. Запуск в RustRover

1. В RustRover выберите **File → Open**, выберите папку `~/projects/rust/code/p00-hello` (ту, где лежит `Cargo.toml`) и нажмите **Open**. Выберите **This Window** или **New Window**.
2. Если RustRover спросит, доверять ли проекту, выберите **Trust Project**. Если он предложит «Attach Cargo project» или покажет уведомление о Cargo, согласитесь: так IDE узнаёт о `Cargo.toml`.
3. Откройте `src/main.rs`. Рядом с `fn main` на полях вы увидите зелёный треугольник. Щёлкните его и выберите **Run 'Run p00-hello'** (или нажмите `Ctrl + R`).
4. Внизу откроется панель **Run** и покажет те же строки, что показал терминал.
5. Попробуйте `Cmd + Option + L` (**Code → Reformat Code**), чтобы отформатировать файл, и посмотрите на красные и жёлтые отметки в редакторе: это сообщения компилятора и Clippy, показанные на месте.

> 💡 **Совет:** IDE и терминал делают ровно одну и ту же работу: оба вызывают Cargo. Знание команд делает вас
> независимым от любой IDE, а на серверах IDE нет вообще.

## 9. Первый взгляд на Git

Помните `.git` и `.gitignore` из раздела 1? Cargo создал в проекте **репозиторий Git**, потому что
контроль версий — стандартная практика. Посмотрите на файл `.gitignore`:

```text
% cat .gitignore
/target
```

Он говорит Git игнорировать папку `target`, потому что результаты сборки большие, их можно пересоздать в любой момент, и им
не место в истории. Спросим у Git, что он видит (это лишь взгляд; всё объяснит Этап 01):

```text
% git status --short
?? .gitignore
?? Cargo.lock
?? Cargo.toml
?? src/
```

Каждое `??` означает «новый файл, за которым Git ещё не следит». `target/` нет в списке из-за `.gitignore`.
В историю пока ничего не сохранено; первый **коммит** — тема [Урока 01.3](../01-git-and-version-control/03-your-first-repository-and-commits.md).

> ⚠️ **Частая ошибка:** если проект создан *внутри* папки, которая уже является репозиторием Git (например,
> когда вы превратите `~/projects/rust` в свой учебный репозиторий на Этапе 01), `cargo new` **не** создаёт ещё один
> `.git` и `.gitignore`. Тогда нужно убедиться, что `target` игнорируется главным репозиторием.
> Этап 01 покажет, как.

## Частые ошибки

1. **Забыли `!` в `println!`.**

```rust
fn main() {
    println("Hello, world!");
}
```

```text
error[E0423]: cannot find function `println` in this scope
 --> src/main.rs:2:5
  |
2 |     println("Hello, world!");
  |     ^^^^^^^ not found in this scope
  |
  = note: a macro named `println` exists in another namespace
help: use `!` to invoke the macro
  |
2 |     println!("Hello, world!");
  |            +
```

   *Значение:* нет *функции* с именем `println`, но есть *макрос*. Компилятор предлагает исправление, поставив `+` под пропущенным `!`.

2. **Забыли точку с запятой.**

```rust
fn main() {
    println!("first")
    println!("second");
}
```

```text
error: expected `;`, found `println`
 --> src/main.rs:2:22
  |
2 |     println!("first")
  |                      ^ help: add `;` here
3 |     println!("second");
  |     ------- unexpected token
```

   *Значение:* строка 2 закончилась без `;`. Значок `^` указывает точно на место, где его не хватает.

3. **Незакрытая строка.** Здесь не хватает закрывающей кавычки:

```rust
fn main() {
    println!("Hello, world!);
}
```

```text
error[E0765]: unterminated double quote string
 --> src/main.rs:2:14
  |
2 |       println!("Hello, world!);
  |  ______________^
3 | | }
  | |__^
```

   *Значение:* строка, которая начинается с `"`, должна заканчиваться `"`. Компилятор отмечает всё от открывающей кавычки до конца файла.

4. **Одинарные кавычки для текста.** `println!('Hello, world!');` даёт `error[E0762]: unterminated character literal`.
   В Rust одинарные кавычки предназначены для одного символа; текст пишется в двойных.

5. **Нет функции `main`.** Если у функции другое имя:

```text
error[E0601]: `main` function not found in crate `p00_hello`
 --> src/main.rs:3:2
  |
3 | }
  |  ^ consider adding a `main` function to `src/main.rs`
```

   *Значение:* у программы нет точки входа. Заметьте, что в имени крейта `p00_hello` дефис превратился в подчёркивание.

6. **Незакрытая фигурная скобка.** Если пропущена последняя `}`: `error: this file contains an unclosed delimiter`. *Разделитель* (delimiter) — это скобка, фигурная скобка или кавычка. Сообщение указывает на `{`, у которой нет пары.

7. **Запуск Cargo вне проекта.**

```text
% cargo run
error: could not find `Cargo.toml` in `/Users/student/projects/rust/code/empty` or any parent directory
```

   *Значение:* вы находитесь в папке без проекта. Выполните `pwd`, затем `cd` в проект (папку, где лежит `Cargo.toml`).

8. **Создание проекта в уже существующей папке.** Двукратный `cargo new code/p00-hello` даёт `error: destination ... already exists` и подсказку использовать `cargo init`. Выберите новое имя или удалите старую папку.

9. **Файл не сохранён.** Вы правите в RustRover, но видите старый вывод. Сохраните (`Cmd + S`) перед `cargo run` в терминале. (`cargo run` внутри RustRover сохраняет автоматически.)

Все эти сообщения подробно разобраны в [Уроке 00.7](07-reading-errors-and-getting-help.md).

## Попробуйте сами

Сделайте всё с нуля, не подглядывая в урок, как проверку памяти. Используйте новое имя проекта `p00-trial`.

1. `cd ~/projects/rust`
2. `cargo new code/p00-trial`
3. `cd code/p00-trial`
4. Замените `src/main.rs` программой, которая печатает ваше имя в одной строке и строку `I am learning Rust` в следующей.
5. `cargo run`. Проверьте вывод.
6. Сделайте намеренную ошибку (удалите одну `;`), снова выполните `cargo run` и прочитайте ошибку. Затем исправьте.
7. `cargo fmt` и `cargo clippy`. Обе не должны сообщать о проблемах.
8. Откройте проект в RustRover и запустите его зелёным треугольником.

## Упражнения

### Упражнение 1 — Три строки (легко)

В `p00-hello` напишите программу, которая печатает ровно эти три строки. Используйте свои данные.

```text
Name: Ada
City: London
Favourite language: Rust
```

<details>
<summary>Подсказка</summary>

Три строки `println!`. Каждая строка текста — в двойных кавычках, и каждый `println!` заканчивается `;`.

</details>

<details>
<summary>Ответ</summary>

```rust
fn main() {
    println!("Name: Ada");
    println!("City: London");
    println!("Favourite language: Rust");
}
```

Вывод:

```text
Name: Ada
City: London
Favourite language: Rust
```

Почему это работает: каждый `println!` печатает одну строку и завершает её переводом строки.

</details>

### Упражнение 2 — Баннер (легко)

Создайте **новый** проект `code/p00-banner` командой `cargo new`. Сделайте так, чтобы он печатал эту рамку:

```text
+----------+
| Rust 1.0 |
+----------+
```

<details>
<summary>Подсказка</summary>

Выполните `cargo new code/p00-banner` из `~/projects/rust`, правьте `code/p00-banner/src/main.rs` и запускайте `cargo run` внутри новой папки проекта. В рамке три строки, значит, нужны три `println!`.

</details>

<details>
<summary>Ответ</summary>

```rust
fn main() {
    println!("+----------+");
    println!("| Rust 1.0 |");
    println!("+----------+");
}
```

Вывод:

```text
+----------+
| Rust 1.0 |
+----------+
```

Почему это работает: текст внутри кавычек печатается ровно как есть, включая символы `+`, `-` и `|`. Когда программа готова, проверьте `cargo fmt --check` и `cargo clippy`: обе проходят без сообщений.

</details>

### Упражнение 3 — Исправьте три программы (средне)

Ни одна из программ ниже не компилируется. Скажите, что не так, не запуская их, затем исправьте и проверьте через
`cargo run` в черновом проекте.

```rust
// Program A
fn main() {
    println("Hello, world!");
}
```

```rust
// Program B
fn main() {
    println!("first")
    println!("second");
}
```

```rust
// Program C
fn start() {
    println!("Hello, world!");
}
```

<details>
<summary>Подсказка</summary>

A: посмотрите на конец имени `println`. B: посмотрите на конец строки 2. C: какое имя Rust ищет как точку входа?

</details>

<details>
<summary>Ответ</summary>

- **A** — `println` нужно писать `println!` (это макрос).
- **B** — первому `println!` нужна точка с запятой в конце.
- **C** — функция должна называться `main`, а не `start`.

Исправленные версии:

```rust
fn main() {
    println!("Hello, world!");
}
```

```rust
fn main() {
    println!("first");
    println!("second");
}
```

```rust
fn main() {
    println!("Hello, world!");
}
```

Выводы: A и C печатают `Hello, world!`; B печатает `first` и `second` в двух строках.

Почему это работает: все три ошибки связаны с точным написанием, которого требует компилятор: `!` у макроса, `;` в конце шага и имя `main`, обозначающее начало.

</details>

### Упражнение 4 — Debug и release (средне)

В `p00-hello` соберите оба профиля и сравните размеры двух исполняемых файлов командой `ls -lh`. Какой
больше и почему? Где лежит каждый файл?

<details>
<summary>Подсказка</summary>

`cargo build` и `cargo build --release`; файлы находятся в двух подпапках `target`.

</details>

<details>
<summary>Ответ</summary>

```text
% cargo build
% cargo build --release
% ls -lh target/debug/p00-hello target/release/p00-hello
-rwxr-xr-x  1 student  wheel   465K Nov 17 16:25 target/debug/p00-hello
-rwxr-xr-x  1 student  wheel   426K Nov 17 16:25 target/release/p00-hello
```

Файл debug (`target/debug/`) больше, потому что содержит дополнительную информацию для инструментов отладки и неоптимизированный код. Файл release (`target/release/`) меньше и для настоящих программ гораздо быстрее. Ваши числа немного отличаются.

Почему это работает: два профиля передают `rustc` разные опции, поэтому получаются разные файлы, а Cargo хранит их в разных папках, чтобы они не затирали друг друга.

</details>

### Упражнение 5 — Таблица умножения, аккуратная и чистая (сложно)

В новом проекте `code/p00-table` напечатайте строки таблицы умножения на 7 от `7 x 2` до `7 x 6`.
Пусть арифметику делает Rust (не набирайте результаты сами). Код должен проходить `cargo fmt --check` и
`cargo clippy` без сообщений.

Ожидаемый вывод:

```text
7 x 2 = 14
7 x 3 = 21
7 x 4 = 28
7 x 5 = 35
7 x 6 = 42
```

<details>
<summary>Подсказка</summary>

Используйте `{}` в тексте и `7 * 2` после запятой. Почему с 2, а не с 1? Попробуйте `7 * 1` и прочитайте, что скажет Clippy.

</details>

<details>
<summary>Ответ</summary>

```rust
fn main() {
    println!("7 x 2 = {}", 7 * 2);
    println!("7 x 3 = {}", 7 * 3);
    println!("7 x 4 = {}", 7 * 4);
    println!("7 x 5 = {}", 7 * 5);
    println!("7 x 6 = {}", 7 * 6);
}
```

Почему это работает: умножение вычисляется первым и затем подставляется в `{}`. Если написать `7 * 1`, Clippy ответит «this operation has no effect», потому что умножение на единицу ничего не меняет. (На Этапе 02 вы сделаете это циклом, чтобы не повторять строку.)

</details>

## Вопросы для самопроверки

1. Что создаёт `cargo new code/p00-hello` и как называется проект?

<details>
<summary>Ответ</summary>

Папку проекта `code/p00-hello` с `Cargo.toml`, `src/main.rs` и репозиторием Git (`.git`, `.gitignore`). Имя — последняя часть пути: `p00-hello`.

</details>

2. В чём разница между `cargo run`, `cargo build` и `cargo check`?

<details>
<summary>Ответ</summary>

`cargo build` компилирует программу. `cargo run` собирает её и затем запускает. `cargo check` только анализирует код на ошибки и не создаёт исполняемый файл, поэтому он самый быстрый.

</details>

3. Что означает `!` в `println!`?

<details>
<summary>Ответ</summary>

Что `println` — макрос, а не обычная функция.

</details>

4. Куда Cargo кладёт исполняемый файл debug-сборки и что такое `target`?

<details>
<summary>Ответ</summary>

В `target/debug/<имя проекта>`. `target` — папка для всех результатов сборки; её можно удалить и пересоздать, и Git её игнорирует.

</details>

5. Почему скорость мы измеряем только с `--release`?

<details>
<summary>Ответ</summary>

Профиль debug не оптимизирует код, поэтому он может быть во много раз медленнее профиля release.

</details>

6. Что делают `cargo fmt` и `cargo clippy`?

<details>
<summary>Ответ</summary>

`cargo fmt` переписывает код в стандартном стиле. `cargo clippy` показывает предупреждения с советами об ошибках и стиле.

</details>

7. Почему `target/` указан в `.gitignore`?

<details>
<summary>Ответ</summary>

Результаты сборки большие, их можно пересоздать в любой момент, и им не место в истории исходного кода.

</details>

## Итоги

- `cargo new code/pNN-name` создаёт проект; `Cargo.toml` хранит его настройки, а `src/main.rs` — начальный файл.
- Каждая программа на Rust начинается с `fn main()`. `println!` печатает строку; `!` отмечает макрос; `{}` заполняется значением.
- `cargo run` = сборка + запуск. `cargo check` — быстрая проверка. `cargo build --release` делает оптимизированную версию.
- В `target/` весь результат сборки, Git его игнорирует; `Cargo.lock` пишет Cargo.
- Выполняйте `cargo fmt` и `cargo clippy` перед каждым коммитом; обе должны быть чистыми.
- `rustc file.rs` компилирует один файл, но обычный способ — Cargo.
- RustRover запускает те же команды Cargo зелёным треугольником.
- У ошибок компилятора есть код, место (`file:line:column`) и часто подсказка с исправлением. Читать их научит Урок 00.7.

## Источники и что почитать

- [The Rust Programming Language — 1.2 Hello, World!](https://doc.rust-lang.org/book/ch01-02-hello-world.html) — официальный разбор первой программы с `rustc`.
- [The Rust Programming Language — 1.3 Hello, Cargo!](https://doc.rust-lang.org/book/ch01-03-hello-cargo.html) — официальное объяснение Cargo, `Cargo.toml` и `cargo build`.
- [The Cargo Book — Creating a New Package](https://doc.rust-lang.org/cargo/guide/creating-a-new-project.html) — что подробно делает `cargo new`.
- [The Cargo Book — Package Layout](https://doc.rust-lang.org/cargo/guide/project-layout.html) — где лежат файлы в проекте Cargo.
- [Standard library — `println!`](https://doc.rust-lang.org/std/macro.println.html) — справочник по макросу, включая все параметры форматирования (Урок 02.6 их учит).
- [Clippy documentation](https://doc.rust-lang.org/clippy/) — как работает Clippy и список его проверок.

---

[🏠 Начало](../START-HERE.md) · [← 00.5 Установка Rust и RustRover](05-install-rust-and-rustrover.md) · [📋 Этап 00](README.md) · [00.7 Как читать ошибки и где искать помощь →](07-reading-errors-and-getting-help.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
