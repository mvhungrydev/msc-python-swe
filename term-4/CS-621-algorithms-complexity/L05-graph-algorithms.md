# CS-621 · Lesson 05 — Graph Algorithms and Modelling

**Estimated study time:** 5 hours
**Prerequisites:** L01–L04

---

## 1. Orientation

The most valuable skill in this lesson is not implementing Dijkstra. It is **recognizing that
your problem is a graph problem**.

- "Which services must be deployed before which?" — topological sort.
- "Can every task be assigned to a qualified worker?" — bipartite matching.
- "What is the maximum data rate through this network?" — max flow.
- "Which modules form a cycle?" — strongly connected components.
- "Which changes must ship together?" — connected components of the co-change graph
  (SE-521 L02).
- "Can these constraints be satisfied?" — 2-SAT, which is an SCC problem.

Each of these is solved, optimally, in near-linear time by an algorithm from the 1970s. The
engineer who recognizes the shape solves it in an afternoon; the one who does not writes a
heuristic and maintains it for years.

## 2. Theory

### 2.1 Representations

| Representation | Space | Edge query | Neighbours | Use |
|---|---|---|---|---|
| Adjacency list | Θ(n + m) | Θ(deg) | Θ(deg) | **default**; sparse graphs |
| Adjacency matrix | Θ(n²) | Θ(1) | Θ(n) | dense; matrix algorithms |
| Edge list | Θ(m) | Θ(m) | Θ(m) | Kruskal; input format |
| CSR (compressed sparse row) | Θ(n + m) | Θ(log deg) | Θ(deg) | large graphs; cache-friendly |

Real graphs are almost always sparse (m = O(n) or O(n log n)), so adjacency lists are the
default. For very large graphs, CSR — two flat arrays, offsets and targets — is what gives
you the locality of PY-602 L06, and it is what `scipy.sparse` and every serious graph library
uses.

In Python: `dict[Node, list[Node]]` for clarity, `networkx` for exploration and correctness,
`scipy.sparse` or `rustworkx`/`igraph` for scale. Know the sizes at which you must switch —
`networkx` is comfortable to about 10⁵–10⁶ edges and painful beyond.

### 2.2 Traversal

**BFS.** Queue. Visits in order of distance from the source. Θ(n + m).

- Shortest paths in an **unweighted** graph. (For weighted, you need Dijkstra — using BFS on
  a weighted graph is a common and silent error.)
- Level structure, bipartiteness testing, connected components.
- **0-1 BFS**: with edge weights only 0 and 1, a deque (push-front for 0, push-back for 1)
  gives shortest paths in Θ(n + m) — a useful trick.

**DFS.** Stack or recursion. Θ(n + m). Produces a forest with a classification of every edge
as tree, back, forward, or cross — and that classification is what makes DFS powerful:

- **A back edge exists ⟺ the graph has a cycle.** (In a directed graph; for undirected, a
  back edge to a non-parent.)
- **Topological order** = reverse of DFS finish times.
- **Strongly connected components** — Tarjan's or Kosaraju's, both DFS-based.
- **Articulation points and bridges** — Tarjan's low-link.

In Python, recursive DFS hits the 1,000-frame recursion limit on graphs of depth > 1,000.
Write it iteratively with an explicit stack for anything real; it is ten lines and it removes
a whole class of production failure.

### 2.3 Shortest paths

| Algorithm | Weights | Complexity | Use |
|---|---|---|---|
| BFS | unweighted | Θ(n + m) | the simple case |
| Dijkstra | non-negative | Θ((n + m) log n) with a binary heap | **the default** |
| Bellman–Ford | any, detects negative cycles | Θ(nm) | negative weights; distance-vector routing |
| Floyd–Warshall | any | Θ(n³) | all pairs, dense, small n |
| Johnson | any | Θ(nm + n² log n) | all pairs, sparse |
| A* | non-negative + heuristic | ≤ Dijkstra | with an admissible heuristic |
| Bidirectional | non-negative | ~√ of the search space | point-to-point on large graphs |

**Dijkstra's correctness** rests on the greedy argument of L03 §2.5: when a vertex is
extracted with minimal tentative distance, that distance is final, *because every remaining
path to it goes through some unextracted vertex whose distance is already ≥ it, and edges are
non-negative*. A negative edge breaks exactly that step.

