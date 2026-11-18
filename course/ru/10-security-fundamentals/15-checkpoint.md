<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 10.14 Проект: зашифрованное хранилище секретов](14-project-encrypted-secrets-vault.md) · [📋 Этап 10](README.md) · [Этап 11 — Linux, командная оболочка и контейнеры →](../11-linux-shell-and-containers/README.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/10-security-fundamentals/15-checkpoint.md) <!-- nav -->

# Урок 10.15 — Контрольная точка

> **Этап 10 — Основы безопасности** · Урок 15 из 15 · ⏱ около 105 минут

## Что вы узнаете

Этот урок не учит ничему новому. Он проверяет, умеете ли вы пользоваться тем, чему научил Этап 10, и показывает, куда вернуться, если что-то всё ещё слабо. Вы:

- Ответите на 18 вопросов о мышлении безопасности, моделировании угроз, OWASP (Open Worldwide Application Security Project, открытый всемирный проект по безопасности приложений) Top 10, проверке ввода, внедрении, паролях, авторизации, секретах, криптографии, TLS (Transport Layer Security, защита транспортного уровня), безопасности памяти и безопасности цепочки поставок.
- Решите 4 практических задания, которые объединяют несколько уроков. У трёх из них есть тесты.
- Найдёте уроки, которые стоит повторить, с прямыми ссылками.

## Перед началом

- Вы прошли [Урок 10.14 — Проект: зашифрованное хранилище секретов](14-project-encrypted-secrets-vault.md). Ваш `vault` работает, его тесты проходят, и он закоммичен.
- Отвечайте на вопросы **без подсказок**: без уроков, компилятора и поиска. Сначала запишите свой ответ, потом откройте скрытый ответ. Неверный ответ, найденный сейчас, обходится дёшево; та же ошибка, найденная в настоящей системе, — нет.
- Для практических заданий можно пользоваться компилятором и уроками, но сначала попробуйте без подсказок. Как всегда на этом этапе, работайте только со своим кодом и своими машинами.
- Создайте один пакет для трёх заданий с кодом: `cargo new --lib code/p10-checkpoint`, затем `cargo add sha2@0.11 hex@0.4 subtle@2.6 getrandom@0.4 argon2@0.6`. Задание 1 идёт в `src/session.rs`, Задание 2 — в `src/record.rs`, Задание 3 — в `src/password.rs`, а в `src/lib.rs` лежит `pub mod password; pub mod record; pub mod session;`.
- Ваш учебный репозиторий чист (`git status` в `~/projects/rust`). Работайте в ветке: `git switch -c lesson/10-15-checkpoint`.

## Новые слова

| Слово | Значение |
|---|---|
| **контрольная точка** (checkpoint) | Пауза в конце этапа, чтобы проверить себя, прежде чем идти дальше. |
| **без подсказок** (closed book) | Ответы только по памяти, без обращения к источникам. |
| **проверка безопасности** (security review) | Чтение кода или проекта с вопросом «как это можно использовать во вред?», а не «работает ли это?». |

## Вопросы для самопроверки

Ставьте по одному баллу за каждый **верный и полный** ответ. Будьте честны: проверяете только вы.

### Мышление, угрозы и OWASP Top 10 (Уроки 10.1–10.3)

1. Что означают буквы триады CIA? Приведите по одному примеру ошибки, нарушающей каждое свойство.

<details>
<summary>Ответ</summary>

Confidentiality (конфиденциальность: секрет в лог-файле), Integrity (целостность: целочисленное переполнение, меняющее сумму; изменённый файл, который принимается) и Availability (доступность: однобайтовый пакет, роняющий сервис). Засчитывается любой верный пример. (Урок 10.1.)

</details>

2. Что означает STRIDE и когда в проекте проводят моделирование угроз?

<details>
<summary>Ответ</summary>

Spoofing (подмена личности), Tampering (подмена данных), Repudiation (отказ от авторства), Information disclosure (раскрытие информации), Denial of service (отказ в обслуживании), Elevation of privilege (повышение привилегий). Его проводят рано, на проекте, пока кода ещё нет, и обновляют при изменении проекта. (Урок 10.2.)

</details>

3. Какая категория возглавляет OWASP Top 10 и какие две привычки предотвращают большую её часть?

