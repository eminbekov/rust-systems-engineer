<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 09.2 Сквозные тесты программ командной строки](02-testing-command-line-programs.md) · [📋 Этап 09](README.md) · [09.4 Тестирование на основе свойств →](04-property-based-testing.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/09-testing-debugging-benchmarking/03-test-doubles.md) <!-- nav -->

# Урок 09.3 — Тестовые двойники: фейки, стабы и моки

> **Этап 09 — Тестирование, отладка и бенчмарки** · Урок 3 из 14 · ⏱ около 135 минут

## Что вы узнаете

- Почему некоторый код трудно тестировать (он общается с часами, сетью, базой данных) и как сделать его лёгким для тестов: передавайте зависимость снаружи.
- Пять видов *тестовых двойников*: dummy, стаб, фейк, шпион (spy) и мок, и когда какой применять.
- Как сначала написать двойники вручную, на обычном Rust.
- Как генерировать моки крейтом `mockall` и чего это стоит.
- Почему стоит предпочитать фейки и стабы, а моки использовать умеренно.

## Перед началом

- Вы прошли [Урок 09.2 — Сквозные тесты программ командной строки](02-testing-command-line-programs.md).
- Вы знаете трейты и обобщения ([Урок 05.2](../05-rust-intermediate/02-traits.md), [Урок 05.1](../05-rust-intermediate/01-generics.md)) и встречали внедрение зависимостей ([Урок 08.9](../08-software-design/09-strategy-and-dependency-injection.md)): функция получает нужное ей как параметры, а не создаёт сама. Этот урок показывает, почему это делает код тестируемым, и строится на этом.
- Урок 09.1 говорил, что тест должен быть *детерминированным* и что часы, случайность и сеть передают как параметры. Теперь вы это сделаете.

## Новые слова

| Слово | Значение |
|---|---|
| **зависимость** (dependency) | Что-то, что вашему коду нужно снаружи для работы: часы, почтовый сервер, база данных. |
| **тестовый двойник** (test double) | Объект-заменитель, который в тесте подменяет настоящую зависимость, как каскадёр в кино. |
| **dummy** (пустышка) | Двойник, который лишь передаётся, чтобы заполнить параметр. По-настоящему он не используется. |
| **стаб** (stub) | Двойник, который возвращает фиксированные, заранее подготовленные ответы. |
| **фейк** (fake) | Двойник с настоящей, работающей, но упрощённой реализацией (например, «база данных» — это `HashMap` в памяти). |
| **шпион** (spy) | Двойник, который работает как обычно и ещё *записывает*, как его вызывали, чтобы тест потом мог посмотреть запись. |
| **мок** (mock) | Двойник, которому заранее сообщают, какие вызовы ожидать (и сколько раз), и который роняет тест, если реальность другая. |
| **проверка состояния** (state verification) | Проверка *результата* или *итогового состояния* после выполнения кода (например, «пользователь в хранилище»). |
| **проверка взаимодействия** (interaction verification) | Проверка того, *какие вызовы были сделаны* к зависимости (например, «`send` вызвали один раз с этой темой»). |
| **`mockall`** | Крейт, который автоматически генерирует моки по трейту. |
| **атрибутный макрос** (attribute macro) | Строка вроде `#[automock]` над элементом; она при компиляции генерирует дополнительный код для этого элемента. |
| **`RefCell`** | Тип Rust, позволяющий менять значение через общую ссылку `&T`; проверка выполняется во время работы (Урок 06.4). |
| **Unix-время** (Unix time) | Момент времени, записанный как число секунд с 1970-01-01 00:00 UTC (Coordinated Universal Time, всемирное координированное время — мировое эталонное время). |

## 1. Почему код трудно тестировать

Вот функция, которая регистрирует пользователя и отправляет приветственное письмо. Пробный период — 14 дней от «сейчас»:

```rust
// Hard to test. Do not write this.
fn sign_up(email: &str) -> Result<User, SignupError> {
    let now = std::time::SystemTime::now(); // the real clock
    let user = User { email: email.to_string(), trial_ends_at: now_plus_14_days(now) };
    real_mail_server::send(email, "Welcome", "…")?; // a real network call
    real_database::save(&user);                      // a real database
    Ok(user)
}
```

