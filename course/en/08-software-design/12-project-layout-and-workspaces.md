<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 08.11 Conventions: naming, Clippy and the Rust API Guidelines](11-conventions-naming-clippy-and-api-guidelines.md) · [📋 Phase 08](README.md) · [08.13 Refactoring and code review →](13-refactoring-and-code-review.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/08-software-design/12-project-layout-and-workspaces.md) <!-- nav -->

# Lesson 08.12 — Project layout and workspaces

> **Phase 08 — Software design: principles, patterns and conventions** · Lesson 12 of 17 · ⏱ about 3.5 hours (2 sittings)

## What you will learn

- The standard layout of a Cargo package: where the program, the library, extra programs, tests and examples go.
- How to organise modules inside `src`, and how `pub use` and `pub(crate)` shape the public side of a crate.
- What a Cargo workspace is, how to build one with three crates, and how members share dependencies, versions and lint settings.
- How the compiler protects your design (no dependency cycles), and when *not* to split a project.

## Before you start

- You finished [Lesson 08.11 — Conventions: naming, Clippy and the Rust API Guidelines](11-conventions-naming-clippy-and-api-guidelines.md).
- You know modules and packages ([Lesson 04.6](../04-structs-enums-collections-errors/06-modules-crates-and-packages.md)), Cargo dependencies ([Lesson 05.8](../05-rust-intermediate/08-cargo-dependencies-and-versions.md)) and the "thin `main`, library does the work" idea ([Lesson 05.13](../05-rust-intermediate/13-project-minigrep.md)).
- Your course folder is already a Git repository, so `cargo new` does not run `git init` again. (If you work outside a repository, add `--vcs none` to `cargo new` in this lesson.)

## New words

| Word | Meaning |
|---|---|
| **target** (Cargo) | One thing Cargo can build from a package: a library, a binary, an example, a test or a benchmark. |
| **workspace** | A group of packages that live in one folder tree and share one `Cargo.lock` and one `target` folder. |
| **member** | A package that belongs to a workspace. |
| **virtual manifest** | A root `Cargo.toml` that has a `[workspace]` table but no `[package]`: the root itself builds nothing. |
| **re-export** (`pub use`) | Making an item that lives in one module available under another path, so that users do not need to know your internal layout. |
| **resolver** | The part of Cargo that picks versions and features of dependencies. `resolver = "3"` is the one for edition 2024. |
| **path dependency** | A dependency on a package in a folder on your disk: `shop-core = { path = "crates/shop-core" }`. |
| **dependency cycle** | Package A needs B, and B (directly or through others) needs A. Cargo does not allow it. |

## 1. One package, several targets

A **package** is described by one `Cargo.toml`. It can produce many **targets**. Cargo finds them by folder
and file names, so you configure nothing:

```text
p08-layout/
├── Cargo.toml
├── src/
│   ├── lib.rs          ← the library (the real work)
│   ├── main.rs         ← the main program, a thin wrapper
│   └── bin/
│       └── shout.rs    ← one more program
├── tests/
│   └── greeting.rs     ← integration test (uses only the public API)
├── examples/
│   └── basic.rs        ← a runnable example for users of the library
└── benches/            ← benchmarks (Phase 09)
```

| Path | Target | How you run it |
|---|---|---|
| `src/lib.rs` | the library; other packages and your own programs `use` it | (not run) |
| `src/main.rs` | the main program, named like the package | `cargo run` |
| `src/bin/NAME.rs` | an extra program | `cargo run --bin NAME` |
| `tests/*.rs` | integration tests; each file is its own crate | `cargo test` |
| `examples/*.rs` | examples; compiled by `cargo test` too, so they cannot go stale | `cargo run --example NAME` |
| `benches/*.rs` | benchmarks | `cargo bench` |
| `build.rs` | a build script that runs before the crate is compiled (rarely needed) | automatic |

Create the package and the files (the crate name in code is `p08_layout`, with an underscore):

```bash
cargo new code/p08-layout
mkdir code/p08-layout/{src/bin,tests,examples}
```

`src/lib.rs`:

```rust
//! Greeting helpers.

/// Builds a greeting.
#[must_use]
pub fn greeting(name: &str) -> String {
    format!("Hello, {name}!")
}
```

`src/main.rs`:

```rust
fn main() {
    println!("{}", p08_layout::greeting("world"));
}
```

`src/bin/shout.rs`:

```rust
fn main() {
    println!("{}", p08_layout::greeting("world").to_uppercase());
}
```

