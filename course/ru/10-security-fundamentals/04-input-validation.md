<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 10.3 OWASP Top 10](03-the-owasp-top-10.md) · [📋 Этап 10](README.md) · [10.5 Атаки внедрения: SQL, команды и пути →](05-injection-attacks.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/10-security-fundamentals/04-input-validation.md) <!-- nav -->

# Урок 10.4 — Проверка ввода: никогда не доверяйте вводу

> **Этап 10 — Основы безопасности** · Урок 4 из 15 · ⏱ около 135 минут

## Что вы узнаете

- Чем отличаются проверка, очистка и экранирование и почему список разрешённого лучше списка запрещённого.
- Правильный порядок проверок: размер, кодировка, канонический вид, синтаксис, смысл.
- Как «разбирать, а не проверять» (parse, don't validate): превращать сырой текст в тип, который может хранить только хорошие значения.
- Как проверять числа, избегать переполнения и безопасно использовать регулярное выражение.
- Как тестировать свою проверку враждебным вводом.

## Перед началом

- Вы прошли [Урок 10.3 — OWASP Top 10](03-the-owasp-top-10.md).
- Вы знаете newtype и «делаем невозможными неверные состояния» ([Урок 08.5](../08-software-design/05-type-driven-design.md)), `Result` и собственные ошибки ([Урок 04.10](../04-structs-enums-collections-errors/10-error-handling.md)), а также разбор через `FromStr`.
- Создайте пакет: `cargo new --lib code/p10-validate`, выполните внутри него `cargo add regex@1.13`, затем `mkdir src/bin`.

## Новые слова

| Слово | Значение |
|---|---|
| **проверка** (validation) | Проверка того, что ввод имеет правильную форму и смысл, и **отклонение** его, если это не так. |
| **очистка** (sanitization) | Изменение ввода для удаления опасных частей (например, удаление символов). Легко сделать неправильно; лучше отклонять. |
| **экранирование** (escaping, или *encoding*) | Запись значения так, чтобы принимающее место считало его обычными данными, а не командой (например, `<` становится `&lt;` в HTML). Урок 10.5. |
| **список разрешённого** (allow-list) | Список того, что **разрешено**; всё остальное отклоняется. Также называется «белым списком». |
| **список запрещённого** (deny-list) | Список того, что **не разрешено**; всё остальное проходит. Также называется «чёрным списком». Слабый подход: атакующие находят то, что вы забыли. |
| **канонический вид** (canonical form) | Единственный стандартный способ записи значения (нижний регистр, без лишних пробелов, одна форма Unicode), чтобы равные значения выглядели равными. |
| **Unicode** | Стандарт, который даёт номер каждому символу каждой письменности и эмодзи. В Rust `char` — это одно такое значение. |
| **омоглиф** (homoglyph) | Символ, похожий на другой, например кириллическая `а` и латинская `a`. Используется для подделки имён. |
| **newtype** | Структура с одним полем, придающая существующему типу новый смысл, например `struct Username(String)`. |
| **регулярное выражение** (regular expression, regex) | Короткий шаблон, описывающий набор текстов, например `^[a-z]+$` для «только строчные буквы». |
| **ReDoS** (Regular expression Denial of Service, отказ в обслуживании через регулярное выражение) | Атака текстом, из-за которого медленный движок регулярных выражений работает минутами и дольше. |
| **ASCII** (American Standard Code for Information Interchange, американский стандартный код обмена информацией) | Самый старый и самый маленький набор символов: 128 символов (английские буквы, цифры, некоторые знаки). Каждый занимает один байт. |
| **UTF-8** (Unicode Transformation Format, 8-bit, формат преобразования Unicode, 8 бит) | Обычный способ записи текста Unicode в виде байтов. Символы ASCII занимают один байт, остальные — от двух до четырёх. Текст в Rust всегда в UTF-8. |
| **HTML** (HyperText Markup Language, язык гипертекстовой разметки) | Язык веб-страниц с тегами вроде `<p>` и `<script>`. |
| **SQL** (Structured Query Language, язык структурированных запросов) | Язык, на котором базу данных просят прочитать или изменить данные, например `SELECT * FROM users`. Урок 10.5 показывает, почему его никогда нельзя собирать из необработанного ввода. |

## 1. Никогда не доверяйте вводу

Каждое значение, пришедшее снаружи вашей программы, — это **ввод**: аргументы командной строки, переменные
окружения, файлы, сетевые данные, поля форм, даже результат другой программы. Первое правило этого урока
простое: ввод враждебен, пока ваш код его не проверил, и проверка происходит **на границе доверия**
([Урок 10.1](01-security-mindset-and-the-cia-triad.md)), один раз, как можно раньше.

Три слова часто путают:

| Приём | Что делает | Когда |
|---|---|---|
| **Проверять** (validate) | Проверяет ввод и отклоняет плохой | Всегда, на границе |
| **Очищать** (sanitize) | Переписывает ввод, убирая плохие части | Редко; только когда отклонить невозможно |
| **Экранировать** (escape) | Делает значение безопасным для одного конкретного назначения (HTML, SQL, оболочка) | Там, где значение **используется**, каждый раз ([10.5](05-injection-attacks.md)) |

Проверка отвечает на вопрос «это допустимое значение?». Экранирование отвечает на вопрос «как безопасно
поместить это значение в тот язык?». Нужно и то и другое, и одно не заменяет другое.

## 2. Список разрешённого, а не список запрещённого

Допустим, программа должна принимать имя файла. Подход со списком запрещённого говорит: «отклонять имена,
содержащие `;`, `|` или `&`». Атакующий пишет `$(whoami)`, обратную кавычку, перевод строки или символ, о
котором вы никогда не слышали. Каждый забытый символ — дыра.

Подход со списком разрешённого говорит: «принимать только буквы, цифры, `_` и `-`, от 3 до 32 символов».
Атакующий может использовать только то, что вы перечислили, и вы перечислили это намеренно. Когда программа
растёт, новые опасные символы не проскальзывают, потому что новые символы по умолчанию отклоняются. Это
«безопасно по умолчанию» ([Урок 10.1](01-security-mindset-and-the-cia-triad.md)), применённое к данным.

## 3. Порядок проверок

Сначала делайте дешёвые общие проверки, а специальные — в конце:

1. **Размер.** Достаточно ли он мал? Отказывайте огромному вводу до его обработки (Урок 10.1, раздел 3.3).
2. **Кодировка.** Это корректный текст? `String` и `&str` в Rust всегда корректный UTF-8 (обычный способ
   записи Unicode в виде байтов). Байты снаружи должны пройти через `std::str::from_utf8` или
   `String::from_utf8`, которые возвращают ошибку для плохих байтов.
3. **Канонический вид.** Преобразуйте в стандартную форму (например, в нижний регистр) **до** проверки или сравнения.
4. **Синтаксис.** Правильная ли форма и символы? (Список разрешённого.)
5. **Смысл.** Разумно ли значение? Число в диапазоне, дата в прошлом, пользователь существует.

## 4. Разбирать, а не проверять: тип `Username`

У функции `is_valid(&str) -> bool` есть слабость: она возвращает `true`, и дальше программа продолжает
работать с тем же обычным `String`. Через десять функций никто не помнит, проверена ли эта строка. Лучший
дизайн **разбирает** текст в тип, существующий только для хороших значений. Это проектирование через типы
из Урока 08.5, теперь применённое для безопасности.

Вот библиотека `src/lib.rs`: тип `Username` с приватным полем, так что создать его можно только через
`Username::parse`:

```rust
use std::fmt;
use std::str::FromStr;

pub const MIN_USERNAME_LEN: usize = 3;
pub const MAX_USERNAME_LEN: usize = 32;

#[derive(Debug, PartialEq, Eq)]
pub enum UsernameError {
    TooShort,
    TooLong,
    MustStartWithLetter,
    BadCharacter(char),
}

impl fmt::Display for UsernameError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            UsernameError::TooShort => {
                write!(f, "must have at least {MIN_USERNAME_LEN} characters")
            }
            UsernameError::TooLong => write!(f, "must have at most {MAX_USERNAME_LEN} characters"),
            UsernameError::MustStartWithLetter => write!(f, "must start with a letter a-z"),
            UsernameError::BadCharacter(c) => {
                write!(f, "contains a character that is not allowed: {c:?}")
            }
        }
    }
}

impl std::error::Error for UsernameError {}

/// A user name that has already been checked. The field is private,
/// so the only way to get a `Username` is through `parse`.
#[derive(Debug, Clone, PartialEq, Eq, Hash)]
pub struct Username(String);

impl Username {
    pub fn parse(input: &str) -> Result<Username, UsernameError> {
        // 1. Size first: a cheap check before we look at every character.
        //    (`len()` counts bytes; a character is at most 4 bytes, so this is safe.)
        if input.len() > MAX_USERNAME_LEN * 4 {
            return Err(UsernameError::TooLong);
        }
        // 2. The first character.
        let first = input.chars().next().ok_or(UsernameError::TooShort)?;
        if !first.is_ascii_alphabetic() {
            return Err(UsernameError::MustStartWithLetter);
        }
        // 3. Allow-list: only these characters are accepted.
        if let Some(bad) = input
            .chars()
            .find(|c| !(c.is_ascii_alphanumeric() || *c == '_' || *c == '-'))
        {
            return Err(UsernameError::BadCharacter(bad));
        }
        // 4. Now the text is plain ASCII, so len() is the number of characters.
        if input.len() < MIN_USERNAME_LEN {
            return Err(UsernameError::TooShort);
        }
        if input.len() > MAX_USERNAME_LEN {
            return Err(UsernameError::TooLong);
        }
        // 5. Canonical form: always lowercase, so "Ana" and "ana" are the same user.
        Ok(Username(input.to_ascii_lowercase()))
    }

    pub fn as_str(&self) -> &str {
        &self.0
    }
}

impl FromStr for Username {
    type Err = UsernameError;

    fn from_str(s: &str) -> Result<Self, Self::Err> {
        Username::parse(s)
    }
}
```

**Что происходит:**

1. `UsernameError` перечисляет все причины отказа. Код `Display` даёт сообщение, понятное пользователю.
   Сообщение говорит, *что не так*, и ничего не раскрывает о системе.
2. У `struct Username(String)` **приватное** поле. Код вне модуля не может написать
   `Username("../etc".to_string())`; он обязан вызвать `parse`. Значит, **любой `Username` в программе
   корректен**.
3. Шаг 1 в `parse` сразу отклоняет огромный ввод. `char` занимает не более 4 байт, поэтому корректное имя
   никогда не длиннее `MAX_USERNAME_LEN * 4` байт. Эта проверка ничего не стоит и не пускает длинную строку в цикл.
4. Шаг 2 смотрит на первый символ (`chars().next()` — это `None` для пустого текста, что становится
   `TooShort`). `is_ascii_alphabetic` истинно только для 52 букв `a-z` и `A-Z`.
5. Шаг 3 — **список разрешённого**: `find` возвращает первый символ, который не является ASCII-буквой или
   цифрой, подчёркиванием или дефисом. Этот символ попадает в ошибку, чтобы сообщение могло его назвать.
6. Шаг 4 проверяет длину. После шага 3 текст — обычный ASCII, поэтому один байт — один символ, и `len()` —
   число символов. (Для текста, который может содержать любой Unicode, `len()` считает байты, а
   `chars().count()` — символы; эта разница — классический источник ошибок.)
7. Шаг 5 строит **канонический вид**: `to_ascii_lowercase`. `Ana` и `ana` становятся одним пользователем.
8. `impl FromStr` позволяет писать `"ana".parse::<Username>()`.

Теперь числовой тип. Добавьте это в тот же файл:

```rust
#[derive(Debug, PartialEq, Eq)]
pub enum QuantityError {
    NotANumber,
    OutOfRange,
}

/// An order quantity from 1 to 1000.
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub struct Quantity(u32);

impl Quantity {
    pub fn parse(text: &str) -> Result<Quantity, QuantityError> {
        let n: u32 = text.trim().parse().map_err(|_| QuantityError::NotANumber)?;
        if (1..=1000).contains(&n) {
            Ok(Quantity(n))
        } else {
            Err(QuantityError::OutOfRange)
        }
    }

    pub fn get(self) -> u32 {
        self.0
    }
}

/// Total price in cents, or `None` if the number does not fit in a `u32`.
pub fn total_cents(unit_price_cents: u32, quantity: Quantity) -> Option<u32> {
    unit_price_cents.checked_mul(quantity.get())
}
```

**Что происходит:**

1. `Quantity::parse` сначала разбирает в `u32`. `u32` не может быть отрицательным, поэтому `-3` даёт
   `NotANumber`. `trim()` убирает пробелы вокруг текста. `map_err(|_| …)` заменяет стандартную ошибку
   разбора нашей.
2. `(1..=1000).contains(&n)` проверяет **диапазон**: бизнес-правило — «от 1 до 1000». Синтаксис (это
   число) и смысл (это разумное число) — две отдельные проверки.
3. `total_cents` умножает через `checked_mul`. Даже допустимые количества могут дать число, которое не
   помещается: `5_000_000 × 1000` больше `u32::MAX`. Без `checked_mul` релизная сборка тихо «перевернула» бы
   число (Урок 10.1, раздел 3.2). С ним вызывающий получает `None` и обязан это обработать.

Наконец, программа, использующая оба типа. Сохраните её как `src/bin/signup.rs` и запустите `cargo run --bin signup`:

```rust
use p10_validate::{Quantity, Username, total_cents};

fn main() {
    let names = [
        "Ana_01",
        "ab",
        "1abc",
        "ana smith",
        "ana;rm",
        "\u{430}na_smith",
        "../etc",
    ];
    for name in names {
        match Username::parse(name) {
            Ok(user) => println!("{name:<14?} accepted as {}", user.as_str()),
            Err(error) => println!("{name:<14?} rejected: {error}"),
        }
    }

    for text in ["12", " 7 ", "0", "-3", "1e3", "99999999999"] {
        match Quantity::parse(text) {
            Ok(q) => println!(
                "{text:<14?} quantity {}, total {:?}",
                q.get(),
                total_cents(2_500, q)
            ),
            Err(error) => println!("{text:<14?} rejected: {error:?}"),
        }
    }
}
```

```text
"Ana_01" accepted as ana_01
"ab" rejected: must have at least 3 characters
"1abc" rejected: must start with a letter a-z
"ana smith" rejected: contains a character that is not allowed: ' '
"ana;rm" rejected: contains a character that is not allowed: ';'
"аna_smith" rejected: must start with a letter a-z
"../etc" rejected: must start with a letter a-z
"12" quantity 12, total Some(30000)
" 7 " quantity 7, total Some(17500)
"0" rejected: OutOfRange
"-3" rejected: NotANumber
"1e3" rejected: NotANumber
"99999999999" rejected: NotANumber
```

**Что происходит:**

1. `{name:<14?}` печатает текст в кавычках `Debug`, дополненный до 14 символов, чтобы вы видели точный
   текст, включая текст `аna_smith`. Присмотритесь: первая буква — кириллическая `а` (U+0430), а не
   латинская `a`. Человеку они кажутся одинаковыми; список разрешённого видит разницу и отклоняет.
   Списку запрещённого пришлось бы знать об этом символе.
2. `"ana smith"` и `"ana;rm"` отклоняются на конкретном символе, который показывает сообщение.
3. `"../etc"` отклоняется, потому что не начинается с буквы, поэтому никогда не дойдёт до файловой системы.
4. `"1e3"` (тысяча в научной записи) и `"99999999999"` (слишком велико для `u32`) не принимаются как
   числа: `parse::<u32>` принимает только цифры.

> 🔒 **Безопасность:** **не «исправляйте» плохой ввод молча.** Если вы удалите странные символы и пойдёте
> дальше, два разных ввода могут превратиться в одно значение, а очищенное значение всё ещё может быть
> опасным. Отклоняйте с понятным сообщением и пусть вызывающий попробует снова.

## 5. Тестируйте враждебным вводом

Код проверки — лучшее место для тестов, потому что ошибки прячутся на краях. Добавьте тесты в конец
`src/lib.rs`:

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn accepts_good_names_and_lowercases_them() {
        assert_eq!(Username::parse("Ana_01").unwrap().as_str(), "ana_01");
    }

    #[test]
    fn border_lengths() {
        assert_eq!(Username::parse("ab"), Err(UsernameError::TooShort));
        assert!(Username::parse("abc").is_ok());
        assert!(Username::parse(&"a".repeat(32)).is_ok());
        assert_eq!(
            Username::parse(&"a".repeat(33)),
            Err(UsernameError::TooLong)
        );
    }

    #[test]
    fn rejects_nasty_input_without_panicking() {
        for input in [
            "",
            "1abc",
            "ana smith",
            "ana;rm",
            "../etc",
            "a\0b",
            "ñandú",
            "😀😀😀",
        ] {
            assert!(Username::parse(input).is_err(), "should reject {input:?}");
        }
        assert!(Username::parse(&"é".repeat(100_000)).is_err());
    }

    #[test]
    fn a_lookalike_letter_is_rejected() {
        // The first letter is the Cyrillic "а" (U+0430), which looks like the Latin "a".
        assert_eq!(
            Username::parse("\u{430}na_smith"),
            Err(UsernameError::MustStartWithLetter)
        );
    }

    #[test]
    fn quantity_range_and_total() {
        assert_eq!(Quantity::parse(" 12 ").unwrap().get(), 12);
        assert_eq!(Quantity::parse("0"), Err(QuantityError::OutOfRange));
        assert_eq!(Quantity::parse("1001"), Err(QuantityError::OutOfRange));
        assert_eq!(Quantity::parse("-3"), Err(QuantityError::NotANumber));
        assert_eq!(Quantity::parse("abc"), Err(QuantityError::NotANumber));
        let q = Quantity::parse("1000").unwrap();
        assert_eq!(total_cents(5_000_000, q), None);
        assert_eq!(total_cents(2_000, q), Some(2_000_000));
    }
}
```

```bash
cargo test
```

```text
running 5 tests
test tests::a_lookalike_letter_is_rejected ... ok
test tests::quantity_range_and_total ... ok
test tests::accepts_good_names_and_lowercases_them ... ok
test tests::border_lengths ... ok
test tests::rejects_nasty_input_without_panicking ... ok

