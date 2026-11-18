<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 08.13 Рефакторинг и код-ревью](13-refactoring-and-code-review.md) · [📋 Этап 08](README.md) · [08.15 Приложение двенадцати факторов →](15-the-twelve-factor-app.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/08-software-design/14-layers-and-hexagonal-architecture.md) <!-- nav -->

# Урок 08.14 — Слои и гексагональная архитектура

> **Этап 08 — Проектирование ПО: принципы, паттерны и соглашения** · Урок 14 из 17 · ⏱ около 3 часов (2 занятия)

## Что вы узнаете

- Что такое *архитектура* для небольшой программы: решение о том, какой код может знать о каком другом коде.
- Слоистую архитектуру и проблему, которая у неё есть, когда бизнес-правила зависят от базы данных.
- Гексагональную архитектуру (порты и адаптеры): ядро определяет трейты, внешний мир подключается к ним.
- Как построить таким способом небольшую программу-список задач, проверить её правила без единого файла и проверить правило зависимостей командой `grep`.
- Когда такая структура избыточна.

## Перед началом

- Вы прошли [Урок 08.13 — Рефакторинг и код-ревью](13-refactoring-and-code-review.md).
- Вы знаете разделение ответственности ([Урок 08.2](02-cohesion-coupling-and-separation-of-concerns.md)), принцип инверсии зависимостей ([Урок 08.3](03-solid-principles.md)), внедрение зависимостей ([Урок 08.9](09-strategy-and-dependency-injection.md)) и структуру проекта из [Урока 08.12](12-project-layout-and-workspaces.md).
- Создайте пакет: `cargo new code/p08-tasks`, затем `cd code/p08-tasks`, `cargo add thiserror@2` и `cargo add --dev tempfile@3`.

## Новые слова

| Слово | Значение |
|---|---|
| **архитектура** (architecture) | Крупная структура программы: её части и правила о том, какая часть может зависеть от какой. |
| **слой** (layer) | Группа кода с одним видом работы, расположенная по порядку, где код может использовать слои ниже себя. |
| **предметная область** (domain) | Часть программы, где хранятся бизнес-правила и понятия, например «у задачи непустой заголовок». Она ничего не знает об экранах, файлах и сетях. |
| **сценарий использования** (use case, прикладной сервис) | Одно, что пользователь хочет сделать с программой: «добавить задачу», «завершить задачу». Для этого он использует предметную область. |
| **инфраструктура** (infrastructure) | Код, который общается с внешним миром: файлы, базы данных, сети, часы. |
| **порт** (port) | Трейт, определённый ядром, который описывает, что ему нужно от внешнего мира (`TaskRepository`) или что оно предлагает. |
| **адаптер** (adapter) | Тип, который соединяет порт с реальной технологией. `FileRepo` приспосабливает порт `TaskRepository` к текстовому файлу. |
| **ведущий адаптер** (driving adapter) | Адаптер, который *вызывает* ядро: интерфейс командной строки, веб-обработчик, тест. |
| **ведомый адаптер** (driven adapter) | Адаптер, который ядро *вызывает* через порт: хранилище, почта, часы. |
| **гексагональная архитектура** (hexagonal architecture, порты и адаптеры) | Способ построения программы с предметной областью и сценариями в середине и портами и адаптерами вокруг них, так что зависимости направлены внутрь. |
| **правило зависимостей** (dependency rule) | Правило этой архитектуры: зависимости в исходном коде направлены к предметной области; она никогда не упоминает адаптеры. |
| **корень композиции** (composition root) | Единственное место (обычно `main`), где выбираются конкретные адаптеры и подключаются к ядру. |
| **CLI** (Command-Line Interface, интерфейс командной строки) | Программа, которой пользуются, вводя команды в терминале. |
| **HTTP** (HyperText Transfer Protocol, протокол передачи гипертекста) | Протокол, по которому общаются веб-браузеры и веб-сервисы. |

## 1. Для чего нужна архитектура

Принципы проектирования (Уроки 08.1–08.9) касаются функций и типов. **Архитектура** — то же мышление для всей программы: какие части существуют и какая часть может знать о какой. Две цели
определяют почти каждое решение:

1. **Менять одно, не переписывая остальное.** Замените текстовый файл базой данных или командную строку веб-страницей, а бизнес-правила останутся нетронутыми.
2. **Проверять правила без реального мира.** «Заголовок не может быть пустым» должно проверяться за микросекунды, без файла, сети и часов.

Если ваши правила вызывают `std::fs` или печатают на экран, обе цели потеряны.

## 2. Слоистая архитектура

Самый старый ответ — **слои**: разложите код по виду работы и позвольте каждому слою использовать только слой ниже.

