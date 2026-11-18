<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 14.3 Условные переменные](03-condition-variables.md) · [📋 Этап 14](README.md) · [14.5 Атомарные операции →](05-atomics.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/14-concurrency-and-async/04-thread-pools-scoped-threads-and-rayon.md) <!-- nav -->

# Урок 14.4 — Пулы потоков, scoped-потоки и rayon

> **Этап 14 — Конкурентность и асинхронность подробно** · Урок 4 из 16 · ⏱ около 2,5 часа (2 занятия)

## Что вы узнаете

- Как разделить одно большое вычисление на все ядра CPU (Central Processing Unit, центральный процессор) с помощью scoped-потоков.
- Почему запускать поток на каждое маленькое задание медленно и как это исправляет пул потоков с каналом на много потребителей.
- Почему куски работы одинакового размера всё равно могут заканчиваться в разное время и как их выравнивает *перехват работы* (work stealing).
- Как пользоваться крейтом `rayon`: параллельные итераторы, `fold`/`reduce`, `par_sort` и `join`.
- Правила для кода на rayon: не блокировать, не делать куски слишком мелкими и всегда измерять.

## Перед началом

- Вы прошли [Урок 14.3 — Условные переменные](03-condition-variables.md).
- Вы знаете scoped-потоки из [Урока 05.10](../05-rust-intermediate/10-threads-channels-and-shared-state.md), итераторы и их адаптеры из [Урока 05.5](../05-rust-intermediate/05-iterators.md) и пул потоков из [Урока 06.11](../06-rust-advanced/11-project-multithreaded-web-server.md).
- В `~/projects/rust/code/p14-concurrency` добавьте два крейта:

```bash
cargo add rayon@1 crossbeam@0.8
```

Теперь в вашем `Cargo.toml` в разделе `[dependencies]` есть `rayon = "1"` и `crossbeam = "0.8"`.

## Новые слова

| Слово | Значение |
|---|---|
| **параллелизм данных** (data parallelism) | Выполнение *одной и той же* операции над многими кусками данных одновременно, на нескольких ядрах. |
| **кусок** (chunk) | Одна часть среза. `slice.chunks(n)` режет срез на куски по `n` элементов (последний может быть короче). |
| **`available_parallelism`** | Стандартная функция, которая говорит, сколько потоков эта машина действительно может выполнять одновременно (обычно — число ядер). |
| **P-ядро / E-ядро** (P-core / E-core) | Ядра *производительности* (быстрые) и *эффективности* (медленнее, тратят меньше энергии). В чипах Apple M есть оба вида. |
| **дисбаланс нагрузки** (load imbalance) | Одни потоки заканчивают рано и простаивают, а другие ещё работают, поэтому вся работа ждёт самый медленный поток. |
| **перехват работы** (work stealing) | Способ планирования: у каждого рабочего потока своя очередь задач; поток, которому нечего делать, *перехватывает* задачи из очереди занятого потока. |
| **rayon** | Крейт для параллелизма данных. Выполняет ваш итератор или замыкания на глобальном пуле потоков с перехватом работы. |
| **параллельный итератор** (parallel iterator) | Итератор rayon (`par_iter`, `into_par_iter`), элементы которого обрабатываются несколькими потоками. |
| **`fold` / `reduce`** | Два шага параллельного подсчёта: каждый поток *сворачивает* (fold) свои элементы в собственный частичный результат, затем частичные результаты *сводятся* (reduce) в один. |
| **MPMC** (multi-producer, multi-consumer, много производителей, много потребителей) | Канал, у которого может быть много отправителей **и** много получателей. `crossbeam::channel` — такой канал. |
| **гранулярность задач** (task granularity) | Насколько велик один кусок параллельной работы. Слишком мелкий — накладные расходы на планирование съедают выигрыш. Слишком крупный — плохой баланс. |
| **I/O** (input/output, ввод-вывод) | Чтение и запись данных вне программы: файлы, сеть, терминал. Поток, ждущий ввода-вывода, не делает работы на CPU. |
| **переполнение стека** (stack overflow) | Поток израсходовал всю память своего стека, обычно из-за слишком глубокой рекурсии. Программа останавливается. |

## 1. Разделите работу сами: scoped-потоки

Простейший вид параллелизма: одна большая работа, разрезанная на куски по одному на ядро, каждый кусок в своём потоке, а в конце результаты складываются. Для этого идеально подходят scoped-потоки (Урок 05.10), потому что потоки могут заимствовать данные. Создайте `src/bin/chunks.rs`:

