<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 17.5 Indexes: a hash index](05-hash-indexes.md) · [📋 Phase 17](README.md) · [17.7 LSM trees →](07-lsm-trees.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/17-storage-and-databases/06-b-trees.md) <!-- nav -->

# Lesson 17.6 — B-trees

> **Phase 17 — Storage and database internals** · Lesson 6 of 15 · ⏱ about 2.5 hours (2 sittings)

## What you will learn

- Why a binary search tree is a bad index on disk, and how a B-tree fixes it with wide nodes the size of a page.
- The rules of a B-tree, and why its height grows only very slowly (three levels can hold a million keys).
- How search works, and how insertion keeps the tree balanced by *splitting* full nodes.
- How to write an in-memory B-tree in Rust and test it against `std::collections::BTreeMap`.
- How real databases store B-tree nodes in pages on disk, what a B+ tree is, and why updating pages in place needs a WAL (write-ahead log).

## Before you start

- You finished [Lesson 17.5 — Indexes: a hash index](05-hash-indexes.md) and know its limits: keys in RAM (Random Access Memory) and no range
  queries.
- You know binary search trees and why their shape matters from
  [Lesson 07.11 — Trees and binary search trees](../07-algorithms-and-data-structures/11-trees-and-binary-search-trees.md),
  and `binary_search` on a sorted slice from [Lesson 07.10](../07-algorithms-and-data-structures/10-binary-search.md).
- You know pages and blocks from [Lesson 17.1](01-how-disks-and-files-work.md) and the WAL from
  [Lesson 17.4](04-write-ahead-log.md).

## New words

| Word | Meaning |
|---|---|
| **BST** (Binary Search Tree) | A tree where every node has at most two children: smaller keys to the left, bigger keys to the right (Lesson 07.11). |
| **B-tree** | A balanced search tree whose nodes hold many sorted keys and have many children. Designed so that one node fits into one disk page. |
| **fanout** | How many children a node has. A B-tree node can have hundreds; a binary tree node has two. |
| **minimum degree** (`t`) | The number that fixes the node size in our B-tree: every node except the root has between `t - 1` and `2t - 1` keys. |
| **split** | When a node is full, cut it into two half-full nodes and move its middle key up into the parent. |
| **internal node** | A node that has children (not a leaf). |
| **B+ tree** | The variant real databases use: all values are in the leaves, the internal nodes only guide the search, and the leaves are linked in key order. |
| **page** | A fixed-size block of a database file (often 4 KiB or 8 KiB; a KiB, kibibyte, is 1,024 bytes) that is read and written as a whole. Here one page holds one tree node. |
| **slotted page** | A page layout with a small array of offsets at the start that point to variable-size records inside the page. |

## 1. Why not a binary search tree on disk?

In Lesson 07.11 a balanced binary search tree (BST) found any of `n` keys in about `log₂ n` steps. For a million keys that
is 20 steps. In memory, each step is a few nanoseconds. On disk, each step is a jump to another node somewhere in the file:
a random read of tens of microseconds on an SSD (solid-state drive), and milliseconds on an HDD, a hard disk drive (Lesson 17.1). Twenty random reads for one
lookup is far too many.

Worse, each read fetches a whole 4 KiB block (Lesson 17.1), but a BST node with one key and two pointers uses perhaps
30 bytes of it. 99% of every read is wasted.

The fix is to make each node **as wide as a page**: put hundreds of sorted keys and hundreds of child pointers into one
node. Then one page read gives you a lot of information, and the tree needs only a few levels:

```text
 fanout (children per node)    levels for 1,000,000 keys    levels for 1,000,000,000 keys
            2  (a BST)                    20                            30
          100                              3                             5
          500                              3                             4
```

That is a **B-tree**. With 500 children per node, a lookup in a billion keys reads four pages, and the top one or two
levels are used so often that they stay in RAM anyway. (The levels are `log` with base equal to the fanout; Exercise 1
computes them.)

