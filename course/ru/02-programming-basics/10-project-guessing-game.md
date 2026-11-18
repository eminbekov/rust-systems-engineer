<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 02.9 Основы отладки](09-debugging-basics.md) · [📋 Этап 02](README.md) · [02.11 Контрольная точка →](11-checkpoint.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/02-programming-basics/10-project-guessing-game.md) <!-- nav -->

# Урок 02.10 — Проект: игра «Угадай число»

> **Этап 02 — Основы программирования на Rust** · Урок 10 из 11 · ⏱ около 2,5 часа (2 занятия)

## Что вы узнаете

- Как спланировать небольшую программу и построить её шаг за шагом, по вехам.
- Как использовать внешний крейт (`rand`) с помощью Cargo.
- Как использовать `Ordering` и `match` для сравнения чисел.
- Как сохранять программу тестируемой, отделяя логику от ввода и вывода.
- Как работать с Git как профессионал: ветка для функции, коммит на каждую веху, слияние и тег в конце.

## Перед началом

- Вы прошли [Урок 02.9 — Основы отладки](09-debugging-basics.md) и все уроки до него: переменные, типы, операторы, функции, ветвления и циклы, форматирование, ввод, тесты.
- Git работает, и у вас есть учебный репозиторий в `~/projects/rust` ([Урок 01.10](../01-git-and-version-control/10-project-your-learning-repo-and-checkpoint.md)). Вы знаете ветки и слияние ([Урок 01.6](../01-git-and-version-control/06-branches-and-merging.md)).
- Для первой сборки нужно подключение к интернету (Cargo скачивает крейт `rand`). Дальше всё работает офлайн.

## Новые слова

| Слово | Значение |
|---|---|
| **крейт** (crate) | Единица кода Rust, которая компилируется вместе: программа или библиотека. Чужие библиотеки тоже называют крейтами. |
| **зависимость** (dependency) | Крейт, который использует ваш проект. |
| **crates.io** | Публичный сайт и реестр, где публикуются крейты Rust. |
| **`Cargo.toml`** | Файл, описывающий ваш проект: имя, версию, редакцию (edition) и зависимости. |
| **`Cargo.lock`** | Файл, в котором Cargo записывает **точные** версии всех используемых крейтов, чтобы каждая сборка была одинаковой. |
| **семантическое версионирование** (semantic versioning, SemVer) | Номера версий `MAJOR.MINOR.PATCH`. Изменение `MAJOR` может сломать ваш код; `MINOR` и `PATCH` не должны. |
| **случайное число** (random number) | Число, которое нельзя предсказать. Компьютеры делают *псевдослучайные* числа с помощью алгоритма. |
| **enum** (enumeration, перечисление) | Тип с фиксированным списком возможных значений, называемых вариантами. `Ordering` — это enum. Перечисления изучает Этап 04, урок 3. |
| **`Ordering`** | Стандартный enum с тремя вариантами: `Less`, `Equal`, `Greater`. Это результат сравнения двух значений. |
| **переменная окружения** (environment variable) | Именованное значение, которое операционная система передаёт программе. Её можно задать в терминале перед командой. |
| **фича-ветка** (feature branch) | Ветка Git, созданная для одной части работы и влитая обратно в `main`, когда она готова. |
| **тег** (tag) | Имя, прикреплённое к коммиту в Git, обычно для релиза. |
| **README** | Текстовый файл, объясняющий, что это за проект и как им пользоваться. |
| **приёмочный тест** (acceptance test) | Проверка того, что готовая программа ведёт себя так, как сказано в контракте. Здесь: команда и ожидаемый вывод. |
| **веха** (milestone) | Шаг плана с ясной целью и ясным способом проверить, что он выполнен. |
| **контракт** (contract) | Точное описание того, что должна делать программа: ввод, вывод и правила. |

## 1. Что вы создадите

Игра «Угадай число» для терминала. Компьютер загадывает секретное число от 1 до 100. У игрока 7 попыток. После каждой догадки игра говорит *Too small!* или *Too big!*, пока игрок не выиграет, не проиграет или не сдастся.

```text
Guess the number between 1 and 100! You have 7 attempts. Type 'quit' to give up.
Attempt 1 of 7 - type your guess:
50
Too big!
Attempt 2 of 7 - type your guess:
25
Too small!
...
```

Вы используете всё из этого этапа: переменные и константы, `u32`, `if` и `match`, цикл, функции, `Result`, чтение и разбор ввода, форматирование, тесты и привычки отладки. Вы также познакомитесь с тремя новыми вещами: внешним крейтом, `Ordering` и переменной окружения.

