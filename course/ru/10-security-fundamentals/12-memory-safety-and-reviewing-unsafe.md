<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 10.11 TLS и сертификаты](11-tls-and-certificates.md) · [📋 Этап 10](README.md) · [10.13 Безопасность цепочки поставок: cargo-audit и cargo-deny →](13-supply-chain-security.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/10-security-fundamentals/12-memory-safety-and-reviewing-unsafe.md) <!-- nav -->

# Урок 10.12 — Безопасность памяти и проверка небезопасного кода

> **Этап 10 — Основы безопасности** · Урок 12 из 15 · ⏱ около 135 минут

## Что вы узнаете

- Какие ошибки памяти исключает безопасный Rust и какие проблемы безопасности он **не** предотвращает.
- Как паника, огромный заявленный размер или глубоко вложенный ввод превращают безопасный код в дыру для отказа в обслуживании и как исправить каждый случай.
- Как непроверенное целочисленное вычисление превращается в неверный размер и как `checked_*` это исправляет.
- Как проверять `unsafe`-код по чек-листу и как ловить неопределённое поведение с помощью Miri и AddressSanitizer.
- Какие lint-правила Clippy помогают рано находить рискованный код.

## Перед началом

- Вы прошли [Урок 10.11 — TLS и сертификаты](11-tls-and-certificates.md).
- Вы знаете владение, стек и кучу ([Урок 03.1](../03-ownership-and-borrowing/01-memory-stack-and-heap.md)), переполнение целых чисел ([Урок 02.3](../02-programming-basics/03-operators-and-expressions.md)) и обработку ошибок ([Урок 04.10](../04-structs-enums-collections-errors/10-error-handling.md)).
- Вы знаете, что значит `unsafe`, комментарии `// SAFETY:`, неопределённое поведение и Miri ([Урок 06.6](../06-rust-advanced/06-unsafe-rust-and-miri.md)). Этот урок не учит `unsafe` заново: он показывает, как его **проверять**.
- Один раз установите nightly-тулчейн с Miri (stable остаётся основным), точно как в Уроке 06.6:

```bash
rustup toolchain install nightly --component miri
```

- Создайте учебный пакет. Примеры лежат в `src/bin/`:

```bash
cd ~/projects/rust
cargo new code/p10-memory
cd code/p10-memory
mkdir src/bin
```

Добавьте эти строки в конец `Cargo.toml`. Они просят Clippy (линтер Rust) предупреждать о коде, который может паниковать или переполняться, и о блоках `unsafe` без комментария:

```toml
[lints.clippy]
indexing_slicing = "warn"
unwrap_used = "warn"
arithmetic_side_effects = "warn"
undocumented_unsafe_blocks = "warn"
```

> 🔒 **Безопасность и этика:** все «атаки» здесь — это входные данные, которые вы пишете сами, против ваших собственных небольших программ.

## Новые слова

| Слово | Значение |
|---|---|
| **безопасность памяти** (memory safety) | Программа читает и пишет только ту память, которой владеет и которая ещё действительна. Безопасный Rust это гарантирует. |
| **переполнение буфера** (buffer overflow) | Чтение или запись за концом массива или буфера. |
| **использование после освобождения** (use after free) | Использование памяти после её освобождения. |
| **DoS** (Denial of Service, отказ в обслуживании) | Атака, из-за которой сервис становится недоступным: он падает, зависает или съедает всю память. |
| **исчерпание ресурсов** (resource exhaustion) | DoS, при котором атакующий заставляет вашу программу израсходовать память, процессорное время (CPU, Central Processing Unit, центральный процессор) или диск. |
| **корректный / некорректный** (sound / unsound) | Безопасная функция *корректна* (sound), если **никакой** ввод не может вызвать неопределённое поведение. Если какой-то ввод может, она *некорректна* (unsound), и это ошибка, даже если ваш код сам никогда не передаёт такой ввод. |
| **санитайзер** (sanitizer) | Встроенный в компилятор инструмент, который добавляет в программу проверки во время работы и сообщает об ошибках памяти. |
| **ASan** (AddressSanitizer) | Санитайзер, который находит выходы за границы и использование после освобождения. |
| **переполнение стека** (stack overflow) | Стек вызовов исчерпал место (например, при бесконечной или очень глубокой рекурсии). Rust аварийно завершает процесс. |
| **MiB** (mebibyte, мебибайт) | 1 048 576 байт (1024 × 1024). *GiB* — это 1024 MiB. |

## 1. Что предотвращает безопасный Rust и что остаётся

