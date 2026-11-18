<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 10.7 Авторизация, сессии и токены](07-authorization-sessions-and-tokens.md) · [📋 Этап 10](README.md) · [10.9 Криптография I: хеши, MAC и случайность →](09-cryptography-hashes-macs-and-randomness.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/10-security-fundamentals/08-secrets-management.md) <!-- nav -->

# Урок 10.8 — Управление секретами

> **Этап 10 — Основы безопасности** · Урок 8 из 15 · ⏱ около 135 минут

## Что вы узнаете

- Что считать секретом и в каких местах секреты утекают чаще всего.
- Как программа должна получать секреты: файлы, переменные окружения, но никогда не код и не аргументы командной строки.
- Как держать секрет вне журналов с помощью крейта `secrecy` и как стирать память с помощью `zeroize`.
- Как безопасно загрузить секрет, включая проверку прав доступа к файлу, с тестами.
- Что делать, когда секрет утёк: сначала заменить, потом убрать.

## Перед началом

- Вы прошли [Урок 10.7 — Авторизация, сессии и токены](07-authorization-sessions-and-tokens.md).
- Вы знаете правила о секретах и Git из [Урока 01.3](../01-git-and-version-control/03-your-first-repository-and-commits.md) и [Урока 01.5](../01-git-and-version-control/05-undoing-changes-safely.md), а также настройку через окружение из [Урока 08.15](../08-software-design/15-the-twelve-factor-app.md).
- Docker Desktop запущен (один небольшой эксперимент использует контейнер Linux). Контейнер назван с префиксом `course-` и удаляется автоматически по завершении (`--rm`).
- Создайте пакет: `cargo new --lib code/p10-secrets`, затем внутри него `cargo add secrecy@0.10 zeroize@1.9`, `cargo add --dev tempfile@3.27` и `mkdir src/bin`.

## Новые слова

| Слово | Значение |
|---|---|
| **секрет** (secret) | Значение, дающее доступ или власть тому, кто его знает: пароль, ключ API, токен, закрытый ключ, строка подключения к базе данных. |
| **ключ API** (API key, Application Programming Interface key) | Секретный текст, который идентифицирует и авторизует программу, когда она обращается к другому сервису. |
| **переменная окружения** (environment variable) | Именованное текстовое значение, которое операционная система передаёт каждой запускаемой программе (`HOME`, `PATH`). Программы читают их через `std::env::var`. |
| **процесс** (process) | Работающая программа. |
| **ротация** (rotation) | Замена секрета новым, по расписанию или после утечки, и отмена старого. |
| **отозвать** (revoke) | Отменить секрет в сервисе, который его выдал, чтобы он перестал работать. |
| **менеджер секретов** (secret manager) | Сервис, который хранит секреты в зашифрованном виде, управляет тем, кто может их читать, и записывает каждый доступ (примеры: HashiCorp Vault, облачные сервисы ключей, Kubernetes Secrets с шифрованием). |
| **режим файла** (file mode, права доступа) | Числа, которые говорят, кто может читать, записывать или запускать файл. `0o600` означает: владелец может читать и писать, больше никто ничего не может. |
| **обнулить** (zeroize) | Перезаписать память нулями, чтобы секрет не оставался в ней после использования. |
| **радиус поражения** (blast radius) | Сколько вреда может нанести один утёкший секрет. Чем меньше, тем лучше. |
| **CI** (Continuous Integration, непрерывная интеграция) | Сервер, который автоматически запускает ваши проверки и сборки при каждой отправке (Этап 09). |
| **PID** (Process IDentifier, идентификатор процесса) | Число, которое операционная система присваивает работающей программе. |
| **root** | Всемогущий пользователь-администратор системы Linux. |

## 1. Что такое секрет и где он утекает?

**Секрет** — любое значение, дающее доступ: пароли, ключи API, токены (Урок 10.7), ключ подписи JWT (JSON Web Token, веб-токен в формате JSON), закрытые
ключи, пароли баз данных. Обратите внимание, что секретом **не** является: имя пользователя, адрес сервера,
используемый алгоритм (открытый дизайн, Урок 10.1).

Секреты редко утекают из-за хитрых атак. Они утекают через обычные места:

| Место | Почему утекает |
|---|---|
| **Исходный код** | Каждый, кто может читать репозиторий, может читать ключ, навсегда ([Урок 01.3](../01-git-and-version-control/03-your-first-repository-and-commits.md)). |
| **История Git** | Удалённый файл остаётся в истории ([Урок 01.5](../01-git-and-version-control/05-undoing-changes-safely.md)). |
| **Журналы и сообщения об ошибках** | `{:?}` печатает каждое поле (Урок 10.1). Журналы копируют, пересылают и читают многие люди. |
| **Аргументы командной строки** | Список процессов показывает их другим пользователям машины. |
| **Переменные окружения** | Дочерние процессы наследуют их; они попадают в отчёты о падениях, страницы отладки и, в Linux, в `/proc/<pid>/environ`. |
| **Образы контейнеров** | `ENV` и `COPY` записывают секрет в слой, который может прочитать любой, у кого есть образ. |
| **Резервные копии, чаты, тикеты, скриншоты** | Люди вставляют секреты туда, куда не следует. |

Небольшой эксперимент делает это наглядным. Он выполняется в контейнере Linux, который удаляется в конце, а
«секреты» — выдуманные значения.

```bash
docker run --rm --name course-10-08-leaks ubuntu:26.04 bash -c '
bash -c "sleep 60; true" myapp --password=hunter2-demo &
sleep 1
export APP_TOKEN=demo-token-12345
bash -c "sleep 60; true" &
CH=$!
sleep 1
ps -eo pid,args | grep -v grep | grep myapp
tr "\0" "\n" < /proc/$CH/environ | grep APP_TOKEN
umask 077; echo -n s3cret > /tmp/secret.txt; ls -l /tmp/secret.txt | cut -c1-10
'
```

```text
      7 bash -c sleep 60; true myapp --password=hunter2-demo
APP_TOKEN=demo-token-12345
-rw-------
```

**Что происходит:**

1. Первая команда `bash -c "sleep 60; true" myapp --password=hunter2-demo` имитирует программу, запущенную как
   `myapp --password=...`. Команда `ps -eo pid,args` перечисляет каждый процесс **вместе с его аргументами**, поэтому
   пароль виден каждому пользователю машины. (На общем сервере это настоящая утечка.)
2. Второй процесс был запущен с `APP_TOKEN` в окружении. Файл `/proc/<pid>/environ` показывает окружение процесса в
   Linux. Его могут прочитать владелец процесса и `root`; так же и всё, что наследует окружение.
3. После `umask 077` (настройка, которая убирает все права для группы и для остальных у новых файлов) новый файл
   имеет режим `-rw-------`: прочитать его может только владелец. Это самый безопасный из трёх вариантов.

Секрет, записанный в образ контейнера, ещё хуже: `ENV API_KEY=...` в Dockerfile остаётся в метаданных образа, а
`docker inspect` и `docker history` показывают его любому, кто может скачать образ. Образы вы будете строить
правильно в [Уроке 11.13](../11-linux-shell-and-containers/13-dockerfiles-for-rust.md); держите секреты вне них.

## 2. Как программа должна получать секрет

От лучшего к худшему:

1. **Менеджер секретов или платформа** выдаёт программе недолговечный секрет или монтирует его как **файл**
   внутри контейнера (так делают секреты Docker и Kubernetes Secrets). Программа получает только путь.
2. **Файл** с режимом `0600` вне репозитория. Программа проверяет режим перед чтением.
3. **Переменная окружения**, заданная платформой при запуске. Это просто и распространено, но см. таблицу выше.
4. **Никогда:** исходный код, закоммиченный файл `.env`, аргумент командной строки, слой образа.

Хорошая привычка из Docker: программа принимает **и** `NAME`, **и** `NAME_FILE`. Если задан `NAME_FILE`, секрет
читается из этого файла; иначе из `NAME`. Тогда одна и та же программа работает на вашем ноутбуке (переменная) и в
рабочей среде (смонтированный файл).

## 3. Загрузка секрета в Rust

Начнём с главного риска — случайной печати. Сохраните `src/bin/leaky.rs`:

```rust
use secrecy::{ExposeSecret, SecretString};

#[derive(Debug)]
struct PlainConfig {
    user: String,
    password: String,
}

#[derive(Debug)]
struct SafeConfig {
    user: String,
    password: SecretString,
}

fn main() {
    let plain = PlainConfig {
        user: "app".into(),
        password: "hunter2-demo".into(),
    };
    println!(
        "user {} has a password of {} characters",
        plain.user,
        plain.password.len()
    );
    println!("plain: {plain:?}");

    let safe = SafeConfig {
        user: "app".into(),
        password: SecretString::from("hunter2-demo"),
    };
    println!("safe:  {safe:?}");

    // To use the value you must say so, and the word is easy to search for in a code review:
    println!("length: {}", safe.password.expose_secret().len());
    println!("user: {}", safe.user);
}
```

```text
user app has a password of 12 characters
plain: PlainConfig { user: "app", password: "hunter2-demo" }
safe:  SafeConfig { user: "app", password: SecretBox<str>([REDACTED]) }
length: 12
user: app
```

**Что происходит:**

1. `PlainConfig` хранит пароль как `String`. `#[derive(Debug)]` его печатает (как в Уроке 10.1). У `SafeConfig` та же
   форма, но пароль — `SecretString`.
2. `SecretString` из крейта `secrecy` — обёртка над текстом. Её `Debug` печатает только `[REDACTED]`. У неё **нет**
   `Display`, и она не реализует `Serialize` или `Clone`, пока вы этого не попросите, поэтому её трудно случайно
   скопировать или напечатать. Она также перезаписывает текст нулями, когда её удаляют.
3. Чтобы прочитать значение, нужно вызвать `expose_secret()` (из трейта `ExposeSecret`). Это слово легко искать при
   код-ревью: каждое использование — место, где нужно спросить «это безопасно?». Держите такие места немногочисленными
   и короткими.

Теперь библиотека. Сохраните `src/lib.rs`:

```rust
use std::fmt;
use std::fs;
use std::path::PathBuf;

use secrecy::SecretString;

#[derive(Debug, PartialEq, Eq)]
pub enum SecretError {
    Missing(String),
    Unreadable(PathBuf),
    TooPermissive(PathBuf),
    Empty(String),
}

// The messages name the secret and the file, never the value.
impl fmt::Display for SecretError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            SecretError::Missing(name) => {
                write!(f, "secret {name} is not set (set {name} or {name}_FILE)")
            }
            SecretError::Unreadable(path) => write!(f, "cannot read {}", path.display()),
            SecretError::TooPermissive(path) => write!(
                f,
                "{} can be read by other users; run: chmod 600 {}",
                path.display(),
                path.display()
            ),
            SecretError::Empty(name) => write!(f, "secret {name} is empty"),
        }
    }
}

impl std::error::Error for SecretError {}

/// A secret file must not be readable by the group or by everybody.
#[cfg(unix)]
fn check_permissions(path: &PathBuf) -> Result<(), SecretError> {
    use std::os::unix::fs::PermissionsExt;
    let meta = fs::metadata(path).map_err(|_| SecretError::Unreadable(path.clone()))?;
    if meta.permissions().mode() & 0o077 != 0 {
        return Err(SecretError::TooPermissive(path.clone()));
    }
    Ok(())
}

#[cfg(not(unix))]
fn check_permissions(_path: &PathBuf) -> Result<(), SecretError> {
    Ok(())
}

/// Finds the secret `name`. `env` looks up an environment variable (a parameter, so tests need no real variables).
/// Order: 1. the file named by `{name}_FILE`   2. the variable `{name}`.
pub fn load_secret(
    name: &str,
    env: impl Fn(&str) -> Option<String>,
) -> Result<SecretString, SecretError> {
    let value = if let Some(path) = env(&format!("{name}_FILE")) {
        let path = PathBuf::from(path);
        check_permissions(&path)?;
        let text = fs::read_to_string(&path).map_err(|_| SecretError::Unreadable(path.clone()))?;
        text.trim_end_matches(['\n', '\r']).to_string()
    } else if let Some(value) = env(name) {
        value
    } else {
        return Err(SecretError::Missing(name.to_string()));
    };
    if value.is_empty() {
        return Err(SecretError::Empty(name.to_string()));
    }
    Ok(SecretString::from(value))
}

/// Settings of a program. `Debug` is safe: the password prints as [REDACTED].
#[derive(Debug)]
pub struct DbConfig {
    pub user: String,
    pub password: SecretString,
}
```

