<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 05.2 Трейты](02-traits.md) · [📋 Этап 05](README.md) · [05.4 Замыкания →](04-closures.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/05-rust-intermediate/03-lifetimes.md) <!-- nav -->

# Урок 05.3 — Времена жизни

> **Этап 05 — Rust: средний уровень** · Урок 3 из 14 · ⏱ около 4 часов (2 занятия)

## Что вы узнаете

- Что такое время жизни (lifetime) и какую проблему компилятор решает с его помощью (висячие ссылки).
- Как читать и писать аннотации времён жизни вроде `'a` в функциях и структурах.
- Три *правила пропуска* (elision), благодаря которым большинство аннотаций можно не писать.
- Что значит `'static` и как исправлять типичные ошибки времён жизни.
- Как времена жизни позволяют писать быстрый код «без копирования» (zero-copy), возвращающий куски своих входных данных.

## Перед началом

- Вы прошли [Урок 05.2 — Трейты](02-traits.md). Времена жизни — это вид обобщённого параметра,
  поэтому [Урок 05.1 — Обобщённые типы](01-generics.md) здесь тоже важен.
- Вы понимаете ссылки и правила заимствования ([Урок 03.4](../03-ownership-and-borrowing/04-references-and-borrowing.md)
  и [Урок 03.5](../03-ownership-and-borrowing/05-mutable-references-and-the-borrow-rules.md))
  и строковые срезы ([Урок 03.6](../03-ownership-and-borrowing/06-slices-and-string-types.md)).
- У вас есть пакет `code/p05-lessons` из Урока 05.1.

## Новые слова

| Слово | Значение |
|---|---|
| **время жизни** (lifetime) | Часть программы, в течение которой ссылка действительна: от создания до последнего использования. Значение, на которое она указывает, должно жить как минимум столько же. |
| **аннотация времени жизни** | Имя вроде `'a` (апостроф и короткое имя), которое пишут у ссылок, чтобы описать, как связаны их времена жизни. |
| **параметр времени жизни** | Объявление такого имени в угловых скобках: `'a` в `fn longest<'a>(…)`. |
| **висячая ссылка** (dangling reference) | Ссылка на значение, которого уже нет. Rust делает её невозможной. |
| **проверка заимствований** (borrow checker) | Часть компилятора, проверяющая все правила заимствования и времён жизни (Урок 03.4). |
| **пропуск времён жизни** (lifetime elision) | Правила компилятора, которые подставляют очевидные времена жизни, чтобы вам не приходилось их писать. |
| **`'static`** | Самое долгое время жизни: весь запуск программы. Оно есть у строковых литералов. |
| **`'_`** | «Анонимное время жизни»: заполнитель, означающий «какое-то время жизни, выведи по правилам пропуска». |
| **zero-copy (без копирования)** | Чтение данных через указатели на оригинал вместо копирования в новые строки. |
| **NLL** (Non-Lexical Lifetimes, нелексические времена жизни) | Способ компилятора заканчивать заимствование в *последнем использовании* ссылки, а не в конце блока. |

## 1. Проблема: ссылка на то, чего уже нет

В Уроке 03.4 вы узнали, что ссылка заимствует значение и не должна его пережить. Вот ошибка, от
которой вас защищает компилятор:

```rust
fn main() {
    let r;
    {
        let x = 5;
        r = &x;
    }
    println!("r: {r}");
}
```

```text
error[E0597]: `x` does not live long enough
 --> examples/dangling.rs:5:13
  |
4 |         let x = 5;
  |             - binding `x` declared here
5 |         r = &x;
  |             ^^ borrowed value does not live long enough
6 |     }
  |     - `x` dropped here while still borrowed
7 |     println!("r: {r}");
  |                   - borrow later used here
```

**Что происходит:**

1. `r` объявлена во внешнем блоке. `x` живёт во внутреннем блоке `{ … }`.
2. На закрывающей `}` внутреннего блока `x` уничтожается (Урок 03.2: значение уничтожается, когда
   его владелец выходит из области видимости).
3. `r` по-прежнему указывает на память, где была `x`. Использование её после этого читало бы мусор.
   Компилятор останавливает это на этапе сборки.

Та же история в виде картинки. **Время жизни** ссылки — это отрезок кода, в течение которого её
используют; значение должно жить как минимум столько же:

```text
fn main() {
    let r;                  // ──────────────┐  r is used until here
    {                       //               │
        let x = 5;          // ─────┐        │
        r = &x;             //      │ x      │  <- r borrows x,
    }                       // ─────┘ ends   │     but x ends first!
    println!("r: {r}");     // ──────────────┘
}
```

Итак, в общем случае **время жизни — это ответ на вопрос «как долго эту ссылку разрешено
использовать?»**. В большинстве случаев компилятор вычисляет его сам. Этот урок — про случаи, когда
ему нужна ваша помощь.

