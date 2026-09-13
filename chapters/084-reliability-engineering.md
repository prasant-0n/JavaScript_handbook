# Chapter 84 — Reliability

> **Part XV — Production JavaScript**  
> **Status:** `[ ] Not Started`  
> **Depth Target:** Principal / Staff-level reasoning  
> **Primary Focus:** Designing JavaScript systems that continue to behave correctly under overload, dependency failure, retries, partial failure, process termination, network uncertainty, resource exhaustion, and change.

---

# 1. Learning Objectives

By the end of this chapter, you should be able to:

1. Define reliability as a system property rather than merely “uptime.”
2. Distinguish reliability, availability, durability, resilience, fault tolerance, and recoverability.
3. Model failures before implementing resilience mechanisms.
4. Identify failure domains and blast radius.
5. Design explicit failure budgets and reliability targets.
6. Explain graceful degradation.
7. Design timeouts and deadline propagation.
8. Design retries with bounded attempts and appropriate backoff.
9. Understand jitter and synchronized retry storms.
10. Design idempotent operations.
11. Design deduplication for jobs and events.
12. Design circuit breakers and understand when they help.
13. Design bulkheads and concurrency limits.
14. Design load shedding and admission control.
15. Design queue backpressure.
16. Design graceful shutdown and startup behavior.
17. Handle dependency outages without turning them into cascading failures.
18. Reason about partial success and partial failure.
19. Design fallback behavior without returning unsafe or misleading results.
20. Distinguish fail-open and fail-closed decisions.
21. Design durable asynchronous workflows.
22. Explain outbox/inbox patterns as reliability techniques.
23. Design recovery and reconciliation workflows.
24. Design health, readiness, and liveness semantics.
25. Reason about database reliability and pool saturation.
26. Reason about cache failure and stale data.
27. Reason about external API failure.
28. Design reliability for multi-tenant systems.
29. Design reliability under traffic spikes.
30. Understand error budgets and SLO-driven engineering.
31. Build resilience mechanisms without creating invisible complexity.
32. Test failure behavior deliberately.
33. Use fault injection and controlled chaos experiments responsibly.
34. Diagnose retry storms, queue growth, cascading failure, and brownouts.
35. Build a reliability-oriented Node.js service.
36. Review an architecture for hidden single points of failure.
37. Defend reliability trade-offs at principal-engineer level.

---

# 2. Prerequisites

You should understand:

- JavaScript functions, promises, async/await, modules, and errors.
- Node.js event loop and process lifecycle.
- AbortController / cancellation.
- HTTP and API architecture.
- Database transactions and connection pools.
- Streams and backpressure.
- Observability.
- Production architecture.

Recommended prior chapters:

- **29** — Errors / Error Handling
- **30** — Resource Management / Cleanup
- **31–40** — Async / Promises / Cancellation / Streaming / Concurrency
- **45–48** — Memory / Garbage Collection / Engine
- **55–57** — Networking / Security
- **58–63** — Node.js Runtime / Streams / Workers / Lifecycle / Diagnostics
- **78** — Production JavaScript Architecture
- **79** — API Design
- **81** — Database Integration
- **82** — API Architecture
- **83** — Observability

---

# 3. What Is It?

Reliability is the ability of a system to continue providing its required behavior despite faults, and to recover when faults occur.

Reliability is broader than:

```text
server is running
```

A service can be “up” while:

```text
payments fail
orders duplicate
data disappears
requests time out
queues grow forever
```

That system is not reliably delivering its intended behavior.

---

## 3.1 Reliability dimensions

A useful model:

```text
Correctness
Availability
Durability
Resilience
Recoverability
Consistency
Security
Operability
```

These overlap but are not identical.

---

## 3.2 Reliability versus availability

Availability asks:

```text
Can the system respond?
```

Reliability asks more:

```text
Does it respond correctly?
Does it preserve required state?
Does it recover from failure?
Does it avoid duplicate side effects?
Can it continue under partial failure?
```

A service returning:

```http
200 OK
```

with incorrect financial data is available but unreliable.

---

# 4. Why Does It Exist?

All non-trivial systems fail.

Failures include:

```text
network timeout
database overload
CPU saturation
memory pressure
process crash
dependency outage
bad deployment
expired certificate
DNS issue
queue backlog
duplicate message
lost response
disk full
credential failure
schema mismatch
```

Reliability engineering exists because:

> **The absence of failure is not a realistic design assumption.**

---

# 5. Mental Model

Think of reliability as layers:

```text
                 User / Consumer
                        │
                 API / Contract
                        │
               Application Logic
                        │
        ┌───────────────┼───────────────┐
        │               │               │
      Database        Queue         Dependency
        │               │               │
        └───────────────┼───────────────┘
                        │
                    Runtime
                        │
                 Infrastructure
```

For every layer ask:

```text
What can fail?
How is failure detected?
How does failure propagate?
Can work be retried?
Can work duplicate?
What is the blast radius?
How does the system recover?
```

---

# 6. Core Rules

## Rule 1 — Start with failure modes

Do not begin with:

```text
add retry
add circuit breaker
add cache
```

Begin with:

```text
what failed?
why?
what effect did it produce?
what behavior is required?
```

---

## Rule 2 — Timeouts are mandatory at remote boundaries

An operation that can wait forever is a reliability hazard.

Node.js currently provides `AbortSignal.timeout()` for creating a signal that aborts after a specified delay, and `AbortSignal.any()` for combining cancellation signals. citeturn288849search2

