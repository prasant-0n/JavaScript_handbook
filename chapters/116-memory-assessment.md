# Chapter 116 — Memory Assessment — 10 Questions

> **JavaScript Mastery — Part XXI: Assessment**
>
> **Assessment:** 10 progressive memory problems focused on object reachability, garbage collection, closures, caches, listeners, WeakMap/WeakRef, retained state, Node.js/browser memory diagnosis, and production memory judgment.
>
> **Role perspective:** Principal JavaScript Engineer · ECMAScript Specialist · Runtime Engineer · Browser Engineer · Node.js Architect · Performance Engineer · Debugging Specialist
>
> **Status:** `[ ] Not Started`
>
> **Core rule:** **Garbage collection answers “can this object still be reached?” not “does the application still want it?”**

---

# 1. Assessment Mission

This assessment tests whether you can reason about memory from first principles.

The central model is:

```text
allocation
→ references created
→ object graph changes
→ reachability changes
→ garbage-collection eligibility
→ collection
```

Memory bugs are often caused by:

```text
unbounded growth
unexpected references
long-lived owners
listener retention
cache retention
closure capture
resource retention
```

Do not equate:

```text
high memory
```

with:

```text
memory leak
```

---

# 2. Scope Discipline

Separate:

```text
ECMAScript object reachability
JavaScript engine GC behavior
browser host memory
Node.js process/native memory
application resource lifetime
```

The language model can tell you about object references and semantics.

A specific engine may decide:

```text
when GC occurs
how generations work
what collector is used
how much memory is reserved
```

Do not turn an implementation strategy into a universal language guarantee.

---

# 3. Assessment Scoring

Each question:

```text
0–10 points
```

Total:

```text
100 points
```

Recommended interpretation:

```text
90–100 → Principal-level memory reasoning
80–89  → Strong memory reasoning
70–79  → Good foundation with targeted gaps
60–69  → Significant memory-model gaps
<60     → Rebuild reachability and memory fundamentals
```

---

# 4. Full-Credit Standard

A strong answer includes:

```text
1. object/reference graph
2. lifetime owner
3. reachability analysis
4. expected GC eligibility
5. root cause
6. minimal fix
7. regression protection
8. production diagnostic strategy
```

For advanced questions also include:

```text
retention path
allocation pattern
boundedness
resource lifecycle
engine-specific uncertainty
```

---

# 5. Question 1 — Reachability Basics

```js
let user = {
  name: "A",
  profile: {
    city: "B"
  }
};

let alias = user;

user = null;
```

### Task

After the final statement:

```text
Is the original object eligible for garbage collection?
Is the nested profile object eligible?
```

Draw the reference graph before and after:

```text
user = null
```

Then explain why:

```text
alias
```

changes the answer.

---

# 6. Question 2 — Closure Retention

```js
function createHandler() {
  const data = new Array(5_000_000).fill({
    active: true
  });

  return function handler() {
    return data.length;
  };
}

let handler = createHandler();

handler = null;
```

### Task

Analyze whether `data` remains reachable after:

```js
handler = null;
```

Explain:

```text
closure
environment
returned function
root references
GC eligibility
```

Then answer:

```text
What changes if some other long-lived object still stores handler?
```

---

# 7. Question 3 — Unbounded Cache

```js
const cache = new Map();

function remember(key, value) {
  cache.set(key, value);
}

for (let i = 0; i < 1_000_000; i++) {
  remember(`user:${i}`, {
    id: i,
    data: new Array(100).fill(i)
  });
}
```

### Symptom

Memory usage continuously grows.

### Task

Diagnose whether this should be called:

```text
garbage-collector failure
```

or:

```text
application retention
```

Explain the retention path.

Design at least three bounded-cache strategies:

```text
TTL
LRU
maximum entries
```

For each state the trade-off.

---

# 8. Question 4 — Event Listener Leak

Browser-oriented example:

```js
const listeners = [];

function mount() {
  const largeState = {
    rows: new Array(500_000).fill("row")
  };

  const button = document.querySelector("#save");

  const handler = () => {
    console.log(largeState.rows.length);
  };

  button.addEventListener("click", handler);

  listeners.push(handler);
}

function unmount() {
  listeners.pop();
}
```

### Symptom

The UI is “unmounted,” but memory does not decrease as expected.

### Task

Identify every reference that may matter.

Explain why:

```js
listeners.pop();
```

does not necessarily remove the browser's event listener.

Provide a correct lifecycle design.

Discuss:

```text
removeEventListener
AbortController-based listener cleanup
component lifecycle ownership
```

---

# 9. Question 5 — WeakMap vs Map

Compare:

```js
const metadata = new Map();

function attach(object, data) {
  metadata.set(object, data);
}
```

with:

```js
const metadata = new WeakMap();

function attach(object, data) {
  metadata.set(object, data);
}
```

### Task

Explain the intended difference in object lifetime.

Answer:

```text
What can keep an object reachable?
What does WeakMap weakly reference?
Why can't WeakMap be iterated like Map?
```

Then explain when using WeakMap is appropriate and when it is the wrong abstraction.

Do not claim:

```text
"WeakMap guarantees immediate deletion."
```

---

# 10. Question 6 — FinalizationRegistry Misuse

```js
const registry = new FinalizationRegistry(id => {
  console.log("cleanup", id);
});

function track(object, id) {
  registry.register(object, id);
}
```

A developer concludes:

> “Once an object becomes unreachable, the callback will run immediately, so I can use FinalizationRegistry for critical cleanup.”

### Task

Reject or defend the conclusion.

Explain:

```text
reachability
GC
finalization scheduling
nondeterminism
program correctness
```

Then classify appropriate use cases and inappropriate use cases.

---

# 11. Question 7 — Node.js Process Memory Diagnosis

A production Node.js service reports:

```text
RSS: steadily increasing
heapUsed: mostly stable
heapTotal: mostly stable
```

### Task

Do not immediately conclude:

```text
JavaScript heap leak
```

Design an investigation that distinguishes:

```text
JavaScript heap
external memory
ArrayBuffer / Buffer usage
native allocations
libuv/runtime resources
OS/process memory
```

Describe the measurements and evidence you would collect.

Also explain why:

```text
heapUsed
```

alone is insufficient to describe total process memory.

---

# 12. Question 8 — Array Buffer Retention

```js
let buffer = new ArrayBuffer(50 * 1024 * 1024);

function makeView() {
  return new Uint8Array(buffer);
}

const view = makeView();

buffer = null;
```

### Task

Determine whether the underlying memory is necessarily eligible for reclamation.

Explain:

```text
ArrayBuffer
TypedArray view
backing storage
references
```

Then discuss what changes when:

```js
view = null;
```

is also executed.

Be precise about what is guaranteed by the language model versus what a particular engine/runtime may do with physical memory.

---

# 13. Question 9 — Retention Path Through a Global Registry

```js
const sessions = new Map();

function createSession(id) {
  const session = {
    id,
    data: new Array(100_000).fill(id)
  };

  sessions.set(id, session);

  return session;
}

function closeSession(id) {
  const session = sessions.get(id);

  if (!session) {
    return;
  }

  session.closed = true;
}
```

### Symptom

Sessions are “closed,” but memory keeps increasing.

### Task

Find the retention bug.

Explain:

```text
closed state
vs
object lifetime
```

Then decide whether the correct fix is:

```text
delete from Map
WeakMap
TTL
LRU
external storage
```

You must justify the choice from ownership requirements.

---

# 14. Question 10 — Principal-Level Memory Incident

A Node.js service has this pattern:

```js
const cache = new Map();
const subscribers = new Map();

function load(key) {
  if (cache.has(key)) {
    return cache.get(key);
  }

  const value = createLargeValue(key);

  cache.set(key, value);

  return value;
}

function subscribe(key, handler) {
  if (!subscribers.has(key)) {
    subscribers.set(key, new Set());
  }

  subscribers.get(key).add(handler);
}

function unsubscribe(key, handler) {
  subscribers.get(key)?.delete(handler);
}
```

Production symptoms:

```text
1. heap usage increases over days
2. old tenant data remains present
3. subscriber counts increase over time
4. unsubscribe appears to succeed
5. RSS is higher than heapUsed
6. restarting the process fixes the issue temporarily
7. throughput slowly degrades
```

### Task

Treat this as a production memory incident.

Identify at least six hypotheses.

For each provide:

```text
Hypothesis
Evidence
Retention path
How to test
Likely root cause
Fix
Trade-off
Regression protection
```

Your investigation must consider:

```text
unbounded cache
large-value retention
subscriber ownership
function identity
tenant isolation
stale closures
external/native memory
fragmentation
long-lived process roots
```

