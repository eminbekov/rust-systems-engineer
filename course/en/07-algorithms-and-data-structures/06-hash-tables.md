<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 07.5 Stacks and queues](05-stacks-and-queues.md) · [📋 Phase 07](README.md) · [07.7 Recursion and the call stack →](07-recursion.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/07-algorithms-and-data-structures/06-hash-tables.md) <!-- nav -->

# Lesson 07.6 — How hash tables work

> **Phase 07 — Algorithms and data structures** · Lesson 6 of 20 · ⏱ about 2.5 hours (2 sittings)

## What you will learn

- What the hash function does, what `Hash` and `Hasher` are in Rust, and the contract between `Hash` and `Eq`.
- What a collision is, and the two ways to deal with it: chaining and open addressing.
- Why a hash table is `O(1)` on average and `O(n)` in the worst case, and what the load factor and rehashing have to do with it.
- How to build a hash map with chaining, see a bad hash function ruin it, and compare it with `std::collections::HashMap`.

## Before you start

- You finished [Lesson 07.5 — Stacks and queues](05-stacks-and-queues.md).
- You read section 7 of [Lesson 04.9 — Hash maps](../04-structs-enums-collections-errors/09-hash-maps.md) ("How a hash map works inside"): key, hash function, bucket, collision. This lesson goes deeper and does not repeat the basics.
- You know traits and generics with bounds ([Lesson 05.1](../05-rust-intermediate/01-generics.md), [Lesson 05.2](../05-rust-intermediate/02-traits.md)).

## New words

| Word | Meaning |
|---|---|
| **hash function** | A function that turns a key of any size into a fixed-size number (the *hash*). The same key always gives the same hash. |
| **`Hash` / `Hasher`** | The Rust trait for "can be fed to a hash function" / the object that receives the bytes and produces the number (`finish()`). |
| **bucket** | One slot of the table that holds the entries whose hash points to it. |
| **chaining** | Collision strategy: each bucket holds a small list (chain) of all entries that landed in it. |
| **open addressing** | Collision strategy: all entries live in the table itself; on a collision you look at another slot (for example, the next one). |
| **probing** | Looking at the next candidate slot in open addressing. *Linear probing* tries the slots one after another. |
| **load factor** | Entries divided by buckets. A table with 6 entries and 8 buckets has load factor 0.75. |
| **rehashing** | Building a bigger table and putting every entry into its new bucket (the hash is the same, but the remainder changes). |
| **avalanche** | The property of a good hash function: changing one letter of the key changes the whole hash, so similar keys end up far apart. |
| **tombstone** | A marker left in a slot of an open-addressing table after a removal, so that lookups do not stop too early. |
| **hash flooding** | An attack that chooses many keys with the same hash to make a table slow (Lesson 04.9). |

## 1. Recap, and a contract

From Lesson 04.9: a hash map turns a key into a number with a hash function, uses the number to pick a **bucket**, and
looks only in that bucket. Two keys in the same bucket are a **collision**. This lesson answers the questions that
Lesson 04.9 left open: *what makes a good hash function, what exactly happens in a collision, and when does a hash table become slow?*

In Rust the hash function is split into two traits:

- `Hash` is implemented by the **key type**. Its method `hash(&self, state: &mut H)` feeds the parts of the key into a
  `Hasher`. You almost always get it with `#[derive(Hash)]`.
- `Hasher` is the machine that mixes the bytes it receives. After the key has been fed in, `finish()` returns the `u64` hash.

`examples/e06_hashers.rs` shows the behaviour:

```rust
use std::collections::hash_map::{DefaultHasher, RandomState};
use std::hash::{BuildHasher, Hash, Hasher};

fn main() {
    // DefaultHasher::new() always starts the same way
    let mut a = DefaultHasher::new();
    "hello".hash(&mut a);
    let mut b = DefaultHasher::new();
    "hello".hash(&mut b);
    println!("same input, same hasher : {} {}", a.finish(), b.finish());

    let mut c = DefaultHasher::new();
    "hellp".hash(&mut c);
    println!("one letter changed       : {}", c.finish());

    // RandomState is what HashMap uses: every new state has its own secret keys
    let first = RandomState::new();
    let second = RandomState::new();
    println!("RandomState #1 for hello : {}", first.hash_one("hello"));
    println!("RandomState #1 again     : {}", first.hash_one("hello"));
    println!("RandomState #2 for hello : {}", second.hash_one("hello"));
}
```

