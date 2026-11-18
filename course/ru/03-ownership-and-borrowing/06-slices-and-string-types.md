<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 03.5 Изменяемые ссылки и правила заимствования](05-mutable-references-and-the-borrow-rules.md) · [📋 Этап 03](README.md) · [03.7 Исправляем частые ошибки владения →](07-fixing-common-ownership-errors.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/03-ownership-and-borrowing/06-slices-and-string-types.md) <!-- nav -->

# Урок 03.6 — Срезы и строковые типы

> **Этап 03 — Владение и заимствование** · Урок 6 из 8 · ⏱ около 3 часов (2 занятия)

## Что вы узнаете

- Что такое **срез** (slice): заимствованное представление части строки или массива.
- Почему срез безопаснее, чем сохранённый индекс.
- Как хранится текст (UTF-8), почему `s[0]` не разрешён и почему `len()` считает байты.
- Чем `String` отличается от `&str` и какой из них использовать в параметрах функций.
- Как преобразовывать их друг в друга и какие полезные методы `&str` понадобятся для проекта.

## Перед началом

- Вы прошли [Урок 03.5 — Изменяемые ссылки и правила заимствования](05-mutable-references-and-the-borrow-rules.md).
- Вы знаете `String::from`, `push_str` и идею строкового литерала из [Урока 02.2 — Типы данных](../02-programming-basics/02-data-types.md).
- Создайте проект: `cargo new code/p03-slices` (из `~/projects/rust`).

## Новые слова

| Слово | Значение |
|---|---|
| **срез** (slice) | Ссылка на непрерывную часть последовательности (текста или элементов). Это указатель и длина. Записывается `&str` для текста и `&[T]` для элементов. |
| **`&str`** (строковый срез) | Заимствованное представление UTF-8 текста. Строковые литералы имеют этот тип. |
| **`String`** | Владеющий, растущий кусок UTF-8 текста, выделенный в куче. |
| **строковый литерал** (string literal) | Текст, записанный в коде между двойными кавычками, например `"hello"`. Хранится внутри файла программы и имеет тип `&str`. |
| **UTF-8** (Unicode Transformation Format, 8-bit) | Стандартный способ хранить текст в виде байтов. Каждый символ занимает от 1 до 4 байтов. |
| **Unicode** | Всемирный список символов (буквы, цифры, знаки, эмодзи) с номером у каждого. |
| **ASCII** (American Standard Code for Information Interchange, американский стандартный код обмена информацией) | Старый небольшой набор символов: английские буквы, цифры и знаки препинания. Каждый символ ASCII занимает в UTF-8 ровно 1 байт. |
| **граница символа** (char boundary) | Позиция в байтах в строке, где начинается символ. Срезы разрешены только по границам. |
| **диапазон** (range) | Два числа с `..` между ними, например `0..5`. Означает «от 0 до 5, не включая 5». |
| **приведение при разыменовании** (deref coercion) | Автоматическое преобразование, позволяющее использовать `&String` там, где ожидается `&str`. |
| **Option** | Значение, которое либо `Some(что-то)`, либо `None` («ничего»). Методы вроде `find` используют его, когда ответа может не быть. Полностью `Option` изучается в Уроке 04.4. |

## 1. Задача: индекс, который устаревает

Допустим, нам нужно первое слово текста. Самый простой вариант возвращает *позицию*, где первое слово заканчивается:

```rust
fn space_position(s: &String) -> usize {
    for (i, byte) in s.bytes().enumerate() {
        if byte == b' ' {
            return i;
        }
    }
    s.len()
}

fn main() {
    let mut text = String::from("hello world");
    let end = space_position(&text);
    text.clear(); // the text is now empty, but `end` is still 5
    println!("end = {end}");
    println!("first word: {}", &text[0..end]);
}
```

**Что происходит:**

1. `s.bytes()` выдаёт байты текста по одному. `.enumerate()` соединяет каждый байт с его позицией, так что `i` — это позиция. (Это *итераторы*; Урок 05.5.) `b' '` — значение байта пробела.
2. Функция возвращает позицию первого пробела, здесь `5`.
3. `text.clear()` очищает строку. Число `5` в `end` об этом ничего не знает.
4. `&text[0..end]` запрашивает байты с 0 по 5 пустой строки. Программа падает.

