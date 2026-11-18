<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 07.10 Двоичный поиск](10-binary-search.md) · [📋 Этап 07](README.md) · [07.12 Кучи и очереди с приоритетом →](12-heaps-and-priority-queues.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/07-algorithms-and-data-structures/11-trees-and-binary-search-trees.md) <!-- nav -->

# Урок 07.11 — Деревья и двоичные деревья поиска

> **Этап 07 — Алгоритмы и структуры данных** · Урок 11 из 20 · ⏱ около 3 часов (2 занятия)

## Что вы узнаете

- Что такое дерево и слова для его частей: корень, узел, потомок, лист, поддерево, высота.
- Правило двоичного дерева поиска (BST) и как из него следуют поиск, вставка, минимум, максимум и удаление.
- Как построить BST в Rust на `Option<Box<Node<T>>>` и три способа обойти дерево (симметричный, прямой, обратный).
- Почему BST может выродиться в список, что такое самобалансирующееся дерево и какие стандартные типы использовать (`BTreeMap`, `BTreeSet`).

## Перед началом

- Вы прошли [Урок 07.10 — Двоичный поиск](10-binary-search.md). Поиск в BST — та же идея на другой структуре.
- Вы знаете `Box` и рекурсивные типы ([Урок 05.9 — Умные указатели](../05-rust-intermediate/09-smart-pointers.md), раздел 3), `Option` ([Урок 04.4](../04-structs-enums-collections-errors/04-option-and-match.md)) и итераторы ([Урок 05.5](../05-rust-intermediate/05-iterators.md)).
- В `code/p07-algorithms` создайте `src/trees.rs` и добавьте `pub mod trees;` в `src/lib.rs`.

## Новые слова

| Слово | Значение |
|---|---|
| **дерево** (tree) | Структура данных из **узлов**, соединённых связями, с одним верхним узлом и без циклов. Как семейное древо или как папки внутри папок. |
| **узел** (node) | Один элемент дерева. Хранит значение и связи со своими потомками. |
| **корень** (root) | Верхний узел. У него нет родителя. |
| **потомок / родитель** (child / parent) | Узел непосредственно под другим узлом (его потомок) / узел непосредственно над ним (его родитель). |
| **лист** (leaf) | Узел без потомков. |
| **поддерево** (subtree) | Узел вместе со всем, что находится под ним. Каждый потомок — корень поддерева. |
| **двоичное дерево** (binary tree) | Дерево, где у каждого узла не более двух потомков, называемых **левым** и **правым**. |
| **высота** (height) | Число уровней дерева: длина самого длинного пути от корня вниз до листа, считаемая в узлах. У пустого дерева высота 0. |
| **двоичное дерево поиска** (binary search tree, BST) | Двоичное дерево, в котором для каждого узла все значения в левом поддереве меньше, а все значения в правом поддереве больше. |
| **обход** (traversal) | Посещение каждого узла дерева в некотором порядке. |
| **симметричный / прямой / обратный обход** (in-order / pre-order / post-order) | Порядки обхода: левое–узел–правое / узел–левое–правое / левое–правое–узел. |
| **сбалансированное** (balanced) | Дерево, высота которого близка к log₂ n. *Самобалансирующееся* дерево восстанавливает свою форму после каждого изменения. |
| **AVL-дерево** (AVL tree, Adelson-Velsky and Landis) | Классическое самобалансирующееся двоичное дерево поиска, названное по двум его авторам — Адельсону-Вельскому и Ландису. |

## 1. Деревья

У списка есть первый элемент, и у каждого элемента один следующий. **Дерево** позволяет элементу иметь несколько «следующих». Деревья вы видите каждый день:
папки на диске (папка содержит файлы и другие папки), схема компании, страница HTML (HyperText Markup Language, язык разметки веб-страниц). В **двоичном** дереве у каждого узла не более двух потомков:

```text
            50              level 1: the root
          /    \
        30      70          level 2
       /  \    /  \
     20   40  60   80       level 3: these four nodes are leaves
          /
        35                  level 4: also a leaf
```