---

## Rule 3 — Retries require idempotency or equivalent safety

Never assume:

```text
retry = safe
```

---

## Rule 4 — Bound everything

Bound:

```text
timeouts
retries
queue length
concurrency
memory
payload size
batch size
cache size
```

Unbounded systems eventually become unreliable.

---

## Rule 5 — Fail fast when continuing cannot succeed

If a dependency is known to be unavailable, repeated calls may worsen the outage.

---

## Rule 6 — Separate critical from non-critical work

For example:

```text
place order → critical
send recommendation email → non-critical
```

Do not block the critical workflow on optional work.

---

## Rule 7 — Preserve correctness before availability

Returning incorrect state is often worse than returning an error.

---

## Rule 8 — Make recovery explicit

Every important failure needs:

```text
detection
mitigation
recovery
reconciliation
```

---

## Rule 9 — Observability is part of reliability

You cannot reliably operate behavior you cannot observe.

---

## Rule 10 — Reliability mechanisms have costs

Every retry, queue, cache, fallback, and circuit breaker adds:

```text
state
complexity
latency behavior
debugging difficulty
operational burden
```

Use mechanisms because they solve identified failures.

---

# 7. Syntax

## 7.1 Timeout

```js
const signal = AbortSignal.timeout(2_000);

const response = await fetch(url, {
  signal,
});
```

Node's current global `AbortSignal` implementation supports `timeout()` and `any()`; verify the minimum Node version in your service compatibility policy before using these APIs. citeturn288849search2

---

## 7.2 Retry loop

```js
async function retry(operation, {
  attempts = 3,
}) {
  let lastError;

  for (let attempt = 1; attempt <= attempts; attempt += 1) {
    try {
      return await operation();
    } catch (error) {
      lastError = error;

      if (attempt === attempts) {
        throw error;
      }
    }
  }

  throw lastError;
}
```

This is incomplete production logic because it lacks:

```text
retry classification
backoff
jitter
deadline
idempotency
observability
```

---

## 7.3 Concurrency limit

Conceptually:

```js
const semaphore = createSemaphore(20);

await semaphore.run(async () => {
  return operation();
});
```

The point is:

```text
at most 20 concurrent operations
```

---

# 8. Basic Examples

## Example 1 — Timeout

```js
await fetch(url, {
  signal: AbortSignal.timeout(3_000),
});
```

Without timeout:

```text
request
→ dependency becomes unhealthy
→ request waits indefinitely
→ resources remain occupied
```

With timeout:

```text
request
→ dependency too slow
→ abort
→ free resources
→ classify failure
```

---

## Example 2 — Retryable versus permanent

Potentially retryable:

```text
temporary network failure
connection reset
some overload responses
serialization conflict
```

Usually not retryable:

```text
invalid input
permission denied
unknown resource
business rule violation
```

Actual retryability depends on the operation, protocol, provider, and failure semantics.

---

# 9. Execution Walkthrough

Consider:

```http
POST /orders
```

A reliability-aware flow:

```text
receive request
  ↓
authenticate
  ↓
validate
  ↓
idempotency check
  ↓
transaction
  ├─ create order
  ├─ reserve inventory
  └─ record durable intent
  ↓
commit
  ↓
return response
  ↓
async side effects
```

If payment provider is down:

```text
order state = payment_pending
```

rather than:

```text
request hangs forever
```

The correct choice depends on business semantics.

---

# 10. Internal Mechanics

## 10.1 Failure propagation

Suppose:

```text
API
 ↓
Service A
 ↓
Service B
 ↓
Database
```

If the database slows:

```text
DB latency ↑
→ B latency ↑
→ A latency ↑
→ API latency ↑
→ clients timeout
→ clients retry
→ load ↑
```

This is a feedback loop.

Reliability engineering interrupts feedback loops with:

```text
timeouts
concurrency limits
load shedding
backpressure
circuit breakers
retry budgets
```

---

## 10.2 Deadline propagation

A request may have:

```text
total deadline = 2 seconds
```

Do not allow every downstream call to use:

```text
2 seconds
```

independently.

Otherwise:

```text
A waits 2s
B waits 2s
C waits 2s
```

The upstream deadline can already be expired.

Instead:

```text
remaining deadline
→ downstream timeout
```

---

## 10.3 Retry backoff

Naive:

```text
retry immediately
retry immediately
retry immediately
```

Problem:

```text
many clients fail simultaneously
→ all retry simultaneously
→ dependency gets more traffic
→ remains unhealthy
```

Exponential backoff:

```text
100 ms
200 ms
400 ms
800 ms
```

Jitter randomizes timing.

---

## 10.4 Retry budget

Suppose:

```text
100 original requests/s
```

and every request can retry three times.

Worst-case attempt count can approach:

```text
400 attempts/s
```

depending on which failures occur and when.

Treat retries as additional traffic.

---

# 11. ECMAScript / Specification Semantics

Reliability is mostly above ECMAScript.

Separate:

```text
ECMAScript
  → Promise / function / object semantics

Node.js
  → timers / AbortSignal / process behavior

HTTP
  → request/response semantics

Database
  → transaction and durability semantics

Application
  → retries / idempotency / fallback

Operations
  → SLO / incident / recovery policy
```

For example:

```js
await operation();
```

does not imply:

```text
retry
timeout
rollback
idempotency
durability
recovery
```

Those properties require system design.

---

