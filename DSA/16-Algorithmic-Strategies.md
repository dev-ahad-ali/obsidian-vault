---
tags:
  - dsa
  - computer-science
  - go
  - typescript
  - algorithm-design
type: study-note
status: complete
date_created: 2026-09-28
---

# Algorithmic Strategies

## Overview

Individual algorithms are worth knowing. Strategies are worth more, because they tell you what to try on a problem you have never seen. There are four that cover most of what you will meet.

Divide and conquer splits a problem into independent subproblems. Greedy makes the locally best choice and never reconsiders. Dynamic programming solves overlapping subproblems once and reuses the answers. Backtracking explores choices systematically and undoes them when they fail.

Choosing among them is the actual skill. The sections below give the signals that point at each one.

## Key Concepts & Analogies

### Divide and conquer

Split into independent subproblems, solve each, combine. Merge sort, quicksort, and binary search are all this. The signal is that the subproblems do not overlap: sorting the left half tells you nothing about the right half. Cost follows a recurrence like `T(n) = a·T(n/b) + f(n)`, and the Master Theorem solves the common cases.

Karatsuba multiplication is the classic surprise: splitting two n-digit numbers and using three multiplications instead of four gives O(n^1.585) instead of O(n²). Nobody expects arithmetic to have a better-than-obvious algorithm.

### Greedy

Take the best-looking option right now, commit, move on. Fast and simple when it works, and when it does not work it fails quietly by producing a plausible wrong answer. That is the danger.

Greedy is correct only when the problem has the *greedy choice property*: some locally optimal choice is part of a globally optimal solution. Interval scheduling sorted by earliest end time is provably optimal. Coin change with US denominations is greedy-optimal. Coin change with {1, 3, 4} making 6 is not: greedy takes 4+1+1 for three coins, the optimum is 3+3 for two. Same problem shape, different data, and greedy silently breaks.

Prove the exchange argument or test against brute force on small inputs. Do not assume.

### Dynamic programming

Use DP when subproblems overlap and the problem has an optimal substructure (the optimal answer is built from optimal answers to subproblems). Naive recursion on overlapping subproblems is exponential; caching makes it polynomial.

Two directions:
- Top-down (memoization) is recursion plus a cache. Easier to write, follows the recurrence directly, only computes the states you need.
- Bottom-up (tabulation) fills a table in dependency order. No recursion overhead, and it usually reveals that you only need the last row or two, which cuts space from O(n·m) to O(m).

The hard part is never the code, it is defining the state. Once you can say "dp[i][j] means the best answer using the first i items with capacity j", the recurrence usually writes itself. When stuck, write the brute-force recursion first and then add a cache. That path works almost every time.

### Backtracking

Build a candidate incrementally and abandon a branch as soon as it cannot lead to a solution. It is DFS over a decision tree with undo. The multiplier is pruning: N-queens has 8⁸ = 16 million naive placements, but checking conflicts as you go cuts the search to about 2,000 nodes.

Structure is always the same three lines: choose, explore, undo. Getting the undo right is where bugs live.

### Other patterns worth naming

- Two pointers and sliding window: linear scans that maintain a window invariant. See [[02-Arrays]].
- Binary search on the answer: when the answer is monotone. See [[11-Searching-Algorithms]].
- Prefix sums: precompute cumulative sums for O(1) range queries.
- Bit manipulation: subsets as bitmasks, XOR for finding the unpaired element.

### How to pick

Ask in this order. Are subproblems independent? Divide and conquer. Do they overlap with optimal substructure? DP. Does one local choice provably stay optimal? Greedy. Do you need to enumerate valid configurations? Backtracking. None of these? Look for a graph in the problem.

## Operations & Time/Space Complexity

