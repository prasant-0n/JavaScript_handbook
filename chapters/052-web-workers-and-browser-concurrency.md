# Chapter 52 — Web Workers and Browser Concurrency

> **Curriculum Position:** Part IX — Browser  
> **Prerequisites:** Chapters 41–51  
> **Primary Focus:** Web Workers, execution contexts, agents, messaging, structured cloning, transferables, SharedArrayBuffer, Atomics, worker lifecycle, concurrency, coordination, performance, memory, security, and production architecture  
> **Status:** `[ ] Not Started`  
> **Depth Target:** Web Platform semantics → Agent model → worker execution → communication → shared memory → synchronization → production concurrency  
> **Important Scope Rule:** Workers are a **Web Platform execution model**, not an ECMAScript feature by themselves. `SharedArrayBuffer` and `Atomics` are standardized language/runtime capabilities, while browser Worker APIs and lifecycle behavior come from Web Platform standards.

---

## Chapter Navigation

- [1. Learning Objectives](#1-learning-objectives)
- [2. Prerequisites](#2-prerequisites)
- [3. What Is a Web Worker?](#3-what-is-a-web-worker)
- [4. Why Do Workers Exist?](#4-why-do-workers-exist)
- [5. Mental Model](#5-mental-model)
- [6. Core Rules](#6-core-rules)
- [7. Execution Contexts and Agents](#7-execution-contexts-and-agents)
- [8. Dedicated Workers](#8-dedicated-workers)
- [9. Shared Workers](#9-shared-workers)
- [10. Service Workers](#10-service-workers)
- [11. Worker Globals](#11-worker-globals)
- [12. Worker Creation and Lifecycle](#12-worker-creation-and-lifecycle)
- [13. `postMessage()` and Message Events](#13-postmessage-and-message-events)
- [14. Structured Clone](#14-structured-clone)
- [15. Transferable Objects](#15-transferable-objects)
- [16. SharedArrayBuffer](#16-sharedarraybuffer)
- [17. Atomics](#17-atomics)
- [18. Data Races and Memory Visibility](#18-data-races-and-memory-visibility)
- [19. Producer / Consumer Concurrency](#19-producer--consumer-concurrency)
- [20. Worker Pools](#20-worker-pools)
- [21. CPU-Bound Work](#21-cpu-bound-work)
- [22. Main Thread Responsiveness](#22-main-thread-responsiveness)
- [23. Messaging Cost and Backpressure](#23-messaging-cost-and-backpressure)
- [24. Error Handling](#24-error-handling)
- [25. Worker Termination](#25-worker-termination)
- [26. Worker and Browser API Availability](#26-worker-and-browser-api-availability)
- [27. Workers and DOM Access](#27-workers-and-dom-access)
- [28. Workers and Timers](#28-workers-and-timers)
- [29. Workers and Fetch / Networking](#29-workers-and-fetch--networking)
- [30. Workers and Storage](#30-workers-and-storage)
- [31. Shared Workers and Cross-Context Coordination](#31-shared-workers-and-cross-context-coordination)
- [32. Service Worker Concurrency Model](#32-service-worker-concurrency-model)
- [33. Synchronization Patterns](#33-synchronization-patterns)
- [34. Common Concurrency Patterns](#34-common-concurrency-patterns)
- [35. Anti-Patterns](#35-anti-patterns)
- [36. Edge Cases](#36-edge-cases)
- [37. Common Misconceptions](#37-common-misconceptions)
- [38. Common Mistakes](#38-common-mistakes)
- [39. Comparison With Related Concepts](#39-comparison-with-related-concepts)
- [40. Performance Considerations](#40-performance-considerations)
- [41. Memory Considerations](#41-memory-considerations)
- [42. Security Considerations](#42-security-considerations)
- [43. Production Usage](#43-production-usage)
- [44. Implementation From Scratch](#44-implementation-from-scratch)
- [45. Debugging Exercises](#45-debugging-exercises)
- [46. Code Review Exercise](#46-code-review-exercise)
- [47. Interview Questions](#47-interview-questions)
- [48. Predict-the-Output Exercises](#48-predict-the-output-exercises)
- [49. Mastery Exercises](#49-mastery-exercises)
- [50. Key Takeaways](#50-key-takeaways)
- [51. Concept Connections](#51-concept-connections)
- [52. Completion Criteria](#52-completion-criteria)
- [53. Revision / Retrieval Record](#53-revision--retrieval-record)
- [54. Canonical References and Source Discipline](#54-canonical-references-and-source-discipline)
- [55. Completion Snapshot](#55-completion-snapshot)

---

# 1. Learning Objectives

By the end of this chapter, you should be able to:

1. Define a Web Worker precisely.
2. Explain why Workers exist and what problem they solve.
3. Distinguish a Worker from:
   - a browser tab,
   - a JavaScript function,
   - a Promise,
   - a thread,
   - a process,
   - a service worker.
4. Explain the relationship among:
   - Realm,
   - Agent,
   - event loop,
   - Worker global,
   - browser process.
5. Explain Dedicated Workers.
6. Explain Shared Workers.
7. Explain Service Workers at a concurrency/lifecycle level.
8. Explain Worker creation and lifecycle.
9. Explain `postMessage()`.
10. Explain `message` events.
11. Explain structured cloning.
12. Explain transferable objects.
13. Explain why transferring is different from cloning.
14. Explain `ArrayBuffer` detachment after transfer.
15. Explain `SharedArrayBuffer` and shared memory.
16. Explain why `SharedArrayBuffer` requires a different concurrency model.
17. Explain atomic operations.
18. Explain `Atomics.load`, `store`, `add`, `compareExchange`, `wait`, `notify`, and related concepts at a practical level.
19. Explain race conditions.
20. Explain lost-update problems.
21. Explain mutual exclusion conceptually.
22. Explain producer/consumer queues.
23. Explain worker pools.
24. Explain when Workers improve responsiveness.
25. Explain when Workers can make an application slower.
26. Explain communication and serialization costs.
27. Explain backpressure in Worker systems.
28. Explain Worker lifecycle, termination, and failure.
29. Explain which browser APIs are or are not available inside Workers.
30. Explain why Workers cannot directly access the page's DOM.
31. Explain how Workers can use browser APIs such as `fetch`.
32. Explain how Workers interact with timers and event loops.
33. Explain cross-context coordination using Shared Workers, BroadcastChannel, Web Locks, and Worker messaging.
34. Explain security/isolation requirements around shared memory.
35. Design safe concurrency without unnecessary shared mutable state.
36. Diagnose deadlocks, races, message storms, worker leaks, and main-thread blocking.
37. Implement a toy Worker/message/concurrency system.
38. Design production Worker architecture using correctness, performance, memory, reliability, observability, security, and maintainability.

### Mastery target

Progress through:

**Understand → Explain → Predict → Implement → Debug → Apply → Compare → Defend**

Reading this chapter alone does not establish mastery.

---

# 2. Prerequisites

## Chapter 44 — Realms, Agents, and Execution Isolation

You must understand:

```text
Realm
Agent
Agent Cluster
execution context
```

Workers make those concepts concrete.

---

## Chapter 45 — Memory and Garbage Collection

Needed for:

- Worker memory,
- cloned objects,
- transferred buffers,
- shared memory,
- worker shutdown,
- retained messaging state.

---

## Chapter 47 — JavaScript Engine Architecture

Needed to distinguish:

```text
JavaScript execution
vs
host scheduling
vs
worker isolation
```

---

## Chapter 49 — DOM Architecture

Needed to understand why a Worker does not directly manipulate:

```js
document
```

---

## Chapter 50 — Browser Events

Needed for:

```js
worker.addEventListener("message", ...)
```

and Worker event handling.

---

## Chapter 51 — Browser Web APIs

Needed for:

```text
Worker
BroadcastChannel
Web Locks
storage
fetch
timers
```

---

## Chapter 31 — Async Fundamentals

Needed for asynchronous messaging.

---

## Chapter 32 — Jobs / Promise Reactions

Needed for reasoning about:

```js
Promise.then(...)
queueMicrotask(...)
```

inside a Worker or Window.

---

## Chapter 33 — Browser Event Loop

Needed for understanding that each relevant execution context has its own event-loop behavior rather than one universal callback queue.

---

# 3. What Is a Web Worker?

A **Web Worker** provides a JavaScript execution context separate from the main Window execution context.

A basic Worker:

```js
const worker = new Worker("/worker.js");

worker.postMessage({
  type: "start"
});
```

Worker code:

```js
self.addEventListener("message", event => {
  console.log(event.data);
});
```

The purpose is to allow work to execute without blocking the main page's JavaScript execution context.

---

## 3.1 Worker Is Not Just “Another Function”

This:

```js
worker.postMessage(data);
```

crosses an execution-context boundary.

It is not equivalent to:

```js
someFunction(data);
```

---

## 3.2 Worker Is Not Automatically an OS Thread

The Web Platform exposes a Worker execution model.

The browser is free to map that execution model onto its internal process/thread architecture.

A common implementation can involve a separate thread, but:

> **The Worker API is the contract. The exact OS-thread mapping is implementation-specific.**

---

# 4. Why Do Workers Exist?

The main Window execution context is responsible for user interaction, JavaScript, and coordination with rendering.

If application code performs:

```js
for (let i = 0; i < 10_000_000_000; i++) {
  expensiveWork(i);
}
```

on the main execution context, the page can become unresponsive.

A Worker can move CPU-heavy computation:

```text
Main context
     ↓
send input
     ↓
Worker
     ↓
compute
     ↓
send result
     ↓
Main context
     ↓
update DOM
```

---

## 4.1 Primary Benefit

Workers mainly provide:

```text
concurrent execution contexts
```

that help keep the main context responsive.

---

## 4.2 Primary Cost

Workers introduce:

```text
startup
memory
communication
serialization
coordination
lifecycle
```

costs.

Therefore:

> **Moving work to a Worker is not automatically a performance improvement.**

---

# 5. Mental Model

Use:

```text
                Browser
                   |
       ┌───────────┴───────────┐
       ↓                       ↓
   Main context            Worker context
       |                       |
     Window                  WorkerGlobalScope
       |                       |
     DOM                     no page DOM
       |                       |
     Events                   Events
       |                       |
       └──── message boundary ┘
```

For multiple Workers:

```text
Main
 ├── Worker A
 ├── Worker B
 └── Worker C
```

Each can execute independently.

---

## 5.1 Shared Memory Model

With `SharedArrayBuffer`:

```text
Worker A ─┐
          ├── Shared memory
Worker B ─┤
          │
Main ─────┘
```

Now the problem changes from:

```text
message passing
```

to:

```text
concurrent shared-state programming
```

That requires synchronization.

---

# 6. Core Rules

## Rule 1 — Workers provide separate execution contexts

They are not simply asynchronous functions.

## Rule 2 — A Worker cannot directly manipulate the page DOM

The Worker does not have the page's `document`.

## Rule 3 — Communication is explicit

Typical mechanism:

```js
postMessage()
```

and:

```js
message
```

events.

## Rule 4 — Ordinary messages are not shared object references

Structured cloning creates equivalent data in the receiving context.

## Rule 5 — Transfer moves ownership

Some objects can be transferred rather than cloned.

## Rule 6 — `SharedArrayBuffer` intentionally shares memory

No ordinary copy is made for the shared memory itself.

## Rule 7 — Shared memory requires synchronization

Use:

```text
Atomics
```

as required by the algorithm.

## Rule 8 — Data races are real

Shared mutable memory creates concurrency hazards.

## Rule 9 — Workers have independent lifecycles

Create and terminate them deliberately.

## Rule 10 — More Workers are not always better

Too many workers can create:

```text
memory pressure
context-switching
scheduling overhead
message contention
```

## Rule 11 — Worker communication can dominate computation

For tiny tasks:

```text
postMessage + clone
```

can cost more than simply doing the work locally.

## Rule 12 — Prefer message passing when possible

Shared memory should be introduced only when its advantages justify synchronization complexity.

## Rule 13 — Service Workers are different

A service worker is event-driven and lifecycle-managed by the browser for network/cache/background capabilities.

It is not simply a permanently running Dedicated Worker.

## Rule 14 — API availability differs by context

Window APIs and Worker APIs are not identical.

## Rule 15 — Cross-origin isolation can matter for shared memory

Shared-memory capabilities are subject to browser security/isolation requirements.

## Rule 16 — Worker correctness must include failure

Workers can:

```text
throw
terminate
lose availability
be replaced
```

Applications must handle this.

---

# 7. Execution Contexts and Agents

## 7.1 Window Context

A typical page has:

```text
Window
+
Document
+
JavaScript global
```

---

## 7.2 Worker Context

A Worker has its own global environment:

```js
self
```

and:

```js
globalThis
```

but no ordinary page `window`/`document`.

---

## 7.3 Agent Connection

Chapter 44 introduced agents.

A simplified mental model:

```text
Window Agent
    |
Worker Agent A
Worker Agent B
```

These agents execute independently.

---

## 7.4 Event Loop

Each relevant execution environment has event-loop behavior appropriate to its agent/context.

Therefore:

```text
Main event-loop work
```

does not simply execute inside:

```text
Worker event loop
```

---

## 7.5 Parallelism

Workers can enable genuine concurrent execution.

This differs from:

```text
Promise
async/await
```

which provide asynchronous control flow but do not inherently create parallel execution contexts.

---

# 8. Dedicated Workers

A Dedicated Worker is associated with one creating context.

Create:

```js
const worker = new Worker("/worker.js");
```

Worker:

```js
self.onmessage = event => {
  const result = compute(event.data);
  self.postMessage(result);
};
```

Main:

```js
worker.postMessage(10);

worker.onmessage = event => {
  console.log(event.data);
};
```

---

## 8.1 Lifetime

A Dedicated Worker generally remains available while references/lifecycle conditions keep it relevant and until terminated or otherwise stopped by the platform.

Explicitly terminate:

```js
worker.terminate();
```

---

## 8.2 Error Handling

```js
worker.onerror = event => {
  console.error(event.message);
};
```

Also consider:

```js
worker.onmessageerror = event => {
  console.error("Message error");
};
```

where relevant.

---

# 9. Shared Workers

A Shared Worker can be connected to by multiple same-origin browsing contexts under its defined connection model.

Example:

```js
const worker = new SharedWorker("/shared-worker.js");

worker.port.start();

worker.port.postMessage({
  type: "hello"
});
```

Worker:

```js
self.onconnect = event => {
  const port = event.ports[0];

  port.onmessage = messageEvent => {
    port.postMessage("hello back");
  };

  port.start();
};
```

---

## 9.1 Why Use a Shared Worker?

Potential uses:

```text
shared coordination
centralized per-origin state
resource sharing
communication among tabs
```

---

## 9.2 Port-Based Model

Communication commonly occurs through:

```text
MessagePort
```

rather than directly through the Worker object.

---

## 9.3 Lifecycle

Shared Workers can have multiple connected ports.

Cleanup requires reasoning about:

```text
which ports remain
which contexts remain
```

---

# 10. Service Workers

Service Workers are specialized Worker-like execution contexts integrated with browser-controlled:

```text
fetch
cache
notifications
push
background lifecycle
```

Service workers are covered more deeply in networking/application architecture.

---

## 10.1 Key Difference

A Dedicated Worker is usually:

```text
application-created and directly owned
```

A Service Worker is:

```text
browser-lifecycle managed
```

and can serve multiple controlled pages.

---

## 10.2 Event-Driven

Service Workers respond to events such as:

```js
self.addEventListener("fetch", event => {
  // ...
});
```

---

## 10.3 Not Always Alive

The browser can stop and restart a service worker.

Therefore:

> Never treat service-worker memory as permanent process-global state.

---

## 10.4 Persistent State

Store durable state in:

```text
IndexedDB
Cache Storage
other appropriate persistence
```

rather than assuming variables survive restarts.

---

# 11. Worker Globals

Inside a Worker:

```js
self
```

is the global worker scope object.

---

## 11.1 `globalThis`

```js
globalThis
```

is the standard global reference.

---

## 11.2 No `document`

```js
typeof document
```

is typically:

```text
"undefined"
```

in a Worker.

---

## 11.3 Timers

Workers can generally use:

```js
setTimeout()
setInterval()
```

subject to Worker/browser semantics.

---

## 11.4 `fetch`

Workers can use Fetch APIs in supported contexts:

```js
const response = await fetch(url);
```

---

## 11.5 Web APIs

Workers expose many APIs but not every Window-only API.

Always check documentation/feature detection.

---

# 12. Worker Creation and Lifecycle

## 12.1 Create

```js
const worker = new Worker("/worker.js");
```

---

## 12.2 Message

```js
worker.postMessage(data);
```

---

## 12.3 Receive

```js
worker.addEventListener("message", handler);
```

---

## 12.4 Error

```js
worker.addEventListener("error", handler);
```

---

## 12.5 Terminate

```js
worker.terminate();
```

---

## 12.6 Lifecycle State

Application architecture should model:

```text
created
→ initializing
→ ready
→ busy
→ stopping
→ terminated
```

even though the actual Worker API does not provide all of these application-level states.

---

# 13. `postMessage()` and Message Events

Basic main-thread code:

```js
worker.postMessage({
  type: "sum",
  values: [1, 2, 3]
});
```

Worker:

```js
self.addEventListener("message", event => {
  const { type, values } = event.data;

  if (type === "sum") {
    self.postMessage({
      type: "result",
      value: values.reduce((a, b) => a + b, 0)
    });
  }
});
```

---

## 13.1 Message Boundary

Think:

```text
sender object graph
       ↓
serialization / transfer boundary
       ↓
receiver object graph
```

---

## 13.2 No Shared Object Identity

```js
const data = { count: 1 };

worker.postMessage(data);
```

The receiver does not receive the exact same ordinary object identity.

---

## 13.3 Mutating After Send

Conceptually:

```js
const data = { count: 1 };

worker.postMessage(data);

data.count = 2;
```

The receiving side does not simply observe:

```text
same object
count changed to 2
```

because ordinary message passing is based on cloned/transferred data.

---

# 14. Structured Clone

The structured clone algorithm is used by many Web APIs to copy rich data structures.

Examples can include:

```text
objects
arrays
Map
Set
Date
RegExp
typed arrays
ArrayBuffer
```

subject to supported cloneability.

---

## 14.1 Why Not JSON?

JSON cannot preserve many JavaScript data structures:

```text
Map
Set
Date semantics
undefined
cycles
typed arrays
```

Structured clone exists for richer values.

---

## 14.2 Cycles

Structured cloning supports cyclic object graphs in contexts that use it.

Example:

```js
const obj = {};
obj.self = obj;

worker.postMessage(obj);
```

This is not equivalent to:

```js
JSON.stringify(obj);
```

which fails on the cycle.

---

## 14.3 Functions

Functions are generally not cloneable through structured cloning.

This is why:

```js
worker.postMessage({
  fn: () => {}
});
```

does not simply send executable function identity.

---

## 14.4 DOM Nodes

Ordinary DOM node identity is not transferred as a live page node through structured cloning.

---

# 15. Transferable Objects

Some objects can be transferred instead of cloned.

Common example:

```js
ArrayBuffer
```

Example:

```js
const buffer = new ArrayBuffer(1024);

worker.postMessage(buffer, [buffer]);
```

Ownership moves to the receiving context.

---

## 15.1 Detachment

After transfer, the sender's `ArrayBuffer` becomes detached.

Conceptually:

```text
sender
  buffer → no longer owns bytes

receiver
  buffer → owns bytes
```

---

## 15.2 Why Transfer?

For large binary data:

```text
clone
→ copy N bytes
```

can be expensive.

Transfer:

```text
move ownership
```

can avoid the large data copy.

---

## 15.3 Transfer Is Not Shared Memory

Transfer:

```text
one owner after transfer
```

SharedArrayBuffer:

```text
multiple contexts observe same memory
```

These are different models.

---

# 16. SharedArrayBuffer

`SharedArrayBuffer` represents memory that can be shared across agents.

Example:

```js
const shared = new SharedArrayBuffer(1024);

worker.postMessage(shared);
```

Both contexts can access the underlying shared memory.

---

## 16.1 Typed Array View

```js
const numbers = new Int32Array(shared);
```

---

## 16.2 Why This Is Powerful

No repeated copying is required for the shared data itself.

This can be valuable for:

```text
large numeric workloads
high-throughput communication
worker pools
ring buffers
real-time coordination
```

---

## 16.3 Why This Is Dangerous

Now two agents can mutate the same state.

Example:

```js
counter[0] += 1;
```

This is not necessarily atomic.

Conceptually:

```text
load
+
1
+
store
```

Two workers can interleave those operations.

---

## 16.4 Security Context

Modern browsers impose security/isolation requirements around shared memory.

Applications should verify the current browser requirements before deploying `SharedArrayBuffer` in production.

---

# 17. Atomics

`Atomics` provides atomic operations over shared typed-array memory.

Example:

```js
Atomics.add(counter, 0, 1);
```

This performs an atomic addition.

---

## 17.1 Load

```js
Atomics.load(view, index);
```

---

## 17.2 Store

```js
Atomics.store(view, index, value);
```

---

## 17.3 Compare-and-Exchange

```js
Atomics.compareExchange(
  view,
  index,
  expected,
  replacement
);
```

Useful for lock-free algorithms.

---

## 17.4 Add/Subtract

```js
Atomics.add(...)
Atomics.sub(...)
```

---

## 17.5 Wait / Notify

For suitable shared integer typed arrays:

```js
Atomics.wait(...)
Atomics.notify(...)
```

can coordinate agents.

---

## 17.6 WaitAsync

Modern environments can provide:

```js
Atomics.waitAsync(...)
```

for asynchronous waiting patterns where supported.

Feature support is version-sensitive.

---

# 18. Data Races and Memory Visibility

## 18.1 Race Condition

Suppose:

```js
shared[0] += 1;
```

is executed by two workers.

Possible reasoning:

```text
Worker A reads 0
Worker B reads 0
Worker A writes 1
Worker B writes 1
```

Final:

```text
1
```

instead of expected:

```text
2
```

---

## 18.2 Atomic Increment

```js
Atomics.add(shared, 0, 1);
```

provides an atomic read-modify-write operation.

---

## 18.3 Race vs Deadlock

### Race

Result depends on timing/interleaving.

### Deadlock

Workers wait forever for one another.

Example conceptual:

```text
A owns lock X
A waits for Y

B owns lock Y
B waits for X
```

---

## 18.4 Lost Update

Two agents:

```text
read
modify
write
```

can overwrite one another.

Atomic operations or ownership protocols solve this.

---

# 19. Producer / Consumer Concurrency

A classic Worker architecture:

```text
Producer
   ↓
queue
   ↓
Consumer Worker
```

For multiple workers:

```text
Producer
   ↓
shared / coordinated queue
   ├── Worker A
   ├── Worker B
   └── Worker C
```

---

## 19.1 Message-Based Queue

Simpler model:

```text
main thread
→ postMessage(task)
```

Worker:

```text
process task
→ postMessage(result)
```

---

## 19.2 Shared Ring Buffer

Advanced model:

```text
SharedArrayBuffer
+
Int32Array metadata
+
Atomics
```

This can avoid copying large messages.

But synchronization correctness becomes significantly harder.

---

# 20. Worker Pools

Creating a Worker for every tiny task is often wasteful.

Instead:

```text
Worker Pool
 ├── Worker 1
 ├── Worker 2
 ├── Worker 3
 └── Worker 4
```

Tasks:

```text
queue
→ available worker
→ process
→ return worker
```

---

## 20.1 Pool Size

A good pool size depends on:

```text
CPU cores
workload
memory
browser scheduling
other application work
```

Do not blindly create:

```js
navigator.hardwareConcurrency
```

Workers.

That value is a hint, not a universal pool-size prescription.

---

## 20.2 Oversubscription

Too many Workers can create:

```text
scheduling contention
cache pressure
memory pressure
```

---

# 21. CPU-Bound Work

Good Worker candidates often include:

```text
image processing
large calculations
parsing
compression
data transformation
search/indexing
cryptographic computation
ML preprocessing/inference
```

provided the API/runtime supports the required operations.

---

## 21.1 Poor Worker Candidate

Tiny task:

```js
x + y
```

Repeated through:

```text
postMessage
→ clone
→ context switch
→ execute
→ clone
→ message back
```

can be much slower than local execution.

---

# 22. Main Thread Responsiveness

The main goal often is not:

```text
maximum CPU throughput
```

but:

```text
keep interaction smooth
```

---

## 22.1 Long Main-Thread Task

```text
click
→ heavy computation
→ 500 ms blocked
```

causes:

```text
input delay
render delay
jank
```

---

## 22.2 Worker Offload

```text
click
→ send data
→ Worker computes
→ main thread remains available
→ result arrives
```

---

## 22.3 Worker Result Handling

Even if computation is offloaded, a huge result can become a new main-thread bottleneck.

Example:

```text
Worker computes 100 MB result
→ postMessage clone
→ main-thread receives
→ main thread processes 100 MB
```

The bottleneck moved; it did not disappear.

---

# 23. Messaging Cost and Backpressure

## 23.1 Message Rate

High-frequency:

```js
worker.postMessage(...)
```

can create a message storm.

---

## 23.2 Backpressure

If producer speed exceeds Worker processing:

```text
producer:
1000 tasks/s

worker:
100 tasks/s
```

the queue grows.

Eventually:

```text
memory ↑
latency ↑
```

---

## 23.3 Batching

Instead of:

```text
1000 messages
```

send:

```text
10 batches × 100 tasks
```

when semantics allow.

---

## 23.4 Transfer Large Binary Data

For binary pipelines, transfer:

```text
ArrayBuffer
MessagePort
ImageBitmap
```

or other suitable transferable objects when supported, rather than cloning large payloads.

---

## 23.5 Shared Memory

For ultra-high-throughput data paths:

```text
SharedArrayBuffer
+
Atomics
```

can reduce message-copy overhead.

The complexity trade-off is substantial.

---

# 24. Error Handling

## 24.1 Worker Exceptions

Unhandled Worker errors can surface through:

```js
worker.onerror = event => {};
```

---

## 24.2 Message Errors

```js
worker.onmessageerror = event => {};
```

can detect message deserialization/transfer problems.

---

## 24.3 Worker Failure Protocol

Production systems should have:

```text
worker state
request IDs
timeouts
retry policy
restart policy
error reporting
```

---

## 24.4 Worker Restart

If a Worker becomes unusable:

```js
worker.terminate();
```

and create a replacement.

---

## 24.5 Idempotency

If tasks can be retried after Worker failure:

```text
task
→ request ID
→ retry
```

must not accidentally produce duplicate externally visible effects.

---

# 25. Worker Termination

## 25.1 Explicit

```js
worker.terminate();
```

---

## 25.2 Why Terminate?

Unneeded Workers can consume:

```text
memory
CPU
scheduler resources
```

---

## 25.3 Cooperative Shutdown

You can also message:

```js
worker.postMessage({
  type: "shutdown"
});
```

Worker:

```js
self.addEventListener("message", event => {
  if (event.data.type === "shutdown") {
    // cleanup
    self.close();
  }
});
```

This allows cleanup before termination.

---

## 25.4 `terminate()` vs `close()`

Conceptually:

```text
Worker.terminate()
→ owner tells Worker to stop

self.close()
→ Worker requests its own global context close
```

Use the mechanism appropriate to ownership/lifecycle.

---

# 26. Worker and Browser API Availability

Workers can use many APIs, but not all Window APIs.

Commonly available in workers:

```text
fetch
timers
Web Crypto
WebSocket in supported environments
IndexedDB
BroadcastChannel
```

depending on context/browser.

---

## 26.1 DOM APIs

Not normally available:

```js
document
window
HTMLElement
```

---

## 26.2 Worker-Specific APIs

Workers have APIs such as:

```js
self
postMessage
importScripts
```

with differences depending on worker type/module configuration.

---

## 26.3 Feature Detection

Use:

```js
if (typeof Worker === "function") {
  // supported
}
```

and context-appropriate checks.

---

# 27. Workers and DOM Access

This is central.

A Dedicated Worker cannot do:

```js
document.querySelector("#app");
```

because it does not own the Window document.

Instead:

```text
Worker
→ calculate
→ postMessage(result)
→ main thread
→ DOM update
```

---

## 27.1 Why This Design?

The browser maintains strong ownership and consistency relationships around document/UI state.

Moving DOM manipulation into arbitrary concurrent agents would introduce much more complicated synchronization requirements.

---

## 27.2 Shared State Alternative

Do not attempt to share arbitrary DOM nodes.

Share data:

```text
model
state
buffers
messages
```

then let the main context update DOM.

---

# 28. Workers and Timers

Workers can generally use:

```js
setTimeout
setInterval
```

Example:

```js
setInterval(() => {
  performBackgroundWork();
}, 1000);
```

---

## 28.1 Separate Scheduling Context

The timer is scheduled in the Worker context.

It does not directly block the main Window context.

---

## 28.2 Worker Still Can Be Blocked

This:

```js
setInterval(work, 10);

while (true) {
  // CPU-heavy loop
}
```

blocks the Worker context itself.

A Worker is not magically immune to synchronous blocking.

---

# 29. Workers and Fetch / Networking

Workers can use:

```js
fetch()
```

in supported contexts.

Example:

```js
self.addEventListener("message", async event => {
  const response = await fetch(event.data.url);
  const data = await response.json();

  self.postMessage(data);
});
```

---

## 29.1 Why This Is Useful

You can move:

```text
network request
+
parsing
+
CPU transformation
```

into a Worker.

---

## 29.2 Beware Main-Thread Result Processing

Sending a huge parsed result back to the main thread can recreate the bottleneck.

Sometimes send only:

```text
small summary
```

or:

```text
incremental chunks
```

---

# 30. Workers and Storage

Workers can often use:

```text
IndexedDB
Cache Storage
```

as supported by context.

---

## 30.1 Background Data Processing

A Worker can:

```text
read data
→ transform
→ index
→ store
```

without tying CPU-heavy work to the Window context.

---

## 30.2 Shared Storage Coordination

Multiple contexts writing the same logical data may require:

```text
Web Locks
versioning
transactions
conflict handling
```

---

# 31. Shared Workers and Cross-Context Coordination

Imagine:

```text
Tab A
   \
Tab B → Shared Worker
   /
Tab C
```

The Shared Worker can act as a coordination hub.

---

## 31.1 Use Cases

Examples:

```text
shared connection
cross-tab computation
centralized indexing
shared application coordinator
```

---

## 31.2 Alternatives

Consider:

```text
BroadcastChannel
Web Locks
Service Worker
Shared Worker
```

depending on whether you need:

```text
message fan-out
exclusive ownership
network interception
long-lived shared computation
```

---

# 32. Service Worker Concurrency Model

Service Workers use a browser-managed lifecycle.

Important principle:

```text
do not assume persistent in-memory state
```

The browser can stop the worker when it is idle.

---

## 32.1 Event-Driven

Examples:

```text
fetch
push
notificationclick
sync-related events where supported
```

---

## 32.2 `event.waitUntil()`

For asynchronous service-worker work:

```js
self.addEventListener("install", event => {
  event.waitUntil(
    caches.open("v1")
  );
});
```

This tells the browser that lifecycle-critical asynchronous work is still active.

---

## 32.3 Why This Matters

Without lifecycle signaling:

```text
browser may consider event work complete
→ worker can be stopped
```

---

# 33. Synchronization Patterns

## 33.1 Ownership Instead of Shared Mutation

Prefer:

```text
Worker owns state
→ main thread sends commands
→ Worker sends results
```

over:

```text
everyone mutates one shared object
```

when practical.

---

## 33.2 Message Passing

Simple pattern:

```text
request
→ queue
→ worker
→ response
```

---

## 33.3 Lock

With shared memory:

```text
acquire
→ critical section
→ release
```

---

## 33.4 Compare-and-Swap

Conceptually:

```text
if state == expected:
    state = newState
```

implemented through:

```js
Atomics.compareExchange(...)
```

---

## 33.5 Producer / Consumer

```text
producer
→ bounded queue
→ consumers
```

---

## 33.6 Ring Buffer

Efficient shared-memory pattern:

```text
head
tail
buffer
```

synchronized with Atomics.

---

## 33.7 Work Stealing

Advanced worker pools can allow workers to take tasks from other queues.

This is substantially more complex and only worthwhile for suitable workloads.

---

# 34. Common Concurrency Patterns

## Pattern 1 — Request/Response

```text
Main → Worker
     task
Worker → Main
     result
```

---

## Pattern 2 — Fire-and-Forget

```text
Main → Worker
     task
```

No result required.

Use carefully because backpressure becomes harder to observe.

---

## Pattern 3 — Streaming

```text
Worker
→ chunk
→ chunk
→ chunk
→ done
```

Useful for large results.

---

## Pattern 4 — Worker Pool

```text
queue
 ↓
worker A
worker B
worker C
```

---

## Pattern 5 — Shared Coordinator

```text
Tabs
 ↓
Shared Worker
 ↓
central coordination
```

---

## Pattern 6 — Shared Memory Pipeline

```text
producer
 ↓
SharedArrayBuffer
 ↓
Atomics
 ↓
consumer workers
```

Advanced and synchronization-heavy.

---

# 35. Anti-Patterns

## Anti-Pattern 1 — Worker Per Tiny Task

Bad:

```text
create Worker
→ x + 1
→ terminate Worker
```

The overhead can dominate.

---

## Anti-Pattern 2 — Thousands of Workers

More concurrency can increase total cost.

---

## Anti-Pattern 3 — Shared Memory Everywhere

Use `SharedArrayBuffer` only when message-passing/transfer models are insufficient.

---

## Anti-Pattern 4 — Giant Result Transfer

Worker computes:

```text
500 MB
```

then sends it all to main.

The transfer/processing stage can become the real bottleneck.

---

## Anti-Pattern 5 — No Backpressure

Producer continuously sends tasks while worker falls behind.

---

## Anti-Pattern 6 — Hidden Shared State

Using shared memory without a documented synchronization protocol creates race conditions.

---

## Anti-Pattern 7 — Worker as a Magic Speed Button

A Worker changes execution architecture; it does not make algorithms asymptotically better.

---

## Anti-Pattern 8 — Ignoring Lifecycle

Workers left alive after UI teardown can waste resources.

---

## Anti-Pattern 9 — Retry Without Idempotency

A Worker failure can cause duplicate side effects if tasks are blindly retried.

---

## Anti-Pattern 10 — Global Shared Locks

Oversized critical sections destroy concurrency.

---

# 36. Edge Cases

## 36.1 Worker Startup Cost

Creating a Worker involves:

```text
script/module loading
initialization
execution-context creation
```

Cost depends on browser/version/device.

---

## 36.2 Module Worker

Modern applications can use module Workers:

```js
new Worker("/worker.js", {
  type: "module"
});
```

This integrates with module loading semantics differently from classic Workers.

---

## 36.3 Worker Terminated While Busy

If:

```js
worker.terminate();
```

occurs during computation, outstanding work is abandoned.

Design task ownership accordingly.

---

## 36.4 Message Ordering

Messages from a given communication path follow defined delivery semantics, but multi-source concurrent systems can still produce application-level ordering complexity.

Do not assume:

```text
Worker A + Worker B
→ global total order
```

without designing one.

---

## 36.5 Shared Memory Visibility

Without appropriate atomic synchronization, concurrent reads/writes create races and visibility problems.

---

## 36.6 `Atomics.wait`

Blocking waits are only available in contexts where the API permits them and on suitable integer shared-array views.

Do not assume every Worker context/API combination can call it.

---

## 36.7 SharedArrayBuffer Availability

Browser security/isolation requirements matter.

An application can have:

```text
SharedArrayBuffer undefined
```

or restricted behavior depending on context/configuration.

---

## 36.8 Cross-Origin Worker Script

Worker creation is constrained by browser security, script loading, origin, and CORS/fetch rules.

---

## 36.9 Credentials

Worker network requests can follow Fetch credential/origin rules that differ from simplistic “same as main page” assumptions.

---

## 36.10 IndexedDB

Worker database operations still need transaction/version/lifecycle handling.

---

## 36.11 Worker Error Recovery

Restarting a Worker can lose:

```text
in-memory queue
partial computation
local state
```

Persist/recover important state explicitly.

---

## 36.12 Service Worker Restart

Never depend on:

```js
let cache = new Map();
```

surviving service-worker restarts.

---

## 36.13 Worker Global `self`

`self` is context-specific and should not be confused with:

```text
Window.self
```

as a universal identical object.

---

## 36.14 Transfer List Mistakes

When transferring an `ArrayBuffer`, the sender cannot continue treating it as if it still owned the bytes.

---

# 37. Common Misconceptions

## Misconception 1 — “Web Workers are just async functions.”

No. They provide separate execution contexts.

## Misconception 2 — “Workers always run on separate CPU cores.”

Not guaranteed.

The API exposes concurrency; the browser/OS schedules actual execution.

## Misconception 3 — “Workers have the DOM.”

No.

## Misconception 4 — “postMessage shares the same object.”

Ordinary messages use structured cloning/transfer semantics.

## Misconception 5 — “Transfer and clone are the same.”

No.

Transfer changes ownership.

## Misconception 6 — “SharedArrayBuffer copies memory.”

No. It provides shared memory.

## Misconception 7 — “Shared memory removes synchronization.”

No. It makes synchronization more important.

## Misconception 8 — “Atomics make the entire application thread-safe.”

No. Atomic operations cover specific memory operations; your algorithm can still deadlock or contain races.

## Misconception 9 — “More Workers = more performance.”

False.

## Misconception 10 — “A Worker makes a bad algorithm good.”

No.

## Misconception 11 — “Service Workers are always running.”

No. Their lifecycle is browser-controlled.

## Misconception 12 — “Worker state survives page lifecycle automatically.”

No.

## Misconception 13 — “Worker result is free because computation happened elsewhere.”

Serialization/transfer and main-thread processing still cost resources.

## Misconception 14 — “BroadcastChannel is the same as SharedArrayBuffer.”

No.

Messaging vs shared memory.

## Misconception 15 — “Worker code can directly call document.querySelector().”

No.

---

# 38. Common Mistakes

## Mistake 1 — Using Workers for trivial work

Communication cost can exceed computation cost.

## Mistake 2 — Sending huge cloned objects

Prefer smaller messages or transferables.

## Mistake 3 — Sending huge results back

The main thread can become the bottleneck.

## Mistake 4 — Creating a new Worker per task

Prefer a pool for repeated CPU work.

## Mistake 5 — No task IDs

Responses become difficult to correlate.

Use:

```js
{
  id,
  type,
  payload
}
```

---

## Mistake 6 — No backpressure

Bound queues.

---

## Mistake 7 — Unbounded shared-memory queues

Memory and correctness become difficult.

---

## Mistake 8 — Locking too broadly

Keep critical sections minimal.

---

## Mistake 9 — Ignoring termination

Workers should have explicit ownership.

---

## Mistake 10 — Restarting without state recovery

Failed workers can lose in-memory task state.

---

# 39. Comparison With Related Concepts

| Concept | Primary purpose | Parallel execution? | Shared ordinary objects? |
|---|---|---:|---:|
| Function call | Execute code | No new context | Yes |
| Promise | Async control flow | Not inherently | No |
| `async`/`await` | Async syntax/control | Not inherently | No |
| `setTimeout` | Schedule callback | Not inherently | No |
| Worker | Separate execution context | Can enable concurrent work | No by default |
| Shared Worker | Shared Worker context | Yes | No by default |
| Service Worker | Browser-managed background context | Yes | No by default |
| SharedArrayBuffer | Shared memory | Enables shared state | Yes, memory |
| Atomics | Synchronize shared memory | Supports safe coordination | Shared memory only |

---

## Worker vs Promise

```text
Promise
→ asynchronous continuation

Worker
→ separate execution context
```

You can use both together:

```text
Worker
→ Promise-based request API
```

---

## Worker vs Process

A browser Worker is not a user-controlled OS process API.

Browser process architecture is implementation-specific.

---

## Worker vs Thread

A Worker is a Web Platform abstraction.

A thread is an OS/runtime implementation concept.

---

## Worker vs Service Worker

### Dedicated Worker

```text
application-owned
direct connection
explicit termination
```

### Service Worker

```text
browser-managed lifecycle
event-driven
scope-controlled
can handle fetch-related events
```

---

## Worker vs Shared Worker

### Dedicated

One owner/connection.

### Shared

Multiple same-origin contexts can connect through ports.

---

# 40. Performance Considerations

## 40.1 Model Total Cost

For a Worker task:

```text
Total cost
=
worker startup
+
message preparation
+
clone/transfer
+
queueing
+
compute
+
result transfer
+
main-thread handling
```

Compare that to:

```text
local compute
```

before deciding.

---

## 40.2 Compute-to-Communication Ratio

Workers are most attractive when:

```text
compute cost
≫
communication cost
```

For:

```text
1 μs compute
+
100 μs messaging
```

a Worker is usually a poor design.

For:

```text
500 ms compute
+
small input/output
```

a Worker can be highly valuable.

These numbers are illustrative, not universal measurements.

---

## 40.3 Chunking

Large computation can be:

```text
task
→ chunk
→ yield
```

even inside a Worker to keep that Worker responsive to control messages.

---

## 40.4 Pooling

Reuse Workers:

```text
create
→ process many jobs
→ terminate on lifecycle end
```

---

## 40.5 Batching

Reduce:

```text
message count
```

by batching suitable work.

---

## 40.6 Transferables

Use transferables for large binary payloads when supported.

---

## 40.7 Shared Memory

Use shared memory only when:

```text
copy/communication cost
```

is genuinely the bottleneck and synchronization complexity is justified.

---

## 40.8 Main-Thread Output

A Worker that sends thousands of tiny UI updates can still overwhelm the main thread.

Batch or throttle visible updates.

---

## 40.9 Worker Count

Use measured concurrency.

More workers can increase contention.

---

# 41. Memory Considerations

## 41.1 Worker Heap

Each Worker has memory associated with its execution context.

---

## 41.2 Clone Cost

Cloning large graphs creates receiver-side memory.

Potentially:

```text
input copy
+
output copy
```

---

## 41.3 Transfer Cost

Transfer avoids copying the underlying `ArrayBuffer` payload in the usual conceptual model, but metadata and coordination still have costs.

---

## 41.4 Shared Memory

`SharedArrayBuffer` avoids duplicate backing memory for the shared region.

But it can create:

```text
long-lived shared state
```

that is more difficult to reason about.

---

## 41.5 Worker Pool Memory

A pool of:

```text
N workers
```

has approximately:

```text
N × per-worker runtime overhead
```

plus application data.

---

## 41.6 Message Queues

Unbounded pending messages can create memory growth.

Implement:

```text
queue limit
backpressure
drop/coalesce policy
```

where semantics permit.

---

## 41.7 Worker Shutdown

Termination should release resources that are no longer needed.

---

# 42. Security Considerations

## 42.1 Worker Does Not Mean Trusted

Worker code executes with the application's privileges appropriate to its origin/context.

---

## 42.2 Untrusted Input

Workers should validate message data just like main-thread code.

---

## 42.3 Shared Memory

Shared-memory capabilities historically created serious side-channel concerns, which is one reason modern browsers impose isolation requirements around `SharedArrayBuffer`.

Always verify current browser requirements before relying on it.

---

## 42.4 Cross-Origin Isolation

Modern `SharedArrayBuffer` use in browsers generally depends on cross-origin isolation being correctly configured.

This commonly involves response headers such as:

```text
Cross-Origin-Opener-Policy
Cross-Origin-Embedder-Policy
```

Verify the current browser requirements and deployment model.

---

## 42.5 Worker Script Loading

Worker scripts are still subject to:

```text
origin
CORS/fetch
Content Security Policy
```

and related browser rules.

---

## 42.6 XSS

A Worker does not neutralize unsafe DOM operations performed by the main page.

---

## 42.7 Cryptographic Work

Workers can be useful for CPU-heavy cryptographic processing, but security-critical cryptography should use well-reviewed platform APIs rather than custom algorithms.

---

## 42.8 Message Authentication

Same-origin application components can trust their own architectural messages differently from cross-origin messages.

Cross-origin communication should use explicit origin validation and appropriate `postMessage` patterns.

---

# 43. Production Usage

## 43.1 CPU-Bound Worker Service

Architecture:

```text
UI
 ↓
Task Manager
 ↓
Worker Pool
 ├── Worker A
 ├── Worker B
 └── Worker C
```

Task format:

```js
{
  id,
  type,
  payload
}
```

Response:

```js
{
  id,
  ok,
  result,
  error
}
```

---

## 43.2 Task Manager Responsibilities

```text
queue
worker assignment
timeouts
retries
failure detection
shutdown
metrics
```

---

## 43.3 Worker Responsibilities

```text
validate task
perform pure computation
return result
avoid DOM assumptions
report errors
```

---

## 43.4 Main Thread Responsibilities

```text
input
DOM
rendering
UX
task scheduling
```

---

## 43.5 Cancellation

Workers do not automatically inherit `AbortSignal` semantics for arbitrary computation.

Design cooperative cancellation:

```js
worker.postMessage({
  type: "cancel",
  id
});
```

Worker checks between chunks.

---

## 43.6 Worker Pool Backpressure

The task manager can cap:

```text
pending jobs
```

and reject/defer work when overloaded.

---

## 43.7 Observability

Track:

```text
queue depth
job duration
worker utilization
worker restart count
message bytes
clone/transfer volume
task failure rate
main-thread impact
```

---

## 43.8 Memory Budget

Define:

```text
max workers
max queue
max message payload
max shared buffer size
```

where appropriate.

---

## 43.9 Production Decision Framework

| Dimension | Question |
|---|---|
| Correctness | Can tasks be executed concurrently without invalid state? |
| Performance | Is computation large enough to justify Worker overhead? |
| Memory | What is per-worker and per-queue memory cost? |
| Security | What data/capabilities cross the boundary? |
| Reliability | What happens when a Worker crashes or is terminated? |
| Accessibility | Is the main UI kept responsive? |
| Maintainability | Are message contracts versioned and explicit? |
| Scalability | Does the pool remain stable under overload? |
| Observability | Can queue depth and worker latency be measured? |
| Operational Complexity | Is shared memory really necessary? |
| Future Change | Will browser support/context requirements evolve? |

---

# 44. Implementation From Scratch

Build a **toy Worker runtime**.

## Stage 1 — Isolated Context

Create a simulated:

```js
WorkerContext
```

with:

```text
global object
task queue
message queue
```

---

## Stage 2 — MessagePort

Implement:

```text
sender
receiver
message queue
```

---

## Stage 3 — Structured Clone

Implement a restricted clone function that supports:

```text
number
string
boolean
null
arrays
plain objects
Date
Map
Set
cycles
```

and rejects:

```text
functions
DOM-like nodes
```

---

## Stage 4 — Transferable Buffer

Implement:

```text
ArrayBuffer
owner
transfer
detached state
```

Test:

```text
sender loses access
receiver gains ownership
```

---

## Stage 5 — Worker Lifecycle

Implement:

```text
create
ready
receive
send
terminate
```

---

## Stage 6 — Task IDs

Define:

```js
{
  id,
  type,
  payload
}
```

and match:

```text
request
→ response
```

---

## Stage 7 — Worker Pool

Implement:

```text
N workers
task queue
worker availability
assignment
completion
```

---

## Stage 8 — Backpressure

Set:

```text
maxQueueSize
```

and implement:

```text
reject
drop
coalesce
```

strategies.

---

## Stage 9 — Shared Memory

Implement a toy shared integer array:

```text
SharedBuffer
```

---

## Stage 10 — Atomic Operations

Implement:

```text
load
store
add
compareExchange
```

with a mutex in the toy runtime.

The goal is semantic understanding, not hardware-accurate implementation.

---

## Stage 11 — Producer / Consumer

Implement:

```text
producer
→ bounded queue
→ worker pool
```

---

## Stage 12 — Ring Buffer

Implement:

```text
head
tail
buffer
```

and test concurrent producer/consumer behavior.

---

## Stage 13 — Failure Recovery

Simulate:

```text
worker crash
timeout
message failure
queue overflow
```

and verify recovery.

---

# 45. Debugging Exercises

## Exercise 1 — Basic Message

Main:

```js
const worker = new Worker("/worker.js");

worker.onmessage = event => {
  console.log(event.data);
};

worker.postMessage(42);
```

Worker:

```js
self.onmessage = event => {
  self.postMessage(event.data * 2);
};
```

Predict:

```text
84
```

---

## Exercise 2 — Object Mutation

Main:

```js
const data = { count: 1 };

worker.postMessage(data);

data.count = 2;
```

Explain why the Worker does not receive a shared ordinary-object reference.

---

## Exercise 3 — Transfer

```js
const buffer = new ArrayBuffer(1024);

worker.postMessage(buffer, [buffer]);
```

Explain what ownership change occurs.

---

## Exercise 4 — Shared Memory Race

Two Workers execute:

```js
shared[0] += 1;
```

a million times.

Why might the final value be less than the expected count?

---

## Exercise 5 — Atomic Increment

Replace with:

```js
Atomics.add(shared, 0, 1);
```

Explain the difference.

---

## Exercise 6 — Backpressure

Producer:

```text
1000 tasks/s
```

Worker:

```text
100 tasks/s
```

What happens if the queue is unbounded?

---

## Exercise 7 — Worker Pool

Why can this be better:

```text
4 long-lived Workers
```

than:

```text
1000 Workers created one at a time
```

for repeated CPU work?

---

## Exercise 8 — Main-Thread Result Bottleneck

Worker computes:

```text
100 MB result
```

and sends it back.

Why can the application still freeze?

---

## Exercise 9 — DOM Access

Inside Worker:

```js
console.log(document);
```

Predict the problem.

---

## Exercise 10 — Worker Termination

```js
worker.postMessage({
  type: "shutdown"
});

worker.terminate();
```

Why might this be racy if the Worker was expected to perform asynchronous cleanup?

---

## Exercise 11 — Task Ordering

Send:

```text
task A
task B
task C
```

to one Worker.

Track:

```text
arrival
start
finish
response
```

Then repeat with two Workers.

Explain why response completion order can differ from request order.

---

## Exercise 12 — Shared Lock

Design two Workers contending for:

```text
shared lock
```

and demonstrate:

```text
correct mutual exclusion
```

then introduce a deadlock.

---

# 46. Code Review Exercise

Review:

```js
function expensiveWork(items) {
  return Promise.all(
    items.map(item =>
      new Promise(resolve => {
        const worker = new Worker("/worker.js");

        worker.onmessage = event => {
          resolve(event.data);
          worker.terminate();
        };

        worker.postMessage(item);
      })
    )
  );
}
```

A developer says:

> “This is parallel and therefore maximally fast.”

## Problems

1. A Worker is created for every item.
2. Worker startup overhead may dominate.
3. There is no concurrency bound.
4. Thousands of items can create thousands of Workers.
5. There is no cancellation.
6. There is no timeout.
7. There is no worker error recovery.
8. There is no task queue/backpressure.
9. A large message payload may be cloned repeatedly.
10. `Promise.all()` retains all results.

---

## Better Architecture

Use a Worker pool:

```text
N workers
+
bounded task queue
+
request IDs
+
failure handling
+
backpressure
```

The number `N` should be measured against:

```text
device CPU
browser
workload
memory
UI requirements
```

---

# 47. Interview Questions

## Foundational

1. What is a Web Worker?
2. Why do Workers exist?
3. Is a Worker a thread?
4. Is a Worker a process?
5. Can a Worker access the DOM?
6. What is `self`?
7. What is the Worker global?
8. What is a Dedicated Worker?

## Messaging

9. How does `postMessage()` work?
10. What is structured cloning?
11. Why are functions not cloneable?
12. What are transferables?
13. What happens to an ArrayBuffer after transfer?
14. Transfer vs clone?
15. What is `MessagePort`?

## Shared Memory

16. What is SharedArrayBuffer?
17. Why does shared memory require synchronization?
18. What is a data race?
19. What is Atomics?
20. What is compare-and-exchange?
21. What is `Atomics.wait()`?
22. What is a deadlock?

## Architecture

23. How does a Worker differ from a Promise?
24. How does a Dedicated Worker differ from a Shared Worker?
25. How does a Service Worker differ from a Dedicated Worker?
26. When should you use a Worker?
27. When should you not use one?
28. Why are Worker pools useful?

## Performance

29. What costs are introduced by a Worker?
30. What is compute-to-communication ratio?
31. Why can too many Workers reduce performance?
32. Why can cloning large objects be expensive?
33. When should you use transferables?
34. When is SharedArrayBuffer justified?

## Principal-level

35. Design a Worker pool for a CPU-bound browser application.
36. How would you implement backpressure?
37. How would you recover from Worker crashes?
38. How would you make task retries idempotent?
39. How would you instrument Worker utilization?
40. How would you decide between message passing and shared memory?
41. How would you design a cross-tab computation service?
42. What security requirements would you verify before deploying SharedArrayBuffer?
43. How would you debug a page that is still janky even after moving computation to a Worker?

---

# 48. Predict-the-Output Exercises

## Exercise 1 — Basic Worker

Worker:

```js
self.onmessage = event => {
  self.postMessage(event.data + 1);
};
```

Main sends:

```js
worker.postMessage(41);
```

### Prediction

```text
42
```

---

## Exercise 2 — Message Copy

```js
const data = {
  value: 1
};

worker.postMessage(data);

data.value = 99;
```

### Prediction

The Worker receives a cloned value corresponding to the message payload at cloning time, not a shared ordinary-object reference.

---

## Exercise 3 — Shared Memory

```js
const buffer = new SharedArrayBuffer(4);
const view = new Int32Array(buffer);

view[0] = 10;
```

Send to a Worker.

If the Worker executes:

```js
Atomics.add(view, 0, 5);
```

### Prediction

The main context can subsequently observe:

```text
15
```

through the shared memory.

---

## Exercise 4 — Non-Atomic Increment

Two Workers repeatedly execute:

```js
shared[0] += 1;
```

### Prediction

Do not assume the final value equals the mathematical sum of all increments.

---

## Exercise 5 — Atomic Increment

Two Workers repeatedly execute:

```js
Atomics.add(shared, 0, 1);
```

### Prediction

The increments are atomic with respect to each other on the relevant shared integer element, so lost-update behavior from ordinary read-modify-write increments is avoided.

---

## Exercise 6 — Timer Inside Worker

Worker:

```js
console.log("A");

setTimeout(() => {
  console.log("C");
}, 0);

console.log("B");
```

### Prediction

```text
A
B
C
```

The timer does not interrupt the synchronous Worker task.

---

# 49. Mastery Exercises

## Level 1 — Explain

Explain:

```text
Window
Worker
Realm
Agent
event loop
```

and how they relate.

---

## Level 2 — Build Request/Response Worker

Implement:

```text
main
→ request {id,type,payload}
→ worker
→ response {id,result}
```

with error handling.

---

## Level 3 — Add Cancellation

Add:

```text
cancel request
```

and cooperative checks inside long tasks.

---

## Level 4 — Build Worker Pool

Implement:

```text
queue
N workers
job assignment
completion
timeouts
```

---

## Level 5 — Add Backpressure

Create:

```text
max queue size
```

and define behavior for overload.

---

## Level 6 — Add Transferables

Process large binary data using:

```text
ArrayBuffer
```

and compare:

```text
clone
vs
transfer
```

---

## Level 7 — Shared Memory

Implement:

```text
SharedArrayBuffer
Int32Array
Atomics
```

for a producer/consumer queue.

---

## Level 8 — Race Detection

Create a deliberately racy increment example.

Then fix it with Atomics.

---

## Level 9 — Deadlock Lab

Build two locks:

```text
A → X then Y
B → Y then X
```

Demonstrate the deadlock.

Then establish a consistent global lock ordering.

---

## Level 10 — Service Worker Lifecycle

Build a minimal Service Worker and deliberately assume it stays alive.

Then observe why that architecture is wrong.

Refactor to persistent browser storage.

---

## Level 11 — Performance Lab

Compare:

```text
local compute
Worker per task
Worker pool
Worker pool + transfer
```

Measure:

```text
startup
throughput
latency
CPU
memory
main-thread responsiveness
```

---

## Level 12 — Principal Architecture

Design a browser application that:

```text
parses 500 MB of data
maintains interactive UI
supports multi-tab coordination
uses offline persistence
processes background calculations
```

Choose:

```text
main thread
Workers
Shared Worker
Service Worker
IndexedDB
BroadcastChannel
Web Locks
SharedArrayBuffer
```

and defend every choice.

---

# 50. Key Takeaways

1. Web Workers provide separate JavaScript execution contexts.
2. They exist primarily to enable concurrent/background computation and protect main-thread responsiveness.
3. A Worker is a Web Platform abstraction, not necessarily a literal OS thread.
4. Workers do not directly access the page's DOM.
5. Communication usually uses `postMessage()` and message events.
6. Ordinary messages use structured-clone semantics rather than sharing ordinary object identity.
7. Transferables move ownership and can avoid large data copies.
8. `SharedArrayBuffer` enables shared memory across agents.
9. Shared memory introduces races and synchronization requirements.
10. Atomics provide atomic operations and coordination primitives for shared memory.
11. Shared memory does not automatically make an application thread-safe.
12. Workers have independent event-loop/execution behavior.
13. Promises and async/await do not inherently create parallel execution contexts.
14. Dedicated Workers are directly associated with one creating context.
15. Shared Workers can coordinate multiple same-origin contexts through ports.
16. Service Workers have browser-managed, event-driven lifecycles.
17. Service Worker in-memory state cannot be treated as permanent.
18. Worker startup, messaging, serialization, and memory all have costs.
19. A Worker is most valuable when useful computation is large relative to communication overhead.
20. Worker pools are often better than creating a Worker for every small task.
21. Backpressure is essential when producers can outpace Workers.
22. Huge Worker results can move the bottleneck back to the main thread.
23. Lifecycle and failure handling are first-class production concerns.
24. Prefer message passing and explicit ownership when possible.
25. Use shared memory only when its performance value justifies synchronization complexity.
26. Security/isolation requirements matter for shared-memory features.
27. Good browser concurrency design is about controlled ownership, bounded communication, predictable scheduling, and explicit failure—not simply “using threads.”

---

# 51. Concept Connections

## Depends On

```text
Chapter 44 — Realms / Agents / Execution Isolation
        ↓
Chapter 45 — Memory / GC
        ↓
Chapter 47 — Engine Architecture
        ↓
Chapter 49 — DOM Architecture
        ↓
Chapter 50 — Browser Events
        ↓
Chapter 51 — Browser Web APIs
        ↓
Chapter 52 — Web Workers / Concurrency
```

## Builds Toward

```text
Chapter 53 — Web Streams / Data Flow
Chapter 54 — Web Components
Chapter 55 — Fetch / HTTP Networking
Chapter 56 — Browser Security
Chapter 57 — JS Security Engineering
Chapter 61 — Worker Threads / Child Processes / Cluster
Chapter 63 — Async Context / Diagnostics
Chapter 83 — Observability
Chapter 84 — Reliability
Chapter 85 — Performance
```

## Related Concepts

- Agent
- Realm
- event loop
- task queue
- Promise jobs
- structured clone
- transferables
- SharedArrayBuffer
- Atomics
- MessageChannel
- BroadcastChannel
- Web Locks
- IndexedDB
- Service Worker
- WebAssembly
- CPU scheduling
- concurrency
- parallelism

## Concepts Revisited

### Chapter 44 — Agents

Workers provide a practical application of the Agent model.

```text
separate execution
→ separate scheduling state
→ explicit communication
```

### Chapter 45 — Memory

Worker design introduces:

```text
per-worker memory
message copies
transfer ownership
shared memory
```

### Chapter 49 — DOM

Workers cannot directly mutate the page DOM.

The common architecture is:

```text
Worker computes
→ Main thread renders
```

### Chapter 50 — Events

Worker communication uses event-driven APIs:

```text
message
error
messageerror
```

### Chapter 51 — Browser APIs

Workers expose a different capability surface than Window contexts.

### Chapter 37 — Cancellation

Worker cancellation often needs a cooperative application protocol:

```text
cancel request
→ task checks
→ stop safely
```

---

## Why This Chapter Matters Later

Chapter 53 expands browser concurrency/data flow through Streams.

Chapter 55 applies concurrency and Workers to networking.

Chapter 56 applies isolation concepts to browser security.

Chapter 61 transfers these ideas into Node.js:

```text
Worker Threads
Child Processes
Cluster
```

The central reusable principle is:

```text
concurrency
≠
shared mutable state
```

A high-quality system often starts with:

```text
independent ownership
+
message passing
```

and introduces:

```text
shared memory
+
synchronization
```

only when there is a measured need.

---

# 52. Completion Criteria

## Fundamentals

- [ ] Define Worker.
- [ ] Explain why Workers exist.
- [ ] Distinguish Worker from Promise.
- [ ] Distinguish Worker from thread/process.
- [ ] Explain Worker vs Window.
- [ ] Explain Worker vs Service Worker.
- [ ] Explain Worker execution context/agent.

## Messaging

- [ ] Use `postMessage`.
- [ ] Handle message events.
- [ ] Use task IDs.
- [ ] Explain structured clone.
- [ ] Explain clone limitations.
- [ ] Explain transferables.
- [ ] Explain detached ArrayBuffer.

## Shared Memory

- [ ] Explain SharedArrayBuffer.
- [ ] Create typed-array views.
- [ ] Explain race conditions.
- [ ] Explain lost updates.
- [ ] Use Atomics.load/store.
- [ ] Use Atomics.add.
- [ ] Explain compare-and-exchange.
- [ ] Explain wait/notify.
- [ ] Explain deadlock.

## Worker Architecture

- [ ] Build request/response protocol.
- [ ] Build Worker pool.
- [ ] Implement queue.
- [ ] Implement backpressure.
- [ ] Implement timeout.
- [ ] Implement cancellation.
- [ ] Implement Worker restart.
- [ ] Design idempotent retries.

## Browser Context

- [ ] Explain Worker DOM restriction.
- [ ] Explain Worker globals.
- [ ] Explain Worker timers.
- [ ] Explain Worker Fetch.
- [ ] Explain Worker storage.
- [ ] Explain API capability differences.

## Performance

- [ ] Measure Worker startup.
- [ ] Measure message cost.
- [ ] Measure clone vs transfer.
- [ ] Compare local vs Worker execution.
- [ ] Compare Worker per task vs pool.
- [ ] Measure main-thread responsiveness.
- [ ] Measure queue depth.

## Memory

- [ ] Estimate per-worker memory.
- [ ] Diagnose queue growth.
- [ ] Diagnose retained Worker state.
- [ ] Explain clone memory.
- [ ] Explain shared-buffer lifetime.

## Security

- [ ] Understand shared-memory isolation requirements.
- [ ] Explain Worker origin/security model.
- [ ] Handle untrusted message data.
- [ ] Understand Worker script loading constraints.
- [ ] Avoid treating Worker isolation as a complete security sandbox.

## Principal Judgment

- [ ] Choose message passing vs shared memory.
- [ ] Choose worker count.
- [ ] Design backpressure.
- [ ] Design failure recovery.
- [ ] Design lifecycle ownership.
- [ ] Design production observability.

---

# 53. Revision / Retrieval Record

## First-Pass Retrieval

Without opening the chapter:

1. What problem do Web Workers solve?
2. Are Workers threads?
3. Can a Worker access the page DOM?
4. What is `postMessage()`?
5. What is structured cloning?
6. What is a transferable?
7. Why does ArrayBuffer become detached after transfer?
8. What is SharedArrayBuffer?
9. What is a data race?
10. What does Atomics.add solve?
11. What is compare-and-exchange?
12. What is a deadlock?
13. What is a Worker pool?
14. Why is backpressure necessary?
15. Why can a Worker make a program slower?
16. Why can a huge Worker result still block the UI?
17. What is the difference between Dedicated, Shared, and Service Workers?
18. Why cannot Service Worker memory be treated as permanent?
19. When is shared memory justified?
20. What security/isolation requirements apply to SharedArrayBuffer?

## Retrieval Table

| Concept | Can Explain? | Can Predict? | Can Implement? | Needs Revision? |
|---|---:|---:|---:|---:|
| Worker | [ ] | [ ] | [ ] | [ ] |
| Worker vs thread/process | [ ] | [ ] | [ ] | [ ] |
| Worker vs Promise | [ ] | [ ] | [ ] | [ ] |
| Worker context/agent | [ ] | [ ] | [ ] | [ ] |
| Dedicated Worker | [ ] | [ ] | [ ] | [ ] |
| Shared Worker | [ ] | [ ] | [ ] | [ ] |
| Service Worker | [ ] | [ ] | [ ] | [ ] |
| Worker lifecycle | [ ] | [ ] | [ ] | [ ] |
| postMessage | [ ] | [ ] | [ ] | [ ] |
| message events | [ ] | [ ] | [ ] | [ ] |
| structured clone | [ ] | [ ] | [ ] | [ ] |
| transferables | [ ] | [ ] | [ ] | [ ] |
| ArrayBuffer transfer | [ ] | [ ] | [ ] | [ ] |
| SharedArrayBuffer | [ ] | [ ] | [ ] | [ ] |
| Atomics | [ ] | [ ] | [ ] | [ ] |
| races | [ ] | [ ] | [ ] | [ ] |
| deadlock | [ ] | [ ] | [ ] | [ ] |
| producer/consumer | [ ] | [ ] | [ ] | [ ] |
| Worker pools | [ ] | [ ] | [ ] | [ ] |
| backpressure | [ ] | [ ] | [ ] | [ ] |
| Worker errors | [ ] | [ ] | [ ] | [ ] |
| Worker termination | [ ] | [ ] | [ ] | [ ] |
| Worker API availability | [ ] | [ ] | [ ] | [ ] |
| Worker + DOM | [ ] | [ ] | [ ] | [ ] |
| Worker + timers | [ ] | [ ] | [ ] | [ ] |
| Worker + fetch | [ ] | [ ] | [ ] | [ ] |
| Worker + storage | [ ] | [ ] | [ ] | [ ] |
| shared-memory security | [ ] | [ ] | [ ] | [ ] |

## Spaced Retrieval Schedule

### Day 0

Explain:

```text
main thread
Worker
message boundary
shared memory
```

without notes.

### Day 2

Explain:

```text
clone
transfer
share
```

as three different data-movement models.

### Day 7

Design a four-worker pool with backpressure and request IDs.

### Day 14

Implement and debug an atomic producer/consumer queue.

### Day 30

Defend whether a real application should use:

```text
message passing
transferables
SharedArrayBuffer
```

for a large binary workload.

---

# 54. Canonical References and Source Discipline

## Primary Worker References

### WHATWG HTML Standard

https://html.spec.whatwg.org/

Use for:

- Worker environments
- Dedicated Workers
- Shared Workers
- Service Workers integration
- event loops
- Worker lifecycle
- event/task semantics

---

### WHATWG Web Workers Standard

https://html.spec.whatwg.org/multipage/workers.html

Use for:

- Worker APIs
- DedicatedWorkerGlobalScope
- SharedWorker
- Worker lifecycle
- worker communication
- worker event behavior

---

## Structured Clone

### WHATWG HTML Structured Clone

https://html.spec.whatwg.org/multipage/structured-data.html#structured-clone

Use for:

- cloneable values
- serialization semantics
- transfer
- detachment concepts

---

## ECMAScript Shared Memory

### ECMA-262

https://tc39.es/ecma262/

Use for:

- SharedArrayBuffer language/runtime definitions
- Atomics
- typed-array shared-memory operations

---

## MDN

### Web Workers API

https://developer.mozilla.org/en-US/docs/Web/API/Web_Workers_API

### Worker

https://developer.mozilla.org/en-US/docs/Web/API/Worker

### SharedArrayBuffer

https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/SharedArrayBuffer

### Atomics

https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Atomics

### Structured Clone Algorithm

https://developer.mozilla.org/en-US/docs/Web/API/Web_Workers_API/Structured_clone_algorithm

Use MDN for practical browser behavior and compatibility reference.

---

## Security / Shared Memory

When working with `SharedArrayBuffer`, verify current browser requirements for:

```text
cross-origin isolation
COOP
COEP
secure deployment
browser support
```

Do not rely on historical blog posts because shared-memory browser policy has changed over time.

---

## Source Classification

For every claim, classify as:

```text
[ECMAScript]
[HTML Standard]
[Web Workers]
[Structured Clone]
[Browser-specific]
[Measured]
[Historical]
```

Examples:

```text
"Atomics.add is atomic"
→ [ECMAScript]

"Dedicated Worker uses postMessage"
→ [Web Workers]

"Worker cannot access the page DOM"
→ [Web Platform execution-context model]

"Chrome schedules Worker X on OS thread Y"
→ [Browser-specific]

"Worker pool improved p95 latency by 20%"
→ [Measured]
```

---

## Version-Sensitivity

Record:

```text
browser
browser version
device CPU
OS
Worker type
task size
message size
transfer/clone mode
worker count
profiling method
```

Concurrency performance depends heavily on device/browser/workload.

---

## Source-Derived Notes

The Web Platform defines Workers as separate execution contexts and provides dedicated/shared worker models with explicit messaging. The structured clone and transfer mechanisms define how data crosses contexts. ECMAScript defines SharedArrayBuffer and Atomics semantics, while browsers impose additional security/isolation requirements for shared-memory exposure.

Keep these layers separate:

```text
Worker API
→ Web Platform

Structured clone/transfer
→ Web Platform algorithms

SharedArrayBuffer/Atomics
→ ECMAScript

actual threads/processes
→ browser/OS implementation
```

---

# 55. Completion Snapshot

## Chapter Status

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

## Knowledge Snapshot

### I can explain

- [ ] Worker
- [ ] Worker vs thread/process
- [ ] Worker vs Promise
- [ ] Worker vs Window
- [ ] Worker vs Service Worker
- [ ] Realm/Agent relationship
- [ ] Dedicated Worker
- [ ] Shared Worker
- [ ] Service Worker
- [ ] Worker lifecycle
- [ ] Worker global
- [ ] postMessage
- [ ] message event
- [ ] structured clone
- [ ] transfer
- [ ] detached ArrayBuffer
- [ ] SharedArrayBuffer
- [ ] Atomics
- [ ] races
- [ ] deadlock
- [ ] producer/consumer
- [ ] Worker pools
- [ ] backpressure
- [ ] Worker failures
- [ ] termination
- [ ] Worker API availability
- [ ] Worker DOM restriction
- [ ] Worker timers
- [ ] Worker fetch
- [ ] Worker storage
- [ ] shared-memory security

### I can predict

- [ ] basic Worker message exchange
- [ ] cloned-object behavior
- [ ] transferred-buffer ownership
- [ ] shared-memory updates
- [ ] race-condition outcomes
- [ ] timer behavior in Workers
- [ ] Worker-pool completion order
- [ ] backpressure queue growth
- [ ] main-thread bottlenecks after Worker completion

### I can implement

- [ ] request/response Worker
- [ ] task IDs
- [ ] Worker pool
- [ ] task queue
- [ ] backpressure
- [ ] cancellation protocol
- [ ] timeout/retry
- [ ] transfer pipeline
- [ ] shared-memory counter
- [ ] atomic queue
- [ ] deadlock demonstration
- [ ] failure recovery

### I can debug

- [ ] Worker startup problems
- [ ] message errors
- [ ] serialization errors
- [ ] transfer ownership problems
- [ ] queue overload
- [ ] Worker crashes
- [ ] worker leaks
- [ ] race conditions
- [ ] deadlocks
- [ ] main-thread result bottlenecks
- [ ] SharedArrayBuffer availability

### I can defend

- [ ] Worker vs local execution
- [ ] pool size
- [ ] clone vs transfer
- [ ] message passing vs shared memory
- [ ] cancellation strategy
- [ ] retry strategy
- [ ] lifecycle architecture
- [ ] shared-memory security requirements
- [ ] observability model

---

## Final Principal-Level Test

Explain this statement without notes:

> **A Web Worker is not merely “JavaScript running in the background.” It is a separate execution context with its own scheduling and lifecycle, communicating explicitly with other contexts through messages or, in advanced cases, sharing memory through SharedArrayBuffer and coordinating with Atomics.**

Your explanation is complete only when you can connect:

```text
Window context
    ↓
task
    ↓
Worker message
    ↓
Worker execution context
    ↓
compute
    ↓
structured clone / transfer
    ↓
main context
    ↓
DOM/rendering
```

and, for advanced shared-memory systems:

```text
Agent A
   ↓
SharedArrayBuffer
   ↓
Atomics
   ↓
Agent B
```

while clearly distinguishing:

```text
message passing
transfer
shared memory
```

and explaining:

```text
startup cost
communication cost
backpressure
races
deadlocks
memory
failure
lifecycle
security isolation
```

The central mastery target of Chapter 52 is to stop thinking of browser concurrency as “making code async” and start thinking of it as **execution-context architecture with explicit ownership, communication, synchronization, lifecycle, and failure semantics**.