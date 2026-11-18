<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 08.6 Паттерн «строитель»](06-builder-pattern.md) · [📋 Этап 08](README.md) · [08.8 Конечные автоматы и паттерн typestate →](08-state-machines-and-typestate.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/08-software-design/07-raii-and-guards.md) <!-- nav -->

# Урок 08.7 — RAII и guard-объекты

> **Этап 08 — Проектирование ПО: принципы, паттерны и соглашения** · Урок 7 из 17 · ⏱ около 135 минут

## Что вы узнаете

- Что значит RAII и почему он делает очистку невозможной для забывания.
- Как написать собственный **guard-объект** (guard): транзакцию с откатом и соединение, которое само возвращается в пул.
- Точные правила порядка, в котором уничтожаются значения, и ловушку `let _ =`.
- Что происходит, когда guard держат слишком долго, и как сократить его жизнь.
- Почему `Drop` не может сообщать об ошибках и паттерн «явный `finish`, а `Drop` как страховочная сетка».

## Перед началом

- Вы прошли [Урок 08.6 — Паттерн «строитель»](06-builder-pattern.md).
- Вы знаете `Drop` и `drop(x)` из [Урока 05.9 — Умные указатели](../05-rust-intermediate/09-smart-pointers.md), `Mutex` из [Урока 05.10](../05-rust-intermediate/10-threads-channels-and-shared-state.md), `RefCell` из [Урока 06.4](../06-rust-advanced/04-interior-mutability.md) и времена жизни в структурах из [Урока 05.3](../05-rust-intermediate/03-lifetimes.md).

## Новые слова

| Слово | Значение |
|---|---|
| **ресурс** (resource) | Что-то, что берётся и должно быть возвращено: память, открытый файл, блокировка, соединение с базой данных, место в пуле. |
| **RAII** (Resource Acquisition Is Initialization, «получение ресурса — это инициализация») | Устройство, в котором значение *берёт* ресурс при создании и *отдаёт* его при уничтожении. Поскольку Rust уничтожает значения автоматически, очистку нельзя забыть. |
| **guard-объект** (guard, страж) | Значение, жизнь которого означает «я что-то удерживаю». Пока оно существует, ресурс удерживается; когда оно уничтожено, ресурс освобождается. Классический пример — `MutexGuard`. |
| **scope guard** (страж области видимости) | Guard, написанный так, чтобы выполнить код в конце области видимости, как бы она ни закончилась (обычный конец, `return`, `?`, паника). |
| **откат** (rollback) | Отмена изменений незавершённой операции, чтобы данные вернулись в прежнее состояние. |
| **фиксация** (commit) | Объявление изменений операции окончательными. |
| **временное значение** (temporary) | Значение, созданное в середине выражения и не сохранённое в переменной. Уничтожается в конце инструкции. |
| **раскрутка** (unwinding) | То, что делает паника по умолчанию: она идёт вверх по стеку, уничтожая по дороге все живые значения. |
| **утечка** (leak) | Значение, которое никогда не уничтожается, поэтому его очистка не выполняется (`std::mem::forget` делает это намеренно). В Rust это безопасно, но редко нужно. |

## 1. RAII: очистка, которую нельзя забыть

Каждый ресурс нужно вернуть: закрыть файл, освободить блокировку, вернуть соединение. Если вам нужно помнить, что `close()` надо написать на каждом пути выхода из функции, включая ранние возвраты и ошибки, рано или поздно вы забудете один.

RAII устраняет проблему. Вы помещаете код освобождения в `Drop`. Значение уничтожается в конце своей области видимости, как бы она ни закончилась, и вызов вставляет компилятор (Урок 05.9). На этом построена стандартная библиотека:

| Тип | Ресурс | Освобождается, когда значение уничтожено |
|---|---|---|
| `File` | Открытый файл | Файл закрывается |
| `MutexGuard` (из `mutex.lock()`) | Право пользоваться данными внутри `Mutex` | Блокировка освобождается |
| `Ref` / `RefMut` (из `RefCell::borrow` / `borrow_mut`) | Разделяемое или исключительное заимствование | Счётчик заимствований уменьшается |
| `Box`, `Vec`, `String` | Память в куче | Память освобождается |

