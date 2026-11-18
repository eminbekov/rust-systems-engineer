<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 04.8 Строки подробно](08-strings-in-depth.md) · [📋 Этап 04](README.md) · [04.10 Обработка ошибок →](10-error-handling.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/04-structs-enums-collections-errors/09-hash-maps.md) <!-- nav -->

# Урок 04.9 — Хеш-таблицы (HashMap)

> **Этап 04 — Структуры, перечисления, коллекции и ошибки** · Урок 9 из 12 · ⏱ около 4 часов (2 занятия)

## Что вы узнаете

- Что такое хеш-таблица и когда выбирать её вместо вектора.
- Как добавлять, читать, обновлять и удалять записи и как пользоваться API `entry`.
- Почему порядок обхода `HashMap` случаен и как получить отсортированный результат.
- Как работает владение для ключей и значений.
- Что делает тип пригодным для роли ключа и когда вместо этого использовать `BTreeMap` или `HashSet`.

## Перед началом

- Вы прошли [Урок 04.8 — Строки подробно](08-strings-in-depth.md).
- Вы знаете векторы ([Урок 04.7](07-vectors.md)), `Option` и `match` ([Урок 04.4](04-option-and-match.md)) и то, как `use` вводит имя в область видимости ([Урок 04.6](06-modules-crates-and-packages.md)).
- Создайте проект:

```bash
cd ~/projects/rust
cargo new code/p04-hash-maps
cd code/p04-hash-maps
```

## Новые слова

| Слово | Значение |
|---|---|
| **хеш-таблица** (hash map) | Коллекция пар **ключ → значение**. Вы даёте ключ — получаете значение, сохранённое для него. В Rust: `HashMap<K, V>`. |
| **ключ** (key) | Метка, по которой вы находите значение. Каждый ключ встречается в таблице не более одного раза. |
| **значение** (value) | Данные, хранящиеся под ключом. |
| **запись** (entry) | Один ключ вместе с его значением. Так же называется особый тип Rust, который представляет «место для этого ключа, которое может быть пустым». |
| **хеш-функция** (hash function) | Функция, которая превращает любое значение в число. Одно и то же значение всегда даёт одно и то же число. |
| **хеш** (hash) | Число, которое выдаёт хеш-функция. |
| **корзина** (bucket) | Одна ячейка во внутренней таблице хеш-таблицы, где хранятся записи с данным хешем. |
| **коллизия** (collision) | Два разных ключа, попавшие в одну корзину. Таблица справляется с этим сама. |
| **трейт** (trait) | Именованный набор возможностей, которые может иметь тип, например «можно сравнивать» или «можно хешировать». Полный урок: [Урок 05.2 — Трейты](../05-rust-intermediate/02-traits.md). |
| **`derive`** | Атрибут `#[derive(...)]` просит компилятор автоматически написать код трейта. `#[derive(Debug)]` вы уже использовали. |
| **множество** (set) | Коллекция уникальных значений без повторов. В Rust: `HashSet<T>` и `BTreeSet<T>`. |
| **DoS** (Denial of Service, отказ в обслуживании) | Атака, из-за которой программа становится слишком медленной или занятой, чтобы обслуживать настоящих пользователей. |
| **API** (Application Programming Interface, программный интерфейс приложения) | Набор функций, типов и методов, которые библиотека или тип предлагает другому коду. «API entry» — это группа методов, начинающихся с `entry`. |
| **`NaN`** (Not a Number, «не число») | Особое значение с плавающей точкой для результата, который не является числом, например `0.0 / 0.0`. Оно не равно ничему, даже самому себе. |

## 1. Что такое хеш-таблица?

Вектор находит элемент по его **позиции** (`v[2]`). Часто у вас есть не позиция, а **имя**, и
нужен номер телефона; **слово**, и нужно его количество; **идентификатор пользователя**, и
нужен пользователь.

**Хеш-таблица** хранит пары: **ключ** и **значение**. Вы спрашиваете по ключу и получаете
значение. Представьте гардероб: вы отдаёте пальто и получаете номерок (ключ). Потом показываете
номерок и сразу получаете пальто (значение), и никто не перебирает все пальто.

```text
key        value
"alice"  → "+998-90-111-22-33"
"bob"    → "+998-91-444-55-66"
"carol"  → "+998-93-777-88-99"
```

Таблица быстрая: поиск значения занимает примерно одинаковое время, есть в ней 10 записей или
10 миллионов. Вектору пришлось бы сравнивать со всеми элементами по очереди.

> `HashMap` нет в *прелюдии* (prelude — небольшом наборе имён, которые Rust подключает
> автоматически). Его нужно импортировать в начале файла: `use std::collections::HashMap;`.

## 2. Создание и использование таблицы

```rust
use std::collections::HashMap;

fn main() {
    let mut ages: HashMap<String, u32> = HashMap::new();
    ages.insert(String::from("Ali"), 31);
    ages.insert(String::from("Sara"), 27);
    ages.insert(String::from("Vali"), 45);

    println!("size: {}", ages.len());
    println!("Ali: {:?}", ages.get("Ali"));
    println!("Zoe: {:?}", ages.get("Zoe"));
    println!("has Sara? {}", ages.contains_key("Sara"));

    let old = ages.insert(String::from("Ali"), 32);
    println!("old value: {:?}", old);
    println!("Ali now: {:?}", ages.get("Ali"));

    let removed = ages.remove("Vali");
    println!("removed: {:?}", removed);
    println!("size: {}", ages.len());

    let langs = HashMap::from([("rust", 2015), ("python", 1991), ("c", 1972)]);
    println!("c appeared in {}", langs["c"]);
}
```

