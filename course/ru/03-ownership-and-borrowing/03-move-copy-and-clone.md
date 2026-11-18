<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 03.2 Правила владения](02-ownership-rules.md) · [📋 Этап 03](README.md) · [03.4 Ссылки и заимствование →](04-references-and-borrowing.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/03-ownership-and-borrowing/03-move-copy-and-clone.md) <!-- nav -->

# Урок 03.3 — Перемещение, копирование и клонирование

> **Этап 03 — Владение и заимствование** · Урок 3 из 8 · ⏱ около 2,5 часа (2 занятия)

## Что вы узнаете

- Что именно происходит в памяти при **перемещении** (move) значения и почему старая переменная становится непригодной.
- Какие типы **копируются** автоматически (`Copy`), а какие перемещаются.
- Как сделать независимый дубликат с помощью `.clone()` и сколько это стоит.
- Где прячутся перемещения: присваивания, вызовы функций, `Vec::push`, циклы `for` и индексация.
- Как выбирать между перемещением, копированием, клонированием и (скоро) заимствованием.

## Перед началом

- Вы прошли [Урок 03.2 — Правила владения](02-ownership-rules.md). Вы знаете три правила и то, что передача `String` в функцию перемещает её.
- Создайте проект: `cargo new code/p03-move-copy-clone` (из `~/projects/rust`).

## Новые слова

| Слово | Значение |
|---|---|
| **перемещение** (move) | Передача владения значением новому владельцу. Само значение не копируется; старый владелец становится недействительным. |
| **дескриптор** (handle) | Небольшая часть значения фиксированного размера, которая живёт на стеке и указывает на настоящие данные. Для `String`: указатель, длина, ёмкость. |
| **Copy** | *Трейт* (набор возможностей; Урок 05.2), который помечает тип как «дёшево дублируемый копированием его байтов». Значения таких типов копируются вместо перемещения. |
| **Clone** | Трейт с методом `.clone()`, который делает полный независимый дубликат значения, включая его данные в куче. |
| **глубокая копия** (deep copy) | Дубликат, который дублирует также всё, чем значение владеет в куче. `.clone()` у `String` делает глубокую копию. |
| **побитовое копирование** (bitwise copy) | Копирование байтов значения в точности как они есть, без лишней работы. |
| **`.clone()`** | Метод, выполняющий клонирование. Он всегда виден в вашем коде, поэтому видно, где создаются копии. |
| **итератор** (iterator) | Штука, которая выдаёт элементы по одному. Цикл `for` использует итератор. (Итераторы изучаются в Уроке 05.5.) |

## 1. Перемещение на картинках

Возьмём такой код:

```rust
fn main() {
    let s = String::from("hello");
    let before = s.as_ptr();
    println!("s: len = {}, capacity = {}, letters at {:p}", s.len(), s.capacity(), before);

    let t = s; // the move
    println!("t: len = {}, capacity = {}, letters at {:p}", t.len(), t.capacity(), t.as_ptr());
    println!("same letters in memory: {}", before == t.as_ptr());
}
```

**Что происходит:**

1. `s` — это `String`: дескриптор (указатель, длина, ёмкость) на стеке и буквы `hello` в куче. Адрес букв мы сохраняем в `before`.
2. `let t = s;` копирует 24-байтовый **дескриптор** в `t`. Буквы **не** копируются. Затем Rust помечает `s` как «перемещённую» и не позволит использовать её снова.
3. Мы выводим данные `t`. Они такие же, какими были у `s`, а буквы находятся по тому же адресу.

```text
s: len = 5, capacity = 5, letters at 0x102fc92d0
t: len = 5, capacity = 5, letters at 0x102fc92d0
same letters in memory: true
```

На картинке:

```text
BEFORE `let t = s;`                      AFTER `let t = s;`

 STACK                 HEAP               STACK                  HEAP
┌──────────────┐     ┌─────────┐        ┌──────────────┐       ┌─────────┐
│ s  ptr ──────┼───► │ h e l.. │        │ s  (moved,   │       │ h e l.. │
│    len = 5   │     └─────────┘        │    unusable) │       └─────────┘
│    cap = 5   │                        ├──────────────┤            ▲
└──────────────┘                        │ t  ptr ──────┼────────────┘
                                        │    len = 5   │
                                        │    cap = 5   │
                                        └──────────────┘
```