**Guard** — значение такого рода, цель которого — удерживать что-то на время. Для его написания не нужны ни слово `unsafe`, ни особые возможности: хватает структуры и `impl Drop`.

## 2. Scope guard: транзакции с откатом

Представьте банковский перевод из нескольких шагов. Если третий шаг не удался, первые два нужно отменить. Без RAII код отмены приходится писать на каждом пути ошибки. С guard отмена пишется один раз:

```rust
struct Account {
    balance: i64,
}

struct Transaction<'a> {
    account: &'a mut Account,
    original: i64,
    committed: bool,
}

impl Account {
    fn begin(&mut self) -> Transaction<'_> {
        let original = self.balance;
        Transaction {
            account: self,
            original,
            committed: false,
        }
    }
}

impl Transaction<'_> {
    fn withdraw(&mut self, amount: i64) -> Result<(), String> {
        if self.account.balance < amount {
            return Err(format!(
                "cannot withdraw {amount}: balance is {}",
                self.account.balance
            ));
        }
        self.account.balance -= amount;
        Ok(())
    }

    fn commit(mut self) {
        self.committed = true; // `self` is dropped at the end of this method
    }
}

impl Drop for Transaction<'_> {
    fn drop(&mut self) {
        if !self.committed {
            self.account.balance = self.original;
            println!("rolled back to {}", self.original);
        }
    }
}

fn pay(account: &mut Account, amounts: &[i64]) -> Result<(), String> {
    let mut tx = account.begin();
    for amount in amounts {
        tx.withdraw(*amount)?; // an early return drops `tx` without commit
    }
    tx.commit();
    Ok(())
}

fn main() {
    let mut account = Account { balance: 100 };
    println!("{:?}", pay(&mut account, &[30, 20]));
    println!("balance: {}", account.balance);
    println!("{:?}", pay(&mut account, &[30, 40]));
    println!("balance: {}", account.balance);
}
```

**Что происходит:**

1. `account.begin()` создаёт `Transaction`, который изменяемо заимствует счёт (`&'a mut Account`) и запоминает исходный баланс.
2. `withdraw` меняет баланс или возвращает ошибку, если денег не хватает.
3. `commit(mut self)` помечает транзакцию окончательной. Он принимает `self` по значению, поэтому транзакция уничтожается в конце `commit`, а `Drop` видит `committed == true` и ничего не делает.
4. `impl Drop for Transaction`: если транзакция **не** зафиксирована, он возвращает исходный баланс и печатает сообщение.
5. В `pay` оператор `?` делает ранний возврат, когда снятие не удалось. Ранний возврат уничтожает `tx`, и guard откатывает изменения. Кода отмены в `pay` нет вообще.
6. `main` выполняет два платежа. Первый (30 + 20) успешен. Второй (30 + 40) не удаётся на 40, потому что после первых 30 остаётся только 20, и баланс возвращается к 50.

```text
Ok(())
balance: 50
rolled back to 50
Err("cannot withdraw 40: balance is 20")
balance: 50
```

Обратите внимание на порядок: `rolled back to 50` печатается, когда `pay` возвращается, раньше, чем `main` печатает `Err`. И в `account.balance` никогда не бывает полузавершённого состояния, потому что guard покрывает и панику посреди `pay`.

> 🧪 **Тестирование:** guard легко тестировать по его эффекту: выполните путь с ошибкой и проверьте данные. Вот два теста для `pay`, помещённые в тот же файл:

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn successful_payment_is_committed() {
        let mut account = Account { balance: 100 };
        assert!(pay(&mut account, &[30, 20]).is_ok());
        assert_eq!(account.balance, 50);
    }

    #[test]
    fn failed_payment_is_rolled_back_completely() {
        let mut account = Account { balance: 100 };
        assert!(pay(&mut account, &[60, 60]).is_err());
        assert_eq!(account.balance, 100);
    }
}
```

Они проходят:

```text
running 2 tests
test tests::failed_payment_is_rolled_back_completely ... ok
test tests::successful_payment_is_committed ... ok

