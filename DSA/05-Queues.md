---
tags:
  - dsa
  - computer-science
  - go
  - typescript
  - queues
type: study-note
status: complete
date_created: 2026-09-28
---

# Queues

## Overview

A queue is FIFO: first in, first out. You add at the back and remove from the front. Where a stack is about the most recent item, a queue is about fairness and order preservation. Things get handled in the order they arrived.

The naive implementation is an array where dequeue shifts everything left, which is O(n) and wrong. The right implementation is a ring buffer (two indices wrapping around a fixed array) or a linked list with head and tail pointers. Both give O(1) on both ends.

## Key Concepts & Analogies

**A checkout line.** Arrive at the back, leave from the front, no queue-jumping. If someone could jump, it would be a [[06-Priority-Queue-and-Heap|priority queue]].

**The ring buffer.** Keep `head`, `tail`, and `count` over a fixed array. Enqueue writes at `tail` and advances it with `tail = (tail + 1) % capacity`. Dequeue reads at `head` and advances the same way. The array never shifts and never reallocates if you size it right. This is how audio drivers, network cards, and lock-free producer-consumer queues all work.

**Why not just `shift()` a JavaScript array.** `Array.prototype.shift` is O(n) because it reindexes every element. Do it n times and you have an accidental O(n²). For small queues nobody notices; for a BFS over a large graph it is the bottleneck.

**Deque, or double-ended queue.** Push and pop at both ends, all O(1). A deque is a superset of both a stack and a queue, which is why many standard libraries ship only a deque. The sliding-window-maximum problem uses a monotonic deque to get O(n).

**Bounded versus unbounded.** A bounded queue rejects or blocks when full. That is a feature: backpressure. An unbounded queue in a producer-consumer system just moves the failure from "queue full" to "out of memory". Go channels are bounded queues with blocking semantics built in.

**Circular queue off-by-one.** With only `head` and `tail`, a full queue and an empty queue look identical. Either keep an explicit `count` (what the code below does) or waste one slot. Keeping the count is clearer.

## Operations & Time/Space Complexity

| Operation | Average Case | Worst Case | Space Complexity |
| :--- | :--- | :--- | :--- |
| Enqueue | O(1) amortized | O(n) on resize | O(1) |
| Dequeue | O(1) | O(1) | O(1) |
| Peek front | O(1) | O(1) | O(1) |
| Search | O(n) | O(n) | O(1) |
| Size / IsEmpty | O(1) | O(1) | O(1) |

Total storage is O(n). A fixed-capacity ring buffer has a true O(1) worst case for enqueue because it never resizes.

## Code Implementations

### 1. Go (Golang)

