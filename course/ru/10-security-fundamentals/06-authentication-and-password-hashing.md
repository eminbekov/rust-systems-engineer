<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 10.5 Атаки внедрения: SQL, команды и пути](05-injection-attacks.md) · [📋 Этап 10](README.md) · [10.7 Авторизация, сессии и токены →](07-authorization-sessions-and-tokens.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/10-security-fundamentals/06-authentication-and-password-hashing.md) <!-- nav -->

# Урок 10.6 — Аутентификация и хеширование паролей

> **Этап 10 — Основы безопасности** · Урок 6 из 15 · ⏱ около 135 минут

## Что вы узнаете

- Чем отличаются аутентификация и авторизация и какие бывают три вида подтверждения личности.
- Почему программа никогда не должна хранить пароли открытым текстом и почему даже обычного хеша недостаточно.
- Что такое соль и как использовать Argon2id с крейтом `argon2`.
- Как построить небольшую систему входа, которая даёт одинаковый ответ на «нет такого пользователя» и «неверный пароль».
- Как сравнивать секреты за постоянное время и что ещё нужно для безопасного входа.

## Перед началом

- Вы прошли [Урок 10.5 — Атаки внедрения](05-injection-attacks.md).
- Вы знаете собственные типы ошибок ([Урок 04.10](../04-structs-enums-collections-errors/10-error-handling.md)), `HashMap` ([Урок 04.9](../04-structs-enums-collections-errors/09-hash-maps.md)) и крейты ([Урок 05.8](../05-rust-intermediate/08-cargo-dependencies-and-versions.md)).
- Создайте пакет: `cargo new --lib code/p10-passwords`, затем внутри него `cargo add argon2@0.6 sha2@0.11 subtle@2.6` и `mkdir src/bin`.
- В этом уроке используется версия 0.6 крейта `argon2`; его API (Application Programming Interface, программный интерфейс: функции и типы, которые он предлагает) изменился в 2025–2026 годах, поэтому код из старых руководств может не компилироваться.

## Новые слова

| Слово | Значение |
|---|---|
| **аутентификация** (authentication, часто «authn») | Доказательство того, **кто вы**: «этот запрос действительно пришёл от Ани». |
| **авторизация** (authorization, «authz») | Решение о том, **что вам разрешено**, когда уже известно, кто вы. Урок 10.7. |
| **учётные данные** (credential) | Доказательство, которое даёт пользователь: пароль, ключ, код. |
| **фактор аутентификации** (authentication factor) | Вид доказательства: то, что вы **знаете** (пароль), что у вас **есть** (телефон, аппаратный ключ), что вы **есть** (отпечаток пальца). |
| **MFA** (Multi-Factor Authentication, многофакторная аутентификация) | Запрос двух или более разных факторов, например пароля и кода с телефона. |
| **хеш-функция** (hash function) | Функция, превращающая любые данные в короткий отпечаток фиксированного размера. Один и тот же вход всегда даёт один и тот же отпечаток, а от отпечатка вернуться к входу нельзя. Хеши подробно объясняет Урок 10.9. |
| **соль** (salt) | Случайное значение, своё для каждого пароля, которое подмешивается перед хешированием. |
| **перебор** (brute force) | Пробовать много вариантов один за другим, пока один не подойдёт. |
| **радужная таблица** (rainbow table) | Большой заранее вычисленный список пар «пароль → хеш», позволяющий атакующему обращать быстрые хеши без соли поиском. |
| **Argon2id** | Алгоритм хеширования паролей, который намеренно медленный и требует много памяти, из-за чего массовый подбор становится дорогим. Победил в международном конкурсе 2015 года. |
| **строка PHC** (PHC string, формат Password Hashing Competition) | Один текст вроде `$argon2id$v=19$m=19456,t=2,p=1$salt$hash`, хранящий алгоритм, его настройки, соль и хеш. |
| **атака по времени** (timing attack) | Узнавание секрета по измерению того, сколько времени программа отвечает. |
| **сравнение за постоянное время** (constant-time comparison) | Сравнение, которое занимает одно и то же время при любых данных. |
| **перечисление пользователей** (user enumeration) | Выяснение, какие имена пользователей существуют, по небольшим различиям в ответах. |
| **SHA-256** (Secure Hash Algorithm, 256-bit) | Очень распространённая быстрая хеш-функция. Её результат всегда занимает 32 байта. |
| **MAC** (Message Authentication Code, код аутентификации сообщения) | Короткое значение, доказывающее, что сообщение не изменено и пришло от того, кто знает секретный ключ. Урок 10.9. |
| **КиБ / МиБ** (KiB / MiB, кибибайт / мебибайт) | Единицы размера памяти: 1024 байта / 1024 × 1024 байта. |

