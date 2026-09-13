# Chapter 98 — Anti-Patterns and Failure Modes

> **JavaScript Mastery — Part XIX: Judgment**
>
> **Role perspective:** Principal JavaScript Engineer · ECMAScript Language Specialist · Runtime/Engine Engineer · Browser/Platform Engineer · Node.js Architect · Performance/Security Engineer · Library Author · Principal Interviewer
>
> **Status:** `[ ] Not Started`
>
> **Verification date:** 2026-09-11
>
> **Core rule:** **An anti-pattern is not merely code that looks ugly. It is a recurring approach whose trade-offs systematically produce failure, complexity, risk, or poor outcomes in a given context.**

---

# 0. Chapter Mission

JavaScript mastery becomes meaningful when you can recognize not only:

```text
what works
```

but also:

```text
what fails
why it fails
when it fails
what makes it fail
how to detect the failure
how to repair it
when the “bad” pattern is actually justified
```

A principal engineer does not build a blacklist like:

```text
never use X
always use Y
```

Instead, the principal model is:

```text
Context
 ↓
Goal
 ↓
Constraints
 ↓
Pattern
 ↓
Trade-offs
 ↓
Failure modes
 ↓
Evidence
 ↓
Decision
```

This chapter studies failure patterns across:

```text
language semantics
scope
objects
prototypes
async
promises
events
browser APIs
Node.js
modules
dependencies
security
memory
performance
testing
architecture
distributed systems
edge/serverless
WebAssembly
```

---

# 1. Learning Objectives

By the end of this chapter, you should be able to:

- Define anti-pattern.
- Distinguish anti-pattern from:
  - style preference,
  - legacy code,
  - bug,
  - trade-off,
  - intentional design.
- Identify common JavaScript failure modes.
- Explain why each failure occurs.
- Predict the observable result.
- Build minimal reproductions.
- Diagnose root causes.
- Separate symptom from mechanism.
- Recognize hidden state.
- Recognize accidental coupling.
- Recognize temporal coupling.
- Recognize implicit contracts.
- Recognize abstraction leaks.
- Recognize premature abstraction.
- Recognize under-abstraction.
- Recognize overengineering.
- Recognize shared mutable state.
- Recognize global state.
- Recognize uncontrolled mutation.
- Recognize incorrect use of `this`.
- Recognize scope/closure mistakes.
- Recognize coercion traps.
- Recognize prototype hazards.
- Recognize enumeration mistakes.
- Recognize async race conditions.
- Recognize unhandled rejections.
- Recognize promise anti-patterns.
- Recognize callback hazards.
- Recognize event-listener leaks.
- Recognize timer leaks.
- Recognize cancellation mistakes.
- Recognize stream/backpressure failures.
- Recognize memory-retention bugs.
- Recognize unbounded caches.
- Recognize excessive allocation.
- Recognize engine deoptimization myths.
- Recognize module-graph problems.
- Recognize circular dependency hazards.
- Recognize package boundary problems.
- Recognize dependency bloat.
- Recognize supply-chain risks.
- Recognize unsafe dynamic code.
- Recognize XSS/SSRF/prototype-pollution paths.
- Recognize weak validation.
- Recognize insecure fallback behavior.
- Recognize incompatible platform assumptions.
- Recognize serverless failure modes.
- Recognize edge data-placement problems.
- Recognize Wasm FFI failures.
- Recognize testing anti-patterns.
- Recognize debugging anti-patterns.
- Recognize architecture-level failure modes.
- Design repairs rather than merely naming smells.
- Build an anti-pattern decision matrix.
- Conduct principal-level code and architecture reviews.

---

# 2. Prerequisites

Recommended chapters:

```text
Chapter 05 — Variables / Declarations
Chapter 07 — Coercion / Equality
Chapter 10 — Scope / Lexical Environments
Chapter 11 — Hoisting / TDZ
Chapter 13 — Closures
Chapter 14 — this / Invocation
Chapter 15 — Objects / Properties
Chapter 17 — Prototypes
Chapter 19 — Proxy / Reflect
Chapter 22 — Arrays
Chapter 24 — Map / Set / WeakMap / WeakSet
Chapter 25 — Iterables / Iterators
Chapter 29 — Errors
Chapter 31 — Async Fundamentals
Chapter 32 — Promise Jobs
Chapter 33 — Browser Event Loop
Chapter 34 — Node Event Loop
Chapter 35 — Promises
Chapter 36 — Async/Await
Chapter 37 — Cancellation / Abort
Chapter 38 — Async Iteration / Streams
Chapter 45 — Memory / GC
Chapter 46 — Weak References
Chapter 48 — V8 / Optimization
Chapter 55 — Fetch / HTTP
Chapter 56 — Browser Security
Chapter 57 — JavaScript Security Engineering
Chapter 58 — Node Architecture
Chapter 63 — Async Context / Diagnostics
Chapter 64 — ES Modules
Chapter 65 — CommonJS
Chapter 67 — Dependency Management / Supply Chain
Chapter 69 — Bundlers / Build Systems
Chapter 78 — Production Architecture
Chapter 79 — API Design
Chapter 83 — Observability
Chapter 84 — Reliability
Chapter 85 — Performance
Chapter 86 — Testing
Chapter 88 — Debugging
Chapter 89 — Code Review / Refactoring
Chapter 94 — Compatibility Engineering
Chapter 95 — Legacy JavaScript
Chapter 96 — WebAssembly / Native Interoperability
Chapter 97 — Edge / Serverless JavaScript

---

# 3. What Is an Anti-Pattern?

An anti-pattern is a recurring approach that appears attractive or convenient but reliably creates undesirable consequences under identifiable conditions.

A useful structure:

```text
Pattern
→ local benefit
→ hidden cost
→ repeated failure
→ recognisable signature
→ better alternative
```

Example:

```text
global mutable state
→ easy access
→ hidden coupling
→ race/test contamination
→ identifiable smell
→ explicit dependency/state boundary
```

---

# 4. Anti-Pattern vs Bug

A bug is an incorrect behavior.

An anti-pattern is a design approach that increases the probability or cost of incorrect behavior.

Example:

```js
let currentUser;
```

This is not automatically a bug.

But using global mutable `currentUser` across concurrent requests is an anti-pattern because it creates a high-risk shared-state design.

---

# 5. Anti-Pattern vs Style

Consider:

```js
function add(a, b) {
  return a + b;
}
```

versus:

```js
const add = (a, b) => a + b;
```

Either may be preferable in different code styles.

That is not necessarily an anti-pattern.

Anti-pattern analysis requires:

```text
observable risk
```

not aesthetic disagreement.

---

# 6. Anti-Pattern vs Trade-Off

Every design has costs.

Example:

```text
Map
```

versus:

```text
Object
```

Neither is universally correct.

The anti-pattern appears when the choice systematically violates the domain:

```text
Object as complex key/value index
+
untrusted keys
+
prototype-sensitive behavior
```

The problem is contextual misuse.

---

# 7. Mental Model — Failure Chain

```text
small shortcut
     ↓
implicit assumption
     ↓
hidden coupling
     ↓
edge case
     ↓
production failure
     ↓
workaround
     ↓
more complexity
```

Many large incidents begin with a tiny shortcut.

---

# 8. Failure Taxonomy

A useful classification:

```text
Correctness
Performance
Memory
Security
Reliability
Maintainability
Scalability
Observability
Compatibility
Developer Experience
Operational Complexity
```

One anti-pattern can damage several dimensions simultaneously.

---

# 9. The Most Important Review Question

When reviewing suspicious code, ask:

> **What assumption must be true for this code to remain correct?**

Then ask:

```text
Who guarantees that assumption?
How is it tested?
What happens when it stops being true?
```

That is often enough to uncover the real risk.

---

# 10. Category I — State Anti-Patterns

State problems are among the most common causes of hard-to-debug JavaScript failures.

Typical signals:

```text
global mutable state
module singleton abuse
shared request state
hidden caches
mutation through aliases
```

---

# 11. Anti-Pattern — Global Mutable State

Bad:

```js
let currentUser;

export async function handle(request) {
  currentUser = authenticate(request);

  await doWork();

  return render(currentUser);
}
```

If requests overlap, state can be corrupted.

The problem is:

```text
request-local fact
stored in
process/global state
```

---

# 12. Better Pattern — Explicit Request State

```js
export async function handle(request) {
  const currentUser =
    authenticate(request);

  await doWork();

  return render(currentUser);
}
```

The lifetime is visible.

---

# 13. Anti-Pattern — Hidden Singleton

Example:

```js
const cache = new Map();

export function getData(key) {
  return cache.get(key);
}
```

The singleton may silently become:

```text
shared mutable state
unbounded memory
cross-tenant leakage
stale data
hard-to-test behavior
```

A singleton is not automatically wrong.

The issue is undocumented scope and lifecycle.

---

# 14. Anti-Pattern — Mutable Export

```js
export const config = {};

config.timeout = 5000;
```

Every module importing `config` can observe mutation.

Better:

```js
export const config =
  Object.freeze({
    timeout: 5000
  });
```

or provide explicit configuration construction.

---

# 15. Anti-Pattern — Temporal Coupling

Code:

```js
initialize();

useSystem();
```

but `useSystem()` only works after `initialize()`.

The API has an implicit sequencing contract.

Better:

```js
const system =
  createSystem();

system.use();
```

or encapsulate initialization.

---

# 16. Anti-Pattern — Hidden Initialization Side Effects

```js
import "./register.js";
```

can be valid.

But if a package requires consumers to know that:

```text
import order
```

controls application correctness, the module contract may be too implicit.

Make initialization boundaries explicit where practical.

---

# 17. Anti-Pattern — Mutation Through Aliases

```js
const config = {
  retries: 3
};

const copy = config;

copy.retries = 10;
```

The original changed.

This is not a JavaScript bug.

It is aliasing.

Use immutable values or controlled ownership when shared mutation creates risk.

---

# 18. Anti-Pattern — Shared Mutable Configuration

```js
config.timeout = 1000;
config.timeout = 5000;
config.timeout = 100;
```

Different subsystems now depend on timing.

Prefer:

```js
const client =
  createClient({
    timeout: 1000
  });
```

Configuration should usually be fixed for a component lifetime.

---

# 19. Category II — Function / Scope Anti-Patterns

Common failures:

```text
wrong closure
wrong this
accidental globals
argument mutation
overloaded functions
```

---

# 20. Anti-Pattern — Arrow Everywhere

Replacing every function with an arrow can change:

```text
this
arguments
constructability
prototype
```

Example:

```js
const obj = {
  value: 10,

  get: () => this.value
};
```

This is not equivalent to a method.

---

# 21. Anti-Pattern — Dynamic `this` Reliance

If a function is passed around:

```js
const fn = obj.method;
```

the original receiver can be lost.

If `this` is central to the API, document or bind the receiver.

Or use closures/dependency injection where appropriate.

---

# 22. Anti-Pattern — Callback Context Assumptions

Legacy:

```js
button.addEventListener(
  "click",
  obj.handle
);
```

If `obj.handle` relies on:

```js
this.value
```

the listener's invocation semantics matter.

Safer:

```js
button.addEventListener(
  "click",
  event => obj.handle(event)
);
```

or bind deliberately when appropriate.

---

# 23. Anti-Pattern — `var` in Async Loops

```js
for (var i = 0; i < 3; i++) {
  setTimeout(
    () => console.log(i),
    0
  );
}
```

The shared function-scoped binding creates surprising output.

Prefer iteration bindings or explicit capture.

---

# 24. Anti-Pattern — Excessive Closure Capture

A callback may accidentally retain a huge object:

```js
const huge = loadHugeObject();

return () => {
  console.log(smallValue);
};
```

If the closure retains references to the surrounding scope, a long-lived callback can extend lifetimes unexpectedly.

Use lifetime analysis.

---

# 25. Anti-Pattern — Function With Too Many Roles

Example:

```js
async function processOrder() {
  validate();
  authenticate();
  fetchUser();
  calculateTax();
  chargeCard();
  saveOrder();
  sendEmail();
  log();
}
```

This becomes:

```text
hard to test
hard to retry
hard to roll back
hard to observe
hard to reason about
```

Split by meaningful boundaries.

---

# 26. Anti-Pattern — Boolean Parameter Explosion

Bad:

```js
createUser(
  data,
  true,
  false,
  true,
  false
);
```

Call-site semantics are opaque.

Prefer options:

```js
createUser(data, {
  sendWelcomeEmail: true,
  verify: false
});
```

---

# 27. Anti-Pattern — Boolean Flag API Growth

Eventually:

```js
run(
  input,
  true,
  false,
  true,
  false,
  true
);
```

The function becomes an implicit state machine.

Use named options or separate operations.

---

# 28. Anti-Pattern — Any-Value Utility Function

```js
function process(value) {
  // handles string, number,
  // array, object, null,
  // Promise, function...
}
```

Universal utilities tend to hide domain rules.

Prefer narrow contracts.

---

# 29. Category III — Coercion Anti-Patterns

JavaScript's coercion is powerful and sometimes useful.

The failure is relying on coercion where the domain requires explicitness.

---

# 30. Anti-Pattern — `==` Everywhere

```js
if (value == expected) {
  ...
}
```

Implicit coercion can hide bugs.

But:

```js
value == null
```

can intentionally test:

```text
null
or
undefined
```

Therefore the review question is:

> Is the coercion intentional and documented?

---

# 31. Anti-Pattern — `||` for Defaulting

Bad when zero/empty/false are valid:

```js
const retries =
  input.retries || 3;
```

If:

```js
input.retries = 0;
```

result becomes:

```text
3
```

Use:

```js
input.retries ?? 3
```

when nullish semantics are intended.

---

# 32. Anti-Pattern — Truthiness as Validation

Bad:

```js
if (!value) {
  throw new Error("Invalid");
}
```

This rejects:

```text
0
false
""
```

which may be valid.

Validate by domain.

---

# 33. Anti-Pattern — Stringly-Typed State

```js
status = "pending";
status = "in_progress";
status = "complete";
status = "done";
```

Different names can represent the same conceptual state.

Use:

```text
controlled enum-like values
central constants
schemas
state machines
```

where the domain requires it.

---

# 34. Anti-Pattern — Magic String Protocols

```js
if (message.type === "USER_DELETED") {}
```

Repeated strings create typo risk.

Centralize protocol constants or schema validation.

---

# 35. Category IV — Object / Prototype Anti-Patterns

Common issues:

```text
prototype pollution
prototype mutation
incorrect inheritance
descriptor assumptions
dictionary misuse
```

---

# 36. Anti-Pattern — Object as Untrusted Dictionary

```js
const map = {};
map[userKey] = value;
```

This can collide with special object keys and inherited properties.

Use:

```js
const map = new Map();
```

or:

```js
Object.create(null)
```

when appropriate.

---

# 37. Anti-Pattern — `for...in` Without Ownership Check

```js
for (const key in object) {
  use(object[key]);
}
```

Inherited enumerable properties can appear.

Use:

```js
Object.keys(object)
```

or:

```js
Object.entries(object)
```

for own enumerable string properties as appropriate.

---

# 38. Anti-Pattern — Global Prototype Extension

Bad:

```js
Array.prototype.sum = function () {};
```

Problems:

```text
collision
enumeration
shared global mutation
dependency interaction
future standard collision
```

---

# 39. Anti-Pattern — Prototype Mutation at Runtime

```js
SomeType.prototype.foo = foo;
```

after instances are already used can create dynamic shape changes and hidden plugin coupling.

Use stable definitions where possible.

---

# 40. Anti-Pattern — `Object.setPrototypeOf()` in Hot Paths

Changing prototypes dynamically can impair optimization and complicate object reasoning.

Prefer:

```text
create correct object shape once
```

instead of frequent prototype mutation.

Measure before making performance claims.

---

# 41. Anti-Pattern — Inheritance for Reuse

Example:

```text
class ReportService extends StringUtils
```

just to reuse one method.

This creates a false “is-a” relationship.

Prefer composition.

---

# 42. Anti-Pattern — Deep Inheritance Trees

```text
A
 ↓
