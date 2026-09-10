
# Chapter 30 — Resource Management and Cleanup (`using`, `await using`, DisposableStack)

## 1. Learning Objectives

By the end of this chapter, the learner must be able to:

- Explain what a resource is in JavaScript application design.
- Distinguish memory-managed values from external resources that require deterministic cleanup.
- Explain why garbage collection does not provide timely release of arbitrary external resources.
- Explain the motivation behind explicit resource-management semantics.
- Understand synchronous disposal with `using`.
- Understand asynchronous disposal with `await using`.
- Understand `Symbol.dispose` and `Symbol.asyncDispose`.
- Explain how disposal is triggered on scope exit.
- Explain why cleanup semantics must work for normal completion, `return`, and thrown failures.
- Explain the role of `DisposableStack`.
- Explain the difference between `DisposableStack` and `AsyncDisposableStack`.
- Register cleanup actions safely and reason about cleanup order.
- Explain why resource cleanup is fundamentally a control-flow and reliability problem, not merely a memory problem.
- Compare `using` with traditional `try/finally`.
- Design APIs that expose disposable resources correctly.
- Implement disposable abstractions from scratch before using language/runtime facilities.
- Debug leaks, double disposal, cleanup failures, and partially initialized resources.
- Design production-safe resource ownership boundaries.
- Reason about cleanup under nested scopes, asynchronous work, exceptions, cancellation, and concurrency.
- Evaluate trade-offs among explicit disposal, pooling, ownership transfer, reference counting, and garbage collection.
- Understand how resource-management features connect to networking, files, streams, databases, workers, locks, and transactional systems.
- Make principal-level decisions about resource lifetime, ownership, cleanup guarantees, and failure policy.

### Mastery Gate

The chapter is mastered only when the learner can:

> Understand → Explain → Predict → Implement → Debug → Apply → Compare → Defend

Reading this chapter is not sufficient to claim mastery.

---

## 2. Prerequisites

The learner should already understand:

- JavaScript values, objects, and prototypes.
- Classes and constructors.
- Symbols and well-known symbols.
- Control flow and abrupt completion.
- `throw`, `try`, `catch`, and `finally`.
- Functions and lexical scope.
- Promises and async/await at a foundational level.
- Typed arrays and binary data at a foundational level.
- Basic Node.js/browser resource concepts.
- Error propagation and causal error handling.

Primary dependencies:

- Chapter 12 — Execution Contexts / Execution Model
- Chapter 15 — Objects / Property Semantics
- Chapter 18 — Classes / OOP
- Chapter 20 — Symbols / Well-Known Symbols
- Chapter 26 — Generators / Async Generators
- Chapter 27 — Typed Arrays / Binary Data
- Chapter 29 — Errors / Error Handling

This chapter is the bridge between language-level control flow and production-grade resource lifetime management.

---

## 3. What Is It?

A **resource** is anything whose useful lifetime is associated with an external or finite capability that should be released when the owning operation is finished.

Examples include:

- file handles;
- sockets;
- database connections;
- transactions;
- locks;
- subscriptions;
- streams;
- worker-related handles;
- native resources exposed through JavaScript bindings;
- temporary buffers or OS-backed resources;
- application-level resources that maintain an ongoing registration.

A resource has a lifetime:

```text
created
  ↓
acquired
  ↓
used
  ↓
released
```

The central question is:

> Who owns the resource, and exactly when does that ownership end?

Garbage collection answers a different question:

> When is a JavaScript object unreachable?

Those are not equivalent.

An object can become unreachable while an external system still requires explicit release, or the object may remain reachable much longer than the desired resource lifetime.

Resource management therefore requires **deterministic cleanup**.

Modern JavaScript provides language-level resource-management capabilities based on:

```js
using
await using
Symbol.dispose
Symbol.asyncDispose
DisposableStack
AsyncDisposableStack
```

The design goal is to make ownership and cleanup composable with lexical scope.

Conceptually:

```text
enter scope
   ↓
acquire resource
   ↓
use resource
   ↓
leave scope
   ↓
automatic disposal
```

---

## 4. Why Does It Exist?

Historically, JavaScript developers used:

```js
const resource = acquire();

try {
  use(resource);
} finally {
  resource.close();
}
```

This works.

But once a function owns multiple resources, cleanup becomes repetitive and error-prone:

```js
const a = acquireA();
try {
  const b = acquireB();

  try {
    const c = acquireC();

    try {
      use(a, b, c);
    } finally {
      c.close();
    }
  } finally {
    b.close();
  }
} finally {
  a.close();
}
```

The problem becomes harder when:

- acquisition itself can fail;
- different resources have different cleanup methods;
- some cleanup is asynchronous;
- resources are conditionally acquired;
- ownership is transferred;
- cleanup can throw;
- multiple cleanup failures occur;
- cancellation interrupts normal work.

Resource management features aim to make the ownership structure explicit:

```js
{
  using a = acquireA();
  using b = acquireB();
  await using c = acquireC();

  await use(a, b, c);
}
```

The language/runtime can then connect lexical scope exit with disposal.

The deeper reason is reliability:

> Cleanup is a correctness obligation, not an optional afterthought.

A leaked file handle, database connection, subscription, lock, or stream can cause production failures even when the application otherwise “works.”

---

## 5. Mental Model

Think in terms of **ownership**.

```text
scope owns resource
      │
      ├── use resource
      │
      └── scope exits
             │
             ▼
        dispose resource
```

For multiple resources:

```text
scope
 ├── resource A
 ├── resource B
 └── resource C

exit scope
   ↓
C disposed
B disposed
A disposed
```

The cleanup order is conceptually reverse acquisition order.

Why?

Because dependencies often flow in the same direction as acquisition:

```text
A created
  ↓
B depends on A
  ↓
C depends on B
```

Therefore:

```text
C → B → A
```

is the natural destruction sequence.

This resembles stack unwinding:

```text
acquire A
  acquire B
    acquire C
    release C
  release B
release A
```

The core mental model:

> Scope defines ownership; leaving scope triggers disposal.

For asynchronous resources:

```text
async scope
    ↓
await work
    ↓
scope exits
    ↓
await disposal
```

The disposal operation itself becomes part of the program's control flow.

---

## 6. Core Rules

### Rule 1 — Explicit resource ownership is different from garbage collection

GC tracks JavaScript object reachability.

Disposal tracks a resource's lifecycle contract.

### Rule 2 — A disposable value exposes a protocol

Synchronous disposal uses:

```js
Symbol.dispose
```

Asynchronous disposal uses:

```js
Symbol.asyncDispose
```

### Rule 3 — `using` is scope-bound

A resource declared with `using` is disposed when its containing scope exits according to the resource-management semantics.

### Rule 4 — `await using` is for asynchronous disposal

Its cleanup may return a promise and therefore requires asynchronous scope semantics.

### Rule 5 — Cleanup occurs on abnormal exits too

Resource disposal must account for:

- normal completion;
- `return`;
- thrown exceptions;
- other abrupt exits supported by the construct.

### Rule 6 — Disposal order is important

Multiple resources are generally disposed in reverse declaration/acquisition order.

### Rule 7 — Cleanup itself can fail

Disposal is code.

Therefore:

```js
resource[Symbol.dispose]()
```

can throw.

Async disposal can reject.

### Rule 8 — Cleanup failure must not be treated as impossible

The design must preserve meaningful failure information when both the main operation and cleanup fail.

### Rule 9 — A resource should have one clear owner at a time

Ambiguous ownership leads to:

- double disposal;
- leaked resources;
- use-after-disposal;
- premature cleanup.

### Rule 10 — Scope boundaries should match ownership boundaries

A resource should generally live no longer than necessary.

### Rule 11 — Async cleanup is not interchangeable with sync cleanup

A synchronous resource cannot magically become asynchronous just because surrounding code uses `await`.

### Rule 12 — Disposal is a lifecycle operation

Do not confuse:

```text
dispose
close
abort
cancel
release
destroy
shutdown
```

They may have different semantics.

The API contract must define what actually happens.

---

## 7. Syntax

### `using`

Representative form:

```js
{
  using resource = acquireResource();
  use(resource);
}
```

The resource is disposed when the scope exits.

### `await using`

Representative form:

```js
{
  await using resource = acquireAsyncResource();
  await use(resource);
}
```

The disposal step can be asynchronous.

### Resource protocols

Synchronous:

```js
class Resource {
  [Symbol.dispose]() {
    // release
  }
}
```

Asynchronous:

```js
class AsyncResource {
  async [Symbol.asyncDispose]() {
    // asynchronous release
  }
}
```

### Disposable stack

```js
const stack = new DisposableStack();

stack.use(resource);
stack.defer(() => cleanupSomething());
```

At the end:

```js
stack.dispose();
```

For asynchronous lifetimes, use the async counterpart where supported:

```js
const stack = new AsyncDisposableStack();

stack.use(resource);
stack.defer(async () => {
  await cleanup();
});
```

The exact supported syntax/API surface depends on the current ECMAScript/runtime implementation. Distinguish language semantics from runtime availability.

### Ownership transfer

A resource manager may need an explicit way to move ownership:

```text
owner A
  ↓ transfer
owner B
```

`DisposableStack`-style APIs are designed to support composable ownership patterns, including detaching/disarming cleanup when ownership is transferred.

---

## 8. Basic Examples

### Basic synchronous disposable object

```js
class FileHandle {
  constructor(path) {
    this.path = path;
    this.closed = false;
  }

  write(data) {
    if (this.closed) {
      throw new Error("Handle is closed");
    }

    console.log(`Writing ${data} to ${this.path}`);
  }

  [Symbol.dispose]() {
    if (this.closed) {
      return;
    }

    this.closed = true;
    console.log(`Closed ${this.path}`);
  }
}
```

Usage:

```js
{
  using file = new FileHandle("data.txt");

  file.write("hello");
}
```

Conceptually:

```text
construct
→ write
→ scope exits
→ dispose
```

### Traditional equivalent

The same ownership idea can be expressed as:

```js
const file = new FileHandle("data.txt");

try {
  file.write("hello");
} finally {
  file[Symbol.dispose]();
}
```

The language-level feature mainly reduces ceremony and makes cleanup composable.

### Multiple resources

```js
{
  using first = createResource("first");
  using second = createResource("second");

  use(first, second);
}
```

The disposal order should be reasoned about as:

```text
second
first
```

### Asynchronous disposal

```js
class AsyncConnection {
  async connect() {
    console.log("connected");
  }

  async [Symbol.asyncDispose]() {
    console.log("closing...");
    await new Promise(resolve => setTimeout(resolve, 10));
    console.log("closed");
  }
}
```

Conceptually:

```js
{
  await using connection = new AsyncConnection();
  await connection.connect();
}
```

The scope cannot be considered fully exited until asynchronous disposal semantics complete.

---

## 9. Execution Walkthrough

Consider:

```js
class Resource {
  constructor(name) {
    this.name = name;
  }

  [Symbol.dispose]() {
    console.log(`dispose ${this.name}`);
  }
}

function work() {
  using a = new Resource("A");
  using b = new Resource("B");

  console.log("work");
}

work();
```

### Step 1

Enter `work()`.

### Step 2

Create resource `A`.

Ownership becomes:

```text
work scope → A
```

### Step 3

Create resource `B`.

Ownership becomes:

```text
work scope → A, B
```

### Step 4

Execute:

```js
console.log("work");
```

### Step 5

The function scope exits.

### Step 6

The language resource-management semantics perform disposal.

### Step 7

`B` is disposed.

### Step 8

`A` is disposed.

Expected output:

```text
work
dispose B
dispose A
```

The important point is that the cleanup is attached to scope exit rather than relying on every early-return branch being manually updated.

---

## 10. Internal Mechanics

### 10.1 Disposal is structured cleanup

The semantics conceptually maintain disposable resources associated with lexical execution state.

When a `using` declaration succeeds, the resource becomes part of the scope's disposal obligations.

When the scope exits, the associated disposal operations are performed according to the resource-management rules.

