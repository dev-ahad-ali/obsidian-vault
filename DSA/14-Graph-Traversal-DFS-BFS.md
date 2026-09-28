---
tags:
  - dsa
  - computer-science
  - go
  - typescript
  - graph-traversal
type: study-note
status: complete
date_created: 2026-09-28
---

# Graph Traversal: DFS and BFS

## Overview

Traversal means visiting every reachable vertex exactly once. There are two ways to do it and they differ by one thing: which data structure holds the frontier of vertices you have discovered but not yet processed.

DFS uses a stack, so it dives as deep as possible before backing up. BFS uses a queue, so it fans out level by level. Same O(V + E) cost, same visited-set bookkeeping, radically different visit order, and that order is what makes each one suited to different problems.

Swap the stack for a queue in a working DFS and you have BFS. That is the whole difference, and noticing it is the moment graph traversal clicks.

## Key Concepts & Analogies

**DFS is exploring a maze by always taking the first unexplored corridor** and only backtracking at dead ends. You might end up very far from the entrance having covered little ground.

**BFS is a ripple in a pond.** Everything one step away, then everything two steps away. That expanding-shell property is why BFS finds shortest paths in unweighted graphs: the first time you reach a vertex, you reached it by the fewest possible edges.

**The visited set is not optional.** Without it, any cycle makes traversal loop forever and even a DAG with diamond-shaped paths causes exponential re-visiting. Mark vertices when you enqueue or push them, not when you dequeue or pop, or the same vertex enters the frontier many times.

**BFS gives shortest paths only when all edges cost the same.** Add weights and BFS is wrong, because three cheap edges can beat one expensive one. That is when you need [[15-Shortest-Path-Dijkstras-Algorithm|Dijkstra]], which is BFS with a priority queue instead of a plain queue.

**Recursive DFS is cleaner; iterative DFS is safer.** The recursive version is five lines and matches how you think about it. The iterative version with an explicit stack avoids stack overflow on deep graphs. A chain of a million vertices will crash recursive DFS in Node.js and survive it in Go.

**Cycle detection needs three states in a directed graph.** White = unvisited, gray = on the current recursion stack, black = fully explored. Reaching a gray vertex means a back edge, which means a cycle. Reaching a black vertex is fine, it just means you have seen that subgraph already. In an undirected graph you only need to check whether the visited neighbor is your parent.

**Topological sort orders a DAG so every edge points forward.** Two ways: DFS post-order reversed, or Kahn's algorithm, which repeatedly removes vertices with in-degree zero. Kahn's has a useful side effect: if it finishes with vertices remaining, those vertices are in a cycle, so you get cycle detection free. Use Kahn's when you need to report the cycle.

**Connected components.** Run traversal from every unvisited vertex; each run discovers one component. For undirected graphs this is also solvable with Union-Find. See [[10-Graphs]].

**Bipartite checking is two-coloring during BFS.** Color each vertex the opposite of its parent. A conflict means an odd-length cycle, which means not bipartite.

## Operations & Time/Space Complexity

| Algorithm | Time | Space | Finds shortest path | Notes |
| :--- | :--- | :--- | :--- | :--- |
| DFS (recursive) | O(V + E) | O(V) visited + O(V) stack | No | Cleanest code |
| DFS (iterative) | O(V + E) | O(V) visited + O(V) stack | No | Safe on deep graphs |
| BFS | O(V + E) | O(V) visited + O(V) queue | Yes, unweighted | Level-by-level |
| Connected components | O(V + E) | O(V) | n/a | Traverse from each unvisited |
| Cycle detection (directed) | O(V + E) | O(V) | n/a | Needs three colors |
| Cycle detection (undirected) | O(V + E) | O(V) | n/a | Track parent |
| Topological sort (DFS) | O(V + E) | O(V) | n/a | Reverse post-order |
| Topological sort (Kahn) | O(V + E) | O(V) | n/a | Detects cycles too |
| Bipartite check | O(V + E) | O(V) | n/a | Two-coloring BFS |

On an adjacency matrix every one of these becomes O(V²), because listing a vertex's neighbors costs O(V).

