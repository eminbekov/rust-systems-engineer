<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 07.6 Как работают хеш-таблицы](06-hash-tables.md) · [📋 Этап 07](README.md) · [07.8 Простые алгоритмы сортировки →](08-simple-sorting-algorithms.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/07-algorithms-and-data-structures/07-recursion.md) <!-- nav -->

# Урок 07.7 — Рекурсия и стек вызовов

> **Этап 07 — Алгоритмы и структуры данных** · Урок 7 из 20 · ⏱ около 135 минут

## Что вы узнаете

- Что такое рекурсия: функция, которая вызывает саму себя на меньшей задаче, и базовый случай, который её останавливает.
- Что происходит на стеке вызовов при рекурсивном вызове и что такое переполнение стека.
- Как увидеть стоимость рекурсивного алгоритма: глубина (память стека), число вызовов (время) и почему наивные числа Фибоначчи — экспонента.
- «Разделяй и властвуй»: деление задачи пополам держит глубину небольшой.
- Как превратить рекурсию в цикл с собственным стеком.

## Перед началом

- Вы прошли [Урок 07.6 — Как работают хеш-таблицы](06-hash-tables.md).
- Вы знаете стек вызовов и кадры стека ([Урок 03.1](../03-ownership-and-borrowing/01-memory-stack-and-heap.md), где показано переполнение стека), рекурсивные типы с `Box` ([Урок 05.9](../05-rust-intermediate/09-smart-pointers.md)) и стек из Урока 07.5.

## Новые слова

| Слово | Значение |
|---|---|
| **рекурсия** (recursion) | Функция, вызывающая саму себя. Каждый вызов решает меньшую версию той же задачи. |
| **базовый случай** (base case) | Вход, достаточно малый, чтобы ответить сразу, без нового вызова. Он останавливает рекурсию. |
| **рекурсивный случай** (recursive case) | Часть, которая уменьшает задачу и снова вызывает функцию. |
| **стек вызовов** (call stack) | Стек кадров выполняющихся сейчас вызовов функций (Урок 03.1). Вызов добавляет кадр; возврат убирает его. |
| **глубина рекурсии** (recursion depth) | Сколько вызовов одновременно ждут на стеке. Это цена рекурсии по памяти. |
| **«разделяй и властвуй»** (divide and conquer) | Метод: разбить задачу на части (часто на две половины), решить каждую рекурсивно и объединить ответы. |
| **хвостовой вызов** (tail call) | Вызов, который функция делает самым последним. В некоторых языках для него переиспользуют кадр; Rust этого не обещает. |
| **экспоненциальный** (exponential) | Растущий как `2ⁿ`: каждый лишний шаг `n` удваивает работу (Урок 07.1). |

## 1. Функция, которая вызывает саму себя

Факториал `n!` — это `n * (n-1) * ... * 1`. Заметьте, что `n! = n * (n-1)!`. Это и есть рекурсия: **ответ для `n` составляется из
ответа для меньшего `n`**. У каждой рекурсивной функции две части:

1. **базовый случай** — вход, достаточно малый, чтобы ответить сразу (`0!` и `1!` равны `1`);
2. **рекурсивный случай** — вызов с *меньшим* входом, чтобы вызовы приближались к базовому случаю.

Без базового случая функция вызывает себя бесконечно. Без уменьшения входа она тоже никогда не закончится. Создайте
`src/recursion.rs`, добавьте `pub mod recursion;` в `src/lib.rs` и напишите первую функцию:

```rust
/// n! = n * (n-1) * ... * 1. Returns `None` if the result does not fit in a `u64`.
/// Base case: n <= 1. Recursive case: n * (n-1)!.
pub fn factorial(n: u64) -> Option<u64> {
    if n <= 1 {
        Some(1)
    } else {
        factorial(n - 1).and_then(|smaller| smaller.checked_mul(n))
    }
}
```

**Что происходит:** если `n <= 1`, ответ — `1` (базовый случай). Иначе `factorial(n - 1)` даёт `(n-1)!`, а `and_then` умножает его на
`n` через `checked_mul`. Результат, слишком большой для `u64`, становится `None`, а не переполнением (Урок 07.2), и `None` поднимается
обратно через все вызовы.

