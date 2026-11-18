<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 11.15 Project: deploy a Rust service](15-project-deploy-a-rust-service.md) · [📋 Phase 11](README.md) · [Phase 12 — Computer systems →](../12-computer-systems/README.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/11-linux-shell-and-containers/16-checkpoint-stage-2-complete.md) <!-- nav -->

# Lesson 11.16 — Checkpoint: Stage 2 complete

> **Phase 11 — Linux, the shell and containers** · Lesson 16 of 16 · ⏱ about 2.5 hours (2 sittings)

## What you will learn

- Whether you own the material of Stage 2 (Phases 07 to 11): algorithms, software design, testing, security, and Linux with containers.
- How to combine several of these skills in practical tasks: a shell tool and a Rust tool that must agree, a property test, a security fix, and a broken deployment to repair.
- Which lessons to review, with direct links.
- What Stage 3 expects from you.

## Before you start

- You finished [Lesson 11.15 — Project: deploy a Rust service](15-project-deploy-a-rust-service.md) and the earlier phases of Stage 2.
- The VM `dev` is running and Docker Desktop is running. Some tasks use both.
- Plan about 2 hours. Do the quiz first **without looking at the lessons**; write your answers down, then open the hidden answers. A wrong answer shows what to review.
- Commands marked "in the VM" run inside `limactl shell dev`; the others run on the Mac. Code goes in `~/projects/rust/code/p11-checkpoint`.

## New words

| Word | Meaning |
|---|---|
| **checkpoint** | A lesson that tests what you learned instead of teaching something new. |
| **property** | A statement that must be true for **all** inputs of some kind, for example "searching a sorted list finds every element that is in it". A property test checks it on many random inputs. |
| **regression test** | A test that keeps a fixed bug from coming back. |
| **troubleshooting** | Finding the cause of a failing system step by step, from the symptom to the first thing that is wrong. |

## Stage 2 at a glance

| Phase | What you learned | Your main project |
|---|---|---|
| 07 Algorithms and data structures | Big-O, arrays and vectors, linked lists, stacks, queues, hash tables, recursion, sorting, binary search, trees, heaps, graphs, dynamic programming | Route finder |
| 08 Software design | KISS, DRY, YAGNI, cohesion and coupling, SOLID, composition, type-driven design, builder, RAII, typestate, layers, twelve-factor | Refactor a messy program |
| 09 Testing, debugging and benchmarking | Test pyramid, test doubles, property-based and snapshot tests, TDD, coverage and mutation tests, fuzzing, debuggers, logging, benchmarks, CI | A well-tested library with CI |
| 10 Security fundamentals | CIA, STRIDE, OWASP Top 10, input validation, injection, passwords, sessions, secrets, cryptography, TLS, supply chain | An encrypted secrets vault |
| 11 Linux, the shell and containers | VM, file system, pipes, users and permissions, processes and signals, systemd, apt, bash, SSH, networking, Docker, Compose | Deploy a Rust service |

The thread through the stage: **a program is more than its code**. It has a cost (Big-O), a shape (design), a proof that it works (tests), a defence (security) and a place to run (Linux, containers).

## Self-test quiz

Twenty questions. Answer each in one or two sentences before you open the answer.

### Phase 07 — Algorithms and data structures

1. Binary search needs one condition on its input and then takes O(log n) steps. Which condition, and why does the work shrink so fast?

<details>
<summary>Answer</summary>

The input must be **sorted**. Each comparison with the middle element throws away half of the remaining range, so n elements need only about log₂ n steps (1,000,000 elements: about 20).

</details>

2. A `HashMap` lookup is O(1) on average. When is it slower, and how slow can it get?

<details>
<summary>Answer</summary>

When many keys land in the same bucket (collisions), the lookup must compare keys one by one. In the worst case, with all keys in one bucket, it is O(n). A good hash function and a resize at a high load factor keep the average at O(1).

</details>

3. Which graph search finds the shortest path (fewest edges) in an unweighted graph, and what data structure does it use?

<details>
<summary>Answer</summary>

Breadth-first search (BFS). It uses a queue and visits the nodes in order of their distance from the start, so the first time it reaches a node is along a shortest path. Depth-first search uses a stack (or recursion) and does not give that guarantee.

</details>

4. What is the base case of a recursive function, and what happens if there is none?

<details>
<summary>Answer</summary>

The base case is the input for which the function answers directly without calling itself again. Without it (or if the input never reaches it) the calls pile up on the call stack until it overflows and the program crashes.