**Что происходит:**

1. `HashMap<String, u32>` означает: ключи — `String`, значения — `u32`. Два типа записываются в
   угловых скобках, сначала ключ.
2. `insert(key, value)` добавляет запись. Ключи и значения типа `String` **перемещаются** в
   таблицу (раздел 5).
3. `get("Ali")` ищет ключ и возвращает `Option<&V>`: `Some(&31)` или `None`, если ключа нет.
   Искать ключ `String` можно по `&str`, поэтому не нужно создавать `String` только ради поиска.
4. `contains_key` отвечает «да» или «нет». `len` возвращает число записей.
5. `insert` с уже существующим ключом **заменяет** значение и возвращает старое как `Some(old)`.
   Для нового ключа возвращается `None`.
6. `remove("Vali")` удаляет запись и возвращает удалённое значение в `Option`.
7. `HashMap::from([...])` строит таблицу из массива пар `(ключ, значение)`. Здесь Rust сам
   выводит `HashMap<&str, i32>`. `langs["c"]` читает значение через квадратные скобки.

```text
size: 3
Ali: Some(31)
Zoe: None
has Sara? true
old value: Some(31)
Ali now: Some(32)
removed: Some(45)
size: 2
c appeared in 1972
```

### `get` или `[]`?

Как и у вектора, у таблицы есть два способа чтения. `map[key]` короче, но если ключа нет,
программа **паникует**:

```rust
use std::collections::HashMap;

fn main() {
    let mut m: HashMap<String, i32> = HashMap::new();
    m.insert(String::from("a"), 1);
    println!("{}", m["missing"]);
}
```

```text

thread 'main' (3928098) panicked at src/main.rs:6:21:
no entry found for key
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
```

> 📏 **Соглашение:** используйте `get`, когда ключа может не быть (почти всегда, если ключ приходит
> от пользователя или из файла). Используйте `map[key]` только тогда, когда отсутствие ключа было
> бы ошибкой в вашем собственном коде.

## 3. Изменение значений

Чтобы изменить значение на месте, попросите изменяемую ссылку через `get_mut`.

```rust
use std::collections::HashMap;

fn main() {
    let mut stock: HashMap<&str, u32> = HashMap::new();
    stock.insert("apples", 10);

    match stock.get("apples") {
        Some(n) => println!("we have {n} apples"),
        None => println!("no apples"),
    }

    if let Some(n) = stock.get_mut("apples") {
        *n += 5;
    }
    println!("{:?}", stock.get("apples"));

    let pears = stock.get("pears").copied().unwrap_or(0);
    println!("pears: {pears}");
}
```

**Что происходит:**

1. `get("apples")` даёт `Some(&10)`; `n` — ссылка, а `{n}` печатает число за ней.
2. `get_mut("apples")` даёт `Some(&mut 10)`. Запись `*n += 5` меняет хранящееся значение
   ([Урок 03.5](../03-ownership-and-borrowing/05-mutable-references-and-the-borrow-rules.md)). `if let` выполняет блок, только если ключ существует ([Урок 04.5](05-if-let-let-else-and-patterns.md)).
3. `get("pears")` — это `None`. `.copied()` превращает `Option<&u32>` в `Option<u32>`, а
   `unwrap_or(0)` заменяет `None` на `0`. Это самый короткий способ сказать «значение или
   значение по умолчанию».

```text
we have 10 apples
Some(15)
pears: 0
```

## 4. API `entry`: вставить, если нет, затем обновить

Очень частая задача: «если ключ есть — изменить его значение; если нет — сначала создать». С
`get` и `insert` это занимает несколько строк и два поиска. **API entry** делает это за один. (**API**, Application Programming Interface, — это просто набор методов, которые предлагает тип; API entry — методы вокруг `entry`.)

`map.entry(key)` возвращает *запись* (entry): дескриптор места, где живёт ключ, заполнено оно или
нет. Дальше вы решаете:

- `.or_insert(default)` — если место пусто, сохранить `default`. В любом случае вернуть изменяемую
  ссылку `&mut V` на значение.
- `.or_default()` — то же самое, но значением по умолчанию служит *значение типа по умолчанию*
  (`0` для чисел, пустая `String`, пустой `Vec`). «Значение по умолчанию» берётся из трейта
  `Default`.

**Подсчёт слов** — классический случай:

```rust
use std::collections::HashMap;

fn main() {
    let text = "the cat and the dog and the bird";
    let mut counts: HashMap<String, u32> = HashMap::new();

    for word in text.split_whitespace() {
        let count = counts.entry(word.to_string()).or_insert(0);
        *count += 1;
    }

    let mut words: Vec<&String> = counts.keys().collect();
    words.sort();
    for w in words {
        println!("{w}: {}", counts[w]);
    }
}
```

**Что происходит:**

1. Для каждого слова `entry(word.to_string())` находит место для этого слова.
2. `or_insert(0)` сохраняет `0`, если слово новое, и в любом случае возвращает `&mut u32` на
   значение.
3. `*count += 1` прибавляет единицу через ссылку. Первый раз: `0 → 1`. Второй раз: `1 → 2`.
4. `counts.keys()` — итератор по ключам; `.collect()` кладёт их в `Vec<&String>`
   ([Урок 04.8](08-strings-in-depth.md)). Мы сортируем этот вектор и печатаем по порядку (раздел 6 объясняет зачем).

```text
and: 2
bird: 1
cat: 1
dog: 1
the: 3
```

**Группировка** — хранить список элементов для каждого ключа. `or_default()` в первый раз создаёт
пустой `Vec`:

```rust
use std::collections::HashMap;

fn main() {
    let names = ["Ali", "Anvar", "Bobur", "Alisher", "Bakhtiyor", "Dilnoza"];
    let mut by_letter: HashMap<char, Vec<&str>> = HashMap::new();

    for name in names {
        if let Some(first) = name.chars().next() {
            by_letter.entry(first).or_default().push(name);
        }
    }

    let mut letters: Vec<char> = Vec::new();
    for letter in by_letter.keys() {
        letters.push(*letter);
    }
    letters.sort();

    for letter in letters {
        println!("{letter}: {:?}", by_letter[&letter]);
    }
}
```

**Что происходит:**

1. `by_letter` сопоставляет первой букве вектор имён, которые с неё начинаются.
2. `entry(first).or_default()` возвращает `&mut Vec<&str>` (в первый раз — новый пустой вектор), а
   `.push(name)` добавляет в него имя.
3. Мы копируем ключи в `Vec<char>`, сортируем его и печатаем группы в алфавитном порядке.

```text
A: ["Ali", "Anvar", "Alisher"]
B: ["Bobur", "Bakhtiyor"]
D: ["Dilnoza"]
```

> 💡 **Совет:** когда ловите себя на том, что пишете «если таблица содержит ключ { ... } иначе
> { insert }», используйте `entry`. Это короче, быстрее (один поиск) и меньше способов
> ошибиться.

## 5. Владение ключами и значениями

Вставка подчиняется известным вам правилам владения ([Урок 03.2](../03-ownership-and-borrowing/02-ownership-rules.md)):

- типы, реализующие `Copy` (числа, `char`, `bool`), **копируются** в таблицу;
- типы вроде `String` и `Vec` **перемещаются**: владельцем становится таблица.

```rust
use std::collections::HashMap;

fn main() {
    let mut m = HashMap::new();
    let key = String::from("alice");
    m.insert(key, 1);
    println!("{}", key);
}
```

```text
error[E0382]: borrow of moved value: `key`
 --> src/main.rs:7:20
  |
5 |     let key = String::from("alice");
  |         --- move occurs because `key` has type `String`, which does not implement the `Copy` trait
6 |     m.insert(key, 1);
  |              --- value moved here
7 |     println!("{}", key);
  |                    ^^^ value borrowed here after move
  |
help: consider cloning the value if the performance cost is acceptable
  |
6 |     m.insert(key.clone(), 1);
  |                 ++++++++
```

**Что означает ошибка:** `insert` забрал `String` у `key`. Теперь ею владеет таблица. Если текст
всё ещё нужен, вставьте копию (`key.clone()`) или прочитайте его обратно из таблицы.

В качестве ключей или значений можно хранить и **ссылки** (как `HashMap<&str, u32>` выше), но тогда
данные, на которые они указывают, должны жить не меньше, чем таблица. Это правило — тема
[Урока 05.3 — Времена жизни](../05-rust-intermediate/03-lifetimes.md). В этом этапе хранить собственные `String` — самый простой выбор.

## 6. Обход: порядок случаен

```rust
use std::collections::HashMap;

fn main() {
    let mut m: HashMap<String, i32> = HashMap::new();
    m.insert(String::from("one"), 1);
    m.insert(String::from("two"), 2);
    m.insert(String::from("three"), 3);
    m.insert(String::from("four"), 4);

    println!("--- order is not promised:");
    for (key, value) in &m {
        println!("{key} = {value}");
    }

    println!("--- add 10 to every value:");
    for value in m.values_mut() {
        *value += 10;
    }

    println!("--- sorted by key:");
    let mut keys: Vec<&String> = m.keys().collect();
    keys.sort();
    for k in keys {
        println!("{k} = {}", m[k]);
    }
}
```

**Что происходит:**

1. `for (key, value) in &m` проходит по всем записям. Каждый виток даёт пару ссылок, которую
   образец `(key, value)` раскладывает ([Урок 04.5](05-if-let-let-else-and-patterns.md)).
2. `m.values_mut()` даёт изменяемые ссылки только на значения; есть также `m.keys()` и
   `m.values()`.
3. Чтобы печатать в стабильном порядке, мы собираем ключи в вектор, сортируем его и читаем каждое
   значение через `m[k]`.

```text
--- order is not promised:
two = 2
four = 4
three = 3
one = 1
--- add 10 to every value:
--- sorted by key:
four = 14
one = 11
three = 13
two = 12
```

Первый блок **отличается при каждом запуске**: когда мы запустили ту же программу три раза,
ключи выходили в порядке `four, one, two, three`, затем `one, four, two, three`, затем
`two, four, one, three`. Rust делает порядок случайным намеренно. (У вас порядок будет другим, и
это нормально.)

> ⚠️ **Частая ошибка:** полагаться на порядок `HashMap` (в выводе, в тестах, в файлах). Сегодня
> программа может работать, а завтра печатать записи иначе. Если порядок важен, **сортируйте**
> ключи или используйте `BTreeMap` (раздел 8).

## 7. Как хеш-таблица устроена внутри

Для использования таблицы это знать не обязательно, но так проще понять правила, о которых
дальше пойдёт речь.

```text
key "alice" ──▶ hash function ──▶ 8247019 ──▶ 8247019 mod 4 = 3 ──▶ bucket 3
key "bob"   ──▶ hash function ──▶ 1139284 ──▶ 1139284 mod 4 = 0 ──▶ bucket 0

 bucket 0: ("bob",   "+998-91-444-55-66")
 bucket 1: (empty)
 bucket 2: (empty)
 bucket 3: ("alice", "+998-90-111-22-33")
```

