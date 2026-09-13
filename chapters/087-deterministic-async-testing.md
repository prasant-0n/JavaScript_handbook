# Chapter 87 — Deterministic Async Testing

> **Part XVI — Testing / Debugging**  
> **Status:** `[ ] Not Started`  
> **Depth Target:** Principal / Staff-level reasoning  
> **Primary Focus:** Testing asynchronous JavaScript behavior deterministically across Promises, timers, event-loop scheduling, cancellation, retries, concurrency, queues, streams, workers, HTTP, and resource lifecycles.

---

# 1. Learning Objectives

By the end of this chapter, you should be able to:

1. Explain why asynchronous testing is harder than synchronous testing.
2. Distinguish asynchronous completion from concurrency.
3. Distinguish Promise microtasks from timer/task scheduling.
4. Explain why `await` does not automatically make a test deterministic.
5. Design tests that explicitly await asynchronous completion.
6. Control timers without relying on arbitrary sleeps.
7. Test retry/backoff logic without waiting for real time.
8. Test cancellation and abort behavior deterministically.
9. Test deadline propagation.
10. Test timeout behavior without real network delays.
11. Test concurrent operations.
12. Design deterministic race-condition tests.
13. Understand why some races are inherently difficult to reproduce.
14. Coordinate barriers/latches in concurrency tests.
15. Control scheduling with explicit test doubles.
16. Test Promise ordering and settlement semantics.
17. Test event-loop-dependent code.
18. Test microtask/macrotask interactions carefully.
19. Test Node.js timers and timer callbacks.
20. Test callback-style APIs.
21. Test event emitters.
22. Test streams and backpressure.
23. Test async iterators and generators.
24. Test queue consumers and retries.
25. Test background workers.
26. Test process lifecycle and graceful shutdown.
27. Use `AsyncLocalStorage` appropriately in tests and understand context propagation.
28. Understand Node.js test runner concurrency and isolation.
29. Avoid shared-state races between tests.
30. Diagnose flaky asynchronous tests systematically.
31. Build deterministic test clocks and schedulers where appropriate.
32. Distinguish a slow test from a flaky test.
33. Distinguish timing-dependent production behavior from testing artifacts.
34. Design failure injection for async workflows.
35. Test observability and correlation across async boundaries.
36. Build a deterministic async testing toolkit.
37. Evaluate whether test synchronization mechanisms accidentally change the behavior being tested.
38. Build high-confidence async tests without over-mocking.
39. Debug async failures with timelines and causality.
40. Defend an asynchronous testing strategy at principal-engineer level.

---

# 2. Prerequisites

You should understand:

- JavaScript execution semantics.
- Execution contexts and lexical environments.
- Closures.
- `this` and invocation.
- Promises.
- Async/await.
- ECMAScript jobs / microtasks.
- Browser event loop.
- Node.js event loop/libuv.
- Cancellation with `AbortController`.
- Async iteration.
- Streams.
- Node.js process lifecycle.
- Testing fundamentals.
- Reliability.
- Observability.
- Performance.

Recommended prior chapters:

- **10–14** — Scope / Hoisting / Execution / Closures / `this`
- **25–26** — Iterables / Generators
- **29–30** — Errors / Resource Management
- **31–40** — Async Fundamentals / Jobs / Event Loop / Promises / Async-Await / Cancellation / Async Iteration / Concurrency
- **52–53** — Workers / Web Streams
- **58–63** — Node.js / Streams / Workers / Process Lifecycle / Async Context
- **70** — Source Maps / Production Debugging
- **78–85** — Production Architecture / API / Database / Observability / Reliability / Performance
- **86** — Testing

---

# 3. What Is It?

Deterministic async testing means:

> The test controls or observes the asynchronous conditions necessary to make its result reproducible.

A deterministic test should not pass because:

```text
the CI runner happened to be fast
the network happened to respond
a timer happened to fire before another callback
the test ran in a lucky order
the operating system scheduled work favorably
```

Instead:

```text
test controls conditions
→ operation runs
→ known events occur
→ test observes them
→ assertions execute
```

---

## 3.1 Asynchrony versus concurrency

These are related but different.

Asynchronous:

```js
const value = await load();
```

means completion occurs later.

Concurrent:

```js
const a = loadA();
const b = loadB();

await Promise.all([a, b]);
```

means multiple operations can be in progress at the same time.

A system can be asynchronous but sequential.

A system can also be asynchronous and concurrent.

Tests must know which behavior they are validating.

---

# 4. Why Does It Exist?

Asynchronous systems depend on time and ordering.

Examples:

```text
Promise settlement
timer callbacks
network response
database response
queue delivery
abort signal
event listener
stream chunk
worker message
process signal
```

If tests simply sleep:

```js
await sleep(100);
```

they hope the expected event has occurred.

This creates:

```text
slow tests
flaky tests
false positives
false negatives
poor diagnosis
```

Deterministic testing replaces waiting-for-luck with explicit synchronization.

---

# 5. Mental Model

Think of an async test as a controlled timeline.

```text
Time ─────────────────────────────────────>

Test
 │
 ├── arrange
 │
 ├── start operation
 │
 ├── control scheduler
 │
 ├── trigger event
 │
 ├── await observable completion
 │
 └── assert
```

For concurrency:

```text
Task A ──┐
         ├── barrier ──┐
Task B ──┘             │
                       ├── continue
Task C ─────────────────┘
```

For retries:

```text
attempt 1
   ↓
failure
   ↓
virtual delay
   ↓
attempt 2
   ↓
failure
   ↓
virtual delay
   ↓
attempt 3
```

The test should decide when the virtual delays advance.

---

# 6. Core Rules

## Rule 1 — Never use real time to prove ordinary async behavior

Avoid:

```js
await sleep(500);
assert.equal(done, true);
```

unless the test is explicitly testing real-time behavior.

---

## Rule 2 — Return or await every async operation

If a test starts a Promise and does not await/return it, the test may finish too early.

---

## Rule 3 — Synchronize on events, not elapsed time

Prefer:

```text
wait until operation completes
```

over:

```text
wait 100ms
```

---

## Rule 4 — Control clocks when time is the behavior

For:

```text
TTL
retry backoff
timeout
debounce
throttle
lease
deadline
```

inject or virtualize time.

---

## Rule 5 — Control randomness when randomness affects assertions

Inject:

```js
random: () => 0.5
```

when deterministic behavior is required.

---

## Rule 6 — Make concurrency explicit

If a race matters, construct the race.

Do not rely on CPU timing.

---

## Rule 7 — Avoid changing production semantics solely for the test

A test hook should make behavior controllable without creating a fake architecture that production never uses.

---

## Rule 8 — Test completion and cleanup separately

A Promise completing does not mean:

```text
timers closed
listeners removed
sockets closed
workers stopped
```

---

## Rule 9 — Async tests need failure boundaries

Know whether failure is:

```text
assertion
rejection
timeout
resource leak
race
environment
```

---

## Rule 10 — Keep timelines reconstructable

When a test fails, you should be able to answer:

```text
what started
what completed
what was cancelled
what retried
what waited
what remained open
```

---

# 7. Syntax

## 7.1 Promise-returning test

```js
test("loads user", async () => {
  const user = await loadUser("u1");

  assert.equal(user.id, "u1");
});
```

Node's built-in test runner supports asynchronous tests that return Promises; a rejected Promise causes the test to fail. citeturn661777search3

---

## 7.2 Test timeout

