<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 19.7 SIMD: одна инструкция, много данных](07-simd.md) · [📋 Этап 19](README.md) · [19.9 Производительность асинхронного кода и ввода-вывода →](09-async-and-io-performance.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/19-performance-and-profiling/08-build-settings-for-speed.md) <!-- nav -->

# Урок 19.8 — Настройки сборки для скорости: профили, LTO и PGO

> **Этап 19 — Производительность и профилирование** · Урок 8 из 11 · ⏱ около 2,5 часа (2 занятия)

## Что вы узнаете

- Что такое профиль Cargo, что входит в профили `dev` и `release` и как создавать свои профили.
- Что делают единицы кодогенерации (codegen units), LTO (оптимизация на этапе компоновки) и `panic = "abort"` и как они меняют скорость, размер бинарника и время сборки.
- Как компилировать под конкретный процессор, на котором вы работаете (`target-cpu`), и почему из-за этого программа может падать на других компьютерах.
- Как пользоваться PGO (оптимизацией по профилю) через `cargo pgo`.
- Как честно оценить, помогла ли настройка.

## Перед началом

- Вы прошли [Урок 19.7 — SIMD: одна инструкция, много данных](07-simd.md).
- Вы умеете пользоваться `hyperfine` и читать `mean ± σ` ([Урок 09.11 — Бенчмарки подробно](../09-testing-debugging-benchmarking/11-benchmarking-in-depth.md)).
- Вы знаете, что делают компилятор и компоновщик ([Урок 12.8 — От Rust до машинного кода](../12-computer-systems/08-from-rust-to-machine-code.md), [Урок 12.12 — Компоновка и загрузка программ](../12-computer-systems/12-linking-and-loading.md)), и что такое паника и раскрутка стека ([Урок 04.10 — Обработка ошибок](../04-structs-enums-collections-errors/10-error-handling.md)).

## Новые слова

| Слово | Значение |
|---|---|
| **профиль** (profile) | Именованный набор настроек компилятора в `Cargo.toml`. `cargo build` использует профиль `dev`, `cargo build --release` — профиль `release`. |
| **уровень оптимизации** (`opt-level`) | Насколько старается компилятор, чтобы код был быстрым: от `0` (совсем нет) до `3` (больше всего) или `"s"` / `"z"` (маленький код). |
| **LLVM** (изначально Low Level Virtual Machine, «низкоуровневая виртуальная машина»; сегодня просто название) | Генератор кода внутри `rustc`. Он оптимизирует программу и превращает её в машинный код. |
| **единица кодогенерации** (codegen unit) | Часть одного крейта, которую LLVM оптимизирует отдельно, параллельно с другими частями. Больше частей — быстрее сборка; меньше частей — LLVM видит больше кода сразу. |
| **встраивание** (inlining) | Копирование тела вызываемой функции в вызывающую, чтобы вызов исчез и код можно было оптимизировать вместе. |
| **LTO** (Link-Time Optimization, оптимизация на этапе компоновки) | Повторная оптимизация всей программы, когда все крейты собираются вместе, чтобы оптимизатор мог встраивать и удалять код *между* крейтами. |
| **толстая / тонкая LTO** (fat / thin LTO) | Толстая LTO сливает весь код в одну большую единицу и оптимизирует её (медленная сборка). Тонкая LTO оптимизирует параллельно, пользуясь сводкой по всей программе (сборка быстрее, результат часто похожий). |
| **стратегия паники** (panic strategy) | Что происходит при панике: `unwind` (подняться по стеку и выполнить код `Drop`) или `abort` (сразу завершить процесс). |
| **символ** (symbol) | Сохранённое в бинарнике имя функции или переменной, которым пользуются отладчики, профилировщики и отчёты о сбоях. *Удаление символов* (stripping) их убирает. |
| **целевой процессор** (target CPU) | Модель процессора, на которую рассчитывает компилятор. От неё зависит, какие необязательные инструкции (целевые возможности, Урок 19.7) может использовать код. |
| **PGO** (Profile-Guided Optimization, оптимизация по профилю) | Двойная сборка программы: сначала со счётчиками, затем вы запускаете её на типичной работе, затем компилятор использует подсчёты, чтобы оптимизировать горячие пути. |
| **инструментированный бинарник** (instrumented binary) | Специальная сборка, которая считает, сколько раз выполняется каждая ветка и функция, и при выходе записывает подсчёты в файл. |
| **BOLT** (Binary Optimization and Layout Tool, инструмент оптимизации и раскладки бинарников) | Инструмент LLVM, который по профилю переставляет машинный код готового бинарника. `cargo pgo` умеет использовать его в Linux; здесь мы его не используем. |

