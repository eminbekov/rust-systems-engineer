<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 05.3 Времена жизни](03-lifetimes.md) · [📋 Этап 05](README.md) · [05.5 Итераторы →](05-iterators.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/05-rust-intermediate/04-closures.md) <!-- nav -->

# Урок 05.4 — Замыкания

> **Этап 05 — Rust: средний уровень** · Урок 4 из 14 · ⏱ около 4 часов (2 занятия)

## Что вы узнаете

- Что такое замыкание, как его написать и чем оно отличается от обычной функции.
- Как замыкание *захватывает* переменные из места, где было создано: по ссылке, по изменяемой ссылке или перемещением.
- Три трейта замыканий — `Fn`, `FnMut` и `FnOnce` — и какой из них выбирает компилятор.
- Как передавать замыкания в функции и возвращать их из функций.
- Где вы уже используете замыкания в стандартной библиотеке (`sort_by_key`, `retain`, `map`, `unwrap_or_else`, …).

## Перед началом

- Вы прошли [Урок 05.3 — Времена жизни](03-lifetimes.md).
- Вы знаете трейты и ограничения трейтов ([Урок 05.2](02-traits.md)) и обобщения ([Урок 05.1](01-generics.md)). Замыкания используются через ограничения трейтов.
- Вы помните владение: перемещение, заимствование, изменяемое заимствование ([Уроки 03.2–03.5](../03-ownership-and-borrowing/02-ownership-rules.md)).
- У вас есть пакет `code/p05-lessons` из Урока 05.1.

## Новые слова

| Слово | Значение |
|---|---|
| **замыкание** (closure) | Анонимная (безымянная) функция, записываемая как `\|x\| x + 1`, которая может использовать переменные из места своего создания. |
| **захват** (capture) | Когда замыкание использует переменную, определённую снаружи, оно *захватывает* её: заимствует или забирает во владение. |
| **окружение** (environment) | Переменные вокруг замыкания, которые оно может захватить. |
| **`Fn`** | Трейт замыканий, которые только *читают* захваченное. Их можно вызывать много раз, даже одновременно. |
| **`FnMut`** | Трейт замыканий, которые *изменяют* захваченное. Их можно вызывать много раз, одно за другим. |
| **`FnOnce`** | Трейт замыканий, которые могут *поглотить* (переместить наружу) захваченное. Их можно вызвать только один раз. |
| **замыкание `move`** | Замыкание, записанное как `move \|…\| …`, которое забирает во владение всё, что захватывает. |
| **обратный вызов** (callback) | Функция (или замыкание), которую вы отдаёте другому коду, чтобы он вызвал вас в нужный момент. |
| **указатель на функцию** (function pointer) | Значение типа `fn(i32) -> i32`, указывающее на обычную функцию (или на замыкание, ничего не захватывающее). |
| **мемоизация** (memoization) | Сохранение результата вычисления, чтобы следующий вызов с тем же входом был мгновенным. |

## 1. Что такое замыкание

Замыкание — это маленькая функция без имени, которую можно сохранить в переменной или передать
другому коду. Параметры пишутся между двумя вертикальными чертами `|…|`, затем идёт тело:

```rust
fn add_one_fn(x: i32) -> i32 {
    x + 1
}

fn main() {
    let add_one_closure = |x: i32| -> i32 { x + 1 };
    let add_one_short = |x| x + 1;
    let add = |a, b| a + b;
    let hello = || println!("hello from a closure");

    println!("{}", add_one_fn(5));
    println!("{}", add_one_closure(5));
    println!("{}", add_one_short(5));
    println!("{}", add(2, 3));
    hello();

    let multiline = |x: i32| {
        let doubled = x * 2;
        doubled + 1
    };
    println!("{}", multiline(10));
}
```

**Что происходит:**

1. `add_one_fn` — обычная функция, для сравнения.
2. `|x: i32| -> i32 { x + 1 }` — замыкание в самой длинной форме: параметр с типом, тип результата,
   тело в блоке.
3. `|x| x + 1` — короткая форма. Типы необязательны, потому что компилятор выводит их из того, как
   замыкание используется. Тело однострочного замыкания не требует фигурных скобок.
4. `|| println!(…)` без параметров, поэтому черты пустые.
5. Замыкание из нескольких строк требует скобок `{ … }`; последнее выражение (без `;`) — результат.
6. Замыкание вызывается так же, как функция: `add(2, 3)`.

```text
6
6
6
5
hello from a closure
21
```

Пока замыкание выглядит как более короткая функция. Настоящая разница — в следующем разделе.

### У замыкания один фиксированный тип

Типы параметров выводятся **один раз**, по первому использованию. Замыкание не может быть
обобщённым:

```rust
fn main() {
    let identity = |x| x;
    let s = identity(String::from("text"));
    let n = identity(5);
    println!("{s} {n}");
}
```

```text
error[E0308]: mismatched types
 --> examples/infer_err.rs:4:22
  |
4 |     let n = identity(5);
  |             -------- ^ expected `String`, found integer
  |             |
  |             arguments to this function are incorrect
  |
note: expected because the closure was earlier called with an argument of type `String`
```