<details>
<summary>Ответ</summary>

Broken Access Control (нарушенный контроль доступа). Проверяйте доступ на **сервере**, для каждого запроса и каждого объекта, и по умолчанию запрещайте. (Урок 10.3.)

</details>

4. Что значит «fail closed» (закрываться при сбое)? Приведите пример.

<details>
<summary>Ответ</summary>

Когда проверка безопасности не может принять решение или ломается (ошибка, отсутствующая настройка), ответ — «запретить», а не «разрешить». Пример: если не удалось найти разрешения, запрос отклоняется. (Уроки 10.1 и 10.3.)

</details>

### Ввод и внедрение (Уроки 10.4 и 10.5)

5. Почему список разрешённого лучше списка запрещённого?

<details>
<summary>Ответ</summary>

Список разрешённого принимает только то, что заведомо допустимо, поэтому всё, о чём вы не подумали, отвергается. Список запрещённого отвергает только то, что вы догадались перечислить, а атакующие находят то, что вы забыли. (Урок 10.4.)

</details>

6. Функция строит `format!("SELECT * FROM users WHERE name = '{name}'")`. В чём ошибка и как её исправить?

<details>
<summary>Ответ</summary>

SQL-внедрение (SQL, Structured Query Language, язык структурированных запросов): данные становятся частью кода. Используйте параметризованный запрос (`?` с привязанными значениями), чтобы данные и код передавались раздельно. (Урок 10.5.)

</details>

7. Как безопасно запустить внешнюю программу с именем файла от пользователя в Rust?

<details>
<summary>Ответ</summary>

Не использовать командную оболочку. Применять `Command::new("program").arg(file_name)`, чтобы имя было одним аргументом, и добавить `--` или отвергать имена, начинающиеся с `-`, чтобы предотвратить внедрение аргументов. (Урок 10.5.)

</details>

### Пароли, авторизация и секреты (Уроки 10.6–10.8)

8. Как сервер должен хранить пароли и почему `SHA-256(пароль)` — неправильно?

<details>
<summary>Ответ</summary>

В виде хеша Argon2id со случайной солью, хранимого одной PHC-строкой (PHC, Password Hashing Competition, формат строки). SHA-256 быстрый, поэтому вор с базой данных может пробовать миллиарды вариантов в секунду. (Уроки 10.6 и 10.9.)

</details>

9. Вход возвращает «unknown user» для отсутствующей учётной записи и «wrong password» для неверного пароля. Что не так?

<details>
<summary>Ответ</summary>

Это сообщает атакующему, какие имена пользователей существуют (перебор имён пользователей). Возвращайте одно и то же сообщение (и тратьте примерно одно и то же время) в обоих случаях. (Урок 10.6.)

</details>

10. Назовите три свойства хорошего токена сессии.

<details>
<summary>Ответ</summary>

Не менее 128 случайных бит от CSPRNG операционной системы; сервер хранит только его хеш и ведёт состояние, с таймаутами простоя и абсолютным; токен меняется при входе и удаляется на сервере при выходе. В cookie: `HttpOnly`, `Secure`, `SameSite`. (Урок 10.7.)

</details>

11. Секрет по ошибке закоммичен в Git и отправлен. Что вы делаете в первую очередь?

<details>
<summary>Ответ</summary>

Отзываете его и выпускаете новый: считайте, что он уже скомпрометирован. Чистка истории — потом, и она не делает старое значение снова безопасным. (Урок 10.8.)

</details>

### Криптография и TLS (Уроки 10.9–10.11)

12. Чем отличаются хеш, MAC и цифровая подпись?

<details>
<summary>Ответ</summary>

У хеша нет ключа: он обнаруживает только случайные повреждения. MAC использует общий секретный ключ: верную метку могут создать и проверить только владельцы ключа. Подпись использует закрытый ключ для подписи и открытый для проверки: проверить может любой, подписать — только владелец. (Уроки 10.9 и 10.10.)

</details>

13. Почему метку MAC или токен нужно сравнивать за постоянное время?

<details>
<summary>Ответ</summary>

Обычное сравнение останавливается на первом различающемся байте, поэтому его время показывает, сколько байтов было верно. Атакующий может использовать это, чтобы угадать секрет байт за байтом. Используйте `verify_slice` или `subtle::ConstantTimeEq`. (Урок 10.9.)