```text
┌──────────────────────────────────────┐
│ Interface    CLI, web page, ...      │  talks to people and other programs
├──────────────────────────────────────┤
│ Application  use cases               │  "add a task", "complete a task"
├──────────────────────────────────────┤
│ Domain       rules and concepts      │  what a task is, what a title may be
├──────────────────────────────────────┤
│ Infrastructure  files, database, ... │  talks to the outside world
└──────────────────────────────────────┘
        code may call downwards only
```

Это уже лучше одного гигантского `main`: в интерфейсе нет бизнес-правил, а в правилах нет кода экрана. Но у классической картины есть изъян: **предметная область стоит выше инфраструктуры, поэтому вызывает её**.
Функция предметной области `complete_task` вызывала бы `database::save(...)`. Теперь правила зависят от базы данных: без неё их не проверить, а замена базы данных означает вскрытие правил. Четыре слоя нужны редко; два-три
достаточно. Важно **направление стрелок**.

## 3. Гексагональная архитектура: порты и адаптеры

Решение — развернуть стрелку между ядром и инфраструктурой (принцип инверсии зависимостей из Урока 08.3). Ядро не вызывает базу данных. Ядро *объявляет, что ему нужно*, в виде трейта, а инфраструктура реализует этот трейт.

```text
 driving adapters                                           driven adapters
 (call the core)                                            (called by the core)

  CLI  ───────┐                                       ┌───────  FileRepo
              ▼                                       ▼
  tests ──► [use cases]  ◄── domain rules ──►  [port: TaskRepository]  ◄── InMemoryRepo
              ▲                                       ▲
  HTTP ───────┘                                       └───────  (a database, later)

                 └──────────── the core ────────────┘
        Arrows of source-code dependency all point towards the core.
```

- **Ядро** — это предметная область плюс сценарии использования. Это обычный Rust: ни `std::fs`, ни печати, ни фреймворков.
- **Порт** — трейт, принадлежащий ядру. `TaskRepository` говорит: «Мне нужно нечто, что умеет добавлять, перечислять и сохранять задачи». Трейтом владеет ядро, поэтому он формируется нуждами ядра, а не базы данных.
- **Адаптер** — тип во внешнем слое, который реализует порт или вызывает ядро. `FileRepo` реализует `TaskRepository` с помощью текстового файла. `InMemoryRepo` реализует его с помощью вектора. CLI — адаптер
  с другой стороны: он превращает аргументы командной строки в вызовы сценариев использования.
- **Корень композиции**, `main`, — единственное место, которое знает все конкретные типы. Он выбирает `FileRepo`, передаёт его в `TaskService` и запускает CLI.

> 📏 **Соглашение:** **правило зависимостей**: строки `use` направлены внутрь. `domain` ничего не импортирует из остальной программы. `ports` и `service` импортируют `domain` (и `ports`). Только `adapters` и `main` импортируют
> `std::fs`, `std::env` или другие технологии. Правило можно проверить командой `grep`, как вы увидите в разделе 5.

## 4. Рабочий пример: задачи

Вы построите список задач с командами `add`, `done` и `list`. Файлы:

```text
p08-tasks/
├── Cargo.toml
├── src/
│   ├── lib.rs          ← объявляет модули
│   ├── domain.rs       ← Title, Task, DomainError            (ядро)
│   ├── ports.rs        ← TaskRepository, StoreError          (ядро)
│   ├── service.rs      ← TaskService: сценарии использования (ядро)
│   ├── adapters/
│   │   ├── mod.rs
│   │   ├── memory.rs   ← InMemoryRepo  (ведомый адаптер)
│   │   └── file.rs     ← FileRepo      (ведомый адаптер)
│   └── main.rs         ← CLI + корень композиции (ведущий адаптер)
└── tests/
    └── contract.rs     ← одни и те же проверки для каждого адаптера
```

`src/lib.rs`:

```rust
//! A tiny task list built as ports and adapters.

pub mod adapters;
pub mod domain;
pub mod ports;
pub mod service;
```

**Предметная область.** В `src/domain.rs` лежат понятия и правила; из внешнего используется только `thiserror` (Урок 05.11) для сообщений об ошибках:

```rust
//! The domain: what a task is and which rules always hold.

use thiserror::Error;

/// A validated task title: not empty, one line.
#[derive(Debug, Clone, PartialEq)]
pub struct Title(String);

/// A rule of the domain was broken.
#[derive(Debug, Error, PartialEq)]
pub enum DomainError {
    /// The title is empty or only spaces.
    #[error("the title must not be empty")]
    EmptyTitle,
    /// The title has more than one line.
    #[error("the title must be one line")]
    MultiLineTitle,
}

impl Title {
    /// Checks the rules and makes a title.
    pub fn new(text: &str) -> Result<Title, DomainError> {
        let text = text.trim();
        if text.is_empty() {
            Err(DomainError::EmptyTitle)
        } else if text.contains('\n') {
            Err(DomainError::MultiLineTitle)
        } else {
            Ok(Title(text.to_string()))
        }
    }

    /// The text of the title.
    #[must_use]
    pub fn as_str(&self) -> &str {
        &self.0
    }
}

/// Something to do.
#[derive(Debug, Clone, PartialEq)]
pub struct Task {
    /// The identifier, chosen by the repository.
    pub id: u32,
    /// What to do.
    pub title: Title,
    /// Is it finished?
    pub done: bool,
}
```