**Полного решения в этом уроке нет.** Вы пишете программу сами, имея контракт, план, подсказки и проверки. Если застряли, прочитайте подсказку к вехе, затем урок, который учит нужной идее.

## 2. Новые инструменты, которые вам нужны

### 2.1 Использование крейта: `rand`

В стандартной библиотеке Rust нет случайных чисел. Они берутся из крейта сообщества **`rand`**. Проект перечисляет нужные ему крейты в `Cargo.toml` в разделе `[dependencies]`. Простой способ добавить крейт — команда:

```bash
cargo add rand@0.10
```

Она сама правит `Cargo.toml`:

```toml
[dependencies]
rand = "0.10"
```

Что происходит:

1. `rand = "0.10"` значит «любая версия, совместимая с 0.10». По правилам **семантического версионирования** Cargo может использовать `0.10.4` или `0.10.5`, но не `0.11`. (Для версий ниже 1.0 средний номер считается как мажорный.)
2. При следующей `cargo build` Cargo скачивает `rand` и его собственные зависимости с **crates.io**, компилирует их и записывает точные версии в **`Cargo.lock`**. Коммитьте `Cargo.lock` для программ: это делает сборку воспроизводимой.
3. В Rust 1.100, если вы добавили крейт и пока не используете его, Cargo предупреждает: `unused dependency 'rand'`. Предупреждение исчезает, когда вы используете крейт.

Скачанное Cargo кэшируется в `~/.cargo`, поэтому последующие сборки работают офлайн. Версии и зависимости подробно объясняет Этап 05, урок 8.

Чтобы получить случайное число в диапазоне от 1 до 100 (включительно):

```rust
use rand::RngExt;

fn main() {
    let n = rand::rng().random_range(1..=100);
    println!("{n}");
}
```

- `rand::rng()` даёт генератор случайных чисел текущего потока.
- `.random_range(1..=100)` берёт число из диапазона, включая оба конца (Урок 02.5).
- `use rand::RngExt;` импортирует **трейт**, который даёт `random_range`. Без него вы получите такую ошибку:

```text
error[E0599]: no method named `random_range` found for struct `ThreadRng` in the current scope
 --> src/main.rs:2:25
  |
2 |     let n = rand::rng().random_range(1..=100);
  |                         ^^^^^^^^^^^^
  |
  = help: items from traits can only be used if the trait is in scope
```

(То же правило, что и для `Write` в Уроке 02.7: чтобы вызвать метод, принадлежащий трейту, импортируйте трейт.) Обратите внимание на строку `items from traits can only be used if the trait is in scope`, которая называет исправление.

> 🔒 **Безопасность:** `rand::rng()` хорош для игр и симуляций, но его числа не предназначены для паролей или ключей. Криптографически стойкую случайность объясняет Этап 10.

### 2.2 Сравнение с помощью `Ordering`

Два числа можно сравнить через `<`, `>` и `==`. Стандартная библиотека также даёт метод `cmp`, который возвращает результат в виде значения типа `Ordering`. `Ordering` — это **enum**: тип, который может быть только одним значением из фиксированного списка:

```text
Ordering::Less      the first value is smaller
Ordering::Equal     they are equal
Ordering::Greater   the first value is bigger
```

```rust
use std::cmp::Ordering;

fn main() {
    let secret = 42;
    for guess in [10, 42, 99] {
        match guess.cmp(&secret) {
            Ordering::Less => println!("{guess}: Too small!"),
            Ordering::Greater => println!("{guess}: Too big!"),
            Ordering::Equal => println!("{guess}: You win!"),
        }
    }
}
```

```text
10: Too small!
42: You win!
99: Too big!
```

Что происходит:

1. `use std::cmp::Ordering;` импортирует enum.
2. `guess.cmp(&secret)` сравнивает `guess` с `secret`. `&` перед `secret` заимствует значение (вы знаете это по `&mut input` из Урока 02.7; объясняет Этап 03).
3. У `match` есть ветка для каждого из трёх вариантов, записанных как `Ordering::Less` и так далее. Поскольку у `Ordering` ровно три значения, ветки покрывают всё, и ветка `_` не нужна. Если забыть одну, компилятор скажет, какую:

