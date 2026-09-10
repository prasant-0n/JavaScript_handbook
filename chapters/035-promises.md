
# Chapter 35 — Promises

## 1. Learning Objectives

By the end of this chapter, the learner must be able to:

- Explain what a JavaScript Promise is and what problem it solves.
- Distinguish promise state from promise result and from promise settlement.
- Distinguish resolving a promise from fulfilling or rejecting it.
- Explain the `pending`, `fulfilled`, and `rejected` states.
- Explain the promise resolution procedure.
- Explain why a promise can be resolved while remaining pending.
- Explain thenable assimilation.
- Explain why a promise settles only once.
- Explain the executor function and precisely when it runs.
- Explain `resolve` and `reject` behavior.
- Explain `.then()`, `.catch()`, and `.finally()`.
- Explain that promise methods return new promises rather than mutating the original result chain.
- Explain promise chaining as a transformation pipeline.
- Explain fulfillment propagation and rejection propagation.
- Explain how returned values become downstream fulfillment values.
- Explain how thrown exceptions become downstream rejections.
- Explain how returned promises and thenables are adopted.
- Explain the semantic difference between `Promise.resolve(value)` and simply storing `value`.
- Explain `Promise.reject(reason)`.
- Explain `Promise.all()`, `Promise.allSettled()`, `Promise.race()`, and `Promise.any()`.
- Explain the different failure and completion contracts of the promise combinators.
- Understand why `Promise.all()` does not cancel its remaining operations after one rejection.
- Understand why `Promise.race()` does not cancel losing operations.
- Understand `AggregateError` in relation to `Promise.any()`.
- Explain `Promise.withResolvers()` and when it is useful.
- Explain promise subclassing and constructor/species considerations at a conceptual level.
- Explain common promise anti-patterns.
- Diagnose unhandled and accidentally detached rejections.
- Reason about promise concurrency, memory retention, queueing, and cancellation.
- Implement promise-like abstractions for learning.
- Implement safe concurrency combinators.
- Test promise behavior deterministically.
- Design production promise APIs with explicit completion, failure, ownership, and cancellation contracts.
- Debug complex promise chains without relying on timing intuition.
- Compare promises with callbacks and explicit result objects.
- Explain promises precisely at ECMAScript specification level.
- Defend promise architecture decisions at senior/principal level.

### Mastery Gate

The chapter is mastered only when the learner can:

> Understand → Explain → Predict → Implement → Debug → Apply → Compare → Defend

---

## 2. Prerequisites

The learner should understand:

- Values and types.
- Functions and lexical scope.
- Objects and classes.
- Errors and abrupt completion.
- Async fundamentals.
- ECMAScript Jobs and Promise Reaction Jobs.
- Browser event-loop fundamentals.
- Node event-loop fundamentals.
- Resource management and cleanup.

Primary dependencies:

- Chapter 09 — Functions / First-Class Behavior
- Chapter 12 — Execution Contexts / Execution Model
- Chapter 18 — Classes / OOP
- Chapter 20 — Symbols / Well-Known Symbols
- Chapter 29 — Errors / Error Handling
- Chapter 30 — Resource Management / Cleanup
- Chapter 31 — Asynchronous JavaScript Fundamentals
- Chapter 32 — ECMAScript Jobs / Promise Reactions
- Chapter 33 — Browser Event Loop
- Chapter 34 — Node Event Loop / libuv

Later chapters deepen this topic through:

- Chapter 36 — Async/Await
- Chapter 37 — Cancellation / Abort
- Chapter 38 — Async Iteration / Streaming
- Chapter 39 — Concurrency / Parallelism
- Chapter 63 — Async Context / Diagnostics
- Chapter 84 — Reliability
- Chapter 86 — Testing
- Chapter 87 — Deterministic Async Testing

---

## 3. What Is It?

A **Promise** is a JavaScript object representing the eventual outcome of an asynchronous or deferred computation.

A promise has one of three states:

```text
pending
fulfilled
rejected
```

Once it becomes fulfilled or rejected, it is **settled**.

```text
pending
  │
  ├── fulfill(value) ──→ fulfilled
  │
  └── reject(reason) ──→ rejected
```

The core idea is:

```text
“Here is an object representing a future result.”
```

This allows asynchronous operations to be composed.

Instead of callback nesting:

```js
getUser(id, (error, user) => {
  if (error) {
    // ...
    return;
  }

  getProfile(user, (error, profile) => {
    // ...
  });
});
```

promise composition can express:

```js
getUser(id)
  .then(user => getProfile(user))
  .then(profile => {
    // ...
  })
  .catch(error => {
    // ...
  });
```

A promise provides:

- eventual success/failure;
- compositional chaining;
- standardized reaction behavior;
- error propagation;
- multiple consumers;
- concurrency combinators.

But a Promise is **not**:

- a thread;
- a scheduler by itself;
- cancellation;
- a network request;
- guaranteed parallel execution;
- a guarantee that underlying work has even started.

The underlying operation and the promise representing its result are separate concepts.

---

## 4. Why Does It Exist?

Callback APIs become difficult to compose as systems become deeper.

Typical problems include:

```text
nested control flow
error propagation duplication
multiple callback invocation
unclear ownership
difficult composition
manual result forwarding
```

Promises introduce a normalized abstraction:

```text
operation
   ↓
Promise
   ↓
reaction
   ↓
new Promise
   ↓
reaction
   ↓
...
```

This enables:

```js
const user = await getUser(id);
```

and:

```js
Promise.all([
  fetchProfile(),
  fetchPermissions(),
  fetchSettings()
]);
```

The deeper design goal is not merely avoiding callback nesting.

It is:

> Represent asynchronous completion as a composable value-like abstraction.

This enables higher-level reasoning about:

- dependencies;
- sequencing;
- concurrency;
- errors;
- cleanup;
- cancellation;
- aggregation.

---

## 5. Mental Model

Think of a Promise as a small state machine plus a reaction graph.

```text
                    ┌─────────────┐
                    │   pending   │
                    └──────┬──────┘
                           │
                  ┌────────┴────────┐
                  │                 │
                  ▼                 ▼
             fulfilled          rejected
                value              reason
                  │                 │
                  └───────┬─────────┘
                          ▼
                    reactions
                          │
                          ▼
                  downstream promises
```

For:

```js
const q = p.then(onFulfilled, onRejected);
```

think:

```text
p
│
├── reaction
│      ├── onFulfilled
│      ├── onRejected
│      └── capability for q
│
└── q
```

The downstream promise `q` depends on the reaction's eventual completion.

A second important model:

```text
resolve(value)
```

does **not** always mean:

```text
fulfilled with value
```

If `value` is a promise/thenable, the promise can become **resolved to follow that value** while still being pending.

Therefore:

```text
resolved
```

and:

```text
settled
```

are not synonyms.

This distinction is one of the most important promise concepts.

---

## 6. Core Rules

### Rule 1 — A promise has one settlement

After fulfillment or rejection:

```text
future resolve/reject attempts do not replace the settled result
```