```text
same input, same hasher : 16156531084128653017 16156531084128653017
one letter changed       : 15091304090336206315
RandomState #1 for hello : 13061328524403966515
RandomState #1 again     : 13061328524403966515
RandomState #2 for hello : 11387501393696546303
```

(The exact numbers can differ on your computer and between Rust versions; what matters is which numbers are equal.)

**What happens:**

1. Feeding the same text to two fresh `DefaultHasher::new()` gives **the same hash**. A hash function must be a pure function of the key.
2. Changing one letter gives a **completely different** number. A good hash function spreads similar keys all over the range. That is called *avalanche*.
3. `RandomState` is what `HashMap` uses by default. Each new `RandomState` has its own secret keys, so two maps hash `"hello"` differently, while one map always hashes it the same way. An attacker
   who cannot see the secret cannot plan many keys with the same hash: this is the protection against hash flooding from Lesson 04.9. The price is a little speed.

**The contract.** If `a == b`, then `hash(a)` must equal `hash(b)`. (The opposite is not required: different keys may have the same hash; that is a collision.) `derive(Hash, PartialEq, Eq)` together keeps the contract.
Break it, and lookups fail, as section 6 shows.

## 2. Collisions: two strategies

A hash is a 64-bit number, but the table has only a few buckets, so the index is `hash % bucket_count`. With 8 buckets and 100 keys there are many collisions. Whatever the strategy, a hash table must handle them.

**Chaining** (used by our implementation): each bucket holds a list of entries.

```text
 bucket 0: ─▶ ("bob", 7)
 bucket 1: (empty)
 bucket 2: ─▶ ("alice", 1) ─▶ ("eve", 5)       <- collision: two keys, one bucket
 bucket 3: ─▶ ("dan", 9)
```

Lookup of `"eve"`: hash, take the remainder, go to bucket 2, compare `"eve"` with each entry in the chain until one is equal. The cost is the chain length.

**Open addressing**: there are no lists. Every entry sits in a slot of the table itself. If the slot is taken, try the next one (**linear probing**):

```text
 slot:    0       1        2        3       4       5
        [bob ] [      ] [alice ] [eve  ] [dan  ] [      ]
 "eve" hashes to slot 2, which holds alice  -> probe slot 3: free -> store it there.
 Lookup "eve": start at slot 2 (alice, not equal) -> slot 3 (eve, equal) -> found.
 Lookup of a missing key stops at the first empty slot.
```

Open addressing keeps everything in one array, which is friendly to the processor cache (Lesson 07.4). It needs more care: removing an entry must leave a *tombstone* marker, otherwise a later lookup would stop too early at the hole. The standard
`HashMap` uses open addressing, in a refined design called a *Swiss table* (the `hashbrown` library inside `std`), which checks many slots at once with special processor instructions. Chaining is simpler to write correctly, so we build that.

## 3. Load factor and rehashing

How long are the chains? On average, the **load factor**: entries divided by buckets. With 1,000 entries in 2,048 buckets it is about 0.49, so most chains have 0 or 1 entries, and a lookup does a constant amount of work. If the number of buckets never grew, the load factor would
grow with `n`, the chains would grow, and a lookup would become `O(n)`.

So the table **grows** when the load factor passes a limit (we use 0.75): it allocates twice as many buckets and **rehashes**: every entry moves to the bucket given by the same hash with the new remainder. Rehashing costs `O(n)`, but it happens after doubling, so the cost is spread over many inserts, as with
the vector in Lesson 07.3: insert is **amortized `O(1)`**. This is the reason a map should be created with `HashMap::with_capacity(n)` when you know how many items you will add.

## 4. A hash map with chaining