### Что делает компьютер: стек вызовов

`examples/e07_trace.rs` печатает строку, когда вызов начинается и когда он возвращается, с отступом по глубине:

```rust
fn factorial(n: u64, depth: usize) -> u64 {
    let indent = "  ".repeat(depth);
    println!("{indent}factorial({n}) called");
    let result = if n <= 1 {
        1
    } else {
        n * factorial(n - 1, depth + 1)
    };
    println!("{indent}factorial({n}) returns {result}");
    result
}

fn main() {
    factorial(4, 0);
}
```

```text
factorial(4) called
  factorial(3) called
    factorial(2) called
      factorial(1) called
      factorial(1) returns 1
    factorial(2) returns 2
  factorial(3) returns 6
factorial(4) returns 24
```

Читайте вывод как картину **стека вызовов** (Урок 03.1). `factorial(4)` не может закончиться раньше `factorial(3)`, который ждёт
`factorial(2)`, который ждёт `factorial(1)`. В самый глубокий момент на стеке четыре кадра; затем вызовы возвращаются один за другим,
и каждый умножает:

```text
 time ─────────────────────────────────────────────────▶
 stack  [f4] [f4 f3] [f4 f3 f2] [f4 f3 f2 f1] [f4 f3 f2] [f4 f3] [f4] []
         call  call    call       call (base)    return    return return return
                                  returns 1      2*1       3*2    4*6
```

**Глубина** для `factorial(4)` равна 4, а в общем случае `n`. Каждый кадр занимает несколько байт стека (параметры, локальные
переменные, адрес возврата). Поэтому цена рекурсии по памяти — **её глубина, умноженная на размер кадра**.

### У стека есть предел

Стек невелик (около 8 МиБ у главного потока программы на macOS, у других потоков меньше). Слишком глубокая рекурсия его исчерпывает.
`examples/e07_overflow.rs`:

```rust
fn depth(n: u64) -> u64 {
    if n == 0 { 0 } else { 1 + depth(n - 1) }
}

fn main() {
    println!("depth(1000) = {}", depth(1_000));
    println!("depth(100000000) = {}", depth(100_000_000));
}
```

```text
depth(1000) = 1000

thread 'main' (5430643) has overflowed its stack
fatal runtime error: stack overflow, aborting
```

`depth(1000)` — нормально. Сто миллионов вложенных вызовов — нет: систему останавливает программу (переполнение стека, как в Уроке
03.1). Это не паника: перехватить её нельзя.

> ⚠️ **Частая ошибка:** не рассчитывайте, что компилятор уберёт рекурсию. В *релизной* сборке (`cargo run --release --example e07_overflow`)
> оптимизатор случайно превращает именно эту функцию в цикл, и программа печатает ответ. Rust **не гарантирует** оптимизацию хвостовых
> вызовов, так что программа, работающая в релизе, может упасть в отладочной сборке (возможно и наоборот). Если глубина может быть
> большой или приходит снаружи, не используйте рекурсию или ограничьте глубину.

> 🔒 **Безопасность:** рекурсивная функция, повторяющая форму внешнего ввода (вложенный JSON — JavaScript Object Notation, распространённый текстовый формат данных; глубоко вложенное выражение, длинная
> цепочка ссылок), может быть обрушена атакующим, который пришлёт глубокую вложенность. Ограничьте глубину или используйте цикл и
> собственный стек (раздел 4).

## 2. Дешёвая рекурсия

**Алгоритм Евклида** находит наибольший общий делитель (наибольшее число, на которое делятся оба). Идея: `gcd(a, b)` равно
`gcd(b, a % b)`, а `gcd(a, 0)` равно `a`.

```rust
/// Greatest common divisor (Euclid's algorithm): gcd(a, 0) = a, gcd(a, b) = gcd(b, a % b).
pub fn gcd(a: u64, b: u64) -> u64 {
    if b == 0 { a } else { gcd(b, a % b) }
}
```

Для `gcd(48, 18)`: `gcd(18, 12)`, `gcd(12, 6)`, `gcd(6, 0)`, так что ответ — 6. Остаток быстро уменьшается: минимум вдвое за каждые
два шага, поэтому глубина — `O(log n)`.