```js
test("completes promptly", {
  timeout: 1_000,
}, async () => {
  await operation();
});
```

Current Node test-runner documentation supports per-test and subtest `timeout` options. citeturn661777search3

---

## 7.3 Test cancellation

```js
const controller = new AbortController();

const promise = operation({
  signal: controller.signal,
});

controller.abort();

await assert.rejects(promise);
```

---

## 7.4 AsyncLocalStorage

Node's current documentation marks `AsyncLocalStorage` stable and describes it as preserving context across asynchronous operations; this is useful for correlation IDs and request context in tests. citeturn661777search1

```js
const storage = new AsyncLocalStorage();

await storage.run(
  { requestId: "req-1" },
  async () => {
    await operation();

    assert.equal(
      storage.getStore().requestId,
      "req-1",
    );
  },
);
```

---

# 8. Basic Examples

## Example 1 — Correct async assertion

```js
test("returns value asynchronously", async () => {
  const result = await Promise.resolve(42);

  assert.equal(result, 42);
});
```

---

## Example 2 — Correct rejection assertion

```js
test("rejects invalid input", async () => {
  await assert.rejects(
    () => parseAsync(null),
    {
      code: "INVALID_INPUT",
    },
  );
});
```

---

## Example 3 — Deterministic dependency

```js
const fakeClock = {
  now() {
    return 1_700_000_000_000;
  },
};
```

The application can now test time-sensitive logic without depending on the actual wall clock.

---

# 9. Execution Walkthrough

Suppose:

```js
async function retryingOperation() {
  await delay(100);
  throw new Error("temporary");
}
```

A weak test:

```js
await operation();
```

does not control:

```text
delay
```

A deterministic architecture:

```text
operation
  ↓
clock/scheduler abstraction
  ↓
test scheduler
```

Test:

```text
start operation
  ↓
observe first attempt
  ↓
advance virtual time by 100ms
  ↓
observe second attempt
```

The exact scheduler mechanism depends on the code and test tooling.

---

# 10. Internal Mechanics

## 10.1 Promise jobs

When a Promise settles, reactions are scheduled according to ECMAScript Promise/job semantics.

Example:

```js
Promise.resolve().then(() => {
  console.log("microtask");
});

console.log("sync");
```

Typical output:

```text
sync
microtask
```

Tests that depend on microtask completion must explicitly wait for the Promise chain.

---

## 10.2 Timers

Timers are host/runtime behavior.

```js
setTimeout(() => {
  console.log("timer");
}, 0);
```

The callback does not execute synchronously.

A timer test should not use:

```js
sleep(10);
```

to “hope” the callback fired.

---

## 10.3 Node test concurrency

The current Node test runner supports test-file concurrency and test-level/subtest concurrency. Test-file concurrency is separate from async subtest concurrency, and async concurrency remains managed through Node's event loop within the relevant execution context. citeturn661777search3turn661777search2

This matters because:

```text
parallel test files
≠
parallel async operations
```

---

## 10.4 Test isolation

Current Node test-runner documentation describes process-level isolation in which matching test files can execute in separate child processes; it also documents non-isolated operation where files share a context and can interact with one another. citeturn661777search3

Do not assume:

```text
all tests always have isolated globals
```

or:

```text
all tests always share one process
```

Verify the selected runner configuration.

---

# 11. ECMAScript / Specification Semantics

Async tests operate across multiple layers:

```text
ECMAScript
  ↓
Promise jobs
  ↓
Node runtime
  ↓
timers / event loop
  ↓
test runner
  ↓
application code
```

ECMAScript defines Promise and job semantics.

Node provides:

```text
timers
I/O
AbortSignal
AsyncLocalStorage
process events
```

The test runner provides:

```text
test lifecycle
timeouts
concurrency
isolation
mocking
reporting
```

Therefore do not attribute Node test behavior to the language specification.

---

# 12. Advanced Behavior

## 12.1 Microtask versus timer ordering

Consider:

```js
setTimeout(() => {
  console.log("timer");
}, 0);

Promise.resolve().then(() => {
  console.log("promise");
});

console.log("sync");
```

Typical Node/browser ordering:

```text
sync
promise
timer
```

The precise event-loop model differs by host, but Promise reactions do not execute synchronously before the current stack completes.

Testing should assert the behavior relevant to the supported environment rather than importing browser/event-loop assumptions into Node tests.

---

## 12.2 `await` as a scheduling boundary

```js
async function example() {
  console.log("A");

  await Promise.resolve();

  console.log("B");
}

example();

console.log("C");
```

Typical output:

```text
A
C
B
```

The synchronous prefix runs immediately; continuation occurs later through Promise scheduling.

This is essential when testing code that mutates state before and after an `await`.

---

## 12.3 Concurrent Promise start

Compare:

```js
await a();
await b();
```

with:

```js
const aPromise = a();
const bPromise = b();

await Promise.all([
  aPromise,
  bPromise,
]);
```

Tests should verify concurrency only when concurrency is actually required.

A test that asserts ordering between truly independent operations can accidentally encode an implementation detail.

---

## 12.4 Barriers

A barrier deliberately pauses workers until a condition is satisfied.

```js
function createBarrier(count) {
  let arrived = 0;
  let release;

  const promise = new Promise(resolve => {
    release = resolve;
  });

  return {
    arrive() {
      arrived += 1;

      if (arrived === count) {
        release();
      }

      return promise;
    },
  };
}
```

Test:

```js
const barrier = createBarrier(2);

const first = workerA(barrier);
const second = workerB(barrier);

await Promise.all([
  first,
  second,
]);
```

The barrier creates an intentional synchronization point.

---

## 12.5 Latches

A latch waits for one event:

```text
not released
    ↓
event occurs
    ↓
released
```

Useful for:

```text
server ready
worker started
request received
dependency call started
```

---

# 13. Edge Cases

## 13.1 Promise created but never awaited

```js
test("bad", () => {
  doAsyncWork();
});
```

Potential result:

```text
test passes
async failure occurs later
```

This is one of the most important async-testing mistakes.

---

## 13.2 Promise rejection after test completion

```js
test("bad", () => {
  Promise.reject(new Error("boom"));
});
```

The test does not establish ownership of the Promise lifecycle.

---

## 13.3 Nested async callback

```js
test("bad", async () => {
  setTimeout(async () => {
    await operation();
    assert.equal(...);
  }, 0);
});
```

The outer test can complete before the timer callback.

---

## 13.4 Real network calls

A test:

```js
await fetch(realUrl);
```

may fail because of:

```text
DNS
internet
server availability
rate limits
network latency
```

Use dedicated integration/contract environments for real dependency tests.

---

## 13.5 Date boundaries

Time-based tests can fail around:

```text
midnight
month-end
DST transitions
leap days
timezone conversions
```

Use an injected clock and explicit timezone assumptions.

---

## 13.6 Abort after completion

```js
const result = await operation();

controller.abort();
```

The abort may have no effect because the operation already completed.

Tests should define expected semantics.

---

# 14. Common Misconceptions

### “`await` means no concurrency.”

No. Async operations can be started before awaiting them.

### “`Promise.all()` cancels remaining operations when one rejects.”

No. It rejects the aggregate Promise; started operations are not automatically cancelled.

### “Zero-millisecond timers run immediately.”

No. A timer callback is scheduled for later execution.

### “Sleeping makes async tests reliable.”

It makes them slower and can remain unreliable.

### “Every race can be reproduced with `setTimeout()`.”

