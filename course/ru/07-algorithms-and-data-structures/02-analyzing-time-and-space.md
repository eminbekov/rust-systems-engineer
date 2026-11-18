<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 07.1 Алгоритмы и нотация „O большое“](01-algorithms-and-big-o.md) · [📋 Этап 07](README.md) · [07.3 Массивы и векторы изнутри →](03-arrays-and-vectors-under-the-hood.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/07-algorithms-and-data-structures/02-analyzing-time-and-space.md) <!-- nav -->

# Урок 07.2 — Анализ времени и памяти

> **Этап 07 — Алгоритмы и структуры данных** · Урок 2 из 20 · ⏱ около 135 минут

## Что вы узнаете

- Рецепт для нахождения временной сложности кода: какие «формы» кода сколько стоят и как заметить скрытый цикл.
- Чем отличаются лучшая, худшая, средняя и амортизированная стоимость.
- Как посчитать память, которая нужна алгоритму (сложность по памяти).
- Компромисс между временем и памятью — на настоящем примере и настоящем бенчмарке.
- Почему «более медленный» алгоритм может выигрывать на малых входах.

## Перед началом

- Вы прошли [Урок 07.1 — Алгоритмы и нотация „O большое“](01-algorithms-and-big-o.md) и у вас есть пакет `code/p07-algorithms`.
- Вы знаете `HashSet` ([Урок 04.9](../04-structs-enums-collections-errors/09-hash-maps.md)) и основы `criterion` ([Урок 05.12](../05-rust-intermediate/12-benchmarking-basics.md)).

## Новые слова

| Слово | Значение |
|---|---|
| **средний случай** (average case) | Стоимость для типичного входа, усреднённая по многим входам. |
| **амортизированная стоимость** (amortized cost) | Стоимость одной операции, если сложить длинную серию операций. Редкая дорогая операция «оплачивается» многими дешёвыми. |
| **дополнительная память** (auxiliary space) | Память, которая нужна алгоритму сверх его входа. Когда говорят «сложность по памяти», обычно считают именно её. |
| **на месте** (in place) | Работа внутри памяти самого входа, с несколькими дополнительными переменными (`O(1)` дополнительной памяти). |
| **компромисс** (trade-off) | Получить одно хорошее, заплатив другим. Здесь: скорость за память. |
| **предвычисление** (precomputation) | Работа, сделанная один раз до вопросов, чтобы каждый вопрос стал дешёвым. |
| **префиксные суммы** (prefix sums) | Вектор, у которого элемент `k` — сумма первых `k` элементов входа. Он отвечает на вопрос «чему равна сумма на этом отрезке?» одним вычитанием. |
| **точка пересечения** (crossover point) | Размер входа, при котором один алгоритм перестаёт быть быстрее, а другой выходит вперёд. |

## 1. Рецепт подсчёта времени

Считать каждый шаг, как в Уроке 07.1, не нужно. Смотрите на **форму** кода:

| Форма кода | Время |
|---|---|
| несколько простых операторов, индекс `v[i]` | `O(1)` |
| цикл по всем `n` элементам, простое тело | `O(n)` |
| цикл внутри цикла, оба примерно по `n` элементов | `O(n²)` |
| цикл, который делит (или удваивает) свой счётчик | `O(log n)` |
| один блок **после** другого | стоимости **складываются**: побеждает самая большая |
| один блок **внутри** другого | стоимости **перемножаются** |
| `if` / `else` | более дорогая ветка (худший случай) |
| вызов функции | стоимость этой функции |

И одна ловушка: **вызов метода может скрывать цикл.** `v.contains(&x)` выглядит как один шаг, но проходит по
всему вектору: это `O(n)`. `v.insert(0, x)` сдвигает каждый элемент: `O(n)`. `v.iter().sum()` — `O(n)`. Поставьте
такой вызов внутрь цикла по `n` элементов — и получится `O(n²)`, хотя второго `for` вы не писали. Вот функция со
всеми четырьмя видами частей (`examples/e02_shapes.rs`):

