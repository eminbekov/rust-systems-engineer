<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 06.4 Внутренняя изменяемость](04-interior-mutability.md) · [📋 Этап 06](README.md) · [06.6 Небезопасный Rust и Miri →](06-unsafe-rust-and-miri.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/06-rust-advanced/05-lifetimes-in-depth.md) <!-- nav -->

# Урок 06.5 — Времена жизни подробно

> **Этап 06 — Rust: продвинутый уровень** · Урок 5 из 12 · ⏱ около 4 часов (2 занятия)

## Что вы узнаете

- Точные правила пропуска времён жизни (elision), чтобы знать, когда компилятор подставляет их за вас.
- Как использовать несколько времён жизни в одной сигнатуре и когда это важно.
- Как структуры и блоки `impl` используют времена жизни и чем `&'a T` отличается от `&self`.
- Что на самом деле означают `'static` и `T: 'static` (и почему `thread::spawn` их требует).
- Подтипизацию и вариантность: почему `&'static str` можно использовать как `&'a str`, а `&mut &'static str` сократить нельзя.
- Ограничения трейтов высшего ранга (`for<'a>`) и то, как `impl Trait` захватывает времена жизни в редакции 2024.

## Перед началом

- Вы прошли [Урок 06.4 — Внутренняя изменяемость](04-interior-mutability.md).
- Вы знаете основы времён жизни по [Уроку 05.3 — Времена жизни](../05-rust-intermediate/03-lifetimes.md): синтаксис `'a`, структуры со ссылками и идею, что ссылка не должна пережить свои данные.
- Вы знаете замыкания ([Урок 05.4](../05-rust-intermediate/04-closures.md)) и потоки ([Урок 05.10](../05-rust-intermediate/10-threads-channels-and-shared-state.md)).
- Создайте учебный проект: `cargo new code/p06-lifetimes`.

## Новые слова

| Слово | Значение |
|---|---|
| **время жизни** (lifetime) | Участок программы, в течение которого ссылка действительна. Записывается `'a`, `'b`, ... |
| **пропуск времён жизни** (lifetime elision) | Правила компилятора, позволяющие в частых случаях не писать времена жизни в сигнатурах. |
| **переживает** (outlives) | `'long: 'short` означает «`'long` длится не меньше, чем `'short`». |
| **`'static`** | Самое долгое время жизни: вся работа программы. Оно у строковых литералов. |
| **подтип** (subtype) | Тип, который можно использовать там, где ожидается другой тип. Более долгое время жизни — подтип более короткого. |
| **вариантность** (variance) | Правило, которое говорит, как отношение подтипов передаётся через тип: ковариантность (то же направление), инвариантность (никак), контравариантность (обратное направление). |
| **API** (Application Programming Interface, программный интерфейс приложения) | Функции и типы, которые ваш код предлагает другому коду. |
| **HRTB** (Higher-Ranked Trait Bound, ограничение трейта высшего ранга) | Ограничение с `for<'a>`, которое должно выполняться для каждого времени жизни `'a`, а не для одного выбранного. |
| **поток с областью видимости** (scoped thread) | Поток, запущенный через `thread::scope`. Он гарантированно завершается до конца области, поэтому может заимствовать локальные данные. |
| **самоссылающаяся структура** (self-referential struct) | Структура, у которой поле указывает внутрь другого поля той же структуры. Безопасный Rust этого выразить не может. |

## 1. Быстрое повторение

Ссылка указывает на данные, которыми владеет кто-то другой. **Время жизни** — это участок кода, где ссылка действительна. Компилятор сравнивает время жизни каждой ссылки со временем жизни данных и отвергает программу, если ссылка может пережить свои данные:

```rust
fn main() {
    let reference;
    {
        let value = 5;
        reference = &value;
    }
    println!("{reference}");
}
```

```text
error[E0597]: `value` does not live long enough
 --> src/main.rs:5:21
  |
4 |         let value = 5;
  |             ----- binding `value` declared here
5 |         reference = &value;
  |                     ^^^^^^ borrowed value does not live long enough
6 |     }
  |     - `value` dropped here while still borrowed
7 |     println!("{reference}");
  |                --------- borrow later used here

For more information about this error, try `rustc --explain E0597`.
error: could not compile `demo` (bin "demo") due to 1 previous error
```

**Что происходит:**

1. `value` живёт только во внутреннем блоке. `reference` объявлена снаружи и хранит `&value`.
2. На закрывающей `}` блока `value` уничтожается, а `reference` после этого ещё используется.
3. Компилятор отмечает точное место, где данные уничтожаются (`dropped here while still borrowed`), и место, где ссылка используется позже (`borrow later used here`). Эти две отметки — ключ к чтению почти любой ошибки времён жизни.

Аннотации времён жизни никогда не меняют, как долго живут данные. Они только **описывают** компилятору связи, чтобы он мог их проверить.

## 2. Правила пропуска времён жизни — точно

Времена жизни вы пишете редко, потому что компилятор применяет к сигнатурам функций три правила. Если после них все времена жизни результата известны, писать ничего не нужно:

1. **Каждый параметр-ссылка получает собственное время жизни.** `fn f(a: &str, b: &str)` превращается в `fn f<'1, '2>(a: &'1 str, b: &'2 str)`.
2. **Если входное время жизни ровно одно, оно отдаётся всем выходным.** `fn f(a: &str) -> &str` превращается в `fn f<'1>(a: &'1 str) -> &'1 str`.
3. **Если один из параметров — `&self` или `&mut self`, его время жизни отдаётся всем выходным.**

