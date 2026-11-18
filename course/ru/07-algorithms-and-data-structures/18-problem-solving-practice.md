<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 07.17 Жадные алгоритмы](17-greedy-algorithms.md) · [📋 Этап 07](README.md) · [07.19 Проект: поиск маршрутов →](19-project-route-finder.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/07-algorithms-and-data-structures/18-problem-solving-practice.md) <!-- nav -->

# Урок 07.18 — Практика решения задач

> **Этап 07 — Алгоритмы и структуры данных** · Урок 18 из 20 · ⏱ около 135 минут

## Что вы узнаете

- Повторяемый метод решения алгоритмической задачи: понять, маленькие примеры, перебор, найти лишнюю работу, лучший алгоритм, тесты, бенчмарк.
- Как по размеру входа выбрать сложность, которую можно себе позволить.
- Как превратить медленное решение в быстрое, отыскав повторяющуюся работу.
- Пять задач в духе собеседований на стеки, скользящие окна, кучи, графы и динамическое программирование, каждая с тестами и указанной сложностью.

## Перед началом

- Вы прошли [Урок 07.17 — Жадные алгоритмы](17-greedy-algorithms.md) и уроки до него: здесь используются стеки и очереди (07.5), хеш-таблицы (07.6), BFS (Breadth-First Search, поиск в ширину, 07.13), кучи (07.12), скользящие окна (07.15) и динамическое программирование (07.16).
- Вы работаете в `code/p07-algorithms`. Этот урок добавляет модуль `src/practice.rs` (и `pub mod practice;` в `src/lib.rs`) и один бенчмарк.
- Решайте каждую задачу сами **до** того, как откроете подсказки. Подсказки идут по шагам: откройте одну, попробуйте снова, откройте следующую. Цель этого урока — привычка, а не ответы.

## Новые слова

| Слово | Значение |
|---|---|
| **задача в духе собеседования** (interview-style problem) | Короткий самодостаточный вопрос («найдите …», «посчитайте …») с ясными входом и выходом, из тех, что задают на собеседованиях. Проверяет мышление, а не знание библиотеки. |
| **перебор** (brute force) | Простейшее верное решение: проверить все возможности. Оно медленное, но его легко сделать верным, поэтому это лучший судья для вашего более быстрого решения. |
| **судья** (judge) | В этом курсе: медленная, заведомо верная функция (или вызов стандартной библиотеки), по которой тесты проверяют быструю функцию на многих входах. |
| **ограничение** (constraint) | Предел на вход, данный вместе с задачей, например «не более 100 000 чисел». Он говорит, какая сложность достаточно быстра. |
| **узкое место** (bottleneck) | Часть алгоритма, которая стоит больше всего времени. Ускорять нужно сначала её. |
| **компромисс** (trade-off) | Получение одного ценой другого, например скорость за память: хеш-таблица делает поиск быстрым, но использует O(n) дополнительной памяти. |

## 1. Метод

Новичок читает задачу, начинает печатать, застревает и начинает заново. Метод ниже заменяет это семью маленькими шагами. У каждого шага один вопрос.

| Шаг | Вопрос | Результат |
|---|---|---|
| 1. Понять | Каковы вход и выход? Каковы пределы? Что если вход пуст? | Задача, пересказанная своими словами, и ограничения. |
| 2. Маленькие примеры | Каковы ответы для трёх крошечных входов, включая граничный случай? | Несколько строк `assert_eq!`, написанных до кода. |
| 3. Перебор | Каково простейшее верное решение, пусть и медленное? | Работающая функция и её сложность. |
| 4. Найти лишнюю работу | Что перебор вычисляет снова и снова или ищет медленно? | Предложение: «он повторяет …» или «он ищет … линейно». |
| 5. Лучший алгоритм | Какой инструмент убирает лишнюю работу? Сортировка, хеш-таблица, два указателя, куча, таблица? | Более быстрая функция и её сложность. |
| 6. Тесты | Совпадает ли быстрая функция с перебором на тысячах маленьких случайных входов? | Тест с судьёй и граничные случаи из шага 2. |
| 7. Бенчмарк | Во сколько раз быстрее на реалистичном размере? | Два числа от `criterion` (только если скорость важна). |

**Читайте пределы (шаг 1).** Размер входа `n` говорит, какая сложность достаточно быстра. Компьютер делает примерно от 10⁸ до 10⁹ простых шагов в секунду, поэтому:

| Размер входа `n` | Допустимая сложность | Типичные инструменты |
|---|---|---|
| до примерно 20 | O(2ⁿ), O(n!) | перебор всех подмножеств или порядков (перебор) |
| от примерно 1000 до 5000 | O(n²) | вложенные циклы, двумерная таблица |
| от примерно 100 000 до 1 000 000 | O(n log n) | сортировка, кучи, двоичный поиск |
| больше 1 000 000 | O(n) или O(log n) | один проход, хеш-таблица, два указателя |

> 💡 **Совет:** если вы застряли на шаге 5, вернитесь к шагу 3 и снова посмотрите на перебор. Лучший алгоритм почти всегда прячется в его самой медленной строке.

## 2. Разобранный пример: два числа с заданной суммой

**1. Понять.** Срез чисел `i32` (не отсортированный) и `target`. Верните две *разные* позиции `i < j` с `values[i] + values[j] == target` или `None`. Предел: до 100 000 чисел. (В Уроке 07.15 срез был отсортирован; теперь нет, поэтому два указателя не подходят.)

**2. Маленькие примеры.** `[2, 7, 11, 15]` с целью 9 даёт позиции `(0, 1)`. `[3, 3]` с 6 даёт `(0, 1)`: одно и то же *значение* дважды допустимо, одну и ту же *позицию* дважды — нет. `[1, 2, 3]` с 100 даёт `None`. Пустой срез даёт `None`.

**3. Перебор.** Проверяем каждую пару:

```rust
/// Two different positions `i < j` with `values[i] + values[j] == target`.
/// Brute force: try every pair. Time O(n * n), space O(1).
pub fn two_sum_slow(values: &[i32], target: i32) -> Option<(usize, usize)> {
    for i in 0..values.len() {
        for j in (i + 1)..values.len() {
            if i64::from(values[i]) + i64::from(values[j]) == i64::from(target) {
                return Some((i, j));
            }
        }
    }
    None
}
```

**Что происходит:** два вложенных цикла обходят около n²/2 пар: **время O(n²)**, память O(1). Сумма считается в `i64`, чтобы не могло быть переполнения. Для 100 000 чисел это около пяти миллиардов шагов: слишком медленно.

**4. Найти лишнюю работу.** Для каждого числа `values[j]` внутренний цикл *ищет в остатке среза* партнёра `target - values[j]`. Поиск в срезе стоит O(n). Хеш-таблица отвечает на вопрос «видел ли я это значение и где?» за O(1) в среднем.

```text
values = [2, 7, 11, 15], target = 9

j=0  value 2  partner 7   seen = {}          -> not found; remember 2 -> 0
j=1  value 7  partner 2   seen = {2: 0}      -> found at 0  => answer (0, 1)
```

**5. Лучший алгоритм.** Пройдите срез один раз. Для каждого значения спросите у таблицы его партнёра; если он там, всё готово, иначе запомните текущее значение и его позицию.

```rust
/// The same, with a hash map from value to position.
/// Time O(n) on average, space O(n).
pub fn two_sum(values: &[i32], target: i32) -> Option<(usize, usize)> {
    let mut seen: HashMap<i32, usize> = HashMap::new();
    for (j, &value) in values.iter().enumerate() {
        // the partner we are looking for (None if it does not fit into i32)
        let partner = i32::try_from(i64::from(target) - i64::from(value)).ok();
        if let Some(&i) = partner.and_then(|p| seen.get(&p)) {
            return Some((i, j));
        }
        seen.insert(value, j);
    }
    None
}
```

**Что происходит:**

1. `seen` отображает значение в позицию, где мы его видели. В начале шага `j` в ней только позиции меньше `j`, поэтому в каждом ответе `i < j`.
2. `i32::try_from(...)` превращает партнёра типа `i64` обратно в `i32`; если он не помещается, получается `Err` (а `.ok()` даёт `None`), и тогда никакое число среза им быть не может.
3. `partner.and_then(|p| seen.get(&p))` ищет партнёра; образец `Some(&i)` копирует позицию из ссылки.
4. Поиск выполняется *до* вставки, поэтому число не может образовать пару само с собой. Для `[3, 3]` первая тройка сохраняется, а вторая её находит.
5. Один проход с O(1) операций таблицы в среднем: **время O(n) в среднем**, **память O(n)** на таблицу. Это компромисс: время оплачено памятью ([Урок 07.6](06-hash-tables.md) объясняет, почему поиск в хеш-таблице — O(1) в среднем и O(n) в худшем случае).