```rust
fn shapes(v: &[i64], other: &[i64]) -> i64 {
    let first = v[0]; // O(1)
    let total: i64 = v.iter().sum(); // O(n)

    let mut bigger_pairs = 0; // O(n²): n times n comparisons
    for a in v {
        for b in v {
            if a < b {
                bigger_pairs += 1;
            }
        }
    }

    let mut shared = 0; // looks like O(n), but `contains` is a hidden loop
    for x in v {
        if other.contains(x) {
            shared += 1;
        }
    }

    first + total + bigger_pairs + shared
}

fn main() {
    let v = [4, 8, 15, 16, 23, 42];
    let other = [8, 16, 32];
    println!("{}", shapes(&v, &other));
}
```

**Что происходит:** `v[0]` — `O(1)`, `v.iter().sum()` — `O(n)`. Двойной цикл сравнивает каждый элемент с каждым:
`O(n²)`. Последний цикл выглядит как `O(n)`, но `other.contains(x)` — это цикл по `other`, так что при
`m = other.len()` получается `O(n * m)`. Части идут одна за другой, поэтому стоимости складываются:
`O(1) + O(n) + O(n²) + O(n * m)`. Оставляем самые большие части: `O(n² + n * m)`, то есть `O(n²)`, когда `other`
примерно такой же длины, как `v`. (Программа печатает `129`; вывод не важен, важно читать код именно так.)

## 2. Лучший, худший, средний, амортизированный

Одна и та же функция может стоить по-разному на разных входах одного и того же размера `n`.

- **Худший случай**: самый дорогой вход. Это значение Big-O по умолчанию (Правило 3 из Урока 07.1).
- **Лучший случай**: самый дешёвый вход. Сам по себе редко полезен, но «лучший случай `O(n)`» для алгоритма
  сортировки говорит, что отсортированный вход обходится дёшево, а это иногда важно (Урок 07.8).
- **Средний случай**: типичный вход. Для линейного поиска с целью в случайной позиции это около `n / 2`
  сравнений, то есть по-прежнему `O(n)`: постоянный множитель `1/2` отбрасывается.
- **Амортизированная**: стоимость одной операции в длинной серии операций. Классический пример — `Vec::push`.
  Большинство `push` просто записывают один элемент, это дёшево. Иногда вектор заполнен: он выделяет больший блок
  и копирует все `n` элементов, это стоит `O(n)`. Но вектор каждый раз удваивает ёмкость (Урок 04.7), поэтому такие
  дорогие `push` редки: за `n` вызовов копирование в сумме даёт меньше `2n` перемещений элементов. Значит, один
  `push` стоит `O(1)` *амортизированно*. Урок 07.3 доказывает это счётчиком.

> ⚠️ **Частая ошибка:** «Амортизированная `O(1)`» не значит, что каждый отдельный `push` быстр. Один `push` может
> стоить `O(n)`. Если программа обязана отвечать на каждый запрос в жёсткий срок (кадр игры, торговая система),
> редкая медленная операция может быть важна, и место резервируют заранее через `Vec::with_capacity`.

## 3. Сложность по памяти

Время — не единственная цена. **Сложность по памяти** говорит, как память растёт вместе с `n`. Мы считаем
**дополнительную память**: ту, которую алгоритм выделяет сам, а не вход, который он получает (им уже владеет
вызывающий код).

- `largest` из Урока 07.1 использует несколько переменных: `O(1)` дополнительной памяти. Он работает **на месте**.
- `v.iter().map(...).collect::<Vec<_>>()` строит новый вектор по одному элементу на каждый элемент входа: `O(n)`.
- Алгоритм, который строит таблицу `n` на `n` ячеек, использует `O(n²)` памяти. При `n = 100 000` это десять
  миллиардов ячеек: программу убьют задолго до того, как время станет важным.

