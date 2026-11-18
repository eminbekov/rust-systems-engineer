<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 24.7 Coding and system design interviews](07-coding-and-system-design-interviews.md) · [📋 Phase 24](README.md) · [24.9 Checkpoint: course complete →](09-checkpoint-course-complete.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/24-career-and-open-source/08-keep-learning.md) <!-- nav -->

# Lesson 24.8 — Keep learning: books, papers and habits

> **Phase 24 — Career and open source** · Lesson 8 of 9 · ⏱ about 120 minutes

## What you will learn

- Which parts of your knowledge will age quickly and which will stay useful for decades.
- A reading list for each area of this course: books that experienced systems engineers keep on their desks.
- How to read a research paper in three passes, and a first list of classic systems papers.
- Habits that keep you learning for years, and a small spaced-repetition program in Rust that tells you which
  flashcards to review today.

## Before you start

- You finished [Lesson 24.7 — Coding and system design interviews](07-coding-and-system-design-interviews.md).
- You know the learning journal and the review schedule from
  [Lesson 00.8 — Study routine, notes in Markdown and checkpoint](../00-getting-started/08-study-routine-notes-and-checkpoint.md).
  This lesson turns that schedule into a program.

## New words

| Word | Meaning |
|---|---|
| **fundamentals** | Ideas that change very slowly: how memory, processes, networks, storage and agreement between machines work. |
| **research paper** | An article in which researchers describe a new idea or system and the evidence for it, usually reviewed by other experts before publication. |
| **abstract** | The short summary at the start of a paper. |
| **three-pass method** | Reading a paper three times, each time deeper: first for the idea, then for the content, then for the details. |
| **spaced repetition** | Reviewing material after growing gaps (1, 3, 7, 30 days) so that it stays in long-term memory (Lesson 00.8). |
| **flashcard** | A card with a question on one side and a short answer on the other. |
| **Leitner system** | A way of doing spaced repetition with boxes: a card you remember moves to a box that is reviewed less often; a card you forget goes back to the first box. |
| **day number** | In this lesson: the number of whole days since 1970-01-01. Days are easy to compare and subtract as integers. |
| **UTC** (Coordinated Universal Time) | The world's reference time, without time zones or summer time (Lesson 18.4). |
| **LSM tree** (log-structured merge tree) | A storage structure that collects writes in memory and writes them to disk in sorted batches (Lesson 17.7). |
| **release notes** | The list of changes in a new version of a program; Rust publishes them on its blog for every release. |

## 1. What ages and what stays

Look back at this course. Some things you learned will look different in five years: the newest version of a
crate, the best command-line flag, the popular cloud tool. Other things were true forty years ago and will be true in
forty years: a cache miss is slower than a hit, a disk write is not durable until it is flushed, a network message can
be lost, two machines cannot agree without exchanging messages.

This gives a simple rule for your learning time:

- Learn **tools** when you need them, from their official documentation. They change; keep up only with the ones you use.
- Invest in **fundamentals** deliberately, from books and papers. They pay back for your whole career.

The rest of this lesson is about the second part, and about the habits that keep both going.

## 2. A reading list

You do not need all of these. Pick the books of your role and specialization (Lessons 24.1 and 22.1) and read them
slowly, with a notebook and with code. A book read once with exercises is worth more than three books skimmed.