`examples/basic.rs`:

```rust
fn main() {
    println!("{}", p08_layout::greeting("example"));
}
```

`tests/greeting.rs`:

```rust
#[test]
fn greets_by_name() {
    assert_eq!(p08_layout::greeting("Ana"), "Hello, Ana!");
}
```

Now `cargo run` complains, because the package has two programs:

```text
error: `cargo run` could not determine which binary to run. Use the `--bin` option to specify a binary, or the `default-run` manifest key.
available binaries: p08-layout, shout
```

Add `default-run = "p08-layout"` to the `[package]` table of `Cargo.toml`. Now all three work:

```bash
cargo run
cargo run --bin shout
cargo run --example basic
```

```text
Hello, world!
HELLO, WORLD!
Hello, example!
```

**What happens:**

1. `main.rs` and `shout.rs` are separate programs. Both call the same library, so there is one copy of the logic.
2. `cargo test` runs the unit tests of each target, the files in `tests/`, and the doc tests. The integration test sees only what is `pub`, just like a user of your crate.
3. Keeping the logic in `lib.rs` and the programs thin (arguments, printing, exit status) is the layout of Lesson 05.13. It makes the logic testable.

## 2. Organising `src`

Lesson 04.6 showed how a module becomes a file (`mod parser;` finds `parser.rs`, or `parser/mod.rs`). Here are the design
decisions that sit on top of that:

- **`lib.rs` is the table of contents.** Keep it short: `mod` lines, a few `pub use` lines and the crate documentation. A reader should understand
  what the crate offers in one screen.
- **Prefer `name.rs` plus a `name/` folder** for children over `name/mod.rs`. With `mod.rs`, every folder has a file with the same name and
  the tabs in your editor become useless.
- **Keep files small and focused.** A file of 200 to 400 lines about one thing is easy to hold in your head. A file called `utils.rs` or `helpers.rs`
  that collects unrelated functions is the "god module" smell of Lesson 08.10: name modules after what they are *about*.
- **Make things private first.** Everything is private by default. Use `pub(crate)` for helpers that several of your own modules share, and `pub` only for what
  users should rely on. A `pub` item is a promise.
- **Re-export to hide your layout.** Your internal folders will change; your users' code should not break. Inside `lib.rs`:

```rust
mod model;
mod parser;

pub use model::Item;
pub use parser::parse_line;
```

  Users write `my_crate::Item`, not `my_crate::model::Item`. You can move `Item` to another file without a breaking change.
- **Group by feature, not by kind of thing.** A layout `src/orders/`, `src/customers/`, `src/payments/` keeps the code that changes together in one
  place. A layout `src/models/`, `src/handlers/`, `src/services/` spreads one feature over three folders (the "shotgun surgery" smell). Layers are the topic of
  [Lesson 08.14](14-layers-and-hexagonal-architecture.md); even there, the usual advice is feature folders first, layers inside.

> 📏 **Convention:** In `lib.rs` use `//!` for the crate documentation, then `mod` declarations, then `pub use`. In any other file, put `use` lines first,
> then the types, then the `impl` blocks, then `#[cfg(test)] mod tests` at the end.

## 3. Cargo workspaces

When a project grows, you may want several packages that are built, tested and versioned together. A **workspace** does that. You will build
`p08-shop`, a tiny shop with three crates:

```text
shop-cli ──► shop-report ──► shop-core
   └──────────────────────────►
```

- `shop-core` holds the domain: items and prices. It depends on nothing of ours.
- `shop-report` turns items into text. It depends on `shop-core`.
- `shop-cli` is the program. It depends on both.

The arrows show the **direction of dependencies**: from the outside (program) towards the inside (domain). The inside knows nothing about the outside. This is the
rule you will meet again in Lesson 08.14.

### 3.1 Create the workspace

```bash
mkdir code/p08-shop
cd code/p08-shop
mkdir crates
cargo new --lib crates/shop-core
cargo new --lib crates/shop-report
cargo new crates/shop-cli
```

Put this in the **root** `Cargo.toml` (create it; there is no `[package]` in it, so this is a *virtual manifest*):

```toml
[workspace]
resolver = "3"
members = ["crates/*"]

[workspace.package]
version = "0.1.0"
edition = "2024"
license = "MIT"

[workspace.dependencies]
shop-core = { path = "crates/shop-core" }
shop-report = { path = "crates/shop-report" }
thiserror = "2"

[workspace.lints.clippy]
pedantic = { level = "warn", priority = -1 }
```

**What happens:**