Create `src/hash_table.rs` and add `pub mod hash_table;` to `src/lib.rs`. The type, the constructor, and the heart of the structure: turning a key into a bucket index.

```rust
//! Lesson 07.6: a hash table with separate chaining.

use std::collections::hash_map::DefaultHasher;
use std::hash::{Hash, Hasher};

/// A hash map. `buckets[i]` is the chain (a small vector) of all entries whose
/// key hashes to bucket `i`.
pub struct ChainedMap<K, V> {
    buckets: Vec<Vec<(K, V)>>,
    len: usize,
}

impl<K: Hash + Eq, V> ChainedMap<K, V> {
    pub fn new() -> ChainedMap<K, V> {
        ChainedMap {
            buckets: Self::empty_buckets(4),
            len: 0,
        }
    }

    fn empty_buckets(count: usize) -> Vec<Vec<(K, V)>> {
        (0..count).map(|_| Vec::new()).collect()
    }

    /// The bucket for a key: hash the key to a number, then take the remainder.
    fn bucket_index(&self, key: &K) -> usize {
        let mut hasher = DefaultHasher::new();
        key.hash(&mut hasher);
        (hasher.finish() % self.buckets.len() as u64) as usize
    }
```

**What happens:**

1. `buckets` is a vector of chains. Each chain is a `Vec<(K, V)>`. `new` starts with 4 empty buckets.
2. `bucket_index` creates a hasher, feeds the key into it with `key.hash(&mut hasher)`, takes the `u64` from `finish()`, and reduces it to a bucket number with `%`.
   `DefaultHasher::new()` is deterministic, which makes our tests repeatable. (A production table would use a random key like `RandomState`.)
3. The bounds `K: Hash + Eq` say what a key must be able to do.

The accessors, including two for studying the table:

```rust
    pub fn len(&self) -> usize {
        self.len
    }

    pub fn is_empty(&self) -> bool {
        self.len == 0
    }

    pub fn bucket_count(&self) -> usize {
        self.buckets.len()
    }

    /// Entries per bucket, on average.
    pub fn load_factor(&self) -> f64 {
        self.len as f64 / self.buckets.len() as f64
    }

    /// The length of the longest chain: the worst case of one lookup.
    pub fn longest_chain(&self) -> usize {
        self.buckets
            .iter()
            .map(|chain| chain.len())
            .max()
            .unwrap_or(0)
    }
```

The three operations:

```rust
    /// Adds or replaces an entry; returns the old value if the key was there.
    pub fn insert(&mut self, key: K, value: V) -> Option<V> {
        let index = self.bucket_index(&key);
        for entry in &mut self.buckets[index] {
            if entry.0 == key {
                return Some(std::mem::replace(&mut entry.1, value));
            }
        }
        // a new key: keep the load factor at 0.75 or less
        if (self.len + 1) * 4 > self.buckets.len() * 3 {
            self.grow();
        }
        let index = self.bucket_index(&key);
        self.buckets[index].push((key, value));
        self.len += 1;
        None
    }

    pub fn get(&self, key: &K) -> Option<&V> {
        let index = self.bucket_index(key);
        self.buckets[index]
            .iter()
            .find(|entry| entry.0 == *key)
            .map(|entry| &entry.1)
    }

    pub fn remove(&mut self, key: &K) -> Option<V> {
        let index = self.bucket_index(key);
        let position = self.buckets[index]
            .iter()
            .position(|entry| entry.0 == *key)?;
        self.len -= 1;
        Some(self.buckets[index].swap_remove(position).1)
    }
```

**What happens:**

1. `insert` first looks into the chain of the key's bucket. If the key is already there, `std::mem::replace` swaps in the new value and returns the old one (Lesson 03.5). If the key is new and the table would pass the load factor 0.75
   (`(len + 1) * 4 > buckets * 3`, integer arithmetic without floats), it grows first. Then it computes the bucket again (the count changed) and pushes the entry.
2. `get` goes to one bucket and scans its chain with `find`. `remove` finds the position and uses `swap_remove`, which is `O(1)` and allowed here because the order inside a chain does not matter.
3. With short chains all three are `O(1)`. With a chain of length `L` they cost `O(L)`.

