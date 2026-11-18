<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 17.2 Надёжность записи: fsync и устойчивость к сбоям](02-durability-and-fsync.md) · [📋 Этап 17](README.md) · [17.4 Журнал упреждающей записи (WAL) →](04-write-ahead-log.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/17-storage-and-databases/03-binary-file-formats.md) <!-- nav -->

# Урок 17.3 — Двоичные форматы файлов и сериализация

> **Этап 17 — Как устроены хранилища и базы данных** · Урок 3 из 15 · ⏱ около 2,5 часа (2 занятия)

## Что вы узнаете

- Почему движки хранения держат данные в двоичном виде, а не в виде текста.
- Как превращать целые числа в байты и обратно и почему нужно выбрать порядок байтов и записать его.
- Как спроектировать формат записи с префиксами длины, чтобы читатель знал, где кончается каждая запись.
- Как контрольная сумма CRC-32 обнаруживает разорванные записи и повреждённые байты.
- Как сериализовать целые структуры с `serde` и `postcard` и что такое varint.

## Перед началом

- Вы прошли [Урок 17.2 — Надёжность записи: fsync и устойчивость к сбоям](02-durability-and-fsync.md): знаете
  разорванные записи, и в `code/p17-storage` есть `src/crash.rs`.
- Вы знаете биты, байты, шестнадцатеричную запись и порядок байтов из
  [Урока 12.1 — Биты, байты и системы счисления](../12-computer-systems/01-bits-bytes-and-number-systems.md), а битовые
  маски — из [Урока 12.4](../12-computer-systems/04-bitwise-operations.md).
- Вы пользовались `serde` с JSON (JavaScript Object Notation, текстовый формат данных) в [Уроке 05.11](../05-rust-intermediate/11-practical-crates-serde-clap-errors.md).
- Добавьте крейты этого урока:

```bash
cd ~/projects/rust/code/p17-storage
cargo add crc32fast@1
cargo add postcard@1 --features alloc
cargo add serde@1 --features derive
```

## Новые слова

| Слово | Значение |
|---|---|
| **двоичный формат** (binary format) | Способ хранить данные как сырые байты с заранее заданным смыслом (например, «байты 4–7 — длина ключа»), а не как читаемый текст. |
| **кодировать / декодировать** (encode / decode) | Превратить значение в байты / превратить байты обратно в значение. |
| **сериализация** (serialization) | Кодирование целой структуры (struct, списка) в байты, чтобы её можно было сохранить или отправить, и последующее декодирование. |
| **little-endian / big-endian** | Порядки байтов: младший байт числа идёт первым / старший байт идёт первым. |
| **префикс длины** (length prefix) | Число, записанное *перед* куском данных, которое говорит, сколько в нём байтов. |
| **запись** (record) | Одна единица данных в файле; здесь — один ключ и одно значение с заголовком. |
| **контрольная сумма** (checksum) | Небольшое число, вычисленное по блоку байтов. Если байты меняются, контрольная сумма (почти наверняка) тоже меняется. |
| **CRC** (Cyclic Redundancy Check, циклический избыточный код) | Семейство контрольных сумм, которые быстро считаются и очень хорошо ловят повреждённые байты. CRC-32 даёт 32-битное число. |
| **MAC** (Message Authentication Code, код аутентификации сообщения) | Короткая метка, вычисленная по данным и секретному ключу; правильную метку может сделать только владелец ключа (Урок 10.9). |
| **магическое число** (magic number) | Фиксированные байты в начале файла, которые говорят «это мой формат», чтобы программа не прочитала чужой файл. |
| **varint** (variable-length integer, целое переменной длины) | Кодирование целых чисел, при котором маленькие числа занимают меньше байтов: 7 бит на байт. Его также называют LEB128 (Little-Endian Base 128). |

## 1. Текст или двоичный вид?

Число 1000 можно хранить как текст `"1000"` или как четыре байта `u32`. Поместите это в `src/bin/int_bytes.rs`:

```rust
fn main() {
    let n: u32 = 1000;
    println!("as text:       {:?}", n.to_string().as_bytes());
    println!("little-endian: {:02x?}", n.to_le_bytes());
    println!("big-endian:    {:02x?}", n.to_be_bytes());

    let back = u32::from_le_bytes([0xe8, 0x03, 0x00, 0x00]);
    println!("decoded:       {back}");
}
```