В дереве 8 узлов и высота 4. У узла `30` есть родитель `50`, потомки `20` и `40` и поддерево `{30, 20, 40, 35}`.
В Rust узлу нужны значение и два необязательных потомка. Потомок живёт в куче внутри `Box`, потому что узел, содержащий
другой узел непосредственно, имел бы бесконечный размер (Урок 05.9):

```text
struct Node<T> { value: T, left: Option<Box<Node<T>>>, right: Option<Box<Node<T>>> }
```

`None` означает «здесь потомка нет». Всё дерево — просто указатель на корень: `Option<Box<Node<T>>>`.

## 2. Правило двоичного дерева поиска

**Двоичное дерево поиска** поддерживает в каждом узле одно дополнительное правило:

> всё в левом поддереве **меньше** узла, всё в правом поддереве **больше**.

Картинка выше ему подчиняется: у `50` слева меньшие значения (30, 20, 40, 35), а справа большие (70, 60, 80). Это правило даёт нам двоичный поиск бесплатно.
Ищем `35`: в `50` идём влево (35 < 50); в `30` идём вправо (35 > 30); в `40` идём влево (35 < 40); нашли `35`. Четыре шага, по одному на уровень. Ищем `36`: после `35`
правый потомок равен `None`: в дереве этого значения нет. **Вставка** работает так же: ищем значение и там, где поиск выходит за пределы дерева, подвешиваем новый лист.

Стоимость поиска и вставки — это **высота** дерева. Для дерева хорошей формы высота около log₂ n, поэтому получается O(log n), как в двоичном
поиске. Минимум — самый левый узел (идите влево, пока можно), максимум — самый правый.

### Обход дерева

Есть три классических порядка обхода в глубину. Для каждого узла вы делаете три вещи: посетить сам узел, обойти левое поддерево, обойти правое поддерево. Они различаются порядком:

```text
in-order    (left, node, right):   20 30 35 40 50 60 70 80     <- sorted!
pre-order   (node, left, right):   50 30 20 40 35 70 60 80     <- a parent before its children
post-order  (left, right, node):   20 35 40 30 60 80 70 50     <- children before their parent
```

Симметричный обход BST даёт значения **в порядке возрастания**: так BST сортирует. Прямой обход хорош для копирования дерева, обратный — для его удаления.

## 3. BST в Rust

Поместите типы, `insert` и `contains` в `src/trees.rs`:

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

**Что происходит:**

1. `type Tree<T> = Option<Box<Node<T>>>` — **псевдоним типа** (type alias): более короткое имя для длинного типа. `Tree<T>` — это либо пустое дерево (`None`), либо узел (`Some`) с ещё двумя
   `Tree<T>` внутри. Структура рекурсивна, в точности как определение в разделе 1.
2. `BinarySearchTree` хранит корень и счётчик `len`, поэтому `len()` стоит O(1). Мы реализуем **множество**: каждое значение встречается не более одного раза.
3. В `insert` переменная `current` — это *изменяемая ссылка на ячейку* (`Tree<T>`). Она начинается с ячейки корня. Пока ячейка содержит узел, мы сравниваем и переносим ссылку в левую
   или правую ячейку этого узла. Когда путь доходит до пустой ячейки (`None`), цикл заканчивается, и `*current = Some(...)` кладёт в неё новый узел.
   Рекурсия не используется, поэтому высокое дерево здесь не переполнит стек вызовов.
4. `Ordering::Equal` возвращает `false`: дубликаты мы не храним. (`Ord` гарантирует, что `cmp` — согласованный порядок, Урок 07.8.)
5. `contains` — тот же путь с общими ссылками и без изменений. Каждый виток цикла спускается на уровень, поэтому стоимость — O(высота).
6. `Default` (стандартный трейт для «разумного пустого значения») добавлен, потому что Clippy требует его всякий раз, когда есть `new()` без аргументов.

Теперь остальные операции. Добавьте второй блок `impl`:

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

**Что происходит:**