The growing step and the `Default` implementation:

```rust
    /// Doubles the number of buckets and puts every entry in its new bucket (rehashing): O(n).
    fn grow(&mut self) {
        let new_count = self.buckets.len() * 2;
        let old = std::mem::replace(&mut self.buckets, Self::empty_buckets(new_count));
        for (key, value) in old.into_iter().flatten() {
            let index = self.bucket_index(&key);
            self.buckets[index].push((key, value));
        }
    }
}

impl<K: Hash + Eq, V> Default for ChainedMap<K, V> {
    fn default() -> ChainedMap<K, V> {
        ChainedMap::new()
    }
}
```

**What happens:** `grow` swaps in a new empty set of buckets twice as big (`mem::replace` returns the old ones), then walks over every old entry (`into_iter().flatten()` goes through all chains in turn, taking ownership) and puts it into its new bucket. No key is cloned: entries are moved.

### A bad hash function

What if all keys hash to the same number? `examples/e06_chains.rs` compares the plain `u32` key with a `BadKey` whose `Hash` implementation feeds nothing to the hasher:

```rust
use p07_algorithms::hash_table::ChainedMap;
use std::hash::{Hash, Hasher};

/// A key whose hash is always the same.
#[derive(PartialEq, Eq)]
struct BadKey(u32);

impl Hash for BadKey {
    fn hash<H: Hasher>(&self, _state: &mut H) {}
}

fn main() {
    let mut good = ChainedMap::new();
    let mut bad = ChainedMap::new();
    for i in 0..1000u32 {
        good.insert(i, ());
        bad.insert(BadKey(i), ());
    }
    println!(
        "good hash: {} entries, {} buckets, load factor {:.2}, longest chain {}",
        good.len(),
        good.bucket_count(),
        good.load_factor(),
        good.longest_chain()
    );
    println!(
        "bad hash : {} entries, {} buckets, load factor {:.2}, longest chain {}",
        bad.len(),
        bad.bucket_count(),
        bad.load_factor(),
        bad.longest_chain()
    );
}
```

```text
good hash: 1000 entries, 2048 buckets, load factor 0.49, longest chain 4
bad hash : 1000 entries, 2048 buckets, load factor 0.49, longest chain 1000
```

With a good hash the 1,000 entries are spread over 2,048 buckets, and the **longest chain has 4 entries**. With the bad hash everything is in **one chain of 1,000 entries**: the table is a slow linked list, and each lookup is `O(n)`. The map is still correct. It is only slow.
This is exactly what a hash-flooding attacker wants.

### Tests

The tests compare with `std::collections::HashMap` on random operations, check the load factor while the table grows, and check the bad-hash chain. Add at the bottom of `src/hash_table.rs`:

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use crate::testutil::Rng;
    use std::collections::HashMap;

    /// A key whose hash is always the same: every key lands in one bucket.
    #[derive(PartialEq, Eq)]
    struct BadKey(u32);

    impl Hash for BadKey {
        fn hash<H: Hasher>(&self, _state: &mut H) {}
    }

    #[test]
    fn behaves_like_hash_map_on_random_operations() {
        let mut rng = Rng::new(17);
        let mut mine: ChainedMap<u64, u64> = ChainedMap::new();
        let mut reference: HashMap<u64, u64> = HashMap::new();
        for step in 0..20_000 {
            let key = rng.next_u64() % 500;
            match rng.below(3) {
                0 => {
                    let value = rng.next_u64();
                    assert_eq!(
                        mine.insert(key, value),
                        reference.insert(key, value),
                        "step {step}"
                    );
                }
                1 => assert_eq!(mine.remove(&key), reference.remove(&key), "step {step}"),
                _ => assert_eq!(mine.get(&key), reference.get(&key), "step {step}"),
            }
            assert_eq!(mine.len(), reference.len(), "step {step}");
        }
    }

    #[test]
    fn table_grows_and_keeps_the_load_factor_low() {
        let mut map = ChainedMap::new();
        for i in 0..1000u32 {
            map.insert(i, i);
            assert!(map.load_factor() <= 0.75);
        }
        assert_eq!(map.bucket_count(), 2048);
        assert!((0..1000).all(|i| map.get(&i) == Some(&i)));
    }

    #[test]
    fn a_bad_hash_puts_everything_in_one_chain() {
        let mut map = ChainedMap::new();
        for i in 0..100 {
            map.insert(BadKey(i), i);
        }
        assert_eq!(map.longest_chain(), 100);
        assert_eq!(map.get(&BadKey(57)), Some(&57)); // still correct, only slow
    }
}
```

```text
running 3 tests
test hash_table::tests::a_bad_hash_puts_everything_in_one_chain ... ok
test hash_table::tests::table_grows_and_keeps_the_load_factor_low ... ok
test hash_table::tests::behaves_like_hash_map_on_random_operations ... ok

