\
# Chapter 63 — Async Context and Diagnostics

> **Curriculum position:** Part XI — Node.js  
> **Previous chapter:** Chapter 62 — Process Lifecycle  
> **Next chapter:** Chapter 64 — ES Modules  
> **Primary environment:** Modern Node.js; API details are version-sensitive and should be verified against current Node.js documentation.

---

# Chapter Mission

Learn to preserve, inspect, and reason about **causal context across asynchronous execution** in Node.js, and use that context to diagnose production systems.

The central problem is simple to state:

```text
Request A
  ├── database query
  ├── Promise continuation
  ├── timer
  ├── worker/task
  └── outgoing HTTP call
```

By the time the database callback runs, the JavaScript call stack that originally received Request A is gone.

Yet production engineers still need to answer:

> Which request caused this database query?

> Which user operation caused this log?

> Which trace/span does this asynchronous callback belong to?

> Which request created this timeout?

> Which background operation inherited the wrong tenant, transaction, or correlation ID?

Node.js provides **async context propagation** primitives, especially `AsyncLocalStorage`, on top of the lower-level async hooks machinery.

This chapter builds the model from:

```text
ECMAScript execution
      ↓
Node async resources
      ↓
async context propagation
      ↓
diagnostics
      ↓
observability
      ↓
production debugging
```

The goal is not merely to learn an API.

The goal is to understand:

- causal context,
- async resource lifetime,
- context boundaries,
- propagation guarantees,
- context loss,
- diagnostics APIs,
- event-loop and runtime visibility,
- tracing architecture,
- performance trade-offs,
- failure modes,
- and production design.

---

# 1. Learning Objectives

After completing this chapter, you should be able to:

## Core theory

- Explain why ordinary lexical variables do not automatically model request context across arbitrary asynchronous boundaries.
- Explain what an asynchronous resource is conceptually.
- Explain how Node associates asynchronous work with execution context.
- Distinguish call stack, lexical scope, Promise state, async resource, and async context.
- Explain `AsyncLocalStorage`.
- Explain `AsyncResource`.
- Explain the conceptual role of `async_hooks`.
- Explain context propagation versus context storage.
- Explain context loss and restoration.
- Explain why context should be treated as request metadata rather than mutable global state.

## Diagnostics

- Inspect process metadata.
- Inspect active resources.
- Understand diagnostic reports.
- Use runtime metrics appropriately.
- Correlate asynchronous work with requests.
- Build request-scoped logging.
- Diagnose lost context.
- Diagnose incorrect context inheritance.
- Reason about async lifecycle events.

## Production

- Design request correlation.
- Design tenant/request/job context.
- Integrate context with structured logging.
- Integrate context with tracing.
- Avoid context leaks and cross-request contamination.
- Define safe context contents.
- Bound diagnostic overhead.
- Build deterministic diagnostics for shutdown and incident response.

## Interview / reasoning

- Explain how async context differs from a global variable.
- Explain why `AsyncLocalStorage` is not a transaction system.
- Explain why `AsyncLocalStorage` should not be used as an uncontrolled mutable global.
- Explain when `AsyncResource` is appropriate.
- Diagnose a context that disappears in a callback.
- Defend a production context design under principal-level review.

---

# 2. Prerequisites

Strongly recommended:

- Chapter 10 — Scope and Lexical Environments
- Chapter 12 — Execution Contexts and Execution Model
- Chapter 13 — Closures
- Chapter 31 — Async Fundamentals
- Chapter 32 — ECMAScript Jobs and Promise Reactions
- Chapter 34 — Node Event Loop and libuv
- Chapter 35 — Promises
- Chapter 36 — Async/Await
- Chapter 37 — Cancellation and Abort
- Chapter 38 — Async Iteration and Streaming
- Chapter 58 — Node.js Architecture
- Chapter 62 — Process Lifecycle

Especially important:

> **A JavaScript call stack is short-lived. A production request is not.**

---

# 3. What Is It?

## 3.1 Async context

Async context is metadata associated with a chain of asynchronous execution.

For example:

```text
HTTP request
   │
   ├── requestId = "req-123"
   ├── tenantId = "tenant-9"
   └── traceId = "abc..."
          │
          ├── Promise
          ├── timer
          ├── DB query
          ├── outgoing fetch
          └── log statement
```

The desired property is:

```text
all related asynchronous operations
        ↓
see the same logical context
```

without passing the context manually through every function.

---

# 4. Why Does It Exist?

Without explicit async context propagation, code often degenerates into:

```js
doThing(requestId, tenantId, traceId, userId, logger, ...)
```

Then:

```js
doThing(
  requestId,
  tenantId,
  traceId,
  userId,
  logger,
  ...
);
```

and eventually:

```text
function signatures become context transport pipes
```

This causes:

- noisy APIs,
- accidental omission,
- inconsistent correlation,
- difficult tracing,
- poor observability,
- and fragile refactoring.

Async context provides a runtime-supported way to associate state with an asynchronous execution chain.

---

# 5. Mental Model

## 5.1 Five different concepts

Do not collapse these concepts:

| Concept | Meaning |
|---|---|
| Call stack | Current synchronous execution frames |
| Lexical scope | Static identifier visibility |
| Closure | Function + captured lexical environment |
| Promise | Result/continuation abstraction |
| Async context | Runtime-associated logical execution metadata |

Example:

```js
let requestId;
```

is not automatically request-scoped just because a request exists.

This:

```js
const storage = new AsyncLocalStorage();
```

creates the mechanism for storing context that follows supported async execution paths.

---

# 6. The Key Distinction: Context vs Scope

Lexical scope:

```js
function handler(request) {
  const requestId = request.id;

  service();
}
```

`requestId` is visible only where lexical rules allow it.

Async context:

```js
storage.run({ requestId: request.id }, () => {
  service();
});
```

code later executed through supported asynchronous propagation can retrieve:

```js
storage.getStore()
```

The two mechanisms solve different problems.

---

# 7. Basic API

Import:

```js
import { AsyncLocalStorage } from 'node:async_hooks';

const storage = new AsyncLocalStorage();
```

Create a context:

```js
storage.run(
  { requestId: 'req-123' },
  () => {
    console.log(storage.getStore());
  },
);
```

Expected conceptual result:

```js
{ requestId: 'req-123' }
```

Retrieve outside the active context:

```js
storage.getStore();
```

may return:

```js
undefined
```

depending on where execution occurs.

---

# 8. Basic Request Context

