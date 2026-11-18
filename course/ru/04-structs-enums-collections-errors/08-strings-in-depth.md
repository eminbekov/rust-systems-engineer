<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 04.7 Векторы](07-vectors.md) · [📋 Этап 04](README.md) · [04.9 Хеш-таблицы (HashMap) →](09-hash-maps.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/04-structs-enums-collections-errors/08-strings-in-depth.md) <!-- nav -->

# Урок 04.8 — Строки подробно

> **Этап 04 — Структуры, перечисления, коллекции и ошибки** · Урок 8 из 12 · ⏱ около 4 часов (2 занятия)

## Что вы узнаете

- Как компьютер хранит текст: байты, Unicode и UTF-8, и почему `"привет".len()` — это не 6.
- Как создавать и наращивать `String` и чем `+` отличается от `format!`.
- Почему нельзя написать `s[0]` для строки и что делать вместо этого.
- Как обходить текст по байтам, по символам и по позициям.
- Повседневные методы: `trim`, `split`, `split_once`, `find`, `replace`, `parse` и другие.

## Перед началом

- Вы прошли [Урок 04.7 — Векторы](07-vectors.md).
- Вы знаете разницу между `String` и `&str` ([Урок 03.6](../03-ownership-and-borrowing/06-slices-and-string-types.md)) и умеете читать и разбирать текст ([Урок 02.7](../02-programming-basics/07-reading-input-and-parsing.md)).
- Создайте проект:

```bash
cd ~/projects/rust
cargo new code/p04-strings
cd code/p04-strings
```

## Новые слова

| Слово | Значение |
|---|---|
| **байт** | Число от 0 до 255. Наименьший кусок памяти, с которым обычно работает программа. В Rust для него есть тип `u8`. |
| **ASCII** (American Standard Code for Information Interchange, американский стандартный код обмена информацией) | Старая таблица, которая даёт номер 128 символам: английским буквам, цифрам, знакам препинания и управляющим кодам. Например, `A` — это 65. |
| **Unicode** | Огромная таблица, которая даёт уникальный номер каждому символу каждой системы письма, а также символам и эмодзи (их более 150 000). |
| **кодовая точка** (code point) | Один номер из таблицы Unicode, например U+044F для кириллической буквы `я`. |
| **кодировка** (encoding) | Правило, по которому кодовые точки превращаются в байты (и обратно), чтобы текст можно было хранить или передавать. |
| **UTF-8** (Unicode Transformation Format, 8-bit, формат преобразования Unicode, 8 бит) | Самая используемая кодировка. Она тратит от 1 до 4 байт на каждую кодовую точку. Строки в Rust — всегда корректный UTF-8. |
| **`char`** | Тип Rust, который хранит одну кодовую точку Unicode (она называется *скалярным значением Unicode*). Всегда занимает 4 байта. Литерал пишется в одинарных кавычках: `'a'`. |
| **`String`** | Строка, которой вы владеете: растущий кусок текста в куче. Внутри это `Vec<u8>`, гарантированно содержащий корректный UTF-8. |
| **`&str`** (строковый срез) | Заимствованный взгляд на текст: указатель и длина. Строковые литералы вроде `"hello"` имеют этот тип. |
| **граница символа** (char boundary) | Позиция в байтах, где заканчивается один символ и начинается следующий. Резать текст можно только по границам. |
| **конкатенация** | Соединение двух кусков текста в один более длинный. |
| **разделитель** (delimiter) | Символ или текст, который разделяет части в длинном тексте, например запятая в `"a,b,c"`. |
| **FFI** (Foreign Function Interface, интерфейс внешних функций) | Способ, которым код на Rust вызывает код, написанный на другом языке, например на C. См. [Урок 06.10 — FFI](../06-rust-advanced/10-ffi-calling-c.md). |
| **графемный кластер** (grapheme cluster) | То, что человек читает как одну «букву» на экране. Может состоять из нескольких кодовых точек, например `e` плюс отдельный знак ударения. Стандартная библиотека не делит текст таким образом. |

## 1. Как компьютер хранит текст

Компьютер хранит только числа. Чтобы хранить текст, люди договорились о таблице: «число 65
означает букву `A`». Первой большой таблицей был **ASCII**. В нём было 128 записей — хватало для
английского, и каждый символ занимал один байт.

В мире символов гораздо больше 128: `é`, `я`, `世`, `😀`. **Unicode** — современная таблица,
которая охватывает их все. Каждая запись — **кодовая точка**, она записывается как `U+044F`.

Кодовая точка — это число, но всё равно нужно решить, как положить это число в байты. Это правило
называется **кодировкой**. Кодировка, которая сегодня используется почти везде (и единственная,
которую используют строки Rust), — **UTF-8**:

| Кодовые точки | Сколько байт | Пример |
|---|---|---|
| U+0000 до U+007F (ASCII) | 1 | `a` |
| U+0080 до U+07FF | 2 | `é`, `я` |
| U+0800 до U+FFFF | 3 | `世` |
| U+10000 до U+10FFFF | 4 | `😀` |

