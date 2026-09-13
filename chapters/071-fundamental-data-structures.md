# Chapter 71 — Fundamental Data Structures

> **Curriculum position:** Part XIII — Data Structures and Algorithms
> **Previous chapter:** Chapter 70 — Source Maps and Production Debugging
> **Next chapter:** Chapter 72 — Complexity
> **Primary environment:** Modern JavaScript / TypeScript in Node.js and browsers.

---

# Chapter Mission

Master fundamental data structures as **engineering representations**, not as a list of interview definitions.

The central chain is:

```text
requirement
    ↓
operations
    ↓
data representation
    ↓
invariants
    ↓
time cost
    ↓
space cost
    ↓
runtime / memory behavior
    ↓
production trade-offs
```

A principal engineer should be able to hear:

> “We need a bounded FIFO queue with cancellation by ID.”

and reason through:

```text
What behavior?
What operations dominate?
Is size bounded?
Do we need arbitrary lookup?
How often is cancellation used?
What happens under memory pressure?
What representation keeps the important operations cheap?
```

This chapter covers arrays, dynamic arrays, stacks, queues, deques, ring buffers, linked lists, hash tables, Map, Set, heaps, priority queues, trees, tries, graphs, adjacency structures, specialized representations, composite structures, invariants, memory behavior, benchmarking, security, and production selection.

The chapter's principal-level rule is:

> **Choose a representation because it fits the workload and invariants, not because the structure has a famous Big-O label.**


# 1. Learning Objectives

By completion you should be able to:

- distinguish an abstract data type from a concrete data structure;
- define and defend representation invariants;
- explain contiguous versus linked representations;
- explain arrays, dynamic arrays, stacks, queues, deques, and ring buffers;
- implement singly and doubly linked lists;
- explain hashing, collisions, load factor, and resizing;
- choose Map and Set by semantics rather than folklore;
- implement a heap and priority queue;
- distinguish binary trees from binary search trees;
- explain tries and prefix-oriented lookup;
- model directed, undirected, weighted, cyclic, and disconnected graphs;
- compare adjacency lists and adjacency matrices;
- reason about memory overhead, garbage collection, and locality;
- recognize accidental quadratic behavior;
- design composite structures such as LRU caches;
- benchmark competing implementations under realistic workloads;
- defend a data-structure choice in a production design review.


# 2. Prerequisites

Recommended prior knowledge:

- Chapter 02 — Values and Types
- Chapter 15 — Objects and Property Semantics
- Chapter 16 — Property Keys, Ordering, and Enumeration
- Chapter 22 — Arrays
- Chapter 24 — Objects, Map, Set, WeakMap, WeakSet
- Chapter 27 — Typed Arrays and Binary Data
- Chapter 45 — Memory and Garbage Collection
- Chapter 47 — JavaScript Engine Architecture
- Chapter 48 — V8 Internals and Optimization
- Chapter 63 — Async Context and Diagnostics

Chapter 72 formalizes asymptotic complexity. This chapter uses cost language while keeping the emphasis on representation and workload.


# 3. What Is It?

A data structure is an organized representation of data that supports a defined set of operations.

Examples:

```text
Array       → indexed sequence
Stack       → LIFO
Queue       → FIFO
Deque       → two-ended sequence
Map         → key/value association
Set         → unique membership
Heap        → priority
Trie        → prefix structure
Graph       → relationships
```

A data structure has at least four conceptual pieces:

```text
state
operations
invariants
cost profile
```

Two structures can represent the same abstract data but produce very different runtime behavior.


# 4. Why Does It Exist?

Data structures exist because different workloads require different efficient operations.

Example:

```text
exact key lookup
```

does not have the same representation needs as:

```text
remove oldest item
```

or:

```text
return smallest priority
```

or:

```text
find every word starting with "jav"
```

A useful structure makes the dominant operation natural.

A poor structure forces repeated work.

This is why data-structure selection is part of architecture.


# 5. Mental Model

For every structure ask:

```text
1. What abstract behavior is required?
2. What is the physical/logical representation?
3. What invariant guarantees correctness?
4. Which operations are common?
5. Which operations are expensive?
6. How does the structure grow?
7. How does it shrink?
8. What references can remain live?
9. What happens under malformed or hostile input?
10. Can the implementation change behind the API?
```

The representation should be hidden behind a stable abstraction whenever practical.


# 6. Abstract Data Type vs Data Structure

An **abstract data type (ADT)** specifies behavior.

A **data structure** specifies representation.

```text
Queue = behavior
Array / linked list / ring buffer = possible representations
```

Example queue contract:

```text
enqueue(x)
dequeue()
peek()
isEmpty()
```

with FIFO semantics.

The consumer should depend on the behavior, not necessarily on the representation.

This distinction allows:

```text
implementation A
      ↓
implementation B
```

without changing callers.


# 7. Representation Invariant

A representation invariant is a property that must remain true for every valid state.

Examples:

### Stack

```text
last pushed logical item is at top
```

### Queue

```text
head identifies the oldest logical item
```

### Singly linked list

```text
tail.next === null
```

for a non-circular implementation.

### Doubly linked list

For every neighboring node pair:

```text
node.next.prev === node
```

when `next` exists.

### Min-heap

```text
parent <= child
```

for every parent/child relation.

### Hash map

```text
every live key maps to exactly one logical value
```

A mutation is correct when it preserves the invariant.


# 8. Contiguous vs Linked Storage

A useful conceptual distinction is:

```text
contiguous-like representation:
[A][B][C][D][E]

linked representation:
A → B → C → D → E
```

Contiguous representations tend to favor indexed access and locality.

Linked representations can make local structural mutation convenient when the relevant node/reference is already known.

Neither is universally better.

In JavaScript, the runtime representation is implementation-specific, so avoid claims such as:

> “Every Array is exactly a C-style contiguous buffer.”

The language exposes semantics; engines choose internal layouts and optimize according to observed usage.


# 9. Arrays

A JavaScript array represents an indexed sequence:

```js
const values = [10, 20, 30, 40];

console.log(values[2]);
console.log(values.length);
```

Prediction:

```text
30
4
```

Arrays are natural for:

- ordered collections;
- indexed access;
- iteration;
- append-heavy workloads;
- batch processing.

Important edge cases include:

- holes;
- sparse indices;
- front insertion/removal;
- mutation during iteration;
- large lengths;
- non-index properties.


# 10. Dynamic Arrays and Amortized Growth

A dynamic-array-like strategy keeps logical length separate from storage capacity.

Conceptually:

```text
capacity = 8
length   = 6

[A][B][C][D][E][F][ ][ ]
```

Appending `G` is cheap while capacity exists.

When capacity is exhausted:

```text
allocate larger storage
copy/move data as needed
continue
```

Therefore an individual append may occasionally be expensive even though a long sequence of appends can have a favorable amortized cost.

The exact growth strategy is engine-specific.


# 11. Front Operations

`unshift()` and `shift()` change the logical front of a sequence.

Example:

```js
const values = ["B", "C"];
values.unshift("A");
console.log(values);
```

Result:

```text
["A", "B", "C"]
```

Repeated front operations on a large sequence may require substantial work.

This leads to a classic mistake:

```js
while (queue.length) {
  process(queue.shift());
}
```

For high-volume queues, use a representation designed for efficient front removal.


# 12. Sparse Arrays and Holes

A hole is different from an explicit `undefined`.

```js
const a = [];
a[2] = "x";

console.log(a.length);
console.log(0 in a);
console.log(2 in a);
```

Prediction:

```text
3
false
true
```

By contrast:

```js
const b = new Array(3).fill(undefined);
```

contains actual properties at those positions.

Sparse patterns can also influence JavaScript engine optimization choices.


# 13. Stack

A stack is LIFO:

```text
Last In, First Out
```

Operations:

```text
push
pop
peek
```

Natural JavaScript representation:

```js
class Stack {
  #items = [];

  push(value) {
    this.#items.push(value);
  }

  pop() {
    return this.#items.pop();
  }

  peek() {
    return this.#items.at(-1);
  }

  get size() {
    return this.#items.length;
  }
}
```

For ordinary JavaScript application code this is usually clearer than building a linked-node stack without a requirement for one.


# 14. Queue

A queue is FIFO:

```text
First In, First Out
```

Operations:

```text
enqueue
dequeue
peek
```

Avoid assuming an ordinary array is always an appropriate implementation.

For a high-volume queue, alternatives include:

```text
head-index array
ring buffer
deque
linked representation
two-stack queue
```

Choose using the actual operation profile.


# 15. Head-Index Queue

A simple strategy is:

