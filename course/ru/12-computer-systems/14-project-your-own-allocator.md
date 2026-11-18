<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 12.13 Как работают аллокаторы памяти](13-how-memory-allocators-work.md) · [📋 Этап 12](README.md) · [12.15 Контрольная точка →](15-checkpoint.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/12-computer-systems/14-project-your-own-allocator.md) <!-- nav -->

# Урок 12.14 — Проект: собственный аллокатор памяти

> **Этап 12 — Компьютерные системы** · Урок 14 из 15 · ⏱ около 8 часов (4 занятия)

## Что вы узнаете

- Как превратить идеи аллокаторов из Урока 12.13 в две настоящие реализации `GlobalAlloc`: bump-аллокатор и аллокатор со списком свободных блоков.
- Как сохранять корректность кода с сырыми указателями: комментарии `// SAFETY:`, документация `# Safety`, маленькие блоки `unsafe` и Miri на каждом тесте.
- Как тестировать аллокатор: модульные тесты на каждое правило, случайный нагрузочный тест, который проверяет каждый байт, и тест с несколькими потоками.
- Как запускать настоящие программы на собственном аллокаторе и читать его статистику.

## Перед началом

- Вы прошли [Урок 12.13 — Как работают аллокаторы памяти](13-how-memory-allocators-work.md): `Layout`, `GlobalAlloc`, `align_up`, bump-выделение, списки свободных блоков, разделение и слияние.
- Вы умеете писать код `unsafe` и запускать Miri ([Урок 06.6](../06-rust-advanced/06-unsafe-rust-and-miri.md)), знаете константные обобщения вроде `[u8; N]` ([Урок 05.1](../05-rust-intermediate/01-generics.md)) и `thread::scope` ([Урок 05.10](../05-rust-intermediate/10-threads-channels-and-shared-state.md)).
- Создайте проект как библиотеку с собственной историей Git, как в прошлых проектах:

```bash
cd ~/projects/rust
cargo new --lib code/p12-allocator
cd code/p12-allocator
mkdir examples tests
```

## Новые слова

| Слово | Значение |
|---|---|
| **спин-блокировка** (spin lock) | Блокировка, которая ждёт, снова и снова проверяя атомарный флаг («крутясь»), а не просит операционную систему усыпить поток. Она никогда не выделяет память, поэтому её может использовать аллокатор. Урок 14.7 подробно строит такую. |
| **guard-объект блокировки** (lock guard) | Значение, которое доказывает, что вы держите блокировку, и снимает её, когда уничтожается (RAII, Resource Acquisition Is Initialization, «получение ресурса есть инициализация», Урок 08.7). |
| **нагрузочный тест** (stress test) | Тест, который выполняет тысячи случайных операций и после каждой проверяет, что ничего не сломалось. |
| **инвариант** (invariant) | Правило, которое всегда должно выполняться для структуры данных, например «список свободных блоков отсортирован по адресам, и никакие два свободных блока не соприкасаются». |
| **аварийное завершение** (abort) | Немедленная остановка процесса без раскрутки стека. Rust аварийно завершается, когда глобальный аллокатор возвращает null (сигнал `SIGABRT`, код возврата 134). |
| **`NonNull<T>`** | Сырой указатель, который никогда не равен null (`std::ptr::NonNull`). `Option<NonNull<T>>` имеет размер указателя (ниша из Урока 12.11). |

## Проект

### Что вы создадите

Библиотечный крейт `p12_allocator` с двумя аллокаторами, которые могут заменить глобальный аллокатор Rust:

1. **`BumpAlloc<N>`**: bump-аллокатор над ареной в `N` байт, которая живёт внутри самого значения аллокатора.
2. **`FreeListAlloc<N>`**: аллокатор со списком свободных блоков по принципу первого подходящего, над собственной ареной в `N` байт, с разделением и слиянием. Его ядро — `FreeListHeap`, обычная структура, которую можно тестировать, не делая её глобальной.

Две программы-примера запускают настоящий код на Rust (`Vec`, `format!`, `HashMap`) поверх них. В конце каждый тест проходит и под Miri.

```text
   p12-allocator/
   ├── Cargo.toml
   ├── README.md
   ├── src/
   │   ├── lib.rs          ← module list and re-exports
   │   ├── spin.rs         ← SpinLock (given code, below)
   │   ├── align.rs        ← align_up                      (M1)
   │   ├── bump.rs         ← BumpAlloc<N>                  (M2)
   │   └── free_list.rs    ← FreeListHeap, FreeListAlloc<N> (M3, M4)
   ├── examples/
   │   ├── bump_global.rs        (M2)
   │   └── free_list_global.rs   (M4)
   └── tests/
       ├── stress.rs       (M5)
       └── threads.rs      (M4)
```

