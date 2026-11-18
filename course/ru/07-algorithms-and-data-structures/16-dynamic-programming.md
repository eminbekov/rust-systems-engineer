<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 07.15 Два указателя и скользящее окно](15-two-pointers-and-sliding-window.md) · [📋 Этап 07](README.md) · [07.17 Жадные алгоритмы →](17-greedy-algorithms.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/07-algorithms-and-data-structures/16-dynamic-programming.md) <!-- nav -->

# Урок 07.16 — Динамическое программирование

> **Этап 07 — Алгоритмы и структуры данных** · Урок 16 из 20 · ⏱ около 2,5 часа (2 занятия)

## Что вы узнаете

- Почему простое рекурсивное решение может быть нелепо медленным и как посчитать повторную работу.
- Два лекарства: *мемоизация* (сверху вниз) и *таблица* (снизу вверх), и как уменьшить таблицу.
- Рецепт динамического программирования: состояние, рекуррентное соотношение, базовые случаи, порядок, ответ.
- Две классические задачи: наименьшее число монет и наибольшая общая подпоследовательность.
- Как проверить решение на динамическом программировании по решению перебором.

## Перед началом

- Вы прошли [Урок 07.15 — Два указателя и скользящее окно](15-two-pointers-and-sliding-window.md).
- Вы знаете рекурсию, базовые случаи и стек вызовов ([Урок 07.7](07-recursion.md)) и векторы ([Урок 07.3](03-arrays-and-vectors-under-the-hood.md)).
- Вы работаете в `code/p07-algorithms`. Этот урок добавляет модуль `src/dp.rs` (и `pub mod dp;` в `src/lib.rs`) и один бенчмарк.

## Новые слова

| Слово | Значение |
|---|---|
| **динамическое программирование** (dynamic programming, DP) | Решение задачи через решение её меньших версий по одному разу, сохранение ответов и их объединение. («Программирование» здесь означает «заполнение таблицы», а не написание кода.) |
| **подзадача** (subproblem) | Меньшая версия той же задачи, например `fib(3)` внутри `fib(5)`. |
| **перекрывающиеся подзадачи** (overlapping subproblems) | Одна и та же подзадача нужна снова и снова разным частям решения. Из-за этого простая рекурсия медленна, а DP полезно. |
| **оптимальная подструктура** (optimal substructure) | Лучший ответ задачи складывается из лучших ответов её подзадач. |
| **мемоизация** (memoization, memo) | Сохранение результата вызова в блокноте (векторе или словаре), чтобы следующий вызов с тем же входом только искал ответ. Ещё называется DP *сверху вниз* (top-down). |
| **таблица** (table, снизу вверх) | Заполнение массива ответов от наименьшей подзадачи к наибольшей в таком порядке, чтобы всё нужное уже было заполнено. |
| **состояние** (state) | Несколько чисел, которые полностью описывают подзадачу, например «сумма `a`». В таблице одна ячейка на состояние. |
| **рекуррентное соотношение** (recurrence) | Формула, которая даёт ответ состояния через ответы меньших состояний, например `fib(n) = fib(n-1) + fib(n-2)`. |
| **подпоследовательность** (subsequence) | Некоторые символы текста в том же порядке, не обязательно рядом. `"AC"` — подпоследовательность `"ABC"`. |
| **LCS** (Longest Common Subsequence, наибольшая общая подпоследовательность) | Самая длинная последовательность, которая является подпоследовательностью обоих из двух текстов. |

## 1. Повторная работа: цена простой рекурсии

Числа Фибоначчи начинаются с 0 и 1, а каждое следующее равно сумме двух предыдущих: 0, 1, 1, 2, 3, 5, 8, 13, … Прямой перевод этого правила на Rust, а затем его версия, которая считает собственные вызовы. Создайте `src/dp.rs`:

```rust
/// Fibonacci by plain recursion. Time O(2^n), very slow.
pub fn fib_naive(n: u32) -> u64 {
    if n < 2 {
        u64::from(n)
    } else {
        fib_naive(n - 1) + fib_naive(n - 2)
    }
}

/// The same recursion, but it counts how many calls were made.
pub fn fib_counting(n: u32, calls: &mut u64) -> u64 {
    *calls += 1;
    if n < 2 {
        u64::from(n)
    } else {
        fib_counting(n - 1, calls) + fib_counting(n - 2, calls)
    }
}
```

Посмотрите на вызовы для `fib(5)`. Каждый прямоугольник — один вызов; одни и те же вызовы появляются снова и снова:

```text
                          fib(5)
                  /                  \
             fib(4)                  fib(3)             fib(3) is computed 2 times
            /      \                /      \            fib(2) is computed 3 times
        fib(3)    fib(2)        fib(2)    fib(1)
        /    \    /    \        /    \
    fib(2) fib(1) fib(1) fib(0) fib(1) fib(0)
    /    \
 fib(1) fib(0)                                          15 calls for n = 5
```

