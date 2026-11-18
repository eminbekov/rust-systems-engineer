<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 08.8 Конечные автоматы и паттерн typestate](08-state-machines-and-typestate.md) · [📋 Этап 08](README.md) · [08.10 Антипаттерны и «запахи» кода →](10-anti-patterns-and-code-smells.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/08-software-design/09-strategy-and-dependency-injection.md) <!-- nav -->

# Урок 08.9 — Стратегия и внедрение зависимостей

> **Этап 08 — Проектирование ПО: принципы, паттерны и соглашения** · Урок 9 из 17 · ⏱ около 2,5 часа (2 занятия)

## Что вы узнаете

- Паттерн **стратегия** (strategy): замена одного алгоритма другим за общим интерфейсом.
- Четыре способа написать его в Rust (перечисление, обобщённый трейт, трейт-объект, замыкание) и как выбирать.
- **Внедрение зависимостей** (dependency injection): передача типу того, от чего он зависит, вместо того чтобы позволять ему создавать это самому.
- Как внедрение делает код тестируемым с помощью фейков без фреймворка для моков.
- Цену обеих идей и когда их пропускать.

## Перед началом

- Вы прошли [Урок 08.8 — Конечные автоматы и паттерн typestate](08-state-machines-and-typestate.md).
- Вы знаете трейты и обобщения ([Уроки 05.1 и 05.2](../05-rust-intermediate/01-generics.md)), замыкания с `Fn` ([Урок 05.4](../05-rust-intermediate/04-closures.md)) и `dyn Trait` со статической и динамической диспетчеризацией ([Урок 06.1](../06-rust-advanced/01-trait-objects-and-dynamic-dispatch.md)).
- Вы знаете принцип инверсии зависимостей из [Урока 08.3](03-solid-principles.md). Этот урок — его практическая форма.

## Новые слова

| Слово | Значение |
|---|---|
| **стратегия** (strategy) | Один из нескольких взаимозаменяемых способов выполнить одну и ту же работу (например, способы вычисления задержки). *Паттерн «стратегия»* позволяет коду выбирать или менять способ, не меняя код, который им пользуется. |
| **откат с задержкой** (backoff) | Время ожидания между повторными попытками неудавшейся операции. Ожидание обычно растёт с каждой попыткой, чтобы не засыпать запросами сервер, которому и так трудно. |
| **зависимость** (dependency) | Всё, что коду нужно для работы: другой тип, часы, отправитель почты, база данных. |
| **внедрение зависимостей** (dependency injection, DI) | Передача типу его зависимостей снаружи (через `new` или параметр), вместо того чтобы позволять ему создавать их самому. |
| **корень композиции** (composition root) | Единственное место программы (обычно `main`), которое решает, какие настоящие реализации используются, и соединяет их. |
| **тестовый двойник** (test double) | Замена настоящей зависимости, используемая в тесте. |
| **фейк** (fake) | Тестовый двойник с простой, но рабочей реализацией, например хранилище в памяти вместо базы данных. |
| **стаб** (stub) | Тестовый двойник, который возвращает фиксированные ответы (или ничего не делает). |
| **мок** (mock) | Тестовый двойник, который ещё и проверяет, как его вызывали (какие методы, с какими аргументами, сколько раз). |

## 1. Стратегия: замените алгоритм

Программа повторяет неудавшийся сетевой вызов. Перед повтором номер 0, 1, 2, ... она ждёт какое-то число **миллисекунд**. Сколько? Есть несколько разумных правил:

- **Постоянная**: всегда 200 мс.
- **Экспоненциальная**: 100 мс, затем 200, 400, 800... (удвоение).

Каждое правило — **стратегия**: способ выполнить одну и ту же работу «задержка для попытки n». Код, выполняющий повторы, не должен заботиться, какое правило используется. Вот самая простая версия на Rust, с перечислением:

```rust
enum Backoff {
    Constant(u64),
    Exponential { first: u64, factor: u64 },
}

impl Backoff {
    // How long to wait before retry number `attempt` (0, 1, 2, ...), in milliseconds.
    fn delay_ms(&self, attempt: u32) -> u64 {
        match self {
            Backoff::Constant(ms) => *ms,
            Backoff::Exponential { first, factor } => first * factor.pow(attempt),
        }
    }
}

fn schedule(strategy: &Backoff, attempts: u32) -> Vec<u64> {
    (0..attempts).map(|n| strategy.delay_ms(n)).collect()
}

fn main() {
    println!("{:?}", schedule(&Backoff::Constant(200), 4));
    println!(
        "{:?}",
        schedule(
            &Backoff::Exponential {
                first: 100,
                factor: 2
            },
            4
        )
    );
}
```