No. Scheduling-based races can disappear or change under different environments.

### “AsyncLocalStorage is a general replacement for all async_hooks APIs.”

Node currently recommends `AsyncLocalStorage` for context-tracking use cases and discourages direct use of lower-level async hooks APIs for such purposes. citeturn661777search0turn661777search1

### “Test timeout means application timeout.”

No. A test timeout bounds test execution; application timeout behavior must be tested separately.

### “A faster CI machine fixes flaky tests.”

Not reliably. Faster execution can actually change scheduling and reveal different races.

---

# 15. Common Mistakes

## Mistake 1 — Hidden sleeps

```js
await sleep(100);
```

---

## Mistake 2 — Missing `await`

```js
assert.rejects(...);
```

without:

```js
await
```

---

## Mistake 3 — Shared async state

```js
let result;

test("A", async () => {
  result = await operationA();
});

test("B", async () => {
  assert.equal(result, ...);
});
```

---

## Mistake 4 — Unbounded concurrent tests

Many tests write to the same:

```text
database rows
filesystem
port
global cache
```

---

## Mistake 5 — Mocking away timing

A test replaces every asynchronous boundary with synchronous code and then claims to test asynchronous behavior.

---

## Mistake 6 — Testing with real time

Retry test waits:

```text
100ms
200ms
400ms
800ms
```

every execution.

---

## Mistake 7 — Ignoring cleanup

Timer, socket, worker, or DB connection remains active.

---

## Mistake 8 — Testing an accidental scheduling detail

A test requires:

```text
callback A always executes before B
```

even though the application contract does not guarantee this.

---

# 16. Comparison With Related Concepts

| Technique | Main Purpose | Main Risk |
|---|---|---|
| Real time | Tests actual timing | Slow/flaky |
| Fake/virtual time | Controls timing | Can diverge from runtime behavior |
| Await completion | Establishes lifecycle | Does not control scheduler |
| Barrier | Coordinates concurrency | Can over-constrain implementation |
| Latch | Waits for explicit event | Requires clean release |
| Polling | Wait for condition | May be slow/flaky |
| Event signal | Wait for exact occurrence | Requires observable event |
| Fake dependency | Deterministic behavior | Can diverge from real dependency |
| Integration test | Real boundary | Slower |
| Fault injection | Controlled failure | More infrastructure |
| Test timeout | Prevents infinite tests | Not a substitute for app timeout |
| AsyncLocalStorage | Context tracking | Context bugs / lifecycle complexity |

---


---

# 16A. Explicit Test Timelines

Before writing an asynchronous test, write the expected timeline.

Example:

```text
t0   operation starts
t1   dependency request starts
t2   dependency rejects
t3   retry delay begins
t4   retry delay expires
t5   second attempt starts
t6   second attempt succeeds
t7   operation resolves
```

Then identify which timestamps are:

```text
observable
controllable
implementation-specific
contractual
```

A good test should generally assert contractual events rather than incidental timestamps.

---

## Timeline table

For a retry test:

| Time | Event | Controlled? | Asserted? |
|---|---|---:|---:|
| 0 | attempt 1 starts | yes | yes |
| 0 | attempt 1 fails | yes | yes |
| 100 | backoff expires | yes | yes |
| 100 | attempt 2 starts | yes | yes |
| 100 | attempt 2 succeeds | yes | yes |
| 100 | operation resolves | yes | yes |

This is substantially stronger than:

```js
await sleep(150);
assert.equal(attempts, 2);
```

---

# 16B. Deferred Promises

A deferred Promise is useful when a test needs to control completion.

```js
function createDeferred() {
  let resolve;
  let reject;

  const promise = new Promise((resolvePromise, rejectPromise) => {
    resolve = resolvePromise;
    reject = rejectPromise;
  });

  return {
    promise,
    resolve,
    reject,
  };
}
```

Example:

```js
test("waits for dependency", async () => {
  const deferred = createDeferred();

  const operation = loadUser({
    fetchUser: () => deferred.promise,
  });

  // Operation has started but cannot finish yet.
  deferred.resolve({
    id: "u1",
  });

  const result = await operation;

  assert.equal(result.id, "u1");
});
```

A deferred Promise is a coordination tool.

Do not expose it in production APIs merely because tests need it.

---

# 16C. Barriers for Race Construction

Suppose two operations must both reach the same point before either continues.

```js
function createBarrier(participants) {
  let arrived = 0;
  let release;

  const released = new Promise(resolve => {
    release = resolve;
  });

  return {
    async arrive() {
      arrived += 1;

      if (arrived === participants) {
        release();
      }

      await released;
    },
  };
}
```

Use:

```js
const barrier = createBarrier(2);

async function operationA() {
  const value = await readStock();

  await barrier.arrive();

  return writeStock(value - 1);
}

async function operationB() {
  const value = await readStock();

  await barrier.arrive();

  return writeStock(value - 1);
}
```

This constructs a specific interleaving.

---

## Why this is powerful

Without a barrier:

```text
race occurs sometimes
```

With a barrier:

```text
read A
read B
release both
write A
write B
```

The test controls the race.

---

# 16D. Testing Order Without Over-Specifying Order

Sometimes order is the contract:

```text
authentication
→ authorization
→ business operation
```

Sometimes order is not.

For independent tasks:

```js
await Promise.all([
  a(),
  b(),
]);
```

do not write:

```js
assert.deepEqual(events, [
  "a",
  "b",
]);
```

unless the order is explicitly required.

Better:

```js
assert.deepEqual(
  new Set(events),
  new Set(["a", "b"]),
);
```

The exact assertion depends on the behavior being specified.

---

# 16E. Testing Concurrency Limits

Suppose a service should process at most three tasks concurrently.

Record:

```js
let active = 0;
let peak = 0;

async function work() {
  active += 1;
  peak = Math.max(peak, active);

  await dependency();

  active -= 1;
}
```

Test:

```js
await runWithLimit(tasks, 3);

assert.equal(peak <= 3, true);
```

A stronger test can also assert:

```text
all tasks completed
no task was lost
no task executed twice
```

Performance and correctness both matter.

---

# 16F. Testing Backpressure

For a stream or queue, construct:

```text
producer faster than consumer
```

Example:

```text
producer = 100 items/sec
consumer = 10 items/sec
```

Test that the system:

```text
does not allocate unbounded memory
respects queue capacity
applies backpressure/rejection
preserves required ordering
reports overload
```

The test should inspect an observable contract such as:

```text
max buffer size
rejected count
processing count
```

rather than only waiting for “everything to finish.”

---

# 16G. Testing Cancellation as a State Machine

A cancellation-aware operation can be viewed as:

```text
created
  ↓
running
  ├── completed
  ├── failed
  └── cancelled
```

Test every meaningful transition:

```text
created → cancelled
running → cancelled
running → completed
running → failed
completed → cancelled
failed → cancelled
```

Not every implementation should expose these states publicly.

The test should map them to the actual contract.

---

# 16H. Cancellation Races

Consider:

```text
operation completes
abort signal fires
```

There are two events:

```text
complete
abort
```

The test should determine which outcome the API promises.

Potential contracts:

```text
first terminal event wins
completion wins after completion
abort causes rejection if not yet complete
```

Do not infer semantics from what feels intuitive.

Document them.

---

# 16I. Testing Retry State

A retry system can be modeled:

```text
attempt 1
  ↓
failure classification
  ↓
retry decision
  ↓
backoff
  ↓
attempt 2
```

