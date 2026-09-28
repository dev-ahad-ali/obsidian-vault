---
tags:
  - dsa
  - computer-science
  - go
  - typescript
  - shortest-path
type: study-note
status: complete
date_created: 2026-09-28
---

# Shortest Path: Dijkstra's Algorithm

## Overview

Dijkstra's algorithm finds the shortest path from one source vertex to every other vertex in a graph with non-negative edge weights. Published by Edsger Dijkstra in 1959, reportedly designed in about twenty minutes over coffee while thinking about the shortest route between two Dutch cities.

The core idea is greedy: always expand the unvisited vertex with the smallest known distance. Because weights are non-negative, once you pop a vertex from the frontier you can prove no shorter route to it exists, so its distance is final. That proof is the whole algorithm, and it is also the reason negative weights break it.

Think of it as BFS where the queue is replaced by a min-heap keyed on cumulative distance. See [[14-Graph-Traversal-DFS-BFS]].

## Key Concepts & Analogies

**Water spreading through a pipe network at constant speed.** The wavefront reaches nearby junctions first. When water first arrives at a junction, it came by the fastest route by definition.

**Relaxation is the single operation.** For an edge u → v with weight w, if `dist[u] + w < dist[v]`, you found a better route, so update `dist[v]` and record u as v's predecessor. Every shortest-path algorithm is a different schedule of relaxations. Dijkstra relaxes in order of increasing distance; Bellman-Ford relaxes everything V-1 times.

**Why negative weights break it.** Dijkstra finalizes a vertex when it is popped. A negative edge discovered later could make an already-finalized path shorter, and the algorithm never revisits it. Bellman-Ford handles negative weights in O(V·E) and also detects negative cycles, where "shortest path" stops being meaningful because you can loop forever getting cheaper.

**The heap is what makes it fast.** The original formulation scans all vertices to find the minimum, which is O(V²). Fine for dense graphs. With a binary heap it becomes O((V + E) log V), which is dramatically better on sparse graphs and is what you should implement. See [[06-Priority-Queue-and-Heap]].

**Lazy deletion instead of decrease-key.** Textbook Dijkstra calls decrease-key when it finds a better distance, which requires tracking each vertex's heap position. The practical version just pushes a new entry and skips stale ones on pop (`if d > dist[v] { continue }`). The heap grows to O(E) instead of O(V) and the asymptotics are unchanged. Every real implementation does this because it is much simpler.

**Reconstructing the path** needs a predecessor map. Store `prev[v] = u` on every successful relaxation, then walk backward from the target and reverse. Storing only distances tells you how far, not how.

**Early exit for single-target queries.** If you only need one destination, return when you pop it. On a road network that can cut the work enormously.

**A\* is Dijkstra with a heuristic.** Prioritize by `dist[v] + h(v)` where `h` estimates the remaining distance to the goal. If `h` never overestimates (admissible), A\* is still optimal but explores far less. With `h = 0` it is exactly Dijkstra. Straight-line distance is the standard heuristic for maps, and it is what makes game pathfinding practical.

**Algorithm selection:**
- Unweighted graph: BFS, O(V + E). Do not reach for Dijkstra.
- Non-negative weights, one source: Dijkstra.
- Negative weights: Bellman-Ford, O(V·E).
- All pairs, dense graph: Floyd-Warshall, O(V³) and about ten lines.
- Known goal plus a good heuristic: A\*.

## Operations & Time/Space Complexity

| Algorithm | Time | Space | Negative weights | Scope |
| :--- | :--- | :--- | :--- | :--- |
| BFS (unweighted) | O(V + E) | O(V) | n/a | Single source |
| Dijkstra (array scan) | O(V²) | O(V) | No | Single source |
| Dijkstra (binary heap) | O((V + E) log V) | O(V + E) | No | Single source |
| Dijkstra (Fibonacci heap) | O(E + V log V) | O(V) | No | Single source |
| Bellman-Ford | O(V·E) | O(V) | Yes, detects cycles | Single source |
| Floyd-Warshall | O(V³) | O(V²) | Yes, no neg cycles | All pairs |
| A\* | O(E) best case | O(V) | No | Single pair |