```js
import http from 'node:http';
import { AsyncLocalStorage } from 'node:async_hooks';

const asyncLocalStorage = new AsyncLocalStorage();

const server = http.createServer((req, res) => {
  const context = {
    requestId: crypto.randomUUID(),
  };

  asyncLocalStorage.run(context, () => {
    setTimeout(() => {
      console.log(asyncLocalStorage.getStore());
    }, 10);

    res.end('ok');
  });
});

server.listen(3000);
```

The timer callback can observe the context because it is created within the async context.

---

# 9. Execution Walkthrough

Consider:

```js
storage.run({ requestId: 'A' }, () => {
  setTimeout(() => {
    console.log(storage.getStore());
  }, 100);
});
```

Conceptually:

```text
run(context A)
      ↓
establish current async context
      ↓
schedule timer
      ↓
callback returns
      ↓
original call stack disappears
      ↓
timer resource remains
      ↓
timer fires later
      ↓
Node restores associated context
      ↓
callback runs
      ↓
getStore() → context A
```

The important idea is:

> context follows asynchronous execution through host-managed relationships.

---

# 10. Internal Mechanics

## 10.1 Async resources

Node represents different asynchronous operations as resources.

Examples include concepts associated with:

- timers,
- Promises,
- sockets,
- file-system operations,
- TCP handles,
- immediate callbacks,
- DNS operations,
- streams,
- user-defined async resources.

The internal lifecycle can be thought of as:

```text
resource created
      ↓
resource initialized
      ↓
resource executes
      ↓
resource may trigger other async work
      ↓
resource destroyed
```

---

# 11. `async_hooks` Mental Model

Node's async-hooks machinery exposes lifecycle signals for async resources.

Conceptually:

```text
init
before
after
destroy
```

and Promise-related lifecycle concepts may also be surfaced depending on the API.

The callbacks allow tooling to reason about asynchronous relationships.

However:

> low-level async hooks are an advanced runtime diagnostic mechanism, not a default business-logic API.

Use higher-level `AsyncLocalStorage` when the goal is context propagation.

---

# 12. `AsyncLocalStorage`

## 12.1 Purpose

`AsyncLocalStorage` provides a way to maintain state throughout an asynchronous execution context.

Common uses:

- request IDs,
- correlation IDs,
- trace IDs,
- tenant context,
- authenticated principal metadata,
- feature-evaluation metadata,
- job IDs,
- diagnostic flags.

---

# 13. `run()`

Core pattern:

```js
storage.run(store, callback);
```

Example:

```js
storage.run(
  {
    requestId: 'req-1',
  },
  () => {
    performWork();
  },
);
```

Everything created through the supported async execution chain can inherit this context.

---

# 14. `getStore()`

```js
const context = storage.getStore();
```

Typical helper:

```js
function getRequestContext() {
  return asyncLocalStorage.getStore();
}
```

Then:

```js
logger.info({
  requestId: getRequestContext()?.requestId,
}, 'database query');
```

---

# 15. `enterWith()`

Node also exposes:

```js
storage.enterWith(store);
```

This changes the current execution context and can cause subsequent synchronous and asynchronous work in that execution chain to observe the entered store.

It is more dangerous to use casually than `run()` because the context transition is broader and easier to accidentally leak across event-driven control flow.

Prefer a scoped:

```js
storage.run(context, callback);
```

when you have a clear operation boundary.

---

# 16. `disable()`

Node exposes:

```js
storage.disable();
```

Conceptually this disables the storage instance and exits its active contexts.

Use lifecycle ownership carefully.

For a process-wide request context object, repeatedly calling `disable()` as normal request cleanup is generally the wrong mental model.

Instead:

```text
process-wide AsyncLocalStorage
       +
request-scoped store values
```

is a more typical architecture.

---

# 17. Context Immutability

Avoid:

```js
storage.run(
  { requestId: 'req-1' },
  () => {
    const ctx = storage.getStore();
    ctx.user = anotherUser;
  },
);
```

when many layers can mutate the same store.

Prefer treating context as immutable metadata:

```js
storage.run(
  Object.freeze({
    requestId,
    tenantId,
    traceId,
  }),
  handler,
);
```

or use immutable update patterns when a legitimate context transition is required.

### Rule

> Context should be easy to read and hard to accidentally mutate.

---

# 18. Request Logging

A useful logger wrapper:

```js
function log(level, message, fields = {}) {
  const context = asyncLocalStorage.getStore();

  console[level]({
    ...fields,
    requestId: context?.requestId,
    traceId: context?.traceId,
  }, message);
}
```

Now:

```js
log('info', 'query started', {
  sqlOperation: 'findUser',
});
```

can automatically attach correlation.

---

# 19. Structured Logging

Prefer:

```js
logger.info({
  event: 'db.query.start',
  requestId,
  traceId,
  operation: 'findUser',
}, 'Database query started');
```

over:

```js
console.log(
  `request ${requestId} started database query findUser`,
);
```

Structured data is easier to:

- search,
- aggregate,
- parse,
- correlate,
- alert on.

---

# 20. Context Propagation vs Explicit Parameters

## Explicit

```js
await service.doWork({
  requestId,
  traceId,
  tenantId,
});
```

Advantages:

- explicit dependencies,
- easy unit testing,
- obvious data flow.

Disadvantages:

- repetitive,
- context plumbing,
- easy omission.

## Async context

```js
await service.doWork();
```

with context available through:

```js
storage.getStore();
```

Advantages:

- convenient,
- excellent for cross-cutting observability.

Disadvantages:

- hidden dependency,
- harder mental model,
- possible test contamination,
- risk of misuse as global mutable state.

### Principal trade-off

> Use async context for metadata that truly crosses many layers; do not hide essential business inputs just because context storage exists.

---

# 21. Context Should Not Replace Function Arguments Everywhere

Bad:

```js
const userId = asyncLocalStorage.getStore().userId;
```

inside every function that logically needs a specific user ID.

If the function's core business contract requires:

```text
userId
```

then passing:

```js
getUser(userId)
```

can be clearer.

Better async-context candidates are:

```text
requestId
traceId
correlationId
diagnostic flags
request start time
tenant metadata
```

depending on architecture and trust model.

---

# 22. Context Loss

A common production problem:

```js
storage.getStore() === undefined
```

inside a callback where context was expected.

Possible causes include:

- custom callback APIs that do not preserve expected execution relationships,
- manually detached callbacks,
- third-party native integrations,
- incorrect use of `enterWith`,
- work created outside the intended context,
- unusual event emitter patterns,
- context being created too late,
- incompatible abstractions.

When context disappears, do not immediately blame Promises.

First identify:

```text
where context was created
where expected context propagation broke
which async resource introduced the boundary
```

---

# 23. `AsyncResource`

When you build custom asynchronous abstractions, Node provides `AsyncResource` to associate callbacks with a logical async resource.

