# Chapter 114 — Debugging Assessment — 20 Questions

> **JavaScript Mastery — Part XXI: Assessment**
>
> **Assessment:** 20 progressive debugging problems designed to test whether you can move from symptom → evidence → hypothesis → minimal reproduction → root cause → fix → regression protection.
>
> **Role perspective:** Principal JavaScript Engineer · ECMAScript Specialist · Runtime/Engine Engineer · Browser Engineer · Node.js Architect · Performance/Security Engineer · Principal Interviewer
>
> **Status:** `[ ] Not Started`
>
> **Core rule:** **Do not patch the symptom before proving the cause.**

---

# 1. Assessment Mission

This assessment evaluates whether you can debug JavaScript systematically rather than by trial and error.

The target skill is:

```text
observe
→ reproduce
→ isolate
→ instrument
→ form hypotheses
→ eliminate hypotheses
→ identify root cause
→ implement minimal correct fix
→ add regression protection
→ verify
```

The assessment intentionally mixes:

```text
language bugs
scope bugs
object/prototype bugs
async bugs
Promise bugs
event-loop bugs
Node.js runtime bugs
browser bugs
state bugs
memory/resource issues
performance problems
security-sensitive failures
architecture/debugging judgment
```

---

# 2. Debugging Standard

For every question, do not immediately rewrite the code.

First record:

```text
Observed symptom
Reproduction
Expected behavior
Actual behavior
Environment
Evidence
Hypotheses
Tests
Root cause
Fix
Regression test
```

---

# 3. Scoring

Each question:

```text
0–5 points
```

Total:

```text
100 points
```

Recommended interpretation:

```text
90–100 → Principal-level debugging discipline
80–89  → Strong debugging ability with targeted gaps
70–79  → Functional but inconsistent
60–69  → Significant process gaps
<60     → Rebuild debugging methodology
```

---

# 4. Full-Credit Standard

A 5-point answer demonstrates:

```text
1. accurate diagnosis
2. evidence-based reasoning
3. minimal root-cause fix
4. explanation of why the bug happened
5. regression protection
```

A correct patch with no explanation is not full credit.

---

# 5. Debugging Workflow

Use this sequence:

```text
1. Define the failure.
2. Reproduce it reliably.
3. Establish environment assumptions.
4. Capture the smallest failing case.
5. Separate symptom from cause.
6. Form competing hypotheses.
7. Instrument only what helps distinguish them.
8. Identify the root cause.
9. Fix the smallest correct layer.
10. Test the regression.
11. Check adjacent failure modes.
12. Document the invariant.
```

---

# 6. Question 1 — Scope Regression

```js
function buildLabels(items) {
  var labels = [];

  for (var i = 0; i < items.length; i++) {
    labels.push(() => `Item ${i}`);
  }

  return labels;
}

const labels = buildLabels(["A", "B", "C"]);

console.log(labels[0]());
console.log(labels[1]());
console.log(labels[2]());
```

### Task

The developer expected:

```text
Item 0
Item 1
Item 2
```

but receives a different result.

Find:

```text
root cause
minimal fix
```

Then explain whether the issue is:

```text
closure
loop scope
callback timing
```

---

# 7. Question 2 — Destructuring Default Bug

```js
function createUser({ name = "Anonymous", age = 18 }) {
  return {
    name,
    age
  };
}

console.log(createUser({ name: "A", age: 0 }));
```

A test reports that age is unexpectedly `0` in one environment and the developer tries to “fix” it with:

```js
age = age || 18;
```

### Task

Explain whether that change is correct.

Identify:

```text
semantic distinction
data-validity issue
proper fix
```

Your answer must distinguish:

```text
undefined
0
null
false
```

when appropriate.

---

# 8. Question 3 — Mutating Shared State

```js
const defaultOptions = {
  retries: 3,
  headers: {}
};

function createClient(options = defaultOptions) {
  options.headers["x-client"] = "demo";
  return options;
}

const a = createClient();
const b = createClient();

console.log(a.headers);
console.log(b.headers);
```

### Symptom

Adding a header for one client unexpectedly affects another client.

### Task

Diagnose:

```text
root cause
aliasing path
minimal fix
```

Then explain whether you need:

```text
shallow copy
deep copy
structural redesign
```

and why.

---

# 9. Question 4 — Prototype Leakage

