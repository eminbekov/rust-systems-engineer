<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 20.2 Ошибки и паники в продакшене](02-errors-and-panics-in-production.md) · [📋 Этап 20](README.md) · [20.4 Miri, санитайзеры и loom в CI →](04-miri-sanitizers-and-loom-in-ci.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/20-reliability-and-hardening/03-fuzzing-in-depth.md) <!-- nav -->

# Урок 20.3 — Фаззинг подробно

> **Этап 20 — Надёжность и защита** · Урок 3 из 11 · ⏱ около 135 минут

## Что вы узнаете

- Как фаззить целую *последовательность операций* и сравнивать ваш код с простой моделью, чтобы фаззер находил неверные ответы, а не только аварийные завершения.
- Почему выведенный через derive вход `Arbitrary` может атаковать фаззинг-цель вместо вашего кода и как придать входу форму с помощью собственной реализации `Arbitrary`.
- Как работать с корпусом: начальные входы (seeds), словари и минимизация корпуса через `cargo fuzz cmin`.
- Как устроен непрерывный фаззинг: короткие прогоны на каждом pull request, долгие прогоны каждую ночь и корпус, который сохраняется между прогонами.

## Перед началом

- Вы прошли [Урок 20.2 — Ошибки и паники в продакшене](02-errors-and-panics-in-production.md).
- Вы знаете основы фаззинга из [Урока 09.8 — Фаззинг](../09-testing-debugging-benchmarking/08-fuzzing.md): `cargo fuzz init`, фаззинг-цель, `cargo +nightly fuzz run`, файлы аварий, `tmin` и крейт `arbitrary` с `#[derive(Arbitrary)]`. Этот урок опирается на всё это.
- Вы знаете запланированный workflow фаззинга из [Урока 09.12 — Непрерывная интеграция](../09-testing-debugging-benchmarking/12-continuous-integration.md) и `VecDeque` из [Урока 07.5 — Стеки и очереди](../07-algorithms-and-data-structures/05-stacks-and-queues.md).
- Создайте пакет для этого урока (нужны nightly-тулчейн и `cargo-fuzz` из Урока 09.8):

```bash
cd ~/projects/rust
cargo new --lib code/p20-fuzz
cd code/p20-fuzz
cargo fuzz init -t ops
cargo add arbitrary@1 --features derive --manifest-path fuzz/Cargo.toml
```

`cargo fuzz init -t ops` создаёт пакет `fuzz/` с одной целью по имени `ops`. Последняя команда добавляет `arbitrary` в пакет *фаззинга*, а не в вашу библиотеку.

## Новые слова

| Слово | Значение |
|---|---|
| **модель** (model, test oracle) | Вторая, очень простая и очевидно правильная реализация, которая нужна только для проверки настоящей. Здесь это `VecDeque`. |
| **дифференциальный фаззинг** (differential fuzzing) | Фаззинг двух реализаций одним и тем же входом с падением, когда их ответы расходятся. |
| **фаззинг с состоянием** (stateful fuzzing) | Фаззинг последовательности операций над объектом (push, pop, …), а не одного вызова функции. |
| **`Unstructured`** | Тип из крейта `arbitrary`, который оборачивает сырые байты фаззера и выдаёт их по частям для построения значений. |
| **начальный вход** (seed) | Написанный вручную вход, который вы кладёте в корпус, чтобы фаззер начинал с чего-то осмысленного. |
| **словарь** (dictionary) | Файл байтовых строк (ключевые слова, «магические» числа), которые фаззер может вставлять во входы. |
| **минимизация корпуса** (corpus minimization, `cmin`) | Удаление файлов корпуса, которые не добавляют нового покрытия, чтобы корпус оставался маленьким и быстро воспроизводился. |
| **непрерывный фаззинг** (continuous fuzzing) | Автоматический и регулярный запуск фаззеров (на каждое изменение, каждую ночь) с сохранением корпуса между прогонами. |
| **OSS-Fuzz** (Open Source Software Fuzz, фаззинг программ с открытым кодом) | Бесплатный сервис Google, который круглосуточно фаззит важные проекты с открытым кодом. |
| **OOM** (Out Of Memory, нехватка памяти) | Отказ, при котором программа запрашивает больше памяти, чем позволяет система. |

## 1. Не только «падает ли?»