**Что происходит:**

1. У `SecretError` четыре случая. Его текст `Display` называет секрет и файл, но **никогда не значение**. Сообщение даже
   говорит, как исправить проблему (`chmod 600`).
2. `check_permissions` компилируется только на Unix (`#[cfg(unix)]`; macOS и Linux). `mode() & 0o077` оставляет шесть
   битов прав **группы** и **остальных**; если хоть один установлен, файл могут прочитать другие, и мы отказываем. На
   других системах вторая версия функции ничего не делает.
3. `load_secret` принимает имя и функцию `env`, которая ищет переменные. В `main` мы передаём настоящий
   `std::env::var`; в тестах — поддельную. (Установка настоящих переменных окружения в тестах небезопасна в Rust 2024
   и заставляет тесты мешать друг другу, поэтому внедрение функции поиска — более чистый дизайн. Это та же идея,
   что передача `now` в Уроке 10.7.)
4. Сначала она пробует `{name}_FILE`. Файл проверяется, читается, и завершающий перевод строки убирается (редакторы
   и `echo` его добавляют; пароль со скрытым переводом строки — классическая ошибка). Иначе используется
   переменная `{name}`, а если нет ни того ни другого, возвращается `Missing`.
5. Пустое значение отклоняется: почти всегда это означает ошибку в конфигурации.
6. Результат — `SecretString`, так что у секрета есть безопасная обёртка с первого момента.
7. `DbConfig` хранит обычный `user` и секретный `password`. Его производный `Debug` теперь безопасен.

Сохраните `src/bin/load.rs`; он читает `DB_PASSWORD` или `DB_PASSWORD_FILE` из настоящего окружения:

```rust
use p10_secrets::{DbConfig, SecretError, load_secret};
use secrecy::ExposeSecret;

fn run() -> Result<(), SecretError> {
    let env = |name: &str| std::env::var(name).ok();

    let config = DbConfig {
        user: String::from("app"),
        password: load_secret("DB_PASSWORD", env)?,
    };

    println!("loaded: {config:?}");
    println!(
        "the password has {} characters",
        config.password.expose_secret().len()
    );
    Ok(())
}

fn main() {
    if let Err(error) = run() {
        eprintln!("error: {error}");
        std::process::exit(1);
    }
}
```

Соберите его, затем попробуйте четыре ситуации. Каждая команда — отдельный запуск, а выводы перечислены ниже в том же порядке:

```bash
cargo build --bin load
./target/debug/load
DB_PASSWORD=hunter2-demo ./target/debug/load
mkdir -p sandbox && printf 'hunter2-demo\n' > sandbox/db_password && chmod 644 sandbox/db_password
DB_PASSWORD_FILE=sandbox/db_password ./target/debug/load
chmod 600 sandbox/db_password
DB_PASSWORD_FILE=sandbox/db_password ./target/debug/load
rm -r sandbox
```

```text
error: secret DB_PASSWORD is not set (set DB_PASSWORD or DB_PASSWORD_FILE)
loaded: DbConfig { user: "app", password: SecretBox<str>([REDACTED]) }
the password has 12 characters
error: sandbox/db_password can be read by other users; run: chmod 600 sandbox/db_password
loaded: DbConfig { user: "app", password: SecretBox<str>([REDACTED]) }
the password has 12 characters
```

**Что происходит:**

1. Когда ничего не задано, программа завершается при запуске с понятным сообщением и кодом возврата 1. Она не
   работает «наполовину настроенной».
2. С `DB_PASSWORD` секрет загружается. `{config:?}` показывает `[REDACTED]`. В пароле 12 символов (завершающий
   перевод строки в случае файла убран).
3. Файл с режимом `644` отклоняется: другие пользователи машины могли бы его прочитать.
4. После `chmod 600` файл принимается.

Наконец, тесты, в конце `src/lib.rs`:

```rust
#[cfg(test)]
mod tests {
    use std::collections::HashMap;

    use secrecy::ExposeSecret;

    use super::*;

    type EnvFn = Box<dyn Fn(&str) -> Option<String>>;

    fn env_of(pairs: &[(&str, &str)]) -> EnvFn {
        let map: HashMap<String, String> = pairs
            .iter()
            .map(|(k, v)| (k.to_string(), v.to_string()))
            .collect();
        Box::new(move |name| map.get(name).cloned())
    }

    #[test]
    fn a_missing_secret_is_an_error_that_names_it() {
        let error = load_secret("DB_PASSWORD", env_of(&[])).unwrap_err();
        assert_eq!(error, SecretError::Missing("DB_PASSWORD".into()));
        assert!(error.to_string().contains("DB_PASSWORD_FILE"));
    }

    #[test]
    fn the_variable_works_and_empty_is_refused() {
        let secret = load_secret("K", env_of(&[("K", "value")])).unwrap();
        assert_eq!(secret.expose_secret(), "value");
        assert_eq!(
            load_secret("K", env_of(&[("K", "")])).unwrap_err(),
            SecretError::Empty("K".into())
        );
    }

    #[cfg(unix)]
    #[test]
    fn a_file_wins_over_the_variable_and_must_be_private() {
        use std::os::unix::fs::PermissionsExt;

        let dir = tempfile::tempdir().unwrap();
        let path = dir.path().join("secret.txt");
        fs::write(&path, "from-file\n").unwrap();
        let path_text = path.to_str().unwrap().to_string();
        let env = env_of(&[("K_FILE", &path_text), ("K", "from-variable")]);

        fs::set_permissions(&path, fs::Permissions::from_mode(0o644)).unwrap();
        assert_eq!(
            load_secret("K", &env).unwrap_err(),
            SecretError::TooPermissive(path.clone())
        );

        fs::set_permissions(&path, fs::Permissions::from_mode(0o600)).unwrap();
        let secret = load_secret("K", &env).unwrap();
        assert_eq!(secret.expose_secret(), "from-file"); // the newline was removed
    }

    #[test]
    fn debug_output_never_contains_the_secret() {
        let config = DbConfig {
            user: "app".into(),
            password: SecretString::from("hunter2-demo"),
        };
        let shown = format!("{config:?} {config:#?}");
        assert!(!shown.contains("hunter2"));
        assert!(shown.contains("REDACTED"));
    }
}
```

```bash
cargo test
```