The Fibonacci heap bound is better on paper and slower in practice because of its constants. Nobody ships it.

## Code Implementations

### 1. Go (Golang)

```go
package main

import (
	"container/heap"
	"fmt"
	"math"
)

type Edge struct {
	To     string
	Weight int
}

type WeightedGraph struct {
	adj      map[string][]Edge
	directed bool
}

func NewWeightedGraph(directed bool) *WeightedGraph {
	return &WeightedGraph{adj: make(map[string][]Edge), directed: directed}
}

func (g *WeightedGraph) AddEdge(from, to string, weight int) {
	g.adj[from] = append(g.adj[from], Edge{To: to, Weight: weight})
	if _, ok := g.adj[to]; !ok {
		g.adj[to] = nil
	}
	if !g.directed {
		g.adj[to] = append(g.adj[to], Edge{To: from, Weight: weight})
	}
}

func (g *WeightedGraph) Vertices() []string {
	out := make([]string, 0, len(g.adj))
	for v := range g.adj {
		out = append(out, v)
	}
	return out
}

// item is a heap entry. Stale entries are tolerated and skipped on pop,
// which avoids needing decrease-key.
type item struct {
	vertex string
	dist   int
}

type minHeap []item

func (h minHeap) Len() int            { return len(h) }
func (h minHeap) Less(i, j int) bool  { return h[i].dist < h[j].dist }
func (h minHeap) Swap(i, j int)       { h[i], h[j] = h[j], h[i] }
func (h *minHeap) Push(x any)         { *h = append(*h, x.(item)) }
func (h *minHeap) Pop() any {
	old := *h
	n := len(old)
	v := old[n-1]
	*h = old[:n-1]
	return v
}

// Dijkstra returns shortest distances from source and the predecessor
// map for path reconstruction. O((V+E) log V).
func (g *WeightedGraph) Dijkstra(source string) (map[string]int, map[string]string) {
	dist := make(map[string]int, len(g.adj))
	prev := make(map[string]string, len(g.adj))
	for _, v := range g.Vertices() {
		dist[v] = math.MaxInt
	}
	dist[source] = 0

	pq := &minHeap{{vertex: source, dist: 0}}
	heap.Init(pq)

	for pq.Len() > 0 {
		cur := heap.Pop(pq).(item)

		// Stale entry: we already found a better route to this vertex.
		if cur.dist > dist[cur.vertex] {
			continue
		}

		for _, e := range g.adj[cur.vertex] {
			// Relaxation: is going through cur.vertex better?
			if candidate := cur.dist + e.Weight; candidate < dist[e.To] {
				dist[e.To] = candidate
				prev[e.To] = cur.vertex
				heap.Push(pq, item{vertex: e.To, dist: candidate})
			}
		}
	}
	return dist, prev
}

// ShortestPath walks the predecessor map backward from target.
func (g *WeightedGraph) ShortestPath(source, target string) ([]string, int) {
	dist, prev := g.Dijkstra(source)
	if dist[target] == math.MaxInt {
		return nil, -1 // unreachable
	}
	var path []string
	for at := target; at != ""; at = prev[at] {
		path = append(path, at)
		if at == source {
			break
		}
	}
	for l, r := 0, len(path)-1; l < r; l, r = l+1, r-1 {
		path[l], path[r] = path[r], path[l]
	}
	return path, dist[target]
}

// DijkstraToTarget exits as soon as the target is finalized. On large
// graphs this is much less work than computing every distance.
func (g *WeightedGraph) DijkstraToTarget(source, target string) int {
	dist := map[string]int{source: 0}
	pq := &minHeap{{vertex: source, dist: 0}}
	heap.Init(pq)

	for pq.Len() > 0 {
		cur := heap.Pop(pq).(item)
		if cur.vertex == target {
			return cur.dist // popped means finalized, so this is the answer
		}
		if d, ok := dist[cur.vertex]; ok && cur.dist > d {
			continue
		}
		for _, e := range g.adj[cur.vertex] {
			candidate := cur.dist + e.Weight
			if d, ok := dist[e.To]; !ok || candidate < d {
				dist[e.To] = candidate
				heap.Push(pq, item{vertex: e.To, dist: candidate})
			}
		}
	}
	return -1
}

// BellmanFord handles negative weights and reports negative cycles.
// O(V*E): relax every edge V-1 times, because a shortest path uses at
// most V-1 edges.
func (g *WeightedGraph) BellmanFord(source string) (map[string]int, error) {
	dist := make(map[string]int)
	vertices := g.Vertices()
	for _, v := range vertices {
		dist[v] = math.MaxInt
	}
	dist[source] = 0

	for i := 0; i < len(vertices)-1; i++ {
		changed := false
		for _, u := range vertices {
			if dist[u] == math.MaxInt {
				continue
			}
			for _, e := range g.adj[u] {
				if dist[u]+e.Weight < dist[e.To] {
					dist[e.To] = dist[u] + e.Weight
					changed = true
				}
			}
		}
		if !changed {
			break // converged early
		}
	}

	// A V-th improvement is only possible if a negative cycle exists.
	for _, u := range vertices {
		if dist[u] == math.MaxInt {
			continue
		}
		for _, e := range g.adj[u] {
			if dist[u]+e.Weight < dist[e.To] {
				return nil, fmt.Errorf("negative cycle reachable from %s", source)
			}
		}
	}
	return dist, nil
}

// FloydWarshall computes all-pairs shortest paths. O(V^3) but the inner
// loop is three lines and it handles negative edges.
func (g *WeightedGraph) FloydWarshall() map[string]map[string]int {
	vertices := g.Vertices()
	const inf = math.MaxInt / 4 // headroom so additions do not overflow

	dist := make(map[string]map[string]int, len(vertices))
	for _, u := range vertices {
		dist[u] = make(map[string]int, len(vertices))
		for _, v := range vertices {
			if u == v {
				dist[u][v] = 0
			} else {
				dist[u][v] = inf
			}
		}
	}
	for _, u := range vertices {
		for _, e := range g.adj[u] {
			if e.Weight < dist[u][e.To] {
				dist[u][e.To] = e.Weight
			}
		}
	}

	// k must be the outer loop: "paths using only {v1..vk} as intermediates".
	for _, k := range vertices {
		for _, i := range vertices {
			for _, j := range vertices {
				if through := dist[i][k] + dist[k][j]; through < dist[i][j] {
					dist[i][j] = through
				}
			}
		}
	}
	return dist
}

func main() {
	roads := NewWeightedGraph(false)
	roads.AddEdge("Dhaka", "Tangail", 98)
	roads.AddEdge("Dhaka", "Comilla", 97)
	roads.AddEdge("Tangail", "Bogura", 120)
	roads.AddEdge("Comilla", "Sylhet", 200)
	roads.AddEdge("Bogura", "Rangpur", 145)
	roads.AddEdge("Sylhet", "Rangpur", 420)
	roads.AddEdge("Dhaka", "Khulna", 270)

	dist, _ := roads.Dijkstra("Dhaka")
	for _, city := range []string{"Rangpur", "Sylhet", "Khulna"} {
		fmt.Printf("Dhaka -> %-8s %d km\n", city, dist[city])
	}

	path, total := roads.ShortestPath("Dhaka", "Rangpur")
	fmt.Println(path, total) // [Dhaka Tangail Bogura Rangpur] 363

	fmt.Println(roads.DijkstraToTarget("Dhaka", "Sylhet")) // 297

	// Negative weights: Dijkstra would be wrong, Bellman-Ford is right.
	neg := NewWeightedGraph(true)
	neg.AddEdge("a", "b", 4)
	neg.AddEdge("a", "c", 5)
	neg.AddEdge("b", "d", 3)
	neg.AddEdge("c", "b", -3)
	d, err := neg.BellmanFord("a")
	fmt.Println(d["d"], err) // 5, no error: a->c->b->d costs 5-3+3

	all := roads.FloydWarshall()
	fmt.Println(all["Khulna"]["Rangpur"])
}
```