</details>

### Phase 08 — Software design

5. State KISS, DRY and YAGNI in one sentence each.

<details>
<summary>Answer</summary>

KISS: keep it simple, choose the simplest solution that works. DRY: do not repeat yourself, each piece of knowledge has one place. YAGNI: you aren't gonna need it, do not build features before they are needed.

</details>

6. Rust has no class inheritance. What do you use instead, and why is it preferred?

<details>
<summary>Answer</summary>

Composition (a struct holds other values) and traits (shared behaviour). It couples types loosely: you combine small parts and swap them, instead of inheriting a whole tree of behaviour that you cannot change.

</details>

7. Which twelve-factor rule says where configuration lives, and how did the Stage 2 services follow it?

<details>
<summary>Answer</summary>

Factor III: store configuration in the **environment**, not in the code. `p11-hello` reads `BIND` and `GREETING` from environment variables, and Compose or systemd sets them per deployment.

</details>

### Phase 09 — Testing, debugging and benchmarking

8. Describe the test pyramid and why it has that shape.

<details>
<summary>Answer</summary>

Many fast, small unit tests at the bottom; fewer integration tests in the middle; a few slow end-to-end tests at the top. Lower tests are cheap and pinpoint the failure; higher tests are slow, fragile and tell you only that something is wrong.

</details>

9. What does a property-based test do that an example test does not?

<details>
<summary>Answer</summary>

It states a rule that must hold for all inputs and checks it on many generated inputs, then shrinks a failing input to a small counterexample. It finds cases the author did not think of.

</details>

10. A suite has 100% line coverage. Does that prove the tests are good? What can tell you more?

<details>
<summary>Answer</summary>

No: coverage only shows that lines ran, not that the tests would notice a bug. Mutation testing changes the code (a `<` becomes `<=`) and checks that some test fails; surviving mutants show weak tests.

</details>

11. Why do services log to standard output with levels instead of writing their own files?

<details>
<summary>Answer</summary>

The platform (systemd's journal, Docker's log driver) collects, rotates and routes standard output. Levels (`error`, `warn`, `info`, `debug`) let you choose how much to see without changing the code.

</details>

### Phase 10 — Security fundamentals

12. What do the letters C, I and A stand for in the CIA triad?

<details>
<summary>Answer</summary>

Confidentiality (only the right people read the data), Integrity (data is not changed without permission), Availability (the system works when needed).

</details>

13. What is the root cause of an injection attack, and what is the cure?

<details>
<summary>Answer</summary>

Untrusted input is mixed into a command or query and interpreted as code (SQL, a shell command, a path). The cure is to keep data and code apart: parameterized queries, no `eval`, passing arguments as separate values, and validating input with an allow-list.

</details>

14. How should a server store user passwords?

<details>
<summary>Answer</summary>

Only as a hash made by a slow password-hashing function with a unique random salt for each user (for example Argon2), never as plain text and never with a fast hash such as plain SHA-256.

</details>

15. A password was committed to Git and pushed. What do you do first?

<details>
<summary>Answer</summary>

**Revoke or rotate the secret** immediately (change the password, issue a new key). Removing the commit from history is not enough, because copies may exist. Then clean up the history if needed and find out how it happened.

</details>

### Phase 11 — Linux, the shell and containers

16. What do the three parts of `set -euo pipefail` do?

<details>
<summary>Answer</summary>

`-e` stops the script when a command fails; `-u` makes an unset variable an error; `-o pipefail` makes a pipeline fail if any command in it fails, not only the last one.

</details>

17. A file has mode `640` and belongs to `root:app`. Who can do what?

<details>
<summary>Answer</summary>

Owner `root`: read and write (6). Group `app`: read only (4). Everybody else: nothing (0). A service running as a user in group `app` can read it, others cannot.

</details>

18. For key-based SSH login, which key goes to the server, which stays with you, and where does the server keep it?

<details>
<summary>Answer</summary>

The public key goes to the server, into `~/.ssh/authorized_keys` of the user you log in as. The private key (mode `600`, ideally with a passphrase) never leaves your computer.

</details>

19. Why must the `CMD` of a Dockerfile be in exec form for a service that should stop cleanly?

<details>
<summary>Answer</summary>

In exec form (`CMD ["/app"]`) the program is PID 1 and receives `SIGTERM` from `docker stop`. In shell form `/bin/sh -c` is PID 1 and does not pass the signal on, so Docker waits for the grace period and kills the program.

