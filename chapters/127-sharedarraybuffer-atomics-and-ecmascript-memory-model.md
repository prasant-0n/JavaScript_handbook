# Chapter 127 — SharedArrayBuffer, Atomics & ECMAScript Memory Model

> **JavaScript Mastery — Part XXII: Advanced JavaScript Language & Platform**
>
> **Mission:** Understand JavaScript shared memory at specification level: `SharedArrayBuffer`, shared `TypedArray` views, Agents and agent clusters, atomic operations, `Atomics.wait` / `waitAsync` / `notify`, data races, happens-before relationships, sequential consistency, tear-free guarantees, and the boundary between ECMAScript's memory model and host/engine threading.
>
> **Role perspective:** Principal JavaScript Engineer · ECMAScript Language Specialist · Concurrency Engineer · Runtime/Engine Engineer · Systems Programmer · Node.js Worker Engineer · Browser Concurrency Engineer
>
> **Status:** `[ ] Not Started`
>
> **Core rule:** **Shared memory changes JavaScript's mental model from “each execution context owns ordinary state” to “multiple agents can observe one shared memory block under a formal memory model.”**

---

# 1. Learning Objectives

By completing this chapter, you should be able to:

```text
[ ] explain why SharedArrayBuffer exists
[ ] distinguish ArrayBuffer from SharedArrayBuffer
[ ] distinguish ordinary objects from shared memory
[ ] explain shared data blocks conceptually
[ ] explain Agent and Agent Cluster relationships
[ ] explain SharedArrayBuffer sharing across workers/agents
[ ] explain why SharedArrayBuffer is not transferable in the ordinary sense
[ ] explain TypedArray views over shared memory
[ ] explain atomic vs non-atomic shared accesses
[ ] explain Atomics operations
[ ] explain atomic read-modify-write
[ ] explain compareExchange
[ ] explain sequential consistency
[ ] explain data races
[ ] explain why racy programs can produce surprising values
[ ] explain what the ECMAScript memory model guarantees
[ ] explain what the ECMAScript memory model does not guarantee
[ ] understand non-atomic shared accesses
[ ] understand atomic alignment/element restrictions
[ ] explain wait/notify coordination
[ ] explain Atomics.wait
[ ] explain Atomics.waitAsync
[ ] explain Atomics.notify
[ ] build a mutex using Atomics
[ ] build a semaphore using Atomics
[ ] build a bounded queue using shared memory
[ ] reason about lost wakeups
[ ] reason about spurious wake-style loops
[ ] distinguish blocking waiting from asynchronous waiting
[ ] understand why Atomics.wait is inappropriate on browser main-thread-like contexts
[ ] understand agent-level parallelism vs ordinary async concurrency
[ ] reason about worker-thread ownership
[ ] explain shared memory security risks
[ ] explain side-channel concerns
[ ] debug race conditions
[ ] implement concurrency primitives safely
[ ] distinguish specification semantics from CPU/cache implementation details
```

---

# 2. Prerequisites

You should already understand:

```text
Chapter 27 — Typed Arrays / Binary Data
Chapter 31 — Async Fundamentals
Chapter 32 — ECMAScript Jobs / Promise Reactions
Chapter 34 — Node Event Loop / libuv
Chapter 37 — Cancellation / Abort
Chapter 44 — Realms / Agents / Execution Isolation
Chapter 45 — Memory / GC
Chapter 47 — JavaScript Engine Architecture
Chapter 52 — Web Workers / Concurrency
Chapter 61 — Worker Threads / Child Processes / Cluster
Chapter 72 — Complexity
Chapter 84 — Reliability
Chapter 85 — Performance
Chapter 123 — ECMAScript Grammar
Chapter 124 — Execution / Completion / References
Chapter 125 — Promise Internals
Chapter 126 — Module Linking / Evaluation
```

Useful systems knowledge:

```text
threads
atomic operations
mutual exclusion
memory ordering
race conditions
queues
condition variables
cache coherence
```

---

# 3. Why Shared Memory Exists

Ordinary JavaScript execution is often easiest to reason about as:

```text
agent A
owns ordinary mutable objects
```

while another agent:

```text
agent B
owns different ordinary mutable objects
```

Communication can use:

```text
messages
structured cloning
transfers
events
Promises
```

Shared memory enables a different architecture:

```text
Agent A ─────┐
             ├── shared memory
Agent B ─────┤
             └── shared memory
Agent C ─────┘
```

This can avoid copying large buffers and enable low-level synchronization.

Costs include:

```text
race conditions
coordination complexity
memory-ordering reasoning
security concerns
debugging difficulty
```

---

# 4. Current ECMAScript Memory Model

The ECMAScript specification defines a formal memory consistency model for:

```text
SharedArrayBuffer-backed TypedArray access
Atomics operations
```

The current specification states that:

```text
atomic accesses are sequentially consistent
```

while data-racy non-atomic accesses can exhibit sequentially inconsistent behavior. It explicitly models memory events and allowed execution graphs. citeturn811838search0

Important:

```text
no data races
→ behavior appears sequentially consistent

data races
→ surprising reorderings/observations may be possible
```

The specification does not define this through a simple executable algorithm; it describes an axiomatic model using constraints over memory events. citeturn811838search0

---

# 5. ArrayBuffer vs SharedArrayBuffer

## ArrayBuffer

Conceptually:

```text
private byte storage
```

and ordinary transfer/detachment semantics can apply.

## SharedArrayBuffer

Conceptually:

```text
shared byte storage
```

accessible by multiple agents that receive a reference to the same shared data block.

The current specification states that `SharedArrayBuffer` instances are not detached in the manner ordinary `ArrayBuffer` instances can be. citeturn811838search3

---

# 6. SharedArrayBuffer Mental Model

Use:

```text
SharedArrayBuffer object
        ↓
shared data block
        ↓
TypedArray/DataView views
        ↓
agents access shared bytes
```

The important distinction:

```text
SharedArrayBuffer object
```

is itself a JavaScript object in each relevant agent/realm context,

while:

```text
shared data block
```

is the underlying shared memory abstraction.

---

# 7. Shared Data Is Not Shared Object Graphs

Consider:

```js
const shared = new SharedArrayBuffer(16);
```

What is shared?

```text
bytes
```

What is not automatically shared?

```text
ordinary JavaScript objects
arrays
Maps
Sets
closures
class instances
DOM nodes
Promise state
```

You share memory through:

```text
TypedArray/DataView views
```

not arbitrary object identity.

---

# 8. Sharing Across Agents

A worker can receive a `SharedArrayBuffer` and create its own view:

```js
const view = new Int32Array(sharedBuffer);
```

The views are separate JavaScript objects.

But they can address the same underlying shared memory.

Conceptually:

```text
Agent A
  viewA
    ↓
  shared bytes
    ↑
  viewB
Agent B
```

---

# 9. Agents

An ECMAScript Agent represents an independent execution context capable of progressing its own execution.

Do not equate:

```text
Agent
```

with:

```text
OS thread
```

The host/engine determines how agents are mapped onto execution resources.

Possible implementation:

```text
one OS thread
```

or:

```text
worker thread
```

or another scheduling mechanism.

The specification defines the semantic agent abstraction.

---

# 10. Agent Cluster

Shared memory requires agents to participate in an appropriate agent cluster.

Conceptually:

```text
Agent A
Agent B
Agent C
   \ | /
 shared-memory relationship
```

The host controls which execution contexts can share memory.

This is why:

```text
same JavaScript program
```

does not imply:

```text
all contexts share every SharedArrayBuffer.
```

---

# 11. SharedArrayBuffer and Transfer

A `SharedArrayBuffer` reference can be made visible to another agent through supported host/worker communication mechanisms.