```rust
// Правило 1: каждый параметр-ссылка получает своё время жизни.
// Правило 2: если входное время жизни ровно одно, результат получает его.
fn first_word(text: &str) -> &str {
    text.split_whitespace().next().unwrap_or("")
}

// Та же функция с явно записанными временами жизни:
#[allow(clippy::needless_lifetimes)] // Clippy убрал бы их; мы хотим их видеть
fn first_word_explicit<'a>(text: &'a str) -> &'a str {
    text.split_whitespace().next().unwrap_or("")
}

struct Library {
    name: String,
}

impl Library {
    // Правило 3: при `&self` результат получает время жизни `self`.
    fn label(&self, prefix: &str) -> &str {
        if prefix.is_empty() {
            &self.name
        } else {
            &self.name[1..]
        }
    }
}

fn main() {
    println!("{}", first_word("hello brave world"));
    println!("{}", first_word_explicit("good morning"));

    let library = Library {
        name: String::from("Central"),
    };
    println!("{} {}", library.label(""), library.label("x"));
}
```

```text
hello
good
Central entral
```

**Что происходит:**

1. У `first_word` одна входная ссылка, поэтому применяется правило 2. Она эквивалентна `first_word_explicit`.
2. У `label(&self, prefix: &str) -> &str` два входных времени жизни, но правило 3 говорит, что результат получает время жизни `self`. Поэтому результат заимствован у `Library`, а не у `prefix`.
3. Два вызова печатают `Central` (без префикса) и `entral` (срез без первого символа).

Если ни одно правило не может решить, время жизни нужно написать самому:

```text
error[E0106]: missing lifetime specifier
 --> src/main.rs:1:33
  |
1 | fn longest(a: &str, b: &str) -> &str {
  |               ----     ----     ^ expected named lifetime parameter
  |
  = help: this function's return type contains a borrowed value, but the signature does not say whether it is borrowed from `a` or `b`
help: consider introducing a named lifetime parameter
  |
1 | fn longest<'a>(a: &'a str, b: &'a str) -> &'a str {
  |           ++++     ++          ++          ++
```

Два входа, нет `self`: правила 2 и 3 не применяются, и компилятор не может угадать, идёт результат из `a` или из `b`.

## 3. Несколько времён жизни в одной сигнатуре

Простое исправление ошибки выше — использовать одно и то же время жизни `'a` для всего. Это безопасно, но делает сигнатуру строже, чем нужно. Посмотрите на функцию, которая возвращает только первый аргумент:

```rust
// Результат связан только с `x`. `y` может быть уничтожен раньше.
fn first_of<'a>(x: &'a str, _y: &str) -> &'a str {
    x
}

fn main() {
    let long_lived = String::from("I live long");
    let result;
    {
        let short_lived = String::from("short");
        result = first_of(&long_lived, &short_lived);
    } // `short_lived` уничтожается здесь
    println!("{result}");
}
```

```text
I live long
```

**Что происходит:**

1. У `x` именованное время жизни `'a`, и у результата тоже `'a`: «результат заимствован у `x`».
2. `_y: &str` — вход без имени. По правилу 1 у него своё, отдельное время жизни. Результат с ним никак не связан.
3. В `main` значение `short_lived` уничтожается до `println!`, но `result` зависит только от `long_lived`, поэтому программа принимается.

Теперь поставьте `'a` на оба параметра:

```rust
fn first_of<'a>(x: &'a str, _y: &'a str) -> &'a str {
    x
}

fn main() {
    let long_lived = String::from("I live long");
    let result;
    {
        let short_lived = String::from("short");
        result = first_of(&long_lived, &short_lived);
    }
    println!("{result}");
}
```

```text
error[E0597]: `short_lived` does not live long enough
  --> src/main.rs:10:40
   |
 9 |         let short_lived = String::from("short");
   |             ----------- binding `short_lived` declared here
10 |         result = first_of(&long_lived, &short_lived);
   |                                        ^^^^^^^^^^^^ borrowed value does not live long enough
11 |     }
   |     - `short_lived` dropped here while still borrowed
12 |     println!("{result}");
   |                ------ borrow later used here

For more information about this error, try `rustc --explain E0597`.
error: could not compile `demo` (bin "demo") due to 1 previous error
```

Когда у обоих параметров одно `'a`, компилятор должен предполагать, что результат может быть заимствован у любого из них. Результат живёт столько же, сколько **более короткое** из двух, поэтому `short_lived` должен жить до последнего использования `result`. Этого нет, и компилятор останавливается.

> 📏 **Соглашение:** давайте имя времени жизни только тем ссылкам, которые действительно связаны. Не ставьте `'a` везде, «чтобы скомпилировалось». Лишняя связь делает ваш API неудобнее. Линт Clippy `needless_lifetimes` помогает находить лишние имена.

## 4. Времена жизни в структурах и блоках `impl`

Структуре, которая хранит ссылку, нужен параметр времени жизни. Структура не может жить дольше данных, на которые указывает. В блоке `impl` параметр объявляется один раз (`impl<'a> Excerpt<'a>`) и используется в методах:

```rust
// 'long: 'short читается «'long переживает 'short».
fn pick<'short, 'long: 'short>(a: &'short str, b: &'long str, take_b: bool) -> &'short str {
    if take_b { b } else { a }
}

// Структура со ссылкой: структура не может пережить данные.
struct Excerpt<'a> {
    text: &'a str,
}

impl<'a> Excerpt<'a> {
    // Возвращает ТО ЖЕ время жизни, что у данных, а не время жизни `&self`.
    fn text(&self) -> &'a str {
        self.text
    }
}

fn main() {
    let a = String::from("short-lived");
    let result;
    {
        let b = String::from("long-lived");
        result = pick(&a, &b, true).len();
    }
    println!("{result}");

    let book = String::from("Call me Ishmael. Some years ago...");
    let kept: &str;
    {
        let excerpt = Excerpt { text: &book[..15] };
        kept = excerpt.text();
    } // Excerpt уничтожен, но `kept` по-прежнему указывает внутрь `book`
    println!("{kept}");
}
```

