<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 10.2 Моделирование угроз по STRIDE](02-threat-modeling-with-stride.md) · [📋 Этап 10](README.md) · [10.4 Проверка ввода: никогда не доверяйте вводу →](04-input-validation.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/10-security-fundamentals/03-the-owasp-top-10.md) <!-- nav -->

# Урок 10.3 — OWASP Top 10

> **Этап 10 — Основы безопасности** · Урок 3 из 15 · ⏱ около 135 минут

## Что вы узнаете

- Что такое OWASP и что такое список «Top 10» (и чем он не является).
- Десять категорий риска издания 2025 года простыми словами и какой урок курса охватывает каждую.
- Четыре категории на практике с небольшими программами на Rust: нарушенный контроль доступа, небезопасная конфигурация, неправильная обработка ошибок (fail open) и слабое логирование.
- Как превратить категорию в вопрос-чек-лист для вашего собственного кода.

## Перед началом

- Вы прошли [Урок 10.2 — Моделирование угроз по STRIDE](02-threat-modeling-with-stride.md).
- Вы знаете обработку ошибок ([Урок 04.10](../04-structs-enums-collections-errors/10-error-handling.md)) и `Option` ([Урок 04.4](../04-structs-enums-collections-errors/04-option-and-match.md)).
- Создайте пакет: `cargo new code/p10-owasp`, затем `mkdir src/bin` и `rm src/main.rs`. Каждая программа ниже — один файл в `src/bin/`.

## Новые слова

| Слово | Значение |
|---|---|
| **OWASP** (Open Worldwide Application Security Project) | Некоммерческое сообщество, которое публикует бесплатные руководства, инструменты и списки по безопасности приложений. |
| **OWASP Top 10** | Список десяти самых серьёзных видов риска для веб-приложений, составленный по реальным данным и обновляемый раз в несколько лет. В этом уроке используется издание 2025 года. |
| **контроль доступа** (access control) | Правила, решающие, кто что может делать (читать эту заметку, удалить того пользователя). Также называется авторизацией (Урок 10.7). |
| **IDOR** (Insecure Direct Object Reference, небезопасная прямая ссылка на объект) | Ошибка, когда программа берёт идентификатор из запроса (`/notes/42`) и использует его, не проверив, что этот пользователь может видеть этот объект. |
| **SSRF** (Server-Side Request Forgery, подделка запросов на стороне сервера) | Атака, при которой атакующий заставляет ваш сервер отправить запрос по адресу, выбранному атакующим, часто внутреннему, до которого атакующий не может добраться напрямую. |
| **неправильная конфигурация** (misconfiguration) | Неверная или небезопасная настройка: пароль по умолчанию, режим отладки в рабочей среде, служба, открытая всей сети. |
| **fail open / fail closed** | Что делает проверка, когда не может принять решение. *Fail open* разрешает действие, *fail closed* отказывает. Проверки безопасности должны отказывать (fail closed). |
| **журнал аудита** (audit log) | Запись важных событий (входы, отклонённые запросы, изменения), которую хранят, чтобы позже можно было найти атаки и ошибки. |
| **внедрение в журнал** (log injection) | Атака, при которой текст, введённый атакующим, содержит перевод строки и поэтому создаёт поддельные строки в файле журнала. |
| **подстановка учётных данных** (credential stuffing) | Попытки применить пароли, украденные с одного сайта, на множестве других сайтов, потому что многие люди повторно используют пароли. |
| **SQL** (Structured Query Language, язык структурированных запросов) | Язык, на котором общаются с большинством баз данных (Урок 10.5). |

## 1. Что такое OWASP Top 10

OWASP — сообщество, которое публикует бесплатные знания по безопасности. Самый известный его документ —
**Top 10**: список десяти категорий слабых мест, которые наносят самый серьёзный ущерб веб-приложениям. OWASP
собирает данные из множества реальных приложений и от специалистов по безопасности и раз в несколько лет
публикует новое издание. Здесь используется издание **OWASP Top 10:2025**.