**6. Тесты.** Добавьте модуль `tests`. Он использует генератор `Rng` из `src/testutil.rs` с методом `range` из [Урока 07.15](15-two-pointers-and-sliding-window.md) (`range` работает с `i64`) и добавляет небольшую вспомогательную функцию `random_text`, которая составляет случайный текст из букв алфавита; обе нужны в следующих задачах:

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use crate::testutil::Rng;
    use std::collections::HashSet;

    /// A random text of `len` characters taken from `alphabet`.
    fn random_text(rng: &mut Rng, alphabet: &str, len: i64) -> String {
        let letters: Vec<char> = alphabet.chars().collect();
        (0..len)
            .map(|_| letters[rng.range(0, letters.len() as i64 - 1) as usize])
            .collect()
    }

    #[test]
    fn two_sum_examples() {
        assert_eq!(two_sum(&[2, 7, 11, 15], 9), Some((0, 1)));
        assert_eq!(two_sum(&[3, 3], 6), Some((0, 1)));
        assert_eq!(two_sum(&[1, 2, 3], 100), None);
        assert_eq!(two_sum(&[], 0), None);
        assert_eq!(two_sum(&[i32::MAX, i32::MIN], -1), Some((0, 1)));
    }

    #[test]
    fn two_sum_agrees_with_brute_force() {
        let mut rng = Rng::new(1);
        for _ in 0..2000 {
            let len = rng.range(0, 10);
            let v: Vec<i32> = (0..len).map(|_| rng.range(-10, 10) as i32).collect();
            let target = rng.range(-20, 20) as i32;
            let fast = two_sum(&v, target);
            assert_eq!(fast.is_some(), two_sum_slow(&v, target).is_some(), "{v:?}");
            if let Some((i, j)) = fast {
                assert!(i < j && v[i] + v[j] == target);
            }
        }
    }
}
```

(`HashSet` используется тестами в упражнениях.) Последний пример в `two_sum_examples` — граничный случай: `i32::MAX + i32::MIN` равно `-1`, а наивное `target - value` в `i32` переполнилось бы. Быстрая функция и перебор могут вернуть *разные* пары, когда их несколько, поэтому случайный тест сравнивает только «есть ли ответ?» и затем проверяет, что быстрая пара действительно даёт нужную сумму.

**7. Бенчмарк.** Обеим версиям приходится просмотреть всё, когда пары нет. Берём 5000 чётных чисел и нечётную цель. Добавьте в `Cargo.toml` секцию `[[bench]]` с `name = "practice"` и `harness = false` (criterion уже в `[dev-dependencies]`) и создайте `benches/practice.rs`:

```rust
use criterion::{Criterion, criterion_group, criterion_main};
use p07_algorithms::practice::{two_sum, two_sum_slow};
use std::hint::black_box;

fn bench_two_sum(c: &mut Criterion) {
    // no pair adds up to the target, so both versions must look at everything
    let values: Vec<i32> = (0..5_000).map(|i| i * 2).collect();
    let target = 1; // an odd target, all numbers are even
    let mut group = c.benchmark_group("two_sum n = 5000");
    group.bench_function("brute force", |b| {
        b.iter(|| two_sum_slow(black_box(&values), black_box(target)))
    });
    group.bench_function("hash map", |b| {
        b.iter(|| two_sum(black_box(&values), black_box(target)))
    });
    group.finish();
}

criterion_group!(benches, bench_two_sum);
criterion_main!(benches);
```

```bash
cargo bench --bench practice -- --warm-up-time 1 --measurement-time 3
```

```text
two_sum n = 5000/brute force
                        time:   [3.4147 ms 3.5832 ms 3.7948 ms]
two_sum n = 5000/hash map
                        time:   [157.32 µs 171.30 µs 188.69 µs]
```

(Mac на Apple M4; ваши числа будут другими.) Хеш-таблица быстрее примерно в 21 раз при n = 5000, и разрыв растёт с n, потому что O(n²) растёт быстрее O(n). Обратите внимание, что это *не* в 5000 раз: хеширование и выделение памяти стоят реального времени. Для крошечных входов (десяток чисел) перебор может даже выиграть. Поэтому и нужен шаг 7.

> ⚡ **Производительность:** асимптотическое улучшение (с O(n²) до O(n)) даёт большой выигрыш, только когда `n` велико. Всегда проверяйте ограничения и измеряйте на реалистичном размере, прежде чем «оптимизировать».

## 3. Пять задач

Пять задач ниже — ваши упражнения. Для каждой: пересформулируйте её, напишите три примера как тесты, напишите перебор или проверенного судью, затем настоящее решение. **Назовите временную и пространственную сложность до того, как откроете ответ**; в каждом ответе указана своя. Кладите свои решения в `src/practice.rs` (в ответах использованы показанные имена).

## Частые ошибки

**1. Не обработан пустой вход.** `values[0]` для пустого среза:

```rust
fn max_subarray_sum(values: &[i64]) -> i64 {
    let mut best = values[0];
    let mut ending_here = values[0];
    for &value in &values[1..] {
        ending_here = value.max(ending_here + value);
        best = best.max(ending_here);
    }
    best
}

