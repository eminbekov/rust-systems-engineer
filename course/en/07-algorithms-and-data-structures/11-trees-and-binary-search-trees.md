<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 07.10 Binary search](10-binary-search.md) · [📋 Phase 07](README.md) · [07.12 Heaps and priority queues →](12-heaps-and-priority-queues.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/07-algorithms-and-data-structures/11-trees-and-binary-search-trees.md) <!-- nav -->

# Lesson 07.11 — Trees and binary search trees

> **Phase 07 — Algorithms and data structures** · Lesson 11 of 20 · ⏱ about 3 hours (2 sittings)

## What you will learn

- What a tree is, and the words for its parts: root, node, child, leaf, subtree, height.
- The binary search tree (BST) rule, and how search, insert, minimum, maximum and remove follow from it.
- How to build a BST in Rust with `Option<Box<Node<T>>>`, and the three ways to walk through a tree (in-order, pre-order, post-order).
- Why a BST can degrade into a list, what a self-balancing tree is, and which standard types to use (`BTreeMap`, `BTreeSet`).

## Before you start

- You finished [Lesson 07.10 — Binary search](10-binary-search.md). The BST search is the same idea on a different structure.
- You know `Box` and recursive types ([Lesson 05.9 — Smart pointers](../05-rust-intermediate/09-smart-pointers.md), Section 3), `Option` ([Lesson 04.4](../04-structs-enums-collections-errors/04-option-and-match.md)) and iterators ([Lesson 05.5](../05-rust-intermediate/05-iterators.md)).
- In `code/p07-algorithms` create `src/trees.rs` and add `pub mod trees;` to `src/lib.rs`.

## New words

| Word | Meaning |
|---|---|
| **tree** | A data structure made of **nodes** connected by links, with one top node and no cycles. Like a family tree, or like folders inside folders. |
| **node** | One element of a tree. It holds a value and links to its children. |
| **root** | The top node. It has no parent. |
| **child / parent** | A node directly below another node (its child) / the node directly above it (its parent). |
| **leaf** | A node with no children. |
| **subtree** | A node together with everything below it. Every child is the root of a subtree. |
| **binary tree** | A tree where every node has at most two children, called **left** and **right**. |
| **height** | The number of levels of the tree: the length of the longest path from the root down to a leaf, counted in nodes. An empty tree has height 0. |
| **binary search tree (BST)** | A binary tree in which, for every node, all values in its left subtree are smaller and all values in its right subtree are bigger. |
| **traversal** | Visiting every node of a tree in some order. |
| **in-order / pre-order / post-order** | Traversal orders: left–node–right / node–left–right / left–right–node. |
| **balanced** | A tree whose height is close to log₂ n. A *self-balancing* tree repairs its shape after each change. |
| **AVL tree** (Adelson-Velsky and Landis) | A classic self-balancing binary search tree, named after its two inventors. |

## 1. Trees

A list has a first item and each item has one next item. A **tree** lets an item have several "next" items. You see trees every day:
folders on a disk (a folder holds files and other folders), a company chart, an HTML (HyperText Markup Language, the language of web pages) page. In a **binary** tree each node has at most two children:

```text
            50              level 1: the root
          /    \
        30      70          level 2
       /  \    /  \
     20   40  60   80       level 3: these four nodes are leaves
          /
        35                  level 4: also a leaf
```

The tree has 8 nodes and height 4. The node `30` has the parent `50`, the children `20` and `40`, and the subtree `{30, 20, 40, 35}`.
In Rust a node needs a value and two optional children. A child lives on the heap in a `Box`, because a node that contains
another node directly would have an infinite size (Lesson 05.9):

```text
struct Node<T> { value: T, left: Option<Box<Node<T>>>, right: Option<Box<Node<T>>> }
```

`None` means "no child here". The whole tree is just a pointer to its root: `Option<Box<Node<T>>>`.

## 2. The binary search tree rule

A **binary search tree** keeps one extra rule at every node:

> everything in the left subtree is **smaller** than the node, everything in the right subtree is **bigger**.