# 12. Advanced Behavior

## 12.1 Idempotency

An operation is idempotent when repeating the same intended operation does not cause additional undesired effect.

Example:

```http
PUT /users/42
```

may semantically set a known state.

For non-idempotent creation/payment workflows:

```http
POST /payments
Idempotency-Key: pmt-abc
```

can allow server-side deduplication.

A typical record:

```text
idempotency_key
request_hash
status
response
created_at
expires_at
```

---

## 12.2 Idempotency must include semantics

This is not enough:

```js
if (seen(key)) return;
```

You must consider:

```text
what operation?
which actor?
which tenant?
same request body?
same endpoint?
same version?
what if first operation is still running?
what if first operation failed after partial side effect?
```

---

## 12.3 Circuit breaker

Conceptual state:

```text
CLOSED
  ↓ failures exceed threshold
OPEN
  ↓ recovery period
HALF_OPEN
  ↓ success
CLOSED
```

Open:

```text
fail fast
```

instead of repeatedly calling an unhealthy dependency.

Do not implement a circuit breaker merely because the name sounds resilient.

It should improve failure containment.

---

## 12.4 Bulkhead

Separate resources:

```text
payment concurrency = 20
recommendation concurrency = 10
search concurrency = 20
```

Payment overload cannot consume every available worker slot.

This is analogous to compartments in a ship.

---

## 12.5 Load shedding

When capacity is exceeded:

```text
reject early
```

rather than:

```text
accept everything
queue forever
timeout everything
```

Possible forms:

```text
429 Too Many Requests
503 Service Unavailable
bounded queue rejection
feature disablement
```

---

## 12.6 Graceful degradation

Examples:

```text
recommendations unavailable
→ show product without recommendations

analytics unavailable
→ complete purchase without analytics

cache unavailable
→ read from source of truth

optional search unavailable
→ return basic results
```

Do not degrade in ways that violate correctness or security.

---

## 12.7 Fail-open versus fail-closed

Example authorization dependency:

```text
auth service unavailable
```

Fail-open:

```text
allow request
```

Fail-closed:

```text
deny request
```

For security-sensitive authorization, fail-open may create catastrophic exposure.

For a non-critical feature, fail-open may be reasonable.

The correct choice is domain-specific.

---

# 13. Edge Cases

## 13.1 Timeout after success

```text
remote operation succeeds
→ response lost
→ local timeout
```

Retry can duplicate the operation.

---

## 13.2 Cancellation race

```text
request cancelled
provider operation already committed
```

Cancellation does not guarantee remote rollback.

Node's current `AbortSignal` behavior distinguishes abort signaling from automatically undoing arbitrary external work. `AbortSignal.timeout()` and `AbortSignal.any()` provide cancellation signals; the operation itself must honor cancellation for actual interruption. citeturn288849search2

---

## 13.3 Retry after partial completion

A workflow may have completed step 1 but failed at step 2.

Naively restarting from step 1 can duplicate step 1.

Prefer:

```text
durable workflow state
```

or:

```text
idempotent steps
```

---

## 13.4 Queue consumer crash

```text
receive message
→ perform side effect
→ crash before acknowledgement
```

Message may be delivered again.

Consumer must be safe under duplicate delivery when the transport provides at-least-once semantics.

---

## 13.5 Poison message

A message always fails:

```text
consume
→ fail
→ retry
→ fail
→ retry
```

Without a dead-letter strategy, it can block or consume disproportionate capacity.

---

# 14. Common Misconceptions

### “Retries make systems more reliable.”

Only when bounded and semantically safe.

### “Timeout means cancellation.”

A timeout may stop waiting locally; the remote operation may continue.

### “Circuit breakers prevent outages.”

They can limit dependency pressure, but do not repair the dependency.

### “Queues absorb infinite traffic.”

Queues only delay overload until capacity or storage is exhausted.

### “Caching improves reliability.”

Caching can reduce dependency load, but stale or corrupted cache state can create new failures.

### “Fallback is always better.”

A fallback that returns incorrect or unsafe information is worse than an explicit failure.

### “High availability means correct behavior.”

No.

### “Graceful shutdown is optional.”

In-flight work can be lost or corrupted without a planned lifecycle.

---

# 15. Common Mistakes

## Mistake 1 — No timeout

```js
await fetch(url);
```

at a remote boundary with no explicit deadline policy.

---

## Mistake 2 — Retry everything

```js
catch (error) {
  return retry();
}
```

---

## Mistake 3 — Unbounded queue

```js
pending.push(job);
```

forever.

---

## Mistake 4 — One global concurrency pool

Critical and non-critical operations compete for the same resource.

---

## Mistake 5 — Retrying a non-idempotent write

Payment, reservation, and provisioning operations can duplicate.

---

## Mistake 6 — Fallback hides critical failure

Returning stale financial state as if it were current can be worse than an error.

---

## Mistake 7 — Retry budget ignored

Multiple layers all retry:

```text
client
gateway
service
SDK
database driver
```

Traffic multiplies.

---

## Mistake 8 — Shutdown closes resources in wrong order

Database closes before in-flight requests complete.

---

# 16. Comparison With Related Concepts