**Что это значит:** первый вызов сделал `x` строкой `String`; второй вызов с числом ему противоречит.
Если одна операция нужна для многих типов, напишите обобщённую функцию (Урок 05.1).

## 2. Захват окружения

Обычная функция может использовать только свои параметры. Замыкание может использовать ещё и
переменные, которые существуют вокруг него. Это называется **захватом**. Компилятор выбирает самый
слабый вид захвата, при котором тело работает.

### Захват по общей ссылке

```rust
fn main() {
    let name = String::from("Dilnoza");
    let hello = || println!("hello, {name}");
    hello();
    hello();
    println!("name is still mine: {name}");
}
```

**Что происходит:** замыкание только *читает* `name`, поэтому заимствует её через `&name`. Его можно
вызывать много раз, а `name` можно по-прежнему использовать снаружи, потому что много читателей
разрешено (правила заимствования из Урока 03.5).

```text
hello, Dilnoza
hello, Dilnoza
name is still mine: Dilnoza
```

### Захват по изменяемой ссылке

```rust
fn main() {
    let mut count = 0;
    let mut increment = || {
        count += 1;
        println!("count is now {count}");
    };
    increment();
    increment();
    println!("final count: {count}");
}
```

**Что происходит:**

1. Замыкание *изменяет* `count`, поэтому заимствует её через `&mut count`.
2. Поскольку оно меняет захваченное состояние, саму переменную замыкания нужно объявить `mut`:
   `let mut increment`. (Вызов меняет внутреннее состояние, как вызов метода с `&mut self`.)
3. Изменяемое заимствование длится, пока используется замыкание. После последнего вызова `count`
   снова свободна.

```text
count is now 1
count is now 2
final count: 2
```

Если использовать `count`, пока замыкание ещё держит изменяемое заимствование, действуют правила
заимствования:

```rust
fn main() {
    let mut count = 0;
    let mut increment = || count += 1;
    println!("before: {count}");
    increment();
}
```

```text
error[E0502]: cannot borrow `count` as immutable because it is also borrowed as mutable
 --> examples/capture_mut_err.rs:4:24
  |
3 |     let mut increment = || count += 1;
  |                         -- ----- first borrow occurs due to use of `count` in closure
  |                         |
  |                         mutable borrow occurs here
4 |     println!("before: {count}");
  |                        ^^^^^ immutable borrow occurs here
5 |     increment();
  |     --------- mutable borrow later used here
```

**Что это значит:** замыкание держит `&mut count` со строки 3 до последнего вызова в строке 5.
Чтение `count` между ними нарушило бы правило «один пишущий или много читающих». Исправление:
сначала закончить с замыканием или вывести значение после последнего вызова.

### Захват перемещением

Добавьте ключевое слово `move` перед чертами — и замыкание **забирает во владение** каждую
переменную, которую использует:

```rust
fn main() {
    let data = vec![1, 2, 3];
    let owns_data = move || println!("data inside the closure: {:?}", data);
    owns_data();
    owns_data();
}
```

```text
data inside the closure: [1, 2, 3]
data inside the closure: [1, 2, 3]
```

После перемещения внешней переменной больше нет (Урок 03.3):

```rust
fn main() {
    let data = vec![1, 2, 3];
    let owns_data = move || println!("{:?}", data);
    owns_data();
    println!("{:?}", data);
}
```

```text
error[E0382]: borrow of moved value: `data`
 --> examples/capture_move_err.rs:5:22
  |
2 |     let data = vec![1, 2, 3];
  |         ---- move occurs because `data` has type `Vec<i32>`, which does not implement the `Copy` trait
3 |     let owns_data = move || println!("{:?}", data);
  |                     -------                  ---- variable moved due to use in closure
  |                     |
  |                     value moved into closure here
4 |     owns_data();
5 |     println!("{:?}", data);
  |                      ^^^^ value borrowed here after move
```

**Что это значит:** `data` была перемещена в замыкание в строке 3. Компилятор даже предлагает
клонировать (`data.clone()`) перед перемещением, если вам действительно нужны обе. Для типов с
`Copy` (числа, `bool`, `char`) `move` копирует значение, и оригинал остаётся пригодным.

> 💡 **Совет:** `move` нужен в основном тогда, когда замыкание должно жить **дольше**, чем функция,
> которая его создала. Два типичных случая: возврат замыкания (Раздел 4) и передача замыкания в новый
> поток (Урок 05.10).

## 3. Три трейта замыканий: `Fn`, `FnMut`, `FnOnce`

Как функция говорит «дай мне замыкание»? Через ограничение трейта. Каждое замыкание автоматически
реализует один, два или все три из этих трейтов в зависимости от того, **что тело делает с
захваченными значениями** (а не от того, написали ли вы `move`):

| Трейт | Тело… | Можно вызвать | Образ |
|---|---|---|---|
| `FnOnce` | может *переместить захваченное значение наружу* (поглотить его) | один раз | билет в один конец |
| `FnMut` | *изменяет* захваченное значение, но не перемещает его наружу | много раз, по одному | тетрадь, в которой пишут |
| `Fn` | только *читает* захваченные значения (или ничего не захватывает) | много раз, даже совместно | справочник |