## 1. Профили: `dev` и `release`

Когда вы запускаете `cargo build`, Cargo передаёт `rustc` много параметров. **Профиль** — это именованный набор этих параметров. Двумя из них вы пользуетесь с Этапа 02:

| Настройка | `dev` (`cargo build`) | `release` (`cargo build --release`) | Что она определяет |
|---|---|---|---|
| `opt-level` | `0` | `3` | насколько оптимизируется код |
| `debug` | `true` | `false` | отладочная информация для отладчиков |
| `debug-assertions` | `true` | `false` | проверки `debug_assert!` |
| `overflow-checks` | `true` | `false` | паника при переполнении целых чисел |
| `incremental` | `true` | `false` | повторное использование работы прошлой сборки |
| `codegen-units` | `256` | `16` | на сколько частей LLVM делит работу для параллельной обработки |
| `lto` | `false` | `false` | оптимизация на этапе компоновки (`false` означает: только внутри одного крейта) |
| `panic` | `"unwind"` | `"unwind"` | стратегия паники |

Профиль `dev` сделан для быстрого цикла «правка — компиляция — запуск»; `release` — для быстрых программ. Любую настройку можно поменять в `Cargo.toml`, и можно создавать новые профили.

Наша тестовая программа — маленький калькулятор, который 60 раз разбирает и вычисляет 50 000 случайных выражений вроде `(712 + 33) * 9 / 4`. Создайте её:

```bash
cd ~/projects/rust
cargo new code/p19-build
cd code/p19-build
```

`src/lib.rs`, калькулятор (парсер рекурсивного спуска: по одной функции на каждое правило грамматики):

```rust
//! A tiny calculator: parses and evaluates expressions like `(3 + 4) * 2 - 7 / 3`.

/// Evaluates one expression with `+ - * /`, parentheses and whole numbers.
/// Returns `None` for bad input or division by zero.
pub fn eval(text: &str) -> Option<i64> {
    let mut p = Parser {
        bytes: text.as_bytes(),
        pos: 0,
    };
    let value = p.expr()?;
    p.skip_spaces();
    if p.pos == p.bytes.len() {
        Some(value)
    } else {
        None
    }
}

struct Parser<'a> {
    bytes: &'a [u8],
    pos: usize,
}

impl Parser<'_> {
    fn skip_spaces(&mut self) {
        while self.pos < self.bytes.len() && self.bytes[self.pos] == b' ' {
            self.pos += 1;
        }
    }

    fn peek(&mut self) -> Option<u8> {
        self.skip_spaces();
        self.bytes.get(self.pos).copied()
    }

    // expr = term (('+' | '-') term)*
    fn expr(&mut self) -> Option<i64> {
        let mut value = self.term()?;
        while let Some(op @ (b'+' | b'-')) = self.peek() {
            self.pos += 1;
            let rhs = self.term()?;
            value = if op == b'+' {
                value.wrapping_add(rhs)
            } else {
                value.wrapping_sub(rhs)
            };
        }
        Some(value)
    }

    // term = factor (('*' | '/') factor)*
    fn term(&mut self) -> Option<i64> {
        let mut value = self.factor()?;
        while let Some(op @ (b'*' | b'/')) = self.peek() {
            self.pos += 1;
            let rhs = self.factor()?;
            value = if op == b'*' {
                value.wrapping_mul(rhs)
            } else {
                value.checked_div(rhs)?
            };
        }
        Some(value)
    }

    // factor = number | '(' expr ')'
    fn factor(&mut self) -> Option<i64> {
        match self.peek()? {
            b'(' => {
                self.pos += 1;
                let value = self.expr()?;
                if self.peek()? != b')' {
                    return None;
                }
                self.pos += 1;
                Some(value)
            }
            b'0'..=b'9' => {
                let mut value: i64 = 0;
                while let Some(&d @ b'0'..=b'9') = self.bytes.get(self.pos) {
                    value = value.wrapping_mul(10).wrapping_add(i64::from(d - b'0'));
                    self.pos += 1;
                }
                Some(value)
            }
            _ => None,
        }
    }
}

#[cfg(test)]
mod tests {
    use super::eval;

    #[test]
    fn follows_the_usual_rules() {
        assert_eq!(eval("1 + 2 * 3"), Some(7));
        assert_eq!(eval("(1 + 2) * 3"), Some(9));
        assert_eq!(eval("7 / 0"), None);
        assert_eq!(eval("2 +"), None);
    }
}
```

`src/main.rs`, который создаёт входные данные и запускает калькулятор:

```rust
use p19_build::eval;

/// A small, fast pseudo-random number generator (xorshift), so every run sees the same input.
struct Rng(u64);

impl Rng {
    fn next(&mut self) -> u64 {
        self.0 ^= self.0 << 13;
        self.0 ^= self.0 >> 7;
        self.0 ^= self.0 << 17;
        self.0
    }
}

/// Writes a random expression with about `depth` levels of nesting into `out`.
fn gen_expr(rng: &mut Rng, depth: u32, out: &mut String) {
    if depth == 0 || rng.next().is_multiple_of(3) {
        out.push_str(&(rng.next() % 1000).to_string());
        return;
    }
    let paren = rng.next().is_multiple_of(4);
    if paren {
        out.push('(');
    }
    gen_expr(rng, depth - 1, out);
    let op = [" + ", " - ", " * ", " / "][(rng.next() % 4) as usize];
    out.push_str(op);
    gen_expr(rng, depth - 1, out);
    if paren {
        out.push(')');
    }
}

fn main() {
    // Optional first argument: how deeply the expressions are nested (default 6).
    let depth: u32 = std::env::args()
        .nth(1)
        .and_then(|s| s.parse().ok())
        .unwrap_or(6);
    let mut rng = Rng(42);
    let mut lines = Vec::new();
    for _ in 0..50_000 {
        let mut line = String::new();
        gen_expr(&mut rng, depth, &mut line);
        lines.push(line);
    }

    let mut ok = 0u64;
    let mut failed = 0u64;
    let mut checksum = 0i64;
    for _ in 0..60 {
        for line in &lines {
            match eval(line) {
                Some(v) => {
                    ok += 1;
                    checksum = checksum.wrapping_add(v);
                }
                None => failed += 1,
            }
        }
    }
    println!("ok {ok}, failed {failed}, checksum {checksum}");
}
```

**Что происходит:**

1. В пакете два крейта: библиотека `p19_build` (`src/lib.rs`) и программа (`src/main.rs`), которая её использует. Это важно для LTO в разделе 2.
2. `Rng` — крошечный генератор случайных чисел **xorshift**: три сдвига и операции XOR (exclusive or, исключающее ИЛИ, Урок 12.4) дают новое число, похожее на случайное. Он всегда начинает с 42, поэтому каждый запуск создаёт в точности те же выражения.
3. `gen_expr` строит случайное дерево выражения глубиной до `depth` уровней. Примерно одно выражение из 25 содержит деление на ноль, и `eval` возвращает для него `None`.
4. `checksum` складывает все результаты. Если бы настройка сборки когда-нибудь изменила поведение программы, контрольная сумма это показала бы.

Запустите тест, затем соберите и сравните два профиля:

```bash
cargo test
cargo build
cargo build --release
./target/release/p19-build
hyperfine -N --warmup 1 --runs 5 -n dev ./target/debug/p19-build -n release ./target/release/p19-build
```

```text
ok 2879220, failed 120780, checksum -6775572847266405736
Benchmark 1: dev (5 runs)
                mean     ±       σ          min     …     max
  Wall Time    2.344 s   ±   0.151 s      2.194 s   …   2.575 s
  ...
Benchmark 2: release (5 runs)
                mean     ±       σ          min     …     max
  Wall Time    0.482 s   ±   0.056 s      0.431 s   …   0.549 s            -79.4%  (4.9x faster)
  ...
```

`-N` запускает каждую команду напрямую, без командной оболочки посередине, и это немного уменьшает шум; `-n` даёт каждой команде короткое имя в отчёте. Сборка release примерно в 5 раз быстрее. **Это самая важная настройка сборки, с большим отрывом**: никогда не измеряйте и не поставляйте сборку `dev`. Всё остальное в этом уроке меняет скорость максимум на несколько процентов, как вы увидите.

## 2. Единицы кодогенерации и LTO

LLVM может оптимизировать только тот код, который **видит** одновременно. Сколько он видит, решают две настройки.

**Единицы кодогенерации.** Чтобы собирать быстрее, `rustc` режет каждый крейт на части (16 в `release`) и даёт LLVM оптимизировать их параллельно на нескольких ядрах процессора. Функцию из одной части нельзя встроить в функцию из другой части. `codegen-units = 1` даёт LLVM весь крейт сразу: сборка медленнее, код иногда быстрее.

**LTO.** Обычно каждый крейт оптимизируется отдельно, а компоновщик только склеивает готовый машинный код. С **LTO** компилятор сохраняет форму LLVM каждого крейта (включая `std` и все ваши зависимости) и ещё раз оптимизирует программу целиком, на этапе компоновки. Тогда функцию из зависимости можно встроить в ваш код, а код, который никто не использует, удаляется.

