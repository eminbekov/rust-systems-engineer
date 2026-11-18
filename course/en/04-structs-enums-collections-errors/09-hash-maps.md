<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 04.8 Strings in depth](08-strings-in-depth.md) · [📋 Phase 04](README.md) · [04.10 Error handling →](10-error-handling.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/04-structs-enums-collections-errors/09-hash-maps.md) <!-- nav -->

# Lesson 04.9 — Hash maps

> **Phase 04 — Structs, enums, collections and errors** · Lesson 9 of 12 · ⏱ about 4 hours (2 sittings)

## What you will learn

- What a hash map is and when to choose it instead of a vector.
- How to insert, read, update and remove entries, and how to use the `entry` API.
- Why the iteration order of a `HashMap` is random, and how to get a sorted result.
- How ownership works for keys and values.
- What makes a type usable as a key, and when to use `BTreeMap` or `HashSet` instead.

## Before you start

- You finished [Lesson 04.8 — Strings in depth](08-strings-in-depth.md).
- You know vectors ([Lesson 04.7](07-vectors.md)), `Option` and `match` ([Lesson 04.4](04-option-and-match.md)), and how `use` brings a name into scope ([Lesson 04.6](06-modules-crates-and-packages.md)).
- Create a project:

```bash
cd ~/projects/rust
cargo new code/p04-hash-maps
cd code/p04-hash-maps
```

## New words

| Word | Meaning |
|---|---|
| **hash map** | A collection of **key → value** pairs. You give a key, you get the value stored for it. In Rust: `HashMap<K, V>`. |
| **key** | The label you use to find a value. Each key appears at most once in a map. |
| **value** | The data stored under a key. |
| **entry** | One key together with its value. Also the name of a special Rust type that represents "the place for this key, which may be empty". |
| **hash function** | A function that turns any value into a number. The same value always gives the same number. |
| **hash** | The number a hash function produces. |
| **bucket** | One slot in the hash map's internal table where entries with a given hash are kept. |
| **collision** | Two different keys that land in the same bucket. The map handles it for you. |
| **trait** | A named set of abilities that a type can have, such as "can be compared" or "can be hashed". Full lesson: [Lesson 05.2 — Traits](../05-rust-intermediate/02-traits.md). |
| **`derive`** | The attribute `#[derive(...)]` asks the compiler to write the code for a trait automatically. You used `#[derive(Debug)]` already. |
| **set** | A collection of unique values with no duplicates. In Rust: `HashSet<T>` and `BTreeSet<T>`. |
| **DoS** (Denial of Service) | An attack that makes a program too slow or too busy to serve real users. |
| **API** (Application Programming Interface) | The set of functions, types and methods that a library or a type offers to other code. "The `entry` API" means the group of methods that start with `entry`. |
| **`NaN`** (Not a Number) | A special floating-point value for a result that is not a number, such as `0.0 / 0.0`. It is not equal to anything, not even to itself. |

## 1. What is a hash map?

A vector finds an item by its **position** (`v[2]`). Often that is not what you have. You have a
**name** and want a phone number; a **word** and want its count; a **user id** and want the user.

A **hash map** stores pairs: a **key** and a **value**. You ask for a key and get back the value.
Think of a coat check: you hand over a coat and get a numbered ticket (the key). Later you show
the ticket and get the coat (the value) at once, without anyone looking through all the coats.

```text
key        value
"alice"  → "+998-90-111-22-33"
"bob"    → "+998-91-444-55-66"
"carol"  → "+998-93-777-88-99"
```

The map is fast: finding a value takes about the same time whether the map has 10 entries or
10 million. A vector would have to compare against every item one by one.

> `HashMap` is not in the *prelude* (the small set of names Rust brings in automatically). You
> must import it at the top of the file: `use std::collections::HashMap;`.

## 2. Creating and using a map

```rust
use std::collections::HashMap;

fn main() {
    let mut ages: HashMap<String, u32> = HashMap::new();
    ages.insert(String::from("Ali"), 31);
    ages.insert(String::from("Sara"), 27);
    ages.insert(String::from("Vali"), 45);

    println!("size: {}", ages.len());
    println!("Ali: {:?}", ages.get("Ali"));
    println!("Zoe: {:?}", ages.get("Zoe"));
    println!("has Sara? {}", ages.contains_key("Sara"));

    let old = ages.insert(String::from("Ali"), 32);
    println!("old value: {:?}", old);
    println!("Ali now: {:?}", ages.get("Ali"));

    let removed = ages.remove("Vali");
    println!("removed: {:?}", removed);
    println!("size: {}", ages.len());

    let langs = HashMap::from([("rust", 2015), ("python", 1991), ("c", 1972)]);
    println!("c appeared in {}", langs["c"]);
}
```

**What happens:**

1. `HashMap<String, u32>` means: keys are `String`, values are `u32`. The two types go in the angle
   brackets, key first.
2. `insert(key, value)` adds an entry. The `String` keys and values are **moved** into the map
   (section 5).