Трейты образуют цепочку: каждое замыкание `Fn` одновременно `FnMut`, а каждое `FnMut` — `FnOnce`.
Поэтому функция, просящая `FnOnce`, принимает любое замыкание, а просящая `Fn` — меньше всего.

```rust
fn call_twice<F: Fn()>(f: F) {
    f();
    f();
}

fn call_three_times<F: FnMut()>(mut f: F) {
    f();
    f();
    f();
}

fn call_once<F: FnOnce() -> String>(f: F) -> String {
    f()
}

fn main() {
    let name = String::from("Rust");
    call_twice(|| println!("Fn: reads {name}"));

    let mut total = 0;
    call_three_times(|| total += 10);
    println!("FnMut: total = {total}");

    let owned = String::from("moved out");
    let result = call_once(move || owned);
    println!("FnOnce: got {result}");
}
```

**Что происходит:**

1. `F: Fn()` означает «замыкание, которое ничего не принимает, ничего не возвращает и только читает».
   Первое замыкание читает `name`, поэтому подходит.
2. `F: FnMut()` допускает замыкание, меняющее своё окружение. Параметр записан `mut f`, потому что
   для вызова `FnMut` нужна изменяемая привязка.
3. `F: FnOnce() -> String` принимает замыкание, возвращающее `String`. Тело `owned` *перемещает*
   строку наружу как результат, поэтому замыкание можно вызвать только один раз.
4. Запись `Fn(A, B) -> R` в ограничении перечисляет типы аргументов и тип результата.

```text
Fn: reads Rust
Fn: reads Rust
FnMut: total = 30
FnOnce: got moved out
```

Компилятор проверяет, что замыкание подходит под запрошенный вами трейт:

```rust
fn call_twice<F: Fn()>(f: F) {
    f();
    f();
}

fn main() {
    let s = String::from("hello");
    call_twice(move || drop(s));
}
```

```text
error[E0507]: cannot move out of `s`, a captured variable in an `Fn` closure
 --> examples/fnonce_err.rs:8:29
  |
7 |     let s = String::from("hello");
  |         -   --------------------- move occurs because `s` has type `String`, which does not implement the `Copy` trait
  |         |
  |         captured outer variable
8 |     call_twice(move || drop(s));
  |                -------      ^ `s` is moved here
  |                |
  |                captured by this `Fn` closure
  |
help: `Fn` and `FnMut` closures require captured values to be able to be consumed multiple times, but `FnOnce` closures may consume them only once
```

**Что это значит:** `drop(s)` поглощает `s`, значит, замыкание — `FnOnce`, а `call_twice` вызывает его
дважды. То же правило видно, когда вы сами вызываете поглощающее замыкание:

```rust
fn main() {
    let s = String::from("hello");
    let consume = move || s;
    let a = consume();
    let b = consume();
    println!("{a} {b}");
}
```

```text
error[E0382]: use of moved value: `consume`
 --> examples/fnonce_twice_err.rs:5:13
  |
4 |     let a = consume();
  |             --------- `consume` moved due to this call
5 |     let b = consume();
  |             ^^^^^^^ value used here after move
  |
note: closure cannot be invoked more than once because it moves the variable `s` out of its environment
```

> 📏 **Соглашение:** когда пишете функцию, принимающую замыкание, просите **самый слабый** трейт,
> который подходит: `FnOnce`, если вызываете один раз; `FnMut`, если вызываете несколько раз и оно
> может менять состояние; `Fn` — только если оно должно вызываться одновременно или вам нужна
> уверенность, что оно ничего не меняет. Чем слабее ваше требование, тем больше замыканий смогут
> передать вызывающие.

## 4. Замыкания как аргументы и как возвращаемые значения

Замыкания как аргументы вы уже видели. Чтобы **вернуть** замыкание, используйте `impl Fn…` (Урок
05.2) и `move`, чтобы замыкание владело тем, что ему нужно:

```rust
fn make_adder(n: i32) -> impl Fn(i32) -> i32 {
    move |x| x + n
}

fn make_counter() -> impl FnMut() -> u32 {
    let mut count = 0;
    move || {
        count += 1;
        count
    }
}

fn compose<A, B, C>(f: impl Fn(A) -> B, g: impl Fn(B) -> C) -> impl Fn(A) -> C {
    move |x| g(f(x))
}

fn main() {
    let add5 = make_adder(5);
    println!("{} {}", add5(1), add5(10));

    let mut counter = make_counter();
    println!("{} {} {}", counter(), counter(), counter());

    let double_then_text = compose(|x: i32| x * 2, |y: i32| format!("<{y}>"));
    println!("{}", double_then_text(21));
}
```

**Что происходит:**

1. `make_adder(5)` создаёт замыкание, которое помнит `n = 5`. Каждый вызов `make_adder` делает новое
   замыкание со своим `n`.
2. `make_counter` возвращает `FnMut`. Замыкание владеет собственным `count` (перемещённым через
   `move`), и он сохраняется между вызовами, хотя `make_counter` давно завершилась.
