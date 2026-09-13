# Chapter 82 — API Architecture

> **Part XV — Production JavaScript**  
> **Status:** `[ ] Not Started`  
> **Depth Target:** Principal / Staff-level reasoning  
> **Primary Focus:** Designing the architecture behind APIs: service boundaries, contracts, workflows, integration patterns, consistency, reliability, security, observability, and long-term evolution.

---

# 1. Learning Objectives

By the end of this chapter, you should be able to:

1. Distinguish API design from API architecture.
2. Explain where API responsibilities should live in a production JavaScript system.
3. Design API boundaries around business capabilities.
4. Design service interfaces between modules and processes.
5. Choose between modular monolith APIs, service APIs, RPC, REST-style HTTP, GraphQL, streaming, and events.
6. Define ownership of API contracts.
7. Separate transport, application, domain, and infrastructure concerns.
8. Design request and response boundaries.
9. Design synchronous and asynchronous workflows.
10. Model long-running operations.
11. Design idempotency and deduplication.
12. Design retries, timeouts, backoff, and failure budgets.
13. Design rate limits, quotas, and admission control.
14. Design API gateways and edge responsibilities.
15. Design backend-for-frontend and aggregation patterns.
16. Design service-to-service communication.
17. Handle compatibility between independently deployed consumers and providers.
18. Design contract versioning and deprecation.
19. Design API schemas and contract testing.
20. Design authorization across service boundaries.
21. Design tenant isolation across APIs.
22. Design pagination, filtering, sorting, and bulk operations architecturally.
23. Design long-running workflows and job-based APIs.
24. Design webhook contracts and event delivery.
25. Understand consistency boundaries across multiple APIs.
26. Design distributed workflows without assuming distributed transactions are free.
27. Integrate APIs with databases, queues, caches, and external providers.
28. Design observability across request boundaries.
29. Diagnose performance bottlenecks caused by API topology.
30. Identify distributed-system failure modes hidden behind apparently simple API calls.
31. Design a production Node.js API architecture.
32. Review an existing API platform for correctness, security, performance, reliability, and evolvability.
33. Defend architectural choices at principal-engineer level.

---

# 2. Prerequisites

You should understand:

- JavaScript functions, modules, promises, async/await.
- Node.js HTTP and process behavior.
- Streams and backpressure.
- API design and HTTP semantics.
- Database integration and transactions.
- Production JavaScript architecture.
- Security fundamentals.
- Observability fundamentals.

Recommended prior chapters:

- **29** — Errors / Error Handling
- **31–40** — Async / Concurrency / Streaming
- **55** — Fetch / HTTP Networking
- **56–57** — Browser Security / JavaScript Security Engineering
- **58–63** — Node.js Architecture / Core APIs / Process Lifecycle / Diagnostics
- **64–70** — Modules / Packages / Build Systems
- **78** — Production JavaScript Architecture
- **79** — API Design
- **80** — Library Authoring
- **81** — Database Integration

---

# 3. What Is It?

API design asks:

```text
What does this endpoint mean?
What request does it accept?
What response does it return?
```

API architecture asks:

```text
Who owns this capability?
Where is the boundary?
Which process implements it?
Which data does it own?
Which other services does it call?
What consistency is guaranteed?
What happens if dependencies fail?
How does the API evolve?
How is it observed and secured?
```

Therefore:

```text
API design
    ↓
contract shape

API architecture
    ↓
system structure behind and around that contract
```

---

## 3.1 Example

A client calls:

```http
POST /orders
```

API design defines:

```text
request schema
status codes
response schema
errors
```

API architecture defines:

```text
order ownership
customer lookup
inventory reservation
payment workflow
transaction boundary
event publication
idempotency
authorization
observability
retry behavior
deployment boundary
```

---

# 4. Why Does It Exist?

An API becomes architectural when multiple components depend on it.

For example:

```text
Web App
Mobile App
Partner App
Internal Services
Background Jobs
```

may all depend on:

```text
Orders API
```

Now a change to:

```text
field
status
error
timing
authentication
rate limit
```

can affect many systems.

API architecture exists to control these dependencies.

---

# 5. Mental Model

Think of an API platform as:

```text
                 Consumers
      ┌────────────┼────────────┐
      │            │            │
     Web         Mobile       Partners
      │            │            │
      └────────────▼────────────┘
                 API Edge
                    │
          ┌─────────▼─────────┐
          │ Application/API   │
          │ boundary          │
          └─────────┬─────────┘
                    │
          ┌─────────▼─────────┐
          │ Business modules  │
          └──────┬─────┬──────┘
                 │     │
             database  events
                 │     │
          ┌──────▼─────▼──────┐
          │ External systems  │
          └───────────────────┘
```

With cross-cutting concerns:

```text
authentication
authorization
rate limits
timeouts
observability
security
compatibility
```

---

# 6. Core Rules

## Rule 1 — A service should own a capability

Avoid:

```text
CustomerDatabaseService
OrderTableService
PaymentTableService
```

Prefer capabilities such as:

```text
Orders
Payments
Inventory
Shipping
```

when those correspond to meaningful ownership boundaries.

---

## Rule 2 — Every distributed boundary has a cost

