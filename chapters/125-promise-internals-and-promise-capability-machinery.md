# Chapter 125 — Promise Internals & Promise Capability Machinery

> **JavaScript Mastery — Part XXII: Advanced JavaScript Language & Platform**
>
> **Mission:** Understand the ECMAScript specification machinery behind Promises: PromiseCapability Records, resolving functions, reaction records, reaction jobs, thenable assimilation, settlement, chaining, combinators, species construction, and host rejection tracking.
>
> **Role perspective:** Principal JavaScript Engineer · ECMAScript Language Specialist · Runtime Engineer · Async Systems Engineer · Compiler/Engine Engineer · Production Debugging Specialist
>
> **Status:** `[ ] Not Started`
>
> **Core rule:** **A Promise is not “a callback.” It is a stateful synchronization object whose resolution machinery determines when and how reactions become jobs.**

---

# 1. Learning Objectives

By completing this chapter, you should be able to:

```text
[ ] explain the internal Promise model
[ ] distinguish Promise state from Promise capability
[ ] explain PromiseCapability Records
[ ] explain the [[Promise]], [[Resolve]], and [[Reject]] fields
[ ] explain NewPromiseCapability
[ ] explain CreateResolvingFunctions
[ ] explain the once-only settlement rule
[ ] distinguish resolving a Promise from fulfilling it immediately
[ ] explain thenable assimilation
[ ] explain Promise Resolution Procedure concepts
[ ] explain FulfillPromise
[ ] explain RejectPromise
[ ] explain PromiseReaction Records
[ ] explain PerformPromiseThen
[ ] explain TriggerPromiseReactions
[ ] explain NewPromiseReactionJob
[ ] trace a `.then()` chain at specification level
[ ] explain why handlers run asynchronously even for fulfilled Promises
[ ] explain propagation when handlers are absent
[ ] explain how handler return values settle derived Promises
[ ] explain thenable-return behavior
[ ] explain rejection propagation
[ ] explain finally semantics
[ ] explain constructor/SpeciesConstructor interaction
[ ] explain Promise.resolve identity behavior
[ ] explain Promise combinator architecture
[ ] explain HostPromiseRejectionTracker
[ ] distinguish language-level Promise semantics from host scheduling
[ ] reason about memory/retention in Promise chains
[ ] debug Promise timing and rejection failures
[ ] implement a pedagogical Promise model from scratch
[ ] recognize engine implementation details that are not ECMAScript guarantees
```

---

# 2. Prerequisites

You should already understand:

```text
Chapter 09 — Functions / First-Class Behavior
Chapter 10 — Scope / Lexical Environments / Identifier Resolution
Chapter 12 — Execution Contexts / Execution Model
Chapter 13 — Closures
Chapter 29 — Errors / Error Handling
Chapter 31 — Async Fundamentals
Chapter 32 — ECMAScript Jobs / Promise Reactions
Chapter 35 — Promises
Chapter 36 — Async / Await
Chapter 37 — Cancellation / Abort
Chapter 38 — Async Iteration / Streaming
Chapter 41 — Specification Architecture
Chapter 42 — Abstract Operations
Chapter 44 — Realms / Agents / Execution Isolation
Chapter 124 — Execution Records, Completion Records & References
```

---

# 3. Why Promise Internals Matter

Most application code uses:

```js
promise.then(...)
promise.catch(...)
await promise
```

without needing the specification machinery.

But difficult behavior requires a deeper model.

Examples:

```js
Promise.resolve(thenable)
p.then(handler)

return anotherPromise

throw inside handler

Promise.resolve(existingPromise)

finally()

multiple handlers

late rejection handlers

custom Promise subclasses
```

A shallow mental model often produces statements such as:

```text
"Promises run callbacks later."
```

A stronger model asks:

```text
What state is stored?

Which resolving function was created?

Who owns the capability?

Which reactions are registered?

When is a reaction enqueued?

What Promise becomes the result?

What happens if the handler returns a thenable?

Which host observes an unhandled rejection?
```

---

# 4. Specification Source Discipline

The ECMAScript specification defines Promise machinery using abstract records and operations including:

```text
PromiseCapability Records
PromiseReaction Records
CreateResolvingFunctions
FulfillPromise
RejectPromise
NewPromiseCapability
PromiseResolve
PerformPromiseThen
TriggerPromiseReactions
NewPromiseReactionJob
NewPromiseResolveThenableJob
```

The current ECMAScript specification explicitly defines these abstractions rather than exposing them as JavaScript objects. citeturn881674search0turn881674search1

Important distinction:

```text
ECMAScript specification
≠
V8 implementation
≠
Node.js event loop
≠
browser event loop
```

---

# 5. Mental Model

Use this chain:

```text
new Promise(executor)
        ↓
PromiseCapability
        ↓
Promise + resolve + reject
        ↓
Promise state
        ↓
then/catch/finally
        ↓
PromiseReaction
        ↓
Promise Job
        ↓
handler execution
        ↓
derived Promise settlement
        ↓
next reaction
```

And for assimilation:

```text
resolve(x)
   ↓
Is x already this Promise?
   ↓ no
Is x a native Promise/object?
   ↓
get then
   ↓
is then callable?
   ↓
yes → enqueue resolve-thenable job
   ↓
invoke then
   ↓
first effective resolve/reject wins
   ↓
settlement
```

---

# 6. Promise State vs Promise Capability

These are different concepts.

A Promise object represents an eventual outcome.

A PromiseCapability Record groups:

```text
[[Promise]]
[[Resolve]]
[[Reject]]
```

Conceptually:

```text
PromiseCapability
├── promise
├── resolve
└── reject
```

The capability gives an algorithm the ability to:

```text
create/obtain the Promise
settle it
```

A Promise object by itself does not expose:

```js
promise.resolve(...)
promise.reject(...)
```

as public instance methods.

---

# 7. PromiseCapability Record

The specification uses a PromiseCapability Record to store:

```text
[[Promise]]
[[Resolve]]
[[Reject]]
```

These fields let Promise-producing algorithms coordinate:

```text
the Promise result
+
the resolving functions
```

This is central to:

```text
Promise constructor
Promise.prototype.then
Promise.resolve
Promise.all
Promise.allSettled
Promise.race
Promise.any
```

and related machinery. citeturn881674search0turn881674search2

---

# 8. Why Capabilities Exist

Consider:

```js
const p = new Promise((resolve, reject) => {
  // executor
});
```

The constructor must internally have access to:

```text
p
resolve
reject
```

as a coordinated set.

The capability abstraction lets specification algorithms pass these together.

This is especially useful for combinators.

For example:

```js
Promise.all(iterable)
```

must produce:

```text
one result Promise
```

while internally coordinating:

```text
many input reactions
```

A PromiseCapability Record gives the algorithm the output Promise plus the ability to settle it.

---

# 9. NewPromiseCapability

Conceptually:

```text
NewPromiseCapability(C)
```

creates a capability for constructor:

```text
C
```

The operation needs to obtain:

```text
promise
resolve function
reject function
```

The constructor is not assumed to be the intrinsic `%Promise%`.

This matters for:

```text
Promise subclasses
constructor species behavior
custom Promise constructors
```

The specification's `Promise.prototype.then` obtains a constructor and then creates a result capability before performing the reaction setup. citeturn881674search1

---

# 10. Capability Construction Is Not the Same as Settlement

This distinction is crucial.

```text
NewPromiseCapability
```

creates the coordination machinery.

