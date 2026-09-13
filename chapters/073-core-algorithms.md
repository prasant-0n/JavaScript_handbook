# Chapter 73 — Core Algorithms

> **Curriculum position:** Part XIII — Data Structures and Algorithms  
> **Previous chapter:** Chapter 72 — Complexity  
> **Next chapter:** Chapter 74 — Functional Programming  
> **Primary environment:** Modern JavaScript / TypeScript, Node.js, browsers, backend services, and production systems.

---

# Chapter Mission

Master the core algorithmic patterns that turn:

```text
data structures
+
complexity reasoning
```

into useful computation.

This chapter is not a catalog of code snippets.

The goal is to recognize the shape of a problem:

```text
search
sort
traverse
partition
count
aggregate
transform
optimize
schedule
connect
```

and choose an algorithm deliberately.

The central engineering loop is:

```text
problem
  ↓
constraints
  ↓
model
  ↓
data structure
  ↓
algorithm
  ↓
correctness argument
  ↓
complexity
  ↓
implementation
  ↓
benchmark
  ↓
production decision
```

A principal engineer should be able to answer:

```text
What exactly are we computing?
What assumptions hold?
What invariant makes the algorithm correct?
What is the worst case?
What is the expected case?
What data structure supports the algorithm?
What is the memory cost?
What happens at production scale?
What happens with hostile input?
What simpler alternative exists?
```

The principal-level goal is:

> **Recognize reusable algorithmic patterns, prove their correctness, quantify their cost, and choose them based on real constraints.**


# 1. Learning Objectives

By the end of this chapter you should be able to:

## Core patterns

- linear search;
- binary search;
- two pointers;
- sliding window;
- prefix sums;
- frequency counting;
- hashing-based lookup;
- sorting;
- stable versus unstable sorting as a conceptual distinction;
- divide and conquer;
- merge sort;
- quicksort;
- heap-based selection;
- breadth-first search;
- depth-first search;
- topological sorting;
- cycle detection;
- shortest-path foundations;
- dynamic programming fundamentals;
- greedy reasoning;
- backtracking;
- memoization;
- recursion-to-iteration transformations.

## Correctness

- state loop invariants;
- state recursive invariants;
- explain termination;
- reason about completeness;
- reason about duplicate handling;
- reason about boundary conditions.

## Complexity

- derive time complexity;
- derive auxiliary space;
- identify output size;
- explain expected versus worst case;
- identify hidden copying;
- identify repeated work;
- compare competing algorithms.

## Production

- choose algorithms according to constraints;
- bound attacker-controlled inputs;
- preserve tail latency;
- avoid event-loop blocking;
- control memory growth;
- choose iterative alternatives when recursion depth is unsafe;
- benchmark realistic workloads;
- design observability around algorithmic pressure.


# 2. Prerequisites

Recommended:

- Chapter 22 — Arrays
- Chapter 24 — Map and Set
- Chapter 25 — Iterables and Iterators
- Chapter 26 — Generators
- Chapter 27 — Typed Arrays
- Chapter 45 — Memory and Garbage Collection
- Chapter 47 — JavaScript Engine Architecture
- Chapter 71 — Fundamental Data Structures
- Chapter 72 — Complexity


# 3. What Is an Algorithm?

An algorithm is a finite, well-defined procedure for solving a class of problems.

It contains:

```text
inputs
state
operations
control flow
termination
output
correctness properties
```

Example:

```text
binary search
```

takes:

```text
sorted sequence
+
target
```

and produces:

```text
found index
or
not found
```

The implementation language is not the algorithm.

The same algorithm can be implemented in:

```text
JavaScript
TypeScript
Java
Rust
Python
C++
```


# 4. Algorithm vs Data Structure

A data structure organizes information.

An algorithm performs computation over that information.

Example:

```text
Map
+
lookup algorithm
```

or:

```text
Graph
+
BFS
```

or:

```text
Heap
+
priority scheduling algorithm
```

The two concepts are deeply coupled.

Good algorithm selection often begins with:

```text
choose representation
```


# 5. Problem Classification

Before coding, classify the problem.

Common shapes:

```text
exact lookup
ordered lookup
pair matching
contiguous range
frequency
streaming aggregation
sorting
tree traversal
graph traversal
dependency ordering
shortest path
optimization
enumeration
```

Recognition reduces unnecessary reinvention.


# 6. Constraint-First Thinking

Write down:

```text
n = input size
m = second input size
V = vertices
E = edges
k = key/range bound
```

Then:

```text
maximum n?
memory budget?
input sorted?
duplicates allowed?
data static or dynamic?
online or offline?
exact or approximate?
latency target?
```

Algorithm choice should follow constraints.


# 7. Correctness First

For every algorithm define:

```text
precondition
invariant
termination condition
postcondition
```

Example binary search:

```text
precondition:
array sorted according to comparator

invariant:
if target exists, it is inside [low, high]

termination:
low > high or target found

postcondition:
correct index or not-found result
```


# 8. Linear Search

Linear search scans sequentially.

```js
function linearSearch(values, target) {
  for (let i = 0; i < values.length; i++) {
    if (Object.is(values[i], target)) {
      return i;
    }
  }

  return -1;
}
```

Complexity:

```text
best  = O(1)
worst = O(n)
space = O(1) auxiliary
```

Use it when:

```text
data is unsorted
n is modest
one/few searches exist
building an index is not worth it
```


# 9. Binary Search

Binary search repeatedly halves a sorted search interval.

```js
function binarySearch(values, target) {
  let low = 0;
  let high = values.length - 1;

  while (low <= high) {
    const mid = low + Math.floor((high - low) / 2);
    const value = values[mid];

    if (value === target) return mid;

    if (value < target) {
      low = mid + 1;
    } else {
      high = mid - 1;
    }
  }

  return -1;
}
```

Typical complexity:

```text
O(log n)
```

but only because the ordering precondition exists.


# 10. Binary Search Invariant

At every iteration:

```text
if target exists,
it exists somewhere in [low, high]
```

When:

```text
values[mid] < target
```

everything at or before `mid` can be excluded under the sorted-order assumption.

When:

```text
values[mid] > target
```

everything at or after `mid` can be excluded.

The invariant is the proof.


# 11. Binary Search Edge Cases

Test:

```text
empty array
one element
target first
target last
target absent
duplicate target
two elements
even length
odd length
```

Decide whether duplicates return:

```text
any match
first match
last match
lower bound
upper bound
```

That is an API decision, not merely an implementation detail.


# 12. Lower Bound and Upper Bound

For sorted data, useful variants are:

```text
lower bound:
first index where value >= target

upper bound:
first index where value > target
```

These support:

```text
range counting
insertion points
duplicate intervals
```

A robust implementation focuses on the boundary invariant rather than simply finding an arbitrary equal value.


# 13. Two-Pointer Pattern

Two pointers often process a sequence in one pass.

Example:

```js
function reverseInPlace(values) {
  let left = 0;
  let right = values.length - 1;

  while (left < right) {
    [values[left], values[right]] =
      [values[right], values[left]];

    left++;
    right--;
  }
}
```

Complexity:

```text
O(n) time
O(1) auxiliary space
```

The pointers move monotonically.


# 14. Two-Pointer Pair Search

For sorted data, pair-sum reasoning can use:

```text
left = start
right = end
```

If:

```text
values[left] + values[right] < target
```

increase `left`.

If:

```text
sum > target
```

decrease `right`.

Sorting changes the problem structure enough to avoid testing every pair.


# 15. Sliding Window

Sliding window handles contiguous ranges.

Typical model:

```text
[left ........ right]
```

Move:

```text
right
```

to expand.

Move:

```text
left
```

to restore a condition.

Useful for:

```text
maximum subarray under a condition
longest valid substring
fixed-size windows
streaming metrics
```


# 16. Fixed-Size Sliding Window

Example:

```js
function maxWindowSum(values, k) {
  if (k <= 0 || k > values.length) {
    return undefined;
  }

  let sum = 0;

  for (let i = 0; i < k; i++) {
    sum += values[i];
  }

  let best = sum;

  for (let right = k; right < values.length; right++) {
    sum += values[right];
    sum -= values[right - k];
    best = Math.max(best, sum);
  }

  return best;
}
```