За многими серьёзными уязвимостями стоят ошибки памяти: переполнения буфера, использование после освобождения, двойное освобождение, чтение неинициализированной памяти и гонки данных.
Безопасный Rust делает их **невозможными**: компилятор проверяет владение, заимствование и времена жизни (Этап 03), а обращения к массивам проверяются на границы во время работы.

Это огромный плюс, но не гарантия «уязвимостей нет»:

| Проблема | Предотвращает ли безопасный Rust? | Что делаете вы |
|---|---|---|
| Переполнение буфера, использование после освобождения, гонка данных | **Да** | Оставайтесь в безопасном коде. |
| Паника на плохом вводе (индекс, `unwrap`, деление на ноль) | Нет: программа *останавливается* | Возвращайте ошибки для недоверенного ввода (раздел 2). |
| Исчерпание памяти или процессора огромными размерами | Нет | Ограничивайте каждый размер, пришедший снаружи (раздел 3). |
| Переполнение стека из-за глубокой рекурсии | Нет: процесс аварийно завершается | Ограничивайте глубину (раздел 4). |
| Целочисленное переполнение, дающее неверный размер | Нет (в релизных сборках значение «заворачивается») | Арифметика `checked_*` (раздел 3). |
| Логические ошибки: отсутствие авторизации, SQL-внедрение (SQL, Structured Query Language, язык запросов к базам данных), слабая криптография | Нет | Остальная часть этого этапа. |
| Ошибки внутри блоков `unsafe`, в вашем коде или в зависимости | Нет | Проверяйте и тестируйте их (раздел 5). |

## 2. Паника — это отказ в обслуживании

Паника завершает поток; если это случилось в главном потоке или в сервере, который её не перехватывает, останавливается вся программа. Если **недоверенный ввод может вызвать панику**, любой может уронить ваш сервис одним запросом. Создайте `src/bin/panic_dos.rs`:

```rust
/// Reads a 2-byte big-endian length from the start of a message.
fn read_len(message: &[u8]) -> u16 {
    u16::from_be_bytes([message[0], message[1]])
}

fn main() {
    println!("{}", read_len(&[0x01, 0x02, 0xff]));
    println!("{}", read_len(&[0x01]));
    println!("never reached");
}
```

```text
258

thread 'main' (4767495) panicked at src/bin/panic_dos.rs:3:37:
index out of bounds: the len is 1 but the index is 1
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
```

**Что происходит:**

1. `message[0]` и `message[1]` проверяются на границы: поэтому это *безопасно для памяти* (выхода за границы не происходит).
2. Но проверка **паникует**, когда в сообщении всего один байт. Программа останавливается до `never reached`. Сетевой сервис, читающий это из сокета, упал бы от однобайтового пакета.

(Номер потока в сообщении на вашем компьютере будет другим.) Запустите `cargo clippy`, и lint `indexing_slicing` укажет на оба места:

```text
warning: indexing may panic
 --> src/bin/panic_dos.rs:3:25
  |
3 |     u16::from_be_bytes([message[0], message[1]])
  |                         ^^^^^^^^^^
  |
  = help: consider using `.get(n)` or `.get_mut(n)` instead
```

Исправление: считать короткий ввод обычной, ожидаемой ситуацией и вернуть `Option` (или `Result` с типом ошибки, Урок 04.10). Создайте `src/bin/panic_fixed.rs`:

```rust
/// Reads a 2-byte big-endian length from the start of a message.
/// Short input is an expected situation, so the answer is an `Option`, not a panic.
fn read_len(message: &[u8]) -> Option<u16> {
    let [a, b, ..] = message else {
        return None;
    };
    Some(u16::from_be_bytes([*a, *b]))
}

fn main() {
    println!("{:?}", read_len(&[0x01, 0x02, 0xff]));
    println!("{:?}", read_len(&[0x01]));
    println!("still running");
}
```

```text
Some(258)
None
still running
```

**Что происходит:** образец `[a, b, ..]` подходит срезу, в котором **не менее** двух элементов, и привязывает первые два; `let … else` выполняет блок `else`, когда образец не подошёл. Индексации нет, значит нет и паники, и Clippy молчит.

> 📏 **Соглашение:** `unwrap`, `expect`, `[i]` и `panic!` допустимы для *ваших собственных* инвариантов («здесь не может быть `None`, я только что проверил») и в тестах. Они недопустимы для всего, что зависит от ввода из файла, сети, от пользователя или другой программы.

> 🧪 **Тестирование:** тестируйте парсеры на **коротких, пустых и абсурдных** входах. Хороший тест вызывает функцию со всеми длинами от 0 до нескольких байтов и проверяет, что она никогда не паникует (Упражнение 2).

