<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 06.3 Образцы подробно](03-patterns-in-depth.md) · [📋 Этап 06](README.md) · [06.5 Времена жизни подробно →](05-lifetimes-in-depth.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/06-rust-advanced/04-interior-mutability.md) <!-- nav -->

# Урок 06.4 — Внутренняя изменяемость

> **Этап 06 — Rust: продвинутый уровень** · Урок 4 из 12 · ⏱ около 3 часов (2 занятия)

## Что вы узнаете

- Почему Rust иногда нужно менять значение через общую ссылку `&T`.
- Как `Cell<T>` и `RefCell<T>` переносят проверку заимствований с этапа компиляции на время работы.
- Как `Rc<RefCell<T>>` даёт нескольким владельцам значение, которое все они могут менять.
- `OnceCell`, `OnceLock`, `LazyCell` и `LazyLock`: значения, которые задаются один раз.
- Почему эти типы не потокобезопасны и что использовать между потоками.
- Как выбрать подходящий тип и какие бывают классические ошибки.

## Перед началом

- Вы прошли [Урок 06.3 — Образцы подробно](03-patterns-in-depth.md).
- Вы знаете правила заимствования ([Урок 03.5](../03-ownership-and-borrowing/05-mutable-references-and-the-borrow-rules.md)), `Box` и `Rc` ([Урок 05.9](../05-rust-intermediate/09-smart-pointers.md)), а также потоки с `Arc` и `Mutex` ([Урок 05.10](../05-rust-intermediate/10-threads-channels-and-shared-state.md)). Если вы встречали там `RefCell`, этот урок идёт дальше; никаких предположений он не делает.
- Создайте учебный проект: `cargo new code/p06-interior-mutability`.

## Новые слова

| Слово | Значение |
|---|---|
| **внутренняя изменяемость** (interior mutability) | Изменение значения через общую ссылку `&T`. Безопасность этого гарантирует сам тип. |
| **`Cell<T>`** | Коробка, которая позволяет через `&` получить копию (`get`) и записать новое значение (`set`). Для небольших значений `Copy`. |
| **`RefCell<T>`** | Коробка, которая выдаёт заимствования `&T` и `&mut T` и проверяет правила заимствования во время работы программы. |
| **`Ref` / `RefMut`** | Значения-стражи, которые возвращают `borrow()` и `borrow_mut()`. Пока страж существует, заимствование активно. |
| **проверка во время работы** (run-time check) | Проверка, которая выполняется при работе программы, а не при компиляции. Если она не прошла, программа паникует. |
| **`OnceCell` / `OnceLock`** | Коробка, которая сначала пуста, заполняется ровно один раз и затем выдаёт общие ссылки. `OnceLock` работает и между потоками. |
| **`LazyCell` / `LazyLock`** | Значение, которое вычисляется замыканием при первом чтении. `LazyLock` работает между потоками. |
| **тестовый дублёр / мок** (test double / mock) | Подделка, которую используют в тесте вместо настоящего объекта. Она часто записывает, как её использовали. |
| **мемоизация** (memoization) | Сохранение результата функции, чтобы не вычислять один и тот же вызов дважды. |
| **`UnsafeCell<T>`** | Низкоуровневый «кирпичик» под всеми типами этого урока. Единственный законный способ менять данные за `&` (требует `unsafe`). |
| **`Send` / `Sync`** | Маркерные трейты. `Send`: значение можно передать в другой поток. `Sync`: общую ссылку `&T` можно использовать из многих потоков. |

## 1. Задача: иногда правило слишком строгое

В Rust есть простое правило из [Урока 03.5](../03-ownership-and-borrowing/05-mutable-references-and-the-borrow-rules.md): в любой момент существует **либо много общих ссылок (`&T`), либо одна изменяемая (`&mut T`)**. Компилятор проверяет это до запуска программы.

Иногда метод снаружи выглядит как «только чтение», но внутри ему нужно что-то запомнить. Хороший пример — счётчик посещений:

```rust
struct Counter {
    visits: u32,
}

impl Counter {
    fn visit(&self) {
        self.visits += 1;
    }
}

fn main() {
    let counter = Counter { visits: 0 };
    counter.visit();
    println!("{}", counter.visits);
}
```

```text
error[E0594]: cannot assign to `self.visits`, which is behind a `&` reference
 --> src/main.rs:7:9
  |
7 |         self.visits += 1;
  |         ^^^^^^^^^^^^^^^^ `self` is a `&` reference, so it cannot be written to
  |
help: consider changing this to be a mutable reference
  |
6 |     fn visit(&mut self) {
  |               +++
```

**Что происходит:**

1. `visit(&self)` принимает общую ссылку, поэтому не может писать в `self.visits`.
2. Компилятор предлагает `&mut self`. Это работает, но тогда каждому вызывающему нужна переменная `mut` и исключительный доступ даже ради такой мелочи, как счётчик.

**Внутренняя изменяемость** — это решение: тип, который разрешает изменения через `&self`, но по-своему сохраняет правила безопасными. Есть две идеи:

- **Сделать нарушение правил невозможным** (`Cell`): никогда не выдавать ссылку на содержимое.
- **Проверять правила во время работы программы** (`RefCell`): считать активные заимствования и паниковать при нарушении.

> 📏 **Соглашение:** внутренняя изменяемость — инструмент, а не вариант по умолчанию. Предпочитайте обычный `&mut self`. Используйте `Cell`/`RefCell`, когда дизайну нужен общий доступ (несколько владельцев, обратный вызов, тестовый дублёр).

## 2. `Cell<T>`: заменить значение целиком

`Cell<T>` никогда не даёт ссылку на значение внутри. Можно только *скопировать значение наружу* (`get`) или *положить новое значение внутрь* (`set`). Поскольку ссылки на содержимое не существует, ничего сломать нельзя.

```rust
use std::cell::Cell;

struct Counter {
    visits: Cell<u32>,
}

impl Counter {
    fn visit(&self) {
        self.visits.set(self.visits.get() + 1);
    }
}

fn main() {
    let counter = Counter {
        visits: Cell::new(0),
    };
    counter.visit();
    counter.visit();
    println!("visits: {}", counter.visits.get());

    let old = counter.visits.replace(100);
    println!("replaced {old} with {}", counter.visits.get());

    let taken = counter.visits.take();
    println!("took {taken}, left {}", counter.visits.get());
}
```

```text
visits: 2
replaced 2 with 100
took 100, left 0
```

**Что происходит:**

1. `visits: Cell<u32>` заменяет обычный `u32`. Метод `visit(&self)` теперь компилируется: `set` принимает `&self`.
2. `self.visits.set(self.visits.get() + 1)` читает копию, прибавляет единицу и сохраняет результат.
3. `replace(100)` сохраняет 100 и возвращает старое значение (2).
4. `take()` возвращает текущее значение (100) и оставляет вместо него значение по умолчанию (`0` для чисел).
5. У `Cell` нет затрат на подсчёт заимствований: он так же дёшев, как обычное значение.

`get` требует `T: Copy`. Для остальных типов используйте `replace` или `take`:

```text
error[E0599]: the method `get` exists for struct `Cell<String>`, but its trait bounds were not satisfied
 --> src/main.rs:5:21
  |
5 |     let copy = name.get();
  |                     ^^^
  |
  = note: the following trait bounds were not satisfied:
          `String: Copy`

For more information about this error, try `rustc --explain E0599`.
error: could not compile `demo` (bin "demo") due to 1 previous error
```

Для `Cell<String>` вы напишете `name.take()` (забирает текст, оставляя пустую `String`) или `name.replace(new_value)`. Если нужно *заглянуть внутрь* значения без `Copy`, используйте `RefCell`.

## 3. `RefCell<T>`: проверка заимствований во время работы

`RefCell<T>` владеет значением и позволяет его заимствовать, как это делает компилятор, но проверки происходят **во время работы программы**:

- `borrow()` возвращает `Ref<T>`: общее заимствование. Их может быть много одновременно.
- `borrow_mut()` возвращает `RefMut<T>`: исключительное заимствование. Оно может быть только одно, и одновременно не должно существовать ни одного `Ref`.
- Если нарушить правило, программа **паникует**. `try_borrow()` и `try_borrow_mut()` вместо этого возвращают `Result`.

Стражи возвращают заимствование при уничтожении (в конце своей области видимости), точно как `MutexGuard` из [Урока 05.10](../05-rust-intermediate/10-threads-channels-and-shared-state.md).

```rust
use std::cell::RefCell;

struct Logger {
    lines: RefCell<Vec<String>>,
}

impl Logger {
    fn log(&self, message: &str) {
        self.lines.borrow_mut().push(message.to_string());
    }

    fn count(&self) -> usize {
        self.lines.borrow().len()
    }
}

fn main() {
    let logger = Logger {
        lines: RefCell::new(Vec::new()),
    };
    logger.log("starting");
    logger.log("working");
    println!("{} lines", logger.count());
    println!("{:?}", logger.lines.borrow());

    // try_borrow_mut сообщает о конфликте вместо паники
    let first = logger.lines.borrow();
    match logger.lines.try_borrow_mut() {
        Ok(_) => println!("got a mutable borrow"),
        Err(e) => println!("cannot borrow mutably: {e}"),
    }
    println!("first borrow still sees {} lines", first.len());
}
```

```text
2 lines
["starting", "working"]
cannot borrow mutably: RefCell already borrowed
first borrow still sees 2 lines
```

**Что происходит:**

1. `lines: RefCell<Vec<String>>` живёт внутри структуры, методы которой принимают `&self`.
2. `self.lines.borrow_mut().push(...)` берёт исключительное заимствование на время этого оператора, добавляет элемент и отпускает заимствование.
3. `self.lines.borrow().len()` — общее заимствование на один оператор.
4. `first` — это `Ref`, который мы нарочно держим живым. Пока он существует, `try_borrow_mut()` не удаётся и возвращает `Err`, что печатает `RefCell already borrowed`.
5. Сравните с обычным `&mut`: компилятор отверг бы такую программу. `RefCell` её принимает и проверяет во время работы.