**Что происходит:**

1. У `Backoff` по одному варианту на стратегию. Данные каждой стратегии (константа или первая задержка и множитель) живут в её варианте.
2. `delay_ms` — это `match`. Каждая ветка — формула одной стратегии.
3. `schedule` — «пользователь» стратегии. Она запрашивает задержки попыток 0, 1, 2 и 3 и не знает формул.
4. `main` печатает расписание для двух стратегий.

```text
[200, 200, 200, 200]
[100, 200, 400, 800]
```

Для закрытого набора стратегий, которые вы пишете сами, это лучший выбор: просто, проверяется компилятором (новый вариант заставляет обновить каждый `match`, Урок 08.3) и быстро.

## 2. Стратегия через трейт или замыкание

Когда другой код (другой крейт, плагин, тест) должен добавлять стратегии или когда выбор происходит во время работы программы, используйте трейт. Тот же пример, тремя способами использования:

```rust
trait Backoff {
    fn delay_ms(&self, attempt: u32) -> u64;
}

struct Constant(u64);
struct Exponential {
    first: u64,
    factor: u64,
}

impl Backoff for Constant {
    fn delay_ms(&self, _attempt: u32) -> u64 {
        self.0
    }
}

impl Backoff for Exponential {
    fn delay_ms(&self, attempt: u32) -> u64 {
        self.first * self.factor.pow(attempt)
    }
}

// Generic: the compiler makes one copy of this function for every strategy type.
fn schedule(strategy: &impl Backoff, attempts: u32) -> Vec<u64> {
    (0..attempts).map(|n| strategy.delay_ms(n)).collect()
}

// Dynamic: one copy; the method is found at run time through a pointer.
fn schedule_dyn(strategy: &dyn Backoff, attempts: u32) -> Vec<u64> {
    (0..attempts).map(|n| strategy.delay_ms(n)).collect()
}

// Trait object: the strategy is chosen while the program runs.
fn from_name(name: &str) -> Option<Box<dyn Backoff>> {
    match name {
        "constant" => Some(Box::new(Constant(200))),
        "exponential" => Some(Box::new(Exponential {
            first: 100,
            factor: 2,
        })),
        _ => None,
    }
}

// Closure: the strategy is just a function.
fn schedule_with(strategy: impl Fn(u32) -> u64, attempts: u32) -> Vec<u64> {
    (0..attempts).map(strategy).collect()
}

fn main() {
    println!(
        "{:?}",
        schedule(
            &Exponential {
                first: 100,
                factor: 2
            },
            4
        )
    );

    for name in ["constant", "exponential", "unknown"] {
        match from_name(name) {
            Some(strategy) => println!("{name}: {:?}", schedule_dyn(strategy.as_ref(), 3)),
            None => println!("{name}: no such strategy"),
        }
    }

    println!(
        "{:?}",
        schedule_with(|attempt| 50 * u64::from(attempt + 1), 4)
    );
}
```

**Что происходит:**

1. `trait Backoff` — общий интерфейс. `Constant` и `Exponential` — две стратегии; каждая — маленькая структура со своими данными.
2. `schedule(&impl Backoff, ..)` — **обобщённая**: компилятор делает отдельную быструю копию функции для каждого типа, с которым вы её вызываете (статическая диспетчеризация, Урок 06.1).
3. `schedule_dyn(&dyn Backoff, ..)` принимает **трейт-объект**: одна копия функции, а метод находится во время работы через указатель (динамическая диспетчеризация).
4. `from_name` превращает текст (возможно, из файла настроек) в `Box<dyn Backoff>`. Здесь нужен именно трейт-объект: тип неизвестен, пока программа не запущена.
5. `schedule_with(impl Fn(u32) -> u64, ..)` принимает **замыкание**: стратегия — просто функция, без нового типа вообще.
6. `main` показывает все три.

```text
[100, 200, 400, 800]
constant: [200, 200, 200]
exponential: [100, 200, 400]
unknown: no such strategy
[50, 100, 150, 200]
```

Как выбирать:

| Форма | Выбирайте, когда | Цена |
|---|---|---|
| **Перечисление** | Вы знаете все стратегии; они ваши | Добавление стратегии правит перечисление |
| **Обобщение** (`impl Trait`) | Вызывающие выбирают стратегию в своём коде, и важна скорость | Копия функции на каждый тип; параметры типа расползаются к вызывающим |
| **Трейт-объект** (`dyn Trait`) | Стратегия выбирается во время работы (настройки, ввод пользователя) или нужен список разных стратегий | Вызов через указатель и обычно `Box` |
| **Замыкание** (`impl Fn`) | Стратегия — одна небольшая функция | Нет именованного типа; не идеально, если у стратегии несколько методов или нужно имя в ошибках |

> 📏 **Соглашение:** начните с перечисления. Переходите к трейту, когда посторонние должны добавлять стратегии. Используйте замыкание, когда «интерфейс» — одна функция. Это продолжает разговор об открытости/закрытости из Урока 08.3.

## 3. Внедрение зависимостей

**Зависимость** — всё, что вашему коду нужно извне. Вот функция регистрации. Она создаёт собственный список пользователей и сама отправляет почту:

```rust
use std::collections::HashMap;

fn register(email: &str) -> Result<(), String> {
    // The function creates everything it needs, itself.
    let mut users: HashMap<String, bool> = HashMap::new();
    if users.contains_key(email) {
        return Err(format!("{email} is already registered"));
    }
    users.insert(email.to_string(), true);
    println!("SENDING MAIL to {email}: welcome!");
    Ok(())
}

fn main() {
    println!("{:?}", register("ana@mail.org"));
}
```

```text
SENDING MAIL to ana@mail.org: welcome!
Ok(())
```

Она работает, но обратите внимание, что не так:

- Функция **создаёт** список пользователей, поэтому второй вызов снова начинает с пустого списка (ветка «уже зарегистрирован» никогда не выполнится), а настоящую базу данных нельзя подключить, не правя функцию.
- Почта «отправляется» через `println!` внутри функции. Тест не может спросить «было ли отправлено приветственное письмо на этот адрес?» Он мог бы лишь прочитать экран.
- Любое изменение отправки почты (SMTP, Simple Mail Transfer Protocol — протокол отправки почты; очередь; другой текст) правит логику регистрации. Две причины меняться в одной функции (Урок 08.3).

**Внедрение зависимостей** исправляет это одним правилом: **тип получает свои зависимости; он не создаёт их.** В Rust это значит: опишите нужное трейтом и принимайте его параметром `new`. Фреймворк не нужен.

```rust
use std::collections::HashSet;

trait Mailer {
    fn send(&self, to: &str, text: &str);
}

trait UserStore {
    fn contains(&self, email: &str) -> bool;
    fn add(&mut self, email: &str);
}

// The real parts (the "production" implementations).
struct ConsoleMailer;

impl Mailer for ConsoleMailer {
    fn send(&self, to: &str, text: &str) {
        println!("MAIL to {to}: {text}");
    }
}

#[derive(Default)]
struct MemoryStore {
    emails: HashSet<String>,
}

impl UserStore for MemoryStore {
    fn contains(&self, email: &str) -> bool {
        self.emails.contains(email)
    }
    fn add(&mut self, email: &str) {
        self.emails.insert(email.to_string());
    }
}

// The service receives what it needs through `new`.
struct SignupService<S: UserStore, M: Mailer> {
    store: S,
    mailer: M,
}

impl<S: UserStore, M: Mailer> SignupService<S, M> {
    fn new(store: S, mailer: M) -> Self {
        SignupService { store, mailer }
    }

    fn register(&mut self, email: &str) -> Result<(), String> {
        if self.store.contains(email) {
            return Err(format!("{email} is already registered"));
        }
        self.store.add(email);
        self.mailer.send(email, "welcome!");
        Ok(())
    }
}

// The composition root: the one place that picks the real parts.
fn main() {
    let mut service = SignupService::new(MemoryStore::default(), ConsoleMailer);
    println!("{:?}", service.register("ana@mail.org"));
    println!("{:?}", service.register("ana@mail.org"));
}

#[cfg(test)]
mod tests {
    use super::*;
    use std::cell::RefCell;
    use std::rc::Rc;

    // A fake that remembers what it was asked to send. Clones share the same list.
    #[derive(Clone, Default)]
    struct RecordingMailer {
        sent: Rc<RefCell<Vec<(String, String)>>>,
    }

    impl Mailer for RecordingMailer {
        fn send(&self, to: &str, text: &str) {
            self.sent
                .borrow_mut()
                .push((to.to_string(), text.to_string()));
        }
    }

    #[test]
    fn registering_sends_one_welcome_mail() {
        let mailer = RecordingMailer::default();
        let mut service = SignupService::new(MemoryStore::default(), mailer.clone());
        assert_eq!(service.register("bo@mail.org"), Ok(()));
        assert_eq!(
            *mailer.sent.borrow(),
            vec![("bo@mail.org".to_string(), "welcome!".to_string())]
        );
    }

    #[test]
    fn a_duplicate_is_rejected_and_sends_nothing() {
        let mailer = RecordingMailer::default();
        let mut service = SignupService::new(MemoryStore::default(), mailer.clone());
        service.register("bo@mail.org").unwrap();
        assert!(service.register("bo@mail.org").is_err());
        assert_eq!(mailer.sent.borrow().len(), 1);
    }
}
```

