<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 09.2 End-to-end tests for command-line programs](02-testing-command-line-programs.md) · [📋 Phase 09](README.md) · [09.4 Property-based testing →](04-property-based-testing.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/09-testing-debugging-benchmarking/03-test-doubles.md) <!-- nav -->

# Lesson 09.3 — Test doubles: fakes, stubs and mocks

> **Phase 09 — Testing, debugging and benchmarking** · Lesson 3 of 14 · ⏱ about 135 minutes

## What you will learn

- Why some code is hard to test (it talks to the clock, the network, a database) and how to make it easy: pass the dependency in.
- The five kinds of *test doubles*: dummy, stub, fake, spy and mock, and when to use each.
- How to write doubles by hand, in plain Rust, first.
- How to generate mocks with the `mockall` crate, and what it costs you.
- Why you should prefer fakes and stubs, and use mocks sparingly.

## Before you start

- You finished [Lesson 09.2 — End-to-end tests for command-line programs](02-testing-command-line-programs.md).
- You know traits and generics ([Lesson 05.2](../05-rust-intermediate/02-traits.md), [Lesson 05.1](../05-rust-intermediate/01-generics.md)) and have met dependency injection ([Lesson 08.9](../08-software-design/09-strategy-and-dependency-injection.md)): a function receives what it needs as parameters instead of creating it. This lesson shows why that makes code testable, and builds on it.
- Lesson 09.1 told you that a test must be *deterministic* and that you pass the clock, randomness and the network in as parameters. Now you do it.

## New words