Чтобы почувствовать числа, запустите `examples/e02_space.rs`:

```rust
use std::collections::HashSet;
use std::mem::size_of;

fn main() {
    let n = 1000;
    let items: Vec<i64> = (0..n).collect();
    let set: HashSet<i64> = items.iter().copied().collect();

    println!("one i64 takes {} bytes", size_of::<i64>());
    println!(
        "Vec<i64> with {} items: about {} bytes of item data",
        items.len(),
        items.capacity() * size_of::<i64>()
    );
    println!(
        "HashSet<i64> with {} items has room for {} items",
        set.len(),
        set.capacity()
    );
}
```

```text
one i64 takes 8 bytes
Vec<i64> with 1000 items: about 8000 bytes of item data
HashSet<i64> with 1000 items has room for 1792 items
```

Вектору из 1 000 чисел `i64` нужно около 8 000 байт, потому что каждое число занимает 8 байт. У множества
`HashSet` с теми же 1 000 числами есть место для 1 792 элементов (множество нарочно держит часть ячеек пустыми,
чтобы поиск оставался быстрым; Урок 07.6 объяснит, почему) и ему нужно заметно больше памяти, чем вектору. Точные
значения зависят от реализации; вывод такой: «`O(n)` памяти» всё равно может быть большим `n`, умноженным на
большую постоянную.

## 4. Компромисс: время против памяти

Вопрос: *есть ли в срезе пара элементов, сумма которых равна `target`?* Например, в `[8, 3, 5, 2]` при цели `10`
ответ «да» (`8 + 2`). Первое решение сравнивает каждую пару:

```rust
/// Is there a pair of items (at two different positions) whose sum is `target`?
/// Compares every pair: O(n²) time, O(1) extra memory.
pub fn pair_sum_quadratic(items: &[i64], target: i64) -> bool {
    for i in 0..items.len() {
        for j in (i + 1)..items.len() {
            if items[i].checked_add(items[j]) == Some(target) {
                return true;
            }
        }
    }
    false
}
```

Это `O(n²)` времени и `O(1)` памяти. (`checked_add` возвращает `None` вместо переполнения, см. примечание о
безопасности ниже.) Второе решение меняет память на скорость. Для каждого элемента `x` оно спрашивает: *видел ли я
уже число `target - x`?* Хеш-множество отвечает за `O(1)` в среднем:

```rust
/// The same question. Remembers the items already seen in a hash set:
/// O(n) time on average, O(n) extra memory.
pub fn pair_sum_hashset(items: &[i64], target: i64) -> bool {
    let mut seen = HashSet::new();
    for &x in items {
        let found = target
            .checked_sub(x)
            .is_some_and(|needed| seen.contains(&needed));
        if found {
            return true;
        }
        seen.insert(x);
    }
    false
}
```

**Что происходит:**

1. `seen` хранит все элементы до текущего. В начале оно пусто.
2. Для элемента `x` нужный партнёр — `target - x`. Если `seen` его содержит, пара найдена, возвращаем `true`.
3. Иначе запоминаем `x` и идём дальше. Каждый элемент рассматривается один раз, а каждый поиск и вставка — `O(1)`
   в среднем, поэтому время `O(n)`. Множество может вырасти до `n` элементов, поэтому память `O(n)`.
4. Проверка идёт *до* вставки, поэтому элемент никогда не образует пару сам с собой. `[5]` с целью `10` — это
   `false`, а `[5, 5]` — `true`.

| | время | дополнительная память |
|---|---|---|
| `pair_sum_quadratic` | `O(n²)` | `O(1)` |
| `pair_sum_hashset` | `O(n)` в среднем | `O(n)` |

Бесплатных обедов не бывает: одна версия дёшева по памяти и дорога по времени, у другой наоборот. Что лучше —
зависит от `n`, от объёма памяти и от того, как часто вы задаёте вопрос. У хеш-множества есть и худший случай
`O(n)` на операцию, когда много ключей сталкиваются (Урок 07.6), поэтому мы и говорим «в среднем».