**Что происходит:**

1. `Mailer` и `UserStore` — трейты, которые нужны сервису. Они описывают, *что* сервису нужно, а не *как* это сделано (Урок 08.3, инверсия зависимостей).
2. `ConsoleMailer` и `MemoryStore` — простые настоящие реализации. В реальном приложении `MemoryStore` заменило бы хранилище в базе данных; сервис не изменился бы.
3. `SignupService<S, M>` хранит хранилище и почтовик, которые **внедряются** через `new`. Он никогда не создаёт их.
4. `register` содержит только правила регистрации: отклонить дубликаты, добавить пользователя, отправить приветственное письмо.
5. `main` — **корень композиции**: единственное место, которое выбирает `MemoryStore` и `ConsoleMailer` и соединяет их с сервисом.
6. Тесты строят тот же сервис с `RecordingMailer` — фейком, который сохраняет каждый вызов в списке, общем через `Rc<RefCell<..>>`. Затем тест читает список.

```text
MAIL to ana@mail.org: welcome!
Ok(())
Err("ana@mail.org is already registered")
```

```text
running 2 tests
test tests::a_duplicate_is_rejected_and_sends_nothing ... ok
test tests::registering_sends_one_welcome_mail ... ok

test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

Первый тест доказывает, что приветственное письмо отправлено ровно один раз на правильный адрес. Второй доказывает, что дубликат ничего не отправляет. Ни один тест не касался экрана или сети.

> 🧪 **Тестирование:** для большинства тестов достаточно фейкового почтовика. Словарь: у **фейка** простая рабочая реализация (хранилище в памяти, записывающий почтовик); **стаб** возвращает фиксированные значения или ничего не делает; **мок** ещё и проверяет вызовы. Крейты для моков вроде `mockall` (версия 0.15 на момент проверки этого курса) генерируют моки за вас, но начинайте с фейков, написанных вручную: они короче, чем вы думаете, и проверяют поведение, а не точный порядок вызовов, из-за чего тесты менее хрупки.

## 4. Обобщения или трейт-объекты для внедряемых частей?

`SignupService<S, M>` обобщён: компилятор знает точные типы, и вызовы прямые. Цена в том, что параметры типа попадают в имя типа. Любая функция, принимающая `SignupService`, тоже должна писать `SignupService<S, M>` (или быть обобщённой), и параметры расползаются по программе.

Альтернатива — трейт-объекты в полях: `store: Box<dyn UserStore>, mailer: Box<dyn Mailer>`. Тогда у типа нет параметров, настоящие части можно выбирать во время работы, а цена — вызов через указатель на каждый метод (обычно неважный рядом с сетевым вызовом). В Упражнении 3 вы преобразуете сервис.

| | Обобщённые параметры | Поля `Box<dyn Trait>` |
|---|---|---|
| Тип сервиса | `SignupService<S, M>` | `SignupService` |
| Выбор частей во время работы | Трудно | Легко |
| Скорость вызова | Прямой, можно встроить | Через указатель |
| Хорошо для | Библиотек, кода, чувствительного к скорости | Приложений, плагинов, выбора по настройкам |

> ⚡ **Производительность:** разница между статической и динамической диспетчеризацией — несколько наносекунд на вызов. Когда зависимость занимается вводом-выводом (I/O, Input/Output — общение с внешним миром: база данных, сеть), она теряется в шуме. Выбирайте форму, которая делает код проще, и измеряйте, прежде чем менять (Этап 19).

## 5. Когда НЕ использовать эти паттерны

- **Одна реализация, и это не ввод-вывод.** Чистой функции вроде `total_cents` трейт `Calculator` не нужен. Внедряйте то, что касается внешнего мира (время, файлы, сеть, случайность) или действительно варьируется.
- **Трейт на каждую структуру.** Если у трейта никогда не будет второй реализации, даже фейка, удалите его (YAGNI, You Aren't Gonna Need It — «тебе это не понадобится», Урок 08.1).
- **Обобщения повсюду.** Восемь параметров типа в структуре — знак использовать `dyn` на этой границе или сгруппировать связанные зависимости в один трейт.
- **«DI-контейнер» или фреймворк.** Большие библиотеки для этого существуют, но в Rust почти всегда хватает обычных параметров конструктора и одного корня композиции.
- **Скрытые глобальные значения.** Переменная `static` или функция, тайком читающая окружение, — скрытая зависимость. Передавайте значение (или трейт, который его поставляет).
- **Моки, копирующие реализацию.** Тест, проверяющий «метод A вызван, затем B, затем снова A», ломается при каждом рефакторинге. Проверяйте результат, который важен.

## Частые ошибки

**Ошибка 1: создание зависимости внутри.** `fn register(&self) { let mailer = SmtpMailer::new(); ... }`. Сервис нельзя протестировать, не отправив настоящую почту. Внедряйте.

**Ошибка 2: передача всего мира.** Сервис, принимающий гигантский `Context` со всем, легко вызвать и невозможно понять. Принимайте только нужные трейты (Урок 08.2, просите минимум).

**Ошибка 3: слишком раннее внедрение.** Не добавляйте трейт в первый же день, если вызывающий один и ввода-вывода нет; добавьте, когда этого потребует первый тест или вторая реализация. Тогда это небольшой рефакторинг (Урок 08.13).

**Ошибка 4: неправильный фейк.** Фейковое хранилище, ведущее себя иначе, чем настоящее (например, разрешающее дубликаты), заставляет тесты проходить, а продакшен — падать. Держите фейки простыми и проверяйте и их правила.

## Попробуйте сами

1. Создайте пакет и скопируйте пример внедрения зависимостей из раздела 3 (`SignupService` вместе с модулем `tests`) в `src/main.rs`:

```bash
cd ~/projects/rust
cargo new code/p08-strategy
cd code/p08-strategy
```

2. Запустите `cargo run`, затем `cargo test`. Два теста проходят.

3. В модуль `tests` добавьте стаб, который ничего не делает, и третий тест, использующий его:

```rust
struct SilentMailer;

