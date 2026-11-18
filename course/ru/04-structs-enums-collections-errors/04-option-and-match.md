<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 04.3 Перечисления](03-enums.md) · [📋 Этап 04](README.md) · [04.5 if let, let else и образцы →](05-if-let-let-else-and-patterns.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/04-structs-enums-collections-errors/04-option-and-match.md) <!-- nav -->

# Урок 04.4 — Option и match

> **Этап 04 — Структуры, перечисления, коллекции и ошибки** · Урок 4 из 12 · ⏱ около 3 часов (2 занятия)

## Что вы узнаете

- Почему «значения может не быть» — большой источник ошибок и как Rust решает это с помощью `Option<T>`.
- Как создать `Option` и как безопасно достать из него значение.
- Самые полезные методы `Option`: `unwrap_or`, `map`, `and_then`, `filter`, `ok_or`, `as_ref` и другие.
- `match` подробно: диапазоны, несколько образцов, охранные условия, привязки `@`, кортежи и вложенные образцы.
- Почему `unwrap` опасен в настоящих программах и что использовать вместо него.

## Перед началом

- Вы прошли [Урок 04.3 — Перечисления](03-enums.md). Вы знаете варианты с данными и первую форму `match`.
- Вы помните, как перемещается `String` ([Урок 03.3 — Перемещение, копирование и клонирование](../03-ownership-and-borrowing/03-move-copy-and-clone.md)), и что такое ссылки `&` ([Урок 03.4](../03-ownership-and-borrowing/04-references-and-borrowing.md)).

## Новые слова

| Слово | Значение |
|---|---|
| **`Option<T>`** | Перечисление с двумя вариантами: `Some(value)` (значение типа `T` есть) и `None` (значения нет). |
| **`T`** (параметр типа) | Заместитель «любого типа». `Option<i32>` означает «необязательный `i32`»; `Option<String>` — «необязательная `String`». Свои собственные вы напишете в [Уроке 05.1 — Обобщённые типы](../05-rust-intermediate/01-generics.md). |
| **null** | Особая метка «нет значения», которую используют многие языки. Её случайное использование приводит к падению программ. В Rust null нет. |
| **`unwrap`** | Метод, возвращающий значение внутри `Some` или **паникующий** (останавливающий программу с ошибкой) на `None`. |
| **паника** (panic) | Намеренное аварийное завершение программы (или текущего потока) с сообщением об ошибке. |
| **`expect`** | Как `unwrap`, но сообщение паники задаёте вы. |
| **замыкание** (closure) | Небольшая безымянная функция, записанная как `\|x\| x + 1`. Как следует вы изучите их в [Уроке 05.4 — Замыкания](../05-rust-intermediate/04-closures.md); пока `\|w\| w * 2` можно читать как «функция, которая принимает `w` и возвращает `w * 2`». |
| **охранное условие** (guard) | Дополнительное условие `if` у ветки `match`: `x if x > 5 => ...`. |
| **привязка** (binding) | Присвоение имени части сопоставленного значения, например `n` в `Some(n)`. |
| **образец-диапазон** (range pattern) | Образец, подходящий к промежутку значений: `1..=9` означает от 1 до 9 включительно. |
| **частичное перемещение** (partial move) | Перемещение лишь части значения (поля или нагрузки), после которого всё значение становится непригодным. |
| **`Result`** | Перечисление для операций, которые могут завершиться ошибкой: `Ok(value)` или `Err(error)`. Рассматривается в [Уроке 04.10](10-error-handling.md). |

## 1. Проблема: «нет значения»

Многие функции могут не суметь дать ответ. Поиск слова в тексте: слова может не быть. Деление: делитель может быть нулём. Запрос пятого элемента списка: в списке может быть три.

Старые языки используют для «нет значения» особую метку, часто называемую **null**. Любая переменная может втайне оказаться null, а использование такой переменной роняет программу. Тони Хоар, который в 1965 году ввёл null в язык, позже назвал это своей «ошибкой на миллиард долларов». Беда в том, что тип не предупреждает вас: переменная типа «число» может содержать число, а может не содержать, и в коде нигде не сказано, что именно.

В Rust null нет. Вместо него «может отсутствовать» входит в **тип**. Стандартная библиотека определяет перечисление:

```text
enum Option<T> {
    None,
    Some(T),
}
```

`T` означает «любой тип». `Option<i32>` — это либо `Some(5)` (`i32` есть), либо `None` (`i32` нет). `Option` настолько распространён, что `Option`, `Some` и `None` доступны везде без всякого `use`.

