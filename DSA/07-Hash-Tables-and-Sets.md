---
tags:
  - dsa
  - computer-science
  - go
  - typescript
  - hashing
type: study-note
status: complete
date_created: 2026-09-28
---

# Hash Tables and Sets

## Overview

A hash table turns a key into an array index by running it through a hash function. Because array indexing is O(1), lookup becomes O(1) too. That is the whole idea, and it is the single most useful data structure in practical programming.

A set is a hash table where you only care whether the key is present. Same mechanics, no values.

The catch is collisions. Two different keys can hash to the same slot, and the entire engineering of hash tables is about handling that gracefully.

## Key Concepts & Analogies

**A coat check.** You hand over a coat and get a numbered tag. Retrieval does not involve searching the rack; the number tells the attendant exactly where to look. The hash function is the tag-numbering scheme.

**A library where the shelf is computed from the title.** No catalog walk, no scanning. Compute, go, done.

**Hash function requirements.** It must be deterministic, fast, and spread keys evenly. Even spreading matters most: a hash function that sends 90% of keys to the same slot gives you a linked list wearing a hash table costume. FNV-1a and xxHash are good general choices; cryptographic hashes like SHA-256 are overkill and slow for this purpose.

**Collision resolution, two families:**

*Separate chaining* puts a linked list (or small slice) in each bucket. Simple, tolerates load factors above 1, wastes a pointer per entry. This is what the code below implements because it is the clearer teaching version.

*Open addressing* stores everything in the main array and probes for the next free slot on collision. Linear probing (`i+1`), quadratic probing (`i+k²`), and double hashing are the variants. Better cache behavior, no per-entry allocation, but deletion needs tombstones and performance collapses past a load factor of about 0.7. Go's map and Python's dict both use open addressing variants.

**Load factor is the tuning knob.** Load factor = entries / buckets. When it crosses a threshold (0.75 is a common choice) the table resizes: allocate roughly double the buckets and rehash every entry. Rehashing is O(n), which makes individual inserts spike, but it amortizes to O(1).

**The O(1) is average, not worst.** With adversarial keys that all collide, every operation degrades to O(n). This is a real attack, called hash flooding: send an HTTP request with thousands of headers that collide and a server spends quadratic time parsing. The fix is a randomly seeded hash per process, which Go and most modern runtimes do by default.

**Keys must be hashable and immutable in practice.** Mutating a key after insertion changes its hash, and the entry becomes unreachable. In Go the compiler enforces comparability. In JavaScript `Map` uses reference identity for objects, which is often not what people expect.

**Ordering is not guaranteed and Go actively randomizes it.** Iterating a Go map gives a different order every run, on purpose, to stop people depending on it. JavaScript `Map` does preserve insertion order, which is a language guarantee you can rely on.

## Operations & Time/Space Complexity

| Operation | Average Case | Worst Case | Space Complexity |
| :--- | :--- | :--- | :--- |
| Search / Get | O(1) | O(n) | O(1) |
| Insertion | O(1) amortized | O(n) on resize | O(1) |
| Deletion | O(1) | O(n) | O(1) |
| Iterate all | O(n + b) | O(n + b) | O(1) |
| Resize / rehash | O(n) | O(n) | O(n) |

b is the bucket count. Total storage is O(n + b). Worst case assumes every key collides, which good hashing plus a random seed makes vanishingly unlikely.

## Code Implementations

### 1. Go (Golang)

