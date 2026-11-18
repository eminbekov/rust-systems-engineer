<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 11.15 Проект: развёртывание сервиса на Rust](15-project-deploy-a-rust-service.md) · [📋 Этап 11](README.md) · [Этап 12 — Компьютерные системы →](../12-computer-systems/README.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/11-linux-shell-and-containers/16-checkpoint-stage-2-complete.md) <!-- nav -->

# Урок 11.16 — Контрольная точка: стадия 2 пройдена

> **Этап 11 — Linux, командная оболочка и контейнеры** · Урок 16 из 16 · ⏱ около 2,5 часа (2 занятия)

## Что вы узнаете

- Владеете ли вы материалом Стадии 2 (Этапы 07–11): алгоритмы, проектирование ПО, тестирование, безопасность, Linux и контейнеры.
- Как соединять несколько из этих навыков в практических заданиях: утилита на shell и утилита на Rust, которые должны совпадать, тест свойств, исправление уязвимости и ремонт сломанного развёртывания.
- Какие уроки повторить, с прямыми ссылками.
- Чего Стадия 3 ждёт от вас.

## Перед началом

- Вы прошли [Урок 11.15 — Проект: развёртывание сервиса на Rust](15-project-deploy-a-rust-service.md) и предыдущие этапы Стадии 2.
- ВМ `dev` и Docker Desktop запущены. Некоторые задания используют обе.
- Отведите около 2 часов. Сначала пройдите тест **не подглядывая в уроки**; запишите ответы, затем откройте скрытые ответы. Неверный ответ показывает, что повторить.
- Команды с пометкой «в ВМ» выполняются внутри `limactl shell dev`; остальные — на Mac. Код хранится в `~/projects/rust/code/p11-checkpoint`.

## Новые слова

| Слово | Значение |
|---|---|
| **контрольная точка** (checkpoint) | Урок, который проверяет, чему вы научились, вместо того чтобы учить новому. |
| **свойство** (property) | Утверждение, которое должно быть верно для **всех** входов определённого вида, например «поиск в отсортированном списке находит каждый элемент, который в нём есть». Тест свойств проверяет его на многих случайных входах. |
| **регрессионный тест** (regression test) | Тест, который не даёт исправленной ошибке вернуться. |
| **поиск причин неисправности** (troubleshooting) | Пошаговое выяснение причины сбоя системы, от симптома до первой неправильной вещи. |

## Стадия 2 в одной таблице

| Этап | Чему вы научились | Ваш главный проект |
|---|---|---|
| 07 Алгоритмы и структуры данных | Big-O, массивы и векторы, связные списки, стеки, очереди, хеш-таблицы, рекурсия, сортировка, двоичный поиск, деревья, кучи, графы, динамическое программирование | Поиск маршрута |
| 08 Проектирование ПО | KISS, DRY, YAGNI, связность и сопряжение, SOLID, композиция, проектирование на типах, builder, RAII, typestate, слои, двенадцать факторов | Рефакторинг запутанной программы |
| 09 Тестирование, отладка и бенчмарки | Пирамида тестов, тестовые двойники, тесты свойств и снапшот-тесты, TDD, покрытие и мутационные тесты, фаззинг, отладчики, логирование, бенчмарки, CI | Хорошо протестированная библиотека с CI |
| 10 Основы безопасности | CIA, STRIDE, OWASP Top 10, проверка ввода, внедрение, пароли, сессии, секреты, криптография, TLS, цепочка поставок | Хранилище зашифрованных секретов |
| 11 Linux, командная оболочка и контейнеры | ВМ, файловая система, конвейеры, пользователи и права, процессы и сигналы, systemd, apt, bash, SSH, сеть, Docker, Compose | Развёртывание сервиса на Rust |

Главная мысль стадии: **программа — это больше, чем её код**. У неё есть цена (Big-O), форма (проектирование), доказательство, что она работает (тесты), защита (безопасность) и место, где она работает (Linux, контейнеры).

## Вопросы для самопроверки

Двадцать вопросов. Отвечайте на каждый одним-двумя предложениями, прежде чем открывать ответ.

### Этап 07 — Алгоритмы и структуры данных

1. Двоичному поиску нужно одно условие на вход, после чего он делает O(log n) шагов. Какое условие и почему работа сокращается так быстро?

<details>
<summary>Ответ</summary>

Вход должен быть **отсортирован**. Каждое сравнение со средним элементом отбрасывает половину оставшегося диапазона, поэтому для n элементов нужно лишь около log₂ n шагов (для 1 000 000 элементов — около 20).

</details>

2. Поиск в `HashMap` — O(1) в среднем. Когда он медленнее и насколько?

<details>
<summary>Ответ</summary>

Когда много ключей попадают в одну корзину (коллизии), поиск вынужден сравнивать ключи по очереди. В худшем случае, когда все ключи в одной корзине, это O(n). Хорошая хеш-функция и увеличение таблицы при высоком коэффициенте заполнения удерживают среднее на уровне O(1).

