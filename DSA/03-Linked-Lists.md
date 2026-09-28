---
tags:
  - dsa
  - computer-science
  - go
  - typescript
  - linked-lists
type: study-note
status: complete
date_created: 2026-09-28
---

# Linked Lists

## Overview

A linked list stores elements in separately allocated nodes, each holding a value and a pointer to the next node. The list itself is just a reference to the first node. Nothing is contiguous, so there is no index arithmetic and no random access. To reach the tenth element you follow nine pointers.

What you buy with that is O(1) insertion and deletion once you hold a reference to the right node, with no shifting and no reallocation. That is the entire trade against [[02-Arrays]].

## Key Concepts & Analogies

**A treasure hunt.** Each clue tells you where the next clue is. You cannot skip to clue seven; you have to walk the chain. Losing one clue loses everything after it, which is exactly what happens when you drop a pointer mid-splice.

**Three shapes:**
- Singly linked: each node points forward only. Smallest memory footprint. Deleting a node requires its predecessor.
- Doubly linked: each node points forward and backward. Deleting a known node is O(1) with no search. Costs one extra pointer per node.
- Circular: the tail points back to the head. Useful for round-robin rotation with no end-of-list special case.

**The sentinel (dummy head) trick.** Insertion and deletion at the head are special cases that clutter every method. Allocate one dummy node in front of the real first element and the special cases vanish, because every real node now has a predecessor. This is the single highest-value habit in linked-list code.

**Fast and slow pointers.** Advance one pointer by 1 and another by 2. When the fast one hits the end, the slow one is at the midpoint. If the two ever meet, there is a cycle. This is Floyd's cycle detection, and it uses O(1) space.

**Reversal is a three-pointer dance.** Keep `prev`, `curr`, `next`. Save next, point curr backward, slide both forward. Write it out on paper once and it stops being scary.

**Cache behavior is the hidden cost.** Nodes scattered across the heap mean a cache miss per hop. An O(n) array scan usually beats an O(n) list walk by a wide margin in wall clock. Linked lists win when you are splicing constantly and never scanning.

## Operations & Time/Space Complexity

| Operation | Average Case | Worst Case | Space Complexity |
| :--- | :--- | :--- | :--- |
| Access by index | O(n) | O(n) | O(1) |
| Search by value | O(n) | O(n) | O(1) |
| Insert at head | O(1) | O(1) | O(1) |
| Insert at tail (tail pointer) | O(1) | O(1) | O(1) |
| Insert at index | O(n) | O(n) | O(1) |
| Delete at head | O(1) | O(1) | O(1) |
| Delete known node (doubly) | O(1) | O(1) | O(1) |
| Delete known node (singly) | O(n) | O(n) | O(1) |
| Reverse | O(n) | O(n) | O(1) |

Total storage is O(n) plus one pointer per node (two for doubly linked).

## Code Implementations

### 1. Go (Golang)