B
 ↓
C
 ↓
D
 ↓
E
```

Behavior becomes difficult to predict.

Prefer:

```text
small interfaces
composition
delegation
```

when inheritance does not express real subtype semantics.

---

# 43. Anti-Pattern — Class for Everything

JavaScript supports many useful styles:

```text
functions
closures
objects
composition
classes
modules
```

Do not introduce classes when a pure function is clearer.

---

# 44. Category V — Promise / Async Anti-Patterns

Async failures are often caused by wrong mental models.

---

# 45. Anti-Pattern — `async` Without Await

```js
async function add(a, b) {
  return a + b;
}
```

This is valid.

It is not automatically an anti-pattern.

The issue is when `async` is added purely for style and changes the API contract unnecessarily.

---

# 46. Anti-Pattern — `new Promise(async ...)`

Bad:

```js
new Promise(async (resolve, reject) => {
  const value = await work();
  resolve(value);
});
```

The Promise constructor already models asynchronous completion.

The async executor introduces confusing error and control-flow behavior.

Prefer:

```js
async function run() {
  return work();
}
```

---

# 47. Anti-Pattern — Sequential Await in Independent Work

Bad:

```js
const a = await fetchA();
const b = await fetchB();
const c = await fetchC();
```

when all three are independent.

Potentially better:

```js
const [a, b, c] =
  await Promise.all([
    fetchA(),
    fetchB(),
    fetchC()
  ]);
```

Only do this when the operations are actually independent.

---

# 48. Anti-Pattern — `Promise.all` for Huge Unbounded Work

Bad:

```js
await Promise.all(
  items.map(process)
);
```

for millions of tasks.

This can create:

```text
memory pressure
concurrency explosion
downstream overload
```

Use bounded concurrency.

---

# 49. Anti-Pattern — Fire-and-Forget Without Ownership

```js
void sendEmail();
return response;
```

The caller may not know:

```text
whether it succeeded
who retries
who observes failures
```

Fire-and-forget is acceptable only when ownership and failure semantics are explicit.

---

# 50. Anti-Pattern — Ignored Promise

```js
save();
```

If rejection matters, this can create unhandled failure.

Make intent explicit:

```js
await save();
```

or:

```js
void save().catch(report);
```

---

# 51. Anti-Pattern — Catch and Ignore

```js
try {
  await work();
} catch {
}
```

This converts an error into silence.

At minimum:

```text
classify
log/observe
recover
rethrow
```

according to the contract.

---

# 52. Anti-Pattern — Catch Everything and Return Null

```js
try {
  return await work();
} catch {
  return null;
}
```

This collapses:

```text
not found
timeout
permission denied
programmer bug
dependency outage
```

into one ambiguous value.

---

# 53. Anti-Pattern — Promise Pyramid

```js
doA()
  .then(a =>
    doB(a)
      .then(b =>
        doC(b)
      )
  );
```

Nested Promise chains often obscure control flow.

Flatten:

```js
doA()
  .then(a => doB(a))
  .then(b => doC(b));
```

or use `async`/`await`.

---

# 54. Anti-Pattern — Mixing Callback and Promise Control Flow

```js
return doWork()
  .then(value => callback(null, value));
```

while also:

```js
callback(...)
```

elsewhere.

This can produce:

```text
double completion
unhandled rejection
timing bugs
```

Choose one completion model at the boundary.

---

# 55. Anti-Pattern — Resolve and Then Throw

```js
return new Promise((resolve, reject) => {
  resolve(value);
  throw new Error("late");
});
```

The Promise is already settled.

The later throw does not become a rejection of that Promise in the way developers often expect.

Control flow should be straightforward.

---

# 56. Anti-Pattern — Nested `try/catch` for Normal Control Flow

Exceptions should not normally implement ordinary branching.

Use:

```text
result
status
validation
```

for expected alternatives when clearer.

---

# 57. Anti-Pattern — Retry Everything

```js
for (let i = 0; i < 10; i++) {
  try {
    return await operation();
  } catch {}
}
```

This can retry:

```text
invalid input
auth failure
duplicate write
programming error
```

Classify errors.

---

# 58. Anti-Pattern — Retry Without Backoff

Immediate retries amplify outages.

Use bounded:

```text
backoff
jitter
attempt limit
deadline
```

where retry is appropriate.

---

# 59. Anti-Pattern — Retry Without Idempotency

Retrying:

```text
charge
create order
send message
```

without idempotency can duplicate side effects.

---

# 60. Anti-Pattern — Timeout Without Cancellation

```js
await Promise.race([
  work(),
  timeout()
]);
```

The timeout may reject while `work()` continues.

Use actual cancellation when supported.

---

# 61. Anti-Pattern — `Promise.race` as Cancellation

`Promise.race` chooses a settlement result.

It does not automatically stop the losing operation.

This is a crucial semantic distinction.

---

# 62. Category VI — Event Anti-Patterns

---

# 63. Anti-Pattern — Listener Leak

```js
element.addEventListener(
  "click",
  handler
);
```

repeatedly without removal.

The result can be:

```text
duplicate work
memory retention
increasing latency
```

Use lifecycle-managed registration/removal.

---

# 64. Anti-Pattern — Anonymous Listener You Cannot Remove

```js
element.addEventListener(
  "click",
  () => handle()
);
```

If later removal is required, the function identity is unavailable.

Use a named/reference-held handler or `AbortSignal` where appropriate.

---

# 65. Anti-Pattern — Event Bus Everywhere

A giant:

```text
global EventEmitter
```

can become hidden dependency injection.

Problems:

```text
who emits?
who listens?
when?
ordering?
ownership?
cleanup?
```

Use explicit domain boundaries where possible.

---

# 66. Anti-Pattern — Event Storm

One state change emits:

```text
event A
→ B
→ C
→ D
→ E
```

Each triggers more updates.

Possible outcomes:

```text
loops
duplicate work
latency spikes
hard debugging
```

Model causal dependencies explicitly.

---

# 67. Anti-Pattern — DOM Event Delegation Without Boundaries

Event delegation can be good.

But one listener on a huge root can accumulate unrelated concerns.

Keep delegated handlers scoped to meaningful components.

---

# 68. Anti-Pattern — Synchronous Heavy Event Handler

```js
button.onclick = () => {
  hugeComputation();
};
```

This blocks the UI thread.

Move appropriate CPU work to:

```text
Worker
Wasm Worker
incremental tasks
```

when necessary.

---

# 69. Category VII — Memory Anti-Patterns

---

# 70. Anti-Pattern — Unbounded Cache

```js
const cache = new Map();

function remember(key, value) {
  cache.set(key, value);
}
```

If keys never expire:

```text
memory grows forever
```

Add:

```text
TTL
size bound
LRU
invalidation
weak ownership where appropriate
```

---

# 71. Anti-Pattern — Cache as Source of Truth

A cache should not accidentally become authoritative business state.

If the process restarts, the application must remain correct where the cache is not durable state.

---

# 72. Anti-Pattern — Large Closure Retention

A long-lived callback can accidentally keep a large graph reachable.

Review:

```text
event handlers
timers
subscriptions
promises
global callbacks
```

for retained references.

---

# 73. Anti-Pattern — Timer Retention

```js
setInterval(() => {
  use(largeObject);
}, 1000);
```

If the interval is never cleared, `largeObject` can remain reachable.

Timers need lifecycle ownership.

---

# 74. Anti-Pattern — DOM Retention

Holding references to removed DOM trees can keep data alive.

Example:

```js
const oldNode = document.querySelector(...);
```

A long-lived module retaining `oldNode` may keep associated structures alive.

Nulling variables is not a universal fix; eliminate ownership paths.

---

# 75. Anti-Pattern — Rebuilding Huge Objects Per Request

```js
return rows.map(row =>
  deepClone(row)
);
```

can create unnecessary allocation pressure.

Measure before optimizing, but inspect whether immutable transformation is truly required.

---

# 76. Anti-Pattern — JSON Deep Clone

```js
const clone =
  JSON.parse(
    JSON.stringify(value)
  );
```

Problems:

```text
undefined
BigInt
Date
Map
Set
cycles
custom prototypes
special values
```

Do not use it as a universal clone.

---

# 77. Anti-Pattern — `structuredClone` Everywhere

`structuredClone()` is more capable than JSON cloning but still not necessarily the correct data-model solution.

Do not clone large graphs merely to avoid reasoning about ownership.

---

# 78. Category VIII — Performance Anti-Patterns

---

# 79. Anti-Pattern — Optimize Without Measurement

```text
I heard this is faster.
```

is not a performance argument.

Use:

```text
benchmark
profile
production telemetry
```

---

# 80. Anti-Pattern — Micro-Optimization Before Architecture

Optimizing:

```js
for loop
vs
forEach
```

while the system spends:

```text
500 ms in network
```

is wasted effort.

Optimize the dominant cost.

---

# 81. Anti-Pattern — Excessive Abstraction in Hot Paths

Layers such as:

```text
mapper
wrapper
proxy
adapter
decorator
validator
serializer
```

can add overhead in very hot paths.

Do not remove abstractions blindly.

Measure the actual hot path.

---

# 82. Anti-Pattern — Premature Object Pooling

Manual object pools can introduce:

```text
complexity
bugs
lifetime hazards
```

before the allocation cost has been proven significant.

Use pooling only with evidence.

---

# 83. Anti-Pattern — Dynamic Object Shape Churn

Creating objects with inconsistent property layouts:

```js
const a = { x: 1, y: 2 };
const b = { y: 2, x: 1, z: 3 };
const c = { x: 1 };
```

can make optimization harder in some engines.

But do not turn this into a rigid coding rule.

Use stable structures where hot-path profiling shows benefit.

---

# 84. Anti-Pattern — Premature `Object.freeze` Everywhere

Freezing can be useful for contracts but can also:

```text
add overhead in some paths
complicate mutation-based APIs
```

Use it for semantic guarantees where appropriate.

---

# 85. Anti-Pattern — Regex for Parsing Everything

Complex grammars should not always be parsed with one massive regular expression.

Problems:

```text
readability
catastrophic backtracking
maintenance
error reporting
```

Use a parser when the domain is actually a language.

---

# 86. Anti-Pattern — Repeated Parsing in Loops

```js
for (const row of rows) {
  const value =
    JSON.parse(row.payload);
}
```

If parsing can safely occur once earlier, move it to a boundary.

---

# 87. Anti-Pattern — Logging Huge Objects

```js
console.log(hugeObject);
```

can trigger:

```text
serialization
inspection
memory retention
PII exposure
```

Log structured summaries.

---

# 88. Category IX — Security Anti-Patterns

---

# 89. Anti-Pattern — `eval` for Configuration

```js
const config =
  eval(input);
```

Never turn untrusted configuration into code.

Use:

```text
JSON
schema
validated configuration
```

---

# 90. Anti-Pattern — `innerHTML` With Untrusted Data

```js
container.innerHTML =
  userProvidedValue;
```

This can enable XSS.

Use:

```text
textContent
safe DOM APIs
sanitization
```

according to the context.

---

# 91. Anti-Pattern — Unsafe Dynamic URLs

```js
location.href =
  "/search?q=" + userInput;
