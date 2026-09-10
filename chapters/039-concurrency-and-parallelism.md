
# Chapter 39 — Concurrency and Parallelism

## 1. Learning Objectives

By the end of this chapter, the learner must be able to:

- Define concurrency precisely.
- Define parallelism precisely.
- Explain why concurrency does not require multiple CPU cores.
- Explain why parallelism requires multiple independent execution resources.
- Distinguish concurrency from asynchronous programming.
- Distinguish concurrency from multitasking, interleaving, and scheduling.
- Explain cooperative concurrency in JavaScript.
- Explain how multiple asynchronous operations can overlap while JavaScript execution on one agent remains serialized.
- Understand concurrency limits as an application-level resource policy.
- Explain why `Promise.all()` can create concurrency but is not itself a general concurrency scheduler.
- Explain sequential, fully concurrent, and bounded-concurrent execution.
- Explain the throughput/latency/resource-utilization trade-off of concurrency.
- Explain why more concurrency can initially improve throughput and eventually reduce it.
- Understand queueing, contention, saturation, and the “knee” of a throughput/latency curve.
- Explain Little's Law at a practical level and use it to reason about concurrency, throughput, and latency.
- Distinguish CPU-bound, I/O-bound, memory-bound, lock-bound, and dependency-bound workloads.
- Choose between event-loop concurrency, bounded asynchronous concurrency, worker threads, child processes, and external job queues.
- Explain structured concurrency as a design principle involving ownership, lifetime, failure propagation, and cancellation.
- Understand work queues, semaphores, pools, task groups, and executors.
- Implement bounded concurrency.
- Implement a semaphore and async task pool.
- Implement a worker queue with backpressure.
- Implement cancellation-aware concurrency control.
- Understand fairness and starvation.
- Understand head-of-line blocking.
- Understand work stealing at a conceptual level.
- Understand admission control and overload protection.
- Explain why unbounded concurrency can create memory leaks, rate-limit failures, database saturation, socket exhaustion, and cascading failure.
- Explain how concurrency interacts with retries, timeouts, cancellation, backpressure, and resource ownership.
- Explain how parallelism can improve CPU throughput while introducing synchronization and data-transfer costs.
- Understand worker-thread and process isolation trade-offs.
- Diagnose concurrency bugs including races, lost updates, duplicate work, starvation, deadlocks, live locks, queue growth, and oversubscription.
- Measure concurrency using active work, queue depth, throughput, latency, utilization, and saturation signals.
- Design production concurrency budgets for APIs, batch processing, streams, databases, and background jobs.
- Evaluate concurrency decisions through correctness, performance, memory, security, reliability, observability, maintainability, and operational complexity.
- Defend concurrency architecture at senior/principal level.

### Mastery Gate

The chapter is mastered only when the learner can:

> Understand → Explain → Predict → Implement → Debug → Apply → Compare → Defend

---

## 2. Prerequisites

The learner should understand:

- Functions and control flow.
- Execution contexts.
- Promises.
- Async/await.
- ECMAScript Jobs and Promise Reaction Jobs.
- Browser event loop.
- Node event loop and libuv.
- Cancellation.
- Async iteration and streaming.
- Resource management and cleanup.
- Basic performance measurement.

Primary dependencies:

- Chapter 31 — Asynchronous JavaScript Fundamentals
- Chapter 32 — ECMAScript Jobs / Promise Reactions
- Chapter 33 — Browser Event Loop
- Chapter 34 — Node Event Loop / libuv
- Chapter 35 — Promises
- Chapter 36 — Async/Await
- Chapter 37 — Cancellation / Abort
- Chapter 38 — Async Iteration / Streaming

Related background:

- Chapter 12 — Execution Contexts / Execution Model
- Chapter 29 — Errors / Error Handling
- Chapter 30 — Resource Management / Cleanup
- Chapter 45 — Memory / GC

Later chapters build directly on this chapter:

- Chapter 40 — Observables / Reactive
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
- Chapter 87 — Deterministic Async Testing
- Chapter 88 — Debugging Methodology
- Chapter 101 — Real-world Production Scenarios
- Chapter 107 — Job Queue
- Chapter 110 — Production JS Backend
- Chapter 111 — Large-scale JS Platform
- Chapter 121 — System Design
- Chapter 122 — Final Principal JS Project

---

## 3. What Is It?

**Concurrency** is the capability to have multiple units of work in progress during the same overall period, even if the actual execution of their instructions is interleaved rather than simultaneous.

**Parallelism** means multiple units of computation are executing at the same time on independent execution resources.

A single JavaScript execution agent can therefore support concurrency:

```text
Task A starts
Task A waits

Task B starts
Task B waits

Task C starts
Task C waits

A completes
B completes
C completes
```

The work overlaps in time even though JavaScript code executes one segment at a time on that agent.

Parallelism looks different:

```text
CPU core 1 → work A
CPU core 2 → work B
CPU core 3 → work C
```

The central distinction:

```text
Concurrency:
  multiple activities are in progress

Parallelism:
  multiple activities execute simultaneously
```

JavaScript applications use concurrency constantly:

```js
const a = fetchA();
const b = fetchB();

await Promise.all([a, b]);
```

This can overlap I/O.

For CPU-heavy work, parallelism may require:

- Web Workers;
- Node worker threads;
- child processes;
- native extensions;
- WebAssembly;
- external workers.

The critical engineering question is:

> How many units of work should be active at once, and on how many execution resources?

---

## 4. Why Does It Exist?

Real systems contain independent work:

```text
request A
request B
request C
database query
network request
file read
background job
CPU transformation
```

Serializing all work:

```text
A → B → C → D
```

can waste available resources.

But unlimited concurrency:

```text
A B C D E F G H ... millions
```

can overload:

- memory;
- databases;
- remote services;
- sockets;
- CPU;
- thread pools;
- queues.

Therefore production systems need **controlled concurrency**.

The goal is not:

> Maximum possible concurrency.

The goal is:

> The highest useful concurrency that remains within correctness, capacity, latency, and resource constraints.

Concurrency exists as an architectural tool for utilization.

Concurrency control exists because resources are finite.

---

## 5. Mental Model

Think in terms of:

```text
work
  ↓
admission
  ↓
queue
  ↓
active concurrency
  ↓
resource
  ↓
completion
```

Example:

```text
1000 tasks arrive
       ↓
admission control
       ↓
queue of waiting tasks
       ↓
10 active tasks
       ↓
results return
       ↓
next tasks admitted
```

The active concurrency is:

```text
10
```

The queue size is:

```text
990
```

This is often safer than launching 1000 operations simultaneously.

A second model:

```text
                     workload
                        │
                        ▼
                  concurrency
                        │
           ┌────────────┼────────────┐
           ▼            ▼            ▼
         CPU           I/O       dependency
           │            │            │
           ▼            ▼            ▼
       cores/workers   sockets   remote capacity
```

Concurrency must match the bottleneck.

---

## 6. Core Rules

### Rule 1 — Concurrency is not parallelism

Multiple operations can be in progress without simultaneously executing JavaScript instructions.

### Rule 2 — Async is not the same as concurrency

An async program can still be fully sequential:

```js
await A();
await B();
await C();
```

### Rule 3 — Parallelism is not free

Parallel execution introduces:

- synchronization;
- data transfer;
- scheduling;
- memory overhead;
- coordination complexity.

### Rule 4 — `Promise.all()` is aggregation, not a full scheduler

It waits on multiple Promises, but does not provide:

- queueing;
- bounded admission;
- fairness;
- backpressure;
- cancellation;
- priority.

### Rule 5 — More concurrency can increase throughput only until a bottleneck saturates