Adding a service call introduces:

```text
latency
serialization
failure
timeouts
versioning
observability
deployment coordination
```

Treat process/network boundaries as expensive decisions.

---

## Rule 3 — Keep causal paths short

A request:

```text
API
→ service A
→ service B
→ service C
→ service D
→ database
```

has a larger failure surface than:

```text
API
→ service A
→ database
```

Do not add hops merely to make diagrams symmetrical.

---

## Rule 4 — Contract ownership must be explicit

A contract needs an owner:

```text
who defines it
who tests it
who versions it
who deprecates it
who supports it
```

---

## Rule 5 — Architecture must model partial failure

In distributed systems:

```text
service can be down
service can be slow
response can be lost
request can be duplicated
message can arrive late
message can arrive twice
```

Design for these states.

---

## Rule 6 — Synchronous and asynchronous coupling are different

Synchronous:

```text
A waits for B
```

Asynchronous:

```text
A records intent
B processes later
```

The second can improve temporal decoupling but introduces delivery and consistency complexity.

---

## Rule 7 — Do not pretend multiple systems are one transaction

A database transaction does not automatically include:

```text
payment provider
email provider
queue
another database
```

Design explicit workflows.

---

## Rule 8 — Authentication is not authorization

Authentication establishes identity.

Authorization decides access.

API architecture must preserve both boundaries.

---

## Rule 9 — API gateways should stay focused

An edge layer can handle:

```text
TLS termination
routing
authentication integration
rate limits
request shaping
observability
```

Do not let it become the place where all business logic accumulates.

---

## Rule 10 — APIs should be designed for evolution

A successful API eventually has:

```text
old clients
new clients
mixed versions
partial migrations
```

Compatibility is normal operating reality.

---

# 7. Syntax

API architecture is expressed through:

```text
modules
services
interfaces
HTTP routes
RPC handlers
message consumers
schemas
events
queues
dependency injection
composition roots
```

A service interface can be represented in JavaScript:

```js
export function createOrderService({
  inventory,
  payment,
  repository,
}) {
  return {
    async place(command) {
      // orchestrate business workflow
    },
  };
}
```

A process-facing client:

```js
export function createPaymentsClient({
  baseUrl,
  fetchImpl,
}) {
  return {
    async authorize(request) {
      // HTTP call
    },
  };
}
```

The architectural distinction is:

```text
in-process dependency
vs
network dependency
```

---

# 8. Basic Examples

## Example 1 — Modular monolith

```text
HTTP
 ↓
Orders module
 ↓
Payments module
 ↓
same process
```

No network hop exists between the modules.

---

## Example 2 — Service architecture

```text
HTTP
 ↓
Orders Service
 ↓
Payments Service
```

Now the payment dependency has:

```text
network latency
timeouts
retry semantics
versioning
service discovery
```

---

## Example 3 — Asynchronous workflow

```text
POST /orders
      ↓
create order
      ↓
publish OrderPlaced
      ↓
HTTP 202
      ↓
worker processes payment
```

The client must understand that:

```text
request accepted
≠
final business state complete
```

---

# 9. Execution Walkthrough

Consider:

```http
POST /orders
Idempotency-Key: abc
```

Possible architecture:

```text
edge
  ↓
authentication
  ↓
rate limit
  ↓
request validation
  ↓
idempotency lookup
  ↓
order use case
  ↓
transaction
  ├─ create order
  ├─ reserve inventory
  └─ create outbox event
  ↓
commit
  ↓
return 201
```

Then:

```text
outbox publisher
  ↓
OrderCreated event
  ↓
payment worker
  ↓
payment provider
  ↓
payment result
  ↓
Order state update
```

Notice that the original HTTP request did not necessarily wait for every external system.

---

# 10. Internal Mechanics

## 10.1 In-process boundary

```js
await paymentService.authorize(input);
```

Cost:

```text
function call
object creation
Promise scheduling if async
```

---

## 10.2 Network boundary

```js
await paymentClient.authorize(input);
```

Cost:

```text
serialization
socket acquisition
network
remote queueing
remote compute
response parsing
timeout possibility
failure possibility
```

The two lines can look similar and have radically different system behavior.

---

## 10.3 API gateway

Conceptually:

```text
client
  ↓
gateway
  ├─ authentication
  ├─ rate limit
  ├─ request ID
  ├─ routing
  └─ observability
        ↓
     service
```

The gateway should generally avoid:

```text
business workflow orchestration
domain-specific transactions
long-lived business state
```

unless it is explicitly designed as an application composition layer.

---

## 10.4 Backend-for-Frontend

BFF:

```text
Mobile BFF
Web BFF
     ↓
domain services
```

This can reduce consumer-specific complexity inside core services.

Trade-off:

```text
additional service/application layer
```

Use when consumer needs genuinely differ.

---

# 11. ECMAScript / Specification Semantics

API architecture is not defined by ECMAScript.

Keep the layers distinct:

```text
ECMAScript
  → language behavior

Node.js
  → runtime and host APIs

HTTP
  → transport protocol

API schema
  → application contract

Service architecture
  → system design
```

For example:

```js
await client.get("/orders");
```