1. `self.root.as_ref()?` превращает `&Option<Box<Node>>` в `Option<&Box<Node>>`, а `?` возвращает `None` для пустого дерева. Затем `min` идёт по ссылкам `left`,
   пока они есть; `max` делает то же справа.
2. `height` и `in_order` **рекурсивны**, как и определение дерева. Высота пустого дерева равна 0; высота узла — 1 плюс большая из высот двух его поддеревьев.
3. `in_order` следует порядку из своих комментариев: левое поддерево, затем узел, затем правое поддерево. Благодаря правилу BST результат отсортирован.
   Время жизни `'a` говорит, что собираемые нами ссылки указывают внутрь дерева.

### Удаление значения

`remove` — самая трудная операция, потому что после исчезновения узла дерево должно сохранить своё правило. Сначала найдите узел, как в `contains`. Затем возможны три случая:

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

Почему наименьшее значение правого поддерева? Оно больше всего слева и не больше ничего справа, поэтому ровно подходит в образовавшийся промежуток. Добавьте вспомогательные функции:

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

**Что происходит:**

1. `let Some(node) = tree else { return false; }` — `let`-`else` (Урок 04.5): для пустого дерева значения здесь нет.
2. `Less` и `Greater` спускаются на уровень вниз рекурсивно с изменяемой ссылкой на левую или правую ячейку.
3. При `Equal` вызовы `node.left.take()` и `node.right.take()` перемещают обоих потомков **из** узла и оставляют на их месте `None`. (`Option::take` — способ вынуть значение
   из места, которое вы только заимствуете; раздел «Частые ошибки» показывает, что будет без него.) Сопоставление пары потомков даёт три случая.
4. Для листа `*tree = None` уничтожает узел. Для одного потомка `*tree = Some(child)` ставит потомка на место узла.
5. Для двух потомков `take_min` удаляет самый левый узел из правого поддерева и возвращает его значение. Это значение перезаписывает `node.value`, а оба поддерева подвешиваются обратно.
6. `take_min` идёт вниз по ссылкам `left`. В самом левом узле она вынимает узел из его ячейки и кладёт туда его собственное правое поддерево (у самого левого узла нет левого потомка).

## 4. Тестирование

Эталон для тестов — `BTreeSet` из стандартной библиотеки, отсортированное множество (раздел 6). Мы применяем одни и те же случайные операции к нашему дереву и к эталону и сравниваем каждый ответ. Добавьте в конец `src/trees.rs`:

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

**Что происходит:**

1. `sample()` строит дерево с картинок. Тесты проверяют числа, которые мы посчитали вручную: 8 узлов, высота 4, минимум 20, максимум 80, симметричный обход отсортирован.
2. `remove_leaf_one_child_and_two_children` удаляет по одному узлу каждого вида. После удаления корня `50` значение `60` (наименьшее в правом поддереве) становится новым корнем.
3. `behaves_like_btreeset` — это **дифференциальное тестирование** (Урок 07.8): 2000 случайных операций на небольшом диапазоне значений (от 0 до 49), чтобы вставки, удаления и поиски часто попадали на существующие значения.
   Операцию выбирает `rng.below(3)` (`Rng` из Урока 07.3). После каждого шага размеры должны совпадать, а в конце обходы должны быть идентичны.
4. Последний тест намеренно показывает проблему из раздела 5.

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

(Отфильтрованы 36 тестов из основного текста Уроков 07.1–07.10; с тестами из упражнений ваше число будет больше.)

## 5. Проблема: форма дерева

Скорость BST зависит от его **высоты**, а высота зависит от **порядка вставки**. Если вставлять уже отсортированные значения, каждое становится правым потомком предыдущего, и
дерево превращается в цепочку: связный список с лишними полями (последний тест выше). Тогда поиск требует O(n). Создайте `examples/e11_tree_heights.rs`:

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

**Как это читать:** после вставки отсортированных значений высота равна `n`; поиск среди 10 000 элементов требует до 10 000 шагов. После случайной вставки высота равна 31, это примерно в 2–2,5
раза больше `log₂ n`: не идеально, но нормально. Реальные данные часто частично отсортированы (отметки времени, идентификаторы), поэтому плохой случай встречается часто.