## 1. Кто вы? Аутентификация

**Аутентификация** отвечает на вопрос «кто это?». Она идёт раньше **авторизации** («что этому человеку
разрешено?»). Пользователь доказывает личность одним или несколькими *факторами*: тем, что он **знает** (пароль),
что у него **есть** (телефон) или чем он **является** (отпечаток пальца). Использование двух разных факторов —
это **MFA**, и оно останавливает большинство атак, которые крадут только пароль.

Пароли остаются самым частым фактором. Поэтому этот урок отвечает на практический вопрос: **как серверу
проверить пароль, не храня его?**

## 2. Три способа хранить пароль

Представьте атакующего, который украл вашу базу данных (с настоящими компаниями это случается каждый год). Что
он получает?

| Хранение | Что получает вор | Вердикт |
|---|---|---|
| **Открытый текст** | Все пароли сразу. Пользователи, повторно использующие пароль, теряют и другие учётные записи (*подстановка учётных данных*, [Урок 10.3](03-the-owasp-top-10.md)). | Никогда |
| **Быстрый хеш**, например SHA-256, без соли | Все распространённые пароли за минуты, потому что одинаковые пароли имеют одинаковые хеши, а радужная таблица их обращает. | Никогда |
| **Медленный хеш пароля с солью** (Argon2id) | Только отпечатки, которые дорого подбирать по одному. | Да |

Серверу никогда не нужен сам пароль. При регистрации он сохраняет *хеш*. При входе он снова хеширует введённый
пароль с теми же настройками и солью и сравнивает результаты.

## 3. Почему «медленно» — это достоинство

Для большинства задач хеш должен быть быстрым. Для паролей верно обратное: законный вход хеширует **один**
пароль, а вор с украденной базой должен хешировать **миллиарды** вариантов. Если одна попытка стоит десять
миллисекунд вместо десяти наносекунд, работа вора становится в миллион раз дороже, а пользователь этого не замечает.

Измерьте это сами. Сохраните `src/bin/speed.rs`:

```rust
use std::time::Instant;

use argon2::Argon2;
use argon2::password_hash::PasswordHasher;
use sha2::{Digest, Sha256};

fn main() {
    // SHA-256 is a fast hash (Lesson 10.9). How many guesses can a thief try per second?
    let start = Instant::now();
    let mut last = [0u8; 32];
    for i in 0..1_000_000u32 {
        last = Sha256::digest(format!("guess-{i}").as_bytes()).into();
    }
    let sha_time = start.elapsed();
    println!(
        "1,000,000 SHA-256 hashes: {sha_time:.2?} (last byte {})",
        last[31]
    );

    // Argon2id with the default settings.
    let argon2 = Argon2::default();
    let start = Instant::now();
    for i in 0..5 {
        argon2
            .hash_password(format!("guess-{i}").as_bytes())
            .expect("hashing failed");
    }
    let argon_time = start.elapsed() / 5;
    println!("one Argon2id hash:        {argon_time:.2?}");

    let ratio = sha_time.as_secs_f64() / 1_000_000.0;
    println!(
        "one Argon2id hash costs about {:.0} times more than one SHA-256 hash",
        argon_time.as_secs_f64() / ratio
    );
}
```

Запустите: `cargo run --release --bin speed`. Используйте `--release`: криптографический код в режиме debug очень медленный.

```text
1,000,000 SHA-256 hashes: 51.97ms (last byte 83)
one Argon2id hash:        10.77ms
one Argon2id hash costs about 207304 times more than one SHA-256 hash
```