test result: ok. 5 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

**Что происходит:**

1. `border_lengths` проверяет значения **на** границе: 2 (слишком коротко), 3 (нормально), 32 (нормально),
   33 (слишком длинно). Ошибки на единицу живут именно там.
2. `rejects_nasty_input_without_panicking` проходит по списку неприятных текстов: пустой, цифра первой,
   пробел, точка с запятой, путь, символ NUL (байт со значением ноль, `\0`), не-ASCII буквы, эмодзи и текст из 100 000 символов.
   Тест ещё и обещание: **никакой ввод не вызывает панику валидатора**.
3. `a_lookalike_letter_is_rejected` использует `\u{430}`, экранирование кириллической буквы, чтобы сам файл
   теста ясно показывал, что проверяется.
4. Последний тест проверяет обе ошибки диапазона, случаи «не число» и переполнение в `total_cents`.

> 🧪 **Тестирование:** такие модульные тесты находят ошибки, о которых вы подумали. Чтобы найти ошибки, о
> которых вы не подумали, подайте валидатору случайный ввод: через тестирование на основе свойств
> ([Урок 09.4](../09-testing-debugging-benchmarking/04-property-based-testing.md)) или фаззинг
> ([Урок 09.8](../09-testing-debugging-benchmarking/08-fuzzing.md)). Проверяемое свойство: «для любого
> ввода `parse` возвращает `Ok` или `Err` и никогда не паникует».