Conceptually:

```text
custom library operation
        ↓
create AsyncResource
        ↓
run callback in correct async scope
        ↓
AsyncLocalStorage sees expected context
```

A typical custom resource may use:

```js
const resource = new AsyncResource('MY_RESOURCE');

resource.runInAsyncScope(callback, thisArg, ...args);
```

The exact lifecycle management depends on the custom abstraction.

---

# 24. Why `AsyncResource` Exists

Suppose you write a custom task scheduler:

```text
submit(task A)
submit(task B)
...
later execute task A
```

The execution may be logically related to the context that submitted it.

If the scheduler simply stores:

```js
queue.push(callback);
```

and executes it later from another context, the relationship may not be represented correctly for diagnostics/context propagation.

An explicit async resource lets the abstraction communicate:

> this callback execution belongs to this logical asynchronous operation.

---

# 25. When to Use `AsyncResource`

Good candidates:

- custom callback queues,
- worker-pool libraries,
- native-addon integrations,
- framework adapters,
- custom async abstractions.

Bad candidate:

> adding `AsyncResource` everywhere merely because async code exists.

Use it when you own an abstraction that creates a meaningful asynchronous boundary not already modeled by the runtime.

---

# 26. Worker Threads and Context

Worker threads are separate execution contexts.

Do not assume process-wide async context automatically crosses:

```text
main thread
     ↓
worker thread
```

Use explicit messaging:

```js
worker.postMessage({
  requestId,
  traceId,
  jobId,
});
```

Then establish the desired context within the worker.

Conceptually:

```text
main thread context
       ↓
serialized metadata
       ↓
worker message
       ↓
worker creates local context
```

This is a separate-process-like propagation boundary.

---

# 27. Child Processes and Context

Likewise, child processes do not magically inherit application-level `AsyncLocalStorage` state.

Pass what the child needs:

```text
IPC / argv / env / protocol message
```

Then establish context independently.

---

# 28. HTTP Boundaries

At an incoming HTTP boundary:

```text
socket
  ↓
HTTP parser
  ↓
request handler
  ↓
AsyncLocalStorage.run()
```

The best place for request context creation is near the boundary where the logical request identity becomes known.

Example:

```js
const server = http.createServer((req, res) => {
  const context = {
    requestId: req.headers['x-request-id'] ?? crypto.randomUUID(),
  };

  asyncLocalStorage.run(context, () => {
    route(req, res);
  });
});
```

---

# 29. Trust Boundaries for Correlation IDs

Never blindly treat arbitrary incoming metadata as trusted identity.

For example:

```http
x-tenant-id: admin
```

does not prove the caller is an admin.

Separate:

```text
correlation metadata
```

from:

```text
authorization facts
```

A request ID can often be accepted or replaced.

An authenticated principal must be derived from the actual authentication/authorization system.

---

# 30. Trace Context

Observability systems often use:

```text
trace ID
span ID
parent relationship
```

Async context provides a runtime location to store references to the current tracing state.

Conceptually:

```text
HTTP request
  ↓
trace/span context
  ↓
AsyncLocalStorage
  ↓
DB call
  ↓
outbound HTTP
  ↓
logs/metrics
```

The tracing standard and tracing SDK define propagation semantics; `AsyncLocalStorage` is the Node runtime mechanism often used underneath or alongside such systems.

---

# 31. Diagnostics Architecture

A production diagnostic stack may look like:

```text
                     Application
                         │
          ┌──────────────┼──────────────┐
          ▼              ▼              ▼
       Logging        Metrics         Tracing
          │              │              │
          └────────── async context ────┘
                         │
                         ▼
                    Node runtime
                         │
              ┌──────────┼──────────┐
              ▼          ▼          ▼
          event loop   resources   reports
```

Async context is the correlation layer.

It is not the whole observability system.

---

# 32. Process Diagnostics

Useful process information includes:

```js
process.pid;
process.ppid;
process.version;
process.platform;
process.arch;
process.uptime();
process.memoryUsage();
process.resourceUsage();
process.getActiveResourcesInfo();
```

Use these as diagnostic signals.

Do not infer too much from any single API.

---

# 33. Active Resources

A useful investigation tool:

```js
console.log(process.getActiveResourcesInfo());
```

This can help answer:

> What resource types are currently keeping the event loop alive?

It is particularly useful during:

- tests,
- shutdown,
- CLI debugging,
- leak investigations.

It is not a full causal graph of resource ownership.

---

# 34. Diagnostic Reports

Node provides diagnostic reports for deeper runtime inspection.

Conceptually:

```text
process
 ├── JS state
 ├── native/runtime state
 ├── resource information
 ├── stack information
 └── process metadata
```

Reports can be valuable for:

- crashes,
- hung services,
- native/runtime problems,
- production incidents.

Use them carefully because reports can contain sensitive operational information.

---

# 35. Memory Diagnostics

A lifecycle-aware service should correlate context with memory where useful.

For example:

```js
logger.info({
  requestId: asyncLocalStorage.getStore()?.requestId,
  memory: process.memoryUsage(),
}, 'diagnostic snapshot');
```

Avoid dumping large objects into context.

Context is attached to many async operations.

If you place:

```js
hugeRequestBody
largeUserObject
binaryBuffer
```

in a long-lived context, you may increase retention and memory pressure.

---

# 36. Memory Retention Hazard

Bad:

```js
asyncLocalStorage.run(
  {
    request,
    response,
    body: hugeBuffer,
    massiveCacheObject,
  },
  handler,
);
```

Better:

```js
asyncLocalStorage.run(
  {
    requestId,
    traceId,
    tenantId,
  },
  handler,
);
```

Context should contain **small, stable, diagnostic metadata**.

---

# 37. Security of Async Context

Treat context as potentially sensitive.

Potentially sensitive values include:

- user IDs,
- tenant IDs,
- authorization metadata,
- trace data,
- request headers,
- IP addresses,
- internal identifiers.

Do not automatically send all context into:

- logs,
- metrics labels,
- traces,
- error reports.

---

# 38. Cardinality Hazard

Never casually use arbitrary user IDs as metric dimensions:

```text
metric{user_id="123456789"}
```

High-cardinality metrics can become expensive or operationally useless.

Use context to correlate logs/traces, while keeping metric labels bounded.

---

# 39. Context Leaks

A context leak is when data from one logical operation becomes visible in another operation incorrectly.

Example conceptual failure:

```text
Request A
  context = A

Request B
  unexpectedly sees A
```

Consequences can include:

- incorrect logs,
- broken authorization assumptions,
- tenant isolation failure,
- corrupted traces,
- misleading diagnostics.

