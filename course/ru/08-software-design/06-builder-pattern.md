<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 08.5 Проектирование через типы: недопустимые состояния невозможны](05-type-driven-design.md) · [📋 Этап 08](README.md) · [08.7 RAII и guard-объекты →](07-raii-and-guards.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/08-software-design/06-builder-pattern.md) <!-- nav -->

# Урок 08.6 — Паттерн «строитель»

> **Этап 08 — Проектирование ПО: принципы, паттерны и соглашения** · Урок 6 из 17 · ⏱ около 2,5 часа (2 занятия)

## Что вы узнаете

- Почему конструктор с большим числом параметров трудно читать и менять.
- Более простой первый шаг: `Default` и синтаксис обновления структуры `..Default::default()`.
- Как написать **строитель** (builder): методы-установщики, которые выстраиваются в цепочку, и метод `build`, проверяющий правила.
- Два стиля строителя в Rust (потребляющий `self` и `&mut self`) и типичную ошибку компилятора второго стиля.
- Когда строитель — это уже слишком.

## Перед началом

- Вы прошли [Урок 08.5 — Проектирование через типы](05-type-driven-design.md): `build()` — это умный конструктор.
- Вы знаете структуры и методы ([Уроки 04.1 и 04.2](../04-structs-enums-collections-errors/01-structs.md)), `Option`, `Result` и владение `self` ([Урок 03.2](../03-ownership-and-borrowing/02-ownership-rules.md)).
- Вы знаете трейт `Default` и `#[derive(...)]` ([Урок 05.2](../05-rust-intermediate/02-traits.md)).

## Новые слова

| Слово | Значение |
|---|---|
| **строитель** (builder) | Вспомогательный тип, который по одному собирает настройки и создаёт итоговое значение на последнем шаге. |
| **цепочка методов** (method chaining) | Вызов методов один за другим в одном выражении: `a.x().y().z()`. Работает, если каждый метод возвращает то, что может использовать следующий. |
| **синтаксис обновления структуры** (struct update syntax) | `Server { port: 9000, ..other }` создаёт структуру из `other`, заменяя перечисленные поля. |
| **потребляющий строитель** (consuming builder) | Строитель, методы которого принимают `self` по значению и возвращают его (или новый). |
| **строитель на `&mut`** | Строитель, методы которого принимают `&mut self` и возвращают `&mut Self`. |
| **необязательный параметр** (optional parameter) | Настройка с разумным значением по умолчанию, которую вызывающий может опустить. В Rust нет необязательных аргументов функций; пробел заполняют строители и `Default`. |
| **именованный аргумент** (named argument) | Аргумент, который передаётся как `имя: значение`. В Rust их тоже нет; методы строителя читаются как именованные аргументы (`.port(8443)`). |

## 1. Проблема: конструктор со множеством параметров

Серверу нужны хост, порт, предел соединений и несколько флагов. Очевидный конструктор:

```rust
#[allow(dead_code)] // the fields are only printed with {:?}
#[derive(Debug)]
struct Server {
    host: String,
    port: u16,
    max_connections: u32,
    tls: bool,
    verbose: bool,
    timeout_secs: u32,
}

impl Server {
    fn new(
        host: &str,
        port: u16,
        max_connections: u32,
        tls: bool,
        verbose: bool,
        timeout_secs: u32,
    ) -> Server {
        Server {
            host: host.to_string(),
            port,
            max_connections,
            tls,
            verbose,
            timeout_secs,
        }
    }
}

fn main() {
    // What do `true, false` and `30` mean here? Which one is `tls`?
    let server = Server::new("localhost", 8080, 100, true, false, 30);
    println!("{server:?}");
}
```

```text
Server { host: "localhost", port: 8080, max_connections: 100, tls: true, verbose: false, timeout_secs: 30 }
```

