<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 07.13 Графы: BFS и DFS](13-graphs-bfs-and-dfs.md) · [📋 Этап 07](README.md) · [07.15 Два указателя и скользящее окно →](15-two-pointers-and-sliding-window.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/07-algorithms-and-data-structures/14-shortest-paths-and-topological-sort.md) <!-- nav -->

# Урок 07.14 — Кратчайшие пути и топологическая сортировка

> **Этап 07 — Алгоритмы и структуры данных** · Урок 14 из 20 · ⏱ около 3 часов (2 занятия)

## Что вы узнаете

- Почему BFS (Breadth-First Search, поиск в ширину из Урока 07.13) недостаточно, когда у рёбер разная длина, и что такое взвешенный граф.
- Алгоритм Дейкстры: идею, код с `BinaryHeap`, почему нужны неотрицательные веса и его стоимость O((V + E) log V).
- Топологическую сортировку: как расставить задачи в порядке, соблюдающем «A нужно сделать до B», и как обнаружить цикл.
- Как тестировать эти алгоритмы против медленных эталонов и когда обращаться к библиотеке.

## Перед началом

- Вы прошли [Урок 07.13 — Графы: BFS и DFS](13-graphs-bfs-and-dfs.md) и [Урок 07.12 — Кучи и очереди с приоритетом](12-heaps-and-priority-queues.md).
- Вы знаете `Reverse` и `BinaryHeap` из Урока 07.12 и тип `Graph` с `bfs` из Урока 07.13.
- В `code/p07-algorithms` создайте два файла, `src/shortest_paths.rs` и `src/topological.rs`, и добавьте `pub mod shortest_paths;` и `pub mod topological;` в `src/lib.rs`.

## Новые слова