Practical notes: Python's `heapq` has no decrease-key, so the standard implementation pushes
duplicates and skips stale entries on pop. This is Θ(m log m) rather than Θ(m + n log n), and
it is fine — a Fibonacci heap's theoretical advantage is not realized in practice.

**A\*** adds a heuristic `h(v)` estimating the distance to the target, and prioritizes by
`g(v) + h(v)`. It is optimal if `h` is **admissible** (never overestimates) and efficient if
`h` is **consistent** (`h(u) ≤ w(u,v) + h(v)`). With `h = 0` it *is* Dijkstra. The heuristic
is where the domain knowledge goes, and a good one turns an intractable search into an
instant one.

### 2.4 Connectivity

**Union–Find (disjoint set union).** With union by rank and path compression, m operations
on n elements take Θ(m α(n)) where α is the inverse Ackermann function — under 5 for any n
you will ever see, so effectively constant.

Uses: Kruskal's MST, dynamic connectivity, cycle detection in an undirected graph, image
segmentation, and — the one you will actually use — grouping related records. Implement it
once; it is fifteen lines and it is one of the highest value-per-line data structures in
existence.

**Strongly connected components.** A directed graph's maximal sets of mutually reachable
vertices. Tarjan's algorithm: Θ(n + m), one DFS.

The **condensation** — contract each SCC to a node — is always a DAG. That is the key
structural fact and it powers:

- **Module dependency analysis.** SCCs are the circular-dependency groups (SE-521 L02 §4).
  Reporting them is exactly what a "find the import cycles" tool does.
- **2-SAT.** A formula is satisfiable iff no variable and its negation are in the same SCC of
  the implication graph. Θ(n + m). Startlingly useful for constraint problems that look hard.
- **Deadlock detection.** A cycle in the wait-for graph (PY-601 L03 §2.4).

### 2.5 Flows and matchings

**Max flow.** Given a network with capacities, find the maximum flow from s to t.

- **Ford–Fulkerson / Edmonds–Karp** — Θ(nm²), simple, and the one to implement for
  understanding.
- **Dinic's** — Θ(n²m) generally, Θ(m√n) on unit-capacity networks. The practical choice.
- **Push–relabel** — Θ(n³) or better; fastest in practice for dense networks.

**Max-flow min-cut theorem.** The maximum flow equals the minimum capacity of a cut
separating s from t. This is the most useful duality in applied algorithms: a max-flow
algorithm *also* gives you the bottleneck, which is often what you actually wanted.

**Bipartite matching reduces to max flow.** Add a source connected to one side and a sink to
the other, all capacities 1; the max flow is the maximum matching. Θ(m√n) with Hopcroft–Karp.

This reduction is the single most useful one in this lesson, because bipartite matching
appears constantly and is almost never recognized:

- Assigning tasks to workers with skill constraints.
- Scheduling shifts.
- Matching orders to inventory.
- Assigning shards to nodes.
- Ad allocation.

**Min-cost max flow** handles the weighted version — assignment with costs. The **Hungarian
algorithm** solves the assignment problem directly in Θ(n³).

**König's theorem**: in a bipartite graph, maximum matching = minimum vertex cover. Another
duality that turns one problem into another you can solve.

### 2.6 Modelling: turning your problem into a graph

The skill. The procedure:

**1. What are the nodes?** Often not the obvious entities. For a scheduling problem the nodes
might be (task, time-slot) pairs. For a state-machine problem, the nodes are states. For a
puzzle, the nodes are configurations.

**2. What are the edges?** A relation, a transition, a compatibility, a dependency, a
conflict.

**3. What are the weights?** Cost, time, capacity, probability (use `−log p` to turn products
into sums, so shortest path finds the most likely sequence).

**4. What graph problem is this?** Match against the catalogue:

| If you want… | It is… |
|---|---|
| An order respecting dependencies | topological sort |
| To detect circular dependencies | cycle detection / SCC |
| Cheapest route | shortest path |
| To connect everything cheaply | minimum spanning tree |
| To pair up two groups | bipartite matching |
| Maximum throughput | max flow |
| The bottleneck | min cut |
| Groups of mutually reachable things | SCC |
| To split into two groups minimizing crossings | min cut / graph partitioning |
| Whether constraints are satisfiable (2 per clause) | 2-SAT via SCC |
| Most likely sequence of states | shortest path with −log probabilities |
| Reachability | BFS/DFS, or transitive closure |

**5. Check the size.** n and m determine whether Θ(n³) is acceptable. This is where a
beautiful model meets reality.