After saturation, additional concurrency can increase:

```text
queueing
latency
memory
timeouts
retries
```

without increasing useful throughput.

### Rule 6 — Every resource has a concurrency capacity

Examples:

```text
CPU cores
database connections
remote API rate limit
file descriptors
memory
worker pool
```

### Rule 7 — Concurrency should usually be bounded

The bound should be derived from measured capacity and workload characteristics.

### Rule 8 — Queueing is part of concurrency control

When active capacity is exhausted:

```text
new work → queue
```

or:

```text
reject / drop / defer
```

according to policy.

### Rule 9 — Cancellation should remove unnecessary work from the system

Queued work should ideally be cancellable before it consumes execution resources.

### Rule 10 — Backpressure prevents producer overload

If consumers cannot keep up:

```text
slow producer
or
bounded queue
or
drop/coalesce
```

must be considered.

### Rule 11 — Work should have an owner

Each active task needs:

```text
caller
scope
resource
completion
cancellation
```

### Rule 12 — Completion order is not start order

Concurrent tasks may finish in any order.

### Rule 13 — Result order and execution order are separate

```js
Promise.all([
  slow(),
  fast()
]);
```

returns results in input order even if execution completes differently.

### Rule 14 — Concurrency creates race opportunities

Shared mutable state becomes more difficult to reason about when operations interleave.

### Rule 15 — More workers can create oversubscription

If the machine has limited CPU resources:

```text
too many active CPU workers
→ context switching
→ cache pressure
→ lower throughput
```

### Rule 16 — CPU concurrency and I/O concurrency have different optimal values

Do not use one universal limit for every workload.

### Rule 17 — Fairness matters

One tenant or queue should not necessarily monopolize all available capacity.

### Rule 18 — Admission control is part of reliability

Rejecting work early can be safer than accepting work that cannot finish within useful deadlines.

### Rule 19 — Concurrency and retries multiply load

If concurrency is 100 and each failed request retries three times, effective pressure can become far larger than expected.

### Rule 20 — Observability is required to tune concurrency

Measure before tuning.

---

## 7. Syntax

### Sequential

```js
const a = await taskA();
const b = await taskB();
```

### Concurrent initiation

```js
const aPromise = taskA();
const bPromise = taskB();

const [a, b] = await Promise.all([
  aPromise,
  bPromise
]);
```

### Bounded concurrency pattern

```js
async function mapLimited(items, limit, worker) {
  // concurrency controller
}
```

### Semaphore-style API

```js
const semaphore = new Semaphore(10);

const release = await semaphore.acquire();

try {
  await work();
} finally {
  release();
}
```

### Queue

```js
const queue = new TaskQueue({
  concurrency: 10
});

queue.add(() => work());
```

### Worker threads

Node-style conceptual usage:

```js
new Worker("./worker.js");
```

### Browser worker

```js
const worker = new Worker("worker.js");
```

The worker provides a separate execution agent.

---

## 8. Basic Examples

### Example 1 — Sequential

```js
console.time("sequential");

await taskA();
await taskB();
await taskC();

console.timeEnd("sequential");
```

### Example 2 — Concurrent

```js
console.time("concurrent");

await Promise.all([
  taskA(),
  taskB(),
  taskC()
]);

console.timeEnd("concurrent");
```

### Example 3 — Bounded

```js
const limit = 5;

await mapLimited(items, limit, processItem);
```

Only five items are active at a time.

### Example 4 — Semaphore

```js
const semaphore = new Semaphore(2);

async function protectedOperation() {
  const release = await semaphore.acquire();

  try {
    return await work();
  } finally {
    release();
  }
}
```

### Example 5 — CPU parallelism

```text
main thread
   ↓
worker A
worker B
worker C
```

Independent CPU-heavy work can execute on multiple worker execution resources.

### Example 6 — Shared downstream limit

```js
const databaseSemaphore = new Semaphore(20);

async function query(sql) {
  const release = await databaseSemaphore.acquire();

  try {
    return await db.query(sql);
  } finally {
    release();
  }
}
```

This prevents the application from opening unlimited simultaneous database work.

---

## 9. Execution Walkthrough

Consider:

```js
const results = await Promise.all([
  fetchA(),
  fetchB(),
  fetchC()
]);
```

### Step 1

The first call expression begins:

```js
fetchA()
```

### Step 2

The second begins:

```js
fetchB()
```

### Step 3

The third begins:

```js
fetchC()
```

The important point is that all three operations are initiated before the outer `await` waits for the aggregate.

### Step 4

Suppose completion occurs:

```text
B
C
A
```

### Step 5

`Promise.all()` still produces:

```text
[A-result, B-result, C-result]
```

because result positions follow the input ordering.

### Step 6

The async function resumes when the aggregate Promise fulfills.

Now consider bounded concurrency:

```js
await mapLimited(items, 2, process);
```

Suppose the items are:

```text
A B C D
```

The scheduler may produce:

```text
start A
start B

A finishes
start C

B finishes
start D

C finishes
D finishes
```

At every point:

```text
active <= 2
```

This is the fundamental behavior of bounded concurrency.

---

## 10. Internal Mechanics

### 10.1 Concurrency is a scheduling property

A Promise does not itself decide:

```text
how many operations may exist simultaneously
```

The application or host determines that.

### 10.2 Semaphore

A semaphore maintains permits:

```text
capacity = N
```

Acquire:

```text
available permit?
→ take one

none?
→ wait
```

Release:

```text
permit returned
→ wake waiting task
```

### 10.3 Queue

A bounded executor commonly contains:

```text
task queue
+
active counter
+
completion handlers
+
cancellation
```

### 10.4 Pool

A pool creates a fixed number of reusable execution/resource slots:

```text
worker 1
worker 2
worker 3
worker 4
```

Tasks enter the queue and are assigned to available workers.

### 10.5 Event-loop concurrency

In browser/Node JavaScript:

```text
multiple I/O operations
```

can be in progress while:

```text
JavaScript callback execution
```

remains serialized on one agent.

### 10.6 Worker parallelism

A worker thread or Web Worker provides another execution agent.

Now:

```text
worker A → CPU computation
worker B → CPU computation
```

can execute simultaneously on separate underlying execution resources when the platform schedules them in parallel.

### 10.7 Queueing

Suppose:

```text
arrival rate = λ
service rate = μ
```

If arrivals consistently exceed processing capacity:

```text
queue length → grows
```

No amount of Promise syntax fixes an overloaded system.

### 10.8 Little's Law

For a stable system:

```text
L = λW
```

where:

```text
L = average work/items in system
λ = throughput
W = average time in system
```

This helps reason about concurrency.

Example:

```text
throughput = 100 requests/sec
average latency = 0.2 sec
```

Then the average in-flight work is approximately:

```text
100 × 0.2 = 20
```

This does not automatically tell you the ideal concurrency limit, but it gives a powerful sanity check.

### 10.9 Saturation

A resource has a useful capacity.

For example:

```text
database pool = 20
```

Launching:

```text
200 database operations
```

does not create 200 simultaneous database connections.

It creates contention/queueing around the limited resource.

### 10.10 Head-of-line blocking

Suppose:

```text
worker 1 → slow task
worker 2 → slow task
worker 3 → slow task
worker 4 → fast tasks
```

A queue design may cause fast work to wait behind slow work.

### 10.11 Fair scheduling

A production scheduler may need per-tenant or per-class limits:

```text
tenant A → 10
tenant B → 10
tenant C → 10
```

rather than one global pool consumed by whichever tasks arrive first.

### 10.12 Priority

Work can be:

```text
high priority
normal
background
```

But priority creates complexity:

```text
starvation
priority inversion
fairness
```

### 10.13 Work stealing

Thread pools can distribute tasks dynamically when workers become available.

The concept:

```text
idle worker
→ take work from another worker's queue
```

This can improve utilization but requires synchronization and careful runtime design.

### 10.14 Oversubscription

Suppose:

```text
8 CPU cores
64 CPU-heavy workers
```

The system may spend substantial time switching among work rather than computing.

### 10.15 I/O concurrency

I/O can tolerate higher concurrency than CPU work when:

- the operation mostly waits;
- downstream systems can handle it;
- memory remains bounded.

### 10.16 Memory-based concurrency

Sometimes the true limit is memory:

```text
one task = 50 MB
available budget = 500 MB
```

A concurrency limit much above 10 may be unsafe.

### 10.17 External quota-based concurrency

An API may allow:

```text
100 requests/sec
```

The concurrency limit should account for:

```text
request latency
rate limit
burst allowance
retry behavior
```

### 10.18 Connection-pool concurrency

A database may have:

```text
20 connections
```

The application should not pretend it can execute 1000 database transactions simultaneously without queueing.

### 10.19 Structured concurrency

A useful conceptual model:

```text
parent scope
  ├── child A
  ├── child B
  └── child C

parent owns:
  lifetime
  cancellation
  failure policy
```

The parent waits for or explicitly supervises child work.

### 10.20 Detached concurrency

Detached work:

```text
parent starts task
parent returns
task continues
```

can be valid but needs a separate owner/supervisor.

### 10.21 Failure propagation

A task group must define:

```text
one child fails
→ cancel siblings?
→ continue siblings?
→ fail parent?
```

There is no universal correct answer.

### 10.22 Concurrency collapse

Repeated failures can trigger retries:

```text
load high
→ timeout
→ retry
→ load higher
→ more timeouts
```

This creates a positive feedback loop.

### 10.23 Bulkheads

A bulkhead isolates capacity:

```text
critical requests → pool A
background jobs   → pool B
```

One workload cannot consume all resources.

### 10.24 Circuit breakers

When a dependency is failing:

```text
stop sending work temporarily
```

This reduces load and protects the rest of the system.

### 10.25 Admission control

When capacity is exhausted:

```text
reject
shed
queue
degrade
```

rather than accepting unlimited work.

---

## 11. ECMAScript / Specification Semantics

Concurrency is partly a language/runtime concern and partly an application architecture concern.

### 11.1 ECMAScript

ECMAScript provides:

- Jobs;
- Promises;
- async functions;
- async iterators;
- shared-memory primitives such as `Atomics` in relevant environments.

It does not define a universal:

```text
“maximum Promise concurrency”
```

mechanism.

### 11.2 Promise combinators

`Promise.all`, `race`, `any`, and `allSettled` define aggregate settlement semantics.

They do not define resource-aware scheduling policy.

### 11.3 Agents

ECMAScript execution agents provide the basis for separate execution contexts.

Multiple agents can support parallel execution in host environments.

### 11.4 SharedArrayBuffer and Atomics

Shared memory enables coordination among agents.

This introduces true concurrency concerns:

- races;
- atomicity;
- memory ordering;
- deadlock;
- livelock.

Later engine/runtime chapters cover this more deeply.

### 11.5 Host/runtime

Browsers and Node provide:

- workers;
- worker threads;
- processes;
- I/O facilities;
- queues;
- thread pools.

### 11.6 Application layer

Applications define:

- concurrency limits;
- priorities;
- fairness;
- retries;
- backpressure;
- overload behavior;
- ownership.

The source discipline is:

```text
language semantics
→ runtime mechanisms
→ application policy
```

Do not present a production semaphore implementation as an ECMAScript language feature.

---

## 12. Advanced Behavior

### 12.1 Sequential execution

```js
for (const item of items) {
  await process(item);
}
```

Concurrency:

```text
1
```

This can be desirable for:

- ordering;
- rate limits;
- transactional workflows.

### 12.2 Full fan-out

```js
await Promise.all(
  items.map(process)
);
```

Conceptually:

```text
concurrency ≈ number of items
```

This can be dangerous.

### 12.3 Bounded concurrency

```text
N active
rest queued
```

This is usually the right architecture for large batches.

### 12.4 Sliding-window concurrency

A scheduler can keep:

```text
N active operations
```

while continuously replacing completed tasks.

### 12.5 Batch concurrency

Process:

```text
batch 1
→ wait
→ batch 2
```

This is simpler but can leave capacity idle near batch boundaries.

### 12.6 Dynamic concurrency

A production scheduler may adapt based on:

```text
latency
error rate
CPU
memory
downstream saturation
```

For example:

```text
healthy dependency → increase concurrency
timeouts increase  → decrease concurrency
```

This resembles adaptive load control.

### 12.7 Fixed concurrency versus rate limiting

Concurrency controls:

```text
how many are active
```

Rate limiting controls:

```text
how many start per unit time
```

They are not equivalent.

### 12.8 Both concurrency and rate may be required

Example:

```text
max active = 20
max starts = 100/sec
```

### 12.9 Concurrency + timeout

Without timeout:

```text
one stuck task
→ occupies slot forever
→ effective capacity falls
```

Timeout releases capacity when appropriate.

### 12.10 Cancellation + queue

A cancelled queued task should ideally be removed or marked inactive:

```text
queued
  ↓ cancel
cancelled
```

It should not consume an active slot.

### 12.11 Concurrency + retry

Retrying failed tasks consumes concurrency slots.

Retry policies must be included in capacity calculations.

### 12.12 Exponential backoff

A common policy:

```text
delay = base × 2^attempt
```

plus jitter.

Backoff reduces synchronized retry storms.

### 12.13 Fan-out/fan-in

Common architecture:

```text
one request
   ↓
fan out
 ┌─┼─┐
 A B C
 └─┼─┘
   ↓
fan in
```

The fan-out must be bounded.

### 12.14 Failure domains

If:

```text
A fails
```

should B and C continue?

Possible policies:

```text
fail-fast
best-effort
partial success
cancel siblings
```

### 12.15 Partial results

Streaming and batch systems may produce partial progress before failure.

Concurrency design should define whether partial progress is committed.

### 12.16 Idempotency

Retries and concurrent duplication can produce:

```text
same operation twice
```

External side effects should be designed for idempotency where retries are possible.

### 12.17 Duplicate work

A concurrency bug can cause:

```text
same job
→ worker A
→ worker B
```

Deduplication may require:

- idempotency keys;
- locks;
- leases;
- unique constraints.

### 12.18 Race conditions

Example:

```js
let balance = 100;

async function spend(amount) {
  const current = await loadBalance();
  await delay(10);
  await saveBalance(current - amount);
}
```

Two calls can read the same balance and overwrite one another.

### 12.19 Lost update

Concurrency at the application layer must respect transactional guarantees at the database layer.

### 12.20 Lock contention

Parallel work may compete for:

- mutexes;
- database locks;
- file locks;
- memory bandwidth.

Higher concurrency can reduce useful throughput.

### 12.21 Deadlock

If two workers wait for resources in opposite order:

```text
A holds lock 1 → waits lock 2
B holds lock 2 → waits lock 1
```

neither progresses.

### 12.22 Livelock

Tasks remain active and change state but make no useful progress.

### 12.23 Starvation

A task may wait indefinitely because other work continually receives resources first.

### 12.24 Priority inversion

A high-priority task can wait for a resource held by low-priority work while medium-priority tasks consume the CPU.

### 12.25 Fairness

