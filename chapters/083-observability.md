# Chapter 83 — Observability

> **Part XV — Production JavaScript**  
> **Status:** `[ ] Not Started`  
> **Depth Target:** Principal / Staff-level reasoning  
> **Primary Focus:** Designing logs, metrics, traces, diagnostics, alerts, and operational feedback loops for production JavaScript systems.

---

# 1. Learning Objectives

By the end of this chapter, you should be able to:

1. Explain observability as a property of a production system rather than a collection of monitoring tools.
2. Distinguish observability from monitoring, debugging, logging, profiling, and alerting.
3. Explain the three classical telemetry signals: logs, metrics, and traces.
4. Understand when each signal is useful and when it is insufficient.
5. Design structured application logs.
6. Define event names, log levels, and stable machine-readable fields.
7. Protect sensitive information in telemetry.
8. Design useful metrics.
9. Distinguish counters, gauges, histograms, and rates.
10. Reason about metric cardinality.
11. Design request and dependency traces.
12. Understand spans, trace IDs, span IDs, context, and propagation.
13. Correlate logs, metrics, and traces.
14. Design observability for asynchronous queues and background jobs.
15. Design observability for databases and external APIs.
16. Design health checks, readiness, liveness, and startup diagnostics.
17. Understand SLI, SLO, SLA, error budget, and alerting relationships.
18. Design alerts around user impact instead of raw infrastructure noise.
19. Diagnose latency, errors, saturation, and traffic.
20. Diagnose p95/p99 latency and tail behavior.
21. Detect queue growth and backpressure failures.
22. Detect database pool saturation and dependency bottlenecks.
23. Design runtime diagnostics for Node.js.
24. Use profiling and heap analysis appropriately.
25. Understand sampling and telemetry cost.
26. Design observability for multi-tenant systems without leaking tenant data.
27. Design observability during incidents.
28. Build an observable Node.js service.
29. Test observability itself.
30. Review telemetry for correctness, cardinality, security, and operational value.
31. Defend observability architecture at principal-engineer level.

---

# 2. Prerequisites

You should understand:

- JavaScript functions, modules, promises, async/await.
- Node.js runtime behavior.
- HTTP and API architecture.
- Database integration.
- Concurrency and streams.
- Production architecture.
- Security boundaries.
- Basic distributed-system behavior.

Recommended prior chapters:

- **29** — Errors / Error Handling
- **31–40** — Async / Concurrency / Streaming
- **45–48** — Memory / Garbage Collection / Engine
- **55–57** — Networking / Browser Security / Security Engineering
- **58–63** — Node.js Architecture / Core APIs / Streams / Workers / Lifecycle / Diagnostics
- **70** — Source Maps / Production Debugging
- **78** — Production JavaScript Architecture
- **79** — API Design
- **80** — Library Authoring
- **81** — Database Integration
- **82** — API Architecture

---

# 3. What Is It?

Observability is the ability to infer the internal state and behavior of a system from the outputs it produces.

Production systems emit evidence:

```text
logs
metrics
traces
events
profiles
heap snapshots
diagnostic reports
health states
```

Observability turns those outputs into answers to questions such as:

```text
What is broken?
Who is affected?
When did it begin?
Which dependency is responsible?
Is the problem global or isolated?
What changed?
Is latency or error rate increasing?
Is capacity being exhausted?
What should we do next?
```

---

## 3.1 Observability versus monitoring

Monitoring traditionally asks:

```text
Is known condition X happening?
```

Observability asks:

```text
Can we investigate conditions we did not fully anticipate?
```

Example:

Monitoring:

```text
CPU > 80%
```

Observability:

```text
Why are checkout requests slow?
Which dependency contributes most of the latency?
Which tenant is affected?
Did the regression start after deployment 184?
```

Monitoring is useful.

Observability is broader.

---

# 4. Why Does It Exist?

Production software behaves differently from local development.

A failure may depend on:

```text
traffic shape
specific tenant
database state
network latency
timing
concurrency
deployment version
dependency behavior
resource saturation
```

You cannot attach a debugger to every production process and inspect everything manually.

You need the system to produce enough evidence to reconstruct what happened.

---

# 5. Mental Model

Think of observability as an evidence system:

```text
                  Production System
                         │
        ┌────────────────┼────────────────┐
        │                │                │
       Logs           Metrics           Traces
        │                │                │
        └────────────────┼────────────────┘
                         │
                    Correlation
                         │
                  Diagnostics
                         │
                  Investigation
                         │
                    Decision
```

Add resource-level evidence:

```text
CPU
memory
GC
event loop
database
network
queues
external dependencies
```

The objective is not “collect everything.”

The objective is:

> **Collect the minimum high-value evidence necessary to understand important production behavior.**

---

# 6. Core Rules

## Rule 1 — Every important business workflow needs an observable path

Example:

```text
place order
  ↓
authorization
  ↓
inventory
  ↓
payment
  ↓
database
  ↓
event
```

An operator should be able to follow the path.

---