```js
const permissions = {
  admin: true
};

const user = Object.create(permissions);

user.name = "Milan";

function canAccess(key) {
  return !!user[key];
}

console.log(canAccess("admin"));
```

A developer intended to allow access only for explicitly assigned user permissions.

### Task

Find the security-sensitive bug.

Explain:

```text
property lookup
prototype chain
own-property validation
```

Then propose a hardened approach.

---

# 10. Question 5 — Getter With Hidden Side Effect

```js
const account = {
  balance: 100,
  get available() {
    this.balance -= 10;
    return this.balance;
  }
};

console.log(account.available);
console.log(account.available);
```

### Symptom

A seemingly harmless read changes account state.

### Task

Explain:

```text
why the code is legal
why it is dangerous
what invariant is violated
```

Propose a production-safe design.

---

# 11. Question 6 — Async Race on Shared State

```js
let currentUser = null;

async function loadUser(id) {
  const response = await fetch(`/users/${id}`);
  return response.json();
}

async function selectUser(id) {
  currentUser = await loadUser(id);
}

selectUser(1);
selectUser(2);
```

### Symptom

Selecting user 2 can sometimes result in user 1 appearing on screen.

### Task

Diagnose the race.

Explain why:

```text
request order
```

does not imply:

```text
completion order
```

Propose at least two correct solutions:

```text
request identity
cancellation
```

Then compare them.

---

# 12. Question 7 — Missing Promise Error Handling

```js
async function loadData() {
  return JSON.parse("{broken}");
}

loadData().then(data => {
  console.log(data);
});
```

### Symptom

The application reports:

```text
unhandled rejection
```

### Task

Identify the failure.

Explain:

```text
throw inside async function
→ rejected Promise
```

Then show where the error should be handled.

Also explain why wrapping only the `.then()` body in `try/catch` would not address the original rejection.

---

# 13. Question 8 — `try/catch` Around Asynchronous Work

```js
try {
  setTimeout(() => {
    throw new Error("boom");
  }, 0);
} catch (error) {
  console.log("caught");
}
```

### Symptom

The developer expects:

```text
caught
```

### Task

Explain why the expectation is wrong.

Identify:

```text
synchronous boundary
callback execution
exception ownership
```

Then give two correct strategies depending on the API design.

---

# 14. Question 9 — Promise Chain Swallows Failure

```js
fetch("/api/data")
  .then(response => response.json())
  .then(data => {
    process(data);
  })
  .then(() => {
    console.log("done");
  });
```

`process(data)` can throw, but monitoring shows no error.

### Task

Diagnose how the rejection becomes invisible.

Explain:

```text
Promise chain propagation
terminal handler absence
```

Give a production-safe pattern.

---

# 15. Question 10 — Event Listener Identity

```js
const button = document.querySelector("#save");

button.addEventListener("click", () => {
  save();
});

button.removeEventListener("click", () => {
  save();
});
```

### Symptom

The listener is not removed.

### Task

Explain why.

Identify the identity requirement for:

```text
removeEventListener
```

Then provide:

```text
minimal fix
```

and one architectural pattern that makes listener lifecycle easier to manage.

---

# 16. Question 11 — Stale Closure in UI State

```js
let count = 0;

function scheduleIncrement() {
  const snapshot = count;

  setTimeout(() => {
    count = snapshot + 1;
  }, 100);
}

scheduleIncrement();
scheduleIncrement();
```

### Symptom

The developer expected the final value to be:

```text
2
```

but gets:

```text
1
```

### Task

Find the stale-state problem.

Explain:

```text
snapshot
closure
delayed mutation
lost update
```

Provide two fixes:

```text
derive current state at execution time
serialize / coordinate updates
```

---

# 17. Question 12 — Node.js Environment Variable Parsing

```js
const enabled = process.env.FEATURE_X || false;

if (enabled) {
  console.log("enabled");
}
```

Environment:

```text
FEATURE_X=false
```

### Symptom

The feature is enabled.

### Task

Diagnose the bug.

Explain:

```text
environment variables are strings
truthiness
configuration parsing
```

Provide a safer parser.

---

# 18. Question 13 — Event Loop Starvation

Node.js code:

```js
function block() {
  const end = Date.now() + 3000;

  while (Date.now() < end) {}
}

setTimeout(() => {
  console.log("timer fired");
}, 0);

block();
```

### Symptom

The timer fires approximately three seconds later.

### Task

Explain why.

Distinguish:

```text
timer registration
CPU execution
event-loop progress
```

Then explain why increasing timer priority is not the correct fix.

---

# 19. Question 14 — Memory Retention Through Closure

```js
function createHandler() {
  const largeData = new Array(10_000_000).fill("x");

  return function handler() {
    console.log(largeData[0]);
  };
}

const handlers = [];

for (let i = 0; i < 100; i++) {
  handlers.push(createHandler());
}
```

### Symptom

Memory usage grows dramatically.

### Task

Identify the retention path.

Explain:

```text
closure
reachable object
GC eligibility
```

Then propose ways to reduce memory retention.

Do not claim:

```text
"GC should automatically delete it."
```

without explaining reachability.

---

# 20. Question 15 — Accidental O(n²) Work

```js
function mergeUsers(users, permissions) {
  return users.map(user => {
    const permission = permissions.find(
      permission => permission.userId === user.id
    );

    return {
      ...user,
      permission
    };
  });
}
```

### Symptom

Performance degrades sharply as data size grows.

### Task

Diagnose the complexity.

Then redesign using an appropriate data structure.

Explain:

```text
current complexity
new complexity
memory tradeoff
```

Do not optimize without stating the workload assumption.

---

# 21. Question 16 — Duplicate Side Effect After Retry

```js
async function createOrder(order) {
  const response = await fetch("/orders", {
    method: "POST",
    body: JSON.stringify(order)
  });

  if (!response.ok) {
    throw new Error("request failed");
  }

  return response.json();
}

async function submit(order) {
  try {
    return await createOrder(order);
  } catch {
    return createOrder(order);
  }
}
```

### Symptom

Customers occasionally receive two orders.

### Task

Explain the deeper problem.

The key question is:

```text
Did the server fail before processing the request,
or did the client fail after the server processed it?
```

Design a reliable approach using:

```text
idempotency
request identity
retry policy
```

---

# 22. Question 17 — AbortController Used Too Late

```js
const controller = new AbortController();

const response = await fetch("/large-report", {
  signal: controller.signal
});

controller.abort();

const data = await response.json();
```

### Symptom

The developer expects abort to stop everything after `fetch()` resolves.

### Task

Explain what `abort()` can and cannot guarantee at this point.

Distinguish:

```text
request cancellation
response processing
already-completed work
consumer logic
```

Then propose a correct cancellation architecture.

---

# 23. Question 18 — Security Bug Through Dynamic Evaluation

```js
function calculate(expression) {
  return Function(`"use strict"; return (${expression})`)();
}

console.log(calculate(userInput));
```

### Symptom

The feature works for arbitrary arithmetic but security review rejects the design.

### Task

Identify the security boundary violation.

Explain:

```text
code
vs
data
```

and why restricting the expression to “expected arithmetic” is not sufficient protection when arbitrary JavaScript evaluation is reachable.

Provide a safer architecture.

---

# 24. Question 19 — Debugging a Production-Only Bug

A Node.js service reports:

```text
Error: Cannot read properties of undefined
```

Only 0.1% of requests fail.

Local reproduction:

```text
impossible
```

Logs contain:

```text
request received
request completed
```

but no user ID, request ID, dependency timing, or stack trace.

### Task

You are the principal engineer.

Do not jump to a code fix.

Design the debugging plan:

```text
instrumentation
correlation
sampling
reproduction strategy
safe logging
error classification
rollback criteria
```

Also state what information you would explicitly avoid logging.

---

# 25. Question 20 — Principal-Level Multi-Cause Incident

The service contains:

```js
const cache = new Map();

async function getProduct(id) {
  if (cache.has(id)) {
    return cache.get(id);
  }

  const response = await fetch(`/products/${id}`);

  if (!response.ok) {
    throw new Error("failed");
  }

  const product = await response.json();

  cache.set(id, product);

  return product;
}
```

Production symptoms:

```text
1. Some requests fetch the same product repeatedly.
2. During outages, latency spikes.
3. Memory usage grows over time.
4. Some callers report stale product data.
5. Retried requests can amplify dependency load.
```

### Task

Treat this as a real incident.

Identify at least five possible failure mechanisms.

For each mechanism state:

```text
Observation
Hypothesis
How to test
Likely root cause
Fix
Trade-off
Regression test
```

Your answer must consider:

```text
cache correctness
cache lifetime
in-flight deduplication
negative caching
staleness
memory growth
dependency failure
retry amplification
timeouts
cancellation
observability
```