```text
10
Call me Ishmael
```

**Что происходит:**

1. `pick<'short, 'long: 'short>` объявляет **ограничение «переживает»** (outlives): `'long` длится не меньше, чем `'short`. Это позволяет вернуть `b` (`&'long str`) там, где обещан `&'short str`. Запись `'long: 'short` читается «long переживает short».
2. В `main` функция `pick` вызывается с `&a` и `&b`; длина результата (10) берётся внутри внутреннего блока, поэтому всё действительно.
3. `Excerpt<'a>` хранит `&'a str`. Метод `text(&self) -> &'a str` возвращает данные со временем жизни **`'a`** исходного текста. Сравните с правилом 3: обычная `fn text(&self) -> &str` вернула бы ссылку, связанную с `&self`.
4. Эта разница важна: `kept` остаётся пригодной после конца внутреннего блока, когда `excerpt` уже нет. Если бы `text` возвращала `&str`, связанный с `&self`, то `kept` не могла бы пережить `excerpt`, и программа не скомпилировалась бы.

> 💡 **Совет:** если структура хранит ссылку `&'a T`, методы, возвращающие части этих данных, должны возвращать `&'a ...`, а не ссылку, связанную с `&self`. Тогда вызывающий сможет сохранить результат после того, как структуры не станет.

Та же идея в настоящем примере: итератор, который отдаёт куски заимствованного текста.

```rust
// Итератор, который отдаёт куски строки, которую он ЗАИМСТВУЕТ.
struct Words<'a> {
    rest: &'a str,
}

impl<'a> Iterator for Words<'a> {
    type Item = &'a str; // связан с исходным текстом, а не с &mut self

    fn next(&mut self) -> Option<&'a str> {
        let trimmed = self.rest.trim_start();
        if trimmed.is_empty() {
            return None;
        }
        let end = trimmed.find(' ').unwrap_or(trimmed.len());
        let (word, rest) = trimmed.split_at(end);
        self.rest = rest;
        Some(word)
    }
}

fn main() {
    let text = String::from("the quick  brown fox");
    let words: Vec<&str> = Words { rest: &text }.collect();
    println!("{words:?}");
    // Все слова можно хранить одновременно: каждое указывает внутрь `text`.
    println!("{} words, first = {}", words.len(), words[0]);
}
```

```text
["the", "quick", "brown", "fox"]
4 words, first = the
```

**Что происходит:**

1. `Words<'a>` хранит ещё не прочитанный текст: `rest: &'a str`.
2. Трейт `Iterator` говорит `fn next(&mut self) -> Option<Self::Item>`. `Item` здесь — `&'a str`: время жизни **текста**, а не заимствования `&mut self`. Поэтому вызывающий может собрать все слова в `Vec` и держать их одновременно; каждое слово указывает внутрь `text`.
3. Стандартный `Iterator` не может возвращать ссылки внутрь самого итератора (тогда каждый элемент был бы связан с заимствованием `&mut self`, и одновременно жил бы только один элемент). Для таких «одалживающих итераторов» (lending iterators) нужен другой дизайн.

## 5. `'static` и ограничение `T: 'static`

`'static` — самое долгое время жизни: вся работа программы. Оно у строковых литералов, потому что они хранятся внутри файла программы.

Ограничение `T: 'static` означает немного другое: «**`T` не содержит ссылок, которые могут истечь**». Оно *не* означает «`T` живёт вечно». Владеющая `String` удовлетворяет `T: 'static`: в ней нет заимствованных данных, и она может жить столько, сколько захочет её владелец.

```rust
// `T: 'static` означает: в T нет заимствованных данных, которые могут истечь.
fn needs_static<T: 'static>(value: T) -> T {
    value
}

fn main() {
    let owned = String::from("I own my data"); // внутри нет ссылок: 'static
    let literal: &'static str = "text baked into the program";
    let number = 42;

    println!("{}", needs_static(owned)); // String ПЕРЕМЕЩАЕТСЯ внутрь; это не ссылка
    println!("{}", needs_static(literal));
    println!("{}", needs_static(number));
}
```

```text
I own my data
text baked into the program
42
```

Ссылка на локальную переменную ему не удовлетворяет:

```rust
fn needs_static<T: 'static>(value: T) -> T {
    value
}

fn main() {
    let owned = String::from("local");
    let borrowed: &str = &owned; // ссылка на локальную переменную
    println!("{}", needs_static(borrowed));
}
```

```text
error[E0597]: `owned` does not live long enough
 --> src/main.rs:7:26
  |
6 |     let owned = String::from("local");
  |         ----- binding `owned` declared here
7 |     let borrowed: &str = &owned; // a reference to a local variable
  |                          ^^^^^^ borrowed value does not live long enough
8 |     println!("{}", needs_static(borrowed));
  |                    ---------------------- argument requires that `owned` is borrowed for `'static`
9 | }
  | - `owned` dropped here while still borrowed
  |
note: requirement that the value outlives `'static` introduced here
 --> src/main.rs:1:20
  |
1 | fn needs_static<T: 'static>(value: T) -> T {
  |                    ^^^^^^^
```

> ⚠️ **Частая ошибка:** когда компилятор предлагает `'static`, не добавляйте его лишь бы заглушить ошибку. Обычно данные должны быть владеющими (`String` вместо `&str`) или ссылке нужно настоящее время жизни.

Самое важное место, где вы встретите `'static`, — это `thread::spawn`. Новый поток может работать дольше, чем функция, которая его запустила, поэтому замыкание не должно заимствовать локальные данные:

```rust
use std::thread;

fn main() {
    let names = vec![String::from("Ana"), String::from("Bo")];
    let handle = thread::spawn(|| {
        println!("{}", names.len());
    });
    handle.join().unwrap();
}
```

```text
error[E0373]: closure may outlive the current function, but it borrows `names`, which is owned by the current function
 --> src/main.rs:5:32
  |
5 |     let handle = thread::spawn(|| {
  |                                ^^ may outlive borrowed value `names`
6 |         println!("{}", names.len());
  |                        ----- `names` is borrowed here
  |
note: function requires argument type to outlive `'static`
 --> src/main.rs:5:18
  |
5 |       let handle = thread::spawn(|| {
  |  __________________^
6 | |         println!("{}", names.len());
7 | |     });
  | |______^
help: to force the closure to take ownership of `names` (and any other referenced variables), use the `move` keyword
  |
5 |     let handle = thread::spawn(move || {
  |                                ++++

For more information about this error, try `rustc --explain E0373`.
error: could not compile `demo` (bin "demo") due to 1 previous error
```

Есть два исправления. Можно переместить данные в поток (`move ||`, тогда поток ими владеет). Или, если поток должен только заимствовать, использовать **поток с областью видимости** (scoped thread). `thread::scope` гарантирует, что каждый поток внутри него завершится до конца области, поэтому заимствование безопасно:

```rust
use std::thread;

fn main() {
    let numbers = [1, 2, 3, 4, 5, 6, 7, 8];
    let (left, right) = numbers.split_at(4);

    // Потоки с областью видимости могут ЗАИМСТВОВАТЬ локальные данные: область заканчивается раньше, чем `numbers`.
    let (sum_left, sum_right) = thread::scope(|s| {
        let a = s.spawn(|| left.iter().sum::<i32>());
        let b = s.spawn(|| right.iter().sum::<i32>());
        (a.join().unwrap(), b.join().unwrap())
    });

    println!(
        "left = {sum_left}, right = {sum_right}, total = {}",
        sum_left + sum_right
    );
    println!("numbers is still ours: {}", numbers.len());
}
```

```text
left = 10, right = 26, total = 36
numbers is still ours: 8
```

**Что происходит:**

1. `split_at(4)` делит вектор на две заимствованные половины.
2. `thread::scope(|s| { ... })` открывает область. `s.spawn(...)` запускает поток, которому разрешено заимствовать `left` и `right`.
3. `a.join().unwrap()` ждёт поток и возвращает сумму. Область также дождётся любого потока, который вы не присоединили.
4. После области `numbers` снова полностью доступен: все заимствования закончились.

## 6. Подтипизация и вариантность

Если `'long` переживает `'short`, то `&'long str` можно использовать там, где ожидается `&'short str`. Мы говорим, что `&'long T` — **подтип** `&'short T`. Времена жизни делают эту подтипизацию автоматической.

```rust
// Общая ссылка КОВАРИАНТНА: более долгое время жизни может заменить более короткое.
#[allow(clippy::needless_lifetimes)] // имя 'short здесь только для обучения
fn print_it<'short>(text: &'short str) {
    println!("{text}");
}

fn main() {
    let forever: &'static str = "lives for the whole program";
    print_it(forever); // &'static str принимается там, где ожидается &'short str

    let owned = String::from("a local string");
    let shorter: &str = &owned;
    print_it(shorter);
}
```

```text
lives for the whole program
a local string
```

**Что происходит:** `print_it` принимает ссылку с *каким-то* коротким временем жизни. `&'static str` автоматически сокращается до этого времени жизни, поэтому вызов допустим. Во время работы ничего не копируется и не преобразуется; это целиком идея этапа компиляции.

Разрешено ли такое сокращение внутри более крупного типа, определяет **вариантность**. Бывает три вида:

| Вариантность | Смысл | Пример |
|---|---|---|
| **Ковариантность** | Более долгое время жизни может заменить более короткое внутри типа. | `&'a T`, `Box<T>`, `Vec<T>`, `Option<T>` |
| **Инвариантность** | Замена не разрешена ни в одну сторону; время жизни должно совпадать точно. | `&mut T` (по `T`), `Cell<T>`, `RefCell<T>` |
| **Контравариантность** | Обратное направление: более короткое время жизни может заменить более долгое. | Аргументы функций: `fn(&'a str)` контравариантна по `'a` |

Почему `&mut T` инвариантна по `T`? Потому что через неё можно писать. Допустим, Rust позволил бы рассматривать `&mut &'static str` как `&mut &'short str`. Тогда через неё можно было бы сохранить недолговечную ссылку, и исходная переменная (которая обещает `'static`) осталась бы с висячей ссылкой. Компилятор этого не допускает:

```rust
fn assign<'a>(target: &mut &'a str, value: &'a str) {
    *target = value;
}

fn main() {
    let mut slot: &'static str = "static text";
    let local = String::from("local text");
    assign(&mut slot, &local);
    println!("{slot}");
}
```

```text
error[E0597]: `local` does not live long enough
  --> src/main.rs:8:23
   |
 6 |     let mut slot: &'static str = "static text";
   |                   ------------ type annotation requires that `local` is borrowed for `'static`
 7 |     let local = String::from("local text");
   |         ----- binding `local` declared here
 8 |     assign(&mut slot, &local);
   |                       ^^^^^^ borrowed value does not live long enough
 9 |     println!("{slot}");
10 | }
   | - `local` dropped here while still borrowed

For more information about this error, try `rustc --explain E0597`.
error: could not compile `demo` (bin "demo") due to 1 previous error
```

**Что происходит:**

1. У `slot` тип `&'static str`. `&mut slot` — это `&mut &'static str`.
2. `assign<'a>` требует, чтобы и цель, и значение использовали *одно и то же* `'a`. Поскольку `&mut` инвариантна, для цели `'a` должно быть точно `'static`.
3. Тогда значение `&local` тоже должно быть `'static`, что невозможно, поэтому компилятор сообщает `local does not live long enough`.