Naive approach:

```text
recompute every window
```

can be quadratic.

Sliding update reduces repeated work to linear time.


# 17. Variable-Size Sliding Window

Typical pattern:

```text
expand right
while constraint violated:
    move left
record best
```

The key is often:

```text
left never moves backward
```

so total pointer movement is bounded.

This is another example where nested control flow can still be linear.


# 18. Prefix Sums

Prefix sums precompute cumulative information.

For:

```text
values = [a, b, c, d]
```

prefix:

```text
[0, a, a+b, a+b+c, a+b+c+d]
```

A range sum becomes:

```text
prefix[r] - prefix[l]
```

after choosing the indexing convention.

Trade-off:

```text
precompute O(n)
memory O(n)
range query O(1)-style
```


# 19. Frequency Counting

A frequent transformation:

```text
repeated scan
→
frequency table
```

Example:

```js
function countValues(values) {
  const counts = new Map();

  for (const value of values) {
    counts.set(
      value,
      (counts.get(value) ?? 0) + 1
    );
  }

  return counts;
}
```

This converts repeated equality counting into one pass plus map operations under their normal expected-efficiency model.


# 20. Hashing as an Algorithmic Pattern

Hashing is often used to turn:

```text
search every previous item
```

into:

```text
check indexed membership
```

Example duplicate detection:

```js
function hasDuplicate(values) {
  const seen = new Set();

  for (const value of values) {
    if (seen.has(value)) return true;
    seen.add(value);
  }

  return false;
}
```

Typical expected:

```text
O(n)
```

with:

```text
O(n)
```

additional memory.


# 21. Sorting

Sorting transforms:

```text
unordered data
```

into:

```text
ordered data
```

This often enables:

```text
binary search
two pointers
grouping
deduplication
range queries
```

Sorting is therefore not just presentation logic.

It is often a structural preprocessing step.


# 22. Comparator Semantics

JavaScript `sort()` accepts a comparison function:

```js
values.sort((a, b) => a - b);
```

The comparator defines ordering.

Be careful with:

```text
NaN
mixed types
objects
locale rules
ascending/descending
stability requirements
```

Do not assume subtraction works for arbitrary values.


# 23. Stable Sorting

A stable sort preserves the relative ordering of equal-key elements.

Example:

```text
A: score 10
B: score 10
C: score 20
```

Sorting by score stably preserves:

```text
A before B
```

when both remain equal under the comparator.

Stability matters for multi-stage sorting and deterministic pipelines.

Modern ECMAScript specifies stable `Array.prototype.sort` behavior; check the target environment when compatibility across unusual runtimes matters.


# 24. Merge Sort

Merge sort uses divide and conquer:

```text
split
 ↓
sort left
sort right
 ↓
merge
```

Typical complexity:

```text
Θ(n log n)
```

Auxiliary memory is commonly:

```text
Θ(n)
```

depending on implementation.

Strengths:

```text
predictable O(n log n)
stable variants
```

Costs:

```text
extra memory
```


# 25. Merge Operation

Given sorted arrays:

```text
A = [1, 4, 9]
B = [2, 3, 8]
```

merge by maintaining:

```text
i
j
```

and repeatedly choosing the smaller next element.

Every element is emitted once.

Therefore merge work is:

```text
O(n + m)
```

for arrays of lengths `n` and `m`.


# 26. Quicksort

Quicksort chooses a pivot:

```text
partition
```

then recursively solves subproblems.

Average/expected behavior under suitable pivot behavior:

```text
O(n log n)
```

Worst case can be:

```text
O(n²)
```

The partition strategy and pivot selection matter.

In production, prefer the runtime's well-tested sorting implementation unless custom control is actually required.


# 27. Quicksort Failure Mode

Already-ordered or adversarial input can cause poor partitions for naive pivot strategies:

```text
n - 1
+
0
```

repeatedly.

This creates a deep recursion tree.

Use randomized/robust strategies or production library implementations when appropriate.


# 28. Heap-Based Selection

If only the top `k` items matter, fully sorting `n` items may be unnecessary.

A heap can maintain the best `k`:

```text
O(n log k)
```

under the typical heap model.

This can beat:

```text
O(n log n)
```

when:

```text
k << n
```

Example:

```text
top 100 errors
from 10,000,000 records
```


# 29. Divide and Conquer

General pattern:

```text
divide
 ↓
solve smaller instances
 ↓
combine
```

Analyze:

```text
subproblem count
subproblem size
combine work
```

Classic examples:

```text
binary search
merge sort
quicksort
divide-and-conquer geometry
```


# 30. Recursion Correctness

For recursive algorithms define:

```text
base case
recursive progress
subproblem correctness
combination correctness
```

A recursion that does not reduce toward a base case is not merely slow.

It is incorrect or nonterminating.


# 31. Recursion vs Iteration

Many algorithms have both forms.

Recursion can improve:

```text
clarity
tree/graph expression
divide-and-conquer representation
```

Iteration can improve:

```text
stack safety
explicit resource control
very deep traversal
```

For arbitrary-depth input in JavaScript, an explicit stack is often safer than unbounded recursion.


# 32. Breadth-First Search

BFS explores by distance layers:

```text
start
 ↓
neighbors
 ↓
neighbors of neighbors
```

Use a queue.

Example:

```js
function bfs(graph, start) {
  const queue = [start];
  let head = 0;
  const visited = new Set([start]);

  while (head < queue.length) {
    const node = queue[head++];

    for (const next of graph.neighbors(node)) {
      if (!visited.has(next)) {
        visited.add(next);
        queue.push(next);
      }
    }
  }

  return visited;
}
```

For adjacency lists:

```text
O(V + E)
```

under the standard model.


# 33. BFS for Unweighted Shortest Paths

In an unweighted graph, BFS explores nodes in nondecreasing edge distance from the start.

Therefore the first discovery of a node gives a shortest edge-count path from the source, assuming normal BFS traversal and unit edge weights.

This does not directly solve weighted shortest paths.


# 34. Depth-First Search

DFS explores as deeply as possible before backtracking.

Recursive model:

```js
function dfs(node) {
  if (visited.has(node)) return;
  visited.add(node);

  for (const next of graph.neighbors(node)) {
    dfs(next);
  }
}
```

For large/deep graphs, use an explicit stack when recursion depth can be problematic.


# 35. DFS With Explicit Stack

Iterative DFS:

```js
function dfs(graph, start) {
  const stack = [start];
  const visited = new Set();

  while (stack.length) {
    const node = stack.pop();

    if (visited.has(node)) continue;
    visited.add(node);

    for (const next of graph.neighbors(node)) {
      if (!visited.has(next)) {
        stack.push(next);
      }
    }
  }

  return visited;
}
```

This makes traversal depth an explicit data structure rather than the JavaScript call stack.


# 36. Cycle Detection in Directed Graphs

A directed DFS can track three states:

```text
unvisited
visiting
visited
```

If traversal encounters:

```text
visiting
```

then a back edge exists and a cycle has been found.

State is more informative than a simple `visited` set for this problem.


# 37. Topological Sorting

Topological order exists for a directed acyclic graph.

Kahn's algorithm:

```text
1. compute indegree
2. enqueue zero-indegree nodes
3. remove one
4. decrement neighbors
5. enqueue newly zero-indegree nodes
6. repeat
```

If processed count is less than `V`:

```text
cycle exists
```

Typical complexity:

```text
O(V + E)
```

with adjacency-list representation.


# 38. Graph Algorithm Selection

Use:

```text
BFS
→ unweighted shortest path / levels

DFS
→ reachability / components / structural analysis

topological sort
→ dependency ordering

Dijkstra-style approach
→ nonnegative weighted shortest path

priority queue
→ repeatedly select minimum/maximum candidate
```

Always verify the algorithm's assumptions.


# 39. Weighted Shortest Paths

