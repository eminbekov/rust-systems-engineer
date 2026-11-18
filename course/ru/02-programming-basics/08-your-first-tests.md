<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 02.7 Чтение ввода и преобразование строк](07-reading-input-and-parsing.md) · [📋 Этап 02](README.md) · [02.9 Основы отладки →](09-debugging-basics.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/02-programming-basics/08-your-first-tests.md) <!-- nav -->

# Урок 02.8 — Ваши первые тесты

> **Этап 02 — Основы программирования на Rust** · Урок 8 из 11 · ⏱ около 3 часов (2 занятия)

## Что вы узнаете

- Зачем мы пишем тесты и как выглядит хороший тест.
- Как написать тест с помощью `#[test]` и запустить его командой `cargo test`.
- Макросы-утверждения: `assert!`, `assert_eq!`, `assert_ne!`.
- Как читать упавший тест и как проверить, что функция паникует.
- Как запускать только часть тестов и как работает разработка через тестирование (TDD).

## Перед началом

- Вы прошли [Урок 02.7 — Чтение ввода и преобразование строк](07-reading-input-and-parsing.md).
- Вы использовали `assert_eq!` внутри `main` в Уроках 02.1–02.7. В этом уроке вы перенесёте такие проверки в настоящие тесты.
- Вы умеете делать коммит Git ([Урок 01.3](../01-git-and-version-control/03-your-first-repository-and-commits.md)).

## Новые слова

| Слово | Значение |
|---|---|
| **тест** (test) | Небольшой фрагмент кода, который запускает ваш код и проверяет, что результат соответствует ожидаемому. |
| **модульный тест** (unit test) | Тест одной небольшой части (одной функции). Тесты в этом уроке — модульные. |
| **запускатель тестов** (test runner) | Программа, которая находит и запускает все тесты и сообщает результат. `cargo test` запускает запускатель Rust. |
| **утверждение** (assertion) | Проверка внутри теста: «это должно быть истинно». |
| **атрибут** (attribute) | Метка вида `#[name]`, которая даёт компилятору дополнительную информацию о следующем элементе. |
| **`#[test]`** | Атрибут, который помечает функцию как тест. |
| **`#[cfg(test)]`** | Атрибут, который говорит «компилировать следующий элемент только при запуске тестов». `cfg` — сокращение от «configuration» (конфигурация). |
| **модуль** (module) | Именованная коробка с кодом, записывается `mod name { … }`. Модули изучает Этап 04, урок 6. |
| **библиотечный крейт** (library crate) | Пакет кода, который может использовать другой код, без функции `main`. Его корневой файл — `src/lib.rs`. |
| **регрессия** (regression) | Ошибка, которая вернулась после исправления, или то, что раньше работало, а теперь сломано. |
| **крайний случай** (edge case) | Необычный ввод на границе допустимого: ноль, пустой текст, наибольшее значение. |
| **TDD** (Test-Driven Development, разработка через тестирование) | Способ работы: сначала пишется падающий тест, затем код, который делает его проходящим, затем наводится порядок. |
| **AAA** (Arrange, Act, Assert, «подготовить, выполнить, проверить») | Форма хорошего теста: подготовить данные, запустить код, проверить результат. |
| **CI** (Continuous Integration, непрерывная интеграция) | Сервис, который автоматически запускает ваши тесты при каждом изменении. Его изучает Этап 09. |

## 1. Зачем писать тесты?

До сих пор вы проверяли программы, запуская их и глядя на вывод. Для маленькой программы это работает, но у этого есть три проблемы:

1. Это **медленно**. После каждого изменения вы повторяете одни и те же шаги.
2. Это **забывчиво**. Через неделю вы не вспомните все случаи, которые проверяли.
3. Это не **защищает** вас. Вы меняете функцию A, а функция B (на которую вы не смотрели) молча ломается. Это **регрессия**.

**Тест** — это код, который проверяет код. Вы пишете проверку один раз, а компьютер повторяет её за долю секунды, каждый раз, всегда. Тесты дают:

- **Уверенность** при изменении кода: если тесты проходят, вы, вероятно, ничего не сломали.
- **Документацию**: тест показывает, как предполагается использовать функцию.
- **Обратную связь по дизайну**: код, который трудно тестировать, обычно плохо спроектирован (поэтому в Уроке 02.7 мы отделили чтение от разбора).

Профессиональные команды запускают все тесты автоматически при каждом изменении (**CI**, непрерывная интеграция — Этап 09). Ваш первый шаг — выучить, как выглядит тест в Rust.

## 2. Тестовый проект

Тестирование встроено в язык Rust и в Cargo. Создайте для этого урока новый **библиотечный** проект:

```bash
cd ~/projects/rust
cargo new code/p02-first-tests --lib
cd code/p02-first-tests
cat src/lib.rs
```

