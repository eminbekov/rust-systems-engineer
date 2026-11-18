<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 05.13 Проект: minigrep](13-project-minigrep.md) · [📋 Этап 05](README.md) · [Этап 06 — Rust: продвинутый уровень →](../06-rust-advanced/README.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/05-rust-intermediate/14-checkpoint.md) <!-- nav -->

# Урок 05.14 — Контрольная точка

> **Этап 05 — Rust: средний уровень** · Урок 14 из 14 · ⏱ около 2,5 часа (2 занятия)

## Что вы узнаете

Этот урок не учит ничему новому. Он проверяет, что вы можете применять то, чему научил Этап 05, и показывает, куда вернуться, если что-то ещё слабо. Вы:

- Ответите на 20 вопросов об обобщённых типах, трейтах, временах жизни, замыканиях, итераторах, тестах, документации, Cargo, умных указателях, потоках, практичных крейтах, бенчмарках и Git.
- Решите 5 практических заданий, в каждом из которых есть тесты и которые объединяют несколько уроков.
- Найдёте уроки, которые стоит повторить, с прямыми ссылками.

## Перед началом

- Вы прошли [Урок 05.13 — Проект: minigrep](13-project-minigrep.md). Ваш `minigrep` работает, его тесты проходят, и он закоммичен.
- Вопросы выполняйте **с закрытой книгой**: без уроков, без компилятора, без поиска. Сначала запишите ответ на бумаге или в заметках, затем откройте скрытый ответ. Неверный ответ, найденный сейчас, обходится дёшево; та же ошибка, найденная на работе, — нет.
- Для практических заданий можно пользоваться компилятором и уроками, но сначала попробуйте без подсказок.
- Ваш учебный репозиторий чист (`git status` в `~/projects/rust` говорит `nothing to commit, working tree clean`). Работайте в ветке: `git switch -c lesson/05-14-checkpoint`.

## Новые слова

| Слово | Значение |
|---|---|
| **контрольная точка** (checkpoint) | Пауза в конце этапа, чтобы проверить себя, прежде чем идти дальше. |
| **самопроверка** (self-test) | Тест, который вы даёте себе сами и сами проверяете, чтобы найти пробелы. |
| **с закрытой книгой** (closed book) | Ответ только по памяти, без обращения к источникам. |
| **контроль качества** (quality gate) | Команды, которые должны пройти, прежде чем вы примете свою работу: `cargo fmt --check`, `cargo clippy --all-targets -- -D warnings`, `cargo test`. |
| **API** (Application Programming Interface, интерфейс программирования приложений) | Публичные функции и типы библиотеки, которыми пользуется другой код. |

## Вопросы для самопроверки

Ставьте по одному баллу за каждый ответ, который **верен и полон**. Будьте честны: проверяет только вы.

### Обобщённые типы, трейты и времена жизни

1. Что такое мономорфизация и чего она стоит?

<details>
<summary>Ответ</summary>

Компилятор делает отдельную копию обобщённой функции или типа для каждого используемого конкретного типа (`largest::<i32>`, `largest::<f64>`, …). Благодаря этому обобщённый код так же быстр, как написанный вручную (нет затрат во время работы). Платить приходится увеличенным временем компиляции и большим размером бинарного файла ([Урок 05.1](01-generics.md)).

</details>

2. В `fn largest<T: PartialOrd + Copy>(items: &[T]) -> T` зачем нужны оба ограничения?

<details>
<summary>Ответ</summary>

`PartialOrd` нужен, чтобы сравнивать два элемента через `>`. `Copy` нужен, чтобы взять значение из среза (`let mut max = items[0]`) и вернуть его, не перемещая из заимствованного среза. Компилятор проверяет тело функции, используя только возможности, названные в ограничениях ([Урок 05.1](01-generics.md)).

</details>

3. Что напечатает эта программа? (Не запускайте её.)

```rust
fn largest<T: PartialOrd + Copy>(items: &[T]) -> T {
    let mut max = items[0];
    for &item in items {
        if item > max {
            max = item;
        }
    }
    max
}

fn main() {
    println!("{}", largest(&[3, 9, 4]));
    println!("{}", largest(&[1.5, 0.2]));
    println!("{}", largest(&['x', 'b']));
}
```

<details>
<summary>Ответ</summary>

```text
9
1.5
x
```

Одна и та же функция работает для целых чисел, чисел с плавающей точкой и символов, потому что все они реализуют `PartialOrd` и `Copy`. Символы сравниваются по коду (`'x'` больше, чем `'b'`).

</details>