```text
without LTO:   crate A ──optimize──▶ machine code ─┐
               crate B ──optimize──▶ machine code ─┴─ link ──▶ program

with LTO:      crate A ──┐
               crate B ──┴─ link + optimize everything together ──▶ program
```

Создайте три новых профиля в конце `Cargo.toml`. `inherits = "release"` означает «взять все настройки release и поменять только эти»:

```toml
[profile.cu1]
inherits = "release"
codegen-units = 1

[profile.thin]
inherits = "release"
lto = "thin"

[profile.fat]
inherits = "release"
lto = "fat"
codegen-units = 1
```

Соберите их с `--profile`. Каждый профиль получает в `target/` свою папку с тем же именем:

```bash
cargo build --profile cu1
cargo build --profile thin
cargo build --profile fat
ls -l target/*/p19-build | awk '{print $5, $9}'
```

```text
455376 target/cu1/p19-build
544520 target/debug/p19-build
378752 target/fat/p19-build
455392 target/release/p19-build
430816 target/thin/p19-build
```

**Что происходит:**

1. `awk '{print $5, $9}'` печатает из `ls -l` только столбец 5 (размер в байтах) и столбец 9 (имя файла).
2. Толстая LTO делает бинарник примерно на 17% меньше: она удалила код из `std`, который наша программа никогда не вызывает. Тонкая LTO удаляет меньше.
3. Одна единица кодогенерации сама по себе здесь ничего не меняет: наша библиотека настолько мала, что и так помещается в одну часть.

Скорость всех вариантов мы измерим вместе в разделе 5. Почему выигрыш для этой программы мал, объясняет один факт: начиная с Rust 1.75 `rustc` сам помечает **маленькие** функции так, чтобы их можно было встраивать в другие крейты, даже без LTO. LTO помогает сильнее, когда программа тратит время в *больших* функциях из других крейтов (например, в парсере JSON (JavaScript Object Notation) или хеш-таблице из зависимости).

> ⚡ **Производительность:** толстая LTO и `codegen-units = 1` замедляют **сборку**, потому что LLVM оптимизирует всё одной частью на одном ядре. На этом крошечном пакете разница — одна-две секунды; у сервиса с сотнями зависимостей release-сборка с толстой LTO может идти в несколько раз дольше. Поэтому `release` не использует их по умолчанию.

## 3. Стратегия паники и удаление символов: профиль для поставки

Ещё две настройки меняют в основном **размер**, а не скорость:

- `panic = "abort"`: паника сразу завершает процесс. Код раскрутки стека (вызов `Drop` для каждого значения по пути наверх) не нужен и удаляется.
- `strip = true`: удаляет **символы** — имена функций, хранящиеся в бинарнике.

У обеих есть цена. С `panic = "abort"` паника в любом потоке убивает весь процесс, и код, который перехватывает панику (`std::panic::catch_unwind`; Tokio, например, превращает панику внутри задачи в ошибку для того, кто её ожидает), перестаёт работать; код `Drop`, например сброс файлового буфера, не выполняется. Без символов профилировщики и отчёты о сбоях показывают адреса вместо имён функций.

Хорошая привычка — отдельный профиль для сборок, которые вы поставляете, чтобы ежедневные сборки `--release` оставались быстрыми. Добавьте его в `Cargo.toml`:

```toml
[profile.dist]
inherits = "release"
lto = "fat"
codegen-units = 1
panic = "abort"
strip = true
```

Соберите его с `-v` (подробный вывод), чтобы увидеть, в какие параметры `rustc` превратился профиль:

```bash
cargo build --profile dist -v
ls -l target/release/p19-build target/dist/p19-build | awk '{print $5, $9}'
```

Строка `rustc` для программы содержит (сокращено):

```text
-C opt-level=3 -C panic=abort -C lto=fat -C codegen-units=1 ... -C strip=symbols
```

```text
302576 target/dist/p19-build
455392 target/release/p19-build
```

Бинарник `dist` на треть меньше `release`. Профиль — это не что иное, как список параметров `-C` для `rustc`.

> 📏 **Соглашение:** профили читаются только из `Cargo.toml` в корне рабочего пространства (пакета или рабочего пространства, которое вы собираете). Профили в `Cargo.toml` зависимостей игнорируются. Держите `release` близким к умолчанию, а медленные в сборке настройки выносите в отдельный профиль вроде `dist`.

## 4. Сборка под ваш конкретный процессор

Компилятор должен создавать код, который работает на *любом* процессоре целевой платформы. Для `aarch64-apple-darwin` он рассчитывает на процессор первого Apple M1, `apple-m1`, поэтому новые инструкции вашего M4 не используются. `-C target-cpu=native` говорит `rustc`: «используй всё, что есть у процессора *этого* компьютера»:

```bash
rustc --print target-cpus | grep native
```

```text
    native          - Select the CPU of the current host (currently apple-m4).
```

Параметры компилятора, которых нет в профиле, передают через переменную окружения `RUSTFLAGS`. Другое значение `RUSTFLAGS` меняет каждый крейт, поэтому используйте отдельную папку сборки, чтобы сохранить обычные сборки:

```bash
RUSTFLAGS="-C target-cpu=native" cargo build --profile dist --target-dir target/native
```

Чтобы проект всегда собирался так, запишите это в `.cargo/config.toml` в папке проекта:

```toml
[build]
rustflags = ["-C", "target-cpu=native"]
```

> ⚠️ **Частая ошибка:** поставлять бинарник, собранный с `target-cpu=native`. Он может использовать инструкции, которых нет у других процессоров. На сервере со старым процессором он сразу падает с «illegal instruction» (`SIGILL`, Урок 13.7). Используйте `native` только для программ, которые работают на той же машине, где собраны, или указывайте конкретный более старый уровень процессора (для серверов x86-64, например, `-C target-cpu=x86-64-v3`, который требует AVX2, как в Уроке 19.7).

## 5. PGO: пусть настоящий запуск направляет оптимизатор

Оптимизатору приходится угадывать многое: какая ветка `if` обычна, какая функция горячая и стоит встраивания, какой код холодный и его можно убрать подальше. **PGO** заменяет догадки фактами из настоящего запуска:

```text
1. build with counters      ──▶  instrumented binary
2. run it on typical work   ──▶  counts written to *.profraw files
3. merge the counts         ──▶  one *.profdata file
4. build again using it     ──▶  optimized binary
```

Инструмент `cargo-pgo` делает шаги 1, 3 и 4 за вас. Ему нужен `llvm-profdata`, который входит в компонент `llvm-tools-preview`. Установите оба один раз:

```bash
rustup component add llvm-tools-preview
cargo install --locked cargo-pgo
cargo pgo info
```

```text
[rustc version]: 1.100.0 is recent enough
[llvm-profdata]: found at /Users/you/.rustup/toolchains/stable-aarch64-apple-darwin/lib/rustlib/aarch64-apple-darwin/bin/llvm-profdata
[llvm-bolt]: could not be found (Build LLVM with BOLT and add its `bin` directory to PATH.)
[merge-fdata]: could not be found (Build LLVM with BOLT and add its `bin` directory to PATH.)
Some requirements were not satisfied
```

Недостающие инструменты BOLT нужны только для необязательного шага BOLT; PGO работает без них. Теперь четыре шага:

```bash
cargo pgo build
./target/aarch64-apple-darwin/release/p19-build
cargo pgo optimize
```

Вывод (сокращён; метки времени и длинные пути убраны):

```text
INFO  cargo_pgo::pgo::instrument] PGO-instrumented binary p19-build built successfully.
INFO  cargo_pgo::pgo::instrument] Now run .../target/aarch64-apple-darwin/release/p19-build on your workload.
ok 2879220, failed 120780, checksum -6775572847266405736
INFO  cargo_pgo::pgo::optimize] Found 1 PGO profile file with total size 2.59 KiB at .../target/pgo-profiles.
INFO  cargo_pgo::pgo::optimize] Merged PGO profile(s) to .../target/pgo-profiles/merged-59625e....profdata.
INFO  cargo_pgo::pgo::optimize] PGO-optimized binary p19-build built successfully.
PGO optimized build successfully finished.
WARN  cargo_pgo::pgo::optimize] PGO profile data was not found for 1 function.
```

**Что происходит:**

1. `cargo pgo build` собирает инструментированный бинарник. Он передаёт Cargo `--target aarch64-apple-darwin`, поэтому бинарник оказывается в `target/aarch64-apple-darwin/release/`.
2. Запустить его — **ваша** задача: здесь вы даёте ему типичную работу. При выходе он записывает файл `.profraw` в `target/pgo-profiles/`.
3. `cargo pgo optimize` объединяет файлы через `llvm-profdata` и собирает программу снова с `-C profile-use=...`. Итоговый бинарник заменяет инструментированный в той же папке.
4. Предупреждение безвредно: одна функция ни разу не выполнилась во время обучающего запуска, поэтому подсчётов для неё нет.

Теперь сравните всё (это займёт около минуты):

```bash
hyperfine -N --warmup 3 --runs 20 -n release ./target/release/p19-build -n cu1 ./target/cu1/p19-build -n thin ./target/thin/p19-build -n fat ./target/fat/p19-build -n dist ./target/dist/p19-build -n native ./target/native/dist/p19-build -n pgo ./target/aarch64-apple-darwin/release/p19-build --export-markdown results.md
```