| Mechanism | Main Purpose | Main Cost |
|---|---|---|
| Timeout | Bound waiting | Possible incomplete remote work |
| Retry | Recover transient failure | More load / duplicates |
| Backoff | Reduce retry pressure | More latency |
| Jitter | Desynchronize retries | Slight unpredictability |
| Circuit breaker | Fail fast on dependency | Temporary rejection |
| Bulkhead | Contain resource impact | Capacity fragmentation |
| Load shedding | Protect system capacity | Explicit rejection |
| Queue | Decouple time | Backlog / durability complexity |
| Cache | Reduce dependency load | Staleness / invalidation |
| Fallback | Preserve partial function | Semantics complexity |
| Idempotency | Safely handle repeats | Storage/state |
| Deduplication | Suppress duplicate work | State / retention |
| Rate limit | Protect service | User rejection |
| Graceful shutdown | Preserve in-flight correctness | Slower termination |
| Health/readiness | Control routing | Can be misleading if poorly designed |

---

# 17. Performance Considerations

Reliability and performance are coupled.

## 17.1 Queueing

As utilization approaches capacity:

```text
latency increases
```

A service running close to saturation has little room for bursts.

---

## 17.2 Retries increase load

If:

```text
base traffic = 1,000 req/s
```

and average attempt count becomes:

```text
1.4
```

effective downstream traffic can become approximately:

```text
1,400 attempts/s
```

Retries must be included in capacity planning.

---

## 17.3 Timeouts consume resources

Long timeout:

```text
more in-flight requests
more memory
more sockets
more concurrency
```

Short timeout:

```text
more failures
possible premature cancellation
```

Choose based on actual latency distributions and business deadlines.

---

## 17.4 Circuit breakers reduce waste

Failing fast can reduce:

```text
connection usage
worker occupancy
queue growth
retry pressure
```

but can also reduce availability when dependency failures are temporary.

---

# 18. Memory Considerations

Reliability failures often become memory failures.

Example:

```text
dependency slow
→ in-flight requests increase
→ promises retained
→ response buffers retained
→ heap grows
→ GC increases
→ latency increases
→ more timeouts
```

This is a positive feedback loop.

Bound:

```text
concurrency
queue size
payload size
buffer size
cache size
retry backlog
```

---

## 18.1 Telemetry memory

OpenTelemetry documentation explicitly treats telemetry loss as preferable to significantly changing application behavior when telemetry infrastructure fails, reinforcing the principle that observability should not become a critical dependency that destabilizes the application. citeturn288849search10

---

# 19. Security Considerations

Reliability decisions can become security decisions.

## 19.1 Fail-open authorization

Potentially catastrophic.

```text
authorization service unavailable
→ allow request
```

Avoid unless the security model explicitly permits it.

---

## 19.2 Retry amplification as attack surface

An attacker can intentionally trigger failures and exploit:

```text
retries
queues
downstream calls
```

to amplify load.

---

## 19.3 Resource exhaustion

Bound:

```text
request body
concurrency
query complexity
uploads
queue size
batch size
```

OWASP includes unrestricted resource consumption among major API security risks. citeturn277088search4

---

## 19.4 Secret rotation failure

Reliability issue:

```text
credential expires
→ every request fails
```

Design:

```text
rotation
refresh
staged deployment
fallback credential where policy allows
```

---

# 20. Production Usage

## 20.1 Reliability architecture

```text
Client
  ↓
Rate Limit
  ↓
API
  ↓
Timeout / Deadline
  ↓
Use Case
  ↓
┌─────────────┬─────────────┐
│             │             │
Database    Queue        External API
│             │             │
└─────────────┴─────────────┘
  ↓
Observability
  ↓
Recovery / Reconciliation
```

---

## 20.2 Reliability budget

A useful design review asks:

```text
How much failure can the user tolerate?
How much data loss can we tolerate?
How long can processing be delayed?
How much stale data is acceptable?
Which operations must never duplicate?
Which features may degrade?
```

---

## 20.3 SLO

Example:

```text
99.9% of valid checkout requests complete successfully within 500 ms
```

This is more actionable than:

```text
service must be highly reliable
```

---

## 20.4 Error budget

Suppose:

```text
SLO = 99.9%
```

Then some amount of unreliability is tolerated during the measurement window.

Use error-budget thinking to balance:

```text
feature velocity
reliability work
risk
release frequency
```

---

## 20.5 Health and readiness

Node process:

```text
started
↓
initializing
↓
ready
↓
draining
↓
stopped
```

Readiness should be removed when the process cannot safely serve new work.

---

## 20.6 Graceful shutdown

Recommended conceptual order:

```text
receive termination signal
  ↓
stop accepting new work
  ↓
mark not ready
  ↓
wait for bounded in-flight work
  ↓
stop background intake
  ↓
flush essential state
  ↓
close database/clients/workers
  ↓
exit
```

Node's process/runtime behavior should be implemented according to the actual server framework and infrastructure environment.

---

## 20.7 Node.js cancellation

Node's current `AbortSignal` supports:

```js
AbortSignal.timeout(ms)
AbortSignal.any(signals)
```

and Node's documentation recommends one-shot abort listeners to avoid leaks. citeturn288849search2turn288849search5

Timers in the Node timers/promises API can also be cancelled through `AbortSignal`; this can make bounded asynchronous operations easier to compose. citeturn288849search8

---

## 20.8 Dependency reliability policy

For each external dependency define:

```text
timeout
retryability
maximum retries
backoff
jitter
fallback
circuit policy
bulkhead
rate limit
authentication
observability
owner
```

---

# 21. Implementation From Scratch

Build a reliability layer for a Node.js service.

## Stage 1 — Guided

Create a timeout-aware client:

```js
export async function requestWithTimeout(
  url,
  {
    timeoutMs = 2_000,
    signal,
  } = {},
) {
  const timeoutSignal = AbortSignal.timeout(timeoutMs);

  const combinedSignal = signal
    ? AbortSignal.any([
        signal,
        timeoutSignal,
      ])
    : timeoutSignal;

  return fetch(url, {
    signal: combinedSignal,
  });
}
```

Document that timeout controls local waiting/cancellation and does not guarantee remote rollback.

---

## Stage 2 — Partially Guided

Add:

```text
retry classification
backoff
jitter
attempt limit
overall deadline
request ID
telemetry
```

---

## Stage 3 — No Reference

Implement:

```js
createReliableClient({
  fetchImpl,
  timeoutMs,
  retryPolicy,
  logger,
  metrics,
});
```

Requirements:

```text
deadline
retry
abort
error classification
bounded attempts
observability
```

---

## Stage 4 — Edge-case hardened

Test:

```text
abort before request
abort during request
timeout
network reset
HTTP 429
HTTP 500
HTTP 400
slow dependency
duplicate request
retry budget exhaustion
```

---

## Stage 5 — Production-grade

Add:

```text
circuit breaker
bulkhead
rate limiting
adaptive concurrency where justified
request hedging only when justified
idempotency
load shedding
health/readiness
graceful shutdown
metrics
tracing
runbooks
fault injection
```

---

## Stage 6 — Reliable workflow

Build:

```text
Order API
   ↓
database transaction
   ↓
outbox
   ↓
worker
   ↓
payment provider
   ↓
reconciliation
```

Requirements:

```text
at-least-once delivery
idempotent consumer
bounded retries
dead-letter handling
replay
audit trail
```

---

# 22. Debugging Exercises

## Exercise 1 — Retry storm

```text
dependency error rate = 30%
client retries
gateway retries
service retries
```

Find the amplification factor.

---

## Exercise 2 — Queue overload

```text
arrival = 10,000 jobs/sec
processing = 2,000 jobs/sec
```

What happens over time?

Determine:

```text
queue growth
memory/storage impact
job latency
rejection policy
recovery time
```

---

## Exercise 3 — Cascading timeout

```text
API timeout = 5s
service A = 5s
service B = 5s
database = 5s
```

Explain why the architecture is not bounded by the upstream deadline.

---

## Exercise 4 — Duplicate payment

```text
payment provider succeeds
service times out
client retries
```

Design:

```text
idempotency key
provider idempotency
reconciliation
```

---

## Exercise 5 — Circuit breaker stuck open

A dependency recovers but all calls still fail.

Find possible causes:

```text
no half-open transition
wrong clock
health probe broken
threshold too aggressive
state shared incorrectly
```

---

## Exercise 6 — Cache dependency outage

Cache becomes unavailable.

Application crashes because every request assumes cache success.

Redesign:

```text
cache = optimization
source of truth = database
```

---

## Exercise 7 — Shutdown data loss

A worker receives `SIGTERM` and exits immediately while processing a job.

Find the lifecycle bug.

---

## Exercise 8 — Memory pressure during outage

Dependency slows.

Inflight requests rise.

Heap increases.

GC time rises.

Latency rises.

Describe the positive feedback loop and the first controls to apply.

---

# 23. Code Review Exercise

Review:

```js
async function createOrder(order) {
  const payment = await fetch(
    "https://payments.example/charge",
    {
      method: "POST",
      body: JSON.stringify(order),
    },
  );

  if (!payment.ok) {
    throw new Error("payment failed");
  }

  return db.orders.insert(order);
}
```

Identify at least 25 reliability problems.

Consider:

```text
timeout
cancellation
idempotency
retry
payment-before-DB sequencing
partial failure
transaction boundary
duplicate charge
response loss
error classification
status mapping
dependency saturation
observability
audit
reconciliation
database failure after payment
payment success after database failure
request disconnect
queue alternative
rate limiting
bulkhead
security
tenant context
```

---

# 24. Interview Questions

## Fundamental

1. What is reliability?
2. Reliability versus availability?
3. What is resilience?
4. What is fault tolerance?
5. What is graceful degradation?
6. What is a timeout?
7. Why are retries dangerous?
8. What is idempotency?
9. What is backpressure?
10. What is load shedding?

## Intermediate

11. What is exponential backoff?
12. What is jitter?
13. What is a circuit breaker?
14. What is a bulkhead?
15. What is an error budget?
16. What is graceful shutdown?
17. What is a dead-letter queue?
18. What is reconciliation?
19. How do you design an idempotent consumer?
20. How do you choose a timeout?

## Advanced

21. How do you prevent retry storms?
22. How do you design reliability across service boundaries?
23. How do you handle lost responses?
24. How do you design durable workflows?
25. How do you distinguish transient and permanent failures?
26. How do you design fail-open versus fail-closed behavior?
27. How do you prevent queue overload?
28. How do you test failure paths?
29. How do you recover from partial completion?
30. How do you design database/external-API workflows?

## Principal

31. What failures deserve graceful degradation versus hard failure?
32. How do you define a service's reliability budget?
33. Where should retry policy live?
34. How do you prevent multiple retry layers from multiplying traffic?
35. How do you choose between queueing and load shedding?
36. What is the blast radius of this dependency?
37. How do you quantify reliability trade-offs?
38. How do you design a system that remains useful during partial outage?
39. How do you decide whether a circuit breaker adds value?
40. How would you challenge a system described as “99.99% available” but known to lose orders?

