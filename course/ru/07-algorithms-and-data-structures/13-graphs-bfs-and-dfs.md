<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 07.12 Кучи и очереди с приоритетом](12-heaps-and-priority-queues.md) · [📋 Этап 07](README.md) · [07.14 Кратчайшие пути и топологическая сортировка →](14-shortest-paths-and-topological-sort.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/07-algorithms-and-data-structures/13-graphs-bfs-and-dfs.md) <!-- nav -->

# Урок 07.13 — Графы: BFS и DFS

> **Этап 07 — Алгоритмы и структуры данных** · Урок 13 из 20 · ⏱ около 3 часов (2 занятия)

## Что вы узнаете

- Что такое граф, слова для его частей (вершина, ребро, путь, цикл, связность) и как хранить его в виде списка смежности.
- Поиск в ширину (BFS): как он находит путь с наименьшим числом рёбер и почему ему нужна очередь.
- Поиск в глубину (DFS): как он сначала уходит как можно дальше, с рекурсией и с явным стеком, и когда рекурсия опасна.
- Как тестировать код для графов против медленного эталона и что означает стоимость O(V + E).

## Перед началом

- Вы прошли [Урок 07.12 — Кучи и очереди с приоритетом](12-heaps-and-priority-queues.md). Вы знаете очереди и стеки ([Урок 07.5](05-stacks-and-queues.md)) и рекурсию ([Урок 07.7](07-recursion.md)).
- Вы знаете `Vec<Vec<T>>` ([Урок 04.7](../04-structs-enums-collections-errors/07-vectors.md)) и `Option` ([Урок 04.4](../04-structs-enums-collections-errors/04-option-and-match.md)).
- В `code/p07-algorithms` создайте `src/graphs.rs` и добавьте `pub mod graphs;` в `src/lib.rs`.

## Новые слова

| Слово | Значение |
|---|---|
| **граф** (graph) | Множество **вершин** (предметов) и **рёбер** (связей между двумя вершинами). Дороги между городами, друзья в сети, веб-страницы и их ссылки. |
| **вершина** (vertex, мн. ч. vertices) | Один предмет в графе (также называется *узлом*). |
| **ребро** (edge) | Связь между двумя вершинами. |
| **неориентированный / ориентированный** (undirected / directed) | В неориентированном графе ребро работает в обе стороны (дорога с двусторонним движением). В ориентированном графе у ребра есть направление, `a -> b` (улица с односторонним движением, веб-ссылка). |
| **сосед** (neighbour) | Вершина, до которой можно добраться из другой вершины по одному ребру. |
| **путь** (path) | Последовательность вершин, в которой каждая соседняя пара соединена ребром. Его *длина* — число рёбер. |
| **цикл** (cycle) | Путь, который начинается и заканчивается в одной и той же вершине. |
| **связный** (connected) | Неориентированный граф связен, если между любыми двумя вершинами есть путь. **Компонента связности** (connected component) — максимальная группа вершин, достижимых друг из друга. |
| **список смежности** (adjacency list) | Способ хранить граф: для каждой вершины — список её соседей. |
| **матрица смежности** (adjacency matrix) | Другой способ: таблица `V × V` с отметкой в ячейке `(a, b)`, когда есть ребро `a -> b`. |
| **V и E** | Число вершин и число рёбер. Время работы графовых алгоритмов записывают через них: O(V + E). |
| **BFS** (Breadth-First Search, поиск в ширину) | Посетить все вершины на расстоянии 1, затем все на расстоянии 2 и так далее. Использует **очередь**. |
| **DFS** (Depth-First Search, поиск в глубину) | Уйти как можно глубже по одному пути, затем вернуться и попробовать следующий. Использует **стек** (или рекурсию). |
| **двудольный** (bipartite) | Граф, вершины которого можно разбить на две группы так, чтобы каждое ребро соединяло вершины из разных групп. |

## 1. Графы

Граф — самая общая структура «предметы и связи». Список — это граф, в котором у каждой вершины одна следующая; дерево (Урок 07.11) — граф без циклов и с корнем.
Вот небольшой неориентированный граф с 7 вершинами и 7 рёбрами:

```text
   0 --- 1 --- 2
   |     |     |
   3 --- 4 ----+         5 --- 6
```

Рёбра: `0-1`, `1-2`, `0-3`, `1-4`, `2-4`, `3-4` и `5-6`. Путь из `0` в `2` — `0, 1, 2` (длина 2). Цикл — `1, 2, 4, 1`. Граф не связен: у него **две компоненты связности**,
`{0, 1, 2, 3, 4}` и `{5, 6}`, и из `0` в `6` пути нет.

Типичные вопросы и алгоритмы этого урока, которые на них отвечают:

| Вопрос | Чем отвечать |
|---|---|
| Есть ли маршрут из A в B? | BFS или DFS |
| Какой маршрут содержит меньше всего шагов? | BFS |
| Сколько отдельных групп? | BFS или DFS в цикле |
| Можно ли выполнить каждую задачу после тех, что ей нужны? | топологическая сортировка (Урок 07.14) |
| Какой маршрут короче всего, когда у дорог есть длины? | Дейкстра (Урок 07.14) |

## 2. Хранение графа

**Матрица смежности**: таблица `V × V`; ячейка `(a, b)` говорит, существует ли ребро. Проверка одного ребра — O(1), но таблице нужно O(V²) памяти, даже если рёбер мало, а поиск
соседей вершины означает просмотр целой строки.