Поскольку `Option<i32>` и `i32` — разные типы, компилятор **заставляет** вас разобраться со случаем «нет значения», прежде чем вы сможете использовать число.

```rust
fn checked_div(a: i32, b: i32) -> Option<i32> {
    if b == 0 {
        None
    } else {
        Some(a / b)
    }
}

fn main() {
    for (a, b) in [(10, 2), (7, 0), (-9, 3)] {
        match checked_div(a, b) {
            Some(result) => println!("{a} / {b} = {result}"),
            None => println!("{a} / {b} is impossible"),
        }
    }
}
```

**Что происходит:**

1. `checked_div` возвращает `Option<i32>`. Сама сигнатура сообщает читателю: «число может не получиться».
2. Когда делитель равен нулю, мы возвращаем `None`. Иначе оборачиваем ответ: `Some(a / b)`.
3. Цикл `for` идёт по массиву пар. `(a, b)` разбирает каждую пару на две переменные (это образец, [Урок 04.5](05-if-let-let-else-and-patterns.md)).
4. У `match` две ветки, по одной на вариант. `Some(result)` привязывает внутреннее число к имени `result`. Компилятор отверг бы `match`, если бы ветки `None` не было.

```text
10 / 2 = 5
7 / 0 is impossible
-9 / 3 = -3
```

### `Option` нельзя использовать так, будто это само значение

В этом и следующем примерах показаны только строки внутри `fn main() { … }`; номера строк в сообщениях об ошибках считаются от начала `src/main.rs`, где строка 1 — это `fn main() {`.

```rust
let x: i32 = 5;
let y: Option<i32> = Some(5);
let sum = x + y;
```

```text
error[E0277]: cannot add `Option<i32>` to `i32`
 --> src/main.rs:4:17
  |
4 |     let sum = x + y;
  |                 ^ no implementation for `i32 + Option<i32>`
  |
  = help: the trait `Add<Option<i32>>` is not implemented for `i32`
```

(Настоящее сообщение продолжается списком других типов, поддерживающих `+`; здесь оно сокращено.) В этом и смысл: компилятор не даёт сложить возможно отсутствующее число. Сначала число нужно достать и решить, что делать, когда его нет.

### Для `None` нужен тип

```rust
let nothing = None;
```

```text
error[E0282]: type annotations needed for `Option<_>`
 --> src/main.rs:2:9
  |
2 |     let nothing = None;
  |         ^^^^^^^   ---- type must be known at this point
  |
help: consider giving `nothing` an explicit type, where the type for type parameter `T` is specified
  |
2 |     let nothing: Option<T> = None;
  |                +++++++++++
```

Одно `None` не говорит «необязательное — что?». Напишите `let nothing: Option<i32> = None;`. Обычно тип ясен из дальнейшего использования, и аннотация не нужна.

## 2. Как достать значение

### Через `match`

`match` — самый общий способ, вы видели его выше. В каждой ветке можно выполнить любой код, и ветки дают значение:

```text
let text = match maybe { Some(n) => format!("got {n}"), None => String::from("nothing") };
```

### Через `unwrap`, `expect` и их родственников

Для коротких случаев есть методы. Безопасные дают запасное значение:

```rust
fn main() {
    let some: Option<i32> = Some(7);
    let none: Option<i32> = None;

    println!("{}", some.unwrap_or(0));
    println!("{}", none.unwrap_or(0));
    println!("{}", none.unwrap_or_default());
    println!("{} {}", some.is_some(), none.is_none());
    println!("{}", some.expect("a number must be present"));
}
```

**Что происходит:**

1. `unwrap_or(0)` возвращает внутреннее значение или `0`, если его нет.
2. `unwrap_or_default()` возвращает значение типа по умолчанию (`0` для чисел, пустую `String` для текста; см. `Default` в [Уроке 04.2](02-methods-and-associated-functions.md)).
3. `is_some()` и `is_none()` отвечают на вопросы «да/нет», не отдавая значение.
4. `expect("...")` возвращает значение, а если его нет — **паникует** с вашим сообщением. Здесь значение есть, так что получается просто `7`.

```text
7
0
0
true true
7
```

Опасны `unwrap()` и `expect()` на `None`:

```rust
fn main() {
    let none: Option<i32> = None;
    println!("before");
    let n = none.unwrap();
    println!("after {n}");
}
```

```text
before

thread 'main' (3975970) panicked at src/main.rs:4:18:
called `Option::unwrap()` on a `None` value
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
```

Программа останавливается на этой строке. «after» не выводится никогда. Число в скобках — идентификатор потока, оно меняется при каждом запуске. С `expect("config must contain a port")` сообщением будет ваш текст, что гораздо полезнее при отладке.