## 2. The rules

Our B-tree has a **minimum degree** `t` (at least 2). Then:

1. Every node holds its keys **sorted**. A node with `k` keys has either no children (a **leaf**) or exactly `k + 1`
   children (an **internal node**).
2. Child number `i` holds the keys that lie *between* key `i - 1` and key `i` of its parent:

```text
                 [ 20 | 50 ]
                /     |     \
      [ 5 | 10 ]  [ 30 | 40 ]  [ 60 | 70 | 80 ]
       keys < 20   20 < keys < 50   keys > 50
```

3. Every node except the root holds between `t - 1` and `2t - 1` keys. It is never more than full and never less than half
   full. (With `t = 2`, as in the picture, a node holds 1 to 3 keys.)
4. **All leaves are at the same depth.** This is what "balanced" means here, and it is why no input order, not even sorted
   input, can make the tree tall.

**Search** follows from rule 2. Start at the root. Binary-search the node's keys. If the key is there, done. Otherwise the
search tells you between which two keys it would be; go to that child. At a leaf without the key, the key does not exist.

**Insertion** always adds the key to a **leaf**, in sorted position. The problem is a full leaf (`2t - 1` keys). Then the
node is **split**: its middle key moves up into the parent, and the keys left and right of it become two nodes with
`t - 1` keys each. Here `t = 2` and we insert 40 into a full leaf:

```text
before:      [ 20 ]                   after:        [ 20 | 30 ]
            /      \                               /     |     \
      [ 10 ]    [ 25 | 30 | 35 ]             [ 10 ]  [ 25 ]  [ 35 | 40 ]
                     full!                                         40 goes here
```

What if the parent is full too? Our code avoids the question with a simple trick: on the way **down**, it splits every full
node it is about to enter. So when a split moves a key up, the parent always has room. If the **root** is full, a new
root is put above it first. That is the only way the tree gets taller: it grows at the top, never at the bottom, so all
leaves stay at the same depth.

## 3. A B-tree in Rust

Create `src/btree.rs` and add `pub mod btree;` to `src/lib.rs`. The types:

```rust
use std::cmp::Ordering;

/// A B-tree in memory. Every node except the root holds between t-1 and 2t-1 keys,
/// and all leaves are at the same depth.
pub struct BTree<K, V> {
    root: Node<K, V>,
    t: usize,
    len: usize,
}

struct Node<K, V> {
    keys: Vec<K>,              // sorted
    vals: Vec<V>,              // vals[i] belongs to keys[i]
    children: Vec<Node<K, V>>, // empty for a leaf, otherwise keys.len() + 1
}

impl<K, V> Node<K, V> {
    fn new() -> Self {
        Node {
            keys: Vec::new(),
            vals: Vec::new(),
            children: Vec::new(),
        }
    }

    fn is_leaf(&self) -> bool {
        self.children.is_empty()
    }
}
```

A node owns its children directly in a `Vec<Node>`; a child cannot outlive its parent, so no `Box` or `Rc` is needed. The
tree is generic over the key type `K` (which must be `Ord` to be sorted) and the value type `V` (Lesson 05.1).

Search and the small helpers:

```rust
impl<K: Ord, V> BTree<K, V> {
    /// Creates an empty tree with minimum degree `t` (a node holds at most 2t-1 keys).
    pub fn new(t: usize) -> Self {
        assert!(t >= 2, "minimum degree must be at least 2");
        BTree {
            root: Node::new(),
            t,
            len: 0,
        }
    }

    pub fn len(&self) -> usize {
        self.len
    }

    pub fn is_empty(&self) -> bool {
        self.len == 0
    }

    /// Number of levels: 1 for a tree that is only a root.
    pub fn height(&self) -> usize {
        let mut height = 1;
        let mut node = &self.root;
        while !node.is_leaf() {
            node = &node.children[0];
            height += 1;
        }
        height
    }

    pub fn get(&self, key: &K) -> Option<&V> {
        let mut node = &self.root;
        loop {
            match node.keys.binary_search(key) {
                Ok(i) => return Some(&node.vals[i]),
                Err(_) if node.is_leaf() => return None,
                Err(i) => node = &node.children[i],
            }
        }
    }
```