It does not mean:

```text
the Promise is fulfilled.
```

Likewise:

```text
resolve(value)
```

does not necessarily mean:

```text
fulfill immediately with value.
```

Resolving can initiate:

```text
thenable assimilation
```

or:

```text
adoption of another Promise's eventual state
```

---

# 11. CreateResolvingFunctions

The Promise constructor creates resolving functions.

Conceptually:

```text
CreateResolvingFunctions(promise)
        ↓
resolve function
reject function
```

The pair has internal state ensuring:

```text
only the first effective settlement wins
```

This is why:

```js
const p = new Promise((resolve, reject) => {
  resolve(1);
  resolve(2);
  reject(3);
});
```

does not produce three settlements.

Only one final state is observed.

---

# 12. Once-Only Settlement

Promise settlement obeys:

```text
pending
→ fulfilled
```

or:

```text
pending
→ rejected
```

but not:

```text
fulfilled
→ rejected
```

and not:

```text
rejected
→ fulfilled
```

The resolving machinery ensures subsequent settlement attempts have no effect on the already-settled Promise.

---

# 13. Resolve vs Fulfill

Do not treat:

```text
resolve
```

and:

```text
fulfill
```

as synonyms.

For example:

```js
const thenable = {
  then(resolve) {
    resolve(42);
  }
};

const p = new Promise(resolve => {
  resolve(thenable);
});
```

Calling:

```text
resolve(thenable)
```

does not necessarily fulfill `p` with:

```text
thenable
```

Instead, Promise resolution can assimilate the thenable.

Conceptually:

```text
resolve(x)
→ inspect x
→ if x is thenable, follow its eventual state
```

---

# 14. Promise Resolution

Promise resolution is a process.

It can involve:

```text
identity check
self-resolution rejection
object/then lookup
thenable assimilation
recursive resolution
eventual fulfillment
eventual rejection
```

Therefore:

```text
resolved
```

is not always identical to:

```text
fulfilled
```

A Promise may be:

```text
resolved to another pending Promise
```

while still not being fulfilled itself.

---

# 15. Self-Resolution

Consider:

```js
let resolveOuter;

const p = new Promise(resolve => {
  resolveOuter = resolve;
});

resolveOuter(p);
```

A Promise cannot resolve to itself.

Conceptually:

```text
promise === resolution
→ reject with TypeError
```

This prevents an impossible recursive state:

```text
p depends on p to settle
```

---

# 16. Thenable Assimilation

JavaScript Promises do not require a value to be a genuine native Promise before adopting its state.

Example:

```js
const thenable = {
  then(resolve, reject) {
    resolve("value");
  }
};

Promise.resolve(thenable).then(console.log);
```

The Promise machinery checks whether the input provides a callable:

```text
then
```

and, if so, adopts the thenable's eventual outcome.

This is a key interoperability design.

---

# 17. Why Thenables Exist

Thenable assimilation allows Promise ecosystems to interoperate with objects implementing:

```js
{
  then(resolve, reject) { ... }
}
```

without requiring:

```text
instanceof Promise
```

This supports interoperability across:

```text
libraries
realms
Promise implementations
legacy async abstractions
```

It also introduces hazards.

---

# 18. Thenable Hazards

A malicious or badly designed thenable can:

```text
call resolve twice
call reject after resolve
throw
return another thenable
never settle
perform side effects while reading then
```

Promise resolution machinery must protect the Promise's state from multiple settlement attempts.

---

# 19. Then Property Access Can Throw

Consider:

```js
const thenable = {};

Object.defineProperty(thenable, "then", {
  get() {
    throw new Error("boom");
  }
});
```

When Promise resolution inspects:

```text
thenable.then
```

the property access itself can fail.

The Promise must turn this failure into the appropriate rejection behavior.

The important lesson:

```text
thenable assimilation is executable behavior
```

not merely:

```text
type checking
```

---

# 20. NewPromiseResolveThenableJob

When Promise resolution determines that a thenable must be assimilated, the specification uses:

```text
NewPromiseResolveThenableJob
```

This produces a Job that later invokes the thenable's `then` behavior.

This matters because:

```text
thenable invocation
```

is not simply:

```text
"run immediately inside resolve"
```

The specification models it through a Promise Job. citeturn881674search2

---

# 21. Promise Reaction Records

A PromiseReaction Record stores how a Promise should react once it settles.

Conceptually it contains information about:

```text
handler
reaction type
result capability
```

The specification describes PromiseReaction Records as records used to store information about how a Promise should react when it becomes resolved or rejected. citeturn881674search0turn881674search1

---

# 22. Reaction Type

A reaction is associated with:

```text
fulfill
```

or:

```text
reject
```

This tells the reaction job which handler to use when the originating Promise settles.

Conceptually:

```text
promise fulfills
→ fulfill reactions

promise rejects
→ reject reactions
```

---

# 23. PerformPromiseThen

A call such as:

```js
p.then(onFulfilled, onRejected)
```

eventually uses:

```text
PerformPromiseThen
```

The specification defines this operation as the core setup for Promise reactions. citeturn881674search1

It:

```text
normalizes handlers
creates/uses result capability
creates reaction records
registers them or schedules them
marks rejection handling where applicable
```

---

# 24. Handler Normalization

If:

```js
p.then(undefined, onRejected);
```

there is no fulfillment handler.

Likewise:

```js
p.catch(onRejected)
```

is fundamentally built around the rejection side of the reaction mechanism.

Missing handlers do not disappear.

They have propagation semantics.

---

# 25. Missing Fulfillment Handler

Consider:

```js
Promise.resolve(10)
  .then(undefined)
  .then(console.log);
```

The missing fulfillment handler causes the fulfillment value to propagate to the derived Promise.

Conceptually:

```text
input fulfills with 10
→ no fulfillment handler
→ derived Promise fulfills with 10
```

---

# 26. Missing Rejection Handler

Consider:

```js
Promise.reject(error)
  .then(onFulfilled)
  .then(...)
```

If a rejection handler is absent:

```text
rejection propagates
```

to the derived Promise.

This is why an error can travel through a long chain without being transformed.

---

# 27. Handler Return Value

Suppose:

```js
Promise.resolve(10)
  .then(value => value * 2);
```

The handler returns:

```text
20
```

The derived Promise fulfills with:

```text
20
```

Conceptually:

```text
handler result
→ Promise resolution of derived Promise
```

Not merely:

```text
store result directly
```

This distinction becomes critical for thenables.

---

# 28. Handler Returns a Promise

Consider:

```js
Promise.resolve(10)
  .then(value => Promise.resolve(value * 2));
```

The derived Promise adopts the returned Promise's eventual state.

Conceptually:

```text
handler returns p2
→ resolve(resultPromise, p2)
→ adopt p2
```

This is why `.then()` chains flatten returned Promises rather than producing:

```text
Promise<Promise<number>>
```

as an observable nested structure.

---

# 29. Handler Throws

Consider:

```js
Promise.resolve(10)
  .then(() => {
    throw new Error("boom");
  });
```

The derived Promise is rejected.

Conceptually:

```text
handler evaluation
→ throw completion
→ reject derived Promise
```

This connects the Promise machinery to:

```text
Chapter 124 — Completion Records
```

---

# 30. Handler Returns Thenable

Consider:

```js
Promise.resolve()
  .then(() => ({
    then(resolve) {
      resolve("done");
    }
  }));
```

The derived Promise does not fulfill with the raw thenable object.