</details>

3. Какой поиск в графе находит кратчайший путь (с наименьшим числом рёбер) в невзвешенном графе и какую структуру данных он использует?

<details>
<summary>Ответ</summary>

Поиск в ширину (BFS). Он использует очередь и посещает узлы в порядке удаления от начала, поэтому первый раз он достигает узла по кратчайшему пути. Поиск в глубину использует стек (или рекурсию) и такой гарантии не даёт.

</details>

4. Что такое базовый случай рекурсивной функции и что происходит, если его нет?

<details>
<summary>Ответ</summary>

Базовый случай — это вход, для которого функция отвечает напрямую, не вызывая себя снова. Без него (или если вход никогда не доходит до него) вызовы накапливаются в стеке вызовов, пока он не переполнится и программа не упадёт.

</details>

### Этап 08 — Проектирование ПО

5. Сформулируйте KISS, DRY и YAGNI одним предложением каждый.

<details>
<summary>Ответ</summary>

KISS: делайте проще, выбирайте самое простое решение, которое работает. DRY: не повторяйтесь, у каждого знания одно место. YAGNI: вам это не понадобится, не создавайте возможности раньше, чем они нужны.

</details>

6. В Rust нет наследования классов. Что используется вместо него и почему это предпочтительнее?

<details>
<summary>Ответ</summary>

Композиция (структура хранит другие значения) и трейты (общее поведение). Это слабо связывает типы: вы собираете маленькие части и заменяете их, а не наследуете целое дерево поведения, которое нельзя изменить.

</details>

7. Какое правило двенадцати факторов говорит, где живёт конфигурация, и как ей следовали службы Стадии 2?

<details>
<summary>Ответ</summary>

Фактор III: храните конфигурацию в **окружении**, а не в коде. `p11-hello` читает `BIND` и `GREETING` из переменных окружения, а Compose или systemd задаёт их для каждого развёртывания.

</details>

### Этап 09 — Тестирование, отладка и бенчмарки

8. Опишите пирамиду тестов и почему у неё такая форма.

<details>
<summary>Ответ</summary>

Внизу много быстрых маленьких модульных тестов; в середине меньше интеграционных; наверху немного медленных сквозных. Нижние тесты дёшевы и точно указывают место сбоя; верхние медленны, хрупки и говорят лишь о том, что что-то не так.

</details>

9. Что делает тест на основе свойств такого, чего не делает тест с примерами?

<details>
<summary>Ответ</summary>

Он формулирует правило, которое должно выполняться для всех входов, и проверяет его на многих сгенерированных входах, а затем сокращает упавший вход до небольшого контрпримера. Он находит случаи, о которых автор не подумал.

</details>

10. В наборе тестов 100% покрытия строк. Доказывает ли это, что тесты хороши? Что скажет больше?

<details>
<summary>Ответ</summary>

Нет: покрытие показывает только, что строки выполнялись, а не что тесты заметили бы ошибку. Мутационное тестирование меняет код (`<` становится `<=`) и проверяет, что какой-то тест падает; выжившие мутанты показывают слабые тесты.

</details>

11. Почему службы пишут журнал в стандартный вывод с уровнями, а не ведут собственные файлы?

<details>
<summary>Ответ</summary>

Платформа (журнал systemd, драйвер журналов Docker) собирает, ротирует и направляет стандартный вывод. Уровни (`error`, `warn`, `info`, `debug`) позволяют выбирать, сколько видеть, не меняя код.

</details>

### Этап 10 — Основы безопасности

12. Что означают буквы C, I и A в триаде CIA?

<details>
<summary>Ответ</summary>

Confidentiality (конфиденциальность: данные читают только те, кому можно), Integrity (целостность: данные не меняют без разрешения), Availability (доступность: система работает, когда нужна).

</details>

13. Какова первопричина атаки внедрения и каково лекарство?

<details>
<summary>Ответ</summary>

Недоверенный ввод смешивается с командой или запросом и интерпретируется как код (SQL, команда оболочки, путь). Лекарство — разделять данные и код: параметризованные запросы, никакого `eval`, передача аргументов отдельными значениями и проверка ввода по списку разрешённого.

</details>

14. Как сервер должен хранить пароли пользователей?

<details>
<summary>Ответ</summary>

Только как хеш, полученный медленной функцией хеширования паролей с уникальной случайной солью для каждого пользователя (например, Argon2), никогда открытым текстом и никогда быстрым хешем вроде простого SHA-256.

</details>

15. Пароль закоммитили в Git и отправили. Что делать в первую очередь?

<details>
<summary>Ответ</summary>

**Немедленно отозвать или сменить секрет** (изменить пароль, выпустить новый ключ). Удаления коммита из истории недостаточно, потому что копии могут существовать. Затем при необходимости очистить историю и выяснить, как это случилось.

