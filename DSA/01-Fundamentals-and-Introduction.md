---
tags:
  - dsa
  - computer-science
  - go
  - typescript
  - complexity-analysis
type: study-note
status: complete
date_created: 2026-09-28
---

# Fundamentals and Introduction

## Overview

A data structure is a decision about how to lay out data in memory. An algorithm is a decision about what steps to run over that layout. The two are joined at the hip, because the layout you pick determines which steps are cheap.

Complexity analysis is the tool for comparing those decisions without running the code. It answers one question: as the input grows, how does the work grow? Not "how many milliseconds", which depends on your CPU and the phase of the moon, but the shape of the growth curve.

## Key Concepts & Analogies

**Finding a name in a phone book.** Flipping page by page is O(n). Opening to the middle and halving the search space is O(log n). For a book with a million names, that is a million steps versus twenty. The difference is not a tweak, it is a different kind of program.

**Big-O drops constants and low-order terms.** `3n² + 500n + 9000` is O(n²). This looks like sloppiness and sometimes it is, because for n = 10 the 9000 dominates everything. Big-O describes behavior as n heads toward infinity. For small inputs, measure instead of theorize.

**Three notations, and only one you will use.**
- Big-O is the upper bound. The worst it can get.
- Big-Omega is the lower bound. The best it can get.
- Big-Theta is both, a tight bound.

People say Big-O and usually mean Big-Theta. Nobody will correct you outside a lecture hall.

**The growth ladder**, from best to worst:

O(1) < O(log n) < O(n) < O(n log n) < O(n²) < O(2ⁿ) < O(n!)

The jump from O(n log n) to O(n²) is the one that kills production systems. At n = 1,000,000 that is 20 million operations versus 1 trillion.

**Space complexity counts extra memory**, not the input itself. An in-place sort is O(1) auxiliary space even though the array it sorts is O(n).

**Amortized cost** is the average over a sequence of operations. Appending to a dynamic array occasionally triggers an O(n) copy, but the copies are rare enough that the per-append cost averages to O(1). See [[02-Arrays]].

**The memory hierarchy is invisible in Big-O and very visible in benchmarks.** A cache hit is ~1ns, a main-memory read is ~100ns. This is why an O(n) scan over a contiguous array often beats an O(log n) walk over a pointer-chasing tree for small n. Big-O tells you when to stop worrying about constants, not that constants do not exist.

## Operations & Time/Space Complexity

Reference costs for the structures covered in this vault, average case:

| Structure | Access | Search | Insertion | Deletion | Space |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Array | O(1) | O(n) | O(n) | O(n) | O(n) |
| Dynamic array (append) | O(1) | O(n) | O(1) amortized | O(n) | O(n) |
| Singly linked list | O(n) | O(n) | O(1) at head | O(1) at head | O(n) |
| Stack | O(n) | O(n) | O(1) | O(1) | O(n) |
| Queue | O(n) | O(n) | O(1) | O(1) | O(n) |
| Hash table | n/a | O(1) | O(1) | O(1) | O(n) |
| Binary search tree (balanced) | O(log n) | O(log n) | O(log n) | O(log n) | O(n) |
| Binary heap | O(1) for min/max | O(n) | O(log n) | O(log n) | O(n) |
| Trie | O(k) | O(k) | O(k) | O(k) | O(n·k) |

k is the length of the key. n is the number of elements.

## Code Implementations

### 1. Go (Golang)