Приятное свойство: обычный английский текст занимает один байт на букву, точно как в ASCII, а
другие языки при этом тоже работают. Посмотрим на настоящие байты.

```rust
fn main() {
    let samples = ["a", "é", "я", "世", "😀"];
    for s in samples {
        println!("{:<3} chars = {}, bytes = {}, utf-8 = {:02x?}", s, s.chars().count(), s.len(), s.as_bytes());
    }
}
```

**Что происходит:**

1. `s.len()` возвращает число **байт** в тексте, а не число букв.
2. `s.chars().count()` проходит по тексту символ за символом и считает их (`.chars()` создаёт
   итератор по значениям `char`, как `.iter()` в [Уроке 04.7](07-vectors.md)).
3. `s.as_bytes()` показывает «сырые» байты в виде среза из `u8`. Формат `{:02x?}` печатает каждый
   байт двумя шестнадцатеричными цифрами (система счисления по основанию 16: цифры `0-9` и буквы
   `a-f`; `c3` означает 195).
4. `{:<3}` дополняет текст пробелами до ширины 3, выравнивая по левому краю.

```text
a   chars = 1, bytes = 1, utf-8 = [61]
é   chars = 1, bytes = 2, utf-8 = [c3, a9]
я   chars = 1, bytes = 2, utf-8 = [d1, 8f]
世   chars = 1, bytes = 3, utf-8 = [e4, b8, 96]
😀   chars = 1, bytes = 4, utf-8 = [f0, 9f, 98, 80]
```

Это главный факт всего урока: **один символ может занимать 1, 2, 3 или 4 байта.** Поэтому Rust
так осторожен со строками.

```text
String "aя😀" in memory (heap):

 byte offset:   0     1    2     3    4    5    6
              ┌────┬─────────┬──────────────────┐
              │ 61 │ d1   8f │ f0   9f   98  80 │
              └────┴─────────┴──────────────────┘
 character:      a      я            😀
 boundaries:  ^0    ^1        ^3                ^7
```

## 2. Создание `String`

```rust
fn main() {
    let a = String::new();
    let b = String::from("hello");
    let c = "hello".to_string();
    let d = "hello".to_owned();
    let e: String = "hello".into();
    let f = format!("{}-{}", "hel", "lo");
    println!("[{a}] [{b}] [{c}] [{d}] [{e}] [{f}]");
    println!("{}", b == c);
    println!("{}", b == "hello");
}
```

**Что происходит:**

1. `String::new()` — пустая строка, как `Vec::new()`.
2. `String::from("hello")`, `"hello".to_string()`, `"hello".to_owned()` и `"hello".into()` делают
   одно и то же: копируют текст литерала в кучу и дают вам `String`, которой вы владеете.
   (`.into()` требует, чтобы слева был записан тип `: String`. Он использует правило
   преобразования, которое вы изучите в [Уроке 05.2 — Трейты](../05-rust-intermediate/02-traits.md).)
3. `format!` работает как `println!`, но возвращает `String`, а не печатает.
4. `String` можно сравнивать с другой `String` или с `&str` через `==`.

```text
[] [hello] [hello] [hello] [hello] [hel-lo]
true
true
```

> 📏 **Соглашение:** `String::from("text")` и `"text".to_string()` встречаются одинаково часто.
> Выберите один способ и используйте его во всём проекте. В этом курсе используется `String::from`.

## 3. Наращивание `String`

```rust
fn main() {
    let mut s = String::from("Rust");
    s.push(' ');
    s.push_str("is fun");
    s += "!";
    println!("{s}");

    let first = String::from("Hello, ");
    let second = String::from("world");
    let joined = first + &second;
    println!("{joined}");
    println!("{second}");

    let x = String::from("tic");
    let y = String::from("tac");
    let z = String::from("toe");
    let game = format!("{x}-{y}-{z}");
    println!("{game}");
    println!("{x} {y} {z}");
}
```

**Что происходит:**

1. `push(' ')` добавляет **один символ** (`char`, одинарные кавычки). `push_str("is fun")`
   добавляет **кусок текста** (`&str`, двойные кавычки). `s += "!"` — ещё один способ записать
   `push_str`.
2. `first + &second` — это **конкатенация**. Она *перемещает* `first` в операцию, дописывает текст
   `second` и возвращает тот же буфер как `joined`. Поэтому `first` больше использовать нельзя.
   Справа должен быть `&str`; `&second` — это `&String`, и Rust преобразует её сам. `second` только
   заимствована, так что остаётся пригодной (второй `println!` это доказывает).
3. `format!("{x}-{y}-{z}")` строит новую `String` из нескольких частей и **заимствует** их все.
   Ничего не перемещается, поэтому последняя строка по-прежнему может напечатать `x`, `y` и `z`.

```text
Rust is fun!
Hello, world
world
tic-tac-toe
tic tac toe
```

> 💡 **Совет:** для двух частей `+` подойдёт. Для трёх и более используйте `format!`. Его проще
> читать, и он ничего не перемещает.