(Числа выдуманы; настоящие хеши — 64-битные числа.) При `insert` таблица пропускает ключ через
**хеш-функцию**, получает число, по числу выбирает **корзину** и сохраняет там запись. При `get`
она повторяет те же шаги и смотрит только в эту корзину. Всю таблицу она не просматривает. Когда
в одной корзине оказывается слишком много записей (**коллизия**) или таблица заполняется,
таблица сама увеличивается, как перераспределяется вектор.

Из такого устройства следуют два правила:

**Правило 1 — ключ должен быть хешируемым и сравнимым.** Типу нужны трейты `Hash` (можно
превратить в число) и `Eq` (можно точно сравнить на равенство). Они есть у `String`, `&str`,
целых чисел, `char`, `bool` и кортежей из них. У чисел с плавающей точкой их **нет**, потому что
`NaN` (Not a Number, «не число» — особое значение для результатов вроде `0.0 / 0.0`) не равно самому себе, так что равенство не точное. Попытка использовать `f64` как ключ не компилируется:

```rust
use std::collections::HashMap;

fn main() {
    let mut prices: HashMap<f64, &str> = HashMap::new();
    prices.insert(1.5, "tea");
}
```

```text
error[E0599]: the method `insert` exists for struct `HashMap<f64, &str>`, but its trait bounds were not satisfied
 --> src/main.rs:5:12
  |
5 |     prices.insert(1.5, "tea");
  |            ^^^^^^
  |
  = note: the following trait bounds were not satisfied:
          `f64: Eq`
          `f64: Hash`
```

Исправление: используйте другой ключ (например, цену в центах как целое число).

**Правило 2 — ключ не должен меняться, пока он лежит в таблице**, иначе изменился бы его хеш, и
таблица не смогла бы его найти. Rust делает это для вас простым: таблица владеет своими ключами и
не позволяет их редактировать.

Ключами могут быть и ваши собственные типы. Попросите компилятор написать два трейта (и
`PartialEq`, на котором строится `Eq`) через `derive`:

```rust
use std::collections::HashMap;

#[derive(Debug, Hash, PartialEq, Eq)]
enum Color {
    Red,
    Green,
}

fn main() {
    let mut grid: HashMap<(i32, i32), char> = HashMap::new();
    grid.insert((0, 0), 'S');
    grid.insert((3, 4), 'E');
    println!("{:?}", grid.get(&(3, 4)));
    println!("{:?}", grid.get(&(1, 1)));

    let mut votes: HashMap<Color, u32> = HashMap::new();
    *votes.entry(Color::Red).or_insert(0) += 1;
    *votes.entry(Color::Red).or_insert(0) += 1;
    *votes.entry(Color::Green).or_insert(0) += 1;
    println!("red: {:?}, green: {:?}", votes.get(&Color::Red), votes.get(&Color::Green));
}
```

**Что происходит:**

1. Кортеж целых чисел `(i32, i32)` работает как ключ «из коробки»: удобный способ хранить сетку
   или карту координат.
2. `#[derive(Hash, PartialEq, Eq)]` даёт `Color` возможности, нужные ключу. Без него вы получите
   такую же ошибку, как для `f64`.
3. `entry(Color::Red).or_insert(0)` и затем `+= 1` считают голоса.

```text
Some('E')
None
red: Some(2), green: Some(1)
```

> 🔒 **Безопасность:** если злоумышленник может выбирать ваши ключи (слова в запросе, имена в
> форме) и находит много ключей с одинаковым хешем, таблица становится медленной, а программу
> можно перегрузить. Это атака **DoS** (Denial of Service, отказ в обслуживании), которая
> называется *hash flooding* («затопление хешами»). Хешер Rust по умолчанию намеренно
> спроектирован так, чтобы ей противостоять: он подмешивает случайный секрет, свой для каждой
> таблицы. Он немного медленнее простых хешеров, и это правильное значение по умолчанию. Не
> заменяйте его, пока не измерили реальную проблему и не поняли риск.

## 8. `BTreeMap` и `HashSet`

**`BTreeMap<K, V>`** — это таблица, которая хранит записи **отсортированными по ключу**. Она тоже
находится в `std::collections`. Ключам нужен трейт `Ord` (можно упорядочить), а не `Hash`.

```rust
use std::collections::BTreeMap;

fn main() {
    let mut m: BTreeMap<&str, i32> = BTreeMap::new();
    m.insert("one", 1);
    m.insert("two", 2);
    m.insert("three", 3);
    m.insert("four", 4);

    for (key, value) in &m {
        println!("{key} = {value}");
    }
    println!("first: {:?}", m.first_key_value());
    println!("last:  {:?}", m.last_key_value());
}
```

**Что происходит:** цикл каждый раз печатает записи в алфавитном порядке ключей; отдельный шаг
сортировки не нужен. `first_key_value()` и `last_key_value()` возвращают наименьший и наибольший
ключ вместе со значением в виде `Option`.

```text
four = 4
one = 1
three = 3
two = 2
first: Some(("four", 4))
last:  Some(("two", 2))
```

| | `HashMap` | `BTreeMap` |
|---|---|---|
| Порядок обхода | случайный | отсортирован по ключу |
| Что нужно ключу | `Hash` + `Eq` | `Ord` |
| Скорость поиска | очень высокая (в среднем постоянное время) | высокая (медленно растёт с размером) |
| Когда использовать | нужно просто искать | нужен отсортированный вывод, «наименьший/наибольший ключ» или диапазоны |

