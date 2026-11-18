<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 09.13 Проект: хорошо протестированная библиотека с CI](13-project-tested-library-with-ci.md) · [📋 Этап 09](README.md) · [Этап 10 — Основы безопасности →](../10-security-fundamentals/README.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/09-testing-debugging-benchmarking/14-checkpoint.md) <!-- nav -->

# Урок 09.14 — Контрольная точка

> **Этап 09 — Тестирование, отладка и бенчмарки** · Урок 14 из 14 · ⏱ около 120 минут

## Что вы узнаете

- Умеете ли вы выбрать правильный вид теста для задачи и объяснить почему.
- Умеете ли вы найти ошибку с помощью теста свойств, фаззера, трассировки стека и отладчика.
- Умеете ли вы честно измерить два решения и написать рабочий процесс CI (continuous integration, непрерывная интеграция), охраняющий проект.
- Какой урок этого этапа перечитать для каждого пробела.

## Перед началом

- Вы прошли [Урок 09.13 — Проект: хорошо протестированная библиотека с CI](13-project-tested-library-with-ci.md). Если проект не закончен, сначала закончите его: это лучшая подготовка.
- Выполняйте самопроверку **не заглядывая** в уроки. Запишите ответы на бумаге или в файле, затем откройте скрытые ответы. Практические задания можно выполнять с открытыми уроками.
- Инструменты: `cargo-fuzz` с тулчейном nightly, `rust-lldb`, `actionlint` и Docker (по желанию). Код лежит в `~/projects/rust/code/p09-checkpoint`.

## Новые слова

| Слово | Значение |
|---|---|
| **точка пересечения** (crossover point) | Размер входа, при котором одно решение перестаёт быть быстрее другого. Бенчмарк по нескольким размерам её находит. |
| **допустимый сбой** (allowed failure) | Задание CI, красный результат которого не делает красным весь конвейер (`continue-on-error`), например прогон тестов на компиляторе beta. |
| **оракул** (oracle) | Надёжный способ вычислить правильный ответ, например медленная и простая версия функции, которой тест проверяет быструю версию. |

## Вопросы для самопроверки

Сначала отвечайте на каждый вопрос своими словами.

1. Программа командной строки должна завершаться с кодом 2 и печатать сообщение в поток ошибок, когда аргумент неверен. Какой уровень пирамиды тестов может это проверить и почему не модульный тест?

<details>
<summary>Ответ</summary>

Сквозной тест: он запускает настоящую программу отдельным процессом (например, через `assert_cmd`) и видит код возврата и вывод в поток ошибок. Модульный тест вызывает функцию внутри процесса, поэтому не видит ни кода возврата процесса, ни потоков вывода. (Уроки 09.1 и 09.2.)

</details>

2. Почему в здоровом наборе много модульных тестов, меньше интеграционных и мало сквозных?

<details>
<summary>Ответ</summary>

Нижние уровни быстрее, точнее (сбой указывает на небольшой кусок кода) и менее хрупки. Верхние уровни дают больше уверенности, что части работают вместе, но они медленнее и падают по большему числу причин. Каждая проверка должна выполняться на самом низком уровне, который ещё может поймать проблему. (Урок 09.1.)

</details>

3. В чём разница между стабом, фейком и моком?

<details>
<summary>Ответ</summary>

Стаб возвращает фиксированные заранее подготовленные ответы. Фейк имеет настоящую, но упрощённую реализацию (`HashMap` в памяти вместо базы данных). Мок заранее знает, какие вызовы ожидаются, и проваливает тест, если вызовы отличаются. Предпочитайте настоящие объекты, затем стабы и фейки; моки используйте, только когда сам вызов — результат, который нужно проверить. (Урок 09.3.)

</details>

4. Назовите три приёма поиска свойства для теста и скажите, что такое «сжатие» (shrinking).

<details>
<summary>Ответ</summary>

Приёмы: круговое преобразование (декодирование после кодирования даёт исходное), инвариант (результат всегда отсортирован, длина не изменилась), оракул (сравнение с медленной и простой версией), идемпотентность (сделать дважды = сделать один раз), границы / отсутствие паники. Сжатие: когда сгенерированный вход падает, `proptest` пробует более мелкие и простые входы, которые всё ещё падают, и сообщает минимальный. (Урок 09.4.)

</details>

5. Запуск `proptest` упал и создал папку `proptest-regressions`. Что с ней делать?

