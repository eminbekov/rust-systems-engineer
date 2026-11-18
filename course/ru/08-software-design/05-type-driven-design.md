<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 08.4 Композиция вместо наследования](04-composition-over-inheritance.md) · [📋 Этап 08](README.md) · [08.6 Паттерн «строитель» →](06-builder-pattern.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/08-software-design/05-type-driven-design.md) <!-- nav -->

# Урок 08.5 — Проектирование через типы: недопустимые состояния невозможны

> **Этап 08 — Проектирование ПО: принципы, паттерны и соглашения** · Урок 5 из 17 · ⏱ около 135 минут

## Что вы узнаете

- Почему обычные значения `String` и `u32` повсюду порождают ошибки, которые компилятор мог бы предотвратить.
- **Parse, don't validate** («разбирай, а не проверяй»): как превратить проверку в новый тип, который несёт доказательство.
- Как построить «умный конструктор» с приватным полем и использовать `FromStr` и `TryFrom`.
- Как проектировать перечисления так, чтобы противоречивые состояния вообще нельзя было записать.
- Когда этот стиль стоит дороже, чем даёт.

## Перед началом

- Вы прошли [Урок 08.4 — Композиция вместо наследования](04-composition-over-inheritance.md).
- Вы знаете паттерн newtype и правило сирот из [Урока 06.2 — Продвинутые трейты и типы](../06-rust-advanced/02-advanced-traits-and-types.md). Здесь мы идём дальше: newtype, который можно создать только в корректном состоянии.
- Вы знаете `Result` и типы ошибок ([Урок 04.10 — Обработка ошибок](../04-structs-enums-collections-errors/10-error-handling.md)), а также трейты `Display` и `FromStr` ([Урок 05.2](../05-rust-intermediate/02-traits.md)).

## Новые слова

| Слово | Значение |
|---|---|
| **инвариант** (invariant) | Факт о значении, который должен быть истинным всегда. «В адресе электронной почты есть текст до и после `@`» — инвариант типа `Email`. |
| **одержимость примитивами** (primitive obsession) | Использование простых типов (`String`, `u32`, `bool`) для всего, из-за чего компилятор не отличает адрес почты от имени, а цену от количества. |
| **доменный тип** (domain type) | Тип, названный по идее решаемой вами задачи (`Email`, `Percent`, `OrderId`), а не по способу хранения. |
| **умный конструктор** (smart constructor) | Функция (часто `parse` или `new`), которая является единственным способом создать значение типа и проверяет инвариант. Возвращает `Result` или `Option`. |
| **проверить** (validate) | Проверить значение и выбросить ответ (`fn is_valid(&str) -> bool`). Значение сохраняет прежний тип. |
| **разобрать** (parse) | Проверить значение и, если оно хорошее, вернуть *новое значение более строгого типа*, доказывающее, что проверка выполнена. |
| **граница** (boundary) | Место, где данные попадают в вашу программу снаружи: ввод пользователя, файлы, сеть. Лучшее место для разбора. |
| **`FromStr`** | Стандартный трейт, стоящий за `text.parse::<T>()`. |
| **`TryFrom`** | Стандартный трейт для преобразования, которое может не удаться: `T::try_from(value)` возвращает `Result`. |
| **непредставимое состояние** (unrepresentable state) | Сочетание значений, которое типы делают невозможным записать. |

## 1. Проблема: всё — `String`

Посмотрите на эту функцию и её вызов:

```rust
fn is_valid_email(text: &str) -> bool {
    text.contains('@')
}

fn send_email(to: String, subject: String, body: String) {
    println!("To: {to}\nSubject: {subject}\n{body}");
}

fn main() {
    let to = "ana@mail.org".to_string();
    let subject = "Welcome".to_string();
    let body = "Glad to see you".to_string();
    if is_valid_email(&to) {
        send_email(subject, to, body); // arguments swapped: still compiles
    }
}
```

**Что происходит:**