The semantic goal is:

```text
both agents access the same shared data block
```

rather than:

```text
copy bytes into a new independent ArrayBuffer
```

This is one of its primary performance uses.

---

# 12. Shared TypedArray Views

Valid shared-memory atomic operations operate through appropriate TypedArray views.

Example:

```js
const buffer = new SharedArrayBuffer(4);
const view = new Int32Array(buffer);

Atomics.store(view, 0, 42);
const value = Atomics.load(view, 0);
```

The view provides:

```text
element type
indexing
byte interpretation
```

while:

```text
Atomics
```

provides synchronization/atomic operations.

---

# 13. Supported Atomic Element Types

Atomic operations are constrained to supported integer/bigint typed arrays.

Examples commonly used:

```text
Int8Array
Uint8Array
Int16Array
Uint16Array
Int32Array
Uint32Array
BigInt64Array
BigUint64Array
```

Specific operations have type restrictions.

Do not assume every `TypedArray` method is atomic.

---

# 14. DataView Is Different

A `DataView` can access shared memory, but the `Atomics` API expects a suitable integer TypedArray for its atomic methods.

For synchronization code:

```text
Int32Array
```

is a common control structure.

Do not use:

```text
DataView
```

as a drop-in replacement for:

```text
Atomics operations.
```

---

# 15. Atomic Access

An atomic operation guarantees that its access participates in the ECMAScript memory model's atomic ordering.

Examples:

```js
Atomics.load(view, index);
Atomics.store(view, index, value);
Atomics.add(view, index, value);
Atomics.sub(view, index, value);
Atomics.and(view, index, value);
Atomics.or(view, index, value);
Atomics.xor(view, index, value);
Atomics.exchange(view, index, value);
Atomics.compareExchange(view, index, expected, replacement);
```

The current specification describes atomic operations and `wait`/`notify` as dedicated built-ins. citeturn811838search2

---

# 16. Why Atomicity Matters

Consider:

```js
shared[0] = shared[0] + 1;
```

This is conceptually:

```text
load
+
add
+
store
```

Another agent can interleave between these steps.

This is a lost-update race.

By contrast:

```js
Atomics.add(shared, 0, 1);
```

performs an atomic read-modify-write operation.

---

# 17. Lost Update Example

Suppose:

```text
initial = 0
```

Agent A:

```text
read 0
```

Agent B:

```text
read 0
```

A:

```text
write 1
```

B:

```text
write 1
```

Final:

```text
1
```

Expected from two increments:

```text
2
```

This is the classic read-modify-write race.

---

# 18. Atomic Increment

Use:

```js
Atomics.add(view, 0, 1);
```

Now the increments are serialized with respect to the atomic ordering rules.

Conceptually:

```text
A Atomics.add
B Atomics.add
```

must have an ordering consistent with the atomic model.

This does not make an entire application transaction atomic.

It only provides atomicity for the operation specified.

---

# 19. CompareExchange

`compareExchange` is the foundation of many lock-free structures.

Conceptually:

```text
if memory === expected:
    memory = replacement
return old value
```

Example:

```js
const old = Atomics.compareExchange(
  state,
  0,
  0,
  1
);
```

If the old value was:

```text
0
```

the state becomes:

```text
1
```

If not:

```text
no replacement occurs
```

and the observed old value tells the caller that it lost the race.

---

# 20. CompareExchange as Conditional Ownership

A common mutex acquisition pattern is:

```text
0 = unlocked
1 = locked
```

Then:

```js
Atomics.compareExchange(lock, 0, 0, 1);
```

means:

```text
"change unlocked → locked only if nobody else already acquired it."
```

The caller owns the lock only if the returned value was:

```text
0
```

---

# 21. Atomicity Is Not Mutual Exclusion

Atomic operations provide:

```text
individual atomic state transitions
```

They do not automatically make:

```text
multi-step algorithm
```

safe.

For example:

```js
const x = Atomics.load(state, 0);
const y = Atomics.load(state, 1);

Atomics.store(state, 2, x + y);
```

Other agents can modify:

```text
x
y
```

between reads.

A larger invariant may require:

```text
lock
transaction protocol
versioning
CAS loop
```

---

# 22. Sequential Consistency

The current ECMAScript memory model specifies atomic accesses as sequentially consistent.

Conceptually this means:

```text
all agents agree on one total order of atomic operations
```

that respects each agent's own program order.

This makes correctly synchronized programs much easier to reason about than weaker hardware-specific memory models. citeturn811838search0

---

# 23. Sequential Consistency Is Not “Runs on One Thread”

A common misconception:

```text
"Sequentially consistent means there is only one CPU thread."
```

Incorrect.

Multiple agents can execute concurrently.

The model instead constrains:

```text
how atomic operations can be observed.
```

---

# 24. Data Race

A data race can occur when multiple agents access the same shared memory location and at least one access is a write, without sufficient atomic synchronization.

The ECMAScript memory model specifies possible outcomes rather than declaring the entire program to have undefined behavior. citeturn811838search0

This differs from the simplistic belief:

```text
"JavaScript races are impossible because the language is single-threaded."
```

---

# 25. JavaScript Is Not Necessarily Single-Threaded

A more precise statement is:

```text
one JavaScript agent executes its code sequentially
```

but a program can contain:

```text
multiple agents
```

in appropriate hosts.

Examples:

```text
Web Workers
Node.js worker threads
```

can provide concurrent agents.

With:

```text
SharedArrayBuffer
```

they can coordinate through shared memory.

---

# 26. Ordinary Async Concurrency vs Shared-Memory Concurrency

### Async concurrency

```text
one agent
many operations
interleaving through scheduling
```

### Shared-memory concurrency

```text
multiple agents
shared bytes
actual overlapping execution possible
```

They require different mental models.

---

# 27. Message Passing vs Shared Memory

## Message Passing

```text
Agent A
  ↓ copy/transfer/message
Agent B
```

Advantages:

```text
clear ownership
fewer races
simpler reasoning
```

Costs:

```text
copying
coordination
latency
serialization
```

## Shared Memory

```text
Agent A ──┐
          ├─ shared bytes
Agent B ──┘
```

Advantages:

```text
low-copy communication
high throughput for suitable workloads
fine-grained coordination
```

Costs:

```text
race conditions
synchronization
complexity
debugging
security concerns
```

---

# 28. Atomics.load / Atomics.store

Example:

```js
Atomics.store(state, 0, 1);

const value = Atomics.load(state, 0);
```

Use explicit atomics when shared coordination depends on:

```text
visibility
ordering
synchronization
```

Do not replace all ordinary reads/writes with Atomics mechanically.

Atomic operations have costs and semantic implications.

---

# 29. Read-Modify-Write Operations

Examples:

```js
Atomics.add(...)
Atomics.sub(...)
Atomics.and(...)
Atomics.or(...)
Atomics.xor(...)
Atomics.exchange(...)
Atomics.compareExchange(...)
```

These combine:

```text
read
+
operation
+
write
```

into an atomic memory operation.

---

# 30. Atomic Exchange

Example:

```js
const previous = Atomics.exchange(state, 0, 1);
```

This:

```text
writes 1
```

and returns:

```text
previous value
```

This can be useful for ownership handoff or flag replacement.

---

# 31. Lock-Free Loop With CompareExchange

Conceptual pattern:

```js
while (true) {
  const current = Atomics.load(state, 0);

  const next = update(current);

  if (Atomics.compareExchange(state, 0, current, next) === current) {
    break;
  }
}
```

Meaning:

```text
read current
compute desired new state
attempt conditional update
if another agent changed it:
    retry
```

This is a classic optimistic concurrency pattern.

---

# 32. CAS Loop Costs