Как это протестировать? Часы меняются каждую секунду, поэтому результат нельзя сравнить с фиксированным числом. Каждый
тест отправлял бы настоящее письмо и писал бы в настоящую базу данных. А как проверить «почтовый сервер не работает»,
когда он работает? У кода три скрытые *зависимости*, и тест не управляет ни одной из них.

Лекарство — идея Урока 08.9: функция не создаёт свои зависимости, а **получает их как параметры**, описанные
трейтами. В продакшене вы передаёте настоящие, в тестах — **тестовые двойники**.

## 2. Тестируемый код

Создайте пакет. `mockall` нужен только тестам, поэтому он идёт в `[dev-dependencies]`:

```bash
cd ~/projects/rust
git switch main
git switch -c lesson/09-03-test-doubles
cargo new --lib code/p09-doubles
cd code/p09-doubles
cargo add --dev mockall@0.15
```

Замените `src/lib.rs` рабочим кодом (тесты будут дальше):

```rust
//! Signing up a new user: check the e-mail, store the user, send a welcome message.

use std::time::{SystemTime, UNIX_EPOCH};

/// Length of the free trial.
pub const TRIAL_DAYS: u64 = 14;
const SECONDS_PER_DAY: u64 = 86_400;

#[derive(Debug, Clone, PartialEq, Eq)]
pub struct User {
    pub email: String,
    /// Moment when the free trial ends, in seconds since 1970-01-01 (Unix time).
    pub trial_ends_at: u64,
}

#[derive(Debug, PartialEq, Eq)]
pub enum SignupError {
    InvalidEmail,
    AlreadyRegistered,
    MailFailed,
}

/// The sending of a message failed.
#[derive(Debug, PartialEq, Eq)]
pub struct MailError;

/// Sends messages to users.
#[cfg_attr(test, mockall::automock)]
pub trait Mailer {
    fn send(&self, to: &str, subject: &str, body: &str) -> Result<(), MailError>;
}

/// Tells the current time, in seconds since 1970-01-01.
#[cfg_attr(test, mockall::automock)]
pub trait Clock {
    fn now(&self) -> u64;
}

/// Remembers the users.
pub trait UserStore {
    fn exists(&self, email: &str) -> bool;
    fn save(&mut self, user: User);
}

/// The real clock.
pub struct SystemClock;

impl Clock for SystemClock {
    fn now(&self) -> u64 {
        SystemTime::now()
            .duration_since(UNIX_EPOCH)
            .expect("the system clock is before 1970")
            .as_secs()
    }
}

/// Registers a new user and sends the welcome message.
/// The user is stored only if the message was sent.
pub fn sign_up(
    email: &str,
    store: &mut impl UserStore,
    mailer: &impl Mailer,
    clock: &impl Clock,
) -> Result<User, SignupError> {
    if !email.contains('@') {
        return Err(SignupError::InvalidEmail);
    }
    if store.exists(email) {
        return Err(SignupError::AlreadyRegistered);
    }
    let user = User {
        email: email.to_string(),
        trial_ends_at: clock.now() + TRIAL_DAYS * SECONDS_PER_DAY,
    };
    let body = format!("Your free trial ends at {}.", user.trial_ends_at);
    mailer
        .send(email, "Welcome", &body)
        .map_err(|_| SignupError::MailFailed)?;
    store.save(user.clone());
    Ok(user)
}
```

**Что происходит:**

1. Три зависимости описаны тремя маленькими трейтами: `Mailer`, `Clock` и `UserStore`. Трейт говорит, *что* зависимость
   умеет, а не *как*.
2. `sign_up` принимает их как параметры типа `&impl Trait` («ссылка на любой тип, реализующий этот трейт», Урок 05.2).
   Хранилище — `&mut`, потому что `save` его изменяет. Функция не знает, общается ли она с настоящим почтовым
   сервером или с двойником.
3. `SystemClock` — настоящие часы. В боевом коде вы бы так же написали `SmtpMailer` (SMTP, Simple Mail Transfer Protocol, простой протокол передачи почты, — протокол, по которому почтовые серверы отправляют письма) и `DatabaseStore`.
   Для знакомства с тестами они нам не нужны.
