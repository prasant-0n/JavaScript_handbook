# Chapter 115 — Async / Event Loop Assessment — 15 Questions

> **JavaScript Mastery — Part XXI: Assessment**
>
> **Assessment:** 15 progressive asynchronous JavaScript problems focused on Promise jobs, `await`, microtasks, timers, browser/Node host boundaries, event ordering, concurrency, cancellation, races, and production scheduling judgment.
>
> **Role perspective:** Principal JavaScript Engineer · ECMAScript Specialist · Runtime Engineer · Browser Engineer · Node.js Architect · Concurrency Engineer · Principal Interviewer
>
> **Status:** `[ ] Not Started`
>
> **Core rule:** **Do not predict async behavior from source-code order alone. Model scheduling boundaries explicitly.**

---

# 1. Assessment Mission

This assessment tests whether you can reason about asynchronous JavaScript without relying on vague rules such as:

```text
"Promises are always faster."
"await blocks JavaScript."
"setTimeout runs immediately."
"async means parallel."
"microtasks always beat everything."
```

Instead reason through:

```text
synchronous execution
→ asynchronous operation
→ completion
→ queued reaction/job/callback
→ execution
→ newly queued work
→ host scheduling
```

---

# 2. Scope Discipline

Each question may involve more than one layer.

Use this classification:

```text
ECMAScript language semantics
Browser host behavior
Node.js host behavior
Application-level concurrency
```

Do not present a browser-specific or Node-specific rule as a universal ECMAScript guarantee.

---

# 3. Assessment Scoring

Each question:

```text
0–6 points
```

Total:

```text
90 points
```

Recommended interpretation:

```text
81–90 → Principal-level async reasoning
72–80 → Strong async reasoning
63–71 → Good foundation with targeted gaps
54–62 → Significant scheduling/reasoning gaps
<54    → Rebuild async fundamentals
```

---

# 4. Full-Credit Standard

A full-credit answer should contain:

```text
1. prediction
2. ordering trace
3. scheduling explanation
4. language-vs-host classification
5. edge-case observation
6. production implication
```

For implementation questions also include:

```text
correctness
cancellation
error propagation
concurrency limits
```

---

# 5. Required Answer Format

For every prediction problem:

```md
### Prediction
1.
2.
3.

### Actual Result
1.
2.
3.

### Scheduling Trace
1.
2.
3.

### Why
-

### Governing Rule
-

### Confidence
1–5
```

Do not read any reference explanation before making the prediction.

---

# 6. Question 1 — Promise Reaction vs Synchronous Code

```js
console.log("A");

Promise.resolve().then(() => {
  console.log("B");
});

console.log("C");
```

### Task

Predict the exact output order.

Then explain:

```text
which code runs synchronously
which work is deferred
when the Promise reaction executes
```

Classify the core rule as ECMAScript semantics or host behavior.

---

# 7. Question 2 — `queueMicrotask()` and Promise Reaction Ordering

```js
console.log("A");

queueMicrotask(() => {
  console.log("B");
});

Promise.resolve().then(() => {
  console.log("C");
});

queueMicrotask(() => {
  console.log("D");
});

console.log("E");
```

### Task

Predict the exact order.

Explain the queueing sequence rather than merely saying:

```text
"microtasks happen after synchronous code."
```

Your trace must identify:

```text
registration order
queue order
execution order
```

---

# 8. Question 3 — Nested Microtasks

```js
console.log("A");

queueMicrotask(() => {
  console.log("B");

  queueMicrotask(() => {
    console.log("C");
  });

  Promise.resolve().then(() => {
    console.log("D");
  });
});

queueMicrotask(() => {
  console.log("E");
});

console.log("F");
```

### Task

Predict the output.

Then explain why newly queued microtasks affect the final order.

Do not use the shortcut:

```text
"all microtasks are one batch"
```

without explaining the queue-drain model.

---

# 9. Question 4 — `await` Splits Execution

