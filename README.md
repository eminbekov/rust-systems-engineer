# Rust → Systems Engineer

My road from the first line of Rust to working as a systems engineer — learned in the open, one
commit at a time.

This repository is two things:

1. **My learning history.** Every exercise, project and note I make while I study lands here, in
   `code/` and `notes/`, with a commit after each step. My progress is ticked in
   [course/en/PROGRESS.md](course/en/PROGRESS.md).
2. **A complete, free course that anyone can follow.** [course/](course/README.md) holds 353 lessons
   in English and Russian: from "what is a terminal" to operating systems, networks, databases,
   distributed systems, performance and cloud infrastructure, all in Rust. Everything is explained
   inside the lessons, so it works offline; official sources are linked at the end of each lesson.

## Where I am now

| | |
|---|---|
| Started | November 2026 |
| Current phase | [Phase 00 — Getting started](course/en/00-getting-started/README.md) |
| Study time | about 10 hours per week |
| Whole course | about 1,040 hours, about 2 years |

I update this table as I move through the course.

## How this repository is organized

```text
rust-systems-engineer/
├── README.md        ← this page: my goal and my progress
├── LICENSE          ← MIT, for the code
├── code/            ← my exercises and projects, one folder each: code/pNN-<name>
├── notes/           ← my study notes and weekly reviews
└── course/          ← the course itself (CC BY 4.0)
    ├── README.md    ← choose a language
    ├── en/          ← English: START-HERE.md, course map, progress, glossary, 25 phases
    └── ru/          ← Russian: the same structure
```

`code/` and `notes/` grow as I study (the course creates them in Lessons 00.6 and 00.8).

## The course in five stages

| Stage | Phases | What it covers |
|---|---|---|
| 1. Foundations | 00–06 | The terminal, Git, and the Rust language from zero to advanced |
| 2. Engineering fundamentals | 07–11 | Algorithms, software design, testing and benchmarking, security, Linux and containers |
| 3. How computers really work | 12–14 | CPUs and memory, operating systems, concurrency and async |
| 4. Build real systems | 15–18 | Networking, backend services, storage engines, distributed systems |
| 5. Mastery and career | 19–24 | Performance, reliability, Kubernetes, a specialization, capstone projects, career |

**Start:** 🇬🇧 [course/en/START-HERE.md](course/en/START-HERE.md) ·
🇷🇺 [course/ru/START-HERE.md](course/ru/START-HERE.md)

## Use the course yourself

You do not need to fork my history to learn with it:

1. Download the course:

   ```bash
   git clone --depth 1 https://github.com/eminbekov/rust-systems-engineer.git rust-course
   ```

2. Make your own learning folder and copy only the course into it:

   ```bash
   mkdir -p ~/projects/rust && cp -R rust-course/course ~/projects/rust/
   ```

3. Clear my ticks from the progress tracker, so it starts empty for you:

   ```bash
   perl -pi -e 's/^- \[x\]/- [ ]/' ~/projects/rust/course/en/PROGRESS.md ~/projects/rust/course/ru/PROGRESS.md
   ```

4. Open `~/projects/rust/course/en/START-HERE.md` (or `course/ru/START-HERE.md`) and follow the
   **Next →** links. Lesson 01.10 shows how to turn your folder into your own Git repository.

The lessons are written for macOS on an Apple Silicon Mac with RustRover; almost everything also
works on Linux. You need about 10 hours per week. Found a mistake? Open an issue — thank you.

## Commit style

Short, imperative subjects with a Conventional Commits prefix and the phase as the scope, for
example `feat(p02): add the guessing game`, `test(p07): cover empty input`,
`docs(notes): week 3 review`.

## License

- The course text in [`course/`](course/) is licensed under
  [CC BY 4.0](course/LICENSE): you may share and adapt it, also commercially, if you give credit.
- My code in `code/` and the code examples in the lessons are licensed under the
  [MIT License](LICENSE).

## Коротко по-русски

Это мой путь от первой строчки на Rust до системного инженера и одновременно полный бесплатный
курс на английском и русском языках. Начать курс: [course/ru/START-HERE.md](course/ru/START-HERE.md).
