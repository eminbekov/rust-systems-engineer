<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 07.14 Кратчайшие пути и топологическая сортировка](14-shortest-paths-and-topological-sort.md) · [📋 Этап 07](README.md) · [07.16 Динамическое программирование →](16-dynamic-programming.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/07-algorithms-and-data-structures/15-two-pointers-and-sliding-window.md) <!-- nav -->

# Урок 07.15 — Два указателя и скользящее окно

> **Этап 07 — Алгоритмы и структуры данных** · Урок 15 из 20 · ⏱ около 2,5 часа (2 занятия)

## Что вы узнаете

- Приём *два указателя*: два индекса идут по данным вместо двух вложенных циклов.
- Две его формы: указатели идут навстречу друг другу и указатели идут в одном направлении.
- *Скользящее окно*: блок соседних элементов с накапливаемой суммой.
- Почему такие циклы работают за O(n), а не за O(n²), и когда приём **не** работает.
- Как проверить хитрый алгоритм по медленному и заведомо верному на тысячах входов.

## Перед началом

- Вы прошли [Урок 07.14 — Кратчайшие пути и топологическая сортировка](14-shortest-paths-and-topological-sort.md).
- Вы знаете нотацию «O большое» и умеете считать работу цикла ([Урок 07.1](01-algorithms-and-big-o.md), [Урок 07.2](02-analyzing-time-and-space.md)).
- У вас есть библиотечный пакет `code/p07-algorithms` из предыдущих уроков (один модуль на тему, тесты рядом с кодом, бенчмарки в `benches/`). Этот урок добавляет модуль `src/two_pointers.rs` (и строку `pub mod two_pointers;` в `src/lib.rs`) и один бенчмарк. `criterion` был представлен в [Уроке 05.12](../05-rust-intermediate/12-benchmarking-basics.md).

## Новые слова

| Слово | Значение |
|---|---|
| **два указателя** (two pointers) | Приём с двумя индексами (позициями) в одних и тех же данных. Каждый шаг двигает один из них, назад они не возвращаются, поэтому работа — O(n). |
| **указатель** (pointer, в этом уроке) | Индекс: обычное число `usize`, которое говорит «я на позиции 3». Это не адрес в памяти и не ссылка Rust. |
| **скользящее окно** (sliding window) | Блок соседних элементов, заданный левым и правым индексом. Правый индекс идёт вперёд, чтобы добавить элемент, левый идёт вперёд, чтобы убрать элемент. |
| **на месте** (in place) | Изменение полученных данных с несколькими дополнительными переменными (O(1) дополнительной памяти) вместо создания нового вектора. |
| **инвариант** (invariant) | Утверждение о цикле, которое верно в начале каждого шага, например «всё левее `write` уже готово». Инварианты помогают убедиться, что цикл верен. |
| **амортизированная** (amortized) | Стоимость, усреднённая по всему запуску: один шаг может быть дорогим, но в сумме затраты малы. |
| **псевдослучайные** (pseudo-random) | Числа, которые выглядят случайными, но получены по фиксированной формуле из начального числа (*seed*). Один и тот же seed даёт одни и те же числа, поэтому упавший тест можно повторить. |

## 1. Указатели с двух концов

**Задача.** Срез чисел **отсортирован** по возрастанию. Найдите две *разные* позиции, числа в которых в сумме дают `target`.

Перебор всех пар требует двух вложенных циклов: O(n²). Для 100 000 чисел это около пяти миллиардов пар. Порядок сортировки позволяет сделать намного лучше. Поставьте `left` на первый (наименьший) элемент, а `right` — на последний (наибольший) и посмотрите на сумму:

- **Слишком мала:** число на `left` — наименьшее из оставшихся, и даже с самым большим партнёром сумма мала. В ответ оно попасть не может. Отбросьте его: `left += 1`.
- **Слишком велика:** число на `right` не может быть в ответе по той же причине. Отбросьте его: `right -= 1`.
- **Равна:** готово.

Каждый шаг навсегда отбрасывает одно число, поэтому шагов не больше `n - 1`.

```text
target = 13          index:  0  1  2  3  4   5
                     value:  1  3  4  6  9  11

step 1   L=0 R=5     1 + 11 = 12   too small  -> L moves right
         [1] 3  4  6  9 [11]
step 2   L=1 R=5     3 + 11 = 14   too big    -> R moves left
          1 [3] 4  6  9 [11]
step 3   L=1 R=4     3 +  9 = 12   too small  -> L moves right
          1 [3] 4  6 [9] 11
step 4   L=2 R=4     4 +  9 = 13   found!
          1  3 [4] 6 [9] 11
```

