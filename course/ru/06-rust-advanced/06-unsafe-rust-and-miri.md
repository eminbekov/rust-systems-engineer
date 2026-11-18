<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 06.5 Времена жизни подробно](05-lifetimes-in-depth.md) · [📋 Этап 06](README.md) · [06.7 Декларативные макросы →](07-declarative-macros.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/06-rust-advanced/06-unsafe-rust-and-miri.md) <!-- nav -->

# Урок 06.6 — Небезопасный Rust и Miri

> **Этап 06 — Rust: продвинутый уровень** · Урок 6 из 12 · ⏱ около 4 часов (2 занятия)

## Что вы узнаете

- Что означает `unsafe`: это обещание, которое вы даёте компилятору, а не переключатель, отключающий проверки.
- Пять вещей, которые умеет только `unsafe`-код, и как пользоваться сырыми указателями.
- Как писать небезопасный код ответственно: комментарии `// SAFETY:`, раздел `# Safety` в документации и небольшие безопасные обёртки.
- Что такое неопределённое поведение (UB) и почему программа с UB может «работать» на вашем компьютере.
- Как запускать код под Miri — инструментом, который обнаруживает многие виды UB.
- Правила редакции 2024: `unsafe_op_in_unsafe_fn` и `static_mut_refs`.

## Перед началом

- Вы прошли [Урок 06.5 — Времена жизни подробно](05-lifetimes-in-depth.md).
- Вы знаете владение, заимствование и срезы ([Этап 03](../03-ownership-and-borrowing/01-memory-stack-and-heap.md)), `Box` ([Урок 05.9](../05-rust-intermediate/09-smart-pointers.md)) и тесты ([Урок 05.6](../05-rust-intermediate/06-testing-in-depth.md)).
- Создайте учебный проект: `cargo new code/p06-unsafe`.
- Чтобы пользоваться Miri, нужен nightly-тулчейн с компонентом Miri. Установите его один раз; ваш обычный (stable) тулчейн останется основным:

```sh
rustup toolchain install nightly --component miri
cargo +nightly miri --version
```

Первый запуск Miri печатает `Preparing a sysroot for Miri ... done`. Это нормально: собирается специальная копия стандартной библиотеки, на это уходит около минуты.

## Новые слова

| Слово | Значение |
|---|---|
| **`unsafe`** | Ключевое слово, отмечающее код, где **вы**, а не компилятор, гарантируете выполнение правил Rust. |
| **сырой указатель** (raw pointer) | Указатель без правил безопасности ссылок: `*const T` (чтение) и `*mut T` (чтение и запись). Может быть нулевым, висячим или невыровненным. |
| **разыменование** (dereference) | Переход по указателю, чтобы прочитать или записать значение, на которое он указывает (`*pointer`). |
| **UB** (Undefined Behaviour, неопределённое поведение) | Программа, нарушающая правила языка. Компилятор считает, что этого никогда не происходит, поэтому результат может быть любым: сбой, мусор или «всё работает». |
| **висячий указатель** (dangling pointer) | Указатель на память, которая освобождена или никогда не существовала. |
| **алиасинг** (aliasing) | Два указателя или ссылки на одну и ту же память. Rust запрещает существование `&mut` вместе с любой другой ссылкой на то же место. |
| **API** (Application Programming Interface, программный интерфейс приложения) | Функции и типы, которые ваш код предлагает другому коду. |
| **корректность API** (soundness) | Безопасный API **корректен** (sound), если никакой безопасный код не может с его помощью вызвать UB. API, который можно неправильно использовать из безопасного кода, **некорректен** (unsound). |
| **безопасная абстракция** (safe abstraction) | Функция с безопасной сигнатурой, тело которой внутри использует `unsafe` и проверяет всё необходимое для корректности. |
| **Miri** | Интерпретатор MIR (Mid-level Intermediate Representation, упрощённая форма вашей программы внутри компилятора). Он выполняет программу шаг за шагом и останавливается, когда видит UB. |
| **Stacked Borrows / Tree Borrows** | Два экспериментальных набора правил, по которым Miri решает, какие использования указателей допустимы (правила алиасинга). |
| **union** | Тип, все поля которого делят одну и ту же память. Чтение поля — `unsafe`. |
| **FFI** (Foreign Function Interface, интерфейс внешних функций) | Вызов кода, написанного на другом языке. Вы изучите его в [Уроке 06.10](10-ffi-calling-c.md). |

## 1. Что на самом деле означает `unsafe`

Безопасный Rust даёт сильные гарантии: нет использования после освобождения, нет гонок данных, нет чтения за границами. Их проверяет компилятор. Но некоторые корректные программы компилятор доказать не может: работа с оборудованием, вызов кода на C, построение структур вроде `Vec` или `Rc`.

Для таких случаев в Rust есть ключевое слово `unsafe`. Оно **не** отключает проверку заимствований или проверку типов. Оно лишь открывает пять дополнительных возможностей (раздел 2). Взамен вы даёте **обещание**: «Я проверил, что здесь правила выполняются, хотя компилятор этого не может».

Представьте банковское хранилище с надёжной дверью. Безопасный Rust — это дверь с замком, который проверяет банк. `unsafe` — это чёрный ход, который вы открываете сами: банк доверяет, что вы правильно его закроете. Если нет, ущерб может быть где угодно.

Ключевое слово используется в двух направлениях:

- `unsafe { ... }` (блок) означает: «**я** обещаю, что этот код корректен». Он нужен там, где выполняются небезопасные операции.
- `unsafe fn` и `unsafe trait` означают: «**вы**, вызывающий или реализующий, должны соблюдать дополнительные правила, которые я описываю в документации».

> 🔒 **Безопасность:** ошибки безопасности памяти (использование после освобождения, выход за границы, двойное освобождение) составляют очень большую долю серьёзных уязвимостей в программах на языках без таких проверок (например, на C — языке, который вы будете вызывать из Rust в Уроке 06.10). Безопасный Rust устраняет этот класс целиком. Каждый блок `unsafe` — место, где он может вернуться, поэтому ревьюеры читают их особенно внимательно.

## 2. Пять небезопасных возможностей

Внутри блока или функции `unsafe` можно делать ровно эти пять вещей, которые безопасному коду запрещены:

| № | Возможность | Пример |
|---|---|---|
| 1 | Разыменовать сырой указатель | `*pointer` |
| 2 | Вызвать `unsafe`-функцию или метод | `slice.get_unchecked(i)` |
| 3 | Обратиться к изменяемой `static`-переменной или изменить её | `COUNTER += 1` |
| 4 | Реализовать `unsafe`-трейт | `unsafe impl Send for Handle {}` |
| 5 | Обратиться к полю `union` | `value.int` |