```text
end = 5
thread 'main' panicked at src/main.rs:15:37:
end byte index 5 is out of bounds for string of length 0
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
```

Компилятор не смог помочь: число `5` — просто число, никак не связанное со строкой. Такую ошибку очень легко написать. Нам хотелось бы, чтобы компилятор **сохранял связь** между «первым словом» и строкой, из которой оно взято. Именно это делают срезы.

## 2. Строковые срезы: `&str`

**Строковый срез** — это ссылка на часть `String`. Вы создаёте его с диапазоном в квадратных скобках:

```rust
fn main() {
    let s = String::from("hello world");
    println!("[0..5]  -> '{}'", &s[0..5]);
    println!("[6..11] -> '{}'", &s[6..11]);
    println!("[..5]   -> '{}'", &s[..5]);
    println!("[6..]   -> '{}'", &s[6..]);
    println!("[..]    -> '{}'", &s[..]);
    println!("[0..=4] -> '{}'", &s[0..=4]);
}
```

```text
[0..5]  -> 'hello'
[6..11] -> 'world'
[..5]   -> 'hello'
[6..]   -> 'world'
[..]    -> 'hello world'
[0..=4] -> 'hello'
```

Правила диапазонов; всё считается в **байтах**, начиная с 0:

| Диапазон | Значение |
|---|---|
| `a..b` | от `a` до `b`, **не** включая `b` |
| `a..=b` | от `a` до `b` **включительно** |
| `..b` | от начала до `b` (не включая) |
| `a..` | от `a` до конца |
| `..` | всё целиком |

Срез — это указатель на первый байт части плюс её длина. Он ничем не владеет:

```text
 s: String                     HEAP
┌──────────────┐              ┌───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┐
│ ptr ─────────┼────────────► │ h │ e │ l │ l │ o │   │ w │ o │ r │ l │ d │
│ len = 11     │              └───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┘
│ cap = 11     │                ▲                       ▲
└──────────────┘                │                       │
 &s[0..5]                       │                       │
┌──────────────┐                │                 &s[6..11]
│ ptr ─────────┼────────────────┘                ┌──────────────┐
│ len = 5      │                                 │ ptr ─────────┼──┘ (points at 'w')
└──────────────┘                                 │ len = 5      │
                                                 └──────────────┘
```

Тип строкового среза записывается **`&str`** (читается «строковый срез» или «реф-стр»). `&str` занимает 16 байтов: указатель (8) и длина (8). Ёмкости нет, потому что срез не может расти.

Теперь лучший вариант: вернуть срез вместо числа.

```rust
fn first_word(s: &str) -> &str {
    for (i, byte) in s.bytes().enumerate() {
        if byte == b' ' {
            return &s[0..i];
        }
    }
    s
}

fn main() {
    let mut text = String::from("hello world");
    let word = first_word(&text);
    println!("first word: {word}");

    text.clear(); // fine: `word` is not used any more
    println!("text is now: '{text}'");
}
```

```text
first word: hello
text is now: ''
```

(Сигнатура `fn first_word(s: &str) -> &str` говорит, что результат — это представление части входных данных. Компилятор понимает это без лишних слов, потому что входная ссылка только одна. Правило объясняет Урок 05.3.)

Срез `word` — это **заимствование** `text`. Значит, действуют правила Урока 03.5. Если попытаться использовать `word` *после* очистки строки:

```rust
fn main() {
    let mut text = String::from("hello world");
    let word = first_word(&text);
    text.clear();
    println!("first word: {word}");
}
```

```text
error[E0502]: cannot borrow `text` as mutable because it is also borrowed as immutable
  --> src/main.rs:13:5
   |
12 |     let word = first_word(&text);
   |                           ----- immutable borrow occurs here
13 |     text.clear();
   |     ^^^^^^^^^^^^ mutable borrow occurs here
14 |     println!("first word: {word}");
   |                            ---- immutable borrow later used here
```

Ошибка устаревшего индекса из раздела 1 теперь — **ошибка компиляции**. В этом весь смысл срезов: компилятор знает, что `word` зависит от `text`.

## 3. Текст — это UTF-8: байты не равны символам

Компьютеры хранят текст в виде чисел. В старой таблице ASCII было всего 128 символов (английские буквы, цифры, знаки препинания), каждый помещался в один байт. Миру нужно гораздо больше: кириллица, китайский, арабский, эмодзи. **Unicode** даёт каждому символу номер, а **UTF-8** — это способ записать эти номера в байты: английские буквы занимают 1 байт, кириллические — 2, большинство китайских иероглифов — 3, эмодзи — 4.

