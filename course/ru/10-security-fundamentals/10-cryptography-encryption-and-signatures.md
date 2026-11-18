<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 10.9 Криптография I: хеши, MAC и случайность](09-cryptography-hashes-macs-and-randomness.md) · [📋 Этап 10](README.md) · [10.11 TLS и сертификаты →](11-tls-and-certificates.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/10-security-fundamentals/10-cryptography-encryption-and-signatures.md) <!-- nav -->

# Урок 10.10 — Криптография II: шифрование и подписи

> **Этап 10 — Основы безопасности** · Урок 10 из 15 · ⏱ около 135 минут

## Что вы узнаете

- Чем отличается симметричное шифрование (один общий ключ) от асимметричной криптографии (пара ключей).
- Что такое AEAD (аутентифицированное шифрование) и почему нельзя использовать шифрование без аутентификации.
- Почему nonce никогда не должен повторяться с одним и тем же ключом и что будет, если он повторится.
- Как шифровать и расшифровывать с XChaCha20-Poly1305 и как хранить nonce рядом с шифртекстом.
- Как подписывать данные с Ed25519 и проверять подпись открытым ключом.

## Перед началом

- Вы прошли [Урок 10.9 — Криптография I: хеши, MAC и случайность](09-cryptography-hashes-macs-and-randomness.md). Всё здесь опирается на ключи, MAC (message authentication code, коды аутентификации сообщений) и безопасные случайные байты.
- Вы знаете `Result`, `Option` и тесты ([Урок 04.10](../04-structs-enums-collections-errors/10-error-handling.md), [Урок 05.6](../05-rust-intermediate/06-testing-in-depth.md)).
- Создайте учебный пакет (примеры лежат в `src/bin/`, как в прошлом уроке):

```bash
cd ~/projects/rust
cargo new code/p10-encryption
cd code/p10-encryption
cargo add chacha20poly1305@0.11 aes-gcm@0.11 ed25519-dalek@3 getrandom@0.4 hex@0.4
mkdir src/bin
```

## Новые слова

| Слово | Значение |
|---|---|
| **открытый текст / шифртекст** (plaintext / ciphertext) | Читаемые данные до шифрования / нечитаемый результат после шифрования. |
| **шифрование / расшифрование** (encryption / decryption) | Превращение открытого текста в шифртекст с помощью ключа / обратное превращение с правильным ключом. |
| **симметричное шифрование** (symmetric encryption) | Шифрование, где **один и тот же** секретный ключ шифрует и расшифровывает. |
| **асимметричная криптография** (asymmetric cryptography) | Криптография с **парой ключей**: *открытый ключ*, который может знать любой, и *закрытый ключ*, который остаётся секретным. |
| **шифр** (cipher) | Алгоритм шифрования, например ChaCha20 или AES (Advanced Encryption Standard, расширенный стандарт шифрования). |
| **AEAD** (Authenticated Encryption with Associated Data, аутентифицированное шифрование с присоединёнными данными) | Шифрование, которое дополнительно создаёт метку аутентификации. Оно скрывает данные **и** обнаруживает любое изменение. |
| **nonce** («number used once», одноразовое число) | Значение, которое должно быть уникальным для каждого сообщения, зашифрованного одним ключом. Оно не секретно. |
| **метка аутентификации** (authentication tag) | 16-байтовое контрольное значение, которое AEAD добавляет к шифртексту. Расшифрование не удаётся, если данные или метка изменились. |
| **присоединённые данные** (associated data, AAD) | Данные, которые не шифруются, но покрыты меткой, например заголовок. |
| **цифровая подпись** (digital signature) | Значение, созданное закрытым ключом, которое любой может проверить подходящим открытым ключом. Оно доказывает, кто подписал и что данные не менялись. |
| **Ed25519** | Современный быстрый алгоритм подписи. Ключи по 32 байта, подписи по 64 байта. |
| **KDF** (Key Derivation Function, функция формирования ключа) | Функция, превращающая пароль в ключ нужной длины. Для паролей используют медленные, например Argon2. |

## 1. Два семейства

**Симметричная** криптография использует один секретный ключ в обе стороны, как ключ от дома, который и запирает, и отпирает одну и ту же дверь.
Она быстрая и применяется к самим данным. Её проблема — ключ: он должен быть у обеих сторон и больше ни у кого.

**Асимметричная** криптография использует пару ключей. То, что делает закрытый ключ, проверит только открытый (подписи), а
то, что делает открытый ключ, отменит только закрытый (шифрование открытым ключом и обмен ключами). Открытый ключ можно опубликовать на весь мир.