### Контракт

**Без зависимостей.** Только стандартная библиотека. Редакция 2024.

**Публичный API** (`src/lib.rs` реэкспортирует эти имена):

```text
pub fn align_up(addr: usize, align: usize) -> Option<usize>

pub struct BumpAlloc<const N: usize>
    pub const fn new() -> BumpAlloc<N>
    pub fn used(&self) -> usize                  // bytes used so far, padding included
    unsafe impl GlobalAlloc

pub struct FreeListHeap
    pub const fn empty() -> FreeListHeap
    pub unsafe fn add_region(&mut self, start: *mut u8, size: usize)
    pub fn alloc(&mut self, layout: Layout) -> *mut u8        // null if nothing fits
    pub unsafe fn dealloc(&mut self, ptr: *mut u8, layout: Layout)
    pub fn free_bytes(&self) -> usize
    pub fn free_blocks(&self) -> usize

pub struct FreeListAlloc<const N: usize>
    pub const fn new() -> FreeListAlloc<N>
    pub fn stats(&self) -> (usize, usize)        // (free bytes, free blocks)
    unsafe impl GlobalAlloc

pub struct SpinLock<T>, pub struct SpinGuard<'a, T>   // given code
```

**Поведение:**

1. `align_up(addr, align)` округляет вверх до кратного `align` (степени двойки) и возвращает `None` вместо переполнения.
2. Каждый блок, который возвращает `alloc`, содержит не меньше `layout.size()` байт, начинается с кратного `layout.align()`, целиком лежит внутри арены и не пересекается ни с одним другим живым блоком.
3. Когда ни один блок не подходит, `alloc` возвращает нулевой указатель. Он никогда не паникует и никогда не выделяет память.
4. `BumpAlloc` повторно использует память, только когда освобождены **все** его блоки (тогда `used()` снова равен 0).
5. `FreeListHeap` держит список свободных блоков **отсортированным по адресам**, использует **первый подходящий**, **разделяет** блок, когда в остатке ещё помещается заголовок свободного блока, и **сливает** освобождённый блок со свободными соседями с обеих сторон. Когда освобождён каждый блок, куча снова — **один** свободный блок со всеми своими байтами.
6. Блок списка занимает не меньше 16 байт (место для заголовка, описанного в M3), а его размер кратен 8. `free_bytes()` считает эти настоящие размеры.
7. Оба аллокатора безопасно использовать из многих потоков одновременно: всё общее состояние находится за `SpinLock`.

**Правила кода:** у каждого блока `unsafe` есть комментарий `// SAFETY:`, объясняющий, почему он корректен; у каждой `pub unsafe fn` есть раздел `# Safety` в документации; `cargo clippy --all-targets -- -D warnings` и `cargo fmt --check` чисты; есть короткий `README.md` (что это за крейт, как запускать тесты и примеры, устройство в пяти строках); и каждая веха заканчивается одним коммитом Git.

**Готовый код.** Аллокатор не может безопасно использовать `std::sync::Mutex`: блокировка никогда не должна выделять память, а это пришлось бы доказывать для каждой платформы. Используйте вместо неё эту маленькую спин-блокировку. Поместите её в `src/spin.rs` как есть; понимать каждую деталь пока не нужно (Урок 14.7 это объясняет):

```rust
//! A minimal spin lock. Given code: Lesson 14.7 explains how it works.

use std::cell::UnsafeCell;
use std::ops::{Deref, DerefMut};
use std::sync::atomic::{AtomicBool, Ordering};

/// A lock that waits by spinning. It never allocates, so an allocator may use it.
pub struct SpinLock<T> {
    locked: AtomicBool,
    value: UnsafeCell<T>,
}

// SAFETY: only one thread at a time can reach `value`, through a `SpinGuard`.
unsafe impl<T: Send> Sync for SpinLock<T> {}

impl<T> SpinLock<T> {
    pub const fn new(value: T) -> SpinLock<T> {
        SpinLock {
            locked: AtomicBool::new(false),
            value: UnsafeCell::new(value),
        }
    }

    pub fn lock(&self) -> SpinGuard<'_, T> {
        while self.locked.swap(true, Ordering::Acquire) {
            std::hint::spin_loop();
        }
        SpinGuard { lock: self }
    }
}

/// Gives access to the value; unlocks when dropped.
pub struct SpinGuard<'a, T> {
    lock: &'a SpinLock<T>,
}

impl<T> Deref for SpinGuard<'_, T> {
    type Target = T;
    fn deref(&self) -> &T {
        // SAFETY: the guard proves that this thread holds the lock.
        unsafe { &*self.lock.value.get() }
    }
}

impl<T> DerefMut for SpinGuard<'_, T> {
    fn deref_mut(&mut self) -> &mut T {
        // SAFETY: the guard proves that this thread holds the lock.
        unsafe { &mut *self.lock.value.get() }
    }
}

impl<T> Drop for SpinGuard<'_, T> {
    fn drop(&mut self) {
        self.lock.locked.store(false, Ordering::Release);
    }
}
```