**Список смежности**: для каждой вершины список её соседей. Ему нужно O(V + E) памяти, а соседи вершины лежат прямо рядом. Это вариант по умолчанию почти для всех графов, которые *разрежены* (рёбер мало по сравнению с V²).
Пример графа в виде списков:

```text
0: [1, 3]
1: [0, 2, 4]
2: [1, 4]
3: [0, 4]
4: [1, 2, 3]
5: [6]
6: [5]
```

В Rust это `Vec<Vec<usize>>`: вершины — числа `0..V`, а `adjacency[v]` — список соседей `v`. Когда вершины — имена (города), каждому имени назначают номер и держат рядом
`Vec<String>` и `HashMap<String, usize>` (проект Урока 07.19 делает именно так).

Поместите тип графа в `src/graphs.rs`:

```rust
//! Graphs as adjacency lists, with breadth-first and depth-first search.

use std::collections::VecDeque;

/// A graph whose vertices are the numbers `0..vertex_count`, stored as adjacency lists.
#[derive(Debug, Clone)]
pub struct Graph {
    adjacency: Vec<Vec<usize>>,
    directed: bool,
    edge_count: usize,
}

impl Graph {
    /// An undirected graph with `vertex_count` vertices and no edges.
    pub fn new(vertex_count: usize) -> Self {
        Graph {
            adjacency: vec![Vec::new(); vertex_count],
            directed: false,
            edge_count: 0,
        }
    }

    /// A directed graph with `vertex_count` vertices and no edges.
    pub fn new_directed(vertex_count: usize) -> Self {
        Graph {
            directed: true,
            ..Graph::new(vertex_count)
        }
    }

    pub fn vertex_count(&self) -> usize {
        self.adjacency.len()
    }

    pub fn edge_count(&self) -> usize {
        self.edge_count
    }

    /// Adds the edge `from -> to` (and `to -> from` in an undirected graph).
    ///
    /// # Panics
    /// Panics if a vertex number is not smaller than `vertex_count()`.
    pub fn add_edge(&mut self, from: usize, to: usize) {
        let n = self.vertex_count();
        assert!(
            from < n && to < n,
            "edge {from} -> {to}: a vertex number must be below {n}"
        );
        self.adjacency[from].push(to);
        if !self.directed {
            self.adjacency[to].push(from);
        }
        self.edge_count += 1;
    }

    /// The vertices that can be reached from `vertex` by one edge.
    pub fn neighbours(&self, vertex: usize) -> &[usize] {
        &self.adjacency[vertex]
    }
}
```

**Что происходит:**

1. `vec![Vec::new(); vertex_count]` создаёт `vertex_count` пустых списков (макрос `vec!` клонирует пустой `Vec`).
2. `new_directed` использует *синтаксис обновления структуры* `..Graph::new(vertex_count)`: все не названные поля берутся из второго значения, а изменяется только `directed`.
3. `add_edge` добавляет `to` в список `from`; в неориентированном графе также `from` в список `to`. `edge_count` считает каждое ребро один раз.
4. `assert!` останавливает программу с понятным сообщением, когда номер вершины неверен, вместо неясного «index out of bounds» позже. Вызывающий код, получающий номера вершин извне программы, должен проверять их заранее (🔒 никогда не доверяйте вводу);
   проект Урока 07.19 возвращает ошибку вместо паники.
5. `neighbours` возвращает срез `&[usize]`: вызывающий код может читать список, но не менять его.

## 3. Поиск в ширину (BFS)

BFS исследует граф **волнами**. Сначала стартовая вершина, затем все её соседи (расстояние 1), затем все их новые соседи (расстояние 2) и так далее. В первый раз, когда BFS достигает вершины, он использовал
**наименьшее возможное число рёбер**, поэтому BFS находит кратчайшие пути в графах, где все рёбра считаются одинаково.

Чтобы сохранить порядок «самый старый первым», мы используем **очередь** (Урок 07.5): вершины ждут в очереди, и мы всегда берём ту, что ждёт дольше всех. Чтобы не ходить по кругу, мы запоминаем, какие вершины уже
виденные. BFS на примере графа из `0` (начало очереди слева):

```text
start:    queue [0]            dist: 0:0
take 0:   new 1, 3             queue [1, 3]        dist: 1:1  3:1
take 1:   0 seen; new 2, 4     queue [3, 2, 4]     dist: 2:2  4:2
take 3:   0, 4 seen            queue [2, 4]
take 2:   1, 4 seen            queue [4]
take 4:   1, 2, 3 seen         queue []            done: 5 and 6 were never reached
```

Добавьте в `src/graphs.rs`:

```rust
/// What breadth-first search learns about every vertex.
#[derive(Debug, PartialEq, Eq)]
pub struct BfsResult {
    /// Number of edges on the shortest path from the start, or `None` if unreachable.
    pub distance: Vec<Option<usize>>,
    /// The vertex from which we first reached this vertex, or `None` for the start and unreachable vertices.
    pub previous: Vec<Option<usize>>,
}

/// Breadth-first search from `start`: visits vertices in order of their distance from `start`.
pub fn bfs(graph: &Graph, start: usize) -> BfsResult {
    let n = graph.vertex_count();
    let mut distance = vec![None; n];
    let mut previous = vec![None; n];
    let mut queue = VecDeque::new();

    distance[start] = Some(0); // mark when we put a vertex in the queue, not when we take it out
    queue.push_back(start);
    while let Some(vertex) = queue.pop_front() {
        let next_distance = distance[vertex].expect("a queued vertex has a distance") + 1;
        for &neighbour in graph.neighbours(vertex) {
            if distance[neighbour].is_none() {
                distance[neighbour] = Some(next_distance);
                previous[neighbour] = Some(vertex);
                queue.push_back(neighbour);
            }
        }
    }
    BfsResult { distance, previous }
}

/// A path with the fewest edges from `from` to `to`, including both ends, or `None` if there is none.
pub fn shortest_path(graph: &Graph, from: usize, to: usize) -> Option<Vec<usize>> {
    let result = bfs(graph, from);
    result.distance[to]?; // unreachable: give up
    let mut path = vec![to];
    let mut current = to;
    while let Some(before) = result.previous[current] {
        path.push(before);
        current = before;
    }
    path.reverse();
    Some(path)
}
```

**Что происходит:**

1. `distance[v]` равно `None`, пока `v` не видели, и `Some(d)`, когда оно попало в очередь. Оно выполняет две задачи: служит флагом «виденная» и является ответом. `previous[v]` запоминает, кто первым нашёл `v`.
2. Старт получает расстояние 0 и попадает в очередь. Цикл берёт переднюю вершину и смотрит на каждого соседа. Сосед, у которого ещё нет расстояния, — новый: он получает `distance + 1`, `previous` и место в **конце** очереди.
3. Мы помечаем вершину **когда она попадает в очередь**, а не когда выходит. Тогда ни одна вершина не окажется в очереди дважды. (Раздел «Частые ошибки» объясняет, что идёт не так иначе.)
4. `shortest_path` запускает BFS и идёт по `previous` от цели назад к старту, строя путь в обратном порядке, затем разворачивает его. `result.distance[to]?` рано возвращает `None`, когда `to` не была достигнута.
   Цепочка значений `previous` — это дерево (*дерево BFS*) с корнем в старте.

**Стоимость.** Каждая вершина попадает в очередь не более одного раза: V операций с очередью. Каждый список соседей читается один раз, поэтому каждое ребро просматривается один раз (дважды в неориентированном графе): E шагов. Всего **O(V + E)** времени, **O(V)** дополнительной памяти.

## 4. Поиск в глубину (DFS)

DFS идёт другим путём: из вершины он берёт первого невиденного соседа и продолжает оттуда, как можно глубже. Когда он упирается в тупик, он делает шаг **назад** и пробует следующего соседа. Это ходок по лабиринту, который
всегда выбирает первый неисследованный коридор и в тупике возвращается к последней развилке. Ожидающие развилки хранятся в **стеке**, который в рекурсивной функции — это стек вызовов (Урок 07.7).

DFS из `0` на примере графа. Каждая строка — вершина, в которую мы входим, и сосед, которого она выбирает дальше:

```text
visit 0:  neighbours [1, 3]       -> go to 1
visit 1:  neighbours [0, 2, 4]    -> 0 seen, go to 2
visit 2:  neighbours [1, 4]       -> 1 seen, go to 4
visit 4:  neighbours [1, 2, 3]    -> 1, 2 seen, go to 3
visit 3:  neighbours [0, 4]       -> both seen: step back
(back at 4, 1 and 0: nothing new)          order: 0 1 2 4 3
```

Добавьте рекурсивную версию:

```rust
/// Depth-first search with recursion. Returns the vertices in the order they are first visited.
pub fn dfs_recursive(graph: &Graph, start: usize) -> Vec<usize> {
    let mut visited = vec![false; graph.vertex_count()];
    let mut order = Vec::new();
    visit(graph, start, &mut visited, &mut order);
    order
}

fn visit(graph: &Graph, vertex: usize, visited: &mut [bool], order: &mut Vec<usize>) {
    visited[vertex] = true;
    order.push(vertex);
    for &neighbour in graph.neighbours(vertex) {
        if !visited[neighbour] {
            visit(graph, neighbour, visited, order);
        }
    }
}
```

**Что происходит:** `visit` помечает вершину, записывает её и вызывает себя для каждого соседа, который ещё не посещён. Флаги `visited` не дают обходу ходить по кругу вдоль цикла. Когда цикл по соседям
заканчивается, функция возвращается, и это «шаг назад». Глубина рекурсии равна длине текущего пути, до V.

Эта глубина — слабое место. Граф в виде длинной цепочки (100 000 вершин и больше) делает рекурсию такой глубокой, что стек вызовов переполняется. Версия с **явным стеком** (`Vec` в куче) такого предела не имеет:

```rust
/// Depth-first search with an explicit stack: the same order, but no deep recursion.
pub fn dfs_iterative(graph: &Graph, start: usize) -> Vec<usize> {
    let mut visited = vec![false; graph.vertex_count()];
    let mut order = Vec::new();
    let mut stack = vec![start];
    while let Some(vertex) = stack.pop() {
        if visited[vertex] {
            continue; // it was pushed twice before we reached it
        }
        visited[vertex] = true;
        order.push(vertex);
        // Push in reverse, so that the first neighbour is popped (visited) first.
        for &neighbour in graph.neighbours(vertex).iter().rev() {
            if !visited[neighbour] {
                stack.push(neighbour);
            }
        }
    }
    order
}
```