The picture above obeys it: `50` has smaller values on the left (30, 20, 40, 35) and bigger on the right (70, 60, 80). The rule gives us binary search for free.
To look for `35`: at `50`, go left (35 < 50); at `30`, go right (35 > 30); at `40`, go left (35 < 40); found `35`. Four steps, one per level. Looking for `36`: after `35`,
the right child is `None`: not in the tree. **Insert** works the same way: search for the value, and where the search falls off the tree, hang a new leaf.

The cost of search and insert is the **height** of the tree. For a tree that is well shaped, the height is about log₂ n, so we get O(log n), as with binary
search. The minimum is the leftmost node (go left until you cannot), the maximum is the rightmost.

### Walking through a tree

There are three classic depth-first orders. For each node you do three things: visit the node itself, walk the left subtree, walk the right subtree. They differ in the order:

```text
in-order    (left, node, right):   20 30 35 40 50 60 70 80     <- sorted!
pre-order   (node, left, right):   50 30 20 40 35 70 60 80     <- a parent before its children
post-order  (left, right, node):   20 35 40 30 60 80 70 50     <- children before their parent
```

In-order on a BST gives the values **in increasing order**: this is how a BST sorts. Pre-order is good for copying a tree, post-order for deleting one.

## 3. A BST in Rust

Put the types, `insert` and `contains` into `src/trees.rs`:

```rust
//! A binary search tree.

use std::cmp::Ordering;

#[derive(Debug)]
struct Node<T> {
    value: T,
    left: Tree<T>,
    right: Tree<T>,
}

/// A (possibly empty) tree: `None` is the empty tree, `Some` is a node with its subtrees.
type Tree<T> = Option<Box<Node<T>>>;

/// A set of values kept in a binary search tree.
#[derive(Debug)]
pub struct BinarySearchTree<T> {
    root: Tree<T>,
    len: usize,
}

impl<T: Ord> BinarySearchTree<T> {
    pub fn new() -> Self {
        BinarySearchTree { root: None, len: 0 }
    }

    pub fn len(&self) -> usize {
        self.len
    }

    pub fn is_empty(&self) -> bool {
        self.len == 0
    }

    /// Adds `value`. Returns false (and changes nothing) if it is already in the tree.
    pub fn insert(&mut self, value: T) -> bool {
        let mut current = &mut self.root;
        while let Some(node) = current {
            match value.cmp(&node.value) {
                Ordering::Less => current = &mut node.left,
                Ordering::Greater => current = &mut node.right,
                Ordering::Equal => return false,
            }
        }
        *current = Some(Box::new(Node {
            value,
            left: None,
            right: None,
        }));
        self.len += 1;
        true
    }

    pub fn contains(&self, value: &T) -> bool {
        let mut current = &self.root;
        while let Some(node) = current {
            match value.cmp(&node.value) {
                Ordering::Less => current = &node.left,
                Ordering::Greater => current = &node.right,
                Ordering::Equal => return true,
            }
        }
        false
    }
}

impl<T: Ord> Default for BinarySearchTree<T> {
    fn default() -> Self {
        Self::new()
    }
}
```

**What happens:**

1. `type Tree<T> = Option<Box<Node<T>>>` is a **type alias**: a shorter name for a long type. A `Tree<T>` is either empty (`None`) or a node (`Some`) with two more
   `Tree<T>` inside. The structure is recursive, exactly like the definition in Section 1.
2. `BinarySearchTree` stores the root and a counter `len`, so `len()` costs O(1). We implement a **set**: every value appears at most once.
3. In `insert`, `current` is a *mutable reference to a slot* (a `Tree<T>`). It starts at the root slot. While the slot holds a node, we compare and move the reference into the left
   or right slot of that node. Where the walk meets an empty slot (`None`), the loop ends and `*current = Some(...)` puts the new node into it.
   No recursion is used, so a tall tree cannot overflow the call stack here.