| Слово | Значение |
|---|---|
| **взвешенный граф** (weighted graph) | Граф, у рёбер которого есть число (*вес*), например длина дороги. |
| **кратчайший путь** (shortest path) | Путь, у которого **суммарный вес** наименьший. В невзвешенном графе — путь с наименьшим числом рёбер. |
| **алгоритм Дейкстры** (Dijkstra's algorithm) | Алгоритм, находящий кратчайшие пути от одной стартовой вершины ко всем остальным, когда нет отрицательных весов. Назван по имени Эдсгера Дейкстры. |
| **релаксация ребра** (relaxing an edge) | Проверка, даёт ли путь через вершину более короткий способ добраться до её соседа, чем лучший известный до сих пор, и, если да, обновление. |
| **устаревшая запись** (stale entry) | Запись в куче, которая была верной при добавлении, но устарела из-за более позднего, более короткого результата. Мы пропускаем её, когда она выходит. |
| **DAG** (Directed Acyclic Graph, ориентированный ациклический граф) | Ориентированный граф без циклов. Зависимости между задачами обычно образуют DAG. |
| **топологический порядок** (topological order) | Порядок вершин DAG, в котором у каждого ребра `a -> b` вершина `a` стоит раньше. |
| **входящая степень** (in-degree) | Число рёбер, которые ведут **в** вершину. |
| **алгоритм Кана** (Kahn's algorithm) | Топологическая сортировка, которая многократно берёт вершину с входящей степенью 0. Опубликован Артуром Каном. |

## 1. Почему BFS недостаточно

BFS (Урок 07.13) находит путь с **наименьшим числом рёбер**. На карте дорог у дорог есть длины, и маршрут с наименьшим числом дорог может быть намного длиннее другого маршрута.
Небольшая карта с пятью городами (числа — длины в километрах):

```text
          4
    A --------- B
    |         / |
  1 |      2 /  | 1
    |      /    |
    C --------- D ---- E
          5           3
```

Рёбра: `A-B` 4, `A-C` 1, `C-B` 2, `B-D` 1, `C-D` 5, `D-E` 3. От `A` до `D`:

| Маршрут | Дорог | Длина |
|---|---|---|
| `A, B, D` (его находит BFS: 2 ребра) | 2 | 4 + 1 = 5 |
| `A, C, D` | 2 | 1 + 5 = 6 |
| `A, C, B, D` (кратчайший) | 3 | 1 + 2 + 1 = **4** |

У лучшего маршрута больше дорог, чем у остальных. Нам нужен алгоритм, который складывает веса: **алгоритм Дейкстры**.

## 2. Алгоритм Дейкстры

**Идея.** Храните для каждой вершины лучшее известное расстояние от старта (сначала бесконечность для всех, 0 для старта). Повторяйте: возьмите незавершённую вершину с **наименьшим** известным расстоянием; это расстояние теперь окончательное.
Затем **ослабьте** её рёбра: для каждого соседа проверьте, лучше ли `distance[vertex] + weight` известного расстояния соседа, и если да, обновите его.

Почему наименьшее расстояние окончательное? Любая другая незавершённая вершина не ближе. Любой другой путь к нашей вершине должен был бы пройти через такую вершину, а затем добавить ещё вес (он не может быть меньше нуля). Значит, более короткого пути нет.
**Это верно, только если нет отрицательных весов.** С отрицательным ребром дорога, которая казалась длиннее, могла бы позже стать дешевле, и рассуждение ломается.

Чтобы быстро находить «незавершённую вершину с наименьшим расстоянием», мы используем **min-кучу** (Урок 07.12). Мы кладём в неё пары `(расстояние, вершина)`. Когда для вершины находится лучшее расстояние, мы добавляем новую пару, а старая остаётся в куче. Когда старая
пара наконец выходит, мы распознаём её как **устаревшую запись** (её расстояние больше лучшего известного) и пропускаем. Это проще, чем удалять её из кучи.

Дейкстра на карте выше, из `A`. Куча хранит `(расстояние, вершина)`; лучшие расстояния записаны как `A=0` и так далее:

```text
start:       A=0                                            heap: (0,A)
pop (0,A):   B = 4 (via A), C = 1 (via A)                   heap: (1,C) (4,B)
pop (1,C):   B = 1+2 = 3 is better than 4: update B;
             D = 1+5 = 6                                    heap: (3,B) (4,B)* (6,D)
pop (3,B):   D = 3+1 = 4 is better than 6: update D         heap: (4,B)* (4,D) (6,D)*
pop (4,B)*:  stale: B is already 3, skip
pop (4,D):   E = 4+3 = 7                                    heap: (6,D)* (7,E)
pop (6,D)*:  stale, skip
pop (7,E):   nothing to improve                             heap: empty
result:      A=0  B=3  C=1  D=4  E=7      (* = stale entry)
```

### Код

Поместите взвешенный граф и Дейкстру в `src/shortest_paths.rs`:

```rust
//! Weighted graphs and Dijkstra's shortest-path algorithm.

use std::cmp::Reverse;
use std::collections::BinaryHeap;

/// A graph whose edges have a length (a weight). Vertices are the numbers `0..vertex_count`.
#[derive(Debug, Clone)]
pub struct WeightedGraph {
    adjacency: Vec<Vec<(usize, u32)>>, // for each vertex: (neighbour, weight)
    directed: bool,
}

impl WeightedGraph {
    /// An undirected graph (every road works both ways) with no edges.
    pub fn new(vertex_count: usize) -> Self {
        WeightedGraph {
            adjacency: vec![Vec::new(); vertex_count],
            directed: false,
        }
    }

    /// A directed graph with no edges.
    pub fn new_directed(vertex_count: usize) -> Self {
        WeightedGraph {
            directed: true,
            ..WeightedGraph::new(vertex_count)
        }
    }

    pub fn vertex_count(&self) -> usize {
        self.adjacency.len()
    }

    /// Adds an edge with the given weight.
    ///
    /// # Panics
    /// Panics if a vertex number is not smaller than `vertex_count()`.
    pub fn add_edge(&mut self, from: usize, to: usize, weight: u32) {
        let n = self.vertex_count();
        assert!(
            from < n && to < n,
            "edge {from} -> {to}: a vertex number must be below {n}"
        );
        self.adjacency[from].push((to, weight));
        if !self.directed {
            self.adjacency[to].push((from, weight));
        }
    }

    /// The `(neighbour, weight)` pairs of `vertex`.
    pub fn neighbours(&self, vertex: usize) -> &[(usize, u32)] {
        &self.adjacency[vertex]
    }
}

/// The result of Dijkstra's algorithm for one start vertex.
#[derive(Debug, PartialEq, Eq)]
pub struct Paths {
    /// The length of the shortest path from the start, or `None` if unreachable.
    pub distance: Vec<Option<u64>>,
    /// The vertex before this one on a shortest path, or `None` for the start and unreachable vertices.
    pub previous: Vec<Option<usize>>,
}

/// Dijkstra's algorithm: shortest paths from `start` to every vertex.
/// Weights are `u32`, so they cannot be negative, which the algorithm requires.
pub fn dijkstra(graph: &WeightedGraph, start: usize) -> Paths {
    let n = graph.vertex_count();
    let mut distance: Vec<Option<u64>> = vec![None; n];
    let mut previous = vec![None; n];
    // A min-heap of (distance, vertex): `Reverse` makes the smallest distance come out first.
    let mut heap = BinaryHeap::new();

    distance[start] = Some(0);
    heap.push(Reverse((0u64, start)));
    while let Some(Reverse((d, vertex))) = heap.pop() {
        if distance[vertex].is_some_and(|best| d > best) {
            continue; // a stale entry: a shorter way to `vertex` was found after this was pushed
        }
        for &(neighbour, weight) in graph.neighbours(vertex) {
            let candidate = d + u64::from(weight);
            if distance[neighbour].is_none_or(|best| candidate < best) {
                distance[neighbour] = Some(candidate);
                previous[neighbour] = Some(vertex);
                heap.push(Reverse((candidate, neighbour)));
            }
        }
    }
    Paths { distance, previous }
}

/// The shortest path from the start of `paths` to `to`, including both ends, or `None`.
pub fn path_to(paths: &Paths, to: usize) -> Option<Vec<usize>> {
    paths.distance[to]?; // unreachable
    let mut path = vec![to];
    let mut current = to;
    while let Some(before) = paths.previous[current] {
        path.push(before);
        current = before;
    }
    path.reverse();
    Some(path)
}
```

**Что происходит:**

1. Ребро — это пара `(neighbour, weight)` с `weight: u32`. Беззнаковый тип не может быть отрицательным, поэтому **тип делает главное требование Дейкстры невозможным для нарушения** (📏 делайте недопустимые состояния непредставимыми). Если в ваших данных есть отрицательные числа,
   эту функцию использовать нельзя, и компилятор вам это скажет.
2. Расстояния — `u64`, а `u64::from(weight)` расширяет вес перед сложением. У пути может быть много рёбер до `u32::MAX` каждое, и сумма переполнила бы `u32` (тест ниже это проверяет).
3. `Reverse((0u64, start))` кладёт пару `(расстояние, вершина)` в min-кучу. Кортежи сравниваются поле за полем, поэтому наименьшее расстояние выходит первым.
4. Цикл извлекает ближайшую запись. Если её расстояние `d` больше лучшего известного расстояния этой вершины, это устаревшая запись, и `continue` её пропускает. Иначе мы **ослабляем** каждое ребро: если путь через эту вершину
   лучше лучшего известного расстояния до соседа (или его ещё нет), мы записываем его, запоминаем `previous` и добавляем новую пару.
5. `path_to` идёт по `previous` назад, как `shortest_path` в Уроке 07.13.

**Стоимость.** Каждое ребро ослабляется не более одного раза за каждую обработку его начальной вершины, а вершина обрабатывается один раз (устаревшие записи пропускаются), поэтому в кучу кладётся не более E пар. Операция с кучей стоит O(log E), а E < V² означает log E ≤ 2 log V.
Всего **O((V + E) log V)** времени и **O(V + E)** памяти. С неотсортированным массивом вместо кучи поиск минимума стоит O(V) каждый раз, а всего O(V²), что лучше только для очень плотных графов.

### Две ошибки компиляции, которые вы можете встретить

Если забыть, что в куче лежат значения `Reverse`, или смешать вес `u32` с расстоянием `u64`:

```rust
use std::cmp::Reverse;
use std::collections::BinaryHeap;

pub fn first_vertex(heap: &mut BinaryHeap<Reverse<(u64, usize)>>) -> Option<usize> {
    let (_distance, vertex) = heap.pop()?;
    Some(vertex)
}

pub fn add(d: u64, weight: u32) -> u64 {
    d + weight
}
```

```text
error[E0308]: mismatched types
 --> src/lib.rs:5:9
  |
5 |     let (_distance, vertex) = heap.pop()?;
  |         ^^^^^^^^^^^^^^^^^^^   ----------- this expression has type `Reverse<(u64, usize)>`
  |         |
  |         expected `Reverse<(u64, usize)>`, found `(_, _)`
  |
help: try wrapping the pattern in `std::cmp::Reverse`
  |
5 |     let std::cmp::Reverse((_distance, vertex)) = heap.pop()?;
  |         ++++++++++++++++++                   +

error[E0308]: mismatched types
  --> src/lib.rs:10:9
   |
10 |     d + weight
   |         ^^^^^^ expected `u64`, found `u32`

error[E0277]: cannot add `u32` to `u64`
  --> src/lib.rs:10:7
   |
10 |     d + weight
   |       ^ no implementation for `u64 + u32`
```

Первая ошибка: куча возвращает `Reverse<(u64, usize)>`, поэтому образец должен быть `Reverse((distance, vertex))`. Вторая: Rust никогда не преобразует числа молча. Пишите `d + u64::from(weight)` (безопасное расширение, которое не может завершиться ошибкой).

## 3. Тестирование Дейкстры

Стратегия тестирования та же, что и для BFS: медленный эталон, который ослабляет все рёбра, пока ничего не меняется, случайные графы, включая нулевые веса, и пример с картинки. Два дополнительных теста: при всех весах 1 Дейкстра должна давать расстояния BFS из Урока 07.13, а очень длинные пути не должны переполняться. Добавьте в конец `src/shortest_paths.rs`:

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use crate::graphs::{Graph, bfs};
    use crate::testutil::Rng;

    /// The graph of the lesson: A = 0, B = 1, C = 2, D = 3, E = 4.
    fn sample() -> WeightedGraph {
        let mut graph = WeightedGraph::new(5);
        for (a, b, weight) in [
            (0, 1, 4),
            (0, 2, 1),
            (2, 1, 2),
            (1, 3, 1),
            (2, 3, 5),
            (3, 4, 3),
        ] {
            graph.add_edge(a, b, weight);
        }
        graph
    }

    /// Pseudo-random edges (the `Rng` of Lesson 07.3): `(from, to, weight)`.
    fn random_edges(
        seed: u64,
        vertices: usize,
        edges: usize,
        max_weight: u32,
    ) -> Vec<(usize, usize, u32)> {
        let mut rng = Rng::new(seed);
        (0..edges)
            .map(|_| {
                let from = rng.below(vertices);
                let to = rng.below(vertices);
                let weight = (rng.next_u64() % (u64::from(max_weight) + 1)) as u32;
                (from, to, weight)
            })
            .collect()
    }

    fn build(vertices: usize, edges: &[(usize, usize, u32)], directed: bool) -> WeightedGraph {
        let mut graph = if directed {
            WeightedGraph::new_directed(vertices)
        } else {
            WeightedGraph::new(vertices)
        };
        for &(from, to, weight) in edges {
            graph.add_edge(from, to, weight);
        }
        graph
    }

    /// A slow but obviously correct reference: relax all edges until nothing changes.
    fn reference_distances(graph: &WeightedGraph, start: usize) -> Vec<Option<u64>> {
        let mut distance = vec![None; graph.vertex_count()];
        distance[start] = Some(0);
        loop {
            let mut changed = false;
            for vertex in 0..graph.vertex_count() {
                if let Some(d) = distance[vertex] {
                    for &(next, weight) in graph.neighbours(vertex) {
                        let candidate = d + u64::from(weight);
                        if distance[next].is_none_or(|old| candidate < old) {
                            distance[next] = Some(candidate);
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
    fn the_example_of_the_lesson() {
        let paths = dijkstra(&sample(), 0);
        assert_eq!(
            paths.distance,
            [Some(0), Some(3), Some(1), Some(4), Some(7)]
        );
        assert_eq!(path_to(&paths, 3), Some(vec![0, 2, 1, 3])); // 3 edges, length 4
        assert_eq!(path_to(&paths, 0), Some(vec![0]));
    }

    #[test]
    fn unreachable_vertices() {
        let graph = build(4, &[(0, 1, 5)], false);
        let paths = dijkstra(&graph, 0);
        assert_eq!(paths.distance, [Some(0), Some(5), None, None]);
        assert_eq!(path_to(&paths, 3), None);
    }

    #[test]
    fn matches_the_reference() {
        for seed in 1..=40 {
            for directed in [false, true] {
                let edges = random_edges(seed, 20, 45, 20); // weights 0..=20, zero allowed
                let graph = build(20, &edges, directed);
                for start in 0..20 {
                    assert_eq!(
                        dijkstra(&graph, start).distance,
                        reference_distances(&graph, start),
                        "seed {seed}, directed {directed}, start {start}"
                    );
                }
            }
        }
    }

    #[test]
    fn unit_weights_give_the_bfs_distances() {
        for seed in 1..=30 {
            let edges = random_edges(seed, 25, 40, 0);
            let mut weighted = WeightedGraph::new(25);
            let mut plain = Graph::new(25);
            for &(from, to, _) in &edges {
                weighted.add_edge(from, to, 1);
                plain.add_edge(from, to);
            }
            let dijkstra_distance = dijkstra(&weighted, 0).distance;
            let bfs_distance: Vec<Option<u64>> = bfs(&plain, 0)
                .distance
                .iter()
                .map(|d| d.map(|d| d as u64))
                .collect();
            assert_eq!(dijkstra_distance, bfs_distance, "seed {seed}");
        }
    }

    #[test]
    fn long_distances_do_not_overflow() {
        let mut graph = WeightedGraph::new(4);
        for v in 0..3 {
            graph.add_edge(v, v + 1, u32::MAX);
        }
        assert_eq!(
            dijkstra(&graph, 0).distance[3],
            Some(3 * u64::from(u32::MAX))
        );
    }
}
```

**Что происходит:**

1. `the_example_of_the_lesson` проверяет числа из трассировки выше: `A=0 B=3 C=1 D=4 E=7` и что маршрут до `D` — `A, C, B, D`: три ребра, а не два, которые выбрал бы BFS.
2. `matches_the_reference` сравнивает Дейкстру с медленным эталоном для 40 зёрен × ориентированный и неориентированный × все 20 стартовых вершин. Среди весов есть 0, важный граничный случай: дороги нулевой длины не должны зацикливать алгоритм.
3. `unit_weights_give_the_bfs_distances` строит одни и те же случайные рёбра в обоих типах графа с весом 1 и сравнивает расстояния. Два алгоритма должны совпадать всякий раз, когда должны.
4. `long_distances_do_not_overflow` использует три ребра по `u32::MAX`: сумма (12 884 901 885) больше `u32`, а расстояние `u64` точное.

```bash
cd ~/projects/rust/code/p07-algorithms
cargo test shortest_paths
```

```text
running 5 tests
test shortest_paths::tests::unreachable_vertices ... ok
test shortest_paths::tests::long_distances_do_not_overflow ... ok
test shortest_paths::tests::the_example_of_the_lesson ... ok
test shortest_paths::tests::unit_weights_give_the_bfs_distances ... ok
test shortest_paths::tests::matches_the_reference ... ok

test result: ok. 5 passed; 0 failed; 0 ignored; 0 measured; 53 filtered out; finished in 0.01s
```

(Отфильтрованы 53 теста из основного текста Уроков 07.1–07.13; с тестами из упражнений ваше число будет больше.)

## 4. Топологическая сортировка

Теперь другой вопрос. У вас есть задачи, и некоторые нужно закончить до других: «проектирование до кода», «код до тестов», «тесты и документация до выпуска». В каком порядке можно сделать их все? Это **ориентированный** граф: ребро `a -> b` означает «a до b».
Порядок, соблюдающий каждое ребро, — это **топологический порядок**. Он существует ровно тогда, когда в графе **нет цикла** (**DAG**): если `a` должна предшествовать `b`, а `b` должна предшествовать `a`, ничего нельзя начать.

**Алгоритм Кана.** Посчитайте для каждой вершины её **входящую степень**: сколько рёбер ведут в неё, то есть сколько вещей она ещё ждёт. Вершину с входящей степенью 0 можно делать сейчас. Сделайте её и сообщите её преемникам: каждый из них ждёт на одну вещь меньше. Каждый преемник, у которого входящая степень стала 0,
готов. Повторяйте, пока ничего не готово. Если все вершины сделаны, порядок найден. Если какие-то никогда не стали готовыми, они ждут друг друга в цикле.

Задачи `design(0) code(1) test(2) docs(3) ship(4)` с правилами `0->1, 1->2, 0->3, 2->4, 3->4`:

```text
in-degrees:  design 0, code 1, test 1, docs 1, ship 2          ready: [design]
do design:   code 1->0, docs 1->0                              ready: [code, docs]
do code:     test 1->0                                         ready: [docs, test]
do docs:     ship 2->1                                         ready: [test]
do test:     ship 1->0                                         ready: [ship]
do ship:                                                       order: design code docs test ship
```

Готовые вершины ждут в очереди (Урок 07.5). Поместите это в `src/topological.rs`:

```rust
//! Topological sort of a directed graph.

use crate::graphs::Graph;
use std::collections::VecDeque;

/// Orders the vertices so that for every edge `a -> b`, `a` comes before `b`.
/// Returns `None` if the graph has a cycle (then no such order exists).
pub fn topological_sort(graph: &Graph) -> Option<Vec<usize>> {
    let n = graph.vertex_count();

    // in_degree[v] = how many edges lead into v = how many things v still waits for.
    let mut in_degree = vec![0usize; n];
    for vertex in 0..n {
        for &next in graph.neighbours(vertex) {
            in_degree[next] += 1;
        }
    }

    let mut ready: VecDeque<usize> = (0..n).filter(|&v| in_degree[v] == 0).collect();
    let mut order = Vec::with_capacity(n);
    while let Some(vertex) = ready.pop_front() {
        order.push(vertex);
        for &next in graph.neighbours(vertex) {
            in_degree[next] -= 1;
            if in_degree[next] == 0 {
                ready.push_back(next); // everything `next` waited for is done
            }
        }
    }

    // If some vertices never became ready, they lie on or behind a cycle.
    (order.len() == n).then_some(order)
}
```

**Что происходит:**

1. Первый цикл посещает каждое ребро один раз и считает входящие степени. Он использует тип `Graph` из Урока 07.13 (создайте его через `Graph::new_directed`).
2. `ready` начинается со всех вершин, которые ничего не ждут. Главный цикл берёт одну, добавляет её в `order` и убирает по одному «ожиданию» у каждого преемника. Преемник, дошедший до 0, готов и попадает в очередь.
3. Вершина на цикле никогда не достигает входящей степени 0, потому что вершина, которую она ждёт, ждёт её. Поэтому `order.len() < n` означает цикл, а `then_some` превращает проверку в `None` или `Some(order)`.
   `bool::then_some(x)` возвращает `Some(x)`, если bool истинно, иначе `None`.
4. Дублирующиеся рёбра и вершина с ребром в саму себя обрабатываются правильно: дубликат считается дважды и дважды вычитается; петля — это цикл и возвращает `None`.

**Стоимость.** Каждая вершина попадает в очередь не более одного раза, а каждое ребро просматривается дважды (один раз для подсчёта, один для вычитания): **O(V + E)** времени, **O(V)** дополнительной памяти. Порядок не единственный; алгоритм Кана с очередью даёт один из них. (Топологический порядок можно построить и через DFS (Depth-First Search, поиск в глубину из Урока 07.13), записывая
вершины в порядке, обратном тому, в котором DFS их завершает, но алгоритм Кана ещё и сообщает о циклах без дополнительной работы.)

Создайте `examples/e14_build_order.rs` и запустите:

```rust
use p07_algorithms::graphs::Graph;
use p07_algorithms::topological::topological_sort;

fn main() {
    let tasks = ["design", "code", "test", "docs", "ship"];
    // (a, b) means: task `a` must be finished before task `b` can start.
    let rules = [(0, 1), (1, 2), (0, 3), (2, 4), (3, 4)];

    let mut graph = Graph::new_directed(tasks.len());
    for (a, b) in rules {
        graph.add_edge(a, b);
    }

    match topological_sort(&graph) {
        Some(order) => {
            let names: Vec<&str> = order.iter().map(|&i| tasks[i]).collect();
            println!("do the tasks in this order: {}", names.join(" -> "));
        }
        None => println!("the rules contain a cycle: no order exists"),
    }

    graph.add_edge(4, 0); // "ship" must come before "design": a cycle
    match topological_sort(&graph) {
        Some(_) => println!("an order exists"),
        None => println!("after adding ship -> design: the rules contain a cycle: no order exists"),
    }
}
```

```bash
cargo run --example e14_build_order
```

```text
do the tasks in this order: design -> code -> docs -> test -> ship
after adding ship -> design: the rules contain a cycle: no order exists
```

Системы сборки, менеджеры пакетов (порядок компиляции зависимостей; так делает Cargo), формулы в таблицах и пререквизиты курсов — все используют топологическую сортировку.

### Тестирование топологической сортировки

У порядка есть **проверяемое свойство**: для каждого ребра источник стоит раньше цели. Эталонный ответ нам не нужен, потому что верных порядков много. А ответ на вопрос «есть ли цикл» мы сравниваем с медленным эталоном (транзитивное замыкание: может ли `v` дойти до самой себя?).
Добавьте в конец `src/topological.rs`:

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use crate::testutil::Rng;

    /// A pseudo-random directed graph (the `Rng` of Lesson 07.3).
    fn random_graph(seed: u64, vertices: usize, edges: usize, acyclic: bool) -> Graph {
        let mut rng = Rng::new(seed);
        let mut graph = Graph::new_directed(vertices);
        for _ in 0..edges {
            let a = rng.below(vertices);
            let b = rng.below(vertices);
            if acyclic {
                if a != b {
                    graph.add_edge(a.min(b), a.max(b)); // edges only go "upwards": no cycle is possible
                }
            } else {
                graph.add_edge(a, b);
            }
        }
        graph
    }

    /// Does some vertex lead back to itself? Slow reference: the transitive closure.
    fn has_cycle(graph: &Graph) -> bool {
        let n = graph.vertex_count();
        let mut reach = vec![vec![false; n]; n];
        for (a, row) in reach.iter_mut().enumerate() {
            for &b in graph.neighbours(a) {
                row[b] = true;
            }
        }
        for k in 0..n {
            for i in 0..n {
                for j in 0..n {
                    if reach[i][k] && reach[k][j] {
                        reach[i][j] = true;
                    }
                }
            }
        }
        (0..n).any(|v| reach[v][v])
    }

    #[test]
    fn a_small_example() {
        let mut graph = Graph::new_directed(5);
        for (a, b) in [(0, 1), (1, 2), (0, 3), (3, 4), (2, 4)] {
            graph.add_edge(a, b);
        }
        assert_eq!(topological_sort(&graph), Some(vec![0, 1, 3, 2, 4]));
    }

    #[test]
    fn empty_and_edgeless_graphs() {
        assert_eq!(topological_sort(&Graph::new_directed(0)), Some(vec![]));
        assert_eq!(
            topological_sort(&Graph::new_directed(3)),
            Some(vec![0, 1, 2])
        );
    }

    #[test]
    fn cycles_are_detected() {
        let mut triangle = Graph::new_directed(3);
        for (a, b) in [(0, 1), (1, 2), (2, 0)] {
            triangle.add_edge(a, b);
        }
        assert_eq!(topological_sort(&triangle), None);

        let mut self_loop = Graph::new_directed(2);
        self_loop.add_edge(1, 1);
        assert_eq!(topological_sort(&self_loop), None);
    }

    #[test]
    fn the_order_respects_every_edge() {
        for seed in 1..=50 {
            let graph = random_graph(seed, 15, 30, true);
            let order =
                topological_sort(&graph).expect("a graph with only upward edges has no cycle");
            assert_eq!(order.len(), 15);
            let mut position = [0; 15];
            for (i, &v) in order.iter().enumerate() {
                position[v] = i;
            }
            for a in 0..15 {
                for &b in graph.neighbours(a) {
                    assert!(position[a] < position[b], "edge {a} -> {b}, seed {seed}");
                }
            }
        }
    }

    #[test]
    fn sorting_succeeds_exactly_when_there_is_no_cycle() {
        for seed in 1..=100 {
            let graph = random_graph(seed, 6, 7, false);
            assert_eq!(
                topological_sort(&graph).is_some(),
                !has_cycle(&graph),
                "seed {seed}"
            );
        }
    }
}
```

**Что происходит:**

1. `random_graph(..., acyclic: true)` создаёт только рёбра от меньшего номера к большему, поэтому цикл невозможен. `the_order_respects_every_edge` хранит позицию каждой вершины в порядке и проверяет `position[a] < position[b]` для каждого ребра.
2. `sorting_succeeds_exactly_when_there_is_no_cycle` использует графы, в которых могут быть циклы, а `has_cycle` — медленный судья (O(V³), нормально для 6 вершин).
3. `a_small_example` фиксирует точный порядок из урока. Другая верная реализация могла бы вернуть другой допустимый порядок; если вы измените алгоритм, этот один тест, возможно, придётся обновить, а тест свойства останется.

```bash
cargo test topological
```

```text
running 5 tests
test topological::tests::empty_and_edgeless_graphs ... ok
test topological::tests::a_small_example ... ok
test topological::tests::cycles_are_detected ... ok
test topological::tests::the_order_respects_every_edge ... ok
test topological::tests::sorting_succeeds_exactly_when_there_is_no_cycle ... ok

test result: ok. 5 passed; 0 failed; 0 ignored; 0 measured; 58 filtered out; finished in 0.01s
```

> 🧪 **Тестирование:** когда верных ответов много, проверяйте **свойство**, делающее ответ верным («каждое ребро идёт вперёд в порядке»), а не один фиксированный ответ. Это идея тестирования на основе свойств (Этап 09).

## 5. Что использовать в настоящем коде

В стандартной библиотеке нет графовых алгоритмов; `BinaryHeap`, `Reverse` и `VecDeque` — строительные блоки, как здесь. В крейте `petgraph` есть `dijkstra`, `toposort` и многое другое (A\*, Беллман–Форд, сильно связные компоненты).
Знайте следующих родственников Дейкстры, которые могут вам когда-нибудь понадобиться:

| Задача | Алгоритм | Время |
|---|---|---|
| кратчайшие пути, **отрицательные** веса разрешены | Беллман–Форд | O(V · E) |
| кратчайший путь с хорошей оценкой расстояния до цели | A\* («А со звёздочкой») | зависит от оценки |
| кратчайшие пути между **всеми** парами | Флойд–Уоршелл | O(V³) |
| все веса равны | BFS | O(V + E) |

## Частые ошибки

**1. Дейкстра с отрицательными весами.** Доказательство «наименьшее расстояние окончательное» перестаёт работать. Держите веса беззнаковыми, как в этом уроке, или используйте Беллмана–Форда.

**2. Забытая проверка устаревших записей.** Без `if distance[vertex].is_some_and(|best| d > best) { continue; }` результат всё равно верен, но каждая устаревшая запись обрабатывается заново со всеми её рёбрами. На больших графах это может быть во много раз медленнее.

**3. `u32` для суммы расстояний.** Веса помещаются в `u32`, а пути — нет. Используйте `u64` для сумм (или проверяйте переполнение через `checked_add`).

**4. Топологическая сортировка неориентированного графа.** Неориентированное ребро `a - b` — это ещё и `b -> a`, цикл из двух вершин. Используйте `Graph::new_directed`.

## Попробуйте сами

1. Добавьте `shortest_paths.rs` и его тесты, затем `topological.rs` и его тесты. Запустите `cargo test`.
2. В `dijkstra` удалите проверку устаревших записей (строки `if ... continue`). Проходят ли тесты по-прежнему? Почему ответ всё равно верен? (См. ошибку 2.) Верните проверку.
3. В `topological_sort` сделайте `ready` вектором `Vec<usize>`, используемым как стек (`pop()` вместо `pop_front()`, `push(next)` вместо `push_back(next)`). Запустите `cargo test topological`. Падают два теста: какие и почему тесты свойств всё равно проходят?
4. Запустите `cargo fmt`, `cargo clippy --all-targets` и сделайте коммит: `git add code/p07-algorithms && git commit -m "feat(p07): Dijkstra and topological sort"`.

## Упражнения

### Упражнение 1 — Дейкстра вручную (легко)

Неориентированный граф имеет рёбра `0-1` (7), `0-2` (9), `0-5` (14), `1-2` (10), `1-3` (15), `2-3` (11), `2-5` (2), `3-4` (6), `4-5` (9). Выполните Дейкстру из вершины `0` вручную. Дайте итоговые расстояния до всех вершин и для каждой вершины — `previous`.

<details>
<summary>Подсказка</summary>

Порядок извлечения по расстоянию: вершина 0 (0), затем 1 (7), затем 2 (9). Из 2 вершина 5 получает 9 + 2 = 11 (лучше, чем 14). Продолжайте с наименьшим незавершённым расстоянием.

</details>

<details>
<summary>Ответ</summary>

```text
distance:  0:0   1:7   2:9   3:20   4:20   5:11
previous:  0:-   1:0   2:0   3:2    4:5    5:2
```

Почему: `1` и `2` достигаются напрямую (7 и 9). Из `2`: `3` получает 9 + 11 = 20, `5` получает 9 + 2 = 11, что лучше 14 (прямая дорога). Из `5` (11): `4` получает 11 + 9 = 20. Из `1`: `3` была бы 7 + 15 = 22, хуже 20. Из `3` или `4` улучшений нет. Путь до `4` — `0, 2, 5, 4` (длина 20).

</details>

### Упражнение 2 — Пути складываются (легко)

Напишите тест: для случайных неориентированных графов для каждой достижимой цели `path_to` начинается в старте и заканчивается в цели, а сумма самых лёгких рёбер между каждыми двумя соседними вершинами пути равна `distance[target]`.

<details>
<summary>Подсказка</summary>

`path.windows(2)` даёт каждую соседнюю пару. Для пары `(a, b)` посмотрите на `graph.neighbours(a)`, оставьте записи, у которых сосед равен `b`, и возьмите минимальный вес (могут быть параллельные рёбра). Складывайте веса как `u64`.

</details>

<details>
<summary>Ответ</summary>

```rust
    #[test]
    fn paths_add_up_to_the_distances() {
        for seed in 1..=30 {
            let edges = random_edges(seed, 15, 30, 9);
            let graph = build(15, &edges, false);
            let paths = dijkstra(&graph, 0);
            for target in 0..15 {
                match (path_to(&paths, target), paths.distance[target]) {
                    (Some(path), Some(distance)) => {
                        assert_eq!((path[0], path[path.len() - 1]), (0, target));
                        let length: u64 = path
                            .windows(2)
                            .map(|pair| {
                                graph
                                    .neighbours(pair[0])
                                    .iter()
                                    .filter(|&&(next, _)| next == pair[1])
                                    .map(|&(_, weight)| u64::from(weight))
                                    .min()
                                    .expect("neighbours on a path are joined by an edge")
                            })
                            .sum();
                        assert_eq!(length, distance, "seed {seed}, target {target}");
                    }
                    (None, None) => {}
                    other => panic!("path and distance disagree: {other:?}"),
                }
            }
        }
    }
```

Почему это работает: тест проверяет, что **путь** и **число** рассказывают одну и ту же историю, что ловит неверный `previous`, даже когда расстояния верны. Ветка `(None, None)` охватывает недостижимые цели, а последняя ветка паникует, если существует только один из двух ответов.

</details>

### Упражнение 3 — Назовите сложность (средне)

1. Каково время Дейкстры с двоичной кучей для V вершин и E рёбер? Каково максимальное число записей в куче?
2. На очень плотном графе (E близко к V²) вариант Дейкстры, который находит ближайшую незавершённую вершину просмотром массива, стоит O(V²). Что лучше тогда и почему?
3. Каково время топологической сортировки? Что она возвращает для графа с циклом?

<details>
<summary>Подсказка</summary>

Для (2) сравните V² и (V + E) log V при E ≈ V². Помните, что операция с кучей стоит логарифм.

</details>

<details>
<summary>Ответ</summary>

1. **O((V + E) log V)** времени. Каждое успешное ослабление добавляет одну запись, поэтому в куче не более **E** записей (плюс старт).
2. При E ≈ V² вариант с кучей стоит около V² log V, поэтому для очень плотных графов **лучше вариант с массивом за O(V²)**: он избегает логарифма. Для разреженных графов (E близко к V) вариант с кучей намного лучше: V log V против V².
3. **O(V + E)**. Для графа с циклом она возвращает `None`, потому что вершины цикла никогда не достигают входящей степени 0.

</details>

### Упражнение 4 — Самая длинная цепочка (средне)

В DAG задач самая длинная цепочка зависимостей определяет, сколько «раундов» вам нужно, если можно запускать любое число независимых задач одновременно. Напишите `longest_chain(graph: &Graph) -> Option<usize>`, возвращающую число вершин на самом длинном пути, или `None`, если в графе есть цикл. Проверьте её на примере порядка сборки (ответ 4), на пустом графе, на графе без рёбер и на графе с циклом.

<details>
<summary>Подсказка</summary>

Сначала получите топологический порядок. Храните `depth[v]` — число вершин на самой длинной цепочке, заканчивающейся в `v`, начиная с 1. Посещайте вершины в топологическом порядке; для каждого ребра `v -> w`: `depth[w] = max(depth[w], depth[v] + 1)`. Ответ — наибольшая глубина.

</details>

<details>
<summary>Ответ</summary>

```rust
/// The number of vertices on the longest chain of dependencies, or `None` if there is a cycle.
pub fn longest_chain(graph: &Graph) -> Option<usize> {
    let order = topological_sort(graph)?;
    let mut depth = vec![1usize; graph.vertex_count()]; // a vertex alone is a chain of length 1
    for &vertex in &order {
        for &next in graph.neighbours(vertex) {
            depth[next] = depth[next].max(depth[vertex] + 1);
        }
    }
    Some(depth.into_iter().max().unwrap_or(0))
}
```

Тест внутри модуля `tests`:

```rust
    #[test]
    fn longest_chain_examples() {
        let mut graph = Graph::new_directed(5);
        for (a, b) in [(0, 1), (1, 2), (0, 3), (2, 4), (3, 4)] {
            graph.add_edge(a, b);
        }
        assert_eq!(longest_chain(&graph), Some(4)); // 0 -> 1 -> 2 -> 4
        assert_eq!(longest_chain(&Graph::new_directed(0)), Some(0));
        assert_eq!(longest_chain(&Graph::new_directed(3)), Some(1));
        graph.add_edge(4, 0);
        assert_eq!(longest_chain(&graph), None);
    }
```

Почему это работает: в топологическом порядке, когда мы доходим до вершины, все её предшественники уже обработаны, поэтому её `depth` окончателен и его можно передать её преемникам. Это первое знакомство с динамическим программированием (Урок 07.16): ответ для вершины строится из ответов вершин перед ней. Время O(V + E).

</details>

## Вопросы для самопроверки

1. Почему BFS не может найти кратчайший маршрут на карте дорог с длинами?

<details>
<summary>Ответ</summary>

BFS минимизирует **число рёбер**, а не сумму их весов. Маршрут с большим числом, но более коротких дорог может быть короче в сумме.

</details>

2. Почему алгоритму Дейкстры нужны неотрицательные веса?

<details>
<summary>Ответ</summary>

Он считает ближайшую незавершённую вершину окончательной, потому что любой другой путь к ней должен был бы пройти через более далёкую вершину и добавить ещё вес. Отрицательное ребро могло бы сделать такой путь короче, и рассуждение (а с ним и результат) было бы неверным.

</details>

3. Что такое устаревшая запись в куче и что с ней делают?

<details>
<summary>Ответ</summary>

Запись `(расстояние, вершина)`, расстояние в которой больше лучшего известного расстояния вершины, потому что более короткий путь был найден после её добавления. Когда её извлекают, её пропускают.

</details>

4. Что означает в алгоритме Кана, что некоторые вершины так и не попали в очередь?

<details>
<summary>Ответ</summary>

Они лежат на цикле или зависят от него: каждая ждёт то, что ждёт её, поэтому их входящая степень никогда не достигает 0. Топологического порядка не существует.

</details>

## Итоги

- На взвешенном графе кратчайший путь — тот, у которого наименьший суммарный вес; BFS минимизирует только число рёбер.
- Алгоритм Дейкстры многократно берёт ближайшую незавершённую вершину (из min-кучи пар `(расстояние, вершина)` в `Reverse`) и ослабляет её рёбра. Устаревшие записи кучи пропускаются.
- Ему нужны неотрицательные веса (используйте беззнаковые веса, чтобы тип это гарантировал), а стоит он O((V + E) log V). Суммируйте расстояния в `u64`.
- Топологический порядок ставит источник каждого ребра перед его целью и существует ровно для DAG. Алгоритм Кана многократно берёт вершины с входящей степенью 0; он работает за O(V + E) и обнаруживает циклы.
- Тестируйте кратчайшие пути против медленного эталона с ослаблением и против BFS при единичных весах; тестируйте топологические порядки по свойству, а обнаружение циклов — против транзитивного замыкания.

## Источники и что почитать

- [Rust standard library — std::collections::BinaryHeap](https://doc.rust-lang.org/std/collections/struct.BinaryHeap.html) — на официальной странице есть пример Дейкстры с упорядочением в духе `Reverse`.
- [Rust standard library — std::cmp::Reverse](https://doc.rust-lang.org/std/cmp/struct.Reverse.html) — обёртка, превращающая max-кучу в min-кучу.
- [Rust standard library — bool::then_some](https://doc.rust-lang.org/std/primitive.bool.html#method.then_some) — превращение условия в `Option`.
- [petgraph::algo on docs.rs](https://docs.rs/petgraph/latest/petgraph/algo/index.html) — готовые `dijkstra`, `toposort`, `bellman_ford`, `astar` и многое другое для промышленного кода.

---

[🏠 Начало](../START-HERE.md) · [← 07.13 Графы: BFS и DFS](13-graphs-bfs-and-dfs.md) · [📋 Этап 07](README.md) · [07.15 Два указателя и скользящее окно →](15-two-pointers-and-sliding-window.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