| Area | Book | Why read it | Course phases |
|---|---|---|---|
| Rust | *The Rust Programming Language* (free online) | Re-read it now: you will notice details you missed as a beginner. | 02–06 |
| Rust | *Rust for Rustaceans*, Jon Gjengset | Idiomatic, intermediate-to-advanced Rust: interface design, unsafe code, testing. | 05, 06, 08 |
| Rust | *Rust Atomics and Locks*, Mara Bos (free online) | Atomics, memory ordering and how locks are built, from the ground up. | 14 |
| Rust | *The Rustonomicon* (free online) | The rules of `unsafe` Rust. | 06, 10 |
| Computer systems | *Computer Systems: A Programmer's Perspective*, Bryant and O'Hallaron | Data representation, machine code, caches, linking, virtual memory. | 12 |
| Operating systems | *Operating Systems: Three Easy Pieces*, Arpaci-Dusseau and Arpaci-Dusseau (free online) | Processes, scheduling, memory, concurrency, file systems, very readable. | 13, 14 |
| Linux | *The Linux Programming Interface*, Michael Kerrisk | The reference for Linux system calls, with examples. | 11, 13 |
| Networking | *High Performance Browser Networking*, Ilya Grigorik (free online) | How the network protocols of the web (Phase 15) behave, with a focus on latency. | 15 |
| Data systems | *Designing Data-Intensive Applications*, Martin Kleppmann | Storage engines, replication, partitioning, transactions, consistency. | 17, 18 |
| Databases | *Database Internals*, Alex Petrov | B-trees, LSM trees and distributed database algorithms in depth. | 17, 18 |
| Performance | *Systems Performance*, Brendan Gregg | Methods and tools for finding where time goes, on Linux. | 19 |
| Reliability | *Site Reliability Engineering* (free online) | How production systems are run: reliability targets, incidents, automation. | 20, 21 |

> 💡 **Tip:** Read with code. After each chapter, write a small program or a test that shows one idea of the
> chapter, in a `code/reading/` folder. If you cannot write one, you have found what to re-read.

## 3. Reading research papers

Many of the ideas in this course started as **research papers**: logical clocks, Raft, LSM trees, the end-to-end
argument. Reading the original is a skill worth learning: it shows you *why* an idea exists and what its authors
assumed.

Papers are dense. A well-known approach, described by S. Keshav in the short article *How to Read a Paper*, is to read
in **three passes**:

1. **First pass (10 minutes).** Read the title, the abstract, the introduction, the section headings and the
   conclusion. Look at the figures. Answer: what problem, what idea, is it relevant for me?
2. **Second pass (about an hour).** Read the whole paper, but skip proofs and details you do not need. Note the main
   claims and the evidence (measurements, comparisons). Write five sentences that summarise it.
3. **Third pass (several hours, only for important papers).** Read as if you had to re-create the work: follow every
   step, question every assumption, and if you can, implement the core idea in a small program.

Most papers deserve only the first pass. A few deserve all three. Here is a first list of classic systems papers that
connect to this course:

| Paper | Year | What it teaches | Course lesson |
|---|---|---|---|
| Lamport, *Time, Clocks, and the Ordering of Events in a Distributed System* | 1978 | Logical clocks and "happened before". | 18.5 |
| Saltzer, Reed and Clark, *End-to-End Arguments in System Design* | 1984 | Where in a layered system a function such as reliability belongs. | 15.1 |
| Lampson, *Hints for Computer System Design* | 1983 | Practical design advice from building real systems. | 08.1 |
| Ghemawat, Gobioff and Leung, *The Google File System* | 2003 | A large distributed file system built for big sequential reads and appends. | 18.1 |
| DeCandia et al., *Dynamo: Amazon's Highly Available Key-value Store* | 2007 | Consistent hashing, replication and eventual consistency in production. | 18.6–18.8 |
| Ongaro and Ousterhout, *In Search of an Understandable Consensus Algorithm* | 2014 | Raft: leader election and log replication. | 18.9–18.11 |

The community project *Papers We Love* collects many more, sorted by topic, and runs meetups where people present
papers.

> 📏 **Convention:** Write a short note for every paper you read past the first pass: the problem, the idea, the
> evidence, what you doubt, and one sentence on how it relates to something you built. These notes become your own
> reference library, and good material for write-ups (Lesson 24.2).

## 4. Habits for the long run

Learning for years works only as a rhythm, not as bursts of motivation. A rhythm that many engineers keep:

| How often | What | Time |
|---|---|---|
| Every week | Read *This Week in Rust*; watch one technical talk; write three lines about it in your journal. | 1 hour |
| Every week | Build or contribute: one small program, one test, one review, one pull request. | 3–5 hours |
| Every six weeks | Read the release notes of the new Rust version on the Rust blog; try one new feature. | 30 minutes |
| Every month | One paper (at least the first pass) or one book chapter with code; one short write-up. | 4 hours |
| Every three months | Rate your skills again (Lesson 24.1); compare with last time; adjust the plan. | 1 hour |