fn main() {
    println!("{}", max_subarray_sum(&[-2, 1, -3, 4]));
    println!("{}", max_subarray_sum(&[]));
}
```

```text
4

thread 'main' (4806768) panicked at src/main.rs:2:20:
index out of bounds: the len is 0 but the index is 0
```

**Что это значит:** шаг 2 метода требует граничный случай именно поэтому. **Исправление:** возвращайте `Option` и используйте `split_first()?`, как в Упражнении 5 ниже.

**2. Зависимость от порядка `HashMap`.** `HashMap` не обещает никакого порядка, и Rust намеренно меняет его от запуска к запуску. Одна и та же программа, запущенная четыре раза:

```rust
use std::collections::HashMap;

fn main() {
    let mut groups: HashMap<&str, usize> = HashMap::new();
    for key in ["aet", "ant", "abt"] {
        groups.insert(key, key.len());
    }
    let keys: Vec<_> = groups.keys().collect();
    println!("{keys:?}");
}
```

```text
["aet", "ant", "abt"]
["abt", "aet", "ant"]
["ant", "abt", "aet"]
["ant", "abt", "aet"]
```

**Что это значит:** тест, сравнивающий результат, построенный обходом `HashMap`, с фиксированным ожидаемым списком, время от времени будет падать (нестабильный тест). **Исправление:** сортируйте перед возвратом или сравнением либо используйте `BTreeMap`, который хранит ключи по порядку.

**3. Начинать писать код до примеров.** Если вы не можете записать три примера с ответами, вы ещё не поняли задачу. Если записать примеры сначала как тесты, вы заодно получаете бесплатную проверку перебора.

## Попробуйте сами

1. Возьмите разобранный пример и *измените задачу*: верните **все** пары позиций с нужной суммой (а не только первую). Пройдите семь шагов письменно: что меняется в переборе, в лишней работе, в лучшем алгоритме, в сложности (подсказка: число ответов само может быть O(n²))?
2. Запустите `cargo test` после добавления `two_sum` и его тестов; затем `cargo bench --bench practice -- --warm-up-time 1 --measurement-time 3` и запишите два числа.
3. `cargo fmt && cargo clippy --all-targets -- -D warnings && cargo test`, затем `git add code/p07-algorithms && git commit -m "feat(p07): practice problems"` после каждой законченной задачи, если хотите (один маленький коммит на задачу делает историю читаемой).

## Упражнения

### Упражнение 1 — Сбалансированные скобки (легко)

Текст может содержать скобки `(`, `)`, `[`, `]`, `{`, `}` и любые другие символы. Скобки *сбалансированы*, когда каждая закрывающая скобка соответствует последней ещё открытой скобке **того же вида** и в конце ничего не остаётся открытым. Напишите `pub fn is_balanced(text: &str) -> bool`. Примеры: `"(a[b]{c})"` сбалансирован; `"(]"`, `"(()"`, `")("` и `"]"` нет; `""` да. Назовите сложность.

<details>
<summary>Подсказка 1</summary>

Последняя ещё открытая скобка: какая структура данных даёт «последним пришёл — первым вышел» ([Урок 07.5](05-stacks-and-queues.md))?

</details>

<details>
<summary>Подсказка 2</summary>

Кладите в стек каждую открывающую скобку. Для закрывающей достаньте элемент и сравните с ожидаемой открывающей. Что если стек пуст (`pop` даёт `None`)? Что остаётся в стеке в конце?

</details>

<details>
<summary>Ответ</summary>

```rust
/// Exercise 1: are the brackets `()`, `[]`, `{}` in the text balanced?
/// Time O(n), space O(n) for the stack.
pub fn is_balanced(text: &str) -> bool {
    let mut stack: Vec<char> = Vec::new();
    for c in text.chars() {
        match c {
            '(' | '[' | '{' => stack.push(c),
            ')' | ']' | '}' => {
                let open = match c {
                    ')' => '(',
                    ']' => '[',
                    _ => '{',
                };
                if stack.pop() != Some(open) {
                    return false;
                }
            }
            _ => {} // other characters are ignored
        }
    }
    stack.is_empty()
}
```

Тесты: примеры и судья, который снова и снова удаляет из текста пары `()`, `[]`, `{}`, пока что-то меняется (медленно, но заведомо верно):

```rust
    #[test]
    fn brackets() {
        assert!(is_balanced(""));
        assert!(is_balanced("(a[b]{c})"));
        assert!(!is_balanced("(]"));
        assert!(!is_balanced("(()"));
        assert!(!is_balanced(")("));
        assert!(!is_balanced("]"));
    }

    #[test]
    fn brackets_agree_with_repeated_removal() {
        fn judge(text: &str) -> bool {
            let mut s: String = text.chars().filter(|c| "()[]{}".contains(*c)).collect();
            loop {
                let shorter = s.replace("()", "").replace("[]", "").replace("{}", "");
                if shorter == s {
                    return s.is_empty();
                }
                s = shorter;
            }
        }
        let mut rng = Rng::new(2);
        for _ in 0..3000 {
            let len = rng.range(0, 10);
            let text = random_text(&mut rng, "()[]{}x", len);
            assert_eq!(is_balanced(&text), judge(&text), "{text:?}");
        }
    }