4. Порядок внутри `sign_up` — это решение, которое мы будем тестировать: пользователь сохраняется **после** отправки
   письма, поэтому сбой почтового сервера не оставляет наполовину зарегистрированного пользователя.
5. `#[cfg_attr(test, mockall::automock)]` значит: «только при компиляции тестов добавить сюда `#[mockall::automock]`». Атрибут
   генерирует типы-моки `MockMailer` и `MockClock`; мы используем их в разделе 5. В обычных сборках
   атрибут исчезает, и `mockall` не нужен.

## 3. Пять видов тестовых двойников

Люди говорят «мок» обо всех них, но различия полезны. Вот они, от простейшего:

| Вид | Что делает | Типичное применение | В нашей программе |
|---|---|---|---|
| **dummy** | Ничего. Только заполняет параметр. | Код в этом тесте его не трогает. | `BrokenMailer`, переданный в тест, который заканчивается до отправки. |
| **стаб** | Возвращает подготовленные ответы. | Дать коду фиксированный вход из внешнего мира. | `FixedClock(1_000)`; `BrokenMailer` (всегда падает). |
| **фейк** | Рабочая упрощённая реализация. | Заменить тяжёлое (базу данных) простым (`HashMap`). | `InMemoryStore`. |
| **шпион** | Работает и записывает вызовы. | Проверить потом, *что было отправлено*. | `RecordingMailer`. |
| **мок** | Заранее знает, что должно случиться; иначе падает. | Проверить, что вызов происходит ровно так, как задано. | `MockMailer` из `mockall`. |

Отсюда два способа проверки. При **проверке состояния** вы смотрите на *результат* после работы кода
(сохранён ли пользователь?). При **проверке взаимодействия** вы смотрите на *разговор* с зависимостью
(вызывали ли `send`, сколько раз, с какими аргументами?). Фейки и стабы поддерживают проверку состояния; шпионы и моки
поддерживают проверку взаимодействия.

## 4. Двойники, написанные вручную

Библиотека для создания двойника не нужна. Двойник — просто тип, реализующий трейт. Добавьте тесты в
конец `src/lib.rs`:

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use std::cell::RefCell;
    use std::collections::HashMap;

    /// A stub: always answers the same.
    struct FixedClock(u64);

    impl Clock for FixedClock {
        fn now(&self) -> u64 {
            self.0
        }
    }

    /// A fake: a real, working, but simple implementation (memory instead of a database).
    #[derive(Default)]
    struct InMemoryStore {
        users: HashMap<String, User>,
    }

    impl UserStore for InMemoryStore {
        fn exists(&self, email: &str) -> bool {
            self.users.contains_key(email)
        }
        fn save(&mut self, user: User) {
            self.users.insert(user.email.clone(), user);
        }
    }

    /// A spy: remembers every message it was asked to send.
    #[derive(Default)]
    struct RecordingMailer {
        sent: RefCell<Vec<(String, String, String)>>,
    }

    impl Mailer for RecordingMailer {
        fn send(&self, to: &str, subject: &str, body: &str) -> Result<(), MailError> {
            self.sent
                .borrow_mut()
                .push((to.to_string(), subject.to_string(), body.to_string()));
            Ok(())
        }
    }

    /// A stub that makes the mail server "break".
    struct BrokenMailer;

    impl Mailer for BrokenMailer {
        fn send(&self, _to: &str, _subject: &str, _body: &str) -> Result<(), MailError> {
            Err(MailError)
        }
    }