**`HashSet<T>`** — коллекция **уникальных** значений. Это как `HashMap`, но только с ключами.
`insert` возвращает `true`, если значение новое, и `false`, если оно уже было.

```rust
use std::collections::HashSet;

fn main() {
    let mut seen: HashSet<String> = HashSet::new();
    for word in ["a", "b", "a", "c", "b"] {
        let is_new = seen.insert(String::from(word));
        println!("{word}: new? {is_new}");
    }
    println!("unique words: {}", seen.len());
    println!("contains b? {}", seen.contains("b"));
}
```

```text
a: new? true
b: new? true
a: new? false
c: new? true
b: new? false
unique words: 3
contains b? true
```

`BTreeSet<T>` — отсортированная версия множества.

## 9. Тестирование кода с таблицами

> 🧪 **Тестирование:** поскольку порядок `HashMap` случаен, никогда не сравнивайте в тесте его
> напечатанный текст. Проверяйте отдельные записи через `get` или сравнивайте две таблицы целиком
> через `==` (две таблицы равны, когда в них одни и те же записи в любом порядке).

```rust
use std::collections::HashMap;

/// Counts how many times each word appears. Words are compared in lower case.
fn word_counts(text: &str) -> HashMap<String, u32> {
    let mut counts = HashMap::new();
    for word in text.split_whitespace() {
        *counts.entry(word.to_lowercase()).or_insert(0) += 1;
    }
    counts
}

fn main() {
    let counts = word_counts("Rust rust RUST is fun");
    println!("rust: {:?}", counts.get("rust"));
    println!("fun:  {:?}", counts.get("fun"));
    println!("go:   {:?}", counts.get("go"));
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn counts_ignoring_case() {
        let counts = word_counts("Rust rust RUST is fun");
        assert_eq!(counts.get("rust"), Some(&3));
        assert_eq!(counts.get("fun"), Some(&1));
        assert_eq!(counts.get("go"), None);
    }

    #[test]
    fn empty_text_gives_empty_map() {
        assert!(word_counts("").is_empty());
    }

    #[test]
    fn whole_map_comparison() {
        let expected = HashMap::from([(String::from("a"), 2), (String::from("b"), 1)]);
        assert_eq!(word_counts("a b a"), expected);
    }
}
```

**Что происходит:**

1. `word_counts` строит и возвращает таблицу; вызывающий код становится её владельцем.
2. `counts.get("rust")` возвращает `Option<&u32>`, поэтому тест сравнивает с `Some(&3)`: ссылкой
   на 3.
3. `use super::*;` внутри `mod tests` импортирует всё из родительского модуля, включая имя
   `HashMap` ([Урок 04.6](06-modules-crates-and-packages.md)).

```text
rust: Some(3)
fun:  Some(1)
go:   None
```

```text
running 3 tests
test tests::empty_text_gives_empty_map ... ok
test tests::whole_map_comparison ... ok
test tests::counts_ignoring_case ... ok

test result: ok. 3 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

> ⚡ **Производительность:** `HashMap::with_capacity(n)` резервирует место под `n` записей и
> избавляет от увеличения таблицы, как и у вектора. Для горстки элементов маленький `Vec` пар
> может оказаться быстрее таблицы, но не гадайте: измеряйте.

## Частые ошибки

**1. Забыли импорт.**

```rust
fn main() {
    let m: HashMap<String, i32> = HashMap::new();
    println!("{}", m.len());
}
```

```text
error[E0425]: cannot find type `HashMap` in this scope
 --> src/main.rs:2:12
  |
2 |     let m: HashMap<String, i32> = HashMap::new();
  |            ^^^^^^^ not found in this scope
  |
help: consider importing this struct
  |
1 + use std::collections::HashMap;
  |
```

(Вторая ошибка, E0433, указывает на `HashMap::new()`.) Исправление: добавьте
`use std::collections::HashMap;`.

**2. Передали значение там, где `get` ждёт ссылку.**

```rust
use std::collections::HashMap;

fn main() {
    let mut m: HashMap<String, i32> = HashMap::new();
    m.insert(String::from("a"), 1);
    let name = String::from("a");
    println!("{:?}", m.get(name));
}
```

```text
error[E0308]: mismatched types
 --> src/main.rs:7:28
  |
7 |     println!("{:?}", m.get(name));
  |                        --- ^^^^ expected `&_`, found `String`
  |                        |
  |                        arguments to this method are incorrect
  |
help: consider borrowing here
  |
7 |     println!("{:?}", m.get(&name));
  |                            +
```

`get` заимствует ключ для поиска, поэтому ему нужна ссылка: `m.get(&name)`.

**3. Изменение таблицы во время обхода циклом.**

```rust
use std::collections::HashMap;

fn main() {
    let mut m: HashMap<String, i32> = HashMap::new();
    m.insert(String::from("a"), 1);
    for (k, v) in &m {
        if *v == 1 {
            m.insert(format!("{k}2"), 2);
        }
    }
}
```

```text
error[E0502]: cannot borrow `m` as mutable because it is also borrowed as immutable
  --> src/main.rs:8:13
   |
 6 |       for (k, v) in &m {
   |       -             --
   |       |             |
   |       |             immutable borrow occurs here
   |  _____|             immutable borrow later used here
   | |
 7 | |         if *v == 1 {
 8 | |             m.insert(format!("{k}2"), 2);
   | |             ^^^^^^^^^^^^^^^^^^^^^^^^^^^^ mutable borrow occurs here
 9 | |         }
10 | |     }
   | |_____- this for loop borrows `m` immutably, preventing mutation within its body