В Уроке 09.8 у фаззинг-цели была одна задача: вызвать парсер и посмотреть, не паникует ли он. Утверждение о полном цикле (round trip) было первым шагом дальше. Здесь мы идём до конца: фаззер генерирует **список операций**, выполняет их над настоящим кодом и над **моделью** и после каждого шага проверяет, что оба согласны. Так находятся *неверные ответы* — тихие отказы из Урока 20.1.

Наш код — буфер, который хранит только самые новые значения, как сервис хранит «последние 100 строк лога» для отладочной страницы. Положите это в `src/lib.rs`:

```rust
//! A buffer that keeps only the newest values, for example the last log lines of a service.

use std::num::NonZeroUsize;

/// Keeps at most `capacity` values. When it is full, a new value replaces the oldest one.
#[derive(Debug, Clone)]
pub struct RecentBuffer {
    slots: Vec<u32>,
    head: usize, // index of the oldest value
    len: usize,
}

impl RecentBuffer {
    pub fn new(capacity: NonZeroUsize) -> Self {
        RecentBuffer {
            slots: vec![0; capacity.get()],
            head: 0,
            len: 0,
        }
    }

    pub fn push(&mut self, value: u32) {
        let capacity = self.slots.len();
        let tail = (self.head + self.len) % capacity;
        self.slots[tail] = value;
        if self.len < capacity {
            self.len += 1;
        }
        // BUG: when the buffer was full, `value` replaced the oldest value,
        // so `head` must move to the next oldest one. It does not.
    }

    pub fn pop_oldest(&mut self) -> Option<u32> {
        if self.len == 0 {
            return None;
        }
        let value = self.slots[self.head];
        self.head = (self.head + 1) % self.slots.len();
        self.len -= 1;
        Some(value)
    }

    pub fn len(&self) -> usize {
        self.len
    }

    pub fn is_empty(&self) -> bool {
        self.len == 0
    }
}
```

**Что происходит:**

1. `NonZeroUsize` (из `std::num`) — это `usize`, который никогда не равен нулю. Если принимать ёмкость в таком типе, «буфер, в который ничего не влезает», становится невозможным, поэтому `% capacity` никогда не делит на ноль.
2. Значения лежат в `slots`, используемом как кольцевой буфер (Урок 07.5): `head` указывает на самое старое значение, а следующая свободная ячейка — `head + len`, завёрнутая по кругу через `%`.
3. Комментарий отмечает подложенный баг. Тесты, которые никогда не заполняют буфер полностью, его не заметят. Добавьте два таких теста сами (Попробуйте сами, шаг 1): они проходят.

## 2. Фаззинг-цель с состоянием и моделью

Замените `fuzz/fuzz_targets/ops.rs`:

```rust
#![no_main]
//! Property: RecentBuffer behaves exactly like a simple model built on VecDeque.

use std::collections::VecDeque;
use std::num::NonZeroUsize;

use arbitrary::Arbitrary;
use libfuzzer_sys::fuzz_target;
use p20_fuzz::RecentBuffer;

#[derive(Debug, Arbitrary)]
enum Op {
    Push(u32),
    Pop,
}

#[derive(Debug, Arbitrary)]
struct Scenario {
    capacity: NonZeroUsize,
    ops: Vec<Op>,
}

fuzz_target!(|scenario: Scenario| {
    let capacity = scenario.capacity.get();
    let mut real = RecentBuffer::new(scenario.capacity);
    let mut model: VecDeque<u32> = VecDeque::new();

    for op in scenario.ops {
        match op {
            Op::Push(value) => {
                real.push(value);
                if model.len() == capacity {
                    model.pop_front();
                }
                model.push_back(value);
            }
            Op::Pop => assert_eq!(real.pop_oldest(), model.pop_front()),
        }
        assert_eq!(real.len(), model.len());
    }
});
```

**Что происходит:**

1. `Op` перечисляет операции; `#[derive(Arbitrary)]` позволяет фаззеру выбрать из своих байтов один вариант (и `u32` для `Push`).
2. `Scenario` — это ёмкость плюс список операций любой длины.
3. Модель — `VecDeque` с правилом, записанным самым очевидным способом: «если полон, выбросить первый, затем добавить в конец». Для продакшена она слишком медленная и слишком простая, и именно поэтому мы ей доверяем.
4. После каждой операции обе стороны должны дать одинаковый ответ и одинаковую длину. Первое расхождение — упавший `assert_eq!`, о котором фаззер сообщает как об аварии.

