<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 17.10 Транзакции и ACID](10-transactions-and-acid.md) · [📋 Этап 17](README.md) · [17.12 MVCC: многоверсионное управление конкурентным доступом →](12-mvcc.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/17-storage-and-databases/11-isolation-levels.md) <!-- nav -->

# Урок 17.11 — Уровни изоляции

> **Этап 17 — Как устроены хранилища и базы данных** · Урок 11 из 15 · ⏱ около 120 минут

## Что вы узнаете

- Пять классических аномалий: грязное чтение, неповторяющееся чтение, фантомное чтение, потерянное обновление и
  перекос записи.
- Четыре уровня изоляции стандарта SQL (Structured Query Language, язык запросов к реляционным базам данных) и что на самом деле делает PostgreSQL на каждом из них.
- Как вызвать каждую аномалию двумя терминалами `psql` и как более сильный уровень её останавливает.
- Три способа избежать потерянных обновлений: атомарный `UPDATE`, `SELECT ... FOR UPDATE` и `SERIALIZABLE` с
  повторными попытками.
- Как правильно повторять транзакцию после ошибки сериализации (код `40001`) в Rust через sqlx.

## Перед началом

- Вы прошли [Урок 17.10 — Транзакции и ACID](10-transactions-and-acid.md) и видели там потерянное обновление.
- Вы знаете состояния гонки из [Урока 14.1 — Почему конкурентность сложна: гонки и взаимные блокировки](../14-concurrency-and-async/01-why-concurrency-is-hard.md)
  и sqlx из [Урока 16.6 — Работа с базами данных через sqlx](../16-backend-services/06-databases-with-sqlx.md).
- Контейнер `p17-postgres` из [Урока 17.9](09-page-cache-and-buffer-pool.md) запущен
  (`docker start p17-postgres`, если он остановлен).

## Новые слова

| Слово | Значение |
|---|---|
| **аномалия** (anomaly) | Результат, который никогда не получился бы, если бы транзакции выполнялись одна за другой. |
| **последовательное выполнение** (serial execution) | Выполнение транзакций по одной, без пересечений. Всегда корректно, но медленно. |
| **сериализуемость** (serializable) | Способ выполнять транзакции одновременно так, что результат совпадает с *каким-то* последовательным порядком. |
| **уровень изоляции** (isolation level) | Настройка, которая говорит, какие аномалии транзакция допускает в обмен на скорость. |
| **грязное чтение** (dirty read) | Чтение изменения другой транзакции, которая ещё не зафиксирована (и ещё может откатиться). |
| **неповторяющееся чтение** (non-repeatable read) | Двукратное чтение одной строки в одной транзакции даёт два разных значения, потому что между чтениями другая транзакция зафиксировалась. |
| **фантомное чтение** (phantom read) | Один и тот же запрос с `WHERE` дважды возвращает разный *набор* строк, потому что другая транзакция вставила или удалила строки. |
| **перекос записи** (write skew) | Две транзакции читают одни данные, каждая принимает по ним решение и меняет *разные* строки; вместе они нарушают правило, которое каждая по отдельности соблюдала. |
| **снимок** (snapshot) | Застывшая картина зафиксированных данных на один момент. Транзакция, читающая из снимка, не видит более поздних фиксаций. |
| **ошибка сериализации** (serialization failure) | База данных прерывает транзакцию, потому что её фиксация привела бы к аномалии. PostgreSQL сообщает о ней кодом `40001`; лечится повторной попыткой. |
| **SQLSTATE** | Пятисимвольный код ошибки, определённый стандартом SQL, например `40001` (ошибка сериализации) или `23505` (нарушение уникальности). |
| **блокировка строки** (row lock) | Блокировка одной строки: другие транзакции, которые хотят изменить (или заблокировать) ту же строку, ждут, пока владелец не зафиксируется или не откатится. |
| **SSI** (Serializable Snapshot Isolation, сериализуемая изоляция снимков) | Метод PostgreSQL для `SERIALIZABLE`: снимки плюс отслеживание зависимостей чтения и записи; транзакция прерывается, когда появляется опасный шаблон. |

## 1. Почему бы не выполнять транзакции по одной?