> ⚡ **Производительность:** каждый `push_str` может потребовать больше памяти, как и `push` у
> вектора. У `String` тоже есть длина и ёмкость, а `String::with_capacity(n)` резервирует место
> заранее (та же идея, что в [Уроке 04.7](07-vectors.md)). Не используйте `+` в цикле для построения очень длинного текста;
> лучше `push_str` для одной `String`.

## 4. Строку нельзя индексировать

Можно ожидать, что `text[0]` даст первую букву, как `numbers[0]` даёт первый элемент массива. Для строки это не компилируется.

```rust
fn main() {
    let s = String::from("hello");
    let c = s[0];
    println!("{c}");
}
```

```text
error[E0277]: the type `str` cannot be indexed by `{integer}`
 --> src/main.rs:3:15
  |
3 |     let c = s[0];
  |               ^ string indices are ranges of `usize`
  |
  = note: you can use `.chars().nth()` or `.bytes().nth()`
```

**Почему?** `s[0]` должна была бы вернуть «первый символ», но в UTF-8 символ может начинаться с
любого смещения в байтах и занимать от 1 до 4 байт. Rust не может найти 5-й символ за постоянное
время (ему пришлось бы идти по байтам с начала). Возврат «первого байта» дал бы вам половину
буквы `я`. Поэтому разработчики запретили эту операцию и заставляют вас сказать, что вы имеете
в виду.

Скажите, что имеете в виду:

- **n-й символ**: `s.chars().nth(n)` (возвращает `Option<char>`);
- **n-й байт**: `s.as_bytes()[n]` или `s.bytes().nth(n)`;
- **часть текста** по байтовым позициям: `&s[a..b]`.

Последнее разрешено, но **оба конца должны лежать на границах символов**, иначе программа
паникует.

```rust
fn main() {
    let s = String::from("Здравствуйте");
    let part = &s[0..4];
    println!("{part}");
    let bad = &s[0..3];
    println!("{bad}");
}
```

```text
Зд

thread 'main' (3885762) panicked at src/main.rs:5:17:
end byte index 3 is not a char boundary; it is inside 'д' (bytes 2..4 of string)
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
```

**Что происходит:**

1. Каждая кириллическая буква занимает 2 байта. Байты 0..4 — это ровно две буквы, `Зд`, поэтому
   первый срез работает.
2. Байты 0..3 разрезали бы вторую букву пополам. Программа паникует, и сообщение говорит, какую
   букву разрезали.

Чтобы избежать паники, используйте `get`, который возвращает `Option` (как `Vec::get`), и
`is_char_boundary`, чтобы проверить позицию:

```rust
fn main() {
    let text = "naïve café";
    println!("{}", text.is_char_boundary(3));
    println!("{}", text.is_char_boundary(2));
    println!("{:?}", text.get(0..3));
    println!("{:?}", text.get(0..4));
}
```

```text
false
true
None
Some("naï")
```

Здесь `ï` занимает байты 2 и 3, поэтому позиция 3 приходится на её середину.

> 🔒 **Безопасность:** текст от пользователя, из файла или из сети может содержать любые символы.
> Код вроде `&input[..10]` для «сокращения» паникует, когда байт 10 находится внутри буквы, а
> злоумышленник может нарочно выбрать такой ввод. Режьте текст по символам (Упражнение 5) или
> используйте `get`.

## 5. Обход текста

Есть три распространённых взгляда на один и тот же текст.

```rust
fn main() {
    let word = "Привет";
    println!("len()           = {}", word.len());
    println!("chars().count() = {}", word.chars().count());

    for c in word.chars() {
        print!("[{c}]");
    }
    println!();

    for b in "Hi!".bytes() {
        print!("{b} ");
    }
    println!();

    for (i, c) in "aя😀".char_indices() {
        println!("byte offset {i}: {c}");
    }
}
```

**Что происходит:**

1. `len()` считает байты: 6 букв × 2 байта = 12. `chars().count()` считает символы: 6.
2. `for c in word.chars()` на каждом витке даёт один `char`. `print!` — как `println!`, но без
   перевода строки в конце; затем одиночный `println!()` завершает строку.
3. `.bytes()` даёт «сырые» числа: `H` — 72, `i` — 105, `!` — 33.
4. `.char_indices()` даёт пары (смещение в байтах, символ). Смещения 0, 1, 3 соответствуют
   картинке из раздела 1: `a` начинается с 0, `я` — с 1, эмодзи — с 3.

```text
len()           = 12
chars().count() = 6
[П][р][и][в][е][т]
72 105 33 
byte offset 0: a
byte offset 1: я
byte offset 3: 😀
```

> Замечание о слове «символ». `char` — это *кодовая точка*, а не всегда то, что человек видит как
> одну букву. Некоторые буквы записываются двумя кодовыми точками (базовая буква плюс отдельный
> знак ударения), а многие эмодзи склеены из нескольких кодовых точек. Чтобы делить текст на то,
> что видят люди (графемные кластеры), нужен крейт `unicode-segmentation`; как подключать крейты,
> вы узнаете в [Уроке 05.8 — Cargo: зависимости и версии](../05-rust-intermediate/08-cargo-dependencies-and-versions.md). Для большинства программ достаточно `chars()`.

