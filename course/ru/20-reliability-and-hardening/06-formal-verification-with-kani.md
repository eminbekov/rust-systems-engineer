<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 20.5 Внедрение сбоев и хаос-тестирование](05-fault-injection-and-chaos-testing.md) · [📋 Этап 20](README.md) · [20.7 Безопасность цепочки поставок подробно →](07-supply-chain-security-in-depth.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/20-reliability-and-hardening/06-formal-verification-with-kani.md) <!-- nav -->

# Урок 20.6 — Основы формальной верификации с Kani

> **Этап 20 — Надёжность и защита** · Урок 6 из 11 · ⏱ около 135 минут

## Что вы узнаете

- Чем тест, который проверяет некоторые входы, отличается от доказательства, которое проверяет *все* входы.
- Как работает инструмент проверки моделей Kani, как его установить и как написать *обвязку доказательства* (proof harness) с `kani::any()`.
- Как читать контрпример и превращать его в обычный модульный тест.
- Как формулировать допущения через `kani::assume`, как проверять циклы с `#[kani::unwind]` и что значит «ограниченный».
- Где Kani стоит рядом с тестами, тестами свойств и фаззингом и как запускать его в CI (непрерывной интеграции).

## Перед началом

- Вы прошли [Урок 20.5 — Внедрение сбоев и хаос-тестирование](05-fault-injection-and-chaos-testing.md).
- Вы знаете тестирование на основе свойств из [Урока 09.4](../09-testing-debugging-benchmarking/04-property-based-testing.md) и переполнение целых чисел из [Урока 12.2 — Целые числа в памяти](../12-computer-systems/02-integers-in-memory.md).
- Около 1 ГБ (гигабайта) свободного места на диске для Kani и его собственного тулчейна.
- Создайте пакет для этого урока:

```bash
cd ~/projects/rust
cargo new --lib code/p20-kani
cd code/p20-kani
```

## Новые слова

| Слово | Значение |
|---|---|
| **формальная верификация** (formal verification) | Доказательство математическими методами, что у программы есть свойство для каждого возможного входа, вместо перебора примеров. |
| **проверка моделей** (model checking) | Вид формальной верификации: инструмент исследует *все* возможные состояния программы (или её модели) и ищет то, которое нарушает правило. |
| **Kani** | Инструмент проверки моделей для Rust, который Amazon Web Services разрабатывает как открытый проект. Проверяет паники, переполнения, неправильную работу с памятью и упавшие утверждения. |
| **CBMC** (C Bounded Model Checker, ограниченный проверщик моделей для C) | Движок проверки моделей, который Kani использует внутри. Kani переводит ваш код на Rust в форму, которую может проверить CBMC. |
| **SAT-решатель** (SAT solver, satisfiability solver, решатель задачи выполнимости) | Программа, которая решает, можно ли сделать большую логическую формулу истинной, и если да, находит значения, которые это делают. CBMC использует его, чтобы искать плохой вход. |
| **обвязка доказательства** (proof harness) | Функция с пометкой `#[kani::proof]`, которая готовит входы и говорит, что должно быть истинно. Похожа на тест, но Kani проверяет её для всех входов. |
| **символьное значение** (symbolic value) | Значение из `kani::any()`, которое обозначает сразу *каждое* возможное значение своего типа, а не одно случайное. |
| **допущение** (assumption) | Условие, заданное через `kani::assume(...)`: Kani рассматривает только входы, для которых оно истинно. |
| **контрпример** (counterexample) | Конкретные входные значения, при которых проверка не проходит. Kani может напечатать их в виде модульного теста (*concrete playback*, конкретное воспроизведение). |
| **ограниченный** (bounded) | Проверенный только до некоторого предела: циклы до заданного числа повторений, массивы до заданной длины. Ограниченное доказательство ничего не говорит о входах побольше. |
| **развёртка** (unwinding) | Превращение цикла в фиксированное число копий его тела, чтобы проверщик мог о нём рассуждать. `#[kani::unwind(n)]` задаёт это число. |