`String` и `&str` в Rust всегда хранят корректный UTF-8. У этого есть последствия:

```rust
fn main() {
    let words = ["hello", "привет", "日本", "🦀"];
    for w in words {
        println!("{w}: {} bytes, {} chars", w.len(), w.chars().count());
    }
}
```

```text
hello: 5 bytes, 5 chars
привет: 12 bytes, 6 chars
日本: 6 bytes, 2 chars
🦀: 4 bytes, 1 chars
```

**Что происходит:** `w.len()` считает **байты**. `w.chars()` проходит по **символам** (`char` в Rust — скалярное значение Unicode), а `.count()` их считает. Для английского текста два числа равны; для другого текста они различаются.

> ⚠️ **Частая ошибка:** считать, что `s.len()` — это число букв. Это число байтов. Для числа символов используйте `s.chars().count()`. (Но даже это не всегда «то, что человек видит как одну букву» — эмодзи с оттенком кожи состоит из двух `char`. У размера текста три смысла: байты, `char` и то, что видит глаз. Для этого этапа достаточно байтов и `char`.)

Диапазоны срезов задаются в байтах и должны попадать на **границы символов**:

```rust
fn main() {
    let s = String::from("привет");
    println!("{}", &s[0..2]);
    println!("{}", &s[0..1]);
}
```

```text
п
thread 'main' panicked at src/main.rs:4:22:
end byte index 1 is not a char boundary; it is inside 'п' (bytes 0..2 of string)
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
```

Первый срез, байты с 0 по 2, — это ровно буква `п` (2 байта). Второй разрезает букву пополам, и программа во время выполнения **паникует** (останавливается с сообщением об ошибке). Поэтому Rust не позволяет индексировать строку одним числом:

```rust
fn main() {
    let s = String::from("hello");
    let first = s[0];
    println!("{first}");
}
```

```text
error[E0277]: the type `str` cannot be indexed by `{integer}`
 --> src/main.rs:3:19
  |
3 |     let first = s[0];
  |                   ^ string indices are ranges of `usize`
  |
  = help: the trait `SliceIndex<str>` is not implemented for `{integer}`
  = note: you can use `.chars().nth()` or `.bytes().nth()`
          for more information, see chapter 8 in The Book: <https://doc.rust-lang.org/book/ch08-02-strings.html#indexing-into-strings>
```

(Настоящее сообщение продолжается ещё несколькими техническими замечаниями; ключевые строки выше.) Если бы `s[0]` было разрешено, это был бы первый байт или первый символ? Rust отказывается гадать. Используйте `s.chars().nth(0)` для первого символа или диапазон среза, когда знаете границы. Урок 04.8 подробнее разбирает строки.

> 🔒 **Безопасность:** текст, пришедший извне (файлы, сеть, пользователи), может быть повреждённым или враждебным. Срезы по вычисленным позициям могут вызвать панику и остановить вашу программу — возможный *отказ в обслуживании*. Когда позиции берутся из входных данных, предпочитайте методы вроде `find`, `split_whitespace` или `chars`, которые сами учитывают границы.

## 4. `String` и `&str`: два типа, одна идея

| | `String` | `&str` |
|---|---|---|
| Владеет текстом? | **да** | нет, заимствует |
| Где текст? | куча (растущий) | где угодно: куча, стек или файл программы |
| Может расти или меняться? | да (если `mut`) | нет, представление только для чтения |
| Размер на стеке | 24 байта (ptr, len, cap) | 16 байтов (ptr, len) |
| Создаётся из | `String::from("x")`, `"x".to_string()`, `format!(...)` | `"literal"`, `&some_string`, `&s[a..b]` |

**Строковый литерал** вроде `"hello"` — это `&str`. Текст лежит внутри вашей скомпилированной программы (только для чтения), а переменная хранит указатель на него и длину. Это объясняет, почему литерал не может расти: вы им не владеете.

Распространённые способы создавать и объединять строки:

```rust
fn main() {
    let a = String::from("hello");
    let b = "world".to_string();
    let c = "!".to_owned();

    let mut s = String::new();
    s.push_str(&a);
    s.push(' ');
    s.push_str(&b);
    s.push_str(&c);
    println!("{s}");

    let t = format!("{a}-{b}-{c}");
    println!("{t}");
    println!("{a} {b} {c}");
}
```

