---
tags:
  - dsa
  - computer-science
  - go
  - typescript
  - recursion
type: study-note
status: complete
date_created: 2026-09-28
---

# Recursion

## Overview

A recursive function solves a problem by calling itself on a smaller version of the same problem. Two parts are mandatory: a base case that returns without recursing, and a recursive case that makes progress toward the base case. Miss either one and you get infinite recursion, which ends in a stack overflow.

Recursion is not faster than iteration. Each call costs a stack frame, and the function-call overhead is real. What recursion buys you is that some problems have a recursive shape (trees, graphs, nested structures, divide and conquer) and expressing them iteratively means building an explicit stack and doing by hand what the runtime would do for you.

## Key Concepts & Analogies

**Russian nesting dolls.** Open a doll, find a smaller doll, repeat. The base case is the solid doll that does not open. Without it you would be opening forever.

**Standing in a queue and asking "how many people are ahead of me?"** Ask the person in front. They ask the person in front of them. The person at the front says "zero" (base case) and the answer propagates back, each person adding one. That propagation back up is the part beginners miss: the work does not all happen on the way down.

**The call stack is what makes it work.** Each call pushes a frame with its own copy of the parameters and locals. Return values unwind back up. See [[04-Stacks]]. Default stack limits are roughly 10,000 frames in Node.js and about 1MB on a typical OS thread, so recursion depth is a hard resource. Go is the outlier: goroutine stacks start at 8KB and grow dynamically up to 1GB, so deep recursion works where it would crash elsewhere.

**Tail recursion is when the recursive call is the last thing the function does.** A compiler can then reuse the frame instead of pushing a new one, making it a loop. Scheme and OCaml guarantee this. Go, JavaScript, Python, and Java do not. ES6 specified tail-call optimization and essentially no engine shipped it. Do not rely on it.

**Recursion with memoization is dynamic programming top-down.** Naive recursive Fibonacci is O(2ⁿ) because it recomputes the same subproblems exponentially often. Cache results in a map and it becomes O(n). This one change is the whole idea behind DP. See [[16-Algorithmic-Strategies]].

**Tree recursion versus linear recursion.** One recursive call per invocation gives a chain, O(n) depth. Two or more gives a tree, and the branching factor determines the cost. Recognizing which you have tells you whether the problem is O(n) or O(2ⁿ).

**Analyzing cost with recurrences.** `T(n) = 2T(n/2) + O(n)` is merge sort, which solves to O(n log n). `T(n) = T(n-1) + O(1)` is a linear walk, O(n). `T(n) = 2T(n-1) + O(1)` is O(2ⁿ) and means you are enumerating subsets. The Master Theorem mechanizes the divide-and-conquer cases.

**When to convert to iteration.** Convert when depth could exceed the stack, when the overhead shows up in a profile, or when the iterative version is genuinely clearer (linear recursion usually is). Keep recursion for tree and graph work, where the iterative version is strictly worse to read.

## Operations & Time/Space Complexity

| Recursion pattern | Time | Stack space | Example |
| :--- | :--- | :--- | :--- |
| Linear, one call | O(n) | O(n) | Factorial, list length |
| Tail recursive (optimized) | O(n) | O(1) | Accumulator sum in Scheme |
| Binary, halving | O(log n) | O(log n) | Binary search |
| Divide and conquer, 2 halves + merge | O(n log n) | O(log n) | Merge sort |
| Tree, two calls on n-1 | O(2ⁿ) | O(n) | Naive Fibonacci |
| Tree, memoized | O(n) | O(n) | Fibonacci with a cache |
| Permutations | O(n!·n) | O(n) | Full permutation generation |
| Subsets | O(2ⁿ·n) | O(n) | Power set |
| Tree traversal | O(n) | O(h) | DFS on a tree |

h is tree height, O(log n) balanced and O(n) degenerate.

## Code Implementations

### 1. Go (Golang)