`--lib` создаёт **библиотечный крейт**: пакет функций без `main`, корневой файл которого — `src/lib.rs`. Cargo написал для вас пример:

```rust
pub fn add(left: u64, right: u64) -> u64 {
    left + right
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn it_works() {
        let result = add(2, 2);
        assert_eq!(result, 4);
    }
}
```

Запустите тест командой:

```bash
cargo test
```

```text
   Compiling p02-first-tests v0.1.0 (/Users/you/projects/rust/code/p02-first-tests)
    Finished `test` profile [unoptimized + debuginfo] target(s) in 0.19s
     Running unittests src/lib.rs (target/debug/deps/p02_first_tests-3c41ed7b09edacbb)

running 1 test
test tests::it_works ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s

   Doc-tests p02_first_tests

running 0 tests

test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

(Хеш после `p02_first_tests-` на вашем компьютере другой.) Как это читать:

1. `Running unittests src/lib.rs` — Cargo собрал специальную **тестовую версию** вашего кода и запускает модульные тесты, найденные в этом файле.
2. `running 1 test` и `test tests::it_works ... ok` — один тест с именем `tests::it_works` (модуль `tests`, функция `it_works`). Он прошёл.
3. `test result: ok. 1 passed; 0 failed; 0 ignored; …` — итог. Если что-то падает, строка начинается с `FAILED`, а `cargo test` завершается с ошибкой (ненулевой код выхода), по этому CI и узнаёт о проблеме.
4. `Doc-tests` — Cargo также запускает примеры кода, написанные в документирующих комментариях `///` (Урок 02.6). У нас их нет, поэтому `running 0 tests`.

## 3. Устройство теста

Разберём шаблон построчно:

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn it_works() {
        let result = add(2, 2);
        assert_eq!(result, 4);
    }
}
```

1. `mod tests { … }` создаёт **модуль** — именованную коробку — с именем `tests`. Тесты живут в своей коробке рядом с кодом, который они проверяют, в том же файле.
2. `#[cfg(test)]` — это **атрибут**. Он говорит «компилировать этот модуль только для `cargo test`». В итоговой программе тестов нет, поэтому они ничего не стоят во время работы.
3. `use super::*;` приносит всё из родительского модуля (`super` = «коробка вокруг меня») в коробку теста. Без этого тест не видит `add`. (Помните: имена существуют только в своей области видимости, Урок 02.1.)
4. `#[test]` помечает следующую функцию как тест. Запускаются только функции с этим атрибутом. Тест не принимает параметров и ничего не возвращает.
5. Тело выполняет три шага **AAA**: *Arrange* (подготовить входные данные), *Act* (вызвать код), *Assert* (проверить результат).
6. Тест **проходит**, если функция завершается. Он **падает**, если функция паникует. Ложное утверждение паникует. Вот и весь механизм.

Если забыть `use super::*;`, получится знакомая по виду ошибка. Здесь в файле есть функция `double` (она удваивает число) и тест, который вызывает `double(4)`, но в тестовом модуле нет `use super::*;`:

```text
error[E0425]: cannot find function `double` in this scope
  --> src/lib.rs:10:20
   |
10 |         assert_eq!(double(4), 8);
   |                    ^^^^^^ not found in this scope
   |
help: consider importing this function
   |
 7 +     use crate::double;
   |
```

## 4. Пишем настоящие тесты

Замените `src/lib.rs` этими функциями и тестами:

```rust
use std::num::ParseIntError;

/// Returns true when `n` is even.
pub fn is_even(n: i32) -> bool {
    n % 2 == 0
}

/// Divides `a` by `b`. Panics when `b` is zero.
pub fn divide(a: i32, b: i32) -> i32 {
    if b == 0 {
        panic!("cannot divide by zero");
    }
    a / b
}

/// Converts degrees Celsius to degrees Fahrenheit.
pub fn celsius_to_fahrenheit(c: f64) -> f64 {
    c * 9.0 / 5.0 + 32.0
}

/// Parses a whole number from text, ignoring spaces around it.
pub fn parse_number(text: &str) -> Result<i32, ParseIntError> {
    text.trim().parse()
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn even_numbers_are_even() {
        assert!(is_even(4));
        assert!(is_even(0));
        assert!(is_even(-2));
    }

    #[test]
    fn odd_numbers_are_not_even() {
        assert!(!is_even(7));
        assert!(!is_even(-3), "-3 should not be even");
    }

    #[test]
    fn divide_works() {
        assert_eq!(divide(10, 2), 5);
        assert_eq!(divide(7, 2), 3);
        assert_ne!(divide(7, 2), 4);
    }

    #[test]
    #[should_panic(expected = "cannot divide by zero")]
    fn divide_by_zero_panics() {
        divide(1, 0);
    }

    #[test]
    fn boiling_point() {
        assert_eq!(celsius_to_fahrenheit(100.0), 212.0);
    }

    #[test]
    fn body_temperature_with_tolerance() {
        let f = celsius_to_fahrenheit(36.6);
        assert!((f - 97.88).abs() < 1e-9, "got {f}");
    }

    #[test]
    fn parse_number_accepts_spaces() {
        assert_eq!(parse_number("  42\n"), Ok(42));
    }

    #[test]
    fn parse_number_rejects_text() {
        assert!(parse_number("abc").is_err());
        assert!(parse_number("").is_err());
    }

    #[test]
    #[ignore = "slow example, run with --ignored"]
    fn a_slow_test() {
        let mut sum: u64 = 0;
        for i in 0..1_000_000u64 {
            sum += i;
        }
        assert_eq!(sum, 499_999_500_000);
    }
}
```