```js
async function run() {
  console.log("A");

  await 0;

  console.log("B");
}

console.log("C");

run();

console.log("D");
```

### Task

Predict the order.

Explain the semantic boundary created by:

```js
await 0
```

Then answer:

```text
Does await block the JavaScript thread?
Does await always wait for an actual external I/O operation?
```

---

# 10. Question 5 — `await` With Already-Resolved Promise

```js
async function run() {
  console.log("A");

  await Promise.resolve("value");

  console.log("B");
}

run();

console.log("C");
```

### Task

Predict:

```text
A
B
C
```

or:

```text
A
C
B
```

Then explain why an already-fulfilled Promise still creates an asynchronous continuation point for the `async` function.

---

# 11. Question 6 — Timer vs Promise Reaction

Assume a standard JavaScript host where timers and Promise reactions are available:

```js
console.log("A");

setTimeout(() => {
  console.log("B");
}, 0);

Promise.resolve().then(() => {
  console.log("C");
});

console.log("D");
```

### Task

Predict the common observable order.

Then explicitly separate:

```text
what ECMAScript specifies
```

from:

```text
what the host must provide
```

Do not describe `setTimeout` itself as an ECMAScript language feature.

---

# 12. Question 7 — Timer Schedules More Microtasks

```js
setTimeout(() => {
  console.log("A");

  Promise.resolve().then(() => {
    console.log("B");
  });

  queueMicrotask(() => {
    console.log("C");
  });
}, 0);

Promise.resolve().then(() => {
  console.log("D");
});
```

### Task

Predict the order.

Explain what happens when a timer callback queues Promise/microtask work.

Then answer:

```text
Why can microtask execution appear "inside" a timer callback sequence from the developer's perspective?
```

---

# 13. Question 8 — Promise.all Is Not a Concurrency Magic Wand

```js
async function work(id, ms) {
  await new Promise(resolve => setTimeout(resolve, ms));
  console.log(id);
  return id;
}

async function run() {
  const results = await Promise.all([
    work("A", 100),
    work("B", 10),
    work("C", 50)
  ]);

  console.log(results);
}

run();
```

### Task

Answer all of the following:

```text
What order do A/B/C logs appear?
What order does results preserve?
Are the three operations initiated before Promise.all waits?
Does Promise.all make them CPU-parallel?
```

Then distinguish:

```text
concurrency
parallelism
result ordering
completion ordering
```

---

# 14. Question 9 — Sequential vs Concurrent Awaits

Compare:

```js
async function sequential() {
  const a = await work("A", 100);
  const b = await work("B", 100);
  return [a, b];
}
```

with:

```js
async function concurrent() {
  const promiseA = work("A", 100);
  const promiseB = work("B", 100);

  const a = await promiseA;
  const b = await promiseB;

  return [a, b];
}
```

### Task

Explain:

```text
why completion time can differ
what work has started at each await
which result ordering is preserved
```

Then state when sequential execution is actually the correct design.

---

# 15. Question 10 — Error Propagation Across `await`

```js
async function inner() {
  throw new Error("failure");
}

async function outer() {
  try {
    await inner();
    console.log("never");
  } catch (error) {
    console.log("caught");
  }
}

outer();
```

### Task

Explain:

```text
how synchronous throw inside async function becomes Promise rejection
how await observes that rejection
why catch handles it
```

Then change the problem:

```js
async function outer() {
  try {
    inner();
    console.log("continued");
  } catch (error) {
    console.log("caught");
  }
}
```

Explain the difference.

---

# 16. Question 11 — Unhandled Rejection Timing

```js
const promise = Promise.reject(new Error("boom"));

setTimeout(() => {
  promise.catch(() => {
    console.log("handled");
  });
}, 0);
```

### Task

Do not answer from memory.

Analyze:

```text
Promise creation
rejection state
reaction attachment
host unhandled-rejection reporting
```

Explain why the exact timing of an unhandled-rejection notification may depend on the host.

Your answer must distinguish:

```text
language-level Promise state
```

from:

```text
host reporting policy
```