3. `compose` принимает два замыкания и возвращает новое, которое применяет `f`, а затем `g`. Так из
   маленьких кусочков собирают большие.
4. `let mut counter` нужно, потому что вызов `FnMut` изменяет его.

```text
6 15
1 2 3
<42>
```

Без `move` замыкание заимствовало бы `n`, но `n` умирает, когда `make_adder` возвращается:

```rust
fn make_adder(n: i32) -> impl Fn(i32) -> i32 {
    |x| x + n
}
```

```text
error[E0373]: closure may outlive the current function, but it borrows `n`, which is owned by the current function
 --> examples/return_closure_err.rs:2:5
  |
2 |     |x| x + n
  |     ^^^     - `n` is borrowed here
  |     |
  |     may outlive borrowed value `n`
  |
note: closure is returned here
help: to force the closure to take ownership of `n` (and any other referenced variables), use the `move` keyword
  |
2 |     move |x| x + n
  |     ++++
```

**Что это значит:** это проблема висячей ссылки из Урока 05.3 в новом месте. Решение — ровно то, что
предлагает компилятор: добавить `move`.

### Замыкание, которое передают и вызывают позже: `retry`

Полезный системный пример: повторять операцию, которая может не удаться, например сетевой вызов.
Операция — это замыкание, вызываемое несколько раз, и ей разрешено менять состояние (значит,
`FnMut`):

```rust
fn retry<F>(mut operation: F, attempts: u32) -> Result<u32, String>
where
    F: FnMut() -> Result<u32, String>,
{
    let mut last_error = String::from("no attempts made");
    for attempt in 1..=attempts {
        match operation() {
            Ok(value) => return Ok(value),
            Err(e) => {
                println!("attempt {attempt} failed: {e}");
                last_error = e;
            }
        }
    }
    Err(last_error)
}

fn main() {
    let mut calls = 0;
    let result = retry(
        || {
            calls += 1;
            if calls < 3 {
                Err(String::from("connection refused"))
            } else {
                Ok(calls)
            }
        },
        5,
    );
    println!("{:?} after {} calls", result, calls);

    let hopeless = retry(|| Err(String::from("always fails")), 2);
    println!("{:?}", hopeless);
}
```

**Что происходит:**

1. `1..=attempts` — диапазон от 1 до `attempts` включая конец (Урок 02.5).
2. `retry` вызывает `operation()`. При успехе она сразу возвращает результат; при неудаче печатает
   ошибку, запоминает её и пробует снова.
3. Первое замыкание дважды терпит неудачу, затем успешно срабатывает на вызове 3. Оно меняет
   захваченную `calls`, поэтому это `FnMut`.
4. После завершения `retry` переменную `calls` снова можно читать, потому что замыкание было
   перемещено в `retry` и исчезло.
5. Второе замыкание никогда не завершается успехом, поэтому `retry` сдаётся и возвращает последнюю
   ошибку.

```text
attempt 1 failed: connection refused
attempt 2 failed: connection refused
Ok(3) after 3 calls
attempt 1 failed: always fails
attempt 2 failed: always fails
Err("always fails")
```

(Настоящий код повторов ещё и ждёт между попытками и сдаётся на ошибках, которые повтор не исправит.
Этап 18 возвращается к этой идее под названием *backoff* — «задержка с нарастанием».)

## 5. Замыкания в стандартной библиотеке

Программные интерфейсы, принимающие замыкания, вы использовали уже давно. Вот обзор самых частых:

```rust
use std::collections::HashMap;

fn main() {
    let mut words = vec!["pear", "fig", "banana", "kiwi"];
    words.sort_by_key(|w| w.len());
    println!("by length: {:?}", words);
    words.sort_by(|a, b| b.cmp(a));
    println!("reverse alphabetical: {:?}", words);

    let mut numbers = vec![1, 2, 3, 4, 5, 6];
    numbers.retain(|n| n % 2 == 0);
    println!("only even: {:?}", numbers);

    let maybe: Option<i32> = Some(4);
    println!("{:?}", maybe.map(|n| n * 10));
    let nothing: Option<i32> = None;
    println!("{}", nothing.unwrap_or_else(|| -1));
    println!("{}", maybe.map_or(0, |n| n + 1));
    println!("{:?}", maybe.filter(|n| *n > 10));

    let parsed: Result<i32, String> = "x1".parse::<i32>().map_err(|e| format!("bad number: {e}"));
    println!("{:?}", parsed);

    let mut groups: HashMap<usize, Vec<&str>> = HashMap::new();
    for w in ["a", "bb", "cc", "d"] {
        groups.entry(w.len()).or_insert_with(Vec::new).push(w);
    }
    println!("{:?}", groups.get(&2));

    println!("{:?}", (5 > 3).then(|| "yes"));
}
```

**Что происходит:**

1. `sort_by_key(|w| w.len())` — замыкание вычисляет ключ сортировки для каждого элемента. Слова с
   равными ключами сохраняют относительный порядок («pear» перед «kiwi», как на входе).
