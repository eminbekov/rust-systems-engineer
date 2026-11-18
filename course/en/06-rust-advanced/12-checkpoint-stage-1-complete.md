<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 06.11 Project: multithreaded web server](11-project-multithreaded-web-server.md) · [📋 Phase 06](README.md) · [Phase 07 — Algorithms and data structures →](../07-algorithms-and-data-structures/README.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/06-rust-advanced/12-checkpoint-stage-1-complete.md) <!-- nav -->

# Lesson 06.12 — Checkpoint: Stage 1 complete

> **Phase 06 — Rust advanced** · Lesson 12 of 12 · ⏱ about 3 hours (2 sittings)

## What you will learn

- Whether you really own the material of Stage 1 (Phases 00–06): tools, Git, the Rust language from variables to async and FFI.
- How to check the health of your whole learning repository: history, tags, ignored files, and quality gates on every project.
- Which topics to review, with direct links to the lessons.
- Which tools and habits to bring into Stage 2.

## Before you start

- You finished [Lesson 06.11 — Project: multithreaded web server](11-project-multithreaded-web-server.md) and merged the `p06-webserver` branch.
- You have the code of earlier projects in `~/projects/rust/code/`.
- Plan about 90 minutes. Do the quiz first **without looking at the lessons**. Write your answers on paper or in your notes file, then open the hidden answers. A wrong answer is useful: it tells you what to review.

## New words

| Word | Meaning |
|---|---|
| **checkpoint** | A lesson that checks what you learned, instead of teaching something new. |
| **Stage** | One of five big groups of phases in this course. Stage 1 is "Foundations: tools, Git and the Rust language" (Phases 00–06). |
| **quality gate** | A check that every change must pass before you accept it: here `cargo fmt --check`, `cargo clippy -- -D warnings` and `cargo test`. |
| **tag** | A permanent, named label on one commit in Git, for example `stage-1-complete`. |
| **retrospective** | A short look back: what went well, what was hard, what to change. |
| **portfolio** | A collection of your work that you can show to other people: your repository with projects, READMEs and tests. |
| **CI** (Continuous Integration) | A server that runs your quality gates automatically on every push. You will set it up in Phase 09. |
| **spaced repetition** | A study method: you review a topic again after one day, then a week, then a month, so that you remember it for a long time. |

## Stage 1 at a glance

| Phase | What you learned | Your main project |
|---|---|---|
| 00 Getting started | What programming is, how a computer runs a program, the terminal, Rust and RustRover, reading errors | Hello, world |
| 01 Git and version control | Commits, history, undoing, branches, conflicts, GitHub and GitLab, pull requests, conventions | Your learning repository |
| 02 Programming basics | Variables, types, functions, control flow, input, tests, debugging | Guessing game |
| 03 Ownership and borrowing | Stack and heap, ownership, move/copy/clone, references, slices | Word counter |
| 04 Structs, enums, collections, errors | Structs, enums, `Option`, `match`, modules, `Vec`, `String`, `HashMap`, `Result` | Phone book |
| 05 Rust intermediate | Generics, traits, lifetimes, closures, iterators, docs, Cargo, smart pointers, threads, crates, benchmarks | minigrep |
| 06 Rust advanced | Trait objects, advanced traits, patterns, interior mutability, unsafe and Miri, macros, async, FFI (Foreign Function Interface) | Multithreaded web server |

## Self-test quiz

Answer from memory. Each answer is hidden; open it after you wrote yours.

**Phase 00–01: tools and Git**

1. What is the difference between source code and a binary, and which tool turns one into the other in a Rust project?

<details>
<summary>Answer</summary>

Source code is text that you write and read. A binary is the machine code that the CPU (Central Processing Unit, the chip that runs instructions) runs. The compiler, `rustc`, translates one into the other; you normally call it through Cargo: `cargo build` or `cargo run`.

</details>

2. What is the difference between `git add` and `git commit`?

<details>
<summary>Answer</summary>

`git add` puts the current version of chosen files in the staging area ("what goes into the next snapshot"). `git commit` takes everything in the staging area and records it as a new snapshot (a commit) with a message in the history.

</details>

3. You pushed a commit with a bug. Should you use `git revert` or `git reset`, and why?

<details>
<summary>Answer</summary>

`git revert <commit>`. It creates a **new** commit that undoes the bad one, so shared history is not rewritten and other people's copies stay valid. `git reset` moves the branch backwards and rewrites history; use it only for commits that nobody else has.

