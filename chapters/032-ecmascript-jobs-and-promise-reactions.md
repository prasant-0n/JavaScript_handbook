
# Chapter 32 — ECMAScript Jobs and Promise Reactions

## 1. Learning Objectives

By the end of this chapter, the learner must be able to:

- Explain what an ECMAScript Job is at a conceptual and specification level.
- Explain why JavaScript needs deferred execution mechanisms even before considering browser or Node event loops.
- Distinguish synchronous evaluation from job-based continuation.
- Explain promise reactions and how they are scheduled.
- Distinguish promise state from execution of its reactions.
- Explain why `.then()`, `.catch()`, and `.finally()` callbacks do not normally run synchronously.
- Explain how a promise settlement causes relevant reactions to become scheduled.
- Understand the relationship among promises, reactions, jobs, and host scheduling.
- Explain thenable assimilation and why promise resolution is not identical to simply assigning a value.
- Explain how chained promises create additional reaction jobs.
- Predict ordering in nested promise/reaction examples.
- Explain why callback registration order can affect reaction ordering.
- Distinguish job creation, queueing, and execution.
- Explain the role of the host in determining when the execution of jobs is allowed to proceed.
- Understand why “microtask” is useful host/runtime terminology but is not a complete substitute for the ECMAScript Job model.
- Explain how recursive promise scheduling can create starvation-like behavior.
- Explain why a fulfilled promise does not imply that all registered reactions have already executed.
- Distinguish promise propagation from ordinary synchronous exception propagation.
- Explain how errors thrown inside promise reactions become downstream rejections.
- Reason about promise chain flattening and returned thenables.
- Debug ordering and rejection behavior using precise timelines.
- Implement a minimal promise-reaction scheduler as a learning exercise.
- Design production async flows with correct error, ordering, cancellation, and resource-lifetime boundaries.
- Defend job scheduling behavior at specification, runtime, and application architecture levels.

### Mastery Gate

The chapter is mastered only when the learner can:

> Understand → Explain → Predict → Implement → Debug → Apply → Compare → Defend

Reading the chapter is not sufficient.

---

## 2. Prerequisites

The learner should understand:

- JavaScript values and types.
- Functions and lexical scope.
- Execution contexts.
- Control flow and abrupt completion.
- Objects and prototypes.
- Iterables and iterators.
- Promises at a foundational level.
- Async/await at a conceptual level.
- Basic host-runtime scheduling.
- Error propagation.

Primary dependencies:

- Chapter 09 — Functions / First-Class Behavior
- Chapter 12 — Execution Contexts / Execution Model
- Chapter 25 — Iterables / Iterators
- Chapter 29 — Errors / Error Handling
- Chapter 31 — Asynchronous JavaScript Fundamentals

The next chapters build outward from this specification-level foundation:

- Chapter 33 — Browser Event Loop
- Chapter 34 — Node Event Loop / libuv
- Chapter 35 — Promises
- Chapter 36 — Async/Await
- Chapter 37 — Cancellation / Abort

---

## 3. What Is It?

An **ECMAScript Job** is a specification-level unit of work that is scheduled to execute later rather than as part of the current synchronous evaluation.

Jobs are especially important for promise behavior.

When you write:

```js
Promise.resolve("value").then(value => {
  console.log(value);
});
```

the reaction callback is not simply called as an immediate consequence of `.then()`.

Instead, conceptually:

```text
promise reaction registered
        ↓
promise settles / is already settled
        ↓
reaction becomes eligible
        ↓
a job is queued
        ↓
job executes later
        ↓
callback runs
```

This mechanism lets the ECMAScript language define consistent asynchronous behavior without making the language itself depend on one specific browser or server event loop.

The key distinction is:

> A Job is a specification-level scheduling abstraction; a browser or Node.js event loop is a host-level scheduling system with additional responsibilities.

Promise reactions are one of the most important users of the Job model.

---

## 4. Why Does It Exist?

Consider:

```js
const p = Promise.resolve(42);

p.then(value => {
  console.log(value);
});

console.log("after");
```

If the reaction executed synchronously, behavior would be:

```text
42
after
```

Instead, promise reactions are deferred:

```text
after
42
```

This gives promise-based APIs a predictable asynchronous contract.

It also prevents code from behaving differently merely because a promise happened to be already fulfilled versus fulfilling later.

For example:

```js
function getValue() {
  return Promise.resolve(42);
}
```

A caller can consistently attach:

```js
getValue().then(handle);
```

without needing one code path for:

```text
already available
```

and another for:

```text
available later
```

The Job model therefore helps create a uniform abstraction:

```text
future completion
→ scheduled reaction
```

rather than:

```text
sometimes synchronous
sometimes asynchronous
```

That uniformity is critical for composability.

---

## 5. Mental Model

Use this model:

```text
Synchronous JavaScript
        │
        ▼
Promise operation
        │
        ├── pending
        │
        └── settled
               │
               ▼
        Promise reactions
               │
               ▼
            Jobs
               │
               ▼
      host-permitted execution
               │
               ▼
        callback execution
               │
               ▼
      next promise settlement
               │
               ▼
        more reaction jobs
```

A second mental model is:

```text
Promise = state + reactions + eventual result propagation

Job = a scheduled unit that performs one piece of deferred work
```

Do not collapse these into one object.

A promise may be fulfilled now.

A reaction may still be waiting to execute.

A new promise produced by the reaction may still be pending.

A useful timeline:

```text
T0  current code runs
T1  `.then()` registers reaction
T2  promise is settled
T3  reaction job is enqueued
T4  current synchronous execution finishes
T5  job executes
T6  callback runs
T7  returned value settles next promise
T8  next reaction job is enqueued
```

The exact observable behavior depends on the specific program and host scheduling boundary, but this model is extremely useful.

---

## 6. Core Rules

### Rule 1 — Promise reactions are not ordinary immediate calls

```js
Promise.resolve().then(fn);
```

does not execute `fn` as part of the current synchronous statement sequence.

### Rule 2 — A settled promise can still have pending reactions

Settlement and reaction execution are distinct events.

### Rule 3 — `.then()` creates promise-reaction state