<details>
<summary>Ответ</summary>

Закоммитить. В ней хранится падающее начальное значение, поэтому этот же случай пробуется первым при каждом следующем запуске (включая CI), и ошибка не вернётся незамеченной. Также вы исправляете код и можете записать сжатый вход как обычный тест на примере. (Урок 09.4.)

</details>

6. Снапшот-тест падает с разницей. Какие две правильные реакции и файлы каких типов вы коммитите?

<details>
<summary>Ответ</summary>

Прочитать разницу. Если изменение задумано, принять его (`cargo insta accept` или `cargo insta review`); если нет, исправить код. Никогда не принимайте не читая. Коммитьте файлы `.snap`; не коммитьте файлы `.snap.new` и ожидающие снапшоты. В CI `insta` не записывает файлы, и тест падает. (Урок 09.5.)

</details>

7. Почему нужно увидеть, как новый тест падает («красный»), прежде чем заставить его проходить?

<details>
<summary>Ответ</summary>

Чтобы доказать, что тест может упасть и что он падает по верной причине. Тест, ни разу не бывший красным, может ничего не проверять. (Уроки 02.8, 05.6 и 09.6.)

</details>

8. Покрытие равно 100%. Значит ли это, что тесты хорошие?

<details>
<summary>Ответ</summary>

Нет. Покрытие говорит, какие строки выполнялись, а не проверялось ли что-нибудь: тест без проверок выполняет всё и ничего не подтверждает. Мутационное тестирование находит такие слабые тесты. (Урок 09.7.)

</details>

9. Что такое *пропущенный* мутант? Всегда ли пропущенный мутант означает отсутствующий тест?

<details>
<summary>Ответ</summary>

`cargo mutants` слегка меняет код (например, `>` на `>=`) и запускает тесты; пропущенный мутант — изменение, которого не заметил ни один тест, что обычно означает отсутствующий или слабый тест (часто на границе). Некоторые мутанты *эквивалентны*: изменённый код ведёт себя так же, поэтому ни один тест заметить не может; их можно игнорировать. (Урок 09.7.)

</details>

10. Почему `cargo fuzz` требует компилятор nightly и что делать с найденным падением?

<details>
<summary>Ответ</summary>

Он использует параметры компилятора (санитайзер и инструментирование покрытия, флаги `-Z`), которые есть только в nightly: `cargo +nightly fuzz run <target>`. С падением: воспроизведите его напечатанной командой, минимизируйте (`cargo fuzz tmin`), исправьте код и добавьте обычный регрессионный тест, потому что папки `artifacts` и `corpus` не коммитятся. (Урок 09.8.)

</details>

11. Что делают в LLDB (отладчике проекта LLVM) `next`, `step` и `finish` и для чего нужна точка наблюдения?

<details>
<summary>Ответ</summary>

`next` выполняет текущую строку целиком и останавливается на следующей; `step` заходит в функцию, вызываемую на строке; `finish` выполняет, пока текущая функция не вернётся. Точка наблюдения останавливает программу, когда в ячейку памяти пишут, поэтому отвечает на вопрос «кто меняет это значение?». (Урок 09.9.)

</details>

12. Как остановить программу ровно там, где она паникует, и почему точка останова на строке в релизной сборке часто остаётся «ожидающей» (pending)?

<details>
<summary>Ответ</summary>

Поставьте точку останова на функцию `rust_panic` (`breakpoint set --name rust_panic`), запустите и выберите свой кадр через `frame select`. В релизной сборке оптимизатор может встроить или убрать код, поэтому у некоторых строк нет адреса в машинном коде; отлаживайте отладочную сборку. (Урок 09.9.)

</details>

13. Что делает `RUST_LOG=warn,my_crate=debug` и почему для функции, принимающей пароль, нужно писать `#[instrument(skip_all, ...)]`?

<details>
<summary>Ответ</summary>

Показывает `WARN` и выше для всех крейтов, но `DEBUG` и выше для `my_crate`. `#[instrument]` без `skip` записывает каждый аргумент через `Debug`, поэтому пароль попал бы в лог; `skip_all` это останавливает, а безопасные поля вы перечисляете сами. В логах никогда не должно быть секретов. (Урок 09.10.)

</details>

14. В запуске `criterion` вы видите `Change within noise threshold` вместе с `p = 0.00 < 0.05`. Что вы заключаете? И зачем нужен `iter_batched`?