4. Что такое правило сирот (orphan rule) и что делать, когда оно мешает?

<details>
<summary>Ответ</summary>

Реализовать трейт для типа можно, только если трейт или тип определён в вашем собственном крейте. Поэтому нельзя написать `impl Display for Vec<i32>` (оба чужие). Обходной путь — *newtype*: собственная кортежная структура, оборачивающая чужой тип (`struct Numbers(Vec<i32>);`), для которой вы реализуете трейт ([Урок 05.2](02-traits.md)).

</details>

5. Чем обязательный метод трейта отличается от метода по умолчанию? Что делает `#[derive(Debug, Clone, PartialEq)]`?

<details>
<summary>Ответ</summary>

У обязательного метода есть только сигнатура; каждый тип, реализующий трейт, должен написать его сам. У метода по умолчанию в трейте есть тело; реализующие типы получают его бесплатно и могут переопределить. Атрибут `derive` просит компилятор написать стандартную реализацию названных трейтов для вашей структуры или перечисления ([Урок 05.2](02-traits.md)).

</details>

6. Продлевает ли аннотация времени жизни вроде `'a` жизнь значения? В чём её роль?

<details>
<summary>Ответ</summary>

Нет. Она никогда не меняет, сколько что живёт. Она лишь *описывает* связь между временами жизни входов и выходов (например, «результат действителен, пока действителен `contents`»), чтобы компилятор мог проверить, что ни одна ссылка не переживает значение, на которое указывает ([Урок 05.3](03-lifetimes.md)).

</details>

7. Этот код не компилируется. Какая ошибка и как её исправить?

```rust
fn longest(a: &str, b: &str) -> &str {
    if a.len() > b.len() { a } else { b }
}
```

<details>
<summary>Ответ</summary>

`error[E0106]: missing lifetime specifier`. Функция возвращает ссылку, и компилятор не может определить, откуда она — из `a` или из `b`. Правила пропуска (elision) не помогают, потому что входных ссылок две. Исправление: `fn longest<'a>(a: &'a str, b: &'a str) -> &'a str` ([Урок 05.3](03-lifetimes.md)).

</details>

### Замыкания и итераторы

8. Чем отличаются `Fn`, `FnMut` и `FnOnce`?

<details>
<summary>Ответ</summary>

`Fn` только читает захваченное и может вызываться много раз. `FnMut` изменяет захваченное и может вызываться много раз (нужен `mut`). `FnOnce` потребляет (перемещает наружу) захваченное и может быть вызвано только один раз. Любое замыкание как минимум `FnOnce`; просите самый слабый трейт, который нужен вашей функции ([Урок 05.4](04-closures.md)).

</details>

9. Что напечатает эта программа? (Не запускайте её.)

```rust
fn main() {
    let words = vec!["apple", "bob", "cat", "doge"];
    let n: usize = words.iter().filter(|w| w.len() > 3).map(|w| w.len()).sum();

    let mut counter = 0;
    let mut inc = || {
        counter += 2;
        counter
    };
    inc();
    inc();

    println!("{n} {counter}");
}
```

<details>
<summary>Ответ</summary>

```text
9 4
```

Слова длиннее 3 букв — `apple` (5) и `doge` (4), и 5 + 4 = 9. Замыкание `inc` заимствует `counter` изменяемо и дважды прибавляет 2, поэтому `counter` равен 4. Изменяемое заимствование заканчивается после последнего использования `inc`, так что `println!` может прочитать `counter`.

</details>

10. Что напечатает эта программа? (Не запускайте её.)

```rust
fn main() {
    let lazy = (1..4).map(|x| {
        println!("working on {x}");
        x * 10
    });
    println!("created the iterator");
    let total: i32 = lazy.sum();
    println!("total = {total}");
}
```

<details>
<summary>Ответ</summary>

```text
created the iterator
working on 1
working on 2
working on 3
total = 60
```

Адаптеры итераторов вроде `map` ленивы: при создании конвейера ничего не происходит. Работа начинается, когда потребитель (`sum`) запрашивает элементы ([Урок 05.5](05-iterators.md)).

</details>

11. Чем отличаются `iter()`, `iter_mut()` и `into_iter()` для `Vec<String>`?

<details>
<summary>Ответ</summary>

`iter()` даёт `&String` (общие заимствования; вектором можно пользоваться дальше). `iter_mut()` даёт `&mut String` (элементы можно менять). `into_iter()` даёт `String` и потребляет вектор (он перемещается) ([Урок 05.5](05-iterators.md)).

</details>

### Тесты, документация и Cargo

