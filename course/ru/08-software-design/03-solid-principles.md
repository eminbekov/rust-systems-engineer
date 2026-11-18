<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 08.2 Связность, сопряжение и разделение ответственности](02-cohesion-coupling-and-separation-of-concerns.md) · [📋 Этап 08](README.md) · [08.4 Композиция вместо наследования →](04-composition-over-inheritance.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/08-software-design/03-solid-principles.md) <!-- nav -->

# Урок 08.3 — Принципы SOLID

> **Этап 08 — Проектирование ПО: принципы, паттерны и соглашения** · Урок 3 из 17 · ⏱ около 2,5 часа (2 занятия)

## Что вы узнаете

- Что говорят пять принципов SOLID простыми словами.
- Как каждый принцип выглядит в Rust: структуры, перечисления, трейты и обобщения.
- Почему `enum` и `match` в Rust иногда лучше идеи «открытости для расширения» и когда лучше трейт.
- Как защитить контракт трейта тестом.
- Когда SOLID превращается в избыточное проектирование.

## Перед началом

- Вы прошли [Урок 08.2 — Связность, сопряжение и разделение ответственности](02-cohesion-coupling-and-separation-of-concerns.md).
- Вы знаете трейты ([Урок 05.2](../05-rust-intermediate/02-traits.md)) и трейт-объекты с `dyn` ([Урок 06.1](../06-rust-advanced/01-trait-objects-and-dynamic-dispatch.md)).
- Вы знаете перечисления и `match` ([Урок 04.3](../04-structs-enums-collections-errors/03-enums.md)).

## Новые слова

| Слово | Значение |
|---|---|
| **SOLID** | Запоминающееся название пяти принципов проектирования: **S**ingle responsibility (единственная ответственность), **O**pen/closed (открытость/закрытость), **L**iskov substitution (подстановка Лисков), **I**nterface segregation (разделение интерфейсов), **D**ependency inversion (инверсия зависимостей). |
| **ответственность** (responsibility) | Причина, по которой кусок кода может измениться. «Считает итог» и «формирует HTML» — две ответственности. |
| **расширение** (extension) | Добавление в программу нового поведения, например нового вида фигуры. |
| **контракт** (contract) | То, что трейт обещает своим пользователям: что возвращают методы, когда они могут завершиться ошибкой, чего они никогда не делают. Компилятор проверяет только типы; остальное записано в документации и обеспечивается программистом. |
| **подстановка** (substitution) | Замена одной реализации трейта другой так, что код, использующий трейт, этого не замечает. |
| **абстракция** (abstraction), снова | Здесь: трейт, который код верхнего уровня использует вместо конкретного типа. |
| **код верхнего и нижнего уровня** (high-level / low-level code) | Код верхнего уровня решает, *что* делает программа (бизнес-правила). Код нижнего уровня занимается техническими деталями (файлы, экран, сеть). |

## 1. Что такое SOLID и чем он не является

SOLID — набор из пяти рекомендаций, который собрал инженер Роберт Мартин (Robert C. Martin) в начале 2000-х. Сначала они были записаны для языков, построенных вокруг классов и наследования, поэтому некоторые слова в Rust звучат непривычно. Но сами *идеи* старые и полезные: каждая описывает способ, которым программа становится мучительной для изменения. В этом уроке мы переведём каждую идею на инструменты Rust.

| Буква | Принцип | Кратко | Главный инструмент Rust |
|---|---|---|---|
| **S** | Единственная ответственность | У части одна причина меняться. | структуры, модули |
| **O** | Открытость/закрытость | Добавляйте поведение, не правя старый код. | трейты (или перечисления) |
| **L** | Подстановка Лисков | Каждая реализация соблюдает контракт. | контракты трейтов, тесты |
| **I** | Разделение интерфейсов | Маленькие трейты; не заставляйте реализовывать ненужные методы. | несколько небольших трейтов |
| **D** | Инверсия зависимостей | Зависьте от трейта, а не от конкретной детали. | обобщения, `impl Trait`, `dyn Trait` |

Помните Урок 08.1: это практические правила, а не законы. Каждый раздел заканчивается пунктом «Когда НЕ применять».

## 2. S — единственная ответственность

*У части программы должна быть одна причина меняться.* В Уроке 08.2 вы встречали это как связность. Вот `Invoice` (счёт), который знает свои строки, складывает их и ещё и записывает себя как HTML (HyperText Markup Language, язык веб-страниц):