```rust
/// Looks for two different positions `i < j` of a SORTED slice with
/// `sorted[i] + sorted[j] == target`. Time O(n), extra space O(1).
pub fn pair_with_sum(sorted: &[i32], target: i32) -> Option<(usize, usize)> {
    if sorted.len() < 2 {
        return None;
    }
    let target = i64::from(target);
    let mut left = 0;
    let mut right = sorted.len() - 1;
    while left < right {
        // i64, so that two big i32 numbers cannot overflow
        let sum = i64::from(sorted[left]) + i64::from(sorted[right]);
        if sum == target {
            return Some((left, right));
        } else if sum < target {
            left += 1; // we need a bigger sum: take a bigger left number
        } else {
            right -= 1; // we need a smaller sum: take a smaller right number
        }
    }
    None
}
```

**Что происходит:**

1. В срезе меньше двух элементов пары нет. Ранний возврат защищает и `sorted.len() - 1` в следующих строках (см. «Частые ошибки»).
2. `left` и `right` — индексы `usize` на двух концах. Цикл идёт, пока `left < right`, поэтому указатели никогда не оказываются на одном элементе (нам нужны две *разные* позиции).
3. Сумма считается в `i64`: два значения `i32` около `i32::MAX` переполнили бы `i32`, а отладочная сборка запаниковала бы.
4. Слишком мала — двигаем `left`, слишком велика — двигаем `right`, равна — возвращаем пару. Если указатели встретились, все числа были отброшены без совпадения: `None`.

> 🔒 **Безопасность:** арифметика над числами, которые выбрали не вы (содержимое файла, ввод пользователя), может переполниться. Расширение до большего типа через `i64::from` или `checked_add` почти ничего не стоит и убирает сбой или неверный ответ.

**Время и память.** Расстояние между указателями начинается с `n - 1` и уменьшается на единицу за шаг: не больше `n - 1` шагов по O(1) работы, то есть **время O(n)**. Хранятся только два индекса: **память O(1)**. Медленная версия, которой не нужен отсортированный вход, такова:

```rust
/// The same question, answered with two nested loops. Time O(n * n).
pub fn pair_with_sum_slow(values: &[i32], target: i32) -> Option<(usize, usize)> {
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

> 🧪 **Тестирование:** хитрому алгоритму не верят, пока он не совпал с медленным на тысячах маленьких случайных входов. Случайные числа даёт *псевдослучайный* генератор `Rng` из `src/testutil.rs` ([Урок 07.3](03-arrays-and-vectors-under-the-hood.md)). Тестам этого и следующих уроков нужны числа из диапазона, который может включать отрицательные значения, поэтому сначала добавьте один метод внутрь `impl Rng` в `src/testutil.rs`, под `below`:

```rust
    /// A number from `lo` to `hi`, both included. `lo` must not be bigger than `hi`.
    pub fn range(&mut self, lo: i64, hi: i64) -> i64 {
        lo + (self.next_u64() % (hi - lo + 1) as u64) as i64
    }
```

Затем поместите это в конец `src/two_pointers.rs`. Один и тот же `Rng` используют все тесты этого урока.

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use crate::testutil::Rng;

    #[test]
    fn pair_small_examples() {
        let v = [1, 3, 4, 6, 9, 11];
        assert_eq!(pair_with_sum(&v, 13), Some((2, 4)));
        assert_eq!(pair_with_sum(&v, 100), None);
        assert_eq!(pair_with_sum(&[], 1), None);
        assert_eq!(pair_with_sum(&[5], 10), None);
        assert_eq!(pair_with_sum(&[i32::MAX, i32::MAX], i32::MAX), None);
    }

    #[test]
    fn pair_agrees_with_slow_version() {
        let mut rng = Rng::new(12345);
        for _ in 0..2000 {
            let len = rng.range(0, 12) as usize;
            let mut v: Vec<i32> = (0..len).map(|_| rng.range(-20, 20) as i32).collect();
            v.sort();
            let target = rng.range(-30, 30) as i32;
            let fast = pair_with_sum(&v, target);
            let slow = pair_with_sum_slow(&v, target);
            assert_eq!(
                fast.is_some(),
                slow.is_some(),
                "v = {v:?}, target = {target}"
            );
            if let Some((i, j)) = fast {
                assert!(i < j);
                assert_eq!(v[i] + v[j], target);
            }
        }
    }
}
```

**Что происходит:**

1. `range(lo, hi)` берёт остаток от деления большого случайного числа на размер диапазона `hi - lo + 1` и прибавляет `lo`, поэтому результат — число от `lo` до `hi`. `Rng::new(12345)` запускает генератор с seed 12345: один и тот же seed всегда даёт одну и ту же последовательность, поэтому сбой можно повторить.
2. `pair_small_examples` проверяет картинку выше и границы: пустой срез, один элемент и случай переполнения.
3. `pair_agrees_with_slow_version` строит 2000 случайных отсортированных векторов из 0–12 чисел и спрашивает обе функции. Если пар несколько, функции могут вернуть *разные* пары, поэтому мы сравниваем только «есть ли ответ?», а затем проверяем, что пара быстрой функции действительно даёт нужную сумму. Текст после аргументов `assert_eq!` показывает входные данные упавшего случая.

