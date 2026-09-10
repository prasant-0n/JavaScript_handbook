
# Chapter 36 — Async Functions and `await`

## 1. Learning Objectives

By the end of this chapter, the learner must be able to:

- Explain what an `async` function is and what contract it provides.
- Explain why every call to an async function produces a Promise.
- Distinguish the synchronous portion of an async function from its suspended continuation.
- Explain `await` as a control-flow mechanism built around Promise resolution.
- Explain why `await` does not block the entire JavaScript execution environment.
- Explain how `await` affects the returned Promise of an async function.
- Explain how synchronous throws inside async functions become Promise rejections.
- Explain how rejected awaited Promises become throw-like failures inside the async function.
- Predict execution ordering around `await`.
- Predict behavior when awaiting ordinary values, fulfilled Promises, rejected Promises, and thenables.
- Understand why `await` always introduces asynchronous continuation semantics even when the value is already fulfilled.
- Explain async function return-value normalization.
- Explain how returned thenables are adopted by the async function's result Promise.
- Explain error propagation through `try/catch/finally` around `await`.
- Explain common differences between `return await` and `return promise`.
- Understand sequential versus concurrent `await`.
- Explain accidental serialization in async functions.
- Use `Promise.all`, `allSettled`, `race`, and `any` appropriately inside async code.
- Understand async function stack behavior and debugging boundaries.
- Explain the interaction between `await`, promise reactions, Jobs, microtasks, and host scheduling.
- Explain how `await` interacts with resource lifetime and cleanup.
- Explain cancellation limitations and why `await` alone does not cancel the underlying operation.
- Diagnose unhandled rejections and detached asynchronous work.
- Understand memory retention across suspension points.
- Design production async functions with explicit success, failure, timeout, cancellation, and ownership behavior.
- Implement async-like control flow from scratch for learning.
- Build robust sequential and concurrent async workflows.
- Debug complex async/await ordering without relying on intuition.
- Defend async/await design decisions at senior/principal level.

### Mastery Gate

The chapter is mastered only when the learner can:

> Understand → Explain → Predict → Implement → Debug → Apply → Compare → Defend

---

## 2. Prerequisites

The learner should understand:

- Functions and lexical scope.
- Execution contexts.
- Errors and abrupt completion.
- Promises.
- Promise resolution and chaining.
- ECMAScript Jobs and Promise Reaction Jobs.
- Browser and Node event-loop fundamentals.
- Resource management and cleanup.

Primary dependencies:

- Chapter 12 — Execution Contexts / Execution Model
- Chapter 29 — Errors / Error Handling
- Chapter 30 — Resource Management / Cleanup
- Chapter 31 — Asynchronous JavaScript Fundamentals
- Chapter 32 — ECMAScript Jobs / Promise Reactions
- Chapter 33 — Browser Event Loop
- Chapter 34 — Node Event Loop / libuv
- Chapter 35 — Promises

Later chapters build directly on this chapter:

- Chapter 37 — Cancellation / Abort
- Chapter 38 — Async Iteration / Streaming
- Chapter 39 — Concurrency / Parallelism
- Chapter 63 — Async Context / Diagnostics
- Chapter 84 — Reliability
- Chapter 86 — Testing
- Chapter 87 — Deterministic Async Testing
- Chapter 88 — Debugging Methodology

---

## 3. What Is It?

An `async` function is a JavaScript function whose completion is represented by a Promise.

Example:

```js
async function add(a, b) {
  return a + b;
}
```

Calling:

```js
const result = add(2, 3);
```

does not produce the number directly.

It produces:

```text
Promise → eventually fulfilled with 5
```

An `async` function can also suspend at:

```js
await expression;
```

Example:

```js
async function load() {
  const value = await getValue();
  return value;
}
```

The function can be understood as two broad execution regions:

```text
before await
     ↓
suspend
     ↓
future continuation
     ↓
after await
```

The central idea is:

> `async/await` provides structured control flow for Promise-based asynchronous operations.

It does not create a separate thread.

It does not automatically parallelize work.

It does not automatically cancel work.

It changes how asynchronous completion is expressed and composed.

---

## 4. Why Does It Exist?

Promise chains can become structurally difficult:

```js
getUser()
  .then(user => getProfile(user))
  .then(profile => getPermissions(profile))
  .then(permissions => {
    // ...
  })
  .catch(handleError);
```

The equivalent async style:

```js
async function load() {
  try {
    const user = await getUser();
    const profile = await getProfile(user);
    const permissions = await getPermissions(profile);

    return permissions;
  } catch (error) {
    handleError(error);
  }
}
```

often maps more directly onto the conceptual sequence:

```text
get user
then get profile
then get permissions
```

This improves readability, but readability is not the only reason.

`async/await` also provides a structured control-flow model for:

- local variables across suspension;
- `try/catch/finally`;
- loops;
- conditional logic;
- early returns;
- resource ownership;
- sequential workflows.

The key distinction is:

> Async/await changes the control-flow representation of Promise-based programs; it does not remove asynchronous semantics.

---

## 5. Mental Model

Think of an async function as a state machine.

```text
                    ┌───────────────┐
                    │ running sync  │
                    └───────┬───────┘
                            │
                         await
                            │
                            ▼
                    ┌───────────────┐
                    │   suspended   │
                    └───────┬───────┘
                            │
                 awaited outcome available
                            │
                    ┌───────┴────────┐
                    │                │
                fulfillment       rejection
                    │                │
                    ▼                ▼
                 resume           throw-like
                 with value       failure
                    │                │
                    └───────┬────────┘
                            ▼
                       continue
                            │
                            ▼
                         return
                            │
                            ▼
                   async function Promise
```

Another useful model:

```text
async function call
      ↓
create result Promise
      ↓
start function body
      ↓
reach await
      ↓
pause function progress
      ↓
arrange continuation
      ↓
later resume
      ↓
eventual return/rejection
```

The important distinction:

```text
function is suspended
```

does not mean:

```text
entire JavaScript runtime is blocked
```

Other work can proceed.

---

## 6. Core Rules

### Rule 1 — Calling an async function returns a Promise

```js
async function f() {
  return 42;
}

f() instanceof Promise;
```

Conceptually:

```text
fulfilled Promise<42>
```

### Rule 2 — Code before the first `await` can run synchronously