Если вызвать `borrow_mut()`, пока заимствование живо, вы получите панику, а не сообщение на этапе компиляции:

```rust
use std::cell::RefCell;

fn main() {
    let data = RefCell::new(vec![1, 2, 3]);
    let reader = data.borrow();
    println!("reader sees {} items", reader.len());
    data.borrow_mut().push(4); // читатель ещё жив: здесь будет паника
    println!("never printed");
}
```

```text
reader sees 3 items

thread 'main' (4075730) panicked at src/main.rs:7:10:
RefCell already borrowed
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
```

> ⚠️ **Частая ошибка:** `RefCell` превращает ошибку компиляции в возможный сбой в продакшне. Делайте каждый `borrow()` / `borrow_mut()` коротким: не храните стража в переменной, которая живёт долго, и не вызывайте другой код, пока держите его.

> 🧪 **Тестирование:** `RefCell` — стандартный инструмент для *тестового дублёра*. Настоящий объект принимает `&self`, поэтому и поддельный должен записывать вызовы через `&self`. Вы создадите такой в Упражнении 2.

## 4. `Rc<RefCell<T>>`: несколько владельцев, которые могут менять значение

`Rc<T>` даёт совместное владение, но только доступ на чтение. Положите внутрь `RefCell`, и каждый владелец сможет менять значение:

```rust
use std::cell::RefCell;
use std::rc::Rc;

fn main() {
    // Rc: несколько владельцев. RefCell: изменение через общий дескриптор.
    let cart = Rc::new(RefCell::new(vec![String::from("bread")]));

    let alice = Rc::clone(&cart);
    let bob = Rc::clone(&cart);

    alice.borrow_mut().push(String::from("milk"));
    bob.borrow_mut().push(String::from("eggs"));

    println!("{:?}", cart.borrow());
    println!("owners: {}", Rc::strong_count(&cart));
}
```

```text
["bread", "milk", "eggs"]
owners: 3
```

**Что происходит:**

1. `Rc::new(RefCell::new(vec![...]))` создаёт в куче один список со счётчиком ссылок и флагом заимствования.
2. `Rc::clone(&cart)` не копирует список. Он создаёт ещё одного владельца и увеличивает счётчик. Теперь `cart`, `alice` и `bob` указывают на один и тот же список.
3. `alice.borrow_mut().push(...)` и `bob.borrow_mut().push(...)` меняют его по очереди. Каждое заимствование заканчивается на точке с запятой.
4. `Rc::strong_count` показывает 3 владельцев.

Этот приём часто встречается в графах, деревьях с указателями на родителя и обработчиках событий. Будьте осторожны с циклами (`Rc`, который через другой `Rc` указывает на самого себя): счётчик никогда не достигнет нуля, и память утечёт.

## 5. Не для потоков

`Cell`, `RefCell` и `Rc` ничего не проверяют про другие потоки. Компилятор знает об этом через маркерные трейты `Send` и `Sync`: `Rc` не `Send`, а `RefCell` не `Sync`. Поэтому компилятор вас остановит:

```rust
use std::cell::RefCell;
use std::rc::Rc;
use std::thread;

fn main() {
    let counter = Rc::new(RefCell::new(0));
    let for_thread = Rc::clone(&counter);
    let handle = thread::spawn(move || {
        *for_thread.borrow_mut() += 1;
    });
    handle.join().unwrap();
    println!("{}", counter.borrow());
}
```

```text
error[E0277]: `Rc<RefCell<i32>>` cannot be sent between threads safely
  --> src/main.rs:8:32
   |
 8 |       let handle = thread::spawn(move || {
   |                    ------------- ^------
   |                    |             |
   |  __________________|_____________within this `{closure@src/main.rs:8:32: 8:39}`
   | |                  |
   | |                  required by a bound introduced by this call
 9 | |         *for_thread.borrow_mut() += 1;
10 | |     });
   | |_____^ `Rc<RefCell<i32>>` cannot be sent between threads safely
   |
   = help: within `{closure@src/main.rs:8:32: 8:39}`, the trait `Send` is not implemented for `Rc<RefCell<i32>>`
note: required because `for_thread` is used within this closure
  --> src/main.rs:9:10
   |
 9 |         *for_thread.borrow_mut() += 1;
   |          ^^^^^^^^^^
note: required by a bound in `spawn`
  --> /rustc/0ef1bc386059596dada8fb65b50425aab2ea637a/library/std/src/thread/functions.rs:125:0
```

У потокобезопасных аналогов такая же форма:

| Один поток | Несколько потоков |
|---|---|
| `Rc<T>` | `Arc<T>` |
| `RefCell<T>` | `Mutex<T>` или `RwLock<T>` |
| `Cell<T>` | атомарные типы вроде `AtomicU32` (Этап 14) |
| `OnceCell<T>` | `OnceLock<T>` |
| `LazyCell<T>` | `LazyLock<T>` |