```text
running 9 tests
test tests::a_slow_test ... ignored, slow example, run with --ignored
test tests::body_temperature_with_tolerance ... ok
test tests::boiling_point ... ok
test tests::parse_number_accepts_spaces ... ok
test tests::even_numbers_are_even ... ok
test tests::divide_works ... ok
test tests::parse_number_rejects_text ... ok
test tests::odd_numbers_are_not_even ... ok
test tests::divide_by_zero_panics - should panic ... ok

test result: ok. 8 passed; 0 failed; 1 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

(Порядок строк может быть разным при каждом запуске: тесты выполняются параллельно, в нескольких потоках.)

Что происходит:

1. `assert!(condition)` проходит, если условие равно `true`. Можно добавить сообщение, которое показывается при падении, по тем же правилам формата, что и в `println!`: `assert!(!is_even(-3), "-3 should not be even")`.
2. `assert_eq!(left, right)` проходит, если значения равны; `assert_ne!(left, right)` проходит, если они различаются. При падении они печатают оба значения, поэтому `assert_eq!` лучше, чем `assert!(a == b)`.
3. У двух значений должен быть тип, который можно сравнивать (`PartialEq`) и показывать через `{:?}` (`Debug`). У всех типов этого этапа есть и то и другое. (Эти трейты объясняет Этап 05.)
4. `#[should_panic(expected = "…")]` помечает тест, который **проходит, только если код паникует**, а (с `expected`) только если сообщение паники содержит этот текст. Так проверяются пути «ошибок».
5. `#[ignore]` по умолчанию пропускает тест (удобно для медленных тестов); игнорируемые тесты запускаются командой `cargo test -- --ignored`.
6. Названия тестов — предложения, говорящие, **что должно быть истинно**: `parse_number_rejects_text`. Когда тест падает, название уже подсказывает, что сломалось.
7. Числа с плавающей точкой сравниваются с допуском (Урок 02.2). Сообщение `"got {f}"` появляется, только если утверждение ложно.

## 5. Как читать упавший тест

Тест, который никогда не падает, ничего вам не говорит. Сломаем код намеренно: изменим `is_even` на `n % 2 == 1`.

```text
running 9 tests
test tests::a_slow_test ... ignored, slow example, run with --ignored
test tests::body_temperature_with_tolerance ... ok
test tests::parse_number_rejects_text ... ok
test tests::parse_number_accepts_spaces ... ok
test tests::divide_works ... ok
test tests::boiling_point ... ok
test tests::odd_numbers_are_not_even ... FAILED
test tests::divide_by_zero_panics - should panic ... ok
test tests::even_numbers_are_even ... FAILED

failures:

---- tests::odd_numbers_are_not_even stdout ----

thread 'tests::odd_numbers_are_not_even' (4113760) panicked at src/lib.rs:39:9:
assertion failed: !is_even(7)

---- tests::even_numbers_are_even stdout ----

thread 'tests::even_numbers_are_even' (4113759) panicked at src/lib.rs:32:9:
assertion failed: is_even(4)


failures:
    tests::even_numbers_are_even
    tests::odd_numbers_are_not_even

test result: FAILED. 6 passed; 2 failed; 1 ignored; 0 measured; 0 filtered out; finished in 0.00s

error: test failed, to rerun pass `--lib`
```

Как это читать:

1. В списке `FAILED` у двух тестов. Остальные по-прежнему `ok`, значит, ошибка ограничена `is_even`.
2. Для каждого падения вы видите файл, строку и столбец (`src/lib.rs:32:9`) и утверждение, которое оказалось ложным: `assertion failed: is_even(4)`.
3. Итог говорит `FAILED. 6 passed; 2 failed`.
4. Последняя строка, `error: test failed`, показывает, что команда завершилась неудачей (код выхода не ноль).