test result: ok. 3 passed; 0 failed; 0 ignored; 0 measured; 17 filtered out; finished in 0.01s
```

## 5. Measuring

`benches/hash_tables.rs` looks up 2,000 existing keys in three maps: ours, ours with the bad key, and `std::collections::HashMap`. Register it in `Cargo.toml` (like in Lesson 07.2):

```toml
[[bench]]
name = "hash_tables"
harness = false
```

```rust
use criterion::{Criterion, criterion_group, criterion_main};
use p07_algorithms::hash_table::ChainedMap;
use std::collections::HashMap;
use std::hash::{Hash, Hasher};
use std::hint::black_box;

const N: u64 = 2_000;

/// A key with a terrible hash function: every key gets the same hash.
#[derive(PartialEq, Eq)]
struct BadKey(u64);

impl Hash for BadKey {
    fn hash<H: Hasher>(&self, _state: &mut H) {}
}

fn bench_lookups(c: &mut Criterion) {
    let mut good = ChainedMap::new();
    let mut bad = ChainedMap::new();
    let mut std_map = HashMap::new();
    for i in 0..N {
        good.insert(i, i);
        bad.insert(BadKey(i), i);
        std_map.insert(i, i);
    }

    let mut group = c.benchmark_group("2_000 lookups");
    group.bench_function("our ChainedMap", |b| {
        b.iter(|| {
            (0..N)
                .map(|i| *good.get(&black_box(i)).unwrap())
                .sum::<u64>()
        })
    });
    group.bench_function("our ChainedMap, bad hash", |b| {
        b.iter(|| {
            (0..N)
                .map(|i| *bad.get(&BadKey(black_box(i))).unwrap())
                .sum::<u64>()
        })
    });
    group.bench_function("std HashMap", |b| {
        b.iter(|| {
            (0..N)
                .map(|i| *std_map.get(&black_box(i)).unwrap())
                .sum::<u64>()
        })
    });
    group.finish();
}

criterion_group!(benches, bench_lookups);
criterion_main!(benches);
```

Run `cargo bench --bench hash_tables -- --warm-up-time 1 --measurement-time 3`:

```text
2_000 lookups/our ChainedMap
                        time:   [15.119 µs 15.221 µs 15.347 µs]
2_000 lookups/our ChainedMap, bad hash
                        time:   [309.49 µs 314.92 µs 320.55 µs]
2_000 lookups/std HashMap
                        time:   [11.896 µs 11.940 µs 11.993 µs]
```

(Apple M4 Pro; **your numbers will differ**.)

**How to read it:**

1. Our map does 2,000 lookups in 15 µs: about **7.6 ns per lookup**. The standard map takes 12 µs, about 6 ns. Our simple chaining is about 25% slower than the very tuned standard table.
2. With the bad hash the same 2,000 lookups take 315 µs, **21 times slower**, and the gap grows with `n`: each lookup scans a chain of up to 2,000 entries (it was `O(1)`, now it is `O(n)`).
3. The hash function is half of a hash table. A table with a bad hash is a list in disguise.

## 6. The contract, broken

Here is the bug that the contract warns about. `Name` compares without regard to case, but derives `Hash` from the text as it is:

```rust
use p07_algorithms::hash_table::ChainedMap;

