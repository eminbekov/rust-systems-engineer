<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 09.8 Фаззинг](08-fuzzing.md) · [📋 Этап 09](README.md) · [09.10 Логирование и трассировка →](10-logging-and-tracing.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/09-testing-debugging-benchmarking/09-debugging-with-lldb-and-rustrover.md) <!-- nav -->

# Урок 09.9 — Отладка подробно: LLDB и RustRover

> **Этап 09 — Тестирование, отладка и бенчмарки** · Урок 9 из 14 · ⏱ около 135 минут

## Что вы узнаете

- Как работает отладчик и небольшой набор команд LLDB, который покрывает 95% реальной отладки: точки останова, `bt`, `frame variable`, пошаговое выполнение и `continue`.
- Как с помощью **условной точки останова** и **точки наблюдения** выяснить, *кто* изменил значение.
- Как отлаживать упавший тест и как остановить программу ровно там, где она паникует.
- Как читать трассировку стека с `RUST_BACKTRACE=1`.
- Те же шаги в RustRover, словами, и то, чего отладчик не умеет (релизные сборки).

## Перед началом

- Вы прошли [Урок 09.8 — Фаззинг](08-fuzzing.md).
- Вы уже проводили первый сеанс отладки в [Уроке 02.9 — Основы отладки](../02-programming-basics/09-debugging-basics.md): точка останова, пошаговое выполнение и наблюдение за переменными, в RustRover и в LLDB. Этот урок идёт глубже и добавляет точки наблюдения, тесты и паники.
- Вы знаете структуры и методы ([Урок 04.2](../04-structs-enums-collections-errors/02-methods-and-associated-functions.md)) и переполнение целых чисел ([Урок 02.2](../02-programming-basics/02-data-types.md)).
- Код лежит в `~/projects/rust/code/p09-debugging`.

## Новые слова

| Слово | Значение |
|---|---|
| **отладчик** (debugger) | Программа, которая запускает другую программу под своим контролем, чтобы вы могли её приостановить, заглянуть внутрь и выполнять пошагово. |
| **LLDB** | Отладчик проекта LLVM. Это отладчик, которым пользуются RustRover и macOS. |
| **`rust-lldb`** | Небольшой скрипт-обёртка, который запускает LLDB и загружает вспомогательные файлы, чтобы типы Rust, такие как `Vec` и `String`, показывались читаемо. Поставляется с Rust. |
| **отладочная информация** (debug information) | Дополнительные данные внутри скомпилированной программы, которые связывают машинный код с именами файлов, строками и именами переменных. В отладочных сборках она есть. |
| **точка останова** (breakpoint) | Метка на строке. Когда программа до неё доходит, она останавливается до выполнения строки. |
| **условная точка останова** (conditional breakpoint) | Точка останова, которая срабатывает, только если выражение истинно, например `qty > 1`. |
| **точка наблюдения** (watchpoint) | Метка на *ячейке памяти*. Программа останавливается, когда в неё кто-то пишет. |
| **кадр стека** (stack frame) | Часть стека вызовов, принадлежащая одной работающей функции: её аргументы и локальные переменные. |
| **трассировка стека** (backtrace, `bt`) | Список кадров стека от функции, которая работает сейчас, до `main`. Отвечает на вопрос «как мы сюда попали?». |
| **шаг с обходом / шаг внутрь / шаг наружу** (step over / into / out) | Выполнить текущую строку целиком / войти в вызываемую функцию / выполнять, пока текущая функция не вернётся. |
| **REPL** (Read-Eval-Print Loop, цикл «прочитать — выполнить — вывести») | Приглашение, где вы вводите команду, программа вычисляет её и печатает результат. Приглашение `(lldb)` работает именно так. |

## 1. Ошибка, на которую мы охотимся

Отладчик нужен в момент, когда вы не знаете, *почему* программа делает не то. Нам нужна программа с настоящей ошибкой. Создайте проект:

```bash
cd ~/projects/rust/code
cargo new p09-debugging
cd p09-debugging
mkdir -p src/bin
```

Поместите в `src/lib.rs`. Это корзина покупок, которая хранит текущую сумму в центах:

```rust
/// One line of a shopping cart.
#[derive(Debug)]
pub struct Item {
    pub name: String,
    pub price_cents: u64,
    pub qty: u32,
}

#[derive(Debug, Default)]
pub struct Cart {
    pub items: Vec<Item>,
    pub total_cents: u64,
}

impl Cart {
    pub fn new() -> Cart {
        Cart::default()
    }

    pub fn add(&mut self, name: &str, price_cents: u64, qty: u32) {
        let line_total = price_cents * u64::from(qty);
        self.items.push(Item {
            name: name.to_string(),
            price_cents,
            qty,
        });
        self.total_cents += line_total;
    }

    /// Takes `percent` percent off the total.
    pub fn apply_discount(&mut self, percent: u64) {
        let discount = self.total_cents * (percent / 100);
        self.total_cents -= discount;
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn ten_percent_off() {
        let mut cart = Cart::new();
        cart.add("pen", 250, 4);
        cart.add("book", 1500, 1);
        cart.apply_discount(10);
        assert_eq!(cart.total_cents, 2250);
    }
}
```