> 🔒 **Безопасность:** `panic` на вводе, который приходит снаружи — из файла, с клавиатуры, из сетевого запроса, — это ошибка, которую злоумышленник может вызвать нарочно, чтобы уронить вашу программу (*отказ в обслуживании*). Никогда не вызывайте `unwrap` на данных, которые вы не контролируете. Используйте `match`, `unwrap_or` или верните проблему вызывающему коду.

> 📏 **Соглашение:** `unwrap` и `expect` уместны в трёх местах: в тестах, в быстрых экспериментах и когда вы можете доказать, что значение есть (тогда лучше `expect("почему этого не может случиться")`, чтобы доказательство было записано). Везде в остальных случаях обрабатывайте `None`.

## 3. Работа со значением, не открывая коробку

Часто `Option` вообще не нужно открывать; нужно *преобразовать* то, что внутри, и сохранить «может быть» на потом. Эти методы принимают **замыкание** — крошечную функцию. Читайте `|w| w * 2` как «получив `w`, вернуть `w * 2`».

```rust
fn main() {
    let width: Option<u32> = Some(4);
    let nothing: Option<u32> = None;

    // map: изменить значение внутри, если оно есть
    println!("{:?}", width.map(|w| w * 2));
    println!("{:?}", nothing.map(|w| w * 2));

    // and_then: сам шаг может дать None
    let half = |n: u32| if n % 2 == 0 { Some(n / 2) } else { None };
    println!("{:?}", Some(10).and_then(half));
    println!("{:?}", Some(7).and_then(half));

    // filter: оставить значение, только если проверка пройдена
    println!("{:?}", Some(15).filter(|n| *n > 10));
    println!("{:?}", Some(5).filter(|n| *n > 10));

    // ok_or: превратить None в значение-ошибку (Result)
    let r1: Result<u32, &str> = width.ok_or("no width given");
    let r2: Result<u32, &str> = nothing.ok_or("no width given");
    println!("{:?} {:?}", r1, r2);

    // unwrap_or_else: вычислять запасное значение, только когда оно нужно
    println!("{}", nothing.unwrap_or_else(|| 10 + 20));
}
```

**Что происходит:**

1. `map` применяет функцию к содержимому `Some` и снова оборачивает результат. `None` остаётся `None`; функция не выполняется.
2. `and_then` нужен для шага, который сам может не удаться. `half` возвращает `Some` для чётных чисел и `None` для нечётных. С `map` вы получили бы неудобный `Option<Option<u32>>`; `and_then` «сплющивает» его.
3. `filter` превращает `Some(x)` в `None`, если `x` не проходит проверку. В замыкании `n` — ссылка на число, поэтому `*n` читает значение.
4. `ok_or("...")` преобразует `Option` в `Result`: `Some(v)` становится `Ok(v)`, а `None` — `Err("...")`. Это мост к обработке ошибок ([Урок 04.10](10-error-handling.md)).
5. `unwrap_or_else(|| ...)` выполняет замыкание, только когда значения нет. Используйте его вместо `unwrap_or`, когда запасное значение дорого вычислять.

```text
Some(8)
None
Some(5)
None
Some(15)
None
Ok(4) Err("no width given")
30
```