## 3. Недоверенные размеры

### Целочисленное переполнение даёт неверные размеры

Урок 02.3 показал, что арифметическое переполнение паникует в отладочных сборках и «заворачивается» в релизных. Для размера, взятого из ввода, это становится уязвимостью. Создайте `src/bin/overflow.rs`:

```rust
/// Total bytes needed for `count` records of `size` bytes each.
fn total_bytes(count: u32, size: u32) -> u32 {
    count * size
}

fn main() {
    println!("{}", total_bytes(1000, 4096));
    // An attacker announces 2 million records of 4096 bytes: the true total is 8,192,000,000.
    let total = total_bytes(2_000_000, 4096);
    println!("allocating {total} bytes");
}
```

Сравните `cargo run --bin overflow` (debug) с `cargo run --release --bin overflow`:

```text
4096000

thread 'main' (4784981) panicked at src/bin/overflow.rs:3:5:
attempt to multiply with overflow
```

```text
4096000
allocating 3897032704 bytes
```

**Что происходит:** в отладочной сборке переполнение вызывает панику (это DoS, но хотя бы его замечают). В релизной сборке результат молча «заворачивается»: 8 192 000 000 не помещается в `u32` (максимум 4 294 967 295), и программа продолжает работу с числом 3 897 032 704, которое неверно на 4,3 миллиарда. Программа, которая выделит `total` байт, а затем скопирует `count × size` байт, запишет далеко за пределы своего буфера
(в безопасном Rust проверка границ остановит это; в `unsafe`-коде это было бы классическим переполнением кучи). Исправление: `checked_mul` возвращает `None` при переполнении. Создайте `src/bin/overflow_fixed.rs`:

```rust
/// Total bytes needed, or `None` if the multiplication does not fit.
fn total_bytes(count: u32, size: u32) -> Option<u32> {
    count.checked_mul(size)
}

fn main() {
    println!("{:?}", total_bytes(1000, 4096));
    println!("{:?}", total_bytes(2_000_000, 4096));
}
```

```text
Some(4096000)
None
```

> 📏 **Соглашение:** для арифметики над значениями, пришедшими снаружи, используйте `checked_add`, `checked_mul`, `checked_sub` (или `saturating_*`, когда ограничение значением верно) и преобразуйте через `try_from`, а не `as` (`as` молча отбрасывает старшие биты). Lint `arithmetic_side_effects` находит обычные `+`, `-` и `*`, на которые стоит посмотреть ещё раз.

### Никогда не доверяйте длине

Очень распространённый формат — «длина, затем данные»: первые 4 байта говорят, сколько байтов идёт дальше. Если создать буфер заявленного размера **до** того, как пришли данные, атакующему хватит 4 байтов, чтобы ваша программа запросила 4 GiB (гибибайта). Делайте так:

1. Задайте **максимум**, разумный для вашего протокола, и отвергайте большие значения.
2. Выделяйте память через `try_reserve`, чтобы неудача была ошибкой, а не аварийным завершением.
3. Читайте через `take(len)`, чтобы никогда не прочитать больше заявленного.

Создайте `src/bin/frame_alloc.rs`:

```rust
use std::io::{self, Cursor, Read};

const MAX_FRAME: usize = 1024 * 1024; // 1 MiB

/// Reads one frame: a 4-byte big-endian length, then that many bytes.
fn read_frame(input: &mut impl Read) -> io::Result<Vec<u8>> {
    let mut len_bytes = [0u8; 4];
    input.read_exact(&mut len_bytes)?;
    let len = u32::from_be_bytes(len_bytes) as usize;

    if len > MAX_FRAME {
        return Err(io::Error::new(io::ErrorKind::InvalidData, format!("frame of {len} bytes is too large")));
    }
    let mut frame = Vec::new();
    frame.try_reserve_exact(len).map_err(|_| io::Error::new(io::ErrorKind::OutOfMemory, "cannot allocate"))?;
    // `take` makes sure we never read more than `len` bytes, even if the input is endless.
    input.take(len as u64).read_to_end(&mut frame)?;
    if frame.len() != len {
        return Err(io::Error::new(io::ErrorKind::UnexpectedEof, "frame is shorter than announced"));
    }
    Ok(frame)
}

fn main() {
    // A good frame: length 5, then "hello".
    let mut good = Cursor::new(vec![0, 0, 0, 5, b'h', b'e', b'l', b'l', b'o']);
    println!("good : {:?}", read_frame(&mut good));

    // A hostile frame: only 4 bytes of input, but it announces 4 GiB.
    let mut evil = Cursor::new(vec![0xff, 0xff, 0xff, 0xff]);
    println!("evil : {:?}", read_frame(&mut evil));

    // A lying frame: announces 100 bytes, delivers 3.
    let mut short = Cursor::new(vec![0, 0, 0, 100, 1, 2, 3]);
    println!("short: {:?}", read_frame(&mut short));

    // What the allocator says to an absurd request when we ask politely with try_reserve:
    let mut v: Vec<u8> = Vec::new();
    println!("1 PiB: {:?}", v.try_reserve_exact(1 << 50).map_err(|e| e.to_string()));
}
```