*(Это пример с одной машины; ваши числа будут другими.)*

**Что происходит:**

1. Первый цикл вычисляет миллион хешей SHA-256 через `Sha256::digest`. (`sha2` — крейт проекта RustCrypto;
   трейт `Digest` даёт метод `digest`.) Быстрый хеш заканчивает за десятки миллисекунд.
2. Вторая часть пять раз запускает Argon2id с настройками по умолчанию и берёт среднее.
3. `Argon2::default()` — это Argon2id версии 19 с 19 МиБ памяти, 2 проходами и 1 потоком. Каждая попытка должна
   заполнить и перемешать 19 МиБ памяти, что делает специальное оборудование для взлома гораздо менее полезным.
4. Последняя строка печатает отношение. Стоимость одной попытки для вора в сотни тысяч раз выше.

## 4. Соль и строка PHC

Если два пользователя выбрали один пароль, а у хеша нет соли, два сохранённых значения одинаковы, и одна удачная
догадка открывает обоих. **Соль** — это случайное значение, создаваемое для **каждого** пароля и хранимое рядом с
хешем. Она не секретна. Она лишь гарантирует, что каждое сохранённое значение отличается и что таблицы заранее
вычисленных хешей бесполезны. Сохраните `src/bin/hash.rs`:

```rust
use argon2::Argon2;
use argon2::password_hash::phc::PasswordHash;
use argon2::password_hash::{PasswordHasher, PasswordVerifier};

fn main() {
    let password = b"correct horse battery";
    let argon2 = Argon2::default();

    // The salt is chosen at random inside hash_password.
    let first = argon2
        .hash_password(password)
        .expect("hashing failed")
        .to_string();
    let second = argon2
        .hash_password(password)
        .expect("hashing failed")
        .to_string();
    println!("first:  {first}");
    println!("second: {second}");
    println!("same text? {}", first == second);

    // Verification reads the settings and the salt from the stored text.
    let parsed = PasswordHash::new(&first).expect("not a valid hash text");
    println!(
        "right password: {:?}",
        argon2.verify_password(password, &parsed)
    );
    println!(
        "wrong password: {:?}",
        argon2.verify_password(b"correct horse batterY", &parsed)
    );
}
```

```text
first:  $argon2id$v=19$m=19456,t=2,p=1$FYXVw/VkzT+3kvgzFLRSuw$g1Y/9adpmya9FCAsn3eLtODYr3wGeiloOwxxeC4svO8
second: $argon2id$v=19$m=19456,t=2,p=1$ZisNyWGnIyV89bq0Lgnzqg$sJ7glk8AD6bCrxiZNr/rcKeZmAHmqh0uzCsGNRodmjs
same text? false
right password: Ok(())
wrong password: Err(PasswordInvalid)
```

*(Соль и хеш меняются при каждом запуске.)*

**Что происходит:**

1. `argon2.hash_password(password)` создаёт свежую случайную соль из защищённого генератора случайных чисел
   операционной системы, хеширует пароль и возвращает `PasswordHash`. `.to_string()` превращает его в строку PHC.
   Метод предоставляет трейт `PasswordHasher`, поэтому строка `use` нужна.
2. Один и тот же пароль даёт два разных текста, потому что соли различаются. `same text? false`.
3. Читайте текст, считая `$` разделителем: `argon2id` (алгоритм), `v=19` (версия), `m=19456,t=2,p=1` (настройки:
   память в КиБ, число проходов, параллельные потоки), затем соль, затем хеш, оба в Base64 (способ записи байтов
   буквами и цифрами). **Всё, что нужно для проверки пароля, находится в одном тексте**, поэтому хранится один
   столбец.
4. `PasswordHash::new(&first)` разбирает текст. `verify_password` читает из него настройки и соль, хеширует введённый
   пароль и сравнивает. Сравнение выполняется за постоянное время. Неверный пароль возвращает
   `Err(PasswordInvalid)`.