## 2. Аннотации времён жизни в функциях

Напишем функцию, которая возвращает более длинный из двух строковых срезов:

```rust
fn longest(x: &str, y: &str) -> &str {
    if x.len() > y.len() {
        x
    } else {
        y
    }
}
```

```text
error[E0106]: missing lifetime specifier
 --> examples/longest_err.rs:1:33
  |
1 | fn longest(x: &str, y: &str) -> &str {
  |               ----     ----     ^ expected named lifetime parameter
  |
  = help: this function's return type contains a borrowed value, but the signature does not say whether it is borrowed from `x` or `y`
help: consider introducing a named lifetime parameter
  |
1 | fn longest<'a>(x: &'a str, y: &'a str) -> &'a str {
  |           ++++     ++          ++          ++
```

**Что это значит:** функция возвращает ссылку. Ссылка должна указывать на что-то, чем владеет кто-то
другой. Сигнатура не говорит, указывает ли результат внутрь `x` или внутрь `y`, а компилятору нужно
знать это, потому что от ответа зависит, как долго результат можно использовать. Здесь ответ зависит
от длин во время выполнения, поэтому компилятор не может угадать. Вы должны описать это:

```rust
fn longest<'a>(x: &'a str, y: &'a str) -> &'a str {
    if x.len() > y.len() {
        x
    } else {
        y
    }
}

fn main() {
    let first = String::from("lifetimes");
    let second = String::from("rust");
    let result = longest(first.as_str(), second.as_str());
    println!("the longest is {result}");
}
```

```text
the longest is lifetimes
```

**Что происходит:**

1. `<'a>` объявляет **параметр времени жизни** с именем `'a`. Он работает как `<T>` из Урока 05.1,
   только для времён жизни вместо типов. По соглашению имена времён жизни короткие, в нижнем
   регистре и начинаются с апострофа.
2. `x: &'a str` и `y: &'a str` означают: «обе ссылки действительны как минимум в течение времени
   жизни `'a`».
3. `-> &'a str` означает: «результат действителен в течение того же `'a`».
4. Вместе: **результат живёт не дольше, чем та из `x` и `y`, которая живёт меньше**. `'a` становится
   пересечением двух входных времён жизни.

> 💡 **Совет:** читайте `<'a>` как «есть некий промежуток времени, назовём его `a`». Аннотации
> связывают ссылки друг с другом. Они ничему не добавляют времени.

### Аннотации описывают, но ничего не меняют

Аннотация времени жизни никогда не заставляет значение жить дольше. Она лишь сообщает компилятору,
как связаны времена жизни разных ссылок, чтобы он мог проверить безопасность использования
результата. Чтобы увидеть проверку в деле, вызовем `longest` с двумя строками, жизнь которых
заканчивается в разное время:

```rust
fn main() {
    let first = String::from("lifetimes");
    let result;
    {
        let second = String::from("rust");
        result = longest(first.as_str(), second.as_str());
    }
    println!("the longest is {result}");
}
```

```text
error[E0597]: `second` does not live long enough
  --> examples/longest_scope_err.rs:14:42
   |
13 |         let second = String::from("rust");
   |             ------ binding `second` declared here
14 |         result = longest(first.as_str(), second.as_str());
   |                                          ^^^^^^ borrowed value does not live long enough
15 |     }
   |     - `second` dropped here while still borrowed
16 |     println!("the longest is {result}");
   |                               ------ borrow later used here
```

**Что это значит:** `result` может указывать на `second` (мы знаем, что не указывает, но сигнатура
обещает лишь «меньшее из двух»). `second` уничтожается на `}`, а `result` используется после этого.
Исправление — использовать `result` внутри блока, где `second` ещё жива:

```rust
fn main() {
    let first = String::from("lifetimes");
    {
        let second = String::from("rust");
        let result = longest(first.as_str(), second.as_str());
        println!("the longest is {result}");
    }
}
```

```text
the longest is lifetimes
```

### Связывайте только то, что действительно связано

Не всем параметрам нужно одно и то же время жизни. Если результат может прийти только из `x`,
скажите это, и другой аргумент останется свободным:

```rust
fn first_of<'a>(x: &'a str, _y: &str) -> &'a str {
    x
}

fn main() {
    let a = String::from("kept");
    let result;
    {
        let b = String::from("temporary");
        result = first_of(&a, &b);
    }
    println!("{result}");
}
```

```text
kept
```

**Что происходит:** результат привязан только ко времени жизни `'a` у `x`. Временная `b` может
умереть раньше, чем будет использован `result`. Имя `_y` начинается с подчёркивания, чтобы сказать
компилятору «не используется намеренно». Это показывает идею, которую стоит запомнить: **сигнатура —
это обещание о том, что делает функция**, и вызывающие полагаются только на сигнатуру (но никогда на
тело).