For nonnegative edge weights, Dijkstra's algorithm repeatedly selects the currently closest unsettled vertex.

A priority queue makes this efficient.

Do not use Dijkstra blindly when negative-weight edges exist.

Algorithm preconditions are part of correctness.


# 40. Greedy Algorithms

A greedy algorithm makes a locally optimal choice.

This works only when the problem has the necessary structural property, such as an exchange argument or related greedy-choice property.

Greedy is not:

```text
"take the best-looking next step"
```

It requires proof or a known theorem that local choices lead to a global optimum.


# 41. Dynamic Programming

Dynamic programming applies when a problem has:

```text
overlapping subproblems
+
optimal substructure
```

Typical strategies:

```text
top-down memoization
bottom-up tabulation
```

The transformation is:

```text
repeated recursive computation
→
store previously solved states
```


# 42. Memoization

Example:

```js
function fib(n, memo = new Map()) {
  if (n < 2) return n;

  if (memo.has(n)) {
    return memo.get(n);
  }

  const result =
    fib(n - 1, memo) +
    fib(n - 2, memo);

  memo.set(n, result);
  return result;
}
```

The number of distinct states becomes linear in `n`.

But memory grows with state count.

Memoization is a space-time trade-off.


# 43. Bottom-Up Dynamic Programming

Instead of recursion:

```text
smallest states
 ↓
larger states
```

Example Fibonacci:

```js
function fib(n) {
  if (n < 2) return n;

  let prev = 0;
  let curr = 1;

  for (let i = 2; i <= n; i++) {
    [prev, curr] = [curr, prev + curr];
  }

  return curr;
}
```

Time:

```text
O(n)
```

Auxiliary space:

```text
O(1)
```

The recurrence only depends on the two previous states.


# 44. State Compression

If DP state `i` only depends on a constant number of previous states:

```text
full table
```

may be reduced to:

```text
rolling variables
```

This turns:

```text
O(n) memory
```

into:

```text
O(1)
```

when the required history can be compressed safely.


# 45. Backtracking

Backtracking explores candidate solutions and abandons partial candidates that cannot lead to valid solutions.

Pattern:

```text
choose
 ↓
recurse
 ↓
undo
```

Typical problems:

```text
permutations
combinations
constraint satisfaction
N-Queens
subset enumeration
```

Worst-case complexity is often exponential.


# 46. Backtracking Correctness

The key invariant:

```text
current partial state contains exactly the choices on the active recursion path
```

The undo step is essential:

```js
path.push(choice);
search();
path.pop();
```

Without restoration, one branch contaminates another.


# 47. Branch-and-Bound Intuition

Backtracking can sometimes prune using a bound:

```text
current best
vs
best possible completion
```

If the branch cannot beat the current solution:

```text
prune
```

This does not automatically improve worst-case complexity, but can massively reduce practical search.


# 48. Deduplication

Deduplication can use:

```text
Set
sorting
frequency map
```

Compare:

```text
Set-based one-pass
```

versus:

```text
sort then unique adjacent values
```

Selection depends on:

```text
memory
ordering needs
data types
streaming requirements
```


# 49. Sorting + Two Pointers

A powerful composition:

```text
sort
+
two pointers
```

Many pair/triple/range problems transform from:

```text
O(n²) or worse
```

toward:

```text
O(n log n)
```

after sorting and then scanning.

The sort is preprocessing cost.

The scan exploits the resulting order.


# 50. Prefix Techniques

Prefix sums generalize to:

```text
prefix counts
prefix minima/maxima
difference arrays
cumulative frequency
```

The pattern:

```text
precompute reusable partial results
```

is one of the most important ways to eliminate repeated work.


# 51. Monotonic Structures

Monotonic stacks/queues maintain elements in increasing or decreasing order.

Typical use cases:

```text
next greater element
sliding-window maximum
histogram problems
```

The key observation is:

```text
elements can be removed permanently
because future comparisons can never need them
```

This often turns repeated search into linear amortized work.


# 52. Monotonic Stack Example

For each value, determine the next greater value.

Conceptual algorithm:

```text
stack contains candidates
for each current value:
    while top is smaller:
        resolve top
    push current
```

Each element is:

```text
pushed once
popped at most once
```

Therefore total stack operations are:

```text
O(n)
```

even though there is a loop inside a loop.


# 53. Selection vs Full Ordering

If the requirement is:

```text
minimum
maximum
top k
median
```

do not automatically sort everything.

Candidates:

```text
linear scan
heap
quickselect-style selection
sorted structure
```

The simplest correct choice often wins when only a single extremum is required.


# 54. Quickselect Intuition

Quickselect partitions around a pivot and recursively/iteratively keeps only the side containing the desired rank.

Typical expected behavior:

```text
O(n)
```

Worst case:

```text
O(n²)
```

It can find a kth element without fully sorting all values.

As always, pivot strategy matters.


# 55. Online vs Offline Algorithms

An **online** algorithm processes data as it arrives without requiring the future.

Examples:

```text
streaming maximum
online frequency tracking
incremental queue processing
```

An **offline** algorithm can inspect the full input first.

Sorting-based approaches are often easier offline.

Production systems often need online algorithms because waiting for the complete dataset is impossible.


# 56. Streaming Algorithms

Streaming algorithms process data with bounded state.

Example:

```text
running sum
running count
running maximum
top-k with bounded heap
```

They are valuable when:

```text
input is huge
input never fully fits in memory
data arrives continuously
latency matters
```


# 57. Approximation and Sketches

When exact answers are too expensive, systems may use approximate algorithms.

Examples include:

```text
cardinality sketches
frequency sketches
sampling
reservoir sampling
```

The engineering trade-off becomes:

```text
memory
accuracy
latency
```

Approximation is justified only when the error bounds are acceptable.


# 58. Randomized Algorithms

Randomization can improve expected performance or simplify algorithms.

Examples:

```text
random pivot selection
sampling
randomized load distribution
```

Analyze:

```text
expected cost
failure probability
randomness source
reproducibility
```

For security-sensitive randomness, use a cryptographically secure source rather than `Math.random()`.


# 59. Determinism

Production algorithms may need deterministic output.

Ask:

```text
Does iteration order matter?
Are equal elements ordered consistently?
Does randomized pivoting need a seed?
Will retries produce the same result?
```

Determinism improves:

```text
testing
debugging
reproducibility
incident analysis
```

but should not be imposed where it adds unnecessary cost.


# 60. Algorithmic Stability

Stability can mean more than sorting stability.

A production algorithm should have stable, documented behavior around:

```text
ties
duplicates
empty input
invalid input
ordering
```

Deterministic tie handling often reduces downstream complexity.


# 61. Algorithmic Invariants — Checklist

Before implementation write:

```text
[ ] precondition
[ ] state representation
[ ] loop invariant
[ ] progress measure
[ ] termination
[ ] postcondition
[ ] duplicate policy
[ ] empty input
[ ] invalid input
```

This prevents many subtle bugs.


# 62. Termination

An algorithm should have a decreasing or otherwise bounded progress measure.

Examples:

```text
binary search:
search interval shrinks

two pointers:
left/right move toward termination

BFS:
finite unvisited nodes decrease

backtracking:
finite remaining choices
```

If you cannot explain termination, you do not yet fully understand the algorithm.


# 63. Production Constraint — Recursion Depth

JavaScript does not guarantee unlimited recursion depth.

For large trees/graphs:

```text
recursive DFS
```

can fail because of call-stack limits.

Use:

```text
explicit stack
```

when input depth may be large or attacker-controlled.


# 64. Production Constraint — Event Loop

Algorithms that are acceptable in batch jobs may freeze the browser or block Node.

Examples:

```text
O(n²) comparison
large sort
deep graph traversal
large JSON transformation
```

on synchronous execution.

Consider:

```text
chunking
workers
background jobs
streaming
bounded work
```

when the input can be large.


# 65. Production Constraint — Memory

An algorithm that uses:

```text
O(n)
```

extra memory

may fail if:

```text
n = 100 million
```

Even if the asymptotic class looks acceptable.

Estimate:

```text
bytes per element
metadata overhead
peak temporary memory
GC impact
```

not just:

```text
O(n)
```


# 66. Production Constraint — Remote Work

Never hide remote calls inside an algorithmic loop:

```js
for (const item of items) {
  await database.lookup(item.id);
}
```

This may perform:

```text
n database round trips
```

A better algorithm often:

```text
collect IDs
batch lookup
index results
process locally
```

The algorithmic improvement is partly a systems-design improvement.


# 67. Production Constraint — Retries

An algorithm can accidentally amplify work when a failed operation is retried.

Suppose:

```text
base work = n
retry count = r
```

Effective work can approach:

```text
n × r
```

and distributed fan-out can multiply it further.

Always analyze:

```text
worst retry path
```

for production algorithms.


# 68. Algorithm + Data Structure Pairing

Common pairings:

```text
binary search + sorted array
BFS + queue
DFS + stack
Dijkstra + priority queue
topological sort + indegree map/queue
frequency counting + Map
deduplication + Set
LRU + Map + linked structure
top-k + heap
prefix search + Trie
```

The data structure enables the algorithmic pattern.


# 69. Algorithm Transformation Pattern

A useful sequence:

```text
naive repeated work
        ↓
identify redundancy
        ↓
store reusable information
        ↓
index / sort / cache / prefix
        ↓
perform cheaper repeated operation
```

Examples:

```text
find() in loop
→ Map index

recompute range sum
→ prefix sums

check every pair
→ sorting + two pointers

recompute recursive state
→ memoization
```


# 70. Algorithm Composition

Real solutions combine algorithms.

Example:

```text
parse
→ filter
→ index
→ sort subset
→ top-k
→ serialize
```

Analyze the entire pipeline:

```text
sum of sequential costs
+
dominant stage
+
memory
+
I/O
```

Do not optimize one stage while another dominates.


# 71. Code Walkthrough — Two Sum

Naive:

```js
function twoSum(values, target) {
  for (let i = 0; i < values.length; i++) {
    for (let j = i + 1; j < values.length; j++) {
      if (values[i] + values[j] === target) {
        return [i, j];
      }
    }
  }

  return undefined;
}
```

Complexity:

```text
O(n²)
```

A Map-based approach can reduce expected time to:

```text
O(n)
```

with:

```text
O(n)
```

additional memory.

This is the canonical:

```text
nested search
→
indexing
```

transformation.


# 72. Code Walkthrough — Two Sum With Map

```js
function twoSum(values, target) {
  const seen = new Map();

  for (let i = 0; i < values.length; i++) {
    const needed = target - values[i];

    if (seen.has(needed)) {
      return [seen.get(needed), i];
    }

    seen.set(values[i], i);
  }

  return undefined;
}
```

Invariant:

```text
seen contains exactly the relevant earlier values
and their indices
```

The algorithm trades:

```text
memory
```

for:

```text
fewer comparisons
```


# 73. Code Walkthrough — Merge

```js
function mergeSorted(a, b) {
  const result = [];
  let i = 0;
  let j = 0;

  while (i < a.length && j < b.length) {
    if (a[i] <= b[j]) {
      result.push(a[i++]);
    } else {
      result.push(b[j++]);
    }
  }

  while (i < a.length) result.push(a[i++]);
  while (j < b.length) result.push(b[j++]);

  return result;
}
```

Every input element is emitted once:

```text
O(n + m)
```

plus output space.


# 74. Code Walkthrough — BFS Shortest Path

```js
function shortestPath(graph, start, goal) {
  const queue = [start];
  let head = 0;

  const previous = new Map([
    [start, null],
  ]);

  while (head < queue.length) {
    const node = queue[head++];

    if (node === goal) break;

    for (const next of graph.neighbors(node)) {
      if (!previous.has(next)) {
        previous.set(next, node);
        queue.push(next);
      }
    }
  }

  if (!previous.has(goal)) {
    return undefined;
  }

  const path = [];

  for (let node = goal; node !== null; node = previous.get(node)) {
    path.push(node);
  }

  path.reverse();
  return path;
}
```

The `previous` map simultaneously:

```text
marks visited
+
stores predecessor
```

This avoids maintaining two separate structures.


# 75. Code Walkthrough — Topological Sort

Conceptual implementation:

```js
function topologicalSort(graph) {
  const indegree = new Map();
  const queue = [];

  for (const node of graph.vertices()) {
    indegree.set(node, 0);
  }

  for (const node of graph.vertices()) {
    for (const next of graph.neighbors(node)) {
      indegree.set(next, indegree.get(next) + 1);
    }
  }

  for (const [node, degree] of indegree) {
    if (degree === 0) {
      queue.push(node);
    }
  }

  const result = [];

  while (queue.length) {
    const node = queue.shift();
    result.push(node);

    for (const next of graph.neighbors(node)) {
      const degree = indegree.get(next) - 1;
      indegree.set(next, degree);

      if (degree === 0) {
        queue.push(next);
      }
    }
  }

  if (result.length !== indegree.size) {
    throw new Error("Graph contains a cycle");
  }

  return result;
}
```

For a production large graph, replace repeated `shift()` with a head-index queue or deque.


# 76. Debugging Exercise — Binary Search

A binary search sometimes returns:

```text
-1
```

even though the target exists.

Investigate:

```text
low update
high update
mid calculation
sorted precondition
comparator
duplicate policy
```

The invariant should be:

```text
if target exists, it remains in the search interval
```


# 77. Debugging Exercise — Sliding Window

A longest-substring algorithm works until duplicate characters appear.

Check:

```text
what the window represents
when left moves
whether state is removed
whether the recorded answer matches the current valid window
```

A sliding-window invariant must state exactly what is guaranteed inside:

```text
[left, right]
```


# 78. Debugging Exercise — BFS

BFS visits the same node many times.

Possible causes:

```text
visited marking occurs too late
```

versus:

```text
visited marking occurs when enqueued
```

For many graph traversals, marking at enqueue/discovery time prevents repeated queue insertion.


# 79. Debugging Exercise — DFS

Recursive DFS crashes on a deeply nested graph.

Possible fix:

```text
replace call stack
with explicit stack
```

Do not merely increase assumptions about recursion depth.

First ask whether the input depth is fundamentally bounded.


# 80. Debugging Exercise — DP

A memoized algorithm still performs far too much work.

Inspect:

```text
state definition
memo key
number of distinct states
unnecessary dimensions
state normalization
```

Bad memoization:

```text
stores too little to uniquely identify a subproblem
```

or:

```text
stores an unnecessarily huge state
```


# 81. Code Review Exercise

Review:

```js
function findMatches(users, queries) {
  return queries.map(query =>
    users.filter(user =>
      user.name.toLowerCase().includes(
        query.toLowerCase()
      )
    )
  );
}
```

Assume:

```text
n users
q queries
average name length = L
```

Questions:

```text
1. What repeated work occurs?
2. What happens when q ≈ n?
3. Is exact indexing possible?
4. Would prefix indexing work?
5. Would a Trie help?
6. Does substring search require a more complex index?
7. What memory trade-off would an index introduce?
```


# 82. Predict-the-Output — Search

Before running:

```js
const values = [2, 4, 6, 8];

console.log(binarySearch(values, 6));
console.log(binarySearch(values, 5));
```

Predict both results.

Then trace:

```text
low
high
mid
```


# 83. Predict-the-Output — Queue BFS

Graph:

```text
A → B, C
B → D
C → D
D → none
```

Starting from `A`, predict BFS discovery order when neighbors are iterated as written.

Then explain why `D` should normally enter the queue only once.


# 84. Predict-the-Output — Topological Ordering

Given:

```text
A → C
B → C
C → D
```

Possible topological orders include more than one valid result depending on queue ordering.

The important property is:

```text
A before C
B before C
C before D
```

Do not mistake:

```text
one valid order
```

for:

```text
the only order
```


# 85. Interview Questions — Foundation

1. What is an algorithm?
2. What is linear search?
3. When is binary search valid?
4. What is an invariant?
5. Explain two pointers.
6. What is a sliding window?
7. What are prefix sums?
8. What is frequency counting?
9. Why does hashing reduce repeated search?
10. What is divide and conquer?