## 6. Регулярные выражения как список разрешённого

**Регулярное выражение** — компактный способ записать список разрешённого. Крейт `regex` — стандартный выбор
в Rust. Самая частая ошибка — забыть **якоря**: `^` означает «начало текста», а `$` — «конец текста». Без них
регулярное выражение лишь спрашивает «есть ли совпадение *где-то внутри*?». Сохраните `src/bin/anchors.rs`:

```rust
use regex::Regex;

fn main() {
    let loose = Regex::new("[a-z]+").unwrap();
    let strict = Regex::new("^[a-z]+$").unwrap();

    for input in ["hello", "hello; echo INJECTED", "HELLO", "hello\n"] {
        println!(
            "{input:<24?} loose: {:<5} strict: {}",
            loose.is_match(input),
            strict.is_match(input)
        );
    }
}
```

```text
"hello" loose: true  strict: true
"hello; echo INJECTED" loose: true  strict: false
"HELLO" loose: false strict: false
"hello\n" loose: true  strict: false
```

**Что происходит:**

1. У `loose` нет якорей. Оно говорит «где-то есть последовательность строчных букв», поэтому
   `hello; echo INJECTED` проходит: слово `hello` в нём есть.
2. У `strict` оба якоря: **весь** текст должен состоять из строчных букв. Текст атаки, заглавные буквы и
   текст с завершающим переводом строки отклоняются.