Три вещи, которые нужно знать, прежде чем читать список:

1. Это список **категорий**, а не отдельных ошибок. «Внедрение» (injection) содержит десятки разных атак.
2. Это документ для **осведомлённости**. Он говорит, где искать; это не полный чек-лист всего, что может
   пойти не так. Программа, избегающая всех десяти, всё равно может иметь другие ошибки.
3. Он про веб-приложения, но почти всё применимо к любой программе, которая принимает ввод снаружи:
   инструментам командной строки, сервисам, библиотекам.

## 2. Десять категорий 2025 года

| ID | Категория | Простыми словами | Где этот курс это изучает |
|---|---|---|---|
| A01 | **Broken Access Control** (нарушенный контроль доступа) | Пользователи могут делать то, чего не должны (читать данные другого пользователя, открывать страницы администратора). В эту категорию входит и SSRF. | Раздел 3 здесь; [10.7](07-authorization-sessions-and-tokens.md) |
| A02 | **Security Misconfiguration** (небезопасная конфигурация) | Небезопасные настройки: пароли по умолчанию, включённая отладка, слишком много открытых портов, подробные страницы ошибок. | Раздел 4 здесь; [10.8](08-secrets-management.md) |
| A03 | **Software Supply Chain Failures** (сбои цепочки поставок ПО) | Проблемы в коде и инструментах, от которых вы зависите: уязвимая или вредоносная библиотека, скомпрометированная система сборки или обновления. | [10.13](13-supply-chain-security.md) |
| A04 | **Cryptographic Failures** (сбои криптографии) | Нет шифрования там, где оно нужно, слабые алгоритмы, плохая работа с ключами, неправильное хранение паролей. | [10.6](06-authentication-and-password-hashing.md), [10.9](09-cryptography-hashes-macs-and-randomness.md), [10.10](10-cryptography-encryption-and-signatures.md), [10.11](11-tls-and-certificates.md) |
| A05 | **Injection** (внедрение) | Ввод принимается как часть команды или запроса (SQL, оболочка, путь к файлу) и меняет их смысл. | [10.4](04-input-validation.md), [10.5](05-injection-attacks.md) |
| A06 | **Insecure Design** (небезопасное проектирование) | В самом проекте не хватает защиты, поэтому никакое аккуратное программирование этого не исправит. | [10.2](02-threat-modeling-with-stride.md) |
| A07 | **Authentication Failures** (сбои аутентификации) | Разрешены слабые пароли, нет защиты от подбора, сломанные сессии. | [10.6](06-authentication-and-password-hashing.md), [10.7](07-authorization-sessions-and-tokens.md) |
| A08 | **Software or Data Integrity Failures** (сбои целостности ПО или данных) | Доверие коду или данным без проверки подлинности: неподписанные обновления, небезопасная десериализация. | [10.10](10-cryptography-encryption-and-signatures.md), [10.13](13-supply-chain-security.md) |
| A09 | **Security Logging and Alerting Failures** (сбои журналирования и оповещения) | Важные события не записываются или никого не уведомляют, и атаки остаются незамеченными. | Раздел 6 здесь |
| A10 | **Mishandling of Exceptional Conditions** (неправильная обработка исключительных ситуаций) | Ошибки и необычные ситуации обрабатываются плохо: падения, проверки с fail open, утечка деталей ошибок. | Раздел 5 здесь; [10.4](04-input-validation.md) |

По сравнению с изданием 2021 года Security Misconfiguration поднялась на второе место; прежняя категория
«Vulnerable and Outdated Components» выросла в более широкую категорию цепочки поставок; Server-Side Request
Forgery стала частью Broken Access Control; а Mishandling of Exceptional Conditions — новая категория.

> 💡 **Совет:** используйте список как вопросы при код-ревью: «Кому разрешено это вызывать?» (A01), «Какое
> значение по умолчанию?» (A02), «От чего мы зависим?» (A03), «Что произойдёт, когда это не сработает?» (A10).