# 86. Interview Questions — Intermediate

11. Compare merge sort and quicksort.
12. What makes a sort stable?
13. Explain BFS.
14. Explain DFS.
15. Why is BFS useful for unweighted shortest paths?
16. Explain topological sorting.
17. Explain cycle detection.
18. What is memoization?
19. What is dynamic programming?
20. What is backtracking?


# 87. Interview Questions — Advanced

21. Transform O(n²) pair search into expected O(n).
22. Design a top-k algorithm.
23. Solve a bounded-window problem.
24. Design shortest path for an unweighted graph.
25. Design a dependency-ordering algorithm.
26. Analyze a monotonic stack.
27. Explain why nested loops can still be O(n).
28. Explain an algorithm's invariant formally.
29. Choose recursion or iteration for a deep tree.
30. Explain the memory/time trade-off in memoization.


# 88. Interview Questions — Principal

31. How do you decide whether to preprocess with sorting?
32. When is an O(n log n) algorithm preferable to expected O(n)?
33. How would you prevent a CPU-bound algorithm from blocking Node?
34. How would you constrain an algorithm exposed to untrusted input?
35. How do network calls change the algorithmic model?
36. How would you choose between exact and approximate algorithms?
37. How do you prove a production optimization did not change semantics?
38. How do you benchmark algorithm crossover points?
39. How would you instrument algorithmic pressure in production?
40. When should a simpler slower algorithm win?


# 89. Mastery Exercises — Search and Sorting

Implement without references:

```text
linear search
binary search
lower bound
upper bound
merge
merge sort
quicksort
quickselect
```

For each provide:

```text
preconditions
invariant
time
space
edge cases
tests
```


# 90. Mastery Exercises — Sliding / Hashing

Implement:

```text
two sum
duplicate detection
frequency counter
fixed sliding window
variable sliding window
prefix sum query
range count
top-k frequency
```

For each:

```text
naive approach
optimized approach
trade-off
```


# 91. Mastery Exercises — Graphs

Implement:

```text
BFS
DFS
connected components
cycle detection
topological sort
unweighted shortest path
weighted shortest path with nonnegative edges
```

Use both:

```text
recursive
iterative
```

where the problem permits, and explain the production implications.


# 92. Mastery Exercises — Dynamic Programming

Implement:

```text
Fibonacci
climbing stairs
coin change
0/1 knapsack
longest common subsequence
edit distance
```

For each:

```text
state
transition
base case
answer
time
space
state compression possibility
```


# 93. Mastery Exercises — Backtracking

Implement:

```text
subsets
permutations
combinations
N-Queens
word search
constraint assignment
```

For each define:

```text
choice
constraint
base case
undo
pruning
```

Then estimate worst-case complexity.


# 94. Mastery Project — Algorithm Library

Create a small educational library:

```text
search/
sort/
array/
string/
graph/
dynamic-programming/
backtracking/
```

Each algorithm must have:

```text
implementation
tests
complexity note
invariants
README
benchmark
```

Do not optimize prematurely.


# 95. Mastery Project — Production Scheduler

Design a scheduler supporting:

```text
priority
FIFO tie breaking
cancellation
bounded memory
```

Possible building blocks:

```text
heap
Map
queue/index
state flags
```

Defend:

```text
data structure
algorithm
complexity
memory
failure mode
observability
```


# 96. Mastery Project — Dependency Resolver

Implement a dependency resolver that:

```text
accepts directed dependencies
detects cycles
produces valid order
reports cycle participants
handles disconnected components
```

Use:

```text
graph
indegree or DFS states
```

Then test malformed input.


# 97. Mastery Project — Large Dataset Pipeline

Build:

```text
stream records
→ validate
→ deduplicate
→ aggregate
→ top-k
→ output
```

Requirements:

```text
bounded memory
no full dataset copy
measurable throughput
error counters
```

Document:

```text
algorithmic complexity
peak memory model
backpressure strategy
```


# 98. Algorithm Selection Matrix

| Problem shape | Strong first candidates |
|---|---|
| unsorted exact search | linear scan / Map if repeated |
| sorted exact search | binary search |
| pair in sorted data | two pointers |
| contiguous range | sliding window / prefix sums |
| repeated membership | Set |
| repeated key lookup | Map |
| top k | heap / selection |
| unweighted shortest path | BFS |
| graph reachability | DFS/BFS |
| dependency ordering | topological sort |
| nonnegative weighted shortest path | Dijkstra-style algorithm |
| overlapping subproblems | dynamic programming |
| exhaustive constrained search | backtracking |
| repeated subproblem | memoization |
| prefix queries | Trie / sorted index |


# 99. Algorithmic Trade-Off Matrix

| Goal | Possible trade-off |
|---|---|
| lower time | more memory |
| lower memory | more repeated work |
| lower latency | more preprocessing |
| bounded memory | approximate result |
| deterministic output | less randomization |
| simpler code | potentially slower |
| lower CPU | higher network or memory traffic |
| lower tail latency | more batching/chunking |


# 100. Common Misconceptions

### “Binary search is always better.”
Only when the data is ordered and the ordering can be maintained.

### “Hashing always beats sorting.”
Hashing may use more memory, lacks range-order semantics, and has expected rather than unconditional constant-time behavior.

### “BFS is always better than DFS.”
Different goals produce different choices.

### “Dynamic programming means recursion.”
DP can be top-down or bottom-up and may be fully iterative.

### “Greedy means optimal.”
Only with the required proof/property.

### “O(n) is always enough.”
`n` may be enormous, and inner work/I/O may dominate.

### “Recursion is cleaner, therefore preferable.”
Depth and stack limits matter.

### “Library algorithms are less educational.”
Production correctness, edge cases, and performance engineering often favor well-tested platform/library implementations over custom versions.


# 101. Common Mistakes

```text
[ ] binary search on unsorted data
[ ] incorrect boundary updates
[ ] forgotten duplicate policy
[ ] quadratic copying inside a loop
[ ] queue implemented with repeated shift in a hot path
[ ] BFS without visited tracking
[ ] DFS recursion on unbounded depth
[ ] topological sort without cycle detection
[ ] Dijkstra with negative edge weights
[ ] memoization with incomplete state keys
[ ] backtracking without undo
[ ] greedy algorithm without proof
[ ] ignoring output space
[ ] ignoring remote calls
[ ] optimizing without benchmarks
```


# 102. Production Algorithm Checklist

```text
[ ] inputs defined
[ ] constraints defined
[ ] preconditions documented
[ ] invariant documented
[ ] termination explained
[ ] correctness tested
[ ] worst case analyzed
[ ] expected case stated where relevant
[ ] memory analyzed
[ ] output size analyzed
[ ] remote work counted
[ ] retries counted
[ ] attacker-controlled input bounded
[ ] event-loop impact considered
[ ] benchmark exists
[ ] p95/p99 impact considered
[ ] observability exists
[ ] simpler alternative evaluated
```


# 103. Spaced Retrieval Plan

### Day 0
Explain:

```text
linear search
binary search
two pointers
sliding window
BFS
DFS
topological sort
dynamic programming
backtracking
```

### Day 2
Implement:

```text
binary search
two sum
BFS
DFS
```

without references.

### Day 7
Implement:

```text
merge sort
quickselect
topological sort
```

### Day 14
Solve:

```text
one sliding-window problem
one DP problem
one graph problem
one backtracking problem
```

### Day 30
Conduct a production algorithm review for a large data-processing endpoint.


# 104. Track A — Core Theory

Study:

```text
invariants
termination
search
sort
divide-and-conquer
graph traversal
greedy
dynamic programming
backtracking
amortized reasoning
```

For every algorithm know:

```text
why it is correct
```

before memorizing:

```text
time complexity
```


# 105. Track B — Implementation

Progress:

```text
guided
→ partially guided
→ no-reference
→ edge-case hardened
→ production-grade
```

For each implementation include:

```text
tests
invariants
benchmark
failure behavior
documentation
```


# 106. Track C — Interview / Reasoning

Practice:

```text
recognize pattern
derive solution
prove invariant
calculate complexity
compare alternatives
predict edge cases
defend trade-offs
```

Do not solve by memorizing the name of an algorithm alone.

Map:

```text
constraint → pattern
```


# 107. Principal Decision Framework

Evaluate an algorithm across:

| Dimension | Question |
|---|---|
| Correctness | What proof/invariant guarantees the result? |
| Performance | How does work scale? |
| Memory | What is peak/retained space? |
| Security | What adversarial input worsens cost? |
| Reliability | What happens under partial/failure states? |
| Maintainability | Can engineers understand and modify it? |
| Scalability | What happens at 10× / 100× input? |
| Observability | Can pressure and failures be measured? |
| Developer Experience | Is the implementation/API clear? |
| Operational Complexity | Does it need preprocessing, indexes, caches, or tuning? |
| Future Change | Can the assumptions remain true as requirements evolve? |


# 108. Completion Criteria

```text
[ ] Linear search
[ ] Binary search
[ ] Lower/upper bound
[ ] Two pointers
[ ] Sliding window
[ ] Prefix sums
[ ] Frequency counting
[ ] Hash-based indexing
[ ] Sorting
[ ] Stable sort concept
[ ] Merge sort
[ ] Quicksort
[ ] Heap selection
[ ] Quickselect
[ ] Divide and conquer
[ ] Recursion/iteration trade-off
[ ] BFS
[ ] DFS
[ ] Shortest path foundations
[ ] Topological sort
[ ] Cycle detection
[ ] Greedy reasoning
[ ] Dynamic programming
[ ] Memoization
[ ] Backtracking
[ ] Monotonic structures
[ ] Streaming algorithms
[ ] Approximation awareness
[ ] Randomized algorithms
[ ] Correctness invariants
[ ] Production algorithm review
```


# 109. Mastery Gate

### Understand
You can describe the major algorithmic patterns.

### Explain
You can teach the invariant and preconditions.

### Predict
You can predict behavior and complexity before execution.

### Implement
You can implement core algorithms without copying.

### Debug
You can identify which invariant or boundary failed.

### Apply
You can recognize the right pattern from real constraints.

### Compare
You can defend alternatives and trade-offs.

### Defend
You can explain why the algorithm is appropriate for:

```text
correctness
scale
memory
security
latency
throughput
maintainability
operational complexity
```


# 110. Status

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


# Chapter 73 — Revision / Retrieval Record

| Date | Retrieval task | Result | Weak area | Next action |
|---|---|---|---|---|
| YYYY-MM-DD | Implement binary search | ✅ / ❌ | ... | ... |
| YYYY-MM-DD | Solve two-pointer problem | ✅ / ❌ | ... | ... |
| YYYY-MM-DD | Implement sliding window | ✅ / ❌ | ... | ... |
| YYYY-MM-DD | Implement BFS/DFS | ✅ / ❌ | ... | ... |
| YYYY-MM-DD | Explain topological sort | ✅ / ❌ | ... | ... |
| YYYY-MM-DD | Solve DP problem | ✅ / ❌ | ... | ... |
| YYYY-MM-DD | Solve backtracking problem | ✅ / ❌ | ... | ... |
| YYYY-MM-DD | Principal algorithm review | ✅ / ❌ | ... | ... |

### Retrieval Prompts

```text
1. What precondition makes binary search valid?
2. What is a loop invariant?
3. Why can a nested loop be O(n)?
4. When does sorting enable a better algorithm?
5. Why does BFS find shortest paths in unweighted graphs?
6. Why does DFS need cycle/visited reasoning?
7. Why can topological sort fail?
8. What property does Dijkstra require?
9. What makes dynamic programming applicable?
10. What makes greedy algorithms correct?
11. How do you bound recursion depth?
12. When should you trade memory for speed?
13. How do remote calls change algorithmic cost?
14. How would you benchmark an algorithm crossover point?
15. What makes an algorithm production-grade?
```


# Chapter 73 — Canonical References and Source Discipline

## Primary language and platform references

- ECMAScript Language Specification  
  https://tc39.es/ecma262/
- MDN JavaScript  
  https://developer.mozilla.org/docs/Web/JavaScript
- Node.js Documentation  
  https://nodejs.org/docs/latest/api/

Use the ECMAScript specification for language semantics and platform documentation for host/runtime behavior.

## Algorithm references

For formal algorithm analysis, use established algorithm textbooks or university-level algorithm references for:

```text
sorting lower bounds
graph algorithms
dynamic programming
greedy proofs
divide and conquer
randomized algorithms
amortized analysis
```

## Source discipline

1. State the problem model before selecting an algorithm.
2. State all preconditions.
3. Separate correctness from complexity.
4. State worst/expected/amortized qualifiers explicitly.
5. Preserve multiple input dimensions such as `V` and `E`.
6. Count output space separately when useful.
7. Treat network/database operations as real work.
8. Do not treat JavaScript syntax as a proof of a particular complexity.
9. Use built-in/platform algorithms in production when they are adequate and better tested.
10. Benchmark custom optimizations against the actual workload.
11. Never use an algorithm outside its proven assumptions without documenting the resulting risk.


# Chapter 73 — Completion Snapshot

## Search / Array Patterns

```text
[ ] linear search
[ ] binary search
[ ] lower bound
[ ] upper bound
[ ] two pointers
[ ] sliding window
[ ] prefix sums
[ ] frequency counting
```

## Sorting / Selection

```text
[ ] merge sort
[ ] quicksort
[ ] heap selection
[ ] quickselect
[ ] stable ordering
```

## Graphs

```text
[ ] BFS
[ ] DFS
[ ] visited state
[ ] cycle detection
[ ] topological sort
[ ] shortest path foundations
```

## Optimization Patterns

```text
[ ] memoization
[ ] dynamic programming
[ ] greedy reasoning
[ ] backtracking
[ ] monotonic structures
[ ] streaming
[ ] approximation
```

## Production

```text
[ ] invariants
[ ] termination
[ ] complexity
[ ] memory
[ ] security
[ ] event-loop impact
[ ] remote work
[ ] benchmarking
[ ] observability
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

Core algorithms are not isolated tricks.

They are reusable transformations:

```text
repeated search
→ indexing

repeated range work
→ prefix state

pairwise comparison
→ ordering + two pointers

recursive recomputation
→ memoization / DP

relationship traversal
→ BFS / DFS

dependency ordering
→ topological sort

priority selection
→ heap

bounded candidate selection
→ quickselect / heap

exhaustive constrained search
→ backtracking + pruning
```

The principal engineer's job is to ask:

```text
What assumptions make this algorithm valid?
What invariant proves correctness?
What does the input scale look like?
What is the worst case?
What is the expected case?
What memory does it require?
What happens under adversarial input?
Does it block the event loop?
Does it perform remote work?
Can we benchmark the decision?
Is the additional complexity worth it?
```

The deepest lesson is:

> **Algorithmic skill is pattern recognition plus proof plus cost awareness.**

The sequence is:

```text
data structures
      ↓
complexity
      ↓
algorithms
      ↓
design patterns and programming paradigms
```

Chapter 74 will build on this foundation by changing the way computation itself is structured:

```text
imperative operations
      ↓
composition
      ↓