```js
async function f() {
  console.log("A");
  await something;
  console.log("B");
}
```

Calling `f()` can print `"A"` before the caller continues.

### Rule 3 — `await` suspends the async function's continuation

It does not block the entire JavaScript environment.

### Rule 4 — Awaiting a value normalizes it through Promise-like semantics

```js
await 42
```

is valid.

### Rule 5 — Awaiting a rejected promise causes a throw-like failure inside the async function

```js
try {
  await Promise.reject(error);
} catch (error) {
  // handles rejection
}
```

### Rule 6 — A throw from an async function produces a rejected Promise

```js
async function fail() {
  throw new Error("boom");
}
```

The caller observes:

```text
rejected Promise
```

### Rule 7 — A normal return fulfills the async function's Promise

```js
async function f() {
  return 42;
}
```

### Rule 8 — Returning another Promise is adopted by the async function's completion model

```js
async function f() {
  return somePromise;
}
```

The caller observes the eventual outcome rather than receiving a useful “Promise of Promise” API shape.

### Rule 9 — Every `await` introduces a continuation boundary

Even:

```js
await Promise.resolve(42);
```

does not simply behave like:

```js
const x = 42;
```

inside the async function.

### Rule 10 — Sequential awaits are sequential

```js
const a = await first();
const b = await second();
```

The second operation begins only after the first expression has completed if `second()` is invoked after the first await.

### Rule 11 — Start independent work before awaiting to overlap it

```js
const aPromise = first();
const bPromise = second();

const [a, b] = await Promise.all([aPromise, bPromise]);
```

### Rule 12 — `await` itself is not cancellation

The underlying operation may continue after the caller stops waiting.

### Rule 13 — `try/catch` can catch awaited rejection

```js
try {
  await task();
} catch (error) {
  // handle
}
```

### Rule 14 — Unawaited Promise work requires explicit ownership

```js
task();
```

should never be interpreted as harmless merely because the function is async.

### Rule 15 — Async scope must match resource lifetime

Do not dispose a resource before the async work using it finishes.

### Rule 16 — `return await` and `return promise` are related but not identical in every observable scenario

The difference becomes especially important around local `try/catch/finally`, error transformation, and async stack diagnostics.

---

## 7. Syntax

### Async function declaration

```js
async function fetchData() {
  return value;
}
```

### Async function expression

```js
const fetchData = async function () {
  return value;
};
```

### Async arrow function

```js
const fetchData = async () => {
  return value;
};
```

### Await

```js
const result = await promise;
```

`await` is valid in:

- async functions;
- modules with top-level await where supported by the module system/runtime.

### Error handling

```js
async function run() {
  try {
    const value = await task();
    return value;
  } catch (error) {
    handle(error);
  }
}
```

### Sequential operations

```js
const a = await taskA();
const b = await taskB();
```

### Concurrent operations

```js
const aPromise = taskA();
const bPromise = taskB();

const [a, b] = await Promise.all([
  aPromise,
  bPromise
]);
```

### Top-level await

In an appropriate module:

```js
const config = await loadConfig();
```

Top-level await affects module evaluation and dependency readiness.

The deeper module semantics are covered later in the modules chapters.

---

## 8. Basic Examples

### Example 1 — Basic async return

```js
async function f() {
  return 42;
}

f().then(console.log);
```

Output:

```text
42
```

### Example 2 — Basic await

```js
async function f() {
  const value = await Promise.resolve(42);
  console.log(value);
}

f();
```

Output:

```text
42
```

### Example 3 — Synchronous prefix

```js
async function f() {
  console.log("A");
  await Promise.resolve();
  console.log("B");
}

console.log("C");
f();
console.log("D");
```

Typical output:

```text
C
A
D
B
```

### Example 4 — Rejection

```js
async function fail() {
  throw new Error("boom");
}

fail().catch(error => {
  console.log(error.message);
});
```

### Example 5 — Await rejection

```js
async function run() {
  try {
    await Promise.reject(new Error("boom"));
  } catch (error) {
    console.log(error.message);
  }
}

run();
```

### Example 6 — Sequential operations

```js
async function load() {
  const user = await getUser();
  const profile = await getProfile(user.id);

  return profile;
}
```

### Example 7 — Concurrent independent operations

```js
async function load() {
  const userPromise = getUser();
  const configPromise = getConfig();

  const [user, config] = await Promise.all([
    userPromise,
    configPromise
  ]);

  return { user, config };
}
```

### Example 8 — Loop with sequential awaits

```js
for (const item of items) {
  await process(item);
}
```

This is often intentionally sequential.

### Example 9 — Parallel array processing

```js
const results = await Promise.all(
  items.map(item => process(item))
);
```

This can create unbounded concurrency and therefore requires workload analysis.

---

## 9. Execution Walkthrough

Consider:

```js
async function f() {
  console.log("A");

  const value = await Promise.resolve("B");

  console.log(value);

  return "C";
}

console.log("D");

const promise = f();

console.log("E");

promise.then(value => {
  console.log(value);
});
```

### Step 1

Print:

```text
D
```

### Step 2

Call `f()`.

The async function starts executing its body.

### Step 3

Print:

```text
A
```

### Step 4

Evaluate:

```js
await Promise.resolve("B")
```

The awaited expression is normalized through Promise semantics.

### Step 5

The async function suspends before printing the next line.

The caller receives the async function's result Promise.

### Step 6

Print:

```text
E
```

### Step 7

The awaited continuation is scheduled for later execution.

### Step 8

The async function resumes and prints:

```text
B
```

### Step 9

The function returns:

```text
C
```

The async function's result Promise becomes fulfilled with `"C"`.

### Step 10

Its attached reaction later executes:

```text
C
```

Typical output:

```text
D
A
E
B
C
```

This demonstrates:

```text
async invocation
→ synchronous prefix
→ await suspension
→ continuation
→ final Promise settlement
→ consumer reaction
```

---

## 10. Internal Mechanics

### 10.1 Async function result Promise

Calling an async function creates a Promise representing the function's eventual completion.

Conceptually:

```text
call async function
     ↓
completion capability
     ↓
result Promise
```

### 10.2 Execution before suspension

The body can begin synchronously:

```js
async function f() {
  doSynchronousWork();
  await something();
}
```

`doSynchronousWork()` executes before the first suspension.

### 10.3 Await operation