</details>

4. What does a merge conflict look like in a file, and how do you finish the merge?

<details>
<summary>Answer</summary>

Git writes both versions between markers: `<<<<<<< HEAD`, `=======`, `>>>>>>> branch`. You edit the file to the version you want and delete the markers, run `git add <file>`, and then `git commit` (or `git merge --continue`).

</details>

5. Why does the root `.gitignore` of your learning repository contain `target/`, and what happens when you run `cargo new` inside an existing repository?

<details>
<summary>Answer</summary>

`target/` holds build output: it is large, generated, and can be rebuilt, so it does not belong in history. When `cargo new` runs inside an existing repository, it does **not** create a `.gitignore` for the new package, so the root `.gitignore` must already ignore `target/`. You can check with `git check-ignore -v code/<project>/target`.

</details>

**Phase 02: basics**

6. What do `checked_add`, `wrapping_add` and `saturating_add` return for `255u8` plus `1`?

<details>
<summary>Answer</summary>

`checked_add` returns `None` (it reports the overflow), `wrapping_add` returns `0` (wraps around), `saturating_add` returns `255` (stays at the maximum). A plain `+` panics on overflow in a debug build.

```text
None
0
255
```

</details>

**Phase 03: ownership and borrowing**

7. Why does this not compile? What is the rule?

```rust
let mut v = vec![1, 2, 3];
let first = &v[0];
v.push(4);
println!("{first}");
```

<details>
<summary>Answer</summary>

```text
error[E0502]: cannot borrow `v` as mutable because it is also borrowed as immutable
```

At any time you may have **either** any number of shared references (`&T`) **or** exactly one mutable reference (`&mut T`), never both. `first` is a shared borrow that is used later, while `push` needs a mutable borrow. It matters: `push` may reallocate the vector, and then `first` would point to freed memory. Fix: use `first` before the `push`, or copy the value (`let first = v[0];`).

</details>

8. What is wrong here, and what are two fixes?

```rust
let a = String::from("hi");
let b = a;
println!("{a} {b}");
```

<details>
<summary>Answer</summary>

```text
error[E0382]: borrow of moved value: `a`
```

`String` owns heap memory and does not implement `Copy`, so `let b = a;` **moves** ownership to `b`; `a` can no longer be used. Fixes: `let b = a.clone();` (an independent copy), or borrow instead of move: `let b = &a;`.

</details>

**Phase 04: structs, enums, errors**

9. What does the `?` operator do in `let n: i32 = text.trim().parse()?;` inside a function returning `Result<i32, ParseIntError>`?

<details>
<summary>Answer</summary>

If the value is `Ok(n)`, `?` unwraps it and continues. If it is `Err(e)`, `?` returns early from the function with that error (converted with `From` if needed). The function's return type must be a compatible `Result` (or `Option`). Real output for `double(" 21 ")` and `double("x")`:

```text
Ok(42)
Err(ParseIntError { kind: InvalidDigit })
```

</details>

**Phase 05: intermediate**

10. Why does this not compile, and how do you fix it?

```rust
fn longest(a: &str, b: &str) -> &str {
    if a.len() > b.len() { a } else { b }
}
```

<details>
<summary>Answer</summary>

```text
error[E0106]: missing lifetime specifier
```

The result is a reference, but the compiler does not know whether it is borrowed from `a` or from `b`. A lifetime parameter says it: `fn longest<'a>(a: &'a str, b: &'a str) -> &'a str`. It means: the result is valid as long as both inputs are.

</details>

11. What does this program print, and why?

```rust
let doubled = (1..4).map(|x| {
    println!("map {x}");
    x * 2
});
println!("before");
let total: i32 = doubled.sum();
println!("total {total}");
```

<details>
<summary>Answer</summary>

```text
before
map 1
map 2
map 3
total 12
```

Iterator adapters such as `map` are **lazy**: they do nothing until something consumes the iterator. `sum()` consumes it, and only then the closure runs.

</details>

12. What does this print, and which closure trait does `inc` implement?

```rust
let mut count = 0;
let mut inc = || {
    count += 1;
    count
};
println!("{} {}", inc(), inc());
println!("{count}");
```

<details>
<summary>Answer</summary>

```text
1 2
2
```

`inc` changes a captured variable, so it implements `FnMut` (it can be called many times and mutates state), and the binding must be `mut`. After the last call, the mutable borrow of `count` ends, so `count` can be printed.