```rust
use std::sync::{Arc, Mutex};
use std::thread;

fn main() {
    // Потокобезопасная версия: Arc вместо Rc, Mutex вместо RefCell.
    let counter = Arc::new(Mutex::new(0));
    let for_thread = Arc::clone(&counter);
    let handle = thread::spawn(move || {
        *for_thread.lock().unwrap() += 1;
    });
    handle.join().unwrap();
    println!("{}", counter.lock().unwrap());
}
```

```text
1
```

**Что происходит:** это программа из ошибки, только с `Arc` и `Mutex`. `lock()` ждёт, пока ни один другой поток не держит блокировку, а `unwrap()` обрабатывает случай, когда другой поток запаниковал, удерживая её.

## 6. Значения, задаваемые один раз: `OnceCell`, `OnceLock`, `LazyCell`, `LazyLock`

Некоторые значения записываются ровно один раз, а потом только читаются: конфигурация, загруженная при старте, или таблица, вычисленная при первом использовании. Для них в стандартной библиотеке есть особые ячейки:

```rust
use std::cell::{LazyCell, OnceCell};
use std::sync::{LazyLock, OnceLock};

// Глобальное значение, которое задаётся ровно один раз, из любого потока.
static CONFIG: OnceLock<String> = OnceLock::new();

// Глобальное значение, которое вычисляется при первом чтении.
static PRIMES: LazyLock<Vec<u32>> = LazyLock::new(|| {
    println!("(computing primes)");
    (2..30).filter(|n| (2..*n).all(|d| n % d != 0)).collect()
});

fn main() {
    // OnceCell: однопоточное «записать один раз, читать много раз»
    let nickname: OnceCell<String> = OnceCell::new();
    println!("before: {:?}", nickname.get());
    println!("set: {:?}", nickname.set(String::from("Sasha")));
    println!("second set: {:?}", nickname.set(String::from("Alex")));
    println!("after: {:?}", nickname.get());

    // get_or_init: задать при первом использовании
    let cell: OnceCell<u32> = OnceCell::new();
    println!("{}", cell.get_or_init(|| 41 + 1));
    println!("{}", cell.get_or_init(|| 0)); // замыкание больше не запускается

    // OnceLock: то же самое, но для static и потоков
    println!("{:?}", CONFIG.set(String::from("production")));
    println!("{:?}", CONFIG.set(String::from("testing")));
    println!("config = {}", CONFIG.get().unwrap());

    // LazyLock: инициализируется при первом обращении
    println!("before first use");
    println!("first prime: {}", PRIMES[0]);
    println!("{} primes below 30", PRIMES.len());

    // LazyCell: ленивое значение без поддержки потоков
    let lazy = LazyCell::new(|| {
        println!("(lazy value computed)");
        7 * 6
    });
    println!("answer: {}", *lazy);
    println!("answer again: {}", *lazy);
}
```

```text
before: None
set: Ok(())
second set: Err("Alex")
after: Some("Sasha")
42
42
Ok(())
Err("testing")
config = production
before first use
(computing primes)
first prime: 2
10 primes below 30
(lazy value computed)
answer: 42
answer again: 42
```

**Что происходит:**

1. `OnceCell::new()` создаёт пустую ячейку. `get()` возвращает `None`.
2. `set(value)` заполняет её и возвращает `Ok(())`. Второй `set` возвращает `Err` с отвергнутым значением: ячейка больше никогда не меняется.
3. `get_or_init(closure)` возвращает значение и запускает замыкание, только если ячейка ещё пуста.
4. `static CONFIG: OnceLock<String>` — глобальная переменная, которую можно заполнить ровно один раз, даже если много потоков пытаются одновременно. Обычный `OnceCell` не может быть `static`, потому что он не `Sync`.
5. `static PRIMES: LazyLock<Vec<u32>>` хранит замыкание. Оно запускается при **первом обращении** (`PRIMES[0]`), что видно по «(computing primes)», напечатанному после «before first use». Следующие обращения используют результат.
6. `LazyCell` — то же самое без поддержки потоков, для локальной переменной.

> 📏 **Соглашение:** для глобальных констант, которые требуют работы при создании (`HashMap`, регулярные выражения, таблицы), используйте `LazyLock`. Он заменяет старый макрос `lazy_static!` и большинство применений крейта `once_cell`. Оба теперь есть в стандартной библиотеке.

## 7. Что под капотом: `UnsafeCell`

Все перечисленные типы построены на `std::cell::UnsafeCell<T>`. Это единственный способ в Rust получить `*mut T` (*сырой указатель*, через который можно писать; урок 06.6 объясняет сырые указатели) из `&UnsafeCell<T>`. Без него компилятор считает, что данные за `&T` никогда не меняются, и оптимизирует исходя из этого предположения; изменение таких данных было бы *неопределённым поведением*: правила языка нарушены, и скомпилированная программа может делать что угодно, в том числе казаться работающей, пока внезапно не сломается. Напрямую `UnsafeCell` вы не используете, кроме случаев, когда пишете собственную ячейку или блокировку (Урок 06.6 объясняет `unsafe`).