1. `members = ["crates/*"]` makes every folder under `crates/` a member. A new crate joins the workspace when you create its folder.
2. `[workspace.package]` holds values that all members share (version, edition, licence), so you change them in one place.
3. `[workspace.dependencies]` lists dependencies **once**: external crates with a version (`thiserror`, from Lesson 05.11) and our own crates by path. Members then say
   `thiserror.workspace = true` and inherit the line. All members are guaranteed to use the same version.
4. `[workspace.lints]` is the lint policy of Lesson 08.11 for the whole workspace. Each member opts in with `[lints] workspace = true`.
5. `resolver = "3"` is the dependency resolver for edition 2024. (A workspace needs it spelled out, because the root has no `edition` to take it from.)

Now the three member manifests. `crates/shop-core/Cargo.toml`:

```toml
[package]
name = "shop-core"
version.workspace = true
edition.workspace = true
license.workspace = true

[dependencies]
thiserror.workspace = true

[lints]
workspace = true
```

`crates/shop-report/Cargo.toml`:

```toml
[package]
name = "shop-report"
version.workspace = true
edition.workspace = true
license.workspace = true

[dependencies]
shop-core.workspace = true

[lints]
workspace = true
```

`crates/shop-cli/Cargo.toml` is the same with `name = "shop-cli"` and two dependencies:

```toml
[package]
name = "shop-cli"
version.workspace = true
edition.workspace = true
license.workspace = true

[dependencies]
shop-core.workspace = true
shop-report.workspace = true

[lints]
workspace = true
```

### 3.2 The code

`crates/shop-core/src/lib.rs`:

```rust
//! The domain: items and prices.

use thiserror::Error;

/// Something that can be bought.
#[derive(Debug, Clone, PartialEq)]
pub struct Item {
    name: String,
    price_cents: u32,
}

/// Why an item could not be created.
#[derive(Debug, Error, PartialEq)]
pub enum ItemError {
    /// The name is empty.
    #[error("the name must not be empty")]
    EmptyName,
}

impl Item {
    /// Creates an item.
    ///
    /// # Errors
    ///
    /// Returns [`ItemError::EmptyName`] when `name` is empty.
    pub fn new(name: &str, price_cents: u32) -> Result<Item, ItemError> {
        if name.is_empty() {
            return Err(ItemError::EmptyName);
        }
        Ok(Item {
            name: name.to_string(),
            price_cents,
        })
    }

    /// The item name.
    #[must_use]
    pub fn name(&self) -> &str {
        &self.name
    }

    /// The price in cents.
    #[must_use]
    pub fn price_cents(&self) -> u32 {
        self.price_cents
    }
}

/// The sum of all prices, in cents.
#[must_use]
pub fn total_cents(items: &[Item]) -> u32 {
    items.iter().map(Item::price_cents).sum()
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn total_adds_prices() {
        let items = [
            Item::new("tea", 250).unwrap(),
            Item::new("cake", 400).unwrap(),
        ];
        assert_eq!(total_cents(&items), 650);
    }

    #[test]
    fn empty_name_is_rejected() {
        assert_eq!(Item::new("", 100), Err(ItemError::EmptyName));
    }
}
```

`crates/shop-report/src/lib.rs`:

```rust
//! Text reports about items.

use shop_core::{Item, total_cents};

fn line(label: &str, cents: u32) -> String {
    format!("{label:<8}{cents:>6}\n")
}

/// Renders one line per item and a total line.
#[must_use]
pub fn render(items: &[Item]) -> String {
    let mut lines: Vec<String> = items
        .iter()
        .map(|item| line(item.name(), item.price_cents()))
        .collect();
    lines.push(line("total", total_cents(items)));
    lines.concat()
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn report_has_a_total_line() {
        let items = [Item::new("tea", 250).unwrap()];
        assert_eq!(render(&items), "tea        250\ntotal      250\n");
    }
}
```

`crates/shop-cli/src/main.rs`:

```rust
use shop_core::Item;
use shop_report::render;

fn main() {
    let items = [
        Item::new("tea", 250).expect("valid item"),
        Item::new("cake", 400).expect("valid item"),
    ];
    print!("{}", render(&items));
}
```

**What happens:**

1. `Item` has private fields and a validating constructor (Lessons 08.5 and 08.11). Another crate can read an `Item` only through `name()` and `price_cents()`.
   Across crates, `pub` is the only way to share: `pub(crate)` ends at the crate border. A workspace therefore forces you to decide what the public side is.