Всё остальное по-прежнему проверяется: владение, заимствование, типы и проверки границ при обычной индексации работают и внутри `unsafe`.

## 3. Сырые указатели

**Сырой указатель** — это просто адрес. Бывает два вида: `*const T` для чтения и `*mut T` для чтения и записи. В отличие от ссылки, сырой указатель:

- может быть нулевым или висячим;
- не отслеживается проверкой заимствований;
- может существовать одновременно с другими указателями или ссылками на то же место.

**Создавать** сырой указатель безопасно. **Использовать** его (разыменовывать) небезопасно, потому что только вы знаете, действителен ли адрес:

```rust
fn main() {
    let mut number = 10;

    // Создание сырых указателей БЕЗОПАСНО. Небезопасно только их использование.
    let read_ptr: *const i32 = &number;
    let write_ptr: *mut i32 = &mut number;

    // SAFETY: оба указателя получены из `number`, который жив и не заимствован
    // ничем другим, пока мы пользуемся указателями.
    unsafe {
        *write_ptr += 5;
        println!("through write_ptr: {}", *write_ptr);
        println!("through read_ptr: {}", *read_ptr);
    }

    let nothing: *const i32 = std::ptr::null();
    println!("is null? {}", nothing.is_null());
}
```

```text
through write_ptr: 15
through read_ptr: 15
is null? true
```

**Что происходит:**

1. `let read_ptr: *const i32 = &number;` преобразует ссылку в сырой указатель. `unsafe` не нужен.
2. `let write_ptr: *mut i32 = &mut number;` делает то же для изменяемого.
3. Внутри `unsafe` запись `*write_ptr += 5` пишет через указатель, а `*read_ptr` читает. Оба указателя указывают на `number`, поэтому видят 15.
4. Комментарий, начинающийся с `// SAFETY:`, объясняет, **почему** блок корректен. Пишите такой для каждого блока `unsafe` (раздел 4).
5. `std::ptr::null()` создаёт нулевой указатель. `is_null()` — безопасный метод, который это проверяет.

Если забыть блок `unsafe`, будет ошибка:

```text
error[E0133]: dereference of raw pointer is unsafe and requires unsafe block
 --> src/main.rs:4:20
  |
4 |     println!("{}", *pointer);
  |                    ^^^^^^^^ dereference of raw pointer
  |
  = note: raw pointers may be null, dangling or unaligned; they can violate aliasing rules and cause data races: all of these are undefined behavior

For more information about this error, try `rustc --explain E0133`.
error: could not compile `demo` (bin "demo") due to 1 previous error
```

> 📏 **Соглашение:** преобразуйте ссылки в сырые указатели как можно позже, а обратно — как можно раньше. Делайте блок `unsafe` как можно меньше, в идеале одна-две строки.

## 4. Небезопасные функции и комментарии `// SAFETY:`

Функция помечается `unsafe fn`, когда у неё есть требования, которые компилятор проверить не может. Вызывающий должен прочитать контракт и убедиться, что он выполнен. По соглашению контракт записывается в разделе `# Safety` документационного комментария, а у каждого места вызова есть комментарий `// SAFETY:`:

```rust
/// Возвращает элемент по `index` без проверки границ.
///
/// # Safety
///
/// `index` должен быть меньше `values.len()`. Иначе поведение не определено.
unsafe fn get_unchecked(values: &[i32], index: usize) -> i32 {
    // SAFETY: вызывающий пообещал, что `index < values.len()`,
    // поэтому указатель остаётся внутри среза.
    unsafe { *values.as_ptr().add(index) }
}

fn main() {
    let values = [10, 20, 30];
    let index = 1;

    // SAFETY: `index` равен 1, а в `values` 3 элемента.
    let value = unsafe { get_unchecked(&values, index) };
    println!("value = {value}");
}
```

```text
value = 20
```

**Что происходит:**

1. `get_unchecked` читает элемент без проверки `index`. Это быстрее обычной индексации, но допустимо только при `index < values.len()`.
2. Раздел `# Safety` формулирует это условие. Это контракт, который должен выполнить **вызывающий**.
3. Внутри функции `values.as_ptr().add(index)` вычисляет адрес элемента, а `*` читает его. Для этого нужен блок `unsafe` со своим комментарием `// SAFETY:`, который ссылается на контракт.
4. В `main` вызов обёрнут в `unsafe { ... }` с комментарием `// SAFETY:`, объясняющим, почему контракт выполнен.

**Редакция 2024:** тело `unsafe fn` больше *не* является автоматически небезопасным блоком. Для каждой небезопасной операции внутри всё равно нужен `unsafe { }`. Линт `unsafe_op_in_unsafe_fn` предупреждает, если вы забыли:

```text
warning[E0133]: dereference of raw pointer is unsafe and requires unsafe block
 --> src/main.rs:2:5
  |
2 |     *pointer
  |     ^^^^^^^^ dereference of raw pointer
  |
  = note: raw pointers may be null, dangling or unaligned; they can violate aliasing rules and cause data races: all of these are undefined behavior
note: an unsafe function restricts its caller, but its body is safe by default
 --> src/main.rs:1:1
  |
1 | unsafe fn read(pointer: *const i32) -> i32 {
  | ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  = note: for more information, see <https://doc.rust-lang.org/edition-guide/rust-2024/unsafe-op-in-unsafe-fn.html>
  = note: `#[warn(unsafe_op_in_unsafe_fn)]` (part of `#[warn(rust_2024_compatibility)]`) on by default

For more information about this error, try `rustc --explain E0133`.
```

Причина: длинная `unsafe fn` иначе скрывала бы, какие строки действительно опасны.

> 📏 **Соглашение:** у каждого блока `unsafe` есть комментарий `// SAFETY:`. У каждой `unsafe fn` есть раздел документации `# Safety`. Clippy может это проверять:

```rust
#![warn(clippy::undocumented_unsafe_blocks)]

fn main() {
    let numbers = [1, 2, 3];
    let value = unsafe { *numbers.as_ptr().add(1) };
    println!("{value}");
}
```

```text
warning: unsafe block missing a safety comment
 --> src/main.rs:5:17
  |
5 |     let value = unsafe { *numbers.as_ptr().add(1) };
  |                 ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  |
  = help: consider adding a safety comment on the preceding line
  = help: for further information visit https://rust-lang.github.io/rust-clippy/rust-1.100.0/index.html#undocumented_unsafe_blocks
note: the lint level is defined here
 --> src/main.rs:1:9
  |
1 | #![warn(clippy::undocumented_unsafe_blocks)]
  |         ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
```