Таблица из `results.md` (M4; у вас будет иначе):

```text
| Command | Mean Wall Time [ms] | Change | Factor |
|:---|---:|---:|:---|
| `release` | 465.3 ± 33.1 |  |  |
| `cu1` | 460.2 ± 18.1 | -1.1% |  |
| `thin` | 464.2 ± 48.9 | -0.3% |  |
| `fat` | 447.0 ± 18.9 | -3.9% |  |
| `dist` | 450.3 ± 28.6 | -3.2% |  |
| `native` | 440.8 ± 9.7 | -5.3% |  |
| `pgo` | 436.0 ± 19.3 | -6.3% |  |
```

**Что происходит:**

1. Все различия — от 0 до 6 процентов. Лучше всех выглядят PGO и `native`.
2. Теперь прочитайте столбец `±`. Шум `σ` у `release` — 33 мс, это 7 процентов. Разница меньше шума — это **ещё не** результат. В других запусках на том же Mac PGO был то на 4 процента быстрее, то на 1 процент медленнее.
3. Чтобы решить, повторите измерение (больше запусков, спокойная машина, Урок 09.11) и оставляйте настройку, только если выигрыш явно больше шума и стоит своей цены: более медленные сборки для LTO, шаг обучения в каждом релизе для PGO, менее переносимые бинарники для `native`.

Сравните с разделом 1: переход с `dev` на `release` дал 5 раз. В [Уроке 19.10](10-project-make-a-slow-program-fast.md) вы ускорите программу в 10 раз, меняя код. Настройки сборки — это последние несколько процентов, а не первый большой шаг.

> 🧪 **Тестирование:** настройки вроде `panic`, `overflow-checks` и `debug-assertions` меняют то, как программа **ведёт себя**, а не только её скорость. Запускайте тесты с тем профилем, который поставляете: `cargo test --profile dist`.

## 6. Разумная схема

| Цель | Настройка |
|---|---|
| Ежедневная работа | `dev`; если программу слишком медленно пробовать, добавьте `opt-level = 1` в `[profile.dev]` (наш калькулятор: 2,3 с → 0,44 с, почти как `release`) |
| Бенчмарки и профилирование | `release` для бенчмарков; для профилировщиков — профиль `profiling` из Урока 19.3 (`release` плюс `debug = true`). `debug = "line-tables-only"` — более компактный вариант, который всё равно даёт имена функций и строки |
| Поставка | отдельный профиль вроде `dist`: `lto = "fat"` или `"thin"`, `codegen-units = 1` и, возможно, `panic = "abort"` |
| Самый маленький бинарник | `opt-level = "z"` вместе с `strip = true` (обычно медленнее; см. Упражнение 1) |
| Последние несколько процентов | PGO с реалистичной обучающей нагрузкой, аккуратно измеренный |

## Частые ошибки

> ⚠️ **Частая ошибка:** забыть `--profile`. `cargo build --release` не использует настройки `dist`. Проверьте, что строка `Finished` называет профиль `dist` и что бинарник лежит в `target/dist/`.

> ⚠️ **Частая ошибка:** обучать PGO на необычной работе, например только на крошечных тестовых входах. Тогда оптимизатор ускоряет не те пути. Используйте вход, похожий на реальное использование.

> ⚠️ **Частая ошибка:** называть разницу в 2 процента «ускорением», когда `σ` — 5 процентов. Всегда сравнивайте разницу с шумом.

> ⚠️ **Частая ошибка:** включить `panic = "abort"` в сервисе, который рассчитывает на перехват паник (например, паникующая задача Tokio не должна останавливать весь сервер).

## Попробуйте сами

1. Соберите `code/p19-build` и повторите сравнение `dev` и `release`.
2. Добавьте профили `cu1`, `thin`, `fat` и `dist`, соберите их и запишите размеры бинарников.
3. Выполните шаги PGO и дважды запустите большую команду `hyperfine`. Совпадают ли числа двух запусков?
4. Добавьте `opt-level = 1` в раздел `[profile.dev]`, выполните `cargo build` и измерьте `./target/debug/p19-build`.

## Упражнения

### Упражнение 1 — Самый маленький бинарник (легко)

Создайте профиль `small`, который наследует `dist` и задаёт `opt-level = "z"`. Сравните его размер и скорость с `dist`. Стоит ли это того для этой программы?

<details>
<summary>Подсказка</summary>

`[profile.small]` с `inherits = "dist"`. Затем `cargo build --profile small`, `ls -l` и `hyperfine` с обоими бинарниками.

</details>