## 6. Повседневные методы

Большинство этих методов работают с `&str`, поэтому работают и с `String` (Rust позволяет вызывать
методы `&str` у `String` напрямую и передавать `&my_string` туда, где ожидается `&str`).

### Очистка и разделение

```rust
fn main() {
    let text = "  Alice, Bob,Carol ,  Dave  ";
    println!("[{}]", text.trim());

    for part in text.split(',') {
        println!("part: [{}]", part.trim());
    }

    let words: Vec<&str> = "one two   three".split_whitespace().collect();
    println!("{:?}", words);

    let line = "add alice +998-90-123-45-67";
    match line.split_once(' ') {
        Some((command, rest)) => println!("command = {command}, rest = {rest}"),
        None => println!("no space"),
    }

    let poem = "line one\nline two\nline three";
    for (n, l) in poem.lines().enumerate() {
        println!("{}: {}", n + 1, l);
    }
}
```

**Что происходит:**

1. `trim()` убирает пробелы, табуляции и переводы строк с обоих концов. (`trim_start` и `trim_end`
   работают с одной стороной.) Он не копирует: возвращает более короткий `&str`, указывающий в
   тот же текст.
2. `split(',')` режет текст по каждой запятой. Он возвращает итератор, поэтому его можно
   использовать в цикле `for`. В каждой части остаются пробелы, поэтому для каждой части мы
   вызываем `trim()`.
3. `split_whitespace()` режет по любой серии пробелов, поэтому три пробела между `two` и `three`
   не создают пустого слова.
4. `.collect()` собирает элементы итератора в коллекцию. Какую именно, мы указываем типом слева:
   `Vec<&str>`. (Как работают итераторы и `collect` — тема [Урока 05.5](../05-rust-intermediate/05-iterators.md).)
5. `split_once(' ')` режет только по **первому** пробелу и возвращает `Option<(&str, &str)>`:
   часть до и часть после. Это лучший инструмент для разбора «команда + остальное».
6. `lines()` делит текст по концам строк.

```text
[Alice, Bob,Carol ,  Dave]
part: [Alice]
part: [Bob]
part: [Carol]
part: [Dave]
["one", "two", "three"]
command = add, rest = alice +998-90-123-45-67
1: line one
2: line two
3: line three
```

### Поиск и изменение

```rust
fn main() {
    let s = "Hello, World";
    println!("{}", s.contains("World"));
    println!("{}", s.starts_with("Hello"));
    println!("{}", s.ends_with('d'));
    println!("{:?}", s.find("World"));
    println!("{:?}", s.find('z'));
    println!("{}", s.replace("l", "L"));
    println!("{}", s.to_uppercase());
    println!("{}", s.to_lowercase());
    println!("{}", "ab".repeat(3));
    println!("{:?}", "add alice".strip_prefix("add "));
    println!("{:?}", "list".strip_prefix("add "));
    println!("{}", "Alice".eq_ignore_ascii_case("ALICE"));
    println!("{:?}", "42".parse::<u32>());
    println!("{:?}", "4x2".parse::<u32>());
}
```

**Что происходит:**

1. `contains`, `starts_with` и `ends_with` отвечают на вопросы «да/нет». Они принимают `&str` или
   один `char`.
2. `find` возвращает **байтовое** смещение первого совпадения в виде `Option<usize>`: `Some(7)` для
   `World`, `None`, если совпадений нет.
3. `replace`, `to_uppercase`, `to_lowercase` и `repeat` **не** меняют исходный текст. Они строят и
   возвращают новую `String`.
4. `strip_prefix("add ")` убирает начало, если оно есть, и возвращает `Some(остаток)`; если текст
   не начинается с префикса, возвращает `None`.
5. `eq_ignore_ascii_case` сравнивает два текста, не различая регистр английских букв.
6. `parse::<u32>()` превращает текст в число. Он возвращает `Result`, то есть либо `Ok(значение)`,
   либо `Err(причина)`. `Result` вы изучите в [Уроке 04.10](10-error-handling.md).

```text
true
true
true
Some(7)
None
HeLLo, WorLd
HELLO, WORLD
hello, world
ababab
Some("alice")
None
true
Ok(42)
Err(ParseIntError { kind: InvalidDigit })
```

### Методы `char`

```rust
fn main() {
    let c = 'q';
    println!("{} {} {}", c.is_alphabetic(), c.is_numeric(), c.is_ascii_digit());
    println!("{}", c.to_ascii_uppercase());
    println!("{:?}", '7'.to_digit(10));
    println!("{:?}", 'x'.to_digit(10));
    println!("{}", 'a' as u32);
    println!("{}", 'я' as u32);
    println!("{:?}", char::from_u32(0x1F600));
    println!("{}", c.len_utf8());
    println!("{}", 'я'.len_utf8());
}
```

**Что происходит:**

1. `is_alphabetic`, `is_numeric` и `is_ascii_digit` классифицируют символ. `is_alphabetic` знает
   буквы всех алфавитов.
2. `'7'.to_digit(10)` превращает символ-цифру в число, которое он обозначает (`10` — основание
   системы счисления), или в `None`, если это не цифра.