```

or:

```js
fetch(userInput);
```

can create security problems.

Use correct URL construction and allowlists.

---

# 92. Anti-Pattern — Trusting Client Validation

```js
if (form.amount > 0) {
  fetch("/pay", ...);
}
```

Server must still validate:

```text
amount
authorization
currency
tenant
state
```

Client validation is UX, not the security boundary.

---

# 93. Anti-Pattern — Authorization by ID

```js
if (request.userId === request.params.userId) {
  allow();
}
```

Identity equality is not sufficient authorization.

Check:

```text
resource ownership
role
permission
tenant
state
```

---

# 94. Anti-Pattern — Prototype-Pollution-Prone Deep Merge

Dangerous recursive merge utilities can allow attacker-controlled keys to modify object prototypes.

Use:

```text
safe schema validation
allowlists
safe merge semantics
```

and test security boundaries.

---

# 95. Anti-Pattern — Secret in Bundle

```js
const API_KEY = "secret";
```

Anything shipped to an untrusted client should be assumed recoverable.

Keep sensitive secrets server-side.

---

# 96. Anti-Pattern — Logging Secrets

Never log:

```text
tokens
passwords
API keys
private keys
session cookies
```

Even debug logs can become permanent production data.

---

# 97. Anti-Pattern — Security Fallback Downgrade

Example:

```text
secure API unavailable
→ use insecure alternative
```

This can be worse than failing.

For security-critical operations:

```text
unsupported
→ fail closed
```

when the contract requires strong security.

---

# 98. Category X — Module / Dependency Anti-Patterns

---

# 99. Anti-Pattern — Circular Module Web

```text
A → B → C → A → D → A
```

A large cycle graph increases initialization ambiguity.

Break cycles with:

```text
dependency inversion
interfaces
events where appropriate
factories
```

but do not replace one hidden dependency with a global event bus.

---

# 100. Anti-Pattern — Importing Everything

```js
import * as utils from "huge-library";
```

may enlarge bundles and obscure actual dependencies.

Prefer targeted imports where the package supports them.

---

# 101. Anti-Pattern — Dependency for Tiny Functionality

Installing a large library for:

```text
one helper
```

can add:

```text
security surface
bundle size
startup
updates
```

But do not reimplement complex, security-sensitive logic merely to remove a dependency.

---

# 102. Anti-Pattern — Dependency Duplication

Multiple versions of the same major library can increase:

```text
bundle
memory
bug surface
```

Investigate whether versions can be deduplicated safely.

---

# 103. Anti-Pattern — Floating Dependency Without Locking

Production builds should be reproducible.

Use appropriate:

```text
lockfiles
version policy
artifact capture
```

for applications.

---

# 104. Anti-Pattern — Ignoring Engine Constraints

A dependency may require:

```text
newer Node
newer browser
native API
```

Ignoring `engines` and compatibility metadata can produce production failures.

---

# 105. Anti-Pattern — Vendoring Without Ownership

Copying external code into a repository creates:

```text
security update responsibility
license responsibility
version drift
```

Vendor intentionally, with ownership.

---

# 106. Category XI — API Design Anti-Patterns

---

# 107. Anti-Pattern — Generic `string` Timestamp

```ts
createdAt: string;
date: string;
```

This hides:

```text
Instant
PlainDate
PlainDateTime
ZonedDateTime
```

Define semantics.

Chapter 92 provides the Temporal model.

---

# 108. Anti-Pattern — Generic Options Object

```js
doThing({
  a,
  b,
  c,
  d,
  e,
  f,
  g,
  h
});
```

Options objects are useful.

The anti-pattern is an API with dozens of interacting flags and undocumented combinations.

Use explicit domain operations or validated schemas.

---

# 109. Anti-Pattern — God API

```js
service.execute(options)
```

with every behavior encoded inside `options`.

This can hide a dozen different operations behind one function.

Prefer cohesive operations.

---

# 110. Anti-Pattern — Boolean Return for Rich Outcomes

```js
return true;
```

for:

```text
created
updated
already existed
rejected
not authorized
```

collapses important semantics.

Return structured outcomes when the caller needs them.

---

# 111. Anti-Pattern — Throwing for Expected Branches

Do not use exceptions as normal parsing:

```js
try {
  JSON.parse(input);
} catch {
  // expected validation
}
```

This can be appropriate at a boundary.

But if invalid input is routine and performance-critical, a parser/result model may be more appropriate.

---

# 112. Anti-Pattern — Inconsistent Error Contracts

One method:

```js
throw Error
```

another:

```js
return null
```

another:

```js
return { error: ... }
```

Callers cannot reason consistently.

Define the contract.

---

# 113. Category XII — Reliability Anti-Patterns

---

# 114. Anti-Pattern — No Timeout

Any network call without a bounded lifecycle can hang longer than the business request permits.

Use deadlines/timeouts.

---

# 115. Anti-Pattern — Timeout Without Cleanup

A timed-out operation may keep running.

Cancel where possible and ensure resources are released.

---

# 116. Anti-Pattern — Retry Amplification

Layers:

```text
client retries × API retries × SDK retries × queue retries
```

can multiply attempts.

Define retry ownership.

---

# 117. Anti-Pattern — No Backpressure

A producer can overwhelm a consumer:

```text
requests
→ tasks
→ queue
→ DB
```

without bounded concurrency.

Use:

```text
queues
limits
backpressure
load shedding
```

---

# 118. Anti-Pattern — No Idempotency

At-least-once execution without idempotent side effects is a reliability trap.

---

# 119. Anti-Pattern — One Huge Transaction

Trying to keep a transaction open while doing:

```text
network
email
file
queue
```

creates long lock duration and failure complexity.

Keep transaction boundaries tight.

---

# 120. Anti-Pattern — Distributed State Without Ownership

Multiple services update the same field independently.

No clear source of truth.

Result:

```text
lost updates
inconsistent state
race conditions
```

Define ownership.

---

# 121. Anti-Pattern — “Eventually Consistent” Without Definition

Saying:

```text
it will become consistent
```

without defining:

```text
when
which reads
which region
which SLA
```

is not a consistency strategy.

---

# 122. Category XIII — Observability Anti-Patterns

---

# 123. Anti-Pattern — Logs Without Context

```text
Error occurred
```

is nearly useless.

Include:

```text
request ID
operation
service
version
safe identifiers
error category
```

---

# 124. Anti-Pattern — Logs as Metrics

Parsing logs for:

```text
count
latency
success
```

is a poor substitute for structured metrics.

Use dedicated metrics.

---

# 125. Anti-Pattern — High-Cardinality Metrics

Do not label metrics with:

```text
user ID
request ID
full URL
random values
```

This can overload metric systems.

---

# 126. Anti-Pattern — No Correlation IDs

Distributed systems become difficult to diagnose without:

```text
trace ID
request ID
job ID
```

where appropriate.

---

# 127. Anti-Pattern — Monitoring Symptoms Only

Monitoring:

```text
CPU
```

without:

```text
latency
error rate
business outcomes
```

can miss major incidents.

Observe the service contract.

---

# 128. Category XIV — Testing Anti-Patterns

---

# 129. Anti-Pattern — Testing Implementation Instead of Contract

```js
expect(obj._privateCache.size)
  .toBe(1);
```

This can block valid refactors.

Prefer testing externally meaningful behavior.

---

# 130. Anti-Pattern — Snapshot Everything

Snapshots are useful.

But enormous snapshots can:

```text
hide meaningful changes
create noisy diffs
discourage review
```

Snapshot intentionally.

---

# 131. Anti-Pattern — Over-Mocking

Mocking every dependency can produce tests that prove:

```text
the mocks behave as expected
```

rather than:

```text
the system works
```

Use integration tests at important boundaries.

---

# 132. Anti-Pattern — Under-Testing Failure

A system tested only on:

```text
success path
```

is not reliable.

Test:

```text
timeout
retry
duplicate
malformed
partial failure
cancellation
```

---

# 133. Anti-Pattern — Non-Deterministic Tests

Tests that depend on:

```text
current time
randomness
real network
global mutable state
parallel ordering
```

can flake.

Control nondeterminism.

---

# 134. Anti-Pattern — Sleep in Tests

```js
await delay(1000);
expect(...);
```

is brittle.

Use:

```text
fake timers
explicit synchronization
poll-with-deadline where necessary
event completion
```

---

# 135. Anti-Pattern — One Giant Integration Test

Large tests with hundreds of steps are difficult to diagnose.

Use layered tests:

```text
unit
contract
integration
end-to-end
```

with a clear purpose.

---

# 136. Category XV — Debugging Anti-Patterns

---

# 137. Anti-Pattern — Randomly Editing Code

Symptoms:

```text
change
run
change
run
```

without a hypothesis.

Use:

```text
observe
hypothesize
experiment
verify
```

---

# 138. Anti-Pattern — Debugging the Last Change Only

A failure may originate much earlier.

Trace:

```text
bad output
← transformation
← input
← source
```

---

# 139. Anti-Pattern — Logging Everything

Huge logs can hide the signal and leak secrets.

Log targeted state.

---

# 140. Anti-Pattern — Console.log as Permanent Observability

`console.log()` is useful during diagnosis.

Production observability needs:

```text
structured
searchable
correlated
redacted
measured
```

telemetry.

---

# 141. Anti-Pattern — Guessing About the Event Loop

Do not reason:

```text
promise first because promises are faster
```

without understanding:

```text
job queue
host task queue
microtasks
timers
I/O
```

Reproduce with a minimal program.

---

# 142. Anti-Pattern — Blaming the Runtime

```text
Node is slow
Safari is broken
V8 changed everything
```

without a reproducible case.

First identify:

```text
input
runtime version
host
workload
measurement
```

---

# 143. Category XVI — Architecture Anti-Patterns

---

# 144. Anti-Pattern — God Object

```js
class Application {
  // everything
}
```

It becomes a hidden dependency graph.

Split by domain responsibility.

---

# 145. Anti-Pattern — God Module

One module exports:

```text
auth
db
logging
HTTP
cache
payments
```

Consumers become coupled to everything.

Prefer cohesive module boundaries.

---

# 146. Anti-Pattern — Shared Utility Dump

```text
utils.js
```

with 500 unrelated functions.

Utility modules should have coherent semantics.

---

# 147. Anti-Pattern — Abstraction for One Caller

Building:

```text
generic framework
```

for one current use case can add more maintenance than value.

Abstract after repeated stable patterns.

---

# 148. Anti-Pattern — Abstraction Too Early

Two similar functions:

```text
A
B
```

are not always the same concept.

Duplicated code can be cheaper until the domain pattern stabilizes.

---

# 149. Anti-Pattern — Abstraction Too Late

At the opposite extreme:

```text
20 copies
```

of the same policy produce drift.

Refactor when repeated behavior has stable semantics.

---

# 150. Anti-Pattern — Leaky Abstraction

A “repository” still exposes:

```text
SQL fragments
driver connection
transaction object
```

everywhere.

The abstraction is not actually isolating the dependency.

---

# 151. Anti-Pattern — Framework-Centric Domain

Business logic depends directly on:

```text
HTTP request
ORM
framework decorators
provider bindings
```

This makes the domain hard to reuse and test.

Keep domain semantics separate where valuable.

---

# 152. Anti-Pattern — Architecture by Buzzword

```text
microservices
event-driven
serverless
Kubernetes
Wasm
AI
```

without demonstrating why the architecture solves the actual problem.

Technology should follow constraints.

---

# 153. Anti-Pattern — Microservices for Team Size One

Splitting everything into services can create:

```text
network overhead
deployment burden
observability complexity
distributed failures
```

before the organization needs it.

---

# 154. Anti-Pattern — Distributed Monolith

Many services:

```text
A must call B
B must call C
C must call D
D must call A
```

This is a monolith with network failure modes.

Services should have meaningful ownership boundaries.

---

# 155. Anti-Pattern — Shared Database Across Services

If every service writes every table, service boundaries become fictional.

Prefer:

```text
clear ownership
published interfaces
events
read models
```

where justified.

---

# 156. Category XVII — Serverless / Edge Failure Modes

---

# 157. Anti-Pattern — Memory as Durable State

```js
const sessions = new Map();
```

This is not a distributed source of truth.

Use durable storage.

---

# 158. Anti-Pattern — Assuming Warm Reuse

Warm reuse is a performance opportunity, not correctness.

---

# 159. Anti-Pattern — Database Connection Explosion

```text
many invocations
×
many connections
=
database saturation
```

Use:

```text
pooler
proxy
serverless driver
connection strategy
```

---

# 160. Anti-Pattern — Edge Compute + Central Database for Everything

Edge code can still wait on a distant database.

Optimize the data path.

---

# 161. Anti-Pattern — Global Cache Without Tenant Isolation

Caching user-specific output globally can become a data leak.

Cache keys and response semantics must include the right dimensions.

---

# 162. Anti-Pattern — Background Promise After Response

Do not assume arbitrary work survives the invocation lifecycle.

Use:

```text
queue
workflow
documented continuation
```

---

# 163. Anti-Pattern — Provider-Specific Code Everywhere

```text
env.KV
env.QUEUE
env.DB
```

inside the domain can make migration expensive.

Use adapters when portability has real value.

---

# 164. Anti-Pattern — Portability Theater

Creating abstractions for five cloud providers when the company will operate only one can produce unnecessary complexity.

Portability is a business decision.

---

# 165. Category XVIII — WebAssembly / FFI Failure Modes

---

# 166. Anti-Pattern — Tiny Wasm Calls

```js
for (const x of values) {
  wasm.process(x);
}
```

Boundary overhead can dominate.

Batch data.

---

# 167. Anti-Pattern — JSON for Huge High-Frequency FFI

```text
JS object
→ JSON
→ Wasm
→ JSON
→ JS object
```

Serialization can erase the performance benefit.

Use binary/buffer-oriented interfaces when justified.

---

# 168. Anti-Pattern — Undefined Memory Ownership

If nobody knows who owns:

```text
ptr
```

you can get:

```text
use-after-free
leak
double free
```

Document ownership.

---

# 169. Anti-Pattern — Raw ABI Everywhere

Do not expose:

```text
ptr
len
malloc
free
```

to every application module.

Centralize ABI details in a wrapper.

---

# 170. Anti-Pattern — Wasm Without Benchmark

“Compiled native code” is not a performance measurement.

Benchmark:

```text
JS
boundary
Wasm
copies
```

as a whole.

---

# 171. Category XIX — Compatibility Failure Modes

---

# 172. Anti-Pattern — Stage Number as Production Approval

```text
Stage 3
→ production
```

is not a valid deployment policy.

Standards maturity and fleet support are different dimensions.

---

# 173. Anti-Pattern — Browser Name Instead of Capability

```js
if (isSafari()) ...
```

when the real requirement is:

```text
supports feature X
```

Prefer capability detection.

---

# 174. Anti-Pattern — Polyfill Everything

Global polyfill bundles can add:

```text
startup
bundle size
global mutation
maintenance
```

Use target-based compatibility.

---

# 175. Anti-Pattern — Supporting Forever

A compatibility fallback with no exit criteria becomes permanent complexity.

Record:

```text
owner
target versions
removal condition
```

---

# 176. Category XX — API / Data Modeling Failure Modes

---

# 177. Anti-Pattern — Ambiguous Date Strings

```json
{
  "date": "09/10/2026"
}
```

This is ambiguous across locales.

Use an explicit wire contract.

---

# 178. Anti-Pattern — Everything as UTC Timestamp

A birthday is not necessarily an instant.

A store opening time is not necessarily an instant.

Use semantic temporal types.

---

# 179. Anti-Pattern — Everything as Local Time

A payment event needs an exact timestamp.

Local time without zone/offset may be ambiguous.

---

# 180. Anti-Pattern — Floating-Point Currency

```js
0.1 + 0.2
```

is not exactly `0.3` under binary floating-point semantics.

Use:

```text
minor units
decimal library
exact domain model
```

where financial correctness requires it.

---

# 181. Anti-Pattern — Mixing Units

```js
timeout = 5;
```

Is that:

```text
seconds?
milliseconds?
minutes?
```

Make units explicit.

---

# 182. Anti-Pattern — Generic Number Field

A number can mean:

```text
price
count
milliseconds
percentage
ratio
ID
```

Use domain naming/validation.

---

# 183. Category XXI — API Boundary Failures

---

# 184. Anti-Pattern — Validate Deep Inside

If invalid data crosses many layers before validation, failure becomes expensive.

Validate at boundaries.

---

# 185. Anti-Pattern — Trusting Internal Assumptions at Boundaries

Internal code may assume:

```text
non-null
authorized
normalized
bounded
```

External inputs do not get that privilege.

---

# 186. Anti-Pattern — Schema Drift

Client and server silently disagree on:

```text
field names
types
nullability
meaning
```

Use versioned schemas/contracts.

---

# 187. Anti-Pattern — Breaking Change Without Versioning

Changing:

```text
error shape
field semantics
module exports
```

without consumer migration creates hidden breakage.

---

# 188. Anti-Pattern — Backward Compatibility With No Sunset

Keeping every old API forever creates:

```text
maintenance
test burden
security surface
```

Compatibility should have lifecycle.

---

# 189. Category XXII — Reliability / Distributed Failure

---

# 190. Anti-Pattern — Time-Based Coordination

```js
await delay(500);
assumeOtherServiceFinished();
```

Timing is not synchronization.

Use:

```text
ack
event
status
lease
transaction
```

depending on the system.

---

# 191. Anti-Pattern — Client-Side Ordering Assumption

```text
request A sent
request B sent
```

does not guarantee:

```text
A completes before B
```

Design explicit sequencing.

---

# 192. Anti-Pattern — Timestamp as Distributed Order

Timestamps can help correlate events.

They do not automatically provide causal ordering.

Use:

```text
sequence
logical clock
message offset
database order
```

where needed.

---

# 193. Anti-Pattern — No Duplicate Handling

Distributed systems retry.

Assume duplication is possible where the infrastructure contract permits it.

---

# 194. Anti-Pattern — Single Retry Layer Assumption

SDKs, clients, queues, and proxies can all retry.

Audit the full stack.

---

# 195. Category XXIII — Production Process Anti-Patterns

---

# 196. Anti-Pattern — “Works Locally”

Local success proves:

```text
one environment works
```

not:

```text
production compatibility
```

---

# 197. Anti-Pattern — No Reproducible Build

If the same commit produces different artifacts:

```text
debugging
security
rollback
```

become harder.

Pin dependencies/toolchains appropriately.

---

# 198. Anti-Pattern — Manual Production Hotfix Without Backport

A manual fix that exists only in production creates drift.

Backport to source and deployment artifacts.

---

# 199. Anti-Pattern — No Rollback

Every significant migration should answer:

```text
How do we stop?
How do we reverse?
What data changed?
```

---

# 200. Anti-Pattern — No Owner

If everyone owns a compatibility layer:

```text
nobody owns its removal
```

Assign ownership.

---

# 201. Principal Review Framework

For any suspicious pattern, evaluate:

```text
1. What local benefit does it provide?
2. What hidden assumption does it introduce?
3. What failure mode follows?
4. What is the blast radius?
5. How likely is failure?
6. How detectable is failure?
7. What is the replacement cost?
8. What is the migration risk?
9. Is the pattern intentionally justified?
10. What evidence would change the decision?
```

---

# 202. Smell → Cause → Failure → Repair

Use this template:

```text
Smell:
global mutable object