**Три настройки.** *Память* (`m`) делает подбор дорогим для видеокарт. *Проходы* (`t`) повторяют работу. *Потоки*
(`p`) допускают параллельную работу. Минимум OWASP для Argon2id — 19 МиБ, 2 прохода и 1 поток, и это значения
крейта по умолчанию. Выбирайте самую сильную настройку, которую может позволить ваш сервер: каждому входу нужно
столько памяти **одновременно**, поэтому тысяче одновременных входов нужно в тысячу раз по 19 МиБ. Поскольку
настройки хранятся в тексте, их можно позже повысить, не ломая старые хеши: каждый старый хеш по-прежнему
проверяется со своими настройками.

> 🔒 **Безопасность:** используйте **алгоритм хеширования паролей** (Argon2id; scrypt или bcrypt, если иначе
> нельзя), никогда не обычный SHA-256 и никогда не изобретённый вами алгоритм. Не обрезайте пароли (bcrypt читает
> только первые 72 байта; Argon2 читает все).

## 5. Небольшая система входа

Теперь соберём всё в `src/lib.rs`. Проектные решения отмечены в комментариях.

```rust
use std::collections::HashMap;

use argon2::password_hash::{PasswordHasher, PasswordVerifier};
use argon2::{Algorithm, Argon2, Params, Version};

pub const MIN_PASSWORD_CHARS: usize = 15;
pub const MAX_PASSWORD_CHARS: usize = 128;

#[derive(Debug, PartialEq, Eq)]
pub enum RegisterError {
    UserExists,
    PasswordTooShort,
    PasswordTooLong,
    HashingFailed,
}

/// One error for every kind of failure: the caller learns nothing extra.
#[derive(Debug, PartialEq, Eq)]
pub struct InvalidCredentials;

pub struct UserStore {
    argon2: Argon2<'static>,
    users: HashMap<String, String>, // user name -> PHC string
    dummy_hash: String,             // used when the user does not exist
}

impl UserStore {
    /// Safe defaults: Argon2id, 19 MiB of memory, 2 passes, 1 lane.
    pub fn new() -> Self {
        Self::with_params(Params::default())
    }

    /// Lets tests use cheap settings. Never do this in production.
    pub fn with_params(params: Params) -> Self {
        let argon2 = Argon2::new(Algorithm::Argon2id, Version::V0x13, params);
        let dummy_hash = argon2
            .hash_password(b"a password that nobody has")
            .expect("hashing with valid parameters works")
            .to_string();
        UserStore {
            argon2,
            users: HashMap::new(),
            dummy_hash,
        }
    }

    pub fn register(&mut self, name: &str, password: &str) -> Result<(), RegisterError> {
        let length = password.chars().count();
        if length < MIN_PASSWORD_CHARS {
            return Err(RegisterError::PasswordTooShort);
        }
        if length > MAX_PASSWORD_CHARS {
            return Err(RegisterError::PasswordTooLong);
        }
        if self.users.contains_key(name) {
            return Err(RegisterError::UserExists);
        }
        let hash = self
            .argon2
            .hash_password(password.as_bytes())
            .map_err(|_| RegisterError::HashingFailed)?
            .to_string();
        self.users.insert(name.to_string(), hash);
        Ok(())
    }

    pub fn login(&self, name: &str, password: &str) -> Result<(), InvalidCredentials> {
        let known = self.users.get(name);
        // Always do the expensive work, even for an unknown user, so that the
        // answer takes the same time and does not reveal which names exist.
        let hash = known.unwrap_or(&self.dummy_hash);
        let password_ok = self
            .argon2
            .verify_password(password.as_bytes(), hash.as_str())
            .is_ok();
        if known.is_some() && password_ok {
            Ok(())
        } else {
            Err(InvalidCredentials)
        }
    }

    /// For the demo only: the stored text of a user.
    pub fn stored_hash(&self, name: &str) -> Option<&str> {
        self.users.get(name).map(String::as_str)
    }
}

impl Default for UserStore {
    fn default() -> Self {
        Self::new()
    }
}
```

**Что происходит:**

1. `RegisterError` перечисляет, почему может не удаться регистрация. `InvalidCredentials` — **одна** ошибка для
   любого неудачного входа. Вызывающий код (а значит, и атакующий) не может отличить «нет такого пользователя» от
   «неверный пароль».