Коротко: `swap(true, …)` устанавливает флаг и возвращает старое значение; пока старое значение было `true`, блокировку держит другой поток, и мы пробуем снова. Guard-объект даёт `&T` / `&mut T` через `Deref` и `DerefMut` (Урок 05.9) и сбрасывает флаг в `Drop`.

**Обязательные тесты** (модульные тесты в том модуле, который они проверяют; имена — часть контракта, поэтому ожидаемые выводы ниже совпадают):

| Модуль | Тест | Проверяет |
|---|---|---|
| `align` | `align_up_keeps_aligned_addresses` | `align_up(0, 8)` и `align_up(4096, 4096)` не сдвигаются |
| `align` | `align_up_rounds_up` | `1 → 8`, `13 → 16` (выравнивание 4), `4097 → 8192` (выравнивание 4096) |
| `align` | `align_up_reports_overflow` | `align_up(usize::MAX, 8)` равен `None` |
| `bump` | `bump_blocks_are_aligned_and_separate` | `u8`, затем `u64`: оба не null, второй выровнен по 8 и лежит после первого |
| `bump` | `bump_returns_null_when_full` | в арене на 64 байта блок в 64 байта выделяется, а ещё 1 байт даёт null |
| `bump` | `bump_resets_when_everything_is_freed` | выделить, освободить, затем `used() == 0`, и выделение снова работает |
| `free_list` | `free_list_starts_with_one_block` | свежая область в 1024 байта — 1 блок в 1024 байта |
| `free_list` | `free_list_splits_and_reuses` | `alloc(100, align 8)` оставляет свободными 1024 − 104; после освобождения возвращается тот же адрес |
| `free_list` | `free_list_respects_alignment` | после блока в 8 байт блок с выравниванием 64 выровнен по 64 |
| `free_list` | `free_list_returns_null_when_full` | блок в 1024 байта заполняет область; ещё 1 байт даёт null |
| `free_list` | `free_list_coalesces_neighbours` | три блока по 32 байта a, b, c; освободить a и c (2 свободных блока), затем b: 1 блок в 1024 байта |

В тестах `free_list` используйте область, выровненную по 16, чтобы числа были точными: `#[repr(align(16))] struct Region([u8; 1024]);`.

### Пошаговый план

#### M1 — `align_up`, который не переполняется

**Цель:** `src/align.rs` с `align_up` и тремя его тестами; `src/lib.rs` с `mod align; mod spin;` и строками `pub use`; готовый `src/spin.rs`.

**Указания:** начните с версии из Урока 12.13, но используйте `checked_add`, чтобы огромный адрес давал `None`, а не заворачивался в маленький. Оставьте `debug_assert!`, что `align` — степень двойки.

<details>
<summary>Подсказка</summary>

`addr.checked_add(align - 1)` возвращает `Option<usize>`; `.map(|a| a & !(align - 1))` завершает округление внутри `Option`.

</details>

**Готово, когда:**

```bash
cargo test --lib align_up
```

```text
running 3 tests
test align::tests::align_up_keeps_aligned_addresses ... ok
test align::tests::align_up_reports_overflow ... ok
test align::tests::align_up_rounds_up ... ok

test result: ok. 3 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.01s
```

(Тесты выполняются параллельно, поэтому порядок строк `test …` может отличаться. Позже, когда появятся другие тесты, число «filtered out» вырастет — это нормально.) Затем `git add -A` и `git commit -m "feat(p12-allocator): align_up with overflow check"`.

#### M2 — Bump-аллокатор

**Цель:** `BumpAlloc<N>` в `src/bump.rs` реализует `GlobalAlloc`, проходит три своих теста и запускает настоящую программу в роли глобального аллокатора.

**Указания:**