| | Симметричная | Асимметричная |
|---|---|---|
| Ключи | один общий секрет | открытый + закрытый |
| Скорость | очень быстрая | медленнее |
| Типичное применение | шифрование файлов, баз данных, сетевого трафика | подписи, согласование общего ключа (TLS, Transport Layer Security, протокол защиты транспортного уровня, — следующий урок) |
| Пример в этом уроке | XChaCha20-Poly1305 | Ed25519 |

Настоящие системы сочетают обе: асимметричная криптография подтверждает личность и согласует ключ, а затем симметричное шифрование защищает данные.

> 📏 **Соглашение:** используйте только **AEAD**-шифры: XChaCha20-Poly1305, ChaCha20-Poly1305 или AES-256-GCM (AES с 256-битным ключом в режиме GCM,
> Galois/Counter Mode, режим Галуа/счётчика). «Сырой» шифр (один AES или старый *режим* — способ применить блочный шифр к длинным данным — вроде
> ECB, Electronic Codebook, режима электронной кодовой книги, или CBC, Cipher Block Chaining, режима сцепления блоков, без MAC) скрывает данные, но не обнаруживает изменений, и им легко воспользоваться неправильно. Злоумышленник, который может переворачивать биты в вашем шифртексте, часто может перевернуть соответствующие биты в открытом тексте.

## 2. Шифрование с XChaCha20-Poly1305

AEAD-шифр получает четыре входа и выдаёт один результат:

```text
  key (32 bytes, secret) ──┐
  nonce (24 bytes)        ─┤
  plaintext               ─┼──►  encrypt  ──►  ciphertext + tag (16 bytes)
  associated data (opt.)  ─┘

  decrypt(key, nonce, ciphertext + tag)  =  plaintext   or   Error  (never garbage)
```

Создайте `src/bin/encrypt.rs`:

```rust
use chacha20poly1305::aead::{Aead, Generate, Key, KeyInit};
use chacha20poly1305::{XChaCha20Poly1305, XNonce};

fn main() {
    // 1. Случайный 32-байтовый ключ и случайный 24-байтовый nonce (от операционной системы).
    let key = Key::<XChaCha20Poly1305>::generate();
    let nonce = XNonce::generate();
    let cipher = XChaCha20Poly1305::new(&key);

    // 2. Шифруем.
    let plaintext = b"meet me at noon";
    let ciphertext = cipher.encrypt(&nonce, plaintext.as_ref()).unwrap();
    println!("plaintext  : {} bytes", plaintext.len());
    println!("ciphertext : {} bytes = {}", ciphertext.len(), hex::encode(&ciphertext));

    // 3. Расшифровываем с правильными ключом и nonce.
    let back = cipher.decrypt(&nonce, ciphertext.as_ref()).unwrap();
    println!("decrypted  : {}", String::from_utf8(back).unwrap());

    // 4. Переворачиваем один бит шифртекста: расшифрование не удаётся, ничего не возвращается.
    let mut tampered = ciphertext.clone();
    tampered[0] ^= 1;
    println!("tampered   : {:?}", cipher.decrypt(&nonce, tampered.as_ref()));

    // 5. Неверный ключ: тоже ошибка.
    let other = XChaCha20Poly1305::new(&Key::<XChaCha20Poly1305>::generate());
    println!("wrong key  : {:?}", other.decrypt(&nonce, ciphertext.as_ref()));
}
```

**Что происходит:**

1. `Key::<XChaCha20Poly1305>::generate()` создаёт случайный ключ нужной длины (32 байта), а `XNonce::generate()` — случайный 24-байтовый nonce. Оба используют источник случайности операционной системы из Урока 10.9. Трейт `Generate` даёт `generate`; `KeyInit` даёт `new`; `Aead` даёт `encrypt` и `decrypt`. Все три нужно импортировать.
2. `encrypt` возвращает `Vec<u8>`: шифртекст, за которым идёт 16-байтовая метка аутентификации. Поэтому 15 байт открытого текста стали 31 байтом.
3. `decrypt` с теми же ключом и nonce возвращает исходные байты.
4. Один перевёрнутый бит ломает проверку метки. Вы получаете `Err`, а **не** испорченный открытый текст. Это и есть «аутентифицированная» часть AEAD.
5. Неверный ключ приводит к ошибке так же. Ошибка намеренно пустая: она не говорит, что именно неверно — ключ, nonce или данные.

```text
plaintext  : 15 bytes
ciphertext : 31 bytes = fc60277b93f988d3faef3d95172669cef1919c30a9beaae837cb1fb0915ae4
decrypted  : meet me at noon
tampered   : Err(Error)
wrong key  : Err(Error)
```

Hex шифртекста и ключ меняются при каждом запуске (они случайные); размеры и результаты — нет.

> 🔒 **Безопасность:** nonce **не секретен**: его хранят или отправляют рядом с шифртекстом. Секрет — это ключ.