**Что происходит:**

1. Стек хранит вершины, которые ждут посещения; `pop` берёт **самую новую**, это противоположность очереди в BFS. Это единственное изменение превращает поиск в ширину в поиск в глубину.
2. В отличие от BFS, мы проверяем `visited`, когда **достаём** вершину, потому что вершина может быть добавлена в стек несколько раз до посещения (от разных соседей). Во второй раз `continue` её пропускает.
3. Стек выдаёт последнего добавленного соседа первым, поэтому мы добавляем в обратном порядке; тогда первый сосед посещается первым, и порядок в точности совпадает с рекурсивной версией (тест это докажет).

Чтобы увидеть разницу, создайте `examples/e13_long_chain.rs`:

```rust
use p07_algorithms::graphs::{Graph, dfs_iterative, dfs_recursive};

fn main() {
    let n = 200_000;
    let mut graph = Graph::new(n);
    for v in 1..n {
        graph.add_edge(v - 1, v);
    }
    println!("iterative: {} vertices", dfs_iterative(&graph, 0).len());
    println!("recursive: {} vertices", dfs_recursive(&graph, 0).len());
}
```

```bash
cargo run --example e13_long_chain
```

```text
iterative: 200000 vertices

thread 'main' (5429589) has overflowed its stack
fatal runtime error: stack overflow, aborting
```

Явный стек справляется с 200 000 вершин; рекурсия падает. (Число в скобках — идентификатор потока, он различается при каждом запуске; релизная сборка тоже падает.) Для графов, построенных из недоверенных данных, всегда предпочитайте BFS или итеративный DFS.

> 🔒 **Безопасность:** программу, которая рекурсивно заходит по разу на каждую вершину входного графа, может обрушить любой, кто пришлёт длинную цепочку. Ограничивайте глубину или используйте явный стек, когда вход недоверенный.

**Стоимость.** Та же, что у BFS: каждая вершина посещается один раз, и каждое ребро просматривается один раз, **O(V + E)** времени; память — **O(V)** (флаги и стек).

### BFS или DFS?

| | BFS | DFS |
|---|---|---|
| Структура данных | очередь (первым пришёл — первым вышел) | стек (последним пришёл — первым вышел) или рекурсия |
| Посещает | волна за волной, ближайшие первыми | один глубокий путь за раз |
| Находит | путь с **наименьшим числом рёбер** | *какой-то* путь (необязательно кратчайший) |
| Типичное применение | кратчайший маршрут по шагам, уровни расстояния, лабиринты | компоненты связности, поиск циклов, упорядочение задач (Урок 07.14) |

Ещё одно применение того же обхода: цикл по всем вершинам, который начинает новый поиск в каждой непосещённой, считает **компоненты связности**. Добавьте:

```rust
/// The number of connected components of an undirected graph
/// (groups of vertices that can reach each other).
pub fn count_components(graph: &Graph) -> usize {
    let mut seen = vec![false; graph.vertex_count()];
    let mut components = 0;
    for start in 0..graph.vertex_count() {
        if seen[start] {
            continue; // already part of an earlier component
        }
        components += 1;
        seen[start] = true;
        let mut stack = vec![start];
        while let Some(vertex) = stack.pop() {
            for &neighbour in graph.neighbours(vertex) {
                if !seen[neighbour] {
                    seen[neighbour] = true;
                    stack.push(neighbour);
                }
            }
        }
    }
    components
}
```

Каждый поиск начинается с вершины, до которой не дошёл ни один предыдущий поиск, поэтому каждый поиск находит одну новую компоненту. Флаги `seen` общие, так что общая стоимость по-прежнему O(V + E).

## 5. Тестирование кода для графов