2. `sort_by(|a, b| b.cmp(a))` — замыкание сравнивает два элемента и возвращает `Ordering`; обмен `a`
   и `b` местами переворачивает порядок.
3. `retain(|n| n % 2 == 0)` — оставить только те элементы, для которых замыкание вернёт `true`.
4. `Option::map` преобразует значение внутри `Some`; `None` остаётся `None`. `map_or(default, f)`
   даёт значение по умолчанию для `None`. `filter` превращает `Some(x)` в `None`, если замыкание
   говорит `false`.
5. `unwrap_or_else(|| -1)` — замыкание выполняется **только если** значение — `None`. Сравните с
   `unwrap_or(expensive())`, где аргумент вычисляется всегда. Используйте `_else`, когда запасное
   значение стоит дорого.
6. `map_err` преобразует ошибку внутри `Result` (та же идея, что `From` в Уроке 05.2).
7. `entry(key).or_insert_with(Vec::new)` — имя функции можно использовать там, где ожидается
   замыкание. Новый вектор создаётся только когда ключа нет.
8. `bool::then(|| value)` возвращает `Some(value)`, если bool истинно, иначе `None`.

```text
by length: ["fig", "pear", "kiwi", "banana"]
reverse alphabetical: ["pear", "kiwi", "fig", "banana"]
only even: [2, 4, 6]
Some(40)
-1
5
None
Err("bad number: invalid digit found in string")
Some(["bb", "cc"])
Some("yes")
```

> 💡 **Совет:** приём «передать замыкание вместо значения» нужен для **ленивых** вычислений (делать
> работу только при необходимости) и для **настройки поведения** (порядок сортировки, правило
> фильтра). В следующем уроке мы объединим его с итераторами, где замыкания встречаются повсюду.

## 6. Замыкания в структурах и указатели на функции

Структура может хранить замыкание. Его тип вам неизвестен, поэтому структура обобщена по нему:

```rust
use std::collections::HashMap;

struct Memo<F: Fn(u64) -> u64> {
    func: F,
    cache: HashMap<u64, u64>,
    calls: u32,
}

impl<F: Fn(u64) -> u64> Memo<F> {
    fn new(func: F) -> Self {
        Memo { func, cache: HashMap::new(), calls: 0 }
    }

    fn get(&mut self, arg: u64) -> u64 {
        if let Some(&value) = self.cache.get(&arg) {
            return value;
        }
        self.calls += 1;
        let value = (self.func)(arg);
        self.cache.insert(arg, value);
        value
    }
}

fn main() {
    let mut square = Memo::new(|x| {
        println!("  computing {x} * {x}");
        x * x
    });
    println!("{}", square.get(12));
    println!("{}", square.get(12));
    println!("{}", square.get(3));
    println!("closure ran {} times", square.calls);
}
```

**Что происходит:**

1. `Memo<F: Fn(u64) -> u64>` хранит замыкание `func` и `HashMap` прежних ответов. Это
   **мемоизация**: медленное вычисление выполняется один раз для каждого входа.
2. `(self.func)(arg)` вызывает замыкание, хранящееся в поле. Скобки обязательны, иначе Rust решит,
   что вы вызываете *метод* с именем `func`.
3. Второй `get(12)` находит ответ в `cache`, поэтому «computing 12 * 12» печатается только один раз.

```text
  computing 12 * 12
144
144
  computing 3 * 3
9
closure ran 2 times
```

**Указатели на функции.** Обычную функцию тоже можно передать как значение. Её тип записывается
`fn(i32) -> i32` (с маленькой `fn`; не путайте с трейтом `Fn`):

```rust
fn double(x: i32) -> i32 {
    x * 2
}

fn apply(f: fn(i32) -> i32, x: i32) -> i32 {
    f(x)
}

fn apply_generic<F: Fn(i32) -> i32>(f: F, x: i32) -> i32 {
    f(x)
}

fn main() {
    println!("{}", apply(double, 21));
    println!("{}", apply(|x| x + 1, 41)); // a closure without captures becomes a fn pointer
    let offset = 100;
    println!("{}", apply_generic(|x| x + offset, 1));
    println!("{}", apply_generic(double, 4));
}
```

```text
42
42
101
8
```

**Что происходит:** `apply` принимает только указатели на функции. Замыкание, ничего не
захватывающее, можно превратить в такой указатель, а замыкание с захватом (`offset`) — нельзя; для
него нужна обобщённая форма `Fn`. Поскольку указатели на функции тоже реализуют `Fn`, `apply_generic`
принимает оба варианта. **Предпочитайте обобщённые ограничения `Fn`** в собственном коде;
указатели `fn(...)` нужны при общении с кодом на C (Урок 06.10).

> ⚡ **Производительность:** обобщённая `F: Fn…` мономорфизируется (Урок 05.1): каждое замыкание
> получает свою специализированную копию функции, и вызов даже может быть встроен, поэтому
> замыкания работают так же быстро, как код, написанный вручную. Поэтому же нельзя напрямую
> положить два *разных* замыкания в один `Vec`: это разные типы. Для `Vec` из смешанных замыканий
> нужны трейт-объекты (`Box<dyn Fn…>`), о них рассказывает Урок 06.1.

