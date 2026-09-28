---
tags:
  - dsa
  - computer-science
  - go
  - typescript
  - stacks
type: study-note
status: complete
date_created: 2026-09-28
---

# Stacks

## Overview

A stack is a collection with one rule: the last thing you put in is the first thing you take out. LIFO. You can only touch the top. That restriction sounds limiting and is exactly what makes stacks useful, because a huge class of problems is about matching a thing to the most recent unmatched thing before it.

The interface is three operations: push, pop, peek. All O(1). The backing store is usually an array, occasionally a linked list.

## Key Concepts & Analogies

**A stack of plates in a cafeteria.** You add to the top and take from the top. Reaching the bottom plate means removing everything above it first.

**The browser back button.** Every page you visit gets pushed. Back pops. That is the whole feature.

**The call stack is a stack.** Every function call pushes a frame holding local variables and the return address. Returning pops it. Infinite recursion pushes frames until the region is exhausted, which is what a stack overflow is. See [[13-Recursion]].

**Array-backed versus list-backed.** An array-backed stack is faster in practice because the top elements stay hot in cache and there is no per-node allocation. A list-backed stack never needs to resize and never copies. Go's slices make array-backing so easy that it is almost always the right call.

**The monotonic stack pattern.** Keep the stack sorted (increasing or decreasing) by popping anything that violates the order before pushing. This solves "next greater element", "largest rectangle in a histogram", and "daily temperatures" in O(n) instead of O(n²). Each element is pushed once and popped once, which is where the linear bound comes from. This pattern is worth learning properly; it looks like magic until you see the amortized argument.

**Any recursive algorithm can be rewritten with an explicit stack.** Sometimes you must, when the recursion depth would blow the call stack. Iterative DFS is the canonical example. See [[14-Graph-Traversal-DFS-BFS]].

## Operations & Time/Space Complexity

| Operation | Average Case | Worst Case | Space Complexity |
| :--- | :--- | :--- | :--- |
| Push | O(1) amortized | O(n) on resize | O(1) |
| Pop | O(1) | O(1) | O(1) |
| Peek / Top | O(1) | O(1) | O(1) |
| Search | O(n) | O(n) | O(1) |
| IsEmpty / Size | O(1) | O(1) | O(1) |

Total storage is O(n). A linked-list stack has no resize spike, so push is a true O(1) worst case.

## Code Implementations

### 1. Go (Golang)