Вы нечасто будете писать вариантность, но теперь сообщения вроде «type annotation requires that ... is borrowed for `'static`» понятны.

## 7. Ограничения трейтов высшего ранга: `for<'a>`

Посмотрите на ограничение `F: Fn(&str) -> &str`. Такое замыкание должно работать для **каждого** времени жизни аргумента: сегодня его вызовут со строкой, живущей секунду, а завтра — со строкой, живущей час. Длинная запись ограничения:

```rust
F: for<'a> Fn(&'a str) -> &'a str
```

`for<'a>` читается «для всех времён жизни `'a`». Это **ограничение трейта высшего ранга** (HRTB). Чаще всего компилятор пишет его за вас, когда вы используете короткую запись.

```rust
// `for<'a>` читается «для каждого времени жизни 'a».
fn apply<F>(f: F) -> usize
where
    F: for<'a> Fn(&'a str) -> &'a str,
{
    let first = String::from("  padded  ");
    let second = String::from("  other");
    // f вызывается с двумя разными временами жизни, по одному на каждую локальную String
    f(&first).len() + f(&second).len()
}

// Короткая запись `Fn(&str) -> &str` означает то же самое.
fn apply_short<F: Fn(&str) -> &str>(f: F) -> usize {
    f("  x  ").len()
}

// Частое настоящее применение: «по ссылке на C можно итерироваться при любом её времени жизни».
fn total<C>(collection: &C) -> i32
where
    for<'a> &'a C: IntoIterator<Item = &'a i32>,
{
    collection.into_iter().sum()
}

fn main() {
    println!("{}", apply(|s| s.trim()));
    println!("{}", apply_short(|s| s.trim()));
    println!("{}", total(&vec![1, 2, 3]));
    println!("{}", total(&[10, 20]));
}
```

```text
11
1
6
30
```

**Что происходит:**

1. `apply` вызывает `f` дважды с двумя разными локальными строками. У каждого вызова своё время жизни, поэтому `f` должна работать для всех. Вот почему нужен `for<'a>`.
2. `apply_short` использует короткую запись `Fn(&str) -> &str`. Она означает ровно то же.
3. `total` использует HRTB, который нужно писать вручную: «для каждого времени жизни `'a` из `&'a C` можно получить итератор по `&'a i32`». Работает для `Vec<i32>` и для массивов.
4. Числа: `"  padded  "` после обрезки — 6 символов, а `"  other"` — 5, итого 11; `"x"` даёт 1; 1 + 2 + 3 = 6; 10 + 20 = 30.

Если забыть HRTB там, где он нужен, компилятор скажет:

```text
error[E0637]: `&` without an explicit lifetime name cannot be used here
 --> src/main.rs:3:5
  |
3 |     &C: IntoIterator<Item = &i32>,
  |     ^ explicit lifetime name needed here
  |
help: consider introducing a higher-ranked lifetime here
  |
3 |     for<'a> &'a C: IntoIterator<Item = &i32>,
  |     +++++++  ++
```

У замыканий есть особая ловушка. Если определить замыкание в отдельном `let` с аннотированным параметром, компилятор не знает, что результат связан с аргументом, и не может сделать замыкание высшего ранга:

```rust
fn apply<F>(f: F) -> usize
where
    F: for<'a> Fn(&'a str) -> &'a str,
{
    f("  padded  ").len()
}

fn main() {
    let first_word = |s: &str| s.split(' ').next().unwrap_or("");
    println!("{}", apply(first_word));
}
```

```text
error: lifetime may not live long enough
 --> src/main.rs:9:32
  |
9 |     let first_word = |s: &str| s.split(' ').next().unwrap_or("");
  |                          -   - ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ returning this value requires that `'1` must outlive `'2`
  |                          |   |
  |                          |   return type of closure is &'2 str
  |                          let's call the lifetime of this reference `'1`
```

Исправление — написать замыкание прямо там, где оно передаётся (`apply(|s| s.split(' ').next().unwrap_or(""))`), потому что тогда ожидаемая сигнатура известна, и компилятор выводит её правильно. Вы используете это в Упражнении 3.

## 8. `impl Trait` и захваченные времена жизни (редакция 2024)

Функция может возвращать `impl Trait`: скрытый тип, который вы не называете ([Урок 06.1](01-trait-objects-and-dynamic-dispatch.md)). Вопрос: какие времена жизни может содержать скрытый тип? В **редакции 2024** ответ прост: **все**, которые находятся в области видимости, в том числе из параметров. В редакции 2021 нужно было писать `+ '_` или ограничение `use<'_>`, иначе возникала ошибка `E0700`.

```rust
// Редакция 2024: `impl Trait` в возвращаемом типе захватывает ВСЕ времена жизни в области видимости.
fn letters(text: &str) -> impl Iterator<Item = char> {
    text.chars()
}

// `use<>` говорит: ничего не захватывать. Результат не зависит от `text`.
fn owned_letters(text: &str) -> impl Iterator<Item = char> + use<> {
    text.chars().collect::<Vec<char>>().into_iter()
}