Создайте `examples/e16_calls.rs` и запустите его в релизном режиме (оптимизированная сборка из [Урока 05.12](../05-rust-intermediate/12-benchmarking-basics.md)):

```rust
use p07_algorithms::dp::{fib_counting, fib_memo, fib_two_vars};
use std::time::Instant;

fn main() {
    for n in [5, 10, 20, 30] {
        let mut calls = 0;
        let value = fib_counting(n, &mut calls);
        println!("fib({n}) = {value}, recursive calls: {calls}");
    }
    let start = Instant::now();
    let mut calls = 0;
    let value = fib_counting(40, &mut calls);
    println!(
        "fib(40) = {value}, recursive calls: {calls}, took {:?}",
        start.elapsed()
    );
    println!("fib_memo(90) = {:?}", fib_memo(90));
    println!("fib_two_vars(93) = {:?}", fib_two_vars(93));
    println!("fib_two_vars(94) = {:?}", fib_two_vars(94));
}
```

(Функции `fib_memo` и `fib_two_vars` появятся в разделе 2; добавьте их сначала или закомментируйте эти три строки.)

```bash
cargo run -q --release --example e16_calls
```

```text
fib(5) = 5, recursive calls: 15
fib(10) = 55, recursive calls: 177
fib(20) = 6765, recursive calls: 21891
fib(30) = 832040, recursive calls: 2692537
fib(40) = 102334155, recursive calls: 331160281, took 382.728166ms
fib_memo(90) = Some(2880067194370816120)
fib_two_vars(93) = Some(12200160415121876738)
fib_two_vars(94) = None
```

**Что происходит:**

1. Число вызовов растёт примерно в 1,6 раза при увеличении `n` на 1 (177 → 21 891 → 2 692 537 для n = 10, 20, 30). Это *экспоненциальный* рост, O(2ⁿ) (точнее, около 1,6ⁿ). Запрос `fib(40)` делает 331 миллион вызовов и занимает на этом Mac около 0,4 секунды; `fib(100)` потребовало бы десятки тысяч лет.
2. Почти все вызовы повторяют работу. *Разных* подзадач всего 41: от `fib(0)` до `fib(40)`. Если решить каждую один раз, хватило бы 41 маленького шага.
3. Последние строки — предпросмотр: быстрые версии ниже считают `fib(90)` мгновенно. `fib(94)` не помещается в `u64`, поэтому мы возвращаем `None` вместо паники.

Такая ситуация — рекурсия с перекрывающимися подзадачами — и есть сигнал для динамического программирования. (Когда подзадачи **не** перекрываются, как в сортировке слиянием, рекурсия уже хороша.)

## 2. Два лекарства: заметки и таблица

**Лекарство 1: мемоизация (сверху вниз).** Оставьте рекурсию, но добавьте блокнот. Перед вычислением `fib(n)` загляните в блокнот; после вычисления запишите в него результат.

```rust
/// Top-down: recursion plus a memo (a notebook of answers already found).
/// Returns `None` when the result does not fit into `u64` (from n = 94 on).
pub fn fib_memo(n: usize) -> Option<u64> {
    let mut memo: Vec<Option<u64>> = vec![None; n + 1];
    fib_memo_rec(n, &mut memo)
}

fn fib_memo_rec(n: usize, memo: &mut Vec<Option<u64>>) -> Option<u64> {
    if n < 2 {
        return Some(n as u64);
    }
    if let Some(known) = memo[n] {
        return Some(known); // already solved: no new work
    }
    let result = fib_memo_rec(n - 1, memo)?.checked_add(fib_memo_rec(n - 2, memo)?)?;
    memo[n] = Some(result);
    Some(result)
}
```

**Что происходит:**

1. `memo[n]` равно `None`, пока `fib(n)` неизвестно, и `Some(value)` после решения. Блокнот передаётся вниз по `&mut`, чтобы все вызовы делили его.
2. Первый раз, когда состояние нужно, оно вычисляется (два рекурсивных вызова); каждый следующий раз — одно чтение. Состояний `n + 1`, и каждое вычисляется один раз: **время O(n), память O(n)**.
3. `checked_add` возвращает `None` при переполнении, а `?` передаёт этот `None` вверх. Функция никогда не паникует ([Урок 04.10](../04-structs-enums-collections-errors/10-error-handling.md)).

**Лекарство 2: таблица (снизу вверх).** Вообще не рекурсируйте. Заполняйте массив от наименьшего состояния вверх, чтобы при вычислении `table[i]` нужные ячейки уже были заполнены.

```rust
/// Bottom-up: fill the answers from small to large. Time O(n), space O(n).
pub fn fib_table(n: usize) -> Option<u64> {
    let mut table = vec![0u64; n.max(1) + 1];
    table[1] = 1;
    for i in 2..=n {
        table[i] = table[i - 1].checked_add(table[i - 2])?;
    }
    Some(table[n])
}
```