**Что происходит:**

1. `n.to_string().as_bytes()` показывает текстовую форму: коды ASCII (American Standard Code for Information Interchange,
   американский стандартный код для обмена информацией) символов `'1'`, `'0'`, `'0'`, `'0'`.
2. `to_le_bytes` возвращает `[u8; 4]` с младшим байтом впереди; `to_be_bytes` — со старшим байтом впереди. `{:02x?}`
   печатает каждый байт двумя шестнадцатеричными цифрами.
3. `u32::from_le_bytes` делает обратное: на входе четыре байта, на выходе одно число.

`cargo run --bin int_bytes`:

```text
as text:       [49, 48, 48, 48]
little-endian: [e8, 03, 00, 00]
big-endian:    [00, 00, 03, e8]
decoded:       1000
```

1000 — это `0x3e8`. Как текст здесь ему нужно 4 байта, но для 4 000 000 000 уже 10 байт, и читателю приходится искать
конец и разбирать цифры. Как `u32` оно **всегда занимает 4 байта**, а декодирование — это одно копирование. Движкам
хранения нужно именно это: фиксированные размеры, чтобы можно было сразу перейти к «байту 8», не читая байты 0–7, и
никакого разбора.

Цена в том, что без формата байты ничего не значат. `[e8, 03, 00, 00]` — это 1000, только если вы знаете, что это
`u32` в little-endian. Поэтому каждый двоичный формат должен **записать** свой порядок байтов. В этом курсе используется
little-endian — тот же порядок, который естественен для процессоров вашего Mac и почти всех серверов.

> 📏 **Соглашение:** указывайте порядок байтов в имени каждой вспомогательной функции (`to_le_bytes`, `read_u32_le`).
> Никогда не используйте `to_ne_bytes` («native endian», родной порядок) в формате файла: файл, записанный на одной
> машине, должен читаться на любой другой.

## 2. Формат записи

Хранилищу «ключ-значение» нужно сохранять в файле пары вроде `apple → red`, одну за другой. Читатель должен знать, где
кончается одна запись и начинается следующая, и должен замечать разорванную или повреждённую запись (Урок 17.2). Такое
расположение решает обе задачи:

```text
 offset 0      4           8             12
 ┌────────────┬───────────┬─────────────┬─────────────┬───────────────┐
 │ crc32 (u32)│ key_len   │ value_len   │ key bytes   │ value bytes   │
 │            │ (u32, LE) │ (u32, LE)   │ (key_len)   │ (value_len)   │
 └────────────┴───────────┴─────────────┴─────────────┴───────────────┘
   checksum of everything after it ─────────────────────────────────────►
```

Два **префикса длины** говорят читателю, сколько байтов идёт дальше. **Контрольная сумма** в начале покрывает остаток
записи, включая длины. Создайте `src/record.rs` и начните с типов:

```rust
use std::fmt;

/// Bytes before the key: checksum, key length and value length, each a little-endian u32.
pub const HEADER_LEN: usize = 12;
/// The largest key or value we accept (1 MiB). Anything bigger on disk is damage.
pub const MAX_LEN: usize = 1 << 20;

#[derive(Debug, Clone, PartialEq, Eq)]
pub struct Record {
    pub key: Vec<u8>,
    pub value: Vec<u8>,
}

#[derive(Debug, PartialEq, Eq)]
pub enum DecodeError {
    /// Fewer bytes than the record needs: the end of a torn write.
    Truncated,
    /// A length field larger than MAX_LEN.
    TooLarge(usize),
    /// The checksum does not match the bytes.
    BadChecksum,
}

impl fmt::Display for DecodeError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            DecodeError::Truncated => write!(f, "record is truncated"),
            DecodeError::TooLarge(n) => write!(f, "length {n} is larger than {MAX_LEN}"),
            DecodeError::BadChecksum => write!(f, "checksum mismatch"),
        }
    }
}

impl std::error::Error for DecodeError {}
```

Ключи и значения — это `Vec<u8>`, просто байты: движку хранения всё равно, текст в них, числа или картинки. Затем
кодировщик, в том же файле:

```rust
/// Appends one encoded record to `out`.
pub fn encode(key: &[u8], value: &[u8], out: &mut Vec<u8>) {
    assert!(
        key.len() <= MAX_LEN && value.len() <= MAX_LEN,
        "key or value too large"
    );
    let start = out.len();
    out.extend_from_slice(&[0; 4]); // room for the checksum
    out.extend_from_slice(&(key.len() as u32).to_le_bytes());
    out.extend_from_slice(&(value.len() as u32).to_le_bytes());
    out.extend_from_slice(key);
    out.extend_from_slice(value);
    let crc = crc32fast::hash(&out[start + 4..]);
    out[start..start + 4].copy_from_slice(&crc.to_le_bytes());
}
```

**Что происходит:**

1. `assert!` защищает формат: длина должна помещаться в `u32`, и мы также отказываемся от всего, что больше `MAX_LEN`. Без
   этой проверки `as u32` молча обрезал бы слишком большую длину.
2. `encode` дописывает в `out`, а не возвращает новый `Vec`, поэтому много записей можно собрать в одном буфере и
   записать одним системным вызовом (Урок 17.1).
3. Сначала мы кладём четыре нулевых байта как заглушку, потом длины и данные.
4. `crc32fast::hash` вычисляет CRC-32 всего, что идёт после заглушки; `copy_from_slice` кладёт результат в заглушку.

## 3. Читаем обратно

Декодирование должно быть осторожным: байты приходят с диска, возможно, после сбоя. Добавьте в `src/record.rs`:

```rust
fn read_u32(bytes: &[u8], at: usize) -> u32 {
    let four: [u8; 4] = bytes[at..at + 4].try_into().unwrap();
    u32::from_le_bytes(four)
}

/// Decodes the record at the start of `bytes`.
/// Returns the record and the number of bytes it used.
pub fn decode(bytes: &[u8]) -> Result<(Record, usize), DecodeError> {
    if bytes.len() < HEADER_LEN {
        return Err(DecodeError::Truncated);
    }
    let crc = read_u32(bytes, 0);
    let key_len = read_u32(bytes, 4) as usize;
    let value_len = read_u32(bytes, 8) as usize;
    for len in [key_len, value_len] {
        if len > MAX_LEN {
            return Err(DecodeError::TooLarge(len));
        }
    }
    let total = HEADER_LEN + key_len + value_len;
    if bytes.len() < total {
        return Err(DecodeError::Truncated);
    }
    if crc32fast::hash(&bytes[4..total]) != crc {
        return Err(DecodeError::BadChecksum);
    }
    let key = bytes[HEADER_LEN..HEADER_LEN + key_len].to_vec();
    let value = bytes[HEADER_LEN + key_len..total].to_vec();
    Ok((Record { key, value }, total))
}
```

**Что происходит:**

1. `try_into()` превращает срез из 4 байт в массив `[u8; 4]`. Он возвращает `Result`, потому что срез может быть любой
   длины; у нашего ровно 4 байта, так что `unwrap` не может сработать с ошибкой.
2. Проверки идут в безопасном порядке: сначала «есть ли целый заголовок?», затем «разумны ли длины?», затем «все ли байты
   на месте?» и только потом «верна ли контрольная сумма?». Ни один шаг не читает за концом `bytes`.
3. `decode` возвращает, сколько байтов заняла запись, чтобы вызывающий знал, где начинается следующая.

Зарегистрируйте модуль в `src/lib.rs` (строки из Урока 17.2 оставьте):

```rust
pub mod crash;
pub mod record;
pub mod synclog;
```

Теперь программа, которая пишет две записи в файл и читает их обратно, `src/bin/records.rs`:

```rust
use p17_storage::record::{self, DecodeError};
use std::fs;

fn main() -> Result<(), Box<dyn std::error::Error>> {
    // 1. Encode two records into one buffer and save it.
    let mut buf = Vec::new();
    record::encode(b"apple", b"red", &mut buf);
    record::encode(b"kiwi", b"green", &mut buf);
    fs::write("records.bin", &buf)?;
    println!("wrote {} bytes", buf.len());

    // 2. Read the file back and decode records one after another.
    let bytes = fs::read("records.bin")?;
    let mut pos = 0;
    while pos < bytes.len() {
        match record::decode(&bytes[pos..]) {
            Ok((rec, used)) => {
                println!(
                    "offset {pos:>2}: {} = {}",
                    String::from_utf8_lossy(&rec.key),
                    String::from_utf8_lossy(&rec.value)
                );
                pos += used;
            }
            Err(DecodeError::Truncated) => {
                println!("offset {pos:>2}: torn record, stop");
                break;
            }
            Err(e) => return Err(e.into()),
        }
    }
    Ok(())
}
```