Запустите его максимум на 30 секунд:

```bash
cargo +nightly fuzz run ops -- -max_total_time=30
```

```text
INFO: A corpus is not provided, starting from an empty corpus
...
==4575==ERROR: AddressSanitizer: requested allocation size 0x1c00000002c2c (0x1c00000003c30 after adjustments for alignment, red zones etc.) exceeds maximum supported size of 0x10000000000 (thread T0)
...
Output of `std::fmt::Debug`:

	Scenario {
	    capacity: 123145302313739,
	    ops: [],
	}
```

(Сокращено; ваши числа будут другими.) Фаззер нашёл проблему меньше чем за секунду, но не ту, которую мы подложили. Он выбрал ёмкость в 123 триллиона, и `vec![0; capacity]` запросил сотни терабайт памяти. AddressSanitizer (проверка памяти, которую включает `cargo fuzz`, Урок 09.8) остановил выделение как **OOM**: нехватку памяти.

Это баг в `RecentBuffer`? Не совсем: вызывающий, который просит буфер на 123 триллиона значений, получает то, что просил. Это слабость нашей **фаззинг-цели**: выведенный `Arbitrary` выдаёт *любой* возможный `usize`, и почти все они бесполезны для проверки кольцевого буфера. Входу нужно придать форму.

> 🔒 **Безопасность:** если бы ёмкость приходила из файла конфигурации или запроса, это *была бы* настоящая проблема: одно значение могло бы заставить сервис занять всю память. Это урок о «недоверенных размерах» из 10.12 и о лимитах из Урока 20.8.

## 3. Придаём входу форму собственным `Arbitrary`

У трейта `Arbitrary` один обязательный метод: построить значение из `Unstructured`. `Unstructured` оборачивает байты фаззера и предлагает помощников, которые берут байты с начала. Замените строку `use arbitrary::Arbitrary;` и тип `Scenario` в `ops.rs`:

```rust
use arbitrary::{Arbitrary, Unstructured};
```

```rust
#[derive(Debug)]
struct Scenario {
    capacity: NonZeroUsize,
    ops: Vec<Op>,
}

impl<'a> Arbitrary<'a> for Scenario {
    fn arbitrary(u: &mut Unstructured<'a>) -> arbitrary::Result<Self> {
        // A small capacity, 1 to 8, so that the buffer fills up and wraps around often.
        let extra = u.int_in_range(0..=7)?;
        let capacity = NonZeroUsize::MIN.saturating_add(extra);
        // The rest of the bytes become the list of operations.
        let ops: Vec<Op> = u.arbitrary()?;
        Ok(Scenario { capacity, ops })
    }
}
```

**Что происходит:**

1. `'a` — время жизни среза байтов фаззера (Урок 06.5): значение `Arbitrary` может заимствовать из этих байтов. Мы ничего не заимствуем, но трейту нужен этот параметр.
2. `u.int_in_range(0..=7)` берёт байт и превращает его в число от 0 до 7. `NonZeroUsize::MIN` равно 1, а `saturating_add` складывает без риска переполнения, поэтому ёмкость — от 1 до 8.
3. `u.arbitrary()` строит любой тип, реализующий `Arbitrary`, здесь `Vec<Op>`, из оставшихся байтов.
4. `?` передаёт дальше `arbitrary::Error`, например когда байты закончились. Фаззер тогда просто пропускает этот вход.

Почему от 1 до 8? При маленькой ёмкости буфер заполняется после нескольких push, поэтому интересный путь «полон, заменить самое старое» выполняется почти в каждом входе. Придавая входу форму, вы *направляете* фаззер на код, который вам важен.

```bash
cargo +nightly fuzz run ops -- -max_total_time=30
```

```text
#3	INITED cov: 75 ft: 78 corp: 2/16b exec/s: 0 rss: 36Mb
#8	NEW    cov: 76 ft: 79 corp: 3/21b lim: 8 exec/s: 0 rss: 37Mb L: 5/8 MS: 5 CrossOver-ShuffleBytes-CrossOver-ChangeBinInt-CrossOver-
...
thread '<unnamed>' (6026437) panicked at fuzz_targets/ops.rs:48:24:
assertion `left == right` failed
  left: Some(421075225)
 right: Some(2602113305)
...
Output of `std::fmt::Debug`:

	Scenario {
	    capacity: 2,
	    ops: [
	        Push(
	            2913840473,
	        ),
	        Push(
	            2602113305,
	        ),
	        Push(
	            421075225,
	        ),
	        Pop,
	    ],
	}
```