</details>

### Этап 11 — Linux, командная оболочка и контейнеры

16. Что делают три части `set -euo pipefail`?

<details>
<summary>Ответ</summary>

`-e` останавливает скрипт, когда команда не срабатывает; `-u` делает неопределённую переменную ошибкой; `-o pipefail` заставляет конвейер завершаться ошибкой, если ошибкой завершилась любая его команда, а не только последняя.

</details>

17. У файла режим `640`, он принадлежит `root:app`. Кто что может?

<details>
<summary>Ответ</summary>

Владелец `root`: чтение и запись (6). Группа `app`: только чтение (4). Все остальные: ничего (0). Служба, работающая от пользователя из группы `app`, может прочитать файл, остальные — нет.

</details>

18. Для входа по ключу через SSH: какой ключ попадает на сервер, какой остаётся у вас и где сервер его хранит?

<details>
<summary>Ответ</summary>

Открытый ключ попадает на сервер, в `~/.ssh/authorized_keys` того пользователя, от имени которого вы входите. Закрытый ключ (режим `600`, лучше с парольной фразой) никогда не покидает ваш компьютер.

</details>

19. Почему `CMD` в Dockerfile для службы, которая должна корректно останавливаться, должен быть в exec-форме?

<details>
<summary>Ответ</summary>

В exec-форме (`CMD ["/app"]`) программа — PID 1 и получает `SIGTERM` от `docker stop`. В shell-форме PID 1 — это `/bin/sh -c`, он не передаёт сигнал дальше, поэтому Docker ждёт льготный период и убивает программу.

</details>

20. Контейнер публикует `-p 8080:8080`, но `curl` получает «connection reset». Сервер печатает `listening on 127.0.0.1:8080`. Что не так?

<details>
<summary>Ответ</summary>

Он слушает петлю контейнера. Опубликованный трафик приходит на сетевой интерфейс контейнера, поэтому никто его не слышит. Привяжитесь к `0.0.0.0:8080` внутри контейнера, а доступ ограничьте способом публикации (`-p 127.0.0.1:8080:8080`).

</details>

## Практические задания

Создайте `~/projects/rust/code/p11-checkpoint` с отдельной папкой на каждое задание. Делайте коммит после каждого задания.

### Задание 1 — Один ответ от двух инструментов

**Цель:** конвейер shell и программа на Rust, считающие слова, должны печатать абсолютно одинаковый список лидеров, а скрипт это доказывает. Здесь сходятся хеш-таблицы (Этап 07), тесты (Этап 09) и конвейеры и надёжные скрипты (Этап 11).

Напишите в ВМ:

1. `topwords.sh [N]`: читает текст из стандартного ввода и печатает `N` самых частых слов (по умолчанию 3) с их количеством, самые частые первыми, при равенстве — по алфавиту. Слова — это буквы и цифры, регистр не важен. Используйте `tr`, `sort`, `uniq -c`, `head`; строгий режим; задайте `LC_ALL=C`; `N` должно быть положительным числом, иначе сообщение в стандартный поток ошибок и код 2.
2. Программу на Rust `p11-topwords` с тем же поведением, использующую `HashMap`. Вынесите логику в функцию `top_words(text, n)` и напишите не менее трёх модульных тестов. Печатайте через `{count:>7} {word}`, чтобы вывод выглядел как вывод `uniq -c` в Linux.
3. `compare.sh FILE`, который запускает обе для `N` = 1, 3 и 10 и сообщает `ok` или `FAIL`.

<details>
<summary>Подсказка</summary>

Shell: `tr -cs '[:alnum:]' '\n' | tr '[:upper:]' '[:lower:]' | sed '/^$/d' | sort | uniq -c | sort -k1,1nr -k2,2 | head -n "$n"`. Rust: разбивайте через `text.split(|c: char| !c.is_ascii_alphanumeric())`, считайте через `*counts.entry(word.to_ascii_lowercase()).or_insert(0) += 1`, затем сортируйте `Vec` по `b.1.cmp(&a.1).then_with(|| a.0.cmp(&b.0))`. Соберите программу на Rust в ВМ (`cargo build --release`) или в образе `rust:1.100`, если хотите. В скрипте используйте `diff <(…) <(…)`.

</details>

<details>
<summary>Ответ</summary>

`topwords.sh`:

```bash
#!/usr/bin/env bash
# topwords.sh [N] - read text from standard input, print the N most frequent words.
set -euo pipefail
export LC_ALL=C

n=${1:-3}
[[ $n =~ ^[1-9][0-9]*$ ]] || { echo "topwords.sh: N must be a positive number, got '$n'" >&2; exit 2; }

tr -cs '[:alnum:]' '\n' | tr '[:upper:]' '[:lower:]' | sed '/^$/d' \
    | sort | uniq -c | sort -k1,1nr -k2,2 | head -n "$n"
```