```text
good : Ok([104, 101, 108, 108, 111])
evil : Err(Custom { kind: InvalidData, error: "frame of 4294967295 bytes is too large" })
short: Err(Custom { kind: UnexpectedEof, error: "frame is shorter than announced" })
1 PiB: Err("memory allocation failed because the memory allocator returned an error")
```

**Что происходит:**

1. `read_exact` читает ровно 4 байта или завершается ошибкой, а `u32::from_be_bytes` превращает их в длину. `as usize` здесь безопасно: `u32` всегда помещается в `usize` на 64-битных системах.
2. Проверка лимита сразу отвергает заявку на 4 GiB, потратив всего 4 байта ввода. `MAX_FRAME` — это проектное решение: выберите наибольшее значение, которое действительно нужно вашему протоколу.
3. `try_reserve_exact` просит память и возвращает `Err` вместо аварийного завершения, когда система не может её дать (последняя строка показывает абсурдный запрос на 1 PiB, пебибайт, который чисто завершается неудачей).
4. `input.take(len as u64)` — это читатель, который останавливается после `len` байт. Проверка после чтения ловит ввод, закончившийся раньше.

> ⚡ **Производительность:** `Vec::with_capacity(len)` и `vec![0; len]` с недоверенным `len` — привычные ошибки. То же правило относится к распаковке («zip-бомбы»: крошечный файл, разворачивающийся в терабайты), массивам JSON (JavaScript Object Notation, нотация объектов JavaScript) и `read_to_end` на сокете. Всегда ограничивайте размер **результата** через `Read::take` или счётчик.

## 4. Недоверенная глубина

Рекурсия использует стек вызовов, который невелик (около 8 MiB для главного потока в macOS; проверьте командой `ulimit -s`). Парсер вложенных данных (скобки, JSON, XML — два распространённых текстовых формата структурированных данных: JavaScript Object Notation и Extensible Markup Language), который вызывает сам себя на каждом уровне, можно вытолкнуть за край стека вводом всего в один мегабайт. Создайте `src/bin/nesting.rs`:

```rust
/// A tree made of nested brackets: "((()))" is Group(Group(Group(Empty))).
#[allow(dead_code)]
enum Node {
    Empty,
    Group(Box<Node>),
}

/// Parses `(` ... `)` recursively. Looks innocent, right?
fn parse(input: &[u8], pos: &mut usize) -> Option<Node> {
    if input.get(*pos) == Some(&b'(') {
        *pos += 1;
        let inner = parse(input, pos)?;
        if input.get(*pos) != Some(&b')') {
            return None;
        }
        *pos += 1;
        Some(Node::Group(Box::new(inner)))
    } else {
        Some(Node::Empty)
    }
}

fn main() {
    let mut pos = 0;
    let small = parse(b"((()))", &mut pos);
    println!("small input parsed: {}", small.is_some());

    // An attacker sends one million opening brackets: only one megabyte of input.
    let evil = vec![b'('; 1_000_000];
    let mut pos = 0;
    let result = parse(&evil, &mut pos);
    println!("evil input parsed: {}", result.is_some());
}
```

```text
small input parsed: true

thread 'main' (4773359) has overflowed its stack
fatal runtime error: stack overflow, aborting
```

**Что происходит:** каждая `(` создаёт ещё один вложенный вызов `parse`, каждый со своим кадром стека. Миллиону уровней нужно гораздо больше 8 MiB, поэтому процесс **аварийно завершается**. В отличие от паники, переполнение стека перехватить нельзя. Исправление — ограничение глубины, которое передаётся вниз параметром. Создайте `src/bin/nesting_fixed.rs` (новые части помечены):