Лекарство — **самобалансирующееся дерево**: после каждой вставки или удаления оно поворачивает несколько узлов, сохраняя высоту O(log n). Известные виды — AVL-дерево (AVL — инициалы его авторов, Адельсона-Вельского и Ландиса; высоты двух поддеревьев каждого узла в нём отличаются не больше чем на 1) и красно-чёрное дерево. Стандартная библиотека использует
другой вид, **B-дерево**, в котором один узел хранит много значений, что подходит современным процессорам, потому что требует меньше переходов по памяти. Оно
доступно как `BTreeMap` и `BTreeSet`, и именно эти типы нужно использовать в настоящем коде. Сбалансированное дерево самостоятельно вы в этом курсе писать не будете; достаточно знать, зачем оно нужно.

> ⚠️ **Частая ошибка:** несбалансированный BST выглядит нормально в тестах с маленькими или случайными данными и ломается в продакшене на отсортированных данных. Если данные могут приходить по порядку, никогда не используйте простой BST.

## 6. Что использовать в настоящем коде

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

**Что происходит:** `BTreeMap` хранит ключи отсортированными. Итерация даёт ключи по порядку (как наш симметричный обход); `first_key_value` возвращает наименьший ключ; а `range("b".."c")` возвращает
только ключи от `"b"` до `"c"`, не включая `"c"`. Всё это — O(log n) на операцию, и можно не думать о форме.

| Что нужно | Что использовать | Поиск | Упорядоченный обход, диапазоны |
|---|---|---|---|
| самый быстрый поиск, порядок не важен | `HashMap` / `HashSet` | в среднем O(1) | нет |
| отсортированное, со вставками и удалениями | `BTreeMap` / `BTreeSet` | O(log n) | да |
| строится один раз, часто ищется | отсортированный `Vec` + `binary_search` | O(log n) | да (срезы) |
| понять, как работает дерево | собственный `BinarySearchTree` | O(высота) | да |

Деревья встречаются повсюду, помимо поиска: файловые системы, парсеры выражений, устройство индекса базы данных (Этап 17), куча
из следующего урока. BST — простой пример «рекурсивной структуры с рекурсивными функциями».

Все четыре структуры отвечают на поиск за малую долю микросекунды, когда в данных 100 000 чисел; линейному перебору нужно намного больше (Урок 07.10). Поэтому выбирайте по нужным операциям, как в таблице, а не
гадая о скорости. Если вам нужны числа для вашего случая, напишите бенчмарк на `criterion`, как в Уроке 05.12.

## Частые ошибки

**1. Перемещение потомка из заимствованного узла.** В `remove` может возникнуть соблазн сопоставлять потомков напрямую:

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

`node` — всего лишь заимствованная ссылка, поэтому нельзя вынуть потомков и оставить на их месте дыру. Исправление — `Option::take()`, который перемещает значение **и** оставляет на его месте `None`:
`match (node.left.take(), node.right.take())`. (Та же ошибка появляется в списках на `Rc` и во многих других рекурсивных структурах: используйте `take()`.)

**2. BST для упорядоченного ввода.** См. раздел 5: высота становится равной `n`. Используйте `BTreeMap` / `BTreeSet`.

**3. Забытый `len` при удалении.** Если `remove` возвращает `true`, счётчик должен уменьшиться. Тест `behaves_like_btreeset` сравнивает `len()` после каждого шага и ловит такую ошибку.

## Попробуйте сами

1. Добавьте код разделов 3 и 4 шаг за шагом. Запустите `cargo test trees`.
2. Запустите пример `e11_tree_heights`. Замените отсортированный ввод на убывающие числа (`n - i`). Какова теперь высота?
3. В `remove_from` закомментируйте строку `node.right = right;` в случае двух потомков и запустите тесты. Какой тест падает? Верните строку.
4. Запустите `cargo fmt`, `cargo clippy --all-targets` и сделайте коммит: `git add code/p07-algorithms && git commit -m "feat(p07): binary search tree"`.