Теперь два способа просуммировать срез. Оба — `O(n)` по времени. Они различаются глубиной.

```rust
/// Sums a slice as "first item + sum of the rest": the recursion is n calls deep.
pub fn sum_recursive(items: &[i64]) -> i64 {
    match items.split_first() {
        None => 0,
        Some((first, rest)) => first + sum_recursive(rest),
    }
}

/// Sums a slice as "sum of the left half + sum of the right half": only about log n calls deep.
pub fn sum_halves(items: &[i64]) -> i64 {
    match items.len() {
        0 => 0,
        1 => items[0],
        n => {
            let (left, right) = items.split_at(n / 2);
            sum_halves(left) + sum_halves(right)
        }
    }
}
```

**Что происходит:**

1. `sum_recursive` отрезает первый элемент (`split_first`) и прибавляет его к сумме остатка. Срез уменьшается на 1 каждый раз, поэтому
   глубина — **`n`**. Для среза из миллиона элементов это был бы миллион кадров.
2. `sum_halves` — это **«разделяй и властвуй»**: она режет срез на две половины (`split_at`), рекурсивно суммирует каждую и складывает
   два результата. Срез каждый раз делится пополам, поэтому глубина всего **`log n`**: около 20 для миллиона элементов. Общее число
   вызовов по-прежнему около `2n`, так что время — `O(n)`.

```text
sum_halves([1,2,3,4,5,6,7,8])
                 ┌───────── [1..8] ─────────┐
           [1,2,3,4]                    [5,6,7,8]
          [1,2]   [3,4]                [5,6]   [7,8]
         [1] [2] [3] [4]              [5] [6] [7] [8]       depth: 4 = log₂ 8 + 1
```

Урок: если задачу можно делить пополам, глубина рекурсии — `O(log n)`. На этой идее держатся сортировка слиянием и быстрая
сортировка (Урок 07.9) и двоичный поиск (Урок 07.10).

## 3. Дорогая рекурсия: Фибоначчи

Числа Фибоначчи — `0, 1, 1, 2, 3, 5, 8, ...`: каждое равно сумме двух предыдущих. Определение рекурсивное, и соблазн переписать его
напрямую велик:

```rust
/// The n-th Fibonacci number (0, 1, 1, 2, 3, 5, 8, ...), straight from the definition.
pub fn fib_recursive(n: u32) -> u64 {
    if n < 2 {
        n as u64
    } else {
        fib_recursive(n - 1) + fib_recursive(n - 2)
    }
}
```

Это правильно и ужасно медленно. Чтобы понять почему, посчитаем вызовы:

```rust
/// How many calls `fib_recursive(n)` makes (including the first call).
pub fn fib_calls(n: u32) -> u64 {
    if n < 2 {
        1
    } else {
        1 + fib_calls(n - 1) + fib_calls(n - 2)
    }
}
```

```rust
/// The same numbers with a loop: n steps and no recursion.
pub fn fib_iterative(n: u32) -> u64 {
    let (mut current, mut next) = (0u64, 1u64);
    for _ in 0..n {
        (current, next) = (next, current + next);
    }
    current
}
```

`fib_calls` устроена так же, как `fib_recursive`, но считает вызовы, а не складывает числа. `fib_iterative` вычисляет те же числа циклом,
который хранит последние два значения. `examples/e07_calls.rs` печатает оба числа:

```rust
use p07_algorithms::recursion::{fib_calls, fib_iterative, hanoi};

fn main() {
    println!("{:>4} {:>12} {:>14}", "n", "fib(n)", "calls");
    for n in [10, 20, 30, 35] {
        println!("{:>4} {:>12} {:>14}", n, fib_iterative(n), fib_calls(n));
    }

    println!();
    let mut moves = Vec::new();
    hanoi(3, 'A', 'C', 'B', &mut moves);
    println!("hanoi(3): {moves:?}");
    for n in [10, 20] {
        let mut moves = Vec::new();
        hanoi(n, 'A', 'C', 'B', &mut moves);
        println!("hanoi({n}) needs {} moves", moves.len());
    }
}
```