This is one reason context should not contain mutable security-critical global state.

---

# 40. Context Contamination in Tests

Async-context bugs often appear in tests where shared state survives unexpectedly.

Use explicit test setup:

```js
storage.run(
  {
    requestId: 'test-request',
  },
  async () => {
    await testBody();
  },
);
```

Ensure test cleanup and isolation.

Test parallelism can make hidden context assumptions especially difficult to debug.

---

# 41. Event Emitters

Event emitters deserve special attention because events can cross abstraction boundaries.

Potential pattern:

```js
emitter.on('data', handler);
```

Ask:

> In which async context will `handler` execute?

If a library schedules event delivery through a custom queue or resource boundary, you may need an explicit async-resource strategy.

Do not assume that every callback retains the context in the exact way you expect merely because a callback was registered inside `run()`.

---

# 42. Streams

Streams create many callbacks across time.

For example:

```js
stream.on('data', chunk => {
  logger.info({
    requestId: getContext()?.requestId,
    size: chunk.length,
  }, 'chunk received');
});
```

For request-scoped streams, confirm that the stream's lifecycle preserves the intended context.

For reusable streams, avoid associating a single request context with a stream whose lifetime crosses requests.

---

# 43. Queue Consumers

Request context and job context are different.

For a queue:

```text
job
 ├── jobId
 ├── tenantId
 ├── traceId
 └── attempt
```

Establish context when the worker begins processing that job:

```js
asyncLocalStorage.run(
  {
    jobId: job.id,
    tenantId: job.tenantId,
    traceId: job.traceId,
  },
  () => processJob(job),
);
```

This prevents accidental reuse of an HTTP request's context.

---

# 44. Scheduled Jobs

Cron-like workloads should create explicit context:

```js
asyncLocalStorage.run(
  {
    jobId: 'nightly-reconciliation',
    trigger: 'cron',
  },
  runReconciliation,
);
```

Do not assume scheduled work belongs to whichever request happened to execute earlier.

---

# 45. Diagnostics as a Causal Graph

The deepest mental model:

```text
origin
  │
  ├── async resource A
  │       ├── child A1
  │       └── child A2
  │
  └── async resource B
          └── child B1
```

Context propagation lets diagnostic systems approximate:

```text
logical cause
     ↓
asynchronous descendants
```

This is why it is so valuable for distributed systems debugging.

---

# 46. Why Call Stacks Are Not Enough

A production incident might show:

```text
Error: connection reset
    at executeQuery (...)
```

That tells you:

> where the error surfaced.

It may not tell you:

> which request caused it.

Async context supplies correlation:

```text
requestId=req-928
traceId=abc123
operation=checkout
database=orders
```

Now stack + context becomes much more actionable.

---

# 47. Async Context and Error Handling

A request handler can log:

```js
try {
  await execute();
} catch (error) {
  logger.error({
    error,
    requestId: getContext()?.requestId,
  }, 'operation failed');

  throw error;
}
```

But avoid turning context into hidden error-control state.

Errors should still move through explicit application boundaries.

---

# 48. `diagnostics_channel`

Node also exposes `node:diagnostics_channel` for publishing and subscribing to diagnostic events.

Conceptually:

```text
instrumented component
      ↓
diagnostics channel
      ↓
observer / instrumentation
```

This can enable low-coupling diagnostics and instrumentation.

Use it for observation and tooling rather than arbitrary application business events.

---

# 49. Diagnostics Channel vs AsyncLocalStorage

| Tool | Primary job |
|---|---|
| AsyncLocalStorage | store/retrieve execution context |
| async_hooks | inspect async resource lifecycle |
| AsyncResource | model custom async resources |
| diagnostics_channel | publish/observe diagnostic events |
| process APIs | inspect process/runtime state |

These tools can complement each other.

---

# 50. Debugging Context Loss

Use this workflow:

## Step 1

Log where context is created.

```js
log('context-created');
```

## Step 2

Log before the async boundary.

```js
log('before-await');
```

## Step 3

Log after the boundary.

```js
log('after-await');
```

## Step 4

Check event-emitter/custom-resource boundaries.

## Step 5

Check third-party/native integrations.

## Step 6

Build a minimal reproduction.

## Step 7

Use `AsyncResource` only when the missing relationship is truly your abstraction's responsibility.

---

# 51. Debugging Wrong Context

This is more dangerous than missing context.

Example:

```text
Request A starts
Request B starts
A logs as B
```

Possible causes:

- shared mutable store,
- global variable,
- incorrect `enterWith`,
- manually cached context,
- reused singleton object,
- custom async abstraction crossing contexts incorrectly.

---

# 52. Global Variable Anti-Pattern

Never do:

```js
let currentRequestId;

server.on('request', (req) => {
  currentRequestId = req.headers['x-request-id'];

  doAsyncWork().then(() => {
    console.log(currentRequestId);
  });
});
```

Concurrent requests overwrite the shared variable.

Async context avoids this class of race:

```text
A → context A
B → context B
```

instead of:

```text
global mutable variable
```

---

# 53. `enterWith()` Footgun

Conceptual hazard:

```js
emitter.on('event', () => {
  storage.enterWith({ requestId: 'A' });
});

emitter.on('event', () => {
  console.log(storage.getStore());
});
```

Because `enterWith()` changes the current context for subsequent execution in that chain, event-driven code can accidentally inherit context farther than intended.

Prefer:

```js
storage.run(context, () => {
  emitter.emit('event');
});
```

when you can establish a clear scope.

---

# 54. Context Ownership

For every context field ask:

```text
Who creates it?
Who is allowed to modify it?
How long does it live?
Who trusts it?
Where is it logged?
Can it leak across tenants?
```

This makes async context a governed architecture instead of magical state.

---

# 55. Performance Considerations

Async context propagation has runtime cost.

Potential sources:

- context bookkeeping,
- object allocation,
- async resource tracking,
- instrumentation,
- logging enrichment.

Do not assume the overhead is zero.

The right approach is:

```text
measure
   ↓
identify actual bottleneck
   ↓
optimize only where evidence supports it
```

---

# 56. Context Object Size

Small:

```js
{
  requestId,
  traceId,
  tenantId,
}
```

Potentially expensive:

```js
{
  request,
  user,
  headers,
  parsedBody,
  databaseClient,
  logger,
  giantCache,
}
```

The second design can retain large object graphs unnecessarily.

---

# 57. Avoid Context as Service Locator

Bad:

```js
const ctx = storage.getStore();

ctx.db
ctx.cache
ctx.userService
ctx.config
ctx.featureFlags
```

Now async context has become a hidden dependency injection container.