## Упражнения

### Упражнение 1 — Нарисуйте и обойдите (легко)

Вставьте `8, 3, 10, 1, 6, 14, 4, 7, 13` по очереди в пустой BST. Нарисуйте дерево и запишите его высоту и обходы: симметричный, прямой и обратный. Затем удалите `3` и запишите симметричный обход и нового левого потомка корня.

<details>
<summary>Подсказка</summary>

`8` — корень. `3 < 8` идёт влево; `10 > 8` идёт вправо; `1 < 8`, `1 < 3` идёт влево от `3`; `6` идёт вправо от `3`; и так далее. Для удаления: у `3` два потомка (`1` и `6`); замена — наименьшее значение поддерева с корнем `6`.

</details>

<details>
<summary>Ответ</summary>

```text
            8
          /   \
        3      10
       / \       \
      1   6       14
         / \      /
        4   7    13
```

Высота 4 (путь 8, 3, 6, 4).

```text
in-order:    1 3 4 6 7 8 10 13 14
pre-order:   8 3 1 6 4 7 10 14 13
post-order:  1 4 7 6 3 13 14 10 8
```

Удаление `3` (два потомка): наименьшее значение в правом поддереве (с корнем `6`) — это `4`. Оно перемещается в узел `3`, а старый лист `4` отрезается. Симметричный обход становится `1 4 6 7 8 10 13 14`,
а левый потомок корня теперь `4`, с `1` слева и `6` (с потомком `7`) справа.

</details>

### Упражнение 2 — Проверьте порядок (легко)

Напишите тест, который вставляет много псевдослучайных чисел и проверяет, что `to_vec()` строго возрастает (отсортирован, без равных соседей) и содержит ровно `len()` элементов.

<details>
<summary>Подсказка</summary>

Получайте числа из `Rng`, как в `behaves_like_btreeset`. `to_vec()` даёт `Vec<&i32>`. Сравните каждую пару через `windows(2)`: `pair[0] < pair[1]`.

</details>

<details>
<summary>Ответ</summary>

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

Почему это работает: значения берутся из диапазона от 0 до 499, поэтому многие вставки — дубликаты, которые дерево отвергает; `len()` считает только различные значения. Строго возрастающий симметричный список — видимый результат правила BST.

</details>

### Упражнение 3 — Назовите сложность (средне)

1. Какова стоимость `contains` в BST с `n` узлами и высотой `h`? Чему равна `h` в лучшей и в худшей форме?
2. Сколько стоит построить BST вставкой `n` отсортированных значений и `n` хорошо перемешанных (в среднем)?
3. Какова дополнительная память `in_order` для дерева высотой `h`?

<details>
<summary>Подсказка</summary>

Каждый шаг поиска спускается на один уровень. Для (3) подумайте, сколько вызовов ждёт в стеке вызовов в самой глубокой точке.

</details>

<details>
<summary>Ответ</summary>

1. **O(h)**. В лучшей форме `h ≈ log₂ n`, то есть O(log n); в худшей форме (цепочка) `h = n`, то есть O(n).
2. Отсортированные значения: вставка номер `i` проходит `i` уровней, поэтому всего `1 + 2 + … + n`, **O(n²)**. Перемешанные значения: каждая вставка в среднем стоит около O(log n), поэтому всего **O(n log n)**.
3. **O(h)** на стек вызовов (плюс выходной список). Для цепочки из `n` узлов это O(n) вызовов, и очень длинная цепочка может переполнить стек. Это ещё одна причина балансировать.

</details>

### Упражнение 4 — Это действительно BST? (средне)

Напишите функцию `is_bst<T: Ord>(tree: &Tree<T>, lower: Option<&T>, upper: Option<&T>) -> bool`, которая проверяет правило BST для всего дерева (внутри модуля `tests`, это помощник для тестов). Проверьте её на примере дерева и на собранном вручную дереве, которое **неверно**,
хотя каждый узел корректен по сравнению с собственными потомками.

<details>
<summary>Подсказка</summary>