```rust
const MAX_DEPTH: usize = 128; // new

#[allow(dead_code)]
enum Node {
    Empty,
    Group(Box<Node>),
}

/// Same parser, but it refuses to go deeper than MAX_DEPTH.
fn parse(input: &[u8], pos: &mut usize, depth: usize) -> Option<Node> { // new parameter
    if depth > MAX_DEPTH {
        return None; // too deep: reject the input instead of crashing
    }
    if input.get(*pos) == Some(&b'(') {
        *pos += 1;
        let inner = parse(input, pos, depth + 1)?;
        if input.get(*pos) != Some(&b')') {
            return None;
        }
        *pos += 1;
        Some(Node::Group(Box::new(inner)))
    } else {
        Some(Node::Empty)
    }
}

fn main() {
    let mut pos = 0;
    println!("small input parsed: {}", parse(b"((()))", &mut pos, 0).is_some());

    let evil = vec![b'('; 1_000_000];
    let mut pos = 0;
    println!("evil input parsed: {}", parse(&evil, &mut pos, 0).is_some());
}
```

```text
small input parsed: true
evil input parsed: false
```

Известные парсеры делают ровно это: например, `serde_json` по умолчанию останавливается на глубине вложенности 128.

## 5. Проверка `unsafe`-кода

Внутри блока `unsafe` компилятор доверяет **вам**. У проверки одна цель: убедиться, что *никакой ввод* снаружи, каким бы враждебным он ни был, не может нарушить правила. Чек-лист:

| Вопрос | Зачем |
|---|---|
| Нужен ли `unsafe` вообще? | Большинство случаев «`unsafe` нужен для скорости» исчезают с `get`, итераторами или срезами. Лучший результат проверки — удалить блок. |
| Есть ли у каждого блока `unsafe` комментарий `// SAFETY:`, объясняющий, **почему** правила здесь выполняются (а не что делает код)? | Если вы не можете написать причину, код небезопасен. |
| Может ли **безопасная** функция дойти до блока `unsafe` с любым вводом? | Тогда функция обязана проверить всё, что нужно блоку. Иначе она *некорректна* (unsound) или должна стать `unsafe fn` с разделом `# Safety`. |
| Откуда берутся индексы, длины и указатели? | Всё, что пришло из ввода (поле заголовка, файл), нужно сначала сверить с реальным размером. |
| Какие вызовы — классические ловушки? | `get_unchecked`, `from_raw_parts`, `ptr.add`/`offset`, `set_len`, `assume_init`, `transmute`, `from_utf8_unchecked`. |
| Есть ли тесты, запускаемые под Miri, с враждебным вводом? | Miri проверяет реальное выполнение по правилам. |

Найти весь `unsafe` в своём коде можно командой `grep -rn "unsafe" src/`. Если крейту `unsafe` не нужен, запретите его через `#![forbid(unsafe_code)]` в начале `lib.rs` или `main.rs`: тогда компилятор отвергнет любой `unsafe`. В зависимостях `unsafe` тоже бывает (Урок 10.13).

Вот функция, которая снаружи выглядит безопасной, но таковой не является. Создайте `src/bin/unsound.rs`:

```rust
/// Returns the byte at `index`. Safe to call... or is it?
pub fn byte_at(data: &[u8], index: usize) -> u8 {
    // SAFETY: the caller passes a valid index.
    unsafe { *data.as_ptr().add(index) }
}

fn main() {
    let data = vec![10u8, 20, 30];
    println!("{}", byte_at(&data, 1));
    println!("{}", byte_at(&data, 5)); // index 5 of a 3-byte vector
}
```

**Что происходит:** `byte_at` — *безопасная* функция, поэтому любой вызывающий может передать любой `index`. Комментарий «вызывающий передаёт корректный индекс» — это пожелание, а не доказательство: ничто не делает его истинным. Индекс 5 лежит за пределами 3-байтового выделения, поэтому чтение — **неопределённое поведение**. Посмотрите, что делают четыре разных способа запуска:

```bash
cargo run --quiet --bin unsound
cargo run --quiet --release --bin unsound
```

Вывод отладочной сборки, затем релизной:

```text
20
0
```

```text
20
20
```

Отладочная сборка напечатала `0`, а релизная — `20` для индекса 5: ни ошибки, ни предупреждения, и **разные неверные ответы**. Так выглядит неопределённое поведение: оно «работает». Теперь запустите под Miri (Урок 06.6):

```bash
cargo +nightly miri run --quiet --bin unsound
```

```text
20
error: Undefined Behavior: in-bounds pointer arithmetic failed: attempting to offset pointer by 5 bytes, but got alloc290 which is only 3 bytes from the end of the allocation
 --> src/bin/unsound.rs:4:15
  |
4 |     unsafe { *data.as_ptr().add(index) }
  |               ^^^^^^^^^^^^^^^^^^^^^^^^ Undefined Behavior occurred here
  |
help: alloc290 was allocated here:
 --> src/bin/unsound.rs:8:16
  |
8 |     let data = vec![10u8, 20, 30];
  |                ^^^^^^^^^^^^^^^^^^
```