CAS loops can fail repeatedly under contention.

Symptoms:

```text
high retry rate
CPU consumption
starvation
tail latency
```

Therefore:

```text
lock-free
```

does not automatically mean:

```text
faster
```

Measure contention.

---

# 33. `Atomics.wait`

`Atomics.wait` allows an agent to wait on an integer shared-memory location.

Conceptually:

```js
Atomics.wait(view, index, expectedValue);
```

means:

```text
if memory does not equal expectedValue:
    return immediately with "not-equal"

otherwise:
    wait until notified / timeout / relevant wake condition
```

It is a synchronization primitive, not a general Promise mechanism.

---

# 34. Return Values of `Atomics.wait`

The result can indicate outcomes such as:

```text
"not-equal"
"ok"
"timed-out"
```

Always check the return value.

Do not assume:

```text
wait returned
→ another agent successfully completed the desired operation.
```

The waking event and the protected condition are separate concepts.

---

# 35. Condition-Loop Pattern

The correct conceptual pattern is:

```js
while (!condition()) {
  Atomics.wait(state, index, expected);
}
```

Why?

Because waking does not prove that:

```text
the condition is now true.
```

Another agent may have:

```text
consumed the resource
changed the state
```

before you resume.

The condition must be rechecked.

---

# 36. Lost Wakeup Thinking

A naive design:

```text
check condition
wait
```

can be dangerous when another agent changes state between those steps.

Correct coordination combines:

```text
shared state
+
atomic state transition
+
wait/notify protocol
```

The state itself should be the source of truth.

---

# 37. `Atomics.notify`

Example:

```js
Atomics.notify(state, 0, 1);
```

This requests that waiting agents blocked on the relevant location be woken according to the specification's wait/notify rules.

A notify does not mean:

```text
"the requested business condition is definitely satisfied."
```

It means:

```text
"wake waiters so they can re-check shared state."
```

---

# 38. Notify Is Not a Message Queue

Do not treat:

```js
Atomics.notify(...)
```

as:

```text
send(data)
```

The data must already exist in shared memory.

A useful model is:

```text
write shared state
→ notify
→ waiter wakes
→ waiter checks state
```

---

# 38. `Atomics.waitAsync`

`Atomics.waitAsync` supports a non-blocking waiting style.

It can return either:

```text
synchronous "not-equal"
```

or:

```text
an object representing an asynchronous wait
```

whose eventual result can be awaited.

This is useful when an agent must avoid blocking its execution mechanism.

The current ECMAScript specification includes `Atomics.waitAsync` as part of the Atomics API. citeturn811838search2

---

# 40. `wait` vs `waitAsync`

### `Atomics.wait`

Conceptually:

```text
block the calling agent's execution
```

### `Atomics.waitAsync`

Conceptually:

```text
initiate asynchronous waiting
without blocking the agent in the same way
```

Use based on:

```text
host constraints
execution role
latency
throughput
```

---

# 41. Browser Main-Thread Constraint

Blocking the browser's main JavaScript execution agent is generally incompatible with responsive UI behavior and host restrictions around waiting.

Therefore shared-memory synchronization for browser applications should usually be designed around:

```text
Workers
+
waitAsync where appropriate
+
message passing
```

rather than trying to block the UI thread.

The exact host rules must be verified for the deployment context.

---

# 42. Node.js Worker Threads

Node.js worker threads provide an important practical environment for:

```text
multiple JavaScript agents
```

and shared memory can be used between workers.

Typical architecture:

```text
main thread
   │
   ├── Worker A
   ├── Worker B
   └── Worker C
        ↘
      SharedArrayBuffer
```

Use this when:

```text
shared-memory coordination
```

provides measurable benefit.

---

# 43. Shared Memory Is Not a Replacement for Workers

`SharedArrayBuffer` does not create concurrency by itself.

You need:

```text
multiple agents
```

capable of accessing the shared memory.

A single agent with a SharedArrayBuffer is still:

```text
one execution agent
```

---

# 44. Basic Mutex

A pedagogical mutex can use:

```text
0 = unlocked
1 = locked
```

Acquire:

```js
while (Atomics.compareExchange(lock, 0, 0, 1) !== 0) {
  Atomics.wait(lock, 0, 1);
}
```

Release:

```js
Atomics.store(lock, 0, 0);
Atomics.notify(lock, 0, 1);
```

This is a learning model, not a complete production lock.

---

# 45. Why the Mutex Uses a Loop

Suppose:

```text
waiter wakes
```

That does not imply:

```text
lock is available
```

Another agent may have acquired it first.

Therefore:

```text
wake
→ retry compareExchange
```

The condition is ownership state, not notification.

---

# 46. Production Mutex Concerns

A real lock design must consider:

```text
fairness
starvation
priority inversion
worker failure
lock abandonment
timeouts
deadlocks
critical-section length
contention
```

A simple two-state integer cannot solve all of these.

---

# 47. Semaphore

A semaphore can represent:

```text
number of available permits
```

For example:

```text
permits = 4
```

A worker atomically decrements the count when claiming a permit.

When no permits remain:

```text
wait
```

On release:

```text
increment
notify
```

This maps directly to:

```text
bounded concurrency
```

---

# 48. Bounded Queue

A shared-memory ring buffer can use:

```text
head
tail
capacity
data region
```

Conceptual structure:

```text
head → next item to consume
tail → next free slot to produce

[ item ][ item ][ free ][ free ][ item ]
```

Atomics coordinate:

```text
producer ownership
consumer ownership
head movement
tail movement
```

---

# 49. Single-Producer Single-Consumer Queue

The simplest high-performance queue assumes:

```text
one producer
one consumer
```

Then:

```text
producer owns tail
consumer owns head
```

with shared visibility rules.

This can be simpler than:

```text
multi-producer/multi-consumer
```

because fewer races exist.

Design the simplest concurrency model that satisfies the workload.

---

# 50. Multi-Producer/Multi-Consumer Complexity

With multiple producers and consumers you need to reason about:

```text
slot ownership
ABA-like hazards
sequence metadata
CAS contention
fairness
memory visibility
queue fullness
queue emptiness
```

Do not implement this casually.

Prefer a proven implementation when production correctness matters.

---

# 51. Shared Memory and Backpressure

Shared queues need boundedness.

Without limits:

```text
producer faster than consumer
→ queue grows
→ memory pressure
```

With bounded capacity:

```text
queue full
→ producer waits / rejects / sheds load
```

This connects shared memory to:

```text
Chapter 38 — Async Iteration / Streaming
Chapter 60 — Node.js Streams / Backpressure
Chapter 84 — Reliability
```

---

# 52. Shared Memory and Memory Budgets

Define:

```text
buffer capacity
queue capacity
number of workers
per-message size
control-region size
```

Shared memory does not eliminate resource constraints.

It changes:

```text
where the data lives
```

and:

```text
how agents coordinate access.
```

---

# 53. Memory Model: Events

The ECMAScript memory model describes shared-memory behavior through memory events.

Conceptually:

```text
ReadSharedMemory
WriteSharedMemory
ReadModifyWrite
```

and relations over those events.

The model defines which event graphs are allowed.

This is much closer to formal concurrency theory than ordinary Promise scheduling.

---

# 54. Memory Model: Atomic vs Data Accesses

The specification distinguishes:

```text
atomic accesses
```

from:

```text
data accesses
```

Atomic accesses are sequentially consistent.

Non-atomic shared accesses can participate in data races and may be observed in ways that would surprise someone assuming sequential consistency for all accesses. citeturn811838search0

---

# 55. Why Data Races Are Dangerous

Consider:

```js
// Agent A
shared[0] = 1;
shared[1] = 1;

// Agent B
if (shared[1] === 1) {
  console.log(shared[0]);
}
```