**Как это читать:** ёмкость 2, три push, один pop. Третий push должен заменить самое старое значение (2913840473), так что самым старым становится 2602113305: это и вернула модель (`right`). Настоящий буфер вернул *самое новое* значение (`left`), потому что `head` всё ещё указывает на только что перезаписанную ячейку. Неверный ответ, без падения, найденный примерно за десять тысяч прогонов. Вход уже крошечный, так что `tmin` не нужен.

Исправьте `push` в `src/lib.rs`:

```rust
        if self.len < capacity {
            self.len += 1;
        } else {
            // The buffer was full: `value` replaced the oldest value,
            // so the oldest one is now the next slot.
            self.head = (self.head + 1) % capacity;
        }
```

и сохраните находку как обычный тест (Урок 09.8, раздел 5) в модуле `tests` в конце `src/lib.rs`:

```rust
#[cfg(test)]
mod tests {
    use super::*;

    fn cap(n: usize) -> NonZeroUsize {
        NonZeroUsize::new(n).unwrap()
    }

    #[test]
    fn found_by_fuzzing_a_full_buffer_drops_the_oldest_value() {
        let mut buffer = RecentBuffer::new(cap(2));
        buffer.push(1);
        buffer.push(2);
        buffer.push(3); // replaces 1
        assert_eq!(buffer.pop_oldest(), Some(2));
        assert_eq!(buffer.pop_oldest(), Some(3));
        assert_eq!(buffer.pop_oldest(), None);
    }
}
```

```bash
cargo test -q
cargo +nightly fuzz run ops -- -max_total_time=30
```

```text
test result: ok. 3 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
...
#8334597	DONE   cov: 109 ft: 400 corp: 87/18Kb lim: 4096 exec/s: 268857 rss: 630Mb
Done 8334597 runs in 31 second(s)
```

(Три теста: два ваших из шага 1 «Попробуйте сами» и регрессионный.) Восемь миллионов сценариев без расхождений.

> 💡 **Совет:** модель — самое сильное утверждение, которое можно дать фаззеру. Хорошие модели повсюду: `HashMap` для вашей собственной хеш-таблицы, `BTreeMap` для B-дерева (Этап 17), старая медленная функция для её новой быстрой версии. Такое сравнение двух реализаций называется **дифференциальным фаззингом**.

> 📏 **Соглашение:** для одного поля крейт `arbitrary` предлагает вместо целого `impl` атрибут: `#[arbitrary(with = some_function)]`. Упражнение 3 его использует. Пишите полный `impl`, когда несколько полей зависят друг от друга.

## 4. Работа с корпусом

Корпус в `fuzz/corpus/ops/` — это память фаззера: каждый файл — вход, который дошёл до нового кода. Хороший корпус позволяет следующему прогону начать там, где остановился предыдущий. Помогают три инструмента.

**Начальные входы.** Перед первым прогоном можно положить в папку корпуса свои файлы, например настоящие сообщения для парсера протокола или файлы аварий старых багов. Фаззер сначала читает их и мутирует. Для парсера текстового формата несколько правильных примеров в качестве начальных входов избавляют фаззер от долгого угадывания.

**Словари.** Словарь — текстовый файл байтовых строк, по одной на строку в двойных кавычках, которые фаззер может вставлять во входы. Он помогает с ключевыми словами и «магическими» числами, которые случайная мутация находит редко. Создайте `fuzz/ops.dict`:

```text
# A capacity byte that gives the largest buffer, and two common values.
"\x07"
"\x00\x00\x00\x00"
"\xff\xff\xff\xff"
```

```bash
cargo +nightly fuzz run ops -- -dict=fuzz/ops.dict -max_total_time=3
```

```text
Dictionary: 3 entries
...
```

В конце каждого прогона libFuzzer ещё печатает **рекомендованный словарь**: байтовые строки, которые больше всего помогли в этом прогоне (`###### Recommended dictionary. ######` и строки вроде `"\377\377" # Uses: 59733`). Копируйте полезные строки в свой файл словаря. Для текстовых форматов, таких как HTTP (HyperText Transfer Protocol, протокол передачи гипертекста) или JSON (JavaScript Object Notation, нотация объектов JavaScript), словарь содержит слова формата: `"GET"`, `"Content-Length:"`, `"\r\n"`.