## 3. Правило nonce

Для каждого ключа каждому сообщению нужен nonce, который **никогда раньше** не использовался с этим ключом. Если зашифровать два сообщения одним
ключом и одним nonce, злоумышленник, увидевший оба шифртекста, узнает многое. Создайте `src/bin/nonce_reuse.rs`, чтобы это увидеть (здесь
ваш собственный ключ и ваши собственные сообщения, поэтому опыт безопасен):

```rust
use chacha20poly1305::aead::{Aead, Generate, Key, KeyInit};
use chacha20poly1305::{ChaCha20Poly1305, Nonce};

fn main() {
    let key = Key::<ChaCha20Poly1305>::generate();
    let cipher = ChaCha20Poly1305::new(&key);
    let nonce = Nonce::generate();

    // ОШИБКА: два разных сообщения зашифрованы одним ключом И одним nonce.
    let m1 = b"attack at dawn!!";
    let m2 = b"retreat at noon!";
    let c1 = cipher.encrypt(&nonce, m1.as_ref()).unwrap();
    let c2 = cipher.encrypt(&nonce, m2.as_ref()).unwrap();

    // Подслушивающий видит только c1 и c2 (без 16-байтовой метки в конце)
    // и считает их XOR. Секретная гамма сокращается.
    let xor_of_ciphertexts: Vec<u8> = c1.iter().zip(&c2).take(m1.len()).map(|(a, b)| a ^ b).collect();
    let xor_of_plaintexts: Vec<u8> = m1.iter().zip(m2).map(|(a, b)| a ^ b).collect();
    println!("c1 xor c2 = {}", hex::encode(&xor_of_ciphertexts));
    println!("m1 xor m2 = {}", hex::encode(&xor_of_plaintexts));
    println!("equal: {}", xor_of_ciphertexts == xor_of_plaintexts);
}
```

**Что происходит:**

1. ChaCha20 — *потоковый шифр*: из ключа и nonce он делает длинную секретную «гамму» (keystream) и объединяет с открытым текстом операцией XOR (исключающее «или», битовая операция, результат которой равен 1 там, где входные биты различаются): `c = m ^ keystream`.
2. При одинаковых ключе и nonce оба сообщения используют **одну и ту же** гамму. XOR двух шифртекстов равен `(m1 ^ k) ^ (m2 ^ k) = m1 ^ m2`: гамма исчезает.
3. Теперь у атакующего есть XOR двух открытых текстов, и ключ ему не понадобился. Догадками и известными шаблонами текста (например, `attack at`) оба сообщения обычно удаётся восстановить.

```text
c1 xor c2 = 13110013060a54411554440f18014f00
m1 xor m2 = 13110013060a54411554440f18014f00
equal: true
```

Как безопасно выбирать nonce:

| Шифр | Ключ | Nonce | Стратегия для nonce |
|---|---|---|---|
| **XChaCha20-Poly1305** | 32 байта | 24 байта | Случайный nonce для каждого сообщения безопасен. **Берите его по умолчанию.** |
| ChaCha20-Poly1305 | 32 байта | 12 байтов | Счётчик или случайный nonce только для ограниченного числа сообщений. |
| AES-256-GCM | 32 байта | 12 байтов | То же ограничение: случайные 12-байтовые nonce безопасны только примерно для 2^32 сообщений на один ключ. Нужна осторожность. |

24-байтовый случайный nonce настолько велик, что два одинаковых значения практически не встретятся, поэтому с XChaCha20-Poly1305 вы просто создаёте новый случайный nonce для каждого шифрования.
(AES-GCM — тоже хороший, широко применяемый AEAD, а процессоры (CPU, Central Processing Unit) с инструкциями AES выполняют его очень быстро. У крейта `aes-gcm` тот же API (Application Programming Interface, программный интерфейс: функции, которые предлагает крейт); см. Упражнение 1.)

## 4. Запечатанная коробка: nonce и шифртекст вместе

Для расшифрования нужен nonce, поэтому его хранят вместе с шифртекстом. Простой распространённый формат — `nonce || ciphertext+tag`, где `||`
значит «за которым идёт»:

```text
 offset:   0                          24
           ┌──────────────────────────┬──────────────────────────────┐
           │ nonce (24 bytes, random) │ ciphertext  +  tag (16 bytes) │
           └──────────────────────────┴──────────────────────────────┘
```

Создайте библиотеку `src/lib.rs` (в пакете могут быть и библиотека, и программы в `src/bin/`):

