<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 20.8 Перегрузка, лимиты и обратное давление](08-overload-limits-and-backpressure.md) · [📋 Этап 20](README.md) · [20.10 Проект: укрепить сервис →](10-project-harden-a-service.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/20-reliability-and-hardening/09-production-readiness-checklist.md) <!-- nav -->

# Урок 20.9 — Чек-лист готовности к продакшену

> **Этап 20 — Надёжность и защита** · Урок 9 из 11 · ⏱ около 2,5 часа (2 занятия)

## Что вы узнаете

- Что такое проверка готовности к продакшену и почему каждому пункту чек-листа нужны **доказательства**.
- Полный чек-лист для сервиса на Rust со ссылкой на урок, который учит каждому пункту.
- Как проверить всю конфигурацию при запуске и отказаться работать с понятным сообщением.
- Чем отличаются проверки живости (liveness) и готовности (readiness) и как сервис отводит трафик (draining) перед остановкой.
- Как поручить скрипту проверку тех пунктов, которые может проверить машина.

## Перед началом

- Вы прошли [Урок 20.8 — Перегрузка, лимиты и обратное давление](08-overload-limits-and-backpressure.md).
- Вы знаете корректное завершение и сигналы ([Урок 14.14](../14-concurrency-and-async/14-cancellation-timeouts-and-graceful-shutdown.md)), проверки работоспособности и конфигурацию ([Урок 16.13](../16-backend-services/13-configuration-health-checks-and-containers.md)), линты Clippy против паник ([Урок 20.2](02-errors-and-panics-in-production.md)) и тестирование маршрутизатора через `oneshot` ([Урок 16.3](../16-backend-services/03-web-services-with-axum.md)).
- Создайте пакет для этого урока. Это библиотека (части, которые мы тестируем) и программа (`src/main.rs`) в одном пакете:

```bash
cd ~/projects/rust
cargo new --lib code/p20-ready
cd code/p20-ready
cargo add tokio@1 --features full
cargo add axum@0.8
cargo add --dev tower@0.5 --features util
```

## Новые слова

| Слово | Значение |
|---|---|
| **продакшен** (production) | Настоящее окружение, где сервисом пользуются настоящие пользователи. |
| **PRR** (Production Readiness Review, проверка готовности к продакшену) | Проверка перед выходом сервиса в продакшен (или после большого изменения): команда проходит по чек-листу и показывает доказательства по каждому пункту. |
| **доказательство** (evidence) | То, что подтверждает выполнение пункта: тест, команда и её вывод, дашборд, документ. «Мы так думаем» — не доказательство. |
| **быстрый отказ** (fail fast) | Сразу остановиться с понятной ошибкой, когда что-то не так, вместо того чтобы работать в сломанном состоянии. |
| **живость** (liveness) | «Жив ли процесс и может ли он отвечать?» Если нет, платформа его перезапускает. |
| **готовность** (readiness) | «Нужно ли отправлять этой копии трафик прямо сейчас?» Если нет, балансировщик нагрузки перестаёт слать ей запросы, но не перезапускает её. |
| **отвод трафика** (draining) | Перед остановкой: сообщить «не готов», подождать, пока балансировщик нагрузки перестанет слать новые запросы, затем завершить открытые. |
| **SIGTERM** | Сигнал «пожалуйста, завершись», который менеджеры служб (systemd, Kubernetes) отправляют перед остановкой процесса. |
| **ранбук** (runbook) | Короткий документ для дежурного: как распознать проблему, что проверить, что делать. |
| **откат** (rollback) | Возврат к предыдущей рабочей версии после неудачного развёртывания. |

## 1. Что значит «готов к продакшену»

Код, который проходит тесты на вашем компьютере, готов к **ревью**. Код, готовый к **продакшену**, к тому же переживает плохой ввод, медленные зависимости, перегрузку, перезапуски, атаки и ночь, когда чинить его придётся кому-то другому. Всё это никто не помнит наизусть, поэтому команды используют **чек-лист** на **проверке готовности к продакшену** (PRR): перед запуском команда проходит по каждому пункту и показывает доказательства.

Три правила делают чек-лист полезным:

1. **У каждого пункта есть доказательство.** «Тайм-ауты: да» — недостаточно. «Тайм-ауты: каждый исходящий вызов идёт через `call_with_deadline`, тест `slow_dependency_gives_503`» — достаточно.
2. **«Не нужно» допустимо, но с причиной.** Инструменту командной строки не нужен эндпоинт готовности. Напишите *почему*, чтобы следующему проверяющему не пришлось гадать.
3. **Чек-лист повторяют.** Не только перед первым запуском, но и после каждого большого изменения: новая зависимость, новое хранилище данных, новый вид трафика.