## Rule 2 — Structured telemetry beats ad-hoc strings

Weak:

```text
"payment failed"
```

Stronger:

```json
{
  "event": "payment.authorize.failed",
  "payment_id": "pay_123",
  "error_code": "TIMEOUT",
  "dependency": "payment-provider"
}
```

---

## Rule 3 — Stable names matter

Telemetry consumers depend on field names.

Avoid changing:

```text
paymentId
payment_id
payment.id
```

randomly across services.

---

## Rule 4 — Cardinality is a budget

Do not use unbounded values as metric labels.

Dangerous:

```text
user_id
request_id
full_url
email
stack_trace
```

as dimensions of every metric.

---

## Rule 5 — Never log secrets

Avoid:

```text
password
access token
session secret
API key
private credential
```

Telemetry is a copy of application information and therefore another security surface.

---

## Rule 6 — Logs, metrics, and traces complement each other

Use:

```text
metrics → detect
traces → localize
logs → explain
profiles → optimize
```

This is a useful heuristic, not an absolute rule.

---

## Rule 7 — Measure user-impacting outcomes

Prefer:

```text
checkout success rate
request latency
queue age
payment authorization failures
```

over only:

```text
CPU utilization
```

---

## Rule 8 — Alert on actionable conditions

Every alert should have:

```text
condition
impact
owner
runbook
action
```

---

## Rule 9 — Telemetry must not become the outage

Telemetry pipelines consume:

```text
CPU
memory
network
storage
```

Design them with limits and failure behavior.

---

## Rule 10 — Observability must work during failure

An observability system that only works when the system is healthy is incomplete.

---

# 7. Syntax

## 7.1 Structured logging

```js
logger.info({
  event: "order.created",
  orderId: order.id,
  tenantId: context.tenantId,
  requestId: context.requestId,
});
```

Avoid:

```js
console.log(
  `Order ${order.id} created for ${context.tenantId}`,
);
```

unless the environment provides strong structured processing.

---

## 7.2 Metrics conceptually

```js
ordersCreated.add(1, {
  region: "ap-south-1",
});
```

Keep metric attributes bounded.

---

## 7.3 Tracing conceptually

```js
const span = tracer.startSpan("orders.create");

try {
  // application work
} catch (error) {
  span.recordException(error);
  throw error;
} finally {
  span.end();
}
```

The exact API depends on the telemetry library and version.

---

# 8. Basic Examples

## Example 1 — Log

```json
{
  "timestamp": "2026-09-10T12:00:00.000Z",
  "level": "info",
  "event": "order.created",
  "service": "orders-api",
  "version": "2026.09.10",
  "request_id": "req_42",
  "order_id": "ord_123"
}
```

---

## Example 2 — Metric

Conceptually:

```text
http.server.request.duration
```

with bounded attributes such as:

```text
method
route
status
```

OpenTelemetry defines semantic conventions for HTTP spans, metrics, and exceptions, providing standardized names and attributes that improve cross-service interpretation. citeturn412332search0turn412332search10

---

## Example 3 — Trace

```text
Trace: 7b...
|
├─ HTTP POST /orders
|   |
|   ├─ DB INSERT orders
|   |
|   ├─ DB UPDATE inventory
|   |
|   └─ publish OrderCreated
|
└─ payment-worker
    |
    └─ POST payment-provider
```

A trace can reveal where total request time is spent.

---

# 9. Execution Walkthrough

Suppose:

```http
POST /orders
```

The telemetry path should conceptually be:

```text
incoming request
  ↓
create/continue trace
  ↓
request context
  ↓
HTTP server span
  ↓
order use-case span
  ↓
database span
  ↓
payment dependency span
  ↓
event/message span
  ↓
response
```

Associated logs should include correlation fields:

```text
trace_id
request_id
service
route
operation
error
```

Metrics capture aggregate behavior:

```text
request rate
error rate
duration
dependency duration
queue depth
database pool saturation
```

---

# 10. Internal Mechanics

## 10.1 Logs

A log is a discrete record of an event.

Good structured fields:

```text
timestamp
level
event
service
version
request_id
trace_id
operation
error.type
error.code
```

OpenTelemetry semantic conventions include standardized error-related attributes such as `error.type`, `exception.message`, `exception.stacktrace`, and service identity fields. citeturn412332search8

---

## 10.2 Metrics

Metrics aggregate observations.

Common conceptual instruments:

```text
Counter
Gauge / UpDown-style value
Histogram
```

### Counter

Counts events:

```text
orders_created_total
```

### Gauge-like state

Represents a current value:

```text
queue_depth
active_connections
```

### Histogram

Captures a distribution:

```text
request_duration
database_query_duration
```

Histograms are especially useful for latency distributions.

---

## 10.3 Traces

A trace represents a causal operation across components.

```text
trace
  ↓
spans
  ↓
events / attributes
```

Span attributes should be useful and appropriately bounded.

---

## 10.4 Context propagation

A request may become:

```text
HTTP
  ↓
service A
  ↓
queue
  ↓
worker
  ↓
service B
```

The causal context must survive the boundary when possible.

OpenTelemetry JavaScript provides APIs/SDKs for context and propagation and supports instrumentation for Node.js and browser environments. citeturn412332search1

---

# 11. ECMAScript / Specification Semantics

Observability is mostly above the ECMAScript language layer.

Separate:

```text
ECMAScript
  → language behavior

Node.js
  → runtime diagnostics and APIs

Telemetry SDK
  → instrumentation/collection

OpenTelemetry
  → standardized observability model

Application
  → business events and operational policies
```

For example:

```js
console.log("x");
```

is a JavaScript/host operation.

It is not automatically:

```text
structured logging
distributed tracing
metric emission
```

Similarly:

```js
await operation();
```

does not automatically produce useful telemetry.

Instrumentation must be designed.

---

# 12. Advanced Behavior

## 12.1 High-value log taxonomy

Useful categories:

```text
business event
security event
operational event
dependency failure
lifecycle event
diagnostic event
```

Avoid logging every internal branch.

---

## 12.2 Log levels

Common conceptual levels:

```text
trace
debug
info
warn
error
fatal
```

Do not decide severity solely from the presence of an exception.

Ask:

```text
Does the service remain healthy?
Does a user-facing operation fail?
Does an operator need to act?
Is the condition expected?
```

---

## 12.3 Metric cardinality

Suppose:

```text
orders_total{tenant_id="..."}
```

and there are:

```text
100,000 tenants
```

A metric system may face enormous time-series cardinality.

Prefer bounded dimensions:

```text
region
service
route
status_class
```

Use logs/traces for high-cardinality investigation values.

---

## 12.4 Histograms and percentiles

Average latency:

```text
mean = 100 ms
```

can hide:

```text
p99 = 2 seconds
```

For user experience, tail latency can matter more than mean latency.

---

## 12.5 Exemplars and correlation

A metric spike can be much more useful when it links to a representative trace or investigation context.

Conceptually:

```text
metric spike
  ↓
trace exemplar
  ↓
specific slow request
  ↓
database span
```

---

# 13. Edge Cases

## 13.1 Logging inside a hot loop

```js
for (const item of items) {
  logger.info({ item });
}
```

Can produce massive telemetry volume.

Ask:

```text
Can aggregate metrics represent this?
Should debug logging be sampled?
Should only failures be logged?
```

---

## 13.2 Logging secrets

```js
logger.error({
  request,
});
```

may accidentally serialize authorization headers or credentials.

Never assume object logging is safe.

---

## 13.3 Dynamic URLs as metric labels

Bad:

```text
http.server.duration{url="/users/123"}
http.server.duration{url="/users/124"}
```

This creates unbounded dimensions.

Prefer route templates:

```text
/users/:id
```

OpenTelemetry's HTTP conventions explicitly distinguish standardized HTTP attributes; use semantic conventions rather than inventing unbounded attributes. citeturn412332search0

---

## 13.4 Missing context after queue hop

A queue consumer logs:

```text
payment failed
```

but no:

```text
trace_id
order_id
job_id
```

Now the operator cannot reconstruct the workflow.

---

## 13.5 Clock skew

Distributed machines can disagree slightly about wall-clock time.

Tracing systems therefore use causal context and span structure rather than relying only on timestamps to infer relationships.

---

# 14. Common Misconceptions

### “More logs means better observability.”

No. More data can mean more noise and cost.

### “Metrics replace logs.”

No. Metrics summarize; logs can capture detailed event context.

### “Traces replace logs.”

No. Traces provide causality and timing; logs often carry detailed application context.

### “CPU is observability.”

CPU is one signal of resource behavior.

### “100% test coverage guarantees observability.”

No. Telemetry correctness needs dedicated testing.

### “Every exception should be an error alert.”

No. Some exceptions are expected or handled.

### “Every field should be a metric label.”

No. High-cardinality dimensions can become operationally expensive.

### “Observability means OpenTelemetry.”

No. OpenTelemetry is an observability framework/ecosystem; observability is the broader engineering capability.

---

# 15. Common Mistakes

## Mistake 1 — String-only logs

```text
Failed to process request
```

No useful context.

---

## Mistake 2 — No stable event names

```text
order-created
order_created
created_order
```

Fragmented search and dashboards.

---

## Mistake 3 — High-cardinality metrics

```text
user_id
request_id
session_id
```

as metric dimensions.

---

## Mistake 4 — Missing trace propagation

Each service creates a new unrelated trace.

---

## Mistake 5 — Alerting on raw CPU

A CPU spike can be harmless; user-facing failures may occur at normal CPU.

---

## Mistake 6 — No deployment/version context

A regression cannot be correlated with the code version that introduced it.

---

## Mistake 7 — Telemetry pipeline has no limits

A logging storm can consume the same resources needed for application recovery.

---

# 16. Comparison With Related Concepts

