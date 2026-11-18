<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 03.3 Перемещение, копирование и клонирование](03-move-copy-and-clone.md) · [📋 Этап 03](README.md) · [03.5 Изменяемые ссылки и правила заимствования →](05-mutable-references-and-the-borrow-rules.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/03-ownership-and-borrowing/04-references-and-borrowing.md) <!-- nav -->

# Урок 03.4 — Ссылки и заимствование

> **Этап 03 — Владение и заимствование** · Урок 4 из 8 · ⏱ около 2,5 часа (2 занятия)

## Что вы узнаете

- Что такое **ссылка** (`&`) и чем она отличается от владения.
- Как позволить функции *посмотреть на* значение, не забирая его (**заимствование**, borrowing).
- Как читать через ссылку с помощью `*` (разыменование) и почему нельзя изменить значение через `&`.
- Почему Rust никогда не допускает **висячую ссылку** и как долго длится заимствование.
- Когда функции стоит заимствовать, а когда владеть.

## Перед началом

- Вы прошли [Урок 03.3 — Перемещение, копирование и клонирование](03-move-copy-and-clone.md). Вы знаете, что передача `String` по значению перемещает её.
- Создайте проект: `cargo new code/p03-borrowing` (из `~/projects/rust`).

## Новые слова

| Слово | Значение |
|---|---|
| **ссылка** (reference) | Значение, которое указывает на другое значение, не владея им. Записывается `&value`. Её тип — `&T`, «ссылка на `T`». |
| **заимствование** (borrow) | Создание ссылки. Пока ссылка существует, владелец по-прежнему владеет значением. |
| **разделяемая ссылка** (shared reference, также *неизменяемая ссылка*) | Ссылка `&T`, которая может только **читать**. Одновременно может существовать сколько угодно таких ссылок. |
| **разыменование** (dereference) | Переход по ссылке к значению, на которое она указывает. Записывается `*r`. |
| **висячая ссылка** (dangling reference) | Ссылка на значение, которого больше не существует. Rust делает её невозможной. |
| **время жизни** (lifetime) | Период, в течение которого ссылка действительна. Его проверяет компилятор. (Времена жизни полностью изучаются в Уроках 05.3 и 06.5.) |
| **NLL** (Non-Lexical Lifetimes, «нелексические времена жизни») | Правило компилятора: заимствование заканчивается на *последнем использовании* ссылки, а не в конце блока. |
| **сигнатура** (signature) | Первая строка функции: её имя, параметры и тип возвращаемого значения. |

## 1. Задача, которую нужно решить

В Уроке 03.2 мы написали функцию, которой нужна была только *длина* строки. Поскольку передача `String` её перемещает, функции пришлось вернуть строку обратно:

```rust
fn calculate_length(s: String) -> (String, usize) {
    let length = s.len();
    (s, length)
}

fn main() {
    let s1 = String::from("hello");
    let (s2, len) = calculate_length(s1);
    println!("The length of '{s2}' is {len}.");
}
```

```text
The length of 'hello' is 5.
```

Это работает, но выглядит глупо. Функции не нужно *владеть* строкой; ей нужно лишь на миг на неё взглянуть. Человеку, который хочет прочитать вашу книгу, не обязательно ею владеть. Он берёт её взаймы и возвращает.

## 2. Ссылки: заимствование без владения

**Ссылка** позволяет обратиться к значению, не забирая его во владение. Вы создаёте её оператором `&`, а её тип записывается как `&T`:

```rust
fn calculate_length(s: &String) -> usize {
    s.len()
}

fn main() {
    let s1 = String::from("hello");
    let len = calculate_length(&s1);
    println!("The length of '{s1}' is {len}.");
}
```

```text
The length of 'hello' is 5.
```

**Что происходит:**