## 2. Чек-лист

Скопируйте эти таблицы в файл `READINESS.md` в своём проекте и заполните столбец доказательств. Последний столбец показывает, где в курсе изучался пункт.

**A. Корректность**

| Пункт | Доказательство (пример) | Урок |
|---|---|---|
| Модульные и интеграционные тесты для основных путей и путей ошибок | `cargo test`, отчёт о покрытии | 09.1, 09.7 |
| У каждого парсера недоверенного ввода есть фаззинг-цель, которая регулярно запускается | `fuzz/fuzz_targets/`, задание CI (Continuous Integration, непрерывная интеграция) по расписанию | 20.3 |
| У каждого блока `unsafe` есть комментарий `SAFETY`, и он проверен под Miri | `cargo +nightly miri test` | 06.6, 10.12, 20.4 |
| Конкурентный код с атомарными операциями или своими блокировками проверен с loom | тесты loom | 14.9, 20.4 |
| Форматирование и Clippy чистые в CI | лог CI | 09.12 |

**B. Обработка сбоев**

| Пункт | Доказательство (пример) | Урок |
|---|---|---|
| Ожидаемые ошибки возвращаются, а не вызывают панику; линты против паник включены | `[lints.clippy]` в `Cargo.toml` | 20.2 |
| Паника в одном запросе не убивает процесс и попадает в лог | panic hook, тест | 20.2 |
| У каждого исходящего вызова (база данных, HTTP — HyperText Transfer Protocol, кэш) есть тайм-аут | код-ревью, тест с медленным фейком | 14.14, 18.2 |
| Повторы только для идемпотентных операций, с задержкой и джиттером | код, тест | 18.3 |
| Поведение при недоступной зависимости проверено | хаос-тест с toxiproxy | 20.5 |

**C. Лимиты**

| Пункт | Доказательство (пример) | Урок |
|---|---|---|
| Максимальный размер тела запроса | `RequestBodyLimitLayer`, тест с большим телом | 16.4 |
| Лимит конкурентности со сбросом нагрузки или ограниченная очередь | стек слоёв, нагрузочный тест | 20.8 |
| Ограничение частоты для каждого клиента, где клиенты могут злоупотреблять | тест | 20.8 |
| Ограниченные каналы и пулы; размеры записаны и объяснены | код | 20.8, 16.6 |
| Ёмкость известна (запросов в секунду при целевой задержке) | отчёт нагрузочного теста | 19.2 |

**D. Наблюдаемость**

| Пункт | Доказательство (пример) | Урок |
|---|---|---|
| Структурированные логи с ID (идентификатором) запроса; в логах нет секретов и персональных данных | примеры строк лога | 09.10, 16.4 |
| Метрики частоты, ошибок и длительности для каждого эндпоинта | дашборд | 16.12 |
| Трассировки через границы сервисов | трасса в Jaeger | 16.12 |
| SLO (Service Level Objective, цель уровня обслуживания) и алерт, который срабатывает, когда он под угрозой | правило алерта | 20.1, 21.11 |

**E. Работоспособность и жизненный цикл**

| Пункт | Доказательство (пример) | Урок |
|---|---|---|
| Эндпоинты живости и готовности с разным смыслом | `/livez`, `/readyz`, тест | 16.13, этот урок |
| По SIGTERM: «не готов», отвод трафика, завершение открытых запросов, выход с `0` | скрипт остановки и его вывод | 14.14, этот урок |
| Жёсткий лимит времени на остановку | код | 14.14 |
| Сервис запускается без ручных шагов (миграции, прогрев) | лог запуска | 16.7 |

**F. Конфигурация и секреты**

| Пункт | Доказательство (пример) | Урок |
|---|---|---|
| Вся конфигурация проверяется при запуске; плохие значения останавливают запуск с понятным сообщением | тест, запуск с плохими значениями | 16.13, этот урок |
| Нет секретов в Git, образах и логах; они приходят из хранилища секретов или окружения | ревью, сканер секретов | 10.8 |
| Значения по умолчанию безопасны (например, слушать на `127.0.0.1`, если не сказано иное) | тесты конфигурации | 10.1 |

**G. Безопасность и цепочка поставок**