## 1. Тесты проверяют примеры; доказательства проверяют всё

Модульный тест проверяет один вход. Тест свойств (Урок 09.4) проверяет несколько сотен случайных входов. Фаззер (Уроки 09.8 и 20.3) — миллионы. Всё это **примеры**. У функции от двух значений `u32` 2⁶⁴ возможных входов, примерно 18 миллиардов миллиардов. Даже при миллиарде тестов в секунду перебор всех занял бы больше 500 лет.

**Формальная верификация** отвечает на вопрос иначе: она *рассуждает* обо всех входах сразу. **Kani** делает это через **проверку моделей**:

```text
your Rust code          Kani            CBMC                    SAT solver
+ a proof harness  ──►  translates ──►  builds one huge   ──►  searches for values
                        to a model      logical formula:        that make the formula
                                        "some check fails"      true
                                                                 │
                         VERIFICATION: SUCCESSFUL ◄── no values exist
                         VERIFICATION: FAILED     ◄── found them: a counterexample
```

Если решатель доказывает, что никакой вход не может провалить проверку, свойство выполняется для *каждого* входа: это доказательство, а не тест. Kani автоматически добавляет проверки на всё, что может вызвать панику или неопределённое поведение: арифметическое переполнение, деление на ноль, индексацию за границами, `unwrap` от `None` и неправильные указатели в `unsafe`-коде. Ваши собственные вызовы `assert!` тоже проверяются.

Цена: это работает на небольших кусках кода, может быть медленным, а циклы должны быть **ограниченными** (раздел 4).

## 2. Установите Kani и напишите первое доказательство

Kani устанавливается в два шага (по официальному руководству по установке): небольшой установщик с crates.io, затем настоящие инструменты, которые `cargo kani setup` скачивает в `~/.kani` (задайте переменную окружения `KANI_HOME`, чтобы использовать другую папку):

```bash
cargo install --locked kani-verifier
cargo kani setup
cargo kani --version
```

```text
   Installed package `kani-verifier v0.68.0` (executables `cargo-kani`, `kani`)
[0/5] Running Kani first-time setup...
...
[2/5] Downloading Kani release bundle: kani-0.68.0-aarch64-apple-darwin.tar.gz
[3/5] Installing rust toolchain version: nightly-2026-08-21-aarch64-apple-darwin
...
[5/5] Successfully completed Kani first-time setup.
Kani Rust Verifier 0.68.0 (cargo plugin)
CBMC 6.11.0
```

(Сокращено. Kani приносит собственный nightly-тулчейн с фиксированной датой; ваши тулчейны менять не нужно.)

Во время компиляции Kani задаёт условие `cfg(kani)`, поэтому доказательства живут в модуле, который обычные сборки игнорируют. Сообщите компилятору, что `kani` — известное имя условия, как вы делали для `loom` в Уроке 14.9. Добавьте в `Cargo.toml`:

```toml
[lints.rust]
unexpected_cfgs = { level = "warn", check-cfg = ["cfg(kani)"] }
```

Теперь код. Замените `src/lib.rs`:

```rust
//! Small functions with proofs checked by Kani.

/// The point halfway between `a` and `b`, rounded down.
pub fn midpoint(a: u32, b: u32) -> u32 {
    (a + b) / 2
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn midpoint_of_small_numbers() {
        assert_eq!(midpoint(2, 8), 5);
        assert_eq!(midpoint(3, 4), 3);
    }
}

#[cfg(kani)]
mod proofs {
    use super::*;

    #[kani::proof]
    fn midpoint_is_between_the_two_values() {
        let a: u32 = kani::any();
        let b: u32 = kani::any();
        let m = midpoint(a, b);
        assert!(a.min(b) <= m && m <= a.max(b));
    }
}
```

