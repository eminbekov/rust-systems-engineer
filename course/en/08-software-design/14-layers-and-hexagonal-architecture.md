<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 08.13 Refactoring and code review](13-refactoring-and-code-review.md) · [📋 Phase 08](README.md) · [08.15 The twelve-factor app →](15-the-twelve-factor-app.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/08-software-design/14-layers-and-hexagonal-architecture.md) <!-- nav -->

# Lesson 08.14 — Layers and hexagonal architecture

> **Phase 08 — Software design: principles, patterns and conventions** · Lesson 14 of 17 · ⏱ about 3 hours (2 sittings)

## What you will learn

- What *architecture* means for a small program: deciding which code may know about which other code.
- Layered architecture, and the problem it has when business rules depend on the database.
- Hexagonal architecture (ports and adapters): the core defines traits, the outside world plugs in.
- How to build a small task-list program this way, test its rules without any file, and check the dependency rule with `grep`.
- When this structure is too much.

## Before you start

- You finished [Lesson 08.13 — Refactoring and code review](13-refactoring-and-code-review.md).
- You know separation of concerns ([Lesson 08.2](02-cohesion-coupling-and-separation-of-concerns.md)), the dependency inversion principle ([Lesson 08.3](03-solid-principles.md)), dependency injection ([Lesson 08.9](09-strategy-and-dependency-injection.md)) and the project layout of [Lesson 08.12](12-project-layout-and-workspaces.md).
- Create the package: `cargo new code/p08-tasks`, then `cd code/p08-tasks`, `cargo add thiserror@2` and `cargo add --dev tempfile@3`.

## New words

| Word | Meaning |
|---|---|
| **architecture** | The big structure of a program: its parts and the rules about which part may depend on which. |
| **layer** | A group of code with one kind of job, placed in an order, where code may use the layers below it. |
| **domain** | The part of the program that holds the business rules and concepts, such as "a task has a non-empty title". It knows nothing about screens, files or networks. |
| **use case** (application service) | One thing a user wants to do with the program: "add a task", "complete a task". It uses the domain to do it. |
| **infrastructure** | Code that talks to the outside world: files, databases, networks, the clock. |
| **port** | A trait defined by the core that describes what it needs from the outside (`TaskRepository`) or what it offers. |
| **adapter** | A type that connects a port to real technology. `FileRepo` adapts the port `TaskRepository` to a text file. |
| **driving adapter** | An adapter that *calls* the core: a command-line interface, a web handler, a test. |
| **driven adapter** | An adapter that the core *calls* through a port: storage, email, the clock. |
| **hexagonal architecture** (ports and adapters) | A way of building a program with the domain and use cases in the middle, and ports and adapters around them, so that dependencies point inwards. |
| **dependency rule** | The rule of this architecture: source code dependencies point towards the domain; the domain never mentions adapters. |
| **composition root** | The one place (usually `main`) where concrete adapters are chosen and plugged into the core. |
| **CLI** (Command-Line Interface) | A program that you use by typing commands in a terminal. |
| **HTTP** (HyperText Transfer Protocol) | The protocol that web browsers and web services use to talk to each other. |

## 1. What architecture is for

Design principles (Lessons 08.1 to 08.9) are about functions and types. **Architecture** is the same thinking for the whole program: which parts exist, and which part may know about which. Two goals drive
almost every decision:

1. **Change one thing without rewriting the rest.** Replace a text file by a database, or a command line by a web page, and the business rules stay untouched.
2. **Test the rules without the real world.** "A title may not be empty" must be testable in microseconds, without a file, a network or a clock.

If your rules call `std::fs` or print to the screen, both goals are lost.

## 2. Layered architecture

The oldest answer is **layers**: sort the code by kind of job, and let each layer use only the layer below.

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

This already beats one giant `main`: the interface has no business rules, and the rules have no screen code. But the classic picture has a flaw: the **domain sits above infrastructure, so it calls it**. The
domain's `complete_task` would call `database::save(...)`. Now the rules depend on the database: you cannot test them without one, and replacing the database means opening the rules. You rarely need four layers; two or three
are enough. What matters is the **direction of the arrows**.

## 3. Hexagonal architecture: ports and adapters

The fix is to invert the arrow between the core and the infrastructure (the dependency inversion principle of Lesson 08.3). The core does not call the database. The core *declares what
it needs* as a trait, and the infrastructure implements that trait.

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

- The **core** is the domain plus the use cases. It is plain Rust: no `std::fs`, no printing, no framework.
- A **port** is a trait that belongs to the core. `TaskRepository` says "I need something that can add, list and save tasks". The core owns the trait, so it is shaped by the needs of the core, not by the database.
- An **adapter** is a type in the outside layer that implements a port or calls the core. `FileRepo` implements `TaskRepository` with a text file. `InMemoryRepo` implements it with a vector. The CLI is an
  adapter on the other side: it turns command-line arguments into calls of the use cases.