<details>
<summary>Ответ</summary>

Разница статистически реальна, но меньше порога шума (2%), поэтому не сообщайте её как улучшение или регрессию. `iter_batched` готовит свежий вход для каждого вызова и не измеряет время подготовки, что нужно, когда функция изменяет свой вход (как сортировка). (Урок 09.11.)

</details>

15. Какой инструмент отвечает на каждый вопрос: (а) какая из двух программ быстрее целиком, (б) замедлил ли мой коммит эту функцию, (в) какая строка медленная?

<details>
<summary>Ответ</summary>

(а) `hyperfine`. (б) `criterion` с сохранённой базовой линией. (в) Профилировщик (например, `cargo flamegraph` или `samply`), который выборочно смотрит, куда уходит время. (Урок 09.11.)

</details>

16. Тест проходит на вашем Mac и падает в CI. Назовите три возможные причины и первый шаг для выяснения.

<details>
<summary>Ответ</summary>

Другая операционная система (пути, окончания строк), отсутствующая или другая переменная окружения, скрытая зависимость от ваших файлов либо от порядка или времени тестов, другая версия компилятора или нестабильный тест. Первый шаг: запустить ту же команду в чистом контейнере Linux (`docker run … rust:1.100`), чтобы воспроизвести сбой на своём компьютере. (Уроки 09.12 и 05.6.)

</details>

17. Зачем рабочий процесс CI задаёт `permissions: contents: read` и почему конвейер должен работать с пустым кэшем?

<details>
<summary>Ответ</summary>

Строка с правами даёт токену задания минимальные права, чтобы скомпрометированный шаг не мог изменить репозиторий. Кэш — только ускорение; он может отсутствовать или быть вытеснен, поэтому конвейер, зависящий от него, ненадёжен. (Урок 09.12.)

</details>

18. В GitLab CI какое ключевое слово играет роль `runs-on:` и какое — аналог `actions/cache`?

<details>
<summary>Ответ</summary>

`image:` (образ Docker; или `tags:` раннера) и `cache:`. (Урок 09.12.)

</details>

## Практические задания

Создайте пакет один раз: `cargo new --lib code/p09-checkpoint`, затем `cd code/p09-checkpoint`.

### Задание 1 — Найти ошибку тестом свойств

Добавьте `proptest` (`cargo add --dev proptest@1`). Поместите эту функцию в `src/lib.rs`. Она должна сливать два **отсортированных** среза в один отсортированный вектор, и в ней есть ошибка:

```rust
/// Merges two sorted slices into one sorted vector.
pub fn merge_sorted(a: &[i32], b: &[i32]) -> Vec<i32> {
    let mut result = Vec::with_capacity(a.len() + b.len());
    let (mut i, mut j) = (0, 0);
    while i < a.len() && j < b.len() {
        if a[i] < b[j] {
            result.push(a[i]);
            i += 1;
        } else if a[i] > b[j] {
            result.push(b[j]);
            j += 1;
        } else {
            result.push(a[i]);
            i += 1;
            j += 1;
        }
    }
    result.extend_from_slice(&a[i..]);
    result.extend_from_slice(&b[j..]);
    result
}
```

Напишите два свойства в `tests/merge_props.rs`: результат отсортирован, и ничего не потеряно и не добавлено (результат равен отсортированной конкатенации входов). Дайте `proptest` найти ошибку, прочитайте минимальный падающий вход и исправьте функцию.

<details>
<summary>Подсказка</summary>

Генерируйте каждый вход как отсортированный вектор: `prop::collection::vec(strategy, 0..20)` и затем `.prop_map(|mut v| { v.sort(); v })`. Если все свойства проходят, ваши числа слишком разбросаны: ошибки про *равные* элементы требуют равных элементов, поэтому генерируйте малые числа, например `-5i32..5`. Для второго свойства постройте ожидаемый вектор через `[a.clone(), b.clone()].concat()` и `sort()`.

</details>

<details>
<summary>Ответ</summary>

`tests/merge_props.rs`:

```rust
use p09_checkpoint::merge_sorted;
use proptest::prelude::*;

fn sorted_vec() -> impl Strategy<Value = Vec<i32>> {
    prop::collection::vec(-5i32..5, 0..20).prop_map(|mut v| {
        v.sort();
        v
    })
}

proptest! {
    #[test]
    fn result_is_sorted(a in sorted_vec(), b in sorted_vec()) {
        let merged = merge_sorted(&a, &b);
        prop_assert!(merged.windows(2).all(|pair| pair[0] <= pair[1]));
    }

    #[test]
    fn nothing_is_lost_or_invented(a in sorted_vec(), b in sorted_vec()) {
        let merged = merge_sorted(&a, &b);
        let mut expected = [a.clone(), b.clone()].concat();
        expected.sort();
        prop_assert_eq!(merged, expected);
    }
}
```

```bash
cargo test --test merge_props
```

```text
test nothing_is_lost_or_invented ... FAILED
test result_is_sorted ... ok
Test failed: assertion failed: `(left == right)` 
  left: `[0]`,
 right: `[0, 0]` at tests/merge_props.rs:23.
minimal failing input: a = [
    0,
], b = [
    0,
]
```

Сжатый вход — `a = [0]`, `b = [0]`: когда два элемента **равны**, ветка `else` добавляет только один из них и сдвигает оба индекса, поэтому один элемент исчезает. Исправление — считать равенство «меньше»:

```rust
        if a[i] <= b[j] {
            result.push(a[i]);
            i += 1;
        } else {
            result.push(b[j]);
            j += 1;
        }
```

Теперь `cargo test --test merge_props` показывает `2 passed`. Обратите внимание на две вещи. Первое свойство (`result_is_sorted`) само **не** нашло бы ошибку, потому что ошибочный результат всё равно отсортирован: слишком слабое свойство. А с `any::<i32>()` стратегия редко создаёт равные числа, поэтому ошибка может спрятаться: *стратегия* должна уметь добираться до случая.

</details>

### Задание 2 — Прочитать панику и увидеть значение

Добавьте эту программу как `src/bin/report.rs` и эту функцию в `src/lib.rs`:

```rust
/// The average of the values, rounded down.
pub fn average(values: &[u32]) -> u32 {
    let sum: u32 = values.iter().sum();
    sum / values.len() as u32
}
```

```rust
use p09_checkpoint::average;

fn main() {
    let morning: [u32; 3] = [4, 8, 6];
    let evening: [u32; 0] = [];
    println!("morning average: {}", average(&morning));
    println!("evening average: {}", average(&evening));
}
```

Запустите. (1) Какая строка **вашего** кода — первый кадр трассировки (используйте `RUST_BACKTRACE=1`)? (2) С помощью `rust-lldb` остановитесь на панике и покажите аргумент `values`. (3) Исправьте `average`, чтобы она не могла паниковать, и добавьте тест.

<details>
<summary>Подсказка</summary>

Для (2) поставьте точку останова на `rust_panic`, запустите, найдите через `bt 8` кадр `average`, затем `frame select N` и `frame variable`. Для (3): чему должно быть равно среднее из ничего? Пусть функция возвращает `Option<u32>`; `checked_div` возвращает `None` при нулевом делителе.

</details>

<details>
<summary>Ответ</summary>

```bash
RUST_BACKTRACE=1 cargo run --bin report
```

```text
morning average: 6

thread 'main' (5604221) panicked at src/lib.rs:26:5:
attempt to divide by zero
stack backtrace:
   0: __rustc::rust_begin_unwind
   1: core::panicking::panic_fmt
   2: core::panicking::panic_const::panic_const_div_by_zero
   3: p09_checkpoint::average
             at ./src/lib.rs:26:5
   4: report::main
             at ./src/bin/report.rs:7:37
```

(1) Кадр 3: `p09_checkpoint::average` в `src/lib.rs:26:5` (ваши номера строк могут отличаться), вызванная из `report::main`, строка 7. (2) В `rust-lldb`:

```text
(lldb) breakpoint set --name rust_panic
(lldb) run
(lldb) bt 8
...
    frame #7: ... report`p09_checkpoint::average(values=size=0) at lib.rs:26:5