Instead:

```text
handler result
→ Promise resolution
→ thenable assimilation
→ eventual settlement
```

This is one of the most important reasons Promise resolution is more complex than:

```text
"set value."
```

---

# 31. Reaction Jobs

When a settled Promise has reactions, the specification triggers Promise reaction jobs.

Conceptually:

```text
Promise settles
→ TriggerPromiseReactions
→ NewPromiseReactionJob
→ HostEnqueuePromiseJob
```

The current specification states that `TriggerPromiseReactions` creates a Job for each reaction and enqueues it through `HostEnqueuePromiseJob`. citeturn881674search0turn881674search3

---

# 32. TriggerPromiseReactions

Conceptually:

```text
for each reaction:
    create PromiseReactionJob
    enqueue job
```

This means multiple `.then()` handlers attached to the same Promise can correspond to separate reaction jobs.

Example:

```js
const p = Promise.resolve("x");

p.then(() => console.log("A"));
p.then(() => console.log("B"));
```

There are distinct reaction registrations.

---

# 33. NewPromiseReactionJob

A Promise reaction job captures enough information to:

```text
identify the reaction
invoke the correct handler
process the handler result
settle the derived Promise
```

The current specification defines `NewPromiseReactionJob` as a Job Abstract Closure that applies the handler and uses the handler result to resolve/reject the derived Promise. citeturn881674search0turn881674search1

---

# 34. Promise Chaining

Consider:

```js
Promise.resolve(1)
  .then(x => x + 1)
  .then(x => x * 2)
  .then(console.log);
```

Conceptual graph:

```text
P0
 ↓ reaction
P1
 ↓ reaction
P2
 ↓ reaction
P3
```

Each `.then()` generally creates a new Promise capability and a reaction associated with the derived result.

This is why a Promise chain is:

```text
a graph of promises and reactions
```

not merely:

```text
one Promise with many callbacks
```

---

# 35. Promise Chain Trace

For:

```js
const p1 = Promise.resolve(1);

const p2 = p1.then(x => x + 1);

const p3 = p2.then(x => x * 2);
```

Think:

```text
p1 fulfilled
  ↓
reaction R1 queued
  ↓
R1 executes
  ↓
returns 2
  ↓
resolve p2 with 2
  ↓
reaction R2 queued
  ↓
R2 executes
  ↓
returns 4
  ↓
resolve p3 with 4
```

This is the core chaining machinery.

---

# 36. Why `.then()` Is Asynchronous

Even:

```js
const p = Promise.resolve(1);

p.then(() => console.log("later"));

console.log("now");
```

produces:

```text
now
later
```

The handler is not called synchronously simply because:

```text
p is already fulfilled
```

The reaction is scheduled as a Job.

---

# 37. Already-Settled Promise

When `.then()` is attached after settlement:

```text
Promise already settled
        ↓
PerformPromiseThen
        ↓
queue appropriate reaction job
```

The handler still runs through the Promise job mechanism.

This preserves consistent asynchronous reaction behavior.

---

# 38. Multiple Handlers

Example:

```js
const p = Promise.resolve("x");

p.then(() => console.log("A"));
p.then(() => console.log("B"));
p.then(() => console.log("C"));
```

Each registration creates its own reaction.

The conceptual order reflects:

```text
reaction registration order
→ reaction job order
```

for the same settlement event.

---

# 39. Promise Resolution vs Job Scheduling

These are separate ideas.

### Promise resolution

Determines:

```text
what eventual state/value should this Promise adopt?
```

### Job scheduling

Determines:

```text
when registered reactions get a chance to run
```

Mixing these concepts produces incorrect predictions.

---

# 40. Promise State Machine

Use:

```text
Pending
  |
  +---- resolve(ordinary value) ----> Fulfilled
  |
  +---- resolve(thenable) ----------> follow thenable
  |
  +---- reject(reason) -------------> Rejected
```

Once settled:

```text
Fulfilled → remains Fulfilled
Rejected  → remains Rejected
```

Resolution with another pending Promise can delay final fulfillment/rejection.

---

# 41. Promise Resolution vs Fulfillment Example

Conceptually:

```text
P
resolve(Q)
```

If:

```text
Q is pending
```

then:

```text
P becomes resolved to Q's eventual state
```

but may remain pending from the observable state perspective until Q settles.

This is why:

```text
resolved
```

and:

```text
fulfilled
```

must remain distinct concepts.

---

# 42. Rejection

Rejecting a Promise directly stores:

```text
reason
```

and moves the Promise to:

```text
rejected
```

Then:

```text
TriggerPromiseReactions
```

can enqueue rejection reactions.

The host may also be notified about rejection handling through:

```text
HostPromiseRejectionTracker
```

when the Promise transitions through the relevant handled/unhandled states.

---

# 43. HostPromiseRejectionTracker

The host-defined operation:

```text
HostPromiseRejectionTracker(promise, operation)
```

lets the host react to rejection tracking.

The current specification explicitly identifies this as a host-defined operation with operations such as:

```text
"reject"
"handle"
```

for rejection tracking. citeturn881674search0turn881674search1

This is why:

```text
unhandled rejection behavior
```

is not purely an ECMAScript console/logging rule.

The host decides how to surface it.

---

# 44. Language vs Host Boundary

ECMAScript defines:

```text
Promise state
Promise reactions
Promise jobs
Promise resolution
reaction-job creation
host enqueue hook
rejection tracking hook
```

The host decides how it integrates:

```text
HostEnqueuePromiseJob
HostPromiseRejectionTracker
```

with its broader execution model.

Therefore:

```text
"Promise callbacks run in the microtask queue"
```

is a useful host-level description in common environments.

A specification-level explanation should identify the host integration point.

---

# 45. `then` and Result Capability

Conceptually:

```text
p.then(onFulfilled, onRejected)
```

does not simply return:

```text
p
```

It creates/obtains a new Promise capability.

Therefore:

```js
const p2 = p.then(fn);
```

has:

```text
p2
```

as a distinct Promise from:

```text
p
```

This is the foundation of Promise chains.

---

# 46. `catch`

Conceptually:

```js
p.catch(onRejected)
```

is based on:

```js
p.then(undefined, onRejected)
```

The important part is not the surface syntax.

It is:

```text
a rejection reaction
+
new result capability
```

---

# 47. `finally`

`finally()` is more subtle.

```js
p.finally(onFinally)
```

creates a derived Promise whose handler is designed to run regardless of fulfillment/rejection.

The result must preserve the original outcome unless the `finally` callback itself:

```text
throws
```

or:

```text
returns something that causes rejection
```

The specification implements this through Promise reaction machinery rather than as an entirely separate asynchronous abstraction.

---

# 48. `finally` Fulfillment Path

Conceptually:

```text
p fulfills with value V
        ↓
finally handler
        ↓
wait for handler result if necessary
        ↓
restore/propagate V
```

This explains why:

```js
Promise.resolve(10)
  .finally(() => {})
```

still fulfills with:

```text
10
```

---

# 49. `finally` Rejection Path

Conceptually:

```text
p rejects with reason R
        ↓
finally handler
        ↓
if finally succeeds
        ↓
propagate R
```

But:

```js
Promise.reject("original")
  .finally(() => {
    throw "new";
  });
```

results in rejection with:

```text
"new"
```

because the later abrupt outcome replaces the prior continuation.

---

# 50. Promise.resolve

Consider:

```js
Promise.resolve(value)
```