```text
   n       fib(n)          calls
  10           55            177
  20         6765          21891
  30       832040        2692537
  35      9227465       29860703

hanoi(3): [('A', 'C'), ('A', 'B'), ('C', 'B'), ('A', 'C'), ('B', 'A'), ('B', 'C'), ('A', 'C')]
hanoi(10) needs 1023 moves
hanoi(20) needs 1048575 moves
```

(К второй половине вывода мы вернёмся через минуту.) `fib(35)` — всего 9 миллионов, но для него нужно **почти 30 миллионов вызовов**.
Вызовы множатся: `fib(n)` вызывает `fib(n-1)` и `fib(n-2)`, а те снова и снова вызывают одни и те же меньшие значения:

```text
                     fib(5)
                 ┌──────┴──────┐
              fib(4)         fib(3)           fib(3) is computed twice,
           ┌─────┴─────┐    ┌───┴───┐         fib(2) three times, ...
        fib(3)       fib(2) fib(2) fib(1)
       ┌──┴──┐
    fib(2) fib(1)
```

Число вызовов растёт примерно в 1,6 раза на каждый шаг `n` (золотое сечение): **экспонента**, около `O(1.6ⁿ)`. Глубина рекурсии мала
(`n`), а время огромно. Этот шаблон — рекурсия, решающая одну и ту же меньшую задачу много раз — как раз то, что исправляют
**мемоизация** и динамическое программирование (Урок 07.16). Версия с циклом — `O(n)` по времени и `O(1)` по памяти.

### Ханойские башни

Классическая задача, показывающая, как рекурсия позволяет решить трудную на вид задачу, доверившись меньшей. Три стержня, `n` дисков
разного размера на первом. Нужно перенести все диски на другой стержень по одному, никогда не кладя больший диск на меньший.
Рекурсивное мышление: чтобы перенести `n` дисков, (1) уберите `n-1` меньших дисков с дороги на вспомогательный стержень, (2) перенесите
самый большой диск, (3) перенесите `n-1` дисков на него.

```rust
/// Towers of Hanoi: moves `n` disks from peg `from` to peg `to`, using peg `via` as a helper.
/// Every move is recorded as (from, to). The number of moves is 2^n - 1.
pub fn hanoi(n: u32, from: char, to: char, via: char, moves: &mut Vec<(char, char)>) {
    if n == 0 {
        return;
    }
    hanoi(n - 1, from, via, to, moves); // 1. move the n-1 smaller disks out of the way
    moves.push((from, to)); // 2. move the biggest disk
    hanoi(n - 1, via, to, from, moves); // 3. put the n-1 smaller disks back on top of it
}
```

Вектор `moves` записывает каждый шаг. Вторая половина вывода выше показывает ходы для 3 дисков, `hanoi(3)`: 7 ходов. Число ходов
подчиняется `moves(n) = 2 * moves(n-1) + 1`, что даёт `2ⁿ - 1`: 1 023 хода для 10 дисков и **1 048 575 для 20**. Это принципиально
экспоненциальная задача: никакая хитрость не ускорит её, потому что сам ответ состоит из `2ⁿ - 1` шагов.

## 4. От рекурсии к циклу

Любую рекурсию можно превратить в цикл. Для `fib` цикл был естественным. Для рекурсии по *вложенной* структуре вы держите собственный
**стек** оставшейся работы (Урок 07.5) вместо стека вызовов:

```rust
/// A number or a list of nested things: `[1, [2, 3], [[4], 5]]`.
pub enum Nested {
    Number(i64),
    List(Vec<Nested>),
}

/// The sum of all numbers, by recursion. The depth of the recursion is the depth of the nesting.
pub fn sum_nested(item: &Nested) -> i64 {
    match item {
        Nested::Number(n) => *n,
        Nested::List(items) => items.iter().map(sum_nested).sum(),
    }
}

/// The same, with our own stack instead of the call stack.
pub fn sum_nested_with_stack(item: &Nested) -> i64 {
    let mut to_visit = vec![item];
    let mut total = 0;
    while let Some(current) = to_visit.pop() {
        match current {
            Nested::Number(n) => total += n,
            Nested::List(items) => to_visit.extend(items.iter()),
        }
    }
    total
}
```

**Что происходит:**

