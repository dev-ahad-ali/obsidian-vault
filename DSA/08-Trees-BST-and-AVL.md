---
tags:
  - dsa
  - computer-science
  - go
  - typescript
  - trees
type: study-note
status: complete
date_created: 2026-09-28
---

# Trees, BST and AVL

## Overview

A tree is a hierarchy: one root, every other node with exactly one parent, no cycles. A binary tree limits each node to two children. A binary search tree adds an ordering rule, and that rule is everything: for any node, all keys in the left subtree are smaller and all keys in the right subtree are larger.

That single invariant makes search a series of binary decisions, so a lookup costs O(height). The problem is that height depends on insertion order. Insert 1, 2, 3, 4, 5 into a plain BST and you get a linked list with extra steps, and every operation degrades to O(n). AVL trees fix this by rebalancing with rotations after every mutation, guaranteeing O(log n) forever.

## Key Concepts & Analogies

**A family tree, or an org chart.** Each person has one boss and some reports. No loops, because nobody is their own grandparent.

**A BST is the guess-the-number game.** "Higher or lower?" Every answer eliminates half the remaining range. This is binary search made into a structure instead of an algorithm. See [[11-Searching-Algorithms]].

**Vocabulary that keeps coming up:**
- Height of a node: longest path down to a leaf. A leaf has height 0.
- Depth of a node: distance up to the root. The root has depth 0.
- A complete tree fills every level except possibly the last, which fills left to right. This is what makes the array encoding in [[06-Priority-Queue-and-Heap]] work.
- A balanced tree keeps height at O(log n). Different definitions of "balanced" give different tree types.

**Four traversals, each with a purpose:**
- In-order (left, node, right) visits a BST in sorted order. This is the traversal you use to verify a BST is valid.
- Pre-order (node, left, right) serializes a tree so it can be rebuilt. Good for copying.
- Post-order (left, right, node) processes children before parents. Good for freeing memory and evaluating expression trees.
- Level-order uses a queue and visits by depth. This is BFS on a tree. See [[14-Graph-Traversal-DFS-BFS]].

**Deletion has three cases and the third one is the interesting one.** A leaf just vanishes. A node with one child is replaced by that child. A node with two children is replaced by its in-order successor (the leftmost node of the right subtree), which is then deleted from where it was. The successor is the smallest key still larger than the one you removed, so the ordering invariant survives.

**AVL balance factor** is `height(left) - height(right)`, and it must stay in {-1, 0, 1}. When an insert or delete pushes it to ±2, one of four rotations restores it: left-left needs a right rotation, right-right needs a left rotation, left-right and right-left need a double rotation. The rotations are fiddly to derive and easy to memorize as pictures.

**AVL versus red-black.** AVL is more strictly balanced, so lookups are slightly faster. Red-black tolerates more imbalance, so it does fewer rotations on write. Read-heavy workloads prefer AVL; write-heavy prefer red-black, which is why most standard libraries ship red-black.

## Operations & Time/Space Complexity

| Operation | Average Case | Worst Case (plain BST) | Worst Case (AVL) | Space Complexity |
| :--- | :--- | :--- | :--- | :--- |
| Search / Access | O(log n) | O(n) | O(log n) | O(1) iterative |
| Insertion | O(log n) | O(n) | O(log n) | O(1) iterative |
| Deletion | O(log n) | O(n) | O(log n) | O(1) iterative |
| Min / Max | O(log n) | O(n) | O(log n) | O(1) |
| In-order traversal | O(n) | O(n) | O(n) | O(h) stack |
| Range query | O(log n + k) | O(n) | O(log n + k) | O(h) |

Total storage is O(n). Recursive traversal uses O(h) call-stack space, which is O(log n) when balanced and O(n) when degenerate. k is the number of results in a range query.

## Code Implementations

### 1. Go (Golang)