```go
package main

import (
	"fmt"
	"hash/fnv"
	"sort"
)

const maxLoadFactor = 0.75

type entry[K comparable, V any] struct {
	key   K
	value V
	next  *entry[K, V] // separate chaining
}

// HashMap implements separate chaining so the collision handling is
// visible. Go's built-in map uses open addressing and is faster.
type HashMap[K comparable, V any] struct {
	buckets []*entry[K, V]
	size    int
}

func NewHashMap[K comparable, V any]() *HashMap[K, V] {
	return &HashMap[K, V]{buckets: make([]*entry[K, V], 8)}
}

func (m *HashMap[K, V]) Len() int { return m.size }

// hash uses FNV-1a over the key's default formatting. Real code would
// specialize per type; this keeps the generic signature simple.
func (m *HashMap[K, V]) hash(k K) int {
	h := fnv.New64a()
	fmt.Fprintf(h, "%v", k)
	return int(h.Sum64() % uint64(len(m.buckets)))
}

func (m *HashMap[K, V]) Get(k K) (V, bool) {
	var zero V
	for e := m.buckets[m.hash(k)]; e != nil; e = e.next {
		if e.key == k {
			return e.value, true
		}
	}
	return zero, false
}

// Set inserts or updates, then resizes if the table got too crowded.
func (m *HashMap[K, V]) Set(k K, v V) {
	i := m.hash(k)
	for e := m.buckets[i]; e != nil; e = e.next {
		if e.key == k {
			e.value = v // update in place, size unchanged
			return
		}
	}
	m.buckets[i] = &entry[K, V]{key: k, value: v, next: m.buckets[i]}
	m.size++

	if float64(m.size)/float64(len(m.buckets)) > maxLoadFactor {
		m.resize()
	}
}

func (m *HashMap[K, V]) Delete(k K) bool {
	i := m.hash(k)
	var prev *entry[K, V]
	for e := m.buckets[i]; e != nil; e = e.next {
		if e.key == k {
			if prev == nil {
				m.buckets[i] = e.next
			} else {
				prev.next = e.next
			}
			m.size--
			return true
		}
		prev = e
	}
	return false
}

// resize doubles the bucket count and rehashes everything. O(n), which
// is why insert is amortized rather than strictly constant.
func (m *HashMap[K, V]) resize() {
	old := m.buckets
	m.buckets = make([]*entry[K, V], len(old)*2)
	for _, head := range old {
		for e := head; e != nil; {
			next := e.next
			i := m.hash(e.key) // recomputed against the new bucket count
			e.next = m.buckets[i]
			m.buckets[i] = e
			e = next
		}
	}
}

func (m *HashMap[K, V]) Keys() []K {
	out := make([]K, 0, m.size)
	for _, head := range m.buckets {
		for e := head; e != nil; e = e.next {
			out = append(out, e.key)
		}
	}
	return out
}

// Set is the value-free version. struct{} costs zero bytes.
type Set[T comparable] map[T]struct{}

func NewSet[T comparable](items ...T) Set[T] {
	s := make(Set[T], len(items))
	s.Add(items...)
	return s
}

func (s Set[T]) Add(items ...T) {
	for _, it := range items {
		s[it] = struct{}{}
	}
}

func (s Set[T]) Has(item T) bool { _, ok := s[item]; return ok }
func (s Set[T]) Remove(item T)   { delete(s, item) }
func (s Set[T]) Len() int        { return len(s) }

func (s Set[T]) Union(other Set[T]) Set[T] {
	out := make(Set[T], len(s)+len(other))
	for k := range s {
		out[k] = struct{}{}
	}
	for k := range other {
		out[k] = struct{}{}
	}
	return out
}

// Intersection iterates the smaller set. O(min(n,m)) instead of O(n).
func (s Set[T]) Intersection(other Set[T]) Set[T] {
	small, large := s, other
	if len(other) < len(s) {
		small, large = other, s
	}
	out := make(Set[T])
	for k := range small {
		if large.Has(k) {
			out[k] = struct{}{}
		}
	}
	return out
}

func (s Set[T]) Difference(other Set[T]) Set[T] {
	out := make(Set[T])
	for k := range s {
		if !other.Has(k) {
			out[k] = struct{}{}
		}
	}
	return out
}

// --- What hash maps are actually for ---

// WordFrequency in one pass. The canonical use.
func WordFrequency(words []string) map[string]int {
	counts := make(map[string]int, len(words))
	for _, w := range words {
		counts[w]++ // zero value of int is 0, so no init needed
	}
	return counts
}

// GroupAnagrams keys by sorted letters. Hashing a derived key is a
// pattern worth internalizing.
func GroupAnagrams(words []string) [][]string {
	groups := make(map[string][]string)
	for _, w := range words {
		letters := []byte(w)
		sort.Slice(letters, func(i, j int) bool { return letters[i] < letters[j] })
		key := string(letters)
		groups[key] = append(groups[key], w)
	}
	out := make([][]string, 0, len(groups))
	for _, g := range groups {
		out = append(out, g)
	}
	return out
}

func main() {
	m := NewHashMap[string, int]()
	for i, lang := range []string{"go", "ts", "rust", "zig", "c", "ocaml", "elixir"} {
		m.Set(lang, i)
	}
	v, ok := m.Get("rust")
	fmt.Println(v, ok)      // 2 true
	m.Delete("c")
	fmt.Println(m.Len())    // 6

	a := NewSet(1, 2, 3, 4)
	b := NewSet(3, 4, 5)
	fmt.Println(len(a.Intersection(b)), len(a.Union(b)), len(a.Difference(b)))
	// 2 5 2

	fmt.Println(WordFrequency([]string{"a", "b", "a", "c", "a"}))
	// map[a:3 b:1 c:1]
	fmt.Println(GroupAnagrams([]string{"eat", "tea", "tan", "ate", "nat"}))
}
```