```go
package main

import (
	"fmt"
	"time"
)

// O(1): work does not depend on n.
func first[T any](xs []T) (T, bool) {
	var zero T
	if len(xs) == 0 {
		return zero, false
	}
	return xs[0], true
}

// O(n): one pass.
func sum(xs []int) int {
	total := 0
	for _, x := range xs {
		total += x
	}
	return total
}

// O(log n): the search space halves every iteration.
func countHalvings(n int) int {
	steps := 0
	for n > 1 {
		n /= 2
		steps++
	}
	return steps
}

// O(n^2): nested passes. Fine for n=100, fatal for n=100000.
func hasDuplicateSlow(xs []int) bool {
	for i := 0; i < len(xs); i++ {
		for j := i + 1; j < len(xs); j++ {
			if xs[i] == xs[j] {
				return true
			}
		}
	}
	return false
}

// O(n) time, O(n) space. The classic trade: buy speed with memory.
func hasDuplicateFast(xs []int) bool {
	seen := make(map[int]struct{}, len(xs))
	for _, x := range xs {
		if _, ok := seen[x]; ok {
			return true
		}
		seen[x] = struct{}{}
	}
	return false
}

// Measuring beats guessing. Run this and watch the quadratic version blow up.
func benchmark(name string, n int, f func([]int) bool) {
	xs := make([]int, n)
	for i := range xs {
		xs[i] = i // worst case: no duplicates, so both scan everything
	}
	start := time.Now()
	f(xs)
	fmt.Printf("%-20s n=%-8d %v\n", name, n, time.Since(start))
}

func main() {
	fmt.Println(first([]string{"a", "b"}))     // a true
	fmt.Println(sum([]int{1, 2, 3, 4}))        // 10
	fmt.Println(countHalvings(1_000_000))      // 19

	for _, n := range []int{1_000, 10_000, 40_000} {
		benchmark("quadratic", n, hasDuplicateSlow)
		benchmark("linear", n, hasDuplicateFast)
	}
}
```

### 2. TypeScript

```typescript
// O(1): work does not depend on n.
function first<T>(xs: readonly T[]): T | undefined {
  return xs[0];
}

// O(n): one pass.
function sum(xs: readonly number[]): number {
  let total = 0;
  for (const x of xs) total += x;
  return total;
}

// O(log n): the search space halves every iteration.
function countHalvings(n: number): number {
  let steps = 0;
  while (n > 1) {
    n = Math.floor(n / 2);
    steps++;
  }
  return steps;
}

// O(n^2): nested passes.
function hasDuplicateSlow(xs: readonly number[]): boolean {
  for (let i = 0; i < xs.length; i++) {
    for (let j = i + 1; j < xs.length; j++) {
      if (xs[i] === xs[j]) return true;
    }
  }
  return false;
}

// O(n) time, O(n) space.
function hasDuplicateFast(xs: readonly number[]): boolean {
  const seen = new Set<number>();
  for (const x of xs) {
    if (seen.has(x)) return true;
    seen.add(x);
  }
  return false;
}

type Predicate = (xs: readonly number[]) => boolean;

function benchmark(name: string, n: number, f: Predicate): void {
  const xs = Array.from({ length: n }, (_, i) => i);
  const start = performance.now();
  f(xs);
  const ms = (performance.now() - start).toFixed(2);
  console.log(`${name.padEnd(12)} n=${String(n).padEnd(8)} ${ms}ms`);
}

console.log(first(["a", "b"]));        // "a"
console.log(sum([1, 2, 3, 4]));        // 10
console.log(countHalvings(1_000_000)); // 19

for (const n of [1_000, 10_000, 40_000]) {
  benchmark("quadratic", n, hasDuplicateSlow);
  benchmark("linear", n, hasDuplicateFast);
}
```

## Practical Applications & Real-World Use Cases

- **Capacity planning.** Knowing an endpoint does O(n²) work over a user's items tells you it will fall over at some item count. You can predict the outage before it happens.
- **Database index selection.** A B-tree index turns an O(n) table scan into O(log n). Understanding why is the difference between adding indexes by superstition and adding them on purpose.
- **Code review.** The most common real performance bug is a database query inside a loop. That is an O(n) round trip count where O(1) would do. Spotting it requires the vocabulary.
- **Interview screening.** Nearly every technical interview asks you to state the complexity of what you just wrote. It is table stakes.
- **Choosing a library.** When two packages solve the same problem, their complexity guarantees are usually the deciding factor.

## Related Concepts & Connections

- [[00-DSA-Index|Back to DSA Index]]
- [[02-Arrays]] for the first concrete structure and amortized analysis
- [[13-Recursion]] for analyzing recursive cost with recurrence relations
- [[12-Sorting-Algorithms]] for the O(n log n) lower bound on comparison sorts
- [[16-Algorithmic-Strategies]] for choosing an approach once you can price it