1. `&s1` создаёт ссылку на `s1`. `String` **не перемещается**. `s1` остаётся её владельцем.
2. Параметр функции `s: &String` означает «ссылка на `String`». Внутри функции `s.len()` работает так, как будто `s` — сама строка.
3. Когда функция заканчивается, `s` выходит из области видимости. Но `s` не владеет `String`, поэтому **ничего не уничтожается**. Ссылка просто исчезает.
4. Вернувшись в `main`, мы видим, что `s1` по-прежнему полностью действительна. Мы снова используем её в `println!`.

Создание ссылки называется **заимствованием**. Как и в жизни, вы берёте что-то взаймы, пользуетесь и возвращаете; владелец остаётся владельцем.

Ссылка — это просто адрес. Посмотрим на неё:

```rust
fn main() {
    let s = String::from("hello");
    let r = &s;

    println!("s is at   {:p}", &s);
    println!("r points to {:p}", r);
    println!("r itself is at {:p}", &r);
    println!("letters at  {:p}", s.as_ptr());
    println!("size of s: {} bytes, size of r: {} bytes", size_of::<String>(), size_of::<&String>());
}
```

```text
s is at   0x16d129d28
r points to 0x16d129d28
r itself is at 0x16d129d40
letters at  0x102d752d0
size of s: 24 bytes, size of r: 8 bytes
```

Ссылка `r` хранит адрес `s` (то же число `0x16d129d28`) и занимает всего 8 байтов:

```text
 STACK                               HEAP
┌─────────────────────┐
│ r: &String          │
│   address of s ─────┼──┐
└─────────────────────┘  │
┌─────────────────────┐  │         ┌───┬───┬───┬───┬───┐
│ s: String      ◄────┼──┘         │ h │ e │ l │ l │ o │
│   ptr ──────────────┼──────────► └───┴───┴───┴───┴───┘
│   len = 5           │
│   cap = 5           │
└─────────────────────┘
```

Передача `&s` в функцию копирует всего 8 байтов, какой бы длины ни был текст. Это дёшево, как перемещение, но владелец сохраняет значение.

## 3. Чтение через ссылку

**Много читателей одновременно.** Разделяемая ссылка только читает, поэтому безопасно иметь их сколько угодно, и все они могут использоваться одновременно. Ссылку можно даже скопировать (ссылки на разделяемые данные — `Copy`, как вы видели в Уроке 03.3):

```rust
fn main() {
    let s = String::from("hello");
    let r1 = &s;
    let r2 = &s;
    let r3 = r1; // a reference can be copied
    println!("{r1} {r2} {r3}");
    println!("and the owner can still read: {s}");
}
```

```text
hello hello hello
and the owner can still read: hello
```

**Разыменование.** Для ссылки на число вы используете `*`, чтобы получить само число:

```rust
fn main() {
    let x = 5;
    let r = &x;
    println!("x = {x}, *r = {}", *r);
    println!("*r + 1 = {}", *r + 1);
    println!("r + 1 = {}", r + 1);
    println!("{}", *r == 5);
}
```

```text
x = 5, *r = 5
*r + 1 = 6
r + 1 = 6
true
```

`*r` означает «значение, на которое указывает `r`». Rust часто разыменовывает за вас: вызовы методов вроде `s.len()` и макрос `println!` автоматически смотрят сквозь ссылки. Арифметика с обычным числом тоже случайно работает (`r + 1`), а вот **сравнение** — нет:

```rust
fn main() {
    let x = 5;
    let r = &x;
    if r == 5 {
        println!("five");
    }
}
```

```text
error[E0277]: can't compare `&{integer}` with `{integer}`
 --> src/main.rs:4:10
  |
4 |     if r == 5 {
  |          ^^ no implementation for `&{integer} == {integer}`
  |
  = help: the trait `PartialEq<{integer}>` is not implemented for `&{integer}`
help: consider dereferencing here
  |
4 |     if *r == 5 {
  |        +
```