**Что происходит:** `Title` — это *newtype* с приватным полем; единственный способ получить его — `Title::new`, которая проверяет правила (Урок 08.5), поэтому у `Task` всегда корректный заголовок. `DomainError` говорит
словами предметной области и ничего не говорит о файлах или экранах. `id` выбирает репозиторий, потому что способ создания идентификаторов зависит от хранилища.

**Порт.** `src/ports.rs`:

```rust
//! The ports: what the core needs from the outside world.

use thiserror::Error;

use crate::domain::{Task, Title};

/// Why a repository operation failed.
#[derive(Debug, Error, PartialEq)]
pub enum StoreError {
    /// No task has this identifier.
    #[error("task {0} not found")]
    NotFound(u32),
    /// The storage itself failed (disk, network, ...).
    #[error("storage failed: {0}")]
    Storage(String),
}

/// Where tasks are kept. The core defines it; adapters implement it.
pub trait TaskRepository {
    /// Stores a new task and returns it with its new identifier.
    fn add(&mut self, title: Title) -> Result<Task, StoreError>;

    /// All tasks, oldest first.
    fn all(&self) -> Result<Vec<Task>, StoreError>;

    /// Replaces the stored task that has the same identifier.
    fn save(&mut self, task: &Task) -> Result<(), StoreError>;
}
```

**Что происходит:** `TaskRepository` — дверь, через которую ядро общается с хранилищем, словами ядра (`add`, `all`, `save`), а не файла. `StoreError` тоже принадлежит ядру: адаптер должен
переводить в него свои ошибки (`std::io::Error`, ошибку базы данных), поэтому типы ошибок конкретных технологий никогда не попадают в ядро.

**Сценарии использования.** `src/service.rs`:

```rust
//! The application service: the use cases of the program.

use thiserror::Error;

use crate::domain::{DomainError, Task, Title};
use crate::ports::{StoreError, TaskRepository};

/// Why a use case failed.
#[derive(Debug, Error, PartialEq)]
pub enum ServiceError {
    /// The input broke a rule of the domain.
    #[error(transparent)]
    Invalid(#[from] DomainError),
    /// The storage said no.
    #[error(transparent)]
    Store(#[from] StoreError),
}

/// The use cases, independent of where tasks are stored.
pub struct TaskService<R: TaskRepository> {
    repo: R,
}

impl<R: TaskRepository> TaskService<R> {
    /// Creates a service that uses `repo`.
    pub fn new(repo: R) -> TaskService<R> {
        TaskService { repo }
    }

    /// Adds a task with the given title.
    pub fn add(&mut self, title: &str) -> Result<Task, ServiceError> {
        let title = Title::new(title)?;
        Ok(self.repo.add(title)?)
    }

    /// Marks a task as done.
    pub fn complete(&mut self, id: u32) -> Result<Task, ServiceError> {
        let mut task = self
            .repo
            .all()?
            .into_iter()
            .find(|task| task.id == id)
            .ok_or(StoreError::NotFound(id))?;
        task.done = true;
        self.repo.save(&task)?;
        Ok(task)
    }

    /// The tasks that are not done yet.
    pub fn pending(&self) -> Result<Vec<Task>, ServiceError> {
        let mut tasks = self.repo.all()?;
        tasks.retain(|task| !task.done);
        Ok(tasks)
    }
}
```

**Что происходит:**

1. `TaskService<R: TaskRepository>` обобщён по любому репозиторию. Его публичные методы `add`, `complete` и `pending` — это *ведущий порт*: то, что внешний мир может попросить ядро сделать.
2. `add` просит предметную область проверить заголовок, затем просит репозиторий сохранить его; `?` преобразует `DomainError` и `StoreError` в `ServiceError` благодаря `#[from]`.
3. В `complete` лежит всё правило «найти задачу, пометить выполненной, сохранить». В нём нет ни кода работы с файлами, ни `println!`.
4. Обобщённый параметр означает статическую диспетчеризацию (Урок 05.1). `Box<dyn TaskRepository>` тоже подошёл бы (Урок 06.1) и лучше, когда адаптер выбирается во время выполнения.