Посмотрите на вызов: `Server::new("localhost", 8080, 100, true, false, 30)`. Какой из них `tls` (Transport Layer Security, шифрованные соединения), а какой `verbose` (печатать дополнительные сообщения)? В правильном ли порядке `100` и `30`? Компилятор не подскажет, потому что `true` и `false` оба `bool`, а `100` и `30` оба числа. И каждый раз, когда вы добавляете седьмую настройку, нужно менять все вызовы везде.

В Rust нет именованных и необязательных аргументов, поэтому люди используют другие инструменты.

## 2. Сначала самый простой инструмент: `Default` и обновление структуры

Если у всех полей есть разумные значения по умолчанию, строитель вообще не нужен. Реализуйте `Default` (Урок 05.2) и используйте **синтаксис обновления структуры**:

```rust
#[allow(dead_code)] // the fields are only printed with {:?}
#[derive(Debug)]
struct Server {
    host: String,
    port: u16,
    max_connections: u32,
    tls: bool,
}

impl Default for Server {
    fn default() -> Server {
        Server {
            host: "localhost".to_string(),
            port: 8080,
            max_connections: 100,
            tls: false,
        }
    }
}

fn main() {
    let plain = Server::default();
    let secure = Server {
        port: 8443,
        tls: true,
        ..Default::default()
    };
    println!("{plain:?}");
    println!("{secure:?}");
}
```

**Что происходит:**

1. `impl Default for Server` определяет обычные значения.
2. `Server::default()` создаёт сервер со всеми значениями по умолчанию.
3. `Server { port: 8443, tls: true, ..Default::default() }` задаёт два поля по имени, а остальное берёт из значения по умолчанию. Это читается как именованные аргументы, а новое поле со значением по умолчанию не ломает существующие вызовы.

```text
Server { host: "localhost", port: 8080, max_connections: 100, tls: false }
Server { host: "localhost", port: 8443, max_connections: 100, tls: true }
```

> 📏 **Соглашение:** сначала используйте `Default` и `..Default::default()`. Это самое простое (KISS, Keep It Simple, Stupid — «делай проще», Урок 08.1) и не требует лишнего типа. Переходите к строителю, только когда этого недостаточно.

Когда этого недостаточно?

- Некоторые поля **обязательны** и не имеют разумного значения по умолчанию (хост).
- Значения нужно **проверить** до того, как значение появится (порт не должен быть нулём), и вам нужен `Result`, а не сломанная структура.
- Некоторые значения **вычисляются** из настроек, или у итогового типа приватные поля.
- Шаги настройки **повторяются** или условны (добавить заголовок, добавить ещё заголовок, ...).

## 3. Строитель

Строитель — отдельный тип с теми же настройками, по одному методу на каждую настройку и итоговым `build`. Каждый установщик возвращает строитель, чтобы вызовы выстраивались в цепочку:

```rust
#[allow(dead_code)] // the fields are only printed with {:?}
#[derive(Debug)]
struct Server {
    host: String,
    port: u16,
    max_connections: u32,
    tls: bool,
}

#[derive(Debug, PartialEq)]
enum BuildError {
    EmptyHost,
    ZeroPort,
    NoConnections,
}

struct ServerBuilder {
    host: String,
    port: u16,
    max_connections: u32,
    tls: bool,
}

impl ServerBuilder {
    // The host is required, so it is a parameter of `new`.
    fn new(host: impl Into<String>) -> ServerBuilder {
        ServerBuilder {
            host: host.into(),
            port: 8080,
            max_connections: 100,
            tls: false,
        }
    }

    fn port(mut self, port: u16) -> ServerBuilder {
        self.port = port;
        self
    }

    fn max_connections(mut self, count: u32) -> ServerBuilder {
        self.max_connections = count;
        self
    }

    fn tls(mut self, enabled: bool) -> ServerBuilder {
        self.tls = enabled;
        self
    }

    // `build` checks the rules once and returns the finished value.
    fn build(self) -> Result<Server, BuildError> {
        if self.host.trim().is_empty() {
            return Err(BuildError::EmptyHost);
        }
        if self.port == 0 {
            return Err(BuildError::ZeroPort);
        }
        if self.max_connections == 0 {
            return Err(BuildError::NoConnections);
        }
        Ok(Server {
            host: self.host,
            port: self.port,
            max_connections: self.max_connections,
            tls: self.tls,
        })
    }
}

fn main() {
    let server = ServerBuilder::new("localhost").port(8443).tls(true).build();
    println!("{server:?}");

    let broken = ServerBuilder::new("localhost").max_connections(0).build();
    println!("{broken:?}");
}
```