Линт можно включить для всего крейта с помощью `#![warn(clippy::undocumented_unsafe_blocks)]` в начале `main.rs` или `lib.rs`.

## 5. Безопасные абстракции: пишем `split_at_mut`

Самый важный навык — поместить `unsafe` **внутрь** функции и предложить снаружи **безопасную** функцию. Вызывающему не нужно об этом думать. Классический случай — разделить один срез на две изменяемые половины. Вручную на безопасном Rust это сделать нельзя:

```rust
fn split_at_mut(values: &mut [i32], mid: usize) -> (&mut [i32], &mut [i32]) {
    (&mut values[..mid], &mut values[mid..])
}

fn main() {
    let mut numbers = [1, 2, 3, 4, 5];
    let (left, right) = split_at_mut(&mut numbers, 2);
    left[0] = 100;
    right[0] = 200;
    println!("{numbers:?}");
}
```

```text
error[E0499]: cannot borrow `*values` as mutable more than once at a time
 --> src/main.rs:2:31
  |
1 | fn split_at_mut(values: &mut [i32], mid: usize) -> (&mut [i32], &mut [i32]) {
  |                         - let's call the lifetime of this reference `'1`
2 |     (&mut values[..mid], &mut values[mid..])
  |     --------------------------^^^^^^--------
  |     |     |                   |
  |     |     |                   second mutable borrow occurs here
  |     |     first mutable borrow occurs here
  |     returning this value requires that `*values` is borrowed for `'1`
  |
  = help: use `.split_at_mut(position)` to obtain two mutable non-overlapping sub-slices

For more information about this error, try `rustc --explain E0499`.
error: could not compile `demo` (bin "demo") due to 1 previous error
```

Компилятор не знает, что `..mid` и `mid..` не пересекаются, поэтому видит два заимствования `&mut` у `*values`. Стандартная библиотека решает это с помощью `unsafe` внутри `slice::split_at_mut`. Вот наша версия:

```rust
use std::slice;

/// Делит изменяемый срез на два непересекающихся изменяемых среза.
///
/// # Panics
///
/// Паникует, если `mid > values.len()`.
fn split_at_mut(values: &mut [i32], mid: usize) -> (&mut [i32], &mut [i32]) {
    let len = values.len();
    let pointer = values.as_mut_ptr();

    assert!(mid <= len, "mid must not be greater than the length");

    // SAFETY: `pointer` действителен для `len` элементов, потому что получен из живого
    // среза. Два диапазона `0..mid` и `mid..len` не пересекаются и оба остаются
    // внутри среза (проверено assert выше). `values` заимствован
    // изменяемо на всё время жизни результата, поэтому никто другой не может к нему прикоснуться.
    unsafe {
        (
            slice::from_raw_parts_mut(pointer, mid),
            slice::from_raw_parts_mut(pointer.add(mid), len - mid),
        )
    }
}

fn main() {
    let mut numbers = [1, 2, 3, 4, 5];
    let (left, right) = split_at_mut(&mut numbers, 2);
    left[0] = 100;
    right[0] = 200;
    println!("{numbers:?}");
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn splits_in_the_middle() {
        let mut data = [1, 2, 3, 4];
        let (a, b) = split_at_mut(&mut data, 2);
        assert_eq!(a, &mut [1, 2]);
        assert_eq!(b, &mut [3, 4]);
    }

    #[test]
    fn edges_give_an_empty_part() {
        let mut data = [1, 2, 3];
        let (a, b) = split_at_mut(&mut data, 0);
        assert!(a.is_empty());
        assert_eq!(b.len(), 3);
        let (a, b) = split_at_mut(&mut data, 3);
        assert_eq!(a.len(), 3);
        assert!(b.is_empty());
    }

    #[test]
    #[should_panic(expected = "mid must not be greater")]
    fn panics_when_mid_is_too_big() {
        let mut data = [1, 2, 3];
        let _ = split_at_mut(&mut data, 4);
    }
}
```

```text
[100, 2, 200, 4, 5]
```

**Что происходит:**

1. `values.as_mut_ptr()` даёт `*mut i32` на первый элемент.
2. `assert!(mid <= len, ...)` проверяет единственное условие, которое делает блок `unsafe` корректным. Если оно не выполнено, функция паникует. В этом суть: **безопасная функция проверяет всё**, что нужно небезопасным операциям.
3. `slice::from_raw_parts_mut(pointer, mid)` строит `&mut [i32]` из адреса и длины. Это `unsafe`, потому что компилятор не может знать, что память действительна.
4. `pointer.add(mid)` сдвигает адрес вперёд на `mid` элементов (не байтов). Второй срез начинается там и содержит `len - mid` элементов.
5. Комментарий `// SAFETY:` перечисляет все причины: указатель действителен, два диапазона не пересекаются, оба остаются внутри, а исходный срез заимствован изменяемо, поэтому никто другой не может до него добраться.
6. Сигнатуру `(&mut [i32], &mut [i32])` безопасно вызывать. Правило пропуска времён жизни ([Урок 06.5](05-lifetimes-in-depth.md)) связывает оба результата с `values`, поэтому они не могут его пережить.

> 🧪 **Тестирование:** тесты покрывают середину, оба края (пустая половина) и панику через `#[should_panic]`. Запускайте их ещё и под Miri (раздел 9): `cargo +nightly miri test`.

## 6. Изменяемые статические переменные

`static mut` — глобальная переменная, которую может изменить любой. С потоками это опасно, поэтому чтение и запись небезопасны (возможность 3). Редакция 2024 идёт дальше: **ссылку** на `static mut` брать запрещено, потому что ею легко злоупотребить:

```rust
static mut COUNTER: u32 = 0;

fn main() {
    unsafe {
        COUNTER += 1;
        println!("{}", COUNTER);
    }
}
```

```text
error: creating a shared reference to mutable static
 --> src/main.rs:6:24
  |
6 |         println!("{}", COUNTER);
  |                        ^^^^^^^ shared reference to mutable static
  |
  = note: shared references to mutable statics are dangerous; it's undefined behavior if the static is mutated or if a mutable reference is created for it while the shared reference lives
  = help: use a type that relies on "interior mutability" instead; to read more on this, visit <https://doc.rust-lang.org/reference/interior-mutability.html>
  = note: for more information, see <https://doc.rust-lang.org/edition-guide/rust-2024/static-mut-references.html>
  = note: `#[deny(static_mut_refs)]` (part of `#[deny(rust_2024_compatibility)]`) on by default

error: could not compile `demo` (bin "demo") due to 1 previous error
```

Макрос `println!` берёт ссылку на свои аргументы, а именно это и запрещает линт. Если без `static mut` не обойтись, сначала скопируйте значение:

```rust
static mut COUNTER: u32 = 0;