2. `UserStore` хранит объект `Argon2`, словарь от имени пользователя к строке PHC и `dummy_hash`. В настоящей
   системе словарь — это таблица базы данных.
3. `register` проверяет длину пароля в **символах** (`chars().count()`). Правила следуют совету NIST (National Institute
   of Standards and Technology, Национальный институт стандартов и технологий — агентство стандартов США) для паролей, которые являются единственным фактором: не менее 15 символов, длинные пароли
   до 128 разрешены. Правил вроде «одна заглавная буква и один символ» **нет**: они заставляют людей выбирать
   предсказуемые пароли. Настоящая система ещё проверяет список известных утёкших паролей.
4. `hash_password(...).to_string()` — тот же вызов, что в `hash.rs`. Результат сохраняется, пароль отбрасывается.
5. В `login` строка `known.unwrap_or(&self.dummy_hash)` — самая хитрая. Для неизвестного пользователя мы всё равно
   проверяем введённый пароль, но против фиктивного хеша. Иначе «нет такого пользователя» отвечало бы за микросекунду,
   а «неверный пароль» за десять миллисекунд, и атакующий узнавал бы по времени, какие имена существуют
   (перечисление пользователей).
6. Только если пользователь существует **и** пароль верен, `login` возвращает `Ok`.
7. `with_params` существует для тестов: дешёвые настройки (8 КиБ, один проход) позволяют тесту выполниться за
   миллисекунды. `Params::new(m_cost, t_cost, p_cost, output_len)` строит настройки, а `None` означает длину вывода
   по умолчанию. Никогда не используйте дешёвые настройки для настоящих пользователей.

Теперь посмотрим, как это работает. Сохраните `src/bin/login.rs` и запустите `cargo run --release --bin login`:

```rust
use std::time::Instant;

use p10_passwords::UserStore;

fn main() {
    let mut store = UserStore::new();

    println!("{:?}", store.register("ana", "short"));
    println!("{:?}", store.register("ana", "correct horse battery"));
    println!("{:?}", store.register("ana", "another long password"));

    let attempts = [
        ("ana", "correct horse battery"),
        ("ana", "wrong password here"),
        ("nobody", "wrong password here"),
    ];
    for (name, password) in attempts {
        let start = Instant::now();
        let result = store.login(name, password);
        println!("{name:<7} {result:?} in {:.1?}", start.elapsed());
    }
}
```

```text
Err(PasswordTooShort)
Ok(())
Err(UserExists)
ana     Ok(()) in 18.2ms
ana     Err(InvalidCredentials) in 10.7ms
nobody  Err(InvalidCredentials) in 13.2ms
```

Известный пользователь с неверным паролем и неизвестный пользователь получают одну и ту же ошибку и тратят
примерно одно и то же время. (Время немного меняется от запуска к запуску.)

Наконец, тесты, в конце `src/lib.rs`:

```rust
#[cfg(test)]
mod tests {
    use super::*;

    fn cheap_store() -> UserStore {
        // 8 KiB of memory, 1 pass: fast enough for tests (never for real users).
        UserStore::with_params(Params::new(8, 1, 1, None).unwrap())
    }

    #[test]
    fn right_password_logs_in_and_wrong_one_does_not() {
        let mut store = cheap_store();
        store.register("ana", "correct horse battery").unwrap();
        assert_eq!(store.login("ana", "correct horse battery"), Ok(()));
        assert_eq!(
            store.login("ana", "correct horse batterY"),
            Err(InvalidCredentials)
        );
    }

    #[test]
    fn unknown_user_and_wrong_password_give_the_same_error() {
        let mut store = cheap_store();
        store.register("ana", "correct horse battery").unwrap();
        assert_eq!(
            store.login("nobody", "whatever password"),
            store.login("ana", "wrong password!!")
        );
    }

    #[test]
    fn password_length_rules() {
        let mut store = cheap_store();
        assert_eq!(
            store.register("a", "short"),
            Err(RegisterError::PasswordTooShort)
        );
        assert_eq!(
            store.register("a", &"x".repeat(129)),
            Err(RegisterError::PasswordTooLong)
        );
        assert!(store.register("a", &"x".repeat(128)).is_ok());
        assert_eq!(
            store.register("a", "another long password"),
            Err(RegisterError::UserExists)
        );
    }

    #[test]
    fn the_same_password_gives_different_stored_texts() {
        let mut store = cheap_store();
        store.register("ana", "correct horse battery").unwrap();
        store.register("bob", "correct horse battery").unwrap();
        assert_ne!(store.stored_hash("ana"), store.stored_hash("bob"));
    }

    #[test]
    fn the_stored_text_does_not_contain_the_password() {
        let mut store = cheap_store();
        store.register("ana", "correct horse battery").unwrap();
        let stored = store.stored_hash("ana").unwrap();
        assert!(stored.starts_with("$argon2id$v=19$"));
        assert!(!stored.contains("horse"));
    }
}
```