### Нельзя вернуть ссылку на локальное значение

```rust
fn shout<'a>(word: &'a str) -> &'a str {
    let loud = word.to_uppercase();
    loud.as_str()
}
```

```text
error[E0515]: cannot return value referencing local variable `loud`
 --> examples/return_local_err.rs:3:5
  |
3 |     loud.as_str()
  |     ----^^^^^^^^^
  |     |
  |     returns a value referencing data owned by the current function
  |     `loud` is borrowed here
```

**Что это значит:** `loud` создаётся внутри функции и уничтожается, когда функция заканчивается.
Ссылка на неё стала бы висячей. Никакая аннотация времени жизни это не исправит. Решение —
**вернуть значение во владение**, чтобы владельцем стал вызывающий:

```rust
fn shout(word: &str) -> String {
    word.to_uppercase()
}

fn main() {
    println!("{}", shout("hi"));
}
```

```text
HI
```

Похожая ошибка — `fn make_greeting() -> &String { … }` вообще без параметров. Компилятор говорит
`error[E0106]: missing lifetime specifier … this function's return type contains a borrowed value, but
there is no value for it to be borrowed from` и предлагает верное решение: «вероятнее всего, вы
хотите вернуть значение во владение» (you are more likely to want to return an owned value).

### Сигнатура, которая лжёт

Если тело возвращает то, что сигнатура не разрешает, компилятор это поймает:

```rust
fn pick_first<'a, 'b>(x: &'a str, y: &'b str) -> &'a str {
    if x.is_empty() {
        y
    } else {
        x
    }
}
```

```text
error: lifetime may not live long enough
 --> examples/mismatch_err.rs:3:9
  |
1 | fn pick_first<'a, 'b>(x: &'a str, y: &'b str) -> &'a str {
  |               --  -- lifetime `'b` defined here
  |               |
  |               lifetime `'a` defined here
2 |     if x.is_empty() {
3 |         y
  |         ^ function was supposed to return data with lifetime `'a` but it is returning data with lifetime `'b`
```

**Что это значит:** сигнатура обещает результат, привязанный к `x`, а тело может вернуть `y`.
Исправление — использовать одно и то же время жизни для обоих (как в `longest`) или возвращать
только `x`.

## 3. Структуры, которые хранят ссылки

Структура может хранить ссылку, но тогда ей нужен параметр времени жизни: структура не должна жить
дольше данных, на которые указывает.

```rust
#[derive(Debug)]
struct Excerpt<'a> {
    part: &'a str,
}

impl<'a> Excerpt<'a> {
    fn level(&self) -> usize {
        self.part.len()
    }

    fn announce(&self, note: &str) -> &str {
        println!("note: {note}");
        self.part
    }
}

fn main() {
    let novel = String::from("Call me Ishmael. Some years ago...");
    let first_sentence = novel.split('.').next().unwrap();
    let e = Excerpt { part: first_sentence };
    println!("{:?}", e);
    println!("level = {}", e.level());
    println!("{}", e.announce("reading"));
}
```

**Что происходит:**

1. `struct Excerpt<'a>` объявляет параметр времени жизни, а `part: &'a str` его использует. Это
   значит: «`Excerpt` не может жить дольше строки, на которую указывает».
2. `impl<'a> Excerpt<'a>` снова объявляет `'a` для блока `impl` (тот же приём, что `impl<T>
   Point<T>` в Уроке 05.1) и использует его в `Excerpt<'a>`.
3. `novel.split('.')` режет текст по каждой точке, а `.next().unwrap()` берёт первый кусок (Урок
   05.5 объясняет `.next()`). Кусок — это `&str`, указывающий **внутрь** `novel`; новая строка не
   выделялась.
4. `e` хранит эту ссылку, поэтому `e` не должна пережить `novel`. Компилятор это проверяет.
5. `announce` принимает `&self` и `note: &str` и возвращает `&str`. Мы не писали аннотации; в
   следующем разделе объясняется, почему это работает.

```text
Excerpt { part: "Call me Ishmael" }
level = 15
note: reading
Call me Ishmael
```

Без параметра компилятор подскажет, что делать:

```rust
struct Excerpt {
    part: &str,
}
```

```text
error[E0106]: missing lifetime specifier
 --> examples/excerpt_err.rs:2:11
  |
2 |     part: &str,
  |           ^ expected named lifetime parameter
  |
help: consider introducing a named lifetime parameter
  |
1 ~ struct Excerpt<'a> {
2 ~     part: &'a str,
  |
```

> 📏 **Соглашение:** новички часто тянутся к ссылкам внутри структур. Если из-за этого код
> усложняется, храните вместо этого владеющую `String`. Структура со ссылкой уместна, когда она —
> недолговечное «представление» (результат разбора, токен, строка лога), как в разделе 6.