**Что происходит:**

1. Обе записи попадают в один буфер и один `fs::write`.
2. Читатель идёт по байтам с помощью `pos`. После каждой записи он прыгает вперёд на `used` байт.
3. `String::from_utf8_lossy` показывает байты как текст, заменяя недопустимый UTF-8 (кодировка текста из Урока 12.5) на `�`; для печати этого достаточно.
4. Обрезанная запись завершает цикл (разорванный хвост из Урока 17.2); любая другая ошибка возвращается. `e.into()`
   превращает наш `DecodeError` в `Box<dyn Error>`; это работает, потому что мы реализовали `std::error::Error`.

`cargo run --bin records`, затем посмотрите на сырые байты через `xxd` (он печатает смещения, байты в шестнадцатеричном
виде и справа те же байты как текст):

```bash
cargo run --bin records
xxd records.bin
```

```text
wrote 41 bytes
offset  0: apple = red
offset 20: kiwi = green
00000000: 9c22 6d4f 0500 0000 0300 0000 6170 706c  ."mO........appl
00000010: 6572 6564 8368 d6a6 0400 0000 0500 0000  ered.h..........
00000020: 6b69 7769 6772 6565 6e                   kiwigreen
```

Прочитайте первую запись в дампе: `9c22 6d4f` — контрольная сумма, `0500 0000` — длина ключа 5 (little-endian),
`0300 0000` — длина значения 3, затем `apple` и `red`. Вторая запись начинается со смещения 20 (`0x14`) со своей
контрольной суммой `8368 d6a6`.

## 4. Контрольные суммы: ловим повреждения

**Контрольная сумма** — это короткий отпечаток блока байтов. CRC-32 вычисляется по фиксированной формуле очень быстро
(`crc32fast` использует специальные инструкции процессора). Два факта делают её полезной для хранения:

- Если случайное повреждение меняет байты, вероятность, что контрольная сумма всё равно совпадёт, — примерно 1 на
  4 миллиарда (2³²).
- Некоторые повреждения ловятся **всегда**: любой один изменённый бит и любая пачка изменённых битов длиной не больше 32.

Поэтому после сбоя читатель может отличить «целую правильную запись» от «разорванной записи» или «мусора, который
случайно оказался в файле». Разорванная запись обычно проявляется как `Truncated` (файл кончается слишком рано). Но после
сбоя файловая система может оставить в конце файла и старый мусор или нулевые байты; тогда длины выглядят правильными, и
проблему показывает только контрольная сумма.

> 🔒 **Безопасность:** CRC защищает от *случайностей*, а не от злоумышленника. Кто угодно может изменить байты и
> посчитать новую CRC. Чтобы заметить намеренные изменения, нужен MAC (Message Authentication Code, код аутентификации
> сообщения) с секретным ключом из [Урока 10.9](../10-security-fundamentals/09-cryptography-hashes-macs-and-randomness.md).
> Обратите внимание и на `MAX_LEN`: длина, прочитанная с диска, — это **ввод**, и без ограничения повреждённая длина в
> четыре миллиарда заставила бы читателя попытаться выделить 4 ГиБ (гибибайта, 4 × 1 073 741 824 байта).

Тесты формата идут в конец `src/record.rs`:

```rust
#[cfg(test)]
mod tests {
    use super::*;

    fn encoded(key: &[u8], value: &[u8]) -> Vec<u8> {
        let mut buf = Vec::new();
        encode(key, value, &mut buf);
        buf
    }

    #[test]
    fn round_trip() {
        let buf = encoded(b"apple", b"red");
        let (record, used) = decode(&buf).unwrap();
        assert_eq!(record.key, b"apple");
        assert_eq!(record.value, b"red");
        assert_eq!(used, HEADER_LEN + 5 + 3);
    }

    #[test]
    fn a_flipped_bit_is_detected() {
        let mut buf = encoded(b"apple", b"red");
        let last = buf.len() - 1;
        buf[last] ^= 0b0000_0001; // "red" becomes "ree"
        assert_eq!(decode(&buf), Err(DecodeError::BadChecksum));
    }

    #[test]
    fn a_cut_record_is_truncated() {
        let buf = encoded(b"apple", b"red");
        assert_eq!(decode(&buf[..15]), Err(DecodeError::Truncated));
    }

    #[test]
    fn a_huge_length_is_rejected() {
        let mut buf = encoded(b"apple", b"red");
        buf[4..8].copy_from_slice(&u32::MAX.to_le_bytes());
        assert_eq!(decode(&buf), Err(DecodeError::TooLarge(u32::MAX as usize)));
    }
}
```

**Что происходит:**

1. `round_trip` — самый важный тест любого формата: закодировать, декодировать и получить то же самое.
2. `^= 0b0000_0001` переворачивает младший бит последнего байта (`d` = `0x64` становится `e` = `0x65`), как сбой бита на
   плохом устройстве.
3. `&buf[..15]` оставляет 15 байт из 20, как разорванная запись.
4. Последний тест записывает `u32::MAX` в длину ключа, как мусор после сбоя.

`cargo test --lib record::`:

```text
running 4 tests
test record::tests::round_trip ... ok
test record::tests::a_flipped_bit_is_detected ... ok
test record::tests::a_cut_record_is_truncated ... ok
test record::tests::a_huge_length_is_rejected ... ok

test result: ok. 4 passed; 0 failed; 0 ignored; 0 measured; 3 filtered out; finished in 0.00s
```

> 🧪 **Тестирование:** для каждого двоичного формата проверяйте круговой путь (round trip), а затем все виды повреждений,
> какие можете придумать: обрезка, перевёрнутый бит, слишком большая длина, сплошные нули. Декодер должен возвращать
> ошибку для каждого и никогда не паниковать.

## 5. Сериализация структур с serde и postcard

Записывать каждое поле вручную нормально для маленького заголовка записи. Для структур побольше в Rust есть **serde**:
вы выводите (derive) `Serialize` и `Deserialize`, а крейт формата превращает значение в байты. Вы использовали его с
JSON в Уроке 05.11. Для компактных двоичных данных в этом курсе используется **postcard**. (Старые руководства
используют крейт `bincode`; он больше не поддерживается, так что не добавляйте его в новые проекты.) Поместите это в
`src/bin/postcard_demo.rs`:

```rust
use serde::{Deserialize, Serialize};

#[derive(Debug, PartialEq, Serialize, Deserialize)]
struct User {
    id: u32,
    name: String,
    admin: bool,
}

fn main() -> Result<(), postcard::Error> {
    let user = User {
        id: 1000,
        name: "ada".to_string(),
        admin: true,
    };
    let bytes: Vec<u8> = postcard::to_allocvec(&user)?;
    println!("{} bytes: {:02x?}", bytes.len(), bytes);

    let back: User = postcard::from_bytes(&bytes)?;
    println!("{back:?}");
    assert_eq!(back, user);
    Ok(())
}
```

**Что происходит:**

1. `#[derive(Serialize, Deserialize)]` генерирует код, который проходит по полям по порядку.
2. `postcard::to_allocvec` сериализует в новый `Vec<u8>` (его даёт включённая вами фича `alloc`).
3. `postcard::from_bytes` декодирует. Если байты не подходят к типу, он возвращает ошибку, а не паникует.

`cargo run --bin postcard_demo`:

```text
7 bytes: [e8, 07, 03, 61, 64, 61, 01]
User { id: 1000, name: "ada", admin: true }
```

Всего 7 байт. Имён полей в выводе нет (в отличие от JSON): читатель должен знать структуру. `61 64 61` — это `"ada"`,
перед ним его длина `03`, а `01` — это `true`. Но почему 1000 — это `e8 07`, а не `e8 03 00 00`?

postcard записывает целые числа как **varint**. Каждый байт несёт 7 бит числа; старший бит (`0x80`) говорит «дальше
есть ещё байт». 1000 в двоичном виде — `0b111_1101000`:

```text
lowest 7 bits:  1101000 = 0x68, more follows → 0x68 | 0x80 = 0xe8
next 7 bits:    0000111 = 0x07, last byte    → 0x07
```

Маленькие числа, а они встречаются чаще всего, занимают один байт; `u32` занимает не больше 5. В упражнении 3 вы
напишете это кодирование сами.

Что когда использовать? **Горячие фиксированные части** формата хранения (заголовки, длины, контрольные суммы) пишите
вручную: их точные байты никогда не должны меняться, и их читают миллионы раз. Для остального используйте
**serde + postcard**: значения-структуры, файлы метаданных, сообщения.

## 6. Форматы живут годами: магические числа и версии

Файл базы данных, записанный сегодня, может открыть программа следующего года. Поэтому настоящие форматы начинаются с
небольшого **заголовка файла**:

```text
 ┌──────────────────┬──────────────┬─────────────── records … ───
 │ magic: "P17LOG\0"│ version: u16 │
 │ (7 bytes)        │ (2 bytes LE) │
 └──────────────────┴──────────────┴──────────────────────────────
```

**Магическое число** позволяет программе сразу отказаться от чужого файла («это не файл журнала»), а не декодировать
мусор. **Версия** позволяет новому коду читать старые файлы: если версия 1, используется старый декодер. Например, файлы
SQLite начинаются с текста `SQLite format 3` и нулевого байта, а изображения PNG (Portable Network Graphics) — с байтов `89 50 4e 47`.

> ⚠️ **Частая ошибка:** изменить поле в структуре, которая сериализуется через postcard, и затем читать старые файлы. В
> postcard нет имён полей, поэтому старые байты декодируются по новой раскладке. Меняйте форматы только вместе с новым
> номером версии.

## Частые ошибки

1. **Родной порядок байтов (`to_ne_bytes`) в файле.** Тогда файл читается только на машинах с тем же порядком байтов.
2. **Доверять длине с диска.** Без максимума повреждённая длина заставит читателя выделить гигабайты или читать за концом.
   Проверяйте длины, прежде чем их использовать.
3. **Индексировать срезы, не проверив длину.** `bytes[4..8]` паникует, если `bytes` короче. Декодер должен возвращать
   `Truncated`, а не паниковать.
4. **Считать контрольную сумму только по данным, без длин.** Если длина повреждена, читатель ищет данные не там. Наша
   контрольная сумма покрывает всё, что идёт после неё.

## Попробуйте сами

1. Замените `"kiwi"` в `records.rs` на `"banana"` и до запуска предскажите, где начнётся вторая запись.
2. Выполните `xxd records.bin` и найдите два поля длины второй записи.
3. В `postcard_demo.rs` замените `id` на 5 и на 300. Предскажите байты, затем проверьте.

## Упражнения

### Упражнение 1 — Порядок байтов в тесте (легко)

Добавьте тест `integers_are_little_endian` к тестам в `src/record.rs`: для `n: u32 = 0x0102_0304` проверьте результаты
`to_le_bytes` и `to_be_bytes`, а также то, что `from_le_bytes` возвращает `n`.

<details>
<summary>Подсказка</summary>

Младший байт `0x0102_0304` — это `0x04`. Little-endian ставит его первым.

</details>

<details>
<summary>Ответ</summary>

```rust
    #[test]
    fn integers_are_little_endian() {
        let n: u32 = 0x0102_0304;
        assert_eq!(n.to_le_bytes(), [0x04, 0x03, 0x02, 0x01]);
        assert_eq!(n.to_be_bytes(), [0x01, 0x02, 0x03, 0x04]);
        assert_eq!(u32::from_le_bytes([0x04, 0x03, 0x02, 0x01]), n);
    }
```

Почему это работает: шестнадцатеричные цифры идут парами, одна пара на байт, поэтому `0x0102_0304` — это байты
`01 02 03 04` от старшего к младшему. Little-endian переворачивает этот порядок.

</details>

### Упражнение 2 — Любая обрезка — это разорванная запись (средне)

В Уроке 17.2 вы узнали, что сбой оставляет префикс записанного. Напишите тест `every_cut_is_truncated`: закодируйте одну
запись, затем для каждой длины `cut` от 0 до длины записи минус 1 проверьте, что декодирование `&buf[..cut]` возвращает
`Err(DecodeError::Truncated)`. В конце проверьте, что весь буфер декодируется.