Chapter 74 — Functional Programming
```


# 111. Extended Retrieval Bank

### Retrieval Drill 1

Given a workload with approximately `250` elements, choose one algorithmic pattern from:

```text
search
sorting
two pointers
sliding window
hashing
BFS/DFS
dynamic programming
backtracking
```

Then write:

```text
1. problem shape
2. precondition
3. invariant
4. termination argument
5. time complexity
6. auxiliary space
7. production risk
8. credible alternative
```

### Retrieval Drill 2

Given a workload with approximately `500` elements, choose one algorithmic pattern from:

```text
search
sorting
two pointers
sliding window
hashing
BFS/DFS
dynamic programming
backtracking
```

Then write:

```text
1. problem shape
2. precondition
3. invariant
4. termination argument
5. time complexity
6. auxiliary space
7. production risk
8. credible alternative
```

### Retrieval Drill 3

Given a workload with approximately `750` elements, choose one algorithmic pattern from:

```text
search
sorting
two pointers
sliding window
hashing
BFS/DFS
dynamic programming
backtracking
```

Then write:

```text
1. problem shape
2. precondition
3. invariant
4. termination argument
5. time complexity
6. auxiliary space
7. production risk
8. credible alternative
```

### Retrieval Drill 4

Given a workload with approximately `1000` elements, choose one algorithmic pattern from:

```text
search
sorting
two pointers
sliding window
hashing
BFS/DFS
dynamic programming
backtracking
```

Then write:

```text
1. problem shape
2. precondition
3. invariant
4. termination argument
5. time complexity
6. auxiliary space
7. production risk
8. credible alternative
```

### Retrieval Drill 5

Given a workload with approximately `1250` elements, choose one algorithmic pattern from:

```text
search
sorting
two pointers
sliding window
hashing
BFS/DFS
dynamic programming
backtracking
```

Then write:

```text
1. problem shape
2. precondition
3. invariant
4. termination argument
5. time complexity
6. auxiliary space
7. production risk
8. credible alternative
```

### Retrieval Drill 6

Given a workload with approximately `1500` elements, choose one algorithmic pattern from:

```text
search
sorting
two pointers
sliding window
hashing
BFS/DFS
dynamic programming
backtracking
```

Then write:

```text
1. problem shape
2. precondition
3. invariant
4. termination argument
5. time complexity
6. auxiliary space
7. production risk
8. credible alternative
```

### Retrieval Drill 7

Given a workload with approximately `1750` elements, choose one algorithmic pattern from:

```text
search
sorting
two pointers
sliding window
hashing
BFS/DFS
dynamic programming
backtracking
```

Then write:

```text
1. problem shape
2. precondition
3. invariant
4. termination argument
5. time complexity
6. auxiliary space
7. production risk
8. credible alternative
```

### Retrieval Drill 8

Given a workload with approximately `2000` elements, choose one algorithmic pattern from:

```text
search
sorting
two pointers
sliding window
hashing
BFS/DFS
dynamic programming
backtracking
```

Then write:

```text
1. problem shape
2. precondition
3. invariant
4. termination argument
5. time complexity
6. auxiliary space
7. production risk
8. credible alternative
```

### Retrieval Drill 9

Given a workload with approximately `2250` elements, choose one algorithmic pattern from:

```text
search
sorting
two pointers
sliding window
hashing
BFS/DFS
dynamic programming
backtracking
```

Then write:

```text
1. problem shape
2. precondition
3. invariant
4. termination argument
5. time complexity
6. auxiliary space
7. production risk
8. credible alternative
```

### Retrieval Drill 10

Given a workload with approximately `2500` elements, choose one algorithmic pattern from:

```text
search
sorting
two pointers
sliding window
hashing
BFS/DFS
dynamic programming
backtracking
```

Then write:

```text
1. problem shape
2. precondition
3. invariant
4. termination argument
5. time complexity
6. auxiliary space
7. production risk
8. credible alternative
```

### Retrieval Drill 11

Given a workload with approximately `2750` elements, choose one algorithmic pattern from:

```text
search
sorting
two pointers
sliding window
hashing
BFS/DFS
dynamic programming
backtracking
```

Then write:

```text
1. problem shape
2. precondition
3. invariant
4. termination argument
5. time complexity
6. auxiliary space
7. production risk
8. credible alternative
```

### Retrieval Drill 12

Given a workload with approximately `3000` elements, choose one algorithmic pattern from:

```text
search
sorting
two pointers
sliding window
hashing
BFS/DFS
dynamic programming
backtracking
```

Then write:

```text
1. problem shape
2. precondition
3. invariant
4. termination argument
5. time complexity
6. auxiliary space
7. production risk
8. credible alternative
```

### Retrieval Drill 13

Given a workload with approximately `3250` elements, choose one algorithmic pattern from:

```text
search
sorting
two pointers
sliding window
hashing
BFS/DFS
dynamic programming
backtracking
```

Then write:

```text
1. problem shape
2. precondition
3. invariant
4. termination argument
5. time complexity
6. auxiliary space
7. production risk
8. credible alternative
```

### Retrieval Drill 14

Given a workload with approximately `3500` elements, choose one algorithmic pattern from:

```text
search
sorting
two pointers
sliding window
hashing
BFS/DFS
dynamic programming
backtracking
```

Then write:

```text
1. problem shape
2. precondition
3. invariant
4. termination argument
5. time complexity
6. auxiliary space
7. production risk
8. credible alternative
```

### Retrieval Drill 15

Given a workload with approximately `3750` elements, choose one algorithmic pattern from:

```text
search
sorting
two pointers
sliding window
hashing
BFS/DFS
dynamic programming
backtracking
```

Then write:

```text
1. problem shape
2. precondition
3. invariant
4. termination argument
5. time complexity
6. auxiliary space
7. production risk
8. credible alternative
```

### Retrieval Drill 16

Given a workload with approximately `4000` elements, choose one algorithmic pattern from:

```text
search
sorting
two pointers
sliding window
hashing
BFS/DFS
dynamic programming
backtracking
```

Then write:

```text
1. problem shape
2. precondition
3. invariant
4. termination argument
5. time complexity
6. auxiliary space
7. production risk
8. credible alternative
```

### Retrieval Drill 17

Given a workload with approximately `4250` elements, choose one algorithmic pattern from:

```text
search
sorting
two pointers
sliding window
hashing
BFS/DFS
dynamic programming
backtracking
```

Then write:

```text
1. problem shape
2. precondition
3. invariant
4. termination argument
5. time complexity
6. auxiliary space
7. production risk
8. credible alternative
```

### Retrieval Drill 18

Given a workload with approximately `4500` elements, choose one algorithmic pattern from:

```text
search
sorting
two pointers
sliding window
hashing
BFS/DFS
dynamic programming
backtracking
```

Then write:

```text
1. problem shape
2. precondition
3. invariant
4. termination argument
5. time complexity
6. auxiliary space
7. production risk
8. credible alternative
```

### Retrieval Drill 19

Given a workload with approximately `4750` elements, choose one algorithmic pattern from:

```text
search
sorting
two pointers
sliding window
hashing
BFS/DFS
dynamic programming
backtracking
```

Then write:

```text
1. problem shape
2. precondition
3. invariant
4. termination argument
5. time complexity
6. auxiliary space
7. production risk
8. credible alternative
```

### Retrieval Drill 20

Given a workload with approximately `5000` elements, choose one algorithmic pattern from:

```text
search
sorting
two pointers
sliding window
hashing
BFS/DFS
dynamic programming
backtracking
```

Then write:

```text
1. problem shape
2. precondition
3. invariant
4. termination argument
5. time complexity
6. auxiliary space
7. production risk
8. credible alternative
```

### Retrieval Drill 21

Given a workload with approximately `5250` elements, choose one algorithmic pattern from:

```text
search
sorting
two pointers
sliding window
hashing
BFS/DFS
dynamic programming
backtracking
```

Then write:

```text
1. problem shape
2. precondition
3. invariant
4. termination argument
5. time complexity
6. auxiliary space
7. production risk
8. credible alternative
```

### Retrieval Drill 22

Given a workload with approximately `5500` elements, choose one algorithmic pattern from:

```text
search
sorting
two pointers
sliding window
hashing
BFS/DFS
dynamic programming
backtracking
```

Then write:

```text
1. problem shape
2. precondition
3. invariant
4. termination argument
5. time complexity
6. auxiliary space
7. production risk
8. credible alternative
```

### Retrieval Drill 23

Given a workload with approximately `5750` elements, choose one algorithmic pattern from:

```text
search
sorting
two pointers
sliding window
hashing
BFS/DFS
dynamic programming
backtracking
```

Then write:

```text
1. problem shape
2. precondition
3. invariant
4. termination argument
5. time complexity
6. auxiliary space
7. production risk
8. credible alternative
```

### Retrieval Drill 24

Given a workload with approximately `6000` elements, choose one algorithmic pattern from:

```text
search
sorting
two pointers
sliding window
hashing
BFS/DFS
dynamic programming
backtracking
```

Then write:

```text
1. problem shape
2. precondition
3. invariant
4. termination argument
5. time complexity
6. auxiliary space
7. production risk
8. credible alternative
```

### Retrieval Drill 25

Given a workload with approximately `6250` elements, choose one algorithmic pattern from:

```text
search
sorting
two pointers
sliding window
hashing
BFS/DFS
dynamic programming
backtracking
```

Then write:

```text
1. problem shape
2. precondition
3. invariant
4. termination argument
5. time complexity
6. auxiliary space
7. production risk
8. credible alternative
```

### Retrieval Drill 26

Given a workload with approximately `6500` elements, choose one algorithmic pattern from:

```text
search
sorting
two pointers
sliding window
hashing
BFS/DFS
dynamic programming
backtracking
```

Then write:

```text
1. problem shape
2. precondition
3. invariant
4. termination argument
5. time complexity
6. auxiliary space
7. production risk
8. credible alternative
```

### Retrieval Drill 27

Given a workload with approximately `6750` elements, choose one algorithmic pattern from:

```text
search
sorting
two pointers
sliding window
hashing
BFS/DFS
dynamic programming
backtracking
```

Then write:

```text
1. problem shape
2. precondition
3. invariant
4. termination argument
5. time complexity
6. auxiliary space
7. production risk
8. credible alternative
```

### Retrieval Drill 28

Given a workload with approximately `7000` elements, choose one algorithmic pattern from:

```text
search
sorting
two pointers
sliding window
hashing
BFS/DFS
dynamic programming
backtracking
```

Then write:

```text
1. problem shape
2. precondition
3. invariant
4. termination argument
5. time complexity
6. auxiliary space
7. production risk
8. credible alternative
```

### Retrieval Drill 29

Given a workload with approximately `7250` elements, choose one algorithmic pattern from:

```text
search
sorting
two pointers
sliding window
hashing
BFS/DFS
dynamic programming
backtracking
```

Then write:

```text
1. problem shape
2. precondition
3. invariant
4. termination argument
5. time complexity
6. auxiliary space
7. production risk
8. credible alternative
```

### Retrieval Drill 30

Given a workload with approximately `7500` elements, choose one algorithmic pattern from:

```text
search
sorting
two pointers
sliding window
hashing
BFS/DFS
dynamic programming
backtracking
```

Then write:

```text
1. problem shape
2. precondition
3. invariant
4. termination argument
5. time complexity
6. auxiliary space
7. production risk
8. credible alternative
```

### Retrieval Drill 31

Given a workload with approximately `7750` elements, choose one algorithmic pattern from:

```text
search
sorting
two pointers
sliding window
hashing
BFS/DFS
dynamic programming
backtracking
```

Then write:

```text
1. problem shape
2. precondition
3. invariant
4. termination argument
5. time complexity
6. auxiliary space
7. production risk
8. credible alternative
```

### Retrieval Drill 32

Given a workload with approximately `8000` elements, choose one algorithmic pattern from:

```text
search
sorting
two pointers
sliding window
hashing
BFS/DFS
dynamic programming
backtracking
```

Then write:

```text
1. problem shape
2. precondition
3. invariant
4. termination argument
5. time complexity
6. auxiliary space
7. production risk
8. credible alternative
```

### Retrieval Drill 33

Given a workload with approximately `8250` elements, choose one algorithmic pattern from:

```text
search
sorting
two pointers
sliding window
hashing
BFS/DFS
dynamic programming
backtracking
```

Then write:

```text
1. problem shape
2. precondition
3. invariant
4. termination argument
5. time complexity
6. auxiliary space
7. production risk
8. credible alternative
```

### Retrieval Drill 34

Given a workload with approximately `8500` elements, choose one algorithmic pattern from:

```text
search
sorting
two pointers
sliding window
hashing
BFS/DFS
dynamic programming
backtracking
```

Then write:

```text
1. problem shape
2. precondition
3. invariant
4. termination argument
5. time complexity
6. auxiliary space
7. production risk
8. credible alternative
```

### Retrieval Drill 35

Given a workload with approximately `8750` elements, choose one algorithmic pattern from:

```text
search
sorting
two pointers
sliding window
hashing
BFS/DFS
dynamic programming
backtracking
```

Then write:

```text
1. problem shape
2. precondition
3. invariant
4. termination argument
5. time complexity
6. auxiliary space
7. production risk
8. credible alternative
```

### Retrieval Drill 36

Given a workload with approximately `9000` elements, choose one algorithmic pattern from:

```text
search
sorting
two pointers
sliding window
hashing
BFS/DFS
dynamic programming
backtracking
```

Then write:

```text
1. problem shape
2. precondition
3. invariant
4. termination argument
5. time complexity
6. auxiliary space
7. production risk
8. credible alternative
```

### Retrieval Drill 37

Given a workload with approximately `9250` elements, choose one algorithmic pattern from:

```text
search
sorting
two pointers
sliding window
hashing
BFS/DFS
dynamic programming
backtracking
```

Then write:

```text
1. problem shape
2. precondition
3. invariant
4. termination argument
5. time complexity
6. auxiliary space
7. production risk
8. credible alternative
```

### Retrieval Drill 38

Given a workload with approximately `9500` elements, choose one algorithmic pattern from:

```text
search
sorting
two pointers
sliding window
hashing
BFS/DFS
dynamic programming
backtracking
```

Then write:

```text
1. problem shape
2. precondition
3. invariant
4. termination argument
5. time complexity
6. auxiliary space
7. production risk
8. credible alternative
```

### Retrieval Drill 39

Given a workload with approximately `9750` elements, choose one algorithmic pattern from:

```text
search
sorting
two pointers
sliding window
hashing
BFS/DFS
dynamic programming
backtracking
```

Then write:

```text
1. problem shape
2. precondition
3. invariant
4. termination argument
5. time complexity
6. auxiliary space
7. production risk
8. credible alternative
```

### Retrieval Drill 40

Given a workload with approximately `10000` elements, choose one algorithmic pattern from:

```text
search
sorting
two pointers
sliding window
hashing
BFS/DFS
dynamic programming
backtracking
```

Then write:

```text
1. problem shape
2. precondition
3. invariant
4. termination argument
5. time complexity
6. auxiliary space
7. production risk
8. credible alternative
```

### Retrieval Drill 41

Given a workload with approximately `10250` elements, choose one algorithmic pattern from:

```text
search
sorting
two pointers
sliding window
hashing
BFS/DFS
dynamic programming
backtracking
```

Then write:

```text
1. problem shape
2. precondition
3. invariant
4. termination argument
5. time complexity
6. auxiliary space
7. production risk
8. credible alternative
```

### Retrieval Drill 42

Given a workload with approximately `10500` elements, choose one algorithmic pattern from:

```text
search
sorting
two pointers
sliding window
hashing
BFS/DFS
dynamic programming
backtracking
```

Then write:

```text
1. problem shape
2. precondition
3. invariant
4. termination argument
5. time complexity
6. auxiliary space
7. production risk
8. credible alternative
```

### Retrieval Drill 43

Given a workload with approximately `10750` elements, choose one algorithmic pattern from:

```text
search
sorting
two pointers
sliding window
hashing
BFS/DFS
dynamic programming
backtracking
```

Then write:

```text
1. problem shape
2. precondition
3. invariant
4. termination argument
5. time complexity
6. auxiliary space
7. production risk
8. credible alternative
```

### Retrieval Drill 44

Given a workload with approximately `11000` elements, choose one algorithmic pattern from:

```text
search
sorting
two pointers
sliding window
hashing
BFS/DFS
dynamic programming
backtracking
```

Then write:

```text
1. problem shape
2. precondition
3. invariant
4. termination argument
5. time complexity
6. auxiliary space
7. production risk
8. credible alternative
```