Miri интерпретирует программу и проверяет каждую операцию с указателями, поэтому находит точную строку. Второй инструмент — **AddressSanitizer**: компилятор добавляет проверки вокруг каждого обращения к памяти, и программа работает почти с обычной скоростью. Ему нужен nightly-компилятор и имя вашей целевой платформы:

```bash
RUSTFLAGS="-Zsanitizer=address" cargo +nightly run --quiet --target aarch64-apple-darwin --bin unsound 2>&1 | grep -E "^[0-9]|ERROR|READ of|#0 |#1 |located"
```

```text
20
==54022==ERROR: AddressSanitizer: heap-buffer-overflow on address 0x602000000935 at pc 0x0001006bd71c bp 0x00016f741820 sp 0x00016f741818
READ of size 1 at 0x602000000935 thread T0
    #0 0x0001006bd718 in _RNvCskCVANAyUgAU_7unsound7byte_at unsound.rs:4
    #1 0x0001006bd470 in _RNvCskCVANAyUgAU_7unsound4main unsound.rs:10
0x602000000935 is located 2 bytes after 3-byte region [0x602000000930,0x602000000933)
```

**Что происходит:** `-Zsanitizer=address` включает ASan (флаги `-Z` существуют только в nightly). ASan сообщает о *heap-buffer-overflow*: чтение 1 байта, через 2 байта после 3-байтовой области, в `byte_at` на строке 4, вызванной со строки 10. (Адреса и числа при каждом запуске разные.)

Какой инструмент когда? Miri точнее всех знает правила Rust, но медленный и не умеет выполнять код, обращающийся к операционной системе или библиотекам C. ASan быстрый и работает с обычными программами и кодом на C, но видит только обращения, которые попали в память вне выделений. Используйте оба на тестах `unsafe`-кода. В Linux есть и другие санитайзеры (например, ThreadSanitizer для гонок данных).

Исправление — сделать функцию **корректной**. Лучше всего удалить `unsafe`.

```rust
pub fn byte_at(data: &[u8], index: usize) -> Option<u8> {
    data.get(index).copied()
}
```

Если `unsafe` нужно оставить (измеренный горячий путь, Урок 09.11), безопасная функция должна проверить то, на что опирается блок, а комментарий — назвать эту проверку:

```rust
pub fn byte_at(data: &[u8], index: usize) -> Option<u8> {
    if index < data.len() {
        // SAFETY: `index < data.len()` was just checked, so the pointer stays inside the slice.
        Some(unsafe { *data.as_ptr().add(index) })
    } else {
        None
    }
}
```

Или сделайте контракт обязанностью вызывающего: `pub unsafe fn byte_at_unchecked(...)` с разделом `# Safety` («`index` должен быть меньше `data.len()`»). Теперь никто не вызовет её из безопасного кода случайно.

> 🔒 **Безопасность:** «в моих тестах работает» — не результат проверки `unsafe`-кода. Запускайте тесты под Miri, используйте враждебный ввод (пустой, огромный, на единицу больше) и делайте каждый блок `unsafe` как можно меньше.

## Частые ошибки

**1. `unwrap()` на вводе.** `let n: u32 = text.parse().unwrap();` позволяет любому, кто отправит `abc`, уронить сервис. Используйте `?` или `match` и возвращайте ошибку.

**2. `as` для размеров.** `len as u32` молча обрезает `usize` до 32 бит, и ввод длиной 4 294 967 297 байт выглядит как ввод длиной 1 байт. Используйте `u32::try_from(len)?`.

**3. Выделение памяти по заявленному размеру.** `vec![0; announced_len]` и `Vec::with_capacity(announced_len)` до проверки `announced_len`.

**4. Вера комментарию над `unsafe`.** Комментарий `// SAFETY:` — это утверждение; проверяйте его как код. «Вызывающий гарантирует это» допустимо только для `unsafe fn`.

**5. Тестирование только удачного пути.** Парсерам нужны тесты с пустым, однобайтовым, обрезанным, слишком длинным и глубоко вложенным вводом.

## Попробуйте сами

