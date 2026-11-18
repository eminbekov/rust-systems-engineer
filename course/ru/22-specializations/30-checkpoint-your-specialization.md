<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 22.29 Трек cloud native: проект и следующие шаги](29-cloud-native-track-project-and-next-steps.md) · [📋 Этап 22](README.md) · [Этап 23 — Итоговые проекты →](../23-capstone-projects/README.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/22-specializations/30-checkpoint-your-specialization.md) <!-- nav -->

# Урок 22.30 — Контрольная точка: ваша специализация

> **Этап 22 — Специализации** · Урок 30 из 30 · ⏱ около 3,5 часа (2 занятия)

## Что вы узнаете

Этот урок не учит ничему новому. Он проверяет, умеете ли вы применять то, чему научило ваше направление, и
показывает, куда вернуться, если что-то ещё слабо. Вы:

- Ответите на вопросы для всех и на вопросы **своего** направления (всего семь вопросов).
- Решите одну практическую задачу на код из своего направления, с тестами, и две короткие задачи, которые готовят
  следующий этап.
- Найдёте уроки, которые стоит повторить, с прямыми ссылками.

## Перед началом

- Вы прошли своё направление, включая его урок-проект: [22.5](05-os-track-project-and-next-steps.md) (ОС —
  операционные системы), [22.9](09-ebpf-track-project-and-next-steps.md) (eBPF — extended Berkeley Packet Filter,
  расширенный пакетный фильтр Беркли), [22.13](13-embedded-track-project-and-next-steps.md)
  (встраиваемые системы), [22.17](17-databases-track-project-and-next-steps.md) (базы данных),
  [22.21](21-networking-track-project-and-next-steps.md) (сети),
  [22.25](25-compilers-track-project-and-next-steps.md) (компиляторы) или
  [22.29](29-cloud-native-track-project-and-next-steps.md) (cloud native).
- Проходите тест **с закрытой книгой**: без уроков, без компилятора, без поиска. Сначала напишите свой ответ, потом
  откройте скрытый.
- Отвечайте только на вопросы своего направления. Читать вопросы других направлений можно — это дешёвый способ
  узнать, о чём они, — но знать ответы от вас не ждут.
- Для практических задач можно пользоваться компилятором и уроками. Создайте для них один пакет:
  `cargo new code/p22-checkpoint`.

## Новые слова

| Слово | Значение |
|---|---|
| **контрольная точка** (checkpoint) | Пауза в конце этапа, чтобы проверить себя, прежде чем идти дальше. |
| **с закрытой книгой** (closed book) | Ответ только по памяти, ничего не подглядывая. |
| **проект направления** (track project) | Проект последнего урока вашего направления: ядро, инструмент на eBPF, прошивка, движок запросов, прокси, интерпретатор или оператор. |

В тесте встречаются сокращения всех направлений. Каждое объясняется в уроках своего направления; вот короткое напоминание:

| Сокращение | Расшифровка | Направление |
|---|---|---|
| **QEMU** / **UART** / **MMIO** | Quick Emulator, быстрый эмулятор / Universal Asynchronous Receiver-Transmitter, универсальный асинхронный приёмопередатчик (последовательный порт) / ввод-вывод, отображённый в память | ОС |
| **BSS** / **MMU** / **GIC** | Block Started by Symbol (обнуляемые данные) / Memory Management Unit, блок управления памятью / Generic Interrupt Controller, общий контроллер прерываний | ОС |
| **XDP** | eXpress Data Path, самая ранняя точка обработки принятых пакетов | eBPF |
| **HAL** / **SPI** / **I2C** | Hardware Abstraction Layer, слой абстракции оборудования / Serial Peripheral Interface / Inter-Integrated Circuit (две распространённые шины между микросхемами) | встраиваемые системы |
| **SWD** / **RTT** | Serial Wire Debug (подключение отладчика к микросхеме) / Real-Time Transfer (логи через отладчик) | встраиваемые системы |
| **SQL** / **CSV** / **SIMD** | Structured Query Language, язык структурированных запросов / comma-separated values, значения через запятую / Single Instruction, Multiple Data, одна инструкция — много данных | базы данных |
| **TCP** / **HTTP** | Transmission Control Protocol, протокол управления передачей / HyperText Transfer Protocol, протокол передачи гипертекста | сети |
| **AST** | Abstract Syntax Tree, абстрактное синтаксическое дерево, которое строит парсер | компиляторы |
| **CRD** / **RBAC** | CustomResourceDefinition, определение пользовательского ресурса / Role-Based Access Control, управление доступом на основе ролей | cloud native |

## Вопросы для самопроверки

### Для всех

1. Три направления пишут код без стандартной библиотеки. Что убирает `#![no_std]` и что остаётся?

<details>
<summary>Ответ</summary>

Убирается `std`: всё, чему нужна операционная система, — файлы, потоки, сеть и по умолчанию выделяющие память
коллекции. Остаётся `core`: `Option`, `Result`, срезы, итераторы, `fmt` и целочисленные типы. `alloc` (`Vec`,
`String`, `Box`) доступен, только если вы предоставите глобальный аллокатор.

</details>

2. В каждом направлении часть кода оставалась «чистой» (без оборудования, ядра, сети или кластера). Зачем?

<details>
<summary>Ответ</summary>

