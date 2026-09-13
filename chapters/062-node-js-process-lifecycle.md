\
# Chapter 62 — Node.js Process Lifecycle

> **Curriculum position:** Part XI — Node.js  
> **Previous chapter:** Chapter 61 — Worker Threads, Child Processes, and Cluster  
> **Next chapter:** Chapter 63 — Async Context and Diagnostics  
> **Primary environment:** Node.js 26.x documentation baseline; note version-sensitive behavior explicitly where relevant.

---

## Chapter Mission

Become able to reason about a Node.js process as a **living runtime**, not merely a file that happens to execute JavaScript.

By the end of this chapter, you should be able to answer, from first principles:

- What happens between `node app.js` and the first application statement?
- What makes a Node.js process remain alive?
- Why does Node sometimes exit without an explicit `exit()` call?
- Why can a Promise exist without keeping the process alive?
- What exactly is different about `beforeExit` and `exit`?
- Why is `process.exit()` dangerous in production servers?
- What is the difference between a process receiving `SIGTERM`, a process calling `process.exit()`, and a process crashing from an uncaught exception?
- How should an HTTP service drain requests before termination?
- How do workers, child processes, timers, sockets, database pools, and queue consumers affect shutdown?
- How should a service behave when graceful shutdown fails?
- Why is `uncaughtException` usually a crash signal rather than a recovery mechanism?
- How do readiness, liveness, startup, and termination interact in an orchestrated deployment?
- How do you prove that a process is leaking resources or refusing to die?
- How do you design a shutdown coordinator that remains correct under duplicate signals, races, partial failures, and hard deadlines?

This chapter treats process lifecycle as a **state machine plus resource-ownership problem**.

---

# 1. Learning Objectives

After completing this chapter, you should be able to:

### Theory
- Explain Node.js startup, steady state, draining, termination, and exit.
- Distinguish JavaScript execution from event-loop liveness.
- Explain the relationship between the process object, libuv handles/requests, operating-system signals, and application resources.
- Explain the semantics of `beforeExit`, `exit`, `uncaughtException`, `uncaughtExceptionMonitor`, `unhandledRejection`, `rejectionHandled`, and `warning`.
- Explain `process.exitCode` versus `process.exit()`.
- Explain signal delivery and platform differences.
- Explain why some resources keep a process alive and others do not.
- Explain what “graceful shutdown” means operationally.

### Implementation
- Write an idempotent shutdown coordinator.
- Implement graceful HTTP server shutdown.
- Drain long-running work.
- Shut down workers and child processes.
- Add deadlines and escalation.
- Prevent shutdown logic from recursively triggering itself.
- Produce lifecycle telemetry suitable for production operations.

### Interview / reasoning
- Predict whether a process exits.
- Debug “Node won’t exit” and “Node exited too early.”
- Explain why a Promise alone may not keep Node alive.
- Defend why `process.exitCode` is normally preferable to `process.exit()`.
- Design a shutdown strategy for a multi-resource backend.
- Reason about crash-only recovery and external supervisors.

---

# 2. Prerequisites

You should already understand:

- Chapter 12 — Execution Contexts and Execution Model
- Chapter 29 — Errors and Error Handling
- Chapter 30 — Resource Management and Cleanup
- Chapter 31 — Async Fundamentals
- Chapter 32 — ECMAScript Jobs and Promise Reactions
- Chapter 34 — Node Event Loop and libuv
- Chapter 35 — Promises
- Chapter 36 — Async/Await
- Chapter 37 — Cancellation and Abort
- Chapter 58 — Node.js Architecture
- Chapter 59 — Node Core APIs
- Chapter 60 — Node.js Streams and Backpressure
- Chapter 61 — Worker Threads, Child Processes, and Cluster

Especially important:

> **A Promise represents a result relationship. An active OS/runtime resource represents liveness. They are not the same thing.**

---

# 3. What Is It?

A **Node.js process lifecycle** is the sequence of states and transitions through which one operating-system process moves from launch to termination.

A useful abstraction is:

```text
           ┌────────────┐
           │ Bootstrap  │
           └─────┬──────┘
                 │
                 ▼
           ┌────────────┐
           │   Ready    │
           └─────┬──────┘
                 │
                 ▼
           ┌────────────┐
           │   Active   │
           └─────┬──────┘
                 │
       signal / fatal error /
        admin shutdown /
       natural quiescence
                 │
                 ▼
           ┌────────────┐
           │  Draining  │
           └─────┬──────┘
                 │
        resources settle
                 │
                 ▼
           ┌────────────┐
           │ Terminating│
           └─────┬──────┘
                 │
                 ▼
           ┌────────────┐
           │   Exited   │
           └────────────┘
```

The lifecycle is not a single linear JavaScript function.

It is the interaction of:

```text
Operating system
       │
       ▼
Node process
       │
       ├── V8 JavaScript execution
       ├── Node bootstrap
       ├── libuv event loop
       ├── active handles / requests
       ├── stdio / IPC
       ├── worker threads
       └── child processes
              │
              ▼
        application resources
        ├── HTTP servers
        ├── sockets
        ├── DB pools
        ├── queue consumers
        ├── file watchers
        ├── timers
        └── custom background work
```

The process ends when the runtime reaches a termination condition, or when termination is forced.

---

# 4. Why Does It Exist?

A process lifecycle exists because long-running programs need answers to four operational questions:

1. **How do I start correctly?**
2. **How do I stay alive while useful work exists?**
3. **How do I stop without corrupting work?**
4. **How do I communicate success, failure, and termination cause to the outside world?**

Without an explicit lifecycle model, production services commonly suffer from:

- dropped requests,
- half-written output,
- corrupted state,
- orphaned workers,
- hanging processes,
- duplicated shutdown work,
- misleading exit codes,
- restart loops,
- incomplete telemetry,
- secret/resource leakage,
- failed deployments,
- and undefined behavior after fatal exceptions.

Lifecycle engineering therefore sits at the intersection of:

> **runtime semantics + operating-system process control + application reliability.**

---

# 5. Mental Model

## 5.1 Process = computation + liveness + ownership

Think of a production Node.js process as three layers:

```text
Process
├── Computation
│   └── JavaScript, callbacks, promises, jobs
│
├── Liveness
│   └── things that keep the runtime doing work
│
└── Ownership
    └── resources that must be started, drained, and closed
```

### Computation

A Promise may represent work that has not completed.

### Liveness

A resource such as an active listening server, socket, or ref'ed timer may keep the process alive.

### Ownership

The code that opens a resource should know who is responsible for closing it.

This leads to a practical rule:

> **Every long-lived resource should have a lifecycle owner.**

---

## 5.2 Process state is not the same as application state

A service can be:

```text
Application:
READY

Process:
ALIVE
```

or:

```text
Application:
DRAINING

Process:
ALIVE
```

or:

```text
Application:
CRASHED / UNDEFINED STATE

Process:
ALIVE FOR A SHORT PERIOD
```

Do not infer application health solely from process existence.

---

## 5.3 A better lifecycle state machine

For serious services:

```text
BOOTSTRAPPING
     │
     ▼
INITIALIZING
     │
     ├── failure ─────► FAILED_STARTUP
     │
     ▼
READY
     │
     ▼
ACTIVE
     │
     ├── natural quiescence ─► EXITING
     ├── SIGTERM/SIGINT ─────► DRAINING
     ├── fatal exception ────► CRASH_EXIT
     └── operator action ────► DRAINING
                                 │
                                 ▼
                             TERMINATING
                                 │
                                 ▼
                               EXITED
```

The distinction between `FAILED_STARTUP`, `DRAINING`, and `CRASH_EXIT` is operationally important.

---

# 6. Core Rules

## Rule 1 — Node can terminate without `process.exit()`

Normal shutdown occurs when there is no additional work that keeps the event loop active.

Node's `beforeExit` documentation describes this as the event-loop becoming empty with no additional work scheduled. A `beforeExit` listener can schedule asynchronous work and therefore keep the process alive longer. [`beforeExit`, Node.js process API](https://nodejs.org/api/process.html#event-beforeexit)

## Rule 2 — A Promise is not automatically a liveness handle

This is a classic misconception:

```js
await somePromise;
```

does not mean:

> “The operating system must keep the Node process alive until this Promise resolves.”

Whether the process remains alive depends on what runtime resources are active underneath the asynchronous operation.

## Rule 3 — `process.exit()` is abrupt

`process.exit()` terminates synchronously and can cut short pending asynchronous operations, including writes to stdout and stderr. Node's documentation recommends setting `process.exitCode` for ordinary graceful termination instead. [`process.exit()` and `process.exitCode`](https://nodejs.org/api/process.html#process-exitcode)