```rust
use std::thread;

fn main() {
    let cores = thread::available_parallelism().map(|n| n.get()).unwrap_or(1);
    println!("this machine can run {cores} threads in parallel");

    let numbers: Vec<u64> = (1..=1_000_000).collect();
    let chunk_size = numbers.len().div_ceil(cores);

    let total: u64 = thread::scope(|s| {
        let handles: Vec<_> = numbers
            .chunks(chunk_size)
            .map(|chunk| s.spawn(move || chunk.iter().map(|x| x * x).sum::<u64>()))
            .collect();
        handles.into_iter().map(|h| h.join().unwrap()).sum()
    });

    println!("sum of squares = {total}");
}
```

```text
this machine can run 12 threads in parallel
sum of squares = 333333833333500000
```

**Что происходит:**

1. `available_parallelism()` возвращает `Result<NonZeroUsize>`; `.get()` превращает его в `usize`. Если OS (Operating System, операционная система) не может сказать, берём 1. У тестовой машины 12 ядер (8 P-ядер и 4 E-ядра); у вас число может быть другим.
2. `div_ceil` делит с округлением вверх, поэтому `cores` кусков покрывают все числа.
3. `chunks(chunk_size)` режет вектор на срезы, которые **заимствуют** из `numbers`. Каждый scoped-поток получает один срез (`move` перемещает в замыкание только ссылку на срез).
4. Сначала мы через `collect` собираем все дескрипторы, чтобы запустились все потоки, и только потом вызываем `join`. Если бы мы вызывали `join` внутри того же `map`, каждый поток дожидался бы до запуска следующего, и ничего не выполнялось бы параллельно.

Это хорошо работает, когда каждый кусок стоит одинаково. Раздел 3 показывает, что бывает, когда это не так.

## 2. Пулы потоков: не запускайте поток на каждое задание

Запуск потока не бесплатен: OS должна создать стек и управляющий блок и запланировать новый поток. Измерим. Создайте `src/bin/spawn_cost.rs`:

```rust
use std::thread;
use std::time::Instant;

fn main() {
    const JOBS: u32 = 10_000;

    // One new thread per tiny job.
    let start = Instant::now();
    for i in 0..JOBS {
        thread::spawn(move || i * 2).join().unwrap();
    }
    let per_thread = start.elapsed() / JOBS;
    println!("spawn + join one thread: {per_thread:?}");

    // The same tiny job without a thread.
    let start = Instant::now();
    let mut sum = 0u64;
    for i in 0..JOBS {
        sum += std::hint::black_box(i * 2) as u64;
    }
    let per_call = start.elapsed() / JOBS;
    println!("the job itself:          {per_call:?} (sum {sum})");
}
```

`cargo run --release --bin spawn_cost`:

```text
spawn + join one thread: 17.56µs
the job itself:          0ns (sum 99990000)
```

Около 18 микросекунд на поток — ради работы, которая занимает меньше наносекунды. **Пул потоков** платит эту цену один раз: он запускает N рабочих потоков в начале и затем подаёт им задания через очередь.

В Уроке 06.11 рабочие потоки делили один `mpsc::Receiver` через `Arc<Mutex<Receiver>>`, потому что у стандартного канала `mpsc` только один получатель. В крейте `crossbeam` есть канал **MPMC**: получателя можно клонировать (`clone`), и у каждого рабочего потока будет свой. Создайте `src/bin/pool.rs`:

```rust
use crossbeam::channel::{self, Sender};
use std::thread::{self, JoinHandle};

type Job = Box<dyn FnOnce() + Send + 'static>;

struct ThreadPool {
    sender: Option<Sender<Job>>,
    workers: Vec<JoinHandle<()>>,
}

impl ThreadPool {
    fn new(threads: usize, queue: usize) -> Self {
        let (sender, receiver) = channel::bounded::<Job>(queue);
        let workers = (0..threads)
            .map(|_| {
                let receiver = receiver.clone(); // many receivers: no Mutex needed
                thread::spawn(move || {
                    for job in receiver {
                        job();
                    }
                })
            })
            .collect();
        ThreadPool {
            sender: Some(sender),
            workers,
        }
    }

    fn execute(&self, job: impl FnOnce() + Send + 'static) {
        // Blocks while the queue is full: backpressure.
        self.sender.as_ref().unwrap().send(Box::new(job)).unwrap();
    }
}

impl Drop for ThreadPool {
    fn drop(&mut self) {
        drop(self.sender.take()); // close the channel: workers' loops end
        for worker in self.workers.drain(..) {
            worker.join().unwrap();
        }
    }
}

fn main() {
    let pool = ThreadPool::new(4, 16);
    for id in 0..8 {
        pool.execute(move || {
            let name = thread::current().id();
            println!("job {id} ran on {name:?}");
        });
    }
    drop(pool); // waits for all jobs
    println!("all jobs done");
}
```