The test should verify:

```text
attempt count
retryable classification
non-retryable classification
delay calculation
maximum attempts
deadline
abort
final error
```

Do not merely assert:

```js
assert.equal(attempts, 3);
```

Also verify that an error that should not be retried is not retried.

---

# 16J. Testing Jitter

Random jitter creates a tension:

```text
production behavior = intentionally variable
test = should be deterministic
```

Inject randomness:

```js
createRetry({
  random: () => 0.5,
});
```

Now:

```text
same input
+
same random source
=
same schedule
```

Then separately test the random range/property:

```text
delay >= minimum
delay <= maximum
```

Do not test one random sample as proof of a distribution.

---

# 16K. Testing Deadlines

A deadline is different from a timeout.

Timeout:

```text
operation may wait at most N milliseconds
```

Deadline:

```text
operation must finish before absolute time D
```

Example:

```js
function remaining(deadline, now) {
  return Math.max(0, deadline - now);
}
```

Test:

```text
remaining positive
remaining zero
remaining negative
child receives reduced budget
```

For nested calls:

```text
parent deadline = 2,000 ms
child starts after 1,400 ms
remaining = 600 ms
```

Do not give the child another full:

```text
2,000 ms
```

---

# 16L. Testing Timeouts

A timeout test should identify:

```text
what expires
what is aborted
what error is returned
what cleanup occurs
whether underlying resources are released
```

Example:

```js
test("aborts slow request", async () => {
  const controller = new AbortController();

  const promise = request({
    signal: controller.signal,
  });

  controller.abort();

  await assert.rejects(promise);
});
```

Also verify:

```text
no hanging timer
no open connection
no duplicate callback
```

---

# 16M. Testing Async Iterators

Example production behavior:

```js
for await (const item of source) {
  consume(item);
}
```

Test:

```text
first item
next item
end
throw
return/cleanup
backpressure
cancellation
```

Example fake source:

```js
async function* source() {
  yield 1;
  yield 2;
}
```

Test:

```js
const values = [];

for await (const value of source()) {
  values.push(value);
}

assert.deepEqual(values, [1, 2]);
```

For streaming systems, also test early termination.

---

# 16N. Testing Stream Cleanup

A consumer may stop early:

```js
for await (const chunk of stream) {
  if (shouldStop(chunk)) {
    break;
  }
}
```

Verify that:

```text
producer closes
file descriptor closes
socket releases
listener is removed
buffer does not continue growing
```

A test that only verifies the first chunk can miss a resource leak.

---

# 16O. Testing Worker Threads

When using worker threads, test:

```text
worker starts
message sent
result received
worker error
worker exit
termination
serialization
cancellation
```

Do not assume:

```text
terminate()
```

means the current task completed safely.

Treat termination as a lifecycle event requiring explicit semantics.

---

# 16P. Testing Process Signals

Production lifecycle:

```text
SIGTERM
→ stop intake
→ drain
→ cleanup
→ exit
```

Test:

```text
signal arrives before startup completes
signal arrives during active requests
signal arrives during background job
signal arrives while dependency is slow
signal arrives twice
```

Ensure cleanup is:

```text
bounded
idempotent
observable
```

---

# 16Q. Testing Async Context

Node's `AsyncLocalStorage` is designed to preserve context through asynchronous operations. citeturn661777search1

Test:

```text
HTTP request
→ Promise
→ timer
→ database adapter
→ worker callback
```

Example:

```js
test("keeps request context", async () => {
  await storage.run(
    { requestId: "req-1" },
    async () => {
      await Promise.resolve();

      assert.equal(
        storage.getStore().requestId,
        "req-1",
      );
    },
  );
});
```

Then test negative boundaries:

```text
outside context
after context exits
separate concurrent context
```

---

# 16R. Testing Concurrent Async Contexts

Two requests:

```text
request A → context A
request B → context B
```

must not cross-contaminate.

Example:

```js
const results = await Promise.all([
  storage.run("A", async () => {
    await delay();
    return storage.getStore();
  }),

  storage.run("B", async () => {
    await delay();
    return storage.getStore();
  }),
]);

assert.deepEqual(results, ["A", "B"]);
```

This verifies isolation under concurrency.

---

# 16S. Test Runner Concurrency Versus Application Concurrency

These are separate dimensions:

```text
test runner
    ↓
runs tests concurrently

application
    ↓
runs operations concurrently
```

A test can execute sequentially while intentionally creating concurrent application operations.

Conversely, tests can execute in parallel while each test itself is sequential.

Node's current test runner supports test concurrency and test-file concurrency controls; current documentation also explains that process-level isolation changes how files can interact with one another. citeturn661777search2turn661777search3

---

# 16T. Test Resource Allocation

When tests run concurrently, avoid:

```text
port 3000
database schema = test
file = output.json
```

for every test.

Allocate resources uniquely:

```text
worker 1 → port 3101
worker 2 → port 3102
worker 3 → port 3103
```

Node's test-runner documentation provides worker identity/context mechanisms that can be used to partition resources across concurrent test execution. citeturn661777search3

---

# 16U. Flaky Test Classification

When a test fails intermittently, classify before fixing:

```text
timing flake
race condition
shared state
external dependency
randomness
clock/timezone
resource exhaustion
test runner concurrency
environment
cleanup
```

Do not immediately add:

```js
retryTest();
```

A test retry can hide a production-quality problem in the test itself.

---

# 16V. Flake Reproduction

Useful approaches:

```text
repeat the test
run serially
run concurrently
change machine load
seed randomness
capture timeline
enable diagnostics
minimize fixture
```

Example:

```bash
node --test --test-name-pattern="race"
```

Then repeatedly execute in a controlled environment.

The current Node CLI also supports rerunning failed tests; use this as an investigation aid, not as a substitute for fixing nondeterminism. citeturn661777search2

---

# 16W. Deterministic Fault Injection

Instead of waiting for a real outage:

```js
const dependency = {
  async call() {
    throw new TimeoutError();
  },
};
```

Then test:

```text
timeout
retry
fallback
error mapping
metrics
cleanup
```

Fault injection should be explicit and localized.

---

# 16X. Deterministic Network Simulation

Build a fake transport with controls:

```text
latency
status
headers
body
failure
disconnect
```

Example:

```js
const transport = createFakeTransport();

transport.respondWith({
  status: 503,
});

await assert.rejects(
  client.request(),
);
```

For higher-confidence integration behavior, complement fakes with real local HTTP integration tests.

---

# 16Y. Async Test Logging

A deterministic timeline recorder:

```js
function createRecorder() {
  const events = [];

  return {
    record(event) {
      events.push({
        sequence: events.length,
        ...event,
      });
    },

    events() {
      return [...events];
    },
  };
}
```

Use:

```js
recorder.record({
  type: "attempt.start",
});
```

Then inspect:

```text
sequence 0 → request.start
sequence 1 → dependency.start
sequence 2 → dependency.fail
sequence 3 → retry.start
```

Sequence numbers can be more useful than wall-clock timestamps for deterministic tests.

---

# 16Z. Choosing Sequence Numbers Versus Time

Use sequence numbers when testing:

```text
causal order
state transitions
callback invocation order
```

Use controlled time when testing:

```text
TTL
deadline
retry delay
timeout
rate window
```

Use real time only when real timing itself is the feature under test.

---

# 17A. Async Test Design Checklist

Before writing:

```text
What completes the operation?
What can fail?
What can be cancelled?
What can be duplicated?
What depends on time?
What depends on randomness?
What can run concurrently?
What resource must be cleaned?
What ordering is contractual?
What ordering is incidental?
What should be controlled?
What should be observed?
What should remain realistic?
```

Then select:

```text
await
deferred
barrier
latch
fake clock
scheduler
dependency fake
integration environment
stress test
```

Use the least artificial technique that gives sufficient evidence.

---

# 17B. Realism Ladder

Use progressively realistic tests:

```text
Level 1
pure deterministic unit

Level 2
deterministic fake dependencies

Level 3
real local adapter

Level 4
real integration environment

Level 5
load/stress test

Level 6
production telemetry / controlled fault injection
```

No single level is enough for every async system.

---

# 17C. When Not to Virtualize Time

Do not virtualize time when testing:

```text
actual OS scheduling
real timer accuracy
socket timeout implementation
network latency
production event-loop behavior
```

Those are integration/system behaviors.

Instead:

```text
unit tests → virtual time
integration tests → real runtime
performance tests → real environment
```

---

# 17D. Determinism Does Not Mean Serial Execution

A test can be deterministic and concurrent.

Example:

```text
start A
start B
barrier
release A/B
observe results
```

The concurrency is real.

Only the ordering that matters is controlled.

This is stronger than replacing concurrent code with sequential fake calls.

---

# 17E. Async Test Smells

Watch for:

```text
sleep(100)
sleep(1000)
retry test on failure
random port
random assertion timing
global promises
shared mutable mock
unawaited Promise
fake scheduler everywhere
test depends on machine speed
test depends on test order
test swallows rejection
```

Every smell should trigger investigation.

---

# 17F. Async Test Failure Report

A useful failure report should include:

```text
test name
scenario
seed
virtual time
event timeline
attempt count
state
request ID
trace ID
resource state
cleanup result
environment
```

Example:

```text
FAIL: retries payment after timeout

attempts: 2
virtualTime: 1100ms
events:
  0 payment.start
  1 payment.timeout
  2 retry.scheduled
  3 retry.start

expected: completed
actual: pending

activeResources: 0
```

This dramatically shortens debugging.

---

# 17G. Async Testing Architecture Review

Ask:

```text
Can the important asynchronous operation be awaited?
Can dependencies be controlled?
Can time be controlled?
Can cancellation be triggered?
Can concurrency be constructed?
Can cleanup be observed?
Can failure be injected?
Can the test run alone?
Can it run in parallel?
Can it run repeatedly?
Can its failure explain itself?
```

If several answers are “no,” the architecture may not be sufficiently testable.


# 17. Performance Considerations

Async test suites can become extremely slow.

Common causes:

```text
real sleeps
real network
real database setup per test
serial execution
repeated process startup
large fixture creation
```

---

## 17.1 Virtual time

A retry test that takes:

```text
1.5 seconds
```

in real time can often be reduced to:

```text
milliseconds
```

if the application exposes controllable scheduling.

---

## 17.2 Test concurrency

Node's test runner supports concurrent test execution and CLI controls for test-file concurrency. Current documentation also exposes `--test-concurrency`, and test isolation affects how concurrent files share process state. citeturn661777search2turn661777search3

Parallelize only after identifying shared resources.

---

## 17.3 Avoid fake optimization

Do not replace a real integration test with a fake merely to make the suite fast.

Instead create:

```text
fast unit suite
+
focused integration suite
```

---

# 18. Memory Considerations

Async tests can retain:

```text
Promises
closures
timers
event listeners
buffers
streams
request contexts
```

A pending Promise may retain references needed by its continuation.

---

## 18.1 Timer leaks

A repeated timer:

```js
setInterval(...)
```

can keep:

```text
process
test context
closures
```

alive.

Always clean up or explicitly unref where appropriate and semantically safe.

---

## 18.2 Event listeners

Repeated tests can accumulate:

```js
emitter.on("data", handler);
```

without:

```js
emitter.off("data", handler);
```

Leading to:

```text
duplicate callbacks
memory retention
cross-test interference
```

---

# 19. Security Considerations

Async test infrastructure can accidentally bypass production security.

Examples:

```text
mock authorization
mock tenant context incorrectly
disable TLS globally
share test credentials
use production-like sensitive data
```

---

## 19.1 Authorization race tests

Test:

```text
permission revoked
request in flight
```

and decide what contract the application provides.

---

## 19.2 Tenant context

Test that asynchronous work cannot accidentally inherit the wrong:

```text
tenant
actor
request
```

AsyncLocalStorage can carry request context, but the application must still establish and validate the context correctly. Node documents `AsyncLocalStorage.run()` as creating a context available to asynchronous operations created within the callback. citeturn661777search1

---

# 20. Production Usage

## 20.1 Deterministic async test architecture

```text
            Test
             │
     ┌───────┼────────┐
     ↓       ↓        ↓
   Clock  Scheduler  RNG
     │       │        │
     └───────┼────────┘
             ↓
        Application
             │
     ┌───────┼────────┐
     ↓       ↓        ↓
   HTTP      DB     Queue
     │       │        │
    Fake   Test DB   Fake/Emulator
```

Not every test needs all these controls.

Use the smallest set required.

---

## 20.2 Dependency seams

A good asynchronous component often accepts:

```js
createWorker({
  clock,
  scheduler,
  queue,
  logger,
});
```

The production composition root supplies real implementations.

Tests provide deterministic implementations.

---

## 20.3 Deterministic scheduler

Conceptually:

```js
function createScheduler() {
  let now = 0;
  const tasks = [];

  return {
    setTimeout(callback, delay) {
      const task = {
        at: now + delay,
        callback,
      };

      tasks.push(task);

      return task;
    },

    advance(ms) {
      now += ms;

      tasks
        .filter(task => task.at <= now)
        .sort((a, b) => a.at - b.at)
        .forEach(task => task.callback());
    },
  };
}
```

A production-grade scheduler must handle:

```text
cancellation
nested timers
intervals
ordering
microtasks
errors
reentrancy
```

Do not build one casually when mature test tooling already solves the problem.

---

## 20.4 Retries

Production code:

```js
await retry(
  operation,
  {
    scheduler,
    maxAttempts: 3,
  },
);
```

Test:

```text
operation fails
→ scheduler advances
→ retry occurs
→ operation fails
→ scheduler advances
→ final attempt succeeds
```

No real waiting.

---

## 20.5 Timeouts

Test:

```text
start operation
→ advance time to deadline
→ expect timeout
```

Then separately test:

```text
operation succeeds before deadline
```

---

## 20.6 Cancellation

Test both:

```text
abort before start
abort during operation
abort after completion
```

and verify:

```text
resource cleanup
no duplicate callback
correct error/reason
```

---

## 20.7 Queue worker

A deterministic queue test:

```text
enqueue job
  ↓
worker receives job
  ↓
dependency fails
  ↓
job becomes retryable
  ↓
advance scheduler
  ↓
worker retries
```

Test:

```text
attempt count
backoff
state transition
acknowledgement
dead-letter
```

---

## 20.8 Node AsyncLocalStorage

A test can ensure context follows async boundaries:

```js
test("preserves request context", async () => {
  await storage.run(
    { requestId: "req-123" },
    async () => {
      await Promise.resolve();

      assert.equal(
        storage.getStore().requestId,
        "req-123",
      );
    },
  );
});
```