```

**Что происходит:**

1. `FixedClock` — стаб: `now()` всегда возвращает число внутри себя. Время остановилось, поэтому тест может сравнивать с точным числом.
2. `InMemoryStore` — фейк. Он действительно хранит и находит пользователей, но в `HashMap`. Он настолько прост, что ему не нужен
   собственный тест, и он ведёт себя как настоящее хранилище: сохранили — и `exists` истинно.
3. `RecordingMailer` — шпион. Метод трейта принимает `&self`, но шпиону нужно менять свой список. `RefCell` позволяет
   это: `borrow_mut()` даёт временный доступ на запись, проверяемый во время работы программы (Урок 06.4).
4. `BrokenMailer` — стаб с одной задачей: всегда возвращать ошибку, чтобы мы могли проверить ситуацию, которую почти
   невозможно создать с настоящим почтовым сервером.

Теперь тесты, всё ещё внутри `mod tests`:

```rust
    #[test]
    fn a_new_user_is_saved_with_a_trial_of_14_days() {
        let mut store = InMemoryStore::default();
        let mailer = RecordingMailer::default();
        let clock = FixedClock(1_000);

        let user = sign_up("ada@example.com", &mut store, &mailer, &clock).unwrap();

        assert_eq!(user.trial_ends_at, 1_000 + 14 * 86_400);
        assert!(store.exists("ada@example.com"));
    }

    #[test]
    fn a_welcome_message_is_sent_once() {
        let mut store = InMemoryStore::default();
        let mailer = RecordingMailer::default();

        sign_up("ada@example.com", &mut store, &mailer, &FixedClock(0)).unwrap();

        let sent = mailer.sent.borrow();
        assert_eq!(sent.len(), 1);
        assert_eq!(sent[0].0, "ada@example.com");
        assert_eq!(sent[0].1, "Welcome");
        assert_eq!(sent[0].2, "Your free trial ends at 1209600.");
    }

    #[test]
    fn an_invalid_email_sends_nothing_and_saves_nothing() {
        let mut store = InMemoryStore::default();
        let mailer = RecordingMailer::default();

        let result = sign_up("not-an-email", &mut store, &mailer, &FixedClock(0));

        assert_eq!(result, Err(SignupError::InvalidEmail));
        assert!(mailer.sent.borrow().is_empty());
        assert!(store.users.is_empty());
    }

    #[test]
    fn signing_up_twice_is_refused_and_sends_no_second_message() {
        let mut store = InMemoryStore::default();
        let mailer = RecordingMailer::default();
        sign_up("ada@example.com", &mut store, &mailer, &FixedClock(0)).unwrap();

        let second = sign_up("ada@example.com", &mut store, &mailer, &FixedClock(0));

        assert_eq!(second, Err(SignupError::AlreadyRegistered));
        assert_eq!(mailer.sent.borrow().len(), 1);
    }

    #[test]
    fn when_the_mail_fails_the_user_is_not_saved() {
        let mut store = InMemoryStore::default();

        let result = sign_up("ada@example.com", &mut store, &BrokenMailer, &FixedClock(0));

        assert_eq!(result, Err(SignupError::MailFailed));
        assert!(!store.exists("ada@example.com"));
    }
}
```

Закройте модуль последней `}` и запустите `cargo test`. Проходят пять тестов. (Тесты с моками из раздела 5 ещё не написаны.)

**Что происходит:** каждый тест следует схеме Arrange, Act, Assert (Урок 09.1). Первый проверяет *состояние*: конец
пробного периода — фиксированное время плюс 14 дней, и фейковое хранилище теперь знает пользователя. Второй спрашивает у шпиона, что было
отправлено. Третий и четвёртый проверяют, что *ничего* не произошло там, где не должно: ни письма, ни сохранённого пользователя, ни второго
письма. Пятый доказывает правило из раздела 2: сбой почты не оставляет пользователя. Без `BrokenMailer` это вообще
нельзя было бы проверить.

> 🧪 **Тестирование:** тесты не знают, *как* `sign_up` устроена внутри. Если вы поменяете местами проверку `exists` и проверку
> адреса, все тесты по-прежнему проходят. Двойники позволяют тестировать **поведение**, а не реализацию.

## 5. Моки с `mockall`

Написание `RecordingMailer` вручную заняло десять строк. Для трейтов со многими методами ручной работы становится больше,
поэтому крейт `mockall` генерирует двойники за вас. Благодаря `#[cfg_attr(test, mockall::automock)]` типы `MockMailer` и
`MockClock` уже существуют в тестовых сборках. Добавьте два теста (внутрь `mod tests`, перед закрывающей скобкой):

