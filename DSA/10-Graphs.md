---
tags:
  - dsa
  - computer-science
  - go
  - typescript
  - graphs
type: study-note
status: complete
date_created: 2026-09-28
---

# Graphs

## Overview

A graph is a set of vertices and a set of edges connecting them. That is it. No root, no parent-child rule, no acyclic requirement. Every structure in this vault is a special case: a linked list is a graph where each vertex has one outgoing edge, a tree is a connected graph with no cycles.

Because the definition is so loose, graphs model almost anything relational. Social networks, road maps, package dependencies, web links, state machines, task schedules. The payoff for learning graph representation is that a large number of unrelated-looking problems turn out to be the same graph problem.

## Key Concepts & Analogies

**A road map.** Cities are vertices, roads are edges. A one-way street is a directed edge. Distance is an edge weight. "Shortest route" is [[15-Shortest-Path-Dijkstras-Algorithm|Dijkstra]].

**A social network.** Friendship on Facebook is undirected (mutual). Following on Twitter is directed. "People you may know" is a two-hop traversal.

**Vocabulary:**
- Directed vs undirected: do edges have a direction.
- Weighted vs unweighted: do edges have a cost.
- Cyclic vs acyclic. A directed acyclic graph (DAG) is the shape of dependency problems.
- Connected: every vertex reachable from every other. For directed graphs, strongly connected means reachable following edge directions.
- Degree: number of incident edges. In-degree and out-degree for directed graphs.
- Dense vs sparse: does E approach V² or stay near V. This choice decides your representation.

**Two representations, and picking correctly matters:**

*Adjacency list* stores, per vertex, a list of its neighbors. Space is O(V + E). Iterating a vertex's neighbors is O(degree). Checking whether a specific edge exists is O(degree). This is the right default because real graphs are sparse.

*Adjacency matrix* is a V×V grid where `m[i][j]` holds the edge weight or zero. Space is O(V²) regardless of how few edges exist. Edge existence check is O(1). Iterating neighbors is O(V) even if there is one. Worth it only for dense graphs or when you need constant-time edge lookup.

For a social network with a million users averaging 200 friends: adjacency list holds 200 million entries, matrix holds a trillion. The choice is not close.

*Edge list* is just a list of (u, v, weight) triples. Useless for traversal, perfect for Kruskal's MST algorithm, which sorts edges by weight.

**Self-loops and parallel edges.** A multigraph allows more than one edge between the same pair. Most algorithms assume simple graphs and quietly break on multigraphs, so normalize your input.

**Cycle detection differs by direction.** In an undirected graph, finding an already-visited vertex that is not your parent means a cycle. In a directed graph you need three colors (white/gray/black) because revisiting a finished vertex is fine but revisiting one still on the stack is a back edge. See [[14-Graph-Traversal-DFS-BFS]].

**Union-Find (disjoint set union) answers connectivity fast.** With path compression and union by rank, each operation is effectively O(1) (formally O(α(n)), inverse Ackermann, which is under 5 for any n you will ever see). It is the tool for connected components under incremental edge addition, and for Kruskal's MST.

## Operations & Time/Space Complexity

Adjacency list, with V vertices and E edges:

| Operation | Average Case | Worst Case | Space Complexity |
| :--- | :--- | :--- | :--- |
| Add vertex | O(1) | O(1) | O(1) |
| Add edge | O(1) | O(1) | O(1) |
| Remove edge | O(degree) | O(V) | O(1) |
| Has edge | O(degree) | O(V) | O(1) |
| Iterate neighbors | O(degree) | O(V) | O(1) |
| Full traversal (DFS/BFS) | O(V + E) | O(V + E) | O(V) |
| Total storage | O(V + E) | O(V + E) | O(V + E) |

Adjacency matrix for comparison: has-edge and remove-edge drop to O(1), iterate-neighbors rises to O(V), traversal becomes O(V²), storage becomes O(V²).

Union-Find: find and union are O(α(n)) amortized, storage O(V).

## Code Implementations