/// A name that ignores upper and lower case when compared...
#[derive(Hash)]
struct Name(String);

impl PartialEq for Name {
    fn eq(&self, other: &Name) -> bool {
        self.0.eq_ignore_ascii_case(&other.0)
    }
}

impl Eq for Name {}

fn main() {
    let mut map = ChainedMap::new();
    for word in [
        "Rust", "Cargo", "Crate", "Trait", "Slice", "Macro", "Enum", "Box",
    ] {
        map.insert(Name(word.to_string()), word.len());
    }
    let rust_upper = Name("Rust".to_string());
    let rust_lower = Name("rust".to_string());
    println!("Rust == rust : {}", rust_upper == rust_lower);
    println!("get(Rust)    : {:?}", map.get(&rust_upper));
    println!("get(rust)    : {:?}", map.get(&rust_lower));
}
```

```text
Rust == rust : true
get(Rust)    : Some(4)
get(rust)    : None
```

`Rust == rust` is `true`, yet `get(rust)` finds nothing: `"Rust"` and `"rust"` hash to different numbers, so the lookup goes to another bucket. The map did not crash. It quietly gave a wrong answer. Fix: make `Hash` agree with `Eq`,
for example hash the lowercase text: `self.0.to_ascii_lowercase().hash(state)` inside a hand-written `impl Hash`.

## 7. When to use which standard type

| You need | Use | Cost |
|---|---|---|
| look up by key, order does not matter | `HashMap<K, V>` | `O(1)` average |
| "have I seen this value?" | `HashSet<T>` | `O(1)` average |
| sorted keys, ranges (`range(a..b)`), first and last key | `BTreeMap<K, V>` / `BTreeSet<T>` | `O(log n)`; Lesson 07.11 introduces search trees |
| a small number of keys (under about 10 to 20) | a `Vec<(K, V)>` and a linear scan | `O(n)`, but often the fastest |
| keys from untrusted users | `HashMap` with the default `RandomState` | resists hash flooding |

Two reminders about the real `HashMap`: its `get` accepts any borrowed form of the key (you can look up a `String` key with a `&str`, thanks to the `Borrow` trait), which our simplified `get(&K)` does not. And the `entry` API (Application Programming Interface: the set of functions and types that a library offers) of Lesson 04.9 does
"look up, then insert" with one hash computation.

## Common mistakes

**1. Breaking the `Hash`/`Eq` contract** (section 6). Derive both together, or write both by hand from the same fields.

**2. Hashing a mutable field.** A key's hash must not change while the key is in the map. A key with interior mutability (`RefCell`, Lesson 06.4) that changes its hashed field makes the entry unreachable. Use plain, immutable keys.

**3. Expecting an order.** The iteration order of a `HashMap` is random and changes between runs. If you need an order, use `BTreeMap`, or sort the keys.

## Try it yourself

1. Type `hash_table.rs` yourself. Run `cargo test hash_table`, `cargo clippy` and `cargo fmt`.
2. Change the growth limit from 0.75 to 3.0 (`(self.len + 1) > self.buckets.len() * 3`). Run `examples/e06_chains.rs`: what are the load factor and the longest chain now? Run the benchmark: does the map get slower? Change it back.
3. Run `examples/e06_hashers.rs` twice. Which lines change between runs, and which do not? Why?
4. Commit: `git add code/p07-algorithms && git commit -m "feat(p07): hash map with chaining"`.

## Exercises

### Exercise 1 — Load factor (easy)

A table has 16 buckets and grows when the load factor would pass 0.75. (a) How many entries can it hold before the first growth? (b) After it grows, how many buckets does it have, and what is the load factor right after the growth? (c) What is the time of one `insert`, in the worst case and amortized, and why?

<details>
<summary>Hint</summary>

The limit is `buckets * 0.75`. Rehashing moves every entry, but the number of entries does not change.

</details>

<details>
<summary>Answer</summary>

(a) `16 * 0.75 = 12` entries; the 13th insert triggers the growth. (b) 32 buckets; the 13 entries give a load factor of `13 / 32 ≈ 0.41`. (c) Worst case `O(n)` for the one insert that rehashes (and with a terrible hash, also `O(n)` for a long chain); amortized `O(1)`, because
the table doubles and each entry is rehashed only a few times over many inserts.

</details>

### Exercise 2 — Counting words (medium)

Add `get_mut(&mut self, key: &K) -> Option<&mut V>` to `ChainedMap`, and use it in `count_words(text: &str) -> ChainedMap<String, usize>` that counts how often each word appears. Test it against a `HashMap` built with the `entry` API. State the time of `count_words` for a text of `n` words.

<details>
<summary>Hint</summary>

`get_mut` is `get` with `iter_mut()` instead of `iter()`. In `count_words`, `match counts.get_mut(&word)`: increase the count in the `Some` arm, insert 1 in the `None` arm.

</details>

<details>
<summary>Answer</summary>

Inside an `impl<K: Hash + Eq, V> ChainedMap<K, V>` block:

```rust
    /// A mutable reference to the value of `key`, so it can be changed in place.
    pub fn get_mut(&mut self, key: &K) -> Option<&mut V> {
        let index = self.bucket_index(key);
        self.buckets[index]
            .iter_mut()
            .find(|entry| entry.0 == *key)
            .map(|entry| &mut entry.1)
    }