### 2. TypeScript

```typescript
interface Edge<T> { to: T; weight: number }

/** Min-heap over {vertex, dist}. Stale entries get skipped on pop. */
class MinHeap<T> {
  #items: { vertex: T; dist: number }[] = [];

  get size(): number { return this.#items.length; }

  push(vertex: T, dist: number): void {
    this.#items.push({ vertex, dist });
    let i = this.#items.length - 1;
    while (i > 0) {
      const p = (i - 1) >> 1;
      if (this.#items[p].dist <= this.#items[i].dist) break;
      [this.#items[i], this.#items[p]] = [this.#items[p], this.#items[i]];
      i = p;
    }
  }

  pop(): { vertex: T; dist: number } | undefined {
    if (this.#items.length === 0) return undefined;
    const top = this.#items[0];
    const last = this.#items.pop()!;
    if (this.#items.length > 0) {
      this.#items[0] = last;
      let i = 0;
      for (;;) {
        let best = i;
        const l = 2 * i + 1;
        const r = 2 * i + 2;
        if (l < this.#items.length && this.#items[l].dist < this.#items[best].dist) best = l;
        if (r < this.#items.length && this.#items[r].dist < this.#items[best].dist) best = r;
        if (best === i) break;
        [this.#items[i], this.#items[best]] = [this.#items[best], this.#items[i]];
        i = best;
      }
    }
    return top;
  }
}

class WeightedGraph<T> {
  readonly #adj = new Map<T, Edge<T>[]>();
  readonly #directed: boolean;

  constructor(directed = false) {
    this.#directed = directed;
  }

  addEdge(from: T, to: T, weight: number): this {
    if (weight < 0 && !this.#directed) {
      throw new RangeError("negative weights make no sense on undirected edges");
    }
    if (!this.#adj.has(from)) this.#adj.set(from, []);
    if (!this.#adj.has(to)) this.#adj.set(to, []);
    this.#adj.get(from)!.push({ to, weight });
    if (!this.#directed) this.#adj.get(to)!.push({ to: from, weight });
    return this;
  }

  neighbors(v: T): readonly Edge<T>[] { return this.#adj.get(v) ?? []; }
  vertices(): T[] { return [...this.#adj.keys()]; }

  /**
   * Dijkstra with a binary heap and lazy deletion.
   * O((V + E) log V). Requires non-negative weights.
   */
  dijkstra(source: T): { dist: Map<T, number>; prev: Map<T, T> } {
    const dist = new Map<T, number>();
    const prev = new Map<T, T>();
    for (const v of this.vertices()) dist.set(v, Infinity);
    dist.set(source, 0);

    const pq = new MinHeap<T>();
    pq.push(source, 0);

    while (pq.size > 0) {
      const { vertex, dist: d } = pq.pop()!;
      if (d > (dist.get(vertex) ?? Infinity)) continue; // stale entry

      for (const edge of this.neighbors(vertex)) {
        const candidate = d + edge.weight;
        if (candidate < (dist.get(edge.to) ?? Infinity)) {
          dist.set(edge.to, candidate);       // relaxation
          prev.set(edge.to, vertex);
          pq.push(edge.to, candidate);
        }
      }
    }
    return { dist, prev };
  }

  /** Walks the predecessor map backward, then reverses. */
  shortestPath(source: T, target: T): { path: T[]; cost: number } | null {
    const { dist, prev } = this.dijkstra(source);
    const cost = dist.get(target);
    if (cost === undefined || cost === Infinity) return null;

    const path: T[] = [];
    for (let at: T | undefined = target; at !== undefined; at = prev.get(at)) {
      path.push(at);
      if (at === source) break;
    }
    return { path: path.reverse(), cost };
  }

  /** Stops the moment the target is finalized. */
  distanceTo(source: T, target: T): number {
    const dist = new Map<T, number>([[source, 0]]);
    const pq = new MinHeap<T>();
    pq.push(source, 0);

    while (pq.size > 0) {
      const { vertex, dist: d } = pq.pop()!;
      if (vertex === target) return d; // popped implies finalized
      if (d > (dist.get(vertex) ?? Infinity)) continue;
      for (const edge of this.neighbors(vertex)) {
        const candidate = d + edge.weight;
        if (candidate < (dist.get(edge.to) ?? Infinity)) {
          dist.set(edge.to, candidate);
          pq.push(edge.to, candidate);
        }
      }
    }
    return Infinity;
  }

  /**
   * A* : Dijkstra prioritized by dist + heuristic. Optimal as long as
   * the heuristic never overestimates the true remaining distance.
   */
  aStar(source: T, target: T, heuristic: (v: T) => number): { path: T[]; cost: number } | null {
    const gScore = new Map<T, number>([[source, 0]]);
    const prev = new Map<T, T>();
    const open = new MinHeap<T>();
    open.push(source, heuristic(source));
    const closed = new Set<T>();

    while (open.size > 0) {
      const { vertex } = open.pop()!;
      if (vertex === target) {
        const path: T[] = [];
        for (let at: T | undefined = vertex; at !== undefined; at = prev.get(at)) {
          path.push(at);
          if (at === source) break;
        }
        return { path: path.reverse(), cost: gScore.get(target)! };
      }
      if (closed.has(vertex)) continue;
      closed.add(vertex);

      for (const edge of this.neighbors(vertex)) {
        const candidate = gScore.get(vertex)! + edge.weight;
        if (candidate < (gScore.get(edge.to) ?? Infinity)) {
          gScore.set(edge.to, candidate);
          prev.set(edge.to, vertex);
          open.push(edge.to, candidate + heuristic(edge.to)); // f = g + h
        }
      }
    }
    return null;
  }

  /**
   * Bellman-Ford. Handles negative weights, throws on negative cycles.
   * O(V*E) because a shortest path uses at most V-1 edges.
   */
  bellmanFord(source: T): Map<T, number> {
    const dist = new Map<T, number>();
    const vertices = this.vertices();
    for (const v of vertices) dist.set(v, Infinity);
    dist.set(source, 0);

    for (let i = 0; i < vertices.length - 1; i++) {
      let changed = false;
      for (const u of vertices) {
        const du = dist.get(u)!;
        if (du === Infinity) continue;
        for (const edge of this.neighbors(u)) {
          if (du + edge.weight < dist.get(edge.to)!) {
            dist.set(edge.to, du + edge.weight);
            changed = true;
          }
        }
      }
      if (!changed) break; // converged
    }

    // Any further improvement proves a negative cycle.
    for (const u of vertices) {
      const du = dist.get(u)!;
      if (du === Infinity) continue;
      for (const edge of this.neighbors(u)) {
        if (du + edge.weight < dist.get(edge.to)!) {
          throw new Error(`negative cycle reachable from ${String(source)}`);
        }
      }
    }
    return dist;
  }

  /** All pairs, O(V^3). Short enough to be worth it when V is small. */
  floydWarshall(): Map<T, Map<T, number>> {
    const vertices = this.vertices();
    const dist = new Map<T, Map<T, number>>();

    for (const u of vertices) {
      const row = new Map<T, number>();
      for (const v of vertices) row.set(v, u === v ? 0 : Infinity);
      dist.set(u, row);
    }
    for (const u of vertices) {
      for (const edge of this.neighbors(u)) {
        const row = dist.get(u)!;
        if (edge.weight < row.get(edge.to)!) row.set(edge.to, edge.weight);
      }
    }

    // k outermost: "shortest path using only the first k as intermediates".
    for (const k of vertices) {
      for (const i of vertices) {
        for (const j of vertices) {
          const through = dist.get(i)!.get(k)! + dist.get(k)!.get(j)!;
          if (through < dist.get(i)!.get(j)!) dist.get(i)!.set(j, through);
        }
      }
    }
    return dist;
  }
}

const roads = new WeightedGraph<string>();
roads.addEdge("Dhaka", "Tangail", 98)
  .addEdge("Dhaka", "Comilla", 97)
  .addEdge("Tangail", "Bogura", 120)
  .addEdge("Comilla", "Sylhet", 200)
  .addEdge("Bogura", "Rangpur", 145)
  .addEdge("Sylhet", "Rangpur", 420)
  .addEdge("Dhaka", "Khulna", 270);

const { dist } = roads.dijkstra("Dhaka");
console.log(dist.get("Rangpur"), dist.get("Sylhet")); // 363 297
console.log(roads.shortestPath("Dhaka", "Rangpur"));
console.log(roads.distanceTo("Dhaka", "Sylhet"));     // 297

// A* with straight-line distance as the heuristic.
const coords: Record<string, [number, number]> = {
  Dhaka: [23.8, 90.4], Tangail: [24.3, 89.9], Comilla: [23.5, 91.2],
  Bogura: [24.8, 89.4], Sylhet: [24.9, 91.9], Rangpur: [25.7, 89.2],
  Khulna: [22.8, 89.6],
};
const euclidean = (a: string, b: string): number => {
  const [x1, y1] = coords[a];
  const [x2, y2] = coords[b];
  return Math.hypot(x1 - x2, y1 - y2) * 100; // never overestimates road distance
};
console.log(roads.aStar("Dhaka", "Rangpur", (v) => euclidean(v, "Rangpur")));

// Negative weights: Dijkstra would give the wrong answer here.
const neg = new WeightedGraph<string>(true);
neg.addEdge("a", "b", 4).addEdge("a", "c", 5).addEdge("b", "d", 3).addEdge("c", "b", -3);
console.log(neg.bellmanFord("a").get("d")); // 5 via a->c->b->d
```