3. В этом крейте `$` совпадает только в самом конце текста, а не перед завершающим переводом строки, поэтому
   `hello\n` отклоняется. (Некоторые другие движки регулярных выражений трактуют завершающий перевод строки
   иначе, что приводило к настоящим ошибкам, поэтому всегда проверяйте это тестом.)

**ReDoS.** В некоторых движках регулярных выражений специально составленный текст может заставить
сопоставление работать экспоненциальное время. Крейт `regex` спроектирован так, чтобы этого избегать: он не
использует метод «перебора с возвратами» (backtracking), и время сопоставления растёт пропорционально длине
текста (и шаблона). Это защищает вас, когда **текст** приходит снаружи. Если снаружи приходит сам
**шаблон**, огромный шаблон всё равно стоит времени и памяти при компиляции, поэтому ограничивайте его длину и
размер компиляции (см. документацию крейта, раздел «Avoiding pathological patterns»).

> 💡 **Совет:** для простого правила вроде нашего имени пользователя ручная проверка через `chars()` читается
> и тестируется легче, чем регулярное выражение. Используйте регулярное выражение, когда форма действительно
> является шаблоном (даты, коды, версии).

## 7. Ограничения на всё

Кроме формы значения ограничивайте его **количество**: размер запроса или файла (Урок 10.1), **число**
элементов в списке, глубину вложенности JSON (`serde_json` по умолчанию останавливается на глубине 128) и
**время**, которое может занимать запрос. `Vec`, растущий с каждым элементом, который присылает клиент, —
атака на память. Выберите максимум, возвращайте ошибку выше него и напишите на это тест.