fn main() {
    // SAFETY: в этой программе один поток, поэтому никто другой не может обратиться к COUNTER.
    unsafe {
        COUNTER += 1;
        let now = COUNTER; // КОПИЯ значения, а не ссылка на статическую переменную
        println!("{now}");
    }
}
```

```text
1
```

**Что происходит:** `COUNTER += 1` пишет в статическую переменную напрямую (ссылка не создаётся). `let now = COUNTER;` копирует число в локальную переменную. Затем `println!` заимствует `now`, а не статическую переменную.

В настоящем коде `static mut` почти никогда не нужен. Используйте атомарный тип. Это число, устроенное так, чтобы много потоков могли безопасно его менять, без `unsafe`:

```rust
use std::sync::atomic::{AtomicU32, Ordering};
use std::thread;

// Ни `unsafe`, ни `mut`: атомарный тип создан для совместного использования потоками.
static COUNTER: AtomicU32 = AtomicU32::new(0);

fn main() {
    let handles: Vec<_> = (0..4)
        .map(|_| {
            thread::spawn(|| {
                for _ in 0..1000 {
                    COUNTER.fetch_add(1, Ordering::SeqCst);
                }
            })
        })
        .collect();
    for handle in handles {
        handle.join().unwrap();
    }
    println!("total = {}", COUNTER.load(Ordering::SeqCst));
}
```

```text
total = 4000
```

**Что происходит:** четыре потока по тысяче раз прибавляют 1. `fetch_add` — один неделимый шаг, поэтому ни одно обновление не теряется, и итог равен 4000. `Ordering::SeqCst` — самый строгий и безопасный выбор на данный момент; атомарные операции и порядок памяти получат целый урок на Этапе 14.

## 7. Небезопасные трейты: `Send` и `Sync`

Трейт `unsafe`, когда его реализация даёт обещание, которое компилятор проверить не может. Два самых важных — маркерные трейты `Send` («значение можно переместить в другой поток») и `Sync` («`&T` можно делить между потоками»). Компилятор реализует их автоматически, если они есть у всех полей. У сырого указателя нет ни того, ни другого, поэтому любая структура с ним их теряет:

```text
error[E0277]: `*mut i32` cannot be sent between threads safely
  --> src/main.rs:8:32
   |
 8 |       let worker = thread::spawn(move || {
   |                    ------------- ^------
   |                    |             |
   |  __________________|_____________within this `{closure@src/main.rs:8:32: 8:39}`
   | |                  |
   | |                  required by a bound introduced by this call
 9 | |         let handle = handle;
10 | |         handle.0 as usize
11 | |     });
   | |_____^ `*mut i32` cannot be sent between threads safely
   |
   = help: within `{closure@src/main.rs:8:32: 8:39}`, the trait `Send` is not implemented for `*mut i32`
note: required because it appears within the type `Handle`
  --> src/main.rs:3:15
   |
 3 | struct Handle(*mut i32);
   |               ^^^^^^^^ required by this field
note: required because `handle` is used within this closure
  --> src/main.rs:9:22
   |
 9 |         let handle = handle;
   |                      ^^^^^^
note: required by a bound in `spawn`
  --> /rustc/0ef1bc386059596dada8fb65b50425aab2ea637a/library/std/src/thread/functions.rs:125:0
```

Если вы знаете, что ваш тип безопасно отправлять, вы даёте обещание через `unsafe impl`:

```rust
use std::thread;

struct Handle(*mut i32);

// SAFETY: `Handle` используется только для доступа к «утёкшему» выделению, которого не касается никакой другой
// код, и дескриптором владеет только один поток одновременно.
unsafe impl Send for Handle {}

fn main() {
    let raw: *mut i32 = Box::into_raw(Box::new(41));
    let handle = Handle(raw);

    let worker = thread::spawn(move || {
        let handle = handle; // переместить всю структуру, а не только её поле
        // SAFETY: `handle.0` — действительный уникальный указатель из Box::into_raw.
        unsafe {
            *handle.0 += 1;
            *handle.0
        }
    });
    let result = worker.join().unwrap();
    println!("worker returned {result}");

    // SAFETY: рабочий поток завершён, а `raw` получен из Box::into_raw.
    drop(unsafe { Box::from_raw(raw) });
}
```

```text
worker returned 42
```

**Что происходит:**

1. `unsafe impl Send for Handle {}` говорит компилятору: «доверяй мне». Комментарий `// SAFETY:` объясняет почему: здесь указателем одновременно пользуется только один поток.
2. `thread::spawn(move || ...)` теперь принимает замыкание. Внутри `let handle = handle;` перемещает в замыкание всю структуру (без этого замыкание захватило бы только поле `handle.0` — голый `*mut i32`, который не `Send`).
3. Рабочий поток увеличивает число до 42 и возвращает его.
4. После `join` вызов `Box::from_raw(raw)` превращает указатель обратно в `Box`, а `drop` освобождает его. Ровно один раз.

Неверный `unsafe impl Send` — классический источник гонок данных, поэтому делайте так редко и всегда объясняйте.

## 8. Объединения (union)

У **union** несколько полей делят одни и те же байты. Rust не может знать, какое поле записано последним, поэтому чтение поля небезопасно:

```rust
#[repr(C)]
union IntOrFloat {
    int: u32,
    float: f32,
}

fn main() {
    let value = IntOrFloat { float: 1.0 };
    // SAFETY: любой набор битов — корректный u32, поэтому чтение байтов f32
    // как u32 допустимо.
    let bits = unsafe { value.int };
    println!("1.0f32 has the bits {bits:#010x}");

    // Безопасный способ получить тот же ответ:
    println!("{:#010x}", 1.0f32.to_bits());
}
```

```text
1.0f32 has the bits 0x3f800000
0x3f800000
```

**Что происходит:** число `1.0` с плавающей точкой хранится как 32-битный образ `0x3f800000` (это формат IEEE 754; название дано по Institute of Electrical and Electronics Engineers — Институту инженеров электротехники и электроники, который опубликовал стандарт; это обычный способ хранить в компьютере числа с дробной частью, вы изучите его на Этапе 12). Чтение тех же байтов как `u32` показывает этот образ. `#[repr(C)]` фиксирует раскладку, как в C. Безопасный метод `f32::to_bits` даёт тот же ответ, поэтому предпочитайте его. С union вы будете встречаться в основном при работе с кодом на C.

## 9. Неопределённое поведение и Miri