| Signal / Technique | Best For | Limitation |
|---|---|---|
| Logs | Detailed event context | Volume/noise |
| Metrics | Aggregate trends/alerting | Weak detail |
| Traces | Causality/latency | Sampling/storage cost |
| Profiles | CPU/allocation optimization | Not general event context |
| Health check | Liveness/readiness | Very limited diagnosis |
| Heap snapshot | Memory analysis | Snapshot overhead |
| Diagnostic report | Runtime failure analysis | Point-in-time |
| Alert | Immediate action | Poor exploration tool |
| Dashboard | Human monitoring | Often symptom-focused |
| Synthetic test | User-path verification | Limited production context |

---

# 17. Performance Considerations

Observability consumes resources.

Potential costs:

```text
CPU
memory
serialization
network
storage
GC
disk I/O
```

---

## 17.1 Logging cost

JSON serialization can allocate objects and strings.

High-volume logs can increase:

```text
CPU
memory
GC
network
```

Do not instrument blindly.

---

## 17.2 Trace cost

Tracing may create:

```text
span objects
attributes
events
context state
export buffers
```

Use sampling when volume requires it.

OpenTelemetry JavaScript documents sampling as a capability for reducing telemetry creation and export volume. citeturn412332search1

---

## 17.3 Metrics cost

Metric cardinality can dominate cost.

Ten metrics with 10 values each:

```text
10 × 10
```

is easy.

A metric with:

```text
user_id = 10 million values
```

can be disastrous.

---

## 17.4 Tail-focused investigation

Use metrics to detect:

```text
p99 latency increased
```

then traces to locate:

```text
database span = 1.8 s
```

and logs to explain:

```text
query timeout / lock contention
```

This avoids searching millions of log lines.

---

# 18. Memory Considerations

Telemetry can create retention.

Potential sources:

```text
in-memory log buffers
span batches
queued exports
metric aggregators
trace context
diagnostic state
```

Design bounded queues.

Never allow:

```text
telemetry_queue.push(event);
```

to grow indefinitely when the exporter is unavailable.

---

## 18.1 Backpressure

If:

```text
application emits 100k telemetry events/sec
exporter handles 20k/sec
```

something must happen:

```text
sample
drop
buffer
block
shed
```

For most production systems, blocking critical application work indefinitely to emit telemetry is dangerous.

Telemetry should be useful without becoming a dependency that can take the application down.

---

# 19. Security Considerations

Observability is a data-exfiltration surface.

## 19.1 Sensitive data

Avoid:

```text
passwords
tokens
authorization headers
payment secrets
private keys
PII where not required
```

---

## 19.2 Tenant isolation

A multi-tenant observability platform should prevent:

```text
tenant A
  ↓
view tenant B telemetry
```

Telemetry access is an authorization problem.

---

## 19.3 Log injection

If attackers can control log content:

```text
username = "admin\nERROR database compromised"
```

systems that treat logs as raw text can become difficult to trust.

Structured logging reduces many formatting problems.

---

## 19.4 Redaction

Build explicit redaction:

```js
function redact(headers) {
  const copy = { ...headers };

  delete copy.authorization;
  delete copy.cookie;

  return copy;
}
```

Apply at the telemetry boundary.

---

# 20. Production Usage

## 20.1 Production observability architecture

```text
Application
   │
   ├── logs ───────┐
   │               │
   ├── metrics ────┼──→ telemetry pipeline
   │               │
   └── traces ─────┘
                         │
                  ┌──────┼──────┐
                  │      │      │
               storage dashboards alerts
```

Keep application instrumentation separate from telemetry storage concerns.

---

## 20.2 OpenTelemetry

OpenTelemetry JavaScript currently provides instrumentation and SDKs for Node.js and browser environments; its documented functional status lists traces and metrics as stable and logs as development. Browser client instrumentation is described as experimental/mostly unspecified, so do not assume Node and browser support have identical maturity. citeturn412332search1

OpenTelemetry semantic conventions define common attributes and naming for HTTP, databases, messaging, RPC, and other domains. citeturn412332search12turn412332search0turn412332search3

---

## 20.3 HTTP telemetry

Prefer standardized route-level telemetry:

```text
HTTP method
route
status
duration
error
server/service identity
```

OpenTelemetry's HTTP conventions define semantic conventions for HTTP spans, metrics, and exceptions and distinguish stabilized versus evolving convention sets. citeturn412332search0turn412332search7

---

## 20.4 Database telemetry

Capture useful database dimensions:

```text
operation
system
database
duration
error
result characteristics where safe
```

OpenTelemetry maintains database semantic conventions for database spans, metrics, and exceptions; verify the convention version emitted by your instrumentation before changing dashboards or alerts. citeturn412332search3

---

## 20.5 Messaging telemetry

For queues and pub/sub:

```text
queue
operation
consumer
producer
message ID
processing duration
failure
retry
```

OpenTelemetry currently publishes messaging semantic conventions, with parts of the model still marked as development; verify the version/stability of the conventions used by your instrumentation. citeturn412332search2

---

## 20.6 Health