1. `is_valid_email` проверяет, что в тексте есть `@`, и возвращает `bool`. Ответ используется один раз и забывается.
2. `send_email` принимает три `String`. Компилятор видит три значения одного типа и не может знать, какое из них что значит.
3. В `main` программист по ошибке поменял местами `to` и `subject`. Программа компилируется, проходит проверку для `to` и отправляет письмо с неверными строками заголовка.

```text
To: Welcome
Subject: ana@mail.org
Glad to see you
```

Здесь две проблемы:

- **Путаница** компилируется. `String` означает «текст»; он не говорит, *какой именно* это текст.
- **Проверку забывают.** `send_email` не может знать, что кто-то раньше проверил `to`. Каждая функция должна проверять заново (повторная работа, Урок 08.1) или доверять вслепую (ошибка, которая ждёт своего часа).

## 2. Parse, don't validate

Исправление — одна идея: вместо того чтобы проверить значение и оставить прежний тип, **проверьте один раз и верните новый тип**, который существует только для корректных значений. Тип — это доказательство.

```rust
use std::fmt;
use std::str::FromStr;

mod email {
    use super::*;

    #[derive(Debug, PartialEq)]
    pub struct Email(String); // the field is private

    #[derive(Debug, PartialEq)]
    pub enum EmailError {
        Empty,
        MissingAt,
        EmptyPart,
    }

    impl Email {
        pub fn parse(text: &str) -> Result<Email, EmailError> {
            let text = text.trim();
            if text.is_empty() {
                return Err(EmailError::Empty);
            }
            let (user, host) = text.split_once('@').ok_or(EmailError::MissingAt)?;
            if user.is_empty() || host.is_empty() {
                return Err(EmailError::EmptyPart);
            }
            Ok(Email(text.to_string()))
        }
    }

    impl FromStr for Email {
        type Err = EmailError;
        fn from_str(text: &str) -> Result<Email, EmailError> {
            Email::parse(text)
        }
    }

    impl fmt::Display for Email {
        fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
            write!(f, "{}", self.0)
        }
    }
}

use email::Email;

struct Subject(String);

// No validation inside: an `Email` is already proven valid.
fn send_email(to: &Email, subject: &Subject, body: &str) {
    println!("To: {to}\nSubject: {}\n{body}", subject.0);
}

fn main() {
    let subject = Subject("Welcome".to_string());
    match "  ana@mail.org ".parse::<Email>() {
        Ok(to) => send_email(&to, &subject, "Glad to see you"),
        Err(e) => println!("bad address: {e:?}"),
    }
    println!("{:?}", Email::parse("ana.mail.org"));
    println!("{:?}", Email::parse("@mail.org"));
    println!("{:?}", Email::parse("   "));
}
```

**Что происходит:**

1. `struct Email(String)` — newtype (Урок 06.2). Его поле **приватное**, поэтому никакой код вне `mod email` не может создать или изменить `Email` напрямую.
2. `Email::parse` — единственный вход. Она обрезает пробелы, отвергает пустой текст, требует `@` с чем-то по обе стороны и возвращает `Result<Email, EmailError>`. Тип ошибки — перечисление, которое говорит, *какое* правило нарушено.
3. `impl FromStr` позволяет писать `"…".parse::<Email>()` — стандартный способ разбора. `impl Display` заставляет `{to}` печатать адрес.
4. `send_email` принимает `&Email` и `&Subject`. Она **ничего не проверяет**: если у неё есть `&Email`, адрес корректен. Проверка перенесена на границу (`main`), и доказательство путешествует вместе со значением.
5. `main` разбирает `"  ana@mail.org "` (с пробелами) и отправляет. Остальные три строки показывают ошибки.

```text
To: ana@mail.org
Subject: Welcome
Glad to see you
Err(MissingAt)
Err(EmptyPart)
Err(Empty)
```

Теперь путаница из раздела 1 — ошибка компиляции:

```rust
mod email {
    pub struct Email(String);
    impl Email {
        pub fn parse(text: &str) -> Option<Email> {
            text.contains('@').then(|| Email(text.to_string()))
        }
    }
}

use email::Email;

struct Subject(String);

fn send_email(to: &Email, subject: &Subject) {
    let _ = (to, subject);
}

fn main() {
    let to = Email::parse("ana@mail.org").unwrap();
    let subject = Subject("Welcome".to_string());
    send_email(&subject, &to);
}
```