```bash
cargo test
```

```text
running 5 tests
test tests::the_stored_text_does_not_contain_the_password ... ok
test tests::password_length_rules ... ok
test tests::the_same_password_gives_different_stored_texts ... ok
test tests::unknown_user_and_wrong_password_give_the_same_error ... ok
test tests::right_password_logs_in_and_wrong_one_does_not ... ok

test result: ok. 5 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

**Что происходит:** тесты проверяют хороший вход и плохой, что оба случая отказа возвращают равные ошибки, правило
длины (короткий пароль отклоняется, 128 символов принимаются, 129 отклоняются, а повторная регистрация того же имени
не удаётся), что одинаковые пароли дают разные сохранённые тексты и что сохранённый текст начинается с
`$argon2id$v=19$` и не содержит пароля.

> 🧪 **Тестирование:** «время» нельзя надёжно проверить модульным тестом, поэтому проверяйте *структуру*, которая
> делает его безопасным: один тип ошибки и один и тот же путь выполнения для обоих отказов. Внимательно
> проверяйте этот путь при ревью.

## 6. Сравнение секретов за постоянное время

Обычный `==` для двух строк байтов может остановиться на первом различающемся байте. Время его работы тогда
подсказывает вызывающему, сколько начальных байтов было верно: это **атака по времени**, которая может восстановить
секрет байт за байтом. Для хешей паролей библиотека уже делает безопасно. Для всего остального, что вы сравниваете
сами (ключ API, токен сессии, MAC из [Урока 10.9](09-cryptography-hashes-macs-and-randomness.md)), используйте крейт
`subtle`. Добавьте это в функцию `#[test]` в ваших тестах:

```rust
use subtle::ConstantTimeEq;

let stored: &[u8] = b"s3cr3t-token-value";
let attempt: &[u8] = b"s3cr3t-token-valuX";
let equal: bool = stored.ct_eq(attempt).into();
assert!(!equal);
```

`ct_eq` (из трейта `ConstantTimeEq`) сравнивает **каждый** байт и возвращает `Choice` — значение, не являющееся
обычным `bool`, поэтому компилятор не делает по нему раннего ветвления. `.into()` в конце преобразует его в `bool`.

## 7. Что ещё нужно для безопасного входа

- **Ограничение частоты.** Разрешайте лишь несколько попыток в минуту на учётную запись и на адрес, с растущими
  задержками. Подбор онлайн должен быть медленным. Осторожно: жёсткая блокировка после трёх попыток позволяет
  атакующему блокировать других (Урок 10.1, раздел 3.4).
- **MFA.** Тогда одного украденного или угаданного пароля недостаточно.
- **Проверка на утёкшие пароли.** Отклоняйте пароли, встречающиеся в публичных утечках (сервисы вроде
  «Have I Been Pwned» предлагают запрос по диапазону, не раскрывающий пароль).
- **Шифрованная передача.** Пароль едет на сервер, поэтому используйте TLS (Transport Layer Security, протокол шифрования сетевых соединений, [Урок 10.11](11-tls-and-certificates.md)).
- **Безопасный сброс.** Ссылка для сброса должна быть случайной, одноразовой и недолговечной, а сброс не должен
  раскрывать, существует ли адрес электронной почты. Никогда не отправляйте пароль по почте.
- **Никаких паролей в журналах, ошибках и URL** (Uniform Resource Locator, веб-адресах; [Урок 10.8](08-secrets-management.md)).