### 10.2 Acquisition and registration are not the same event

The expression:

```js
using resource = acquire();
```

has two important phases:

```text
evaluate initializer
       ↓
obtain value
       ↓
validate / register disposable behavior
```

If acquisition fails before ownership exists, there may be nothing to dispose.

If acquisition succeeds but subsequent registration/use fails, cleanup semantics must still preserve the resource obligation.

This distinction matters for partial initialization.

### 10.3 LIFO cleanup

Suppose:

```js
{
  using A = acquireA();
  using B = acquireB();
  using C = acquireC();
}
```

The resulting cleanup stack behaves conceptually like:

```text
push A
push B
push C

pop C
pop B
pop A
```

This ordering minimizes dependency violations in common ownership graphs.

### 10.4 Disposal and abrupt completion

Suppose:

```js
{
  using resource = acquire();

  throw new Error("work failed");
}
```

The resource must still be disposed as part of scope exit.

The resulting outcome must account for both:

```text
primary failure
+
cleanup outcome
```

This connects directly to Chapter 29.

### 10.5 Async disposal

With:

```js
await using resource = acquire();
```

or an equivalent asynchronous-disposal pattern, disposal may involve awaiting the result of the async disposal protocol.

That means:

```text
scope exit
   ↓
call async disposer
   ↓
await completion
   ↓
continue propagation
```

The scope is therefore coupled to asynchronous cleanup.

### 10.6 `Symbol.dispose`

A disposable object can define:

```js
[Symbol.dispose]() {}
```

This avoids imposing one universal method name such as:

```js
close()
```

Different domains can continue using their preferred APIs while providing a common protocol for automatic cleanup.

### 10.7 `Symbol.asyncDispose`

Asynchronous cleanup uses:

```js
[Symbol.asyncDispose]()
```

The method may return a promise-like completion.

This lets the language distinguish:

```text
cleanup is immediate
```

from:

```text
cleanup must complete asynchronously
```

### 10.8 Disposable stacks

A disposable stack is an explicit dynamic ownership structure.

A static lexical scope works well when resources are declared directly:

```js
using a = acquireA();
using b = acquireB();
```

A dynamic stack is useful when resources are acquired conditionally or from helper functions:

```js
const stack = new DisposableStack();

stack.use(a);

if (condition) {
  stack.use(b);
}

stack.defer(cleanup);
```

The stack makes the cleanup plan explicit even when acquisition is dynamic.

---

## 11. ECMAScript / Specification Semantics

This chapter requires careful source discipline because explicit resource management has a specification/proposal history and runtime support can vary by environment.

The learner must distinguish:

```text
standardized language semantics
       vs
proposal-stage semantics
       vs
runtime implementation support
```

Do not assume that every JavaScript runtime supports every `using` form merely because the syntax is documented somewhere.

### 11.1 Disposal protocol

The core protocol is represented by well-known symbols:

```js
Symbol.dispose
Symbol.asyncDispose
```

These symbols allow objects to opt into standardized disposal behavior.

### 11.2 Lexical resource lifetime

A resource declaration associates a binding with a disposal obligation for the lifetime of the surrounding scope.

### 11.3 Abrupt completion

Resource cleanup integrates with the broader completion model from Chapter 29.

The important connection is:

```text
normal completion
return
throw
```

can all cause scope exit, and scope exit can trigger disposal.

### 11.4 Suppressed cleanup failures

When both body execution and cleanup fail, the system must preserve meaningful information about both failures rather than blindly discarding one.

Resource-management semantics therefore build directly on the language's abrupt-completion model and multi-error representation ideas.

### 11.5 `DisposableStack`

`DisposableStack` gives dynamic code an explicit stack of cleanup records.

Important conceptual operations include:

- register a disposable;
- register arbitrary cleanup behavior;
- release/dispose the stack;
- transfer ownership;
- prevent accidental cleanup after ownership transfer.

### 11.6 `AsyncDisposableStack`

The asynchronous form extends the same ownership model to cleanup operations that must be awaited.

The exact names and availability of APIs must be verified against the target ECMAScript/runtime version.

### 11.7 Runtime support is part of engineering correctness

A codebase cannot be considered production-ready merely because a language construct has specification-level semantics.

Check:

```text
engine support
runtime version
transpilation requirements
build target
test environment
deployment environment
```

This is especially important for language features that are relatively recent.

---

## 12. Advanced Behavior

### 12.1 Partial initialization

Suppose:

```js
{
  using a = acquireA();
  using b = acquireB();
  using c = acquireC();
}
```

and `acquireC()` throws.

Correct reasoning:

```text
A acquired
B acquired
C failed
↓
C has no acquired resource
B must be disposed
A must be disposed
```

This is a major benefit of scope-based resource management.

Manual cleanup is often where developers accidentally miss these partial-initialization branches.

### 12.2 Conditional acquisition

A dynamic resource set is common:

```js
const stack = new DisposableStack();

stack.use(primary);

if (useCache) {
  stack.use(cache);
}

if (useMetrics) {
  stack.defer(flushMetrics);
}
```

Every successful registration contributes to the cleanup plan.

### 12.3 Deferred cleanup

Sometimes the cleanup behavior is not exposed as a disposable object.

A stack can register:

```js
stack.defer(() => releaseLock(lockId));
```

This turns arbitrary cleanup logic into a managed lifetime.

### 12.4 Ownership transfer

Consider a helper that creates a resource:

```js
function createConnection() {
  return new Connection();
}
```

Who owns it?

The creator?

The caller?

A manager?

A transaction object?

A production API should define the answer.

Ownership transfer means:

```text
create → temporary owner
return → caller becomes owner
```

A resource should not remain registered with two independent cleanup owners.

### 12.5 Double disposal

Good disposable abstractions often make cleanup idempotent:

```js
[Symbol.dispose]() {
  if (this.closed) {
    return;
  }

  this.closed = true;
  release();
}
```

Whether idempotency is guaranteed should be part of the contract.

Do not assume every third-party resource is safe to dispose multiple times.