**Что происходит:**

1. `ServerBuilder::new(host)` принимает только обязательное. `impl Into<String>` позволяет передать `&str` или `String` (параметр преобразуется через `.into()`).
2. У каждой другой настройки есть значение по умолчанию в `new` и установщик. Установщик принимает `mut self` (строитель по значению), меняет одно поле и возвращает `self`. Поэтому `.port(8443).tls(true)` выстраивается в цепочку: каждый вызов потребляет старый строитель и возвращает тот же, изменённый.
3. `build(self)` — **умный конструктор** из Урока 08.5. Он проверяет правила и возвращает `Result<Server, BuildError>`. Существующий `Server` всегда корректен.
4. `main` строит хороший сервер и плохой (ноль соединений) и печатает оба результата.

```text
Ok(Server { host: "localhost", port: 8443, max_connections: 100, tls: true })
Err(NoConnections)
```

Вызов `ServerBuilder::new("localhost").port(8443).tls(true).build()` читается почти как английский, у каждого значения есть имя, а компилятор больше не даст перепутать `bool` и `u32`.

> 📏 **Соглашение:** стандартные имена: `Type::builder()` или `TypeBuilder::new()` для начала, по одному методу на настройку с именем настройки (а не `set_port`) и `build()` для завершения. В Rust API Guidelines (API — Application Programming Interface, программный интерфейс: то, что библиотека предлагает другому коду; Урок 08.11) это называется паттерном «строитель» (пункт C-BUILDER).

> 🧪 **Тестирование:** тестируйте `build` так же, как любой умный конструктор: один тест для значений по умолчанию и один для каждого правила, из-за которого сборка не удаётся. Такие тесты вы напишете в упражнениях.

## 4. Два стиля: `self` и `&mut self`

Строитель из раздела 3 — **потребляющий**: каждый метод принимает `self` по значению. В стандартной библиотеке есть и другой стиль: `std::process::Command` и `std::fs::OpenOptions` используют `&mut self` и возвращают `&mut Self`, а `std::thread::Builder` потребляет `self`.

| | Потребляющий (`self -> Self`) | На `&mut` (`&mut self -> &mut Self`) |
|---|---|---|
| Одна цепочка, заканчивающаяся `build()` | Работает | Работает |
| Настройка по шагам, с `if` или циклами | Нужно `builder = builder.x()` | Естественно: `builder.x();` |
| Несколько значений из одного строителя | Нужен `Clone` | `build(&self)` можно вызывать много раз |
| `build` может забирать поля | Да, копии не нужны | Нет, данные приходится клонировать |

У стиля на `&mut` есть известная ловушка:

```rust
struct Server {
    port: u16,
}

struct ServerBuilder {
    port: u16,
}

impl ServerBuilder {
    fn new() -> ServerBuilder {
        ServerBuilder { port: 8080 }
    }
    fn port(&mut self, port: u16) -> &mut ServerBuilder {
        self.port = port;
        self
    }
    fn build(&self) -> Server {
        Server { port: self.port }
    }
}

fn main() {
    let builder = ServerBuilder::new().port(9000);
    let server = builder.build();
    println!("{}", server.port);
}
```

```text
error[E0716]: temporary value dropped while borrowed
  --> src/main.rs:23:19
   |
23 |     let builder = ServerBuilder::new().port(9000);
   |                   ^^^^^^^^^^^^^^^^^^^^           - temporary value is freed at the end of this statement
   |                   |
   |                   creates a temporary value which is freed while still in use
24 |     let server = builder.build();
   ...

For more information about this error, try `rustc --explain E0716`.
error: could not compile `demo` (bin "demo") due to 1 previous error
```