At a high level:

```text
evaluate awaited expression
        ↓
normalize/adopt as Promise-like outcome
        ↓
suspend current async function
        ↓
register fulfillment/rejection continuations
        ↓
later resume
```

### 10.4 Fulfillment resume

If the awaited outcome fulfills:

```text
resume with fulfillment value
```

### 10.5 Rejection resume

If it rejects:

```text
resume through throw-like failure
```

This is why local `try/catch` works:

```js
try {
  await task();
} catch (error) {
  ...
}
```

### 10.6 Async function return

When the function returns normally:

```text
return value
→ fulfill async result Promise
```

### 10.7 Async function throw

When the function throws:

```text
throw
→ reject async result Promise
```

### 10.8 Returned promise adoption

Example:

```js
async function f() {
  return task();
}
```

The async function's completion follows the returned promise's outcome.

### 10.9 Promise identity nuance

Although:

```js
async function f() {
  return p;
}
```

eventually follows `p`, the returned Promise from the async function is not necessarily the same object `p`.

This matters when reasoning about identity:

```js
const p = Promise.resolve(42);

async function f() {
  return p;
}

f() === p; // false
```

The async function's own Promise completion adopts `p`'s state.

---

## 11. ECMAScript / Specification Semantics

### 11.1 Async functions are part of ECMAScript

Unlike browser-specific timers, `async function` and `await` are language features.

### 11.2 Async function invocation

The specification defines async function invocation so that the function has a Promise-based completion capability.

### 11.3 `await`

The specification's await evaluation performs Promise-like normalization and sets up continuation behavior.

At a conceptual level:

```text
await expression
→ evaluate expression
→ promise-like normalization
→ suspend
→ schedule resume
```

### 11.4 Awaiting ordinary values

```js
await 42;
```

is valid because the await machinery treats the value through Promise resolution semantics.

### 11.5 Awaiting thenables

If:

```js
await {
  then(resolve) {
    resolve(42);
  }
}
```

the thenable protocol can participate in resolution.

### 11.6 Awaiting rejection

A rejection resumes the async function through its throw path.

This is why:

```js
try {
  await Promise.reject(error);
} catch (error) {
  ...
}
```

works.

### 11.7 Async completion

The async function's result Promise is fulfilled when the function completes normally and rejected when it completes abruptly through a throw.

### 11.8 Job scheduling

Continuation execution is connected to Promise/Job machinery.

The exact host integration remains a separate layer.

### 11.9 `return await`

Consider:

```js
async function f() {
  return await g();
}
```

versus:

```js
async function f() {
  return g();
}
```

Both commonly produce the same eventual outcome.

But `return await` causes the function to await `g()` within its own async control flow, which can matter for:

- local `try/catch`;
- local `finally`;
- error transformation;
- observable sequencing;
- debugging stack behavior in some runtimes.

Use it when the surrounding function needs to observe the awaited outcome.

### 11.10 Example where `return await` matters

```js
async function f() {
  try {
    return await g();
  } catch (error) {
    return fallback(error);
  }
}
```

The local `catch` can observe the rejection.

Compare:

```js
async function f() {
  try {
    return g();
  } catch (error) {
    return fallback(error);
  }
}
```

The local `catch` does not catch the later rejection from `g()`.

### 11.11 `return` versus `return await`

Do not adopt:

```text
always use return await
```

or:

```text
never use return await
```

as a universal rule.

The correct choice depends on whether the local function needs to observe the awaited result before completing.

---

## 12. Advanced Behavior

### 12.1 Awaiting an already-fulfilled Promise

Even when:

```js
await Promise.resolve(42)
```

the async function continuation does not simply continue within the same synchronous statement execution.

The continuation is deferred.

### 12.2 Awaiting a rejected Promise

```js
async function f() {
  await Promise.reject(new Error("boom"));
}
```

The returned Promise from `f()` rejects.

### 12.3 Awaiting a thenable

```js
async function f() {
  const value = await {
    then(resolve) {
      resolve(42);
    }
  };

  return value;
}
```

The thenable participates in Promise resolution.

### 12.4 Awaiting a forever-pending Promise

```js
await new Promise(() => {});
```

The async function never reaches subsequent statements unless the awaited operation eventually settles.

Its associated Promise remains pending.

### 12.5 Sequential versus concurrent awaits

Sequential:

```js
await a();
await b();
```

Concurrency:

```js
const pa = a();
const pb = b();

await Promise.all([pa, pb]);
```

The difference is when the underlying operations are initiated.

### 12.6 Conditional concurrency

```js
const aPromise = a();

if (needsB) {
  const bPromise = b();
  return await Promise.all([aPromise, bPromise]);
}

return await aPromise;
```

Be careful: initiating work before knowing it is needed can itself be wasteful.

### 12.7 `await` in loops

A loop can intentionally enforce:

```text
one item at a time
```

This can be correct when:

- order matters;
- rate limits matter;
- each operation depends on the previous.

### 12.8 `forEach` and async

This is a classic bug:

```js
items.forEach(async item => {
  await process(item);
});
```

The outer code does not await the callbacks.

Better:

```js
await Promise.all(
  items.map(item => process(item))
);
```

or sequentially:

```js
for (const item of items) {
  await process(item);
}
```

### 12.9 Async callbacks and APIs

An API that accepts callbacks does not automatically await an async callback:

```js
items.forEach(async item => {
  await work(item);
});
```

The API controls the callback lifecycle.

### 12.10 Async function as callback

```js
button.addEventListener("click", async () => {
  await doWork();
});
```

The browser dispatch mechanism does not automatically await the returned Promise for the purpose of event dispatch.

The Promise exists independently of event listener invocation semantics.

### 12.11 Async errors at host boundaries

Errors in an async event handler often become Promise rejections.

They are not necessarily caught by a surrounding synchronous `try/catch` around the event registration call.

### 12.12 Detached async functions

```js
async function save() {
  await db.write();
}

save();
```

The caller has chosen not to observe the returned Promise.

This can create:

```text
unhandled failure
unclear completion
unclear lifecycle
```

### 12.13 Structured async ownership

A better design usually makes ownership explicit:

```js
await save();
```

or:

```js
const task = save();
registerOwnedTask(task);
```

### 12.14 `finally` with async cleanup

```js
try {
  await work();
} finally {
  await cleanup();
}
```