У теста графов две трудности: нужны входные данные и нужен правильный ответ. Для входов мы генерируем случайные графы. Для ответа мы пишем **медленный, заведомо верный эталон**: снова и снова уточняем расстояния по всем рёбрам, пока ничего не меняется.
Он работает за O(V · E), слишком медленно для настоящего использования, но так прост, что ошибка в нём маловероятна. Добавьте в конец `src/graphs.rs`:

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use crate::testutil::Rng;

    /// The graph of the lesson:  0 - 1 - 2     5 - 6 (a separate pair)
    ///                           |   |   |
    ///                           3 - 4 --+
    fn sample() -> Graph {
        let mut graph = Graph::new(7);
        for (a, b) in [(0, 1), (1, 2), (0, 3), (1, 4), (2, 4), (3, 4), (5, 6)] {
            graph.add_edge(a, b);
        }
        graph
    }

    /// A pseudo-random graph (the `Rng` of Lesson 07.3).
    fn random_graph(seed: u64, vertices: usize, edges: usize, directed: bool) -> Graph {
        let mut rng = Rng::new(seed);
        let mut graph = if directed {
            Graph::new_directed(vertices)
        } else {
            Graph::new(vertices)
        };
        for _ in 0..edges {
            let from = rng.below(vertices);
            let to = rng.below(vertices);
            graph.add_edge(from, to);
        }
        graph
    }

    /// A slow but obviously correct reference: relax all edges until nothing changes.
    fn reference_distances(graph: &Graph, start: usize) -> Vec<Option<usize>> {
        let mut distance = vec![None; graph.vertex_count()];
        distance[start] = Some(0);
        loop {
            let mut changed = false;
            for vertex in 0..graph.vertex_count() {
                if let Some(d) = distance[vertex] {
                    for &next in graph.neighbours(vertex) {
                        if distance[next].is_none_or(|old| d + 1 < old) {
                            distance[next] = Some(d + 1);
                            changed = true;
                        }
                    }
                }
            }
            if !changed {
                return distance;
            }
        }
    }

    #[test]
    fn counts_vertices_and_edges() {
        let graph = sample();
        assert_eq!(graph.vertex_count(), 7);
        assert_eq!(graph.edge_count(), 7);
        assert_eq!(graph.neighbours(1), [0, 2, 4]);
    }

    #[test]
    fn bfs_distances_and_paths() {
        let graph = sample();
        let result = bfs(&graph, 0);
        assert_eq!(
            result.distance,
            [Some(0), Some(1), Some(2), Some(1), Some(2), None, None]
        );
        assert_eq!(shortest_path(&graph, 0, 2), Some(vec![0, 1, 2]));
        assert_eq!(shortest_path(&graph, 3, 3), Some(vec![3]));
        assert_eq!(shortest_path(&graph, 0, 6), None);
    }

    #[test]
    fn dfs_orders_agree() {
        let graph = sample();
        assert_eq!(dfs_recursive(&graph, 0), [0, 1, 2, 4, 3]);
        assert_eq!(dfs_iterative(&graph, 0), [0, 1, 2, 4, 3]);
        for seed in 1..=20 {
            let graph = random_graph(seed, 30, 40, seed % 2 == 0);
            assert_eq!(
                dfs_recursive(&graph, 0),
                dfs_iterative(&graph, 0),
                "seed {seed}"
            );
        }
    }

    #[test]
    fn bfs_matches_the_reference() {
        for seed in 1..=30 {
            for directed in [false, true] {
                let graph = random_graph(seed, 25, 35, directed);
                for start in 0..graph.vertex_count() {
                    assert_eq!(
                        bfs(&graph, start).distance,
                        reference_distances(&graph, start),
                        "seed {seed}, directed {directed}, start {start}"
                    );
                }
            }
        }
    }

    #[test]
    fn components() {
        assert_eq!(count_components(&sample()), 2);
        assert_eq!(count_components(&Graph::new(4)), 4);
        assert_eq!(count_components(&Graph::new(0)), 0);
    }

    #[test]
    fn a_long_chain_is_fine_for_bfs_and_iterative_dfs() {
        let n = 100_000;
        let mut graph = Graph::new(n);
        for v in 1..n {
            graph.add_edge(v - 1, v);
        }
        assert_eq!(bfs(&graph, 0).distance[n - 1], Some(n - 1));
        assert_eq!(dfs_iterative(&graph, 0).len(), n);
    }

    #[test]
    #[should_panic(expected = "a vertex number must be below 5")]
    fn add_edge_checks_vertex_numbers() {
        Graph::new(5).add_edge(0, 7);
    }
}
```

**Что происходит:**

1. `sample()` — граф с картинок, поэтому ожидаемые значения в тестах — те, что мы вывели вручную (расстояния 0, 1, 2, 1, 2 и порядок DFS `0 1 2 4 3`).
2. `random_graph` создаёт графы со случайными рёбрами, включая повторяющиеся рёбра и петли из вершины в саму себя. Такие «грязные» входы хороши для тестов. `rng.below(vertices)` даёт номер вершины от 0 до `vertices - 1`.
3. `bfs_matches_the_reference` сравнивает BFS с медленным эталоном для 30 зёрен × 2 видов (неориентированный и ориентированный) × всех 25 стартовых вершин: 1500 проверок. `dfs_orders_agree` доказывает, что обе версии DFS посещают в одном порядке.
4. Тест с длинной цепочкой показывает, что BFS и итеративный DFS переживают 100 000 вершин в потоке теста. `#[should_panic(expected = "...")]` проверяет, что `add_edge` отвергает неверный номер вершины с нашим сообщением.

```bash
cd ~/projects/rust/code/p07-algorithms
cargo test graphs
```

```text
running 7 tests
test graphs::tests::components ... ok
test graphs::tests::counts_vertices_and_edges ... ok
test graphs::tests::bfs_distances_and_paths ... ok
test graphs::tests::dfs_orders_agree ... ok
test graphs::tests::add_edge_checks_vertex_numbers - should panic ... ok
test graphs::tests::bfs_matches_the_reference ... ok
test graphs::tests::a_long_chain_is_fine_for_bfs_and_iterative_dfs ... ok

test result: ok. 7 passed; 0 failed; 0 ignored; 0 measured; 46 filtered out; finished in 0.01s
```

(Отфильтрованы 46 тестов из основного текста Уроков 07.1–07.12; с тестами из упражнений ваше число будет больше.)

> 🧪 **Тестирование:** для алгоритмов с простым определением («кратчайший путь — наименьшее число рёбер») сначала напишите самое медленное заведомо верное решение и используйте его как судью. Оно стоит десять строк и
> находит ошибки в хитрой версии.

## 6. Что использовать в настоящем коде

В стандартной библиотеке нет типа графа; `Vec<Vec<usize>>` (как здесь) и `VecDeque` — стандартные инструменты, а `HashMap<K, Vec<K>>` — когда вершины не числа. Для больших или сложных графов используйте крейт `petgraph`
(версия 0.8 на момент написания), который предлагает типы графов и алгоритмы этого этапа. Знайте, как работают BFS и DFS, потому что вы будете использовать их напрямую в каждой задаче, где есть «шаги», «соседи», «области» или «зависимости».