```go
package main

import (
	"cmp"
	"fmt"
)

type Node[T cmp.Ordered] struct {
	Key         T
	Left, Right *Node[T]
	height      int // AVL bookkeeping; a plain BST would not need this
}

// AVLTree stays balanced after every mutation, so O(log n) is a
// guarantee rather than a hope about insertion order.
type AVLTree[T cmp.Ordered] struct {
	root *Node[T]
	size int
}

func (t *AVLTree[T]) Len() int { return t.size }

func height[T cmp.Ordered](n *Node[T]) int {
	if n == nil {
		return -1 // so a leaf comes out at height 0
	}
	return n.height
}

func balanceFactor[T cmp.Ordered](n *Node[T]) int {
	if n == nil {
		return 0
	}
	return height(n.Left) - height(n.Right)
}

func updateHeight[T cmp.Ordered](n *Node[T]) {
	n.height = 1 + max(height(n.Left), height(n.Right))
}

// rotateRight fixes a left-heavy node.
//
//	    y            x
//	   / \          / \
//	  x   C   ->   A   y
//	 / \              / \
//	A   B            B   C
func rotateRight[T cmp.Ordered](y *Node[T]) *Node[T] {
	x := y.Left
	y.Left = x.Right
	x.Right = y
	updateHeight(y)
	updateHeight(x)
	return x
}

// rotateLeft is the mirror image, for a right-heavy node.
func rotateLeft[T cmp.Ordered](x *Node[T]) *Node[T] {
	y := x.Right
	x.Right = y.Left
	y.Left = x
	updateHeight(x)
	updateHeight(y)
	return y
}

// rebalance applies whichever of the four cases is needed.
func rebalance[T cmp.Ordered](n *Node[T]) *Node[T] {
	updateHeight(n)
	switch bf := balanceFactor(n); {
	case bf > 1:
		if balanceFactor(n.Left) < 0 {
			n.Left = rotateLeft(n.Left) // left-right case
		}
		return rotateRight(n)
	case bf < -1:
		if balanceFactor(n.Right) > 0 {
			n.Right = rotateRight(n.Right) // right-left case
		}
		return rotateLeft(n)
	}
	return n
}

func (t *AVLTree[T]) Insert(key T) {
	var inserted bool
	t.root, inserted = insert(t.root, key)
	if inserted {
		t.size++
	}
}

func insert[T cmp.Ordered](n *Node[T], key T) (*Node[T], bool) {
	if n == nil {
		return &Node[T]{Key: key}, true
	}
	var ok bool
	switch {
	case key < n.Key:
		n.Left, ok = insert(n.Left, key)
	case key > n.Key:
		n.Right, ok = insert(n.Right, key)
	default:
		return n, false // duplicates ignored
	}
	return rebalance(n), ok
}

// Search walks down making one comparison per level. O(log n).
func (t *AVLTree[T]) Search(key T) bool {
	n := t.root
	for n != nil {
		switch {
		case key < n.Key:
			n = n.Left
		case key > n.Key:
			n = n.Right
		default:
			return true
		}
	}
	return false
}

func (t *AVLTree[T]) Delete(key T) {
	var removed bool
	t.root, removed = remove(t.root, key)
	if removed {
		t.size--
	}
}

func remove[T cmp.Ordered](n *Node[T], key T) (*Node[T], bool) {
	if n == nil {
		return nil, false
	}
	var ok bool
	switch {
	case key < n.Key:
		n.Left, ok = remove(n.Left, key)
	case key > n.Key:
		n.Right, ok = remove(n.Right, key)
	default:
		ok = true
		// Cases one and two: zero or one child, splice it out.
		if n.Left == nil {
			return n.Right, true
		}
		if n.Right == nil {
			return n.Left, true
		}
		// Case three: replace with the in-order successor, then delete
		// the successor from the right subtree.
		succ := n.Right
		for succ.Left != nil {
			succ = succ.Left
		}
		n.Key = succ.Key
		n.Right, _ = remove(n.Right, succ.Key)
	}
	if !ok {
		return n, false
	}
	return rebalance(n), true
}

func (t *AVLTree[T]) Min() (T, bool) {
	var zero T
	if t.root == nil {
		return zero, false
	}
	n := t.root
	for n.Left != nil {
		n = n.Left
	}
	return n.Key, true
}

// InOrder yields sorted order. This is the defining property of a BST.
func (t *AVLTree[T]) InOrder() []T {
	out := make([]T, 0, t.size)
	var walk func(*Node[T])
	walk = func(n *Node[T]) {
		if n == nil {
			return
		}
		walk(n.Left)
		out = append(out, n.Key)
		walk(n.Right)
	}
	walk(t.root)
	return out
}

// PreOrder serializes shape and values, so it can rebuild the tree.
func (t *AVLTree[T]) PreOrder() []T {
	out := make([]T, 0, t.size)
	var walk func(*Node[T])
	walk = func(n *Node[T]) {
		if n == nil {
			return
		}
		out = append(out, n.Key)
		walk(n.Left)
		walk(n.Right)
	}
	walk(t.root)
	return out
}

// LevelOrder is BFS on a tree: a queue, one level at a time.
func (t *AVLTree[T]) LevelOrder() [][]T {
	if t.root == nil {
		return nil
	}
	var out [][]T
	queue := []*Node[T]{t.root}
	for len(queue) > 0 {
		level := make([]T, 0, len(queue))
		var next []*Node[T]
		for _, n := range queue {
			level = append(level, n.Key)
			if n.Left != nil {
				next = append(next, n.Left)
			}
			if n.Right != nil {
				next = append(next, n.Right)
			}
		}
		out = append(out, level)
		queue = next
	}
	return out
}

// RangeQuery collects keys in [lo,hi]. Pruning subtrees that cannot
// contain matches is what makes this O(log n + k).
func (t *AVLTree[T]) RangeQuery(lo, hi T) []T {
	var out []T
	var walk func(*Node[T])
	walk = func(n *Node[T]) {
		if n == nil {
			return
		}
		if n.Key > lo {
			walk(n.Left)
		}
		if n.Key >= lo && n.Key <= hi {
			out = append(out, n.Key)
		}
		if n.Key < hi {
			walk(n.Right)
		}
	}
	walk(t.root)
	return out
}

func (t *AVLTree[T]) Height() int { return height(t.root) }

func main() {
	t := &AVLTree[int]{}
	// Sorted input, the worst case for an unbalanced BST.
	for i := 1; i <= 15; i++ {
		t.Insert(i)
	}
	fmt.Println("size:", t.Len(), "height:", t.Height())
	// height 3, not 14: the rotations did their job

	fmt.Println(t.InOrder())        // 1..15 sorted
	fmt.Println(t.LevelOrder()[0])  // [8]
	fmt.Println(t.RangeQuery(5, 9)) // [5 6 7 8 9]
	fmt.Println(t.Search(7), t.Search(99))
	t.Delete(8) // two-child case: successor replaces it
	fmt.Println(t.InOrder())
}
```