Calling:

```js
p.then(onFulfilled, onRejected);
```

does more than “register a callback.”

It creates a reaction associated with a resulting promise.

### Rule 4 — Reaction callbacks execute through scheduled work

They are not simply called inline by the `.then()` operation.

### Rule 5 — Reaction return values settle downstream promises

If:

```js
const q = p.then(() => 42);
```

then `q` fulfills with `42` after the reaction executes.

### Rule 6 — Throwing inside a reaction rejects the downstream promise

```js
const q = p.then(() => {
  throw new Error("boom");
});
```

Now `q` becomes rejected.

### Rule 7 — Returning a promise or thenable causes adoption

```js
const q = p.then(() => anotherPromise);
```

The resulting promise does not simply fulfill with the promise object. It follows the resolution/adoption semantics.

### Rule 8 — Multiple reactions preserve registration ordering for the same promise

```js
p.then(() => console.log("A"));
p.then(() => console.log("B"));
```

For the same settlement, their reactions are processed according to the language's ordering rules.

### Rule 9 — Chained reactions can create additional jobs

```js
p.then(a).then(b);
```

The execution of `b` depends on the outcome of `a`, so its reaction cannot simply run at the same time as `a`.

### Rule 10 — A job queue is not the same thing as the entire event loop

The host may have other scheduling queues and responsibilities.

### Rule 11 — Job execution is not parallel execution

Jobs execute according to the runtime's execution model; scheduling work for later does not itself create a second JavaScript execution thread.

### Rule 12 — Infinite or recursive job creation can prevent other work from progressing

For example:

```js
function loop() {
  queueMicrotask(loop);
}

loop();
```

can continuously schedule more work.

The exact starvation consequences are host-dependent, but the general danger is real.

---

## 7. Syntax

The primary syntax relevant to this chapter includes promise reaction APIs:

### `then`

```js
promise.then(onFulfilled, onRejected);
```

### `catch`

```js
promise.catch(onRejected);
```

Conceptually equivalent to:

```js
promise.then(undefined, onRejected);
```

### `finally`

```js
promise.finally(onFinally);
```

It establishes cleanup-like behavior that preserves the original fulfillment/rejection outcome unless the `finally` callback itself changes the completion.

### Explicit job-like host API

Many environments provide:

```js
queueMicrotask(callback);
```

This is useful for experimenting with deferred execution, but it should not be treated as a complete synonym for every specification-level Job concept.

### Promise construction

```js
new Promise((resolve, reject) => {
  // executor
});
```

A critical distinction:

```text
executor function → called synchronously during construction
reaction callback → scheduled for later execution
```

---

## 8. Basic Examples

### Example 1 — Basic reaction ordering

```js
console.log("A");

Promise.resolve().then(() => {
  console.log("B");
});

console.log("C");
```

Output:

```text
A
C
B
```

### Example 2 — Multiple reactions

```js
const p = Promise.resolve();

p.then(() => console.log("A"));
p.then(() => console.log("B"));

console.log("C");
```

Output:

```text
C
A
B
```

### Example 3 — Chaining

```js
Promise.resolve()
  .then(() => {
    console.log("A");
    return "B";
  })
  .then(value => {
    console.log(value);
  });
```

Output:

```text
A
B
```

The second callback depends on the first reaction's completion.

### Example 4 — Reaction throw

```js
Promise.resolve()
  .then(() => {
    throw new Error("failed");
  })
  .catch(error => {
    console.log(error.message);
  });
```

Output:

```text
failed
```

### Example 5 — Return a promise

```js
Promise.resolve()
  .then(() => {
    return Promise.resolve("done");
  })
  .then(value => {
    console.log(value);
  });
```

Output:

```text
done
```

The downstream promise adopts the returned promise's eventual state.

---

## 9. Execution Walkthrough

Consider:

```js
console.log("1");

const p = Promise.resolve("A");

p.then(value => {
  console.log(value);
  return "B";
}).then(value => {
  console.log(value);
});

console.log("2");
```

### Step 1

`"1"` prints.

### Step 2

`Promise.resolve("A")` produces an already-fulfilled promise.

### Step 3

The first `.then()` registers a fulfillment reaction.

Because the promise is already fulfilled, the reaction becomes ready for scheduled execution.

### Step 4

The second `.then()` is attached to the promise returned by the first `.then()`.

At this point, the first reaction has not run, so the second promise has not yet acquired its eventual fulfillment from the callback.

### Step 5

The synchronous code continues:

```text
2
```

### Step 6

The first reaction job executes.

It logs:

```text
A
```

and returns:

```text
B
```

### Step 7

The first downstream promise fulfills with `"B"`.

### Step 8

The second reaction becomes schedulable.

### Step 9

The second reaction job executes and prints:

```text
B
```

Final output:

```text
1
2
A
B
```

The crucial rule:

> Each stage of a promise chain depends on the settlement created by the previous stage.

---

## 10. Internal Mechanics

### 10.1 Promise reaction records

Conceptually, a `.then()` call creates reaction information containing things such as:

```text
handler
capability/result promise
fulfillment vs rejection behavior
```

You can imagine:

```text
Promise P
  ├── Reaction R1 → downstream Promise Q1
  └── Reaction R2 → downstream Promise Q2
```

When `P` settles, the relevant reactions become schedulable.

### 10.2 Promise capabilities

A promise reaction needs a resulting promise.

Therefore:

```js
const q = p.then(handler);
```

does not return `p`.

It creates a new promise `q`.

### 10.3 Reaction job

A reaction job performs approximately:

```text
retrieve settled promise outcome
      ↓
select fulfillment/rejection handler
      ↓
call handler if appropriate
      ↓
capture returned value or thrown error
      ↓
resolve/reject downstream promise
```

This is a central bridge between promise state and actual callback execution.

### 10.4 Handler selection

For:

```js
p.then(onFulfilled, onRejected);
```

- fulfilled source → fulfillment handler;
- rejected source → rejection handler.

If the relevant handler is missing, the result is propagated to the downstream promise according to promise resolution semantics.

### 10.5 Returned values

If:

```js
p.then(() => 42);
```

the downstream promise fulfills with `42`.

If:

```js
p.then(() => {
  throw error;
});
```

the downstream promise rejects with `error`.

### 10.6 Returned thenables

Suppose:

```js
p.then(() => ({
  then(resolve) {
    resolve("value");
  }
}));
```

The result is not simply the object itself.

The promise resolution process assimilates thenable behavior.

This is why promise resolution is a protocol rather than a simple assignment.

### 10.7 Thenable hazards

A foreign object can define:

```js
then(resolve, reject) {
  // arbitrary behavior
}
```

Therefore assimilation can involve:

- getter access;
- reentrancy;
- exceptions;
- multiple calls;
- hostile behavior.

The promise resolution procedure must defend its invariants.

### 10.8 Fulfillment versus reaction execution

Consider:

```js
const p = Promise.resolve("A");

console.log(p);

p.then(console.log);
```

The promise can already be fulfilled when the logging reaction has not run.

This distinction is essential.

---

## 11. ECMAScript / Specification Semantics

### 11.1 Jobs are specification-level execution units

The ECMAScript specification uses Jobs to model deferred execution that cannot occur during the current synchronous evaluation.

Promise behavior is specified in terms of promise reactions and associated jobs.

### 11.2 Promise reaction jobs

A reaction is associated with a promise and a handler.

When the source promise settles, the appropriate reactions are enqueued for later execution.

Conceptually:

```text
settled promise
      ↓
trigger relevant reactions
      ↓
HostEnqueuePromiseJob / equivalent scheduling point
      ↓
reaction job eventually executes
```

The exact specification names and abstract operations should be learned from the relevant ECMAScript edition, but the conceptual architecture is stable:

```text
promise state
→ reaction record
→ job
→ handler execution
→ downstream promise settlement
```

### 11.3 `PerformPromiseThen`

The specification's promise-then machinery establishes reactions and a downstream promise capability.

The important mental model is:

```text
source promise
  +
handlers
  +
result promise
  =
registered reaction relationship
```

### 11.4 `NewPromiseReactionJob`

The specification models the eventual callback execution as a promise reaction job.

Its responsibilities include:

- selecting the correct handler;
- invoking it with the settlement value/reason;
- resolving or rejecting the downstream promise.

### 11.5 Promise resolution

The resolution procedure is not:

```js
promise.value = result;
```

It handles:

- ordinary values;
- promises;
- thenables;
- self-resolution;
- getter errors;
- first-call-wins behavior for resolving functions.

### 11.6 Self-resolution

This is invalid:

```js
let resolvePromise;

const p = new Promise(resolve => {
  resolvePromise = resolve;
});

resolvePromise(p);
```

A promise must not resolve to itself because that would create an impossible recursive dependency.

The specification rejects self-resolution.

### 11.7 First call wins

For promise resolving functions:

```js
resolve(value);
reject(error);
```

the first effective settlement controls the promise.

Subsequent settlement attempts do not change the promise's already-settled state.

### 11.8 Exception handling inside reactions

If a handler throws:

```js
p.then(() => {
  throw error;
});
```

the exception is converted into rejection of the downstream promise.

This is one of the central reasons promise chains can model failure without requiring a synchronous `try/catch` around every callback.

### 11.9 Host interaction

The language defines how promise jobs are created and requests their scheduling through host integration.

The host determines the broader scheduling environment.

Therefore:

```text
ECMAScript controls:
  promise state + reaction semantics + job concept

Host controls:
  when/where jobs are integrated with the runtime's broader event system
```

### 11.10 Not every deferred callback is a promise job

Timers, DOM events, filesystem callbacks, and other host operations may have distinct scheduling behavior.

A reliable architecture never assumes that all asynchronous callbacks share one universal queue.

---

## 12. Advanced Behavior

### 12.1 Already-settled promises still defer reactions

This design prevents promise APIs from becoming timing-dependent.

Bad API behavior would be:

```text
sometimes callback now
sometimes callback later
```

Promise reactions provide a consistent deferred mechanism.

### 12.2 Reaction order

For one promise:

```js
p.then(A);
p.then(B);
p.then(C);
```

the fulfillment reactions are associated in registration order and run correspondingly under the promise reaction scheduling model.

This gives deterministic local ordering.

### 12.3 Chaining creates causal scheduling

Consider:

```js
p.then(A).then(B);
p.then(C);
```

The likely conceptual sequence after `p` fulfills is:

```text
A
C
B
```

Why?

- `A` and `C` are reactions directly attached to `p`;
- `B` is attached to the promise produced by `A`;
- `B` cannot execute until `A` has completed and the intermediate promise has settled.

This is a foundational prediction exercise.

### 12.4 `catch` propagation

Given:

```js
Promise.reject(error)
  .then(null, handle)
  .then(next);
```

If `handle` returns normally, the downstream promise fulfills.

If `handle` throws, the downstream promise rejects again.

Each reaction transforms the promise state.

### 12.5 `finally`

`finally` is designed for cleanup-like behavior.

Conceptually:

```js
p.finally(cleanup)
```

behaves like a transparent continuation that:

- runs cleanup regardless of fulfillment/rejection;
- preserves the original outcome if cleanup completes normally;
- replaces the outcome if cleanup itself fails.

This connects directly to Chapters 29 and 30.

### 12.6 Returning a pending promise

```js
p.then(() => pendingPromise);
```

The downstream promise remains pending until the returned promise settles.

Therefore one reaction job can trigger a much longer asynchronous dependency chain.

### 12.7 Multiple `.then()` calls on the same promise

```js
const p = Promise.resolve();

p.then(() => console.log("A"));
p.then(() => console.log("B"));

```

This creates independent reactions.

One callback does not “consume” the promise result.

### 12.8 Promise sharing

A single promise can have many consumers:

```js
const result = fetchSomething();

componentA(result);
componentB(result);
componentC(result);
```

Each consumer can attach its own reaction chain.

### 12.9 Promise adoption

A promise returned by a callback can be adopted:

```js
p.then(() => asyncOperation());
```

This creates a natural flattening behavior:

```text
outer promise
  ↓
reaction executes
  ↓
inner promise returned
  ↓
outer downstream promise follows inner outcome
```

### 12.10 Thenable interop

Libraries do not have to use native `Promise` objects to be interoperable in every case.

Objects with a callable `then` can participate in promise resolution.

This flexibility also creates complexity.

### 12.11 Thenable getter side effects

Even retrieving a `then` property can execute arbitrary code:

```js
const value = {
  get then() {
    console.log("side effect");
    return resolve => resolve(1);
  }
};
```

Promise resolution must therefore treat thenable assimilation as potentially effectful.

### 12.12 Microtask queue terminology

Browsers and Node.js commonly expose or document a “microtask queue.”

It is useful practical terminology.

But when reasoning from the specification, start with:

```text
Jobs
PromiseReactionJobs
host scheduling
```

Then map that model onto the runtime.

### 12.13 Job starvation

Consider:

```js
function loop() {
  queueMicrotask(loop);
}

loop();
```

This continually adds more deferred work.

A runtime that repeatedly drains such work can delay other categories of work.

This is a design hazard for:

- recursive promise chains;
- reactive systems;
- task schedulers;
- queue processors.

### 12.14 Fairness is not automatic

A local chain can be logically correct but globally unfair.

A production scheduler may need:

- batching;
- yielding;
- bounded work;
- explicit task queues.

### 12.15 Error boundaries in reaction chains

Example:

```js
p
  .then(stepA)
  .then(stepB)
  .catch(handle);
```

A rejection can skip fulfillment handlers until a rejection handler is encountered.

This creates a structured error-propagation path.

---

## 13. Edge Cases

### 13.1 Promise already fulfilled before `.then()`

The callback still runs through the asynchronous reaction mechanism.

### 13.2 Promise already rejected before `.catch()`

The rejection handler is attached and scheduled according to promise reaction semantics.

### 13.3 Handler returns `undefined`

The downstream promise fulfills with `undefined` when the handler completes normally.

### 13.4 Handler throws

The downstream promise rejects with the thrown value.

### 13.5 Handler returns a rejected promise

The downstream promise becomes rejected according to promise resolution/adoption.

### 13.6 Handler returns itself indirectly

Self-referential promise chains can create cycles.

The promise resolution model prevents direct self-resolution.

More complicated logical cycles may remain pending rather than producing a useful value.

### 13.7 Thenable calls resolve twice

Example:

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

### 13.8 Thenable throws after resolving

```js
const thenable = {
  then(resolve) {
    resolve("A");
    throw new Error("later");
  }
};
```

The promise resolution machinery prevents the later exception from replacing the already-effective resolution.

### 13.9 `.finally()` changes the outcome

```js
Promise.resolve("A")
  .finally(() => {
    throw new Error("cleanup");
  });
```

The resulting promise rejects with the cleanup failure.

### 13.10 Multiple reaction callbacks

If several reactions are registered before settlement, they are associated with the same promise and processed according to the defined ordering.

### 13.11 Nested jobs

A reaction can queue another microtask/job:

```js
Promise.resolve().then(() => {
  console.log("A");
  queueMicrotask(() => console.log("B"));
});
```

The new scheduled work belongs to a later point in the scheduling sequence, not the current callback body.

### 13.12 Recursive scheduling

A callback that always schedules another callback can create effectively unbounded deferred work.

### 13.13 Unhandled rejection timing

Whether and when an environment reports an unhandled rejection is partly host/runtime policy.

Do not treat unhandled-rejection reporting as pure ECMAScript semantics.

### 13.14 Cross-runtime differences

Browser and Node behavior around unhandled rejections, additional queues, and lifecycle can differ.

The language-level promise semantics remain the foundation.

---

## 14. Common Misconceptions

### Misconception 1 — “A fulfilled promise executes its callbacks immediately.”

No.

Settlement and callback execution are separate.

### Misconception 2 — “`.then()` is basically a function call.”

No.

It registers a reaction and creates a downstream promise.

### Misconception 3 — “A promise is a queue.”

A promise has state and reactions; job scheduling is the mechanism through which reactions execute.

### Misconception 4 — “The event loop is the promise job queue.”

Not exactly.

The host event loop contains broader scheduling machinery.

### Misconception 5 — “Every async callback is a microtask.”

No.

Timers, I/O, events, and other host facilities may use different scheduling paths.

### Misconception 6 — “Returning a promise from `.then()` creates a nested promise.”

Promise resolution adopts the returned promise/thenable outcome.

The downstream result is flattened conceptually.

### Misconception 7 — “`catch()` changes the original promise.”

No.

It creates a new downstream promise.

### Misconception 8 — “Throwing in a `.then()` callback escapes directly to the outer `try/catch`.”

Not generally.

The throw becomes rejection of the callback's resulting promise.

### Misconception 9 — “`finally()` always preserves the original outcome.”

Only when its callback completes normally.

### Misconception 10 — “Jobs mean another thread.”

No.

A job is a scheduling abstraction, not a thread.

---

## 15. Common Mistakes

### Mistake 1 — Assuming immediate execution

```js
Promise.resolve().then(fn);
doSomething();
```

Expecting `fn` to run before `doSomething()`.

### Mistake 2 — Ignoring downstream promises

```js
p.then(step);
```

without determining who observes rejection from `step`.

### Mistake 3 — Returning the wrong value

```js
p.then(() => {
  doAsyncWork();
});
```

If the intention was to wait for `doAsyncWork()`, failing to return/await it can create detached work.

Prefer:

```js
p.then(() => {
  return doAsyncWork();
});
```

or:

```js
p.then(async () => {
  await doAsyncWork();
});
```

### Mistake 4 — Confusing registration order with completion order across different promises

Registration ordering is local to a given reaction relationship.

Different asynchronous sources can complete at different times.

### Mistake 5 — Creating unbounded reaction chains

### Mistake 6 — Assuming one queue explains the entire runtime

### Mistake 7 — Using `finally` for state mutation without understanding outcome replacement

### Mistake 8 — Ignoring thenable assimilation

Interop objects can execute arbitrary code.