impl Mailer for SilentMailer {
    fn send(&self, _to: &str, _text: &str) {}
}

#[test]
fn a_registered_email_is_stored() {
    let mut service = SignupService::new(MemoryStore::default(), SilentMailer);
    service.register("bo@mail.org").unwrap();
    assert!(service.store.contains("bo@mail.org"));
}
```

4. Запустите `cargo test`. Все три теста проходят:

```text
running 3 tests
test tests::a_duplicate_is_rejected_and_sends_nothing ... ok
test tests::a_registered_email_is_stored ... ok
test tests::registering_sends_one_welcome_mail ... ok

test result: ok. 3 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

5. Сохраните:

```bash
cargo fmt
cargo clippy
git add .
git commit -m "feat(p08): signup service with injected parts"
```

**Что происходит:** тест может читать приватное поле `service.store`, потому что модуль `tests` — дочерний по отношению к модулю, где определён тип. Обратите внимание, что новый тест вы построили, не трогая сервис: внедрение сделало тест дешёвым.

## Упражнения

### Упражнение 1 — Какая форма? (легко)

Выберите между перечислением, обобщённым трейтом, трейт-объектом и замыканием:

1. Функции повторов нужен способ решить, стоит ли повторять при ошибке. Правило — одна небольшая функция.
2. Приложение загружает имя «бэкенда хранения» из файла настроек: `"memory"` или `"disk"`.
3. Три правила скидок (нет, студент, пенсионер) все написаны вами, и никто другой не добавит новых.

<details>
<summary>Подсказка</summary>

Кто добавляет стратегии? Когда делается выбор? Сколько методов у интерфейса?

</details>