</details>

13. Why do you share data between threads with `Arc<Mutex<T>>` and not with `Rc<RefCell<T>>`?

<details>
<summary>Answer</summary>

`Rc` uses a non-atomic counter and is not `Send`, and `RefCell` is not `Sync`: the compiler refuses to move them between threads. `Arc` has an atomic counter, and `Mutex` gives exclusive access with real locking, so the pair is safe for threads.

</details>

**Phase 06: advanced**

14. When do you choose `fn f<T: Shape>(x: &T)` and when `fn f(x: &dyn Shape)`? What is the size of `&dyn Shape`?

<details>
<summary>Answer</summary>

The generic version uses **static dispatch**: the compiler makes a separate copy for each type (monomorphization), so calls are direct and can be inlined, but the code grows and all values must have one type per call. `&dyn Shape` uses **dynamic dispatch**: one copy of the code, a call through a vtable at run time, and values of different types can live in one collection (`Vec<Box<dyn Shape>>`). A `&dyn Trait` is a **fat pointer** of two words (data pointer and vtable pointer):

```text
&u8: 8
&[u8]: 16
&dyn Debug: 16
Box<dyn Debug>: 16
```

</details>

15. What happens when this code runs, and what does it teach?

```rust
let cell = RefCell::new(1);
let a = cell.borrow_mut();
let b = cell.borrow_mut();
```

<details>
<summary>Answer</summary>

It compiles, but panics at run time:

```text
thread 'main' panicked at src/main.rs:6:18:
RefCell already borrowed
```

`RefCell` moves the borrow rules from compile time to run time. You can change data through a shared reference, but you pay with a possible panic when you break the rules. Use `Cell` for small `Copy` values (no borrow checks needed), `RefCell` when you need references to the inside, and `Mutex` for threads.

</details>

16. Name the five things only `unsafe` code can do. What is a `// SAFETY:` comment, and what does Miri do?

<details>
<summary>Answer</summary>

(1) Dereference a raw pointer, (2) call an `unsafe` function, (3) access or modify a mutable `static`, (4) implement an `unsafe` trait, (5) access a field of a `union`. `unsafe` does not switch off borrow checking. A `// SAFETY:` comment above each `unsafe` block explains why the rules hold there. Miri runs your tests in an interpreter and reports undefined behaviour (dangling pointers, out-of-bounds access, aliasing violations); it cannot call foreign C code.

</details>

17. Name the two families of macros and the three kinds of procedural macros.

<details>
<summary>Answer</summary>

Declarative macros (`macro_rules!`: patterns and repetitions) and procedural macros (Rust functions from `TokenStream` to `TokenStream`, in a crate with `proc-macro = true`). The procedural kinds are function-like (`name!(...)`), derive (`#[derive(Name)]`) and attribute (`#[name]`).

</details>

18. What does an `async fn` return when you call it? Why must you never call `std::thread::sleep` inside async code?

<details>
<summary>Answer</summary>

It returns a future; the body does not run until the future is polled (for example with `.await`). `std::thread::sleep` blocks the whole thread, so the runtime cannot run other tasks on it; use `tokio::time::sleep(...).await`, or `spawn_blocking` for blocking work. In the experiment, two 300 ms tasks on one thread took about 600 ms with the blocking sleep and about 300 ms with the async one.

</details>

19. In FFI, why do you use `CString` to pass text to C, and who must free memory that C allocated with `malloc`?

<details>
<summary>Answer</summary>