Чистый код можно проверять модульными тестами на Mac за миллисекунды. Тогда часть, которая трогает реальный мир,
остаётся маленькой и покрывается меньшим числом более медленных тестов (QEMU, привилегированный контейнер, плата,
кластер kind).

</details>

3. Что такое target triple (тройка целевой платформы) и какую не-маковскую тройку использовало ваше направление?

<details>
<summary>Ответ</summary>

Имя платформы, для которой собирает компилятор: архитектура CPU (Central Processing Unit, центральный процессор),
производитель, операционная система (а иногда и двоичный интерфейс). Ваш Mac — `aarch64-apple-darwin`. Примеры:
`aarch64-unknown-none-softfloat` (ОС), `thumbv8m.main-none-eabihf` (Pico 2), `bpfel-unknown-none` (eBPF), цель
`wasm32` (компиляторы) и Linux-тройка `aarch64-unknown-linux-gnu` в ВМ `dev` (eBPF, сети) и в образе контейнера
оператора (cloud native).

</details>

### Направление ОС

1. Какие три вещи должна предоставить автономная программа (freestanding binary) вместо того, что раньше давала `std`?

<details>
<summary>Ответ</summary>

Обработчик паники (`#[panic_handler]`), собственную точку входа (с `#![no_main]`, потому что нет рантайма, который
вызвал бы `main`) и стратегию паники без раскрутки стека (`panic = "abort"`).

</details>

2. Почему регистры UART читают и записывают volatile-обращениями?

<details>
<summary>Ответ</summary>

Это MMIO: регистры устройства по адресам памяти. Запись имеет эффект за пределами программы, поэтому компилятор не
должен удалять, объединять или переставлять такие обращения, как он может делать с обычной памятью. Volatile-обращения
это запрещают.

</details>

3. Что на arm64 происходит между «пришло прерывание таймера» и «работает ваш обработчик»?

<details>
<summary>Ответ</summary>

CPU останавливает текущий код, сохраняет адрес возврата и состояние процессора в системных регистрах и переходит к
записи таблицы векторов, которая соответствует виду и уровню исключения. Ваш обработчик сохраняет остальные регистры,
спрашивает у GIC номер прерывания, обрабатывает и подтверждает его, восстанавливает регистры и возвращается через
`eret`.

</details>

4. Что такое тождественное отображение (identity mapping) и зачем оно при включении MMU?

<details>
<summary>Ответ</summary>

Отображение, при котором виртуальный адрес равен физическому. Код, который включает MMU, после переключения продолжает
работать по тем же адресам, поэтому следующая инструкция всё ещё находится.

</details>

### Направление eBPF

1. Что проверяет верификатор eBPF и зачем вашим программам были нужны явные проверки границ?

<details>
<summary>Ответ</summary>

Прежде чем ядро загрузит программу, верификатор доказывает её безопасность: каждое обращение к памяти в границах,
программа завершается, вызывает только разрешённые помощники. Если верификатор не может доказать, что чтение пакета в
границах, он отклоняет программу, поэтому перед каждым чтением вы проверяете `start + size <= end`.

</details>

2. Чем kprobe отличается от tracepoint?

<details>
<summary>Ответ</summary>

Kprobe цепляется почти к любой функции ядра; это гибко, но зависит от внутренностей ядра, которые меняются между
версиями. Tracepoint — фиксированная точка, которую разработчики ядра поставили намеренно, с документированным
стабильным форматом.

</details>

3. Для чего нужны карты (maps) eBPF?

<details>
<summary>Ответ</summary>

Это общая память eBPF: таблицы «ключ — значение», массивы и кольцевые буферы, которые могут читать и писать и
программа в ядре, и ваша программа в пространстве пользователя (через Aya), например счётчики по процессам или
события для вывода.

</details>

4. Когда выполняется программа XDP и какие решения она может вернуть?

<details>
<summary>Ответ</summary>

В сетевом драйвере, для каждого принятого пакета, до того, как сетевой стек ядра начнёт с ним работать. Она
возвращает действие: `XDP_PASS` (продолжить как обычно), `XDP_DROP`, `XDP_TX` (отправить обратно через тот же
интерфейс), `XDP_REDIRECT` или `XDP_ABORTED` (ошибка).

</details>

### Направление встраиваемых систем

1. Чего нет у микроконтроллера по сравнению с вашим Mac и что это значит для программы на Rust?

<details>
<summary>Ответ</summary>

Нет операционной системы, очень мало RAM (random-access memory, оперативная память), обычно нет MMU и файловой
системы. Программа — `no_std` и `no_main`, крейт рантайма даёт точку входа после сброса и таблицу векторов, а
обработчик паники предоставляете вы.

</details>

2. Что дают трейты `embedded-hal`?

<details>
<summary>Ответ</summary>

Общие интерфейсы, такие как `OutputPin`, `InputPin`, `DelayNs`, и шины вроде SPI и I2C. Драйвер, написанный под эти
трейты, работает с HAL любой микросхемы, и его можно тестировать на Mac с поддельными выводами.

</details>

3. Зачем кнопке подавление дребезга и что значит «активный низкий уровень» (active low)?

<details>
<summary>Ответ</summary>

Металлические контакты несколько миллисекунд дребезжат, поэтому одно нажатие выглядит как несколько быстрых
изменений. Активный низкий уровень означает, что при нажатии вывод читается как *low*, потому что в остальное время
подтягивающий резистор держит его в high.

</details>

4. Что даёт Embassy по сравнению с «суперциклом» (super loop)?