```

```rust
/// Counts how often each word appears.
pub fn count_words(text: &str) -> ChainedMap<String, usize> {
    let mut counts = ChainedMap::new();
    for word in text.split_whitespace() {
        match counts.get_mut(&word.to_string()) {
            Some(count) => *count += 1,
            None => {
                counts.insert(word.to_string(), 1);
            }
        }
    }
    counts
}
```

The test (inside `mod tests`):

```rust
    #[test]
    fn word_counts_match_hash_map() {
        let text = "the cat and the dog and the bird";
        let mine = count_words(text);
        let mut reference: HashMap<String, usize> = HashMap::new();
        for word in text.split_whitespace() {
            *reference.entry(word.to_string()).or_insert(0) += 1;
        }
        assert_eq!(mine.len(), reference.len());
        for (word, count) in &reference {
            assert_eq!(mine.get(word), Some(count));
        }
    }
```

Time `O(n)` on average for `n` words (one lookup, and at most one insert, per word, each `O(1)` on average), plus the growth, which is amortized. The extra space is `O(d)` for `d` different words.

</details>

### Exercise 3 — A set from a map (hard)

Build `ChainedSet<K>` on top of `ChainedMap<K, ()>` with `insert(key) -> bool` (`true` if the key was new), `contains`, `remove -> bool`, `len` and `is_empty`. Test it against `std::collections::HashSet` on 5,000 random operations. What are the costs?

<details>
<summary>Hint</summary>

`ChainedMap::insert` returns the old value (`Option<()>`): the key was new exactly when that is `None`. A set is a map whose values carry no information.

</details>

<details>
<summary>Answer</summary>

```rust
/// A set of keys, built on the map with empty values.
pub struct ChainedSet<K> {
    map: ChainedMap<K, ()>,
}

impl<K: Hash + Eq> ChainedSet<K> {
    pub fn new() -> ChainedSet<K> {
        ChainedSet {
            map: ChainedMap::new(),
        }
    }

    /// Returns `true` if the key was not in the set before.
    pub fn insert(&mut self, key: K) -> bool {
        self.map.insert(key, ()).is_none()
    }

    pub fn contains(&self, key: &K) -> bool {
        self.map.get(key).is_some()
    }

    pub fn remove(&mut self, key: &K) -> bool {
        self.map.remove(key).is_some()
    }

    pub fn len(&self) -> usize {
        self.map.len()
    }

    pub fn is_empty(&self) -> bool {
        self.map.is_empty()
    }
}