(lldb) frame select 7
(lldb) frame variable
```

`values=size=0`: срез пуст, и `0` — делитель. (3) Исправление и тест:

```rust
/// The average of the values, rounded down; `None` for no values.
pub fn average(values: &[u32]) -> Option<u32> {
    let sum: u32 = values.iter().sum();
    sum.checked_div(values.len() as u32)
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn average_of_nothing_is_none() {
        assert_eq!(average(&[]), None);
        assert_eq!(average(&[4, 8, 6]), Some(6));
    }
}
```

Обновите `report.rs`, чтобы он печатал `average(&morning).unwrap()` и `average(&evening).unwrap_or(0)`: программа печатает `morning average: 6` и `evening average: 0`. Почему это работает: ошибка больше невозможна, а тип `Option` заставляет каждого вызывающего решить, что значит «нет значений». (Сама сумма может переполниться на огромных входах; надёжная версия использовала бы `u64` или `checked_add`.)

</details>

### Задание 3 — Дать фаззеру сломать функцию

Добавьте эту функцию в `src/lib.rs`:

```rust
/// Splits `key=value` into its two parts.
pub fn split_key_value(line: &str) -> (&str, &str) {
    let at = line.find('=').unwrap();
    (&line[..at], &line[at + 1..])
}
```

Создайте для неё цель фаззинга (`cargo fuzz init`), запустите на 20 секунд, сообщите падающий вход, исправьте функцию и добавьте регрессионный тест.

<details>
<summary>Подсказка</summary>

Тело цели: превратите байты в `&str` через `std::str::from_utf8(data)` (при `Err` вернитесь досрочно), затем вызовите функцию. Запуск: `cargo +nightly fuzz run fuzz_target_1 -- -max_total_time=20`. Для исправления посмотрите на `str::split_once`.

</details>

<details>
<summary>Ответ</summary>

`fuzz/fuzz_targets/fuzz_target_1.rs`:

```rust
#![no_main]

use libfuzzer_sys::fuzz_target;

fuzz_target!(|data: &[u8]| {
    if let Ok(line) = std::str::from_utf8(data) {
        let _ = p09_checkpoint::split_key_value(line);
    }
});
```

```text
thread '<unnamed>' panicked at src/lib.rs:31:29:
called `Option::unwrap()` on a `None` value
...
Failing input:

	fuzz/artifacts/fuzz_target_1/crash-da39a3ee5e6b4b0d3255bfef95601890afd80709

Output of `std::fmt::Debug`:

	[]
```

Падающий вход — **пустой**: строка без `=` заставляет `find` вернуть `None`, а `unwrap` паникует. Падает любой текст без `=`. Исправление возвращает `Option`, как `split_once`:

```rust
/// Splits `key=value` into its two parts; `None` when there is no `=`.
pub fn split_key_value(line: &str) -> Option<(&str, &str)> {
    line.split_once('=')
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn key_value_without_equals_sign() {
        assert_eq!(split_key_value(""), None);
        assert_eq!(split_key_value("a=b=c"), Some(("a", "b=c")));
    }
}
```

После исправления 10-секундный запуск (`-max_total_time=10`) заканчивается строкой `Done 16473967 runs in 11 second(s)` без `Failing input`. Регрессионный тест сохраняет случай навсегда. Почему это работает: `split_once` делит по первому `=` и возвращает `None`, когда его нет, поэтому сбой — это значение в типе, а не паника.

</details>

### Задание 4 — Честно измерить два решения

Что быстрее: `Vec::contains` или `HashSet::contains` для поиска числа, которого **нет** в коллекции? Напишите бенчмарк `criterion` (`cargo add --dev criterion@0.8`, `[[bench]]` с `harness = false`) для размеров 10, 1000 и 100 000 и ответьте: есть ли точка пересечения?

<details>
<summary>Подсказка</summary>

Используйте `benchmark_group`, цикл по размерам, `bench_with_input(BenchmarkId::new(name, size), ...)` и `black_box` для искомого числа. Стройте вектор и множество **вне** измеряемого замыкания. Ищите `size + 1`, которого в коллекции никогда нет. Запуск: `cargo bench --bench lookup -- --warm-up-time 1 --measurement-time 2`.

</details>

<details>
<summary>Ответ</summary>

`benches/lookup.rs`:

```rust
use criterion::{BenchmarkId, Criterion, criterion_group, criterion_main};
use std::collections::HashSet;
use std::hint::black_box;

fn bench_lookup(c: &mut Criterion) {
    let mut group = c.benchmark_group("lookup of a missing number");
    for size in [10u32, 1_000, 100_000] {
        let vec: Vec<u32> = (0..size).collect();
        let set: HashSet<u32> = vec.iter().copied().collect();
        let missing = size + 1;
        group.bench_with_input(BenchmarkId::new("Vec::contains", size), &vec, |b, vec| {
            b.iter(|| vec.contains(black_box(&missing)))
        });
        group.bench_with_input(BenchmarkId::new("HashSet::contains", size), &set, |b, set| {
            b.iter(|| set.contains(black_box(&missing)))
        });
    }
    group.finish();
}