`src/main.rs`:

```rust
use p09_debugging::Cart;

fn main() {
    let mut cart = Cart::new();
    cart.add("pen", 250, 4);
    cart.add("book", 1500, 1);
    cart.apply_discount(10);
    println!("total: {} cents", cart.total_cents);
}
```

(`src/main.rs` использует библиотеку по имени крейта `p09_debugging`, с подчёркиванием. Дополнительные программы лежат в `src/bin/`; мы воспользуемся этим в разделе 6.)

```bash
cargo run
cargo test
```

```text
total: 2500 cents
```

```text
---- tests::ten_percent_off stdout ----

thread 'tests::ten_percent_off' (4658211) panicked at src/lib.rs:47:9:
assertion `left == right` failed
  left: 2500
 right: 2250
```

Четыре ручки по 250 и одна книга за 1500 дают 2500. Скидка десять процентов должна дать 2250, но сумма всё ещё 2500: скидка ничего не сделала. Где ошибка? Возможно, вы уже её видите, но представьте, что нет, и пусть её покажет отладчик.

## 2. Запуск LLDB

Обычная команда `cargo build` (профиль `dev`) уже включает отладочную информацию, как вы видели в Уроке 02.9. Запустите LLDB на скомпилированной программе через `rust-lldb`:

```bash
cargo build
rust-lldb target/debug/p09-debugging
```

Вы получите приглашение `(lldb)`. Это **REPL** (Read-Eval-Print Loop): вы вводите команду, LLDB выполняет её и печатает результат. Нужные команды:

| Команда (короткая форма) | Что делает |
|---|---|
| `breakpoint set --file lib.rs --line 32` (`b lib.rs:32`) | Остановиться, когда будет достигнута строка 32 файла `lib.rs`. |
| `breakpoint set --name apply_discount` (`b apply_discount`) | Остановиться в начале функции. |
| `run` (`r`) | Запустить программу. |
| `continue` (`c`) | Выполнять до следующей остановки. |
| `next` (`n`) | Шаг с обходом: выполнить текущую строку, остановиться на следующей. |
| `step` (`s`) | Шаг внутрь: войти в функцию, вызываемую на этой строке. |
| `finish` | Шаг наружу: выполнять, пока текущая функция не вернётся. |
| `bt` | Напечатать трассировку стека. `bt 3` печатает только три самых внутренних кадра. |
| `frame variable` (`fr v`) | Показать переменные текущего кадра. |
| `frame select 2` / `up` / `down` | Перейти к другому кадру трассировки. |
| `print EXPR` (`p`) | Вычислить выражение и напечатать его. |
| `breakpoint list` / `breakpoint delete 1` | Показать / удалить точки останова. |
| `quit` (`q`) | Выйти из LLDB. |

Сеанс отладки трудно показать одним блоком, потому что вы вводите, а LLDB отвечает. В этом уроке каждый блок показывает команды после приглашения `(lldb)`, а под ними ответы. (Мы сократили вывод: LLDB печатает длинные строки `Process …` и пути, которые на каждом компьютере свои. Список команд можно запустить и без ввода вручную: `rust-lldb -b -o 'command 1' -o 'command 2' -- target/debug/p09-debugging`, где `-b` означает «пакетный режим» (batch), а каждый `-o` — одну команду. Удобно, чтобы повторять сеанс.)

> 💡 **Совет:** используйте `rust-lldb`, а не обычный `lldb`. Обычный `lldb` ничего не знает о типах Rust и для `self` показывает только сырой адрес:
>
> ```text
> (p09_debugging::Cart *) self = 0x000000016fdfe0c0
> ```
>
> `rust-lldb` показывает, что внутри: `self = 0x000000016fdfe030 {items:size=2, total_cents:2500}`.

## 3. Точки останова, стек и переменные

Поставьте точку останова внутри `apply_discount`, запустите и осмотритесь:

```text
(lldb) breakpoint set --file lib.rs --line 32
Breakpoint 1: where = p09-debugging`<p09_debugging::Cart>::apply_discount + 28 at lib.rs:32:24, address = 0x0000000100000bd4
(lldb) run
Process 34145 stopped
* thread #1, name = 'main', queue = 'com.apple.main-thread', stop reason = breakpoint 1.1
      frame #0: 0x0000000100000bd4 p09-debugging`<p09_debugging::Cart>::apply_discount(self={items:size=2, total_cents:2500}, percent=10) at lib.rs:32:24
   29  	
   30  	    /// Takes `percent` percent off the total.
   31  	    pub fn apply_discount(&mut self, percent: u64) {
-> 32  	        let discount = self.total_cents * (percent / 100);
    	                       ^
   33  	        self.total_cents -= discount;
   34  	    }
   35  	}