4. `Ordering::Equal` returns `false`: we do not store duplicates. (`Ord` guarantees that `cmp` is a consistent order, Lesson 07.8.)
5. `contains` is the same walk with shared references and no change. Each loop turn goes one level down, so the cost is O(height).
6. `Default` (the standard trait for "a sensible empty value") is added because Clippy asks for it whenever there is a `new()` without arguments.

Now the other operations. Add a second `impl` block:

```rust
impl<T: Ord> BinarySearchTree<T> {
    /// The smallest value: always the leftmost node.
    pub fn min(&self) -> Option<&T> {
        let mut current = self.root.as_ref()?;
        while let Some(left) = current.left.as_ref() {
            current = left;
        }
        Some(&current.value)
    }

    /// The biggest value: always the rightmost node.
    pub fn max(&self) -> Option<&T> {
        let mut current = self.root.as_ref()?;
        while let Some(right) = current.right.as_ref() {
            current = right;
        }
        Some(&current.value)
    }

    /// The number of levels: 0 for an empty tree, 1 for a single node.
    pub fn height(&self) -> usize {
        height(&self.root)
    }

    /// All values in increasing order (in-order traversal).
    pub fn to_vec(&self) -> Vec<&T> {
        let mut out = Vec::with_capacity(self.len);
        in_order(&self.root, &mut out);
        out
    }

    /// Removes `value`. Returns false if it was not in the tree.
    pub fn remove(&mut self, value: &T) -> bool {
        let removed = remove_from(&mut self.root, value);
        if removed {
            self.len -= 1;
        }
        removed
    }
}

fn height<T>(tree: &Tree<T>) -> usize {
    match tree {
        None => 0,
        Some(node) => 1 + height(&node.left).max(height(&node.right)),
    }
}

fn in_order<'a, T>(tree: &'a Tree<T>, out: &mut Vec<&'a T>) {
    if let Some(node) = tree {
        in_order(&node.left, out); // 1. everything smaller
        out.push(&node.value); // 2. this node
        in_order(&node.right, out); // 3. everything bigger
    }
}
```

**What happens:**

1. `self.root.as_ref()?` turns `&Option<Box<Node>>` into `Option<&Box<Node>>`, and `?` returns `None` for an empty tree. Then `min` follows the `left` links
   until there is none; `max` does the same on the right.
2. `height` and `in_order` are **recursive**, like the definition of the tree. The height of an empty tree is 0; the height of a node is 1 plus the bigger height of its two subtrees.
3. `in_order` follows the order in its comments: left subtree, then the node, then the right subtree. Because of the BST rule, the output is sorted.
   The lifetime `'a` says that the references we collect point into the tree.

### Removing a value

`remove` is the hardest operation, because the tree must keep its rule after the node is gone. First find the node, as in `contains`. Then there are three cases:

```text
1. A leaf:        just cut it off.                 50                50
                                                 /  \     20?      /  \
                                               30    70   --->   30    70

2. One child:     the child takes its place.      30                 30
                                                 /  \     40?      /  \
                                               20   40   --->    20   35
                                                    /
                                                  35

3. Two children:  find the smallest value of the right subtree (the next value in the
                  sorted order), move it into this node, and remove it from where it was.
                         50 <- remove                   60
                        /  \                           /  \
                      30    70          --->         30    70
                           /  \                             \
                         60    80                           80
```

Why the smallest value of the right subtree? It is bigger than everything on the left and not bigger than anything on the right, so it fits exactly into the gap. Add the helpers:

```rust
fn remove_from<T: Ord>(tree: &mut Tree<T>, value: &T) -> bool {
    let Some(node) = tree else {
        return false; // empty tree: the value is not here
    };
    match value.cmp(&node.value) {
        Ordering::Less => remove_from(&mut node.left, value),
        Ordering::Greater => remove_from(&mut node.right, value),
        Ordering::Equal => {
            match (node.left.take(), node.right.take()) {
                (None, None) => *tree = None,             // a leaf: just cut it off
                (Some(left), None) => *tree = Some(left), // one child: it takes the place
                (None, Some(right)) => *tree = Some(right),
                (Some(left), Some(right)) => {
                    // Two children: move the smallest value of the right subtree up.
                    let mut right = Some(right);
                    node.value = take_min(&mut right);
                    node.left = Some(left);
                    node.right = right;
                }
            }
            true
        }
    }
}

/// Removes the leftmost node of a non-empty tree and returns its value.
fn take_min<T>(tree: &mut Tree<T>) -> T {
    if tree.as_ref().is_some_and(|node| node.left.is_some()) {
        take_min(&mut tree.as_mut().unwrap().left)
    } else {
        let node = tree.take().expect("take_min needs a non-empty tree");
        *tree = node.right; // the right subtree takes the place of the node
        node.value
    }
}
```

