---
tags:
  - dsa
  - computer-science
  - go
  - typescript
  - heaps
type: study-note
status: complete
date_created: 2026-09-28
---

# Priority Queue and Heap

## Overview

A priority queue serves the highest-priority element next, regardless of when it arrived. That is the abstract type. A binary heap is the usual implementation: a complete binary tree where every parent compares favorably to its children, stored flat in an array.

The heap property is weaker than sorted order, and that weakness is the point. Maintaining full sorted order costs O(n) per insert. Maintaining only "parent beats children" costs O(log n), and it is enough to answer "what is the minimum" in O(1).

## Key Concepts & Analogies

**A hospital emergency room.** Arrival time does not decide who gets seen. Severity does. A cardiac arrest that walks in at 3pm goes ahead of the sprained ankle that arrived at noon.

**A tournament bracket that only tracks the winner.** You know the champion immediately. You do not know who came third without doing more work. The heap knows its extreme and is deliberately vague about everything else.

**The array encoding is the elegant part.** A complete binary tree has no gaps, so you can store it in an array with no pointers at all. For index `i` (0-based):
- parent = `(i - 1) / 2`
- left child = `2i + 1`
- right child = `2i + 2`

No node objects, no allocation per element, perfect cache locality. This trick is worth remembering on its own.

**Sift up and sift down are the only two operations.** Insert appends at the end and sifts up until the parent is smaller. Extract swaps the root with the last element, shrinks by one, and sifts down until both children are larger. Everything else is built from these.

**Heapify is O(n), not O(n log n).** Building a heap from an existing array by sifting down from the last parent backward is linear. The intuition: most nodes are near the bottom and barely move. Half the nodes are leaves and move zero levels. This surprises people and it is a real result.

**Min-heap versus max-heap is just a comparator flip.** Write one implementation that takes a `less` function and you get both.

**Top-k in O(n log k).** To find the k largest of n items, keep a min-heap of size k. Push each item; if the heap exceeds k, pop the minimum. Memory stays at k, which matters when n does not fit in RAM. This is the answer to most "top trending" problems.

## Operations & Time/Space Complexity

| Operation | Average Case | Worst Case | Space Complexity |
| :--- | :--- | :--- | :--- |
| Peek min/max | O(1) | O(1) | O(1) |
| Insert (push) | O(log n) | O(log n) | O(1) |
| Extract min/max (pop) | O(log n) | O(log n) | O(1) |
| Delete arbitrary (known index) | O(log n) | O(log n) | O(1) |
| Decrease/increase key | O(log n) | O(log n) | O(1) |
| Search arbitrary value | O(n) | O(n) | O(1) |
| Build heap from array | O(n) | O(n) | O(1) in place |
| Heapsort | O(n log n) | O(n log n) | O(1) in place |

Total storage is O(n) with no pointer overhead.

## Code Implementations

### 1. Go (Golang)