---

# 25. Predict-the-Output Exercises

## Exercise A — Abort state

Predict:

```js
const signal = AbortSignal.abort("test");

console.log(signal.aborted);
console.log(signal.reason);
```

### Actual Result

```text
true
test
```

Node's current documentation specifies that `AbortSignal.abort(reason)` creates an already-aborted signal whose reason is the supplied value. citeturn288849search2

---

## Exercise B — Timeout signal

Predict:

```js
const signal = AbortSignal.timeout(10);

console.log(signal.aborted);
```

Immediately after construction, before the timeout fires:

```text
false
```

After the timeout delay expires:

```text
true
```

Node documents `AbortSignal.timeout(delay)` as creating a signal aborted after the specified number of milliseconds. citeturn288849search2

---

## Exercise C — Promise retry

Predict:

```js
let attempts = 0;

async function operation() {
  attempts += 1;

  if (attempts < 3) {
    throw new Error("temporary");
  }

  return "ok";
}
```

With a correctly implemented retry loop allowing at least three attempts:

```text
ok
```

Final attempt count:

```text
3
```

### Rule

Retry behavior belongs to the application policy. JavaScript does not automatically retry rejected Promises.

---

## Exercise D — Timeout does not rewrite remote state

Conceptual:

```text
local request
→ remote charge succeeds
→ local timeout
```

Question:

```text
Did the remote charge necessarily fail?
```

Answer:

```text
No.
```

---

# 26. Mastery Exercises

## Track A — Core Theory

### A1

For:

```text
API
Database
Payment Provider
Queue
Cache
```

create a failure matrix:

| Dependency | Failure | Detection | Retry | Fallback | Blast Radius | Recovery |
|---|---|---|---|---|---|---|
| Database | ____ | ____ | ____ | ____ | ____ | ____ |
| Payment | ____ | ____ | ____ | ____ | ____ | ____ |
| Queue | ____ | ____ | ____ | ____ | ____ | ____ |
| Cache | ____ | ____ | ____ | ____ | ____ | ____ |

---

### A2

Design a deadline tree:

```text
request = 2s
 ├─ auth = 100ms
 ├─ inventory = 500ms
 └─ payment = remaining budget
```

Explain why independent 2-second child timeouts are wrong.

---

## Track B — Implementation

### B1 — Reliable HTTP client

Implement:

```text
timeout
abort
retry
backoff
jitter
retry classification
overall deadline
metrics
traces
```

### B2 — Circuit breaker

Implement:

```text
closed
open
half-open
```

with tests for:

```text
success
failure threshold
cooldown
recovery
concurrent probes
```

### B3 — Bulkhead

Create independent concurrency limits for:

```text
payments
search
notifications
```

Prove that one overloaded dependency cannot consume all concurrency.

### B4 — Reliable worker

Build:

```text
queue
consumer
retry
dead-letter
idempotency
replay
graceful shutdown
```

---

## Track C — Interview / Reasoning

### C1

A checkout service has:

```text
99.99% availability
```

but users report:

```text
duplicate charges
```

Explain why availability is not enough.

### C2

A dependency has:

```text
p50 = 50 ms
p99 = 5 s
```

Choose a timeout.

Defend using:

```text
business deadline
tail latency
retry
fallback
```

---

### C3

An API experiences:

```text
traffic spike × 10
```

Choose among:

```text
queue
load shedding
autoscaling
cache
rate limit
degradation
```

Explain what each solves and what it cannot solve.

---

# 27. Key Takeaways

1. Reliability is broader than uptime.
2. Reliability begins with failure modeling.
3. Every remote dependency needs a bounded waiting strategy.
4. Timeouts prevent indefinite resource occupation.
5. Abort signals do not automatically undo remote side effects.
6. Retries can improve availability or amplify outages.
7. Retry policy must include classification, bounds, and idempotency.
8. Backoff reduces synchronized retry pressure.
9. Jitter reduces synchronized retries across many clients.
10. Circuit breakers can contain dependency failure.
11. Bulkheads prevent one workload from consuming all resources.
12. Load shedding is often safer than infinite queueing.
13. Queues provide temporal decoupling, not infinite capacity.
14. Idempotency protects retryable workflows.
15. Deduplication needs durable state and retention strategy.
16. Graceful degradation must preserve correctness and security.
17. Fail-open and fail-closed choices are domain-specific.
18. Graceful shutdown is part of correctness.
19. Error budgets make reliability a measurable engineering trade-off.
20. Observability must remain useful during failure.
21. Reliability mechanisms add complexity and operational cost.
22. Database, queue, cache, and external-service failure require different strategies.
23. Reconciliation is essential when systems can partially complete work.
24. Reliability should be tested, not assumed.
25. Principal reliability engineering is about controlling failure propagation and preserving the most important invariants under stress.

---

# 28. Concept Connections

## Depends On

- **Chapter 29** — Errors
- **Chapter 30** — Resource Management
- **Chapter 31–40** — Async / Promises / Cancellation / Streaming / Concurrency
- **Chapter 45–48** — Memory / GC / Engine
- **Chapter 55** — HTTP Networking
- **Chapter 58–63** — Node.js / Streams / Workers / Process Lifecycle / Diagnostics
- **Chapter 78** — Production Architecture
- **Chapter 79** — API Design
- **Chapter 81** — Database Integration
- **Chapter 82** — API Architecture
- **Chapter 83** — Observability