```rust
use chacha20poly1305::aead::{Aead, Generate, KeyInit};
use chacha20poly1305::{XChaCha20Poly1305, XNonce};

const NONCE_LEN: usize = 24;

/// Шифрует `plaintext`. Результат: `nonce (24 bytes) || ciphertext+tag`.
pub fn seal(key: &[u8; 32], plaintext: &[u8]) -> Vec<u8> {
    let cipher = XChaCha20Poly1305::new_from_slice(key).expect("the key is 32 bytes");
    let nonce = XNonce::generate(); // новый случайный nonce для каждого сообщения
    let mut out = nonce.to_vec();
    out.extend(cipher.encrypt(&nonce, plaintext).expect("encryption cannot fail for in-memory data"));
    out
}

/// Расшифровывает то, что создал `seal`. Возвращает `None`, если данные изменены, обрезаны или ключ неверный.
pub fn open(key: &[u8; 32], sealed: &[u8]) -> Option<Vec<u8>> {
    if sealed.len() < NONCE_LEN {
        return None;
    }
    let (nonce, ciphertext) = sealed.split_at(NONCE_LEN);
    let cipher = XChaCha20Poly1305::new_from_slice(key).expect("the key is 32 bytes");
    let nonce = XNonce::try_from(nonce).ok()?;
    cipher.decrypt(&nonce, ciphertext).ok()
}

#[cfg(test)]
mod tests {
    use super::*;

    const KEY: [u8; 32] = [7; 32];

    #[test]
    fn round_trip() {
        let sealed = seal(&KEY, b"hello");
        assert_eq!(open(&KEY, &sealed).as_deref(), Some(&b"hello"[..]));
    }

    #[test]
    fn same_message_gives_different_output_each_time() {
        assert_ne!(seal(&KEY, b"hello"), seal(&KEY, b"hello"));
    }

    #[test]
    fn any_flipped_bit_is_detected() {
        let sealed = seal(&KEY, b"hello");
        for i in 0..sealed.len() {
            let mut bad = sealed.clone();
            bad[i] ^= 1;
            assert_eq!(open(&KEY, &bad), None, "flip at byte {i} was not detected");
        }
    }

    #[test]
    fn wrong_key_and_truncation_are_rejected() {
        let sealed = seal(&KEY, b"hello");
        assert_eq!(open(&[8; 32], &sealed), None);
        assert_eq!(open(&KEY, &sealed[..10]), None);
        assert_eq!(open(&KEY, &[]), None);
    }
}
```

**Что происходит:**

1. `seal` создаёт новый nonce, начинает результат с байтов nonce и добавляет зашифрованные данные. Функцию, принимающую `&[u8; 32]`, нельзя вызвать с ключом неверной длины: это проверяет тип.
2. `open` написана для **недоверенного ввода**: сначала она проверяет, что есть хотя бы 24 байта (иначе `split_at` запаниковал бы), отделяет nonce и превращает любую неудачу в `None`. На плохих данных она никогда не паникует.
3. `new_from_slice(key).expect(…)` здесь не может завершиться ошибкой, потому что тип массива гарантирует 32 байта.
4. Тесты покрывают важные свойства: круговой путь, разный результат каждый раз (новый nonce), **каждый перевёрнутый байт обнаружен** (включая байты nonce), а неверный ключ и обрезанные данные отвергаются.

Запустите `cargo test`:

```text
running 4 tests
test tests::wrong_key_and_truncation_are_rejected ... ok
test tests::round_trip ... ok
test tests::same_message_gives_different_output_each_time ... ok
test tests::any_flipped_bit_is_detected ... ok

test result: ok. 4 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

> 🧪 **Тестирование:** для каждой функции шифрования проверяйте подмену данных. Функция, которая «возвращается туда и обратно», но принимает изменённые данные, сломана самым важным образом.

### Присоединённые данные

Иногда часть сообщения должна оставаться читаемой, но не должна меняться: заголовок формата файла, номер версии, имя пользователя, которому принадлежит запись.
Это *присоединённые данные*: они не шифруются, но метка их покрывает. Создайте `src/bin/aad.rs`:

```rust
use chacha20poly1305::aead::{Aead, Generate, Key, KeyInit, Payload};
use chacha20poly1305::{XChaCha20Poly1305, XNonce};