(lldb) bt 2
* thread #1, name = 'main', queue = 'com.apple.main-thread', stop reason = breakpoint 1.1
  * frame #0: 0x0000000100000bd4 p09-debugging`<p09_debugging::Cart>::apply_discount(self={items:size=2, total_cents:2500}, percent=10) at lib.rs:32:24
    frame #1: 0x0000000100000ad0 p09-debugging`p09_debugging::main at main.rs:7:10
(lldb) frame variable
(p09_debugging::Cart *) self = 0x000000016fdfe030 {items:size=2, total_cents:2500}
(unsigned long) percent = 10
```

**Как это читать:**

1. Точка останова «разрешилась» в настоящее место: функция `apply_discount`, строка 32. `+ 28` — число байтов от начала функции.
2. Программа **остановилась перед** выполнением строки 32. Стрелка `->` отмечает следующую строку, которая будет выполнена. LLDB показывает строки вокруг неё.
3. `bt 2` показывает два кадра. Кадр 0 — функция, которая работает; кадр 1 — её вызывающий: `main` в `main.rs:7`, строка `cart.apply_discount(10);`. Звёздочка `*` отмечает выбранный кадр.
4. `frame variable` показывает переменные: `self` — указатель на `Cart` с 2 товарами и суммой 2500, а `percent` равен 10. Всё, как мы ожидаем. `discount` ещё нет: строка 32 не выполнена.

Теперь выполните строку 32 командой `next` и посмотрите снова:

```text
(lldb) next
Process 34145 stopped
* thread #1, name = 'main', queue = 'com.apple.main-thread', stop reason = step over
      frame #0: 0x0000000100000c00 p09-debugging`<p09_debugging::Cart>::apply_discount(self={items:size=2, total_cents:2500}, percent=10) at lib.rs:33:9
   32  	        let discount = self.total_cents * (percent / 100);
-> 33  	        self.total_cents -= discount;
(lldb) frame variable
(p09_debugging::Cart *) self = 0x000000016fdfe030 {items:size=2, total_cents:2500}
(unsigned long) percent = 10
(unsigned long) discount = 0
(lldb) p percent / 100
(unsigned long) 0
```

Вот оно: `discount = 0`. А выражение `percent / 100` равно `0`, потому что оба числа целые: `10 / 100` — это `0` (целочисленное деление отбрасывает дробную часть, Урок 02.3). Мы хотели `total * percent / 100`. Умножение должно выполняться до деления.

> ⚠️ **Частая ошибка:** `print` (`p`) в LLDB не понимает Rust. Об этом говорят его собственные сообщения: *«Expression evaluation in Rust not supported. Falling back to default language.»* Он вычисляет простые вещи вроде `percent / 100`, доступ к полям через `->` (`p self->total_cents`) и переменные, но не вызовы методов: `p self->items.len()` даёт `error: called object type 'unsigned long' is not a function or function pointer`. Используйте `frame variable` (он работает всегда), а для `Vec` смотрите `size=` в его краткой записи.

Исправьте строку и проверьте тестом:

```rust
        let discount = self.total_cents * percent.min(100) / 100;
```

`percent.min(100)` также не даёт проценту превысить 100 (раздел 6 покажет, зачем это нужно). Запустите `cargo test`: тест проходит. В следующих разделах оставьте в файле **ошибочный** вариант строки (`self.total_cents * (percent / 100)`), чтобы ваш вывод совпадал с уроком; исправьте её в конце.

## 4. Условные точки останова и точки наблюдения

Точка останова на строке внутри `add` срабатывает дважды: для ручки и для книги. В цикле, который выполняется миллион раз, останавливаться каждый раз невозможно. **Условная точка останова** срабатывает только когда условие истинно. Остановимся в `add` только при `qty > 1`:

```text
(lldb) breakpoint set --file lib.rs --line 27 --condition "qty > 1"
Breakpoint 1: where = p09-debugging`<p09_debugging::Cart>::add + 160 at lib.rs:27:9, address = 0x0000000100000ce8
(lldb) run
Process 36375 stopped
* thread #1, name = 'main', queue = 'com.apple.main-thread', stop reason = breakpoint 1.1
      frame #0: ... p09-debugging`<p09_debugging::Cart>::add(self={items:size=1, total_cents:0}, name="pen", price_cents=250, qty=4) at lib.rs:27:9