1. `Nested` — это число или список значений `Nested`: рекурсивный тип (Урок 05.9; косвенность обеспечивает `Vec`).
2. `sum_nested` следует типу: число даёт само себя, список рекурсивно суммирует свои элементы. Её глубина равна глубине вложенности.
3. `sum_nested_with_stack` использует `Vec` как стек: достаём элемент; если это число — прибавляем; если список — кладём в стек все его
   элементы. Стек лежит в **куче** и растёт, пока хватает памяти, поэтому глубокая вложенность не может переполнить стек вызовов.
   Порядок обхода другой, но сумме это безразлично.

Что выбрать? Рекурсию — когда глубина мала и известна (сбалансированное дерево, деление пополам) и код становится яснее. Цикл с
собственным стеком — когда глубина может быть большой или приходит из внешнего ввода.

## 5. Тесты

Добавьте внизу `src/recursion.rs`:

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use crate::testutil::Rng;

    #[test]
    fn factorial_values_and_overflow() {
        assert_eq!(factorial(0), Some(1));
        assert_eq!(factorial(5), Some(120));
        assert_eq!(factorial(20), Some(2_432_902_008_176_640_000));
        assert_eq!(factorial(21), None); // 21! does not fit into a u64
    }

    #[test]
    fn gcd_examples() {
        assert_eq!(gcd(48, 18), 6);
        assert_eq!(gcd(17, 5), 1);
        assert_eq!(gcd(0, 9), 9);
        assert_eq!(gcd(9, 0), 9);
    }

    #[test]
    fn both_sums_match_the_iterator_sum() {
        let mut rng = Rng::new(4);
        for _ in 0..200 {
            let items: Vec<i64> = (0..rng.below(50))
                .map(|_| rng.below(1000) as i64 - 500)
                .collect();
            let expected: i64 = items.iter().sum();
            assert_eq!(sum_recursive(&items), expected);
            assert_eq!(sum_halves(&items), expected);
        }
    }

    #[test]
    fn fibonacci_versions_agree_and_calls_grow_like_fib() {
        for n in 0..=25 {
            assert_eq!(fib_recursive(n), fib_iterative(n), "n = {n}");
            // the number of calls is 2 * fib(n + 1) - 1
            assert_eq!(fib_calls(n), 2 * fib_iterative(n + 1) - 1, "n = {n}");
        }
        assert_eq!(fib_iterative(50), 12_586_269_025);
    }

    #[test]
    fn hanoi_makes_legal_moves_and_two_to_the_n_minus_one_of_them() {
        for n in 0..=10u32 {
            let mut moves = Vec::new();
            hanoi(n, 'A', 'C', 'B', &mut moves);
            assert_eq!(moves.len(), (1usize << n) - 1);

            // play the moves on three stacks of disks; a bigger disk must never lie on a smaller one
            let mut pegs: [Vec<u32>; 3] = [(1..=n).rev().collect(), Vec::new(), Vec::new()];
            let index = |peg: char| (peg as u8 - b'A') as usize;
            for (from, to) in moves {
                let disk = pegs[index(from)].pop().expect("the source peg has a disk");
                assert!(pegs[index(to)].last().is_none_or(|top| *top > disk));
                pegs[index(to)].push(disk);
            }
            assert_eq!(pegs[2].len(), n as usize); // all disks are on peg C
        }
    }

    fn random_nested(rng: &mut Rng, depth: u32) -> Nested {
        if depth == 0 || rng.below(3) == 0 {
            Nested::Number(rng.below(100) as i64)
        } else {
            let count = rng.below(4);
            Nested::List((0..count).map(|_| random_nested(rng, depth - 1)).collect())
        }
    }

    #[test]
    fn nested_sums_agree() {
        let mut rng = Rng::new(12);
        for _ in 0..300 {
            let tree = random_nested(&mut rng, 6);
            assert_eq!(sum_nested(&tree), sum_nested_with_stack(&tree));
        }
        let fixed = Nested::List(vec![
            Nested::Number(1),
            Nested::List(vec![Nested::Number(2), Nested::Number(3)]),
            Nested::List(vec![
                Nested::List(vec![Nested::Number(4)]),
                Nested::Number(5),
            ]),
        ]);
        assert_eq!(sum_nested(&fixed), 15);
        assert_eq!(sum_nested_with_stack(&fixed), 15);
    }
}
```

**Что происходит:**

1. Тест факториала включает предел переполнения: `20!` помещается в `u64`, `21!` — нет.
2. `both_sums_match_the_iterator_sum` сравнивает обе рекурсивные суммы с `iter().sum()` на 200 случайных срезах.
3. Тест Фибоначчи проверяет, что рекурсивная версия и версия с циклом совпадают для `n` от 0 до 25 и что число вызовов подчиняется
   формуле `2 * fib(n + 1) - 1`. Это хороший пример тестирования результата *анализа*, а не только ответа.
4. Тест Ханоя проверяет число `2ⁿ - 1`, а затем **разыгрывает ходы** на трёх стеках, проверяя, что больший диск никогда не кладётся
   на меньший и что все диски оказываются на стержне `C`. Тест, который моделирует, сильнее того, что лишь считает.
5. Последний тест сравнивает рекурсивную сумму и сумму на стеке на 300 случайных вложенных структурах.

```text
running 6 tests
test recursion::tests::gcd_examples ... ok
test recursion::tests::factorial_values_and_overflow ... ok
test recursion::tests::hanoi_makes_legal_moves_and_two_to_the_n_minus_one_of_them ... ok
test recursion::tests::both_sums_match_the_iterator_sum ... ok
test recursion::tests::nested_sums_agree ... ok
test recursion::tests::fibonacci_versions_agree_and_calls_grow_like_fib ... ok