<details>
<summary>Ответ</summary>

```toml
[profile.small]
inherits = "dist"
opt-level = "z"
```

```text
302464 target/dist/p19-build
286080 target/small/p19-build
```

```text
Benchmark 1: dist (10 runs)
  Wall Time    443.2 ms  ±    31.2 ms     427.4 ms  …   528.5 ms
Benchmark 2: small (10 runs)
  Wall Time    632.2 ms  ±    41.5 ms     593.6 ms  …   714.1 ms           +42.7%  (1.4x slower)
```

Почему это работает: `"z"` говорит LLVM предпочитать маленький код быстрому (например, без развёртки циклов, с меньшим встраиванием). Бинарник меньше всего на 5%, потому что большая его часть — код `std`, который уже удалили или ужали LTO и удаление символов, а программа в 1,4 раза медленнее. Для программы на обычном компьютере это того не стоит; для крошечного устройства с маленькой памятью — может стоить.

</details>

### Упражнение 2 — Тест, который зависит от профиля (средне)

Создайте профиль `safe`, который наследует `release` и задаёт `overflow-checks = true`. Напишите интеграционный тест `tests/overflow.rs`, который проверяет, что сложение `250u8 + 10` паникует с сообщением «attempt to add with overflow». Запустите его с `cargo test --profile safe` и с `cargo test --release`. Что происходит и почему?

<details>
<summary>Подсказка</summary>

Используйте `#[should_panic(expected = "...")]`. Спрячьте числа через `std::hint::black_box`, иначе компилятор увидит переполнение ещё при компиляции и откажется собирать (линт `arithmetic_overflow`).

</details>

<details>
<summary>Ответ</summary>

```toml
[profile.safe]
inherits = "release"
overflow-checks = true
```

```rust
use std::hint::black_box;

#[test]
#[should_panic(expected = "attempt to add with overflow")]
fn adding_past_u8_max_panics() {
    let big: u8 = black_box(250);
    let _ = big + black_box(10);
}
```

`cargo test --profile safe --test overflow`:

```text
test adding_past_u8_max_panics - should panic ... ok
```

`cargo test --release --test overflow`:

```text
test adding_past_u8_max_panics - should panic ... FAILED
note: test did not panic as expected at tests/overflow.rs:5:4
```

Почему это работает: в `release` настройка `overflow-checks` равна `false`, поэтому `250 + 10` молча заворачивается в 4, и паники нет. Один и тот же код ведёт себя по-разному в разных профилях. Некоторые команды поставляют программы с `overflow-checks = true`, потому что паника безопаснее неверного числа (Урок 12.2); цена обычно небольшая, но её стоит измерить.

</details>

### Упражнение 3 — PGO с неправильным обучением (средне)

Программа принимает глубину вложенности первым аргументом. Обучите PGO на очень простых выражениях (`./target/aarch64-apple-darwin/release/p19-build 1` вместо обычного запуска), скопируйте результат в `pgo-bad`, затем обучите снова как обычно и скопируйте результат в `pgo-good`. Сравните `release`, `pgo-good` и `pgo-bad` на обычной нагрузке. Что вы видите и чего ожидали бы на большой программе?

<details>
<summary>Подсказка</summary>

`cargo pgo build`, запуск с `1`, `cargo pgo optimize`, `cp target/aarch64-apple-darwin/release/p19-build pgo-bad`. Затем то же без `1`, в `pgo-good`. `cargo pgo build` сам очищает старые профили.

</details>

<details>
<summary>Ответ</summary>

```bash
cargo pgo build
./target/aarch64-apple-darwin/release/p19-build 1
cargo pgo optimize
cp target/aarch64-apple-darwin/release/p19-build pgo-bad
cargo pgo build
./target/aarch64-apple-darwin/release/p19-build
cargo pgo optimize
cp target/aarch64-apple-darwin/release/p19-build pgo-good
hyperfine -N --warmup 3 --runs 20 -n release ./target/release/p19-build -n pgo-good ./pgo-good -n pgo-bad ./pgo-bad
```

Один запуск на M4:

```text
Benchmark 1: release (20 runs)
  Wall Time    426.7 ms  ±     7.6 ms     414.6 ms  …   444.0 ms
Benchmark 2: pgo-good (20 runs)
  Wall Time    430.8 ms  ±     8.6 ms     418.1 ms  …   457.9 ms            +1.0%
Benchmark 3: pgo-bad (20 runs)
  Wall Time    431.7 ms  ±    14.3 ms     421.1 ms  …   470.4 ms            +1.2%
```