```go
package main

import "fmt"

// Stack is array-backed. A Go slice already gives amortized growth,
// so this is a thin, cache-friendly wrapper.
type Stack[T any] struct {
	items []T
}

func NewStack[T any]() *Stack[T] { return &Stack[T]{} }

func (s *Stack[T]) Len() int      { return len(s.items) }
func (s *Stack[T]) IsEmpty() bool { return len(s.items) == 0 }

func (s *Stack[T]) Push(v T) {
	s.items = append(s.items, v)
}

func (s *Stack[T]) Pop() (T, bool) {
	var zero T
	if len(s.items) == 0 {
		return zero, false
	}
	last := len(s.items) - 1
	v := s.items[last]
	s.items[last] = zero // drop the reference so the GC can collect it
	s.items = s.items[:last]
	return v, true
}

func (s *Stack[T]) Peek() (T, bool) {
	var zero T
	if len(s.items) == 0 {
		return zero, false
	}
	return s.items[len(s.items)-1], true
}

// --- Classic stack problems ---

// IsBalanced checks bracket matching. Push openers, pop on closers,
// and the stack must be empty at the end.
func IsBalanced(s string) bool {
	pairs := map[rune]rune{')': '(', ']': '[', '}': '{'}
	st := NewStack[rune]()
	for _, ch := range s {
		switch ch {
		case '(', '[', '{':
			st.Push(ch)
		case ')', ']', '}':
			top, ok := st.Pop()
			if !ok || top != pairs[ch] {
				return false
			}
		}
	}
	return st.IsEmpty()
}

// EvalRPN evaluates reverse Polish notation. Operands go on the stack,
// operators pop two and push the result.
func EvalRPN(tokens []string) (int, error) {
	st := NewStack[int]()
	for _, tok := range tokens {
		switch tok {
		case "+", "-", "*", "/":
			b, ok1 := st.Pop()
			a, ok2 := st.Pop()
			if !ok1 || !ok2 {
				return 0, fmt.Errorf("not enough operands for %q", tok)
			}
			switch tok {
			case "+":
				st.Push(a + b)
			case "-":
				st.Push(a - b)
			case "*":
				st.Push(a * b)
			case "/":
				if b == 0 {
					return 0, fmt.Errorf("division by zero")
				}
				st.Push(a / b)
			}
		default:
			var n int
			if _, err := fmt.Sscanf(tok, "%d", &n); err != nil {
				return 0, fmt.Errorf("bad token %q", tok)
			}
			st.Push(n)
		}
	}
	result, ok := st.Pop()
	if !ok || !st.IsEmpty() {
		return 0, fmt.Errorf("malformed expression")
	}
	return result, nil
}

// NextGreater is a monotonic decreasing stack. O(n) total: each index
// is pushed once and popped at most once.
func NextGreater(nums []int) []int {
	out := make([]int, len(nums))
	for i := range out {
		out[i] = -1
	}
	st := NewStack[int]() // holds indices whose answer is still unknown
	for i, n := range nums {
		for {
			j, ok := st.Peek()
			if !ok || nums[j] >= n {
				break
			}
			st.Pop()
			out[j] = n
		}
		st.Push(i)
	}
	return out
}

// MinStack returns the minimum in O(1) by storing a running min
// alongside each value. Costs O(n) extra space, buys O(1) Min().
type MinStack struct {
	entries []struct{ val, min int }
}

func (m *MinStack) Push(v int) {
	min := v
	if n := len(m.entries); n > 0 && m.entries[n-1].min < v {
		min = m.entries[n-1].min
	}
	m.entries = append(m.entries, struct{ val, min int }{v, min})
}

func (m *MinStack) Pop() (int, bool) {
	if len(m.entries) == 0 {
		return 0, false
	}
	e := m.entries[len(m.entries)-1]
	m.entries = m.entries[:len(m.entries)-1]
	return e.val, true
}

func (m *MinStack) Min() (int, bool) {
	if len(m.entries) == 0 {
		return 0, false
	}
	return m.entries[len(m.entries)-1].min, true
}

func main() {
	fmt.Println(IsBalanced("{[()]}"))  // true
	fmt.Println(IsBalanced("{[(])}"))  // false

	v, _ := EvalRPN([]string{"2", "3", "4", "*", "+"})
	fmt.Println(v) // 14

	fmt.Println(NextGreater([]int{2, 1, 2, 4, 3})) // [4 2 4 -1 -1]

	m := &MinStack{}
	for _, n := range []int{5, 2, 7, 1} {
		m.Push(n)
	}
	min, _ := m.Min()
	fmt.Println(min) // 1
	m.Pop()
	min, _ = m.Min()
	fmt.Println(min) // 2
}
```

### 2. TypeScript