### 2. TypeScript

```typescript
interface Entry<K, V> {
  key: K;
  value: V;
  next: Entry<K, V> | null;
}

/**
 * Separate-chaining hash map. The built-in Map is faster; this exists
 * so the collision handling and resize are not hidden.
 */
class HashMap<K, V> implements Iterable<[K, V]> {
  static readonly #MAX_LOAD = 0.75;
  #buckets: (Entry<K, V> | null)[] = new Array(8).fill(null);
  #size = 0;

  get size(): number { return this.#size; }

  /** FNV-1a over the key's string form. */
  #hash(key: K): number {
    const s = typeof key === "string" ? key : JSON.stringify(key) ?? String(key);
    let h = 0x811c9dc5;
    for (let i = 0; i < s.length; i++) {
      h ^= s.charCodeAt(i);
      h = Math.imul(h, 0x01000193) >>> 0; // imul keeps 32-bit overflow correct
    }
    return h % this.#buckets.length;
  }

  get(key: K): V | undefined {
    for (let e = this.#buckets[this.#hash(key)]; e !== null; e = e.next) {
      if (e.key === key) return e.value;
    }
    return undefined;
  }

  has(key: K): boolean {
    for (let e = this.#buckets[this.#hash(key)]; e !== null; e = e.next) {
      if (e.key === key) return true;
    }
    return false;
  }

  set(key: K, value: V): this {
    const i = this.#hash(key);
    for (let e = this.#buckets[i]; e !== null; e = e.next) {
      if (e.key === key) {
        e.value = value;
        return this;
      }
    }
    this.#buckets[i] = { key, value, next: this.#buckets[i] };
    this.#size++;
    if (this.#size / this.#buckets.length > HashMap.#MAX_LOAD) this.#resize();
    return this;
  }

  delete(key: K): boolean {
    const i = this.#hash(key);
    let prev: Entry<K, V> | null = null;
    for (let e = this.#buckets[i]; e !== null; e = e.next) {
      if (e.key === key) {
        if (prev === null) this.#buckets[i] = e.next;
        else prev.next = e.next;
        this.#size--;
        return true;
      }
      prev = e;
    }
    return false;
  }

  /** Doubles the buckets and rehashes. O(n), amortized away over inserts. */
  #resize(): void {
    const old = this.#buckets;
    this.#buckets = new Array(old.length * 2).fill(null);
    for (const head of old) {
      let e = head;
      while (e !== null) {
        const next = e.next;
        const i = this.#hash(e.key);
        e.next = this.#buckets[i];
        this.#buckets[i] = e;
        e = next;
      }
    }
  }

  *[Symbol.iterator](): Iterator<[K, V]> {
    for (const head of this.#buckets) {
      for (let e = head; e !== null; e = e.next) yield [e.key, e.value];
    }
  }
}

/** Set algebra over the built-in Set. Intersection walks the smaller one. */
const SetOps = {
  union<T>(a: ReadonlySet<T>, b: ReadonlySet<T>): Set<T> {
    return new Set([...a, ...b]);
  },
  intersection<T>(a: ReadonlySet<T>, b: ReadonlySet<T>): Set<T> {
    const [small, large] = a.size <= b.size ? [a, b] : [b, a];
    return new Set([...small].filter((x) => large.has(x)));
  },
  difference<T>(a: ReadonlySet<T>, b: ReadonlySet<T>): Set<T> {
    return new Set([...a].filter((x) => !b.has(x)));
  },
  isSubset<T>(a: ReadonlySet<T>, b: ReadonlySet<T>): boolean {
    return a.size <= b.size && [...a].every((x) => b.has(x));
  },
} as const;

/** One pass, O(n). The most common hash map use there is. */
function wordFrequency(words: readonly string[]): Map<string, number> {
  const counts = new Map<string, number>();
  for (const w of words) counts.set(w, (counts.get(w) ?? 0) + 1);
  return counts;
}

/** Key by sorted letters. Deriving a canonical key is the trick here. */
function groupAnagrams(words: readonly string[]): string[][] {
  const groups = new Map<string, string[]>();
  for (const w of words) {
    const key = [...w].sort().join("");
    const bucket = groups.get(key);
    if (bucket) bucket.push(w);
    else groups.set(key, [w]);
  }
  return [...groups.values()];
}

/** First non-repeating character. Two passes, O(n). */
function firstUnique(s: string): string | null {
  const counts = new Map<string, number>();
  for (const ch of s) counts.set(ch, (counts.get(ch) ?? 0) + 1);
  for (const ch of s) if (counts.get(ch) === 1) return ch;
  return null;
}

const hm = new HashMap<string, number>();
["go", "ts", "rust", "zig", "c", "ocaml", "elixir"].forEach((l, i) => hm.set(l, i));
console.log(hm.get("rust"), hm.size); // 2 7
hm.delete("c");
console.log(hm.size);                 // 6

const a = new Set([1, 2, 3, 4]);
const b = new Set([3, 4, 5]);
console.log([...SetOps.intersection(a, b)]); // [3, 4]
console.log([...SetOps.difference(a, b)]);   // [1, 2]

console.log(wordFrequency(["a", "b", "a", "c", "a"]));
console.log(groupAnagrams(["eat", "tea", "tan", "ate", "nat"]));
console.log(firstUnique("swiss")); // "w"
```