Поэтому перемещение **очень дёшево**: копируется всего несколько байтов, как бы длинен ни был текст. Данные в куче остаются на месте.

Почему Rust делает `s` непригодной? Подумайте, что было бы иначе. Если бы и `s`, и `t` указывали на буквы и обе были действительны, то в конце области видимости были бы уничтожены **обе**, и одна и та же память в куче освободилась бы дважды. Это двойное освобождение из Урока 03.1. Правило «в каждый момент один владелец» предотвращает это: после перемещения владелец только `t`, поэтому память освобождает только `t`.

Если попытаться использовать `s` после перемещения, компилятор остановит вас:

```rust
fn main() {
    let s = String::from("hello");
    let t = s;
    println!("{s}, world!");
    println!("{t}");
}
```

```text
error[E0382]: borrow of moved value: `s`
 --> src/main.rs:4:16
  |
2 |     let s = String::from("hello");
  |         - move occurs because `s` has type `String`, which does not implement the `Copy` trait
3 |     let t = s;
  |             - value moved here
4 |     println!("{s}, world!");
  |                ^ value borrowed here after move
  |
help: consider cloning the value if the performance cost is acceptable
  |
3 |     let t = s.clone();
  |              ++++++++
```

Важная фраза — «which does not implement the `Copy` trait». Раздел 3 объясняет её.

## 2. Где прячутся перемещения

Перемещение происходит всякий раз, когда значение с дескриптором *используется по значению*. Вызовы функций вы уже знаете. Вот другие частые места.

**Добавление в `Vec`.** `push` принимает элемент по значению, поэтому элемент перемещается в вектор:

```rust
fn main() {
    let mut names = Vec::new();
    let first = String::from("Ada");
    names.push(first);
    println!("{first}");
    println!("{:?}", names);
}
```

```text
error[E0382]: borrow of moved value: `first`
 --> src/main.rs:5:16
  |
3 |     let first = String::from("Ada");
  |         ----- move occurs because `first` has type `String`, which does not implement the `Copy` trait
4 |     names.push(first);
  |                ----- value moved here
5 |     println!("{first}");
  |                ^^^^^ value borrowed here after move
```

Теперь вектор — владелец строки `"Ada"`. Это логично: список, который хранит строку, должен ею владеть, иначе строку могли бы освободить, пока она ещё лежит в списке.

**Цикл `for` по `Vec`.** `for name in names` берёт весь вектор по значению. Он его *потребляет*: каждый элемент по очереди перемещается в `name`, а сам вектор после цикла исчезает.

```rust
fn main() {
    let names = vec![String::from("Ada"), String::from("Grace")];
    for name in names {
        println!("hello, {name}");
    }
    println!("{} names", names.len());
}
```

```text
error[E0382]: borrow of moved value: `names`
 --> src/main.rs:6:26
  |
2 |     let names = vec![String::from("Ada"), String::from("Grace")];
  |         ----- move occurs because `names` has type `Vec<String>`, which does not implement the `Copy` trait
3 |     for name in names {
  |                 ----- `names` moved due to this implicit call to `.into_iter()`
...
6 |     println!("{} names", names.len());
  |                          ^^^^^ value borrowed here after move
  |
help: consider iterating over a slice of the `Vec<String>`'s content to avoid moving into the `for` loop
  |
3 |     for name in &names {
  |                 +
```

Компилятор подсказывает исправление: напишите `for name in &names`. Символ `&` означает «заимствовать вектор, а не забирать его». Ссылки — тема следующего урока; пока просто скопируйте этот приём:

```rust
fn main() {
    let names = vec![String::from("Ada"), String::from("Grace")];
    for name in &names {
        println!("hello, {name}");
    }
    println!("{} names", names.len());
}
```

```text
hello, Ada
hello, Grace
2 names
```

**Перемещение только в одной ветке.** Если значение перемещается хотя бы на *одном* из путей выполнения кода, Rust считает его перемещённым и после:

```rust
fn main() {
    let s = String::from("hello");
    let keep = std::env::args().count() > 5;
    if keep {
        drop(s);
    }
    println!("{s}");
}
```