```

Почему это работает: закрывающая скобка должна соответствовать *последней* неспаренной открывающей, а это вершина стека. `stack.pop() != Some(open)` обрабатывает сразу обе ошибки: пустой стек (`None`) и скобку не того вида. В конце стек должен быть пуст. **Время O(n)**, один проход; **память O(n)** на стек в худшем случае (текст из одних открывающих скобок). Судья в худшем случае O(n²), но для 10 символов это нормально.

</details>

### Упражнение 2 — Самый длинный блок без повторов (средне)

Напишите `pub fn longest_unique_run(text: &str) -> usize`: длина самого длинного блока соседних символов, в котором ни один символ не встречается дважды. `"abcabcbb"` даёт 3 (`"abc"`), `"bbbb"` даёт 1, `"pwwkew"` даёт 3 (`"wke"`), `"abba"` даёт 2, `""` даёт 0. Назовите сложность перебора и вашего решения.

<details>
<summary>Подсказка 1</summary>

Перебор: проверить каждый блок (начало, конец) на повторы: O(n³) с множеством. Лишняя работа: когда окно `[left, right]` без повторов и вы добавляете один символ, нужно знать только, есть ли *этот* символ уже внутри. Это скользящее окно ([Урок 07.15](15-two-pointers-and-sliding-window.md)).

</details>

<details>
<summary>Подсказка 2</summary>

Запоминайте для каждого символа последнюю позицию, где вы его видели. Если новый символ в `right` последний раз был в `position`, и эта позиция внутри окна, сдвиньте `left` на `position + 1`. Используйте `left.max(position + 1)`: зачем `max`? Подумайте про `"abba"`.

</details>

<details>
<summary>Ответ</summary>

```rust
/// Exercise 2: the length of the longest block of characters without a repeat.
/// Sliding window with the last position of each character.
/// Time O(n) on average, space O(k) for k different characters.
pub fn longest_unique_run(text: &str) -> usize {
    let mut last_seen: HashMap<char, usize> = HashMap::new();
    let mut left = 0; // the window is chars[left..=right]
    let mut best = 0;
    for (right, c) in text.chars().enumerate() {
        if let Some(&position) = last_seen.get(&c) {
            left = left.max(position + 1); // jump over the earlier copy
        }
        last_seen.insert(c, right);
        best = best.max(right - left + 1);
    }
    best
}
```

Тесты:

```rust
    #[test]
    fn unique_run() {
        assert_eq!(longest_unique_run(""), 0);
        assert_eq!(longest_unique_run("abcabcbb"), 3);
        assert_eq!(longest_unique_run("bbbb"), 1);
        assert_eq!(longest_unique_run("pwwkew"), 3);
        assert_eq!(longest_unique_run("abba"), 2);
    }

    #[test]
    fn unique_run_agrees_with_brute_force() {
        let mut rng = Rng::new(3);
        for _ in 0..2000 {
            let len = rng.range(0, 12);
            let text = random_text(&mut rng, "abcd", len);
            let chars: Vec<char> = text.chars().collect();
            let mut best = 0;
            for start in 0..chars.len() {
                for end in start..chars.len() {
                    let block = &chars[start..=end];
                    if block.iter().collect::<HashSet<_>>().len() == block.len() {
                        best = best.max(block.len());
                    }
                }
            }
            assert_eq!(longest_unique_run(&text), best, "{text:?}");
        }
    }
