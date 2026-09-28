---
tags:
  - dsa
  - computer-science
  - go
  - typescript
  - searching
type: study-note
status: complete
date_created: 2026-09-28
---

# Searching Algorithms

## Overview

Searching answers "where is this value, or is it here at all". Two algorithms cover nearly every case. Linear search checks every element and works on anything. Binary search repeatedly halves a sorted range and works only on sorted data with random access.

The interesting part is not the code, which is short. It is knowing when the O(n log n) cost of sorting first is worth paying to get O(log n) lookups afterward, and recognizing the problems that are secretly binary search in disguise.

## Key Concepts & Analogies

**Linear search is flipping through an unsorted stack of papers.** No shortcuts available. You stop when you find it or run out.

**Binary search is the dictionary flip.** Open the middle. Too far? The answer is in the left half. Not far enough? Right half. Each check throws away half the remaining candidates, so a million entries take twenty checks.

**Binary search preconditions matter and people forget them.** The data must be sorted by the key you search on, and you must be able to jump to index `mid` in O(1). That rules out linked lists, where finding the middle costs O(n) and destroys the benefit.

**The overflow bug.** `mid := (lo + hi) / 2` overflows when `lo + hi` exceeds the integer range. Jon Bentley wrote that the binary search in the JDK carried this bug for nine years. Write `mid := lo + (hi-lo)/2` instead. Go's `int` is 64-bit so it rarely bites in practice, but the habit is free.

**Lower bound and upper bound are more useful than plain search.** Lower bound gives the first index where the value could be inserted keeping order. Upper bound gives the last. Together they give you the count of occurrences in O(log n) and the insertion point for a sorted insert. Most real binary search usage is one of these, not "find this exact element".

**Binary search on the answer.** When a problem asks for the minimum value satisfying some monotone predicate, binary search the answer space rather than an array. "What is the smallest ship capacity that finishes deliveries in D days?" The predicate "capacity c works" is false below a threshold and true above it, so you binary search the threshold. Spotting monotonicity is the whole skill here, and it converts a lot of apparently hard problems into fifteen lines.

**Exponential search for unbounded ranges.** Double an index until you overshoot, then binary search inside that window. O(log i) where i is the target position, which beats O(log n) when the target is near the front. Useful for searching a stream or an API with unknown length.

**Interpolation search** guesses the position from the value's magnitude instead of always taking the midpoint. On uniformly distributed data it hits O(log log n). On skewed data it degrades to O(n), which is why nobody uses it by default.

**Hashing beats both when you only need equality.** O(1) average lookup, no sorting required. Binary search stays relevant because it gives you ordering: predecessors, successors, and ranges. See [[07-Hash-Tables-and-Sets]].

## Operations & Time/Space Complexity

| Algorithm | Average Case | Worst Case | Space Complexity | Requires sorted |
| :--- | :--- | :--- | :--- | :--- |
| Linear search | O(n) | O(n) | O(1) | No |
| Binary search (iterative) | O(log n) | O(log n) | O(1) | Yes |
| Binary search (recursive) | O(log n) | O(log n) | O(log n) stack | Yes |
| Lower / upper bound | O(log n) | O(log n) | O(1) | Yes |
| Exponential search | O(log i) | O(log n) | O(1) | Yes |
| Jump search | O(√n) | O(√n) | O(1) | Yes |
| Interpolation search | O(log log n) | O(n) | O(1) | Yes, uniform |
| Hash lookup | O(1) | O(n) | O(n) table | No |
| BST search | O(log n) | O(n) unbalanced | O(n) tree | Ordered structure |

i is the index of the target in exponential search.

## Code Implementations

### 1. Go (Golang)