JavaScript semantics define Promise/async behavior.

They do not define:

```text
whether GET is idempotent
whether an order exists
whether the remote system committed
whether retry is safe
whether response is authoritative
```

Those are protocol/application semantics.

---

# 12. Advanced Behavior

## 12.1 Service boundaries

A good service boundary often aligns with:

```text
business capability
data ownership
team ownership
change rate
failure isolation
scaling characteristics
security boundary
```

No single dimension is sufficient.

---

## 12.2 Bounded context

Suppose:

```text
Orders
Payments
Accounting
```

Each may interpret:

```text
customer
status
amount
transaction
```

differently.

Do not force a shared canonical object when contexts have legitimately different models.

---

## 12.3 Anti-corruption layer

An adapter can translate:

```text
Payment Provider Model
```

into:

```text
Application Payment Model
```

Example:

```js
function mapProviderPayment(providerPayment) {
  return {
    providerReference: providerPayment.id,
    status: normalizeStatus(providerPayment.state),
  };
}
```

This protects the internal model from provider vocabulary.

---

## 12.4 Aggregator API

Suppose a client needs:

```text
customer
orders
recommendations
```

Option:

```text
client → 3 services
```

or:

```text
client → aggregator → 3 services
```

The aggregator can reduce client-side network chatter.

But it introduces:

```text
another hop
orchestration logic
failure aggregation
latency coupling
```

Use based on actual consumer/system needs.

---

## 12.5 Fan-out

An API calls:

```text
A
B
C
D
```

in parallel.

Potential latency:

```text
max(A, B, C, D)
```

rather than:

```text
A + B + C + D
```

but the failure surface expands.

If one dependency fails:

```text
whole request?
partial response?
fallback?
stale data?
```

must be designed.

---

# 13. Edge Cases

## 13.1 Timeout after remote success

Sequence:

```text
client
  ↓
service
  ↓
remote operation succeeds
  ↓
response lost
  ↓
service times out
```

Retry can duplicate the operation.

Idempotency must be part of the API architecture.

---

## 13.2 Partial response

Aggregator:

```text
customer ✅
orders ✅
recommendations ❌
```

Possible policies:

```text
fail whole request
return partial data
return stale cache
use fallback
```

Choose explicitly.

---

## 13.3 Event arrives after state changed

```text
OrderCancelled
OrderCreated
```

may arrive out of order in some systems.

Consumers should not assume event arrival order unless the transport/architecture guarantees it.

---

## 13.4 Duplicate event

```text
OrderCreated
OrderCreated
```

Consumer should often be idempotent:

```js
if (await alreadyProcessed(event.id)) {
  return;
}
```

The exact deduplication strategy depends on storage and correctness requirements.

---

## 13.5 Schema evolution

Producer adds:

```json
{
  "currency": "INR"
}
```

Older consumers may ignore it safely.

But changing:

```json
"status": "pending"
```

to:

```json
"status": "PENDING"
```

can break strict consumers.

---

# 14. Common Misconceptions

### “Every bounded context should be a microservice.”

No. Bounded contexts can exist inside a modular monolith.

### “A service should have one database table.”

No. Service data ownership may involve many tables.

### “More services mean better separation.”

More services increase distributed coordination.

### “An API gateway solves architecture.”

A gateway cannot fix poor domain ownership.

### “Async makes the system reliable.”

Async changes failure and timing semantics; it does not remove failure.

### “Retries solve intermittent errors.”

Retries can amplify overload.

### “Events are always more decoupled.”

Events reduce some temporal coupling but increase:

```text
debugging
eventual consistency
schema evolution
delivery semantics
```

### “A 202 means the request succeeded.”

Usually it means accepted for processing, not that the eventual business operation is complete.

---

# 15. Common Mistakes

## Mistake 1 — Distributed monolith

Signs:

```text
service A cannot deploy without B
service B cannot deploy without C
A calls B calls C for every request
shared database across all services
```

This is distributed deployment without meaningful independence.

---

## Mistake 2 — Shared database as integration API

If every service freely reads every table:

```text
orders
payments
inventory
all query same database
```

service boundaries become weak.

---

## Mistake 3 — Business logic in gateway

Gateway:

```text
if country === "IN"
  call service A
else
  call service B
```

might be legitimate in some edge composition designs, but if business policy continually accumulates there, ownership is probably misplaced.

---

## Mistake 4 — Unbounded fan-out

One API triggers:

```text
20 downstream requests
```

Per-request dependency count becomes a reliability and latency problem.

---

## Mistake 5 — No correlation

Distributed calls without a shared:

```text
request ID
trace context
causal identifiers
```

make debugging extremely difficult.

---

# 16. Comparison With Related Concepts

| Architecture | Strength | Main Risk |
|---|---|---|
| Modular monolith | Low operational cost | Requires module discipline |
| Microservices | Independent deployment/ownership | Distributed complexity |
| BFF | Consumer-specific composition | Additional layer |
| API gateway | Central edge controls | Business-logic creep |
| REST-style service API | Familiar HTTP semantics | Contract discipline required |
| RPC | Explicit operations | Can underuse HTTP semantics |
| GraphQL | Flexible consumer queries | Query complexity and governance |
| Event-driven | Temporal decoupling | Eventual consistency |
| Queue-based worker | Resilient async processing | Operational queue complexity |
| Aggregator | Simplifies consumers | Fan-out/failure aggregation |
| Shared database | Easy initial integration | Weak ownership and coupling |

