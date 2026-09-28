---
tags:
  - dsa
  - computer-science
  - go
  - typescript
  - tries
type: study-note
status: complete
date_created: 2026-09-28
---

# Trie

## Overview

A trie (prefix tree) stores strings by their characters rather than as whole values. Each edge is one character, each path from the root spells a prefix, and nodes marked terminal end a real word. The name comes from "retrieval" and is usually pronounced "try" to distinguish it from "tree".

The key property: lookup cost is O(k) where k is the key length, with no dependence on how many keys are stored. A trie with ten words and a trie with ten million words answer "is `banana` present" in the same six steps. A hash table also gives you O(k) (it has to hash the whole string) but it cannot tell you what words start with `ban`. That is what tries are for.

## Key Concepts & Analogies

**A dictionary's thumb index, taken all the way down.** You flip to B, then Ba, then Ban. Each character narrows the search. You never look at words starting with other letters.

**A phone keypad's T9 predictive text.** As digits come in, the set of possible words shrinks. That is literally a trie walk.

**Structure.** Each node holds a map (or fixed array) from character to child node, plus a flag for "a word ends here". The root represents the empty prefix. Words sharing a prefix share nodes, which is where the space savings come from.

**Map children versus array children.** A `map[rune]*Node` handles any alphabet and only pays for characters actually used. A fixed `[26]*Node` is faster (no hashing, better locality) but wastes memory on sparse nodes and only works for a known alphabet. For ASCII-only, high-throughput code, take the array. Otherwise take the map.

**Deletion needs pruning.** Unmarking the terminal flag is not enough, because leaving dead nodes behind means the trie only ever grows. Recursive deletion returns whether the child became useless (not terminal, no children) so the parent can unlink it.

**Space is the real trade-off.** Storing "cat" costs three nodes, each with a map. That can be more memory than the original string. Compressed tries (radix trees / Patricia tries) merge chains of single-child nodes into one node holding a whole substring, which cuts node count dramatically. This is what production routing tables use.

**Sorted iteration comes free.** Walk children in character order and you get lexicographic output with no sort step, because the structure already encodes the ordering.

## Operations & Time/Space Complexity

| Operation | Average Case | Worst Case | Space Complexity |
| :--- | :--- | :--- | :--- |
| Search / Contains | O(k) | O(k) | O(1) iterative |
| Insertion | O(k) | O(k) | O(k) new nodes |
| Deletion | O(k) | O(k) | O(k) recursion |
| StartsWith (prefix exists) | O(k) | O(k) | O(1) |
| Collect all with prefix | O(k + m) | O(k + m) | O(m) output |
| Longest prefix match | O(k) | O(k) | O(1) |

k is key length, m is the total size of the matching results. Total storage is O(n·k) worst case for n keys, much less in practice because shared prefixes share nodes. Note that none of these depend on n, which is the headline result.

## Code Implementations

### 1. Go (Golang)