**What happens:**

1. `height` walks down the leftmost children. Rule 4 says all leaves are at the same depth, so any path gives the height.
2. `binary_search` returns `Ok(i)` when `keys[i]` is the key, and `Err(i)` with the position where the key *would* be. That
   position is exactly the number of the child to go down to (rule 2). This is the search from Section 2 in four lines.
3. `Err(_) if node.is_leaf()` is a match arm with a **guard** (Lesson 04.5): it matches only if the condition is true.

Insertion, the end of `impl BTree`:

```rust
    /// Inserts or overwrites. Returns the old value if the key was already there.
    pub fn insert(&mut self, key: K, value: V) -> Option<V> {
        if self.root.keys.len() == 2 * self.t - 1 {
            // The root is full: put a new, empty root above it and split the old root.
            let old_root = std::mem::replace(&mut self.root, Node::new());
            self.root.children.push(old_root);
            self.root.split_child(0, self.t);
        }
        let old = self.root.insert_non_full(key, value, self.t);
        if old.is_none() {
            self.len += 1;
        }
        old
    }
}
```

`std::mem::replace(&mut self.root, Node::new())` puts a new empty node in place of the root and gives back the old root.
We need it because Rust does not let you move a value out of a field and leave the field empty, even for a moment. The old
root becomes the only child of the new one, and splitting it gives the new root its first key.

The node methods that do the real work:

```rust
impl<K: Ord, V> Node<K, V> {
    /// Splits the full child `i` (2t-1 keys) into two nodes of t-1 keys
    /// and moves its middle key up into `self`.
    fn split_child(&mut self, i: usize, t: usize) {
        let child = &mut self.children[i];
        let right = Node {
            keys: child.keys.split_off(t),
            vals: child.vals.split_off(t),
            children: if child.is_leaf() {
                Vec::new()
            } else {
                child.children.split_off(t)
            },
        };
        let middle_key = child.keys.pop().expect("a full node has a middle key");
        let middle_val = child.vals.pop().expect("a full node has a middle value");
        self.keys.insert(i, middle_key);
        self.vals.insert(i, middle_val);
        self.children.insert(i + 1, right);
    }

    /// Inserts into a node that is known not to be full.
    fn insert_non_full(&mut self, key: K, value: V, t: usize) -> Option<V> {
        let mut i = match self.keys.binary_search(&key) {
            Ok(i) => return Some(std::mem::replace(&mut self.vals[i], value)),
            Err(i) => i,
        };
        if self.is_leaf() {
            self.keys.insert(i, key);
            self.vals.insert(i, value);
            return None;
        }
        if self.children[i].keys.len() == 2 * t - 1 {
            self.split_child(i, t);
            // The middle key of the child is now keys[i]: choose the side to go down.
            match key.cmp(&self.keys[i]) {
                Ordering::Equal => return Some(std::mem::replace(&mut self.vals[i], value)),
                Ordering::Greater => i += 1,
                Ordering::Less => {}
            }
        }
        self.children[i].insert_non_full(key, value, t)
    }
}
```

**What happens:**

1. In `split_child`, the child has `2t - 1` keys at positions `0 .. 2t-1`. `split_off(t)` cuts off positions `t ..` (that
   is `t - 1` keys) into the new right node. The child keeps `t` keys; `pop` removes the last of them, position `t - 1`, the
   **middle key**. The child is left with `t - 1` keys. An internal child has `2t` children, and `split_off(t)` gives `t` of
   them to each side.
2. The middle key and its value go into the parent at position `i`, and the right node becomes child `i + 1`: exactly the
   picture in Section 2.
3. `insert_non_full` first checks whether the key already exists in this node; then it overwrites the value and returns
   the old one, just like `HashMap::insert`.