test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

## 3. Guard, который возвращает: пул

Открытие соединения с базой данных медленно, поэтому серверы держат **пул** готовых соединений. Клиент берёт одно, использует и должен вернуть. Guard делает возврат автоматическим:

```rust
use std::cell::RefCell;

struct Pool {
    free: RefCell<Vec<u32>>,
}

struct Conn<'a> {
    pool: &'a Pool,
    id: u32,
}

impl Pool {
    fn new(size: u32) -> Pool {
        Pool {
            free: RefCell::new((1..=size).collect()),
        }
    }

    fn acquire(&self) -> Option<Conn<'_>> {
        let id = self.free.borrow_mut().pop()?;
        println!("take {id}");
        Some(Conn { pool: self, id })
    }
}

impl Drop for Conn<'_> {
    fn drop(&mut self) {
        println!("give back {}", self.id);
        self.pool.free.borrow_mut().push(self.id);
    }
}

fn main() {
    let pool = Pool::new(2);
    let a = pool.acquire().unwrap();
    let b = pool.acquire().unwrap();
    println!("third: {}", pool.acquire().is_none());
    drop(a);
    let c = pool.acquire().unwrap();
    println!("c has id {}, b has id {}", c.id, b.id);
}
```

**Что происходит:**

1. `Pool` хранит свободные идентификаторы в `RefCell<Vec<u32>>` (внутренняя изменяемость, Урок 06.4), поэтому `acquire` может принимать `&self`, и много guard-объектов могут делить один пул.
2. `acquire` достаёт идентификатор и возвращает guard `Conn`, который хранит `&Pool`. Если идентификаторов не осталось, `pop()?` рано возвращает `None`.
3. `impl Drop for Conn` кладёт идентификатор обратно в пул.
4. `main` берёт оба идентификатора (2, затем 1), видит, что третий запрос не удаётся, явно уничтожает `a` (это возвращает идентификатор 2), и следующий `acquire` получает тот же идентификатор обратно.
5. В конце `main` оставшиеся guard-объекты уничтожаются в обратном порядке создания (`c`, затем `b`).

```text
take 2
take 1
third: true
give back 2
take 2
c has id 2, b has id 1
give back 2
give back 1
```

Время жизни `'_` в `Conn<'_>` связывает guard с пулом: компилятор не позволит guard пережить пул, которому он принадлежит. В настоящей программе пул, общий для потоков, использовал бы `Mutex` вместо `RefCell`; идея guard та же.

## 4. Правила уничтожения и ловушка `let _`

Главное правило вы узнали в Уроке 05.9: значения уничтожаются, когда выходят из области видимости, в порядке, обратном созданию. Вот полный список:

- **Локальные переменные** уничтожаются в порядке, обратном объявлению.
- **Поля структуры** уничтожаются в том порядке, в котором они записаны в структуре, *после* того как выполнится собственный `drop` структуры.
- **Временные значения** уничтожаются в конце инструкции, которая их создала.
- Значение, которое было **перемещено**, уничтожает его новый владелец, а не старый. `drop(x)` перемещает `x` в функцию, которая сразу заканчивается.
- Параметры функции уничтожаются, когда функция возвращается.

Теперь ловушка, в которую попадают почти все — один раз:

```rust
struct Guard(&'static str);

impl Drop for Guard {
    fn drop(&mut self) {
        println!("release {}", self.0);
    }
}

fn main() {
    let _ = Guard("underscore"); // not a variable: dropped at once
    println!("after `let _ =`");
    let _named = Guard("named"); // a variable: lives until the end of the block
    println!("after `let _named =`");
}
```

**Что происходит:**