| Word | Meaning |
|---|---|
| **dependency** | Something your code needs from the outside to do its job: the clock, a mail server, a database. |
| **test double** | A stand-in object that replaces a real dependency in a test, like a stunt double in a film. |
| **dummy** | A double that is only passed to fill a parameter. It is never really used. |
| **stub** | A double that returns fixed, prepared answers. |
| **fake** | A double with a real, working, but simplified implementation (for example a "database" that is a `HashMap` in memory). |
| **spy** | A double that works normally and also *records* how it was called, so the test can look at the record afterwards. |
| **mock** | A double that is told in advance which calls to expect (and how often), and makes the test fail if reality differs. |
| **state verification** | Checking the *result* or the *final state* after the code ran (for example, "the user is in the store"). |
| **interaction verification** | Checking *which calls were made* to a dependency (for example, "`send` was called once with this subject"). |
| **`mockall`** | A crate that generates mocks from a trait automatically. |
| **attribute macro** | A line like `#[automock]` placed above an item; it generates extra code for the item at compile time. |
| **`RefCell`** | A Rust type that allows changing a value through a shared reference `&T`; the check is made at run time (Lesson 06.4). |
| **Unix time** | A moment written as the number of seconds since 1970-01-01 00:00 UTC (Coordinated Universal Time, the world's reference time). |

## 1. Why code is hard to test

Here is a function that registers a user and sends a welcome message. The trial period is 14 days from "now":

```rust
// Hard to test. Do not write this.
fn sign_up(email: &str) -> Result<User, SignupError> {
    let now = std::time::SystemTime::now(); // the real clock
    let user = User { email: email.to_string(), trial_ends_at: now_plus_14_days(now) };
    real_mail_server::send(email, "Welcome", "…")?; // a real network call
    real_database::save(&user);                      // a real database
    Ok(user)
}
```

How would you test it? The clock changes every second, so you cannot compare the result with a fixed number. Every
test would send a real e-mail and write to a real database. And how do you test "the mail server is down", when
the mail server is up? The code has three hidden *dependencies*, and the test cannot control any of them.

The cure is the idea of Lesson 08.9: the function does not create its dependencies, it **receives them as
parameters**, described by traits. In production you pass the real ones; in tests you pass **test doubles**.

## 2. The code under test

Create the package. `mockall` is needed only by tests, so it goes into `[dev-dependencies]`:

```bash
cd ~/projects/rust
git switch main
git switch -c lesson/09-03-test-doubles
cargo new --lib code/p09-doubles
cd code/p09-doubles
cargo add --dev mockall@0.15
```

Replace `src/lib.rs` with the production code (the tests come next):

```rust
//! Signing up a new user: check the e-mail, store the user, send a welcome message.

use std::time::{SystemTime, UNIX_EPOCH};

/// Length of the free trial.
pub const TRIAL_DAYS: u64 = 14;
const SECONDS_PER_DAY: u64 = 86_400;

#[derive(Debug, Clone, PartialEq, Eq)]
pub struct User {
    pub email: String,
    /// Moment when the free trial ends, in seconds since 1970-01-01 (Unix time).
    pub trial_ends_at: u64,
}

#[derive(Debug, PartialEq, Eq)]
pub enum SignupError {
    InvalidEmail,
    AlreadyRegistered,
    MailFailed,
}

/// The sending of a message failed.
#[derive(Debug, PartialEq, Eq)]
pub struct MailError;

/// Sends messages to users.
#[cfg_attr(test, mockall::automock)]
pub trait Mailer {
    fn send(&self, to: &str, subject: &str, body: &str) -> Result<(), MailError>;
}

/// Tells the current time, in seconds since 1970-01-01.
#[cfg_attr(test, mockall::automock)]
pub trait Clock {
    fn now(&self) -> u64;
}

/// Remembers the users.
pub trait UserStore {
    fn exists(&self, email: &str) -> bool;
    fn save(&mut self, user: User);
}

/// The real clock.
pub struct SystemClock;

impl Clock for SystemClock {
    fn now(&self) -> u64 {
        SystemTime::now()
            .duration_since(UNIX_EPOCH)
            .expect("the system clock is before 1970")
            .as_secs()
    }
}

/// Registers a new user and sends the welcome message.
/// The user is stored only if the message was sent.
pub fn sign_up(
    email: &str,
    store: &mut impl UserStore,
    mailer: &impl Mailer,
    clock: &impl Clock,
) -> Result<User, SignupError> {
    if !email.contains('@') {
        return Err(SignupError::InvalidEmail);
    }
    if store.exists(email) {
        return Err(SignupError::AlreadyRegistered);
    }
    let user = User {
        email: email.to_string(),
        trial_ends_at: clock.now() + TRIAL_DAYS * SECONDS_PER_DAY,
    };
    let body = format!("Your free trial ends at {}.", user.trial_ends_at);
    mailer
        .send(email, "Welcome", &body)
        .map_err(|_| SignupError::MailFailed)?;
    store.save(user.clone());
    Ok(user)
}
```

**What happens:**

1. The three dependencies are described by three small traits: `Mailer`, `Clock` and `UserStore`. A trait says *what*
   a dependency can do, not *how*.
2. `sign_up` takes them as parameters of type `&impl Trait` ("a reference to any type that implements this trait", Lesson 05.2).
   The store is `&mut` because `save` changes it. The function does not know whether it talks to a real mail
   server or to a double.
3. `SystemClock` is the real clock. In production code you would write an `SmtpMailer` (SMTP, Simple Mail Transfer Protocol, is the protocol that mail servers use to send e-mail) and a `DatabaseStore` in the same way.
   We do not need them to learn about tests.
4. The order inside `sign_up` is a decision we will test: the user is saved **after** the message was sent, so a
   failure of the mail server leaves no half-registered user.
5. `#[cfg_attr(test, mockall::automock)]` means: "only when compiling tests, add `#[mockall::automock]` here." The
   attribute generates the mock types `MockMailer` and `MockClock`; we use them in Section 5. In normal builds the
   attribute disappears and `mockall` is not needed.

## 3. The five kinds of test doubles

People say "mock" for all of them, but the differences are useful. Here they are, from the simplest:

| Kind | What it does | Typical use | In our program |
|---|---|---|---|
| **dummy** | Nothing. It only fills a parameter. | The code never touches it in this test. | A `BrokenMailer` passed in a test that stops before sending. |
| **stub** | Returns prepared answers. | Give the code a fixed input from the outside world. | `FixedClock(1_000)`; `BrokenMailer` (always fails). |
| **fake** | A working shortcut implementation. | Replace something heavy (a database) with something simple (a `HashMap`). | `InMemoryStore`. |
| **spy** | Works, and records the calls. | Check afterwards *what was sent*. | `RecordingMailer`. |
| **mock** | Knows in advance what must happen; fails otherwise. | Check that a call happens exactly as specified. | `MockMailer` from `mockall`. |

Two ways of checking follow from this. With **state verification** you look at the *result* after the code ran
(was the user saved?). With **interaction verification** you look at the *conversation* with the dependency
(was `send` called, how many times, with which arguments?). Fakes and stubs support state verification; spies and mocks
support interaction verification.

## 4. Doubles written by hand

You do not need a library to make a double. A double is only a type that implements the trait. Add the tests to the
end of `src/lib.rs`:

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use std::cell::RefCell;
    use std::collections::HashMap;

    /// A stub: always answers the same.
    struct FixedClock(u64);

    impl Clock for FixedClock {
        fn now(&self) -> u64 {
            self.0
        }
    }

    /// A fake: a real, working, but simple implementation (memory instead of a database).
    #[derive(Default)]
    struct InMemoryStore {
        users: HashMap<String, User>,
    }

    impl UserStore for InMemoryStore {
        fn exists(&self, email: &str) -> bool {
            self.users.contains_key(email)
        }
        fn save(&mut self, user: User) {
            self.users.insert(user.email.clone(), user);
        }
    }

    /// A spy: remembers every message it was asked to send.
    #[derive(Default)]
    struct RecordingMailer {
        sent: RefCell<Vec<(String, String, String)>>,
    }

    impl Mailer for RecordingMailer {
        fn send(&self, to: &str, subject: &str, body: &str) -> Result<(), MailError> {
            self.sent
                .borrow_mut()
                .push((to.to_string(), subject.to_string(), body.to_string()));
            Ok(())
        }
    }

    /// A stub that makes the mail server "break".
    struct BrokenMailer;

    impl Mailer for BrokenMailer {
        fn send(&self, _to: &str, _subject: &str, _body: &str) -> Result<(), MailError> {
            Err(MailError)
        }
    }
```

**What happens:**

1. `FixedClock` is a stub: `now()` always returns the number inside it. Time stops, so a test can compare with an exact number.
2. `InMemoryStore` is a fake. It really stores and finds users, but in a `HashMap`. It is so simple that it needs no
   test of its own, and it behaves like a real store: save, then `exists` is true.
3. `RecordingMailer` is a spy. The trait method takes `&self`, but a spy must change its list. `RefCell` allows
   that: `borrow_mut()` gives temporary write access, checked while the program runs (Lesson 06.4).
4. `BrokenMailer` is a stub with one job: always return an error, so that we can test a situation that is almost
   impossible to create with a real mail server.

Now the tests, still inside `mod tests`:

```rust
    #[test]
    fn a_new_user_is_saved_with_a_trial_of_14_days() {
        let mut store = InMemoryStore::default();
        let mailer = RecordingMailer::default();
        let clock = FixedClock(1_000);

        let user = sign_up("ada@example.com", &mut store, &mailer, &clock).unwrap();

        assert_eq!(user.trial_ends_at, 1_000 + 14 * 86_400);
        assert!(store.exists("ada@example.com"));
    }

    #[test]
    fn a_welcome_message_is_sent_once() {
        let mut store = InMemoryStore::default();
        let mailer = RecordingMailer::default();

        sign_up("ada@example.com", &mut store, &mailer, &FixedClock(0)).unwrap();

        let sent = mailer.sent.borrow();
        assert_eq!(sent.len(), 1);
        assert_eq!(sent[0].0, "ada@example.com");
        assert_eq!(sent[0].1, "Welcome");
        assert_eq!(sent[0].2, "Your free trial ends at 1209600.");
    }

    #[test]
    fn an_invalid_email_sends_nothing_and_saves_nothing() {
        let mut store = InMemoryStore::default();
        let mailer = RecordingMailer::default();

        let result = sign_up("not-an-email", &mut store, &mailer, &FixedClock(0));

        assert_eq!(result, Err(SignupError::InvalidEmail));
        assert!(mailer.sent.borrow().is_empty());
        assert!(store.users.is_empty());
    }

    #[test]
    fn signing_up_twice_is_refused_and_sends_no_second_message() {
        let mut store = InMemoryStore::default();
        let mailer = RecordingMailer::default();
        sign_up("ada@example.com", &mut store, &mailer, &FixedClock(0)).unwrap();

        let second = sign_up("ada@example.com", &mut store, &mailer, &FixedClock(0));

        assert_eq!(second, Err(SignupError::AlreadyRegistered));
        assert_eq!(mailer.sent.borrow().len(), 1);
    }

    #[test]
    fn when_the_mail_fails_the_user_is_not_saved() {
        let mut store = InMemoryStore::default();

        let result = sign_up("ada@example.com", &mut store, &BrokenMailer, &FixedClock(0));

        assert_eq!(result, Err(SignupError::MailFailed));
        assert!(!store.exists("ada@example.com"));
    }
}
```

Close the module with the final `}` and run `cargo test`. Five tests pass. (The mock tests of Section 5 are not written yet.)

**What happens:** each test follows Arrange, Act, Assert (Lesson 09.1). The first checks the *state*: the trial end
is the fixed time plus 14 days, and the fake store now knows the user. The second asks the spy what was sent. The
third and fourth check that *nothing* happened where nothing should (no message, no stored user, no second message).
The fifth proves the rule from Section 2: a mail failure leaves no user behind. Without `BrokenMailer` you
could not test this at all.

> 🧪 **Testing:** The tests do not know *how* `sign_up` works inside. If you swap the order of the `exists` check and the
> e-mail check, all tests still pass. Doubles let you test **behaviour**, not implementation.

## 5. Mocks with `mockall`

Writing `RecordingMailer` by hand took ten lines. For traits with many methods, the hand-written work grows, so
the `mockall` crate generates doubles for you. Because of `#[cfg_attr(test, mockall::automock)]`, the types `MockMailer` and
`MockClock` already exist in test builds. Add two tests (inside `mod tests`, before the closing brace):