Do not assume all five symptoms have one cause.

---

# 26. Debugging Hypothesis Table

Use this for every question.

```md
| Hypothesis | Evidence For | Evidence Against | Test | Result |
|---|---|---|---|---|
| H1 | | | | |
| H2 | | | | |
| H3 | | | | |
```

The best debugger does not ask:

```text
"What is probably wrong?"
```

The better question is:

```text
"What observation would distinguish these hypotheses?"
```

---

# 27. Root-Cause Template

```md
## Root Cause

### Symptom
-

### Expected
-

### Actual
-

### Minimal Reproduction
```js
// smallest failing example
```

### Environment
-

### Root Cause
-

### Why It Happened
-

### Minimal Correct Fix
```js
// fix
```

### Why the Fix Works
-

### Alternatives
-

### Trade-offs
-

### Regression Test
```js
// regression test
```

### Monitoring
-
```

---

# 28. Five-Layer Debugging Model

Classify every defect at the correct layer:

```text
Layer 1 — Language
Layer 2 — Runtime / Engine
Layer 3 — Host Platform
Layer 4 — Application
Layer 5 — System / Architecture
```

Examples:

```text
closure bug
→ language

Promise scheduling assumption
→ language + host interaction

Node timer behavior
→ host runtime

incorrect API retry
→ application

unbounded cache
→ system/application architecture
```

Do not fix a Layer 5 design problem with a Layer 1 workaround.

---

# 29. Evidence Hierarchy

Prefer evidence in this order:

```text
1. reproducible failing test
2. minimal reproduction
3. deterministic trace
4. structured logs
5. metrics
6. distributed traces
7. profiler / heap evidence
8. source inspection
9. intuition
```

Intuition can generate a hypothesis.

It should not close the investigation.

---

# 30. Symptom vs Root Cause

Examples:

```text
"API is slow"
```

is a symptom.

Possible root causes:

```text
database latency
N+1 queries
event-loop blocking
retry storm
connection pool exhaustion
large serialization
GC pressure
dependency timeout
```

Similarly:

```text
"memory is high"
```

is not a root cause.

---

# 31. Debugging With Invariants

For every important subsystem define:

```text
what must always be true?
```

Examples:

```text
A request ID remains unique per request.

A canceled operation does not update the caller-owned state.

A cache entry is not returned after its validity period.

A retry cannot create duplicate business state.

An authorization check does not trust inherited properties.
```

Then debug the violation.

---

# 32. Minimal Reproduction Ladder

Use:

```text
production
→ service
→ endpoint
→ function
→ input
→ smallest failing input
```

Example:

```text
large API request
        ↓
single field
        ↓
specific branch
        ↓
single function
        ↓
three-line reproduction
```

The goal is not merely fewer lines.

The goal is preserving the causal mechanism.

---

# 33. Instrumentation Principles

Instrumentation should answer a question.

Bad:

```text
console.log("here")
console.log("here2")
console.log(value)
console.log(value2)
```

Better:

```text
request_id
operation
input identity
state transition
latency
dependency
result
error classification
```

Prefer structured events for production systems.

---

# 34. Async Debugging Checklist

When debugging asynchronous code inspect:

```text
1. Who creates the Promise?
2. Who owns the Promise?
3. Where can it reject?
4. Where is rejection observed?
5. What work continues after failure?
6. Can operations overlap?
7. Can completion order differ from initiation order?
8. Can cancellation race with completion?
9. Can stale state overwrite fresh state?
10. Can retries duplicate side effects?
```

---

# 35. Node.js Debugging Checklist

For Node.js incidents inspect:

```text
event-loop delay
CPU saturation
memory / heap
GC behavior
open handles
socket state
connection pools
timers
Promise rejections
worker utilization
process signals
dependency latency
```

Do not infer:

```text
CPU high
```

from:

```text
API slow
```

without evidence.

---

# 36. Browser Debugging Checklist

For browser incidents inspect:

```text
DOM mutation
event listener lifecycle
network waterfall
main-thread work
layout/paint pressure
memory retention
Web Worker boundaries
AbortController lifecycle
Promise chains
state synchronization
storage/cache behavior
```

---

# 37. Security Debugging Checklist

When debugging security-sensitive behavior ask:

```text
What is attacker-controlled?

Where is it interpreted?

Which boundary failed?

Was data treated as code?

Was authorization checked?

Was identity validated?

Was inherited state trusted?

Was sensitive data logged?

Can malformed input alter control flow?
```

---

# 38. Performance Debugging Checklist

Do not start with:

```text
"optimize this function"
```

Start with:

```text
What is slow?

How slow?

Compared with what?

At what input size?

CPU or I/O?

Which phase?

Which resource?

What changed?

Is the measurement reproducible?
```

Then choose:

```text
algorithmic improvement
data structure
batching
caching
parallelism
serialization reduction
allocation reduction
runtime scheduling
```

---

# 39. Production Incident Questions

For a real incident ask:

```text
What changed?

When did it start?

Who is affected?

What is the blast radius?

Can we reproduce it?

Is the failure increasing?

What metrics changed?

Which dependency changed?

Can we safely roll back?

What is the fastest safe mitigation?

What is the durable fix?
```

---

# 40. Wrong-Fix Patterns

Reject fixes like:

```text
add random timeout
retry everything
catch and ignore
increase memory indefinitely
disable validation
disable security checks
force garbage collection without diagnosis
add logging everywhere
copy every object
make everything async
make everything synchronous
```

A fix is correct only when it addresses the causal mechanism.

---

# 41. Regression Protection

Every resolved bug should become at least one of:

```text
unit test
integration test
contract test
property-based test
load test
race test
security test
monitoring assertion
```

The protection type should match the failure mode.

---

# 42. Debugging Interview Framework

When asked:

> "How would you debug this?"

Answer in this order:

```text
1. Clarify expected vs actual.
2. Define environment.
3. Reproduce.
4. Reduce.
5. Instrument.
6. Form hypotheses.
7. Eliminate.
8. Find root cause.
9. Apply minimal fix.
10. Add regression protection.
11. Verify production impact.
```

Avoid jumping straight to a favorite technique.

---

# 43. Question Review Matrix

After the assessment, classify each question:

```text
[ ] Language semantics
[ ] Scope / closure
[ ] Objects / prototypes
[ ] Async
[ ] Promise
[ ] Event loop
[ ] Browser
[ ] Node
[ ] Memory
[ ] Performance
[ ] Security
[ ] Architecture
[ ] Debugging process
```

---

# 44. Misdiagnosis Taxonomy

Classify failed answers:

```text
A — Symptom mistaken for cause
B — No minimal reproduction
C — Wrong environment assumption
D — Insufficient evidence
E — Async ordering error
F — State ownership error
G — Identity/aliasing error
H — Memory reachability error
I — Complexity error
J — Security-boundary error
K — Incorrect runtime assumption
L — Fix without regression test
M — Over-engineered fix
N — Under-specified diagnosis
```

---

# 45. Debugging Confidence

For every diagnosis record:

```text
Confidence:
1 = guess
2 = plausible
3 = supported
4 = strongly supported
5 = demonstrated by reproduction
```

Do not confuse:

```text
high confidence
```

with:

```text
proof
```

---

# 46. Implementation Challenge

Choose the three questions you found hardest.

For each produce:

```text
1. broken implementation
2. minimal reproduction
3. failing test
4. fixed implementation
5. regression test
6. benchmark if performance-related
7. security test if security-related
8. explanation
```

Progression:

```text
Guided
→ Partially Guided
→ No Reference
→ Edge-Case Hardened
→ Production-Grade
```

---

# 47. Principal Judgment Exercise

For Question 20, rank possible fixes using:

```text
Correctness
Performance
Memory
Security
Reliability
Maintainability
Scalability
Observability
Developer Experience
Operational Complexity
Future Change
```

Do not choose the fix with the lowest code size automatically.

---

# 48. Debugging Mastery Gate

You can mark:

```text
[+] Completed
```

when:

```text
[ ] all 20 problems attempted
[ ] root causes identified
[ ] minimal reproductions written
[ ] fixes proposed
[ ] regression protection described
```

Mark:

```text
[*] Mastered
```

only when you can:

```text
[ ] debug unfamiliar JavaScript systematically
[ ] distinguish symptom from cause
[ ] prove hypotheses with evidence
[ ] reason across async boundaries
[ ] identify state races
[ ] reason about memory reachability
[ ] analyze performance bottlenecks
[ ] recognize security boundaries
[ ] debug Node/browser runtime behavior
[ ] defend the chosen fix
```

---

# 49. Retrieval Record