### 12.6 Use-after-disposal

A disposed resource may be logically invalid:

```js
{
  using connection = createConnection();
}

connection.query("...");
```

In practice, lexical scoping often prevents this exact pattern, but references can escape:

```js
let connection;

{
  using local = createConnection();
  connection = local;
}

connection.query("...");
```

The object still exists as a JavaScript value, but its resource lifetime has ended.

This demonstrates:

> Object reachability and resource validity are different dimensions.

### 12.7 Async cleanup and cancellation

Imagine:

```text
operation starts
   ↓
resource acquired
   ↓
request cancelled
   ↓
operation stops
   ↓
resource must still be cleaned
```

Cancellation should not automatically mean:

```text
cleanup skipped
```

The correct design is often:

```text
cancel work
→ perform required cleanup
→ report cancellation
```

### 12.8 Disposal and transactions

A transaction may expose:

```text
commit
rollback
close
```

These are not automatically interchangeable.

For example:

```text
normal success → commit
exception → rollback
scope exit → release connection
```

A disposable abstraction must not accidentally turn these distinct business states into one generic `dispose()` operation unless that is precisely the intended contract.

### 12.9 Disposal and pooling

Pooling changes ownership semantics.

```text
dispose connection
```

may actually mean:

```text
return connection to pool
```

The abstraction should make the semantics clear.

### 12.10 Disposal and finalization

`FinalizationRegistry` is not a replacement for deterministic disposal.

Finalization is GC-related and timing is not an appropriate contract for ordinary resource release.

Later chapters should connect this distinction to weak references and garbage collection.

---

## 13. Edge Cases

### 13.1 Initializer throws

```js
{
  using resource = acquire();
}
```

If `acquire()` throws, the resource was not successfully acquired.

Do not assume disposal can run on a value that never existed.

### 13.2 Disposer throws

```js
class BadResource {
  [Symbol.dispose]() {
    throw new Error("cleanup failed");
  }
}
```

The cleanup error becomes part of the scope's final failure behavior.

### 13.3 Multiple disposers throw

With:

```text
A disposer throws
B disposer throws
C disposer succeeds
```

the implementation must preserve the multiple-failure semantics according to the resource-management model rather than simply forgetting earlier failures.

### 13.4 Body throws and cleanup throws

This is the most important edge case:

```text
body → Error A
cleanup → Error B
```

The final observable failure must preserve enough context to diagnose both.

### 13.5 Async disposer rejects

Equivalent concern:

```text
body → Error A
async cleanup → rejection B
```

The cleanup failure is part of the final completion path.

### 13.6 Resource returned from helper

A helper may create and return a disposable value:

```js
function makeResource() {
  return new Resource();
}
```

The caller must clearly own the returned value.

### 13.7 Resource escapes scope

Escaping references create lifecycle bugs:

```js
let leaked;

{
  using resource = acquire();
  leaked = resource;
}
```

`leaked` may refer to an already-disposed resource.

### 13.8 Async work outlives resource scope

Danger:

```js
{
  using connection = createConnection();

  startBackgroundTask(() => {
    connection.query(...);
  });
}
```

The background work may execute after disposal.

The resource scope must cover the complete lifetime of dependent work, or the work must take ownership another way.

### 13.9 Detached cleanup

When ownership is transferred:

```text
manager A no longer responsible
manager B now responsible
```

the original cleanup registration must be detached correctly.

### 13.10 Disposal is not rollback

Closing a transaction resource is not necessarily equivalent to rollback.

Model each lifecycle operation explicitly.

---

## 14. Common Misconceptions

### Misconception 1 — “Garbage collection cleans up every resource.”

No.

GC manages memory reachability, not arbitrary external resource semantics.

### Misconception 2 — “`using` is just syntax sugar for `finally`.”

Conceptually related, but the resource protocol, disposal ordering, ownership registration, and multi-error behavior provide more than a trivial textual rewrite.

### Misconception 3 — “Disposal always happens immediately when the object becomes unused.”

No.

Disposal is tied to resource-management semantics and scope, not ordinary reachability.

### Misconception 4 — “`await using` means every resource is async.”

No.

It means the disposal protocol may be asynchronous.

### Misconception 5 — “A disposer can never throw.”

It is ordinary program code.

It can fail.

### Misconception 6 — “Once a resource object exists, it remains usable.”

Not necessarily.

The underlying capability may already have been released.

### Misconception 7 — “Closing and disposing are always the same thing.”

Not necessarily.

The semantics are determined by the API contract.

### Misconception 8 — “Disposable resources should always be classes.”

No.

Objects can implement the disposal symbols without using classes.

### Misconception 9 — “Resource cleanup is only a Node.js concern.”

No.

Browsers also manage resources such as streams, subscriptions, locks, workers, and other host capabilities.

### Misconception 10 — “Async cleanup can be ignored because the process will exit.”

Production systems must not depend on process termination as a cleanup strategy unless the lifecycle model explicitly permits it.

---

## 15. Common Mistakes

### Mistake 1 — Ambiguous ownership

Two layers both believe they own the same resource.

Result:

```text
double dispose
```

or one layer disposes a resource still needed by another.

### Mistake 2 — Resource escape

Returning or storing a disposed resource.

### Mistake 3 — Background work outliving ownership

Starting async work and exiting the resource scope before the work finishes.

### Mistake 4 — Cleanup omission

A manual path forgets one of several resources.

### Mistake 5 — Cleanup ordering bug

Releasing a dependency before a dependent resource.

### Mistake 6 — Swallowing disposal errors

```js
try {
  resource.close();
} catch {}
```

This can hide a real reliability failure.

### Mistake 7 — Treating all cleanup as synchronous

Some APIs require async release.

### Mistake 8 — Using finalization as normal cleanup

Finalization is not deterministic resource management.

### Mistake 9 — Mixing business and lifecycle semantics

For example, treating:

```text
commit
rollback
dispose
```

as interchangeable.

### Mistake 10 — Assuming language support equals production support

Verify target runtimes before adopting modern resource syntax.

---

## 16. Comparison With Related Concepts