impl<K: Hash + Eq> Default for ChainedSet<K> {
    fn default() -> ChainedSet<K> {
        ChainedSet::new()
    }
}
```

```rust
    #[test]
    fn set_matches_hash_set() {
        use std::collections::HashSet;
        let mut rng = Rng::new(21);
        let mut mine: ChainedSet<u64> = ChainedSet::new();
        let mut reference: HashSet<u64> = HashSet::new();
        for step in 0..5000 {
            let key = rng.next_u64() % 100;
            match rng.below(3) {
                0 => assert_eq!(mine.insert(key), reference.insert(key), "step {step}"),
                1 => assert_eq!(mine.remove(&key), reference.remove(&key), "step {step}"),
                _ => assert_eq!(mine.contains(&key), reference.contains(&key), "step {step}"),
            }
            assert_eq!(mine.len(), reference.len());
        }
        assert_eq!(mine.is_empty(), reference.is_empty());
    }
```

Every operation delegates to the map: `O(1)` on average, amortized for `insert`. This is also how the standard `HashSet<T>` is built on top of `HashMap<T, ()>`.

</details>

## Quiz

1. What does the contract between `Hash` and `Eq` say?

<details>
<summary>Answer</summary>

If two keys are equal, their hashes must be equal. (Different keys may share a hash; that is a collision.)

</details>

2. Why is a hash map lookup `O(1)` on average but `O(n)` in the worst case?

<details>
<summary>Answer</summary>

With a good hash and a bounded load factor, chains are short (a constant length on average). If many keys land in one bucket (a bad hash, or an attacker), one chain holds nearly all entries and a lookup scans them all.

</details>

3. What is the load factor, and why does the table grow when it passes a limit?

<details>
<summary>Answer</summary>

Entries divided by buckets. It is the average chain length. Growing (and rehashing) keeps it below the limit, so chains stay short and operations stay `O(1)`.

</details>

4. Why can a `HashMap<String, V>` be searched with a `&str`, and why can't our `ChainedMap::get`?

<details>
<summary>Answer</summary>

The standard `get` is generic over a borrowed form `Q` of the key (`K: Borrow<Q>`), and `str` hashes and compares the same as `String`. Our `get` takes `&K` only, so it needs a `&String`.

</details>

5. When can `BTreeMap` be a better choice than `HashMap`?

<details>
<summary>Answer</summary>

When you need the keys in sorted order, range queries, or a guaranteed `O(log n)` worst case with no dependence on hashing.

</details>

## Summary

- A hash function maps a key to a number; the table maps the number to a bucket with `%`. Equal keys must have equal hashes (the `Hash`/`Eq` contract).
- Collisions are unavoidable. Chaining keeps a list per bucket; open addressing probes for another slot (the standard `HashMap` does this, as a Swiss table).
- The load factor is the average chain length. Keeping it below a limit by doubling the bucket count and rehashing makes `insert` amortized `O(1)` and `get` `O(1)` on average.
- A bad or attacker-chosen hash turns the table into a list: `O(n)` per operation.
- Test your table against `HashMap` on random operations, and check the structure itself (load factor, longest chain).
- In real code use `HashMap`/`HashSet`; use `BTreeMap` when you need order.

## Sources and further reading

- [std::collections::HashMap](https://doc.rust-lang.org/std/collections/struct.HashMap.html) — the standard hash map, with a short description of its design, the hashing and the `entry` API.
- [std::hash — Hash, Hasher and BuildHasher](https://doc.rust-lang.org/std/hash/index.html) — the traits behind hashing, and the contract between `Hash` and `Eq`.
- [hashbrown](https://github.com/rust-lang/hashbrown) — the Swiss-table implementation that `std::collections::HashMap` is built on.
- [std::borrow::Borrow](https://doc.rust-lang.org/std/borrow/trait.Borrow.html) — why a `String` key can be looked up with a `&str`.
- [std::collections::BTreeMap](https://doc.rust-lang.org/std/collections/struct.BTreeMap.html) — the ordered map.

---

[🏠 Start](../START-HERE.md) · [← 07.5 Stacks and queues](05-stacks-and-queues.md) · [📋 Phase 07](README.md) · [07.7 Recursion and the call stack →](07-recursion.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