</details>

20. A container publishes `-p 8080:8080` but `curl` gets "connection reset". The server prints `listening on 127.0.0.1:8080`. What is wrong?

<details>
<summary>Answer</summary>

It listens on the container's loopback. Published traffic arrives on the container's network interface, so nobody hears it. Bind to `0.0.0.0:8080` inside the container, and restrict access with the way you publish (`-p 127.0.0.1:8080:8080`).

</details>

## Practical tasks

Create `~/projects/rust/code/p11-checkpoint` with one folder per task. Commit after each task.

### Task 1 — The same answer from two tools

**Goal:** a shell pipeline and a Rust program that count words must print exactly the same top list, and a script proves it. This joins hash tables (Phase 07), tests (Phase 09) and pipes and robust scripts (Phase 11).

Write, in the VM:

1. `topwords.sh [N]`: reads text from standard input and prints the `N` most frequent words (default 3) with their counts, most frequent first, ties in alphabetical order. Words are letters and digits, case does not matter. Use `tr`, `sort`, `uniq -c`, `head`; strict mode; set `LC_ALL=C`; `N` must be a positive number, otherwise a message on standard error and status 2.
2. A Rust program `p11-topwords` with the same behaviour, using `HashMap`. Put the logic in a function `top_words(text, n)` and write at least three unit tests. Print with `{count:>7} {word}` so it looks like the output of `uniq -c` on Linux.
3. `compare.sh FILE` that runs both for `N` = 1, 3 and 10 and reports `ok` or `FAIL`.

<details>
<summary>Hint</summary>

Shell: `tr -cs '[:alnum:]' '\n' | tr '[:upper:]' '[:lower:]' | sed '/^$/d' | sort | uniq -c | sort -k1,1nr -k2,2 | head -n "$n"`. Rust: split with `text.split(|c: char| !c.is_ascii_alphanumeric())`, count with `*counts.entry(word.to_ascii_lowercase()).or_insert(0) += 1`, then sort a `Vec` by `b.1.cmp(&a.1).then_with(|| a.0.cmp(&b.0))`. Build the Rust program inside the VM (`cargo build --release`), or in the `rust:1.100` image if you prefer. In the script use `diff <(…) <(…)`.

</details>

<details>
<summary>Answer</summary>

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

`topwords/src/main.rs` (a package without dependencies; name it `p11-topwords`):

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

`compare.sh` (the Rust binary is expected at `topwords/target/release/p11-topwords`; adjust `rust_bin` if you build elsewhere):

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

With a small file `sample.txt` with these two lines:

```text
The cat sat on the mat. The dog sat on the log!
A cat and a dog: the best of friends, the best of days.
```

the output of `bash topwords.sh 4 < sample.txt` is:

```text
      6 the
      2 a
      2 best
      2 cat
```

and `cargo test` and `compare.sh` print:

```text
test result: ok. 3 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
ok   - top 1 words are the same
ok   - top 3 words are the same
ok   - top 10 words are the same
```

Also try `bash compare.sh /usr/share/common-licenses/GPL-3` (a longer text): the same three `ok` lines, and `bash topwords.sh 3 < /usr/share/common-licenses/GPL-3` starts with `345 the`.

Why it works: both tools define the same rule (count, then alphabetical) and the same format, so a byte-for-byte `diff` is a strong test. `LC_ALL=C` makes `sort` use byte order, as Rust's string comparison does; without it, the order of ties can depend on the locale. The Rust version does the counting in a `HashMap` in O(n) and sorts only the distinct words; the pipeline sorts every word.

</details>

### Task 2 — Repair a broken deployment

**Goal:** find four mistakes in a Dockerfile and a Compose file from their symptoms. This joins networking, containers, Compose and the security habits.

Take the service `p11-hello` (Lesson 11.13) and use this `Dockerfile` and `compose.yaml` in a copy of the project:

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

Put `Dockerfile`, `compose.yaml` and `target` in `.dockerignore`. Run `docker compose up -d --build` and collect the symptoms with:

```bash
docker compose ps --format '{{.Ports}}'
curl -sS -m 3 http://localhost:8080/; echo "curl status $?"
docker compose exec app id
SECONDS=0; docker compose stop app; echo "took ${SECONDS}s"
docker inspect course-broken-app-1 --format 'exit={{.State.ExitCode}}'
```