Per-tenant quotas and weighted scheduling can prevent one workload from monopolizing resources.

### 12.26 Work conservation

A scheduler should avoid idle capacity when runnable work exists, unless deliberate reservation/fairness policies require it.

### 12.27 Queue discipline

Possible policies:

```text
FIFO
LIFO
priority
deadline
tenant-aware
weighted fair
```

Each changes behavior.

### 12.28 Deadline-aware concurrency

A task with a near deadline may be prioritized over a long-running background task.

### 12.29 Bulkheads

Separate pools can protect critical workloads:

```text
customer API pool
internal maintenance pool
analytics pool
```

### 12.30 External job queues

When work exceeds the lifetime/capacity of one process:

```text
API
 ↓
durable queue
 ↓
workers
```

This creates stronger lifecycle and scaling boundaries.

### 12.31 Worker threads vs async I/O

Use async I/O when the bottleneck is waiting.

Use workers/processes when CPU work must execute in parallel or be isolated.

### 12.32 Browser workers

A browser worker can move CPU-heavy work away from the page's primary execution agent.

But:

```text
communication
data transfer
serialization
memory
```

still have costs.

### 12.33 Node worker threads

Worker threads provide separate JavaScript execution environments.

Shared memory or message passing can coordinate them.

### 12.34 Child processes

Processes provide stronger isolation:

```text
separate memory
separate failure domain
separate runtime
```

but with greater communication overhead.

### 12.35 External workers

A distributed queue provides:

```text
durability
horizontal scaling
retries
isolation
```

at the cost of operational complexity.

---

## 13. Edge Cases

### 13.1 Empty workload

```js
await Promise.all([]);
```

completes immediately according to Promise combinator semantics.

### 13.2 Limit zero

A bounded-concurrency scheduler with:

```text
limit = 0
```

must reject configuration or define a special paused state.

Do not let it deadlock silently.

### 13.3 Limit greater than workload

If:

```text
limit = 100
items = 5
```

only five tasks should run.

### 13.4 Worker throws synchronously

A scheduler must treat synchronous worker throws as failures.

### 13.5 Worker rejects asynchronously

A scheduler must release its slot after rejection.

### 13.6 Forgotten release

Semaphore misuse:

```js
const release = await semaphore.acquire();

await work();

// forgot release()
```

can permanently reduce capacity.

### 13.7 Double release

Releasing twice can corrupt the semaphore accounting.

### 13.8 Cancellation while waiting for a permit

A waiting task should stop waiting if its signal aborts.

### 13.9 Cancellation after acquiring permit

The active task must still release the permit.

### 13.10 Task completes during cancellation

Completion and cancellation can race.

The scheduler needs a single final state.

### 13.11 Retry creates more work

A retry mechanism can accidentally exceed concurrency limits if retries are not counted against capacity.

### 13.12 Queue growth

A bounded active pool with an unbounded waiting queue can still exhaust memory.

### 13.13 Large task payloads

Each queued task may retain large objects.

### 13.14 Slow task

One slow task can hold a slot for a long time.

### 13.15 Head-of-line blocking

FIFO ordering can delay short tasks behind long ones.

### 13.16 Starvation under priority

High-priority work can starve background work forever.

### 13.17 Rate/concurrency mismatch

A workload can respect concurrency but still violate a remote rate limit.

### 13.18 Retry storm

Multiple clients timing out together may all retry simultaneously.

### 13.19 Thundering herd

A large group of workers may wake at once and hit one dependency.

### 13.20 Worker startup cost

Creating a new worker per task may be much more expensive than maintaining a pool.

### 13.21 Oversubscription

Creating:

```text
100 CPU workers
```

on a machine with:

```text
4 cores
```

may reduce performance.

### 13.22 Shared memory race

Shared mutable memory between agents requires atomics/synchronization.

### 13.23 Data-transfer cost

Sending huge objects to workers may erase the benefit of parallelism.

### 13.24 Queue durability

An in-memory queue disappears when the process crashes.

### 13.25 Process restart duplication

A job may be executed again after a crash.

Exactly-once semantics require additional system design.

### 13.26 Database bottleneck

Increasing application concurrency cannot exceed the useful capacity of the database.

### 13.27 Connection pool exhaustion

Requests may wait on the pool even though the event loop is responsive.

### 13.28 Memory bandwidth saturation

CPU workers can compete for memory bandwidth before cores are fully saturated.

### 13.29 Garbage-collection pressure

High concurrent allocation rates can increase GC work.

### 13.30 Cancellation of irreversible work

A cancellation request cannot necessarily undo side effects already performed.

---

## 14. Common Misconceptions

### Misconception 1 — “Concurrency means multiple threads.”

No.

Concurrency can exist on one JavaScript execution agent.

### Misconception 2 — “Parallelism means async.”

No.

Parallel work can be synchronous within separate execution resources.

### Misconception 3 — “`Promise.all` controls concurrency.”

No.

It aggregates Promise outcomes.

### Misconception 4 — “More concurrency always means more throughput.”

No.

After saturation, throughput may flatten or fall.

### Misconception 5 — “A Promise represents a worker.”

No.

### Misconception 6 — “If the event loop is responsive, the system is not overloaded.”

Worker pools, databases, remote services, memory, and queues can be saturated independently.

### Misconception 7 — “Concurrency limit equals rate limit.”

No.

### Misconception 8 — “A concurrency limit of 100 is universally good.”

No.

It depends on:

```text
CPU
memory
latency
downstream capacity
work size
```

### Misconception 9 — “Parallel CPU work is always faster.”

Synchronization and transfer overhead can dominate.

### Misconception 10 — “Workers share all state automatically.”

No.

Workers have separate execution environments; sharing requires defined mechanisms.

### Misconception 11 — “Retries are independent from concurrency.”

No.

Retries consume capacity.

### Misconception 12 — “Cancellation automatically removes queued work.”

Only if the scheduler supports it.

### Misconception 13 — “Queueing is free.”

Queued work consumes memory and increases latency.

### Misconception 14 — “FIFO is always fair.”

FIFO can still disadvantage workloads with different task lengths or priorities.

### Misconception 15 — “If a database supports many connections, use as many as possible.”

More connections can increase contention and reduce useful throughput.

---

## 15. Common Mistakes

### Mistake 1 — Unbounded `Promise.all`

### Mistake 2 — One global concurrency limit for unrelated resources

### Mistake 3 — Missing semaphore release

### Mistake 4 — Double semaphore release

### Mistake 5 — No cancellation while queued

### Mistake 6 — Retrying without counting retries against capacity

### Mistake 7 — Ignoring downstream rate limits

### Mistake 8 — Ignoring queue memory

### Mistake 9 — Creating one worker per task

### Mistake 10 — Oversubscribing CPU workers

### Mistake 11 — Ignoring data-transfer costs to workers

### Mistake 12 — Using parallelism for tightly coupled sequential logic

### Mistake 13 — Sharing mutable state without synchronization

### Mistake 14 — No fairness across tenants

### Mistake 15 — No timeout on tasks

### Mistake 16 — No overload policy

### Mistake 17 — Assuming cancellation undoes side effects

### Mistake 18 — Measuring only throughput

### Mistake 19 — Tuning concurrency without measuring downstream saturation

### Mistake 20 — Detached child tasks with no supervisor

---

## 16. Comparison With Related Concepts