fn main() {
    let upper: String = letters("abc").map(|c| c.to_ascii_uppercase()).collect();
    println!("{upper}");

    let iter = {
        let temp = String::from("temp");
        owned_letters(&temp) // `temp` уничтожается после этого блока
    };
    println!("{}", iter.count());
}
```

```text
ABC
4
```

**Что происходит:**

1. `letters` возвращает `text.chars()`, который заимствует `text`. В возвращаемом типе время жизни не упомянуто, но в редакции 2024 скрытому типу всё равно разрешено захватить время жизни `text`.
2. `owned_letters` возвращает итератор по новому `Vec<char>`. После возврата `text` ему не нужен. Ограничение `+ use<>` говорит «не захватывать никаких времён жизни». Это позволяет вызывающему уничтожить `temp`, продолжая держать итератор, как делает `main`.
3. В редакции 2021 первая функция не компилируется (`E0700: hidden type for impl Iterator captures lifetime that does not appear in bounds`). Это старое правило, которое заменила редакция 2024.

## 9. Чего времена жизни не умеют

- **Самоссылающиеся структуры.** Структуру, которая владеет `String` и одновременно хранит `&str` внутрь этой `String`, нельзя записать простыми временами жизни: ссылке понадобилось бы время жизни «столько же, сколько сама структура», а перемещение структуры сделало бы указатель недействительным. Решения: хранить индексы (диапазоны) вместо ссылок, хранить данные и представление отдельно или, в сложных случаях, использовать `Pin` (Урок 06.9) и `unsafe`. Такие приёмы упаковывают крейты вроде `ouroboros`.
- **Чтение из будущего.** Времена жизни говорят «действительно до», но никогда «действительно с». Они не могут описать, когда будет создано значение.

## 10. Расшифровщик ошибок времён жизни

| Что вы видите | Что это значит | Что попробовать сначала |
|---|---|---|
| `E0106 missing lifetime specifier` | Правила пропуска не могут выбрать выходное время жизни. | Назовите время жизни и свяжите результат с нужным параметром либо верните владеющее значение (`String`). |
| `E0597 ... does not live long enough` | Ссылка используется после уничтожения её данных. | Посмотрите на две отметки: где уничтожается и где используется. Заставьте данные жить дольше или скопируйте их. |
| `E0515 cannot return reference to local variable` | Вы возвращаете ссылку на то, что создано внутри функции. | Верните владеющее значение (`String`, `Vec`). |
| `E0373 closure may outlive the current function` | Замыкание, которое может выполниться позже, заимствует локальную переменную. | Используйте `move` или поток с областью видимости. |
| `lifetime may not live long enough` | Сигнатура обещает более долгое время жизни, чем может дать код. | Добавьте или исправьте связь времён жизни (`'a: 'b`) либо измените возвращаемый тип. |
| `E0521 borrowed data escapes outside of function` | Ссылка пытается попасть туда, где требуется `'static`. | Владейте данными (клонируйте) или передайте владеющее значение. |

Самая частая в повседневном коде:

```text
error[E0515]: cannot return reference to local variable `text`
 --> src/main.rs:3:5
  |
3 |     &text
  |     ^^^^^ returns a reference to data owned by the current function

For more information about this error, try `rustc --explain E0515`.
error: could not compile `demo` (bin "demo") due to 1 previous error
```

## Частые ошибки

### 1. Одно `'a` на всё

Делает сигнатуры слишком строгими. Используйте раздельные времена жизни для несвязанных ссылок или доверьте это правилам пропуска.

### 2. Возврат ссылки, связанной с `&self`, когда у данных есть собственное время жизни

`fn text(&self) -> &str` связывает результат с заимствованием `&self`. Возвращайте `&'a str`, если структура хранит `&'a str`.

### 3. Добавление `'static`, чтобы ошибка исчезла

`T: 'static` означает «нет заимствованных данных». Лучше владейте данными или используйте поток с областью видимости, когда нужно просто поделиться заимствованием.

### 4. Путаница `T: 'static` с «живёт вечно»

Владеющая `String` — это `'static`, и она освобождается, когда владелец её уничтожает.

### 5. Замыкания, определённые в отдельном `let` с аргументами `&`

Сигнатура не выводится как сигнатура высшего ранга. Пишите замыкание в том месте, где оно передаётся.

## Попробуйте сами

1. Откройте `~/projects/rust/code/p06-lifetimes`.
2. Скопируйте `words.rs` (раздел 4) в `src/main.rs` и запустите.
3. Добавьте в `Words<'a>` метод `fn remaining(&self) -> &'a str`, который возвращает непрочитанный остаток текста без ведущих пробелов (`self.rest.trim_start()`). В `main` дважды вызовите `next()` у `let mut words = Words { rest: &text };` и напечатайте `words.remaining()`. Ожидается `brown fox`.
4. Измените возвращаемый тип `remaining` на `&str` (без `'a`) и попробуйте сохранить результат после того, как `words` вышла из области видимости (создайте `words` во внутреннем блоке, а результат используйте снаружи). Прочитайте ошибку: это работает время жизни `&self`. Верните `&'a str`.
5. Выполните `cargo fmt` и `cargo clippy` и сделайте коммит: `git add code/p06-lifetimes` и `git commit -m "feat(p06): borrowed word iterator"`.

## Упражнения

### Упражнение 1 — Два времени жизни (легко)

Напишите `fn pick_primary<'a>(primary: &'a str, _fallback: &str) -> &'a str`, которая возвращает первый аргумент. Напишите тест, в котором `fallback` создаётся во внутреннем блоке и уничтожается до использования результата.

<details>
<summary>Подсказка</summary>

У второго параметра своё анонимное время жизни (правило пропуска 1), поэтому результат с ним не связан. В тесте объявите `let result;` до блока и присвойте внутри.

</details>

<details>
<summary>Ответ</summary>

```rust
// Результат заимствован только у `primary`.
fn pick_primary<'a>(primary: &'a str, _fallback: &str) -> &'a str {
    primary
}

fn main() {
    let name = String::from("Ana");
    let answer;
    {
        let temporary = String::from("temporary");
        answer = pick_primary(&name, &temporary);
    }
    println!("{answer}");
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn result_can_outlive_the_second_argument() {
        let primary = String::from("main");
        let result;
        {
            let fallback = String::from("fallback");
            result = pick_primary(&primary, &fallback);
        }
        assert_eq!(result, "main");
    }
}
```