| Strategy | Typical time | Typical space | Signal to use it |
| :--- | :--- | :--- | :--- |
| Divide and conquer | O(n log n) | O(log n) to O(n) | Independent subproblems |
| Greedy | O(n log n) with sort | O(1) | Provable local-optimal choice |
| DP (memoized) | O(states × transitions) | O(states) | Overlapping subproblems |
| DP (tabulated) | O(states × transitions) | O(states), often reducible | Same, want no recursion |
| Backtracking | O(bᵈ) worst | O(d) depth | Enumerate or find valid configs |
| Two pointers | O(n) | O(1) | Sorted array, pair or window |
| Sliding window | O(n) | O(k) | Contiguous subarray property |
| Bitmask DP | O(2ⁿ · n) | O(2ⁿ) | Subset states, n ≤ 20 |

Specific problems: 0/1 knapsack O(n·W) time and O(W) space with the rolling-array trick. Edit distance O(n·m). LIS O(n log n) with binary search. N-queens roughly O(n!) with pruning.

## Code Implementations

### 1. Go (Golang)

```go
package main

import (
	"fmt"
	"sort"
)

// --- Divide and conquer ---

// MaxSubarrayDC solves maximum subarray by splitting. O(n log n).
// Kadane's does it in O(n), but this shows the D&C shape: the answer is
// either entirely left, entirely right, or crosses the midpoint.
func MaxSubarrayDC(nums []int) int {
	if len(nums) == 1 {
		return nums[0]
	}
	mid := len(nums) / 2
	left := MaxSubarrayDC(nums[:mid])
	right := MaxSubarrayDC(nums[mid:])

	// Best sum ending at mid-1, extending leftward.
	leftBest, sum := nums[mid-1], 0
	for i := mid - 1; i >= 0; i-- {
		sum += nums[i]
		leftBest = max(leftBest, sum)
	}
	// Best sum starting at mid, extending rightward.
	rightBest, sum := nums[mid], 0
	for i := mid; i < len(nums); i++ {
		sum += nums[i]
		rightBest = max(rightBest, sum)
	}
	return max(max(left, right), leftBest+rightBest)
}

// --- Greedy ---

type Interval struct{ Start, End int }

// MaxNonOverlapping sorts by end time and takes greedily. This is
// provably optimal: finishing earliest leaves the most room for the rest.
func MaxNonOverlapping(intervals []Interval) []Interval {
	sorted := make([]Interval, len(intervals))
	copy(sorted, intervals)
	sort.Slice(sorted, func(i, j int) bool { return sorted[i].End < sorted[j].End })

	var chosen []Interval
	lastEnd := -1 << 62
	for _, iv := range sorted {
		if iv.Start >= lastEnd {
			chosen = append(chosen, iv)
			lastEnd = iv.End
		}
	}
	return chosen
}

// CoinChangeGreedy is fast and sometimes wrong. With {1,3,4} and amount
// 6 it returns 3 (4+1+1) when the optimum is 2 (3+3).
func CoinChangeGreedy(coins []int, amount int) int {
	sorted := make([]int, len(coins))
	copy(sorted, coins)
	sort.Sort(sort.Reverse(sort.IntSlice(sorted)))

	count := 0
	for _, c := range sorted {
		count += amount / c
		amount %= c
	}
	if amount > 0 {
		return -1
	}
	return count
}

// --- Dynamic programming ---

// CoinChangeDP is always correct. O(amount * len(coins)).
// dp[a] = fewest coins summing to exactly a.
func CoinChangeDP(coins []int, amount int) int {
	const unreachable = 1 << 30
	dp := make([]int, amount+1)
	for i := 1; i <= amount; i++ {
		dp[i] = unreachable
	}
	for a := 1; a <= amount; a++ {
		for _, c := range coins {
			if c <= a && dp[a-c]+1 < dp[a] {
				dp[a] = dp[a-c] + 1
			}
		}
	}
	if dp[amount] >= unreachable {
		return -1
	}
	return dp[amount]
}

// Knapsack01: maximize value under a weight limit, each item once.
// dp[w] = best value achievable with capacity w.
func Knapsack01(weights, values []int, capacity int) int {
	dp := make([]int, capacity+1)
	for i := range weights {
		// Descending w is what enforces "each item at most once": an
		// ascending loop would let the same item be reused.
		for w := capacity; w >= weights[i]; w-- {
			if cand := dp[w-weights[i]] + values[i]; cand > dp[w] {
				dp[w] = cand
			}
		}
	}
	return dp[capacity]
}

// EditDistance (Levenshtein). dp[i][j] = cost to turn a[:i] into b[:j].
// Only the previous row is ever read, so space is O(len(b)).
func EditDistance(a, b string) int {
	prev := make([]int, len(b)+1)
	curr := make([]int, len(b)+1)
	for j := range prev {
		prev[j] = j // deleting everything from an empty prefix
	}

	for i := 1; i <= len(a); i++ {
		curr[0] = i
		for j := 1; j <= len(b); j++ {
			if a[i-1] == b[j-1] {
				curr[j] = prev[j-1] // free match
			} else {
				// insert, delete, or substitute
				curr[j] = 1 + min(prev[j-1], min(prev[j], curr[j-1]))
			}
		}
		prev, curr = curr, prev
	}
	return prev[len(b)]
}

// LongestIncreasingSubsequence in O(n log n). tails[k] holds the
// smallest possible tail of an increasing subsequence of length k+1.
func LongestIncreasingSubsequence(nums []int) int {
	var tails []int
	for _, n := range nums {
		i := sort.SearchInts(tails, n) // first index with tails[i] >= n
		if i == len(tails) {
			tails = append(tails, n)
		} else {
			tails[i] = n // a smaller tail leaves more room later
		}
	}
	return len(tails)
}

// LongestCommonSubsequence, the basis of diff tools.
func LongestCommonSubsequence(a, b string) int {
	prev := make([]int, len(b)+1)
	curr := make([]int, len(b)+1)
	for i := 1; i <= len(a); i++ {
		for j := 1; j <= len(b); j++ {
			if a[i-1] == b[j-1] {
				curr[j] = prev[j-1] + 1
			} else {
				curr[j] = max(prev[j], curr[j-1])
			}
		}
		prev, curr = curr, prev
		for j := range curr {
			curr[j] = 0
		}
	}
	return prev[len(b)]
}

// --- Backtracking ---

// SolveNQueens places n queens with no two attacking. Pruning by column
// and diagonal turns an intractable search into a fast one.
func SolveNQueens(n int) [][]int {
	var solutions [][]int
	queens := make([]int, 0, n) // queens[row] = column
	cols := make([]bool, n)
	diag1 := make([]bool, 2*n) // row + col
	diag2 := make([]bool, 2*n) // row - col + n

	var place func(row int)
	place = func(row int) {
		if row == n {
			solutions = append(solutions, append([]int(nil), queens...))
			return
		}
		for col := 0; col < n; col++ {
			if cols[col] || diag1[row+col] || diag2[row-col+n] {
				continue // prune: this square is attacked
			}
			// choose
			queens = append(queens, col)
			cols[col], diag1[row+col], diag2[row-col+n] = true, true, true
			// explore
			place(row + 1)
			// undo
			queens = queens[:len(queens)-1]
			cols[col], diag1[row+col], diag2[row-col+n] = false, false, false
		}
	}
	place(0)
	return solutions
}

// CombinationSum finds every multiset of candidates summing to target.
// Sorting lets us break early instead of checking every branch.
func CombinationSum(candidates []int, target int) [][]int {
	sorted := make([]int, len(candidates))
	copy(sorted, candidates)
	sort.Ints(sorted)

	var out [][]int
	var current []int

	var backtrack func(start, remaining int)
	backtrack = func(start, remaining int) {
		if remaining == 0 {
			out = append(out, append([]int(nil), current...))
			return
		}
		for i := start; i < len(sorted); i++ {
			if sorted[i] > remaining {
				break // sorted, so everything after is too big
			}
			current = append(current, sorted[i])
			backtrack(i, remaining-sorted[i]) // i, not i+1: reuse allowed
			current = current[:len(current)-1]
		}
	}
	backtrack(0, target)
	return out
}

// --- Two pointers and prefix sums ---

// ThreeSum finds triplets summing to zero. Sort, then for each element
// run two pointers over the rest. O(n^2) instead of O(n^3).
func ThreeSum(nums []int) [][]int {
	sorted := make([]int, len(nums))
	copy(sorted, nums)
	sort.Ints(sorted)

	var out [][]int
	for i := 0; i < len(sorted)-2; i++ {
		if i > 0 && sorted[i] == sorted[i-1] {
			continue // skip duplicate anchors
		}
		l, r := i+1, len(sorted)-1
		for l < r {
			sum := sorted[i] + sorted[l] + sorted[r]
			switch {
			case sum < 0:
				l++
			case sum > 0:
				r--
			default:
				out = append(out, []int{sorted[i], sorted[l], sorted[r]})
				for l < r && sorted[l] == sorted[l+1] {
					l++
				}
				for l < r && sorted[r] == sorted[r-1] {
					r--
				}
				l, r = l+1, r-1
			}
		}
	}
	return out
}

// PrefixSums trades O(n) setup for O(1) range queries.
type PrefixSums struct{ cumulative []int }

func NewPrefixSums(nums []int) *PrefixSums {
	c := make([]int, len(nums)+1)
	for i, n := range nums {
		c[i+1] = c[i] + n
	}
	return &PrefixSums{cumulative: c}
}

// RangeSum over [lo, hi) in O(1).
func (p *PrefixSums) RangeSum(lo, hi int) int {
	return p.cumulative[hi] - p.cumulative[lo]
}

func main() {
	nums := []int{-2, 1, -3, 4, -1, 2, 1, -5, 4}
	fmt.Println("max subarray:", MaxSubarrayDC(nums)) // 6

	meetings := []Interval{{1, 4}, {2, 3}, {3, 5}, {5, 7}, {6, 8}}
	fmt.Println("scheduled:", MaxNonOverlapping(meetings))
	// [{2 3} {3 5} {5 7}]

	// Greedy is wrong here, DP is right.
	fmt.Println(CoinChangeGreedy([]int{1, 3, 4}, 6)) // 3, wrong
	fmt.Println(CoinChangeDP([]int{1, 3, 4}, 6))     // 2, correct

	fmt.Println(Knapsack01([]int{1, 3, 4, 5}, []int{1, 4, 5, 7}, 7)) // 9
	fmt.Println(EditDistance("kitten", "sitting"))                   // 3
	fmt.Println(LongestIncreasingSubsequence([]int{10, 9, 2, 5, 3, 7, 101, 18})) // 4
	fmt.Println(LongestCommonSubsequence("ABCBDAB", "BDCABA"))       // 4

	fmt.Println("8-queens solutions:", len(SolveNQueens(8))) // 92
	fmt.Println(CombinationSum([]int{2, 3, 6, 7}, 7))        // [[2 2 3] [7]]
	fmt.Println(ThreeSum([]int{-1, 0, 1, 2, -1, -4}))

	ps := NewPrefixSums([]int{1, 2, 3, 4, 5})
	fmt.Println(ps.RangeSum(1, 4)) // 9
}
```