| Mechanism | Main purpose | Deterministic? | Typical role |
|---|---|---:|---|
| `using` | Scoped synchronous disposal | Yes | Lexical ownership |
| `await using` | Scoped asynchronous disposal | Yes, with await | Async ownership |
| `try/finally` | General cleanup/control flow | Yes | Universal fallback |
| `DisposableStack` | Dynamic cleanup aggregation | Yes when disposed | Conditional/dynamic resources |
| `AsyncDisposableStack` | Dynamic async cleanup | Yes when awaited/disposed | Async dynamic resources |
| GC | Reclaim unreachable memory | No deterministic timing | Memory management |
| `FinalizationRegistry` | Observe finalization opportunities | No | Secondary cleanup/diagnostics |
| `close()` | Domain-specific release | Depends on caller | Resource API |
| `abort()` | Stop/cancel an operation | Usually explicit | Cancellation |
| Pool return | Reuse resource | Explicit | Connection/resource pooling |

### `using` vs `try/finally`

`try/finally` is more general:

```js
const resource = acquire();

try {
  use(resource);
} finally {
  cleanup(resource);
}
```

`using` gives a protocol and ownership abstraction:

```js
{
  using resource = acquire();
  use(resource);
}
```

The first is universally useful.

The second is more composable when the resource itself follows the disposable protocol.

### `using` vs garbage collection

```text
GC → object lifetime
using → resource ownership lifetime
```

These lifetimes can differ.

### Disposable stack vs lexical declarations

Use lexical resource declarations when ownership is naturally static.

Use a disposable stack when resources are:

- conditional;
- dynamically discovered;
- registered by helper functions;
- composed across multiple branches.

---

## 17. Performance Considerations

Resource management improves reliability, but it is not free.

### 17.1 Disposal has runtime work

Cleanup can involve:

- system calls;
- network operations;
- flushing;
- lock release;
- pool management;
- asynchronous awaits.

Measure resource lifecycle cost in real workloads.

### 17.2 Over-scoping

Holding a resource longer than needed increases:

- concurrency pressure;
- memory;
- connection usage;
- lock contention.

Prefer the smallest scope that satisfies the actual ownership requirement.

### 17.3 Under-scoping

Closing too early can cause retries, reconnects, and expensive reinitialization.

The goal is not:

> “always make scopes as small as possible”

but:

> “match scope to the true dependency lifetime.”

### 17.4 Cleanup storms

Large-scale systems can release thousands of resources at once.

A mass-disposal event can create:

```text
many close operations
→ network pressure
→ CPU pressure
→ scheduler pressure
```

Consider batching and lifecycle staggering where appropriate.

### 17.5 Pooling changes the cost model

A pooled resource may be cheaper to retain briefly than to repeatedly create/destroy.

This is an architectural trade-off, not a blanket rule.

---

## 18. Memory Considerations

Deterministic cleanup can reduce retained external state, but it does not automatically free JavaScript memory.

A disposed object may remain reachable:

```js
let reference;

{
  using resource = createResource();
  reference = resource;
}
```

Now:

```text
resource disposed
resource object still reachable
```

Memory reclamation and resource release are separate.

Disposable stacks can also retain:

- cleanup callbacks;
- resource references;
- closure environments.

Large stacks should therefore have bounded lifetime.

Avoid accidentally registering massive object graphs:

```js
stack.defer(() => use(hugeObject));
```

because the closure can retain `hugeObject` until the stack is disposed.

---

## 19. Security Considerations

Resource lifetime has security implications.

### 19.1 Locks

Failure to release locks can create:

- denial of service;
- deadlocks;
- starvation.

### 19.2 File handles

Leaks can exhaust process limits.

### 19.3 Sockets

Leaked network resources can exhaust connection capacity.

### 19.4 Credentials

A resource object may retain:

- access tokens;
- TLS state;
- database credentials;
- sensitive buffers.

Prompt disposal reduces the lifetime of sensitive capabilities.

### 19.5 Temporary files

Failure to clean temporary files can expose sensitive data.

### 19.6 Resource exhaustion

Attackers can deliberately induce paths that allocate resources but fail to release them.

A robust system must test:

```text
success
failure
timeout
cancellation
partial acquisition
```

not only the happy path.

### 19.7 Cleanup race conditions

Concurrent code can accidentally dispose a resource while another task still uses it.

Ownership must be synchronized with the actual concurrent activity.

---

## 20. Production Usage

### 20.1 Database transactions

A transaction often has semantics like:

```text
begin
  ↓
work
  ↓
success → commit
failure → rollback
  ↓
release connection
```

A disposable abstraction can own the connection while explicit business logic controls commit/rollback.

Do not blindly map `dispose()` to `commit()`.

### 20.2 File processing

Conceptually:

```js
{
  using file = openFile(path);

  process(file);
}
```

The scope clearly communicates:

```text
file belongs to this operation
```

### 20.3 Network connections

A connection may need:

```text
flush
shutdown
close
```

The disposal contract must define which lifecycle behavior is required.

### 20.4 Subscription management

A subscription often has a cleanup operation:

```js
const subscription = observable.subscribe(handler);

try {
  await work();
} finally {
  subscription.unsubscribe();
}
```

A disposable protocol can make this composable.

### 20.5 Locks

Lock ownership should map naturally to a scope:

```text
acquire lock
  ↓
critical section
  ↓
release lock
```

This is one of the clearest examples where deterministic disposal is correctness-critical.

### 20.6 Temporary resources

Temporary directories, files, buffers, or registrations should have clear ownership.

### 20.7 Worker/session lifecycle

When an operation creates a worker/session:

```text
create
use
terminate
```

the owning scope must define when termination happens.

### 20.8 HTTP request scopes

A request may own:

- transaction;
- tracing span;
- subscription;
- temporary storage;
- connection;
- cancellation linkage.

Request completion can act as an ownership boundary.

### 20.9 Production architecture

A mature system should make these relationships explicit:

```text
resource
  ↓
owner
  ↓
scope
  ↓
cleanup mechanism
  ↓
failure policy
  ↓
observability
```