Conceptually:

```text
PromiseResolve(C, x)
```

may return an existing Promise when:

```text
x is a Promise
```

and its constructor identity is appropriate for:

```text
C
```

Otherwise, a new capability can be created and the input is resolved into it.

This is why:

```js
Promise.resolve(existingPromise) === existingPromise
```

can be true for the intrinsic Promise constructor.

---

# 51. Thenable Conversion Through Promise.resolve

Consider:

```js
Promise.resolve(thenable)
```

This is a common way to normalize:

```text
Promise
ordinary value
thenable
```

into Promise behavior.

Conceptually:

```text
ordinary value
→ fulfilled Promise

Promise
→ reuse/adopt appropriately

thenable
→ assimilate
```

---

# 52. Promise Constructor Semantics

The constructor:

```js
new Promise(executor)
```

receives an executor.

Conceptually:

```text
create capability
→ create resolving functions
→ call executor(resolve, reject)
→ if executor throws:
   reject Promise
```

The executor is called synchronously during construction.

But Promise reactions run asynchronously through the job mechanism.

This distinction is fundamental.

---

# 53. Constructor Executor vs Reaction

Consider:

```js
const p = new Promise(resolve => {
  console.log("executor");
  resolve(1);
});

p.then(() => console.log("handler"));

console.log("after");
```

Conceptually:

```text
executor
after
handler
```

Why?

```text
executor call
→ synchronous constructor logic

then handler
→ reaction job
```

---

# 54. Executor Throws

Consider:

```js
new Promise(() => {
  throw new Error("boom");
});
```

The constructor catches the abrupt completion from the executor and rejects the Promise.

Conceptually:

```text
executor
→ throw completion
→ RejectPromise
```

This connects:

```text
Completion Records
```

to:

```text
Promise rejection.
```

---

# 55. Resolve Called With a Thenable

Consider:

```js
new Promise(resolve => {
  resolve({
    then(resolve) {
      resolve(10);
    }
  });
});
```

Important:

```text
executor finishes before thenable's reaction job runs
```

The Promise can remain pending while the thenable is being assimilated.

This is another place where:

```text
resolve
```

is not equivalent to:

```text
fulfill now
```

---

# 56. Thenable With Multiple Calls

Consider:

```js
const thenable = {
  then(resolve, reject) {
    resolve(1);
    resolve(2);
    reject(3);
  }
};
```

The Promise created from this thenable observes only the first effective settlement.

The thenable's own code may still execute all three calls.

The Promise machinery protects its state.

Important distinction:

```text
user callback execution
```

can continue even after:

```text
Promise settlement
```

has become fixed.

---

# 57. Thenable That Throws After Resolve

```js
const thenable = {
  then(resolve) {
    resolve(1);
    throw new Error("ignored by settlement state");
  }
};
```

Once the resolve/reject machinery has already been effectively called, later abrupt completion should not overwrite the already-established settlement.

This is precisely why the resolving functions maintain an internal once-only state.

---

# 58. PromiseReaction Retention

Promise chains can retain references to:

```text
handlers
captured closures
capabilities
input values
errors
```

while pending work remains reachable.

This matters when:

```text
long chains
never-settling Promises
large captured objects
unbounded task creation
```

are involved.

A Promise is not automatically memory-free just because it is “small.”

---

# 59. Never-Settling Promise

Example:

```js
const forever = new Promise(() => {});
```

This Promise never settles.

A reaction attached to it:

```js
forever.then(() => {
  console.log("never");
});
```

can remain pending indefinitely.

If the reaction captures large state:

```js
const large = createLargeObject();

forever.then(() => {
  use(large);
});
```

that state can remain reachable as long as the Promise/reaction chain remains reachable.

---

# 60. Promise Memory vs Garbage Collection

Do not say:

```text
"Promise causes a memory leak."
```

without identifying:

```text
retention path
```

Ask:

```text
Who owns the pending Promise?

Who owns its reactions?

What do the handlers capture?

Will the Promise ever settle?

What references keep the chain alive?
```

---

# 61. Promise Chains and Error Handling

Consider:

```js
doWork()
  .then(step1)
  .then(step2)
  .then(step3);
```

If:

```text
step1 throws
```

then:

```text
step2 is skipped
step3 is skipped
derived chain becomes rejected
```

unless a rejection handler intercepts the failure.

This is reaction propagation.

---

# 62. Branching Chains

Promise chains form graphs.

```js
const p = load();

p.then(a);
p.then(b);
```

is:

```text
          → reaction A
P --------|
          → reaction B
```

not:

```text
P → A → B
```

unless the second handler is attached to A's returned Promise.

---

# 63. Chain vs Branch Example

### Branch

```js
const p = Promise.resolve(1);

const p1 = p.then(a);
const p2 = p.then(b);
```

### Chain

```js
const p = Promise.resolve(1);

const p1 = p.then(a);
const p2 = p1.then(b);
```

The topology is different.

This matters for:

```text
ordering
error flow
memory
side effects
parallelism
```

---

# 64. Promise Combinator Architecture

The combinators:

```text
Promise.all
Promise.allSettled
Promise.race
Promise.any
```

share common design ideas:

```text
output Promise capability
input iteration
per-input reactions
remaining-element tracking
settlement policy
```

The exact algorithms differ.

The architecture is:

```text
many inputs
→ observe each through reactions
→ coordinate one output Promise
```

---

# 65. `Promise.all`

Conceptually:

```text
create result capability
iterate inputs
create per-element result handlers
track remaining count
on all fulfill:
    fulfill output array
on first rejection:
    reject output
```

The output array preserves input position rather than completion order.

This is an important distinction.

---

# 66. `Promise.all` Does Not Cancel Inputs

Consider:

```js
const p = Promise.all([
  slowOperation(),
  failingOperation(),
  anotherSlowOperation()
]);
```

If one input rejects, the aggregate Promise rejects.

That does not inherently mean:

```text
other operations stop
```

The input operations may continue running.

This distinction matters for:

```text
resource usage
side effects
cancellation
```

---

# 67. `Promise.allSettled`

Conceptually:

```text
wait for every input to settle
```

and produce:

```js
{
  status: "fulfilled",
  value: ...
}
```

or:

```js
{
  status: "rejected",
  reason: ...
}
```

The combinator converts many independent outcomes into one structured result.

---

# 68. `Promise.race`

Conceptually:

```text
first input settlement
→ settle output
```

Again:

```text
settling output
```

does not automatically:

```text
cancel remaining operations
```

Cancellation must be separately designed.

---

# 69. `Promise.any`

Conceptually:

```text
first fulfillment
→ fulfill output
```

If every input rejects:

```text
AggregateError
```

is produced.

This makes it useful for:

```text
alternative providers
redundant sources
race-to-success
```

when the application can tolerate losing branches continuing or has explicit cancellation.

---

# 70. Species and Promise Subclasses

Promise methods can interact with constructor selection.

For:

```js
subclassedPromise.then(...)
```

the resulting Promise can depend on:

```text
SpeciesConstructor
```

This matters for:

```text
Promise subclassing
custom Promise constructors
```

and explains why:

```text
every `.then()` always returns an ordinary `%Promise%`
```

is not a completely general statement.

---

# 71. Constructor Identity

Consider:

```js
class MyPromise extends Promise {}

const p = MyPromise.resolve(1);
```

The constructor context influences the produced Promise.

The specification intentionally supports subclass-aware behavior.