Better:

```text
context → metadata
services → explicit dependencies
```

---

# 58. Memory and Lifetime

A context object can stay reachable as long as its associated asynchronous work remains reachable.

Therefore:

```text
long-lived async operation
       +
large context
       =
potential retention
```

This matters especially for:

- streaming,
- long polling,
- WebSockets,
- queue consumers,
- worker pools.

---

# 59. Production Architecture

A clean design:

```text
Incoming boundary
       │
       ▼
Context factory
       │
       ▼
AsyncLocalStorage.run()
       │
       ├──────────────┐
       ▼              ▼
business logic    observability
       │              │
       ▼              ▼
   explicit APIs   logging/tracing
```

Context should be introduced at system boundaries.

---

# 60. Context Factory

Example:

```js
function createContext({ requestId, traceId, tenantId }) {
  return Object.freeze({
    requestId,
    traceId,
    tenantId,
    startedAt: Date.now(),
  });
}
```

Then:

```js
const context = createContext({
  requestId,
  traceId,
  tenantId,
});

asyncLocalStorage.run(context, () => {
  return handleRequest(req, res);
});
```

---

# 61. Context Accessor

Centralize access:

```js
function getContext() {
  return asyncLocalStorage.getStore();
}

function requireContext() {
  const context = getContext();

  if (!context) {
    throw new Error('No async context available');
  }

  return context;
}
```

Decide explicitly where context is required and where it is optional.

---

# 62. Context-Enriched Logger

```js
function createLogger(baseLogger) {
  return {
    info(fields, message) {
      const context = getContext();

      baseLogger.info(
        {
          ...fields,
          requestId: context?.requestId,
          traceId: context?.traceId,
          tenantId: context?.tenantId,
        },
        message,
      );
    },
  };
}
```

Prefer filtering and redaction before emitting fields.

---

# 63. Transaction Context: Important Caution

It may be tempting to store:

```js
{
  transaction: dbTransaction
}
```

inside async context.

This can hide ownership and make transaction boundaries difficult to reason about.

A stronger design is often:

```text
explicit transaction object
+
optional context containing transaction ID / diagnostics
```

rather than:

```text
transaction exists because context says so
```

---

# 64. Cancellation + Context

A mature request context can include:

```js
{
  requestId,
  traceId,
  signal,
}
```

where `signal` is an `AbortSignal` representing request cancellation.

But separate concerns:

```text
context = metadata
signal = cancellation control
```

Do not confuse the two.

---

# 65. Context Across Retries

A retry is a design choice.

Should retries have:

```text
same trace
same request ID
new attempt ID
```

Often a useful model is:

```text
logical operation
   │
   ├── attempt 1
   ├── attempt 2
   └── attempt 3
```

Store both:

```text
operationId
attempt
```

when diagnostics require that distinction.

---

# 66. Context Across Queues

A queue hop changes the execution boundary.

Use explicit serialization:

```js
{
  jobId,
  traceId,
  tenantId,
  attempt
}
```

Then reconstruct context in the consumer.

This is more reliable than expecting in-process context to magically survive a network boundary.

---

# 67. Context Across HTTP

Likewise, outgoing HTTP should propagate only appropriate tracing/correlation metadata.

Do not blindly forward every request header.

Use explicit allowlists.

---

# 68. Distributed Context

A useful conceptual stack:

```text
process context
   ↓
request context
   ↓
trace context
   ↓
outgoing propagation
   ↓
remote service context
```

The process-local runtime handles only part of the chain.

Network propagation requires a protocol.

---

# 69. Diagnostic Signals

Combine:

```text
context
+
process state
+
resource state
+
error
+
timing
```

For example:

```js
logger.error({
  event: 'request.failed',
  requestId,
  traceId,
  pid: process.pid,
  uptime: process.uptime(),
  resources: process.getActiveResourcesInfo(),
  error,
}, 'Request failed');
```

Use expensive diagnostics selectively rather than on every request.

---

# 70. Production Incident Example

Suppose:

```text
checkout request
   ↓
DB insert
   ↓
Kafka publish
   ↓
email service call
   ↓
timeout
```

Without context:

```text
DB timeout
Kafka timeout
HTTP timeout
```

seem like unrelated failures.

With context:

```text
requestId=req-781
traceId=trace-22
operation=checkout
```

they can be correlated into one causal chain.

---

# 71. Debugging Exercise — Context Loss

```js
const storage = new AsyncLocalStorage();

storage.run({ requestId: 'A' }, () => {
  customLibrary((value) => {
    console.log(storage.getStore());
  });
});
```

### Task

Assume the library later executes the callback through a custom scheduler and the context is unexpectedly missing.

Determine whether:

1. the library should preserve context,
2. the caller should wrap the callback,
3. the library should model a custom `AsyncResource`.

Defend the design.

---

# 72. Debugging Exercise — Context Contamination

```js
const context = {};

storage.run(context, async () => {
  context.userId = 'A';
  await doWork();
});
```

Then another layer mutates the same object.

### Task

Explain why mutable context can become difficult to reason about.

---

# 73. Debugging Exercise — Cross-Request Global

```js
let currentUser;

server.on('request', async (req) => {
  currentUser = await authenticate(req);

  await service();

  console.log(currentUser);
});
```

### Task

Prove why this is broken under concurrency.

---

# 74. Debugging Exercise — Worker Boundary

A request starts:

```text
requestId = A
```

then submits work to a worker thread.

The worker logs:

```text
requestId = undefined
```

### Task

Design explicit propagation.

---

# 75. Debugging Exercise — Long-Lived Context

A WebSocket connection keeps a context containing a 20 MB parsed object.

The process has increasing memory usage.

### Task

Explain the retention risk and redesign the context.

---

# 76. Common Misconceptions

## Misconception 1

> `AsyncLocalStorage` creates global variables.

Not exactly.

It provides context associated with asynchronous execution.

## Misconception 2

> It automatically solves distributed tracing.

No.

Network propagation still requires a protocol and explicit propagation.

## Misconception 3

> It should contain all request state.

No.

Keep context small.

## Misconception 4

> `AsyncLocalStorage` replaces parameters.

No.

It is best for cross-cutting metadata, not every dependency.

## Misconception 5

> Every callback automatically preserves the context I want.

Usually supported async paths do, but custom abstractions and unusual boundaries can require explicit modeling.

## Misconception 6

> `async_hooks` should be used for all async code.

No.

It is a lower-level mechanism.

## Misconception 7

> Context is safe to trust for authorization.

No.

Metadata can be propagated without being authoritative security state.

---

# 77. Common Mistakes

## Mistake 1

Storing large objects.

## Mistake 2