```bash
cargo test two_pointers
```

```text
running 2 tests
test two_pointers::tests::pair_small_examples ... ok
test two_pointers::tests::pair_agrees_with_slow_version ... ok

test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 63 filtered out; finished in 0.00s
```

(Порядок строк может отличаться: тесты идут параллельно. Отфильтрованы 63 теста из основного текста Уроков 07.1–07.14; с тестами из упражнений ваше число будет больше. Строки `Running unittests` и `Doc-tests` здесь и ниже опущены.)

## 2. Читатель и писатель

Во второй форме оба указателя начинают с начала, и один бежит впереди: **читатель** (`read`) смотрит на каждый элемент один раз, **писатель** (`write`) отмечает конец уже готовой части. Классическая задача — удалить повторяющихся соседей из отсортированного вектора *на месте*: `[1, 1, 2, 2, 2, 3]` превращается в `[1, 2, 3]`.

```text
v = [1, 1, 2, 2, 2, 3]       W = write (last unique item), R = read

R=1   v[R]=1 equals v[W]=1   -> skip
R=2   v[R]=2 differs         -> W=1, swap(W, R):   [1  2] 1  2  2  3
R=3   v[R]=2 equals v[W]=2   -> skip
R=4   v[R]=2 equals v[W]=2   -> skip
R=5   v[R]=3 differs         -> W=2, swap(W, R):   [1  2  3] 2  2  1
end   truncate to W+1 = 3 items                    [1, 2, 3]
```

```rust
/// Removes repeated neighbours from a SORTED vector, in place.
/// `write` is the last place of the unique part, `read` scans the rest.
pub fn dedup_sorted<T: PartialEq>(v: &mut Vec<T>) {
    if v.is_empty() {
        return;
    }
    let mut write = 0;
    for read in 1..v.len() {
        if v[read] != v[write] {
            write += 1;
            v.swap(write, read);
        }
    }
    v.truncate(write + 1);
}
```

**Что происходит:**

1. `write` начинается с 0, потому что первый элемент всегда уникален.
2. Для каждого `read`, начиная с 1: если элемент отличается от последнего уникального, уникальная часть вырастает на одно место, и новый элемент переносится на это место обменом. `swap` не требует `Clone` и нового вектора; `T: PartialEq` нужен только для `!=`.
3. `truncate(write + 1)` отрезает остатки после уникальной части.
4. **Инвариант:** в начале каждого шага `v[..=write]` содержит уникальные элементы, виденные до сих пор, по порядку. Цикл сохраняет это утверждение верным; когда `read` доходит до конца, утверждение и есть ответ.

Время O(n), дополнительная память O(1). Судья здесь — стандартная библиотека; добавьте в `mod tests`:

```rust
    #[test]
    fn dedup_agrees_with_std() {
        let mut rng = Rng::new(777);
        for _ in 0..1000 {
            let len = rng.range(0, 15) as usize;
            let mut mine: Vec<i64> = (0..len).map(|_| rng.range(0, 6)).collect();
            mine.sort();
            let mut expected = mine.clone();
            expected.dedup();
            dedup_sorted(&mut mine);
            assert_eq!(mine, expected);
        }
    }
```

> 💡 **Совет:** в настоящем коде используйте `Vec::dedup` (удаляет *подряд идущие* равные элементы, поэтому работает после сортировки) или `Vec::retain` (оставляет элементы, для которых замыкание вернуло `true`). Если написать цикл самому, вы освоите схему «читатель и писатель», которая вернётся в шаге разбиения быстрой сортировки ([Урок 07.9](09-merge-sort-and-quicksort.md)) и при слиянии отсортированных списков.

## 3. Скользящее окно фиксированного размера

**Задача.** Даны числа и размер `k`. Найдите наибольшую сумму `k` соседних чисел.

Если считать сумму каждого блока заново, это k сложений для каждого из n блоков: O(n · k). Но соседние блоки делят `k - 1` чисел. Когда блок сдвигается на шаг вправо, справа одно число **входит**, слева одно **выходит**, поэтому сумму можно *обновлять*:

```text
values = [2, 1, 5, 1, 3, 2],   k = 3

[2 1 5] 1 3 2      sum = 8
 2 [1 5 1] 3 2     sum = 8 + 1 (enters) - 2 (leaves) = 7
 2  1 [5 1 3] 2    sum = 7 + 3 - 1 = 9      <- best
 2  1  5 [1 3 2]   sum = 9 + 2 - 5 = 6
```