## Частые ошибки

**1. Использование переменной, пока замыкание держит изменяемое заимствование** (`E0502`, раздел 2).
Сначала закончите с замыканием или перестройте код.

**2. Использование переменной после `move`** (`E0382`, раздел 2). Клонируйте перед перемещением или
не перемещайте.

**3. Замыкание, поглощающее захваченное, но вызываемое дважды** (`E0507` / `E0382`, раздел 3).
Попросите `FnOnce` в своей функции или измените замыкание так, чтобы оно не перемещало значение
наружу (клонируйте его или только читайте).

**4. Возврат замыкания без `move`** (`E0373`, раздел 4). Добавьте `move`.

**5. Вызов замыкания с разными типами** (`E0308`, раздел 1). У замыкания одна сигнатура; для многих
типов используйте обобщённую функцию.

**6. Забытый `mut` у переменной замыкания**, которое меняет своё состояние:

```rust
fn main() {
    let mut count = 0;
    let increment = || count += 1;
    increment();
}
```

```text
error[E0596]: cannot borrow `increment` as mutable, as it is not declared as mutable
```

Объявите `let mut increment`.

## Попробуйте сами

На этот раз вы потренируетесь **подготавливать к коммиту только часть изменений**: так коммиты
остаются сфокусированными.

1. Начните ветку и создайте небольшой пример:

   ```bash
   cd ~/projects/rust
   git switch main
   git switch -c lesson/05-04-closures
   ```

   Создайте `code/p05-lessons/examples/closures_demo.rs`:

   ```rust
   fn main() {
       let price = 10;
       let total = price * 3;
       println!("total: {total}");
       println!("done");
   }
   ```

   Сделайте коммит: `git add .` и `git commit -m "feat(p05): closures demo"`.

2. Теперь внесите в тот же файл **два несвязанных изменения**: замените `price * 3` на `price * 4`
   и добавьте в конец новую функцию:

   ```rust
   fn debug_dump(value: i32) {
       println!("debug: {value}");
   }
   ```

3. Вы хотите закоммитить только первое изменение, потому что отладочный помощник ещё не готов.
   Используйте **интерактивную подготовку**: `git add -p` (p от *patch*, «заплатка»). Git показывает
   каждый изменённый кусок (*hunk*) и спрашивает, что с ним делать. Оба изменения расположены рядом,
   поэтому Git показывает один кусок; нажмите `s`, чтобы *разделить* (split) его:

   ```bash
   git add -p
   ```

   ```text
   (1/1) Stage this hunk [y,n,q,a,d,s,e,p,P,?]? s
   Split into 2 hunks.
   @@ -1,6 +1,6 @@
    fn main() {
        let price = 10;
   -    let total = price * 3;
   +    let total = price * 4;
        println!("total: {total}");
        println!("done");
    }
   (1/2) Stage this hunk [y,n,q,a,d,k,K,j,J,g,/,e,p,P,?]? y
   @@ -4,3 +4,7 @@
   ...
   (2/2) Stage this hunk [y,n,q,a,d,K,J,g,/,e,p,P,?]? n
   ```

   Буквы означают: `y` — подготовить этот кусок, `n` — не подготавливать, `s` — разделить на
   меньшие куски, `q` — выйти, `?` — справка. Теперь проверьте, что подготовлено, а что нет:

   ```bash
   git status --short
   git diff --staged --stat
   ```

   ```text
   MM code/p05-lessons/examples/closures_demo.rs
    code/p05-lessons/examples/closures_demo.rs | 2 +-
    1 file changed, 1 insertion(+), 1 deletion(-)
   ```

   `MM` значит: в файле есть изменения в области подготовки *и* ещё изменения, которые не
   подготовлены. Закоммитьте только подготовленную часть:

   ```bash
   git commit -m "fix(p05): charge four units instead of three"
   git add .
   git commit -m "feat(p05): add debug_dump helper"
   ```

   Теперь у вас два небольших коммита с честными сообщениями вместо одного с названием «всякое».

4. Завершите урок как обычно: добавьте примеры `retry` и `make_counter`, закоммитьте каждый, слейте в
   `main`.

> 📏 **Соглашение:** один коммит — одна идея. Коммиты, где смешаны исправление ошибки,
> переименование и новая возможность, трудно проверять и трудно откатывать. `git add -p` позволяет
> разделить их, даже если вы уже всё отредактировали.

## Упражнения

### Упражнение 1 — Фабрика умножителей (легко)

Напишите `fn make_multiplier(factor: i32) -> impl Fn(i32) -> i32` так, чтобы `make_multiplier(3)(4)`
давало 12. Напишите тесты: один на результат и один, показывающий, что два умножителя не влияют друг
на друга.

<details>
<summary>Подсказка</summary>

Возвращаемое замыкание должно владеть `factor`, поэтому напишите `move |x| x * factor`.

</details>

<details>
<summary>Ответ</summary>