У подхода снизу вверх нет стека вызовов: нет риска переполнения стека и нет стоимости вызова функций. Сверху вниз писать проще, когда у вас уже есть рекурсивная формула и когда нужны только некоторые состояния.

**Уменьшение таблицы.** Для `table[i]` нужны только две предыдущие ячейки, так что весь массив — расточительство. Храните две переменные:

```rust
/// Bottom-up with only the last two answers. Time O(n), space O(1).
pub fn fib_two_vars(n: usize) -> Option<u64> {
    if n == 0 {
        return Some(0);
    }
    // `older` and `newer` are fib(i - 1) and fib(i); we start at i = 1
    let (mut older, mut newer) = (0u64, 1u64);
    for _ in 1..n {
        (older, newer) = (newer, older.checked_add(newer)?);
    }
    Some(newer)
}
```

> 📏 **Соглашение:** сначала получите верное DP с полной таблицей. Уменьшайте память, только если это важно, и только после того, как тесты проходят.

> 🧪 **Тестирование:** медленная `fib_naive` — судья для малых `n`, а граница переполнения — классический граничный случай. Добавьте в `src/dp.rs`:

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use crate::testutil::Rng;

    #[test]
    fn fib_small_values() {
        let expected = [0, 1, 1, 2, 3, 5, 8, 13, 21, 34, 55];
        for (n, &want) in expected.iter().enumerate() {
            assert_eq!(fib_naive(n as u32), want);
            assert_eq!(fib_memo(n), Some(want));
            assert_eq!(fib_table(n), Some(want));
            assert_eq!(fib_two_vars(n), Some(want));
        }
    }

    #[test]
    fn fib_versions_agree_and_overflow_is_detected() {
        for n in 0..=25 {
            assert_eq!(fib_naive(n as u32), fib_two_vars(n).unwrap());
        }
        assert_eq!(fib_two_vars(93), Some(12_200_160_415_121_876_738));
        assert_eq!(fib_memo(93), Some(12_200_160_415_121_876_738));
        assert_eq!(fib_table(93), Some(12_200_160_415_121_876_738));
        assert_eq!(fib_two_vars(94), None);
        assert_eq!(fib_memo(94), None);
        assert_eq!(fib_table(94), None);
    }
}
```

`fib(93)` = 12 200 160 415 121 876 738 — наибольшее число Фибоначчи, которое помещается в `u64`; `fib(94)` не помещается. (Остальные тесты этого урока попадают в тот же модуль `tests`.)

## 3. Рецепт

Каждое решение на DP отвечает на пять вопросов. Запишите ответы комментариями перед кодом:

1. **Состояние.** Какие несколько чисел описывают подзадачу? (Фибоначчи: `n`.)
2. **Рекуррентное соотношение.** Как ответ состояния получается из меньших состояний? (`fib(n) = fib(n-1) + fib(n-2)`.)
3. **Базовые случаи.** У каких состояний есть ответ без соотношения? (`fib(0) = 0`, `fib(1) = 1`.)
4. **Порядок.** В каком порядке заполняются состояния, чтобы каждое соотношение читало только заполненные ячейки? (По возрастанию `n`.)
5. **Ответ.** В какой ячейке итоговый ответ? (`table[n]`.)

Время работы почти всегда равно **(число состояний) × (работа на состояние)**, а память — числу состояний (или меньше, если соотношение читает только несколько строк назад).

## 4. Пример: наименьшее число монет

**Задача.** Даны номиналы монет (каждого сколько угодно) и сумма `amount`. Найдите **наименьшее** число монет, номиналы которых в сумме дают ровно эту сумму, или скажите, что это невозможно.

1. **Состояние:** `best[a]` = наименьшее число монет для суммы `a` (или «невозможно»).
2. **Рекуррентное соотношение:** последней может быть любая монета `c` с `c <= a`; до неё нужно было `best[a - c]` монет. Значит, `best[a] = 1 + min(best[a - c])` по всем подходящим монетам `c`.
3. **Базовый случай:** `best[0] = 0` (ноль монет дают ноль).
4. **Порядок:** `a` от 1 до `amount`.
5. **Ответ:** `best[amount]`.

```text
coins = [1, 3, 4], amount = 6

a        0  1  2  3  4  5  6
best[a]  0  1  2  1  1  2  2

best[6] = 1 + min( best[6-1] = best[5] = 2,     <- use coin 1
                   best[6-3] = best[3] = 1,     <- use coin 3
                   best[6-4] = best[2] = 2 )    <- use coin 4
        = 1 + 1 = 2       (3 + 3)