| Пункт | Доказательство (пример) | Урок |
|---|---|---|
| Ввод проверяется на границе; аутентификация и авторизация на каждом эндпоинте | тесты | 10.4, 10.7, 16.8 |
| `cargo audit` и `cargo deny` в CI, при каждой отправке и еженедельно | конфигурация CI | 10.13 |
| Новые зависимости проходят ревью (`cargo vet`); у релизов есть SBOM (Software Bill of Materials, спецификация состава ПО), контрольные суммы и подпись | `supply-chain/`, файлы релиза | 20.7 |
| Контейнер работает от пользователя без прав root, из небольшого образа | Dockerfile | 11.13 |

**H. Развёртывание и восстановление**

| Пункт | Доказательство (пример) | Урок |
|---|---|---|
| Откат на предыдущую версию **опробован**, а не только запланирован | заметки с проверки | 21.7 |
| Миграции базы данных обратно совместимы с работающей версией | ревью миграции | 16.7 |
| Резервные копии существуют **и восстановление из них проверено** | лог восстановления | — |
| Работает больше одной копии, так что один сбой — не отказ всего сервиса | конфигурация развёртывания | 20.1 |

**I. Эксплуатация**

| Пункт | Доказательство (пример) | Урок |
|---|---|---|
| Названы команда-владелец и дежурный | страница команды | 21.12 |
| Есть ранбук для самых вероятных алертов | `RUNBOOK.md` | 21.12 |
| Дашборды показывают SLO и основные зависимости | ссылка на дашборд | 21.9, 21.11 |

Некоторые пункты указывают на Этап 21, который идёт следующим, а несколько (отмечены —) в этом курсе не изучаются; узнайте о них из инструментов своей команды. Запишите их сейчас как «ещё не сделано»: чек-лист показывает и то, чего пока не хватает.

> 📏 **Соглашение:** держите чек-лист **достаточно коротким, чтобы им пользовались**. Тридцать точных пунктов с доказательствами лучше двухсот расплывчатых, которые никто не читает. Когда инцидент показывает пробел, добавьте для него один пункт.

В остальной части урока мы построим в коде два пункта из групп E и F, а затем скрипт для пунктов, которые может проверить машина.

## 3. Быстрый отказ при плохой конфигурации

Сервис, который запустился с неверной настройкой, хуже того, который не запустился вовсе: он может проработать часы, а потом упасть ночью, когда плохое значение впервые понадобится. Поэтому читайте и проверяйте **всё** при запуске, сообщайте о **каждой** проблеме сразу и останавливайтесь с ненулевым кодом возврата.

Замените `src/lib.rs`:

```rust
//! Two production-readiness items in code: a checked configuration and health endpoints.

use std::time::Duration;

/// Everything the service needs to start. Built once, checked once.
#[derive(Debug, Clone, PartialEq)]
pub struct Config {
    pub port: u16,
    pub drain: Duration,
}

impl Config {
    /// Reads the settings through `lookup` (environment variables in `main`, a list in tests)
    /// and reports **every** problem, not only the first one.
    pub fn from_lookup(lookup: impl Fn(&str) -> Option<String>) -> Result<Config, Vec<String>> {
        let mut errors = Vec::new();

        let port = match lookup("PORT") {
            None => 3000,
            Some(text) => match text.parse::<u16>() {
                Ok(port) if port > 0 => port,
                _ => {
                    errors.push(format!(
                        "PORT must be a number from 1 to 65535, got {text:?}"
                    ));
                    0
                }
            },
        };

        let drain_seconds = match lookup("DRAIN_SECONDS") {
            None => 2,
            Some(text) => match text.parse::<u64>() {
                Ok(seconds) if seconds <= 60 => seconds,
                _ => {
                    errors.push(format!("DRAIN_SECONDS must be 0 to 60, got {text:?}"));
                    0
                }
            },
        };

        if errors.is_empty() {
            Ok(Config {
                port,
                drain: Duration::from_secs(drain_seconds),
            })
        } else {
            Err(errors)
        }
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    /// A fake environment made from a list of (name, value) pairs.
    fn env(pairs: &[(&str, &str)]) -> impl Fn(&str) -> Option<String> {
        let pairs: Vec<(String, String)> = pairs
            .iter()
            .map(|(k, v)| (k.to_string(), v.to_string()))
            .collect();
        move |name| {
            pairs
                .iter()
                .find(|(k, _)| k == name)
                .map(|(_, v)| v.clone())
        }
    }

    #[test]
    fn reports_every_bad_value() {
        let errors =
            Config::from_lookup(env(&[("PORT", "eighty"), ("DRAIN_SECONDS", "600")])).unwrap_err();
        assert_eq!(errors.len(), 2, "{errors:?}");
        assert!(errors[0].contains("PORT"));
        assert!(errors[1].contains("DRAIN_SECONDS"));
    }
}
```