### 2. TypeScript

```typescript
interface TreeNode<T> {
  key: T;
  left: TreeNode<T> | null;
  right: TreeNode<T> | null;
  height: number;
}

type Compare<T> = (a: T, b: T) => number;

/** Self-balancing BST. Height stays O(log n) regardless of insert order. */
class AVLTree<T> {
  #root: TreeNode<T> | null = null;
  #size = 0;
  readonly #compare: Compare<T>;

  constructor(compare: Compare<T> = (a, b) => (a < b ? -1 : a > b ? 1 : 0)) {
    this.#compare = compare;
  }

  get size(): number { return this.#size; }
  get height(): number { return AVLTree.#height(this.#root); }

  static #height<T>(n: TreeNode<T> | null): number {
    return n === null ? -1 : n.height;
  }

  static #balanceFactor<T>(n: TreeNode<T> | null): number {
    return n === null ? 0 : AVLTree.#height(n.left) - AVLTree.#height(n.right);
  }

  static #update<T>(n: TreeNode<T>): void {
    n.height = 1 + Math.max(AVLTree.#height(n.left), AVLTree.#height(n.right));
  }

  /** Fixes a left-heavy node by promoting its left child. */
  static #rotateRight<T>(y: TreeNode<T>): TreeNode<T> {
    const x = y.left!;
    y.left = x.right;
    x.right = y;
    AVLTree.#update(y);
    AVLTree.#update(x);
    return x;
  }

  static #rotateLeft<T>(x: TreeNode<T>): TreeNode<T> {
    const y = x.right!;
    x.right = y.left;
    y.left = x;
    AVLTree.#update(x);
    AVLTree.#update(y);
    return y;
  }

  static #rebalance<T>(n: TreeNode<T>): TreeNode<T> {
    AVLTree.#update(n);
    const bf = AVLTree.#balanceFactor(n);
    if (bf > 1) {
      if (AVLTree.#balanceFactor(n.left) < 0) n.left = AVLTree.#rotateLeft(n.left!);
      return AVLTree.#rotateRight(n);
    }
    if (bf < -1) {
      if (AVLTree.#balanceFactor(n.right) > 0) n.right = AVLTree.#rotateRight(n.right!);
      return AVLTree.#rotateLeft(n);
    }
    return n;
  }

  insert(key: T): this {
    const before = this.#size;
    this.#root = this.#insertAt(this.#root, key);
    if (this.#size === before) return this; // duplicate
    return this;
  }

  #insertAt(n: TreeNode<T> | null, key: T): TreeNode<T> {
    if (n === null) {
      this.#size++;
      return { key, left: null, right: null, height: 0 };
    }
    const c = this.#compare(key, n.key);
    if (c < 0) n.left = this.#insertAt(n.left, key);
    else if (c > 0) n.right = this.#insertAt(n.right, key);
    else return n; // duplicates ignored
    return AVLTree.#rebalance(n);
  }

  /** Iterative search. O(log n), no call-stack cost. */
  has(key: T): boolean {
    let n = this.#root;
    while (n !== null) {
      const c = this.#compare(key, n.key);
      if (c === 0) return true;
      n = c < 0 ? n.left : n.right;
    }
    return false;
  }

  delete(key: T): boolean {
    const before = this.#size;
    this.#root = this.#deleteAt(this.#root, key);
    return this.#size < before;
  }

  #deleteAt(n: TreeNode<T> | null, key: T): TreeNode<T> | null {
    if (n === null) return null;
    const c = this.#compare(key, n.key);
    if (c < 0) {
      n.left = this.#deleteAt(n.left, key);
    } else if (c > 0) {
      n.right = this.#deleteAt(n.right, key);
    } else {
      this.#size--;
      if (n.left === null) return n.right;   // zero or one child
      if (n.right === null) return n.left;
      // Two children: promote the in-order successor.
      let succ = n.right;
      while (succ.left !== null) succ = succ.left;
      n.key = succ.key;
      this.#size++; // the recursive call decrements again for the successor
      n.right = this.#deleteAt(n.right, succ.key);
    }
    return AVLTree.#rebalance(n);
  }

  min(): T | undefined {
    if (this.#root === null) return undefined;
    let n = this.#root;
    while (n.left !== null) n = n.left;
    return n.key;
  }

  max(): T | undefined {
    if (this.#root === null) return undefined;
    let n = this.#root;
    while (n.right !== null) n = n.right;
    return n.key;
  }

  /** Sorted order. A generator so callers can stop early. */
  *inOrder(node: TreeNode<T> | null = this.#root): Generator<T> {
    if (node === null) return;
    yield* this.inOrder(node.left);
    yield node.key;
    yield* this.inOrder(node.right);
  }

  *preOrder(node: TreeNode<T> | null = this.#root): Generator<T> {
    if (node === null) return;
    yield node.key;
    yield* this.preOrder(node.left);
    yield* this.preOrder(node.right);
  }

  *postOrder(node: TreeNode<T> | null = this.#root): Generator<T> {
    if (node === null) return;
    yield* this.postOrder(node.left);
    yield* this.postOrder(node.right);
    yield node.key;
  }

  /** BFS by level, using a queue. */
  levelOrder(): T[][] {
    if (this.#root === null) return [];
    const out: T[][] = [];
    let queue: TreeNode<T>[] = [this.#root];
    while (queue.length > 0) {
      out.push(queue.map((n) => n.key));
      const next: TreeNode<T>[] = [];
      for (const n of queue) {
        if (n.left) next.push(n.left);
        if (n.right) next.push(n.right);
      }
      queue = next;
    }
    return out;
  }

  /** Keys in [lo, hi]. Prunes subtrees that cannot match. */
  range(lo: T, hi: T): T[] {
    const out: T[] = [];
    const walk = (n: TreeNode<T> | null): void => {
      if (n === null) return;
      if (this.#compare(n.key, lo) > 0) walk(n.left);
      if (this.#compare(n.key, lo) >= 0 && this.#compare(n.key, hi) <= 0) out.push(n.key);
      if (this.#compare(n.key, hi) < 0) walk(n.right);
    };
    walk(this.#root);
    return out;
  }
}

const tree = new AVLTree<number>((a, b) => a - b);
for (let i = 1; i <= 15; i++) tree.insert(i); // sorted input, worst case for a plain BST
console.log("size", tree.size, "height", tree.height); // height 3, not 14
console.log([...tree.inOrder()]);
console.log(tree.levelOrder()[0]);  // [8]
console.log(tree.range(5, 9));      // [5, 6, 7, 8, 9]
console.log(tree.has(7), tree.has(99));
tree.delete(8);
console.log([...tree.inOrder()]);
```