```text
hello world!
hello-world-!
hello world !
```

**Что происходит:** `String::from`, `.to_string()` и `.to_owned()` превращают `&str` в владеющую `String`. `push_str` дописывает `&str` (поэтому мы передаём `&a`, то есть `&String`, который Rust преобразует — см. раздел 5). `push` дописывает один `char`. `format!` работает как `println!`, но возвращает новую `String` и лишь заимствует свои аргументы (последняя строка показывает, что `a`, `b`, `c` всё ещё наши).

У оператора `+` для строк есть сюрприз: он **перемещает** свою левую часть.

```rust
fn main() {
    let s1 = String::from("hello, ");
    let s2 = String::from("world");
    let s3 = s1 + &s2;
    println!("{s3}");
    println!("{s1}");
}
```

```text
error[E0382]: borrow of moved value: `s1`
 --> src/main.rs:6:16
  |
2 |     let s1 = String::from("hello, ");
  |         -- move occurs because `s1` has type `String`, which does not implement the `Copy` trait
3 |     let s2 = String::from("world");
4 |     let s3 = s1 + &s2;
  |              -- value moved here
5 |     println!("{s3}");
6 |     println!("{s1}");
  |                ^^ value borrowed here after move
```

`s1 + &s2` забирает владение `s1`, дописывает к ней текст `s2` (используя буфер `s1`, так что в лучшем случае новое выделение не нужно) и возвращает результат как `s3`. Правая часть должна быть `&str`: передача самой `s2` — ошибка типа `expected &str, found String` (`E0308`), которая исправляется записью `&s2`. Исправленная программа сохраняет `s2` и теряет `s1`:

```rust
fn main() {
    let s1 = String::from("hello, ");
    let s2 = String::from("world");
    let s3 = s1 + &s2; // s1 is moved into s3, s2 is only borrowed
    println!("{s3}");
    println!("{s2}");
}
```

```text
hello, world
world
```

Для более чем двух кусков `format!` понятнее цепочки из `+`.

`&str` нельзя изменить, потому что это представление только для чтения:

```rust
fn main() {
    let s: &str = "hello";
    s.push_str(" world");
    println!("{s}");
}
```

```text
error[E0599]: no method named `push_str` found for reference `&str` in the current scope
 --> src/main.rs:3:7
  |
3 |     s.push_str(" world");
  |       ^^^^^^^^ method not found in `&str`
```

Если нужно изменять текст, нужна `String`.

## 5. Используйте `&str` в параметрах функций

Это самое важное соглашение урока:

> 📏 **Соглашение:** когда функция только читает текст, принимайте **`&str`**, а не `&String`.

Почему? Параметр `&str` принимает *всё*: литерал, целую `String` и часть строки.

```rust
fn shout(text: &str) -> String {
    text.to_uppercase()
}

fn main() {
    let owned = String::from("owned string");
    println!("{}", shout(&owned));          // &String becomes &str
    println!("{}", shout("a literal"));     // a literal is already &str
    println!("{}", shout(&owned[0..5]));    // a slice is &str too
}
```

```text
OWNED STRING
A LITERAL
OWNED
```

**Что происходит:** компилятор незаметно превращает `&String` в `&str` там, где ожидается `&str`. Это **приведение при разыменовании** (deref coercion). Функция с параметром `&String` отказалась бы принять литерал и срез, поэтому была бы менее полезной. `cargo clippy` даже предупреждает о параметрах `&String` (вы увидите это в Уроке 03.7). То же правило действует для векторов: принимайте `&[T]` вместо `&Vec<T>`.

## 6. Срезы других вещей: `&[T]`

Срезы бывают не только у текста. Любую непрерывную последовательность элементов — массив или `Vec` — можно срезать. Тип — `&[T]`, «срез из `T`».

```rust
fn sum(numbers: &[i32]) -> i32 {
    let mut total = 0;
    for n in numbers {
        total += n;
    }
    total
}

fn main() {
    let array = [1, 2, 3, 4, 5];
    let vector = vec![10, 20, 30];

    println!("whole array:  {}", sum(&array));
    println!("array [1..4]: {}", sum(&array[1..4]));
    println!("vector:       {}", sum(&vector));
    println!("vector [..2]: {}", sum(&vector[..2]));
    println!("size of &[i32]: {} bytes", size_of::<&[i32]>());
    println!("size of &str:   {} bytes", size_of::<&str>());
}
```