1. Поля: арена `UnsafeCell<[u8; N]>` (`UnsafeCell`, потому что аллокатор выдаёт память для записи через `&self`) и `SpinLock<BumpState>`, где `BumpState` хранит `next` (смещение в арене) и `live` (число живых блоков).
2. `new` должен быть `const fn`, потому что `static` инициализируется при компиляции: `UnsafeCell::new([0; N])` и `SpinLock::new(…)` оба `const`.
3. `BumpAlloc` содержит `UnsafeCell`, поэтому автоматически не является `Sync`. Добавьте `unsafe impl<const N: usize> Sync for BumpAlloc<N> {}` с комментарием `// SAFETY:`, объясняющим, почему совместное использование безопасно.
4. В `alloc`: возьмите блокировку; вычислите выровненный **адрес**, а не выровненное смещение (сама арена может начинаться по любому адресу); проверьте конец относительно `N`; при любой неудаче возвращайте `null_mut()`.
5. Превращайте смещение в указатель через `base.wrapping_add(offset)`, где `base` — это `self.arena.get().cast::<u8>()`. Не превращайте число обратно в указатель через `as *mut u8`: указатель потеряет информацию о том, к какому выделению памяти он относится, и Miri сообщит об этом.
6. Clippy попросит реализацию `Default`, потому что `new()` не принимает аргументов; добавьте такую, которая вызывает `new()`.

<details>
<summary>Подсказка</summary>

Ядро `alloc` без обработки ошибок: `let start = align_up(base as usize + state.next, layout.align())?; let offset = start - base as usize; let end = offset + layout.size();`, затем сравните `end` с `N`. Поскольку `alloc` возвращает указатель, а не `Option`, используйте `let … else { return ptr::null_mut(); }` для каждого шага, который может не удаться.

</details>

Теперь напишите `examples/bump_global.rs`. Он устанавливает bump-аллокатор для всей программы, а затем просит больше памяти, чем есть в арене:

```rust
use p12_allocator::BumpAlloc;

#[global_allocator]
static GLOBAL: BumpAlloc<{ 1024 * 1024 }> = BumpAlloc::new();

fn main() {
    let numbers: Vec<u64> = (1..=1000).collect();
    let text = format!("sum = {}", numbers.iter().sum::<u64>());
    println!("{text}");
    println!("used {} of {} bytes", GLOBAL.used(), 1024 * 1024);

    // Now ask for more than the arena holds.
    let big = vec![0u8; 2 * 1024 * 1024];
    println!("never printed: {}", big.len());
}
```

**Готово, когда:**

```bash
cargo test --lib bump
cargo run --example bump_global; echo "exit code $?"
```

```text
running 3 tests
test bump::tests::bump_blocks_are_aligned_and_separate ... ok
test bump::tests::bump_resets_when_everything_is_freed ... ok
test bump::tests::bump_returns_null_when_full ... ok

test result: ok. 3 passed; 0 failed; 0 ignored; 0 measured; 3 filtered out; finished in 0.00s

sum = 500500
used 9720 of 1048576 bytes
memory allocation of 2097152 bytes failed
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
exit code 134
```

Ваше число `used` может немного отличаться: оно включает память, которую стандартная библиотека выделила до и во время `main` (например, буфер стандартного вывода). Важны последние строки: ваш `alloc` вернул null, поэтому Rust сообщил о неудачном выделении и аварийно завершил программу (код возврата 134). Коммит: `feat(p12-allocator): bump allocator`.

#### M3 — Куча со списком свободных блоков: выделение, разделение, освобождение

**Цель:** `FreeListHeap` в `src/free_list.rs` с `empty`, `add_region`, `alloc`, `dealloc`, `free_bytes`, `free_blocks` и первыми четырьмя тестами `free_list`. Слияние будет в M4.

**Указания:**

1. Список свободных блоков живёт внутри свободной памяти. Каждый свободный блок начинается с заголовка:

```text
struct FreeBlock {
    size: usize,                       // size of this free block in bytes
    next: Option<NonNull<FreeBlock>>,  // next free block, higher address
}
```

Это 16 байт с выравниванием 8, поэтому самый маленький блок — 16 байт, а размер каждого блока кратен 8 (правило 6 контракта). Вспомогательная функция `fn block_size(layout: Layout) -> Option<usize>` округляет `layout.size().max(16)` вверх до кратного 8. Сама куча хранит только `head: Option<NonNull<FreeBlock>>`.

2. Сначала напишите одну приватную вспомогательную функцию: `unsafe fn insert(&mut self, ptr: *mut u8, size: usize)`. Она проходит список до первого блока с большим адресом, записывает новый `FreeBlock` по `ptr` через `ptr.cast::<FreeBlock>().write(…)` и вставляет его в список. `add_region` (после выравнивания начала по 8 и обрезки размера до кратного 8) и `dealloc` просто вызывают `insert`.
3. `alloc` проходит список с указателем `prev`. Для каждого блока решите, подходит ли запрос: начальный адрес — адрес блока, округлённый вверх до `layout.align()` (не меньше 8). Любой **передний** промежуток должен быть 0 или не меньше 16 байт (чтобы он мог остаться свободным блоком), и **остаток** после блока тоже должен быть 0 или не меньше 16. Если блок подходит: исключите его из списка, вставьте через `insert` передний промежуток и остаток, если они не пусты, и верните начало.
4. `free_bytes` и `free_blocks` проходят список. Итератор, построенный через `std::iter::from_fn`, делает их короткими.
5. `FreeListHeap` хранит сырые указатели, поэтому не является `Send`. В M4 это понадобится; добавьте `unsafe impl Send for FreeListHeap {}` с комментарием `// SAFETY:` (куча указывает только в память, которой управляет сама).