Падение `assert_eq!` также печатает обе стороны. Чтобы увидеть это, добавьте в `src/lib.rs` небольшой пример: функцию `double` со специально оставленной ошибкой (она возвращает `n * 2 + 1`) и тест для неё. Раздел 6 использует этот же тест.

```rust
/// Удваивает число. (В этой версии специально оставлена ошибка.)
pub fn double(n: i32) -> i32 {
    n * 2 + 1
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn doubles() {
        println!("testing double(4)");
        assert_eq!(double(4), 8, "double(4) should be 8");
    }
}
```

```text
assertion `left == right` failed: double(4) should be 8
  left: 9
 right: 8
```

`left` — первое значение, переданное в `assert_eq!` (то, что вернул код, `9`), а `right` — второе (то, что вы ожидали, `8`). Посмотрев на падение, исправьте `double`, чтобы она возвращала `n * 2`.

И два способа, которыми падает тест `should_panic`. Они взяты из двух небольших тестов, которые здесь не показаны: `no_panic_happens` ожидает панику, которой не происходит, а `wrong_message` ожидает сообщение паники со словом `negative`, но настоящее сообщение другое:

```text
---- tests::no_panic_happens stdout ----
note: test did not panic as expected at src/lib.rs:30:8
---- tests::wrong_message stdout ----

thread 'tests::wrong_message' (4115500) panicked at src/lib.rs:7:9:
cannot divide by zero
note: panic did not contain expected string
      panic message: "cannot divide by zero"
 expected substring: "negative"
```

Исправьте код, снова запустите `cargo test` и смотрите, как красное становится зелёным.

## 6. Запуск части тестов и просмотр вывода

```bash
cargo test divide                 # only tests whose name contains "divide"
cargo test -- --ignored           # only the ignored tests
cargo test -- --list              # list the tests without running them
cargo test doubles -- --nocapture # show println! output of passing tests
```

```text
running 2 tests
test tests::divide_works ... ok
test tests::divide_by_zero_panics - should panic ... ok
test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 7 filtered out; finished in 0.00s
```

Это результат `cargo test divide`: текст после `cargo test` — **фильтр** по именам тестов; `7 filtered out` — тесты, которые не подошли (вы увидите `8`, если добавили пример `doubles` из раздела 5).

По умолчанию запускатель **скрывает** то, что проходящие тесты печатают через `println!` (чтобы отчёт был коротким), и показывает это только для упавших. Флаг `-- --nocapture` (после двойного тире, которое отделяет флаги Cargo от флагов запускателя) показывает всё:

```text
testing double(4)
test tests::doubles ... ok
```

Поскольку тесты выполняются **параллельно**, они должны быть **независимыми**: ни один тест не может полагаться на результат другого или на порядок. Тест, который работает только после другого, — ошибка, ждущая своего часа. (`cargo test -- --test-threads=1` запускает по одному, для отладки.)

## 7. Тесты в программе с `main` и в других местах

В бинарном крейте (проект с `src/main.rs`, как игра «Угадай число») модульные тесты пишутся так же:

```rust
fn add(a: i32, b: i32) -> i32 {
    a + b
}

fn main() {
    println!("2 + 3 = {}", add(2, 3));
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn add_works() {
        assert_eq!(add(2, 3), 5);
    }
}
```

```text
     Running unittests src/main.rs (target/debug/deps/bincrate-5894bf05e1c343f8)

running 1 test
test tests::add_works ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

В вашем учебном проекте `p02-basics`, где каждый файл в `src/bin/` — это программа, один файл можно тестировать командой `cargo test --bin 04_functions`.

В Rust есть три вида тестов (подробно их разбирает Этап 05, урок 6):

| Вид | Где | Что проверяет |
|---|---|---|
| Модульные тесты | `#[cfg(test)] mod tests` в том же файле | одну функцию или модуль, включая приватные элементы |
| Интеграционные тесты | файлы в папке `tests/` | публичные части вашей библиотеки так, как их видит пользователь |
| Doc-тесты | примеры кода в комментариях `///` | что документация верна |

## 8. Что делает тест хорошим

- **Одно поведение на тест**, с названием, говорящим, что должно быть истинно: `parse_number_rejects_text`, а не `test1`.
- **Подготовить, выполнить, проверить**: подготовка, вызов, проверка. Делайте каждый тест коротким.
- **Проверяйте границы**: ноль, один, максимум, пустой текст, отрицательное число, наибольшее значение плюс один. Большинство ошибок живёт там. Список для функции над числом: `0`, `1`, `-1`, `MAX`, `MIN`.
- **Проверяйте отказы**: неверный ввод должен давать ошибку, которую вы задумали (`is_err()`, `#[should_panic]`).
- **Будьте детерминированными**: тест должен давать один и тот же результат каждый раз. Никакой случайности, текущего времени, сети (Этап 09 покажет, как с этим справляться).
- **Проверяйте поведение, а не детали**: проверяйте, *что* возвращает функция, а не *как* она устроена внутри; иначе каждый рефакторинг ломает ваши тесты.
- **Никогда не удаляйте и не игнорируйте падающий тест только ради зелёного результата.** Тест вам что-то сообщает.