3. `get("Ali")` looks up a key and returns `Option<&V>`: `Some(&31)` or `None` when the key is
   missing. You can look up a `String` key with a `&str`, so you do not need to create a `String`
   just to search.
4. `contains_key` answers yes or no. `len` returns the number of entries.
5. `insert` with an existing key **replaces** the value and returns the old one as
   `Some(old)`. For a new key it returns `None`.
6. `remove("Vali")` deletes the entry and returns the removed value in an `Option`.
7. `HashMap::from([...])` builds a map from an array of `(key, value)` pairs. Here Rust infers
   `HashMap<&str, i32>` itself. `langs["c"]` reads a value with square brackets.

```text
size: 3
Ali: Some(31)
Zoe: None
has Sara? true
old value: Some(31)
Ali now: Some(32)
removed: Some(45)
size: 2
c appeared in 1972
```

### `get` or `[]`?

Like a vector, a map has two ways to read. `map[key]` is short, but if the key is missing the
program **panics**:

```rust
use std::collections::HashMap;

fn main() {
    let mut m: HashMap<String, i32> = HashMap::new();
    m.insert(String::from("a"), 1);
    println!("{}", m["missing"]);
}
```

```text

thread 'main' (3928098) panicked at src/main.rs:6:21:
no entry found for key
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
```

> 📏 **Convention:** Use `get` when the key may be missing (nearly always, if the key comes from a
> user or a file). Use `map[key]` only when a missing key would be a bug in your own code.

## 3. Changing values

To change a value in place, ask for a mutable reference with `get_mut`.

```rust
use std::collections::HashMap;

fn main() {
    let mut stock: HashMap<&str, u32> = HashMap::new();
    stock.insert("apples", 10);

    match stock.get("apples") {
        Some(n) => println!("we have {n} apples"),
        None => println!("no apples"),
    }

    if let Some(n) = stock.get_mut("apples") {
        *n += 5;
    }
    println!("{:?}", stock.get("apples"));

    let pears = stock.get("pears").copied().unwrap_or(0);
    println!("pears: {pears}");
}
```

**What happens:**

1. `get("apples")` gives `Some(&10)`; `n` is a reference, and `{n}` prints the number behind it.
2. `get_mut("apples")` gives `Some(&mut 10)`. Writing `*n += 5` changes the stored value
   ([Lesson 03.5](../03-ownership-and-borrowing/05-mutable-references-and-the-borrow-rules.md)). `if let` runs the block only when the key exists ([Lesson 04.5](05-if-let-let-else-and-patterns.md)).
3. `get("pears")` is `None`. `.copied()` turns `Option<&u32>` into `Option<u32>`, and
   `unwrap_or(0)` replaces `None` with `0`. This is the shortest way to say "the value, or a
   default".

```text
we have 10 apples
Some(15)
pears: 0
```

## 4. The `entry` API: insert if missing, then update

A very common job: "if the key is there, change its value; if not, create it first." Doing it
with `get` and `insert` takes several lines and two lookups. The **entry API** does it in one. (An **API**, Application Programming Interface, is simply the set of methods that a type offers; the entry API is the methods around `entry`.)

`map.entry(key)` returns an *entry*: a handle to the place where the key lives, whether or not
the place is filled. Then you decide:

- `.or_insert(default)` — if the place is empty, store `default`. Either way, return a mutable
  reference `&mut V` to the value.
- `.or_default()` — the same, but the default is the type's *default value* (`0` for numbers, an
  empty `String`, an empty `Vec`). A "default value" comes from the `Default` trait.

**Counting words** — the classic use:

```rust
use std::collections::HashMap;

fn main() {
    let text = "the cat and the dog and the bird";
    let mut counts: HashMap<String, u32> = HashMap::new();

    for word in text.split_whitespace() {
        let count = counts.entry(word.to_string()).or_insert(0);
        *count += 1;
    }

    let mut words: Vec<&String> = counts.keys().collect();
    words.sort();
    for w in words {
        println!("{w}: {}", counts[w]);
    }
}
```

**What happens:**

1. For each word, `entry(word.to_string())` finds the place for that word.
2. `or_insert(0)` stores `0` if the word is new, and returns `&mut u32` to the value either way.
3. `*count += 1` adds one through the reference. First time: `0 → 1`. Second time: `1 → 2`.
4. `counts.keys()` is an iterator over the keys; `.collect()` puts them in a `Vec<&String>`
   ([Lesson 04.8](08-strings-in-depth.md)). We sort that vector and print in order (section 6 explains why).

```text
and: 2
bird: 1
cat: 1
dog: 1
the: 3
```

**Grouping** — keep a list of items per key. `or_default()` creates an empty `Vec` the first time:

```rust
use std::collections::HashMap;

fn main() {
    let names = ["Ali", "Anvar", "Bobur", "Alisher", "Bakhtiyor", "Dilnoza"];
    let mut by_letter: HashMap<char, Vec<&str>> = HashMap::new();

    for name in names {
        if let Some(first) = name.chars().next() {
            by_letter.entry(first).or_default().push(name);
        }
    }

    let mut letters: Vec<char> = Vec::new();
    for letter in by_letter.keys() {
        letters.push(*letter);
    }
    letters.sort();

    for letter in letters {
        println!("{letter}: {:?}", by_letter[&letter]);
    }
}
```

**What happens:**

1. `by_letter` maps a first letter to a vector of names that start with it.
2. `entry(first).or_default()` returns `&mut Vec<&str>` (a new empty vector the first time), and
   `.push(name)` appends to it.
3. We copy the keys into a `Vec<char>`, sort it, and print the groups in alphabetical order.

```text
A: ["Ali", "Anvar", "Alisher"]
B: ["Bobur", "Bakhtiyor"]
D: ["Dilnoza"]
```

> 💡 **Tip:** When you catch yourself writing "if the map contains the key { ... } else { insert }",
> use `entry` instead. It is shorter, faster (one lookup) and has fewer ways to go wrong.

## 5. Ownership of keys and values

Inserting follows the ownership rules you know ([Lesson 03.2](../03-ownership-and-borrowing/02-ownership-rules.md)):

- types that implement `Copy` (numbers, `char`, `bool`) are **copied** into the map;
- types like `String` and `Vec` are **moved**: the map becomes the owner.

```rust
use std::collections::HashMap;

fn main() {
    let mut m = HashMap::new();
    let key = String::from("alice");
    m.insert(key, 1);
    println!("{}", key);
}
```

```text
error[E0382]: borrow of moved value: `key`
 --> src/main.rs:7:20
  |
5 |     let key = String::from("alice");
  |         --- move occurs because `key` has type `String`, which does not implement the `Copy` trait
6 |     m.insert(key, 1);
  |              --- value moved here
7 |     println!("{}", key);
  |                    ^^^ value borrowed here after move
  |
help: consider cloning the value if the performance cost is acceptable
  |
6 |     m.insert(key.clone(), 1);
  |                 ++++++++
```

**What the error means:** `insert` took the `String` away from `key`. The map owns it now. If you
still need the text, insert a copy (`key.clone()`) or read it back from the map.

You can also store **references** as keys or values (like `HashMap<&str, u32>` above), but then
the data they point to must live at least as long as the map. That rule is the subject of
[Lesson 05.3 — Lifetimes](../05-rust-intermediate/03-lifetimes.md). In this phase, storing owned `String`s is the simplest choice.

## 6. Iterating: the order is random

```rust
use std::collections::HashMap;

fn main() {
    let mut m: HashMap<String, i32> = HashMap::new();
    m.insert(String::from("one"), 1);
    m.insert(String::from("two"), 2);
    m.insert(String::from("three"), 3);
    m.insert(String::from("four"), 4);

    println!("--- order is not promised:");
    for (key, value) in &m {
        println!("{key} = {value}");
    }

    println!("--- add 10 to every value:");
    for value in m.values_mut() {
        *value += 10;
    }

    println!("--- sorted by key:");
    let mut keys: Vec<&String> = m.keys().collect();
    keys.sort();
    for k in keys {
        println!("{k} = {}", m[k]);
    }
}
```

**What happens:**

1. `for (key, value) in &m` walks over all entries. Each round gives a pair of references, which
   the pattern `(key, value)` splits ([Lesson 04.5](05-if-let-let-else-and-patterns.md)).
2. `m.values_mut()` gives mutable references to the values only; there is also `m.keys()` and
   `m.values()`.
3. To print in a stable order we collect the keys into a vector, sort it, and read each value with
   `m[k]`.

```text
--- order is not promised:
two = 2
four = 4
three = 3
one = 1
--- add 10 to every value:
--- sorted by key:
four = 14
one = 11
three = 13
two = 12
```

The first block is **different on every run**: when we ran the same program three times, the
keys came out as `four, one, two, three`, then `one, four, two, three`, then `two, four, one, three`.
Rust randomizes the order on purpose. (Your order will differ from ours, and that is fine.)

> ⚠️ **Common mistake:** Relying on the order of a `HashMap` (in output, in tests, in files). Your
> program may work today and print entries differently tomorrow. If order matters, **sort** the
> keys first, or use a `BTreeMap` (section 8).

## 7. How a hash map works inside

You do not need this to use a map, but it explains the rules you meet next.

```text
key "alice" ──▶ hash function ──▶ 8247019 ──▶ 8247019 mod 4 = 3 ──▶ bucket 3
key "bob"   ──▶ hash function ──▶ 1139284 ──▶ 1139284 mod 4 = 0 ──▶ bucket 0

 bucket 0: ("bob",   "+998-91-444-55-66")
 bucket 1: (empty)
 bucket 2: (empty)
 bucket 3: ("alice", "+998-90-111-22-33")
```