```text
error[E0382]: borrow of moved value: `s`
 --> src/main.rs:7:16
  |
2 |     let s = String::from("hello");
  |         - move occurs because `s` has type `String`, which does not implement the `Copy` trait
...
5 |         drop(s);
  |              - value moved here
6 |     }
7 |     println!("{s}");
  |                ^ value borrowed here after move
```

(`std::env::args().count()` считает аргументы командной строки; здесь он просто даёт значение, которое компилятор не может знать заранее.) Компилятор не знает, выполнится ли `if`, поэтому предполагает худший случай.

**Перемещённой переменной можно присвоить новое значение.** Переменную, объявленную с `mut`, из которой переместили значение, можно присвоить снова, и она снова действительна:

```rust
fn main() {
    let mut s = String::from("first");
    let t = s;
    println!("t = {t}");

    s = String::from("second");
    println!("s = {s}");
}
```

```text
t = first
s = second
```

## 3. Copy: типы, которые не перемещаются

Теперь обратный вопрос. Почему это работает?

```rust
fn main() {
    let a = 5;
    let b = a;
    println!("a = {a}, b = {b}");

    let pair = (1, 2.5);
    let pair2 = pair;
    println!("{:?} and {:?}", pair, pair2);

    let letters = ['x', 'y', 'z'];
    let letters2 = letters;
    println!("{:?} and {:?}", letters, letters2);

    let flag = true;
    let flag2 = flag;
    println!("{flag} and {flag2}");
}
```

```text
a = 5, b = 5
(1, 2.5) and (1, 2.5)
['x', 'y', 'z'] and ['x', 'y', 'z']
true and true
```

(`{:?}` выводит значение в «отладочном» формате, который работает для кортежей и массивов.)

Здесь `let b = a;` не делает `a` непригодной. Причина в том, что у `i32` есть особая способность: он реализует трейт **`Copy`**. Тип `Copy` настолько мал и прост, что Rust копирует его байты всякий раз, когда вы его «перемещаете» — это **побитовое копирование**. Обе переменные остаются действительными, и у каждой своя копия. Здесь нет ни кучи, ни ресурса, поэтому нечего освобождать дважды.

Какие типы `Copy`?

| Тип | Copy? | Почему |
|---|---|---|
| `i32`, `u8`, `u64`, `usize`, … (все целые) | да | простые числа |
| `f32`, `f64` | да | простые числа |
| `bool`, `char` | да | простые значения |
| `(i32, f64)`, `[char; 3]` | да, **если каждый элемент `Copy`** | все части — небольшие копии |
| `&T` (разделяемая ссылка, Урок 03.4) | да | это просто адрес |
| `&str` | да | это разделяемая ссылка |
| `String`, `Vec<T>`, `Box<T>` | **нет** | они владеют памятью в куче |
| `(i32, String)` | **нет** | содержит `String` |
| `&mut T` (изменяемая ссылка, Урок 03.5) | **нет** | в каждый момент может существовать только одна |

Практическое правило: **тип может быть `Copy` только если он не владеет ничем, что нужно убирать.** Тип, у которого есть реализация `Drop` или который владеет памятью в куче, никогда не может быть `Copy`.

Кортеж `Copy` только если `Copy` все его элементы:

```rust
fn main() {
    let mixed = (1, String::from("x"));
    let other = mixed;
    println!("{:?}", mixed);
    println!("{:?}", other);
}
```

```text
error[E0382]: borrow of moved value: `mixed`
 --> src/main.rs:4:22
  |
2 |     let mixed = (1, String::from("x"));
  |         ----- move occurs because `mixed` has type `(i32, String)`, which does not implement the `Copy` trait
3 |     let other = mixed;
  |                 ----- value moved here
4 |     println!("{:?}", mixed);
  |                      ^^^^^ value borrowed here after move
```

> 📏 **Соглашение:** когда в Уроке 04.1 вы будете проектировать собственные типы, вы увидите `#[derive(Clone, Copy)]` — однострочный способ сделать небольшой тип `Copy`. Делайте так для небольших типов «простых данных» (точка, цвет). Не делайте так для типов, владеющих данными в куче — компилятор откажет.

## 4. Clone: явный независимый дубликат

