<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 07.20 Checkpoint](../07-algorithms-and-data-structures/20-checkpoint.md) · [08.1 Good design: KISS, DRY and YAGNI →](01-good-design-kiss-dry-yagni.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/08-software-design/README.md) <!-- nav -->

# Phase 08 — Software design: principles, patterns and conventions

> **Stage 2 — Engineering fundamentals** · ⏱ about 4–5 weeks at ~10 hours per week

## What this phase is about

A program that works is only the beginning. Real programs are changed again and again: new features, new rules, bug fixes, new people reading the code. **Software design** is the set of habits that keep this change cheap and safe. This phase teaches those habits in the form that suits Rust: small principles with a clear "before" and "after", and a few patterns written with structs, enums, traits and closures.

You start with the simple principles that every engineer uses daily (keep it simple, do not repeat knowledge, do not build what you do not need), then move to the structure of code: parts that belong together, parts that depend on each other as little as possible, and the five SOLID ideas. Then you learn what Rust offers instead of inheritance, and how to let the compiler refuse wrong programs by choosing good types.

The middle of the phase is a small toolbox of patterns that appear in almost every Rust code base: the builder, RAII guards, state machines with typestate, strategies and dependency injection. After that come the things that make a team's code look consistent (conventions, Clippy, project layout), the skills of refactoring and code review, and the first look at architecture (layers, hexagonal architecture and the twelve-factor app). The phase ends with a project in which you clean up a messy program step by step.

Every principle is taught with a small example that goes wrong, the real problem it causes, a better version, and, just as important, **when not to use it**. Principles are advice, not laws.

## What you will be able to do

- [ ] Explain KISS, DRY and YAGNI, and say when each one should be ignored.
- [ ] Judge a piece of code by its cohesion and coupling, and split a program that "does everything" into parts with one job each.
- [ ] Recognise the five SOLID problems in Rust code, and choose between an enum and a trait for extension.
- [ ] Reuse code with composition, traits and wrappers instead of inheritance.
- [ ] Use types to make invalid states impossible: newtypes, smart constructors, enums with data.
- [ ] Write a builder, an RAII guard, a state machine (with an enum and with typestate) and a strategy.
- [ ] Inject dependencies with traits, and test code with hand-written fakes.
- [ ] Follow Rust naming and API conventions, run Clippy, organise a project, refactor safely under tests and review code.
- [ ] Describe layers, hexagonal architecture and the twelve factors of a service.

## Lessons

| # | Lesson | Time |
|---|---|---|
| 08.1 | [Good design: KISS, DRY and YAGNI](01-good-design-kiss-dry-yagni.md) | 135 min |
| 08.2 | [Cohesion, coupling and separation of concerns](02-cohesion-coupling-and-separation-of-concerns.md) | 135 min |
| 08.3 | [The SOLID principles](03-solid-principles.md) | 2.5 h |
| 08.4 | [Composition over inheritance](04-composition-over-inheritance.md) | 120 min |
| 08.5 | [Type-driven design: make invalid states impossible](05-type-driven-design.md) | 135 min |
| 08.6 | [The builder pattern](06-builder-pattern.md) | 2.5 h |
| 08.7 | [RAII and guards](07-raii-and-guards.md) | 135 min |
| 08.8 | [State machines and the typestate pattern](08-state-machines-and-typestate.md) | 2.5 h |
| 08.9 | [Strategy and dependency injection](09-strategy-and-dependency-injection.md) | 2.5 h |
| 08.10 | [Anti-patterns and code smells](10-anti-patterns-and-code-smells.md) | 2.5 h |
| 08.11 | [Conventions: naming, Clippy and the Rust API Guidelines](11-conventions-naming-clippy-and-api-guidelines.md) | 135 min |
| 08.12 | [Project layout and workspaces](12-project-layout-and-workspaces.md) | 3.5 h |
| 08.13 | [Refactoring and code review](13-refactoring-and-code-review.md) | 120 min |
| 08.14 | [Layers and hexagonal architecture](14-layers-and-hexagonal-architecture.md) | 3 h |
| 08.15 | [The twelve-factor app](15-the-twelve-factor-app.md) | 120 min |
| 08.16 | [Project: refactor a messy program](16-project-refactor-a-messy-program.md) | 6 h |
| 08.17 | [Checkpoint](17-checkpoint.md) | 135 min |

## Projects in this phase

- **Small packages in the lessons** (`code/p08-kiss-dry`, `code/p08-coupling`, `code/p08-solid`, `code/p08-composition`, `code/p08-types`, `code/p08-builder`, `code/p08-raii`, `code/p08-typestate`, `code/p08-strategy`, and more): each lesson has a short follow-along task with tests.
- **[Project 08.16 — Refactor a messy program](16-project-refactor-a-messy-program.md)** (`code/p08-refactor`): take a messy but working program, add tests that pin down its behaviour, and refactor it step by step using the principles and patterns of this phase.

## How to study this phase

1. **Type the examples.** Design ideas feel obvious when you read them and slippery when you must apply them. Type each example, run it, then change it and see what breaks.
2. **Always look for the "when not".** Each lesson ends its principle with a case where the rule hurts. Be able to say that case in your own words; that is the difference between knowing a rule and using it with judgement.
3. **Refactor your own earlier code.** Open a program from Phases 04 to 07 and ask the questions of this phase: what is duplicated knowledge, what is tangled, where could a type prevent a bug? Small, tested changes only.
4. **Tests first.** Almost every exercise asks you to write a test. Write it before you change the code; a refactoring without tests is only a hope.
5. **Do not rush the patterns.** One lesson per sitting is enough. When a pattern feels heavy, that is useful information: ask whether a simpler tool would do.

---

[🏠 Start](../START-HERE.md) · [← 07.20 Checkpoint](../07-algorithms-and-data-structures/20-checkpoint.md) · [08.1 Good design: KISS, DRY and YAGNI →](01-good-design-kiss-dry-yagni.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