```

```rust
/// The fewest coins that add up to `amount`, or `None` if it is impossible.
/// Every coin value must be at least 1. Time O(amount * coins), space O(amount).
pub fn min_coins(coins: &[usize], amount: usize) -> Option<usize> {
    // best[a] = the fewest coins for the amount a (None = impossible)
    let mut best: Vec<Option<usize>> = vec![None; amount + 1];
    best[0] = Some(0);
    for a in 1..=amount {
        for &coin in coins {
            // `checked_sub` is None when the coin is bigger than the amount
            if let Some(before) = a.checked_sub(coin).and_then(|rest| best[rest]) {
                let candidate = before + 1;
                if best[a].is_none_or(|current| candidate < current) {
                    best[a] = Some(candidate);
                }
            }
        }
    }
    best[amount]
}
```

**Что происходит:**

1. `None` означает «невозможно». Это безопаснее магического числа «бесконечность» (см. «Частые ошибки»).
2. Для каждой суммы `a` и каждой монеты `a.checked_sub(coin)` даёт остаток `a - coin` или `None`, если монета слишком велика. Затем `.and_then(|rest| best[rest])` ищет ответ для остатка; он тоже `None`, если остаток невозможен.
3. `is_none_or` равно `true`, когда `best[a]` всё ещё `None` или когда новый кандидат меньше текущего значения.
4. Два цикла: `amount` состояний на `coins.len()` монет, значит, **время O(amount · coins)**; один вектор — **память O(amount)**.

> 💡 **Совет:** попробуйте жадную идею «всегда брать самую большую подходящую монету» на монетах `[1, 3, 4]` и сумме 6: она берёт 4 + 1 + 1 = 3 монеты, а DP нашло 3 + 3 = 2. Жадный алгоритм работает не всегда; DP работает всегда. [Урок 07.17](17-greedy-algorithms.md) показывает, когда жадность безопасна.

Судья — рекурсия перебором, которая пробует каждую первую монету и ничего не запоминает. Она экспоненциальна, но для сумм до 14 это неважно:

```rust
/// Tries every way (exponential time). Used only as a judge in tests.
pub fn min_coins_slow(coins: &[usize], amount: usize) -> Option<usize> {
    if amount == 0 {
        return Some(0);
    }
    coins
        .iter()
        .filter(|&&coin| coin <= amount)
        .filter_map(|&coin| min_coins_slow(coins, amount - coin))
        .map(|count| count + 1)
        .min()
}
```

```rust
    #[test]
    fn coins_examples() {
        assert_eq!(min_coins(&[1, 3, 4], 6), Some(2)); // 3 + 3
        assert_eq!(min_coins(&[2], 3), None);
        assert_eq!(min_coins(&[5, 2], 0), Some(0));
        assert_eq!(min_coins(&[], 5), None);
        assert_eq!(min_coins(&[1, 5, 10, 25], 63), Some(6)); // 25+25+10+1+1+1
    }

```

Случайный тест использует `Rng` из `src/testutil.rs` с методом `range`, который вы добавили в Уроке 07.15 (строка `use crate::testutil::Rng;` уже стоит в начале модуля `tests`). `range` возвращает `i64`, поэтому количества и сумма переводятся в `usize` через `as usize`:

```rust
    #[test]
    fn coins_agree_with_the_slow_version() {
        let mut rng = Rng::new(2024);
        for _ in 0..500 {
            let count = rng.range(0, 4) as usize;
            let coins: Vec<usize> = (0..count).map(|_| rng.range(1, 7) as usize).collect();
            let amount = rng.range(0, 14) as usize;
            assert_eq!(
                min_coins(&coins, amount),
                min_coins_slow(&coins, amount),
                "coins = {coins:?}, amount = {amount}"
            );
        }
    }
```

Случайный тест создаёт от 0 до 4 монет номиналом от 1 до 7 и суммы до 14, поэтому покрывает «нет монет», «невозможные суммы» и повторяющиеся монеты.

## 5. Таблица с двумя измерениями: наибольшая общая подпоследовательность

**Задача.** Найдите длину наибольшей общей подпоследовательности (LCS) двух текстов. Для `"AGCAT"` и `"GAC"` она равна 2 (например, `"AC"` или `"GC"`).

1. **Состояние:** `table[i][j]` = длина LCS первых `i` символов `a` и первых `j` символов `b`. Два числа, значит, двумерная таблица.
2. **Рекуррентное соотношение:** посмотрите на последние символы `a[i-1]` и `b[j-1]`. Если они равны, они продолжают лучший ответ для более коротких текстов: `table[i-1][j-1] + 1`. Если нет, один из них не используется: берём лучшее из `table[i-1][j]` и `table[i][j-1]`.
3. **Базовые случаи:** для пустого текста ответ 0: строка 0 и столбец 0 состоят из нулей.
4. **Порядок:** строки сверху вниз, каждая строка слева направо (каждой ячейке нужны только ячейки выше или левее).
5. **Ответ:** правая нижняя ячейка.

```text
            b:   ""  G  A  C
 a: ""            0  0  0  0
    A             0  0  1  1
    G             0  1  1  1
    C             0  1  1  2
    A             0  1  2  2
    T             0  1  2  2     <- the answer is the last cell: 2