```typescript
class Stack<T> implements Iterable<T> {
  #items: T[] = [];

  get size(): number { return this.#items.length; }
  get isEmpty(): boolean { return this.#items.length === 0; }

  push(...values: T[]): this {
    this.#items.push(...values);
    return this;
  }

  pop(): T | undefined {
    return this.#items.pop();
  }

  peek(): T | undefined {
    return this.#items.at(-1);
  }

  /** Iterates top to bottom, matching pop order. */
  *[Symbol.iterator](): Iterator<T> {
    for (let i = this.#items.length - 1; i >= 0; i--) yield this.#items[i];
  }
}

/** Bracket matching. Push openers, verify on closers, end empty. */
function isBalanced(input: string): boolean {
  const pairs: Record<string, string> = { ")": "(", "]": "[", "}": "{" };
  const stack = new Stack<string>();
  for (const ch of input) {
    if (ch === "(" || ch === "[" || ch === "{") {
      stack.push(ch);
    } else if (ch in pairs) {
      if (stack.pop() !== pairs[ch]) return false;
    }
  }
  return stack.isEmpty;
}

type Operator = "+" | "-" | "*" | "/";
const OPS: Record<Operator, (a: number, b: number) => number> = {
  "+": (a, b) => a + b,
  "-": (a, b) => a - b,
  "*": (a, b) => a * b,
  "/": (a, b) => Math.trunc(a / b),
};

function isOperator(token: string): token is Operator {
  return token in OPS;
}

/** Reverse Polish notation. Operands stack up, operators consume two. */
function evalRPN(tokens: readonly string[]): number {
  const stack = new Stack<number>();
  for (const token of tokens) {
    if (isOperator(token)) {
      const b = stack.pop();
      const a = stack.pop();
      if (a === undefined || b === undefined) {
        throw new SyntaxError(`not enough operands for "${token}"`);
      }
      if (token === "/" && b === 0) throw new RangeError("division by zero");
      stack.push(OPS[token](a, b));
    } else {
      const n = Number(token);
      if (Number.isNaN(n)) throw new SyntaxError(`bad token "${token}"`);
      stack.push(n);
    }
  }
  const result = stack.pop();
  if (result === undefined || !stack.isEmpty) {
    throw new SyntaxError("malformed expression");
  }
  return result;
}

/** Monotonic decreasing stack of indices. O(n) total. */
function nextGreater(nums: readonly number[]): number[] {
  const out = new Array<number>(nums.length).fill(-1);
  const stack: number[] = [];
  for (let i = 0; i < nums.length; i++) {
    while (stack.length > 0 && nums[stack.at(-1)!] < nums[i]) {
      out[stack.pop()!] = nums[i];
    }
    stack.push(i);
  }
  return out;
}

/** O(1) min by carrying the running minimum on every frame. */
class MinStack {
  #entries: { value: number; min: number }[] = [];

  push(value: number): void {
    const prev = this.#entries.at(-1);
    this.#entries.push({ value, min: prev ? Math.min(prev.min, value) : value });
  }

  pop(): number | undefined {
    return this.#entries.pop()?.value;
  }

  get min(): number | undefined {
    return this.#entries.at(-1)?.min;
  }
}

console.log(isBalanced("{[()]}"));                    // true
console.log(isBalanced("{[(])}"));                    // false
console.log(evalRPN(["2", "3", "4", "*", "+"]));      // 14
console.log(nextGreater([2, 1, 2, 4, 3]));            // [4, 2, 4, -1, -1]

const ms = new MinStack();
[5, 2, 7, 1].forEach((n) => ms.push(n));
console.log(ms.min); // 1
ms.pop();
console.log(ms.min); // 2
```

## Practical Applications & Real-World Use Cases

- **Function call management.** Every language runtime uses a stack for call frames. Stack traces are literally a dump of it.
- **Expression parsing and compilers.** The shunting-yard algorithm converts infix to postfix with two stacks. Virtual machines like the JVM and WASM are stack machines that evaluate by pushing and popping operands.
- **Undo in editors and design tools.** Push each command, pop to undo. Pair with a second stack for redo.
- **Backtracking.** Maze solving, sudoku, N-queens. The stack holds the choices you can still unmake. See [[16-Algorithmic-Strategies]].
- **Syntax highlighting and linters.** Unclosed brackets and tags are found with exactly the balanced-parentheses algorithm above.
- **Depth-first traversal without recursion.** When a graph is deep enough that recursion would overflow, an explicit stack is the fix.

## Related Concepts & Connections

- [[00-DSA-Index|Back to DSA Index]]
- [[05-Queues]] for the FIFO counterpart
- [[13-Recursion]] because the call stack is this structure
- [[14-Graph-Traversal-DFS-BFS]] for iterative DFS
- [[02-Arrays]] and [[03-Linked-Lists]] as the two backing stores