## 4. Пропуск времён жизни: почему `'a` пишут редко

Вы писали много функций со ссылками и ни разу не нуждались в `'a`. Например:

```rust
fn first_word(s: &str) -> &str {
    match s.find(' ') {
        Some(i) => &s[..i],
        None => s,
    }
}

// the same function with the lifetimes written out
fn first_word_explicit<'a>(s: &'a str) -> &'a str {
    match s.find(' ') {
        Some(i) => &s[..i],
        None => s,
    }
}

fn main() {
    println!("{}", first_word("hello brave world"));
    println!("{}", first_word_explicit("single"));
}
```

```text
hello
single
```

Для компилятора обе функции одинаковы. В ранних версиях Rust длинная форма требовалась каждый раз,
но одни и те же шаблоны повторялись снова и снова, и компилятор научился подставлять их сам. Это
**правила пропуска времён жизни (lifetime elision)**. Компилятор применяет их по порядку, и если
после всех трёх время жизни результата всё ещё неизвестно, он сообщает `E0106` и просит вас
аннотировать.

| № | Правило | Пример (что вы пишете → что видит компилятор) |
|---|---|---|
| 1 | Каждый параметр-ссылка получает **своё** время жизни. | `fn f(a: &str, b: &str)` → `fn f<'a, 'b>(a: &'a str, b: &'b str)` |
| 2 | Если входное время жизни **ровно одно**, оно отдаётся всем выходным ссылкам. | `fn f(a: &str) -> &str` → `fn f<'a>(a: &'a str) -> &'a str` |
| 3 | Если один из входов — `&self` или `&mut self`, **его** время жизни отдаётся всем выходным ссылкам. | `fn announce(&self, n: &str) -> &str` → результат получает время жизни `self` |

Применим правила к примерам:

- `first_word(s: &str) -> &str`: правило 1 даёт `s` время жизни, правило 2 отдаёт его результату. ✓
- `longest(x: &str, y: &str) -> &str`: правило 1 даёт два разных времени жизни; правило 2 не
  применимо (два входа); правило 3 не применимо (нет `self`). Результат неизвестен. ✗ поэтому нужно
  аннотировать.
- `Excerpt::announce(&self, note: &str) -> &str`: применимо правило 3, значит, результат привязан к
  `self`. ✓

Анонимное время жизни `'_` говорит «время жизни, выведенное по правилам». Оно полезно в типах, где
`<'a>` было бы шумом:

```rust
fn parse_line(line: &str) -> Option<LogLine<'_>>
```

читается как «возвращает `LogLine`, который заимствует из входа», без называния времени жизни.

## 5. Особое время жизни `'static`

`'static` — время жизни всего запуска программы. Оно есть у **строковых литералов**, потому что текст
хранится прямо в файле программы и существует, пока существует программа:

```rust
fn motto() -> &'static str {
    "make it work, make it right, make it fast"
}

fn main() {
    let s: &'static str = "string literals live for the whole program";
    println!("{s}");
    println!("{}", motto());
}
```

```text
string literals live for the whole program
make it work, make it right, make it fast
```

> ⚠️ **Частая ошибка:** когда компилятор жалуется на времена жизни, не «исправляйте» это, повсюду
> дописывая `'static`. Это обещает, что данные живут всю программу, а для вычисленных вами данных это
> почти никогда не так; вы лишь получите другую ошибку. Сначала спросите: *кто владеет этими данными
> и как долго?* Обычно верное решение — вернуть значение во владение (`String`) или продлить жизнь
> владельцу.

С `'static` вы встретитесь снова в Уроке 05.10: поток, запущенный через `thread::spawn`, может пережить
функцию, которая его создала, поэтому данные, которые он использует, должны быть во владении потока
(или `'static`).

## 6. Практика: разбор без копирования

Системные программы часто читают большие входные данные (логи, сетевые пакеты, файлы). Копировать
каждое поле в новую `String` стоит памяти и времени. С временами жизни разборщик может вернуть
**срезы, указывающие внутрь исходного текста**:

```rust
#[derive(Debug)]
struct LogLine<'a> {
    level: &'a str,
    module: &'a str,
    message: &'a str,
}

fn parse_line(line: &str) -> Option<LogLine<'_>> {
    let (level, rest) = line.split_once(' ')?;
    let (module, message) = rest.split_once(": ")?;
    Some(LogLine { level, module, message })
}

fn main() {
    let text = String::from("ERROR net: connection refused");
    match parse_line(&text) {
        Some(line) => {
            println!("{:?}", line);
            println!("level={} module={} message={}", line.level, line.module, line.message);
        }
        None => println!("not a log line"),
    }
    println!("{:?}", parse_line("garbage").is_none());
}
```

**Что происходит:**