```rust
    #[test]
    fn mock_the_welcome_message_has_the_right_recipient_and_subject() {
        let mut mailer = MockMailer::new();
        mailer
            .expect_send()
            .times(1)
            .withf(|to, subject, _body| to == "ada@example.com" && subject == "Welcome")
            .returning(|_, _, _| Ok(()));
        let mut clock = MockClock::new();
        clock.expect_now().return_const(1_000u64);
        let mut store = InMemoryStore::default();

        let user = sign_up("ada@example.com", &mut store, &mailer, &clock).unwrap();

        assert_eq!(user.trial_ends_at, 1_000 + 14 * 86_400);
    }

    #[test]
    fn mock_an_invalid_email_never_calls_the_mailer() {
        let mut mailer = MockMailer::new();
        mailer.expect_send().never();
        let mut store = InMemoryStore::default();

        let result = sign_up("not-an-email", &mut store, &mailer, &FixedClock(0));

        assert_eq!(result, Err(SignupError::InvalidEmail));
    }
```

```bash
cargo test
```

```text
running 7 tests
test tests::an_invalid_email_sends_nothing_and_saves_nothing ... ok
test tests::mock_an_invalid_email_never_calls_the_mailer ... ok
test tests::when_the_mail_fails_the_user_is_not_saved ... ok
test tests::a_new_user_is_saved_with_a_trial_of_14_days ... ok
test tests::signing_up_twice_is_refused_and_sends_no_second_message ... ok
test tests::mock_the_welcome_message_has_the_right_recipient_and_subject ... ok
test tests::a_welcome_message_is_sent_once ... ok

test result: ok. 7 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

**What happens:**

1. `MockMailer::new()` creates a mock that does **nothing allowed**: any call without an expectation panics.
2. `mailer.expect_send()` starts an *expectation* for the method `send`. Every method `foo` of the trait gets an
   `expect_foo`. The following calls refine it:
   - `.times(1)`: it must be called exactly once (`.never()` means zero times; `.times(1..)` accepts a range);
   - `.withf(|to, subject, _body| …)`: a check on the arguments; the expectation matches only if the closure returns `true`;
   - `.returning(|_, _, _| Ok(()))`: what the mock answers when called; a closure receives the arguments.
3. `clock.expect_now().return_const(1_000u64)` is the shortcut for "always answer this value". It makes `MockClock` act
   as a stub. A mock can play any role.
4. The mocks are checked **automatically when they are dropped** at the end of the test: if `send` was expected once
   and never called, the test fails there, without any `assert`.
5. In the second test the expectation is `never()`: the test proves that for a bad e-mail the mailer is never called.
   `RecordingMailer` proved the same thing in `an_invalid_email_sends_nothing…`, with a visible `assert!(…is_empty())`.
   Both ways are correct; the mock puts the assertion *before* the code runs.

### When a mock fails

If the expectation is not satisfied, the message says what was expected and what happened. Change `"Welcome"` in
`withf` to `"Hello"` and run the test:

```text
---- tests::mock_the_welcome_message_has_the_right_recipient_and_subject stdout ----
thread 'tests::mock_the_welcome_message_has_the_right_recipient_and_subject' (4708734) panicked at src/lib.rs:28:18:
MockMailer::send("ada@example.com", "Welcome", "Your free trial ends at 1210600."): No matching expectation found
```

**What it means:** the code called `send` with these three arguments, but no expectation accepted them. Change `times(1)` to
`times(2)` and the message becomes:

```text
MockMailer::send: Expectation(<function>) called 1 time(s) which is fewer than expected 2
```

Both mistakes show up as a panic with the name of the method. (The line number points into the code generated for the
trait, and may differ on your computer.)

## 6. Which double should you choose?

A short guide, from best to use to use only when needed:

1. **Use the real thing** when it is fast, safe and deterministic: a pure function, a `Vec`, the string library.
2. **Use a stub** for inputs you must control: the time, random numbers, a failing server.
3. **Use a fake** for a heavy dependency with real state: a database, a file system, a message queue. The test then
   checks the *state*, which survives refactoring.
4. **Use a spy or a mock** when the *call itself* is the result: sending an e-mail, charging a card, writing a log line.
   No state remains to inspect, so checking the call is the only way.

Rules that save you pain:

- **Mock what you own.** Double your *own* trait (`Mailer`), not a type from someone else's library. A trait you own
  stays stable; the library's methods can change.
- **Do not over-specify.** `times(1)` and `withf` on every argument make the test a copy of the code. When the code
  is refactored, the tests fail although the behaviour is the same. Specify only what matters to the user.
- **Doubles must behave like the real thing.** A fake store that allowed two users with one e-mail, when the real
  database refuses it, makes tests pass for code that is wrong. For important dependencies, run the same checks
  against both the fake and the real implementation.
- **Test the real thing, too.** Doubles replace the dependency in unit tests, but one test at a higher level
  (Lesson 09.1) must use the real mail server or database in a safe test environment, because only that proves the
  real wiring works.

> 📏 **Convention:** Keep doubles small and next to the tests. When several test files need the same fake, move it to
> `tests/common/mod.rs` (Lesson 05.6).

> 🔒 **Security:** Doubles also make security tests possible. A stub that returns "invalid password" or "expired token"
> lets you test your handling of attacks without any real accounts.

## Common mistakes

**1. Forgetting an expectation on a mock.** Calling a method that has no `expect_…` panics with
`No matching expectation found`, as in Section 5. Every method the code will call needs an expectation. If the
expectation is allowed to be satisfied in any way, write `.returning(…)` and no `.times(…)`.

**2. A double that is too clever.** A fake with a hundred lines of its own logic has its own bugs. If a double needs
a test, simplify the design.

**3. Testing the double.** An assertion like `assert_eq!(fake_store.exists("x"), true)` right after `fake_store.save(…)`
in the test itself proves only that the fake works. Put the assertion after calling the code under test.

**4. Mocking everything.** If a function takes a `Vec` and returns a `Vec`, no double is needed. Doubles are for
*boundaries*: places where your code meets the clock, the network, the disk and the database.

**5. Using the `mockall` mock in `tests/`.** `#[cfg_attr(test, …)]` is active only when the library itself is compiled for
its own tests, so `MockMailer` does not exist in an integration test file. For integration tests, write the double
by hand in the test file.