Some principles make the rhythm work:

- **Build to learn.** Reading feels like progress; building shows whether there is any. Every topic gets a small
  program.
- **Teach to learn.** Explaining something in a write-up, a talk or an answer on a forum shows you exactly what you do
  not understand yet.
- **Leave the tutorial loop.** After the basics of a topic, stop following step-by-step guides and build something of
  your own with the official documentation. Getting stuck is part of learning.
- **Review on a schedule.** Knowledge you do not use fades. Spaced repetition (the next section) keeps the important
  facts available.
- **Rest.** Sleep, breaks and holidays are when memories settle. A sustainable pace beats a heroic month followed by
  three months of nothing.

## 5. A spaced-repetition program

Lesson 00.8 suggested reviewing after 1, 3, 7 and 30 days. The **Leitner system** makes this mechanical with boxes:
every card starts in box 1 (reviewed after 1 day); a card you remember moves one box up (3, 7, then 30 days); a card you
forget goes back to box 1.

Keep the cards in a plain text file, `cards.txt`, one card per line: box, the day number of the last review, the
question. Lines starting with `#` are comments:

```text
# box | last review (day number) | question
1 | 20774 | What does `cargo check` do?
2 | 20772 | What is a NUL byte?
3 | 20772 | What does `git blame -s` show?
4 | 20745 | Name the seven steps of a system design interview.
```

A **day number** is the number of whole days since 1970-01-01. Day 20775 is 2026-11-18. In the terminal, `date +%s`
prints the seconds since 1970, so this prints today's day number:

```bash
echo $(( $(date +%s) / 86400 ))
```

```text
20775
```

(You will see a larger number, because you read this later.) Now the program, `src/bin/review.rs`:

```rust
// A tiny spaced-repetition helper: which flashcards are due for review today?

/// Days to wait before the next review, for boxes 1, 2, 3 and 4.
const INTERVALS: [u32; 4] = [1, 3, 7, 30];

#[derive(Debug, PartialEq)]
struct Card<'a> {
    card_box: usize,
    last_review: u32,
    question: &'a str,
}

/// Parses a line `box | last review day | question`; returns None for other lines.
fn parse_card(line: &str) -> Option<Card<'_>> {
    let mut parts = line.splitn(3, '|').map(str::trim);
    let card_box: usize = parts.next()?.parse().ok()?;
    let last_review: u32 = parts.next()?.parse().ok()?;
    let question = parts.next()?;
    if card_box == 0 || card_box > INTERVALS.len() {
        return None;
    }
    Some(Card {
        card_box,
        last_review,
        question,
    })
}

/// A card is due when its waiting time has passed.
fn is_due(card: &Card, today: u32) -> bool {
    today >= card.last_review + INTERVALS[card.card_box - 1]
}

/// After a review: remembered moves one box up (at most the last box), forgotten goes back to box 1.
fn next_box(card_box: usize, remembered: bool) -> usize {
    if remembered {
        (card_box + 1).min(INTERVALS.len())
    } else {
        1
    }
}

fn main() {
    let mut args = std::env::args().skip(1);
    let (Some(path), Some(today)) = (args.next(), args.next()) else {
        eprintln!("usage: review <cards.txt> <today as a day number>");
        std::process::exit(2);
    };
    let Ok(today) = today.parse::<u32>() else {
        eprintln!("today must be a whole number of days, got {today}");
        std::process::exit(2);
    };
    let text = std::fs::read_to_string(&path).unwrap_or_else(|err| {
        eprintln!("cannot read {path}: {err}");
        std::process::exit(2);
    });

    let cards: Vec<Card> = text.lines().filter_map(parse_card).collect();
    let due: Vec<&Card> = cards.iter().filter(|c| is_due(c, today)).collect();
    println!(
        "{} of {} cards are due on day {today}:",
        due.len(),
        cards.len()
    );
    for card in due {
        let up = next_box(card.card_box, true);
        println!("  [box {} -> {up}] {}", card.card_box, card.question);
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn parses_a_card_and_skips_other_lines() {
        let card = parse_card("2 | 100 | What is a NUL byte?").unwrap();
        assert_eq!(
            card,
            Card {
                card_box: 2,
                last_review: 100,
                question: "What is a NUL byte?"
            }
        );
        assert_eq!(parse_card("# box | last | question"), None);
        assert_eq!(parse_card("9 | 100 | a box that does not exist"), None);
    }

    #[test]
    fn due_after_the_interval_of_its_box() {
        let card = Card {
            card_box: 2,
            last_review: 100,
            question: "q",
        };
        assert!(!is_due(&card, 102));
        assert!(is_due(&card, 103));
    }

    #[test]
    fn boxes_move_up_or_back_to_one() {
        assert_eq!(next_box(1, true), 2);
        assert_eq!(next_box(4, true), 4);
        assert_eq!(next_box(3, false), 1);
    }
}
```