<details>
<summary>Подсказка 1: проверка, подходит ли блок</summary>

Напишите её как чистую функцию над числами — такую легко тестировать: `fn place(block_addr, block_size, size, align) -> Option<(usize, usize)>` возвращает `(передний промежуток, начальный адрес)`. Если первое выровненное начало оставляет передний промежуток от 1 до 15 байт, попробуйте снова от `block_addr + 16`. Верните `None`, если `start + size` выходит за конец блока или если остаток от 1 до 15 байт.

</details>

<details>
<summary>Подсказка 2: изменение списка</summary>

Чтобы исключить блок, присвойте `next` блока либо `prev.next`, либо `self.head`. Получайте `&mut FreeBlock` из `NonNull` через `unsafe { block.as_mut() }`, а читайте через `block.as_ref()`. Скопируйте `size` и `next` из блока **до того**, как запишете новый заголовок в ту же память.

</details>

**Готово, когда:**

```bash
cargo test --lib free_list
cargo +nightly miri test --lib
```

Первая команда показывает четыре теста: `free_list_respects_alignment`, `free_list_returns_null_when_full`, `free_list_splits_and_reuses` и `free_list_starts_with_one_block`, все `... ok`, и заканчивается на `test result: ok. 4 passed`. Miri печатает те же имена тестов и заканчивает на `test result: ok. 10 passed` (3 + 3 + 4) без «Undefined Behavior». Коммит: `feat(p12-allocator): free-list heap with first fit and splitting`.

#### M4 — Слияние и глобальный аллокатор со списком свободных блоков

**Цель:** освобождённые блоки сливаются с соседями; `FreeListAlloc<N>` реализует `GlobalAlloc`; тест с потоками и второй пример проходят.

**Указания:**

1. Слияние происходит в `insert`, потому что список отсортирован: после вставки нового блока между `prev` и `next` сначала проверьте, заканчивается ли новый блок ровно там, где начинается `next` (тогда поглотите `next`), затем — заканчивается ли `prev` ровно там, где начинается новый блок (тогда пусть `prev` поглотит новый блок). Такой порядок справляется и со случаем, когда блок закрывает промежуток между двумя свободными блоками.
2. Clippy может предложить *цепочку let* (let chain) для кода вроде `if let Some(next) = current { if … { … } }`: `if let Some(next) = current && addr + size == next.as_ptr() as usize { … }` (Урок 04.5).
3. У `FreeListAlloc<N>` есть арена `UnsafeCell<[u8; N]>` и `SpinLock<(bool, FreeListHeap)>`. `bool` говорит, отдана ли уже арена куче: `new()` — `const`, а `const fn` не может знать итоговый адрес арены, поэтому `add_region` вызывает первый `alloc`.
4. `stats()` берёт блокировку и возвращает `(free_bytes(), free_blocks())`.

<details>
<summary>Подсказка</summary>

Тест слияния в числах: после `a`, `b`, `c` (по 32 байта, на 0, 32, 64) освобождение `a` даёт список `[0..32)`, `[96..1024)`; освобождение `c` сливает его с блоком после него: `[0..32)`, `[64..1024)`, два блока; освобождение `b` касается обоих: `[0..1024)`, один блок.

</details>

Напишите `tests/threads.rs`. Он использует аллокатор напрямую (не как глобальный) из четырёх потоков сразу:

```rust
use p12_allocator::FreeListAlloc;
use std::alloc::{GlobalAlloc, Layout};
use std::thread;

static ALLOC: FreeListAlloc<{ 64 * 1024 }> = FreeListAlloc::new();

#[test]
fn free_list_alloc_works_from_many_threads() {
    let rounds = if cfg!(miri) { 20 } else { 10_000 };
    thread::scope(|s| {
        for t in 0..4u8 {
            s.spawn(move || {
                let layout = Layout::from_size_align(48, 8).unwrap();
                for _ in 0..rounds {
                    // SAFETY: the layout has a non-zero size.
                    let p = unsafe { ALLOC.alloc(layout) };
                    assert!(!p.is_null());
                    // SAFETY: `p` points to 48 bytes that this thread owns.
                    unsafe {
                        p.write_bytes(t, 48);
                        assert!(std::slice::from_raw_parts(p, 48).iter().all(|&b| b == t));
                        ALLOC.dealloc(p, layout);
                    }
                }
            });
        }
    });
    assert_eq!(ALLOC.stats().1, 1, "all blocks merged again");
}
```