Node's current documentation states that `AsyncLocalStorage` stores remain coherent across asynchronous operations and recommends this stable API for context tracking rather than direct lower-level async hooks usage. citeturn661777search1turn661777search0

---

# 21. Implementation From Scratch

Build a deterministic async testing toolkit.

## Stage 1 — Guided

Implement:

```text
manual clock
event latch
barrier
deferred Promise
```

### Deferred Promise

```js
function createDeferred() {
  let resolve;
  let reject;

  const promise = new Promise(
    (res, rej) => {
      resolve = res;
      reject = rej;
    },
  );

  return {
    promise,
    resolve,
    reject,
  };
}
```

Use:

```js
const deferred = createDeferred();

const promise = worker({
  dependency: {
    load: () => deferred.promise,
  },
});

deferred.resolve("value");

await promise;
```

---

## Stage 2 — Partially Guided

Build a fake clock:

```text
now()
setTimeout()
clearTimeout()
advance()
```

Requirements:

```text
deterministic
stable ordering
cancellation
nested timers
```

---

## Stage 3 — No Reference

Build a retry engine:

```js
createRetry({
  operation,
  attempts,
  scheduler,
  shouldRetry,
});
```

Then create deterministic tests for:

```text
success first attempt
success after retries
permanent failure
timeout
cancellation
```

---

## Stage 4 — Edge-case hardened

Test:

```text
attempt throws synchronously
attempt rejects
attempt resolves
abort during backoff
abort during operation
timer cancelled
multiple concurrent retries
deadline exhausted
```

---

## Stage 5 — Production-grade

Build:

```text
deterministic scheduler
latches/barriers
async context helper
fault injection
timeline recorder
```

Timeline:

```js
[
  { t: 0, event: "attempt.start" },
  { t: 0, event: "attempt.fail" },
  { t: 100, event: "retry.start" },
]
```

The timeline becomes a debugging artifact.

---

## Stage 6 — Complete async workflow

Build:

```text
POST /orders
→ database
→ queue
→ payment worker
→ retry
→ timeout
→ cancellation
→ completion
```

Write deterministic tests for:

```text
success
failure
retry
duplicate delivery
cancellation
shutdown
```

---

# 22. Debugging Exercises

## Exercise 1 — Test passes too early

```js
test("loads data", () => {
  loadData().then(value => {
    assert.equal(value, 42);
  });
});
```

Find why the test is broken.

---

## Exercise 2 — Hidden timer

A test waits:

```js
await sleep(100);
```

but fails on slow CI.

Replace elapsed-time waiting with explicit synchronization.

---

## Exercise 3 — Flaky race

Two operations occasionally:

```text
both read stock = 1
both succeed
```

Design a barrier that guarantees both reads occur before either write.

---

## Exercise 4 — Retry test takes 7 seconds

Backoff:

```text
1s
2s
4s
```

Replace real waiting with controlled scheduling.

---

## Exercise 5 — Async context disappears

A worker logs:

```text
requestId = undefined
```

after queue processing.

Find the missing context propagation boundary.

---

## Exercise 6 — Test hangs

Node process stays alive after tests.

Find:

```text
interval
socket
worker
database pool
event listener
```

---

## Exercise 7 — Parallel test collision

Two tests use:

```text
port 3000
```

and fail intermittently.

Design unique resource allocation.

---

## Exercise 8 — Cancellation race

The operation completes and then abort fires.

Determine expected behavior.

---

# 23. Code Review Exercise

Review:

```js
test("retries payment", async () => {
  let attempts = 0;

  const resultPromise = retry(
    async () => {
      attempts += 1;

      if (attempts < 3) {
        throw new Error("temporary");
      }

      return "ok";
    },
    {
      delay: 1000,
    },
  );

  await new Promise(resolve =>
    setTimeout(resolve, 2500),
  );

  assert.equal(
    await resultPromise,
    "ok",
  );
});
```

Identify at least 20 issues.

Consider:

```text
real-time waiting
implicit scheduling
slow execution
timing assumptions
race possibility
no deadline model
no cancellation test
no retry classification
no backoff inspection
no attempt timeline
no scheduler injection
```

Rewrite the test using a deterministic scheduler.

---

# 24. Interview Questions

## Fundamental

1. Why is async testing difficult?
2. Async versus concurrency?
3. Why is sleeping a bad test strategy?
4. How do you test Promise rejection?
5. How do you test timeout behavior?
6. How do you test cancellation?
7. What is a barrier?
8. What is a latch?
9. Why must async tests await completion?
10. What causes flaky async tests?

## Intermediate

11. Microtask versus timer?
12. How does `await` change scheduling?
13. How do you test retries without waiting?
14. How do you test race conditions?
15. What is a fake clock?
16. What is a fake scheduler?
17. How do you test an event emitter?
18. How do you test streams?
19. How do you test queue consumers?
20. How do you test graceful shutdown?

## Advanced

21. How do you make concurrency deterministic?
22. How do you test cancellation races?
23. How do you control randomness and time?
24. How do you test async context propagation?
25. How do you diagnose a flaky async test?
26. How do you test worker threads?
27. How do you test process signals?
28. How do you test backpressure?
29. How do you test retry storms?
30. How do you test distributed workflows?

## Principal

31. What async behavior should be controlled versus observed?
32. When does a test scheduler become too artificial?
33. How do you avoid encoding implementation scheduling into tests?
34. Which races should be deterministic and which should be stress-tested?
35. How do you combine deterministic and probabilistic concurrency tests?
36. How do you define a test's time model?
37. How do you make async failures diagnosable?
38. How do you prevent test infrastructure from changing production behavior?
39. How do you evaluate whether an async test suite is trustworthy?
40. How would you redesign a test suite with 20% flakiness?

---

# 25. Predict-the-Output Exercises

## Exercise A — `await` boundary

Predict:

```js
async function run() {
  console.log("A");

  await Promise.resolve();

  console.log("B");
}

run();

console.log("C");
```

### Actual Result

```text
A
C
B
```

### Trace

```text
run()
→ logs A
→ reaches await
→ returns Promise

current synchronous code continues
→ logs C

Promise continuation runs
→ logs B
```

### Rule

`await` pauses the async function's continuation; it does not block the JavaScript thread synchronously.

---

## Exercise B — Promise versus timer

Predict:

```js
setTimeout(() => {
  console.log("timer");
}, 0);

Promise.resolve().then(() => {
  console.log("promise");
});

console.log("sync");
```

### Typical Result

```text
sync
promise
timer
```

### Important Discipline

The exact host scheduling model matters for task/macrotask behavior. Do not generalize one host's event-loop details as ECMAScript language guarantees.

---

## Exercise C — Parallel start

Predict:

```js
let active = 0;
let peak = 0;

async function work() {
  active += 1;
  peak = Math.max(peak, active);

  await Promise.resolve();

  active -= 1;
}

const p1 = work();
const p2 = work();
const p3 = work();

await Promise.all([p1, p2, p3]);

console.log(peak);
```

### Actual Result

```text
3
```

### Rule

All three operations begin before the first `await Promise.all()`.

---

## Exercise D — Promise.all rejection

Conceptually:

```js
const slow = new Promise(resolve => {
  setTimeout(() => resolve("slow"), 100);
});

const fastFailure = Promise.reject(
  new Error("fail"),
);

await Promise.all([
  slow,
  fastFailure,
]);
```

### Result

The aggregate Promise rejects because one input rejects.

But the already-started `slow` operation is not automatically cancelled.

### Principal Lesson

Aggregate Promise failure is not equivalent to cancellation of constituent operations.