Иногда вам действительно нужны две независимые строки. Вы просите об этом с помощью `.clone()`:

```rust
fn main() {
    let original = String::from("hello");
    let mut copy = original.clone();
    copy.push_str(", world");

    println!("original = {original}");
    println!("copy     = {copy}");
    println!("original letters at {:p}", original.as_ptr());
    println!("copy letters at     {:p}", copy.as_ptr());
}
```

**Что происходит:**

1. `original.clone()` просит у аллокатора новую память в куче, копирует в неё буквы и строит новый дескриптор. Это **глубокая копия**: дескриптор *и* данные в куче.
2. `copy.push_str(...)` меняет только `copy`. `original` не затронута, потому что у них нет ничего общего.
3. Два адреса букв различаются. Это два отдельных куска памяти в куче.

```text
original = hello
copy     = hello, world
original letters at 0x1036d12d0
copy letters at     0x1036d1300
```

```text
original                     copy
 ptr ──► [h e l l o]          ptr ──► [h e l l o , _ w o r l d]
 len 5                        len 12
```

Теперь у прежнего примера с `Vec` есть исправление: клонируем, добавляем клон и сохраняем оригинал.

```rust
fn main() {
    let mut names = Vec::new();
    let first = String::from("Ada");
    names.push(first.clone());
    println!("{first}");
    println!("{:?}", names);
}
```

```text
Ada
["Ada"]
```

> ⚡ **Производительность:** перемещение стоит несколько байтов; клонирование стоит выделения памяти плюс копирования всех данных, так что клонирование 100-мегабайтной строки копирует 100 МБ. Главное: **клонирование всегда видно** в коде как `.clone()`. Когда вы проверяете код, каждый `.clone()` — повод спросить: «Эта копия действительно нужна?». Новички часто добавляют `.clone()`, чтобы просто убрать ошибку. Это компилируется, но может скрывать проблему проектирования. Сначала попробуйте заимствовать (Урок 03.4); клонируйте, когда действительно нужны два независимых владельца.

## 5. Перемещение из Vec по индексу

Частая неожиданность для новичков:

```rust
fn main() {
    let names = vec![String::from("Ada"), String::from("Grace")];
    let first = names[0];
    println!("{first}");
}
```

```text
error[E0507]: cannot move out of index of `Vec<String>`
 --> src/main.rs:3:17
  |
3 |     let first = names[0];
  |                 ^^^^^^^^ move occurs because value has type `String`, which does not implement the `Copy` trait
  |
help: consider borrowing here
  |
3 |     let first = &names[0];
  |                 +
help: consider cloning the value if the performance cost is acceptable
  |
3 |     let first = names[0].clone();
  |                         ++++++++
```

Если бы это работало, в векторе осталась бы «дыра» на позиции 0: элемент, который перемещён наружу, но вектор всё ещё считает, что владеет им. В конце вектор попытался бы его уничтожить, и новый владелец тоже уничтожил бы его — двойное освобождение. Поэтому Rust запрещает перемещать из индекса. Компилятор предлагает два исправления:

```rust
fn main() {
    let names = vec![String::from("Ada"), String::from("Grace")];

    let borrowed = &names[0];
    println!("borrowed: {borrowed}");

    let owned = names[1].clone();
    println!("owned: {owned}");

    println!("names is still whole: {:?}", names);
}
```

```text
borrowed: Ada
owned: Grace
names is still whole: ["Ada", "Grace"]
```

С `&names[0]` вы *смотрите на* элемент (заимствование; следующий урок). С `.clone()` вы получаете собственную копию. (Есть также методы, которые удаляют элемент из вектора и отдают его вам, например `remove`. Вы встретите их в Уроке 04.7.)

## 6. Как выбирать

| Ситуация | Что использовать |
|---|---|
| Значение — небольшой простой тип (`i32`, `bool`, `char`, …) | ничего особенного — оно копируется |
| Я закончил со старой переменной и хочу, чтобы новый владелец получил значение | **перемещение** (просто присвойте или передайте) |
| Я хочу лишь ненадолго посмотреть на значение | **заимствование** (`&`, следующий урок) |
| Мне нужны два независимых владельца, которые могут меняться отдельно | **`.clone()`** |
| Компилятор говорит «значение использовано после перемещения», а я не понимаю почему | найдите перемещение (подпись `value moved here`); спросите себя, достаточно ли заимствования |