<details>
<summary>Ответ</summary>

Несколько асинхронных задач могут ждать одновременно (таймер, фронт сигнала на выводе, канал) без активного ожидания.
Исполнитель усыпляет CPU, пока прерывание не разбудит задачу, которая может продвинуться.

</details>

### Направление баз данных

1. Чем логический план отличается от физического?

<details>
<summary>Ответ</summary>

Логический план говорит, *что* вычислить, в виде дерева реляционных операторов (сканирование, фильтр, проекция,
агрегация). Физический план говорит, *как*: какими алгоритмами (например, хеш-агрегацией), в каком порядке, какими
пакетами (batches).

</details>

2. Модель Volcano или векторизованное выполнение: в чём разница?

<details>
<summary>Ответ</summary>

В модели Volcano у каждого оператора есть `next()`, и он вытягивает у потомка по одной строке. При векторизованном
выполнении операторы передают пакеты столбцов (record batches Arrow), поэтому вызовов гораздо меньше, а CPU работает
с длинными плотными массивами.

</details>

3. Почему аналитический движок хранит данные по столбцам (Apache Arrow)?

<details>
<summary>Ответ</summary>

Аналитические запросы читают несколько столбцов по многим строкам. Когда столбцы хранятся вместе, движок читает
только нужные столбцы, значения лежат в памяти рядом (хорошо для кэшей и SIMD) и хорошо сжимаются.

</details>

4. Как хеш-агрегация вычисляет `GROUP BY city` с `COUNT(*)`?

<details>
<summary>Ответ</summary>

Она держит хеш-таблицу от ключа группы (`city`) к аккумулятору (счётчику). Один проход по входу обновляет аккумулятор
ключа каждой строки; в конце каждая запись таблицы становится одной строкой результата.

</details>

### Направление сетей

1. Что позволяет `SO_REUSEPORT` и как это приводит к серверу «поток на ядро» (thread-per-core)?

<details>
<summary>Ответ</summary>

Несколько сокетов могут слушать один и тот же адрес и порт, и Linux распределяет между ними новые соединения. Тогда
каждый поток владеет своим слушающим сокетом и своим рантаймом, и потоки почти ничего не делят.

</details>

2. Что означает «без копирования» (zero-copy), когда вы разбираете данные с `Bytes` и `BytesMut`?

<details>
<summary>Ответ</summary>

Разобранные части (метод, путь, значения заголовков) — это представления (views) буфера, прочитанного из сокета,
разделяемые через счётчик ссылок. Ни один байт не копируется в новые строки.

</details>

3. Почему парсер протокола должен обрабатывать неполный ввод?

<details>
<summary>Ответ</summary>

TCP доставляет поток байтов, а не сообщения. Одно чтение может закончиться посреди запроса, поэтому парсер должен
сказать «неполно, прочитай ещё», а не падать.

</details>

4. Ваш прокси на один запрос отвечает `502`, а на другой — `504`. Что произошло в каждом случае?

<details>
<summary>Ответ</summary>

`502 Bad Gateway`: вышестоящим сервером (upstream) не удалось воспользоваться или он дал некорректный ответ
(например, соединение отклонено). `504 Gateway Timeout`: вышестоящий сервер не ответил вовремя.

</details>

### Направление компиляторов

1. В чём работа лексера и в чём работа парсера?

<details>
<summary>Ответ</summary>

Лексер превращает символы в токены (числа, имена, операторы, ключевые слова). Парсер превращает токены в
синтаксическое дерево (AST) по грамматике.

</details>

2. Что решают приоритет и ассоциативность и как Pratt-парсинг справляется с обоими?

<details>
<summary>Ответ</summary>

Приоритет решает, что `1 + 2 * 3` — это `1 + (2 * 3)`; ассоциативность — что `10 - 4 - 3` — это `(10 - 4) - 3`.
Pratt-парсинг даёт каждому оператору силу связывания (binding power) и продолжает читать операторы, только пока их
сила больше текущего минимума; при равной силе левоассоциативные операторы останавливаются.

</details>

3. Что такое окружение (environment) в вашем интерпретаторе и как вложенные области видимости находят переменную?

<details>
<summary>Ответ</summary>

Место, где живут переменные: стек таблиц от имён к значениям, по одной таблице на область видимости. Новый блок
кладёт таблицу на стек и снимает её в конце; поиск начинается с самой внутренней таблицы и идёт наружу, поэтому
внутренний `let` затеняет внешний, пока блок не закончится.

</details>

4. Что даёт WebAssembly и как вы запускали модуль вне браузера?

<details>
<summary>Ответ</summary>

Переносимый изолированный (sandboxed) двоичный формат, который работает почти со скоростью нативного кода на любом
хосте с рантаймом. Вы добавили цели командой `rustup target add wasm32-wasip1 wasm32-unknown-unknown`, собирали
`cargo build --target ...` и запускали модуль утилитой командной строки `wasmtime` (или из своей программы через крейт
`wasmtime`).

</details>

### Направление cloud native

1. Почему функция согласования должна срабатывать по уровню и быть идемпотентной?

<details>
<summary>Ответ</summary>

События можно пропустить (перезапуски, обрывы сети), поэтому согласование должно работать только от текущего
состояния. Оно много раз выполняется для одного объекта, в том числе после собственных записей, поэтому запуск, которому
нечего исправлять, не должен ничего менять.

</details>