Storing mutable shared objects.

## Mistake 3

Using `enterWith()` casually.

## Mistake 4

Hiding essential business dependencies.

## Mistake 5

Logging every context field.

## Mistake 6

Using high-cardinality context as metric dimensions.

## Mistake 7

Expecting context to cross worker/process/network boundaries automatically.

---

# 78. Comparison With Related Concepts

| Concept | Primary purpose |
|---|---|
| Lexical scope | identifier visibility |
| Closure | retain lexical bindings |
| Promise | represent async completion |
| Async resource | runtime representation of async operation |
| `AsyncLocalStorage` | propagate logical context |
| `AsyncResource` | model custom async execution |
| `async_hooks` | observe async resource lifecycle |
| `diagnostics_channel` | publish/observe diagnostics |
| logger | record events |
| tracer | represent causal execution |
| metric | aggregate numerical behavior |

---

# 79. Specification Boundary

`AsyncLocalStorage`, `async_hooks`, process diagnostics, and Node's async-resource model are **Node.js host/runtime APIs**, not ECMAScript language features.

ECMAScript defines the JavaScript language and Promise/job semantics.

Node defines additional runtime mechanisms for:

- async resource tracking,
- context propagation,
- process diagnostics,
- operating-system integration.

Always separate:

```text
ECMAScript guarantee
```

from:

```text
Node.js runtime behavior
```

---

# 80. Performance Model

A useful conceptual cost model:

```text
request
  ↓
context allocation
  ↓
async operations
  ↓
context propagation bookkeeping
  ↓
logging/tracing
```

Costs can arise from:

- context object creation,
- runtime bookkeeping,
- instrumentation,
- serialization,
- logging,
- high-frequency diagnostic work.

The answer is not:

> “Never use async context.”

The answer is:

> **Use it where its observability and correctness value exceeds its measured cost.**

---

# 81. Memory Model

Context lifetime follows logical async work.

Therefore:

```text
short operation + small context
    = low retention risk

long operation + huge context
    = higher retention risk
```

Be particularly cautious with:

- WebSockets,
- long polls,
- subscriptions,
- streaming uploads,
- streaming downloads,
- background consumers.

---

# 82. Security Model

Async context can become a data-propagation channel.

Security questions:

```text
Can tenant A's context reach tenant B?
Can secrets reach logs?
Can untrusted correlation IDs be trusted?
Can context cross privilege boundaries?
Can child processes receive sensitive context?
Can trace metadata leak sensitive identifiers?
```

Treat context as a controlled data plane.

---

# 83. Production Usage Pattern

A strong service architecture can use:

```js
import { AsyncLocalStorage } from 'node:async_hooks';

const contextStorage = new AsyncLocalStorage();

function createRequestContext(req) {
  return Object.freeze({
    requestId: getOrCreateRequestId(req),
    traceId: extractTraceId(req),
    startedAt: Date.now(),
  });
}

async function handleRequest(req, res) {
  const context = createRequestContext(req);

  return contextStorage.run(
    context,
    async () => {
      logger.info({
        event: 'request.started',
        requestId: context.requestId,
      }, 'request started');

      try {
        await route(req, res);
      } catch (error) {
        logger.error({
          event: 'request.failed',
          requestId: context.requestId,
          error,
        }, 'request failed');

        throw error;
      }
    },
  );
}
```

The key architecture is:

```text
boundary creates context
          ↓
context remains small
          ↓
business inputs remain explicit
          ↓
logging/tracing consume context
          ↓
external boundaries propagate explicitly
```

---

# 84. Implementation From Scratch

## Stage A — Guided

Implement:

```js
createContextStorage()
getContext()
requireContext()
runWithContext()
```

Requirements:

- one global storage instance,
- immutable context objects,
- clear missing-context behavior.

---

## Stage B — Partially Guided

Build an HTTP server with:

```text
requestId
traceId
start time
```

Then log these values from:

- synchronous code,
- Promise continuations,
- timers,
- stream callbacks.

---

## Stage C — No Reference

Build:

```text
request context
  ↓
service
  ↓
repository
  ↓
outgoing HTTP client
```

without passing request ID through every function parameter.

---

## Stage D — Edge-Case Hardened

Add:

- worker-thread propagation,
- queue-job propagation,
- cancellation signal,
- context redaction,
- missing-context detection,
- custom async abstraction.

---

## Stage E — Production Grade

Add:

- structured logs,
- trace integration,
- metrics correlation without high-cardinality dimensions,
- context size limits,
- security review,
- diagnostic report capture strategy,
- performance benchmarks,
- concurrency tests.

---

# 85. No-Reference Implementation Challenge

Implement an API:

```js
createContextManager({
  makeContext,
  validateContext,
  sanitizeContext,
});
```

Expose:

```js
run(context, fn)
get()
require()
```

Then support:

```text
HTTP request
queue job
cron job
worker task
```

as separate context-entry boundaries.

---

# 86. Code Review Exercise

Review:

```js
let context = {};

export function setContext(value) {
  context = value;
}

export async function handle(req, res) {
  context.requestId = req.headers['x-request-id'];

  await service();

  logger.info({
    requestId: context.requestId,
    body: req.body,
  });

  res.end();
}
```

Find at least ten problems.

Expected areas:

- global mutable state,
- concurrency,
- cross-request contamination,
- trust boundary,
- sensitive logging,
- context ownership,
- memory retention,
- explicit dependencies,
- error handling,
- observability design.

---

# 87. Interview Questions

## Foundation

1. What problem does `AsyncLocalStorage` solve?
2. How is async context different from lexical scope?
3. Does a Promise itself define context?
4. What is an async resource?
5. What is `AsyncResource`?
6. What is `async_hooks` used for?
7. Why is `AsyncLocalStorage` better suited than manual async-hooks logic for common context propagation?

## Intermediate

8. How would you add request IDs to every log line?
9. How do you prevent cross-request context contamination?
10. Why should context objects be small?
11. Why is `enterWith()` easier to misuse than `run()`?
12. What happens at a worker-thread boundary?
13. What happens at a child-process boundary?
14. What happens at a network boundary?
15. When should a business value remain an explicit function parameter?

## Advanced

16. How would you debug missing async context?
17. What is the role of `AsyncResource` in custom libraries?
18. How would you reason about context through event emitters?
19. What is the performance cost of async context?
20. How can async context create memory retention?
21. Why should metric dimensions not blindly include request/user IDs?
22. How can wrong context become a security issue?

## Principal Level