```text
Ana
```

```text

running 1 test
.
test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

Почему это работает: с результатом связан только `primary`. Если написать `'a` на обоих параметрах, тест не скомпилируется (раздел 3).

</details>

### Упражнение 2 — Читатель конфигурации, возвращающий долгоживущие значения (средне)

Напишите `struct Config<'a>`, которая хранит `Vec<&'a str>` (строки текста), с функциями `parse(text: &'a str) -> Config<'a>` и `get(&self, key: &str) -> Option<&'a str>`. Строка выглядит как `key = value`. Проверьте тестом, что значение, возвращённое `get`, можно использовать после уничтожения `Config`.

<details>
<summary>Подсказка</summary>

`text.lines().collect()` строит вектор. В `get` используйте `line.split_once('=')`. Тип результата — `Option<&'a str>`, а не `Option<&str>`: значение берётся из текста, а не из `self`.

</details>

<details>
<summary>Ответ</summary>

```rust
struct Config<'a> {
    lines: Vec<&'a str>,
}

impl<'a> Config<'a> {
    fn parse(text: &'a str) -> Config<'a> {
        Config {
            lines: text.lines().collect(),
        }
    }

    // Возвращаемый &str заимствован из ИСХОДНОГО текста ('a), а не из `self`.
    fn get(&self, key: &str) -> Option<&'a str> {
        self.lines.iter().find_map(|line| {
            let (k, v) = line.split_once('=')?;
            if k.trim() == key {
                Some(v.trim())
            } else {
                None
            }
        })
    }
}

fn main() {
    let text = String::from("host = example.org\nport = 8080");
    let port = {
        let config = Config::parse(&text);
        config.get("port")
    }; // `config` уничтожается здесь, но `port` по-прежнему указывает внутрь `text`
    println!("{port:?}");
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn value_outlives_the_config() {
        let text = String::from("host = example.org\nport = 8080");
        let port = {
            let config = Config::parse(&text);
            config.get("port")
        };
        assert_eq!(port, Some("8080"));
    }

    #[test]
    fn missing_key_gives_none() {
        let text = String::from("a = 1");
        assert_eq!(Config::parse(&text).get("b"), None);
    }
}
```

```text
Some("8080")
```

```text

running 2 tests
..
test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

Почему это работает: `'a` в возвращаемом типе говорит «эта ссылка указывает внутрь исходного текста». `Config` можно уничтожить, но текст жив, поэтому ссылка остаётся действительной.

</details>

### Упражнение 3 — Замыкание высшего ранга (средне)

Напишите `fn apply_to_all<'s, F>(items: &'s [String], f: F) -> Vec<&'s str> where F: for<'a> Fn(&'a str) -> &'a str`. Она применяет `f` к каждому элементу. Проверьте тестом с `|s| s.trim()` и с `|s| s.split(' ').next().unwrap_or("")`.

<details>
<summary>Подсказка</summary>

`items.iter().map(|item| f(item)).collect()`. Пишите замыкания прямо в вызове, а не в отдельном `let`.

</details>

<details>
<summary>Ответ</summary>

```rust
fn apply_to_all<'s, F>(items: &'s [String], f: F) -> Vec<&'s str>
where
    F: for<'a> Fn(&'a str) -> &'a str,
{
    items.iter().map(|item| f(item)).collect()
}

fn main() {
    let items = vec![String::from("  a "), String::from("b  "), String::from("c")];
    let trimmed = apply_to_all(&items, |s| s.trim());
    println!("{trimmed:?}");
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn trims_every_item() {
        let items = vec![String::from("  a "), String::from("b  ")];
        assert_eq!(apply_to_all(&items, |s| s.trim()), vec!["a", "b"]);
    }

    #[test]
    fn works_with_another_closure() {
        let items = vec![String::from("hello world")];
        let result = apply_to_all(&items, |s| s.split(' ').next().unwrap_or(""));
        assert_eq!(result, vec!["hello"]);
    }
}
```

```text
["a", "b", "c"]
```

```text

running 2 tests
..
test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

Почему это работает: `f` вызывается со временем жизни каждого элемента, поэтому она должна быть корректна для любого времени жизни. Возвращаемые строки заимствованы из `items` (`'s`).

</details>

### Упражнение 4 — Параллельная сумма без клонирования (сложно)

Напишите `fn parallel_sum(numbers: &[i64]) -> i64`, которая делит срез на две половины и суммирует их в двух потоках с областью видимости. Сравните в тестах с последовательной суммой для 101 числа, для пустого среза и для среза из одного элемента.

<details>
<summary>Подсказка</summary>

`numbers.split_at(numbers.len() / 2)`, затем `thread::scope(|scope| { let a = scope.spawn(|| left.iter().sum::<i64>()); ... })`. Присоедините оба дескриптора и сложите результаты.

</details>

<details>
<summary>Ответ</summary>

```rust
use std::thread;

fn parallel_sum(numbers: &[i64]) -> i64 {
    let middle = numbers.len() / 2;
    let (left, right) = numbers.split_at(middle);
    thread::scope(|scope| {
        let a = scope.spawn(|| left.iter().sum::<i64>());
        let b = scope.spawn(|| right.iter().sum::<i64>());
        a.join().unwrap() + b.join().unwrap()
    })
}

fn main() {
    let data: Vec<i64> = (1..=1000).collect();
    println!("sum = {}", parallel_sum(&data));
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn sums_like_the_sequential_version() {
        let data: Vec<i64> = (1..=101).collect();
        assert_eq!(parallel_sum(&data), data.iter().sum::<i64>());
    }

    #[test]
    fn empty_and_single() {
        assert_eq!(parallel_sum(&[]), 0);
        assert_eq!(parallel_sum(&[5]), 5);
    }
}
```