Two examples worth working through because they are non-obvious:

*Project selection.* Projects with profits, prerequisites with costs; choose a subset
maximizing profit. This is a **min-cut** problem (the "project selection" or "closure"
problem) — model it and the solution is immediate.

*Image segmentation.* Separating foreground from background with per-pixel likelihoods and
smoothness penalties is **min cut** on a grid graph. This is how graph-cut segmentation
works.

### 2.7 Practical graph engineering

- **Θ(n + m) is not fast if m is 10⁹.** Know your sizes. A "linear" algorithm on a billion
  edges is minutes, not milliseconds.
- **Memory dominates.** A billion edges as Python objects is impossible (PY-602 L04); as CSR
  int32 arrays it is 8 GB. The representation *is* the feasibility question.
- **Locality matters enormously.** Graph algorithms are the archetypal cache-hostile
  workload — pointer chasing by nature. Reordering vertices to improve locality (by BFS
  order, or a space-filling curve) gives real speedups. This is PY-602 L06 applied.
- **Dynamic graphs.** Most algorithms assume a static graph. If yours changes, either
  recompute (often fine), use an incremental algorithm, or use a data structure designed for
  it (link-cut trees, Euler tour trees). Recomputing is usually the right engineering answer.
- **Very large graphs** need external-memory or distributed algorithms: Pregel's
  vertex-centric model, GraphX, or simply a database with recursive CTEs. DI-721 L06.
- **Property-test your graph code.** Random graphs, verified against a brute-force reference,
  catches the off-by-one in your traversal that a hand-made example will not.

## 3. Construction: modelling and solving

**Exercise A — implement the core set**, with an iterative (non-recursive) DFS, and verify
each against `networkx` on 1,000 random graphs:

1. BFS and DFS, with edge classification.
2. Topological sort, with cycle detection reporting the cycle.
3. Union–Find with union by rank and path compression.
4. Dijkstra with `heapq` and stale-entry skipping.
5. Bellman–Ford with negative-cycle detection.
6. Tarjan's SCC.
7. Edmonds–Karp max flow, and the min cut it induces.
8. Bipartite matching via the max-flow reduction, and directly via Hopcroft–Karp.

State and verify the complexity of each empirically: run at increasing n and confirm the
growth matches.

**Exercise B — five modelling problems.** For each, apply §2.6's five steps, then implement:

1. **Deployment order.** Services with dependencies, some circular. Produce a deployment
   order, or report the circular groups. *(Topological sort on the SCC condensation.)*
2. **On-call assignment.** Engineers with availability and skills, shifts with requirements.
   Can every shift be covered? *(Bipartite matching.)*
3. **Capacity planning.** A network of services with per-link capacity; what is the maximum
   request rate from ingress to database, and what is the bottleneck? *(Max flow / min cut.)*
4. **Feature flag consistency.** Constraints of the form "if A then not B", "A or B". Is the
   configuration satisfiable? *(2-SAT via SCC.)*
5. **Cheapest cache warming.** Given items with dependencies and costs, and a time budget,
   which to warm? *(Depending on your formulation: shortest path on a DAG, or knapsack — work
   out which, and why.)*

For each: the model, the algorithm, the complexity, the implementation, and a test against a
brute-force reference at small sizes.

**Exercise C — the scale wall.** Take the largest graph you can obtain (a package dependency
graph, an import graph, a road network from OpenStreetMap, or a synthetic one). Measure, for
n and m increasing by powers of ten:

- Memory: `dict`-of-lists versus CSR arrays.
- Time: BFS in pure Python, with `networkx`, and with `scipy.sparse.csgraph`.
- Cache miss rate for the traversal (PY-602 L02 §2.4).

Report the crossover where each representation becomes necessary, and the memory ratio. Then
reorder the vertices in BFS order and re-measure the traversal — report the locality
improvement.

**Exercise D — A\* and the heuristic.** Implement Dijkstra and A\* on a road network (or a
grid with obstacles). Use straight-line distance as the heuristic. Measure nodes expanded and
wall time for both. Then deliberately use an *inadmissible* heuristic (multiply by 1.5) and
report: it is faster, and the paths are no longer optimal. Measure how suboptimal.

This is a concrete instance of a general trade — bounded suboptimality for speed — and it is
the bridge to L09.

## 4. Failure modes

