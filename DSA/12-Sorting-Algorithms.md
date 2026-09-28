---
tags:
  - dsa
  - computer-science
  - go
  - typescript
  - sorting
type: study-note
status: complete
date_created: 2026-09-28
---

# Sorting Algorithms

## Overview

Sorting arranges elements by some ordering. It matters far beyond producing pretty output, because sorted data unlocks binary search, makes duplicates adjacent, makes merging cheap, and makes greedy algorithms correct.

Comparison-based sorts cannot beat O(n log n) in the worst case. The proof is information-theoretic: there are n! possible orderings, each comparison gives one bit, and log₂(n!) is Θ(n log n). Non-comparison sorts like counting and radix sort beat that bound by not comparing at all, at the cost of requiring structured keys.

In practice you call your language's sort. The reason to learn these is that the techniques (divide and conquer, partitioning, merging) reappear everywhere.

## Key Concepts & Analogies

**Insertion sort is how people sort a hand of cards.** Pick up each card and slide it into position among the ones you already hold. O(n²) in general, but O(n) on nearly sorted input and very fast on small arrays. This is why real sort implementations switch to insertion sort for subarrays under about 12 elements.

**Merge sort is the two-pile merge.** Split until each pile has one card, then repeatedly merge two sorted piles by comparing their tops. Guaranteed O(n log n), stable, needs O(n) scratch space.

**Quicksort picks a pivot and partitions.** Everything smaller goes left, larger goes right, then recurse. O(n log n) average with excellent constants and in-place operation. The worst case is O(n²) when the pivot is always extreme, which happens on already-sorted input with a naive first-element pivot. Random or median-of-three pivot selection makes that practically impossible.

**Stability is a real requirement, not trivia.** A stable sort keeps equal elements in their original relative order. It lets you sort by one field, then another, and keep the first as a tiebreaker. Merge sort is stable, quicksort is not, heapsort is not. Go's `sort.Slice` is unstable and `sort.SliceStable` exists for exactly this reason. JavaScript's `Array.prototype.sort` has been required to be stable since ES2019.

**Counting sort skips comparison entirely.** If keys are integers in a small range, count occurrences and rebuild. O(n + k) where k is the range. Sorting a million ages (0 to 120) this way is dramatically faster than any comparison sort. Sorting a million 64-bit integers this way would need a counter array of 2⁶⁴ slots, which is the limitation.

**Radix sort applies counting sort digit by digit**, least significant first, using a stable inner sort. O(d(n + k)) for d digits. This is what sorts strings and fixed-width integers at scale.

**Real-world sorts are hybrids.** Timsort (Python, Java for objects, JavaScript engines) finds existing sorted runs and merges them, so partially sorted real data sorts in nearly O(n). Pdqsort (Go since 1.19, Rust) is quicksort with pattern detection and a heapsort fallback that caps the worst case at O(n log n). Nobody ships a textbook algorithm.

**The comparator is where bugs live.** A comparator must be consistent: if `cmp(a,b) < 0` then `cmp(b,a) > 0`, and it must be transitive. An inconsistent comparator does not just give wrong order, it can crash or loop. The classic mistake is `(a, b) => a.score > b.score` returning a boolean instead of a number.

## Operations & Time/Space Complexity

| Algorithm | Best | Average | Worst | Space | Stable |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Bubble sort | O(n) | O(n²) | O(n²) | O(1) | Yes |
| Selection sort | O(n²) | O(n²) | O(n²) | O(1) | No |
| Insertion sort | O(n) | O(n²) | O(n²) | O(1) | Yes |
| Merge sort | O(n log n) | O(n log n) | O(n log n) | O(n) | Yes |
| Quicksort | O(n log n) | O(n log n) | O(n²) | O(log n) stack | No |
| Heapsort | O(n log n) | O(n log n) | O(n log n) | O(1) | No |
| Counting sort | O(n + k) | O(n + k) | O(n + k) | O(k) | Yes |
| Radix sort | O(d(n+k)) | O(d(n+k)) | O(d(n+k)) | O(n + k) | Yes |
| Timsort | O(n) | O(n log n) | O(n log n) | O(n) | Yes |

k is the key range, d is the number of digits. Bubble sort's O(n) best case requires the early-exit check; without it there is no best case.

## Code Implementations

### 1. Go (Golang)