### Mistake 9 — Relying on unhandled-rejection reporting for correctness

Application code should explicitly observe important promises.

### Mistake 10 — Treating microtask scheduling as a performance freebie

Excessive deferred work can create latency and fairness problems.

---

## 16. Comparison With Related Concepts

| Concept | Meaning | Layer |
|---|---|---|
| Promise | Eventual completion state + reactions | ECMAScript |
| Promise reaction | Registered fulfillment/rejection behavior | ECMAScript |
| Promise Reaction Job | Scheduled execution of a reaction | ECMAScript |
| Job | General specification-level deferred execution unit | ECMAScript |
| Microtask | Common host/runtime term for a high-priority deferred queue | Host/runtime terminology |
| Task/macrotask | Host scheduling category | Browser/runtime |
| Timer callback | Host-scheduled callback | Host |
| Event callback | Host-scheduled callback | Host |
| `queueMicrotask` callback | Host-visible microtask scheduling API | Host |
| Call stack | Active synchronous execution | Engine/runtime |
| Worker thread | Separate execution agent | Host/runtime |

### Promise reaction vs callback

A callback is simply a function value used by some API.

A promise reaction is a structured record connecting:

```text
source promise
+
handler
+
result promise
```

### Job vs microtask

A Job is the specification-level concept.

“Microtask” is a widely used runtime term describing one class of deferred execution behavior.

Keep the terminology layered rather than treating them as exact synonyms in every context.

### Promise job vs timer task

A promise reaction can be processed through the promise/job mechanism.

A timer callback belongs to host timer scheduling.

The runtime may order these categories differently from naive “FIFO of everything” reasoning.

---

## 17. Performance Considerations

### 17.1 Each chain stage creates scheduling work

Long chains:

```js
p
  .then(a)
  .then(b)
  .then(c)
  .then(d);
```

can create multiple promise objects and reaction jobs.

Usually this is acceptable, but high-frequency paths may make allocation and scheduling overhead relevant.

### 17.2 Excessive microtask work

A large quantity of promise callbacks can delay other runtime work.

### 17.3 Job batching

A producer that schedules work one item at a time may create substantial scheduling overhead.

Batching can reduce:

```text
queue operations
allocations
context switching
```

### 17.4 Promise allocation

Each `.then()` creates a downstream promise.

Deep pipelines can therefore increase:

- allocations;
- GC pressure;
- bookkeeping.

### 17.5 Thenable assimilation

Assimilating foreign thenables can involve dynamic property access and arbitrary user code.

### 17.6 Error stacks

Rejected promises involving many created errors can increase diagnostic cost.

### 17.7 Concurrency versus queue depth

A fast producer creating jobs faster than consumers can process them can generate memory pressure even when individual callbacks are small.

### 17.8 Scheduling latency

A correct promise chain can still suffer latency because its reaction jobs depend on other work ahead of them.

Measure:

```text
operation latency
+
queue delay
+
handler CPU time
+
downstream latency
```

rather than looking only at promise creation time.

---

## 18. Memory Considerations

Promises and reaction records can retain references.

For example:

```js
const huge = createHugeObject();

somePromise.then(() => {
  use(huge);
});
```

The closure can keep `huge` alive until the reaction no longer needs it.

### Promise chains

Long-lived pending promises may retain:

- reaction handlers;
- closures;
- intermediate promises;
- captured resources.

### Detached promises

A forgotten promise may retain data and resources longer than intended.

### Cycles

Promise graphs can form complex object/reference structures.

GC handles memory reachability, but logical lifecycle bugs can still retain objects unnecessarily.

### Queue growth

Repeated scheduling:

```js
queueMicrotask(produceMoreWork);
```

can retain an increasing amount of state.

### Diagnostic metadata

Errors and causes attached to rejected promises can also retain objects.

---

## 19. Security Considerations

### 19.1 Thenable execution

Thenable assimilation can invoke attacker-controlled code.

Never assume:

```js
value.then
```

is a harmless property lookup.

### 19.2 Promise rejection data

Rejected values may contain:

- credentials;
- tokens;
- database details;
- internal paths.

Do not log or expose them automatically.

### 19.3 Job flooding

An attacker may trigger code paths that create huge numbers of promises or microtasks.

Potential effects:

- CPU exhaustion;
- event-loop delay;
- memory pressure;
- downstream overload.

### 19.4 Async race vulnerabilities

Promise scheduling can widen the time between:

```text
validate
```

and:

```text
use
```

### 19.5 Cleanup failures

A rejected cleanup promise can create secondary failure paths.

### 19.6 Unhandled rejection behavior

Different runtimes may react differently to unhandled rejections.

Production systems should not depend on a particular default.

### 19.7 Supply-chain thenables

Third-party libraries may return thenables or promise-like objects.

Normalize and validate boundaries carefully when security is important.

---

## 20. Production Usage

### 20.1 API service pipeline

```text
request
  ↓
validation
  ↓
async dependency
  ↓
promise reaction
  ↓
business transformation
  ↓
response
```

Each stage should define:

- success;
- failure;
- timeout;
- cancellation;
- cleanup.

### 20.2 Error boundary

Promise chains should terminate at explicit error boundaries:

```js
runOperation()
  .then(handleResult)
  .catch(handleFailure);
```

A production boundary should not merely log and discard failure.

### 20.3 Concurrency management

Promises make it easy to write:

```js
Promise.all(items.map(process));
```

But an application may need a bounded scheduler instead.

### 20.4 Resource ownership

A reaction must not outlive its resource owner:

```text
resource scope
   ↓
async operation
   ↓
promise reactions
   ↓
cleanup
```

This connects directly to Chapter 30.

### 20.5 Observability

Track:

- promise operation latency;
- queue delay where measurable;
- rejection rates;
- retry counts;
- active concurrency;
- timeout rates;
- cancellation.

### 20.6 Background work

Do not accidentally create:

```js
doSomething().then(report);
return response;
```

without deciding:

- who owns the continuation;
- what happens if it fails;
- whether it must finish before response;
- whether it should be cancelled.

### 20.7 Graceful shutdown