## Practical Applications & Real-World Use Cases

- **Navigation apps.** Google Maps and Waze run heavily optimized variants (contraction hierarchies, A\* with landmarks) over road networks where edge weights update from live traffic. Plain Dijkstra on a continent-scale graph is too slow, but every optimization starts from it.
- **Network routing protocols.** OSPF and IS-IS have each router build a link-state map and run Dijkstra to compute its forwarding table. This is Dijkstra running in production on essentially every enterprise network.
- **Game pathfinding.** A\* on a navigation mesh or grid. The heuristic is what makes it fast enough to run for hundreds of units per frame.
- **Flight and transit routing.** Minimize total travel time, layover count, or price. Changing the weight function changes what "shortest" means without changing the algorithm.
- **Currency arbitrage detection.** Take negative logs of exchange rates and a negative cycle is a profitable arbitrage loop. Bellman-Ford finds it, which is a genuinely elegant reduction.
- **Robot motion planning.** Dijkstra or A\* over a discretized configuration space, with weights encoding terrain difficulty.
- **Dependency resolution with costs.** When multiple valid dependency versions exist, shortest path picks the cheapest install plan.
- **Image processing.** Seam carving finds the minimum-energy vertical path through a pixel grid, which is a shortest-path problem on a DAG.

## Related Concepts & Connections

- [[00-DSA-Index|Back to DSA Index]]
- [[14-Graph-Traversal-DFS-BFS]] for BFS, which is Dijkstra with all weights equal
- [[06-Priority-Queue-and-Heap]] for the heap that makes it efficient
- [[10-Graphs]] for representations and for MST, the other classic weighted-graph result
- [[16-Algorithmic-Strategies]] for why greedy works here and dynamic programming behind Floyd-Warshall
- [[11-Searching-Algorithms]] for the search-strategy contrast