<details>
<summary>Подсказка</summary>

Цикл `for cut in 0..buf.len()`. Добавьте `"cut at {cut}"` как сообщение `assert_eq!`, чтобы при неудаче было видно, где.

</details>

<details>
<summary>Ответ</summary>

```rust
    #[test]
    fn every_cut_is_truncated() {
        let buf = encoded(b"apple", b"red");
        for cut in 0..buf.len() {
            assert_eq!(
                decode(&buf[..cut]),
                Err(DecodeError::Truncated),
                "cut at {cut}"
            );
        }
        assert!(decode(&buf).is_ok());
    }
```

Почему это работает: при обрезке короче 12 байт заголовок неполон; при более длинной длины правильные, но данных не
хватает. В обоих случаях `decode` замечает это **до того**, как читает недостающие байты. На этом свойстве держится WAL (журнал упреждающей записи)
из Урока 17.4: разорванная последняя запись распознаётся как `Truncated`, а не как повреждение или паника.

</details>

### Упражнение 3 — Свой varint (сложно)

Создайте `src/varint.rs` (и добавьте `pub mod varint;` в `src/lib.rs`) с
`pub fn encode_varint(n: u64, out: &mut Vec<u8>)` и `pub fn decode_varint(bytes: &[u8]) -> Option<(u64, usize)>`
(число и сколько байтов использовано, или `None`, если ввод кончился слишком рано). Тест: 5 кодируется как `[0x05]`;
1000 — как `[0xe8, 0x07]`, ровно как `postcard::to_allocvec(&1000u32)`; круговой путь работает для 0, 1, 127, 128, 300,
`1 << 35` и `u64::MAX`; а одинокий `[0xe8]` декодируется как `None`.

<details>
<summary>Подсказка</summary>

Кодирование: пока `n >= 0x80`, кладите `(n as u8 & 0x7f) | 0x80` и сдвигайте `n` вправо на 7; затем положите последний
байт. Декодирование: для байта номер `i` прибавляйте `(byte & 0x7f) << (7 * i)`; остановитесь, когда старший бит равен 0.
Для `u64` нужно не больше 10 байт.

</details>

<details>
<summary>Ответ</summary>

```rust
/// Appends `n` as a varint (LEB128): 7 bits per byte, lowest bits first,
/// the top bit of each byte says "more bytes follow".
pub fn encode_varint(mut n: u64, out: &mut Vec<u8>) {
    while n >= 0x80 {
        out.push((n as u8 & 0x7f) | 0x80);
        n >>= 7;
    }
    out.push(n as u8);
}

/// Decodes a varint from the start of `bytes`.
/// Returns the number and how many bytes it used, or `None` if the input ends too early or is too long.
pub fn decode_varint(bytes: &[u8]) -> Option<(u64, usize)> {
    let mut n: u64 = 0;
    for (i, &byte) in bytes.iter().enumerate().take(10) {
        n |= u64::from(byte & 0x7f) << (7 * i);
        if byte & 0x80 == 0 {
            return Some((n, i + 1));
        }
    }
    None
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn small_numbers_take_one_byte() {
        let mut out = Vec::new();
        encode_varint(5, &mut out);
        assert_eq!(out, [0x05]);
    }

    #[test]
    fn matches_postcard() {
        let mut out = Vec::new();
        encode_varint(1000, &mut out);
        assert_eq!(out, [0xe8, 0x07]);
        assert_eq!(out, postcard::to_allocvec(&1000u32).unwrap());
    }

    #[test]
    fn round_trip() {
        for n in [0, 1, 127, 128, 300, 1 << 35, u64::MAX] {
            let mut out = Vec::new();
            encode_varint(n, &mut out);
            assert_eq!(decode_varint(&out), Some((n, out.len())), "n = {n}");
        }
    }

    #[test]
    fn missing_last_byte_is_none() {
        assert_eq!(decode_varint(&[0xe8]), None);
    }
}
```

`cargo test --lib` со всеми упражнениями этапа на данный момент:

```text
test result: ok. 13 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.02s
```