---

# 17. Question 12 — Race Condition With Shared State

```js
let value = 0;

async function add(amount, delay) {
  const current = value;

  await new Promise(resolve => setTimeout(resolve, delay));

  value = current + amount;
}

add(10, 50);
add(20, 10);
```

### Task

Determine the final value and explain why.

Then answer:

```text
Why is this a race even though JavaScript executes one piece of JavaScript at a time?
```

Provide at least three designs:

```text
serialized update
atomic state transition at execution time
coordination primitive / queue
```

Compare the trade-offs.

---

# 18. Question 13 — Cancellation Race

```js
const controller = new AbortController();

async function task() {
  try {
    const response = await fetch("/data", {
      signal: controller.signal
    });

    const data = await response.json();

    console.log("commit", data);
  } catch (error) {
    if (error.name === "AbortError") {
      console.log("cancelled");
      return;
    }

    throw error;
  }
}

task();

setTimeout(() => {
  controller.abort();
}, 50);
```

### Task

Analyze multiple timing possibilities:

```text
fetch still pending
fetch completed but response body pending
body parsed before abort
commit about to happen
```

For each state answer:

```text
what abort can affect
what work may already have happened
what stale-state protection is still needed
```

Do not equate:

```text
abort requested
```

with:

```text
all related application work is instantly undone
```

---

# 19. Question 14 — Browser Event Handler and Async Ordering

Assume a browser environment:

```js
button.addEventListener("click", async () => {
  console.log("handler-start");

  await Promise.resolve();

  console.log("handler-after-await");
});

button.addEventListener("click", () => {
  console.log("second-handler");
});

console.log("ready");
```

### Task

When the button is clicked, reason about the relative order of:

```text
handler-start
handler-after-await
second-handler
```

Explain:

```text
event dispatch
listener invocation
async continuation
microtask checkpoint
```

Do not assume the first listener must complete all of its async body before the second listener can run.

---

# 20. Question 15 — Node.js Scheduling Investigation

Assume a current Node.js runtime and analyze this program:

```js
console.log("A");

setTimeout(() => {
  console.log("timeout");
}, 0);

setImmediate(() => {
  console.log("immediate");
});

Promise.resolve().then(() => {
  console.log("promise");
});

queueMicrotask(() => {
  console.log("microtask");
});

console.log("B");
```

### Task

Your answer must not claim that there is one universal fixed order for every invocation context.

Instead:

```text
1. Identify the definitely synchronous output.
2. Identify Promise/microtask behavior.
3. Identify the Node.js host scheduling boundary.
4. Explain why timer vs setImmediate ordering can depend on context/timing.
5. State what additional experiment you would run to make the observation reproducible.
```

Then compare execution from:

```text
top-level script
I/O callback
timer callback
```

where relevant.

---

# 21. Async Scheduling Trace Sheet

For difficult questions, build this table:

```md
| Step | Currently Executing | Sync Stack | Promise/Microtask Queue | Timer/I/O Work | State |
|---|---|---|---|---|---|
| 1 | | | | | |
| 2 | | | | | |
| 3 | | | | | |
```

Never skip directly from:

```text
source code
```

to:

```text
final output
```

when the question contains multiple scheduling boundaries.

---

# 22. Async Failure Taxonomy

Classify every missed question as one of:

```text
A — synchronous/async boundary confusion
B — Promise reaction ordering error
C — microtask queue error
D — await continuation error
E — timer/host confusion
F — event-dispatch confusion
G — completion-order/result-order confusion
H — concurrency/parallelism confusion
I — race condition
J — cancellation race
K — rejection propagation failure
L — unhandled rejection host-policy confusion
M — runtime-specific scheduling assumption
N — incorrect mental model
```

---

# 23. Language vs Host Boundary

Use this table:

```md
| Concept | Primary Layer |
|---|---|
| Promise state/reactions | ECMAScript |
| async function / await semantics | ECMAScript |
| job/microtask scheduling model | ECMAScript + host integration |
| setTimeout | Host API |
| DOM event dispatch | Browser host |
| Browser rendering opportunities | Browser host |
| setImmediate | Node.js host |
| libuv phases | Node.js host/runtime |
| I/O readiness | Host/runtime |
```

The exact observable behavior may depend on how the host integrates ECMAScript execution.

---

# 24. Concurrency Vocabulary

Do not use these words interchangeably.

### Concurrency

Multiple operations are in progress during overlapping time intervals.

### Parallelism

Work actually executes simultaneously on multiple execution resources.

### Asynchrony

Work completion is decoupled from the current synchronous call stack.

### Serialization

Work is deliberately constrained to a sequence.

### Interleaving

Multiple operations progress in a non-overlapping but interwoven schedule.

---

# 25. Async State-Machine Thinking

Model important operations as:

```text
created
→ pending
→ fulfilled
```

or:

```text
created
→ pending
→ rejected
```

For cancellation-aware operations:

```text
created
→ pending
→ cancel requested
→ cancellation observed
→ settled
```

Then distinguish:

```text
operation state
```

from:

```text
application state
```

A canceled request does not automatically roll back application state already committed elsewhere.

---

# 26. Async Debugging Checklist

When an async bug appears, ask:

```text
1. Where is the first asynchronous boundary?
2. What is synchronous before it?
3. What object represents the eventual result?
4. Who owns that Promise?
5. Where is fulfillment observed?
6. Where is rejection observed?
7. Which work can overlap?
8. Can completion order differ from initiation order?
9. Can stale work commit after newer work?
10. Can cancellation race with completion?
11. Which scheduling rules are language guarantees?
12. Which rules belong to the host?
```

---

# 27. Concurrency Control Exercise

Design a function:

```js
limitConcurrency(tasks, limit)
```

Requirements:

```text
tasks is an array of functions returning Promises
limit is a positive integer
at most limit tasks may be active at once
result order matches input order
one failure should not silently disappear
remaining work should follow a documented policy
```

Implement in stages:

```text
Guided
→ Partially Guided
→ No Reference
→ Edge-Case Hardened
→ Production-Grade
```

Then test:

```text
0 tasks
1 task
limit = 1
limit > task count
one rejection
multiple rejections
synchronous task throw
slow first task
fast later task
```

---

# 28. Async Iterator Exercise

Implement an async generator that:

```text
produces values from 1 to N
waits between values
supports consumer cancellation through return()
does not leave unnecessary pending timers
```

Then explain:

```text
producer
consumer
backpressure
cleanup
cancellation
```

---

# 29. Promise.all / allSettled / race / any Comparison

Build a comparison matrix:

```md
| Combinator | Resolves When | Rejects When | Result Shape | Typical Use |
|---|---|---|---|---|
| Promise.all | | | | |
| Promise.allSettled | | | | |
| Promise.race | | | | |
| Promise.any | | | | |
```

Then answer:

```text
Which is appropriate when every operation is required?
Which is appropriate when partial failure is expected?
Which is appropriate when the first settlement matters?
Which is appropriate when the first successful result matters?
```

---

# 30. Production Scenario

A service performs:

```text
100 dependency calls
```

per incoming request.

Developers replace:

```js
for (const id of ids) {
  results.push(await load(id));
}
```

with:

```js
await Promise.all(ids.map(load));
```

Latency improves dramatically.

One hour later:

```text
dependency CPU rises
timeouts increase
retries increase
your service gets slower
```

### Task

Explain the systems problem.

The fix is not automatically:

```text
"Use sequential awaits again."
```

Design a better strategy involving:

```text
bounded concurrency
timeouts
cancellation
retry policy
backpressure
dependency protection
observability
```

---

# 31. Async Anti-Patterns

Recognize:

```js
await Promise.all(items.map(async item => {
  await save(item);
}));
```

when the operation:

```text
has no concurrency budget
```

Also recognize:

```js
items.map(async item => save(item));
```

when the returned Promises are ignored.