```text
error[E0308]: arguments to this function are incorrect
  --> src/main.rs:21:5
   |
21 |     send_email(&subject, &to);
   |     ^^^^^^^^^^ --------  --- expected `&Subject`, found `&Email`
   |                |
   |                expected `&Email`, found `&Subject`
   |
note: function defined here
   ...

For more information about this error, try `rustc --explain E0308`.
error: could not compile `demo` (bin "demo") due to 1 previous error
```

Компилятор даже предлагает исправление (`swap these arguments` — поменять аргументы местами). А поскольку поле приватное, никто не может пропустить проверку:

```rust
mod email {
    pub struct Email(String);
}

fn main() {
    let sneaky = email::Email("not an address".to_string());
    let _ = sneaky;
}
```

```text
error[E0603]: tuple struct constructor `Email` is private
 --> src/main.rs:6:25
  |
2 |     pub struct Email(String);
  |                      ------ a constructor is private if any of the fields is private
...
6 |     let sneaky = email::Email("not an address".to_string());
  |                         ^^^^^ private tuple struct constructor
   ...

For more information about this error, try `rustc --explain E0603`.
error: could not compile `demo` (bin "demo") due to 1 previous error
```

> 🔒 **Безопасность:** разбирайте на границе, один раз. Данные от пользователей, из файлов и по сети недоверенные (Этап 10). Если всё внутри вашей программы использует `Email`, `Percent` и `OrderId` вместо сырого текста и чисел, целый класс ошибок «забыли проверить», некоторые из которых — дыры в безопасности, становится невозможным.

> ⚡ **Производительность:** у newtype в памяти тот же вид, что и у значения внутри. Дополнительный тип ничего не стоит во время работы; стоит только разовая проверка.

> 🧪 **Тестирование:** тестируйте умный конструктор на принимаемых *и* отвергаемых примерах, включая граничные случаи (пустая строка, только пробелы, `@` в начале). Остальной программе такие тесты не нужны, потому что тип делает плохие значения невозможными: компилятор и есть ваш тест.

## 3. Стандартные трейты для разбора

Вы уже использовали их выше. Практическое правило:

| Трейт | Для чего | Вызов |
|---|---|---|
| `FromStr` | Разбор из текста | `"ana@mail.org".parse::<Email>()` |
| `TryFrom<T>` | Преобразование, которое может не удаться, из другого типа | `Percent::try_from(150u8)` |
| `From<T>` | Преобразование, которое **не может** не удаться | `String::from("x")` |
| `Display` | Показ значения | `println!("{to}")` |

Если вы реализуете `TryFrom` для своего типа, вы бесплатно получаете и `.try_into()`. В Упражнении 2 используется `TryFrom`.

> 📏 **Соглашение:** называйте умный конструктор `parse`, когда он читает текст, и `new` или `try_new`, когда он принимает уже типизированные части. Не создавайте `Email::new`, который принимает всё. Если конструктор не может завершиться ошибкой, у типа, вероятно, нет инварианта, и хватит обычной структуры.

## 4. Делаем недопустимые состояния непредставимыми

Newtype защищает отдельные значения. Та же идея защищает целые структуры: проектируйте типы так, чтобы противоречия нельзя было записать. Посмотрите на список задач:

```rust
struct Task {
    title: String,
    done: bool,
    completed_on_day: Option<u32>,
}

fn describe(task: &Task) -> String {
    if task.done {
        match task.completed_on_day {
            Some(day) => format!("{}: done on day {day}", task.title),
            None => format!("{}: done on ??? (invalid!)", task.title),
        }
    } else if task.completed_on_day.is_some() {
        format!("{}: open, but has a completion day (invalid!)", task.title)
    } else {
        format!("{}: open", task.title)
    }
}

fn main() {
    let good = Task {
        title: "write".to_string(),
        done: true,
        completed_on_day: Some(12),
    };
    let bad = Task {
        title: "test".to_string(),
        done: true,
        completed_on_day: None,
    };
    println!("{}", describe(&good));
    println!("{}", describe(&bad));
}
```