```js
class Queue {
  #items = [];
  #head = 0;

  enqueue(value) {
    this.#items.push(value);
  }

  dequeue() {
    if (this.#head >= this.#items.length) return undefined;
    return this.#items[this.#head++];
  }

  peek() {
    return this.#items[this.#head];
  }

  get size() {
    return this.#items.length - this.#head;
  }
}
```

The important subtlety is retention: the backing array still contains old references until compaction becomes necessary.

A production implementation needs an explicit compaction strategy or a different bounded representation.


# 16. Deque

A deque supports both ends:

```text
pushFront
pushBack
popFront
popBack
```

Visual model:

```text
front ← [A][B][C] → back
```

Useful workloads include:

- sliding windows;
- BFS;
- work queues;
- bounded buffers;
- cache structures.

A ring buffer is one practical bounded deque representation.


# 17. Ring Buffer

A ring buffer treats storage as circular.

Conceptually:

```text
[0][1][2][3][4][5]
 ↑           ↑
head        tail
```

After position 5:

```text
next = 0
```

Typical state:

```text
head = next readable position
tail = next writable position
size = logical element count
capacity = fixed storage length
```

This is particularly effective when the maximum size is known.


# 18. Ring Buffer Invariants

A production ring buffer should define:

```text
0 <= size <= capacity
head is always a valid storage index
tail is always a valid storage index
logical count matches occupied slots
```

If full and empty states share the same head/tail coordinates, an explicit `size` or an alternate sentinel convention is required.

Edge cases:

```text
capacity = 0
capacity = 1
enqueue when full
dequeue when empty
wrap-around
exactly-full state
exactly-empty state
```

These cases should be tested explicitly.


# 19. Singly Linked List

A singly linked list stores:

```text
value
next
```

Example:

```js
class Node {
  constructor(value, next = null) {
    this.value = value;
    this.next = next;
  }
}
```

Visual:

```text
A → B → C → null
```

Strength:

```text
local pointer changes
```

Weakness:

```text
finding a distant position requires traversal
```

For JavaScript, object allocation and reference chasing can make linked lists slower than expected for many workloads.


# 20. Doubly Linked List

A doubly linked node stores:

```text
prev
value
next
```

Visual:

```text
null ← A ⇄ B ⇄ C → null
```

Benefits:

- bidirectional traversal;
- convenient removal of a known node;
- useful for LRU-style structures.

Costs:

- another reference per node;
- more mutation points;
- more invariant checks;
- more memory overhead.


# 21. Circular Linked List

In a circular list:

```text
A → B → C
↑       ↓
└───────┘
```

The last node points back to the first.

Useful for:

- cyclic scheduling;
- round-robin processing;
- repeated sequences.

The major danger is traversal termination.

Every traversal needs a deliberate stopping condition.


# 22. Why Linked Lists Are Often Overused

The common interview statement is:

> “Linked-list insertion is O(1).”

The hidden condition is important:

```text
you already know the insertion position
```

If finding that position requires a traversal, total work includes that traversal.

JavaScript introduces additional practical factors:

```text
object allocation
reference chasing
GC pressure
locality
property access
```

Therefore a theoretically favorable operation may lose to an array in a real benchmark.


# 23. Hash Tables

A hash table maps:

```text
key → value
```

Conceptually:

```text
hash(key)
   ↓
bucket / slot
   ↓
entry
```

The hash function converts a key to a value from which an index can be derived.

Different keys can produce the same bucket or probe path.

That is a collision.


# 24. Collision Strategies

Two classical families:

### Separate chaining

```text
bucket 3
  ↓
[A] → [B] → [C]
```

### Open addressing

```text
slot A occupied
      ↓
probe another slot
```

Real production runtimes can use specialized internal strategies.

The important engineering point is that collision behavior affects actual lookup cost.


# 25. Load Factor and Resize

Conceptually:

```text
load factor = entries / capacity
```

As the table becomes crowded, collision/probe behavior can worsen.

A resize typically means:

```text
1. allocate larger table
2. revisit entries
3. calculate their new positions
4. restore invariants
```

A crucial detail:

```text
old bucket index
≠
new bucket index
```

in general.

A resize bug that moves entries without recomputing placement can silently lose lookups.


# 26. JavaScript Map

`Map` expresses key/value collection semantics:

```js
const users = new Map();

users.set("u1", { name: "A" });

console.log(users.get("u1"));
console.log(users.has("u1"));
```

Use `Map` when the data is naturally a collection of associations.

Important semantic differences from objects include:

- arbitrary key values;
- explicit collection methods;
- `size`;
- iteration semantics;
- different prototype behavior;
- key identity rules.


# 27. JavaScript Set

`Set` represents unique membership.

```js
const ids = new Set([1, 2, 2, 3]);

console.log([...ids]);
```

Prediction:

```text
[1, 2, 3]
```

Use it when the dominant question is:

```text
"Have I already seen this?"
```

rather than:

```text
"What value belongs to this key?"
```


# 28. Key Identity

Object keys in `Map` and object values in `Set` use JavaScript identity semantics.

```js
const a = {};
const b = {};

const set = new Set([a]);

console.log(set.has(a));
console.log(set.has(b));
```

Prediction:

```text
true
false
```

because `a` and `b` are distinct objects.

Never substitute a derived string representation for identity unless the domain intentionally defines canonicalization.


# 29. Heap

A binary min-heap maintains:

```text
parent <= children
```

Example:

```text
        2
      /   \
     5     7
    / \
   8   9
```

The heap root contains the minimum.

A heap is **not** a sorted sequence.

It only maintains the priority invariant required for efficient root selection.


# 30. Array-Backed Heap

For zero-based indexing:

```text
parent(i) = floor((i - 1) / 2)
left(i)   = 2i + 1
right(i)  = 2i + 2
```

Because a complete binary tree fills levels in order, the structure maps compactly to an array.

This avoids explicit:

```text
left
right
parent
```

references per node.


# 31. Heap Insert and Remove

Insert:

```text
append at end
↓
sift up
```

Remove minimum:

```text
save root
↓
move final item to root
↓
sift down
```

The invariant drives both algorithms.

Before writing code, state the invariant:

```text
every parent is <= each existing child
```

Then write mutations that restore it.


# 32. Priority Queue

A priority queue is an abstraction:

```text
insert(item, priority)
peek best priority
remove best priority
```

A heap is a common representation.

But alternative representations can be sensible for different workloads:

```text
sorted array
balanced tree
bucketed priorities
specialized scheduler
```

Choose based on:

```text
priority range
insert frequency
extract frequency
required stability
update-priority frequency
```


# 33. Binary Trees

A binary tree limits each node to at most two children.

Example:

```text
        A
      /   \
     B     C
    / \
   D   E
```

A binary tree is not necessarily ordered.

Its primary property is structural:

```text
at most two children
```


# 34. Binary Search Trees

A binary search tree adds an ordering invariant.

One common policy:

```text
left values < node value < right values
```

Duplicate handling must be specified.

Search follows:

```text
target < current → left
target > current → right
target === current → found
```

Performance depends strongly on tree shape.


# 35. Balanced Trees

A sorted tree can degenerate:

```text
1
 \
  2
   \
    3
```

Balanced tree families such as:

```text
AVL
Red-Black
B-Tree
B+ Tree
```

maintain stronger shape guarantees.

B-tree families are especially important in storage/index systems because their branching structure works well with block-oriented storage.


# 36. Trie

A trie stores strings through shared prefixes.

For:

```text
cat
car
cart
```

the `ca` prefix can be shared.

A node can contain:

```js
class TrieNode {
  constructor() {
    this.children = new Map();
    this.end = false;
  }
}
```

Useful for:

- autocomplete;
- prefix filtering;
- dictionaries;
- routing;
- lexical indexes.


# 37. Trie Invariant

For every node:

```text
children maps one edge label to one child
```

and:

```text
end === true
```

means:

```text
a complete stored key ends here
```

A node can be both:

```text
prefix
```

and:

```text
complete word
```

For example:

```text
car
cart
```

means the `r` node is both a completed key and a prefix of `cart`.


# 38. Graphs

A graph models:

```text
vertices
+
edges
```

Example:

```text
A ── B
│    │
C ── D
```

Graphs model:

- service dependencies;
- routes;
- social relationships;
- workflows;
- state transitions;
- package dependencies;
- networks.


# 39. Directed Graphs

A directed edge has orientation:

```text
A → B
```

which does not imply:

```text
B → A
```

Directed graphs are natural for:

```text
dependency
causal flow
state transitions
build order
```

Always state whether your graph permits:

```text
self-loops
parallel edges
cycles
```


# 40. Weighted Graphs

An edge may carry a weight:

```text
A --5--> B
```

where weight can represent:

```text
cost
distance
latency
risk
capacity
```

Once weights exist, algorithms must preserve them in the representation.

Do not reduce a weighted graph to a simple adjacency set unless the weight is intentionally discarded.


# 41. Adjacency Matrix

An adjacency matrix stores pair relationships:

```text
    A B C
A [ 0 1 0 ]
B [ 1 0 1 ]
C [ 0 1 0 ]
```

Strength:

```text
direct edge existence lookup
```

Cost:

```text
space for many possible pairs
```

It is attractive for dense graphs or algorithms that naturally operate on all vertex pairs.


# 42. Adjacency List

An adjacency list stores neighbors that actually exist.

```js
const graph = new Map([
  ["A", new Set(["B", "C"])],
  ["B", new Set(["A"])],
  ["C", new Set(["A"])],
]);
```

This is often suitable for sparse graphs.

Using `Set` here explicitly encodes:

```text
duplicate neighbor not allowed
```

If parallel edges are valid, choose a representation that preserves them.


# 43. Array vs Map vs Set

Choose Array when:

```text
sequence
position
batch iteration
```

Choose Map when:

```text
key → value
exact association
```

Choose Set when:

```text
membership
uniqueness
```

A useful review question is:

> “What is the primary question this collection answers?”

That question often exposes the correct abstraction.


# 44. Specialized Representations

Sometimes domain constraints justify specialized structures.

Examples:

```text
dense integer membership → bitset
millions of fixed-width numbers → typed array
bounded FIFO → ring buffer
prefix search → trie
priority extraction → heap
```

Specialization can reduce:

```text
memory
branching
allocation
```

but adds:

```text
implementation complexity
```

Use it when the workload and domain justify it.


# 45. Bitsets

A bitset represents Boolean state compactly.

Conceptually:

```text
index: 0 1 2 3 4 5 6 7
bits:  1 0 1 1 0 0 1 0
```

This can be much more compact than a general collection of integer IDs when the universe is dense and bounded.

A JavaScript implementation can use typed arrays and bit operations.

The domain assumption is essential.


# 46. Typed Arrays

Examples:

```js
const bytes = new Uint8Array(1024);
const values = new Int32Array(1_000_000);
```

Use cases include:

- binary protocols;
- file buffers;
- graphics;
- numeric workloads;
- low-level data interchange.

They provide a different representation model from ordinary JavaScript arrays and are particularly useful when the element type is known.


# 47. Composite Data Structures

Real production designs frequently combine structures.

Examples:

```text
Map + doubly linked list
Map + Heap
Set + Queue
Trie + metadata map
Graph + priority queue
```

This is often more powerful than searching for a single magical structure.

Example LRU:

```text
Map:
key → node

Doubly linked list:
MRU ⇄ ... ⇄ LRU
```

Lookup and recency updates can then be handled by different components.


# 48. LRU Cache

A common LRU design:

```text
Map:
key → list node

List:
front = most recently used
back  = least recently used
```

On `get(key)`:

```text
lookup node
move node to front
return value
```

On eviction:

```text
remove back node
delete key from Map
```

The structure is a composition of semantic tools.

Do not claim every LRU must use exactly this design; the workload may permit other representations.


# 49. Why Map Alone Is Not an LRU

`Map` has insertion-order-aware iteration semantics, but LRU requires:

```text
recency changes after access
```

An implementation must deliberately update recency when a key is used.

Simply calling:

```js
cache.get(key)
```

does not universally mean:

```text
key becomes newest
```

An LRU policy is an application-level behavior that must be implemented.


# 50. Memory and Garbage Collection

A data structure can be logically small but retain large object graphs.

Example:

```js
const cache = new Map();

for (const item of stream) {
  cache.set(item.id, item);
}
```

Without eviction:

```text
more input
↓
more reachable entries
↓
more memory
```

JavaScript's GC collects unreachable objects, not logically unused objects that are still reachable through your data structure.

Therefore lifecycle is part of data-structure design.


# 51. Locality and Allocation

Conceptually:

```text
nearby storage
[A][B][C][D]

pointer chasing
A → ... → B → ... → C
```

Modern hardware can benefit from locality.

In JavaScript, engines abstract physical layout but do not eliminate its consequences.

Linked-node structures can introduce:

```text
more allocations
more references
more pointer-like traversal
```

Arrays and typed arrays can often exploit more compact layouts for the right workload.


# 52. Security Considerations

Data structures can become attack surfaces.

Threats include:

```text
unbounded memory growth
pathological graph traversal
huge fan-out
very deep nesting
collision-heavy custom hashing
resource exhaustion
```

For untrusted data consider:

```text
size limits
depth limits
operation budgets
timeouts
cycle detection
eviction
```

Security is not separate from representation.


# 53. Graph Traversal Safety

A graph can contain cycles:

```text
A → B → C → A
```

A traversal must track visited state.

Typical model:

```js
const visited = new Set();
```

Then:

```text
if visited:
    do not expand again
else:
    mark visited
    expand neighbors
```

For hostile input, also consider:

```text
depth limit
node budget
edge budget
```

Avoid assuming recursive traversal is safe for arbitrarily deep graphs.


# 54. Serialization and Boundaries

Before choosing a custom structure, ask whether it must cross:

```text
process boundary
worker boundary
network boundary
storage boundary
structured-clone boundary
```

A custom linked structure can be awkward to serialize.

A stable wire representation such as:

```json
[
  {"id":"A","next":"B"},
  {"id":"B","next":null}
]
```

may be better than trying to transmit implementation objects directly.

Separate internal representation from external protocol.


# 55. Immutability and Persistent Structures

Mutation-oriented structures update in place:

```text
queue.enqueue()
map.set()
heap.add()
```

Persistent structures preserve previous versions through structural sharing.

Conceptually:

```text
version 1
   \
    shared nodes
       \
      version 2
```

This can improve:

```text
history
state reasoning
safe sharing
```

but may increase:

```text
allocation
indirection
implementation complexity
```

Choose according to application semantics.


# 56. Workload-First Selection

Define the workload before selecting the structure.

Record:

```text
read frequency
write frequency
delete frequency
front operations
back operations
lookup frequency
ordering requirement
growth limit
mutation pattern
```

Example:

```text
90% exact-key reads
9% updates
1% deletes
```

and:

```text
90% FIFO operations
```

are radically different workloads even if both store one million objects.


# 57. Production Example — Job Queue

Requirement:

```text
FIFO execution
cancel by job ID
100,000+ pending jobs
```

A plain array provides:

```text
enqueue
```

but cancellation by ID requires search.

Potential composition:

```text
Queue/deque → execution order
Map         → job ID lookup
state flag  → cancellation
```

Cancellation can be eager or lazy.

The decision depends on:

```text
cancellation frequency
memory budget
latency requirement
complexity tolerance
```


# 58. Production Example — Priority Scheduler

Requirement:

```text
highest-priority eligible job first
cancel by ID
```

Potential architecture:

```text
Map  → ID lookup
Heap → priority ordering
Set  → cancellation or state
```

On extraction:

```text
while root is cancelled:
    remove it
```

This lazy approach may be simpler than arbitrary heap deletion.

Again, design choices depend on workload.


# 59. Production Example — Autocomplete

Requirement:

```text
return words matching a prefix
```

Candidates:

```text
Trie
sorted array + binary search
search index
```

The correct choice depends on:

```text
dataset size
query volume
update rate
memory budget
prefix distribution
```

A trie is conceptually natural but can consume significant node overhead if implemented naively in JavaScript objects.


# 60. Production Example — Dependency Graph

Requirement:

```text
represent package dependencies
detect cycles
find transitive dependencies
derive build order
```

Use a graph representation.

Common tools:

```text
Map → adjacency
Set → visited
stack/state map → cycle detection
```

Topological ordering requires a DAG.

If cycles are allowed, topological ordering cannot produce a valid total dependency order over the cyclic subgraph.


# 61. Production Example — Recent Events

Requirement:

```text
retain latest 50,000 events
append constantly
evict oldest
inspect recent entries
```

Candidates:

```text
Array with head
Deque
Ring buffer
```

Because the maximum size is known, a ring buffer is often attractive.

The principal question is:

```text
What is the simplest representation that satisfies the required operations and memory bound?
```


# 62. Implementation Progression

For every structure use:

### Stage 1 — Guided
Follow a reference implementation.

### Stage 2 — Partially Guided
Write the API and invariants first.

### Stage 3 — No Reference
Implement from memory.

### Stage 4 — Edge Hardened
Test empty, singleton, duplicates, boundaries, and invalid states.

### Stage 5 — Production Grade
Add:

```text
tests
benchmarks
metrics
documentation
resource limits
```