## Practical Applications & Real-World Use Cases

- **Database indexes.** B-trees and B+ trees are the generalization: many keys per node so one disk page holds a whole node. Every relational database index you have ever created is one. The BST logic here is the concept, scaled for disk.
- **Filesystems.** ext4 uses HTrees for directories, Btrfs is named after B-trees, and NTFS indexes directories the same way.
- **Ordered maps in standard libraries.** C++ `std::map`, Java `TreeMap`, and Rust `BTreeMap` all keep keys sorted so you can iterate in order and do range queries. Hash maps cannot do either. See [[07-Hash-Tables-and-Sets]].
- **Interval and range queries.** Finding all calendar events overlapping a window, or all IP ranges containing an address, uses augmented trees.
- **Expression trees in compilers.** Post-order evaluation of a parse tree computes the result bottom-up.
- **Decision trees in ML.** Every split is a comparison node; prediction is a root-to-leaf walk.
- **Spatial indexes.** Quadtrees, k-d trees, and R-trees extend the idea to two or more dimensions for map tiles and collision detection.

## Related Concepts & Connections

- [[00-DSA-Index|Back to DSA Index]]
- [[06-Priority-Queue-and-Heap]] for a different tree with a weaker invariant
- [[07-Hash-Tables-and-Sets]] for O(1) lookup without ordering
- [[09-Trie]] for a tree keyed by string prefixes
- [[10-Graphs]] because a tree is a connected acyclic graph
- [[11-Searching-Algorithms]] for binary search, the algorithm a BST encodes
- [[13-Recursion]] because every traversal here is recursive