</details>

14. Что такое nonce, секретен ли он и что будет, если использовать его повторно с тем же ключом?

<details>
<summary>Ответ</summary>

Число, которое должно быть уникальным для каждого сообщения при одном ключе. Он не секретен (хранится рядом с шифртекстом). Повторное использование позволяет атакующему сложить через XOR два шифртекста и узнать XOR открытых текстов, а для некоторых шифров ломает и аутентификацию. (Урок 10.10.)

</details>

15. Какие четыре проверки TLS-клиент делает для сертификата сервера? Что делает `curl -k`?

<details>
<summary>Ответ</summary>

Цепочка подписей заканчивается доверенным корнем; каждый сертификат в пределах срока действия; имя хоста есть в SAN; сертификат можно использовать как серверный. `-k` отключает проверку, а значит, убирает защиту от атаки «человек посередине». (Урок 10.11.)

</details>

### Память, цепочка поставок и проект (Уроки 10.12–10.14)

16. Назовите три проблемы безопасности, которые безопасный Rust **не** предотвращает.

<details>
<summary>Ответ</summary>

Любые три из: паники на плохом вводе (отказ в обслуживании), огромные выделения памяти по заявленным размерам, переполнение стека из-за глубокой рекурсии, «заворачивание» целочисленной арифметики в релизных сборках, логические ошибки (внедрение, отсутствие авторизации), ошибки в `unsafe`-коде или в зависимостях. (Урок 10.12.)

</details>

17. Чем отличаются `cargo audit` и `cargo deny`?

<details>
<summary>Ответ</summary>

`cargo audit` ищет крейты из `Cargo.lock` в базе бюллетеней RustSec (уязвимости, неподдерживаемые, некорректные). `cargo deny` проверяет четыре политики: бюллетени, лицензии, запреты и источники. (Урок 10.13.)

</details>

18. Почему в хранилище заголовок файла передаётся как присоединённые данные и почему одно сообщение об ошибке для «неверный пароль» и «файл изменён»?

<details>
<summary>Ответ</summary>

Заголовок (версия, параметры KDF, соль) должен быть покрыт меткой, иначе атакующий мог бы незаметно его изменить. Проверка метки в обоих случаях не проходит одинаково и не может их различить; более конкретное сообщение было бы лишь догадкой. (Урок 10.14.)

</details>

**Баллы:** 16–18: вы готовы к следующему этапу. 12–15: повторите уроки к вопросам, на которые не ответили (см. таблицу в конце). Меньше 12: повторите этап начиная с Урока 10.6, переделайте упражнения и пройдите тест снова через несколько дней.

## Практические задания

Для Заданий 1–3 положите код в файлы с указанными именами, запустите `cargo test` и закоммитьте каждое задание.

### Задание 1 — Токены сессии (Уроки 10.7, 10.9)

В `src/session.rs` напишите три функции:

- `pub fn new_token() -> String`: 32 случайных байта от операционной системы в виде 64 hex-символов;
- `pub fn token_hash(token: &str) -> String`: SHA-256 токена в hex. Это то, что хранит сервер: **никогда не сам токен**;
- `pub fn check(stored_hash: &str, presented: &str) -> bool`: хеширует предъявленный токен и сравнивает с сохранённым хешем **за постоянное время**.

Напишите тесты на: форму токена (длина, hex-цифры) и то, что два токена различаются; то, что сохранённый хеш не равен токену; то, что `check` принимает верный токен и отвергает другой токен, пустой токен и пустой сохранённый хеш.

<details>
<summary>Подсказка</summary>

`getrandom::fill`, `hex::encode`, `Sha256::digest` и `subtle::ConstantTimeEq::ct_eq(...)` с `.into()`, чтобы получить `bool`. Сравнивайте **хеши** (у обоих одинаковая длина), а не сами токены.

</details>

<details>
<summary>Ответ</summary>

