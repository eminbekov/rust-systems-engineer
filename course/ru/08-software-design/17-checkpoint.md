<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 08.16 Проект: рефакторинг запутанной программы](16-project-refactor-a-messy-program.md) · [📋 Этап 08](README.md) · [Этап 09 — Тестирование, отладка и бенчмарки →](../09-testing-debugging-benchmarking/README.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/08-software-design/17-checkpoint.md) <!-- nav -->

# Урок 08.17 — Контрольная точка

> **Этап 08 — Проектирование ПО: принципы, паттерны и соглашения** · Урок 17 из 17 · ⏱ около 135 минут

## Что вы узнаете

- Можете ли вы объяснить своими словами принципы проектирования этого этапа (KISS, DRY, YAGNI, связность и сопряжение, SOLID, композиция).
- Можете ли вы выбирать и писать паттерны: типы по принципу «разбирай, а не проверяй», строители, RAII-guard, конечные автоматы и typestate, стратегии и внедрение зависимостей.
- Можете ли вы распознавать «запахи», следовать соглашениям, организовывать проект, рефакторить под защитой тестов, проводить код-ревью и держать зависимости направленными внутрь.
- Какие уроки повторить перед Этапом 09.

## Перед началом

- Вы прошли [Урок 08.16 — Проект: рефакторинг запутанной программы](16-project-refactor-a-messy-program.md), и ваши ветки закоммичены.
- Отвечайте на вопросы **не** заглядывая в уроки. Сначала запишите свой ответ, затем откройте скрытый.
- Практические задания делайте в новом пакете: `cargo new --lib code/p08-checkpoint`. Каждое задание — в своём файле (`src/port.rs`, `src/transaction.rs`, `src/discount.rs`, `src/label.rs`), а модули объявите в `src/lib.rs` строками `pub mod port;` и так далее.

## Новые слова