1. `split_once(' ')` разделяет `&str` по первому пробелу и возвращает `Option<(&str, &str)>`: обе
   части — срезы оригинала (Урок 03.6). `?` возвращает `None`, если пробела нет.
2. Мы делим ещё раз по `": "`, чтобы отделить модуль от сообщения.
3. `LogLine { level, module, message }` использует *сокращённую запись полей* (Урок 04.1): имена
   переменных совпадают с именами полей.
4. Результат заимствует из `line` (`LogLine<'_>`). Ничего не копировалось; три поля — это просто
   указатели и длины внутри `text`.
5. У `parse_line("garbage")` нет пробела, поэтому возвращается `None`.

```text
LogLine { level: "ERROR", module: "net", message: "connection refused" }
level=ERROR module=net message=connection refused
true
```

Цена такова, что результат привязан к своему источнику. Если источник исчезает, компилятор
отказывает:

```rust
fn main() {
    let text = String::from("WARN disk: almost full");
    let line = parse_line(&text).unwrap();
    drop(text);
    println!("{:?}", line);
}
```

```text
error[E0505]: cannot move out of `text` because it is borrowed
  --> examples/logline_err.rs:15:10
   |
13 |     let text = String::from("WARN disk: almost full");
   |         ---- binding `text` declared here
14 |     let line = parse_line(&text).unwrap();
   |                           ----- borrow of `text` occurs here
15 |     drop(text);
   |          ^^^^ move out of `text` occurs here
16 |     println!("{:?}", line);
   |                      ---- borrow later used here
```

**Что это значит:** `drop(text)` уничтожил бы текст, внутрь которого указывает `line`. В языках без
такой проверки это классическая ошибка *use-after-free* (использование после освобождения) и дыра в
безопасности. В Rust это ошибка компиляции.

> ⚡ **Производительность:** разбор без копирования — причина, по которой быстрые инструменты на Rust
> (обработчики логов, парсеры, сетевые прокси) справляются с огромными входными данными почти без
> выделения памяти. Везде, где можно вернуть срезы `&str` вместо `String`, вы экономите одно выделение
> памяти на поле. Но *сначала измерьте* (Урок 05.12); понятный код со `String` — нормально, когда
> скорость не важна.

### Заимствование заканчивается на последнем использовании (NLL)

Компилятор не держит заимствование до конца блока, а только до **последнего использования** ссылки.
Это называется NLL (Non-Lexical Lifetimes):

```rust
fn main() {
    let mut text = String::from("hi");
    let r = &text;
    println!("{r}");
    text.push('!');
    println!("{text}");
}
```

```text
hi
hi!
```

**Что происходит:** `r` неизменяемо заимствует `text`. `r` в последний раз используется в первом
`println!`, так что заимствование закончилось. `text.push('!')` требует изменяемого заимствования,
что теперь разрешено. В очень старых версиях Rust эта программа не компилировалась.

## 7. Времена жизни, обобщения и ограничения трейтов вместе

Параметры времён жизни и параметры типов живут в одних угловых скобках, времена жизни идут первыми:

```rust
use std::fmt::Display;

fn longest_with_note<'a, T>(x: &'a str, y: &'a str, note: T) -> &'a str
where
    T: Display,
{
    println!("note: {note}");
    if x.len() >= y.len() {
        x
    } else {
        y
    }
}

fn main() {
    println!("{}", longest_with_note("alpha", "be", "comparing"));
}
```

```text
note: comparing
alpha
```

**Что происходит:** `<'a, T>` объявляет время жизни и тип. У `note` свой тип `T` (здесь `&str`),
и он никак не связан с `'a`, потому что в результат могут попасть только `x` и `y`. Урок 06.5
возвращается к временам жизни с более сложными случаями (ограничения вроде `T: 'a`, времена жизни в
трейтах).

## Частые ошибки

**1. Нет времени жизни у результата функции** (`E0106`, раздел 2). Добавьте `<'a>` и свяжите входы,
из которых может заимствовать результат.

**2. Нет времени жизни у поля структуры** (`E0106`, раздел 3). Добавьте `<'a>` к структуре и к `impl`.

**3. Использование результата после исчезновения данных** (`E0597`, разделы 1 и 2). Переместите
использование внутрь области видимости владельца или продлите жизнь владельцу (объявите его раньше).

**4. Возврат ссылки на локальную переменную** (`E0515`, раздел 2). Верните значение во владение
(`String`, `Vec`, …).

**5. Сигнатура обещает больше, чем может выполнить тело** (`lifetime may not live long enough`).
Согласуйте времена жизни с тем, что возвращает тело.

**6. Уничтожение или изменение источника, пока жив «вид» на него** (`E0505`, раздел 6). Держите
источник живым до последнего использования вида.

## Попробуйте сами