2. Чем отличаются CRD, пользовательский ресурс и оператор?

<details>
<summary>Ответ</summary>

CRD добавляет в API тип с его схемой; пользовательский ресурс — один объект этого типа; оператор — это CRD плюс
контроллер, который согласует его объекты.

</details>

3. Когда использовать ссылку на владельца, а когда финализатор?

<details>
<summary>Ответ</summary>

Ссылку на владельца — для дочерних объектов в том же пространстве имён: сборщик мусора удаляет их вместе с владельцем.
Финализатор — для всего, до чего ссылка на владельца не дотянется (другие пространства имён, внешние системы): он
откладывает удаление, пока не выполнится ваша очистка.

</details>

4. Оператор работает через `cargo run`, а в кластере пишет в лог `forbidden`. Почему и как найти исправление?

<details>
<summary>Ответ</summary>

На Mac он использовал ваш kubeconfig администратора; в кластере он работает от имени своего ServiceAccount только с
правами своей роли RBAC. Сообщение называет недостающий глагол и ресурс; проверьте через
`kubectl auth can-i <verb> <resource> --as=system:serviceaccount:<ns>:<name>` и добавьте ровно этот глагол.

</details>

## Практические задания

### Задание 1 — Ваше направление в одной небольшой функции

Решите задачу **своего** направления в `code/p22-checkpoint`. Каждая — обычный Rust, который запускается и
тестируется на Mac: маленькое чистое ядро того, что ваше направление делает по-настоящему. Положите её в
`src/bin/<name>.rs`, запускайте через `cargo run --bin <name>` и тестируйте через `cargo test --bin <name>`.
Напишите хотя бы один тест.

**Направление ОС — `os_hex`.** Ядро не может пользоваться `format!`, пока у него нет кучи. Напишите
`fn write_hex(n: u64, buf: &mut [u8; 18]) -> &str`, которая записывает в `buf` `0x` и ровно 16 строчных
шестнадцатеричных цифр и возвращает их как `&str`, используя только `core`. `main` печатает `0xdead_beef` и `u64::MAX`.

**Направление eBPF — `ebpf_xdp`.** Запишите решение межсетевого экрана на XDP обычной функцией:
`fn decide(packet: &[u8], blocked: &[[u8; 4]]) -> Action` (`Pass` или `Drop`). Отбрасывайте пакет IPv4 (тип Ethernet
`0x0800` в байтах 12–13), адрес источника которого (байты 26–29: 14 байт заголовка Ethernet плюс смещение 12 в
заголовке IPv4) заблокирован. Все остальные пакеты, включая слишком короткие для обоих заголовков, проходят.

**Направление встраиваемых систем — `embedded_debounce`.** Напишите `Debouncer` с
`fn sample(&mut self, pressed: bool) -> Option<bool>`, который сообщает новое состояние кнопки только после `needed`
одинаковых отсчётов подряд, а иначе возвращает `None`.

**Направление баз данных — `db_count_by`.** Напишите
`fn count_by(csv: &str, column: &str) -> Result<BTreeMap<String, u64>, String>` — движок запроса
`SELECT <column>, COUNT(*) … GROUP BY <column>` по тексту CSV, первая строка которого — заголовок. Неизвестные
столбцы и короткие строки — ошибки.

**Направление сетей — `net_request_line`.** Напишите парсер строки HTTP-запроса без копирования
(`GET /index.html HTTP/1.1\r\n`), который возвращает `Complete { method, path, consumed }` со срезами входа,
`Incomplete`, когда `\r\n` ещё нет, или `Invalid`.

**Направление компиляторов — `compiler_pratt`.** Напишите `fn eval(src: &str) -> Result<i64, String>` для целых чисел
с `+ - * /` и скобками, на Pratt-парсинге (силы связывания). `10 - 4 - 3` должно давать `3`; деление на ноль — ошибка.

**Направление cloud native — `cloud_status`.** Напишите `fn status_for(desired: usize, pods: &[Pod]) -> Status`, где у
`Pod` есть флаги `ready` и `terminating`. Считаются только готовые и не завершающиеся поды; фаза — `Ready`, когда их
число достигает `desired`, иначе `Pending`.

<details>
<summary>Подсказка</summary>

- ОС: полубайт (nibble) — это 4 бита; цифра для позиции `i` (0 — самая левая) — это `(n >> ((15 - i) * 4)) & 0xf`.
  Берите её по индексу из `b"0123456789abcdef"`. `core::str::from_utf8` превращает буфер обратно в `&str`.
- eBPF: **сначала** проверьте длину, как заставляет верификатор; `u16::from_be_bytes` читает тип в сетевом порядке байтов.
- Встраиваемые системы: храните стабильное состояние, состояние-кандидат и счётчик. Отсчёт, равный стабильному
  состоянию, сбрасывает счётчик.
- Базы данных: `header.split(',').position(...)` находит индекс столбца; `entry(...).or_insert(0)` считает.
- Сети: `buf.windows(2).position(|w| w == b"\r\n")` находит конец; `split(|&b| b == b' ')` на срезе возвращает
  подсрезы, а не копии.
- Компиляторы: `expr(min_bp)` разбирает атом, затем крутится, пока сила следующего оператора **больше** `min_bp`,
  вызывая `expr(power)` для правой части.
- Cloud native: `pods.iter().filter(...).count()`, и помните, что при нуле желаемых реплик это уже `Ready`.

