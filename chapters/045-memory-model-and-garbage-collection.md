# Chapter 45 — JavaScript Memory and Garbage Collection

## Chapter Metadata

```text
Chapter: 45
Title: JavaScript Memory and Garbage Collection
Part: VIII — JavaScript Engine
Status: [+] Expanded
Track A: [ ] Core Theory
Track B: [ ] Implementation
Track C: [ ] Interview / Reasoning
Mastery: [ ] Not Mastered
```

---

# 1. Learning Objectives

By the end of this chapter, the learner must be able to:

- Explain what memory management means in JavaScript.
- Distinguish language semantics from engine-specific memory implementation.
- Explain the conceptual JavaScript memory model.
- Distinguish stack-like execution state from heap-managed objects without treating these as universal physical layouts.
- Explain allocation and object lifetime.
- Explain reachability.
- Explain why garbage collection is possible in JavaScript.
- Explain the basic idea of automatic memory management.
- Explain roots and the reachable object graph.
- Explain why “unreferenced” is a better starting concept than “unused”.
- Explain why garbage collection cannot generally determine business-level liveness.
- Explain:
  - tracing garbage collection;
  - mark-and-sweep;
  - generational collection;
  - copying/evacuation;
  - compaction;
  - incremental GC;
  - concurrent GC;
  - remembered sets/card marking at a conceptual level.
- Explain young-generation versus old-generation allocation strategies.
- Explain why many JavaScript objects die young.
- Explain promotion.
- Explain write barriers.
- Explain allocation fast paths.
- Explain object retention.
- Explain accidental memory retention.
- Explain closure-related retention.
- Explain listener/subscription retention.
- Explain timers and queued tasks as retention sources.
- Explain cache-related retention.
- Explain detached DOM-tree retention conceptually.
- Distinguish memory leak from legitimate long-lived memory.
- Explain heap growth versus leak.
- Explain temporary allocation pressure.
- Explain GC churn.
- Explain stop-the-world pauses conceptually.
- Explain incremental and concurrent collection.
- Understand why GC does not guarantee immediate memory return to the OS.
- Understand fragmentation and compaction.
- Explain weak references and their relationship to GC at a high level.
- Distinguish strong and weak reachability.
- Understand why `WeakMap`/`WeakSet` exist.
- Explain `WeakRef` and finalization as advanced, nondeterministic facilities.
- Understand why manual `delete` is not a universal memory-leak fix.
- Understand how object lifetime relates to execution contexts, closures, Promises, Jobs, workers, and resources.
- Explain memory behavior across Realms and Agents.
- Explain worker/process memory isolation conceptually.
- Understand external memory and off-heap resources at a high level.
- Understand why ArrayBuffers/typed-array backing stores can have memory behavior distinct from ordinary object fields.
- Explain memory consequences of:
  - cloning;
  - transfer;
  - shared memory;
  - queues;
  - streams;
  - concurrency;
  - caches.
- Diagnose memory leaks.
- Use heap snapshots and allocation profiling conceptually.
- Analyze retaining paths.
- Distinguish allocation rate from retained size.
- Understand shallow size versus retained size.
- Explain dominators at a conceptual level.
- Design memory-safe application structures.
- Design bounded caches and queues.
- Design lifecycle-driven cleanup.
- Explain production memory observability.
- Defend memory-management decisions at principal-engineer depth.

### Mastery Gate

> Understand → Explain → Predict → Implement → Debug → Apply → Compare → Defend

---

# 2. Prerequisites

Required:

- Chapter 12 — Execution Contexts / Execution Model
- Chapter 13 — Closures
- Chapter 15 — Objects / Property Semantics
- Chapter 17 — Prototypes / Prototype Chains
- Chapter 24 — Objects / Map / Set / WeakMap / WeakSet
- Chapter 27 — Typed Arrays / Binary Data
- Chapter 28 — JSON / Serialization / Structured Clone
- Chapter 30 — Resource Management / Cleanup
- Chapter 31 — Asynchronous JavaScript Fundamentals
- Chapter 35 — Promises
- Chapter 36 — Async/Await
- Chapter 37 — Cancellation / Abort
- Chapter 38 — Async Iteration / Streaming
- Chapter 39 — Concurrency / Parallelism
- Chapter 41 — ECMAScript Specification Architecture
- Chapter 44 — Realms, Agents, and Execution Isolation

Strongly related:

- Chapter 19 — Proxy / Reflect / Metaprogramming
- Chapter 25 — Iterables / Iterators
- Chapter 26 — Generators / Async Generators
- Chapter 29 — Errors / Error Handling
- Chapter 43 — Ordinary Object Internal Methods

Builds directly toward:

- Chapter 46 — Weak References / Finalization
- Chapter 47 — JavaScript Engine Architecture
- Chapter 48 — V8 Internals / Optimization
- Chapter 52 — Web Workers / Concurrency
- Chapter 53 — Web Streams / Data Flow
- Chapter 60 — Node Streams
- Chapter 61 — Worker Threads / Child Processes / Cluster
- Chapter 62 — Process Lifecycle
- Chapter 63 — Async Context / Diagnostics
- Chapter 78 — Production JS Architecture
- Chapter 83 — Observability
- Chapter 84 — Reliability
- Chapter 85 — Performance
- Chapter 86 — Testing
- Chapter 88 — Debugging Methodology
- Chapter 89 — Code Review / Refactoring
- Chapter 98 — Anti-Patterns / Failure Modes
- Chapter 100 — Cost Model / Tradeoffs
- Chapter 101 — Real-world Production Scenarios
- Chapter 108 — Cache System
- Chapter 110 — Production JavaScript Backend
- Chapter 111 — Large-scale JavaScript Platform
- Chapter 121 — System Design

---

# 3. What Is It?

JavaScript uses automatic memory management.

Application code creates values:

```js
const user = {
  name: "A",
  profile: {
    age: 30
  }
};
```

The engine must store the data somewhere and eventually reclaim memory that is no longer needed.

The central concept is **reachability**.

A simplified model:

```text
GC Roots
   ↓
reachable objects
   ↓
reachable objects
   ↓
reachable objects
```

Anything not reachable through the engine's relevant roots can become eligible for garbage collection.

This gives the core lifecycle:

```text
allocate
   ↓
use
   ↓
retain or release
   ↓
become unreachable
   ↓
GC eventually reclaims
```

Important:

> Becoming unreachable makes memory collectible; it does not mean reclamation happens immediately.

---

# 4. Why Does It Exist?

Without automatic memory management, application code would need to explicitly free most objects.

That creates difficult failure modes:

```text
use after free
double free
dangling pointer
manual lifetime bugs
```

JavaScript instead provides:

```text
automatic reclamation
```

This improves developer safety.

But automatic memory management does not eliminate memory problems.

JavaScript applications can still suffer from:

```text
unbounded cache growth
unreleased subscriptions
retained closures
queued work
large object graphs
long-lived global references
```

GC can reclaim only what the engine determines is unreachable.

Therefore:

> Garbage collection solves automatic reclamation of unreachable memory, not incorrect application ownership.

---

# 5. Mental Model

Think of memory as a graph.

```text
              Root
               │
               ▼
              A
            /   \
           ▼     ▼
          B       C
          │       │
          ▼       ▼
          D       E
```

All nodes reachable from roots are live from the GC's perspective.

Now:

```text
Root
 │
 ▼
 A
 │
 ▼
 B

 C → D → E
```

If `C` is no longer connected to any root:

```text
C → D → E
```

the entire subgraph can become collectible.

The critical point:

> GC operates on reachability, not on whether the programmer subjectively considers an object “finished”.

---

# 6. Core Rules

### Rule 1 — JavaScript memory is automatically managed

Application code normally does not explicitly free ordinary objects.

### Rule 2 — GC is reachability-based in tracing collectors

The exact engine implementation varies, but reachability is the fundamental model.

### Rule 3 — Unreachable does not mean immediately collected

Collection is scheduled by the engine.

### Rule 4 — Long-lived references can keep large graphs alive

One global reference can retain a huge object graph.

### Rule 5 — A memory leak can exist in a garbage-collected language

The application can retain objects longer than intended.

### Rule 6 — Allocation is not the same as retention

High allocation rate can be harmless if objects die quickly.

### Rule 7 — Retained memory is often the more important leak signal

A small number of long-lived objects can retain huge graphs.

### Rule 8 — Closures retain reachable environments

A closure can keep objects alive through its captured references.

### Rule 9 — Event listeners can retain state

A listener registration can keep a callback and reachable data alive.

### Rule 10 — Timers and queued work can retain closures

Unfinished asynchronous work can extend object lifetime.

### Rule 11 — Caches are deliberate retention

A cache becomes a memory leak when it has no effective eviction/boundary policy despite growing demand.

### Rule 12 — Weak references intentionally avoid ordinary strong retention

They are specialized and nondeterministic.

### Rule 13 — GC does not solve external resource lifetime

Sockets, files, GPU resources, workers, and native handles often require explicit lifecycle management.

### Rule 14 — Memory returned to the allocator is not necessarily immediately returned to the OS

Engine memory management has multiple layers.

### Rule 15 — More concurrency generally increases live state

This connects directly to Chapter 39.

---

# 7. Syntax

There is no general JavaScript syntax for “free this object”.

Instead, memory eligibility is usually changed by changing references:

```js
let cache = createLargeObject();

cache = null;
```

or:

```js
array.length = 0;
```

or by ending the owner/lifecycle that held the reference.

Weak-reference APIs include:

```js
new WeakMap();
new WeakSet();
new WeakRef(object);
```

Finalization support includes:

```js
new FinalizationRegistry(callback);
```

These advanced features do not provide deterministic destruction.

---

# 8. Basic Examples

## Example 1 — Reachability

```js
let user = {
  name: "A"
};

const alias = user;

user = null;
```

The object remains reachable through:

```text
alias
```

Therefore setting one reference to `null` does not make the object collectible.

## Example 2 — Object becomes unreachable

```js
let user = {
  name: "A"
};

user = null;
```

If no other relevant references exist, the object can become collectible.

## Example 3 — Closure retention

```js
function createHandler() {
  const largeData = new Array(1_000_000);

  return function handler() {
    return largeData[0];
  };
}

const handler = createHandler();
```

As long as:

```text
handler
```

remains reachable, its closure can keep `largeData` reachable.

## Example 4 — Cache retention

```js
const cache = new Map();

function remember(key, value) {
  cache.set(key, value);
}
```

If entries are never removed and keys continue arriving:

```text
Map size ↑
memory ↑
```

The objects are not a GC leak in the narrow sense—they remain strongly reachable by design.

The application policy is the problem.

## Example 5 — WeakMap

```js
const metadata = new WeakMap();

let object = {};
metadata.set(object, { expensive: true });

object = null;
```

The WeakMap entry does not, by itself, keep the key strongly reachable.

## Example 6 — Timer retention

```js
const largeData = new Array(1_000_000);

const timer = setInterval(() => {
  console.log(largeData.length);
}, 1000);
```

As long as the interval remains active, its callback can keep reachable state alive.

---

# 9. Execution Walkthrough

Consider:

```js
function createSession() {
  const session = {
    user: { id: 1 },
    data: new Array(1_000_000)
  };

  return function getUser() {
    return session.user;
  };
}

let getUser = createSession();
```

### Step 1

`createSession()` allocates the session graph.

```text
session
├── user
└── data
```

### Step 2

The returned function closes over the environment containing `session`.

### Step 3

`getUser` points to the returned function.

### Step 4

The function is reachable.

### Step 5

Its closure/environment is reachable.

### Step 6

The captured `session` becomes reachable.

### Step 7

Therefore:

```text
session
user
data
```

remain reachable.

Now:

```js
getUser = null;
```

If no other reference exists to that function/environment/session graph, the entire graph can become unreachable.

Important:

```text
getUser = null
```

does not directly “free the array”.

It removes one strong path to the object graph.

---

# 10. Internal Mechanics

## 10.1 Allocation

When code creates objects:

```js
{}
[]
new Map()
new ArrayBuffer(...)
```

the engine must arrange memory for the relevant runtime representation.

The physical allocation strategy is engine-specific.

## 10.2 GC roots

A tracing collector starts from roots such as conceptually:

```text
active execution state
global references
reachable runtime structures
engine-managed handles
```

The exact root set is engine-specific.

## 10.3 Mark-and-sweep

A simplified tracing cycle:

```text
1. Start from roots
2. Mark reachable objects
3. Unmarked objects are unreachable
4. Reclaim their memory
```

## 10.4 Mark phase

Traverse:

```text
root → reference → reference → ...
```

and mark visited objects.

## 10.5 Sweep phase

Reclaim memory associated with unreachable objects.

## 10.6 Compaction

A collector may move live objects closer together to reduce fragmentation.

Conceptually:

```text
live live dead live dead live
```

becomes:

```text
live live live live
```

with references updated as required.

## 10.7 Copying collection

Instead of sweeping in place, a collector can copy live objects to another region.

This is often useful for young generations.

## 10.8 Generational GC

A common observation:

> Many objects die young.

Therefore collectors can divide memory into generations.

Conceptually:

```text
Young generation
      ↓ survive
Old generation
```

## 10.9 Nursery / young space

New allocations can begin in a region designed for frequent collection.

## 10.10 Promotion

Objects that survive enough collections may be promoted to older memory regions.

## 10.11 Minor collection

Young-generation collection can be relatively frequent.