## Try it yourself

1. Create the package and type the production code. Run `cargo build` to see that it compiles.
2. Add the doubles and the five hand-written tests. Run `cargo test` and check that five tests pass.
3. Add the two `mockall` tests and check that seven tests pass.
4. Break the code on purpose: in `sign_up`, call `store.save(user.clone())` *before* the `send`. Which test fails? It is
   `when_the_mail_fails_the_user_is_not_saved`, with the message `assertion failed: !store.exists("ada@example.com")`.
   Restore the code.
5. Run `cargo fmt --check && cargo clippy --all-targets -- -D warnings && cargo test`. Commit:
   `git add code/p09-doubles && git commit -m "Add p09-doubles: stubs, fakes, spies and mocks"`.

## Exercises

### Exercise 1 — Check the message body (easy)

Write a test with a clock fixed at 1,000 that checks the body of the welcome message exactly:
`Your free trial ends at 1210600.`

<details>
<summary>Hint</summary>

Use `FixedClock(1_000)` and `RecordingMailer`. 1,000 + 14 × 86,400 = 1,210,600. Read the body from `sent[0].2`.

</details>

<details>
<summary>Answer</summary>

```rust
    #[test]
    fn the_message_body_contains_the_trial_end() {
        let mut store = InMemoryStore::default();
        let mailer = RecordingMailer::default();

        sign_up("ada@example.com", &mut store, &mailer, &FixedClock(1_000)).unwrap();

        assert_eq!(
            mailer.sent.borrow()[0].2,
            "Your free trial ends at 1210600."
        );
    }
```