4. In a leaf, `Vec::insert` puts the key at its sorted position. In an internal node, the code splits the child **before**
   going down if that child is full. After the split, the child's middle key sits at `keys[i]`, so the new key may belong
   to the right half (`i += 1`) or may even be that middle key.
5. The recursion goes down one level per call, so it is only as deep as the tree is tall.

> ⚡ **Performance:** `Vec::insert` in the middle of a node moves the keys after it, which is O(fanout) work. That sounds
> bad, but a node is small and lies in a few cache lines (Lesson 12.9); moving a few hundred bytes is cheaper than
> following one more pointer. This is also why the standard library's `BTreeMap` is a B-tree.

## 4. Testing against `BTreeMap`

The best test for your own data structure is a trusted one that does the same job: `std::collections::BTreeMap`. Feed both
the same operations and compare every answer. Add at the end of `src/btree.rs`:

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use std::collections::BTreeMap;

    #[test]
    fn small_tree_by_hand() {
        let mut tree = BTree::new(2); // at most 3 keys per node
        for k in [10, 20, 30, 40, 50] {
            tree.insert(k, k * 100);
        }
        assert_eq!(tree.height(), 2); // the root was split once
        assert_eq!(tree.get(&40), Some(&4000));
        assert_eq!(tree.get(&35), None);
        assert_eq!(tree.insert(40, 1), Some(4000)); // overwrite
        assert_eq!(tree.len(), 5);
    }

    #[test]
    fn behaves_like_std_btreemap() {
        for t in [2, 3, 8] {
            let mut ours = BTree::new(t);
            let mut std_map = BTreeMap::new();
            let mut x: u64 = 7;
            for _ in 0..10_000 {
                x ^= x << 13;
                x ^= x >> 7;
                x ^= x << 17;
                let key = x % 5_000; // many repeated keys: overwrites
                assert_eq!(ours.insert(key, x), std_map.insert(key, x));
            }
            assert_eq!(ours.len(), std_map.len());
            for key in 0..5_000 {
                assert_eq!(ours.get(&key), std_map.get(&key), "t = {t}, key = {key}");
            }
        }
    }
}
```

**What happens:**

1. The first test is small enough to follow on paper: 10, 20, 30 fill the root; inserting 40 splits it into `[20]` with
   children `[10]` and `[30]`; 40 and 50 join `[30]`. Height 2.
2. The second test uses the xorshift generator from Lesson 17.1 to make 10,000 keys in `0..5000`, so many keys repeat and
   overwrites are tested too. Small `t` values like 2 and 3 cause many splits, which is where bugs hide.
3. `insert` must return the same `Option` as `BTreeMap::insert`, and every `get` must match.

`cargo test --lib btree`:

```text
running 2 tests
test btree::tests::small_tree_by_hand ... ok
test btree::tests::behaves_like_std_btreemap ... ok

test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 21 filtered out; finished in 0.03s
```

> 🧪 **Testing:** comparing with a simple, trusted implementation (a *model*) is one of the strongest tests for data
> structures. Property-based testing from Lesson 09.4 automates it: let the tool generate the operations.

## 5. How tall does it get?

In Lesson 07.11 a plain BST built from **sorted** keys had a height equal to the number of keys. Let us give our B-tree the
same worst case. Put this into `src/bin/btree_heights.rs`:

```rust
use p17_storage::btree::BTree;

const N: u64 = 1_000_000;

fn main() {
    println!("{:>4} {:>14} {:>7}", "t", "max keys/node", "height");
    for t in [2, 16, 128] {
        let mut tree = BTree::new(t);
        for key in 0..N {
            tree.insert(key, key); // sorted input: the worst case for a plain BST
        }
        assert_eq!(tree.len(), N as usize);
        assert_eq!(tree.get(&777_777), Some(&777_777));
        println!("{:>4} {:>14} {:>7}", t, 2 * t - 1, tree.height());
    }
}
```

`cargo run --release --bin btree_heights`:

```text
   t  max keys/node  height
   2              3      19
  16             31       5
 128            255       3