```go
package main

import (
	"errors"
	"fmt"
)

var ErrQueueFull = errors.New("queue is full")

// RingQueue is a fixed-capacity circular buffer. No allocation after
// construction, no shifting, true O(1) on both ends.
type RingQueue[T any] struct {
	buf   []T
	head  int // index of the front element
	tail  int // index where the next element goes
	count int // explicit count avoids the full/empty ambiguity
}

func NewRingQueue[T any](capacity int) *RingQueue[T] {
	if capacity < 1 {
		capacity = 1
	}
	return &RingQueue[T]{buf: make([]T, capacity)}
}

func (q *RingQueue[T]) Len() int      { return q.count }
func (q *RingQueue[T]) Cap() int      { return len(q.buf) }
func (q *RingQueue[T]) IsEmpty() bool { return q.count == 0 }
func (q *RingQueue[T]) IsFull() bool  { return q.count == len(q.buf) }

func (q *RingQueue[T]) Enqueue(v T) error {
	if q.IsFull() {
		return ErrQueueFull // backpressure instead of unbounded growth
	}
	q.buf[q.tail] = v
	q.tail = (q.tail + 1) % len(q.buf)
	q.count++
	return nil
}

func (q *RingQueue[T]) Dequeue() (T, bool) {
	var zero T
	if q.count == 0 {
		return zero, false
	}
	v := q.buf[q.head]
	q.buf[q.head] = zero // release the reference
	q.head = (q.head + 1) % len(q.buf)
	q.count--
	return v, true
}

func (q *RingQueue[T]) Peek() (T, bool) {
	var zero T
	if q.count == 0 {
		return zero, false
	}
	return q.buf[q.head], true
}

// GrowableQueue amortizes to O(1) by consuming from the front of a slice
// and only compacting when the wasted prefix gets large. This is the
// pragmatic choice when you do not know the size in advance.
type GrowableQueue[T any] struct {
	items []T
	head  int
}

func (q *GrowableQueue[T]) Len() int { return len(q.items) - q.head }

func (q *GrowableQueue[T]) Enqueue(v T) { q.items = append(q.items, v) }

func (q *GrowableQueue[T]) Dequeue() (T, bool) {
	var zero T
	if q.head >= len(q.items) {
		return zero, false
	}
	v := q.items[q.head]
	q.items[q.head] = zero
	q.head++
	// Compact once the dead prefix is at least half the slice.
	if q.head*2 >= len(q.items) {
		q.items = append(q.items[:0], q.items[q.head:]...)
		q.head = 0
	}
	return v, true
}

// Deque supports both ends. A superset of stack and queue.
type Deque[T any] struct {
	items []T
}

func (d *Deque[T]) Len() int        { return len(d.items) }
func (d *Deque[T]) PushBack(v T)    { d.items = append(d.items, v) }
func (d *Deque[T]) PushFront(v T)   { d.items = append([]T{v}, d.items...) }

func (d *Deque[T]) PopBack() (T, bool) {
	var zero T
	if len(d.items) == 0 {
		return zero, false
	}
	v := d.items[len(d.items)-1]
	d.items = d.items[:len(d.items)-1]
	return v, true
}

func (d *Deque[T]) PopFront() (T, bool) {
	var zero T
	if len(d.items) == 0 {
		return zero, false
	}
	v := d.items[0]
	d.items = d.items[1:]
	return v, true
}

func (d *Deque[T]) Front() (T, bool) {
	var zero T
	if len(d.items) == 0 {
		return zero, false
	}
	return d.items[0], true
}

func (d *Deque[T]) Back() (T, bool) {
	var zero T
	if len(d.items) == 0 {
		return zero, false
	}
	return d.items[len(d.items)-1], true
}

// SlidingWindowMax uses a monotonic decreasing deque of indices.
// O(n) total: every index enters and leaves the deque once.
func SlidingWindowMax(nums []int, k int) []int {
	if k <= 0 || k > len(nums) {
		return nil
	}
	out := make([]int, 0, len(nums)-k+1)
	dq := make([]int, 0, len(nums)) // indices, values decreasing
	for i, n := range nums {
		// Drop indices that have slid out of the window.
		if len(dq) > 0 && dq[0] <= i-k {
			dq = dq[1:]
		}
		// Anything smaller than the incoming value can never be the max.
		for len(dq) > 0 && nums[dq[len(dq)-1]] <= n {
			dq = dq[:len(dq)-1]
		}
		dq = append(dq, i)
		if i >= k-1 {
			out = append(out, nums[dq[0]])
		}
	}
	return out
}

func main() {
	q := NewRingQueue[string](3)
	q.Enqueue("a")
	q.Enqueue("b")
	q.Enqueue("c")
	fmt.Println(q.Enqueue("d")) // queue is full
	v, _ := q.Dequeue()
	fmt.Println(v)              // a
	q.Enqueue("d")              // wraps around to index 0
	for !q.IsEmpty() {
		v, _ := q.Dequeue()
		fmt.Print(v, " ")
	}
	fmt.Println() // b c d

	fmt.Println(SlidingWindowMax([]int{1, 3, -1, -3, 5, 3, 6, 7}, 3))
	// [3 3 5 5 6 7]
}
```

### 2. TypeScript