-> 27  	        self.total_cents += line_total;
(lldb) frame variable
(p09_debugging::Cart *) self = 0x000000016fdfe030 {items:size=1, total_cents:0}
(&str) name = "pen" {
  [0] = 'p'
  [1] = 'e'
  [2] = 'n'
}
(unsigned long) price_cents = 250
(unsigned int) qty = 4
(unsigned long) line_total = 1000
(lldb) breakpoint list
1: file = 'lib.rs', line = 27, exact_match = 0, locations = 1, resolved = 1, hit count = 1
Condition: qty > 1
```

Книга (`qty = 1`) программу не остановила; остановила только ручка. `hit count = 1` говорит, что точка сработала один раз.

Теперь другой вопрос: *кто меняет `total_cents`?* В большой программе записать в него могут многие функции. **Точка наблюдения** говорит LLDB: «остановись, как только в эту память что-то запишут». В CPU (Central Processing Unit, центральный процессор — чип, выполняющий инструкции) для этого есть специальные регистры, поэтому программа не становится медленной. Остановитесь в `main` после `Cart::new()` и наблюдайте за полем:

```text
(lldb) breakpoint set --file main.rs --line 5
(lldb) run
(lldb) watchpoint set variable cart.total_cents
Watchpoint created: Watchpoint 1: addr = 0x16fdfe048 size = 8 state = enabled type = m
    watchpoint spec = 'cart.total_cents'
(lldb) continue

Watchpoint 1 hit:
old value: 0
new value: 1000
* thread #1, name = 'main', queue = 'com.apple.main-thread', stop reason = watchpoint 1
      frame #0: ... p09-debugging`<p09_debugging::Cart>::add(self={items:size=1, total_cents:1000}, name="pen", price_cents=250, qty=4) at lib.rs:28:6
(lldb) continue

Watchpoint 1 hit:
old value: 1000
new value: 2500
      frame #0: ... p09-debugging`<p09_debugging::Cart>::add(self={items:size=2, total_cents:2500}, name="book", price_cents=1500, qty=1) at lib.rs:28:6
(lldb) continue
total: 2500 cents
```

**Как это читать:**

1. `size = 8` — число наблюдаемых байтов: это `u64`.
2. Первая остановка показывает старое и новое значение и функцию, которая записала: `add`, для ручки. Вторая остановка: снова `add`, для книги. Кадр показан на строке 28, то есть на строке после записи, потому что CPU сообщает о записи после её завершения.
3. После второго `continue` программа напечатала `total: 2500 cents`, и **третьей остановки в `apply_discount` не было**. `apply_discount` выполнила `self.total_cents -= discount`, но вычла 0, поэтому сохранённое значение не изменилось, а LLDB сообщает о точке наблюдения только при изменении значения. Это подтверждает то, что показала точка останова: скидка равна нулю.

(Если продолжать вводить `continue`, будет ещё одна остановка: точка наблюдения за локальной переменной `main` остаётся активной, когда `main` завершается, и память стека используется другим кодом, например `fwrite` — библиотечной функцией, стоящей за `println!`. В отчёте кадром будет `libsystem_c.dylib`. Это не ошибка вашей программы. Когда закончите, удалите точку наблюдения командой `watchpoint delete`.)

> 🔒 **Безопасность:** точки наблюдения — также способ расследовать порчу памяти: «кто перезаписал этот буфер?». Инструмент останавливается на самой записи. Вместе с `bt` он даёт полный путь плохой записи.

## 5. Отладка упавшего теста

Те же инструменты работают для тестов. Тестовый бинарный файл — тоже программа, но нужно найти его файл. `cargo test --no-run` собирает тесты, не запуская их, и печатает путь:

```bash
cargo test --no-run
```

```text
  Executable unittests src/lib.rs (target/debug/build/p09-debugging/c14c00238fa9a8c4/out/p09_debugging-c14c00238fa9a8c4)
  Executable unittests src/main.rs (target/debug/build/p09-debugging/0366ed2c2e057707/out/p09_debugging-0366ed2c2e057707)
```

(Хеш в имени, `c14c00238fa9a8c4`, на вашем компьютере другой. Скопируйте путь из строки `unittests src/lib.rs`.) Запустите LLDB на этом файле. Всё после `--` передаётся тестовой программе как аргументы: имя теста и `--exact` (запустить только тест с точно таким именем):

```bash
rust-lldb -- target/debug/build/p09-debugging/c14c00238fa9a8c4/out/p09_debugging-c14c00238fa9a8c4 tests::ten_percent_off --exact
```

```text
(lldb) breakpoint set --file lib.rs --line 46
(lldb) run
running 1 test
* thread #2, name = 'tests::ten_percent_off', stop reason = breakpoint 1.1
-> 46  	        cart.apply_discount(10);
(lldb) step
* thread #2, name = 'tests::ten_percent_off', stop reason = step in
      frame #0: ... <p09_debugging::Cart>::apply_discount(self={items:size=2, total_cents:2500}, percent=10) at lib.rs:32:24