### Rule 2 — A promise can be resolved before it is fulfilled

Example:

```js
new Promise(resolve => {
  resolve(new Promise(resolveInner => {
    setTimeout(() => resolveInner("done"), 100);
  }));
});
```

The outer promise follows the inner promise.

It is resolved to that promise but remains pending until the inner promise settles.

### Rule 3 — The executor runs synchronously

```js
new Promise(() => {
  console.log("runs now");
});
```

The executor is called during construction.

### Rule 4 — Promise reactions run asynchronously

```js
Promise.resolve().then(() => {
  console.log("later");
});
```

The reaction is scheduled rather than called inline.

### Rule 5 — `.then()` creates a new promise

```js
const q = p.then(handler);
```

`q` is distinct from `p`.

### Rule 6 — Returning a normal value fulfills the downstream promise

```js
p.then(() => 42);
```

The returned promise eventually fulfills with `42`.

### Rule 7 — Throwing in a handler rejects the downstream promise

```js
p.then(() => {
  throw error;
});
```

### Rule 8 — Returning a promise adopts its outcome

```js
p.then(() => otherPromise);
```

### Rule 9 — Missing handlers propagate the original outcome

Conceptually:

```js
p.then()
```

passes fulfillment/rejection onward.

### Rule 10 — `.catch(onRejected)` is rejection handling through promise chaining

Conceptually:

```js
p.catch(onRejected)
```

is equivalent to:

```js
p.then(undefined, onRejected)
```

### Rule 11 — `.finally()` is outcome-transparent when it succeeds

```js
p.finally(cleanup)
```

normally preserves the original fulfillment/rejection.

If cleanup throws or rejects, the resulting promise rejects instead.

### Rule 12 — `Promise.all()` fails fast on the first observed rejection

The returned promise rejects when an input rejects.

Remaining input operations continue unless separately cancelled.

### Rule 13 — `Promise.allSettled()` waits for every input to settle

It gives a result record for each input.

### Rule 14 — `Promise.race()` settles from the first input settlement

It may fulfill or reject depending on which settles first.

It does not cancel the losers.

### Rule 15 — `Promise.any()` fulfills from the first successful fulfillment

It rejects only when all inputs reject, using `AggregateError`.

### Rule 16 — Combinators consume iterables

The input need not literally be an array.

### Rule 17 — Non-promise values are normalized

Combinators and promise resolution can adopt ordinary values as completed inputs.

### Rule 18 — `Promise.resolve(promise)` can return the same promise

For an appropriate native Promise of the relevant constructor, no unnecessary new promise is required.

### Rule 19 — `Promise.reject(reason)` creates a rejected promise

It does not throw synchronously to the caller.

### Rule 20 — Promise settlement does not cancel underlying work

A promise has no universal cancellation primitive.

Cancellation must be designed separately.

---

## 7. Syntax

### Constructor

```js
const promise = new Promise((resolve, reject) => {
  // asynchronous setup
});
```

### Resolve

```js
resolve(value);
```

### Reject

```js
reject(error);
```

### Then

```js
promise.then(onFulfilled, onRejected);
```

### Catch

```js
promise.catch(onRejected);
```

### Finally

```js
promise.finally(onFinally);
```

### Resolve helper

```js
Promise.resolve(value);
```

### Reject helper

```js
Promise.reject(reason);
```

### Aggregation

```js
Promise.all(iterable);
Promise.allSettled(iterable);
Promise.race(iterable);
Promise.any(iterable);
```

### External resolver access

Modern ECMAScript provides:

```js
const { promise, resolve, reject } = Promise.withResolvers();
```

This is useful when the resolution events are controlled outside a single Promise constructor executor.

### Promise subclassing

```js
class MyPromise extends Promise {}
```

Promise instance methods participate in constructor/species behavior.

Use subclassing only when there is a strong semantic reason.

---

## 8. Basic Examples

### Example 1 — Basic fulfillment

```js
const promise = new Promise(resolve => {
  resolve(42);
});

promise.then(value => {
  console.log(value);
});
```

Output:

```text
42
```

### Example 2 — Basic rejection

```js
const promise = Promise.reject(new Error("failed"));

promise.catch(error => {
  console.log(error.message);
});
```

### Example 3 — Chaining

```js
Promise.resolve(10)
  .then(value => value * 2)
  .then(value => value + 5)
  .then(console.log);
```

Result:

```text
25
```

### Example 4 — Throw becomes rejection

```js
Promise.resolve()
  .then(() => {
    throw new Error("boom");
  })
  .catch(error => {
    console.log(error.message);
  });
```

### Example 5 — Return promise

```js
Promise.resolve()
  .then(() => Promise.resolve("done"))
  .then(console.log);
```

The final handler receives:

```text
done
```

### Example 6 — `finally`

```js
Promise.resolve("value")
  .finally(() => {
    console.log("cleanup");
  })
  .then(value => {
    console.log(value);
  });
```

Output:

```text
cleanup
value
```

### Example 7 — `Promise.all`

```js
const results = await Promise.all([
  getUser(),
  getSettings()
]);
```

All must fulfill.

### Example 8 — `Promise.allSettled`

```js
const results = await Promise.allSettled([
  taskA(),
  taskB(),
  taskC()
]);
```

Every operation gets a final result record.

### Example 9 — `Promise.any`

```js
const fastestSuccess = await Promise.any([
  replicaA(),
  replicaB(),
  replicaC()
]);
```

The first fulfillment wins.

### Example 10 — `Promise.race`

```js
const result = await Promise.race([
  operation(),
  timeoutPromise()
]);
```

Important:

```text
timeout winning does not stop operation()
```

---

## 9. Execution Walkthrough

Consider:

```js
const p = new Promise(resolve => {
  console.log("executor");
  resolve(1);
});

console.log("after");

p.then(value => {
  console.log(value);
});
```

### Step 1

Promise construction begins.

### Step 2

The executor runs immediately:

```text
executor
```

### Step 3

`resolve(1)` resolves the promise.

Because `1` is an ordinary value, the promise can fulfill with `1`.

### Step 4

The constructor returns the promise.

### Step 5

Synchronous code prints:

```text
after
```

### Step 6

`.then()` registers a reaction.

The source promise is already settled, so its reaction is scheduled rather than executed inline.

### Step 7

The reaction job executes and prints:

```text
1
```

Final output:

```text
executor
after
1
```

This demonstrates three separate events:

```text
executor execution
promise settlement
reaction execution
```

---

## 10. Internal Mechanics

### 10.1 Promise state

A Promise has internal state conceptually equivalent to:

```text
[[PromiseState]]
[[PromiseResult]]
[[PromiseFulfillReactions]]
[[PromiseRejectReactions]]
```

The precise internal-slot model is defined by ECMAScript.

### 10.2 Pending state

While pending:

```text
state = pending
result = no final outcome
reactions = retained
```

Handlers registered by `.then()` remain associated with the promise.

### 10.3 Fulfillment

When fulfilled:

```text
state = fulfilled
result = value
```