test result: ok. 6 passed; 0 failed; 0 ignored; 0 measured; 20 filtered out; finished in 0.00s
```

## 6. Измеряем

`benches/recursion.rs` сравнивает две версии Фибоначчи при `n = 30`. Зарегистрируйте его в `Cargo.toml` как в Уроке 07.2
(`name = "recursion"`, `harness = false`).

```rust
use criterion::{Criterion, criterion_group, criterion_main};
use p07_algorithms::recursion::{fib_iterative, fib_recursive};
use std::hint::black_box;

fn bench_fib(c: &mut Criterion) {
    let mut group = c.benchmark_group("fibonacci(30)");
    group.bench_function("recursive", |b| b.iter(|| fib_recursive(black_box(30))));
    group.bench_function("loop", |b| b.iter(|| fib_iterative(black_box(30))));
    group.finish();
}

criterion_group!(benches, bench_fib);
criterion_main!(benches);
```

Запустите `cargo bench --bench recursion -- --warm-up-time 1 --measurement-time 3`:

```text
fibonacci(30)/recursive time:   [1.4337 ms 1.4422 ms 1.4523 ms]
fibonacci(30)/loop      time:   [8.5653 ns 8.6005 ns 8.6363 ns]
```

(Apple M4 Pro; **ваши числа будут другими**.) Рекурсивная версия занимает 1,44 мс (2,7 миллиона вызовов); цикл — 8,6 нс. Цикл
примерно **в 170 000 раз быстрее** при том же ответе. Big-O предсказала направление (`1.6ⁿ` против `n`); бенчмарк показывает размер.

## 7. Когда использовать рекурсию

- **Подходит:** данные рекурсивны (деревья, вложенные списки, выражения, каталоги) и неглубоки или сбалансированы; задача естественно
  делится («разделяй и властвуй»); перебор с возвратом (попробовать, откатить, попробовать снова).
- **Не подходит:** глубина достигает `n` при большом `n` (используйте цикл); одни и те же подзадачи повторяются (используйте
  мемоизацию, Урок 07.16, или цикл).
- **Всегда:** знайте базовый случай, следите, чтобы каждый вызов приближал к нему, и знайте глубину.

## Частые ошибки

**1. Нет базового случая или вызов не уменьшает задачу.** `fn f(n: u64) -> u64 { f(n - 1) }` никогда не останавливается (а `n - 1` при
0 паникует в отладочной сборке раньше, чем переполнится стек).

**2. Повторное вычисление одного и того же** (наивный Фибоначчи). Если рекурсия вызывает себя дважды на пересекающихся задачах,
посчитайте вызовы, прежде чем ей доверять.

**3. Глубокая рекурсия по вводу, который вы не контролируете.** См. примечание о безопасности в разделе 1.

## Попробуйте сами

1. Наберите `recursion.rs` сами. Запустите `cargo test recursion`, `cargo clippy` и `cargo fmt`.
2. Запустите `e07_trace` для `factorial(6)`. Предскажите максимальный отступ до запуска.
3. Запустите `e07_calls` с `fib_calls(40)`. Сколько это длится? (Не поднимайтесь сильно выше.)
4. Запустите `e07_overflow` в отладочной и в релизной сборке. Что отличается?
5. Зафиксируйте: `git add code/p07-algorithms && git commit -m "feat(p07): recursion examples"`.

## Упражнения

### Упражнение 1 — Время и глубина (легко)

Для каждой функции с размером входа `n` назовите время и максимальную глубину рекурсии: (а) `factorial(n)`; (б) `sum_recursive` на срезе
из `n` элементов; (в) `sum_halves`; (г) `fib_recursive(n)`; (д) `hanoi(n, ..)`.

<details>
<summary>Подсказка</summary>

Глубина — самая длинная цепочка вызовов, ждущих друг друга. Время — общее число вызовов (умноженное на работу в одном вызове).

</details>

<details>
<summary>Ответ</summary>

(а) время `O(n)`, глубина `n`. (б) время `O(n)`, глубина `n`. (в) время `O(n)`, глубина около `log n`. (г) время экспоненциальное,
около `O(1.6ⁿ)`; глубина `n`. (д) время `O(2ⁿ)`, глубина `n`.

</details>

### Упражнение 2 — Двоичные цифры (средне)

Напишите `to_binary(n: u32) -> String` рекурсией: двоичные цифры `n / 2`, затем последняя цифра `n`. Напишите тест, который сравнивает её
с `format!("{n:b}")` для всех `n` меньше 2 000 и для `u32::MAX`. Каковы её время и глубина?

<details>
<summary>Подсказка</summary>

Базовый случай — `n < 2`: текст — просто `"0"` или `"1"`. Иначе рекурсивно вызовите для `n / 2` и допишите в конец `'0'` или `'1'`.

</details>

<details>
<summary>Ответ</summary>

```rust
/// The binary digits of `n` as text, by recursion: the digits of `n / 2`, then the last digit.
pub fn to_binary(n: u32) -> String {
    if n < 2 {
        n.to_string()
    } else {
        let mut digits = to_binary(n / 2);
        digits.push(if n.is_multiple_of(2) { '0' } else { '1' });
        digits
    }
}
```

```rust
    #[test]
    fn to_binary_matches_the_standard_formatting() {
        for n in 0..2000u32 {
            assert_eq!(to_binary(n), format!("{n:b}"));
        }
        assert_eq!(to_binary(u32::MAX), "1".repeat(32));
    }