A production service should know whether pending asynchronous work is:

```text
required before shutdown
safe to abandon
safe to retry elsewhere
required to be cancelled
```

### 20.8 Libraries

Libraries should document:

- returned promise semantics;
- rejection behavior;
- whether callbacks are always deferred;
- cancellation behavior;
- cleanup expectations.

---

## 21. Implementation From Scratch

The goal is not to recreate every promise specification detail immediately. The goal is to understand the relationship between:

```text
promise state
reaction registration
job scheduling
handler execution
downstream settlement
```

### Stage 1 — Guided reaction queue

Implement a tiny scheduler:

```js
class JobQueue {
  constructor() {
    this.queue = [];
  }

  enqueue(job) {
    this.queue.push(job);
  }

  runNext() {
    const job = this.queue.shift();

    if (job) {
      job();
    }
  }
}
```

Then simulate:

```text
current code
↓
enqueue reaction
↓
finish synchronous work
↓
run reaction
```

### Stage 2 — Partially Guided

Build a minimal promise-like object:

```js
class MiniPromise {
  constructor(executor) {}
  then(onFulfilled, onRejected) {}
}
```

Support:

- pending;
- fulfilled;
- rejected;
- reaction registration;
- deferred reaction execution.

### Stage 3 — No Reference

Implement:

```js
MiniPromise.resolve(value)
MiniPromise.reject(error)
```

and chaining:

```js
new MiniPromise(resolve => resolve(1))
  .then(x => x + 1)
  .then(console.log);
```

### Stage 4 — Edge-Case Hardened

Add:

- thrown executor errors;
- handler throws;
- returned promises;
- thenables;
- self-resolution protection;
- first-call-wins;
- multiple reactions;
- rejection propagation.

### Stage 5 — Production-Oriented Learning Implementation

Build a documented miniature promise engine containing:

```text
state machine
reaction records
job queue
resolution procedure
thenable assimilation
error propagation
debug instrumentation
```

Do not deploy this as a production Promise replacement.

The purpose is semantic understanding.

---

## 22. Debugging Exercises

### Exercise 1 — Fulfilled does not mean callback already ran

```js
const p = Promise.resolve("A");

console.log("before");

p.then(value => console.log(value));

console.log("after");
```

Explain:

```text
promise state
vs
reaction execution
```

### Exercise 2 — Chain ordering

Predict:

```js
const p = Promise.resolve();

p.then(() => console.log("A"));
p.then(() => console.log("B"));
p.then(() => console.log("C"));

console.log("D");
```

### Exercise 3 — Chain dependency

Predict:

```js
Promise.resolve()
  .then(() => console.log("A"))
  .then(() => console.log("B"));

Promise.resolve().then(() => console.log("C"));
```

Explain why the result is not based on simple source-code indentation.

### Exercise 4 — Error propagation

```js
Promise.resolve()
  .then(() => {
    throw new Error("A");
  })
  .then(
    () => console.log("success"),
    error => console.log(error.message)
  );
```

Identify which handler executes and why.

### Exercise 5 — Returned promise

```js
Promise.resolve()
  .then(() => new Promise(resolve => {
    setTimeout(() => resolve("A"), 10);
  }))
  .then(console.log);
```

Which callback waits for the inner promise?

### Exercise 6 — Thenable

```js
Promise.resolve()
  .then(() => ({
    then(resolve) {
      resolve("A");
    }
  }))
  .then(console.log);
```

Explain the assimilation step.

### Exercise 7 — Starvation

```js
let count = 0;

function loop() {
  count++;

  if (count < 100000) {
    queueMicrotask(loop);
  }
}

loop();
```

What categories of work might be delayed while this executes?

---

## 23. Code Review Exercise

Review:

```js
function process(items) {
  items.forEach(item => {
    fetchItem(item)
      .then(result => saveResult(result))
      .catch(error => console.error(error));
  });
}
```

Analyze:

- concurrency;
- ordering;
- error ownership;
- promise observation;
- completion signaling;
- backpressure;
- cancellation;
- graceful shutdown;
- whether `process()` should return a promise;
- whether errors are being swallowed at the wrong layer.

Redesign the function with an explicit completion contract.

---

## 24. Interview Questions

### Foundational

1. What is an ECMAScript Job?
2. Why are promise callbacks deferred?
3. What is a promise reaction?
4. What happens when `.then()` is called?
5. Why does `.then()` return a new promise?
6. What happens when a reaction returns a value?
7. What happens when it throws?
8. What happens when it returns another promise?
9. What is thenable assimilation?
10. Why does a fulfilled promise still defer its reactions?

### Intermediate

11. What is the difference between a promise and a promise reaction job?
12. What is the difference between a Job and a browser task?
13. What does `catch()` do conceptually?
14. How does `finally()` affect settlement?
15. Why can chain stages execute at different times?
16. How is rejection propagated?
17. What does first-call-wins mean for promise resolution?
18. Why is self-resolution invalid?
19. Why can recursive microtasks create starvation?
20. Why is `Promise.all` not itself a scheduler?

### Advanced

21. Explain the lifecycle of a promise reaction.
22. Explain the role of the downstream promise capability.
23. Explain promise resolution versus fulfillment.
24. Explain thenable assimilation and its security implications.
25. Predict reaction ordering for multiple chains.
26. Explain how a thrown reaction callback becomes a rejection.
27. Explain why host event loops cannot be reduced to “the promise queue.”
28. Explain how promise chains retain memory.
29. Explain how job flooding can affect latency.
30. Explain the difference between specification scheduling and runtime scheduling.

### Principal-Level

31. Design an async abstraction with deterministic completion semantics.
32. How would you prevent microtask starvation?
33. How would you design bounded promise concurrency?
34. How would you debug hidden scheduling latency?
35. How would you classify detached promise chains in a production system?
36. How would you instrument promise failures without duplicate logging?
37. How would you model cancellation across chained promises?
38. How would you preserve resource ownership across promise boundaries?
39. When should a library expose a promise versus a callback?
40. How would you explain promise scheduling to engineers without collapsing specification and host concepts?

---

## 25. Predict-the-Output Exercises