relevant fulfillment reactions become eligible for reaction-job scheduling.

### 10.4 Rejection

When rejected:

```text
state = rejected
result = reason
```

relevant rejection reactions become eligible.

### 10.5 Reaction records

Conceptually:

```text
reaction:
  type
  handler
  capability
```

The capability connects reaction execution to the downstream promise.

### 10.6 Promise capability

A promise capability contains conceptually:

```text
promise
resolve
reject
```

This is why promise machinery can connect callback execution to the promise returned from `.then()`.

### 10.7 Resolver functions

The resolving functions created for a promise protect the settlement invariant.

Conceptually:

```text
already resolved?
   yes → ignore later attempt
   no  → mark resolved and process value
```

### 10.8 Resolution versus settlement

This deserves explicit emphasis.

```js
let resolveOuter;

const outer = new Promise(resolve => {
  resolveOuter = resolve;
});

const inner = new Promise(resolve => {
  setTimeout(() => resolve("done"), 100);
});

resolveOuter(inner);
```

The outer promise now follows `inner`.

The outer is not necessarily fulfilled immediately.

### 10.9 Thenable assimilation

If a resolution value behaves like a thenable:

```js
{
  then(resolve, reject) {}
}
```

the Promise machinery obtains and invokes its `then` behavior according to the resolution procedure.

### 10.10 Handler execution

For a fulfillment reaction:

```text
source fulfillment value
    ↓
handler(value)
    ↓
returned value OR throw
    ↓
settle downstream promise
```

For rejection:

```text
source rejection reason
    ↓
onRejected(reason)
    ↓
returned value OR throw
    ↓
settle downstream promise
```

### 10.11 Downstream promise

Every `.then()` creates another promise.

This makes promise chains graph-like:

```text
P0 → P1 → P2 → P3
```

Branches are possible:

```text
        → P1
P0
        → P2
        → P3
```

### 10.12 Promise sharing

Multiple consumers can attach to one promise:

```js
const shared = loadConfig();

shared.then(useByA);
shared.then(useByB);
shared.then(useByC);
```

The promise is not consumed by the first listener.

---

## 11. ECMAScript / Specification Semantics

### 11.1 Promise constructor

The Promise constructor:

1. creates a Promise object;
2. initializes its state;
3. creates resolving functions;
4. calls the executor;
5. converts executor throws into rejection.

The executor is synchronous.

### 11.2 Resolve functions

Calling the resolve function does not always immediately settle the promise.

For an ordinary value:

```text
resolve(value)
→ fulfill
```

For a promise/thenable:

```text
resolve(thenable)
→ adopt/follow
→ possibly remain pending
```

### 11.3 Reject function

Calling reject transitions an unresolved promise toward rejection.

Subsequent settlement attempts do not replace the outcome.

### 11.4 `then`

The specification's `PerformPromiseThen` machinery:

- creates the result capability;
- normalizes handlers;
- records reactions;
- schedules reactions when the source is already settled.

The current ECMAScript specification defines `Promise.prototype.then` in this model. citeturn842571search8turn842571search4

### 11.5 Promise Reaction Jobs

When a relevant promise reaction executes, the Promise Reaction Job:

- determines the handler;
- calls it if present;
- resolves the derived promise with the returned value;
- rejects the derived promise if the handler throws.

This is the bridge between promise settlement and asynchronous continuation. citeturn842571search4

### 11.6 Handler normalization

A missing fulfillment handler behaves like value propagation.

A missing rejection handler behaves like rejection propagation.

Conceptually:

```js
p.then()
```

does not consume the result.

### 11.7 Promise resolution procedure

Resolution recursively handles:

```text
ordinary values
thenables
promises
```

It must protect invariants such as:

```text
single settlement
self-resolution prevention
thenable first-call behavior
exception handling
```

### 11.8 Thenable assimilation

A foreign object with a callable `then` can participate in Promise resolution.

This supports interoperation across promise-like implementations.

### 11.9 `Promise.resolve`

`Promise.resolve(x)` normalizes its input into a promise.

If the value is already a suitable Promise from the same constructor, it can return that promise rather than unnecessarily creating another wrapper.

### 11.10 `Promise.reject`

`Promise.reject(reason)` creates a new rejected promise.

It does not synchronously throw `reason`.

### 11.11 Combinators

The combinators consume iterables and build aggregate promise behavior:

```text
all
allSettled
race
any
```

Each has distinct settlement rules.

### 11.12 `Promise.withResolvers`

`Promise.withResolvers()` creates a Promise together with externally accessible `resolve` and `reject` functions.

It is particularly useful when a promise must be settled from event-driven code outside the constructor executor. Current developer documentation lists it as a widely available modern Promise feature. citeturn842571search9

### 11.13 `finally`

`finally` schedules cleanup-style work while preserving the prior outcome when the cleanup completes normally.

If cleanup throws or rejects, the resulting promise rejects with that failure. citeturn842571search2

### 11.14 Subclassing

Promise methods participate in constructor/species behavior.

A `.then()` call on a Promise subclass can produce a derived promise of an appropriate constructor.

This connects Promise behavior with Chapter 21's constructor/species model.

---

## 12. Advanced Behavior

### 12.1 Promise resolution is recursive

Suppose:

```js
resolve(
  Promise.resolve(
    Promise.resolve(42)
  )
);
```

The outer promise conceptually follows through the nested resolution until the eventual outcome becomes an ordinary value.

### 12.2 Self-resolution

This is invalid:

```js
let resolve;

const p = new Promise(r => {
  resolve = r;
});

resolve(p);
```

A promise must not resolve to itself.

### 12.3 Thenable first-call semantics

Consider:

```js
const thenable = {
  then(resolve, reject) {
    resolve("A");
    resolve("B");
    reject(new Error("C"));
  }
};
```

Only the first effective resolution wins.

### 12.4 Thenable throws after resolution

```js
const thenable = {
  then(resolve) {
    resolve("A");
    throw new Error("B");
  }
};
```

The later throw cannot replace the earlier effective resolution.

### 12.5 Throw inside executor

```js
const p = new Promise(() => {
  throw new Error("boom");
});
```

The promise becomes rejected.

The exception does not escape synchronously as an ordinary uncaught throw from the constructor call.

### 12.6 Throw inside reaction

```js
Promise.resolve()
  .then(() => {
    throw new Error("boom");
  });
```

The resulting downstream promise becomes rejected.

### 12.7 Return undefined

```js
Promise.resolve(1)
  .then(() => {});
```

The downstream promise fulfills with `undefined`.

### 12.8 Return a thenable

```js
Promise.resolve()
  .then(() => ({
    then(resolve) {
      resolve("A");
    }
  }))
  .then(console.log);
```

The downstream chain receives `"A"` after thenable assimilation.

### 12.9 Multiple consumers

```js
const p = expensiveOperation();

p.then(a);
p.then(b);
p.then(c);
```

All consumers observe the same underlying promise outcome.

### 12.10 Branches do not coordinate automatically