criterion_group!(benches, bench_lookup);
criterion_main!(benches);
```

```text
lookup of a missing number/Vec::contains/10
                        time:   [1.8436 ns 1.8905 ns 1.9437 ns]
lookup of a missing number/HashSet::contains/10
                        time:   [4.1029 ns 4.1467 ns 4.1946 ns]
lookup of a missing number/Vec::contains/1000
                        time:   [44.127 ns 51.155 ns 59.305 ns]
lookup of a missing number/HashSet::contains/1000
                        time:   [4.6973 ns 4.8279 ns 4.9711 ns]
lookup of a missing number/Vec::contains/100000
                        time:   [3.7518 µs 3.7938 µs 3.8507 µs]
lookup of a missing number/HashSet::contains/100000
                        time:   [4.7922 ns 4.8823 ns 4.9744 ns]
```

(Ваши числа будут другими, а случай `Vec` на 1000 показывает широкий интервал: повторите на тихом компьютере.) Картина: `HashSet` занимает около 4–5 нс **при любом размере** (он вычисляет хеш и смотрит в одно место). `Vec` должен пройти по всем элементам, поэтому его время растёт с размером: 1,9 нс для 10 чисел, около 50 нс для 1000 и 3,8 мкс для 100 000. **Точка пересечения** где-то между 10 и 1000 элементами: для примерно 10 элементов `Vec` быстрее (нет хеширования), а на 1000 элементах `HashSet` уже выигрывает в десять раз. Где именно линии пересекаются, смогут сказать только дополнительные размеры (например, 20, 50, 100, 200). Честный вывод с условием: «для поиска отсутствующих значений в коллекциях от 1000 элементов используйте `HashSet`; для примерно 10 элементов `Vec` не хуже». Поиск значения, найденного в начале `Vec`, дал бы другую картину, поэтому измеряйте тот случай, который действительно есть в вашей программе.

</details>

### Задание 5 — Написать рабочий процесс CI по спецификации

Напишите `.github/workflows/ci.yml` для репозитория Rust, который удовлетворяет всем требованиям: запускается при отправках в `main` и для pull request; старый запуск той же ветки отменяется при появлении нового коммита; токен может только читать репозиторий; выполняет `cargo fmt --check`, `cargo clippy --all-targets -- -D warnings`, `cargo test --all-targets` и `cargo test --doc`; использует кэш; работает на **обоих** компиляторах, `stable` и `beta`, но сбой на `beta` не должен делать конвейер красным. Проверьте через `actionlint`.

<details>
<summary>Подсказка</summary>

Используйте `strategy: matrix: toolchain: [stable, beta]` и `${{ matrix.toolchain }}`. Настройка, разрешающая заданию падать, — `continue-on-error` на уровне задания с выражением: `${{ matrix.toolchain == 'beta' }}`. Устанавливайте через `rustup toolchain install … --profile minimal --component rustfmt,clippy` и выбирайте через `rustup default …`.

</details>

<details>
<summary>Ответ</summary>

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:

concurrency:
  group: ci-${{ github.ref }}
  cancel-in-progress: true

permissions:
  contents: read

jobs:
  test:
    name: test (${{ matrix.toolchain }})
    runs-on: ubuntu-latest
    continue-on-error: ${{ matrix.toolchain == 'beta' }}
    strategy:
      fail-fast: false
      matrix:
        toolchain: [stable, beta]
    steps:
      - uses: actions/checkout@v7
      - name: Install Rust ${{ matrix.toolchain }}
        run: |
          rustup toolchain install ${{ matrix.toolchain }} --profile minimal --component rustfmt,clippy
          rustup default ${{ matrix.toolchain }}
      - uses: Swatinem/rust-cache@v2
      - run: cargo fmt --check
      - run: cargo clippy --all-targets -- -D warnings
      - run: cargo test --all-targets
      - run: cargo test --doc
```