### 1. Go (Golang)

```go
package main

import (
	"fmt"
	"sort"
)

// Graph is an adjacency-list graph with optional edge weights.
// Using a map per vertex makes HasEdge O(1) while keeping O(V+E) space.
type Graph[T comparable] struct {
	adj      map[T]map[T]int // vertex -> neighbor -> weight
	directed bool
}

func NewGraph[T comparable](directed bool) *Graph[T] {
	return &Graph[T]{adj: make(map[T]map[T]int), directed: directed}
}

func (g *Graph[T]) AddVertex(v T) {
	if _, ok := g.adj[v]; !ok {
		g.adj[v] = make(map[T]int)
	}
}

// AddEdge inserts both directions when the graph is undirected.
func (g *Graph[T]) AddEdge(u, v T, weight int) {
	g.AddVertex(u)
	g.AddVertex(v)
	g.adj[u][v] = weight
	if !g.directed {
		g.adj[v][u] = weight
	}
}

func (g *Graph[T]) RemoveEdge(u, v T) {
	delete(g.adj[u], v)
	if !g.directed {
		delete(g.adj[v], u)
	}
}

func (g *Graph[T]) HasEdge(u, v T) bool {
	_, ok := g.adj[u][v]
	return ok
}

func (g *Graph[T]) Weight(u, v T) (int, bool) {
	w, ok := g.adj[u][v]
	return w, ok
}

func (g *Graph[T]) Neighbors(v T) map[T]int { return g.adj[v] }
func (g *Graph[T]) VertexCount() int        { return len(g.adj) }

func (g *Graph[T]) EdgeCount() int {
	total := 0
	for _, nbrs := range g.adj {
		total += len(nbrs)
	}
	if !g.directed {
		return total / 2 // every edge was counted twice
	}
	return total
}

func (g *Graph[T]) Vertices() []T {
	out := make([]T, 0, len(g.adj))
	for v := range g.adj {
		out = append(out, v)
	}
	return out
}

// Degree counts outgoing edges.
func (g *Graph[T]) Degree(v T) int { return len(g.adj[v]) }

// InDegree requires scanning every adjacency list. Keep a reverse index
// if you need this in a hot loop.
func (g *Graph[T]) InDegree(v T) int {
	count := 0
	for _, nbrs := range g.adj {
		if _, ok := nbrs[v]; ok {
			count++
		}
	}
	return count
}

// --- Adjacency matrix, for dense graphs ---

// MatrixGraph trades O(V^2) space for O(1) edge lookup.
type MatrixGraph struct {
	n    int
	m    [][]int
	has  [][]bool // distinguishes "weight 0" from "no edge"
	dirn bool
}

func NewMatrixGraph(n int, directed bool) *MatrixGraph {
	m := make([][]int, n)
	has := make([][]bool, n)
	for i := range m {
		m[i] = make([]int, n)
		has[i] = make([]bool, n)
	}
	return &MatrixGraph{n: n, m: m, has: has, dirn: directed}
}

func (g *MatrixGraph) AddEdge(u, v, w int) {
	g.m[u][v], g.has[u][v] = w, true
	if !g.dirn {
		g.m[v][u], g.has[v][u] = w, true
	}
}

func (g *MatrixGraph) HasEdge(u, v int) bool { return g.has[u][v] } // O(1)

func (g *MatrixGraph) Neighbors(u int) []int { // O(V) even for one neighbor
	var out []int
	for v := 0; v < g.n; v++ {
		if g.has[u][v] {
			out = append(out, v)
		}
	}
	return out
}

// --- Union-Find: connectivity in near-constant time ---

// UnionFind with path compression and union by rank. Each operation is
// O(alpha(n)), which is effectively constant.
type UnionFind[T comparable] struct {
	parent map[T]T
	rank   map[T]int
	count  int // number of disjoint components
}

func NewUnionFind[T comparable](items ...T) *UnionFind[T] {
	uf := &UnionFind[T]{parent: make(map[T]T), rank: make(map[T]int)}
	for _, it := range items {
		uf.Add(it)
	}
	return uf
}

func (uf *UnionFind[T]) Add(x T) {
	if _, ok := uf.parent[x]; !ok {
		uf.parent[x] = x
		uf.count++
	}
}

// Find flattens the path to the root as it walks, so repeated queries
// get cheaper over time.
func (uf *UnionFind[T]) Find(x T) T {
	for uf.parent[x] != x {
		uf.parent[x] = uf.parent[uf.parent[x]] // path halving
		x = uf.parent[x]
	}
	return x
}

// Union attaches the shorter tree under the taller one to keep depth low.
func (uf *UnionFind[T]) Union(a, b T) bool {
	ra, rb := uf.Find(a), uf.Find(b)
	if ra == rb {
		return false // already together; in Kruskal this means a cycle
	}
	if uf.rank[ra] < uf.rank[rb] {
		ra, rb = rb, ra
	}
	uf.parent[rb] = ra
	if uf.rank[ra] == uf.rank[rb] {
		uf.rank[ra]++
	}
	uf.count--
	return true
}

func (uf *UnionFind[T]) Connected(a, b T) bool { return uf.Find(a) == uf.Find(b) }
func (uf *UnionFind[T]) Components() int       { return uf.count }

// --- Kruskal's MST: sort edges, add any that does not form a cycle ---

type Edge[T comparable] struct {
	U, V   T
	Weight int
}

// MinimumSpanningTree is Kruskal's algorithm. O(E log E) from the sort.
// Union-Find is what makes the cycle check cheap.
func MinimumSpanningTree[T comparable](vertices []T, edges []Edge[T]) ([]Edge[T], int) {
	sorted := make([]Edge[T], len(edges))
	copy(sorted, edges)
	sort.Slice(sorted, func(i, j int) bool { return sorted[i].Weight < sorted[j].Weight })

	uf := NewUnionFind(vertices...)
	var tree []Edge[T]
	total := 0
	for _, e := range sorted {
		if uf.Union(e.U, e.V) { // false means both ends already connected
			tree = append(tree, e)
			total += e.Weight
			if len(tree) == len(vertices)-1 {
				break // a spanning tree has exactly V-1 edges
			}
		}
	}
	return tree, total
}

func main() {
	g := NewGraph[string](false)
	g.AddEdge("Dhaka", "Chittagong", 250)
	g.AddEdge("Dhaka", "Sylhet", 240)
	g.AddEdge("Chittagong", "Coxs Bazar", 150)
	g.AddEdge("Sylhet", "Chittagong", 330)

	fmt.Println(g.VertexCount(), g.EdgeCount()) // 4 4
	fmt.Println(g.HasEdge("Dhaka", "Sylhet"))   // true
	fmt.Println(g.Degree("Dhaka"))              // 2

	vertices := []string{"A", "B", "C", "D"}
	edges := []Edge[string]{
		{"A", "B", 1}, {"B", "C", 2}, {"A", "C", 4}, {"C", "D", 3}, {"B", "D", 5},
	}
	tree, cost := MinimumSpanningTree(vertices, edges)
	fmt.Println(tree, cost) // 3 edges, total 6

	uf := NewUnionFind(1, 2, 3, 4, 5)
	uf.Union(1, 2)
	uf.Union(3, 4)
	fmt.Println(uf.Connected(1, 2), uf.Connected(1, 3), uf.Components())
	// true false 3
}
```