| Concept | What it controls | Typical use |
|---|---|---|
| Concurrency | Number of active/in-progress tasks | I/O fan-out, job processing |
| Parallelism | Simultaneous execution | CPU-heavy work |
| Async | Future completion coordination | Network, timers, I/O |
| Semaphore | Concurrent capacity | Protect finite resource |
| Queue | Waiting work | Admission and buffering |
| Pool | Reusable finite workers/resources | DB connections, workers |
| Rate limiter | Starts per time unit | API quotas |
| Backpressure | Producer rate relative to consumer | Streams |
| Retry | Re-execution after failure | Transient failures |
| Circuit breaker | Dependency admission | Protect failing dependency |
| Bulkhead | Capacity isolation | Separate workloads |
| Worker | Separate execution agent | CPU isolation |
| Child process | Separate process | Strong isolation |
| Job queue | Durable/distributed work | Long-running background work |
| Structured concurrency | Parent-owned child lifetime | Task groups |
| EventEmitter | Push notifications | Events |
| Observable | Multi-value reactive flow | Reactive systems |

### Concurrency vs parallelism

```text
concurrency:
A and B are both in progress

parallelism:
A and B are executing at the same time
```

### Concurrency vs rate limiting

```text
concurrency:
how many active

rate:
how many starts per time
```

### Concurrency vs backpressure

```text
concurrency:
capacity of active work

backpressure:
how production responds when consumption cannot keep up
```

### Semaphore vs queue

Semaphore:

```text
how many may run
```

Queue:

```text
what waits
```

Production schedulers often need both.

### Worker pool vs external queue

Worker pool:

```text
same process/runtime
```

External queue:

```text
durable/distributed boundary
```

### Structured concurrency vs detached work

Structured:

```text
parent owns child
```

Detached:

```text
child has another owner
```

---

## 17. Performance Considerations

### 17.1 Throughput curve

A typical system can behave like:

```text
concurrency
    ↑
throughput
    ┌────────────
   /
  /
 /
```

At some point:

```text
resource saturates
```

and additional concurrency adds little useful throughput.

### 17.2 Latency curve

As saturation increases:

```text
queueing ↑
latency ↑
```

This is why a system can have:

```text
same throughput
+
much worse latency
```

after increasing concurrency.

### 17.3 Little's Law

Use:

```text
L = λW
```

as a sanity check.

Example:

```text
100 ops/sec
250ms average latency

L ≈ 25 in-flight
```

If instrumentation reports:

```text
1000 active
```

there may be substantial queueing or long-lived work.

### 17.4 Optimal CPU parallelism

CPU-bound work often benefits from parallelism near the available execution capacity, but the optimum depends on:

- CPU architecture;
- memory bandwidth;
- task granularity;
- synchronization;
- runtime overhead.

### 17.5 I/O concurrency

I/O workloads may tolerate higher concurrency because much of the wall-clock time is waiting.

But downstream quotas can still be the limiting factor.

### 17.6 Context switching

Too many runnable workers can reduce CPU efficiency.

### 17.7 Cache locality

Parallel workers can compete for cache and memory bandwidth.

### 17.8 Serialization cost

Worker messaging and structured cloning can dominate small tasks.

### 17.9 Batching

Batching can reduce scheduling overhead:

```text
100 tiny tasks
→ 10 batches
```

### 17.10 Queue memory

At:

```text
1,000,000 queued tasks
```

even tiny metadata can consume significant memory.

### 17.11 Retry amplification

If failure probability increases with load:

```text
more concurrency
→ more timeouts
→ more retries
→ even more load
```

This can destabilize the system.

### 17.12 Tail latency

Average latency may look fine while P95/P99 grows dramatically.

Concurrency tuning should consider tail latency.

### 17.13 Fairness overhead

Tenant-aware scheduling is more complex but can protect critical workloads.

### 17.14 Adaptive limits

Adaptive concurrency can improve utilization if feedback signals are reliable.

But unstable control loops can oscillate:

```text
increase
→ overload
→ decrease
→ underutilize
→ increase
→ ...
```

### 17.15 Measurement

Monitor:

```text
active concurrency
queue depth
queue delay
throughput
P50/P95/P99 latency
error rate
timeouts
CPU
memory
downstream saturation
retry rate
```

---

## 18. Memory Considerations

### 18.1 Concurrency increases live state

More active tasks generally means more:

- Promises;
- closures;
- buffers;
- request objects;
- database results.

### 18.2 Queue memory

A queue of pending work can become the largest memory consumer.

### 18.3 Large task payloads

Avoid storing unnecessary full request bodies in queued tasks.

### 18.4 Worker memory

Each worker can have its own runtime/object memory.

### 18.5 Process memory

Child processes have separate memory footprints.

### 18.6 Shared memory

Shared buffers reduce copying in some architectures but introduce synchronization complexity.

### 18.7 GC pressure

More active allocations can increase garbage-collection frequency and pause/CPU costs.

### 18.8 Backpressure

Bounded buffering limits memory growth.

### 18.9 Resource retention

A queued closure can retain:

```text
request
credentials
database objects
buffers
```

for longer than expected.

### 18.10 Concurrency as a memory budget

A useful mental model:

```text
memory per task × active tasks
+
queued task memory
+
runtime overhead
≤ safe budget
```

---

## 19. Security Considerations

### 19.1 Resource-exhaustion attacks

Attackers can exploit unbounded concurrency to exhaust:

- CPU;
- memory;
- sockets;
- database connections;
- remote API quotas.

### 19.2 Tenant isolation

One tenant should not necessarily consume the whole concurrency budget.

### 19.3 Retry storms

Retries can amplify an attack or outage.

### 19.4 Queue poisoning

A malicious workload can fill queues with expensive tasks.

### 19.5 Priority abuse

If clients can influence priority, they may starve other traffic.

### 19.6 Worker isolation

Sensitive or untrusted CPU work may benefit from process isolation rather than sharing one runtime.

### 19.7 Shared memory races

Shared memory can create correctness/security vulnerabilities if synchronization is incorrect.

### 19.8 Deadlocks

Locks can create availability failures.

### 19.9 Stale authorization

Concurrent tasks may continue using permissions after authorization state changes.

### 19.10 Cancellation abuse

Attackers may repeatedly start and cancel expensive work, making setup/cleanup itself costly.

### 19.11 Amplification through fan-out

One external request can trigger:

```text
100 downstream requests
```

creating an amplification factor.

Bound fan-out.

---

## 20. Production Usage

### 20.1 HTTP API fan-out

Suppose one request needs:

```text
profile
permissions
recommendations
inventory
```

Use concurrency when dependencies are independent, but set a budget:

```text
max downstream fan-out = N
```

### 20.2 Database access

A database connection pool defines a physical resource boundary.

Application concurrency should account for:

```text
pool size
query latency
transaction duration
database CPU
lock contention
```

### 20.3 Batch processing

For:

```text
1 million records
```

use:

```text
bounded queue
+
bounded workers
+
checkpointing
```

instead of:

```js
Promise.all(allMillion)
```

### 20.4 API rate-limited integration

Combine:

```text
concurrency limit
+
rate limit
+
timeout
+
retry/backoff
+
circuit breaker
```

### 20.5 Worker-thread CPU pool

For expensive CPU tasks:

```text
main process
→ worker pool
→ results
```

Choose pool size based on measured CPU utilization and task behavior.

### 20.6 Browser application

For large client-side computations:

```text
UI agent
→ worker pool
```

Keep UI execution responsive.

### 20.7 Queue consumer

A production consumer often needs:

```text
max active jobs
max queue wait
visibility timeout
retry count
dead-letter policy
cancellation
shutdown
```

### 20.8 Stream processing

Combine:

```text
async iterator
+
bounded buffering
+
concurrency limit
+
backpressure
+
cancellation
```

### 20.9 Tenant-aware concurrency

Example:

```text
global = 100
tenant A = 20
tenant B = 20
tenant C = 20
```

This prevents one tenant from monopolizing global capacity.

### 20.10 Graceful shutdown

On shutdown:

```text
stop admission
→ stop producers
→ stop queue growth
→ drain or cancel active work
→ release resources
→ terminate workers
```

### 20.11 Adaptive concurrency

A mature system may dynamically change limits using:

```text
latency
error rate
resource saturation
```

This should be introduced only when fixed limits are demonstrably insufficient.

### 20.12 Bulkheads

Separate:

```text
interactive traffic
background traffic
maintenance traffic
```

so background work cannot consume all capacity.

### 20.13 Observability

Expose:

```text
active
queued
completed
failed
cancelled
timed out
retried
```

and:

```text
queue age
P95/P99 latency
resource saturation
```

### 20.14 SLO-aware concurrency

Concurrency should preserve:

```text
availability
latency SLO
error budget
```

rather than optimize throughput alone.

---

## 21. Implementation From Scratch

### Stage 1 — Guided semaphore

Implement:

```js
class Semaphore {
  constructor(limit) {
    this.limit = limit;
    this.active = 0;
    this.waiters = [];
  }

  async acquire() {
    // return release function
  }
}
```

Requirements:

- maximum active count;
- queue waiters;
- release capacity;
- reject invalid limits.

### Stage 2 — Partially Guided

Implement:

```js
async function mapLimited(items, limit, worker) {}
```

Requirements:

- preserve result ordering;
- bound active work;
- support synchronous worker throws;
- support rejected worker Promises.

### Stage 3 — No Reference

Build:

```js
class TaskPool {
  constructor({
    concurrency,
    signal
  }) {}

  submit(task) {}

  async close() {}

  stats() {}
}
```

Track:

```text
queued
active
completed
failed
cancelled
```

### Stage 4 — Edge-Case Hardened

Add:

- cancellation while queued;
- cancellation while running;
- timeout;
- retries;
- backoff;
- fairness;
- priority;
- queue limit;
- shutdown;
- task ownership.

### Stage 5 — Production Grade

Build:

```js
class AdaptiveConcurrencyController {
  constructor({
    min,
    max,
    initial,
    targetLatency,
    signal
  }) {}

  async run(task) {}

  recordSuccess(metrics) {}

  recordFailure(metrics) {}

  metrics() {}

  async shutdown() {}
}
```

Design an explicit control loop.

Do not let adaptation oscillate uncontrollably.

---

## 22. Debugging Exercises

### Exercise 1 — Unbounded fan-out

```js
await Promise.all(
  millionItems.map(processItem)
);
```

Find:

- memory risk;
- downstream overload;
- error aggregation issues;
- cancellation problems.

### Exercise 2 — Semaphore leak

```js
const release = await semaphore.acquire();

try {
  await work();
} catch (error) {
  throw error;
}
// release forgotten
```

What happens to future tasks?

### Exercise 3 — Double release

Call:

```js
release();
release();
```

What invariant does this violate?

### Exercise 4 — Queue growth

Create:

```text
producer = 10,000/sec
consumer = 2,000/sec
```

Measure memory growth.

### Exercise 5 — Retry amplification

Simulate:

```text
100 active
20% timeout
3 retries
```

Estimate how retry volume changes load.

### Exercise 6 — Head-of-line blocking

Create:

```text
one 10-second task
+
100 10-millisecond tasks
```

Compare FIFO and shortest-first/priority-style scheduling.

### Exercise 7 — Oversubscription

Compare CPU performance for:

```text
2 workers
4 workers
8 workers
16 workers
32 workers
```

on a controlled workload.

### Exercise 8 — Worker transfer cost

Measure:

```text
small payload
large payload
transferable payload
```

across worker boundaries.

### Exercise 9 — Race condition

Create a lost-update bug using two concurrent balance modifications.

Fix it using a proper transactional/locking strategy.

### Exercise 10 — Shutdown

Start 100 tasks, then cancel/shutdown.

Verify:

```text
queued tasks stop
active tasks clean up
resources release
final metrics accurate
```

---

## 23. Code Review Exercise

Review:

```js
async function processBatch(items) {
  return Promise.all(
    items.map(async item => {
      for (let attempt = 0; attempt < 5; attempt++) {
        try {
          return await callRemoteService(item);
        } catch (error) {
          await delay(100 * 2 ** attempt);
        }
      }

      throw new Error("failed");
    })
  );
}
```

Identify issues involving:

- unbounded concurrency;
- retry amplification;
- backoff synchronization;
- cancellation;
- timeout;
- queueing;
- rate limits;
- failure classification;
- error causes;
- observability;
- shutdown;
- partial progress.

Redesign it using:

```text
bounded concurrency
+
retry policy
+
jitter
+
timeout
+
cancellation
+
metrics
```

---

## 24. Interview Questions

### Foundational

1. What is concurrency?
2. What is parallelism?
3. Can concurrency exist on one JavaScript thread?
4. Is async the same as concurrency?
5. Is Promise.all a concurrency scheduler?
6. What is a semaphore?
7. What is a worker pool?
8. What is backpressure?
9. What is a queue?
10. Why should concurrency usually be bounded?

### Intermediate

11. Why can more concurrency reduce performance?
12. What is saturation?
13. What is queueing?
14. What is Little's Law?
15. How do you choose a concurrency limit?
16. What is the difference between rate limiting and concurrency limiting?
17. Why do retries affect concurrency?
18. What is head-of-line blocking?
19. What is fairness?
20. What is a bulkhead?

### Advanced

21. Compare event-loop concurrency and worker parallelism.
22. Explain structured concurrency.
23. Explain detached work.
24. Explain cancellation in a task pool.
25. Explain worker transfer costs.
26. Explain CPU oversubscription.
27. Explain shared-memory races.
28. Explain deadlock, livelock, and starvation.
29. Explain retry storms.
30. Explain adaptive concurrency.

### Principal-Level

31. Design a concurrency model for a high-throughput API.
32. Design per-tenant concurrency isolation.
33. Design an adaptive concurrency controller.
34. Design overload protection for a downstream dependency.
35. Design a retry policy that does not amplify outages.
36. Design a graceful shutdown for a large worker pool.
37. Decide between async I/O, worker threads, processes, and external queues.
38. Diagnose a service whose throughput stayed flat while concurrency doubled and P99 latency tripled.
39. Design fair scheduling across workloads with different task sizes.
40. Define the concurrency budget for a production service and defend every constraint.

---

## 25. Predict-the-Output Exercises

### Exercise A

```js
const a = Promise.resolve("A");
const b = Promise.resolve("B");

console.log("start");

await Promise.all([a, b]);

console.log("done");
```

Inside an async context, reason about the suspension point even though both inputs are already fulfilled.

### Exercise B

```js
const tasks = [1, 2, 3];

for (const task of tasks) {
  await work(task);
}

console.log("done");
```

What is the maximum concurrency?

### Exercise C

```js
const tasks = [1, 2, 3];

await Promise.all(
  tasks.map(task => work(task))
);

console.log("done");
```

What is the maximum intended concurrency?

### Exercise D

```js
async function worker(id) {
  console.log("start", id);
  await delay(id === 1 ? 30 : 0);
  console.log("end", id);
}

await Promise.all([
  worker(1),
  worker(2)
]);
```

Predict the execution/completion ordering.

### Exercise E

```js
async function limited(items) {
  const semaphore = new Semaphore(2);

  return Promise.all(
    items.map(async item => {
      const release = await semaphore.acquire();

      try {
        return await process(item);
      } finally {
        release();
      }
    })
  );
}
```