## Частые ошибки

**1. Изменение графа во время цикла по его соседям.** Список, возвращаемый `neighbours`, заимствован из графа:

```rust
use p07_algorithms::graphs::Graph;

pub fn double_the_edges_of_zero(graph: &mut Graph) {
    for &neighbour in graph.neighbours(0) {
        graph.add_edge(0, neighbour);
    }
}
```

```text
error[E0502]: cannot borrow `*graph` as mutable because it is also borrowed as immutable
 --> src/lib.rs:5:9
  |
4 |       for &neighbour in graph.neighbours(0) {
  |       -                 -------------------
  |       |                 |
  |       |                 immutable borrow occurs here
  |  _____|                 immutable borrow later used here
  | |
5 | |         graph.add_edge(0, neighbour);
  | |         ^^^^^^^^^^^^^^^^^^^^^^^^^^^^ mutable borrow occurs here
6 | |     }
  | |_____- this for loop borrows `*graph` immutably, preventing mutation within its body
```

Цикл держит общее заимствование списка, пока `add_edge` хочет его изменить (а если бы мог, список мог бы вырасти и переехать в памяти, пока мы его читаем). Исправление: сначала скопируйте соседей: `let neighbours = graph.neighbours(0).to_vec();`, затем идите циклом по копии.

**2. Нет пометок «виденная».** Без `distance[neighbour].is_none()` (или `visited`) граф с циклом отправляет поиск по кругу цикла навечно, а очередь или стек растут без конца. Любому поиску по графу нужно множество «виденных».

**3. Слишком поздняя пометка вершины в BFS.** Если BFS помечает вершину только когда она *выходит* из очереди, одна и та же вершина может быть добавлена в очередь много раз (по разу от каждого соседа, который увидел её первым). Результат всё равно верен,
но очередь может вырасти намного больше V, а время вырастет. Помечайте в момент `push_back`. (DFS с явным стеком устроен иначе: там дубликаты обрабатываются проверкой после `pop`.)

**4. Рекурсия на недоверенных или огромных графах.** Раздел 4: используйте явный стек.

## Попробуйте сами

1. Добавьте код разделов 2–5 шаг за шагом. Запустите `cargo test graphs`.
2. Запустите `cargo run --example e13_long_chain`. Измените `n` на `10_000`. Переживает ли рекурсивная версия? Найдите размер, с которого она начинает падать на вашем компьютере.
3. В `bfs` замените `queue.pop_front()` на `queue.pop_back()` (это превращает очередь в стек). Запустите тесты. Какие тесты падают? Что вы построили вместо BFS?
4. Запустите `cargo fmt`, `cargo clippy --all-targets` и сделайте коммит: `git add code/p07-algorithms && git commit -m "feat(p07): graphs, BFS and DFS"`.

## Упражнения

### Упражнение 1 — BFS вручную (легко)

Для примера графа выполните BFS из вершины `4` вручную. Запишите очередь после каждого шага и итоговый список расстояний для вершин `0`–`6`. В каком порядке вершины достаются из очереди?

<details>
<summary>Подсказка</summary>

Соседи `4` в порядке добавления рёбер — `[1, 2, 3]`; соседи `1` — `[0, 2, 4]`. Вершины `5` и `6` с `4` не связаны.

</details>

<details>
<summary>Ответ</summary>

```text
start:   queue [4]           4:0
take 4:  new 1, 2, 3         queue [1, 2, 3]     1:1 2:1 3:1
take 1:  new 0               queue [2, 3, 0]     0:2
take 2:  nothing new         queue [3, 0]
take 3:  nothing new         queue [0]
take 0:  nothing new         queue []
```

Порядок выхода из очереди: `4, 1, 2, 3, 0`. Расстояния для вершин 0–6: `[2, 1, 1, 1, 0, нет, нет]`.

</details>

### Упражнение 2 — Тест свойства (легко)

Напишите тест: в неориентированном случайном графе для каждого ребра `u - v` либо оба конца достижимы из вершины 0 и их расстояния BFS отличаются не более чем на 1, либо ни один не достижим.

<details>
<summary>Подсказка</summary>

Вычислите `distance` один раз. Для каждого `u` и каждого `v` из `graph.neighbours(u)` сопоставьте `match (distance[u], distance[v])`: оба `Some`, оба `None` или смешанный случай, который не должен происходить никогда. `usize::abs_diff` даёт разность без переполнения вниз.

</details>

<details>
<summary>Ответ</summary>

```rust
    #[test]
    fn neighbours_differ_by_at_most_one() {
        for seed in 1..=20 {
            let graph = random_graph(seed, 30, 40, false);
            let distance = bfs(&graph, 0).distance;
            for u in 0..graph.vertex_count() {
                for &v in graph.neighbours(u) {
                    match (distance[u], distance[v]) {
                        (Some(a), Some(b)) => assert!(a.abs_diff(b) <= 1),
                        (None, None) => {}
                        _ => panic!("an edge joins a reachable and an unreachable vertex"),
                    }
                }
            }
        }
    }
```