## 10.12 Major/old-generation collection

Older memory may require more extensive collection.

Exact terminology varies by engine.

## 10.13 Write barriers

If old objects reference young objects, the collector may need remembered information about cross-generation references.

A write barrier records relevant relationships.

## 10.14 Remembered sets

A collector can maintain metadata describing references that must be considered during partial collection.

## 10.15 Incremental GC

A collector can split work into smaller pieces:

```text
mark
→ application work
→ mark
→ application work
→ ...
```

reducing long uninterrupted pauses.

## 10.16 Concurrent GC

Some GC work can occur concurrently with application execution.

The exact supported phases and synchronization differ by engine.

## 10.17 Stop-the-world phases

Some operations can still require the application to pause.

Do not assume “concurrent GC” means “no pauses”.

## 10.18 Safepoints

The engine can coordinate application/collector transitions at controlled points.

Implementation details are engine-specific.

## 10.19 Allocation fast path

Engines can optimize frequent small allocations through fast allocation paths.

This can make object creation cheap in common cases.

## 10.20 Allocation rate

High allocation rate can cause:

```text
more GC work
more memory traffic
higher CPU use
```

even if retained memory stays moderate.

## 10.21 Retained size

An object's retained size is the memory that would become collectible if that object were removed from the reachable graph, under a particular heap-analysis model.

## 10.22 Shallow size

Shallow size approximates the object's own directly represented memory.

## 10.23 Dominators

In heap graphs, a node can dominate another node if every path from the chosen roots to the downstream object passes through that node.

This helps find high-leverage retention points.

## 10.24 Retaining path

A retaining path shows why an object remains reachable:

```text
Window
→ app state
→ cache
→ request
→ huge payload
```

The important debugging question is:

> Which reference chain is keeping this object alive?

---

# 11. ECMAScript / Specification Semantics

### 11.1 ECMAScript does not define one universal GC algorithm

The language defines semantics and observable behavior.

It does not mandate:

```text
mark-and-sweep
generational GC
specific heap size
specific compaction strategy
```

for every implementation.

### 11.2 Garbage collection is mostly an implementation concern

Whether an unreachable object is physically reclaimed at a particular moment is normally not observable through standard language semantics.

### 11.3 Weak references expose limited GC interaction

ECMAScript provides weak-reference facilities whose semantics intentionally avoid guaranteeing deterministic collection timing.

### 11.4 `WeakMap`

A WeakMap key does not create the same kind of strong reachability relationship as a normal Map key.

### 11.5 `WeakSet`

Likewise, WeakSet membership does not keep the object strongly alive merely because it is a member.

### 11.6 `WeakRef`

A WeakRef can observe whether an object is still reachable, but does not guarantee when collection happens.

### 11.7 Finalization

Finalization is intentionally nondeterministic.

It must not be treated as:

```text
destructor
```

or:

```text
exactly-once cleanup point
```

### 11.8 Host/runtime memory

Browsers and Node.js can also manage:

```text
native resources
network buffers
OS handles
graphics resources
worker resources
```

outside the ordinary JavaScript object-heap model.

---

# 12. Advanced Behavior

## 12.1 Strong references

A normal object reference creates a strong reachability relationship.

```js
const a = {};
const b = a;
```

Both bindings point to the same object.

## 12.2 Object graphs

Real applications are graphs, not isolated objects.

```text
application state
→ users
→ sessions
→ caches
→ requests
→ buffers
```

A single root can retain enormous memory.

## 12.3 Closure capture

Closures are not inherently leaks.

They become retention problems when a long-lived closure unnecessarily captures large state.

## 12.4 Capturing more than necessary

Compare:

```js
function create() {
  const huge = createHugeData();

  return () => huge.id;
}
```

The closure can retain the entire reachable `huge` graph.

A design that extracts only the needed primitive may reduce retention:

```js
function create() {
  const huge = createHugeData();
  const id = huge.id;

  return () => id;
}
```

This is a design consideration, not a universal guarantee about compiler behavior.

## 12.5 Event listener leaks

A component can become retained through:

```text
global event target
→ listener
→ component callback
→ component state
→ DOM/application graph
```

## 12.6 Subscription leaks

Reactive/async subscriptions can keep:

```text
callbacks
buffers
state
network connections
```

alive.

## 12.7 Timer leaks

Long-lived intervals can keep callbacks and their captured state alive.

## 12.8 Queue retention

An unbounded task queue can retain millions of:

```text
closures
payloads
buffers
metadata
```

## 12.9 Promise retention

An unresolved Promise with attached reactions can retain state reachable from those reactions.

## 12.10 Async function retention

Suspended async functions can retain variables needed for future continuation.

Therefore concurrency can increase live memory.

## 12.11 Await and lifetime

A suspended function can keep part of its local state reachable until it resumes or becomes otherwise unreachable.

## 12.12 Streaming memory

Streams and async iterators can retain:

```text
buffers
pending chunks
consumer callbacks
producer state
```

## 12.13 Backpressure

Without backpressure:

```text
producer rate > consumer rate
→ queued memory ↑
```

## 12.14 Cache retention

Caches intentionally create strong references.

A production cache should have explicit policies such as:

```text
max size
TTL
LRU
LFU
admission
eviction
```

## 12.15 Memoization

Memoization can leak memory if argument/result references remain forever.

## 12.16 DOM retention

A detached node can remain alive if JavaScript still references it.

Conceptually:

```text
JS state
→ detached DOM subtree
```

The node is not collectible merely because it is no longer attached to the document.

## 12.17 Native/off-heap memory

Objects can reference memory maintained outside the ordinary JS heap.

Examples can include:

```text
ArrayBuffer backing stores
native resources
buffers
GPU resources
```

The exact accounting depends on the runtime/engine.

## 12.18 ArrayBuffer

An ArrayBuffer can own or reference storage whose memory behavior differs from the ordinary object header/property graph.

## 12.19 SharedArrayBuffer

Shared memory has lifetime implications across Agents.

## 12.20 Transfer

Transferring a resource can change which execution context owns usable access without duplicating the underlying payload.

## 12.21 Cloning

Cloning can duplicate object graphs and therefore increase memory.

## 12.22 Serialization buffers

Temporary serialized representations can create short-lived allocation spikes.

## 12.23 Worker memory

A Worker/Agent can have its own runtime memory.

## 12.24 Process memory

Child processes have separate memory spaces.

## 12.25 Cross-Realm memory

Multiple Realms can create separate intrinsic graphs and additional reachable state.

## 12.26 WeakMap use case

Metadata associated with object lifetime is a classic WeakMap use case:

```js
const metadata = new WeakMap();

function attachMetadata(object, metadataValue) {
  metadata.set(object, metadataValue);
}
```

The metadata does not need to keep the key alive.

## 12.27 WeakMap misuse