```go
package main

import (
	"cmp"
	"fmt"
	"math/rand"
)

// InsertionSort: O(n^2) in general, O(n) on nearly sorted data.
// Fast enough on small slices that real sorts fall back to it.
func InsertionSort[T cmp.Ordered](xs []T) {
	for i := 1; i < len(xs); i++ {
		v := xs[i]
		j := i - 1
		for j >= 0 && xs[j] > v {
			xs[j+1] = xs[j]
			j--
		}
		xs[j+1] = v
	}
}

// SelectionSort: always O(n^2) but does only O(n) writes, which matters
// on media where writing is far more expensive than reading.
func SelectionSort[T cmp.Ordered](xs []T) {
	for i := 0; i < len(xs)-1; i++ {
		minIdx := i
		for j := i + 1; j < len(xs); j++ {
			if xs[j] < xs[minIdx] {
				minIdx = j
			}
		}
		xs[i], xs[minIdx] = xs[minIdx], xs[i]
	}
}

// BubbleSort with early exit. Included for completeness; never use it.
func BubbleSort[T cmp.Ordered](xs []T) {
	for n := len(xs); n > 1; n-- {
		swapped := false
		for i := 1; i < n; i++ {
			if xs[i-1] > xs[i] {
				xs[i-1], xs[i] = xs[i], xs[i-1]
				swapped = true
			}
		}
		if !swapped {
			return // already sorted, this is the O(n) best case
		}
	}
}

// MergeSort: guaranteed O(n log n), stable, O(n) extra space.
// Allocating the buffer once instead of per level saves a lot of GC work.
func MergeSort[T cmp.Ordered](xs []T) {
	if len(xs) < 2 {
		return
	}
	buf := make([]T, len(xs))
	mergeSort(xs, buf)
}

func mergeSort[T cmp.Ordered](xs, buf []T) {
	if len(xs) < 12 {
		InsertionSort(xs) // small-slice cutoff, what real sorts do
		return
	}
	mid := len(xs) / 2
	mergeSort(xs[:mid], buf[:mid])
	mergeSort(xs[mid:], buf[mid:])
	merge(xs, buf, mid)
}

// merge walks two sorted halves. Using <= for the left side is what
// makes this stable.
func merge[T cmp.Ordered](xs, buf []T, mid int) {
	copy(buf, xs)
	left, right, out := 0, mid, 0
	for left < mid && right < len(xs) {
		if buf[left] <= buf[right] {
			xs[out] = buf[left]
			left++
		} else {
			xs[out] = buf[right]
			right++
		}
		out++
	}
	// One side is exhausted; copy whatever remains of the other.
	for left < mid {
		xs[out] = buf[left]
		left++
		out++
	}
	for right < len(xs) {
		xs[out] = buf[right]
		right++
		out++
	}
}

// QuickSort with a random pivot. The randomization is what makes the
// O(n^2) worst case practically unreachable.
func QuickSort[T cmp.Ordered](xs []T) {
	if len(xs) < 12 {
		InsertionSort(xs)
		return
	}
	p := partition(xs)
	QuickSort(xs[:p])
	QuickSort(xs[p+1:])
}

// partition is the Lomuto scheme: everything < pivot ends up left of
// the returned index, everything >= ends up right.
func partition[T cmp.Ordered](xs []T) int {
	r := rand.Intn(len(xs))
	xs[r], xs[len(xs)-1] = xs[len(xs)-1], xs[r] // pivot to the end
	pivot := xs[len(xs)-1]

	i := 0
	for j := 0; j < len(xs)-1; j++ {
		if xs[j] < pivot {
			xs[i], xs[j] = xs[j], xs[i]
			i++
		}
	}
	xs[i], xs[len(xs)-1] = xs[len(xs)-1], xs[i]
	return i
}

// ThreeWayQuickSort groups equal elements together. On data with many
// duplicates this turns O(n^2) into O(n).
func ThreeWayQuickSort[T cmp.Ordered](xs []T) {
	if len(xs) < 2 {
		return
	}
	pivot := xs[rand.Intn(len(xs))]
	lt, i, gt := 0, 0, len(xs)-1
	for i <= gt {
		switch {
		case xs[i] < pivot:
			xs[lt], xs[i] = xs[i], xs[lt]
			lt++
			i++
		case xs[i] > pivot:
			xs[i], xs[gt] = xs[gt], xs[i]
			gt--
		default:
			i++ // equal to pivot: leave it in the middle band
		}
	}
	ThreeWayQuickSort(xs[:lt])
	ThreeWayQuickSort(xs[gt+1:])
}

// HeapSort: O(n log n) worst case, in place, not stable.
func HeapSort[T cmp.Ordered](xs []T) {
	n := len(xs)
	for i := n/2 - 1; i >= 0; i-- { // O(n) heapify
		siftDown(xs, i, n)
	}
	for end := n - 1; end > 0; end-- {
		xs[0], xs[end] = xs[end], xs[0]
		siftDown(xs, 0, end)
	}
}

func siftDown[T cmp.Ordered](xs []T, i, n int) {
	for {
		largest := i
		if l := 2*i + 1; l < n && xs[l] > xs[largest] {
			largest = l
		}
		if r := 2*i + 2; r < n && xs[r] > xs[largest] {
			largest = r
		}
		if largest == i {
			return
		}
		xs[i], xs[largest] = xs[largest], xs[i]
		i = largest
	}
}

// CountingSort: O(n + k), no comparisons. Only for small integer ranges.
func CountingSort(xs []int, maxVal int) []int {
	counts := make([]int, maxVal+1)
	for _, x := range xs {
		counts[x]++
	}
	out := make([]int, 0, len(xs))
	for v, c := range counts {
		for ; c > 0; c-- {
			out = append(out, v)
		}
	}
	return out
}

// RadixSort: counting sort per digit, least significant first.
// Stability of the inner pass is what makes the whole thing correct.
func RadixSort(xs []int) {
	if len(xs) < 2 {
		return
	}
	maxVal := xs[0]
	for _, x := range xs {
		if x > maxVal {
			maxVal = x
		}
	}
	buf := make([]int, len(xs))
	for exp := 1; maxVal/exp > 0; exp *= 10 {
		var counts [10]int
		for _, x := range xs {
			counts[(x/exp)%10]++
		}
		for d := 1; d < 10; d++ {
			counts[d] += counts[d-1] // prefix sums give output positions
		}
		for i := len(xs) - 1; i >= 0; i-- { // right to left keeps it stable
			d := (xs[i] / exp) % 10
			counts[d]--
			buf[counts[d]] = xs[i]
		}
		copy(xs, buf)
	}
}

// QuickSelect finds the k-th smallest without sorting. O(n) average,
// because it recurses into only one side of the partition.
func QuickSelect[T cmp.Ordered](xs []T, k int) (T, bool) {
	var zero T
	if k < 0 || k >= len(xs) {
		return zero, false
	}
	work := make([]T, len(xs))
	copy(work, xs)
	for {
		if len(work) == 1 {
			return work[0], true
		}
		p := partition(work)
		switch {
		case p == k:
			return work[p], true
		case p > k:
			work = work[:p]
		default:
			work = work[p+1:]
			k -= p + 1
		}
	}
}

func main() {
	base := []int{38, 27, 43, 3, 9, 82, 10, 1, 55, 4, 91, 7, 22, 60}

	clone := func() []int { c := make([]int, len(base)); copy(c, base); return c }

	for name, sortFn := range map[string]func([]int){
		"insertion": InsertionSort,
		"selection": SelectionSort,
		"bubble":    BubbleSort,
		"merge":     MergeSort,
		"quick":     QuickSort,
		"threeway":  ThreeWayQuickSort,
		"heap":      HeapSort,
		"radix":     RadixSort,
	} {
		xs := clone()
		sortFn(xs)
		fmt.Printf("%-10s %v\n", name, xs)
	}

	fmt.Println(CountingSort([]int{4, 2, 2, 8, 3, 3, 1}, 8)) // [1 2 2 3 3 4 8]

	median, _ := QuickSelect(base, len(base)/2)
	fmt.Println("median-ish:", median)
}
```