**Минимизация корпуса.** Когда несколько прогонов, на нескольких машинах или в разные ночи, добавляют файлы в один корпус, он заполняется входами, которые покрывают один и тот же код. `cargo fuzz cmin` воспроизводит все файлы и оставляет только наименьший набор с тем же покрытием. Вот корпус из 170 файлов, полученный двумя отдельными 20-секундными прогонами, скопированными вместе:

```bash
ls fuzz/corpus/ops | wc -l
cargo +nightly fuzz cmin ops
ls fuzz/corpus/ops | wc -l
```

```text
     170
...
MERGE-OUTER: 170 files, 0 in the initial corpus, 0 processed earlier
...
MERGE-OUTER: 92 new files with 411 new features added; 120 new coverage edges
      92
```

92 оставшихся файла доходят ровно до того же кода (тех же «features»), что и 170 файлов. Меньший корпус означает более быстрый старт, а это важно в CI (непрерывной интеграции), где каждый прогон начинается с воспроизведения всего корпуса.

> ⚡ **Производительность:** фаззинг-цель должна выполняться за микросекунды. Два приёма сохраняют её быстрой: придайте входу форму, чтобы он был маленьким (наша ёмкость от 1 до 8), и избегайте выделений памяти и логирования в цели. Следите за `exec/s` в строках прогресса: здесь около 270 000 прогонов в секунду.

## 5. Непрерывный фаззинг

Фаззер, запущенный один раз, нашёл баги того дня. Код продолжает меняться, значит, фаззинг тоже должен продолжаться. У **непрерывного фаззинга** три части:

1. **Короткий прогон на каждый pull request**, например 60 секунд на цель. Он ловит очевидную аварию, которую приносит изменение, до слияния.
2. **Долгий прогон каждую ночь**, например 30 минут, который уходит глубже.
3. **Корпус, который переживает прогоны**, чтобы каждый прогон продолжал с того места, где остановился предыдущий, а не начинал с нуля.

Вот workflow из Урока 09.12, выросший в непрерывный фаззинг. Сохраните его как `.github/workflows/fuzz.yml`:

```yaml
name: Fuzz

on:
  pull_request:
  schedule:
    - cron: "17 2 * * *" # каждую ночь в 02:17 UTC
  workflow_dispatch:

permissions:
  contents: read

jobs:
  fuzz:
    runs-on: ubuntu-latest
    timeout-minutes: 45
    env:
      # Короткие прогоны для pull request, долгие — ночью.
      FUZZ_SECONDS: ${{ github.event_name == 'pull_request' && '60' || '1800' }}
    steps:
      - uses: actions/checkout@v7
      - run: rustup toolchain install nightly --profile minimal
      - run: cargo install --locked cargo-fuzz

      - name: Restore the corpus from earlier runs
        uses: actions/cache/restore@v6
        with:
          path: fuzz/corpus
          key: fuzz-corpus-${{ github.run_id }}
          restore-keys: fuzz-corpus-

      - name: Fuzz
        run: cargo +nightly fuzz run ops -- -max_total_time="$FUZZ_SECONDS"

      - name: Minimize the corpus
        if: github.event_name != 'pull_request'
        run: cargo +nightly fuzz cmin ops

      - name: Save the corpus for the next run
        if: github.event_name != 'pull_request'
        uses: actions/cache/save@v6
        with:
          path: fuzz/corpus
          key: fuzz-corpus-${{ github.run_id }}

      - name: Keep the crash files
        if: failure()
        uses: actions/upload-artifact@v7
        with:
          name: fuzz-artifacts
          path: fuzz/artifacts/
```

**Что происходит:**

1. Workflow запускается на pull request, каждую ночь (`schedule`) и вручную (`workflow_dispatch`).
2. `FUZZ_SECONDS` равно 60 для pull request и 1800 (30 минут) в остальных случаях. `a && b || c` — способ языка выражений записать «если a, то b, иначе c».
3. `actions/cache/restore` ищет кэш с точным ключом (ID, идентификатор этого прогона, которого ещё не существует), а затем откатывается к самому новому кэшу, ключ которого начинается с `fuzz-corpus-`: корпусу последнего ночного прогона.
4. Только ночной прогон минимизирует и сохраняет корпус, под новым ключом. Кэши нельзя перезаписать, поэтому каждое сохранение получает новый ключ, а старые истекают сами. Pull request только читают его.
5. Если фаззер находит аварию, задание падает, а файлы аварий загружаются, чтобы вы воспроизвели её у себя.