- **Not recognizing a graph problem.** The expensive one.
- **BFS on a weighted graph.** Silently wrong; use Dijkstra.
- **Dijkstra with negative weights.** Silently wrong; use Bellman–Ford.
- **Recursive DFS on a deep graph.** `RecursionError` in production.
- **Θ(n²) adjacency matrix on a sparse graph.** Memory blowup.
- **Python objects for a large graph.** Use arrays.
- **Forgetting that `networkx` is slow.** Fine for correctness and exploration, wrong for
  scale.
- **An inadmissible A\* heuristic** presented as optimal.
- **Assuming a DAG.** Check for cycles; real dependency graphs have them.
- **Ignoring disconnected components.** Many algorithms need a loop over all vertices, not a
  single traversal from one source.
- **Multi-edges and self-loops** breaking an implementation that assumed a simple graph.
- **Recomputing from scratch on every change** when the graph is large and the change is
  small — or, equally, building a complex incremental structure when recomputation was fine.

## 5. Exercises

### Warm-up (30 min)

**W1.** Implement iterative DFS with edge classification. Use it to detect a cycle and report
the actual cycle, not just its existence.

**W2.** Implement Union–Find with union by rank and path compression in under 20 lines. Verify
Kruskal against `networkx`'s MST on 1,000 random weighted graphs.

**W3.** Demonstrate BFS giving a wrong answer on a weighted graph, and Dijkstra giving a wrong
answer with a negative edge. Identify the failing step in each.

### Core (3 h)

**C1 — The core set.** Complete Exercise A, all eight, with the `networkx` cross-verification
and the empirical complexity confirmation.

**C2 — Modelling.** Complete Exercise B, all five. Deliverable: the five-step model for each,
the implementations, the brute-force verification, and — the assessed part — a 600-word note
on which model was hardest to see and what made it click.

**C3 — Scale.** Complete Exercise C. Deliverable: the memory and time tables, the crossovers,
the cache-miss measurement, and the vertex-reordering result with the locality improvement
quantified.

**C4 — A\*.** Complete Exercise D. Report nodes expanded, wall time, and the suboptimality of
the inflated heuristic. Then write 300 words on where you would accept bounded suboptimality
in a real system.

### Challenge

**X1.** Implement Dinic's algorithm and use it for bipartite matching. Compare with
Edmonds–Karp and with Hopcroft–Karp on graphs of increasing size. Report the measured
complexities against the theoretical ones. Then apply it to a real assignment problem from
your work and report the result — including whether the optimal assignment differed
meaningfully from whatever heuristic is used today.

**X2.** Build a module-dependency analyser for a large codebase: parse the imports, build the
graph, find SCCs (circular dependency groups), compute the condensation, find the
topological layers, identify the vertices with highest afferent and efferent coupling
(SE-521 L02 §2.2), and produce a report ranking the "worst" cycles by size and by how many
modules depend on them. Run it on three real projects. This is a genuinely useful tool and it
is about 200 lines.

## 6. Self-check

1. Give four graph representations with their space and query costs, and say which is the
   default and why.
2. What does a back edge in a DFS tell you, and what three algorithms are built on the edge
   classification?
3. Give the shortest-path algorithm for each of: unweighted, non-negative, negative weights,
   all-pairs dense, and with a heuristic.
4. State the complexity of Union–Find with both optimizations, and three uses.
5. What is the condensation of a directed graph, and what is it always?
6. State max-flow min-cut and give the bipartite matching reduction.
7. Give the five modelling steps and five entries from the recognition table.
8. Why are graph algorithms cache-hostile, and what can you do about it?

## 7. Primary sources

- Kleinberg & Tardos, chs. 3 (traversal), 4 (MST, Dijkstra), 7 (network flow — an excellent
  chapter, especially the applications section).
- CLRS, 4th ed., part VI.
- Tarjan, "Depth-First Search and Linear Graph Algorithms" (SIAM J. Comput., 1972).
- Ford & Fulkerson (1956); Dinic (1970); Hopcroft & Karp (1973).
- Aspvall, Plass & Tarjan, "A Linear-Time Algorithm for Testing the Truth of Certain
  Quantified Boolean Formulas" (1979) — the 2-SAT result.
- Malewicz et al., "Pregel: A System for Large-Scale Graph Processing" (SIGMOD 2010).

---

**Previous:** [L04](L04-dynamic-programming.md) · **Next:**
[L06 — Randomized Algorithms and Concentration](L06-randomized-algorithms.md)