**What happens:**

1. `Card<'a>` borrows its question from the text of the file, so no strings are copied; the lifetime `'a` says that a
   card cannot outlive that text (Lesson 05.3).
2. `splitn(3, '|')` splits into at most three parts, so a question may itself contain `|`. Each `?` in `parse_card`
   returns `None` early: for a comment line, the first `parse()` fails and the line is skipped.
3. `is_due` adds the waiting time of the card's box to the day of the last review. Box numbers start at 1, array
   indexes at 0, hence `card_box - 1`.
4. `next_box` implements the Leitner rule; `min(INTERVALS.len())` keeps a remembered card in the last box.
5. `main` uses a `let ... else` on a tuple of two `Option`s: both arguments must be present. In the second
   `let ... else`, the message still uses the original text `today`, because the new `today` (the number) exists only
   after the statement succeeds.
6. `filter_map(parse_card)` keeps only lines that are cards; the output shows each due card and the box it moves to if
   you remember it.

```bash
cargo run -q --bin review -- cards.txt 20775
cargo test --bin review
```

```text
3 of 4 cards are due on day 20775:
  [box 1 -> 2] What does `cargo check` do?
  [box 2 -> 3] What is a NUL byte?
  [box 4 -> 4] Name the seven steps of a system design interview.
```

```text
running 3 tests
test tests::boxes_move_up_or_back_to_one ... ok
test tests::due_after_the_interval_of_its_box ... ok
test tests::parses_a_card_and_skips_other_lines ... ok

test result: ok. 3 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

The card in box 3 was reviewed on day 20772 and waits 7 days, so it is due on day 20779, not today. After the review,
edit the file by hand: new box and today's day number. (Making the program update the file is a good stretch task.)

> 🧪 **Testing:** The rules of the system are the boundaries: due exactly on day `last + interval`, not one day earlier;
> the last box stays the last. The tests check exactly these edges with tiny numbers such as 100, 102 and 103, which
> are easier to reason about than real day numbers.

## Common mistakes

**Collecting resources instead of using them.** A list of fifty bookmarked books teaches nothing. Choose one book and
one paper for this month.

**Reading without writing.** After a week, unwritten ideas are mostly gone. Every reading session ends with a few lines
in the journal or a small program.

**Trying to follow everything.** New crates, tools and opinions appear every day. Follow a few sources well; ignore the
rest without guilt.

**Only learning, never resting.** A plan that needs every evening will break. Plan for about ten hours a week, as in
this course, and keep free days.

## Try it yourself

1. Add `review` to `code/p24-career` and create your own `cards.txt` with ten questions from the quizzes of this
   phase. Use today's day number for all of them, in box 1.
2. Choose one book from Section 2 and one paper from Section 3 for the next month. Write the plan in `notes/reading.md`.
3. Do the first pass of the Raft paper (about 10 minutes) and write the three answers of the first pass.
4. Put the weekly rhythm from Section 4 into your calendar.

## Exercises

### Exercise 1 — A fifth box (easy)

Add a fifth box with a waiting time of 90 days. What changes in the output for day 20775, and which test fails? Fix
the test so that it does not depend on the number of boxes.

<details>
<summary>Hint</summary>

The array type includes its length: `[u32; 5]`. `next_box` and `parse_card` already use `INTERVALS.len()`.

</details>

<details>
<summary>Answer</summary>

```rust
/// Days to wait before the next review, for boxes 1 to 5.
const INTERVALS: [u32; 5] = [1, 3, 7, 30, 90];
```

The last output line becomes `[box 4 -> 5] Name the seven steps of a system design interview.`, and the test
`boxes_move_up_or_back_to_one` fails:

```text
assertion `left == right` failed
  left: 5
 right: 4