-> 32  	        let discount = self.total_cents * (percent / 100);
(lldb) step
-> 33  	        self.total_cents -= discount;
(lldb) frame variable
(p09_debugging::Cart *) self = 0x0000000170006628 {items:size=2, total_cents:2500}
(unsigned long) percent = 10
(unsigned long) discount = 0
(lldb) finish
* thread #2, name = 'tests::ten_percent_off', stop reason = step out
-> 46  	        cart.apply_discount(10);
(lldb) next
-> 47  	        assert_eq!(cart.total_cents, 2250);
(lldb) frame variable cart
(p09_debugging::Cart) cart = {items:size=2, total_cents:2500}
(lldb) continue
test tests::ten_percent_off ... FAILED
```

**Что происходит:**

1. Точка останова стоит на строке `cart.apply_discount(10);` внутри теста (строка 46 файла).
2. Тест выполняется в **потоке №2**, а не в главном: тестовая обвязка (harness — программа, которую собирает `cargo test`) запускает по одному потоку на тест. LLDB следует за ним.
3. `step` заходит **внутрь** `apply_discount`. Второй `step` переходит со строки 32 на строку 33, и видно `discount = 0`.
4. `finish` выполняет, пока `apply_discount` не вернётся, и мы снова в тесте, на строке вызова (вызов только что завершился). `next` переходит на строку `assert_eq!`, а `frame variable cart` показывает неверную сумму.
5. `continue` дает тесту дойти до конца, и обвязка печатает `FAILED`.

> 🧪 **Тестирование:** отладчик не заменяет тест. Сначала сделайте сбой **повторяемым** в маленьком тесте, затем с помощью отладчика поймите его, затем исправьте и оставьте тест. Остаётся именно тест.

## 6. Остановка на панике и трассировка стека

Добавьте вторую программу, которая делает другую ошибку: скидка 250 процентов. Создайте `src/bin/overflow.rs`:

```rust
use p09_debugging::Cart;

fn main() {
    let mut cart = Cart::new();
    cart.add("pen", 250, 4);
    cart.apply_discount(250);
    println!("total: {} cents", cart.total_cents);
}
```

(С ошибочной строкой `250 / 100` равно `2`, поэтому скидка вдвое больше суммы, и `total -= discount` ушло бы ниже нуля. Беззнаковое число не может быть отрицательным.)

```bash
cargo run --bin overflow
```

```text
thread 'main' (4664184) panicked at src/lib.rs:33:9:
attempt to subtract with overflow
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
```

Сообщение о панике уже называет файл и строку. Но кто вызвал `apply_discount` с плохим числом? Переменная окружения `RUST_BACKTRACE=1` заставляет Rust напечатать трассировку стека паники:

```bash
RUST_BACKTRACE=1 cargo run --bin overflow
```

```text
thread 'main' (4664334) panicked at src/lib.rs:33:9:
attempt to subtract with overflow
stack backtrace:
   0: __rustc::rust_begin_unwind
             at /rustc/0ef1bc386059596dada8fb65b50425aab2ea637a/library/std/src/panicking.rs:676:5
   1: core::panicking::panic_fmt
             at /rustc/0ef1bc386059596dada8fb65b50425aab2ea637a/library/core/src/panicking.rs:80:14
   2: core::panicking::panic_const::panic_const_sub_overflow
             at /rustc/0ef1bc386059596dada8fb65b50425aab2ea637a/library/core/src/panicking.rs:175:17
   3: <p09_debugging::Cart>::apply_discount
             at ./src/lib.rs:33:9
   4: overflow::main
             at ./src/bin/overflow.rs:6:10
   5: <fn() as core::ops::function::FnOnce<()>>::call_once
             at /rustc/0ef1bc386059596dada8fb65b50425aab2ea637a/library/core/src/ops/function.rs:250:5
note: Some details are omitted, run with `RUST_BACKTRACE=full` for a verbose backtrace.
```

**Как читать трассировку:**

1. Она перечисляет работавшие функции, **самая внутренняя первой**. Номера от 0 до 2 — механизм паники внутри стандартной библиотеки Rust; их можно пропустить.
2. **Кадр 3 — первая из ваших собственных функций**: `Cart::apply_discount`, `src/lib.rs:33:9`. Именно здесь вычитание переполнилось.
3. Кадр 4 — вызывающий: `overflow::main` в `src/bin/overflow.rs:6:10`. Это строка, передавшая 250.
4. Строки после кадра 5 опущены: это код, запускающий `main`. Правило: **найдите первый кадр, файл которого ваш.**

Трассировка показывает *где*, но не *значения*. Для значений остановите программу на панике и посмотрите кадр. Паники проходят через функцию `rust_panic`; поставьте на неё точку останова:

```text
(lldb) breakpoint set --name rust_panic
(lldb) run
thread 'main' panicked at src/lib.rs:33:9:
attempt to subtract with overflow
* thread #1, name = 'main', queue = 'com.apple.main-thread', stop reason = breakpoint 1.1
      frame #0: 0x0000000100035374 overflow`__rustc::rust_panic at panicking.rs:883:16 [opt]