1. `let _ = Guard("underscore");` **не** создаёт переменную. Подчёркивание означает «игнорировать это значение», поэтому guard — временное значение и уничтожается в конце инструкции.
2. `let _named = Guard("named");` создаёт переменную (ведущее подчёркивание лишь заглушает предупреждение «неиспользуемая переменная»), поэтому guard живёт до конца `main`.

```text
release underscore
after `let _ =`
after `let _named =`
release named
```

Если вы напишете `let _ = mutex.lock().unwrap();`, блокировка освободится немедленно, и вы совсем не защищены. Всегда давайте guard имя: `let _guard = ...`.

> ⚠️ **Частая ошибка:** `let _ = guard;` и `let _guard = guard;` выглядят почти одинаково, а ведут себя совершенно по-разному. Когда важна *жизнь* значения, дайте ему имя.

Помните также, что Rust не гарантирует выполнение `Drop` во всех случаях. Он не выполняется, если процесс убит, если вызван `std::process::exit`, если программа аварийно завершается (abort) или если значение утекло через `std::mem::forget`. Во время паники значения *уничтожаются* (раскрутка), если только программа не собрана так, чтобы завершаться при панике.

> 🔒 **Безопасность:** не полагайтесь на `Drop` в том, что должно произойти даже при убийстве программы, например в сохранении единственной копии важных данных. Используйте его, чтобы освобождать ресурсы и сохранять инварианты, а не как единственную защиту.

## 5. Guard, который держат слишком долго

Guard удерживает ресурс всю свою жизнь, а переменная живёт до конца своего блока. Это часто дольше, чем вы задумывали:

```rust
use std::cell::RefCell;

fn main() {
    let cell = RefCell::new(vec![1, 2, 3]);
    let first = cell.borrow_mut(); // a guard: lives until the end of main
    println!("first guard holds {} items", first.len());
    let second = cell.borrow_mut(); // a second guard at the same time
    println!("never printed: {}", second.len());
}
```

```text
first guard holds 3 items

thread 'main' (5600741) panicked at src/main.rs:7:23:
RefCell already borrowed
```

**Что происходит:**

1. `cell.borrow_mut()` возвращает guard `RefMut`. `RefCell` запоминает «кто-то заимствует это исключительно».
2. Первый guard ещё жив (он живёт до конца `main`), поэтому второй `borrow_mut()` находит ячейку занятой и паникует. С `Mutex` запрос второй блокировки в том же потоке может привести к **взаимной блокировке** (deadlock, ожидание навсегда), что хуже паники.

Лекарство — закончить жизнь guard как можно раньше:

- Поместите работу под защитой во внутренний блок `{ ... }`; guard уничтожается на закрывающей скобке.
- Или вызовите `drop(guard)` явно.
- Или не давайте guard имя, когда хватает одной инструкции: `cell.borrow_mut().push(4);` (guard — временное значение и уничтожается на точке с запятой).

В Упражнении 2 вас просят исправить такую программу.

> ⚡ **Производительность:** блокировка, которую держат во время медленной работы (чтение файла, ожидание сети), заставляет ждать все остальные потоки. Возьмите блокировку, скопируйте или измените нужное и уничтожьте guard до медленной части.

## 6. `Drop` не может завершиться ошибкой

Метод `drop` ничего не возвращает. Он не может сказать вызывающему «закрыть не удалось», и он не должен паниковать (паника во время раскрутки другой паники аварийно завершает всю программу). Но настоящая очистка может не удаваться: сброс файла на диск может вернуть ошибку.

Стандартный паттерн состоит из двух частей:

1. **Явный метод**, который потребляет значение и возвращает `Result`: `finish()`, `close()`, `commit()`. Вызывающий может обработать ошибку.
2. Реализация `Drop` как **страховочная сетка**: она выполняет очистку по мере возможности (или лишь предупреждает), потому что вызывающий мог забыть или мог произойти ранний возврат.