2. `shop_report` uses `shop_core::{Item, total_cents}`. The crate name in code is the package name with underscores.
3. `render` builds the lines in a vector and joins them with `concat`. (A first version that used `text.push_str(&format!(...))` was rejected by the workspace's
   `pedantic` setting with `` `format!(..)` appended to existing `String` `` — the lint policy works across all members.)
4. The format `{label:<8}{cents:>6}` pads the label to 8 characters, left-aligned, and the number to 6, right-aligned.

### 3.3 Build, run, test

From the workspace root (`code/p08-shop`):

```bash
cargo run
```

```text
tea        250
cake       400
total      650
```

`cargo run` at the root works because there is only one program in the whole workspace. Then:

```bash
cargo test --workspace
cargo clippy --workspace --all-targets -- -D warnings
cargo test -p shop-core
```

`cargo test --workspace` runs the tests of every member (shortened):

```text
test tests::empty_name_is_rejected ... ok
test tests::total_adds_prices ... ok
test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
test tests::report_has_a_total_line ... ok
test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

(Cargo also prints `0 passed` lines for the targets that have no tests.) The `clippy` command prints only `Checking` and `Finished`. `-p NAME` (`--package`) limits a command to one member.

Look at the folder. All members share **one** `Cargo.lock` and **one** `target` folder, both in the root, so a dependency is compiled once for the whole workspace:

```text
p08-shop/
├── Cargo.toml          ← virtual manifest
├── Cargo.lock          ← one for all members
├── target/             ← one build folder
└── crates/
    ├── shop-core/      (Cargo.toml, src/lib.rs)
    ├── shop-report/    (Cargo.toml, src/lib.rs)
    └── shop-cli/       (Cargo.toml, src/main.rs)
```

> 📏 **Convention:** Commit `Cargo.lock` for workspaces that contain a program (Lesson 05.8), and make `target/` ignored in `.gitignore`.

### 3.4 The compiler guards your design

Suppose a developer in a hurry makes `shop-core` use the report code: they add `shop-report.workspace = true` to the `[dependencies]` of `shop-core`. Cargo refuses to build:

```text
error: cyclic package dependency: package `shop-core v0.1.0 (...)` depends on itself. Cycle:
package `shop-core v0.1.0 (...)`
    ... which satisfies path dependency `shop-core` (locked to 0.1.0) of package `shop-report v0.1.0 (...)`
    ... which satisfies path dependency `shop-report` (locked to 0.1.0) of package `shop-core v0.1.0 (...)`