### Exercise A

```js
console.log("A");

Promise.resolve().then(() => console.log("B"));

console.log("C");
```

Expected:

```text
A
C
B
```

### Exercise B

```js
const p = Promise.resolve();

p.then(() => console.log("A"));
p.then(() => console.log("B"));

console.log("C");
```

Expected:

```text
C
A
B
```

### Exercise C

```js
Promise.resolve()
  .then(() => console.log("A"))
  .then(() => console.log("B"));

Promise.resolve().then(() => console.log("C"));
```

Predict the ordering and explain why `C` can run before `B`.

### Exercise D

```js
Promise.resolve()
  .then(() => {
    console.log("A");
    return Promise.resolve("B");
  })
  .then(console.log);
```

Explain why the second callback does not receive the promise object itself.

### Exercise E

```js
Promise.resolve()
  .then(() => {
    throw new Error("A");
  })
  .catch(error => {
    console.log(error.message);
    return "B";
  })
  .then(value => {
    console.log(value);
  });
```

Predict:

```text
A
B
```

### Exercise F

```js
Promise.resolve()
  .then(() => console.log("A"))
  .finally(() => console.log("B"))
  .then(() => console.log("C"));
```

Explain the sequence and outcome.

---

## 26. Mastery Exercises

### Exercise 1 — Draw the job graph

For:

```js
p
  .then(a)
  .then(b);

p.then(c);
```

Draw:

```text
promise
reactions
jobs
downstream promises
```

Then predict execution ordering.

### Exercise 2 — Mini reaction engine

Implement:

```js
class ReactionQueue {
  enqueue(reaction) {}
  drain() {}
}
```

Then connect it to a miniature promise abstraction.

### Exercise 3 — Thenable assimilation

Implement:

```js
resolvePromise(value)
```

handling:

```text
ordinary value
native-like promise
thenable
throwing then getter
double resolve
self-resolution
```

### Exercise 4 — Promise scheduler instrumentation

Create a debugging representation:

```text
job #1
  source promise: P1
  reaction: onFulfilled
  created downstream: P2
```

Trace a chain through completion.

### Exercise 5 — Starvation detector

Build a scheduler that detects excessive recursive deferred work.

### Exercise 6 — Concurrency controller

Implement:

```js
mapWithConcurrency(items, limit, worker)
```

using promises.

Compare:

```text
unbounded
bounded
sequential
```

### Exercise 7 — Production review

Analyze a real async service and identify:

- reaction chains;
- error boundaries;
- detached promises;
- resource scope;
- cancellation;
- concurrency;
- queueing;
- observability.

Produce a failure and latency model, not just code changes.

---

## 27. Key Takeaways

1. ECMAScript Jobs are a specification-level model for deferred execution.
2. Promise reactions are executed through scheduled reaction jobs rather than immediate callback calls.
3. Promise settlement and reaction execution are distinct events.
4. `.then()` establishes a reaction and creates a downstream promise.
5. Returning a value fulfills the downstream promise.
6. Throwing from a reaction rejects the downstream promise.
7. Returning a promise or thenable causes promise-resolution/adoption behavior.
8. Multiple reactions attached to the same promise follow deterministic registration ordering.
9. Chained reactions depend on intermediate promise settlement.
10. Promise resolution is more sophisticated than assigning a value.
11. Thenables can execute arbitrary code during assimilation.
12. Self-resolution is invalid.
13. Promise resolving functions use first-effective-settlement behavior.
14. `catch()` creates another promise in the chain.
15. `finally()` can preserve or replace the original outcome depending on its completion.
16. “Microtask” is useful runtime terminology but should not replace precise ECMAScript Job reasoning.
17. A promise job is not a thread.
18. Excessive job creation can harm fairness, latency, CPU, and memory.
19. Host event loops contain scheduling behavior beyond the ECMAScript promise job model.
20. The central mental model is:

> Promise state determines which reactions become eligible; Jobs provide the deferred execution mechanism that runs those reactions; the host determines how that work is integrated into the broader runtime.

---

## 28. Concept Connections

### Depends On

- Chapter 09 — Functions / First-Class Behavior
- Chapter 12 — Execution Contexts / Execution Model
- Chapter 25 — Iterables / Iterators
- Chapter 29 — Errors / Error Handling
- Chapter 31 — Asynchronous JavaScript Fundamentals

### Builds Toward

- Chapter 33 — Browser Event Loop
- Chapter 34 — Node Event Loop / libuv
- Chapter 35 — Promises
- Chapter 36 — Async/Await
- Chapter 37 — Cancellation / Abort
- Chapter 38 — Async Iteration / Streaming
- Chapter 39 — Concurrency / Parallelism
- Chapter 40 — Observables / Reactive
- Chapter 45 — Memory / GC
- Chapter 46 — Weak Refs / Finalization
- Chapter 53 — Web Streams / Data Flow
- Chapter 55 — Fetch / HTTP Networking
- Chapter 60 — Node Streams
- Chapter 61 — Worker Threads / Child Processes / Cluster
- Chapter 63 — Async Context / Diagnostics
- Chapter 83 — Observability
- Chapter 84 — Reliability
- Chapter 85 — Performance
- Chapter 86 — Testing
- Chapter 87 — Deterministic Async Testing
- Chapter 88 — Debugging Methodology
- Chapter 98 — Anti-patterns / Failure Modes
- Chapter 100 — Cost Model / Tradeoffs
- Chapter 101 — Real-world Production Scenarios
- Chapter 107 — Job Queue
- Chapter 110 — Production JS Backend
- Chapter 111 — Large-scale JS Platform
- Chapter 121 — System Design
- Chapter 122 — Final Principal JS Project

### Related Concepts

- Promises
- Promise reactions
- Jobs
- Microtasks
- Tasks
- Event loops
- Thenables
- Promise resolution
- Async functions
- Error propagation
- Cancellation
- Backpressure
- Concurrency limits
- Resource ownership
- Observability

### Concepts Revisited

This chapter revisits:

- functions;
- execution contexts;
- abrupt completion;
- errors;
- asynchronous control flow;
- resource lifetime.