`actionlint` ничего не печатает (проверено, из корня репозитория Git). Почему это работает: `concurrency` с `cancel-in-progress` отменяет старые запуски; `permissions: contents: read` — минимальный токен; матрица создаёт два задания; `continue-on-error` истинно только для `beta`, поэтому красное задание `beta` показывается, но не роняет запуск; `fail-fast: false` позволяет завершиться обоим заданиям. Задание `beta` заранее предупреждает о проблеме с будущей версией компилятора, не блокируя вашу работу.

</details>

## Если что-то не получилось

| Тема | Какой урок перечитать |
|---|---|
| Пирамида тестов, что тестировать где | [09.1 Пирамида тестов и стратегия тестирования](01-the-test-pyramid.md) |
| Сквозные тесты программ | [09.2 Сквозные тесты программ командной строки](02-testing-command-line-programs.md) |
| Стабы, фейки, моки | [09.3 Тестовые двойники: фейки, стабы и моки](03-test-doubles.md) |
| Свойства, сжатие, стратегии | [09.4 Тестирование на основе свойств](04-property-based-testing.md) |
| Снапшоты и `cargo insta` | [09.5 Снапшот-тестирование](05-snapshot-testing.md) |
| Красный, зелёный, рефакторинг | [09.6 Разработка через тестирование на практике](06-tdd-in-practice.md) и [05.6 Тестирование подробно](../05-rust-intermediate/06-testing-in-depth.md) |
| Покрытие и мутанты | [09.7 Покрытие кода и мутационное тестирование](07-code-coverage-and-mutation-testing.md) |
| Фаззинг | [09.8 Фаззинг](08-fuzzing.md) |
| Отладчик, трассировка, точки наблюдения | [09.9 Отладка подробно: LLDB и RustRover](09-debugging-with-lldb-and-rustrover.md) и [02.9 Основы отладки](../02-programming-basics/09-debugging-basics.md) |
| Логи, спаны, `RUST_LOG` | [09.10 Логирование и трассировка](10-logging-and-tracing.md) |
| Бенчмарки, базовые линии, шум | [09.11 Бенчмарки подробно](11-benchmarking-in-depth.md) и [05.12 Основы бенчмарков](../05-rust-intermediate/12-benchmarking-basics.md) |
| CI, `actionlint`, GitLab CI | [09.12 Непрерывная интеграция](12-continuous-integration.md) и [01.9 Командная работа и pull request](../01-git-and-version-control/09-teamwork-pull-requests-and-conventions.md) |
| Всё вместе | [09.13 Проект: хорошо протестированная библиотека с CI](13-project-tested-library-with-ci.md) |

## Что дальше

Теперь вы умеете строить страховочную сетку вокруг кода, находить ошибку нужным инструментом, честно измерять и давать CI охранять вашу работу. **Этап 10** смотрит на программы со стороны атакующего: [Этап 10 — Основы безопасности](../10-security-fundamentals/README.md) охватывает моделирование угроз, проверку ввода и внедрение, аутентификацию, секреты, основы криптографии и проверки цепочки поставок через `cargo audit` и `cargo deny`. Конвейер, который вы построили на этом этапе, — естественное место для запуска этих проверок, а фаззер и хороший набор тестов тоже являются инструментами безопасности.

## Источники и что почитать

- [The Rust Programming Language — 11. Writing Automated Tests](https://doc.rust-lang.org/book/ch11-00-testing.html) — модульные, интеграционные и doc-тесты: основа всех заданий выше.
- [The Proptest Book](https://proptest-rs.github.io/proptest/intro.html) — стратегии и сжатие, для задания 1.
- [The Rust Fuzz Book](https://rust-fuzz.github.io/book/) — цели `cargo fuzz`, артефакты и `tmin`, для задания 3.
- [Criterion.rs User Guide](https://bheisler.github.io/criterion.rs/book/) — группы бенчмарков, `BenchmarkId` и базовые замеры, для задания 4.
- [Workflow syntax for GitHub Actions](https://docs.github.com/en/actions/writing-workflows/workflow-syntax-for-github-actions) — `strategy.matrix`, `continue-on-error` и `concurrency`, для задания 5.
- [LLDB tutorial](https://lldb.llvm.org/use/tutorial.html) — точки останова, `bt`, `frame select` и `frame variable`, для задания 2.

---

[🏠 Начало](../START-HERE.md) · [← 09.13 Проект: хорошо протестированная библиотека с CI](13-project-tested-library-with-ci.md) · [📋 Этап 09](README.md) · [Этап 10 — Основы безопасности →](../10-security-fundamentals/README.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