Explain each symptom, fix the files, and show that the symptoms are gone. Remove everything afterwards (`docker compose down`, `docker image rm course-broken:1`).

<details>
<summary>Hint</summary>

There are four findings, one per command above. Think of: the address inside the container; who is allowed to reach the published port; the user; and the form of `CMD` (compare the stop time and the exit code with Lesson 11.13). A fifth, smaller point is the order of `COPY` for the build cache.

</details>

<details>
<summary>Answer</summary>

Symptoms of the broken version:

```text
0.0.0.0:8080->8080/tcp, [::]:8080->8080/tcp
curl: (56) Recv failure: Connection reset by peer
curl status 56
uid=0(root) gid=0(root) groups=0(root)
took 3s
exit=137
```

| Symptom | Cause | Fix |
|---|---|---|
| `curl` gets "connection reset" | `ENV BIND=127.0.0.1:8080`: the server listens on the container's loopback only | `ENV BIND=0.0.0.0:8080` |
| the port is published on `0.0.0.0` | `"8080:8080"` opens it to every computer on the network | `"127.0.0.1:8080:8080"` |
| `uid=0(root)` | no `USER`: the service runs as root | `RUN useradd --system --uid 10001 --no-create-home --shell /usr/sbin/nologin app` and `USER app` |
| stop takes the whole grace period, exit `137` | `CMD` in shell form: `/bin/sh -c` is PID 1 and swallows `SIGTERM` | `CMD ["/usr/local/bin/p11-hello"]` |

The fifth point: `COPY . .` before `cargo build` makes every source change rebuild all dependencies; copy `Cargo.toml` and `Cargo.lock` first and build the dependencies in their own layer (Lesson 11.13). The fixed runtime stage:

```dockerfile
FROM debian:trixie-slim
RUN useradd --system --uid 10001 --no-create-home --shell /usr/sbin/nologin app
COPY --from=build /app/target/release/p11-hello /usr/local/bin/p11-hello
USER app
ENV BIND=0.0.0.0:8080
EXPOSE 8080
CMD ["/usr/local/bin/p11-hello"]
```

After the fixes and `ports: ["127.0.0.1:8080:8080"]` the same commands print:

```text
127.0.0.1:8080->8080/tcp
Hello from p11-hello
uid=10001(app) gid=999(app) gr…
took 0s
exit=0
```

Why it works: each symptom points to one layer of the ladder from Lesson 11.11 (bind address), the publishing rule (Lesson 11.12), the user (Lesson 11.13) and the signal path (Lessons 11.5 and 11.12). Note that the health check of Lesson 11.13 would have said "healthy" with the first bug, because it checks from inside; only the `curl` from outside shows the problem.

</details>

### Task 3 — A property test for binary search

**Goal:** write binary search yourself and test it with properties, so that a wrong version cannot hide. This joins Phase 07 and Phase 09.

Create the library `p11-bsearch` (`cargo new --lib`), add `proptest = "1.12"` under `[dev-dependencies]`, and write `binary_search(sorted: &[i32], target: i32) -> Option<usize>` (an index of some element equal to `target`). Write: one ordinary unit test; one property "the result agrees with `contains`"; and one property "every element that is in the slice is found". Then **break the function on purpose** (for example `low = middle + 2` instead of `low = middle + 1`), watch which tests fail, and put it back.

<details>
<summary>Hint</summary>

Use a half-open range `low..high` (`while low < high`, `high = middle`, `low = middle + 1`) and `low + (high - low) / 2` for the middle. For the second property pick the target **from the data**: `pick in any::<prop::sample::Index>()` and `data[pick.index(data.len())]`. A random `i32` almost never equals an element of a random vector, so the first property alone rarely exercises the "found" case.

</details>

<details>
<summary>Answer</summary>

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

With the bug `low = middle + 2`, the run reports `test tests::finds_and_misses ... FAILED` and `test tests::finds_every_present_element ... FAILED`, with a line `Test failed: assertion failed: index.is_some()` and a "minimal failing input". The property `agrees_with_contains` still passes, which is a lesson in itself: random targets are almost never in the data, so that property tests mostly the "not found" side.

Why it works: the first property can only fail when the function returns a wrong answer for an absent or a found element, but it checks the found case rarely; the second property builds the "present" case on purpose. A property is only as strong as the inputs that reach the interesting code. After the experiment, delete the folder `proptest-regressions` if `proptest` created one, or keep it as regression data.

</details>

### Task 4 — Close an injection hole