---

## Exercise E — AsyncLocalStorage context

Conceptually:

```js
const storage = new AsyncLocalStorage();

await storage.run("A", async () => {
  await Promise.resolve();

  console.log(storage.getStore());
});
```

### Actual Result

```text
A
```

Node's documentation describes `AsyncLocalStorage` as preserving context across asynchronous operations created within the context. citeturn661777search1

---

# 26. Mastery Exercises

## Track A — Core Theory

### A1

Draw timelines for:

```text
Promise
timer
I/O callback
async function
queue worker
```

and identify:

```text
synchronous region
scheduling boundary
completion signal
cleanup
```

### A2

Explain why:

```text
sleep
```

is weaker than:

```text
event
```

as a test synchronization mechanism.

### A3

Design a taxonomy:

```text
timing bug
race bug
resource leak
missing await
wrong cancellation
test isolation problem
scheduler assumption
```

---

## Track B — Implementation

### B1 — Deterministic clock

Implement:

```text
now
setTimeout
clearTimeout
advance
```

Then test:

```text
debounce
retry
TTL
deadline
```

### B2 — Concurrency harness

Implement:

```text
barrier
latch
deferred
```

Use them to force:

```text
two concurrent reads
one competing write
```

### B3 — Reliable worker tests

Test:

```text
job success
retry
backoff
duplicate delivery
dead-letter
cancellation
shutdown
```

without arbitrary sleeps.

### B4 — Async context

Test:

```text
requestId
tenantId
traceId
```

across:

```text
Promise
timer
HTTP
queue
worker boundary
```

Document which contexts require explicit propagation.

---

## Track C — Interview / Reasoning

### C1

A test suite has:

```text
10% flaky tests
```

most failures are:

```text
timeout
race
open handle
```

Design a remediation plan.

### C2

A concurrency bug occurs once every:

```text
100,000 requests
```

Design a testing strategy combining:

```text
deterministic scheduling
stress testing
fault injection
production observability
```

### C3

A team built a sophisticated fake scheduler that no longer behaves like Node.

Decide whether to:

```text
keep
simplify
remove
move to integration tests
```

Defend the decision.

---

# 27. Key Takeaways

1. Deterministic async testing controls conditions instead of waiting for luck.
2. Async is not identical to concurrency.
3. `await` does not eliminate concurrency.
4. Async tests must own their Promise lifecycle.
5. Real sleeps are usually a poor synchronization mechanism.
6. Synchronize on events, not elapsed time.
7. Virtual time is valuable for time-dependent logic.
8. Deterministic schedulers should be used carefully.
9. Barriers and latches can construct reproducible races.
10. Race tests should control ordering rather than rely on machine timing.
11. `Promise.all()` does not automatically cancel other started operations.
12. Timeout of a test is not the same as application timeout.
13. Cancellation must be tested before, during, and after operations.
14. Async cleanup is part of correctness.
15. Node test-runner concurrency and isolation must be understood before parallelizing suites. citeturn661777search2turn661777search3
16. `AsyncLocalStorage` is the stable Node API for common async-context tracking use cases. citeturn661777search1
17. Direct lower-level `async_hooks` tracking has significant caveats and Node recommends higher-level mechanisms for context tracking. citeturn661777search0
18. Deterministic unit tests and realistic integration/stress tests complement each other.
19. Test infrastructure should not create a fictional version of production.
20. The goal is not to eliminate all nondeterminism; it is to make important behavior reproducible enough to test and diagnose.
21. Principal async testing balances determinism, realism, execution cost, and failure coverage.

---

# 28. Concept Connections

## Depends On

- **31** — Async Fundamentals
- **32** — ECMAScript Jobs / Promise Reactions
- **33** — Browser Event Loop
- **34** — Node Event Loop / libuv
- **35** — Promises
- **36** — Async / Await
- **37** — Cancellation / Abort
- **38** — Async Iteration / Streaming
- **39** — Concurrency / Parallelism
- **45–48** — Memory / GC / Engine / V8
- **52–53** — Workers / Web Streams
- **58–63** — Node.js / Streams / Workers / Lifecycle / Async Context
- **83** — Observability
- **84** — Reliability
- **85** — Performance
- **86** — Testing

## Builds Toward

- **88** — Debugging Methodology
- **89** — Code Review / Refactoring
- **94** — Compatibility Engineering
- **98–101** — Failure Modes / Judgment / Production Scenarios
- **102–111** — Production Projects
- **113–121** — Advanced Assessments / System Design / Principal Project

## Related Concepts

```text
Deterministic Async Testing
  ├─ promises
  ├─ microtasks
  ├─ timers
  ├─ event loop
  ├─ cancellation
  ├─ concurrency
  ├─ synchronization
  ├─ async context
  ├─ streams
  ├─ workers
  ├─ queues
  └─ lifecycle
```

## Concepts Revisited

```text
Promise jobs
event-loop scheduling
AbortSignal
AsyncLocalStorage
resource cleanup
backpressure
retry
timeouts
idempotency
observability
reliability
performance
```

## Why This Chapter Matters Later

Many of the hardest production JavaScript bugs involve:

```text
timing
concurrency
cancellation
retry
resource lifetime
```

Deterministic async testing turns these from “works most of the time” behavior into repeatable engineering evidence.

---

# 29. Completion Criteria

Mark:

```text
[~] In Progress
```

when you understand:

```text
async test lifecycle
Promise completion
timer behavior
basic concurrency
```

Mark:

```text
[?] Needs Revision
```

when you repeatedly confuse:

```text
async vs concurrent
await vs cancellation
timeout vs cancellation
Promise rejection vs cancellation
sleep vs synchronization
test timeout vs application timeout
test isolation vs test concurrency
```

Mark:

```text
[+] Completed
```

when you can:

- write deterministic async tests;
- test Promise rejection;
- control time-dependent behavior;
- test cancellation;
- test retries;
- construct concurrency races;
- test resource cleanup;
- test async context;
- run safe parallel tests.

Mark:

```text
[*] Mastered
```

only when you can:

- diagnose flaky asynchronous tests;
- construct deterministic races;
- choose control versus observation boundaries;
- design async test infrastructure;
- test distributed workflow timing;
- combine deterministic and stress testing;
- preserve production semantics while improving testability;
- defend an async testing strategy at principal level.

Reading alone does not qualify as mastery.

---

# Chapter 87 — Revision / Retrieval Record

| Date | Retrieval Goal | Result | Status |
|---|---|---|---|
| ____ | Explain async test lifecycle | ____ | `[ ]` |
| ____ | Explain Promise scheduling | ____ | `[ ]` |
| ____ | Explain timer scheduling | ____ | `[ ]` |
| ____ | Design deterministic time control | ____ | `[ ]` |
| ____ | Test cancellation | ____ | `[ ]` |
| ____ | Test retry/backoff | ____ | `[ ]` |
| ____ | Construct a race deterministically | ____ | `[ ]` |
| ____ | Test resource cleanup | ____ | `[ ]` |
| ____ | Test async context | ____ | `[ ]` |
| ____ | Test queue/worker behavior | ____ | `[ ]` |
| ____ | Debug a flaky async test | ____ | `[ ]` |
| ____ | Defend async test architecture | ____ | `[ ]` |

## Spaced Retrieval

```text
Review 1 — same day
Review 2 — +1 day
Review 3 — +3 days
Review 4 — +7 days
Review 5 — +14 days
Review 6 — +30 days
Review 7 — +60 days
```