> 📏 **Соглашение:** размещайте модульные тесты внизу файла в `#[cfg(test)] mod tests`. Называйте тестовые функции предложениями в `snake_case`.

> 🔒 **Безопасность:** тесты — инструмент безопасности. Напишите тест для каждого правила безопасности («возраст выше 150 отвергается», «пустое имя отвергается», «слишком большое число не переполняется»). Если кто-то позже нарушит правило, тест упадёт раньше, чем код дойдёт до пользователей.

## 9. Разработка через тестирование (TDD)

В **TDD** вы пишете тест **до** кода. Цикл состоит из трёх шагов:

1. **Красный** — напишите тест для функции, которая пока не работает. Запустите и посмотрите, как он падает.
2. **Зелёный** — напишите самый простой код, при котором тест проходит.
3. **Рефакторинг** — наведите порядок в коде, запуская тесты, чтобы убедиться, что ничего не сломалось.

Построим `grade(score: u32) -> char`: 90 и выше — `'A'`, 80 и выше — `'B'`, 70 и выше — `'C'`, всё остальное — `'F'`. Сначала тесты и тело функции, которое говорит «ещё не написано» — `todo!()` — макрос, который паникует с сообщением `not yet implemented`:

```rust
/// Turns a score from 0 to 100 into a letter grade.
pub fn grade(score: u32) -> char {
    todo!()
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn ninety_and_above_is_a() {
        assert_eq!(grade(100), 'A');
        assert_eq!(grade(90), 'A');
    }

    #[test]
    fn eighty_to_eighty_nine_is_b() {
        assert_eq!(grade(89), 'B');
        assert_eq!(grade(80), 'B');
    }

    #[test]
    fn seventy_to_seventy_nine_is_c() {
        assert_eq!(grade(79), 'C');
        assert_eq!(grade(70), 'C');
    }

    #[test]
    fn below_seventy_is_f() {
        assert_eq!(grade(69), 'F');
        assert_eq!(grade(0), 'F');
    }
}
```

**Красный:** `cargo test` показывает

```text
running 4 tests
test tests::seventy_to_seventy_nine_is_c ... FAILED
test tests::below_seventy_is_f ... FAILED
test tests::eighty_to_eighty_nine_is_b ... FAILED
test tests::ninety_and_above_is_a ... FAILED
...
thread 'tests::below_seventy_is_f' (4114830) panicked at src/lib.rs:3:5:
not yet implemented
...
test result: FAILED. 0 passed; 4 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

(Компилятор также предупреждает о неиспользуемой переменной `score`.) **Зелёный:** напишите код:

```rust
pub fn grade(score: u32) -> char {
    if score >= 90 {
        'A'
    } else if score >= 80 {
        'B'
    } else if score >= 70 {
        'C'
    } else {
        'F'
    }
}
```

```text
running 4 tests
test tests::below_seventy_is_f ... ok
test tests::eighty_to_eighty_nine_is_b ... ok
test tests::ninety_and_above_is_a ... ok
test tests::seventy_to_seventy_nine_is_c ... ok
test result: ok. 4 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

**Рефакторинг:** код уже короткий; оставляем как есть. Обратите внимание, что тесты проверяют *границы* (89 и 90, 79 и 80), где неверное `>` вместо `>=` было бы сразу найдено.

## Частые ошибки

**1. Забыли `#[test]`.** Функция компилируется, но никогда не запускается, а `cargo test` говорит `running 0 tests`. Всегда проверяйте, что число тестов — то, которое вы ожидаете.

**2. Забыли `use super::*;`.** `E0425 cannot find function`.

**3. Мысль, что тест видит только элементы `pub`.** Модульный тест в том же файле может вызывать и приватные функции. (Интеграционные тесты в папке `tests/` видят только элементы `pub`.)

**4. Тесты, зависящие друг от друга или от порядка.** Они выполняются параллельно, в любом порядке.

**5. Сравнение чисел с плавающей точкой через `assert_eq!`.** Используйте допуск, если только значения не представимы точно.

**6. Тест, который не может упасть.** Например `assert!(true)` или проверка значения, которое вы только что вычислили тем же кодом. Всегда спрашивайте себя: «какую ошибку поймает этот тест?»

**7. Проверка только «счастливого пути».** Добавьте пустые, нулевые, огромные и неверные входные данные.