<details>
<summary>Ответ</summary>

1. Замыкание (`impl Fn(&Error) -> bool`): интерфейс — одна функция.
2. Трейт-объект (`Box<dyn Storage>`): выбор делается во время работы из текста.
3. Перечисление: закрытый набор, которым вы владеете, проверяемый компилятором.

Почему это работает: вопросы «кто добавляет стратегии» и «когда делается выбор» определяют форму.

</details>

### Упражнение 2 — Внедрите часы (средне)

Функция приветствия должна говорить `"Good morning"` до полудня и `"Good day"` начиная с полудня. Если она сама вызывает системные часы, её нельзя протестировать. Напишите трейт `Clock { fn hour(&self) -> u32 }`, настоящий `SystemClock`, функцию `greeting(&impl Clock)` и **тесты** с `FixedClock`.

<details>
<summary>Подсказка</summary>

Час по UTC (Coordinated Universal Time, всемирное координированное время) равен `(seconds_since_epoch / 3600) % 24`. Секунды получите через `SystemTime::now().duration_since(UNIX_EPOCH)`. Тестам нужны граничные часы 0, 11, 12 и 23.

</details>

<details>
<summary>Ответ</summary>

```rust
use std::time::{SystemTime, UNIX_EPOCH};

trait Clock {
    fn hour(&self) -> u32; // 0 to 23, in UTC
}

struct SystemClock;

impl Clock for SystemClock {
    fn hour(&self) -> u32 {
        let secs = SystemTime::now()
            .duration_since(UNIX_EPOCH)
            .unwrap()
            .as_secs();
        ((secs / 3600) % 24) as u32
    }
}

fn greeting(clock: &impl Clock) -> String {
    if clock.hour() < 12 {
        "Good morning".to_string()
    } else {
        "Good day".to_string()
    }
}

fn main() {
    // The result depends on the time you run it.
    println!("{}", greeting(&SystemClock));
}

#[cfg(test)]
mod tests {
    use super::*;

    struct FixedClock(u32);

    impl Clock for FixedClock {
        fn hour(&self) -> u32 {
            self.0
        }
    }

    #[test]
    fn before_noon_it_is_morning() {
        assert_eq!(greeting(&FixedClock(0)), "Good morning");
        assert_eq!(greeting(&FixedClock(11)), "Good morning");
    }

    #[test]
    fn from_noon_it_is_day() {
        assert_eq!(greeting(&FixedClock(12)), "Good day");
        assert_eq!(greeting(&FixedClock(23)), "Good day");
    }
}
```

Почему это работает: `greeting` получает часы, поэтому тест подставляет `FixedClock`, и результат больше не зависит от времени суток. Время, случайность и файловая система — самые частые кандидаты на внедрение.

</details>

### Упражнение 3 — Выберите почтовик во время работы (сложно)

Измените сервис так, чтобы его почтовик был `Box<dyn Mailer>`, и напишите `mailer_from_config(name) -> Option<Box<dyn Mailer>>`, понимающую `"console"` и `"silent"`. **Напишите тесты** для известных и неизвестных имён и один тест, что сервис работает с «тихим» почтовиком.

<details>
<summary>Подсказка</summary>

Поле сервиса становится `mailer: Box<dyn Mailer>`, а `new` принимает `Box<dyn Mailer>`. `main` вызывает `mailer_from_config("console").expect("unknown mailer")`.

</details>

<details>
<summary>Ответ</summary>

```rust
use std::collections::HashSet;

trait Mailer {
    fn send(&self, to: &str, text: &str);
}

struct ConsoleMailer;
struct SilentMailer;

impl Mailer for ConsoleMailer {
    fn send(&self, to: &str, text: &str) {
        println!("MAIL to {to}: {text}");
    }
}

impl Mailer for SilentMailer {
    fn send(&self, _to: &str, _text: &str) {}
}

// The name usually comes from a settings file or an environment variable.
fn mailer_from_config(name: &str) -> Option<Box<dyn Mailer>> {
    match name {
        "console" => Some(Box::new(ConsoleMailer)),
        "silent" => Some(Box::new(SilentMailer)),
        _ => None,
    }
}

struct SignupService {
    known: HashSet<String>,
    mailer: Box<dyn Mailer>,
}

impl SignupService {
    fn new(mailer: Box<dyn Mailer>) -> SignupService {
        SignupService {
            known: HashSet::new(),
            mailer,
        }
    }

    fn register(&mut self, email: &str) -> bool {
        let is_new = self.known.insert(email.to_string());
        if is_new {
            self.mailer.send(email, "welcome!");
        }
        is_new
    }
}

fn main() {
    let mailer = mailer_from_config("console").expect("unknown mailer");
    let mut service = SignupService::new(mailer);
    println!("{}", service.register("ana@mail.org"));
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn known_names_are_found_and_unknown_are_not() {
        assert!(mailer_from_config("console").is_some());
        assert!(mailer_from_config("silent").is_some());
        assert!(mailer_from_config("pigeon").is_none());
    }

    #[test]
    fn registering_twice_is_rejected_with_any_mailer() {
        let mut service = SignupService::new(mailer_from_config("silent").unwrap());
        assert!(service.register("a@b.c"));
        assert!(!service.register("a@b.c"));
    }
}
```