**Что происходит:**

1. `ServerBuilder::new()` создаёт строитель как **временное** значение; оно не сохранено в переменной.
2. `.port(9000)` возвращает `&mut ServerBuilder` — *ссылку на временное значение*. Ссылка сохраняется в `builder`.
3. В конце инструкции временное значение уничтожается, но `builder` всё ещё указывает на него. Следующая строка использовала бы висячую ссылку, поэтому компилятор её останавливает (Урок 03.4).

Исправления: завершить цепочку в одном выражении (`...port(9000).build()`) или сначала сохранить строитель в переменной (`let mut builder = ServerBuilder::new();`), как делает рабочая версия:

```rust
struct Server {
    port: u16,
}

struct ServerBuilder {
    port: u16,
}

impl ServerBuilder {
    fn new() -> ServerBuilder {
        ServerBuilder { port: 8080 }
    }
    fn port(&mut self, port: u16) -> &mut ServerBuilder {
        self.port = port;
        self
    }
    fn build(&self) -> Server {
        Server { port: self.port }
    }
}

fn main() {
    // Style 1: one chain, in a single statement.
    let first = ServerBuilder::new().port(9000).build();

    // Style 2: keep the builder in a variable, configure it step by step, reuse it.
    let mut builder = ServerBuilder::new();
    let production = true;
    if production {
        builder.port(443);
    }
    let second = builder.build();
    let third = builder.port(444).build();
    println!("{} {} {}", first.port, second.port, third.port);
}
```

```text
9000 443 444
```

`first` использует одну цепочку. `builder` хранится в переменной, настраивается через `if` и собирается дважды; второй `build` использует его повторно с другим портом. Какой стиль выбрать? Для строителя, который `build()` превращает в значение с владеющими данными (строки, векторы), проще и без копий потребляющий стиль. Для строителя, который настраивает действие со многими необязательными шагами (`Command`), удобен стиль на `&mut`. Подходят оба; будьте последовательны внутри одного крейта.

## 5. Строители и остальной дизайн

- **Обязательные поля.** Поместите их в `new`, как `host` выше. Если обязательное поле забыто, код не компилируется. Строитель, который позволяет пропустить обязательное поле и жалуется только в `build()`, переносит ошибку с этапа компиляции на этап выполнения. Урок 08.8 показывает, как паттерн typestate может даже сделать `build()` недоступным, пока не заданы обязательные поля.
- **Проверка живёт в `build`.** Одно место, один `Result`. Установщики остаются простыми.
- **Повторяющиеся настройки.** Установщик может добавлять в список (`.header("a").header("b")`). В Упражнении 2 так сделано с начинками.
- **Много строителей.** Писать строители вручную однообразно. Крейты генерируют их по определению структуры (например, `bon`, версия 3.10 на момент проверки этого курса). Сначала научитесь писать строитель вручную, чтобы понимать, что пишет макрос.

## 6. Когда строитель — это слишком