```rust
    #[test]
    fn mock_the_welcome_message_has_the_right_recipient_and_subject() {
        let mut mailer = MockMailer::new();
        mailer
            .expect_send()
            .times(1)
            .withf(|to, subject, _body| to == "ada@example.com" && subject == "Welcome")
            .returning(|_, _, _| Ok(()));
        let mut clock = MockClock::new();
        clock.expect_now().return_const(1_000u64);
        let mut store = InMemoryStore::default();

        let user = sign_up("ada@example.com", &mut store, &mailer, &clock).unwrap();

        assert_eq!(user.trial_ends_at, 1_000 + 14 * 86_400);
    }

    #[test]
    fn mock_an_invalid_email_never_calls_the_mailer() {
        let mut mailer = MockMailer::new();
        mailer.expect_send().never();
        let mut store = InMemoryStore::default();

        let result = sign_up("not-an-email", &mut store, &mailer, &FixedClock(0));

        assert_eq!(result, Err(SignupError::InvalidEmail));
    }
```

```bash
cargo test
```

```text
running 7 tests
test tests::an_invalid_email_sends_nothing_and_saves_nothing ... ok
test tests::mock_an_invalid_email_never_calls_the_mailer ... ok
test tests::when_the_mail_fails_the_user_is_not_saved ... ok
test tests::a_new_user_is_saved_with_a_trial_of_14_days ... ok
test tests::signing_up_twice_is_refused_and_sends_no_second_message ... ok
test tests::mock_the_welcome_message_has_the_right_recipient_and_subject ... ok
test tests::a_welcome_message_is_sent_once ... ok

test result: ok. 7 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

**Что происходит:**

1. `MockMailer::new()` создаёт мок, у которого **ничего не разрешено**: любой вызов без ожидания вызывает панику.
2. `mailer.expect_send()` начинает *ожидание* для метода `send`. Каждый метод `foo` трейта получает
   `expect_foo`. Следующие вызовы уточняют ожидание:
   - `.times(1)`: метод должен быть вызван ровно один раз (`.never()` значит ноль раз; `.times(1..)` принимает диапазон);
   - `.withf(|to, subject, _body| …)`: проверка аргументов; ожидание подходит, только если замыкание возвращает `true`;
   - `.returning(|_, _, _| Ok(()))`: что мок отвечает при вызове; замыкание получает аргументы.
3. `clock.expect_now().return_const(1_000u64)` — сокращение для «всегда отвечай этим значением». Оно заставляет `MockClock` вести себя
   как стаб. Мок может играть любую роль.
4. Моки проверяются **автоматически при уничтожении** в конце теста: если `send` ожидался один раз
   и не был вызван, тест падает в этот момент, без всякого `assert`.
5. Во втором тесте ожидание — `never()`: тест доказывает, что для плохого адреса почтовый отправитель не вызывается никогда.
   `RecordingMailer` доказывал то же самое в `an_invalid_email_sends_nothing…`, с видимым `assert!(…is_empty())`.
   Оба способа правильны; мок ставит проверку *до* запуска кода.

### Когда мок падает

Если ожидание не выполнено, сообщение говорит, что ожидалось и что произошло. Замените `"Welcome"` в
`withf` на `"Hello"` и запустите тест:

```text
---- tests::mock_the_welcome_message_has_the_right_recipient_and_subject stdout ----
thread 'tests::mock_the_welcome_message_has_the_right_recipient_and_subject' (4708734) panicked at src/lib.rs:28:18:
MockMailer::send("ada@example.com", "Welcome", "Your free trial ends at 1210600."): No matching expectation found
```

**Что это значит:** код вызвал `send` с этими тремя аргументами, но ни одно ожидание их не приняло. Измените `times(1)` на
`times(2)`, и сообщение станет таким:

```text
MockMailer::send: Expectation(<function>) called 1 time(s) which is fewer than expected 2
```

Обе ошибки проявляются паникой с именем метода. (Номер строки указывает на код, сгенерированный для
трейта, и на вашем компьютере может отличаться.)

## 6. Какой двойник выбрать?

Короткое руководство — от самого предпочтительного к тому, что нужно только при необходимости:

1. **Используйте настоящий объект**, когда он быстрый, безопасный и детерминированный: чистая функция, `Vec`, строковая библиотека.
2. **Используйте стаб** для входов, которыми вы должны управлять: время, случайные числа, отказавший сервер.
3. **Используйте фейк** для тяжёлой зависимости с настоящим состоянием: база данных, файловая система, очередь сообщений. Тогда тест
   проверяет *состояние*, которое переживает рефакторинг.
4. **Используйте шпиона или мок**, когда результат — это *сам вызов*: отправка письма, списание с карты, запись строки в лог.
   Состояния для проверки не остаётся, и проверка вызова — единственный способ.

Правила, которые избавят вас от боли:

- **Подменяйте то, чем владеете.** Делайте двойник *своего* трейта (`Mailer`), а не типа из чужой библиотеки. Ваш собственный трейт
  остаётся стабильным; методы библиотеки могут меняться.
- **Не задавайте лишнего.** `times(1)` и `withf` на каждый аргумент превращают тест в копию кода. Когда код
  рефакторят, тесты падают, хотя поведение то же. Задавайте только то, что важно пользователю.
- **Двойники должны вести себя как настоящие.** Фейковое хранилище, которое допускало бы двух пользователей с одним адресом, когда настоящая
  база данных это запрещает, заставляет тесты проходить для неверного кода. Для важных зависимостей запускайте одни и те же проверки
  и на фейке, и на настоящей реализации.
- **Тестируйте и настоящее тоже.** Двойники заменяют зависимость в модульных тестах, но один тест на более высоком уровне
  (Урок 09.1) должен использовать настоящий почтовый сервер или базу данных в безопасной тестовой среде, потому что только это доказывает,
  что настоящие соединения работают.

> 📏 **Соглашение:** держите двойники маленькими и рядом с тестами. Когда один и тот же фейк нужен нескольким файлам тестов, перенесите его в
> `tests/common/mod.rs` (Урок 05.6).

> 🔒 **Безопасность:** двойники делают возможными и тесты безопасности. Стаб, который возвращает «неверный пароль» или «токен просрочен»,
> позволяет проверить вашу реакцию на атаки без каких-либо настоящих учётных записей.

## Частые ошибки

**1. Забытое ожидание у мока.** Вызов метода, для которого нет `expect_…`, вызывает панику с
`No matching expectation found`, как в разделе 5. Каждому методу, который вызовет код, нужно ожидание. Если
ожидание может быть выполнено любым способом, пишите `.returning(…)` и не пишите `.times(…)`.

**2. Слишком умный двойник.** У фейка со ста строками собственной логики есть собственные ошибки. Если двойнику нужен
тест, упростите дизайн.

**3. Тестирование двойника.** Проверка вроде `assert_eq!(fake_store.exists("x"), true)` сразу после `fake_store.save(…)`
в самом тесте доказывает лишь, что фейк работает. Ставьте проверку после вызова тестируемого кода.

**4. Подмена всего подряд.** Если функция принимает `Vec` и возвращает `Vec`, двойник не нужен. Двойники нужны для
*границ*: мест, где ваш код встречается с часами, сетью, диском и базой данных.

**5. Использование мока `mockall` в `tests/`.** `#[cfg_attr(test, …)]` активен только когда сама библиотека компилируется для
собственных тестов, поэтому `MockMailer` не существует в файле интеграционного теста. Для интеграционных тестов пишите двойник
вручную в файле теста.