## 3. A01 Broken Access Control: число в адресе

Представьте приложение для заметок. Вы вошли как `ana`, и ваша заметка находится по адресу `/notes/1`. Вы
меняете `1` на `2` в адресной строке. Если программа возвращает заметку, в ней есть **IDOR**: она проверила,
что вы вошли, но не проверила, что заметка 2 ваша. Это самая частая серьёзная ошибка в вебе, и хитрая атака
для неё не нужна.

Сохраните как `src/bin/access.rs`. Мы имитируем приложение списком в памяти:

```rust
struct Note {
    id: u32,
    owner: &'static str,
    text: &'static str,
}

/// Broken: anyone who knows (or guesses) an id gets the note.
fn get_note_unchecked(notes: &[Note], id: u32) -> Option<&Note> {
    notes.iter().find(|n| n.id == id)
}

/// Fixed: the note must exist AND belong to the logged-in user.
/// Both failures give the same answer, so an attacker cannot learn which ids exist.
fn get_note<'a>(notes: &'a [Note], user: &str, id: u32) -> Option<&'a Note> {
    notes.iter().find(|n| n.id == id && n.owner == user)
}

fn main() {
    let notes = [
        Note {
            id: 1,
            owner: "ana",
            text: "my shopping list",
        },
        Note {
            id: 2,
            owner: "bob",
            text: "bob's private diary",
        },
    ];

    // "ana" is logged in and changes the number 1 to 2, like editing /notes/2 in a web address.
    let found = get_note_unchecked(&notes, 2);
    println!("unchecked: {:?}", found.map(|n| n.text));

    let found = get_note(&notes, "ana", 2);
    println!("checked:   {:?}", found.map(|n| n.text));
    let found = get_note(&notes, "ana", 1);
    println!("own note:  {:?}", found.map(|n| n.text));
}
```

**Что происходит:**

1. У `Note` есть `id`, `owner` и `text`. Поля — `&'static str`, текст, который живёт всю программу.
2. `get_note_unchecked` смотрит только на id. Это сломанная версия.
3. `get_note` находит заметку, только если id совпадает **и** владелец — текущий пользователь. Если не
   выполняется любая из частей, возвращается `None`, поэтому «не существует» и «существует, но не ваша»
   выглядят абсолютно одинаково. Атакующий даже не может узнать, какие id используются.
4. В `main` пользователь `ana` запрашивает заметку 2 сначала сломанной функцией, затем исправленной, и в
   конце свою заметку 1.

```text
unchecked: Some("bob's private diary")
checked:   None
own note:  Some("my shopping list")
```

Сломанная функция отдаёт `ana` дневник Боба. Тесты доказывают исправление и не дают ему сломаться снова.
Добавьте их в конец того же файла:

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn a_user_cannot_read_a_note_of_another_user() {
        let notes = [Note {
            id: 2,
            owner: "bob",
            text: "private",
        }];
        assert!(get_note(&notes, "ana", 2).is_none());
        assert!(get_note(&notes, "bob", 2).is_some());
    }

    #[test]
    fn missing_and_forbidden_look_the_same() {
        let notes = [Note {
            id: 2,
            owner: "bob",
            text: "private",
        }];
        assert_eq!(
            get_note(&notes, "ana", 2).map(|n| n.id),
            get_note(&notes, "ana", 99).map(|n| n.id)
        );
    }
}
```

```bash
cargo test --bin access
```

```text
running 2 tests
test tests::missing_and_forbidden_look_the_same ... ok
test tests::a_user_cannot_read_a_note_of_another_user ... ok

test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

> 🔒 **Безопасность:** проверяйте доступ **на стороне сервера, при каждом запросе, для каждого объекта**. Не
> полагайтесь на скрытые ссылки, на случайно выглядящие id (они помогают, но проверкой не являются) или на
> пользовательский интерфейс. Запрещайте по умолчанию: доступ разрешён, только когда правило говорит «да».