# 63. Implementation — Stack

```js
class Stack {
  #items = [];

  push(value) {
    this.#items.push(value);
  }

  pop() {
    return this.#items.pop();
  }

  peek() {
    return this.#items.at(-1);
  }

  get size() {
    return this.#items.length;
  }
}
```

Invariant:

```text
top = #items[#items.length - 1]
```

Test:

```js
const stack = new Stack();

stack.push("A");
stack.push("B");
stack.push("C");

console.log(stack.pop());
console.log(stack.pop());
```

Expected:

```text
C
B
```


# 64. Implementation — Singly Linked List

```js
class ListNode {
  constructor(value, next = null) {
    this.value = value;
    this.next = next;
  }
}

class SinglyLinkedList {
  #head = null;
  #tail = null;
  #size = 0;

  append(value) {
    const node = new ListNode(value);

    if (this.#head === null) {
      this.#head = node;
      this.#tail = node;
    } else {
      this.#tail.next = node;
      this.#tail = node;
    }

    this.#size++;
  }

  get size() {
    return this.#size;
  }
}
```

Core invariants:

```text
empty:
head === null
tail === null
size === 0

non-empty:
head !== null
tail !== null
tail.next === null
size equals reachable node count
```


# 65. Implementation — Min Heap

```js
class MinHeap {
  #items = [];

  #parent(i) {
    return Math.floor((i - 1) / 2);
  }

  #left(i) {
    return i * 2 + 1;
  }

  #right(i) {
    return i * 2 + 2;
  }

  #swap(a, b) {
    [this.#items[a], this.#items[b]] =
      [this.#items[b], this.#items[a]];
  }

  add(value) {
    this.#items.push(value);

    let i = this.#items.length - 1;

    while (i > 0) {
      const p = this.#parent(i);

      if (this.#items[p] <= this.#items[i]) break;

      this.#swap(p, i);
      i = p;
    }
  }

  peek() {
    return this.#items[0];
  }

  remove() {
    if (this.#items.length === 0) {
      return undefined;
    }

    if (this.#items.length === 1) {
      return this.#items.pop();
    }

    const minimum = this.#items[0];
    this.#items[0] = this.#items.pop();

    let i = 0;

    while (true) {
      const left = this.#left(i);
      const right = this.#right(i);
      let smallest = i;

      if (
        left < this.#items.length &&
        this.#items[left] < this.#items[smallest]
      ) {
        smallest = left;
      }

      if (
        right < this.#items.length &&
        this.#items[right] < this.#items[smallest]
      ) {
        smallest = right;
      }

      if (smallest === i) break;

      this.#swap(i, smallest);
      i = smallest;
    }

    return minimum;
  }
}
```

Invariant:

```text
for every i:
items[i] <= items[left(i)] when left exists
items[i] <= items[right(i)] when right exists
```


# 66. Implementation — Educational Hash Map

The following design is intentionally educational.

```js
class Entry {
  constructor(key, value, next = null) {
    this.key = key;
    this.value = value;
    this.next = next;
  }
}
```

Conceptual architecture:

```text
buckets[]
  ↓
linked entries
```

A simplified string hash:

```js
function hashString(key) {
  let hash = 0;

  for (let i = 0; i < key.length; i++) {
    hash = (hash * 31 + key.charCodeAt(i)) >>> 0;
  }

  return hash;
}
```

This is not a universal production hash strategy.

A real implementation must define:

```text
key equality
collision handling
load factor
resize policy
delete behavior
iteration behavior
```


# 67. Hash Map Invariants

A custom hash map should preserve:

```text
every logical key maps to one logical value
no entry becomes unreachable accidentally
resize preserves all mappings
delete removes exactly the requested key
size equals the number of logical entries
```

A useful testing pattern:

```text
insert N
read all N
resize
read all N again
delete half
read remaining
```

Resize bugs often appear only after the table crosses its capacity threshold.


# 68. Implementation — Trie

```js
class TrieNode {
  constructor() {
    this.children = new Map();
    this.end = false;
  }
}

class Trie {
  #root = new TrieNode();

  add(word) {
    let node = this.#root;

    for (const char of word) {
      if (!node.children.has(char)) {
        node.children.set(char, new TrieNode());
      }

      node = node.children.get(char);
    }

    node.end = true;
  }

  has(word) {
    let node = this.#root;

    for (const char of word) {
      node = node.children.get(char);
      if (!node) return false;
    }

    return node.end;
  }
}
```

The key invariant is that the `end` marker records complete keys independently of prefix existence.


# 69. Implementation — Graph

A directed graph using adjacency sets:

```js
class Graph {
  #adjacency = new Map();

  addVertex(vertex) {
    if (!this.#adjacency.has(vertex)) {
      this.#adjacency.set(vertex, new Set());
    }
  }

  addEdge(from, to) {
    this.addVertex(from);
    this.addVertex(to);

    this.#adjacency.get(from).add(to);
  }

  neighbors(vertex) {
    return this.#adjacency.get(vertex) ?? new Set();
  }
}
```

The representation explicitly disallows duplicate outgoing neighbors.

For weighted graphs, replace the neighbor representation with one that retains weight.


# 70. Debugging by Invariant

When a structure misbehaves:

```text
1. state the invariant
2. inspect the first state where it becomes false
3. identify the mutation that broke it
4. fix the mutation
5. add a regression test
```

Do not start by changing random lines.

Example heap bug:

```text
parent <= child
```

fails only at a deep node.

The bug likely occurs in:

```text
sift up
or
sift down
```

Trace those operations rather than the entire program.


# 71. Debugging Exercise — Queue

Given:

```js
queue.enqueue("A");
queue.enqueue("B");
queue.enqueue("C");
```

The first three removals must be:

```text
A
B
C
```

If the output is:

```text
C
B
A
```

you implemented LIFO semantics.

If the output is:

```text
A
C
B
```

inspect the removal/indexing logic.

Always test:

```text
empty
singleton
two values
many values
enqueue after full drain
```


# 72. Debugging Exercise — Linked List

A list reports:

```text
size = 4
```

but traversal reaches:

```text
3 nodes
```

Likely causes:

```text
size increment without link update
broken next pointer
incorrect tail mutation
```

The invariant tells you what to inspect.

Add a helper in tests:

```js
assertReachableNodeCount(list, list.size);
```

for development builds.


# 73. Debugging Exercise — Heap

Suppose:

```text
[1, 7, 5, 9, 4]
```

is claimed to be a min-heap.

Check parents:

```text
1 <= 7   true
1 <= 5   true
7 <= 9   true
7 <= 4   false
```

Therefore the invariant is broken.

Do not assume the problem is the last inserted element; inspect the sift operation that produced the invalid parent/child relationship.


# 74. Debugging Exercise — Hash Map

A custom map works until resize.

Symptoms:

```text
get(existingKey) → undefined
```

Likely issue:

```text
entry retained under old bucket index
```

After resize, bucket placement must be recomputed according to the new capacity.

Regression test:

```text
insert enough items to force multiple resizes
verify every key after every resize
```


# 75. Debugging Exercise — Trie

The trie correctly recognizes:

```text
cart
```

but not:

```text
car
```

when both are inserted.

Likely issue:

```text
the node for "car" has no end marker
```

A prefix is not automatically a complete key.

Test:

```text
add("car")
add("cart")

has("car")  → true
has("cart") → true
has("ca")   → false
```


# 76. Performance Considerations

Do not judge structures only from textbook labels.

Measure:

```text
throughput
latency distribution
allocation rate
memory
GC behavior
```

Compare realistic workloads rather than isolated operations.

For example:

```text
push 1,000,000 items
remove all
```

is a meaningful queue benchmark.

So is:

```text
mixed enqueue/dequeue
```

with realistic burst patterns.


# 77. Benchmarking Discipline

Record:

```text
runtime version
hardware
OS
input size
operation mix
warm-up strategy
measurement duration
```

Avoid:

```text
single run
tiny dataset
uncontrolled console output
comparing cold code to warm code
```

A benchmark that favors an artificial workload is evidence only for that workload.


# 78. Memory Benchmarking

For Node.js, useful measurements can include:

```js
process.memoryUsage()
```

Relevant fields include concepts such as:

```text
heapUsed
heapTotal
rss
external
arrayBuffers
```

Interpret them in the context of the runtime.

A data structure that is “fast” but doubles resident memory may be a poor production choice.


# 79. Array vs Linked List — Principal Comparison

### Array

Strengths:

```text
simple API
indexed access
compact sequence semantics
often good locality
```

Weaknesses:

```text
front/middle mutation
possible resizing
```

### Linked list

Strengths:

```text
local node insertion/removal
stable node identity when represented explicitly
```

Weaknesses:

```text
traversal
allocation
memory overhead
locality
GC pressure
```

Decision:

```text
measure the real workload
```


# 80. Map vs Object — Principal Comparison

Object is natural for:

```text
record-shaped data
fixed named fields
serialization-oriented structures
```

Map is natural for:

```text
dynamic key/value collections
arbitrary keys
collection operations
```

Do not use performance folklore as the first selection criterion.

Use semantic fit first, then benchmark when performance is materially important.


# 81. Heap vs Sorted Array

If you repeatedly:

```text
insert
extract minimum
```

a heap is often a natural candidate.

If data is:

```text
mostly static
frequent ordered traversal
rare updates
```

a sorted representation may be simpler.

Again:

```text
operation mix
→ structure
```


# 82. Tree vs Graph

A tree is a constrained graph with hierarchical structure.

Graph representations are more general.

Use a tree when the domain guarantees hierarchy such as:

```text
filesystem-like structure
AST
organizational hierarchy
```

Use a graph when relationships may contain:

```text
multiple parents
cycles
cross-links
```


# 83. Security and Resource Exhaustion

A custom structure exposed to user-controlled data needs explicit budgets.

Examples:

```text
maximum collection size
maximum graph depth
maximum graph nodes
maximum edges
maximum queue age
maximum cache size
```

Without limits:

```text
valid feature
→ unbounded input
→ unbounded memory/work
→ service degradation
```

This is especially important for graph and indexing structures.


# 84. API Design and Encapsulation

Expose behavior:

```js
cache.get(key)
cache.set(key, value)
queue.enqueue(job)
queue.dequeue()
```

rather than internal arrays:

```js
cache.nodes
queue.items
```

Private fields, module boundaries, and narrow interfaces preserve invariants and allow future representation changes.


# 85. Code Review Exercise

Review:

```js
class JobQueue {
  constructor() {
    this.jobs = [];
  }

  add(job) {
    this.jobs.push(job);
  }

  cancel(id) {
    const index = this.jobs.findIndex(job => job.id === id);

    if (index !== -1) {
      this.jobs.splice(index, 1);
    }
  }

  next() {
    return this.jobs.shift();
  }
}
```

Requirements:

```text
100,000+ jobs
frequent cancellation
FIFO execution
```

Identify concerns:

```text
O(n)-style cancellation search
front-removal cost
repeated array shifts
allocation/copy behavior
lack of explicit bounds
no state model for cancellation races
no observability
```

Possible redesign:

```text
Map for ID lookup
+
deque/ring buffer for order
+
lazy cancellation state
```

but verify the exact workload before committing.


# 86. Interview Questions — Foundation

1. What is a data structure?
2. What is an ADT?
3. What is a representation invariant?
4. Why are arrays useful for sequences?
5. Why can repeated `shift()` be expensive?
6. What is LIFO?
7. What is FIFO?
8. What is a deque?
9. What is a hash table?
10. What is a heap?


# 87. Interview Questions — Intermediate

11. Why are append operations on dynamic arrays often amortized efficient?
12. What is a sparse array?
13. Why are linked lists not automatically faster for insertion?
14. What is a hash collision?
15. What is load factor?
16. Why resize a hash table?
17. Why use Map rather than an object for dynamic associations?
18. What does a min-heap guarantee?
19. How is a binary heap represented in an array?
20. What problem does a trie specialize in?


# 88. Interview Questions — Advanced

21. Design a queue for millions of messages.
22. Design an LRU cache.
23. Design an autocomplete system.
24. Design a priority scheduler.
25. Design a dependency graph.
26. How would you handle cancellation efficiently?
27. How would you validate heap invariants?
28. How would you debug a custom hash-map resize bug?
29. How do locality and GC affect linked-list performance?
30. How would you represent millions of dense integers?


# 89. Interview Questions — Principal

31. A team proposes a custom hash map in a production Node service. How do you review it?
32. A linked-list implementation is theoretically efficient but p99 latency worsens. Why?
33. How do you choose between deque and ring buffer?
34. When is a theoretically worse data structure the better production choice?
35. How do you prevent an in-memory structure from becoming a reliability risk?
36. How do data-structure choices interact with serialization boundaries?
37. How would you benchmark structures without overfitting to a microbenchmark?
38. How would you design a bounded cache under a strict memory budget?
39. How would you make graph traversal safe for hostile input?
40. How should a public API isolate its internal data structure?


# 90. Predict-the-Output Exercises

### Exercise 1

```js
const a = [];
a[2] = "x";

console.log(a.length);
console.log(0 in a);
console.log(2 in a);
```

Predict before running.

---

### Exercise 2

```js
const a = {};
const b = {};

const map = new Map([[a, 1]]);

console.log(map.get(a));
console.log(map.get(b));
```

---

### Exercise 3

```js
const set = new Set([1, 1, 2, 3, 3]);
console.log(set.size);
```

---

### Exercise 4

```js
const values = ["A", "B", "C"];

console.log(values.shift());
console.log(values.pop());
console.log(values);
```

---

### Exercise 5

```js
const stack = [];

stack.push("A");
stack.push("B");
stack.push("C");

console.log(stack.pop());
console.log(stack.pop());
```


# 91. Invariant Exercises

Write the invariant before implementing:

### Queue
Define:

```text
head
tail
size
```

### Ring buffer
Define:

```text
capacity
head
tail
size
```

### Heap
Define the parent/child ordering rule.

### Doubly linked list
Define neighbor consistency.

### Trie
Define the meaning of the `end` flag.

### Hash map
Define what must remain true after resize.


# 92. Mastery Exercises

Implement from scratch:

```text
DynamicArray
Stack
Queue
Deque
RingBuffer
SinglyLinkedList
DoublyLinkedList
HashMap
MinHeap
PriorityQueue
BST
Trie
Graph
```

For each, submit:

```text
API
representation
invariants
tests
edge cases
benchmark
trade-off note
```


# 93. Mastery Exercise — Structure Selection

Choose and defend a structure for:

```text
A. browser history
B. autocomplete
C. unique user IDs
D. recent event buffer
E. service dependency graph
F. exact key lookup
G. dense integer membership
H. priority scheduler
```

Your answer should include:

```text
required operations
candidate structures
chosen structure
why
what would make you change your decision
```


# 94. Mastery Exercise — Benchmark

Benchmark:

```text
Array + shift
head-index queue
ring buffer
linked-list queue
```

At:

```text
1,000
100,000
1,000,000
10,000,000
```

operations.

Record:

```text
throughput
p50
p95
p99 where meaningful
memory
GC observations
```

Then write a conclusion that separates:

```text
measured facts
from
general assumptions
```


# 95. Mastery Exercise — LRU Cache

Implement:

```js
class LRUCache {
  get(key) {}
  set(key, value) {}
  delete(key) {}
}
```

Requirements:

```text
bounded capacity
lookup
recency update
eviction
```

Then answer:

```text
Why this representation?
What is retained in memory?
How is eviction triggered?
What happens when capacity = 0?
What happens when an existing key is updated?
```


# 96. Track A — Core Theory

Study without code:

```text
ADT
representation
invariant
contiguous storage
linked storage
hashing
load factor
priority
tree ordering
prefix indexing
graph relationships
memory retention
```

You should be able to explain each from first principles.


# 97. Track B — Implementation

Build:

```text
guided
→ partially guided
→ no-reference
→ edge-case hardened
→ production-grade
```

At each stage, preserve the invariant explicitly.

Do not move to the next stage merely because the happy-path test passes.


# 98. Track C — Interview / Reasoning

Practice:

```text
predict
compare
debug
benchmark
choose
defend
```

The goal is not to recite:

```text
"X is O(1)"
```

The goal is to state:

```text
under these assumptions
for these operations
this representation is appropriate
because...
```


# 99. Principal Decision Framework

Evaluate each structure across:

| Dimension | Question |
|---|---|
| Correctness | Does it preserve required semantics and invariants? |
| Performance | Which operations dominate? |
| Memory | What is the per-entry and total overhead? |
| Security | Can untrusted input cause pathological cost? |
| Reliability | What happens at empty/full/boundary states? |
| Maintainability | Can engineers reason about the implementation? |
| Scalability | What happens as data grows? |
| Observability | Can size, pressure, and failures be measured? |
| Developer Experience | Is the API intuitive? |
| Operational Complexity | Does it need compaction, eviction, or rebalancing? |
| Future Change | Can the representation change behind the API? |


# 100. Production Checklist