- **Два-три поля, у всех есть значения по умолчанию.** Достаточно `Default` с обновлением структуры. Строитель здесь — шаблонный код (YAGNI, You Aren't Gonna Need It — «тебе это не понадобится», Урок 08.1).
- **Все поля обязательны.** Тогда понятен обычный конструктор `new(a, b, c)` с хорошо типизированными параметрами (newtype, Урок 08.5), и строитель ничего не добавляет.
- **Публичные поля без правил.** Структуру с полями `pub` можно создать литералом структуры; не оборачивайте её в строитель «для стиля».
- **Строитель для каждой структуры программы.** Оставьте строители для типов, которые настраиваются многими способами во многих местах: серверы, запросы, клиенты.

## Частые ошибки

**Ошибка 1: забыть шаг `build`.** `let server = ServerBuilder::new("h").port(80);` даёт `ServerBuilder`, а не `Server`. Компилятор сообщает о несовпадении типов только там, где вы используете его как сервер.

**Ошибка 2: проверка в установщиках.** Если каждый установщик возвращает `Result`, цепочки становятся мучительными (`?` после каждого вызова). Сначала соберите, проверьте один раз в `build`.

**Ошибка 3: публичные поля у готового типа плюс строитель.** Если кто-то всё ещё может написать `Server { port: 0, .. }`, проверки строителя можно обойти. Делайте поля приватными, когда правила важны (Урок 08.2).

**Ошибка 4: строитель на `&mut`, сохранённый из временного значения.** См. раздел 4.

## Попробуйте сами

1. Создайте пакет и скопируйте первый пример строителя из раздела 3 (`ServerBuilder` вместе с его `main`) в `src/main.rs`:

```bash
cd ~/projects/rust
cargo new code/p08-builder
cd code/p08-builder
```

2. Запустите его через `cargo run`. Вы увидите две строки из раздела 3.

3. Добавьте новое правило: порт ниже 1024 разрешён, только когда включён `tls`. Добавьте вариант `BuildError::PrivilegedPortWithoutTls` и проверку в `build`:

```rust
if self.port < 1024 && !self.tls {
    return Err(BuildError::PrivilegedPortWithoutTls);
}
```

4. Добавьте тесты ниже и запустите `cargo test`:

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn defaults_are_valid() {
        assert!(ServerBuilder::new("localhost").build().is_ok());
    }

    #[test]
    fn empty_host_is_rejected() {
        assert_eq!(
            ServerBuilder::new("  ").build().unwrap_err(),
            BuildError::EmptyHost
        );
    }

    #[test]
    fn low_port_needs_tls() {
        let no_tls = ServerBuilder::new("h").port(80).build();
        assert_eq!(no_tls.unwrap_err(), BuildError::PrivilegedPortWithoutTls);
        assert!(ServerBuilder::new("h").port(443).tls(true).build().is_ok());
    }
}
```

```text
running 3 tests
test tests::defaults_are_valid ... ok
test tests::empty_host_is_rejected ... ok
test tests::low_port_needs_tls ... ok

test result: ok. 3 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

5. Проверьте и сохраните:

```bash
cargo fmt
cargo clippy
git add .
git commit -m "feat(p08): server builder with validation"
```

**Что происходит:** все правила корректного `Server` теперь живут в одной функции, и у каждого правила один тест. Новое правило — это вариант, проверка и тест.

## Упражнения

### Упражнение 1 — `Default` или строитель? (легко)

Для каждого типа скажите, достаточно ли `Default` с обновлением структуры или лучше строитель:

1. `Margins { top, right, bottom, left }`, все числа со значением по умолчанию 0.
2. `HttpRequest` (запрос HTTP, HyperText Transfer Protocol — протокола веба) с обязательным URL (Uniform Resource Locator, веб-адрес), необязательным методом, любым числом заголовков и правилом, что у GET-запросов (метода HTTP для чтения данных) нет тела.
3. `Color { r, g, b }`, где все три значения всегда заданы.

<details>
<summary>Подсказка</summary>

Ищите обязательные поля, правила между полями и повторяющиеся настройки.

</details>

<details>
<summary>Ответ</summary>

1. `Default` с обновлением структуры: у всех полей есть значения по умолчанию, правил нет.
2. Строитель: URL обязателен, заголовки повторяются, а правило про GET и тело требует проверки в `build`.
3. Ни то ни другое: обычный конструктор или литерал структуры, потому что все значения обязательны.

Почему это работает: чем больше обязательных полей, правил и повторяющихся настроек, тем больше пользы от строителя.

</details>

### Упражнение 2 — Строитель пиццы с повторяющимися настройками (средне)

Напишите `PizzaBuilder::new(size_cm)` с установщиком `topping(name)`, который можно вызывать много раз, установщиком `extra_cheese(bool)` и `build()`, возвращающим ошибку, когда начинок больше 5. **Напишите тесты** для порядка начинок и для предела.