## 8. Какой тип использовать

| Тип | Потоки | Что даёт | Цена | Для чего |
|---|---|---|---|---|
| `Cell<T>` | один | копии и замену | нет | Счётчики, флаги, небольшие значения `Copy` |
| `RefCell<T>` | один | `&T` и `&mut T` с проверкой во время работы | счётчик, паника при конфликте | Тестовые дублёры, кэши, графы с `Rc` |
| `OnceCell<T>` / `LazyCell<T>` | один | запись один раз, затем `&T` | крошечная | Значения, вычисляемые при первом использовании |
| `OnceLock<T>` / `LazyLock<T>` | много | запись один раз, затем `&T` | небольшая | Глобальная конфигурация, таблицы |
| `Mutex<T>` / `RwLock<T>` | много | исключительный (или общий) доступ через блокировку | блокировка | Общее изменяемое состояние между потоками |

## 9. Классика: кэш при `&self`

*Мемоизирующая* функция запоминает свои результаты. С `RefCell` кэш может жить внутри структуры, метод которой принимает `&self`:

```rust
use std::cell::RefCell;
use std::collections::HashMap;

struct Fib {
    cache: RefCell<HashMap<u64, u64>>,
    computed: RefCell<u32>,
}

impl Fib {
    fn new() -> Self {
        Fib {
            cache: RefCell::new(HashMap::new()),
            computed: RefCell::new(0),
        }
    }

    // &self, а не &mut self: вызывающие могут делить один Fib.
    fn get(&self, n: u64) -> u64 {
        // Временное заимствование заканчивается в конце этого оператора.
        let cached = self.cache.borrow().get(&n).copied();
        if let Some(value) = cached {
            return value;
        }
        let value = if n < 2 {
            n
        } else {
            self.get(n - 1) + self.get(n - 2)
        };
        *self.computed.borrow_mut() += 1;
        self.cache.borrow_mut().insert(n, value);
        value
    }
}

fn main() {
    let fib = Fib::new();
    println!("fib(50) = {}", fib.get(50));
    println!("computed {} values", fib.computed.borrow());
    println!("fib(50) again = {}", fib.get(50));
    println!("computed {} values", fib.computed.borrow());
}
```

```text
fib(50) = 12586269025
computed 51 values
fib(50) again = 12586269025
computed 51 values
```

**Что происходит:**

1. `get(&self, n)` ищет значение в кэше. Временный `Ref` от `self.cache.borrow()` уничтожается в конце оператора `let cached = ...;`, поэтому заимствование освобождается **до** рекурсивных вызовов.
2. Если значения нет, `get` вызывает сама себя для `n - 1` и `n - 2`. Каждый из этих вызовов тоже ненадолго заимствует кэш.
3. Результат сохраняется через `borrow_mut().insert(...)`.
4. Для `fib(50)` вычисляется 51 значение, по одному разу каждое (числа от 0 до 50). Второй вызов читает кэш, поэтому счётчик остаётся 51. Без кэша для этой рекурсии понадобились бы миллиарды вызовов.

Теперь та же функция с ошибкой: страж хранится в переменной и живёт до конца функции.

```rust
use std::cell::RefCell;
use std::collections::HashMap;

struct Fib {
    cache: RefCell<HashMap<u64, u64>>,
}

impl Fib {
    fn get(&self, n: u64) -> u64 {
        let cache = self.cache.borrow(); // этот Ref живёт до конца функции
        if let Some(value) = cache.get(&n) {
            return *value;
        }
        let value = if n < 2 {
            n
        } else {
            self.get(n - 1) + self.get(n - 2)
        };
        self.cache.borrow_mut().insert(n, value); // конфликт с `cache`
        value
    }
}

fn main() {
    let fib = Fib {
        cache: RefCell::new(HashMap::new()),
    };
    println!("{}", fib.get(5));
}
```

```text

thread 'main' (4077050) panicked at src/main.rs:19:20:
RefCell already borrowed
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
```

**Что происходит:** переменная `cache` держит `Ref` всю функцию. Когда рекурсия доходит до базового случая, строка `self.cache.borrow_mut().insert(...)` просит исключительное заимствование, пока собственное общее заимствование этой функции ещё существует. Компилятор этого не видит; `RefCell` паникует во время работы. Исправление — версия выше: не держите стража; возьмите то, что нужно (`.copied()`), и дайте стражу уничтожиться.

> 🔒 **Безопасность:** паника, зависящая от ввода (например, запрос, который вызывает двойное заимствование), может использоваться, чтобы уронить сервер. Считайте паники `RefCell` ошибками и покрывайте код тестами.

## Частые ошибки

### 1. Хранение `Ref` или `RefMut` в долгоживущей переменной