```go
package main

import (
	"cmp"
	"fmt"
)

// LinearSearch works on anything. O(n), no preconditions.
func LinearSearch[T comparable](xs []T, target T) int {
	for i, x := range xs {
		if x == target {
			return i
		}
	}
	return -1
}

// BinarySearch on a sorted slice. Iterative, so O(1) space.
// The half-open interval [lo, hi) keeps the arithmetic simple.
func BinarySearch[T cmp.Ordered](xs []T, target T) int {
	lo, hi := 0, len(xs)
	for lo < hi {
		mid := lo + (hi-lo)/2 // never overflows, unlike (lo+hi)/2
		switch {
		case xs[mid] == target:
			return mid
		case xs[mid] < target:
			lo = mid + 1
		default:
			hi = mid
		}
	}
	return -1
}

// LowerBound returns the first index with xs[i] >= target. That is the
// insertion point that preserves order. Returns len(xs) if none.
func LowerBound[T cmp.Ordered](xs []T, target T) int {
	lo, hi := 0, len(xs)
	for lo < hi {
		mid := lo + (hi-lo)/2
		if xs[mid] < target {
			lo = mid + 1
		} else {
			hi = mid
		}
	}
	return lo
}

// UpperBound returns the first index with xs[i] > target.
func UpperBound[T cmp.Ordered](xs []T, target T) int {
	lo, hi := 0, len(xs)
	for lo < hi {
		mid := lo + (hi-lo)/2
		if xs[mid] <= target {
			lo = mid + 1
		} else {
			hi = mid
		}
	}
	return lo
}

// CountOccurrences in O(log n) using the two bounds.
func CountOccurrences[T cmp.Ordered](xs []T, target T) int {
	return UpperBound(xs, target) - LowerBound(xs, target)
}

// ExponentialSearch doubles a bound until it passes the target, then
// binary searches the window. Good when the target is near the front
// or the length is unknown.
func ExponentialSearch[T cmp.Ordered](xs []T, target T) int {
	if len(xs) == 0 {
		return -1
	}
	if xs[0] == target {
		return 0
	}
	bound := 1
	for bound < len(xs) && xs[bound] < target {
		bound *= 2
	}
	lo := bound / 2
	hi := min(bound+1, len(xs))
	if idx := BinarySearch(xs[lo:hi], target); idx >= 0 {
		return lo + idx
	}
	return -1
}

// SearchRotated handles a sorted array rotated at an unknown pivot.
// Still O(log n): one of the two halves is always properly sorted, and
// checking which one tells you where the target can live.
func SearchRotated(xs []int, target int) int {
	lo, hi := 0, len(xs)-1
	for lo <= hi {
		mid := lo + (hi-lo)/2
		if xs[mid] == target {
			return mid
		}
		if xs[lo] <= xs[mid] { // left half is sorted
			if target >= xs[lo] && target < xs[mid] {
				hi = mid - 1
			} else {
				lo = mid + 1
			}
		} else { // right half is sorted
			if target > xs[mid] && target <= xs[hi] {
				lo = mid + 1
			} else {
				hi = mid - 1
			}
		}
	}
	return -1
}

// --- Binary search on the answer ---

// SearchAnswer finds the smallest x in [lo,hi] where pred(x) is true,
// assuming pred is false then true (monotone). This generalization is
// worth more than any specific search function.
func SearchAnswer(lo, hi int, pred func(int) bool) int {
	for lo < hi {
		mid := lo + (hi-lo)/2
		if pred(mid) {
			hi = mid
		} else {
			lo = mid + 1
		}
	}
	return lo
}

// MinShipCapacity: what is the smallest daily capacity that ships all
// packages within `days`? Capacity is monotone, so binary search it.
func MinShipCapacity(weights []int, days int) int {
	lo, hi := 0, 0
	for _, w := range weights {
		lo = max(lo, w) // must at least fit the heaviest package
		hi += w         // ship everything in one day
	}
	return SearchAnswer(lo, hi, func(capacity int) bool {
		needed, load := 1, 0
		for _, w := range weights {
			if load+w > capacity {
				needed++
				load = 0
			}
			load += w
		}
		return needed <= days
	})
}

// IntSqrt: the largest x with x*x <= n. Binary search, no floats.
func IntSqrt(n int) int {
	if n < 2 {
		return n
	}
	lo, hi := 1, n/2+1
	for lo < hi {
		mid := lo + (hi-lo+1)/2 // bias up to avoid an infinite loop
		if mid*mid <= n {
			lo = mid
		} else {
			hi = mid - 1
		}
	}
	return lo
}

func main() {
	xs := []int{1, 3, 3, 3, 5, 7, 9, 11}
	fmt.Println(LinearSearch(xs, 7))       // 5
	fmt.Println(BinarySearch(xs, 7))       // 5
	fmt.Println(BinarySearch(xs, 8))       // -1
	fmt.Println(LowerBound(xs, 3))         // 1
	fmt.Println(UpperBound(xs, 3))         // 4
	fmt.Println(CountOccurrences(xs, 3))   // 3
	fmt.Println(ExponentialSearch(xs, 11)) // 7

	fmt.Println(SearchRotated([]int{4, 5, 6, 7, 0, 1, 2}, 0)) // 4

	fmt.Println(MinShipCapacity([]int{1, 2, 3, 4, 5, 6, 7, 8, 9, 10}, 5)) // 15
	fmt.Println(IntSqrt(2_000_000))                                       // 1414
}
```