```rust
struct Invoice {
    lines: Vec<(String, u64)>,
}

impl Invoice {
    fn total_cents(&self) -> u64 {
        self.lines.iter().map(|(_, cents)| cents).sum()
    }

    fn to_html(&self) -> String {
        format!("<p>Total: {} cents</p>", self.total_cents())
    }
}

fn main() {
    let invoice = Invoice {
        lines: vec![("tea".to_string(), 250), ("cake".to_string(), 400)],
    };
    println!("{}", invoice.to_html());
}
```

Она печатает `<p>Total: 650 cents</p>`. У `Invoice` две причины меняться: правила денег (налоги, округление) и внешний вид страницы (дизайнер хочет таблицу вместо списка). Изменение страницы может случайно сломать вычисление, а дизайнеру приходится читать финансовый код.

Перенесём страницу в отдельный модуль:

```rust
struct Invoice {
    lines: Vec<(String, u64)>,
}

impl Invoice {
    fn total_cents(&self) -> u64 {
        self.lines.iter().map(|(_, cents)| cents).sum()
    }
}

mod render {
    use super::Invoice;

    pub fn html(invoice: &Invoice) -> String {
        format!("<p>Total: {} cents</p>", invoice.total_cents())
    }
}

fn main() {
    let invoice = Invoice {
        lines: vec![("tea".to_string(), 250), ("cake".to_string(), 400)],
    };
    println!("{}", render::html(&invoice));
}
```

Она печатает ту же строку. Теперь `Invoice` отвечает за деньги, а `render` — за страницу. Функцию `render::csv` можно добавить рядом с `html`, не трогая `Invoice`.

**Когда НЕ применять:** не делайте структуру на каждую функцию. «Одна причина меняться» — про *причины*, а не про число методов. У `Rectangle` с методами `area`, `perimeter` и `scale` одна причина меняться (геометрия), и это нормально.

## 3. O — открытость/закрытость

*Программа должна быть открыта для расширения, но закрыта для изменения:* можно добавлять новое поведение, не правя код, который уже работает. В Rust есть два способа, и выбор важен.

**Способ 1: перечисление.** Все фигуры — варианты одного типа, а `area` — это `match`. Допустим, добавили новый вариант `Triangle`, а `area` обновить забыли:

```rust
enum Shape {
    Circle(f64),
    Rect(f64, f64),
    Triangle(f64, f64), // the new variant
}

fn area(shape: &Shape) -> f64 {
    match shape {
        Shape::Circle(r) => std::f64::consts::PI * r * r,
        Shape::Rect(w, h) => w * h,
    }
}

fn main() {
    println!("{}", area(&Shape::Rect(2.0, 3.0)));
}
```

```text
error[E0004]: non-exhaustive patterns: `&Shape::Triangle(_, _)` not covered
  --> src/main.rs:8:11
   |
 8 |     match shape {
   |           ^^^^^ pattern `&Shape::Triangle(_, _)` not covered
   |
   ...

For more information about this error, try `rustc --explain E0004`.
error: could not compile `demo` (bin "demo") due to 1 previous error
```

На первый взгляд это похоже на нарушение открытости/закрытости: чтобы добавить фигуру, нужно править `area` и каждый другой `match`. Но посмотрите, что сделал компилятор: он нашёл **каждое место**, которое вы забыли (ошибка E0004, *non-exhaustive patterns* — неполное покрытие образцов). С перечислением «забыть обработать новый случай» невозможно. Это страховочная сетка, и она именно то, что нужно, когда набор случаев **закрыт**: вы знаете их все (дни недели, состояния заказа, команды вашей собственной программы).

**Способ 2: трейт.** Когда другие люди должны иметь возможность добавлять фигуры, сделайте `Shape` трейтом. Новые фигуры — это новые типы, и старый код не правится:

```rust
trait Shape {
    fn area(&self) -> f64;
}

struct Circle(f64);
struct Rect(f64, f64);

impl Shape for Circle {
    fn area(&self) -> f64 {
        std::f64::consts::PI * self.0 * self.0
    }
}

impl Shape for Rect {
    fn area(&self) -> f64 {
        self.0 * self.1
    }
}

// Added later, maybe by somebody else: nothing above is edited.
struct Triangle(f64, f64);

impl Shape for Triangle {
    fn area(&self) -> f64 {
        self.0 * self.1 / 2.0
    }
}

fn total_area(shapes: &[Box<dyn Shape>]) -> f64 {
    shapes.iter().map(|s| s.area()).sum()
}

fn main() {
    let shapes: Vec<Box<dyn Shape>> = vec![
        Box::new(Circle(1.0)),
        Box::new(Rect(2.0, 3.0)),
        Box::new(Triangle(4.0, 5.0)),
    ];
    println!("{}", total_area(&shapes));
}
```

**Что происходит:**

1. `trait Shape` — точка расширения: любая вещь с `area` является фигурой.
2. `Circle` и `Rect` реализуют его. `total_area` работает с любым `Box<dyn Shape>` (Урок 06.1).
3. `Triangle` добавлен ниже, пусть даже в другом файле или крейте. Ни `Circle`, ни `Rect`, ни `total_area` не изменились.
4. `main` складывает площади (около 3,14, 6 и 10) и печатает сумму.

```text
19.141592653589793
```

Как выбирать:

| | Перечисление | Трейт |
|---|---|---|
| Добавление нового **типа** (варианта) | Править перечисление и каждый `match` (компилятор их перечислит) | Написать новый тип, старое не трогать |
| Добавление новой **операции** (новой функции) | Написать новую функцию с `match`, старое не трогать | Добавить метод в трейт и каждому реализующему типу |
| Кто может добавлять типы? | Только вы | Любой, даже другие крейты |

> 📏 **Соглашение:** если случаи известны и неизменны, используйте `enum`. Если случаи должны добавлять посторонние, используйте трейт. Сомневаясь, начинайте с перечисления: оно проще (KISS, Keep It Simple, Stupid — «делай проще», Урок 08.1), и компилятор его охраняет.