```js
const p = Promise.resolve();

const a = p.then(stepA);
const b = p.then(stepB);
```

`a` and `b` are separate chains.

If `a` takes longer, `b` does not wait for `a`.

### 12.11 Chains are dependent

```js
const a = p.then(stepA);
const b = a.then(stepB);
```

`b` depends on `a`'s settlement.

### 12.12 `Promise.all` result ordering

Input:

```js
[
  slow,
  fast
]
```

may complete as:

```text
fast
slow
```

but:

```js
await Promise.all([slow, fast])
```

produces results in input order:

```text
[slowResult, fastResult]
```

The combinator separates completion order from result-position order.

### 12.13 `Promise.all` failure

If any input rejects:

```text
returned aggregate promise → rejected
```

Other operations are not automatically cancelled.

### 12.14 `Promise.allSettled`

Result shape is conceptually:

```js
[
  {
    status: "fulfilled",
    value: ...
  },
  {
    status: "rejected",
    reason: ...
  }
]
```

### 12.15 `Promise.any`

If:

```text
A rejects
B rejects
C fulfills
```

the aggregate fulfills with C.

If all reject:

```text
Promise.any(...)
→ AggregateError
```

### 12.16 `Promise.race`

The first settlement wins:

```text
fulfillment → aggregate fulfillment
rejection → aggregate rejection
```

A losing operation continues unless separately cancelled.

### 12.17 Empty combinators

Important edge cases:

```text
Promise.all([])        → fulfills with []
Promise.allSettled([]) → fulfills with []
Promise.any([])        → rejects with AggregateError
Promise.race([])       → remains pending
```

### 12.18 Non-promise inputs

```js
Promise.all([1, 2, 3]);
```

normalizes the values into completed inputs.

### 12.19 Thenable inputs

Combinators can receive thenables and apply promise-resolution behavior.

### 12.20 Constructor capture

Static combinators are constructor-sensitive.

Subclassing can therefore change the constructor used for derived promises.

### 12.21 Promise subclassing

Subclassing may be useful for specialized semantics, but it can complicate:

- constructor behavior;
- species;
- interoperability;
- combinators;
- ecosystem expectations.

Default to ordinary Promise unless there is a concrete design requirement.

### 12.22 `Promise.withResolvers`

Good use case:

```text
external callback/event
       ↓
resolve/reject
       ↓
promise
```

For example, an event-driven adapter can maintain a promise whose resolver is invoked later by an event callback.

### 12.23 `withResolvers` misuse

It can make promise ownership too manual:

```js
const { promise, resolve, reject } = Promise.withResolvers();
```

If the resolver escapes too broadly, the lifecycle becomes difficult to reason about.

### 12.24 Promise as memoization state

A promise can represent an in-flight shared operation:

```js
let current;

function load() {
  if (!current) {
    current = expensiveLoad();
  }

  return current;
}
```

But failure-reset policy must be designed:

```text
cache forever?
retry after rejection?
invalidate on timeout?
```

### 12.25 Promise as state machine

A promise can represent a one-time transition:

```text
pending → fulfilled/rejected
```

It is not an ideal replacement for reusable mutable state machines.

### 12.26 Promise as synchronization primitive

Promises can coordinate one-time events:

```js
const ready = initialize();
await ready;
```

But reusable synchronization such as locks, semaphores, and queues requires additional abstractions.

### 12.27 Promise cancellation limit

There is no universal:

```js
promise.cancel()
```

The operation must expose cancellation separately.

Common modern pattern:

```js
operation({ signal })
```

where the operation observes an `AbortSignal`.

---

## 13. Edge Cases

### 13.1 Resolve and reject both called

```js
new Promise((resolve, reject) => {
  resolve("A");
  reject(new Error("B"));
});
```

Result:

```text
fulfilled with A
```

### 13.2 Reject then resolve

```text
rejected with first effective outcome
```

### 13.3 Executor throws after resolve

The prior effective resolution remains in control.

### 13.4 Handler returns itself

A handler creating an indirect cycle can cause a promise to remain pending or trigger rejection depending on the exact cycle.

### 13.5 Direct self-return

```js
let p;

p = Promise.resolve().then(() => p);
```

This can create self-resolution problems.

### 13.6 `then` property getter throws

Thenable assimilation may trigger property-access exceptions.

### 13.7 `then` is non-callable

An object with:

```js
then: 123
```

is treated according to promise resolution rules rather than as a usable thenable.

### 13.8 Thenable calls multiple callbacks

First effective settlement wins.

### 13.9 `Promise.all` receives an invalid non-iterable

```js
Promise.all(123);
```

throws/rejects according to the API's invocation semantics rather than behaving like a normal iterable aggregation.

### 13.10 `Promise.any` all reject

The result rejects with `AggregateError` containing rejection reasons.

### 13.11 `Promise.race([])`

It remains pending indefinitely.

### 13.12 Pending promise retaining handlers

A pending promise can retain registered reactions for as long as it remains reachable.

### 13.13 Chaining on forever-pending promises

```js
const forever = new Promise(() => {});

forever.then(handler);
```

The handler remains retained because the source never settles.

### 13.14 Promise returned from `finally`

```js
p.finally(() => someAsyncCleanup());
```

The next promise waits for cleanup before reflecting the original result.

### 13.15 `finally` throws

The original result is replaced by rejection.

### 13.16 `finally` returns a rejected promise

Same principle:

```text
cleanup failure
→ resulting promise rejected
```

### 13.17 `Promise.all` with a long-lived pending promise

Repeatedly combining a forever-pending promise can accumulate reactions and memory.

### 13.18 Losing `race` operations

A losing operation may continue executing and retaining resources.

### 13.19 `Promise.any` losers

Rejected losers continue until they settle; their underlying operations are not cancelled.

### 13.20 Unhandled rejection timing

Reporting policies are host-specific.

Do not treat one runtime's unhandled-rejection timing as universal language behavior.

---

## 14. Common Misconceptions

### Misconception 1 — “Promise means asynchronous execution.”

A Promise represents eventual completion. The operation producing it may begin synchronously.

### Misconception 2 — “The Promise executor is asynchronous.”

Usually false.

The executor runs synchronously during Promise construction.

### Misconception 3 — “Resolved means fulfilled.”

Not always.

A promise can be resolved to another thenable while remaining pending.

### Misconception 4 — “A promise can settle twice.”

No.

A promise has one final settlement.

### Misconception 5 — “`.then()` modifies the promise result.”

It registers a reaction and returns a new promise.

### Misconception 6 — “The first `.then()` consumes the Promise.”

No.

Multiple consumers can observe the same Promise.

### Misconception 7 — “Returning a promise creates nested promises.”

Promise resolution adopts the returned promise/thenable outcome.

### Misconception 8 — “Promise rejection automatically cancels work.”

No.

Rejection describes outcome, not cancellation.

### Misconception 9 — “`Promise.race` cancels slower operations.”

No.

It only settles the returned promise from the first settlement.

### Misconception 10 — “`Promise.all` runs everything in parallel.”