На этот раз вы потренируете два инструмента Git, полезные, когда код не компилируется: одноразовую
**экспериментальную ветку** и **stash**.

1. Начните ветку урока и сохраните рабочую программу `longest`:

   ```bash
   cd ~/projects/rust
   git switch main
   git switch -c lesson/05-03-lifetimes
   ```

   Создайте `code/p05-lessons/examples/lifetimes_longest.rs` с рабочей программой `longest` из
   Раздела 2, запустите `cargo run --example lifetimes_longest`, затем сделайте коммит:

   ```bash
   git add .
   git commit -m "feat(p05): longest with a lifetime parameter"
   ```

2. Теперь **нарочно** сломайте код, ничем не рискуя. Создайте экспериментальную ветку и воспроизведите
   ошибку области видимости из раздела 2 (`result` используется вне внутреннего блока). Запустите и
   прочитайте ошибку.

   ```bash
   git switch -c experiment/break-lifetimes
   # edit the example, run it, read the error message
   git commit -am "wip: break lifetimes on purpose"
   ```

   Когда узнали всё, что хотели, выбросьте весь эксперимент целиком:

   ```bash
   git switch lesson/05-03-lifetimes
   git branch -D experiment/break-lifetimes
   ```

   ```text
   Deleted branch experiment/break-lifetimes (was 24855d9).
   ```

   `-D` (заглавная) удаляет ветку, даже если она не была слита. Используйте её только для
   экспериментов, которые хотите забыть. Ветка урока не затронута.

3. Эксперимент, который **не был закоммичен**, можно отбросить через `git restore`:

   ```bash
   echo "x" >> code/p05-lessons/src/main.rs
   git status --short
   git restore code/p05-lessons/src/main.rs
   git status --short
   ```

   Первый `git status --short` покажет ` M code/p05-lessons/src/main.rs` (изменён); после
   `git restore` второй не покажет ничего. ⚠️ `restore` нельзя отменить: изменения пропали.

4. Представьте, что вы посреди недописанного изменения и должны переключить ветку. Git отказывается
   переключаться, если это перезапишет ваши правки. **`git stash`** кладёт незакоммиченные изменения
   на полку и возвращает чистую рабочую папку:

   ```bash
   echo "// half-finished idea" >> code/p05-lessons/src/main.rs
   git stash push -m "half-finished idea"
   git status --short
   git stash list
   git stash pop
   ```

   ```text
   Saved working directory and index state On lesson/05-03-lifetimes: half-finished idea
   stash@{0}: On lesson/05-03-lifetimes: half-finished idea
   ```

   `git status --short` ничего не вывел (чисто). `git stash pop` возвращает изменения и убирает
   запись с полки; его вывод заканчивается `Dropped refs/stash@{0} (…)`. Отмените условную правку
   командой `git restore code/p05-lessons/src/main.rs`.

5. Завершите урок как раньше: закоммитьте примеры `Excerpt` и `LogLine`, слейте в `main`, удалите
   ветку урока.

## Упражнения

### Упражнение 1 — Исправьте сигнатуру (легко)

Эта функция не компилируется. Добавьте аннотации времён жизни так, чтобы она возвращала более
короткую из двух строк (первую, если длины равны). Добавьте два теста.

```rust
fn shorter(a: &str, b: &str) -> &str {
    if a.len() <= b.len() {
        a
    } else {
        b
    }
}
```

<details>
<summary>Подсказка</summary>

Результат может прийти из `a` или из `b`, поэтому обоим параметрам и результату нужно одно и то же
имя времени жизни: `<'a>`, `a: &'a str`, `b: &'a str`, `-> &'a str`.

</details>

<details>
<summary>Ответ</summary>

```rust
fn shorter<'a>(a: &'a str, b: &'a str) -> &'a str {
    if a.len() <= b.len() {
        a
    } else {
        b
    }
}

fn main() {
    println!("{}", shorter("tree", "forest"));
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn picks_the_shorter_text() {
        assert_eq!(shorter("tree", "forest"), "tree");
        assert_eq!(shorter("abc", "x"), "x");
    }

    #[test]
    fn equal_length_returns_the_first() {
        assert_eq!(shorter("ab", "cd"), "ab");
    }
}
```

```text
tree
```

Почему это работает: одно и то же `'a` на обоих входах и на выходе означает «результат действителен,
пока действительны оба входа». Раньше компилятор сообщал `E0106`, потому что правилу 2 пропуска
нужна ровно одна входная ссылка.

</details>

### Упражнение 2 — Строка настроек (средне)

Напишите `struct Setting<'a> { key: &'a str, value: &'a str }` и
`fn parse_setting(line: &str) -> Option<Setting<'_>>`, которая делит `"port = 8080"` по `=` и убирает
пробелы вокруг обеих частей. Строка без `=` даёт `None`. Выведите `Debug` и `PartialEq` и напишите
два теста.