```text
job 2 ran on ThreadId(5)
job 4 ran on ThreadId(5)
job 5 ran on ThreadId(5)
job 6 ran on ThreadId(5)
job 7 ran on ThreadId(5)
job 0 ran on ThreadId(4)
job 1 ran on ThreadId(2)
job 3 ran on ThreadId(3)
all jobs done
```

**Что происходит:**

1. `Job` — упакованное в `Box` замыкание, которое можно вызвать один раз и отправить в другой поток: трейт-объект ([Урок 06.1](../06-rust-advanced/01-trait-objects-and-dynamic-dispatch.md)).
2. `channel::bounded(16)` создаёт канал MPMC, который держит не больше 16 заданий. Как и очередь из Урока 14.3, `send` блокирует, когда канал полон: обратное давление.
3. Каждый рабочий поток владеет клоном получателя и выполняет цикл `for job in receiver`. Цикл берёт задания одно за другим и заканчивается, когда канал закрыт и пуст.
4. `Drop` уничтожает единственного отправителя. Это закрывает канал; рабочие потоки доделывают оставшиеся задания и выходят из своих циклов; затем `join` дожидается каждого из них. Это корректное завершение из Урока 06.11 в нескольких строках.
5. Какой рабочий поток получит какое задание, решается во время выполнения. Здесь один поток случайно забрал пять заданий. Идентификаторы потоков и порядок у вас будут другими.

> 💡 **Совет:** в `crossbeam::channel` есть также `unbounded()`, тайм-ауты (`recv_timeout`) и макрос `select!`, который ждёт сразу на нескольких каналах. Когда нужен канал между потоками с несколькими получателями, обычно выбирают его.

## 3. Неравная работа и перехват работы

Куски одинакового размера справедливы, только когда каждый элемент стоит одинаково. Часто это не так. Хороший пример — подсчёт простых чисел: проверить, простое ли большое число, дольше, чем проверить маленькое. Вдобавок в чипе Apple M есть быстрые P-ядра и более медленные E-ядра, поэтому даже одинаковая работа заканчивается в разное время.

```text
 equal chunks, 4 threads (time →)
 thread 1  ███                       done, idle .........
 thread 2  ██████                    done, idle ......
 thread 3  █████████                 done, idle ...
 thread 4  ████████████              ← everybody waits for this one
```

Это **дисбаланс нагрузки**: работа закончена, только когда закончен самый медленный поток.

**Перехват работы** исправляет это без того, чтобы вам угадывать стоимость. Работа режется на множество *маленьких* задач. У каждого рабочего потока своя очередь задач. Когда очередь потока пуста, он не простаивает: он **перехватывает** задачу с другого конца очереди занятого потока.

```text
 worker A queue: [t1 t2 t3 t4 t5 t6]   ← A takes its next task from this end (t6, then t5, ...)
 worker B queue: [ ]                   ← B is idle, so it steals t1 from the OTHER end of A's queue
```

Раз владелец и перехватчик берут задачи с противоположных концов, они редко трогают одну и ту же задачу, поэтому перехват дешёвый. Результат: все ядра заняты до самого конца. Крейт `rayon` делает именно это за вас.

## 4. rayon: параллельные итераторы

rayon превращает обычный итератор в параллельный: замените `iter()` на `par_iter()` (или `into_iter()` на `into_par_iter()`), и элементы будут обрабатываться на глобальном пуле потоков rayon, в котором по одному рабочему потоку на ядро и который использует перехват работы. Нужен один импорт: `use rayon::prelude::*;`. Создайте `src/bin/rayon_tour.rs`:

```rust
use rayon::prelude::*;

fn fib(n: u32) -> u64 {
    if n < 2 {
        n as u64
    } else {
        fib(n - 1) + fib(n - 2)
    }
}

fn main() {
    // 1. par_iter: like iter, but the items are shared out between threads.
    let words = vec!["apple", "banana", "cherry", "date"];
    let lengths: Vec<usize> = words.par_iter().map(|w| w.len()).collect();
    println!("lengths = {lengths:?}");

    // 2. Ranges and sums.
    let total: u64 = (1..=100u64).into_par_iter().sum();
    println!("total = {total}");

    // 3. Sorting a slice in parallel.
    let mut numbers: Vec<i32> = (0..10).rev().collect();
    numbers.par_sort();
    println!("sorted = {numbers:?}");

    // 4. join: run two closures, maybe in parallel, and wait for both.
    let (a, b) = rayon::join(|| fib(30), || fib(31));
    println!("fib(30) = {a}, fib(31) = {b}");
}
```