## Попробуйте сами

1. Создайте пакет и наберите рабочий код. Запустите `cargo build`, чтобы убедиться, что он компилируется.
2. Добавьте двойники и пять написанных вручную тестов. Запустите `cargo test` и убедитесь, что проходят пять тестов.
3. Добавьте два теста с `mockall` и убедитесь, что проходят семь тестов.
4. Сломайте код нарочно: в `sign_up` вызовите `store.save(user.clone())` *до* `send`. Какой тест падает? Это
   `when_the_mail_fails_the_user_is_not_saved` с сообщением `assertion failed: !store.exists("ada@example.com")`.
   Верните код как было.
5. Запустите `cargo fmt --check && cargo clippy --all-targets -- -D warnings && cargo test`. Закоммитьте:
   `git add code/p09-doubles && git commit -m "Add p09-doubles: stubs, fakes, spies and mocks"`.

## Упражнения

### Упражнение 1 — Проверьте тело письма (легко)

Напишите тест с часами, зафиксированными на 1000, который точно проверяет тело приветственного письма:
`Your free trial ends at 1210600.`

<details>
<summary>Подсказка</summary>

Используйте `FixedClock(1_000)` и `RecordingMailer`. 1000 + 14 × 86 400 = 1 210 600. Читайте тело из `sent[0].2`.