(The numbers are made up; real hashes are 64-bit numbers.) To `insert`, the map runs the key through
a **hash function** to get a number, uses the number to choose a **bucket**, and stores the
entry there. To `get`, it repeats the same steps and looks in just that bucket. It never scans
the whole table. When too many entries share a bucket (**collision**) or the table gets full,
the map resizes itself, like a vector reallocating.

Two rules follow from this design:

**Rule 1 — the key must be hashable and comparable.** The type needs the traits `Hash` (can be turned into
   a number) and `Eq` (can be compared for equality exactly). `String`, `&str`, integers, `char`,
   `bool` and tuples of those all have them. Floats do **not**, because `NaN` (Not a Number, the special float value for results like `0.0 / 0.0`) is not equal to itself, so equality is not exact. Trying to use
   `f64` as a key fails:

```rust
use std::collections::HashMap;

fn main() {
    let mut prices: HashMap<f64, &str> = HashMap::new();
    prices.insert(1.5, "tea");
}
```

```text
error[E0599]: the method `insert` exists for struct `HashMap<f64, &str>`, but its trait bounds were not satisfied
 --> src/main.rs:5:12
  |
5 |     prices.insert(1.5, "tea");
  |            ^^^^^^
  |
  = note: the following trait bounds were not satisfied:
          `f64: Eq`
          `f64: Hash`
```

The fix is to use another key (for example, the price in cents as an integer).

**Rule 2 — a key must never change while it is in the map**, otherwise its hash would change and the map
   could not find it. Rust makes this easy for you: the map owns its keys and does not let you edit them.

Your own types can be keys too. Ask the compiler to write the two traits (plus `PartialEq`,
which `Eq` builds on) with `derive`:

```rust
use std::collections::HashMap;

#[derive(Debug, Hash, PartialEq, Eq)]
enum Color {
    Red,
    Green,
}

fn main() {
    let mut grid: HashMap<(i32, i32), char> = HashMap::new();
    grid.insert((0, 0), 'S');
    grid.insert((3, 4), 'E');
    println!("{:?}", grid.get(&(3, 4)));
    println!("{:?}", grid.get(&(1, 1)));

    let mut votes: HashMap<Color, u32> = HashMap::new();
    *votes.entry(Color::Red).or_insert(0) += 1;
    *votes.entry(Color::Red).or_insert(0) += 1;
    *votes.entry(Color::Green).or_insert(0) += 1;
    println!("red: {:?}, green: {:?}", votes.get(&Color::Red), votes.get(&Color::Green));
}
```

**What happens:**

1. A tuple of integers `(i32, i32)` works as a key out of the box: a handy way to store a grid
   or a map of coordinates.
2. `#[derive(Hash, PartialEq, Eq)]` gives `Color` the abilities a key needs. Without it, you get
   the same kind of error as for `f64`.
3. `entry(Color::Red).or_insert(0)` then `+= 1` counts the votes.

```text
Some('E')
None
red: Some(2), green: Some(1)
```

> 🔒 **Security:** If an attacker can choose your keys (words in a request, names in a form) and
> finds many keys with the same hash, the map becomes slow and the program can be overwhelmed.
> This is a **DoS** (Denial of Service) attack called *hash flooding*. Rust's default hasher is
> deliberately designed to resist it by mixing in a random secret that is different for every map. It is a
> bit slower than simple hashers, and that is the right default. Do not replace it until you have
> measured a real problem and understood the risk.

## 8. `BTreeMap` and `HashSet`

**`BTreeMap<K, V>`** is a map that keeps its entries **sorted by key**. It lives in
`std::collections` too. The keys need the trait `Ord` (can be put in order), not `Hash`.

```rust
use std::collections::BTreeMap;

fn main() {
    let mut m: BTreeMap<&str, i32> = BTreeMap::new();
    m.insert("one", 1);
    m.insert("two", 2);
    m.insert("three", 3);
    m.insert("four", 4);

    for (key, value) in &m {
        println!("{key} = {value}");
    }
    println!("first: {:?}", m.first_key_value());
    println!("last:  {:?}", m.last_key_value());
}
```

**What happens:** the loop prints the entries in alphabetical key order every time; no sorting step
is needed. `first_key_value()` and `last_key_value()` return the smallest and largest key with its
value, as an `Option`.

```text
four = 4
one = 1
three = 3
two = 2
first: Some(("four", 4))
last:  Some(("two", 2))
```

| | `HashMap` | `BTreeMap` |
|---|---|---|
| Order of iteration | random | sorted by key |
| Key needs | `Hash` + `Eq` | `Ord` |
| Lookup speed | very fast (constant time on average) | fast (grows slowly with size) |
| Use when | you just look things up | you need sorted output or "smallest/largest key" or ranges |

**`HashSet<T>`** is a collection of **unique** values. It is like a `HashMap` with keys only.
`insert` returns `true` when the value is new and `false` when it was already there.