## Builds Toward

- **Chapter 85** — Performance
- **Chapter 86** — Testing
- **Chapter 87** — Deterministic Async Testing
- **Chapter 88** — Debugging Methodology
- **Chapter 89** — Code Review / Refactoring
- **Chapter 94** — Compatibility Engineering
- **Chapter 101** — Real-world Production Scenarios
- **Chapter 107** — Job Queue
- **Chapter 108** — Cache System
- **Chapter 109** — Event-driven Application
- **Chapter 110** — Production JavaScript Backend
- **Chapter 111** — Large-scale JavaScript Platform
- **Chapter 121** — Principal System Design

## Related Concepts

```text
Reliability
  ├─ timeouts
  ├─ retries
  ├─ backoff
  ├─ jitter
  ├─ idempotency
  ├─ circuit breakers
  ├─ bulkheads
  ├─ load shedding
  ├─ backpressure
  ├─ graceful degradation
  ├─ queues
  ├─ reconciliation
  └─ recovery
```

## Concepts Revisited

```text
AbortController
Promises
streams
database transactions
connection pools
API contracts
service boundaries
observability
resource limits
process lifecycle
```

## Why This Chapter Matters Later

Reliability becomes concrete in the project chapters:

```text
job queues
caches
event-driven apps
production backend
large-scale platform
```

This chapter provides the mental model required to build those systems without relying on luck.

---

# 29. Completion Criteria

Mark:

```text
[~] In Progress
```

when you understand:

```text
timeouts
retries
idempotency
basic failure containment
graceful shutdown
```

Mark:

```text
[?] Needs Revision
```

when you repeatedly confuse:

```text
timeout vs cancellation
retry vs idempotency
availability vs reliability
queue vs infinite capacity
cache vs source of truth
fallback vs correctness
fail-open vs fail-safe
```

Mark:

```text
[+] Completed
```

when you can:

- model dependency failures;
- define timeout policies;
- design retry policies;
- implement idempotency;
- implement bounded concurrency;
- design graceful degradation;
- implement graceful shutdown;
- explain queue/backpressure behavior;
- design recovery/reconciliation.

Mark:

```text
[*] Mastered
```

only when you can:

- diagnose cascading failures;
- quantify retry amplification;
- design reliability boundaries across multiple services;
- design failure-safe workflows;
- choose between queueing and load shedding;
- define meaningful SLOs/error budgets;
- test failure scenarios deliberately;
- defend a reliability architecture at principal level.

Reading alone does not qualify as mastery.

---

# Chapter 84 — Revision / Retrieval Record

| Date | Retrieval Goal | Result | Status |
|---|---|---|---|
| ____ | Define reliability | ____ | `[ ]` |
| ____ | Model failure modes | ____ | `[ ]` |
| ____ | Design timeout/deadline strategy | ____ | `[ ]` |
| ____ | Design retries/backoff/jitter | ____ | `[ ]` |
| ____ | Explain idempotency | ____ | `[ ]` |
| ____ | Explain circuit breakers | ____ | `[ ]` |
| ____ | Explain bulkheads | ____ | `[ ]` |
| ____ | Explain load shedding | ____ | `[ ]` |
| ____ | Design graceful degradation | ____ | `[ ]` |
| ____ | Design graceful shutdown | ____ | `[ ]` |
| ____ | Design reconciliation | ____ | `[ ]` |
| ____ | Defend reliability architecture | ____ | `[ ]` |

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

1. Define reliability.
2. Reliability versus availability?
3. Why must remote work have timeouts?
4. Why do retries amplify outages?
5. What is idempotency?
6. What is exponential backoff?
7. Why add jitter?
8. What is a circuit breaker?
9. What is a bulkhead?
10. What is load shedding?
11. What happens when a queue grows faster than consumers can process?
12. How do you design graceful shutdown?
13. What is reconciliation?
14. How do you handle a lost response after remote success?
15. What makes a fallback unsafe?

---

# Chapter 84 — Canonical References and Source Discipline

## 1. Node.js AbortSignal

Node.js current documentation for `AbortController`/`AbortSignal` documents:

```text
AbortSignal.abort()
AbortSignal.timeout()
AbortSignal.any()
abort reason
throwIfAborted()
```

and describes the lifecycle of abort events. citeturn288849search2

Primary:

- https://nodejs.org/api/globals.html

Verify minimum supported Node version before adopting APIs in a library or production platform.

---

## 2. Node.js Timers

Node's timers/promises documentation supports cancellation of promise-based timers with `AbortSignal`. citeturn288849search8

Primary:

- https://nodejs.org/api/timers.html

---

## 3. Node.js HTTP

Node's current HTTP documentation includes request-level signal behavior and timeout/server controls. citeturn288849search12

Primary:

- https://nodejs.org/api/http.html

---

## 4. OpenTelemetry

OpenTelemetry's JavaScript documentation currently describes Node.js/browser support, tracing, metrics, logs, context, propagation, exporters, sampling, and instrumentation. It currently lists traces and metrics as stable and logs as development. citeturn288849search0

Primary:

- https://opentelemetry.io/docs/languages/js/

---

## 5. OpenTelemetry Error Handling

OpenTelemetry's specification states that instrumentation should not significantly change application behavior and says implementations must not throw unhandled exceptions at runtime. This supports the broader reliability principle that telemetry should fail independently of core application behavior. citeturn288849search10