```go
package main

import (
	"fmt"
	"sort"
)

// Heap is a generic binary heap. The less function decides min vs max:
// less(a,b) == a<b gives a min-heap, a>b gives a max-heap.
type Heap[T any] struct {
	items []T
	less  func(a, b T) bool
}

func NewHeap[T any](less func(a, b T) bool) *Heap[T] {
	return &Heap[T]{less: less}
}

// Heapify builds from an existing slice in O(n), not O(n log n).
// Sift down from the last parent backward; leaves need no work.
func Heapify[T any](items []T, less func(a, b T) bool) *Heap[T] {
	h := &Heap[T]{items: items, less: less}
	for i := len(items)/2 - 1; i >= 0; i-- {
		h.siftDown(i)
	}
	return h
}

func (h *Heap[T]) Len() int { return len(h.items) }

func (h *Heap[T]) Peek() (T, bool) {
	var zero T
	if len(h.items) == 0 {
		return zero, false
	}
	return h.items[0], true
}

// Push appends then sifts up. O(log n).
func (h *Heap[T]) Push(v T) {
	h.items = append(h.items, v)
	h.siftUp(len(h.items) - 1)
}

// Pop swaps root with last, truncates, then sifts down. O(log n).
func (h *Heap[T]) Pop() (T, bool) {
	var zero T
	n := len(h.items)
	if n == 0 {
		return zero, false
	}
	top := h.items[0]
	h.items[0] = h.items[n-1]
	h.items[n-1] = zero
	h.items = h.items[:n-1]
	if len(h.items) > 0 {
		h.siftDown(0)
	}
	return top, true
}

// siftUp walks a node toward the root while it beats its parent.
func (h *Heap[T]) siftUp(i int) {
	for i > 0 {
		parent := (i - 1) / 2
		if !h.less(h.items[i], h.items[parent]) {
			break
		}
		h.items[i], h.items[parent] = h.items[parent], h.items[i]
		i = parent
	}
}

// siftDown walks a node toward the leaves, always swapping with the
// better of its two children so the heap property survives.
func (h *Heap[T]) siftDown(i int) {
	n := len(h.items)
	for {
		best := i
		if l := 2*i + 1; l < n && h.less(h.items[l], h.items[best]) {
			best = l
		}
		if r := 2*i + 2; r < n && h.less(h.items[r], h.items[best]) {
			best = r
		}
		if best == i {
			return
		}
		h.items[i], h.items[best] = h.items[best], h.items[i]
		i = best
	}
}

// --- A priority queue with real payloads ---

type Task struct {
	Name     string
	Priority int // lower number means more urgent
}

type PriorityQueue struct {
	h *Heap[Task]
}

func NewPriorityQueue() *PriorityQueue {
	return &PriorityQueue{h: NewHeap(func(a, b Task) bool {
		return a.Priority < b.Priority
	})}
}

func (pq *PriorityQueue) Add(name string, priority int) {
	pq.h.Push(Task{Name: name, Priority: priority})
}

func (pq *PriorityQueue) Next() (Task, bool) { return pq.h.Pop() }
func (pq *PriorityQueue) Len() int           { return pq.h.Len() }

// TopK returns the k largest values using a min-heap of size k.
// O(n log k) time, O(k) space. The reason: you never hold more than k.
func TopK(nums []int, k int) []int {
	if k <= 0 {
		return nil
	}
	h := NewHeap(func(a, b int) bool { return a < b }) // min-heap
	for _, n := range nums {
		h.Push(n)
		if h.Len() > k {
			h.Pop() // evict the smallest of the k+1 candidates
		}
	}
	out := make([]int, 0, h.Len())
	for h.Len() > 0 {
		v, _ := h.Pop()
		out = append(out, v)
	}
	sort.Sort(sort.Reverse(sort.IntSlice(out)))
	return out
}

// HeapSort: heapify with a max-heap, then repeatedly move the root to
// the end. In place, O(n log n) worst case, not stable.
func HeapSort(nums []int) {
	h := Heapify(nums, func(a, b int) bool { return a > b })
	for end := len(nums) - 1; end > 0; end-- {
		h.items[0], h.items[end] = h.items[end], h.items[0]
		h.items = h.items[:end]
		h.siftDown(0)
	}
	h.items = nums
}

func main() {
	pq := NewPriorityQueue()
	pq.Add("deploy hotfix", 1)
	pq.Add("update docs", 9)
	pq.Add("fix flaky test", 4)
	pq.Add("page on-call", 0)
	for pq.Len() > 0 {
		t, _ := pq.Next()
		fmt.Printf("p%d %s\n", t.Priority, t.Name)
	}
	// p0 page on-call / p1 deploy hotfix / p4 fix flaky test / p9 update docs

	fmt.Println(TopK([]int{5, 1, 9, 3, 14, 7, 2}, 3)) // [14 9 7]

	nums := []int{5, 1, 9, 3, 14, 7, 2}
	HeapSort(nums)
	fmt.Println(nums) // [1 2 3 5 7 9 14]
}
```

### 2. TypeScript