```

`n` делится пополам при каждом вызове, поэтому глубина около `log₂ n` (не больше 32 для `u32`), а время — `O(log n)` вызовов
(построение строки добавляет небольшую стоимость на цифру).

</details>

### Упражнение 3 — Все подмножества (сложно)

Напишите `subsets<T: Clone>(items: &[T]) -> Vec<Vec<T>>`, которая возвращает все `2ⁿ` подмножеств элементов (включая пустое)
рекурсией по первому элементу. Проверьте тестом, что число равно `2ⁿ` и что все подмножества различны, для `n` от 0 до 10. Назовите
время и глубину.

<details>
<summary>Подсказка</summary>

Каждое подмножество либо не содержит первый элемент (это подмножества остатка), либо содержит (это подмножества остатка, к каждому
из которых добавлен первый элемент). Базовый случай: у пустого среза одно подмножество — пустое.

</details>

<details>
<summary>Ответ</summary>

```rust
/// Every subset of `items` (2^n of them), by recursion on the first item:
/// the subsets without it, and the same subsets with it added at the front.
pub fn subsets<T: Clone>(items: &[T]) -> Vec<Vec<T>> {
    match items.split_first() {
        None => vec![Vec::new()],
        Some((first, rest)) => {
            let without = subsets(rest);
            let mut result = without.clone();
            for mut subset in without {
                subset.insert(0, first.clone());
                result.push(subset);
            }
            result
        }
    }
}
```

```rust
    #[test]
    fn subsets_has_two_to_the_n_distinct_subsets() {
        for n in 0..=10usize {
            let items: Vec<usize> = (0..n).collect();
            let all = subsets(&items);
            assert_eq!(all.len(), 1 << n);
            let distinct: HashSet<Vec<usize>> = all.into_iter().collect();
            assert_eq!(distinct.len(), 1 << n);
        }
    }