```rust
use sha2::{Digest, Sha256};
use subtle::ConstantTimeEq;

/// A new session token: 32 random bytes as 64 hex characters.
pub fn new_token() -> String {
    let mut bytes = [0u8; 32];
    getrandom::fill(&mut bytes).expect("the operating system has no random numbers");
    hex::encode(bytes)
}

/// What the server stores: the SHA-256 of the token, in hex. The token itself is never stored.
pub fn token_hash(token: &str) -> String {
    hex::encode(Sha256::digest(token.as_bytes()))
}

/// Does `presented` match the stored hash? Compares in constant time.
pub fn check(stored_hash: &str, presented: &str) -> bool {
    let presented_hash = token_hash(presented);
    stored_hash.as_bytes().ct_eq(presented_hash.as_bytes()).into()
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn tokens_have_the_right_shape_and_differ() {
        let (a, b) = (new_token(), new_token());
        assert_eq!(a.len(), 64);
        assert!(a.chars().all(|c| c.is_ascii_hexdigit()));
        assert_ne!(a, b);
    }

    #[test]
    fn the_stored_hash_is_not_the_token() {
        let t = new_token();
        assert_ne!(token_hash(&t), t);
        assert_eq!(token_hash(&t).len(), 64);
    }

    #[test]
    fn check_accepts_only_the_right_token() {
        let t = new_token();
        let stored = token_hash(&t);
        assert!(check(&stored, &t));
        assert!(!check(&stored, &new_token()));
        assert!(!check(&stored, ""));
        assert!(!check("", &t));
    }
}
```

Почему это работает: токен сессии — это случайные данные с высокой энтропией, поэтому для их хранения хватает быстрого хеша (в отличие от пароля): если таблица утечёт, атакующий всё равно не сможет вернуться от хеша к рабочему токену. `ct_eq` на срезах разной длины просто возвращает «не равны», поэтому пустой сохранённый хеш отвергается.

</details>

### Задание 2 — Парсер, который никогда не паникует (Уроки 10.4, 10.12)

В `src/record.rs` напишите `pub fn parse_record(input: &[u8]) -> Result<(Record, usize), ParseError>` для двоичной записи: один байт `name_len`, затем `name_len` байтов имени, затем два байта `value_len` (big-endian), затем `value_len` байтов значения. Правила: имя состоит из 1–32 символов `a-z`, `0-9` и `_`; значение — не более 1024 байтов. Функция возвращает запись и число использованных байтов. У `Record` есть `name: String` и `value: Vec<u8>`; у `ParseError` — `Truncated`, `BadName` и `ValueTooLarge`.

Тесты: верная запись; **каждый префикс** верной записи — `Truncated` (никогда не паника); пустое имя, имя длиннее 32 байтов и имя с заглавной буквой — `BadName`; заявленная длина значения 65 535 — `ValueTooLarge`; 200 случайных вводов разной длины никогда не вызывают панику.

<details>
<summary>Подсказка</summary>

Никакой индексации `[i]`: используйте `split_first`, `get(..n)` и `ok_or(ParseError::Truncated)?`. Проверяйте лимиты **до** того, как нарезать срез. Добавьте `Debug` и `PartialEq` через derive к обоим типам, чтобы работал `assert_eq!`.

</details>

<details>
<summary>Ответ</summary>