Useful endpoints:

```text
/live
/ready
```

But health checks should be designed around lifecycle semantics.

Possible meaning:

```text
live = process exists and can respond
ready = process can safely receive work
```

Do not automatically query every dependency on every health request.

---

## 20.7 SLI / SLO

An SLI measures a property:

```text
successful requests / valid requests
```

An SLO sets a target:

```text
99.9% success over a defined window
```

An SLA may be a contractual commitment.

These are not interchangeable.

---

## 20.8 Error budget

If the SLO is:

```text
99.9%
```

the system has a bounded amount of allowable unreliability over the chosen measurement window.

That budget can inform:

```text
release velocity
risk tolerance
incident response
capacity work
```

---

# 21. Implementation From Scratch

Build observability into a Node.js order service.

## Stage 1 — Guided logging

Create:

```js
function log(level, event, fields = {}) {
  process.stdout.write(
    JSON.stringify({
      time: new Date().toISOString(),
      level,
      event,
      ...fields,
    }) + "\n",
  );
}
```

Use:

```js
log("info", "order.created", {
  orderId,
  requestId,
});
```

---

## Stage 2 — Request context

Create:

```js
const context = {
  requestId,
  traceId,
  tenantId,
  actorId,
};
```

Pass or propagate it explicitly.

---

## Stage 3 — Metrics

Track:

```text
request_count
request_duration
request_errors
database_duration
external_dependency_duration
queue_depth
queue_age
```

Use bounded dimensions.

---

## Stage 4 — Tracing

Instrument:

```text
HTTP request
use case
database operation
external API
queue publish
queue consume
```

Add:

```text
trace_id
span_id
```

and propagate context across boundaries.

---

## Stage 5 — Runtime diagnostics

Track:

```text
event loop delay
heap usage
GC behavior
open handles
socket counts
worker state
```

Node's diagnostic APIs and runtime tooling can provide more specialized evidence than application logs; choose the least invasive mechanism that answers the debugging question.

---

## Stage 6 — Failure resilience

Simulate:

```text
telemetry backend unavailable
telemetry exporter slow
log volume spike
trace sampling
database outage
queue outage
```

Ensure:

```text
application continues safely
telemetry degrades gracefully
buffers remain bounded
critical workflows are not blocked indefinitely
```

---

# 22. Debugging Exercises

## Exercise 1 — High p99 latency

Metrics:

```text
p50 = 60 ms
p95 = 100 ms
p99 = 2.4 s
```

Traces show:

```text
database span = 2.1 s
```

Determine:

```text
where to investigate next
what metrics to inspect
what logs to search
```

---

## Exercise 2 — Error rate normal, queue age rising

Requests succeed, but:

```text
queue age
↑
```

Explain why the API can look healthy while background processing is failing.

---

## Exercise 3 — CPU normal, memory rising

```text
CPU = 40%
heap = continuously rising
```

Use observability evidence to investigate:

```text
cache
retained objects
listener leaks
queues
buffers
```

---

## Exercise 4 — Trace missing after queue

HTTP request has:

```text
trace_id = A
```

worker logs have:

```text
trace_id = none
```

Find the context propagation boundary.

---

## Exercise 5 — Dashboard overload

A team created:

```text
100 dashboards
900 alerts
```

but incidents remain hard to diagnose.

Identify the observability-design failure.

---

## Exercise 6 — Cardinality explosion

A metric suddenly has:

```text
20 million time series
```

Find likely attribute sources.

---

## Exercise 7 — Logging outage

The logging collector becomes unavailable.

Application memory starts rising.

Find the telemetry backpressure problem.

---

## Exercise 8 — Sensitive telemetry

Logs contain:

```text
Authorization: Bearer ...
```

Design:

```text
redaction
retroactive access controls
rotation if exposed
audit
prevention
```

---

# 23. Code Review Exercise

Review:

```js
app.use(async (req, res, next) => {
  console.log({
    url: req.url,
    headers: req.headers,
    body: req.body,
  });

  const start = Date.now();

  try {
    await next();
  } finally {
    console.log(
      `request took ${Date.now() - start}ms`
    );
  }
});
```

Identify at least 20 problems.

Consider:

```text
secret leakage
PII
unbounded body size
raw URL cardinality
no route normalization
no request ID
no trace context
no status
no user impact
no error taxonomy
no sampling
no level policy
no structured event name
no duration histogram
Date.now resolution/clock semantics
telemetry failure handling
no environment/version
no tenant security model
no redaction
```

---

# 24. Interview Questions

## Fundamental

1. What is observability?
2. Observability versus monitoring?
3. Logs versus metrics versus traces?
4. What is a trace?
5. What is a span?
6. What is a metric cardinality problem?
7. What is a histogram?
8. What is a correlation ID?
9. What is a health check?
10. What is an SLO?

## Intermediate