```

The domain cannot depend on the code that depends on the domain. This is a feature: the arrows in the picture of Section 3 are *enforced* by the compiler. Inside a single crate nothing stops
`domain.rs` from calling `cli.rs`; between crates, the build fails. If you ever see this error, the fix is almost never to merge the crates. Ask which of
the two should not know about the other; usually the answer is to move the shared piece "downwards", or to let the lower crate define a trait that the upper crate implements (Lessons 08.3 and 08.9).

## 4. When to split, and when not to

Start with **one package** (library + thin program). Split into a workspace when you have a reason you can name:

- **A boundary you want enforced.** The domain must not know about the command line or the database (Section 3.4).
- **Different users.** One part is used by a program, a server and a test tool; another part only by the server.
- **Different dependencies.** The domain should not pull in a web framework. A separate crate keeps heavy dependencies away from the crates that do not need them.
- **Build time.** Cargo compiles crates in parallel and rebuilds only changed ones, so independent crates can compile faster than one huge crate (measure first, Lesson 05.12).
- **Publishing.** A part that you want to publish as a library on a registry.

Do *not* split because "big projects have many crates". A workspace with fifteen crates of fifty lines each is a "micro-crates" smell: every public item must be
`pub`, every change touches three manifests, and nobody can see the whole program. A module is the right tool for tidiness; a crate is the right tool for a boundary.

## 5. The files around the code

A project root usually holds more than Cargo files:

| File | Purpose |
|---|---|
| `README.md` | What the project is, how to build and run it (Lesson 05.7). |
| `LICENSE` | The licence. Without one, others legally may not use your code. |
| `.gitignore` | Must contain `target/`. `cargo new` creates it only when it also creates a new Git repository; inside an existing repository (like your course folder) the root `.gitignore` must hold the rule. |
| `rust-toolchain.toml` | Pins the Rust version for the project (see below). |
| `rustfmt.toml`, `clippy.toml` | Format and lint settings (Lesson 08.11); most projects need neither. |
| `CHANGELOG.md` | What changed in each release (Lesson 05.7). |
| `deny.toml` | The `cargo-deny` policy for licences and advisories (Phase 10). |
| `.github/workflows/` | CI (Continuous Integration) pipelines (Phase 09). |

A `rust-toolchain.toml` makes everybody (and CI) use the same compiler:

```toml
[toolchain]
channel = "1.100.0"
components = ["rustfmt", "clippy"]
```

When rustup sees this file and the toolchain is not installed, it downloads it before running any `cargo` command; the first line you see is
`info: syncing channel updates for 1.100.0-aarch64-apple-darwin`. (Only add the file when you want that. A scratch project in your learning repository
does not need it.)

## Common mistakes

1. **Putting all logic in `main.rs`.** Integration tests cannot import a binary crate. Move the logic into `lib.rs`.
2. **Forgetting `default-run`** in a package with several binaries; `cargo run` then asks you to choose.
3. **Writing the package name with a dash in code** (`use shop-core::Item;`). In code it is `shop_core`.
4. **Different versions of one dependency in different members.** Put shared dependencies in `[workspace.dependencies]` and use `dep.workspace = true`.
5. **Forgetting `[lints] workspace = true`** in a member. The `[workspace.lints]` table does nothing until each member opts in.
6. **Cyclic dependencies.** Fix the direction (Section 3.4); do not merge the crates to hide it.

## Try it yourself

1. Build `p08-shop` exactly as in Section 3 and run `cargo run`, `cargo test --workspace` and `cargo clippy --workspace --all-targets -- -D warnings`.
2. Run `cargo tree` in the root. Find `shop-cli`, and see that `shop-core` and `shop-report` appear below it, and `thiserror` below `shop-core`.
3. Cause the cycle error of Section 3.4 (add `shop-report.workspace = true` under `[dependencies]` in `crates/shop-core/Cargo.toml`), read the message, and remove the line.
4. Commit: `git add code/p08-shop && git commit -m "feat(p08): shop workspace with three crates"`.

## Exercises

### Exercise 1 — Which file? (easy)

In which file or folder of a package do these belong? (a) a second program that converts a file; (b) a test that uses only the public functions; (c) a short program that
shows a user how to call your library; (d) a benchmark; (e) helper code shared by two modules but not meant for users.

<details>
<summary>Hint</summary>

Section 1, table of targets. For (e), think about visibility, not about folders.

</details>

<details>
<summary>Answer</summary>

(a) `src/bin/convert.rs`, run with `cargo run --bin convert`. (b) `tests/some_name.rs`. (c) `examples/some_name.rs`, run with `cargo run --example some_name`.
(d) `benches/some_name.rs`. (e) a module inside `src` with `pub(crate)` items.

</details>

### Exercise 2 — Add a crate with a test (medium)

Add a fourth member `shop-discount` to the workspace, with `pub fn apply_percent(cents: u64, percent: u64) -> u64` that takes `percent` percent off, rounding the discount down
and treating a percent above 100 as 100. Write tests for 10% of 1000, 50% of 999 and 150% of 1000. The new crate must inherit the version, edition, licence and lints from the workspace. Do you have to edit the root `Cargo.toml`?

<details>
<summary>Hint</summary>

`members = ["crates/*"]` picks up new folders. Copy the `[package]` and `[lints]` tables of another member. The discount of 50% of 999 is `999 * 50 / 100 = 499` (integer division), so the result is 500.

</details>

<details>
<summary>Answer</summary>

No edit of the root file is needed. `cargo new --lib crates/shop-discount`, then `crates/shop-discount/Cargo.toml`:

```toml
[package]
name = "shop-discount"
version.workspace = true
edition.workspace = true
license.workspace = true

[lints]
workspace = true
```

and `crates/shop-discount/src/lib.rs`:

```rust
//! Discount rules.