```text
running 4 tests
test tests::a_missing_secret_is_an_error_that_names_it ... ok
test tests::debug_output_never_contains_the_secret ... ok
test tests::the_variable_works_and_empty_is_refused ... ok
test tests::a_file_wins_over_the_variable_and_must_be_private ... ok

test result: ok. 4 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

**Что происходит:** вспомогательная функция `env_of` строит поддельное окружение из списка пар, упакованное как
функция. Тесты покрывают отсутствующий секрет (ошибка называет `DB_PASSWORD_FILE`), хорошую переменную и пустую,
файл, который побеждает переменную и отклоняется при режиме `644`, но принимается при `600` (с убранным переводом
строки; `tempfile` создаёт папку, которая потом удаляется), и, самое важное, что вывод `Debug` конфигурации никогда не
содержит секрет, как в компактной форме `{:?}`, так и в красивой `{:#?}`.

> 🧪 **Тестирование:** тест, который форматирует ваши типы с секретами и проверяет, что секрета **нет** в выводе, —
> дешёвая защита от самой частой утечки. Пишите такой для каждого типа, хранящего секрет.

## 4. Стирание памяти с помощью `zeroize`

Когда программа закончила работу с секретом, байты остаются в памяти, пока что-то их не перезапишет. Дамп памяти,
файл подкачки или ошибка, читающая освобождённую память, могут их раскрыть. Крейт `zeroize` перезаписывает память
так, что компилятору **не разрешено это оптимизировать** (обычный цикл `for`, записывающий нули, может быть удалён,
потому что после него эту память никто не читает). Сохраните `src/bin/zero.rs`:

```rust
use zeroize::{Zeroize, Zeroizing};

fn main() {
    // Explicit wiping of a buffer:
    let mut key = [7u8; 8];
    println!("before: {key:?}");
    key.zeroize();
    println!("after:  {key:?}");

    // `Zeroizing<T>` wipes the value for you when it goes out of scope.
    {
        let token = Zeroizing::new(String::from("temporary-token"));
        println!(
            "inside the block the value is usable: {} bytes",
            token.len()
        );
    } // <- the memory of `token` is overwritten with zeros here
    println!("the block ended");
}
```

```text
before: [7, 7, 7, 7, 7, 7, 7, 7]
after:  [0, 0, 0, 0, 0, 0, 0, 0]
inside the block the value is usable: 15 bytes
the block ended
```

**Что происходит:**

1. `key.zeroize()` (из трейта `Zeroize`) перезаписывает все восемь байтов нулями.
2. `Zeroizing::new(value)` оборачивает любое значение, которое можно обнулить. Когда обёртка удаляется в конце
   блока, она стирает значение за вас. Она ведёт себя как внутреннее значение (`token.len()` работает).

> ⚠️ **Частая ошибка:** считать, что `zeroize` делает секреты нечитаемыми в памяти. Он стирает **ту копию, которую вы
> держите**. Копии, сделанные раньше (`String`, который вырос и переместился, буфер в сетевой библиотеке, файл
> подкачки операционной системы), не стираются. Считайте это одним слоем эшелонированной защиты, а не гарантией.
> Лучшая стратегия — держать секреты недолго, в немногих местах и никогда не клонировать их. `SecretString` уже
> использует `zeroize` за вас.

## 5. Безопасная запись файла с секретом

Если ваша программа создаёт файл с секретом, создавайте его **приватным с первого байта**: если сначала создать файл,
а потом вызвать `chmod`, другие пользователи смогут прочитать его в промежутке. Упражнение 3 просит вас написать эту
функцию.

## 6. Когда секрет утёк

Относитесь к любой утечке как к краже, даже если «никто не видел». Выполняйте эти шаги **в таком порядке**:

1. **Отзовите и замените** секрет в сервисе (удалите ключ API, смените пароль, смените ключ подписи). Это
   единственный шаг, который действительно останавливает вред.
2. **Выясните, что было раскрыто**: проверьте журналы сервиса на использование старого ключа.
3. **Уберите секрет** оттуда, где он утёк (код, история с помощью инструмента вроде `git filter-repo`, чат). Это
   уборка, а не лекарство: копии могут уже существовать.
4. **Извлеките урок**: добавьте проверку, которая поймала бы это.

**Профилактика** стоит дёшево: добавьте `.env`, `*.pem` и `*.key` в `.gitignore` до первого коммита; закоммитьте
`.env.example` с поддельными значениями; включите сканирование секретов на вашем хостинге Git (GitHub делает это для
известных форматов токенов) и запускайте сканер вроде `gitleaks` в CI ([Урок 09.12](../09-testing-debugging-benchmarking/12-continuous-integration.md))
или как хук pre-commit.

## 7. Хорошие привычки

- **Один секрет на цель и на окружение**, чтобы утёкший тестовый ключ не открывал рабочую среду (малый **радиус поражения**).
- **Наименьшие привилегии**: учётной записи базы данных приложения нужны только его собственные таблицы.
- **Меняйте** по расписанию и после любой утечки и предпочитайте **недолговечные** учётные данные.
- **Не пишите в журнал, не печатайте и не передавайте секреты в URL** (URL хранятся в журналах и истории браузера) и
  не вставляйте их в чат.

## Частые ошибки

> ⚠️ **Частая ошибка:** `println!("{:?}", config)` для типа, хранящего секретное поле типа `String`.

> ⚠️ **Частая ошибка:** base64 — это не шифрование. Kubernetes Secret или переменную CI, которая всего лишь в Base64,
> может прочитать любой, кто имеет к ней доступ. Убедитесь, что ваша платформа шифрует секреты при хранении и
> ограничивает тех, кто может их читать.

## Попробуйте сами

1. Наберите библиотеку, три программы и тесты. Запустите `cargo test` и эксперименты из раздела 3.
2. Запустите `load` с `DB_PASSWORD=""` (пустое значение). Какую ошибку вы получите?
3. Напишите собственную структуру с полем `api_key: SecretString`, добавьте derive `Debug` и докажите тестом, что
   `{:#?}` его скрывает.

## Упражнения

### Упражнение 1 — Где это должно жить? (легко)

Выберите лучшее место для каждого секрета и объясните почему: (a) пароль базы данных для программы в контейнере на
сервере; (b) тестовый ключ API для локальной разработки; (c) ключ подписи JWT, общий для всех сервисов; (d) пароль,
нужный скрипту один раз.

<details>
<summary>Подсказка</summary>

Используйте порядок из раздела 2. Подумайте, кто может читать каждое место.

</details>

<details>
<summary>Ответ</summary>

(a) Файл, смонтированный платформой (секрет Docker или Kubernetes), читаемый через `DB_PASSWORD_FILE`, а не слой
образа. (b) Переменная окружения или файл `0600` вне репозитория, с закоммиченным `.env.example` из поддельных
значений. (c) Менеджер секретов, доставляющий его каждому сервису как файл или при запуске, с планом ротации.
(d) Запросить его при запуске, не показывая вводимые символы на экране, или прочитать из файла `0600`; никогда не передавать в аргументе, потому что
он будет виден в списке процессов.

</details>

### Упражнение 2 — Отсутствующий файл (средне)

Напишите **тест**, доказывающий, что `load_secret` возвращает `SecretError::Unreadable(path)` (и не паникует), когда
`NAME_FILE` указывает на несуществующий файл.

<details>
<summary>Подсказка</summary>

Используйте `env_of(&[("K_FILE", "/definitely/not/here")])`. `check_permissions` вызывает `fs::metadata`, которая
завершается ошибкой.

</details>

<details>
<summary>Ответ</summary>

```rust
    #[test]
    fn a_missing_file_is_unreadable_not_a_panic() {
        let env = env_of(&[("K_FILE", "/definitely/not/here")]);
        assert_eq!(
            load_secret("K", env).unwrap_err(),
            SecretError::Unreadable(PathBuf::from("/definitely/not/here"))
        );
    }
```

Поместите его внутрь `mod tests`. Почему это работает: проверка прав сначала запрашивает у файловой системы данные о
файле и превращает сбой в `Unreadable`, поэтому программа сообщает понятную ошибку, а не падает.

</details>

### Упражнение 3 — Приватный файл с первого байта (сложно)

Напишите `write_secret_file(path: &Path, text: &str) -> io::Result<()>` для Unix. Он должен создавать файл с режимом
`0o600` и **завершаться ошибкой**, если файл уже существует. Напишите **тест**, проверяющий режим и то, что второй
вызов завершается ошибкой и оставляет содержимое неизменным.

<details>
<summary>Подсказка</summary>

`std::fs::OpenOptions::new().write(true).create_new(true).mode(0o600).open(path)`; метод `mode` приходит из
`std::os::unix::fs::OpenOptionsExt`. Прочитайте режим обратно через `metadata(...).permissions().mode() & 0o777`.

</details>

<details>
<summary>Ответ</summary>

```rust
#[cfg(unix)]
pub fn write_secret_file(path: &std::path::Path, text: &str) -> std::io::Result<()> {
    use std::io::Write;
    use std::os::unix::fs::OpenOptionsExt;

    let mut file = fs::OpenOptions::new()
        .write(true)
        .create_new(true) // never overwrite an existing file
        .mode(0o600) // owner: read and write; group and others: nothing
        .open(path)?;
    file.write_all(text.as_bytes())
}
```

```rust
    #[cfg(unix)]
    #[test]
    fn secret_files_are_created_private_and_never_overwritten() {
        use std::os::unix::fs::PermissionsExt;

        let dir = tempfile::tempdir().unwrap();
        let path = dir.path().join("token");
        write_secret_file(&path, "abc\n").unwrap();
        let mode = fs::metadata(&path).unwrap().permissions().mode() & 0o777;
        assert_eq!(mode, 0o600);
        assert!(write_secret_file(&path, "other").is_err());
        assert_eq!(fs::read_to_string(&path).unwrap(), "abc\n");
    }
```

Почему это работает: режим передаётся операционной системе **в момент создания файла**, поэтому нет момента, когда
он доступен для чтения другим. `create_new(true)` заставляет вызов завершиться ошибкой, если файл существует, так что
старый файл (или ссылка, подложенная атакующим) никогда не перезаписывается.

</details>

## Вопросы для самопроверки

1. Почему аргумент командной строки — плохое место для пароля?

<details>
<summary>Ответ</summary>

Список процессов показывает аргументы другим пользователям машины, и они могут попасть в историю командной оболочки
и журналы.

</details>

2. От чего защищает `SecretString` и от чего не защищает?

<details>
<summary>Ответ</summary>

Эта обёртка защищает от случайной печати (`Debug` показывает `[REDACTED]`, нет `Display`) и стирает свою память при
удалении. Она не защищает от кода, который вызывает `expose_secret()` и печатает значение, и от копий, сделанных
в другом месте.

</details>

3. Что нужно сделать первым делом после того, как секрет попал в коммит Git?

<details>
<summary>Ответ</summary>

Отозвать и заменить секрет в сервисе, который его выдал. Очистка истории идёт после, потому что копии старого
значения уже могут существовать.

</details>

4. Почему `load_secret` принимает поиск по окружению как параметр?

<details>
<summary>Ответ</summary>

Чтобы тесты могли подставить поддельное окружение, не меняя настоящие переменные всего процесса, что небезопасно в
Rust 2024 и делает тесты зависимыми друг от друга.

</details>

## Итоги

- Секрет — всё, что даёт доступ; он утекает в основном через обычные места: код, историю Git, журналы, аргументы, окружение, образы.
- Предпочитайте файлы, предоставленные платформой (режим `0600`), или менеджер секретов; избегайте аргументов и образов; никогда не коммитьте секреты.
- Принимайте и `NAME`, и `NAME_FILE`, завершайтесь сразу при запуске и отклоняйте пустые значения и файлы, доступные другим.
- Храните секреты в `SecretString` (скрытый `Debug`, стирание при удалении); вызывайте `expose_secret()` редко и ненадолго.
- `zeroize` стирает ту копию, которую вы держите; это один слой, а не гарантия.
- Создавайте файлы с секретами приватными с первого байта (`mode(0o600)`, `create_new`).
- Когда секрет утёк: сначала отзовите и замените, затем расследуйте, уберите и добавьте проверку.
- Разделяйте секреты по окружениям, используйте наименьшие привилегии, меняйте их и предпочитайте недолговечные учётные данные.

## Источники и что почитать

- [OWASP Secrets Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html) — хранение, ротация, контроль доступа и что делать после утечки.
- [The Twelve-Factor App — Config](https://12factor.net/config) — почему конфигурация и секреты приходят из окружения, а не из кода.
- [Документация крейта `secrecy`](https://docs.rs/secrecy/0.10.3/secrecy/) — `SecretString`, `SecretBox` и `ExposeSecret`.
- [Документация крейта `zeroize`](https://docs.rs/zeroize/1.9.1/zeroize/) — `Zeroize`, `Zeroizing` и пределы стирания памяти.
- [Docker — Manage sensitive data with Docker secrets](https://docs.docker.com/engine/swarm/secrets/) — как платформа монтирует секреты как файлы.
- [Kubernetes — Secrets](https://kubernetes.io/docs/concepts/configuration/secret/) — как хранятся Secrets и хорошие практики вокруг них.
- [GitHub Docs — About secret scanning](https://docs.github.com/en/code-security/secret-scanning/introduction/about-secret-scanning) — автоматическое обнаружение утёкших токенов.
- [gitleaks](https://github.com/gitleaks/gitleaks) — сканер секретов в репозиториях Git.
- [Rust — `std::os::unix::fs::OpenOptionsExt`](https://doc.rust-lang.org/std/os/unix/fs/trait.OpenOptionsExt.html) — установка режима файла при его создании.

---

[🏠 Начало](../START-HERE.md) · [← 10.7 Авторизация, сессии и токены](07-authorization-sessions-and-tokens.md) · [📋 Этап 10](README.md) · [10.9 Криптография I: хеши, MAC и случайность →](09-cryptography-hashes-macs-and-randomness.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