fn main() {
    let key = Key::<XChaCha20Poly1305>::generate();
    let cipher = XChaCha20Poly1305::new(&key);
    let nonce = XNonce::generate();

    // Заголовок ("vault v1") НЕ шифруется, но аутентифицируется.
    let header = b"vault v1";
    let sealed = cipher
        .encrypt(&nonce, Payload { msg: b"top secret", aad: header })
        .unwrap();

    let ok = cipher.decrypt(&nonce, Payload { msg: &sealed, aad: header });
    println!("same header : {:?}", ok.map(|v| String::from_utf8(v).unwrap()));

    let bad = cipher.decrypt(&nonce, Payload { msg: &sealed, aad: b"vault v2" });
    println!("other header: {bad:?}");
}
```

```text
same header : Ok("top secret")
other header: Err(Error)
```

**Что происходит:** `Payload { msg, aad }` передаёт сообщение и присоединённые данные вместе. Расшифрование успешно, только если `aad` совпадает байт в байт. Проект 10.14 использует эту идею для защиты заголовка своего файла.

## 5. Откуда берётся ключ?

Ключ шифрования — это 32 **случайных** байта. Есть три честных источника:

1. **Сгенерирован случайно** и надёжно сохранён (хранилище секретов, связка ключей операционной системы, защищённый файл). См. Урок 10.8.
2. **Получен из пароля** медленной KDF вроде Argon2 (Урок 10.6): `key = Argon2id(password, salt)`. Соль случайна, не секретна и хранится рядом с данными. Именно так делает Проект 10.14.
3. **Согласован двумя сторонами** асимметричным обменом ключами (TLS, следующий урок).

Никогда не используйте байты самого пароля как ключ и никогда не используйте `SHA-256(пароль)` как ключ (SHA-256, Secure Hash Algorithm, безопасный алгоритм хеширования на 256 бит): его быстро перебрать, как объяснял Урок 10.9.

> ⚠️ **Частая ошибка:** хранить ключ рядом с шифртекстом («зашифрованная» база с ключом в том же файле настроек). Шифрование лишь переносит проблему на ключ: защищайте ключ как минимум так же хорошо, как данные.

## 6. Подписи с Ed25519

Подпись отвечает на вопрос «кто это создал и менялось ли оно?». Подписывающий использует **закрытый** ключ; все проверяют **открытым**.
Создайте `src/bin/sign.rs`:

```rust
use ed25519_dalek::{Signature, Signer, SigningKey, Verifier, VerifyingKey};

fn main() {
    // 1. Ключ подписи (закрытый) — это 32 случайных байта от операционной системы.
    let mut seed = [0u8; 32];
    getrandom::fill(&mut seed).expect("no OS randomness");
    let signing_key = SigningKey::from_bytes(&seed);
    let verifying_key: VerifyingKey = signing_key.verifying_key();
    println!("private key: {} bytes (keep secret)", signing_key.to_bytes().len());
    println!("public key : {}", hex::encode(verifying_key.to_bytes()));

    // 2. Подписываем сообщение.
    let message = b"release v1.2.0 sha256=2cf24dba";
    let signature: Signature = signing_key.sign(message);
    println!("signature  : {} bytes", signature.to_bytes().len());

    // 3. Любой, у кого есть ОТКРЫТЫЙ ключ, может проверить.
    println!("genuine    : {:?}", verifying_key.verify(message, &signature));

    // 4. Изменённое сообщение не проходит.
    println!("tampered   : {:?}", verifying_key.verify(b"release v1.2.0 sha256=deadbeef", &signature));

    // 5. Другая пара ключей не проходит.
    let mut other_seed = [0u8; 32];
    getrandom::fill(&mut other_seed).unwrap();
    let other_public = SigningKey::from_bytes(&other_seed).verifying_key();
    println!("wrong key  : {:?}", other_public.verify(message, &signature));
}
```

**Что происходит:**

1. Закрытый ключ Ed25519 — это просто 32 случайных байта. `SigningKey::from_bytes` строит из них ключ подписи, а `verifying_key()` вычисляет парный 32-байтовый открытый ключ.
2. `sign` возвращает 64-байтовую `Signature`. Трейт `Signer` даёт `sign`, а `Verifier` — `verify`.
3. `verify` возвращает `Ok(())` для настоящего сообщения и ошибку для изменённого сообщения или для другого открытого ключа.

```text
private key: 32 bytes (keep secret)
public key : 41f320cdb76504b67fcd037d72544c5afac68ec61a4b4b0ea83c4b739c0a7c40
signature  : 64 bytes
genuine    : Ok(())
tampered   : Err(signature::Error {})
wrong key  : Err(signature::Error {})
```

(Открытый ключ при каждом запуске другой.) Сравните с MAC из Урока 10.9:

| | MAC (HMAC) | Подпись (Ed25519) |
|---|---|---|
| Ключ | один общий секрет | закрытый ключ подписывает, открытый проверяет |
| Кто может создать верную метку | каждый, у кого есть ключ (поэтому любой из них мог бы подделать) | только владелец закрытого ключа |
| Кто может проверить | каждый, у кого есть ключ | **любой**, у кого есть открытый ключ |
| Для чего | сообщения между двумя сторонами, доверяющими друг другу | релизы программ, сертификаты, документы, токены, которые проверяют многие сервисы |

Остаётся вопрос: как узнать, что открытый ключ действительно принадлежит тому, кому вы думаете? Если злоумышленник подменит открытый ключ, ваша
проверка бесполезна. Ответ — *сертификаты*: доверенная сторона подписывает «этот открытый ключ принадлежит этому имени». Следующий урок показывает, как это работает в TLS.

> 🔒 **Безопасность:** закрытый ключ подписи — самый чувствительный из ваших файлов. Никогда не коммитьте его в Git (Этап 01), не пишите в лог и не держите
> в исходном коде. Публикуйте только открытый ключ. Урок 10.8 рассказывает о хранении секретов, а Урок 10.13 показывает, как контрольные суммы в `Cargo.lock` и база уязвимостей защищают крейты, которые вы скачиваете.

## Частые ошибки

**1. API из старого руководства.** Во многих примерах вызывается `XChaCha20Poly1305::generate_nonce(&mut OsRng)`. В `chacha20poly1305` 0.11 такой функции больше нет:

```text
error[E0599]: no associated function or constant named `generate_nonce` found for struct `ChaChaPoly1305<C, N>` in the current scope
 --> src/bin/encrypt.rs:7:36
  |