## Частые ошибки

**1. Добавлять `.clone()` везде «пока не скомпилируется».** Это работает, но скрывает настоящий вопрос (кто должен этим владеть?) и стоит памяти и времени. Сначала исправьте проект или заимствуйте.

**2. Думать, что перемещение копирует данные в куче.** Оно копирует только дескриптор. Поэтому перемещения дёшевы, а старую переменную приходится отключать.

**3. Ожидать, что ваши собственные типы автоматически `Copy`.** Новый тип *не* `Copy`, пока вы этого не скажете (Урок 04.1).

**4. Забывать, что `for x in vec` потребляет вектор.** Пишите `for x in &vec`, если вектор нужен дальше.

**5. Перемещать из вектора через `v[0]` элемент, не являющийся `Copy` (`E0507`).** Используйте `&v[0]` или `v[0].clone()`.

## Попробуйте сами

1. В `~/projects/rust` выполните `cargo new code/p03-move-copy-clone` (если вы ещё не сделали этого в разделе «Перед началом»). В `src/main.rs` напишите программу `move_demo` из раздела 1 и запустите её. Действительно ли `same letters in memory` равно `true`?
2. Добавьте `println!("{s}");` в конец `main`. Прочтите ошибку компилятора и найдите три подписи.
3. Замените `let t = s;` на `let t = s.clone();`. Ошибка исчезнет. Теперь сравните `s.as_ptr()` и `t.as_ptr()`: что вы видите?
4. Замените `String` на `i32` (`let s = 5;`) и уберите строки с указателями. Всё компилируется, и обе переменные работают. Почему?
5. Сделайте коммит: `git add code/p03-move-copy-clone && git commit -m "p03: move, copy, clone experiments"`.

## Упражнения

### Упражнение 1 — Скомпилируется ли? (легко)

Для каждой программы скажите, компилируется ли она. Если нет — назовите код ошибки.

```rust
// a
let a = String::from("one");
let b = a;
println!("{b}");

// b
let a = 10;
let b = a;
println!("{a} {b}");

// c
let a = String::from("one");
let b = a.clone();
println!("{a} {b}");

// d
fn take(s: String) { println!("{s}"); }
let a = String::from("one");
take(a);
take(a);

// e
let a = [1, 2, 3];
let b = a;
println!("{:?} {:?}", a, b);

// f
let a = vec![1, 2, 3];
let b = a;
println!("{:?} {:?}", a, b);
```

(Каждый фрагмент — это тело `main`, только `take` в (d) определена рядом.)

<details>
<summary>Подсказка</summary>

Для каждого типа спросите: `Copy` ли он? Если нет, `let b = a;` перемещает, и `a` после этого мертва.

</details>

<details>
<summary>Ответ</summary>

- a: компилируется (`a` не используется после перемещения).
- b: компилируется (`i32` — `Copy`).
- c: компилируется (`clone` делает независимую копию).
- d: **ошибка `E0382`** (`a` перемещена в первый `take`).
- e: компилируется (массив из `Copy`-элементов — `Copy`).
- f: **ошибка `E0382`** (`Vec` владеет памятью в куче, поэтому перемещается).

Почему это работает: неявно копировать можно только типы, которые не владеют ничем, что нужно убирать.

</details>

### Упражнение 2 — Copy или нет? (легко)

Ничего не запуская, решите, какие из этих типов — `Copy`: `i32`, `String`, `bool`, `(i32, f64)`, `(i32, String)`, `[i32; 3]`, `Vec<i32>`, `char`, `&str`, `&String`. Затем проверьте себя этой программой, которая компилируется, только если каждый её тип — `Copy`:

```rust
fn is_copy<T: Copy>() {}

fn main() {
    is_copy::<i32>();
    is_copy::<bool>();
    is_copy::<char>();
    is_copy::<(i32, f64)>();
    is_copy::<[i32; 3]>();
    is_copy::<&str>();
    is_copy::<&String>();
    println!("all of these are Copy");
}
```

(`T: Copy` означает «`T` должен быть типом `Copy`»; это ограничение трейта, Урок 05.2.)

<details>
<summary>Подсказка</summary>