Do not assume that all seven symptoms share one cause.

---

# 15. Memory Graph Worksheet

For difficult cases draw:

```text
GC Root
  |
  v
Long-lived owner
  |
  v
Collection / closure / listener
  |
  v
Object
  |
  v
Large retained data
```

Then ask:

```text
Who owns this reference?

Why does that owner live this long?

When should ownership end?

What code removes the reference?

What happens if cleanup never runs?
```

---

# 16. Retention Path vs Allocation Site

Do not confuse:

```text
where memory was allocated
```

with:

```text
why memory remains reachable
```

Example:

```text
allocation:
createLargeValue()

retention:
global Map → session → data
```

The allocation site tells you:

```text
where memory entered the graph
```

The retention path tells you:

```text
why it cannot leave
```

Both matter.

---

# 17. Leak Classification

Classify memory growth as one or more:

```text
A — True object-retention leak
B — Unbounded cache
C — Accidental global/root retention
D — Listener/subscriber leak
E — Closure retention
F — Promise/task retention
G — Resource leak
H — External/native memory growth
I — Fragmentation/reservation behavior
J — Legitimate workload growth
K — Temporary allocation pressure
L — Measurement artifact
```

---

# 18. Memory Debugging Workflow

Use:

```text
1. Reproduce or identify growth pattern.
2. Measure baseline.
3. Measure after representative workload.
4. Determine which memory region grows.
5. Compare heap snapshots where applicable.
6. Find retaining paths.
7. Identify long-lived owners.
8. Verify cleanup lifecycle.
9. Patch ownership.
10. Add regression test/monitoring.
11. Re-measure under equivalent load.
```

Do not conclude:

```text
"GC is broken"
```

before inspecting reachability.

---

# 19. Heap Snapshot Reasoning

When comparing heap snapshots inspect:

```text
retained size
shallow size
retainers
dominator/retaining paths
object counts
growth between snapshots
```

Ask:

```text
What object grew?

Who retains it?

Why is that owner still alive?

Is the growth expected?

Can the owner be bounded?
```

---

# 20. Memory Budgeting

For production systems define:

```text
steady-state memory
peak memory
per-request memory
per-tenant memory
cache budget
queue budget
buffer budget
concurrency budget
```

Then ask:

```text
What happens when the budget is exceeded?
```

Possible policies:

```text
evict
reject
shed load
backpressure
degrade
spill to external storage
restart under controlled policy
```

A system without memory bounds is difficult to make reliable.

---

# 21. Cache Design Review

Every in-memory cache should answer:

```text
1. What is cached?
2. Who owns it?
3. Maximum size?
4. Expiration policy?
5. Eviction policy?
6. Staleness policy?
7. Invalidation mechanism?
8. Tenant isolation?
9. Serialization cost?
10. Memory budget?
11. What happens under outage?
```

Do not introduce a cache merely because:

```text
"it is faster."
```

A cache changes memory, correctness, and failure behavior.

---

# 22. Listener and Subscription Lifecycle

For event-driven systems define:

```text
subscribe
→ active
→ unsubscribe
→ no remaining references
```

The lifecycle must be explicit.

Ask:

```text
Who owns the subscription?

Who performs cleanup?

What happens on exception?

What happens on cancellation?

What happens when the object is destroyed?
```

---

# 23. Closure Review

Closures are not inherently leaks.

Review:

```text
What is captured?

How large is it?

How long can the closure live?

Who owns the closure?

Can captured state be narrowed?
```

Prefer:

```js
const id = largeObject.id;

return () => use(id);
```

over capturing an unnecessarily large object when the callback only needs one small field.

The principle is:

```text
capture the smallest necessary state
```

---

# 24. Weak References Decision Rule

Use weak references only when the semantics truly are:

```text
association should not extend object lifetime
```

Do not use WeakMap/WeakRef to hide:

```text
missing lifecycle ownership
unbounded data
incorrect cleanup
```

Weak references are not a substitute for explicit resource management.

---

# 25. Resource Lifetime vs Object Lifetime

Distinguish:

```text
JavaScript object becomes unreachable
```

from:

```text
external resource is properly released
```

Examples:

```text
socket
file descriptor
database connection
subscription
worker
timer
native allocation
OS handle
```