### 2. TypeScript

```typescript
/**
 * Adjacency-list graph. The inner Map gives O(1) edge lookup while
 * keeping total space at O(V + E).
 */
class Graph<T> {
  readonly #adj = new Map<T, Map<T, number>>();
  readonly #directed: boolean;

  constructor(directed = false) {
    this.#directed = directed;
  }

  get vertexCount(): number { return this.#adj.size; }

  get edgeCount(): number {
    let total = 0;
    for (const nbrs of this.#adj.values()) total += nbrs.size;
    return this.#directed ? total : total / 2;
  }

  addVertex(v: T): this {
    if (!this.#adj.has(v)) this.#adj.set(v, new Map());
    return this;
  }

  addEdge(u: T, v: T, weight = 1): this {
    this.addVertex(u).addVertex(v);
    this.#adj.get(u)!.set(v, weight);
    if (!this.#directed) this.#adj.get(v)!.set(u, weight);
    return this;
  }

  removeEdge(u: T, v: T): boolean {
    const removed = this.#adj.get(u)?.delete(v) ?? false;
    if (!this.#directed) this.#adj.get(v)?.delete(u);
    return removed;
  }

  hasEdge(u: T, v: T): boolean {
    return this.#adj.get(u)?.has(v) ?? false;
  }

  weight(u: T, v: T): number | undefined {
    return this.#adj.get(u)?.get(v);
  }

  neighbors(v: T): ReadonlyMap<T, number> {
    return this.#adj.get(v) ?? new Map();
  }

  vertices(): T[] { return [...this.#adj.keys()]; }

  degree(v: T): number { return this.#adj.get(v)?.size ?? 0; }

  /** Requires a full scan. Cache a reverse index if this is hot. */
  inDegree(v: T): number {
    let count = 0;
    for (const nbrs of this.#adj.values()) if (nbrs.has(v)) count++;
    return count;
  }

  /** Flat edge list, the input Kruskal wants. */
  edges(): { u: T; v: T; weight: number }[] {
    const out: { u: T; v: T; weight: number }[] = [];
    const seen = new Set<string>();
    for (const [u, nbrs] of this.#adj) {
      for (const [v, weight] of nbrs) {
        if (!this.#directed) {
          const key = [String(u), String(v)].sort().join("\u0000");
          if (seen.has(key)) continue;
          seen.add(key);
        }
        out.push({ u, v, weight });
      }
    }
    return out;
  }
}

/**
 * Adjacency matrix. O(V^2) space, O(1) edge checks.
 * Worth it only when the graph is dense.
 */
class MatrixGraph {
  readonly #weights: number[][];
  readonly #present: boolean[][];

  constructor(readonly size: number, private readonly directed = false) {
    this.#weights = Array.from({ length: size }, () => new Array(size).fill(0));
    this.#present = Array.from({ length: size }, () => new Array(size).fill(false));
  }

  addEdge(u: number, v: number, weight = 1): void {
    this.#weights[u][v] = weight;
    this.#present[u][v] = true;
    if (!this.directed) {
      this.#weights[v][u] = weight;
      this.#present[v][u] = true;
    }
  }

  hasEdge(u: number, v: number): boolean { return this.#present[u][v]; } // O(1)

  neighbors(u: number): number[] {  // O(V) regardless of actual degree
    const out: number[] = [];
    for (let v = 0; v < this.size; v++) if (this.#present[u][v]) out.push(v);
    return out;
  }
}

/** Path compression plus union by rank. Effectively O(1) per operation. */
class UnionFind<T> {
  readonly #parent = new Map<T, T>();
  readonly #rank = new Map<T, number>();
  #components = 0;

  constructor(items: Iterable<T> = []) {
    for (const item of items) this.add(item);
  }

  get components(): number { return this.#components; }

  add(x: T): void {
    if (!this.#parent.has(x)) {
      this.#parent.set(x, x);
      this.#rank.set(x, 0);
      this.#components++;
    }
  }

  /** Flattens the path while walking, so later finds are cheaper. */
  find(x: T): T {
    let root = x;
    while (this.#parent.get(root) !== root) {
      const grandparent = this.#parent.get(this.#parent.get(root)!)!;
      this.#parent.set(root, grandparent); // path halving
      root = grandparent;
    }
    return root;
  }

  /** Returns false when both were already in the same set (a cycle). */
  union(a: T, b: T): boolean {
    let ra = this.find(a);
    let rb = this.find(b);
    if (ra === rb) return false;
    if (this.#rank.get(ra)! < this.#rank.get(rb)!) [ra, rb] = [rb, ra];
    this.#parent.set(rb, ra);
    if (this.#rank.get(ra) === this.#rank.get(rb)) {
      this.#rank.set(ra, this.#rank.get(ra)! + 1);
    }
    this.#components--;
    return true;
  }

  connected(a: T, b: T): boolean { return this.find(a) === this.find(b); }
}

interface WeightedEdge<T> { u: T; v: T; weight: number }

/** Kruskal's MST. O(E log E), dominated by the sort. */
function minimumSpanningTree<T>(
  vertices: readonly T[],
  edges: readonly WeightedEdge<T>[],
): { tree: WeightedEdge<T>[]; cost: number } {
  const sorted = [...edges].sort((a, b) => a.weight - b.weight);
  const uf = new UnionFind(vertices);
  const tree: WeightedEdge<T>[] = [];
  let cost = 0;

  for (const edge of sorted) {
    if (uf.union(edge.u, edge.v)) {
      tree.push(edge);
      cost += edge.weight;
      if (tree.length === vertices.length - 1) break; // spanning tree complete
    }
  }
  return { tree, cost };
}

const g = new Graph<string>();
g.addEdge("Dhaka", "Chittagong", 250)
 .addEdge("Dhaka", "Sylhet", 240)
 .addEdge("Chittagong", "Coxs Bazar", 150)
 .addEdge("Sylhet", "Chittagong", 330);

console.log(g.vertexCount, g.edgeCount);        // 4 4
console.log(g.hasEdge("Dhaka", "Sylhet"));      // true
console.log([...g.neighbors("Dhaka")]);         // [["Chittagong",250],["Sylhet",240]]

const mst = minimumSpanningTree(["A", "B", "C", "D"], [
  { u: "A", v: "B", weight: 1 },
  { u: "B", v: "C", weight: 2 },
  { u: "A", v: "C", weight: 4 },
  { u: "C", v: "D", weight: 3 },
  { u: "B", v: "D", weight: 5 },
]);
console.log(mst.cost); // 6

const uf = new UnionFind([1, 2, 3, 4, 5]);
uf.union(1, 2);
uf.union(3, 4);
console.log(uf.connected(1, 2), uf.connected(1, 3), uf.components); // true false 3
```