```rust
use std::collections::HashSet;

fn main() {
    let mut seen: HashSet<String> = HashSet::new();
    for word in ["a", "b", "a", "c", "b"] {
        let is_new = seen.insert(String::from(word));
        println!("{word}: new? {is_new}");
    }
    println!("unique words: {}", seen.len());
    println!("contains b? {}", seen.contains("b"));
}
```

```text
a: new? true
b: new? true
a: new? false
c: new? true
b: new? false
unique words: 3
contains b? true
```

`BTreeSet<T>` is the sorted version of a set.

## 9. Testing code that uses maps

> 🧪 **Testing:** Because the order of a `HashMap` is random, never compare its printed text in a
> test. Check single entries with `get`, or compare two whole maps with `==` (two maps are equal
> when they have the same entries, in any order).

```rust
use std::collections::HashMap;

/// Counts how many times each word appears. Words are compared in lower case.
fn word_counts(text: &str) -> HashMap<String, u32> {
    let mut counts = HashMap::new();
    for word in text.split_whitespace() {
        *counts.entry(word.to_lowercase()).or_insert(0) += 1;
    }
    counts
}

fn main() {
    let counts = word_counts("Rust rust RUST is fun");
    println!("rust: {:?}", counts.get("rust"));
    println!("fun:  {:?}", counts.get("fun"));
    println!("go:   {:?}", counts.get("go"));
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn counts_ignoring_case() {
        let counts = word_counts("Rust rust RUST is fun");
        assert_eq!(counts.get("rust"), Some(&3));
        assert_eq!(counts.get("fun"), Some(&1));
        assert_eq!(counts.get("go"), None);
    }

    #[test]
    fn empty_text_gives_empty_map() {
        assert!(word_counts("").is_empty());
    }

    #[test]
    fn whole_map_comparison() {
        let expected = HashMap::from([(String::from("a"), 2), (String::from("b"), 1)]);
        assert_eq!(word_counts("a b a"), expected);
    }
}
```

**What happens:**

1. `word_counts` builds and returns a map; the caller becomes its owner.
2. `counts.get("rust")` returns `Option<&u32>`, so the test compares with `Some(&3)`: a reference to 3.
3. `use super::*;` inside `mod tests` imports everything from the parent module, including the
   `HashMap` name ([Lesson 04.6](06-modules-crates-and-packages.md)).

```text
rust: Some(3)
fun:  Some(1)
go:   None
```

```text
running 3 tests
test tests::empty_text_gives_empty_map ... ok
test tests::whole_map_comparison ... ok
test tests::counts_ignoring_case ... ok

test result: ok. 3 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

> ⚡ **Performance:** `HashMap::with_capacity(n)` reserves room for `n` entries and avoids resizing,
> just like for a vector. A small `Vec` of pairs can beat a map for a handful of items, but do not
> guess: measure.

## Common mistakes

**1. Forgetting the import.**

```rust
fn main() {
    let m: HashMap<String, i32> = HashMap::new();
    println!("{}", m.len());
}
```

```text
error[E0425]: cannot find type `HashMap` in this scope
 --> src/main.rs:2:12
  |
2 |     let m: HashMap<String, i32> = HashMap::new();
  |            ^^^^^^^ not found in this scope
  |
help: consider importing this struct
  |
1 + use std::collections::HashMap;
  |
```

(A second error, E0433, points at `HashMap::new()`.) Fix: add `use std::collections::HashMap;`.

**2. Passing a value where `get` wants a reference.**

```rust
use std::collections::HashMap;

fn main() {
    let mut m: HashMap<String, i32> = HashMap::new();
    m.insert(String::from("a"), 1);
    let name = String::from("a");
    println!("{:?}", m.get(name));
}
```

```text
error[E0308]: mismatched types
 --> src/main.rs:7:28
  |
7 |     println!("{:?}", m.get(name));
  |                        --- ^^^^ expected `&_`, found `String`
  |                        |
  |                        arguments to this method are incorrect
  |
help: consider borrowing here
  |
7 |     println!("{:?}", m.get(&name));
  |                            +
```

`get` borrows the key to look it up, so it needs a reference: `m.get(&name)`.

**3. Changing the map while looping over it.**

```rust
use std::collections::HashMap;

fn main() {
    let mut m: HashMap<String, i32> = HashMap::new();
    m.insert(String::from("a"), 1);
    for (k, v) in &m {
        if *v == 1 {
            m.insert(format!("{k}2"), 2);
        }
    }
}
```

```text
error[E0502]: cannot borrow `m` as mutable because it is also borrowed as immutable
  --> src/main.rs:8:13
   |
 6 |       for (k, v) in &m {
   |       -             --
   |       |             |
   |       |             immutable borrow occurs here
   |  _____|             immutable borrow later used here
   | |
 7 | |         if *v == 1 {
 8 | |             m.insert(format!("{k}2"), 2);
   | |             ^^^^^^^^^^^^^^^^^^^^^^^^^^^^ mutable borrow occurs here
 9 | |         }
10 | |     }
   | |_____- this for loop borrows `m` immutably, preventing mutation within its body