Почему это работает: если `u` на расстоянии `a`, его сосед `v` на расстоянии не более `a + 1` (через `u`), и симметрично `u` не более `b + 1`, поэтому расстояния отличаются не более чем на 1. А если один конец ребра достижим, то достижим и другой.
Этот тест проверяет *свойство* BFS, не зная правильного ответа.

</details>

### Упражнение 3 — Назовите сложность (средне)

1. Каковы время и дополнительная память BFS на графе с V вершинами и E рёбрами, хранящемся в виде списка смежности? Что меняется с матрицей смежности?
2. Насколько глубока в худшем случае рекурсия `dfs_recursive` и каков наибольший размер очереди в `bfs` для графа-звезды (одна центральная вершина, соединённая с V − 1 другими)?
3. Почему проверка `visited[vertex]` после `pop` не делает `dfs_iterative` медленнее O(V + E)?

<details>
<summary>Подсказка</summary>

Для (2): сколько вершин ждёт в стеке вызовов в цепочке? Сколько вершин центр кладёт в очередь сразу в звезде? Для (3): сколько раз вершина может быть добавлена в стек самое большее?

</details>

<details>
<summary>Ответ</summary>

1. Список смежности: **O(V + E)** времени, **O(V)** дополнительной памяти. С матрицей смежности поиск соседей вершины просматривает строку из V ячеек, поэтому BFS стоит **O(V²)** независимо от E.
2. Глубина рекурсии до **V** (цепочка). Очередь BFS на звезде хранит до **V − 1** вершин одновременно (все листья ждут после извлечения центра). Память BFS в худшем случае O(V).
3. Вершина добавляется в стек не более одного раза на каждое ведущее к ней ребро, поэтому общее число добавлений не больше E; число извлечений такое же. Пропуск уже посещённой вершины стоит O(1). Всего **O(V + E)**.

</details>

### Упражнение 4 — Уровни (средне)

Напишите `levels(graph: &Graph, start: usize) -> Vec<Vec<usize>>`, которая группирует достижимые вершины по расстоянию BFS: элемент 0 — это `[start]`, элемент 1 содержит вершины на расстоянии 1 и так далее. Проверьте её на примере графа из вершины 0.

<details>
<summary>Подсказка</summary>

Вызовите `bfs` и пройдите по `distance` через `enumerate()`. Для вершины с расстоянием `Some(d)` убедитесь, что в `levels` не меньше `d + 1` групп (`levels.resize(d + 1, Vec::new())`), и добавьте вершину в `levels[d]`.

</details>

<details>
<summary>Ответ</summary>

```rust
/// The vertices grouped by their distance from `start`: level 0 is `start` itself.
pub fn levels(graph: &Graph, start: usize) -> Vec<Vec<usize>> {
    let distance = bfs(graph, start).distance;
    let mut levels: Vec<Vec<usize>> = Vec::new();
    for (vertex, d) in distance.iter().enumerate() {
        if let Some(d) = *d {
            if levels.len() <= d {
                levels.resize(d + 1, Vec::new());
            }
            levels[d].push(vertex);
        }
    }
    levels
}
```

Тест внутри модуля `tests`:

```rust
    #[test]
    fn bfs_from_vertex_four() {
        let graph = sample();
        let result = bfs(&graph, 4);
        assert_eq!(
            result.distance,
            [Some(2), Some(1), Some(1), Some(1), Some(0), None, None]
        );
        assert_eq!(levels(&graph, 0), [vec![0], vec![1, 3], vec![2, 4]]);
    }
```

Почему это работает: BFS даёт каждой достижимой вершине расстояние, а расстояния в BFS идут подряд (если есть вершина на расстоянии `d > 0`, есть и на `d − 1`), поэтому `resize` никогда не оставляет пустого пропуска. Тест заодно повторяет результат Упражнения 1.

</details>

### Упражнение 5 — Две группы (сложно)

Граф *двудольный*, если его вершины можно разбить на две группы так, чтобы каждое ребро соединяло вершины из разных групп (например, «команды», в которых ни у кого нет соперника в своей команде). Напишите `is_bipartite(graph: &Graph) -> bool` через BFS: дайте стартовой вершине один цвет, её соседям — другой и так далее; если ребро соединяет две вершины одного цвета, ответ `false`. Обработайте несколько компонент. Проверьте её на циклах (чётной и нечётной длины), на петле и сравните с перебором всех раскрасок для небольших случайных графов.

<details>
<summary>Подсказка</summary>

Храните `colour: Vec<Option<bool>>`. Для каждой нераскрашенной вершины начинайте новый BFS с цвета `false`. Увидев соседа без цвета, дайте ему `!цвет_текущей` и поставьте в очередь; если у него тот же цвет, что у текущей вершины, верните `false`. Перебор: для каждой битовой маски из `n` бит проверьте, что у каждого ребра `(v, w)` биты `v` и `w` различны.

</details>

<details>
<summary>Ответ</summary>

```rust
/// Can the vertices be split into two groups so that every edge joins different groups?
pub fn is_bipartite(graph: &Graph) -> bool {
    let mut colour: Vec<Option<bool>> = vec![None; graph.vertex_count()];
    for start in 0..graph.vertex_count() {
        if colour[start].is_some() {
            continue;
        }
        colour[start] = Some(false);
        let mut queue = VecDeque::from([start]);
        while let Some(vertex) = queue.pop_front() {
            let mine = colour[vertex].expect("a queued vertex has a colour");
            for &neighbour in graph.neighbours(vertex) {
                match colour[neighbour] {
                    None => {
                        colour[neighbour] = Some(!mine);
                        queue.push_back(neighbour);
                    }
                    Some(theirs) if theirs == mine => return false,
                    Some(_) => {}
                }
            }
        }
    }
    true
}
```