## Retrieval Prompts

Without reading:

1. Why is sleeping weaker than synchronization?
2. Async versus concurrent?
3. What does `await` actually do?
4. Why can a missing await make a test pass incorrectly?
5. How do you test retries without waiting?
6. How do you construct a deterministic race?
7. What is a barrier?
8. What is a latch?
9. Why does `Promise.all()` not cancel remaining operations?
10. How do you test cancellation?
11. How do you test timeouts?
12. What causes async test flakiness?
13. What is AsyncLocalStorage?
14. What is the difference between test timeout and application timeout?

---

# Chapter 87 — Canonical References and Source Discipline

## 1. Node.js Test Runner

The current Node.js test runner documentation describes:

```text
async tests
timeouts
subtests
concurrency
process-level isolation
sharding
worker/test context
```

It documents asynchronous tests as Promise-returning tests and provides `timeout` and `concurrency` options. citeturn661777search3

Primary:

- https://nodejs.org/api/test.html

---

## 2. Node.js Test CLI

The current Node.js CLI documents `--test-concurrency`, which controls the maximum number of test files executed concurrently and interacts with test isolation. citeturn661777search2

Primary:

- https://nodejs.org/api/cli.html

---

## 3. Async Context

Node.js currently documents `AsyncLocalStorage` as stable and recommends it for associating state with asynchronous operations. It supports `run()`, `getStore()`, `bind()`, and `snapshot()` among other APIs. citeturn661777search1

Primary:

- https://nodejs.org/api/async_context.html

---

## 4. Lower-Level Async Hooks

Node's current documentation strongly discourages general use of low-level `async_hooks` lifecycle APIs because of usability, safety, and performance concerns, recommending `AsyncLocalStorage` for context tracking and other higher-level diagnostics mechanisms for other purposes. citeturn661777search0

Primary:

- https://nodejs.org/api/async_hooks.html

---

## 5. ECMAScript

Use the ECMAScript specification for:

```text
Promise behavior
async functions
job/microtask semantics
language-level completion
```

Primary:

- https://tc39.es/ecma262/

Do not use ECMAScript as the authority for:

```text
Node timer phases
Node test-runner behavior
process-level test isolation
Node AsyncLocalStorage
```

Those are runtime facilities.

---

## Source Discipline

Classify every async-testing observation as:

```text
ECMAScript scheduling semantics
Node.js event-loop behavior
Node.js timer behavior
Node test-runner behavior
test-tool behavior
application behavior
environment behavior
```

Be especially careful with statements such as:

> “Promises always run before timers.”

The broader rule is host-dependent scheduling plus ECMAScript job semantics; use the actual runtime documentation and a small executable experiment for the environment being tested.

Likewise:

> “AsyncLocalStorage always propagates everywhere.”

The API is designed for async context propagation, but custom async resources and unusual lifecycle boundaries can require additional care. Node's documentation includes `AsyncResource` for library authors handling their own asynchronous resources. citeturn661777search0turn661777search1

---

# Chapter 87 — Completion Snapshot

## Core Theory

- [ ] Async test lifecycle
- [ ] Async vs concurrency
- [ ] Promise jobs
- [ ] `await` scheduling boundary
- [ ] Timers
- [ ] Event loop
- [ ] Test timeout
- [ ] Application timeout
- [ ] Virtual time
- [ ] Fake clock
- [ ] Scheduler
- [ ] Deferred Promise
- [ ] Latch
- [ ] Barrier
- [ ] Deterministic race
- [ ] Cancellation
- [ ] Retry timing
- [ ] Deadline
- [ ] Async iterators
- [ ] Streams
- [ ] Queue workers
- [ ] Worker threads
- [ ] Async context
- [ ] AsyncLocalStorage
- [ ] Test concurrency
- [ ] Test isolation
- [ ] Resource cleanup
- [ ] Flakiness
- [ ] Fault injection

## Implementation

- [ ] Async Promise test
- [ ] Rejection test
- [ ] Deferred helper
- [ ] Fake clock
- [ ] Deterministic scheduler
- [ ] Barrier
- [ ] Latch
- [ ] Retry test
- [ ] Backoff test
- [ ] Timeout test
- [ ] Cancellation test
- [ ] Race-condition test
- [ ] Queue-worker test
- [ ] Stream test
- [ ] Async iterator test
- [ ] Worker test
- [ ] Shutdown test
- [ ] AsyncLocalStorage test
- [ ] Fault-injection test
- [ ] Timeline recorder
- [ ] Parallel test isolation
- [ ] Flaky-test diagnosis

## Interview / Reasoning

- [ ] Explain async test lifecycle
- [ ] Explain Promise scheduling
- [ ] Explain timer behavior
- [ ] Explain `await`
- [ ] Explain `Promise.all()`
- [ ] Explain deterministic time
- [ ] Explain synchronization
- [ ] Explain race construction
- [ ] Explain cancellation
- [ ] Explain retry testing
- [ ] Explain AsyncLocalStorage
- [ ] Explain Node test isolation
- [ ] Diagnose flaky tests
- [ ] Design async testing strategy
- [ ] Defend test realism

## Mastery Gate

```text
Understand      [ ]
Explain         [ ]
Predict         [ ]
Implement       [ ]
Debug           [ ]
Apply           [ ]
Compare         [ ]
Defend          [ ]
```

## Final Principal Test

Given an asynchronous JavaScript test suite, can you determine:

```text
Does every async operation have an explicit lifecycle owner?
Are any Promises started without being awaited?
Are there hidden sleeps?
Does the suite rely on real time unnecessarily?
Is the clock controllable?
Is scheduling controllable?
Where are concurrency boundaries?
Where are race conditions?
Can important races be reproduced deterministically?
Which behavior is guaranteed by ECMAScript?
Which behavior is Node-specific?
Which behavior is test-runner-specific?
Are test files isolated?
Can tests interfere through shared state?
Can timers leak?
Can sockets leak?
Can workers leak?
Can database connections leak?
Can async context leak?
Are retries deterministic?
Are backoffs deterministic?
Are deadlines deterministic?
Are cancellations tested?
Are duplicate operations tested?
Are partial failures tested?
Are queue retries tested?
Are shutdown races tested?
Are tests overly mocked?
Does the fake scheduler resemble production closely enough?
Which behaviors still require stress testing?
Which failures remain nondeterministic?
How are flaky tests diagnosed?
What evidence would convince you the suite is trustworthy?
```

A principal async testing engineer does not attempt to make every asynchronous system artificially synchronous.

They create **controlled synchronization points where correctness depends on ordering, controlled time where correctness depends on time, and realistic integration/stress tests where production behavior depends on an environment that a unit test should not pretend to reproduce.**

---

## Principal Async Testing Decision Framework

For every asynchronous behavior, record:

```text
Behavior:
Async Boundary:
Concurrency:
Scheduling Dependency:
Time Dependency:
Cancellation:
Retry:
Resource:
External Dependency:
Required Guarantee:
Determinism Strategy:
Synchronization Mechanism:
Test Level:
Production Similarity:
Failure Modes:
Cleanup:
Observability:
Flake Risk:
Execution Cost:
Maintenance Cost:
False-Confidence Risk:
Decision:
Revisit Trigger:
```

Then ask:

> **Are we controlling the uncertainty that matters, or are we merely making the test appear deterministic by replacing the behavior we actually need to validate?**

That question separates reliable async testing from timing-based test folklore.