C strings end with a NUL byte (the byte with value `0`) and cannot contain NUL inside; `CString::new` adds the NUL and refuses text with an inner NUL. Memory allocated by C must be freed by the matching C function (`free`, or the library's own `free_...` function), never by Rust's allocator. A wrapper type usually does this in `Drop`.

</details>

20. In the web server project, why is the channel receiver wrapped in `Arc<Mutex<...>>`, why is the queue bounded, and how does the pool stop?

<details>
<summary>Answer</summary>

An `mpsc` channel has one receiver, but many workers need to take jobs: `Arc` shares it and `Mutex` lets one worker receive at a time (the lock must be released before running the job). The queue is bounded to give backpressure: when it is full the accept loop answers `503` instead of using unlimited memory. To stop, `Drop` drops the sender; every worker's `recv()` then returns `Err`, the workers leave their loops, and `Drop` joins the threads.

</details>

## Practical tasks

Do these on your computer. Each task has a hint and a verified answer.

### Task 1 — Check the health of your learning repository

In `~/projects/rust`: (a) make sure `git status` is clean and `target/` is ignored; (b) look at your last commits; (c) mark the end of Stage 1 with an annotated tag `stage-1-complete`; (d) if you have a remote (Lesson 01.8), push the branch and the tags.

<details>
<summary>Hint</summary>

`git status`, `git check-ignore -v code/<project>/target`, `git log --oneline --decorate -n 5`, `git tag -a <name> -m "<message>"`, and `git push origin main --tags`. If `git status` shows changes, commit them first with a good message.

</details>

<details>
<summary>Answer</summary>

```sh
cd ~/projects/rust
git status
git check-ignore -v code/p06-webserver/target
git log --oneline --decorate -n 5
git tag -a stage-1-complete -m "Stage 1 complete: phases 00-06"
git tag -n
git push origin main --tags     # only if you have a remote called origin
```

Expected shape of the output (your hashes and names will differ):

```text
On branch main
nothing to commit, working tree clean
.gitignore:1:target/	code/p06-webserver/target
3363654 (HEAD -> main, tag: stage-1-complete, tag: p06-webserver-v1.0) Merge branch 'p06-webserver': multithreaded web server
b675a56 chore: start learning repository
ed5a1e7 (p06-webserver) feat(p06-webserver): M3 thread pool
p06-webserver-v1.0 p06-webserver: all milestones done
stage-1-complete   Stage 1 complete: phases 00-06
```

Why it works: a clean tree means everything is saved in history. `git check-ignore -v` prints which rule ignores the path (if it prints nothing, add `target/` to `.gitignore`). An **annotated** tag (`-a`) stores a message, the author and the date, and it is the right kind for releases and milestones. `git push` does not send tags unless you add `--tags`.

</details>

### Task 2 — One command that checks every project

Write `scripts/check-all.sh` in the root of your learning repository. For each folder `code/*/` that has a `Cargo.toml`, it runs `cargo fmt --check`, `cargo clippy --all-targets -- -D warnings` and `cargo test`. It prints `== <folder>` and then `OK` or `FAILED`, and exits with a non-zero code if any project failed. Run it and fix what it finds.

<details>
<summary>Hint</summary>

Loop with `for manifest in code/*/Cargo.toml`. Use `dirname` to get the folder, and a subshell `( cd "$dir" && ... )` so that the `cd` does not affect the loop. Chain the three commands with `&&` inside an `if`. Keep a variable `status` that you set to `1` on failure and `exit "$status"` at the end.

</details>

<details>
<summary>Answer</summary>

```sh
#!/usr/bin/env bash
# Runs the quality gates in every Cargo project under code/.
# Usage, from the repository root:  bash scripts/check-all.sh
status=0
for manifest in code/*/Cargo.toml; do
    [ -f "$manifest" ] || continue
    dir=$(dirname "$manifest")
    echo "== $dir"
    if (cd "$dir" && cargo fmt --check && cargo clippy --all-targets -q -- -D warnings && cargo test -q >/dev/null); then
        echo "   OK"
    else
        echo "   FAILED"
        status=1
    fi
done
exit $status
```

Run it with `bash scripts/check-all.sh`. With three small projects, one of which has a clippy warning, a real run printed:

```text
== code/p02-hello
   OK
== code/p03-words
   OK
== code/p04-book
error: length comparison to zero
 --> src/main.rs:3:13
  |
3 |     let n = names.len() == 0;
  |             ^^^^^^^^^^^^^^^^ help: using `is_empty` is clearer and more explicit: `names.is_empty()`
  |
  = note: `-D clippy::len-zero` implied by `-D warnings`
error: could not compile `p04-book` (bin "p04-book") due to 1 previous error
   FAILED
```

and the script exited with code `1`. Fix each reported problem (`cargo fmt` fixes formatting itself), run again until every project says `OK`, then commit: `git add scripts/check-all.sh && git commit -m "chore: add check-all quality gate script"`.

Why it works: the three commands are the same gates you used in every project. The script gives you one command for all of them, and the exit code makes it usable later in CI (Phase 09), which runs the same script on every push.

</details>

### Task 3 — A job runner: trait objects, a macro and threads

Write a program with a trait `Job: Send` (methods `name(&self) -> String` and `run(&self) -> u64`), three job types (`Square`, `SumTo`, `Fib`), a macro `jobs![...]` that builds a `Vec<Box<dyn Job>>`, and a function `run_all` that runs each job on its own thread, sends `(name, result)` through a channel, and returns the results sorted. Add tests for the normal case, the empty list and `Fib`.

<details>
<summary>Hint</summary>

The macro: `($($job:expr),* $(,)?) => {{ let list: Vec<Box<dyn Job>> = vec![$(Box::new($job)),*]; list }}`. The type annotation lets the compiler turn each `Box<Square>` into a `Box<dyn Job>`. `Job: Send` makes `Box<dyn Job>` movable into `thread::spawn`. Clone the sender for each thread and `drop` the original so that `rx.iter()` ends.

</details>

<details>
<summary>Answer</summary>

```rust
use std::sync::mpsc;
use std::thread;

trait Job: Send {
    fn name(&self) -> String;
    fn run(&self) -> u64;
}

struct Square(u64);
struct SumTo(u64);
struct Fib(u32);

impl Job for Square {
    fn name(&self) -> String {
        format!("square({})", self.0)
    }
    fn run(&self) -> u64 {
        self.0 * self.0
    }
}

impl Job for SumTo {
    fn name(&self) -> String {
        format!("sum_to({})", self.0)
    }
    fn run(&self) -> u64 {
        (1..=self.0).sum()
    }
}

impl Job for Fib {
    fn name(&self) -> String {
        format!("fib({})", self.0)
    }
    fn run(&self) -> u64 {
        let (mut a, mut b) = (0u64, 1u64);
        for _ in 0..self.0 {
            (a, b) = (b, a + b);
        }
        a
    }
}

macro_rules! jobs {
    ($($job:expr),* $(,)?) => {{
        let list: Vec<Box<dyn Job>> = vec![$(Box::new($job)),*];
        list
    }};
}

fn run_all(jobs: Vec<Box<dyn Job>>) -> Vec<(String, u64)> {
    let (tx, rx) = mpsc::channel();
    let mut handles = Vec::new();
    for job in jobs {
        let tx = tx.clone();
        handles.push(thread::spawn(move || {
            tx.send((job.name(), job.run())).unwrap();
        }));
    }
    drop(tx);
    let mut results: Vec<(String, u64)> = rx.iter().collect();
    for handle in handles {
        handle.join().unwrap();
    }
    results.sort();
    results
}

fn main() {
    let results = run_all(jobs![Square(12), SumTo(100), Fib(10), Fib(50)]);
    for (name, value) in results {
        println!("{name} = {value}");
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn every_job_reports_once() {
        let results = run_all(jobs![Square(3), SumTo(4)]);
        assert_eq!(
            results,
            vec![("square(3)".to_string(), 9), ("sum_to(4)".to_string(), 10)]
        );
    }

    #[test]
    fn empty_list_gives_empty_results() {
        assert!(run_all(jobs![]).is_empty());
    }

    #[test]
    fn fib_values() {
        assert_eq!(Fib(0).run(), 0);
        assert_eq!(Fib(1).run(), 1);
        assert_eq!(Fib(10).run(), 55);
    }
}
```

```text
fib(10) = 55
fib(50) = 12586269025
square(12) = 144
sum_to(100) = 5050
```

`cargo test` shows `test result: ok. 3 passed`.

Why it works: the trait object lets three different types live in one `Vec`. The `Send` supertrait lets the boxes move to other threads. The channel collects results in whatever order the threads finish, so sorting makes the output stable and testable. Dropping the original sender is essential: `rx.iter()` ends only when **all** senders are gone.

</details>

### Task 4 — A safe wrapper for C's `qsort`

Use the C library function `qsort` to sort a slice of `i32`. Declare it in an `unsafe extern "C"` block, write the callback `extern "C" fn compare_i32`, and a safe function `sort_ints(values: &mut [i32])` with `// SAFETY:` comments. Add tests for a normal slice, an empty slice and a one-element slice.

<details>
<summary>Hint</summary>

`void qsort(void *base, size_t count, size_t size, int (*compare)(const void *, const void *))`. In Rust the pointer types are `*mut c_void` and `*const c_void`. Get the base pointer with `values.as_mut_ptr().cast()`. In the callback, cast the two pointers back with `*(a as *const i32)` and return `a.cmp(&b) as c_int`: `Ordering` converts to `-1`, `0`, `1`.

</details>

<details>
<summary>Answer</summary>

```rust
use std::ffi::{c_int, c_void};

unsafe extern "C" {
    fn qsort(
        base: *mut c_void,
        count: usize,
        size: usize,
        compare: extern "C" fn(*const c_void, *const c_void) -> c_int,
    );
}

extern "C" fn compare_i32(a: *const c_void, b: *const c_void) -> c_int {
    // SAFETY: `qsort` calls us with pointers to two elements of the array,
    // and the array holds `i32` values.
    let (a, b) = unsafe { (*(a as *const i32), *(b as *const i32)) };
    a.cmp(&b) as c_int
}

/// Sorts the numbers in place using the C library's `qsort`.
pub fn sort_ints(values: &mut [i32]) {
    // SAFETY: the pointer and the length describe exactly the slice, the
    // element size is the size of `i32`, and `compare_i32` matches the type.
    unsafe {
        qsort(
            values.as_mut_ptr().cast(),
            values.len(),
            std::mem::size_of::<i32>(),
            compare_i32,
        );
    }
}

fn main() {
    let mut numbers = [42, -7, 19, 0, 3, -7];
    sort_ints(&mut numbers);
    println!("{:?}", numbers);
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn sorts_numbers() {
        let mut values = [5, 3, 9, -1, 3];
        sort_ints(&mut values);
        assert_eq!(values, [-1, 3, 3, 5, 9]);
    }

    #[test]
    fn empty_and_single_are_fine() {
        let mut empty: [i32; 0] = [];
        sort_ints(&mut empty);
        let mut one = [7];
        sort_ints(&mut one);
        assert_eq!(one, [7]);
    }
}
```

```text
[-7, -7, 0, 3, 19, 42]
```

Why it works: a slice is "pointer + length", exactly what `qsort` wants. The element size comes from `size_of::<i32>()`, so C moves whole numbers. The callback is `extern "C"` and does not panic. The safe function hides all the `unsafe`: callers cannot pass a wrong length or size.

</details>

### Task 5 — Concurrent fetches with a time limit

Write an async program: `fetch(id)` sleeps `id * 100` ms and returns `page-<id>`. `fetch_with_limit(id, limit)` returns `Ok(page)` or `Err("page-<id> timed out")` when the limit is exceeded. In `main`, start fetches for ids 1 to 5 **concurrently** with a limit of 350 ms and print the results in the order of the ids. Add two `#[tokio::test]` tests.

<details>
<summary>Hint</summary>

Spawn each `fetch_with_limit` with `tokio::spawn` and keep the `JoinHandle`s in a `Vec`; await them in order. `tokio::time::timeout(limit, fetch(id)).await` gives `Result<String, Elapsed>`. Add Tokio with `cargo add tokio --features rt-multi-thread,macros,time`.

</details>

<details>
<summary>Answer</summary>

```rust
use std::time::Duration;
use tokio::time::{sleep, timeout};

async fn fetch(id: u64) -> String {
    sleep(Duration::from_millis(id * 100)).await;
    format!("page-{id}")
}

async fn fetch_with_limit(id: u64, limit: Duration) -> Result<String, String> {
    match timeout(limit, fetch(id)).await {
        Ok(page) => Ok(page),
        Err(_) => Err(format!("page-{id} timed out")),
    }
}

#[tokio::main]
async fn main() {
    let limit = Duration::from_millis(350);
    let mut handles = Vec::new();
    for id in 1..=5 {
        handles.push(tokio::spawn(fetch_with_limit(id, limit)));
    }
    for handle in handles {
        match handle.await.unwrap() {
            Ok(page) => println!("ok:      {page}"),
            Err(message) => println!("failed:  {message}"),
        }
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[tokio::test]
    async fn fast_fetch_succeeds() {
        let result = fetch_with_limit(1, Duration::from_millis(200)).await;
        assert_eq!(result, Ok("page-1".to_string()));
    }

    #[tokio::test]
    async fn slow_fetch_times_out() {
        let result = fetch_with_limit(10, Duration::from_millis(50)).await;
        assert_eq!(result, Err("page-10 timed out".to_string()));
    }
}
```

```text
ok:      page-1
ok:      page-2
ok:      page-3
failed:  page-4 timed out
failed:  page-5 timed out
```

`cargo test` shows `test result: ok. 2 passed`.

Why it works: the five fetches run at the same time, so the program needs about 400 ms (the longest allowed wait is 350 ms), not 1,500 ms. `timeout` drops the slow future, which cancels it. The tests keep generous margins (200 ms against 100 ms; 50 ms against 1,000 ms), because timing tests that are too tight fail randomly.

</details>

## If you got something wrong

| Topic | Review this lesson |
|---|---|
| Source code, compiler, binary | [Lesson 00.3 — How a computer runs a program](../00-getting-started/03-how-a-computer-runs-a-program.md) |
| Reading compiler errors | [Lesson 00.7 — Reading errors and getting help](../00-getting-started/07-reading-errors-and-getting-help.md) |
| `git add`, `git commit`, `.gitignore` | [Lesson 01.3 — Your first repository and commits](../01-git-and-version-control/03-your-first-repository-and-commits.md) |
| `git log`, `git diff` | [Lesson 01.4 — Viewing history and changes](../01-git-and-version-control/04-viewing-history-and-changes.md) |
| `git revert`, `git reset`, `git restore` | [Lesson 01.5 — Undoing changes safely](../01-git-and-version-control/05-undoing-changes-safely.md) |
| Branches and `git merge` | [Lesson 01.6 — Branches and merging](../01-git-and-version-control/06-branches-and-merging.md) |
| Merge conflicts | [Lesson 01.7 — Merge conflicts](../01-git-and-version-control/07-merge-conflicts.md) |
| Remotes, `git push` | [Lesson 01.8 — Remote repositories: GitHub and GitLab](../01-git-and-version-control/08-remotes-github-and-gitlab.md) |
| Commit messages, pull requests, tags | [Lesson 01.9 — Teamwork: pull requests, code review and conventions](../01-git-and-version-control/09-teamwork-pull-requests-and-conventions.md) |
| Integer types and overflow | [Lesson 02.2 — Data types](../02-programming-basics/02-data-types.md) |
| Tests and debugging | [Lesson 02.8 — Your first tests](../02-programming-basics/08-your-first-tests.md), [Lesson 02.9 — Debugging basics](../02-programming-basics/09-debugging-basics.md) |
| Ownership and move | [Lesson 03.2 — The ownership rules](../03-ownership-and-borrowing/02-ownership-rules.md), [Lesson 03.3 — Move, copy and clone](../03-ownership-and-borrowing/03-move-copy-and-clone.md) |
| Borrowing and `&mut` | [Lesson 03.4 — References and borrowing](../03-ownership-and-borrowing/04-references-and-borrowing.md), [Lesson 03.5 — Mutable references and the borrowing rules](../03-ownership-and-borrowing/05-mutable-references-and-the-borrow-rules.md) |
| `Result` and the `?` operator | [Lesson 04.10 — Error handling](../04-structs-enums-collections-errors/10-error-handling.md) |
| Lifetimes | [Lesson 05.3 — Lifetimes](../05-rust-intermediate/03-lifetimes.md), [Lesson 06.5 — Lifetimes in depth](05-lifetimes-in-depth.md) |
| Closures and iterators | [Lesson 05.4 — Closures](../05-rust-intermediate/04-closures.md), [Lesson 05.5 — Iterators](../05-rust-intermediate/05-iterators.md) |
| `Arc`, `Mutex`, threads, channels | [Lesson 05.10 — Threads, channels and shared state](../05-rust-intermediate/10-threads-channels-and-shared-state.md) |
| Generics vs trait objects, vtable | [Lesson 06.1 — Trait objects and dynamic dispatch](01-trait-objects-and-dynamic-dispatch.md) |
| `Cell`, `RefCell`, `Rc<RefCell<T>>` | [Lesson 06.4 — Interior mutability](04-interior-mutability.md) |
| `unsafe`, raw pointers, Miri | [Lesson 06.6 — Unsafe Rust and Miri](06-unsafe-rust-and-miri.md) |
| `macro_rules!` | [Lesson 06.7 — Declarative macros](07-declarative-macros.md) |
| Procedural macros | [Lesson 06.8 — Procedural macros](08-procedural-macros.md) |
| `async`/`await`, Tokio | [Lesson 06.9 — Introduction to async/await](09-async-await-introduction.md) |
| FFI, `CString`, `build.rs` | [Lesson 06.10 — FFI: calling C code](10-ffi-calling-c.md) |
| Thread pool, HTTP (HyperText Transfer Protocol) basics, graceful shutdown | [Lesson 06.11 — Project: multithreaded web server](11-project-multithreaded-web-server.md) |

If you missed more than five questions, do not rush to Stage 2. Review the lessons in the table, redo one exercise from each, and take the quiz again after a day. Re-asking yourself after a day, then after a week, is **spaced repetition**, and it works much better than reading everything twice in one evening.

## What's next

**Stage 2 — Engineering fundamentals** starts with [Phase 07 — Algorithms and data structures](../07-algorithms-and-data-structures/README.md). It uses everything from Stage 1: tests, benchmarks, generics and iterators. Stages 3–5 then go down into how computers and operating systems really work, and up into networks, databases and distributed systems.

Before you start Phase 07:

1. **Write a short retrospective** in your learning journal (10 minutes): which three topics were hardest, which three projects you are proud of, which one thing you will do differently in Stage 2. Add the date.
2. **Schedule your reviews:** put the topics you missed in the quiz on your calendar for tomorrow, in a week, and in a month.
3. **Look at your portfolio.** Your repository now holds projects with tests, READMEs and history. Open it on GitHub or GitLab as a stranger would: is the top-level `README.md` clear? Does each project say how to run it and how to test it? Fix the worst page.
4. **Use the Git habits** from this stage in every project from now on:

| Habit | Command |
|---|---|
| Look before you commit | `git status`, `git diff` |
| Small commits with clear messages | `git add <files>`, `git commit -m "feat(scope): what changed"` |
| Work on a branch, merge with history | `git switch -c name`, `git merge --no-ff name` |
| Mark milestones | `git tag -a name -m "message"`, `git push --tags` |
| Undo safely | `git restore <file>`, `git revert <commit>` |
| See the story | `git log --oneline --graph --decorate` |
| Never commit build output or secrets | `.gitignore` with `target/`, and check `git status` |

5. **Know your toolbox.** These are the tools of the next stages. Do not install them all now; each lesson tells you when to install. Versions were checked on 2026-11-17.

| Tool | Version | What it is for | Used in |
|---|---|---|---|
| `cargo fmt`, `cargo clippy`, `cargo test` | with Rust | formatting, lints, tests: the quality gates | every project |
| `cargo-expand` | 1.0 | shows macro expansion | Lessons 06.7, 06.8 |
| Miri (`cargo +nightly miri`) | nightly component | finds undefined behaviour in Rust code | Lesson 06.6, Phase 20 |
| `cargo-nextest` | 0.9 | a faster test runner | Phase 09 |
| `criterion` | 0.8 | statistical benchmarks in Rust | Phase 05, Phase 09 |
| `hyperfine` | 2.0 | benchmarks whole commands | Phase 09 |
| `cargo-llvm-cov` | 0.9 | test coverage | Phase 09 |
| `cargo-audit`, `cargo-deny` | 0.23, 0.21 | security and license checks of dependencies | Phase 10 |
| `oha` | 1.17 | HTTP load testing | Phase 19 |
| `just` | 1.59 | a small command runner for project tasks | optional |
| `bacon` | 3.26 | re-runs `cargo check` or tests when files change | optional |

Open [START-HERE.md](../START-HERE.md) if you want to re-read how the course is organized, and keep [the glossary](../GLOSSARY.md) open while you read: Stage 2 introduces many new words.

## Sources and further reading

- [The Rust Programming Language](https://doc.rust-lang.org/book/) — the official book; every Stage 1 topic has a chapter you can reread.
- [Rust by Example](https://doc.rust-lang.org/rust-by-example/) — short runnable examples for the same topics.
- [Rustlings](https://github.com/rust-lang/rustlings) — small exercises that you solve in your editor; a good way to practice before Stage 2.
- [Exercism — Rust track](https://exercism.org/tracks/rust) — practice problems with feedback.
- [The Cargo Book](https://doc.rust-lang.org/cargo/) — everything about Cargo, workspaces, build scripts and profiles.
- [Clippy lints](https://rust-lang.github.io/rust-clippy/master/index.html) — the list of lints with explanations, to understand what `clippy` tells you.
- [Pro Git](https://git-scm.com/book/en/v2) — the free book about Git, for the parts of Phase 01 that you want to deepen.

---

[🏠 Start](../START-HERE.md) · [← 06.11 Project: multithreaded web server](11-project-multithreaded-web-server.md) · [📋 Phase 06](README.md) · [Phase 07 — Algorithms and data structures →](../07-algorithms-and-data-structures/README.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
