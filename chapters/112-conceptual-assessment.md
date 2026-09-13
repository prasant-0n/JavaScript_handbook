# Chapter 112 — Conceptual Assessment — 50 Questions

> **JavaScript Mastery — Part XXI: Assessment**
>
> **Assessment:** 50 conceptual questions across the complete JavaScript curriculum.
>
> **Role perspective:** Principal JavaScript Engineer · ECMAScript Specialist · Runtime/Engine Engineer · Browser Engineer · Node Architect · Security Engineer · Performance Engineer · Backend Architect · Principal Interviewer
>
> **Status:** `[ ] Not Started`
>
> **Assessment rule:** **Do not search for answers while attempting the assessment. Retrieve first. Explain second. Verify later.**
>
> **Scoring rule:** Correctness is not merely selecting the right conclusion. A strong answer should explain **what**, **why**, **how**, **internals**, **cost**, **failure modes**, and **trade-offs** where the question requires them.

---

# 1. Assessment Mission

This assessment tests whether the learner can reason about JavaScript rather than merely recall syntax.

You are expected to connect:

```text
language semantics
→ execution model
→ runtime
→ engine
→ host platform
→ application architecture
→ production behavior
```

The questions intentionally move across abstraction levels.

---

# 2. Instructions

Before answering:

```text
Close notes.
Close search.
Predict.
Explain from memory.
Then verify.
```

For each question record:

```text
answer
confidence
reasoning
missing knowledge
follow-up
```

---

# 3. Confidence Scale

Use:

```text
5 — Certain and can defend
4 — Confident
3 — Partially confident
2 — Guessing
1 — No idea
```

Do not confuse confidence with correctness.

---

# 4. Suggested Scoring

Each question:

```text
0–4 points
```

Suggested interpretation:

```text
4 → correct, precise, explained
3 → correct with minor gap
2 → partially correct
1 → weak/major misconception
0 → incorrect/no meaningful answer
```

Total:

```text
50 × 4 = 200 points
```

Interpretation:

```text
180–200 → Principal-level conceptual readiness
160–179 → Strong, targeted revision needed
140–159 → Good foundation, meaningful gaps remain
120–139 → Intermediate understanding
<120    → Rebuild dependencies before advanced assessment
```

These bands are learning heuristics, not hiring standards.

---

# 5. Answer Format

For each question write:

```text
Answer:
Reasoning:
Confidence:
Key rule:
Related concept:
```

For advanced questions also include:

```text
Trade-off:
Failure mode:
Production implication:
```

---

# 6. Question 1 — JavaScript and ECMAScript

What is the difference between:

```text
JavaScript
ECMAScript
Node.js
Browser JavaScript
```

Explain:

```text
which layer defines language semantics
which layer defines host APIs
why "JavaScript behavior" is sometimes not specified by ECMAScript alone
```

---

# 7. Question 2 — Values and Types

JavaScript has primitive values and objects.

Explain:

```text
primitive values
object values
reference-like behavior
identity
mutability
```

Why can:

```js
const obj = {};
obj.x = 1;
```

work even though:

```text
obj
```

is declared with:

```text
const
```

---

# 8. Question 3 — `number` and Floating Point

Explain why:

```js
0.1 + 0.2 === 0.3
```

does not behave like many beginners expect.

Your answer should cover:

```text
IEEE 754
binary floating-point representation
rounding
comparison
when to use integer scaling
when BigInt is appropriate
```

---

# 9. Question 4 — BigInt

Explain:

```text
what BigInt solves
what it does not solve
```

Why does mixing:

```js
1n
+
1
```

cause a problem?

Discuss:

```text
precision
operator behavior
serialization
interop
```

---

# 10. Question 5 — Strings and Unicode

Explain why:

```js
"😀".length
```

is not necessarily the number of user-perceived characters.

Discuss:

```text
UTF-16 code units
code points
grapheme clusters
```

Why is:

```text
string.length
```

not a universal "character count"?

---

# 11. Question 6 — Coercion and Equality

Compare:

```text
==
===
Object.is()
```

Explain:

```text
type coercion
NaN
+0 / -0
```

Give a situation where:

```text
Object.is
```

has different semantics from:

```text
===
```

---

# 12. Question 7 — Scope

Explain:

```text
lexical scope
dynamic scope
lexical environments
identifier resolution
```