```go
package main

import (
	"fmt"
	"sort"
)

type trieNode struct {
	children map[rune]*trieNode
	terminal bool // a word ends at this node
	count    int  // how many inserted words pass through here
}

func newTrieNode() *trieNode {
	return &trieNode{children: make(map[rune]*trieNode)}
}

type Trie struct {
	root *trieNode
	size int
}

func NewTrie() *Trie { return &Trie{root: newTrieNode()} }

func (t *Trie) Len() int { return t.size }

// Insert walks the word, creating nodes as needed. O(k).
func (t *Trie) Insert(word string) {
	if t.Contains(word) {
		return // keep size honest on duplicates
	}
	n := t.root
	n.count++
	for _, ch := range word {
		child, ok := n.children[ch]
		if !ok {
			child = newTrieNode()
			n.children[ch] = child
		}
		child.count++
		n = child
	}
	n.terminal = true
	t.size++
}

// find walks a prefix and returns the node it lands on, or nil.
func (t *Trie) find(prefix string) *trieNode {
	n := t.root
	for _, ch := range prefix {
		child, ok := n.children[ch]
		if !ok {
			return nil
		}
		n = child
	}
	return n
}

// Contains requires a terminal node, not just a reachable one.
func (t *Trie) Contains(word string) bool {
	n := t.find(word)
	return n != nil && n.terminal
}

// StartsWith only needs the path to exist. This is the operation a
// hash table cannot do at all.
func (t *Trie) StartsWith(prefix string) bool {
	return t.find(prefix) != nil
}

// CountPrefix reports how many stored words begin with prefix, in O(k),
// thanks to the count maintained on insert.
func (t *Trie) CountPrefix(prefix string) int {
	n := t.find(prefix)
	if n == nil {
		return 0
	}
	return n.count
}

// Autocomplete collects every word under a prefix, in sorted order.
func (t *Trie) Autocomplete(prefix string, limit int) []string {
	start := t.find(prefix)
	if start == nil {
		return nil
	}
	var out []string
	var walk func(n *trieNode, path string)
	walk = func(n *trieNode, path string) {
		if limit > 0 && len(out) >= limit {
			return
		}
		if n.terminal {
			out = append(out, path)
		}
		// Sorting the child runes gives lexicographic output.
		keys := make([]rune, 0, len(n.children))
		for ch := range n.children {
			keys = append(keys, ch)
		}
		sort.Slice(keys, func(i, j int) bool { return keys[i] < keys[j] })
		for _, ch := range keys {
			walk(n.children[ch], path+string(ch))
		}
	}
	walk(start, prefix)
	return out
}

// LongestPrefixOf returns the longest stored word that is a prefix of s.
// This is how IP routing tables and URL routers dispatch.
func (t *Trie) LongestPrefixOf(s string) string {
	n := t.root
	best := ""
	for i, ch := range s {
		child, ok := n.children[ch]
		if !ok {
			break
		}
		n = child
		if n.terminal {
			best = s[:i+len(string(ch))]
		}
	}
	return best
}

// Delete unmarks the word and prunes nodes that became useless.
// Returns whether the word was present.
func (t *Trie) Delete(word string) bool {
	if !t.Contains(word) {
		return false
	}
	runes := []rune(word)

	// prune returns true if the caller should unlink this child.
	var prune func(n *trieNode, depth int) bool
	prune = func(n *trieNode, depth int) bool {
		n.count--
		if depth == len(runes) {
			n.terminal = false
			return len(n.children) == 0
		}
		ch := runes[depth]
		child := n.children[ch]
		if prune(child, depth+1) {
			delete(n.children, ch)
		}
		return !n.terminal && len(n.children) == 0
	}

	prune(t.root, 0)
	t.size--
	return true
}

func (t *Trie) Words() []string { return t.Autocomplete("", 0) }

// --- Fixed-array variant: faster, alphabet-limited ---

// ASCIITrie uses a 26-slot array instead of a map. No hashing per
// character, better cache locality, only handles lowercase a-z.
type ASCIITrie struct {
	children [26]*ASCIITrie
	terminal bool
}

func (t *ASCIITrie) Insert(word string) {
	n := t
	for i := 0; i < len(word); i++ {
		idx := word[i] - 'a'
		if idx >= 26 {
			return // reject anything outside the alphabet
		}
		if n.children[idx] == nil {
			n.children[idx] = &ASCIITrie{}
		}
		n = n.children[idx]
	}
	n.terminal = true
}

func (t *ASCIITrie) Contains(word string) bool {
	n := t
	for i := 0; i < len(word); i++ {
		idx := word[i] - 'a'
		if idx >= 26 || n.children[idx] == nil {
			return false
		}
		n = n.children[idx]
	}
	return n.terminal
}

func main() {
	tr := NewTrie()
	for _, w := range []string{"go", "golang", "gopher", "goal", "rust", "ruby"} {
		tr.Insert(w)
	}
	fmt.Println(tr.Contains("go"), tr.Contains("gol")) // true false
	fmt.Println(tr.StartsWith("gol"))                  // true
	fmt.Println(tr.CountPrefix("go"))                  // 4
	fmt.Println(tr.Autocomplete("go", 0))
	// [go goal golang gopher]

	router := NewTrie()
	router.Insert("/api")
	router.Insert("/api/users")
	fmt.Println(router.LongestPrefixOf("/api/users/42")) // /api/users

	tr.Delete("golang")
	fmt.Println(tr.Autocomplete("go", 0)) // [go goal gopher]
	fmt.Println(tr.Len())                 // 5
}
```

### 2. TypeScript