```go
package main

import "fmt"

// Factorial: the textbook linear recursion. O(n) time, O(n) stack.
func Factorial(n int) int {
	if n <= 1 { // base case
		return 1
	}
	return n * Factorial(n-1) // recursive case, strictly smaller
}

// FactorialIter is what the above compiles to conceptually. O(1) space.
// Prefer this shape for linear recursion in production Go.
func FactorialIter(n int) int {
	result := 1
	for i := 2; i <= n; i++ {
		result *= i
	}
	return result
}

// FibNaive is O(2^n). Fib(40) takes about a second; Fib(50) takes minutes.
// It recomputes Fib(38) millions of times.
func FibNaive(n int) int {
	if n < 2 {
		return n
	}
	return FibNaive(n-1) + FibNaive(n-2)
}

// FibMemo caches subproblems, turning O(2^n) into O(n). This single
// change is the essence of top-down dynamic programming.
func FibMemo(n int) int {
	memo := make(map[int]int, n)
	var fib func(int) int
	fib = func(n int) int {
		if n < 2 {
			return n
		}
		if v, ok := memo[n]; ok {
			return v
		}
		v := fib(n-1) + fib(n-2)
		memo[n] = v
		return v
	}
	return fib(n)
}

// FibIter is O(n) time, O(1) space. The bottom-up version.
func FibIter(n int) int {
	a, b := 0, 1
	for i := 0; i < n; i++ {
		a, b = b, a+b
	}
	return a
}

// --- Recursion over structure, where it genuinely reads better ---

type TreeNode struct {
	Val         int
	Left, Right *TreeNode
}

// SumTree: the recursion mirrors the data. No iterative version is clearer.
func SumTree(n *TreeNode) int {
	if n == nil {
		return 0
	}
	return n.Val + SumTree(n.Left) + SumTree(n.Right)
}

func TreeHeight(n *TreeNode) int {
	if n == nil {
		return -1
	}
	return 1 + max(TreeHeight(n.Left), TreeHeight(n.Right))
}

// Flatten handles arbitrarily nested slices. Recursion is the only sane
// approach when the nesting depth is unknown.
func Flatten(nested []any) []int {
	var out []int
	for _, item := range nested {
		switch v := item.(type) {
		case int:
			out = append(out, v)
		case []any:
			out = append(out, Flatten(v)...)
		}
	}
	return out
}

// --- Backtracking: recursion that undoes its choices ---

// Permutations generates every ordering. O(n!) results, so O(n!*n) work.
func Permutations[T any](xs []T) [][]T {
	var out [][]T
	var backtrack func(start int)
	backtrack = func(start int) {
		if start == len(xs) {
			perm := make([]T, len(xs))
			copy(perm, xs) // must copy; xs keeps mutating
			out = append(out, perm)
			return
		}
		for i := start; i < len(xs); i++ {
			xs[start], xs[i] = xs[i], xs[start] // choose
			backtrack(start + 1)                // explore
			xs[start], xs[i] = xs[i], xs[start] // undo
		}
	}
	backtrack(0)
	return out
}

// Subsets generates the power set. 2^n results: include or exclude each.
func Subsets[T any](xs []T) [][]T {
	var out [][]T
	var current []T
	var backtrack func(i int)
	backtrack = func(i int) {
		if i == len(xs) {
			subset := make([]T, len(current))
			copy(subset, current)
			out = append(out, subset)
			return
		}
		backtrack(i + 1) // exclude xs[i]
		current = append(current, xs[i])
		backtrack(i + 1) // include xs[i]
		current = current[:len(current)-1]
	}
	backtrack(0)
	return out
}

// TowerOfHanoi: three lines that move 2^n-1 discs correctly. The proof
// that recursion is sometimes shorter than the explanation.
func TowerOfHanoi(n int, from, to, via string, moves *[]string) {
	if n == 0 {
		return
	}
	TowerOfHanoi(n-1, from, via, to, moves)
	*moves = append(*moves, fmt.Sprintf("%s->%s", from, to))
	TowerOfHanoi(n-1, via, to, from, moves)
}

// Ackermann grows faster than any primitive recursive function.
// Included as a reminder that recursion depth can explode fast:
// Ackermann(4,2) has 19729 digits.
func Ackermann(m, n int) int {
	switch {
	case m == 0:
		return n + 1
	case n == 0:
		return Ackermann(m-1, 1)
	default:
		return Ackermann(m-1, Ackermann(m, n-1))
	}
}

func main() {
	fmt.Println(Factorial(10), FactorialIter(10)) // 3628800 3628800
	fmt.Println(FibNaive(25), FibMemo(90), FibIter(90))

	root := &TreeNode{1,
		&TreeNode{2, &TreeNode{4, nil, nil}, nil},
		&TreeNode{3, nil, nil},
	}
	fmt.Println(SumTree(root), TreeHeight(root)) // 10 2

	fmt.Println(Flatten([]any{1, []any{2, []any{3, 4}}, 5})) // [1 2 3 4 5]
	fmt.Println(Permutations([]int{1, 2, 3}))                // 6 permutations
	fmt.Println(len(Subsets([]int{1, 2, 3, 4})))             // 16

	var moves []string
	TowerOfHanoi(3, "A", "C", "B", &moves)
	fmt.Println(len(moves), moves[:3]) // 7 [A->B A->C B->C]

	fmt.Println(Ackermann(2, 3)) // 9
}
```