Проверьте его до отправки, как в Уроке 09.12:

```bash
actionlint .github/workflows/fuzz.yml && echo "actionlint: no problems"
```

```text
actionlint: no problems
```

(Запустить серверы GitHub из этого курса мы не можем; `actionlint` проверяет синтаксис, выражения и входы actions.)

Для проектов с открытым кодом есть бесплатные сервисы, которые делают всё это в гораздо большем масштабе. **OSS-Fuzz** фаззит важные проекты с открытым кодом на машинах Google днём и ночью и приватно сообщает о багах мейнтейнерам; в нём много крейтов на Rust. **ClusterFuzzLite** — уменьшенная версия той же системы, которая работает внутри вашего собственного CI. Оба используют те же цели `cargo fuzz`, которые вы пишете в этом уроке.

> 🧪 **Тестирование:** фаззинг не заменяет `cargo test`. Обычные тесты проверяют известные случаи за миллисекунды при каждой сборке; фаззер ищет неизвестные случаи минутами и часами. Каждая находка становится обычным тестом, поэтому оба растут вместе.

## Частые ошибки

**1. Позволять фаззеру выбирать огромные размеры.** Выведенный `Arbitrary` для ёмкости, длины или количества даёт значения вплоть до `usize::MAX`. Тогда фаззер сообщает об OOM в вашей *цели* (раздел 2). Придавайте таким значениям форму через `int_in_range`.

**2. Модель, которая делит код с настоящей реализацией.** Если модель вызывает ту же вспомогательную функцию, что и настоящий код, оба могут ошибаться одинаково, и фаззер не увидит разницы. Держите модель независимой и как можно более простой.

**3. Менять формат входа и оставлять старый корпус.** После изменения `Scenario` или `Op` старые файлы корпуса декодируются в другие значения. Это не ошибка, но корпус теряет большую часть ценности. После больших изменений запускайте `cmin` или начинайте новый корпус.

**4. Коммитить корпус в Git.** Он меняется при каждом прогоне и сильно растёт. Храните его в кэше CI (раздел 5) или в отдельном хранилище; коммитьте только цели, словари, начальные входы и регрессионные тесты.

## Попробуйте сами

1. Создайте `code/p20-fuzz` и добавьте `src/lib.rs` с багом. Добавьте два теста, которые не заполняют буфер: один кладёт 1 и 2 в буфер на 3 и достаёт их, другой кладёт 1 и 2 в буфер на 2, делает один pop, кладёт 3 и делает два pop. Запустите `cargo test`: оба проходят, несмотря на баг.
2. Напишите цель с выведенным `Arbitrary` и посмотрите на отчёт об OOM.
3. Замените `Scenario` на собственный `Arbitrary` и дайте фаззеру найти неверный ответ. Исправьте `push`, добавьте регрессионный тест и снова запустите фаззер на 30 секунд.
4. Создайте `fuzz/ops.dict` и запустите с `-dict=fuzz/ops.dict`. Найдите строку `Dictionary: 3 entries`.
5. Выполните `cargo +nightly fuzz cmin ops` и сравните число файлов корпуса до и после.

## Упражнения

### Упражнение 1 — Прочитайте отчёт (легко)

Какое значение в отчёте из раздела 3 вернул настоящий буфер, какое — модель и какое из них правильное? Объясните одним предложением, почему настоящий буфер дал такой ответ.

<details>
<summary>Подсказка</summary>

`left` — это первый аргумент `assert_eq!`, здесь `real.pop_oldest()`.

</details>

<details>
<summary>Ответ</summary>

Настоящий буфер вернул 421075225 (третье, самое новое значение); модель вернула 2602113305 (второе значение). Права модель: после третьего push в буфер на 2 первого значения больше нет, и самым старым становится второе. Настоящий буфер записал новое значение в ячейку самого старого, но не сдвинул `head`, поэтому `head` указывал на самое новое значение.

</details>

### Упражнение 2 — Новая операция, с тестом (средне)

Добавьте в `RecentBuffer` метод `pub fn peek_oldest(&self) -> Option<u32>`: он возвращает самое старое значение, не удаляя его. Напишите для него модульный тест, в котором есть заполненный буфер. Затем добавьте `Peek` в `Op` и сверяйте его с моделью в фаззинг-цели. Фаззите 10 секунд.