`topwords/src/main.rs` (пакет без зависимостей; назовите его `p11-topwords`):

```rust
use std::collections::HashMap;
use std::io::Read;
use std::process::ExitCode;

/// Counts the words of `text` (letters and digits, case-insensitive) and returns
/// the `n` most frequent ones: highest count first, ties in alphabetical order.
fn top_words(text: &str, n: usize) -> Vec<(String, usize)> {
    let mut counts: HashMap<String, usize> = HashMap::new();
    for word in text.split(|c: char| !c.is_ascii_alphanumeric()) {
        if !word.is_empty() {
            *counts.entry(word.to_ascii_lowercase()).or_insert(0) += 1;
        }
    }
    let mut list: Vec<(String, usize)> = counts.into_iter().collect();
    list.sort_by(|a, b| b.1.cmp(&a.1).then_with(|| a.0.cmp(&b.0)));
    list.truncate(n);
    list
}

fn main() -> ExitCode {
    let n = match std::env::args().nth(1) {
        None => 3,
        Some(text) => match text.parse::<usize>() {
            Ok(n) if n > 0 => n,
            _ => {
                eprintln!("topwords: N must be a positive number, got '{text}'");
                return ExitCode::from(2);
            }
        },
    };

    let mut input = String::new();
    if let Err(error) = std::io::stdin().read_to_string(&mut input) {
        eprintln!("topwords: cannot read standard input: {error}");
        return ExitCode::FAILURE;
    }
    for (word, count) in top_words(&input, n) {
        println!("{count:>7} {word}");
    }
    ExitCode::SUCCESS
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn counts_ignoring_case_and_punctuation() {
        let result = top_words("The cat. THE dog! the end", 2);
        assert_eq!(result, vec![("the".to_string(), 3), ("cat".to_string(), 1)]);
    }

    #[test]
    fn ties_are_sorted_alphabetically() {
        let result = top_words("b a c", 3);
        let words: Vec<&str> = result.iter().map(|(w, _)| w.as_str()).collect();
        assert_eq!(words, ["a", "b", "c"]);
    }

    #[test]
    fn empty_text_has_no_words() {
        assert!(top_words("", 3).is_empty());
    }
}
```

`compare.sh` (бинарный файл Rust ожидается в `topwords/target/release/p11-topwords`; поправьте `rust_bin`, если собираете в другом месте):

```bash
#!/usr/bin/env bash
# compare.sh FILE - the shell pipeline and the Rust program must print the same thing.
set -euo pipefail
here=$(cd "$(dirname "$0")" && pwd)
file=${1:?usage: compare.sh FILE}
rust_bin=$here/topwords/target/release/p11-topwords

for n in 1 3 10; do
    if diff <("$here/topwords.sh" "$n" < "$file") <("$rust_bin" "$n" < "$file") > /dev/null; then
        echo "ok   - top $n words are the same"
    else
        echo "FAIL - top $n words differ"
        exit 1
    fi
done
```

С небольшим файлом `sample.txt` из этих двух строк:

```text
The cat sat on the mat. The dog sat on the log!
A cat and a dog: the best of friends, the best of days.
```

вывод `bash topwords.sh 4 < sample.txt` такой:

```text
      6 the
      2 a
      2 best
      2 cat
```

а `cargo test` и `compare.sh` печатают:

```text
test result: ok. 3 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
ok   - top 1 words are the same
ok   - top 3 words are the same
ok   - top 10 words are the same
```

Попробуйте также `bash compare.sh /usr/share/common-licenses/GPL-3` (более длинный текст): те же три строки `ok`, а `bash topwords.sh 3 < /usr/share/common-licenses/GPL-3` начинается с `345 the`.

Почему это работает: оба инструмента задают одно правило (счёт, затем алфавит) и один формат, поэтому побайтовый `diff` — сильный тест. `LC_ALL=C` заставляет `sort` использовать порядок байтов, как это делает сравнение строк в Rust; без него порядок равных значений может зависеть от локали. Версия на Rust считает в `HashMap` за O(n) и сортирует только различные слова; конвейер сортирует каждое слово.

</details>

### Задание 2 — Почините сломанное развёртывание

**Цель:** найти четыре ошибки в Dockerfile и файле Compose по их симптомам. Здесь сходятся сеть, контейнеры, Compose и привычки безопасности.

Возьмите службу `p11-hello` (Урок 11.13) и используйте этот `Dockerfile` и `compose.yaml` в копии проекта:

```dockerfile
FROM rust:1.100 AS build
WORKDIR /app
COPY . .
RUN cargo build --release

FROM debian:trixie-slim
COPY --from=build /app/target/release/p11-hello /usr/local/bin/p11-hello
ENV BIND=127.0.0.1:8080
EXPOSE 8080
CMD /usr/local/bin/p11-hello
```