<details>
<summary>Подсказка</summary>

Строитель хранит `Vec<String>`; `topping` добавляет имя и возвращает `self`. В тесте для предела вызывайте `topping` в цикле через `builder = builder.topping(name)`.

</details>

<details>
<summary>Ответ</summary>

```rust
#[derive(Debug, PartialEq)]
struct Pizza {
    size_cm: u32,
    toppings: Vec<String>,
    extra_cheese: bool,
}

#[derive(Debug, PartialEq)]
enum PizzaError {
    TooManyToppings,
}

struct PizzaBuilder {
    size_cm: u32,
    toppings: Vec<String>,
    extra_cheese: bool,
}

impl PizzaBuilder {
    fn new(size_cm: u32) -> PizzaBuilder {
        PizzaBuilder {
            size_cm,
            toppings: Vec::new(),
            extra_cheese: false,
        }
    }

    // Can be called many times: each call adds one topping.
    fn topping(mut self, name: &str) -> PizzaBuilder {
        self.toppings.push(name.to_string());
        self
    }

    fn extra_cheese(mut self, yes: bool) -> PizzaBuilder {
        self.extra_cheese = yes;
        self
    }

    fn build(self) -> Result<Pizza, PizzaError> {
        if self.toppings.len() > 5 {
            return Err(PizzaError::TooManyToppings);
        }
        Ok(Pizza {
            size_cm: self.size_cm,
            toppings: self.toppings,
            extra_cheese: self.extra_cheese,
        })
    }
}

fn main() {
    let pizza = PizzaBuilder::new(30)
        .topping("ham")
        .topping("olives")
        .extra_cheese(true)
        .build();
    println!("{pizza:?}");
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn collects_toppings_in_order() {
        let pizza = PizzaBuilder::new(30)
            .topping("ham")
            .topping("olives")
            .build()
            .unwrap();
        assert_eq!(pizza.toppings, vec!["ham", "olives"]);
        assert!(!pizza.extra_cheese);
        assert_eq!(pizza.size_cm, 30);
    }

    #[test]
    fn six_toppings_are_too_many() {
        let mut builder = PizzaBuilder::new(30);
        for name in ["a", "b", "c", "d", "e", "f"] {
            builder = builder.topping(name);
        }
        assert_eq!(builder.build(), Err(PizzaError::TooManyToppings));
    }
}
```

Почему это работает: каждый вызов `topping` потребляет строитель и возвращает его с ещё одним элементом. Цикл в тесте показывает цену потребляющего стиля: результат нужно присваивать обратно в `builder`.

</details>

### Упражнение 3 — Повторное использование настроенной основы (сложно)

Программа создаёт основное и резервное соединения, которые делят хост и `tls`, но различаются портом. Напишите потребляющий `ConnectionBuilder`, который можно клонировать, чтобы общая основа настраивалась один раз. **Напишите тест**, что две сборки из одной основы различаются только портом.

<details>
<summary>Подсказка</summary>

Добавьте к строителю `#[derive(Clone)]` и вызывайте `base.clone()` перед потребляющим установщиком.

</details>

<details>
<summary>Ответ</summary>

```rust
#[allow(dead_code)] // the fields are only printed with {:?}
#[derive(Debug)]
struct Connection {
    host: String,
    port: u16,
    tls: bool,
}

#[derive(Clone)]
struct ConnectionBuilder {
    host: String,
    port: u16,
    tls: bool,
}

impl ConnectionBuilder {
    fn new(host: &str) -> ConnectionBuilder {
        ConnectionBuilder {
            host: host.to_string(),
            port: 80,
            tls: false,
        }
    }
    fn port(mut self, port: u16) -> ConnectionBuilder {
        self.port = port;
        self
    }
    fn tls(mut self, tls: bool) -> ConnectionBuilder {
        self.tls = tls;
        self
    }
    fn build(self) -> Connection {
        Connection {
            host: self.host,
            port: self.port,
            tls: self.tls,
        }
    }
}

fn main() {
    // A shared base, configured once.
    let base = ConnectionBuilder::new("db.local").tls(true);
    let primary = base.clone().port(5432).build();
    let replica = base.clone().port(5433).build();
    println!("{primary:?}");
    println!("{replica:?}");
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn clones_share_the_base_but_not_later_changes() {
        let base = ConnectionBuilder::new("h").tls(true);
        let a = base.clone().port(1).build();
        let b = base.clone().port(2).build();
        assert_eq!((a.port, b.port), (1, 2));
        assert!(a.tls && b.tls);
        assert_eq!(a.host, "h");
    }
}
```