WeakMap is not a general-purpose “memory leak prevention” switch.

If another strong reference keeps the key alive, its WeakMap entry remains relevant.

## 12.28 WeakRef misuse

Do not use WeakRef as a deterministic cache eviction mechanism.

## 12.29 Finalization misuse

Do not use finalization to release critical resources:

```text
file
socket
database connection
lock
```

Use explicit lifecycle management.

## 12.30 GC scheduling pressure

The engine balances:

```text
allocation
CPU
latency
memory
```

and may change when/how it collects.

## 12.31 Heap growth

A growing heap does not automatically mean a leak.

It can reflect:

```text
temporary workload
heap reservation
GC heuristics
fragmentation
legitimate retained state
```

## 12.32 Leak signature

A common leak signature:

```text
after GC
retained baseline memory
keeps increasing
```

across repeated identical workload cycles.

## 12.33 Allocation churn signature

Another pattern:

```text
memory rises
GC occurs
memory drops substantially
```

but CPU usage becomes high due to allocation/collection churn.

## 12.34 Fragmentation

Free memory can exist but be poorly arranged for some allocation patterns.

Compaction can help.

## 12.35 Memory pressure

The runtime may trigger more aggressive GC under pressure.

## 12.36 OOM

If useful memory demand exceeds available limits, the process/runtime can fail even in a garbage-collected language.

---

# 13. Edge Cases

- Setting one reference to `null` does not guarantee collection.
- An alias can keep an object alive.
- A closure can retain an object graph.
- A timer can retain a closure.
- An event listener can retain application state.
- A Promise reaction can retain state while pending.
- An async function can retain local state while suspended.
- A queue can retain large payloads.
- A cache can intentionally retain memory.
- A detached DOM subtree can remain reachable from JavaScript.
- A WeakMap does not guarantee immediate deletion/collection visibility.
- WeakRef dereferencing can change over time.
- Finalization timing is nondeterministic.
- Garbage collection does not imply immediate OS memory release.
- Multiple Realms can increase memory without being a leak.
- Multiple Workers can legitimately consume significant memory.
- Shared memory can remain allocated while multiple Agents can reach it.
- Transferring data can change ownership instead of duplicating it.
- Cloning large graphs can temporarily multiply memory consumption.
- External/native memory can be significant even when JS heap usage looks acceptable.
- High allocation rate can cause GC pressure without a retention leak.
- A heap snapshot can itself change timing/memory behavior.
- Debugging tools may retain or alter objects in ways that complicate measurement.
- Production memory behavior can differ between development and optimized builds.
- JIT optimizations can change physical allocation behavior without changing language semantics.
- An object that appears small can retain a very large subgraph.
- A cache key itself can retain a large graph if object keys are used and remain reachable.
- Weak collections are not iterable because exposing their live membership would conflict with their weak semantics.

---

# 14. Common Misconceptions

### Misconception 1 — “GC frees everything I no longer use.”

No. It reclaims memory that is unreachable according to the collector's model.

### Misconception 2 — “Setting a variable to null immediately frees memory.”

No.

### Misconception 3 — “Garbage-collected languages cannot leak memory.”

False.

### Misconception 4 — “Any rising heap is a leak.”

No.

### Misconception 5 — “GC runs after every function.”

No.

### Misconception 6 — “GC has a fixed schedule developers can rely on.”

No.

### Misconception 7 — “WeakMap immediately removes an entry when a key is unreachable.”

Do not rely on deterministic timing.

### Misconception 8 — “WeakRef is a safe cache.”

It is nondeterministic and should not be used as a primary cache policy.

### Misconception 9 — “FinalizationRegistry is a destructor.”

No.

### Misconception 10 — “More GC always means less memory.”

More GC can increase CPU cost and does not solve strong retention.

### Misconception 11 — “Heap memory is the only memory that matters.”

No. Native/off-heap resources can be significant.

### Misconception 12 — “Workers share one heap.”

Not as a general ordinary-object model.

### Misconception 13 — “Cloning is free.”

No.

### Misconception 14 — “Transfer duplicates data.”

Not necessarily; transfer semantics can move usable ownership/access.

### Misconception 15 — “Object identity is irrelevant to memory.”

Identity determines how references connect the object graph.

### Misconception 16 — “Deleting properties immediately shrinks the process memory.”

Not necessarily.

### Misconception 17 — “A DOM node removed from the document is automatically collected.”

Not if it remains reachable through JavaScript/runtime references.

### Misconception 18 — “GC solves resource cleanup.”

No.

---

# 15. Common Mistakes

1. Treating garbage collection as an explicit destructor.
2. Using `null` assignments as a substitute for ownership design.
3. Ignoring closure retention.
4. Ignoring listener/subscription cleanup.
5. Ignoring timer cleanup.
6. Ignoring queue bounds.
7. Ignoring cache eviction.
8. Confusing allocation spikes with leaks.
9. Looking only at total heap size.
10. Ignoring retaining paths.
11. Ignoring retained size.
12. Ignoring off-heap/native memory.
13. Using WeakRef as a primary cache strategy.
14. Using finalization for critical resources.
15. Assuming GC behavior is deterministic.
16. Assuming heap size equals RSS/process memory.
17. Ignoring worker/process memory.
18. Ignoring cloning/serialization costs.
19. Ignoring concurrency-driven live state.
20. Benchmarking memory without controlling workload and lifecycle.

---

# 16. Comparison With Related Concepts

| Concept | Main purpose |
|---|---|
| Allocation | Obtain memory for runtime state |
| Reachability | Determine whether state remains connected to roots |
| GC | Reclaim unreachable memory |
| Mark-and-sweep | Trace and reclaim unreachable objects |
| Generational GC | Optimize for different object lifetimes |
| Incremental GC | Spread collector work over time |
| Concurrent GC | Perform some GC work alongside application execution |
| Compaction | Reduce fragmentation by relocating live objects |
| Strong reference | Keeps object reachable through the reference graph |
| Weak reference | Does not create the same strong retention relationship |
| WeakMap | Weakly associates metadata with object keys |
| WeakRef | Weakly observes an object reference |
| FinalizationRegistry | Receives nondeterministic cleanup notifications |
| Cache eviction | Application-level memory policy |
| Resource cleanup | Explicit lifecycle management for external resources |

### Memory leak vs allocation churn

```text
Leak:
retained baseline keeps growing

Churn:
allocation/GC repeatedly grows and shrinks
```

### Heap vs process memory

```text
JS heap:
managed language objects

Process memory:
heap + native + stacks + executable/code + runtime structures + other resources
```

### Strong vs weak

```text
strong:
contributes to reachability

weak:
does not create the same keeping-alive relationship
```

### GC vs resource management

```text
GC:
memory reclamation

resource management:
lifecycle of sockets/files/workers/locks/etc.
```