Why does JavaScript use lexical scoping?

---

# 13. Question 8 — Hoisting and TDZ

Explain the difference between:

```text
var
let
const
function declarations
```

during initialization.

What does:

```text
Temporal Dead Zone
```

actually mean?

Avoid saying merely:

```text
"let is not hoisted."
```

Explain the underlying behavior.

---

# 14. Question 9 — Closures

Define a closure precisely.

Explain:

```js
function makeCounter() {
  let count = 0;

  return () => ++count;
}
```

Why does `count` remain available after:

```text
makeCounter()
```

returns?

Discuss:

```text
lexical environment
lifetime
memory
performance
```

---

# 15. Question 10 — `this`

Explain why:

```js
const obj = {
  value: 10,
  getValue() {
    return this.value;
  }
};
```

can behave differently from:

```js
const getValue = obj.getValue;
```

What determines `this`?

Discuss:

```text
method call
plain call
constructor call
bind/call/apply
arrow functions
```

---

# 16. Question 11 — Objects and Property Keys

Explain JavaScript property keys:

```text
strings
symbols
```

What happens when using:

```js
obj[42]
```

?

How does this differ from:

```js
obj["42"]
```

?

---

# 17. Question 12 — Property Descriptors

Explain:

```text
writable
enumerable
configurable
value
get
set
```

How can two properties with identical values behave differently during:

```text
assignment
enumeration
deletion
definition
```

?

---

# 18. Question 13 — Prototype Chain

Explain:

```text
own property lookup
prototype lookup
prototype chain
```

For:

```js
obj.x
```

what happens when:

```text
x exists on obj
x does not exist on obj
x exists on a prototype
x exists nowhere
```

?

---

# 19. Question 14 — Classes

Explain what:

```js
class User {}
```

really provides in JavaScript.

Discuss:

```text
prototype methods
constructor
inheritance
instanceof
class syntax vs object model
```

Why is JavaScript still fundamentally prototype-based?

---

# 20. Question 15 — Proxy and Reflect

What does a `Proxy` allow you to intercept?

Explain:

```text
get
set
has
ownKeys
getOwnPropertyDescriptor
defineProperty
```

Why should Proxy handlers preserve language invariants?

Discuss:

```text
metaprogramming
performance
debugging
security
```

---

# 21. Question 16 — Symbols

Why do Symbols exist?

Explain:

```text
unique identity
non-string property keys
well-known symbols
protocol customization
```

Give examples of language behavior influenced by well-known symbols.

---

# 22. Question 17 — Iterables

Distinguish:

```text
iterable
iterator
iteration protocol
```

Explain how:

```js
for (const x of value) {}
```

finds values.

What role does:

```text
Symbol.iterator
```

play?

---

# 23. Question 18 — Generators

Explain:

```js
function* numbers() {
  yield 1;
  yield 2;
}
```

What does:

```text
yield
```

do to execution?

How does a generator differ from:

```text
normal function
iterator
async generator
```

?

---

# 24. Question 19 — Typed Arrays

Why are:

```text
TypedArray
ArrayBuffer
DataView
```

different concepts?

Explain the relationship between:

```text
raw bytes
views
numeric interpretation
```

Where are typed arrays useful in production?

---

# 25. Question 20 — JSON vs Structured Clone

Why is:

```text
JSON serialization
```

not equivalent to:

```text
structured cloning
```

Discuss values such as:

```text
undefined
BigInt
Map
Set
Date
ArrayBuffer
cycles
```

---

# 26. Question 21 — Errors

Explain the difference between:

```text
throwing
rejecting a Promise
returning an error object
```

When should an error be:

```text
caught
translated
logged
re-thrown
```

?

---

# 27. Question 22 — Disposable Resources

What problem do:

```text
using
await using
DisposableStack
AsyncDisposableStack
```

address?

Compare:

```text
exception-based cleanup
manual finally
structured resource management
```

---

# 28. Question 23 — Async JavaScript

Explain what happens when:

```js
async function f() {
  return 42;
}
```

is called.

Why is the result not:

```text
42
```

but:

```text
Promise
```

?

---

# 29. Question 24 — Promise Reactions

Explain conceptually:

```text
Promise state
reaction registration
job scheduling
reaction execution
```

Why does:

```js
Promise.resolve().then(() => console.log("A"));
console.log("B");
```

print:

```text
B
A
```