`cargo test --lib`:

```text
test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

**Что происходит:**

1. `from_lookup` сам не вызывает `std::env::var`. Он получает **функцию**, которая ищет значение по имени. `main` передаёт `|name| std::env::var(name).ok()`, тесты — поддельное окружение. Тесты, которые меняют настоящие переменные окружения, мешали бы друг другу, потому что все тесты одного бинарного файла работают в одном процессе.
2. У каждой настройки есть безопасное значение по умолчанию (`None => 3000`) и проверка диапазона. `Ok(port) if port > 0` — это охранное условие (guard) ветки `match` (Урок 04.4): порт 0 разбирается как `u16`, но слушать на нём нельзя.
3. Плохое значение не останавливает функцию. Сообщение попадает в `errors`, значение-заглушка позволяет коду идти дальше, а в конце функция возвращает **все** сообщения. Тот, кто исправляет конфигурацию, видит все проблемы за один запуск, а не по одной за перезапуск.
4. Каждое сообщение говорит, какая настройка, что допустимо и что передано (`{text:?}` добавляет кавычки, так что видно и пустое значение, и значение из одних пробелов).

## 4. Живость, готовность и отвод трафика

Платформа вроде Kubernetes задаёт каждой копии сервиса два разных вопроса (настраивать их вы будете в Уроке 21.5):

| Вопрос | Эндпоинт | Если ответ «нет» |
|---|---|---|
| **Живость:** ты жив? | `/livez` | Процесс **перезапускается**. |
| **Готовность:** можно слать тебе трафик? | `/readyz` | Балансировщик нагрузки **перестаёт слать** запросы этой копии; её не перезапускают. |

(В Уроке 16.13 эндпоинт живости назывался `/healthz`, и на Этапе 21 это имя остаётся у проверки живости (liveness probe) в Kubernetes. Имя — лишь соглашение: сам API-сервер Kubernetes использует `/livez` и `/readyz`, как и этот этап. Важно, что два эндпоинта отвечают на разные вопросы; проверку можно направить на любой путь.)

Разница важна. Готовность — «нет», пока сервис запускается (кэши ещё пусты) и пока он останавливается. Живость в обоих случаях должна оставаться «да», иначе платформа убьёт копию, которая просто занята запуском или завершением работы.

Замените строку `use std::time::Duration;` в начале `src/lib.rs` на эти строки:

```rust
use std::sync::Arc;
use std::sync::atomic::{AtomicBool, Ordering};
use std::time::Duration;

use axum::Router;
use axum::extract::State;
use axum::http::StatusCode;
use axum::routing::get;
```

и допишите часть про работоспособность:

```rust
/// Shared state: is this copy of the service ready to receive traffic?
#[derive(Clone, Default)]
pub struct Health {
    ready: Arc<AtomicBool>,
}

impl Health {
    pub fn set_ready(&self, ready: bool) {
        self.ready.store(ready, Ordering::SeqCst);
    }

    pub fn is_ready(&self) -> bool {
        self.ready.load(Ordering::SeqCst)
    }
}

/// Liveness: "the process is alive and can answer". Never checks other services.
async fn livez() -> &'static str {
    "alive\n"
}

/// Readiness: "send me traffic". False while starting and while shutting down.
async fn readyz(State(health): State<Health>) -> (StatusCode, &'static str) {
    if health.is_ready() {
        (StatusCode::OK, "ready\n")
    } else {
        (StatusCode::SERVICE_UNAVAILABLE, "not ready\n")
    }
}

/// A slow request, to watch what happens to work that is in flight during shutdown.
async fn slow() -> &'static str {
    tokio::time::sleep(Duration::from_secs(3)).await;
    "slow request finished\n"
}

pub fn app(health: Health) -> Router {
    Router::new()
        .route("/livez", get(livez))
        .route("/readyz", get(readyz))
        .route("/slow", get(slow))
        .with_state(health)
}
```

Создайте `src/main.rs`. Он использует сигналы Unix, поэтому работает в macOS и Linux:

```rust
use std::process::ExitCode;

use p20_ready::{Config, Health, app};
use tokio::signal::unix::{SignalKind, signal};

/// Waits for Ctrl+C (SIGINT) or SIGTERM, the signal that service managers send.
async fn stop_signal() {
    let mut term = signal(SignalKind::terminate()).expect("cannot listen for SIGTERM");
    tokio::select! {
        _ = tokio::signal::ctrl_c() => {}
        _ = term.recv() => {}
    }
}