### 2. TypeScript

```typescript
// --- Divide and conquer ---

/** The answer is left-only, right-only, or crossing the midpoint. */
function maxSubarrayDC(nums: readonly number[]): number {
  if (nums.length === 1) return nums[0];
  const mid = nums.length >> 1;
  const left = maxSubarrayDC(nums.slice(0, mid));
  const right = maxSubarrayDC(nums.slice(mid));

  let leftBest = -Infinity;
  let sum = 0;
  for (let i = mid - 1; i >= 0; i--) {
    sum += nums[i];
    leftBest = Math.max(leftBest, sum);
  }
  let rightBest = -Infinity;
  sum = 0;
  for (let i = mid; i < nums.length; i++) {
    sum += nums[i];
    rightBest = Math.max(rightBest, sum);
  }
  return Math.max(left, right, leftBest + rightBest);
}

// --- Greedy ---

interface Interval { start: number; end: number }

/** Sort by end time and take greedily. Provably optimal. */
function maxNonOverlapping(intervals: readonly Interval[]): Interval[] {
  const sorted = [...intervals].sort((a, b) => a.end - b.end);
  const chosen: Interval[] = [];
  let lastEnd = -Infinity;
  for (const iv of sorted) {
    if (iv.start >= lastEnd) {
      chosen.push(iv);
      lastEnd = iv.end;
    }
  }
  return chosen;
}

/** Fast, and wrong for coin systems like {1,3,4}. */
function coinChangeGreedy(coins: readonly number[], amount: number): number {
  let remaining = amount;
  let count = 0;
  for (const c of [...coins].sort((a, b) => b - a)) {
    count += Math.floor(remaining / c);
    remaining %= c;
  }
  return remaining > 0 ? -1 : count;
}

// --- Dynamic programming ---

/** Always correct. dp[a] = fewest coins summing to exactly a. */
function coinChangeDP(coins: readonly number[], amount: number): number {
  const dp = new Array<number>(amount + 1).fill(Infinity);
  dp[0] = 0;
  for (let a = 1; a <= amount; a++) {
    for (const c of coins) {
      if (c <= a) dp[a] = Math.min(dp[a], dp[a - c] + 1);
    }
  }
  return dp[amount] === Infinity ? -1 : dp[amount];
}

/** 0/1 knapsack with a rolling array: O(n·W) time, O(W) space. */
function knapsack01(
  weights: readonly number[],
  values: readonly number[],
  capacity: number,
): number {
  const dp = new Array<number>(capacity + 1).fill(0);
  for (let i = 0; i < weights.length; i++) {
    // Descending is what prevents reusing item i more than once.
    for (let w = capacity; w >= weights[i]; w--) {
      dp[w] = Math.max(dp[w], dp[w - weights[i]] + values[i]);
    }
  }
  return dp[capacity];
}

/** Levenshtein distance, two rows instead of a full matrix. */
function editDistance(a: string, b: string): number {
  let prev = Array.from({ length: b.length + 1 }, (_, j) => j);
  let curr = new Array<number>(b.length + 1).fill(0);

  for (let i = 1; i <= a.length; i++) {
    curr[0] = i;
    for (let j = 1; j <= b.length; j++) {
      curr[j] = a[i - 1] === b[j - 1]
        ? prev[j - 1]                                        // free match
        : 1 + Math.min(prev[j - 1], prev[j], curr[j - 1]);   // sub, del, ins
    }
    [prev, curr] = [curr, prev];
  }
  return prev[b.length];
}

/** O(n log n) LIS. tails[k] = smallest tail of a length-(k+1) subsequence. */
function longestIncreasingSubsequence(nums: readonly number[]): number {
  const tails: number[] = [];
  for (const n of nums) {
    let lo = 0;
    let hi = tails.length;
    while (lo < hi) {
      const mid = (lo + hi) >> 1;
      if (tails[mid] < n) lo = mid + 1;
      else hi = mid;
    }
    tails[lo] = n; // extends the array when lo === tails.length
  }
  return tails.length;
}

/** LCS: the engine behind diff. */
function longestCommonSubsequence(a: string, b: string): number {
  let prev = new Array<number>(b.length + 1).fill(0);
  let curr = new Array<number>(b.length + 1).fill(0);
  for (let i = 1; i <= a.length; i++) {
    for (let j = 1; j <= b.length; j++) {
      curr[j] = a[i - 1] === b[j - 1]
        ? prev[j - 1] + 1
        : Math.max(prev[j], curr[j - 1]);
    }
    [prev, curr] = [curr, prev.fill(0)];
  }
  return prev[b.length];
}

/** Top-down DP. Memoization is just recursion plus a cache. */
function uniquePaths(rows: number, cols: number): number {
  const memo = new Map<string, number>();
  const walk = (r: number, c: number): number => {
    if (r === rows - 1 || c === cols - 1) return 1;
    const key = `${r},${c}`;
    const cached = memo.get(key);
    if (cached !== undefined) return cached;
    const total = walk(r + 1, c) + walk(r, c + 1);
    memo.set(key, total);
    return total;
  };
  return walk(0, 0);
}

// --- Backtracking ---

/** N-queens. Pruning by column and both diagonals is what makes it fast. */
function solveNQueens(n: number): number[][] {
  const solutions: number[][] = [];
  const queens: number[] = [];
  const cols = new Set<number>();
  const diag1 = new Set<number>(); // row + col
  const diag2 = new Set<number>(); // row - col

  const place = (row: number): void => {
    if (row === n) {
      solutions.push([...queens]);
      return;
    }
    for (let col = 0; col < n; col++) {
      if (cols.has(col) || diag1.has(row + col) || diag2.has(row - col)) continue;
      queens.push(col);                                        // choose
      cols.add(col); diag1.add(row + col); diag2.add(row - col);
      place(row + 1);                                          // explore
      queens.pop();                                            // undo
      cols.delete(col); diag1.delete(row + col); diag2.delete(row - col);
    }
  };

  place(0);
  return solutions;
}

/** Every multiset of candidates summing to target. Sorted so we can break. */
function combinationSum(candidates: readonly number[], target: number): number[][] {
  const sorted = [...candidates].sort((a, b) => a - b);
  const out: number[][] = [];
  const current: number[] = [];

  const backtrack = (start: number, remaining: number): void => {
    if (remaining === 0) {
      out.push([...current]);
      return;
    }
    for (let i = start; i < sorted.length; i++) {
      if (sorted[i] > remaining) break; // sorted: everything after is worse
      current.push(sorted[i]);
      backtrack(i, remaining - sorted[i]); // i, not i+1: reuse allowed
      current.pop();
    }
  };

  backtrack(0, target);
  return out;
}

/** Word break with memoization on the start index. */
function wordBreak(s: string, words: readonly string[]): boolean {
  const dict = new Set(words);
  const memo = new Map<number, boolean>();

  const canBreak = (start: number): boolean => {
    if (start === s.length) return true;
    const cached = memo.get(start);
    if (cached !== undefined) return cached;
    for (let end = start + 1; end <= s.length; end++) {
      if (dict.has(s.slice(start, end)) && canBreak(end)) {
        memo.set(start, true);
        return true;
      }
    }
    memo.set(start, false);
    return false;
  };

  return canBreak(0);
}

// --- Two pointers, sliding window, prefix sums ---

/** Sort plus two pointers. O(n^2) instead of O(n^3). */
function threeSum(nums: readonly number[]): number[][] {
  const sorted = [...nums].sort((a, b) => a - b);
  const out: number[][] = [];

  for (let i = 0; i < sorted.length - 2; i++) {
    if (i > 0 && sorted[i] === sorted[i - 1]) continue; // skip duplicate anchors
    let l = i + 1;
    let r = sorted.length - 1;
    while (l < r) {
      const sum = sorted[i] + sorted[l] + sorted[r];
      if (sum < 0) l++;
      else if (sum > 0) r--;
      else {
        out.push([sorted[i], sorted[l], sorted[r]]);
        while (l < r && sorted[l] === sorted[l + 1]) l++;
        while (l < r && sorted[r] === sorted[r - 1]) r--;
        l++;
        r--;
      }
    }
  }
  return out;
}

/** Variable-size sliding window. O(n) with one pass. */
function longestUniqueSubstring(s: string): number {
  const lastSeen = new Map<string, number>();
  let best = 0;
  let start = 0;
  for (let i = 0; i < s.length; i++) {
    const prev = lastSeen.get(s[i]);
    if (prev !== undefined && prev >= start) start = prev + 1; // shrink
    lastSeen.set(s[i], i);
    best = Math.max(best, i - start + 1);
  }
  return best;
}

/** O(n) setup buys O(1) range sums. */
class PrefixSums {
  readonly #cumulative: number[];

  constructor(nums: readonly number[]) {
    this.#cumulative = [0];
    for (const n of nums) this.#cumulative.push(this.#cumulative.at(-1)! + n);
  }

  /** Sum over [lo, hi). O(1). */
  rangeSum(lo: number, hi: number): number {
    return this.#cumulative[hi] - this.#cumulative[lo];
  }
}

console.log(maxSubarrayDC([-2, 1, -3, 4, -1, 2, 1, -5, 4])); // 6

console.log(maxNonOverlapping([
  { start: 1, end: 4 }, { start: 2, end: 3 },
  { start: 3, end: 5 }, { start: 5, end: 7 }, { start: 6, end: 8 },
]));

console.log(coinChangeGreedy([1, 3, 4], 6)); // 3, wrong
console.log(coinChangeDP([1, 3, 4], 6));     // 2, correct

console.log(knapsack01([1, 3, 4, 5], [1, 4, 5, 7], 7));                  // 9
console.log(editDistance("kitten", "sitting"));                          // 3
console.log(longestIncreasingSubsequence([10, 9, 2, 5, 3, 7, 101, 18])); // 4
console.log(longestCommonSubsequence("ABCBDAB", "BDCABA"));              // 4
console.log(uniquePaths(3, 7));                                          // 28

console.log(solveNQueens(8).length);              // 92
console.log(combinationSum([2, 3, 6, 7], 7));     // [[2,2,3],[7]]
console.log(wordBreak("applepenapple", ["apple", "pen"])); // true

console.log(threeSum([-1, 0, 1, 2, -1, -4]));
console.log(longestUniqueSubstring("abcabcbb")); // 3
console.log(new PrefixSums([1, 2, 3, 4, 5]).rangeSum(1, 4)); // 9
```