(lldb) bt 8
  * frame #0: ... overflow`__rustc::rust_panic at panicking.rs:883:16 [opt]
    frame #1: ... overflow`std::panicking::panic_with_hook at panicking.rs:836:5 [opt]
    ...
    frame #6: ... overflow`core::panicking::panic_const::panic_const_sub_overflow at panicking.rs:175:17 [opt]
    frame #7: ... overflow`<p09_debugging::Cart>::apply_discount(self={items:size=1, total_cents:1000}, percent=250) at lib.rs:33:9
(lldb) frame select 7
frame #7: ... overflow`<p09_debugging::Cart>::apply_discount(self={items:size=1, total_cents:1000}, percent=250) at lib.rs:33:9
-> 33  	        self.total_cents -= discount;
(lldb) frame variable
(p09_debugging::Cart *) self = 0x000000016fdfe030 {items:size=1, total_cents:1000}
(unsigned long) percent = 250
(unsigned long) discount = 2000
```

Теперь картина полная: `discount = 2000` больше, чем `total_cents = 1000`. Кадры стандартной библиотеки помечены `[opt]` (оптимизированы) и не содержат информации о переменных (`frame variable` для них говорит «no variable information is available»), поэтому выберите свой кадр командой `frame select 7`.

С `percent.min(100)` в исправлении процент выше 100 обрезается до 100, и сумма останавливается на нуле.

> 📏 **Соглашение:** смотрите на **первый кадр в вашем коде**, а не на верх трассировки. Не включайте `RUST_BACKTRACE=1` в продакшене по умолчанию, если вывод идёт пользователям; это инструмент разработчика.

## 7. То же самое в RustRover

RustRover тоже использует LLDB, поэтому всё описанное выше есть в виде кнопок. Пользуйтесь тем, что быстрее.

1. **Точка останова:** щёлкните в сером поле рядом с номером строки. Появится красная точка. **Условная точка останова:** щёлкните по красной точке правой кнопкой (или нажмите `Ctrl+Shift+F8`) и впишите выражение, например `qty > 1`, в поле **Condition**. **Логирующая точка останова:** удерживая `Shift`, щёлкните по полю; она печатает значение, не останавливая программу.
2. **Запуск:** щёлкните зелёный значок жука рядом с `main` или рядом с функцией `#[test]` и выберите **Debug**. RustRover соберёт проект с отладочной информацией и сам запустит LLDB. Чтобы отлаживать один тест, используйте значок жука рядом с ним.
3. **Осмотр:** в окне инструментов **Debug** есть область **Threads & Variables**. Список кадров — это ваш `bt`; щёлкните кадр, чтобы выбрать его (как `frame select`). Переменные этого кадра перечислены, и можно раскрывать `self` и `items`. Это представление использует те же средства печати типов Rust, что и `rust-lldb`.
4. **Шаги:** кнопки панели **Step Over** (как `next`), **Step Into** (как `step`), **Step Out** (как `finish`), **Run to Cursor** и **Resume Program** (как `continue`).
5. **Вычисление:** действие **Evaluate Expression** позволяет ввести выражение и увидеть его значение, а **Watches** держит выражения на экране, пока вы идёте по шагам.
6. **Остановка на панике:** откройте вкладку **LLDB** (консоль) окна инструментов **Debug** (см. следующий пункт) и введите `breakpoint set --name rust_panic`, как в разделе 6. Когда программа остановится, выберите в списке кадров свой кадр, чтобы увидеть его переменные.
7. **Точка наблюдения:** способ добавить её отличается в разных версиях RustRover, поэтому посмотрите меню переменной в списке переменных (щёлкните её правой кнопкой). Если не нашли, у окна **Debug** есть вкладка консоли **LLDB**, где можно ввести `watchpoint set variable cart.total_cents` точно так же, как в разделе 4.

## 8. Чего отладчик не умеет

Отладчику нужны отладочная информация и программа, которая всё ещё похожа на ваш исходный код. Соберите ту же программу в режиме release (добавив отладочную информацию переменной окружения) и попробуйте ту же точку останова:

```bash
CARGO_PROFILE_RELEASE_DEBUG=true cargo build --release --bin p09-debugging
rust-lldb target/release/p09-debugging
```

```text
(lldb) breakpoint set --file lib.rs --line 33
Breakpoint 1: no locations (pending).
WARNING:  Unable to resolve breakpoint to any actual locations.
(lldb) run
total: 2500 cents
Process 38249 exited with status = 0 (0x00000000)
```