Владеет ли тип памятью в куче? Если да, он не `Copy`. Разделяемая ссылка (`&…`) — это лишь адрес, поэтому она `Copy`.

</details>

<details>
<summary>Ответ</summary>

`Copy`: `i32`, `bool`, `(i32, f64)`, `[i32; 3]`, `char`, `&str`, `&String`. Не `Copy`: `String`, `(i32, String)`, `Vec<i32>`.

```text
all of these are Copy
```

Если добавить `is_copy::<String>();`, компилятор ответит `error[E0277]: the trait bound `String: Copy` is not satisfied`.

Почему это работает: функцию `is_copy` можно инстанцировать только с типами `Copy`. Это небольшой приём, чтобы задать компилятору вопрос о типе.

</details>

### Упражнение 3 — Исправьте минимальным изменением (средне)

Эта программа падает с `E0382`. Исправьте её так, чтобы она выводила заголовок, а затем вектор. Используйте ровно один `.clone()` в правильном месте и объясните, почему это правильное место.

```rust
fn main() {
    let title = String::from("Rust");
    let mut titles = Vec::new();
    titles.push(title);
    titles.push(String::from("Ownership"));
    println!("first title was {title}");
    println!("{:?}", titles);
}
```

<details>
<summary>Подсказка</summary>

Вектору нужна собственная `String`. Вы также хотите сохранить `title`. Значит, вектор должен получить копию.

</details>

<details>
<summary>Ответ</summary>

```rust
fn main() {
    let title = String::from("Rust");
    let mut titles = Vec::new();
    titles.push(title.clone());
    titles.push(String::from("Ownership"));
    println!("first title was {title}");
    println!("{:?}", titles);
}
```

```text
first title was Rust
["Rust", "Ownership"]
```

Почему это работает: `push` нужно владение. Клонирование в месте вызова даёт вектору собственную `String`, а `title` сохраняет оригинал. Второму `push` клон не нужен, потому что новая `String::from(...)` — это свежее значение, которым больше никто не пользуется.

</details>

### Упражнение 4 — Дубликат, с тестами (средне)

Напишите `fn duplicate(s: String) -> (String, String)`, которая возвращает две независимые копии своего аргумента. Напишите два теста: один проверяет, что обе строки равны, а другой меняет первую и проверяет, что вторая не изменилась.

<details>
<summary>Подсказка</summary>

Сначала склонируйте, затем верните кортеж `(s, copy)`. Во втором тесте объявите первую часть как `mut`, вызовите для неё `push_str` и сравните обе через `assert_eq!`.

</details>

<details>
<summary>Ответ</summary>

```rust
fn duplicate(s: String) -> (String, String) {
    let copy = s.clone();
    (s, copy)
}

fn main() {
    let (a, b) = duplicate(String::from("echo"));
    println!("{a} {b}");
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn both_halves_are_equal() {
        let (a, b) = duplicate(String::from("echo"));
        assert_eq!(a, b);
    }

    #[test]
    fn halves_are_independent() {
        let (mut a, b) = duplicate(String::from("echo"));
        a.push_str("!!");
        assert_eq!(a, "echo!!");
        assert_eq!(b, "echo");
        assert_ne!(a.as_ptr(), b.as_ptr());
    }
}
```