Простейшая корректная база данных выполняет одну транзакцию, потом следующую: **последовательное выполнение**.
Ничто не пересекается, поэтому ни одна аномалия невозможна. Но одна медленная транзакция (отчёт, который читает данные
минуту) заблокировала бы всех остальных клиентов.

Настоящие базы данных выполняют много транзакций сразу и дают вам выбор — **уровень изоляции**: насколько транзакции
могут «видеть» друг друга. Самый сильный уровень, **serializable**, обещает, что результат совпадёт с *каким-то*
последовательным порядком. Более слабые уровни быстрее и допускают определённые аномалии. Чтобы выбрать уровень, нужно
знать аномалии.

## 2. Аномалии

Каждая аномалия — это временная линия двух транзакций, T1 и T2. Время идёт вниз.

```text
dirty read                         non-repeatable read                phantom read
T1                T2               T1                T2               T1                       T2
                  set x = 0        read x -> 100                      count(on_call) -> 2
read x -> 0  (!)                                     set x = 70                                 insert on_call
                  ROLLBACK                           COMMIT                                     COMMIT
                                   read x -> 70 (!)                   count(on_call) -> 3 (!)

lost update                        write skew (rule: at least one doctor on call)
T1                T2               T1                          T2
read x -> 100                      count(on_call) -> 2
                  read x -> 100                                count(on_call) -> 2
write x = 90                       ann: on_call = false
COMMIT                                                         ben: on_call = false
                  write x = 90     COMMIT                      COMMIT
                  COMMIT  (!)      -> nobody is on call (!)
```

- **Грязное чтение**: T1 увидела значение, которого официально никогда не существовало.
- **Неповторяющееся чтение**: одна и та же строка, прочитанная дважды, даёт два ответа внутри одной транзакции.
- **Фантомное чтение**: один и тот же запрос возвращает другой набор строк (появилась строка-«фантом»).
- **Потерянное обновление**: обе прочитали 100 и сняли по 10; результат 90 вместо 80. Вы построили это в Уроке 17.10.
- **Перекос записи**: каждая транзакция проверила «дежурят 2 врача, значит, я могу уйти». Каждое изменение по
  отдельности было в порядке; вместе они нарушили правило. Ни одна строка не была записана дважды, поэтому перекос
  записи трудно заметить.

## 3. Уровни изоляции: стандарт и PostgreSQL

Стандарт SQL определяет четыре уровня через аномалии, которые они запрещают (таблица стандарта перечисляет только первые
три):

| Уровень | Грязное чтение | Неповторяющееся чтение | Фантомное чтение |
|---|---|---|---|
| `READ UNCOMMITTED` | возможно | возможно | возможно |
| `READ COMMITTED` | нет | возможно | возможно |
| `REPEATABLE READ` | нет | нет | возможно |
| `SERIALIZABLE` | нет | нет | нет |

Стандарт говорит, что уровень **обязан запрещать**; база данных может запрещать больше. PostgreSQL так и делает:

| Вы просите | PostgreSQL даёт | Каждый оператор видит | Потерянное обновление | Перекос записи |
|---|---|---|---|---|
| `READ UNCOMMITTED` | то же, что `READ COMMITTED` | — | — | — |
| `READ COMMITTED` (по умолчанию) | новый снимок для **каждого оператора** | данные, зафиксированные до начала оператора | возможно (чтение, затем запись) | возможно |
| `REPEATABLE READ` | один снимок на **всю транзакцию** (изоляция снимков) | данные, зафиксированные до первого оператора | ошибка `40001` | возможно |
| `SERIALIZABLE` | изоляция снимков + SSI | как в `REPEATABLE READ` | ошибка `40001` | ошибка `40001` |

Значит, в PostgreSQL грязное чтение невозможно никогда, а `REPEATABLE READ` защищает и от фантомов. Как снимки устроены
внутри — тема Урока 17.12; здесь мы ими только пользуемся.

## 4. Аномалии в PostgreSQL своими глазами

Откройте **два** терминала и в каждом выполните `docker exec -it p17-postgres psql -U postgres`. Назовём их T1 и T2. В
любом из них создайте две маленькие таблицы:

```sql
CREATE TABLE accounts (name text PRIMARY KEY, balance int NOT NULL);
INSERT INTO accounts VALUES ('alice', 100), ('bob', 50);
CREATE TABLE doctors (name text PRIMARY KEY, on_call boolean NOT NULL);
INSERT INTO doctors VALUES ('ann', true), ('ben', true);
```

Теперь вводите операторы ниже в порядке номеров шагов, каждый в своём терминале. Стрелка показывает, что напечатал
`psql`. Оператор, введённый вне `BEGIN ... COMMIT`, — это отдельная маленькая транзакция, которая сразу фиксируется.

**Неповторяющееся чтение на `READ COMMITTED` и как его останавливает `REPEATABLE READ`:**

```text
step  T1                                                        T2
1     BEGIN;
2     SELECT balance FROM accounts WHERE name = 'alice';  -> 100
3                                                               UPDATE accounts SET balance = 70 WHERE name = 'alice';
4     SELECT balance FROM accounts WHERE name = 'alice';  -> 70
5     COMMIT;
6                                                               UPDATE accounts SET balance = 100 WHERE name = 'alice';
7     BEGIN ISOLATION LEVEL REPEATABLE READ;
8     SELECT balance FROM accounts WHERE name = 'alice';  -> 100
9                                                               UPDATE accounts SET balance = 70 WHERE name = 'alice';
10    SELECT balance FROM accounts WHERE name = 'alice';  -> 100
11    COMMIT;
12    SELECT balance FROM accounts WHERE name = 'alice';  -> 70
13                                                              UPDATE accounts SET balance = 100 WHERE name = 'alice';
```

На `READ COMMITTED` каждый оператор берёт свежий снимок, поэтому шаг 4 видит фиксацию T2. На `REPEATABLE READ` снимок
берётся на первом операторе (шаг 8) и сохраняется, поэтому шаг 10 по-прежнему видит 100. После `COMMIT` шаг 12 видит
новое значение.

**Никакого грязного чтения, даже если о нём попросить:**

```text
step  T1                                                        T2
1                                                               BEGIN;
2                                                               UPDATE accounts SET balance = 0 WHERE name = 'alice';
3     BEGIN ISOLATION LEVEL READ UNCOMMITTED;
4     SELECT balance FROM accounts WHERE name = 'alice';  -> 100
5                                                               ROLLBACK;
6     COMMIT;
```

**Потерянное обновление на `READ COMMITTED`:** обе читают 100, обе пишут 90, обе фиксируются; после этого
`SELECT balance FROM accounts WHERE name = 'alice'` печатает 90. Десять потеряны — ровно как в Уроке 17.10. Верните
alice 100 и попробуйте то же на `REPEATABLE READ`:

```text
step  T1                                                        T2
1     BEGIN ISOLATION LEVEL REPEATABLE READ;
2     SELECT balance FROM accounts WHERE name = 'alice';  -> 100
3                                                               BEGIN ISOLATION LEVEL REPEATABLE READ;
4                                                               SELECT balance FROM accounts WHERE name = 'alice';  -> 100
5     UPDATE accounts SET balance = 90 WHERE name = 'alice';
6     COMMIT;
7                                                               UPDATE accounts SET balance = 90 WHERE name = 'alice';
8                                                               ROLLBACK;
```

Шаг 7 в T2 падает:

```text
ERROR:  could not serialize access due to concurrent update
```

T2 хотела изменить строку, которую изменила транзакция, зафиксированная после снимка T2. Вместо того чтобы молча
потерять обновление T1, PostgreSQL прерывает T2. Её приложение должно начать всю транзакцию заново (снова прочитать,
снова решить, снова записать).

**Перекос записи на `REPEATABLE READ`, остановленный `SERIALIZABLE`:** выполните это дважды: сначала с
`REPEATABLE READ` на шагах 1 и 2, затем (после `UPDATE doctors SET on_call = true;`) с `SERIALIZABLE`:

```text
step  T1                                                        T2
1     BEGIN ISOLATION LEVEL REPEATABLE READ;
2                                                               BEGIN ISOLATION LEVEL REPEATABLE READ;
3     SELECT count(*) FROM doctors WHERE on_call;  -> 2
4                                                               SELECT count(*) FROM doctors WHERE on_call;  -> 2
5     UPDATE doctors SET on_call = false WHERE name = 'ann';
6                                                               UPDATE doctors SET on_call = false WHERE name = 'ben';
7     COMMIT;
8                                                               COMMIT;
```