**Что происходит:**

1. `midpoint` — классический способ найти середину между двумя числами, используемый, например, в двоичном поиске (Урок 07.10). Модульный тест проходит.
2. `#[cfg(kani)]` компилирует модуль `proofs`, только когда работает Kani. Крейт `kani` там доступен автоматически; добавлять его в `Cargo.toml` не нужно.
3. `#[kani::proof]` помечает **обвязку доказательства**. `kani::any()` даёт **символьное значение**: не один случайный `u32`, а «вообще любой `u32`».
4. Утверждение формулирует свойство: середина лежит между двумя значениями. Kani также проверяет каждую арифметическую операцию на переполнение.

```bash
cargo test -q
cargo kani --output-format terse
```

```text
test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
Kani Rust Verifier 0.68.0 (cargo plugin)
CBMC 6.11.0
...
Checking harness proofs::midpoint_is_between_the_two_values...

VERIFICATION RESULT:
 ** 1 of 16 failed
Failed Checks: attempt to add with overflow
 File: "src/lib.rs", line 5, in midpoint

VERIFICATION:- FAILED
Verification Time: 0.029502625s

Manual Harness Summary:
Verification failed for - proofs::midpoint_is_between_the_two_values
Complete - 0 successfully verified harnesses, 1 failures, 1 total.
```

**Как это читать:**

1. `--output-format terse` показывает только проваленные проверки. Без него Kani перечисляет все 16 сделанных проверок, большинство из них внутри стандартной библиотеки. (Kani также печатает `warning` про `#![feature(register_tool)]` из собственной настройки; его можно игнорировать.)
2. `attempt to add with overflow` в строке 5: `a + b` может оказаться больше `u32::MAX`. Модульный тест с 2 и 8 никогда бы этого не показал.
3. Весь поиск по всем 2⁶⁴ парам входов занял 0,03 секунды.

Какие входы? Попросите **контрпример** в виде готового теста через *concrete playback* (это пока нестабильный параметр, отсюда `-Z`):

```bash
cargo kani -Z concrete-playback --concrete-playback=print --output-format terse
```

```text
Concrete playback unit test for `proofs::midpoint_is_between_the_two_values`:
```
```rust
/// Test generated for harness `proofs::midpoint_is_between_the_two_values` 
///
/// Check for `assertion`: "attempt to add with overflow"

#[test]
fn kani_concrete_playback_midpoint_is_between_the_two_values_5132311777728133568() {
    let concrete_vals: Vec<Vec<u8>> = vec![
        // 2147483648
        vec![0, 0, 0, 128],
        // 2147483648
        vec![0, 0, 0, 128],
    ];
    kani::concrete_playback_run(concrete_vals, midpoint_is_between_the_two_values);
}
```

Комментарии показывают значения: `a = 2147483648` и `b = 2147483648` (по 2³¹, так что сумма равна 2³², на единицу больше `u32::MAX`). Сгенерированный тест работает только со вспомогательной библиотекой Kani; проще скопировать *значения* в обычный модульный тест (Упражнение 3).

Исправьте `midpoint`, чтобы она никогда не складывала два больших числа:

```rust
pub fn midpoint(a: u32, b: u32) -> u32 {
    let (low, high) = (a.min(b), a.max(b));
    low + (high - low) / 2
}
```

```bash
cargo kani --output-format terse
```

```text
Checking harness proofs::midpoint_is_between_the_two_values...

VERIFICATION RESULT:
 ** 0 of 17 failed

VERIFICATION:- SUCCESSFUL
Verification Time: 0.051254917s

Manual Harness Summary:
Complete - 1 successfully verified harnesses, 0 failures, 1 total.
```

`high - low` не может уйти ниже нуля, потому что `high >= low`, а `low + (high - low) / 2` не больше `high`. Kani подтверждает это для каждой пары значений `u32`. (В стандартной библиотеке такая функция тоже есть: `a.midpoint(b)`.)