```

Почему это работает: `left` никогда не идёт назад (для этого и `max`: в `"abba"` последняя `a` была в позиции 0, которая уже левее окна), а `right` идёт вперёд один раз на символ, поэтому цикл **O(n)** с O(1) операций таблицы в среднем. Память O(k) для k разных символов. Судья — перебор: каждый блок, с `HashSet` для обнаружения повторов (O(n³), для 12 символов нормально).

</details>

### Упражнение 3 — k наибольших чисел (средне)

Напишите `pub fn k_largest(values: &[i32], k: usize) -> Vec<i32>`, которая возвращает `k` наибольших чисел, начиная с наибольшего. Если чисел меньше `k`, верните все. Сделайте это за время O(n log k), не сортируя весь вход, и объясните, почему это важно, когда `k` мало, а `n` огромно (например, 10 наибольших из 100 миллионов чисел).

<details>
<summary>Подсказка 1</summary>

Сортировка всего стоит O(n log n). Вам нужно помнить только `k` наибольших виденных чисел. Когда приходит новое число, какое из запомненных первым покидает набор?

</details>

<details>
<summary>Подсказка 2</summary>

Покидает *наименьшее* из запомненных: min-куча размера `k` (`BinaryHeap<Reverse<i32>>`, как в [Уроке 07.12](12-heaps-and-priority-queues.md)). Положите новое число; если в куче больше `k` элементов, сделайте pop. В конце отсортируйте (не более `k`) элементов.

</details>

<details>
<summary>Ответ</summary>

```rust
/// Exercise 3: the `k` biggest numbers, biggest first.
/// A min-heap of size `k` keeps the k biggest seen so far.
/// Time O(n log k), space O(k).
pub fn k_largest(values: &[i32], k: usize) -> Vec<i32> {
    let mut heap: BinaryHeap<Reverse<i32>> = BinaryHeap::new();
    for &value in values {
        heap.push(Reverse(value));
        if heap.len() > k {
            heap.pop(); // drop the smallest of the k + 1
        }
    }
    let mut result: Vec<i32> = heap.into_iter().map(|Reverse(v)| v).collect();
    result.sort_unstable_by(|a, b| b.cmp(a));
    result
}
```

Тест, где судья — стандартная библиотека (отсортировать всё, взять `k`):

```rust
    #[test]
    fn k_largest_examples_and_judge() {
        assert_eq!(k_largest(&[3, 1, 5, 12, 2, 11], 3), vec![12, 11, 5]);
        assert_eq!(k_largest(&[1, 2], 5), vec![2, 1]);
        assert_eq!(k_largest(&[1, 2], 0), Vec::<i32>::new());
        let mut rng = Rng::new(4);
        for _ in 0..1000 {
            let len = rng.range(0, 12);
            let v: Vec<i32> = (0..len).map(|_| rng.range(-5, 5) as i32).collect();
            let k = rng.range(0, 14) as usize;
            let mut expected = v.clone();
            expected.sort_unstable_by(|a, b| b.cmp(a));
            expected.truncate(k);
            assert_eq!(k_largest(&v, k), expected, "{v:?} k = {k}");
        }
    }
```

Почему это работает: в куче никогда не больше `k + 1` элементов, поэтому каждое из `n` чисел стоит O(log k) на `push` и `pop`: **время O(n log k)** плюс O(k log k) на итоговую сортировку; **память O(k)**. Для 10 наибольших из 100 миллионов чисел в куче не больше 11 элементов: память крошечная, а вход можно читать потоком, тогда как «отсортировать всё» требует всех чисел в памяти и времени O(n log n). При `k = 0` каждое число сразу кладётся и вынимается, поэтому результат пуст.

</details>

### Упражнение 4 — Сосчитайте острова (средне)

Карта — это список строк одинаковой длины из `'1'` (суша) и `'0'` (вода). *Остров* — группа клеток суши, связанных вверх, вниз, влево или вправо (не по диагонали). Напишите `pub fn count_islands(grid: &[&str]) -> usize`. Примеры: `["11000", "11000", "00100", "00011"]` содержит 3 острова; `["101", "010", "101"]` — 5 (диагонали не связывают). Назовите сложность.

<details>
<summary>Подсказка 1</summary>

Это задача на графы в маскировке: каждая клетка суши — вершина, а соседние клетки суши соединены рёбрами ([Урок 07.13](13-graphs-bfs-and-dfs.md)). Остров — компонента связности. Как посчитать компоненты?

</details>

<details>
<summary>Подсказка 2</summary>

Просмотрите все клетки. Найдя клетку суши, которую вы ещё не посещали, посчитайте один остров и обойдите весь остров поиском в ширину (очередь), чтобы никогда не считать его клетки повторно. Ведите сетку `seen`. Рекурсивный DFS (Depth-First Search, поиск в глубину) может переполнить стек на большой карте, поэтому надёжнее BFS или собственный стек.

</details>

<details>
<summary>Ответ</summary>

```rust
/// Exercise 4: how many islands ('1' = land, '0' = water, 4 directions)?
/// Breadth-first search from each unvisited land cell.
/// Time O(rows * columns), space O(rows * columns).
pub fn count_islands(grid: &[&str]) -> usize {
    let land: Vec<Vec<bool>> = grid
        .iter()
        .map(|row| row.chars().map(|c| c == '1').collect())
        .collect();
    let mut seen: Vec<Vec<bool>> = land.iter().map(|row| vec![false; row.len()]).collect();
    let mut islands = 0;
    for r in 0..land.len() {
        for c in 0..land[r].len() {
            if !land[r][c] || seen[r][c] {
                continue;
            }
            islands += 1; // a new island: flood it so we never count it again
            seen[r][c] = true;
            let mut queue = VecDeque::from([(r, c)]);
            while let Some((row, col)) = queue.pop_front() {
                for (dr, dc) in [(-1i32, 0i32), (1, 0), (0, -1), (0, 1)] {
                    let (nr, nc) = (row as i32 + dr, col as i32 + dc);
                    if nr < 0 || nc < 0 {
                        continue;
                    }
                    let (nr, nc) = (nr as usize, nc as usize);
                    let is_land = land.get(nr).and_then(|line| line.get(nc)) == Some(&true);
                    if is_land && !seen[nr][nc] {
                        seen[nr][nc] = true;
                        queue.push_back((nr, nc));
                    }
                }
            }
        }
    }
    islands
}
```

Тесты:

```rust
    #[test]
    fn islands_examples() {
        assert_eq!(count_islands(&[]), 0);
        assert_eq!(count_islands(&["000", "000"]), 0);
        assert_eq!(count_islands(&["111", "111"]), 1);
        assert_eq!(count_islands(&["11000", "11000", "00100", "00011"]), 3);
        assert_eq!(count_islands(&["101", "010", "101"]), 5); // diagonals do not connect
    }