```rust
/// The largest sum of `k` neighbouring numbers. Time O(n), space O(1).
/// Returns `None` when `k` is 0 or bigger than the slice.
pub fn max_sum_of_k(values: &[i64], k: usize) -> Option<i64> {
    if k == 0 || k > values.len() {
        return None;
    }
    let mut window: i64 = values[..k].iter().sum();
    let mut best = window;
    for i in k..values.len() {
        window += values[i]; // the number that enters on the right
        window -= values[i - k]; // the number that leaves on the left
        best = best.max(window);
    }
    Some(best)
}

/// The same answer, recomputing every window. Time O(n * k).
pub fn max_sum_of_k_slow(values: &[i64], k: usize) -> Option<i64> {
    if k == 0 || k > values.len() {
        return None;
    }
    values.windows(k).map(|w| w.iter().sum::<i64>()).max()
}
```

**Что происходит:**

1. `k == 0` не имеет смысла, а `k > values.len()` не помещается: оба случая дают `None`. Проверка защищает и `windows(k)`, который паникует при `k == 0`.
2. Первое окно суммируется один раз (k сложений). Для каждого следующего `i` число `values[i]` входит, а `values[i - k]` выходит: две операции за шаг, чему бы ни было равно `k`. Всего `k + (n - k)` = O(n); память O(1).
3. Медленная версия использует стандартный метод `slice::windows(k)`, который выдаёт каждый блок из `k` соседей как под-срез. Для небольших данных это хорошо, и это судья нашего теста:

```rust
    #[test]
    fn max_sum_agrees_with_slow_version() {
        let mut rng = Rng::new(99);
        for _ in 0..1000 {
            let len = rng.range(0, 20) as usize;
            let v: Vec<i64> = (0..len).map(|_| rng.range(-50, 50)).collect();
            let k = rng.range(0, 22) as usize;
            assert_eq!(
                max_sum_of_k(&v, k),
                max_sum_of_k_slow(&v, k),
                "v = {v:?}, k = {k}"
            );
        }
    }
```

`k` доходит до 22 на векторах до 19 чисел, поэтому тест заодно покрывает `k == 0` и «слишком большое k».

> ⚡ **Производительность:** насколько O(n) быстрее O(n · k)? Измерьте. В `Cargo.toml` убедитесь, что `criterion = "0.8"` есть в `[dev-dependencies]`, и добавьте:

```toml
[[bench]]
name = "windows"
harness = false
```

Создайте `benches/windows.rs`:

```rust
use criterion::{Criterion, criterion_group, criterion_main};
use p07_algorithms::two_pointers::{max_sum_of_k, max_sum_of_k_slow};
use std::hint::black_box;

fn bench_windows(c: &mut Criterion) {
    let values: Vec<i64> = (0..50_000).map(|i| (i * 7919 % 1000) - 500).collect();
    let k = 500;
    let mut group = c.benchmark_group("max_sum_of_k");
    group.bench_function("slow", |b| {
        b.iter(|| max_sum_of_k_slow(black_box(&values), black_box(k)))
    });
    group.bench_function("sliding window", |b| {
        b.iter(|| max_sum_of_k(black_box(&values), black_box(k)))
    });
    group.finish();
}

criterion_group!(benches, bench_windows);
criterion_main!(benches);
```

```bash
cargo bench --bench windows -- --warm-up-time 1 --measurement-time 3
```

```text
max_sum_of_k/slow       time:   [1.7174 ms 1.7445 ms 1.7778 ms]
max_sum_of_k/sliding window
                        time:   [26.614 µs 27.514 µs 28.580 µs]
```

(Criterion печатает ещё строки о ходе работы и отчёт о выбросах; они опущены. Эти числа получены на Mac с Apple M4. На каждом компьютере числа разные, но соотношение должно быть похожим.)

**Что происходит:** `--bench windows` запускает только этот файл бенчмарка; без него Cargo передаёт `--warm-up-time` ещё и программе модульных тестов, которая не знает такого параметра. Для 50 000 чисел и `k = 500` скользящему окну нужно около 27 мкс, а медленной версии — около 1,7 мс: примерно в 60 раз быстрее. Медленная версия быстрее, чем следует из её 25 миллионов сложений, потому что компилятор превращает внутреннюю сумму в очень быстрые векторные инструкции. Разрыв растёт с `k`: скользящее окно от `k` не зависит вовсе.

## 4. Скользящее окно, меняющее размер

**Задача.** Срез чисел, которые **не отрицательны**, и `target`. Найдите длину *кратчайшего* блока соседних чисел, сумма которого **не меньше** `target`.

Правый край идёт вперёд и добавляет числа. Как только сумма достигла цели, запишите длину, затем **сжимайте** слева, пока сумма всё ещё достигает цели, записывая каждую длину. Потом продолжайте с правым краем.