```text
whole array:  15
array [1..4]: 9
vector:       60
vector [..2]: 30
size of &[i32]: 16 bytes
size of &str:   16 bytes
```

Одна функция теперь работает для массивов, векторов и частей и тех и других. `&str` — по сути та же идея, что `&[u8]`, с обещанием, что байты — корректный UTF-8.

Есть и изменяемый срез `&mut [T]`, позволяющий функции менять элементы (но не длину):

```rust
fn zero_first_two(numbers: &mut [i32]) {
    numbers[0] = 0;
    numbers[1] = 0;
}

fn main() {
    let mut data = [9, 8, 7, 6];
    zero_first_two(&mut data);
    println!("{:?}", data);
    zero_first_two(&mut data[2..]);
    println!("{:?}", data);
}
```

```text
[0, 0, 7, 6]
[0, 0, 0, 0]
```

Второй вызов передаёт только часть `data[2..]`. Функция видит срез длиной 2 и меняет его первые два элемента — на самом деле это `data[2]` и `data[3]`. Правила заимствования из Урока 03.5 действуют и для срезов.

## 7. Удобные методы `&str`

Вы будете пользоваться ими постоянно (в том числе в проекте). Все они работают и для `&str`, и для `String`:

```rust
fn main() {
    let line = "  Rust is fast, safe and fun  ";
    println!("trim:      '{}'", line.trim());
    println!("words:     {}", line.split_whitespace().count());
    println!("starts:    {}", line.trim().starts_with("Rust"));
    println!("contains:  {}", line.contains("safe"));
    println!("find ',':  {:?}", line.find(','));
    println!("find 'z':  {:?}", line.find('z'));
    println!("lower:     '{}'", line.trim().to_lowercase());
    println!("replace:   '{}'", line.trim().replace("fast", "quick"));

    let poem = "one\ntwo\nthree";
    for l in poem.lines() {
        println!("line: {l}");
    }
    for w in "to be or not".split_whitespace() {
        println!("word: {w}");
    }
}
```

```text
trim:      'Rust is fast, safe and fun'
words:     6
starts:    true
contains:  true
find ',':  Some(14)
find 'z':  None
lower:     'rust is fast, safe and fun'
replace:   'Rust is quick, safe and fun'
line: one
line: two
line: three
word: to
word: be
word: or
word: not
```

| Метод | Что делает | Возвращает |
|---|---|---|
| `trim()` | убирает пробелы и переводы строк с обоих концов | `&str` (срез оригинала!) |
| `split_whitespace()` | проходит по словам, разделённым любыми пробельными символами | то, что можно использовать в `for` |
| `lines()` | проходит по строкам | то, что можно использовать в `for` |
| `chars()` | проходит по символам | то, что можно использовать в `for` |
| `starts_with(x)`, `contains(x)` | проверки | `bool` |
| `find(x)` | позиция (в байтах) первого совпадения | `Option<usize>`: `Some(14)` или `None` |
| `to_lowercase()`, `to_uppercase()`, `replace(a, b)` | строят изменённый текст | новая `String` |
| `len()`, `is_empty()` | байты и «нет байтов» | `usize`, `bool` |

Важная деталь: методы, которые только *смотрят* (`trim`, `split_whitespace`, `lines`), возвращают срезы оригинального текста. Они дёшевы: без выделения памяти. Методы, которые *меняют* текст (`to_lowercase`, `replace`), обязаны построить новую `String`.

`find` возвращает **`Option`**, потому что совпадения может не быть. `Some(14)` означает «найдено на байте 14», а `None` — «не найдено». `Option` и `match` полностью изучаются в Уроке 04.4.

## 8. Несколько родственных типов

- **`char`** — один символ Unicode, записывается в одинарных кавычках: `'a'`, `'я'`, `'🦀'`. Занимает 4 байта.
- **`&[u8]`** — сырые байты без обещаний о тексте. Файлы и сети дают байты; вы преобразуете их в текст (проверяя, что это корректный UTF-8).
- **`Path` / `OsStr`** — имена файлов в настоящей операционной системе *не всегда* корректный UTF-8, поэтому в стандартной библиотеке для них есть специальные типы. Вы встретите их на Этапе 11.