The function does not finish until the cleanup completes, subject to cleanup success/failure.

### 12.15 Async resource management

This connects directly to:

```js
{
  await using resource = acquire();
  await work(resource);
}
```

The resource lifetime must encompass the awaited work.

### 12.16 Cancellation

This:

```js
await operation();
```

does not imply:

```text
operation is cancellable
```

Cancellation must be communicated through an explicit mechanism such as `AbortSignal`.

### 12.17 Timeout wrappers

```js
await Promise.race([
  operation(),
  timeout(1000)
]);
```

is not cancellation.

### 12.18 Retries

Async functions make retry loops readable:

```js
for (let attempt = 1; attempt <= 3; attempt++) {
  try {
    return await operation();
  } catch (error) {
    if (!shouldRetry(error, attempt)) {
      throw error;
    }
  }
}
```

The policy must define:

- retryable failures;
- backoff;
- jitter;
- cancellation;
- maximum attempts.

### 12.19 Async recursion

Recursive async functions can appear safe because each recursion may suspend:

```js
async function poll() {
  await delay(1000);
  return poll();
}
```

This still creates a perpetual lifecycle and needs explicit cancellation/shutdown.

### 12.20 Async stack traces

Modern engines may provide useful async stack information, but exact stack formatting and retention behavior are implementation-specific.

Use runtime diagnostic tooling rather than relying on universal stack formatting.

---

## 13. Edge Cases

### 13.1 `await` of a primitive

```js
const x = await 42;
```

works.

### 13.2 `await` of a thenable that throws

```js
const value = await {
  get then() {
    throw new Error("boom");
  }
};
```

The async function rejects.

### 13.3 `await` of a thenable that resolves twice

Promise resolution semantics ensure only the first effective settlement wins.

### 13.4 Awaiting a promise that never settles

The async function stays pending.

### 13.5 Returning a rejected Promise inside async

```js
async function f() {
  return Promise.reject("boom");
}
```

The async function's result rejects.

### 13.6 Throwing after an await

```js
async function f() {
  await task();
  throw new Error("boom");
}
```

The async function's result rejects.

### 13.7 `return await` in `finally`

```js
async function f() {
  try {
    return await task();
  } finally {
    await cleanup();
  }
}
```

Cleanup delays function completion.

If cleanup fails, the final result reflects the cleanup failure according to normal abrupt-completion semantics.

### 13.8 `finally` masking

```js
async function f() {
  try {
    throw new Error("A");
  } finally {
    throw new Error("B");
  }
}
```

The resulting Promise rejects with the failure from the final completion path.

### 13.9 `await` inside `finally`

```js
async function f() {
  try {
    return 1;
  } finally {
    await cleanup();
  }
}
```

The function must complete cleanup before the returned Promise fulfills.

### 13.10 Async `forEach`

`forEach` does not await callback Promises.

### 13.11 `map(async ...)`

```js
const values = items.map(async item => process(item));
```

produces an array of Promises, not resolved values.

Use:

```js
const values = await Promise.all(
  items.map(item => process(item))
);
```

### 13.12 Async constructor

Class constructors cannot be declared `async`.

Use a static factory:

```js
class Service {
  static async create() {
    const service = new Service();
    await service.initialize();
    return service;
  }
}
```

### 13.13 Async arrow and lexical `this`

An async arrow preserves lexical `this` just like a normal arrow function.

### 13.14 Top-level await cycle

Module dependency cycles involving top-level await can create complex readiness behavior.

Module semantics must be understood separately.

### 13.15 Awaiting the same pending Promise multiple times

Multiple async functions can await one shared promise.

They resume according to their respective reactions when it settles.

### 13.16 Shared Promise rejection

Many consumers may observe the same rejection.

Failure policy should avoid duplicate noisy handling.

---

## 14. Common Misconceptions

### Misconception 1 — “Async functions run on another thread.”

No.

They use Promise-based asynchronous semantics.

### Misconception 2 — “`await` blocks the thread.”

It suspends the async function's continuation.

It does not block unrelated JavaScript execution.

### Misconception 3 — “Async makes CPU work non-blocking.”

No.

CPU work after or before `await` still executes synchronously on the relevant JavaScript execution agent.

### Misconception 4 — “An async function starts entirely later.”

No.

Its synchronous prefix may run immediately.

### Misconception 5 — “Awaiting an already-resolved Promise is synchronous.”

No.

Continuation remains asynchronous.

### Misconception 6 — “Every await makes independent tasks concurrent.”

No.

`await` often creates sequential execution.

### Misconception 7 — “`await` cancels the Promise when the function exits.”

No.

### Misconception 8 — “`Promise.all` is required for every async operation.”

No.

Sequential execution can be intentional and necessary.

### Misconception 9 — “`forEach` waits for async callbacks.”

No.

### Misconception 10 — “`map(async ...)` produces resolved values.”

No.

It produces Promises.

### Misconception 11 — “Returning a Promise from async creates a Promise of Promise API.”

The async function adopts the returned Promise's eventual outcome.

### Misconception 12 — “`return await` is always bad.”

No.

It is useful when local error/finally handling needs to observe the awaited result.

### Misconception 13 — “`return await` is always necessary.”

No.

If no local observation is needed, directly returning the Promise can be simpler.

### Misconception 14 — “Async event handlers are awaited by the browser.”

Generally, event dispatch does not treat an async listener's returned Promise as a completion contract for the event.

### Misconception 15 — “If the caller does not await, the work stops.”

No.

The async operation can continue.

---

## 15. Common Mistakes

### Mistake 1 — Accidental serialization

```js
await a();
await b();
await c();
```

without checking independence.

### Mistake 2 — Unbounded concurrency

```js
await Promise.all(items.map(process));
```

with huge inputs.

### Mistake 3 — Async `forEach`

### Mistake 4 — Missing `await` on important work

```js
save();
return response;
```

### Mistake 5 — Missing return in Promise callback chains

### Mistake 6 — Treating timeout as cancellation

### Mistake 7 — Catching only synchronous call-site errors

```js
try {
  asyncTask();
} catch {
  // does not catch later rejection
}
```

### Mistake 8 — Resource cleanup before async completion

### Mistake 9 — Detached background work without supervision

### Mistake 10 — Retrying every error

### Mistake 11 — Forgetting backoff and jitter