## Code Implementations

### 1. Go (Golang)

```go
package main

import "fmt"

// Graph is a minimal adjacency-list graph for traversal examples.
type Graph struct {
	adj      map[string][]string
	directed bool
}

func NewGraph(directed bool) *Graph {
	return &Graph{adj: make(map[string][]string), directed: directed}
}

func (g *Graph) AddEdge(u, v string) {
	g.adj[u] = append(g.adj[u], v)
	if _, ok := g.adj[v]; !ok {
		g.adj[v] = nil
	}
	if !g.directed {
		g.adj[v] = append(g.adj[v], u)
	}
}

func (g *Graph) Vertices() []string {
	out := make([]string, 0, len(g.adj))
	for v := range g.adj {
		out = append(out, v)
	}
	return out
}

// DFSRecursive is the clearest formulation. Depth is bounded by the
// longest path, which can be V, so watch out on deep graphs.
func (g *Graph) DFSRecursive(start string) []string {
	visited := make(map[string]bool)
	var order []string

	var visit func(string)
	visit = func(v string) {
		visited[v] = true
		order = append(order, v)
		for _, n := range g.adj[v] {
			if !visited[n] {
				visit(n)
			}
		}
	}
	visit(start)
	return order
}

// DFSIterative uses an explicit stack. Same order as recursive DFS if
// you push neighbors in reverse; no stack-overflow risk.
func (g *Graph) DFSIterative(start string) []string {
	visited := make(map[string]bool)
	var order []string
	stack := []string{start}

	for len(stack) > 0 {
		v := stack[len(stack)-1]
		stack = stack[:len(stack)-1]
		if visited[v] {
			continue
		}
		visited[v] = true
		order = append(order, v)
		// Reverse push so the first neighbor is processed first.
		for i := len(g.adj[v]) - 1; i >= 0; i-- {
			if !visited[g.adj[v][i]] {
				stack = append(stack, g.adj[v][i])
			}
		}
	}
	return order
}

// BFS uses a queue. Mark on enqueue, not on dequeue, or vertices enter
// the queue multiple times.
func (g *Graph) BFS(start string) []string {
	visited := map[string]bool{start: true}
	queue := []string{start}
	var order []string

	for len(queue) > 0 {
		v := queue[0]
		queue = queue[1:]
		order = append(order, v)
		for _, n := range g.adj[v] {
			if !visited[n] {
				visited[n] = true
				queue = append(queue, n)
			}
		}
	}
	return order
}

// BFSLevels groups vertices by distance from start. Processing the whole
// current level before the next is the trick.
func (g *Graph) BFSLevels(start string) [][]string {
	visited := map[string]bool{start: true}
	current := []string{start}
	var levels [][]string

	for len(current) > 0 {
		levels = append(levels, current)
		var next []string
		for _, v := range current {
			for _, n := range g.adj[v] {
				if !visited[n] {
					visited[n] = true
					next = append(next, n)
				}
			}
		}
		current = next
	}
	return levels
}

// ShortestPath in an unweighted graph. BFS is correct here precisely
// because every edge costs the same.
func (g *Graph) ShortestPath(start, goal string) []string {
	if start == goal {
		return []string{start}
	}
	parent := map[string]string{start: ""}
	queue := []string{start}

	for len(queue) > 0 {
		v := queue[0]
		queue = queue[1:]
		for _, n := range g.adj[v] {
			if _, seen := parent[n]; seen {
				continue
			}
			parent[n] = v
			if n == goal {
				// Walk parents backward, then reverse.
				path := []string{goal}
				for cur := v; cur != ""; cur = parent[cur] {
					path = append(path, cur)
				}
				for l, r := 0, len(path)-1; l < r; l, r = l+1, r-1 {
					path[l], path[r] = path[r], path[l]
				}
				return path
			}
			queue = append(queue, n)
		}
	}
	return nil // unreachable
}

// ConnectedComponents runs a traversal from every unvisited vertex.
func (g *Graph) ConnectedComponents() [][]string {
	visited := make(map[string]bool)
	var components [][]string

	for _, v := range g.Vertices() {
		if visited[v] {
			continue
		}
		var component []string
		stack := []string{v}
		for len(stack) > 0 {
			u := stack[len(stack)-1]
			stack = stack[:len(stack)-1]
			if visited[u] {
				continue
			}
			visited[u] = true
			component = append(component, u)
			stack = append(stack, g.adj[u]...)
		}
		components = append(components, component)
	}
	return components
}

// Three colors for directed cycle detection.
type color int

const (
	white color = iota // unvisited
	gray               // on the current DFS path
	black              // fully explored
)

// HasCycleDirected: reaching a gray vertex means a back edge, which
// means a cycle. Reaching black is harmless.
func (g *Graph) HasCycleDirected() bool {
	colors := make(map[string]color)

	var visit func(string) bool
	visit = func(v string) bool {
		colors[v] = gray
		for _, n := range g.adj[v] {
			switch colors[n] {
			case gray:
				return true // back edge
			case white:
				if visit(n) {
					return true
				}
			}
		}
		colors[v] = black
		return false
	}

	for _, v := range g.Vertices() {
		if colors[v] == white && visit(v) {
			return true
		}
	}
	return false
}

// HasCycleUndirected: a visited neighbor that is not your parent closes
// a cycle. Tracking the parent is all the extra state you need.
func (g *Graph) HasCycleUndirected() bool {
	visited := make(map[string]bool)

	var visit func(v, parent string) bool
	visit = func(v, parent string) bool {
		visited[v] = true
		for _, n := range g.adj[v] {
			if !visited[n] {
				if visit(n, v) {
					return true
				}
			} else if n != parent {
				return true
			}
		}
		return false
	}

	for _, v := range g.Vertices() {
		if !visited[v] && visit(v, "") {
			return true
		}
	}
	return false
}

// TopologicalSortKahn repeatedly removes in-degree-zero vertices.
// If the result is short, the leftovers form a cycle.
func (g *Graph) TopologicalSortKahn() ([]string, error) {
	inDegree := make(map[string]int)
	for _, v := range g.Vertices() {
		inDegree[v] += 0
		for _, n := range g.adj[v] {
			inDegree[n]++
		}
	}

	var queue []string
	for v, d := range inDegree {
		if d == 0 {
			queue = append(queue, v)
		}
	}

	var order []string
	for len(queue) > 0 {
		v := queue[0]
		queue = queue[1:]
		order = append(order, v)
		for _, n := range g.adj[v] {
			inDegree[n]--
			if inDegree[n] == 0 {
				queue = append(queue, n)
			}
		}
	}

	if len(order) != len(inDegree) {
		return nil, fmt.Errorf("graph has a cycle: only %d of %d vertices ordered",
			len(order), len(inDegree))
	}
	return order, nil
}

// TopologicalSortDFS is reverse post-order. Shorter, but it will not
// tell you which vertices are in the cycle.
func (g *Graph) TopologicalSortDFS() []string {
	visited := make(map[string]bool)
	var order []string

	var visit func(string)
	visit = func(v string) {
		visited[v] = true
		for _, n := range g.adj[v] {
			if !visited[n] {
				visit(n)
			}
		}
		order = append(order, v) // post-order: after all descendants
	}

	for _, v := range g.Vertices() {
		if !visited[v] {
			visit(v)
		}
	}
	for l, r := 0, len(order)-1; l < r; l, r = l+1, r-1 {
		order[l], order[r] = order[r], order[l]
	}
	return order
}

// IsBipartite two-colors during BFS. A conflict means an odd cycle.
func (g *Graph) IsBipartite() bool {
	colors := make(map[string]int)
	for _, start := range g.Vertices() {
		if _, done := colors[start]; done {
			continue
		}
		colors[start] = 0
		queue := []string{start}
		for len(queue) > 0 {
			v := queue[0]
			queue = queue[1:]
			for _, n := range g.adj[v] {
				c, seen := colors[n]
				if !seen {
					colors[n] = 1 - colors[v]
					queue = append(queue, n)
				} else if c == colors[v] {
					return false
				}
			}
		}
	}
	return true
}

func main() {
	// Undirected social graph.
	social := NewGraph(false)
	social.AddEdge("ada", "grace")
	social.AddEdge("ada", "alan")
	social.AddEdge("grace", "edsger")
	social.AddEdge("alan", "edsger")
	social.AddEdge("linus", "ken") // separate component

	fmt.Println("DFS:  ", social.DFSRecursive("ada"))
	fmt.Println("BFS:  ", social.BFS("ada"))
	fmt.Println("levels:", social.BFSLevels("ada"))
	fmt.Println("path: ", social.ShortestPath("ada", "edsger"))
	fmt.Println("components:", social.ConnectedComponents())
	fmt.Println("has cycle:", social.HasCycleUndirected()) // true

	// Directed build-dependency DAG.
	deps := NewGraph(true)
	deps.AddEdge("config", "server")
	deps.AddEdge("logger", "server")
	deps.AddEdge("server", "main")
	deps.AddEdge("config", "logger")

	order, err := deps.TopologicalSortKahn()
	fmt.Println(order, err)
	fmt.Println("cycle:", deps.HasCycleDirected()) // false

	deps.AddEdge("main", "config") // now it is circular
	_, err = deps.TopologicalSortKahn()
	fmt.Println(err)
	fmt.Println("cycle:", deps.HasCycleDirected()) // true
}
```