```

The same idea as pushing to a vector during a loop ([Lesson 04.7](07-vectors.md)): inserting may resize the table
and invalidate the references the loop holds. Fix: collect what you want to add into a separate
`Vec` during the loop and insert after the loop.

**4. Using a `String` after inserting it** — see section 5.

**5. Using a float as a key** — see section 7.

**6. Expecting a fixed order** — see section 6.

## Try it yourself

In `code/p04-hash-maps`, write a program that counts how often each letter appears in a word and
prints the result in alphabetical order:

```rust
use std::collections::BTreeMap;

fn main() {
    let word = "mississippi";
    let mut counts: BTreeMap<char, u32> = BTreeMap::new();
    for c in word.chars() {
        *counts.entry(c).or_insert(0) += 1;
    }
    for (letter, n) in &counts {
        println!("{letter}: {n}");
    }
}
```

1. Run it with `cargo run`. Expected output:

```text
i: 4
m: 1
p: 2
s: 4
```

2. Change `BTreeMap` to `HashMap` (also in the `use` line) and run it three times. Does the order
   stay the same?
3. Change the word to `"Mississippi"`. Is `M` counted together with `m`? How could you fix that? (Use
   `c.to_lowercase()` from [Lesson 04.8](08-strings-in-depth.md).)
4. Commit: `git add .` and `git commit -m "Count letters with a map"`.

## Exercises

### Exercise 1 — A tiny phone book (easy)

Create a `HashMap<String, String>` with `alice → +998-90-111-22-33` and `bob → +998-91-444-55-66`.
Look up `"alice"` and `"carol"` and print either the number or `not found`. Expected output:

```text
alice: +998-90-111-22-33
carol: not found
```

<details>
<summary>Hint</summary>

Use `insert` twice. Loop over the array `["alice", "carol"]` and `match` on `phones.get(name)`.

</details>

<details>
<summary>Answer</summary>

```rust
use std::collections::HashMap;

fn main() {
    let mut phones: HashMap<String, String> = HashMap::new();
    phones.insert(String::from("alice"), String::from("+998-90-111-22-33"));
    phones.insert(String::from("bob"), String::from("+998-91-444-55-66"));

    for name in ["alice", "carol"] {
        match phones.get(name) {
            Some(number) => println!("{name}: {number}"),
            None => println!("{name}: not found"),
        }
    }
}
```

Why it works: `get` accepts a `&str` for a `String` key and returns `None` for a missing key, so
nothing panics.

</details>

### Exercise 2 — Inventory (easy)

Make a stock map with `apples → 10` and `pears → 4`. Remove 3 apples with `get_mut`, delete the pears with
`remove`, and add 6 plums with `entry`. Print the final stock sorted by name. Expected output:

```text
apples: 7
plums: 6
```

<details>
<summary>Hint</summary>

`if let Some(n) = stock.get_mut("apples") { *n -= 3; }`. For plums:
`*stock.entry(String::from("plums")).or_insert(0) += 6;`. To print sorted, collect the keys into a
`Vec<&String>` and sort it.

</details>

<details>
<summary>Answer</summary>

```rust
use std::collections::HashMap;

fn main() {
    let mut stock: HashMap<String, u32> = HashMap::new();
    stock.insert(String::from("apples"), 10);
    stock.insert(String::from("pears"), 4);

    if let Some(n) = stock.get_mut("apples") {
        *n -= 3;
    }
    stock.remove("pears");
    *stock.entry(String::from("plums")).or_insert(0) += 6;

    let mut names: Vec<&String> = stock.keys().collect();
    names.sort();
    for name in names {
        println!("{name}: {}", stock[name]);
    }
}
```

Why it works: `get_mut` edits an existing value, `remove` deletes an entry, and `entry` creates a
missing one. Sorting the keys makes the output stable.

</details>

### Exercise 3 — First duplicate, with tests (medium)

Write `fn first_duplicate(v: &[i32]) -> Option<i32>` that returns the first number that
appears for the second time. For `[3, 1, 4, 1, 5, 3]` the answer is `Some(1)` (1 repeats before 3
does). For `[1, 2, 3]` it is `None`. Write three tests: a repeat, no repeat and an empty slice.

<details>
<summary>Hint</summary>

Use a `HashSet<i32>`. For each number call `seen.insert(n)`. It returns `false` when the number was
already in the set, and that is the moment to `return Some(n)`.

</details>

<details>
<summary>Answer</summary>

```rust
use std::collections::HashSet;

/// Returns the first number that appears a second time, or None.
fn first_duplicate(v: &[i32]) -> Option<i32> {
    let mut seen = HashSet::new();
    for &n in v {
        if !seen.insert(n) {
            return Some(n);
        }
    }
    None
}