С `REPEATABLE READ` обе фиксации успешны, и `SELECT * FROM doctors;` показывает `ann | f` и `ben | f`: никто не
дежурит. Две транзакции меняли разные строки, поэтому изоляции снимков нечего было обнаружить. С `SERIALIZABLE` шаг 8
падает:

```text
ERROR:  could not serialize access due to read/write dependencies among transactions
DETAIL:  Reason code: Canceled on identification as a pivot, during commit attempt.
HINT:  The transaction might succeed if retried.
```

SSI заметила шаблон: T1 читала строки, которые изменила T2, а T2 читала строки, которые изменила T1. Никакой
последовательный порядок не даёт «ушли оба», поэтому одна из них должна уйти. Изменение Ann зафиксировано, Ben
по-прежнему дежурит. Прочитайте `HINT`: повторная попытка — ожидаемая реакция.

> 💡 **Совет:** `SHOW transaction_isolation;` печатает уровень текущей транзакции. Чтобы изменить уровень по умолчанию
> для одного сеанса: `SET default_transaction_isolation = 'serializable';`.

## 5. Три способа избежать потерянных обновлений

1. **Пусть арифметику делает база.** `UPDATE accounts SET balance = balance - 10 WHERE name = 'alice'` читает и пишет
   строку **одним** оператором, под **блокировкой строки**. Если две транзакции выполняют его одновременно, вторая ждёт
   фиксации первой, затем перечитывает строку и вычитает ещё раз. Мы выполнили его из двух терминалов: у alice было
   100, стало 80 — на уровне по умолчанию `READ COMMITTED`. Это лучший инструмент, когда изменение — простая
   арифметика.
2. **Блокируйте то, что читаете.** `SELECT ... FOR UPDATE` берёт блокировки строк уже при чтении. Вторая транзакция,
   выполняющая тот же `SELECT ... FOR UPDATE`, ждёт, пока первая не зафиксируется, и затем видит новые данные.
   Упражнение 4 так чинит пример с врачами.
3. **Используйте `SERIALIZABLE` и повторяйте.** Вы пишете простой код «прочитать, затем записать», а база прерывает
   транзакции, которые привели бы к аномалии. Это защищает даже от ошибок, о которых вы не подумали, но каждая
   транзакция должна быть готова выполниться снова.

> 🔒 **Безопасность:** многие реальные ошибки в магазинах и банках — потерянные обновления и перекосы записи, которые
> злоумышленники вызывают намеренно: отправляют двадцать запросов в одну миллисекунду, чтобы использовать один купон
> двадцать раз. Тесты, отправляющие по одному запросу, такие ошибки никогда не находят. Раздел 6 — тест того вида,
> который находит.

## 6. Повторные попытки в Rust

Теперь потерянное обновление с настоящей конкурентностью: 8 задач Tokio по 50 раз прибавляют 1 к счётчику, читая
значение и записывая значение + 1 в транзакции. Сначала на `READ COMMITTED`, затем на `SERIALIZABLE` с повторами.

```bash
cd ~/projects/rust
cargo new code/p17-isolation
cd code/p17-isolation
cargo add sqlx@0.9 --features runtime-tokio,postgres
cargo add tokio@1 --features macros,rt-multi-thread
```

Замените `src/main.rs` на:

```rust
use sqlx::PgPool;
use sqlx::postgres::PgPoolOptions;

const TASKS: u32 = 8;
const STEPS: u32 = 50;

#[tokio::main]
async fn main() -> Result<(), sqlx::Error> {
    let url = std::env::var("DATABASE_URL")
        .unwrap_or("postgres://postgres:dev-password@127.0.0.1:5432/postgres".into());
    let pool = PgPoolOptions::new()
        .max_connections(8)
        .connect(&url)
        .await?;
    sqlx::query("CREATE TABLE IF NOT EXISTS counter (id int PRIMARY KEY, n bigint NOT NULL)")
        .execute(&pool)
        .await?;

    for level in ["READ COMMITTED", "SERIALIZABLE"] {
        sqlx::query("INSERT INTO counter VALUES (1, 0) ON CONFLICT (id) DO UPDATE SET n = 0")
            .execute(&pool)
            .await?;
        let mut handles = Vec::new();
        for _ in 0..TASKS {
            let pool = pool.clone();
            handles.push(tokio::spawn(async move {
                let mut retries = 0;
                for _ in 0..STEPS {
                    // try again until the transaction commits
                    while let Err(e) = increment(&pool, level).await {
                        if !is_serialization_failure(&e) {
                            return Err(e);
                        }
                        retries += 1;
                    }
                }
                Ok::<u32, sqlx::Error>(retries)
            }));
        }
        let mut retries = 0;
        for handle in handles {
            retries += handle.await.expect("task panicked")?;
        }
        let n: i64 = sqlx::query_scalar("SELECT n FROM counter WHERE id = 1")
            .fetch_one(&pool)
            .await?;
        println!(
            "{level:<15} counter = {n} (expected {}), retries = {retries}",
            TASKS * STEPS
        );
    }
    Ok(())
}

/// Reads the counter, then writes counter + 1, in one transaction.
async fn increment(pool: &PgPool, level: &str) -> Result<(), sqlx::Error> {
    let mut tx = pool.begin().await?;
    let set_level = match level {
        "SERIALIZABLE" => "SET TRANSACTION ISOLATION LEVEL SERIALIZABLE",
        _ => "SET TRANSACTION ISOLATION LEVEL READ COMMITTED",
    };
    sqlx::query(set_level).execute(&mut *tx).await?;
    let n: i64 = sqlx::query_scalar("SELECT n FROM counter WHERE id = 1")
        .fetch_one(&mut *tx)
        .await?;
    sqlx::query("UPDATE counter SET n = $1 WHERE id = 1")
        .bind(n + 1)
        .execute(&mut *tx)
        .await?;
    tx.commit().await
}

/// PostgreSQL's code 40001 means "serialization failure: please retry".
fn is_serialization_failure(e: &sqlx::Error) -> bool {
    matches!(e, sqlx::Error::Database(db) if db.code().as_deref() == Some("40001"))
}
```

```bash
cargo run
```

```text
READ COMMITTED  counter = 86 (expected 400), retries = 0
SERIALIZABLE    counter = 400 (expected 400), retries = 2443
```

У вас числа будут другими при каждом запуске (другой запуск напечатал 84 и 2325), потому что они зависят от того, как
по времени чередуются задачи.

**Что происходит:**

1. Программа подключается по URL (Uniform Resource Locator, здесь адрес базы данных) в том же стиле, что и в Уроке 16.6 (задайте `DATABASE_URL`, чтобы использовать другую
   базу). Она создаёт `counter` и обнуляет его перед каждым уровнем.
2. `increment` начинает транзакцию, устанавливает её уровень изоляции (это должен быть первый оператор транзакции),
   читает `n`, пишет `n + 1` и фиксирует.
3. На `READ COMMITTED` восемь задач снова и снова читают одно и то же `n` и перезаписывают друг друга: 314 из 400
   прибавлений потеряны, и никакой ошибки не сообщается. В этом и опасность: **неверные данные без ошибки**.
4. На `SERIALIZABLE` PostgreSQL прерывает транзакцию всякий раз, когда другая изменила `n` после её снимка. Ошибка —
   `sqlx::Error::Database`, а `db.code()` возвращает SQLSTATE `"40001"`. Цикл `while let` выполняет **всю** транзакцию
   заново. Счётчик точен.
5. Цена — 2443 повтора: восемь задач, сражающихся за одну строку, — худший случай для оптимистичных методов. Для
   «горячего» счётчика атомарный `UPDATE` из раздела 5 гораздо лучше (Упражнение 2).

> 🧪 **Тестирование:** тест на ошибки конкурентности должен создавать настоящую конкурентность: много задач, много
> повторений, настоящую базу данных. Упражнение 3 превращает эту программу в тест, который падает, если потеряно хотя
> бы одно прибавление.

## Частые ошибки