```text
sum = 500500
```

```text

running 2 tests
..
test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

Почему это работает: потоки с областью видимости могут заимствовать `left` и `right`, потому что область их дожидается. Никакого `Arc`, никакого `clone`, никакого `'static`. Для крошечных входов потоки стоят дороже, чем экономят; измеряйте, прежде чем распараллеливать ([Урок 05.12](../05-rust-intermediate/12-benchmarking-basics.md)).

</details>

## Вопросы для самопроверки

1. Сформулируйте три правила пропуска времён жизни.

<details>
<summary>Ответ</summary>

(1) Каждый параметр-ссылка получает своё время жизни. (2) Если входное время жизни ровно одно, оно отдаётся всем выходным. (3) Если есть параметр `&self` или `&mut self`, все выходные времена жизни получают время жизни `self`.

</details>

2. Почему `fn longest(a: &str, b: &str) -> &str` — ошибка?

<details>
<summary>Ответ</summary>

Два входных времени жизни и нет `self`, поэтому ни одно правило пропуска не может выбрать выходное время жизни. Его нужно написать (например, `<'a>` на обоих параметрах и результате) или вернуть владеющую `String`.

</details>

3. Что означает `'long: 'short`?

<details>
<summary>Ответ</summary>

`'long` переживает `'short`: оно длится не меньше. Ссылку со временем жизни `'long` можно использовать там, где ожидается `'short`.

</details>

4. Означает ли `T: 'static`, что `T` живёт всю программу?

<details>
<summary>Ответ</summary>

Нет. Оно означает, что в `T` нет заимствованных данных, которые могли бы истечь. Владеющая `String` ему удовлетворяет и освобождается как обычно.

</details>

5. Почему `thread::spawn` требует замыкание `'static` и как `thread::scope` этого избегает?

<details>
<summary>Ответ</summary>

Запущенный поток может пережить функцию, которая его создала, поэтому он не должен заимствовать локальные данные. `thread::scope` гарантирует, что все потоки, запущенные в области, завершатся до её конца, поэтому они могут заимствовать локальные переменные.

</details>

6. Почему `&mut T` инвариантна по `T`?

<details>
<summary>Ответ</summary>

Потому что через неё можно писать. Если бы `&mut &'static str` можно было сократить до `&mut &'short str`, в место, обещающее `'static`, можно было бы сохранить недолговечную ссылку, получив висячую ссылку.

</details>

7. Что означает `for<'a> Fn(&'a str) -> &'a str`?

<details>
<summary>Ответ</summary>

Функция или замыкание должны работать для любого возможного времени жизни `'a` аргумента, а у результата то же время жизни, что у аргумента. Короткая запись `Fn(&str) -> &str` означает то же.

</details>

8. Какие времена жизни может захватить возвращаемый `impl Trait` в редакции 2024 и как отказаться от захвата?

<details>
<summary>Ответ</summary>

Все времена жизни в области видимости, включая времена жизни параметров. Чтобы отказаться, добавьте `+ use<>` (или перечислите нужные времена жизни, например `use<'a>`).

</details>

## Итоги

- Время жизни описывает, как долго действительна ссылка. Аннотации никогда не продлевают жизнь; они только описывают связи, чтобы компилятор мог их проверить.
- Правила пропуска: одно время жизни на каждый параметр-ссылку; единственное входное время жизни переходит в результат; `&self` отдаёт своё время жизни результату.
- Используйте раздельные времена жизни для несвязанных ссылок. Возвращайте `&'a T` (а не заимствование `&self`), когда данные берутся из ссылки, которую хранит структура.
- `'long: 'short` — ограничение «переживает». `T: 'static` означает «нет заимствованных данных», а не «живёт вечно».
- `thread::spawn` требует `'static`; `thread::scope` позволяет потокам заимствовать локальные данные.
- `&'a T` ковариантна; `&mut T` и `Cell<T>` инвариантны по `T`; аргументы функций контравариантны.
- `for<'a>` означает «для всех времён жизни». Пишите замыкания прямо там, где они передаются, чтобы компилятор вывел сигнатуру высшего ранга.
- В редакции 2024 `impl Trait` захватывает все времена жизни в области видимости; `use<>` отменяет это.

## Источники и что почитать

- [The Rust Programming Language — 10.3 Validating References with Lifetimes](https://doc.rust-lang.org/book/ch10-03-lifetime-syntax.html) — основы, на которых строится этот урок, включая правила пропуска.
- [The Rust Reference — Lifetime elision](https://doc.rust-lang.org/reference/lifetime-elision.html) — точные правила, включая трейт-объекты и заголовки `impl`.
- [The Rustonomicon — Subtyping and Variance](https://doc.rust-lang.org/nomicon/subtyping.html) — полное объяснение вариантности с таблицей типов.
- [The Rustonomicon — Higher-Rank Trait Bounds](https://doc.rust-lang.org/nomicon/hrtb.html) — зачем нужен `for<'a>`.
- [The Rust Reference — Subtyping and variance](https://doc.rust-lang.org/reference/subtyping.html) — формальная версия.
- [std::thread::scope](https://doc.rust-lang.org/std/thread/fn.scope.html) — потоки с областью видимости в стандартной библиотеке.

---

[🏠 Начало](../START-HERE.md) · [← 06.4 Внутренняя изменяемость](04-interior-mutability.md) · [📋 Этап 06](README.md) · [06.6 Небезопасный Rust и Miri →](06-unsafe-rust-and-miri.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