```text
lengths = [5, 6, 6, 4]
total = 5050
sorted = [0, 1, 2, 3, 4, 5, 6, 7, 8, 9]
fib(30) = 832040, fib(31) = 1346269
```

**Что происходит:**

1. `par_iter()` даёт параллельный итератор по элементам `&str`. `map` и `collect` выглядят как обычные. `collect` сохраняет **исходный порядок**, хотя элементы обрабатывались в любом порядке и в любых потоках.
2. `into_par_iter()` на диапазоне делит его на куски; `sum` складывает частичные суммы.
3. `par_sort()` — параллельная версия `sort()` (стабильная, как `sort`); есть и `par_sort_unstable()`.
4. `rayon::join(a, b)` — основной строительный блок rayon: он кладёт `b` в очередь текущего рабочего потока, выполняет `a`, потом выполняет `b` сам, если только другой рабочий поток тем временем не перехватил `b`. Затем он ждёт обоих. В Упражнении 3 на нём строится параллельная сортировка.

Теперь честное сравнение на неравной работе. Создайте `src/bin/primes.rs`:

```rust
use rayon::prelude::*;
use std::thread;
use std::time::Instant;

/// Slow on purpose: trial division. Bigger numbers take longer.
fn is_prime(n: u64) -> bool {
    if n < 2 {
        return false;
    }
    let mut d = 2;
    while d * d <= n {
        if n.is_multiple_of(d) {
            return false;
        }
        d += 1;
    }
    true
}

const N: u64 = 5_000_000;

fn main() {
    let start = Instant::now();
    let count = (1..=N).filter(|&n| is_prime(n)).count();
    println!("sequential   : {count} primes in {:.1?}", start.elapsed());

    let threads = thread::available_parallelism().unwrap().get() as u64;
    let start = Instant::now();
    let count: usize = thread::scope(|s| {
        let handles: Vec<_> = (0..threads)
            .map(|t| {
                let (from, to) = (t * N / threads + 1, (t + 1) * N / threads);
                s.spawn(move || (from..=to).filter(|&n| is_prime(n)).count())
            })
            .collect();
        handles.into_iter().map(|h| h.join().unwrap()).sum()
    });
    println!("equal chunks : {count} primes in {:.1?}", start.elapsed());

    let start = Instant::now();
    let count = (1..=N).into_par_iter().filter(|&n| is_prime(n)).count();
    println!("rayon        : {count} primes in {:.1?}", start.elapsed());
}
```

`cargo run --release --bin primes`, три запуска:

```text
sequential   : 348513 primes in 422.6ms
equal chunks : 348513 primes in 62.1ms
rayon        : 348513 primes in 51.4ms
sequential   : 348513 primes in 409.5ms
equal chunks : 348513 primes in 62.6ms
rayon        : 348513 primes in 51.3ms
sequential   : 348513 primes in 412.4ms
equal chunks : 348513 primes in 62.7ms
rayon        : 348513 primes in 54.2ms
```

**Что происходит:**

1. `is_prime` перебирает все делители до квадратного корня, поэтому числа около 5 000 000 стоят дороже чисел около 1, а дороже всего простые числа (нет раннего выхода).
2. **Равные куски**: 12 потоков, у каждого один диапазон одинаковой длины. В последних диапазонах самые дорогие числа, а часть потоков работает на E-ядрах. Результат примерно в 6,6 раза быстрее последовательного, а не в 12: часть потоков в конце простаивает.
3. **rayon**: одна строка кода, примерно в 8 раз быстрее последовательной версии и примерно на 18 % быстрее наших кусков, потому что перехват работы держит каждое ядро занятым до конца.
4. Все три версии находят одинаковое число простых. Параллельная версия, которая даёт другой ответ, — это ошибка, а не ускорение; именно это проверяют Упражнения 1 и 2.

> ⚡ **Производительность:** двенадцать ядер никогда не дают ускорения в 12 раз. Часть работы нельзя разделить, потоки нужно запускать и дожидаться, результаты нужно объединять, а E-ядра медленнее. Измеряйте в релизном режиме, сравнивайте с последовательной версией и оставляйте параллельную, только если она явно быстрее.

## 5. Правила для кода на rayon