```rust
struct Report {
    lines: Vec<String>,
    finished: bool,
}

impl Report {
    fn new() -> Report {
        Report {
            lines: Vec::new(),
            finished: false,
        }
    }

    fn add(&mut self, line: &str) {
        self.lines.push(line.to_string());
    }

    // The explicit way: can report a problem to the caller.
    fn finish(mut self) -> Result<usize, String> {
        if self.lines.is_empty() {
            return Err("empty report".to_string()); // `self` is dropped here, not finished
        }
        self.finished = true;
        println!("saved {} lines", self.lines.len());
        Ok(self.lines.len())
    }
}

// The safety net: cannot return an error, so it only warns.
impl Drop for Report {
    fn drop(&mut self) {
        if !self.finished {
            println!("warning: report dropped without finish()");
        }
    }
}

fn main() {
    let mut good = Report::new();
    good.add("hello");
    println!("{:?}", good.finish());

    let mut forgotten = Report::new();
    forgotten.add("lost");
    println!("end of main");
}
```

**Что происходит:**

1. `finish(mut self)` принимает отчёт по значению, делает настоящую работу, может вернуть `Err` и при успехе устанавливает `finished` в `true`.
2. `Drop` проверяет флаг. Если `finish` не вызывали (или он не удался), он предупреждает.
3. `good.finish()` печатает `saved 1 lines`; предупреждения для него нет, потому что `finished` равен `true`, когда значение уничтожается в конце `finish`.
4. `forgotten` никогда не завершают, поэтому в конце `main` его `drop` печатает предупреждение.

```text
saved 1 lines
Ok(1)
end of main
warning: report dropped without finish()
```

Стандартная библиотека использует именно такой дизайн: `BufWriter` сбрасывает буфер в `drop` по мере возможности и игнорирует ошибки, поэтому документация советует вызывать `flush()` самому, когда ошибки вам важны.

**Когда НЕ использовать RAII и `Drop`:**

- Если очистка необязательна или может не удаваться так, что вызывающий должен это обработать, дайте ей явный метод (выше).
- Не прячьте в `drop` тяжёлую, медленную или неожиданную работу. Он выполняется молча, в конце области видимости.
- Не используйте `Drop` для обычной логики программы («когда это значение исчезнет, отправить письмо»). Место, где выполняется код, становится невидимым.
- Не пишите guard для ресурса, у которого нет периода «удержания». Обычная функция проще.

## Частые ошибки

**Ошибка 1: `let _ = guard`.** См. раздел 4. Guard-объект уничтожается сразу.

**Ошибка 2: `Drop` с паникой или `unwrap` внутри.** Падающий `drop` во время раскрутки аварийно завершает программу. Лучше запишите в журнал и продолжайте.

**Ошибка 3: guard в долгоживущей переменной.** Guard блокировки, сохранённый в поле структуры или в переменной в начале `main`, удерживается всю программу.

**Ошибка 4: вызов `.drop()`.** Так нельзя: `a.drop()` — ошибка E0040 (явные вызовы деструктора запрещены). Используйте `drop(a)` (Урок 05.9).

## Попробуйте сами

1. Создайте пакет и скопируйте пример с транзакцией из раздела 2 (`Account`, `Transaction` и `main`) в `src/main.rs`:

```bash
cd ~/projects/rust
cargo new code/p08-raii
cd code/p08-raii
```

2. Запустите его через `cargo run`. Вы увидите пять строк из раздела 2.

3. Добавьте два теста из блока 🧪 и запустите `cargo test`. Они проходят.

4. Сломайте guard намеренно: в `Drop` удалите проверку `if !self.committed`, чтобы guard *всегда* откатывал. Снова запустите `cargo test`. Какой тест падает и почему? Верните проверку на место.

5. Сохраните работу:

```bash
cargo fmt
cargo clippy
git add .
git commit -m "feat(p08): transaction guard"
```

**Что происходит:** на шаге 4 падает тест `successful_payment_is_committed`, потому что guard откатывает зафиксированную транзакцию. Это показывает, что тесты описывают контракт guard в обе стороны.