```

```rust
/// The length of the longest common subsequence of two texts.
/// Time O(n * m), space O(n * m).
pub fn lcs_len(a: &str, b: &str) -> usize {
    let a: Vec<char> = a.chars().collect();
    let b: Vec<char> = b.chars().collect();
    // table[i][j] = answer for the first i characters of a and the first j of b
    let mut table = vec![vec![0usize; b.len() + 1]; a.len() + 1];
    for i in 1..=a.len() {
        for j in 1..=b.len() {
            table[i][j] = if a[i - 1] == b[j - 1] {
                table[i - 1][j - 1] + 1
            } else {
                table[i - 1][j].max(table[i][j - 1])
            };
        }
    }
    table[a.len()][b.len()]
}
```

**Что происходит:**

1. Тексты становятся `Vec<char>`, чтобы можно было индексировать символы (`String` нельзя индексировать по позиции, [Урок 04.8](../04-structs-enums-collections-errors/08-strings-in-depth.md)).
2. `vec![vec![0; m + 1]; n + 1]` создаёт `n + 1` строк по `m + 1` нулей. Строка 0 и столбец 0 — базовые случаи.
3. Ячейка в строке `i` и столбце `j` говорит о `a[i - 1]` и `b[j - 1]`, потому что в таблице есть лишняя строка и лишний столбец для пустого префикса. Забыть этот сдвиг — самая частая ошибка.
4. `n · m` состояний с O(1) работы каждое: **время и память O(n · m)**. Поскольку строке нужна только строка выше, память можно уменьшить до двух строк (Упражнение 2).

Судья пробует каждую подпоследовательность `a` (их 2ⁿ, они выбираются битами числа) и проверяет, является ли она подпоследовательностью `b`:

```rust
    /// Judge for lcs_len: try every subsequence of `a` (2^n of them).
    fn lcs_slow(a: &str, b: &str) -> usize {
        let a: Vec<char> = a.chars().collect();
        let mut best = 0;
        for mask in 0u32..(1 << a.len()) {
            let picked: Vec<char> = (0..a.len())
                .filter(|&i| mask & (1 << i) != 0)
                .map(|i| a[i])
                .collect();
            let mut rest = b.chars();
            if picked.iter().all(|c| rest.any(|x| x == *c)) {
                best = best.max(picked.len());
            }
        }
        best
    }

    #[test]
    fn lcs_examples_and_judge() {
        assert_eq!(lcs_len("AGCAT", "GAC"), 2);
        assert_eq!(lcs_len("", "abc"), 0);
        assert_eq!(lcs_len("abc", "abc"), 3);
        assert_eq!(lcs_len("abc", "xyz"), 0);
        let mut rng = Rng::new(7);
        for _ in 0..300 {
            let make = |rng: &mut Rng| -> String {
                (0..rng.range(0, 8))
                    .map(|_| (b'a' + rng.range(0, 2) as u8) as char)
                    .collect()
            };
            let a = make(&mut rng);
            let b = make(&mut rng);
            assert_eq!(lcs_len(&a, &b), lcs_slow(&a, &b), "a = {a:?}, b = {b:?}");
        }
    }
```

В `lcs_slow` вызов `rest.any(...)` расходует итератор `rest` по мере поиска, поэтому каждый выбранный символ ищется *после* предыдущего: это ровно правило подпоследовательности. В текстах используются только буквы `a`, `b`, `c` (`b'a' + 0..=2`), чтобы общие подпоследовательности встречались часто. Запустите `cargo test dp`:

```text
running 5 tests
test dp::tests::coins_examples ... ok
test dp::tests::fib_small_values ... ok
test dp::tests::fib_versions_agree_and_overflow_is_detected ... ok
test dp::tests::lcs_examples_and_judge ... ok
test dp::tests::coins_agree_with_the_slow_version ... ok

test result: ok. 5 passed; 0 failed; 0 ignored; 0 measured; 70 filtered out; finished in 0.04s
```

(Порядок строк может отличаться. Отфильтрованы 70 тестов из основного текста Уроков 07.1–07.15; с тестами из упражнений ваше число будет больше.)

## 6. Измеряем разницу

Один бенчмарк, в стиле [Урока 05.12](../05-rust-intermediate/12-benchmarking-basics.md). Добавьте в `Cargo.toml` (`criterion = "0.8"` должен быть в `[dev-dependencies]`):

```toml
[[bench]]
name = "dp"
harness = false
```

`benches/dp.rs`:

```rust
use criterion::{Criterion, criterion_group, criterion_main};
use p07_algorithms::dp::{fib_memo, fib_naive, fib_table, fib_two_vars};
use std::hint::black_box;