1. Запустите `cargo clippy` в пакете и прочитайте каждое предупреждение. Какие lint-правила срабатывают на `overflow.rs`? (`arithmetic_side_effects` указывает на `count * size`.)
2. Измените `MAX_DEPTH` в `nesting_fixed.rs` на `10_000_000` и запустите. Что происходит и почему?
3. Замените `byte_at` в `unsound.rs` исправленной версией, возвращающей `Option` (замените в `println!` формат на `{:?}`), и снова запустите под Miri.

## Упражнения

### Упражнение 1 — Два инструмента, одна ошибка (легко)

Запустите `unsound.rs` под Miri и под AddressSanitizer. Для каждого инструмента запишите: вид ошибки и строку исходника, на которую он указывает.

<details>
<summary>Подсказка</summary>

Используйте две команды из раздела 5. Ищите первую строку, где названы файл и номер строки.

</details>

<details>
<summary>Ответ</summary>

Miri: `Undefined Behavior: in-bounds pointer arithmetic failed: attempting to offset pointer by 5 bytes, but got alloc290 which is only 3 bytes from the end of the allocation`, в `src/bin/unsound.rs:4:15`, с пометкой, что выделение сделано на строке 8.

ASan: `heap-buffer-overflow`, `READ of size 1`, `2 bytes after 3-byte region`, в `byte_at` на `unsound.rs:4`, вызвана из `main` на `unsound.rs:10`.

Почему это работает: оба сообщают об одной и той же ошибке (чтение вне 3-байтового выделения) разными словами. Miri проверяет правила Rust в интерпретаторе; ASan проверяет реальные обращения к памяти в скомпилированной программе.

</details>

### Упражнение 2 — Парсер, который никогда не паникует (средне)

В новом библиотечном пакете напишите `pub fn read_len(message: &[u8]) -> Option<u16>` (исправленная функция из раздела 2). Напишите тест, который вызывает её со входами всех длин от 0 до 3 и проверяет, что она возвращает `Some` только когда есть хотя бы два байта, и ещё один тест на значение `258`.

<details>
<summary>Подсказка</summary>

`vec![0xff; size]` создаёт ввод любой длины. Свойство: `result.is_some() == (size >= 2)`.

</details>

<details>
<summary>Ответ</summary>

```rust
/// Reads a 2-byte big-endian length. Never panics: short input gives `None`.
pub fn read_len(message: &[u8]) -> Option<u16> {
    let [a, b, ..] = message else {
        return None;
    };
    Some(u16::from_be_bytes([*a, *b]))
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn read_len_never_panics_on_short_input() {
        for size in 0..4 {
            let input = vec![0xff; size];
            let result = read_len(&input);
            assert_eq!(result.is_some(), size >= 2, "size {size}");
        }
        assert_eq!(read_len(&[0x01, 0x02, 0xff]), Some(258));
    }
}
```

`cargo test` показывает тест как `ok`.

Почему это работает: тест покрывает границу (размеры 1, 2) и пустой ввод — ровно там, где версия с индексацией запаниковала бы.

</details>

### Упражнение 3 — Читатель кадров с лимитом (сложно)

Добавьте в свою библиотеку `read_frame` (раздел 3) с `MAX_FRAME` как `pub fn`. Напишите тесты с `std::io::Cursor`: нормальный кадр, заявленная длина 4 GiB (должна быть `InvalidData`, и память выделяться не должна), кадр короче заявленного и пустой ввод (оба — `UnexpectedEof`).

<details>
<summary>Подсказка</summary>

`io::Error::kind()` возвращает `ErrorKind`. `Cursor::new(vec![...])` — это `Read`, читающий из памяти. Используйте функцию из `frame_alloc.rs`.

</details>

<details>
<summary>Ответ</summary>

Функция — та, что в разделе 3 (сделайте её `pub`). Тесты внутри `mod tests` с `use std::io::Cursor;`:

```rust
    #[test]
    fn good_frame_is_read() {
        let mut input = Cursor::new(vec![0, 0, 0, 5, b'h', b'e', b'l', b'l', b'o']);
        assert_eq!(read_frame(&mut input).unwrap(), b"hello");
    }

    #[test]
    fn huge_announced_length_is_rejected_without_allocating() {
        let mut input = Cursor::new(vec![0xff, 0xff, 0xff, 0xff]);
        let err = read_frame(&mut input).unwrap_err();
        assert_eq!(err.kind(), io::ErrorKind::InvalidData);
    }

    #[test]
    fn short_and_empty_inputs_are_errors() {
        let mut short = Cursor::new(vec![0, 0, 0, 100, 1, 2, 3]);
        assert_eq!(read_frame(&mut short).unwrap_err().kind(), io::ErrorKind::UnexpectedEof);
        let mut empty = Cursor::new(Vec::new());
        assert_eq!(read_frame(&mut empty).unwrap_err().kind(), io::ErrorKind::UnexpectedEof);
    }
```