Страж удерживает заимствование. Используйте значение в одном выражении или скопируйте нужное наружу и дайте стражу уничтожиться (см. сломанный пример `Fib` в разделе 9).

### 2. `Cell<String>` с `get`

`E0599 ... trait bounds were not satisfied: String: Copy`. Используйте `take()` или `replace()`, либо `RefCell`.

### 3. Отправка `Rc<RefCell<T>>` в другой поток

`E0277 ... cannot be sent between threads safely`. Перейдите на `Arc<Mutex<T>>`.

### 4. Циклы ссылок с `Rc`

Два `Rc`, указывающие друг на друга, никогда не освобождаются. Для обратного указателя используйте `Weak<T>` (`Rc::downgrade`; см. главу 15.6 Rust Book в источниках ниже).

### 5. `RefCell` как способ не разбираться с проверкой заимствований

Если вы тянетесь к `Rc<RefCell<T>>` в каждой структуре, дизайну, вероятно, нужен более ясный владелец. Используйте его, когда совместное владение действительно необходимо.

## Попробуйте сами

1. Откройте `~/projects/rust/code/p06-interior-mutability`.
2. Скопируйте пример `Logger` из раздела 3 в `src/main.rs`.
3. Добавьте метод `fn last(&self) -> Option<String>`, который возвращает **копию** последней записанной строки (`self.lines.borrow().last().cloned()`).
4. Вызовите `logger.last()` дважды и напечатайте результаты. Вы должны увидеть `Some("working")` дважды.
5. Сломайте программу нарочно: в `main` напишите `let _guard = logger.lines.borrow_mut();`, а затем вызовите `logger.log("x")`. Запустите и прочитайте панику. Удалите эту строку.
6. Выполните `cargo fmt` и `cargo clippy`, затем сделайте коммит: `git add code/p06-interior-mutability` и `git commit -m "feat(p06): logger with RefCell"`.

## Упражнения

### Упражнение 1 — Кликер на `Cell` (легко)

Создайте `struct Clicker` с полем `Cell<u32>`. Методы `click(&self)` (прибавляет единицу) и `reset(&self) -> u32` (ставит 0 и возвращает старое значение). Напишите тест, который дважды вызывает `click` через общую ссылку `&Clicker` и проверяет счётчик, и тест для `reset`.

<details>
<summary>Подсказка</summary>

`self.count.set(self.count.get() + 1)` для `click` и `self.count.replace(0)` для `reset`.

</details>

<details>
<summary>Ответ</summary>

```rust
use std::cell::Cell;

struct Clicker {
    count: Cell<u32>,
}

impl Clicker {
    fn new() -> Self {
        Clicker {
            count: Cell::new(0),
        }
    }

    fn click(&self) {
        self.count.set(self.count.get() + 1);
    }

    fn reset(&self) -> u32 {
        self.count.replace(0)
    }
}

fn main() {
    let clicker = Clicker::new();
    clicker.click();
    clicker.click();
    clicker.click();
    println!("count = {}", clicker.count.get());
    println!("reset returned {}", clicker.reset());
    println!("count = {}", clicker.count.get());
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn counts_clicks_through_a_shared_reference() {
        let clicker = Clicker::new();
        let shared = &clicker;
        shared.click();
        shared.click();
        assert_eq!(clicker.count.get(), 2);
    }

    #[test]
    fn reset_returns_old_value() {
        let clicker = Clicker::new();
        clicker.click();
        assert_eq!(clicker.reset(), 1);
        assert_eq!(clicker.count.get(), 0);
    }
}
```

```text
count = 3
reset returned 3
count = 0
```

```text

running 2 tests
..
test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

Почему это работает: `Cell::set` и `Cell::replace` принимают `&self`, поэтому методам не нужен `&mut self`. Тест это доказывает: он вызывает `click` через `&Clicker`.

</details>

### Упражнение 2 — Мок-уведомитель (средне)

Даны `trait Notifier { fn send(&self, message: &str); }` и `fn check_temperature(notifier: &dyn Notifier, celsius: i32)` (она отправляет `too hot: N` для 40 и выше и `too cold: N` для -20 и ниже). Напишите `MockNotifier` внутри модуля с тестами. Он записывает сообщения в `RefCell<Vec<String>>`. Проверьте, что для температур 20, 45 и -30 отправлено ровно два сообщения.

<details>
<summary>Подсказка</summary>

`send(&self, ...)` принимает `&self`, поэтому моку нужен `RefCell`, чтобы добавлять в вектор: `self.sent.borrow_mut().push(message.to_string())`. В конце сравните с `*mock.sent.borrow()`.

</details>

<details>
<summary>Ответ</summary>

```rust
trait Notifier {
    fn send(&self, message: &str);
}

struct PrintNotifier;

impl Notifier for PrintNotifier {
    fn send(&self, message: &str) {
        println!("NOTIFY: {message}");
    }
}

fn check_temperature(notifier: &dyn Notifier, celsius: i32) {
    if celsius >= 40 {
        notifier.send(&format!("too hot: {celsius}"));
    } else if celsius <= -20 {
        notifier.send(&format!("too cold: {celsius}"));
    }
}