И `examples/free_list_global.rs`, который трижды запускает подсчёт слов на вашем аллокаторе и печатает его состояние:

```rust
use p12_allocator::FreeListAlloc;
use std::collections::HashMap;

#[global_allocator]
static GLOBAL: FreeListAlloc<{ 4 * 1024 * 1024 }> = FreeListAlloc::new();

fn main() {
    let (free, blocks) = GLOBAL.stats();
    println!("at start of main: {free} bytes free in {blocks} blocks");

    for round in 1..=3 {
        let mut words: HashMap<String, usize> = HashMap::new();
        for i in 0..10_000 {
            *words.entry(format!("word{}", i % 500)).or_insert(0) += 1;
        }
        let (free, blocks) = GLOBAL.stats();
        println!(
            "round {round}: {} words, {free} bytes free in {blocks} blocks",
            words.len()
        );
    }

    let (free, blocks) = GLOBAL.stats();
    println!("at the end:  {free} bytes free in {blocks} blocks");
}
```

**Готово, когда:**

```bash
cargo test --lib free_list
cargo test --test threads
cargo run --release --example free_list_global
```

```text
running 5 tests
test free_list::tests::free_list_coalesces_neighbours ... ok
test free_list::tests::free_list_respects_alignment ... ok
test free_list::tests::free_list_returns_null_when_full ... ok
test free_list::tests::free_list_splits_and_reuses ... ok
test free_list::tests::free_list_starts_with_one_block ... ok

test result: ok. 5 passed; 0 failed; 0 ignored; 0 measured; 6 filtered out; finished in 0.00s

running 1 test
test free_list_alloc_works_from_many_threads ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.02s

at start of main: 4193680 bytes free in 1 blocks
round 1: 500 words, 4150792 bytes free in 2 blocks
round 2: 500 words, 4150792 bytes free in 2 blocks
round 3: 500 words, 4150792 bytes free in 2 blocks
at the end:  4192592 bytes free in 1 blocks
```

Как читать пример: стандартная библиотека использовала 624 байта до старта `main` (4 194 304 − 4 193 680). Во время раунда словарь и его строки живы; когда `words` уничтожается в конце раунда, всё освобождается и сливается, поэтому каждый раунд начинается из одного и того же состояния и требует одной и той же памяти. В конце используется на 1088 байт больше, чем в начале `main`: в основном это буфер стандартного вывода (1 KiB), который создал первый `println!` и который живёт до завершения программы. Ваши числа могут немного отличаться; картина (одинаковые числа в каждом раунде, один блок в конце) должна быть той же. Коммит: `feat(p12-allocator): coalescing and global free-list allocator`.

#### M5 — Нагрузочный тест, Miri, README

**Цель:** случайный нагрузочный тест, чистый запуск всего под Miri, README и чистая проверка качества.

Напишите `tests/stress.rs`. Он выполняет тысячи случайных выделений и освобождений, заполняет каждый блок байтом-меткой, проверяет метку перед освобождением и в конце проверяет, что куча снова один блок:

```rust
use p12_allocator::FreeListHeap;
use std::alloc::Layout;

fn next_random(state: &mut u64) -> u64 {
    *state ^= *state << 13;
    *state ^= *state >> 7;
    *state ^= *state << 17;
    *state
}

#[test]
fn random_allocs_and_frees_keep_every_block_intact() {
    let steps = if cfg!(miri) { 300 } else { 20_000 };
    let mut region = vec![0u8; 64 * 1024];
    let mut heap = FreeListHeap::empty();
    // SAFETY: `region` lives until the end of the test and is used only by the heap.
    unsafe { heap.add_region(region.as_mut_ptr(), region.len()) };
    let initial = heap.free_bytes();

    let mut seed = 0x2545_F491_4F6C_DD1D;
    let mut live: Vec<(*mut u8, Layout, u8)> = Vec::new();
    for step in 0..steps {
        let r = next_random(&mut seed);
        if !r.is_multiple_of(3) || live.is_empty() {
            let size = 1 + (r >> 8) as usize % 300;
            let align = 1 << ((r >> 20) % 7); // 1, 2, 4, …, 64
            let layout = Layout::from_size_align(size, align).unwrap();
            let p = heap.alloc(layout);
            if p.is_null() {
                continue; // full for now: fine
            }
            assert_eq!(p as usize % align, 0, "misaligned block at step {step}");
            let fill = (step % 251) as u8;
            // SAFETY: `p` points to `size` bytes that we own.
            unsafe { p.write_bytes(fill, size) };
            live.push((p, layout, fill));
        } else {
            let index = (r >> 8) as usize % live.len();
            let (p, layout, fill) = live.swap_remove(index);
            // SAFETY: `p` points to `layout.size()` bytes that we still own.
            let bytes = unsafe { std::slice::from_raw_parts(p, layout.size()) };
            assert!(
                bytes.iter().all(|&b| b == fill),
                "block overwritten before step {step}"
            );
            // SAFETY: `p` came from `heap.alloc(layout)`.
            unsafe { heap.dealloc(p, layout) };
        }
    }
    for (p, layout, _) in live.drain(..) {
        // SAFETY: every remaining block came from `heap.alloc(layout)`.
        unsafe { heap.dealloc(p, layout) };
    }
    assert_eq!(heap.free_bytes(), initial);
    assert_eq!(
        heap.free_blocks(),
        1,
        "everything coalesced back into one block"
    );
}
```