And:

```js
try {
  promiseReturningFunction();
} catch {}
```

when rejection is expected asynchronously.

For each, identify:

```text
what is wrong
why it is tempting
what production failure it can create
```

---

# 32. Principal-Level Async Review Questions

Before approving asynchronous code, ask:

```text
1. What is the concurrency model?
2. What is the maximum in-flight work?
3. What controls backpressure?
4. What happens when one operation fails?
5. What happens when the caller cancels?
6. What happens after timeout?
7. Can retries overlap?
8. Can stale results commit?
9. Can duplicated side effects occur?
10. How is completion observed?
11. How is resource cleanup guaranteed?
12. What happens under load?
```

---

# 33. Scoring Rubric

### 0 — Incorrect

Incorrect prediction and no usable model.

### 1 — Guess

Correct answer with little or no reasoning.

### 2 — Basic

Correct ordering but incomplete scheduling explanation.

### 3 — Strong

Correct ordering and explanation.

### 4 — Advanced

Explains queue transitions and boundaries precisely.

### 5 — Production-Aware

Adds host/language distinction, edge cases, and failure implications.

### 6 — Principal

Precisely explains semantics, scheduling, concurrency, failure behavior, and design trade-offs.

---

# 34. Retrieval Record

```md
# Chapter 115 — Async / Event Loop Assessment — Retrieval Record

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
- Q11:
- Q12:
- Q13:
- Q14:
- Q15:

## Strongest Areas
-

## Weakest Areas
-

## Promise Ordering Errors
-

## Event Loop Errors
-

## Host Boundary Errors
-

## Concurrency Errors
-

## Cancellation Errors
-

## Questions Requiring Rework
-

## Next Review
-
```

---

# 35. Spaced Retrieval Schedule

### Day 0

Complete all 15.

### Day 1

Redo every question where:

```text
confidence < 4
```

### Day 3

Redo:

```text
Q1–5
```

without looking at notes.

### Day 7

Redo:

```text
Q6–10
```

using a queue/state trace.

### Day 14

Redo:

```text
Q11–15
```

with explicit host-boundary classification.

### Day 21

Rebuild the concurrency limiter from memory.

### Day 30

Create five original async scheduling puzzles and solve them aloud.

---

# 36. Dependency Graph

```text
Chapters 01–30
        ↓
values + scope + functions + objects + errors
        ↓
Chapters 31–40
        ↓
async foundations + jobs + browser/Node event loops
        ↓
Chapters 41–48
        ↓
spec + engine/runtime model
        ↓
Chapters 49–70
        ↓
browser APIs + workers + streams + Node + modules/tooling
        ↓
Chapters 71–101
        ↓
algorithms + paradigms + production architecture
        ↓
Chapters 102–111
        ↓
projects
        ↓
Chapter 112 — Conceptual Assessment
        ↓
Chapter 113 — Output Prediction Assessment
        ↓
Chapter 114 — Debugging Assessment
        ↓
Chapter 115 — Async / Event Loop Assessment
        ↓
Chapter 116 — Memory Assessment
```

---

# 37. Concept Connections

## Depends On

```text
Promises
async/await
ECMAScript Jobs
browser event loop
Node.js event loop
cancellation
async iteration
streams
```

## Builds Toward

```text
memory reasoning
performance reasoning
reliability engineering
production concurrency control
system design
principal-level incident analysis
```

## Concepts Revisited

```text
Promise reactions
microtasks
closures
state
errors
cancellation
streams
backpressure
worker concurrency
```

## Why This Chapter Matters Later

Async systems fail at boundaries.

A strong JavaScript engineer can explain:

```text
what is executing now
what is waiting
what is queued
what can overlap
what can fail
what can be canceled
what can become stale
```

That model underpins reliable browser and Node.js systems.

---

# 38. Track A — Core Theory

Master:

```text
Promise state
Promise reactions
async functions
await continuation
job/microtask scheduling
host integration
timers
event dispatch
concurrency
parallelism
cancellation
backpressure
```