<details>
<summary>Подсказка</summary>

`line.split_once('=')?` даёт две половины. `trim()` возвращает `&str` без пробелов в начале и в
конце; он по-прежнему указывает внутрь исходного текста, поэтому копирования нет.

</details>

<details>
<summary>Ответ</summary>

```rust
#[derive(Debug, PartialEq)]
struct Setting<'a> {
    key: &'a str,
    value: &'a str,
}

fn parse_setting(line: &str) -> Option<Setting<'_>> {
    let (key, value) = line.split_once('=')?;
    Some(Setting { key: key.trim(), value: value.trim() })
}

fn main() {
    println!("{:?}", parse_setting("port = 8080"));
    println!("{:?}", parse_setting("no equals sign"));
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn parses_key_and_value() {
        assert_eq!(
            parse_setting("port = 8080"),
            Some(Setting { key: "port", value: "8080" })
        );
    }

    #[test]
    fn line_without_equals_is_none() {
        assert_eq!(parse_setting("just text"), None);
    }
}
```

```text
Some(Setting { key: "port", value: "8080" })
None
```

Почему это работает: правило 2 пропуска привязывает возвращаемый `Setting` к единственной входной
ссылке, `line`. Все четыре среза (`key`, `value` и их обрезанные версии) указывают внутрь `line`.

</details>

### Упражнение 3 — Исправьте ошибку области видимости двумя способами (средне)

Программа ниже падает с `E0597` (это программа из раздела 2). Исправьте её **(а)** изменив только
`main` и **(б)** изменив функцию так, чтобы она возвращала владеющую `String` (внутренний блок в
`main` оставьте как есть).

```rust
fn main() {
    let first = String::from("lifetimes");
    let result;
    {
        let second = String::from("rust");
        result = longest(first.as_str(), second.as_str());
    }
    println!("the longest is {result}");
}
```

<details>
<summary>Подсказка</summary>

(а) Владелец `second` должен жить, пока используется `result`: переместите `println!` или объявление.
(б) Если функция возвращает `String`, результат ни у кого не заимствует. `x.to_string()` создаёт
владеющую копию `&str`.

</details>

<details>
<summary>Ответ</summary>

(а) Объявите обе строки в одном блоке, чтобы они жили одинаково долго (это программа из раздела 2,
которая компилируется):

```rust
fn longest<'a>(x: &'a str, y: &'a str) -> &'a str {
    if x.len() > y.len() {
        x
    } else {
        y
    }
}

fn main() {
    let first = String::from("lifetimes");
    let second = String::from("rust");
    let result = longest(first.as_str(), second.as_str());
    println!("the longest is {result}");
}
```

(б) Верните значение во владение:

```rust
fn longest_owned(x: &str, y: &str) -> String {
    if x.len() > y.len() {
        x.to_string()
    } else {
        y.to_string()
    }
}

fn main() {
    let first = String::from("lifetimes");
    let result;
    {
        let second = String::from("rust");
        result = longest_owned(&first, &second);
    }
    println!("the longest is {result}");
}
```

```text
the longest is lifetimes
```

Почему это работает: в (а) владельцы переживают использование. В (б) результат больше не указывает
на `second`, поэтому `second` можно уничтожить. Цена — одно выделение памяти на копию. Это
компромисс из раздела 6: ссылки быстрее, значения во владении проще.

</details>

### Упражнение 4 — Разбор конфигурации без копирования (сложно)

Напишите `fn parse_config(text: &str) -> Vec<(&str, &str)>`. Она читает текст построчно, пропускает
пустые строки и строки, начинающиеся с `#`, а для каждой строки с `=` добавляет в вектор пару
`(ключ, значение)` с обрезанными пробелами. Строки без `=` игнорируются. Напишите тесты: комментарии,
пустые и «сломанные» строки; пустой текст; значение, которое само содержит `=` (`url = a=b` даёт
`("url", "a=b")`).

<details>
<summary>Подсказка</summary>

`text.lines()` можно использовать в цикле `for` (Урок 05.5 объясняет это). Перекройте переменную
цикла: `let line = line.trim();`. Используйте `line.is_empty()`, `line.starts_with('#')` и
`continue`. `split_once('=')` делит только по **первому** `=`, что решает последний тест. Тип
результата не нуждается в аннотации: работает правило 2 пропуска.

</details>

<details>
<summary>Ответ</summary>