Выделение на двух шагах из трёх и освобождение на третьем заставляют кучу то заполняться, то пустеть, поэтому проверяются и разделение, и слияние, и случай «всё занято». `cfg!(miri)` равен `true` только под Miri, который работает намного медленнее, поэтому там тест делает меньше шагов. Затем напишите `README.md`.

**Готово, когда:**

```bash
cargo test --test stress
cargo +nightly miri test
cargo clippy --all-targets -- -D warnings
cargo fmt --check
```

```text
running 1 test
test random_allocs_and_frees_keep_every_block_intact ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.07s
```

`cargo +nightly miri test` запускает все тесты под Miri и трижды заканчивается на `test result: ok`: 11 модульных тестов, нагрузочный тест, тест с потоками и 0 doc-тестов — без единого сообщения «Undefined Behavior» (это занимает около 10 секунд). `cargo clippy` печатает только `Checking p12-allocator …` и `Finished …`, а `cargo fmt --check` не печатает ничего. Коммит: `test(p12-allocator): stress test, Miri, README`.

> 🧪 **Тестирование:** когда нагрузочный тест падает, он сообщает шаг. Сделайте так, чтобы он падал рано и воспроизводимо: напечатайте шаг, уменьшайте `steps`, пока тест едва падает, и проверяйте список свободных блоков (`free_blocks`, `free_bytes`) после каждой операции. Поскольку у генератора случайных чисел фиксированное начальное значение, каждый запуск одинаков — именно это делает такую ошибку находимой.

### Дополнительные цели

1. **Классы размеров.** Добавьте `SegregatedAlloc`: отдельные списки свободных блоков для блоков в 16, 32, 64, 128, 256 и 512 байт, которые пополняются из bump-арены; запросы побольше идут в кучу со списком свободных блоков. Сравните скорость примера с `FreeListAlloc`.
2. **Бенчмарк.** Соберите версию `free_list_global`, которая использует `System`, и сравните обе через `hyperfine` (Урок 05.12) на большей нагрузке. Запишите числа в README вместе с компьютером, на котором измеряли.
3. **Лучший подходящий.** Добавьте режим поиска по принципу лучшего подходящего и измерьте фрагментацию: сколько свободных блоков и сколько свободных байт остаётся после фазы выделений нагрузочного теста при каждой стратегии?
4. **`realloc` на месте.** Переопределите `GlobalAlloc::realloc`: если блок за текущим свободен и достаточно велик, растите в него без копирования. Посчитайте, как часто это удаётся в примере.
5. **Пиковое использование.** Отслеживайте наибольший объём памяти, используемой в любой момент, и печатайте его в примерах.

## Частые ошибки

**1. Чтение заголовка после того, как он перезаписан.** В `alloc` остаток разделённого блока может начинаться внутри старого заголовка. Сначала скопируйте `size` и `next` в локальные переменные, затем исключите блок, затем вставьте куски через `insert`.

**2. Выравнивание смещения вместо адреса.** `align_up(offset, align)` неверно, когда арена начинается не по выровненному адресу. Выравнивайте `base + offset`, а затем снова вычитайте `base`.

**3. `std::sync::Mutex`, `println!` или `Vec` внутри `alloc`.** Всё, что может выделять память, может снова вызвать ваш аллокатор, пока он держит собственную блокировку: взаимная блокировка (deadlock) или бесконечная рекурсия (Урок 12.13). Используйте спин-блокировку и простые поля; отлаживайтесь по счётчикам из `stats()` после события.