12. Чем отличаются модульный тест, интеграционный тест и doc-тест? Где живёт каждый? Почему интеграционному тесту нужен `lib.rs`?

<details>
<summary>Ответ</summary>

Модульный тест лежит в том же файле, в `#[cfg(test)] mod tests`, и видит приватные элементы. Интеграционный тест — файл в `tests/`, компилируется как отдельный крейт и видит только публичный API. Doc-тест — блок кода в комментарии `///`, который `cargo test` компилирует и запускает. Интеграционный тест может подключить через `use` только крейт-библиотеку; крейт-бинарник он импортировать не может, поэтому логика должна быть в `src/lib.rs` ([Урок 05.6](06-testing-in-depth.md), [Урок 05.7](07-documentation.md)).

</details>

13. Какие версии принимает `serde = "1.0.200"`? А `itertools = "0.15"`? Нужно ли хранить `Cargo.lock` программы в Git?

<details>
<summary>Ответ</summary>

`"1.0.200"` принимает версии от `1.0.200` до `2.0.0` (не включая). `"0.15"` принимает от `0.15.0` до `0.16.0` (не включая) (у крейтов `0.x` «ломающим» числом является среднее). Да, коммитьте `Cargo.lock` для программ, чтобы все собирали с одинаковыми версиями; `target/` не коммитьте никогда ([Урок 05.8](08-cargo-dependencies-and-versions.md)).

</details>

14. Функция из документации крейта «не найдена», хотя крейт есть в `Cargo.toml`. Какова самая вероятная причина и как её исправить?

<details>
<summary>Ответ</summary>

Функция спрятана за *фичей* Cargo, которая не включена (`error[E0599]: no function found …`). Прочитайте список фич на docs.rs или в `cargo info <crate>`, затем добавьте `features = ["name"]` к зависимости (`cargo add <crate> --features name`) ([Урок 05.8](08-cargo-dependencies-and-versions.md)).

</details>

### Умные указатели и потоки

15. Выберите правильный тип: (а) рекурсивное перечисление `List`; (б) узел графа с несколькими владельцами в одном потоке; (в) счётчик, который увеличивают восемь потоков; (г) ссылка на родителя в дереве, где потомками владеет родитель.

<details>
<summary>Ответ</summary>

(а) `Box<List>`: указатель фиксированного размера разрывает бесконечный размер. (б) `Rc<T>`, а если узел должен меняться — `Rc<RefCell<T>>`. (в) `Arc<Mutex<u64>>` (или `AtomicU64` для простого счётчика). (г) `Weak<T>` для ссылки вверх, чтобы не было цикла ссылок ([Урок 05.9](09-smart-pointers.md), [Урок 05.10](10-threads-channels-and-shared-state.md)).

</details>

16. Что напечатает эта программа? (Не запускайте её.)

```rust
use std::cell::RefCell;
use std::rc::Rc;

fn main() {
    let a = Rc::new(RefCell::new(vec![1]));
    let b = Rc::clone(&a);
    b.borrow_mut().push(2);
    println!("{:?} {}", a.borrow(), Rc::strong_count(&a));
}
```

<details>
<summary>Ответ</summary>

```text
[1, 2] 2
```

`a` и `b` — два владельца одного вектора, поэтому добавление через `b` видно через `a`, а сильный счётчик равен 2. `RefCell` позволяет изменение за общим `Rc`; его правила проверяются во время работы ([Урок 05.9](09-smart-pointers.md)).

</details>

17. Почему замыкание, переданное в `thread::spawn`, обычно должно быть `move`? Почему `Rc` нельзя отправить в поток? Что такое взаимная блокировка и как её избежать?

<details>
<summary>Ответ</summary>

Запущенный поток может пережить функцию, которая его создала, поэтому он должен владеть своими данными (`move`); иначе ошибка `E0373`. У `Rc` неатомарный счётчик, он не `Send`, и компилятор его отвергает (`E0277`); используйте `Arc`. Взаимная блокировка — это когда два потока или больше вечно ждут блокировки друг друга. Избегайте её: берите блокировки везде в одном порядке, держите их недолго и не вызывайте неизвестный код, пока блокировка удерживается ([Урок 05.10](10-threads-channels-and-shared-state.md)).

</details>

### Практичные крейты, бенчмарки и Git

18. Какой крейт вы возьмёте для каждой задачи и почему: (а) превратить структуру в JSON; (б) разобрать `--count 3`; (в) тип ошибки библиотеки; (г) тип ошибки на верхнем уровне приложения? Что делает `#[from]`?

<details>
<summary>Ответ</summary>