Why can no more than two `process()` operations be active at once?

### Exercise F

```js
let active = 0;
let maximum = 0;

async function tracked() {
  active++;
  maximum = Math.max(maximum, active);

  await delay(10);

  active--;
}

await Promise.all(
  Array.from({ length: 100 }, tracked)
);

console.log(maximum);
```

What does this measure, and what is the expected maximum for this unbounded fan-out?

---

## 26. Mastery Exercises

### Exercise 1 — Semaphore

Implement:

```js
Semaphore(limit)
```

with:

- FIFO waiters;
- cancellation;
- timeout;
- safe release;
- metrics.

### Exercise 2 — Bounded map

Implement:

```js
mapLimited(items, limit, worker, options)
```

Support:

- ordered results;
- cancellation;
- timeout;
- retries;
- queue bound.

### Exercise 3 — Multi-resource scheduler

Build:

```text
CPU limit = 4
database limit = 10
API limit = 20
```

A task may require multiple resources.

Ensure resource acquisition does not create deadlocks.

### Exercise 4 — Tenant isolation

Build:

```text
global limit = 100
per tenant = 10
```

with fair scheduling.

### Exercise 5 — Rate + concurrency control

Implement both:

```text
max active = 20
max starts = 100/sec
```

### Exercise 6 — Retry-aware scheduler

Ensure retries:

- return to the scheduling queue;
- honor concurrency;
- honor rate limits;
- respect cancellation;
- apply backoff/jitter.

### Exercise 7 — CPU worker pool

Build a worker-thread/process pool for CPU-heavy tasks.

Compare:

```text
main-thread
worker
process
```

on:

- throughput;
- latency;
- memory;
- startup;
- communication cost.

### Exercise 8 — Adaptive concurrency

Implement a controller that modifies:

```text
concurrency limit
```

based on:

```text
latency target
error rate
resource saturation
```

Add hysteresis to reduce oscillation.

### Exercise 9 — Graceful shutdown

Build:

```js
pool.shutdown({
  mode: "drain"
});
```

and:

```js
pool.shutdown({
  mode: "cancel"
});
```

Define exact semantics for:

```text
queued
running
retrying
waiting
```

### Exercise 10 — Principal concurrency model

Design concurrency for:

```text
10,000 incoming requests/sec
database pool = 100
remote API quota = 500 req/sec
CPU budget = 8 cores
P99 target = 250ms
```

Define:

- admission;
- queue size;
- concurrency;
- rate;
- timeout;
- retry;
- bulkheads;
- cancellation;
- fairness;
- shutdown;
- observability.

Defend every decision.

---

## 27. Key Takeaways

1. Concurrency means multiple activities are in progress.
2. Parallelism means multiple activities execute simultaneously on independent resources.
3. Concurrency does not require multiple JavaScript threads.
4. Async programming is not identical to concurrency.
5. `Promise.all()` aggregates asynchronous outcomes but is not a resource-aware scheduler.
6. Concurrency should normally be bounded.
7. The correct concurrency limit depends on the bottleneck.
8. CPU, I/O, memory, database, network, and external API workloads have different capacities.
9. More concurrency can improve throughput until saturation.
10. Beyond saturation, queueing and latency can increase dramatically.
11. Little's Law provides a useful relationship among throughput, latency, and in-flight work.
12. Concurrency and rate limiting solve different problems.
13. Backpressure controls producer behavior when consumers cannot keep up.
14. Semaphores control active capacity.
15. Queues hold work that cannot currently execute.
16. Pools provide finite reusable capacity.
17. Structured concurrency aligns child lifetime, cancellation, failure, and ownership with a parent scope.
18. Detached work requires another explicit owner and supervisor.
19. Concurrency introduces race conditions and shared-state hazards.
20. Retries consume concurrency and can amplify load.
21. Timeouts protect capacity from stuck work.
22. Cancellation should remove unnecessary queued and active work where safe.
23. Fairness prevents one workload from monopolizing shared capacity.
24. Bulkheads isolate critical workloads.
25. Worker threads/processes can provide parallelism for CPU-heavy work.
26. Worker communication and startup costs must be included in the performance model.
27. Oversubscription can reduce CPU efficiency.
28. Durable external queues provide stronger scaling and lifecycle boundaries than in-memory pools.
29. Production concurrency should be measured using active work, queue depth, latency, throughput, errors, and resource saturation.
30. The central principle is:

> Concurrency is not about doing as much work as possible at once; it is about admitting the right amount of work into finite resources while preserving correctness, latency, fairness, and recoverability.

---

## 28. Concept Connections

### Depends On

- Chapter 12 — Execution Contexts / Execution Model
- Chapter 29 — Errors / Error Handling
- Chapter 30 — Resource Management / Cleanup
- Chapter 31 — Asynchronous JavaScript Fundamentals
- Chapter 32 — ECMAScript Jobs / Promise Reactions
- Chapter 33 — Browser Event Loop
- Chapter 34 — Node Event Loop / libuv
- Chapter 35 — Promises
- Chapter 36 — Async/Await
- Chapter 37 — Cancellation / Abort
- Chapter 38 — Async Iteration / Streaming

### Builds Toward

- Chapter 40 — Observables / Reactive
- Chapter 45 — Memory / GC
- Chapter 46 — Weak Refs / Finalization
- Chapter 52 — Web Workers / Concurrency
- Chapter 53 — Web Streams / Data Flow
- Chapter 55 — Fetch / HTTP Networking
- Chapter 58 — Node Architecture
- Chapter 60 — Node Streams
- Chapter 61 — Worker Threads / Child Processes / Cluster
- Chapter 62 — Process Lifecycle
- Chapter 63 — Async Context / Diagnostics
- Chapter 78 — Production JS Architecture
- Chapter 79 — API Design
- Chapter 82 — API Architecture
- Chapter 83 — Observability
- Chapter 84 — Reliability
- Chapter 85 — Performance
- Chapter 86 — Testing
- Chapter 87 — Deterministic Async Testing
- Chapter 88 — Debugging Methodology
- Chapter 89 — Code Review / Refactoring
- Chapter 98 — Anti-patterns / Failure Modes
- Chapter 100 — Cost Model / Tradeoffs
- Chapter 101 — Real-world Production Scenarios
- Chapter 104 — Production HTTP Client
- Chapter 105 — Node REST API
- Chapter 106 — Real-time WebSocket
- Chapter 107 — Job Queue
- Chapter 110 — Production JS Backend
- Chapter 111 — Large-scale JS Platform
- Chapter 121 — System Design
- Chapter 122 — Final Principal JS Project

### Related Concepts

- Concurrency
- Parallelism
- Scheduling
- Semaphore
- Queue
- Pool
- Executor
- Worker
- Worker thread
- Child process
- Rate limiting
- Backpressure
- Admission control
- Bulkhead
- Circuit breaker
- Retry
- Backoff
- Jitter
- Fairness
- Starvation
- Deadlock
- Livelock
- Head-of-line blocking
- Oversubscription
- Structured concurrency
- Task supervision
- Little's Law
- Saturation
- Queueing
- Tail latency

### Concepts Revisited

This chapter revisits:

- promises;
- async/await;
- Jobs;
- event loops;
- cancellation;
- resource management;
- async iteration;
- backpressure;
- error handling.

### Why This Chapter Matters Later

Concurrency is the point where asynchronous programming becomes systems engineering.

The code:

```js
Promise.all(tasks)
```

is easy.

The difficult questions are:

```text
How many tasks?
Why this number?
What resource limits it?
What if tasks stall?
What if they fail?
What if they retry?
What if the user cancels?
What if the dependency is overloaded?
What if one tenant sends 100× more traffic?
What happens during shutdown?
```