```

Та же идея, что и с `push` в вектор во время цикла ([Урок 04.7](07-vectors.md)): вставка может увеличить таблицу и
сделать недействительными ссылки, которые держит цикл. Исправление: во время цикла собирайте то,
что хотите добавить, в отдельный `Vec`, а вставляйте после цикла.

**4. Использование `String` после её вставки** — см. раздел 5.

**5. Число с плавающей точкой в роли ключа** — см. раздел 7.

**6. Ожидание фиксированного порядка** — см. раздел 6.

## Попробуйте сами

В `code/p04-hash-maps` напишите программу, которая считает, сколько раз каждая буква встречается в
слове, и печатает результат в алфавитном порядке:

```rust
use std::collections::BTreeMap;

fn main() {
    let word = "mississippi";
    let mut counts: BTreeMap<char, u32> = BTreeMap::new();
    for c in word.chars() {
        *counts.entry(c).or_insert(0) += 1;
    }
    for (letter, n) in &counts {
        println!("{letter}: {n}");
    }
}
```

1. Запустите командой `cargo run`. Ожидаемый вывод:

```text
i: 4
m: 1
p: 2
s: 4
```

2. Замените `BTreeMap` на `HashMap` (и в строке `use` тоже) и запустите три раза. Остаётся ли
   порядок прежним?
3. Замените слово на `"Mississippi"`. Считается ли `M` вместе с `m`? Как это исправить?
   (Используйте `c.to_lowercase()` из [Урока 04.8](08-strings-in-depth.md).)
4. Сделайте коммит: `git add .` и `git commit -m "Count letters with a map"`.

## Упражнения

### Упражнение 1 — Крошечная телефонная книга (легко)

Создайте `HashMap<String, String>` с `alice → +998-90-111-22-33` и `bob → +998-91-444-55-66`.
Найдите `"alice"` и `"carol"` и напечатайте либо номер, либо `not found`. Ожидаемый вывод:

```text
alice: +998-90-111-22-33
carol: not found
```

<details>
<summary>Подсказка</summary>

Два раза вызовите `insert`. Пройдите циклом по массиву `["alice", "carol"]` и сделайте `match` по
`phones.get(name)`.

</details>

<details>
<summary>Ответ</summary>

```rust
use std::collections::HashMap;

fn main() {
    let mut phones: HashMap<String, String> = HashMap::new();
    phones.insert(String::from("alice"), String::from("+998-90-111-22-33"));
    phones.insert(String::from("bob"), String::from("+998-91-444-55-66"));

    for name in ["alice", "carol"] {
        match phones.get(name) {
            Some(number) => println!("{name}: {number}"),
            None => println!("{name}: not found"),
        }
    }
}
```

Почему это работает: `get` принимает `&str` для ключа `String` и возвращает `None` для
отсутствующего ключа, поэтому паники нет.

</details>

### Упражнение 2 — Склад (легко)

Создайте таблицу склада: `apples → 10` и `pears → 4`. Уберите 3 яблока через `get_mut`, удалите
груши через `remove` и добавьте 6 слив через `entry`. Напечатайте итоговый склад, отсортировав по
названию. Ожидаемый вывод:

```text
apples: 7
plums: 6
```

<details>
<summary>Подсказка</summary>

`if let Some(n) = stock.get_mut("apples") { *n -= 3; }`. Для слив:
`*stock.entry(String::from("plums")).or_insert(0) += 6;`. Чтобы напечатать по порядку, соберите
ключи в `Vec<&String>` и отсортируйте его.

</details>

<details>
<summary>Ответ</summary>

```rust
use std::collections::HashMap;

fn main() {
    let mut stock: HashMap<String, u32> = HashMap::new();
    stock.insert(String::from("apples"), 10);
    stock.insert(String::from("pears"), 4);

    if let Some(n) = stock.get_mut("apples") {
        *n -= 3;
    }
    stock.remove("pears");
    *stock.entry(String::from("plums")).or_insert(0) += 6;

    let mut names: Vec<&String> = stock.keys().collect();
    names.sort();
    for name in names {
        println!("{name}: {}", stock[name]);
    }
}
```

Почему это работает: `get_mut` редактирует существующее значение, `remove` удаляет запись, а
`entry` создаёт отсутствующую. Сортировка ключей делает вывод стабильным.

</details>

### Упражнение 3 — Первый дубликат, с тестами (средне)

Напишите `fn first_duplicate(v: &[i32]) -> Option<i32>`, которая возвращает первое число,
встретившееся во второй раз. Для `[3, 1, 4, 1, 5, 3]` ответ — `Some(1)` (1 повторяется раньше, чем
3). Для `[1, 2, 3]` ответ — `None`. Напишите три теста: есть повтор, нет повтора и пустой срез.

<details>
<summary>Подсказка</summary>

Используйте `HashSet<i32>`. Для каждого числа вызывайте `seen.insert(n)`. Он возвращает `false`,
если число уже было в множестве, и именно в этот момент нужно сделать `return Some(n)`.

</details>

<details>
<summary>Ответ</summary>

```rust
use std::collections::HashSet;

/// Returns the first number that appears a second time, or None.
fn first_duplicate(v: &[i32]) -> Option<i32> {
    let mut seen = HashSet::new();
    for &n in v {
        if !seen.insert(n) {
            return Some(n);
        }
    }
    None
}