fn bench_fib(c: &mut Criterion) {
    let mut group = c.benchmark_group("fib 30");
    group.bench_function("naive recursion", |b| b.iter(|| fib_naive(black_box(30))));
    group.bench_function("memo (top-down)", |b| b.iter(|| fib_memo(black_box(30))));
    group.bench_function("table (bottom-up)", |b| b.iter(|| fib_table(black_box(30))));
    group.bench_function("two variables", |b| b.iter(|| fib_two_vars(black_box(30))));
    group.finish();
}

criterion_group!(benches, bench_fib);
criterion_main!(benches);
```

```bash
cargo bench --bench dp -- --warm-up-time 1 --measurement-time 3
```

```text
fib 30/naive recursion  time:   [1.4161 ms 1.4206 ms 1.4255 ms]
fib 30/memo (top-down)  time:   [84.362 ns 84.663 ns 85.002 ns]
fib 30/table (bottom-up)
                        time:   [42.399 ns 42.889 ns 43.700 ns]
fib 30/two variables    time:   [8.2058 ns 8.2905 ns 8.3895 ns]
```

(Числа с Mac на Apple M4; у вас они будут другими, соотношения будут похожими.)

**Что происходит:** версия с мемоизацией примерно в 17 000 раз быстрее наивной рекурсии при `n = 30`, а версия с двумя переменными — примерно в 170 000 раз. У мемоизации есть расходы на выделение вектора и рекурсию, поэтому она медленнее таблицы; таблица выделяет вектор, поэтому она медленнее двух переменных. При `n = 40` разрыв был бы примерно в 100 раз больше, потому что наивная версия растёт экспоненциально, а остальные — линейно.

> ⚡ **Производительность:** DP превращает экспоненциальный алгоритм в полиномиальный за счёт запоминания. Цена — память под таблицу. Если посчитать число состояний, умноженное на работу на состояние, стоимость будет известна до того, как вы напишете код.

## Частые ошибки

**1. «Бесконечность», которая переполняется.** Во многих решениях на DP «невозможно» отмечают наибольшим числом. Тогда прибавление единицы к нему ломается:

```rust
fn min_coins(coins: &[usize], amount: usize) -> usize {
    let mut best = vec![usize::MAX; amount + 1]; // "infinity" for "impossible"
    best[0] = 0;
    for a in 1..=amount {
        for &coin in coins {
            if coin <= a && best[a - coin] + 1 < best[a] {
                best[a] = best[a - coin] + 1;
            }
        }
    }
    best[amount]
}

fn main() {
    println!("{}", min_coins(&[2], 3));
}
```

```text
thread 'main' (4697954) panicked at src/main.rs:6:29:
attempt to add with overflow
```

**Что это значит:** `best[1]` — «бесконечность» (`usize::MAX`), и `best[1] + 1` переполняется. **Исправление:** используйте `Option<usize>`, как в `min_coins`, либо явно пропускайте невозможные состояния, либо используйте «большое, но безопасное» значение, например `amount + 1`, в роли бесконечности.

**2. Переполнение стека в версии сверху вниз.** Мемоизация всё равно рекурсирует, на один вызов глубже на каждый шаг. Большое `n` исчерпывает стек вызовов ([Урок 07.7](07-recursion.md)):

```rust
fn fib_memo(n: usize, memo: &mut Vec<Option<u64>>) -> u64 {
    if n < 2 {
        return n as u64;
    }
    if let Some(known) = memo[n] {
        return known;
    }
    let result = fib_memo(n - 1, memo).wrapping_add(fib_memo(n - 2, memo));
    memo[n] = Some(result);
    result
}