7 |     let nonce = XChaCha20Poly1305::generate_nonce().expect("no OS randomness");
  |                                    ^^^^^^^^^^^^^^ associated function or constant not found in `ChaChaPoly1305<…>`
```

Исправление: импортируйте трейт `Generate` и напишите `XNonce::generate()`. Если код из блога не компилируется, откройте документацию для **точной версии** из `Cargo.toml`.

**2. Повторное использование nonce** (константа вроде `[0u8; 24]` или один nonce на весь файл). Код компилируется и «работает», а шифрование ломается (раздел 3).

**3. Шифрование без аутентификации или игнорирование ошибки `decrypt`.** `decrypt(...).unwrap_or_default()` превращает атаку в пустой файл. Считайте `Err` признаком того, что данным нельзя доверять.

**4. Вера в то, что шифрование доказывает личность.** Любой, у кого есть ключ, может создать верный шифртекст. Если нужно знать, *кто* это написал, подписывайте.

**5. Собственная смесь примитивов** («я сделаю XOR с SHA-256 от ключа»). Используйте AEAD API как есть.

## Попробуйте сами

1. Запустите `cargo run --bin encrypt` три раза. Какие значения меняются, а какие остаются прежними?
2. В `encrypt.rs` переверните бит в **последнем** байте шифртекста вместо первого. Будет ли это обнаружено? (Последние 16 байтов — это метка.)
3. В `sign.rs` измените один байт подписи перед проверкой: `let mut bytes = signature.to_bytes(); bytes[0] ^= 1; let bad = Signature::from_bytes(&bytes);`.

## Упражнения

### Упражнение 1 — Переход на AES-256-GCM (легко)

Напишите `src/bin/aes.rs`, который шифрует и расшифровывает `meet me at noon` через `Aes256Gcm`. Выведите длину nonce и длину шифртекста.

<details>
<summary>Подсказка</summary>

У крейта `aes-gcm` те же трейты: `use aes_gcm::aead::{Aead, Generate, Key, KeyInit};` и `use aes_gcm::{Aes256Gcm, Nonce};`.

</details>

<details>
<summary>Ответ</summary>

```rust
use aes_gcm::aead::{Aead, Generate, Key, KeyInit};
use aes_gcm::{Aes256Gcm, Nonce};

fn main() {
    let key = Key::<Aes256Gcm>::generate();
    let cipher = Aes256Gcm::new(&key);
    let nonce = Nonce::generate(); // 12 bytes for AES-GCM
    let ct = cipher.encrypt(&nonce, b"meet me at noon".as_ref()).unwrap();
    println!("nonce {} bytes, ciphertext {} bytes", nonce.len(), ct.len());
    let pt = cipher.decrypt(&nonce, ct.as_ref()).unwrap();
    println!("{}", String::from_utf8(pt).unwrap());
}
```

```text
nonce 12 bytes, ciphertext 31 bytes
meet me at noon
```

Почему это работает: все AEAD-шифры семейства RustCrypto используют общие трейты `aead`, поэтому смена алгоритма меняет только имена типов. Nonce — 12 байтов, метка добавляет 16 байтов (15 + 16 = 31).

</details>

### Упражнение 2 — Привязать заголовок к запечатанной коробке (средне)

Добавьте в `src/lib.rs` функции `seal_with_header(key, header, plaintext)` и `open_with_header(key, header, sealed)`. Они работают как `seal` и `open`, но передают заголовок как присоединённые данные. Напишите тест, который проверяет, что открытие с другим заголовком не удаётся.

<details>
<summary>Подсказка</summary>

Соберите `Payload { msg: plaintext, aad: header }` для шифрования и `Payload { msg: ciphertext, aad: header }` для расшифрования. Импортируйте `Payload` из `chacha20poly1305::aead`.

</details>

<details>
<summary>Ответ</summary>

Замените первую строку на `use chacha20poly1305::aead::{Aead, Generate, KeyInit, Payload};` и добавьте:

```rust
/// Like `seal`, but also binds a plain-text `header` to the message.
pub fn seal_with_header(key: &[u8; 32], header: &[u8], plaintext: &[u8]) -> Vec<u8> {
    let cipher = XChaCha20Poly1305::new_from_slice(key).expect("the key is 32 bytes");
    let nonce = XNonce::generate();
    let mut out = nonce.to_vec();
    let payload = Payload { msg: plaintext, aad: header };
    out.extend(cipher.encrypt(&nonce, payload).expect("encryption cannot fail for in-memory data"));
    out
}