For critical resources prefer deterministic lifecycle APIs and explicit cleanup rather than waiting for garbage collection.

---

# 26. Memory and Performance Connection

Memory behavior can affect:

```text
allocation rate
GC pressure
CPU consumption
latency
cache locality
serialization cost
process RSS
```

But avoid saying:

```text
"fewer allocations are always faster."
```

Measure the real workload.

---

# 27. Production Memory Runbook

When memory rises:

```text
1. Is the increase real?
2. Which memory metric rises?
3. Is heapUsed rising?
4. Is RSS rising independently?
5. Does memory return after workload falls?
6. Are object counts growing?
7. Are retaining paths stable?
8. Is cache size bounded?
9. Are listeners/subscribers cleaned?
10. Are buffers/external allocations involved?
11. Is workload itself increasing?
12. Did a deployment change ownership/lifetime?
```

---

# 28. Principal Judgment Exercise

Rank these fixes for an unbounded cache:

```text
A. increase container memory
B. call GC more often
C. delete random keys
D. enforce maximum size
E. add TTL only
F. use LRU + size budget
G. move cache to external store
```

Your ranking must depend on:

```text
correctness
workload
staleness tolerance
memory budget
latency requirements
operational complexity
failure modes
```

There is no universal winner.

---

# 29. Regression Protection

For a memory bug create measurable protection.

Examples:

```text
cache cardinality assertion
listener lifecycle test
heap-growth benchmark
load test
snapshot comparison
object-count monitoring
RSS alert
heapUsed alert
retention-path investigation
subscription count metric
```

The exact protection must match the actual failure.

---

# 30. Memory Assessment Rubric

### 0–2 — Guessing

Talks about memory without identifying references.

### 3–4 — Basic

Recognizes obvious retention but misses ownership.

### 5–6 — Strong

Can explain reachability and lifecycle.

### 7–8 — Advanced

Identifies retaining paths and measurement strategy.

### 9 — Production

Designs bounded ownership and diagnostics.

### 10 — Principal

Connects memory semantics, runtime behavior, application architecture, operational limits, and regression protection.

---

# 31. Retrieval Record

```md
# Chapter 116 — Memory Assessment — Retrieval Record

## Attempt
- Date:
- Duration:
- Score:
- Percentage:
- Status before:
- Status after:

## Question Scores
- Q1:
- Q2:
- Q3:
- Q4:
- Q5:
- Q6:
- Q7:
- Q8:
- Q9:
- Q10:

## Strongest Areas
-

## Weakest Areas
-

## Reachability Errors
-

## Cache / Ownership Errors
-

## GC Model Errors
-

## External Memory Errors
-

## Diagnostic Gaps
-

## Questions Requiring Rework
-

## Next Review
-
```

---

# 32. Spaced Retrieval Schedule

### Day 0

Complete all 10.

### Day 1

Redo every question where:

```text
confidence < 4
```

### Day 3

Redraw the object/reference graphs for:

```text
Q1–Q5
```

### Day 7

Redo:

```text
Q6–Q8
```

without notes.

### Day 14

Rework:

```text
Q9–Q10
```

as production incidents.

### Day 21

Design one bounded cache from memory.

### Day 30

Take a new application and identify its five largest potential retention risks.

---

# 33. Dependency Graph

```text
Chapters 01–30
        ↓
values + objects + functions + closures
        ↓
Chapters 31–44
        ↓
async execution + agents + isolation
        ↓
Chapters 45–48
        ↓
memory + GC + engine architecture
        ↓
Chapters 49–70
        ↓
browser + Node + streams + workers + tooling
        ↓
Chapters 71–101
        ↓
algorithms + production architecture
        ↓
Chapters 102–111
        ↓
projects
        ↓
Chapter 112 — Conceptual Assessment
        ↓
Chapter 113 — Output Prediction
        ↓
Chapter 114 — Debugging Assessment
        ↓
Chapter 115 — Async / Event Loop Assessment
        ↓
Chapter 116 — Memory Assessment
        ↓
Chapter 117 — Performance Assessment
```

---

# 34. Concept Connections

## Depends On

```text
objects
references
closures
WeakMap
WeakRef
FinalizationRegistry
async tasks
event listeners
Node process memory
engine/GC concepts
```

## Builds Toward

```text
performance analysis
reliability
capacity planning
production architecture
principal incident response
```

## Concepts Revisited