### Mistake 12 — Creating unnecessary async functions

```js
const f = async x => x;
```

when no asynchronous behavior is needed can add Promise semantics and overhead.

### Mistake 13 — Ignoring pending operation memory

### Mistake 14 — Assuming async means scalable

A system can remain asynchronous while still being bottlenecked by CPU, memory, database capacity, or downstream limits.

---

## 16. Comparison With Related Concepts

| Concept | Main idea | Key difference |
|---|---|---|
| `async` function | Promise-returning function | Completion is Promise-based |
| `await` | Suspend current async function until Promise-like outcome | Structured continuation |
| `.then()` | Register Promise reaction | Explicit chain |
| Callback | API-specific completion callback | No universal Promise contract |
| `Promise.all` | Aggregate concurrent/independent outcomes | Does not create cancellation |
| Generator | Suspends/resumes synchronously around `yield` | Different protocol and completion model |
| Worker | Separate execution agent | Actual parallel JS execution possible |
| Thread | Execution resource | Not created by async/await |
| AbortSignal | Cancellation notification | Separate lifecycle mechanism |
| `try/finally` | Cleanup/control flow | Can span awaits inside async functions |

### Async/await vs `.then()`

Equivalent broad intent:

```js
const value = await task();
```

versus:

```js
return task().then(value => {
  return value;
});
```

But async/await integrates more naturally with ordinary statement-level control flow.

### Async/await vs generators

Generators provide explicit suspension with:

```js
yield
```

Async functions integrate suspension with Promise completion.

### Async/await vs workers

Async/await:

```text
convenient asynchronous control flow
```

Worker:

```text
separate execution agent
```

### Async/await vs cancellation

Async/await:

```text
how to wait
```

Cancellation:

```text
how to stop
```

### Sequential await vs concurrent Promise aggregation

Sequential:

```text
A
↓
B
↓
C
```

Concurrent initiation:

```text
A ─┐
B ─┼→ aggregate
C ─┘
```

---

## 17. Performance Considerations

### 17.1 `await` has continuation overhead

Each suspension/resumption involves Promise/job machinery.

Do not place unnecessary async boundaries in extremely hot synchronous code.

### 17.2 Async function calls create Promise semantics

Calling:

```js
async function f() {
  return 1;
}
```

still creates Promise-based completion.

### 17.3 Sequential awaits increase wall-clock latency

Independent operations should be evaluated for safe concurrent initiation.

### 17.4 Unbounded concurrency increases resource pressure

```js
await Promise.all(hugeArray.map(process));
```

may create massive:

- Promise counts;
- closures;
- network load;
- memory use.

### 17.5 Bounded concurrency

A scheduler can control:

```text
active operations ≤ limit
```

and often provides a better production cost model.

### 17.6 Async does not reduce CPU cost

Heavy processing remains heavy.

### 17.7 Serialization across awaits

Every dependency boundary can add latency.

Measure:

```text
operation time
+
queueing
+
await/scheduling overhead
+
dependency latency
```

### 17.8 `return await`

In modern engines, simplistic claims that `return await` is always a major performance problem are unreliable.

The decision should primarily be semantic unless profiling demonstrates a real hot-path difference.

### 17.9 Promise allocation and GC

Heavy async workloads can create substantial temporary object graphs.

### 17.10 Concurrency is not free

Higher concurrency can reduce latency until a bottleneck becomes saturated.

Beyond that point it can increase:

```text
queueing
timeouts
retries
memory
```

---

## 18. Memory Considerations

### 18.1 Async locals can survive suspension

```js
async function f() {
  const data = createLargeData();
  await slowOperation();
  use(data);
}
```

`data` may remain live across the suspension.

### 18.2 Closures

Continuation functions can retain surrounding state.

### 18.3 Pending promises

Forever-pending operations can retain:

- reactions;
- closures;
- resources.

### 18.4 Large Promise.all

Aggregators can retain:

- input references;
- result arrays;
- reaction state.

### 18.5 Detached work

Background tasks can extend object lifetime unexpectedly.

### 18.6 Cancellation reduces retention

Actually stopping abandoned work can release references earlier.

### 18.7 Async queues

An unbounded queue is an unbounded memory-risk unless downstream capacity matches production rate.

### 18.8 Resource state

An open resource may remain live through an async continuation.

Resource management must therefore include both:

```text
memory lifetime
resource lifetime
```

---

## 19. Security Considerations

### 19.1 Race conditions across await

State can change during suspension:

```text
check
↓
await
↓
use
```

The state at use time may differ.

### 19.2 Authorization windows

Security-sensitive authorization decisions may need to be repeated or tied to immutable state.

### 19.3 Request abandonment

A client may disconnect while server-side async work continues.

The server needs cancellation or lifecycle policies.

### 19.4 Unbounded concurrency

Attackers can exploit fan-out endpoints to exhaust resources.

### 19.5 Timeout without cancellation

A timeout can leave expensive operations running.

### 19.6 Detached async failures

Ignored rejections can hide security-relevant failures.

### 19.7 Error leakage

Async boundaries should preserve internal diagnostics without automatically exposing them.

### 19.8 Resource lifetime

A long-lived async task can keep credentials, buffers, or sockets alive longer than intended.

### 19.9 Stale writes

An older asynchronous result can overwrite newer state unless versioning or cancellation is used.

---

## 20. Production Usage

### 20.1 HTTP handler

```js
async function handler(req, res) {
  try {
    const user = await getUser(req.params.id);
    const data = await buildResponse(user);

    res.json(data);
  } catch (error) {
    handleHttpError(res, error);
  }
}
```

The boundary should explicitly own:

- errors;
- timeout;
- cancellation;
- response lifecycle.

### 20.2 Concurrent dependency loading

```js
async function loadDashboard(id) {
  const userPromise = getUser(id);
  const statsPromise = getStats(id);
  const notificationsPromise = getNotifications(id);

  const [user, stats, notifications] = await Promise.all([
    userPromise,
    statsPromise,
    notificationsPromise
  ]);

  return {
    user,
    stats,
    notifications
  };
}
```

Use concurrency only when the operations are independent and capacity allows it.

### 20.3 Bounded concurrency

```js
async function mapLimited(items, limit, worker) {
  // production implementation required
}
```

Use for:

- batch jobs;
- API fan-out;
- migration scripts;
- image processing;
- queue consumers.