23. Design context propagation for HTTP → queue → worker → DB → outbound HTTP.
24. How would you test context isolation under 10,000 concurrent requests?
25. How would you handle retries while preserving causal tracing?
26. How would you separate correlation metadata from authorization state?
27. How would you diagnose a context leak across reusable stream consumers?
28. When would you reject `AsyncLocalStorage` in favor of explicit parameters?
29. How would you benchmark the performance impact?
30. How would you make the design work across process and network boundaries?

---

# 88. Predict-the-Output Exercises

## Exercise A

```js
const storage = new AsyncLocalStorage();

storage.run({ id: 'A' }, () => {
  console.log(storage.getStore().id);
});
```

Predict the output.

---

## Exercise B

```js
storage.run({ id: 'A' }, () => {
  setTimeout(() => {
    console.log(storage.getStore().id);
  }, 0);
});
```

Predict the conceptual result.

---

## Exercise C

```js
storage.run({ id: 'A' }, async () => {
  await Promise.resolve();
  console.log(storage.getStore().id);
});
```

Predict the result.

---

## Exercise D

```js
storage.run({ id: 'A' }, () => {
  console.log(storage.getStore().id);
});

console.log(storage.getStore());
```

Predict both observations.

---

## Exercise E

Two concurrent requests each execute:

```js
storage.run({ id }, async () => {
  await delay(10);
  console.log(storage.getStore().id);
});
```

with IDs:

```text
A
B
```

Can the two callbacks safely observe their own context?

Explain why.

---

# 89. Mastery Exercises

## Exercise 1 — Request Correlation

Implement:

```text
HTTP server
  ↓
AsyncLocalStorage
  ↓
structured logger
```

Every request must have an isolated correlation ID.

---

## Exercise 2 — Context Isolation Test

Generate:

```text
1,000 concurrent logical requests
```

Each with a unique ID.

Verify:

```text
no callback ever sees another request's ID
```

---

## Exercise 3 — Custom AsyncResource

Build a toy scheduler:

```text
submit(contextualTask)
```

and use `AsyncResource` to preserve the intended async relationship.

---

## Exercise 4 — Queue Boundary

Serialize:

```js
{
  jobId,
  traceId,
  tenantId,
}
```

through a fake queue.

Reconstruct async context at the consumer.

---

## Exercise 5 — Diagnostic Incident

Create a service where:

```text
Request A
Request B
Request C
```

perform interleaved asynchronous work.

Produce logs that allow an operator to reconstruct each request independently.

---

## Exercise 6 — Memory Pressure

Intentionally place a large object in context.

Measure memory behavior during long-lived async work.

Then remove the large object and compare.

---

## Exercise 7 — Principal Architecture

Design context propagation for:

```text
API Gateway
   ↓
Node service
   ↓
Kafka
   ↓
worker threads
   ↓
PostgreSQL
   ↓
third-party HTTP API
```

Identify every context boundary and specify what is:

```text
stored
propagated
trusted
redacted
terminated
```

---

# 90. Production Failure Scenarios

## Scenario 1 — Wrong Tenant in Logs

Logs for tenant B contain tenant A's ID.

Investigate:

```text
global mutable state
↓
context reuse
↓
incorrect enterWith
↓
custom async boundary
```

---

## Scenario 2 — Trace IDs Disappear in DB Logs

Investigate:

```text
context created too late
context lost in integration library
custom scheduler
native boundary
```

---

## Scenario 3 — Memory Grows During WebSocket Load

Investigate:

```text
large context
+
long-lived connection
=
retention
```

---

## Scenario 4 — Metrics Explode in Cardinality

Investigate:

```text
requestId/userId
      ↓
metric labels
      ↓
millions of time series
```

Redesign metrics independently of log/trace correlation.

---

# 91. Principal Decision Framework

When designing async context, evaluate:

| Dimension | Question |
|---|---|
| Correctness | Does each async operation see the correct context? |
| Performance | Is propagation overhead measured and acceptable? |
| Memory | Is the context small and bounded? |
| Security | Can sensitive context cross trust boundaries? |
| Reliability | Does diagnostic behavior survive failures? |
| Maintainability | Is context usage centralized and documented? |
| Scalability | Does context work under high concurrency? |
| Observability | Does it materially improve causal diagnosis? |
| Developer Experience | Is usage predictable and testable? |
| Operational Complexity | Are propagation boundaries explicit? |
| Future Change | Can workers, queues, and services be added safely? |

---

# 92. Deep Mental Model

Keep this model permanently:

```text
Lexical scope
    = where names are visible

Closure
    = what lexical state a function retains

Promise
    = relationship to a future result

Async resource
    = runtime-managed unit of asynchronous work

Async context
    = logical metadata associated with async execution

AsyncLocalStorage
    = convenient Node mechanism to access that context

AsyncResource
    = mechanism for custom async abstractions to model context correctly

Diagnostics
    = evidence about what the runtime/application is doing

Observability
    = turning that evidence into operational understanding
```

---

# 93. Chapter Connections

## Depends On

- Chapter 10 — Scope and Lexical Environments
- Chapter 12 — Execution Contexts
- Chapter 13 — Closures
- Chapter 31 — Async Fundamentals
- Chapter 32 — ECMAScript Jobs
- Chapter 34 — Node Event Loop
- Chapter 35 — Promises
- Chapter 36 — Async/Await
- Chapter 37 — Cancellation
- Chapter 38 — Async Iteration
- Chapter 58 — Node Architecture
- Chapter 62 — Process Lifecycle

## Builds Toward

- Chapter 63 diagnostics mastery in production workflows
- Chapter 70 — Source Maps and Production Debugging
- Chapter 78 — Production JavaScript Architecture
- Chapter 83 — Observability
- Chapter 84 — Reliability
- Chapter 85 — Performance
- Chapter 88 — Debugging Methodology
- Chapter 101 — Real-World Production Scenarios
- Chapter 105 — Node REST API
- Chapter 106 — Real-time WebSocket
- Chapter 107 — Job Queue
- Chapter 110 — Production JS Backend
- Chapter 111 — Large-Scale JS Platform

## Related Concepts

- event loop,
- execution context,
- tracing,
- structured logging,
- cancellation,
- workers,
- queues,
- streams,
- diagnostics,
- observability.

## Why This Chapter Matters Later

At principal level, debugging is not:

> “Where did the exception happen?”

It is:

> **“What logical operation caused this work, what happened along the async path, and what evidence lets us reconstruct that causal chain?”**

Async context is one of the key runtime mechanisms enabling that answer.

---

# 94. Spaced Retrieval Plan

## Day 0

Explain:

```text
scope ≠ context
Promise ≠ context
context ≠ authorization
context ≠ dependency injection
```

## Day 2

Implement:

```text
requestId propagation
```

without notes.

## Day 7