**Goal:** find and fix a command injection in a script, and write the test that keeps it fixed. This joins Phase 10 (injection, input validation) and Phase 11 (robust scripts).

This script prints a report file, `reports/NAME.txt`:

```bash
#!/bin/bash
# show-report.sh NAME - print the file reports/NAME.txt
name=$1
eval "cat reports/$name.txt"
```

In a scratch folder in the VM, create `reports/jan.txt` and show the attack with a harmless payload (`./show-report.sh 'jan.txt; echo INJECTED; #'`). Then write `show-report.sh` again so that it is safe, and `test-show-report.sh` that checks: a normal name works; the injection payload does **not** run; a name with `../` is refused with status 2.

<details>
<summary>Hint</summary>

`eval` runs its argument as shell code, so `;` ends the command and starts another. Do not build commands from strings: call `cat` directly with `"reports/$name.txt"` as **one argument**. Validate with an allow-list: `[[ $name =~ ^[A-Za-z0-9_-]+$ ]]`. End options with `--`. In the test, run the script in a `mktemp -d` folder with its own `reports/`.

</details>

<details>
<summary>Answer</summary>

The attack prints the report **and** runs the injected command:

```text
january numbers
INJECTED
```

The fixed `show-report.sh`:

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

and `test-show-report.sh`:

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

Run against the original script the test prints:

```text
ok   - normal name works
FAIL - injection is not executed (expected [0], got [1])
FAIL - path trick is refused with status 2 (expected [2], got [1])
```

and against the fixed one:

```text
ok   - normal name works
ok   - injection is not executed
ok   - path trick is refused with status 2
```

Why it works: the allow-list accepts only letters, digits, `_` and `-`, so a name cannot contain `;`, spaces, quotes or `/`; and without `eval` the name is never parsed as code, only used as a file name. Two layers (no code execution, and validation) are the "defence in depth" idea of Phase 10. Running the test against the **old** script first shows that the test can fail, as a good test must.

</details>

## If you got something wrong

| Topic | Review this lesson |
|---|---|
| Big-O, binary search | [Lesson 07.1 — What an algorithm is, and Big-O notation](../07-algorithms-and-data-structures/01-algorithms-and-big-o.md), [Lesson 07.10 — Binary search](../07-algorithms-and-data-structures/10-binary-search.md) |
| Hash tables, collisions | [Lesson 07.6 — How hash tables work](../07-algorithms-and-data-structures/06-hash-tables.md) |
| Recursion and the call stack | [Lesson 07.7 — Recursion and the call stack](../07-algorithms-and-data-structures/07-recursion.md) |
| BFS and DFS | [Lesson 07.13 — Graphs: BFS and DFS](../07-algorithms-and-data-structures/13-graphs-bfs-and-dfs.md) |
| KISS, DRY, YAGNI | [Lesson 08.1 — Good design: KISS, DRY and YAGNI](../08-software-design/01-good-design-kiss-dry-yagni.md) |
| Composition and traits | [Lesson 08.4 — Composition over inheritance](../08-software-design/04-composition-over-inheritance.md) |
| Configuration in the environment | [Lesson 08.15 — The twelve-factor app](../08-software-design/15-the-twelve-factor-app.md) |
| Test pyramid | [Lesson 09.1 — The test pyramid and a testing strategy](../09-testing-debugging-benchmarking/01-the-test-pyramid.md) |
| Property-based tests | [Lesson 09.4 — Property-based testing](../09-testing-debugging-benchmarking/04-property-based-testing.md) |
| Coverage and mutation testing | [Lesson 09.7 — Code coverage and mutation testing](../09-testing-debugging-benchmarking/07-code-coverage-and-mutation-testing.md) |
| Logging | [Lesson 09.10 — Logging and tracing](../09-testing-debugging-benchmarking/10-logging-and-tracing.md) |
| CIA triad | [Lesson 10.1 — The security mindset and the CIA triad](../10-security-fundamentals/01-security-mindset-and-the-cia-triad.md) |
| Injection, input validation | [Lesson 10.4 — Input validation](../10-security-fundamentals/04-input-validation.md), [Lesson 10.5 — Injection attacks](../10-security-fundamentals/05-injection-attacks.md) |
| Password hashing | [Lesson 10.6 — Authentication and password hashing](../10-security-fundamentals/06-authentication-and-password-hashing.md) |
| Secrets | [Lesson 10.8 — Secrets management](../10-security-fundamentals/08-secrets-management.md) |
| Pipes, `sort`, `uniq`, redirection | [Lesson 11.3 — Text tools, pipes and redirection](03-text-tools-pipes-and-redirection.md) |
| Permissions | [Lesson 11.4 — Users, groups and permissions](04-users-groups-and-permissions.md) |
| Signals, exit statuses | [Lesson 11.5 — Processes and signals](05-processes-and-signals.md) |
| systemd units | [Lesson 11.6 — Services with systemd](06-services-with-systemd.md) |
| Strict mode, `trap`, ShellCheck | [Lesson 11.9 — Robust bash scripts](09-robust-bash-scripts.md) |
| SSH keys, `rsync` | [Lesson 11.10 — SSH and remote servers](10-ssh-and-remote-servers.md) |
| Bind address, `ss`, `curl` | [Lesson 11.11 — Networking commands](11-networking-commands.md) |
| Containers, PID 1, volumes | [Lesson 11.12 — Containers and Docker basics](12-containers-and-docker-basics.md) |
| Dockerfile, multi-stage, exec form | [Lesson 11.13 — Dockerfiles for Rust programs](13-dockerfiles-for-rust.md) |
| Compose, restart policy | [Lesson 11.14 — Docker Compose](14-docker-compose.md) |
| Deploy and rollback | [Lesson 11.15 — Project: deploy a Rust service](15-project-deploy-a-rust-service.md) |