```typescript
type Comparator<T> = (a: T, b: T) => number;

/**
 * Binary heap over a flat array. Negative comparator result means
 * "a comes first", so (a,b) => a-b is a min-heap.
 */
class Heap<T> {
  #items: T[] = [];
  readonly #compare: Comparator<T>;

  constructor(compare: Comparator<T>, initial: readonly T[] = []) {
    this.#compare = compare;
    if (initial.length > 0) {
      this.#items = [...initial];
      // O(n) build: sift down from the last parent backward.
      for (let i = (this.#items.length >> 1) - 1; i >= 0; i--) this.#siftDown(i);
    }
  }

  get size(): number { return this.#items.length; }
  get isEmpty(): boolean { return this.#items.length === 0; }

  peek(): T | undefined { return this.#items[0]; }

  /** O(log n). */
  push(value: T): this {
    this.#items.push(value);
    this.#siftUp(this.#items.length - 1);
    return this;
  }

  /** O(log n). */
  pop(): T | undefined {
    if (this.#items.length === 0) return undefined;
    const top = this.#items[0];
    const last = this.#items.pop()!;
    if (this.#items.length > 0) {
      this.#items[0] = last;
      this.#siftDown(0);
    }
    return top;
  }

  #siftUp(i: number): void {
    while (i > 0) {
      const parent = (i - 1) >> 1;
      if (this.#compare(this.#items[i], this.#items[parent]) >= 0) break;
      this.#swap(i, parent);
      i = parent;
    }
  }

  #siftDown(i: number): void {
    const n = this.#items.length;
    for (;;) {
      let best = i;
      const l = 2 * i + 1;
      const r = 2 * i + 2;
      if (l < n && this.#compare(this.#items[l], this.#items[best]) < 0) best = l;
      if (r < n && this.#compare(this.#items[r], this.#items[best]) < 0) best = r;
      if (best === i) return;
      this.#swap(i, best);
      i = best;
    }
  }

  #swap(i: number, j: number): void {
    [this.#items[i], this.#items[j]] = [this.#items[j], this.#items[i]];
  }

  /** Drains the heap in priority order. Consumes it. */
  *drain(): Generator<T> {
    while (this.#items.length > 0) yield this.pop()!;
  }
}

interface Task {
  readonly name: string;
  readonly priority: number; // lower is more urgent
}

class PriorityQueue {
  #heap = new Heap<Task>((a, b) => a.priority - b.priority);

  add(name: string, priority: number): void {
    this.#heap.push({ name, priority });
  }

  next(): Task | undefined { return this.#heap.pop(); }
  get size(): number { return this.#heap.size; }
}

/** k largest via a min-heap capped at k. O(n log k) time, O(k) space. */
function topK(nums: readonly number[], k: number): number[] {
  if (k <= 0) return [];
  const heap = new Heap<number>((a, b) => a - b);
  for (const n of nums) {
    heap.push(n);
    if (heap.size > k) heap.pop();
  }
  return [...heap.drain()].reverse();
}

/** Merges k sorted lists in O(n log k) by heaping the list heads. */
function mergeSortedLists(lists: readonly (readonly number[])[]): number[] {
  type Cursor = { value: number; list: number; index: number };
  const heap = new Heap<Cursor>((a, b) => a.value - b.value);

  lists.forEach((list, i) => {
    if (list.length > 0) heap.push({ value: list[0], list: i, index: 0 });
  });

  const out: number[] = [];
  while (!heap.isEmpty) {
    const { value, list, index } = heap.pop()!;
    out.push(value);
    const next = index + 1;
    if (next < lists[list].length) {
      heap.push({ value: lists[list][next], list, index: next });
    }
  }
  return out;
}

const pq = new PriorityQueue();
pq.add("deploy hotfix", 1);
pq.add("update docs", 9);
pq.add("fix flaky test", 4);
pq.add("page on-call", 0);
while (pq.size > 0) {
  const t = pq.next()!;
  console.log(`p${t.priority} ${t.name}`);
}

console.log(topK([5, 1, 9, 3, 14, 7, 2], 3));                  // [14, 9, 7]
console.log(mergeSortedLists([[1, 4, 9], [2, 3], [0, 10]]));   // [0,1,2,3,4,9,10]
```

## Practical Applications & Real-World Use Cases

- **Dijkstra's algorithm.** The min-heap is what turns the O(V²) version into O((V+E) log V). This is the headline application. See [[15-Shortest-Path-Dijkstras-Algorithm]].
- **A* pathfinding.** Same structure, different priority function. Every game with navigation uses it.
- **OS schedulers.** Linux CFS uses a red-black tree, but priority-based real-time schedulers are heaps.
- **Huffman coding.** Repeatedly pop the two least frequent symbols and push their merged parent. The heap is the whole algorithm. See [[16-Algorithmic-Strategies]].
- **Top-k analytics.** Trending topics, top error messages, heaviest queries. A bounded heap gives you the answer in one pass over a stream.
- **External merge sort.** Merging k sorted runs from disk uses a heap over the run heads, exactly the `mergeSortedLists` code above.
- **Event simulation.** Discrete-event simulators keep pending events in a min-heap keyed by timestamp and pop the next one to happen.

## Related Concepts & Connections

- [[00-DSA-Index|Back to DSA Index]]
- [[05-Queues]] for the FIFO version where arrival order wins
- [[08-Trees-BST-and-AVL]] for a tree that keeps full order at higher cost
- [[12-Sorting-Algorithms]] for heapsort in context
- [[15-Shortest-Path-Dijkstras-Algorithm]] for the application that justifies learning this
- [[02-Arrays]] for the flat encoding that makes heaps allocation-free