Primary:

- https://opentelemetry.io/docs/specs/otel/error-handling/

---

## 6. OWASP API Security

OWASP's API Security Top 10 includes:

```text
Unrestricted Resource Consumption
Broken Authentication
Broken Object Level Authorization
Unsafe Consumption of APIs
```

among its major API risks. Reliability controls such as limits, timeouts, and admission policies should therefore be designed with security as well as availability in mind. citeturn277088search4

Primary:

- https://owasp.org/API-Security/

---

## Reliability Literature

Useful conceptual references:

- Google SRE — *Site Reliability Engineering*
- Google SRE — *The Site Reliability Workbook*
- Michael Nygard — *Release It!*
- Martin Kleppmann — *Designing Data-Intensive Applications*
- Sam Newman — *Building Microservices*
- Alex Xu — *System Design Interview*

These are architectural and operational references, not ECMAScript specifications.

---

## Source Discipline

Every reliability statement should be classified as:

```text
ECMAScript behavior
Node.js runtime behavior
HTTP behavior
Database behavior
Messaging behavior
Library behavior
Architecture choice
Operational policy
Security policy
```

Do not claim:

> “Abort means the remote work stopped.”

Instead:

> “The local operation was signaled for cancellation; the remote system must independently support cancellation or idempotent recovery.”

Do not claim:

> “Retries guarantee reliability.”

Instead:

> “Retries can recover certain transient failures when bounded, observable, and semantically safe.”

---

# Chapter 84 — Completion Snapshot

## Core Theory

- [ ] Reliability
- [ ] Availability
- [ ] Durability
- [ ] Resilience
- [ ] Fault tolerance
- [ ] Recoverability
- [ ] Failure modeling
- [ ] Failure domains
- [ ] Blast radius
- [ ] Timeouts
- [ ] Deadlines
- [ ] Cancellation
- [ ] Retries
- [ ] Backoff
- [ ] Jitter
- [ ] Retry budgets
- [ ] Idempotency
- [ ] Deduplication
- [ ] Circuit breakers
- [ ] Bulkheads
- [ ] Load shedding
- [ ] Backpressure
- [ ] Queues
- [ ] Graceful degradation
- [ ] Fail-open / fail-closed
- [ ] Graceful shutdown
- [ ] Health/readiness
- [ ] SLO
- [ ] Error budget
- [ ] Reconciliation
- [ ] Fault injection

## Implementation

- [ ] Timeout-aware HTTP client
- [ ] Abort support
- [ ] Retry classification
- [ ] Backoff
- [ ] Jitter
- [ ] Retry budget
- [ ] Idempotency store
- [ ] Deduplication
- [ ] Circuit breaker
- [ ] Bulkhead
- [ ] Concurrency limiter
- [ ] Rate limiter
- [ ] Load shedding
- [ ] Queue
- [ ] Dead-letter queue
- [ ] Replay
- [ ] Graceful shutdown
- [ ] Health/readiness
- [ ] Recovery workflow
- [ ] Reconciliation
- [ ] Failure metrics
- [ ] Distributed tracing
- [ ] Fault injection

## Interview / Reasoning

- [ ] Explain reliability vs availability
- [ ] Explain timeout
- [ ] Explain retry amplification
- [ ] Explain idempotency
- [ ] Explain backoff/jitter
- [ ] Explain circuit breaker
- [ ] Explain bulkhead
- [ ] Explain load shedding
- [ ] Design graceful degradation
- [ ] Design recovery/reconciliation
- [ ] Analyze cascading failure
- [ ] Design SLO/error budget
- [ ] Defend failure-handling architecture

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

Given an unfamiliar production JavaScript system, can you answer:

```text
What are the system's critical user outcomes?
What failures can occur?
Which failures are transient?
Which are permanent?
Where are the timeout boundaries?
What is the overall deadline?
Where are retries?
Who owns retry policy?
Can operations duplicate?
Where is idempotency enforced?
Where is deduplication enforced?
Where are concurrency limits?
Where is backpressure?
Where is load shedding?
Where is queue capacity bounded?
What happens when dependencies are slow?
What happens when dependencies are unavailable?
What happens when responses are lost?
What happens when messages duplicate?
What happens when workers crash mid-operation?
What happens during deployment?
What happens during shutdown?
What data can be stale?
What features can degrade?
Which features must fail closed?
How is recovery initiated?
How is reconciliation performed?
What does the SLO measure?
What is the error budget?
How is the system observed during failure?
What is the largest blast radius?
What reliability mechanism would you remove if it adds more complexity than value?
```

A principal reliability engineer does not attempt to make failure impossible.

They design the system so that **failure is bounded, observable, recoverable, and unable to silently violate the most important invariants.**

---

## Principal Reliability Decision Framework

For every significant reliability mechanism, record:

```text
Failure:
Trigger:
Affected Capability:
User Impact:
Detection:
Blast Radius:
Timeout:
Deadline:
Retryability:
Idempotency:
Backoff:
Jitter:
Concurrency Limit:
Queue:
Fallback:
Fail-Open/Fail-Closed:
Observability:
Recovery:
Reconciliation:
Security Impact:
Operational Cost:
Complexity:
Decision:
Removal Criteria:
Revisit Trigger:
```

Then ask:

> **Does this mechanism reduce the probability or impact of an important failure enough to justify the state, complexity, latency, and operational cost it introduces?**

That is the principal-level reliability question.