Cause:
hidden shared state

Failure:
race/test contamination

Repair:
explicit ownership

Verification:
concurrency test
```

This is better than merely saying:

```text
“global is bad.”
```

---

# 203. Anti-Pattern Severity

Classify:

```text
P0 — security/correctness critical
P1 — high production risk
P2 — maintainability/performance concern
P3 — cleanup opportunity
```

Severity should depend on context.

---

# 204. Risk Matrix

| Likelihood | Impact | Action |
|---|---|---|
| Low | Low | monitor |
| High | Low | fix when convenient |
| Low | High | add controls |
| High | High | immediate mitigation |

Use a more formal model for critical systems.

---

# 205. Repair Strategy

Possible actions:

```text
remove
replace
wrap
isolate
constrain
document
monitor
accept
```

Not every anti-pattern deserves a rewrite.

---

# 206. “Leave It Alone” Can Be Correct

A legacy pattern may be:

```text
stable
well-tested
low-risk
cheap
not worth changing
```

Principal engineering includes knowing when **not** to refactor.

---

# 207. Refactoring Trigger

Refactor when:

```text
failure cost
+
maintenance cost
+
security/performance risk
```

exceeds:

```text
migration risk
+
engineering cost
```

This is the practical economics of cleanup.

---

# 208. Anti-Pattern Removal Risk

Removing an anti-pattern can create a new failure.

Example:

```text
global cache
→ removed
→ database load spikes
```

Therefore the correct migration may be:

```text
measure
→ replace with bounded cache
→ load test
→ canary
→ remove old cache
```

---

# 209. Root Cause vs Cosmetic Fix

Incident:

```text
duplicate order
```

Cosmetic:

```text
add if statement
```

Root cause:

```text
non-idempotent retry path
```

Always ask:

> What mechanism allowed the failure?

---

# 210. Debugging Anti-Pattern in Reviews

Do not say:

```text
“this code smells.”
```

Say:

```text
“this shared mutable state can be observed concurrently, which can associate one request's user with another request. The current test suite does not exercise overlapping requests.”
```

Evidence beats vocabulary.

---

# 211. Production Review Checklist

```text
[ ] correctness assumptions explicit
[ ] state ownership explicit
[ ] concurrency considered
[ ] lifecycle explicit
[ ] errors classified
[ ] retries bounded
[ ] cancellation considered
[ ] security boundary defined
[ ] resource lifetime defined
[ ] compatibility verified
[ ] performance measured
[ ] observability sufficient
[ ] rollback possible
```

---

# 212. Implementation — Anti-Pattern Scanner

Build a tool that detects:

```text
eval
new Function
var
global mutable exports
setInterval
addEventListener without obvious cleanup
Promise.all over unbounded arrays
JSON deep clone
user-agent checks
innerHTML with suspicious data flow
hard-coded secrets
process.env assumptions
```

Do not blindly mark all matches as defects.

Produce:

```text
pattern
location
confidence
possible risk
manual review
```

---

# 213. Implementation — Heuristic Reviewer

Input:

```text
source file
```

Output:

```text
finding
confidence
reason
suggested investigation
```

Example:

```text
Finding:
unbounded Map

Confidence:
medium

Reason:
Map grows through request-derived key
No visible eviction

Investigation:
determine lifecycle and maximum cardinality
```

---

# 214. Implementation — Failure Reproduction

For every finding, build:

```text
minimal reproduction
```

Example:

```js
const cache = new Map();

for (let i = 0; i < 1_000_000; i++) {
  cache.set(String(i), i);
}
```

Then measure:

```text
memory
time
```

A concrete reproduction is better than a vague warning.

---

# 215. Implementation — Concurrency Test

Build tests that intentionally overlap requests:

```js
await Promise.all([
  requestAs("user-a"),
  requestAs("user-b")
]);
```

Then detect cross-request state contamination.

---

# 216. Implementation — Retry Test

Simulate:

```text
first attempt succeeds server-side
response lost
client retries
```

Verify idempotency.

---

# 217. Implementation — Cancellation Test

Start:

```text
long operation
```

then abort.

Verify:

```text
network canceled
resource released
worker stopped
temporary state cleaned
```

---

# 218. Implementation — Memory Leak Test

Create:

```text
register
remove
repeat
```

Use heap snapshots or runtime metrics.

Look for retained:

```text
listeners
timers
closures
DOM nodes
caches
```

---

# 219. Implementation — Compatibility Test

Run the same contract suite against:

```text
minimum runtime
current runtime
latest runtime
edge runtime where relevant
```

This converts compatibility assumptions into evidence.

---

# 220. Implementation — Security Test

Create test cases for:

```text
XSS
SSRF
prototype pollution
invalid URL
oversized input
malformed JSON
secret leakage
authorization bypass
```

---

# 221. Implementation — Production Review Tool

Build a report:

```text
file
finding
severity
confidence
category
owner
status
waiver
deadline
```

Support:

```text
open
accepted
fixed
false positive
waived
```

---

# 222. Debugging Exercises

## Exercise 1 — Shared State

```js
let currentUser;

async function handler(user) {
  currentUser = user;
  await delay();
  return currentUser;
}
```

Predict concurrent behavior.

---

## Exercise 2 — Promise Concurrency

```js
await a();
await b();
```

Question:

> Are these necessarily independent?

No. Only parallelize when dependency analysis proves independence.

---

## Exercise 3 — Promise.all

```js
await Promise.all(
  millionItems.map(work)
);
```

Question:

> What resource can explode?

Potentially:

```text
concurrency
memory
downstream load
```

---

## Exercise 4 — Timeout Race

```js
await Promise.race([
  work(),
  timeout(1000)
]);
```

Question:

> Does `work()` automatically stop?

No.

---

## Exercise 5 — Event Leak

Register an event handler every render.

Question:

> What happens if cleanup is missing?

Repeated handlers and retained references can accumulate.

---

# 223. Code Review Exercise

Review:

```js
async function process(items) {
  return Promise.all(
    items.map(async item => {
      try {
        return await expensive(item);
      } catch {
        return null;
      }
    })
  );
}
```

Possible issues:

```text
unbounded concurrency
errors collapsed into null
no retry policy
no cancellation
unclear partial-failure contract
```

The correct repair depends on the domain.

---

# 224. Code Review Exercise — Global Config

```js
export const settings = {};

export function configure(next) {
  Object.assign(settings, next);
}
```

Risks:

```text
global mutation
ordering
test contamination
concurrent reconfiguration
```

Ask whether runtime reconfiguration is genuinely required.

---

# 225. Code Review Exercise — Event Bus

```js
bus.emit("order-created", order);
```

Ask:

```text
who listens?
what order?
what failures?
what ownership?
what retry?
what transaction?
```

An event is an architecture boundary, not merely a function call.

---

# 226. Code Review Exercise — Generic Catch

```js
try {
  await operation();
} catch {
  return false;
}
```

Potentially collapses:

```text
timeout
auth error
bug
validation
dependency outage
```

into:

```text
false
```

Require an explicit error contract.

---

# 227. Interview Questions — Fundamentals

1. What is an anti-pattern?
2. How is it different from a bug?
3. How is it different from style?
4. Can an anti-pattern ever be justified?
5. What is global mutable state?
6. What is temporal coupling?
7. What is an abstraction leak?
8. Why is `||` risky for defaults?
9. Why is `Promise.all` dangerous for huge inputs?
10. Why is fire-and-forget risky?
11. Why can listeners leak?
12. Why is an unbounded cache dangerous?

---

# 228. Interview Questions — Senior

1. What are the most common JavaScript async anti-patterns?
2. How do you detect shared-state races?
3. How do you avoid retry storms?
4. How do you design cancellation?
5. How do you identify memory leaks?
6. Why is over-abstraction dangerous?
7. How do you review a global event bus?
8. How do you distinguish intentional legacy from technical debt?
9. How do you migrate away from a dangerous pattern safely?
10. What should an anti-pattern scanner report?

---

# 229. Interview Questions — Principal

1. How would you create an organization-wide JavaScript anti-pattern policy?
2. How would you prevent an anti-pattern list from becoming cargo cult?
3. How would you quantify technical risk?
4. How would you prioritize thousands of findings?
5. How would you design automated detection plus human review?
6. How would you prove that a refactor reduced incident risk?
7. When would you intentionally retain an anti-pattern?
8. How would you manage exceptions?
9. How would you connect anti-pattern remediation to reliability metrics?
10. How would you build a principal-level code-review framework?

---

# 230. Predict-the-Output Exercise

```js
const jobs = [];

for (var i = 0; i < 3; i++) {
  jobs.push(() => i);
}

console.log(
  jobs.map(fn => fn())
);
```

Expected:

```text
[3, 3, 3]
```

Explain:

```text
function-scoped binding
+
closure
```

---

# 231. Predict-the-Output Exercise

```js
const jobs = [];

for (let i = 0; i < 3; i++) {
  jobs.push(() => i);
}

console.log(
  jobs.map(fn => fn())
);
```

Expected:

```text
[0, 1, 2]
```

---

# 232. Predict-the-Output Exercise

```js
const value = 0;

console.log(value || 10);
console.log(value ?? 10);
```

Expected:

```text
10
0
```

---

# 233. Predict-the-Output Exercise

```js
async function demo() {
  try {
    return await Promise.reject(
      new Error("boom")
    );
  } catch {
    return "recovered";
  }
}

console.log(
  await demo()
);
```

Inside an async-capable context, the result is:

```text
"recovered"
```

The point is error handling, not a recommendation to catch everything.

---

# 234. Mastery Exercise — Anti-Pattern Catalog

Create your own catalog with:

```text
anti-pattern
context
local benefit
hidden cost
failure mode
detection
repair
evidence
severity
owner
```

Build at least:

```text
10 language-level
10 async
10 memory/performance
10 security
10 architecture
```

---

# 235. Mastery Exercise — Incident Mapping

Take five production incidents.

For each identify:

```text
symptom
anti-pattern
root cause
enabling assumption
missing control
repair
regression test
```

---

# 236. Mastery Exercise — Refactor Economics

Choose three anti-patterns.

Estimate:

```text
current maintenance cost
incident cost
security risk
migration cost
rollback cost
```

Then decide:

```text
fix now
fix later
accept
```

---

# 237. Mastery Exercise — Automated Detection

Build a static analyzer rule for:

```text
global mutable export
```

Then test:

```text
true positive
false positive
edge case
intentional exception
```

---

# 238. Mastery Exercise — Async Capacity

Take:

```js
await Promise.all(
  items.map(process)
);
```

Design bounded concurrency.

Then measure:

```text
throughput
latency
memory
downstream pressure
```

---

# 239. Mastery Exercise — Memory Leak

Create a reproducible leak caused by:

```text
event listener
timer
cache
closure
```

Then fix it and prove the retained graph decreases.

---

# 240. Mastery Exercise — Security

Pick:

```text
XSS
SSRF
prototype pollution
secret leakage
authorization bypass
```

Build:

```text
attack
root cause
mitigation
test
monitoring
```

---

# 241. Mastery Exercise — Architecture Review

Review:

```text
edge
→ serverless
→ queue
→ database
```

Find at least five possible anti-patterns.

For each, provide:

```text
trigger
impact
mitigation
```

---

# 242. Spaced Retrieval Schedule

### Day 0

Define:

```text
anti-pattern
bug
style
trade-off
```

### Day 1

Explain:

```text
shared state
temporal coupling
aliasing
```

### Day 3

Explain:

```text
Promise.all
fire-and-forget
retry
timeout
cancellation
```

### Day 7

Explain:

```text
listener leaks
unbounded caches
closure retention
```

### Day 14

Conduct a security anti-pattern review.

### Day 30

Review a production architecture.

### Day 60

Build an anti-pattern scanner.

### Day 90

Lead a principal-level risk review.

---

# 243. Retrieval Prompts

Answer without notes:

```text
What makes an anti-pattern?
Why isn't ugly code automatically an anti-pattern?
When can an anti-pattern be justified?
Why is global mutable state risky?
Why is temporal coupling dangerous?
Why can Promise.all overload a system?
Why doesn't Promise.race cancel work?
Why can retries duplicate side effects?
Why do listeners leak?
Why do unbounded caches hurt memory?
Why is JSON deep clone dangerous?
Why are user-agent checks fragile?
Why are stage numbers not production approval?
Why can Wasm FFI be slow?
Why can edge execution still be slow?
Why is portability expensive?
How do you prioritize anti-pattern remediation?
```

---

# 244. Dependency Graph

```text
Chapter 07 — Coercion
        ↓
Chapter 10/11/13/14 — Scope / Hoisting / Closures / this
        ↓
Chapter 15/17/18 — Objects / Prototypes / Classes
        ↓
Chapter 31–38 — Async / Promises / Cancellation / Streams
        ↓
Chapter 45/48 — Memory / Optimization
        ↓
Chapter 55–57 — Networking / Security
        ↓
Chapter 64–70 — Modules / Tooling
        ↓
Chapter 78–85 — Production / Reliability / Performance
        ↓
Chapter 88/89 — Debugging / Review
        ↓
Chapter 94 — Compatibility
        ↓
Chapter 95–97 — Legacy / Wasm / Edge
        ↓
Chapter 98 — Anti-Patterns and Failure Modes
```

---

# 245. Concept Connections

## Depends On

- JavaScript semantics.
- Runtime behavior.
- Async execution.
- Memory.
- Security.
- Modules.
- Networking.
- Compatibility.
- Performance.
- Production architecture.

## Builds Toward

- misconceptions.
- cost/trade-off analysis.
- real-world production scenarios.
- project architecture.
- principal-level engineering judgment.

## Related Concepts

- code smells.
- technical debt.
- failure modes.
- incident analysis.
- risk management.
- resilience.
- threat modeling.

## Concepts Revisited

- scope
- closures
- `this`
- promises
- event loop
- memory
- prototypes
- modules
- security
- compatibility
- WebAssembly
- edge/serverless

## Why This Chapter Matters Later

A strong JavaScript engineer can write correct code.

A principal engineer can recognize how local coding decisions become:

```text
systemic failure
```

and can decide when to fix them and when to leave them alone.

---

# 246. Principal Decision Framework

For every suspected anti-pattern:

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

Then ask:

```text
What local benefit does the pattern provide?
What hidden assumptions exist?
What is the likely failure mode?
What is the blast radius?
How detectable is failure?
What is the migration cost?
What is the rollback?
Is the pattern intentionally justified?
```

---

# 247. Production Checklist

```text
[ ] suspicious pattern identified
[ ] context understood
[ ] local benefit understood
[ ] hidden assumption identified
[ ] failure mode documented
[ ] severity assessed
[ ] evidence collected
[ ] replacement evaluated
[ ] migration risk assessed
[ ] tests added
[ ] observability checked
[ ] rollback defined
[ ] owner assigned
[ ] exception documented if retained
```

---

# 248. Anti-Pattern Review Record

```md
# Anti-Pattern Review