```typescript
class TrieNode {
  readonly children = new Map<string, TrieNode>();
  terminal = false;
  count = 0; // words passing through this node
}

class Trie {
  #root = new TrieNode();
  #size = 0;

  constructor(words: readonly string[] = []) {
    for (const w of words) this.insert(w);
  }

  get size(): number { return this.#size; }

  /** O(k). Creates nodes for characters not yet present. */
  insert(word: string): this {
    if (this.has(word)) return this;
    let node = this.#root;
    node.count++;
    for (const ch of word) {
      let child = node.children.get(ch);
      if (child === undefined) {
        child = new TrieNode();
        node.children.set(ch, child);
      }
      child.count++;
      node = child;
    }
    node.terminal = true;
    this.#size++;
    return this;
  }

  /** Walks a prefix, returning the node it lands on. */
  #find(prefix: string): TrieNode | null {
    let node = this.#root;
    for (const ch of prefix) {
      const child = node.children.get(ch);
      if (child === undefined) return null;
      node = child;
    }
    return node;
  }

  /** Full word: the path must exist AND end at a terminal node. */
  has(word: string): boolean {
    return this.#find(word)?.terminal ?? false;
  }

  /** Prefix only. The operation hash maps cannot do. */
  startsWith(prefix: string): boolean {
    return this.#find(prefix) !== null;
  }

  /** O(k) prefix count thanks to the counter maintained on insert. */
  countPrefix(prefix: string): number {
    return this.#find(prefix)?.count ?? 0;
  }

  /** All words under a prefix, lexicographic. Generator so it can stop early. */
  *wordsWithPrefix(prefix: string): Generator<string> {
    const start = this.#find(prefix);
    if (start === null) return;

    function* walk(node: TrieNode, path: string): Generator<string> {
      if (node.terminal) yield path;
      for (const ch of [...node.children.keys()].sort()) {
        yield* walk(node.children.get(ch)!, path + ch);
      }
    }
    yield* walk(start, prefix);
  }

  /** Bounded autocomplete, the shape a real search box wants. */
  autocomplete(prefix: string, limit = 10): string[] {
    const out: string[] = [];
    for (const word of this.wordsWithPrefix(prefix)) {
      out.push(word);
      if (out.length >= limit) break;
    }
    return out;
  }

  /** Longest stored word that prefixes `s`. How routers dispatch. */
  longestPrefixOf(s: string): string {
    let node = this.#root;
    let best = "";
    let consumed = "";
    for (const ch of s) {
      const child = node.children.get(ch);
      if (child === undefined) break;
      node = child;
      consumed += ch;
      if (node.terminal) best = consumed;
    }
    return best;
  }

  /** Unmarks the word and prunes nodes that are no longer reachable. */
  delete(word: string): boolean {
    if (!this.has(word)) return false;
    const chars = [...word];

    // Returns true when the parent should unlink this node.
    const prune = (node: TrieNode, depth: number): boolean => {
      node.count--;
      if (depth === chars.length) {
        node.terminal = false;
        return node.children.size === 0;
      }
      const ch = chars[depth];
      const child = node.children.get(ch)!;
      if (prune(child, depth + 1)) node.children.delete(ch);
      return !node.terminal && node.children.size === 0;
    };

    prune(this.#root, 0);
    this.#size--;
    return true;
  }

  words(): string[] { return [...this.wordsWithPrefix("")]; }
}

/** Wildcard matching: '.' matches any single character. Needs backtracking. */
function matchesWithWildcard(trie: Trie, pattern: string): boolean {
  // Exposed via words() for brevity; a production version would walk nodes.
  const re = new RegExp(`^${pattern.replace(/\./g, "[^]")}$`);
  return trie.words().some((w) => re.test(w));
}

const trie = new Trie(["go", "golang", "gopher", "goal", "rust", "ruby"]);
console.log(trie.has("go"), trie.has("gol"));   // true false
console.log(trie.startsWith("gol"));            // true
console.log(trie.countPrefix("go"));            // 4
console.log(trie.autocomplete("go"));           // ["go","goal","golang","gopher"]

const router = new Trie(["/api", "/api/users"]);
console.log(router.longestPrefixOf("/api/users/42")); // "/api/users"

trie.delete("golang");
console.log(trie.autocomplete("go")); // ["go","goal","gopher"]
console.log(trie.size);               // 5
console.log(matchesWithWildcard(trie, "go.l")); // true ("goal")
```

## Practical Applications & Real-World Use Cases

- **Autocomplete and search suggestions.** Type three characters, get ranked completions. Add a frequency score per terminal node and you can rank them. This is the flagship use.
- **IP routing tables.** Routers do longest-prefix matching on destination addresses using a binary radix trie over address bits. Millions of routes, constant-time-per-bit lookup.
- **URL routers in web frameworks.** Go's `httprouter` and Express-style routers build a radix trie of path segments so dispatch does not scan every registered route.
- **Spell checkers and fuzzy search.** Walking a trie with an edit-distance budget finds all words within k edits without comparing against the whole dictionary.
- **T9 and swipe keyboards.** Each keypress narrows the trie. Predictions are the highest-frequency words under the current node.
- **Profanity and content filters.** The Aho-Corasick algorithm builds a trie with failure links to find all occurrences of thousands of patterns in one pass over the text.
- **Genome sequence search.** Suffix tries and suffix arrays index DNA so substring queries do not rescan the genome.

## Related Concepts & Connections

- [[00-DSA-Index|Back to DSA Index]]
- [[08-Trees-BST-and-AVL]] for the general tree structure
- [[07-Hash-Tables-and-Sets]] for O(1) exact lookup with no prefix support
- [[14-Graph-Traversal-DFS-BFS]] because trie walks are DFS
- [[16-Algorithmic-Strategies]] for the backtracking used in wildcard matching