## Частые ошибки

**1. Использовать `s.len()` как число букв.** Он считает байты.

**2. Резать по вычисленной позиции внутри символа.** Программа паникует. Используйте `char_indices()` или `find`, чтобы получить корректные позиции.

**3. Писать `fn f(s: &String)` или `fn f(v: &Vec<T>)`.** Используйте `&str` и `&[T]`: гибче, цена та же.

**4. Писать `s1 + s2` вместо `s1 + &s2`.** Ошибка `E0308`: ожидалось `&str`, найдено `String`.

**5. Забывать, что `s1 + &s2` потребляет `s1`.** Используйте `format!` или `s1.clone() + &s2`.

**6. Пытаться вызвать `push_str` у `&str`.** Ошибка `E0599`. Сделайте `String` (`to_string()`).

**7. Держать срез и затем менять строку.** Ошибка `E0502`. Так компилятор защищает вас от устаревшего представления.

## Попробуйте сами

1. В `~/projects/rust` выполните `cargo new code/p03-slices` (если вы ещё не сделали этого в разделе «Перед началом»). Напишите программу `first_word`. Запустите её.
2. Перенесите `text.clear();` непосредственно *перед* первым `println!`, который использует `word`. Прочтите ошибку; найдите три подписи (immutable borrow, mutable borrow, «later used here»).
3. Выведите `len()` и `chars().count()` для `"café"`, `"naïve"` и `"Привет, мир"`. Сначала предскажите.
4. Напишите `fn shout(text: &str) -> String` и вызовите её с литералом, с `&String` и со срезом.
5. Сделайте коммит: `git add code/p03-slices && git commit -m "p03: slices and strings"`.

## Упражнения

### Упражнение 1 — Предскажите срезы (легко)

Что выведет каждая строка?

```rust
fn main() {
    let s = "systems";
    println!("1: '{}'", &s[..3]);
    println!("2: '{}'", &s[3..]);
    println!("3: '{}'", &s[1..=2]);
    println!("4: '{}'", &s[2..2]);
    println!("5: '{}'", &s[..]);
}
```

<details>
<summary>Подсказка</summary>

Буквы нумеруются с 0: `s`=0, `y`=1, `s`=2, `t`=3, `e`=4, `m`=5, `s`=6. Конец `a..b` не включается; конец `a..=b` включается.

</details>

<details>
<summary>Ответ</summary>

```text
1: 'sys'
2: 'tems'
3: 'ys'
4: ''
5: 'systems'
```

Почему это работает: `..3` — это байты 0, 1, 2. `3..` — байты с 3 до конца. `1..=2` — байты 1 и 2. `2..2` — пустой диапазон, а пустой срез допустим. `..` — всё целиком.

</details>

### Упражнение 2 — Байты или символы? (легко)

Предскажите `len()` и `chars().count()` для каждой строки, затем запустите программу и проверьте:

```rust
fn main() {
    for text in ["abc", "héllo", "日本語", "👍🏽", ""] {
        println!("{:?}: {} bytes, {} chars", text, text.len(), text.chars().count());
    }
}
```

<details>
<summary>Подсказка</summary>

Буквы ASCII занимают 1 байт, `é` — 2, каждый из этих иероглифов — 3, а эмодзи — 4. «Большой палец вверх» с оттенком кожи на самом деле состоит из двух символов.

</details>

<details>
<summary>Ответ</summary>

```text
"abc": 3 bytes, 3 chars
"héllo": 6 bytes, 5 chars
"日本語": 9 bytes, 3 chars
"👍🏽": 8 bytes, 2 chars
"": 0 bytes, 0 chars
```

Почему это работает: `{:?}` выводит строку в кавычках. `héllo` — это 4 буквы ASCII (4 байта) плюс `é` (2 байта) = 6. Эмодзи «большой палец вверх» (4 байта) сопровождается символом-модификатором оттенка кожи (4 байта): 2 символа, 8 байтов, но одна картинка на экране.

</details>

### Упражнение 3 — Расширение файла, с тестами (средне)

Напишите `fn file_extension(name: &str) -> &str`, которая возвращает часть после **последней** точки или пустую строку, если точки нет. Напишите тесты: `"photo.png"` → `"png"`, `"archive.tar.gz"` → `"gz"`, `"README"` → `""` и `"odd."` → `""`.