**SSRF в одном абзаце.** Некоторые программы загружают URL (Uniform Resource Locator, веб-адрес), который указал пользователь (предпросмотр
ссылки, импорт картинки). Атакующий подставляет внутренний адрес вроде `http://127.0.0.1:8080/admin`, и ваш
сервер послушно запрашивает его изнутри, где межсетевой экран ему доверяет. Защита — та же идея: разрешайте
только ожидаемые хосты (список разрешённого, [Урок 10.4](04-input-validation.md)) и никогда не следуйте
перенаправлениям вслепую.

## 4. A02 Security Misconfiguration: безопасно по умолчанию

Многие взломы вообще не используют ошибок в коде: служба работала с паролем по умолчанию, с включённой
отладкой или слушала все сетевые интерфейсы. Лучшая защита — конфигурация, **безопасная по умолчанию**,
которая **отказывается запускаться**, если задано что-то опасное. Сохраните `src/bin/config.rs`:

```rust
use std::net::{IpAddr, Ipv4Addr};

#[derive(Debug)]
struct Config {
    bind: IpAddr,
    debug: bool,
    admin_password: Option<String>,
}

/// Secure by default: local only, no debug output, and NO default password.
impl Default for Config {
    fn default() -> Self {
        Config {
            bind: IpAddr::V4(Ipv4Addr::LOCALHOST),
            debug: false,
            admin_password: None,
        }
    }
}

impl Config {
    /// Refuse to start with a configuration that is known to be unsafe.
    fn validate(&self) -> Result<(), String> {
        match &self.admin_password {
            None => return Err(String::from("admin password is not set")),
            Some(p) if p.len() < 12 => {
                return Err(String::from(
                    "admin password must have at least 12 characters",
                ));
            }
            Some(_) => {}
        }
        if self.debug && !self.bind.is_loopback() {
            return Err(String::from(
                "debug mode is allowed only on a local address",
            ));
        }
        Ok(())
    }
}

fn main() {
    let config = Config::default();
    println!(
        "default config: bind={} debug={}",
        config.bind, config.debug
    );
    println!("validate: {:?}", config.validate());

    let risky = Config {
        bind: IpAddr::V4(Ipv4Addr::UNSPECIFIED), // 0.0.0.0: reachable from the whole network
        debug: true,
        admin_password: Some(String::from("a-long-enough-password")),
    };
    println!("risky config: {:?}", risky.validate());
}
```

**Что происходит:**

1. `Default` даёт самые безопасные значения: слушать только `127.0.0.1` (`LOCALHOST`, только этот
   компьютер), отладка выключена и **никакого пароля администратора вообще**. Нет встроенного «admin/admin»,
   который можно забыть.
2. `validate` возвращает текст ошибки для каждого небезопасного случая: нет пароля, короткий пароль или режим
   отладки на нелокальном адресе. `is_loopback()` истинно для адресов только для этого компьютера.
3. `Some(p) if p.len() < 12` — *охранное выражение* (match guard): эта ветка срабатывает, только когда
   дополнительное `if` истинно. (`len()` считает байты, что для простых английских букв то же самое, что
   символы; [Урок 10.4](04-input-validation.md) обращается с текстом аккуратнее.)
4. `main` пробует конфигурацию по умолчанию и рискованную (`0.0.0.0` означает «все сетевые интерфейсы», то
   есть режим отладки был бы доступен снаружи). Обе отклоняются.

```text
default config: bind=127.0.0.1 debug=false
validate: Err("admin password is not set")
risky config: Err("debug mode is allowed only on a local address")
```

> 📏 **Соглашение:** программа, которая запускается с небезопасной конфигурацией, — ошибка, ждущая своего
> часа. Проверяйте конфигурацию при запуске и завершайтесь с понятным сообщением («fail fast»). Подход
> twelve-factor к конфигурации разобран в [Уроке 08.15](../08-software-design/15-the-twelve-factor-app.md).