## Частые ошибки

> ⚠️ **Частая ошибка:** проверять в одном месте, а использовать в другом. Если проверка есть в веб-форме, а та
> же функция вызывается и из импорта файла, импорт файла не защищён. Поместите проверку в тип
> (`Username::parse`), чтобы каждый путь проходил через неё.

> ⚠️ **Частая ошибка:** проверять **до** приведения к каноническому виду. Если вы проверяете текст `admin`, а
> потом приводите к нижнему регистру, то `ADMIN` проходит проверку и становится `admin`. Сначала приводите к
> каноническому виду.

> ⚠️ **Частая ошибка:** `if name.len() > 32` для не-ASCII текста в уверенности, что `len()` считает символы.
> Он считает байты. В `"ñandú"` 5 символов и 7 байт.

> ⚠️ **Частая ошибка:** сообщения об ошибках, которые повторяют ввод на странице или в журнале без
> экранирования. Сообщение становится атакой. Печатайте его через `{:?}`, как в Уроке 10.3, раздел 6, или
> экранируйте для HTML.

## Попробуйте сами

1. Наберите библиотеку, две программы и тесты. Запустите `cargo test` и программы.
2. Измените `MAX_USERNAME_LEN` на 8. Какой тест упадёт (`border_lengths`) и почему? Исправьте тест, а не код,
   если новое ограничение — то, что вам нужно.