## Упражнения

### Упражнение 1 — Предскажите уничтожения (легко)

Прочитайте программу ниже и запишите вывод **до** запуска. Затем запустите и сравните.

```rust
struct Noisy(&'static str);

impl Drop for Noisy {
    fn drop(&mut self) {
        println!("drop {}", self.0);
    }
}

fn make() -> Noisy {
    let _temp = Noisy("temp");
    Noisy("result")
}

fn main() {
    let a = Noisy("a");
    let _b = make();
    {
        let _c = Noisy("c");
        println!("inner");
    }
    let _ = Noisy("d");
    println!("end");
    drop(a);
}
```

<details>
<summary>Подсказка</summary>

Какие значения — временные или локальные в `make`? Что делает `let _ =`? Какие переменные ещё живы в конце `main`?

</details>

<details>
<summary>Ответ</summary>

```text
drop temp
inner
drop c
drop d
end
drop a
drop result
```

Почему это работает: `_temp` уничтожается, когда `make` возвращается (`drop temp`). Внутренний блок заканчивается до `drop c`. `let _ = Noisy("d")` уничтожается сразу, поэтому `drop d` идёт до `end`. `drop(a)` — явный. `_b`, последняя живая переменная, уничтожается в конце `main` (`drop result`).

</details>

### Упражнение 2 — Сократите жизнь guard (средне)

Эта программа паникует. Выясните почему и исправьте её **не меняя** `total`.

```rust
use std::cell::RefCell;

fn total(cell: &RefCell<Vec<i32>>) -> i32 {
    cell.borrow().iter().sum()
}

fn main() {
    let cell = RefCell::new(vec![1, 2]);
    let mut items = cell.borrow_mut();
    items.push(3);
    println!("total: {}", total(&cell));
}
```

<details>
<summary>Подсказка</summary>

`items` — guard `RefMut`, который живёт до конца `main`. Где можно закончить его жизнь раньше?

</details>

<details>
<summary>Ответ</summary>

```rust
use std::cell::RefCell;

fn total(cell: &RefCell<Vec<i32>>) -> i32 {
    cell.borrow().iter().sum()
}

fn main() {
    let cell = RefCell::new(vec![1, 2]);
    {
        let mut items = cell.borrow_mut();
        items.push(3);
    } // the guard is dropped here
    println!("total: {}", total(&cell));
}
```

Почему это работает: внутренний блок уничтожает `items` на закрывающей скобке, поэтому `total` снова может заимствовать ячейку. `drop(items);` перед `println!` тоже сработал бы.

</details>

### Упражнение 3 — Посчитайте активные билеты (сложно)

Напишите `Counter` с методом `enter()`, возвращающим guard `Ticket`. `counter.active()` должен возвращать, сколько билетов живо прямо сейчас. **Напишите тест**: 0 в начале, 2 после двух билетов, 1 после уничтожения одного.

<details>
<summary>Подсказка</summary>

Используйте `Cell<u32>` для счётчика (он допускает изменения через `&self`). `enter` прибавляет один, а `Drop` у `Ticket` вычитает один.

</details>

<details>
<summary>Ответ</summary>

```rust
use std::cell::Cell;

struct Counter {
    active: Cell<u32>,
}

struct Ticket<'a> {
    counter: &'a Counter,
}

impl Counter {
    fn new() -> Counter {
        Counter {
            active: Cell::new(0),
        }
    }

    fn enter(&self) -> Ticket<'_> {
        self.active.set(self.active.get() + 1);
        Ticket { counter: self }
    }

    fn active(&self) -> u32 {
        self.active.get()
    }
}

impl Drop for Ticket<'_> {
    fn drop(&mut self) {
        self.counter.active.set(self.counter.active.get() - 1);
    }
}

fn main() {
    let counter = Counter::new();
    let first = counter.enter();
    let _second = counter.enter();
    println!("active: {}", counter.active());
    drop(first);
    println!("active: {}", counter.active());
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn active_follows_the_lifetime_of_the_tickets() {
        let counter = Counter::new();
        assert_eq!(counter.active(), 0);
        let first = counter.enter();
        let _second = counter.enter();
        assert_eq!(counter.active(), 2);
        drop(first);
        assert_eq!(counter.active(), 1);
    }
}
```