## 5. A10 Mishandling of Exceptional Conditions: отказ в закрытом состоянии

Любая проверка безопасности может сама дать сбой: база данных недоступна, файла нет, сеть не ответила вовремя.
То, что проверка делает **тогда**, решает, безопасна ли она. Сохраните `src/bin/failopen.rs`:

```rust
#[derive(Debug, PartialEq)]
enum Role {
    Admin,
    User,
}

#[derive(Debug)]
struct LookupError;

/// Pretend database: it is "down" for the user "carol".
fn lookup_role(user: &str) -> Result<Role, LookupError> {
    match user {
        "ana" => Ok(Role::Admin),
        "bob" => Ok(Role::User),
        _ => Err(LookupError),
    }
}

/// Broken: when the check itself fails, it lets the user in (fails open).
fn is_admin_open(user: &str) -> bool {
    match lookup_role(user) {
        Ok(role) => role == Role::Admin,
        Err(_) => true, // "do not block people because of our own error"
    }
}

/// Fixed: when the check fails, the answer is "no" (fails closed).
fn is_admin(user: &str) -> bool {
    match lookup_role(user) {
        Ok(role) => role == Role::Admin,
        Err(_) => false,
    }
}

fn main() {
    for user in ["ana", "bob", "carol"] {
        println!(
            "{user:<6} open: {:<5} closed: {}",
            is_admin_open(user),
            is_admin(user)
        );
    }
}
```

**Что происходит:**

1. `lookup_role` изображает базу данных ролей. Она знает `ana` и `bob`; для всех остальных возвращает
   `Err`, как будто база не смогла ответить.
2. `is_admin_open` обрабатывает случай `Err` значением `true`. Разработчик подумал: «не будем блокировать людей
   из-за нашей собственной ошибки». Теперь любой сбой поиска делает пользователя администратором.
3. `is_admin` при `Err` возвращает `false`. Проверка **отказывает в закрытом состоянии** (fails closed).
4. `main` показывает трёх пользователей с обеими функциями.

```text
ana    open: true  closed: true
bob    open: false closed: false
carol  open: true  closed: false
```

Для `carol` две версии расходятся: `open: true`. Атакующему достаточно вызвать сбой поиска, например странным
именем пользователя или перегрузкой базы данных. Добавьте тест, который защищает исправление:

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn unknown_user_is_not_admin() {
        assert!(!is_admin("carol"));
        assert!(is_admin("ana"));
        assert!(!is_admin("bob"));
    }
}
```

Вторая ошибка этой категории — небрежный `unwrap()`. Сохраните `src/bin/quantity_panic.rs`:

```rust
fn main() {
    let text = "twelve";
    let quantity: u32 = text.parse().unwrap();
    println!("quantity: {quantity}");
}
```

```text
thread 'main' (4685023) panicked at src/bin/quantity_panic.rs:3:38:
called `Result::unwrap()` on an `Err` value: ParseIntError { kind: InvalidDigit }
```

**Что происходит:** `"twelve"` — не число, поэтому `parse` возвращает `Err`, а `unwrap()` превращает `Err` в
панику. Если обработчик запроса выполняет этот код с текстом из сети, один плохой запрос останавливает
программу (или поток), то есть это отказ в обслуживании ([Урок 10.1](01-security-mindset-and-the-cia-triad.md)).
Используйте `match`, `?` или `map_err` и возвращайте ошибку вызывающему коду, как вы узнали в [Уроке 04.10](../04-structs-enums-collections-errors/10-error-handling.md).

> 🔒 **Безопасность:** ещё два правила для ошибок. Во-первых, показывайте пользователю короткое общее
> сообщение («не удалось обработать запрос»), а подробности (пути к файлам, текст SQL, трассировки стека)
> пишите только в журнал. Подробности на странице ошибки помогают атакующему. Во-вторых, всегда освобождайте
> ресурсы (файлы, блокировки) и на пути ошибки; правила drop в Rust делают это в основном за вас.

## 6. A09 Security Logging and Alerting Failures

Если никто не записывает неудачные входы, отклонённые запросы и изменения прав, атака может месяцами оставаться
незамеченной. Хорошее логирование отвечает: **кто** что **сделал**, **когда**, **откуда** и с каким
**результатом**. Но у журналов есть собственные риски:

- **Никогда не пишите в журнал секреты** и полные персональные данные (пароли, токены, номера карт). См.
  Урок 10.1, раздел 3.1.
- **Никогда не пишите в строку журнала сырой пользовательский ввод.** Имя пользователя с переводом строки
  может подделать строку. Сохраните `src/bin/logging.rs`:

```rust
fn log_failed_login_bad(user: &str) {
    println!("WARN login failed for user {user}");
}