## Упражнения

### Упражнение 1 — Список разрешённого или запрещённого? (легко)

Для каждого правила скажите, список ли это разрешённого или запрещённого, и назовите один ввод, который
его обходит: (a) «отклонять символы `<`, `>` и `&`»; (b) «принимать `[a-z0-9]{8}`»; (c) «отклонять `..`»;
(d) «принимать только расширения `.png` и `.jpg`».

<details>
<summary>Подсказка</summary>

Спросите: правило перечисляет то, что **хорошо**, или то, что **плохо**?

</details>

<details>
<summary>Ответ</summary>

(a) Список запрещённого. Символ кавычки (`"` или `'`) всё равно проходит. (b) Список разрешённого;
ничего за пределами перечисленных символов и точной длины 8 не проходит. (c) Список запрещённого. Путь вроде
`/etc/passwd` вообще не содержит `..`, а закодированные формы вроде `%2e%2e` могут быть декодированы позже.
(d) Список разрешённого (но проверяйте расширение в каноническом виде: `PNG`, а у `a.png.exe` расширение `.exe`).

</details>

### Упражнение 2 — Тип `Port` (средне)

Напишите `Port::parse(&str) -> Result<Port, PortError>`. Порт — целое число от 1 до 65 535. Напишите
**тесты** для `"0"`, `"1"`, `"65535"`, `"65536"`, `""`, `"-1"`, `" 80 "` и `"8o"`.