fn main() {
    let notifier = PrintNotifier;
    for t in [20, 45, -30] {
        check_temperature(&notifier, t);
    }
}

#[cfg(test)]
mod tests {
    use super::*;
    use std::cell::RefCell;

    // Тестовый дублёр: он записывает, что его просили отправить.
    struct MockNotifier {
        sent: RefCell<Vec<String>>,
    }

    impl Notifier for MockNotifier {
        fn send(&self, message: &str) {
            self.sent.borrow_mut().push(message.to_string());
        }
    }

    #[test]
    fn sends_only_for_extreme_values() {
        let mock = MockNotifier {
            sent: RefCell::new(Vec::new()),
        };
        check_temperature(&mock, 20);
        check_temperature(&mock, 45);
        check_temperature(&mock, -30);
        assert_eq!(*mock.sent.borrow(), vec!["too hot: 45", "too cold: -30"]);
    }
}
```

```text
NOTIFY: too hot: 45
NOTIFY: too cold: -30
```

```text

running 1 test
.
test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

Почему это работает: функция принимает `&dyn Notifier` ([Урок 06.1](01-trait-objects-and-dynamic-dispatch.md)), поэтому тест может передать мок вместо печатающего уведомителя. Мок определён в модуле тестов, поэтому в настоящую программу он не попадает.

</details>

### Упражнение 3 — Два замыкания, один счётчик (средне)

Напишите `fn make_counter() -> (impl Fn(), impl Fn() -> u32)`. Первое замыкание прибавляет единицу к скрытому счётчику. Второе возвращает текущее значение. Оба замыкания должны делить один и тот же счётчик. Проверьте тестом, что после двух увеличений `read()` возвращает 2.

<details>
<summary>Подсказка</summary>

Создайте `let count = Rc::new(Cell::new(0));`, клонируйте `Rc` один раз для первого замыкания, а оригинал переместите во второе. Обоим замыканиям нужно ключевое слово `move`.

</details>

<details>
<summary>Ответ</summary>

```rust
use std::cell::Cell;
use std::rc::Rc;

// Два замыкания, которые делят один счётчик.
fn make_counter() -> (impl Fn(), impl Fn() -> u32) {
    let count = Rc::new(Cell::new(0));
    let for_increment = Rc::clone(&count);
    let increment = move || for_increment.set(for_increment.get() + 1);
    let read = move || count.get();
    (increment, read)
}

fn main() {
    let (increment, read) = make_counter();
    increment();
    increment();
    println!("read() = {}", read());
    increment();
    println!("read() = {}", read());
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn read_sees_increments() {
        let (increment, read) = make_counter();
        assert_eq!(read(), 0);
        increment();
        increment();
        assert_eq!(read(), 2);
    }
}
```

```text
read() = 2
read() = 3
```

```text

running 1 test
.
test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

Почему это работает: `Rc` даёт обоим замыканиям владение одним и тем же `Cell`, а `Cell` позволяет `set` через общий дескриптор. Каждому замыканию достаточно `Fn`, не `FnMut`.

</details>

### Упражнение 4 — Настройки, загружаемые один раз (сложно)

Напишите `fn settings() -> &'static HashMap<&'static str, u32>`, которая строит карту `{"port": 8080, "workers": 4}` при первом вызове и каждый раз возвращает одну и ту же карту. Печатайте `(loading settings)` внутри инициализатора. Напишите тест: два вызова возвращают один и тот же адрес, а `port` равен 8080.

<details>
<summary>Подсказка</summary>

Используйте `static SETTINGS: OnceLock<HashMap<...>> = OnceLock::new();` и `SETTINGS.get_or_init(|| ...)`. Адреса сравнивайте через `std::ptr::eq(a, b)`.

</details>

<details>
<summary>Ответ</summary>

```rust
use std::collections::HashMap;
use std::sync::OnceLock;

static SETTINGS: OnceLock<HashMap<&'static str, u32>> = OnceLock::new();

fn settings() -> &'static HashMap<&'static str, u32> {
    SETTINGS.get_or_init(|| {
        println!("(loading settings)");
        HashMap::from([("port", 8080), ("workers", 4)])
    })
}

fn main() {
    println!("port = {}", settings()["port"]);
    println!("workers = {}", settings()["workers"]);
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn settings_are_loaded_once_and_shared() {
        let first = settings();
        let second = settings();
        assert!(std::ptr::eq(first, second));
        assert_eq!(first["port"], 8080);
    }
}
```

```text
(loading settings)
port = 8080
workers = 4
```

```text

running 1 test
.
test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

Почему это работает: `get_or_init` запускает замыкание только для первого вызывающего (даже при нескольких потоках) и затем возвращает `&'static` на сохранённое значение. `std::ptr::eq` показывает, что оба вызова получили одну и ту же карту.

</details>

## Вопросы для самопроверки

1. Что такое внутренняя изменяемость?