3. `'a' as u32` показывает кодовую точку: 97. У `'я'` это 1103.
4. `char::from_u32(0x1F600)` строит символ из кодовой точки или возвращает `None`, если число не
   является корректной кодовой точкой. `0x1F600` — число, записанное в шестнадцатеричной системе
   (по основанию 16).
5. `len_utf8()` говорит, сколько байт символ занимает в UTF-8.

```text
true false false
Q
Some(7)
None
97
1103
Some('😀')
1
2
```

## 7. Тестирование кода для текста

> 🧪 **Тестирование:** в функции, работающие с текстом, всегда включайте тест с не-ASCII
> символами. Функция, которая проходит с `"hello"`, может запаниковать с `"привет"`. Вот проверка
> палиндрома (текста, который читается одинаково в обоих направлениях) с пятью тестами, включая
> кириллицу.

```rust
/// Keeps only letters and digits, in lowercase.
fn normalize(text: &str) -> String {
    let lower = text.to_lowercase();
    let mut result = String::new();
    for c in lower.chars() {
        if c.is_alphanumeric() {
            result.push(c);
        }
    }
    result
}

fn is_palindrome(text: &str) -> bool {
    let chars: Vec<char> = normalize(text).chars().collect();
    let mut reversed = chars.clone();
    reversed.reverse();
    chars == reversed
}

fn main() {
    println!("{}", is_palindrome("A man, a plan, a canal: Panama"));
    println!("{}", is_palindrome("Rust"));
    println!("{}", is_palindrome("А роза упала на лапу Азора"));
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn simple_palindrome() {
        assert!(is_palindrome("level"));
    }

    #[test]
    fn ignores_case_and_punctuation() {
        assert!(is_palindrome("A man, a plan, a canal: Panama"));
    }

    #[test]
    fn works_with_non_ascii_letters() {
        assert!(is_palindrome("А роза упала на лапу Азора"));
    }

    #[test]
    fn not_a_palindrome() {
        assert!(!is_palindrome("Rust"));
    }

    #[test]
    fn empty_is_a_palindrome() {
        assert!(is_palindrome(""));
    }
}
```

**Что происходит:**

1. `normalize` переводит весь текст в нижний регистр, затем копирует в новую `String` только
   буквы и цифры.
2. `is_palindrome` собирает символы в `Vec<char>`, делает перевёрнутую копию через `clone()` и
   `reverse()` ([Урок 04.7](07-vectors.md)) и сравнивает два вектора через `==`.
3. `assert!(условие)` проваливает тест, если условие равно `false`; `assert!(!x)` ожидает `false`.

```text
true
false
true
```

```text
running 5 tests
test tests::not_a_palindrome ... ok
test tests::empty_is_a_palindrome ... ok
test tests::simple_palindrome ... ok
test tests::works_with_non_ascii_letters ... ok
test tests::ignores_case_and_punctuation ... ok

test result: ok. 5 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

## 8. Другие строковые типы, которые вам встретятся

`String` и `&str` всегда содержат корректный UTF-8. Внешний мир менее аккуратен, поэтому в
стандартной библиотеке есть и другие типы. Они вам пока не нужны; эта таблица лишь говорит, что
они существуют.

| Тип | Для чего | Где вы с ним встретитесь |
|---|---|---|
| `OsString` / `&OsStr` | Имена, которые вам даёт операционная система (имена файлов, аргументы командной строки), они могут не быть корректным UTF-8 | работа с файлами и путями в следующих этапах |
| `PathBuf` / `&Path` | Пути в файловой системе | работа с файлами в следующих этапах |
| `CString` / `&CStr` | Текст для кода на C, который заканчивается нулевым байтом | [Урок 06.10 — FFI](../06-rust-advanced/10-ffi-calling-c.md) |
| `Vec<u8>` / `&[u8]` | Сырые байты, которые не обязательно являются текстом | сетевые и файловые форматы |

(FFI — Foreign Function Interface, интерфейс внешних функций: вызов кода, написанного на другом
языке. Он объясняется в [Уроке 06.10](../06-rust-advanced/10-ffi-calling-c.md).)

## Частые ошибки

**1. Сложение двух `String` через `+`.**

```rust
fn main() {
    let a = String::from("Hello, ");
    let b = String::from("world");
    let c = a + b;
    println!("{c}");
}
```

```text
error[E0308]: mismatched types
 --> src/main.rs:4:17
  |
4 |     let c = a + b;
  |                 ^ expected `&str`, found `String`
  |
help: consider borrowing here
  |
4 |     let c = a + &b;
  |                 +
```

Справа от `+` должен быть `&str`. Исправление: напишите `a + &b`.

**2. Использование левой части после `+`.**

```rust
fn main() {
    let a = String::from("Hello, ");
    let b = String::from("world");
    let c = a + &b;
    println!("{c} {a}");
}
```

```text
error[E0382]: borrow of moved value: `a`
 --> src/main.rs:5:20
  |
4 |     let c = a + &b;
  |             - value moved here