It aggregates promises. Whether underlying operations run concurrently depends on the operations and host.

### Misconception 11 — “`Promise.all` stops the other operations after failure.”

No.

The aggregate promise rejects, but input operations can continue.

### Misconception 12 — “`Promise.any` returns the fastest operation.”

More precisely, it returns the first fulfilled input.

### Misconception 13 — “`finally` receives the value/error.”

It does not receive the original outcome as an argument.

### Misconception 14 — “A pending Promise is harmless.”

Pending promises can retain handlers, closures, and associated memory.

### Misconception 15 — “Promises are threads.”

No.

They are language-level completion abstractions.

---

## 15. Common Mistakes

### Mistake 1 — Creating Promise wrappers unnecessarily

```js
return new Promise(resolve => {
  resolve(existingPromise);
});
```

This may add needless complexity.

### Mistake 2 — Nested Promise construction

```js
return new Promise((resolve, reject) => {
  somePromise.then(resolve, reject);
});
```

Often unnecessary when an API already returns a promise.

### Mistake 3 — Missing returns in chains

```js
doA()
  .then(() => {
    doB();
  })
  .then(() => {
    // may run before doB completes
  });
```

Correct:

```js
doA()
  .then(() => {
    return doB();
  });
```

### Mistake 4 — Fire-and-forget without a policy

```js
doImportantWork();
```

### Mistake 5 — Using `Promise.all` for huge unbounded inputs

### Mistake 6 — Treating timeout as cancellation

### Mistake 7 — Swallowing rejection

```js
promise.catch(() => {});
```

### Mistake 8 — Logging and rethrowing at every layer

### Mistake 9 — Retaining forever-pending promises

### Mistake 10 — Resolving a Promise with a resource and losing ownership semantics

### Mistake 11 — Using one shared promise as a permanent cache without invalidation strategy

### Mistake 12 — Assuming `Promise.any` or `race` cleans up losers

### Mistake 13 — Creating a promise for ordinary synchronous branching

### Mistake 14 — Depending on host-specific unhandled rejection timing

---

## 16. Comparison With Related Concepts

| Concept | Main role | Key property |
|---|---|---|
| Promise | Future completion | One final settlement |
| Callback | Function invoked by an API | API-defined invocation |
| Result object | Explicit success/failure value | Synchronous/data-oriented |
| `async/await` | Syntax over promise-based async flow | Structured control flow |
| Generator | Suspended synchronous/iterative computation | Explicit `yield` |
| Observable | Multiple future values over time | Many emissions |
| EventEmitter | Repeated event notifications | Many events |
| Queue | Work ordering | Multiple tasks |
| AbortSignal | Cancellation notification | Separate from promise settlement |

### Promise vs callback

Callback:

```js
read(callback);
```

Promise:

```js
read().then(...);
```

Promises give:

- composition;
- chaining;
- standard propagation;
- combinators.

### Promise vs Observable

Promise:

```text
one eventual result
```

Observable:

```text
zero/one/many values over time
```

### Promise vs event emitter

Promise:

```text
one settlement
```

Event emitter:

```text
repeated notifications
```

### Promise vs result object

Promise is appropriate when completion is asynchronous.

A result object can be clearer when the outcome is an expected synchronous domain branch.

### Promise vs cancellation

Promise:

```text
what happened
```

Cancellation:

```text
stop trying to make it happen
```

These are complementary.

---

## 17. Performance Considerations

### 17.1 Promise allocation

Each Promise object has memory and bookkeeping costs.

Deep chains create:

```text
promise objects
reaction records
closures
jobs
```

### 17.2 High-frequency promise creation

In very hot synchronous loops, unnecessary promise creation can be expensive.

Do not promisify purely synchronous work without a reason.

### 17.3 Microtask volume

Large promise pipelines can generate many deferred reaction jobs.

### 17.4 Combinator fan-out

```js
Promise.all(hugeArray.map(operation))
```

can create large numbers of:

- promises;
- reactions;
- closures;
- retained input values.

### 17.5 Bounded concurrency

Instead of:

```js
await Promise.all(items.map(process));
```

consider a bounded scheduler when the input is large or dependencies are rate-limited.

### 17.6 Promise chains and latency

Each dependent stage can increase total wall-clock latency:

```text
A → B → C → D
```

independent stages can sometimes be reorganized:

```text
A ─┐
B ─┼→ aggregate
C ─┘
```

### 17.7 `Promise.all` is not a worker scheduler

Its performance depends on the underlying operations.

### 17.8 Memory retained by pending chains

A forever-pending promise with thousands of handlers can consume significant memory.

### 17.9 Repeated `Promise.race`

Repeatedly racing a short timeout against a long-lived promise can accumulate handlers on the long-lived promise.

### 17.10 Error overhead

Creating and retaining deep error chains can add cost.

### 17.11 Measurement

Measure:

```text
promise allocation
GC
reaction count
operation latency
queue delay
concurrency
downstream saturation
```

rather than optimizing promise syntax by intuition.

---

## 18. Memory Considerations

### 18.1 Pending promises retain handlers

```js
const pending = new Promise(() => {});

pending.then(() => use(largeObject));
```

If `pending` remains reachable forever, the reaction and its closure can remain retained.

### 18.2 Shared promises retain consumers

A long-lived shared promise can retain all attached handlers until settlement.

### 18.3 Promise chains retain intermediate state

Long chains may keep closures and intermediate references alive.

### 18.4 Large combinators

`Promise.all()` often retains:

- input values;
- result array;
- reaction state;

until all required outcomes are resolved or the aggregate rejects.

### 18.5 Losing operations

With `race`:

```text
winner settles
losers continue
```

Those losing operations can keep memory/resources alive.

### 18.6 Cancellation reduces retention

Actual cancellation can stop work that would otherwise retain resources.

### 18.7 In-flight memoization

A shared in-flight promise is useful, but stale or permanently pending operations require expiration/invalidation.

---

## 19. Security Considerations

### 19.1 Promise rejection can carry secrets

Never automatically log arbitrary rejection reasons.

### 19.2 Thenable execution

Thenable assimilation can execute attacker-controlled code.

### 19.3 Unbounded concurrency

Attackers can trigger high fan-out Promise creation and overload downstream systems.

### 19.4 Promise races

Security-sensitive code can have stale-result races.

Example:

```text
authorization check
↓
await
↓
state changed
↓
action continues
```

### 19.5 Timeout is not cancellation

A request can continue after the caller has stopped waiting.

That can cause:

- duplicated writes;
- resource exhaustion;
- inconsistent audit trails.

### 19.6 Rejection disclosure

Returning raw errors from rejected promises to clients can leak:

- stack traces;
- infrastructure names;
- database details.

### 19.7 Untrusted thenables

Do not assume a `.then()` property is inert.

### 19.8 Promise-based resource leakage

A rejected or abandoned promise can leave resources alive if the underlying operation has no cancellation/cleanup path.

---

## 20. Production Usage

### 20.1 API design

A production async function should make clear:

```text
what the promise represents
success shape
failure shape
cancellation
timeout
resource ownership
side effects
```

### 20.2 Shared request deduplication

Example:

```js
const inFlight = new Map();

function loadUser(id) {
  if (!inFlight.has(id)) {
    const promise = fetchUser(id)
      .finally(() => {
        inFlight.delete(id);
      });

    inFlight.set(id, promise);
  }

  return inFlight.get(id);
}
```

This allows concurrent callers to share one in-flight operation.

The invalidation policy is essential.

### 20.3 Dependency fan-out

```js
const [profile, permissions, settings] =
  await Promise.all([
    getProfile(id),
    getPermissions(id),
    getSettings(id)
  ]);
```

Use only when the operations are independent and downstream capacity allows it.

### 20.4 Graceful failure

When one dependency fails, decide whether:

```text
fail whole request
degrade partially
use fallback
retry
serve stale cache
```

Do not let Promise combinators make business decisions accidentally.

### 20.5 `Promise.allSettled` for partial results

Use when every operation's outcome matters.

Examples:

- batch notifications;
- multi-destination telemetry;
- best-effort cleanup;
- diagnostics.

### 20.6 `Promise.any` for redundant providers

Useful where:

```text
any successful replica/provider is sufficient
```

Combine with cancellation if losing requests should stop.

### 20.7 `Promise.race` for time-bound policy

Use when a first outcome is needed, but pair it with actual cancellation where possible.

### 20.8 `withResolvers` for event adapters

Useful for:

```text
callback/event API
→ one Promise settlement
```

Be careful with resolver ownership and lifetime.

### 20.9 Cleanup

```js
await operation()
  .finally(cleanup);
```

For richer ownership semantics, integrate with resource-management mechanisms from Chapter 30.

### 20.10 Process boundaries

At HTTP/worker/CLI boundaries:

```text
Promise rejection
→ explicit boundary
→ classify
→ log/trace
→ recover/translate/terminate
```

### 20.11 Observability

Record:

- operation name;
- duration;
- success/failure;
- error class/code;
- retry count;
- cancellation;
- dependency;
- request/trace context.

Do not make logging the only failure-handling mechanism.

---

## 21. Implementation From Scratch

### Stage 1 — Guided

Implement a minimal asynchronous result abstraction:

```js
class MiniPromise {
  constructor(executor) {
    // state
    // reactions
    // resolve
    // reject
  }
}
```

Support:

```text
pending
fulfilled
rejected
```

### Stage 2 — Partially Guided

Add:

```js
then(onFulfilled, onRejected)
```

and deferred execution.

Requirements:

- return a new MiniPromise;
- propagate values;
- propagate errors;
- convert handler throws to rejection.

### Stage 3 — No Reference

Implement:

```js
MiniPromise.resolve(value)
MiniPromise.reject(reason)
```

and chaining:

```js
MiniPromise.resolve(1)
  .then(x => x + 1)
  .then(console.log);
```

### Stage 4 — Edge-Case Hardened

Add:

- thenable assimilation;
- self-resolution protection;
- multiple resolve/reject calls;
- executor throws;
- returned promises;
- error propagation;
- empty combinators;
- multiple consumers.

### Stage 5 — Production-Oriented Learning Engine

Do not use this as a replacement for native Promise.

Build it to expose instrumentation:

```text
promise ID
parent ID
state
created at
settled at
reaction count
handler duration
cause/error
```

Then study:

```text
allocation
scheduling
retention
chain depth
latency
```

---

## 22. Debugging Exercises

### Exercise 1 — Missing return

```js
doA()
  .then(() => {
    doB();
  })
  .then(() => {
    console.log("done");
  });
```

Determine why `"done"` may occur before `doB()` completes.

### Exercise 2 — Detached rejection

```js
function run() {
  doWork().then(saveResult);
}

run();
```

Who owns failures from `saveResult`?

### Exercise 3 — Race timeout

```js
await Promise.race([
  doWork(),
  timeout(1000)
]);
```

Determine what still runs when timeout wins.

### Exercise 4 — `all` cancellation misconception

```js
await Promise.all([
  taskA(),
  taskB(),
  taskC()
]);
```

Assume B rejects immediately.

What happens to A and C?

### Exercise 5 — Forever pending

```js
const pending = new Promise(() => {});

for (let i = 0; i < 100000; i++) {
  pending.then(() => {});
}
```

Identify memory consequences.

### Exercise 6 — Thenable trap

```js
const value = {
  then(resolve) {
    console.log("then called");
    resolve(42);
  }
};

Promise.resolve(value).then(console.log);
```

Trace the assimilation sequence.

### Exercise 7 — Shared in-flight promise

Implement:

```js
getUser(id)
```

so simultaneous requests share one network operation.

Then answer:

```text
What happens on success?
What happens on failure?
What happens on timeout?
What happens if the promise never settles?
```

---

## 23. Code Review Exercise

Review:

```js
async function loadDashboard() {
  const user = await getUser();
  const stats = await getStats();
  const notifications = await getNotifications();

  return {
    user,
    stats,
    notifications
  };
}
```

Determine:

- which dependencies are real;
- which operations could overlap;
- what should happen if notifications fail;
- whether partial data is acceptable;
- whether all work should be cancellable;
- whether a shared in-flight cache is appropriate.

Then review this variation:

```js
async function loadDashboard() {
  return Promise.all([
    getUser(),
    getStats(),
    getNotifications()
  ]);
}
```

Determine whether it is semantically equivalent.

---

## 24. Interview Questions

### Foundational

1. What is a Promise?
2. What states can it have?
3. What does “settled” mean?
4. What is the difference between resolved and fulfilled?
5. Does the Promise executor run synchronously?
6. Why does `.then()` return another Promise?
7. What happens when a `.then()` handler returns a value?
8. What happens when it throws?
9. What happens when it returns another Promise?
10. What does `.catch()` do?

### Intermediate

11. What is thenable assimilation?
12. Why can a Promise be resolved but pending?
13. Why can multiple consumers attach to the same Promise?
14. What does `Promise.all()` do on rejection?
15. Does `Promise.all()` cancel remaining work?
16. What is the difference between `all` and `allSettled`?
17. What is the difference between `race` and `any`?
18. When does `Promise.any()` reject?
19. What is `AggregateError`?
20. What is `Promise.withResolvers()`?

### Advanced

21. Explain Promise resolution precisely.
22. Explain self-resolution.
23. Explain first-effective-settlement behavior.
24. Explain the relationship between Promise reactions and Jobs.
25. Explain missing handlers and propagation.
26. Explain `finally()` semantics.
27. Explain Promise memory retention.
28. Explain why `race()` is not cancellation.
29. Explain Promise subclass/species behavior.
30. Explain how Promise combinators consume iterables.

### Principal-Level

31. Design a production Promise-based API contract.
32. Design in-flight request deduplication.
33. Design bounded concurrent Promise processing.
34. Design cancellation around Promise-based operations.
35. Design partial-failure handling with `allSettled`.
36. Design redundant-provider selection with `any`.
37. Design timeout + cancellation correctly.
38. Diagnose a promise-heavy memory leak.
39. Diagnose a detached rejection in production.
40. Defend when to use Promise combinators versus an explicit scheduler.