```rust
fn make_multiplier(factor: i32) -> impl Fn(i32) -> i32 {
    move |x| x * factor
}

fn main() {
    let triple = make_multiplier(3);
    println!("{} {}", triple(4), triple(-2));
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn multiplies_by_the_captured_factor() {
        let triple = make_multiplier(3);
        assert_eq!(triple(4), 12);
        assert_eq!(triple(-2), -6);
    }

    #[test]
    fn two_multipliers_do_not_interfere() {
        let by_two = make_multiplier(2);
        let by_ten = make_multiplier(10);
        assert_eq!(by_two(5), 10);
        assert_eq!(by_ten(5), 50);
    }
}
```

```text
12 -6
```

Почему это работает: каждый вызов `make_multiplier` создаёт новое замыкание, владеющее собственной
копией `factor`. Оно только читает её, поэтому замыкание — `Fn`.

</details>

### Упражнение 2 — Сортировка по своему правилу (легко)

Напишите `fn sort_words(words: &mut [&str])`, которая сортирует по длине (короткие первыми), а слова
одинаковой длины — по алфавиту. Напишите тест с `["pear", "fig", "apple", "kiwi", "date"]`.

<details>
<summary>Подсказка</summary>

`words.sort_by(|a, b| …)`. Сравните длины через `a.len().cmp(&b.len())`. У `Ordering` есть метод
`.then(other)`, который использует `other` только если первое сравнение говорит «равно»:
`.then(a.cmp(b))`.

</details>

<details>
<summary>Ответ</summary>

```rust
fn sort_words(words: &mut [&str]) {
    words.sort_by(|a, b| a.len().cmp(&b.len()).then(a.cmp(b)));
}

fn main() {
    let mut words = vec!["pear", "fig", "apple", "kiwi", "date"];
    sort_words(&mut words);
    println!("{:?}", words);
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn shorter_first_then_alphabetical() {
        let mut words = vec!["pear", "fig", "apple", "kiwi", "date"];
        sort_words(&mut words);
        assert_eq!(words, vec!["fig", "date", "kiwi", "pear", "apple"]);
    }
}
```

```text
["fig", "date", "kiwi", "pear", "apple"]
```

Почему это работает: три слова из четырёх букв упорядочены по алфавиту («date», «kiwi», «pear»),
потому что сравнение длин даёт «равно», и `.then` переходит к сравнению текста.

</details>

### Упражнение 3 — Счётчик со своим состоянием (средне)

Напишите `fn make_counter() -> impl FnMut() -> u32`, которая при первом вызове возвращает 1, при втором
— 2 и так далее. Напишите тесты: счёт вверх и два счётчика, не разделяющих состояние.

<details>
<summary>Подсказка</summary>

Объявите `let mut count = 0;` внутри `make_counter`, затем верните `move || { count += 1; count }`.
В тестах переменная замыкания должна быть `let mut c = make_counter();`.

</details>

<details>
<summary>Ответ</summary>

```rust
fn make_counter() -> impl FnMut() -> u32 {
    let mut count = 0;
    move || {
        count += 1;
        count
    }
}

fn main() {
    let mut a = make_counter();
    let mut b = make_counter();
    println!("{} {} {}", a(), a(), b());
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn counts_up_from_one() {
        let mut c = make_counter();
        assert_eq!(c(), 1);
        assert_eq!(c(), 2);
        assert_eq!(c(), 3);
    }

    #[test]
    fn each_counter_has_its_own_state() {
        let mut a = make_counter();
        let mut b = make_counter();
        a();
        a();
        assert_eq!(b(), 1);
    }
}
```

```text
1 2 1
```

Почему это работает: `count` перемещается в замыкание, поэтому каждое замыкание, созданное
`make_counter`, несёт собственный `count`. Его изменение делает замыкание `FnMut`.

</details>

### Упражнение 4 — Напишите и протестируйте `retry` (сложно)

Напишите `fn retry<F>(operation: F, attempts: u32) -> Result<u32, String>` для
`FnMut() -> Result<u32, String>`. Она вызывает операцию до `attempts` раз, возвращает первый `Ok`, а
иначе возвращает **последнюю** ошибку. При `attempts == 0` она не должна вызывать операцию и должна
вернуть `Err`. Напишите три теста: успех на третьей попытке (проверьте число вызовов), отказ с
последней ошибкой и ноль попыток.

<details>
<summary>Подсказка</summary>

Заведите `let mut last_error = String::from("no attempts made");` и цикл `for _ in 0..attempts`.
Внутри `match operation()` с `Ok(v) => return Ok(v)` и `Err(e) => last_error = e`. Объявите параметр
как `mut operation`. В тестах замыкание меняет внешний счётчик `calls`; читайте `calls` после
возврата из `retry`.

</details>

<details>
<summary>Ответ</summary>