(а) `serde` с `serde_json` (`#[derive(Serialize, Deserialize)]`). (б) `clap` (`#[derive(Parser)]`). (в) `thiserror`: точное перечисление, которое вызывающий код может разбирать. (г) `anyhow`: один универсальный тип ошибки с `.context(...)`. `#[from]` реализует `From<SourceError>` для вашего перечисления, чтобы `?` преобразовывал автоматически, и запоминает исходную ошибку ([Урок 05.11](11-practical-crates-serde-clap-errors.md)).

</details>

19. Бенчмарк показывает `41 ns` для цикла в 200 миллионов итераций. Что произошло и каковы четыре правила честного измерения?

<details>
<summary>Ответ</summary>

Компилятор удалил работу (результат нигде не использовался) или вычислил её при компиляции, поэтому измерено было только чтение часов. Правила: используйте `--release` (или `cargo bench`); применяйте `black_box` для входов и результатов; запускайте много раз с прогревом и смотрите на медиану и разброс; меняйте одну вещь за раз и сравнивайте с сохранённой базовой линией; держите тест на правильность; записывайте окружение ([Урок 05.12](12-benchmarking-basics.md)).

</details>

20. В `minigrep` вы ускорили вывод в 16 раз. Какие шаги Git делают изменение безопасным и доказуемым? Каковы три статуса завершения инструмента?

<details>
<summary>Ответ</summary>

Работайте в ветке (`git switch -c perf/buffered-output`), меняйте одну вещь, запускайте тесты, измеряйте до и после одной и той же командой, коммитьте с числами в сообщении (`perf(minigrep): buffer the output (734 ms -> 45 ms)`), сливайте, удаляйте ветку и ставьте тег релиза (`git tag -a minigrep-v0.1.0`). Статусы завершения: `0` (что-то выбрано, ошибок нет), `1` (ничего не выбрано), `2` (любая ошибка) ([Урок 05.13](13-project-minigrep.md)).

</details>

**Ваш результат:** посчитайте верные ответы. 18–20: отлично. 14–17: хорошо, повторите уроки по вопросам, где вы ошиблись (таблица ниже). Меньше 14: повторите слабые уроки и переделайте их упражнения до Этапа 06.

## Практические задания

Создайте один учебный пакет для заданий: `cd ~/projects/rust && git switch -c lesson/05-14-checkpoint && cargo new --lib code/p05-checkpoint`. Добавьте зависимости: `cargo add serde@1 --features derive`, `cargo add serde_json@1 thiserror@2 anyhow@1`. Каждое задание пишите сначала с тестов: тесты, потом код. После каждого задания запускайте контроль качества (`cargo fmt --check`, `cargo clippy --all-targets -- -D warnings`, `cargo test`) и коммитьте через `git add . && git commit -m "feat(p05): …"`.

### Задание 1 — Обобщённый стек с Display

Напишите `Stack<T>` (структура с `Vec<T>` внутри) с методами `new`, `push`, `pop` (возвращает `Option<T>`), `peek` (возвращает `Option<&T>`), `len` и `is_empty`. Добавьте `Debug` и `Default`. Реализуйте `Display` для `Stack<T>` при `T: Display`: элементы от дна к вершине в квадратных скобках, через запятую и пробел (`[a, b]`; пустой стек — `[]`). Тесты: последним пришёл — первым вышел; `peek`; вывод пустого и заполненного стека.

<details>
<summary>Подсказка</summary>

`impl<T> Stack<T> { … }` для методов и отдельный `impl<T: fmt::Display> fmt::Display for Stack<T>` для печати. В `fmt` используйте `write!(f, "[")?;`, затем цикл с `enumerate()`, записывая `", "` перед каждым элементом, кроме первого, затем `write!(f, "]")`. `Vec::last()` даёт `Option<&T>`.

</details>

<details>
<summary>Ответ</summary>