**Что происходит:**

1. `done` говорит, закончена ли задача. `completed_on_day` говорит, когда. Два поля, четыре сочетания, и осмысленны только **два**: (выполнена, с днём) и (не выполнена, без дня).
2. `describe` должна обрабатывать два невозможных сочетания ветками «invalid!». Каждая другая функция, работающая с `Task`, должна делать то же самое или игнорировать проблему и надеяться.
3. В `main` небрежная строка строит сочетание `done: true, completed_on_day: None`.

```text
write: done on day 12
test: done on ??? (invalid!)
```

Корень ошибки в том, что тип допускает больше состояний, чем есть в реальности. Используйте перечисление, чтобы описать **реальные** состояния, и поместите данные туда, где они нужны:

```rust
enum Status {
    Open,
    Done { on_day: u32 },
}

struct Task {
    title: String,
    status: Status,
}

fn describe(task: &Task) -> String {
    match task.status {
        Status::Open => format!("{}: open", task.title),
        Status::Done { on_day } => format!("{}: done on day {on_day}", task.title),
    }
}

fn main() {
    let tasks = [
        Task {
            title: "write".to_string(),
            status: Status::Done { on_day: 12 },
        },
        Task {
            title: "test".to_string(),
            status: Status::Open,
        },
    ];
    for task in &tasks {
        println!("{}", describe(task));
    }
}
```

**Что происходит:**

1. У `Status` ровно два настоящих состояния. День существует **только** внутри `Done`, поэтому его нельзя забыть и он не может появиться у открытой задачи.
2. `describe` — это `match` с двумя ветками. Писать ветку «invalid» не нужно, потому что невалидного значения нет.
3. Если позже добавится третье состояние (`Cancelled`), компилятор перечислит все `match`, которые нужно обновить (Урок 08.3).

```text
write: done on day 12
test: open
```

Небольшой список проверки для таких проектов:

- Два `bool`, зависящие друг от друга? Скорее всего, нужно одно перечисление.
- `Option`, который должен быть `Some` ровно тогда, когда истинно что-то другое? Поместите данные в вариант перечисления, которому они нужны.
- `Vec`, который никогда не бывает пустым? Тип с первым элементом (`first`) и остатком (`rest`) (Упражнение 3).
- Строка с форматом? Newtype с умным конструктором.

## 5. Когда этот стиль стоит дороже, чем даёт

- **Слишком много типов.** Newtype для каждой `String` в скрипте на 50 строк — шум. Используйте доменные типы там, где путаница или плохое значение навредят: идентификаторы, деньги, единицы измерения, проверенный ввод пользователя.
- **Поле `pub` у newtype** уничтожает гарантию. `pub struct Email(pub String)` позволяет любому написать `Email("rubbish".into())`.
- **Шаблонный код преобразований.** Вам понадобятся `Display`, `FromStr`, иногда `AsRef<str>` и сравнения. Выводите (derive) всё, что можно (`#[derive(Debug, PartialEq, Clone)]`), а остальное добавляйте, когда это нужно вызывающему коду.
- **Меняющиеся инварианты.** Если правила корректного адреса меняются каждый месяц, жёсткая запись их в типе означает много правок. Держите типы для устойчивых правил.
- **Данные, которые приходят снаружи большими порциями.** Библиотеки, которые автоматически заполняют ваши типы (например, из JSON, JavaScript Object Notation — популярного текстового формата данных), могут обойти ваш конструктор, если не сказать им использовать его. Всегда перепроверяйте там, где данные входят (с `serde` вы познакомились в Уроке 05.11).

## Частые ошибки

**Ошибка 1: умный конструктор, который не проверяет.** `fn new(text: &str) -> Email { Email(text.to_string()) }` даёт *видимость* безопасности и ни капли самой безопасности.

**Ошибка 2: `unwrap()` на каждом разборе.** В настоящей программе плохое значение — ожидаемое событие. Передавайте ошибку наверх через `?` (Урок 04.10) или показывайте пользователю сообщение; `unwrap` — для тестов и случаев, которые действительно невозможны.