> ⚡ **Производительность:** тесты компилируются без оптимизаций (`unoptimized + debuginfo`), поэтому тяжёлые вычисления в тестах медленны; `cargo test --release` собирает с оптимизациями. Держите модульные тесты быстрыми: весь набор должен выполняться за секунды, чтобы вы запускали его постоянно.

## Попробуйте сами

1. Создайте `code/p02-first-tests`, как показано в разделе 2, и выполните `cargo test`. Прочитайте каждую строку отчёта.
2. Замените `src/lib.rs` кодом из раздела 4. Выполните `cargo test`, затем `cargo test divide`, затем `cargo test -- --ignored`, затем `cargo test -- --list`.
3. Намеренно сломайте `is_even` (`== 1`), прочитайте сообщение о падении, затем исправьте.
4. Добавьте новую функцию `fn is_positive(n: i32) -> bool` с тремя тестами (положительное, ноль, отрицательное) и проверьте, что ноль *не* положителен.
5. Вернитесь в проект `p02-basics`: перенесите проверки `assert_eq!` одной функции из Урока 02.4 (например, `is_even`) в `#[cfg(test)] mod tests` в том же файле `src/bin/` и выполните `cargo test --bin <name>`.
6. Выполните `cargo fmt` и `cargo clippy`. Сделайте коммит:

```bash
cd ~/projects/rust
git add code/p02-first-tests code/p02-basics
git commit -m "test(p02): add first unit tests"
```

Тип коммита `test` — стандартная часть Conventional Commits ([Урок 01.9](../01-git-and-version-control/09-teamwork-pull-requests-and-conventions.md)).

## Упражнения

### Упражнение 1 — Тесты для високосных лет (легко)

Функция `is_leap_year` дана. **Напишите тесты** (не функцию): 2024 и 2000 — високосные годы; 2023 и 1900 — нет. Используйте `assert!` и `assert!(!…)`.

```rust
pub fn is_leap_year(year: u32) -> bool {
    (year % 4 == 0 && year % 100 != 0) || year % 400 == 0
}
```

<details>
<summary>Подсказка</summary>

Создайте `#[cfg(test)] mod tests { use super::*; … }` и одну `#[test] fn` с четырьмя утверждениями или четыре маленьких теста с именами-предложениями.

</details>

<details>
<summary>Ответ</summary>

```rust
pub fn is_leap_year(year: u32) -> bool {
    (year % 4 == 0 && year % 100 != 0) || year % 400 == 0
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn leap_years() {
        assert!(is_leap_year(2024));
        assert!(is_leap_year(2000));
        assert!(!is_leap_year(2023));
        assert!(!is_leap_year(1900));
    }
}
```

Почему это работает: 1900 и 2000 — интересные случаи правила «делится на 100 — не високосный, если только не делится на 400». Хороший тест включает их.

</details>

### Упражнение 2 — Найдите ошибку с помощью тестов (легко)

В этой функции есть ошибка, и тесты её находят. Запустите тесты, прочитайте сообщение о падении и исправьте функцию (не тесты).

```rust
pub fn fahrenheit_to_celsius(f: f64) -> f64 {
    (f - 32.0) * 9.0 / 5.0
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn fahrenheit_conversion() {
        assert_eq!(fahrenheit_to_celsius(32.0), 0.0);
        assert_eq!(fahrenheit_to_celsius(212.0), 100.0);
        assert!((fahrenheit_to_celsius(98.6) - 37.0).abs() < 1e-9);
    }
}
```

<details>
<summary>Подсказка</summary>

Падение показывает `left: 324.0`, `right: 100.0`. Множитель перевёрнут: градусы Цельсия меньше градусов Фаренгейта, поэтому множитель должен быть меньше 1.

</details>

<details>
<summary>Ответ</summary>

```rust
pub fn fahrenheit_to_celsius(f: f64) -> f64 {
    (f - 32.0) * 5.0 / 9.0
}
```

Вывод падения до исправления:

```text
assertion `left == right` failed
  left: 324.0
 right: 100.0
```

Почему это работает: при неверном множителе `(212 - 32) * 9 / 5 = 324`; с `5 / 9` получается `180 * 5 / 9 = 100`. Первое утверждение (`32 → 0`) проходило даже с ошибкой, потому что `0`, умноженный на любой множитель, — это `0` — напоминание проверять больше одного значения.

</details>

### Упражнение 3 — TDD: числа-палиндромы (средне)

Используйте TDD, чтобы написать `pub fn is_palindrome_number(n: u32) -> bool` (число, одинаково читающееся с обеих сторон: `121`, `12321`; но не `10`). Сначала напишите тесты для `0`, `7`, `121`, `12321` (истина) и `10`, `123` (ложь) и посмотрите, как они падают (используйте `todo!()` в теле). Затем реализуйте функцию циклом `while`, который разворачивает число.

<details>
<summary>Подсказка</summary>