> 🧪 **Тестирование:** обвязка доказательства — это тест с гораздо более сильным обещанием. Пишите её как тест свойства: «для всех входов это свойство выполняется». Многие тесты свойств из Урока 09.4 можно превратить в доказательства Kani, заменив сгенерированные значения на `kani::any()`.

## 3. Допущения: что могут представлять собой входы

У большинства функций есть правила для входов. Добавьте это в `src/lib.rs` (над модулем `tests`), а в модуль `proofs` — вторую обвязку:

```rust
/// What percentage of `total` is `part`, rounded down.
pub fn percent(part: u32, total: u32) -> u32 {
    part * 100 / total
}
```

```rust
    #[kani::proof]
    fn percent_is_at_most_100() {
        let part: u32 = kani::any();
        let total: u32 = kani::any();
        let p = percent(part, total);
        assert!(p <= 100);
    }
```

`--harness` запускает только обвязки, в имени которых есть заданный текст:

```bash
cargo kani --output-format terse --harness percent_is_at_most_100
```

```text
Checking harness proofs::percent_is_at_most_100...

VERIFICATION RESULT:
 ** 3 of 5 failed
Failed Checks: assertion failed: p <= 100
 File: "src/lib.rs", line 53, in proofs::percent_is_at_most_100
Failed Checks: attempt to multiply with overflow
 File: "src/lib.rs", line 11, in percent
Failed Checks: attempt to divide by zero
 File: "src/lib.rs", line 11, in percent

VERIFICATION:- FAILED
```

(Номера строк зависят от того, куда вы поместили код.) Сразу три проблемы. Две из них — про *неправильное использование*: `total = 0` и `part` больше `total` (тогда результат больше 100 %). Это правила вызывающего, и функция должна их документировать. **Допущения** говорят Kani рассматривать только входы, которые следуют правилам. Добавьте в обвязку одну строку перед вызовом:

```rust
        kani::assume(total > 0 && part <= total);
```

```text
Checking harness proofs::percent_is_at_most_100...

VERIFICATION RESULT:
 ** 1 of 5 failed
Failed Checks: attempt to multiply with overflow
 File: "src/lib.rs", line 11, in percent

VERIFICATION:- FAILED
```

Переполнение остаётся, и это настоящий баг: для любого `part` больше 42 949 672 значение `part * 100` не помещается в `u32`, даже при совершенно правильных входах вроде `percent(50_000_000, 100_000_000)`. Упражнение 2 просит его исправить.

> ⚠️ **Частая ошибка:** слишком сильное допущение ничего не доказывает. `kani::assume(total > 0 && part <= total && part < 1000)` заставило бы обвязку пройти и спрятало бы переполнение. Допускайте только то, что документация функции требует от вызывающих, и обеспечивайте эти правила в коде (проверкой, типом вроде `NonZeroU32`), где это возможно.

## 4. Циклы и развёртка

Проверщик моделей не может выполнять цикл «пока он не остановится»; ему нужно знать, сколько повторений рассматривать. Добавьте функцию с циклом и обвязку, которая проверяет её для **каждого среза длиной до 4 байтов**:

```rust
/// How many bytes in `data` are zero.
pub fn count_zeros(data: &[u8]) -> usize {
    let mut count = 0;
    for &byte in data {
        if byte == 0 {
            count += 1;
        }
    }
    count
}
```

```rust
    #[kani::proof]
    #[kani::unwind(2)]
    fn count_zeros_is_at_most_the_length() {
        let data: [u8; 4] = kani::any();
        let len: usize = kani::any();
        kani::assume(len <= data.len());
        let slice = &data[..len];
        assert!(count_zeros(slice) <= slice.len());
    }
```

**Что происходит:**