This is one reason `NewPromiseCapability` accepts a constructor rather than hardcoding the intrinsic Promise object.

---

# 72. `Promise.resolve` Identity and Subclasses

The identity optimization:

```js
Promise.resolve(p) === p
```

depends on the constructor identity conditions.

Do not generalize it into:

```text
Promise.resolve always returns the same Promise.
```

The normalization operation is constructor-aware.

---

# 73. Foreign Realm Promises

A Promise from another realm may have different constructor identity from:

```text
the current realm's Promise
```

This is another reason:

```text
constructor identity
```

matters when reasoning about normalization and capability creation.

This connects Promise internals to:

```text
Chapter 44 — Realms / Agents / Execution Isolation
```

---

# 74. Cross-Realm Thenables

Thenable assimilation also provides an interoperability mechanism across realms.

Instead of requiring:

```text
same Promise intrinsic
```

the resolution process can inspect:

```text
then
```

and assimilate the behavior.

This is powerful but means:

```text
property access and callback execution can cross abstraction boundaries
```

---

# 75. `Promise.withResolvers`

Modern ECMAScript provides:

```js
const { promise, resolve, reject } = Promise.withResolvers();
```

This exposes the same useful shape that Promise capability machinery models:

```text
Promise
resolve
reject
```

to application code.

The API makes capability-style coordination explicit without manually constructing:

```js
new Promise((resolve, reject) => ...)
```

just to capture the functions.

Use it deliberately.

The internal specification concept and the public API are related, but they are not identical abstractions.

---

# 76. `Promise.try`

Modern ECMAScript also provides:

```js
Promise.try(callback)
```

which normalizes:

```text
synchronous callback result
```

and:

```text
synchronous callback throw
```

into Promise settlement behavior.

Conceptually:

```text
call callback
→ normal result → fulfill
→ throw → reject
```

It can simplify APIs that need to normalize sync/async computation into Promise form.

---

# 77. Promise Internals and Async/Await

An `async` function returns a Promise.

Conceptually:

```text
async function call
→ Promise capability
→ execute async body
→ await/return/throw
→ resolve/reject resulting Promise
```

`await` introduces additional asynchronous continuation machinery.

The Promise mechanisms in this chapter provide much of the infrastructure needed to understand that behavior.

---

# 78. Await and Thenable Assimilation

Consider:

```js
async function f() {
  return thenable;
}
```

The returned Promise does not simply fulfill with the thenable object.

Its resolution behavior must account for the value returned from the async function.

This is another reason:

```text
Promise resolution
```

is a foundational primitive.

---

# 79. Promise Jobs and Microtask Terminology

Application developers often say:

```text
"Promise callback goes into the microtask queue."
```

A specification-oriented statement is more precise:

```text
Promise reactions are turned into Jobs and handed to the host's Promise-job enqueue mechanism.
```

The host commonly integrates that with a microtask mechanism.

The distinction matters when comparing:

```text
browser
Node.js
other ECMAScript hosts
```

---

# 80. Reaction Job Environment

A PromiseReactionJob carries a realm association in the specification model.

The job must execute with the appropriate semantic context.

This matters for:

```text
intrinsics
realm-sensitive behavior
cross-realm interactions
```

Do not assume all Promise jobs are just:

```text
anonymous callbacks
```

with no execution metadata.

---

# 81. Promise Error Boundaries

Promise machinery turns errors at several points into rejections.

Potential failure points include:

```text
executor
then property access
thenable invocation
reaction handler
Promise constructor behavior
iterator access in combinators
result constructor/capability creation
```

The exact path matters.

Do not say:

```text
"Promises catch everything."
```

without identifying which algorithm is doing the catching/conversion.

---

# 82. Promise Combinator Iterator Failure

Combinators consume iterables.

Therefore failures can happen while:

```text
obtaining iterator
next()
reading values
```

not only inside the Promises themselves.

This connects Promise internals to:

```text
iterators
iterable protocols
abrupt completion propagation
```

A robust implementation must distinguish:

```text
input iteration failure
```

from:

```text
input Promise rejection
```

---

# 83. `Promise.all` Result Ordering

For:

```js
Promise.all([
  slowA,
  fastB,
  mediumC
]);
```

completion might occur:

```text
B
C
A
```

but output is:

```text
[A-result, B-result, C-result]
```

The combinator stores results by input position.

This is a coordination algorithm, not an execution-order guarantee.

---

# 84. Promise.race Result Ordering

For:

```js
Promise.race([A, B, C])
```

the output corresponds to:

```text
first settlement
```

not:

```text
first input
```

The distinction between:

```text
input order
completion order
result order
```

should remain explicit.

---

# 85. Promise.any Failure Aggregation

When:

```text
every input rejects
```

the output rejection aggregates reasons.

This is different from:

```text
Promise.all
```

whose failure is determined by the first rejection.

The combinator's settlement policy is its defining semantic difference.

---

# 86. Common Misconceptions

### Misconception 1

> “Resolving means fulfilling.”

Correction:

```text
resolution can involve adopting another Promise/thenable.
```

### Misconception 2

> “A Promise stores a callback.”

Correction:

```text
Promise reactions, capabilities, jobs, and state are distinct concepts.
```

### Misconception 3

> “`.then()` runs immediately on resolved Promises.”

Correction:

```text
reaction execution is scheduled as a Promise Job.
```

### Misconception 4

> “Promise rejection automatically cancels work.”

Correction:

```text
settlement propagation is not cancellation.
```

### Misconception 5

> “Promise.all cancels unfinished Promises.”

Correction:

```text
aggregate settlement does not inherently cancel inputs.
```

### Misconception 6

> “Every Promise is the intrinsic Promise.”

Correction:

```text
constructors, subclasses, and realms matter.
```

### Misconception 7

> “Thenable means Promise.”

Correction:

```text
thenable is any suitable object with callable then behavior.
```

### Misconception 8

> “The ECMAScript spec contains a JavaScript microtask queue exactly as engines implement it.”

Correction:

```text
Promise Jobs are handed to host enqueue machinery; host integration matters.
```

---

# 87. Common Mistakes

```text
[ ] conflating resolve with fulfill
[ ] ignoring thenable assimilation
[ ] forgetting self-resolution
[ ] forgetting then-property access can throw
[ ] assuming multiple resolve calls change state
[ ] assuming handler return values bypass Promise resolution
[ ] assuming returned Promises remain nested
[ ] forgetting missing handlers propagate outcomes
[ ] confusing branch and chain topology
[ ] assuming Promise.all cancels inputs
[ ] confusing result order with completion order
[ ] ignoring constructor/species behavior
[ ] assuming unhandled rejection policy is purely ECMAScript
[ ] treating specification records as literal runtime objects
```

---

# 88. Comparison With Related Concepts

| Concept | Main Question |
|---|---|
| Promise state | What eventual state/value does this Promise have? |
| PromiseCapability | Who can create/settle the associated Promise? |
| Resolving functions | How is a Promise resolution attempt processed? |
| PromiseReaction | What should happen when the Promise settles? |
| Promise Job | When/how is a reaction handler executed? |
| Thenable | What object can participate in Promise resolution? |
| Completion Record | How did synchronous evaluation finish? |
| Host enqueue mechanism | How does the host schedule Promise Jobs? |
| Cancellation | How does the application stop or abandon work? |

---

# 89. Performance Considerations

Promise-heavy systems may incur costs from:

```text
Promise allocations
reaction records
closures
job scheduling
context capture
thenable assimilation
chained continuations
error construction
combinator bookkeeping
```

Do not assume:

```text
fewer `.then()` calls
```

always means:

```text
measurably faster system.
```

Measure:

```text
throughput
latency
allocation
GC
microtask/job volume
```

under representative workloads.

---

# 90. Memory Considerations

Potential retention sources include:

```text
pending Promises
reaction handlers
closures
captured state
Promise chains
pending combinators
never-settling thenables
unresolved async tasks
```

The right question is:

```text
What is still reachable?
```

not:

```text
"How many Promises exist?"
```

---

# 91. Security Considerations

Thenable execution means Promise resolution can invoke attacker-controlled behavior if the input is untrusted.

For example:

```js
Promise.resolve(untrustedObject);
```

may cause:

```text
property access for "then"
```

and potentially:

```text
execution of its then method
```

Do not blindly treat Promise normalization as inert data conversion.

This matters when processing:

```text
plugins
third-party objects
cross-realm values
untrusted library boundaries
```

---

# 92. Production Usage

Understanding Promise internals improves:

```text
retry systems
HTTP clients
job queues
database workflows
event-driven systems
cancellation
concurrency control
async testing
observability
incident debugging
```

It is especially useful when diagnosing:

```text
unexpected ordering
hung operations
rejection storms
promise chains that never settle
memory retention
duplicate work
```

---

# 93. Implementation From Scratch — Pedagogical Promise

Implement a simplified Promise-like type:

```js
class MiniPromise {
  constructor(executor) {}

  then(onFulfilled, onRejected) {}

  catch(onRejected) {}

  static resolve(value) {}

  static reject(reason) {}
}
```

Required conceptual state:

```text
pending
fulfilled
rejected
```

Required queues:

```text
fulfillment reactions
rejection reactions
```

Required transitions:

```text
resolve
reject
settle
schedule reactions
```

Do not claim this is a standards-complete Promise.

It is a learning implementation.

---

# 94. Mini Promise — Capability Model

Implement an internal structure such as:

```js
{
  promise,
  resolve,
  reject
}
```

This should mirror the conceptual PromiseCapability model.

Then implement:

```text
createCapability()
```

which returns the three coordinated components.

---

# 95. Mini Promise — Once-Only Settlement

Implement:

```js
let alreadyResolved = false;
```

inside the resolving machinery.

Then test:

```js
resolve(1);
resolve(2);
reject(3);
```

and:

```js
reject(1);
resolve(2);
```

The first effective settlement wins.

---

# 96. Mini Promise — Reaction Jobs

Do not run handlers synchronously.

Use an asynchronous scheduling mechanism available in your chosen environment.

Document that:

```text
this scheduling mechanism is a host-level approximation
```

of the ECMAScript Promise Job integration.

The exercise is about semantic structure, not reproducing an engine.

---

# 97. Mini Promise — Thenable Assimilation

Implement a simplified path:

```text
resolve(x)
→ if x is MiniPromise:
    adopt
→ else if object/function:
    get then
→ if then callable:
    schedule assimilation
→ else:
    fulfill
```

Handle:

```text
then getter throws
then calls resolve
then calls reject
then calls both
then never settles
```

---

# 98. Mini Promise — Chaining

Implement:

```js
mini.then(x => x + 1)
```

so that the returned Promise:

```text
p2
```

settles according to the handler result.

Support:

```text
ordinary value
Promise-like result
thenable result
throw
```

---

# 99. Mini Promise — `catch` and `finally`

Implement:

```js
catch(onRejected)
finally(onFinally)
```

using your existing reaction machinery.

Do not duplicate the entire Promise implementation.

The goal is to demonstrate:

```text
composition over duplication
```

---

# 100. Mini Combinators

Implement simplified:

```text
MiniPromise.all
MiniPromise.allSettled
MiniPromise.race
MiniPromise.any
```

Test:

```text
empty input
single input
mixed values/Promises/thenables
multiple rejection
different completion order
```

---

# 101. Implementation Progression

### Guided

Implement:

```text
state
settlement
basic then
```

### Partially Guided

Add:

```text
reaction queues
chaining
```

### No Reference

Implement:

```text
thenable assimilation
combinators
```

from written requirements.

### Edge-Case Hardened

Handle:

```text
self-resolution
multiple calls
throwing handlers
throwing thenables
never-settling thenables
iterator failures
```

### Production-Grade Learning Version

Add:

```text
deterministic tests
diagnostic tracing
benchmark
fuzzing
memory analysis
```

---

# 102. Execution Walkthrough — Simple Chain

Code:

```js
const p = Promise.resolve(1);

const q = p.then(x => x + 1);

q.then(console.log);
```

Trace:

```text
1. Create/obtain p.
2. p is fulfilled with 1.
3. `.then()` creates result capability q.
4. A fulfillment reaction is registered or scheduled.
5. Reaction job runs.
6. Handler receives 1.
7. Handler returns 2.
8. q is resolved with 2.
9. q's reaction is triggered.
10. Another reaction job runs.
11. console.log receives 2.
```

The important concept:

```text
p and q are separate Promise state machines connected by reactions.
```

---

# 103. Execution Walkthrough — Throwing Handler

Code:

```js
const p = Promise.resolve(1);

const q = p.then(() => {
  throw new Error("boom");
});

q.catch(error => {
  console.log(error.message);
});
```

Trace:

```text
p fulfilled
→ reaction queued
→ handler runs
→ handler throws
→ reaction job observes abrupt completion
→ q rejected
→ q rejection reaction queued
→ catch handler runs
```

This is:

```text
Completion
→ Promise rejection
→ Reaction
```

---

# 104. Execution Walkthrough — Returned Promise

```js
const p = Promise.resolve(1);

const q = p.then(() => new Promise(resolve => {
  setTimeout(() => resolve(2), 100);
}));
```

Trace:

```text
p fulfilled
→ handler runs
→ handler creates p2
→ handler returns p2
→ q resolves to p2's eventual state
→ q remains pending while p2 remains pending
→ p2 fulfills
→ q becomes fulfilled
```

This is why:

```text
return Promise
```

flattens through Promise resolution.

---

# 105. Execution Walkthrough — Thenable

```js
const q = Promise.resolve(1).then(() => ({
  then(resolve) {
    resolve(2);
  }
}));
```

Trace:

```text
handler runs
→ returns thenable
→ resolve q with thenable
→ inspect then
→ create resolve-thenable job
→ invoke then
→ thenable calls resolve(2)
→ q fulfills with 2
```

The thenable is part of the resolution protocol.

---

# 106. Debugging Exercises

## Exercise A — “Why is this still pending?”

```js
const p = Promise.resolve({
  then() {}
});
```

Explain why the Promise can remain pending.

Identify:

```text
thenable
non-settlement
```

---

## Exercise B — “Why didn't the second resolve win?”

```js
new Promise(resolve => {
  resolve(1);
  resolve(2);
});
```

Trace the resolving-function state.

---

## Exercise C — “Why is the handler later?”

```js
const p = Promise.resolve();

p.then(() => console.log("handler"));

console.log("sync");
```

Explain the Promise Job boundary.

---

## Exercise D — “Why didn't Promise.all stop the work?”

```js
Promise.all([
  slow(),
  failFast(),
  slowAgain()
]).catch(handle);
```

Explain aggregate rejection vs cancellation.

---

# 107. Code Review Exercise

Review:

```js
function normalize(value) {
  if (value instanceof Promise) {
    return value;
  }

  return Promise.resolve(value);
}
```

Questions:

```text
Does instanceof correctly identify all thenables?

What about cross-realm Promises?

What about Promise subclasses?

What about arbitrary thenables?

What about objects with a throwing `then` getter?
```

Then explain why:

```js
Promise.resolve(value)
```

is a stronger normalization mechanism than an `instanceof Promise` check for interoperability.

---

# 108. Interview Questions

### Fundamentals

```text
1. What is a PromiseCapability Record?
2. What is a PromiseReaction Record?
3. What does resolve mean internally?
4. What is the difference between resolve and fulfill?
5. Why can a resolved Promise still appear pending?
```

### Advanced

```text
6. What is thenable assimilation?
7. Why can accessing `then` cause rejection?
8. Why does Promise settlement happen only once?
9. What does PerformPromiseThen do?
10. What does TriggerPromiseReactions do?
11. What does a PromiseReactionJob do?
12. Why does `.then()` create a new Promise?
13. How do handler return values settle the derived Promise?
14. What happens when a handler throws?
15. Why do returned Promises flatten?
```

### Principal

```text
16. How would you model Promise resolution in a runtime?
17. How would you prevent malicious thenables from corrupting Promise state?
18. How would you diagnose a Promise chain that retains large memory?
19. Why is unhandled rejection behavior partly host-specific?
20. What Promise internals matter most when designing a high-throughput async system?
```

---

# 109. Predict-the-Behavior Exercises

Predict before checking.

## Exercise 1

```js
const p = Promise.resolve(1);

console.log("A");

p.then(() => console.log("B"));

console.log("C");
```

Explain the job boundary.

---

## Exercise 2

```js
Promise.resolve(1)
  .then(x => x + 1)
  .then(console.log);
```

Trace the two derived Promise steps.

---

## Exercise 3

```js
Promise.resolve()
  .then(() => {
    throw new Error("x");
  })
  .then(
    () => console.log("fulfilled"),
    () => console.log("rejected")
  );
```

Trace:

```text
reaction
→ abrupt completion
→ derived rejection
→ rejection reaction
```

---

## Exercise 4

```js
const thenable = {
  then(resolve) {
    resolve(42);
  }
};

Promise.resolve(thenable).then(console.log);
```

Identify:

```text
thenable assimilation
resolve-thenable job
final fulfillment
```

---

## Exercise 5

```js
let resolve;

const p = new Promise(r => {
  resolve = r;
});

resolve(p);
```

Classify the result and explain self-resolution.

---

## Exercise 6

```js
Promise.all([
  Promise.resolve("A"),
  new Promise(resolve => setTimeout(() => resolve("B"), 10))
]).then(console.log);
```

Distinguish:

```text
completion timing
result ordering
```

---

# 110. Advanced Mastery Exercises

### Exercise 1 — Draw the Promise Graph

For:

```js
const p2 = p1.then(a);
const p3 = p2.then(b);
const p4 = p1.then(c);
```

draw:

```text
Promises
Reactions
Branches
Chains
```

### Exercise 2 — Resolution Graph

Model:

```text
P → Q → Thenable → R
```

and explain how final settlement propagates.

### Exercise 3 — Rejection Graph

Create a chain where:

```text
first handler throws
second catches
third throws
fourth catches
```

Trace each derived Promise.

### Exercise 4 — Hanging Thenable

Create a thenable that never settles.

Measure:

```text
pending Promise lifetime
captured state
```

### Exercise 5 — Combinator Analyzer

Implement a tool that displays:

```text
input count
settlement order
output settlement
```

for:

```text
all
allSettled
race
any
```

---

# 111. Performance Investigation

Benchmark:

```text
direct synchronous call
Promise.resolve(value)
one `.then()`
ten chained `.then()`
```

Measure:

```text
wall-clock time
throughput
allocations where available
memory
```

Then state:

```text
what the benchmark does not prove
```

Do not infer general runtime performance from one synthetic microbenchmark.

---

# 112. Memory Investigation

Create:

```js
const forever = new Promise(() => {});
```

Attach handlers capturing:

```text
large arrays
large object graphs
closures
```

Then remove all external references that can safely be removed.

Analyze:

```text
what remains reachable
what does not
```

Use heap evidence when available.

---

# 113. Security Investigation

Create an intentionally controlled thenable:

```js
const thenable = {
  get then() {
    // observable side effect
    return resolve => resolve(1);
  }
};
```

Observe:

```text
property access
```

during Promise normalization.

Then document why untrusted thenables should not be assumed to be inert data.

---

# 114. Specification Reading Exercise

Open the ECMAScript Promise section and trace:

```text
PromiseCapability Records
→ NewPromiseCapability
→ CreateResolvingFunctions
→ Promise Resolve Functions
→ FulfillPromise
→ RejectPromise
→ PerformPromiseThen
→ TriggerPromiseReactions
→ NewPromiseReactionJob
```

For each, write:

```text
Input
Output
State change
Possible abrupt completion
Next operation
```

The goal is not memorization.

The goal is fluent specification navigation.

---

# 115. Specification / Runtime Source Discipline

Prefer:

```text
1. ECMAScript Promise abstract operations
2. ECMAScript Promise Jobs
3. host Promise-job integration
4. browser/Node event-loop behavior
5. engine implementation
```

The official ECMAScript specification currently documents PromiseCapability Records, PromiseReaction Records, `PerformPromiseThen`, `TriggerPromiseReactions`, and Promise reaction jobs. citeturn881674search0turn881674search1

When explaining runtime performance, label claims such as:

```text
"V8 may represent this without allocating a visible object"
```

as implementation-specific.

---

# 116. Common Failure Modes

```text
Failure 1:
Treating Promise as just a callback wrapper.

Failure 2:
Confusing resolution with fulfillment.

Failure 3:
Ignoring thenable assimilation.

Failure 4:
Assuming all thenables are trustworthy.

Failure 5:
Forgetting self-resolution.

Failure 6:
Forgetting handler return values are Promise-resolved.

Failure 7:
Ignoring derived Promise creation.

Failure 8:
Confusing branches with chains.

Failure 9:
Confusing aggregate settlement with cancellation.

Failure 10:
Treating host rejection reporting as pure ECMAScript behavior.

Failure 11:
Assuming specification records are literal heap objects.

Failure 12:
Ignoring constructor/species behavior.
```

---

# 117. Principal Decision Framework

When choosing Promise architecture or abstractions, evaluate:

```text
Correctness
Ordering
Failure propagation
Cancellation
Concurrency
Memory retention
Performance
Security
Observability
Testing complexity
Maintainability
Runtime compatibility
```

A Promise abstraction is useful when it preserves:

```text
clear ownership
predictable settlement
explicit failure
bounded work
observable behavior
```

---

# 118. Retrieval Record

```md
# Chapter 125 — Revision / Retrieval Record

## Attempt
- Date:
- Duration:
- Status before:
- Status after:

## Promise State
-

## Capability Model
-

## Resolution
-

## Thenable Assimilation
-

## Reaction Model
-

## Promise Jobs
-

## Chaining
-

## Combinators
-

## Host Boundary
-

## Strongest Areas
-

## Weakest Areas
-

## Specification Reading Difficulty
-

## Implementation Progress
-

## Questions Requiring Rework
-

## Next Review
-
```

---

# 119. Spaced Retrieval Schedule

### Day 0

Study:

```text
capability
resolution
reaction
job
```

and complete the prediction exercises.

### Day 1

Trace:

```text
resolve
→ settle
→ reaction
→ derived Promise
```

for a simple chain.

### Day 3

Explain from memory:

```text
resolve vs fulfill
```

and:

```text
thenable assimilation
```

### Day 7

Implement the mini Promise state machine.

### Day 14

Implement:

```text
all
allSettled
race
any
```

in the learning runtime.

### Day 21

Read the specification algorithms without using a tutorial.

### Day 30

Explain Promise internals in ten minutes from memory.

---

# 120. Dependency Graph

```text
Chapter 12
execution contexts
        ↓
Chapter 29
errors / abrupt outcomes
        ↓
Chapter 32
ECMAScript Jobs / Promise Reactions
        ↓
Chapter 35
Promises
        ↓
Chapter 36
Async / Await
        ↓
Chapter 41
Specification Architecture
        ↓
Chapter 42
Abstract Operations
        ↓
Chapter 44
Realms / Agents
        ↓
Chapter 124
Completion Records / References
        ↓
Chapter 125
Promise Internals / Capability Machinery
        ↓
Chapter 126
Module Linking / Async Module Evaluation
```

---

# 121. Concept Connections

## Depends On

```text
Completion Records
Execution Contexts
Functions
Closures
Jobs
Promises
Errors
Iterables
Constructors
Species
Realms
```

## Builds Toward

```text
async/await internals
module evaluation
engine implementation
host scheduling
concurrency architecture
```

## Related Concepts

```text
Future/Task abstractions
event loops
microtasks
cancellation
observables
async iterators
streams
queues
```

## Concepts Revisited

```text
Completion
Jobs
Closures
Constructors
Species
Realms
Thenables
Error propagation
```

## Why This Chapter Matters

Promises are a central bridge between:

```text
ECMAScript language semantics
```

and:

```text
real asynchronous applications
```

Without the capability/reaction/job model, advanced Promise behavior appears inconsistent.

With it, the system becomes:

```text
state
→ resolution
→ reaction registration
→ job creation
→ handler execution
→ derived settlement
```

---

# 122. Track A — Core Theory

Master:

```text
PromiseCapability
CreateResolvingFunctions
resolve vs fulfill
Promise resolution
thenable assimilation
self-resolution
PromiseReaction
PerformPromiseThen
TriggerPromiseReactions
NewPromiseReactionJob
Promise chaining
finally
combinators
species
host rejection tracking
```

Deliverable:

```text
trace a Promise from creation through settlement and reaction.
```

---

# 123. Track B — Implementation

Build:

```text
MiniPromise
MiniPromiseCapability
MiniPromiseReaction
reaction scheduler
thenable assimilation
chain handling
combinators
```

Deliverable:

```text
show how Promise semantics can be implemented.
```

---

# 124. Track C — Interview / Reasoning

Practice:

```text
"What happens internally when resolve is called?"

"Why does thenable assimilation exist?"

"Why does `.then()` return a new Promise?"

"Why does a handler throw create rejection?"

"Why does Promise.all not cancel inputs?"

"What is the difference between resolution and fulfillment?"

"Where does host behavior enter?"
```

Deliverable:

```text
precise Promise reasoning without hand-waving.
```

---

# 125. Mastery Gate

You may mark:

```text
[+] Completed
```

when:

```text
[ ] capability model understood
[ ] settlement state understood
[ ] resolve vs fulfill understood
[ ] thenables understood
[ ] reactions understood
[ ] reaction jobs understood
[ ] chaining understood
[ ] combinators understood
[ ] host boundary understood
[ ] mini Promise implemented
```

Mark:

```text
[*] Mastered
```

only when you can:

```text
[ ] trace Promise state transitions
[ ] explain thenable assimilation
[ ] explain derived Promise creation
[ ] reason about reaction ordering
[ ] reason about error propagation
[ ] reason about memory retention
[ ] distinguish host scheduling from ECMAScript semantics
[ ] explain combinator architecture
[ ] implement the core model without reference
[ ] read the specification algorithms fluently
```

---

# 126. Completion Snapshot

```md
# Chapter 125 — Completion Snapshot

Status:
[ ] Not Started
[~] In Progress
[?] Needs Revision
[+] Completed
[*] Mastered

Primary Gaps:
-

Promise State:
[ ] weak
[ ] developing
[ ] strong
[ ] principal

Capability Machinery:
[ ] weak
[ ] developing
[ ] strong
[ ] principal

Resolution / Thenables:
[ ] weak
[ ] developing
[ ] strong
[ ] principal

Reaction Jobs:
[ ] weak
[ ] developing
[ ] strong
[ ] principal

Promise Chaining:
[ ] weak
[ ] developing
[ ] strong
[ ] principal

Combinators:
[ ] weak
[ ] developing
[ ] strong
[ ] principal

Specification Reading:
[ ] weak
[ ] developing
[ ] strong
[ ] principal

Mini Promise:
[ ] not started
[ ] partial
[ ] complete
[ ] hardened
```

---

# 127. Completion Criteria

```text
[ ] PromiseCapability explained
[ ] resolving functions explained
[ ] once-only settlement explained
[ ] resolve vs fulfill distinguished
[ ] Promise resolution explained
[ ] self-resolution explained
[ ] thenable assimilation explained
[ ] then-property failure explained
[ ] PromiseReaction Records explained
[ ] PerformPromiseThen explained
[ ] TriggerPromiseReactions explained
[ ] NewPromiseReactionJob explained
[ ] handler return propagation explained
[ ] handler throw propagation explained
[ ] Promise chaining explained
[ ] finally behavior explained
[ ] Promise.resolve behavior explained
[ ] combinator architecture explained
[ ] species/subclass behavior explained
[ ] HostPromiseRejectionTracker understood
[ ] language-vs-host boundary understood
[ ] mini Promise implemented
[ ] combinators implemented
[ ] memory analysis performed
[ ] performance experiment performed
[ ] security boundary considered
```

---

# 128. Final Principal Mental Model

Use:

```text
Promise creation
    ↓
capability
    ↓
resolve/reject functions
    ↓
pending state
    ↓
settlement / resolution
    ↓
reaction records
    ↓
Promise Jobs
    ↓
handlers
    ↓
handler completion/value
    ↓
derived Promise resolution
    ↓
next reactions
```

For a returned object:

```text
handler result
    ↓
ordinary value?
    → fulfill-derived Promise

Promise?
    → adopt its state

thenable?
    → assimilate

throw?
    → reject-derived Promise
```

For rejection:

```text
reject
→ rejected state
→ rejection reactions
→ possible host rejection tracking
```

For combinators:

```text
many inputs
→ many reactions
→ coordination state
→ one output capability
→ aggregate settlement
```

---

# 129. Final Principal Principle

> **Promises are a protocol for eventual state, not merely a syntax for asynchronous callbacks.**

The mature model is:

```text
capability
+
resolution
+
settlement
+
reaction
+
job
+
derived capability
```

Once you understand that pipeline, the behavior of:

```text
then
catch
finally
resolve
reject
all
allSettled
race
any
async
await
thenables
```

stops being a collection of special cases.

It becomes one coherent system:

```text
state
→ resolution
→ reaction
→ job
→ handler
→ derived settlement
```

That model is the foundation for:

```text
advanced async debugging
runtime design
concurrency systems
Promise implementations
engine optimization analysis
specification reading
production reliability
```