## Pattern
-

## Context
-

## Local Benefit
-

## Hidden Assumption
-

## Failure Mode
-

## Severity
-

## Evidence
-

## Options
- remove
- replace
- isolate
- constrain
- monitor
- accept

## Decision
-

## Owner
-

## Verification
-

## Removal / Review Date
-
```

---

# 249. Final Mental Model

```text
Pattern
 ↓
Assumption
 ↓
Context changes
 ↓
Assumption breaks
 ↓
Failure
 ↓
Impact
```

Repair:

```text
Observe
 ↓
Explain
 ↓
Classify
 ↓
Measure
 ↓
Prioritize
 ↓
Repair
 ↓
Verify
 ↓
Monitor
```

---

# 250. Final Principal Rule

> **Do not build a list of forbidden JavaScript patterns. Build a system for identifying assumptions, measuring consequences, and choosing the lowest-risk design for the actual context.**

The principal engineer's goal is not:

```text
zero anti-patterns
```

It is:

```text
acceptable risk
+
explicit trade-offs
+
observable behavior
+
maintainable architecture
```

---

# Chapter 98 — Canonical References and Source Discipline

Primary references:

1. **ECMAScript Specification**
   https://tc39.es/ecma262/

2. **MDN JavaScript Reference**
   https://developer.mozilla.org/en-US/docs/Web/JavaScript

3. **MDN Promise**
   https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise

4. **MDN EventTarget**
   https://developer.mozilla.org/en-US/docs/Web/API/EventTarget

5. **MDN Fetch**
   https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API

6. **MDN WebAssembly**
   https://developer.mozilla.org/en-US/docs/WebAssembly

7. **Node.js Documentation**
   https://nodejs.org/docs/

8. **TC39 Proposals**
   https://github.com/tc39/proposals

9. **WebAssembly Specifications**
   https://webassembly.org/specs/

10. **OWASP**
    https://owasp.org/

11. **OWASP JavaScript Security Guidance**
    https://cheatsheetseries.owasp.org/

Source discipline:

```text
language semantics
→ ECMAScript specification

host behavior
→ relevant Web/Node/platform specification

security
→ authoritative security guidance + actual threat model

performance
→ benchmarks/profiling

production reliability
→ telemetry + incident evidence

anti-pattern decision
→ context + measured risk
```

Avoid treating style guides as universal laws.

---

# Chapter 98 — Revision / Retrieval Record

```md
## Review Record

### Review #
- Date:
- Duration:
- Status before:
- Status after:

### Retrieval
- Could I define anti-pattern precisely? [ ]
- Could I distinguish anti-pattern from style? [ ]
- Could I explain shared mutable state? [ ]
- Could I explain temporal coupling? [ ]
- Could I explain promise anti-patterns? [ ]
- Could I explain retry amplification? [ ]
- Could I explain listener/memory leaks? [ ]
- Could I explain security anti-patterns? [ ]
- Could I explain compatibility mistakes? [ ]
- Could I identify architectural anti-patterns? [ ]
- Could I build a failure hypothesis? [ ]
- Could I justify a remediation decision? [ ]

### Gaps
-

### New Insights
-

### Follow-up
-
```

---

# Chapter 98 — Completion Snapshot

```text
Part XIX — Judgment

Chapter 98 — Anti-Patterns and Failure Modes
[ ] Not Started

Track A — Core Theory
[ ] Anti-pattern definition
[ ] bug vs anti-pattern
[ ] style vs anti-pattern
[ ] trade-offs
[ ] state failures
[ ] scope failures
[ ] coercion failures
[ ] object/prototype failures
[ ] async failures
[ ] event failures
[ ] memory failures
[ ] performance failures
[ ] security failures
[ ] module/dependency failures
[ ] API failures
[ ] reliability failures
[ ] observability failures
[ ] testing failures
[ ] architecture failures
[ ] edge/serverless failures
[ ] Wasm failures
[ ] compatibility failures

Track B — Implementation
[ ] Anti-pattern scanner
[ ] heuristic reviewer
[ ] concurrency reproduction
[ ] retry reproduction
[ ] cancellation test
[ ] memory leak reproduction
[ ] compatibility test
[ ] security test
[ ] production review tool

Track C — Interview / Reasoning
[ ] Explain an anti-pattern
[ ] Defend contextual judgment
[ ] Identify root cause
[ ] Quantify risk
[ ] Prioritize remediation
[ ] Design migration
[ ] Defend leaving it alone
[ ] Review architecture
[ ] Build governance

Mastery Gate
[ ] Understand
[ ] Explain
[ ] Predict
[ ] Implement
[ ] Debug
[ ] Apply
[ ] Compare
[ ] Defend
```

---

# Completion Criteria

Do not mark this chapter mastered because you can name 50 anti-patterns.

You are ready to move forward when you can independently:

1. Define anti-pattern.
2. Distinguish anti-pattern from bug and style.
3. Explain contextual trade-offs.
4. Identify hidden assumptions.
5. Predict likely failure modes.
6. Explain shared-state risks.
7. Explain async failure modes.
8. Explain memory-retention patterns.
9. Explain security anti-patterns.
10. Explain module/dependency risks.
11. Explain reliability failure patterns.
12. Explain edge/serverless traps.
13. Explain Wasm FFI traps.
14. Build a minimal reproduction.
15. Separate symptom from root cause.
16. Quantify severity.
17. Choose a remediation strategy.
18. Design regression tests.
19. Measure improvement.
20. Decide when not to change the code.
21. Defend an anti-pattern decision under architecture review.

---

# 251. Principal Challenge

Conduct a full failure-mode review of a global JavaScript platform.

Architecture:

```text
Browser
 ↓
Edge
 ↓
Regional API
 ↓
Queue
 ↓
Worker
 ↓
Database
 ↓
Object storage
```

The platform also contains:

```text
legacy JavaScript
Node
ESM/CommonJS
Wasm
third-party dependencies
global cache
```

Identify at least:

```text
10 correctness risks
10 reliability risks
10 security risks
10 performance risks
10 maintainability risks
```

For every finding:

```text
Pattern:
Context:
Assumption:
Failure:
Impact:
Evidence:
Severity:
Repair:
Test:
Owner:
```

Then select the **five highest-value changes**.

Defend why these five should happen first.

Your decision must use:

```text
blast radius
likelihood
business impact
security
operational cost
migration cost
```

---

# Final Reference Card

```text
Anti-pattern
=
recurring approach whose trade-offs
systematically create undesirable outcomes
in a given context

Bad code
≠
automatically anti-pattern

Old code
≠
automatically anti-pattern

Complex code
≠
automatically anti-pattern

Pattern + context + evidence
→ engineering judgment

Common high-risk signals:

global mutable state
temporal coupling
unbounded concurrency
ignored promises
retry storms
missing cancellation
listener leaks
unbounded caches
dynamic code
unsafe HTML
prototype pollution
dependency sprawl
ambiguous temporal data
missing idempotency
missing timeouts
shared database ownership
provider assumptions
raw Wasm ABI
browser sniffing
implementation-only tests

Principal loop:

Observe
→ Explain
→ Measure
→ Prioritize
→ Repair
→ Verify
→ Monitor
```

> **Mastery reminder:** Reading alone does not mark completion. Mastery requires retrieval, prediction, implementation, debugging, application, comparison, and defense.

---

# 252. Failure Analysis Template

For every production defect, write:

```text
Observed symptom:
Affected requests/users:
First known occurrence:
Environment:
Trigger:
Immediate mechanism:
Root cause:
Contributing anti-pattern:
Missing control:
Blast radius:
Detection gap:
Repair:
Regression test:
Permanent prevention:
```

This prevents the review from stopping at:

```text
“we added a null check.”
```

The real target is the mechanism that allowed the null state to become reachable.

---

# 253. Five Whys — JavaScript Example

Incident:

```text
duplicate notification sent
```

Why 1:

```text
job executed twice
```

Why 2:

```text
queue redelivered message
```

Why 3:

```text
ack happened after processing timeout
```

Why 4:

```text
processing duration exceeded visibility window
```

Why 5:

```text
job had no bounded concurrency and downstream API became slow
```

The anti-pattern is not simply:

```text
“queue duplicate.”
```

It is a combination of:

```text
unbounded processing
+
poor deadline management
+
non-idempotent side effect
```

---

# 254. Fault Tree Analysis

For a failure such as:

```text
customer sees another tenant's data
```

build a fault tree:

```text
Cross-tenant exposure
       │
       ├── wrong cache key
       │      └── tenant omitted
       │
       ├── wrong request state
       │      └── global mutable variable
       │
       ├── authorization bug
       │      └── resource owner not checked
       │
       └── stale state
              └── cache invalidation failure
```

This is more useful than searching for one suspicious line.

---

# 255. Control Mapping

For every anti-pattern, identify the control that should catch it.

| Risk | Prevent | Detect | Recover |
|---|---|---|---|
| shared state | request-local design | concurrency test | restart/replay |
| XSS | output encoding | security test | invalidate/redeploy |
| retry duplicate | idempotency | duplicate metric | reconcile |
| memory leak | lifecycle ownership | heap telemetry | recycle process |
| unsupported API | baseline policy | CI matrix | fallback |
| oversized payload | request limit | metric | reject |

A mature system does not depend on code review alone.

---

# 256. Prevention vs Detection

Prevention:

```text
make invalid state harder to represent
```

Detection:

```text
notice invalid state quickly
```

Recovery:

```text
restore service safely
```

Example:

```text
idempotency key
→ prevention

duplicate-operation metric
→ detection

reconciliation job
→ recovery
```

---

# 257. Invalid States

One of the strongest anti-pattern-reduction techniques is to reduce invalid states.

Bad model:

```js
const result = {
  loading: true,
  data: null,
  error: null,
  retrying: false,
  cancelled: false
};
```

Many impossible combinations are representable.

A state machine can make the valid states explicit:

```text
idle
loading
success
failure
```

and define transitions.

---

# 258. Boolean Explosion as State-Model Failure

If an object has:

```text
isLoading
isSaving
isDeleting
hasError
isRetrying
isCancelled
```

the number of theoretically possible combinations grows rapidly.

Some combinations may be impossible.

When state complexity grows, consider a finite state model rather than adding more booleans.

---

# 259. State Machine Example

```js
const states = {
  idle: {
    SUBMIT: "submitting"
  },
  submitting: {
    SUCCESS: "success",
    FAILURE: "failure"
  },
  failure: {
    RETRY: "submitting"
  },
  success: {}
};
```

A state transition table makes invalid transitions visible.

---

# 260. Anti-Pattern — Implicit State Machine

Code:

```js
if (loading) {
  if (!error) {
    if (!cancelled) {
      // maybe continue
    }
  }
}
```

This hides a state machine inside boolean logic.

Refactor when the state model becomes a major reasoning burden.

---

# 261. Anti-Pattern — Temporal Coupling Through Calls

```js
client.connect();
client.authenticate();
client.send();
```

If `send()` silently assumes the first two calls already happened, the object has a hidden state protocol.

Prefer an API that makes valid sequencing visible.

---

# 262. Anti-Pattern — Temporal Coupling Through Imports

```text
import setup
import plugin
import feature
```

where changing import order breaks startup.

Prefer explicit initialization contracts.

---

# 263. Anti-Pattern — Temporal Coupling Through Environment

Code only works when:

```text
ENV_VAR
```

has been loaded by another script first.

Move environment validation to startup.

---

# 264. Anti-Pattern — Configuration Mutation After Startup

```js
configure({ timeout: 1000 });
start();
configure({ timeout: 5000 });
```

Unless dynamic reconfiguration is explicitly supported, this produces inconsistent behavior.

Prefer immutable startup configuration.

---

# 265. Anti-Pattern — Ambient Context

Ambient context means code obtains critical dependencies implicitly from its environment.

Examples:

```text
process.env
window.location
module-level singleton
current request global
current user global
```

Ambient context can be convenient at a boundary.

It becomes an anti-pattern when core business logic silently depends on it.

---

# 266. Anti-Pattern — Dependency Discovery at Call Time

```js
function save() {
  const db = globalThis.database;
  return db.insert(...);
}
```

The dependency is hidden.

Prefer construction-time injection where practical.

---

# 267. Dependency Injection Is Not Automatically Better

This is also possible:

```js
function createEverything(container) {
  // 50 dependencies
}
```

Excessive dependency injection can create:

```text
wiring complexity
service locator behavior
```

The goal is explicit, cohesive dependencies.

---

# 268. Anti-Pattern — Service Locator

```js
container.get("database")
container.get("mailer")
container.get("cache")
```

inside every service hides dependencies behind runtime lookup.

Prefer constructors/functions that declare the dependencies they require.

---

# 269. Anti-Pattern — God Dependency Container

A giant container containing every service can become:

```text
global state
lifecycle ambiguity
circular dependencies
```

Use bounded composition roots.

---

# 270. Composition Root

A composition root is a place where the application wires together:

```text
configuration
adapters
repositories
services
controllers
```

Business modules should not discover infrastructure by themselves.

---

# 271. Anti-Pattern — Business Logic in HTTP Handler

```js
async function handler(request) {
  authenticate();
  validate();
  calculateTax();
  chargeCard();
  saveOrder();
  sendEmail();
}
```

An HTTP adapter should translate transport concerns into an application command.

---

# 272. Better Layering

```text
HTTP
 ↓
application command
 ↓
domain
 ↓
ports
 ↓