Without appropriate atomic synchronization, you should not casually infer the visibility/order you want from ordinary shared accesses.

The memory model exists precisely because:

```text
compiler transformations
+
hardware execution
+
concurrent agents
```

make naive sequential reasoning unsafe in the presence of races.

---

# 56. No Undefined Behavior

The current ECMAScript specification explicitly states that the memory model defines possible behavior for races rather than adopting a C/C++-style blanket “undefined behavior” rule. citeturn811838search0

This does not make data races safe.

It means:

```text
the specification still constrains what outcomes are permitted.
```

Racy code can still be extremely difficult to reason about.

---

# 57. Happens-Before Intuition

A useful practical mental model is:

```text
A happens-before B
```

means:

```text
A's relevant effects are ordered before B under the synchronization relation.
```

Atomics can establish ordering relationships that make concurrent algorithms predictable.

Do not treat “happens-before” as merely:

```text
"line A ran before line B."
```

It is a concurrency relation.

---

# 58. Agent-Local Program Order

Each agent has its own sequence of evaluation.

Conceptually:

```text
Agent A:
A1 → A2 → A3

Agent B:
B1 → B2 → B3
```

Shared-memory semantics constrain how:

```text
A*
```

and:

```text
B*
```

can be observed.

---

# 59. Sequential Consistency Example

Suppose:

```text
A:
Atomics.store(x, 1)
Atomics.store(y, 1)

B:
r1 = Atomics.load(y)
r2 = Atomics.load(x)
```

Because atomic operations are sequentially consistent, the result must be compatible with one global ordering of atomic events that respects each agent's program order. citeturn811838search0

This provides a strong reasoning model.

---

# 60. Atomic Operations Do Not Protect Ordinary Objects

This does not make:

```js
const object = {};
```

shared between agents.

There is no direct:

```text
Atomics.lock(object)
```

model.

Shared-memory algorithms should represent coordination state in supported shared typed-array storage.

---

# 61. Atomicity of 64-Bit Operations

`BigInt64Array` and `BigUint64Array` participate in relevant atomic operations.

Do not assume:

```text
all ordinary Number accesses are atomic
```

for arbitrary shared memory.

The supported atomic operation/type combinations matter.

---

# 62. `Atomics.isLockFree`

The API includes:

```js
Atomics.isLockFree(size)
```

which can help determine whether a given access size is expected to have lock-free atomic support on the implementation.

This is not a guarantee that your algorithm is:

```text
contention-free
```

or:

```text faster.
```

Lock-free support is about implementation capability for an access size, not application-level performance.

---

# 63. Growable SharedArrayBuffer

Modern ECMAScript specifications include growable `SharedArrayBuffer` support.

Conceptually:

```js
const buffer = new SharedArrayBuffer(1024, {
  maxByteLength: 4096
});
```

and the buffer can grow within its declared maximum under the defined semantics.

The current specification documents fixed-length and growable `SharedArrayBuffer` objects and `SharedArrayBuffer.prototype.grow`. citeturn811838search3

Do not assume all deployed runtimes/hosts support every modern feature equally; compatibility must be verified for the target environment.

---

# 64. Growable Shared Memory Trade-Offs

Benefits:

```text
avoid allocating a new shared buffer for growth
```

Costs:

```text
bounds coordination
view length considerations
capacity planning
host/runtime support
synchronization complexity
```

Design carefully.

---

# 65. SharedArrayBuffer Cannot Be “Detached Away”

Ordinary `ArrayBuffer` can participate in transfer/detachment behavior.

Shared memory is designed differently because other agents may still rely on the same storage.

The current specification explicitly notes that a `SharedArrayBuffer`'s internal data does not become null through ordinary detachment. citeturn811838search3

This is an important semantic difference.

---

# 66. Web Security Context

Historically, powerful shared-memory features in browsers have had strong security requirements because shared memory can enable fine-grained timing and cross-context side channels.

Browser deployments should therefore explicitly verify:

```text
cross-origin isolation
```

requirements and browser support for the specific feature.

Do not assume:

```text
"works in Node"
```

means:

```text
"works in a normal web page."
```

---

# 67. Security Threat Model

Shared memory increases the importance of:

```text
timing channels
resource exhaustion
worker abuse
denial of service
cross-context information leakage
```

A malicious algorithm can intentionally create:

```text
busy-waiting
cache contention
high-frequency Atomics operations
```

Use:

```text
timeouts
bounded work
least privilege
worker isolation
```

where appropriate.

---

# 68. Busy Waiting

Bad pattern:

```js
while (Atomics.load(state, 0) === 0) {}
```

This can consume an entire CPU core.

Possible alternatives:

```text
Atomics.wait
Atomics.waitAsync
message passing
bounded polling
```

Choose according to the environment and workload.

---

# 69. Spinlocks

A spinlock repeatedly checks:

```text
lock state
```

instead of sleeping.

Advantages:

```text
low wait latency for very short critical sections
```

Costs:

```text
CPU consumption
contention
poor behavior under long waits
```

Use only when the workload justifies it.

---

# 70. Hybrid Spin-Then-Wait

A more nuanced lock can:

```text
spin briefly
→ if still unavailable
→ wait
```

This can trade:

```text
latency
```

against:

```text
CPU
```

but adds complexity.

Benchmark under actual contention.

---

# 71. Deadlock

Shared-memory locks introduce classic deadlock risks.

Example:

```text
Agent A holds Lock 1 → waits Lock 2
Agent B holds Lock 2 → waits Lock 1
```

Neither progresses.

Avoid through:

```text
global lock ordering
short critical sections
timeouts
lock hierarchy
single-owner designs
```

---

# 72. Starvation

A worker may repeatedly lose a CAS race and fail to acquire a resource.

This is:

```text
starvation
```

A lock-free algorithm can still provide poor fairness.

Measure:

```text
wait distribution
p95/p99 acquisition time
retry counts
```

---

# 73. Priority Inversion

A high-priority operation can be blocked behind lower-priority work holding a resource.

Shared-memory JavaScript systems can inherit these classic concurrency concerns.

If priority matters:

```text
do not assume a plain mutex is sufficient.
```

---

# 74. ABA Problem

CompareExchange algorithms can encounter an ABA-style issue:

```text
A
→ B
→ A
```

A CAS observing:

```text
A
```

may incorrectly infer:

```text
"nothing important changed."
```

Mitigations may include:

```text
version counters
tagged state
sequence numbers
```

This is an advanced reason not to implement lock-free data structures casually.

---

# 75. Shared Memory and Immutability

Shared memory does not force every algorithm to be mutable.

You can use:

```text
immutable message payload regions
versioned slots
copy-on-write within buffers
```

and keep mutation limited to:

```text
atomic control words
```

This can dramatically simplify reasoning.

---

# 76. Recommended Architecture

For many systems prefer:

```text
immutable/shared data
+
small atomic coordination region
```

rather than:

```text
large shared mutable state graph
```

This reduces:

```text
race surface
debugging complexity
lock contention
security risk
```

---

# 77. Shared Memory + Message Passing Hybrid

A useful architecture:

```text
large data
    ↓
SharedArrayBuffer

small control message
    ↓
Atomics / message passing

worker
    ↓
processes segment
```

Use:

```text
shared memory for bulk data
```

and:

```text
message passing for control flow
```

when that produces a simpler design.

---

# 78. Example — Work Flag

Shared state:

```text
0 = no work
1 = work available
```

Producer:

```js
Atomics.store(state, 0, 1);
Atomics.notify(state, 0, 1);
```

Consumer:

```js
while (Atomics.load(state, 0) === 0) {
  Atomics.wait(state, 0, 0);
}

consumeWork();
```