---

# 17. Performance Considerations

## 17.1 Latency budget

Suppose:

```text
total target = 300 ms
```

and service A calls:

```text
B = 100 ms
C = 100 ms
D = 100 ms
```

Sequential:

```text
≈ 300 ms
```

before additional overhead.

Parallel:

```text
≈ max(B, C, D)
```

but only if dependencies are independent and resources permit concurrency.

---

## 17.2 Tail latency

Average latency can look healthy while p95/p99 becomes unacceptable.

Example:

```text
p50 = 50 ms
p95 = 150 ms
p99 = 900 ms
```

A fan-out API can amplify tail latency because the aggregate waits for slow dependencies.

---

## 17.3 Payload cost

Large service-to-service payloads increase:

```text
serialization
network bandwidth
memory
GC
parse time
```

Prefer contracts that transfer necessary information rather than entire internal object graphs.

---

## 17.4 Retry amplification

Suppose:

```text
100 requests/sec
```

each retries once during a failure.

Traffic can become:

```text
200 requests/sec
```

and potentially trigger further overload.

Add:

```text
bounded retries
backoff
jitter
timeouts
circuit/bulkhead behavior
```

where appropriate.

---

# 18. Memory Considerations

API architecture can create memory growth through:

```text
request buffering
aggregation
fan-out result storage
queues
caches
in-flight promises
retry queues
connection pools
```

A BFF that calls:

```text
20 services
```

and retains every response before constructing the final payload can consume significant memory.

Prefer bounded concurrency and streaming where the architecture supports it.

---

# 19. Security Considerations

## 19.1 Trust boundaries

```text
Internet
  ↓
edge
  ↓
API
  ↓
internal service
  ↓
database
```

Do not assume an internal service is automatically authorized to do everything.

---

## 19.2 Service identity

Service-to-service requests should have identity and authorization appropriate to the environment.

Consider:

```text
who is calling
which service
for which tenant
which action
with which credentials
```

---

## 19.3 Tenant context

Never trust:

```http
X-Tenant-ID: customer-a
```

as authoritative identity just because a header exists.

Tenant identity should be tied to authenticated context and authorization.

---

## 19.4 Sensitive propagation

Avoid forwarding:

```text
access tokens
private headers
internal credentials
```

to downstream services unless required.

Use explicit propagation rules.

---

# 20. Production Usage

## 20.1 Reference API platform

```text
                         Consumers
                  ┌────────┼─────────┐
                  │        │         │
                 Web     Mobile    Partner
                  │        │         │
                  └────────▼─────────┘
                         Edge
                  ┌───────┴────────┐
                  │ Auth / Limits  │
                  │ Routing / Obs. │
                  └───────┬────────┘
                          │
                    API boundary
                          │
       ┌──────────────────┼──────────────────┐
       │                  │                  │
    Orders             Payments           Catalog
       │                  │                  │
       └──────────┬───────┴──────────┬───────┘
                  │                  │
                DBs              Event Bus
                                     │
                                  Workers
```

---

## 20.2 Contract ownership

For every API, define:

```text
Provider Owner
Consumer Owners
Schema Owner
SLO Owner
Security Owner
Deprecation Owner
```

A technical contract without organizational ownership eventually decays.

---

## 20.3 API inventory

Maintain:

```text
API name
owner
purpose
consumers
authentication
version
SLO
dependencies
deprecation status
```

Improper API inventory and lifecycle management are recognized API security concerns; OWASP's API Security Top 10 includes Improper Inventory Management. citeturn277088search4

---

## 20.4 Synchronous service call policy

Every remote call should have an explicit:

```text
timeout
retry policy
maximum concurrency
error mapping
fallback policy
observability
```

Do not inherit accidental defaults forever.

---

## 20.5 Asynchronous API

For long-running operations:

```http
POST /reports
```

Response:

```http
202 Accepted
Location: /reports/rpt_123
```

Client can later:

```http
GET /reports/rpt_123
```

Possible state:

```text
queued
running
completed
failed
cancelled
```

This separates request acceptance from work completion.

---

## 20.6 Webhooks

Webhook architecture:

```text
event occurs
  ↓
outbox
  ↓
delivery queue
  ↓
HTTP POST
  ↓
consumer
```

Production considerations:

```text
signature
timestamp
idempotency
retry
backoff
delivery timeout
dead-letter handling
replay
event version
```

---

## 20.7 API cancellation

Long operations should have cancellation semantics where useful.

Example:

```http
DELETE /jobs/job_123
```

could mean:

```text
request cancellation
```

not necessarily immediate erasure of the job.

The contract must distinguish:

```text
cancel requested
cancelled
already completed
cannot cancel
```

---

# 21. Implementation From Scratch

Build a production-style API platform.

## Stage 1 — Guided

Create:

```text
api/
  router.js
  middleware.js
  errors.js
  context.js

orders/
  application.js
  domain.js
  repository.js

clients/
  payments.js
```