fn main() {
    println!("{:?}", first_duplicate(&[3, 1, 4, 1, 5, 3]));
    println!("{:?}", first_duplicate(&[1, 2, 3]));
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn finds_first_repeat() {
        assert_eq!(first_duplicate(&[3, 1, 4, 1, 5, 3]), Some(1));
    }

    #[test]
    fn no_repeat() {
        assert_eq!(first_duplicate(&[1, 2, 3]), None);
    }

    #[test]
    fn empty() {
        assert_eq!(first_duplicate(&[]), None);
    }
}
```

```text
Some(1)
None
```

Why it works: checking and inserting is one step, so each number is looked at once. Rust
infers `HashSet<i32>` from the `insert(n)` call.

</details>

### Exercise 4 — Group words by length (medium)

Given `["tree", "cat", "sun", "house", "dog", "moon", "a"]`, build a `HashMap<usize, Vec<&str>>`
that groups the words by their length, and print the groups in order of length. Expected output:

```text
1: ["a"]
3: ["cat", "sun", "dog"]
4: ["tree", "moon"]
5: ["house"]
```

<details>
<summary>Hint</summary>

`by_len.entry(w.len()).or_default().push(w);` does the grouping. To print in order, copy the keys
into a `Vec<usize>` with a loop, sort it, and print `by_len[&len]`.

</details>

<details>
<summary>Answer</summary>

```rust
use std::collections::HashMap;

fn main() {
    let words = ["tree", "cat", "sun", "house", "dog", "moon", "a"];
    let mut by_len: HashMap<usize, Vec<&str>> = HashMap::new();
    for w in words {
        by_len.entry(w.len()).or_default().push(w);
    }

    let mut lengths: Vec<usize> = Vec::new();
    for len in by_len.keys() {
        lengths.push(*len);
    }
    lengths.sort();
    for len in lengths {
        println!("{len}: {:?}", by_len[&len]);
    }
}
```

Why it works: `or_default()` creates an empty vector the first time a length is seen. The words
inside each group keep the order in which they were added.

</details>

### Exercise 5 — Most common word, with tests (hard)

Write `fn most_common(text: &str) -> Option<(String, u32)>` that returns the most frequent word
(compared in lower case) and its count. If several words tie, the alphabetically smallest wins.
An empty text gives `None`. Tests: a clear winner, a tie, mixed case and empty text.

<details>
<summary>Hint</summary>

First build the counts with `entry` (like `word_counts` in section 9). Then keep a variable
`best: Option<(String, u32)>` and loop over the map. A pair is better when `best` is `None`, or its
count is bigger, or the count is equal and the word is smaller. Because a map has no order, the
tie rule is what makes the result predictable.

</details>

<details>
<summary>Answer</summary>

```rust
use std::collections::HashMap;

/// The most frequent word (lower case) and its count.
/// If several words tie, the alphabetically smallest wins.
fn most_common(text: &str) -> Option<(String, u32)> {
    let mut counts: HashMap<String, u32> = HashMap::new();
    for word in text.split_whitespace() {
        *counts.entry(word.to_lowercase()).or_insert(0) += 1;
    }

    let mut best: Option<(String, u32)> = None;
    for (word, count) in counts {
        let better = match &best {
            None => true,
            Some((best_word, best_count)) => {
                count > *best_count || (count == *best_count && word < *best_word)
            }
        };
        if better {
            best = Some((word, count));
        }
    }
    best
}

fn main() {
    println!("{:?}", most_common("the cat and the dog and the bird"));
    println!("{:?}", most_common("b a b a"));
    println!("{:?}", most_common(""));
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn finds_the_winner() {
        assert_eq!(most_common("the cat and the dog and the bird"), Some((String::from("the"), 3)));
    }

    #[test]
    fn tie_goes_to_smaller_word() {
        assert_eq!(most_common("b a b a"), Some((String::from("a"), 2)));
    }

    #[test]
    fn case_is_ignored() {
        assert_eq!(most_common("Go go GO"), Some((String::from("go"), 3)));
    }

    #[test]
    fn empty_text() {
        assert_eq!(most_common(""), None);
    }
}
```

```text
Some(("the", 3))
Some(("a", 2))
None
```

Why it works: `for (word, count) in counts` consumes the map and gives owned pairs, so we can move
`word` into `best`. `match &best` only borrows `best` while we compare, so assigning to `best`
afterwards is allowed. The tie rule removes the dependence on the random order.

</details>

### Exercise 6 — Remember results (hard)

The Fibonacci numbers are `0, 1, 1, 2, 3, 5, 8, …` (each is the sum of the two before it). A
simple recursive function repeats the same work an enormous number of times. Write
`fn fib(n: u64, memo: &mut HashMap<u64, u64>) -> u64` that stores each result in the map
(*memoization*: remembering answers) and reuses it. Print `fib(10)`, `fib(80)` and how many values
were stored. Expected output:

```text
fib(10) = 55
fib(80) = 23416728348467685
cached values: 79
```

<details>
<summary>Hint</summary>

If `n <= 1`, return `n`. Next check `memo.get(&n)`: if it is `Some(&cached)`, return `cached`. Otherwise compute
`fib(n - 1, memo) + fib(n - 2, memo)`, `insert` it, and return it. Pass the map on with `memo`
(it is already a `&mut`), and call from `main` with `&mut memo`.

</details>

<details>
<summary>Answer</summary>

```rust
use std::collections::HashMap;