?

---

# 30. Question 25 — Microtasks / Jobs

Explain:

```text
synchronous execution
job/microtask execution
task/macrotask execution
```

without collapsing all runtimes into one simplified phrase.

Why can excessive microtask work delay:

```text
timers
rendering
I/O
```

?

---

# 31. Question 26 — Browser Event Loop

Explain the browser's event loop at a conceptual level.

Discuss the relationship between:

```text
JavaScript execution
tasks
microtasks
rendering opportunities
Web APIs
```

Why is:

```text
setTimeout(..., 0)
```

not an immediate execution guarantee?

---

# 32. Question 27 — Node.js Event Loop

Explain how Node.js uses:

```text
event loop
libuv
OS facilities
thread pool where applicable
```

Discuss why:

```text
I/O-bound async work
```

can scale well while:

```text
CPU-heavy JavaScript
```

can block a single Node event loop thread.

---

# 33. Question 28 — `async` / `await`

What does:

```js
const value = await promise;
```

mean operationally?

Explain why `await`:

```text
does not block the entire Node process
```

but:

```text
does suspend the current async function
```

for the awaited operation.

---

# 34. Question 29 — Cancellation

Why is:

```text
Promise cancellation
```

not simply the same as:

```text
stopping a Promise
```

?

Explain the role of:

```text
AbortController
AbortSignal
```

and why cancellation must propagate into actual underlying operations to have strong effect.

---

# 35. Question 30 — Streams and Backpressure

Why is streaming preferable to:

```text
load entire dataset into memory
```

for some workloads?

Explain:

```text
producer
consumer
buffer
backpressure
flow control
```

What can go wrong when a producer ignores a slow consumer?

---

# 36. Question 31 — Realms and Agents

What are:

```text
Realm
Agent
execution context
```

at a high level?

Why is it dangerous to casually say:

```text
"each JavaScript file has its own global object"
```

without specifying the host/module/runtime context?

---

# 37. Question 32 — Garbage Collection

Explain:

```text
reachability
garbage collection
generational collection
incremental work
```

Why does:

```text
"setting a variable to null"
```

not automatically mean:

```text
memory is freed immediately
```

?

---

# 38. Question 33 — Weak References

Why do:

```text
WeakRef
FinalizationRegistry
```

exist?

What can they help with?

Why should they not be treated as:

```text
deterministic cleanup tools
```

?

---

# 39. Question 34 — JavaScript Engine Optimization

Explain the purpose of concepts such as:

```text
hidden classes / shapes
inline caches
JIT compilation
deoptimization
optimized code
```

Why can seemingly harmless code changes affect engine optimization?

---

# 40. Question 35 — DOM

What is the DOM?

Explain the difference between:

```text
HTML source
DOM tree
JavaScript objects
browser rendering
```

Why does changing the DOM not necessarily mean:

```text
immediate pixels on screen
```

?

---

# 41. Question 36 — Browser Events

Explain:

```text
capture
target
bubble
```

How does:

```js
event.stopPropagation()
```

differ conceptually from:

```js
event.preventDefault()
```

?

---

# 42. Question 37 — Web APIs vs ECMAScript

Why is:

```js
fetch()
```

not defined by ECMAScript alone?

Distinguish:

```text
language standard
host/browser APIs
Node APIs
Web platform specifications
```

---

# 43. Question 38 — Web Workers

What problem do:

```text
Web Workers
```

solve?

Why can they improve responsiveness for CPU-intensive browser work?

Discuss:

```text
separate execution context
message passing
structured clone
transferables
SharedArrayBuffer
```

at a conceptual level.

---

# 44. Question 39 — Node Architecture

Explain what makes Node.js different from:

```text
a browser
```

and from:

```text
a generic JavaScript engine
```

Discuss:

```text
V8
libuv
Node core APIs
process
filesystem
network
```

---

# 45. Question 40 — Modules

Compare:

```text
ES Modules
CommonJS
```

Discuss:

```text
evaluation
loading
exports/imports
interoperability
caching
resolution
```

Why is:

```text
"ESM is just CommonJS with different syntax"
```

an inadequate explanation?

---

# 46. Question 41 — package.json and Resolution

Explain the role of:

```text
package.json
dependencies
devDependencies
exports
imports
type
engines
```

How can package resolution affect:

```text
which code is executed
```

and therefore:

```text
security
compatibility
performance
```

?

---

# 47. Question 42 — REST API Design

Design a REST endpoint for:

```text
update task title
```

What should you decide about:

```text
method
status codes
validation
authorization
versioning
concurrency
idempotency
errors
```

Explain why API design is more than URL naming.

---

# 48. Question 43 — Database Transactions

Explain why:

```text
transaction scope
```

matters.

Why is:

```text
DB transaction
→ remote HTTP call
→ commit
```

often a poor production design?

Discuss:

```text
locks
latency
failure
rollback
outbox
```

---

# 49. Question 44 — Cache Correctness

Explain why:

```text
cache-aside
```

can still produce stale data even when you do:

```text
write DB
→ delete cache
```

Describe a concurrent reader race.

Then give at least two mitigation strategies.

---

# 50. Question 45 — Job Queues

Why do queues need:

```text
leases
retries
idempotency
dead letters
```

?

What happens when:

```text
worker performs an external side effect
→ crashes
→ job becomes available again
```

?

---

# 51. Question 46 — Event-Driven Architecture

Distinguish:

```text
command
domain event
integration event
job
```

Why should:

```text
TaskCompleted
```

not be treated as the same thing as:

```text
please complete task
```

?

---

# 52. Question 47 — WebSocket Reliability

A client receives:

```text
event 100
event 102
```

What does this imply?

Design a recovery strategy using:

```text
sequence
replay
snapshot
deduplication
reconnect
```

---

# 53. Question 48 — Security

Identify the vulnerability in:

```js
app.patch("/users/:id", async (req, res) => {
  const user =
    await db.users.findById(req.params.id);

  Object.assign(user, req.body);

  await db.users.save(user);

  res.json(user);
});
```

Discuss at least:

```text
authorization
mass assignment
tenant isolation
validation
data exposure
concurrency
```

---

# 54. Question 49 — Performance

An endpoint has:

```text
p50 = 20 ms
p95 = 70 ms
p99 = 900 ms
```

What does this suggest?

Where would you investigate?

```text
database
GC
event loop
downstream dependency
lock contention
cache
serialization
tail amplification
```

Explain why averages alone can hide severe user impact.

---

# 55. Question 50 — Principal Architecture Judgment

You inherit a JavaScript backend with:

```text
microservices
Redis
Kafka
Postgres
WebSockets
three worker systems
Kubernetes
multiple caches
```

The product has:

```text
20,000 users
```

and:

```text
low traffic
small engineering team
```

Leadership asks:

```text
"How can we scale this architecture further?"
```

What do you do first?

Your answer must address:

```text
measurement
actual bottleneck
operational cost
failure modes
team ownership
simplification
future growth
```

The best answer may include:

```text
removing complexity
```

instead of adding more infrastructure.

---

# 56. Scoring Rubric

For each answer evaluate:

## 4 — Principal-quality

```text
correct
precise
internally coherent
explains mechanism
recognizes edge cases
states assumptions
understands trade-offs
```

## 3 — Strong

```text
correct
mostly complete
minor omissions
```

## 2 — Developing

```text
partially correct
important gap
```

## 1 — Weak

```text
memorized phrase
unclear mechanism
major misconception
```

## 0 — Missing

```text
incorrect
or
no meaningful answer
```

---

# 57. Anti-Memorization Rule

Do not award full credit for phrases such as:

```text
"JavaScript is single-threaded."
"Promises are asynchronous."
"let is not hoisted."
"Node is non-blocking."
"Redis is fast."
"Kafka guarantees exactly once."
"React updates the DOM."
"async/await is synchronous."
```

unless the answer explains:

```text
what that statement actually means
```

and:

```text
where it stops being true
```

---

# 58. Principal Reasoning Pattern

For difficult questions use:

```text
Observation
→ mechanism
→ guarantee
→ failure mode
→ cost
→ trade-off
→ production decision
```

---

# 59. Retrieval Record

```md
# Chapter 112 — Conceptual Assessment — Retrieval Record

## Attempt
- Date:
- Duration:
- Total score:
- Percentage:
- Status before:
- Status after:

## Confidence
- Q1:
- Q2:
- Q3:
- ...
- Q50:

## Strong Areas
-

## Weak Areas
-

## Misconceptions Found
-

## Questions Requiring Rebuild
-

## Questions Requiring External Verification
-

## Questions Requiring Implementation
-

## Next Revision Date
-
```