```rust
fn parse_config(text: &str) -> Vec<(&str, &str)> {
    let mut pairs = Vec::new();
    for line in text.lines() {
        let line = line.trim();
        if line.is_empty() || line.starts_with('#') {
            continue;
        }
        if let Some((key, value)) = line.split_once('=') {
            pairs.push((key.trim(), value.trim()));
        }
    }
    pairs
}

fn main() {
    let config = String::from("# server\nhost = example.org\n\nport=8080\nbroken line\n");
    for (key, value) in parse_config(&config) {
        println!("{key} -> {value}");
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn skips_comments_blank_and_broken_lines() {
        let text = "# c\nhost = a\n\nbroken\nport=1";
        assert_eq!(parse_config(text), vec![("host", "a"), ("port", "1")]);
    }

    #[test]
    fn empty_text_gives_no_pairs() {
        assert!(parse_config("").is_empty());
    }

    #[test]
    fn value_may_contain_equals() {
        assert_eq!(parse_config("url = a=b"), vec![("url", "a=b")]);
    }
}
```

```text
host -> example.org
port -> 8080
```

Почему это работает: каждый `&str` в возвращаемом векторе указывает внутрь `text`, а вектор не может
пережить `text`. Для ключей и значений не выделялось ни одной `String`, только сам вектор.

</details>

## Вопросы для самопроверки

1. Какую проблему решают времена жизни?

<details>
<summary>Ответ</summary>

Они позволяют компилятору доказать, что ни одна ссылка не переживает значение, на которое указывает, поэтому висячие ссылки и ошибки use-after-free невозможны в безопасном Rust.

</details>

2. Что обещает `fn f<'a>(x: &'a str, y: &'a str) -> &'a str`?

<details>
<summary>Ответ</summary>

Что возвращаемая ссылка действительна в течение времени жизни `'a`, то есть пересечения обоих входов: пока действительны и `x`, и `y`. Результат может указывать на любой из аргументов.

</details>

3. Делает ли запись `'a` значение более долгоживущим?

<details>
<summary>Ответ</summary>

Нет. Аннотации лишь описывают связь между уже существующими временами жизни, чтобы компилятор мог проверить ваш код. Значение живёт столько, сколько определяет область видимости его владельца.

</details>

4. Почему `fn longest(x: &str, y: &str) -> &str` нуждается в аннотациях, а `fn first_word(s: &str) -> &str` — нет?

<details>
<summary>Ответ</summary>

Правило 2 пропуска: если входная ссылка ровно одна, её время жизни отдаётся выходу. При двух входных ссылках (и без `self`) компилятор не может знать, у какой из них заимствует результат.

</details>

5. Функция строит внутри `String` и хочет вернуть `&str` на неё. Что делать?

<details>
<summary>Ответ</summary>

Вернуть саму `String` (значение во владение). Ссылка на локальную переменную стала бы висячей, когда функция закончится (`E0515`).

</details>

6. Когда разумно хранить ссылку (а значит, и время жизни) внутри структуры?

<details>
<summary>Ответ</summary>

Когда структура — недолговечное представление данных, принадлежащих кому-то другому, например результат разбора строки без копирования. Если структуре нужно жить самостоятельно или долго, храните владеющие данные (`String`).

</details>

## Итоги

- Время жизни — это то, как долго ссылку можно использовать. Компилятор проверяет, что оно никогда не превышает жизнь значения, на которое она указывает.
- Аннотации времён жизни (`'a`) описывают, как связаны времена жизни входов и выходов; они никогда не продлевают жизнь.
- Функции, возвращающие ссылки, должны сказать, у каких входов заимствует результат, если только три правила пропуска не подставят это сами.
- Структуре со ссылкой нужен параметр времени жизни, повторяемый в её блоке `impl`.
- `'static` значит «на всю программу»; оно есть у строковых литералов. Не используйте его, чтобы заглушить ошибки.
- Нельзя вернуть ссылку на локальную переменную: возвращайте значение во владение.
- Разбор без копирования возвращает срезы входных данных; результат привязан к жизни входа.
- Привычка с Git: ломайте код в одноразовой экспериментальной ветке, отбрасывайте через `git branch -D` или `git restore`, откладывайте недоделанное через `git stash`.

## Источники и что почитать

- [The Rust Programming Language — 10.3 Validating References with Lifetimes](https://doc.rust-lang.org/book/ch10-03-lifetime-syntax.html) — официальная глава с семейством примеров вокруг `longest`.
- [The Rust Reference — Lifetime elision](https://doc.rust-lang.org/reference/lifetime-elision.html) — точные правила пропуска.
- [Rust by Example — Lifetimes](https://doc.rust-lang.org/rust-by-example/scope/lifetime.html) — короткие примеры.
- [The Rustonomicon — Lifetimes](https://doc.rust-lang.org/nomicon/lifetimes.html) — более глубокий взгляд на потом (Урок 06.5).
- [Документация Git — git stash](https://git-scm.com/docs/git-stash) — откладывание незакоммиченных изменений.

---

[🏠 Начало](../START-HERE.md) · [← 05.2 Трейты](02-traits.md) · [📋 Этап 05](README.md) · [05.4 Замыкания →](04-closures.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