```

Глубина — `n`. Время — `O(n * 2ⁿ)`: `2ⁿ` подмножеств по нескольку (до `n`) элементов. Как и в Ханое, сам результат экспоненциально
велик, поэтому экспоненциальное время неизбежно. (Тест использует `HashSet` из Урока 07.2.)

</details>

## Вопросы для самопроверки

1. Какие две части должны быть у каждой рекурсивной функции?

<details>
<summary>Ответ</summary>

Базовый случай, который возвращает результат без дальнейшего вызова, и рекурсивный случай, который вызывает функцию с меньшим входом,
чтобы вызовы приближались к базовому случаю.

</details>

2. Какова глубина рекурсии у `sum_recursive` и у `sum_halves` на миллионе элементов?

<details>
<summary>Ответ</summary>

Около миллиона у `sum_recursive` (по кадру на элемент, так что стек, скорее всего, переполнится) и около 20 у `sum_halves` (срез
делится пополам при каждом вызове).

</details>

3. Почему наивный `fib_recursive(40)` работает так долго, хотя его глубина всего 40?

<details>
<summary>Ответ</summary>

Он делает около 330 миллионов вызовов: каждый вызов делает ещё два, и они повторяют одни и те же меньшие подзадачи. Время
экспоненциальное, хотя глубина стека мала.

</details>

4. Ваша программа читает вложенные данные из сети и обрабатывает их рекурсивно. Что может пойти не так и что делать?

<details>
<summary>Ответ</summary>

Атакующий может прислать очень глубокую вложенность и переполнить стек, что аварийно завершит процесс. Ограничьте глубину (и верните
ошибку) или обрабатывайте данные циклом с собственным стеком в куче.

</details>

## Итоги

- Рекурсия: базовый случай плюс рекурсивный случай с меньшим входом. Стек вызовов хранит по кадру на каждый ждущий вызов; глубина —
  цена по памяти.
- Стек невелик. Глубокая рекурсия его переполняет, а Rust не обещает оптимизацию хвостовых вызовов, так что на оптимизатор
  рассчитывать нельзя.
- Деление пополам («разделяй и властвуй») даёт глубину `O(log n)`; отщепление по одному элементу — глубину `O(n)`.
- Рекурсия, решающая пересекающиеся подзадачи (наивный Фибоначчи), экспоненциальна; цикл или мемоизация это исправляют.
- Некоторые задачи экспоненциальны по природе (Ханой, все подмножества): сам результат такой большой.
- Любую рекурсию можно превратить в цикл с собственным стеком в куче, что безопасно для глубокого или враждебного ввода.
- Тестируйте рекурсивный код против простого эталона, тестируйте базовый случай и тестируйте числа, которые предсказывает ваш анализ.

## Источники и что почитать

- [The Rust Programming Language — 3.5 Control Flow](https://doc.rust-lang.org/book/ch03-05-control-flow.html) — циклы как альтернатива рекурсии.
- [std::primitive::slice — split_first and split_at](https://doc.rust-lang.org/std/primitive.slice.html#method.split_first) — два метода среза, на которых построена рекурсия по срезам.
- [Option::and_then](https://doc.rust-lang.org/std/option/enum.Option.html#method.and_then) — цепочка шагов, каждый из которых может вернуть `None`.
- [The Rust Programming Language — 15.1 Using Box<T> to Point to Data on the Heap](https://doc.rust-lang.org/book/ch15-01-box.html) — рекурсивные типы, как у `Nested`.
- [std::thread::Builder::stack_size](https://doc.rust-lang.org/std/thread/struct.Builder.html#method.stack_size) — как дать потоку больший стек, когда глубокая рекурсия действительно нужна.

---

[🏠 Начало](../START-HERE.md) · [← 07.6 Как работают хеш-таблицы](06-hash-tables.md) · [📋 Этап 07](README.md) · [07.8 Простые алгоритмы сортировки →](08-simple-sorting-algorithms.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