```text
[ ] abstract behavior defined
[ ] dominant operations identified
[ ] workload measured or estimated
[ ] ordering requirement stated
[ ] key identity stated
[ ] bounded/unbounded growth stated
[ ] memory budget stated
[ ] invariants documented
[ ] empty/full behavior specified
[ ] duplicate behavior specified
[ ] deletion behavior specified
[ ] hostile-input behavior considered
[ ] serialization boundary considered
[ ] benchmark plan exists
[ ] implementation hidden behind API
[ ] monitoring exists for size/pressure
```


# 101. Failure Modes

### Wrong abstraction
Using Array for high-volume keyed lookup.

### Hidden quadratic behavior
Repeated `shift()` or repeated linear search.

### Memory retention
Old objects remain reachable through caches or stale buffers.

### Broken invariant
Size, head, tail, links, or heap order disagree.

### Overengineering
Custom structure replaces a standard built-in without measurable benefit.

### Underengineering
A simplistic structure fails at realistic scale.

### Wrong workload assumptions
A benchmark or textbook example does not match production.


# 102. Common Misconceptions

### “Linked lists are always faster for insertion.”
Only when the insertion position is already known and the surrounding workload makes the representation useful.

### “Arrays are literally C arrays.”
JavaScript array internals are engine-specific.

### “Map is always faster than object.”
Performance depends on runtime and workload. Semantic fit comes first.

### “Heap means sorted.”
A heap maintains a priority invariant, not total ordering.

### “Binary tree means BST.”
A BST adds an ordering invariant.

### “GC solves memory leaks.”
GC cannot collect reachable objects that your data structures still retain.

### “A trie is always best for autocomplete.”
Memory, update rate, dataset size, and alternatives matter.


# 103. Common Mistakes

```text
[ ] choose before defining operations
[ ] ignore memory
[ ] use shift() in a hot large queue
[ ] forget duplicate policy
[ ] forget cycle detection
[ ] recurse through arbitrarily deep hostile structures
[ ] write custom Map without justification
[ ] benchmark tiny datasets only
[ ] expose internal arrays publicly
[ ] ignore retention after dequeue
[ ] assume theoretical cost equals production latency
```


# 104. Spaced Retrieval Plan

### Day 0
Explain:

```text
Array
Stack
Queue
Deque
Map
Set
Heap
Tree
Trie
Graph
```

### Day 2
Implement:

```text
Stack
Queue
Heap
```

without references.

### Day 7
Implement an LRU cache.

### Day 14
Benchmark queue representations.

### Day 30
Design the data-structure layer for a production scheduler and defend the choice.


# 105. Concept Connections

## Depends On

- Values and Types
- Objects and Property Semantics
- Arrays
- Map and Set
- Typed Arrays
- Memory and GC
- Engine Architecture

## Builds Toward

- Chapter 72 — Complexity
- Chapter 73 — Core Algorithms
- Chapter 74 — Functional Programming
- Chapter 76 — Composition and Abstraction Design
- Chapter 77 — Design Patterns
- Chapter 78 — Production JavaScript Architecture
- Chapter 83 — Observability
- Chapter 84 — Reliability
- Chapter 85 — Performance
- Chapters 102–111 — Projects

## Related Concepts

```text
algorithms
complexity
memory layout
caching
scheduling
indexing
graphs
search
serialization
```

## Why This Chapter Matters Later

Data structures determine the shape of algorithmic work.

Therefore:

```text
representation
→ operation cost
→ algorithm choice
→ runtime behavior
→ system behavior
```


# 106. Completion Criteria

```text
[ ] Define ADT
[ ] Define representation invariant
[ ] Explain arrays
[ ] Explain dynamic growth
[ ] Explain sparse arrays
[ ] Explain Stack
[ ] Explain Queue
[ ] Explain Deque
[ ] Explain Ring Buffer
[ ] Explain Singly Linked List
[ ] Explain Doubly Linked List
[ ] Explain Hash Table
[ ] Explain Map
[ ] Explain Set
[ ] Explain Heap
[ ] Explain Priority Queue
[ ] Explain Binary Tree
[ ] Explain BST
[ ] Explain Trie
[ ] Explain Graph
[ ] Explain adjacency matrix
[ ] Explain adjacency list
[ ] Explain memory/locality trade-offs
[ ] Implement core structures
[ ] Test invariants
[ ] Benchmark realistic workloads
[ ] Design composite structures
[ ] Defend production decisions
```


# 107. Mastery Gate

### Understand
You can explain each major structure and its purpose.

### Explain
You can teach the representation and invariant.

### Predict
You can predict behavior before running code.

### Implement
You can build core structures without copying code.

### Debug
You can locate the first broken invariant.

### Apply
You can select a structure for a real workload.

### Compare
You can explain credible alternatives.

### Defend
You can justify the final choice across:

```text
correctness
performance
memory
security
reliability
maintainability
scalability
observability
```


# 108. Status

```text
[ ] Not Started
[~] In Progress
[?] Needs Revision
[+] Completed
[*] Mastered
```

Current status:

```text
[ ] Not Started
```

Reading alone does not mark mastery.


# Chapter 71 — Revision / Retrieval Record

| Date | Retrieval task | Result | Weak area | Next action |
|---|---|---|---|---|
| YYYY-MM-DD | Explain ADT vs implementation | ✅ / ❌ | ... | ... |
| YYYY-MM-DD | Implement Queue | ✅ / ❌ | ... | ... |
| YYYY-MM-DD | Explain heap invariant | ✅ / ❌ | ... | ... |
| YYYY-MM-DD | Implement HashMap | ✅ / ❌ | ... | ... |
| YYYY-MM-DD | Design LRU | ✅ / ❌ | ... | ... |
| YYYY-MM-DD | Choose structure for autocomplete | ✅ / ❌ | ... | ... |
| YYYY-MM-DD | Benchmark queue strategies | ✅ / ❌ | ... | ... |
| YYYY-MM-DD | Principal production review | ✅ / ❌ | ... | ... |

### Retrieval Prompts

```text
1. What is the difference between an ADT and a structure?
2. What invariant defines a min-heap?
3. Why can repeated shift() become expensive?
4. Why can linked lists lose to arrays?
5. What makes a bounded queue different from an unbounded queue?
6. When is Map the correct semantic choice?
7. What does a trie represent that a hash table does not?
8. How do you prevent graph traversal from looping forever?
9. What data is still reachable after a queue removes an item?
10. How would you defend a data-structure choice to a principal engineer?
```


# Chapter 71 — Canonical References and Source Discipline

## ECMAScript

- ECMAScript Language Specification  
  https://tc39.es/ecma262/

Use the specification for standardized language semantics.

## MDN

- Array  
  https://developer.mozilla.org/docs/Web/JavaScript/Reference/Global_Objects/Array
- Map  
  https://developer.mozilla.org/docs/Web/JavaScript/Reference/Global_Objects/Map
- Set  
  https://developer.mozilla.org/docs/Web/JavaScript/Reference/Global_Objects/Set
- Typed Arrays  
  https://developer.mozilla.org/docs/Web/JavaScript/Guide/Typed_arrays

## TypeScript

Current TypeScript documentation is useful for understanding type-level collection representations but should not be treated as a description of JavaScript engine internals. The current compiler options documentation exposes typed constructs and related configuration; it also demonstrates that runtime data structures and static representations are separate concerns. citeturn906469search0turn906469search1

## Node.js

- Process  
  https://nodejs.org/api/process.html
- Diagnostic Report  
  https://nodejs.org/api/report.html

Node's diagnostic report documentation describes production-oriented reports that can include JavaScript/native stack traces, heap statistics, resource information, and platform details. citeturn906469search7

## Source discipline

1. Use ECMAScript for language semantics.
2. Use platform documentation for runtime APIs.
3. Treat engine representations as implementation-specific.
4. State workload assumptions when discussing performance.
5. Benchmark realistic operations before making strong production claims.
6. Separate semantic behavior from optimization behavior.
7. Prefer standard built-ins unless custom representation is justified.
8. Treat memory lifecycle as part of the design.


# Chapter 71 — Completion Snapshot

## Fundamental Structures

```text
[ ] Array
[ ] Dynamic Array
[ ] Stack
[ ] Queue
[ ] Deque
[ ] Ring Buffer
[ ] Singly Linked List
[ ] Doubly Linked List
[ ] Hash Table
[ ] Map
[ ] Set
[ ] Heap
[ ] Priority Queue
[ ] Binary Tree
[ ] BST
[ ] Trie
[ ] Graph
```

## Reasoning

```text
[ ] ADT vs representation
[ ] invariants
[ ] workload analysis
[ ] memory overhead
[ ] locality
[ ] garbage collection
[ ] security
[ ] serialization
[ ] benchmarking
[ ] composition
```