**Что происходит:** оптимизатор превратил маленькую функцию в несколько инструкций внутри `main` (Урок 05.12 показал, что компилятор убирает работу). «Строки 33» в машинном коде нет, поэтому точка останова остаётся *ожидающей* (pending) и никогда не срабатывает. Правило: **отлаживайте отладочную сборку** (`cargo build`, `cargo test`), а если ошибка есть только в режиме release, используйте логирование (Урок 09.10).

Другие ограничения:

- Отладчик **меняет тайминги**. Ошибка в потоках (гонка) может исчезнуть, когда вы останавливаете один поток. Логирование обычно меняет тайминги меньше.
- Он **интерактивен**. Ошибку, которая случилась на сервере в 3 часа ночи, нужно понимать по логам, а не по отладчику.
- Его повторять дольше, чем тест. Как только вы поняли ошибку, **напишите тест**.

## Частые ошибки

**1. «Breakpoint 1: no locations (pending)».** Имя файла или строка неверны, либо сборка оптимизирована. Сверьте `breakpoint set --file lib.rs --line 32` с настоящим файлом и собирайте обычным `cargo build`.

**2. Обычный `lldb` и одни сырые адреса.** Запустите вместо него `rust-lldb`.

**3. Отладка старого бинарного файла.** LLDB запускает тот файл, который вы ему дали. После правки кода снова выполните `cargo build` (или `cargo test --no-run`), иначе вы отлаживаете вчерашнюю программу. (Изменившийся хеш в имени тестового файла — тоже признак новой сборки.)

**4. Попытка `p some_vec.len()`.** Вызовы методов в LLDB для Rust недоступны. Используйте `frame variable some_vec` и читайте `size=`.

**5. Шаги по коду стандартной библиотеки.** Если `step` заходит в незнакомые функции вроде `core::fmt`, выходите из них командой `finish`, а на неинтересных строках предпочитайте `next`.

## Попробуйте сами

1. Создайте `code/p09-debugging` с тремя файлами из раздела 1 (оставьте ошибочную строку). Убедитесь, что `cargo test` падает.
2. Используйте `rust-lldb -b -o … -- target/debug/p09-debugging` или интерактивный сеанс, чтобы остановиться в `apply_discount`, выполнить `frame variable`, `next`, `frame variable` и найти `discount = 0`.
3. Поставьте точку наблюдения на `cart.total_cents` и сосчитайте остановки. Сколько их до того, как программа напечатает свою строку?
4. Запустите `RUST_BACKTRACE=1 cargo run --bin overflow` и найдите первый кадр в вашем коде.
5. Исправьте строку через `percent.min(100)`, выполните `cargo test`, а также `cargo clippy --all-targets -- -D warnings`.

## Упражнения

### Упражнение 1 — Условная точка останова (легко)

В LLDB сделайте так, чтобы программа останавливалась в `add` только для строки, у которой `price_cents` больше 1000. Какой товар остановит программу и какое у него `qty`?

<details>
<summary>Подсказка</summary>

Используйте `--condition "price_cents > 1000"` на строке `self.total_cents += line_total;` (строка 27), затем `run` и `frame variable`.

</details>

<details>
<summary>Ответ</summary>

```text
(lldb) breakpoint set --file lib.rs --line 27 --condition "price_cents > 1000"
(lldb) run
(lldb) frame variable name qty
```

Остановка происходит на `"book"`, с `qty = 1` (цена 1500). Ручка стоит 250, поэтому её условие ложно.

Почему это работает: LLDB вычисляет условие каждый раз, когда достигается строка, и молча продолжает программу, когда ответ ложен.

</details>

### Упражнение 2 — Регрессионный тест (средне)

Ошибка с 250 процентами уронила `overflow`. Напишите тест `discount_over_100_percent_stops_at_zero`, который добавляет ручки (250 × 4), применяет скидку 250 процентов и проверяет, что сумма равна `0`. Запустите его **с** исправлением `percent.min(100)` и **без** него. Что он печатает без исправления?

<details>
<summary>Подсказка</summary>

Поместите тест в `mod tests` рядом с `ten_percent_off`. Без исправления тест паникует внутри `apply_discount`, поэтому `cargo test` сообщает о нём как об упавшем.

</details>

<details>
<summary>Ответ</summary>

```rust
    #[test]
    fn discount_over_100_percent_stops_at_zero() {
        let mut cart = Cart::new();
        cart.add("pen", 250, 4);
        cart.apply_discount(250);
        assert_eq!(cart.total_cents, 0);
    }
```

