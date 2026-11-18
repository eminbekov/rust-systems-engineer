<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 02.11 Checkpoint](../02-programming-basics/11-checkpoint.md) · [03.1 Memory: the stack and the heap →](01-memory-stack-and-heap.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/03-ownership-and-borrowing/README.md) <!-- nav -->

# Phase 03 — Ownership and borrowing

> **Stage 1 — Foundations: tools, Git and the Rust language** · ⏱ about 2–3 weeks at ~10 hours per week

## What this phase is about

This is the phase where Rust becomes Rust. Every program stores data in memory, and someone must decide when that memory is freed. Some languages leave this to the programmer, which causes crashes and security holes. Others use a cleaner that runs while the program works, which costs speed. Rust uses a third way: a small set of rules, called **ownership**, that the compiler checks before the program ever runs.

You will start by looking at how memory really works: the stack, the heap and pointers. Then you will learn the three ownership rules, what a *move* is, when values are copied and when you must `clone` them, and how to **borrow** a value with a reference instead of taking it. You will meet the **borrowing rules** (many readers or one writer), slices, and the two string types `String` and `&str`.

The compiler will reject your code many times in this phase. That is normal, and it is the point. Each error message is a free lesson. In Lesson 03.7 you will learn to read these messages calmly and fix them with a clear method. This is the most important phase of Stage 1: everything later — traits, threads, async, networking, databases — is built on these ideas.

You will finish with a real command-line tool, a word counter, built the way a Rust programmer builds it: one owner of the text, many functions that only borrow it, tests, formatting, linting and a Git commit for every milestone.

## What you will be able to do

- [ ] Explain the difference between the stack and the heap, and say where a value such as an `i32`, a `String` or a `Box` lives.
- [ ] State the three ownership rules and predict when a value is dropped.
- [ ] Predict whether an assignment or a function call moves, copies or borrows a value, and choose between `Copy`, `.clone()` and a reference.
- [ ] Write functions that take `&T`, `&mut T`, `&str` and `&[T]`, and explain the borrowing rules in your own words.
- [ ] Use string slices correctly, and explain bytes versus characters in UTF-8 text.
- [ ] Read a borrow-checker error (`E0382`, `E0499`, `E0502`, `E0505`, `E0507`, `E0515`, `E0597`, `E0716`) and fix it with the smallest sensible change.
- [ ] Build a small command-line tool that reads a file or standard input, with tests, `cargo fmt`, `cargo clippy` and one Git commit per milestone.

## Lessons

| # | Lesson | Time |
|---|---|---|
| 03.1 | [Memory: the stack and the heap](01-memory-stack-and-heap.md) | 2.5 h |
| 03.2 | [The ownership rules](02-ownership-rules.md) | 2.5 h |
| 03.3 | [Move, copy and clone](03-move-copy-and-clone.md) | 2.5 h |
| 03.4 | [References and borrowing](04-references-and-borrowing.md) | 2.5 h |
| 03.5 | [Mutable references and the borrowing rules](05-mutable-references-and-the-borrow-rules.md) | 3 h |
| 03.6 | [Slices and string types](06-slices-and-string-types.md) | 3 h |
| 03.7 | [Fixing common ownership errors](07-fixing-common-ownership-errors.md) | 3 h |
| 03.8 | [Project: word counter, and checkpoint](08-project-word-counter-and-checkpoint.md) | 3 h |

## Projects in this phase

- **Small experiments** (`code/p03-memory`, `code/p03-ownership`, `code/p03-move-copy-clone`, `code/p03-borrowing`, `code/p03-mut-refs`, `code/p03-slices`, `code/p03-fixing-errors`): one tiny project per lesson, each committed to Git.
- **`code/p03-word-counter`** (Lesson 03.8): a command-line word counter. It counts lines, words, characters and bytes, finds the longest word as a borrowed slice, and counts a chosen word. Five milestones, five commits on a branch, merged and tagged at the end.
- **Checkpoint tasks** (`code/p03-checkpoint`): five short practical tasks with tests that prove you can apply the phase.

## How to study this phase

1. **Type every example yourself** and run it. Ownership is learned by seeing the compiler's reaction, not by reading. Break the examples on purpose and read the new error.
2. **Draw the memory.** For every moved or borrowed variable, sketch the stack on the left and the heap on the right, with arrows. If you cannot draw it, you do not yet understand it. Repeat on paper until the picture comes easily.
3. **Do not silence the compiler with `.clone()`.** First ask: "Who should own this? Does the function only need to read it?" Clone only when two independent owners are truly needed.
4. **Commit after every lesson** and after every project milestone, with a clear message. Use `git diff` to read your own fixes, as shown in Lesson 03.7.
5. **Take your time.** Two weeks is a fair estimate for a reason. If a lesson feels hard, repeat its exercises on the next day; this phase rewards patience and repetition more than any other.

---

[🏠 Start](../START-HERE.md) · [← 02.11 Checkpoint](../02-programming-basics/11-checkpoint.md) · [03.1 Memory: the stack and the heap →](01-memory-stack-and-heap.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