1. **Замыкания должны быть `Send + Sync`**, потому что их одновременно вызывают многие потоки. Компилятор это проверяет: используйте `Rc<i32>` внутри `par_iter().map(...)`, и сборка остановится с ``error[E0277]: `Rc<i32>` cannot be shared between threads safely``, как в Уроке 05.10.
2. **Не блокируйте внутри rayon.** У rayon по одному потоку на ядро. Замыкание, которое спит, долго ждёт блокировку или читает из сети, держит ядро в простое, и другие задачи не могут им воспользоваться. rayon — для работы, **нагружающей CPU**. Ожидание ввода-вывода (I/O) — работа асинхронного кода (Уроки 14.10–14.14).
3. **Не делайте задачи слишком мелкими.** Параллельно сложить 10 чисел медленнее, чем последовательно: планирование стоит дороже самой работы. rayon уже разумно делит итераторы. В рекурсивном коде с `join` переходите на последовательный код ниже порога, как в Упражнении 3.
4. **Управляйте числом потоков, когда нужно.** Переменная окружения `RAYON_NUM_THREADS` задаёт размер глобального пула. На тестовой машине `RAYON_NUM_THREADS=1 cargo run --release --bin primes` напечатал `rayon : 348513 primes in 397.7ms` (никакого выигрыша), а `RAYON_NUM_THREADS=4` — `107.6ms`. Для отдельного пула внутри программы используйте `rayon::ThreadPoolBuilder::new().num_threads(4).build()`.
5. **С общими результатами нужна осторожность.** Не добавляйте в `Mutex<Vec<_>>` из каждого элемента: все потоки будут ждать эту блокировку (Урок 14.2). Возвращайте значения, и пусть их объединяют `collect`, `sum` или `fold`/`reduce`, как в Упражнении 2.

Какой инструмент выбрать:

| Ситуация | Инструмент |
|---|---|
| Разделить вычисление над имеющимися данными на равные куски | `thread::scope` + `chunks` |
| Одна операция над многими элементами разной стоимости | rayon `par_iter` |
| Рекурсия «разделяй и властвуй» (сортировка, обход дерева) | `rayon::join` |
| Много независимых заданий, приходящих со временем | пул потоков с каналом |
| Много заданий, которые в основном *ждут* (сеть, диск, таймеры) | async с Tokio (Уроки 14.13 и 14.14) |

> 🧪 **Тестирование:** для каждой параллельной функции держите простую последовательную версию и пишите тест, который сравнивает обе на многих входах, включая граничные случаи (пустой вход, один элемент, вход меньше числа потоков). Последовательная версия — ваша *эталонная реализация* (reference implementation).

## Частые ошибки

1. **`join` внутри цикла, который запускает потоки.** `for c in chunks { s.spawn(...).join() }` выполняет потоки один за другим. Запустите все, соберите дескрипторы, потом вызывайте `join`.
2. **Параллельный код без последовательной проверки.** Неверный `reduce` или гонка могут дать чуть-чуть неверный результат, которого никто не заметит. Сравнивайте с последовательной версией в тесте.
3. **Блокировка или сон в замыкании rayon.** Это блокирует целое ядро пула rayon. Для ожидания используйте отдельный поток или асинхронный код.
4. **Измерения в отладочном режиме.** Отладочные сборки во много раз медленнее и искажают картину. Для измерения скорости всегда `--release`.
5. **Ожидание ускорения для крошечной работы.** Запуск потоков и планирование задач стоят микросекунды; работа на наносекунды при распараллеливании становится медленнее.

## Попробуйте сами

1. Запустите `cargo run --release --bin primes` с `RAYON_NUM_THREADS=1`, `2`, `4`, `8` и без переменной. Запишите времена и посмотрите, где выигрыш перестаёт расти.
2. В `primes.rs` перепишите часть «equal chunks» так, чтобы `join` каждого потока вызывался внутри `map`: замените `s.spawn(move || …)` на `s.spawn(move || …).join().unwrap()`, а последнюю строку области видимости — на `.sum()` сразу после `map`. Количество по-прежнему верное, но на тестовой машине время выросло примерно с `62ms` до `426.1ms` — столько же, сколько у последовательной версии. Почему?
3. В `pool.rs` замените `ThreadPool::new(4, 16)` на `ThreadPool::new(1, 16)`. Какие идентификаторы потоков вы видите теперь? (На тестовой машине все задания выполнились на `ThreadId(2)`, по порядку от 0 до 7.)

## Упражнения

### Упражнение 1 — Параллельная сумма квадратов с тестом (легко)

Напишите `src/bin/squares.rs` с функциями `sum_of_squares_seq(n: u64) -> u64` (обычный итератор) и `sum_of_squares_par(n: u64) -> u64` (rayon). `main` печатает обе для `n = 1_000_000`. Напишите два теста: функции совпадают для `n` из `[0, 1, 2, 10, 1000, 1_000_000]`, и параллельный результат равен формуле `n(n + 1)(2n + 1) / 6`.

<details>
<summary>Подсказка</summary>