```text
values = [2, 3, 1, 2, 4, 3],  target = 7

R=3   window [2 3 1 2]  sum 8 >= 7: length 4; drop 2 -> [3 1 2] sum 6, stop
R=4   window [3 1 2 4]  sum 10 >= 7: length 4; drop 3 -> [1 2 4] sum 7 >= 7:
                        length 3; drop 1 -> [2 4] sum 6, stop
R=5   window [2 4 3]    sum 9 >= 7: length 3; drop 2 -> [4 3] sum 7 >= 7:
                        length 2 (best); drop 4 -> [3] sum 3, stop
```

```rust
/// The length of the shortest block of neighbouring numbers whose sum is at
/// least `target`. The numbers are unsigned, so the sum only grows when the
/// window grows. Time O(n), space O(1).
pub fn shortest_window_with_sum_at_least(values: &[u32], target: u64) -> Option<usize> {
    if target == 0 {
        return Some(0); // the empty block is enough
    }
    let mut left = 0;
    let mut sum: u64 = 0;
    let mut best: Option<usize> = None;
    for right in 0..values.len() {
        sum += u64::from(values[right]);
        while sum >= target {
            let len = right - left + 1;
            best = Some(best.map_or(len, |b| b.min(len)));
            sum -= u64::from(values[left]);
            left += 1;
        }
    }
    best
}
```

**Что происходит:**

1. Для `target == 0` подходит пустой блок, поэтому ответ — длина 0. При положительной цели цикл `while` ниже всегда останавливается: когда окно становится пустым, `sum` равна 0, то есть меньше цели.
2. Цикл `for` двигает `right` по каждой позиции и добавляет это число к накапливаемой `sum`.
3. Цикл `while` сжимает окно, пока оно ещё достаточно большое. Каждый раз он записывает `right - left + 1`, вычитает `values[left]` из суммы и двигает `left` вперёд.
4. `best.map_or(len, |b| b.min(len))` значит: «если `best` равно `None`, возьми `len`, иначе меньшее из двух».

**Действительно ли это O(n)?** Цикл в цикле выглядит как O(n²). Посчитайте вместо этого *перемещения*: `right` двигается ровно n раз. `left` двигается только вперёд, никогда не назад и никогда дальше `right`, поэтому за весь запуск он двигается не более n раз. Внутренний цикл не начинается с нуля, он продолжает с места, где остановился `left`. Всего не более 2n перемещений: O(n). Это *амортизированное* рассуждение: на одних внешних шагах сжатие большое, на других его нет совсем, а в сумме затраты малы.

**Когда это ломается?** Сжатие опирается на один факт: удаление числа никогда не увеличивает сумму, а добавление никогда не уменьшает. Это верно только для неотрицательных чисел. С отрицательными числами решения «достаточно велика, значит сжимаем» и «слишком мала, значит растим» перестают быть безопасными, и указатели отбросят окна, которые нужно было проверить. Поэтому функция принимает `u32`, а не `i32`: тип фиксирует требование.

Судья снова строится на `windows`: ответ — первая длина, для которой какой-нибудь блок достигает цели.

```rust
    /// The judge: the first length for which some block reaches the target.
    fn shortest_window_slow(values: &[u32], target: u64) -> Option<usize> {
        (1..=values.len()).find(|&len| {
            values
                .windows(len)
                .any(|w| w.iter().map(|&x| u64::from(x)).sum::<u64>() >= target)
        })
    }

    #[test]
    fn shortest_window_examples() {
        assert_eq!(
            shortest_window_with_sum_at_least(&[2, 3, 1, 2, 4, 3], 7),
            Some(2)
        );
        assert_eq!(shortest_window_with_sum_at_least(&[1, 1, 1], 10), None);
        assert_eq!(shortest_window_with_sum_at_least(&[], 1), None);
        assert_eq!(shortest_window_with_sum_at_least(&[9], 9), Some(1));
        assert_eq!(shortest_window_with_sum_at_least(&[1, 2], 0), Some(0));
    }

    #[test]
    fn shortest_window_agrees_with_slow_version() {
        let mut rng = Rng::new(4242);
        for _ in 0..2000 {
            let len = rng.range(0, 15) as usize;
            let v: Vec<u32> = (0..len).map(|_| rng.range(0, 9) as u32).collect();
            let target = rng.range(1, 40) as u64;
            assert_eq!(
                shortest_window_with_sum_at_least(&v, target),
                shortest_window_slow(&v, target),
                "v = {v:?}, target = {target}"
            );
        }
    }
```

Случайные числа включают нули — хитрый случай для окон. Добавьте тест из Упражнения 1 и запустите `cargo test two_pointers`: проходят семь тестов.

```text
running 7 tests
test two_pointers::tests::max_sum_examples ... ok
test two_pointers::tests::pair_small_examples ... ok
test two_pointers::tests::shortest_window_examples ... ok
test two_pointers::tests::max_sum_agrees_with_slow_version ... ok
test two_pointers::tests::dedup_agrees_with_std ... ok
test two_pointers::tests::shortest_window_agrees_with_slow_version ... ok
test two_pointers::tests::pair_agrees_with_slow_version ... ok

test result: ok. 7 passed; 0 failed; 0 ignored; 0 measured; 63 filtered out; finished in 0.00s
```