Почему это работает: `clone` создаёт независимую копию строителя, поэтому потребляющий вызов `port` на копии не затрагивает `base`. Тест проверяет, что оба соединения делят `host` и `tls` и различаются только `port`.

</details>

## Вопросы для самопроверки

1. Почему `Server::new("h", 80, 100, true, false, 30)` трудно читать?

<details>
<summary>Ответ</summary>

У аргументов нет имён, и многие имеют один и тот же тип, поэтому читатель (и компилятор) не может сказать, какой из них что значит.

</details>

2. Что стоит попробовать до написания строителя?

<details>
<summary>Ответ</summary>

`Default` и синтаксис обновления структуры (`..Default::default()`).

</details>

3. Где строитель выполняет проверку и что он возвращает?

<details>
<summary>Ответ</summary>

В `build`, который возвращает `Result` с готовым значением или ошибкой сборки.

</details>

4. Почему `let b = ServerBuilder::new().port(9000);` не компилируется для строителя на `&mut self`?

<details>
<summary>Ответ</summary>

Цепочка возвращает ссылку на временный строитель, который уничтожается в конце инструкции, поэтому `b` оказался бы висячим (ошибка E0716).

</details>

5. Назовите два случая, когда строитель — это слишком.

<details>
<summary>Ответ</summary>

Структура с несколькими полями, у всех из которых есть значения по умолчанию (используйте `Default`); структура, у которой все поля обязательны (используйте обычный конструктор).

</details>

## Итоги

- Конструктор со множеством параметров нечитаем и хрупок; в Rust нет именованных и необязательных аргументов.
- Начинайте с `Default` и `..Default::default()`.
- Используйте **строитель**, когда есть обязательные поля, правила для проверки, вычисляемые значения или повторяющиеся настройки: установщики выстраиваются в цепочку, а `build()` проверяет и возвращает `Result`.
- Потребляющий стиль (`self`) подходит строителям, которые переносят владеющие данные в результат; стиль на `&mut` — пошаговой настройке, но остерегайтесь временных значений (E0716).
- Обязательные значения — в `new`, правила — в `build`, а поля готового типа держите приватными, если правила важны.
- Не пишите строитель для крошечных структур или структур без правил.

## Источники и что почитать

- [Rust API Guidelines — Builders enable construction of complex values (C-BUILDER)](https://rust-lang.github.io/api-guidelines/type-safety.html) — официальные рекомендации по строителю, включая стили `self` и `&mut self`.
- [Rust Design Patterns — Builder](https://rust-unofficial.github.io/patterns/patterns/creational/builder.html) — паттерн с примером и компромиссами.
- [std::process::Command](https://doc.rust-lang.org/std/process/struct.Command.html) — настоящий строитель на `&mut self` в стандартной библиотеке.
- [The Rust Programming Language — 5.1 Defining and Instantiating Structs](https://doc.rust-lang.org/book/ch05-01-defining-structs.html) — синтаксис обновления структуры.
- [bon на docs.rs](https://docs.rs/bon/latest/bon/) — крейт, генерирующий строители; читайте его, когда сможете написать строитель вручную.

---

[🏠 Начало](../START-HERE.md) · [← 08.5 Проектирование через типы: недопустимые состояния невозможны](05-type-driven-design.md) · [📋 Этап 08](README.md) · [08.7 RAII и guard-объекты →](07-raii-and-guards.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