```

Почему это работает: каждая клетка суши попадает в очередь не более одного раза (она помечается в `seen` *при постановке в очередь*), и для каждой клетки мы смотрим на 4 соседей: **время O(rows · columns)**, **память O(rows · columns)** на `seen` и очередь. Знаковая арифметика `i32` и проверка `nr < 0 || nc < 0` избегают потери значимости `usize` на границе, а `land.get(nr).and_then(|line| line.get(nc))` избегает выхода индекса за пределы на остальных границах. Для более сильного теста сравните с рекурсивной заливкой на 500 случайных сетках до 5 × 5 клеток; напишите его для практики по тому же образцу, что и остальные судьи.

</details>

### Упражнение 5 — Лучшая сумма блока (сложно)

Напишите `pub fn max_subarray_sum(values: &[i64]) -> Option<i64>`: наибольшая сумма непустого блока соседних чисел (`None` для пустого среза). Числа могут быть отрицательными. `[-2, 1, -3, 4, -1, 2, 1, -5, 4]` даёт 6 (блок `4, -1, 2, 1`). `[-4, -2, -9]` даёт `-2`. Назовите сложность перебора и вашего решения.

<details>
<summary>Подсказка 1</summary>

Перебор: для каждого начала суммируем до каждого конца: O(n²). Лишняя работа: суммы блоков с одним началом разделяют все элементы, кроме одного. Идея динамического программирования ([Урок 07.16](16-dynamic-programming.md)): пусть *состояние* — «наилучшая сумма блока, который **заканчивается** в позиции `i`».

</details>

<details>
<summary>Подсказка 2</summary>

Лучший блок, заканчивающийся в `i`, — это либо одно число `values[i]`, либо лучший блок, заканчивающийся в `i - 1`, продлённый на `values[i]`. Значит, `ending_here = max(value, ending_here + value)`. Ответ — максимум всех значений `ending_here`. Нужно только предыдущее значение, поэтому хватает одной переменной.

</details>

<details>
<summary>Ответ</summary>

```rust
/// Exercise 5: the biggest sum of a block of neighbouring numbers (Kadane).
/// `None` for an empty slice. Time O(n), space O(1).
pub fn max_subarray_sum(values: &[i64]) -> Option<i64> {
    let (&first, rest) = values.split_first()?;
    let mut best = first;
    let mut ending_here = first; // the best sum of a block that ends at this item
    for &value in rest {
        ending_here = value.max(ending_here + value);
        best = best.max(ending_here);
    }
    Some(best)
}
```

Тест, где судья — перебор за O(n²):

```rust
    #[test]
    fn kadane() {
        assert_eq!(max_subarray_sum(&[]), None);
        assert_eq!(max_subarray_sum(&[-3]), Some(-3));
        assert_eq!(max_subarray_sum(&[-2, 1, -3, 4, -1, 2, 1, -5, 4]), Some(6));
        assert_eq!(max_subarray_sum(&[-4, -2, -9]), Some(-2));
        let mut rng = Rng::new(6);
        for _ in 0..1000 {
            let len = rng.range(1, 12);
            let v: Vec<i64> = (0..len).map(|_| rng.range(-9, 9)).collect();
            let mut best = i64::MIN;
            for start in 0..v.len() {
                let mut sum = 0;
                for &x in &v[start..] {
                    sum += x;
                    best = best.max(sum);
                }
            }
            assert_eq!(max_subarray_sum(&v), Some(best), "{v:?}");
        }
    }