**Когда НЕ применять:** не создавайте трейт «чтобы было открыто», если никто никогда не добавит случай. Это гибкость ради воображаемого будущего, от которой предостерегал принцип YAGNI (You Aren't Gonna Need It, «тебе это не понадобится») в Уроке 08.1.

## 4. L — подстановка Лисков

*Любую реализацию трейта можно использовать везде, где ожидается трейт, без сюрпризов.* Принцип назван в честь учёного-информатика Барбары Лисков (Barbara Liskov). Компилятор проверяет, что у вашего метода правильные *типы*, но у трейта есть и **контракт**, который типами не выразить: «никогда не паникует», «результат не больше входа», «сохраняет порядок».

Вот трейт скидки с контрактом, записанным в документации, и купон, который нарушает его для дешёвых заказов:

```rust
/// Contract: the result is never higher than `cents` and the method never panics.
trait Discount {
    fn apply(&self, cents: u64) -> u64;
}

struct Percent(u64);
struct Coupon(u64);

impl Discount for Percent {
    fn apply(&self, cents: u64) -> u64 {
        cents - cents * self.0 / 100
    }
}

impl Discount for Coupon {
    fn apply(&self, cents: u64) -> u64 {
        cents - self.0
    }
}

fn checkout(cents: u64, discounts: &[Box<dyn Discount>]) -> u64 {
    discounts.iter().fold(cents, |price, d| d.apply(price))
}

fn main() {
    let discounts: Vec<Box<dyn Discount>> = vec![Box::new(Percent(10)), Box::new(Coupon(500))];
    println!("{}", checkout(2000, &discounts));
    println!("{}", checkout(300, &discounts));
}
```

**Что происходит:**

1. Контракт гласит: результат не выше `cents`, и `apply` никогда не паникует.
2. `Percent` его соблюдает. `Coupon` вычитает фиксированную сумму, но если цена ниже купона, `cents - self.0` вызывает переполнение вниз.
3. `checkout` знает только трейт. Для заказа в 2000 центов всё работает. Для заказа в 300 центов купон паникует.

```text
1300

thread 'main' (5591460) panicked at src/main.rs:17:9:
attempt to subtract with overflow
```

(Число после `thread 'main'` и точный текст на вашем компьютере могут отличаться. В сборке `--release` вычитание молча «завернулось» бы в огромное число, что ещё хуже.) Сигнатура типов была в порядке; *поведение* — нет. `checkout` не может защитить себя от каждой реализации, поэтому каждая реализация должна сдерживать обещание.

Исправление маленькое, а тест может защищать его для каждой реализации. Хорошая привычка — один «контрактный тест», который запускается для всех реализаций:

```rust
/// Contract: the result is never higher than `cents` and the method never panics.
trait Discount {
    fn apply(&self, cents: u64) -> u64;
}

struct Percent(u64);
struct Coupon(u64);

impl Discount for Percent {
    fn apply(&self, cents: u64) -> u64 {
        cents - cents * self.0 / 100
    }
}

impl Discount for Coupon {
    fn apply(&self, cents: u64) -> u64 {
        cents.saturating_sub(self.0) // never below zero
    }
}

fn main() {
    println!("{}", Percent(10).apply(2000));
    println!("{}", Coupon(500).apply(300));
}

#[cfg(test)]
mod tests {
    use super::*;

    // One test of the contract, run against every implementation.
    fn check_contract(discount: &dyn Discount) {
        for cents in [0, 1, 99, 300, 10_000] {
            assert!(
                discount.apply(cents) <= cents,
                "discount raised the price of {cents}"
            );
        }
    }

    #[test]
    fn all_discounts_keep_the_contract() {
        check_contract(&Percent(10));
        check_contract(&Percent(100));
        check_contract(&Coupon(500));
    }
}
```

```text
running 1 test
test tests::all_discounts_keep_the_contract ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

> 🧪 **Тестирование:** запишите контракт как функцию, принимающую `&dyn Trait`, и запускайте её для каждой реализации. Когда добавляется новая реализация, добавьте в тест одну строку. У стандартной библиотеки тоже есть контракты, например: если `a == b`, то `hash(a) == hash(b)`. Тип, который нарушает это правило, заставляет `HashMap` терять свои ключи.

**Когда НЕ применять:** Лисков — не повод запрещать любые различия между реализациями. Реализации могут быть быстрее, медленнее или осторожнее. Они обязаны только сдерживать обещания, данные трейтом.

## 5. I — разделение интерфейсов

*Не заставляйте пользователей трейта зависеть от методов, которые им не нужны.* В Rust у этого есть конкретный симптом: метод, который реализация не может поддержать и заполняет `unimplemented!()`.

```rust
trait Store {
    fn get(&self, key: &str) -> Option<String>;
    fn put(&mut self, key: &str, value: &str);
}

struct Snapshot(Vec<(String, String)>); // read-only by nature

impl Store for Snapshot {
    fn get(&self, key: &str) -> Option<String> {
        self.0
            .iter()
            .find(|(k, _)| k == key)
            .map(|(_, v)| v.clone())
    }
    fn put(&mut self, _key: &str, _value: &str) {
        unimplemented!("snapshots are read-only")
    }
}

fn save_default_language(store: &mut impl Store) {
    store.put("lang", "en");
}

fn main() {
    let mut snapshot = Snapshot(vec![("lang".to_string(), "uz".to_string())]);
    println!("{:?}", snapshot.get("lang"));
    save_default_language(&mut snapshot);
}
```

**Что происходит:**

1. У `Store` есть `get` и `put`. `Snapshot` — замороженная копия данных, поэтому записывать он не может.
2. Его `put` вызывает `unimplemented!` — макрос, который паникует с сообщением «not implemented». Код компилируется.
3. `save_default_language` принимает любой `Store`, включая снимок. Ошибка обнаруживается только во время работы:

```text
Some("uz")

thread 'main' (5591981) panicked at src/main.rs:16:9:
not implemented: snapshots are read-only
```

Исправление — разбить трейт на небольшие части, чтобы функция просила только то, что ей действительно нужно:

```rust
trait Lookup {
    fn get(&self, key: &str) -> Option<String>;
}

trait Update {
    fn put(&mut self, key: &str, value: &str);
}

struct Snapshot(Vec<(String, String)>);

impl Lookup for Snapshot {
    fn get(&self, key: &str) -> Option<String> {
        self.0
            .iter()
            .find(|(k, _)| k == key)
            .map(|(_, v)| v.clone())
    }
}

fn save_default_language(store: &mut impl Update) {
    store.put("lang", "en");
}

fn main() {
    let mut snapshot = Snapshot(vec![("lang".to_string(), "uz".to_string())]);
    println!("{:?}", snapshot.get("lang"));
    save_default_language(&mut snapshot);
}
```

Теперь ошибка не компилируется:

```text
error[E0277]: the trait bound `Snapshot: Update` is not satisfied
  --> src/main.rs:27:27
   |
27 |     save_default_language(&mut snapshot);
   |     --------------------- ^^^^^^^^^^^^^ unsatisfied trait bound
   |     |
   |     required by a bound introduced by this call
   |
help: the trait `Update` is not implemented for `Snapshot`
   ...

For more information about this error, try `rustc --explain E0277`.
error: could not compile `demo` (bin "demo") due to 1 previous error
```

Паника во время работы превратилась в ошибку компиляции, указывающую на конкретный вызов. Для записи нужен `Update`, для чтения — `Lookup`, и тип реализует только то, что умеет. Стандартная библиотека поступает так же: `Read`, `Write` и `BufRead` — отдельные трейты, а `Iterator` и `DoubleEndedIterator` тоже отдельные.

**Когда НЕ применять:** не делайте по трейту на каждый метод. Методы, которые всегда используются вместе (`open`, `read`, `close` одного ресурса), принадлежат одному трейту. Разделяйте, когда какие-то реализации не могут, или какие-то пользователи не нуждаются в части методов.

## 6. D — инверсия зависимостей

*Код верхнего уровня должен зависеть от абстракции, а не от детали нижнего уровня.* Бизнес-правило «записать отчёт» не должно знать, идёт отчёт на экран, в файл или в сетевое соединение.

Соблазнительный вариант вызывает `println!` внутри функции отчёта. Тогда функцию нельзя протестировать без перехвата экрана и нельзя использовать для файла. Инвертированный вариант просит *что-то, куда можно записывать*, — это стандартный трейт `std::io::Write`:

```rust
use std::io::{self, Write};

fn write_report(out: &mut impl Write, items: &[&str]) -> io::Result<()> {
    writeln!(out, "{} items", items.len())?;
    for item in items {
        writeln!(out, "- {item}")?;
    }
    Ok(())
}

fn main() -> io::Result<()> {
    write_report(&mut io::stdout(), &["tea", "cake"])
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn report_lists_the_items() {
        let mut buffer: Vec<u8> = Vec::new();
        write_report(&mut buffer, &["tea"]).unwrap();
        assert_eq!(String::from_utf8(buffer).unwrap(), "1 items\n- tea\n");
    }
}
```

**Что происходит:**

1. `write_report` принимает `out: &mut impl Write`: любой тип, реализующий `Write` (терминал, файл, сетевой сокет, обычный `Vec<u8>`).
2. `writeln!` похож на `println!`, но пишет в `out`. Оператор `?` передаёт ошибку наверх, потому что запись может не удаться.
3. `main` подключает настоящий экран: `io::stdout()`.
4. Тест подключает `Vec<u8>` — растущий список байтов — и затем проверяет текст. Ни терминала, ни файла, ни мусора.

```text
2 items
- tea
- cake
```

Тест тоже проходит (`cargo test` показывает `report_lists_the_items ... ok`).

Слово «инверсия» означает, что стрелка зависимости развёрнута: вместо того чтобы отчёт зависел от экрана, оба зависят от маленького трейта `Write`. Функция `main`, которая знает о реальном мире, соединяет части. Урок 08.9 развивает эту идею (внедрение зависимостей).

**Когда НЕ применять:** маленькой программе, которая печатает одну строку, `Write` не нужен. Добавляйте абстракцию там, где вещи действительно различаются: реальный мир против тестов, файл против сети.

## Частые ошибки

**Ошибка 1: трейт на каждую структуру.** `trait UserRepository` ровно с одной реализацией, написанный «ради SOLID», — просто лишняя косвенность (YAGNI, Урок 08.1). Трейт оправдан второй реализацией или настоящей потребностью, например тестированием.

**Ошибка 2: игнорирование контракта.** Реализовать трейт лишь для того, чтобы типы сошлись, а поведение удивляло пользователей, значит нарушить Лисков (раздел 4).

**Ошибка 3: подсчёт принципов вместо чтения кода.** SOLID — набор вопросов, которые стоит задавать, когда изменение даётся тяжело. Если код прост и изменения лёгкие, принципы уже выполнены.

## Попробуйте сами

Вы сделаете «открытую» версию фигур, расширите её, не правя старый код, и защитите тестами.

1. Создайте пакет:

```bash
cd ~/projects/rust
cargo new code/p08-solid
cd code/p08-solid
```

2. Скопируйте трейт `Shape`, `Circle`, `Rect`, `Triangle`, `total_area` и `main` из раздела 3 в `src/main.rs`.

3. **Не меняя** ни один из существующих элементов, добавьте новую фигуру `Square(f64)` (площадь — сторона на сторону) с собственным `impl Shape`. Добавьте её в вектор в `main`. Запустите `cargo run`; итог должен стать около `28.14`.

4. Добавьте модуль `#[cfg(test)]` с двумя тестами: `no_shape_has_a_negative_area` (пройдите по всем четырём фигурам и проверьте `assert!(shape.area() >= 0.0)`) и `total_area_adds_every_shape` (сравните итог с `std::f64::consts::PI + 25.0` через `(total - expected).abs() < 1e-9`, потому что числа с плавающей точкой нельзя сравнивать через `==`).

5. Запустите `cargo test`:

```bash
cargo test
```

```text
running 2 tests
test tests::no_shape_has_a_negative_area ... ok
test tests::total_area_adds_every_shape ... ok

test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

6. Проверьте и сохраните:

```bash
cargo fmt
cargo clippy
git add .
git commit -m "feat(p08): shapes behind a trait"
```

**Что происходит:** `no_shape_has_a_negative_area` — контрактная проверка (Лисков): она запускается для каждой фигуры, и новая фигура, нарушающая обещание, будет поймана сразу. Добавление `Square` не затронуло старый код (открытость/закрытость).

## Упражнения

### Упражнение 1 — Какая буква? (легко)

Сопоставьте каждую проблему с буквой SOLID:

1. Структура и разбирает конфигурацию, и отправляет сетевые запросы.
2. Функцию `fn save(x: &mut FileStore)` нельзя протестировать без настоящего файла.
3. Каждый раз, когда добавляется новый способ оплаты, нужно править шесть выражений `match` по строкам, и некоторые забывают.

<details>
<summary>Подсказка</summary>

Ищите ключевые слова: «и... и» (причины меняться), «нельзя протестировать» (зависит от детали), «забывают» (расширение).

</details>

<details>
<summary>Ответ</summary>

1. S, единственная ответственность.
2. D, инверсия зависимостей: зависьте от трейта вроде `Write` или `Read`, а не от `FileStore`.
3. O, открытость/закрытость. Хорошее исправление — `enum PaymentMethod` (компилятор находит забытые `match`) или трейт, если способы добавляют посторонние. Сопоставление по строкам — худшее из обоих миров.

Почему это работает: каждый принцип называет то, что мешает, и указывает на лекарство.

</details>

### Упражнение 2 — Протестируйте контракт (средне)

Тип `Capped` ниже должен возвращать цену, но не больше `max` центов. Он нарушает контракт `Discount` (результат никогда не должен быть выше входа). **Напишите тест**, который находит ошибку, затем исправьте код.

```rust
/// Contract: the result is never higher than `cents`.
trait Discount {
    fn apply(&self, cents: u64) -> u64;
}

/// Meant to be "the price, but at most `max` cents".
struct Capped(u64);

impl Discount for Capped {
    fn apply(&self, _cents: u64) -> u64 {
        self.0
    }
}

fn main() {
    println!("{}", Capped(100).apply(250));
    println!("{}", Capped(100).apply(50));
}
```

<details>
<summary>Подсказка</summary>

Переберите несколько цен (включая цену ниже предела, например 50) и проверьте `apply(cents) <= cents`. В исправлении используется `min`.

</details>

<details>
<summary>Ответ</summary>

```rust
/// Contract: the result is never higher than `cents`.
trait Discount {
    fn apply(&self, cents: u64) -> u64;
}

struct Capped(u64);

impl Discount for Capped {
    fn apply(&self, cents: u64) -> u64 {
        cents.min(self.0)
    }
}

fn main() {
    println!("{}", Capped(100).apply(250));
    println!("{}", Capped(100).apply(50));
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn capped_never_raises_the_price() {
        let discount = Capped(100);
        for cents in [0, 50, 100, 250] {
            assert!(discount.apply(cents) <= cents, "price {cents} was raised");
        }
    }
}
```

Почему это работает: с ошибочным кодом тест падает на `cents = 0` (результат 100 больше 0). После исправления `cents.min(self.0)` никогда не больше `cents`.

</details>

### Упражнение 3 — Перечисление или трейт? (сложно)

У утилиты командной строки есть команды `start` и `stop`, сегодня они обрабатываются через `match` по перечислению. Через год другие команды должны иметь возможность добавлять свои команды из собственных крейтов. Как вы перепроектируете это и что потеряете?

<details>
<summary>Подсказка</summary>

Если случаи добавляют посторонние, набор случаев открыт. Подумайте о трейте с методом `run` и словаре из имён команд в трейт-объекты.

</details>

<details>
<summary>Ответ</summary>

Определите `trait Command { fn run(&self) -> String; }`, реализуйте его для `Start` и `Stop` и храните реестр `HashMap<String, Box<dyn Command>>`. Крейт-плагин реализует трейт и добавляет свою команду в реестр; больше ничего не правится.

Почему это работает: `Command` — точка расширения. Что вы теряете: компилятор больше не перечисляет «забытые» команды, отсутствующая команда — это сбой поиска во время работы (`HashMap` вернёт `None`), и каждый вызов идёт через указатель `dyn` (Урок 06.1). Это справедливая цена, только если посторонние действительно добавляют команды.

</details>

## Вопросы для самопроверки

1. Что говорит «S» в SOLID, одним предложением?

<details>
<summary>Ответ</summary>

У части программы должна быть одна причина меняться.

</details>

2. Почему ошибка неполного `match` — это достоинство, когда вы используете перечисление?

<details>
<summary>Ответ</summary>

Когда вы добавляете вариант, компилятор перечисляет каждый `match`, который вы забыли обновить, поэтому ничего не пропустить.

</details>

3. Что трейт может обещать такого, чего компилятор не проверит?

<details>
<summary>Ответ</summary>

Свой контракт: поведение вроде «никогда не паникует», «сохраняет порядок», «результат не больше входа».

</details>

4. Как разделение трейта помогает компилятору находить ошибки?

<details>
<summary>Ответ</summary>

Функция просит только нужный ей маленький трейт, поэтому передача типа без него — ошибка компиляции, а не паника `unimplemented!` во время работы.

</details>

## Итоги

- **S**: одна причина меняться на часть; разделяйте по причинам, а не по числу методов.
- **O**: добавляйте поведение, не правя старый код. Используйте `enum` для закрытого набора (компилятор находит забытые случаи) и трейт для открытого.
- **L**: каждая реализация должна соблюдать контракт трейта; защищайте его одним контрактным тестом, запускаемым для всех реализаций.
- **I**: маленькие трейты; метод, заполненный `unimplemented!()`, — симптом. Разделение превращает паники во время работы в ошибки компиляции.
- **D**: зависьте от трейта (`impl Write`, `impl BufRead`), а не от экрана, файла или сети; реальный мир подключает `main`.
- SOLID — набор вопросов, которые стоит задавать, когда изменения даются тяжело, а не рецепт добавления трейтов повсюду.

## Источники и что почитать

- [The Rust Programming Language — 18.2 Using Trait Objects That Allow for Values of Different Types](https://doc.rust-lang.org/book/ch18-02-trait-objects.html) — официальная глава о трейт-объектах и расширении.
- [Rust API Guidelines — Flexibility](https://rust-lang.github.io/api-guidelines/flexibility.html) — советы принимать обобщённые параметры (`impl Read`, `impl Write`), чтобы вызывающий мог подставить то, что у него есть.
- [std::io::Write](https://doc.rust-lang.org/std/io/trait.Write.html) и [std::io::BufRead](https://doc.rust-lang.org/std/io/trait.BufRead.html) — стандартные трейты, использованные в этом уроке.
- [Robert C. Martin — Principles of OOD](http://butunclebob.com/ArticleS.UncleBob.PrinciplesOfOod) — оригинальная статья, собравшая эти принципы (OOD — Object-Oriented Design, объектно-ориентированное проектирование, то есть проектирование вокруг классов и объектов).

---

[🏠 Начало](../START-HERE.md) · [← 08.2 Связность, сопряжение и разделение ответственности](02-cohesion-coupling-and-separation-of-concerns.md) · [📋 Этап 08](README.md) · [08.4 Композиция вместо наследования →](04-composition-over-inheritance.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