**Адаптеры.** `src/adapters/mod.rs` только объявляет два модуля (`pub mod file;` и `pub mod memory;`). `src/adapters/memory.rs` — самый простой адаптер, и именно его используют тесты:

```rust
//! An adapter that keeps tasks in memory. Good for tests.

use crate::domain::{Task, Title};
use crate::ports::{StoreError, TaskRepository};

/// Tasks in a vector.
#[derive(Debug, Default)]
pub struct InMemoryRepo {
    tasks: Vec<Task>,
}

impl TaskRepository for InMemoryRepo {
    fn add(&mut self, title: Title) -> Result<Task, StoreError> {
        let id = u32::try_from(self.tasks.len() + 1).expect("fewer than 2^32 tasks");
        let task = Task {
            id,
            title,
            done: false,
        };
        self.tasks.push(task.clone());
        Ok(task)
    }

    fn all(&self) -> Result<Vec<Task>, StoreError> {
        Ok(self.tasks.clone())
    }

    fn save(&mut self, task: &Task) -> Result<(), StoreError> {
        let slot = self
            .tasks
            .iter_mut()
            .find(|stored| stored.id == task.id)
            .ok_or(StoreError::NotFound(task.id))?;
        *slot = task.clone();
        Ok(())
    }
}
```

`src/adapters/file.rs` — настоящее хранилище. В файле одна строка на задачу, `0 title` или `1 title`, а идентификатор — номер строки:

```rust
//! An adapter that keeps tasks in a text file: one line per task, `0 title` or `1 title`.

use std::fs;
use std::io;
use std::path::PathBuf;

use crate::domain::{Task, Title};
use crate::ports::{StoreError, TaskRepository};

/// Tasks in a text file. The identifier is the line number, starting at 1.
#[derive(Debug)]
pub struct FileRepo {
    path: PathBuf,
}

impl FileRepo {
    /// Uses the file at `path`; it is created on the first `add`.
    pub fn new(path: impl Into<PathBuf>) -> FileRepo {
        FileRepo { path: path.into() }
    }

    fn write_all(&self, tasks: &[Task]) -> Result<(), StoreError> {
        let mut text = String::new();
        for task in tasks {
            text.push_str(if task.done { "1 " } else { "0 " });
            text.push_str(task.title.as_str());
            text.push('\n');
        }
        fs::write(&self.path, text).map_err(storage_error)
    }
}

fn storage_error(error: io::Error) -> StoreError {
    StoreError::Storage(error.to_string())
}

impl TaskRepository for FileRepo {
    fn add(&mut self, title: Title) -> Result<Task, StoreError> {
        let mut tasks = self.all()?;
        let id = u32::try_from(tasks.len() + 1).expect("fewer than 2^32 tasks");
        let task = Task {
            id,
            title,
            done: false,
        };
        tasks.push(task.clone());
        self.write_all(&tasks)?;
        Ok(task)
    }

    fn all(&self) -> Result<Vec<Task>, StoreError> {
        let text = match fs::read_to_string(&self.path) {
            Ok(text) => text,
            Err(error) if error.kind() == io::ErrorKind::NotFound => return Ok(Vec::new()),
            Err(error) => return Err(storage_error(error)),
        };
        let mut tasks = Vec::new();
        for (index, line) in text.lines().enumerate() {
            let (flag, title) = line
                .split_once(' ')
                .ok_or_else(|| StoreError::Storage(format!("bad line {}", index + 1)))?;
            let title = Title::new(title)
                .map_err(|error| StoreError::Storage(format!("line {}: {error}", index + 1)))?;
            let id = u32::try_from(index + 1).expect("fewer than 2^32 tasks");
            tasks.push(Task {
                id,
                title,
                done: flag == "1",
            });
        }
        Ok(tasks)
    }

    fn save(&mut self, task: &Task) -> Result<(), StoreError> {
        let mut tasks = self.all()?;
        let slot = tasks
            .iter_mut()
            .find(|stored| stored.id == task.id)
            .ok_or(StoreError::NotFound(task.id))?;
        *slot = task.clone();
        self.write_all(&tasks)
    }
}
```

**Что происходит:** оба адаптера реализуют один и тот же трейт, поэтому сервис не может их различить. `FileRepo` — единственное место с `std::fs`: отсутствующий файл означает пустой список, а любая другая `io::Error` (или плохая строка) становится
`StoreError::Storage`. `.expect("fewer than 2^32 tasks")` отмечает предположение о нашей собственной программе, а не о вводе (Урок 08.10).

**Ведущий адаптер и корень композиции.** `src/main.rs`:

```rust
use std::env;
use std::process::ExitCode;

use p08_tasks::adapters::file::FileRepo;
use p08_tasks::service::TaskService;

fn main() -> ExitCode {
    let path = env::var("TASKS_FILE").unwrap_or_else(|_| String::from("tasks.txt"));
    let mut service = TaskService::new(FileRepo::new(path));

    let args: Vec<String> = env::args().skip(1).collect();
    let result = match args.first().map(String::as_str) {
        Some("add") => service
            .add(&args[1..].join(" "))
            .map(|task| println!("added {}: {}", task.id, task.title.as_str())),
        Some("done") => match args.get(1).and_then(|text| text.parse().ok()) {
            Some(id) => service
                .complete(id)
                .map(|task| println!("done {}: {}", task.id, task.title.as_str())),
            None => {
                eprintln!("usage: done <number>");
                return ExitCode::from(2);
            }
        },
        Some("list") => service.pending().map(|tasks| {
            for task in tasks {
                println!("{}: {}", task.id, task.title.as_str());
            }
        }),
        _ => {
            eprintln!("usage: add <title> | done <number> | list");
            return ExitCode::from(2);
        }
    };

    match result {
        Ok(()) => ExitCode::SUCCESS,
        Err(error) => {
            eprintln!("error: {error}");
            ExitCode::FAILURE
        }
    }
}
```

**Что происходит:** имя файла берётся из переменной окружения `TASKS_FILE` (по умолчанию `tasks.txt`): конфигурация читается на краю, а не в ядре (почему — объясняет Урок 08.15). `main` выбирает `FileRepo`, передаёт его в
`TaskService::new`, превращает аргументы в вызовы сервиса, печатает и выбирает код возврата (`0` — успех, `1` — ошибка, `2` — неверное использование). Чтобы перейти на базу данных, вы пишете `DatabaseRepo`, реализуете `TaskRepository` и меняете **одну строку** в `main`.

Попробуйте (строка `export` держит файл данных во временном месте):

```bash
export TASKS_FILE=/tmp/p08-tasks.txt
rm -f $TASKS_FILE
cargo run -q -- list
cargo run -q -- add write the lesson
cargo run -q -- add test the lesson
cargo run -q -- list
cargo run -q -- done 1
cargo run -q -- list
cargo run -q -- done 9
cargo run -q -- add "   "
cat $TASKS_FILE
```

```text
added 1: write the lesson
added 2: test the lesson
1: write the lesson
2: test the lesson
done 1: write the lesson
2: test the lesson
error: task 9 not found
error: the title must not be empty
1 write the lesson
0 test the lesson
```

(Первый `list` ничего не печатает, потому что файла ещё нет.)

## 5. Тестирование и проверка правила зависимостей

**Правила без диска.** Тесты сервиса используют `InMemoryRepo` и крошечный `BrokenRepo`, который всегда завершается ошибкой. Добавьте это в конец `src/service.rs`:

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use crate::adapters::memory::InMemoryRepo;

    /// A repository that always fails, to test the error path.
    struct BrokenRepo;

    impl TaskRepository for BrokenRepo {
        fn add(&mut self, _title: Title) -> Result<Task, StoreError> {
            Err(StoreError::Storage(String::from("disk full")))
        }
        fn all(&self) -> Result<Vec<Task>, StoreError> {
            Err(StoreError::Storage(String::from("disk full")))
        }
        fn save(&mut self, _task: &Task) -> Result<(), StoreError> {
            Err(StoreError::Storage(String::from("disk full")))
        }
    }

    #[test]
    fn added_tasks_are_pending_until_completed() {
        let mut service = TaskService::new(InMemoryRepo::default());
        service.add("write lesson").unwrap();
        let second = service.add("test lesson").unwrap();
        assert_eq!(service.pending().unwrap().len(), 2);

        service.complete(second.id).unwrap();
        let pending = service.pending().unwrap();
        assert_eq!(pending.len(), 1);
        assert_eq!(pending[0].title.as_str(), "write lesson");
    }

    #[test]
    fn an_empty_title_is_a_domain_error() {
        let mut service = TaskService::new(InMemoryRepo::default());
        assert_eq!(
            service.add("   "),
            Err(ServiceError::Invalid(DomainError::EmptyTitle))
        );
    }

    #[test]
    fn completing_an_unknown_task_is_not_found() {
        let mut service = TaskService::new(InMemoryRepo::default());
        assert_eq!(
            service.complete(7),
            Err(ServiceError::Store(StoreError::NotFound(7)))
        );
    }

    #[test]
    fn storage_errors_reach_the_caller() {
        let mut service = TaskService::new(BrokenRepo);
        let error = service.add("anything").unwrap_err();
        assert_eq!(error.to_string(), "storage failed: disk full");
    }
}
```

**Что происходит:** первые три теста проверяют сценарии и правила предметной области на адаптере в памяти: микросекунды, никаких файлов. `BrokenRepo` — *тестовый двойник* (их ввёл Урок 08.9, подробнее — Урок 09.3): несколько строк, которые делают
проверяемым путь ошибки хранилища; попробуйте проверить «диск переполнен» на настоящем файле. Модулю тестов можно импортировать адаптер, потому что тесты играют роль `main`.

**Один контракт для всех адаптеров.** Если два адаптера заявляют, что реализуют один порт, они должны вести себя одинаково. `tests/contract.rs` запускает одну функцию на обоих:

```rust
//! Every adapter of `TaskRepository` must pass the same checks.