Чтобы развернуть: держите `reversed = 0` и `rest = n`. Пока `rest > 0`: `reversed = reversed * 10 + rest % 10;` и `rest /= 10;`. В конце сравните `reversed == n`.

</details>

<details>
<summary>Ответ</summary>

```rust
pub fn is_palindrome_number(n: u32) -> bool {
    let mut rest = n;
    let mut reversed = 0;
    while rest > 0 {
        reversed = reversed * 10 + rest % 10;
        rest /= 10;
    }
    reversed == n
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn palindromes() {
        assert!(is_palindrome_number(0));
        assert!(is_palindrome_number(7));
        assert!(is_palindrome_number(121));
        assert!(is_palindrome_number(12321));
        assert!(!is_palindrome_number(10));
        assert!(!is_palindrome_number(123));
    }
}
```

Почему это работает: для `121` цикл строит `1`, затем `12`, затем `121`. Для `10` он строит `0`, затем `1`, и `1 != 10`. Случай `0` работает, потому что цикл не выполняется, и `reversed == 0 == n`.

Ещё один крайний случай, о котором стоит подумать: для очень большого `n`, например `4_000_000_009`, перевёрнутое число `9_000_000_004` не помещается в `u32`, и программа паникует с сообщением `attempt to multiply with overflow`. Если сделать `reversed` типа `u64`, проблема исчезает: напишите `let mut reversed: u64 = 0;`, добавляйте каждую цифру как `(rest % 10) as u64` и сравнивайте с `n as u64` (`as` превращает число в другой тип, Урок 02.2). Именно такие крайние случаи и ловят тесты.

</details>

### Упражнение 4 — Проверка паники (средне)

Напишите `pub fn factorial(n: u64) -> u64`, которая перемножает `1 * 2 * … * n` и **паникует с сообщением, содержащим `too large`**, когда `n > 20` (потому что `21!` не помещается в `u64`). Напишите тесты для `0`, `1`, `5`, `20` (`20! = 2_432_902_008_176_640_000`) и тест `#[should_panic(expected = "too large")]` для `21`.

<details>
<summary>Подсказка</summary>

Используйте `panic!("{n}! is too large for u64");` в начале. Цикл `for i in 2..=n` умножает `result`, который начинается с `1`; для `n = 0` или `1` диапазон `2..=n` пуст.

</details>

<details>
<summary>Ответ</summary>

```rust
pub fn factorial(n: u64) -> u64 {
    if n > 20 {
        panic!("{n}! is too large for u64");
    }
    let mut result = 1;
    for i in 2..=n {
        result *= i;
    }
    result
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn factorials() {
        assert_eq!(factorial(0), 1);
        assert_eq!(factorial(1), 1);
        assert_eq!(factorial(5), 120);
        assert_eq!(factorial(20), 2_432_902_008_176_640_000);
    }

    #[test]
    #[should_panic(expected = "too large")]
    fn factorial_of_21_panics() {
        factorial(21);
    }
}
```

Почему это работает: защита делает отказ явным и ранним, с понятным сообщением, вместо невнятного `attempt to multiply with overflow` посреди цикла. Тест `should_panic` фиксирует это поведение.

</details>

### Упражнение 5 — Сначала тесты для разбора процентов (сложно)

Опишите `pub fn parse_percent(text: &str) -> Result<u8, String>` **через её тесты**, затем реализуйте. Она принимает целое число от 0 до 100 с необязательным `%` в конце и необязательными пробелами. Тесты, которые должны проходить: `"50%"` → `Ok(50)`, `"50"` → `Ok(50)`, `" 7 % "` → `Ok(7)`, `"0"` → `Ok(0)`, `"100%"` → `Ok(100)`. Тесты на ошибки: `"101%"`, `"-1"`, `"%"`, `""`, `"abc"`, `"5 0"`.

<details>
<summary>Подсказка</summary>

Сначала очистите текст: `text.trim().trim_end_matches('%').trim()`. (`trim_end_matches('%')` убирает символы `%` в конце.) Затем `parse::<u8>()` и проверка `n <= 100`. Тип `u8` уже отвергает `-1`.

</details>

<details>
<summary>Ответ</summary>

```rust
pub fn parse_percent(text: &str) -> Result<u8, String> {
    let cleaned = text.trim().trim_end_matches('%').trim();
    match cleaned.parse::<u8>() {
        Ok(n) => {
            if n <= 100 {
                Ok(n)
            } else {
                Err(format!("{n} is more than 100"))
            }
        }
        Err(_) => Err(format!("'{}' is not a percentage", text.trim())),
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn percent_accepts_valid_text() {
        assert_eq!(parse_percent("50%"), Ok(50));
        assert_eq!(parse_percent("50"), Ok(50));
        assert_eq!(parse_percent(" 7 % "), Ok(7));
        assert_eq!(parse_percent("0"), Ok(0));
        assert_eq!(parse_percent("100%"), Ok(100));
    }

    #[test]
    fn percent_rejects_bad_text() {
        assert!(parse_percent("101%").is_err());
        assert!(parse_percent("-1").is_err());
        assert!(parse_percent("%").is_err());
        assert!(parse_percent("").is_err());
        assert!(parse_percent("abc").is_err());
        assert!(parse_percent("5 0").is_err());
    }
}
```