### 2. TypeScript

```typescript
type Compare<T> = (a: T, b: T) => number;
const defaultCompare = <T>(a: T, b: T): number => (a < b ? -1 : a > b ? 1 : 0);

/** O(n). Works on unsorted data. */
function linearSearch<T>(xs: readonly T[], target: T): number {
  for (let i = 0; i < xs.length; i++) if (xs[i] === target) return i;
  return -1;
}

/** O(log n) on sorted input. Half-open interval [lo, hi). */
function binarySearch<T>(
  xs: readonly T[],
  target: T,
  compare: Compare<T> = defaultCompare,
): number {
  let lo = 0;
  let hi = xs.length;
  while (lo < hi) {
    const mid = lo + ((hi - lo) >> 1); // no overflow, no float mid
    const c = compare(xs[mid], target);
    if (c === 0) return mid;
    if (c < 0) lo = mid + 1;
    else hi = mid;
  }
  return -1;
}

/** First index where xs[i] >= target. The order-preserving insert point. */
function lowerBound<T>(
  xs: readonly T[],
  target: T,
  compare: Compare<T> = defaultCompare,
): number {
  let lo = 0;
  let hi = xs.length;
  while (lo < hi) {
    const mid = lo + ((hi - lo) >> 1);
    if (compare(xs[mid], target) < 0) lo = mid + 1;
    else hi = mid;
  }
  return lo;
}

/** First index where xs[i] > target. */
function upperBound<T>(
  xs: readonly T[],
  target: T,
  compare: Compare<T> = defaultCompare,
): number {
  let lo = 0;
  let hi = xs.length;
  while (lo < hi) {
    const mid = lo + ((hi - lo) >> 1);
    if (compare(xs[mid], target) <= 0) lo = mid + 1;
    else hi = mid;
  }
  return lo;
}

/** O(log n) occurrence count from the two bounds. */
function countOccurrences<T>(xs: readonly T[], target: T): number {
  return upperBound(xs, target) - lowerBound(xs, target);
}

/** Keeps an array sorted with O(log n) search plus O(n) splice. */
function insertSorted<T>(xs: T[], value: T, compare: Compare<T> = defaultCompare): T[] {
  xs.splice(lowerBound(xs, value, compare), 0, value);
  return xs;
}

/** Doubling probe then a bounded binary search. O(log i). */
function exponentialSearch<T>(xs: readonly T[], target: T): number {
  if (xs.length === 0) return -1;
  if (xs[0] === target) return 0;
  let bound = 1;
  while (bound < xs.length && xs[bound] < target) bound *= 2;
  const lo = bound >> 1;
  const hi = Math.min(bound + 1, xs.length);
  const idx = binarySearch(xs.slice(lo, hi), target);
  return idx < 0 ? -1 : lo + idx;
}

/** Rotated sorted array. One half is always properly sorted. */
function searchRotated(xs: readonly number[], target: number): number {
  let lo = 0;
  let hi = xs.length - 1;
  while (lo <= hi) {
    const mid = lo + ((hi - lo) >> 1);
    if (xs[mid] === target) return mid;
    if (xs[lo] <= xs[mid]) {
      // left sorted
      if (target >= xs[lo] && target < xs[mid]) hi = mid - 1;
      else lo = mid + 1;
    } else {
      // right sorted
      if (target > xs[mid] && target <= xs[hi]) lo = mid + 1;
      else hi = mid - 1;
    }
  }
  return -1;
}

/**
 * Binary search on the answer space. Finds the smallest x in [lo, hi]
 * where predicate(x) holds, given the predicate is monotone
 * (false... then true...). The most reusable idea on this page.
 */
function searchAnswer(lo: number, hi: number, predicate: (x: number) => boolean): number {
  while (lo < hi) {
    const mid = lo + Math.floor((hi - lo) / 2);
    if (predicate(mid)) hi = mid;
    else lo = mid + 1;
  }
  return lo;
}

/** Smallest daily capacity that ships everything within `days`. */
function minShipCapacity(weights: readonly number[], days: number): number {
  const lo = Math.max(...weights);          // must fit the heaviest item
  const hi = weights.reduce((a, b) => a + b); // one day for everything
  return searchAnswer(lo, hi, (capacity) => {
    let needed = 1;
    let load = 0;
    for (const w of weights) {
      if (load + w > capacity) {
        needed++;
        load = 0;
      }
      load += w;
    }
    return needed <= days;
  });
}

/** Integer square root without floating point. */
function intSqrt(n: number): number {
  if (n < 2) return n;
  let lo = 1;
  let hi = Math.floor(n / 2) + 1;
  while (lo < hi) {
    const mid = lo + Math.ceil((hi - lo) / 2); // bias up so it terminates
    if (mid * mid <= n) lo = mid;
    else hi = mid - 1;
  }
  return lo;
}

const xs = [1, 3, 3, 3, 5, 7, 9, 11];
console.log(linearSearch(xs, 7), binarySearch(xs, 7)); // 5 5
console.log(binarySearch(xs, 8));                      // -1
console.log(lowerBound(xs, 3), upperBound(xs, 3));     // 1 4
console.log(countOccurrences(xs, 3));                  // 3
console.log(exponentialSearch(xs, 11));                // 7
console.log(searchRotated([4, 5, 6, 7, 0, 1, 2], 0));  // 4
console.log(minShipCapacity([1,2,3,4,5,6,7,8,9,10], 5)); // 15
console.log(intSqrt(2_000_000));                       // 1414
console.log(insertSorted([1, 3, 5, 7], 4));            // [1, 3, 4, 5, 7]
```