### Generational vs incremental

```text
generational:
organize collection by object age

incremental:
split collector work into smaller units
```

These are orthogonal ideas and can be combined.

---

# 17. Performance Considerations

## 17.1 Allocation rate

High allocation rates can increase:

```text
GC frequency
CPU
memory bandwidth
```

## 17.2 Young objects

Short-lived allocation is often optimized well by generational strategies.

## 17.3 Promotion

Objects surviving repeated collections can become more expensive to manage in older generations.

## 17.4 Major collection

Large old-generation collections can require significant CPU and may affect latency.

## 17.5 Incremental work

Incremental GC can reduce long pauses but adds bookkeeping/scheduling overhead.

## 17.6 Concurrent work

Concurrent GC can reduce application pauses but competes for CPU/memory resources.

## 17.7 Compaction

Compaction reduces fragmentation but requires moving/updating live objects and therefore has costs.

## 17.8 Retained graphs

Large retained graphs increase:

```text
GC scanning
heap size
memory footprint
```

## 17.9 Allocation churn

Creating many temporary arrays/objects in hot loops can increase GC pressure.

## 17.10 Object lifetime

Long-lived objects are not inherently bad, but they increase old-generation/retention pressure.

## 17.11 Data structures

Choice of:

```text
Array
Object
Map
Set
typed array
```

affects memory layout and behavior differently by engine.

## 17.12 Strings

String representation and substring behavior can have engine-specific memory implications.

Do not generalize from one engine.

## 17.13 Large buffers

Large binary buffers can consume substantial memory outside the ordinary JS object representation.

## 17.14 Concurrency

Higher concurrency often means:

```text
more active Promises
more buffers
more closures
more queued work
```

which increases live memory.

## 17.15 Serialization

Large structured-clone/serialization operations can create temporary memory spikes.

## 17.16 Measurement

Measure:

```text
allocation rate
heap used
retained heap
GC time
GC frequency
pause time
RSS/process memory
external memory
queue depth
cache size
```

---

# 18. Memory Considerations

## 18.1 Memory budget

A production service should define:

```text
baseline
+
per-request live memory
+
cache
+
queue
+
runtime overhead
+
safety margin
```

## 18.2 Retained size

Identify objects whose removal would release substantial subgraphs.

## 18.3 Dominator analysis

Look for high-leverage roots such as:

```text
global cache
singleton
listener registry
subscription manager
pending queue
```

## 18.4 Lifecycle ownership

Every long-lived object should have a clear owner.

## 18.5 Bounded queues

Do not allow:

```text
producer > consumer
```

to create infinite retention.

## 18.6 Bounded caches

Use:

```text
max entries
max bytes
TTL
eviction
```

as appropriate.

## 18.7 Cleanup

Explicitly release:

```text
listeners
intervals
subscriptions
workers
streams
handles
```

## 18.8 Large closures

Avoid capturing entire request/state objects when only one small value is needed.

## 18.9 Weak collections

Use them for metadata keyed by object identity when the metadata should follow the key's lifetime.

## 18.10 External memory

Track runtime-specific external/native memory separately from JS heap where possible.

---

# 19. Security Considerations

### 19.1 Memory exhaustion

Attackers can cause:

```text
huge allocations
unbounded queues
unbounded caches
```

### 19.2 Request amplification

High concurrency can multiply retained state.

### 19.3 Large payloads

Validate and bound payload sizes.

### 19.4 Cache abuse

User-controlled cache keys can grow caches.

### 19.5 Upload/buffer abuse

Large bodies and buffers can cause process-level memory pressure.

### 19.6 Worker exhaustion

Attackers can create excessive workers/tasks if admission is not controlled.

### 19.7 Finalization assumptions

Security-sensitive cleanup must not depend on nondeterministic finalization.

### 19.8 Side-channel considerations

GC timing and shared memory can participate in side-channel analysis.

Exact threat models are runtime/platform-specific.

### 19.9 Heap snapshots

Diagnostic snapshots can contain sensitive data and must be handled securely.

### 19.10 Memory retention of secrets

Long-lived buffers/objects can retain sensitive data longer than necessary.

### 19.11 Cross-tenant retention

Shared caches/queues must not accidentally retain another tenant's data beyond its intended lifecycle.

### 19.12 Denial of service

A service can remain logically correct while becoming unavailable because memory/GC pressure makes progress too expensive.

---

# 20. Production Usage

## 20.1 Memory leak investigation

Use this workflow:

```text
1. Reproduce a stable workload.
2. Establish a baseline.
3. Force equivalent workload cycles.
4. Observe post-GC/steady-state memory.
5. Capture heap snapshots.
6. Compare retained objects.
7. inspect retaining paths.
8. Find the owner keeping them alive.
9. Fix lifecycle/ownership.
10. Verify the plateau returns.
```

## 20.2 Browser memory profiling

Investigate:

```text
event listeners
DOM retention
closures
component state
timers
subscriptions
WebSocket buffers
```

## 20.3 Node.js memory profiling

Investigate:

```text
heap
external/native memory
buffers
queues
caches
worker memory
```

## 20.4 Cache design

Prefer explicit policy:

```text
max size
max bytes
TTL
eviction
admission
```

rather than infinite retention.

## 20.5 Queue design

Use:

```text
bounded capacity
backpressure
drop/reject policy
dead-letter policy
```

where appropriate.

## 20.6 Worker design

Track:

```text
worker count
per-worker memory
queue depth
task duration
termination
```

## 20.7 Streaming

Bound:

```text
buffer
concurrency
pending chunks
```

## 20.8 Large data processing

Prefer:

```text
streaming
chunking
transfer
bounded concurrency
```

over loading everything into memory.

## 20.9 Lifecycle architecture

Tie object lifetime to:

```text
request
session
component
worker
job
process
```

rather than global state whenever possible.

## 20.10 Observability

Production dashboards should distinguish:

```text
heap used
heap total
RSS
external memory
GC time
GC pause
allocation rate
cache size
queue depth
active workers
```

## 20.11 Alerting

Alert on:

```text
memory growth trend
high GC time
high RSS
near-limit conditions
queue growth
cache growth
OOM/restarts
```

not merely on one instantaneous heap number.

## 20.12 Deployment behavior

Compare memory after:

```text
startup
warmup
steady state
traffic spike
traffic recovery
long idle
```

A leak often becomes visible only across repeated cycles.

---

# 21. Implementation From Scratch

The goal is a teaching GC model.

## Stage 1 — Object Graph

Implement:

```js
const heap = new Map();
```

where objects have IDs and references.

## Stage 2 — Roots

Maintain:

```text
root set
```

and object references.

## Stage 3 — Mark

Implement graph traversal:

```text
roots
→ reachable objects
→ reachable objects
```

Mark visited IDs.

## Stage 4 — Sweep