> 💡 **Совет:** запоминать все методы не нужно. Запомните идею (у `Option` много помощников) и ищите список в документации [`Option`](https://doc.rust-lang.org/std/option/enum.Option.html), когда понадобится нужный. Автодополнение RustRover тоже показывает их после точки.

## 4. Откуда берётся `Option`

С `Option` вы будете постоянно встречаться в стандартной библиотеке:

```rust
fn main() {
    let numbers = [10, 20, 30];

    println!("{:?}", numbers.get(1));
    println!("{:?}", numbers.get(5));
    println!("{:?}", numbers.first());
    println!("{:?}", "hello".find('l'));
    println!("{:?}", "hello".find('z'));
    println!("{:?}", '7'.to_digit(10));
    println!("{:?}", 'x'.to_digit(10));
    println!("{:?}", "42".parse::<i32>().ok());
    println!("{:?}", "4x2".parse::<i32>().ok());
}
```

**Что происходит:**

1. `numbers.get(index)` возвращает `Some(&element)` или `None`, если индекс слишком велик. Сравните с `numbers[5]`, который **паникует**. Используйте `get`, когда индекс приходит снаружи.
2. `first()` — первый элемент или `None` для пустого списка.
3. `"hello".find('l')` возвращает позицию (индекс в байтах) первого совпадения: `Some(2)`; отсутствующий символ даёт `None`.
4. `to_digit(10)` превращает символ в его числовое значение по основанию 10 или в `None`, если это не цифра.
5. `parse::<i32>()` на самом деле возвращает `Result`; `.ok()` превращает `Ok(v)` в `Some(v)`, а `Err(_)` — в `None`, отбрасывая подробности ошибки.

```text
Some(20)
None
Some(10)
Some(2)
None
Some(7)
None
Some(42)
None
```

(`get` и `first` возвращают ссылку на элемент массива; `{:?}` печатает её как обычное значение.)

## 5. `match` подробно

Вы видели `match` по перечислениям. `match` может исследовать любое значение, а образцы бывают богатыми.

```rust
fn classify(n: i32) -> &'static str {
    match n {
        i32::MIN..=-1 => "negative",
        0 => "zero",
        1..=9 => "one digit",
        10 | 20 | 30 => "round number",
        x if x % 2 == 0 => "big and even",
        _ => "big and odd",
    }
}

fn describe_pair(pair: (i32, i32)) -> String {
    match pair {
        (0, 0) => String::from("origin"),
        (x, 0) => format!("on the x axis at {x}"),
        (0, y) => format!("on the y axis at {y}"),
        (x, y) => format!("at ({x}, {y})"),
    }
}

fn age_group(age: u32) -> String {
    match age {
        n @ 0..=12 => format!("child, {n} years"),
        n @ 13..=19 => format!("teenager, {n} years"),
        n => format!("adult, {n} years"),
    }
}

fn main() {
    for n in [-5, 0, 7, 20, 44, 51] {
        println!("{n}: {}", classify(n));
    }
    println!("{}", describe_pair((0, 0)));
    println!("{}", describe_pair((5, 0)));
    println!("{}", describe_pair((0, -2)));
    println!("{}", describe_pair((3, 4)));
    println!("{}", age_group(9));
    println!("{}", age_group(15));
    println!("{}", age_group(40));
}
```

**Что происходит:**

1. `i32::MIN..=-1` — **образец-диапазон**: от наименьшего `i32` до `-1` включительно. `1..=9` — то же для чисел от 1 до 9. `..=` означает «до и включая». Диапазоны работают для целых чисел и символов.
2. `10 | 20 | 30` подходит к любому из трёх значений.
3. `x if x % 2 == 0` — ветка с **охранным условием**: образец `x` подходит к любому числу, но ветка используется, только если условие после `if` истинно. Иначе пробуется следующая ветка. Компилятор не заглядывает внутрь охранного условия, поэтому в конце `match` всё равно нужна ветка без него (`_`).
4. В `describe_pair` значение — **кортеж**, и образцы тоже кортежи. Литерал вроде `0` должен совпасть точно; имя вроде `x` подходит к чему угодно и привязывает значение. Побеждает первая подошедшая ветка, поэтому порядок важен: `(0, 0)` проверяется раньше `(x, 0)`.
5. `n @ 0..=12` — **привязка с проверкой**: проверить диапазон *и* сохранить значение под именем `n`. Последняя ветка `n => ...` забирает всё оставшееся.

```text
-5: negative
0: zero
7: one digit
20: round number
44: big and even
51: big and odd
origin
on the x axis at 5
on the y axis at -2
at (3, 4)
child, 9 years
teenager, 15 years
adult, 40 years
```

Образцы также вкладываются друг в друга. Это полезно для `Option` внутри `Option` или перечисления внутри структуры:

```rust
fn main() {
    let maybe_number = Some(Some(3));
    match maybe_number {
        Some(Some(n)) if n > 2 => println!("big inner {n}"),
        Some(Some(n)) => println!("small inner {n}"),
        Some(None) => println!("outer present, inner missing"),
        None => println!("nothing"),
    }

    let scores = [Some(90), None, Some(55)];
    let mut total = 0;
    for score in scores {
        match score {
            Some(s) => total += s,
            None => println!("skipped a missing score"),
        }
    }
    println!("total = {total}");
}
```

**Что происходит:**

1. `Some(Some(n)) if n > 2` заглядывает на два уровня вглубь и добавляет охранное условие. Все четыре возможные формы покрыты, так что компилятор доволен.
2. В цикле ветка `match` может быть выражением-«оператором»: у `total += s` тип `()` (ничего), и у `println!` в другой ветке тоже. Все ветки одного `match` должны иметь **один и тот же тип**.

```text
big inner 3
skipped a missing score
total = 145
```

> 💡 **Совет:** если вы забудете ветку `None`, компилятор на вашей стороне:
>
> ```text
> error[E0004]: non-exhaustive patterns: `None` not covered
>  --> src/main.rs:3:11
>   |
> 3 |     match maybe {
>   |           ^^^^^ pattern `None` not covered
>   |
>   = note: the matched value is of type `Option<i32>`
> help: ensure that all possible cases are being handled by adding a match arm with a wildcard pattern or an explicit pattern as shown
>   |
> 4 ~         Some(n) => println!("got {n}"),
> 5 ~         None => todo!(),
>   |
> ```

## 6. `match` и владение: частичное перемещение

Когда вы применяете `match` к `Option<String>` по значению, `String` **перемещается** в переменную образца. После этого оригинал частично пропадает:

```rust
fn main() {
    let name: Option<String> = Some(String::from("Ferris"));

    match name {
        Some(n) => println!("hello, {n}"),
        None => println!("hello, stranger"),
    }

    println!("{:?}", name);
}
```

```text
error[E0382]: borrow of partially moved value: `name`
 --> src/main.rs:9:22
  |
5 |         Some(n) => println!("hello, {n}"),
  |              - value partially moved here
...
9 |     println!("{:?}", name);
  |                      ^^^^ value borrowed here after partial move
  |
  = note: partial move occurs because value has type `String`, which does not implement the `Copy` trait
help: borrow this binding in the pattern to avoid moving the value
  |
5 |         Some(ref n) => println!("hello, {n}"),
  |              +++
```

Компилятор показывает `ref n` — старый способ заимствовать внутри образца. В современном Rust предпочитайте одно из трёх исправлений:

```rust
fn main() {
    let name: Option<String> = Some(String::from("Ferris"));

    // исправление 1: match по ссылке
    match &name {
        Some(n) => println!("hello, {n}"),
        None => println!("hello, stranger"),
    }

    // исправление 2: as_ref() превращает Option<String> в Option<&String>
    if let Some(n) = name.as_ref() {
        println!("length {}", n.len());
    }

    // исправление 3: as_deref() даёт Option<&str>
    let shown: &str = name.as_deref().unwrap_or("stranger");
    println!("shown = {shown}");

    println!("{:?}", name);
}
```

**Что происходит:**

1. `match &name` сопоставляет ссылку. Внутри ветки `n` — это `&String`, заимствование, поэтому `name` остаётся целым.
2. `name.as_ref()` превращает `&Option<String>` в `Option<&String>`: та же форма, но внутри заимствования. Теперь `if let Some(n) = ...` ([Урок 04.5](05-if-let-let-else-and-patterns.md)) только заимствует.
3. `name.as_deref()` идёт на шаг дальше и даёт `Option<&str>`, что отлично сочетается с `unwrap_or("stranger")`.
4. Последняя строка показывает, что `name` никогда не перемещалась.

```text
hello, Ferris
length 6
shown = Ferris
Some("Ferris")
```

Практическое правило: **чтобы заглянуть в `Option`, заимствуйте его** (`&opt`, `opt.as_ref()`, `opt.as_deref()`); чтобы **поглотить** его, сопоставляйте по значению.

## 7. Тестирование кода, который возвращает `Option`

> 🧪 **Тестирование:** проверяйте обе стороны: случай, дающий `Some(...)`, и случай, дающий `None`. Граничные случаи (пустой текст, пустой список, ноль) обычно дают `None`, поэтому они обязательно должны быть в тестах. `assert_eq!(f(x), None)` работает, потому что `Option<T>` поддерживает `==`, если его поддерживает `T`, и выводится через `{:?}`, если `T` выводится.

```rust
fn first_word_len(text: &str) -> Option<usize> {
    text.split_whitespace().next().map(|w| w.len())
}

fn main() {
    println!("{:?}", first_word_len("hello world"));
    println!("{:?}", first_word_len("   "));
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn finds_first_word() {
        assert_eq!(first_word_len("hello world"), Some(5));
    }

    #[test]
    fn empty_text_has_no_word() {
        assert_eq!(first_word_len(""), None);
        assert_eq!(first_word_len("   "), None);
    }
}
```

**Что происходит:**

1. `split_whitespace()` режет текст по пробелам и табуляциям; `.next()` берёт первый кусок в виде `Option` (итератор заканчивается значением `None`; итераторы будут в [Уроке 05.5](../05-rust-intermediate/05-iterators.md)).
2. `.map(|w| w.len())` превращает слово в его длину и оставляет `None` как есть.
3. Тесты охватывают обычный случай и два случая «ничего нет».

```text
Some(5)
None
```

```text
running 2 tests
..
test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

## Частые ошибки

> ⚠️ **Частая ошибка:** использовать `unwrap()` «лишь бы скомпилировалось». Оно компилируется, а потом падает в самый неподходящий момент. Решите, что должно происходить при `None`: значение по умолчанию (`unwrap_or`), ошибка для вызывающего кода (`ok_or`) или сообщение (`match`).

> ⚠️ **Частая ошибка:** обращаться с `Option<i32>` как с `i32` (`E0277`, раздел 1). Сначала откройте коробку.

> ⚠️ **Частая ошибка:** забыть ветку `None` (`E0004`). Компилятор всегда это ловит.

> ⚠️ **Частая ошибка:** частичное перемещение при сопоставлении `Option<String>` по значению с последующим повторным использованием (`E0382`, раздел 6). Сопоставляйте `&opt` или используйте `as_ref()`.

> ⚠️ **Частая ошибка:** использовать `_` как последнюю ветку, чтобы никогда не видеть новых случаев. У `Option` всего два варианта, поэтому пишите `None` явно.

> ⚠️ **Частая ошибка:** обращаться по индексу `numbers[i]`, когда `i` приходит снаружи; это паникует, если `i` слишком велик. Используйте `numbers.get(i)`.

## Попробуйте сами

1. Создайте проект:

   ```text
   cd ~/projects/rust
   cargo new code/p04-option-match
   cd code/p04-option-match
   ```

2. Поместите в `src/main.rs`:

   ```rust
   fn find_index(items: &[&str], wanted: &str) -> Option<usize> {
       for (i, item) in items.iter().enumerate() {
           if *item == wanted {
               return Some(i);
           }
       }
       None
   }

   fn main() {
       let fruits = ["apple", "pear", "plum"];
       for name in ["pear", "kiwi"] {
           match find_index(&fruits, name) {
               Some(i) => println!("{name} is at index {i}"),
               None => println!("{name} is not in the list"),
           }
       }
   }
   ```

   (`iter().enumerate()` обходит список и выдаёт пары `(индекс, элемент)`. Итераторы — тема Урока 05.5; пока это готовый инструмент.)

3. Запустите `cargo run`. Ожидаемый вывод:

   ```text
   pear is at index 1
   kiwi is not in the list
   ```

4. Измените `match` в `main` так, чтобы для отсутствующего фрукта печаталось сообщение, а для найденного — его индекс плюс один (используйте `map`). Затем добавьте тестовый модуль с одним случаем `Some` и одним `None`.
5. Сделайте коммит:

   ```text
   cd ~/projects/rust
   git add code/p04-option-match
   git commit -m "feat(p04): option and match practice"
   ```

## Упражнения

### Упражнение 1 — Классификация числа (легко)

Напишите `fn classify(n: i32) -> &'static str` с одним `match` и образцами-диапазонами: отрицательные числа дают `"negative"`, `0` — `"zero"`, от `1` до `9` — `"small"`, всё больше — `"large"`. Протестируйте границы: `-1`, `0`, `1`, `9`, `10`.

<details>
<summary>Подсказка</summary>

`i32::MIN..=-1 => ...` подходит ко всем отрицательным числам. Для последней ветки используйте `_`.

</details>

<details>
<summary>Ответ</summary>

```rust
fn classify(n: i32) -> &'static str {
    match n {
        i32::MIN..=-1 => "negative",
        0 => "zero",
        1..=9 => "small",
        _ => "large",
    }
}

fn main() {
    for n in [-3, 0, 4, 9, 10, 1000] {
        println!("{n} is {}", classify(n));
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn edges() {
        assert_eq!(classify(-1), "negative");
        assert_eq!(classify(0), "zero");
        assert_eq!(classify(1), "small");
        assert_eq!(classify(9), "small");
        assert_eq!(classify(10), "large");
    }
}
```

Вывод `cargo run`:

```text
-3 is negative
0 is zero
4 is small
9 is small
10 is large
1000 is large
```

Почему это работает: диапазоны охватывают целые промежутки одной веткой, а тесты стоят ровно на границах между диапазонами, где прячутся ошибки на единицу.

</details>

### Упражнение 2 — Первое чётное число (средне)

Напишите `fn first_even(numbers: &[i32]) -> Option<i32>`, которая возвращает первое чётное число или `None`. Тесты: список с чётным числом, список без него и пустой список.

<details>
<summary>Подсказка</summary>

Идите циклом `for &n in numbers` (образец `&n` копирует число из ссылки). Верните `Some(n)`, как только `n % 2 == 0`. После цикла верните `None`.

</details>

<details>
<summary>Ответ</summary>

```rust
fn first_even(numbers: &[i32]) -> Option<i32> {
    for &n in numbers {
        if n % 2 == 0 {
            return Some(n);
        }
    }
    None
}

fn main() {
    println!("{:?}", first_even(&[1, 3, 8, 10]));
    println!("{:?}", first_even(&[1, 3, 5]));
    println!("{:?}", first_even(&[]));
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn finds_the_first() {
        assert_eq!(first_even(&[1, 3, 8, 10]), Some(8));
    }

    #[test]
    fn none_when_no_even_number() {
        assert_eq!(first_even(&[1, 3, 5]), None);
        assert_eq!(first_even(&[]), None);
    }
}
```

Вывод `cargo run`:

```text
Some(8)
None
None
```

Почему это работает: `return` выходит из функции при первом попадании. Если цикл дошёл до конца, чётных чисел нет, а это в точности случай для `None`.

</details>

### Упражнение 3 — Цепочка через `map` (средне)

Есть `fn find_age(name: &str) -> Option<u32>`, которая знает `"alice"` (30) и `"bob"` (25) и для остальных возвращает `None`. Напишите `fn age_next_year(name: &str) -> Option<u32>` **без единого `match` внутри**, используя `map`. Протестируйте одно известное и одно неизвестное имя.

<details>
<summary>Подсказка</summary>

`find_age(name).map(|age| age + 1)`.

</details>

<details>
<summary>Ответ</summary>

```rust
fn find_age(name: &str) -> Option<u32> {
    match name {
        "alice" => Some(30),
        "bob" => Some(25),
        _ => None,
    }
}

fn age_next_year(name: &str) -> Option<u32> {
    find_age(name).map(|age| age + 1)
}

fn main() {
    println!("{:?}", age_next_year("alice"));
    println!("{:?}", age_next_year("carol"));
    println!("{}", age_next_year("bob").unwrap_or(0));
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn known_name() {
        assert_eq!(age_next_year("bob"), Some(26));
    }

    #[test]
    fn unknown_name() {
        assert_eq!(age_next_year("nobody"), None);
    }
}
```

Вывод `cargo run`:

```text
Some(31)
None
26
```

Почему это работает: `map` выполняет замыкание только для `Some`. Для `None` результат автоматически `None`, так что случай «нет значения» проходит насквозь без лишнего кода.

</details>

### Упражнение 4 — Разбор пары (сложно)

Напишите `fn parse_pair(text: &str) -> Option<(i32, i32)>`, которая принимает текст вроде `"3,4"` или `" 10 , -2 "` и возвращает `Some((3, 4))` / `Some((10, -2))`. Всё остальное (`"3"`, `"a,b"`, `"1,"`, `""`) даёт `None`. Протестируйте корректные и некорректные входные данные.

<details>
<summary>Подсказка</summary>

`text.split_once(',')` возвращает `Option<(&str, &str)>`: текст до первой запятой и после неё. Затем сопоставьте кортеж из двух результатов разбора: `match (left.trim().parse::<i32>(), right.trim().parse::<i32>()) { (Ok(x), Ok(y)) => ..., _ => None }`. `trim()` убирает пробелы с обоих концов.

</details>

<details>
<summary>Ответ</summary>

```rust
fn parse_pair(text: &str) -> Option<(i32, i32)> {
    match text.split_once(',') {
        Some((left, right)) => match (left.trim().parse::<i32>(), right.trim().parse::<i32>()) {
            (Ok(x), Ok(y)) => Some((x, y)),
            _ => None,
        },
        None => None,
    }
}

fn main() {
    for text in ["3,4", " 10 , -2 ", "3", "a,b", ""] {
        println!("{:?} -> {:?}", text, parse_pair(text));
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn valid_pairs() {
        assert_eq!(parse_pair("3,4"), Some((3, 4)));
        assert_eq!(parse_pair(" 10 , -2 "), Some((10, -2)));
    }

    #[test]
    fn invalid_pairs() {
        assert_eq!(parse_pair("3"), None);
        assert_eq!(parse_pair("a,b"), None);
        assert_eq!(parse_pair("1,"), None);
        assert_eq!(parse_pair(""), None);
    }
}
```

Вывод `cargo run`:

```text
"3,4" -> Some((3, 4))
" 10 , -2 " -> Some((10, -2))
"3" -> None
"a,b" -> None
"" -> None
```

Почему это работает: внешний `match` отвечает на вопрос «есть ли запятая?», внутренний `match` по кортежу — на вопрос «разобрались ли обе половины?». Ветка `_` собирает все остальные сочетания. В [Уроке 04.10](10-error-handling.md) вы научитесь говорить, *почему* не получилось, с помощью `Result`.

</details>

## Вопросы для самопроверки

1. Какую проблему решает `Option<T>` и почему в Rust нет `null`?

<details>
<summary>Ответ</summary>

Он делает «значение может отсутствовать» видимым в типе, поэтому компилятор заставляет обработать случай отсутствия. С `null` любая переменная может втайне оказаться пустой и уронить программу во время выполнения.

</details>

2. Каковы два варианта `Option<T>`?

<details>
<summary>Ответ</summary>

`Some(T)` хранит значение; `None` не хранит ничего.

</details>

3. Почему `let x = None;` не компилируется?

<details>
<summary>Ответ</summary>

Компилятор не может узнать тип `T` отсутствующего значения (`E0282`). Задайте тип: `let x: Option<i32> = None;` или используйте `x` там, где тип определится.

</details>

4. Чем `unwrap_or(0)` отличается от `unwrap()`?

<details>
<summary>Ответ</summary>

`unwrap_or(0)` возвращает `0`, когда значения нет. `unwrap()` паникует, когда значения нет.

</details>

5. Когда у `Option` использовать `map`, а когда `and_then`?

<details>
<summary>Ответ</summary>

`map` — когда преобразование всегда удаётся (`T -> U`). `and_then` — когда само преобразование может дать `None` (`T -> Option<U>`); он избавляет от вложенного `Option<Option<U>>`.

</details>

6. Что делает `n @ 1..=9`?

<details>
<summary>Ответ</summary>

Проверяет, лежит ли значение в диапазоне от 1 до 9 (включительно), и если да, привязывает всё значение к имени `n`.

</details>

7. Почему `match name { Some(n) => ..., None => ... }` для `Option<String>` делает `name` непригодным после себя и как этого избежать?

<details>
<summary>Ответ</summary>

`String` перемещается из `Option` в `n` (частичное перемещение). Сопоставляйте ссылку: `match &name`, либо используйте `name.as_ref()` / `name.as_deref()`.

</details>

8. Назовите два места, где `unwrap` допустим.

<details>
<summary>Ответ</summary>

В тестах и в быстрых экспериментах. А также когда значение заведомо есть (тогда лучше `expect` с сообщением, объясняющим почему).

</details>

## Итоги

- `Option<T>` — обычное перечисление `Some(T)` или `None`. Оно заменяет `null` и переносит «может отсутствовать» в тип.
- Использовать `Option<T>` как `T` нельзя; его нужно открывать через `match`, `unwrap_or`, `if let` (следующий урок) или другие методы.
- `unwrap` и `expect` паникуют на `None`; оставьте их для тестов и доказуемых случаев, но никогда не применяйте к внешнему вводу.
- `map`, `and_then`, `filter`, `ok_or`, `unwrap_or_else`, `as_ref` и `as_deref` позволяют работать с `Option`, не открывая его.
- Многие функции стандартной библиотеки возвращают `Option`: `get`, `first`, `find`, `to_digit`, `split_once`. Для внешних индексов предпочитайте `get` вместо `[i]`.
- `match` поддерживает литералы, диапазоны (`1..=9`), альтернативы (`|`), охранные условия (`if`), привязки (`n @ ...`), кортежи и вложенность; все ветки имеют один тип, а `match` должен быть исчерпывающим.
- Чтобы заглянуть в `Option`, не поглощая его, заимствуйте: `&opt`, `opt.as_ref()`, `opt.as_deref()`.
- Тестируйте случаи и `Some`, и `None`, а также границы между диапазонами.

## Источники и что почитать

- [The Rust Programming Language — 6.1 Defining an Enum (the Option section)](https://doc.rust-lang.org/book/ch06-01-defining-an-enum.html) — официальное объяснение `Option` и null.
- [The Rust Programming Language — 6.2 The match Control Flow Construct](https://doc.rust-lang.org/book/ch06-02-match.html) — `match`, образцы, привязывающие значения, и исчерпывающность.
- [The Rust Programming Language — 19.3 Pattern Syntax](https://doc.rust-lang.org/book/ch19-03-pattern-syntax.html) — полный список форм образцов (диапазоны, охранные условия, `@`).
- [std::option::Option](https://doc.rust-lang.org/std/option/enum.Option.html) — полный список методов `Option` с примерами.
- [Rust by Example — Option](https://doc.rust-lang.org/rust-by-example/std/option.html) — запускаемые примеры.
- [Rust by Example — match](https://doc.rust-lang.org/rust-by-example/flow_control/match.html) — запускаемые примеры форм образцов из раздела 5.

---

[🏠 Начало](../START-HERE.md) · [← 04.3 Перечисления](03-enums.md) · [📋 Этап 04](README.md) · [04.5 if let, let else и образцы →](05-if-let-let-else-and-patterns.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