fn main() {
    println!("{:?}", first_duplicate(&[3, 1, 4, 1, 5, 3]));
    println!("{:?}", first_duplicate(&[1, 2, 3]));
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn finds_first_repeat() {
        assert_eq!(first_duplicate(&[3, 1, 4, 1, 5, 3]), Some(1));
    }

    #[test]
    fn no_repeat() {
        assert_eq!(first_duplicate(&[1, 2, 3]), None);
    }

    #[test]
    fn empty() {
        assert_eq!(first_duplicate(&[]), None);
    }
}
```

```text
Some(1)
None
```

Почему это работает: проверка и вставка — один шаг, поэтому на каждое число смотрят один раз.
Rust выводит `HashSet<i32>` из вызова `insert(n)`.

</details>

### Упражнение 4 — Группировка слов по длине (средне)

Для `["tree", "cat", "sun", "house", "dog", "moon", "a"]` постройте
`HashMap<usize, Vec<&str>>`, который группирует слова по длине, и напечатайте группы в порядке
длины. Ожидаемый вывод:

```text
1: ["a"]
3: ["cat", "sun", "dog"]
4: ["tree", "moon"]
5: ["house"]
```

<details>
<summary>Подсказка</summary>

Группировку делает `by_len.entry(w.len()).or_default().push(w);`. Чтобы напечатать по порядку,
скопируйте ключи в `Vec<usize>` циклом, отсортируйте и печатайте `by_len[&len]`.

</details>

<details>
<summary>Ответ</summary>

```rust
use std::collections::HashMap;

fn main() {
    let words = ["tree", "cat", "sun", "house", "dog", "moon", "a"];
    let mut by_len: HashMap<usize, Vec<&str>> = HashMap::new();
    for w in words {
        by_len.entry(w.len()).or_default().push(w);
    }

    let mut lengths: Vec<usize> = Vec::new();
    for len in by_len.keys() {
        lengths.push(*len);
    }
    lengths.sort();
    for len in lengths {
        println!("{len}: {:?}", by_len[&len]);
    }
}
```

Почему это работает: `or_default()` создаёт пустой вектор, когда длина встречается впервые.
Слова внутри каждой группы остаются в том порядке, в котором их добавили.

</details>

### Упражнение 5 — Самое частое слово, с тестами (сложно)

Напишите `fn most_common(text: &str) -> Option<(String, u32)>`, которая возвращает самое частое
слово (сравнение в нижнем регистре) и его количество. Если несколько слов делят первое место,
побеждает меньшее по алфавиту. Пустой текст даёт `None`. Тесты: явный победитель, ничья, разный
регистр и пустой текст.

<details>
<summary>Подсказка</summary>

Сначала постройте счётчики через `entry` (как `word_counts` в разделе 9). Затем заведите
переменную `best: Option<(String, u32)>` и пройдите циклом по таблице. Пара лучше, если `best`
равна `None`, или её счётчик больше, или счётчик равен, а слово меньше. Поскольку у таблицы нет
порядка, именно правило ничьей делает результат предсказуемым.

</details>

<details>
<summary>Ответ</summary>

```rust
use std::collections::HashMap;

/// The most frequent word (lower case) and its count.
/// If several words tie, the alphabetically smallest wins.
fn most_common(text: &str) -> Option<(String, u32)> {
    let mut counts: HashMap<String, u32> = HashMap::new();
    for word in text.split_whitespace() {
        *counts.entry(word.to_lowercase()).or_insert(0) += 1;
    }

    let mut best: Option<(String, u32)> = None;
    for (word, count) in counts {
        let better = match &best {
            None => true,
            Some((best_word, best_count)) => {
                count > *best_count || (count == *best_count && word < *best_word)
            }
        };
        if better {
            best = Some((word, count));
        }
    }
    best
}

fn main() {
    println!("{:?}", most_common("the cat and the dog and the bird"));
    println!("{:?}", most_common("b a b a"));
    println!("{:?}", most_common(""));
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn finds_the_winner() {
        assert_eq!(most_common("the cat and the dog and the bird"), Some((String::from("the"), 3)));
    }

    #[test]
    fn tie_goes_to_smaller_word() {
        assert_eq!(most_common("b a b a"), Some((String::from("a"), 2)));
    }

    #[test]
    fn case_is_ignored() {
        assert_eq!(most_common("Go go GO"), Some((String::from("go"), 3)));
    }

    #[test]
    fn empty_text() {
        assert_eq!(most_common(""), None);
    }
}
```

```text
Some(("the", 3))
Some(("a", 2))
None
```

Почему это работает: `for (word, count) in counts` поглощает таблицу и выдаёт пары во владение,
поэтому `word` можно переместить в `best`. `match &best` только заимствует `best`, пока мы
сравниваем, так что присваивание в `best` потом разрешено. Правило ничьей убирает зависимость от
случайного порядка.

</details>

### Упражнение 6 — Запоминаем результаты (сложно)

Числа Фибоначчи: `0, 1, 1, 2, 3, 5, 8, …` (каждое — сумма двух предыдущих). Простая рекурсивная
функция повторяет одну и ту же работу огромное число раз. Напишите
`fn fib(n: u64, memo: &mut HashMap<u64, u64>) -> u64`, которая сохраняет каждый результат в
таблице (*мемоизация*: запоминание ответов) и переиспользует его. Напечатайте `fib(10)`, `fib(80)`
и сколько значений сохранено. Ожидаемый вывод:

```text
fib(10) = 55
fib(80) = 23416728348467685
cached values: 79
```

<details>
<summary>Подсказка</summary>

Если `n <= 1`, верните `n`. Затем проверьте `memo.get(&n)`: если это `Some(&cached)`, верните
`cached`. Иначе вычислите `fib(n - 1, memo) + fib(n - 2, memo)`, сделайте `insert` и верните
результат. Передавайте таблицу дальше как `memo` (это уже `&mut`), а из `main` вызывайте с
`&mut memo`.

</details>

<details>
<summary>Ответ</summary>

```rust
use std::collections::HashMap;