```text
error[E0004]: non-exhaustive patterns: `std::cmp::Ordering::Equal` not covered
 --> src/main.rs:5:11
  |
5 |     match guess.cmp(&secret) {
  |           ^^^^^^^^^^^^^^^^^^ pattern `std::cmp::Ordering::Equal` not covered
```

Функцию, возвращающую `Ordering`, легко тестировать: `assert_eq!(compare_guess(10, 50), Ordering::Less)` работает, потому что `Ordering` можно сравнивать и печатать.

### 2.3 Переменные окружения

**Переменная окружения** — это именованное текстовое значение, которое передаётся программе при запуске. В терминале её записывают перед командой:

```bash
GUESS_SECRET=42 cargo run -q
```

Программа на Rust читает её через `std::env::var("NAME")`. Результат — `Result`: `Ok(text)`, если переменная существует, и `Err(_)`, если нет:

```rust
fn main() {
    match std::env::var("GUESS_SECRET") {
        Ok(text) => println!("GUESS_SECRET = {text:?}"),
        Err(e) => println!("GUESS_SECRET is not set: {e}"),
    }
}
```

```text
GUESS_SECRET is not set: environment variable not found
```

а при запуске как `GUESS_SECRET=42 cargo run -q`:

```text
GUESS_SECRET = "42"
```

Зачем это нужно? Игру со случайным секретом нельзя проверить по фиксированному ожидаемому выводу. Если задана `GUESS_SECRET`, игра использует её как секретное число, и теперь можно писать **приёмочные тесты**: команду с известным вводом и известным выводом.

> 🔒 **Безопасность:** такая лазейка для тестов, как `GUESS_SECRET`, допустима в учебной игре, но настоящая программа никогда не должна позволять переменной окружения (или любому вводу) отключать защиту или менять секрет. На Этапе 10 вы узнаете, как обращаться с конфигурацией и секретами.

### 2.4 Рабочий процесс Git для этого проекта

Вы будете работать так, как работает команда:

1. Создайте **фича-ветку** для проекта: `git switch -c feature/guessing-game`.
2. Делайте **коммит в конце каждой вехи** с сообщением в стиле [Conventional Commits](https://www.conventionalcommits.org/en/v1.0.0/) (`feat(p02): …`, `test(p02): …`, `docs(p02): …`).
3. Когда всё сделано и проверено, вернитесь в `main`, **слейте** ветку командой `git merge --no-ff` (она всегда создаёт коммит слияния, так что в истории функция видна как одно целое) и добавьте **тег**.

В конце `git log --oneline --graph --decorate` должен выглядеть так (ваши хеши и число коммитов будут другими):

```text
*   ddc7cc5 (HEAD -> main, tag: p02-guessing-game-v0.1) feat(p02): merge the guessing game
|\  
| * e74cdf2 (feature/guessing-game) feat(p02): add parse_guess with tests
| * 72e71a2 feat(p02): add rand dependency
| * 2104478 feat(p02): create guessing game project
|/  
* c669cf5 chore: add .gitignore
```

## 3. Проект

### Что вы создадите

Программа командной строки `p02-guessing-game` в `~/projects/rust/code/p02-guessing-game`:

- компьютер выбирает секретное число от 1 до 100;
- у игрока 7 попыток;
- после каждой корректной догадки программа печатает, было ли число слишком маленьким или слишком большим;
- игрок выигрывает, угадав число, проигрывает после 7 неверных корректных догадок или может ввести `quit`;
- неверный ввод (текст, число вне диапазона) никогда не обрушивает программу и не тратит попытку.

### Контракт

**Запуск.** Программа печатает одну строку:

```text
Guess the number between 1 and 100! You have 7 attempts. Type 'quit' to give up.
```

**Секрет.** Если существует переменная окружения `GUESS_SECRET` и в ней записано целое число от 1 до 100, это число и есть секрет. Иначе секрет случайный, от 1 до 100 включительно.

**Каждый раунд.** Программа печатает `Attempt N of 7 - type your guess:`, где `N` — номер следующей попытки (1, 2, …), затем читает одну строку из стандартного ввода. Затем, в таком порядке:

| Строка (после обрезки пробелов) | Что делает программа |
|---|---|
| `quit` в любом регистре (`Quit`, `QUIT`) | печатает `You gave up. The number was S.` и завершается (`S` — секрет) |
| ввод закончился (EOF, End Of File, «конец файла»: читать больше нечего) | печатает `Input ended. The number was S.` и завершается |
| не целое число от 1 до 100 | печатает сообщение **в стандартный поток ошибок** (см. ниже) и начинает раунд заново **без траты попытки**. Текст счётчика попыток остаётся тем же. |
| корректная догадка, меньше секрета | засчитывает одну попытку; печатает `Too small!` |
| корректная догадка, больше секрета | засчитывает одну попытку; печатает `Too big!` |
| корректная догадка, равна секрету | засчитывает одну попытку; печатает `You win! You guessed it in N attempts.` (или `1 attempt.`, если N равно 1) и завершается |

**Сообщения о неверном вводе** идут в стандартный поток ошибок (`eprintln!`) в одной из таких форм:

```text
'abc' is not a whole number. Please type a whole number from 1 to 100.
0 is outside the range 1..=100. Please type a whole number from 1 to 100.
```

Первое — для текста, не являющегося числом (включая пустую строку, отрицательные числа и десятичные дроби); второе — для чисел, которые целые и неотрицательные, но вне диапазона 1–100.

**Проигрыш.** После 7 корректных догадок без выигрыша программа печатает `Game over! The number was S.` и завершается.

**Код выхода** всегда `0`. **Вывод** идёт в стандартный вывод, кроме сообщений о неверном вводе.

**Обязательная структура.** Логика должна находиться в функциях, которые можно тестировать, с точно такими именами и сигнатурами:

```rust
fn parse_guess(text: &str) -> Result<u32, String>;          // trims, parses, checks 1..=100
fn compare_guess(guess: u32, secret: u32) -> Ordering;
fn win_message(attempts: u32) -> String;                    // "You win! ..." with singular/plural
fn pick_secret() -> u32;                                    // GUESS_SECRET or random
```

**Критерии качества.** В проекте должно быть как минимум 6 модульных тестов в `src/main.rs` (`cargo test` проходит), ни одного предупреждения от `cargo clippy`, код отформатирован `cargo fmt`, короткий `README.md` в проекте и история Git с одним коммитом на каждую веху.

### Пошаговый план

#### M1 — Проект, ветка и зависимость

**Цель.** Новый проект с крейтом `rand` на фича-ветке.

**Руководство.**

1. В учебном репозитории создайте ветку `feature/guessing-game` (от `main`).
2. Выполните `cargo new code/p02-guessing-game`, затем `cargo add rand@0.10` внутри проекта.
3. В `src/main.rs` напечатайте случайное число от 1 до 100 строкой `The secret number is N` (временная строка для проверки).
4. Сделайте коммит.

<details>
<summary>Подсказка</summary>

Команды: `cd ~/projects/rust`, `git switch -c feature/guessing-game`, `cargo new code/p02-guessing-game`, `cd code/p02-guessing-game`, `cargo add rand@0.10`. Не забудьте `use rand::RngExt;`. Перед `git add` убедитесь, что в корневом `.gitignore` уже есть `target/` (Урок 02.1).

</details>

**Готово, когда:**

```bash
cd ~/projects/rust/code/p02-guessing-game
cargo run -q
cargo run -q
git branch --show-current
git log --oneline -n 1
```

печатает две строки вида `The secret number is N` (обычно с разными числами, каждое от 1 до 100), затем `feature/guessing-game`, затем коммит вроде `feat(p02): create guessing game project`.

#### M2 — Тестируемые функции, сначала тесты

**Цель.** Три чистые функции, написанные по принципу «сначала тест» (TDD, Test-Driven Development, разработка через тестирование, урок 02.8).

**Руководство.** Напишите функции `parse_guess`, `compare_guess` и `win_message` из контракта. Для каждой сначала напишите тесты в `#[cfg(test)] mod tests`, посмотрите, как они падают (используйте `todo!()`), затем напишите тело. Вынесите границы в константы: `MIN_NUMBER`, `MAX_NUMBER`, `MAX_ATTEMPTS`.

Тесты, которые нужно написать (минимум):

- `parse_guess`: `"1"`, `"100"` и `"  42\n"` принимаются; `"0"` и `"101"` отвергаются; `"abc"`, `""`, `"-5"` и `"3.5"` отвергаются.
- `compare_guess`: меньше, больше и равно.
- `win_message`: `1` даёт `You win! You guessed it in 1 attempt.`; `5` даёт `You win! You guessed it in 5 attempts.`

<details>
<summary>Подсказка</summary>

`parse_guess`: `match text.trim().parse::<u32>()`. Тип `u32` уже отвергает `-5` и `3.5`. Для диапазона используйте `(MIN_NUMBER..=MAX_NUMBER).contains(&n)` (метод `contains` диапазона) или два сравнения. `Err` несёт `String`, созданный через `format!`: `'abc' is not a whole number` или `0 is outside the range 1..=100` (первая часть сообщений из контракта; `main` добавляет `. Please type a whole number from 1 to 100.` при печати). `compare_guess` — одна строка: `guess.cmp(&secret)`. `win_message`: используйте `if attempts == 1 { … } else { … }` и верните `String` (`String::from("…")` или `format!`).

</details>

**Готово, когда:**

```bash
cargo test
```

печатает результат вроде:

```text
running 5 tests
test tests::compare_guess_orders_correctly ... ok
test tests::parse_guess_accepts_numbers_in_range ... ok
test tests::parse_guess_rejects_out_of_range ... ok
test tests::parse_guess_rejects_text_and_signs ... ok
test tests::win_message_uses_singular_and_plural ... ok

test result: ok. 5 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

(Тестов может быть больше; названия и порядок будут вашими. Компилятор может предупреждать, что функции «never used» (никогда не используются), пока не наступит M4. Пока это нормально.) Сделайте коммит: `feat(p02): add guess parsing and comparing with tests`.

#### M3 — Выбор секрета

**Цель.** `pick_secret()`, поддерживающая тестовую лазейку `GUESS_SECRET`.

**Руководство.** Напишите небольшую вспомогательную функцию `random_secret() -> u32` и `pick_secret() -> u32`. `pick_secret` читает `GUESS_SECRET` через `std::env::var`; если это корректный секрет (используйте `parse_guess`!), вернёт его; во всех остальных случаях (не задана, не число, вне диапазона) вернёт случайное число. Добавьте тест, что 1000 случайных секретов всегда в диапазоне. В `main` снова напечатайте `The secret number is N` для проверки.

<details>
<summary>Подсказка</summary>

Задачу решают два вложенных выражения `match`: внешнее — по `std::env::var("GUESS_SECRET")` с ветками `Ok(text)` и `Err(_)`, внутреннее — по `parse_guess(&text)` с ветками `Ok(n) => n` и `Err(_) => random_secret()`. Ветка `Err(_)` внешнего match тоже даёт `random_secret()`. Обратите внимание: `pick_secret` использует окружение, поэтому не пишите тест, который задаёт переменные (тесты выполняются параллельно; безопасные способы покажет Этап 09).

</details>

**Готово, когда:**

```bash
GUESS_SECRET=42 cargo run -q
GUESS_SECRET=abc cargo run -q
GUESS_SECRET=500 cargo run -q
cargo test
```

Первая команда печатает ровно `The secret number is 42`. Вторая и третья печатают `The secret number is N` со случайным `N` (они игнорируют плохое значение). `cargo test` проходит, включая новый тест. Сделайте коммит: `feat(p02): pick the secret number`.

#### M4 — Один раунд

**Цель.** Напечатать приветствие и сыграть ровно **один** раунд.

**Руководство.** В `main`: напечатайте стартовую строку; напечатайте `Attempt 1 of 7 - type your guess:`; прочитайте строку; разберите её через `parse_guess` (пока при ошибке напечатайте сообщение через `eprintln!` и остановитесь); сравните с секретом с помощью `match` по `Ordering`; напечатайте `Too small!`, `Too big!` или `win_message`. Удалите временную строку «The secret number is».

<details>
<summary>Подсказка</summary>

Используйте шаблон из Урока 02.7: `let mut line = String::new(); io::stdin().read_line(&mut line).expect("failed to read a line");`. В стартовой строке есть константы: `println!("Guess the number between {MIN_NUMBER} and {MAX_NUMBER}! You have {MAX_ATTEMPTS} attempts. Type 'quit' to give up.");`. Используйте `match parse_guess(&line)` с `Ok(n) => n` и `Err(message) => { eprintln!(...); return; }`, где `eprintln!` печатает `{message}. Please type a whole number from {MIN_NUMBER} to {MAX_NUMBER}.`.

</details>

**Готово, когда:**

```bash
echo 50 | GUESS_SECRET=42 cargo run -q
echo 42 | GUESS_SECRET=42 cargo run -q
```

```text
Guess the number between 1 and 100! You have 7 attempts. Type 'quit' to give up.
Attempt 1 of 7 - type your guess:
Too big!
```

```text
Guess the number between 1 and 100! You have 7 attempts. Type 'quit' to give up.
Attempt 1 of 7 - type your guess:
You win! You guessed it in 1 attempt.
```

Сделайте коммит: `feat(p02): play one round`.

#### M5 — Полная игра

**Цель.** Цикл с попытками, проигрышем, `quit`, концом ввода и неверным вводом.

**Руководство.** Превратите один раунд в цикл, который работает, пока `attempts < MAX_ATTEMPTS`. Считайте только корректные догадки. Обработайте в порядке контракта: конец ввода (`read_line` вернул `0`), `quit` (без учёта регистра: `eq_ignore_ascii_case`), неверный ввод (`eprintln!` и `continue`), затем сравнение. После цикла напечатайте `Game over! The number was S.` Помните, что делает `return` внутри `main`: завершает программу.

<details>
<summary>Подсказка</summary>

Создавайте **новый** `String` в начале каждой итерации. Используйте `let bytes = io::stdin().read_line(&mut line)...;` и `if bytes == 0 { … return; }`. Для проверки quit: `if line.trim().eq_ignore_ascii_case("quit")`. Ветка неверного ввода не должна увеличивать `attempts`, но и не должна печатать `Attempt 2`: печатается счётчик `attempts + 1`, а `attempts` не изменился. При выигрыше напечатайте `win_message(attempts)` и сделайте `return`.

</details>

**Готово, когда** (во всех командах используется `GUESS_SECRET=42`):

```bash
printf '50\n25\n37\n43\n40\n42\n' | GUESS_SECRET=42 cargo run -q
```

```text
Guess the number between 1 and 100! You have 7 attempts. Type 'quit' to give up.
Attempt 1 of 7 - type your guess:
Too big!
Attempt 2 of 7 - type your guess:
Too small!
Attempt 3 of 7 - type your guess:
Too small!
Attempt 4 of 7 - type your guess:
Too big!
Attempt 5 of 7 - type your guess:
Too small!
Attempt 6 of 7 - type your guess:
You win! You guessed it in 6 attempts.
```

```bash
printf 'abc\n0\n101\n\n42\n' | GUESS_SECRET=42 cargo run -q 2>&1
```

```text
Guess the number between 1 and 100! You have 7 attempts. Type 'quit' to give up.
Attempt 1 of 7 - type your guess:
'abc' is not a whole number. Please type a whole number from 1 to 100.
Attempt 1 of 7 - type your guess:
0 is outside the range 1..=100. Please type a whole number from 1 to 100.
Attempt 1 of 7 - type your guess:
101 is outside the range 1..=100. Please type a whole number from 1 to 100.
Attempt 1 of 7 - type your guess:
'' is not a whole number. Please type a whole number from 1 to 100.
Attempt 1 of 7 - type your guess:
You win! You guessed it in 1 attempt.
```

```bash
printf '10\nQuit\n' | GUESS_SECRET=42 cargo run -q
printf '10\n' | GUESS_SECRET=42 cargo run -q
printf '1\n2\n3\n4\n5\n6\n7\n8\n' | GUESS_SECRET=42 cargo run -q | tail -3
printf 'abc\n42\n' | GUESS_SECRET=42 cargo run -q 2>/dev/null
```

```text
Guess the number between 1 and 100! You have 7 attempts. Type 'quit' to give up.
Attempt 1 of 7 - type your guess:
Too small!
Attempt 2 of 7 - type your guess:
You gave up. The number was 42.
```

```text
Guess the number between 1 and 100! You have 7 attempts. Type 'quit' to give up.
Attempt 1 of 7 - type your guess:
Too small!
Attempt 2 of 7 - type your guess:
Input ended. The number was 42.
```

```text
Attempt 7 of 7 - type your guess:
Too small!
Game over! The number was 42.
```

```text
Guess the number between 1 and 100! You have 7 attempts. Type 'quit' to give up.
Attempt 1 of 7 - type your guess:
Attempt 1 of 7 - type your guess:
You win! You guessed it in 1 attempt.
```

Последний вывод показывает, что сообщение об `abc` ушло в stderr (скрытый `2>/dev/null`), а счётчик попыток остался равным 1. Сделайте коммит: `feat(p02): the full game loop`.

#### M6 — Завершение: качество, README, слияние и тег

**Цель.** Проект, который не стыдно показать.

**Руководство.**

1. Выполните `cargo fmt`, `cargo clippy` (без предупреждений), `cargo test` (всё проходит).
2. Напишите `README.md` в папке проекта (10–20 строк): что за игра, как запустить (`cargo run`), как играть, как тестировать (`cargo test`) и тестовую лазейку `GUESS_SECRET` с фразой о том, что она существует для тестирования.
3. Сыграйте в игру сами с настоящим случайным секретом и убедитесь, что числа ощущаются правильно.
4. Сделайте коммит (`docs(p02): add README`), затем вернитесь в `main`, слейте с `--no-ff`, поставьте тег на результат и удалите фича-ветку.

<details>
<summary>Подсказка</summary>

Команды: `git switch main`, `git merge --no-ff feature/guessing-game -m "feat(p02): merge the guessing game"`, `git tag -a p02-guessing-game-v0.1 -m "Guessing game finished"`, `git branch -d feature/guessing-game`. Если `cargo clippy` на что-то жалуется, прочитайте текст `help:`; большинство предложений короткие. Если вы используете `if let` (его изучает Этап 04), Clippy может предложить «свёрнутую» форму; вложенный `match` из M3 этого избегает.

</details>

**Готово, когда:**

```bash
cargo fmt --check && echo "fmt ok"
cargo clippy -q && echo "clippy ok"
cargo test 2>&1 | grep "test result"
git log --oneline --graph --decorate | head -8
```

печатает `fmt ok`, `clippy ok` (без предупреждений выше), строку `test result: ok. N passed; 0 failed`, где `N` не меньше 6, и граф, в котором у `main` есть коммит слияния с вашими коммитами вех на боковой ветке и тег `p02-guessing-game-v0.1`.

### Дополнительные цели

1. **Приглашение на той же строке.** Напечатайте приглашение без перевода строки (`print!`) и сбросьте буфер (Урок 02.7), чтобы игрок вводил сразу после `Your guess: `.
2. **Горячо и холодно.** Добавьте подсказку `Very close!`, когда догадка отличается от секрета не более чем на 3.
3. **Уровни сложности.** В начале спросите `easy` (1–50, 10 попыток), `normal` (1–100, 7 попыток) или `hard` (1–1000, 10 попыток). Замените константы переменными и вынесите настройки в функцию, возвращающую кортеж.
4. **Сыграть снова.** После игры спросите `Play again? (y/n)` и сделайте цикл.
5. **Лучший результат.** Храните наименьшее число попыток за сеанс и печатайте его при выигрыше.
6. **Тест поумнее.** Добавьте тест для функции `fn next_round_message(attempts_used: u32) -> String`, которую напишете вы и которая даёт `Attempt N of 7 - type your guess:`. Вынесите из `main` всё, что можно.
7. **Двоичный поиск.** Покажите, что 7 попыток достаточно для 1–100: вычислите `log2(100)` и объясните, почему лучшая стратегия — всегда угадывать середину оставшегося диапазона. Напишите небольшую функцию, которая играет в игру автоматически по этой стратегии, и проверьте, что ей никогда не нужно больше 7 догадок.

## Частые ошибки

- **Использование `.cmp()` без `&`:** `guess.cmp(secret)` — ошибка типов (`expected &u32, found u32`). Пишите `guess.cmp(&secret)`.
- **Забыли `use rand::RngExt;`:** ошибка `no method named random_range` с `items from traits can only be used if the trait is in scope`.
- **Неверный ввод засчитывается как попытка:** увеличивайте `attempts` только после того, как догадка корректна.
- **Повторное использование `String` для ввода:** `read_line` дописывает. Создавайте новый `String` на каждой итерации.
- **Нет обрезки пробелов:** `"42\n"` не разбирается. `parse_guess` должна делать trim.
- **Забыли `mut` у `attempts`:** `E0384 cannot assign twice to immutable variable`.
- **Коммит всего в конце:** один гигантский коммит никого ничему не учит. Делайте коммит на каждой вехе.
- **Коммит `target/`:** убедитесь, что в корневом `.gitignore` есть `target/`.
- **Бесконечный цикл на EOF:** если `read_line` вернул `0`, а вы не остановились, программа крутится вечно. Проверяйте число байтов.

> 🧪 **Тестирование:** функции из контракта чистые: они принимают значения и возвращают значения. Именно такое устройство кода облегчает тестирование. Часть ввода/вывода (`main`) мала и проверяется приёмочными командами из вех. Это разделение — шаблон, который вы будете использовать в каждом проекте курса.

> 📏 **Соглашение:** держите `main` коротким и читаемым. Константы для границ, функции с одной задачей, документирующие комментарии `///` у функций, тесты внизу файла, `cargo fmt` и `cargo clippy` перед каждым коммитом.

> ⚡ **Производительность:** игра ограничена человеком, а не компьютером; здесь нечего оптимизировать. Полезная привычка в любом случае: не вызывайте `rand::rng()` в горячем цикле, если можно создать генератор один раз. Для этой игры это неважно.

## Вопросы для самопроверки

1. Зачем программе нужна `GUESS_SECRET`?

<details>
<summary>Ответ</summary>

Случайный секрет делает вывод разным при каждом запуске, поэтому его нельзя проверить автоматически. С `GUESS_SECRET` секрет известен, и можно писать приёмочные тесты с фиксированным вводом и фиксированным ожидаемым выводом.

</details>

2. Что такое `Ordering` и каковы его значения?

<details>
<summary>Ответ</summary>

Enum из `std::cmp`, который возвращает `cmp`. Его значения: `Less`, `Equal` и `Greater`.

</details>

3. Чем отличаются `Cargo.toml` и `Cargo.lock`?

<details>
<summary>Ответ</summary>

`Cargo.toml` пишете вы, и он говорит, какие крейты и какие диапазоны версий вам нужны (`rand = "0.10"`). `Cargo.lock` пишет Cargo, и он записывает точные использованные версии, чтобы сборки были воспроизводимыми.

</details>

4. Почему `parse_guess`, `compare_guess` и `win_message` — отдельные функции, а не код внутри `main`?

<details>
<summary>Ответ</summary>

Функции с чёткими входами и выходами можно тестировать через `assert_eq!`, не набирая ничего вручную. `main`, который читает из терминала, остаётся небольшим.

</details>

5. Почему неверный ввод не должен тратить попытку?

<details>
<summary>Ответ</summary>

Так сказано в контракте игры: опечатка — это не догадка. Цикл увеличивает `attempts` только после корректной догадки.

</details>

6. Что делает `git merge --no-ff` и зачем он здесь используется?

<details>
<summary>Ответ</summary>

Он всегда создаёт коммит слияния, даже когда Git мог бы просто передвинуть указатель ветки. Тогда история показывает всю функцию (все коммиты вех) как одно целое, которое легко читать и при необходимости откатить.

</details>

## Итоги

- Хороший план делит программу на вехи, у каждой из которых есть цель и проверка.
- Добавляйте крейт командой `cargo add`; Cargo скачивает его, записывает точную версию в `Cargo.lock`, а нужные трейты вы импортируете (`use rand::RngExt;`).
- `Ordering` и `match` дают ясное трёхстороннее сравнение без забытых случаев.
- Переменные окружения (`std::env::var`) могут служить тестовой лазейкой, которая делает случайную программу тестируемой.
- Чистые функции и тонкий `main` делают код тестируемым; приёмочные тесты проверяют программу целиком через `printf … | cargo run`.
- Используйте фича-ветку, один коммит на веху, `merge --no-ff` и тег и держите `fmt`, `clippy` и `test` зелёными.

## Источники и что почитать

- [The Rust Programming Language — 2. Programming a Guessing Game](https://doc.rust-lang.org/book/ch02-00-guessing-game-tutorial.html) — официальное руководство, на котором основан этот проект (там он построен по-другому).
- [The Cargo Book — Specifying Dependencies](https://doc.rust-lang.org/cargo/reference/specifying-dependencies.html) — требования к версиям вроде `"0.10"`.
- [The Cargo Book — Cargo.toml vs Cargo.lock](https://doc.rust-lang.org/cargo/guide/cargo-toml-vs-cargo-lock.html) — зачем нужны оба файла и что коммитить.
- [rand on docs.rs](https://docs.rs/rand/0.10.4/rand/) — документация крейта `rand`, включая `RngExt`.
- [Standard library — std::cmp::Ordering](https://doc.rust-lang.org/std/cmp/enum.Ordering.html) — enum для сравнения значений.
- [Standard library — std::env::var](https://doc.rust-lang.org/std/env/fn.var.html) — чтение переменных окружения.
- [Pro Git — Basic Branching and Merging](https://git-scm.com/book/en/v2/Git-Branching-Basic-Branching-and-Merging) — рабочий процесс с ветками и слиянием, используемый в этом проекте.
- [Semantic Versioning 2.0.0](https://semver.org/) — правила номеров версий.

---

[🏠 Начало](../START-HERE.md) · [← 02.9 Основы отладки](09-debugging-basics.md) · [📋 Этап 02](README.md) · [02.11 Контрольная точка →](11-checkpoint.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