```yaml
name: course-broken

services:
  app:
    build: .
    image: course-broken:1
    ports:
      - "8080:8080"
    restart: always
```

Поместите `Dockerfile`, `compose.yaml` и `target` в `.dockerignore`. Выполните `docker compose up -d --build` и соберите симптомы:

```bash
docker compose ps --format '{{.Ports}}'
curl -sS -m 3 http://localhost:8080/; echo "curl status $?"
docker compose exec app id
SECONDS=0; docker compose stop app; echo "took ${SECONDS}s"
docker inspect course-broken-app-1 --format 'exit={{.State.ExitCode}}'
```

Объясните каждый симптом, исправьте файлы и покажите, что симптомы исчезли. Потом всё уберите (`docker compose down`, `docker image rm course-broken:1`).

<details>
<summary>Подсказка</summary>

Находок четыре, по одной на каждую команду выше. Подумайте: об адресе внутри контейнера; о том, кому разрешено достучаться до опубликованного порта; о пользователе; и о форме `CMD` (сравните время остановки и код возврата с Уроком 11.13). Пятое, менее важное замечание — порядок `COPY` для кэша сборки.

</details>

<details>
<summary>Ответ</summary>

Симптомы сломанной версии:

```text
0.0.0.0:8080->8080/tcp, [::]:8080->8080/tcp
curl: (56) Recv failure: Connection reset by peer
curl status 56
uid=0(root) gid=0(root) groups=0(root)
took 3s
exit=137
```

| Симптом | Причина | Исправление |
|---|---|---|
| `curl` получает «connection reset» | `ENV BIND=127.0.0.1:8080`: сервер слушает только петлю контейнера | `ENV BIND=0.0.0.0:8080` |
| порт опубликован на `0.0.0.0` | `"8080:8080"` открывает его всем компьютерам сети | `"127.0.0.1:8080:8080"` |
| `uid=0(root)` | нет `USER`: служба работает от root | `RUN useradd --system --uid 10001 --no-create-home --shell /usr/sbin/nologin app` и `USER app` |
| остановка занимает весь льготный период, код `137` | `CMD` в shell-форме: `/bin/sh -c` — PID 1 и проглатывает `SIGTERM` | `CMD ["/usr/local/bin/p11-hello"]` |

Пятое замечание: `COPY . .` перед `cargo build` заставляет любое изменение исходников пересобирать все зависимости; сначала копируйте `Cargo.toml` и `Cargo.lock` и собирайте зависимости в отдельном слое (Урок 11.13). Исправленный этап времени выполнения:

```dockerfile
FROM debian:trixie-slim
RUN useradd --system --uid 10001 --no-create-home --shell /usr/sbin/nologin app
COPY --from=build /app/target/release/p11-hello /usr/local/bin/p11-hello
USER app
ENV BIND=0.0.0.0:8080
EXPOSE 8080
CMD ["/usr/local/bin/p11-hello"]
```

После исправлений и `ports: ["127.0.0.1:8080:8080"]` те же команды печатают:

```text
127.0.0.1:8080->8080/tcp
Hello from p11-hello
uid=10001(app) gid=999(app) gr…
took 0s
exit=0
```

Почему это работает: каждый симптом указывает на один уровень: лестницу из Урока 11.11 (адрес привязки), правило публикации (Урок 11.12), пользователя (Урок 11.13) и путь сигнала (Уроки 11.5 и 11.12). Заметьте, что проверка работоспособности из Урока 11.13 при первой ошибке сказала бы «healthy», потому что проверяет изнутри; проблему показывает только `curl` снаружи.

</details>

### Задание 3 — Тест свойств для двоичного поиска

**Цель:** написать двоичный поиск самостоятельно и протестировать его свойствами, чтобы неправильная версия не могла спрятаться. Здесь сходятся Этап 07 и Этап 09.

Создайте библиотеку `p11-bsearch` (`cargo new --lib`), добавьте `proptest = "1.12"` в `[dev-dependencies]` и напишите `binary_search(sorted: &[i32], target: i32) -> Option<usize>` (индекс какого-нибудь элемента, равного `target`). Напишите: один обычный модульный тест; одно свойство «результат совпадает с `contains`»; и одно свойство «каждый элемент, который есть в срезе, находится». Затем **намеренно сломайте функцию** (например, `low = middle + 2` вместо `low = middle + 1`), посмотрите, какие тесты падают, и верните как было.

<details>
<summary>Подсказка</summary>

Используйте полуоткрытый диапазон `low..high` (`while low < high`, `high = middle`, `low = middle + 1`) и `low + (high - low) / 2` для середины. Для второго свойства выбирайте цель **из данных**: `pick in any::<prop::sample::Index>()` и `data[pick.index(data.len())]`. Случайный `i32` почти никогда не равен элементу случайного вектора, поэтому одно первое свойство редко проверяет случай «найдено».

</details>

<details>
<summary>Ответ</summary>