**Ошибка 3: «проверить» через `bool` и потом использовать сырое значение.** Если функция возвращает `bool`, а вызывающий продолжает работать с той же `String`, знание потеряно. Возвращайте разобранный тип.

**Ошибка 4: булевы флаги вместо перечисления.** `is_admin: bool, is_banned: bool, is_guest: bool` даёт восемь сочетаний; настоящих, вероятно, три. Сначала подумайте о реальных состояниях.

## Попробуйте сами

Вы поработаете по принципу «тесты вперёд» над новым доменным типом `Subject` (тема письма).

1. Создайте пакет и скопируйте всю первую программу раздела 2 (модуль `email`, `Subject`, `send_email` и `main`) в `src/main.rs`:

```bash
cd ~/projects/rust
cargo new code/p08-types
cd code/p08-types
```

2. Замените заглушку `Subject` (`struct Subject(String);`) настоящим типом с функцией `parse`, возвращающей `Result<Subject, SubjectError>`. Правила: текст обрезается по пробелам; пустой текст отвергается (`SubjectError::Empty`); больше 40 **символов** отвергается (`SubjectError::TooLong`). Выведите (derive) `Debug` и `PartialEq` для `Subject` и `SubjectError`, чтобы `assert_eq!` мог их сравнивать. Начните с написания этих тестов в конце файла, посмотрите, как они не компилируются, а затем проходят:

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn accepts_a_normal_subject() {
        assert_eq!(Subject::parse(" Hi "), Ok(Subject("Hi".to_string())));
    }

    #[test]
    fn rejects_an_empty_subject() {
        assert_eq!(Subject::parse("   "), Err(SubjectError::Empty));
    }

    #[test]
    fn rejects_a_subject_longer_than_40_characters() {
        assert_eq!(Subject::parse(&"a".repeat(41)), Err(SubjectError::TooLong));
    }

    #[test]
    fn counts_characters_not_bytes() {
        // 40 letters of two bytes each: 80 bytes, but only 40 characters
        assert!(Subject::parse(&"é".repeat(40)).is_ok());
    }
}
```

3. Измените `send_email` и `main`, чтобы использовать `Subject::parse` и обработать ошибку. Запустите `cargo test`:

```text
running 4 tests
test tests::accepts_a_normal_subject ... ok
test tests::counts_characters_not_bytes ... ok
test tests::rejects_a_subject_longer_than_40_characters ... ok
test tests::rejects_an_empty_subject ... ok

test result: ok. 4 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

4. Запустите `cargo run` и убедитесь, что плохая тема печатает сообщение, а не отправляет письмо. Завершите:

```bash
cargo fmt
cargo clippy
git add .
git commit -m "feat(p08): Email and Subject domain types"
```

**Что происходит:** тест `counts_characters_not_bytes` защищает от ошибки с `len()` из Урока 08.1: 40 букв вроде `é` занимают 80 байтов, но это 40 символов, и их нужно принять.

## Упражнения

### Упражнение 1 — Найдите недопустимые состояния (легко)

Для каждой структуры назовите одно сочетание значений, которое типы допускают, но которое не имеет смысла:

1. `struct Light { on: bool, brightness: u8 }`, где яркость 0 означает темноту.
2. `struct Door { open: bool, locked: bool }`
3. `struct Page { number: u32, total: u32 }`, номер страницы внутри документа.

<details>
<summary>Подсказка</summary>

Выпишите все сочетания полей, затем вычеркните те, которые в реальности невозможны.

</details>

<details>
<summary>Ответ</summary>

1. `on: true, brightness: 0` (включено, но темно) и `on: false, brightness: 200`.
2. `open: true, locked: true`: открытая дверь, которая заперта.
3. `number: 9, total: 3`: страница 9 из 3-страничного документа.

Почему это работает: для (1) и (2) перечисление вроде `enum Door { Open, Closed, Locked }` содержит только реальные состояния; для (3) умный конструктор `Page::new(number, total)` может проверить `number <= total`.

</details>

### Упражнение 2 — Тип `Percent` с тестами (средне)