### 2. TypeScript

```typescript
type Compare<T> = (a: T, b: T) => number;
const defaultCompare = <T>(a: T, b: T): number => (a < b ? -1 : a > b ? 1 : 0);

/** O(n^2) worst, O(n) on nearly sorted input. Mutates. */
function insertionSort<T>(xs: T[], compare: Compare<T> = defaultCompare): T[] {
  for (let i = 1; i < xs.length; i++) {
    const v = xs[i];
    let j = i - 1;
    while (j >= 0 && compare(xs[j], v) > 0) {
      xs[j + 1] = xs[j];
      j--;
    }
    xs[j + 1] = v;
  }
  return xs;
}

/** O(n^2) always, but only O(n) writes. */
function selectionSort<T>(xs: T[], compare: Compare<T> = defaultCompare): T[] {
  for (let i = 0; i < xs.length - 1; i++) {
    let minIdx = i;
    for (let j = i + 1; j < xs.length; j++) {
      if (compare(xs[j], xs[minIdx]) < 0) minIdx = j;
    }
    if (minIdx !== i) [xs[i], xs[minIdx]] = [xs[minIdx], xs[i]];
  }
  return xs;
}

/** Guaranteed O(n log n), stable. Returns a new array. */
function mergeSort<T>(xs: readonly T[], compare: Compare<T> = defaultCompare): T[] {
  if (xs.length < 2) return [...xs];
  const mid = xs.length >> 1;
  return merge(
    mergeSort(xs.slice(0, mid), compare),
    mergeSort(xs.slice(mid), compare),
    compare,
  );
}

/** <= 0 on the left keeps equal elements in original order: stable. */
function merge<T>(left: readonly T[], right: readonly T[], compare: Compare<T>): T[] {
  const out: T[] = [];
  let i = 0;
  let j = 0;
  while (i < left.length && j < right.length) {
    out.push(compare(left[i], right[j]) <= 0 ? left[i++] : right[j++]);
  }
  while (i < left.length) out.push(left[i++]);
  while (j < right.length) out.push(right[j++]);
  return out;
}

/** In-place quicksort with a random pivot. O(n log n) average. */
function quickSort<T>(
  xs: T[],
  compare: Compare<T> = defaultCompare,
  lo = 0,
  hi = xs.length - 1,
): T[] {
  while (lo < hi) {
    if (hi - lo < 12) {
      // Insertion sort the small tail in place.
      for (let i = lo + 1; i <= hi; i++) {
        const v = xs[i];
        let j = i - 1;
        while (j >= lo && compare(xs[j], v) > 0) xs[j + 1] = xs[j--];
        xs[j + 1] = v;
      }
      return xs;
    }
    const p = partition(xs, compare, lo, hi);
    // Recurse into the smaller side, loop on the larger: caps stack at O(log n).
    if (p - lo < hi - p) {
      quickSort(xs, compare, lo, p - 1);
      lo = p + 1;
    } else {
      quickSort(xs, compare, p + 1, hi);
      hi = p - 1;
    }
  }
  return xs;
}

/** Lomuto partition with a random pivot moved to the end. */
function partition<T>(xs: T[], compare: Compare<T>, lo: number, hi: number): number {
  const r = lo + Math.floor(Math.random() * (hi - lo + 1));
  [xs[r], xs[hi]] = [xs[hi], xs[r]];
  const pivot = xs[hi];
  let i = lo;
  for (let j = lo; j < hi; j++) {
    if (compare(xs[j], pivot) < 0) {
      [xs[i], xs[j]] = [xs[j], xs[i]];
      i++;
    }
  }
  [xs[i], xs[hi]] = [xs[hi], xs[i]];
  return i;
}

/** O(n log n) worst case, in place, not stable. */
function heapSort<T>(xs: T[], compare: Compare<T> = defaultCompare): T[] {
  const siftDown = (i: number, n: number): void => {
    for (;;) {
      let largest = i;
      const l = 2 * i + 1;
      const r = 2 * i + 2;
      if (l < n && compare(xs[l], xs[largest]) > 0) largest = l;
      if (r < n && compare(xs[r], xs[largest]) > 0) largest = r;
      if (largest === i) return;
      [xs[i], xs[largest]] = [xs[largest], xs[i]];
      i = largest;
    }
  };
  for (let i = (xs.length >> 1) - 1; i >= 0; i--) siftDown(i, xs.length);
  for (let end = xs.length - 1; end > 0; end--) {
    [xs[0], xs[end]] = [xs[end], xs[0]];
    siftDown(0, end);
  }
  return xs;
}

/** O(n + k), no comparisons. Only for small integer ranges. */
function countingSort(xs: readonly number[]): number[] {
  if (xs.length === 0) return [];
  const min = Math.min(...xs);
  const max = Math.max(...xs);
  const counts = new Array<number>(max - min + 1).fill(0);
  for (const x of xs) counts[x - min]++;
  const out: number[] = [];
  counts.forEach((c, i) => {
    for (let k = 0; k < c; k++) out.push(i + min);
  });
  return out;
}

/** Counting sort per digit, least significant first. Non-negative ints. */
function radixSort(xs: readonly number[]): number[] {
  if (xs.length < 2) return [...xs];
  let current = [...xs];
  const max = Math.max(...current);
  for (let exp = 1; Math.floor(max / exp) > 0; exp *= 10) {
    const buckets: number[][] = Array.from({ length: 10 }, () => []);
    for (const x of current) buckets[Math.floor(x / exp) % 10].push(x);
    current = buckets.flat(); // push order preserved, so each pass is stable
  }
  return current;
}

/** k-th smallest in O(n) average without fully sorting. */
function quickSelect<T>(
  input: readonly T[],
  k: number,
  compare: Compare<T> = defaultCompare,
): T | undefined {
  if (k < 0 || k >= input.length) return undefined;
  const xs = [...input];
  let lo = 0;
  let hi = xs.length - 1;
  while (lo <= hi) {
    const p = partition(xs, compare, lo, hi);
    if (p === k) return xs[p];
    if (p > k) hi = p - 1;
    else lo = p + 1;
  }
  return undefined;
}

/**
 * Multi-key sort. Works because Array.prototype.sort is stable in
 * ES2019 and later, so ties fall through to the next comparator.
 */
function sortBy<T>(xs: readonly T[], ...keys: ((item: T) => number | string)[]): T[] {
  return [...xs].sort((a, b) => {
    for (const key of keys) {
      const ka = key(a);
      const kb = key(b);
      if (ka < kb) return -1;
      if (ka > kb) return 1;
    }
    return 0;
  });
}

const base = [38, 27, 43, 3, 9, 82, 10, 1, 55, 4, 91, 7, 22, 60];

console.log(insertionSort([...base]));
console.log(selectionSort([...base]));
console.log(mergeSort(base));
console.log(quickSort([...base]));
console.log(heapSort([...base]));
console.log(countingSort([4, 2, 2, 8, 3, 3, 1])); // [1,2,2,3,3,4,8]
console.log(radixSort(base));
console.log(quickSelect(base, 7));

const people = [
  { name: "Ada", dept: "eng", level: 3 },
  { name: "Grace", dept: "eng", level: 1 },
  { name: "Alan", dept: "research", level: 2 },
];
console.log(sortBy(people, (p) => p.dept, (p) => p.level).map((p) => p.name));
// ["Grace", "Ada", "Alan"]
```