#[tokio::main]
async fn main() -> ExitCode {
    // 1. Check the whole configuration before doing anything else.
    let config = match Config::from_lookup(|name| std::env::var(name).ok()) {
        Ok(config) => config,
        Err(errors) => {
            for error in errors {
                eprintln!("config error: {error}");
            }
            return ExitCode::from(2);
        }
    };

    let health = Health::default(); // not ready yet
    let listener = tokio::net::TcpListener::bind(("127.0.0.1", config.port))
        .await
        .unwrap();
    // A real service would connect to its database and warm its caches here.
    health.set_ready(true);
    println!("listening on 127.0.0.1:{}, ready", config.port);

    // 2. On a signal: report "not ready", wait for the load balancer to notice, then stop.
    let drain = config.drain;
    let draining = health.clone();
    let shutdown = async move {
        stop_signal().await;
        draining.set_ready(false);
        println!("signal: not ready any more, draining for {drain:?}");
        tokio::time::sleep(drain).await;
        println!("drain over: no new connections, finishing open requests");
    };

    axum::serve(listener, app(health))
        .with_graceful_shutdown(shutdown)
        .await
        .unwrap();
    println!("all requests finished, exiting");
    ExitCode::SUCCESS
}
```

Сначала — быстрый отказ. Запустите программу с двумя плохими настройками:

```bash
PORT=eighty DRAIN_SECONDS=600 cargo run -q
echo "exit code: $?"
```

```text
config error: PORT must be a number from 1 to 65535, got "eighty"
config error: DRAIN_SECONDS must be 0 to 60, got "600"
exit code: 2
```

Теперь остановка. Этот скрипт запускает сервис, начинает медленный запрос, отправляет SIGTERM и наблюдает за эндпоинтами. Сохраните его как `shutdown-demo.sh` и сделайте исполняемым:

```bash
#!/usr/bin/env bash
# Starts the service, stops it with SIGTERM, and watches the health endpoints.
set -u
status() { curl -s -o /dev/null -w '%{http_code}' "http://127.0.0.1:3000$1"; }

./target/release/p20-ready &
pid=$!
sleep 0.5
echo "readyz: $(status /readyz)"
curl -s http://127.0.0.1:3000/slow > slow.txt &   # a request that needs 3 seconds
sleep 0.2
kill -TERM "$pid"
sleep 0.3
echo "readyz: $(status /readyz)   livez: $(status /livez)"
wait "$pid"
echo "exit code: $?"
echo "the slow request got: $(cat slow.txt)"
echo "readyz: $(status /readyz)"
```

`cargo build --release`, затем `./shutdown-demo.sh`:

```text
listening on 127.0.0.1:3000, ready
readyz: 200
signal: not ready any more, draining for 2s
readyz: 503   livez: 200
drain over: no new connections, finishing open requests
all requests finished, exiting
exit code: 0
the slow request got: slow request finished
readyz: 000
```

**Что происходит:**

1. До сигнала `/readyz` отвечает `200`. Сервис объявил себя готовым только **после** того, как слушающий сокет был привязан (а в настоящем сервисе — после того, как заработало соединение с базой данных).
2. `kill -TERM` отправляет SIGTERM. Future остановки сразу выставляет готовность в false. Во время отвода трафика `/readyz` отвечает `503`, а `/livez` по-прежнему `200`: «не шлите мне новую работу, но и не убивайте меня».
3. Зачем вообще ждать? Балансировщик нагрузки спрашивает `/readyz` только раз в несколько секунд. Пока он не заметил `503`, он продолжает слать запросы. Если бы сервис сразу перестал принимать соединения, эти запросы упали бы. Время отвода (здесь 2 с) должно быть чуть больше интервала между двумя проверками готовности.
4. После отвода `with_graceful_shutdown` перестаёт принимать новые соединения и ждёт открытые запросы. Медленный запрос, начатый до сигнала, всё равно получает полный ответ.
5. Процесс завершается с кодом `0`, так что менеджер служб знает, что остановка была чистой. Последний `000` означает, что curl вообще не смог подключиться: процесса больше нет.

> ⚠️ **Частая ошибка:** проверять базу данных в **живости**. Когда база недоступна, каждая копия проваливает проверку живости и перезапускается снова и снова. Базу это не чинит, а только добавляет перезапуски поверх отказа. Живость проверяет только сам процесс.

> 💡 **Совет:** здесь у отвода трафика нет верхнего предела для открытых запросов. Настоящему сервису нужен ещё и **жёсткий лимит** на всю остановку (Урок 14.14), а собственный льготный период платформы (например, 30 с в Kubernetes) должен быть больше, чем отвод плюс этот лимит.

## 5. Пусть машина проверит то, что может

Многие пункты из групп A, B и G можно проверить командами. Небольшой скрипт превращает их в список строк PASS и FAIL. Сохраните его как `readiness-check.sh`:

```bash
#!/usr/bin/env bash
# Runs the checklist items that a machine can check. Exit code 1 if any of them fails.
set -u
failed=0