fn log_failed_login(user: &str) {
    // `{:?}` prints the text in quotes and turns a newline into the two characters \n
    println!("WARN login failed for user {user:?}");
}

fn main() {
    // The "user name" that an attacker typed into a login form:
    let attacker = "mallory\nINFO login succeeded for user admin";

    log_failed_login_bad(attacker);
    println!("---");
    log_failed_login(attacker);
}
```

```text
WARN login failed for user mallory
INFO login succeeded for user admin
---
WARN login failed for user "mallory\nINFO login succeeded for user admin"
```

**Что происходит:**

1. «Имя пользователя» содержит `\n`, перевод строки, а за ним текст, похожий на вторую запись журнала.
2. `log_failed_login_bad` печатает его как есть, поэтому вывод содержит **две** строки, и вторая ложно
   сообщает, что вошёл администратор.
3. `log_failed_login` использует `{user:?}`, формат `Debug`: он берёт текст в кавычки и записывает перевод
   строки как два символа `\n`. Одно событие остаётся одной строкой.

Логирование — это не только запись; кто-то (или какое-то оповещение) должен читать сигналы, например много
неудачных входов с одного адреса. Инструменты структурированных журналов и трассировки — тема
[Урока 09.10](../09-testing-debugging-benchmarking/10-logging-and-tracing.md).

## 7. Остальные категории в одном месте

- **A03 Supply Chain:** каждый добавленный крейт — код, который работает с вашими правами. Урок 10.13 учит
  `cargo audit` и `cargo deny`; `Cargo.lock` вы уже знаете из [Урока 05.8](../05-rust-intermediate/08-cargo-dependencies-and-versions.md).
- **A04 Cryptographic Failures:** используйте проверенные библиотеки и высокоуровневые API (Application Programming Interface, программный интерфейс — функции, которые предлагает библиотека). Никогда не
  изобретайте своё.
- **A05 Injection:** держите данные и команды раздельно. Урок 10.5 показывает внедрение SQL, команд оболочки и путей.
- **A06 Insecure Design:** отсутствие ограничения частоты или отсутствие проверки — не оплошность реализации.
  Моделируйте угрозы рано (Урок 10.2).
- **A07 Authentication Failures:** хешируйте пароли медленным алгоритмом, ограничивайте число попыток и не
  сообщайте, какая именно часть входа была неверной. *Подстановка учётных данных* работает, потому что люди
  повторно используют пароли.
- **A08 Integrity Failures:** проверяйте подписи и контрольные суммы того, что скачиваете или загружаете.
  `serde` в Rust не выполняет код из данных, но недоверенные данные всё равно могут быть огромными или глубоко
  вложенными, поэтому ограничивайте размеры.

## Частые ошибки

> ⚠️ **Частая ошибка:** выучить список наизусть и считать, что всё сделано. Top 10 — отправная точка.
> Используйте его вместе с моделированием угроз для вашей собственной программы.

> ⚠️ **Частая ошибка:** защищать только страницу в пользовательском интерфейсе. Скрыть кнопку «Удалить» от
> обычных пользователей не мешает прямому запросу. Проверка должна быть на сервере.

> ⚠️ **Частая ошибка:** писать `Err(_) => true` (или `unwrap_or(true)`) в проверке прав «ради безопасности
> для пользователя». Это безопасно для атакующего. Отказывайте в закрытом состоянии.

## Попробуйте сами

1. Создайте пакет, наберите пять программ и запустите их через `cargo run --bin <name>`.
2. В `access.rs` добавьте третьего пользователя `eve` без заметок. Что возвращает `get_note(&notes, "eve", 1)`?
   Даёт ли этот ответ ей какую-нибудь информацию?

## Упражнения

### Упражнение 1 — Назовите категорию (легко)

Какая категория OWASP 2025 подходит лучше всего? (a) Сайт всё ещё использует пароль `admin`, который
поставлялся с программой. (b) Сервер возвращает браузеру полную ошибку базы данных с названиями таблиц.
(c) Пользователь меняет `?invoice=100` на `?invoice=101` и видит счёт незнакомого человека. (d) Библиотека с
известной уязвимостью два года лежит в `Cargo.lock`. (e) Произошли сотни неудачных входов, и никто не заметил.

<details>
<summary>Подсказка</summary>

Используйте столбец «Простыми словами» таблицы из раздела 2.

</details>

<details>
<summary>Ответ</summary>

(a) A02 Security Misconfiguration (пароль по умолчанию; это также слабость аутентификации).
(b) A10 Mishandling of Exceptional Conditions (также A02, подробные ошибки). (c) A01 Broken Access Control (IDOR).
(d) A03 Software Supply Chain Failures. (e) A09 Security Logging and Alerting Failures.

</details>

### Упражнение 2 — Протестируйте конфигурацию (средне)

Добавьте в `config.rs` **тест**, доказывающий, что конфигурация с надёжным паролем на локальном адресе по
умолчанию принимается, и ещё один, доказывающий, что конфигурация с паролем из 11 символов отклоняется.

<details>
<summary>Подсказка</summary>

Используйте синтаксис обновления структуры: `Config { admin_password: Some(…), ..Config::default() }` копирует
все остальные поля из значения по умолчанию.

</details>

<details>
<summary>Ответ</summary>

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn the_default_config_does_not_start() {
        assert!(Config::default().validate().is_err());
    }

    #[test]
    fn a_strong_local_config_is_accepted() {
        let config = Config {
            admin_password: Some(String::from("a-long-enough-password")),
            ..Config::default()
        };
        assert!(config.validate().is_ok());
    }

    #[test]
    fn an_eleven_character_password_is_refused() {
        let config = Config {
            admin_password: Some(String::from("elevenchars")),
            ..Config::default()
        };
        assert!(config.validate().is_err());
    }
}
```