/// Takes `percent` percent off `cents`, rounding down.
///
/// A `percent` above 100 is treated as 100.
#[must_use]
pub fn apply_percent(cents: u64, percent: u64) -> u64 {
    let percent = percent.min(100);
    cents - cents * percent / 100
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn takes_a_percentage_off() {
        assert_eq!(apply_percent(1000, 10), 900);
        assert_eq!(apply_percent(999, 50), 500);
    }

    #[test]
    fn percent_is_capped_at_100() {
        assert_eq!(apply_percent(1000, 150), 0);
    }
}
```

`cargo test -p shop-discount` prints `test result: ok. 2 passed`. To use it in `shop-cli`, add `shop-discount = { path = "crates/shop-discount" }` to `[workspace.dependencies]`
and `shop-discount.workspace = true` to the `[dependencies]` of `shop-cli`.

</details>

### Exercise 3 — Break the cycle (hard)

A teammate wants `shop-core` to have a function `print_report(items)` that calls `shop_report::render`. Cargo reports a cyclic dependency. Describe two designs that avoid the cycle
without merging the crates.

<details>
<summary>Hint</summary>

Who is allowed to know about whom? Look at the arrows in Section 3. Think of "the lower crate defines a trait" (Lesson 08.9).

</details>

<details>
<summary>Answer</summary>

Design 1: do not put `print_report` in `shop-core`. Put it in `shop-report` (which may depend on `shop-core`) or in `shop-cli`. The domain stays free of output code.

Design 2: if the domain really must trigger the output, let it define a trait, for example `pub trait Reporter { fn report(&self, items: &[Item]); }`, and accept a `&dyn Reporter` or a generic
`R: Reporter`. The `shop-report` crate implements `Reporter`, and `shop-cli` passes the implementation in. The dependency arrow still points from `shop-report` to `shop-core`.

</details>

## Quiz

1. What is the difference between a package, a crate and a workspace?

<details>
<summary>Answer</summary>

A package has one `Cargo.toml` and can contain a library crate and binary crates. A crate is one compilation unit (a library or a program). A workspace is a group of packages
that share one `Cargo.lock` and one `target` folder.

</details>

2. Where does the logic of a program live, and what does `main.rs` do?

<details>
<summary>Answer</summary>

In `lib.rs` and its modules, where tests can reach it. `main.rs` is thin: it reads arguments, calls the library, prints and chooses the exit status.

</details>

3. What do `[workspace.dependencies]` and `dep.workspace = true` give you?

<details>
<summary>Answer</summary>

One place that declares each dependency and its version, so that all members use the same one and an upgrade is a one-line change.

</details>

4. Why can a crate dependency cycle not happen by accident inside one crate, and what does Cargo do between crates?

<details>
<summary>Answer</summary>

Inside one crate, modules may use each other freely, so cycles are possible and nobody stops them. Between crates, Cargo refuses to build ("cyclic package dependency"), which
enforces the direction of dependencies.

</details>

## Summary

- A package builds several *targets* from conventional paths: `src/lib.rs`, `src/main.rs`, `src/bin/*.rs`, `tests/`, `examples/`, `benches/`. Put the logic in the library and keep programs thin.
- Use `default-run` when a package has several programs.
- Inside `src`: `lib.rs` as a table of contents, small focused modules named after what they are about, private by default, `pub(crate)` for internal sharing, `pub use` to hide your layout,
  and feature folders before layer folders.
- A workspace shares `Cargo.lock`, `target`, versions (`[workspace.package]`), dependencies (`[workspace.dependencies]`) and lint policy (`[workspace.lints]`).
- Dependencies point from the outside (programs) to the inside (domain); Cargo rejects cycles between crates.
- Start with one package; split when you can name the reason (a boundary, different users or dependencies, publishing). Avoid micro-crates.

## Sources and further reading

- [The Cargo Book — Cargo Targets](https://doc.rust-lang.org/cargo/reference/cargo-targets.html) — all target kinds and how Cargo finds them.
- [The Cargo Book — Package layout](https://doc.rust-lang.org/cargo/guide/project-layout.html) — the standard layout of a package.
- [The Cargo Book — Workspaces](https://doc.rust-lang.org/cargo/reference/workspaces.html) — `[workspace]`, inheritance of package fields, dependencies and lints.
- [The Rust Programming Language — Chapter 7: Managing Growing Projects with Packages, Crates, and Modules](https://doc.rust-lang.org/book/ch07-00-managing-growing-projects-with-packages-crates-and-modules.html) — modules, paths and visibility.
- [The Rust Programming Language — Chapter 14.3: Cargo Workspaces](https://doc.rust-lang.org/book/ch14-03-cargo-workspaces.html) — a first workspace, step by step.
- [The rustup book — Overrides](https://rust-lang.github.io/rustup/overrides.html) — the `rust-toolchain.toml` file.

---

[🏠 Start](../START-HERE.md) · [← 08.11 Conventions: naming, Clippy and the Rust API Guidelines](11-conventions-naming-clippy-and-api-guidelines.md) · [📋 Phase 08](README.md) · [08.13 Refactoring and code review →](13-refactoring-and-code-review.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