## Practical Applications & Real-World Use Cases

- **Database index lookups.** A B-tree descent is binary search generalized to many keys per node. Every indexed `WHERE` clause runs one.
- **Version bisection.** `git bisect` binary searches commit history for the one that introduced a bug. Fifteen builds to find the culprit in 30,000 commits.
- **Autoscaling and capacity tuning.** "What is the smallest instance count that keeps p99 latency under 200ms" is binary search on the answer, with a load test as the predicate.
- **Rate limiter and quota calibration.** Same shape: monotone predicate, search the threshold.
- **Debugging by bisection.** Comment out half the code, see if the bug persists. Everyone does this; fewer people notice it is O(log n).
- **Timestamp lookup in time series.** Finding the first sample after a given time is lower bound, not exact search.
- **Spell check candidate generation and IP geolocation.** Both look up a value inside sorted ranges, which is upper bound minus one.

## Related Concepts & Connections

- [[00-DSA-Index|Back to DSA Index]]
- [[12-Sorting-Algorithms]] because binary search needs sorted input
- [[07-Hash-Tables-and-Sets]] for O(1) equality lookup without ordering
- [[08-Trees-BST-and-AVL]] for binary search as a persistent structure
- [[02-Arrays]] for the random access binary search depends on
- [[13-Recursion]] for the recursive formulation and its O(log n) stack cost