adapters
```

Not every application needs all four layers.

Use them when they reduce meaningful coupling.

---

# 273. Anti-Pattern — Repository Abstraction Theater

```js
class UniversalRepository {
  findAnything(options) {}
  updateAnything(options) {}
  executeRaw(sql) {}
}
```

If every consumer still knows SQL semantics, the abstraction is not buying much.

---

# 274. Anti-Pattern — ORM Leakage

Business code receives:

```text
ORM entities
transaction handles
lazy-loading proxies
query builders
```

everywhere.

This couples the domain to persistence technology.

Use explicit boundaries when persistence independence is valuable.

---

# 275. Anti-Pattern — Over-Generic Repository

```js
repository.save(anything, options, flags, mode, metadata, strategy, context)
```

Generic APIs can hide the actual domain operations.

Prefer meaningful repository contracts.

---

# 276. Anti-Pattern — Hidden Network Call

A method named:

```js
user.getProfile()
```

may silently trigger an HTTP request.

Callers assume local object access.

Network I/O should be obvious when latency and failure are relevant.

---

# 277. Anti-Pattern — Hidden Disk I/O

A supposedly cheap function reads files internally.

This creates:

```text
latency surprise
error surprise
test complexity
```

Document or separate I/O behavior.

---

# 278. Anti-Pattern — Hidden Async

An API changes from:

```js
function parse(input) {
  return value;
}
```

to:

```js
async function parse(input) {
  return value;
}
```

and callers now receive Promise objects.

This is a breaking contract change even though the returned value is the same after awaiting.

---

# 279. Anti-Pattern — Hidden Mutability

A function appears pure:

```js
normalize(input)
```

but mutates:

```text
input
or
shared global state
```

Document or eliminate side effects.

---

# 280. Anti-Pattern — Mutating Function Arguments

```js
function normalize(user) {
  user.name = user.name.trim();
  return user;
}
```

This may be correct if mutation is the explicit contract.

It is risky when callers assume the input remains unchanged.

---

# 281. Anti-Pattern — Defensive Copy Everywhere

Blindly cloning every argument can produce:

```text
CPU cost
memory cost
identity changes
```

Use ownership rules instead of cloning reflexively.

---

# 282. Anti-Pattern — Deep Copy as Ownership Model

If a system solves all aliasing concerns with:

```js
structuredClone(value)
```

everywhere, it may be avoiding proper ownership design.

Clone when crossing a boundary requires isolation, not automatically.

---

# 283. Anti-Pattern — API Returning Mutable Internal State

```js
getConfig() {
  return this.config;
}
```

A caller can mutate internal state.

Return an immutable value or controlled view when necessary.

---

# 284. Anti-Pattern — Internal Cache Exposed Publicly

```js
getCache() {
  return cache;
}
```

Consumers can now accidentally define the cache's behavior.

Expose operations rather than internal structures.

---

# 285. Anti-Pattern — Stringly-Typed Feature Flags

```js
if (flags["new-checkout-v2"] === "enabled") {}
```

Feature flags should have:

```text
owner
lifecycle
scope
expiration
```

and a stable evaluation contract.

---

# 286. Anti-Pattern — Permanent Feature Flags

A flag introduced for a rollout can become permanent.

This creates:

```text
branching complexity
untested combinations
```

Every temporary flag should have a removal condition.

---

# 287. Anti-Pattern — Flag Matrix Explosion

With `N` independent booleans, possible combinations can approach:

```text
2^N
```

Many combinations will never be tested.

Reduce simultaneous flags or encode explicit rollout states.

---

# 288. Anti-Pattern — Configuration as a Hidden Programming Language

If configuration supports:

```text
nested conditions
expressions
references
plugins
scripts
```

it may have become a second programming language.

At that point define an explicit language/schema or simplify the configuration.

---

# 289. Anti-Pattern — Regex Configuration

Using regexes everywhere for policy can create:

```text
security ambiguity
performance issues
maintenance cost
```

Prefer structured policy models where possible.

---

# 290. Anti-Pattern — Magic Numbers

```js
if (retryCount > 7) {}
```

The number's meaning is invisible.

Prefer:

```js
const MAX_RETRIES = 7;
```

or a named policy object.

---

# 291. Anti-Pattern — Magic Time Units

```js
setTimeout(fn, 60000);
```

The unit is hidden.

Prefer:

```js
const ONE_MINUTE_MS = 60_000;
```

or an API that takes explicit duration semantics.

---

# 292. Anti-Pattern — Mixed Unit APIs

A function accepts:

```text
milliseconds
seconds
minutes
```

without naming the unit.

This invites catastrophic timing bugs.

Use:

```text
timeoutMs
retryAfterSeconds
```

or typed/domain wrappers.

---

# 293. Anti-Pattern — Date Arithmetic by Milliseconds

```js
date.getTime() + 30 * 86400000
```

This can be wrong for calendar semantics.

Use Temporal/domain-specific arithmetic where the question is calendar-based.

---

# 294. Anti-Pattern — UTC-Only Thinking

UTC is excellent for exact timestamps.

It is not a replacement for:

```text
local dates
local times
named time zones
recurring schedules
```

See Chapter 92.

---

# 295. Anti-Pattern — Locale From Host Location

Do not assume:

```text
server region = user locale
```

The execution region is infrastructure, not necessarily business context.

---

# 296. Anti-Pattern — Time as a String

```js
if (time === "9:00") {}
```

This hides:

```text
format
locale
timezone
precision
```

Define the temporal contract explicitly.

---

# 297. Anti-Pattern — Manual URL Parsing

```js
const parts = url.split("/");
```

URLs have structured semantics.

Use platform URL APIs where appropriate.

---

# 298. Anti-Pattern — Manual Cookie Parsing

Cookies contain:

```text
encoding
attributes
multiple values
security semantics
```

Use vetted libraries/platform APIs rather than ad hoc parsing for security-critical flows.

---

# 299. Anti-Pattern — Ad Hoc JSON Validation

```js
if (payload && payload.user && payload.user.email) {}
```

For complex APIs, formal schema validation gives clearer contracts and better failure reporting.

---

# 300. Anti-Pattern — Validation After Side Effect

Bad:

```js
chargeCard();
validateOrder();
```

Validation should generally happen before irreversible side effects unless the domain explicitly requires another ordering.

---

# 301. Anti-Pattern — Side Effects in Validation

A function named:

```js
validateOrder(order)
```

should not silently:

```text
write database
send email
call payment API
```

The name and behavior should agree.

---

# 302. Anti-Pattern — Constructor With Side Effects

```js
new Service()
```

should not unexpectedly:

```text
open network connection
start timer
write database
```

unless lifecycle semantics are explicit.

Prefer explicit `start()` or factory initialization where required.

---

# 303. Anti-Pattern — Async Constructor Illusion

JavaScript constructors do not naturally provide an awaited initialization protocol.

Do not hide asynchronous setup behind a constructor without a clear factory/lifecycle model.

---

# 304. Factory With Hidden Side Effects

Factories can also become surprising:

```js
createService()
```

may contact remote services.

Name and document heavyweight initialization.

---

# 305. Anti-Pattern — Eager Initialization of Everything

At startup:

```text
connect all DBs
load all models
load all plugins
compile all templates
initialize all caches
```

This increases cold start and failure surface.

Initialize according to actual need.

---

# 306. Anti-Pattern — Lazy Initialization Without Concurrency Control

```js
if (!client) {
  client = await createClient();
}
```

Concurrent callers can race and initialize multiple clients.

Store the initialization Promise when appropriate:

```js
let clientPromise;

function getClient() {
  clientPromise ??= createClient();
  return clientPromise;
}
```

Then reason about failure/reset semantics.

---

# 307. Failed Initialization Cache

If:

```js
clientPromise = createClient();
```

rejects and remains cached, every future call may immediately fail.

Whether that is correct depends on the dependency.

Design reset/retry behavior deliberately.

---

# 308. Anti-Pattern — Retrying Initialization Forever

A service that repeatedly retries a broken dependency at startup can create:

```text
CPU churn
log storms
slow readiness
```

Use bounded retry/backoff and clear health semantics.

---

# 309. Anti-Pattern — Health Check That Lies

```js
return { ok: true };
```

while:

```text
database unavailable
queue unavailable
critical configuration missing
```

A health endpoint should match the intended readiness/liveness contract.

---

# 310. Liveness vs Readiness

Liveness asks:

```text
is the process/runtime alive?
```

Readiness asks:

```text
can this instance serve required traffic?
```

Do not combine them carelessly.

---

# 311. Anti-Pattern — Dependency Health Fan-Out

A health check calls:

```text
10 downstream services
```

for every probe.

This can amplify load during incidents.

Prefer shallow health semantics and separate dependency telemetry.

---

# 312. Anti-Pattern — Retry Through Health Endpoint

Never make health checks trigger expensive recovery or initialization unless explicitly designed.

Health checks can run frequently.

---

# 313. Anti-Pattern — Logging in Hot Loops

```js
for (...) {
  console.log(value);
}
```

can dominate runtime and produce huge logs.

Use sampled/aggregated diagnostics.

---

# 314. Anti-Pattern — Debug Logging Left Enabled

Verbose logs may:

```text
expose data
increase cost
increase latency
hide incidents
```

Use controlled logging levels.

---

# 315. Anti-Pattern — Error Message as Contract

Clients should not depend on:

```text
error.message === "something failed"
```

Use stable:

```text
error code
schema
category
```

Messages are for humans and can evolve.

---

# 316. Anti-Pattern — Stack Trace as API

Never design machine logic around stack trace text.

Stack frames change with bundling, runtime, and implementation.

---

# 317. Anti-Pattern — String Parsing Stack Traces

```js
if (error.stack.includes("Timeout")) {}
```

Use typed errors or structured metadata.

---

# 318. Anti-Pattern — Error Swallowing in Event Handlers

```js
emitter.on("event", async () => {
  try {
    await work();
  } catch {}
});
```

Failure disappears.

Route the error to an observable owner.

---

# 319. Anti-Pattern — Unhandled Rejection as Control Flow

```js
doWork().catch(() => {
  // ignore
});
```

If rejection means something, classify it.

---

# 320. Anti-Pattern — Global `unhandledRejection` as Error Handling Strategy

A global handler can help observability.

It should not replace local ownership of expected failures.

---

# 321. Anti-Pattern — Global Error Handler as Recovery

A process-level handler cannot reconstruct the correct context for every failure.

Use it for:

```text
last-resort reporting
controlled shutdown
```

not ordinary domain recovery.

---

# 322. Anti-Pattern — Catching Programmer Errors as User Errors

```js
try {
  return calculate(input);
} catch {
  return { message: "Invalid input" };
}
```

A programming bug can be incorrectly reported as a validation failure.

Separate expected domain failures from unexpected faults.

---

# 323. Anti-Pattern — Error Normalization Too Early

Converting every error to:

```js
new Error("Operation failed")
```

can destroy useful classification.

Preserve cause and metadata.

---

# 324. Anti-Pattern — Logging and Throwing Without Coordination

```js
catch (error) {
  logger.error(error);
  throw error;
}
```

repeated across every layer can produce duplicate logs.

Choose an ownership level for final error logging.

---

# 325. Anti-Pattern — Retry in Every Layer

```text
HTTP client retry
SDK retry
service retry
queue retry
workflow retry
```

The effective attempt count can become enormous.

Define retry responsibility centrally.

---

# 326. Anti-Pattern — Fixed Retry Delay

```js
await delay(1000);
```

for every retry can synchronize many clients.

Use jittered backoff when the failure mode warrants it.

---

# 327. Anti-Pattern — Retry on Validation Errors

Do not retry deterministic failures such as:

```text
invalid request
permission denied
schema mismatch
```

unless there is a specific transient reason.

---

# 328. Anti-Pattern — Retry on Authentication Failure

Blind retries do not make an invalid credential valid.

They add load and delay.

---

# 329. Anti-Pattern — Infinite Polling

```js
while (true) {
  await fetchStatus();
  await delay(1000);
}
```

requires:

```text
cancellation
backoff
maximum lifetime
shutdown handling
```

Prefer durable scheduling or events where appropriate.

---

# 330. Anti-Pattern — Polling a Database for Every Request

If every HTTP request checks a DB for changing state, the system can become database-bound.

Consider:

```text
cache
push/event
read model
```

when valid.

---

# 331. Anti-Pattern — Cache Stampede

A popular key expires and thousands of requests fetch it simultaneously.

Mitigations can include:

```text
single-flight
jittered TTL
stale-while-revalidate
refresh-ahead
```

---

# 332. Anti-Pattern — Cache Without Negative Results

Repeatedly looking up a missing object can overload the backend.

A bounded negative cache can help when semantics permit it.

---

# 333. Anti-Pattern — Cache Without Invalidation Ownership

If nobody owns invalidation, the cache will eventually become a correctness problem.

Define:

```text
owner
TTL
purge
versioning
```

---

# 334. Anti-Pattern — Local Cache Across Tenants

Do not assume the same key means the same data in every tenant.

Tenant scope must be part of the cache model.

---

# 335. Anti-Pattern — Cache Key Incompleteness

If response depends on:

```text
locale
tenant
authorization
experiment
```

those semantics may need representation in the cache key or cache policy.

---

# 336. Anti-Pattern — Authorization After Cache

If a private response is globally cached before authorization semantics are applied, cross-user exposure can occur.

Security and caching must be designed together.

---

# 337. Anti-Pattern — Cache-Control by Guess

Do not add:

```text
public, max-age=86400
```

without knowing whether the response is safe to cache.

---

# 338. Anti-Pattern — N+1 Network Requests

```js
for (const id of ids) {
  await fetch(`/users/${id}`);
}
```

can create one request per item.

Use batching or suitable data-fetch strategies where possible.

---

# 339. Anti-Pattern — N+1 Database Queries

```js
const orders = await getOrders();

for (const order of orders) {
  order.customer = await getCustomer(order.customerId);
}
```

Can create:

```text
1 + N queries
```

Use joins, batch queries, dataloading, or read models as appropriate.

---

# 340. Anti-Pattern — Sequential Independent Network Calls

When independent:

```js
await a();
await b();
await c();
```

adds latency.

Parallelize intentionally.

---

# 341. Anti-Pattern — Parallel Dependent Calls

The opposite mistake:

```js
const [user, permissions] = await Promise.all([
  getUser(),
  getPermissions(user.id)
]);
```

The second call depends on the first.

The code is invalid or conceptually wrong.

---

# 342. Anti-Pattern — Unbounded Fan-Out

One request launches:

```text
1000 downstream requests
```

Even if every request is independent, the system may collapse under load.

Use bounded concurrency.

---

# 343. Bounded Concurrency Model

```js
const limit = createLimiter(20);