---

## Stage 2 — Request context

Create:

```js
function createRequestContext({
  requestId,
  actorId,
  tenantId,
}) {
  return Object.freeze({
    requestId,
    actorId,
    tenantId,
  });
}
```

Use it explicitly across application calls.

---

## Stage 3 — Remote client

```js
export function createPaymentsClient({
  baseUrl,
  fetchImpl,
  timeoutMs,
}) {
  return {
    async authorize(input, { signal } = {}) {
      const controller = new AbortController();

      const timeout = setTimeout(
        () => controller.abort(),
        timeoutMs,
      );

      try {
        const response = await fetchImpl(
          `${baseUrl}/authorizations`,
          {
            method: "POST",
            signal: signal
              ? AbortSignal.any([signal, controller.signal])
              : controller.signal,
            headers: {
              "content-type": "application/json",
            },
            body: JSON.stringify(input),
          },
        );

        return response;
      } finally {
        clearTimeout(timeout);
      }
    },
  };
}
```

In a real implementation, account for runtime support and the semantics of the chosen cancellation APIs.

---

## Stage 4 — Retry policy

Create a policy:

```js
function shouldRetry(error, attempt) {
  if (attempt >= 3) return false;

  return error.code === "ETIMEDOUT"
    || error.code === "ECONNRESET";
}
```

Then add:

```text
backoff
jitter
idempotency
metrics
```

Do not retry arbitrary failures.

---

## Stage 5 — Aggregator

Build:

```http
GET /dashboard
```

that combines:

```text
customer
orders
notifications
```

Requirements:

```text
bounded concurrency
partial failure policy
timeout
fallback
request context
```

---

## Stage 6 — Async job API

Implement:

```http
POST /reports
GET /reports/:id
DELETE /reports/:id
```

Requirements:

```text
202 acceptance
job state
cancellation request
idempotency
worker processing
failure state
```

---

## Stage 7 — Production-grade

Add:

```text
OpenAPI
contract tests
consumer tests
authentication
authorization
rate limits
quota
request limits
timeouts
retries
circuit/bulkhead policy
tracing
metrics
structured logs
webhooks
versioning
deprecation
API inventory
```

---

# 22. Debugging Exercises

## Exercise 1 — Distributed timeout chain

```text
Gateway timeout = 1s
Service A timeout = 2s
Service B timeout = 5s
```

Explain why this is dangerous.

---

## Exercise 2 — Retry multiplication

```text
Client retries 3x
Gateway retries 2x
Service A retries 2x
```

Estimate the potential request amplification and identify where retry ownership should be simplified.

---

## Exercise 3 — Shared database

Three services directly modify:

```text
orders table
```

Find why deployment independence is mostly fictional.

---

## Exercise 4 — Out-of-order event

```text
PaymentCaptured
PaymentAuthorized
```

arrive in this order.

Determine whether the consumer can process them safely.

---

## Exercise 5 — Gateway business logic

Gateway now contains:

```text
pricing
discounts
tax policy
customer segmentation
```

Find the architecture smell.

---

## Exercise 6 — BFF fan-out

A dashboard API calls 15 services per request.

Find:

```text
tail latency risk
failure aggregation
memory cost
deployment coupling
```

---

## Exercise 7 — Contract drift

Provider removes:

```json
currency
```

while an old consumer still requires it.

Design prevention and migration.

---

## Exercise 8 — Duplicate webhook

Provider sends the same webhook four times.

Design:

```text
signature validation
event ID
dedupe
processing transaction
retry response
```

---

# 23. Code Review Exercise

Review:

```js
app.post("/orders", async (req, res) => {
  const user = await fetch(
    `http://users/users/${req.body.userId}`,
  );

  const inventory = await fetch(
    `http://inventory/stock/${req.body.productId}`,
  );

  const payment = await fetch(
    "http://payments/charge",
    {
      method: "POST",
      body: JSON.stringify(req.body),
    },
  );

  const order = await db.query(
    "INSERT INTO orders (...) VALUES (...)",
  );

  res.status(201).json(order);
});
```

Find at least 25 architectural problems.

Consider:

```text
transport/application coupling
service authentication
authorization
tenant context
timeouts
retry ownership
parallelism
idempotency
transaction boundary
shared database boundary
payment side effects
failure order
partial success
error mapping
response contract
observability
request correlation
schema validation
data ownership
service discovery
rate limits
circuit/bulkhead policy
sensitive data propagation
contract evolution
```

---

# 24. Interview Questions

## Fundamental

1. API design versus API architecture?
2. What makes an API an architectural boundary?
3. What is service ownership?
4. What is a bounded context?
5. What is a modular monolith?
6. What is a distributed monolith?
7. What is a BFF?
8. What is an API gateway?
9. What is an aggregator?
10. Why are network boundaries expensive?

## Intermediate

11. How do you decide service boundaries?
12. When should communication be synchronous?
13. When should communication be asynchronous?
14. What is idempotency?
15. How do retries interact across layers?
16. What is a timeout budget?
17. How do you design webhook delivery?
18. How do you design long-running API operations?
19. How do you version service contracts?
20. How do you use contract tests?

## Advanced

21. How would you migrate a modular monolith into services?
22. How do you avoid distributed transactions?
23. How do you preserve authorization across service boundaries?
24. How do you enforce tenant isolation?
25. How do you handle out-of-order events?
26. How do you handle duplicate events?
27. How do you design partial responses?
28. How do you control fan-out?
29. How do you diagnose p99 latency in a service graph?
30. How do you stop retry storms?

## Principal

31. What makes a service boundary real?
32. When should two bounded contexts share a process?
33. How do you identify a distributed monolith?
34. Which business logic belongs in the gateway?
35. How do you decide where orchestration belongs?
36. How do you choose between synchronous calls and events?
37. How do you design an API platform for 100+ teams?
38. How do you evolve a contract with unknown consumers?
39. How do organizational boundaries influence API architecture?
40. What architectural evidence would make you consolidate services?

---

# 25. Predict-the-Output Exercises

## Exercise A — Sequential async calls

Predict conceptually:

```js
async function run(a, b) {
  await a();
  await b();
}
```

If `a()` takes 100 ms and `b()` takes 100 ms, the sequential dependency is approximately:

```text
200 ms
```

excluding overhead.

---

## Exercise B — Parallel independent calls

```js
await Promise.all([
  a(),
  b(),
]);
```

If they are independent and both take approximately 100 ms:

```text
≈ 100 ms
```

excluding overhead and resource contention.

### Principal Lesson

Concurrency can improve latency, but only when the operations are independent and downstream capacity supports it.

---

## Exercise C — Promise.all failure

Predict:

```js
await Promise.all([
  succeeds(),
  fails(),
  slow(),
]);
```

### Rule

The returned Promise rejects when the required Promise.all settlement condition is reached due to a rejection, but other work may already have started and may continue.

A remote side effect is not automatically cancelled because `Promise.all()` rejected.

---

## Exercise D — Remote success versus local timeout

Conceptual trace:

```text
remote request starts
remote operation succeeds
local timer expires
client observes timeout
```

Question:

> Did the remote operation necessarily fail?

### Answer

No.

Timeout means the local caller stopped waiting according to its policy. The remote side may already have completed.

---

# 26. Mastery Exercises

## Track A — Core Theory

### A1

Take:

```text
Orders
Payments
Inventory
Shipping
Notifications
```

Determine:

```text
bounded contexts
data ownership
service candidates
synchronous dependencies
asynchronous dependencies
```

Defend every choice.

### A2

Draw a service dependency graph and calculate:

```text
depth
fan-out
fan-in
critical path
single points of failure
```

---

## Track B — Implementation

### B1 — Modular API platform

Build:

```text
Orders
Payments
Inventory
```

inside one Node.js process first.

Then extract one boundary into a separate process.

Compare:

```text
latency
failure
deployment
testing
observability
```

### B2 — API contract

Create:

```text
OpenAPI
JSON schemas
contract tests
consumer fixtures
```

Validate provider changes.

### B3 — Reliable asynchronous workflow

Implement:

```text
POST /orders
→ transaction
→ outbox
→ worker
→ payment provider
→ status update
```

Add:

```text
idempotency
retry
dead-letter handling
observability
```

---

## Track C — Interview / Reasoning

### C1

An organization has:

```text
15 services
12 databases
200 endpoints
```

but every request traverses 4–8 services.

Determine whether service decomposition is helping.

### C2

A dashboard endpoint has:

```text
p50 = 120 ms
p95 = 800 ms
p99 = 4 s
```

It fans out to 12 services.

Design an investigation.

### C3

An API has 10,000 consumers.

The team wants to rename:

```text
customerId → userId
```

Design a compatibility migration.

---

# 27. Key Takeaways

1. API design defines contract semantics; API architecture defines system ownership and behavior around that contract.
2. Every network boundary adds latency and failure modes.
3. Service boundaries should align with meaningful ownership.
4. Bounded contexts do not require microservices.
5. Modular monoliths can provide strong boundaries with lower operational cost.
6. Distributed monoliths are distributed systems without meaningful independence.
7. Synchronous calls create immediate temporal coupling.
8. Asynchronous communication increases temporal decoupling but introduces eventual consistency and delivery concerns.
9. Idempotency is essential for many retryable operations.
10. Retry policies should be coordinated rather than independently multiplied at every layer.
11. API gateways should remain focused on edge concerns unless explicitly designed otherwise.
12. BFFs can reduce consumer-specific complexity but add another layer.
13. Aggregators trade client simplicity for backend fan-out.
14. Long-running operations should distinguish acceptance from completion.
15. Webhooks require authentication, idempotency, retries, and replay considerations.
16. API contracts require ownership and lifecycle management.
17. Observability context must cross service boundaries.
18. Authorization must survive distributed boundaries.
19. Tenant isolation must be explicit.
20. The best API architecture minimizes unnecessary distributed coordination while making necessary coordination explicit.

---

# 28. Concept Connections

## Depends On

- **Chapter 29** — Errors
- **Chapter 31–40** — Async / Concurrency / Streaming / Cancellation
- **Chapter 55** — Fetch / HTTP Networking
- **Chapter 56–57** — Security
- **Chapter 58–63** — Node.js runtime and diagnostics
- **Chapter 64–70** — Modules / packages / tooling
- **Chapter 78** — Production JavaScript Architecture
- **Chapter 79** — API Design
- **Chapter 80** — Library Authoring
- **Chapter 81** — Database Integration

## Builds Toward

- **Chapter 83** — Observability
- **Chapter 84** — Reliability
- **Chapter 85** — Performance
- **Chapter 86** — Testing
- **Chapter 87** — Deterministic Async Testing
- **Chapter 89** — Code Review / Refactoring
- **Chapter 94** — Compatibility Engineering
- **Chapter 97** — Edge / Serverless JavaScript
- **Chapter 101** — Real-world Production Scenarios
- **Chapter 105** — Node REST API
- **Chapter 107** — Job Queue
- **Chapter 109** — Event-driven Application
- **Chapter 110** — Production JavaScript Backend
- **Chapter 111** — Large-scale JavaScript Platform
- **Chapter 121** — Principal System Design

## Related Concepts

```text
API Architecture
  ├─ service boundaries
  ├─ bounded contexts
  ├─ gateways
  ├─ BFFs
  ├─ synchronous calls
  ├─ asynchronous messaging
  ├─ workflow orchestration
  ├─ contracts
  ├─ consistency
  ├─ reliability
  ├─ security
  └─ observability