```rust
use std::fmt;

#[derive(Debug, Default)]
pub struct Stack<T> {
    items: Vec<T>,
}

impl<T> Stack<T> {
    pub fn new() -> Self {
        Stack { items: Vec::new() }
    }

    pub fn push(&mut self, item: T) {
        self.items.push(item);
    }

    pub fn pop(&mut self) -> Option<T> {
        self.items.pop()
    }

    pub fn peek(&self) -> Option<&T> {
        self.items.last()
    }

    pub fn len(&self) -> usize {
        self.items.len()
    }

    pub fn is_empty(&self) -> bool {
        self.items.is_empty()
    }
}

impl<T: fmt::Display> fmt::Display for Stack<T> {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        write!(f, "[")?;
        for (i, item) in self.items.iter().enumerate() {
            if i > 0 {
                write!(f, ", ")?;
            }
            write!(f, "{item}")?;
        }
        write!(f, "]")
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn stack_is_last_in_first_out() {
        let mut s = Stack::new();
        assert!(s.is_empty());
        s.push(1);
        s.push(2);
        s.push(3);
        assert_eq!(s.len(), 3);
        assert_eq!(s.peek(), Some(&3));
        assert_eq!(s.pop(), Some(3));
        assert_eq!(s.pop(), Some(2));
        assert_eq!(s.pop(), Some(1));
        assert_eq!(s.pop(), None);
    }

    #[test]
    fn stack_displays_bottom_to_top() {
        let mut s = Stack::new();
        assert_eq!(s.to_string(), "[]");
        s.push("a");
        s.push("b");
        assert_eq!(s.to_string(), "[a, b]");
        let words: Stack<String> = Stack::default();
        assert_eq!(format!("{words}"), "[]");
    }
}
```

Почему это работает: методам не нужны ограничения, потому что они только перемещают значения. `Display` нужен `T: Display`, потому что он печатает элементы. `#[derive(Default)]` у обобщённой структуры добавляет ограничение `T: Default` к сгенерированной реализации, поэтому `Stack::<String>::default()` работает, ведь `String` реализует `Default`.

</details>

### Задание 2 — Времена жизни и итераторы

Напишите две функции, которые не создают новых строк для результатов: `longest_line(text: &str) -> Option<&str>` (самая длинная строка или `None` для пустого текста) и `parse_pairs(text: &str) -> Vec<(&str, &str)>`. Вторая делит каждую строку по первому `=`, обрезает обе части и пропускает строки без `=` или с пустым ключом. Используйте адаптеры итераторов. Проверьте на тексте `"name = Ana\nbroken line\n= no key\nage=30\n"`; ожидаемый результат — `[("name", "Ana"), ("age", "30")]`.

<details>
<summary>Подсказка</summary>

`text.lines().max_by_key(|line| line.len())` для первой (пропущенное время жизни связывает результат с `text`). Для второй: `text.lines().filter_map(|line| …).collect()`, а внутри замыкания используйте `line.split_once('=')?` (`?` работает с `Option` внутри замыкания, возвращающего `Option`).

</details>

<details>
<summary>Ответ</summary>

```rust
pub fn longest_line(text: &str) -> Option<&str> {
    text.lines().max_by_key(|line| line.len())
}

pub fn parse_pairs(text: &str) -> Vec<(&str, &str)> {
    text.lines()
        .filter_map(|line| {
            let (key, value) = line.split_once('=')?;
            let key = key.trim();
            if key.is_empty() {
                return None;
            }
            Some((key, value.trim()))
        })
        .collect()
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn finds_the_longest_line() {
        assert_eq!(longest_line("ab\nabcd\nabc"), Some("abcd"));
        assert_eq!(longest_line(""), None);
    }

    #[test]
    fn parses_key_value_lines() {
        let text = "name = Ana\nbroken line\n= no key\nage=30\n";
        assert_eq!(parse_pairs(text), vec![("name", "Ana"), ("age", "30")]);
    }
}
```

Почему это работает: при одной входной ссылке правило пропуска даёт результату то же время жизни, что и у `text`, поэтому аннотация не нужна, а возвращаемые значения `&str` — срезы входа (без копирования). `split_once` возвращает `Option<(&str, &str)>`, `trim` возвращает подсрез, а `filter_map` отбрасывает строки с `None`.

</details>

### Задание 3 — Потоки и каналы

Напишите `word_frequencies(texts: Vec<String>) -> BTreeMap<String, usize>`. Запустите по одному потоку на каждый текст. Каждый поток считает свои слова (в нижнем регистре) в собственной `HashMap` и **отправляет готовую карту** через канал. Главный поток получает все карты и сливает их в одну `BTreeMap`. Проверьте на `"The cat and the dog"` и `"a cat"`: `the` = 2, `cat` = 2, `dog` = 1 и 5 различных слов.

<details>
<summary>Подсказка</summary>

Создайте `let (tx, rx) = mpsc::channel::<HashMap<String, usize>>();`, дайте каждому потоку `tx.clone()` в замыкании `move`, затем `drop(tx)`, чтобы канал закрылся, и переберите `for partial in rx`. Сливайте через `*total.entry(word).or_insert(0) += n;`.

</details>

<details>
<summary>Ответ</summary>