## Rule 4 — `exit` is not an async cleanup hook

At `'exit'`, the process is already committed to terminating. The documentation explicitly warns that there is no way to prevent termination at that point. [`exit`, Node.js process API](https://nodejs.org/api/process.html#event-exit)

## Rule 5 — Signals are control inputs, not “magic shutdown functions”

Receiving `SIGTERM` does not inherently mean:

```text
call gracefulShutdown()
```

Your application must establish the signal handling policy.

## Rule 6 — Fatal exceptions are different from operational shutdown

Graceful shutdown assumes the process is still trustworthy enough to coordinate cleanup.

An uncaught exception can mean:

> application invariants are no longer trustworthy.

Node explicitly warns against continuing normal operation after an uncaught exception. [`uncaughtException`, Node.js process API](https://nodejs.org/api/process.html#event-uncaughtexception)

## Rule 7 — Shutdown must be idempotent

You may receive:

```text
SIGTERM
SIGTERM
SIGINT
deployment timeout
manual shutdown
```

all close together.

Your shutdown coordinator must safely converge on one shutdown sequence.

## Rule 8 — Shutdown needs a deadline

Graceful does not mean infinite.

A production process should have:

```text
grace period
    ↓
drain
    ↓
force
```

## Rule 9 — Readiness should fail before termination

When a service enters draining:

```text
readiness = false
accept new traffic = false
finish existing traffic = yes
```

This creates a critical separation between:

- stopping intake,
- draining work,
- terminating resources,
- exiting the process.

---

# 7. Syntax

## 7.1 Process basics

```js
console.log({
  pid: process.pid,
  ppid: process.ppid,
  cwd: process.cwd(),
  argv: process.argv,
  execArgv: process.execArgv,
  execPath: process.execPath,
  platform: process.platform,
  arch: process.arch,
  version: process.version,
  uptime: process.uptime(),
});
```

## 7.2 Exit code

```js
process.exitCode = 1;
```

Prefer this when the program can naturally finish pending work before leaving.

## 7.3 Immediate exit

```js
process.exit(1);
```

Use sparingly and deliberately.

## 7.4 Lifecycle event hooks

```js
process.on('beforeExit', (code) => {
  console.log('beforeExit:', code);
});

process.on('exit', (code) => {
  console.log('exit:', code);
});
```

## 7.5 Signal handlers

```js
process.on('SIGTERM', () => {
  console.log('SIGTERM received');
});

process.on('SIGINT', () => {
  console.log('SIGINT received');
});
```

## 7.6 Fatal-exception observation

```js
process.on('uncaughtExceptionMonitor', (err, origin) => {
  console.error('fatal:', origin, err);
});
```

Use monitoring hooks to observe failures without converting them into a false recovery path.

---

# 8. Basic Examples

## Example 1 — Natural exit

### Code

```js
console.log('start');
```

### Prediction

Does the process stay alive?

### Actual Result

It normally exits after synchronous work finishes because no additional active work is keeping the event loop alive.

### Trace

```text
startup
  ↓
execute module
  ↓
console.log
  ↓
no active work
  ↓
process termination
```

### Rule

> No pending liveness + no required work = natural termination.

---

## Example 2 — A timer can keep the process alive

```js
setInterval(() => {
  console.log('heartbeat');
}, 1000);
```

The interval creates ongoing scheduled work.

To allow the timer not to keep the process alive:

```js
const timer = setInterval(() => {
  console.log('heartbeat');
}, 1000);

timer.unref();
```

Now the timer may continue to exist while not being sufficient by itself to keep the process alive.

---

## Example 3 — Promise misconception

```js
new Promise((resolve) => {
  setTimeout(resolve, 5000);
});
```

The process stays alive here because the timer is active.

Compare:

```js
new Promise(() => {});
```

There is no underlying timer/socket/handle created by this code.

The Promise remains pending forever, but that alone does not create event-loop work that forces process liveness.

### Principal insight

> **Logical pendingness is not equivalent to runtime liveness.**

---

# 9. Execution Walkthrough

## 9.1 From CLI to application

A simplified path:

```text
OS launches node executable
        ↓
Node bootstrap
        ↓
parse runtime / CLI options
        ↓
initialize V8 + Node internals
        ↓
create process object
        ↓
establish runtime services
        ↓
resolve / load application entry point
        ↓
execute application initialization
        ↓
application registers resources
        ↓
event loop services work
```

This is intentionally simplified. Chapter 58 covers Node architecture in more detail.

---

## 9.2 Bootstrap responsibilities

Typical startup concerns include:

- command-line parsing,
- selecting the entry point,
- loading runtime internals,
- environment setup,
- module loading,
- standard streams,
- timers,
- networking facilities,
- diagnostic facilities,
- signal integration,
- application initialization.

Do not treat this as a pure JavaScript stack.

Node is a host runtime that coordinates:

```text
V8
+
Node internals
+
libuv
+
OS
+
application code
```

---

## 9.3 Entry-point initialization

Your application's entry point often does:

```js
load configuration
connect to database
create HTTP server
start queue consumer
start metrics exporter
register signal handlers
mark service ready
```

That means startup order matters.

Bad:

```text
ready=true
DB connection pending
queue not initialized
HTTP accepts traffic
```

Better:

```text
bootstrap
  ↓
config validated
  ↓
DB ready
  ↓
queues ready
  ↓
HTTP listening
  ↓
readiness=true
```

---

# 10. Internal Mechanics

## 10.1 What makes Node stay alive?

A practical mental model is:

```text
Process remains alive while relevant event-loop work exists.
```

Examples include:

- active network servers,
- sockets,
- certain timers,
- file system watchers,
- IPC channels,
- child-process activity,
- other ref'ed libuv-backed resources.

A useful diagnostic API is:

```js
console.log(process.getActiveResourcesInfo());
```

Node documents `process.getActiveResourcesInfo()` as returning information about active resources that are currently keeping the event loop alive. Use it as a diagnostic clue rather than as a complete ownership graph.

---

## 10.2 Handles vs requests

At the libuv level, long-lived resources are often represented by handles, while one-shot operations may use requests.

Conceptually:

```text
handle:
  "I remain registered with the event loop"

request:
  "Perform this operation"
```

Do not overgeneralize this into a JavaScript API guarantee; the implementation boundary is deeper than the public language model.

---

## 10.3 Ref and unref

Some resources can be detached from process liveness.

Conceptually:

```text
ref'ed resource
    ↓
can keep process alive

unref'ed resource
    ↓
does not by itself keep process alive
```

This is useful for:

- background diagnostics,
- idle cleanup,
- periodic metrics,
- cache maintenance,
- optional heartbeat tasks.

Be careful:

> `unref()` is a liveness policy, not a cancellation API.

---

# 11. ECMAScript / Specification Semantics

Node.js process lifecycle is primarily a **host-runtime** concern, not an ECMAScript language feature.

## 11.1 Standards boundary

| Concern | ECMAScript | Node.js | OS/libuv |
|---|---:|---:|---:|
| Promises | ✅ | hosts them | — |
| Jobs / microtasks | ✅ | integrates them | — |
| Process object | ❌ | ✅ | underlying OS |
| SIGTERM | ❌ | ✅ API | ✅ OS |
| PID | ❌ | ✅ | ✅ |
| `process.exit()` | ❌ | ✅ | OS process termination |
| HTTP server | ❌ | ✅ | networking |
| Event-loop liveness | ❌ as Node concept | ✅ | ✅ |

This distinction prevents one of the most common architectural mistakes:

> treating Node behavior as if ECMAScript itself defines it.

---

## 11.2 Promise jobs are not process lifecycle semantics

ECMAScript defines the semantics of Promise reactions and Jobs.

Node decides how the host schedules and services those jobs in conjunction with its event loop.

Therefore:

```text
ECMAScript promise semantics
        ≠
Node process lifetime
```

---

# 12. Advanced Behavior

## 12.1 `beforeExit`

Node emits `'beforeExit'` when the event loop has no additional work to schedule.

Important properties:

- It can be emitted more than once if a listener schedules additional work.
- It is not emitted for explicit termination such as `process.exit()`.
- It is not emitted for certain fatal conditions such as uncaught exceptions.
- Its listener may schedule asynchronous work and prolong the process.

### Example

```js
process.on('beforeExit', (code) => {
  console.log('beforeExit:', code);

  setTimeout(() => {
    console.log('extra work');
  }, 100);
});
```

This can produce a cycle:

```text
event loop empty
      ↓
beforeExit
      ↓
schedule timer
      ↓
timer executes
      ↓
event loop becomes empty again
      ↓
beforeExit
      ↓
...
```

### Anti-pattern

Never treat `beforeExit` as a normal application shutdown orchestrator.

It is primarily a lifecycle observation point with unusual semantics.

---

## 12.2 `exit`

The `'exit'` event occurs when Node is committed to terminating.

The callback is synchronous.

```js
process.on('exit', (code) => {
  // synchronous final accounting only
});
```

Do not do this:

```js
process.on('exit', async () => {
  await database.close();
});
```

The async function does not turn the exit hook into an awaited shutdown phase.

### Correct mental model

```text
exit event
  ↓
synchronous finalization
  ↓
process terminates
```

not:

```text
exit event
  ↓
arbitrary async cleanup
  ↓
wait
  ↓
terminate
```

---

# 13. Uncaught Exceptions and Fatal Failure

## 13.1 `uncaughtException`

An uncaught exception has escaped normal error boundaries and reached the process-level failure boundary.

Node's current documentation says that the event should be used only as a last resort and explicitly warns that resuming normal operation after an uncaught exception is unsafe. The documented role of such handling is limited cleanup before termination. [Node.js `uncaughtException` docs](https://nodejs.org/api/process.html#event-uncaughtexception)

### Bad

```js
process.on('uncaughtException', (err) => {
  console.error(err);
  // keep accepting requests
});
```

### Better

```js
process.on('uncaughtException', (err, origin) => {
  logFatalSync(err, origin);
  process.exitCode = 1;
});
```

In a production service, the stronger design is usually:

```text
fatal error
   ↓
record useful diagnostics
   ↓
stop normal work
   ↓
attempt bounded cleanup
   ↓
exit
   ↓
external supervisor restarts
```

---

## 13.2 `uncaughtExceptionMonitor`

The monitor event runs before the normal uncaught-exception hook and does not itself change crash behavior.

This makes it appropriate for:

- synchronous crash logging,
- diagnostics,
- fatal counters,
- breadcrumbs.

It is not a recovery mechanism.

---

## 13.3 `unhandledRejection`

A rejected Promise that remains unhandled participates in Node's unhandled-rejection policy.

The current Node documentation describes the process-level events in terms of the list of unhandled rejections growing and shrinking, and documents default handling through the `--unhandled-rejections` policy. [Node.js process events](https://nodejs.org/api/process.html#process-events)

Useful lifecycle reasoning:

```text
Promise rejects
    ↓
no handler observed in time
    ↓
unhandled-rejection policy applies
    ↓
may result in uncaught-exception behavior
    ↓
process may terminate
```

Do not build business logic around subtle timing of process-level unhandled rejection events.

---

## 13.4 `rejectionHandled`

This event indicates that a previously unhandled rejection has become handled later.

It is useful for diagnostics and tracking, not for core application control flow.

---

# 14. Exit Codes

## 14.1 Meaning

An exit code communicates process outcome to the parent environment.

```text
0
  success

non-zero
  failure or special termination
```

In automation:

```bash
node app.js
echo $?
```

Exit codes are part of the contract between your process and:

- shells,
- CI,
- supervisors,
- containers,
- job runners,
- orchestration platforms.

---

## 14.2 `process.exitCode`

Preferred graceful pattern:

```js
try {
  await main();
} catch (err) {
  console.error(err);
  process.exitCode = 1;
}
```

Then allow the runtime to finish required pending I/O.

---

## 14.3 `process.exit()`

```js
process.exit(1);
```

This is a forceful termination instruction.

Why dangerous:

```text
pending log write
pending metric
pending response
pending file write
pending cleanup
        ↓
process.exit()
        ↓
work may be cut short
```

Use it only when the termination semantics are intentionally immediate.

---

# 15. Signals

## 15.1 What is a signal?

A signal is an operating-system process-control mechanism.

Common operational signals:

- `SIGINT` — common interactive interrupt
- `SIGTERM` — conventional graceful-termination request
- `SIGHUP` — historically terminal hangup; often repurposed for reload/config behavior
- `SIGUSR1` / `SIGUSR2` — application-specific conventions on Unix-like systems

`SIGKILL` is fundamentally different:

> it cannot be intercepted for graceful cleanup.

Node's documentation also notes important Windows differences: Windows does not implement POSIX signals in the same way and provides partial emulation. [Node.js signal events](https://nodejs.org/api/process.html#signal-events)

---

## 15.2 `process.kill()` naming trap

```js
process.kill(pid, 'SIGTERM');
```

The name `kill` is misleading if interpreted as “guaranteed termination.”

It means:

> **send a signal to a process.**

Node's child-process documentation explicitly notes that `subprocess.kill()` sends a signal and may not actually terminate the target process. [Node.js child_process docs](https://nodejs.org/api/child_process.html)

---

## 15.3 Signal handling pattern

```js
let shuttingDown = false;

async function shutdown(reason) {
  if (shuttingDown) return;
  shuttingDown = true;

  console.log(`shutdown requested: ${reason}`);

  // stop intake
  // drain work
  // close dependencies
  // set final exit code
}

process.once('SIGTERM', () => {
  void shutdown('SIGTERM');
});

process.once('SIGINT', () => {
  void shutdown('SIGINT');
});
```

The `once` plus idempotent guard combination is defensive programming, not redundancy.

---

# 16. Graceful Shutdown Architecture

A production shutdown sequence should be explicit.

```text
Signal / admin command
          │
          ▼
   transition to draining
          │
          ▼
     stop intake
          │
          ▼
   fail readiness checks
          │
          ▼
   stop new background work
          │
          ▼
   drain active work
          │
          ├──────────────┐
          ▼              ▼
 close HTTP          stop workers
 servers             / children
          │              │
          └──────┬───────┘
                 ▼
          close DB / cache
             / queues
                 │
                 ▼
          flush telemetry
                 │
                 ▼
           set exit code
                 │
                 ▼
               exit
```

---

# 17. HTTP Server Shutdown

## 17.1 Stop accepting new connections

For a Node HTTP server:

```js
await new Promise((resolve, reject) => {
  server.close((err) => {
    if (err) reject(err);
    else resolve();
  });
});
```

Node documents `server.close()` as stopping acceptance of new connections while allowing existing connections to continue until they are closed. [Node.js `net.Server.close()`](https://nodejs.org/api/net.html#serverclosecallback)

---

## 17.2 Drain in-flight requests

Stopping acceptance is not the same as finishing current business operations.

You need:

```text
new request
     ↓
reject / no longer accepted

existing request
     ↓
finish safely

long request
     ↓
deadline
     ↓
force close if required
```

---

## 17.3 Current HTTP server helpers

Node versions include additional server-closing helpers such as:

- `closeIdleConnections()`
- `closeAllConnections()`

Use them with explicit semantics and version awareness. `closeAllConnections()` is a forceful tool; it can terminate active connections and should not be confused with graceful draining.

Check the current Node HTTP documentation before depending on exact timing or connection semantics. [Node.js HTTP API](https://nodejs.org/api/http.html)

---

# 18. Graceful Shutdown of Other Resources

## 18.1 Database pool

Desired sequence:

```text
stop new queries
    ↓
wait for active queries
    ↓
close idle connections
    ↓
terminate pool
```

A database library may expose its own shutdown contract.

Do not assume:

```js
db.close()
```

means the same thing for every driver.

---

## 18.2 Queue consumer

A queue worker should generally:

```text
stop receiving new jobs
      ↓
ack / finish current jobs
      ↓
retry or release unfinished work
      ↓
close broker connection
```

The correct behavior depends on delivery semantics:

- at-most-once,
- at-least-once,
- effectively-once with deduplication,
- transactional outbox,
- explicit lease/visibility timeout.

Shutdown policy must respect the queue's delivery model.

---

## 18.3 Timers

Background timers should be:

- tracked,
- cancellable,
- cleared during shutdown,
- optionally `unref()`ed when intentionally non-liveness-critical.

---

## 18.4 Workers

From Chapter 61:

```js
await worker.terminate();
```

Modern Node exposes asynchronous worker termination. [Node.js Worker Threads API](https://nodejs.org/api/worker_threads.html#workerterminate)

For controlled shutdown, prefer a protocol when possible:

```text
parent:
  stop accepting jobs

worker:
  finish current job
  acknowledge drain
  exit
```

Use forced termination when the worker exceeds its shutdown deadline.

---

## 18.5 Child processes

A child process may need:

```text
stop input
↓
send graceful signal
↓
wait for exit
↓
force kill if deadline exceeded
```

Remember:

> signaling a child is not proof that it has exited.

Node exposes child-process lifecycle events such as `'exit'` and `'close'`, and `subprocess.kill()` sends a signal rather than guaranteeing termination. [Node.js child_process API](https://nodejs.org/api/child_process.html)

---

# 19. Shutdown Deadlines

Never implement:

```js
await closeEverything();
```

without considering the possibility that something never resolves.

Instead:

```js
const shutdownDeadline = new Promise((_, reject) => {
  setTimeout(() => {
    reject(new Error('shutdown deadline exceeded'));
  }, 10_000);
});

await Promise.race([
  closeEverything(),
  shutdownDeadline,
]);
```

Better still, each resource should accept cancellation or an AbortSignal where supported.

---

## 19.1 Escalation model

```text
T0
│
├─ mark draining
├─ readiness=false
├─ stop intake
│
T1
│
├─ graceful close
├─ drain workers
├─ flush telemetry
│
T2 = deadline
│
├─ log forced shutdown
├─ close remaining resources
└─ terminate non-cooperative resources
```

A healthy shutdown should finish before `T2`.

---

# 20. Race Conditions During Shutdown

Shutdown is concurrent.

Potential race:

```text
SIGTERM
    ↓
shutdown begins

SIGINT
    ↓
shutdown begins again
```

Potential race:

```text
request starts
    ↓
SIGTERM
    ↓
server begins closing
    ↓
handler attempts DB query
    ↓
DB already closed
```

Potential race:

```text
worker drain starts
    ↓
queue closes first
    ↓
worker cannot finish current job
```

Therefore shutdown ordering matters.

---

# 21. Dependency Ordering

Model resources as a graph:

```text
HTTP intake
    ↓
application service
    ↓
queue / database
    ↓
network / OS
```

Shutdown normally happens approximately in reverse dependency order:

```text
stop intake
   ↓
stop accepting work
   ↓
drain application work
   ↓
close databases / queues
   ↓
close infrastructure
```

This is analogous to destroying a dependency graph in reverse topological order.

---

# 22. Idempotent Shutdown Coordinator

## 22.1 Minimal production skeleton

```js
class ShutdownCoordinator {
  #started = false;
  #reason = null;
  #promise = null;

  async shutdown(reason, exitCode = 0) {
    if (this.#promise) {
      return this.#promise;
    }

    this.#reason = reason;

    this.#promise = this.#run(reason, exitCode);
    return this.#promise;
  }

  async #run(reason, exitCode) {
    if (this.#started) return;

    this.#started = true;

    try {
      console.log(`shutdown started: ${reason}`);

      await this.#stopIntake();
      await this.#drain();
      await this.#closeResources();

      process.exitCode = exitCode;

      console.log('shutdown completed');
    } catch (error) {
      console.error('shutdown failed', error);
      process.exitCode = 1;
    }
  }

  async #stopIntake() {}
  async #drain() {}
  async #closeResources() {}
}
```

The crucial property is:

```text
first shutdown call owns execution
subsequent shutdown calls await the same promise
```

---

# 23. Production-Grade Shutdown Coordinator

A stronger version records:

- start time,
- reason,
- signal,
- phase,
- deadline,
- failures,
- resources closed,
- final exit code.

```js
class ShutdownCoordinator {
  #promise;
  #state = 'ACTIVE';
  #deadlineMs;

  constructor({ deadlineMs = 10_000 } = {}) {
    this.#deadlineMs = deadlineMs;
  }

  get state() {
    return this.#state;
  }

  shutdown(reason, exitCode = 0) {
    if (this.#promise) return this.#promise;

    this.#promise = this.#execute(reason, exitCode);
    return this.#promise;
  }

  async #execute(reason, exitCode) {
    this.#state = 'DRAINING';

    const controller = new AbortController();
    const timer = setTimeout(
      () => controller.abort(new Error('shutdown deadline exceeded')),
      this.#deadlineMs,
    );

    try {
      await this.#stopIntake({ signal: controller.signal });
      await this.#drain({ signal: controller.signal });
      await this.#closeResources({ signal: controller.signal });

      this.#state = 'TERMINATING';
      process.exitCode = exitCode;
    } catch (error) {
      this.#state = 'TERMINATING';

      console.error({
        phase: this.#state,
        reason,
        error,
      });

      process.exitCode = 1;
    } finally {
      clearTimeout(timer);
      this.#state = 'EXITED';
    }
  }

  async #stopIntake() {}
  async #drain() {}
  async #closeResources() {}
}
```

Production implementations should also consider:

- per-resource timeouts,
- partial failures,
- cleanup ordering,
- metrics flush guarantees,
- forced worker termination,
- repeated signal handling,
- shutdown cause preservation.

---

# 24. Readiness, Liveness, and Startup

These concepts are related but distinct.

## 24.1 Startup probe

Question:

> Has this process finished initialization enough to participate?

Example:

```text
configuration loaded
database connected
server initialized
```

---

## 24.2 Readiness probe

Question:

> Should traffic be sent here right now?

During draining:

```text
process = alive
readiness = false
```

That is correct.

---

## 24.3 Liveness probe

Question:

> Is the process fundamentally alive, or should an external supervisor restart it?

Do not make liveness depend on:

> “Have all requests finished in the last 100 ms?”

That can create restart loops during legitimate load.

---

# 25. Orchestrated Termination

A common production termination timeline:

```text
orchestrator
   │
   │ stop request
   ▼
process receives SIGTERM
   │
   ├── readiness → false
   ├── stop intake
   ├── drain
   └── close resources
           │
           ▼
       normal exit
```

If the deadline expires:

```text
grace period exceeded
        ↓
forced termination
```

This makes shutdown deadline part of your service contract.

---

# 26. Graceful vs Crash-Only Design

## Graceful shutdown

Optimized for expected operational termination:

- deployment,
- scale down,
- node maintenance,
- manual stop,
- failover.

## Crash-only design

Optimized for unexpected corruption or invariant violation:

```text
fatal failure
   ↓
do not trust process state
   ↓
terminate
   ↓
external supervisor restarts
```

The two approaches coexist:

```text
expected stop → graceful drain
fatal fault   → bounded cleanup + exit
```

---

# 27. Why `uncaughtException` Is Not a Restart Mechanism

Suppose:

```js
let cache = new Map();

process.on('uncaughtException', (err) => {
  console.error(err);
});
```

The handler runs, but your process may now contain:

```text
partially mutated state
broken invariants
unknown transaction status
stale locks
half-completed I/O
inconsistent caches
```

Continuing requests can amplify the damage.

Use an external supervisor:

```text
Node process
    │
    ▼
failure
    │
    ▼
exit
    │
    ▼
supervisor
    │
    ▼
new process
```

This is a core reliability boundary.

---

# 28. `beforeExit` Edge Case: The Infinite Revival

```js
process.on('beforeExit', () => {
  setTimeout(() => {
    console.log('revived');
  }, 0);
});
```

Possible lifecycle:

```text
no work
  ↓
beforeExit
  ↓
schedule timer
  ↓
timer runs
  ↓
no work
  ↓
beforeExit
  ↓
...
```

### Lesson

A lifecycle hook can become a source of liveness itself.

Never casually schedule recurring asynchronous work in `beforeExit`.

---

# 29. `exit` Edge Case: Async Cleanup Is Too Late

```js
process.on('exit', async () => {
  await saveMetrics();
});
```

This is not a reliable shutdown strategy.

The process is already in the termination phase.

Correct pattern:

```text
signal
 ↓
graceful shutdown function
 ↓
await saveMetrics()
 ↓
return
 ↓
process exits
```

not:

```text
process exits
 ↓
try to save metrics
```

---

# 30. Startup Failure

Suppose:

```js
await connectDatabase();
server.listen(3000);
```

and database startup fails.

A correct process should usually:

```text
log startup failure
set non-zero outcome
close any partially initialized resources
do not mark ready
exit
```

Avoid entering `ACTIVE` state when initialization invariants are not satisfied.

---

# 31. Partial Startup Rollback

Initialization can fail after several resources have started:

```text
config ✅
logger ✅
database ✅
cache ✅
queue ❌
http not started
```

Rollback should close the successfully initialized resources:

```text
queue failure
   ↓
close cache
close DB
flush logs
exit non-zero
```

This is one reason resource registries and structured cleanup are useful.

---

# 32. Resource Registry

A simple pattern:

```js
const resources = [];

async function register(resource) {
  await resource.start();
  resources.push(resource);
}

async function shutdownAll() {
  for (const resource of [...resources].reverse()) {
    await resource.stop();
  }
}
```

This encodes:

> start in dependency order, stop in reverse order.

For more advanced cleanup semantics, connect this chapter to Chapter 30's `DisposableStack` / `using` concepts.

---

# 33. Resource Management and `using`

The standardized JavaScript resource-management model can help manage scoped resources, but a process-wide shutdown sequence is often broader than a lexical scope.

Conceptually:

```js
await using resource = await openResource();
```

is useful for:

```text
lexically scoped resource
```

Whereas:

```text
process receives SIGTERM
       ↓
global lifecycle coordinator
       ↓
many independent resource owners
```

is a process orchestration problem.

### Important distinction

> `using` can help clean up resources; it does not replace service shutdown architecture.

---

# 34. Process Metadata

Useful operational fields include:

```js
const lifecycleSnapshot = {
  pid: process.pid,
  ppid: process.ppid,
  uptimeSeconds: process.uptime(),
  platform: process.platform,
  arch: process.arch,
  nodeVersion: process.version,
  execPath: process.execPath,
  cwd: process.cwd(),
  argv: process.argv,
  execArgv: process.execArgv,
  exitCode: process.exitCode,
};
```

Avoid logging secrets from:

```js
process.env
```

without an explicit redaction policy.

---

# 35. Diagnostics

Useful APIs and techniques:

```js
process.getActiveResourcesInfo();
process.memoryUsage();
process.resourceUsage();
process.uptime();
process.pid;
process.report.getReport();
```

The process API also exposes diagnostic report functionality. For complex incidents, combine lifecycle telemetry with:

- heap metrics,
- CPU usage,
- event-loop delay,
- active-resource information,
- open connection counts,
- queue depth,
- worker counts,
- child-process state.

---

# 36. Debugging “Node Won’t Exit”

Use this investigation sequence.

## Step 1 — Prove the application reached shutdown

```js
console.log('shutdown requested');
```

## Step 2 — Record current phase

```text
ACTIVE
DRAINING
TERMINATING
```

## Step 3 — Inspect active resources

```js
console.log(process.getActiveResourcesInfo());
```

## Step 4 — Search for common liveness sources

- `setInterval`
- timers
- sockets
- HTTP servers
- file watchers
- child processes
- worker threads
- IPC channels
- open database clients

## Step 5 — Check ownership

Ask:

> Who created this resource, and who is responsible for closing it?

## Step 6 — Check `ref` / `unref`

A timer or channel may be intentionally or accidentally keeping the process alive.

---

# 37. Debugging “Node Exits Too Early”

Check:

1. Did the async operation have a real runtime resource underneath it?
2. Was a timer/socket/server unref'ed?
3. Did the caller forget to await an operation whose implementation also lacks a liveness handle?
4. Did startup throw and set failure state?
5. Was `process.exit()` called?
6. Did the parent process or supervisor terminate the process?
7. Was the process terminated by signal?
8. Did an uncaught exception occur?

---

# 38. Debugging SIGTERM That Appears to Be Ignored

Potential causes:

```text
SIGTERM sent
   ↓
custom handler installed
   ↓
handler begins async shutdown
   ↓
shutdown promise never settles
```

Or:

```text
handler exists
   ↓
resource won't close
   ↓
process stays alive
```

Or:

```text
Windows environment
   ↓
POSIX assumptions do not apply identically
```

Check:

- signal received log,
- shutdown start timestamp,
- shutdown phase,
- resource close completion,
- deadline timer,
- final exit code,
- supervisor behavior.

---

# 39. Debugging Hanging HTTP Shutdown

Symptoms:

```text
SIGTERM received
server.close() called
process never exits
```

Possible causes:

- keep-alive connections,
- WebSockets,
- long-running requests,
- unclosed sockets,
- a second server,
- another active resource unrelated to HTTP.

A correct sequence:

```text
readiness=false
↓
server stops accepting new work
↓
existing work drains
↓
idle/active connection policy applied
↓
server closes
↓
remaining resources close
```

---

# 40. Debugging Workers That Refuse to Die

Symptoms:

```text
HTTP server closed
DB closed
process still alive
```

Possible cause:

```text
Worker thread still alive
```

Or:

```text
Child process still alive
```

Inspect lifecycle events and make worker ownership explicit.

For workers:

```js
await worker.terminate();
```

For children:

```js
child.kill('SIGTERM');
await once(child, 'exit');
```

Then enforce a deadline.

---

# 41. Common Misconceptions

## Misconception 1

> “If there is a pending Promise, Node cannot exit.”

False.

## Misconception 2

> “`process.exit()` is just the synchronous version of normal shutdown.”

False.

It can cut off pending I/O.

## Misconception 3

> “`beforeExit` is where all cleanup belongs.”

False.

Expected shutdown should have an explicit coordinator.

## Misconception 4

> “The `exit` event can await cleanup.”

False.

## Misconception 5

> “SIGTERM always kills a process.”

False.

It is a signal, not a universal immediate-termination guarantee.

## Misconception 6

> “`process.kill()` always kills.”

False.

It sends a signal.

## Misconception 7

> “An `uncaughtException` handler makes the process safe.”

False.

It can make an unsafe process appear alive.

## Misconception 8

> “Closing the HTTP server means the service is fully shut down.”

False.

DB pools, workers, queues, timers, sockets, telemetry, and other resources may remain.

---

# 42. Common Mistakes

## Mistake 1 — Shutdown code scattered everywhere

Bad:

```js
process.on('SIGTERM', closeDb);
process.on('SIGINT', closeQueue);
server.on('close', closeMetrics);
```

This makes ordering hard to reason about.

Prefer one lifecycle coordinator.

## Mistake 2 — Duplicate shutdown sequences

```js
process.on('SIGTERM', () => shutdown());
process.on('SIGINT', () => shutdown());
```

without idempotency.

## Mistake 3 — No timeout

A single hung resource can block the entire deployment.

## Mistake 4 — Mark readiness true too early

A service should not claim readiness before required dependencies are usable.

## Mistake 5 — Logging asynchronously after `process.exit()`

Logs can be lost.

## Mistake 6 — Recovering from fatal state

Continuing service traffic after an uncaught exception is risky by design.

---

# 43. Comparison With Related Concepts

| Concept | Primary question |
|---|---|
| Event loop | What work can execute next? |
| Event-loop liveness | What keeps the process alive? |
| Promise | What async result relationship exists? |
| `beforeExit` | Is Node otherwise out of work? |
| `exit` | Is termination committed? |
| Signal | Did the OS/runtime receive an external control request? |
| `process.exitCode` | What result should normal termination report? |
| `process.exit()` | Should termination happen immediately? |
| graceful shutdown | Can expected termination preserve useful work? |
| crash-only | Is the process too compromised to continue safely? |
| readiness | Should traffic come here? |
| liveness | Should an external system restart this process? |
| startup | Is initialization complete? |

---

# 44. Performance Considerations

Lifecycle code is mostly low-frequency control-plane code, but mistakes can affect the whole service.

## 44.1 Avoid O(N) surprises in request draining

Track active requests explicitly when needed.

## 44.2 Avoid unbounded cleanup

A shutdown should have bounded time.

## 44.3 Do not flush massive buffers synchronously

Prefer bounded, intentional telemetry flushing.

## 44.4 Avoid expensive per-signal diagnostics

Signals may arrive unexpectedly or repeatedly.

## 44.5 Avoid polling when events exist

Prefer:

```js
await once(resource, 'close');
```

over tight polling.

---

# 45. Memory Considerations

Shutdown bugs often reveal memory leaks.

Example:

```js
const subscriptions = new Set();

function start() {
  subscriptions.add(createSubscription());
}

function stop() {
  // forgotten deletion
}
```

Across repeated application lifecycle tests, stale references accumulate.

Track:

- listeners,
- timers,
- sockets,
- request registries,
- workers,
- child processes,
- cache references.

---

# 46. Security Considerations

## 46.1 Secrets during shutdown

Do not dump:

```js
process.env
```

into fatal logs.

## 46.2 Signal trust

Signals and administrative controls should be treated according to your deployment/security model.

## 46.3 Privilege boundaries

A process with elevated privilege has more impact if compromised during recovery or shutdown.

## 46.4 Incomplete cleanup

A crashed process may leave:

- temporary files,
- locks,
- socket files,
- shared-memory resources,
- partial data.

Design restart behavior assuming cleanup may be incomplete.

## 46.5 Child-process termination

Be aware of process trees and shells. Sending a signal to a parent does not universally guarantee all descendants are gone.

Node documents scenarios where shell-created descendants can outlive the immediate child process. [Node.js child_process API](https://nodejs.org/api/child_process.html)

---

# 47. Reliability Considerations

A production lifecycle should support:

- duplicate signals,
- partial failures,
- bounded shutdown,
- observable phase transitions,
- dependency ordering,
- safe defaults,
- supervisor restart,
- forced escalation.

A useful invariant:

> **Shutdown must converge.**

That means repeated shutdown requests, resource errors, and slow dependencies should still lead to a final bounded outcome.

---

# 48. Production Usage Pattern

A strong service bootstrap often resembles:

```js
import http from 'node:http';
import process from 'node:process';

const server = http.createServer(async (req, res) => {
  res.end('ok');
});

let state = 'BOOTSTRAPPING';
let shuttingDown = false;

async function start() {
  await initializeDatabase();
  await initializeQueue();

  await new Promise((resolve, reject) => {
    server.listen(3000, (err) => {
      if (err) reject(err);
      else resolve();
    });
  });

  state = 'READY';
}

async function shutdown(reason) {
  if (shuttingDown) return;
  shuttingDown = true;
  state = 'DRAINING';

  try {
    markNotReady();

    await closeHttpServer(server);
    await drainQueue();
    await closeQueue();
    await closeDatabase();

    state = 'TERMINATING';
  } catch (error) {
    console.error('shutdown failure', error);
    process.exitCode = 1;
  }
}

process.once('SIGTERM', () => {
  void shutdown('SIGTERM');
});

process.once('SIGINT', () => {
  void shutdown('SIGINT');
});

try {
  await start();
} catch (error) {
  console.error('startup failure', error);
  process.exitCode = 1;
}
```

The details vary by application, but the structure is broadly reusable.

---

# 49. Production-Grade Shutdown Checklist

Before calling a service “production-ready,” verify:

```text
[ ] startup has explicit phases
[ ] readiness is separate from process existence
[ ] SIGTERM is handled
[ ] SIGINT is handled where appropriate
[ ] shutdown is idempotent
[ ] new traffic stops before dependencies close
[ ] in-flight requests can drain
[ ] background consumers stop taking new work
[ ] workers terminate
[ ] child processes terminate
[ ] DB pools close
[ ] queues close
[ ] timers are cleared or intentionally unref'ed
[ ] telemetry has a bounded flush strategy
[ ] shutdown has a hard deadline
[ ] repeated shutdown requests are safe
[ ] fatal exceptions do not resume normal traffic
[ ] exit code is meaningful
[ ] shutdown reason is observable
[ ] process liveness can be diagnosed
[ ] restart behavior is tested
```

---

# 50. Implementation From Scratch

## Stage A — Guided

Build a `LifecycleManager` supporting:

```text
BOOTSTRAPPING
READY
DRAINING
TERMINATING
EXITED
```

Requirements:

- `start()`
- `shutdown(reason)`
- idempotency
- exit code
- lifecycle logging

---

## Stage B — Partially Guided

Add:

- HTTP server
- timer
- fake DB pool
- fake queue worker
- worker thread

Implement reverse dependency shutdown.

---

## Stage C — No Reference

Design:

```js
createLifecycleManager({
  deadlineMs,
  resources,
  logger,
  readinessController,
});
```

The caller should be able to:

```js
await lifecycle.start();
await lifecycle.shutdown('SIGTERM');
```

without knowing implementation order.

---

## Stage D — Edge-Case Hardened

Your implementation must handle:

- duplicate signals,
- shutdown while startup is incomplete,
- one resource failing to close,
- resource close timeout,
- worker hanging,
- child process surviving parent,
- readiness changing during shutdown,
- repeated `shutdown()` calls,
- final exit code selection.

---

## Stage E — Production Grade

Add:

- structured logs,
- metrics,
- shutdown phase durations,
- trace/span termination policy,
- deadline escalation,
- test hooks,
- deterministic tests,
- operational documentation.

---

# 51. Debugging Exercises

## Exercise 1 — The Pending Promise

```js
new Promise(() => {});
console.log('done');
```

### Question

Will Node stay alive forever?

Explain why or why not in terms of liveness rather than Promise state.

---

## Exercise 2 — The Timer Leak

```js
function startMetrics() {
  setInterval(() => {
    console.log('metrics');
  }, 1000);
}

startMetrics();
```

### Task

Find why the process refuses to exit.

Then redesign it for:

```text
start()
stop()
```

---

## Exercise 3 — `beforeExit` Loop

```js
process.on('beforeExit', () => {
  setImmediate(() => {
    console.log('again');
  });
});
```

### Task

Predict the lifecycle and explain why this is dangerous.

---

## Exercise 4 — Abrupt Exit

```js
console.log('important output');
process.exit(1);
```

### Task

Explain why the statement may not provide the same guarantees as:

```js
console.log('important output');
process.exitCode = 1;
```

---

## Exercise 5 — SIGTERM Hang

```js
process.on('SIGTERM', async () => {
  await closeDatabase();
});
```

### Task

What happens if `closeDatabase()` never settles?

Design a deadline.

---

## Exercise 6 — Double Shutdown

```js
process.on('SIGTERM', () => void shutdown());
process.on('SIGINT', () => void shutdown());
```

### Task

Prove whether the implementation is safe when both signals arrive within 20 ms.

---

## Exercise 7 — HTTP Drain

A server receives:

```text
SIGTERM
request A = 20ms
request B = 8s
request C = WebSocket
```

Your grace deadline is 5s.

### Task

Design the exact sequence of actions.

---

## Exercise 8 — Worker Drain

A worker is processing a 30-second task and receives shutdown during deployment.

### Task

Design:

```text
graceful completion
vs
cancellation
vs
forced termination
```

and explain the data-consistency implications.

---

## Exercise 9 — Fatal Exception

A request handler throws an uncaught exception after mutating an in-memory cache and before committing a DB transaction.

### Task

Argue whether the process should continue serving traffic.

---

## Exercise 10 — Orphaned Child

A Node process launches:

```text
parent
  └── shell
       └── child app
```

The parent sends `SIGTERM` to the immediate child.

### Task

Explain how the descendant can survive and how you would harden the shutdown design.

---

# 52. Code Review Exercise

Review:

```js
const server = app.listen(3000);

process.on('SIGTERM', async () => {
  await db.close();
  await queue.close();
  process.exit(0);
});

process.on('uncaughtException', async (err) => {
  console.error(err);
  await db.close();
});
```

Identify at least ten production problems.

Expected areas:

- readiness transition,
- HTTP intake,
- in-flight requests,
- idempotency,
- `process.exit()`,
- deadline,
- uncaught-exception semantics,
- async cleanup on fatal path,
- child/worker handling,
- telemetry,
- exit code,
- ordering,
- repeated signals.

---

# 53. Interview Questions

## Foundation

1. What causes a Node.js process to exit naturally?
2. Does a pending Promise keep Node alive?
3. What is `process.exitCode`?
4. What is the difference between `process.exit()` and setting `process.exitCode`?
5. What is the `beforeExit` event?
6. Why is `exit` not a good asynchronous cleanup hook?
7. What does `process.kill()` actually do?
8. What does SIGTERM usually represent operationally?
9. Why must shutdown be idempotent?
10. Why does readiness need to become false before shutdown completes?

## Intermediate

11. Why can a timer keep a Node process alive?
12. What does `unref()` mean?
13. How do active handles differ conceptually from one-shot requests?
14. How would you debug a process that refuses to exit?
15. What is the difference between liveness and readiness?
16. Why should graceful shutdown have a timeout?
17. Why should an HTTP server stop accepting new connections before DB shutdown?
18. What should happen to background queue consumers during shutdown?
19. How should a child process be shut down?
20. Why can `uncaughtException` be dangerous to recover from?

## Advanced

21. Why might `beforeExit` fire more than once?
22. Why does a Promise's liveness differ from its completion semantics?
23. How do microtasks relate to process lifecycle?
24. What happens if shutdown begins while startup is still running?
25. How should partial initialization be rolled back?
26. How should multiple shutdown signals be handled?
27. What should a crash-only service do after a fatal invariant violation?
28. How do you decide the final process exit code?
29. How do you prevent one stuck resource from blocking shutdown forever?
30. What resources commonly keep production Node services alive unexpectedly?

## Principal Level

31. Design a shutdown protocol for a Node service serving HTTP, WebSockets, Kafka, PostgreSQL, Redis, and worker threads.
32. How would you prove that a service drains safely under high load?
33. What is your strategy for long-running requests during deployment?
34. How do you distinguish process liveness from business health?
35. What observability data do you record at startup and shutdown?
36. How do you design lifecycle behavior across local development, containers, Kubernetes, systemd, and serverless?
37. When is forced termination acceptable?
38. How do you make shutdown deterministic enough for integration tests?
39. How would you test signal races?
40. What are the failure modes of using `beforeExit` as an application lifecycle hook?

---

# 54. Predict-the-Output Exercises

## Exercise A

```js
console.log('A');

Promise.resolve().then(() => {
  console.log('B');
});

console.log('C');
```

Predict the output order.

---

## Exercise B

```js
setTimeout(() => console.log('timeout'), 0);

Promise.resolve().then(() => console.log('promise'));

console.log('sync');
```

Predict the order and explain what is guaranteed versus runtime scheduling detail.

---

## Exercise C

```js
process.on('beforeExit', (code) => {
  console.log('beforeExit', code);
});

console.log('main');
```

Predict the lifecycle.

---

## Exercise D

```js
process.on('exit', (code) => {
  console.log('exit', code);
});

process.exitCode = 7;
```

Predict the final reported code.

---

## Exercise E

```js
const pending = new Promise(() => {});

console.log('done');
```

Predict whether the process remains alive and justify using liveness.

---

## Exercise F

```js
process.on('beforeExit', () => {
  console.log('before');
  setTimeout(() => console.log('timer'), 0);
});
```

What lifecycle pattern emerges?

---

## Exercise G

```js
process.on('exit', () => {
  setTimeout(() => console.log('late'), 0);
});
```

Does `late` reliably print?

---

## Exercise H

```js
process.on('SIGTERM', () => {
  console.log('signal');
});

setInterval(() => {}, 1000);
```

What happens after SIGTERM on a platform where the listener is active?

---

# 55. Mastery Exercises

## Exercise 1 — Lifecycle State Machine

Create an explicit state machine with transition validation:

```text
BOOTSTRAPPING
INITIALIZING
READY
ACTIVE
DRAINING
TERMINATING
EXITED
FAILED_STARTUP
CRASHING
```

Reject illegal transitions.

---

## Exercise 2 — Graceful HTTP Server

Build a server that:

- exposes `/health/live`,
- exposes `/health/ready`,
- marks readiness false on shutdown,
- stops accepting new connections,
- drains current requests,
- closes dependencies,
- exits with a meaningful code.

---

## Exercise 3 — Shutdown Budget

Build:

```js
ShutdownBudget
```

with:

```text
deadline
phase timeout
remaining time
elapsed time
```

Use it to bound cleanup.

---

## Exercise 4 — Resource Registry

Implement:

```js
ResourceRegistry
```

that supports:

```js
register(name, start, stop)
startAll()
stopAll()
```

Requirements:

- reverse-order stop,
- partial-start rollback,
- duplicate registration detection,
- stop errors collected,
- final timeout.

---

## Exercise 5 — Worker Pool Drain

Implement a small worker pool that can:

```text
accept
stop intake
drain
terminate
```

Prove that no new jobs are accepted after the drain boundary.

---

## Exercise 6 — Crash Path

Design a fatal-error path that:

```text
observes
→ records diagnostics
→ stops intake
→ performs bounded synchronous-safe cleanup where feasible
→ exits
```

Explain why it does not attempt normal service recovery.

---

## Exercise 7 — Deployment Simulation

Simulate:

```text
old version
   ↓
new version starts
   ↓
new version ready
   ↓
old version SIGTERM
   ↓
old version drains
   ↓
old version exits
```

Verify that no request is lost under controlled test timing.

---

# 56. Principal Decision Framework

When evaluating lifecycle design, score decisions across:

| Dimension | Question |
|---|---|
| Correctness | Can work finish without invalid state? |
| Performance | Is shutdown sufficiently fast? |
| Memory | Are resource references released? |
| Security | Are secrets and privileged resources handled safely? |
| Reliability | Does shutdown converge? |
| Maintainability | Is lifecycle logic centralized? |
| Scalability | Does it work with many workers/connections? |
| Observability | Can operators reconstruct what happened? |
| Developer Experience | Can engineers reproduce lifecycle behavior locally? |
| Operational Complexity | Are deadlines and dependencies understandable? |
| Future Change | Can new resources be added safely? |

A principal engineer should be able to defend the chosen lifecycle based on these criteria, not merely copy a signal-handler snippet.

---

# 57. Failure Modes in Production

## Failure Mode 1 — Deployment hangs

Root cause:

```text
resource never closes
```

## Failure Mode 2 — Requests disappear during rolling restart

Root cause:

```text
process exits before in-flight work drains
```

## Failure Mode 3 — Service is “healthy” while draining

Root cause:

```text
liveness and readiness conflated
```

## Failure Mode 4 — Shutdown starts twice

Root cause:

```text
no idempotency
```

## Failure Mode 5 — Fatal exception leaves corrupted service alive

Root cause:

```text
uncaughtException used as normal recovery
```

## Failure Mode 6 — Logs disappear

Root cause:

```text
process.exit() before async output finishes
```

## Failure Mode 7 — Old workers remain

Root cause:

```text
child/worker ownership not included in lifecycle graph
```

---

# 58. Production Architecture Pattern

A mature Node service often separates lifecycle responsibilities into layers:

```text
main()
  │
  ▼
ApplicationBootstrap
  │
  ├── Config
  ├── Logger
  ├── DB
  ├── Cache
  ├── Queue
  ├── HTTP
  ├── Workers
  └── Metrics
       │
       ▼
LifecycleCoordinator
  │
  ├── startup
  ├── readiness
  ├── signal handling
  ├── draining
  ├── cleanup
  ├── deadline
  └── final outcome
```

This keeps process concerns out of every individual module.

---

# 59. Lifecycle Observability

Record at minimum:

### Startup
- process start timestamp,
- PID,
- Node version,
- configuration version,
- build/release identifier,
- startup phase durations,
- readiness timestamp.

### Shutdown
- shutdown timestamp,
- shutdown reason,
- signal,
- current phase,
- drain duration,
- resource durations,
- cleanup failures,
- forced termination,
- final exit code.

### Fatal failure
- error,
- origin,
- stack,
- process uptime,
- current lifecycle state,
- relevant request/job context.

Do not expose secrets or full environment contents indiscriminately.

---

# 60. Lifecycle Timeline Example

```text
12:00:00.000  process started
12:00:00.120  config loaded
12:00:00.450  database connected
12:00:00.700  queue connected
12:00:00.900  HTTP listening
12:00:00.905  readiness=true

12:05:42.100  SIGTERM received
12:05:42.102  readiness=false
12:05:42.103  stop accepting work
12:05:42.110  18 requests draining
12:05:42.130  queue intake stopped
12:05:43.002  all HTTP requests complete
12:05:43.240  DB pool closed
12:05:43.300  queue closed
12:05:43.320  metrics flushed
12:05:43.321  shutdown complete
12:05:43.322  exit code=0
```

This is what a successful lifecycle looks like operationally: **explicit, bounded, observable**.

---

# 61. Advanced Comparison — Graceful Shutdown Strategies

| Strategy | Strength | Weakness | Best fit |
|---|---|---|---|
| Immediate `process.exit()` | Simple | Drops pending work | CLI/fatal emergency |
| `exitCode` + natural drain | Safe | May wait for unwanted resources | Short-lived jobs |
| Explicit shutdown coordinator | Predictable | More design work | Production services |
| External supervisor restart | Strong recovery boundary | Requires infrastructure | Fatal failures |
| Crash-only architecture | Resistant to corrupted state | Requires good observability | Critical backends |
| Graceful + deadline + force | Balanced | More complex | Most production servers |

---

# 62. Real-World Scenario

You operate a Node.js API:

```text
HTTP
WebSocket
PostgreSQL
Redis
Kafka
6 worker threads
2 child processes
metrics exporter
```

Deployment orchestrator sends SIGTERM.

### Correct plan

```text
1. receive SIGTERM
2. transition ACTIVE → DRAINING
3. set readiness=false
4. stop new HTTP requests
5. stop new WebSocket sessions
6. stop accepting new Kafka jobs
7. let in-flight requests/jobs finish
8. ask workers for graceful drain
9. gracefully stop child processes
10. close WebSocket sessions
11. close Redis
12. close PostgreSQL
13. flush bounded telemetry
14. ensure all known resources are closed
15. set exit code
16. terminate before hard deadline
```

### Wrong plan

```text
SIGTERM
  ↓
process.exit(0)
```

That is not graceful shutdown.

---

# 63. Chapter Connections

## Depends On

- Chapter 12 — Execution Model
- Chapter 29 — Error Handling
- Chapter 30 — Resource Management
- Chapter 31–38 — Async runtime
- Chapter 34 — Node Event Loop
- Chapter 37 — Cancellation
- Chapter 58 — Node Architecture
- Chapter 59 — Node Core APIs
- Chapter 60 — Streams
- Chapter 61 — Workers and Child Processes

## Builds Toward

- Chapter 63 — Async Context and Diagnostics
- Chapter 70 — Source Maps and Production Debugging
- Chapter 78 — Production JavaScript Architecture
- Chapter 83 — Observability
- Chapter 84 — Reliability
- Chapter 85 — Performance
- Chapter 88 — Debugging Methodology
- Chapter 101 — Real-World Production Scenarios
- Chapter 105 — Node REST API
- Chapter 110 — Production JS Backend
- Chapter 111 — Large-Scale JS Platform

## Related Concepts

- Event loop
- libuv handles
- signals
- cancellation
- resource ownership
- supervisors
- containers
- readiness
- liveness
- crash-only design
- structured cleanup

## Why This Chapter Matters Later

At senior and principal levels, “the app works” is not enough.

You must also prove:

> **the app starts predictably, stays alive for the right reasons, stops safely, fails observably, and never depends on undefined cleanup behavior.**

---

# 64. Spaced Retrieval Plan

## Day 0

Explain from memory:

```text
Promise ≠ liveness
beforeExit ≠ shutdown
exit ≠ async cleanup
SIGTERM ≠ guaranteed termination
uncaughtException ≠ safe recovery
```

## Day 2

Implement:

```text
idempotent shutdown coordinator
```

without looking at notes.

## Day 7

Debug:

```text
Node won't exit
```

using only diagnostics.

## Day 14

Design a full graceful shutdown for:

```text
HTTP + DB + queue + workers
```

## Day 30

Defend:

> Why should a fatal exception generally result in process replacement rather than in-process recovery?

---

# 65. Dependency Graph

```text
ECMAScript execution model
          │
          ▼
Async jobs / promises
          │
          ▼
Node event loop / libuv
          │
          ▼
Node process liveness
          │
          ├───────────────┐
          ▼               ▼
signals               resources
  │                     │
  ▼                     ▼
shutdown          HTTP / DB / queue /
strategy          workers / children
          │
          ▼
graceful lifecycle
          │
          ▼
reliability + observability
```

---

# 66. Completion Criteria

You may mark this chapter `[+] Completed` only when you can:

```text
[ ] Explain Node startup to steady state
[ ] Explain natural process exit
[ ] Explain process liveness
[ ] Explain Promise vs liveness
[ ] Explain ref/unref
[ ] Explain beforeExit
[ ] Explain exit
[ ] Explain exitCode vs exit()
[ ] Explain signals
[ ] Explain signal platform differences
[ ] Explain uncaughtException
[ ] Explain uncaughtExceptionMonitor
[ ] Explain unhandledRejection
[ ] Design idempotent shutdown
[ ] Design bounded graceful shutdown
[ ] Drain HTTP work
[ ] Shut down workers
[ ] Shut down child processes
[ ] Handle startup rollback
[ ] Debug hanging processes
[ ] Debug early exits
[ ] Separate readiness from liveness
[ ] Explain crash-only recovery
[ ] Implement a lifecycle state machine
[ ] Pass the principal interview section
```

---

# 67. Mastery Gate

You have **mastered** this chapter only when you can perform all eight:

### 1. Understand
Explain process lifecycle from first principles.

### 2. Explain
Teach graceful shutdown to another engineer.

### 3. Predict
Predict whether a Node process exits from unfamiliar code.

### 4. Implement
Build an idempotent shutdown coordinator from scratch.

### 5. Debug
Find the hidden resource keeping a process alive.

### 6. Apply
Design production lifecycle behavior for a real backend.

### 7. Compare
Defend graceful shutdown versus crash-only behavior.

### 8. Defend
Explain and defend your lifecycle design under principal-level review.

---

# 68. Status

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

Reading this chapter does **not** mark mastery.

---

# 69. Chapter 62 — Revision / Retrieval Record

Use this section after each retrieval session.

| Date | Recall task | Result | Weak area | Next action |
|---|---|---|---|---|
| YYYY-MM-DD | Explain liveness | ✅ / ❌ | ... | ... |
| YYYY-MM-DD | Implement shutdown | ✅ / ❌ | ... | ... |
| YYYY-MM-DD | Debug hanging process | ✅ / ❌ | ... | ... |
| YYYY-MM-DD | Principal defense | ✅ / ❌ | ... | ... |

### Revision Questions

```text
1. What exactly keeps a Node process alive?
2. Why can a Promise remain pending while the process exits?
3. When can beforeExit prolong process life?
4. Why can't exit await cleanup?
5. Why is process.exit dangerous?
6. What does SIGTERM mean operationally?
7. Why must shutdown be idempotent?
8. What is the correct order of stopping a service?
9. Why should readiness become false before resource teardown?
10. Why are uncaught exceptions usually restart boundaries?
```

---

# 70. Chapter 62 — Canonical References and Source Discipline

## Primary source hierarchy

### Level 1 — Official Node.js documentation

- Node.js Process API  
  https://nodejs.org/api/process.html
- Node.js HTTP API  
  https://nodejs.org/api/http.html
- Node.js Net API  
  https://nodejs.org/api/net.html
- Node.js Child Process API  
  https://nodejs.org/api/child_process.html
- Node.js Worker Threads API  
  https://nodejs.org/api/worker_threads.html

### Level 2 — ECMAScript specification

Use ECMA-262 for:

- ECMAScript Jobs,
- Promise semantics,
- execution semantics,
- host integration boundaries.

https://tc39.es/ecma262/

### Level 3 — Runtime implementation

For deeper internals:

- Node.js source,
- libuv source/docs,
- V8 source/docs.

Treat these as implementation evidence rather than universal language guarantees.

---

## Source-discipline rules

When studying process lifecycle:

1. **Prefer current official Node.js documentation for public Node APIs.**
2. Distinguish Node guarantees from OS guarantees.
3. Distinguish public behavior from implementation details.
4. Verify version-sensitive shutdown APIs before production use.
5. Never infer lifecycle semantics solely from browser JavaScript knowledge.
6. Treat platform differences explicitly.
7. Use experiments to validate behavior, but do not confuse one experiment with a universal specification guarantee.

### Current documentation baseline

This chapter was written against the current Node.js 26.x documentation available at the time of authoring, including:

- `beforeExit`,
- `exit`,
- `uncaughtException`,
- `uncaughtExceptionMonitor`,
- `unhandledRejection`,
- `process.exit()`,
- `process.exitCode`,
- signal events,
- `process.getActiveResourcesInfo()`,
- HTTP server shutdown helpers,
- worker termination,
- child process signaling.

---

# 71. Chapter 62 — Completion Snapshot

## Core Theory

```text
[ ] Process lifecycle state model understood
[ ] Node liveness understood
[ ] Signal semantics understood
[ ] Exit semantics understood
[ ] Fatal error semantics understood
```

## Implementation

```text
[ ] LifecycleManager implemented
[ ] Resource registry implemented
[ ] Graceful HTTP shutdown implemented
[ ] Worker shutdown implemented
[ ] Child-process shutdown implemented
[ ] Shutdown deadline implemented
```

## Interview

```text
[ ] Foundation questions passed
[ ] Advanced questions passed
[ ] Principal questions defended
[ ] Output prediction passed
```

## Production

```text
[ ] Readiness/liveness modeled
[ ] Failure modes tested
[ ] Observability added
[ ] Signal races tested
[ ] Deployment termination simulated
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

The most important lesson of this chapter is not the syntax of `process.on('SIGTERM')`.

It is the deeper model:

```text
A Node.js process is a runtime with resources,
not merely a JavaScript program.

Startup establishes invariants.
Liveness determines whether the runtime continues.
Readiness determines whether traffic should arrive.
Draining protects in-flight work.
Cleanup releases ownership.
Deadlines prevent infinite shutdown.
Exit codes communicate outcome.
Fatal errors can invalidate the process itself.
External supervisors provide the final recovery boundary.
```

A principal engineer therefore asks:

> **What state is the process in? What resources does it own? What work is still in flight? What keeps it alive? What happens if cleanup fails? What happens if shutdown is requested twice? What happens if the process is already corrupted? What is the hard deadline? How will operators know exactly why it exited?**

Once you can answer those questions reliably, you no longer “handle SIGTERM.”

You engineer a **process lifecycle**.


---

## Source Notes Used for Version-Sensitive Claims

- Node.js Process API (v26.x): `beforeExit`, `exit`, uncaught exception events, unhandled rejection events, signals, `process.exit()`, `process.exitCode`, process diagnostics:  
  https://nodejs.org/api/process.html
- Node.js Net API (v26.x): `server.close()` and server lifecycle behavior:  
  https://nodejs.org/api/net.html
- Node.js HTTP API (v26.x): HTTP connection and shutdown helpers:  
  https://nodejs.org/api/http.html
- Node.js Worker Threads API (v26.x): asynchronous `worker.terminate()`:  
  https://nodejs.org/api/worker_threads.html
- Node.js Child Process API (v26.x): process signaling, exit/close semantics, signal behavior, and descendant-process caveats:  
  https://nodejs.org/api/child_process.html
- ECMAScript Language Specification:  
  https://tc39.es/ecma262/