Почему это работает: в `elevenchars` ровно 11 символов, поэтому `p.len() < 12` истинно, и `validate` возвращает
`Err`. Проверка граничных значений (11 и 12) — лучший способ поймать ошибку на единицу в правиле безопасности.

</details>

### Упражнение 3 — Запишите отказ в доступе в журнал (сложно)

Измените `get_note` так, чтобы при отказе в доступе она печатала одну строку журнала в виде
`WARN note not available: user="ana" id=2`. Используйте формат, делающий внедрение в журнал невозможным.

<details>
<summary>Подсказка</summary>

Положите результат `find` в переменную, проверьте `is_none()`, напечатайте и верните переменную. Используйте `{user:?}`.

</details>

<details>
<summary>Ответ</summary>

```rust
fn get_note<'a>(notes: &'a [Note], user: &str, id: u32) -> Option<&'a Note> {
    let found = notes.iter().find(|n| n.id == id && n.owner == user);
    if found.is_none() {
        println!("WARN note not available: user={user:?} id={id}");
    }
    found
}
```

Тогда вывод `main` содержит `WARN note not available: user="ana" id=2` перед строкой `checked:   None`.
Почему это работает: имя пользователя проходит через `{:?}`, поэтому даже имя с переводом строки остаётся в одной
строке; id — число и не может содержать перевод строки.