`cargo test` показывает все тесты как `ok`.

Почему это работает: `read_exact` на пустом вводе завершается `UnexpectedEof` до того, как длина известна, а ограничение размера останавливает запрос на 4 GiB после чтения всего 4 байтов.

</details>

## Вопросы для самопроверки

1. Что из этого безопасный Rust **не** предотвращает: использование после освобождения, панику от `data[10]`, гонку данных?

<details>
<summary>Ответ</summary>

Панику от `data[10]`. Безопасный Rust делает обращение безопасным для памяти, останавливая программу; это всё равно отказ в обслуживании, если индекс пришёл из недоверенного ввода.

</details>

2. Сообщение начинается с 4-байтовой длины. Что вы делаете до выделения буфера?

<details>
<summary>Ответ</summary>

Сравниваете длину с разумным максимумом и отвергаете большие значения; затем выделяете через `try_reserve` и читаете через `take`. Никогда не выделяйте память по числу, пришедшему снаружи, без лимита.

</details>

3. Почему для значений из ввода `count * size` требует `checked_mul`?

<details>
<summary>Ответ</summary>

В релизных сборках умножение молча «заворачивается» и даёт неверный, меньший размер. `checked_mul` возвращает `None` при переполнении, и вы можете отвергнуть ввод.

</details>

4. Что делает безопасную функцию с `unsafe` внутри *некорректной* (unsound)?

<details>
<summary>Ответ</summary>

Если какой-то ввод, который может передать безопасный вызывающий, приводит к неопределённому поведению, например непроверенный индекс. Безопасная функция должна быть верна для **любого** ввода.

</details>

5. Когда использовать Miri, а когда AddressSanitizer?

<details>
<summary>Ответ</summary>

Miri — для небольших тестов `unsafe`-кода на Rust: он точен, но медленный и не умеет вызывать код на C. AddressSanitizer — для скомпилированной программы, включая большие программы и библиотеки C, чтобы находить выходы за границы и использование после освобождения. Для `unsafe`-кода используйте оба.

</details>

## Итоги

- Безопасный Rust убирает переполнения буфера, использование после освобождения и гонки данных, но не паники, исчерпание ресурсов, переполнение стека, целочисленное переполнение и логические ошибки.
- Недоверенный ввод никогда не должен вызывать панику: возвращайте `Option` или `Result`, избегайте `[i]` и `unwrap` на вводе и тестируйте короткие, пустые и абсурдные входы.
- Ограничивайте каждый размер и каждую глубину, пришедшие снаружи: размеры выделений, размеры результата, глубину рекурсии.
- Используйте арифметику `checked_*` и `try_from` для размеров из ввода.
- Безопасная функция с `unsafe` внутри должна быть корректной для любого возможного ввода; комментарий `// SAFETY:` должен давать причину, которая действительно верна.
- Неопределённое поведение часто «работает»; для его поиска используйте Miri и AddressSanitizer, а для раннего поиска рискованного кода — lint-правила Clippy (`indexing_slicing`, `unwrap_used`, `arithmetic_side_effects`, `undocumented_unsafe_blocks`).

## Источники и что почитать

- [The Rustonomicon](https://doc.rust-lang.org/nomicon/) — официальное руководство по написанию и проверке `unsafe`-кода, включая то, что такое корректность (soundness).
- [Miri на GitHub](https://github.com/rust-lang/miri) — установка, флаги и список того, что обнаруживает Miri.
- [The Rust Unstable Book: sanitizer](https://doc.rust-lang.org/beta/unstable-book/compiler-flags/sanitizer.html) — какие санитайзеры существуют и как их включать.
- [Список lint-правил Clippy](https://rust-lang.github.io/rust-clippy/master/index.html) — документация по `indexing_slicing`, `unwrap_used`, `arithmetic_side_effects` и `undocumented_unsafe_blocks`.
- [Rust Secure Code Working Group](https://github.com/rust-secure-code/wg) — руководства и инструменты для написания безопасного Rust.
- [std::vec::Vec::try_reserve](https://doc.rust-lang.org/std/vec/struct.Vec.html#method.try_reserve) — выделение памяти с обработкой ошибки.

---

[🏠 Начало](../START-HERE.md) · [← 10.11 TLS и сертификаты](11-tls-and-certificates.md) · [📋 Этап 10](README.md) · [10.13 Безопасность цепочки поставок: cargo-audit и cargo-deny →](13-supply-chain-security.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