Почему это работает: `"%"` превращается в пустой текст, который не разбирается; `"5 0"` сохраняет внутренний пробел, поэтому это не число. Тесты описывают все правила, так что позже вы можете менять реализацию без страха.

</details>

## Вопросы для самопроверки

1. Что делает `#[test]`?

<details>
<summary>Ответ</summary>

Помечает функцию как тест. `cargo test` находит и запускает только функции с этим атрибутом.

</details>

2. Когда тест проходит, а когда падает?

<details>
<summary>Ответ</summary>

Он проходит, если функция завершается без паники. Он падает, если паникует (например, потому что утверждение ложно).

</details>

3. Чем отличается `assert!` от `assert_eq!`?

<details>
<summary>Ответ</summary>

`assert!(cond)` проверяет `bool`. `assert_eq!(a, b)` проверяет, что два значения равны, и при падении печатает оба, что облегчает поиск ошибки.

</details>

4. Что делают `#[cfg(test)]` и `use super::*;`?

<details>
<summary>Ответ</summary>

`#[cfg(test)]` компилирует модуль только при тестировании. `use super::*;` импортирует все имена родительского модуля (тестируемого кода) в тестовый модуль.

</details>

5. Как проверить, что функция паникует на неверном вводе?

<details>
<summary>Ответ</summary>

Добавьте `#[should_panic]` к тесту, а лучше `#[should_panic(expected = "text of the message")]`. Тест проходит, только если код паникует.

</details>

6. Как запустить только тесты со словом `divide` в названии?

<details>
<summary>Ответ</summary>

`cargo test divide`.

</details>

7. Каковы три шага TDD?

<details>
<summary>Ответ</summary>

Красный (написать падающий тест), зелёный (написать самый простой код, при котором он проходит), рефакторинг (навести порядок, пока тесты остаются зелёными).

</details>

8. Почему тесты не должны зависеть друг от друга?

<details>
<summary>Ответ</summary>

Запускатель выполняет тесты параллельно и в любом порядке, поэтому на результат одного теста нельзя полагаться в другом.

</details>

## Итоги

- Тест — это функция с `#[test]`, лежащая в `#[cfg(test)] mod tests` с `use super::*;`. Все тесты запускает `cargo test`.
- Тест проходит, если не паникует. Используйте `assert!`, `assert_eq!`, `assert_ne!` (с необязательным сообщением) и `#[should_panic(expected = "…")]` для паник.
- Читайте падения сверху: имя теста, строка, затем `left` и `right`.
- Фильтруйте с помощью `cargo test name`; `-- --nocapture` показывает вывод; `#[ignore]` и `-- --ignored` работают с медленными тестами.
- Хорошие тесты: по одному поведению, крайние случаи, плохой ввод, детерминированность, независимость.
- TDD: красный, зелёный, рефакторинг.
- Модульные тесты живут рядом с кодом; интеграционные — в `tests/`; doc-тесты — в комментариях `///` (Этап 05).

## Источники и что почитать

- [The Rust Programming Language — 11.1 How to Write Tests](https://doc.rust-lang.org/book/ch11-01-writing-tests.html) — глава официальной книги: `#[test]`, утверждения и `should_panic`.
- [The Rust Programming Language — 11.2 Controlling How Tests Are Run](https://doc.rust-lang.org/book/ch11-02-running-tests.html) — фильтры, `--nocapture`, `--ignored` и потоки.
- [The Rust Programming Language — 11.3 Test Organization](https://doc.rust-lang.org/book/ch11-03-test-organization.html) — модульные и интеграционные тесты.
- [Rust by Example — Unit testing](https://doc.rust-lang.org/rust-by-example/testing/unit_testing.html) — короткие запускаемые примеры.
- [The Cargo Book — cargo test](https://doc.rust-lang.org/cargo/commands/cargo-test.html) — все параметры команды.
- [Standard library — assert_eq!](https://doc.rust-lang.org/std/macro.assert_eq.html) — документация макроса-утверждения.

---

[🏠 Начало](../START-HERE.md) · [← 02.7 Чтение ввода и преобразование строк](07-reading-input-and-parsing.md) · [📋 Этап 02](README.md) · [02.9 Основы отладки →](09-debugging-basics.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