/// Opens data made by `seal_with_header`; fails if the header is not exactly the same.
pub fn open_with_header(key: &[u8; 32], header: &[u8], sealed: &[u8]) -> Option<Vec<u8>> {
    if sealed.len() < NONCE_LEN {
        return None;
    }
    let (nonce, ciphertext) = sealed.split_at(NONCE_LEN);
    let cipher = XChaCha20Poly1305::new_from_slice(key).expect("the key is 32 bytes");
    let nonce = XNonce::try_from(nonce).ok()?;
    cipher.decrypt(&nonce, Payload { msg: ciphertext, aad: header }).ok()
}
```

И внутри `mod tests`:

```rust
    #[test]
    fn header_is_authenticated() {
        let sealed = seal_with_header(&KEY, b"vault v1", b"hello");
        assert_eq!(open_with_header(&KEY, b"vault v1", &sealed).as_deref(), Some(&b"hello"[..]));
        assert_eq!(open_with_header(&KEY, b"vault v2", &sealed), None);
    }
```

Теперь `cargo test` показывает 5 успешных тестов.

Почему это работает: метка вычисляется по шифртексту **и** присоединённым данным, поэтому другой заголовок даёт другую ожидаемую метку.

</details>

### Упражнение 3 — Проверка релиза (сложно)

В новом библиотечном пакете напишите `pub fn verify_release(public_key: &[u8; 32], data: &[u8], signature: &[u8; 64]) -> bool`, которая возвращает `true` только для верной
подписи Ed25519. Все три входа — недоверенные байты, поэтому функция никогда не должна паниковать. Тесты: настоящий релиз, изменённые данные, изменённая подпись, чужой открытый ключ и
недопустимый открытый ключ (`[0xff; 32]`).

<details>
<summary>Подсказка</summary>

`VerifyingKey::from_bytes(public_key)` возвращает `Result` (не любые 32 байта — допустимый ключ). `Signature::from_bytes(signature)` принимает `&[u8; 64]`. `getrandom` нужен только в тестах: `cargo add --dev getrandom@0.4`.

</details>

<details>
<summary>Ответ</summary>

```rust
use ed25519_dalek::{Signature, Verifier, VerifyingKey};

pub fn verify_release(public_key: &[u8; 32], data: &[u8], signature: &[u8; 64]) -> bool {
    let Ok(key) = VerifyingKey::from_bytes(public_key) else {
        return false;
    };
    let signature = Signature::from_bytes(signature);
    key.verify(data, &signature).is_ok()
}

#[cfg(test)]
mod tests {
    use super::*;
    use ed25519_dalek::{Signer, SigningKey};

    fn key_pair() -> SigningKey {
        let mut seed = [0u8; 32];
        getrandom::fill(&mut seed).unwrap();
        SigningKey::from_bytes(&seed)
    }

    #[test]
    fn genuine_release_verifies() {
        let sk = key_pair();
        let sig = sk.sign(b"app-1.0.tar.gz bytes");
        assert!(verify_release(&sk.verifying_key().to_bytes(), b"app-1.0.tar.gz bytes", &sig.to_bytes()));
    }

    #[test]
    fn changed_data_changed_signature_and_wrong_key_fail() {
        let sk = key_pair();
        let pk = sk.verifying_key().to_bytes();
        let sig = sk.sign(b"original").to_bytes();
        assert!(!verify_release(&pk, b"modified", &sig));

        let mut bad_sig = sig;
        bad_sig[0] ^= 1;
        assert!(!verify_release(&pk, b"original", &bad_sig));

        let other = key_pair().verifying_key().to_bytes();
        assert!(!verify_release(&other, b"original", &sig));
    }