fn fib(n: u64, memo: &mut HashMap<u64, u64>) -> u64 {
    if n <= 1 {
        return n;
    }
    if let Some(&cached) = memo.get(&n) {
        return cached;
    }
    let result = fib(n - 1, memo) + fib(n - 2, memo);
    memo.insert(n, result);
    result
}

fn main() {
    let mut memo = HashMap::new();
    println!("fib(10) = {}", fib(10, &mut memo));
    println!("fib(80) = {}", fib(80, &mut memo));
    println!("cached values: {}", memo.len());
}
```

Why it works: each value from 2 to 80 is computed once and stored (79 values). The `if let` copies
the number out of the reference (`&cached`) before the recursion, so no borrow of `memo` is alive
when we call `fib` again. Without the map, `fib(80)` would take longer than you are willing to wait.

</details>

## Quiz

1. What is the difference between finding an item in a vector and in a hash map?

<details>
<summary>Answer</summary>

A vector finds an item by its position (index). A hash map finds a value by its key. A map's
lookup takes about the same time regardless of its size.

</details>

2. What does `map.insert(key, value)` return when the key already exists?

<details>
<summary>Answer</summary>

`Some(old_value)`: the value is replaced and the previous one is returned. For a new key it
returns `None`.

</details>

3. Why does `map.get("x")` return an `Option`?

<details>
<summary>Answer</summary>

Because the key may not be in the map. `Some(&value)` means it was found; `None` means it was not.

</details>

4. What does `*map.entry(word).or_insert(0) += 1;` do?

<details>
<summary>Answer</summary>

It looks up the entry for `word`, stores `0` there if it was missing, and then adds 1 to the value
through the returned mutable reference. It counts how many times `word` was seen, with one lookup.

</details>

5. Why is the order of iteration over a `HashMap` not the same on every run? How can you get a stable order?

<details>
<summary>Answer</summary>

The order depends on the hashes, and Rust randomizes the hashing on purpose (it protects against
hash-flooding attacks). For a stable order, collect and sort the keys, or use a `BTreeMap`.

</details>

6. Why can't `f64` be a key of a `HashMap`?

<details>
<summary>Answer</summary>

Floats do not implement `Eq` and `Hash` because of `NaN` (which is not equal to itself). Use an
integer (for example, cents) or a wrapper type instead.

</details>

7. What happens to a `String` after you `insert` it as a key?

<details>
<summary>Answer</summary>

It is moved into the map, and you cannot use the variable any more. Insert a clone if you still
need the text.

</details>

8. When would you choose `BTreeMap` instead of `HashMap`?

<details>
<summary>Answer</summary>

When you need the entries in sorted key order, the smallest or largest key, or a range of keys.

</details>

## Summary

- A `HashMap<K, V>` stores key → value pairs and finds a value by its key very quickly.
- Import it with `use std::collections::HashMap;`. Use `insert`, `get`, `get_mut`, `remove`,
  `contains_key` and `len`.
- `get` returns an `Option`; `map[key]` panics when the key is missing.
- The `entry` API (`or_insert`, `or_default`) inserts a missing key and updates the value in one
  lookup. It is the tool for counting and grouping.
- Owned keys and values are moved into the map; copyable ones are copied.
- The iteration order of a `HashMap` is random. Sort the keys, or use a `BTreeMap`, when order matters.
- A key needs `Hash` and `Eq`: integers, `String`, `&str`, `char`, `bool` and tuples work; floats do
  not. Use `derive` for your own types.
- `HashSet` stores unique values. In tests, compare single entries or whole maps, not printed text.

## Sources and further reading

- [The Rust Programming Language — 8.3 Storing Keys with Associated Values in Hash Maps](https://doc.rust-lang.org/book/ch08-03-hash-maps.html) — the official book chapter on hash maps.
- [std::collections::HashMap](https://doc.rust-lang.org/std/collections/struct.HashMap.html) — all methods, including the entry API, with examples.
- [std::collections](https://doc.rust-lang.org/std/collections/index.html) — an overview of all standard collections and a guide for choosing one.
- [std::collections::BTreeMap](https://doc.rust-lang.org/std/collections/struct.BTreeMap.html) — the sorted map.
- [std::collections::HashSet](https://doc.rust-lang.org/std/collections/struct.HashSet.html) — the set type.
- [Rust by Example — HashMap](https://doc.rust-lang.org/rust-by-example/std/hash.html) — short runnable examples.

---

[🏠 Start](../START-HERE.md) · [← 04.8 Strings in depth](08-strings-in-depth.md) · [📋 Phase 04](README.md) · [04.10 Error handling →](10-error-handling.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