**What happens:**

1. `let Some(node) = tree else { return false; }` is `let`-`else` (Lesson 04.5): for an empty tree the value is not there.
2. `Less` and `Greater` go down one level, recursively, with a mutable reference to the left or right slot.
3. On `Equal`, `node.left.take()` and `node.right.take()` move both children **out** of the node and leave `None` behind. (`Option::take` is the way to move a value
   out of a place you only borrow; Common mistakes shows what happens without it.) Matching the pair of children gives the three cases.
4. For a leaf, `*tree = None` drops the node. For one child, `*tree = Some(child)` puts the child where the node was.
5. For two children, `take_min` removes the leftmost node from the right subtree and returns its value. That value overwrites `node.value`, and the two subtrees are hung back.
6. `take_min` walks down the `left` links. At the leftmost node it takes the node out of its slot and puts the node's own right subtree there (the leftmost node has no left child).

## 4. Testing

The model for the tests is `BTreeSet` from the standard library, a sorted set (Section 6). We apply the same random operations to our tree and to the model and compare every answer. Add to the end of `src/trees.rs`:

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use crate::testutil::Rng;
    use std::collections::BTreeSet;

    /// The tree drawn in the lesson.
    fn sample() -> BinarySearchTree<i32> {
        let mut tree = BinarySearchTree::new();
        for value in [50, 30, 70, 20, 40, 60, 80, 35] {
            assert!(tree.insert(value));
        }
        tree
    }

    #[test]
    fn empty_tree() {
        let tree: BinarySearchTree<i32> = BinarySearchTree::new();
        assert!(tree.is_empty());
        assert_eq!(tree.height(), 0);
        assert_eq!((tree.min(), tree.max()), (None, None));
        assert!(!tree.contains(&1));
        assert!(tree.to_vec().is_empty());
    }

    #[test]
    fn insert_search_and_traversal() {
        let mut tree = sample();
        assert_eq!(tree.len(), 8);
        assert_eq!(tree.height(), 4);
        assert_eq!(tree.min(), Some(&20));
        assert_eq!(tree.max(), Some(&80));
        assert_eq!(tree.to_vec(), [&20, &30, &35, &40, &50, &60, &70, &80]);
        assert!(tree.contains(&35));
        assert!(!tree.contains(&36));
        assert!(!tree.insert(40)); // a duplicate is refused
        assert_eq!(tree.len(), 8);
    }

    #[test]
    fn remove_leaf_one_child_and_two_children() {
        let mut tree = sample();
        assert!(tree.remove(&20)); // a leaf
        assert!(tree.remove(&40)); // one child (35)
        assert!(tree.remove(&50)); // the root, with two children
        assert!(!tree.remove(&50)); // already gone
        assert_eq!(tree.to_vec(), [&30, &35, &60, &70, &80]);
        assert_eq!(tree.len(), 5);
        assert_eq!(tree.root.as_ref().map(|n| n.value), Some(60)); // 60 moved up
    }

    #[test]
    fn behaves_like_btreeset() {
        for seed in 1..=5u64 {
            let mut rng = Rng::new(seed);
            let mut tree = BinarySearchTree::new();
            let mut model = BTreeSet::new();
            for _ in 0..2000 {
                let value = rng.below(50) as i32;
                match rng.below(3) {
                    0 => assert_eq!(tree.insert(value), model.insert(value)),
                    1 => assert_eq!(tree.remove(&value), model.remove(&value)),
                    _ => assert_eq!(tree.contains(&value), model.contains(&value)),
                }
                assert_eq!(tree.len(), model.len());
            }
            assert_eq!(tree.to_vec(), model.iter().collect::<Vec<_>>());
        }
    }

    #[test]
    fn sorted_input_makes_a_tall_tree() {
        let mut sorted = BinarySearchTree::new();
        for value in 0..500 {
            sorted.insert(value);
        }
        assert_eq!(sorted.height(), 500); // a linked list in disguise
    }
}
```

**What happens:**

1. `sample()` builds the tree of the pictures. The tests check the numbers we computed by hand: 8 nodes, height 4, minimum 20, maximum 80, in-order sorted.
2. `remove_leaf_one_child_and_two_children` removes one node of each kind. After removing the root `50`, the value `60` (the smallest of the right subtree) is the new root.
3. `behaves_like_btreeset` is **differential testing** (Lesson 07.8): 2,000 random operations on a small range of values (0 to 49), so inserts, removes and searches often hit existing values.
   `rng.below(3)` (the `Rng` of Lesson 07.3) chooses one of the three operations. After every step the sizes must agree, and at the end the traversals must be identical.
4. The last test shows the problem of Section 5 on purpose.

```bash
cd ~/projects/rust/code/p07-algorithms
cargo test trees
```

```text
running 5 tests
test trees::tests::empty_tree ... ok
test trees::tests::insert_search_and_traversal ... ok
test trees::tests::remove_leaf_one_child_and_two_children ... ok
test trees::tests::sorted_input_makes_a_tall_tree ... ok
test trees::tests::behaves_like_btreeset ... ok