Why it works: the stub clock makes the time exact, so the whole body is predictable. With the real clock you could not
write this test.

</details>

### Exercise 2 — Name the double (medium)

What kind of double is each of these (dummy, stub, fake, spy, mock)?

1. A `Clock` that always returns `0`.
2. A `UserStore` built on a `Vec<User>` that really stores and finds users.
3. A `Mailer` that does nothing and returns `Ok(())`, passed into a test about e-mail validation, where no message is ever sent.
4. A `Mailer` that counts calls in a `Cell<u32>`, which the test reads at the end.
5. `MockMailer` with `expect_send().times(1)`.

<details>
<summary>Hint</summary>

Ask: does it return prepared answers (stub)? Does it really work in a simpler way (fake)? Does it only fill a hole
(dummy)? Does it record (spy)? Does it check itself (mock)?

</details>

<details>
<summary>Answer</summary>

1. Stub (a fixed answer).
2. Fake (a real, working, simple implementation).
3. Dummy (it fills the parameter and is never used). If the test did call it and relied on `Ok(())`, it would be a stub.
4. Spy (it works and records; the test inspects the record).
5. Mock (the expectation is declared in advance and checked when the mock is dropped).

The boundaries are not strict: the same type can play two roles. The names help you to think about what the double is
*for*.