</details>

<details>
<summary>Ответ — направление ОС</summary>

```rust
/// Writes `n` as `0x` and 16 hexadecimal digits into `buf`, without any allocation.
fn write_hex(n: u64, buf: &mut [u8; 18]) -> &str {
    const DIGITS: &[u8; 16] = b"0123456789abcdef";
    buf[0] = b'0';
    buf[1] = b'x';
    for i in 0..16 {
        let shift = (15 - i) * 4;
        let nibble = ((n >> shift) & 0xf) as usize;
        buf[2 + i] = DIGITS[nibble];
    }
    core::str::from_utf8(buf).expect("only ASCII digits were written")
}

fn main() {
    let mut buf = [0u8; 18];
    println!("{}", write_hex(0xdead_beef, &mut buf));
    println!("{}", write_hex(u64::MAX, &mut buf));
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn digits_are_in_the_right_order() {
        let mut buf = [0u8; 18];
        assert_eq!(write_hex(0x1234_abcd, &mut buf), "0x000000001234abcd");
    }
}
```

```text
0x00000000deadbeef
0xffffffffffffffff
```

Функция использует только `core`, поэтому без изменений работает внутри вашего ядра, где `buf` может лежать на стеке.

</details>

<details>
<summary>Ответ — направление eBPF</summary>

```rust
#[derive(Debug, PartialEq)]
enum Action {
    Pass,
    Drop,
}

const ETH_HEADER: usize = 14;
const ETHERTYPE_IPV4: u16 = 0x0800;

/// Decides like an XDP program: drop IPv4 packets whose source address is blocked.
fn decide(packet: &[u8], blocked: &[[u8; 4]]) -> Action {
    // Every read is checked first, exactly what the eBPF verifier demands.
    if packet.len() < ETH_HEADER + 20 {
        return Action::Pass;
    }
    let ethertype = u16::from_be_bytes([packet[12], packet[13]]);
    if ethertype != ETHERTYPE_IPV4 {
        return Action::Pass;
    }
    let start = ETH_HEADER + 12;
    let source = [
        packet[start],
        packet[start + 1],
        packet[start + 2],
        packet[start + 3],
    ];
    if blocked.contains(&source) {
        Action::Drop
    } else {
        Action::Pass
    }
}

/// Builds a minimal Ethernet + IPv4 header with the given source address.
fn packet_from(source: [u8; 4]) -> Vec<u8> {
    let mut p = vec![0u8; ETH_HEADER + 20];
    p[12] = 0x08; // ethertype 0x0800 = IPv4
    p[13] = 0x00;
    p[ETH_HEADER + 12..ETH_HEADER + 16].copy_from_slice(&source);
    p
}

fn main() {
    let blocked = [[10, 0, 0, 66]];
    println!("{:?}", decide(&packet_from([10, 0, 0, 66]), &blocked));
    println!("{:?}", decide(&packet_from([10, 0, 0, 7]), &blocked));
    println!("{:?}", decide(&[0u8; 10], &blocked));
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn short_or_non_ipv4_packets_pass() {
        assert_eq!(decide(&[0u8; 20], &[[0, 0, 0, 0]]), Action::Pass);
        let mut ipv6 = packet_from([1, 2, 3, 4]);
        ipv6[12] = 0x86;
        ipv6[13] = 0xdd;
        assert_eq!(decide(&ipv6, &[[1, 2, 3, 4]]), Action::Pass);
    }
}
```

```text
Drop
Pass
Pass
```

В Aya та же логика читает через `ctx.data()` и `ctx.data_end()` с той же проверкой границ и возвращает
`xdp_action::XDP_DROP` или `XDP_PASS`.

</details>

<details>
<summary>Ответ — направление встраиваемых систем</summary>

```rust
/// Reports a new button state only after `needed` equal samples in a row.
struct Debouncer {
    stable: bool,
    candidate: bool,
    count: u8,
    needed: u8,
}

impl Debouncer {
    fn new(needed: u8) -> Self {
        Debouncer {
            stable: false,
            candidate: false,
            count: 0,
            needed,
        }
    }

    /// Feeds one sample (`true` = pressed). Returns the new state when it changes.
    fn sample(&mut self, pressed: bool) -> Option<bool> {
        if pressed == self.stable {
            self.count = 0;
            return None;
        }
        if pressed != self.candidate {
            self.candidate = pressed;
            self.count = 0;
        }
        self.count += 1;
        if self.count >= self.needed {
            self.stable = pressed;
            self.count = 0;
            Some(pressed)
        } else {
            None
        }
    }
}

fn main() {
    let mut d = Debouncer::new(3);
    let samples = [true, false, true, true, true, true, false, false, false];
    let events: Vec<Option<bool>> = samples.iter().map(|&s| d.sample(s)).collect();
    println!("{events:?}");
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn steady_press_and_release_are_reported_once() {
        let mut d = Debouncer::new(2);
        assert_eq!(d.sample(true), None);
        assert_eq!(d.sample(true), Some(true));
        assert_eq!(d.sample(true), None);
        assert_eq!(d.sample(false), None);
        assert_eq!(d.sample(false), Some(false));
    }
}
```

```text
[None, None, None, None, Some(true), None, None, None, Some(false)]
```

На плате задача Embassy вызывает `sample` каждые несколько миллисекунд с уровнем вывода (инвертированным для кнопки
с активным низким уровнем) и реагирует на `Some(...)`.