## Частые ошибки

> ⚠️ **Частая ошибка:** хешировать универсальным хешем: `sha256(password)`. Он быстрый и даёт вору миллиарды
> попыток в секунду.

> ⚠️ **Частая ошибка:** возвращать «пользователь не найден» и «неверный пароль» как два разных сообщения или
> отвечать на одно намного быстрее, чем на другое.

> ⚠️ **Частая ошибка:** максимальная длина пароля 12 или 16 символов либо молчаливое обрезание паролей. Разрешайте
> не менее 64.

> ⚠️ **Частая ошибка:** тестировать с боевыми настройками. 19 МиБ на хеш делают большой набор тестов медленным;
> используйте дешёвые настройки **только в тестах**.

## Попробуйте сами

1. Наберите библиотеку, три программы и тесты. Запустите `cargo test`, затем программы с `--release`.
2. В `hash.rs` создайте `Argon2` с `Params::new(64 * 1024, 3, 1, None)` (64 МиБ, 3 прохода) через
   `Argon2::new(Algorithm::Argon2id, Version::V0x13, params)` (добавьте `use argon2::{Algorithm, Params, Version};`).
   Как изменилась строка PHC?

## Упражнения

### Упражнение 1 — Выберите хранение (легко)

Оцените каждый способ хранения `"hunter2hunter2"` как хороший или плохой и объясните почему: (a) сам текст;
(b) base64 от текста; (c) `sha256` без соли; (d) `sha256` от имени пользователя плюс пароля; (e) строка PHC Argon2id.

<details>
<summary>Подсказка</summary>

Задайте два вопроса: может ли вор обратить это сразу (без работы)? Может ли вор пробовать миллиарды вариантов
в секунду?

</details>

<details>
<summary>Ответ</summary>

(a) Плохо: открытый текст. (b) Плохо: Base64 — это кодирование, а не защита; любой может декодировать. (c) Плохо:
быстрый и без соли, поэтому распространённые пароли падают перед радужной таблицей. (d) Плохо: имя пользователя
делает значение уникальным для каждого пользователя (слабая соль), но это всё равно быстрый хеш, поэтому подбор
дёшев, а имя не случайно. (e) Хорошо.

</details>

### Упражнение 2 — Проверьте границу правила (средне)

Напишите **тест**, показывающий, что `register` отклоняет пароль из 14 символов и принимает пароль из 15 символов.
Используйте дешёвые параметры.

<details>
<summary>Подсказка</summary>

`UserStore::with_params(Params::new(8, 1, 1, None).unwrap())`. Строки `"fourteen chars"` (14 символов) и
`"fifteen chars!!"` (15 символов) легко посчитать.

</details>

<details>
<summary>Ответ</summary>

```rust
#[test]
fn fourteen_characters_are_refused_and_fifteen_accepted() {
    let mut store = UserStore::with_params(Params::new(8, 1, 1, None).unwrap());
    assert_eq!(
        store.register("a", "fourteen chars"),
        Err(RegisterError::PasswordTooShort)
    );
    assert!(store.register("a", "fifteen chars!!").is_ok());
}
```

Поместите его внутрь `mod tests`. Почему это работает: правило — `length < 15`, поэтому 14 не проходит, а 15 проходит.
Тесты на границе ловят ошибки `<` вместо `<=`.

</details>

### Упражнение 3 — Разберите проект (сложно)

Коллега предлагает: «Мы храним `sha256(password + SECRET)`, где `SECRET` — одна длинная случайная строка в нашем
исходном коде. Это как соль, так что мы в безопасности». Назовите не менее трёх проблем и предложите исправление.

<details>
<summary>Подсказка</summary>

`SECRET` разный для каждого пользователя? SHA-256 медленный? Куда ещё попадает исходный код?

</details>

<details>
<summary>Ответ</summary>