**Неопределённое поведение (UB)** означает, что программа нарушила правило языка. Компилятору разрешено считать, что UB никогда не случается, и он оптимизирует исходя из этого. Поэтому, когда UB происходит, может последовать *что угодно*: сбой, неверное значение, дыра в безопасности или совершенно нормальный сегодня результат, который сломается после обновления компилятора.

Основные виды UB в Rust:

- Использование висячего, нулевого или невыровненного указателя.
- Чтение за пределами выделенной памяти (выход за границы).
- Чтение неинициализированной памяти как значения.
- Создание некорректного значения: `bool`, не равный 0 или 1, нулевая ссылка, `char`, не являющийся допустимым символом.
- Нарушение правил алиасинга: использование `&mut`, когда другой указатель был использован для доступа к тому же месту.
- Гонка данных (два потока, одна запись, нет синхронизации).

Самое страшное: UB часто *выглядит нормально*. В этих программах есть UB, и вот что они делают при обычном запуске:

```rust
fn main() {
    let numbers = [1, 2, 3];
    let pointer = numbers.as_ptr();

    // ОШИБКА: индекс 3 — на один за концом массива.
    let value = unsafe { *pointer.add(3) };
    println!("value = {value}");
}
```

```text
value = 0
```

Она печатает какое-то число и успешно завершается, хотя прочитала память, не принадлежащую массиву. Следующая программа читает память, в которую никогда не писали:

```rust
use std::mem::MaybeUninit;

fn main() {
    let slot: MaybeUninit<u32> = MaybeUninit::uninit();

    // ОШИБКА: в память ничего не записывали, а мы утверждаем, что там корректный u32.
    let number = unsafe { slot.assume_init() };
    println!("number = {number}");
}
```

```text
number = 36850252
```

Она печатает число, похожее на случайное; на вашем компьютере оно будет другим. А программа, которая читает строку после освобождения (`ub_use_after_free.rs`, показана ниже), на этом Mac падает:

```text
Segmentation fault (exit code 139)
```

На другой машине или с другим компилятором та же программа могла бы напечатать `hello`. **«Мои тесты проходят» ничего не говорит про UB.** Нужен инструмент, который проверяет правила.

Этот инструмент — **Miri**. Это интерпретатор: вместо запуска машинного кода он выполняет вашу программу шаг за шагом на модели памяти Rust и отслеживает каждое выделение, каждый указатель и каждое заимствование. Когда что-то нарушает правило, он останавливается и сообщает, где. Он медленный (во много раз медленнее обычного запуска), требует nightly-тулчейна и может проверить только те пути, которые программа действительно выполняет, но он находит ошибки, которых не найдёт ни один обычный тест.

Используйте его на тех же программах:

```sh
cargo +nightly miri run     # запустить main под Miri
cargo +nightly miri test    # запустить все тесты под Miri
```

**Чтение за границами:**

```text
error: Undefined Behavior: memory access failed: attempting to access 4 bytes, but got alloc196+0xc which is at or beyond the end of the allocation of size 12 bytes
 --> src/main.rs:6:26
  |
6 |     let value = unsafe { *pointer.add(3) };
  |                          ^^^^^^^^^^^^^^^ Undefined Behavior occurred here
  |
  = help: this indicates a bug in the program: it performed an invalid operation, and caused Undefined Behavior
  = help: see https://doc.rust-lang.org/nightly/reference/behavior-considered-undefined.html for further information
help: alloc196 was allocated here:
 --> src/main.rs:2:9
  |
2 |     let numbers = [1, 2, 3];
  |         ^^^^^^^
```

**Использование после освобождения:**

```rust
fn main() {
    let boxed = Box::new(String::from("hello"));
    let raw: *mut String = Box::into_raw(boxed);

    // SAFETY: `raw` получен из Box::into_raw, поэтому это возвращает Box.
    drop(unsafe { Box::from_raw(raw) }); // String освобождается здесь

    // ОШИБКА: `raw` теперь висячий. Чтение через него — неопределённое поведение.
    let text = unsafe { &*raw };
    println!("{text}");
}
```

```text
error: Undefined Behavior: constructing invalid value of type &std::string::String: encountered a dangling reference (use-after-free)
 --> src/main.rs:9:25
  |
9 |     let text = unsafe { &*raw };
  |                         ^^^^^ Undefined Behavior occurred here
  |
  = help: this indicates a bug in the program: it performed an invalid operation, and caused Undefined Behavior
  = help: see https://doc.rust-lang.org/nightly/reference/behavior-considered-undefined.html for further information
```

**Чтение неинициализированной памяти:**

```text
error: Undefined Behavior: constructing invalid value of type u32: encountered uninitialized memory, but expected an integer
 --> src/main.rs:7:27
  |
7 |     let number = unsafe { slot.assume_init() };
  |                           ^^^^^^^^^^^^^^^^^^ Undefined Behavior occurred here
  |
  = help: this indicates a bug in the program: it performed an invalid operation, and caused Undefined Behavior
  = help: see https://doc.rust-lang.org/nightly/reference/behavior-considered-undefined.html for further information
```

**Некорректный `bool`:**

```rust
fn main() {
    let byte: u8 = 3;

    // ОШИБКА: bool должен быть 0 или 1. Любой другой байт — не корректный bool.
    let flag: bool = unsafe { std::mem::transmute::<u8, bool>(byte) };
    if flag {
        println!("true");
    } else {
        println!("false");
    }
}
```

```text
error: Undefined Behavior: constructing invalid value of type bool: encountered 0x03, but expected a boolean
 --> src/main.rs:5:31
  |
5 |     let flag: bool = unsafe { std::mem::transmute::<u8, bool>(byte) };
  |                               ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ Undefined Behavior occurred here
  |
  = help: this indicates a bug in the program: it performed an invalid operation, and caused Undefined Behavior
  = help: see https://doc.rust-lang.org/nightly/reference/behavior-considered-undefined.html for further information
```

**Нарушение правил алиасинга:**

```rust
fn main() {
    let mut value = 5;
    let raw: *mut i32 = &mut value;

    // SAFETY (неверно!): два `&mut` на одно место существуют одновременно.
    let first: &mut i32 = unsafe { &mut *raw };
    let second: &mut i32 = unsafe { &mut *raw };

    *second += 1;
    *first += 10; // ОШИБКА: `first` стал недействительным, когда был создан `second`
    println!("value = {value}");
}
```