<details>
<summary>Ответ</summary>

Изменение значения через общую ссылку `&T`. Тип (`Cell`, `RefCell`, `Mutex`, ...) делает это безопасным: либо никогда не выдаёт ссылок, либо иным способом проверяет правила заимствования.

</details>

2. Почему `Cell<T>::get` можно использовать только при `T: Copy`?

<details>
<summary>Ответ</summary>

`get` возвращает копию значения. Для типа без `Copy` (как `String`) это было бы перемещение или дорогое клонирование, чего `Cell` не делает. Используйте `take`, `replace` или `RefCell`.

</details>

3. Когда `RefCell` паникует?

<details>
<summary>Ответ</summary>

Когда вы вызываете `borrow_mut()` при любом другом активном заимствовании или `borrow()` при активном `RefMut`. Проверка происходит во время работы программы.

</details>

4. Чем отличаются `borrow_mut()` и `try_borrow_mut()`?

<details>
<summary>Ответ</summary>

`borrow_mut()` паникует, если значение уже заимствовано. `try_borrow_mut()` возвращает `Result`: `Err`, когда заимствование активно.

</details>

5. Что даёт `Rc<RefCell<Vec<i32>>>` по сравнению с `Rc<Vec<i32>>`?

<details>
<summary>Ответ</summary>

Все владельцы могут менять вектор. `Rc<Vec<i32>>` делит вектор, но только для чтения.

</details>

6. Почему нельзя отправить `Rc<RefCell<i32>>` в другой поток и что использовать?

<details>
<summary>Ответ</summary>

`Rc` использует счётчик ссылок, который не потокобезопасен, поэтому он не `Send`; `RefCell` не `Sync`. Используйте `Arc<Mutex<i32>>` (или `Arc<RwLock<i32>>`).

</details>

7. Чем `OnceLock` отличается от `LazyLock`?

<details>
<summary>Ответ</summary>

`OnceLock` создаётся пустым, и вы заполняете его вызовом `set` или `get_or_init`. `LazyLock` создаётся вместе с замыканием-инициализатором и вычисляет значение автоматически при первом обращении.

</details>

8. Почему опасно держать `Ref` в долгоживущей переменной?

<details>
<summary>Ответ</summary>

Заимствование активно, пока жив страж. Любой `borrow_mut()` за это время вызовет панику. Делайте стражей короткими или копируйте нужное наружу и давайте стражу уничтожиться.

</details>

## Итоги

- Внутренняя изменяемость меняет значение через `&T`. Это инструмент для общего доступа, а не вариант по умолчанию.
- `Cell<T>` копирует значения внутрь и наружу, без ссылок и без затрат. `RefCell<T>` выдаёт `&T`/`&mut T` и проверяет правила во время работы; нарушение вызывает панику.
- `Rc<RefCell<T>>` — совместное владение плюс изменение; следите за циклами ссылок.
- `OnceCell`/`OnceLock` заполняются один раз; `LazyCell`/`LazyLock` вычисляются при первом использовании. Предпочитайте `LazyLock` вместо `lazy_static!`.
- Эти однопоточные типы не `Send`/`Sync`. Между потоками используйте `Arc`, `Mutex`, `RwLock`, `OnceLock`, `LazyLock` (и атомарные операции, Этап 14).
- Делайте стражей `Ref`/`RefMut` короткими. Не держите заимствование через вызов, который может заимствовать снова.
- Всё построено на `UnsafeCell`.

## Источники и что почитать

- [The Rust Programming Language — 15.5 RefCell<T> and the Interior Mutability Pattern](https://doc.rust-lang.org/book/ch15-05-interior-mutability.html) — официальное объяснение, включая `Rc<RefCell<T>>`.
- [The Rust Programming Language — 15.6 Reference Cycles Can Leak Memory](https://doc.rust-lang.org/book/ch15-06-reference-cycles.html) — как циклы приводят к утечкам и как их предотвращает `Weak<T>`.
- [std::cell module](https://doc.rust-lang.org/std/cell/index.html) — обзор `Cell`, `RefCell`, `OnceCell`, `LazyCell` и `UnsafeCell` с руководством по выбору.
- [std::cell::RefCell](https://doc.rust-lang.org/std/cell/struct.RefCell.html) — все методы, включая `try_borrow`.
- [std::sync::OnceLock](https://doc.rust-lang.org/std/sync/struct.OnceLock.html) — потокобезопасная ячейка для однократной записи.
- [std::sync::LazyLock](https://doc.rust-lang.org/std/sync/struct.LazyLock.html) — ленивые статические значения в стандартной библиотеке.
- [std::cell::UnsafeCell](https://doc.rust-lang.org/std/cell/struct.UnsafeCell.html) — основа внутренней изменяемости.

---

[🏠 Начало](../START-HERE.md) · [← 06.3 Образцы подробно](03-patterns-in-depth.md) · [📋 Этап 06](README.md) · [06.5 Времена жизни подробно →](05-lifetimes-in-depth.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