```rust
use std::collections::{BTreeMap, HashMap};
use std::sync::mpsc;
use std::thread;

pub fn word_frequencies(texts: Vec<String>) -> BTreeMap<String, usize> {
    let (tx, rx) = mpsc::channel::<HashMap<String, usize>>();

    for text in texts {
        let tx = tx.clone();
        thread::spawn(move || {
            let mut counts = HashMap::new();
            for word in text.split_whitespace() {
                *counts.entry(word.to_lowercase()).or_insert(0) += 1;
            }
            tx.send(counts).unwrap();
        });
    }
    drop(tx);

    let mut total = BTreeMap::new();
    for partial in rx {
        for (word, n) in partial {
            *total.entry(word).or_insert(0) += n;
        }
    }
    total
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn counts_words_with_channels() {
        let texts = vec![
            String::from("The cat and the dog"),
            String::from("a cat"),
        ];
        let counts = word_frequencies(texts);
        assert_eq!(counts["the"], 2);
        assert_eq!(counts["cat"], 2);
        assert_eq!(counts["dog"], 1);
        assert_eq!(counts.len(), 5);
    }
}
```

Почему это работает: пока потоки работают, никакие данные не делятся; каждый владеет своей картой, а владение перемещается через канал (передача сообщений). Исходный `tx` уничтожается, поэтому цикл получателя заканчивается, когда завершился последний поток. Тест не зависит от порядка прихода, потому что слияние даёт одно и то же при любом порядке. `BTreeMap` к тому же даёт отсортированный вывод.

</details>

### Задание 4 — Конфигурация с serde, thiserror и anyhow

Определите `Config { name: String, port: u16, verbose: bool }`, которую можно прочитать из JSON (`verbose` необязателен и по умолчанию `false`). Напишите `parse_config(text: &str) -> Result<Config, ConfigError>`, где `ConfigError` (используйте `thiserror`) имеет три варианта: неверный JSON (с `#[from]`), пустое или состоящее из пробелов имя и порт меньше 1024 (сообщение `port 80 is not allowed (use 1024 to 65535)`). Затем напишите `describe(text: &str) -> anyhow::Result<String>`, которая возвращает `"web on port 8080"` для хорошего входа и добавляет к ошибкам контекст `cannot use the configuration`. Проверьте каждый вариант ошибки и форму `{:#}` ошибки из `describe`.

<details>
<summary>Подсказка</summary>

`#[derive(Deserialize, PartialEq, Debug)]` и `#[serde(default)]` у `verbose`. Для теста варианта с неверным JSON используйте `matches!(parse_config("not json"), Err(ConfigError::Json(_)))`. `describe` — это `parse_config(text).context("cannot use the configuration")?`, а затем `format!`.

</details>

<details>
<summary>Ответ</summary>

```rust
use anyhow::Context;
use serde::Deserialize;
use thiserror::Error;

#[derive(Debug, Deserialize, PartialEq)]
pub struct Config {
    pub name: String,
    pub port: u16,
    #[serde(default)]
    pub verbose: bool,
}

#[derive(Debug, Error)]
pub enum ConfigError {
    #[error("the configuration is not valid JSON")]
    Json(#[from] serde_json::Error),
    #[error("the name must not be empty")]
    EmptyName,
    #[error("port {0} is not allowed (use 1024 to 65535)")]
    BadPort(u16),
}

pub fn parse_config(text: &str) -> Result<Config, ConfigError> {
    let config: Config = serde_json::from_str(text)?;
    if config.name.trim().is_empty() {
        return Err(ConfigError::EmptyName);
    }
    if config.port < 1024 {
        return Err(ConfigError::BadPort(config.port));
    }
    Ok(config)
}

pub fn describe(text: &str) -> anyhow::Result<String> {
    let config = parse_config(text).context("cannot use the configuration")?;
    Ok(format!("{} on port {}", config.name, config.port))
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn config_is_checked() {
        let ok = parse_config(r#"{"name":"web","port":8080}"#).unwrap();
        assert_eq!(ok, Config { name: "web".into(), port: 8080, verbose: false });

        assert!(matches!(parse_config("not json"), Err(ConfigError::Json(_))));
        assert!(matches!(
            parse_config(r#"{"name":" ","port":8080}"#),
            Err(ConfigError::EmptyName)
        ));
        let err = parse_config(r#"{"name":"web","port":80}"#).unwrap_err();
        assert_eq!(err.to_string(), "port 80 is not allowed (use 1024 to 65535)");
    }

    #[test]
    fn describe_adds_context() {
        assert_eq!(
            describe(r#"{"name":"web","port":8080}"#).unwrap(),
            "web on port 8080"
        );
        let err = describe(r#"{"name":"web","port":80}"#).unwrap_err();
        assert_eq!(
            format!("{err:#}"),
            "cannot use the configuration: port 80 is not allowed (use 1024 to 65535)"
        );
    }
}
```