---

## 25. Predict-the-Output Exercises

### Exercise A

```js
console.log("A");

new Promise(resolve => {
  console.log("B");
  resolve();
}).then(() => {
  console.log("C");
});

console.log("D");
```

Expected:

```text
A
B
D
C
```

### Exercise B

```js
const p = Promise.resolve("A");

p.then(value => {
  console.log(value);
  return "B";
}).then(value => {
  console.log(value);
});

console.log("C");
```

Predict:

```text
C
A
B
```

### Exercise C

```js
Promise.resolve()
  .then(() => {
    console.log("A");
    throw new Error("B");
  })
  .catch(error => {
    console.log(error.message);
    return "C";
  })
  .then(console.log);
```

Predict:

```text
A
B
C
```

### Exercise D

```js
const p = Promise.resolve();

p.then(() => console.log("A"));
p.then(() => console.log("B"));

p.then(() => {
  console.log("C");
  return Promise.resolve();
}).then(() => {
  console.log("D");
});
```

Explain why the first-level reactions and chained reaction do not all execute as one callback.

### Exercise E

```js
Promise.all([
  Promise.resolve("A"),
  Promise.reject(new Error("B")),
  Promise.resolve("C")
])
  .then(() => console.log("success"))
  .catch(error => console.log(error.message));
```

Predict:

```text
B
```

Then explain what happened to A and C.

### Exercise F

```js
Promise.any([
  Promise.reject("A"),
  Promise.resolve("B"),
  Promise.resolve("C")
]).then(console.log);
```

Predict:

```text
B
```

### Exercise G

```js
Promise.race([
  Promise.resolve("A"),
  Promise.resolve("B")
]).then(console.log);
```

Determine the result based on iterable order and Promise reaction scheduling rather than saying “the first line is faster.”

---

## 26. Mastery Exercises

### Exercise 1 — Promise state machine

Draw:

```text
pending
  ↓
resolved-to-value
  ↓
fulfilled
```

and:

```text
pending
  ↓
resolved-to-thenable
  ↓
follow thenable
  ↓
fulfilled/rejected
```

Explain why “resolved” and “settled” differ.

### Exercise 2 — Promise implementation

Implement a MiniPromise with:

```text
constructor
resolve
reject
then
catch
finally
```

### Exercise 3 — Thenable compatibility

Test your implementation against:

- native Promise;
- custom thenable;
- throwing thenable;
- double-settling thenable;
- self-resolution.

### Exercise 4 — Combinators

Implement:

```js
miniAll
miniAllSettled
miniRace
miniAny
```

using your own Promise-like abstraction.

### Exercise 5 — Bounded concurrency

Implement:

```js
mapConcurrent(items, limit, worker)
```

Requirements:

- preserve result order;
- bound active workers;
- reject/settle according to policy;
- support cancellation;
- avoid unbounded promise creation.

### Exercise 6 — Timeout + cancellation

Implement:

```js
withTimeout(operation, ms, { signal })
```

where the underlying operation can actually be cancelled.

### Exercise 7 — In-flight deduplication

Implement:

```js
getResource(key)
```

where concurrent requests for the same key share one operation.

Add:

- expiration;
- failure reset;
- cancellation policy;
- metrics.

### Exercise 8 — Promise leak investigation

Construct a benchmark with a forever-pending Promise and many `.then()` registrations.

Measure memory growth.

---

## 27. Key Takeaways

1. A Promise represents eventual completion.
2. Promise state is `pending`, `fulfilled`, or `rejected`.
3. Fulfillment/rejection means settlement.
4. Resolution is broader than fulfillment: a Promise can resolve to a thenable and remain pending while it follows that thenable.
5. The Promise executor runs synchronously during construction.
6. Promise reactions execute asynchronously through the language/runtime job mechanism.
7. `.then()` creates a new downstream Promise.
8. Returning a value fulfills the downstream Promise.
9. Throwing in a reaction rejects the downstream Promise.
10. Returning a Promise/thenable causes adoption of its eventual outcome.
11. Promise resolution protects the one-settlement invariant.
12. Thenable assimilation provides interoperability but can execute arbitrary code.
13. `catch()` is rejection handling within the Promise chain.
14. `finally()` is cleanup-oriented and normally preserves the prior outcome.
15. `Promise.all()` requires every input to fulfill and rejects on the first rejection.
16. `Promise.allSettled()` waits for every input.
17. `Promise.race()` settles from the first input settlement.
18. `Promise.any()` fulfills from the first successful input and rejects with `AggregateError` if all inputs reject.
19. None of the standard combinators automatically cancels losing/remaining underlying operations.
20. `Promise.withResolvers()` is useful for externally controlled settlement.
21. Promise chains are dependency graphs, not just lists of callbacks.
22. Pending Promises can retain closures and reactions.
23. Promise-heavy systems need explicit concurrency and lifecycle policies.
24. Promises represent completion; cancellation must be designed separately.
25. The central principle is:

> A Promise is a one-settlement, composable representation of eventual completion—not a thread, scheduler, or cancellation mechanism.

---

## 28. Concept Connections

### Depends On

- Chapter 09 — Functions / First-Class Behavior
- Chapter 12 — Execution Contexts / Execution Model
- Chapter 18 — Classes / OOP
- Chapter 20 — Symbols / Well-Known Symbols
- Chapter 29 — Errors / Error Handling
- Chapter 30 — Resource Management / Cleanup
- Chapter 31 — Asynchronous JavaScript Fundamentals
- Chapter 32 — ECMAScript Jobs / Promise Reactions
- Chapter 33 — Browser Event Loop
- Chapter 34 — Node Event Loop / libuv

### Builds Toward

- Chapter 36 — Async/Await
- Chapter 37 — Cancellation / Abort
- Chapter 38 — Async Iteration / Streaming
- Chapter 39 — Concurrency / Parallelism
- Chapter 40 — Observables / Reactive
- Chapter 45 — Memory / GC
- Chapter 46 — Weak Refs / Finalization
- Chapter 51 — Browser APIs
- Chapter 53 — Web Streams / Data Flow
- Chapter 55 — Fetch / HTTP Networking
- Chapter 58 — Node Architecture
- Chapter 59 — Node Core APIs
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

- Promise state
- Promise resolution
- Thenables
- Promise reactions
- Jobs
- Async/await
- Cancellation
- `AbortSignal`
- Timeout
- Concurrency
- `Promise.all`
- `Promise.allSettled`
- `Promise.race`
- `Promise.any`
- `AggregateError`
- `Promise.withResolvers`
- Resource lifetime
- Error propagation
- Backpressure
- Observability

### Concepts Revisited

This chapter revisits:

- abrupt completion;
- objects;
- classes;
- species/subclassing;
- errors;
- Jobs;
- microtasks;
- asynchronous execution;
- resource ownership.