`discounted` принимает любой `u8` как процент, поэтому `150` компилируется. (В отладочной сборке второй вызов паникует с `attempt to subtract with overflow`.) Создайте `Percent` с приватным значением и `TryFrom<u8>`, принимающий от 0 до 100. Измените `discounted` так, чтобы она принимала `Percent`, и **напишите тесты** для 0, 100 и 101.

```rust
fn discounted(price_cents: u64, percent: u8) -> u64 {
    price_cents - price_cents * u64::from(percent) / 100
}

fn main() {
    println!("{}", discounted(2000, 25));
    println!("{}", discounted(2000, 150)); // nonsense, but it compiles
}
```

<details>
<summary>Подсказка</summary>

`impl TryFrom<u8> for Percent { type Error = OutOfRange; fn try_from(value: u8) -> Result<Percent, OutOfRange> { ... } }`. В `main` вызов `Percent::try_from(25)` возвращает `Result`; продолжайте через `.map(|p| discounted(2000, p))`.

</details>

<details>
<summary>Ответ</summary>

```rust
#[derive(Debug, PartialEq, Clone, Copy)]
struct Percent(u8);

#[derive(Debug, PartialEq)]
struct OutOfRange(u8);

impl TryFrom<u8> for Percent {
    type Error = OutOfRange;

    fn try_from(value: u8) -> Result<Percent, OutOfRange> {
        if value <= 100 {
            Ok(Percent(value))
        } else {
            Err(OutOfRange(value))
        }
    }
}

fn discounted(price_cents: u64, percent: Percent) -> u64 {
    price_cents - price_cents * u64::from(percent.0) / 100
}

fn main() {
    println!("{:?}", Percent::try_from(25).map(|p| discounted(2000, p)));
    println!("{:?}", Percent::try_from(150).map(|p| discounted(2000, p)));
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn accepts_zero_to_one_hundred() {
        assert_eq!(Percent::try_from(0), Ok(Percent(0)));
        assert_eq!(Percent::try_from(100), Ok(Percent(100)));
    }

    #[test]
    fn rejects_values_above_one_hundred() {
        assert_eq!(Percent::try_from(101), Err(OutOfRange(101)));
    }

    #[test]
    fn discount_uses_the_percent() {
        assert_eq!(discounted(2000, Percent(25)), 1500);
    }
}
```

Почему это работает: `Percent` можно создать только через `try_from`, поэтому `discounted` никогда не видит значение больше 100. Ошибка несёт плохое значение для хорошего сообщения. (В настоящем проекте тип жил бы в собственном модуле, чтобы поле было действительно приватным; здесь, чтобы сократить пример, всё в одном файле.)

</details>

### Упражнение 3 — Список, который никогда не пуст (сложно)

`Vec<T>::iter().max()` возвращает `Option`, потому что вектор может быть пустым. Создайте `NonEmptyVec<T>`, хранящий элемент `first` и вектор `rest`, с методами `new(first)`, `from_vec(Vec<T>) -> Option<Self>`, `push` и методом `max`, возвращающим `&T` (не `Option`). **Напишите тесты** для `max` и для `from_vec` с пустым вектором.

<details>
<summary>Подсказка</summary>

`max` может начинаться с `&self.first` и сворачиваться (`fold`) по `self.rest`. Для `from_vec` удалите первый элемент через `items.remove(0)` после проверки, что вектор не пуст.

</details>

<details>
<summary>Ответ</summary>