</details>

### Exercise 3 — A mail server that recovers (hard)

Write a double `FlakyMailer` that fails on the first call and succeeds from the second call on. Then write a test: the
first `sign_up` for `ada@example.com` returns `MailFailed` and saves nothing, the second `sign_up` for the same address
succeeds. Why is this test valuable?

<details>
<summary>Hint</summary>

The trait method takes `&self`, so keep the call count in a `Cell<u32>` (like `RefCell`, but for simple copyable
values: `get()` and `set()`). Use the same store and the same clock for both calls.

</details>

<details>
<summary>Answer</summary>

```rust
    struct FlakyMailer {
        calls: std::cell::Cell<u32>,
    }

    impl Mailer for FlakyMailer {
        fn send(&self, _to: &str, _subject: &str, _body: &str) -> Result<(), MailError> {
            self.calls.set(self.calls.get() + 1);
            if self.calls.get() == 1 { Err(MailError) } else { Ok(()) }
        }
    }

    #[test]
    fn a_failed_signup_can_be_retried() {
        let mut store = InMemoryStore::default();
        let mailer = FlakyMailer { calls: std::cell::Cell::new(0) };
        let clock = FixedClock(0);

        let first = sign_up("ada@example.com", &mut store, &mailer, &clock);
        assert_eq!(first, Err(SignupError::MailFailed));
        assert!(!store.exists("ada@example.com"));

        let second = sign_up("ada@example.com", &mut store, &mailer, &clock);
        assert!(second.is_ok());
        assert!(store.exists("ada@example.com"));
    }
```