### 2. TypeScript

```typescript
/** Linear recursion. O(n) time, O(n) stack. */
function factorial(n: number): number {
  if (n <= 1) return 1;           // base case
  return n * factorial(n - 1);    // recursive case
}

/** Same result, O(1) space. What a TCO-capable compiler would produce. */
function factorialIter(n: number): number {
  let result = 1;
  for (let i = 2; i <= n; i++) result *= i;
  return result;
}

/** O(2^n). fib(35) already takes seconds. */
function fibNaive(n: number): number {
  if (n < 2) return n;
  return fibNaive(n - 1) + fibNaive(n - 2);
}

/** Memoized: O(n). The map turns the recursion tree into a chain. */
function fibMemo(n: number, memo = new Map<number, number>()): number {
  if (n < 2) return n;
  const cached = memo.get(n);
  if (cached !== undefined) return cached;
  const value = fibMemo(n - 1, memo) + fibMemo(n - 2, memo);
  memo.set(n, value);
  return value;
}

/** Generic memoizer. Wraps any pure function keyed on its arguments. */
function memoize<Args extends unknown[], R>(
  fn: (...args: Args) => R,
): (...args: Args) => R {
  const cache = new Map<string, R>();
  return (...args: Args): R => {
    const key = JSON.stringify(args);
    if (!cache.has(key)) cache.set(key, fn(...args));
    return cache.get(key)!;
  };
}

// --- Recursion over structure ---

interface TreeNode {
  value: number;
  left: TreeNode | null;
  right: TreeNode | null;
}

/** The recursion mirrors the shape of the data. */
function sumTree(node: TreeNode | null): number {
  if (node === null) return 0;
  return node.value + sumTree(node.left) + sumTree(node.right);
}

function treeHeight(node: TreeNode | null): number {
  if (node === null) return -1;
  return 1 + Math.max(treeHeight(node.left), treeHeight(node.right));
}

/** Recursive type for arbitrarily nested arrays. */
type Nested<T> = T | Nested<T>[];

function flatten<T>(input: readonly Nested<T>[]): T[] {
  const out: T[] = [];
  for (const item of input) {
    if (Array.isArray(item)) out.push(...flatten(item as Nested<T>[]));
    else out.push(item);
  }
  return out;
}

/** Deep clone. Recursion is the natural fit for recursive data. */
function deepClone<T>(value: T): T {
  if (value === null || typeof value !== "object") return value;
  if (Array.isArray(value)) return value.map(deepClone) as T;
  if (value instanceof Date) return new Date(value.getTime()) as T;
  if (value instanceof Map) {
    return new Map([...value].map(([k, v]) => [deepClone(k), deepClone(v)])) as T;
  }
  const out: Record<string, unknown> = {};
  for (const [k, v] of Object.entries(value)) out[k] = deepClone(v);
  return out as T;
}

// --- Backtracking ---

/** Every ordering. O(n!) outputs. */
function permutations<T>(xs: readonly T[]): T[][] {
  if (xs.length <= 1) return [[...xs]];
  const out: T[][] = [];
  for (let i = 0; i < xs.length; i++) {
    const rest = [...xs.slice(0, i), ...xs.slice(i + 1)];
    for (const perm of permutations(rest)) out.push([xs[i], ...perm]);
  }
  return out;
}

/** Power set. 2^n outputs: include or exclude each element. */
function subsets<T>(xs: readonly T[]): T[][] {
  const out: T[][] = [];
  const current: T[] = [];

  const backtrack = (i: number): void => {
    if (i === xs.length) {
      out.push([...current]);
      return;
    }
    backtrack(i + 1);        // exclude
    current.push(xs[i]);
    backtrack(i + 1);        // include
    current.pop();           // undo
  };

  backtrack(0);
  return out;
}

/** 2^n - 1 moves, three lines of logic. */
function towerOfHanoi(n: number, from = "A", to = "C", via = "B"): string[] {
  if (n === 0) return [];
  return [
    ...towerOfHanoi(n - 1, from, via, to),
    `${from}->${to}`,
    ...towerOfHanoi(n - 1, via, to, from),
  ];
}

/**
 * Converting recursion to iteration with an explicit stack. Needed when
 * depth could exceed the ~10k frame limit in Node.
 */
function sumTreeIterative(root: TreeNode | null): number {
  if (root === null) return 0;
  const stack: TreeNode[] = [root];
  let total = 0;
  while (stack.length > 0) {
    const node = stack.pop()!;
    total += node.value;
    if (node.left) stack.push(node.left);
    if (node.right) stack.push(node.right);
  }
  return total;
}

console.log(factorial(10), factorialIter(10));  // 3628800 3628800
console.log(fibNaive(25), fibMemo(90));

const fastFib = memoize((n: number): number => (n < 2 ? n : fastFib(n - 1) + fastFib(n - 2)));
console.log(fastFib(50)); // 12586269025

const root: TreeNode = {
  value: 1,
  left: { value: 2, left: { value: 4, left: null, right: null }, right: null },
  right: { value: 3, left: null, right: null },
};
console.log(sumTree(root), treeHeight(root), sumTreeIterative(root)); // 10 2 10

console.log(flatten([1, [2, [3, 4]], 5]));       // [1,2,3,4,5]
console.log(permutations([1, 2, 3]).length);     // 6
console.log(subsets([1, 2, 3, 4]).length);       // 16
console.log(towerOfHanoi(3));                    // 7 moves
console.log(deepClone({ a: [1, { b: new Date(0) }] }));
```