A real design must address:

```text
reset
multiple producers
multiple consumers
work count
duplicate consumption
shutdown
```

---

# 79. Example — Work Counter

Use:

```js
Atomics.add(counter, 0, 1);
```

Producer increments work count.

Consumer attempts:

```js
const previous = Atomics.sub(counter, 0, 1);
```

Only proceed if the previous count indicates available work.

The exact protocol must avoid:

```text
negative counts
lost work
over-consumption
```

---

# 80. Example — Shutdown Flag

Shared:

```text
0 = running
1 = shutdown requested
```

Workers periodically check:

```js
if (Atomics.load(control, 0) === 1) {
  return;
}
```

A more complete shutdown protocol may combine:

```text
shutdown flag
active-worker count
queue state
notify
```

This connects to:

```text
graceful shutdown
```

and:

```text
worker lifecycle
```

---

# 81. Shared Memory and Cancellation

An `AbortSignal` does not automatically synchronize shared memory.

A shared-memory algorithm may need:

```text
cancellation flag
```

or:

```text
message + atomic state
```

Then workers must explicitly check the cancellation state.

Cancellation remains a protocol.

---

# 82. Shared Memory and Exceptions

An exception in one agent does not automatically roll back:

```text
shared memory mutations
```

If an agent performs:

```text
write A
write B
throw
```

another agent can observe state changed before the error.

Therefore shared-memory algorithms need explicit transaction-like invariants if partial updates would be invalid.

---

# 83. Shared Memory and Crash/Worker Failure

If a worker owning a lock terminates unexpectedly:

```text
lock may remain held
```

A basic mutex has no automatic:

```text
owner death recovery
```

Possible designs:

```text
timeouts
lease tokens
owner generation numbers
recovery protocol
lock-free state
```

This is a major production concern.

---

# 84. Lease-Based Ownership

Instead of:

```text
lock forever
```

store:

```text
owner ID
generation
deadline
```

Then another worker can determine whether ownership is stale.

But this introduces:

```text
clock/timing assumptions
lease renewal
ABA concerns
failure detection
```

Do not add leases without a concrete failure requirement.

---

# 85. Shared Memory and Observability

Instrument:

```text
CAS retries
lock acquisition latency
lock contention
wait counts
notify counts
queue depth
queue occupancy
worker utilization
timeouts
dropped work
```

A system can be:

```text
functionally correct
```

but operationally broken because:

```text
workers spend most time contending
```

---

# 86. Debugging Race Conditions

When debugging shared-memory code:

```text
1. Reproduce with deterministic workload.
2. Record agent count.
3. Record shared memory layout.
4. Identify every read/write.
5. Classify atomic vs non-atomic.
6. Identify synchronization edges.
7. State the invariant.
8. Find the first possible violation.
9. Add controlled instrumentation.
10. Re-test under contention.
```

Do not rely on:

```text
console.log
```

alone.

Logging can change timing and hide races.

---

# 87. Race Reproduction

Increase probability through:

```text
many iterations
controlled barriers
artificial delays
multiple workers
small critical sections
randomized scheduling points
stress loops
```

Then record:

```text
state
operation
agent
sequence
```

Careful:

```text
timing perturbation
```

can also make a race disappear.

---

# 88. Deterministic Barrier

A shared barrier can coordinate:

```text
all workers reach point N
```

then allow them to proceed together.

Conceptually:

```text
arrived count
+
generation
+
wait/notify
```

Barriers are useful for:

```text
race testing
benchmarks
parallel algorithms
```

but easy to deadlock if:

```text
one participant exits early.
```

---

# 89. Shared Memory Testing Strategy

Test:

```text
1 worker
2 workers
many workers
no contention
high contention
worker termination
timeouts
cancellation
queue full
queue empty
duplicate wakeups
```

Also test:

```text
stress for millions of operations
```

where feasible.

---

# 90. Property-Based Concurrency Testing

Define invariants such as:

```text
produced === consumed + remaining
count >= 0
no item consumed twice
every completed operation has one owner
queue occupancy <= capacity
```

Then generate:

```text
random producer/consumer sequences
```

and test those invariants.

This is usually more valuable than checking only expected final output for one schedule.

---

# 91. Model-Based Testing

Represent shared state as:

```text
abstract model
```

and compare implementation behavior against the model.

Example:

```text
model queue
vs
shared-memory queue
```

This can reveal:

```text
rare interleavings
```

and:

```text
state-transition bugs.
```

---

# 92. Performance Considerations

Shared memory can reduce:

```text
copying
serialization
message overhead
```

but introduces:

```text
CAS contention
cache-line traffic
synchronization
worker coordination
```

and may increase:

```text
CPU
complexity
```

Measure:

```text
throughput
latency
CPU
contention
memory
```

---

# 93. False Sharing

Different frequently modified atomic variables that happen to occupy nearby cache lines can cause expensive coherence traffic at the hardware level.

For example:

```text
worker A updates counter A
worker B updates counter B
```

but both counters share a cache line.

This can produce:

```text
cache invalidation traffic
```

even though logical state is independent.

This is an implementation/performance concept, not an ECMAScript language guarantee.

---

# 94. Data Layout

For high-throughput shared memory:

```text
control words
```

and:

```text
bulk payload
```

should be laid out intentionally.

Consider:

```text
alignment
padding
capacity
slot size
metadata locality
```

but verify with profiling rather than assuming hardware effects.

---

# 95. Memory Bandwidth

Shared memory can shift bottlenecks from:

```text
serialization
```

to:

```text
memory bandwidth
cache coherence
```

A design can become:

```text
CPU-efficient
```

but:

```text
memory-bound.
```

Measure hardware/runtime counters where available.

---

# 96. `Atomics.pause`

Current living ECMAScript specifications include an `Atomics.pause()` operation in the Atomics API. Its purpose is related to giving an implementation a hint during spin-waiting rather than changing the synchronization state itself. citeturn811838search7

Treat this as:

```text
implementation/performance aid
```

not:

```text
correctness primitive.
```

Do not build correctness around the assumption that `pause()` changes memory visibility.

---

# 97. Browser Deployment Considerations

Before deploying shared-memory browser code verify:

```text
browser support
cross-origin isolation requirements
worker support
deployment headers
content security policy interactions
embedding constraints
third-party resource behavior
```

A correct ECMAScript algorithm can still fail operationally because the host environment does not expose the required capability.

---

# 98. Node.js Deployment Considerations

For Node.js shared-memory systems consider:

```text
worker lifecycle
worker count
CPU topology
process supervision
graceful shutdown
error propagation
resource ownership
observability
```

Do not assume:

```text
more workers = more throughput.
```

You can hit:

```text
CPU saturation
memory bandwidth
contention
GC
I/O bottlenecks
```

---

# 99. Security Considerations

Shared-memory designs should consider:

```text
timing side channels
resource exhaustion
cross-context isolation
untrusted worker code
data lifetime
sensitive data reuse
```

Never put:

```text
secrets
credentials
tokens
```

into broadly accessible shared-memory regions unless the isolation model explicitly protects them.

Remember:

```text
shared
```

means:

```text
shared access
```

not:

```text
safe access.
```

---

# 100. Implementation From Scratch — Shared Counter

Build a worker-based counter:

```text
main thread
→ create SharedArrayBuffer
→ launch workers
→ workers Atomics.add(counter, 0, 1)
→ wait for completion
→ read final counter
```

Requirements:

```text
configurable worker count
configurable iterations
verification of final count
timing
```

Compare:

```text
single-thread counter
message-passing counter
shared-memory counter
```

under representative workloads.

---

# 101. Implementation — Mutex

Implement:

```text
acquire()
release()
```

using:

```text
compareExchange
wait
notify
```

Then test:

```text
two workers
many workers
high contention
worker cancellation
worker failure
```

Document the known limitations of your learning implementation.

---

# 102. Implementation — Semaphore

Implement:

```text
new Semaphore(permits)
acquire()
release()
```

Use shared memory to represent permit count.

Test:

```text
permits = 1
permits > 1
more workers than permits
release without acquire
worker failure
```

---

# 103. Implementation — Bounded Ring Buffer

Build:

```text
SharedArrayBuffer
+
Int32Array control region
+
payload region
```

Support:

```text
enqueue
dequeue
full
empty
wait
notify
shutdown
```

Start with:

```text
single producer
single consumer
```

Then consider:

```text
multiple producers/consumers
```

only after the first model is correct.

---

# 104. Implementation Progression

### Guided

```text
shared counter
```

### Partially Guided

```text
mutex
```

### No Reference

```text
semaphore
```

### Edge-Case Hardened

```text
ring buffer
```

### Production-Grade Learning Version

Add:

```text
timeouts
shutdown
metrics
stress tests
fault injection
```

---

# 105. Execution Walkthrough — CAS Lock

Initial:

```text
lock = 0
```

Worker A:

```text
CAS(0 → 1)
```

returns:

```text
0
```

Therefore A owns lock.

Worker B:

```text
CAS(0 → 1)
```

returns:

```text
1
```

Therefore B failed.

B waits.

A:

```text
store(0)
notify()
```

B wakes.

B retries.

The key invariant:

```text
only the agent that successfully changed 0 → 1 owns the lock.
```

---

# 106. Execution Walkthrough — Semaphore

Initial:

```text
permits = 2
```

Workers:

```text
A acquire → 1
B acquire → 0
C acquire → waits
```

A release:

```text
permits → 1
notify
```

C wakes and retries.

The correct model is:

```text
state change
→ notification
→ condition re-check
```

not:

```text
notification
→ guaranteed resource ownership
```

---

# 107. Execution Walkthrough — Bounded Queue

Initial:

```text
capacity = 4
head = 0
tail = 0
```

Producer:

```text
tail = 1
write slot 0
```

Consumer:

```text
head = 1
read slot 0
```

The actual design must use an unambiguous ownership protocol.

Do not update:

```text
control index
```

before the corresponding data is safely published according to the synchronization protocol.

---

# 108. Code Review Exercise

Review:

```js
while (Atomics.load(state, 0) === 0) {}

processWork();
```

Identify:

```text
busy wait
CPU consumption
lack of cancellation
lack of timeout
```

Then compare with:

```js
while (Atomics.load(state, 0) === 0) {
  Atomics.wait(state, 0, 0, 1000);
}
```

Explain why the second version is not automatically correct either.

---

# 109. Code Review Exercise — Broken Mutex

```js
if (Atomics.load(lock, 0) === 0) {
  Atomics.store(lock, 0, 1);
  criticalSection();
  Atomics.store(lock, 0, 0);
}
```

### Task

Find the race.

Two workers can execute:

```text
load 0
load 0
store 1
store 1
```

Both believe they acquired the lock.

Replace with:

```text
compareExchange
```

and explain why.

---

# 110. Code Review Exercise — Unsafe Queue

```js
const slot = tail[0];
tail[0] = slot + 1;
buffer[slot] = item;
```

### Task

Identify the publication-order risk.

The producer must establish the correct relationship between:

```text
claiming slot
writing payload
publishing availability
```

A correct queue needs a synchronization protocol that prevents a consumer from interpreting:

```text
slot reserved
```

as:

```text
slot fully published.
```

---

# 111. Debugging Exercise — Lost Increment

Two workers execute:

```js
shared[0] = shared[0] + 1;
```

Final value is:

```text
1007
```

instead of:

```text
2000
```

### Task

Diagnose:

```text
non-atomic read-modify-write
```

Replace with:

```js
Atomics.add(shared, 0, 1);
```

Then explain why the fix works.

---

# 112. Debugging Exercise — Deadlock

Worker A:

```text
lock 1
wait lock 2
```

Worker B:

```text
lock 2
wait lock 1
```

### Task

Diagnose.

Design:

```text
global ordering
```

such that both workers always acquire:

```text
lock 1 → lock 2
```

instead of:

```text
arbitrary order
```

---

# 113. Debugging Exercise — Starvation

A worker repeatedly reports:

```text
CAS failed 1,000,000 times
```

while another worker succeeds.

### Task

Investigate:

```text
contention
fairness
retry strategy
```

Possible mitigations:

```text
backoff
wait/notify
fair queue
partitioning
lower contention
```

Do not automatically add:

```text
more spin
```

---

# 114. Debugging Exercise — Stale Assumption

A developer writes:

```js
if (Atomics.load(flag, 0) === 1) {
  useSharedData();
}
```

Another worker does:

```js
Atomics.store(flag, 0, 1);
```

but `useSharedData()` reads data that was written through unsynchronized non-atomic accesses.

### Task

Identify the missing memory-order reasoning.

The flag itself being atomic does not automatically make every surrounding ordinary shared access safe under every algorithm.

Design a protocol where:

```text
data publication
+
flag publication
+
consumer observation
```

are correctly ordered.

---

# 115. Concurrency Testing Exercise

Design a stress test that verifies:

```text
N workers
M operations
```

against invariant:

```text
finalCount === N × M
```

Run both:

```text
ordinary shared read/write
```

and:

```text
Atomics.add
```

Record:

```text
failures
runtime
worker count
iterations
```

Explain why:

```text
absence of observed failure
```

does not prove:

```text
racy algorithm is correct.
```

---

# 116. Interview Questions

### Fundamentals

```text
1. Why does SharedArrayBuffer exist?
2. What is the difference between ArrayBuffer and SharedArrayBuffer?
3. What is an Agent?
4. What is an Agent Cluster?
5. What is a shared data block?
```

### Atomic Operations

```text
6. What does Atomics.add do?
7. What does compareExchange do?
8. Why is atomicity different from mutual exclusion?
9. What is sequential consistency?
10. What is a data race?
11. Why can non-atomic shared access behave surprisingly?
```

### Waiting

```text
12. What does Atomics.wait do?
13. What does Atomics.notify do?
14. Why must a wait normally be inside a condition loop?
15. What is Atomics.waitAsync?
16. When would you avoid blocking waits?
```

### Concurrency Design

```text
17. How would you build a mutex?
18. How would you build a semaphore?
19. How would you build a bounded ring buffer?
20. What is the difference between message passing and shared memory?
```

### Principal

```text
21. When should a production system avoid SharedArrayBuffer?
22. How would you diagnose contention?
23. How would you design worker failure recovery for a lock?
24. How would you test a concurrent data structure?
25. How would you reason about the ECMAScript memory model without mapping it directly to a specific CPU?
```

---

# 117. Predict-the-Behavior Exercises

Predict before running.

### Exercise 1

Two workers perform:

```js
shared[0] = shared[0] + 1;
```

What outcomes should you expect under contention?

---

### Exercise 2

Two workers perform:

```js
Atomics.add(shared, 0, 1);
```

What invariant should now hold?

---

### Exercise 3

Worker A:

```js
Atomics.store(flag, 0, 1);
Atomics.notify(flag, 0);
```

Worker B:

```js
while (Atomics.load(flag, 0) === 0) {
  Atomics.wait(flag, 0, 0);
}
```

Explain:

```text
condition
wait
notify
wake
re-check
```

---

### Exercise 4

A worker calls:

```js
Atomics.wait(flag, 0, 1);
```

when:

```text
flag[0] === 0
```

Classify the result.

---