```text
error: Undefined Behavior: attempting a read access using <554> at alloc196[0x0], but that tag does not exist in the borrow stack for this location
  --> src/main.rs:10:5
   |
10 |     *first += 10; // BUG: `first` was invalidated when `second` was created
   |     ^^^^^^^^^^^^ this error occurs as part of an access at alloc196[0x0..0x4]
   |
   = help: this indicates a potential bug in the program: it performed an invalid operation, but the Stacked Borrows rules it violated are still experimental
   = help: see https://github.com/rust-lang/unsafe-code-guidelines/blob/master/wip/stacked-borrows.md for further information
help: <554> was created by a Unique retag at offsets [0x0..0x4]
  --> src/main.rs:6:36
   |
 6 |     let first: &mut i32 = unsafe { &mut *raw };
   |                                    ^^^^^^^^^
help: <554> was later invalidated at offsets [0x0..0x4] by a Unique retag
  --> src/main.rs:7:37
   |
 7 |     let second: &mut i32 = unsafe { &mut *raw };
   |                                     ^^^^^^^^^
```

**Что происходит (алиасинг):**

1. `raw` — сырой указатель на `value`. Две строки создают из него два `&mut i32`. Безопасный Rust никогда не допустил бы двух живых `&mut` на одно место.
2. Создание `second` делает `first` недействительным в модели Miri. Miri называет каждый указатель «тегом» (здесь `<554>`; номер на вашем компьютере будет другим).
3. Использование `first` после этого и есть нарушение. Miri говорит, какая строка создала тег и какая строка его сделала недействительным. Это и есть точное место ошибки.
4. Правила, которые применяет Miri, называются **Stacked Borrows**. Более новая модель, **Tree Borrows**, в некоторых случаях более мягкая. Попробуйте `MIRIFLAGS="-Zmiri-tree-borrows" cargo +nightly miri run`: эта модель тоже сообщает, что в программе есть UB.
5. Правильная версия использует один `&mut` за раз, и Miri молчит:

```rust
fn main() {
    let mut value = 5;

    // Безопасный Rust говорит ясно: используйте один &mut за раз.
    let first = &mut value;
    *first += 10;
    let second = &mut value;
    *second += 1;
    println!("value = {value}");
}
```

```text
value = 16
```

> 💡 **Совет:** запускайте `cargo +nightly miri test` перед публикацией любого крейта с `unsafe`. Добавьте его в CI (Continuous Integration, непрерывная интеграция — сервер, который запускает ваши проверки при каждом изменении) на Этапе 09. Сама стандартная библиотека Rust тестируется с Miri.

Чего Miri не умеет: он не проверяет пути кода, которые никогда не выполняются, не может выполнять большинство вызовов FFI в библиотеки на C и не находит логические ошибки. Он не заменяет внимательное ревью.

## 10. Правила для написания `unsafe`

1. **Сначала попробуйте безопасный Rust.** `unsafe` ради скорости почти никогда не нужен; сначала измеряйте ([Урок 05.12](../05-rust-intermediate/12-benchmarking-basics.md)).
2. **Держите его небольшим.** Размещайте небезопасные операции в нескольких строках внутри одной функции или модуля.
3. **Делайте внешнюю часть безопасной.** Публичная функция, которая может вызвать UB из безопасного кода, *некорректна* (Упражнение 4). Проверяйте условия внутри или помечайте функцию `unsafe` и документируйте контракт.
4. **Документируйте.** `// SAFETY:` у каждого блока, `# Safety` у каждой `unsafe fn` и заметка в документации типа об инвариантах (фактах, которые должны выполняться всегда, например «`len` никогда не превышает ёмкость»).
5. **Тестируйте и запускайте Miri.**
6. **Запрещайте `unsafe` там, где он не нужен.** В крейте, который не должен содержать небезопасного кода, напишите в начале `#![forbid(unsafe_code)]`: компилятор отвергнет любой `unsafe` в этом крейте. Инструмент `cargo-geiger` считает `unsafe` в ваших зависимостях (проверки цепочки поставок рассматриваются на Этапе 10).

## Частые ошибки

### 1. `unsafe` как способ заглушить компилятор

Если компилятор отвергает ваш код, причина почти всегда реальная проблема. Не добавляйте `unsafe`, «чтобы заработало». Сначала найдите безопасное решение (`RefCell`, клон, другая структура).

### 2. Безопасная функция, которая некорректна

```rust
pub fn get(values: &[i32], index: usize) -> i32 {
    unsafe { *values.as_ptr().add(index) }
}
```

Любой может вызвать `get(&[1], 5)` из безопасного кода. Проверяйте индекс или сделайте функцию `unsafe fn`.

### 3. Нет комментария `// SAFETY:`

Если вы не можете записать, *почему* блок корректен, то, скорее всего, он некорректен.

### 4. Ссылки на `static mut`

`error: creating a shared reference to mutable static` (редакция 2024). Используйте атомарный тип, `Mutex` или скопируйте значение.

### 5. Вера в «запустилось, значит всё хорошо»

UB может выглядеть нормально. Запустите `cargo +nightly miri test`.

### 6. Два `&mut` из одного сырого указателя

Создайте ссылку, используйте её, закончите с ней; только потом создавайте следующую.

## Попробуйте сами

1. Откройте `~/projects/rust/code/p06-unsafe`.
2. Скопируйте `split_at_mut.rs` (раздел 5) в `src/main.rs`. Запустите `cargo test`.
3. Установите Miri (команды в разделе «Перед началом») и запустите `cargo +nightly miri test`. Все три теста должны пройти.
4. Намеренно сломайте код: замените `slice::from_raw_parts_mut(pointer.add(mid), len - mid)` на `slice::from_raw_parts_mut(pointer.add(mid), len - mid + 1)` (на один элемент длиннее). Сначала запустите `cargo test`: он может всё равно пройти. Затем запустите `cargo +nightly miri test` и прочитайте ошибку. Верните код.
5. Добавьте `#![warn(clippy::undocumented_unsafe_blocks)]` в начало. Удалите один комментарий `// SAFETY:` и запустите `cargo clippy`, чтобы увидеть предупреждение. Верните комментарий.
6. Сделайте коммит: `git add code/p06-unsafe` и `git commit -m "feat(p06): split_at_mut with unsafe, checked with Miri"`.

## Упражнения

### Упражнение 1 — Удвоить число через сырой указатель (легко)

Напишите `fn double_in_place(value: &mut i32)`, которая превращает `value` в `*mut i32` и удваивает число через указатель. Добавьте комментарий `// SAFETY:` и тест.

<details>
<summary>Подсказка</summary>

`let pointer: *mut i32 = value;` преобразует `&mut i32` автоматически. Затем `unsafe { *pointer *= 2; }`.

</details>

<details>
<summary>Ответ</summary>

```rust
fn double_in_place(value: &mut i32) {
    let pointer: *mut i32 = value;
    // SAFETY: `pointer` создан из действительной `&mut i32`, которая жива и
    // исключительна на весь вызов.
    unsafe {
        *pointer *= 2;
    }
}

fn main() {
    let mut number = 21;
    double_in_place(&mut number);
    println!("{number}");
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn doubles_the_value() {
        let mut x = 4;
        double_in_place(&mut x);
        assert_eq!(x, 8);
    }
}
```