> ⚠️ **Частая ошибка:** собирать SQL через `format!`. sqlx 0.9 отвергает динамическую строку при компиляции:
>
> ```text
> error[E0277]: dynamic SQL strings should be audited for possible injections
>    --> src/main.rs:62:17
>     |
>  62 |     sqlx::query(&sql).execute(&mut *tx).await?;
>     |     ----------- ^^^^ dynamic SQL string
>     |     |
>     |     required by a bound introduced by this call
>     |
>     = help: the trait `SqlSafeStr` is not implemented for `&std::string::String`
> ```
>
> Используйте литеральный SQL с параметрами `$1` для значений. Уровень изоляции не может быть параметром, поэтому
> программа выбирает между двумя литеральными строками через `match`.

> ⚠️ **Частая ошибка:** повторять только упавший оператор, а не всю транзакцию. После ошибки транзакция прервана, а её
> прежние чтения всё равно устарели. Начинайте снова с `begin`.

> ⚠️ **Частая ошибка:** повторять **любую** ошибку. Повторяйте только ошибки сериализации (`40001`) и взаимные
> блокировки (`40P01`). Синтаксическая ошибка или нарушенное ограничение будут падать снова вечно; возвращайте их.

## Попробуйте сами

1. В примере с `REPEATABLE READ` из раздела 4 пусть T2 между двумя `SELECT count(*)` транзакции T1 вставит третьего
   врача (`INSERT INTO doctors VALUES ('cat', true);`). T1 всё равно насчитает 2: изоляция снимков скрывает и фантомы.
2. Поставьте `TASKS` равным 1 и запустите программу. Оба уровня теперь печатают 50 и 0 повторов: без конкурентности
   нет аномалий, поэтому такие ошибки и прячутся во время разработки.

## Упражнения

### Упражнение 1 — Назовите аномалию (легко)

Какая аномалия в каждой истории и какой самый слабый уровень PostgreSQL её предотвращает?

1. Отчёт дважды в одной транзакции суммирует все балансы и получает два разных итога, потому что между ними
   зафиксировался перевод.
2. Два пользователя бронируют переговорную на 10:00: оба проверяют, что в `bookings` нет строки для этой комнаты и
   часа, затем каждый вставляет свою строку бронирования.
3. Два администратора одновременно переименовывают один и тот же товар; одно имя теряется.

<details>
<summary>Подсказка</summary>

Спросите себя: менялись ли те же строки между двумя чтениями (неповторяющееся / фантомное)? Писали ли обе в одну строку
(потерянное обновление)? Писали ли они разные строки после чтения одних данных (перекос записи)?

</details>

<details>
<summary>Ответ</summary>

1. Неповторяющееся чтение (или фантомное, если появились строки): `REPEATABLE READ`.
2. Перекос записи (здесь с фантомом: проверяемых строк ещё не было): обе прочитали «бронирований нет» и вставили разные
   строки. Предотвращает его только `SERIALIZABLE` (или ограничение `UNIQUE` на комнату и час).
3. Потерянное обновление, если они сначала читают, а потом пишут: `REPEATABLE READ` превращает его в ошибку `40001`;
   простой `UPDATE products SET name = $1 WHERE id = $2` без чтения — вовсе не потерянное обновление, а просто
   «побеждает последний записавший».

Почему это работает: аномалия определяется тем, *что каждая транзакция читала и писала*, а не историей из бизнеса.

</details>

### Упражнение 2 — Атомарное прибавление (средне)

Добавьте в программу третий режим `"ATOMIC"`: для него `increment` возвращает `increment_atomic(pool).await`, а
`increment_atomic` выполняет `UPDATE counter SET n = n + 1 WHERE id = 1` без явной транзакции. Запустите все три.

<details>
<summary>Подсказка</summary>

Добавьте `"ATOMIC"` в массив в `main` и проверяйте `if level == "ATOMIC"` в начале `increment`, до `begin`.

</details>

<details>
<summary>Ответ</summary>

```rust
/// One statement: the database reads and writes the row under a row lock.
async fn increment_atomic(pool: &PgPool) -> Result<(), sqlx::Error> {
    sqlx::query("UPDATE counter SET n = n + 1 WHERE id = 1")
        .execute(pool)
        .await?;
    Ok(())
}
```

```text
READ COMMITTED  counter = 80 (expected 400), retries = 0
SERIALIZABLE    counter = 400 (expected 400), retries = 1553
ATOMIC          counter = 400 (expected 400), retries = 0
```