Почему это работает: у сервиса больше нет параметров типа, а выбор почтовика делается в одном месте (`mailer_from_config`), по данным. Эту форму нужно использовать, когда части определяют настройки.

</details>

## Вопросы для самопроверки

1. Что такое стратегия?

<details>
<summary>Ответ</summary>

Один из нескольких взаимозаменяемых способов выполнить одну работу, используемый через общий интерфейс.

</details>

2. Назовите четыре формы стратегии в Rust.

<details>
<summary>Ответ</summary>

Перечисление, обобщённый параметр с ограничением трейта, трейт-объект (`dyn Trait`) и замыкание.

</details>

3. Что такое внедрение зависимостей, одним предложением?

<details>
<summary>Ответ</summary>

Тип получает то, от чего он зависит (обычно в виде трейтов, через `new`), вместо того чтобы создавать это сам.

</details>

4. Что такое корень композиции?

<details>
<summary>Ответ</summary>

Единственное место, обычно `main`, которое выбирает настоящие реализации и соединяет их.

</details>

5. Почему плохо, когда тест проверяет точный порядок вызовов у мока?

<details>
<summary>Ответ</summary>

Он привязывает тест к реализации, поэтому безобидный рефакторинг ломает тест. Проверяйте результат, который важен.

</details>

## Итоги

- **Стратегия** — один из нескольких взаимозаменяемых способов выполнить работу. В Rust: перечисление (закрытый набор), обобщённый трейт (выбирают вызывающие, быстро), `dyn Trait` (выбор во время работы), замыкание (одна функция).
- **Внедрение зависимостей**: тип получает свои зависимости (как трейты) через `new` или параметры; `main` — корень композиции, который соединяет настоящие части.
- Внедрение упрощает тесты: используйте написанные вручную **фейки** и **стабы**; фреймворк не нужен.
- Обобщённые параметры быстры, но расползаются по типам; поля `Box<dyn Trait>` упрощают типы и допускают выбор во время работы.
- Внедряйте то, что касается внешнего мира (время, файлы, сеть, случайность) или действительно варьируется. Не добавляйте трейт для чистого кода или для единственной реализации без потребности в тестах.
- Пусть тесты проверяют результаты, а не точную последовательность вызовов.

## Источники и что почитать

- [Rust Design Patterns — Strategy](https://rust-unofficial.github.io/patterns/patterns/behavioural/strategy.html) — паттерн «стратегия» в Rust, с трейтами и с замыканиями.
- [The Rust Programming Language — 18.2 Using Trait Objects](https://doc.rust-lang.org/book/ch18-02-trait-objects.html) — трейт-объекты и динамическая диспетчеризация.
- [The Rust Programming Language — 13.1 Closures](https://doc.rust-lang.org/book/ch13-01-closures.html) — замыкания и трейты `Fn`.
- [Rust API Guidelines — Flexibility](https://rust-lang.github.io/api-guidelines/flexibility.html) — приём обобщённых параметров, чтобы вызывающий мог подставить то, что у него есть.
- [mockall на docs.rs](https://docs.rs/mockall/latest/mockall/) — крейт, генерирующий моки, на случай, когда фейков, написанных вручную, недостаточно.

---

[🏠 Начало](../START-HERE.md) · [← 08.8 Конечные автоматы и паттерн typestate](08-state-machines-and-typestate.md) · [📋 Этап 08](README.md) · [08.10 Антипаттерны и «запахи» кода →](10-anti-patterns-and-code-smells.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