11. What should be logged?
12. What should never be logged?
13. How do you design structured logging?
14. Why is `user_id` dangerous as a metric label?
15. How do you correlate logs and traces?
16. How do traces work across asynchronous boundaries?
17. What is an error budget?
18. How do you alert on user impact?
19. How do you monitor queues?
20. How do you monitor a database dependency?

## Advanced

21. How would you design observability for 100 services?
22. How do you control telemetry cost?
23. How do you handle high-cardinality investigation?
24. What should be sampled?
25. How do you instrument a shared JavaScript library?
26. How do you test telemetry?
27. How do you handle telemetry exporter failure?
28. How do you detect tail latency?
29. How do you diagnose memory leaks with runtime diagnostics?
30. How do you design observability for async workflows?

## Principal

31. What telemetry is essential versus noise?
32. How do you define observability standards across teams?
33. How do you prevent dashboards from becoming operational debt?
34. How do you design tenant-safe telemetry?
35. How do you make incident response faster through architecture?
36. What should be measured as an SLI?
37. How do you choose between head and tail sampling?
38. How do you keep observability from impacting application reliability?
39. What evidence tells you the system is healthy?
40. How would you redesign a system that is “fully instrumented” but still difficult to debug?

---

# 25. Predict-the-Output Exercises

## Exercise A — Structured event

Predict:

```js
const fields = {
  event: "order.created",
  orderId: "ord_1",
};

console.log(
  JSON.stringify(fields)
);
```

### Actual Result

```text
{"event":"order.created","orderId":"ord_1"}
```

The exact string representation is produced by JSON serialization from the object's enumerable own properties.

### Principle

A structured log is machine-readable data, not merely a sentence.

---

## Exercise B — Counter versus state

Predict:

```js
let count = 0;

function create() {
  count += 1;
}

create();
create();

console.log(count);
```

### Actual Result

```text
2
```

### Observability Lesson

A counter can represent cumulative events, while a gauge-like measurement represents current state.

---

## Exercise C — Async context is not automatic business correlation

Predict conceptually:

```js
const traceId = "abc";

Promise.resolve().then(() => {
  console.log(traceId);
});
```

### Actual Result

```text
abc
```

The lexical binding remains available.

But distributed tracing context is a separate abstraction. It requires propagation/context infrastructure rather than assuming every asynchronous callback is automatically associated with your intended trace in every runtime/library situation.

---

## Exercise D — Unbounded labels

Conceptual:

```text
request_count{request_id="req-1"} = 1
request_count{request_id="req-2"} = 1
request_count{request_id="req-3"} = 1
...
```

Ask:

> Is this a good metric design?

### Answer

Usually no.

The request ID is useful for logs/traces, but generally a poor high-cardinality metric dimension.

---

# 26. Mastery Exercises

## Track A — Core Theory

### A1

Design a telemetry model for:

```text
Orders
Payments
Inventory
Notifications
```

Define:

```text
logs
metrics
traces
health
alerts
SLIs
SLOs
```

### A2

For each field, classify it:

```text
metric label
log field
trace attribute
secret
PII
```

Defend the classification.

---

## Track B — Implementation

### B1 — Observable API

Build a Node API with:

```text
structured logs
request IDs
trace IDs
HTTP metrics
duration histograms
error metrics
database spans
dependency spans
```

### B2 — Background worker

Add:

```text
queue depth
queue age
job processing duration
job failures
retry count
dead-letter count
trace propagation
```

### B3 — Runtime diagnostics

Instrument:

```text
heap usage
event-loop delay
active handles
GC-related evidence
```

Then reproduce a memory leak and diagnose it.

### B4 — OpenTelemetry

Implement OpenTelemetry-based tracing and metrics for your service.

Use current semantic conventions for HTTP and database operations, and verify the convention stability/version emitted by your instrumentation before building long-lived dashboards. citeturn412332search0turn412332search3

---

## Track C — Interview / Reasoning

### C1

A service has:

```text
CPU 35%
memory 70%
p95 150 ms
p99 4 s
error rate 0.1%
```

What do you investigate first?

Defend the order.

### C2

A system has no tracing but excellent logs.

Design an incremental migration that minimizes risk.

### C3

An organization has:

```text
50 teams
100 services
10 telemetry backends
```

Design a standard observability platform.

---

# 27. Key Takeaways

1. Observability is the ability to infer system behavior from emitted evidence.
2. Monitoring detects known conditions; observability supports investigation of unknown conditions.
3. Logs provide detailed event context.
4. Metrics provide aggregate trends and alerting signals.
5. Traces provide cross-component causality and timing.
6. Profiles and runtime diagnostics answer deeper performance/memory questions.
7. Structured telemetry is easier to search and automate.
8. Stable field names are contracts.
9. Metric cardinality must be controlled.
10. Request IDs and trace IDs should not be confused.
11. Context must be propagated across service and asynchronous boundaries.
12. Telemetry must respect security and tenant boundaries.
13. Observability consumes CPU, memory, network, and storage.
14. Telemetry pipelines need bounded buffering and failure behavior.
15. Alerts should correspond to user or system impact.
16. SLIs measure; SLOs target; SLAs can be contractual commitments.
17. Error budgets turn reliability into an operational decision framework.
18. Health checks are not incident diagnosis.
19. OpenTelemetry provides a standardized ecosystem for JavaScript telemetry; current JavaScript documentation lists traces and metrics as stable and logs as development. citeturn412332search1
20. Observability is successful when an operator can move from symptom to cause to action quickly.