### Why This Chapter Matters Later

The most common async explanations skip directly to:

```text
event loop
microtask queue
macrotask queue
```

without first establishing what the JavaScript language itself specifies.

That produces fragile mental models.

This chapter establishes the lower-level foundation:

```text
Promise
→ reaction
→ job
→ handler
→ downstream settlement
```

Only after this model is clear should the learner study browser and Node-specific scheduling systems.

The central principle is:

> First understand what the language schedules; then understand how the host schedules everything around it.

---

## 29. Completion Criteria

Mark Chapter 32 `[+] Completed` only after the learner can demonstrate all of the following.

### Conceptual Understanding

- [ ] Define an ECMAScript Job.
- [ ] Define a promise reaction.
- [ ] Explain why promise callbacks are deferred.
- [ ] Distinguish settlement from reaction execution.
- [ ] Explain downstream promise creation.
- [ ] Explain promise resolution/adoption.
- [ ] Explain thenables.
- [ ] Explain self-resolution.
- [ ] Explain first-effective-settlement behavior.
- [ ] Explain specification vs host scheduling.

### Predictive Mastery

- [ ] Predict multiple reaction ordering.
- [ ] Predict chained reaction ordering.
- [ ] Predict thrown reaction behavior.
- [ ] Predict returned-promise behavior.
- [ ] Predict `catch()` propagation.
- [ ] Predict `finally()` behavior.
- [ ] Predict thenable assimilation outcomes.
- [ ] Predict nested job scheduling.

### Implementation

- [ ] Implement a basic job queue.
- [ ] Implement a miniature reaction mechanism.
- [ ] Implement promise chaining.
- [ ] Implement basic thenable assimilation.
- [ ] Implement first-call-wins behavior.
- [ ] Implement self-resolution protection.
- [ ] Instrument reaction/job execution.

### Debugging

- [ ] Trace a promise chain step by step.
- [ ] Distinguish settlement from callback execution.
- [ ] Identify missing promise returns.
- [ ] Identify detached promise chains.
- [ ] Identify job starvation risks.
- [ ] Identify memory retention through closures/reactions.
- [ ] Identify host-vs-language scheduling confusion.

### Production Engineering

- [ ] Design explicit promise completion boundaries.
- [ ] Define error propagation.
- [ ] Define concurrency limits.
- [ ] Define cancellation behavior.
- [ ] Preserve resource ownership across reactions.
- [ ] Instrument asynchronous failures.
- [ ] Avoid unbounded scheduling.

### Interview Readiness

- [ ] Explain Jobs without conflating them with event loops.
- [ ] Explain promise reaction jobs precisely.
- [ ] Explain thenable assimilation.
- [ ] Predict chained reaction ordering.
- [ ] Defend bounded async scheduling.
- [ ] Explain promise error propagation.
- [ ] Distinguish specification semantics from runtime behavior.

### Track A — Core Theory

- [ ] Understand Jobs.
- [ ] Understand reaction records.
- [ ] Understand promise resolution.
- [ ] Understand thenables.
- [ ] Understand host integration.

### Track B — Implementation

- [ ] Guided implementation completed.
- [ ] Partially guided implementation completed.
- [ ] No-reference implementation completed.
- [ ] Edge-case hardened implementation completed.
- [ ] Production-oriented learning implementation reviewed.

### Track C — Interview / Reasoning

- [ ] Completed output prediction.
- [ ] Completed reaction graph exercise.
- [ ] Completed debugging scenarios.
- [ ] Completed code review exercise.
- [ ] Defended scheduling behavior at specification and production levels.

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

# Chapter 32 — Revision / Retrieval Record

### Retrieval Prompts

1. What is an ECMAScript Job?
2. What is a Promise Reaction Job?
3. Why does a fulfilled promise still defer its reactions?
4. What does `.then()` create?
5. Why does `.then()` return a new promise?
6. What happens when a reaction returns a value?
7. What happens when a reaction throws?
8. What happens when a reaction returns a promise?
9. What is thenable assimilation?
10. Why is self-resolution invalid?
11. What is first-effective-settlement behavior?
12. How does `catch()` transform the chain?
13. How does `finally()` affect the result?
14. Why should Jobs not be confused with the event loop?
15. Why can recursive microtask/job scheduling cause starvation?
16. How can promise chains retain memory?
17. How do specification-level jobs interact with host scheduling?
18. How would you debug a complex reaction-ordering problem?

### Weak Areas

```text
-
-
-
```

### Revision Queue

```text
- [ ] Revisit Job vs host scheduling
- [ ] Revisit Promise Reaction Jobs
- [ ] Revisit promise resolution
- [ ] Revisit thenable assimilation
- [ ] Revisit chain ordering
- [ ] Revisit job starvation
- [ ] Revisit promise memory retention
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

# Chapter 32 — Canonical References and Source Discipline

Use this source hierarchy:

1. ECMAScript specification — Jobs, Promise reaction semantics, promise resolution, `then`, `catch`, `finally`, async-related scheduling primitives, and completion behavior.
2. TC39 proposal/history material where required for historical evolution or terminology changes.
3. JavaScript engine documentation / implementation notes — optimization, diagnostics, and implementation details.
4. Browser runtime documentation — microtask processing, tasks, timers, rendering, and host scheduling behavior.
5. Node.js/runtime documentation — microtask handling, `process.nextTick`, timers, libuv phases, lifecycle, and runtime-specific behavior.
6. Application architecture documentation — queueing, concurrency limits, observability, retries, cancellation, and ownership.

When comparing Jobs with microtasks, explicitly identify which statement is:

```text
ECMAScript semantic
runtime terminology
host implementation detail
application policy
```

Do not use a browser or Node event-loop diagram as a substitute for ECMAScript promise semantics.

---

# Chapter 32 — Completion Snapshot

```text
Chapter: 32
Title: ECMAScript Jobs and Promise Reactions
Part: VI — Async
Status: [+] Expanded
Track A: [ ] Core Theory
Track B: [ ] Implementation
Track C: [ ] Interview / Reasoning
Mastery: [ ] Not Mastered
```