```rust
/// A record: name_len (1 byte) | name | value_len (2 bytes, big-endian) | value
#[derive(Debug, PartialEq)]
pub struct Record {
    pub name: String,
    pub value: Vec<u8>,
}

#[derive(Debug, PartialEq)]
pub enum ParseError {
    Truncated,
    BadName,
    ValueTooLarge,
}

pub const MAX_NAME: usize = 32;
pub const MAX_VALUE: usize = 1024;

/// Parses one record from the start of `input`. Returns the record and the number of bytes used.
/// Never panics, whatever the input.
pub fn parse_record(input: &[u8]) -> Result<(Record, usize), ParseError> {
    let (&name_len, rest) = input.split_first().ok_or(ParseError::Truncated)?;
    let name_len = usize::from(name_len);
    if name_len == 0 || name_len > MAX_NAME {
        return Err(ParseError::BadName);
    }
    let name_bytes = rest.get(..name_len).ok_or(ParseError::Truncated)?;
    if !name_bytes.iter().all(|b| b.is_ascii_lowercase() || b.is_ascii_digit() || *b == b'_') {
        return Err(ParseError::BadName);
    }
    let rest = &rest[name_len..];
    let len_bytes = rest.get(..2).ok_or(ParseError::Truncated)?;
    let value_len = usize::from(u16::from_be_bytes([len_bytes[0], len_bytes[1]]));
    if value_len > MAX_VALUE {
        return Err(ParseError::ValueTooLarge);
    }
    let value = rest.get(2..2 + value_len).ok_or(ParseError::Truncated)?;
    let name = String::from_utf8(name_bytes.to_vec()).map_err(|_| ParseError::BadName)?;
    Ok((Record { name, value: value.to_vec() }, 1 + name_len + 2 + value_len))
}

#[cfg(test)]
mod tests {
    use super::*;

    fn good() -> Vec<u8> {
        let mut v = vec![4];
        v.extend_from_slice(b"user");
        v.extend_from_slice(&[0, 5]);
        v.extend_from_slice(b"alice");
        v
    }

    #[test]
    fn a_good_record_is_parsed() {
        let (record, used) = parse_record(&good()).unwrap();
        assert_eq!(record, Record { name: "user".into(), value: b"alice".to_vec() });
        assert_eq!(used, 12);
    }

    #[test]
    fn every_prefix_of_a_good_record_is_truncated_not_a_panic() {
        let bytes = good();
        for n in 0..bytes.len() {
            assert_eq!(parse_record(&bytes[..n]), Err(ParseError::Truncated), "prefix {n}");
        }
    }

    #[test]
    fn bad_names_and_huge_values_are_rejected() {
        assert_eq!(parse_record(&[0]), Err(ParseError::BadName));
        assert_eq!(parse_record(&[255; 300]), Err(ParseError::BadName));
        assert_eq!(parse_record(&[2, b'A', b'b', 0, 0]), Err(ParseError::BadName));
        // announces 65,535 bytes of value but delivers none
        assert_eq!(parse_record(&[1, b'a', 0xff, 0xff]), Err(ParseError::ValueTooLarge));
    }

    #[test]
    fn random_bytes_never_panic() {
        for len in 0..200 {
            let mut buf = vec![0u8; len];
            getrandom::fill(&mut buf).unwrap();
            let _ = parse_record(&buf);
        }
    }
}
```

Почему это работает: каждое обращение — это `get` или `split_first`, превращающие «не хватает байтов» в `Truncated`. Лимиты имени и значения проверяются до нарезки среза, поэтому заявленная длина 65 535 никогда не приводит к выделению памяти. Единственная индексация `rest[name_len..]` безопасна, потому что `get(..name_len)` только что доказал, что в `rest` не меньше `name_len` байтов. Тесты включают граничные случаи и случайный мусор.

</details>

### Задание 3 — Обёртки хеширования паролей (Уроки 10.6, 10.9)

В `src/password.rs` напишите `pub fn hash_password(password: &str) -> String` (Argon2id с настройками по умолчанию и новой случайной солью; результат — PHC-строка) и `pub fn verify_password(stored: &str, password: &str) -> bool`. Сохранённый текст, не являющийся корректным хешем, должен давать `false`, а не панику. Тесты: верный пароль проходит проверку, а неверный нет; хеш начинается с `$argon2id$`; один и тот же пароль даёт два разных хеша; некорректные тексты `"garbage"`, `""` и `"$argon2id$v=19$m=1,t=1,p=1$"` дают `false`.

<details>
<summary>Подсказка</summary>

Тот же API (Application Programming Interface, программный интерфейс), что в Уроке 10.6: `Argon2::default().hash_password(bytes)`, `PasswordHash::new(text)` из `argon2::password_hash::phc` и `verify_password`. Нужно импортировать оба трейта: `PasswordHasher` и `PasswordVerifier`.

</details>

<details>
<summary>Ответ</summary>