Почему это работает: чтение и запись происходят внутри одного оператора, пока строка заблокирована, поэтому никакая
другая транзакция не может вклиниться между ними. Ждать блокировку строки гораздо дешевле, чем выбросить транзакцию и
выполнить её заново.

</details>

### Упражнение 3 — Тест, который ловит потерянные прибавления (средне)

Напишите в `src/main.rs` `#[tokio::test] async fn serializable_with_retry_loses_no_increment()`: 4 задачи × 25
прибавлений на `SERIALIZABLE` с повторами; проверьте, что счётчик равен 100. Запустите `cargo test`, пока работает
`p17-postgres`.

<details>
<summary>Подсказка</summary>

`#[tokio::test]` похож на `#[test]`, но запускает рантайм Tokio для `async fn`. Внутри цикла повторов проверяйте через
`assert!`, что каждая ошибка — ошибка сериализации, чтобы другие ошибки роняли тест, а не зацикливали его.

</details>

<details>
<summary>Ответ</summary>

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[tokio::test]
    async fn serializable_with_retry_loses_no_increment() {
        let url = std::env::var("DATABASE_URL")
            .unwrap_or("postgres://postgres:dev-password@127.0.0.1:5432/postgres".into());
        let pool = PgPool::connect(&url).await.unwrap();
        sqlx::query("CREATE TABLE IF NOT EXISTS counter (id int PRIMARY KEY, n bigint NOT NULL)")
            .execute(&pool)
            .await
            .unwrap();
        sqlx::query("INSERT INTO counter VALUES (1, 0) ON CONFLICT (id) DO UPDATE SET n = 0")
            .execute(&pool)
            .await
            .unwrap();

        let mut handles = Vec::new();
        for _ in 0..4 {
            let pool = pool.clone();
            handles.push(tokio::spawn(async move {
                for _ in 0..25 {
                    while let Err(e) = increment(&pool, "SERIALIZABLE").await {
                        assert!(is_serialization_failure(&e), "unexpected error: {e}");
                    }
                }
            }));
        }
        for handle in handles {
            handle.await.unwrap();
        }
        let n: i64 = sqlx::query_scalar("SELECT n FROM counter WHERE id = 1")
            .fetch_one(&pool)
            .await
            .unwrap();
        assert_eq!(n, 100);
    }
}
```

```text
running 1 test
test tests::serializable_with_retry_loses_no_increment ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.49s
```

Почему это работает: тест создаёт настоящую борьбу за одну строку. Замените `"SERIALIZABLE"` на `"READ COMMITTED"`, и он
упадёт (`left: 68`, `right: 100` в нашем запуске): тест способен увидеть ошибку. (Он использует ту же таблицу
`counter`, что и `main`, поэтому не запускайте их одновременно.)

</details>

### Упражнение 4 — Почините врачей через `FOR UPDATE` (сложно)

На уровне по умолчанию `READ COMMITTED` сделайте пример с врачами безопасным без `SERIALIZABLE`: каждая транзакция
сначала выполняет `SELECT name FROM doctors WHERE on_call FOR UPDATE;`, сама считает строки и уходит, только если их
хотя бы 2. Выполните это из двух терминалов (сброс — `UPDATE doctors SET on_call = true;`). Что видит T2 и когда?

<details>
<summary>Подсказка</summary>

`SELECT ... FOR UPDATE` в T2 хочет строки, которые заблокировала T1, поэтому терминал T2 «зависает», пока T1 не
зафиксируется.

</details>

<details>
<summary>Ответ</summary>

```text
step  T1                                                        T2
1     BEGIN;
2                                                               BEGIN;
3     SELECT name FROM doctors WHERE on_call FOR UPDATE;  -> ben, ann (2 rows)
4                                                               SELECT name FROM doctors WHERE on_call FOR UPDATE;  (waits)
5     UPDATE doctors SET on_call = false WHERE name = 'ann';
6     COMMIT;                                                   -> ben (1 row)
7                                                               COMMIT;   (1 row: Ben stays)
```

Почему это работает: T1 заблокировала обе прочитанные строки. T2 ждала этих блокировок; когда T1 зафиксировалась, T2
перепроверила строки, увидела, что Ann больше не дежурит, и вернула только Ben. Видя одного врача, T2 решает не уходить,
и правило соблюдено. Цена: T2 пришлось ждать. (На `REPEATABLE READ` T2 вместо ожидания и перепроверки получила бы ошибку
`could not serialize access due to concurrent update`.)

</details>

## Вопросы для самопроверки

1. Что обещает «serializable»?

<details>
<summary>Ответ</summary>

Что результат одновременных транзакций такой же, как если бы они выполнились одна за другой в *каком-то* порядке.

</details>

2. Чем в PostgreSQL отличаются `READ COMMITTED` и `REPEATABLE READ`?

<details>
<summary>Ответ</summary>

`READ COMMITTED` берёт новый снимок для каждого оператора, поэтому транзакция может видеть фиксации, случившиеся во
время её работы. `REPEATABLE READ` берёт один снимок на первом операторе и использует его всю транзакцию.

</details>

3. Почему `REPEATABLE READ` не останавливает перекос записи?

<details>
<summary>Ответ</summary>

Две транзакции пишут **разные** строки, поэтому нет конфликта «запись-запись», который можно обнаружить. Проблема в
том, что каждое решение основано на данных, которые изменила другая; только `SERIALIZABLE` (SSI) отслеживает такие
зависимости чтения и записи.

</details>

4. Ваш код получил SQLSTATE `40001`. Что ему делать и чего не делать?

<details>
<summary>Ответ</summary>

Выполнить всю транзакцию заново с самого начала, возможно несколько раз с короткой паузой. Не нужно повторять только
упавший оператор и не нужно повторять другие виды ошибок.

</details>

5. Какое исправление лучше всего для «горячего» счётчика и почему?

<details>
<summary>Ответ</summary>

Атомарный `UPDATE ... SET n = n + 1`: чтение и запись происходят одним оператором под блокировкой строки, поэтому ничего
не теряется и ничего не нужно повторять.

</details>

## Итоги

- Изоляция — компромисс: последовательное выполнение корректно, но медленно; более слабые уровни ради скорости
  допускают аномалии.
- Аномалии: грязное чтение, неповторяющееся чтение, фантомное чтение, потерянное обновление и перекос записи.
- PostgreSQL: `READ UNCOMMITTED` = `READ COMMITTED` (снимок на оператор, по умолчанию); `REPEATABLE READ` = один снимок
  на транзакцию, потерянные обновления становятся `40001`; `SERIALIZABLE` с помощью SSI останавливает и перекос записи.
- Избегайте потерянных обновлений атомарными `UPDATE`, `SELECT ... FOR UPDATE` или `SERIALIZABLE` с повторами.
- Повторяйте **всю** транзакцию и только при `40001` (и взаимных блокировках, `40P01`).
- Ошибки конкурентности дают неверные данные без ошибок; их находят только тесты с настоящей конкурентностью.

## Источники и что почитать

- [PostgreSQL: Transaction Isolation](https://www.postgresql.org/docs/current/transaction-iso.html) — точное поведение каждого уровня с примерами.
- [PostgreSQL: Serialization Failure Handling](https://www.postgresql.org/docs/current/mvcc-serialization-failure-handling.html) — какие ошибки повторять и как.
- [PostgreSQL: Explicit Locking](https://www.postgresql.org/docs/current/explicit-locking.html) — блокировки строк и `FOR UPDATE`.
- [PostgreSQL: SET TRANSACTION](https://www.postgresql.org/docs/current/sql-set-transaction.html) — как выбрать уровень.
- [A Critique of ANSI SQL Isolation Levels](https://www.microsoft.com/en-us/research/publication/a-critique-of-ansi-sql-isolation-levels/) — классическая статья, показавшая пробелы в определениях стандарта (ANSI, American National Standards Institute, Американский национальный институт стандартов, публикует стандарт SQL в США).
- [Jepsen: Consistency Models](https://jepsen.io/consistency) — карта моделей изоляции и согласованности с понятными определениями.
- [Документация sqlx](https://docs.rs/sqlx/latest/sqlx/) — транзакции и ошибки базы данных в sqlx.

---

[🏠 Начало](../START-HERE.md) · [← 17.10 Транзакции и ACID](10-transactions-and-acid.md) · [📋 Этап 17](README.md) · [17.12 MVCC: многоверсионное управление конкурентным доступом →](12-mvcc.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
