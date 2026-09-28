---
tags:
  - dsa
  - computer-science
  - go
  - typescript
  - moc
type: index
status: complete
date_created: 2026-09-28
---

# DSA Index (Map of Content)

Sixteen notes covering the data structures and algorithms that show up in real systems and in interviews. Every note carries the same shape: definition, mental model, complexity table, a working Go implementation, a working TypeScript implementation, and where the thing actually gets used.

Read them in order if you are starting fresh. The later notes assume the earlier ones.

## Part 1: Foundations

| # | Note | What it covers |
| :--- | :--- | :--- |
| 01 | [[01-Fundamentals-and-Introduction\|Fundamentals and Introduction]] | Big-O, memory model, how to reason about cost |
| 02 | [[02-Arrays\|Arrays]] | Contiguous memory, dynamic resizing, slices |
| 03 | [[03-Linked-Lists\|Linked Lists]] | Pointer chains, singly, doubly, circular |

## Part 2: Linear abstract types

| # | Note | What it covers |
| :--- | :--- | :--- |
| 04 | [[04-Stacks\|Stacks]] | LIFO, call stacks, expression parsing |
| 05 | [[05-Queues\|Queues]] | FIFO, ring buffers, deques |
| 06 | [[06-Priority-Queue-and-Heap\|Priority Queue and Heap]] | Binary heaps, sift up/down, schedulers |

## Part 3: Associative and hierarchical

| # | Note | What it covers |
| :--- | :--- | :--- |
| 07 | [[07-Hash-Tables-and-Sets\|Hash Tables and Sets]] | Hashing, collisions, load factor |
| 08 | [[08-Trees-BST-and-AVL\|Trees, BST and AVL]] | Binary search trees, rotations, balance |
| 09 | [[09-Trie\|Trie]] | Prefix trees, autocomplete, IP routing |
| 10 | [[10-Graphs\|Graphs]] | Adjacency list vs matrix, directed, weighted |

## Part 4: Algorithms

| # | Note | What it covers |
| :--- | :--- | :--- |
| 11 | [[11-Searching-Algorithms\|Searching Algorithms]] | Linear, binary, binary search on answer |
| 12 | [[12-Sorting-Algorithms\|Sorting Algorithms]] | Merge, quick, heap, counting, stability |
| 13 | [[13-Recursion\|Recursion]] | Base cases, call stack, memoization |
| 14 | [[14-Graph-Traversal-DFS-BFS\|Graph Traversal: DFS and BFS]] | Visit orders, cycle detection, topological sort |
| 15 | [[15-Shortest-Path-Dijkstras-Algorithm\|Shortest Path: Dijkstra]] | Greedy relaxation, heaps, Bellman-Ford contrast |
| 16 | [[16-Algorithmic-Strategies\|Algorithmic Strategies]] | Divide and conquer, greedy, DP, backtracking |

## How to use this vault

Pick a note, read the Overview and Key Concepts, then close the file and try to write the implementation yourself before reading the code. Reading code you did not struggle with teaches almost nothing.

The complexity tables are worth memorizing. Not because interviewers quiz you on them, but because the numbers tell you which structure to reach for without having to think.