> 🔒 **Безопасность:** `a + b` для `i64` паникует в отладочной сборке (а в релизной молча «заворачивается»), если
> результат не помещается. Если числа приходят снаружи, неверный ответ или падение — это поверхность атаки. Поэтому
> в коде `checked_add` и `checked_sub`, а в тесте — `i64::MAX` и `i64::MIN` (Урок 02.2).

Тесты для этих функций идут в конце следующего раздела, после последней части модуля.

## 5. Предвычисление: заплатить один раз, отвечать быстро

Ещё один обмен памяти на время: *сделать работу один раз, чтобы каждый следующий вопрос был дешёвым.* Допустим,
у нас длинный срез и мы много раз спрашиваем: «чему равна сумма `items[from..to]`?» (`from..to` — диапазон: он
включает `from` и исключает `to`.) Вычислять каждую сумму с нуля стоит `O(to - from)` на вопрос. С **префиксными
суммами** мы один раз тратим `O(n)` времени и `O(n)` памяти, а каждый вопрос стоит `O(1)`:

```rust
/// The sum of `items[from..to]`, computed from scratch: O(to - from) per question.
pub fn range_sum_naive(items: &[i64], from: usize, to: usize) -> i64 {
    items[from..to].iter().sum()
}

/// Prepared once in O(n) time and O(n) memory; then every range sum costs O(1).
pub struct PrefixSums {
    // sums[k] is the sum of the first k items, so sums[0] == 0 and sums.len() == n + 1
    sums: Vec<i64>,
}

impl PrefixSums {
    pub fn new(items: &[i64]) -> PrefixSums {
        let mut sums = Vec::with_capacity(items.len() + 1);
        sums.push(0);
        for &x in items {
            let last = sums[sums.len() - 1];
            sums.push(last + x);
        }
        PrefixSums { sums }
    }

    /// The sum of `items[from..to]`.
    pub fn range_sum(&self, from: usize, to: usize) -> i64 {
        self.sums[to] - self.sums[from]
    }
}
```

**Что происходит:**

1. `sums[k]` — сумма первых `k` элементов. `sums[0]` равно 0 (элементов нет), поэтому в векторе `n + 1` значений.
2. `new` строит его за один проход: каждое значение — это предыдущее плюс ещё один элемент.
3. Сумма `items[from..to]` — это (первые `to` элементов) минус (первые `from` элементов): одно вычитание.

```text
items:  [ 3, -1,  4,  1, -5 ]
sums:   [ 0,  3,  2,  6,  7,  2 ]        sums[k] = sum of the first k items
sum of items[1..4] = sums[4] - sums[1] = 7 - 3 = 4      (that is -1 + 4 + 1)
```