test result: ok. 5 passed; 0 failed; 0 ignored; 0 measured; 36 filtered out; finished in 0.00s
```

(The 36 filtered-out tests come from the main text of Lessons 07.1 to 07.10; with the tests of the exercises your number is bigger.)

## 5. The problem: the shape of the tree

The speed of a BST depends on its **height**, and the height depends on the **order of insertion**. If you insert already sorted values, each one becomes the right child of the previous one, and the
tree is a chain: a linked list with extra fields (the last test above). Search then needs O(n). Create `examples/e11_tree_heights.rs`:

```rust
use p07_algorithms::trees::BinarySearchTree;

fn main() {
    println!(
        "{:>7} {:>14} {:>14} {:>10}",
        "n", "sorted input", "random input", "log2(n)"
    );
    for n in [1_000u64, 10_000] {
        let mut sorted = BinarySearchTree::new();
        let mut random = BinarySearchTree::new();
        let mut x: u64 = 12345;
        for i in 0..n {
            sorted.insert(i);
            x ^= x << 13;
            x ^= x >> 7;
            x ^= x << 17;
            random.insert(x);
        }
        println!(
            "{:>7} {:>14} {:>14} {:>10.1}",
            n,
            sorted.height(),
            random.height(),
            (n as f64).log2()
        );
    }
}
```

```bash
cargo run --example e11_tree_heights
```

```text
      n   sorted input   random input    log2(n)
   1000           1000             23       10.0
  10000          10000             31       13.3
```

**How to read it:** after sorted insertion the height equals `n`; searching 10,000 items needs up to 10,000 steps. After random insertion the height is 31, about 2 to 2.5
times `log₂ n`: not perfect, but fine. Real data is often partly sorted (timestamps, ids), so the bad case is common.

The cure is a **self-balancing tree**: after each insert or remove it rotates a few nodes to keep the height O(log n). Famous kinds are the AVL tree (AVL: the initials of its inventors Adelson-Velsky and Landis; it keeps the heights of the two subtrees of every node within 1 of each other) and the red-black tree. The standard library uses a
different kind, the **B-tree**, where one node holds many values, which suits modern processors because it needs fewer jumps through memory. It is
available as `BTreeMap` and `BTreeSet`, and these are the types to use in real code. You will not write a balanced tree yourself in this course; knowing why it exists is enough.

> ⚠️ **Common mistake:** An unbalanced BST looks fine in tests with small or random data and fails in production with sorted data. If your data may arrive in order, never use a plain BST.

## 6. What to use in real code

```rust
use std::collections::BTreeMap;