use p08_tasks::adapters::file::FileRepo;
use p08_tasks::adapters::memory::InMemoryRepo;
use p08_tasks::domain::Title;
use p08_tasks::ports::{StoreError, TaskRepository};

fn check_contract(mut repo: impl TaskRepository) {
    assert!(repo.all().unwrap().is_empty());

    let first = repo.add(Title::new("one").unwrap()).unwrap();
    let second = repo.add(Title::new("two").unwrap()).unwrap();
    assert_eq!((first.id, second.id), (1, 2));

    let mut done = first.clone();
    done.done = true;
    repo.save(&done).unwrap();

    let all = repo.all().unwrap();
    assert_eq!(all.len(), 2);
    assert!(all[0].done);
    assert!(!all[1].done);

    let mut ghost = second;
    ghost.id = 99;
    assert_eq!(repo.save(&ghost), Err(StoreError::NotFound(99)));
}

#[test]
fn in_memory_repo_keeps_the_contract() {
    check_contract(InMemoryRepo::default());
}

#[test]
fn file_repo_keeps_the_contract() {
    let dir = tempfile::tempdir().unwrap();
    check_contract(FileRepo::new(dir.path().join("tasks.txt")));
}
```

`cargo test`:

```text
test service::tests::added_tasks_are_pending_until_completed ... ok
test service::tests::an_empty_title_is_a_domain_error ... ok
test service::tests::completing_an_unknown_task_is_not_found ... ok
test service::tests::storage_errors_reach_the_caller ... ok
test result: ok. 4 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
...
test in_memory_repo_keeps_the_contract ... ok
test file_repo_keeps_the_contract ... ok
test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

Новый адаптер (база данных) добавляется в этот тест ещё одной строкой, и он обязан его пройти. Так порты остаются честными.

**Проверка правила зависимостей.** Правило говорит, что ядро не импортирует технологий. Попросите `grep` найти в файлах ядра любую строку `use`, упоминающую адаптер или модуль технологии:

```bash
grep -n "^use crate::adapters\|^use std::fs\|^use std::env" src/domain.rs src/ports.rs src/service.rs
```

Он ничего не печатает (и `grep` завершается с кодом 1, «нет совпадений»). Строка `use crate::adapters::memory::InMemoryRepo;` в `service.rs` есть, но она с отступом внутри `mod tests`, поэтому шаблон `^use` её не находит.
В настоящем проекте эту команду можно запускать в CI (Continuous Integration, непрерывная интеграция). В рабочем пространстве (Урок 08.12) правило может навязать компилятор: поместите ядро в отдельный крейт, в зависимостях которого нет технологий адаптеров.

## 6. Не слишком ли это много?

Такая структура стоит файлов, трейтов и немного косвенности. Используйте её, когда она окупается:

- **Используйте**, когда в программе есть настоящие бизнес-правила; когда хранилище или интерфейс будут меняться или существовать в нескольких видах (сейчас файл, потом база данных; командная строка и веб); когда нужны быстрые тесты правил.
- **Не используйте** для скрипта в 100 строк, разового инструмента или программы, которая перекладывает данные из A в B без правил. Порт с одной реализацией и без тестового двойника ничего не даёт (Урок 08.1: YAGNI, «You Aren't Gonna Need It» — «тебе это не понадобится»). Также избегайте
  сквозных слоёв: «сервис», каждый метод которого пересылает вызов одному методу репозитория, добавляет слой без пользы.
- **Копируйте стрелки, а не названия папок.** Папки `domain`, `ports`, `adapters` — ещё не архитектура; архитектура — это правило, что строки `use` направлены внутрь.
- **Вводите порт, когда** есть вторая реализация, нужен тестовый двойник или вы ожидаете смену технологии. До тех пор достаточно обычной функции в модуле, а трейт можно выделить позже (Урок 08.13).

## Частые ошибки

1. **Бизнес-правила в `main` или в адаптерах.** CLI должен разбирать аргументы и печатать, и ничего больше.
2. **Пускать `std::io::Error` или тип ошибки базы данных в ядро.** Преобразуйте на границе адаптера в собственный тип ошибки ядра.
3. **Формировать порт по технологии** (`fn run_sql(&self, query: &str)`). Порт говорит на языке ядра.
4. **Один трейт на всё.** `Repository` с 30 методами — «божественный» тип. Предпочитайте небольшие порты (разделение интерфейсов, Урок 08.3).

## Попробуйте сами

1. Постройте `p08-tasks` и выполните команды блока «Попробуйте». Сравните вывод.
2. Намеренно сломайте `FileRepo` (напишите `1 ` там, где в `write_all` должно быть `0 `) и запустите `cargo test`: упадёт контрактный тест, тесты сервиса — нет.
3. Выполните команду `grep` из раздела 5. Добавьте `use std::fs;` в `domain.rs`, запустите её снова, чтобы увидеть нарушение, и удалите строку.
4. Коммит: `git add code/p08-tasks && git commit -m "feat(p08): task list with ports and adapters"`.

## Упражнения

### Упражнение 1 — По какую сторону черты? (легко)

Скажите для каждого элемента, относится ли он к ядру (предметная область / сценарии), к порту или к адаптеру: (а) правило, что заголовок не может быть пустым; (б) функция, читающая `tasks.txt`; (в) `trait Clock { fn now(&self) -> u64; }`;
(г) код, превращающий `done 1` из командной строки в вызов `complete(1)`; (д) структура `FixedClock(u64)`, используемая в тестах.

<details>
<summary>Подсказка</summary>

Спросите: упоминает ли это технологию (файл, терминал, сеть)? Принадлежит ли трейт ядру?

</details>

<details>
<summary>Ответ</summary>

(а) ядро (предметная область). (б) адаптер (`FileRepo`, ведомый). (в) порт: ядро владеет описанием того, что ему нужно от часов. (г) адаптер (CLI, ведущий). (д) адаптер порта `Clock`, написанный для тестов.

</details>

### Упражнение 2 — Добавьте сценарий (средне)

Добавьте `TaskService::reopen(id)`, который помечает завершённую задачу как невыполненную, и напишите тест с `InMemoryRepo`. Нужно ли менять порт или какие-либо адаптеры?

<details>
<summary>Подсказка</summary>

`reopen` похож на `complete` с `done = false`. Подумайте, какие операции репозиторий уже предлагает.

</details>

<details>
<summary>Ответ</summary>

Менять порт и адаптеры не нужно: `all` и `save` достаточно. В `impl<R: TaskRepository> TaskService<R>`:

```rust
    /// Marks a finished task as not done again.
    pub fn reopen(&mut self, id: u32) -> Result<Task, ServiceError> {
        let mut task = self
            .repo
            .all()?
            .into_iter()
            .find(|task| task.id == id)
            .ok_or(StoreError::NotFound(id))?;
        task.done = false;
        self.repo.save(&task)?;
        Ok(task)
    }
```

а в `mod tests`:

```rust
    #[test]
    fn a_reopened_task_is_pending_again() {
        let mut service = TaskService::new(InMemoryRepo::default());
        let task = service.add("write lesson").unwrap();
        service.complete(task.id).unwrap();
        assert!(service.pending().unwrap().is_empty());

        service.reopen(task.id).unwrap();
        assert_eq!(service.pending().unwrap().len(), 1);
    }
```

Почему это работает: новый сценарий — это новый код только в ядре. `cargo test --lib` показывает 5 пройденных. (Повторяющийся код «найти задачу» в `complete` и `reopen` — кандидат на рефакторинг: приватный метод `find(id)`; Урок 08.13.)

</details>

### Упражнение 3 — Второй порт (сложно)

Программа должна радоваться, когда задача завершена. Добавьте порт `Notifier` с `fn task_completed(&self, task: &Task);`, второй типовой параметр в `TaskService`, двойник `RecordingNotifier` для тестов и адаптер `ConsoleNotifier` в `main`.

<details>
<summary>Подсказка</summary>

Порт идёт в `ports.rs`. `TaskService<R: TaskRepository, N: Notifier>` хранит уведомитель во втором поле; `complete` вызывает его после успешного `save`. Двойнику нужна внутренняя изменяемость (`RefCell<Vec<String>>`, Урок 06.4), потому что `task_completed` принимает `&self`.

</details>

<details>
<summary>Ответ</summary>

В `src/ports.rs`:

```rust
/// Tells someone that something happened. The core defines it; adapters implement it.
pub trait Notifier {
    /// Called after a task has been completed.
    fn task_completed(&self, task: &Task);
}
```

В `src/service.rs`: `use crate::ports::{Notifier, StoreError, TaskRepository};`, структура становится `pub struct TaskService<R: TaskRepository, N: Notifier> { repo: R, notifier: N }`, заголовок `impl` —
`impl<R: TaskRepository, N: Notifier> TaskService<R, N>`, `new(repo: R, notifier: N)` сохраняет оба, а `complete` вызывает `self.notifier.task_completed(&task);` сразу после `self.repo.save(&task)?;`.

В тестах (добавьте `use std::cell::RefCell;`, и каждый `TaskService::new(x)` становится `TaskService::new(x, RecordingNotifier::default())`):

```rust
    #[derive(Default)]
    struct RecordingNotifier {
        messages: RefCell<Vec<String>>,
    }

    impl Notifier for RecordingNotifier {
        fn task_completed(&self, task: &Task) {
            self.messages.borrow_mut().push(format!("done: {}", task.title.as_str()));
        }
    }
```

В первом тесте после `service.complete(second.id).unwrap();` добавьте
`assert_eq!(service.notifier.messages.borrow().as_slice(), ["done: test lesson"]);`. В `main.rs` напишите `struct ConsoleNotifier;` с `impl Notifier for ConsoleNotifier`, метод которого печатает
`well done! finished: ...`, и вызовите `TaskService::new(FileRepo::new(path), ConsoleNotifier)`. Все прежние тесты продолжают проходить, а `cargo clippy --all-targets -- -D warnings` чист.

Почему это работает: у ядра теперь две двери, и каждый адаптер подключается к своей.

</details>

## Вопросы для самопроверки

1. Что такое правило зависимостей?

<details>
<summary>Ответ</summary>

Зависимости в исходном коде направлены к предметной области. Ядро (предметная область и сценарии) никогда не импортирует адаптеры или технологии; адаптеры и `main` импортируют ядро.

</details>

2. Чем порт отличается от адаптера?

<details>
<summary>Ответ</summary>

Порт — трейт, принадлежащий ядру, который описывает, что ему нужно (или что оно предлагает). Адаптер — тип во внешнем слое, который реализует порт или вызывает ядро, соединяя его с реальной технологией.

</details>

3. Почему `StoreError` принадлежит ядру, а не `FileRepo`?

<details>
<summary>Ответ</summary>

Чтобы ядро не зависело от типа ошибки какой-либо технологии. Адаптеры преобразуют свои ошибки (`io::Error`, ошибки базы данных) в `StoreError`, а ядро обрабатывает только свой словарь.

</details>

4. Когда порт не стоит затрат?

<details>
<summary>Ответ</summary>

Когда реализация одна, тестовый двойник не нужен и технология не изменится: обычная функция проще. Добавляйте трейт, когда его требует вторая реализация или тест.

</details>

## Итоги

- **Архитектура** — это правило о том, какие части программы могут зависеть от каких. Цели: менять одну часть, не переписывая остальное, и проверять правила без реального мира.
- **Слоистая архитектура** упорядочивает код по виду работы и позволяет каждому слою использовать слой ниже; её слабость в том, что предметная область оказывается зависимой от хранилища.
- **Гексагональная архитектура** разворачивает эту стрелку: ядро владеет **портами** (трейтами), **адаптеры** реализуют их (ведомые) или вызывают ядро (ведущие), а `main` — **корень композиции**, который их соединяет.
- **Правило зависимостей**: строки `use` направлены внутрь. Ядро не импортирует `std::fs`, не печатает и не использует фреймворки; проверяйте это через `grep` или границы крейтов.
- Ошибки пересекают границы в собственных типах ядра. Адаптеры переводят ошибки технологий.
- Правила проверяйте адаптером в памяти, пути ошибок — падающим двойником, а на всех адаптерах запускайте один **контрактный тест**.
- Не стройте это для крошечных программ и не добавляйте трейт, пока для него нет причины.

## Источники и что почитать

- [Alistair Cockburn — Hexagonal architecture](https://alistair.cockburn.us/hexagonal-architecture/) — оригинальная статья, в которой появились порты и адаптеры.
- [Martin Fowler — Presentation Domain Data Layering](https://martinfowler.com/bliki/PresentationDomainDataLayering.html) — короткая статья о классических слоях.
- [The Rust Programming Language — Chapter 10.2: Traits](https://doc.rust-lang.org/book/ch10-02-traits.html) — трейты как языковое средство для портов.
- [The Rust Programming Language — Chapter 18: Object-Oriented Programming Features](https://doc.rust-lang.org/book/ch18-00-oop.html) — трейт-объекты, когда адаптер выбирается во время выполнения.
- [The Cargo Book — Workspaces](https://doc.rust-lang.org/cargo/reference/workspaces.html) — навязывайте правило зависимостей границами крейтов.

---

[🏠 Начало](../START-HERE.md) · [← 08.13 Рефакторинг и код-ревью](13-refactoring-and-code-review.md) · [📋 Этап 08](README.md) · [08.15 Приложение двенадцати факторов →](15-the-twelve-factor-app.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
