---
tags:
  - dsa
  - computer-science
  - go
  - typescript
  - arrays
type: study-note
status: complete
date_created: 2026-09-28
---

# Arrays

## Overview

An array is a block of contiguous memory holding elements of the same type. Because every element is the same size and they sit side by side, the address of element `i` is just `base + i * elementSize`. That single multiplication is the whole reason arrays are fast: random access costs one arithmetic operation, no matter how large the array is.

A dynamic array (Go's slice, TypeScript's `Array`, C++'s `vector`) wraps a fixed block with a length and a capacity. When you append past capacity, it allocates a bigger block, copies everything over, and frees the old one.

## Key Concepts & Analogies

**A street of identical houses.** House numbers run 0, 1, 2, and every house is the same width. To reach house 47 you do not walk past the first 46, you compute where it is and go straight there. A linked list is more like a scavenger hunt where each house holds the address of the next one.

**Fixed size versus dynamic.** A true array has a size baked in at creation. Go's `[5]int` is a fixed array and its length is part of its type. Go's `[]int` is a slice: a three-word header of pointer, length, capacity, pointing at a backing array. JavaScript arrays are dynamic by default and are objects under the hood, which is why they can be sparse and hold mixed types.

**Growth and amortized O(1) append.** When capacity runs out, the array typically doubles. Doubling means the copies get rarer as the array grows: inserting n elements costs n + n/2 + n/4 + ... total copy work, which sums to under 2n. Divide by n appends and you get constant cost per append on average. Individual appends still spike to O(n), which matters if you have a latency budget.

**Insertion and deletion in the middle are O(n).** Everything after the insertion point has to shift. There is no way around this; it is the price of contiguity.

**Cache locality is the underrated win.** When the CPU reads one element it pulls in a whole cache line, typically 64 bytes, so the next several elements come along free. Iterating an array is close to the fastest thing a computer does. This is why arrays often beat "asymptotically better" structures at realistic sizes.

**Two-pointer and sliding-window techniques** exist because arrays support cheap index arithmetic. Most "do this in O(n) without extra space" array problems are one of these two patterns.

## Operations & Time/Space Complexity

| Operation | Average Case | Worst Case | Space Complexity |
| :--- | :--- | :--- | :--- |
| Access by index | O(1) | O(1) | O(1) |
| Search (unsorted) | O(n) | O(n) | O(1) |
| Search (sorted, binary) | O(log n) | O(log n) | O(1) |
| Append | O(1) amortized | O(n) on resize | O(n) on resize |
| Insert at index | O(n) | O(n) | O(1) |
| Delete at index | O(n) | O(n) | O(1) |
| Delete last | O(1) | O(1) | O(1) |

Total storage is O(n). A dynamic array may hold up to 2x its length in allocated capacity.

## Code Implementations

### 1. Go (Golang)

```go
package main

import "fmt"

// DynamicArray reimplements what a Go slice already does, so the growth
// mechanics are visible instead of hidden behind append().
type DynamicArray[T any] struct {
	data []T // backing store; len(data) is the capacity
	size  int
}

func NewDynamicArray[T any](capacity int) *DynamicArray[T] {
	if capacity < 1 {
		capacity = 1
	}
	return &DynamicArray[T]{data: make([]T, capacity)}
}

func (a *DynamicArray[T]) Len() int { return a.size }
func (a *DynamicArray[T]) Cap() int { return len(a.data) }

// grow doubles the backing store and copies. This is the O(n) spike.
func (a *DynamicArray[T]) grow() {
	bigger := make([]T, len(a.data)*2)
	copy(bigger, a.data[:a.size])
	a.data = bigger
}

// Append is O(1) amortized.
func (a *DynamicArray[T]) Append(v T) {
	if a.size == len(a.data) {
		a.grow()
	}
	a.data[a.size] = v
	a.size++
}

// Get is O(1).
func (a *DynamicArray[T]) Get(i int) (T, error) {
	var zero T
	if i < 0 || i >= a.size {
		return zero, fmt.Errorf("index %d out of range [0,%d)", i, a.size)
	}
	return a.data[i], nil
}

// Insert is O(n): everything from i onward shifts right one slot.
func (a *DynamicArray[T]) Insert(i int, v T) error {
	if i < 0 || i > a.size {
		return fmt.Errorf("index %d out of range [0,%d]", i, a.size)
	}
	if a.size == len(a.data) {
		a.grow()
	}
	copy(a.data[i+1:a.size+1], a.data[i:a.size])
	a.data[i] = v
	a.size++
	return nil
}

// Delete is O(n): everything after i shifts left one slot.
func (a *DynamicArray[T]) Delete(i int) error {
	if i < 0 || i >= a.size {
		return fmt.Errorf("index %d out of range [0,%d)", i, a.size)
	}
	copy(a.data[i:], a.data[i+1:a.size])
	var zero T
	a.data[a.size-1] = zero // release the reference so the GC can reclaim it
	a.size--
	return nil
}

func (a *DynamicArray[T]) Slice() []T { return a.data[:a.size] }

// --- Common array techniques ---

// TwoSum in O(n) using a seen-map. The index-of-complement trick.
func TwoSum(nums []int, target int) (int, int, bool) {
	seen := make(map[int]int, len(nums))
	for i, n := range nums {
		if j, ok := seen[target-n]; ok {
			return j, i, true
		}
		seen[n] = i
	}
	return 0, 0, false
}

// ReverseInPlace: the two-pointer pattern. O(n) time, O(1) space.
func ReverseInPlace[T any](xs []T) {
	for l, r := 0, len(xs)-1; l < r; l, r = l+1, r-1 {
		xs[l], xs[r] = xs[r], xs[l]
	}
}

// MaxSubarraySum is Kadane's algorithm. One pass, no extra space.
func MaxSubarraySum(nums []int) int {
	if len(nums) == 0 {
		return 0
	}
	best, current := nums[0], nums[0]
	for _, n := range nums[1:] {
		if current+n > n {
			current += n
		} else {
			current = n // starting fresh here beats extending
		}
		if current > best {
			best = current
		}
	}
	return best
}

func main() {
	a := NewDynamicArray[string](2)
	for _, s := range []string{"go", "rust", "zig", "odin"} {
		a.Append(s)
		fmt.Printf("append %-5s len=%d cap=%d\n", s, a.Len(), a.Cap())
	}
	a.Insert(1, "c")
	a.Delete(0)
	fmt.Println(a.Slice()) // [c rust zig odin]

	i, j, _ := TwoSum([]int{2, 7, 11, 15}, 26)
	fmt.Println(i, j) // 2 3

	nums := []int{-2, 1, -3, 4, -1, 2, 1, -5, 4}
	fmt.Println(MaxSubarraySum(nums)) // 6
	ReverseInPlace(nums)
	fmt.Println(nums)
}
```

### 2. TypeScript

```typescript
/**
 * Reimplements the growth mechanics that JS arrays hide.
 * A real JS array is an object with fancy engine optimizations;
 * this models the classic contiguous-buffer version.
 */
class DynamicArray<T> {
  #data: (T | undefined)[];
  #size = 0;

  constructor(capacity = 1) {
    this.#data = new Array<T | undefined>(Math.max(1, capacity));
  }

  get length(): number { return this.#size; }
  get capacity(): number { return this.#data.length; }

  /** Doubles the buffer and copies. The O(n) spike. */
  #grow(): void {
    const bigger = new Array<T | undefined>(this.#data.length * 2);
    for (let i = 0; i < this.#size; i++) bigger[i] = this.#data[i];
    this.#data = bigger;
  }

  /** O(1) amortized. */
  append(value: T): void {
    if (this.#size === this.#data.length) this.#grow();
    this.#data[this.#size++] = value;
  }

  /** O(1). */
  get(index: number): T {
    if (index < 0 || index >= this.#size) {
      throw new RangeError(`index ${index} out of range [0,${this.#size})`);
    }
    return this.#data[index] as T;
  }

  /** O(n): shift right from the tail so nothing is overwritten. */
  insert(index: number, value: T): void {
    if (index < 0 || index > this.#size) {
      throw new RangeError(`index ${index} out of range [0,${this.#size}]`);
    }
    if (this.#size === this.#data.length) this.#grow();
    for (let i = this.#size; i > index; i--) this.#data[i] = this.#data[i - 1];
    this.#data[index] = value;
    this.#size++;
  }

  /** O(n): shift left over the hole. */
  delete(index: number): T {
    if (index < 0 || index >= this.#size) {
      throw new RangeError(`index ${index} out of range [0,${this.#size})`);
    }
    const removed = this.#data[index] as T;
    for (let i = index; i < this.#size - 1; i++) this.#data[i] = this.#data[i + 1];
    this.#data[this.#size - 1] = undefined; // drop the reference
    this.#size--;
    return removed;
  }

  toArray(): T[] { return this.#data.slice(0, this.#size) as T[]; }

  *[Symbol.iterator](): Iterator<T> {
    for (let i = 0; i < this.#size; i++) yield this.#data[i] as T;
  }
}

// --- Common array techniques ---

/** O(n) via a map of value to index. */
function twoSum(nums: readonly number[], target: number): [number, number] | null {
  const seen = new Map<number, number>();
  for (let i = 0; i < nums.length; i++) {
    const j = seen.get(target - nums[i]);
    if (j !== undefined) return [j, i];
    seen.set(nums[i], i);
  }
  return null;
}

/** Two pointers. O(n) time, O(1) space. Mutates in place. */
function reverseInPlace<T>(xs: T[]): T[] {
  for (let l = 0, r = xs.length - 1; l < r; l++, r--) {
    [xs[l], xs[r]] = [xs[r], xs[l]];
  }
  return xs;
}

/** Kadane's algorithm. One pass. */
function maxSubarraySum(nums: readonly number[]): number {
  if (nums.length === 0) return 0;
  let best = nums[0];
  let current = nums[0];
  for (let i = 1; i < nums.length; i++) {
    current = Math.max(nums[i], current + nums[i]);
    best = Math.max(best, current);
  }
  return best;
}

/** Fixed-size sliding window. Also one pass. */
function maxWindowSum(nums: readonly number[], k: number): number {
  if (k <= 0 || k > nums.length) return 0;
  let windowSum = 0;
  for (let i = 0; i < k; i++) windowSum += nums[i];
  let best = windowSum;
  for (let i = k; i < nums.length; i++) {
    windowSum += nums[i] - nums[i - k]; // add the new, drop the old
    best = Math.max(best, windowSum);
  }
  return best;
}

const arr = new DynamicArray<string>(2);
for (const s of ["go", "rust", "zig", "odin"]) {
  arr.append(s);
  console.log(`append ${s} len=${arr.length} cap=${arr.capacity}`);
}
arr.insert(1, "c");
arr.delete(0);
console.log([...arr]); // ["c", "rust", "zig", "odin"]

console.log(twoSum([2, 7, 11, 15], 26));                        // [2, 3]
console.log(maxSubarraySum([-2, 1, -3, 4, -1, 2, 1, -5, 4]));   // 6
console.log(maxWindowSum([1, 12, -5, -6, 50, 3], 4));           // 51
```

## Practical Applications & Real-World Use Cases

- **Image buffers.** A bitmap is one long array of pixels indexed by `y * width + x`. Contiguity is what lets a GPU stream it.
- **Audio and signal processing.** Samples arrive as flat float arrays, and every DSP operation is a pass over one.
- **Ring buffers for logging and networking.** A fixed array plus two indices gives you a bounded queue with zero allocation per message. See [[05-Queues]].
- **Columnar databases.** Parquet and ClickHouse store each column as a contiguous array so that scanning one field does not drag the whole row through cache.
- **The backing store for almost everything else.** Heaps, hash tables, and dynamic programming tables are all arrays underneath. See [[06-Priority-Queue-and-Heap]] and [[07-Hash-Tables-and-Sets]].
- **Matrix math and tensors.** A 2D matrix is usually a 1D array with stride arithmetic, because real contiguity beats an array of row pointers.

## Related Concepts & Connections

- [[00-DSA-Index|Back to DSA Index]]
- [[01-Fundamentals-and-Introduction]] for amortized analysis
- [[03-Linked-Lists]] for the opposite trade: cheap middle insertion, no random access
- [[11-Searching-Algorithms]] for binary search, which needs array indexing
- [[12-Sorting-Algorithms]] for in-place sorts that exploit contiguous memory