<details>
<summary>Подсказка</summary>

Версия модели — `model.front().copied()`: `front()` даёт `Option<&u32>`, а `copied()` превращает его в `Option<u32>`.

</details>

<details>
<summary>Ответ</summary>

В `src/lib.rs`, внутри `impl RecentBuffer`:

```rust
    /// The oldest value, without removing it.
    pub fn peek_oldest(&self) -> Option<u32> {
        if self.len == 0 {
            None
        } else {
            Some(self.slots[self.head])
        }
    }
```

и в модуле `tests`:

```rust
    #[test]
    fn peek_shows_the_oldest_value_and_keeps_it() {
        let mut buffer = RecentBuffer::new(cap(2));
        assert_eq!(buffer.peek_oldest(), None);
        buffer.push(1);
        buffer.push(2);
        buffer.push(3);
        assert_eq!(buffer.peek_oldest(), Some(2));
        assert_eq!(buffer.len(), 2);
    }
```

В `fuzz/fuzz_targets/ops.rs` добавьте вариант `Peek,` в `Op` и такую ветку в `match`:

```rust
            Op::Peek => assert_eq!(real.peek_oldest(), model.front().copied()),
```

```text
#2659717	DONE   cov: 117 ft: 457 corp: 100/22Kb lim: 4096 exec/s: 241792 rss: 423Mb
Done 2659717 runs in 11 second(s)
```

Почему это работает: модульный тест закрепляет один известный случай, включая заполненный буфер, в котором прятался первый баг; фаззинг-цель проверяет `peek_oldest` после любой возможной последовательности операций.

</details>

### Упражнение 3 — Атрибут `with` (средне)

Замените собственный `impl Arbitrary for Scenario` на `#[derive(Arbitrary)]` плюс атрибут поля `#[arbitrary(with = small_capacity)]`, где `small_capacity` — функция, которую вы пишете. Поведение должно остаться прежним (ёмкость от 1 до 8).

<details>
<summary>Подсказка</summary>

У функции сигнатура `fn small_capacity(u: &mut Unstructured) -> arbitrary::Result<NonZeroUsize>`.

</details>

<details>
<summary>Ответ</summary>

```rust
/// A capacity from 1 to 8, so that the buffer fills up and wraps around often.
fn small_capacity(u: &mut Unstructured) -> arbitrary::Result<NonZeroUsize> {
    let extra = u.int_in_range(0..=7)?;
    Ok(NonZeroUsize::MIN.saturating_add(extra))
}

#[derive(Debug, Arbitrary)]
struct Scenario {
    #[arbitrary(with = small_capacity)]
    capacity: NonZeroUsize,
    ops: Vec<Op>,
}
```

Почему это работает: derive генерирует тот же код, что и наш `impl`, но для одного поля, которому нужна особая обработка, вызывает `small_capacity`. 10-секундный прогон заканчивается `DONE` без падений.

</details>

### Упражнение 4 — Баг или нет? Спросите фаззер (сложно)

Начните с исправленного `src/lib.rs`. Попробуйте два изменения в `pop_oldest`, по одному, каждое со свежим корпусом (`rm -rf fuzz/corpus`) и 20-секундным прогоном. Перед запуском предскажите, какое изменение — баг.

- **Изменение A:** после `self.len -= 1;` добавьте `if self.len == 0 { self.head = 0; }`.
- **Изменение B:** замените `self.head = (self.head + 1) % self.slots.len();` на `self.head += 1;`.

<details>
<summary>Подсказка</summary>

Когда буфер пуст, важно ли, куда указывает `head`? И что происходит с `head` после многих pop без `%`?

</details>

<details>
<summary>Ответ</summary>

Изменение A — **не** баг: когда буфер пуст, следующий push пишет в `head + 0`, поэтому подходит любая позиция. Фаззер согласен: `Done 7114681 runs in 21 second(s)`, без падений.

Изменение B — **баг**: `head` уходит за конец `slots`. Фаззер находит его за секунды:

```text
thread '<unnamed>' (6077894) panicked at src/lib.rs:39:31:
index out of bounds: the len is 2 but the index is 2
```