check() {
    local name="$1"
    shift
    if "$@" > /dev/null 2>&1; then
        echo "PASS  $name"
    else
        echo "FAIL  $name   (run: $*)"
        failed=1
    fi
}

check "code is formatted"         cargo fmt --check
check "no Clippy warnings"        cargo clippy --all-targets -- -D warnings
check "no unwrap outside tests"   cargo clippy --lib --bins -- -D clippy::unwrap_used
check "all tests pass"            cargo test
check "no known vulnerabilities"  cargo audit
check "lock file exists"          test -f Cargo.lock
exit "$failed"
```

```bash
chmod +x readiness-check.sh
./readiness-check.sh
echo "exit code: $?"
```

```text
PASS  code is formatted
PASS  no Clippy warnings
FAIL  no unwrap outside tests   (run: cargo clippy --lib --bins -- -D clippy::unwrap_used)
PASS  all tests pass
PASS  no known vulnerabilities
PASS  lock file exists
exit code: 1
```

**Что происходит:**

1. `check` принимает имя и команду (`"$@"` — все оставшиеся аргументы), выполняет команду со скрытым выводом и печатает PASS или FAIL (с командой, чтобы вы могли её запустить и прочитать подробности). Код возврата — `1`, если что-то не прошло, так что CI может запускать скрипт одним шагом.
2. `cargo clippy --lib --bins -- -D clippy::unwrap_used` превращает линт `unwrap_used` из Урока 20.2 в ошибку для библиотеки и программы, но не для тестов. Он нашёл два вызова `.unwrap()` в `main.rs`. Вы уберёте их в разделе «Попробуйте сами».

Скрипт не заменяет проверку людьми. Он не видит, разумен ли тайм-аут, полезен ли ранбук и проверялось ли восстановление. Он убирает скучную часть, чтобы проверка тратила время на эти вопросы.

## Частые ошибки

**1. Чек-лист без доказательств.** Галочки по памяти. Просите тест, команду или ссылку.

**2. Готовность, которая всегда `200`.** Тогда балансировщик нагрузки шлёт трафик копиям, которые запускаются или останавливаются, и каждое развёртывание вызывает короткий всплеск ошибок.

**3. Немедленный выход по SIGTERM.** Открытые запросы обрываются, а запросы, которые уже в пути, падают. Сначала отведите трафик, потом завершайте.

**4. Ленивое чтение конфигурации.** Опечатка в настройке, которая впервые используется в 3 часа ночи. Проверяйте всё при запуске.

## Попробуйте сами

1. Добейтесь, чтобы `./readiness-check.sh` проходил: в `main.rs` замените два вызова `.unwrap()` на `match` (или `if let Err(error) = …`), который печатает ошибку через `eprintln!` и возвращает `ExitCode::FAILURE`. Затем запустите сервис дважды; вторая копия должна напечатать `cannot listen on port 3000: Address already in use (os error 48)` (в Linux номер ошибки — 98) и завершиться с кодом `1`.
2. Запустите `DRAIN_SECONDS=0 ./shutdown-demo.sh`. Что меняется в выводе и почему `0` — рискованное значение в продакшене?
3. Заполните группы E и F чек-листа для `p20-ready` с доказательствами. Каких пунктов ещё не хватает?

## Упражнения

### Упражнение 1 — Какого пункта не хватало? (легко)

Для каждого инцидента назовите пункт чек-листа (группу и пункт), который скорее всего предотвратил бы его.

1. Каждое развёртывание вызывает около 30 секунд ошибок `502`.
2. Платёжный провайдер зависает; через две минуты у сервиса 10 000 открытых запросов, и у него кончается память.
3. После обновления сервис работает шесть часов, а потом падает, когда ночное задание читает настройку с опечаткой.
4. Отказывает диск базы данных. Резервные копии есть, но никто не знает, как из них восстановиться, и восстановление занимает два дня.

<details>
<summary>Подсказка</summary>

Посмотрите на группы B, C, E, F и H.

</details>

<details>
<summary>Ответ</summary>

1. **E:** готовность и отвод трафика по SIGTERM. Копии перестают принимать работу раньше, чем об этом узнаёт балансировщик нагрузки.
2. **B:** тайм-аут на каждый исходящий вызов, и **C:** лимит конкурентности со сбросом нагрузки. Любой из них удержал бы число открытых запросов небольшим.
3. **F:** вся конфигурация проверяется при запуске. Опечатка сразу остановила бы развёртывание.
4. **H:** резервные копии существуют **и восстановление проверено**. Проверенное восстановление к тому же показывает, сколько оно длится.

Почему это работает: большинство инцидентов не экзотичны. Они попадают в несколько групп, а чек-лист — это память об этих группах.

</details>

### Упражнение 2 — Протестировать эндпоинты работоспособности (средне)

Напишите интеграционный тест `tests/health.rs`, который без сети проверяет: `/readyz` возвращает `503` до `set_ready(true)`, `200` после него и снова `503` после `set_ready(false)`; `/livez` всё время возвращает `200`.

<details>
<summary>Подсказка</summary>

Напишите небольшую вспомогательную функцию `async fn status(health: &Health, path: &str) -> StatusCode`, которая строит `app(health.clone())` и отправляет один запрос через `oneshot`. Все клоны `Health` делят один и тот же `Arc<AtomicBool>`.

</details>

<details>
<summary>Ответ</summary>

```rust
use axum::body::Body;
use axum::http::{Request, StatusCode};
use p20_ready::{Health, app};
use tower::ServiceExt;