1. `let data: [u8; 4] = kani::any();` создаёт четыре символьных байта: сразу любое возможное содержимое.
2. Символьная `len` от 0 до 4 заставляет `slice` обозначать каждый срез длины 0, 1, 2, 3 или 4.
3. `#[kani::unwind(2)]` говорит «рассматривать не больше 2 повторений каждого цикла». Это намеренно слишком мало.

```bash
cargo kani --output-format terse --harness count_zeros
```

```text
Checking harness proofs::count_zeros_is_at_most_the_length...

VERIFICATION RESULT:
 ** 1 of 79 failed (78 undetermined)
Failed Checks: unwinding assertion loop 0
 File: "src/lib.rs", line 17, in count_zeros

VERIFICATION:- FAILED
[Kani] info: Verification output shows one or more unwinding failures.
[Kani] tip: Consider increasing the unwinding value or disabling `--unwinding-assertions`.
```

Kani не притворяется: он сообщает, что цикл может выполниться больше 2 раз, поэтому остальное проверить не удалось (`78 undetermined`, 78 не определено). Граница должна быть на единицу больше наибольшего числа повторений (одна дополнительная проверка, что цикл действительно заканчивается). Измените её на `#[kani::unwind(5)]`:

```text
Checking harness proofs::count_zeros_is_at_most_the_length...

VERIFICATION RESULT:
 ** 0 of 79 failed (1 unreachable)

VERIFICATION:- SUCCESSFUL
Verification Time: 0.20878696s
```

Теперь свойство **доказано для всех срезов длиной до 4 байтов**, и не больше. Именно это значит **ограниченный**. На практике это всё равно очень ценно: большинство багов в циклах проявляется на крошечных размерах (пусто, один элемент, точка заворачивания), и Kani проверяет их все полностью. `unreachable` означает, что одна проверка стоит на пути, который никогда не выполняется; здесь это нормально.

> ⚡ **Производительность:** время, которое нужно Kani, быстро растёт с границей и с числом символьных байтов. Начинайте с маленьких границ (2–8) и увеличивайте их, только пока прогон остаётся быстрым. Обвязку, которая работает минуты, стоит разбить на меньшие.

## 5. Место Kani

| Приём | Какие входы проверяются | Хорош для | Слаб в |
|---|---|---|---|
| Модульные тесты | примеры, которые вы пишете | известных случаев, документации | случаях, о которых вы не подумали |
| Тесты свойств (09.4) | сотни случайных входов | быстрых широких проверок при каждой сборке | редких граничных значениях |
| Фаззинг (09.8, 20.3) | миллионы входов, направляемых покрытием | парсеров, большого кода, долгих прогонов | доказательстве отсутствия багов |
| **Kani** | **все** входы, в пределах границ | маленьких критичных функций: арифметики, вычисления индексов, `unsafe`-блоков, парсеров заголовков фиксированного размера | больших программах, глубоких циклах, потоках и async-коде, коде, который вызывает операционную систему |

Используйте Kani для маленьких кусков, где баг обходится дорого, а пространство входов огромно, но код короткий: вычисления размеров, арифметика индексов в кольцевых буферах (Упражнение 4), работа с битами и условия безопасности `unsafe`-блоков (Урок 10.12). Большие проекты используют его именно так: у стандартной библиотеки Rust и у монитора виртуальных машин Firecracker от Amazon есть доказательства Kani для таких кусков.

Kani запускается в CI как любая другая проверка. Сохраните как `.github/workflows/kani.yml`:

```yaml
name: Kani

on:
  pull_request:
  push:
    branches: [main]

permissions:
  contents: read

jobs:
  proofs:
    runs-on: ubuntu-latest
    timeout-minutes: 60
    steps:
      - uses: actions/checkout@v7
      - name: Install Kani
        run: |
          cargo install --locked kani-verifier
          cargo kani setup
      - name: Run all proof harnesses
        run: cargo kani --output-format terse
```

```bash
actionlint .github/workflows/kani.yml && echo "actionlint: no problems"
```