```go
package main

import (
	"fmt"
	"strings"
)

type Node[T any] struct {
	Value T
	Next  *Node[T]
}

// SinglyLinkedList keeps a tail pointer so PushBack stays O(1).
type SinglyLinkedList[T comparable] struct {
	head *Node[T]
	tail *Node[T]
	size int
}

func (l *SinglyLinkedList[T]) Len() int { return l.size }

// PushFront is O(1): the whole point of a linked list.
func (l *SinglyLinkedList[T]) PushFront(v T) {
	n := &Node[T]{Value: v, Next: l.head}
	l.head = n
	if l.tail == nil {
		l.tail = n
	}
	l.size++
}

// PushBack is O(1) only because we track the tail.
func (l *SinglyLinkedList[T]) PushBack(v T) {
	n := &Node[T]{Value: v}
	if l.tail == nil {
		l.head, l.tail = n, n
	} else {
		l.tail.Next = n
		l.tail = n
	}
	l.size++
}

func (l *SinglyLinkedList[T]) PopFront() (T, bool) {
	var zero T
	if l.head == nil {
		return zero, false
	}
	n := l.head
	l.head = n.Next
	if l.head == nil {
		l.tail = nil
	}
	n.Next = nil // help the GC, avoid accidental retention
	l.size--
	return n.Value, true
}

// Remove deletes the first node holding v. O(n) because a singly linked
// node cannot reach its own predecessor.
func (l *SinglyLinkedList[T]) Remove(v T) bool {
	dummy := &Node[T]{Next: l.head} // sentinel kills the head special case
	prev := dummy
	for curr := l.head; curr != nil; curr = curr.Next {
		if curr.Value == v {
			prev.Next = curr.Next
			if curr == l.tail {
				l.tail = prev
			}
			l.head = dummy.Next
			if l.head == nil {
				l.tail = nil
			}
			l.size--
			return true
		}
		prev = curr
	}
	return false
}

// Reverse in place. Three pointers, one pass, O(1) space.
func (l *SinglyLinkedList[T]) Reverse() {
	var prev *Node[T]
	curr := l.head
	l.tail = l.head
	for curr != nil {
		next := curr.Next
		curr.Next = prev
		prev = curr
		curr = next
	}
	l.head = prev
}

// Middle uses fast/slow pointers. One pass, no length needed.
func (l *SinglyLinkedList[T]) Middle() (T, bool) {
	var zero T
	if l.head == nil {
		return zero, false
	}
	slow, fast := l.head, l.head
	for fast.Next != nil && fast.Next.Next != nil {
		slow = slow.Next
		fast = fast.Next.Next
	}
	return slow.Value, true
}

// HasCycle is Floyd's tortoise and hare. O(n) time, O(1) space.
func HasCycle[T any](head *Node[T]) bool {
	slow, fast := head, head
	for fast != nil && fast.Next != nil {
		slow = slow.Next
		fast = fast.Next.Next
		if slow == fast {
			return true
		}
	}
	return false
}

func (l *SinglyLinkedList[T]) String() string {
	var b strings.Builder
	for n := l.head; n != nil; n = n.Next {
		fmt.Fprintf(&b, "%v", n.Value)
		if n.Next != nil {
			b.WriteString(" -> ")
		}
	}
	return b.String()
}

// --- Doubly linked, where deletion of a known node is genuinely O(1) ---

type DNode[T any] struct {
	Value      T
	Prev, Next *DNode[T]
}

// DoublyLinkedList uses two sentinels so no method ever checks for nil.
type DoublyLinkedList[T any] struct {
	head, tail *DNode[T]
	size       int
}

func NewDoublyLinkedList[T any]() *DoublyLinkedList[T] {
	h, t := &DNode[T]{}, &DNode[T]{}
	h.Next, t.Prev = t, h
	return &DoublyLinkedList[T]{head: h, tail: t}
}

func (l *DoublyLinkedList[T]) Len() int { return l.size }

// PushBack inserts before the tail sentinel.
func (l *DoublyLinkedList[T]) PushBack(v T) *DNode[T] {
	n := &DNode[T]{Value: v, Prev: l.tail.Prev, Next: l.tail}
	l.tail.Prev.Next = n
	l.tail.Prev = n
	l.size++
	return n
}

func (l *DoublyLinkedList[T]) PushFront(v T) *DNode[T] {
	n := &DNode[T]{Value: v, Prev: l.head, Next: l.head.Next}
	l.head.Next.Prev = n
	l.head.Next = n
	l.size++
	return n
}

// Unlink is O(1). This is what makes an LRU cache possible.
func (l *DoublyLinkedList[T]) Unlink(n *DNode[T]) {
	if n == nil || n == l.head || n == l.tail {
		return
	}
	n.Prev.Next = n.Next
	n.Next.Prev = n.Prev
	n.Prev, n.Next = nil, nil
	l.size--
}

func (l *DoublyLinkedList[T]) Values() []T {
	out := make([]T, 0, l.size)
	for n := l.head.Next; n != l.tail; n = n.Next {
		out = append(out, n.Value)
	}
	return out
}

func main() {
	l := &SinglyLinkedList[int]{}
	for i := 1; i <= 5; i++ {
		l.PushBack(i)
	}
	fmt.Println(l)          // 1 -> 2 -> 3 -> 4 -> 5
	mid, _ := l.Middle()
	fmt.Println("middle:", mid) // 3
	l.Remove(3)
	l.Reverse()
	fmt.Println(l)          // 5 -> 4 -> 2 -> 1

	d := NewDoublyLinkedList[string]()
	d.PushBack("b")
	d.PushBack("c")
	first := d.PushFront("a")
	fmt.Println(d.Values()) // [a b c]
	d.Unlink(first)         // O(1), no search
	fmt.Println(d.Values()) // [b c]
}
```

### 2. TypeScript