- The **composition root**, `main`, is the only place that knows all concrete types. It chooses `FileRepo`, hands it to `TaskService`, and starts the CLI.

> 📏 **Convention:** The **dependency rule**: `use` lines point inwards. `domain` imports nothing from the rest of the program. `ports` and `service` import `domain` (and `ports`). Only `adapters` and `main` import
> `std::fs`, `std::env` or other technology. You can check the rule with `grep`, as you will see in Section 5.

## 4. A worked example: tasks

You will build a task list with the commands `add`, `done` and `list`. The files:

```text
p08-tasks/
├── Cargo.toml
├── src/
│   ├── lib.rs          ← declares the modules
│   ├── domain.rs       ← Title, Task, DomainError            (core)
│   ├── ports.rs        ← TaskRepository, StoreError          (core)
│   ├── service.rs      ← TaskService: the use cases          (core)
│   ├── adapters/
│   │   ├── mod.rs
│   │   ├── memory.rs   ← InMemoryRepo  (driven adapter)
│   │   └── file.rs     ← FileRepo      (driven adapter)
│   └── main.rs         ← CLI + composition root (driving adapter)
└── tests/
    └── contract.rs     ← the same checks for every adapter
```

`src/lib.rs`:

```rust
//! A tiny task list built as ports and adapters.

pub mod adapters;
pub mod domain;
pub mod ports;
pub mod service;
```

**The domain.** `src/domain.rs` has the concepts and the rules, and uses nothing but `thiserror` (Lesson 05.11) for error messages:

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

**What happens:** `Title` is a *newtype* with a private field; the only way to get one is `Title::new`, which enforces the rules (Lesson 08.5), so a `Task` always has a valid title. `DomainError` speaks the vocabulary of the
domain and says nothing about files or screens. The `id` is chosen by the repository, because how identifiers are made depends on the storage.

**The port.** `src/ports.rs`:

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

**What happens:** `TaskRepository` is the door through which the core talks to storage, in the vocabulary of the core (`add`, `all`, `save`) rather than of a file. `StoreError` is also owned by the core: an adapter must
translate its own errors (a `std::io::Error`, a database error) into it, so technology-specific error types never travel into the core.

**The use cases.** `src/service.rs`:

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

**What happens:**

1. `TaskService<R: TaskRepository>` is generic over any repository. Its public methods `add`, `complete` and `pending` are the *driving port*: what the outside world may ask the core to do.
2. `add` asks the domain to validate the title, then asks the repository to store it; `?` converts `DomainError` and `StoreError` into `ServiceError` thanks to `#[from]`.
3. `complete` holds the whole rule "find the task, mark it done, save it". It has no file code and no `println!`.
4. A generic parameter means static dispatch (Lesson 05.1). `Box<dyn TaskRepository>` would work too (Lesson 06.1) and is better when the adapter is chosen at run time.

**The adapters.** `src/adapters/mod.rs` only declares the two modules (`pub mod file;` and `pub mod memory;`). `src/adapters/memory.rs` is the simplest adapter, and the one the tests use:

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

`src/adapters/file.rs`, the real storage. The file has one line per task, `0 title` or `1 title`, and the identifier is the line number:

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

**What happens:** both adapters implement the same trait, so the service cannot tell them apart. `FileRepo` is the only place with `std::fs`: a missing file means an empty list, and every other `io::Error` (or bad line) becomes a
`StoreError::Storage`. The `.expect("fewer than 2^32 tasks")` marks an assumption about our own program, not about input (Lesson 08.10).

**The driving adapter and composition root.** `src/main.rs`:

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

**What happens:** the file name comes from the environment variable `TASKS_FILE` (default `tasks.txt`): configuration is read at the edge, never in the core (Lesson 08.15). `main` chooses `FileRepo`, gives it to
`TaskService::new`, turns the arguments into service calls, prints, and picks the exit status (`0` success, `1` error, `2` wrong usage). To switch to a database you write `DatabaseRepo`, implement `TaskRepository`, and change **one line** in `main`.

Try it (the `export` line keeps the data file in a throw-away place):

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

(The first `list` prints nothing because the file does not exist yet.)

## 5. Testing, and checking the dependency rule

**Rules without a disk.** The service tests use `InMemoryRepo`, and a tiny `BrokenRepo` that always fails. Add this at the end of `src/service.rs`:

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