await Promise.all(
  items.map(item =>
    limit(() => process(item))
  )
);
```

The exact implementation can vary.

The key principle is explicit concurrency control.

---

# 344. Anti-Pattern — Concurrency Based Only on CPU Count

JavaScript concurrency limits should also account for:

```text
DB capacity
API limits
memory
latency
rate limits
```

CPU count is only one constraint.

---

# 345. Anti-Pattern — Ignoring Backpressure

A readable stream can produce faster than the consumer can process.

If the system buffers without bound:

```text
memory grows
```

Respect backpressure.

---

# 346. Anti-Pattern — Event Buffer Without Limit

```js
const events = [];
```

with no maximum size can turn temporary traffic into permanent memory growth.

Set limits and failure policy.

---

# 347. Anti-Pattern — Queue Without Dead Letter Strategy

Poison messages can retry forever.

Define:

```text
max attempts
backoff
dead-letter path
operator handling
```

---

# 348. Anti-Pattern — Queue Message Without Version

When message schema evolves, old messages may still exist.

Include a version:

```json
{
  "version": 2,
  "type": "order.created"
}
```

---

# 349. Anti-Pattern — Event Schema Without Ownership

A shared event can become impossible to evolve when nobody owns:

```text
meaning
compatibility
versioning
```

Every durable event should have an owner.

---

# 350. Anti-Pattern — Event Name as Domain Contract Without Documentation

```text
order-created
```

does not define:

```text
when emitted
what fields mean
whether delivery is guaranteed
whether duplicates happen
```

Events require contracts.

---

# 351. Anti-Pattern — Synchronous Chain Across Many Services

```text
A → B → C → D → E
```

Each hop adds:

```text
latency
failure
retry
observability complexity
```

Use asynchronous or aggregated boundaries when appropriate.

---

# 352. Anti-Pattern — Distributed Monolith

Many services are coupled through synchronous calls and shared data ownership.

The system has the complexity of microservices with the coupling of a monolith.

---

# 353. Anti-Pattern — Shared Database Writes Everywhere

If five services directly write the same tables, ownership is unclear.

Define authoritative writers.

---

# 354. Anti-Pattern — Eventual Consistency Without UX Design

If a user creates an order and immediately sees:

```text
“order not found”
```

because another region has not replicated it, the data model may be technically valid but product behavior is poor.

Design read-after-write UX intentionally.

---

# 355. Anti-Pattern — Transaction Across Network Calls

Holding a DB transaction open while calling another service can create:

```text
long locks
failure complexity
deadlocks
```

Use durable workflow/saga patterns when cross-system coordination is necessary.

---

# 356. Anti-Pattern — Two-Phase Commit by Accident

A service calls:

```text
DB commit
then payment API
```

If payment fails, state is inconsistent.

Reverse ordering can create the opposite problem.

Use explicit consistency strategy.

---

# 357. Anti-Pattern — “Just Retry” for Distributed Consistency

Retries do not solve:

```text
conflicting writes
lost updates
stale reads
```

They solve some transient failures.

---

# 358. Optimistic Concurrency

Use:

```text
version
ETag
updatedAt
compare-and-swap
```

where lost updates are a risk.

---

# 359. Anti-Pattern — Last Write Wins Without Meaning

```text
new value replaces old value
```

can silently lose a legitimate update.

Use a domain-specific merge/conflict policy.

---

# 360. Anti-Pattern — Timestamp as Version

A timestamp can collide or be reordered by clocks.

Use explicit version counters when versioning is the domain requirement.

---

# 361. Anti-Pattern — Request ID as Business ID

Request IDs are operational identifiers.

They should not automatically become:

```text
order ID
payment ID
customer ID
```

Different identities serve different purposes.

---

# 362. Anti-Pattern — Logging Sensitive Identifiers

A request ID may be safe.

A payment token may not be.

Classify data before logging.

---

# 363. Anti-Pattern — Metrics for Every Entity

Do not create a metric label for every:

```text
user
order
request
URL
```

Use traces/logs for detailed identity and metrics for aggregation.

---

# 364. Anti-Pattern — Alert on Every Error

If every 404 alerts an engineer, the alerting system becomes ignored.

Alert on:

```text
actionable
meaningful
thresholded
```

conditions.

---

# 365. Anti-Pattern — No SLO

Without reliability objectives, teams argue about incidents using opinions.

Define:

```text
availability
latency
error budget
```

when appropriate.

---

# 366. Anti-Pattern — SLO Without User Journey

A service may have:

```text
99.99% HTTP success
```

while checkout still fails because a downstream step breaks.

Measure user-visible outcomes too.

---

# 367. Anti-Pattern — No Error Budget Thinking

If every reliability improvement is optional, reliability work is perpetually deferred.

Use an explicit reliability budget when the organization supports SRE-style planning.

---

# 368. Anti-Pattern — Performance Target Without Measurement Method

```text
“under 100 ms”
```

must specify:

```text
p50?
p95?
p99?
client or server?
cold or warm?
```

---

# 369. Anti-Pattern — Average Latency Only

Averages can hide tail latency.

Use percentiles for user-impact analysis.

---

# 370. Anti-Pattern — Benchmarking the Wrong Layer

Measuring:

```text
Wasm kernel
```

when the application cost is:

```text
serialization + network + database
```

does not guide the right optimization.

---

# 371. Anti-Pattern — Synthetic Benchmark as Production Truth

A microbenchmark can isolate a mechanism.

It cannot automatically predict a distributed production workload.

Use both micro and workload benchmarks.

---

# 372. Anti-Pattern — Benchmark Without Warmup

JIT-based engines can optimize hot code.

Measure:

```text
cold
warm
steady-state
```

when relevant.

---

# 373. Anti-Pattern — Benchmark With Logging Enabled

Logging can dominate the workload.

Benchmark realistic observability configurations.

---

# 374. Anti-Pattern — Benchmark With Unrealistic Data

Branching, allocation, cache locality, and string behavior depend on input shape.

Use representative datasets.

---

# 375. Anti-Pattern — Optimization Without Regression Test

A performance change that subtly changes semantics is not a successful optimization.

Performance and correctness tests belong together.

---

# 376. Anti-Pattern — Premature Parallelism

Threads/workers can add:

```text
serialization
coordination
complexity
```

Use them when CPU or throughput evidence warrants it.

---

# 377. Anti-Pattern — Worker for Tiny Work

Sending one trivial calculation to a Worker may cost more than doing it inline.

Benchmark the crossover point.

---

# 378. Anti-Pattern — Wasm for Tiny Work

Similarly:

```js
wasm.add(1, 2)
```

may not beat:

```js
1 + 2
```

once interop is included.

---

# 379. Anti-Pattern — Edge for Distant Data

Global compute calling one distant database on every request can create:

```text
edge execution
+
origin latency
```

rather than true low-latency service.

---

# 380. Anti-Pattern — Serverless Memory as Queue

```js
const pending = [];
```

is not a durable job queue.

Use the platform's queue/workflow/storage capability.

---

# 381. Anti-Pattern — In-Memory Lock in Distributed System

```js
let locked = false;
```

protects one process/isolate, not a global resource.

Use a distributed coordination primitive.

---

# 382. Anti-Pattern — Local Time as Global Business Time

```js
new Date().getHours()
```

can make behavior depend on where the process runs.

Use explicit business zone/time semantics.

---

# 383. Anti-Pattern — Environment-Dependent Tests

A test that only passes in one developer timezone is not deterministic.

Control:

```text
time
locale
timezone
randomness
network
```

---

# 384. Anti-Pattern — Test Order Dependency

Test A mutates global state.

Test B passes only because A ran first.

Tests should isolate state unless shared fixtures are explicit and reset.

---

# 385. Anti-Pattern — Fake That Does Not Preserve Contract

A mock that says:

```js
getUser() → instant success
```

when production can:

```text
timeout
retry
partial response
```

may hide real failure modes.

Mocks should model important contract behavior.

---

# 386. Anti-Pattern — Mock Everything

If every dependency is mocked, integration boundaries are untested.

Use real integrations selectively.

---

# 387. Anti-Pattern — Integration Test for Every Unit Detail

Using a full browser/database/system test for simple pure logic is slow and brittle.

Choose the cheapest test that proves the contract.

---

# 388. Anti-Pattern — Flaky Test as “Eventually Green”

Rerunning until green is not a strategy.

Measure flake rate, isolate nondeterminism, and fix the cause.

---

# 389. Anti-Pattern — Retry Test Runner

A test suite that automatically retries failed tests can hide concurrency or timing bugs.

Use retries as diagnostics, not proof of stability.

---

# 390. Anti-Pattern — Snapshot as Specification

A snapshot records output.

It does not necessarily explain why the output is correct.

Keep semantic assertions for important behavior.

---

# 391. Anti-Pattern — Coverage as Quality

100% line coverage does not guarantee:

```text
correct concurrency
security
real integration
meaningful assertions
```

Coverage is a signal, not proof.

---

# 392. Anti-Pattern — Branch Coverage as Correctness

Executing both branches does not prove they implement the correct behavior.

Assert outcomes and invariants.

---

# 393. Anti-Pattern — Lint as Architecture

Lint rules can prevent classes of mistakes.

They cannot replace architecture review.

Do not turn every design judgment into a syntax rule.

---

# 394. Anti-Pattern — One Rulebook for Every Runtime

Browser, Node, Worker, serverless, and edge code have different constraints.

Use a shared core policy plus environment-specific rules.

---

# 395. Anti-Pattern — One Browser Matrix Forever

Browser populations evolve.

Refresh your baseline from:

```text
telemetry
compatibility data
customer requirements
```

---

# 396. Anti-Pattern — Runtime Version by Developer Preference

Choose runtime baselines from:

```text
security
support lifecycle
features
fleet
cost
```

not merely the newest release.

---

# 397. Anti-Pattern — Library Supporting Everything

A public library that promises:

```text
all browsers
all Node versions
all bundlers
```

may accumulate unsustainable compatibility debt.

Define a clear support contract.

---

# 398. Anti-Pattern — Application Depending on Hidden Transpilation

Developers write modern syntax locally.

Published package accidentally ships untransformed syntax.

Test the actual distributed artifact.

---

# 399. Anti-Pattern — `dist` Is the Source of Truth

Debugging generated code as if it were the authored source can lead to incorrect fixes.

Trace:

```text
artifact
→ source map
→ source
→ build configuration
```

---

# 400. Anti-Pattern — Manual Patch to Generated File

A production emergency may tempt you to edit:

```text
dist/bundle.js
```

The fix disappears on the next build.

Patch the source/build pipeline, then regenerate.

---

# 401. Anti-Pattern — No Reproducible Dependency Graph

Different builds resolve different transitive versions.

Pin and record the dependency graph appropriately.

---

# 402. Anti-Pattern — Dependency Update Without Runtime Review

A package update can change:

```text
syntax baseline
engine requirement
native dependencies
module format
```

Review platform compatibility.

---

# 403. Anti-Pattern — Security Dependency Update Without Behavioral Test

A security patch can intentionally change behavior.

Run:

```text
security tests
compatibility tests
integration tests
```

---

# 404. Anti-Pattern — Security Tool Output as Final Truth

Static scanners produce findings.

They can have:

```text
false positives
false negatives
context gaps
```

Human threat modeling remains necessary.

---

# 405. Anti-Pattern — Security by Library Name

A library called “secure-parser” is not a threat model.

Understand:

```text
input
parser
output
sink
```

---

# 406. Anti-Pattern — Sanitization Without Output Context

A sanitizer suitable for HTML may not be suitable for:

```text
URL
SQL
shell
CSS
JavaScript
```

Encoding/validation must match the sink.

---

# 407. Anti-Pattern — One Validator for Every Context

```js
sanitize(input)
```

is ambiguous.

Prefer context-specific validation or clearly defined schemas.

---

# 408. Anti-Pattern — Trust Boundary Lost in Utility Function

A function named:

```js
normalize(value)
```

may accept untrusted input but be treated as trusted afterward.

Document trust boundaries.

---

# 409. Anti-Pattern — Authorization in UI Only

Hiding a button is not authorization.

The server/resource boundary must enforce permission.

---

# 410. Anti-Pattern — Tenant ID From Client Without Verification

```js
const tenantId = body.tenantId;
```

Do not assume the client can choose its authority scope.

Derive/verify tenant identity from trusted authentication context.

---

# 411. Anti-Pattern — User ID From URL as Authorization

```text
GET /users/123
```

does not mean the requester may access user 123.

Authorization is a separate decision.

---

# 412. Anti-Pattern — Overly Broad CORS

```text
Access-Control-Allow-Origin: *
```

may be correct for public resources.

It can be dangerous for credentialed/private APIs.

Review the exact browser security model.

---

# 413. Anti-Pattern — Error Detail to Untrusted Client

Returning:

```text
SQL
filesystem paths
stack traces
internal hostnames
```

can expose sensitive implementation details.

---

# 414. Anti-Pattern — Debug Endpoint in Production

A route that dumps:

```text
environment
config
heap
internal state
```

must not be casually left accessible.

---

# 415. Anti-Pattern — Health Endpoint With Secrets

Never expose secrets just because an endpoint is “internal.”

Apply the actual trust model.

---

# 416. Anti-Pattern — SSRF Through “Convenient” Fetch Helper

```js
fetchJson(user.url)
```

can become a network pivot.

Use allowlists and URL validation where external URLs are user-controlled.

---

# 417. Anti-Pattern — Redirect Blindly

A validated URL can redirect to an untrusted destination.

Review redirect handling in SSRF-sensitive workflows.

---

# 418. Anti-Pattern — Timeout Without Overall Deadline

Individual operations may each have a timeout while the total workflow has no bound.

Use a shared deadline for request-level reliability.

---

# 419. Anti-Pattern — Cleanup in Only the Success Path

```js
const resource = acquire();
const result = await work(resource);
resource.close();
```

If `work()` throws, cleanup may never happen.

Use `finally` or explicit resource management.

---

# 420. Anti-Pattern — Cleanup in `finally` With Wrong Ownership

A caller should not close a resource it does not own.

Define ownership before adding cleanup.

---

# 421. Anti-Pattern — Double Cleanup Without Idempotency

```js
close();
close();
```

can fail for some resources.

Either enforce single ownership or make cleanup explicitly idempotent where appropriate.

---

# 422. Anti-Pattern — Cancellation Ignored

Caller aborts:

```js
controller.abort();
```

but downstream work continues.

Propagate cancellation to cancellable operations.

---

# 423. Anti-Pattern — Cancellation as Error

Not every abort is a system fault.

Classify cancellation separately so monitoring does not report normal user aborts as outages.

---

# 424. Anti-Pattern — Cancellation Not Idempotent

Cleanup triggered twice can produce secondary failures.

Design lifecycle operations to tolerate repeated signals when practical.

---

# 425. Anti-Pattern — Ignoring Process Shutdown

A Node service that stops accepting new work but immediately exits can lose in-flight operations.

Implement graceful shutdown where the runtime contract requires it.

---

# 426. Anti-Pattern — Shutdown That Never Finishes

A shutdown waits forever for one stuck dependency.

Use a shutdown deadline.

---

# 427. Anti-Pattern — Startup and Shutdown With No State Machine

Services can move through:

```text
created
starting
ready
draining
stopped
failed
```

Model these states explicitly when lifecycle complexity is high.

---

# 428. Anti-Pattern — Process Restart as Universal Recovery

Restarting can clear memory/state.

It cannot fix:

```text
bad data
poison messages
schema mismatch
external dependency outage
```

Restart only addresses the failure class it actually mitigates.

---

# 429. Anti-Pattern — Automatic Rollback Without Data Compatibility

Code can roll back while data cannot.

Check forward/backward compatibility before using automatic rollback.

---

# 430. Anti-Pattern — Destructive Migration First

A database migration that removes old columns before all consumers are migrated makes rollback difficult.

Prefer expand/contract migration when appropriate.

---

# 431. Expand / Contract Migration

```text
expand
→ add new schema
→ support old + new
→ migrate writers
→ migrate readers
→ remove old schema
```

This is a powerful production pattern.

---

# 432. Anti-Pattern — Breaking Message Schema Immediately

A queue may contain old messages for hours or days.

Consumers should remain compatible for the actual message lifetime.

---

# 433. Anti-Pattern — Version Number Without Compatibility Rules

A schema may have:

```text
version: 2
```

but no defined meaning for:

```text
supported prior versions
upgrade path
```

Versioning needs policy.

---

# 434. Anti-Pattern — Compatibility Layer With No Sunset

A v1 adapter survives forever.

Track usage and define removal conditions.

---

# 435. Anti-Pattern — Migration Flag With No Owner

Temporary infrastructure without ownership becomes permanent.

Every migration item needs a responsible owner.

---

# 436. Anti-Pattern — TODO as Project Management

```js
// TODO: remove later
```

without:

```text
issue
owner
deadline
```

is not a migration plan.

---

# 437. Anti-Pattern — Comments as Substitute for Invariants

A comment says:

```text
“do not call before initialize”
```

but the type/API allows it.

Prefer designs that make invalid use harder.

---

# 438. Anti-Pattern — “Private” by Naming Convention

```js
obj._secret
```

is not actual access control.

Use proper encapsulation when privacy matters.

---

# 439. Anti-Pattern — Security Through Obscurity

Renaming:

```text
/admin
→
/internal-x
```

is not authorization.

---

# 440. Anti-Pattern — Base64 as Encryption

```js
btoa(secret)
```

is encoding, not confidentiality.

Use appropriate cryptography.

---

# 441. Anti-Pattern — Hash Without Threat Model

A hash can provide integrity or one-way transformation, but password storage requires a password-specific hashing/KDF design.

Choose primitives from the threat model.

---

# 442. Anti-Pattern — Homegrown Authentication

Avoid inventing token/signature protocols without strong justification and review.

Use established standards and libraries.

---

# 443. Anti-Pattern — JWT Everywhere

JWTs are not automatically superior to sessions.

Choose based on:

```text
revocation
size
storage
verification
rotation
```

and application needs.

---

# 444. Anti-Pattern — Token in Local Storage by Default

Storage choice must consider:

```text
XSS
CSRF
browser threat model
```

There is no universal storage answer.

---

# 445. Anti-Pattern — Refresh Token Forever

Long-lived credentials require:

```text
rotation
revocation
expiration
secure storage
```

---

# 446. Anti-Pattern — Trusting Client Clock for Security

Do not depend on a browser-provided current time for authoritative expiry decisions.

Server-side/security-domain time must be authoritative.

---

# 447. Anti-Pattern — Security Decision From Locale

Locale is presentation context, not authorization context.

---

# 448. Anti-Pattern — Catching `SyntaxError` as User Validation Everywhere

A syntax error may mean:

```text
programmer bug
malformed generated code
invalid user input
```

Classify by source.

---

# 449. Anti-Pattern — Dynamic Code Generation for Business Logic

Generating JavaScript to represent a business rule can make:

```text
testing
security
deployment
```

harder.

Use a data-driven rule model where possible.

---

# 450. Anti-Pattern — “Configurable” Means “Eval”

Do not implement configuration languages by evaluating JavaScript source unless dynamic code is genuinely the product requirement and the trust boundary supports it.

---

# 451. Principal Code Review Rubric

Score each finding:

```text
0 — no meaningful issue
1 — low-risk smell
2 — local maintainability risk
3 — recurring correctness/performance risk
4 — high production/reliability/security risk
5 — critical exploitable or catastrophic pattern
```

Require evidence for scores 3–5.

---

# 452. Review Evidence Hierarchy

Prefer:

```text
1. Reproduction
2. Production telemetry
3. Test failure
4. Benchmark
5. Specification
6. Implementation documentation
7. Expert reasoning
8. Style preference
```

Style preference should almost never outrank concrete production evidence.

---

# 453. Anti-Pattern Waiver

A retained pattern should document:

```md
# Waiver