## Practical Applications & Real-World Use Cases

- **Parsers and compilers.** Recursive descent parsing has one function per grammar rule, and the recursion follows the grammar's own recursion. JSON parsers, SQL parsers, and most hand-written compilers work this way.
- **Filesystem traversal.** Walking directories is recursion over a tree. `find`, `du`, and every build tool's file globbing do this.
- **JSON and tree serialization.** `JSON.stringify` recurses over nested structure. So does any deep-equal or deep-merge utility.
- **React's reconciler.** Rendering a component tree is recursive by nature. React's fiber architecture famously converted it to an iterative loop with an explicit stack so rendering could be paused and resumed.
- **Divide and conquer algorithms.** Merge sort, quicksort, and binary search are all naturally recursive. See [[12-Sorting-Algorithms]].
- **Graph and tree traversal.** DFS is recursion with a visited set. See [[14-Graph-Traversal-DFS-BFS]].
- **Constraint solving.** Sudoku, N-queens, and SAT solvers are backtracking, which is recursion with undo.
- **Fractals and procedural generation.** Terrain via midpoint displacement, L-systems for plants, and the Mandelbrot set are all defined recursively.

## Related Concepts & Connections

- [[00-DSA-Index|Back to DSA Index]]
- [[04-Stacks]] because the call stack is what makes recursion work
- [[08-Trees-BST-and-AVL]] for the structures that recursion handles best
- [[14-Graph-Traversal-DFS-BFS]] for recursive DFS and its iterative twin
- [[16-Algorithmic-Strategies]] for memoization, DP, and backtracking in full
- [[12-Sorting-Algorithms]] for divide-and-conquer recurrences
- [[01-Fundamentals-and-Introduction]] for analyzing recursive cost