```

Почему это работает: соотношение `ending_here = max(value, ending_here + value)` говорит «начать новый блок здесь или продлить предыдущий»; отрицательное `ending_here` продлевать никогда не стоит, поэтому оно отбрасывается. Это динамическое программирование с состоянием, сжатым до одной переменной (алгоритм известен как *алгоритм Кадане*). **Время O(n)**, **память O(1)**. `split_first()?` возвращает `None` для пустого среза, поэтому паники от `values[0]` нет. Тест начинает `best` с `i64::MIN`, поэтому входы из одних отрицательных чисел покрыты. (Суммы огромных чисел могут переполнить `i64`; задача ограничивает числа, поэтому здесь мы это не проверяем.)

</details>

## Вопросы для самопроверки

1. Каковы семь шагов метода?

<details>
<summary>Ответ</summary>

Понять (вход, выход, пределы, граничные случаи); маленькие примеры (как тесты); перебор; найти лишнюю работу; лучший алгоритм; тесты против перебора; бенчмарк.

</details>

2. На входе 200 000 чисел. Можно ли использовать алгоритм O(n²)?

<details>
<summary>Ответ</summary>

Нет. n² — это 40 миллиардов шагов, то есть десятки секунд. Нужен O(n log n) или O(n): сортировка, куча, хеш-таблица, два указателя или скользящее окно.

</details>

3. Почему в `two_sum` поиск выполняется до вставки?

<details>
<summary>Ответ</summary>

Чтобы число не могло образовать пару само с собой, что нарушило бы правило о разных позициях. Если поиск идёт первым, в таблице только более ранние позиции.

</details>

4. Какая часть `k_largest` делает её O(n log k), а не O(n log n)?

<details>
<summary>Ответ</summary>

Куча никогда не растёт больше `k + 1` элементов, поэтому каждые `push` и `pop` стоят O(log k), а не O(log n).

</details>

## Итоги

- Решайте задачу за семь шагов: понять, маленькие примеры, перебор, найти лишнюю работу, лучший алгоритм, тесты, бенчмарк.
- Размер входа определяет допустимую сложность: около 20 — экспонента подходит, несколько тысяч — O(n²), несколько сотен тысяч — O(n log n), больше — O(n).
- Лучший алгоритм прячется в самой медленной строке перебора: повторяющийся поиск становится поиском в хеш-таблице, повторяющаяся сумма — скользящим окном, повторяющийся минимум — кучей.
- Всегда храните перебор как судью и сравнивайте на тысячах случайных маленьких входов; сначала записывайте граничные случаи (пусто, один элемент, дубликаты, границы) как тесты.
- Не зависьте от порядка `HashMap`; сортируйте вывод.
- Ускорение стоит больше всего, когда `n` велико; измеряйте, прежде чем решать.

## Источники и что почитать

- [The Rust Standard Library — std::collections](https://doc.rust-lang.org/std/collections/index.html) — `HashMap`, `HashSet`, `BinaryHeap`, `VecDeque` и руководство по выбору между ними.
- [The Rust Standard Library — HashMap](https://doc.rust-lang.org/std/collections/struct.HashMap.html) — `entry`, `or_default`, `get` и замечание о произвольном порядке обхода.
- [The Rust Standard Library — BinaryHeap](https://doc.rust-lang.org/std/collections/struct.BinaryHeap.html) — куча, использованная в `k_largest`.
- [The Rust Programming Language — 8.3 Storing Keys with Associated Values in Hash Maps](https://doc.rust-lang.org/book/ch08-03-hash-maps.html) — официальная глава о хеш-таблицах.
- [Criterion.rs User Guide](https://bheisler.github.io/criterion.rs/book/) — библиотека бенчмарков для шага 7.

---

[🏠 Начало](../START-HERE.md) · [← 07.17 Жадные алгоритмы](17-greedy-algorithms.md) · [📋 Этап 07](README.md) · [07.19 Проект: поиск маршрутов →](19-project-route-finder.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