`(1..=n).into_par_iter().map(|x| x * x).sum()`. Вверху поставьте `use rayon::prelude::*;`. В тесте добавьте значение `n` в сообщение assert, чтобы при падении было видно, какой вход не прошёл.

</details>

<details>
<summary>Ответ</summary>

```rust
use rayon::prelude::*;

fn sum_of_squares_seq(n: u64) -> u64 {
    (1..=n).map(|x| x * x).sum()
}

fn sum_of_squares_par(n: u64) -> u64 {
    (1..=n).into_par_iter().map(|x| x * x).sum()
}

fn main() {
    let n = 1_000_000;
    println!("parallel   {}", sum_of_squares_par(n));
    println!("sequential {}", sum_of_squares_seq(n));
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn parallel_equals_sequential() {
        for n in [0, 1, 2, 10, 1000, 1_000_000] {
            assert_eq!(sum_of_squares_par(n), sum_of_squares_seq(n), "n = {n}");
        }
    }

    #[test]
    fn matches_the_formula() {
        let n = 1_000_000u64;
        assert_eq!(sum_of_squares_par(n), n * (n + 1) * (2 * n + 1) / 6);
    }
}
```

```text
parallel   333333833333500000
sequential 333333833333500000
```

`cargo test --bin squares` показывает оба теста `ok`.

Почему это работает: сложение целых чисел даёт один и тот же результат при любом порядке и любой группировке, поэтому частичные суммы потоков складываются в ту же сумму. (Для чисел с плавающей точкой это неверно: там параллельная сумма может отличаться в последних знаках.)

</details>

### Упражнение 2 — Параллельный подсчёт слов (средне)

Напишите в `src/bin/word_count.rs` функции `count_seq(lines: &[&str]) -> HashMap<String, usize>` и `count_par` (rayon). Слова разделены пробельными символами и считаются в нижнем регистре. `count_par` не должна использовать `Mutex`: через `fold` постройте по одному `HashMap` на поток, а через `reduce` слейте их. Проверьте тестом на 10 000 копиях одной строки, что обе версии дают одинаковый словарь.

<details>
<summary>Подсказка</summary>

`fold(HashMap::new, |mut counts, line| { …; counts })` — первый аргумент это функция, которая создаёт пустой словарь для каждого потока. `reduce(HashMap::new, |mut a, b| { for (word, n) in b { … } a })` сливает два словаря в один. `std::iter::repeat_n(text, 10_000).collect()` строит вход для теста.

</details>

<details>
<summary>Ответ</summary>

```rust
use rayon::prelude::*;
use std::collections::HashMap;

fn count_seq(lines: &[&str]) -> HashMap<String, usize> {
    let mut counts = HashMap::new();
    for line in lines {
        for word in line.split_whitespace() {
            *counts.entry(word.to_lowercase()).or_insert(0) += 1;
        }
    }
    counts
}

fn count_par(lines: &[&str]) -> HashMap<String, usize> {
    lines
        .par_iter()
        .fold(HashMap::new, |mut counts, line| {
            for word in line.split_whitespace() {
                *counts.entry(word.to_lowercase()).or_insert(0) += 1;
            }
            counts
        })
        .reduce(HashMap::new, |mut a, b| {
            for (word, n) in b {
                *a.entry(word).or_insert(0) += n;
            }
            a
        })
}

fn main() {
    let lines = ["the cat sat", "The dog sat", "a cat and a dog"];
    let counts = count_par(&lines);
    assert_eq!(counts, count_seq(&lines));
    let mut pairs: Vec<_> = counts.into_iter().collect();
    pairs.sort();
    println!("{pairs:?}");
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn parallel_count_equals_sequential_count() {
        let text = "one two three two three three Four four FOUR four";
        let lines: Vec<&str> = std::iter::repeat_n(text, 10_000).collect();
        let par = count_par(&lines);
        assert_eq!(par, count_seq(&lines));
        assert_eq!(par["four"], 40_000);
    }
}
```

```text
[("a", 2), ("and", 1), ("cat", 2), ("dog", 2), ("sat", 2), ("the", 2)]
```

Почему это работает: во время `fold` каждый поток пишет только в свой словарь, поэтому нечего блокировать и нечего ждать. Затем `reduce` попарно сливает словари, тоже параллельно. У `HashMap` нет порядка, поэтому `main` сортирует пары перед печатью, чтобы вывод был стабильным.

</details>

### Упражнение 3 — Параллельная быстрая сортировка с `rayon::join` (сложно)