<details>
<summary>Подсказка</summary>

`name.rfind('.')` ищет справа и возвращает `Option<usize>`. Используйте `match` с двумя ветками: `Some(position) => &name[position + 1..]` и `None => ""`. (`match` по `Option` разбирается в Уроке 04.4; форма кода здесь дана.)

</details>

<details>
<summary>Ответ</summary>

```rust
fn file_extension(name: &str) -> &str {
    match name.rfind('.') {
        Some(position) => &name[position + 1..],
        None => "",
    }
}

fn main() {
    for name in ["photo.png", "archive.tar.gz", "README"] {
        println!("{name} -> '{}'", file_extension(name));
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn simple_extension() {
        assert_eq!(file_extension("photo.png"), "png");
    }

    #[test]
    fn last_dot_wins() {
        assert_eq!(file_extension("archive.tar.gz"), "gz");
    }

    #[test]
    fn no_dot_means_empty() {
        assert_eq!(file_extension("README"), "");
    }

    #[test]
    fn trailing_dot_means_empty() {
        assert_eq!(file_extension("odd."), "");
    }
}
```

```text
photo.png -> 'png'
archive.tar.gz -> 'gz'
README -> ''
```

Почему это работает: результат — срез `name`, поэтому новый текст не выделяется. Точка — один байт (ASCII), поэтому `position + 1` всегда корректная граница. Для `"odd."` срез `[4..]` пуст.

</details>

### Упражнение 4 — Склейка через дефис (средне)

Напишите `fn join_with_dash(a: &str, b: &str) -> String`, возвращающую `a-b`. Вызовите её с заимствованной `String` и литералом и покажите, что `String` всё ещё пригодна. Добавьте два теста.

<details>
<summary>Подсказка</summary>

`format!("{a}-{b}")` строит новую `String`. В `main` передайте `&first` для `String`.

</details>

<details>
<summary>Ответ</summary>

```rust
fn join_with_dash(a: &str, b: &str) -> String {
    format!("{a}-{b}")
}

fn main() {
    let first = String::from("rust");
    let joined = join_with_dash(&first, "lang");
    println!("{joined}");
    println!("{first} is still mine");
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn joins_two_parts() {
        assert_eq!(join_with_dash("a", "b"), "a-b");
    }

    #[test]
    fn works_with_string_borrows() {
        let x = String::from("left");
        let y = String::from("right");
        assert_eq!(join_with_dash(&x, &y), "left-right");
    }
}
```

```text
rust-lang
rust is still mine
```

Почему это работает: оба параметра — `&str`, поэтому функция принимает литералы и заимствованные `String`. `format!` лишь заимствует, так что ничего не перемещается.

</details>

### Упражнение 5 — Второе слово (сложно)

Напишите `fn second_word(text: &str) -> &str`, которая возвращает второе слово текста (слова разделены любыми пробельными символами; лишние пробелы не важны) или `""`, если слов меньше двух. Результат должен быть срезом входных данных. Добавьте тесты для обычного текста, лишних пробелов и слишком малого числа слов. Затем в `main` сохраните результат `second_word(&line)` в переменной и попробуйте вызвать `line.clear()` до её использования — прочтите ошибку.

<details>
<summary>Подсказка</summary>

Обходите `text.split_whitespace()` со счётчиком. Когда счётчик достигает 2, `return word;`. После цикла верните `""`. Каждое `word` уже является срезом `text`.

</details>

<details>
<summary>Ответ</summary>

```rust
fn second_word(text: &str) -> &str {
    let mut position = 0;
    for word in text.split_whitespace() {
        position += 1;
        if position == 2 {
            return word;
        }
    }
    ""
}

fn main() {
    println!("'{}'", second_word("one two three"));
    println!("'{}'", second_word("alone"));
    let line = String::from("  borrowed   slices  ");
    let word = second_word(&line);
    println!("'{word}' (borrowed from line: {line:?})");
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn finds_second_word() {
        assert_eq!(second_word("one two three"), "two");
    }

    #[test]
    fn extra_spaces_do_not_matter() {
        assert_eq!(second_word("  a    b  "), "b");
    }

    #[test]
    fn too_few_words_gives_empty() {
        assert_eq!(second_word(""), "");
        assert_eq!(second_word("single"), "");
    }
}
```