Why it works: because the user is saved only after the message was sent, the failed attempt left nothing behind and
the retry is not rejected as `AlreadyRegistered`. The test protects exactly that design decision. This scenario
(a server that fails once, then recovers) is very hard to produce with a real server, and trivial with a double.

</details>

## Quiz

1. Why does `sign_up` take a `Clock` as a parameter instead of calling `SystemTime::now()`?

<details>
<summary>Answer</summary>

So that a test can pass a clock that returns a fixed time. With the real clock the result changes every second, and the
test could not compare it with an exact value.

</details>

2. What is the difference between a stub and a mock?

<details>
<summary>Answer</summary>

A stub only returns prepared answers; the test does not care how often it is called. A mock is told in advance which calls to
expect, and the test fails if reality is different.

</details>

3. What is a fake? Give an example.

<details>
<summary>Answer</summary>

A working but simplified implementation, for example a user store that keeps users in a `HashMap` instead of a database.

</details>

4. When is a spy or a mock the right choice, and not a fake?

<details>
<summary>Answer</summary>

When the call itself is the result and no state remains to inspect: sending an e-mail, charging a card, writing a log
line.

</details>

5. Why is "mock what you own" good advice?

<details>
<summary>Answer</summary>

A trait you define stays stable and describes what *your* code needs. Doubling a type from someone else's library ties your tests to
that library's details, and they break when it changes.

</details>

6. What happens if the code calls a method of a `mockall` mock that has no expectation?

<details>
<summary>Answer</summary>

The mock panics with `No matching expectation found`, and the test fails.

</details>

## Summary

- Code that creates its own clock, network or database is hard to test. Pass dependencies in as parameters, described by traits.
- A test double replaces a dependency in a test. The kinds: dummy (fills a parameter), stub (fixed answers), fake (simple but
  working), spy (records calls), mock (expects calls and checks itself).
- State verification checks results and is stable under refactoring; interaction verification checks calls and is
  needed when the call is the result.
- Doubles are plain Rust types that implement the trait; write them by hand first.
- `mockall` generates mocks: `#[cfg_attr(test, mockall::automock)]`, then `expect_<method>().times(n).withf(…).returning(…)`.
- Prefer real objects, then stubs and fakes; use mocks sparingly; mock what you own; do not over-specify.

## Sources and further reading

- [mockall on docs.rs](https://docs.rs/mockall/0.15.0/mockall/) — the full guide to expectations, matchers, sequences and generic traits.
- [The Rust Programming Language — 10.2 Traits](https://doc.rust-lang.org/book/ch10-02-traits.html) — traits and `impl Trait` parameters, the tools behind every double.
- [std::cell::RefCell](https://doc.rust-lang.org/std/cell/struct.RefCell.html) and [std::cell::Cell](https://doc.rust-lang.org/std/cell/struct.Cell.html) — interior mutability used by the spy.
- [Martin Fowler — Mocks Aren't Stubs](https://martinfowler.com/articles/mocksArentStubs.html) — the classic article on state versus interaction verification.
- [Martin Fowler — TestDouble](https://martinfowler.com/bliki/TestDouble.html) — a short definition of the vocabulary used in this lesson.

---

[🏠 Start](../START-HERE.md) · [← 09.2 End-to-end tests for command-line programs](02-testing-command-line-programs.md) · [📋 Phase 09](README.md) · [09.4 Property-based testing →](04-property-based-testing.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