Теперь тесты — внизу `src/analysis.rs`. Первый из них исчерпывающий: он перебирает все 256 векторов длины 4 со
значениями от -1 до 2 и каждую цель от -3 до 4 и требует, чтобы две функции поиска пары давали одинаковый ответ.
Второй проверяет правило «две разные позиции», третий использует крайние значения `i64::MAX` и `i64::MIN`, а
последний сравнивает `range_sum` с наивной суммой для **каждого** диапазона среза из 8 элементов.

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn both_pair_sums_agree_on_all_small_inputs() {
        // all 256 vectors of length 4 with items from -1 to 2, and targets from -3 to 4
        for a in -1..=2 {
            for b in -1..=2 {
                for c in -1..=2 {
                    for d in -1..=2 {
                        let v = [a, b, c, d];
                        for target in -3..=4 {
                            assert_eq!(
                                pair_sum_quadratic(&v, target),
                                pair_sum_hashset(&v, target),
                                "items {v:?}, target {target}"
                            );
                        }
                    }
                }
            }
        }
    }

    #[test]
    fn pair_sum_needs_two_different_positions() {
        assert!(!pair_sum_quadratic(&[5], 10)); // 5 + 5 would reuse one item
        assert!(!pair_sum_hashset(&[5], 10));
        assert!(pair_sum_quadratic(&[5, 5], 10));
        assert!(pair_sum_hashset(&[5, 5], 10));
    }

    #[test]
    fn pair_sum_survives_extreme_values() {
        let v = [i64::MAX, i64::MAX, i64::MIN];
        assert!(!pair_sum_quadratic(&v, 0));
        assert!(!pair_sum_hashset(&v, 0));
        assert!(pair_sum_quadratic(&v, -1)); // MAX + MIN
        assert!(pair_sum_hashset(&v, -1));
    }

    #[test]
    fn prefix_sums_match_the_naive_sum_for_every_range() {
        let items = [3, -1, 4, 1, -5, 9, 2, 6];
        let prefix = PrefixSums::new(&items);
        for from in 0..=items.len() {
            for to in from..=items.len() {
                assert_eq!(
                    prefix.range_sum(from, to),
                    range_sum_naive(&items, from, to)
                );
            }
        }
    }
}
```

Запустите `cargo test analysis`:

```text
running 4 tests
test analysis::tests::prefix_sums_match_the_naive_sum_for_every_range ... ok
test analysis::tests::pair_sum_needs_two_different_positions ... ok
test analysis::tests::pair_sum_survives_extreme_values ... ok
test analysis::tests::both_pair_sums_agree_on_all_small_inputs ... ok

test result: ok. 4 passed; 0 failed; 0 ignored; 0 measured; 5 filtered out; finished in 0.00s
```

(Пять тестов из Урока 07.1 отфильтрованы по имени `analysis`.) Используйте префиксные суммы, когда данные не
меняются, а вопросов много. При 100 000 вопросов к 1 000 000 элементов наивный способ делает до 10¹¹ сложений,
а префиксные суммы — 10⁶ плюс 10⁵.

## 6. Измеряем компромисс

Big-O говорит, что `pair_sum_quadratic` проигрывает при большом `n`. Проигрывает ли он при *любом* `n`?
Измерим. Используем `criterion`, как в Уроке 05.12. Добавьте его и зарегистрируйте бенчмарк:

```bash
cargo add --dev criterion@0.8
mkdir benches
```

Добавьте в конец `Cargo.toml`:

```toml
[[bench]]
name = "analysis"
harness = false
```

Создайте `benches/analysis.rs`:

```rust
use criterion::{BenchmarkId, Criterion, criterion_group, criterion_main};
use p07_algorithms::analysis::{pair_sum_hashset, pair_sum_quadratic};
use std::hint::black_box;

fn bench_pair_sum(c: &mut Criterion) {
    let mut group = c.benchmark_group("pair_sum (worst case)");
    for n in [8usize, 64, 512, 4096] {
        // the items are 0, 1, ..., n-1 and the target is -1: there is no pair,
        // so both functions must look at everything
        let items: Vec<i64> = (0..n as i64).collect();
        group.bench_with_input(BenchmarkId::new("quadratic", n), &items, |b, items| {
            b.iter(|| pair_sum_quadratic(black_box(items), black_box(-1)))
        });
        group.bench_with_input(BenchmarkId::new("hashset", n), &items, |b, items| {
            b.iter(|| pair_sum_hashset(black_box(items), black_box(-1)))
        });
    }
    group.finish();
}