async fn status(health: &Health, path: &str) -> StatusCode {
    let request = Request::get(path).body(Body::empty()).unwrap();
    app(health.clone()).oneshot(request).await.unwrap().status()
}

#[tokio::test]
async fn readiness_follows_the_flag_and_liveness_stays_ok() {
    let health = Health::default();
    assert_eq!(
        status(&health, "/readyz").await,
        StatusCode::SERVICE_UNAVAILABLE
    );
    assert_eq!(status(&health, "/livez").await, StatusCode::OK);

    health.set_ready(true);
    assert_eq!(status(&health, "/readyz").await, StatusCode::OK);

    health.set_ready(false); // shutting down
    assert_eq!(
        status(&health, "/readyz").await,
        StatusCode::SERVICE_UNAVAILABLE
    );
    assert_eq!(status(&health, "/livez").await, StatusCode::OK);
}
```

`cargo test --test health`:

```text
test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

Почему это работает: тест строит новый маршрутизатор для каждого запроса, но все маршрутизаторы делят один флаг, потому что `Health` хранит `Arc`. Именно так работает и настоящий сервис: future остановки и обработчики держат клоны одного `Health`. Этот тест — доказательство для пункта «эндпоинты живости и готовности с разным смыслом».

</details>

### Упражнение 3 — Доступна ли база данных? (сложно)

Проверка готовности иногда спрашивает и «доступна ли моя база данных?». Напишите в `src/lib.rs` публичную функцию `can_connect(address: &str) -> bool`, которая возвращает `true`, только если TCP-соединение (Transmission Control Protocol, протокол управления передачей) с `address` удаётся открыть за 200 мс. Напишите тест с открытым и закрытым портом, без настоящей базы данных. Затем объясните, как бы вы использовали её в `readyz` и в чём опасность.

<details>
<summary>Подсказка</summary>

`tokio::time::timeout(Duration::from_millis(200), tokio::net::TcpStream::connect(address))` даёт `Ok(Ok(stream))` только при успехе. Для тестового порта привяжите `TcpListener` к `127.0.0.1:0` (система выберет свободный порт), прочитайте `local_addr()` и сделайте `drop` слушателя, чтобы снова закрыть порт.

</details>

<details>
<summary>Ответ</summary>

В `src/lib.rs`:

```rust
/// Can we open a TCP connection to `address` within 200 ms?
pub async fn can_connect(address: &str) -> bool {
    let attempt = tokio::net::TcpStream::connect(address);
    matches!(
        tokio::time::timeout(Duration::from_millis(200), attempt).await,
        Ok(Ok(_))
    )
}
```

`tests/dependency.rs`:

```rust
use p20_ready::can_connect;

#[tokio::test]
async fn sees_an_open_port_and_a_closed_one() {
    let listener = tokio::net::TcpListener::bind("127.0.0.1:0").await.unwrap();
    let address = listener.local_addr().unwrap().to_string();
    assert!(can_connect(&address).await);

    drop(listener); // nobody listens there any more
    assert!(!can_connect(&address).await);
}
```

`cargo test --test dependency` проходит.