</details>

## Вопросы для самопроверки

1. Что такое OWASP Top 10 и чем он не является?

<details>
<summary>Ответ</summary>

Список десяти самых серьёзных категорий риска для веб-приложений, составленный по реальным данным и
используемый для осведомлённости. Это не полный перечень всех ошибок и не полный чек-лист.

</details>

2. Что такое IDOR и как его предотвратить?

<details>
<summary>Ответ</summary>

Программа использует идентификатор из запроса, не проверив, что текущий пользователь может получить доступ к
этому объекту. Проверяйте владение или право на сервере для каждого запроса и каждого объекта.

</details>

3. Что означает «fail closed»?

<details>
<summary>Ответ</summary>

Когда проверка безопасности не может принять решение (из-за ошибки), она отказывает в действии.

</details>

4. Как имя пользователя может подделать строку журнала и что этому мешает?

<details>
<summary>Ответ</summary>

Перевод строки в имени начинает новую строку в журнале. Печать значения через `{:?}` (или другой шаг
экранирования) записывает перевод строки как `\n`, и одно событие остаётся в одной строке.

</details>

## Итоги

- OWASP Top 10 перечисляет категории риска; это карта, а не полный чек-лист.
- Порядок 2025 года: Broken Access Control, Security Misconfiguration, Supply Chain Failures, Cryptographic Failures, Injection, Insecure Design, Authentication Failures, Integrity Failures, Logging and Alerting Failures, Mishandling of Exceptional Conditions.
- Проверяйте доступ на сервере для каждого запроса и каждого объекта и запрещайте по умолчанию.
- Делайте конфигурацию по умолчанию безопасной и отказывайтесь запускаться с небезопасной.
- Проверки безопасности отказывают в закрытом состоянии; никогда не используйте `unwrap` на вводе.
- Записывайте в журнал события безопасности, но никогда секреты и никогда сырой ввод.
- Каждая категория указывает на урок этого этапа, где вы изучаете защиту.

## Источники и что почитать

- [OWASP Top 10:2025](https://top10.owasp.org/2025) — официальный список со страницей для каждой категории.
- [OWASP Top 10:2025 — Introduction](https://top10.owasp.org/2025/0x00_2025-Introduction/) — как составляется список и что изменилось с 2021 года.
- [OWASP Cheat Sheet: Insecure Direct Object Reference Prevention](https://cheatsheetseries.owasp.org/cheatsheets/Insecure_Direct_Object_Reference_Prevention_Cheat_Sheet.html) — IDOR подробно.
- [OWASP Cheat Sheet: Authorization](https://cheatsheetseries.owasp.org/cheatsheets/Authorization_Cheat_Sheet.html) — правила контроля доступа, включая запрет по умолчанию.
- [OWASP Cheat Sheet: Error Handling](https://cheatsheetseries.owasp.org/cheatsheets/Error_Handling_Cheat_Sheet.html) — что показывать пользователям и что писать в журнал.
- [OWASP Cheat Sheet: Logging](https://cheatsheetseries.owasp.org/cheatsheets/Logging_Cheat_Sheet.html) — что записывать и как избежать внедрения в журнал.
- [OWASP Cheat Sheet: SSRF Prevention](https://cheatsheetseries.owasp.org/cheatsheets/Server_Side_Request_Forgery_Prevention_Cheat_Sheet.html) — защита от Server-Side Request Forgery.

---

[🏠 Начало](../START-HERE.md) · [← 10.2 Моделирование угроз по STRIDE](02-threat-modeling-with-stride.md) · [📋 Этап 10](README.md) · [10.4 Проверка ввода: никогда не доверяйте вводу →](04-input-validation.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