criterion_group!(benches, bench_pair_sum);
criterion_main!(benches);
```

**Что происходит:**

1. Для каждого размера `n` вход — это `0, 1, ..., n-1`, а цель — `-1`. Никакая пара неотрицательных чисел не даёт
   в сумме -1, поэтому обе функции должны просмотреть всё: это худший случай.
2. `BenchmarkId::new("quadratic", n)` даёт каждому измерению имя с размером, так что отчёт показывает серию.
3. `black_box` прячет аргументы от оптимизатора (Урок 05.12).

Запустите (короткий прогон, около минуты; имя после `--bench` нужно, потому что обычный механизм запуска тестов
не понял бы параметры `criterion`):

```bash
cargo bench --bench analysis -- --warm-up-time 1 --measurement-time 3
```

```text
pair_sum (worst case)/quadratic/8
                        time:   [12.774 ns 12.833 ns 12.904 ns]
pair_sum (worst case)/hashset/8
                        time:   [250.53 ns 251.07 ns 251.69 ns]
pair_sum (worst case)/quadratic/64
                        time:   [871.47 ns 874.23 ns 877.10 ns]
pair_sum (worst case)/hashset/64
                        time:   [2.2221 µs 2.2289 µs 2.2380 µs]
pair_sum (worst case)/quadratic/512
                        time:   [52.690 µs 53.051 µs 53.540 µs]
pair_sum (worst case)/hashset/512
                        time:   [18.465 µs 18.536 µs 18.616 µs]
pair_sum (worst case)/quadratic/4096
                        time:   [3.2139 ms 3.2285 ms 3.2447 ms]
pair_sum (worst case)/hashset/4096
                        time:   [149.49 µs 149.97 µs 150.47 µs]
```

(Criterion печатает больше строк, например прогресс прогрева и число выбросов; здесь они опущены. Числа получены на
Apple M4 Pro с Rust 1.100; **ваши числа будут другими**, смотрите на отношения.)

**Как это читать:**

1. **Квадратичная версия растёт примерно в 60 раз**, когда `n` растёт в 8 раз (874 нс, 53 мкс, 3,2 мс). Это близко
   к `8² = 64`: отличительный признак `O(n²)`.
2. **Хеш-множество растёт примерно в 8 раз** (2,2 мкс, 18,5 мкс, 150 мкс). Это признак `O(n)`.
3. **При `n = 8` и `n = 64` выигрывает «плохой» алгоритм**: 13 нс против 251 нс. Создание хеш-множества, хеширование
   каждого числа и выделение памяти имеют большую постоянную стоимость. На крошечных входах двойной цикл быстрее.
4. **Точка пересечения** на этом компьютере лежит между 64 и 512. При `n = 4096` хеш-множество быстрее в 21 раз.

> ⚡ **Производительность:** Big-O говорит, что происходит при росте `n`. Она не говорит, где точка пересечения.
> Если ваше `n` всегда мало (скажем, меньше 50), простой алгоритм может быть лучшим. Если `n` может быть большим,
> выигрывает скорость роста. Измеряйте на реалистичных размерах ([Урок 05.12](../05-rust-intermediate/12-benchmarking-basics.md)).

## Частые ошибки

**1. Забытый скрытый цикл.** `for x in &a { if b.contains(x) { ... } }` — это `O(a.len() * b.len())`, а не `O(n)`.
Исправление: сначала положить `b` в `HashSet`; тогда цикл стоит `O(a.len() + b.len())`.

**2. Подсчёт только времени.** Алгоритм, который быстр, но требует `O(n²)` памяти, может убить машину. Всегда
задавайте оба вопроса: сколько времени и сколько памяти.

## Попробуйте сами

1. Добавьте `pub mod analysis;` в `src/lib.rs` и наберите код разделов 4 и 5 в `src/analysis.rs`.
2. Запустите `cargo test`, `cargo clippy` и `cargo fmt`.
3. Добавьте бенчмарк из раздела 6, запустите его и запишите три самых удивительных числа в комментарий в начале
   `benches/analysis.rs`.
4. Зафиксируйте: `git add code/p07-algorithms && git commit -m "feat(p07): analysis module, time/space trade-offs"`.

## Упражнения

### Упражнение 1 — Время и память (легко)

Назовите время и дополнительную память каждого фрагмента кода (в `v` лежит `n` элементов):

```rust
let doubled: Vec<i64> = v.iter().map(|x| x * 2).collect();   // (a)