---

# 28. Concept Connections

## Depends On

- **Chapter 29** — Errors
- **Chapter 31–40** — Async / Concurrency / Streaming
- **Chapter 45–48** — Memory / GC / Engine
- **Chapter 58–63** — Node.js / Diagnostics / Lifecycle
- **Chapter 70** — Production Debugging
- **Chapter 78** — Production Architecture
- **Chapter 79** — API Design
- **Chapter 81** — Database Integration
- **Chapter 82** — API Architecture

## Builds Toward

- **Chapter 84** — Reliability
- **Chapter 85** — Performance
- **Chapter 86** — Testing
- **Chapter 87** — Deterministic Async Testing
- **Chapter 88** — Debugging Methodology
- **Chapter 89** — Code Review / Refactoring
- **Chapter 101** — Real-world Production Scenarios
- **Chapter 107** — Job Queue
- **Chapter 109** — Event-driven Application
- **Chapter 110** — Production JavaScript Backend
- **Chapter 111** — Large-scale JavaScript Platform
- **Chapter 121** — Principal System Design

## Related Concepts

```text
Observability
  ├─ logs
  ├─ metrics
  ├─ traces
  ├─ profiling
  ├─ diagnostics
  ├─ health
  ├─ alerts
  ├─ SLI/SLO
  └─ incident response
```

## Concepts Revisited

```text
errors
async context
HTTP
database calls
queues
process lifecycle
memory
performance
security
distributed boundaries
```

## Why This Chapter Matters Later

The next production chapters focus on:

```text
reliability
performance
testing
debugging
```

Observability is what lets you determine whether those properties actually hold in production.

---

# 29. Completion Criteria

Mark:

```text
[~] In Progress
```

when you understand:

```text
logs
metrics
traces
basic alerting
```

Mark:

```text
[?] Needs Revision
```

when you repeatedly confuse:

```text
observability vs monitoring
logs vs metrics
metrics vs traces
request ID vs trace ID
metric dimensions vs log fields
liveness vs readiness
SLI vs SLO vs SLA
```

Mark:

```text
[+] Completed
```

when you can:

- design structured logs;
- define useful metrics;
- trace a request across services;
- propagate context;
- monitor queues and database dependencies;
- design alerts;
- implement runtime diagnostics;
- protect telemetry from secrets;
- handle telemetry backpressure.

Mark:

```text
[*] Mastered
```

only when you can:

- diagnose a production incident from telemetry;
- design organization-wide observability standards;
- control cardinality and telemetry cost;
- design tenant-safe observability;
- choose sampling strategies;
- define meaningful SLOs;
- critique an instrumented system that is still difficult to debug;
- defend observability architecture at principal level.

Reading alone does not qualify as mastery.

---

# Chapter 83 — Revision / Retrieval Record

| Date | Retrieval Goal | Result | Status |
|---|---|---|---|
| ____ | Define observability | ____ | `[ ]` |
| ____ | Compare logs/metrics/traces | ____ | `[ ]` |
| ____ | Explain cardinality | ____ | `[ ]` |
| ____ | Design structured logging | ____ | `[ ]` |
| ____ | Design HTTP metrics | ____ | `[ ]` |
| ____ | Explain trace propagation | ____ | `[ ]` |
| ____ | Design queue observability | ____ | `[ ]` |
| ____ | Define SLI/SLO/error budget | ____ | `[ ]` |
| ____ | Diagnose a production incident | ____ | `[ ]` |
| ____ | Defend telemetry architecture | ____ | `[ ]` |

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

1. Define observability.
2. Logs versus metrics versus traces?
3. What is cardinality?
4. Why are request IDs poor metric labels?
5. What makes a log useful?
6. How does trace context cross a service boundary?
7. What should never be logged?
8. How do you monitor queues?
9. SLI versus SLO versus SLA?
10. What happens if the telemetry backend is unavailable?
11. How do you investigate p99 latency?
12. How do you detect memory leaks?

---

# Chapter 83 — Canonical References and Source Discipline

## 1. OpenTelemetry JavaScript

OpenTelemetry's current JavaScript documentation covers Node.js and browser telemetry, instrumentation, exporters, context, propagation, resources, sampling, and APIs. It currently lists traces and metrics as stable and logs as development; browser client instrumentation is described as experimental/mostly unspecified. citeturn412332search1

Primary:

- https://opentelemetry.io/docs/languages/js/

---

## 2. OpenTelemetry Semantic Conventions

Semantic conventions standardize names and meanings across telemetry.

The current OpenTelemetry semantic-conventions documentation covers HTTP, databases, messaging, RPC, exceptions, system data, metrics, and other domains. citeturn412332search12