Сравнивать узел с потомками недостаточно. Передавайте вниз допустимый диапазон: при движении влево текущее значение становится новым `upper`; при движении вправо — новым `lower`. Удобен `Option::is_none_or`: `None` означает «без ограничения».

</details>

<details>
<summary>Ответ</summary>

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

Почему это работает: у узла `15` допустимый диапазон `(5, 10)` (он справа от `5`, слева от `10`), а `15` не меньше 10, поэтому проверка не проходит. Проверка только пары родитель–потомок приняла бы это дерево, а `contains(&15)` пошёл бы в `10` влево и никогда бы его не нашёл.

</details>

## Вопросы для самопроверки

1. Что такое правило BST?

<details>
<summary>Ответ</summary>

Для каждого узла все значения в его левом поддереве меньше значения узла, а все значения в его правом поддереве больше.

</details>

2. Какой порядок обхода посещает BST в отсортированном порядке?

<details>
<summary>Ответ</summary>

Симметричный: левое поддерево, узел, правое поддерево.

</details>

3. Как удаляется узел с двумя потомками?

<details>
<summary>Ответ</summary>

Найти наименьшее значение его правого поддерева, скопировать его в узел и удалить этот наименьший узел из правого поддерева (у него не более одного потомка, поэтому удаление простое).

</details>

4. Почему BST может быть таким же медленным, как связный список?

<details>
<summary>Ответ</summary>

Если значения вставляются в отсортированном порядке, каждый новый узел становится потомком с одной и той же стороны, поэтому дерево — цепочка высотой `n`, а поиск стоит O(n).

</details>

5. Какие стандартные типы стоит использовать вместо собственного BST?

<details>
<summary>Ответ</summary>

`BTreeMap` и `BTreeSet` (отсортированные, O(log n), запросы по диапазону) или `HashMap` / `HashSet`, когда порядок не нужен.

</details>

## Итоги

- Дерево состоит из узлов с корнем наверху; в двоичном дереве у узла не более двух потомков. Высота, лист и поддерево — основные слова.
- BST держит меньшие значения слева, а большие справа. Поиск, вставка, минимум и максимум идут по одному пути от корня: O(высота).
- В Rust дерево — это `Option<Box<Node<T>>>`; ходите циклами и ссылками на ячейки или рекурсией; используйте `Option::take`, чтобы вынимать потомков.
- Симметричный обход BST отсортирован. Прямой и обратный — два других порядка обхода в глубину.
- У удаления узла три случая: лист, один потомок, два потомка (замена наименьшим значением правого поддерева).
- Скорость определяет высота: отсортированный ввод превращает простой BST в список. Самобалансирующиеся деревья это исправляют; в стандартной библиотеке есть `BTreeMap` и `BTreeSet`.
- Тестируйте коллекцию, выполняя одни и те же случайные операции на ней и на проверенном эталоне, например `BTreeSet`.

## Источники и что почитать

- [Rust standard library — std::collections::BTreeMap](https://doc.rust-lang.org/std/collections/struct.BTreeMap.html) — отсортированный словарь: методы, диапазоны и замечание о его устройстве.
- [Rust standard library — std::collections::BTreeSet](https://doc.rust-lang.org/std/collections/struct.BTreeSet.html) — отсортированное множество.
- [Rust standard library — Option::take](https://doc.rust-lang.org/std/option/enum.Option.html#method.take) — перемещение значения из заимствованного `Option`.
- [The Rust Programming Language — Using Box to enable recursive types](https://doc.rust-lang.org/book/ch15-01-box.html) — почему рекурсивному узлу нужен `Box`.
- [Learn Rust With Entirely Too Many Linked Lists](https://rust-unofficial.github.io/too-many-lists/) — бесплатная книга о структурах на указателях в Rust.

---

[🏠 Начало](../START-HERE.md) · [← 07.10 Двоичный поиск](10-binary-search.md) · [📋 Этап 07](README.md) · [07.12 Кучи и очереди с приоритетом →](12-heaps-and-priority-queues.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