Тесты внутри модуля `tests`:

```rust
    fn cycle(n: usize) -> Graph {
        let mut graph = Graph::new(n);
        for v in 0..n {
            graph.add_edge(v, (v + 1) % n);
        }
        graph
    }

    #[test]
    fn bipartite_examples() {
        assert!(is_bipartite(&cycle(4)));
        assert!(!is_bipartite(&cycle(3)));
        assert!(!is_bipartite(&cycle(5)));
        assert!(is_bipartite(&Graph::new(0)));
        assert!(!is_bipartite(&sample())); // 1, 2 and 4 form a triangle
        let mut loop_graph = Graph::new(2);
        loop_graph.add_edge(1, 1);
        assert!(!is_bipartite(&loop_graph));
    }

    #[test]
    fn bipartite_matches_brute_force() {
        for seed in 1..=40 {
            let graph = random_graph(seed, 7, 6 + (seed % 4) as usize, false);
            let n = graph.vertex_count();
            let brute = (0..1u32 << n).any(|mask| {
                (0..n).all(|v| {
                    graph
                        .neighbours(v)
                        .iter()
                        .all(|&w| (mask >> v) & 1 != (mask >> w) & 1)
                })
            });
            assert_eq!(is_bipartite(&graph), brute, "seed {seed}");
        }
    }
```

Почему это работает: в двудольном графе цвета вдоль BFS должны чередоваться, поэтому цвет каждой вершины определяется расстоянием от старта её компоненты (чётное расстояние — один цвет, нечётное — другой). Ребро между двумя вершинами одного цвета замыкает нечётный цикл, который нельзя разбить на две группы. Время — **O(V + E)**. Вершина с ребром в саму себя — собственный сосед того же цвета, поэтому тест с петлёй возвращает `false`.

</details>

## Вопросы для самопроверки

1. В чём разница между ориентированным и неориентированным графом?

<details>
<summary>Ответ</summary>

В неориентированном графе ребро работает в обе стороны; в ориентированном ребро идёт только от одной вершины к другой. В списке смежности неориентированное ребро появляется в двух списках, а ориентированное — в одном.

</details>

2. Какую структуру данных использует BFS и какую — DFS?

<details>
<summary>Ответ</summary>

BFS использует очередь (первым пришёл — первым вышел), DFS — стек (последним пришёл — первым вышел) или стек вызовов через рекурсию.

</details>

3. Почему BFS находит путь с наименьшим числом рёбер?

<details>
<summary>Ответ</summary>

Он посещает вершины волнами: все на расстоянии 1, затем все на расстоянии 2 и так далее. В первый раз, когда он достигает вершины, более короткого пути существовать не может, потому что все более короткие были бы испробованы раньше.

</details>

4. Почему рекурсивный DFS может упасть на большом графе и как это исправить?

<details>
<summary>Ответ</summary>

Каждый рекурсивный вызов занимает место в стеке вызовов, а глубина может достигать V (для длинной цепочки). Стек вызовов невелик, поэтому он переполняется. Используйте DFS с явным стеком, хранящимся в `Vec`.

</details>

## Итоги

- Граф — это вершины и рёбра; рёбра могут быть ориентированными, а путь, цикл и компонента связности — основные слова.
- Обычное хранение — список смежности `Vec<Vec<usize>>`: O(V + E) памяти и быстрый доступ к соседям. Матрице нужно O(V²).
- BFS использует очередь, посещает волнами и находит путь с наименьшим числом рёбер. Помечайте вершины при попадании в очередь и храните `distance` и `previous`, чтобы восстановить путь.
- DFS использует стек или рекурсию и исследует по одному глубокому пути за раз. Рекурсивная версия может переполнить стек вызовов на длинной цепочке; итеративная — нет.
- Оба работают за O(V + E) времени и O(V) памяти. Цикл, начинающий поиск в каждой непосещённой вершине, считает компоненты связности.
- Тестируйте код для графов случайными графами и медленным заведомо верным эталоном, а также свойствами вроде «соседи отличаются не более чем на один уровень».

## Источники и что почитать

- [Rust standard library — std::collections::VecDeque](https://doc.rust-lang.org/std/collections/struct.VecDeque.html) — очередь, которую использует BFS.
- [Rust standard library — Option::is_none_or](https://doc.rust-lang.org/std/option/enum.Option.html#method.is_none_or) — помощник, использованный в эталонной реализации.
- [The Rust Programming Language — Storing Lists of Values with Vectors](https://doc.rust-lang.org/book/ch08-01-vectors.html) — глава о векторах, основа любого списка смежности.
- [petgraph on docs.rs](https://docs.rs/petgraph/latest/petgraph/) — библиотека графов для промышленного кода, с BFS, DFS, Дейкстрой и многим другим.

---

[🏠 Начало](../START-HERE.md) · [← 07.12 Кучи и очереди с приоритетом](12-heaps-and-priority-queues.md) · [📋 Этап 07](README.md) · [07.14 Кратчайшие пути и топологическая сортировка →](14-shortest-paths-and-topological-sort.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