## 5. Рецепт и что предлагает стандартная библиотека

| Вы видите | Попробуйте |
|---|---|
| **Отсортированный** срез и вопрос о парах или тройках | Указатели с двух концов (раздел 1). |
| «Удалить / переместить / сгруппировать элементы **на месте**» | Читатель и писатель (раздел 2). |
| «Лучший блок из **k** соседей» | Окно фиксированного размера с накапливаемым значением (раздел 3). |
| «Кратчайший / длиннейший блок», где добавление элемента может только помочь (или только навредить) | Окно переменного размера (раздел 4). |

Чтобы построить решение, запишите: что означает каждый указатель, инвариант, когда двигается каждый указатель и **почему указатель никогда не должен возвращаться назад**. Если на последний вопрос ответить нельзя, приём, скорее всего, не подходит.

| Задача | Стандартный инструмент |
|---|---|
| Все перекрывающиеся блоки из `k` соседей | `slice.windows(k)` |
| Блоки, которые не перекрываются | `slice.chunks(k)` |
| Удалить подряд идущие дубликаты / оставить элементы по правилу | `Vec::dedup`, `Vec::retain` |
| Развернуть на месте | `slice.reverse()` |
| Найти позицию в отсортированном срезе | `binary_search`, `partition_point` ([Урок 07.10](10-binary-search.md)) |

Используйте их, когда задача именно такая. Пишите собственный цикл с указателями, когда окну нужно накапливаемое значение (сумма, счётчик), которое `windows` пересчитывал бы снова и снова.

## Частые ошибки

**1. `len() - 1` для пустого среза.** `len()` — это `usize`, он не бывает отрицательным. Без проверки в начале `pair_with_sum`:

```rust
fn pair_with_sum(sorted: &[i32], target: i32) -> Option<(usize, usize)> {
    let mut left = 0;
    let mut right = sorted.len() - 1; // no check that the slice is not empty
    while left < right {
        let sum = sorted[left] + sorted[right];
        if sum == target {
            return Some((left, right));
        } else if sum < target {
            left += 1;
        } else {
            right -= 1;
        }
    }
    None
}

fn main() {
    let empty: Vec<i32> = Vec::new();
    println!("{:?}", pair_with_sum(&empty, 10));
}
```

```text
thread 'main' (4623313) panicked at src/main.rs:3:21:
attempt to subtract with overflow
```

**Что это значит:** `0 - 1` не помещается в `usize`, поэтому отладочная сборка паникует. (Релизная сборка «обернулась» бы до огромного числа и сломалась позже, более странным образом.) **Исправление:** сначала проверьте `len() < 2` или используйте `checked_sub`. Всегда тестируйте пустой вход и вход из одного элемента.

**2. Два указателя на неотсортированных данных.** Если проверку добавить, та же функция компилируется, запускается и даёт неверный ответ:

```rust
fn main() {
    let unsorted = [5, 1, 9, 2];
    // 1 + 9 = 10 exists, but the data is not sorted
    println!("{:?}", pair_with_sum(&unsorted, 10));
}
```

```text
None
```

**Что это значит:** указатели двигались по правилам, но правило «это число слишком мало, чтобы быть в ответе» требует отсортированного порядка, а ни один тип не говорит «отсортировано». **Исправление:** сначала отсортируйте (O(n log n)) либо задокументируйте требование и добавьте в начало `debug_assert!(sorted.is_sorted());`, чтобы тесты поймали ошибку.

**3. Ветка, которая не двигает ни один указатель, или окно на данных с отрицательными числами.** Если один путь в теле цикла не меняет ни один указатель, цикл никогда не кончится: *каждый* путь должен продвигаться. А отрицательные числа ломают логику «сжимать слева» из раздела 4, поэтому проверьте окно на отрицательных входах, прежде чем ему доверять.

## Попробуйте сами

1. Запустите `cargo test two_pointers` и прочитайте названия семи тестов. Затем нарочно сломайте `pair_with_sum` (замените `<` на `>`) и посмотрите, какие тесты падают первыми и что печатает сообщение случайного теста.
2. Поменяйте seed у `Rng` в одном тесте (например, `Rng::new(12345)` на `Rng::new(1)`). Тесты по-прежнему проходят, но на других входах. Почему для теста фиксированный seed лучше по-настоящему случайного?
3. Запустите проверку качества и сделайте один коммит:

```bash
cargo fmt && cargo clippy --all-targets -- -D warnings && cargo test
git add code/p07-algorithms
git commit -m "feat(p07): two pointers and sliding window"
```

## Упражнения

### Упражнение 1 — Тест для `max_sum_of_k` (легко)