</details>

<details>
<summary>Ответ — направление баз данных</summary>

```rust
use std::collections::BTreeMap;

/// `SELECT <column>, COUNT(*) FROM csv GROUP BY <column>`: scan, project, aggregate.
fn count_by(csv: &str, column: &str) -> Result<BTreeMap<String, u64>, String> {
    let mut lines = csv.lines();
    let header = lines.next().ok_or("empty input")?;
    let index = header
        .split(',')
        .position(|name| name == column)
        .ok_or(format!("no column {column}"))?;
    let mut counts = BTreeMap::new();
    for (number, line) in lines.enumerate() {
        let value = line
            .split(',')
            .nth(index)
            .ok_or(format!("row {} is too short", number + 1))?;
        *counts.entry(value.to_string()).or_insert(0) += 1;
    }
    Ok(counts)
}

fn main() {
    let csv = "name,city\nAna,Lisbon\nBen,Oslo\nCai,Lisbon\n";
    println!("{:?}", count_by(csv, "city"));
    println!("{:?}", count_by(csv, "country"));
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn counts_each_group() {
        let csv = "a,b\n1,x\n2,y\n3,x\n";
        let counts = count_by(csv, "b").unwrap();
        assert_eq!(counts["x"], 2);
        assert_eq!(counts["y"], 1);
    }
}
```

```text
Ok({"Lisbon": 2, "Oslo": 1})
Err("no column country")
```

Это сканирование, проекция (один столбец) и хеш-агрегация со счётчиком-аккумулятором — всё за один проход. `BTreeMap`
держит результат отсортированным, поэтому тесты стабильны.

</details>

<details>
<summary>Ответ — направление сетей</summary>

```rust
#[derive(Debug, PartialEq)]
enum Parse<'a> {
    /// A whole request line; `consumed` bytes (including CRLF) can be dropped from the buffer.
    Complete {
        method: &'a [u8],
        path: &'a [u8],
        consumed: usize,
    },
    /// No CRLF yet: read more bytes and try again.
    Incomplete,
    Invalid,
}

/// Parses `METHOD SP PATH SP VERSION CRLF` without copying any bytes.
fn parse_request_line(buf: &[u8]) -> Parse<'_> {
    let Some(end) = buf.windows(2).position(|w| w == b"\r\n") else {
        return Parse::Incomplete;
    };
    let line = &buf[..end];
    let mut parts = line.split(|&b| b == b' ');
    match (parts.next(), parts.next(), parts.next(), parts.next()) {
        (Some(method), Some(path), Some(version), None)
            if !method.is_empty() && path.starts_with(b"/") && version.starts_with(b"HTTP/") =>
        {
            Parse::Complete {
                method,
                path,
                consumed: end + 2,
            }
        }
        _ => Parse::Invalid,
    }
}

fn main() {
    let buf = b"GET /index.html HTTP/1.1\r\nHost: a\r\n";
    if let Parse::Complete {
        method,
        path,
        consumed,
    } = parse_request_line(buf)
    {
        let method = String::from_utf8_lossy(method);
        let path = String::from_utf8_lossy(path);
        println!("method={method} path={path} consumed={consumed}");
    }
    println!("{:?}", parse_request_line(b"GET /index.ht"));
    println!("{:?}", parse_request_line(b"HELLO\r\n"));
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn slices_point_into_the_buffer() {
        let buf = b"POST /api HTTP/1.1\r\n";
        let Parse::Complete {
            method,
            path,
            consumed,
        } = parse_request_line(buf)
        else {
            panic!("expected a complete line");
        };
        assert_eq!(method, b"POST");
        assert_eq!(path, b"/api");
        assert_eq!(consumed, buf.len());
        assert_eq!(path.as_ptr(), buf[5..].as_ptr()); // no copy was made
    }
}
```

```text
method=GET path=/index.html consumed=26
Incomplete
Invalid
```

Тест `slices_point_into_the_buffer` доказывает отсутствие копирования: срез `path` начинается ровно по адресу байта 5
входа. С `BytesMut` вы бы вызвали `split_to(consumed)`, чтобы отрезать строку от буфера.

</details>

<details>
<summary>Ответ — направление компиляторов</summary>