### 2. TypeScript

```typescript
class Graph<T> {
  readonly #adj = new Map<T, T[]>();
  readonly #directed: boolean;

  constructor(directed = false) {
    this.#directed = directed;
  }

  addEdge(u: T, v: T): this {
    if (!this.#adj.has(u)) this.#adj.set(u, []);
    if (!this.#adj.has(v)) this.#adj.set(v, []);
    this.#adj.get(u)!.push(v);
    if (!this.#directed) this.#adj.get(v)!.push(u);
    return this;
  }

  neighbors(v: T): readonly T[] { return this.#adj.get(v) ?? []; }
  vertices(): T[] { return [...this.#adj.keys()]; }

  /** Recursive DFS. Clean, but depth is bounded by the longest path. */
  dfsRecursive(start: T): T[] {
    const visited = new Set<T>();
    const order: T[] = [];

    const visit = (v: T): void => {
      visited.add(v);
      order.push(v);
      for (const n of this.neighbors(v)) if (!visited.has(n)) visit(n);
    };

    visit(start);
    return order;
  }

  /** Explicit stack. No overflow risk on deep graphs. */
  dfsIterative(start: T): T[] {
    const visited = new Set<T>();
    const order: T[] = [];
    const stack: T[] = [start];

    while (stack.length > 0) {
      const v = stack.pop()!;
      if (visited.has(v)) continue;
      visited.add(v);
      order.push(v);
      // Reverse so the first neighbor is popped first.
      const nbrs = this.neighbors(v);
      for (let i = nbrs.length - 1; i >= 0; i--) {
        if (!visited.has(nbrs[i])) stack.push(nbrs[i]);
      }
    }
    return order;
  }

  /** BFS. Marking on enqueue keeps each vertex in the queue once. */
  bfs(start: T): T[] {
    const visited = new Set<T>([start]);
    const queue: T[] = [start];
    const order: T[] = [];
    let head = 0; // index instead of shift(), which would be O(n)

    while (head < queue.length) {
      const v = queue[head++];
      order.push(v);
      for (const n of this.neighbors(v)) {
        if (!visited.has(n)) {
          visited.add(n);
          queue.push(n);
        }
      }
    }
    return order;
  }

  /** Vertices grouped by hop distance from start. */
  bfsLevels(start: T): T[][] {
    const visited = new Set<T>([start]);
    let current: T[] = [start];
    const levels: T[][] = [];

    while (current.length > 0) {
      levels.push(current);
      const next: T[] = [];
      for (const v of current) {
        for (const n of this.neighbors(v)) {
          if (!visited.has(n)) {
            visited.add(n);
            next.push(n);
          }
        }
      }
      current = next;
    }
    return levels;
  }

  /** Fewest-edges path. Correct only because edges are unweighted. */
  shortestPath(start: T, goal: T): T[] | null {
    if (start === goal) return [start];
    const parent = new Map<T, T | null>([[start, null]]);
    const queue: T[] = [start];
    let head = 0;

    while (head < queue.length) {
      const v = queue[head++];
      for (const n of this.neighbors(v)) {
        if (parent.has(n)) continue;
        parent.set(n, v);
        if (n === goal) {
          const path: T[] = [];
          for (let cur: T | null = n; cur !== null; cur = parent.get(cur)!) {
            path.push(cur);
          }
          return path.reverse();
        }
        queue.push(n);
      }
    }
    return null;
  }

  connectedComponents(): T[][] {
    const visited = new Set<T>();
    const components: T[][] = [];

    for (const v of this.vertices()) {
      if (visited.has(v)) continue;
      const component: T[] = [];
      const stack: T[] = [v];
      while (stack.length > 0) {
        const u = stack.pop()!;
        if (visited.has(u)) continue;
        visited.add(u);
        component.push(u);
        stack.push(...this.neighbors(u));
      }
      components.push(component);
    }
    return components;
  }

  /** Three-color DFS. Gray means "still on my path", so it is a cycle. */
  hasCycleDirected(): boolean {
    const WHITE = 0, GRAY = 1, BLACK = 2;
    const color = new Map<T, number>();

    const visit = (v: T): boolean => {
      color.set(v, GRAY);
      for (const n of this.neighbors(v)) {
        const c = color.get(n) ?? WHITE;
        if (c === GRAY) return true;
        if (c === WHITE && visit(n)) return true;
      }
      color.set(v, BLACK);
      return false;
    };

    for (const v of this.vertices()) {
      if ((color.get(v) ?? WHITE) === WHITE && visit(v)) return true;
    }
    return false;
  }

  /** Undirected: a visited neighbor that is not the parent closes a cycle. */
  hasCycleUndirected(): boolean {
    const visited = new Set<T>();

    const visit = (v: T, parent: T | null): boolean => {
      visited.add(v);
      for (const n of this.neighbors(v)) {
        if (!visited.has(n)) {
          if (visit(n, v)) return true;
        } else if (n !== parent) {
          return true;
        }
      }
      return false;
    };

    for (const v of this.vertices()) {
      if (!visited.has(v) && visit(v, null)) return true;
    }
    return false;
  }

  /** Kahn's algorithm. Throws with the cycle size when one exists. */
  topologicalSort(): T[] {
    const inDegree = new Map<T, number>();
    for (const v of this.vertices()) inDegree.set(v, inDegree.get(v) ?? 0);
    for (const v of this.vertices()) {
      for (const n of this.neighbors(v)) inDegree.set(n, (inDegree.get(n) ?? 0) + 1);
    }

    const queue = [...inDegree].filter(([, d]) => d === 0).map(([v]) => v);
    const order: T[] = [];
    let head = 0;

    while (head < queue.length) {
      const v = queue[head++];
      order.push(v);
      for (const n of this.neighbors(v)) {
        const d = inDegree.get(n)! - 1;
        inDegree.set(n, d);
        if (d === 0) queue.push(n);
      }
    }

    if (order.length !== inDegree.size) {
      const stuck = [...inDegree.keys()].filter((v) => !order.includes(v));
      throw new Error(`cycle detected involving: ${stuck.join(", ")}`);
    }
    return order;
  }

  /** Two-coloring BFS. A same-color edge means an odd cycle. */
  isBipartite(): boolean {
    const color = new Map<T, 0 | 1>();
    for (const start of this.vertices()) {
      if (color.has(start)) continue;
      color.set(start, 0);
      const queue: T[] = [start];
      let head = 0;
      while (head < queue.length) {
        const v = queue[head++];
        for (const n of this.neighbors(v)) {
          if (!color.has(n)) {
            color.set(n, color.get(v) === 0 ? 1 : 0);
            queue.push(n);
          } else if (color.get(n) === color.get(v)) {
            return false;
          }
        }
      }
    }
    return true;
  }
}

/** Grid BFS. Mazes and flood fill are graphs where edges are implicit. */
function shortestPathInGrid(grid: readonly string[][]): number {
  const rows = grid.length;
  const cols = grid[0]?.length ?? 0;
  if (rows === 0 || cols === 0 || grid[0][0] === "#") return -1;

  const DIRS = [[0, 1], [1, 0], [0, -1], [-1, 0]] as const;
  const dist = Array.from({ length: rows }, () => new Array(cols).fill(-1));
  dist[0][0] = 0;
  const queue: [number, number][] = [[0, 0]];
  let head = 0;

  while (head < queue.length) {
    const [r, c] = queue[head++];
    if (r === rows - 1 && c === cols - 1) return dist[r][c];
    for (const [dr, dc] of DIRS) {
      const nr = r + dr;
      const nc = c + dc;
      if (nr < 0 || nr >= rows || nc < 0 || nc >= cols) continue;
      if (grid[nr][nc] === "#" || dist[nr][nc] !== -1) continue;
      dist[nr][nc] = dist[r][c] + 1;
      queue.push([nr, nc]);
    }
  }
  return -1;
}

const social = new Graph<string>();
social.addEdge("ada", "grace")
  .addEdge("ada", "alan")
  .addEdge("grace", "edsger")
  .addEdge("alan", "edsger")
  .addEdge("linus", "ken");

console.log("DFS:", social.dfsRecursive("ada"));
console.log("BFS:", social.bfs("ada"));
console.log("levels:", social.bfsLevels("ada"));
console.log("path:", social.shortestPath("ada", "edsger"));
console.log("components:", social.connectedComponents());
console.log("cycle:", social.hasCycleUndirected()); // true

const deps = new Graph<string>(true);
deps.addEdge("config", "server")
  .addEdge("logger", "server")
  .addEdge("server", "main")
  .addEdge("config", "logger");
console.log(deps.topologicalSort());
console.log("cycle:", deps.hasCycleDirected()); // false

deps.addEdge("main", "config");
try { deps.topologicalSort(); } catch (e) { console.log((e as Error).message); }

console.log(shortestPathInGrid([
  [".", ".", "#"],
  ["#", ".", "."],
  [".", ".", "."],
])); // 4
```