Почему это работает: для этого маленького парсера все три равны в пределах шума; PGO здесь почти нечего улучшать. Это тоже честный результат. В большой программе с множеством ветвлений (компилятор, база данных, интерпретатор) профиль, снятый на простых входах, помечает горячими не те пути, и сборка с PGO может оказаться даже медленнее обычного `release`. Обучайте на входе, похожем на реальное использование, и на нём же измеряйте.

</details>

## Вопросы для самопроверки

1. Какая настройка сборки сильнее всего изменила скорость калькулятора?

<details>
<summary>Ответ</summary>

`dev` → `release` (по сути `opt-level` 0 → 3): примерно в 5 раз быстрее. Все остальные настройки меняли скорость максимум на несколько процентов.

</details>

2. Чем тонкая LTO отличается от толстой?

<details>
<summary>Ответ</summary>

Обе оптимизируют между крейтами на этапе компоновки. Толстая LTO сливает весь код в одну единицу (лучший обзор, самая медленная сборка). Тонкая LTO оставляет единицы раздельными, использует общую сводку по всей программе и работает параллельно (сборка быстрее, результат часто похожий).

</details>

3. Назовите два риска `panic = "abort"`.

<details>
<summary>Ответ</summary>

Любая паника сразу завершает весь процесс, поэтому перехват паник (`catch_unwind`, паникующая задача Tokio) больше не работает; и код `Drop` не выполняется, поэтому буферы не сбрасываются и уборка не происходит.

</details>

4. Почему бинарник, собранный с `-C target-cpu=native`, может упасть на другом компьютере?

<details>
<summary>Ответ</summary>

Он может использовать инструкции, которые есть только у процессора машины сборки. На процессоре без них программа останавливается с «illegal instruction».

</details>

5. Из каких шагов состоит PGO?

<details>
<summary>Ответ</summary>

Собрать инструментированный бинарник, запустить его на типичной работе, чтобы он записал подсчёты (`.profraw`), объединить подсчёты (`.profdata`) и собрать снова с профилем. `cargo pgo build`, ваш запуск, `cargo pgo optimize`.

</details>

6. Настройка ускоряет программу на 2%, а `σ` равна 4%. Какой вывод?

<details>
<summary>Ответ</summary>

Пока никакого: разница меньше шума. Измерьте снова с большим числом запусков на спокойной машине; оставляйте настройку, только если выигрыш явно больше шума и стоит своей цены.

</details>

## Итоги

- Профиль — это именованный список параметров `rustc` в `Cargo.toml`. Свой профиль создаётся через `[profile.name]` и `inherits = "release"`, собирается с `--profile name`; результат попадает в `target/name/`.
- `release` вместо `dev` — большой шаг (здесь 5 раз). Остальные настройки — последние несколько процентов.
- `codegen-units = 1` и LTO позволяют LLVM видеть больше кода сразу. Часто они уменьшают бинарник, иногда ускоряют его и всегда замедляют сборку.
- `panic = "abort"` и `strip = true` в основном уменьшают размер, но меняют поведение при панике и убирают имена из отчётов о сбоях и профилей.
- `-C target-cpu=native` использует все инструкции вашего процессора; такой бинарник может упасть на других компьютерах.
- PGO (`cargo pgo build` → запуск → `cargo pgo optimize`) использует подсчёты из настоящего запуска. Обучайте на реалистичной работе.
- Сравнивайте каждый выигрыш с шумом `σ` и тестируйте с тем профилем, который поставляете.

## Источники и что почитать

- [The Cargo Book — Profiles](https://doc.rust-lang.org/cargo/reference/profiles.html) — все настройки профилей, их значения по умолчанию и собственные профили.
- [The rustc book — Codegen options](https://doc.rust-lang.org/rustc/codegen-options/index.html) — параметры `-C`, такие как `opt-level`, `lto`, `codegen-units`, `target-cpu` и `strip`.
- [The rustc book — Profile-guided Optimization](https://doc.rust-lang.org/rustc/profile-guided-optimization.html) — как PGO работает в `rustc` без дополнительных инструментов.
- [cargo-pgo](https://github.com/Kobzol/cargo-pgo) — инструмент из этого урока, с поддержкой BOLT.
- [The Cargo Book — Configuration](https://doc.rust-lang.org/cargo/reference/config.html) — `.cargo/config.toml`, в том числе `build.rustflags`.
- [The Rust Performance Book — Build Configuration](https://nnethercote.github.io/perf-book/build-configuration.html) — практические советы, какие настройки стоит попробовать.

---

[🏠 Начало](../START-HERE.md) · [← 19.7 SIMD: одна инструкция, много данных](07-simd.md) · [📋 Этап 19](README.md) · [19.9 Производительность асинхронного кода и ввода-вывода →](09-async-and-io-performance.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