```rust
/// Evaluates integer expressions with + - * / and parentheses, using Pratt parsing.
fn eval(src: &str) -> Result<i64, String> {
    let tokens: Vec<char> = src.chars().filter(|c| !c.is_whitespace()).collect();
    let mut pos = 0;
    let value = expr(&tokens, &mut pos, 0)?;
    if pos != tokens.len() {
        return Err(format!("unexpected '{}'", tokens[pos]));
    }
    Ok(value)
}

/// Binding power of an infix operator: higher binds tighter.
fn binding_power(op: char) -> Option<u8> {
    match op {
        '+' | '-' => Some(1),
        '*' | '/' => Some(2),
        _ => None,
    }
}

fn expr(t: &[char], pos: &mut usize, min_bp: u8) -> Result<i64, String> {
    let mut left = atom(t, pos)?;
    while let Some(&op) = t.get(*pos) {
        let Some(bp) = binding_power(op) else { break };
        if bp <= min_bp {
            break; // left-associative: equal power stops here
        }
        *pos += 1;
        let right = expr(t, pos, bp)?;
        left = match op {
            '+' => left + right,
            '-' => left - right,
            '*' => left * right,
            _ if right == 0 => return Err("division by zero".to_string()),
            _ => left / right,
        };
    }
    Ok(left)
}

fn atom(t: &[char], pos: &mut usize) -> Result<i64, String> {
    match t.get(*pos) {
        Some('(') => {
            *pos += 1;
            let v = expr(t, pos, 0)?;
            if t.get(*pos) != Some(&')') {
                return Err("expected ')'".to_string());
            }
            *pos += 1;
            Ok(v)
        }
        Some(c) if c.is_ascii_digit() => {
            let mut n: i64 = 0;
            while let Some(d) = t.get(*pos).and_then(|c| c.to_digit(10)) {
                n = n * 10 + i64::from(d);
                *pos += 1;
            }
            Ok(n)
        }
        Some(c) => Err(format!("unexpected '{c}'")),
        None => Err("unexpected end".to_string()),
    }
}

fn main() {
    for src in ["1 + 2 * 3", "(1 + 2) * 3", "10 - 4 - 3", "8 / (2 - 2)"] {
        println!("{src} => {:?}", eval(src));
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn precedence_and_associativity() {
        assert_eq!(eval("1 + 2 * 3"), Ok(7));
        assert_eq!(eval("10 - 4 - 3"), Ok(3));
        assert_eq!(eval("2 * (3 + 4)"), Ok(14));
    }
}
```

```text
1 + 2 * 3 => Ok(7)
(1 + 2) * 3 => Ok(9)
10 - 4 - 3 => Ok(3)
8 / (2 - 2) => Err("division by zero")
```

`bp <= min_bp` делает `-` левоассоциативным: после `10 - 4` второй `-` имеет ту же силу и останавливает внутренний
вызов, поэтому внешний цикл вычитает `3` из `6`.

</details>

<details>
<summary>Ответ — направление cloud native</summary>

```rust
#[derive(Debug, Clone, Copy)]
struct Pod {
    ready: bool,
    terminating: bool,
}

#[derive(Debug, PartialEq)]
struct Status {
    ready_replicas: usize,
    phase: &'static str,
}

/// Computes the status an operator reports for `desired` replicas and the pods it sees.
fn status_for(desired: usize, pods: &[Pod]) -> Status {
    let ready = pods.iter().filter(|p| p.ready && !p.terminating).count();
    let phase = if ready >= desired { "Ready" } else { "Pending" };
    Status {
        ready_replicas: ready,
        phase,
    }
}

fn main() {
    let ok = Pod {
        ready: true,
        terminating: false,
    };
    let starting = Pod {
        ready: false,
        terminating: false,
    };
    let leaving = Pod {
        ready: true,
        terminating: true,
    };
    println!("{:?}", status_for(2, &[ok, starting]));
    println!("{:?}", status_for(2, &[ok, ok, leaving]));
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn terminating_pods_do_not_count() {
        let leaving = Pod {
            ready: true,
            terminating: true,
        };
        assert_eq!(status_for(1, &[leaving]).ready_replicas, 0);
        assert_eq!(status_for(1, &[leaving]).phase, "Pending");
    }
}
```

```text
Status { ready_replicas: 1, phase: "Pending" }
Status { ready_replicas: 2, phase: "Ready" }
```

Это чистая часть логики статуса оператора; в проекте StaticSite число бралось из `readyReplicas` у Deployment, а не из
списка подов.

</details>

### Задание 2 — Объясните свой проект за пять минут (без нового кода)

Объясните вслух свой проект направления, как коллеге, который проходил другое направление: что он делает, два самых
важных проектных решения, одна ошибка, на которую вы наткнулись, и как вы её нашли, и как он тестируется. Запишите
себя на видео или изложите это первым разделом README проекта.

<details>
<summary>Ответ</summary>

Хорошее объяснение начинается с одного предложения о задаче, показывает основной поток данных (помогает маленькая
схема), называет решения вместе с их компромиссом («мы копируем хеш HTML в шаблон пода, потому что…») и заканчивается
тем, как рецензент может запустить тесты. Если понадобилось больше пяти минут, режьте детали, а не решения.

</details>

### Задание 3 — Спланируйте следующий месяц в направлении (без нового кода)

Выберите один open-source-проект из «следующих шагов» последнего урока своего направления. Прочитайте его README и
руководство для контрибьюторов, соберите его, запустите тесты и найдите три задачи с меткой для новичков. Пока ничего
не комментируйте и не открывайте; напишите план: с какой задачи вы бы начали и почему.

<details>
<summary>Ответ</summary>

Полезный план называет проект и точные команды сборки и тестов, которые у вас сработали, своими словами объясняет одну
задачу, перечисляет файлы, которые вы бы изменили, и говорит, как вы проверите изменение. Уроки 24.4 и 24.5 показывают,
как превратить такой план в первый вклад.

</details>

## Если что-то не получилось