со сценарием ёмкости 2 и чередующимися операциями `Pop` и `Push`. (В отчёте указан полный путь к `src/lib.rs`.) Почему это важно: фаззер — быстрый способ проверить ваши *рассуждения* об изменении, а не только найти баги, которых вы не подозревали.

</details>

## Вопросы для самопроверки

1. Что модель добавляет в фаззинг-цель по сравнению с простой целью «вызови и посмотри, не запаникует ли»?

<details>
<summary>Ответ</summary>

Точное ожидание для каждого шага. Тогда фаззер находит неверные ответы (тихие отказы), а не только паники.

</details>

2. Почему первая версия цели сообщила о нехватке памяти и чья это была вина?

<details>
<summary>Ответ</summary>

Выведенный `Arbitrary` создал гигантскую ёмкость `NonZeroUsize`, и `vec![0; capacity]` попытался её выделить. Это слабость фаззинг-цели, а не баг в буфере.

</details>

3. Что делает `u.int_in_range(0..=7)`?

<details>
<summary>Ответ</summary>

Берёт байты из начала входа `Unstructured` и превращает их в число от 0 до 7 (включительно).

</details>

4. Чем начальный вход отличается от словаря?

<details>
<summary>Ответ</summary>

Начальный вход — целый файл, положенный в корпус как отправная точка. Словарь — список маленьких байтовых строк (ключевые слова, «магические» числа), которые фаззер может вставить в любой вход.

</details>

5. Что оставляет `cargo fuzz cmin` и зачем это нужно в CI?

<details>
<summary>Ответ</summary>

Наименьший набор файлов корпуса с тем же покрытием, что и весь корпус. Каждый прогон CI начинается с воспроизведения корпуса, поэтому меньший корпус означает более быстрые прогоны без потерь.

</details>

6. Почему в workflow каждое сохранение корпуса использует новый ключ кэша?

<details>
<summary>Ответ</summary>

Сохранённую запись кэша нельзя перезаписать. Новый ключ на каждый прогон (`github.run_id`) вместе с `restore-keys: fuzz-corpus-` всегда находит самый новый корпус и сохраняет ещё более новый.

</details>

## Итоги

- Фаззинг с состоянием на основе модели выполняет сгенерированную *последовательность* операций над настоящим кодом и над простой моделью и падает при первом расхождении. Он находит неверные ответы, а не только аварии.
- Выведенный `Arbitrary` может давать значения, которые атакуют фаззинг-цель (огромные размеры, OOM). Придавайте входу форму собственным `impl Arbitrary` или через `#[arbitrary(with = …)]`, используя помощники вроде `int_in_range`.
- Форма входа ещё и *направляет* фаззер на интересные пути (маленькая ёмкость быстро заполняется).
- Корпус — память фаззера: дайте ему начальные входы из настоящих примеров, помогите словарём и уменьшайте его через `cargo fuzz cmin`.
- Непрерывный фаззинг = короткие прогоны на pull request, долгие прогоны ночью, корпус в кэше и загрузка файлов аварий. OSS-Fuzz и ClusterFuzzLite делают то же в большом масштабе.
- Каждая находка становится обычным регрессионным тестом.

## Источники и что почитать

- [Rust Fuzz Book](https://rust-fuzz.github.io/book/) — официальное руководство по `cargo fuzz`, включая [structure-aware fuzzing](https://rust-fuzz.github.io/book/cargo-fuzz/structure-aware-fuzzing.html).
- [arbitrary — документация](https://docs.rs/arbitrary/1.5.0/arbitrary/) и [Unstructured](https://docs.rs/arbitrary/1.5.0/arbitrary/struct.Unstructured.html) — трейт, атрибуты derive и все вспомогательные методы.
- [Документация libFuzzer](https://llvm.org/docs/LibFuzzer.html) — значение строк прогресса, словари, `-merge` (его использует `cmin`) и все параметры.
- [actions/cache](https://github.com/actions/cache) — actions `restore` и `save` и как работают ключи кэша.
- [OSS-Fuzz](https://google.github.io/oss-fuzz/) и [ClusterFuzzLite](https://google.github.io/clusterfuzzlite/) — непрерывный фаззинг как сервис и внутри вашего собственного CI.

---

[🏠 Начало](../START-HERE.md) · [← 20.2 Ошибки и паники в продакшене](02-errors-and-panics-in-production.md) · [📋 Этап 20](README.md) · [20.4 Miri, санитайзеры и loom в CI →](04-miri-sanitizers-and-loom-in-ci.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