```text
actionlint: no problems
```

(Команды те же, что и на вашем Mac; сам workflow выполняется на машинах GitHub с Linux, которые этот курс использовать не может. Проект Kani также публикует готовый action, `model-checking/kani-github-action`.) `cargo kani` завершается с ошибкой, когда проваливается любая обвязка, поэтому падает и задание.

> 🔒 **Безопасность:** целочисленное переполнение в вычислениях размеров и индексов — одна из самых частых причин ошибок памяти в `unsafe`-коде. Обвязка Kani над арифметикой, которая вычисляет размер буфера или смещение, даёт гарантию, которую не даст никакое количество тестов.

## Частые ошибки

**1. Забыть `#[cfg(kani)]`.** Без него `cargo build` пытается скомпилировать `kani::any()` и падает с ``error[E0433]: cannot find module or crate `kani` in this scope``. Держите доказательства в модуле с `#[cfg(kani)]`.

**2. Допускать слишком много.** Каждый `kani::assume` убирает входы из доказательства. Сверяйте каждое допущение с документированным контрактом функции (раздел 3).

**3. Читать «SUCCESSFUL» как «правильно».** Kani доказывает ровно те проверки, которые сделал: нет паник, нет переполнений и выполняются *ваши* утверждения. Если обвязка ничего не утверждает о результате, неверный результат всё равно пройдёт. Пишите утверждения, которые описывают, что функция обязана делать.

**4. Забывать, что доказательства ограничены.** `#[kani::unwind(5)]` с массивом из 4 байтов ничего не доказывает о срезе из 100 байтов. Записывайте границу в имя обвязки или в комментарий.

## Попробуйте сами

1. Установите Kani двумя командами из раздела 2 и проверьте `cargo kani --version`.
2. Создайте `code/p20-kani` с переполняющейся `midpoint`, её модульным тестом и доказательством. Запустите `cargo test`, затем `cargo kani --output-format terse`.
3. Напечатайте контрпример через concrete playback, затем исправьте `midpoint` и снова запустите Kani.
4. Добавьте `percent` и её обвязку. Запустите без допущения и с ним.
5. Добавьте `count_zeros` с `#[kani::unwind(2)]`, прочитайте сообщение о развёртке и измените границу на 5.

## Упражнения

### Упражнение 1 — Три провала, три входа (легко)

Для каждой из трёх проваленных проверок `percent_is_at_most_100` (раздел 3, до допущения) приведите одну конкретную пару `(part, total)`, которая её вызывает.

<details>
<summary>Подсказка</summary>

Делению на ноль нужно одно особое значение `total`. Для умножения найдите наименьшее `part`, при котором `part * 100 > u32::MAX`.

</details>

<details>
<summary>Ответ</summary>

- **Деление на ноль:** `(1, 0)`: любое `part` при `total = 0`.
- **Переполнение при умножении:** `(42_949_673, 100_000_000)`: `u32::MAX` равно 4 294 967 295, поэтому каждое `part` начиная с 42 949 673 переполняется при умножении на 100.
- **Утверждение `p <= 100`:** `(5, 1)`: `part` больше `total` даёт 500 %.

</details>

### Упражнение 2 — Исправьте `percent` и докажите (средне)

Измените `percent` так, чтобы она не могла переполниться ни на одном входе, который следует правилам `total > 0` и `part <= total`, и чтобы обвязка с допущением проходила.

<details>
<summary>Подсказка</summary>

Выполняйте умножение в более широком типе: `u64::from(part) * 100` всегда помещается в `u64`.

</details>

<details>
<summary>Ответ</summary>

```rust
/// What percentage of `total` is `part`, rounded down.
/// The caller must make sure that `total > 0` and `part <= total`.
pub fn percent(part: u32, total: u32) -> u32 {
    let wide = u64::from(part) * 100 / u64::from(total);
    wide as u32 // at most 100, because part <= total
}
```