### Exercise 5

A worker is waiting.

Another worker calls:

```js
Atomics.notify(flag, 0, 1);
```

Can you conclude:

```text
the desired business condition is now true?
```

Explain.

---

# 118. Mastery Exercises

### Exercise 1 — Atomic Counter

Implement:

```text
N workers
M increments
```

and prove:

```text
final = N × M
```

### Exercise 2 — Mutex

Implement:

```text
critical section
```

and verify:

```text
max concurrent owners = 1
```

### Exercise 3 — Semaphore

Verify:

```text
max concurrent permits <= limit
```

### Exercise 4 — SPSC Queue

Implement:

```text
single-producer/single-consumer ring buffer
```

with:

```text
bounded capacity
wait/notify
shutdown
```

### Exercise 5 — Fault Injection

Terminate a worker while it owns a logical resource.

Document:

```text
what breaks
how recovery happens
```

### Exercise 6 — Performance

Compare:

```text
message passing
vs
shared memory
```

for:

```text
small messages
large payloads
high frequency
low frequency
```

---

# 119. Production Design Checklist

Before using shared memory ask:

```text
1. Do we actually need shared memory?
2. Why is message passing insufficient?
3. How many agents participate?
4. What data is shared?
5. What data remains agent-local?
6. Which accesses are atomic?
7. What invariants exist?
8. What synchronization establishes them?
9. How is shutdown handled?
10. How does a crashed worker recover?
11. What is the memory bound?
12. What happens under contention?
13. How is the system observed?
14. How is the design tested?
15. What security boundary exists?
```

---

# 120. When Not to Use Shared Memory

Prefer message passing when:

```text
data size is moderate
copying cost is acceptable
correctness complexity matters more
team lacks concurrency expertise
work is naturally task/message oriented
```

Prefer shared memory when:

```text
large shared data
high-frequency communication
copying is measurable bottleneck
fine-grained coordination is truly required
the team can operate/test concurrent code
```

The decision must be evidence-based.

---

# 121. Comparison With Alternatives

| Approach | Strength | Risk |
|---|---|---|
| Message passing | simpler ownership | copy/serialization overhead |
| SharedArrayBuffer | low-copy shared bytes | race/synchronization complexity |
| Worker threads | CPU concurrency | coordination/memory |
| Child processes | stronger isolation | IPC/process overhead |
| Database coordination | durable/shared state | latency/operational cost |
| External queue | durable decoupling | infrastructure complexity |
| Atomics | precise shared-memory synchronization | difficult correctness model |

---

# 122. Performance Considerations

Measure:

```text
throughput
latency
CPU
contention
CAS retry rate
wait time
memory bandwidth
worker utilization
```

Compare:

```text
one worker
few workers
many workers
```

Do not assume linear scaling.

Typical saturation points can include:

```text
CPU
memory bandwidth
cache coherence
atomic contention
worker overhead
```

---

# 123. Memory Considerations

Shared memory shifts the memory lifecycle.

Important questions:

```text
Who holds the SharedArrayBuffer reference?

Which agents have views?

How long does the shared block remain reachable?

What is the maximum size?

How much memory is control vs payload?

Can buffers accumulate?
```

A shared buffer can remain alive as long as references exist in participating agents according to the relevant lifetime semantics.

---

# 124. Security Considerations

Security review should include:

```text
cross-context access
worker trust
shared secret material
timing channels
resource exhaustion
lock-based denial of service
untrusted payloads
worker lifecycle
```

If a shared buffer contains sensitive information:

```text
minimize lifetime
minimize access
clear/reuse deliberately where required
```

Do not assume:

```text
GC
```

is a secure memory wipe mechanism.

---

# 125. Browser vs Node.js

## Browser

Typical tools:

```text
Web Workers
SharedArrayBuffer
Atomics
cross-origin isolation requirements
```

## Node.js

Typical tools:

```text
worker_threads
SharedArrayBuffer
Atomics
process-level isolation alternatives
```

The ECMAScript memory model is shared conceptually.

The host's ability to create/access agents differs.

---

# 126. Specification / Runtime Source Discipline

For shared-memory questions prefer:

```text
1. ECMAScript SharedArrayBuffer specification
2. ECMAScript Atomics semantics
3. ECMAScript memory model
4. host worker/agent model
5. engine implementation
6. CPU/cache architecture
```

The current ECMAScript specification explicitly describes its memory model through allowed memory-event graphs and defines atomic accesses as sequentially consistent. citeturn811838search0

Do not claim:

```text
"Atomics.add maps directly to CPU instruction X"
```

unless you are making an explicitly engine/architecture-specific statement backed by evidence.

---

# 127. Common Misconceptions

### Misconception 1

> “JavaScript is single-threaded, so races cannot happen.”

Correction:

```text
individual agents execute sequentially, but multiple agents can access shared memory.
```

### Misconception 2

> “SharedArrayBuffer shares JavaScript objects.”

Correction:

```text
shared memory is shared bytes, not shared object graphs.
```

### Misconception 3

> “Atomics makes the whole algorithm atomic.”

Correction:

```text
only the specified atomic operation receives the relevant atomic semantics.
```

### Misconception 4

> “notify guarantees the waiter can proceed.”

Correction:

```text
the waiter must re-check the condition.
```

### Misconception 5

> “lock-free means faster.”

Correction:

```text
contention can make lock-free retry loops expensive.
```

### Misconception 6

> “wait means sleep forever.”

Correction:

```text
wait has conditions and timeout behavior.
```

### Misconception 7

> “waitAsync is just Promise.resolve.”

Correction:

```text
it is a shared-memory synchronization mechanism integrated with asynchronous waiting.
```

### Misconception 8

> “A correct Node worker algorithm automatically works in browsers.”

Correction:

```text
host capabilities and security requirements differ.
```

---

# 128. Common Failure Modes

```text
Failure 1:
Unsynchronized shared reads/writes.

Failure 2:
Read-modify-write implemented as separate ordinary operations.

Failure 3:
Busy waiting without a reason.

Failure 4:
Waiting without re-checking the condition.

Failure 5:
Using notify as if it were a message.

Failure 6:
No bounded queue capacity.

Failure 7:
Lock without crash recovery.

Failure 8:
Ignoring starvation.

Failure 9:
Ignoring contention measurement.

Failure 10:
Assuming specification semantics equal CPU instructions.

Failure 11:
Sharing too much mutable state.

Failure 12:
Using shared memory when message passing would be simpler.
```

---

# 129. Principal Decision Framework

Evaluate shared-memory architecture using:

```text
Correctness
Concurrency safety
Performance
Memory
Security
Reliability
Maintainability
Observability
Scalability
Developer Experience
Operational Complexity
Failure Recovery
Future Change
```

The default architectural preference should be:

```text
local state
→ message passing
→ shared memory
```

only when a measured requirement justifies moving rightward in the complexity spectrum.

---

# 130. Retrieval Record

```md
# Chapter 127 — Revision / Retrieval Record

## Attempt
- Date:
- Duration:
- Status before:
- Status after:

## SharedArrayBuffer
-

## Agents / Agent Clusters
-

## Atomics
-

## Memory Model
-

## Data Races
-

## Sequential Consistency
-

## Wait / Notify
-

## waitAsync
-

## Mutex
-

## Semaphore
-

## Queue
-

## Worker Failure
-

## Browser/Node Differences
-

## Strongest Areas
-

## Weakest Areas
-

## Questions Requiring Rework
-

## Next Review
-
```

---

# 131. Spaced Retrieval Schedule

### Day 0

Study:

```text
SharedArrayBuffer
Atomics
memory model
```

and complete prediction exercises.

### Day 1

Trace:

```text
atomic counter
mutex
```