let mut count = 0;                                             // (b)
for x in &v { if v.contains(&(x + 1)) { count += 1; } }

let mut k = 1;                                                 // (c)
while k < v.len() { k *= 2; }
```

<details>
<summary>Подсказка</summary>

Спросите себя: сколько раз выполняется цикл, сколько стоит каждый проход (ищите скрытые циклы) и что выделяется?

</details>

<details>
<summary>Ответ</summary>

(a) время `O(n)`, память `O(n)`: новый вектор. (b) время `O(n²)`, потому что `contains` — это `O(n)` внутри
цикла по `n`; память `O(1)`. (c) время `O(log n)`: `k` удваивается, пока не достигнет `n`; память `O(1)`.

</details>

### Упражнение 2 — Граничные случаи префиксных сумм (легко)

Напишите тест для `PrefixSums` с пустым срезом, срезом из одного элемента и пустым диапазоном (`from == to`).

<details>
<summary>Подсказка</summary>

Для пустого входа вектор `sums` всё равно содержит одно значение, `[0]`, так что `range_sum(0, 0)` работает и равно 0.

</details>

<details>
<summary>Ответ</summary>

```rust
    #[test]
    fn prefix_sums_edge_cases() {
        let empty = PrefixSums::new(&[]);
        assert_eq!(empty.range_sum(0, 0), 0);
        let one = PrefixSums::new(&[7]);
        assert_eq!(one.range_sum(0, 1), 7);
        assert_eq!(one.range_sum(1, 1), 0); // an empty range
    }
```

Почему это работает: `new(&[])` строит `sums = [0]`; `range_sum(0, 0)` — это `0 - 0`. Для `[7]` получается
`sums = [0, 7]`, а диапазон `1..1` — это `7 - 7 = 0`.

</details>

### Упражнение 3 — Дубликаты за O(n) (средне)

В Уроке 07.1 `has_duplicate_counted` сравнивала все пары. Напишите `has_duplicate_hashset(items: &[i32]) -> bool` с
`HashSet`, проверьте её тестом против `has_duplicate_counted` на всех векторах длины 4 со значениями от 0 до 3 и
назовите время и память обеих.

<details>
<summary>Подсказка</summary>

`HashSet::insert` возвращает `false`, если значение уже было в множестве. Это и есть сигнал дубликата.

</details>

<details>
<summary>Ответ</summary>

```rust
pub fn has_duplicate_hashset(items: &[i32]) -> bool {
    let mut seen = HashSet::new();
    for &x in items {
        if !seen.insert(x) {
            return true; // `insert` returns false when the value was already there
        }
    }
    false
}
```

```rust
    #[test]
    fn hashset_duplicates_agree_with_the_quadratic_version() {
        for a in 0..4 {
            for b in 0..4 {
                for c in 0..4 {
                    for d in 0..4 {
                        let v = [a, b, c, d];
                        assert_eq!(has_duplicate_hashset(&v), has_duplicate_counted(&v).0);
                    }
                }
            }
        }
    }