```

A million sorted keys, and the height is 19 even with tiny nodes; with 255 keys per node, it is 3. Stored on disk with one
node per page, a lookup would read 3 pages. The BST from Lesson 07.11 would need up to a million steps for the same input.

## 6. B-trees on disk

The tree above lives in memory. A database stores it in a file, and three ideas make that work:

**One node is one page.** The file is an array of pages of a fixed size (4 KiB in SQLite by default, 8 KiB in
PostgreSQL). Page number `n` starts at offset `n × page_size`. A child "pointer" is just a page number. Reading a node is
one positioned read of one page (Lesson 17.1), and the cache of pages in RAM, the *buffer pool*, is Lesson 17.9.

**Slotted pages.** Keys and values have different sizes, so a page cannot be a simple array. A **slotted page** has a small
header with an array of offsets ("slots") at the front, and the records grow from the end of the page towards the middle:

```text
 ┌────────┬────────┬────────┬───────────── free ─────────────┬──────────┬──────────┐
 │ header │ slot 0 │ slot 1 │                                 │ record 1 │ record 0 │
 │ count=2│ → 4000 │ → 3950 │                                 │ (50 B)   │ (96 B)   │
 └────────┴────────┴────────┴─────────────────────────────────┴──────────┴──────────┘
 0                                                                                4095
```

To keep keys sorted, only the small slots are moved; records stay where they are. Exercise 3 encodes a simpler page
with fixed-size entries.

**B+ trees.** Real databases use a variant: the **B+ tree**. Internal nodes hold only keys and page numbers (no values), so
more of them fit into a page and the tree is even flatter. All values live in the **leaves**, and each leaf points to the
next leaf. A range query ("all keys from `b` to `d`") finds the first leaf and then simply follows the chain to the right,
reading pages in order. PostgreSQL's default index type (`CREATE INDEX` from Lesson 16.5) is such a tree.

**Updates in place need the WAL.** Unlike the logs of Lessons 17.4 and 17.5, a B-tree **changes pages in place**: an
insert rewrites a leaf, and a split rewrites three pages. A crash in the middle can leave a torn page (Lesson 17.2) or a
split that is only half done, and the tree would be broken. So B-tree databases write every page change to the WAL first,
and recovery replays it. That is why every B-tree database you will meet has a WAL.

Deleting from a B-tree is also possible: it is the mirror image of insertion, merging or rebalancing nodes that become less
than half full. It has more cases than insertion and is left out of this lesson; many real systems simply let pages stay
partly empty and clean them up later.

## Common mistakes

1. **Splitting at the wrong index.** With `2t - 1` keys, the middle one is at index `t - 1`. An off-by-one gives nodes with
   `t` and `t - 2` keys, which breaks rule 3 later. The comparison test with `BTreeMap` catches it quickly with `t = 2`.
2. **Forgetting to move the children in a split.** An internal node's children must be split together with its keys
   (`children.split_off(t)`), or the tree points to the wrong subtrees.
3. **Going down the wrong side after a split.** After `split_child(i)`, the key may now belong to child `i + 1`. Compare
   with the new `keys[i]` before you go down.
4. **Testing only with random keys.** Sorted and reversed input split nodes in different patterns; test them too.

## Try it yourself

1. Draw the tree after inserting 1, 2, …, 10 with `t = 2`. Then check your drawing: add a temporary `println!` or a debug
   method that prints the keys of each level.
2. In `btree_heights.rs`, add `t = 1024`. How many keys does a node hold, and what is the height now?
3. Insert the keys in reverse order (`(0..N).rev()`) and compare the heights.

## Exercises

### Exercise 1 — How many levels? (easy)

Create `src/page.rs` (and add `pub mod page;` to `src/lib.rs`) with `pub fn levels_needed(items: u64, fanout: u64) -> u32`:
the smallest number of levels `h` such that `fanout^h >= items`. Write a test: a million items with fanout 100 need 3
levels; a billion with fanout 200 need 4; 100 items with fanout 100 need 1, but 101 need 2.

<details>
<summary>Hint</summary>

Start with `levels = 1` and `capacity = fanout`; while `capacity < items`, multiply `capacity` by `fanout` and add a level.
Use `saturating_mul` so that big numbers do not overflow (Lesson 12.2).

</details>

<details>
<summary>Answer</summary>

```rust
/// Exercise 1: the smallest number of levels such that `fanout`^levels >= `items`.
pub fn levels_needed(items: u64, fanout: u64) -> u32 {
    assert!(fanout >= 2);
    let mut levels = 1;
    let mut capacity = fanout;
    while capacity < items {
        capacity = capacity.saturating_mul(fanout);
        levels += 1;
    }
    levels
}
```

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn levels() {
        assert_eq!(levels_needed(1_000_000, 100), 3);
        assert_eq!(levels_needed(1_000_000_000, 200), 4);
        assert_eq!(levels_needed(100, 100), 1);
        assert_eq!(levels_needed(101, 100), 2);
        assert_eq!(levels_needed(u64::MAX, 2), 64);
    }
}
```