```typescript
interface ListNode<T> {
  value: T;
  next: ListNode<T> | null;
}

class SinglyLinkedList<T> implements Iterable<T> {
  #head: ListNode<T> | null = null;
  #tail: ListNode<T> | null = null;
  #size = 0;

  get length(): number { return this.#size; }

  /** O(1). */
  pushFront(value: T): this {
    const node: ListNode<T> = { value, next: this.#head };
    this.#head = node;
    this.#tail ??= node;
    this.#size++;
    return this;
  }

  /** O(1) because of the tail pointer. */
  pushBack(value: T): this {
    const node: ListNode<T> = { value, next: null };
    if (this.#tail === null) {
      this.#head = this.#tail = node;
    } else {
      this.#tail.next = node;
      this.#tail = node;
    }
    this.#size++;
    return this;
  }

  popFront(): T | undefined {
    if (this.#head === null) return undefined;
    const node = this.#head;
    this.#head = node.next;
    if (this.#head === null) this.#tail = null;
    node.next = null;
    this.#size--;
    return node.value;
  }

  /** O(n). A sentinel removes the head special case entirely. */
  remove(value: T): boolean {
    const dummy: ListNode<T> = { value: undefined as T, next: this.#head };
    let prev = dummy;
    let curr = this.#head;
    while (curr !== null) {
      if (curr.value === value) {
        prev.next = curr.next;
        if (curr === this.#tail) this.#tail = prev === dummy ? null : prev;
        this.#head = dummy.next;
        this.#size--;
        return true;
      }
      prev = curr;
      curr = curr.next;
    }
    return false;
  }

  /** Three-pointer reversal. O(n) time, O(1) space. */
  reverse(): this {
    let prev: ListNode<T> | null = null;
    let curr = this.#head;
    this.#tail = this.#head;
    while (curr !== null) {
      const next: ListNode<T> | null = curr.next;
      curr.next = prev;
      prev = curr;
      curr = next;
    }
    this.#head = prev;
    return this;
  }

  /** Fast/slow pointers. On even length returns the second middle. */
  middle(): T | undefined {
    if (this.#head === null) return undefined;
    let slow = this.#head;
    let fast: ListNode<T> | null = this.#head;
    while (fast.next !== null && fast.next.next !== null) {
      slow = slow.next!;
      fast = fast.next.next;
    }
    return slow.value;
  }

  *[Symbol.iterator](): Iterator<T> {
    for (let n = this.#head; n !== null; n = n.next) yield n.value;
  }

  toString(): string { return [...this].join(" -> "); }
}

/** Floyd's cycle detection. O(n) time, O(1) space. */
function hasCycle<T>(head: ListNode<T> | null): boolean {
  let slow = head;
  let fast = head;
  while (fast !== null && fast.next !== null) {
    slow = slow!.next;
    fast = fast.next.next;
    if (slow === fast) return true;
  }
  return false;
}

// --- Doubly linked with sentinels ---

interface DNode<T> {
  value: T;
  prev: DNode<T> | null;
  next: DNode<T> | null;
}

class DoublyLinkedList<T> {
  readonly #head: DNode<T>;
  readonly #tail: DNode<T>;
  #size = 0;

  constructor() {
    this.#head = { value: undefined as T, prev: null, next: null };
    this.#tail = { value: undefined as T, prev: this.#head, next: null };
    this.#head.next = this.#tail;
  }

  get length(): number { return this.#size; }

  pushBack(value: T): DNode<T> {
    const node: DNode<T> = { value, prev: this.#tail.prev, next: this.#tail };
    this.#tail.prev!.next = node;
    this.#tail.prev = node;
    this.#size++;
    return node;
  }

  pushFront(value: T): DNode<T> {
    const node: DNode<T> = { value, prev: this.#head, next: this.#head.next };
    this.#head.next!.prev = node;
    this.#head.next = node;
    this.#size++;
    return node;
  }

  /** O(1) with no search. This is the LRU cache primitive. */
  unlink(node: DNode<T>): void {
    if (node === this.#head || node === this.#tail) return;
    node.prev!.next = node.next;
    node.next!.prev = node.prev;
    node.prev = node.next = null;
    this.#size--;
  }

  toArray(): T[] {
    const out: T[] = [];
    for (let n = this.#head.next; n !== this.#tail; n = n!.next) out.push(n!.value);
    return out;
  }
}

const list = new SinglyLinkedList<number>();
for (let i = 1; i <= 5; i++) list.pushBack(i);
console.log(list.toString());  // 1 -> 2 -> 3 -> 4 -> 5
console.log(list.middle());    // 3
list.remove(3);
console.log(list.reverse().toString()); // 5 -> 4 -> 2 -> 1

const dll = new DoublyLinkedList<string>();
dll.pushBack("b");
dll.pushBack("c");
const a = dll.pushFront("a");
console.log(dll.toArray()); // ["a", "b", "c"]
dll.unlink(a);
console.log(dll.toArray()); // ["b", "c"]
```

## Practical Applications & Real-World Use Cases

- **LRU caches.** A hash map points at doubly linked nodes. Lookup is O(1) via the map, and promoting an entry to most-recent is O(1) via unlink and push-front. This pairing is the standard answer to a very common interview question and it is also how real caches work.
- **Undo and redo stacks in editors.** A doubly linked list of edit operations lets you walk both directions cheaply.
- **Memory allocators.** Free lists thread the unused blocks together using pointers stored inside the free blocks themselves.
- **The Linux kernel.** `struct list_head` is an intrusive circular doubly linked list used for process lists, file descriptors, and nearly every other kernel collection.
- **Music playlists and round-robin schedulers.** Circular lists give you "next" forever with no wraparound check.
- **Blockchains.** Each block holds the hash of the previous one. That is a singly linked list where the pointer is a cryptographic hash.

## Related Concepts & Connections

- [[00-DSA-Index|Back to DSA Index]]
- [[02-Arrays]] for the contiguous-memory alternative and when it wins
- [[04-Stacks]] and [[05-Queues]], both of which a linked list implements naturally
- [[07-Hash-Tables-and-Sets]] for separate chaining, which uses linked lists per bucket
- [[13-Recursion]] because most list operations have clean recursive forms