Почему это работает: `serde` проверяет форму (отсутствующий `port` или текст там, где ждут число, — ошибка `Json`), а ваш код проверяет смысл (пустое имя, зарезервированный порт). `thiserror` даёт библиотечной части точные варианты; `anyhow` наверху добавляет контекст, а `{:#}` печатает всю цепочку в одной строке. Макрос `matches!` проверяет вариант, не требуя `PartialEq` у типа ошибки.

</details>

### Задание 5 — Измерение и релиз с Git (без нового кода)

Возьмите свой `minigrep` из Урока 05.13 и докажите, что работа завершена и воспроизводима. Сначала переключитесь на ветку, где он лежит (`git switch project/05-minigrep` или `main`, если вы уже слили), затем:

1. Запустите контроль качества и `cargo test` в папке проекта. Всё должно проходить без предупреждений.
2. Проверьте свою историю: `git log --oneline` показывает один коммит на веху и один коммит `perf` с числами «до» и «после» в сообщении.
3. Покажите, что релиз помечен тегом и тег указывает на нужный коммит.
4. Убедитесь, что бенчмарк, интеграционные тесты и README находятся в репозитории (`git ls-files code/p05-minigrep`), а папки `target` там нет.
5. Напишите три строки в учебных заметках: самое медленное место, которое вы нашли, как вы доказали улучшение и какую дополнительную цель хотите попробовать.

<details>
<summary>Подсказка</summary>

`git tag` перечисляет теги; `git log --oneline --decorate` показывает, у какого коммита есть тег. Если вы забыли тег, добавьте его: `git tag -a minigrep-v0.1.0 -m "minigrep 0.1.0"`. `git ls-files` перечисляет файлы, которые отслеживает Git; `target/` там быть не должно.

</details>

<details>
<summary>Ответ</summary>

```bash
cd ~/projects/rust/code/p05-minigrep
cargo fmt --check && cargo clippy --all-targets -- -D warnings && cargo test
git log --oneline --decorate -4
git tag
git ls-files .
```

Аккуратная история выглядит так (идентификаторы на вашем компьютере будут другими):

```text
08bfc4e (HEAD -> project/05-minigrep, tag: minigrep-v0.1.0) perf(minigrep): buffer the output (734 ms -> 45 ms)
0cd2c44 feat(minigrep): add the search function
5f38302 feat(minigrep): parse the command line
```

`git tag` печатает `minigrep-v0.1.0`. `git ls-files .` перечисляет `Cargo.toml`, `Cargo.lock`, `README.md`, `benches/search.rs`, `src/lib.rs`, `src/main.rs` и `tests/cli.rs`, и ни одного файла из `target/`. *Аннотированный* тег (`-a`) хранит, кто его создал, когда и с каким сообщением; `git show minigrep-v0.1.0` показывает это.

Почему это работает: воспроизводимый проект любой человек может собрать из одного репозитория (`Cargo.lock` внутри, `target/` снаружи), он доказывает своё поведение (тесты) и показывает историю решений (один коммит на веху, числа в коммите про производительность). Именно это превращает маленький инструмент в инженерную работу.

</details>

## Если что-то не получилось