| Тема | Урок для повторения |
|---|---|
| Выбор направления и работа по нему | [22.1 — Выбор специализации](01-choosing-a-specialization.md) |
| `no_std`, обработчик паники, точка входа | [22.2 — Трек ОС: программа на Rust без операционной системы](02-os-track-a-freestanding-binary.md) |
| QEMU, скрипт компоновщика, UART, MMIO | [22.3 — Трек ОС: загрузка ядра в QEMU](03-os-track-booting-a-kernel-in-qemu.md) |
| Исключения, таблица векторов, GIC, MMU | [22.4 — Трек ОС: прерывания и страничная память](04-os-track-interrupts-and-paging.md) |
| Что такое eBPF, верификатор | [22.6 — Трек eBPF: что такое eBPF](06-ebpf-track-what-ebpf-is.md) |
| Aya, bpf-linker, первая программа на tracepoint | [22.7 — Трек eBPF: первая программа на Aya](07-ebpf-track-a-first-program-with-aya.md) |
| Карты, kprobe, поля tracepoint | [22.8 — Трек eBPF: карты (maps) и трассировка](08-ebpf-track-maps-and-tracing.md) |
| XDP, проверки границ пакета | [22.9 — Трек eBPF: проект и следующие шаги](09-ebpf-track-project-and-next-steps.md) |
| Микроконтроллеры, `no_std`, прошивка | [22.10 — Трек встраиваемых систем: микроконтроллеры и no_std](10-embedded-track-microcontrollers-and-no-std.md) |
| GPIO, `embedded-hal`, подавление дребезга | [22.11 — Трек встраиваемых систем: GPIO и embedded-hal](11-embedded-track-gpio-and-embedded-hal.md) |
| Embassy, задачи, `defmt` | [22.12 — Трек встраиваемых систем: асинхронность с Embassy](12-embedded-track-async-with-embassy.md) |
| Планы, модель Volcano и векторизованное выполнение, Arrow | [22.14 — Трек баз данных: как работают движки запросов](14-databases-track-how-query-engines-work.md) |
| Токены, синтаксические деревья SQL, sqlparser | [22.15 — Трек баз данных: разбор SQL](15-databases-track-parsing-sql.md) |
| Фильтр, проекция, хеш-агрегация | [22.16 — Трек баз данных: выполнение запросов](16-databases-track-executing-queries.md) |
| Опции сокетов, `SO_REUSEPORT`, поток на ядро | [22.18 — Трек сетей: высокопроизводительные серверы](18-networking-track-high-performance-servers.md) |
| `Bytes`, неполный ввод, без копирования | [22.19 — Трек сетей: разбор протоколов без копирования](19-networking-track-zero-copy-parsing.md) |
| Pingora, upstream, коды статуса | [22.20 — Трек сетей: прокси для продакшена](20-networking-track-production-proxies.md) |
| Лексер, парсер, Pratt-парсинг | [22.22 — Трек компиляторов: лексический и синтаксический анализ](22-compilers-track-lexing-and-parsing.md) |
| Вычисление, окружения, области видимости | [22.23 — Трек компиляторов: интерпретатор обхода дерева](23-compilers-track-an-interpreter.md) |
| Цели `wasm32`, wasmtime | [22.24 — Трек компиляторов: WebAssembly и Rust](24-compilers-track-webassembly.md) |
| Циклы управления, согласование, CRD | [22.26 — Трек cloud native: операторы Kubernetes](26-cloud-native-track-kubernetes-operators.md) |
| kube-rs, server-side apply, ссылки на владельца | [22.27 — Трек cloud native: оператор на kube-rs](27-cloud-native-track-an-operator-with-kube-rs.md) |
| Финализаторы, e2e-тесты, RBAC | [22.28 — Трек cloud native: тестирование и запуск операторов](28-cloud-native-track-testing-and-running-operators.md) |
| Ваш проект направления | урок-проект вашего направления (см. «Перед началом») |

## Что дальше

Этап 23 — итоговый проект: один большой проект на ваш выбор, с проектным документом, тестами, бенчмарками и
укреплением. Ваше направление — хорошая подсказка для выбора: сети естественно ведут к обратному HTTP-прокси, базы
данных и распределённые системы — к хранилищу «ключ — значение», cloud native — к конвейеру наблюдаемости
(развёрнутому вашим собственным оператором), а направления ОС, eBPF, встраиваемых систем и компиляторов — к
«собственной идее», например маленькому ядру с оболочкой, инструменту мониторинга на eBPF, прошивке для датчика или
языку, который компилируется в WebAssembly. Начните с
[Урока 23.1 — Как устроены итоговые проекты](../23-capstone-projects/01-how-capstone-projects-work.md).

## Источники и что почитать

- [Writing an OS in Rust](https://os.phil-opp.com/) и [Rust for Linux](https://rust-for-linux.com/) — направление ОС.
- [The Aya book](https://aya-rs.dev/book/) и [ebpf.io — What is eBPF?](https://ebpf.io/what-is-ebpf/) — направление eBPF.
- [The Embedded Rust Book](https://docs.rust-embedded.org/book/) и [the Embassy book](https://embassy.dev/book/) — направление встраиваемых систем.
- [Apache DataFusion](https://datafusion.apache.org/) — направление баз данных.
- [Pingora](https://github.com/cloudflare/pingora) — направление сетей.
- [Crafting Interpreters](https://craftinginterpreters.com/) и [Wasmtime documentation](https://docs.wasmtime.dev/) — направление компиляторов.
- [kube.rs](https://kube.rs/) — направление cloud native.
- [rustc — Platform Support](https://doc.rust-lang.org/rustc/platform-support.html) — все целевые платформы, которые поддерживает Rust.

---

[🏠 Начало](../START-HERE.md) · [← 22.29 Трек cloud native: проект и следующие шаги](29-cloud-native-track-project-and-next-steps.md) · [📋 Этап 22](README.md) · [Этап 23 — Итоговые проекты →](../23-capstone-projects/README.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