Delete unmarked objects from the simulated heap.

## Stage 5 — Retaining Path

Given an object ID, output:

```text
root
→ ...
→ target
```

## Stage 6 — Dominator Approximation

Compute which nodes dominate high-retention subgraphs in the teaching graph.

## Stage 7 — Generational Model

Create:

```text
young
old
```

spaces.

Promote objects that survive simulated collections.

## Stage 8 — Remembered Set

Track old→young references.

## Stage 9 — Incremental Marking

Split traversal into bounded work units.

## Stage 10 — Allocation Profiler

Track:

```text
allocations
bytes
object lifetime
survivor count
promotion
```

## Stage 11 — Leak Laboratory

Build:

```text
global root
→ cache
→ retained objects
```

and remove the retention path.

## Stage 12 — Production-Oriented Memory Model

Add separate accounting for:

```text
JS heap
external memory
queues
cache
workers
buffers
```

---

# 22. Debugging Exercises

### Exercise 1 — Global Retention

```js
globalThis.leak = [];

setInterval(() => {
  globalThis.leak.push(new Array(100_000));
}, 100);
```

Identify the retaining root.

### Exercise 2 — Closure Retention

Create a large object in a closure and store the closure globally.

Find the retaining path.

### Exercise 3 — Event Listener

Attach a listener to a long-lived event target that closes over a large component state.

Remove the component without removing the listener.

Diagnose the leak.

### Exercise 4 — Timer

Create a recurring interval that captures a large structure.

Cancel it and compare memory behavior.

### Exercise 5 — Cache

Build an infinite Map cache.

Measure:

```text
entries
heap
RSS
```

Then add eviction.

### Exercise 6 — Queue

Produce tasks faster than consumers can process.

Measure queue memory.

### Exercise 7 — Promise Retention

Create a pending Promise with callbacks that retain large state.

Determine what keeps the state reachable.

### Exercise 8 — DOM Retention

Create a detached subtree and hold it from JavaScript state.

Release the reference and compare.

### Exercise 9 — WeakMap

Associate metadata using WeakMap and compare the semantic retention relationship with Map.

### Exercise 10 — Worker Memory

Create many Workers and measure:

```text
count
memory
startup
shutdown
```

---

# 23. Code Review Exercise

Review:

```js
const cache = new Map();

export async function loadUser(id) {
  if (cache.has(id)) {
    return cache.get(id);
  }

  const result = await fetchUser(id);

  cache.set(id, result);

  return result;
}
```

Identify the memory-policy questions:

```text
How large can the cache become?
How long should entries remain?
What happens after millions of IDs?
Are values large?
Can tenants poison the cache?
What is the eviction policy?
What is the maximum memory budget?
Does concurrency cause duplicate loads?
```

Then redesign it with an explicit bounded-memory policy.

---

# 24. Interview Questions

## Foundational

1. What is garbage collection?
2. What is reachability?
3. What are GC roots?
4. What is a memory leak in JavaScript?
5. Can a garbage-collected application leak memory?
6. What is mark-and-sweep?
7. Why are generations useful?
8. What is compaction?
9. What is allocation?
10. What is retention?

## Intermediate

11. What is generational GC?
12. What is promotion?
13. What is a write barrier?
14. What is incremental GC?
15. What is concurrent GC?
16. What is retained size?
17. What is shallow size?
18. What is a retaining path?
19. What is a dominator?
20. Why can closures retain memory?

## Advanced

21. Why can a timer leak memory?
22. Why can event listeners leak memory?
23. Why can pending Promises retain memory?
24. Why can queues cause memory exhaustion?
25. Why can caches become leaks?
26. What is the difference between heap and RSS?
27. What is external/native memory?
28. Why doesn't GC immediately return all memory to the OS?
29. Why is WeakMap useful?
30. Why are WeakRef and finalization nondeterministic?

## Principal-Level

31. Diagnose a production service whose heap grows indefinitely.
32. Distinguish allocation churn from retention leak.
33. Design a memory budget for a Node.js service.
34. Design bounded cache and queue memory.
35. Explain how concurrency affects retained memory.
36. Design a memory leak investigation workflow.
37. Explain worker/process memory trade-offs.
38. Explain clone vs transfer vs shared memory from a memory perspective.
39. Identify whether a memory problem is JS heap, external memory, or OS/process pressure.
40. Defend:

> “Memory management is primarily an ownership and lifetime problem; GC is the reclamation mechanism that handles unreachable memory.”

---

# 25. Predict-the-Output Exercises

For each:

```text
Predict
→ Run
→ Inspect reachability
→ Inspect memory if possible
→ Explain
```

### Exercise A

```js
let a = { value: 1 };
let b = a;

a = null;

console.log(b.value);
```

Explain why the object remains reachable.

### Exercise B

```js
let a = { value: 1 };

a = null;

console.log("done");
```

Explain what can and cannot be concluded about GC timing.

### Exercise C

```js
const parent = {};
const child = { parent };

console.log(child.parent === parent);
```

Explain the object graph.

### Exercise D

```js
const create = () => {
  const large = new Array(100_000);

  return () => large.length;
};

const fn = create();
```

Which reference path can retain `large`?

### Exercise E

```js
const cache = new Map();

for (let i = 0; i < 1000; i++) {
  cache.set(i, new Array(1000));
}
```

What determines whether memory can be reclaimed?

### Exercise F

```js
const weak = new WeakMap();

let key = {};
weak.set(key, { data: new Array(1000) });

key = null;
```

What strong reference remains from the WeakMap key itself?

### Exercise G

```js
const obj = {};
const map = new Map();

map.set(obj, "value");
```

Compare with WeakMap semantics when `obj` loses every other strong reference.

### Exercise H

```js
const values = [];

setInterval(() => {
  values.push(new Array(1000));
}, 100);
```

Identify the root that prevents old entries from becoming unreachable.

---

# 26. Mastery Exercises

### Exercise 1 — Mark-and-Sweep Simulator

Implement:

```text
allocate
addRoot
removeRoot
addReference
removeReference
mark
sweep
```

### Exercise 2 — Retaining Path Analyzer

Given a heap graph, find:

```text
root → target
```

for retained objects.

### Exercise 3 — Leak Detector

Build a test harness that compares post-GC retained memory after repeated identical workloads.

### Exercise 4 — Generational Simulator

Implement:

```text
young
survival
promotion
old
```

### Exercise 5 — Write Barrier Simulator

Track:

```text
old → young
```

references and maintain a remembered set.

### Exercise 6 — Allocation Profiler

Record:

```text
allocation count
bytes
object class/type
age
survival
promotion
```

### Exercise 7 — Cache Budget

Implement a cache with:

```text
max entries
max bytes
TTL
eviction
```

### Exercise 8 — Bounded Queue

Implement:

```text
max queue size
reject/drop policy
```

and measure memory.

### Exercise 9 — Closure Leak Laboratory

Build three variants:

```text
captures huge object
captures small primitive
explicitly releases owner
```

Compare retention.

### Exercise 10 — Principal Memory Architecture

Design memory policy for:

```text
10,000 requests/sec
average request state = 200 KB
P99 latency = 250 ms
cache budget = 2 GB
queue budget = 512 MB
process memory limit = 4 GB
```

Define:

```text
concurrency
queue
cache
buffering
worker count
memory safety margin
observability
OOM strategy
```

Defend every number.

---

# 27. Key Takeaways

1. JavaScript uses automatic memory management for ordinary language objects.
2. Reachability is the core mental model for tracing GC.
3. GC roots anchor the reachable object graph.
4. Becoming unreachable makes memory collectible, not immediately collected.
5. Garbage collection does not prevent logical memory leaks.
6. A memory leak is often unintended retention.
7. Allocation rate and retained memory are different signals.
8. Generational GC exploits the common short lifetime of many objects.
9. Promotion moves survivors toward older-generation treatment.
10. Write barriers help collectors track references across generations.
11. Incremental and concurrent strategies reduce some pause/latency costs but do not eliminate all pauses.
12. Compaction can reduce fragmentation.
13. Heap size is not the same as process/RSS memory.
14. External/native memory can be significant.
15. Closures can retain reachable environments.
16. Timers, event listeners, subscriptions, pending async work, queues, and caches can retain state.
17. Concurrency can increase live memory because more work is simultaneously in progress.
18. Cloning can duplicate memory.
19. Transfer can move ownership without the same duplication cost for supported resources.
20. Shared memory changes the ownership model and can span Agents.
21. WeakMap provides weak object-key association.
22. WeakRef and finalization are nondeterministic and advanced.
23. Finalization is not a deterministic destructor.
24. Explicit resource management remains necessary for non-memory resources.
25. The central principle is:

> Garbage collection answers “what memory is no longer reachable?”; application architecture must answer “what should still be reachable, for how long, and who owns it?”

---

# 28. Concept Connections

## Depends On

- Chapter 12 — Execution Contexts / Execution Model
- Chapter 13 — Closures
- Chapter 15 — Objects / Property Semantics
- Chapter 17 — Prototypes / Prototype Chains
- Chapter 24 — Objects / Map / Set / WeakMap / WeakSet
- Chapter 27 — Typed Arrays / Binary Data
- Chapter 28 — JSON / Serialization / Structured Clone
- Chapter 30 — Resource Management / Cleanup
- Chapter 31 — Asynchronous JavaScript Fundamentals
- Chapter 35 — Promises
- Chapter 36 — Async/Await
- Chapter 37 — Cancellation / Abort
- Chapter 38 — Async Iteration / Streaming
- Chapter 39 — Concurrency / Parallelism
- Chapter 41 — ECMAScript Specification Architecture
- Chapter 42 — ECMAScript Abstract Operations
- Chapter 43 — Ordinary Object Internal Methods
- Chapter 44 — Realms, Agents, and Execution Isolation

## Builds Toward

- Chapter 46 — Weak References / Finalization
- Chapter 47 — JavaScript Engine Architecture
- Chapter 48 — V8 Internals / Optimization
- Chapter 52 — Web Workers / Concurrency
- Chapter 53 — Web Streams / Data Flow
- Chapter 60 — Node Streams
- Chapter 61 — Worker Threads / Child Processes / Cluster
- Chapter 62 — Process Lifecycle
- Chapter 63 — Async Context / Diagnostics
- Chapter 78 — Production JS Architecture
- Chapter 83 — Observability
- Chapter 84 — Reliability
- Chapter 85 — Performance
- Chapter 86 — Testing
- Chapter 88 — Debugging Methodology
- Chapter 89 — Code Review / Refactoring
- Chapter 98 — Anti-Patterns / Failure Modes
- Chapter 100 — Cost Model / Tradeoffs
- Chapter 101 — Real-world Production Scenarios
- Chapter 108 — Cache System
- Chapter 110 — Production JavaScript Backend
- Chapter 111 — Large-scale JavaScript Platform
- Chapter 121 — System Design

## Related Concepts

- Allocation
- Heap
- Reachability
- GC roots
- Mark-and-sweep
- Generational GC
- Young generation
- Old generation
- Promotion
- Write barrier
- Remembered set
- Incremental GC
- Concurrent GC
- Stop-the-world
- Safepoint
- Compaction
- Fragmentation
- Retained size
- Shallow size
- Dominator
- Retaining path
- Memory leak
- Allocation churn
- Closure retention
- Queue retention
- Cache retention
- WeakMap
- WeakSet
- WeakRef
- FinalizationRegistry
- External memory
- Shared memory
- Transfer
- Structured clone

## Concepts Revisited

This chapter revisits:

- closures;
- objects;
- collections;
- async functions;
- Promises;
- queues;
- streams;
- concurrency;
- Realms;
- Agents;
- structured cloning;
- transfer;
- resources.

## Why This Chapter Matters Later

Chapter 45 changes the meaning of “performance” from:

```text
“How fast is this code?”
```

to:

```text
“How much state exists?
How long does it remain reachable?
What retains it?
How expensive is collection?
What is the memory ceiling?
What happens during overload?”
```

This becomes the foundation for:

```text
engine architecture
GC optimization
memory profiling
production reliability
large-scale service design
```

---

# 29. Completion Criteria

## Conceptual Understanding

- [ ] Define allocation.
- [ ] Define reachability.
- [ ] Define GC roots.
- [ ] Explain mark-and-sweep.
- [ ] Explain generational GC.
- [ ] Explain promotion.
- [ ] Explain write barriers.
- [ ] Explain remembered sets.
- [ ] Explain incremental GC.
- [ ] Explain concurrent GC.
- [ ] Explain compaction.
- [ ] Explain fragmentation.
- [ ] Explain retained size.
- [ ] Explain shallow size.
- [ ] Explain retaining path.
- [ ] Explain dominators.
- [ ] Explain allocation churn.
- [ ] Explain memory leak.
- [ ] Explain heap vs process memory.
- [ ] Explain external/native memory.
- [ ] Explain WeakMap.
- [ ] Explain WeakRef.
- [ ] Explain FinalizationRegistry.

## Predictive Mastery

- [ ] Predict strong-reference reachability.
- [ ] Predict alias retention.
- [ ] Predict closure retention.
- [ ] Predict timer retention.
- [ ] Predict listener retention.
- [ ] Predict queue retention.
- [ ] Predict cache growth.
- [ ] Predict WeakMap retention semantics.
- [ ] Predict clone memory impact.
- [ ] Predict transfer memory impact.
- [ ] Predict concurrency-driven live-state growth.
- [ ] Distinguish leak from allocation churn.