Почему это работает: `n as u8 & 0x7f` оставляет младшие 7 бит; `| 0x80` отмечает «дальше есть ещё». Декодер ставит
каждую группу из 7 бит на место сдвигом. `take(10)` не даёт повреждённому вводу, где у всех байтов поднят старший бит,
крутиться бесконечно или сдвигать дальше 64 бит.

</details>

## Вопросы для самопроверки

1. Почему движки хранения предпочитают байты `u32` тексту `"1000"`?

<details>
<summary>Ответ</summary>

Фиксированный размер (всегда 4 байта, поэтому смещения можно вычислить без чтения), и декодирование — это копирование,
а не разбор цифр и поиск конца.

</details>

2. Какие байты у значения `u16` 258 (`0x0102`) в little-endian?

<details>
<summary>Ответ</summary>

`[0x02, 0x01]`: младший байт первым.

</details>

3. Какие две проблемы решает префикс длины, которые не решает символ-разделитель (например, `\n`)?

<details>
<summary>Ответ</summary>

Данные могут содержать сам разделитель (значение с переводом строки), а с длиной читатель знает размер до чтения: может
его проверить, выделить память один раз и заметить обрезанную запись.

</details>

4. Последний байт записи перевернулся на диске. Какую ошибку вернёт `decode` и почему не `Truncated`?

<details>
<summary>Ответ</summary>

`BadChecksum`. Все байты на месте, поэтому с длинами всё в порядке; но CRC, вычисленная по байтам, больше не совпадает с
сохранённой.

</details>

5. Может ли CRC-32 защитить файл от злоумышленника, который его редактирует?

<details>
<summary>Ответ</summary>

Нет. Злоумышленник может просто посчитать CRC новых байтов. Нужен MAC с секретным ключом (Урок 10.9).

</details>

6. Почему postcard кодирует 300 двумя байтами и сколько байтов займёт 100?

<details>
<summary>Ответ</summary>

postcard использует varint с 7 битами на байт. Числу 300 нужно 9 бит, поэтому два байта. 100 помещается в 7 бит, поэтому
занимает один байт.

</details>

## Итоги

- Движки хранения держат данные в двоичном виде: поля фиксированного размера, никакого разбора, точные байты.
- Выбирайте порядок байтов для каждого формата файла и записывайте его; в этом курсе — little-endian (`to_le_bytes`,
  `from_le_bytes`).
- Префиксы длины говорят читателю, где кончается каждая запись; максимальная длина защищает от повреждённых длин.
- Контрольная сумма CRC-32 ловит разорванные и повреждённые записи; от злоумышленников она не защищает.
- Декодер проверяет в безопасном порядке и возвращает ошибки (`Truncated`, `TooLarge`, `BadChecksum`), никогда не
  паникует.
- `serde` + `postcard` компактно сериализуют целые структуры, с varint для целых чисел; `bincode` больше не
  поддерживается.
- Настоящие форматы начинаются с магического числа и версии, чтобы старые файлы оставались читаемыми.

## Источники и что почитать

- [Rust primitive type u32](https://doc.rust-lang.org/std/primitive.u32.html) — `to_le_bytes`, `from_le_bytes` и другие
  методы порядка байтов.
- [Документация crc32fast](https://docs.rs/crc32fast/latest/crc32fast/) — `hash` и пошаговый `Hasher`.
- [Документация postcard](https://docs.rs/postcard/latest/postcard/) и
  [формат передачи postcard](https://postcard.jamesmunns.com/wire-format) — в какие именно байты превращается каждый тип
  Rust.
- [serde.rs](https://serde.rs/) — книга по serde: derive, атрибуты и форматы данных.
- [LevelDB log format](https://github.com/google/leveldb/blob/main/doc/log_format.md) — формат записей настоящего движка
  хранения с контрольными суммами и длинами, близкий к нашему.
- [SQLite database file format](https://www.sqlite.org/fileformat.html) — полный, хорошо документированный двоичный
  формат файла, начиная с его магического заголовка.

---

[🏠 Начало](../START-HERE.md) · [← 17.2 Надёжность записи: fsync и устойчивость к сбоям](02-durability-and-fsync.md) · [📋 Этап 17](README.md) · [17.4 Журнал упреждающей записи (WAL) →](04-write-ahead-log.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