fn main() {
    let n = 1_000_000;
    let mut memo = vec![None; n + 1];
    println!("{}", fib_memo(n, &mut memo));
}
```

```text
thread 'main' (4698048) has overflowed its stack
fatal runtime error: stack overflow, aborting
```

**Исправление:** используйте таблицу снизу вверх (у цикла нет стека вызовов) или следите, чтобы глубина рекурсии была небольшой.

**3. Сдвиг на единицу между таблицей и текстом.** В `lcs_len` ячейка `table[i][j]` описывает первые `i` символов, поэтому сравниваются символы `a[i - 1]` и `b[j - 1]`. Запись `a[i]` паникует на последней строке или сравнивает не те символы. Тесты с пустыми текстами и текстами из одной буквы это ловят.

**4. Чтение ещё не заполненной ячейки.** Если циклы идут в неверном порядке, соотношение читает нуль, который всего лишь начальное значение, и ответ неверен *без* какого-либо сообщения об ошибке. Поэтому каждое решение называет свой порядок, и поэтому судья-перебор так ценен.

## Попробуйте сами

1. Добавьте `fib_counting` и пример из раздела 1 и запустите. Замените `40` на `45` и предскажите число вызовов до запуска (подсказка: каждый шаг умножает примерно на 1,6).
2. Выполните `min_coins(&[1, 3, 4], 6)` вручную карандашом: заполните таблицу, как на картинке раздела 4. Затем замените монеты на `[2, 5]`, а сумму на 3 и проверьте, что ответ — `None`.
3. Запустите `cargo fmt && cargo clippy --all-targets -- -D warnings && cargo test`, затем `git add code/p07-algorithms && git commit -m "feat(p07): dynamic programming"`.

## Упражнения

### Упражнение 1 — Подъём по лестнице (легко)

В лестнице `n` ступенек. За раз можно подняться на 1 или 2 ступеньки. Сколькими различными способами можно добраться до верха? Напишите `pub fn ways_to_climb(n: usize) -> u64` со временем O(n) и памятью O(1) и тест. (Для `n = 0` есть 1 способ, для `n = 1` — 1 способ, для `n = 2` — 2 способа.)

<details>
<summary>Подсказка</summary>

Последний ход — это либо 1 ступенька (со ступеньки `n - 1`), либо 2 ступеньки (со ступеньки `n - 2`). Значит, `ways(n) = ways(n - 1) + ways(n - 2)`. Это правило Фибоначчи с другими начальными значениями. Проверьте числа 1, 1, 2, 3, 5, 8, 13.

</details>

<details>
<summary>Ответ</summary>

```rust
/// Ways to climb `n` stairs with steps of 1 or 2. Time O(n), space O(1).
pub fn ways_to_climb(n: usize) -> u64 {
    let (mut one_before, mut now) = (1u64, 1u64); // ways(0), ways(1)
    for _ in 1..n {
        (one_before, now) = (now, one_before + now);
    }
    now
}

// The test goes into the `tests` module at the end of `src/dp.rs`:
    #[test]
    fn stairs() {
        let expected = [1, 1, 2, 3, 5, 8, 13];
        for (n, &want) in expected.iter().enumerate() {
            assert_eq!(ways_to_climb(n), want, "n = {n}");
        }
    }
```

Почему это работает: состояние `n`, соотношение `ways(n) = ways(n-1) + ways(n-2)`, базовые случаи `ways(0) = ways(1) = 1`, и хранятся только два последних значения. (Для очень больших `n` `u64` переполнится; если это нужно, используйте `checked_add`, как в `fib_two_vars`.)

</details>

### Упражнение 2 — Сложность и память (средне)

(а) Назовите временную и пространственную сложность `min_coins` и `lcs_len`, обозначив `A` сумму, `C` число монет, а `n`, `m` длины текстов. (б) Как `lcs_len` может обойтись памятью O(min(n, m))? (в) Почему память `min_coins` **нельзя** уменьшить таким же способом?

<details>
<summary>Подсказка</summary>

Для (б) посмотрите, какие ячейки читает соотношение: только строку `i - 1` и строку `i`. Для (в): какие более ранние ячейки читает `best[a]`?

</details>

<details>
<summary>Ответ</summary>

(а) `min_coins`: время O(A · C), память O(A). `lcs_len`: время O(n · m), память O(n · m).

(б) Соотношение ячейки `(i, j)` читает только `(i-1, j-1)`, `(i-1, j)` и `(i, j-1)`: предыдущую и текущую строки. Храните только две строки (или одну строку и одну переменную), и пусть вдоль строки идёт более короткий текст: тогда память O(min(n, m)). Цена: по таблице уже нельзя восстановить саму подпоследовательность, можно получить только её длину.

(в) `best[a]` читает `best[a - c]` для *каждой* монеты `c`, поэтому ей может понадобиться ячейка, отстоящая назад на наибольший номинал. Пришлось бы хранить последние `max(coins)` ячеек, а это в общем случае не меньше O(A).

</details>

### Упражнение 3 — Рюкзак 0/1 (средне)

Есть предметы с весами и ценностями. В рюкзак помещается суммарный вес не более `capacity`. Каждый предмет можно взять **не более одного раза**. Напишите `pub fn knapsack(weights: &[usize], values: &[u32], capacity: usize) -> u32`, возвращающую наибольшую суммарную ценность. Проверьте её на небольших примерах, которые вы решите вручную.

<details>
<summary>Подсказка</summary>

Состояние: `best[c]` = наибольшая ценность при лимите веса `c`. Для каждого предмета `(w, v)`: `best[c] = max(best[c], best[c - w] + v)`. Чтобы использовать каждый предмет только один раз с одним вектором, идите `c` от `capacity` **вниз** до `w`. Что пойдёт не так, если идти вверх?

</details>

<details>
<summary>Ответ</summary>

```rust
/// 0/1 knapsack: the biggest total value with total weight <= capacity.
/// Each item is used at most once. Time O(items * capacity), space O(capacity).
pub fn knapsack(weights: &[usize], values: &[u32], capacity: usize) -> u32 {
    // best[c] = the biggest value with weight limit c
    let mut best = vec![0u32; capacity + 1];
    for (&w, &v) in weights.iter().zip(values) {
        // go from big c to small c, so that the item is used at most once
        for c in (w..=capacity).rev() {
            best[c] = best[c].max(best[c - w] + v);
        }
    }
    best[capacity]
}