```

Replace its middle line with:

```rust
        assert_eq!(next_box(INTERVALS.len(), true), INTERVALS.len());
```

Why it works: the test now states the rule ("the last box stays the last") instead of one particular number, so it stays
correct for any number of boxes.

</details>

### Exercise 2 — Today from the clock (medium)

Make the day argument optional: without it, the program computes today's day number itself. Write a function
`today() -> u32` and a test for it.

<details>
<summary>Hint</summary>

`std::time::SystemTime::now().duration_since(std::time::UNIX_EPOCH)` returns the time since 1970-01-01 in UTC as a
`Duration`; `as_secs()` gives seconds. Divide by 86,400. A test cannot know the exact date, but it can check that the
result is not earlier than the day this lesson was written.

</details>

<details>
<summary>Answer</summary>

```rust
/// Today as a day number: whole days since 1970-01-01 (UTC).
fn today() -> u32 {
    let since_1970 = std::time::SystemTime::now()
        .duration_since(std::time::UNIX_EPOCH)
        .expect("the clock is set before 1970");
    (since_1970.as_secs() / 86_400) as u32
}

fn main() {
    let mut args = std::env::args().skip(1);
    let Some(path) = args.next() else {
        eprintln!("usage: review <cards.txt> [today as a day number]");
        std::process::exit(2);
    };
    let today = match args.next() {
        None => today(),
        Some(text) => text.parse::<u32>().unwrap_or_else(|_| {
            eprintln!("today must be a whole number of days, got {text}");
            std::process::exit(2);
        }),
    };
    let text = std::fs::read_to_string(&path).unwrap_or_else(|err| {
        eprintln!("cannot read {path}: {err}");
        std::process::exit(2);
    });

    let cards: Vec<Card> = text.lines().filter_map(parse_card).collect();
    let due: Vec<&Card> = cards.iter().filter(|c| is_due(c, today)).collect();
    println!(
        "{} of {} cards are due on day {today}:",
        due.len(),
        cards.len()
    );
    for card in due {
        let up = next_box(card.card_box, true);
        println!("  [box {} -> {up}] {}", card.card_box, card.question);
    }
}

#[test]
fn today_is_after_this_lesson_was_written() {
    // Day 20775 is 2026-11-18.
    assert!(today() >= 20_775);
}
```

The test goes into `mod tests`. Why it works: `UNIX_EPOCH` is 1970-01-01 00:00 UTC, so whole days since then are the
day number; dividing seconds by 86,400 drops the part of the current day. `duration_since` fails only if the computer's
clock is set before 1970, which `expect` reports clearly. Note that the day changes at midnight UTC, not at your local
midnight.

</details>

### Exercise 3 — Your next three months (easy, no code)

Write a reading plan for the next three months in `notes/reading.md`: one book (with the chapters per month), one paper
per month with the pass you aim for, and the small program you will write for each.

<details>
<summary>Hint</summary>

Choose by your role from Lesson 24.1. Keep the plan small enough for about four hours a month of reading.

</details>

<details>
<summary>Answer</summary>

An example for someone aiming at database roles:

```text
Book: Database Internals, part I (storage engines)
  Month 1: chapters 1-3   -> program: a page layout with slotted records and a test
  Month 2: chapters 4-6   -> program: B-tree node split with property tests
  Month 3: chapters 7-8   -> program: a write-ahead log with crash tests (compare with my Phase 17 code)