```typescript
/**
 * Fixed-capacity ring buffer. Never shifts, never reallocates.
 * Enqueue returns false when full instead of growing without bound.
 */
class RingQueue<T> implements Iterable<T> {
  #buf: (T | undefined)[];
  #head = 0;
  #tail = 0;
  #count = 0;

  constructor(capacity: number) {
    this.#buf = new Array<T | undefined>(Math.max(1, capacity));
  }

  get size(): number { return this.#count; }
  get capacity(): number { return this.#buf.length; }
  get isEmpty(): boolean { return this.#count === 0; }
  get isFull(): boolean { return this.#count === this.#buf.length; }

  enqueue(value: T): boolean {
    if (this.isFull) return false;
    this.#buf[this.#tail] = value;
    this.#tail = (this.#tail + 1) % this.#buf.length;
    this.#count++;
    return true;
  }

  dequeue(): T | undefined {
    if (this.#count === 0) return undefined;
    const value = this.#buf[this.#head] as T;
    this.#buf[this.#head] = undefined;
    this.#head = (this.#head + 1) % this.#buf.length;
    this.#count--;
    return value;
  }

  peek(): T | undefined {
    return this.#count === 0 ? undefined : (this.#buf[this.#head] as T);
  }

  *[Symbol.iterator](): Iterator<T> {
    for (let i = 0; i < this.#count; i++) {
      yield this.#buf[(this.#head + i) % this.#buf.length] as T;
    }
  }
}

/**
 * Unbounded queue with an O(1) amortized dequeue.
 * Using array.shift() here would make it O(n) per dequeue.
 */
class Queue<T> implements Iterable<T> {
  #items: T[] = [];
  #head = 0;

  get size(): number { return this.#items.length - this.#head; }
  get isEmpty(): boolean { return this.size === 0; }

  enqueue(value: T): void {
    this.#items.push(value);
  }

  dequeue(): T | undefined {
    if (this.#head >= this.#items.length) return undefined;
    const value = this.#items[this.#head++];
    // Compact once half the array is dead space.
    if (this.#head * 2 >= this.#items.length) {
      this.#items = this.#items.slice(this.#head);
      this.#head = 0;
    }
    return value;
  }

  peek(): T | undefined {
    return this.#head < this.#items.length ? this.#items[this.#head] : undefined;
  }

  *[Symbol.iterator](): Iterator<T> {
    for (let i = this.#head; i < this.#items.length; i++) yield this.#items[i];
  }
}

/** Both ends, all O(1) amortized. A stack and a queue in one type. */
class Deque<T> {
  #items: T[] = [];

  get size(): number { return this.#items.length; }
  pushBack(v: T): void { this.#items.push(v); }
  pushFront(v: T): void { this.#items.unshift(v); }
  popBack(): T | undefined { return this.#items.pop(); }
  popFront(): T | undefined { return this.#items.shift(); }
  get front(): T | undefined { return this.#items[0]; }
  get back(): T | undefined { return this.#items.at(-1); }
}

/** Monotonic decreasing deque of indices. O(n) total. */
function slidingWindowMax(nums: readonly number[], k: number): number[] {
  if (k <= 0 || k > nums.length) return [];
  const out: number[] = [];
  const dq: number[] = []; // indices; their values strictly decrease

  for (let i = 0; i < nums.length; i++) {
    if (dq.length > 0 && dq[0] <= i - k) dq.shift();      // slid out
    while (dq.length > 0 && nums[dq.at(-1)!] <= nums[i]) dq.pop(); // dominated
    dq.push(i);
    if (i >= k - 1) out.push(nums[dq[0]]);
  }
  return out;
}

const rq = new RingQueue<string>(3);
rq.enqueue("a"); rq.enqueue("b"); rq.enqueue("c");
console.log(rq.enqueue("d")); // false, full
console.log(rq.dequeue());    // "a"
rq.enqueue("d");              // wraps to slot 0
console.log([...rq]);         // ["b", "c", "d"]

console.log(slidingWindowMax([1, 3, -1, -3, 5, 3, 6, 7], 3));
// [3, 3, 5, 5, 6, 7]
```

## Practical Applications & Real-World Use Cases

- **Message brokers.** Kafka, RabbitMQ, and SQS are queues with durability bolted on. Producers enqueue, consumers dequeue, and order is preserved per partition.
- **Breadth-first search.** BFS is defined by its queue. Swap the queue for a stack and you get DFS. See [[14-Graph-Traversal-DFS-BFS]].
- **Task and job scheduling.** Worker pools pull from a shared queue. Round-robin CPU scheduling is a queue per priority level.
- **Ring buffers in drivers.** Network cards and audio devices hand the OS a fixed circular buffer. Fixed size means no allocation in the interrupt path.
- **Rate limiting.** A sliding-window limiter keeps request timestamps in a deque, dropping those older than the window.
- **Event loops.** The JavaScript task queue and microtask queue are exactly this, and the ordering rules between them are why `Promise` callbacks beat `setTimeout`.
- **Printer spools and request buffering.** The original textbook example, still accurate.

## Related Concepts & Connections

- [[00-DSA-Index|Back to DSA Index]]
- [[04-Stacks]] for the LIFO counterpart
- [[06-Priority-Queue-and-Heap]] for when arrival order is the wrong order
- [[14-Graph-Traversal-DFS-BFS]] for the single most important queue application
- [[02-Arrays]] for the ring buffer's backing store