Deliverable:

```text
predict and explain schedules
```

---

# 39. Track B — Implementation

Build:

```text
Promise-based delay
concurrency limiter
timeout wrapper
cancellable task
async iterator
retry coordinator
bounded worker pool
```

Every implementation must include:

```text
failure
cancellation
cleanup
edge-case
regression test
```

---

# 40. Track C — Interview / Reasoning

Practice answering:

```text
"What logs first?"

"Why did this Promise finish later?"

"Why did this race occur?"

"Why did Promise.all overload the dependency?"

"Why did abort not stop the side effect?"

"What is guaranteed by JavaScript and what is host-specific?"
```

Deliverable:

```text
reason without hand-waving
```

---

# 41. Async Mastery Gate

You may mark:

```text
[+] Completed
```

when:

```text
[ ] all 15 questions attempted
[ ] all predictions written before checking
[ ] scheduling traces produced
[ ] host-vs-language boundaries identified
[ ] concurrency exercise completed
[ ] async iterator exercise completed
```

Mark:

```text
[*] Mastered
```

only when you can:

```text
[ ] predict Promise/microtask ordering reliably
[ ] explain await continuation behavior
[ ] distinguish concurrency from parallelism
[ ] identify races
[ ] design bounded concurrency
[ ] reason about cancellation races
[ ] debug rejection propagation
[ ] distinguish ECMAScript from browser/Node scheduling
[ ] explain why production async systems overload dependencies
[ ] defend scheduling decisions at principal level
```

---

# 42. Assessment Completion Snapshot

```md
# Chapter 115 — Completion Snapshot

Status:
[ ] Not Started
[~] In Progress
[?] Needs Revision
[+] Completed
[*] Mastered

Score:
____ / 90

Primary Gaps:
-

Promise Reasoning:
[ ] weak
[ ] developing
[ ] strong
[ ] principal

Microtask Reasoning:
[ ] weak
[ ] developing
[ ] strong
[ ] principal

Event Loop Reasoning:
[ ] weak
[ ] developing
[ ] strong
[ ] principal

Concurrency:
[ ] weak
[ ] developing
[ ] strong
[ ] principal

Cancellation:
[ ] weak
[ ] developing
[ ] strong
[ ] principal

Production Async Design:
[ ] weak
[ ] developing
[ ] strong
[ ] principal
```

---

# 43. Completion Criteria

```text
[ ] 15 questions completed
[ ] 90 points scored
[ ] every prediction attempted before verification
[ ] every async trace identifies the first asynchronous boundary
[ ] Promise/microtask behavior is explained precisely
[ ] host-specific behavior is not mistaken for ECMAScript
[ ] concurrency and parallelism are distinguished
[ ] race conditions are identified correctly
[ ] cancellation limits are understood
[ ] bounded concurrency solution implemented
[ ] async iterator solution implemented
[ ] production dependency overload scenario defended
```

---

# 44. Canonical Async Mental Model

Use this model:

```text
Synchronous JavaScript runs now.

An asynchronous API creates or waits on work.

Completion changes some state.

A reaction/callback/continuation becomes runnable.

The host determines when the execution opportunity occurs.

The callback/continuation runs.

That execution may enqueue more work.

Application state may now change again.
```

Then ask:

```text
What is the current state?

What can happen next?

What ordering is guaranteed?

What ordering is merely observed?

What work may overlap?

What happens on failure?

What happens on cancellation?

What prevents overload?
```

---

# 45. Final Principal Principle

> **Asynchronous correctness is not about memorizing which queue “wins.” It is about identifying execution boundaries, ownership, ordering guarantees, and the consequences of overlap.**

The mature mental model is:

```text
source code
→ execution boundary
→ scheduled work
→ state transition
→ observation
→ possible interleaving
→ next transition
```

For production systems:

```text
async correctness
+
bounded concurrency
+
failure propagation
+
cancellation
+
backpressure
+
observability
=
reliable asynchronous JavaScript
```