**4. Забыть `cfg!(miri)` в длинных тестах.** Под Miri тест на 20 000 шагов работает много минут. Уменьшайте работу под Miri; ошибки он всё равно находит, потому что проверяет каждое обращение.

**5. Тестировать только один поток.** Глобальный аллокатор вызывается из каждого потока. Без теста с потоками отсутствующая блокировка может пройти все остальные тесты и испортить память в настоящей программе намного позже.

## Вопросы для самопроверки

1. Почему `BumpAlloc::new` и `FreeListAlloc::new` должны быть `const fn`?

<details>
<summary>Ответ</summary>

Глобальный аллокатор — это `static`, а `static` инициализируется при компиляции, поэтому его начальное значение должно быть `const`-выражением. Первое выделение происходит ещё до `main`, так что во время выполнения нет момента, чтобы его настроить.

</details>

2. Почему `FreeListAlloc` отдаёт арену куче при первом `alloc`, а не в `new`?

<details>
<summary>Ответ</summary>

В `const fn` итоговый адрес арены ещё не известен: `static` размещают в памяти компоновщик и загрузчик (Урок 12.12). Заголовки списка свободных блоков должны хранить настоящие адреса, поэтому добавить арену в кучу можно только во время выполнения, при первом использовании.

</details>

3. Зачем держать список свободных блоков отсортированным по адресам?

<details>
<summary>Ответ</summary>

Тогда соседи блока в памяти — это его соседи в списке, и при вставке для слияния достаточно сравнить новый блок с блоком перед ним и после него. В неотсортированном списке поиск соседей потребовал бы прохода по всему списку.

</details>

4. Что делает Rust, когда глобальный аллокатор возвращает null, и как вы это увидели?

<details>
<summary>Ответ</summary>

Он вызывает обработчик ошибки выделения памяти, который печатает «memory allocation of N bytes failed» и аварийно завершает процесс. Пример `bump_global` показал это сообщение и код возврата 134 (`SIGABRT`).

</details>

5. Почему здесь допустима спин-блокировка, хотя она тратит время процессора на ожидание?

<details>
<summary>Ответ</summary>

Блокировка удерживается всего несколько сотен инструкций (одна операция со списком), поэтому ожидание очень короткое, и она никогда не выделяет память и не обращается к операционной системе. Для «спящей» блокировки пришлось бы доказывать, что она не выделяет память ни на одной платформе.

</details>

## Итоги

- Вы построили два настоящих глобальных аллокатора: bump-аллокатор ради скорости и простоты и аллокатор со списком свободных блоков, который повторно использует память через первый подходящий, разделение и слияние.
- Арены в `UnsafeCell`, `unsafe impl Sync`, `const fn new` и блокировка, которая никогда не выделяет память, — строительные блоки любого `GlobalAlloc`.
- Сохраняйте происхождение указателей (provenance), получая указатели на блоки из указателя на арену (`wrapping_add`), и выравнивайте адреса, а не смещения.
- Проверяйте каждое правило модульным тестом, каждый байт — случайным нагрузочным тестом с фиксированным начальным значением, проверяйте работу из нескольких потоков и запускайте всё под Miri.
- Настоящие программы (`Vec`, `format!`, `HashMap`) работали на вашем аллокаторе, а его статистика показала, как память освобождается и сливается после каждого раунда.

## Источники и что почитать

- [std::alloc::GlobalAlloc](https://doc.rust-lang.org/std/alloc/trait.GlobalAlloc.html) — точные правила, которым должна следовать ваша реализация.
- [Philipp Oppermann — Allocator Designs](https://os.phil-opp.com/allocator-designs/) — те же устройства bump-аллокатора и аллокатора на связном списке, шаг за шагом для ядра.
- [std::ptr::NonNull](https://doc.rust-lang.org/std/ptr/struct.NonNull.html) и [std::ptr — Provenance](https://doc.rust-lang.org/std/ptr/index.html) — ненулевые сырые указатели и почему указатели нужно получать из того выделения, в которое они указывают.
- [std::cell::UnsafeCell](https://doc.rust-lang.org/std/cell/struct.UnsafeCell.html) — единственный законный способ менять данные за общей ссылкой.
- [Miri на GitHub](https://github.com/rust-lang/miri) — `cfg(miri)`, флаги и что Miri может и не может обнаружить.
- [The Rust Programming Language — 20.1 Unsafe Rust](https://doc.rust-lang.org/book/ch20-01-unsafe-rust.html) — официальная глава о `unsafe`, `unsafe impl` и сырых указателях.

---

[🏠 Начало](../START-HERE.md) · [← 12.13 Как работают аллокаторы памяти](13-how-memory-allocators-work.md) · [📋 Этап 12](README.md) · [12.15 Контрольная точка →](15-checkpoint.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