```text
closure
object identity
Map
WeakMap
event listener lifecycle
async operations
resource ownership
cache
```

## Why This Chapter Matters

Memory discipline is ownership discipline.

The key question is:

```text
Who keeps this object alive, and why?
```

Once you can answer that precisely, many memory problems become graph problems instead of mysterious runtime behavior.

---

# 35. Track A — Core Theory

Master:

```text
reachability
roots
object graphs
closures
GC eligibility
weak references
finalization
ArrayBuffer/backing storage
heap vs external memory
resource lifetime
```

Deliverable:

```text
draw and explain retention paths
```

---

# 36. Track B — Implementation

Build:

```text
bounded LRU cache
TTL cache
subscription registry
cleanup-aware event abstraction
memory stress test
heap-growth benchmark
```

Each implementation must define:

```text
ownership
bound
cleanup
failure behavior
observability
```

---

# 37. Track C — Interview / Reasoning

Practice answering:

```text
"Is this a memory leak?"

"Why isn't GC reclaiming this?"

"Who retains the object?"

"Why is WeakMap useful here?"

"Why is RSS high while heapUsed is stable?"

"How would you prove a memory leak in production?"
```

Deliverable:

```text
reason from references and measurements
```

---

# 38. Memory Mastery Gate

You may mark:

```text
[+] Completed
```

when:

```text
[ ] all 10 questions attempted
[ ] object graphs drawn
[ ] retention paths identified
[ ] bounded-cache strategies compared
[ ] weak-reference semantics understood
[ ] production diagnosis designed
```

Mark:

```text
[*] Mastered
```

only when you can:

```text
[ ] reason about GC through reachability
[ ] explain closure retention
[ ] identify listener/subscriber leaks
[ ] design bounded caches
[ ] distinguish heap from process memory
[ ] reason about weak references without overclaiming
[ ] distinguish object lifetime from resource lifetime
[ ] diagnose memory growth with evidence
[ ] define production memory budgets
[ ] defend memory decisions at principal level
```

---

# 39. Assessment Completion Snapshot

```md
# Chapter 116 — Completion Snapshot

Status:
[ ] Not Started
[~] In Progress
[?] Needs Revision
[+] Completed
[*] Mastered

Score:
____ / 100

Primary Gaps:
-

Reachability:
[ ] weak
[ ] developing
[ ] strong
[ ] principal

GC Reasoning:
[ ] weak
[ ] developing
[ ] strong
[ ] principal

Ownership / Lifecycle:
[ ] weak
[ ] developing
[ ] strong
[ ] principal

Cache Design:
[ ] weak
[ ] developing
[ ] strong
[ ] principal

External Memory:
[ ] weak
[ ] developing
[ ] strong
[ ] principal

Production Diagnosis:
[ ] weak
[ ] developing
[ ] strong
[ ] principal
```

---

# 40. Completion Criteria

```text
[ ] 10 questions completed
[ ] 100 points scored
[ ] reachability is used as the primary GC mental model
[ ] retaining paths are identified before proposing fixes
[ ] cache growth is analyzed as an ownership/boundedness problem
[ ] listener lifecycle is explicit
[ ] WeakMap/WeakRef are used with correct semantics
[ ] FinalizationRegistry is not treated as deterministic cleanup
[ ] heap and process memory are distinguished
[ ] external/native memory is considered
[ ] memory budgets are defined
[ ] production diagnostics are evidence-based
```

---

# 41. Canonical Memory Mental Model

Use this model:

```text
Objects consume memory.

References create edges.

Long-lived roots keep reachable objects alive.

Garbage collection can reclaim objects
when they are no longer reachable according to
the runtime's collection model.

Therefore:

unexpected lifetime
→ unexpected retention edge
→ unexpected memory usage
```

Then debug:

```text
What is allocated?

Who references it?

Who references that owner?

Why does the owner remain alive?

When should that ownership end?

Where is cleanup enforced?

Is growth bounded?
```

---

# 42. Final Principal Principle

> **Memory problems are usually lifetime problems before they are garbage-collector problems.**

The mature production model is:

```text
allocation
+
ownership
+
reachability
+
lifetime
+
boundedness
+
cleanup
+
measurement
```

A reliable JavaScript system does not merely hope that:

```text
GC will eventually clean things up.
```

It deliberately controls:

```text
who owns state
how long it lives
how much may exist
when it is released
how release is verified
```