```rust
use argon2::Argon2;
use argon2::password_hash::phc::PasswordHash;
use argon2::password_hash::{PasswordHasher, PasswordVerifier};

/// Hashes a password with Argon2id and a fresh random salt. Returns the PHC string.
pub fn hash_password(password: &str) -> String {
    Argon2::default().hash_password(password.as_bytes()).expect("hashing failed").to_string()
}

/// True only if `password` matches `stored`. A malformed `stored` text is `false`, not a panic.
pub fn verify_password(stored: &str, password: &str) -> bool {
    match PasswordHash::new(stored) {
        Ok(parsed) => Argon2::default().verify_password(password.as_bytes(), &parsed).is_ok(),
        Err(_) => false,
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn right_password_verifies_and_wrong_does_not() {
        let stored = hash_password("correct horse battery");
        assert!(stored.starts_with("$argon2id$"));
        assert!(verify_password(&stored, "correct horse battery"));
        assert!(!verify_password(&stored, "correct horse batterY"));
    }

    #[test]
    fn the_same_password_gives_different_hashes() {
        assert_ne!(hash_password("pw"), hash_password("pw"));
    }

    #[test]
    fn a_malformed_stored_hash_is_false_not_a_panic() {
        assert!(!verify_password("garbage", "pw"));
        assert!(!verify_password("", "pw"));
        assert!(!verify_password("$argon2id$v=19$m=1,t=1,p=1$", "pw"));
    }
}
```

Почему это работает: сохранённый текст — это данные из базы, поэтому ему тоже нельзя доверять: ошибка разбора превращается в `false`. `verify_password` сравнивает за постоянное время внутри библиотеки. Вместе три задания дают `test result: ok. 10 passed` (3 + 4 + 3 теста).

</details>

### Задание 4 — Проверка собственного хранилища (без нового кода)

Проверьте `code/p10-vault` так, как это сделал бы атакующий. Запишите ответы в `code/p10-vault/SECURITY.md` и закоммитьте.

1. Запустите `cargo audit` и `cargo deny check`. Вставьте последнюю строку каждого. Исправьте каждую уязвимость или запишите, почему вы её игнорируете и до какого срока.
2. Перечислите каждое место, где секрет (пароль, ключ, открытый текст, значение секрета) хранится в памяти. Для каждого: лежит ли он в `Zeroizing` (или затирается вручную)? Печатает ли его какой-нибудь `Debug`, `println!` или сообщение об ошибке?
3. Попробуйте пять враждебных файлов: пустой, из одного байта, из 200 случайных байтов, корректное хранилище с одним перевёрнутым байтом и хранилище с `m_cost`, равным `0xFFFFFFFF` в заголовке. Что печатает каждый и каков код возврата?
4. Запишите модель угроз в шесть строк: три вещи, от которых хранилище защищает, и три, от которых нет.
5. Назовите единственное изменение, которое больше всего улучшило бы безопасность хранилища, и почему вы его пока не сделали.

<details>
<summary>Подсказка</summary>

Для пункта 3 используйте `printf` и `dd` или запишите файлы из небольшого теста. Команды из M4 и M5 Урока 10.14 дают раскладку файла (поле `m_cost` — байты с 5 по 8).

</details>

<details>
<summary>Ответ</summary>

Единственно верного текста нет, но хорошая проверка содержит:

- чистые `cargo audit` и `cargo deny check` (`advisories ok, bans ok, licenses ok, sources ok`) либо каждое исключение с причиной и датой;
- таблицу секретов в памяти: мастер-пароль, полученный ключ, байты расшифрованного JSON (JavaScript Object Notation, нотация объектов JavaScript) и значение, прочитанное со стандартного ввода, лежат в `Zeroizing`; `Vault` затирает свои значения в `Drop`; ни у одного типа с секретами нет `derive(Debug)`;
- для пяти враждебных файлов — строку `error:` и код возврата `1` у каждого и ни одного `panicked`; абсурдный `m_cost` отвергается как недопустимые параметры **до** запуска Argon2;
- модель угроз, как таблица в Уроке 10.14;
- честное «следующее улучшение», например: файл-ключ как второй фактор, `mlock` или ограничение числа попыток ввода пароля через более медленный `t_cost`.

Почему это работает: так выглядит настоящая проверка безопасности: вы атакуете собственную работу, записываете доказательства и фиксируете то, что принимаете.

</details>

## Если что-то не получилось