fn main() {
    let mut scores = BTreeMap::new();
    scores.insert("carol", 72);
    scores.insert("alice", 90);
    scores.insert("bob", 85);
    for (name, score) in &scores {
        println!("{name}: {score}");
    }
    println!("{:?}", scores.first_key_value());
    for (name, score) in scores.range("b".."c") {
        println!("range: {name} {score}");
    }
}
```

```text
alice: 90
bob: 85
carol: 72
Some(("alice", 90))
range: bob 85
```

**What happens:** a `BTreeMap` keeps its keys sorted. Iterating gives the keys in order (as our in-order traversal does); `first_key_value` returns the smallest key; and `range("b".."c")` returns
only the keys from `"b"` up to but not including `"c"`. All of it is O(log n) per operation, with no worry about the shape.

| Need | Use | Lookup | Ordered iteration, ranges |
|---|---|---|---|
| fastest lookup, order does not matter | `HashMap` / `HashSet` | O(1) on average | no |
| sorted, with inserts and removes | `BTreeMap` / `BTreeSet` | O(log n) | yes |
| built once, searched often | sorted `Vec` + `binary_search` | O(log n) | yes (slices) |
| learn how a tree works | your own `BinarySearchTree` | O(height) | yes |

Trees are everywhere besides the search: file systems, expression parsers, the structure of a database index (Phase 17), the
heap of the next lesson. The BST is the plain example of "a recursive structure with recursive functions".

All four structures answer a lookup in a small fraction of a microsecond when the data has 100,000 numbers; a linear scan needs far more (Lesson 07.10). So choose by the operations you need, as in the table, not
by guessing the speed. If you want numbers for your own case, write a `criterion` benchmark as in Lesson 05.12.

## Common mistakes

**1. Moving a child out of a borrowed node.** In `remove` you may try to match on the children directly:

```rust
struct Node {
    value: i32,
    left: Option<Box<Node>>,
    right: Option<Box<Node>>,
}

pub fn cut_children(tree: &mut Option<Box<Node>>) {
    let Some(node) = tree else { return };
    match (node.left, node.right) {
        (None, None) => *tree = None,
        _ => {}
    }
}
```

```text
error[E0507]: cannot move out of `node.left` which is behind a mutable reference
 --> src/lib.rs:9:12
  |