`src/lib.rs`:

```rust
/// Returns the index of some element equal to `target` in the sorted slice, or `None`.
pub fn binary_search(sorted: &[i32], target: i32) -> Option<usize> {
    let mut low = 0;
    let mut high = sorted.len(); // search in sorted[low..high]
    while low < high {
        let middle = low + (high - low) / 2;
        match sorted[middle].cmp(&target) {
            std::cmp::Ordering::Equal => return Some(middle),
            std::cmp::Ordering::Less => low = middle + 1,
            std::cmp::Ordering::Greater => high = middle,
        }
    }
    None
}

#[cfg(test)]
mod tests {
    use super::*;
    use proptest::prelude::*;

    #[test]
    fn finds_and_misses() {
        let data = [1, 3, 5, 7, 9];
        assert_eq!(binary_search(&data, 7), Some(3));
        assert_eq!(binary_search(&data, 4), None);
        assert_eq!(binary_search(&[], 4), None);
    }

    proptest! {
        // Property: the result agrees with a plain linear search.
        #[test]
        fn agrees_with_contains(mut data in prop::collection::vec(any::<i32>(), 0..50), target in any::<i32>()) {
            data.sort();
            match binary_search(&data, target) {
                Some(index) => prop_assert_eq!(data[index], target),
                None => prop_assert!(!data.contains(&target)),
            }
        }

        // Property: every element that is really in the slice is found.
        #[test]
        fn finds_every_present_element(
            mut data in prop::collection::vec(any::<i32>(), 1..50),
            pick in any::<prop::sample::Index>(),
        ) {
            data.sort();
            let target = data[pick.index(data.len())];
            let index = binary_search(&data, target);
            prop_assert!(index.is_some());
            prop_assert_eq!(data[index.unwrap()], target);
        }
    }
}
```

```text
running 3 tests
test tests::finds_and_misses ... ok
test tests::agrees_with_contains ... ok
test tests::finds_every_present_element ... ok

test result: ok. 3 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.01s
```

С ошибкой `low = middle + 2` запуск сообщает `test tests::finds_and_misses ... FAILED` и `test tests::finds_every_present_element ... FAILED`, со строкой `Test failed: assertion failed: index.is_some()` и «minimal failing input». Свойство `agrees_with_contains` при этом всё ещё проходит, и это урок сам по себе: случайные цели почти никогда не лежат в данных, поэтому это свойство в основном проверяет сторону «не найдено».

Почему это работает: первое свойство может упасть, только если функция вернёт неверный ответ для отсутствующего или найденного элемента, но случай «найдено» оно проверяет редко; второе свойство специально строит случай «есть». Свойство сильно настолько, насколько входы доходят до интересного кода. После опыта удалите папку `proptest-regressions`, если `proptest` её создал, или оставьте её как регрессионные данные.

</details>

### Задание 4 — Закройте дыру внедрения

**Цель:** найти и исправить внедрение команды в скрипте и написать тест, который не даст ей вернуться. Здесь сходятся Этап 10 (внедрение, проверка ввода) и Этап 11 (надёжные скрипты).

Этот скрипт печатает файл отчёта `reports/NAME.txt`:

```bash
#!/bin/bash
# show-report.sh NAME - print the file reports/NAME.txt
name=$1
eval "cat reports/$name.txt"
```

В тестовой папке в ВМ создайте `reports/jan.txt` и покажите атаку с безвредной нагрузкой (`./show-report.sh 'jan.txt; echo INJECTED; #'`). Затем перепишите `show-report.sh` так, чтобы он был безопасен, и напишите `test-show-report.sh`, который проверяет: обычное имя работает; нагрузка внедрения **не** выполняется; имя с `../` отклоняется с кодом 2.

<details>
<summary>Подсказка</summary>

`eval` выполняет свой аргумент как код оболочки, поэтому `;` завершает команду и начинает другую. Не собирайте команды из строк: вызывайте `cat` напрямую с `"reports/$name.txt"` как **одним аргументом**. Проверяйте по списку разрешённого: `[[ $name =~ ^[A-Za-z0-9_-]+$ ]]`. Заканчивайте опции через `--`. В тесте запускайте скрипт в папке `mktemp -d` с собственной `reports/`.

</details>

<details>
<summary>Ответ</summary>

Атака печатает отчёт **и** выполняет внедрённую команду:

```text
january numbers
INJECTED
```

Исправленный `show-report.sh`:

```bash
#!/usr/bin/env bash
# show-report.sh NAME - print the file reports/NAME.txt
set -euo pipefail

name=${1:?usage: show-report.sh NAME}
if [[ ! $name =~ ^[A-Za-z0-9_-]+$ ]]; then
    echo "show-report.sh: invalid report name" >&2
    exit 2
fi
cat -- "reports/$name.txt"
```

и `test-show-report.sh`:

```bash
#!/usr/bin/env bash
# test-show-report.sh [SCRIPT] - checks that show-report.sh does not run injected commands.
set -u
script=${1:-./show-report.sh}
failures=0
check() {
    if [[ $2 == "$3" ]]; then echo "ok   - $1"; else echo "FAIL - $1 (expected [$2], got [$3])"; failures=$((failures + 1)); fi
}

work=$(mktemp -d)
trap 'rm -rf "$work"' EXIT
mkdir "$work/reports"
echo "january numbers" > "$work/reports/jan.txt"
script=$(realpath "$script")
cd "$work" || exit 1

check "normal name works" "january numbers" "$(bash "$script" jan 2> /dev/null)"
out=$(bash "$script" 'jan.txt; echo INJECTED; #' 2> /dev/null)
check "injection is not executed" 0 "$(grep -c INJECTED <<< "$out")"
bash "$script" '../x' > /dev/null 2>&1
check "path trick is refused with status 2" 2 "$?"

(( failures == 0 ))
```

На исходном скрипте тест печатает:

```text
ok   - normal name works
FAIL - injection is not executed (expected [0], got [1])
FAIL - path trick is refused with status 2 (expected [2], got [1])
```

а на исправленном:

```text
ok   - normal name works
ok   - injection is not executed
ok   - path trick is refused with status 2
```

Почему это работает: список разрешённого принимает только буквы, цифры, `_` и `-`, поэтому имя не может содержать `;`, пробелы, кавычки или `/`; а без `eval` имя никогда не разбирается как код, а используется только как имя файла. Два слоя (нет выполнения кода и проверка) — это идея «эшелонированной защиты» Этапа 10. Запуск теста сначала на **старом** скрипте показывает, что тест может упасть, как и должен хороший тест.

</details>

## Если что-то не получилось

| Тема | Повторите этот урок |
|---|---|
| Big-O, двоичный поиск | [Урок 07.1 — Что такое алгоритм и нотация Big-O](../07-algorithms-and-data-structures/01-algorithms-and-big-o.md), [Урок 07.10 — Двоичный поиск](../07-algorithms-and-data-structures/10-binary-search.md) |
| Хеш-таблицы, коллизии | [Урок 07.6 — Как работают хеш-таблицы](../07-algorithms-and-data-structures/06-hash-tables.md) |
| Рекурсия и стек вызовов | [Урок 07.7 — Рекурсия и стек вызовов](../07-algorithms-and-data-structures/07-recursion.md) |
| BFS и DFS | [Урок 07.13 — Графы: BFS и DFS](../07-algorithms-and-data-structures/13-graphs-bfs-and-dfs.md) |
| KISS, DRY, YAGNI | [Урок 08.1 — Хороший дизайн: KISS, DRY и YAGNI](../08-software-design/01-good-design-kiss-dry-yagni.md) |
| Композиция и трейты | [Урок 08.4 — Композиция вместо наследования](../08-software-design/04-composition-over-inheritance.md) |
| Конфигурация в окружении | [Урок 08.15 — Приложение двенадцати факторов](../08-software-design/15-the-twelve-factor-app.md) |
| Пирамида тестов | [Урок 09.1 — Пирамида тестов и стратегия тестирования](../09-testing-debugging-benchmarking/01-the-test-pyramid.md) |
| Тесты на основе свойств | [Урок 09.4 — Тестирование на основе свойств](../09-testing-debugging-benchmarking/04-property-based-testing.md) |
| Покрытие и мутационное тестирование | [Урок 09.7 — Покрытие кода и мутационное тестирование](../09-testing-debugging-benchmarking/07-code-coverage-and-mutation-testing.md) |
| Логирование | [Урок 09.10 — Логирование и трассировка](../09-testing-debugging-benchmarking/10-logging-and-tracing.md) |
| Триада CIA | [Урок 10.1 — Мышление безопасности и триада CIA](../10-security-fundamentals/01-security-mindset-and-the-cia-triad.md) |
| Внедрение, проверка ввода | [Урок 10.4 — Проверка ввода](../10-security-fundamentals/04-input-validation.md), [Урок 10.5 — Атаки внедрения](../10-security-fundamentals/05-injection-attacks.md) |
| Хеширование паролей | [Урок 10.6 — Аутентификация и хеширование паролей](../10-security-fundamentals/06-authentication-and-password-hashing.md) |
| Секреты | [Урок 10.8 — Управление секретами](../10-security-fundamentals/08-secrets-management.md) |
| Конвейеры, `sort`, `uniq`, перенаправление | [Урок 11.3 — Текстовые утилиты, конвейеры и перенаправление](03-text-tools-pipes-and-redirection.md) |
| Права доступа | [Урок 11.4 — Пользователи, группы и права доступа](04-users-groups-and-permissions.md) |
| Сигналы, коды возврата | [Урок 11.5 — Процессы и сигналы](05-processes-and-signals.md) |
| Юниты systemd | [Урок 11.6 — Службы и systemd](06-services-with-systemd.md) |
| Строгий режим, `trap`, ShellCheck | [Урок 11.9 — Надёжные скрипты на bash](09-robust-bash-scripts.md) |
| Ключи SSH, `rsync` | [Урок 11.10 — SSH и удалённые серверы](10-ssh-and-remote-servers.md) |
| Адрес привязки, `ss`, `curl` | [Урок 11.11 — Сетевые команды](11-networking-commands.md) |
| Контейнеры, PID 1, тома | [Урок 11.12 — Контейнеры и основы Docker](12-containers-and-docker-basics.md) |
| Dockerfile, многоэтапная сборка, exec-форма | [Урок 11.13 — Dockerfile для программ на Rust](13-dockerfiles-for-rust.md) |
| Compose, политика перезапуска | [Урок 11.14 — Docker Compose](14-docker-compose.md) |
| Развёртывание и откат | [Урок 11.15 — Проект: развёртывание сервиса на Rust](15-project-deploy-a-rust-service.md) |