```text
Checking harness proofs::percent_is_at_most_100...
...
VERIFICATION:- SUCCESSFUL
```

Почему это работает: `u32::MAX * 100` — около 4,3 × 10¹¹, далеко ниже `u64::MAX`, поэтому умножение не может переполниться. При `part <= total` частное не больше 100, поэтому `as u32` ничего не теряет, и Kani доказывает утверждение `p <= 100` для каждой разрешённой пары.

</details>

### Упражнение 3 — От контрпримера к регрессионному тесту (средне)

Напишите обычный модульный тест (он должен запускаться через `cargo test`, без Kani), который поймал бы исходный баг `midpoint`, используя значения из concrete playback. Добавьте второе утверждение с `u32::MAX`.

<details>
<summary>Подсказка</summary>

Значения воспроизведения были `2147483648` и `2147483648`. Какова середина между двумя одинаковыми числами?

</details>

<details>
<summary>Ответ</summary>

```rust
    #[test]
    fn midpoint_of_huge_numbers_found_by_kani() {
        assert_eq!(midpoint(2_147_483_648, 2_147_483_648), 2_147_483_648);
        assert_eq!(midpoint(u32::MAX, u32::MAX - 2), u32::MAX - 1);
    }
```

```text
test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

Почему это работает: со старой `(a + b) / 2` обе строки в отладочной сборке паникуют с `attempt to add with overflow`. Тест сохраняет находку навсегда, в том числе на машинах, где Kani не установлен, так же как находки фаззера становятся регрессионными тестами.

</details>

### Упражнение 4 — Докажите шаг кольцевого буфера (сложно)

Кольцевые буферы (Уроки 07.5 и 20.3) сдвигают индекс вперёд через `(index + 1) % capacity`. Напишите `pub fn next_index(index: usize, capacity: usize) -> usize` и обвязку, которая доказывает для каждой `capacity > 0` и каждого `index < capacity`: результат меньше `capacity`; последний индекс заворачивается в 0; каждый другой индекс сдвигается вперёд ровно на один.

<details>
<summary>Подсказка</summary>

Используйте два символьных значения `usize`, один `kani::assume` и `if index == capacity - 1` с `assert_eq!` в обеих ветках.

</details>

<details>
<summary>Ответ</summary>

```rust
/// The slot after `index` in a ring buffer with `capacity` slots.
pub fn next_index(index: usize, capacity: usize) -> usize {
    (index + 1) % capacity
}
```

```rust
    #[kani::proof]
    fn next_index_stays_inside_and_wraps() {
        let capacity: usize = kani::any();
        let index: usize = kani::any();
        kani::assume(capacity > 0 && index < capacity);
        let next = next_index(index, capacity);
        assert!(next < capacity);
        if index == capacity - 1 {
            assert_eq!(next, 0);
        } else {
            assert_eq!(next, index + 1);
        }
    }
```

```text
Checking harness proofs::next_index_stays_inside_and_wraps...

VERIFICATION RESULT:
 ** 0 of 9 failed