fn fib(n: u64, memo: &mut HashMap<u64, u64>) -> u64 {
    if n <= 1 {
        return n;
    }
    if let Some(&cached) = memo.get(&n) {
        return cached;
    }
    let result = fib(n - 1, memo) + fib(n - 2, memo);
    memo.insert(n, result);
    result
}

fn main() {
    let mut memo = HashMap::new();
    println!("fib(10) = {}", fib(10, &mut memo));
    println!("fib(80) = {}", fib(80, &mut memo));
    println!("cached values: {}", memo.len());
}
```

Почему это работает: каждое значение от 2 до 80 вычисляется один раз и сохраняется (79 значений).
`if let` копирует число из ссылки (`&cached`) до рекурсии, поэтому, когда мы снова вызываем `fib`,
ни одно заимствование `memo` не живо. Без таблицы `fib(80)` считалась бы дольше, чем вы
согласились бы ждать.

</details>

## Вопросы для самопроверки

1. Чем поиск элемента в векторе отличается от поиска в хеш-таблице?

<details>
<summary>Ответ</summary>

Вектор находит элемент по его позиции (индексу). Хеш-таблица находит значение по ключу. Поиск в
таблице занимает примерно одинаковое время независимо от её размера.

</details>

2. Что возвращает `map.insert(key, value)`, если ключ уже существует?

<details>
<summary>Ответ</summary>

`Some(old_value)`: значение заменяется, а предыдущее возвращается. Для нового ключа
возвращается `None`.

</details>

3. Почему `map.get("x")` возвращает `Option`?

<details>
<summary>Ответ</summary>

Потому что ключа может не быть в таблице. `Some(&value)` значит «найдено»; `None` — «не найдено».

</details>

4. Что делает `*map.entry(word).or_insert(0) += 1;`?

<details>
<summary>Ответ</summary>

Находит запись для `word`, сохраняет там `0`, если её не было, и затем прибавляет 1 к значению
через возвращённую изменяемую ссылку. Так считается, сколько раз встретилось `word`, — за один
поиск.

</details>

5. Почему порядок обхода `HashMap` не одинаков при каждом запуске? Как получить стабильный порядок?

<details>
<summary>Ответ</summary>

Порядок зависит от хешей, а Rust намеренно делает хеширование случайным (это защищает от атак
hash flooding). Для стабильного порядка соберите и отсортируйте ключи или используйте `BTreeMap`.

</details>

6. Почему `f64` нельзя использовать как ключ `HashMap`?

<details>
<summary>Ответ</summary>

У чисел с плавающей точкой нет `Eq` и `Hash` из-за `NaN` (оно не равно самому себе). Используйте
целое число (например, центы) или тип-обёртку.

</details>

7. Что происходит со `String` после того, как вы вставили её как ключ?

<details>
<summary>Ответ</summary>

Она перемещается в таблицу, и переменную больше использовать нельзя. Если текст ещё нужен,
вставляйте клон.

</details>

8. Когда вы выберете `BTreeMap` вместо `HashMap`?

<details>
<summary>Ответ</summary>

Когда нужны записи в отсортированном порядке ключей, наименьший или наибольший ключ или диапазон
ключей.

</details>

## Итоги

- `HashMap<K, V>` хранит пары «ключ → значение» и очень быстро находит значение по ключу.
- Импортируйте его через `use std::collections::HashMap;`. Используйте `insert`, `get`, `get_mut`,
  `remove`, `contains_key` и `len`.
- `get` возвращает `Option`; `map[key]` паникует, когда ключа нет.
- API `entry` (`or_insert`, `or_default`) вставляет отсутствующий ключ и обновляет значение за
  один поиск. Это инструмент для подсчёта и группировки.
- Ключи и значения, которыми вы владеете, перемещаются в таблицу; копируемые — копируются.
- Порядок обхода `HashMap` случаен. Когда порядок важен, сортируйте ключи или используйте
  `BTreeMap`.
- Ключу нужны `Hash` и `Eq`: подходят целые числа, `String`, `&str`, `char`, `bool` и кортежи;
  числа с плавающей точкой — нет. Для своих типов используйте `derive`.
- `HashSet` хранит уникальные значения. В тестах сравнивайте отдельные записи или таблицы целиком,
  а не напечатанный текст.

## Источники и что почитать

- [The Rust Programming Language — 8.3 Storing Keys with Associated Values in Hash Maps](https://doc.rust-lang.org/book/ch08-03-hash-maps.html) — глава официальной книги о хеш-таблицах.
- [std::collections::HashMap](https://doc.rust-lang.org/std/collections/struct.HashMap.html) — все методы, включая API entry, с примерами.
- [std::collections](https://doc.rust-lang.org/std/collections/index.html) — обзор всех стандартных коллекций и руководство по выбору.
- [std::collections::BTreeMap](https://doc.rust-lang.org/std/collections/struct.BTreeMap.html) — отсортированная таблица.
- [std::collections::HashSet](https://doc.rust-lang.org/std/collections/struct.HashSet.html) — тип множества.
- [Rust by Example — HashMap](https://doc.rust-lang.org/rust-by-example/std/hash.html) — короткие примеры, которые можно запустить.

---

[🏠 Начало](../START-HERE.md) · [← 04.8 Строки подробно](08-strings-in-depth.md) · [📋 Этап 04](README.md) · [04.10 Обработка ошибок →](10-error-handling.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