```rust
struct NonEmptyVec<T> {
    first: T,
    rest: Vec<T>,
}

impl<T: Ord> NonEmptyVec<T> {
    fn new(first: T) -> Self {
        NonEmptyVec {
            first,
            rest: Vec::new(),
        }
    }

    fn from_vec(mut items: Vec<T>) -> Option<Self> {
        if items.is_empty() {
            return None;
        }
        let first = items.remove(0);
        Some(NonEmptyVec { first, rest: items })
    }

    fn push(&mut self, item: T) {
        self.rest.push(item);
    }

    // No Option here: a non-empty list always has a largest item.
    fn max(&self) -> &T {
        self.rest
            .iter()
            .fold(&self.first, |best, x| if x > best { x } else { best })
    }
}

fn main() {
    let mut list = NonEmptyVec::new(3);
    list.push(9);
    list.push(4);
    println!("{}", list.max());
    println!("{}", NonEmptyVec::<i32>::from_vec(vec![]).is_none());
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn max_of_one_item_is_that_item() {
        assert_eq!(*NonEmptyVec::new(5).max(), 5);
    }

    #[test]
    fn max_finds_the_largest() {
        let list = NonEmptyVec::from_vec(vec![2, 8, 3]).unwrap();
        assert_eq!(*list.max(), 8);
    }

    #[test]
    fn an_empty_vec_cannot_become_a_non_empty_one() {
        assert!(NonEmptyVec::<i32>::from_vec(Vec::new()).is_none());
    }
}
```

Почему это работает: сам тип гарантирует, что есть хотя бы один элемент, поэтому `max` не нужен `Option`. Случай «пусто» обрабатывается один раз, на границе (`from_vec`), и больше никогда. Это «parse, don't validate» для коллекции.

</details>

## Вопросы для самопроверки

1. Чем отличается проверка (validate) от разбора (parse)?

<details>
<summary>Ответ</summary>

Проверка возвращает ответ (`bool`) и оставляет прежний тип, поэтому знание теряется. Разбор возвращает новый, более строгий тип, поэтому знание путешествует вместе со значением.

</details>

2. Почему поле newtype с инвариантом должно быть приватным?

<details>
<summary>Ответ</summary>

Иначе любой код может создать тип напрямую и обойти проверку, и гарантия ничего не стоит.

</details>

3. Что значит сделать недопустимое состояние непредставимым?

<details>
<summary>Ответ</summary>

Спроектировать типы так, чтобы противоречивое сочетание значений нельзя было записать, например поместив данные в тот вариант перечисления, которому они нужны.

</details>

4. Стоит ли newtype скорости или памяти во время работы?

<details>
<summary>Ответ</summary>

Нет. У него то же представление, что и у внутреннего значения; стоит только разовая проверка.

</details>

5. Где нужно разбирать недоверенные данные?

<details>
<summary>Ответ</summary>

На границе, там, где они попадают в программу, один раз.

</details>

## Итоги

- Простые `String`, `u32` и `bool` не говорят, что значит значение, поэтому компилятор не ловит путаницу или забытые проверки.
- **Parse, don't validate**: проверяйте один раз на границе и возвращайте более строгий тип, несущий доказательство.
- **Умный конструктор** с приватным полем — единственный способ создать тип; для стандартных интерфейсов используйте `FromStr`, `TryFrom` и `Display`.
- Проектируйте перечисления так, чтобы каждый вариант хранил ровно те данные, которые верны в этом состоянии; тогда противоречивые сочетания записать нельзя.
- Newtype ничего не стоит во время работы, но используйте их там, где путаница или плохое значение навредят, а не для каждой строки.
- Поле `pub` или конструктор без проверки уничтожает гарантию.

## Источники и что почитать

- [Alexis King — Parse, don't validate](https://lexi-lambda.github.io/blog/2019/11/05/parse-don-t-validate/) — эссе, давшее этой идее название (примеры написаны на другом языке, но идея та же).
- [Rust API Guidelines — Type safety](https://rust-lang.github.io/api-guidelines/type-safety.html) — newtype для статических различий и собственные типы вместо аргументов `bool`.
- [Rust Design Patterns — Newtype](https://rust-unofficial.github.io/patterns/patterns/behavioural/newtype.html) — паттерн newtype и его компромиссы.
- [std::str::FromStr](https://doc.rust-lang.org/std/str/trait.FromStr.html) и [std::convert::TryFrom](https://doc.rust-lang.org/std/convert/trait.TryFrom.html) — стандартные трейты преобразования.

---

[🏠 Начало](../START-HERE.md) · [← 08.4 Композиция вместо наследования](04-composition-over-inheritance.md) · [📋 Этап 08](README.md) · [08.6 Паттерн «строитель» →](06-builder-pattern.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