VERIFICATION:- SUCCESSFUL
Verification Time: 0.9576767s
```

Почему это работает: Kani проверяет все 2¹²⁸ пар значений `usize`, удовлетворяющих допущению, включая `index = usize::MAX - 1` при `capacity = usize::MAX`, где неосторожное `index + 1` могло бы переполниться. Поскольку `index < capacity <= usize::MAX`, `index + 1` никогда не переполняется, и Kani это подтверждает.

</details>

## Вопросы для самопроверки

1. Чем тест свойства отличается от обвязки доказательства Kani?

<details>
<summary>Ответ</summary>

Тест свойства запускает код со множеством случайных входов-примеров. Обвязка Kani использует символьные значения (`kani::any()`), и Kani проверяет свойство для каждого возможного входа (в пределах границ), поэтому успех — это доказательство.

</details>

2. Какие проверки Kani добавляет автоматически, без всякого `assert!` с вашей стороны?

<details>
<summary>Ответ</summary>

Проверки на всё, что может вызвать панику или неопределённое поведение: арифметическое переполнение, деление на ноль, индексацию за границами, `unwrap` от `None` или `Err` и неправильное использование указателей в `unsafe`-коде.

</details>

3. Что делает `kani::assume(x > 0)` и в чём его опасность?

<details>
<summary>Ответ</summary>

Оно ограничивает доказательство входами, где `x > 0`. Если допущение сильнее настоящего контракта функции, оно прячет баги: исключённые входы никогда не проверяются.

</details>

4. Kani сообщает `unwinding assertion loop 0`. Что это значит?

<details>
<summary>Ответ</summary>

Цикл может выполниться больше раз, чем позволяет граница `unwind`, поэтому Kani не смог проверить всё. Увеличьте `#[kani::unwind(n)]` (до числа на единицу больше наибольшего числа повторений) или уменьшите входы.

</details>

5. Обвязка над `[u8; 4]` с `unwind(5)` проходит. Что именно доказано?

<details>
<summary>Ответ</summary>

Что свойство выполняется для каждого возможного содержимого входов до этого размера (здесь — для каждого среза длиной до 4 байтов). Для более длинных входов ничего не доказано: доказательство ограниченное.

</details>

6. Для какого кода Kani лучше фаззинга?

<details>
<summary>Ответ</summary>

Для маленьких критичных функций с огромным пространством входов: арифметики размеров и индексов, работы с битами, условий безопасности `unsafe`-блоков. Фаззинг лучше для большого кода, например целых парсеров, и для долгих прогонов на реалистичных входах.

</details>

## Итоги

- Тесты, тесты свойств и фаззеры проверяют примеры; **формальная верификация** проверяет все входы. Kani делает это через проверку моделей: CBMC и SAT-решатель ищут любой вход, который проваливает проверку.
- **Обвязка доказательства** — функция с `#[kani::proof]` в модуле с `#[cfg(kani)]`; `kani::any()` даёт символьные значения. Kani автоматически проверяет переполнения, паники и безопасность памяти, а также ваши утверждения.
- Провал приходит с **контрпримером**; concrete playback печатает значения, и они становятся обычным регрессионным тестом.
- `kani::assume` формулирует правила для входов; допускайте только то, что требует контракт.
- Циклам нужен `#[kani::unwind(n)]`, а доказательства **ограничены**: они верны до выбранных вами размеров.
- Используйте Kani для маленького критичного кода (размеры, индексы, `unsafe`) и запускайте `cargo kani` в CI.

## Источники и что почитать

- [The Kani Rust Verifier](https://model-checking.github.io/kani/) — официальная книга: [установка](https://model-checking.github.io/kani/install-guide.html), [первые шаги](https://model-checking.github.io/kani/tutorial-first-steps.html) и [развёртка циклов](https://model-checking.github.io/kani/tutorial-loop-unwinding.html).
- [Kani — attributes](https://model-checking.github.io/kani/reference/attributes.html) — `#[kani::proof]`, `#[kani::unwind]` и другие атрибуты.
- [Kani — concrete playback](https://model-checking.github.io/kani/reference/experimental/concrete-playback.html) — как контрпримеры становятся модульными тестами.
- [Kani in GitHub CI](https://model-checking.github.io/kani/install-github-ci.html) — официальный GitHub Action и его параметры.
- [CBMC](https://github.com/diffblue/cbmc) — ограниченный проверщик моделей, работающий внутри.

---

[🏠 Начало](../START-HERE.md) · [← 20.5 Внедрение сбоев и хаос-тестирование](05-fault-injection-and-chaos-testing.md) · [📋 Этап 20](README.md) · [20.7 Безопасность цепочки поставок подробно →](07-supply-chain-security-in-depth.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