Напишите тест `max_sum_examples` для `max_sum_of_k`. Покройте `k`, равное 1, `k`, равное длине, `k` больше длины, `k == 0` и срез только из отрицательных чисел. Начните с `[2, 1, 5, 1, 3, 2]`.

<details>
<summary>Подсказка</summary>

Сначала вычислите ожидаемые ответы вручную. Для `[2, 1, 5, 1, 3, 2]`: `k = 1` даёт самое большое одиночное число (5); `k = 6` даёт сумму всех (14). Для `[-4, -1, -3]` и `k = 2` блоки дают `-5` и `-4`, поэтому лучший — `-4`: результат может быть отрицательным.

</details>

<details>
<summary>Ответ</summary>

```rust
    #[test]
    fn max_sum_examples() {
        let v = [2, 1, 5, 1, 3, 2];
        assert_eq!(max_sum_of_k(&v, 3), Some(9));
        assert_eq!(max_sum_of_k(&v, 1), Some(5));
        assert_eq!(max_sum_of_k(&v, 6), Some(14));
        assert_eq!(max_sum_of_k(&v, 7), None);
        assert_eq!(max_sum_of_k(&v, 0), None);
        assert_eq!(max_sum_of_k(&[-4, -1, -3], 2), Some(-4));
    }
```

Почему это работает: тест покрывает обычный случай, обе границы `k`, недопустимые значения и отрицательные суммы. Частая ошибка — начать `best` с 0, тогда для среза из одних отрицательных чисел вернулось бы неверное `Some(0)`; этот тест её ловит.

</details>

### Упражнение 2 — Назовите сложность (средне)

Назовите временную сложность (через `n = values.len()`) и дополнительную память каждой функции, одним предложением объяснив причину.

```rust
fn a(values: &[i32]) -> usize {
    let mut count = 0;
    for i in 0..values.len() {
        for j in 0..values.len() {
            if values[i] < values[j] {
                count += 1;
            }
        }
    }
    count
}

fn b(values: &[i32]) -> usize {
    let mut left = 0;
    let mut right = values.len();
    let mut steps = 0;
    while left < right {
        if values[left] % 2 == 0 {
            left += 1;
        } else {
            right -= 1;
        }
        steps += 1;
    }
    steps
}

fn c(values: &[i32], k: usize) -> i32 {
    let mut best = i32::MIN;
    for start in 0..=values.len().saturating_sub(k) {
        let sum: i32 = values[start..(start + k).min(values.len())].iter().sum();
        best = best.max(sum);
    }
    best
}
```

<details>
<summary>Подсказка</summary>

Для `a`: посчитайте циклы. Для `b`: каков промежуток `right - left` в начале и насколько он сокращается за шаг? Для `c`: сколько начальных позиций и какой длины каждая внутренняя сумма?

</details>

<details>
<summary>Ответ</summary>

- `a`: время **O(n²)**, память O(1). Два вложенных цикла по всем `n` элементам: n · n сравнений.
- `b`: время **O(n)**, память O(1). Промежуток начинается с `n` и сокращается ровно на единицу за каждый шаг (либо растёт `left`, либо уменьшается `right`), поэтому цикл выполняется `n` раз.
- `c`: время **O(n · k)**, память O(1). Около `n - k + 1` начальных позиций, и каждая внутренняя `sum` складывает до `k` чисел. Скользящее окно из раздела 3 убирает множитель `k`.

</details>

### Упражнение 3 — Три числа с заданной суммой (сложно)

Напишите `pub fn has_triple_with_sum(values: &[i32], target: i64) -> bool`, которая говорит, дают ли три числа на трёх разных позициях в сумме `target`. Используйте сортировку и два указателя, назовите временную сложность и проверьте функцию тестом против тройного цикла на случайных входах.

<details>
<summary>Подсказка</summary>

Отсортируйте копию входа. Для каждой позиции `first` два других числа должны дать в сумме `target - values[first]` и лежать **после** `first`: это раздел 1 для части среза после `first`.

</details>

<details>
<summary>Ответ</summary>

```rust
pub fn has_triple_with_sum(values: &[i32], target: i64) -> bool {
    let mut v = values.to_vec();
    v.sort();
    if v.len() < 3 {
        return false;
    }
    for first in 0..v.len() - 2 {
        let need = target - i64::from(v[first]);
        let mut left = first + 1;
        let mut right = v.len() - 1;
        while left < right {
            let sum = i64::from(v[left]) + i64::from(v[right]);
            if sum == need {
                return true;
            } else if sum < need {
                left += 1;
            } else {
                right -= 1;
            }
        }
    }
    false
}
```

Тест (в модуле `tests`):