## Practical Applications & Real-World Use Cases

- **Diff and version control.** `git diff` computes a longest common subsequence, which is DP. Myers' diff algorithm is an optimized variant. Every merge conflict you have resolved traces back to this.
- **Spell checkers and fuzzy matching.** Edit distance is DP. Search engines use it to decide "did you mean".
- **Bioinformatics.** Needleman-Wunsch and Smith-Waterman align DNA and protein sequences. Both are edit distance with a scoring matrix. This is DP doing real scientific work at scale.
- **Resource allocation and scheduling.** Knapsack models budget allocation, ad slot selection, and cargo loading. Interval scheduling assigns meeting rooms and CPU time.
- **Compiler optimization.** Optimal register allocation, instruction scheduling, and common subexpression elimination all use DP or greedy heuristics.
- **Huffman coding.** Greedy, and provably optimal for prefix-free codes. It sits inside gzip, JPEG, and MP3.
- **Constraint solvers and configuration.** Sudoku, timetabling, and dependency version resolution are backtracking with propagation. SAT solvers are backtracking plus clause learning.
- **Text justification and layout.** TeX's paragraph breaking is DP that minimizes total badness. Greedy line breaking (what most browsers do) is faster and produces worse results, which is a nice concrete illustration of the trade.
- **Autocomplete ranking and financial modeling.** Both use DP over sequences: one for likely next tokens, one for optimal stopping.

## Related Concepts & Connections

- [[00-DSA-Index|Back to DSA Index]]
- [[13-Recursion]] for the base that memoization and backtracking build on
- [[12-Sorting-Algorithms]] for divide and conquer in practice
- [[11-Searching-Algorithms]] for binary search on the answer
- [[06-Priority-Queue-and-Heap]] for the heap greedy algorithms depend on
- [[15-Shortest-Path-Dijkstras-Algorithm]] for greedy (Dijkstra) and DP (Floyd-Warshall) side by side
- [[01-Fundamentals-and-Introduction]] for pricing a strategy before you commit to it