<details>
<summary>Подсказка</summary>

`u16` вмещает ровно от 0 до 65 535. Разберите через `parse::<u16>()`, затем отклоните 0. Учтите, что
`"65536"` не проходит **разбор** (не помещается), а не проверку диапазона, поэтому обе ошибки могут быть одним
вариантом `Invalid`.

</details>

<details>
<summary>Ответ</summary>

```rust
#[derive(Debug, PartialEq, Eq)]
pub struct Port(u16);

#[derive(Debug, PartialEq, Eq)]
pub enum PortError {
    Invalid,
}

impl Port {
    pub fn parse(text: &str) -> Result<Port, PortError> {
        match text.trim().parse::<u16>() {
            Ok(0) | Err(_) => Err(PortError::Invalid),
            Ok(n) => Ok(Port(n)),
        }
    }

    pub fn get(&self) -> u16 {
        self.0
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn port_borders() {
        assert_eq!(Port::parse("0"), Err(PortError::Invalid));
        assert_eq!(Port::parse("1").unwrap().get(), 1);
        assert_eq!(Port::parse("65535").unwrap().get(), 65535);
        assert_eq!(Port::parse("65536"), Err(PortError::Invalid));
        assert_eq!(Port::parse(""), Err(PortError::Invalid));
        assert_eq!(Port::parse("-1"), Err(PortError::Invalid));
        assert_eq!(Port::parse(" 80 ").unwrap().get(), 80);
        assert_eq!(Port::parse("8o"), Err(PortError::Invalid));
    }
}
```

Почему это работает: тип `u16` — проверка верхней границы диапазона, `Ok(0)` — нижней, а любой текст, не
являющийся простым числом, попадает в `Err(_)`.

</details>

### Упражнение 3 — Протестируйте регулярное выражение с якорями (средне)

Напишите функцию `is_word(text: &str) -> bool`, истинную только для непустого текста из строчных букв
`a-z`, с помощью регулярного выражения. Напишите **тест** с вводами, которые прошли бы регулярное выражение
без якорей, но должны быть отклонены.

<details>
<summary>Подсказка</summary>

Используйте `^[a-z]+$`. Проверьте `"hello world"`, `"hello\n"`, `"x; rm"`, `""` и `"Hello"`.

</details>

<details>
<summary>Ответ</summary>