### Why This Chapter Matters Later

Promises are the central asynchronous abstraction for modern JavaScript.

Async/await, Fetch, many Node APIs, browser APIs, concurrency utilities, and production service code all build on Promise semantics.

A shallow Promise model causes recurring mistakes:

```text
executor is async
resolved = fulfilled
race = cancellation
all = parallelism
await = blocking
rejection = cancellation
promise = thread
```

A precise Promise model eliminates these errors before they spread into larger architecture.

The central principle is:

> Promise semantics are the foundation on which modern JavaScript asynchronous control flow is built.

---

## 29. Completion Criteria

Mark Chapter 35 `[+] Completed` only after the learner can demonstrate all of the following.

### Conceptual Understanding

- [ ] Define a Promise.
- [ ] Explain pending, fulfilled, rejected.
- [ ] Explain settlement.
- [ ] Explain resolution versus settlement.
- [ ] Explain the executor.
- [ ] Explain `resolve` and `reject`.
- [ ] Explain thenable assimilation.
- [ ] Explain `.then()`, `.catch()`, `.finally()`.
- [ ] Explain downstream promises.
- [ ] Explain promise combinators.
- [ ] Explain `Promise.withResolvers`.

### Predictive Mastery

- [ ] Predict executor timing.
- [ ] Predict reaction timing.
- [ ] Predict chain ordering.
- [ ] Predict error propagation.
- [ ] Predict returned-promise adoption.
- [ ] Predict `all`.
- [ ] Predict `allSettled`.
- [ ] Predict `race`.
- [ ] Predict `any`.
- [ ] Predict `finally`.
- [ ] Predict thenable edge cases.
- [ ] Predict pending-promise memory behavior.

### Implementation

- [ ] Implement a MiniPromise.
- [ ] Implement chaining.
- [ ] Implement resolution/adoption.
- [ ] Implement combinators.
- [ ] Implement bounded concurrency.
- [ ] Implement timeout + cancellation.
- [ ] Implement in-flight deduplication.

### Debugging

- [ ] Diagnose missing Promise returns.
- [ ] Diagnose detached rejections.
- [ ] Diagnose timeout/race misconceptions.
- [ ] Diagnose unbounded concurrency.
- [ ] Diagnose pending-promise memory retention.
- [ ] Diagnose thenable behavior.
- [ ] Diagnose `finally` masking.
- [ ] Diagnose combinator failure policies.

### Production Engineering

- [ ] Design Promise-based API contracts.
- [ ] Choose appropriate combinators.
- [ ] Define cancellation separately.
- [ ] Design bounded concurrency.
- [ ] Design shared in-flight operations.
- [ ] Define failure policy.
- [ ] Define resource ownership.
- [ ] Instrument async operations.
- [ ] Prevent unhandled/detached failures.

### Interview Readiness

- [ ] Explain resolved vs fulfilled.
- [ ] Explain executor timing.
- [ ] Explain thenable adoption.
- [ ] Explain `.then()` chaining.
- [ ] Explain all/allSettled/race/any.
- [ ] Explain why race is not cancellation.
- [ ] Explain Promise memory behavior.
- [ ] Design production Promise APIs.
- [ ] Defend Promise architecture decisions.

### Track A — Core Theory

- [ ] Understand Promise state.
- [ ] Understand Promise resolution.
- [ ] Understand reactions.
- [ ] Understand combinators.
- [ ] Understand specification semantics.

### Track B — Implementation

- [ ] Guided implementation completed.
- [ ] Partially guided implementation completed.
- [ ] No-reference implementation completed.
- [ ] Edge-case hardened implementation completed.
- [ ] Production-oriented learning implementation reviewed.

### Track C — Interview / Reasoning

- [ ] Completed output prediction.
- [ ] Completed chain-debugging exercises.
- [ ] Completed code review exercise.
- [ ] Completed bounded concurrency design.
- [ ] Completed cancellation design.
- [ ] Defended Promise architecture decisions.

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

# Chapter 35 — Revision / Retrieval Record

### Retrieval Prompts

1. What is the difference between resolved and fulfilled?
2. Does the Promise executor run synchronously?
3. Why are `.then()` handlers asynchronous?
4. Why does `.then()` return a new Promise?
5. What happens when a handler returns a normal value?
6. What happens when it throws?
7. What happens when it returns a Promise?
8. What is thenable assimilation?
9. What is self-resolution?
10. What does `Promise.all()` do when one input rejects?
11. Does `Promise.all()` cancel remaining operations?
12. What does `Promise.allSettled()` guarantee?
13. What does `Promise.race()` guarantee?
14. What does `Promise.any()` guarantee?
15. When does `Promise.any()` produce `AggregateError`?
16. What does `Promise.withResolvers()` solve?
17. Why is timeout not cancellation?
18. How can a pending Promise retain memory?
19. How would you implement bounded concurrent Promise processing?
20. How would you debug a detached Promise rejection?
21. How would you design in-flight request deduplication?
22. When should you avoid creating a new Promise wrapper?

### Weak Areas

```text
-
-
-
```

### Revision Queue

```text
- [ ] Revisit resolved vs fulfilled
- [ ] Revisit thenable assimilation
- [ ] Revisit reaction/chaining semantics
- [ ] Revisit combinators
- [ ] Revisit cancellation limitations
- [ ] Revisit Promise memory retention
- [ ] Revisit bounded concurrency
- [ ] Revisit withResolvers
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

# Chapter 35 — Canonical References and Source Discipline

Use this source hierarchy:

1. ECMAScript specification — Promise objects, constructor semantics, resolution, reactions, Jobs, `then`, `catch`, `finally`, and combinators.
2. TC39 proposal/history material — only where historical evolution or feature introduction is relevant, such as `Promise.withResolvers`.
3. MDN / browser documentation — practical developer behavior and compatibility information.
4. Node.js documentation — host integration, diagnostics, unhandled rejection behavior, and runtime-specific APIs.
5. JavaScript engine/runtime documentation — implementation and performance details.
6. Application architecture documentation — cancellation, concurrency, retries, ownership, observability, and operational policy.

The current ECMAScript specification describes `Promise.prototype.then`, `PerformPromiseThen`, and `NewPromiseReactionJob` in the Promise model. citeturn842571search8turn842571search4

Current developer documentation also describes the modern Promise combinators and `Promise.withResolvers()` behavior. citeturn842571search3turn842571search5turn842571search9

Always distinguish:

```text
ECMAScript semantics
vs
engine implementation
vs
browser behavior
vs
Node.js behavior
vs
application policy
```

Do not claim that Promise settlement cancels underlying host work.

Do not treat `Promise.race()` as a cancellation API.

Do not treat `Promise.all()` as a general-purpose concurrency scheduler.

---

# Chapter 35 — Completion Snapshot

```text
Chapter: 35
Title: Promises
Part: VI — Async
Status: [+] Expanded
Track A: [ ] Core Theory
Track B: [ ] Implementation
Track C: [ ] Interview / Reasoning
Mastery: [ ] Not Mastered
```