```text
'two'
''
'slices' (borrowed from line: "  borrowed   slices  ")
```

Если вызвать `line.clear()`, пока `word` ещё используется позже, компилятор сообщит `E0502` (изменяемое заимствование при живом неизменяемом).

Почему это работает: `split_whitespace` пропускает лишние пробелы и выдаёт срезы оригинального текста. Пустая строка `""` — литерал `&'static str`, и его тоже можно вернуть, потому что он живёт столько же, сколько программа (почему это разрешено, объясняет Урок 05.3).

</details>

## Вопросы для самопроверки

1. Что такое строковый срез и какого размера `&str` на 64-битной машине?

<details>
<summary>Ответ</summary>

Заимствованное представление части UTF-8 текста, состоящее из указателя и длины. Он занимает 16 байтов.

</details>

2. Почему возврат среза безопаснее возврата индекса?

<details>
<summary>Ответ</summary>

Срез — это заимствование исходной строки, поэтому компилятор не даст изменить или уничтожить строку, пока срез используется. Индекс — просто число, и он может незаметно устареть.

</details>

3. Почему `"привет".len()` возвращает 12?

<details>
<summary>Ответ</summary>

`len()` считает байты. Каждая кириллическая буква занимает в UTF-8 2 байта, а букв 6.

</details>

4. Почему нельзя написать `s[0]` для `String`?

<details>
<summary>Ответ</summary>

У символов в UTF-8 разная длина в байтах, поэтому одно число неоднозначно (байт или символ?), а позиция в байтах может попасть внутрь символа. Rust это запрещает; используйте `chars()`, `bytes()` или диапазон.

</details>

5. Какой тип параметра должна использовать функция, которая только читает текст, и почему?

<details>
<summary>Ответ</summary>

`&str`. Он принимает строковые литералы, `&String` (через приведение при разыменовании) и срезы.

</details>

6. Что делает `s1 + &s2` с `s1` и `s2`?

<details>
<summary>Ответ</summary>

Перемещает `s1` (потом её нельзя использовать) и лишь заимствует `s2`. Результат — новый владелец объединённого текста.

</details>

7. Какого типа строковый литерал `"hello"`?

<details>
<summary>Ответ</summary>

`&str` — заимствованный срез текста, хранящегося внутри программы.

</details>

## Итоги

- Срез (`&str`, `&[T]`) — заимствованное представление непрерывной части строки или последовательности: указатель плюс длина. Он ничем не владеет, и к нему применяются правила заимствования.
- Возврат среза вместо индекса позволяет компилятору предотвращать ошибки устаревшего индекса.
- Строки — это UTF-8: `len()` считает байты, `chars().count()` считает символы, диапазоны срезов задаются в байтах и должны лежать на границах символов, а `s[0]` не разрешён.
- `String` владеет и может расти; `&str` заимствует и доступен только для чтения. Литералы — `&str`.
- Принимайте в параметрах `&str` (не `&String`) и `&[T]` (не `&Vec<T>`): они принимают больше.
- `s1 + &s2` перемещает `s1`; используйте `format!`, чтобы объединять, ничего не перемещая.
- `trim`, `split_whitespace`, `lines` возвращают дешёвые срезы; `to_lowercase` и `replace` строят новые `String`; `find` возвращает `Option`.

## Источники и что почитать

- [The Rust Programming Language — 4.3 The Slice Type](https://doc.rust-lang.org/book/ch04-03-slices.html) — официальная глава о срезах.
- [The Rust Programming Language — 8.2 Storing UTF-8 Encoded Text with Strings](https://doc.rust-lang.org/book/ch08-02-strings.html) — почему индексация запрещена, байты против символов.
- [std::primitive::str](https://doc.rust-lang.org/std/primitive.str.html) — все методы `&str`.
- [std::string::String](https://doc.rust-lang.org/std/string/struct.String.html) — все методы `String`.
- [std::primitive::slice](https://doc.rust-lang.org/std/primitive.slice.html) — все методы `&[T]`.

---

[🏠 Начало](../START-HERE.md) · [← 03.5 Изменяемые ссылки и правила заимствования](05-mutable-references-and-the-borrow-rules.md) · [📋 Этап 03](README.md) · [03.7 Исправляем частые ошибки владения →](07-fixing-common-ownership-errors.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