Those are production architecture questions.

This chapter therefore transforms the async curriculum from:

```text
“How does asynchronous code work?”
```

into:

```text
“How should a system admit, execute, coordinate, and terminate concurrent work?”
```

The central principle is:

> Concurrency is a resource-allocation problem disguised as a programming convenience.

---

## 29. Completion Criteria

Mark Chapter 39 `[+] Completed` only after the learner can demonstrate all of the following.

### Conceptual Understanding

- [ ] Define concurrency.
- [ ] Define parallelism.
- [ ] Explain concurrency without threads.
- [ ] Explain async vs concurrency.
- [ ] Explain bounded concurrency.
- [ ] Explain semaphore.
- [ ] Explain queue.
- [ ] Explain pool.
- [ ] Explain rate limiting.
- [ ] Explain backpressure.
- [ ] Explain saturation.
- [ ] Explain Little's Law.
- [ ] Explain structured concurrency.
- [ ] Explain bulkheads and admission control.

### Predictive Mastery

- [ ] Predict sequential concurrency.
- [ ] Predict unbounded fan-out.
- [ ] Predict bounded-pool execution.
- [ ] Predict completion vs input order.
- [ ] Predict queue growth.
- [ ] Predict retry amplification.
- [ ] Predict semaphore leaks.
- [ ] Predict cancellation behavior.
- [ ] Predict CPU oversubscription effects.

### Implementation

- [ ] Implement a semaphore.
- [ ] Implement bounded map.
- [ ] Implement async task pool.
- [ ] Implement cancellation-aware queueing.
- [ ] Implement retry-aware scheduling.
- [ ] Implement rate + concurrency control.
- [ ] Implement tenant-aware fairness.
- [ ] Implement worker pool.
- [ ] Implement graceful shutdown.
- [ ] Implement adaptive concurrency.

### Debugging

- [ ] Diagnose unbounded concurrency.
- [ ] Diagnose queue growth.
- [ ] Diagnose semaphore leaks.
- [ ] Diagnose head-of-line blocking.
- [ ] Diagnose starvation.
- [ ] Diagnose retry storms.
- [ ] Diagnose downstream saturation.
- [ ] Diagnose race conditions.
- [ ] Diagnose oversubscription.
- [ ] Diagnose worker-transfer overhead.

### Production Engineering

- [ ] Define concurrency budgets.
- [ ] Define queue capacity.
- [ ] Define rate limits.
- [ ] Define timeout.
- [ ] Define retry.
- [ ] Define cancellation.
- [ ] Define fairness.
- [ ] Define overload behavior.
- [ ] Define bulkheads.
- [ ] Define shutdown.
- [ ] Define observability.
- [ ] Define worker/process architecture.

### Interview Readiness

- [ ] Explain concurrency vs parallelism.
- [ ] Explain bounded concurrency.
- [ ] Explain saturation.
- [ ] Use Little's Law in reasoning.
- [ ] Design a semaphore/pool.
- [ ] Design retry + concurrency safely.
- [ ] Design tenant isolation.
- [ ] Choose async I/O vs workers vs processes vs queues.
- [ ] Diagnose throughput/latency trade-offs.
- [ ] Defend a production concurrency budget.

### Track A — Core Theory

- [ ] Understand concurrency and parallelism.
- [ ] Understand queueing and saturation.
- [ ] Understand resource capacity.
- [ ] Understand structured concurrency.
- [ ] Understand admission control.
- [ ] Understand fairness and failure amplification.

### Track B — Implementation

- [ ] Guided implementation completed.
- [ ] Partially guided implementation completed.
- [ ] No-reference implementation completed.
- [ ] Edge-case hardened implementation completed.
- [ ] Production-oriented concurrency controller reviewed.

### Track C — Interview / Reasoning

- [ ] Completed output prediction.
- [ ] Completed concurrency debugging.
- [ ] Completed code review.
- [ ] Completed concurrency budgeting.
- [ ] Completed overload-protection design.
- [ ] Completed worker architecture trade-off analysis.

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

# Chapter 39 — Revision / Retrieval Record

### Retrieval Prompts

1. What is concurrency?
2. What is parallelism?
3. Can concurrency exist on one JavaScript execution agent?
4. Why is async not the same as concurrency?
5. Why is `Promise.all` not a concurrency scheduler?
6. What is a semaphore?
7. What is a bounded worker pool?
8. What is saturation?
9. What is Little's Law?
10. How do throughput, latency, and in-flight work relate?
11. How is concurrency limiting different from rate limiting?
12. What is backpressure?
13. Why can more concurrency increase latency?
14. What is head-of-line blocking?
15. What is starvation?
16. What is deadlock?
17. What is livelock?
18. What is oversubscription?
19. Why do retries affect concurrency?
20. What is structured concurrency?
21. Why does cancellation matter in a task queue?
22. Why does queue memory matter?
23. How would you select a database concurrency limit?
24. How would you select a remote API concurrency limit?
25. When should CPU work use workers or processes?
26. When should work move to an external queue?
27. How would you protect one tenant from monopolizing capacity?
28. How would you design graceful shutdown?
29. How would you detect retry storms?
30. How would you tune concurrency from production data?

### Weak Areas

```text
-
-
-
```

### Revision Queue

```text
- [ ] Revisit concurrency vs parallelism
- [ ] Revisit bounded concurrency
- [ ] Revisit semaphores and pools
- [ ] Revisit Little's Law
- [ ] Revisit rate vs concurrency
- [ ] Revisit backpressure
- [ ] Revisit fairness
- [ ] Revisit retry amplification
- [ ] Revisit structured concurrency
- [ ] Revisit workers/processes/queues
- [ ] Revisit graceful shutdown
- [ ] Revisit adaptive concurrency
```

### Assessment History

```text
Date:
Score:
Weak Areas:
Next Review:
```

### Chapter Status

```text
[+] Expanded
[ ] Reviewed
[ ] Practiced
[ ] Assessed
[ ] Mastered
```

---

# Chapter 39 — Canonical References and Source Discipline

Use this source hierarchy:

1. ECMAScript specification — Jobs, Promises, async functions, agents, SharedArrayBuffer/Atomics, and language-level asynchronous semantics.
2. WHATWG / browser standards — Web Workers, scheduling, streams, and browser-specific concurrency mechanisms.
3. Node.js official documentation — worker threads, child processes, streams, timers, diagnostics, and runtime concurrency behavior.
4. libuv documentation — event-loop and worker-pool implementation details.
5. Operating-system/runtime documentation — process/thread scheduling and platform concurrency mechanisms.
6. Queueing/performance literature — Little's Law, saturation, latency/throughput relationships, and scheduling theory.
7. Application architecture documentation — concurrency budgets, admission control, rate limiting, fairness, retries, bulkheads, circuit breakers, and shutdown.

Always distinguish:

```text
ECMAScript language semantics
vs
host/runtime concurrency
vs
OS-level parallelism
vs
application scheduling policy
```

Do not claim that JavaScript Promises provide a universal concurrency limit.

Do not claim that `Promise.all()` creates a worker pool.

Do not claim that parallelism automatically improves performance.

Do not treat a single concurrency number as portable across CPU, database, filesystem, network, and external API workloads.

---

# Chapter 39 — Completion Snapshot

```text
Chapter: 39
Title: Concurrency and Parallelism
Part: VI — Async
Status: [+] Expanded
Track A: [ ] Core Theory
Track B: [ ] Implementation
Track C: [ ] Interview / Reasoning
Mastery: [ ] Not Mastered
```