</details>

<details>
<summary>Ответ</summary>

```rust
    #[test]
    fn the_message_body_contains_the_trial_end() {
        let mut store = InMemoryStore::default();
        let mailer = RecordingMailer::default();

        sign_up("ada@example.com", &mut store, &mailer, &FixedClock(1_000)).unwrap();

        assert_eq!(
            mailer.sent.borrow()[0].2,
            "Your free trial ends at 1210600."
        );
    }
```

Почему это работает: стаб-часы делают время точным, поэтому всё тело предсказуемо. С настоящими часами вы не смогли бы
написать этот тест.

</details>

### Упражнение 2 — Назовите двойник (средне)

Какой вид двойника каждый из этих (dummy, стаб, фейк, шпион, мок)?

1. `Clock`, который всегда возвращает `0`.
2. `UserStore` на основе `Vec<User>`, который действительно хранит и находит пользователей.
3. `Mailer`, который ничего не делает и возвращает `Ok(())`, переданный в тест о проверке адреса, где письмо никогда не отправляется.
4. `Mailer`, считающий вызовы в `Cell<u32>`, который тест читает в конце.
5. `MockMailer` с `expect_send().times(1)`.

<details>
<summary>Подсказка</summary>

Спросите: возвращает ли он подготовленные ответы (стаб)? Работает ли он по-настоящему, но проще (фейк)? Только ли затыкает
дыру (dummy)? Записывает ли (шпион)? Проверяет ли себя сам (мок)?

</details>

<details>
<summary>Ответ</summary>

1. Стаб (фиксированный ответ).
2. Фейк (настоящая, работающая, простая реализация).
3. Dummy (он заполняет параметр и не используется). Если бы тест его вызывал и полагался на `Ok(())`, это был бы стаб.
4. Шпион (он работает и записывает; тест смотрит запись).
5. Мок (ожидание объявлено заранее и проверяется при уничтожении мока).

Границы нестрогие: один и тот же тип может играть две роли. Названия помогают думать о том, *для чего* нужен двойник.

</details>

### Упражнение 3 — Почтовый сервер, который восстанавливается (сложно)

Напишите двойник `FlakyMailer`, который падает при первом вызове и успешно работает со второго. Затем напишите тест: первый
`sign_up` для `ada@example.com` возвращает `MailFailed` и ничего не сохраняет, второй `sign_up` для того же адреса
проходит успешно. Чем ценен этот тест?

<details>
<summary>Подсказка</summary>

Метод трейта принимает `&self`, поэтому держите счётчик вызовов в `Cell<u32>` (как `RefCell`, но для простых копируемых
значений: `get()` и `set()`). Для обоих вызовов используйте одно и то же хранилище и одни и те же часы.

</details>

<details>
<summary>Ответ</summary>

```rust
    struct FlakyMailer {
        calls: std::cell::Cell<u32>,
    }

    impl Mailer for FlakyMailer {
        fn send(&self, _to: &str, _subject: &str, _body: &str) -> Result<(), MailError> {
            self.calls.set(self.calls.get() + 1);
            if self.calls.get() == 1 { Err(MailError) } else { Ok(()) }
        }
    }

    #[test]
    fn a_failed_signup_can_be_retried() {
        let mut store = InMemoryStore::default();
        let mailer = FlakyMailer { calls: std::cell::Cell::new(0) };
        let clock = FixedClock(0);

        let first = sign_up("ada@example.com", &mut store, &mailer, &clock);
        assert_eq!(first, Err(SignupError::MailFailed));
        assert!(!store.exists("ada@example.com"));

        let second = sign_up("ada@example.com", &mut store, &mailer, &clock);
        assert!(second.is_ok());
        assert!(store.exists("ada@example.com"));
    }
```