## Production

```text
[ ] bounded queue
[ ] LRU cache
[ ] scheduler
[ ] autocomplete
[ ] dependency graph
[ ] graph safety
[ ] memory retention
```

## Mastery

```text
[ ] Understand
[ ] Explain
[ ] Predict
[ ] Implement
[ ] Debug
[ ] Apply
[ ] Compare
[ ] Defend
```


# Final Principal Perspective

A data structure is an engineering contract between:

```text
data
+
operations
+
invariants
+
cost
+
lifecycle
```

When designing a production system, ask:

```text
What behavior is required?
Which operation dominates?
How large can the structure become?
How is ordering defined?
What does equality mean?
What happens when it is empty?
What happens when it is full?
What happens during deletion?
What happens with hostile input?
How much memory does each entry retain?
Can we observe pressure?
Can we benchmark the real workload?
Can we change the implementation later?
```

The deepest lesson is:

> **Good data-structure decisions are workload decisions.**

The chain into the next chapter is:

```text
data structure
      ↓
operations
      ↓
cost model
      ↓
Chapter 72 — Complexity
```


# 109. Extended Retrieval Bank

### Retrieval Drill 1

Before looking at the answer, explain the chosen structure, its invariant, and the dominant operation for a workload with `250` logical elements.

```text
Required operation profile:
- 50% reads
- 25% inserts
- 15% deletes
- 10% iteration

Your task:
1. choose a candidate structure;
2. state the invariant;
3. identify the main risk;
4. state one credible alternative.
```

### Retrieval Drill 2

Before looking at the answer, explain the chosen structure, its invariant, and the dominant operation for a workload with `500` logical elements.

```text
Required operation profile:
- 50% reads
- 25% inserts
- 15% deletes
- 10% iteration

Your task:
1. choose a candidate structure;
2. state the invariant;
3. identify the main risk;
4. state one credible alternative.
```

### Retrieval Drill 3

Before looking at the answer, explain the chosen structure, its invariant, and the dominant operation for a workload with `750` logical elements.

```text
Required operation profile:
- 50% reads
- 25% inserts
- 15% deletes
- 10% iteration

Your task:
1. choose a candidate structure;
2. state the invariant;
3. identify the main risk;
4. state one credible alternative.
```

### Retrieval Drill 4

Before looking at the answer, explain the chosen structure, its invariant, and the dominant operation for a workload with `1000` logical elements.

```text
Required operation profile:
- 50% reads
- 25% inserts
- 15% deletes
- 10% iteration

Your task:
1. choose a candidate structure;
2. state the invariant;
3. identify the main risk;
4. state one credible alternative.
```

### Retrieval Drill 5

Before looking at the answer, explain the chosen structure, its invariant, and the dominant operation for a workload with `1250` logical elements.

```text
Required operation profile:
- 50% reads
- 25% inserts
- 15% deletes
- 10% iteration

Your task:
1. choose a candidate structure;
2. state the invariant;
3. identify the main risk;
4. state one credible alternative.
```

### Retrieval Drill 6

Before looking at the answer, explain the chosen structure, its invariant, and the dominant operation for a workload with `1500` logical elements.

```text
Required operation profile:
- 50% reads
- 25% inserts
- 15% deletes
- 10% iteration

Your task:
1. choose a candidate structure;
2. state the invariant;
3. identify the main risk;
4. state one credible alternative.
```

### Retrieval Drill 7

Before looking at the answer, explain the chosen structure, its invariant, and the dominant operation for a workload with `1750` logical elements.

```text
Required operation profile:
- 50% reads
- 25% inserts
- 15% deletes
- 10% iteration

Your task:
1. choose a candidate structure;
2. state the invariant;
3. identify the main risk;
4. state one credible alternative.
```

### Retrieval Drill 8

Before looking at the answer, explain the chosen structure, its invariant, and the dominant operation for a workload with `2000` logical elements.

```text
Required operation profile:
- 50% reads
- 25% inserts
- 15% deletes
- 10% iteration

Your task:
1. choose a candidate structure;
2. state the invariant;
3. identify the main risk;
4. state one credible alternative.
```

### Retrieval Drill 9

Before looking at the answer, explain the chosen structure, its invariant, and the dominant operation for a workload with `2250` logical elements.

```text
Required operation profile:
- 50% reads
- 25% inserts
- 15% deletes
- 10% iteration

Your task:
1. choose a candidate structure;
2. state the invariant;
3. identify the main risk;
4. state one credible alternative.
```

### Retrieval Drill 10

Before looking at the answer, explain the chosen structure, its invariant, and the dominant operation for a workload with `2500` logical elements.

```text
Required operation profile:
- 50% reads
- 25% inserts
- 15% deletes
- 10% iteration

Your task:
1. choose a candidate structure;
2. state the invariant;
3. identify the main risk;
4. state one credible alternative.
```

### Retrieval Drill 11

Before looking at the answer, explain the chosen structure, its invariant, and the dominant operation for a workload with `2750` logical elements.

```text
Required operation profile:
- 50% reads
- 25% inserts
- 15% deletes
- 10% iteration

Your task:
1. choose a candidate structure;
2. state the invariant;
3. identify the main risk;
4. state one credible alternative.
```

### Retrieval Drill 12

Before looking at the answer, explain the chosen structure, its invariant, and the dominant operation for a workload with `3000` logical elements.

```text
Required operation profile:
- 50% reads
- 25% inserts
- 15% deletes
- 10% iteration

Your task:
1. choose a candidate structure;
2. state the invariant;
3. identify the main risk;
4. state one credible alternative.
```

### Retrieval Drill 13

Before looking at the answer, explain the chosen structure, its invariant, and the dominant operation for a workload with `3250` logical elements.

```text
Required operation profile:
- 50% reads
- 25% inserts
- 15% deletes
- 10% iteration

Your task:
1. choose a candidate structure;
2. state the invariant;
3. identify the main risk;
4. state one credible alternative.
```

### Retrieval Drill 14

Before looking at the answer, explain the chosen structure, its invariant, and the dominant operation for a workload with `3500` logical elements.

```text
Required operation profile:
- 50% reads
- 25% inserts
- 15% deletes
- 10% iteration

Your task:
1. choose a candidate structure;
2. state the invariant;
3. identify the main risk;
4. state one credible alternative.
```

### Retrieval Drill 15

Before looking at the answer, explain the chosen structure, its invariant, and the dominant operation for a workload with `3750` logical elements.

```text
Required operation profile:
- 50% reads
- 25% inserts
- 15% deletes
- 10% iteration

Your task:
1. choose a candidate structure;
2. state the invariant;
3. identify the main risk;
4. state one credible alternative.
```

### Retrieval Drill 16

Before looking at the answer, explain the chosen structure, its invariant, and the dominant operation for a workload with `4000` logical elements.

```text
Required operation profile:
- 50% reads
- 25% inserts
- 15% deletes
- 10% iteration

Your task:
1. choose a candidate structure;
2. state the invariant;
3. identify the main risk;
4. state one credible alternative.
```

### Retrieval Drill 17

Before looking at the answer, explain the chosen structure, its invariant, and the dominant operation for a workload with `4250` logical elements.

```text
Required operation profile:
- 50% reads
- 25% inserts
- 15% deletes
- 10% iteration

Your task:
1. choose a candidate structure;
2. state the invariant;
3. identify the main risk;
4. state one credible alternative.
```

### Retrieval Drill 18

Before looking at the answer, explain the chosen structure, its invariant, and the dominant operation for a workload with `4500` logical elements.

```text
Required operation profile:
- 50% reads
- 25% inserts
- 15% deletes
- 10% iteration

Your task:
1. choose a candidate structure;
2. state the invariant;
3. identify the main risk;
4. state one credible alternative.
```

### Retrieval Drill 19

Before looking at the answer, explain the chosen structure, its invariant, and the dominant operation for a workload with `4750` logical elements.

```text
Required operation profile:
- 50% reads
- 25% inserts
- 15% deletes
- 10% iteration

Your task:
1. choose a candidate structure;
2. state the invariant;
3. identify the main risk;
4. state one credible alternative.
```

### Retrieval Drill 20

Before looking at the answer, explain the chosen structure, its invariant, and the dominant operation for a workload with `5000` logical elements.

```text
Required operation profile:
- 50% reads
- 25% inserts
- 15% deletes
- 10% iteration

Your task:
1. choose a candidate structure;
2. state the invariant;
3. identify the main risk;
4. state one credible alternative.
```

### Retrieval Drill 21

Before looking at the answer, explain the chosen structure, its invariant, and the dominant operation for a workload with `5250` logical elements.