**What happens:** the first three tests check use cases and domain rules with the in-memory adapter: microseconds, no files. `BrokenRepo` is a *test double* (Lesson 08.9 introduced them; Lesson 09.3 goes deeper): a few lines that make the storage error
path testable; try testing "the disk is full" with a real file. The test module may import an adapter, because tests play the role of `main`.

**One contract for all adapters.** If two adapters claim to implement the same port, they must behave the same. `tests/contract.rs` runs one function against both:

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

A new adapter (a database) would be added to this test with one more line, and it would have to pass. That is how ports stay honest.

**Checking the dependency rule.** The rule says that the core imports no technology. Ask `grep` for any `use` line in the core files that mentions an adapter or a technology module:

```bash
grep -n "^use crate::adapters\|^use std::fs\|^use std::env" src/domain.rs src/ports.rs src/service.rs
```

It prints nothing (and `grep` exits with status 1, "no match"). The line `use crate::adapters::memory::InMemoryRepo;` exists in `service.rs`, but it is indented inside `mod tests`, so the pattern `^use` does not match it.
In a real project you can run this command in CI (Continuous Integration, an automatic check on a server). In a workspace (Lesson 08.12) the compiler can enforce it: put the core in its own crate that does not list the adapters' dependencies.

## 6. Is this too much?

This structure costs files, traits and a little indirection. Use it when it pays:

- **Use it** when the program has real business rules; when storage or the interface will change or exist in several forms (file now, database later; command line and web); when you want fast tests of rules.
- **Do not use it** for a 100-line script, a one-off tool, or a program that moves data from A to B without rules. A port with one implementation and no test double earns nothing (Lesson 08.1: YAGNI, "You Aren't Gonna Need It"). Also avoid
  pass-through layers: a "service" whose every method forwards to one repository method adds a layer and no value.
- **Copy the arrows, not the folder names.** `domain`, `ports`, `adapters` as folders are not an architecture; the rule that `use` lines point inwards is.
- **Introduce a port when** you have a second implementation, need a test double, or expect the technology to change. Until then a plain function in a module is fine, and you can extract the trait later (Lesson 08.13).

## Common mistakes

1. **Business rules in `main` or in adapters.** The CLI should parse and print, nothing else.
2. **Letting `std::io::Error` or a database error type into the core.** Convert at the adapter boundary into the core's own error type.
3. **Shaping the port after the technology** (`fn run_sql(&self, query: &str)`). A port speaks the language of the core.
4. **One trait for everything.** A `Repository` with 30 methods is a god type. Prefer small ports (Interface Segregation, Lesson 08.3).

## Try it yourself

1. Build `p08-tasks` and run the commands of the "Try it" block. Compare your output.
2. Break `FileRepo` on purpose (write `1 ` where `0 ` belongs in `write_all`) and run `cargo test`: the contract test fails, the service tests do not.
3. Run the `grep` command of Section 5. Add `use std::fs;` to `domain.rs`, run it again to see the violation, and remove the line.
4. Commit: `git add code/p08-tasks && git commit -m "feat(p08): task list with ports and adapters"`.

## Exercises

### Exercise 1 — Which side of the line? (easy)

Say for each item whether it belongs to the core (domain / use cases), to a port, or to an adapter: (a) the rule that a title may not be empty; (b) the function that reads `tasks.txt`; (c) `trait Clock { fn now(&self) -> u64; }`;
(d) the code that turns `done 1` from the command line into a call of `complete(1)`; (e) a struct `FixedClock(u64)` used in tests.

<details>
<summary>Hint</summary>

Ask: does it mention a technology (file, terminal, network)? Does the core own the trait?

</details>

<details>
<summary>Answer</summary>

(a) the core (domain). (b) an adapter (`FileRepo`, driven). (c) a port: the core owns the description of what it needs from a clock. (d) an adapter (the CLI, driving). (e) an adapter for the `Clock` port, written for tests.

</details>

### Exercise 2 — Add a use case (medium)

Add `TaskService::reopen(id)`, which marks a finished task as not done, and write a test with `InMemoryRepo`. Do you have to change the port or any adapter?

<details>
<summary>Hint</summary>

`reopen` looks like `complete`, with `done = false`. Think about which operations the repository already offers.

</details>

<details>
<summary>Answer</summary>

No change to the port or the adapters: `all` and `save` are enough. In `impl<R: TaskRepository> TaskService<R>`:

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

and in `mod tests`:

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

Why it works: a new use case is new code in the core only. `cargo test --lib` shows 5 passed. (The duplicated "find a task" code in `complete` and `reopen` is a refactoring candidate: a private method `find(id)`; Lesson 08.13.)

</details>

### Exercise 3 — A second port (hard)

The program should celebrate when a task is completed. Add a port `Notifier` with `fn task_completed(&self, task: &Task);`, a second type parameter in `TaskService`, a `RecordingNotifier` fake for the tests, and a `ConsoleNotifier` adapter in `main`.