```text
test tests::discount_over_100_percent_stops_at_zero ... ok
test tests::ten_percent_off ... ok
test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

Без исправления тест падает с `attempt to subtract with overflow` — той же паникой, что в разделе 6. Почему это работает: с `percent.min(100)` скидка не может быть больше всей суммы, поэтому `total - discount` может дойти до 0, но не уйти ниже.

</details>

### Упражнение 3 — Значение в момент паники (сложно)

Используя только LLDB и программу `overflow` (с ошибочной строкой), найдите значение `discount` в момент паники **без** ввода `bt`.

<details>
<summary>Подсказка</summary>

Поставьте точку останова на `rust_panic`. Кадр `apply_discount` находится на несколько кадров выше. Перейти к нему можно командой `up` (несколько раз) или `frame select N`.

</details>

<details>
<summary>Ответ</summary>

```text
(lldb) breakpoint set --name rust_panic
(lldb) run
(lldb) frame select 7
(lldb) frame variable discount
(unsigned long) discount = 2000
```

Почему это работает: паника начинается в `apply_discount`, которая вызвала механизм паники стандартной библиотеки; на 7 кадров выше находится кадр с вашими переменными. (Число 7 соответствует этой сборке; если на вашем компьютере иначе, один раз выполните `bt`, чтобы найти номер кадра, или нажимайте `up`, пока не увидите `apply_discount`.) Значение 2000 — это `1000 * (250 / 100)`, оно больше суммы 1000.

</details>

## Вопросы для самопроверки

1. В чём разница между `next` и `step`?

<details>
<summary>Ответ</summary>

`next` (шаг с обходом) выполняет всю текущую строку, включая вызываемые функции, и останавливается на следующей строке. `step` (шаг внутрь) заходит в функцию, вызываемую на строке.

</details>

2. Почему точка наблюдения не срабатывает, когда код выполняет `self.total_cents -= 0`?

<details>
<summary>Ответ</summary>

Значение не изменилось (2500 осталось 2500), а LLDB сообщает о точке наблюдения только когда сохранённое значение меняется.

</details>

3. Какой кадр в трассировке вы читаете первым?

<details>
<summary>Ответ</summary>

Первый кадр, указывающий на файл вашего собственного проекта. Кадры выше него относятся к коду паники стандартной библиотеки.

</details>

4. Почему точка останова в релизной сборке часто остаётся «ожидающей» (pending)?

<details>
<summary>Ответ</summary>

Оптимизатор может встроить, переупорядочить или удалить код, поэтому некоторых строк исходника в машинном коде больше нет. У LLDB нет адреса, на котором можно остановиться.

</details>

5. Почему нужно использовать `rust-lldb`, а не `lldb`?

<details>
<summary>Ответ</summary>

`rust-lldb` загружает вспомогательные файлы, которые читаемо печатают типы Rust (`Vec`, `String`, `&str`). Обычный `lldb` печатает сырые указатели и внутренние поля.

</details>

6. Вы поняли ошибку с помощью отладчика. Что нужно сделать до исправления?

<details>
<summary>Ответ</summary>

Написать тест, который падает из-за этой ошибки (регрессионный тест), затем исправить код и увидеть, что тест проходит. Тест не даст ошибке вернуться.

</details>

## Итоги

- Отладчик приостанавливает работающую программу, чтобы вы могли её осмотреть. Используйте отладочную сборку и `rust-lldb`.
- Основные команды: `breakpoint set`, `run`, `continue`, `next`, `step`, `finish`, `bt`, `frame variable`, `frame select`.
- **Условная точка останова** срабатывает только в интересующем вас случае. **Точка наблюдения** срабатывает при записи значения, поэтому отвечает на вопрос «кто это меняет?».
- Упавший тест можно отлаживать: путь к бинарному файлу даёт `cargo test --no-run`, а имя теста передаётся после `--`.
- При панике читайте трассировку (`RUST_BACKTRACE=1`) с первого кадра в вашем коде или поставьте точку останова на `rust_panic` и используйте `frame select`.
- Оптимизированные сборки и ошибки времени — слабые места отладчика; тогда используйте логи (следующий урок). Всё, что поняли, превращайте в тест.

## Источники и что почитать

- [LLDB tutorial](https://lldb.llvm.org/use/tutorial.html) — официальное введение в команды LLDB с короткими формами.
- [LLDB command map](https://lldb.llvm.org/use/map.html) — таблица команд других отладчиков и их аналогов в LLDB, полезная как справочник по параметрам `breakpoint`, `watchpoint` и `frame`.
- [RustRover: Debugging code](https://www.jetbrains.com/help/rust/debugging-code.html) — официальное руководство по графическому отладчику, точкам останова и пошаговому выполнению.
- [The Rust Programming Language — 9.1 Unrecoverable Errors with panic!](https://doc.rust-lang.org/book/ch09-01-unrecoverable-errors-with-panic.html) — паники и чтение трассировки стека.

---

[🏠 Начало](../START-HERE.md) · [← 09.8 Фаззинг](08-fuzzing.md) · [📋 Этап 09](README.md) · [09.10 Логирование и трассировка →](10-logging-and-tracing.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