## Practical Applications & Real-World Use Cases

- **Database indexes.** Hash indexes give O(1) equality lookup. They cannot do range queries, which is why B-trees remain the default. Knowing the difference tells you which index to create.
- **Caching layers.** Redis and Memcached are hash tables with a network protocol. In-process caches pair a hash map with a doubly linked list for LRU eviction. See [[03-Linked-Lists]].
- **Compilers and interpreters.** Symbol tables map identifiers to type and scope info, hit on every single name resolution.
- **Deduplication.** Whether you are stripping duplicate rows or checking whether a URL has been crawled, a set is the answer.
- **Router tables in web frameworks.** Static routes go in a hash map for O(1) dispatch; dynamic segments fall back to a trie. See [[09-Trie]].
- **Content-addressed storage.** Git names every object by the hash of its contents. Docker layers and IPFS work the same way.
- **Bloom filters.** When even O(n) space is too much, several hash functions over a bit array answer "definitely not present" or "probably present". CDNs and databases use them to skip expensive lookups.

## Related Concepts & Connections

- [[00-DSA-Index|Back to DSA Index]]
- [[03-Linked-Lists]] for separate chaining's per-bucket structure
- [[02-Arrays]] for the bucket array underneath
- [[08-Trees-BST-and-AVL]] for ordered alternatives that support range queries
- [[09-Trie]] for prefix lookups, which hashing cannot do
- [[11-Searching-Algorithms]] for the O(1) versus O(log n) comparison