5 |     println!("{c} {a}");
  |                    ^ value borrowed here after move
  |
help: consider cloning the value if the performance cost is acceptable
  |
4 |     let c = a.clone() + &b;
  |              ++++++++
```

`+` забирает владение `a`. Исправление: используйте `format!("{a}{b}")`, который заимствует обе
строки, или `a.clone() + &b`.

**3. Индексация числом** (`s[0]`): см. раздел 4. Используйте `chars().nth(0)`.

**4. Срез посреди символа:** см. раздел 4. Используйте `get` или работайте через `chars()`.

**5. Мысль, что `len()` — это число букв.** Это число байт. Для символов используйте
`chars().count()`.

**6. Забыли, что `trim()` не меняет оригинал.** Строка `s.trim();` сама по себе ничего полезного
не делает. Пишите `let s = s.trim();`.

## Попробуйте сами

В `code/p04-strings` замените `src/main.rs` крошечным разборщиком команд — той же идеей, которую
вы используете в проекте телефонной книги:

```rust
fn main() {
    let lines = ["add alice +998-90-123-45-67", "  get   alice  ", "count", ""];

    for line in lines {
        let line = line.trim();
        if line.is_empty() {
            println!("(empty line)");
            continue;
        }
        match line.split_once(' ') {
            Some((command, rest)) => println!("command {:?} with argument {:?}", command, rest.trim()),
            None => println!("command {:?} without arguments", line),
        }
    }
}
```

1. Запустите командой `cargo run`. Ожидаемый вывод:

```text
command "add" with argument "alice +998-90-123-45-67"
command "get" with argument "alice"
command "count" without arguments
(empty line)
```

2. Измените ввод так, чтобы строка начиналась с команды в верхнем регистре, `"ADD bob 123"`.
   Сделайте так, чтобы программа печатала команду в нижнем регистре через `to_lowercase()`.
3. Добавьте строку с символом табуляции между командой и аргументом (`"get\talice"`). Найдёт ли
   `split_once(' ')` табуляцию? (Нет. Попробуйте `split_whitespace()` или
   `split_once(char::is_whitespace)`.)
4. Сохраните работу в Git: `git add .` и `git commit -m "Practice string parsing"`.

## Упражнения

### Упражнение 1 — Байты и символы (легко)

Для каждого из слов `"hello"`, `"привет"` и `"日本語"` напечатайте число байт и число символов.
Ожидаемый вывод:

```text
hello: 5 bytes, 5 chars
привет: 12 bytes, 6 chars
日本語: 9 bytes, 3 chars
```

<details>
<summary>Подсказка</summary>

Положите слова в массив и используйте цикл `for`. `len()` считает байты; `chars().count()`
считает символы.

</details>

<details>
<summary>Ответ</summary>

```rust
fn main() {
    for word in ["hello", "привет", "日本語"] {
        println!("{word}: {} bytes, {} chars", word.len(), word.chars().count());
    }
}
```

Почему это работает: латинские буквы занимают по 1 байту, кириллические — по 2 байта, а эти
японские иероглифы — по 3 байта.

</details>

### Упражнение 2 — Соберите приветствие (легко)

Пусть даны `let name = String::from("Sara");` и `let city = "Samarkand";`. Соберите текст
`Hello, Sara. Welcome to Samarkand!` двумя способами: через `push_str`/`push`/`+=` и через
`format!`. Напечатайте первый результат и напечатайте, равны ли два результата.

<details>
<summary>Подсказка</summary>

Начните с `String::from("Hello, ")`. Используйте `push_str(&name)` (передайте ссылку),
`+= ". Welcome to "`, `push_str(city)` и `push('!')`. Две `String` сравниваются через `==`.

</details>

<details>
<summary>Ответ</summary>

```rust
fn main() {
    let name = String::from("Sara");
    let city = "Samarkand";

    let mut greeting = String::from("Hello, ");
    greeting.push_str(&name);
    greeting += ". Welcome to ";
    greeting.push_str(city);
    greeting.push('!');
    println!("{greeting}");

    let same = format!("Hello, {name}. Welcome to {city}!");
    println!("{}", greeting == same);
}
```

```text
Hello, Sara. Welcome to Samarkand!
true
```

Почему это работает: `push_str` заимствует добавляемый текст, поэтому `name` остаётся
пригодной. `format!` даёт тот же результат с меньшим количеством кода.

</details>

### Упражнение 3 — Инициалы, с тестами (средне)

Напишите `fn initials(name: &str) -> String`, которая превращает `"Ali Valiyev"` в `"A.V."`.
Лишние пробелы нужно игнорировать, буквы должны становиться заглавными, а пустой ввод даёт пустую
`String`. Напишите тесты: два слова; лишние пробелы и строчные буквы; пустой ввод; имя не из ASCII
(`"олег иванов"` даёт `"О.И."`).

<details>
<summary>Подсказка</summary>

Пройдите циклом по `name.split_whitespace()`. Для каждого слова `word.chars().next()` возвращает
`Option<char>` (первый символ, если он есть). Используйте `if let Some(first) = ...`. Перевод в
верхний регистр: `first.to_uppercase()` возвращает то, по чему можно пройти циклом, потому что
один символ в верхнем регистре может превратиться в несколько. Добавьте каждый результат, а затем
добавьте `'.'`.

</details>

<details>
<summary>Ответ</summary>

```rust
/// "Ali Valiyev" -> "A.V."
fn initials(name: &str) -> String {
    let mut result = String::new();
    for word in name.split_whitespace() {
        if let Some(first) = word.chars().next() {
            for upper in first.to_uppercase() {
                result.push(upper);
            }
            result.push('.');
        }
    }
    result
}