Почему это работает: так как пользователь сохраняется только после отправки письма, неудачная попытка ничего не оставила, и
повторная попытка не отклоняется как `AlreadyRegistered`. Тест защищает именно это проектное решение. Такой сценарий
(сервер, который один раз падает, а потом восстанавливается) очень трудно воспроизвести с настоящим сервером и тривиально — с двойником.

</details>

## Вопросы для самопроверки

1. Почему `sign_up` принимает `Clock` как параметр, а не вызывает `SystemTime::now()`?

<details>
<summary>Ответ</summary>

Чтобы тест мог передать часы, возвращающие фиксированное время. С настоящими часами результат меняется каждую секунду, и
тест не мог бы сравнить его с точным значением.

</details>

2. В чём разница между стабом и моком?

<details>
<summary>Ответ</summary>

Стаб только возвращает подготовленные ответы; тесту неважно, как часто его вызывают. Моку заранее говорят, какие вызовы
ожидать, и тест падает, если реальность другая.

</details>

3. Что такое фейк? Приведите пример.

<details>
<summary>Ответ</summary>

Рабочая, но упрощённая реализация, например хранилище пользователей, которое держит их в `HashMap`, а не в базе данных.

</details>

4. Когда правильный выбор — шпион или мок, а не фейк?

<details>
<summary>Ответ</summary>

Когда результат — сам вызов и состояния для проверки не остаётся: отправка письма, списание с карты, запись строки
в лог.

</details>

5. Почему «подменяйте то, чем владеете» — хороший совет?

<details>
<summary>Ответ</summary>

Трейт, который вы определили, остаётся стабильным и описывает то, что нужно *вашему* коду. Подмена типа из чужой библиотеки привязывает тесты к
деталям этой библиотеки, и они ломаются, когда она меняется.

</details>

6. Что произойдёт, если код вызовет метод мока `mockall`, для которого нет ожидания?

<details>
<summary>Ответ</summary>

Мок паникует с `No matching expectation found`, и тест падает.

</details>

## Итоги

- Код, который сам создаёт часы, сеть или базу данных, трудно тестировать. Передавайте зависимости как параметры, описанные трейтами.
- Тестовый двойник заменяет зависимость в тесте. Виды: dummy (заполняет параметр), стаб (фиксированные ответы), фейк (простой, но
  рабочий), шпион (записывает вызовы), мок (ожидает вызовы и проверяет себя).
- Проверка состояния проверяет результаты и устойчива к рефакторингу; проверка взаимодействия проверяет вызовы и
  нужна, когда результат — сам вызов.
- Двойники — обычные типы Rust, реализующие трейт; сначала пишите их вручную.
- `mockall` генерирует моки: `#[cfg_attr(test, mockall::automock)]`, затем `expect_<method>().times(n).withf(…).returning(…)`.
- Предпочитайте настоящие объекты, затем стабы и фейки; моки используйте умеренно; подменяйте то, чем владеете; не задавайте лишнего.

## Источники и что почитать

- [mockall on docs.rs](https://docs.rs/mockall/0.15.0/mockall/) — полное руководство по ожиданиям, матчерам, последовательностям и обобщённым трейтам.
- [The Rust Programming Language — 10.2 Traits](https://doc.rust-lang.org/book/ch10-02-traits.html) — трейты и параметры `impl Trait`, инструменты, на которых держится любой двойник.
- [std::cell::RefCell](https://doc.rust-lang.org/std/cell/struct.RefCell.html) и [std::cell::Cell](https://doc.rust-lang.org/std/cell/struct.Cell.html) — внутренняя изменяемость, которую использует шпион.
- [Martin Fowler — Mocks Aren't Stubs](https://martinfowler.com/articles/mocksArentStubs.html) — классическая статья о проверке состояния и проверке взаимодействия.
- [Martin Fowler — TestDouble](https://martinfowler.com/bliki/TestDouble.html) — короткое определение словаря, использованного в этом уроке.

---

[🏠 Начало](../START-HERE.md) · [← 09.2 Сквозные тесты программ командной строки](02-testing-command-line-programs.md) · [📋 Этап 09](README.md) · [09.4 Тестирование на основе свойств →](04-property-based-testing.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