```text
Required operation profile:
- 50% reads
- 25% inserts
- 15% deletes
- 10% iteration

Your task:
1. choose a candidate structure;
2. state the invariant;
3. identify the main risk;
4. state one credible alternative.
```

### Retrieval Drill 22

Before looking at the answer, explain the chosen structure, its invariant, and the dominant operation for a workload with `5500` logical elements.

```text
Required operation profile:
- 50% reads
- 25% inserts
- 15% deletes
- 10% iteration

Your task:
1. choose a candidate structure;
2. state the invariant;
3. identify the main risk;
4. state one credible alternative.
```

### Retrieval Drill 23

Before looking at the answer, explain the chosen structure, its invariant, and the dominant operation for a workload with `5750` logical elements.

```text
Required operation profile:
- 50% reads
- 25% inserts
- 15% deletes
- 10% iteration

Your task:
1. choose a candidate structure;
2. state the invariant;
3. identify the main risk;
4. state one credible alternative.
```

### Retrieval Drill 24

Before looking at the answer, explain the chosen structure, its invariant, and the dominant operation for a workload with `6000` logical elements.

```text
Required operation profile:
- 50% reads
- 25% inserts
- 15% deletes
- 10% iteration

Your task:
1. choose a candidate structure;
2. state the invariant;
3. identify the main risk;
4. state one credible alternative.
```

### Retrieval Drill 25

Before looking at the answer, explain the chosen structure, its invariant, and the dominant operation for a workload with `6250` logical elements.

```text
Required operation profile:
- 50% reads
- 25% inserts
- 15% deletes
- 10% iteration

Your task:
1. choose a candidate structure;
2. state the invariant;
3. identify the main risk;
4. state one credible alternative.
```

### Retrieval Drill 26

Before looking at the answer, explain the chosen structure, its invariant, and the dominant operation for a workload with `6500` logical elements.

```text
Required operation profile:
- 50% reads
- 25% inserts
- 15% deletes
- 10% iteration

Your task:
1. choose a candidate structure;
2. state the invariant;
3. identify the main risk;
4. state one credible alternative.
```

### Retrieval Drill 27

Before looking at the answer, explain the chosen structure, its invariant, and the dominant operation for a workload with `6750` logical elements.

```text
Required operation profile:
- 50% reads
- 25% inserts
- 15% deletes
- 10% iteration

Your task:
1. choose a candidate structure;
2. state the invariant;
3. identify the main risk;
4. state one credible alternative.
```

### Retrieval Drill 28

Before looking at the answer, explain the chosen structure, its invariant, and the dominant operation for a workload with `7000` logical elements.

```text
Required operation profile:
- 50% reads
- 25% inserts
- 15% deletes
- 10% iteration

Your task:
1. choose a candidate structure;
2. state the invariant;
3. identify the main risk;
4. state one credible alternative.
```

### Retrieval Drill 29

Before looking at the answer, explain the chosen structure, its invariant, and the dominant operation for a workload with `7250` logical elements.

```text
Required operation profile:
- 50% reads
- 25% inserts
- 15% deletes
- 10% iteration

Your task:
1. choose a candidate structure;
2. state the invariant;
3. identify the main risk;
4. state one credible alternative.
```

### Retrieval Drill 30

Before looking at the answer, explain the chosen structure, its invariant, and the dominant operation for a workload with `7500` logical elements.

```text
Required operation profile:
- 50% reads
- 25% inserts
- 15% deletes
- 10% iteration

Your task:
1. choose a candidate structure;
2. state the invariant;
3. identify the main risk;
4. state one credible alternative.
```

### Retrieval Drill 31

Before looking at the answer, explain the chosen structure, its invariant, and the dominant operation for a workload with `7750` logical elements.

```text
Required operation profile:
- 50% reads
- 25% inserts
- 15% deletes
- 10% iteration

Your task:
1. choose a candidate structure;
2. state the invariant;
3. identify the main risk;
4. state one credible alternative.
```

### Retrieval Drill 32

Before looking at the answer, explain the chosen structure, its invariant, and the dominant operation for a workload with `8000` logical elements.

```text
Required operation profile:
- 50% reads
- 25% inserts
- 15% deletes
- 10% iteration

Your task:
1. choose a candidate structure;
2. state the invariant;
3. identify the main risk;
4. state one credible alternative.
```

### Retrieval Drill 33

Before looking at the answer, explain the chosen structure, its invariant, and the dominant operation for a workload with `8250` logical elements.

```text
Required operation profile:
- 50% reads
- 25% inserts
- 15% deletes
- 10% iteration

Your task:
1. choose a candidate structure;
2. state the invariant;
3. identify the main risk;
4. state one credible alternative.
```

### Retrieval Drill 34

Before looking at the answer, explain the chosen structure, its invariant, and the dominant operation for a workload with `8500` logical elements.

```text
Required operation profile:
- 50% reads
- 25% inserts
- 15% deletes
- 10% iteration

Your task:
1. choose a candidate structure;
2. state the invariant;
3. identify the main risk;
4. state one credible alternative.
```

### Retrieval Drill 35

Before looking at the answer, explain the chosen structure, its invariant, and the dominant operation for a workload with `8750` logical elements.

```text
Required operation profile:
- 50% reads
- 25% inserts
- 15% deletes
- 10% iteration

Your task:
1. choose a candidate structure;
2. state the invariant;
3. identify the main risk;
4. state one credible alternative.
```

### Retrieval Drill 36

Before looking at the answer, explain the chosen structure, its invariant, and the dominant operation for a workload with `9000` logical elements.

```text
Required operation profile:
- 50% reads
- 25% inserts
- 15% deletes
- 10% iteration

Your task:
1. choose a candidate structure;
2. state the invariant;
3. identify the main risk;
4. state one credible alternative.
```

### Retrieval Drill 37

Before looking at the answer, explain the chosen structure, its invariant, and the dominant operation for a workload with `9250` logical elements.

```text
Required operation profile:
- 50% reads
- 25% inserts
- 15% deletes
- 10% iteration

Your task:
1. choose a candidate structure;
2. state the invariant;
3. identify the main risk;
4. state one credible alternative.
```

### Retrieval Drill 38

Before looking at the answer, explain the chosen structure, its invariant, and the dominant operation for a workload with `9500` logical elements.

```text
Required operation profile:
- 50% reads
- 25% inserts
- 15% deletes
- 10% iteration

Your task:
1. choose a candidate structure;
2. state the invariant;
3. identify the main risk;
4. state one credible alternative.
```

### Retrieval Drill 39

Before looking at the answer, explain the chosen structure, its invariant, and the dominant operation for a workload with `9750` logical elements.

```text
Required operation profile:
- 50% reads
- 25% inserts
- 15% deletes
- 10% iteration

Your task:
1. choose a candidate structure;
2. state the invariant;
3. identify the main risk;
4. state one credible alternative.
```

### Retrieval Drill 40

Before looking at the answer, explain the chosen structure, its invariant, and the dominant operation for a workload with `10000` logical elements.

```text
Required operation profile:
- 50% reads
- 25% inserts
- 15% deletes
- 10% iteration

Your task:
1. choose a candidate structure;
2. state the invariant;
3. identify the main risk;
4. state one credible alternative.
```

### Retrieval Drill 41

Before looking at the answer, explain the chosen structure, its invariant, and the dominant operation for a workload with `10250` logical elements.

```text
Required operation profile:
- 50% reads
- 25% inserts
- 15% deletes
- 10% iteration

Your task:
1. choose a candidate structure;
2. state the invariant;
3. identify the main risk;
4. state one credible alternative.
```

### Retrieval Drill 42

Before looking at the answer, explain the chosen structure, its invariant, and the dominant operation for a workload with `10500` logical elements.

```text
Required operation profile:
- 50% reads
- 25% inserts
- 15% deletes
- 10% iteration

Your task:
1. choose a candidate structure;
2. state the invariant;
3. identify the main risk;
4. state one credible alternative.
```

### Retrieval Drill 43

Before looking at the answer, explain the chosen structure, its invariant, and the dominant operation for a workload with `10750` logical elements.

```text
Required operation profile:
- 50% reads
- 25% inserts
- 15% deletes
- 10% iteration

Your task:
1. choose a candidate structure;
2. state the invariant;
3. identify the main risk;
4. state one credible alternative.
```

### Retrieval Drill 44

Before looking at the answer, explain the chosen structure, its invariant, and the dominant operation for a workload with `11000` logical elements.

```text
Required operation profile:
- 50% reads
- 25% inserts
- 15% deletes
- 10% iteration

Your task:
1. choose a candidate structure;
2. state the invariant;
3. identify the main risk;
4. state one credible alternative.
```