## Practical Applications & Real-World Use Cases

- **Database query execution.** `ORDER BY` runs a sort; so does a merge join, which sorts both inputs then walks them in lockstep. `GROUP BY` often sorts to make equal keys adjacent.
- **External sorting.** Files larger than RAM are sorted by splitting into memory-sized runs, sorting each, and k-way merging with a heap. This is `sort` on the command line and how MapReduce shuffles.
- **Leaderboards and feed ranking.** Sort by score, then by timestamp as a tiebreaker. Stability is what makes the two-pass version work.
- **Top-k without full sorting.** Use a bounded heap or quickselect. Sorting a million records to show ten is wasted work. See [[06-Priority-Queue-and-Heap]].
- **Computational geometry.** Convex hull, line-segment intersection, and closest-pair all begin by sorting points.
- **Interval scheduling.** Sort by end time, then greedily take non-overlapping intervals. The sort is what makes greedy correct here. See [[16-Algorithmic-Strategies]].
- **Compression.** Burrows-Wheeler transform, used in bzip2, sorts all rotations of the input. Huffman coding sorts symbols by frequency.
- **Deduplication.** Sort then scan adjacent pairs, for when a hash set would not fit in memory.

## Related Concepts & Connections

- [[00-DSA-Index|Back to DSA Index]]
- [[11-Searching-Algorithms]] for what sorted data buys you
- [[06-Priority-Queue-and-Heap]] for heapsort and for top-k instead of sorting
- [[13-Recursion]] for the divide-and-conquer recurrences behind merge and quick sort
- [[16-Algorithmic-Strategies]] for divide and conquer as a general technique
- [[01-Fundamentals-and-Introduction]] for the O(n log n) lower-bound argument