---

## 21. Implementation From Scratch

Before relying on language/runtime resource-management features, implement the concepts manually.

### Stage 1 — Guided synchronous disposable

Implement:

```js
function withResource(resource, fn) {
  try {
    return fn(resource);
  } finally {
    resource[Symbol.dispose]();
  }
}
```

Test:

- normal completion;
- return;
- throw.

### Stage 2 — Partially Guided

Support resource creation inside the helper:

```js
function withResource(create, fn) {
  const resource = create();

  try {
    return fn(resource);
  } finally {
    resource[Symbol.dispose]();
  }
}
```

Now test acquisition failure.

### Stage 3 — No Reference

Build:

```js
class DisposableStack {
  constructor() {
    this.stack = [];
    this.disposed = false;
  }

  use(resource) {}

  defer(fn) {}

  dispose() {}
}
```

Requirements:

- LIFO ordering;
- idempotent stack disposal;
- reject registration after disposal;
- support resource protocol;
- support arbitrary cleanup callbacks.

### Stage 4 — Edge-Case Hardened

Add:

- partial initialization;
- cleanup failures;
- multiple cleanup failures;
- ownership transfer;
- nested stacks;
- arbitrary thrown values;
- invalid resource values.

Think carefully about preserving both primary and cleanup failures.

### Stage 5 — Production Grade

Design:

```js
class ResourceScope {
  register(resource) {}
  defer(fn) {}
  transfer() {}
  dispose() {}
}
```

Then build an async variant:

```js
class AsyncResourceScope {
  register(resource) {}
  defer(fn) {}
  deferAsync(fn) {}
  async dispose() {}
}
```

Requirements:

- deterministic LIFO cleanup;
- sync and async protocol support;
- clear ownership state;
- idempotent disposal;
- cleanup aggregation;
- cancellation-aware usage;
- observability hooks;
- bounded retained state;
- tests for every abrupt exit.

---

## 22. Debugging Exercises

### Exercise 1 — Missing cleanup

Find the bug:

```js
function run() {
  const resource = acquire();

  if (!isValid()) {
    return;
  }

  use(resource);
  resource.close();
}
```

Question:

Which control-flow path leaks the resource?

### Exercise 2 — Partial acquisition

```js
const a = acquireA();
const b = acquireB();
const c = acquireC();
```

Suppose `acquireC()` throws.

How will `a` and `b` be released?

Design two solutions.

### Exercise 3 — Async lifetime

```js
{
  using resource = acquire();

  queueMicrotask(() => {
    resource.use();
  });
}
```

Is the callback guaranteed to run before disposal?

Explain the ordering.

### Exercise 4 — Background task

```js
{
  using connection = acquireConnection();

  startAsyncOperation(connection);
}
```

What ownership bug may exist?

### Exercise 5 — Cleanup masking

```js
try {
  doWork();
} finally {
  cleanup();
}
```

Both throw.

Which failure reaches the caller in the ordinary completion model?

How would a production design preserve both?

### Exercise 6 — Escape

```js
let saved;

{
  using resource = createResource();
  saved = resource;
}

saved.use();
```

Why is this dangerous even though `saved` still references a JavaScript object?

---

## 23. Code Review Exercise

Review:

```js
async function processOrder(order) {
  const connection = await db.acquire();
  const transaction = await connection.begin();

  try {
    await saveOrder(transaction, order);
    await transaction.commit();

    return { ok: true };
  } catch (error) {
    await transaction.rollback();
    throw error;
  } finally {
    connection.close();
  }
}
```

Identify design questions involving:

- acquisition failure;
- transaction ownership;
- commit failure;
- rollback failure;
- asynchronous connection cleanup;
- disposal order;
- pool semantics;
- cancellation;
- duplicate ownership;
- error preservation.

Redesign the lifecycle using explicit ownership.

---

## 24. Interview Questions

### Foundational

1. What is a resource?
2. Why is garbage collection insufficient for many resources?
3. What problem does deterministic disposal solve?
4. What is `Symbol.dispose`?
5. What is `Symbol.asyncDispose`?
6. What does `using` represent?
7. What does `await using` represent?
8. Why is disposal usually LIFO?
9. Can disposal throw?
10. What is `DisposableStack`?

### Intermediate

11. How does `using` compare with `try/finally`?
12. Why can resources escape their intended scope?
13. What is ownership transfer?
14. Why should disposal often be idempotent?
15. What happens if resource acquisition partially succeeds?
16. Why is async cleanup different?
17. What is the role of deferred cleanup callbacks?
18. Why should disposal order reflect dependencies?
19. How can background async work cause use-after-disposal?
20. How does pooling affect disposal semantics?

### Advanced

21. Explain how resource cleanup interacts with abrupt completion.
22. How should multiple disposal failures be represented?
23. How should body failure and cleanup failure coexist?
24. When is `DisposableStack` preferable to lexical `using`?
25. What is the difference between resource validity and object reachability?
26. Why is finalization not a substitute for deterministic cleanup?
27. How would you design a disposable API for a database transaction?
28. How would cancellation interact with resource cleanup?
29. How would you test cleanup under every exit path?
30. What runtime-compatibility concerns apply to modern resource-management features?

### Principal-Level

31. Design a resource ownership model for a large Node.js service.
32. Design request-scoped resource management.
33. How would you prevent double ownership across layers?
34. How would you make cleanup observable without creating noisy logs?
35. How would you preserve the primary failure when cleanup also fails?
36. How should pooled resources expose disposal?
37. How would you handle a resource that requires ordered shutdown?
38. How would you prevent background tasks from outliving their resource owners?
39. What invariants should a production `ResourceScope` maintain?
40. When should a team avoid introducing a new disposable abstraction?

---

## 25. Predict-the-Output Exercises

### Exercise A

```js
class R {
  constructor(name) {
    this.name = name;
  }

  [Symbol.dispose]() {
    console.log(this.name);
  }
}

{
  using a = new R("A");
  using b = new R("B");
  console.log("work");
}
```

Predict:

```text
work
B
A
```

### Exercise B