// The test goes into the `tests` module at the end of `src/dp.rs`:
    #[test]
    fn knapsack_examples() {
        assert_eq!(knapsack(&[1, 3, 4, 5], &[1, 4, 5, 7], 7), 9); // items 3 and 4
        assert_eq!(knapsack(&[], &[], 10), 0);
        assert_eq!(knapsack(&[5], &[10], 4), 0);
        assert_eq!(knapsack(&[2, 3, 4], &[3, 4, 5], 5), 7); // items 1 and 2
        assert_eq!(knapsack(&[1, 1, 1], &[1, 2, 3], 2), 5); // items 2 and 3
    }
```

Почему это работает: когда `c` идёт вниз, `best[c - w]` ещё не обновлено для текущего предмета, поэтому предмет учитывается один раз. Если бы `c` шло вверх, `best[c - w]` уже включало бы предмет, и его можно было бы брать много раз (это версия «без ограничений», как с монетами). Время O(items · capacity), память O(capacity). Для более сильного теста напишите судью-перебор по всем подмножествам и сравните на случайных входах, как это делает `coins_agree_with_the_slow_version`.

</details>

## Вопросы для самопроверки

1. Что делает рекурсивное решение кандидатом на динамическое программирование?

<details>
<summary>Ответ</summary>

Перекрывающиеся подзадачи: рекурсия решает одну и ту же меньшую задачу много раз (как `fib(3)` в `fib(5)`), а лучший ответ складывается из лучших ответов меньших задач (оптимальная подструктура). Если подзадачи не перекрываются (сортировка слиянием), хватает простой рекурсии.

</details>

2. Чем отличается DP сверху вниз от DP снизу вверх?

<details>
<summary>Ответ</summary>

Сверху вниз сохраняет рекурсию и добавляет заметки; вычисляются только нужные состояния, но используется стек вызовов, который может переполниться. Снизу вверх заполняет таблицу от наименьшего состояния к наибольшему в цикле; рекурсии нет, обычно это быстрее, но нужно найти верный порядок заполнения.

</details>

3. Каковы время и память `min_coins` для суммы `A` и `C` номиналов?

<details>
<summary>Ответ</summary>

Время O(A · C) (A состояний, по C монет каждое), память O(A) (один вектор из A + 1 ячеек).

</details>

4. Почему `usize::MAX` — опасная метка «невозможно» в таблице DP?

<details>
<summary>Ответ</summary>

Соотношение прибавляет 1 к значениям из таблицы, а `usize::MAX + 1` переполняется (паника в отладочной сборке, неверное значение в релизной). Используйте `Option`, пропускайте невозможные состояния или берите безопасное большое значение, например `amount + 1`.

</details>

## Итоги

- Рекурсия с перекрывающимися подзадачами повторяет работу экспоненциально. Подсчёт вызовов это показывает: `fib(40)` делает 331 миллион вызовов на 41 разную подзадачу.
- Динамическое программирование решает каждую подзадачу один раз: сверху вниз с заметками или снизу вверх с таблицей. Цена — (число состояний) × (работа на состояние).
- Рецепт: состояние, рекуррентное соотношение, базовые случаи, порядок, ответ. Сначала запишите их комментариями.
- Сначала держите полную таблицу; уменьшайте её до нескольких строк или переменных только тогда, когда соотношение читает только их.
- Для «невозможно» и для переполнения используйте `Option` или `checked_*`; тестируйте пустые входы и граничные значения.
- Судья-перебор (все подмножества, все подпоследовательности, простая рекурсия) на крошечных случайных входах находит ошибки на единицу и ошибки порядка. В стандартной библиотеке нет помощника для DP, поэтому таблицу вы пишете сами.

## Источники и что почитать

- [The Rust Programming Language — 8.1 Storing Lists of Values with Vectors](https://doc.rust-lang.org/book/ch08-01-vectors.html) — векторы, на которых построены все таблицы.
- [The Rust Standard Library — Option](https://doc.rust-lang.org/std/option/enum.Option.html) — `is_none_or`, `and_then`, `map_or`.
- [The Rust Standard Library — u64::checked_add](https://doc.rust-lang.org/std/primitive.u64.html#method.checked_add) — арифметика, сообщающая о переполнении.
- [Criterion.rs User Guide](https://bheisler.github.io/criterion.rs/book/) — библиотека бенчмарков из раздела 6.
- [Rust by Example — Iterators](https://doc.rust-lang.org/rust-by-example/trait/iter.html) — короткие запускаемые примеры методов итераторов, использованных в судьях.

---

[🏠 Начало](../START-HERE.md) · [← 07.15 Два указателя и скользящее окно](15-two-pointers-and-sliding-window.md) · [📋 Этап 07](README.md) · [07.17 Жадные алгоритмы →](17-greedy-algorithms.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