## Practical Applications & Real-World Use Cases

- **Navigation.** Google Maps is a weighted directed graph where weights update with live traffic. Route finding is shortest path. See [[15-Shortest-Path-Dijkstras-Algorithm]].
- **Package managers and build systems.** `npm`, `go mod`, Bazel, and Make all build a DAG of dependencies and topologically sort it. A cycle is a hard error and detecting it is a graph problem.
- **Social networks.** Friend recommendations are two-hop neighborhoods. Community detection is clustering on the graph. Degrees of separation is BFS.
- **PageRank.** The web is a directed graph of links, and PageRank is the stationary distribution of a random walk on it. Google was built on a graph algorithm.
- **Network routing.** OSPF runs Dijkstra across routers; BGP runs a path-vector algorithm between autonomous systems.
- **Compiler optimization.** Control-flow graphs drive dead-code elimination; register allocation is graph coloring.
- **Recommendation engines.** A bipartite graph of users and items, with collaborative filtering as traversal.
- **Fraud and anomaly detection.** Rings of colluding accounts show up as unusually dense subgraphs.

## Related Concepts & Connections

- [[00-DSA-Index|Back to DSA Index]]
- [[14-Graph-Traversal-DFS-BFS]] for the two traversals every graph algorithm builds on
- [[15-Shortest-Path-Dijkstras-Algorithm]] for weighted shortest paths
- [[08-Trees-BST-and-AVL]] because a tree is a connected acyclic graph
- [[05-Queues]] and [[04-Stacks]], the structures that distinguish BFS from DFS
- [[06-Priority-Queue-and-Heap]] for the heap Dijkstra and Prim both need
- [[07-Hash-Tables-and-Sets]] for the visited set and the adjacency maps