9 |     match (node.left, node.right) {
  |            ^^^^^^^^^ move occurs because `node.left` has type `Option<Box<Node>>`, which does not implement the `Copy` trait
```

`node` is only a borrowed reference, so you cannot take the children out and leave a hole behind. The fix is `Option::take()`, which moves the value out **and** puts `None` in its place:
`match (node.left.take(), node.right.take())`. (The same error appears with `Rc` lists and many other recursive structures: reach for `take()`.)

**2. Using a BST for ordered input.** See Section 5: the height becomes `n`. Use `BTreeMap` / `BTreeSet`.

**3. Forgetting `len` on remove.** If `remove` returns `true`, the counter must go down. The `behaves_like_btreeset` test compares `len()` after every step, which catches this.

## Try it yourself

1. Add the code of Sections 3 and 4 step by step. Run `cargo test trees`.
2. Run the example `e11_tree_heights`. Change the sorted input to descending numbers (`n - i`). What is the height now?
3. In `remove_from`, comment out the line `node.right = right;` in the two-children case and run the tests. Which test fails? Put the line back.
4. Run `cargo fmt`, `cargo clippy --all-targets`, and commit: `git add code/p07-algorithms && git commit -m "feat(p07): binary search tree"`.

## Exercises

### Exercise 1 — Draw and walk (easy)

Insert `8, 3, 10, 1, 6, 14, 4, 7, 13` one after another into an empty BST. Draw the tree and write down its height and its in-order, pre-order and post-order traversals. Then remove `3` and write down the in-order traversal and the new left child of the root.

<details>
<summary>Hint</summary>

`8` is the root. `3 < 8` goes left; `10 > 8` goes right; `1 < 8`, `1 < 3` goes left of `3`; `6` goes right of `3`; and so on. For the removal: `3` has two children (`1` and `6`); the replacement is the smallest value of the subtree of `6`.

</details>

<details>
<summary>Answer</summary>

```text
            8
          /   \
        3      10
       / \       \
      1   6       14
         / \      /
        4   7    13
```

Height 4 (the path 8, 3, 6, 4).

```text
in-order:    1 3 4 6 7 8 10 13 14
pre-order:   8 3 1 6 4 7 10 14 13
post-order:  1 4 7 6 3 13 14 10 8
```

Removing `3` (two children): the smallest value in the right subtree (the one rooted at `6`) is `4`. It moves into the node of `3`, and the old `4` leaf is cut off. The in-order traversal becomes `1 4 6 7 8 10 13 14`,
and the left child of the root is now `4`, with `1` on its left and `6` (with the child `7`) on its right.

</details>

### Exercise 2 — Test the order (easy)

Write a test that inserts many pseudo-random numbers and checks that `to_vec()` is strictly increasing (sorted, no equal neighbours) and has exactly `len()` items.

<details>
<summary>Hint</summary>

Make the numbers with `Rng`, as in `behaves_like_btreeset`. `to_vec()` gives `Vec<&i32>`. Compare each pair with `windows(2)`: `pair[0] < pair[1]`.

</details>

<details>
<summary>Answer</summary>

```rust
    #[test]
    fn in_order_is_strictly_increasing() {
        let mut tree = BinarySearchTree::new();
        let mut rng = Rng::new(99);
        for _ in 0..1000 {
            tree.insert(rng.below(500) as i32);
        }
        let items = tree.to_vec();
        assert_eq!(items.len(), tree.len());
        assert!(items.windows(2).all(|pair| pair[0] < pair[1]));
    }
```

Why it works: the values come from the range 0 to 499, so many inserts are duplicates, which the tree refuses; `len()` counts only the distinct values. A strictly increasing in-order list is the visible result of the BST rule.

</details>

### Exercise 3 — State the complexity (medium)

1. What is the cost of `contains` in a BST with `n` nodes and height `h`? What are `h` in the best and in the worst shape?
2. What does it cost to build a BST by inserting `n` sorted values, and `n` well-shuffled values (on average)?
3. What is the extra memory of `in_order` for a tree of height `h`?

<details>
<summary>Hint</summary>

Every step of the search goes one level down. For (3) think about how many calls wait on the call stack at the deepest point.

</details>

<details>
<summary>Answer</summary>

1. **O(h)**. The best shape has `h ≈ log₂ n`, so O(log n); the worst shape (a chain) has `h = n`, so O(n).
2. Sorted values: insert number `i` walks `i` levels, so the total is `1 + 2 + … + n`, **O(n²)**. Shuffled values: each insert costs about O(log n) on average, so **O(n log n)** in total.
3. **O(h)** for the call stack (plus the output list). For a chain of `n` nodes that is O(n) calls, and a very long chain can overflow the stack. This is one more reason to balance.

</details>

### Exercise 4 — Is it really a BST? (medium)

Write the function `is_bst<T: Ord>(tree: &Tree<T>, lower: Option<&T>, upper: Option<&T>) -> bool` that checks the BST rule of a whole tree (inside the `tests` module, it is a test helper). Test it on the sample tree and on a hand-built tree that is **wrong**
although every node is correct compared to its own children.

<details>
<summary>Hint</summary>

It is not enough to compare a node with its children. Pass down the allowed range: going left, the current value becomes the new `upper`; going right, it becomes the new `lower`. `Option::is_none_or` is handy: `None` means "no limit".

</details>

<details>
<summary>Answer</summary>

```rust
    /// Is every value in the tree bigger than `lower` and smaller than `upper` (when they exist)?
    fn is_bst<T: Ord>(tree: &Tree<T>, lower: Option<&T>, upper: Option<&T>) -> bool {
        match tree {
            None => true,
            Some(node) => {
                lower.is_none_or(|low| low < &node.value)
                    && upper.is_none_or(|high| &node.value < high)
                    && is_bst(&node.left, lower, Some(&node.value))
                    && is_bst(&node.right, Some(&node.value), upper)
            }
        }
    }

    fn leaf(value: i32) -> Tree<i32> {
        Some(Box::new(Node {
            value,
            left: None,
            right: None,
        }))
    }

    #[test]
    fn is_bst_detects_a_bad_tree() {
        assert!(is_bst(&sample().root, None, None));
        // 15 is the right child of 5 (fine on its own), but it sits in the left subtree of 10 (wrong).
        let bad = Some(Box::new(Node {
            value: 10,
            left: Some(Box::new(Node {
                value: 5,
                left: None,
                right: leaf(15),
            })),
            right: leaf(20),
        }));
        assert!(!is_bst(&bad, None, None));
    }
```

Why it works: the node `15` has the allowed range `(5, 10)` (it is right of `5`, left of `10`), and `15` is not below 10, so the check fails. Checking only parent and child would accept this tree, and a `contains(&15)` would walk left at `10` and never find it.

</details>

## Quiz

1. What is the BST rule?

<details>
<summary>Answer</summary>

For every node, all values in its left subtree are smaller than the node's value and all values in its right subtree are bigger.

</details>

2. Which traversal order visits a BST in sorted order?

<details>
<summary>Answer</summary>

In-order: left subtree, node, right subtree.

</details>

3. How is a node with two children removed?

<details>
<summary>Answer</summary>

Find the smallest value of its right subtree, copy it into the node, and remove that smallest node from the right subtree (it has at most one child, so the removal is simple).

</details>

4. Why can a BST be as slow as a linked list?

<details>
<summary>Answer</summary>

If values are inserted in sorted order, every new node becomes a child on the same side, so the tree is a chain of height `n`, and search costs O(n).

</details>

5. Which standard types should you use instead of your own BST?

<details>
<summary>Answer</summary>

`BTreeMap` and `BTreeSet` (sorted, O(log n), range queries), or `HashMap` / `HashSet` when no order is needed.

</details>

## Summary

- A tree is made of nodes with a root at the top; a binary tree has at most two children per node. Height, leaf and subtree are the basic words.
- A BST keeps smaller values on the left and bigger on the right. Search, insert, minimum and maximum follow one path from the root: O(height).
- In Rust a tree is `Option<Box<Node<T>>>`; walk with loops and references to slots, or with recursion; use `Option::take` to move children out.
- In-order traversal of a BST is sorted. Pre-order and post-order are the two other depth-first orders.
- Removing a node has three cases: leaf, one child, two children (replace by the smallest value of the right subtree).
- The height decides the speed: sorted input turns a plain BST into a list. Self-balancing trees fix this; the standard library has `BTreeMap` and `BTreeSet`.
- Test a collection by running the same random operations on it and on a trusted model such as `BTreeSet`.

## Sources and further reading

- [Rust standard library — std::collections::BTreeMap](https://doc.rust-lang.org/std/collections/struct.BTreeMap.html) — the sorted map: methods, ranges, and a note on how it is built.
- [Rust standard library — std::collections::BTreeSet](https://doc.rust-lang.org/std/collections/struct.BTreeSet.html) — the sorted set.
- [Rust standard library — Option::take](https://doc.rust-lang.org/std/option/enum.Option.html#method.take) — moving a value out of a borrowed `Option`.
- [The Rust Programming Language — Using Box to enable recursive types](https://doc.rust-lang.org/book/ch15-01-box.html) — why a recursive node needs a `Box`.
- [Learn Rust With Entirely Too Many Linked Lists](https://rust-unofficial.github.io/too-many-lists/) — a free book on pointer-based structures in Rust.

---

[🏠 Start](../START-HERE.md) · [← 07.10 Binary search](10-binary-search.md) · [📋 Phase 07](README.md) · [07.12 Heaps and priority queues →](12-heaps-and-priority-queues.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