    #[test]
    fn garbage_public_key_does_not_panic() {
        assert!(!verify_release(&[0xff; 32], b"x", &[0; 64]));
    }
}
```

`cargo test` показывает 3 успешных теста.

Почему это работает: `let … else` превращает ошибку разбора в ранний `false`. Любая неудача заканчивается `false`, и ни одна не паникует — так и должна вести себя функция, обрабатывающая враждебный ввод.

</details>

## Вопросы для самопроверки

1. Чем MAC отличается от AEAD?

<details>
<summary>Ответ</summary>

MAC только подтверждает подлинность сообщения (сообщение остаётся читаемым). AEAD шифрует данные **и** подтверждает их подлинность (а при желании и дополнительных присоединённых данных) одной операцией.

</details>

2. Что будет, если зашифровать два сообщения одним ключом и одним nonce?

<details>
<summary>Ответ</summary>

Одна и та же гамма используется дважды, поэтому XOR двух шифртекстов равен XOR двух открытых текстов. Это раскрывает информацию и часто позволяет восстановить оба сообщения. Всегда используйте новый nonce.

</details>

3. Секретен ли nonce? Где его хранить?

<details>
<summary>Ответ</summary>

Нет, он не секретен. Храните его рядом с шифртекстом, например перед ним (`nonce || ciphertext+tag`).

</details>

4. `decrypt` вернул `Err`. Что должна делать программа?

<details>
<summary>Ответ</summary>

Считать данные недоверенными и сообщить понятную ошибку (неверный ключ либо данные повреждены или подменены). Никогда не продолжать с данными по умолчанию или частичными данными.

</details>

5. Кто может создать верную подпись Ed25519 и кто может её проверить?

<details>
<summary>Ответ</summary>

Создать может только владелец закрытого ключа. Проверить может любой, у кого есть открытый ключ.

</details>

6. Почему `SHA-256(пароль)` — плохой ключ шифрования?

<details>
<summary>Ответ</summary>

Он быстро вычисляется, поэтому злоумышленник может пробовать миллиарды паролей в секунду. Получайте ключи из паролей медленной функцией с солью, например Argon2 (Уроки 10.6 и 10.14).

</details>

## Итоги

- Симметричное шифрование использует один общий ключ и быстро; асимметричная криптография использует пару ключей и применяется для подписей и согласования ключа.
- Всегда используйте AEAD (XChaCha20-Poly1305, ChaCha20-Poly1305 или AES-256-GCM): оно скрывает данные и обнаруживает любое изменение.
- Nonce открыт, но никогда не должен повторяться с одним ключом. XChaCha20-Poly1305 позволяет брать новый случайный 24-байтовый nonce для каждого сообщения.
- Храните nonce вместе с шифртекстом (`nonce || ciphertext+tag`). Присоединённые данные позволяют аутентифицировать заголовок, не шифруя его.
- Функции расшифрования получают недоверенный ввод: проверяйте длины, не паникуйте, возвращайте ошибку и тестируйте подмену.
- Ключи — это случайные байты. Если они получены из пароля, используйте медленную KDF (Argon2) со случайной солью.
- Ed25519 подписывает закрытым ключом и проверяет открытым. Чтобы доверять открытому ключу, нужны сертификаты (следующий урок).

## Источники и что почитать

- [Документация крейта chacha20poly1305](https://docs.rs/chacha20poly1305/0.11.0/chacha20poly1305/) — API AEAD, использованный в этом уроке.
- [Документация крейта aes-gcm](https://docs.rs/aes-gcm/0.11.1/aes_gcm/) — AES-256-GCM с теми же трейтами.
- [Документация крейта ed25519-dalek](https://docs.rs/ed25519-dalek/3.0.0/ed25519_dalek/) — ключи, подпись и проверка Ed25519.
- [RFC 8439 — ChaCha20 and Poly1305 for IETF Protocols](https://datatracker.ietf.org/doc/html/rfc8439) — спецификация ChaCha20-Poly1305 (RFC, Request for Comments, «запрос комментариев», — документ интернет-стандарта; IETF, Internet Engineering Task Force, Инженерный совет интернета, — группа, которая пишет эти стандарты).
- [RFC 8032 — Edwards-Curve Digital Signature Algorithm (EdDSA)](https://datatracker.ietf.org/doc/html/rfc8032) — спецификация Ed25519.
- [OWASP Cryptographic Storage Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Cryptographic_Storage_Cheat_Sheet.html) — какие алгоритмы и практики работы с ключами выбирать.

---

[🏠 Начало](../START-HERE.md) · [← 10.9 Криптография I: хеши, MAC и случайность](09-cryptography-hashes-macs-and-randomness.md) · [📋 Этап 10](README.md) · [10.11 TLS и сертификаты →](11-tls-and-certificates.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