| Тема | Куда вернуться |
|---|---|
| Триада CIA, поверхность атаки, принципы безопасности | [10.1 Мышление безопасности и триада CIA](01-security-mindset-and-the-cia-triad.md) |
| STRIDE, оценка риска, деревья атак | [10.2 Моделирование угроз по STRIDE](02-threat-modeling-with-stride.md) |
| Контроль доступа, неверная конфигурация, fail closed | [10.3 OWASP Top 10](03-the-owasp-top-10.md) |
| Списки разрешённого, разбор в типы, лимиты | [10.4 Проверка ввода: никогда не доверяйте вводу](04-input-validation.md) |
| SQL-внедрение, внедрение команд и путей | [10.5 Атаки внедрения: SQL, команды и пути](05-injection-attacks.md) |
| Argon2id, соли, ошибки входа | [10.6 Аутентификация и хеширование паролей](06-authentication-and-password-hashing.md) |
| Авторизация, сессии, cookie, JWT | [10.7 Авторизация, сессии и токены](07-authorization-sessions-and-tokens.md) |
| Секреты, `secrecy`, `zeroize`, утечки | [10.8 Управление секретами](08-secrets-management.md) |
| Хеши, HMAC, постоянное время, случайность | [10.9 Криптография I: хеши, MAC и случайность](09-cryptography-hashes-macs-and-randomness.md) |
| AEAD, nonce, подписи | [10.10 Криптография II: шифрование и подписи](10-cryptography-encryption-and-signatures.md) |
| TLS, сертификаты, `curl -v`, `openssl s_client` | [10.11 TLS и сертификаты](11-tls-and-certificates.md) |
| Паники как DoS, лимиты, проверка `unsafe`, Miri, ASan | [10.12 Безопасность памяти и проверка небезопасного кода](12-memory-safety-and-reviewing-unsafe.md) |
| `cargo audit`, `cargo deny`, привычки с зависимостями | [10.13 Безопасность цепочки поставок: cargo-audit и cargo-deny](13-supply-chain-security.md) |
| Формат файла, Argon2 + AEAD вместе, укрепление | [10.14 Проект: зашифрованное хранилище секретов](14-project-encrypted-secrets-vault.md) |

## Что дальше

Вы закончили Этап 10. Теперь вы умеете думать как атакующий о собственных проектах, проверять ввод, избегать внедрения, хранить пароли и секреты, пользоваться стандартными криптографическими «кирпичиками», читать TLS-соединение, проверять `unsafe`-код и контролировать свои зависимости. Прежде чем идти дальше:

1. Слейте ветку контрольной точки (`git switch main && git merge lesson/10-15-checkpoint`) и удалите завершённые ветки, которые больше не нужны.
2. Сохраните `SECURITY.md` и модель угроз хранилища: вы захотите обновить их, когда будете его развёртывать.
3. Добавьте несколько строк в свои учебные заметки: какие атаки вас удивили и какую привычку вы начнёте применять уже сейчас (например, `cargo audit` в CI, Continuous Integration).

В **[Этапе 11 — Linux, командная оболочка и контейнеры](../11-linux-shell-and-containers/README.md)** вы перейдёте к самой операционной системе: виртуальная машина Linux на вашем Mac, файловая система, текстовые утилиты, пользователи и права доступа (`0600` файла хранилища обретёт полный смысл), процессы, службы, `bash`, SSH, сетевые команды и Docker. Многие идеи безопасности этого этапа вернутся там: принцип наименьших привилегий, секреты в контейнерах и безопасная обработка недоверенного ввода в скриптах.

## Источники и что почитать

- [OWASP Top 10](https://owasp.org/www-project-top-ten/) — список категорий риска, на котором построен Урок 10.3.
- [OWASP Cheat Sheet Series](https://cheatsheetseries.owasp.org/) — короткие практические руководства по каждой теме этого этапа.
- [The Rust Secure Code Working Group](https://github.com/rust-secure-code/wg) — руководства, инструменты и бюллетени для безопасного Rust.
- [База бюллетеней RustSec](https://rustsec.org/) — уязвимости и неподдерживаемые крейты в экосистеме Rust.
- [Cryptographic Right Answers](https://www.latacora.com/blog/2018/04/03/cryptographic-right-answers/) — короткая статья о том, какой примитив выбирать для какой задачи.
- [The Rustonomicon](https://doc.rust-lang.org/nomicon/) — официальное руководство по `unsafe` Rust.

---

[🏠 Начало](../START-HERE.md) · [← 10.14 Проект: зашифрованное хранилище секретов](14-project-encrypted-secrets-vault.md) · [📋 Этап 10](README.md) · [Этап 11 — Linux, командная оболочка и контейнеры →](../11-linux-shell-and-containers/README.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