```

## Concepts Revisited

```text
HTTP semantics
Promises
AbortSignal
streams
errors
transactions
idempotency
dependency injection
database ownership
authentication
authorization
observability
```

## Why This Chapter Matters Later

This chapter is the bridge between:

```text
endpoint implementation
```

and:

```text
system architecture
```

The later production chapters deepen individual properties:

```text
observability
reliability
performance
testing
```

while the project chapters force you to integrate them.

---

# 29. Completion Criteria

Mark:

```text
[~] In Progress
```

when you understand:

```text
API boundary
service boundary
sync vs async
contract ownership
basic failure modes
```

Mark:

```text
[?] Needs Revision
```

when you repeatedly confuse:

```text
API design vs API architecture
module boundary vs process boundary
authentication vs authorization
retry vs idempotency
202 acceptance vs completed work
bounded context vs microservice
gateway vs business service
async vs reliability
```

Mark:

```text
[+] Completed
```

when you can:

- design an API platform;
- define service/module ownership;
- choose sync/async boundaries;
- design idempotency;
- design long-running operations;
- define contract lifecycle;
- design webhook delivery;
- design API observability;
- implement a service boundary.

Mark:

```text
[*] Mastered
```

only when you can:

- critique a real service graph;
- identify distributed-monolith behavior;
- design incremental decomposition;
- diagnose tail-latency amplification;
- design compatibility migration;
- reason about partial failure;
- defend API architecture at principal level.

Reading alone does not qualify as mastery.

---

# Chapter 82 — Revision / Retrieval Record

| Date | Retrieval Goal | Result | Status |
|---|---|---|---|
| ____ | Explain API design vs API architecture | ____ | `[ ]` |
| ____ | Explain service boundaries | ____ | `[ ]` |
| ____ | Explain bounded contexts | ____ | `[ ]` |
| ____ | Compare modular monolith and microservices | ____ | `[ ]` |
| ____ | Design sync/async workflow | ____ | `[ ]` |
| ____ | Design idempotency/retry | ____ | `[ ]` |
| ____ | Design API gateway/BFF strategy | ____ | `[ ]` |
| ____ | Design API contract evolution | ____ | `[ ]` |
| ____ | Analyze fan-out/tail latency | ____ | `[ ]` |
| ____ | Defend principal-level architecture | ____ | `[ ]` |

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

1. API design versus API architecture?
2. What makes a service boundary real?
3. What is a distributed monolith?
4. Why does a network boundary increase cost?
5. When should a workflow be asynchronous?
6. How do retries amplify failure?
7. What is idempotency?
8. What belongs in an API gateway?
9. When is a BFF useful?
10. How do you evolve a large API contract?
11. How do you preserve authorization across services?
12. How do you analyze API fan-out?

---

# Chapter 82 — Canonical References and Source Discipline

## 1. HTTP Semantics

Use the HTTP specifications for:

```text
methods
status codes
semantics
caching
conditional requests
```

Primary:

- https://www.rfc-editor.org/rfc/rfc9110.html

Do not invent transport semantics that conflict with HTTP.

---

## 2. Problem Details

RFC 9457 defines structured Problem Details for HTTP APIs and the `application/problem+json` representation. citeturn277088search5

- https://www.rfc-editor.org/rfc/rfc9457.html

---

## 3. OpenAPI

The OpenAPI Initiative's current published specification is 3.2.0, released September 19, 2025. citeturn560872search3

- https://spec.openapis.org/oas/latest.html

Use OpenAPI for:

```text
contract documentation
schema definition
tooling
contract review
testing
```

---

## 4. OWASP API Security

OWASP's API Security Top 10 includes risks such as:

```text
Broken Object Level Authorization
Broken Authentication
Broken Object Property Level Authorization
Unrestricted Resource Consumption
Broken Function Level Authorization
Unrestricted Access to Sensitive Business Flows
SSRF
Security Misconfiguration
Improper Inventory Management
Unsafe Consumption of APIs
```

Use these as architectural threat-model prompts. citeturn277088search4

- https://owasp.org/API-Security/

---

## 5. Node.js

Use Node.js documentation for:

```text
HTTP servers
timeouts
AbortSignal
streams
process lifecycle
runtime-specific networking
```

Primary:

- https://nodejs.org/docs/latest/api/

Node's HTTP API exposes request/server timeout and stream-related behavior; version-specific behavior must be verified against the Node versions actually supported by the system. citeturn560872search6

---

## 6. OpenTelemetry

OpenTelemetry JavaScript documentation covers Node.js/browser instrumentation, tracing, metrics, context, propagation, and exporters. Use the deployed OpenTelemetry versions as the implementation authority. citeturn320860search0turn320860search1

- https://opentelemetry.io/docs/languages/js/

---

## Source Discipline

Every architecture decision must be classified:

```text
HTTP protocol requirement
ECMAScript behavior
Node runtime behavior
Framework behavior
API contract choice
Business policy
Operational policy
Security requirement
Organizational convention
```

Never say:

> “The framework requires this”

without checking the framework version.

Never say:

> “Microservices require this”

when it is actually your organization's convention.

Never say:

> “Async guarantees reliability.”

Async changes timing and coupling; reliability requires explicit design.

---

# Chapter 82 — Completion Snapshot

## Core Theory

- [ ] API design vs API architecture
- [ ] Service ownership
- [ ] Bounded contexts
- [ ] Modular monolith
- [ ] Microservices
- [ ] Distributed monolith
- [ ] API gateway
- [ ] BFF
- [ ] Aggregator
- [ ] Sync communication
- [ ] Async communication
- [ ] Workflow orchestration
- [ ] Idempotency
- [ ] Retry policy
- [ ] Timeout budgets
- [ ] Rate limits
- [ ] Quotas
- [ ] Contract ownership
- [ ] Versioning
- [ ] Deprecation
- [ ] Contract testing
- [ ] Tenant isolation
- [ ] Service authorization
- [ ] Webhooks
- [ ] Long-running APIs
- [ ] Event-driven workflows
- [ ] Consistency
- [ ] Observability
- [ ] Tail latency
- [ ] Distributed failure

## Implementation

- [ ] Build modular API
- [ ] Create service boundary
- [ ] Create remote client
- [ ] Add request context
- [ ] Add timeout
- [ ] Add retry policy
- [ ] Add idempotency
- [ ] Add gateway concerns
- [ ] Add BFF/aggregator
- [ ] Build async job API
- [ ] Build webhook delivery
- [ ] Add contract tests
- [ ] Add OpenAPI
- [ ] Add authentication
- [ ] Add authorization
- [ ] Add tenant context
- [ ] Add rate limits
- [ ] Add traces
- [ ] Add metrics
- [ ] Add structured logs
- [ ] Add API inventory
- [ ] Add deprecation workflow

## Interview / Reasoning

- [ ] Define service boundaries
- [ ] Identify distributed monolith
- [ ] Explain sync vs async
- [ ] Explain retry amplification
- [ ] Design idempotency
- [ ] Design API gateway
- [ ] Design BFF
- [ ] Design webhook system
- [ ] Design compatibility migration
- [ ] Diagnose fan-out latency
- [ ] Defend service decomposition
- [ ] Defend modular monolith

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

Given an unfamiliar API platform, can you determine:

```text
Who owns each capability?
What are the boundaries?
Which boundaries are in-process?
Which are network boundaries?
Which data does each service own?
Where are transactions?
Where are asynchronous workflows?
Where are retries?
Where is idempotency?
Where can duplicate work occur?
Where can requests fan out?
Where can tail latency explode?
Where are trust boundaries?
Where is tenant authorization enforced?
What does the gateway own?
What does the gateway incorrectly own?
What does each API contract guarantee?
Who owns the contract?
How are consumers discovered?
How are breaking changes handled?
What happens when a dependency is slow?
What happens when a dependency is unavailable?
What happens when a response is lost after success?
What happens when an event is duplicated?
What happens when events arrive out of order?
How is partial failure represented?
How is long-running work exposed?
How is API health measured?
How is the architecture evolving?
```

A principal API architect does not optimize for the largest number of services.

They optimize for **clear ownership, controlled coupling, explicit failure semantics, secure contracts, observable behavior, and the smallest distributed topology that satisfies the real constraints.**

---

## Principal API Architecture Decision Framework

For each boundary, record:

```text
Business Capability:
Consumer:
Owner:
Data Ownership:
Contract:
Process Boundary:
Communication Style:
Consistency Requirement:
Latency Budget:
Timeout:
Retry Policy:
Idempotency:
Failure Modes:
Authorization:
Tenant Isolation:
Observability:
Security:
Scalability:
Operational Complexity:
Migration Cost:
Alternative:
Decision:
Revisit Trigger:
```

Use this framework to answer the principal question:

> **Is this boundary buying enough independence, safety, or clarity to justify the coordination cost it introduces?**