If you missed more than five questions, review the lessons in the table, redo one exercise from each, and take the quiz again after a day and again after a week (spaced repetition, as in the Stage 1 checkpoint).

## What's next

**Stage 3 — How computers really work** starts with [Phase 12 — Computer systems](../12-computer-systems/README.md). Until now you used the machine as a given: Rust compiled your code, Linux ran it, containers packaged it. Stage 3 opens the box: how numbers and text are stored in bits and bytes, how the CPU runs instructions, how memory and caches behave, how the linker and loader build a process, how an operating system schedules, isolates and protects programs, and how threads, atomics and async runtimes really work (Phases 12, 13 and 14).

What carries over:

- **The VM `dev` and Docker.** From Phase 13 on, you run system-call, process, namespace and cgroup experiments in the VM (or in a container with the right options), never as root on your Mac. Keep the VM; stop it when you do not use it (`limactl stop dev`).
- **Shell and scripts.** You will measure and compare things with scripts and `hyperfine`; keep your habits: strict mode, quoting, a test for every script.
- **Tests and properties.** Low-level code (bit twiddling, allocators, lock-free structures) needs property tests and tools such as Miri; Phase 09 was the preparation.
- **Big-O is not the whole story.** Phase 12 shows why two O(n) loops can differ by ten times: caches, branch prediction and memory layout.

Before you start Phase 12:

1. **Write a short retrospective** in your learning journal: which three topics of Stage 2 were hardest, which project you are proudest of, and what you will do differently.
2. **Schedule reviews** for the quiz questions that you missed: tomorrow, in a week, in a month.
3. **Clean up your machine:** `docker ps -a`, `docker volume ls`, `docker image ls` show what is left from this phase. Remove your own `course-…` objects by name (not with a prune command). In the VM, remove services you no longer need (`sudo systemctl disable --now …`).
4. **Tag your repository:** `git tag -a stage-2-complete -m "Stage 2 complete"`, and look at your top-level README as a stranger would.

## Sources and further reading

- [The Rust Programming Language](https://doc.rust-lang.org/book/) — the official book; keep it open for Stage 3.
- [Rust Performance Book](https://nnethercote.github.io/perf-book/) — a practical guide that starts where Phase 12 ends.
- [The Linux Command Line (William Shotts)](https://linuxcommand.org/tlcl.php) — a free book that deepens the shell skills of Phase 11.
- [Docker documentation](https://docs.docker.com/) — reference for everything in Lessons 11.12 to 11.14.
- [OWASP Cheat Sheet Series](https://cheatsheetseries.owasp.org/) — short, practical security checklists for the topics of Phase 10.
- [Algorithms, Part I and II (Sedgewick and Wayne) — booksite](https://algs4.cs.princeton.edu/home/) — free companion material for the algorithms of Phase 07.

---

[🏠 Start](../START-HERE.md) · [← 11.15 Project: deploy a Rust service](15-project-deploy-a-rust-service.md) · [📋 Phase 11](README.md) · [Phase 12 — Computer systems →](../12-computer-systems/README.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