Здесь сказано, что ссылка на целое и целое — разные типы, поэтому их нельзя сравнивать. Исправление показано в подсказке: напишите `*r == 5`.

**Заимствованный `Vec`.** То же работает и для вектора:

```rust
fn total(numbers: &Vec<i32>) -> i32 {
    let mut sum = 0;
    for n in numbers {
        sum += n;
    }
    sum
}

fn main() {
    let numbers = vec![3, 4, 5];
    println!("total = {}", total(&numbers));
    println!("total again = {}", total(&numbers));
    println!("numbers = {:?}", numbers);
}
```

```text
total = 12
total again = 12
numbers = [3, 4, 5]
```

Здесь `for n in numbers` обходит *ссылку* на вектор, поэтому `n` — это `&i32`, а `sum += n` прибавляет число, на которое она указывает. Поскольку функция лишь заимствовала вектор, `main` может вызвать её дважды и в конце всё равно владеть `numbers`.

> 📏 **Соглашение:** функция, которой нужно только читать значение, должна принимать ссылку. Это не заставляет вызывающий код отдавать значение и дешевле клона. Инструмент Rust `cargo clippy` (см. Урок 03.7), кстати, предложит `&str` вместо `&String` и `&[T]` вместо `&Vec<T>` для параметров. Мы сохраняем `&String` и `&Vec<T>` ещё на несколько уроков; Урок 03.6 объясняет лучшие типы.

## 4. Через `&` нельзя изменить значение

Разделяемая ссылка доступна только для чтения. Если функция пытается изменить то, что заняла:

```rust
fn add_world(s: &String) {
    s.push_str(", world");
}

fn main() {
    let s = String::from("hello");
    add_world(&s);
    println!("{s}");
}
```

```text
error[E0596]: cannot borrow `*s` as mutable, as it is behind a `&` reference
 --> src/main.rs:2:5
  |
2 |     s.push_str(", world");
  |     ^ `s` is a `&` reference, so it cannot be borrowed as mutable
  |
help: consider changing this to be a mutable reference
  |
1 | fn add_world(s: &mut String) {
  |                  +++
```

Это функция безопасности. Читатели делят значение, поэтому никто не имеет права менять его у них под ногами. Если нужно изменить значение через ссылку, нужна **изменяемая ссылка**, `&mut`. Это целиком тема [Урока 03.5](05-mutable-references-and-the-borrow-rules.md), а компилятор уже показывает исправление: `&mut String`.

## 5. Ссылки должны всегда быть действительными

Компилятор гарантирует, что **ссылка никогда не переживёт значение, на которое указывает.** Ссылка на освобождённую память была бы висячим указателем (Урок 03.1). Rust проверяет это при компиляции.

### Возврат ссылки на локальную переменную

```rust
fn make_greeting() -> &String {
    let s = String::from("hello");
    &s
}

fn main() {
    let g = make_greeting();
    println!("{g}");
}
```

```text
error[E0106]: missing lifetime specifier
 --> src/main.rs:1:23
  |
1 | fn make_greeting() -> &String {
  |                       ^ expected named lifetime parameter
  |
  = help: this function's return type contains a borrowed value, but there is no value for it to be borrowed from
help: consider using the `'static` lifetime, but this is uncommon unless you're returning a borrowed value from a `const` or a `static`
  |
1 | fn make_greeting() -> &'static String {
  |                        +++++++
help: instead, you are more likely to want to return an owned value
  |