by hand.

### Day 3

Explain:

```text
data race
sequential consistency
```

without notes.

### Day 7

Implement:

```text
mutex
semaphore
```

from requirements only.

### Day 14

Implement:

```text
SPSC ring buffer
```

with stress testing.

### Day 21

Inject:

```text
worker failure
lock contention
queue full
```

and analyze recovery.

### Day 30

Explain the ECMAScript memory model without mapping it directly to one CPU architecture.

---

# 132. Dependency Graph

```text
Chapter 27
Typed Arrays / Binary Data
        ↓
Chapter 34
Node Event Loop
        ↓
Chapter 44
Realms / Agents / Execution Isolation
        ↓
Chapter 45
Memory / GC
        ↓
Chapter 52
Web Workers / Concurrency
        ↓
Chapter 61
Worker Threads / Child Processes / Cluster
        ↓
Chapter 72
Complexity
        ↓
Chapter 84
Reliability
        ↓
Chapter 85
Performance
        ↓
Chapter 123
Grammar
        ↓
Chapter 124
Execution / Completion / References
        ↓
Chapter 125
Promise Internals
        ↓
Chapter 126
Module Linking
        ↓
Chapter 127
SharedArrayBuffer / Atomics / Memory Model
        ↓
Chapter 128
Internationalization / Intl
```

---

# 133. Concept Connections

## Depends On

```text
Typed Arrays
Workers
Agents
Memory
Async execution
Node.js concurrency
Browser concurrency
Atomics
```

## Builds Toward

```text
advanced runtime engineering
parallel algorithms
high-performance worker pools
binary processing
native interoperability
platform engineering
```

## Related Concepts

```text
mutex
semaphore
CAS
lock-free structures
message passing
channels
queues
condition variables
memory ordering
cache coherence
```

## Concepts Revisited

```text
SharedArrayBuffer
TypedArray
workers
queues
backpressure
cancellation
graceful shutdown
performance
memory
```

## Why This Chapter Matters

This chapter removes one of the most persistent oversimplifications in JavaScript:

```text
"JavaScript is single-threaded."
```

A more accurate model is:

```text
an agent executes JavaScript sequentially
+
multiple agents can execute concurrently
+
shared memory can create true shared-state concurrency
```

That requires a formal memory model.

---

# 134. Track A — Core Theory

Master:

```text
SharedArrayBuffer
shared data blocks
Agents
Agent Clusters
TypedArray views
atomic access
non-atomic access
data races
sequential consistency
memory events
Atomics
compareExchange
wait
waitAsync
notify
```

Deliverable:

```text
explain why a concurrent algorithm is or is not correctly synchronized.
```

---

# 135. Track B — Implementation

Build:

```text
atomic counter
mutex
semaphore
SPSC ring buffer
barrier
stress harness
```

Progression:

```text
Guided
→ Partially Guided
→ No Reference
→ Edge-Case Hardened
→ Production-Grade Learning Version
```

Deliverable:

```text
implement concurrency primitives while preserving explicit invariants.
```

---

# 136. Track C — Interview / Reasoning

Practice:

```text
"Why can JavaScript have data races?"

"What's the difference between Promise concurrency and shared-memory concurrency?"

"Why is compareExchange useful?"

"Why must wait be inside a loop?"

"Why doesn't notify mean success?"

"When would you choose message passing over shared memory?"

"How would you recover from a worker dying while holding a lock?"
```

Deliverable:

```text
reason from memory-model invariants instead of threading folklore.
```

---

# 137. Mastery Gate

You may mark:

```text
[+] Completed
```

when:

```text
[ ] SharedArrayBuffer understood
[ ] Agents understood
[ ] Atomics understood
[ ] sequential consistency understood
[ ] data races understood
[ ] wait/notify understood
[ ] waitAsync understood
[ ] mutex implemented
[ ] semaphore implemented
[ ] bounded queue implemented
[ ] stress testing performed
```

Mark:

```text
[*] Mastered
```

only when you can:

```text
[ ] explain the ECMAScript memory model
[ ] distinguish atomic and non-atomic shared access
[ ] reason about data races
[ ] design synchronization protocols
[ ] implement a correct atomic counter
[ ] implement a bounded synchronization primitive
[ ] diagnose deadlock/starvation
[ ] reason about worker failure
[ ] measure contention
[ ] decide when shared memory is justified
```

---

# 138. Completion Snapshot

```md
# Chapter 127 — Completion Snapshot

Status:
[ ] Not Started
[~] In Progress
[?] Needs Revision
[+] Completed
[*] Mastered

Primary Gaps:
-

SharedArrayBuffer:
[ ] weak
[ ] developing
[ ] strong
[ ] principal

Agents:
[ ] weak
[ ] developing
[ ] strong
[ ] principal

Atomics:
[ ] weak
[ ] developing
[ ] strong
[ ] principal

Memory Model:
[ ] weak
[ ] developing
[ ] strong
[ ] principal

Synchronization:
[ ] weak
[ ] developing
[ ] strong
[ ] principal

Race Debugging:
[ ] weak
[ ] developing
[ ] strong
[ ] principal

Performance / Contention:
[ ] weak
[ ] developing
[ ] strong
[ ] principal

Concurrency Implementation:
[ ] not started
[ ] partial
[ ] complete
[ ] hardened
```

---

# 139. Completion Criteria

```text
[ ] SharedArrayBuffer explained
[ ] shared data blocks explained
[ ] ArrayBuffer difference explained
[ ] Agent model explained
[ ] Agent Cluster model explained
[ ] TypedArray shared views explained
[ ] atomic vs non-atomic access explained
[ ] Atomics API explained
[ ] atomic read-modify-write understood
[ ] compareExchange understood
[ ] sequential consistency understood
[ ] data races understood
[ ] memory-event model understood conceptually
[ ] wait understood
[ ] notify understood
[ ] waitAsync understood
[ ] condition-loop pattern understood
[ ] mutex implemented
[ ] semaphore implemented
[ ] bounded queue implemented
[ ] deadlock analyzed
[ ] starvation analyzed
[ ] worker failure considered
[ ] contention measured
[ ] browser constraints considered
[ ] Node.js constraints considered
[ ] security threats considered
[ ] message passing vs shared memory compared
```

---

# 140. Final Principal Mental Model

Use:

```text
Agent
   ↓
local JavaScript state

multiple agents
   ↓
possible concurrent execution

SharedArrayBuffer
   ↓
shared bytes

TypedArray view
   ↓
shared memory locations

Atomics
   ↓
atomic synchronization

memory model
   ↓
allowed observations/orderings

algorithm
   ↓
invariant preservation
```

For every shared-memory algorithm ask:

```text
What is shared?

Who can read it?

Who can write it?

Which accesses are atomic?

What invariant must hold?

What synchronization establishes the invariant?

What happens under contention?

What happens if a worker dies?

What happens if the queue is full?

How does cancellation work?

How is the algorithm observed?
```

---

# 141. Final Principal Principle

> **Shared memory is a capability, not an optimization you add for free.**

The mature architecture is:

```text
local state
+
message passing
```

by default.

Move to:

```text
shared memory
+
Atomics
```

only when the measured workload justifies the additional concurrency complexity.

When using shared memory, correctness comes from:

```text
explicit state
+
atomic operations
+
synchronization protocol
+
invariants
+
bounded resources
+
failure handling
```

not from:

```text
"JavaScript is single-threaded."
```

The principal-level question is therefore not:

```text
"Can JavaScript share memory?"
```

It is:

```text
"Can we introduce shared-memory concurrency here
without making correctness, security, operations,
and future change harder than the performance benefit justifies?"
```

That is the engineering judgment this chapter is designed to develop.