Напишите `par_quicksort<T: Ord + Copy + Send>(v: &mut [T])` в `src/bin/par_quicksort.rs`. Разделите срез вокруг опорного элемента (*разбиение*, partition), затем отсортируйте две стороны через `rayon::join`. Срезы из 1000 элементов и меньше сортируйте через `sort_unstable`. Проверьте тестом против стандартного `sort` на псевдослучайных данных нескольких длин (сгенерируйте их сами парой арифметических строк, без лишнего крейта) и добавьте тест с 50 000 одинаковых элементов.

<details>
<summary>Подсказка</summary>

`split_at_mut(mid)` даёт два непересекающихся среза `&mut` — именно это нужно проверке заимствований для `join(|| sort(left), || sort(right))`.

Если ваша первая версия берёт последний элемент как опорный и кладёт всё «не меньшее» направо, запустите тест с одинаковыми элементами. Что происходит с размером правой стороны, когда все элементы равны? Насколько глубокой становится рекурсия?

</details>

<details>
<summary>Ответ</summary>

Первая версия с простым двухсторонним разбиением проходит случайный тест, но падает на одинаковых элементах:

```text
thread '<unknown>' (5663817) has overflowed its stack
fatal runtime error: stack overflow, aborting
```

С 50 000 одинаковых элементов каждое разбиение ставит на место только один элемент (опорный), а остальные 49 999 остаются на одной стороне. Рекурсия уходит примерно на 50 000 уровней, и потоку не хватает стека. Исправление — **трёхстороннее разбиение**: меньшие, равные, большие. Равная часть уже стоит на своём месте и повторно не сортируется.

```rust
use std::cmp::Ordering;

/// Sorts `v` with quicksort; the two sides are sorted with `rayon::join`.
fn par_quicksort<T: Ord + Copy + Send>(v: &mut [T]) {
    if v.len() <= 1_000 {
        v.sort_unstable(); // small parts: not worth a parallel task
        return;
    }
    let (lt, gt) = partition3(v);
    let (left, rest) = v.split_at_mut(lt);
    let right = &mut rest[gt - lt..]; // skip the elements equal to the pivot
    rayon::join(|| par_quicksort(left), || par_quicksort(right));
}

/// Three-way partition around the middle element. Returns `(lt, gt)`:
/// `v[..lt]` < pivot, `v[lt..gt]` == pivot, `v[gt..]` > pivot.
fn partition3<T: Ord + Copy>(v: &mut [T]) -> (usize, usize) {
    let pivot = v[v.len() / 2];
    let (mut lt, mut i, mut gt) = (0, 0, v.len());
    while i < gt {
        match v[i].cmp(&pivot) {
            Ordering::Less => {
                v.swap(lt, i);
                lt += 1;
                i += 1;
            }
            Ordering::Greater => {
                gt -= 1;
                v.swap(i, gt);
            }
            Ordering::Equal => i += 1,
        }
    }
    (lt, gt)
}

fn main() {
    let mut v: Vec<u32> = (0..5000).rev().collect();
    par_quicksort(&mut v);
    println!("first: {:?}, last: {:?}", &v[..3], &v[4997..]);
    println!("sorted: {}", v.is_sorted());
}

#[cfg(test)]
mod tests {
    use super::*;

    /// A tiny pseudo-random number generator, so we need no extra crate.
    fn pseudo_random(len: usize, seed: u64) -> Vec<u64> {
        let mut x = seed;
        (0..len)
            .map(|_| {
                x = x
                    .wrapping_mul(6364136223846793005)
                    .wrapping_add(1442695040888963407);
                x >> 33
            })
            .collect()
    }

    #[test]
    fn sorts_like_the_standard_library() {
        for (len, seed) in [(0, 1), (1, 2), (999, 3), (100_000, 4), (1_000_000, 5)] {
            let mut ours = pseudo_random(len, seed);
            let mut expected = ours.clone();
            par_quicksort(&mut ours);
            expected.sort();
            assert_eq!(ours, expected, "len = {len}");
        }
    }

    #[test]
    fn handles_many_equal_elements() {
        let mut v = vec![7u8; 50_000];
        v.extend([3, 9, 1]);
        par_quicksort(&mut v);
        assert!(v.windows(2).all(|w| w[0] <= w[1]));
    }
}
```

```text
first: [0, 1, 2], last: [4997, 4998, 4999]
sorted: true
```

`cargo test --bin par_quicksort` показывает оба теста `ok`.

Почему это работает: `partition3` один раз проходит по срезу и поддерживает три области: `< pivot` в начале, `> pivot` в конце, `== pivot` посередине. Дальше сортируются только крайние области, поэтому одинаковые элементы сразу заканчивают рекурсию. `split_at_mut` доказывает компилятору, что два среза `&mut` не пересекаются, поэтому `rayon::join` может сортировать их в разных потоках. Порог в 1000 элементов держит задачи достаточно крупными, чтобы их стоило планировать (гранулярность задач). Генератор псевдослучайных чисел — *линейный конгруэнтный генератор*: умножить, прибавить, взять старшие биты.