Pattern:
-

Why retained:
-

Risk accepted:
-

Controls:
-

Owner:
-

Review date:
-

Removal trigger:
-
```

This prevents accidental permanence.

---

# 454. Engineering Principle — Make Failure Cheap

Good architecture reduces:

```text
blast radius
recovery time
ambiguity
```

Examples:

```text
small modules
bounded concurrency
idempotent operations
explicit ownership
feature flags with rollback
```

---

# 455. Engineering Principle — Make Invalid States Hard

Use:

```text
types
schemas
state machines
constructors
factories
invariants
```

when they meaningfully reduce failure space.

---

# 456. Engineering Principle — Make Assumptions Visible

Bad:

```js
run(job);
```

where job requires:

```text
authenticated
connected
initialized
not cancelled
```

Better interfaces encode the assumptions through types/state or explicit arguments.

---

# 457. Engineering Principle — Prefer Local Reasoning

A function should ideally be understandable from:

```text
its inputs
its dependencies
its contract
its local state
```

Global mutable state destroys local reasoning.

---

# 458. Engineering Principle — Preserve Causal Information

Avoid transforming:

```text
TimeoutError
AuthError
ValidationError
```

into:

```text
Error("failed")
```

unless the abstraction genuinely requires that loss.

---

# 459. Engineering Principle — Minimize Implicit Work

If a call performs:

```text
network
DB
cache
logging
retry
```

make the API semantics understandable.

Hidden work creates surprise costs.

---

# 460. Engineering Principle — Minimize Hidden State

Prefer:

```text
explicit state
explicit lifecycle
explicit ownership
```

over:

```text
ambient state
implicit caches
module-global mutations
```

---

# 461. Engineering Principle — Use the Cheapest Correct Abstraction

Do not use:

```text
microservice
workflow engine
Wasm
Worker
event bus
```

when a pure function solves the problem.

But do not force everything into one function when the requirements demand stronger isolation or lifecycle management.

---

# 462. Engineering Principle — Optimize Dominant Costs

Find the largest contributor:

```text
network
DB
CPU
memory
serialization
startup
```

before micro-optimizing syntax.

---

# 463. Engineering Principle — Bound Everything That Can Grow

Bound:

```text
queue
cache
concurrency
input
logs
retries
memory
connections
```

Unbounded resources are common incident precursors.

---

# 464. Engineering Principle — Bound Every Retry

Retry budget should have:

```text
attempts
backoff
jitter
deadline
error classification
```

---

# 465. Engineering Principle — Every Resource Has an Owner

For:

```text
memory
connection
listener
timer
file
Wasm allocation
queue message
transaction
```

ask:

```text
who creates?
who uses?
who releases?
what if release fails?
```

---

# 466. Engineering Principle — Every Async Operation Has a Lifecycle

Define:

```text
start
success
failure
cancel
timeout
cleanup
```

An async operation without lifecycle semantics becomes a leak or race candidate.

---

# 467. Engineering Principle — Every Compatibility Path Has an Exit

Track:

```text
why introduced
who needs it
when removed
```

---

# 468. Engineering Principle — Every Distributed Write Needs Replay Thinking

Ask:

```text
what if request repeats?
what if response is lost?
what if consumer crashes after side effect?
```

This leads naturally to idempotency and durable state transitions.

---

# 469. Engineering Principle — Every Cache Has a Correctness Story

Document:

```text
what can be stale?
for how long?
who invalidates?
who can read?
```

---

# 470. Engineering Principle — Every Alert Has an Action

If no human/system knows what to do when an alert fires, the alert is probably noise.

---

# 471. Engineering Principle — Every Benchmark Has a Question

Bad:

```text
benchmark X
```

Good:

```text
Does batching 1000 Wasm calls into one call reduce end-to-end p99 latency?
```

---

# 472. Engineering Principle — Every Refactor Has a Contract

Before rewriting:

```text
what must remain true?
what may change?
```

Then test those statements.

---

# 473. Engineering Principle — Every Production Migration Has Rollback

Define rollback before deployment.

If rollback is impossible, call that out as a migration property rather than discovering it during the incident.

---

# 474. Principal Anti-Pattern Triage

When time is limited, triage in this order:

```text
1. security / data exposure
2. correctness / financial loss
3. reliability / outage amplification
4. memory / resource exhaustion
5. performance affecting users
6. maintainability
7. style / cleanup
```

This prioritizes business impact rather than aesthetic discomfort.

---

# 475. Principal Review — “Is This Actually an Anti-Pattern?”

Ask:

```text
Is it wrong?
Or merely unfamiliar?

Is it risky?
Or simply unconventional?

Is the risk material?
Is the current context affected?
Is there evidence?
Is the replacement safer?
Is migration worth the cost?
```

This prevents cargo-cult engineering.

---

# 476. Principal Review — “Should We Fix It?”

Use:

```text
Expected failure cost
×
likelihood
```

against:

```text
migration cost
+
regression risk
+
operational disruption
```

The answer is not always “yes.”

---

# 477. Principal Review — “Why Now?”

Strong reasons include:

```text
security exposure
unsupported platform
reliability trend
measured performance problem
maintenance bottleneck
dependency retirement
business expansion
```

Weak reason:

```text
“the code looks old.”
```

---

# 478. Principal Review — “How Do We Know?”

Evidence can include:

```text
incident history
production metrics
benchmark
security assessment
customer distribution
dependency support
runtime lifecycle
```

---

# 479. Principal Review — “What If We Do Nothing?”

A good decision memo includes the counterfactual:

```text
cost of inaction
```

Otherwise the migration may be optimized for engineering aesthetics rather than business value.

---

# 480. Final Anti-Pattern Master Drill

Take a real subsystem and identify:

```text
5 state risks
5 async risks
5 memory risks
5 performance risks
5 security risks
5 reliability risks
5 architecture risks
5 compatibility risks
```

For each:

```text
Pattern
Context
Assumption
Failure mode
Evidence
Severity
Repair
Test
Owner
Removal/Review date
```

Then choose the top five.

Defend the ordering.

---

# Chapter 98 — Revision / Retrieval Record

```md
## Review Record

### Review #
- Date:
- Duration:
- Status before:
- Status after:

### Retrieval
- Could I define anti-pattern precisely? [ ]
- Could I distinguish an anti-pattern from a style preference? [ ]
- Could I identify hidden assumptions? [ ]
- Could I explain shared mutable state? [ ]
- Could I explain async failure modes? [ ]
- Could I explain memory/resource leaks? [ ]
- Could I explain security anti-patterns? [ ]
- Could I explain distributed-system failure patterns? [ ]
- Could I explain edge/serverless and Wasm traps? [ ]
- Could I build a failure tree? [ ]
- Could I prioritize remediation? [ ]
- Could I justify retaining a legacy pattern? [ ]

### Gaps
-

### New Insights
-

### Follow-up
-
```

---

# Chapter 98 — Canonical References and Source Discipline

Primary references:

1. **ECMAScript Specification**  
   https://tc39.es/ecma262/

2. **MDN JavaScript Reference**  
   https://developer.mozilla.org/en-US/docs/Web/JavaScript

3. **MDN Promise**  
   https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise

4. **MDN EventTarget**  
   https://developer.mozilla.org/en-US/docs/Web/API/EventTarget

5. **MDN Fetch API**  
   https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API

6. **MDN WebAssembly**  
   https://developer.mozilla.org/en-US/docs/WebAssembly

7. **Node.js Documentation**  
   https://nodejs.org/docs/

8. **TC39 Proposals**  
   https://github.com/tc39/proposals

9. **WebAssembly Specifications**  
   https://webassembly.org/specs/

10. **OWASP**  
    https://owasp.org/

Source discipline:

```text
language semantics
→ ECMAScript specification

host behavior
→ relevant Web / Node / platform specification

security
→ authoritative guidance + threat model + testing

performance
→ benchmark + profile + production telemetry

reliability
→ incident evidence + failure modeling + load testing

architecture
→ explicit requirements + trade-off analysis
```

Do not use this chapter as a universal blacklist.
Use it as a reasoning framework.

---

# Chapter 98 — Completion Snapshot

```text
Part XIX — Judgment

Chapter 98 — Anti-Patterns and Failure Modes
[ ] Not Started

Track A — Core Theory
[ ] Anti-pattern definition
[ ] Contextual judgment
[ ] State failures
[ ] Scope failures
[ ] Coercion failures
[ ] Object/prototype failures
[ ] Async failures
[ ] Event failures
[ ] Memory failures
[ ] Performance failures
[ ] Security failures
[ ] Module/dependency failures
[ ] API failures
[ ] Reliability failures
[ ] Observability failures
[ ] Testing failures
[ ] Architecture failures
[ ] Distributed-system failures
[ ] Edge/serverless failures
[ ] Wasm/FFI failures
[ ] Compatibility failures

Track B — Implementation
[ ] Anti-pattern scanner
[ ] Heuristic reviewer
[ ] Minimal reproductions
[ ] Concurrency test
[ ] Retry test
[ ] Cancellation test
[ ] Memory leak reproduction
[ ] Compatibility test
[ ] Security test
[ ] Migration dashboard

Track C — Interview / Reasoning
[ ] Define anti-pattern
[ ] Distinguish smell from defect
[ ] Identify hidden assumption
[ ] Explain failure mechanism
[ ] Quantify severity
[ ] Prioritize remediation
[ ] Design controls
[ ] Defend keeping legacy
[ ] Defend removing legacy
[ ] Review distributed architecture
[ ] Lead principal risk review

Mastery Gate
[ ] Understand
[ ] Explain
[ ] Predict
[ ] Implement
[ ] Debug
[ ] Apply
[ ] Compare
[ ] Defend
```

---

# Completion Criteria

Do not mark this chapter mastered because you memorized a list of anti-pattern names.

You are ready to move forward when you can independently:

1. Define an anti-pattern precisely.
2. Distinguish it from a bug, style preference, legacy constraint, and deliberate trade-off.
3. Identify the assumption behind a suspicious design.
4. Explain the mechanism by which the assumption can fail.
5. Build a minimal reproduction.
6. Identify the blast radius.
7. Prioritize by business/security/reliability impact.
8. Design a repair.
9. Design a regression test.
10. Add prevention/detection/recovery controls.
11. Evaluate performance and memory implications.
12. Evaluate security implications.
13. Evaluate distributed-system implications.
14. Evaluate compatibility implications.
15. Decide when to remove, isolate, constrain, monitor, or accept a pattern.
16. Document an intentional waiver.
17. Lead a structured code review.
18. Lead a production failure-mode review.
19. Defend the decision to senior engineering leadership.
20. Explain what evidence would cause you to change your mind.

---

# Principal Challenge

Perform a full anti-pattern assessment of this platform:

```text
Browser
 ↓
Edge
 ↓
Regional API
 ↓
Queue
 ↓
Worker
 ↓
Database
 ↓
Object Storage
```

The codebase also contains:

```text
legacy JavaScript
Node.js
ESM/CommonJS
Wasm
third-party dependencies
feature flags
shared cache
```

Identify:

```text
10 correctness risks
10 reliability risks
10 security risks
10 performance risks
10 maintainability risks
```

For every finding provide:

```text
Pattern:
Context:
Assumption:
Failure mode:
Evidence:
Severity:
Repair:
Control:
Regression test:
Owner:
```

Then select the five highest-value fixes.

Defend their ordering using:

```text
likelihood
blast radius
business impact
security impact
migration risk
cost of inaction
```

Final question:

> **Which apparently “bad” pattern would you intentionally keep, and why?**

A principal answer must show that engineering judgment is not the elimination of imperfection.
It is the management of trade-offs under real constraints.

---

# Final Reference Card

```text
Anti-pattern
=
recurring approach whose trade-offs
systematically create undesirable outcomes
in a given context

Review loop:

Observe
→ Explain
→ Measure
→ Classify
→ Prioritize
→ Repair
→ Verify
→ Monitor

High-risk signals:

global mutable state
hidden I/O
hidden async
implicit lifecycle
unbounded resources
unbounded concurrency
retry storms
missing cancellation
listener leaks
unbounded caches
prototype hazards
unsafe dynamic code
security-sensitive fallbacks
ambiguous temporal data
non-idempotent mutations
provider assumptions
raw Wasm ABI
browser sniffing
shared database writes
architecture by buzzword

Principal rule:

Do not optimize for “clean code” as an isolated goal.
Optimize for:

correctness
security
reliability
performance
maintainability
operability
business value

with evidence.
```

> **Mastery reminder:** Reading alone does not mark completion. Mastery requires retrieval, prediction, implementation, debugging, application, comparison, and defense.