### 20.4 Retry with cancellation

```js
async function retry(operation, options) {
  // bounded attempts + backoff + AbortSignal
}
```

Do not retry blindly.

### 20.5 Transaction lifecycle

```js
async function runTransaction() {
  const connection = await acquire();

  try {
    const tx = await connection.begin();

    try {
      await doWork(tx);
      await tx.commit();
    } catch (error) {
      await tx.rollback();
      throw error;
    }
  } finally {
    await connection.close();
  }
}
```

Later resource-management features can simplify this structure.

### 20.6 Graceful shutdown

A service should track important async work:

```text
in-flight requests
background jobs
resource cleanup
workers
streams
```

### 20.7 Request cancellation

```js
async function handler(req) {
  const controller = createLinkedAbortController(req);

  return fetchDependency({
    signal: controller.signal
  });
}
```

The exact implementation depends on the framework and transport.

### 20.8 Streaming

Async functions often coordinate stream consumption:

```js
for await (const chunk of stream) {
  await process(chunk);
}
```

Chapter 38 goes deeper into async iteration and streaming.

### 20.9 Observability

Instrument:

- operation duration;
- await-boundary latency where useful;
- concurrency;
- retries;
- cancellation;
- rejection;
- dependency timing;
- request context.

### 20.10 Background work

Use explicit ownership:

```text
request-bound
or
application-owned
or
queue-owned
```

Do not accidentally create application-owned work from request-local resources.

---

## 21. Implementation From Scratch

### Stage 1 — Guided

Implement a callback-to-Promise adapter:

```js
function promisifyOne(callbackApi) {
  return (...args) =>
    new Promise((resolve, reject) => {
      callbackApi(...args, (error, value) => {
        if (error) {
          reject(error);
          return;
        }

        resolve(value);
      });
    });
}
```

Then consume it with async/await.

### Stage 2 — Partially Guided

Build:

```js
async function sequentialPipeline(steps) {}
```

Requirements:

- preserve output;
- stop on failure;
- expose original cause;
- execute in order.

### Stage 3 — No Reference

Build:

```js
async function mapConcurrent(items, limit, worker) {}
```

Requirements:

- bounded concurrency;
- preserve result order;
- propagate failure;
- cancellation support;
- cleanup;
- graceful completion.

### Stage 4 — Edge-Case Hardened

Add:

- empty input;
- synchronous worker throws;
- asynchronous rejection;
- cancellation before start;
- cancellation during processing;
- worker timeout;
- partial completion;
- cleanup failure.

### Stage 5 — Production Grade

Implement:

```js
class AsyncExecutor {
  constructor(options) {}

  submit(task, options) {}

  cancel(id) {}

  async close(options) {}

  metrics() {}
}
```

Track:

```text
queued
active
completed
failed
cancelled
timed out
```

The implementation is a learning exercise, not a replacement for the platform's native Promise system.

---

## 22. Debugging Exercises

### Exercise 1 — Ordering

Predict:

```js
async function f() {
  console.log("A");
  await Promise.resolve();
  console.log("B");
}

console.log("C");
f();
console.log("D");
```

### Exercise 2 — Async throw

```js
async function f() {
  throw new Error("boom");
}

try {
  f();
} catch {
  console.log("caught");
}
```

Does `"caught"` print?

Why?

### Exercise 3 — Missing await

```js
async function saveEverything() {
  saveA();
  saveB();
}
```

Does completion guarantee A and B are finished?

### Exercise 4 — `forEach`

```js
await items.forEach(async item => {
  await process(item);
});
```

Why is this wrong?

### Exercise 5 — Sequential bottleneck

```js
for (const id of ids) {
  results.push(await fetch(id));
}
```

Determine whether ordering or concurrency requirements justify the design.

### Exercise 6 — Timeout

```js
await Promise.race([
  operation(),
  timeout(1000)
]);
```

Identify what happens after timeout.

### Exercise 7 — `return await`

Compare:

```js
async function a() {
  try {
    return await task();
  } catch (error) {
    return fallback(error);
  }
}
```

with:

```js
async function b() {
  try {
    return task();
  } catch (error) {
    return fallback(error);
  }
}
```

Explain the behavioral difference.

### Exercise 8 — Resource lifetime

```js
{
  using resource = createResource();
  startAsyncWork(resource);
}
```

Identify the lifecycle bug.

---

## 23. Code Review Exercise

Review:

```js
async function syncUsers(users) {
  users.forEach(async user => {
    try {
      const profile = await fetchProfile(user.id);
      await saveProfile(profile);
    } catch (error) {
      console.error(error);
    }
  });

  return "done";
}
```

Identify at least ten issues involving:

- completion semantics;
- concurrency;
- errors;
- ownership;
- cancellation;
- resource lifetime;
- observability;
- return contract;
- memory;
- shutdown.

Redesign it for production.

---

## 24. Interview Questions

### Foundational

1. What does `async` do?
2. What does an async function return?
3. What does `await` do?
4. Does `await` block the thread?
5. Can async function code execute synchronously?
6. What happens when an async function throws?
7. What happens when `await` receives a rejected Promise?
8. Can you await a normal value?
9. What happens when an async function returns a Promise?
10. Why is `await` useful?

### Intermediate

11. What is the difference between sequential and concurrent awaits?
12. Why is async `forEach` problematic?
13. Why does `map(async ...)` produce Promises?
14. What is accidental serialization?
15. What happens if an awaited Promise never settles?
16. Why doesn't `await` cancel work?
17. When should you use `Promise.all`?
18. When should you use `allSettled`?
19. When is `return await` useful?
20. Why does an async event handler not become an event-dispatch completion contract?

### Advanced

21. Explain async function completion semantics.
22. Explain `await` using Promise resolution and reaction jobs.
23. Explain why awaiting an already-fulfilled Promise still creates a continuation boundary.
24. Explain thenable assimilation through `await`.
25. Explain how exceptions inside async functions become rejections.
26. Explain how awaited rejection becomes a local throw path.
27. Explain async function Promise identity.
28. Explain resource lifetime across await.
29. Explain memory retention across suspension.
30. Explain why timeout is not cancellation.

### Principal-Level