---

# 60. Domain Scorecard

After scoring, group questions:

```text
Q1–6
Language foundations

Q7–14
Scope / functions / objects

Q15–20
Advanced object/data semantics

Q21–30
Errors / async / event loop / streams

Q31–34
Spec / memory / engines

Q35–40
Browser / Node / modules

Q41–46
Packages / APIs / DB / cache / queues / events

Q47–50
Real-time / security / performance / architecture
```

---

# 61. Mastery Interpretation

A high total score is not sufficient if an entire domain is weak.

Example:

```text
190/200
```

with:

```text
security = weak
```

does not mean:

```text
production-ready
```

A principal engineer must identify critical weaknesses even when aggregate scores are high.

---

# 62. Critical-Failure Topics

Treat these as mandatory remediation if weak:

```text
authorization
tenant isolation
transactions
concurrency
async execution
event loop
memory
security
error handling
reliability
observability
```

---

# 63. Retrieval Before Verification

For each wrong answer:

```text
1. Re-answer without notes.
2. Explain the misconception.
3. Verify against authoritative source.
4. Write the corrected mental model.
5. Implement or predict behavior.
6. Re-test later.
```

---

# 64. Answer Defense

For every answer ask:

```text
Can I explain it to a junior engineer?
Can I explain its internals to a senior engineer?
Can I identify an edge case?
Can I identify a performance cost?
Can I identify a security implication?
Can I apply it in production?
```

---

# 65. Dependency Graph

```text
Chapters 01–30
       ↓
language + object + error foundations
       ↓
Chapters 31–44
       ↓
async + spec semantics
       ↓
Chapters 45–70
       ↓
engine + browser + Node + tooling
       ↓
Chapters 71–101
       ↓
algorithms + paradigms + production judgment
       ↓
Chapters 102–111
       ↓
project implementation
       ↓
Chapter 112 — Conceptual Assessment
       ↓
Chapter 113 — Output Prediction
       ↓
Chapter 114 — Debugging
       ↓
Chapter 115 — Async / Event Loop
       ↓
...
```

---

# 66. Concept Connections

## This Assessment Re-tests

```text
language semantics
scope
closures
objects
prototypes
async
Promises
event loops
streams
memory
engines
browser
Node
modules
security
APIs
databases
queues
cache
events
WebSockets
architecture
```

## Why It Matters

The assessment checks whether project implementation became:

```text
understanding
```

rather than:

```text
copying patterns
```

---

# 67. Spaced Retrieval Schedule

### Attempt 1

Complete all 50 questions without reference.

### Day 1

Redo all questions scored:

```text
0–2
```

### Day 3

Redo only questions with:

```text
confidence ≤ 3
```

### Day 7

Answer all 50 orally.

### Day 14

Defend the 10 hardest questions.

### Day 30

Repeat the full assessment.

---

# 68. Completion Criteria

Do not mark this chapter completed because:

```text
the questions were answered
```

Mark:

```text
[+] Completed
```

only when you have:

```text
[ ] attempted all 50
[ ] recorded confidence
[ ] scored all 50
[ ] reviewed every wrong answer
[ ] documented misconceptions
[ ] remediated critical security/reliability gaps
[ ] repeated weak questions
```

Mark:

```text
[*] Mastered
```

only when you can:

```text
[ ] answer without notes
[ ] explain mechanisms
[ ] defend edge cases
[ ] connect related concepts
[ ] apply concepts to production
[ ] identify when the concept should not be used
```

---

# 69. Final Mental Model

The goal of this assessment is not:

```text
"What facts do I remember?"
```

It is:

```text
"Can I reason from first principles when the facts are incomplete?"
```

A strong JavaScript engineer should move through:

```text
syntax
→ semantics
→ runtime
→ cost
→ failure
→ debugging
→ architecture
→ judgment
```

A principal engineer additionally asks:

```text
What is guaranteed?
What is implementation-specific?
What is host-specific?
What can fail?
What can become expensive?
What can become insecure?
What can become stale?
What can become impossible to operate?
What is the simplest correct alternative?
```

> **Mastery reminder:** The final objective of conceptual assessment is not perfect recall. It is reliable reasoning: knowing enough about JavaScript to predict behavior, explain mechanisms, challenge assumptions, and make production decisions under uncertainty.