```md
# Chapter 114 — Debugging Assessment — Retrieval Record

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
- Q16:
- Q17:
- Q18:
- Q19:
- Q20:

## Strongest Areas
-

## Weakest Areas
-

## Recurring Misdiagnosis
-

## Async Gaps
-

## Memory / Performance Gaps
-

## Security Gaps
-

## Runtime Gaps
-

## Process Gaps
-

## Questions Requiring Rework
-

## Next Review
-
```

---

# 50. Spaced Retrieval Schedule

### Day 0

Complete all 20.

### Day 1

Redo every question where:

```text
confidence < 4
```

or:

```text
score < 5
```

### Day 3

Redo:

```text
Q1–5
```

for language/object reasoning.

### Day 7

Redo:

```text
Q6–13
```

for async/runtime debugging.

### Day 14

Redo:

```text
Q14–18
```

for memory/performance/security.

### Day 21

Redo:

```text
Q19–20
```

as oral principal-engineer incident exercises.

### Day 30

Create five original debugging problems and solve them using the full workflow.

---

# 51. Dependency Graph

```text
Chapters 01–30
        ↓
language + scope + objects + errors
        ↓
Chapters 31–44
        ↓
async + jobs + cancellation + runtime isolation
        ↓
Chapters 45–70
        ↓
memory + engines + browser + Node + tooling
        ↓
Chapters 71–101
        ↓
algorithms + architecture + production reasoning
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
Chapter 115 — Async/Event Loop Assessment
```

---

# 52. Concept Connections

## Depends On

```text
scope
closures
objects
prototypes
error handling
Promises
async/await
event loop
memory
performance
security
Node/browser runtime
```

## Builds Toward

```text
async debugging
production incident response
architecture assessment
system design
principal-level engineering judgment
```

## Revisited Concepts

```text
coercion
closure
this
prototype lookup
Promise rejection
cancellation
event-loop behavior
cache
memory retention
complexity
security boundaries
```

## Why This Chapter Matters

A principal engineer is rarely valuable because they can write code quickly.

They are valuable because they can answer:

```text
What failed?
Why did it fail?
How do we know?
How do we fix it safely?
How do we stop it happening again?
```

---

# 53. Track A — Core Theory

Study:

```text
execution model
scope
closures
object identity
property lookup
Promise semantics
job scheduling
event-loop boundaries
memory reachability
algorithmic complexity
security boundaries
```

Deliverable:

```text
explain the causal mechanism
```

---

# 54. Track B — Implementation

Build:

```text
minimal reproductions
failing tests
instrumentation
fixed implementations
regression tests
benchmarks
```

Deliverable:

```text
prove the fix
```

---

# 55. Track C — Interview / Reasoning

Practice:

```text
30-second diagnosis
2-minute debugging plan
5-minute root-cause defense
principal-level trade-off analysis
```

Deliverable:

```text
reason clearly under uncertainty
```

---

# 56. Assessment Completion Snapshot

```md
# Chapter 114 — Completion Snapshot

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

Root-Cause Discipline:
[ ] weak
[ ] developing
[ ] strong
[ ] principal

Async Debugging:
[ ] weak
[ ] developing
[ ] strong
[ ] principal

Runtime Debugging:
[ ] weak
[ ] developing
[ ] strong
[ ] principal

Memory / Performance:
[ ] weak
[ ] developing
[ ] strong
[ ] principal

Security:
[ ] weak
[ ] developing
[ ] strong
[ ] principal

Production Judgment:
[ ] weak
[ ] developing
[ ] strong
[ ] principal
```

---

# 57. Completion Criteria

```text
[ ] 20 questions completed
[ ] 100 points scored
[ ] every diagnosis includes evidence
[ ] every major issue has a minimal reproduction
[ ] every fix has regression protection
[ ] async bugs are traced across boundaries
[ ] environment assumptions are explicit
[ ] memory problems are explained through reachability
[ ] performance claims include measurement/workload
[ ] security issues identify the violated boundary
[ ] principal-level incident questions are defended
```

---

# 58. Canonical Debugging Principle

> **Do not debug by changing code until the behavior changes. Debug by changing your evidence until the explanation is proven.**

The reliable sequence is:

```text
symptom
→ evidence
→ hypothesis
→ experiment
→ root cause
→ minimal fix
→ regression protection
→ production verification
```

That is the foundation of production-grade JavaScript debugging.