Why it works: each level multiplies the number of reachable items by the fanout, so `h` levels reach `fanout^h` items.
`200^4 = 1.6 × 10⁹ ≥ 10⁹`, but `200^3 = 8 × 10⁶` is not enough. The last assertion shows that `saturating_mul` stops at
`u64::MAX` instead of overflowing.

</details>

### Exercise 2 — A range query (medium)

Add `pub fn range(&self, lo: &K, hi: &K) -> Vec<(&K, &V)>` to `BTree`. It returns all entries with `lo <= key < hi`, in
key order. Visit only the children that can hold such keys. Test it against `BTreeMap::range(lo..hi)` for several ranges,
including an empty one (`50..50`) and one without any keys.

<details>
<summary>Hint</summary>

Write a recursive helper on `Node` that walks the node in order: for each key `i`, first child `i` (only if `keys[i] > lo`),
then the key itself (if it is in the range); stop as soon as a key is `>= hi`. After the loop, visit the last child. The
helper needs a lifetime `'a` that ties the returned references to `&'a self` (Lesson 05.3).

</details>

<details>
<summary>Answer</summary>

In `impl<K: Ord, V> BTree<K, V>`:

```rust
    /// All entries with `lo <= key < hi`, in key order.
    pub fn range(&self, lo: &K, hi: &K) -> Vec<(&K, &V)> {
        let mut out = Vec::new();
        self.root.range_into(lo, hi, &mut out);
        out
    }
```

In `impl<K: Ord, V> Node<K, V>`:

```rust
    /// Appends the entries of this subtree with `lo <= key < hi` to `out`, in order.
    fn range_into<'a>(&'a self, lo: &K, hi: &K, out: &mut Vec<(&'a K, &'a V)>) {
        for (i, key) in self.keys.iter().enumerate() {
            if !self.is_leaf() && key > lo {
                self.children[i].range_into(lo, hi, out); // keys smaller than `key`
            }
            if key >= hi {
                return; // everything further right is too big
            }
            if key >= lo {
                out.push((key, &self.vals[i]));
            }
        }
        if let Some(last) = self.children.last() {
            last.range_into(lo, hi, out); // keys bigger than the last key
        }
    }
```

The test:

```rust
    #[test]
    fn range_matches_std() {
        let mut ours = BTree::new(3);
        let mut std_map = BTreeMap::new();
        for k in (0..1_000).rev() {
            ours.insert(k * 3, k);
            std_map.insert(k * 3, k);
        }
        for (lo, hi) in [(0, 10), (100, 200), (2_990, 5_000), (50, 50), (7, 8)] {
            let expected: Vec<_> = std_map.range(lo..hi).collect();
            assert_eq!(ours.range(&lo, &hi), expected, "range {lo}..{hi}");
        }
    }
```