```

Время `O(n)` в среднем и память `O(n)`; версия с парами — `O(n²)` времени и `O(1)` памяти: тот же компромисс, что и
в разделе 4.

</details>

### Упражнение 4 — Лучшее окно (сложно)

Напишите `best_window_sum(items: &[i64], k: usize) -> Option<i64>`: наибольшую сумму `k` подряд идущих элементов
или `None`, если `k` равно 0 или больше длины. Используйте `PrefixSums`. Назовите время и память. Может ли память
быть `O(1)`?

<details>
<summary>Подсказка</summary>

Окно, начинающееся в `start`, покрывает `start..start + k`. Переберите каждый `start` от 0 до `len - k` и возьмите
наибольший `range_sum`.

</details>

<details>
<summary>Ответ</summary>

```rust
pub fn best_window_sum(items: &[i64], k: usize) -> Option<i64> {
    if k == 0 || k > items.len() {
        return None;
    }
    let prefix = PrefixSums::new(items);
    let mut best = prefix.range_sum(0, k);
    for start in 1..=(items.len() - k) {
        best = best.max(prefix.range_sum(start, start + k));
    }
    Some(best)
}
```

Время `O(n)` (один проход, чтобы построить суммы, затем `n - k + 1` запросов за константное время) и память `O(n)`.
Да, память `O(1)` возможна: держите бегущую сумму, прибавляйте элемент, входящий в окно, и вычитайте уходящий. Это
приём *скользящего окна* из Урока 07.15.

</details>

## Вопросы для самопроверки

1. Чему равно время цикла по `n` элементам, который в каждом проходе вызывает `v.contains(&x)`?

<details>
<summary>Ответ</summary>

`O(n²)` (точнее, `O(n * v.len())`): `contains` — это цикл по вектору, спрятанный внутри цикла.

</details>

2. Что значит «амортизированная `O(1)`» для `Vec::push`?

<details>
<summary>Ответ</summary>

За длинную серию `push` средняя стоимость одного вызова постоянна. Отдельный `push` может быть медленным (когда
вектору надо вырасти и скопировать элементы), но это случается так редко, что среднее не меняется.

</details>

3. Почему `pair_sum_quadratic` может обогнать `pair_sum_hashset` на 8 элементах?

<details>
<summary>Ответ</summary>

Big-O прячет постоянные множители. Построение хеш-множества (выделение памяти, хеширование) в расчёте на шаг стоит намного
дороже, чем сравнение двух чисел, поэтому на крошечных входах выигрывает дешёвый двойной цикл. Хеш-множество
выигрывает только после точки пересечения.

</details>

## Итоги

- Читайте код по форме: циклы перемножаются, последовательные блоки складываются, `if` стоит как худшая ветка,
  вызов стоит как вызываемая функция. Следите за скрытыми циклами (`contains`, `insert(0, ..)`, `sum`).
- Случаи: худший (по умолчанию), лучший, средний и амортизированный (стоимость операции в длинной серии).
- Сложность по памяти считает дополнительную память; «на месте» значит `O(1)` её.
- Время и память обмениваются друг на друга: хеш-множество превратило `O(n²)` в `O(n)` ценой `O(n)` памяти;
  префиксные суммы один раз тратят `O(n)`, чтобы запросы стали `O(1)`.
- На малых входах может выиграть простой алгоритм. Точку пересечения нужно измерять.
- Быстрые алгоритмы проверяйте против медленных, очевидно правильных, в том числе на крайних значениях
  (переполнение).

## Источники и что почитать

- [std::collections — Performance](https://doc.rust-lang.org/std/collections/index.html#performance) — стоимость каждой операции стандартных коллекций.
- [Vec — Guarantees](https://doc.rust-lang.org/std/vec/struct.Vec.html#guarantees) — что документация обещает про ёмкость и рост вектора (амортизированный `push` за `O(1)`).
- [HashSet::insert](https://doc.rust-lang.org/std/collections/struct.HashSet.html#method.insert) — возвращает, было ли значение вставлено впервые.
- [i64::checked_add](https://doc.rust-lang.org/std/primitive.i64.html#method.checked_add) — сложение с защитой от переполнения.
- [Criterion.rs documentation — comparing functions and input sizes](https://bheisler.github.io/criterion.rs/book/user_guide/comparing_functions.html) — как работают `bench_with_input` и `BenchmarkId`.

---

[🏠 Начало](../START-HERE.md) · [← 07.1 Алгоритмы и нотация „O большое“](01-algorithms-and-big-o.md) · [📋 Этап 07](README.md) · [07.3 Массивы и векторы изнутри →](03-arrays-and-vectors-under-the-hood.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