```rust
    fn has_triple_slow(v: &[i32], target: i64) -> bool {
        let n = v.len();
        (0..n).any(|i| {
            (i + 1..n).any(|j| {
                (j + 1..n).any(|k| i64::from(v[i]) + i64::from(v[j]) + i64::from(v[k]) == target)
            })
        })
    }

    #[test]
    fn triple_agrees_with_slow() {
        let mut rng = Rng::new(9);
        for _ in 0..1000 {
            let len = rng.range(0, 8) as usize;
            let v: Vec<i32> = (0..len).map(|_| rng.range(-10, 10) as i32).collect();
            let target = rng.range(-15, 15);
            assert_eq!(
                has_triple_with_sum(&v, target),
                has_triple_slow(&v, target),
                "{v:?} {target}"
            );
        }
    }
```

Время: сортировка O(n log n), затем n проходов по O(n): всего **O(n²)**, что лучше тройного цикла O(n³). Дополнительная память O(n) для отсортированной копии. Тест проходит.

</details>

## Вопросы для самопроверки

1. Почему `pair_with_sum` нужен отсортированный срез?

<details>
<summary>Ответ</summary>

Решение «это число слишком мало (или слишком велико), чтобы быть в ответе» опирается на порядок. В отсортированном срезе, если `values[left] + values[right]` слишком мала, то `values[left]` с любым другим кандидатом (все не больше `values[right]`) тоже даёт слишком малую сумму, поэтому `left` можно отбросить навсегда. В неотсортированном срезе это рассуждение ложно (Частая ошибка 2).

</details>

2. Сжимающий цикл `while` стоит внутри цикла `for` в `shortest_window_with_sum_at_least`. Почему общее время всё равно O(n)?

<details>
<summary>Ответ</summary>

Указатели никогда не возвращаются назад. `right` идёт вперёд n раз; `left` за весь запуск идёт вперёд не более n раз, потому что никогда не обгоняет `right`. Внутренний цикл продолжает с места, где был `left`, поэтому всего не более 2n перемещений указателей. Стоимость амортизирована по всему запуску.

</details>

3. Каковы время и память `max_sum_of_k` по сравнению с версией, которая суммирует каждый блок `windows(k)`?

<details>
<summary>Ответ</summary>

Скользящее окно: время O(n), память O(1). Версия с `windows(k)`: время O(n · k), память O(1). Ответ тот же; первая версия обновляет сумму одним сложением и одним вычитанием за шаг.

</details>

4. Почему `shortest_window_with_sum_at_least` принимает `&[u32]`, а не `&[i32]`?

<details>
<summary>Ответ</summary>

Сжатие верно, только если добавление элементов никогда не уменьшает сумму, а удаление никогда не увеличивает. Это так только для неотрицательных чисел. Беззнаковый тип делает требование частью сигнатуры.

</details>

## Итоги

- Два указателя заменяют второй цикл вторым индексом. Поскольку указатели не возвращаются назад, работа — O(n), обычно с O(1) дополнительной памяти.
- С двух концов: на отсортированном срезе слишком малая сумма двигает левый указатель, слишком большая — правый. В одном направлении: читатель и писатель, для фильтрации на месте.
- Скользящее окно хранит накапливаемое значение: прибавьте входящий элемент, вычтите выходящий. Окно переменного размера сжимается слева, пока выполняется его условие.
- Цикл в цикле — не всегда O(n²): считайте общее число перемещений указателей (амортизированный анализ).
- Приём зависит от свойства (отсортированные данные, неотрицательные числа). Зафиксируйте его в типе или doc-комментарии и тестируйте границы: пустой вход, один элемент, слишком большое `k`.
- Тестируйте быстрый алгоритм против медленного и простого на случайных входах с фиксированным seed. В настоящем коде предпочитайте `windows`, `chunks`, `dedup`, `retain` и `reverse`, когда они решают задачу.

## Источники и что почитать

- [The Rust Standard Library — slice](https://doc.rust-lang.org/std/primitive.slice.html) — `windows`, `chunks`, `swap`, `reverse`, `is_sorted`, `partition_point`.
- [The Rust Standard Library — Vec](https://doc.rust-lang.org/std/vec/struct.Vec.html) — `dedup`, `retain`, `truncate`.
- [The Rust Standard Library — Option](https://doc.rust-lang.org/std/option/enum.Option.html) — `map_or`, использован для лучшей длины.
- [The Rust Programming Language — 13.2 Processing a Series of Items with Iterators](https://doc.rust-lang.org/book/ch13-02-iterators.html) — официальная глава об итераторах, которые выдают `windows` и `chunks`.
- [Criterion.rs User Guide](https://bheisler.github.io/criterion.rs/book/) — библиотека бенчмарков, использованная здесь.

---

[🏠 Начало](../START-HERE.md) · [← 07.14 Кратчайшие пути и топологическая сортировка](14-shortest-paths-and-topological-sort.md) · [📋 Этап 07](README.md) · [07.16 Динамическое программирование →](16-dynamic-programming.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