fn main() {
    println!("{}", initials("Ali Valiyev"));
    println!("{}", initials("  mary   jane watson "));
    println!("[{}]", initials(""));
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn two_words() {
        assert_eq!(initials("Ali Valiyev"), "A.V.");
    }

    #[test]
    fn extra_spaces_and_lowercase() {
        assert_eq!(initials("  mary   jane watson "), "M.J.W.");
    }

    #[test]
    fn empty_input() {
        assert_eq!(initials(""), "");
    }

    #[test]
    fn non_ascii() {
        assert_eq!(initials("олег иванов"), "О.И.");
    }
}
```

```text
A.V.
M.J.W.
[]
```

Почему это работает: `split_whitespace` убирает пустые куски, поэтому лишние пробелы не
мешают. У слова всегда есть первый символ, но `chars().next()` возвращает `Option`, поэтому код
обрабатывает случай «символа нет» без малейшей возможности паники. `assert_eq!` умеет сравнивать
`String` с `&str`.

</details>

### Упражнение 4 — Разбор `key = value` (средне)

Напишите `fn parse_kv(line: &str) -> Option<(&str, &str)>` для строк конфигурации.
`"name = Ali"` даёт `Some(("name", "Ali"))` (пробелы вокруг ключа и значения убираются). Строка
без `=` даёт `None`. Пустой ключ даёт `None`. Строку делит только **первый** `=`: `"a=b=c"` даёт
`Some(("a", "b=c"))`. Напишите тест на каждое правило.

<details>
<summary>Подсказка</summary>

`line.split_once('=')` возвращает `Option<(&str, &str)>`. Используйте
`let Some((key, value)) = ... else { return None; };` ([Урок 04.5](05-if-let-let-else-and-patterns.md)). Затем сделайте `trim()` обеим частям и проверьте
`key.is_empty()`.

</details>

<details>
<summary>Ответ</summary>

```rust
fn parse_kv(line: &str) -> Option<(&str, &str)> {
    let Some((key, value)) = line.split_once('=') else {
        return None;
    };
    let key = key.trim();
    if key.is_empty() {
        return None;
    }
    Some((key, value.trim()))
}

fn main() {
    println!("{:?}", parse_kv("name = Ali"));
    println!("{:?}", parse_kv("port=8080"));
    println!("{:?}", parse_kv("no equals sign"));
    println!("{:?}", parse_kv(" = value"));
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn trims_both_sides() {
        assert_eq!(parse_kv("  name =  Ali "), Some(("name", "Ali")));
    }

    #[test]
    fn missing_equals() {
        assert_eq!(parse_kv("hello"), None);
    }

    #[test]
    fn empty_key() {
        assert_eq!(parse_kv("=x"), None);
    }

    #[test]
    fn value_may_contain_equals() {
        assert_eq!(parse_kv("a=b=c"), Some(("a", "b=c")));
    }
}
```

```text
Some(("name", "Ali"))
Some(("port", "8080"))
None
None
```

Почему это работает: результат содержит срезы входной строки, а не копии, поэтому память не
выделяется. (Почему возвращаемые ссылки корректны, объясняется в [Уроке 05.3 — Времена жизни](../05-rust-intermediate/03-lifetimes.md).)

</details>

### Упражнение 5 — Безопасное усечение (сложно)

Напишите `fn truncate_chars(s: &str, n: usize) -> &str`, которая оставляет не более `n`
**символов** и никогда не паникует, даже для кириллицы или эмодзи. Тесты: `("hello world", 5)`
даёт `"hello"`; `("привет мир", 6)` даёт `"привет"`; `("hi", 10)` даёт `"hi"`; `("abc", 0)` даёт
`""`; `("a😀b", 2)` даёт `"a😀"`.

<details>
<summary>Подсказка</summary>

`s.char_indices().nth(n)` возвращает `n`-ю пару `(смещение_в_байтах, char)` в виде `Option`.
Если это `Some((i, _))`, то текст до байта `i` — ровно `n` символов, поэтому верните `&s[..i]`. Если
это `None`, то в тексте `n` символов или меньше, поэтому верните весь `s`.

</details>

<details>
<summary>Ответ</summary>

```rust
/// Keeps at most `n` characters. Never panics, even for non-ASCII text.
fn truncate_chars(s: &str, n: usize) -> &str {
    match s.char_indices().nth(n) {
        Some((byte_index, _)) => &s[..byte_index],
        None => s,
    }
}