1 - fn make_greeting() -> &String {
1 + fn make_greeting() -> String {
  |
```

**Что происходит:** `s` создаётся внутри функции и уничтожается, когда функция заканчивается. Ссылка на неё указывала бы на освобождённую память. Компилятор замечает, что возвращаемый тип — ссылка, но снаружи нет значения, которое можно было бы заимствовать. В сообщении упоминается **время жизни** (lifetime): период, в течение которого ссылка действительна. Как записывать времена жизни, вы узнаете в Уроке 05.3. Пока полезна последняя строка: «you are more likely to want to return an owned value» («вероятнее всего, вам нужно вернуть значение во владение»). Это и есть исправление:

```rust
fn make_greeting() -> String {
    String::from("hello")
}

fn main() {
    let g = make_greeting();
    println!("{g}");
}
```

```text
hello
```

Владение перемещается наружу к вызывающему коду, и ничего не «висит».

### Ссылка, которая живёт дольше владельца

```rust
fn main() {
    let r;
    {
        let x = 5;
        r = &x;
    }
    println!("r = {r}");
}
```

```text
error[E0597]: `x` does not live long enough
 --> src/main.rs:5:13
  |
4 |         let x = 5;
  |             - binding `x` declared here
5 |         r = &x;
  |             ^^ borrowed value does not live long enough
6 |     }
  |     - `x` dropped here while still borrowed
7 |     println!("r = {r}");
  |                    - borrow later used here
```

Эта ошибка читается как история: `x` создаётся; `r` её заимствует; `x` уничтожается на закрывающей скобке, «пока ещё заимствована»; а позже `r` используется. `r` указывала бы на мёртвую переменную. Компилятор это останавливает.

### Нельзя переместить владельца, пока он заимствован

```rust
fn main() {
    let s = String::from("hello");
    let r = &s;
    let t = s;
    println!("{r} {t}");
}
```

```text
error[E0505]: cannot move out of `s` because it is borrowed
 --> src/main.rs:4:13
  |
2 |     let s = String::from("hello");
  |         - binding `s` declared here
3 |     let r = &s;
  |             -- borrow of `s` occurs here
4 |     let t = s;
  |             ^ move out of `s` occurs here
5 |     println!("{r} {t}");
  |                - borrow later used here
```

Если бы `s` переместилась в `t`, то `r` указывала бы на старое место, а `t` могла бы затем уничтожить значение, пока `r` ещё используется. Правило: **пока заимствование живо, владелец не может переместить или уничтожить значение.**

### Как долго длится заимствование?

Заимствование длится до **последнего использования** ссылки, а не до конца блока. Это правило называется **NLL** (Non-Lexical Lifetimes). Поэтому эта версия допустима:

```rust
fn main() {
    let s = String::from("hello");
    let r = &s;
    println!("reading through r: {r}"); // last use of r
    let t = s; // fine: the borrow is already over
    println!("t = {t}");
}
```

```text
reading through r: hello
t = hello
```

После `println!` ссылка `r` больше нигде не используется, поэтому заимствование закончилось и перемещение разрешено. Благодаря этому Rust дружелюбнее, чем было бы правило «до закрывающей скобки».

> 🔒 **Безопасность:** висячая ссылка — это та же ошибка, что и использование после освобождения из Урока 03.1. В C компилятор охотно её принимает, и атакующие используют такие ошибки десятилетиями. В Rust три ошибки выше (`E0106`, `E0597`, `E0505`) превращают весь этот класс ошибок в ошибки компиляции.

## 6. Заимствовать или владеть? Краткое руководство

| Функции нужно… | Что принимать |
|---|---|
| прочитать значение | `&T` |
| сохранить значение (хранить, вернуть, отправить дальше) | `T` (владение) |
| изменить значение на месте | `&mut T` (следующий урок) |

Большинство функций только читают, поэтому большинство параметров — ссылки.

## Частые ошибки

**1. Забыть `&` в месте вызова.** Если функция принимает `&String`, а вы передаёте `s`, вы получите ошибку типа `expected &String, found String`. Пишите `calculate_length(&s)`.

**2. Пытаться изменить через `&`.** Ошибка `E0596` (раздел 4). Нужен `&mut` (следующий урок).

**3. Возвращать ссылку на локальную переменную.** Ошибки `E0106` или `E0515` (Урок 03.7 показывает вторую). Возвращайте значение во владение.

**4. Сравнивать ссылку с обычным значением.** Ошибка `E0277`. Разыменуйте с помощью `*`.

**5. Перемещать владельца, пока ссылка ещё используется дальше.** Ошибка `E0505`. Сначала закончите пользоваться ссылкой или переставьте код.

## Попробуйте сами

1. В `~/projects/rust` выполните `cargo new code/p03-borrowing` (если вы ещё не сделали этого в разделе «Перед началом»). В `src/main.rs` напишите `calculate_length` с `&String` и вызовите её дважды с одной и той же строкой. Убедитесь, что строкой можно пользоваться и после.
2. Добавьте строки `{:p}` из раздела 2 и убедитесь, что `&s` и `r` выводят один и тот же адрес.
3. Уберите `&` в вызове `calculate_length(&s1)`. Прочтите ошибку: что в ней сказано, что ожидалось и что было найдено?
4. Напишите функцию `fn print_all(items: &Vec<String>)`, которая выводит каждую строку циклом `for`, и вызовите её дважды.
5. Сделайте коммит: `git add code/p03-borrowing && git commit -m "p03: references and borrowing"`.

## Упражнения

### Упражнение 1 — Посчитайте букву (легко)

Напишите `fn count_char(s: &String, target: char) -> usize`, которая считает, сколько раз `target` встречается в `s`. В `main` вызовите её дважды с одной и той же `String` (один раз для `'a'` и один раз для `'n'` в `"banana"`), а затем выведите строку, чтобы показать, что она всё ещё ваша.

<details>
<summary>Подсказка</summary>

Обходите циклом `for c in s.chars()`. `s.chars()` выдаёт символы по одному. Сравнивайте каждый `c` с `target` и прибавляйте 1 к счётчику.

</details>

<details>
<summary>Ответ</summary>

```rust
fn count_char(s: &String, target: char) -> usize {
    let mut n = 0;
    for c in s.chars() {
        if c == target {
            n += 1;
        }
    }
    n
}

fn main() {
    let text = String::from("banana");
    println!("a appears {} times", count_char(&text, 'a'));
    println!("n appears {} times", count_char(&text, 'n'));
    println!("text is still mine: {text}");
}
```

```text
a appears 3 times
n appears 2 times
text is still mine: banana
```

Почему это работает: функция заимствует строку, поэтому `text` никогда не перемещается и может использоваться снова.

</details>

### Упражнение 2 — Остановите перемещение (легко)

Исправьте эту программу, изменив **только функцию и вызов**, без клонирования:

```rust
fn print_len(v: Vec<i32>) {
    println!("the vector has {} items", v.len());
}

fn main() {
    let numbers = vec![1, 2, 3];
    print_len(numbers);
    println!("first number: {}", numbers[0]);
}
```

<details>
<summary>Подсказка</summary>

Измените тип параметра на `&Vec<i32>` и передавайте `&numbers`.

</details>

<details>
<summary>Ответ</summary>

```rust
fn print_len(v: &Vec<i32>) {
    println!("the vector has {} items", v.len());
}

fn main() {
    let numbers = vec![1, 2, 3];
    print_len(&numbers);
    println!("first number: {}", numbers[0]);
}
```

```text
the vector has 3 items
first number: 1
```

Почему это работает: заимствование оставляет `numbers` у `main`.

</details>

### Упражнение 3 — Наибольшее число, с тестами (средне)

Напишите `fn largest(numbers: &Vec<i32>) -> i32`, которая возвращает наибольшее число в векторе или `i32::MIN` (наименьшее возможное `i32`), когда вектор пуст. Напишите четыре теста: обычный случай, отрицательные числа, пустой вектор и тест, что вектор остаётся пригодным после вызова.

<details>
<summary>Подсказка</summary>

Начните с `let mut best = i32::MIN;`, обходите `for n in numbers` и сравнивайте `*n > best`. В тестах передавайте `&vec![3, 9, 2]`.

</details>

<details>
<summary>Ответ</summary>

```rust
fn largest(numbers: &Vec<i32>) -> i32 {
    let mut best = i32::MIN;
    for n in numbers {
        if *n > best {
            best = *n;
        }
    }
    best
}

fn main() {
    let numbers = vec![3, 9, 2];
    println!("largest = {}", largest(&numbers));
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn finds_the_largest() {
        assert_eq!(largest(&vec![3, 9, 2]), 9);
    }

    #[test]
    fn works_with_negative_numbers() {
        assert_eq!(largest(&vec![-5, -2, -9]), -2);
    }

    #[test]
    fn empty_vector_gives_the_smallest_i32() {
        assert_eq!(largest(&vec![]), i32::MIN);
    }

    #[test]
    fn does_not_consume_the_vector() {
        let v = vec![1, 2, 3];
        let _ = largest(&v);
        assert_eq!(v.len(), 3);
    }
}
```

```text
running 4 tests
....
test result: ok. 4 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

Почему это работает: `n` — это `&i32`, поэтому мы разыменовываем `*n`, чтобы сравнить и сохранить число. Последний тест даже не скомпилировался бы, если бы `largest` принимала вектор по значению.

</details>

### Упражнение 4 — Какие программы компилируются? (средне)

Для каждой программы решите, компилируется ли она. Если нет — назовите код ошибки и объясните почему.

```rust
// a
let name = String::from("Ada");
let r = &name;
let moved = name;
println!("{moved}");
println!("{r}");

// b
let name = String::from("Ada");
let r = &name;
println!("{r}");
let moved = name;
println!("{moved}");

// c
let outer;
{
    let inner = String::from("temporary");
    outer = &inner;
    println!("inside: {outer}");
}

// d
let a = String::from("x");
let r1 = &a;
let r2 = &a;
let r3 = &r1;
println!("{r1} {r2} {r3}");
```

(Каждый фрагмент — это тело `main`.)

<details>
<summary>Подсказка</summary>

Для каждой ссылки спросите: когда она используется в последний раз? И перемещается или уничтожается ли владелец *до* этого момента?

</details>

<details>
<summary>Ответ</summary>

- a: **`E0505`**. `name` перемещается, пока `r` ещё используется позже (`println!("{r}")`).
- b: компилируется. `r` в последний раз используется до перемещения.
- c: компилируется. `outer` используется только внутри блока, пока `inner` жива. (Если бы вы использовали `outer` после закрывающей скобки, получили бы `E0597`.)
- d: компилируется. Может существовать сколько угодно разделяемых ссылок, а `r3` — просто ссылка на ссылку (`&&String`).

Почему это работает: компилятор для каждого заимствования отслеживает участок от его создания до последнего использования. Конфликт возникает, только если владелец перемещается или уничтожается внутри этого участка.

</details>

### Упражнение 5 — Заимствуйте дважды, затем отдайте (сложно)

Напишите `fn total_length(words: &Vec<String>) -> usize`, которая возвращает сумму длин всех строк, и `fn consume(words: Vec<String>)`, которая выводит, сколько слов получила. В `main` создайте вектор из трёх строк, вызовите `total_length` дважды (выведя оба результата) и только затем вызовите `consume`. После этого попробуйте вызвать `total_length` *после* `consume` и прочтите ошибку.

<details>
<summary>Подсказка</summary>

`total_length` принимает `&words`; `consume` принимает сам `words`. После `consume(words)` вектор перемещён.

</details>

<details>
<summary>Ответ</summary>

```rust
fn total_length(words: &Vec<String>) -> usize {
    let mut total = 0;
    for w in words {
        total += w.len();
    }
    total
}

fn consume(words: Vec<String>) {
    println!("consumed {} words", words.len());
}

fn main() {
    let words = vec![String::from("alpha"), String::from("beta"), String::from("gamma")];
    println!("first call:  {}", total_length(&words));
    println!("second call: {}", total_length(&words));
    consume(words);
}
```

```text
first call:  14
second call: 14
consumed 3 words
```

Если добавить `total_length(&words);` после `consume(words);`, вы получите `error[E0382]: borrow of moved value: `words``.

Почему это работает: заимствовать можно сколько угодно раз, пока владелец держит значение. Как только значение переместилось в `consume`, владельца больше нет.

</details>

## Вопросы для самопроверки

1. Что такое ссылка и что создаёт `&s`?

<details>
<summary>Ответ</summary>

Ссылка — это значение, которое указывает на другое значение, не владея им. `&s` создаёт разделяемую (только для чтения) ссылку на `s`; `s` остаётся владельцем.

</details>

2. Что происходит, когда ссылка выходит из области видимости?

<details>
<summary>Ответ</summary>

Ничего не уничтожается, потому что ссылка не владеет значением. Значение уничтожает только конец области видимости владельца.

</details>

3. Сколько разделяемых ссылок на одно значение может существовать одновременно?

<details>
<summary>Ответ</summary>

Сколько угодно. Они только читают, поэтому не мешают друг другу.

</details>

4. Почему нельзя вызвать `push_str` у `&String`?

<details>
<summary>Ответ</summary>

Разделяемая ссылка доступна только для чтения. Для изменения значения нужна изменяемая ссылка `&mut String` (Урок 03.5).

</details>

5. Почему Rust отклоняет `fn f() -> &String { let s = String::from("x"); &s }`?

<details>
<summary>Ответ</summary>

`s` уничтожается, когда функция заканчивается, поэтому возвращённая ссылка висела бы. Верните вместо этого `String` во владение.

</details>

6. Когда заканчивается заимствование?

<details>
<summary>Ответ</summary>

На последнем использовании ссылки (NLL), а не в конце блока.

</details>

7. Что означает `*r`?

<details>
<summary>Ответ</summary>

Разыменование: значение, на которое указывает ссылка `r`.

</details>

## Итоги

- Ссылка (`&T`) указывает на значение, не владея им. Её создание называется заимствованием; владелец сохраняет значение.
- Функция, которая только читает, должна принимать `&T`. Вызывающий код пишет `&value` при вызове.
- Разделяемая ссылка доступна только для чтения, и их может существовать сколько угодно одновременно. Используйте `*r`, чтобы при необходимости получить значение за ссылкой.
- Ссылка никогда не переживает владельца: возврат ссылки на локальную переменную (`E0106`/`E0515`) или использование после уничтожения владельца (`E0597`) — ошибка компиляции.
- Пока заимствование живо, владельца нельзя переместить или уничтожить (`E0505`). Заимствование длится до своего последнего использования.
- Заимствование так же дёшево, как перемещение (8 байтов), и позволяет избежать клонов.

## Источники и что почитать

- [The Rust Programming Language — 4.2 References and Borrowing](https://doc.rust-lang.org/book/ch04-02-references-and-borrowing.html) — официальная глава, включая висячие ссылки.
- [Rust By Example — Borrowing](https://doc.rust-lang.org/rust-by-example/scope/borrow.html) — короткие запускаемые примеры.
- [Error code E0106](https://doc.rust-lang.org/error_codes/E0106.html) — отсутствует спецификатор времени жизни.
- [Error code E0597](https://doc.rust-lang.org/error_codes/E0597.html) — значение живёт недостаточно долго.
- [Error code E0505](https://doc.rust-lang.org/error_codes/E0505.html) — перемещение значения, которое заимствовано.

---

[🏠 Начало](../START-HERE.md) · [← 03.3 Перемещение, копирование и клонирование](03-move-copy-and-clone.md) · [📋 Этап 03](README.md) · [03.5 Изменяемые ссылки и правила заимствования →](05-mutable-references-and-the-borrow-rules.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