Почему это работает: операционная система сразу отвечает на соединение с закрытым локальным портом отказом («connection refused»), а открытый порт его принимает, так что тест быстрый и не требует базы данных. Тайм-аут 200 мс держит `/readyz` быстрым, даже если база зависла. Чтобы использовать функцию, добавьте в `Health` поле `database: Option<String>` и пусть `readyz` отвечает `200`, только если `is_ready()` **и** `can_connect(address).await` (или адреса нет). Опасность: если **каждая** копия зависит от одной базы данных, при её отказе все они разом становятся «не готовы», и балансировщику нагрузки некуда слать трафик — даже запросы, которым база не нужна. Проверяйте только те зависимости, без которых сервис не может сделать ничего полезного, и никогда не проверяйте их в живости.

</details>

## Вопросы для самопроверки

1. Что делает пункт чек-листа «выполненным» на проверке готовности к продакшену?

<details>
<summary>Ответ</summary>

Доказательство: тест, команда с выводом, дашборд или документ, который это подтверждает. Или «не нужно» с письменной причиной.

</details>

2. Чем живость отличается от готовности и что делает платформа, когда не проходит каждая из них?

<details>
<summary>Ответ</summary>

Живость спрашивает «жив ли процесс?»; если нет, процесс перезапускают. Готовность спрашивает «слать ли ему трафик сейчас?»; если нет, балансировщик нагрузки перестаёт слать запросы, но процесс продолжает работать.

</details>

3. Почему сервис после SIGTERM ждёт (отводит трафик), прежде чем перестать принимать соединения?

<details>
<summary>Ответ</summary>

Балансировщик нагрузки проверяет готовность только раз в несколько секунд. Пока он не увидел `503`, он продолжает слать новые запросы. Если подождать чуть дольше этого интервала, ни один запрос не придёт в копию, которая уже отказывается от соединений.

</details>

4. Почему проверка конфигурации должна сообщать обо всех проблемах сразу?

<details>
<summary>Ответ</summary>

Чтобы тот, кто исправляет конфигурацию, увидел всё за один запуск. Если на каждый запуск по одной ошибке, пять опечаток потребуют пяти развёртываний.

</details>

5. Почему проверка живости не должна проверять базу данных?

<details>
<summary>Ответ</summary>

Когда база недоступна, каждая копия провалила бы проверку живости и перезапускалась бы снова и снова. Перезапуски базу не чинят; они только добавляют сбоев и нагрузки.

</details>

## Итоги

- Проверка готовности к продакшену проходит по чек-листу перед запуском и после больших изменений; каждому пункту нужно доказательство или письменное «не нужно».
- Группы чек-листа: корректность, обработка сбоев, лимиты, наблюдаемость, работоспособность и жизненный цикл, конфигурация и секреты, безопасность и цепочка поставок, развёртывание и восстановление, эксплуатация.
- Проверяйте всю конфигурацию при запуске, сообщайте о каждой проблеме и завершайтесь с ненулевым кодом.
- Живость означает «жив», готовность — «шлите мне трафик»; это должны быть разные эндпоинты с разными проверками.
- По SIGTERM: стать «не готов», отвести трафик, перестать принимать соединения, завершить открытые запросы, выйти с `0`, с жёстким лимитом на всю остановку.
- Пусть скрипт проверяет то, что может проверить машина, а проверка людьми тратит время на вопросы, где нужно суждение.

## Источники и что почитать

- SRE (Site Reliability Engineering, инженерия надёжности сайтов): [Google SRE book: Launch Coordination Checklist](https://sre.google/sre-book/launch-checklist/) — чек-лист, из которого выросли проверки запусков в Google.
- [Kubernetes: liveness, readiness and startup probes](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/) — как платформа использует два эндпоинта.
- [Kubernetes: pod termination](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/) — SIGTERM, льготный период и что происходит с трафиком при остановке.
- [axum: `serve` и корректное завершение](https://docs.rs/axum/latest/axum/serve/struct.Serve.html) — `with_graceful_shutdown`.
- [tokio: сигналы Unix](https://docs.rs/tokio/latest/tokio/signal/unix/index.html) — как слушать SIGTERM.
- [The Twelve-Factor App: Config](https://12factor.net/config) — конфигурация в окружении, отдельно от кода.

---

[🏠 Начало](../START-HERE.md) · [← 20.8 Перегрузка, лимиты и обратное давление](08-overload-limits-and-backpressure.md) · [📋 Этап 20](README.md) · [20.10 Проект: укрепить сервис →](10-project-harden-a-service.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