## Implementation

- [ ] Implement mark-and-sweep simulator.
- [ ] Implement retaining-path analysis.
- [ ] Implement generational simulator.
- [ ] Implement remembered set.
- [ ] Implement allocation profiler.
- [ ] Implement bounded cache.
- [ ] Implement bounded queue.
- [ ] Implement leak-detection harness.
- [ ] Build closure-retention laboratory.
- [ ] Build production memory-budget model.

## Debugging

- [ ] Diagnose global retention.
- [ ] Diagnose closure retention.
- [ ] Diagnose timer leaks.
- [ ] Diagnose listener leaks.
- [ ] Diagnose subscription leaks.
- [ ] Diagnose queue growth.
- [ ] Diagnose cache leaks.
- [ ] Diagnose detached DOM retention.
- [ ] Diagnose Promise retention.
- [ ] Diagnose worker memory growth.
- [ ] Diagnose external memory growth.
- [ ] Distinguish heap leak from RSS growth.

## Production Engineering

- [ ] Define memory budget.
- [ ] Define cache budget.
- [ ] Define queue budget.
- [ ] Define buffer limits.
- [ ] Define concurrency memory impact.
- [ ] Define worker memory limits.
- [ ] Define eviction.
- [ ] Define lifecycle ownership.
- [ ] Define memory observability.
- [ ] Define overload behavior.
- [ ] Define OOM/restart strategy.

## Interview Readiness

- [ ] Explain GC.
- [ ] Explain reachability.
- [ ] Explain generational collection.
- [ ] Explain incremental/concurrent GC.
- [ ] Explain memory leaks.
- [ ] Explain closures as retention paths.
- [ ] Explain WeakMap.
- [ ] Explain WeakRef/finalization limitations.
- [ ] Explain heap vs RSS.
- [ ] Diagnose a production memory leak.

## Track A — Core Theory

- [ ] Reachability model understood.
- [ ] GC roots understood.
- [ ] Tracing collection understood.
- [ ] Generational model understood.
- [ ] Incremental/concurrent collection understood.
- [ ] Retention analysis understood.
- [ ] Weak-reference model understood.
- [ ] Heap vs process memory understood.

## Track B — Implementation

- [ ] Guided GC model completed.
- [ ] Partially guided model completed.
- [ ] No-reference simulator completed.
- [ ] Edge-case hardened simulator completed.
- [ ] Memory profiler reviewed.

## Track C — Interview / Reasoning

- [ ] Prediction exercises completed.
- [ ] Leak debugging completed.
- [ ] Code review completed.
- [ ] Heap-retention analysis completed.
- [ ] Production memory budgeting completed.
- [ ] Principal-level architecture defense completed.

### Mastery Status

```text
[ ] Not Started
[~] In Progress
[?] Needs Revision
[+] Completed
[*] Mastered
```

Do not mark `[*] Mastered` until the learner can independently:

> Understand → Explain → Predict → Implement → Debug → Apply → Compare → Defend

---

# Chapter 45 — Revision / Retrieval Record

## Retrieval Prompts

1. What is memory management?
2. What is allocation?
3. What is reachability?
4. What are GC roots?
5. What is mark-and-sweep?
6. Why does generational GC work well for many workloads?
7. What is promotion?
8. What is a write barrier?
9. What is a remembered set?
10. What is incremental GC?
11. What is concurrent GC?
12. What is compaction?
13. What is fragmentation?
14. What is a retaining path?
15. What is retained size?
16. What is shallow size?
17. What is a dominator?
18. What is a memory leak in JavaScript?
19. Why can closures retain large objects?
20. Why can timers retain state?
21. Why can event listeners retain state?
22. Why can subscriptions retain state?
23. Why can pending Promises retain state?
24. Why can queues exhaust memory?
25. Why can caches grow without bound?
26. What is allocation churn?
27. What is the difference between heap and RSS?
28. What is external/native memory?
29. Why doesn't GC immediately return memory to the OS?
30. How does concurrency affect live memory?
31. What does WeakMap change?
32. Why is WeakRef nondeterministic?
33. Why is FinalizationRegistry not a destructor?
34. How does cloning affect memory?
35. How does transfer affect memory?
36. How does shared memory affect memory ownership?
37. How would you debug a production memory leak?
38. How would you design a memory budget?

## Weak Areas

```text
-
-
-
```

## Revision Queue

```text
- [ ] Revisit reachability
- [ ] Revisit GC roots
- [ ] Revisit mark-and-sweep
- [ ] Revisit generational GC
- [ ] Revisit promotion
- [ ] Revisit write barriers
- [ ] Revisit remembered sets
- [ ] Revisit incremental GC
- [ ] Revisit concurrent GC
- [ ] Revisit compaction
- [ ] Revisit retaining paths
- [ ] Revisit retained size
- [ ] Revisit memory leaks
- [ ] Revisit closure/listener/timer retention
- [ ] Revisit queue/cache memory
- [ ] Revisit WeakMap
- [ ] Revisit WeakRef/finalization
- [ ] Revisit heap vs RSS
- [ ] Revisit external memory
```

## Assessment History

```text
Date:
Score:
Weak Areas:
Next Review:
```

## Chapter Status

```text
[+] Expanded
[ ] Reviewed
[ ] Practiced
[ ] Assessed
[ ] Mastered
```

---

# Chapter 45 — Canonical References and Source Discipline

Use this source hierarchy:

1. ECMAScript specification — language-level semantics for objects, WeakMap, WeakSet, WeakRef, FinalizationRegistry, Agents, and related memory-visible behavior.
2. Engine documentation — garbage collection and memory implementation details.
3. V8 documentation/source — V8-specific heap, GC, allocation, and optimization behavior.
4. Browser runtime documentation — DOM/runtime memory behavior.
5. Node.js documentation — process/heap/external-memory behavior and worker/process memory.
6. Profiling tools/documentation — heap snapshots, allocation profiles, retaining paths, and production diagnostics.

Always distinguish:

```text
ECMAScript semantic guarantee
vs
GC algorithm
vs
engine heap representation
vs
OS/process memory
vs
application ownership policy
```

Do not claim that ECMAScript mandates one garbage collector.

Do not claim that an unreachable object is immediately collected.

Do not use GC timing as deterministic application logic.

Do not use WeakRef or finalization as replacements for explicit resource management.

Do not equate:

```text
heap used
```

with:

```text
process RSS
```

---

# Chapter 45 — Completion Snapshot

```text
Chapter: 45
Title: JavaScript Memory and Garbage Collection
Part: VIII — JavaScript Engine
Status: [+] Expanded
Track A: [ ] Core Theory
Track B: [ ] Implementation
Track C: [ ] Interview / Reasoning
Mastery: [ ] Not Mastered
```