```rust
pub fn is_word(text: &str) -> bool {
    regex::Regex::new("^[a-z]+$").unwrap().is_match(text)
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn the_regex_is_anchored() {
        assert!(is_word("hello"));
        assert!(!is_word(""));
        assert!(!is_word("hello world"));
        assert!(!is_word("hello\n"));
        assert!(!is_word("x; rm"));
        assert!(!is_word("Hello"));
    }
}
```

Почему это работает: `^` и `$` заставляют совпасть **весь** текст, `+` требует хотя бы одну букву, а класс
`[a-z]` не содержит пробела, точки с запятой и заглавных букв. (В настоящем коде создавайте `Regex` один раз и
используйте повторно: компиляция при каждом вызове медленная.)

</details>

## Вопросы для самопроверки

1. Чем отличается проверка от экранирования?

<details>
<summary>Ответ</summary>

Проверка убеждается, что значение допустимо, и отклоняет плохие значения на границе. Экранирование делает
значение безопасным для одного назначения (HTML, SQL, оболочка) в месте его использования.

</details>

2. Почему список разрешённого безопаснее списка запрещённого?

<details>
<summary>Ответ</summary>

Список разрешённого отклоняет всё, что вы не перечислили, включая то, что вы забыли или что будет изобретено
позже. Список запрещённого пропускает всё, что вы забыли.

</details>

3. Что значит «разбирать, а не проверять»?

<details>
<summary>Ответ</summary>

Преобразовать сырой ввод в тип, который может хранить только корректные значения (приватное поле, один
конструктор с проверкой). Тогда каждое значение этого типа в программе заведомо корректно, и никакому
последующему коду не нужно проверять снова.

</details>

4. Почему регулярное выражение `[a-z]+` принимает `hello; echo INJECTED`?

<details>
<summary>Ответ</summary>

Без `^` и `$` регулярному выражению достаточно совпадения где-то в тексте, а `hello` совпадает.

</details>

## Итоги

- Ввод враждебен, пока не проверен; проверяйте один раз, на границе доверия, как можно раньше.
- Сначала проверяйте (отклоняйте); экранируйте там, где значение используется; избегайте тихой очистки.
- Используйте список разрешённого. Список запрещённого — это перечень того, что вы надеетесь не забыть.
- Порядок: размер, кодировка, канонический вид, синтаксис, смысл.
- Разбирайте в типы (`Username`, `Quantity`) с приватными полями, чтобы корректные значения были единственными.
- Используйте арифметику `checked_*` для чисел снаружи и проверяйте диапазоны.
- Ставьте якоря в регулярных выражениях; `regex` в Rust сопоставляет за время, пропорциональное длине текста.
- Тестируйте границы и враждебный ввод; остальное ищите фаззингом.

## Источники и что почитать

- [OWASP Input Validation Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Input_Validation_Cheat_Sheet.html) — практические правила синтаксической и смысловой проверки, списки разрешённого и многое другое.
- [Parse, don't validate (Alexis King)](https://lexi-lambda.github.io/blog/2019/11/05/parse-don-t-validate/) — статья, сделавшая идею «parse, don't validate» популярной.
- [Документация крейта `regex`](https://docs.rs/regex/1.13.1/regex/) — синтаксис, якоря и раздел об избегании патологических шаблонов.
- [Стандартная библиотека Rust — примитив `char`](https://doc.rust-lang.org/std/primitive.char.html) — `is_ascii_alphabetic` и другие методы символов.
- [Unicode Technical Standard #39 — Security Mechanisms](https://www.unicode.org/reports/tr39/) — похожие символы (омоглифы) и как их обнаруживать.
- [The Rust Book — To `panic!` or Not to `panic!`](https://doc.rust-lang.org/book/ch09-03-to-panic-or-not-to-panic.html) — когда программе можно паниковать, а когда обязательно возвращать ошибку.

---

[🏠 Начало](../START-HERE.md) · [← 10.3 OWASP Top 10](03-the-owasp-top-10.md) · [📋 Этап 10](README.md) · [10.5 Атаки внедрения: SQL, команды и пути →](05-injection-attacks.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