Papers:
  Month 1: The Log-Structured Merge-Tree, first pass
  Month 2: Dynamo, second pass, 5-sentence summary
  Month 3: Raft, third pass: compare with my Phase 18 implementation
```

A good plan is specific (chapters, passes), small, and every item produces something you can show.

</details>

## Quiz

1. Which kind of knowledge should get most of your deliberate learning time, and why?

<details>
<summary>Answer</summary>

Fundamentals (memory, processes, networks, storage, distributed agreement), because they change slowly and apply to every
tool. Tools are learned when needed from their documentation.

</details>

2. What do you do in the first pass of reading a paper?

<details>
<summary>Answer</summary>

Read the title, abstract, introduction, headings and conclusion, and look at the figures, to learn the problem and the
idea and decide whether to read further.

</details>

3. In the Leitner system, what happens to a card you forget?

<details>
<summary>Answer</summary>

It goes back to the first box, so you see it again after one day.

</details>

4. A card is in box 2 (3 days) and was last reviewed on day 100. On which day is it due?

<details>
<summary>Answer</summary>

On day 103 and every day after it until you review it. On day 102 it is not due yet.

</details>

5. Why does `parse_card` use `splitn(3, '|')` instead of `split('|')`?

<details>
<summary>Answer</summary>

So that the question, the third part, may itself contain `|` characters: everything after the second `|` stays one part.

</details>

## Summary

- Tools change; fundamentals stay. Learn tools when needed, invest in fundamentals on purpose.
- Read a few good books for your role slowly, with code; the reading list covers every area of the course.
- Read papers in three passes; most deserve only the first, a few deserve all three. Start with the classics that match
  your lessons.
- Keep a rhythm: weekly reading and building, Rust release notes every six weeks, a paper a month, a skills check every
  three months.
- Build to learn, teach to learn, review on a schedule, and rest.
- A small Leitner-box program keeps your flashcards on schedule.

## Sources and further reading

- [Operating Systems: Three Easy Pieces](https://pages.cs.wisc.edu/~remzi/OSTEP/), [Rust Atomics and Locks](https://marabos.nl/atomics/),
  [The Rustonomicon](https://doc.rust-lang.org/nomicon/) and [High Performance Browser Networking](https://hpbn.co/) — four
  books from the list that are free to read online.
- [Rust for Rustaceans](https://rust-for-rustaceans.com/), [Computer Systems: A Programmer's Perspective](https://csapp.cs.cmu.edu/),
  [The Linux Programming Interface](https://man7.org/tlpi/), [Database Internals](https://www.databass.dev/),
  [Systems Performance](https://www.brendangregg.com/systems-performance-2nd-edition-book.html) — the book pages of other
  titles from the list.
- [Time, Clocks, and the Ordering of Events in a Distributed System](https://lamport.azurewebsites.net/pubs/time-clocks.pdf),
  [In Search of an Understandable Consensus Algorithm](https://raft.github.io/raft.pdf),
  [Dynamo](https://www.allthingsdistributed.com/files/amazon-dynamo-sosp2007.pdf),
  [End-to-End Arguments in System Design](https://web.mit.edu/Saltzer/www/publications/endtoend/endtoend.pdf),
  [The Google File System](https://research.google/pubs/the-google-file-system/) and
  [Hints for Computer System Design](https://www.microsoft.com/en-us/research/publication/hints-for-computer-system-design/) —
  the papers from Section 3.
- [Papers We Love](https://github.com/papers-we-love/papers-we-love) — a community collection of papers by topic.
- [The Rust Blog](https://blog.rust-lang.org/) and [This Week in Rust](https://this-week-in-rust.org/) — release notes and
  weekly news.
- [std::time::SystemTime](https://doc.rust-lang.org/std/time/struct.SystemTime.html) — the clock used in Exercise 2.

---

[🏠 Start](../START-HERE.md) · [← 24.7 Coding and system design interviews](07-coding-and-system-design-interviews.md) · [📋 Phase 24](README.md) · [24.9 Checkpoint: course complete →](09-checkpoint-course-complete.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