31. Design an async API for a production service.
32. Design bounded concurrency.
33. Design cancellation propagation.
34. Design request-scoped async ownership.
35. Design graceful shutdown for in-flight Promises.
36. Design retry logic with backoff and jitter.
37. Diagnose a service whose latency increased after refactoring Promise chains to async/await.
38. Diagnose an async memory leak.
39. Design a safe detached background-task system.
40. Defend sequential awaits where they reduce concurrency but improve correctness.

---

## 25. Predict-the-Output Exercises

### Exercise A

```js
async function f() {
  console.log("A");
  await Promise.resolve();
  console.log("B");
}

console.log("C");

f();

console.log("D");
```

Expected:

```text
C
A
D
B
```

### Exercise B

```js
async function f() {
  return 42;
}

console.log("A");

f().then(value => {
  console.log(value);
});

console.log("B");
```

Expected:

```text
A
B
42
```

### Exercise C

```js
async function f() {
  try {
    await Promise.reject(new Error("A"));
  } catch (error) {
    console.log(error.message);
    return "B";
  }
}

f().then(value => console.log(value));
```

Expected:

```text
A
B
```

### Exercise D

```js
async function f() {
  console.log("A");
  await Promise.resolve();
  console.log("B");
}

async function g() {
  console.log("C");
  await f();
  console.log("D");
}

g();

Promise.resolve().then(() => {
  console.log("E");
});

console.log("F");
```

Predict carefully.

### Exercise E

```js
const a = async () => {
  console.log("A");
  await null;
  console.log("B");
};

const b = async () => {
  console.log("C");
  await null;
  console.log("D");
};

console.log("E");

a();
b();

console.log("F");
```

Explain:

```text
synchronous prefixes
suspension
continuation ordering
```

### Exercise F

```js
async function f() {
  try {
    return await Promise.reject("A");
  } catch (error) {
    return "B";
  }
}

f().then(console.log);
```

Predict the final value.

---

## 26. Mastery Exercises

### Exercise 1 — Async state machine

Draw the state transitions for:

```js
async function f() {
  const a = await taskA();
  const b = await taskB(a);
  return b;
}
```

Mark:

```text
running
suspended
resumed
fulfilled
rejected
```

### Exercise 2 — Sequential/concurrent comparison

Implement the same workload as:

```text
sequential
fully concurrent
bounded concurrent
```

Measure:

- wall-clock latency;
- memory;
- dependency load;
- failure behavior.

### Exercise 3 — Production async map

Implement:

```js
mapLimited(items, {
  concurrency,
  signal,
  timeout
}, worker)
```

Requirements:

- ordered results;
- bounded active work;
- cancellation;
- timeout;
- error propagation;
- cleanup.

### Exercise 4 — Async transaction

Build a transaction helper that:

```text
acquire
→ begin
→ operation
→ commit OR rollback
→ close
```

Test every failure boundary.

### Exercise 5 — Request ownership

Design:

```text
HTTP request
  ├── async dependency A
  ├── async dependency B
  └── request-scoped resource
```

Ensure no task continues using the resource after the request scope closes.

### Exercise 6 — Retry policy

Implement:

```js
retry(operation, {
  retries,
  baseDelay,
  signal
})
```

Add:

- exponential backoff;
- jitter;
- retry predicate;
- cancellation;
- final error cause.

### Exercise 7 — Background task supervisor

Build:

```js
class TaskSupervisor {
  start(task) {}
  stop(id) {}
  async shutdown() {}
}
```

Requirements:

- ownership;
- error handling;
- cancellation;
- graceful shutdown;
- metrics.

### Exercise 8 — Async memory investigation

Create a workload with:

```text
large closure
pending Promise
long await
```

Measure retained memory and determine when references can be released.

---

## 27. Key Takeaways

1. `async` functions provide Promise-based completion.
2. Calling an async function returns a Promise.
3. Code before the first suspension point can execute synchronously.
4. `await` suspends the current async function's continuation.
5. `await` does not block the whole JavaScript environment.
6. Awaiting ordinary values is valid.
7. Awaiting a rejected Promise creates a throw-like failure path inside the async function.
8. A throw inside an async function becomes rejection of its result Promise.
9. Returning normally fulfills the result Promise.
10. Returning a Promise causes the async completion to follow that Promise's outcome.
11. Awaiting an already-fulfilled Promise still crosses an asynchronous continuation boundary.
12. Sequential awaits are truly sequential when later operations are not initiated until after earlier completion.
13. Independent operations can overlap by being started before the aggregate await.
14. `Promise.all` coordinates concurrent completion but does not provide cancellation.
15. `await` is a waiting/control-flow mechanism, not a cancellation mechanism.
16. `return await` is useful when the local async function needs to observe the awaited outcome.
17. Async callbacks are not automatically awaited by every host API.
18. `forEach(async ...)` does not create an awaitable aggregate.
19. `map(async ...)` creates an array of Promises.
20. Async suspension can retain local variables and closures.
21. Resource scope must encompass all async work that depends on the resource.
22. Timeouts do not automatically stop underlying operations.
23. Unbounded async concurrency can overload memory and downstream systems.
24. Async architecture must define ownership, errors, cancellation, retry, timeout, and shutdown.
25. The central principle is:

> `async/await` makes asynchronous control flow look sequential while preserving asynchronous execution; correctness depends on understanding exactly where execution suspends, what continues concurrently, and who owns the underlying work.

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

### Builds Toward

- Chapter 37 — Cancellation / Abort
- Chapter 38 — Async Iteration / Streaming
- Chapter 39 — Concurrency / Parallelism
- Chapter 40 — Observables / Reactive
- Chapter 45 — Memory / GC
- Chapter 46 — Weak Refs / Finalization
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

- Promise
- Promise reaction
- Job
- Microtask
- Async function
- Await
- Thenable
- Cancellation
- AbortSignal
- Timeout
- Retry
- Backoff
- Concurrency
- Backpressure
- Resource ownership
- Structured concurrency
- Event loop
- Error propagation
- Observability
- Graceful shutdown

### Concepts Revisited

This chapter revisits:

- execution contexts;
- abrupt completion;
- Promise resolution;
- Promise jobs;
- microtasks;
- error handling;
- resource disposal;
- browser and Node event loops.

### Why This Chapter Matters Later

`async/await` is where the Promise and event-loop models become the primary day-to-day syntax of production JavaScript.

The major engineering risk is that readable syntax can hide important scheduling behavior.