<details>
<summary>Hint</summary>

The port goes into `ports.rs`. `TaskService<R: TaskRepository, N: Notifier>` keeps the notifier in a second field; `complete` calls it after a successful `save`. The fake needs interior mutability (`RefCell<Vec<String>>`, Lesson 06.4), because `task_completed` takes `&self`.

</details>

<details>
<summary>Answer</summary>

In `src/ports.rs`:

```rust
/// Tells someone that something happened. The core defines it; adapters implement it.
pub trait Notifier {
    /// Called after a task has been completed.
    fn task_completed(&self, task: &Task);
}
```

In `src/service.rs`, `use crate::ports::{Notifier, StoreError, TaskRepository};`, the struct becomes `pub struct TaskService<R: TaskRepository, N: Notifier> { repo: R, notifier: N }`, the `impl` header is
`impl<R: TaskRepository, N: Notifier> TaskService<R, N>`, `new(repo: R, notifier: N)` stores both, and `complete` calls `self.notifier.task_completed(&task);` right after `self.repo.save(&task)?;`.

In the tests (add `use std::cell::RefCell;`, and every `TaskService::new(x)` becomes `TaskService::new(x, RecordingNotifier::default())`):

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

In the first test, after `service.complete(second.id).unwrap();`, add
`assert_eq!(service.notifier.messages.borrow().as_slice(), ["done: test lesson"]);`. In `main.rs`, write `struct ConsoleNotifier;` with an `impl Notifier for ConsoleNotifier` whose method prints
`well done! finished: ...`, and call `TaskService::new(FileRepo::new(path), ConsoleNotifier)`. All the earlier tests keep passing, and `cargo clippy --all-targets -- -D warnings` is clean.

Why it works: the core now has two doors, and each adapter plugs into one.

</details>

## Quiz

1. What is the dependency rule?

<details>
<summary>Answer</summary>

Source-code dependencies point towards the domain. The core (domain and use cases) never imports adapters or technology; adapters and `main` import the core.

</details>

2. What is the difference between a port and an adapter?

<details>
<summary>Answer</summary>

A port is a trait owned by the core that describes what it needs (or offers). An adapter is a type in the outer layer that implements a port or calls the core, connecting it to real technology.

</details>

3. Why does `StoreError` belong to the core and not to `FileRepo`?

<details>
<summary>Answer</summary>

So the core does not depend on any technology's error type. Adapters convert their own errors (`io::Error`, database errors) into `StoreError`, and the core handles only its own vocabulary.

</details>

4. When is a port not worth it?

<details>
<summary>Answer</summary>

When there is only one implementation, no test double is needed, and the technology will not change: a plain function is simpler. Add the trait when a second implementation or a test needs it.

</details>

## Summary

- **Architecture** is the rule about which parts of a program may depend on which. The goals are changing one part without rewriting the rest and testing rules without the real world.
- **Layered architecture** orders code by job and lets each layer use the layer below; its weakness is that the domain ends up depending on storage.
- **Hexagonal architecture** inverts that arrow: the core owns **ports** (traits), **adapters** implement them (driven) or call the core (driving), and `main` is the **composition root** that wires them.
- The **dependency rule**: `use` lines point inwards. The core imports no `std::fs`, no printing, no framework; check it with `grep` or with crate boundaries.
- Errors cross boundaries in the core's own types. Adapters translate technology errors.
- Test rules with an in-memory adapter, test error paths with a failing double, and run one **contract test** against all adapters.
- Do not build this for tiny programs, and do not add a trait before it has a reason.

## Sources and further reading

- [Alistair Cockburn — Hexagonal architecture](https://alistair.cockburn.us/hexagonal-architecture/) — the original article that introduced ports and adapters.
- [Martin Fowler — Presentation Domain Data Layering](https://martinfowler.com/bliki/PresentationDomainDataLayering.html) — a short article on the classic layers.
- [The Rust Programming Language — Chapter 10.2: Traits](https://doc.rust-lang.org/book/ch10-02-traits.html) — traits as the language tool for ports.
- [The Rust Programming Language — Chapter 18: Object-Oriented Programming Features](https://doc.rust-lang.org/book/ch18-00-oop.html) — trait objects, when the adapter is chosen at run time.
- [The Cargo Book — Workspaces](https://doc.rust-lang.org/cargo/reference/workspaces.html) — enforce the dependency rule with crate boundaries.

---

[🏠 Start](../START-HERE.md) · [← 08.13 Refactoring and code review](13-refactoring-and-code-review.md) · [📋 Phase 08](README.md) · [08.15 The twelve-factor app →](15-the-twelve-factor-app.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