Debug:

```text
undefined async context
```

using a custom reproduction.

## Day 14

Design:

```text
HTTP → queue → worker
```

context propagation.

## Day 30

Defend:

> When is async context the wrong abstraction?

---

# 95. Dependency Graph

```text
Scope / closure
      │
      ▼
Async execution
      │
      ▼
Node async resources
      │
      ├──────────────┐
      ▼              ▼
AsyncLocalStorage   AsyncResource
      │              │
      └──────┬───────┘
             ▼
        context flow
             │
       ┌─────┼─────┐
       ▼     ▼     ▼
    logging tracing diagnostics
       │     │     │
       └─────┼─────┘
             ▼
      production debugging
```

---

# 96. Completion Criteria

```text
[ ] Explain lexical scope vs async context
[ ] Explain Promise vs async resource
[ ] Explain AsyncLocalStorage
[ ] Use run()
[ ] Use getStore()
[ ] Understand enterWith()
[ ] Understand disable()
[ ] Explain async resource lifecycle
[ ] Explain AsyncResource
[ ] Explain async_hooks conceptually
[ ] Diagnose context loss
[ ] Prevent context contamination
[ ] Propagate context across workers explicitly
[ ] Propagate context across queues explicitly
[ ] Separate metadata from authorization state
[ ] Keep context small
[ ] Avoid high-cardinality metrics
[ ] Integrate context with structured logging
[ ] Integrate context with tracing
[ ] Explain diagnostics_channel
[ ] Use process diagnostics
[ ] Reason about memory retention
[ ] Implement context manager from scratch
[ ] Pass principal interview questions
[ ] Pass concurrency isolation tests
```

---

# 97. Mastery Gate

You have mastered this chapter only when you can:

### Understand
Explain async context from runtime first principles.

### Explain
Teach `AsyncLocalStorage` without presenting it as magic global state.

### Predict
Predict whether context survives unfamiliar asynchronous boundaries.

### Implement
Build context propagation and a custom async-resource abstraction.

### Debug
Locate the exact boundary where context is lost or contaminated.

### Apply
Use context safely in HTTP, jobs, workers, streams, and diagnostics.

### Compare
Defend explicit parameters versus async context.

### Defend
Explain the performance, memory, security, and maintenance trade-offs to a principal engineering review.

---

# 98. Status

```text
[ ] Not Started
[~] In Progress
[?] Needs Revision
[+] Completed
[*] Mastered
```

Current chapter status:

```text
[ ] Not Started
```

Reading alone does not mark mastery.

---

# 99. Chapter 63 — Revision / Retrieval Record

| Date | Recall task | Result | Weak area | Next action |
|---|---|---|---|---|
| YYYY-MM-DD | Explain AsyncLocalStorage | ✅ / ❌ | ... | ... |
| YYYY-MM-DD | Diagnose context loss | ✅ / ❌ | ... | ... |
| YYYY-MM-DD | Implement AsyncResource | ✅ / ❌ | ... | ... |
| YYYY-MM-DD | Design distributed context | ✅ / ❌ | ... | ... |

### Retrieval prompts

```text
1. Why is a global variable unsafe for request context?
2. Why is async context different from lexical scope?
3. When should context be explicit?
4. What does AsyncResource solve?
5. Why can context disappear?
6. Why should context remain small?
7. What is the difference between logs, traces, and metrics?
8. What boundaries require explicit propagation?
9. Why is context not authorization?
10. When should you reject AsyncLocalStorage?
```

---

# 100. Chapter 63 — Canonical References and Source Discipline

## Primary Node.js references

- Node.js `async_hooks`  
  https://nodejs.org/api/async_hooks.html
- Node.js `AsyncLocalStorage`  
  https://nodejs.org/api/async_context.html
- Node.js diagnostics channel  
  https://nodejs.org/api/diagnostics_channel.html
- Node.js process API  
  https://nodejs.org/api/process.html
- Node.js diagnostic reports  
  https://nodejs.org/api/report.html
- Node.js worker threads  
  https://nodejs.org/api/worker_threads.html

## ECMAScript

- ECMA-262  
  https://tc39.es/ecma262/

Use ECMAScript for language semantics such as:

- Promise behavior,
- Jobs,
- execution semantics.

Use Node.js documentation for:

- async context,
- async resources,
- process diagnostics,
- host-specific lifecycle behavior.

---

## Source discipline

1. Do not describe `AsyncLocalStorage` as an ECMAScript feature.
2. Distinguish public Node.js guarantees from implementation details.
3. Treat `async_hooks` internals as runtime-specific.
4. Verify current method names and semantics against current Node.js documentation.
5. Treat context behavior through third-party/native libraries as an integration concern that should be tested.
6. Distinguish local propagation from network propagation.
7. Never assume a diagnostic experiment alone establishes a universal language guarantee.

---

# 101. Chapter 63 — Completion Snapshot

## Core Theory

```text
[ ] Async context model
[ ] Async resource model
[ ] AsyncLocalStorage
[ ] AsyncResource
[ ] async_hooks
[ ] diagnostics_channel
```

## Implementation

```text
[ ] Request context manager
[ ] Context-aware logger
[ ] Concurrency isolation tests
[ ] Custom async resource
[ ] Queue context propagation
[ ] Worker context propagation
```

## Diagnostics

```text
[ ] Active resource inspection
[ ] Process metadata
[ ] Diagnostic report strategy
[ ] Context-loss debugging
[ ] Context-contamination debugging
```

## Production

```text
[ ] Context size limits
[ ] Security review
[ ] Redaction
[ ] Metrics cardinality review
[ ] Trace integration
[ ] Performance measurement
[ ] Memory retention review
```

## Mastery

```text
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

# Final Principal Perspective

Async context is valuable because production systems preserve meaning across time.

The call stack disappears.

The request does not.

A single business operation may span:

```text
HTTP
  ↓
Promise
  ↓
database
  ↓
queue
  ↓
worker
  ↓
HTTP client
  ↓
retry
  ↓
error
```

Without context, the system becomes a pile of disconnected observations.

With disciplined context propagation:

```text
logical operation
      ↓
correlated asynchronous descendants
      ↓
structured logs
      ↓
traces
      ↓
diagnostic evidence
      ↓
causal understanding
```

The deeper lesson is not:

> “Use `AsyncLocalStorage`.”

It is:

> **Make asynchronous causality observable without turning hidden context into hidden global state.**

A principal engineer should know exactly:

```text
what context exists
who creates it
who trusts it
where it propagates
where it must be serialized
how it is redacted
how long it lives
what it costs
and how to prove that it never crosses the wrong boundary
```

That is async context engineering.