</details>

## Вопросы для самопроверки

1. Почему мы собираем все `ScopedJoinHandle`, прежде чем вызывать для них `join`?

<details>
<summary>Ответ</summary>

Чтобы сначала запустились все потоки и работали одновременно. Если вызывать `join` сразу после запуска потока, мы ждём его завершения до запуска следующего, и код становится последовательным.

</details>

2. Почему для маленьких заданий пул потоков быстрее, чем новый поток на каждое задание?

<details>
<summary>Ответ</summary>

Запуск потока и ожидание его завершения стоят много микросекунд (около 18 мкс на тестовой машине). Пул запускает свои потоки один раз, а затем только передаёт задания через канал, что гораздо дешевле.

</details>

3. Что такое перехват работы и какую проблему он решает?

<details>
<summary>Ответ</summary>

У каждого рабочего потока своя очередь задач; простаивающий поток берёт задачи из очереди занятого. Это решает проблему дисбаланса нагрузки: когда задачи разной стоимости или ядра разной скорости, никто не простаивает, пока у других ещё есть работа.

</details>

4. Почему не стоит спать или ждать сеть внутри `par_iter().map(...)`?

<details>
<summary>Ответ</summary>

У rayon фиксированное число потоков, примерно по одному на ядро, предназначенных для работы на CPU. Заблокированное замыкание держит один из них в простое, и другие задачи не могут использовать это ядро. Ждать нужно в отдельных потоках или в асинхронном коде.

</details>

5. Что делают `fold` и `reduce` в конвейере rayon и почему это лучше одного общего `Mutex<HashMap>`?

<details>
<summary>Ответ</summary>

`fold` позволяет каждому потоку строить свой частичный результат без всяких блокировок; `reduce` объединяет частичные результаты в один. С общим `Mutex<HashMap>` каждый элемент брал бы одну и ту же блокировку, и потоки тратили бы время на ожидание друг друга.

</details>

6. На 12 ядрах параллельная версия в 6 раз быстрее последовательной. Что-то не так?

<details>
<summary>Ответ</summary>

Не обязательно. Часть работы последовательная, потоки нужно запускать и объединять, а E-ядра медленнее P-ядер. Ускорение меньше числа ядер — это нормально. Проверьте, что результат тот же, и сравните разные способы деления (например, равные куски против rayon), чтобы увидеть, не дисбаланс ли нагрузки виноват.

</details>

## Итоги

- `thread::scope` вместе с `chunks` делит работу по ядрам, заимствуя данные. Сначала запустите все потоки, потом вызывайте `join`.
- Запуск потока стоит около 18 мкс. Пул потоков запускает рабочие потоки один раз; с каналом MPMC (`crossbeam::channel`) каждый рабочий поток может владеть своим получателем, а уничтожение отправителя корректно завершает пул.
- Равные куски — не равная работа. Перехват работы позволяет простаивающим потокам брать задачи у занятых, поэтому все ядра заняты до конца.
- rayon даёт параллельные итераторы (`par_iter`, `into_par_iter`), `fold`/`reduce`, `par_sort` и `join` на глобальном пуле с перехватом работы.
- rayon — для работы, нагружающей CPU: замыкания должны быть `Send + Sync`, не должны блокировать, а задачи не должны быть слишком мелкими. `RAYON_NUM_THREADS` задаёт размер пула.
- Всегда проверяйте параллельный результат по последовательному эталону и измеряйте в релизном режиме.

## Источники и что почитать

- [rayon на docs.rs](https://docs.rs/rayon) — документация крейта: параллельные итераторы, `join`, `scope` и `ThreadPoolBuilder`.
- [rayon FAQ](https://github.com/rayon-rs/rayon/blob/main/FAQ.md) — ответы на частые вопросы, например сколько потоков использует rayon и почему.
- [crossbeam::channel на docs.rs](https://docs.rs/crossbeam/latest/crossbeam/channel/index.html) — ограниченные и неограниченные каналы MPMC и `select!`.
- [std::thread::scope](https://doc.rust-lang.org/std/thread/fn.scope.html) — документация scoped-потоков.
- [std::thread::available_parallelism](https://doc.rust-lang.org/std/thread/fn.available_parallelism.html) — что означает это число и когда оно может быть неверным.

---

[🏠 Начало](../START-HERE.md) · [← 14.3 Условные переменные](03-condition-variables.md) · [📋 Этап 14](README.md) · [14.5 Атомарные операции →](05-atomics.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