| Тема | Какой урок повторить |
|---|---|
| Обобщённые функции и типы, ограничения трейтов, мономорфизация | [Урок 05.1 — Обобщённые типы](01-generics.md) |
| Трейты, `derive`, `Display`, `From`, правило сирот | [Урок 05.2 — Трейты](02-traits.md) |
| Аннотации времён жизни, пропуск, структуры со ссылками | [Урок 05.3 — Времена жизни](03-lifetimes.md) |
| Замыкания, `move`, `Fn` / `FnMut` / `FnOnce` | [Урок 05.4 — Замыкания](04-closures.md) |
| Итераторы, адаптеры, потребители, `collect` | [Урок 05.5 — Итераторы](05-iterators.md) |
| TDD, модульные и интеграционные тесты, параметры тестов | [Урок 05.6 — Тестирование подробно](06-testing-in-depth.md) |
| Doc-комментарии, doc-тесты, README, журнал изменений | [Урок 05.7 — Документация](07-documentation.md) |
| Требования к версиям, `Cargo.lock`, фичи, `cargo tree` | [Урок 05.8 — Cargo: зависимости и версии](08-cargo-dependencies-and-versions.md) |
| `Box`, `Rc`, `RefCell`, `Weak`, `Drop`, `Deref` | [Урок 05.9 — Умные указатели](09-smart-pointers.md) |
| Потоки, каналы, `Arc<Mutex<T>>`, `Send` / `Sync` | [Урок 05.10 — Потоки, каналы и общее состояние](10-threads-channels-and-shared-state.md) |
| `serde`, `clap`, `thiserror`, `anyhow` | [Урок 05.11 — Практичные крейты: serde, clap, thiserror и anyhow](11-practical-crates-serde-clap-errors.md) |
| Режим release, `black_box`, `criterion`, `hyperfine` | [Урок 05.12 — Основы бенчмарков](12-benchmarking-basics.md) |
| Библиотека и тонкий бинарник, статус завершения, интеграционные тесты CLI | [Урок 05.13 — Проект: minigrep](13-project-minigrep.md) |
| Владение, заимствование и ссылки (за многими ошибками здесь) | [Урок 03.5 — Изменяемые ссылки и правила заимствования](../03-ownership-and-borrowing/05-mutable-references-and-the-borrow-rules.md) |
| `Result`, `?` и собственные ошибки | [Урок 04.10 — Обработка ошибок](../04-structs-enums-collections-errors/10-error-handling.md) |
| Ветки, слияние, отмена изменений | [Урок 01.6 — Ветки и слияние](../01-git-and-version-control/06-branches-and-merging.md) и [Урок 01.5 — Как безопасно отменять изменения](../01-git-and-version-control/05-undoing-changes-safely.md) |

Когда вы повторяете урок, не просто перечитывайте его. Переделайте его упражнения в ветке и сначала пишите тесты.

## Что дальше

Вы завершили Этап 05. Теперь вы можете писать обобщённый, протестированный, документированный код; пользоваться распространёнными крейтами; безопасно делить работу между потоками; и измерять скорость. Прежде чем идти дальше:

1. Слейте ветку контрольной точки (`git switch main && git merge lesson/05-14-checkpoint`) и удалите завершённые ветки уроков, которые вам больше не нужны (`git branch -d <name>`).
2. Добавьте несколько строк в учебные заметки: что было трудным в этом этапе, что вы хотите запомнить и какие дополнительные цели `minigrep` вы планируете сделать.
3. Сделайте перерыв. День отдыха помогает памяти больше, чем ещё час чтения.

На **Этапе 06 — Rust: продвинутый уровень** вы углубитесь в те части языка, которых коснулись здесь: трейт-объекты и динамическая диспетчеризация (`Box<dyn Trait>`), продвинутые трейты и образцы, внутренняя изменяемость подробно, времена жизни подробно, небезопасный код и Miri, макросы, `async`/`await` и вызов кода на C. Этап заканчивается многопоточным веб-сервером и контрольной точкой стадии 1. Начните с [Урока 06.1 — Трейт-объекты и динамическая диспетчеризация](../06-rust-advanced/01-trait-objects-and-dynamic-dispatch.md).

## Источники и что почитать

- [The Rust Programming Language — Chapter 10: Generic Types, Traits, and Lifetimes](https://doc.rust-lang.org/book/ch10-00-generics.html) — официальная глава, на которой стоят Уроки 05.1–05.3.
- [The Rust Programming Language — Chapter 13: Functional Language Features](https://doc.rust-lang.org/book/ch13-00-functional-features.html) — замыкания и итераторы.
- [The Rust Programming Language — Chapter 15: Smart Pointers](https://doc.rust-lang.org/book/ch15-00-smart-pointers.html) и [Chapter 16: Fearless Concurrency](https://doc.rust-lang.org/book/ch16-00-concurrency.html) — Уроки 05.9 и 05.10.
- [The Cargo Book](https://doc.rust-lang.org/cargo/) — полный справочник по зависимостям, фичам, профилям и тестам.
- [Rust by Example](https://doc.rust-lang.org/rust-by-example/) — короткие запускаемые примеры по всем темам; хорошо для практики.
- [Pro Git](https://git-scm.com/book/en/v2) — бесплатная официальная книга по Git; главы 2 (основы) и 3 (ветвление) покрывают ваши ежедневные команды.

---

[🏠 Начало](../START-HERE.md) · [← 05.13 Проект: minigrep](13-project-minigrep.md) · [📋 Этап 05](README.md) · [Этап 06 — Rust: продвинутый уровень →](../06-rust-advanced/README.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