| Слово | Значение |
|---|---|
| **контрольная точка** (checkpoint) | Урок без нового материала: самопроверка и практические задания, которые показывают, что вы помните и умеете. |
| **самопроверка** (self-test) | Ответы на вопросы по памяти до проверки ответа. Она помогает учиться лучше, чем перечитывание. |
| **KISS** (Keep It Simple, Stupid, «делай проще») | Выбирайте самое простое работающее решение. |
| **DRY** (Don't Repeat Yourself, «не повторяйся») | У каждого знания одно место. |
| **YAGNI** (You Aren't Gonna Need It, «тебе это не понадобится») | Не стройте под воображаемое будущее. |
| **SOLID** | Пять принципов проектирования, названных по начальным буквам: Single responsibility, Open/closed, Liskov substitution, Interface segregation, Dependency inversion. |
| **RAII** (Resource Acquisition Is Initialization, «получение ресурса — это инициализация») | Значение берёт ресурс при создании и возвращает его при уничтожении. |
| **API** (Application Programming Interface, программный интерфейс) | Элементы, которые крейт предлагает другому коду. |
| **TCP** (Transmission Control Protocol, протокол управления передачей) | Надёжное упорядоченное сетевое соединение, по которому идёт веб-трафик. |
| **LLDB** | Отладчик проекта компилятора LLVM (имя пришло от LLVM, что когда-то означало Low Level Virtual Machine, «низкоуровневая виртуальная машина»); Этап 09 им пользуется. |

## Вопросы для самопроверки

### Принципы

1. Что говорят KISS, DRY и YAGNI и что такое «правило трёх»?

<details>
<summary>Ответ</summary>

KISS: выбирайте самое простое работающее решение. DRY: у каждого *знания* одно место (а не «никогда не повторяй ни одного текста»). YAGNI: не стройте под воображаемое будущее. Правило трёх: потерпите первое дублирование; выделяйте абстракцию, когда то же самое появляется в третий раз, потому что к тому времени видно, что действительно меняется.

</details>

2. Что такое связность и сопряжение и в какую сторону нужно стремиться по каждому?

<details>
<summary>Ответ</summary>

Связность — насколько содержимое одной части подходит друг другу; сопряжение — насколько части зависят от внутренностей друг друга. Стремитесь к высокой связности и низкому сопряжению. Приватные поля (компилятор обеспечивает их ошибкой E0616) и узкие типы параметров вроде `&[u32]` снижают сопряжение.

</details>

3. У трейта есть метод, и одна реализация заполняет его `unimplemented!()`. Какая буква SOLID нарушена и как исправить?

<details>
<summary>Ответ</summary>

**I**, разделение интерфейсов. Разбейте трейт на более мелкие, чтобы тип реализовывал только то, что действительно умеет; это превращает панику во время выполнения в ошибку компиляции.

</details>

4. Для «видов фигур, которые пользователи могут расширять» вы берёте `enum` или трейт? А для «пяти состояний заказа»?

<details>
<summary>Ответ</summary>

Открытое множество, которое расширяют другие, — трейт (принцип открытости/закрытости). Закрытое множество, которым управляете вы, например состояния заказа, — `enum`, потому что `match` без `_` заставляет компилятор перечислить все места, которые нужно обновить при добавлении варианта.

</details>

5. Почему реализация `Deref` ради «наследования» — антипаттерн и что использовать вместо неё?

<details>
<summary>Ответ</summary>

`Deref` меняет только поиск методов: внешний тип не становится внутренним, ограничения трейтов его не видят, а все публичные методы внутреннего типа «протекают» наружу. Используйте композицию (поле) и делегирующие методы, трейт для общего поведения или обёртку.

</details>

### Типы и паттерны

6. Что значит «разбирай, а не проверяй» (parse, don't validate)? Приведите пример типа.

<details>
<summary>Ответ</summary>

Проверьте ввод один раз на границе и верните более строгий тип, несущий доказательство, чтобы остальная программа не могла получить недопустимое значение. Пример: `Port::new(u16) -> Result<Port, PortError>` с приватным полем или `Title::new(&str)`. Реализуйте `FromStr`/`TryFrom` для стандартных интерфейсов.

</details>

7. Когда строитель лучше, чем `Default` с синтаксисом обновления структуры (`..Default::default()`)?

<details>
<summary>Ответ</summary>

Когда часть полей обязательна, когда есть правила для проверки (поэтому `build()` возвращает `Result`), когда значения вычисляются или настройки повторяются. Для небольшой структуры с хорошими значениями по умолчанию и без правил достаточно `Default`.

</details>

8. Что такое RAII и в каких случаях `Drop` *не* запускается?

<details>
<summary>Ответ</summary>

Resource Acquisition Is Initialization («получение ресурса — это инициализация»): значение берёт ресурс при создании и возвращает его в `Drop`, так что очистка пишется один раз и её нельзя забыть. `Drop` не запускается при `std::process::exit`, при аварийном завершении (abort) и после `mem::forget`; поэтому это не место для логики, которая обязана выполниться.

</details>

9. Когда выбирать конечный автомат на `enum`, а когда typestate?

<details>
<summary>Ответ</summary>

`enum`, когда состояние приходит из данных времени выполнения (ввод, файл, сетевое сообщение). Typestate (по типу на состояние, переходы принимают `self`), когда порядок вызовов написан в вашем собственном коде, так что неверные порядки становятся ошибками компиляции.

</details>

10. Назовите четыре способа выразить стратегию в Rust и главный компромисс каждого.

<details>
<summary>Ответ</summary>

`enum` (закрытое множество, просто, все варианты известны); обобщённый параметр `P: Trait` (вызывающий выбирает при компиляции, быстро, но параметр типа расползается); `Box<dyn Trait>` (выбор во время выполнения, простые типы, небольшая цена на вызов); замыкание (одна функция, легче всего).

</details>

11. Что такое внедрение зависимостей и где корень композиции?

<details>
<summary>Ответ</summary>

Тип получает то, от чего зависит (в виде трейтов), через `new` или параметры, а не создаёт сам. Корень композиции — единственное место, обычно `main`, где выбираются настоящие части и соединяются. В тестах вы внедряете написанные вручную фейки.

</details>

### Качество и соглашения

12. Чем «запах» кода отличается от антипаттерна? Назовите три антипаттерна, специфичных для Rust.

<details>
<summary>Ответ</summary>

«Запах» — подсказка о возможной проблеме проектирования (назовите конкретную проблему, прежде чем что-то менять). Антипаттерн — популярный рецепт, который обычно делает хуже. Примеры в Rust: `clone` лишь бы заглушить borrow checker, `unwrap` для внешних данных, `Rc<RefCell<T>>` повсюду, `Deref` как наследование, `#![deny(warnings)]` в исходном коде.

</details>

13. Что обещают префиксы `as_`, `to_` и `into_` и как называть геттер?

<details>
<summary>Ответ</summary>

`as_`: бесплатный заимствованный вид. `to_`: новое значение из заимствованного (может стоить работы). `into_`: потребляет `self`. У геттера имя поля без `get_`: `name()`.

</details>

14. Чем `#[allow(lint)]` отличается от `#[expect(lint, reason = "...")]` и где задаётся политика lint-ов проекта?

<details>
<summary>Ответ</summary>

`expect` ещё и предупреждает, когда lint перестаёт срабатывать, так что устаревшие исключения исчезают, и несёт причину. Политика проекта задаётся в таблице `[lints]` файла `Cargo.toml` (или `[workspace.lints]` с `[lints] workspace = true` в каждом участнике). Контроль качества: `cargo fmt --check`, `cargo clippy --all-targets -- -D warnings`, `cargo test`.

</details>

15. Что делит рабочее пространство Cargo и что Cargo отказывается собирать?

<details>
<summary>Ответ</summary>

Один `Cargo.lock`, одну папку `target` и (через наследование) версии, зависимости и настройки lint-ов. Cargo отказывается собирать цикл зависимостей между крейтами, так что направление зависимостей навязывается.

</details>

### Рефакторинг, архитектура, эксплуатация

16. Что такое рефакторинг и что должно быть до начала? Что такое характеризационный тест?

<details>
<summary>Ответ</summary>

Изменение структуры кода без изменения его наблюдаемого поведения, малыми шагами, с зелёными тестами после каждого. Нужна страховочная сетка из тестов; если её нет, вы пишете характеризационные (golden) тесты, которые записывают, что код делает сейчас, включая странные результаты.

</details>

17. В каком порядке ревьюер смотрит на изменение и что значат метки `blocking:`, `suggestion:` и `nit:`?

<details>
<summary>Ответ</summary>

Проектирование, корректность, тесты, читаемость и стиль в последнюю очередь (форматирование и lint-ы проверяют инструменты). `blocking:` нужно изменить до слияния; `suggestion:` — хорошая идея на усмотрение автора; `nit:` — мелочь вкуса, которую можно игнорировать.

</details>

18. Сформулируйте правило зависимостей архитектуры портов и адаптеров и разницу между портом и адаптером.

<details>
<summary>Ответ</summary>

Зависимости в исходном коде направлены к предметной области: ядро никогда не импортирует адаптеры или технологии. Порт — трейт, принадлежащий ядру, описывающий, что ему нужно; адаптер — тип во внешнем слое, который реализует порт (ведомый) или вызывает ядро (ведущий). `main` их соединяет.

</details>

19. Назовите четыре из двенадцати факторов с их формой в Rust.

<details>
<summary>Ответ</summary>

Например: III конфигурация в окружении (читается один раз в типизированный `Config`, секреты не печатаются); XI логи в стандартный вывод; VI процессы без состояния (никаких важных данных в памяти); VII программа сама занимает порт (`TcpListener::bind`); II зависимости в `Cargo.toml` и закоммиченный `Cargo.lock` (`--locked`); IX утилизируемость (быстрый старт, обработка `SIGTERM`).

</details>

20. Почему в проекте вы сохранили ошибку в сводке и использовали `Vec`, а не `HashMap`?

<details>
<summary>Ответ</summary>

Рефакторинг не должен менять поведение, поэтому ошибку сохраняют и исправляют позже отдельным коммитом, который заодно обновляет golden-файлы. `HashMap` обходится в неопределённом порядке, из-за чего сводка отличалась бы между запусками; `Vec` пар сохраняет порядок первого появления.

</details>

## Практические задания

У каждого задания тесты — в ответе. Сначала напишите собственные тесты (Урок 05.6), затем сравните. Для каждого задания запускайте `cargo test` и `cargo clippy --all-targets -- -D warnings`.

### Задание 1 — Тип, который не может быть неверным (Уроки 08.5 и 08.11)

В `src/port.rs` напишите `Port` — номер TCP-порта от 1 до 65535. `Port::new(u16)` отвергает 0; `"8080".parse::<Port>()` работает (и игнорирует пробелы вокруг числа); текст, который не число или больше 65535, — это `PortError::NotANumber`, а 0 — `PortError::Zero`. Поле должно быть приватным, должен быть геттер `number()`, а `PortError` реализует `Display` и `std::error::Error`. Напишите тесты.

<details>
<summary>Подсказка</summary>

Реализуйте `std::str::FromStr` с `type Err = PortError;`. Сначала разберите как `u16` (число выше 65535 не разберётся), затем вызовите `Port::new`. Помните: у геттера нет префикса `get_`.

</details>

<details>
<summary>Ответ</summary>

```rust
use std::fmt;
use std::str::FromStr;

/// A TCP port that a service may listen on: 1 to 65535.
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub struct Port(u16);

/// Why a port could not be created.
#[derive(Debug, PartialEq, Eq)]
pub enum PortError {
    /// The text is not a number from 0 to 65535.
    NotANumber,
    /// Port 0 is not allowed here.
    Zero,
}

impl fmt::Display for PortError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            PortError::NotANumber => write!(f, "not a number from 1 to 65535"),
            PortError::Zero => write!(f, "port 0 is not allowed"),
        }
    }
}

impl std::error::Error for PortError {}

impl Port {
    /// Creates a port.
    ///
    /// # Errors
    ///
    /// Returns [`PortError::Zero`] for 0.
    pub fn new(number: u16) -> Result<Port, PortError> {
        if number == 0 {
            Err(PortError::Zero)
        } else {
            Ok(Port(number))
        }
    }

    /// The port number.
    #[must_use]
    pub fn number(self) -> u16 {
        self.0
    }
}

impl FromStr for Port {
    type Err = PortError;

    fn from_str(text: &str) -> Result<Port, PortError> {
        let number: u16 = text.trim().parse().map_err(|_| PortError::NotANumber)?;
        Port::new(number)
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn parses_valid_ports() {
        assert_eq!("8080".parse::<Port>().unwrap().number(), 8080);
        assert_eq!(" 443 ".parse::<Port>().unwrap().number(), 443);
    }

    #[test]
    fn rejects_zero_text_and_big_numbers() {
        assert_eq!("0".parse::<Port>(), Err(PortError::Zero));
        assert_eq!("http".parse::<Port>(), Err(PortError::NotANumber));
        assert_eq!("70000".parse::<Port>(), Err(PortError::NotANumber));
        assert_eq!(PortError::Zero.to_string(), "port 0 is not allowed");
    }
}
```

Почему это работает: единственные способы получить `Port` — `Port::new` и `FromStr`, и оба отвергают 0, поэтому каждый `Port` в программе допустим. `u16` при разборе уже отвергает числа выше 65535 и отрицательные числа.

</details>

### Задание 2 — Guard, который откатывает (Урок 08.7)

В `src/transaction.rs` напишите `Transaction<'a>` — guard, хранящий `&'a RefCell<Vec<String>>` (журнал). `Transaction::begin(log)` записывает `"begin"`. `commit(self)` записывает `"commit"`. Если guard уничтожен без `commit`, он записывает `"rollback"`. Напишите функцию `transfer(log, fail: bool) -> Result<(), String>`, которая начинает транзакцию, при `fail == true` рано возвращает `Err`, а иначе фиксирует. Проверьте оба пути.

<details>
<summary>Подсказка</summary>

Держите поле `committed: bool`. `commit` принимает `self` (поэтому его можно вызвать только раз), записывает слово, ставит флаг и позволяет guard-у уничтожиться. В `Drop::drop` записывайте `"rollback"` только если флаг ложен. Ранний `return` запускает `Drop` автоматически.

</details>

<details>
<summary>Ответ</summary>

```rust
use std::cell::RefCell;

/// A guard: if it is dropped without `commit`, it records a rollback.
pub struct Transaction<'a> {
    log: &'a RefCell<Vec<String>>,
    committed: bool,
}

impl<'a> Transaction<'a> {
    /// Starts a transaction and records `begin`.
    pub fn begin(log: &'a RefCell<Vec<String>>) -> Transaction<'a> {
        log.borrow_mut().push(String::from("begin"));
        Transaction {
            log,
            committed: false,
        }
    }

    /// Finishes the transaction successfully.
    pub fn commit(mut self) {
        self.log.borrow_mut().push(String::from("commit"));
        self.committed = true;
    }
}

impl Drop for Transaction<'_> {
    fn drop(&mut self) {
        if !self.committed {
            self.log.borrow_mut().push(String::from("rollback"));
        }
    }
}

/// Pretends to move money; fails when `fail` is true.
///
/// # Errors
///
/// Returns an error when `fail` is true.
pub fn transfer(log: &RefCell<Vec<String>>, fail: bool) -> Result<(), String> {
    let transaction = Transaction::begin(log);
    if fail {
        return Err(String::from("insufficient funds"));
    }
    transaction.commit();
    Ok(())
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn a_successful_transfer_commits() {
        let log = RefCell::new(Vec::new());
        transfer(&log, false).unwrap();
        assert_eq!(*log.borrow(), ["begin", "commit"]);
    }

    #[test]
    fn a_failed_transfer_rolls_back() {
        let log = RefCell::new(Vec::new());
        assert!(transfer(&log, true).is_err());
        assert_eq!(*log.borrow(), ["begin", "rollback"]);
    }
}
```

Почему это работает: при раннем `return Err(...)` переменная `transaction` выходит из области видимости, запускается `Drop` и пишет откат; при успехе `commit(self)` потребляет guard после установки флага, так что `Drop` молчит. Очистка пишется один раз, и её нельзя забыть.

</details>

### Задание 3 — Стратегия, внедрённая, и фейк (Уроки 08.3 и 08.9)

В `src/discount.rs` напишите трейт `DiscountPolicy` с `fn discount_cents(&self, total_cents: u64) -> u64`, две политики (`NoDiscount` и `PercentOff(u64)`, округляющую скидку вниз и считающую более 100 процентов равными 100) и `Cart<P: DiscountPolicy>` с `new(policy)`, `add(price_cents)` и `total()`. Проверьте корзину с каждой политикой и с написанным вручную фейком политики, который всегда возвращает одну и ту же скидку.

<details>
<summary>Подсказка</summary>

`total()` — сумма цен минус `self.policy.discount_cents(sum)`. Фейк живёт только в модуле `tests`: структура-кортеж `Fixed(u64)`, реализующая трейт.

</details>

<details>
<summary>Ответ</summary>

```rust
/// A rule that decides how many cents are taken off a total.
pub trait DiscountPolicy {
    /// The discount for `total_cents`, never more than the total.
    fn discount_cents(&self, total_cents: u64) -> u64;
}

/// No discount at all.
pub struct NoDiscount;

impl DiscountPolicy for NoDiscount {
    fn discount_cents(&self, _total_cents: u64) -> u64 {
        0
    }
}

/// A percentage off, rounded down.
pub struct PercentOff(pub u64);

impl DiscountPolicy for PercentOff {
    fn discount_cents(&self, total_cents: u64) -> u64 {
        total_cents * self.0.min(100) / 100
    }
}

/// A cart of prices in cents. The policy is injected.
pub struct Cart<P: DiscountPolicy> {
    prices: Vec<u64>,
    policy: P,
}

impl<P: DiscountPolicy> Cart<P> {
    /// Creates an empty cart that uses `policy`.
    pub fn new(policy: P) -> Cart<P> {
        Cart {
            prices: Vec::new(),
            policy,
        }
    }

    /// Adds an item.
    pub fn add(&mut self, price_cents: u64) {
        self.prices.push(price_cents);
    }

    /// The total after the discount.
    pub fn total(&self) -> u64 {
        let sum: u64 = self.prices.iter().sum();
        sum - self.policy.discount_cents(sum)
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    /// A fake policy for tests: always the same discount.
    struct Fixed(u64);

    impl DiscountPolicy for Fixed {
        fn discount_cents(&self, _total_cents: u64) -> u64 {
            self.0
        }
    }

    #[test]
    fn no_discount_charges_the_sum() {
        let mut cart = Cart::new(NoDiscount);
        cart.add(250);
        cart.add(400);
        assert_eq!(cart.total(), 650);
    }

    #[test]
    fn percent_off_rounds_the_discount_down() {
        let mut cart = Cart::new(PercentOff(15));
        cart.add(999);
        assert_eq!(cart.total(), 999 - 149);
    }

    #[test]
    fn a_fake_policy_makes_the_test_independent_of_the_rule() {
        let mut cart = Cart::new(Fixed(100));
        cart.add(1000);
        assert_eq!(cart.total(), 900);
    }
}
```

Почему это работает: `Cart` не знает, какое правило использует; его внедряет вызывающий код (корень композиции). Третий тест показывает выгоду: сложение и вычитание в корзине проверяются без зависимости от правила процентов.

</details>

### Задание 4 — Рефакторинг под защитой тестов (Уроки 08.10 и 08.13)

Вот «пахнущая» функция (поместите её в `src/label.rs`):

```rust
pub fn label(temp: i32, unit: &str) -> String {
    let mut text = String::new();
    if unit == "f" {
        if temp <= 32 {
            text = format!("{} F: freezing", temp);
        } else if temp >= 212 {
            text = format!("{} F: boiling", temp);
        } else {
            text = format!("{} F: liquid", temp);
        }
    } else {
        if temp <= 0 {
            text = format!("{} C: freezing", temp);
        } else if temp >= 100 {
            text = format!("{} C: boiling", temp);
        } else {
            text = format!("{} C: liquid", temp);
        }
    }
    text
}
```

Сначала напишите характеризационный тест с таблицей, покрывающей обе единицы, все три слова, границы и неизвестную единицу вроде `"k"` (что с ней делает функция?). Затем отрефакторьте: `enum Unit`, именованные константы и никакого повторяющегося текста, сохранив сигнатуру и тест неизменными. Проверьте через `cargo clippy` до и после.

<details>
<summary>Подсказка</summary>

Границы: 0 и 1, 99 и 100 для Цельсия; 32 и 33, 211 и 212 для Фаренгейта. Запустите функцию и посмотрите, что она делает с `"k"`: ветка `else` означает, что всё, что не `"f"`, — Цельсий, и тест должен это сказать. Clippy сообщает `value assigned to 'text' is never read` на оригинале.

</details>

<details>
<summary>Ответ</summary>

```rust
const FREEZING_C: i32 = 0;
const BOILING_C: i32 = 100;
const FREEZING_F: i32 = 32;
const BOILING_F: i32 = 212;

#[derive(Clone, Copy)]
enum Unit {
    Celsius,
    Fahrenheit,
}

impl Unit {
    fn parse(text: &str) -> Unit {
        if text == "f" {
            Unit::Fahrenheit
        } else {
            Unit::Celsius
        }
    }

    fn symbol(self) -> &'static str {
        match self {
            Unit::Celsius => "C",
            Unit::Fahrenheit => "F",
        }
    }

    fn limits(self) -> (i32, i32) {
        match self {
            Unit::Celsius => (FREEZING_C, BOILING_C),
            Unit::Fahrenheit => (FREEZING_F, BOILING_F),
        }
    }
}

/// Describes a temperature, e.g. `"-5 C: freezing"`.
#[must_use]
pub fn label(temp: i32, unit: &str) -> String {
    let unit = Unit::parse(unit);
    let (freezing, boiling) = unit.limits();
    let word = if temp <= freezing {
        "freezing"
    } else if temp >= boiling {
        "boiling"
    } else {
        "liquid"
    };
    format!("{temp} {}: {word}", unit.symbol())
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn pins_down_the_old_behaviour() {
        let cases = [
            // (temp, unit, expected)
            (-5, "c", "-5 C: freezing"),
            (0, "c", "0 C: freezing"),
            (1, "c", "1 C: liquid"),
            (99, "c", "99 C: liquid"),
            (100, "c", "100 C: boiling"),
            (32, "f", "32 F: freezing"),
            (33, "f", "33 F: liquid"),
            (212, "f", "212 F: boiling"),
            (50, "k", "50 C: liquid"),
        ];
        for (temp, unit, expected) in cases {
            assert_eq!(label(temp, unit), expected);
        }
    }
}
```

Почему это работает: таблица фиксирует границы и неизвестную единицу (`"k"` трактуется как Цельсий, как и раньше), и та же таблица проходит и на оригинале, и на отрефакторенной версии. Слова и пределы теперь существуют по одному разу; чтобы добавить Кельвина, вы добавляете вариант `enum`, и компилятор показывает каждый `match`, которому нужна новая ветка.

</details>

## Если что-то не получилось

| Тема | Какой урок повторить |
|---|---|
| KISS, DRY, YAGNI, правило трёх | [Урок 08.1 — Хороший дизайн: KISS, DRY и YAGNI](01-good-design-kiss-dry-yagni.md) |
| Связность, сопряжение, разделение ответственности | [Урок 08.2 — Связность, сопряжение и разделение ответственности](02-cohesion-coupling-and-separation-of-concerns.md) |
| Пять принципов SOLID | [Урок 08.3 — Принципы SOLID](03-solid-principles.md) |
| Композиция, обёртки, `Deref` — не наследование | [Урок 08.4 — Композиция вместо наследования](04-composition-over-inheritance.md) |
| «Разбирай, а не проверяй»; newtype; недопустимые состояния | [Урок 08.5 — Проектирование через типы: недопустимые состояния невозможны](05-type-driven-design.md) |
| Строители, `Default`, обязательные поля | [Урок 08.6 — Паттерн «строитель»](06-builder-pattern.md) |
| RAII, `Drop`, guard-объекты | [Урок 08.7 — RAII и guard-объекты](07-raii-and-guards.md) |
| Конечные автоматы, typestate | [Урок 08.8 — Конечные автоматы и паттерн typestate](08-state-machines-and-typestate.md) |
| Стратегия, внедрение зависимостей, фейки | [Урок 08.9 — Стратегия и внедрение зависимостей](09-strategy-and-dependency-injection.md) |
| «Запахи» и антипаттерны Rust | [Урок 08.10 — Антипаттерны и «запахи» кода](10-anti-patterns-and-code-smells.md) |
| Имена, `rustfmt`, уровни Clippy, API Guidelines | [Урок 08.11 — Соглашения: имена, Clippy и Rust API Guidelines](11-conventions-naming-clippy-and-api-guidelines.md) |
| Структура пакета, рабочие пространства, циклы зависимостей | [Урок 08.12 — Структура проекта и рабочие пространства](12-project-layout-and-workspaces.md) |
| Характеризационные тесты, шаги рефакторинга, код-ревью | [Урок 08.13 — Рефакторинг и код-ревью](13-refactoring-and-code-review.md) |
| Слои, порты и адаптеры, правило зависимостей | [Урок 08.14 — Слои и гексагональная архитектура](14-layers-and-hexagonal-architecture.md) |
| Конфигурация, логи, процессы без состояния, `SIGTERM` | [Урок 08.15 — Приложение двенадцати факторов](15-the-twelve-factor-app.md) |
| Golden-тесты, рефакторинг с совместимостью по ошибкам | [Урок 08.16 — Проект: рефакторинг запутанной программы](16-project-refactor-a-messy-program.md) |
| Написание тестов, `Result` и ошибки (за многими заданиями) | [Урок 05.6 — Тестирование подробно](../05-rust-intermediate/06-testing-in-depth.md) и [Урок 04.10 — Обработка ошибок](../04-structs-enums-collections-errors/10-error-handling.md) |
| Трейты и трейт-объекты | [Урок 05.2 — Трейты](../05-rust-intermediate/02-traits.md) и [Урок 06.1 — Трейт-объекты и динамическая диспетчеризация](../06-rust-advanced/01-trait-objects-and-dynamic-dispatch.md) |

Когда повторяете урок, переделайте его упражнения в отдельной ветке и сначала пишите тесты.

## Что дальше

Вы закончили Этап 08. Теперь вы можете увидеть, почему код болит, назвать нарушенный принцип, выбрать подходящий паттерн и безопасно менять программу под защитой тестов. Прежде чем идти дальше:

1. Закоммитьте пакет контрольной точки (`git add code/p08-checkpoint && git commit -m "feat(p08): checkpoint tasks"`), убедитесь, что `git status` чист, а ветка проекта из Урока 08.16 влита в `main`. Удалите её, когда она больше не нужна (`git branch -d project/p08-refactor`).
2. В учебных заметках запишите три «запаха», которые вы узнаёте в своём прежнем коде, и урок, исправляющий каждый.
3. Сделайте перерыв. День отдыха помогает памяти больше, чем ещё час чтения.

В **Этапе 09 — Тестирование, отладка и бенчмаркинг** вы будете опираться на этот этап: пирамида тестов, сквозные тесты программ командной строки (golden-тесты проекта, сделанные как следует), тестовые двойники, тесты на основе свойств, снапшот-тесты, разработка через тестирование, покрытие и мутационное тестирование, фаззинг, отладка в LLDB и RustRover, логирование и трассировка, бенчмаркинг и непрерывная интеграция. Начните с [Урока 09.1 — Пирамида тестов и стратегия тестирования](../09-testing-debugging-benchmarking/01-the-test-pyramid.md).

## Источники и что почитать

- [Rust API Guidelines](https://rust-lang.github.io/api-guidelines/) — чек-лист для публичных API (Урок 08.11).
- [Rust Design Patterns](https://rust-unofficial.github.io/patterns/) — идиомы, паттерны и антипаттерны в Rust, бесплатно онлайн.
- [The Rust Programming Language — Chapter 18: Object-Oriented Programming Features](https://doc.rust-lang.org/book/ch18-00-oop.html) — трейт-объекты и паттерн «состояние».
- [The Cargo Book — Workspaces](https://doc.rust-lang.org/cargo/reference/workspaces.html) — общие зависимости, версии и lint-ы.
- [The Twelve-Factor App](https://12factor.net/) — двенадцать правил в оригинальной формулировке.
- [Refactoring — каталог Мартина Фаулера](https://refactoring.com/catalog/) — названия и шаги классических рефакторингов.

---

[🏠 Начало](../START-HERE.md) · [← 08.16 Проект: рефакторинг запутанной программы](16-project-refactor-a-messy-program.md) · [📋 Этап 08](README.md) · [Этап 09 — Тестирование, отладка и бенчмарки →](../09-testing-debugging-benchmarking/README.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