```text
running 2 tests
..
test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

Почему это работает: клон — это глубокая копия, поэтому изменение `a` не может изменить `b`. Последнее утверждение также доказывает, что буквы лежат по разным адресам.

</details>

### Упражнение 5 — Та же память или нет? (сложно)

Предскажите три строки вывода, затем запустите программу.

```rust
fn main() {
    let a = String::from("data");
    let p = a.as_ptr();

    let b = a;
    println!("1) after move:   {}", b.as_ptr() == p);

    let c = b.clone();
    println!("2) clone vs b:   {}", c.as_ptr() == b.as_ptr());

    let d = c.clone();
    let e = d;
    println!("3) d moved to e: {}", e.as_ptr() == c.as_ptr());
}
```

<details>
<summary>Подсказка</summary>

Перемещение копирует только дескриптор. Клонирование выделяет новую память в куче. В строке 3 `e` — это перемещение `d`, но сама `d` была клоном `c`.

</details>

<details>
<summary>Ответ</summary>

```text
1) after move:   true
2) clone vs b:   false
3) d moved to e: false
```

Почему это работает: строка 1: перемещение сохраняет ту же память в куче. Строка 2: клон создаёт новый кусок, поэтому адреса различаются. Строка 3: `e` владеет памятью `d`, которая была создана клонированием `c`, то есть это другой кусок, чем у `c`.

</details>

## Вопросы для самопроверки

1. Что копируется, когда вы пишете `let t = s;` для `String`?

<details>
<summary>Ответ</summary>

Только 24-байтовый дескриптор (указатель, длина, ёмкость). Буквы в куче не копируются. Затем `s` становится непригодной.

</details>

2. Почему Rust отключает `s` после перемещения?

<details>
<summary>Ответ</summary>

Иначе и `s`, и `t` попытались бы освободить одну и ту же память в куче в конце области видимости (двойное освобождение). Когда владелец один, освобождает только `t`.

</details>

3. Чем `i32` отличается от `String` при присваивании?

<details>
<summary>Ответ</summary>

`i32` — `Copy`: он не владеет ничем, что нужно убирать, поэтому его байты просто копируются, и обе переменные остаются действительными. `String` владеет памятью в куче, поэтому перемещается.

</details>

4. Когда кортеж — `Copy`?

<details>
<summary>Ответ</summary>

Только когда `Copy` каждый его элемент. `(i32, f64)` — да; `(i32, String)` — нет.

</details>

5. Сколько стоит `.clone()` по сравнению с перемещением?

<details>
<summary>Ответ</summary>

Клон выделяет новую память в куче и копирует все данные (время и память пропорциональны размеру). Перемещение копирует только небольшой дескриптор.

</details>

6. Почему `for name in names` делает `names` непригодным и как это обычно исправляют?

<details>
<summary>Ответ</summary>

Цикл берёт вектор по значению и потребляет его, перемещая каждый элемент наружу. Напишите `for name in &names`, чтобы вместо этого заимствовать вектор.

</details>

7. Почему нельзя написать `let first = names[0];` для `Vec<String>`?

<details>
<summary>Ответ</summary>

Это переместило бы `String` из середины вектора и оставило бы дыру, что могло бы привести к двойному освобождению. Используйте `&names[0]` или `names[0].clone()`.

</details>

## Итоги

- **Перемещение** копирует только небольшой дескриптор значения и делает старую переменную недействительной; данные в куче не двигаются, и ничего не освобождается дважды.
- Типы, не владеющие ничем, что нужно убирать (числа, `bool`, `char`, `&T` и кортежи/массивы из них), — **`Copy`**: присваивание копирует их, и обе переменные остаются действительными.
- Типы, владеющие памятью в куче (`String`, `Vec`, `Box`), перемещаются.
- `.clone()` делает глубокий независимый дубликат. Он явно виден в коде и имеет реальную цену.
- Перемещения прячутся в вызовах функций, `push`, `for x in vec` и условных ветках; перемещение из индекса (`v[0]`) запрещено для элементов, не являющихся `Copy`.
- Предпочитайте заимствование клонированию и клонируйте, только когда действительно нужны два независимых владельца.

## Источники и что почитать

- [The Rust Programming Language — 4.1 What Is Ownership?](https://doc.rust-lang.org/book/ch04-01-what-is-ownership.html) — разделы «Variables and Data Interacting with Move» и «…with Clone».
- [std::marker::Copy](https://doc.rust-lang.org/std/marker/trait.Copy.html) — точные правила для `Copy` и список стандартных типов, которые его реализуют.
- [std::clone::Clone](https://doc.rust-lang.org/std/clone/trait.Clone.html) — трейт `Clone` и его связь с `Copy`.
- [Rust By Example — Ownership and moves](https://doc.rust-lang.org/rust-by-example/scope/move.html) — ещё небольшие примеры перемещения.
- [Error code E0507](https://doc.rust-lang.org/error_codes/E0507.html) — «cannot move out of borrowed content».

---

[🏠 Начало](../START-HERE.md) · [← 03.2 Правила владения](02-ownership-rules.md) · [📋 Этап 03](README.md) · [03.4 Ссылки и заимствование →](04-references-and-borrowing.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