```js
function f() {
  using r = new Resource();

  return 42;
}
```

Question:

Does disposal happen before the function's caller receives `42`?

Explain in terms of scope exit and completion.

### Exercise C

```js
function f() {
  using r = new Resource();

  throw new Error("boom");
}
```

Question:

Does the resource still get disposed?

Explain.

### Exercise D

```js
{
  using a = new Resource("A");
  using b = new Resource("B");

  throw new Error("work");
}
```

Suppose `b` throws during disposal.

What failure state must be represented?

### Exercise E

```js
{
  using resource = new Resource();

  Promise.resolve().then(() => {
    resource.use();
  });
}
```

Reason carefully about:

```text
scope exit
microtask scheduling
resource disposal
callback execution
```

Do not answer by intuition; derive the ordering.

---

## 26. Mastery Exercises

### Exercise 1 — Disposable API

Implement a resource:

```js
class SocketResource {
  [Symbol.dispose]() {}
}
```

Requirements:

- idempotent disposal;
- reject use after disposal;
- record lifecycle state;
- expose no unsafe internal details.

### Exercise 2 — Async resource

Implement:

```js
class AsyncLock {
  async acquire() {}
  async release() {}
  async [Symbol.asyncDispose]() {}
}
```

Design the ownership rules explicitly.

### Exercise 3 — Dynamic resource scope

Implement:

```js
const scope = new DisposableStack();
```

and support:

- `use`;
- `defer`;
- `move` / ownership transfer;
- `dispose`.

### Exercise 4 — Failure aggregation

Construct a scope where:

```text
main work fails
cleanup A fails
cleanup B succeeds
cleanup C fails
```

Design a representation that preserves the entire failure structure.

### Exercise 5 — Request scope

Design a request-scoped resource manager containing:

```text
database transaction
tracing span
temporary file
subscription
```

Define acquisition and cleanup order.

### Exercise 6 — Cancellation integration

Design:

```text
AbortSignal
+
resource scope
+
async operation
+
cleanup
```

Guarantee that cancellation does not leak resources.

### Exercise 7 — Ownership proof

For a complex API, write down:

```text
Who creates?
Who owns?
Who may transfer?
Who disposes?
When does ownership end?
What if acquisition fails?
What if cleanup fails?
What if work is cancelled?
What if work escapes?
```

Use this as a production design checklist.

---

## 27. Key Takeaways

1. Resource lifetime is not the same as JavaScript object lifetime.
2. Garbage collection does not provide deterministic cleanup for arbitrary external resources.
3. `using` and `await using` connect resource ownership to lexical scope.
4. `Symbol.dispose` defines synchronous disposal behavior.
5. `Symbol.asyncDispose` defines asynchronous disposal behavior.
6. `DisposableStack` supports dynamic cleanup registration and ownership composition.
7. Resource cleanup must run on failure paths as well as success paths.
8. Reverse-order disposal naturally respects many dependency graphs.
9. Partial initialization is a major source of resource leaks in manual code.
10. Cleanup itself can fail.
11. Primary and cleanup failures must be preserved where possible.
12. A disposed object may remain reachable as a JavaScript value.
13. Ownership should be explicit and should normally have one clear owner at a time.
14. Background work must not outlive the resource it depends on unless ownership is transferred.
15. Disposal, close, abort, rollback, and shutdown are different concepts unless the API explicitly equates them.
16. Resource management affects reliability, security, performance, memory, and architecture.
17. `try/finally` remains fundamental and is not obsolete.
18. Modern resource-management syntax is valuable because it makes lifecycle intent visible and composable.
19. Runtime support must be verified before using newer syntax in production.
20. Good resource management answers one core question clearly:

> Who owns this resource, and exactly when does that ownership end?

---

## 28. Concept Connections

### Depends On

- Chapter 12 — Execution Contexts / Execution Model
- Chapter 15 — Objects / Property Semantics
- Chapter 18 — Classes / OOP
- Chapter 20 — Symbols / Well-Known Symbols
- Chapter 26 — Generators / Async Generators
- Chapter 27 — Typed Arrays / Binary Data
- Chapter 29 — Errors / Error Handling

### Builds Toward

- Chapter 31 — Async Fundamentals
- Chapter 32 — ECMAScript Jobs / Promise Reactions
- Chapter 33 — Browser Event Loop
- Chapter 34 — Node Event Loop / libuv
- Chapter 35 — Promises
- Chapter 36 — Async/Await
- Chapter 37 — Cancellation / Abort
- Chapter 38 — Async Iteration / Streaming
- Chapter 39 — Concurrency / Parallelism
- Chapter 45 — Memory / GC
- Chapter 46 — Weak Refs / Finalization
- Chapter 53 — Web Streams / Data Flow
- Chapter 57 — JavaScript Security Engineering
- Chapter 59 — Node Core APIs
- Chapter 60 — Node Streams
- Chapter 61 — Worker Threads / Child Processes / Cluster
- Chapter 62 — Process Lifecycle
- Chapter 63 — Async Context / Diagnostics
- Chapter 78 — Production JS Architecture
- Chapter 83 — Observability
- Chapter 84 — Reliability
- Chapter 85 — Performance
- Chapter 86 — Testing
- Chapter 87 — Deterministic Async Testing
- Chapter 88 — Debugging Methodology
- Chapter 89 — Code Review / Refactoring
- Chapter 98 — Anti-patterns / Failure Modes
- Chapter 100 — Cost Model / Tradeoffs
- Chapter 101 — Real-world Production Scenarios
- Chapter 107 — Job Queue
- Chapter 110 — Production JS Backend
- Chapter 111 — Large-scale JS Platform
- Chapter 121 — System Design
- Chapter 122 — Final Principal JS Project

### Related Concepts

- Ownership
- Scope
- Lifetime
- Abrupt completion
- Error handling
- Cancellation
- Transactions
- Pooling
- Locks
- Streams
- Subscriptions
- Finalization
- Garbage collection
- Async control flow
- Observability
- Resource exhaustion

### Concepts Revisited

This chapter revisits:

- lexical scope;
- objects;
- symbols;
- classes;
- abrupt completion;
- error causes;
- asynchronous cleanup;
- runtime compatibility.

### Why This Chapter Matters Later

Production applications are not made only of memory-managed values.

They interact with systems that have finite and stateful lifetimes:

```text
network
database
filesystem
worker
stream
lock
transaction
subscription
pool
```

Once a program crosses those boundaries, lifecycle correctness becomes first-class architecture.

The central principle is:

> Acquisition without an explicit ownership and cleanup story is an incomplete design.

---

## 29. Completion Criteria

Mark Chapter 30 `[+] Completed` only after the learner can demonstrate all of the following.

### Conceptual Understanding

- [ ] Define resource lifetime.
- [ ] Explain why GC is not deterministic resource cleanup.
- [ ] Explain `using`.
- [ ] Explain `await using`.
- [ ] Explain `Symbol.dispose`.
- [ ] Explain `Symbol.asyncDispose`.
- [ ] Explain `DisposableStack`.
- [ ] Explain reverse-order cleanup.
- [ ] Explain ownership transfer.
- [ ] Explain cleanup failure behavior.

### Predictive Mastery

- [ ] Predict disposal on normal scope exit.
- [ ] Predict disposal on `return`.
- [ ] Predict disposal on `throw`.
- [ ] Predict LIFO cleanup order.
- [ ] Predict partial-initialization cleanup.
- [ ] Predict body + cleanup failure behavior.
- [ ] Predict async cleanup sequencing.
- [ ] Predict use-after-disposal scenarios.

### Implementation

- [ ] Implement a disposable object.
- [ ] Implement `withResource` using `try/finally`.
- [ ] Implement `DisposableStack`.
- [ ] Implement async cleanup.
- [ ] Implement ownership transfer.
- [ ] Implement cleanup-failure preservation.
- [ ] Integrate cancellation with resource lifetime.

### Debugging

- [ ] Detect leaked resources.
- [ ] Detect double disposal.
- [ ] Detect use-after-disposal.
- [ ] Detect background work escaping scope.
- [ ] Detect cleanup masking.
- [ ] Detect incorrect disposal ordering.
- [ ] Detect runtime-support mismatches.

### Production Engineering

- [ ] Design request-scoped resources.
- [ ] Design transaction lifetimes.
- [ ] Design pooled resource semantics.
- [ ] Design lock ownership.
- [ ] Design worker/resource lifetimes.
- [ ] Define cleanup observability.
- [ ] Define failure and cancellation policy.
- [ ] Verify runtime support for deployment targets.

### Interview Readiness

- [ ] Explain why disposal is different from GC.
- [ ] Compare `using` and `try/finally`.
- [ ] Defend LIFO cleanup.
- [ ] Explain body/cleanup error interactions.
- [ ] Design a disposable API.
- [ ] Design a resource boundary for a production service.
- [ ] Defend ownership and lifecycle decisions under failure and cancellation.

### Track A — Core Theory

- [ ] Understand ownership semantics.
- [ ] Understand disposal protocols.
- [ ] Understand lexical resource lifetime.
- [ ] Understand dynamic disposal stacks.
- [ ] Understand abrupt completion integration.

### Track B — Implementation

- [ ] Guided implementation completed.
- [ ] Partially guided implementation completed.
- [ ] No-reference implementation completed.
- [ ] Edge-case hardened implementation completed.
- [ ] Production-oriented implementation reviewed.

### Track C — Interview / Reasoning

- [ ] Completed output prediction.
- [ ] Completed debugging scenarios.
- [ ] Completed code review exercise.
- [ ] Completed resource-lifecycle design exercise.
- [ ] Defended cleanup trade-offs under real production constraints.

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

# Chapter 30 — Revision / Retrieval Record

### Retrieval Prompts

1. Why is GC insufficient for external resource management?
2. What does `using` guarantee conceptually?
3. What are `Symbol.dispose` and `Symbol.asyncDispose`?
4. Why is disposal generally LIFO?
5. What happens when resource acquisition partially fails?
6. What happens when cleanup throws?
7. How should primary and cleanup failures be represented?
8. When is `DisposableStack` better than lexical `using`?
9. What is ownership transfer?
10. Why can an object remain reachable after its resource has been disposed?
11. Why is finalization not deterministic resource management?
12. How can cancellation interact with cleanup?
13. How can async work outlive resource scope?
14. How does pooling change disposal semantics?
15. How would you model database transaction cleanup?
16. What runtime compatibility checks should precede production use of newer syntax?

### Weak Areas

```text
-
-
-
```

### Revision Queue

```text
- [ ] Revisit ownership and lexical scope
- [ ] Revisit disposal ordering
- [ ] Revisit cleanup failure semantics
- [ ] Revisit DisposableStack / dynamic cleanup
- [ ] Revisit async resource lifetime
- [ ] Revisit cancellation + cleanup
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

# Chapter 30 — Canonical References and Source Discipline

For future detailed study of this chapter, use this source hierarchy:

1. ECMAScript specification / standardized language documentation — syntax, disposal protocols, lexical resource semantics, completion behavior, built-in resource-management abstractions, and well-known symbols.
2. TC39 proposal/history material where necessary — only when explaining proposal evolution or distinctions between proposal stages and standardized behavior.
3. JavaScript engine/runtime documentation — implementation and deployment support details.
4. Host runtime documentation — Node.js/browser availability and host-specific resource behavior.
5. Application architecture documentation — ownership conventions, lifecycle contracts, pooling, transactions, observability, cancellation, and operational policies.

Always distinguish:

```text
language semantics
runtime support
host behavior
application conventions
```

Do not present proposal-stage behavior as a universal guarantee without labeling it.

---

# Chapter 30 — Completion Snapshot

```text
Chapter: 30
Title: Resource Management and Cleanup (`using`, `await using`, DisposableStack)
Part: V — Errors / Cleanup
Status: [+] Expanded
Track A: [ ] Core Theory
Track B: [ ] Implementation
Track C: [ ] Interview / Reasoning
Mastery: [ ] Not Mastered
```