fn main() {
    println!("{}", truncate_chars("hello world", 5));
    println!("{}", truncate_chars("привет мир", 6));
    println!("{}", truncate_chars("hi", 10));
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn ascii() {
        assert_eq!(truncate_chars("hello world", 5), "hello");
    }

    #[test]
    fn cyrillic() {
        assert_eq!(truncate_chars("привет мир", 6), "привет");
    }

    #[test]
    fn shorter_than_limit() {
        assert_eq!(truncate_chars("hi", 10), "hi");
    }

    #[test]
    fn zero() {
        assert_eq!(truncate_chars("abc", 0), "");
    }

    #[test]
    fn emoji() {
        assert_eq!(truncate_chars("a😀b", 2), "a😀");
    }
}
```

```text
hello
привет
hi
```

Почему это работает: `char_indices` выдаёт только смещения, которые являются настоящими
границами символов, поэтому `&s[..byte_index]` никогда не разрежет символ пополам.

</details>

## Вопросы для самопроверки

1. Почему `"привет".len()` равно 12, а не 6?

<details>
<summary>Ответ</summary>

`len()` возвращает число байт. В UTF-8 каждая кириллическая буква занимает 2 байта, а букв 6.
`chars().count()` возвращает 6.

</details>

2. Чем `char` отличается от `u8`?

<details>
<summary>Ответ</summary>

`u8` — это один байт (от 0 до 255). `char` — это одна кодовая точка Unicode, он занимает 4 байта; в
строке UTF-8 один `char` занимает от 1 до 4 байт.

</details>

3. Почему Rust не разрешает `s[0]` для `String`?

<details>
<summary>Ответ</summary>

В UTF-8 у символов разная длина в байтах, поэтому байтовый индекс не соответствует символу, а
поиск n-го символа требует прохода по тексту. Rust заставляет вас выбрать, что вы имеете в
виду: `chars().nth(n)`, `bytes()` или срез по диапазону байт.

</details>

4. Чем `a + &b` отличается от `format!("{a}{b}")`?

<details>
<summary>Ответ</summary>

`a + &b` перемещает `a` (после этого её использовать нельзя) и переиспользует её буфер.
`format!` только заимствует обе строки и строит новую `String`, поэтому `a` и `b` остаются
пригодными.

</details>

5. Что произойдёт при `&s[0..3]`, если второй символ `s` — это `д` (2 байта)?

<details>
<summary>Ответ</summary>

Диапазон заканчивается внутри символа, поэтому программа паникует с сообщением «is not a char
boundary». Используйте `s.get(0..3)`, чтобы получить `Option`.

</details>

6. Какой метод делит `"add alice 555"` на `"add"` и `"alice 555"`?

<details>
<summary>Ответ</summary>

`split_once(' ')`. Он делит только по первому пробелу и возвращает `Some(("add", "alice 555"))`.

</details>

7. Меняет ли `s.to_uppercase()` строку `s`?

<details>
<summary>Ответ</summary>

Нет. Он возвращает новую `String`, а оригинал остаётся как был.

</details>

## Итоги

- Текст хранится в виде байтов. Строки Rust — всегда корректный UTF-8, где символ занимает от 1 до
  4 байт.
- `len()` — это байты; `chars().count()` — символы.
- `String` принадлежит вам и растёт; `&str` — заимствованный взгляд. Создавайте строки через
  `String::from`, `to_string` или `format!`.
- Используйте `push` для символа, `push_str` или `+=` для текста, `format!` для соединения многих
  частей без перемещения и избегайте `+` для более чем двух частей.
- Нельзя писать `s[0]`. Используйте `chars()`, `bytes()`, `char_indices()` или срез с корректными
  границами; для безопасности — `get`.
- `trim`, `split`, `split_whitespace`, `split_once`, `lines`, `find`, `replace` и `parse`
  выполняют большую часть ежедневной работы; ни один из них не меняет исходный текст.
- Всегда тестируйте функции для текста на не-ASCII вводе и никогда не режьте недоверенный текст по
  «сырым» байтовым позициям.

## Источники и что почитать

- [The Rust Programming Language — 8.2 Storing UTF-8 Encoded Text with Strings](https://doc.rust-lang.org/book/ch08-02-strings.html) — глава официальной книги о строках.
- [std::string::String](https://doc.rust-lang.org/std/string/struct.String.html) — все методы `String`.
- [std::primitive::str](https://doc.rust-lang.org/std/primitive.str.html) — все методы `&str`, включая `split`, `trim` и `find`.
- [std::primitive::char](https://doc.rust-lang.org/std/primitive.char.html) — все методы `char`.
- [Unicode FAQ — UTF-8, UTF-16, UTF-32 and BOM](https://www.unicode.org/faq/utf_bom.html) — ответы на вопросы об UTF-8, UTF-16 и порядке байтов.
- [Rust by Example — Strings](https://doc.rust-lang.org/rust-by-example/std/str.html) — короткие примеры, которые можно запустить.

---

[🏠 Начало](../START-HERE.md) · [← 04.7 Векторы](07-vectors.md) · [📋 Этап 04](README.md) · [04.9 Хеш-таблицы (HashMap) →](09-hash-maps.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