`cargo test --lib btree` now shows `3 passed`. Why it works: child `i` holds only keys smaller than `keys[i]`, so if
`keys[i] <= lo` the whole child is below the range and is skipped. Once a key reaches `hi`, every key and child to its right
is bigger, so the walk stops. This pruning is what makes ranges cheap: only the paths to the two ends of the range and the
keys inside it are visited. The keys were inserted in reverse order, which also tests that input pattern.

</details>

### Exercise 3 — A leaf in a page (hard)

In `src/page.rs`, write `pub fn encode_leaf(entries: &[(u64, u64)]) -> [u8; 4096]` and
`pub fn decode_leaf(page: &[u8; 4096]) -> Option<Vec<(u64, u64)>>`. The page layout: a little-endian `u16` count, then the
entries (8-byte key, 8-byte value), then zeros. Define `LEAF_CAPACITY`, the number of entries that fit. `decode_leaf` must
return `None` if the count is larger than the capacity. Test: the capacity is 255; a full leaf round-trips; a damaged
count is rejected.

<details>
<summary>Hint</summary>

Capacity: `(4096 - 2) / 16`. Entry `i` starts at byte `2 + i * 16`. Use `to_le_bytes`, `from_le_bytes` and `try_into`
exactly as in Lesson 17.3.

</details>

<details>
<summary>Answer</summary>

```rust
pub const PAGE_SIZE: usize = 4096;
/// How many (u64 key, u64 value) entries fit after the 2-byte count.
pub const LEAF_CAPACITY: usize = (PAGE_SIZE - 2) / 16;

/// Exercise 3: writes a leaf into one page: a u16 count, then the entries, then zeros.
pub fn encode_leaf(entries: &[(u64, u64)]) -> [u8; PAGE_SIZE] {
    assert!(
        entries.len() <= LEAF_CAPACITY,
        "too many entries for one page"
    );
    let mut page = [0u8; PAGE_SIZE];
    page[0..2].copy_from_slice(&(entries.len() as u16).to_le_bytes());
    for (i, (key, value)) in entries.iter().enumerate() {
        let at = 2 + i * 16;
        page[at..at + 8].copy_from_slice(&key.to_le_bytes());
        page[at + 8..at + 16].copy_from_slice(&value.to_le_bytes());
    }
    page
}

/// Exercise 3: reads the entries back; `None` if the count does not fit.
pub fn decode_leaf(page: &[u8; PAGE_SIZE]) -> Option<Vec<(u64, u64)>> {
    let count = u16::from_le_bytes([page[0], page[1]]) as usize;
    if count > LEAF_CAPACITY {
        return None;
    }
    let mut entries = Vec::with_capacity(count);
    for i in 0..count {
        let at = 2 + i * 16;
        let key = u64::from_le_bytes(page[at..at + 8].try_into().unwrap());
        let value = u64::from_le_bytes(page[at + 8..at + 16].try_into().unwrap());
        entries.push((key, value));
    }
    Some(entries)
}
```

Tests, next to the test of Exercise 1:

```rust
    #[test]
    fn a_full_leaf_round_trips() {
        assert_eq!(LEAF_CAPACITY, 255);
        let entries: Vec<(u64, u64)> = (0..255).map(|k| (k * 10, k * k)).collect();
        let page = encode_leaf(&entries);
        assert_eq!(decode_leaf(&page), Some(entries));
    }

    #[test]
    fn a_damaged_count_is_rejected() {
        let mut page = encode_leaf(&[(1, 2)]);
        page[0..2].copy_from_slice(&1000u16.to_le_bytes());
        assert_eq!(decode_leaf(&page), None);
    }
```

`cargo test --lib page`:

```text
running 3 tests
test page::tests::a_damaged_count_is_rejected ... ok
test page::tests::levels ... ok
test page::tests::a_full_leaf_round_trips ... ok

test result: ok. 3 passed; 0 failed; 0 ignored; 0 measured; 24 filtered out; finished in 0.00s
```

Why it works: every entry has a fixed size, so entry `i` is at a computed offset and no slots are needed. The capacity check
in `decode_leaf` is the `MAX_LEN` idea of Lesson 17.3: a count read from disk is input and is checked before it is used.
With 255 entries per leaf and `levels_needed(10⁹, 255)` = 4, a billion such entries need a tree of four levels.

</details>

## Quiz

1. Why is a balanced binary search tree a poor index on disk?

<details>
<summary>Answer</summary>

Its height is about `log₂ n` (20 for a million keys), and each level is a random disk read. Each read fetches a whole page
but uses only a few bytes of it.

</details>

2. With `t = 3`, how many keys can a node hold, and how many children can an internal node have?

<details>
<summary>Answer</summary>

Between `t - 1 = 2` and `2t - 1 = 5` keys (the root may have fewer), and one more child than keys: 3 to 6.

</details>

3. What happens when you insert into a full leaf?

<details>
<summary>Answer</summary>

The leaf is split: its middle key moves up into the parent, and the keys to the left and right of it become two leaves with
`t - 1` keys each. The new key then goes into the correct one of the two.

</details>

4. How does a B-tree get taller, and why does that keep all leaves at the same depth?

<details>
<summary>Answer</summary>

Only by splitting the root: a new root is created above it. The new level is added at the top, so every leaf moves one level
down at the same time.

</details>

5. What is the main difference between a B-tree and a B+ tree?

<details>
<summary>Answer</summary>

In a B+ tree all values are in the leaves and the leaves are linked in key order; internal nodes hold only keys and child
page numbers. This makes internal nodes smaller (higher fanout) and range scans a walk along the leaves.

</details>

6. Why does a database with a B-tree need a WAL, while the log of Lesson 17.5 could do without one?

<details>
<summary>Answer</summary>

A B-tree changes pages in place, and a crash can leave a torn page or a half-done split. The WAL records each change first,
so recovery can redo it. An append-only log never changes old bytes, so its only crash damage is a torn tail.

</details>

## Summary

- A B-tree is a balanced search tree with wide nodes: hundreds of sorted keys per node, one node per disk page.
- Its height is `log` with base equal to the fanout: three or four page reads find a key among billions.
- Search: binary search in the node, then go down to the child between the two neighbouring keys.
- Insertion adds to a leaf; full nodes are split and their middle key moves up; the tree grows only at the root, so it stays
  balanced even for sorted input.
- Testing against `BTreeMap` (a model) catches the off-by-one bugs of splitting.
- On disk, nodes are pages, often slotted pages; real databases use B+ trees with linked leaves for range scans.
- B-trees change pages in place, so they need a WAL for crash safety.

## Sources and further reading

- [std::collections::BTreeMap](https://doc.rust-lang.org/std/collections/struct.BTreeMap.html) — the standard library's
  B-tree, and its note on why it uses a B-tree instead of a binary tree.
- [PostgreSQL documentation — B-Tree Indexes](https://www.postgresql.org/docs/current/btree.html) — how a real database
  implements and uses its default index type.
- [SQLite — Database File Format](https://www.sqlite.org/fileformat2.html) — the section on b-tree pages shows a real
  slotted page layout, byte by byte.
- [Let's Build a Simple Database](https://cstack.github.io/db_tutorial/) — a step-by-step tutorial that builds an on-disk
  B-tree for a tiny SQLite clone (in C, but the ideas carry over).
- [std::mem::replace](https://doc.rust-lang.org/std/mem/fn.replace.html) — the function that lets `insert` swap the root.

---

[🏠 Start](../START-HERE.md) · [← 17.5 Indexes: a hash index](05-hash-indexes.md) · [📋 Phase 17](README.md) · [17.7 LSM trees →](07-lsm-trees.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