```rust
fn retry<F>(mut operation: F, attempts: u32) -> Result<u32, String>
where
    F: FnMut() -> Result<u32, String>,
{
    let mut last_error = String::from("no attempts made");
    for _ in 0..attempts {
        match operation() {
            Ok(value) => return Ok(value),
            Err(e) => last_error = e,
        }
    }
    Err(last_error)
}

fn main() {
    let mut calls = 0;
    let result = retry(
        || {
            calls += 1;
            if calls < 3 {
                Err(String::from("not yet"))
            } else {
                Ok(calls)
            }
        },
        5,
    );
    println!("{:?}", result);
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn succeeds_on_the_third_try() {
        let mut calls = 0;
        let result = retry(
            || {
                calls += 1;
                if calls < 3 { Err(String::from("no")) } else { Ok(calls) }
            },
            5,
        );
        assert_eq!(result, Ok(3));
        assert_eq!(calls, 3);
    }

    #[test]
    fn gives_up_and_returns_the_last_error() {
        let mut calls = 0;
        let result = retry(
            || {
                calls += 1;
                Err(format!("error #{calls}"))
            },
            2,
        );
        assert_eq!(result, Err(String::from("error #2")));
    }

    #[test]
    fn zero_attempts_never_calls_the_operation() {
        let mut calls = 0;
        let result = retry(
            || {
                calls += 1;
                Ok(1)
            },
            0,
        );
        assert!(result.is_err());
        assert_eq!(calls, 0);
    }
}
```

```text
Ok(3)
```

Почему это работает: `retry` принимает замыкание по значению (перемещённым), поэтому после вызова
`calls` снова можно читать. Тесты проверяют поведение (возвращаемые значения и число вызовов), а не
внутреннее устройство.

</details>

## Вопросы для самопроверки

1. Чем замыкание отличается от функции?

<details>
<summary>Ответ</summary>

Замыкание анонимно, обычно имеет выводимые типы и может использовать («захватывать») переменные из окружающего кода. Функция может использовать только свои параметры и глобальные элементы.

</details>

2. Когда обязательно ключевое слово `move`?

<details>
<summary>Ответ</summary>

Когда замыкание должно владеть захваченными значениями, потому что живёт дольше места, где они находятся, например когда вы возвращаете замыкание из функции или отправляете его в другой поток. `move` заставляет замыкание забрать во владение всё, что оно использует.

</details>

3. Какой из трейтов `Fn`, `FnMut`, `FnOnce` реализует замыкание, тело которого перемещает наружу захваченную `String`?

<details>
<summary>Ответ</summary>

Только `FnOnce`. Перемещение значения наружу поглощает его, поэтому замыкание можно выполнить лишь один раз.

</details>

4. Вы пишете функцию, которая вызывает своё замыкание-аргумент несколько раз, и замыкание может менять счётчик. Какое ограничение выбрать?

<details>
<summary>Ответ</summary>

`FnMut`. (`Fn` отвергло бы замыкания, меняющие состояние; `FnOnce` позволило бы вызвать только один раз.)

</details>

5. Почему `unwrap_or_else(|| compute())` иногда лучше, чем `unwrap_or(compute())`?

<details>
<summary>Ответ</summary>

`unwrap_or(compute())` всегда выполняет `compute()` до вызова, даже если значение — `Some`. С `unwrap_or_else` замыкание выполняется только если значение — `None`, так что работа делается лишь при необходимости.

</details>

6. Почему `Vec` не может хранить два разных замыкания, даже если у обоих сигнатура `Fn(i32) -> i32`?

<details>
<summary>Ответ</summary>

У каждого замыкания свой уникальный анонимный тип. `Vec<T>` хранит значения одного типа. Чтобы смешивать замыкания, нужны трейт-объекты вроде `Box<dyn Fn(i32) -> i32>` (Урок 06.1).

</details>

## Итоги

- Замыкание — анонимная функция вида `|args| body`. Типы обычно выводятся и фиксируются после первого использования.
- Замыкания захватывают окружение по общей ссылке, по изменяемой ссылке или перемещением (`move`); компилятор выбирает самый слабый подходящий способ.
- `Fn` читает, `FnMut` изменяет, `FnOnce` поглощает. Просите у функции самый слабый трейт, который ей нужен.
- Чтобы вернуть замыкание, используйте `impl Fn…` и `move`.
- Стандартная библиотека принимает замыкания повсюду: `sort_by_key`, `retain`, `map`, `unwrap_or_else`, `or_insert_with`.
- Храните замыкание в структуре через обобщённый параметр `F: Fn…`; предпочитайте ограничения трейтов указателям `fn`.
- Привычка с Git: `git add -p` подготавливает к коммиту часть файла, так что каждый коммит содержит ровно одну идею.

## Источники и что почитать

- [The Rust Programming Language — 13.1 Closures](https://doc.rust-lang.org/book/ch13-01-closures.html) — официальная глава о захвате и трейтах `Fn`.
- [Rust by Example — Closures](https://doc.rust-lang.org/rust-by-example/fn/closures.html) — короткие примеры захвата, входных параметров и результата.
- [std::ops::FnMut](https://doc.rust-lang.org/std/ops/trait.FnMut.html) — документация трейта и связи между тремя трейтами.
- [Документация Git — git add (интерактивный режим и --patch)](https://git-scm.com/docs/git-add) — все ответы по `git add -p`.

---

[🏠 Начало](../START-HERE.md) · [← 05.3 Времена жизни](03-lifetimes.md) · [📋 Этап 05](README.md) · [05.5 Итераторы →](05-iterators.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