Если вы ошиблись больше чем в пяти вопросах, повторите уроки из таблицы, переделайте по одному упражнению из каждого и пройдите тест снова через день и ещё раз через неделю (интервальное повторение, как в контрольной точке Стадии 1).

## Что дальше

**Стадия 3 — Как на самом деле работают компьютеры** начинается с [Этапа 12 — Компьютерные системы](../12-computer-systems/README.md). До сих пор вы использовали машину как данность: Rust компилировал ваш код, Linux его выполнял, контейнеры его упаковывали. Стадия 3 открывает коробку: как числа и текст хранятся в битах и байтах, как процессор выполняет инструкции, как ведут себя память и кэши, как компоновщик и загрузчик создают процесс, как операционная система планирует, изолирует и защищает программы и как на самом деле работают потоки, атомарные операции и асинхронные рантаймы (Этапы 12, 13 и 14).

Что переходит с вами:

- **ВМ `dev` и Docker.** Начиная с Этапа 13, опыты с системными вызовами, процессами, пространствами имён и cgroups вы выполняете в ВМ (или в контейнере с нужными параметрами), никогда не от root на вашем Mac. Сохраните ВМ; останавливайте её, когда не пользуетесь (`limactl stop dev`).
- **Shell и скрипты.** Вы будете измерять и сравнивать вещи скриптами и `hyperfine`; сохраните привычки: строгий режим, кавычки, тест для каждого скрипта.
- **Тесты и свойства.** Низкоуровневому коду (битовые операции, аллокаторы, неблокирующие структуры) нужны тесты свойств и инструменты вроде Miri; Этап 09 был подготовкой.
- **Big-O — не вся правда.** Этап 12 покажет, почему два цикла O(n) могут отличаться в десять раз: кэши, предсказание переходов и размещение в памяти.

Перед началом Этапа 12:

1. **Напишите короткую ретроспективу** в журнале обучения: какие три темы Стадии 2 были самыми трудными, каким проектом вы гордитесь больше всего и что сделаете иначе.
2. **Запланируйте повторения** вопросов теста, в которых вы ошиблись: завтра, через неделю, через месяц.
3. **Наведите порядок на машине:** `docker ps -a`, `docker volume ls`, `docker image ls` показывают, что осталось от этого этапа. Удалите свои объекты `course-…` по имени (а не командой prune). В ВМ удалите службы, которые больше не нужны (`sudo systemctl disable --now …`).
4. **Поставьте тег в репозитории:** `git tag -a stage-2-complete -m "Stage 2 complete"` и посмотрите на корневой README глазами незнакомца.

## Источники и что почитать

- [The Rust Programming Language](https://doc.rust-lang.org/book/) — официальная книга; держите её открытой на Стадии 3.
- [Rust Performance Book](https://nnethercote.github.io/perf-book/) — практическое руководство, которое начинается там, где заканчивается Этап 12.
- [The Linux Command Line (William Shotts)](https://linuxcommand.org/tlcl.php) — бесплатная книга, углубляющая навыки работы с оболочкой из Этапа 11.
- [Docker documentation](https://docs.docker.com/) — справочник по всему, что в Уроках 11.12–11.14.
- [OWASP Cheat Sheet Series](https://cheatsheetseries.owasp.org/) — короткие практические чек-листы по темам Этапа 10.
- [Algorithms, Part I and II (Sedgewick and Wayne) — booksite](https://algs4.cs.princeton.edu/home/) — бесплатные сопутствующие материалы к алгоритмам Этапа 07.

---

[🏠 Начало](../START-HERE.md) · [← 11.15 Проект: развёртывание сервиса на Rust](15-project-deploy-a-rust-service.md) · [📋 Этап 11](README.md) · [Этап 12 — Компьютерные системы →](../12-computer-systems/README.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