This:

```js
await operation();
```

looks simple.

But a principal engineer must still ask:

```text
What operation?
How long?
What resources are held?
Can it be cancelled?
Who owns it?
What happens if it fails?
Is it sequential unnecessarily?
Can it race with another task?
What memory survives across the await?
What happens during shutdown?
```

The central principle is:

> Readable asynchronous code is not automatically correct asynchronous architecture.

---

## 29. Completion Criteria

Mark Chapter 36 `[+] Completed` only after the learner can demonstrate all of the following.

### Conceptual Understanding

- [ ] Explain async functions.
- [ ] Explain async function Promise completion.
- [ ] Explain `await`.
- [ ] Explain suspension/resumption.
- [ ] Explain synchronous prefix execution.
- [ ] Explain await of ordinary values.
- [ ] Explain await of fulfilled/rejected Promises.
- [ ] Explain awaited thenables.
- [ ] Explain async return/throw behavior.
- [ ] Explain `return await`.

### Predictive Mastery

- [ ] Predict async prefix ordering.
- [ ] Predict continuation ordering.
- [ ] Predict rejected-await behavior.
- [ ] Predict chained async execution.
- [ ] Predict sequential vs concurrent workloads.
- [ ] Predict `forEach(async ...)` behavior.
- [ ] Predict `map(async ...)` results.
- [ ] Predict resource lifetime across await.
- [ ] Predict timeout vs cancellation behavior.

### Implementation

- [ ] Build sequential async pipelines.
- [ ] Build concurrent async pipelines.
- [ ] Build bounded async concurrency.
- [ ] Build cancellation-aware async execution.
- [ ] Build timeout-aware operations.
- [ ] Build retry with backoff.
- [ ] Build task supervision.
- [ ] Build async transaction lifecycle.

### Debugging

- [ ] Diagnose missing await.
- [ ] Diagnose accidental serialization.
- [ ] Diagnose unbounded concurrency.
- [ ] Diagnose async `forEach`.
- [ ] Diagnose detached Promises.
- [ ] Diagnose timeout/cancellation confusion.
- [ ] Diagnose resource use-after-disposal.
- [ ] Diagnose async memory retention.
- [ ] Diagnose stale async results.

### Production Engineering

- [ ] Design async API contracts.
- [ ] Define ownership.
- [ ] Define cancellation.
- [ ] Define timeout.
- [ ] Define retry policy.
- [ ] Define graceful shutdown.
- [ ] Instrument async operations.
- [ ] Control concurrency and backpressure.
- [ ] Preserve resource lifetime across awaits.

### Interview Readiness

- [ ] Explain `async`/`await` precisely.
- [ ] Explain why await does not block the thread.
- [ ] Explain synchronous prefix execution.
- [ ] Explain async error propagation.
- [ ] Explain sequential vs concurrent awaits.
- [ ] Explain `return await`.
- [ ] Explain async callback limitations.
- [ ] Design production-grade async workflows.
- [ ] Defend concurrency and ownership decisions.

### Track A — Core Theory

- [ ] Understand async-function completion.
- [ ] Understand await semantics.
- [ ] Understand suspension/resumption.
- [ ] Understand Promise/Job integration.
- [ ] Understand async error flow.

### Track B — Implementation

- [ ] Guided implementation completed.
- [ ] Partially guided implementation completed.
- [ ] No-reference implementation completed.
- [ ] Edge-case hardened implementation completed.
- [ ] Production-oriented async executor reviewed.

### Track C — Interview / Reasoning

- [ ] Completed output prediction.
- [ ] Completed async debugging.
- [ ] Completed code review.
- [ ] Completed bounded concurrency design.
- [ ] Completed cancellation design.
- [ ] Defended async architecture trade-offs.

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

# Chapter 36 — Revision / Retrieval Record

### Retrieval Prompts

1. What does an async function return?
2. What can execute before the first await?
3. What does await suspend?
4. What happens when awaiting an ordinary value?
5. What happens when awaiting a rejected Promise?
6. What happens when an async function throws?
7. What happens when an async function returns a Promise?
8. Why does `await Promise.resolve()` still introduce a continuation boundary?
9. What is the difference between sequential and concurrent awaits?
10. Why is async `forEach` wrong for waiting?
11. Why does `map(async ...)` produce Promises?
12. When is `return await` useful?
13. Why doesn't await cancel the underlying operation?
14. How do resources interact with await?
15. How can async suspension retain memory?
16. How would you implement bounded async concurrency?
17. How would you design cancellation?
18. How would you design graceful shutdown for async work?
19. How would you diagnose accidental serialization?
20. How would you diagnose an async memory leak?

### Weak Areas

```text
-
-
-
```

### Revision Queue

```text
- [ ] Revisit async function completion
- [ ] Revisit await suspension/resumption
- [ ] Revisit sequential vs concurrent awaits
- [ ] Revisit return await
- [ ] Revisit async callback pitfalls
- [ ] Revisit cancellation
- [ ] Revisit resource lifetime
- [ ] Revisit memory retention
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

# Chapter 36 — Canonical References and Source Discipline

Use this source hierarchy:

1. ECMAScript specification — async functions, `await`, Promise completion, async function execution, Promise resolution, Jobs, and completion semantics.
2. TC39 specification/history material — feature evolution or proposal-stage context only where relevant.
3. Browser platform documentation — top-level await/module integration and host scheduling behavior.
4. Node.js documentation — runtime behavior, diagnostics, process lifecycle, and host-specific async APIs.
5. JavaScript engine documentation — implementation, optimization, diagnostics, and performance behavior.
6. Application architecture documentation — cancellation, timeout, retries, ownership, concurrency, backpressure, and shutdown.

Always distinguish:

```text
ECMAScript language semantics
vs
engine implementation
vs
browser behavior
vs
Node.js behavior
vs
application policy
```

Do not present `async/await` as a threading mechanism.

Do not present `await` as cancellation.

Do not claim that all async APIs share one identical host scheduling path.

---

# Chapter 36 — Completion Snapshot

```text
Chapter: 36
Title: Async Functions and `await`
Part: VI — Async
Status: [+] Expanded
Track A: [ ] Core Theory
Track B: [ ] Implementation
Track C: [ ] Interview / Reasoning
Mastery: [ ] Not Mastered
```