Почему это работает: число не хранится отдельно вручную; оно следует за жизнями guard-объектов. Оно не может разойтись с действительностью, даже когда функция рано возвращается. Так часто строят измеритель «число активных запросов».

</details>

## Вопросы для самопроверки

1. Что означает RAII и что он гарантирует в Rust?

<details>
<summary>Ответ</summary>

Resource Acquisition Is Initialization. Ресурс освобождается в `drop`, когда значение выходит из области видимости, как бы та ни закончилась, поэтому очистку нельзя забыть.

</details>

2. Чем отличаются `let _ = guard;` и `let _guard = guard;`?

<details>
<summary>Ответ</summary>

`let _ =` уничтожает значение сразу; `let _guard =` оставляет его живым до конца области видимости.

</details>

3. В каком порядке уничтожаются локальные переменные?

<details>
<summary>Ответ</summary>

В порядке, обратном объявлению (последняя созданная уничтожается первой).

</details>

4. Почему `Drop` не должен быть единственным местом для очистки, которая может не удаться?

<details>
<summary>Ответ</summary>

`drop` не может вернуть ошибку и не должен паниковать. Явный метод, возвращающий `Result`, позволяет вызывающему обработать сбои; `Drop` — страховочная сетка.

</details>

5. Назовите две ситуации, в которых `Drop` не выполняется.

<details>
<summary>Ответ</summary>

`std::process::exit`, аварийное завершение (abort), убитый процесс или `std::mem::forget` для этого значения.

</details>

## Итоги

- **RAII**: значение берёт ресурс при создании и отдаёт его в `Drop`, поэтому очистка пишется один раз и не может быть забыта.
- **Guard** — структура с `impl Drop`, которая представляет «я что-то удерживаю»: блокировку, заимствование, соединение, транзакцию.
- Guard с откатом отменяет незавершённую работу на каждом пути выхода, включая ранний `return`, `?` и паники; `commit(self)` его обезвреживает.
- Переменные уничтожаются в обратном порядке; поля — в порядке объявления; временные значения — в конце инструкции. `let _ = x` уничтожает сразу, `let _x = x` — нет.
- Держите guard-объекты недолговечными: внутренние блоки, `drop(guard)` или временные значения. Не удерживайте блокировки во время медленной работы.
- `Drop` не может завершиться ошибкой и не вызывается вручную. Сочетайте явный `finish`/`close`, возвращающий `Result`, со страховочной сеткой `Drop`.
- `Drop` не выполняется при `process::exit`, abort или `mem::forget`, поэтому это не место для логики «обязательно должно произойти».

## Источники и что почитать

- [Rust Design Patterns — RAII with guards](https://rust-unofficial.github.io/patterns/patterns/behavioural/RAII.html) — паттерн с примером на `Mutex`.
- [std::ops::Drop](https://doc.rust-lang.org/std/ops/trait.Drop.html) — трейт с правилами порядка уничтожения и замечаниями о паниках.
- [The Rust Reference — Destructors](https://doc.rust-lang.org/reference/destructors.html) — точные правила того, когда и в каком порядке уничтожаются значения.
- [std::sync::MutexGuard](https://doc.rust-lang.org/std/sync/struct.MutexGuard.html) — самый используемый guard в стандартной библиотеке.
- [std::mem::forget](https://doc.rust-lang.org/std/mem/fn.forget.html) — почему утечки разрешены и что это значит для guard.

---

[🏠 Начало](../START-HERE.md) · [← 08.6 Паттерн «строитель»](06-builder-pattern.md) · [📋 Этап 08](README.md) · [08.8 Конечные автоматы и паттерн typestate →](08-state-machines-and-typestate.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