```text
42
```

```text

running 1 test
.
test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

Почему это работает: указатель получен из действительной исключительной ссылки, которая живёт весь вызов, поэтому запись через него корректна. (В настоящем коде вы просто напишете `*value *= 2`.)

</details>

### Упражнение 2 — Обмен двух элементов через сырые указатели (средне)

Напишите `fn swap_two(values: &mut [i32], i: usize, j: usize)` с использованием `std::ptr::swap`. Она должна паниковать с сообщением `index out of bounds`, если индекс слишком велик. Тесты: обычный обмен, `i == j` и паника. Запустите тесты под Miri.

<details>
<summary>Подсказка</summary>

Проверьте оба индекса через `assert!`. Затем `let pointer = values.as_mut_ptr();` и `std::ptr::swap(pointer.add(i), pointer.add(j))` внутри `unsafe`.

</details>

<details>
<summary>Ответ</summary>

```rust
fn swap_two(values: &mut [i32], i: usize, j: usize) {
    assert!(i < values.len() && j < values.len(), "index out of bounds");
    let pointer = values.as_mut_ptr();
    // SAFETY: оба индекса меньше `values.len()` (проверено выше), поэтому оба
    // указателя остаются внутри среза. `ptr::swap` допускает равные указатели.
    unsafe {
        std::ptr::swap(pointer.add(i), pointer.add(j));
    }
}

fn main() {
    let mut data = [1, 2, 3, 4];
    swap_two(&mut data, 0, 3);
    println!("{data:?}");
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn swaps_two_elements() {
        let mut data = [1, 2, 3, 4];
        swap_two(&mut data, 0, 3);
        assert_eq!(data, [4, 2, 3, 1]);
    }

    #[test]
    fn same_index_changes_nothing() {
        let mut data = [1, 2, 3];
        swap_two(&mut data, 1, 1);
        assert_eq!(data, [1, 2, 3]);
    }

    #[test]
    #[should_panic(expected = "index out of bounds")]
    fn rejects_bad_index() {
        let mut data = [1, 2, 3];
        swap_two(&mut data, 0, 3);
    }
}
```

```text
[4, 2, 3, 1]
```

```text

running 3 tests
...
test result: ok. 3 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

```text

running 3 tests
...
test result: ok. 3 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.40s
```

Почему это работает: `assert!` гарантирует, что оба указателя остаются внутри среза, а это условие для `ptr::swap`. (В стандартной библиотеке есть безопасный `slice::swap`; здесь это практика.)

</details>

### Упражнение 3 — Найти двойное освобождение с помощью Miri (средне)

Эта программа освобождает одно и то же выделение дважды. Запустите её под Miri, прочитайте сообщение и исправьте так, чтобы Miri ничего не сообщал.

```rust
fn main() {
    let raw: *mut String = Box::into_raw(Box::new(String::from("data")));
    unsafe {
        drop(Box::from_raw(raw));
        drop(Box::from_raw(raw));
    }
    println!("done");
}
```

<details>
<summary>Подсказка</summary>

`Box::into_raw` отдаёт владение; `Box::from_raw` забирает его обратно. После первого `from_raw` выделение уже освобождено, поэтому второй вызов — использование после освобождения.

</details>

<details>
<summary>Ответ</summary>

Первая строка сообщения Miri (следующие строки показывают трассировку в стандартную библиотеку и `src/main.rs`):

```text
error: Undefined Behavior: constructing invalid value of type std::boxed::Box<std::string::String>: encountered a dangling box (use-after-free)
```

Исправленная программа:

```rust
fn main() {
    let raw: *mut String = Box::into_raw(Box::new(String::from("data")));
    // SAFETY: `raw` получен из Box::into_raw и превращается обратно в Box ровно один раз.
    drop(unsafe { Box::from_raw(raw) });
    println!("done");
}
```

```text
done
```

Почему это работает: `Box::from_raw` нужно вызывать ровно один раз на каждый `Box::into_raw`. Miri отслеживает выделение, поэтому замечает второй вызов на освобождённой памяти.

</details>

### Упражнение 4 — Сделать некорректную функцию корректной (сложно)

Эта безопасная функция некорректна. Покажите с помощью Miri, что она может вызвать UB, а затем напишите корректную `get`, возвращающую `Option<i32>`, с тестом для индекса в диапазоне, вне диапазона и для пустого ввода.

```rust
pub fn get_unsound(values: &[i32], index: usize) -> i32 {
    unsafe { *values.as_ptr().add(index) }
}
```

<details>
<summary>Подсказка</summary>

Сравните `index` с `values.len()` до блока `unsafe`. Верните `None`, если индекс не меньше. Вызовите `get_unsound(&numbers, 3)` для массива из трёх элементов в небольшой `main` и запустите через `cargo +nightly miri run`.

</details>

<details>
<summary>Ответ</summary>

Запуск некорректной функции с плохим индексом под Miri:

```rust
/// НЕКОРРЕКТНО: безопасный код может вызвать это с плохим индексом и получить неопределённое поведение.
pub fn get_unsound(values: &[i32], index: usize) -> i32 {
    unsafe { *values.as_ptr().add(index) }
}

fn main() {
    let numbers = [10, 20, 30];
    println!("{}", get_unsound(&numbers, 3));
}
```

```text
error: Undefined Behavior: memory access failed: attempting to access 4 bytes, but got alloc196+0xc which is at or beyond the end of the allocation of size 12 bytes
 --> src/main.rs:3:14
  |
3 |     unsafe { *values.as_ptr().add(index) }
  |              ^^^^^^^^^^^^^^^^^^^^^^^^^^^ Undefined Behavior occurred here
  |
  = help: this indicates a bug in the program: it performed an invalid operation, and caused Undefined Behavior
  = help: see https://doc.rust-lang.org/nightly/reference/behavior-considered-undefined.html for further information
help: alloc196 was allocated here:
 --> src/main.rs:7:9
  |
7 |     let numbers = [10, 20, 30];
  |         ^^^^^^^
  = note: stack backtrace:
          0: get_unsound
              at src/main.rs:3:14: 3:41
          1: main
              at src/main.rs:8:20: 8:44
```

Корректная версия:

```rust
/// НЕКОРРЕКТНО: безопасный код может вызвать это с плохим индексом и получить неопределённое поведение.
#[allow(dead_code)]
pub fn get_unsound(values: &[i32], index: usize) -> i32 {
    unsafe { *values.as_ptr().add(index) }
}

/// Корректно: границы проверяются, поэтому никакой вызывающий не может вызвать неопределённое поведение.
pub fn get(values: &[i32], index: usize) -> Option<i32> {
    if index < values.len() {
        // SAFETY: условие `index < values.len()` только что проверено.
        Some(unsafe { *values.as_ptr().add(index) })
    } else {
        None
    }
}

fn main() {
    let numbers = [10, 20, 30];
    println!("{:?} {:?}", get(&numbers, 2), get(&numbers, 3));
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn in_range_and_out_of_range() {
        let data = [7, 8, 9];
        assert_eq!(get(&data, 0), Some(7));
        assert_eq!(get(&data, 2), Some(9));
        assert_eq!(get(&data, 3), None);
        assert_eq!(get(&[], 0), None);
    }
}
```

```text
Some(30) None
```

```text

running 1 test
.
test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

Почему это работает: проверка границ превращает контракт («индекс должен быть допустимым») в условие, которое проверяет сама функция. Теперь никакой вызывающий не может вызвать UB — это и значит *корректность*. (Обычный код использует `values.get(index).copied()`; здесь важна сама логика рассуждения.)

</details>

## Вопросы для самопроверки

1. Что `unsafe` отключает в Rust?

<details>
<summary>Ответ</summary>

Ничего не отключается. Он лишь разрешает пять дополнительных операций (разыменование сырых указателей, вызов небезопасных функций, изменяемые статические переменные, небезопасные трейты, доступ к полям union). Проверка заимствований и типов продолжает действовать.

</details>

2. Безопасно ли создание сырого указателя? А его разыменование?

<details>
<summary>Ответ</summary>

Создание безопасно. Разыменование небезопасно, потому что только программист знает, действителен ли адрес, выровнен ли он и нет ли алиасинга.

</details>

3. Что пишут в комментарии `// SAFETY:` и в разделе документации `# Safety`?

<details>
<summary>Ответ</summary>

`// SAFETY:` объясняет, почему правила выполнены в этом блоке `unsafe`. `# Safety` в документации `unsafe fn` указывает, что должен гарантировать вызывающий.

</details>

4. Что такое неопределённое поведение и почему оно может выглядеть безобидно?

<details>
<summary>Ответ</summary>

Программа, нарушающая правила языка. Компилятор считает, что этого не происходит, поэтому результат непредсказуем: возможен сбой, неверные значения или работа, которая только кажется нормальной, пока другой компилятор, уровень оптимизации или машина не изменят результат.

</details>

5. Что гарантирует корректный (sound) безопасный API?

<details>
<summary>Ответ</summary>

Никакой безопасный код не может с его помощью вызвать UB при любых переданных аргументах. Если возможно неправильное использование из безопасного кода, API некорректен: функция должна проверять входные данные или быть помечена `unsafe`.

</details>

6. Что делает Miri и каковы его ограничения?

<details>
<summary>Ответ</summary>

Он интерпретирует программу шаг за шагом и сообщает об UB: висячие указатели, выход за границы, чтение неинициализированной памяти, некорректные значения и нарушения алиасинга. Он медленный, требует nightly, проверяет только выполняемые пути и не умеет выполнять большинство вызовов FFI.

</details>

7. Почему в редакции 2024 нельзя написать `println!("{}", COUNTER)` для `static mut`?

<details>
<summary>Ответ</summary>

`println!` берёт ссылку на статическую переменную, а линт `static_mut_refs` (в редакции 2024 он по умолчанию выдаёт ошибку) запрещает ссылки на изменяемые статические переменные. Сначала скопируйте значение в локальную переменную или используйте атомарный тип либо `Mutex`.

</details>

8. Что изменилось для тел `unsafe fn` в редакции 2024?

<details>
<summary>Ответ</summary>

Тело больше не является неявным небезопасным блоком. Небезопасным операциям внутри по-прежнему нужен свой `unsafe { }` (линт `unsafe_op_in_unsafe_fn` предупреждает), поэтому опасные строки остаются видимыми.

</details>

## Итоги

- `unsafe` — обещание, а не переключатель. Он разрешает пять операций: разыменование сырых указателей, вызов небезопасных функций, изменяемые статические переменные, небезопасные трейты, поля union.
- Создавать сырые указатели безопасно; использовать — нет. Делайте блоки `unsafe` крошечными и пишите `// SAFETY:` для каждого; документируйте `unsafe fn` через `# Safety`.
- Стройте безопасные абстракции: проверки внутри, безопасная сигнатура снаружи, как у `split_at_mut`.
- UB не даёт гарантий и часто выглядит нормально. Прохождение тестов не доказывает его отсутствие.
- Miri (`cargo +nightly miri test`) находит многие виды UB: висячие указатели, выход за границы, чтение неинициализированной памяти, некорректные значения, нарушения алиасинга.
- Редакция 2024: `unsafe_op_in_unsafe_fn` предупреждает, а ссылки на `static mut` запрещены. Предпочитайте атомарные типы, `Mutex` или `LazyLock`.
- Безопасный API, способный вызвать UB, некорректен. Проверяйте входные данные или помечайте функцию `unsafe`.

## Источники и что почитать

- [The Rust Programming Language — 20.1 Unsafe Rust](https://doc.rust-lang.org/book/ch20-01-unsafe-rust.html) — пять возможностей, сырые указатели, безопасные абстракции и `extern`.
- [The Rustonomicon](https://doc.rust-lang.org/nomicon/) — полное руководство по правильному написанию небезопасного Rust; начните с «Meet Safe and Unsafe».
- [The Rust Reference — Behavior considered undefined](https://doc.rust-lang.org/reference/behavior-considered-undefined.html) — официальный список UB.
- [Miri on GitHub](https://github.com/rust-lang/miri) — установка, флаги (`-Zmiri-tree-borrows`) и ограничения.
- [Edition Guide — unsafe_op_in_unsafe_fn](https://doc.rust-lang.org/edition-guide/rust-2024/unsafe-op-in-unsafe-fn.html) — изменение редакции 2024 для небезопасных функций.
- [Edition Guide — static mut references](https://doc.rust-lang.org/edition-guide/rust-2024/static-mut-references.html) — почему ссылки на `static mut` запрещены.
- [Clippy lints](https://rust-lang.github.io/rust-clippy/master/index.html) — ищите `undocumented_unsafe_blocks` и другие линты про `unsafe`.

---

[🏠 Начало](../START-HERE.md) · [← 06.5 Времена жизни подробно](05-lifetimes-in-depth.md) · [📋 Этап 06](README.md) · [06.7 Декларативные макросы →](07-declarative-macros.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