Primary:

- https://opentelemetry.io/docs/specs/semconv/

---

## 3. HTTP Semantic Conventions

Use current HTTP conventions for:

```text
HTTP spans
HTTP metrics
HTTP exceptions
```

OpenTelemetry notes that some HTTP convention sets are still evolving and provides migration/stability guidance; verify the emitted convention version before changing instrumentation contracts. citeturn412332search0turn412332search7

Primary:

- https://opentelemetry.io/docs/specs/semconv/http/

---

## 4. Database Semantic Conventions

Use for:

```text
database client spans
database metrics
database exceptions
```

OpenTelemetry documents stability/migration considerations for database conventions. citeturn412332search3

Primary:

- https://opentelemetry.io/docs/specs/semconv/db/

---

## 5. Messaging Semantic Conventions

Use for:

```text
queue operations
message processing
producer/consumer telemetry
```

Current messaging semantic conventions include technology-specific guidance while the general messaging conventions remain under development; verify the instrumentation/version used in your system. citeturn412332search2

Primary:

- https://opentelemetry.io/docs/specs/semconv/messaging/

---

## 6. Node.js Diagnostics

Use Node.js documentation for runtime-specific tools and APIs related to:

```text
diagnostics
performance
async context
memory
process behavior
```

Primary:

- https://nodejs.org/docs/latest/api/

---

## Source Discipline

Every telemetry decision should be classified:

```text
protocol/specification
runtime behavior
OpenTelemetry convention
instrumentation behavior
application event
operational policy
security policy
```

Never assume an instrumentation library's current attribute names are universal forever.

Always verify:

```text
library version
semantic convention version
runtime version
deployment topology
```

before treating telemetry fields as long-lived contracts.

---

# Chapter 83 — Completion Snapshot

## Core Theory

- [ ] Observability definition
- [ ] Monitoring vs observability
- [ ] Logs
- [ ] Metrics
- [ ] Traces
- [ ] Profiles
- [ ] Runtime diagnostics
- [ ] Structured events
- [ ] Stable telemetry naming
- [ ] Cardinality
- [ ] Histograms
- [ ] Sampling
- [ ] Correlation
- [ ] Context propagation
- [ ] HTTP telemetry
- [ ] Database telemetry
- [ ] Messaging telemetry
- [ ] Queue observability
- [ ] Health/readiness
- [ ] SLI
- [ ] SLO
- [ ] SLA
- [ ] Error budget
- [ ] Alerting
- [ ] Telemetry security
- [ ] Telemetry backpressure

## Implementation

- [ ] Structured logger
- [ ] Redaction layer
- [ ] Request ID
- [ ] Trace context
- [ ] HTTP metrics
- [ ] Duration histograms
- [ ] Error metrics
- [ ] Database spans
- [ ] Dependency spans
- [ ] Queue metrics
- [ ] Queue trace propagation
- [ ] Health/readiness
- [ ] Runtime diagnostics
- [ ] Sampling
- [ ] Telemetry failure handling
- [ ] Bounded buffers
- [ ] SLO dashboards
- [ ] Alerting rules
- [ ] Runbooks
- [ ] Observability tests

## Interview / Reasoning

- [ ] Explain logs/metrics/traces
- [ ] Explain cardinality
- [ ] Design tracing
- [ ] Design metric strategy
- [ ] Design alert strategy
- [ ] Design SLO
- [ ] Diagnose p99
- [ ] Diagnose queue growth
- [ ] Diagnose memory growth
- [ ] Design telemetry security
- [ ] Design organization-wide standards
- [ ] Defend observability architecture

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

Given an unfamiliar production JavaScript platform, can you determine:

```text
How do we know the system is healthy?
What user outcomes are measured?
What are the critical SLIs?
Which SLOs exist?
What alerts are actionable?
Can one request be followed end to end?
Can async work be correlated back to its source?
Which dependencies contribute most to latency?
What does p99 look like?
Where are the saturation signals?
Can we detect queue growth?
Can we diagnose memory leaks?
Can we investigate unknown failure modes?
Which telemetry has high cardinality?
What data is sensitive?
Can one tenant see another tenant's telemetry?
What happens if telemetry storage is down?
Can telemetry itself overload the application?
What deployment/version context exists?
Can an incident responder identify the likely cause quickly?
```

A principal observability engineer designs for the moment when something has gone wrong and **the system must explain itself through evidence**.

---

## Principal Observability Decision Framework

For every telemetry decision, record:

```text
Question:
User Impact:
Signal:
Event/Metric/Span Name:
Attributes:
Cardinality:
Retention:
Sampling:
Security Classification:
PII/Secret Risk:
Operational Cost:
Alert:
Dashboard:
Runbook:
Owner:
Failure Behavior:
Decision:
Revisit Trigger:
```

The objective is not to collect maximum telemetry.

It is to create the **smallest reliable evidence system that lets engineers detect important failures, understand unknown behavior, quantify user impact, and make the next correct operational decision.**