(1) `SECRET` один для всех пользователей, поэтому одинаковые пароли по-прежнему дают одинаковые хеши, а один
взломанный пароль показывает, у кого ещё такой же. Соль должна быть случайной и уникальной для каждого пароля.
(2) SHA-256 быстрый, поэтому вор, у которого есть база и секрет, может пробовать миллиарды вариантов в секунду.
(3) Секрет в исходном коде попадает в историю Git, сборки и резервные копии ([Урок 10.8](08-secrets-management.md)).
Один общий секрет может быть *перцем* (pepper) как **дополнительный** слой, хранимый вне кода и вне базы данных, но
он не заменяет соль и медленный хеш. Исправление: хранить строку PHC Argon2id, созданную `hash_password`, которая
делает случайную соль для каждого пользователя.

</details>

## Вопросы для самопроверки

1. Чем отличаются аутентификация и авторизация?

<details>
<summary>Ответ</summary>

Аутентификация доказывает, кто это; авторизация решает, что этому человеку разрешено.

</details>

2. Почему SHA-256 — плохой выбор для хранения паролей?

<details>
<summary>Ответ</summary>

Он создан быстрым, поэтому вор может пробовать миллиарды вариантов в секунду. Алгоритм хеширования паролей
намеренно медленный и требует много памяти.

</details>

3. Что делает соль и должна ли она быть секретной?

<details>
<summary>Ответ</summary>

Она делает каждый сохранённый хеш разным даже для равных паролей и делает заранее вычисленные таблицы бесполезными.
Она не обязана быть секретной и хранится рядом с хешем.

</details>

4. Почему `login` проверяет пароль против фиктивного хеша для неизвестного пользователя?

<details>
<summary>Ответ</summary>

Чтобы ответ занимал столько же времени, сколько для известного пользователя, и атакующий не мог выяснить по времени,
какие имена существуют.

</details>

5. Когда нужно использовать сравнение за постоянное время?

<details>
<summary>Ответ</summary>

Когда вы сами сравниваете секретные значения (токены, ключи, MAC). Обычный `==` может остановиться раньше и
выдать, сколько начальных байтов было верно.

</details>

## Итоги

- Аутентификация доказывает личность; авторизация решает права. Используйте MFA, когда можете.
- Никогда не храните пароли открытым текстом или с быстрым хешем. Храните хеш Argon2id с солью как одну строку PHC.
- Медленное и требовательное к памяти хеширование делает массовый подбор дорогим; настройки хранятся в тексте и их можно повысить позже.
- Давайте одну и ту же ошибку и примерно одно и то же время для неизвестного пользователя и неверного пароля.
- Разрешайте длинные пароли, требуйте разумную минимальную длину, без правил состава, и проверяйте список запрещённых.
- Сравнивайте секреты функциями с постоянным временем (`subtle`).
- Добавьте ограничение частоты, TLS, безопасный сброс и никогда не пишите пароли в журналы.

## Источники и что почитать

- [OWASP Password Storage Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Password_Storage_Cheat_Sheet.html) — рекомендуемые настройки Argon2id и альтернативы.
- [OWASP Authentication Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html) — общие сообщения об ошибках, ограничение частоты, MFA и сброс.
- [NIST SP 800-63B-4 — Authentication and Authenticator Management](https://pages.nist.gov/800-63-4/sp800-63b.html) — официальные рекомендации правительства США о длине паролей, списках запрещённого и правилах состава (SP — Special Publication, «специальная публикация», серия руководств NIST по безопасности).
- [RFC 9106 — Argon2](https://www.rfc-editor.org/rfc/rfc9106) — спецификация алгоритма и советы по параметрам (RFC, Request for Comments, «запрос комментариев», — документ официальной серии интернет-стандартов).
- [PHC string format](https://github.com/P-H-C/phc-string-format/blob/master/phc-sf-spec.md) — как устроен текст `$argon2id$…`.
- [Документация крейта `argon2`](https://docs.rs/argon2/0.6.0/argon2/) — использованный здесь API.
- [Документация крейта `subtle`](https://docs.rs/subtle/2.6.1/subtle/) — сравнение за постоянное время.

---

[🏠 Начало](../START-HERE.md) · [← 10.5 Атаки внедрения: SQL, команды и пути](05-injection-attacks.md) · [📋 Этап 10](README.md) · [10.7 Авторизация, сессии и токены →](07-authorization-sessions-and-tokens.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