## Practical Applications & Real-World Use Cases

- **Build systems and package managers.** `npm install`, `go mod graph`, Make, and Bazel topologically sort dependencies so nothing builds before its inputs. A cycle is a fatal error, reported using exactly the detection above.
- **Web crawlers.** BFS from a seed URL crawls breadth-first, which naturally prioritizes pages close to the entry point. The visited set is what stops the crawler looping forever.
- **Social network features.** "Degrees of separation" is BFS depth. "People you may know" is the set of vertices at distance exactly two.
- **Garbage collection.** Mark-and-sweep does a traversal from the roots; anything unreached is garbage. Tri-color marking is the same three colors used for cycle detection.
- **Maze solving and game AI.** BFS on a grid gives the fewest-moves path. Flood fill (paint bucket, Minesweeper's empty-region reveal) is grid BFS or DFS.
- **Deadlock detection.** A wait-for graph with a cycle means deadlocked transactions. Databases run this check and kill a victim.
- **Spreadsheet recalculation.** Cell formulas form a DAG. Recalculation is a topological sort, and a circular reference error is cycle detection.
- **Network broadcast and peer discovery.** Flooding a message to all reachable nodes is BFS across the network.

## Related Concepts & Connections

- [[00-DSA-Index|Back to DSA Index]]
- [[10-Graphs]] for representations these algorithms run on
- [[15-Shortest-Path-Dijkstras-Algorithm]] for BFS generalized to weighted edges
- [[05-Queues]] for BFS's frontier structure
- [[04-Stacks]] for DFS's frontier structure
- [[13-Recursion]] for recursive DFS and converting it to iteration
- [[07-Hash-Tables-and-Sets]] for the visited set
