# Chapter 78 — Production JavaScript Architecture

> **Part XV — Production JavaScript**  
> **Status:** `[ ] Not Started`  
> **Depth Target:** Principal / Staff-level reasoning  
> **Primary Runtime Focus:** Production Node.js systems, with browser/application-boundary considerations where relevant  
> **Language Focus:** Modern JavaScript and ECMAScript semantics; TypeScript examples are intentionally minimized unless type-level structure clarifies an architectural point.

---

## Mission

This chapter is about designing JavaScript systems that remain understandable, testable, observable, secure, performant, operable, and evolvable after the codebase becomes large.

The goal is not to memorize “clean architecture” diagrams. The goal is to reason about architecture as a set of **boundaries, dependencies, responsibilities, state transitions, failure modes, and operational constraints**.

By the end of this chapter, you should be able to look at a production JavaScript system and explain:

- where the important boundaries are;
- who owns each piece of business behavior;
- which dependencies point inward or outward and why;
- where transactions and consistency boundaries exist;
- how requests become domain behavior and then side effects;
- how errors cross boundaries;
- how retries interact with idempotency;
- how queues and asynchronous work change architecture;
- where caching belongs and what consistency it introduces;
- how observability is designed into the system;
- how configuration, secrets, and trust boundaries are handled;
- how graceful startup and shutdown work;
- how to choose between a modular monolith, services, workers, serverless, or edge execution;
- how architecture should evolve without a dangerous rewrite.

The principal-level question throughout the chapter is:

> **What boundary should exist here, what invariant does it protect, and what operational cost does it introduce?**

---

# 1. Learning Objectives

After completing this chapter, you should be able to:

1. Define production architecture in terms of boundaries, dependencies, state, contracts, and operational behavior.
2. Distinguish code organization from architecture.
3. Design a modular monolith with strong dependency direction.
4. Separate presentation, application, domain, and infrastructure responsibilities.
5. Apply ports-and-adapters / hexagonal thinking without turning it into ceremonial abstraction.
6. Identify bounded contexts and avoid accidental coupling.
7. Prefer feature-oriented or vertical-slice organization when it reduces cross-cutting friction.
8. Design application services / use cases that coordinate work without becoming giant “god services.”
9. Model domain entities, value objects, aggregates, domain services, and business invariants appropriately.
10. Design repositories and gateways as boundaries rather than database-shaped business logic.
11. Distinguish DTOs, domain models, persistence models, and integration messages.
12. Design transaction boundaries deliberately.
13. Explain synchronous versus asynchronous communication.
14. Design retry-safe workflows using idempotency.
15. Explain outbox/inbox-style reliability patterns and their trade-offs.
16. Choose consistency models intentionally rather than accidentally.
17. Handle configuration and secrets as startup-time dependencies.
18. Design process lifecycle behavior for startup, readiness, graceful shutdown, and forced termination.
19. Reason about concurrency, backpressure, timeouts, retries, bulkheads, and circuit breakers.
20. Design observability using logs, metrics, traces, context propagation, and actionable signals.
21. Design authentication and authorization boundaries.
22. Evaluate architecture choices using cost, performance, memory, security, reliability, maintainability, scalability, and operational complexity.
23. Refactor architectural failures incrementally.
24. Defend architecture decisions in a principal-level interview or design review.
25. Build a production-grade modular Node.js service skeleton without relying on a framework-specific architecture template.

---

# 2. Prerequisites

You should already understand:

- JavaScript values, objects, functions, scope, closures, modules, promises, async/await, and errors.
- ECMAScript execution semantics at a foundational level.
- Node.js runtime architecture and core APIs.
- Node.js event loop and asynchronous I/O.
- Streams and backpressure.
- ESM and CommonJS interoperability.
- Package resolution and dependency management.
- Security fundamentals.
- HTTP/API fundamentals.
- Databases and transactions at a basic level.
- Testing and debugging fundamentals.

Relevant prior chapters:

- **12** — Execution Contexts / Execution Model
- **13** — Closures
- **15–21** — Objects and object semantics
- **29–30** — Errors and cleanup
- **31–40** — Async, concurrency, cancellation, streaming, reactive concepts
- **45–48** — Memory and engine behavior
- **55–57** — Networking and security
- **58–63** — Node.js architecture, core APIs, streams, workers, process lifecycle, diagnostics
- **64–70** — Modules, package resolution, dependencies, builds, source maps
- **74–77** — Functional programming, OOP, composition, design patterns

---

# 3. What Is It?

## 3.1 Production architecture

Production architecture is the deliberate organization of a software system around:

- responsibilities;
- boundaries;
- state;
- dependencies;
- interfaces;
- data flow;
- failure handling;
- operational controls;
- security controls;
- changeability.

A directory tree is not an architecture.

A diagram is not automatically an architecture.

A framework's recommended folders are not automatically an architecture.

A production architecture exists when the system has rules governing:

```text
who may depend on whom
who owns state
who validates what
where business rules live
where side effects happen
how failures propagate
how requests cross boundaries
how work is retried
how the system starts and stops
how the system is observed
how the system changes
```

---

## 3.2 Architecture versus implementation

Consider:

```js
await db.orders.insert(order);
```

This is implementation.

The architectural question is:

> Should the domain/application layer know that an `orders` table exists?

Often the answer is no.

A boundary might instead look like:

```js
await orderRepository.save(order);
```

Now the architecture says:

```text
Use case
  ↓
Repository port
  ↓
Postgres adapter
```

The point is not to hide SQL because SQL is “dirty.”

The point is to preserve the boundary where the business behavior depends on **the capability** “persist an order,” not on one concrete storage implementation.

---

## 3.3 Production architecture is constraint management

Architecture exists because systems have constraints.

Common constraints include:

- changing business rules;
- multiple teams;
- independent deployment;
- data consistency;
- latency targets;
- traffic spikes;
- limited memory;
- partial failures;
- security requirements;
- compliance requirements;
- operational ownership;
- external APIs;
- legacy dependencies;
- organizational boundaries.

A good architecture makes the important constraints visible.

A bad architecture hides them until production exposes them.

---

# 4. Why Does It Exist?

Small JavaScript systems can often survive with:

```text
route
  → query database
  → call API
  → return response
```

As a system grows, the same request may involve:

```text
authentication
authorization
validation
business rules
multiple repositories
transactions
cache
external APIs
events
audit
metrics
tracing
retries
timeouts
idempotency
```

Without boundaries, responsibilities collapse into a few large modules.

Typical consequences:

- every module imports infrastructure;
- every function knows database details;
- tests require the whole application;
- retries duplicate side effects;
- error handling becomes inconsistent;
- changing persistence becomes risky;
- unrelated features share mutable state;
- observability becomes fragmented;
- circular dependencies appear;
- deploys become difficult to reason about.

Architecture exists to **control change**.

---

# 5. Mental Model

Think of a production system as five interacting dimensions.

```text
                 ┌────────────────────┐
                 │     Business       │
                 │      Rules         │
                 └─────────┬──────────┘
                           │
                 ┌─────────▼──────────┐
                 │   Application      │
                 │     Use Cases      │
                 └─────────┬──────────┘
                           │
              ┌────────────┴────────────┐
              │                         │
      ┌───────▼────────┐       ┌────────▼───────┐
      │   Interfaces   │       │ Infrastructure │
      │ HTTP / Jobs    │       │ DB / APIs      │
      └────────────────┘       └────────────────┘

             Orthogonal concerns:
   security • observability • configuration
   lifecycle • reliability • performance
```

The central principle:

> **Business behavior should not become a hostage to transport, storage, framework, or infrastructure details.**

But the opposite failure is also real:

> **Do not introduce abstractions that merely rename implementation details without protecting a meaningful boundary.**

---

# 6. Core Rules

## Rule 1 — Architecture follows responsibility

Put code where its responsibility is easiest to understand and protect.

## Rule 2 — Dependencies should be intentional

A dependency is architectural information.

```text
A imports B
```

means:

> A is coupled to B.

A large architecture with undocumented coupling is fragile.

---

## Rule 3 — Business invariants deserve protected boundaries

Examples:

```text
An order cannot be shipped before payment is confirmed.
A refund cannot exceed the captured amount.
A user cannot transfer more than their available balance.
A tenant can only access its own resources.
```

These are stronger architectural boundaries than “controllers should be thin.”

---

## Rule 4 — Side effects should be visible

Database writes, network calls, queue publishing, file writes, email delivery, and cache mutation are side effects.

Do not hide important side effects behind innocent-looking domain calculations.

---

## Rule 5 — Validate at the right boundary

Validation is not one thing.

Different validation belongs at different levels:

```text
Transport validation
  HTTP body shape

Application validation
  request is valid for the use case

Domain validation
  invariant is valid

Infrastructure validation
  database/API constraints
```

---

## Rule 6 — Errors have boundaries too

An infrastructure exception should not automatically escape as a database-specific error to an HTTP client.

Think:

```text
PostgresUniqueViolation
        ↓
Infrastructure translation
        ↓
DuplicateOrderError
        ↓
Application error policy
        ↓
HTTP 409
```

---

## Rule 7 — Architecture must survive failure

Ask:

```text
What happens if the database is slow?
What happens if the queue retries?
What happens if the HTTP client times out?
What happens if the process dies after the DB commit?
What happens if the same message arrives twice?
What happens if shutdown begins during a request?
```

Architecture is incomplete until these questions have answers.

---

## Rule 8 — Optimize boundaries, not diagrams

A cleaner diagram that adds 12 network calls is often a worse system.

---

## Rule 9 — Prefer local reasoning

A developer should be able to understand a feature by reading a small number of nearby modules.

This strongly favors:

- feature folders;
- vertical slices;
- bounded contexts;
- explicit contracts.

---

## Rule 10 — Architecture should be reversible where possible

Prefer decisions that can be changed without rewriting the business core.

For example:

```text
Domain
  ↓
Repository port
  ↓
Postgres adapter
```

can later become:

```text
Domain
  ↓
Repository port
  ↓
Distributed storage adapter
```

without rewriting domain logic.

---

# 7. Syntax

Architecture has no JavaScript keyword.

It is expressed through:

- module boundaries;
- exports/imports;
- object interfaces;
- functions;
- classes;
- factories;
- dependency injection;
- composition roots;
- process boundaries;
- message schemas;
- HTTP contracts;
- database interfaces.

A minimal port:

```js
export function createOrderRepository({ db }) {
  return {
    async save(order) {
      await db.query(
        "INSERT INTO orders (id, total) VALUES ($1, $2)",
        [order.id, order.total],
      );
    },
  };
}
```

A use case depends on the capability:

```js
export function createPlaceOrder({ orderRepository }) {
  return async function placeOrder(input) {
    const order = createOrder(input);
    await orderRepository.save(order);
    return order;
  };
}
```

Composition happens at the edge:

```js
const orderRepository = createOrderRepository({ db });

const placeOrder = createPlaceOrder({
  orderRepository,
});
```

This boundary is the architecture.

---

# 8. Basic Examples

## Example 1 — Layered architecture

```text
HTTP Controller
      ↓
Application Service
      ↓
Domain
      ↓
Repository
      ↓
Database
```

Advantages:

- simple mental model;
- familiar to teams;
- clear responsibility boundaries.

Risks:

- layers can become passive pass-through code;
- features may require touching every layer;
- generic layers can hide business boundaries.

---

## Example 2 — Feature-oriented architecture

```text
src/
  orders/
    order.domain.js
    order.application.js
    order.http.js
    order.repository.js
    order.tests.js

  payments/
    payment.domain.js
    payment.application.js
    payment.http.js
    payment.repository.js

  users/
    user.domain.js
    user.application.js
    user.http.js
```

This makes related behavior local.

---

## Example 3 — Vertical slice

A single feature may cross technical layers:

```text
orders/
  create-order/
    command.js
    handler.js
    policy.js
    repository.js
    route.js
```

The feature is the organizing principle.

---

# 9. Execution Walkthrough

Consider:

```http
POST /orders
```

A robust production path can look like:

```text
HTTP request
  ↓
router
  ↓
authentication middleware
  ↓
authorization policy
  ↓
request validation
  ↓
application command
  ↓
use case
  ↓
domain behavior
  ↓
transaction boundary
  ↓
repository / gateway
  ↓
commit
  ↓
integration event
  ↓
response mapping
```

A trace should conceptually follow the same path:

```text
request span
  ├─ auth span
  ├─ database span
  ├─ external payment span
  └─ queue publish span
```

The architecture should make this flow visible.

---

# 10. Internal Mechanics

## 10.1 Dependency direction

Suppose:

```text
orders/application.js
    imports
orders/postgres-repository.js
```

Then the application layer is now coupled to Postgres.

A cleaner direction:

```text
orders/application.js
    ↓
repository contract
    ↑
postgres adapter
```

The application expresses what it needs.

Infrastructure fulfills it.

---

## 10.2 Composition root

A composition root is the place where concrete dependencies are assembled.

```js
const db = createDatabase(config.database);
const orderRepository = createOrderRepository({ db });
const paymentGateway = createPaymentGateway(config.payment);
const placeOrder = createPlaceOrder({
  orderRepository,
  paymentGateway,
});
```

Then handlers receive the ready use case:

```js
app.post("/orders", async (request, response) => {
  const order = await placeOrder(request.body);
  response.status(201).json(order);
});
```

This prevents dependency construction from spreading everywhere.

---

## 10.3 Dependency injection without a container

JavaScript makes dependency injection easy:

```js
export function createService({
  repository,
  clock,
  idGenerator,
}) {
  return {
    async execute(input) {
      const now = clock.now();
      const id = idGenerator();
      // ...
    },
  };
}
```

Do not install a dependency injection framework merely because dependency injection exists.

The architectural goal is:

```text
construction is centralized
usage is explicit
dependencies are visible
```

---

## 10.4 Domain versus infrastructure

Domain code:

```js
export function confirmPayment(payment) {
  if (payment.status !== "authorized") {
    throw new PaymentCannotBeConfirmedError();
  }

  return {
    ...payment,
    status: "confirmed",
  };
}
```

Infrastructure code:

```js
export async function loadPayment(db, paymentId) {
  return db.query("SELECT ...", [paymentId]);
}
```

The domain should not need to know how SQL connections are pooled.

---

# 11. ECMAScript / Specification Semantics

Architecture is largely an application-level concern, but it is built on standardized language behavior.

Important ECMAScript concepts that affect architecture include:

### Modules

ES modules provide static import/export structure:

```js
import { createOrder } from "./order.js";
export { createOrder };
```

This enables tooling and explicit dependency graphs.

### Closures

Factories and dependency injection often rely on closures:

```js
function createService({ repository }) {
  return {
    execute() {
      return repository.save();
    },
  };
}
```

The returned function retains access to `repository`.

### Objects and references

Dependencies are usually object references:

```js
const dependency = {
  execute() {},
};

const service = createService({
  dependency,
});
```

Mutability therefore matters architecturally.

### Promises

Application boundaries commonly expose promise-based operations:

```js
await repository.save(entity);
```

Promise semantics define asynchronous composition, but they do not define transactionality, retries, or reliability.

That distinction is essential:

> `Promise` is a language-level asynchronous abstraction.  
> A transaction is a system-level consistency mechanism.

---

# 12. Advanced Behavior

## 12.1 Layered Architecture

Typical:

```text
Presentation
Application
Domain
Infrastructure
```

### Strengths

- easy to teach;
- clear initial structure;
- predictable ownership.

### Weaknesses

- “everything goes through every layer” can create ceremony;
- technical layers can become more important than business capabilities;
- shared utility modules become accidental coupling hubs.

Use when:

- the system has a coherent technical topology;
- team members are familiar with it;
- feature count is moderate;
- strict layering improves safety.

---

## 12.2 Clean Architecture

A common conceptual structure:

```text
        Frameworks & Drivers
                ↓
        Interface Adapters
                ↓
        Application / Use Cases
                ↓
             Domain
```

The dependency rule says dependencies point toward the inner policy.

The useful idea is **dependency inversion**, not a fixed folder structure.

---

## 12.3 Hexagonal Architecture

Hexagonal architecture emphasizes:

```text
        inbound adapters
              ↓
        application core
              ↑
        outbound adapters
```

Examples:

```text
Inbound:
HTTP
CLI
queue message

Outbound:
database
email provider
payment provider
filesystem
```

The “hexagon” is a conceptual model, not a requirement to have six things.

---

## 12.4 Feature-oriented Architecture

Instead of:

```text
controllers/
services/
repositories/
validators/
utils/
```

consider:

```text
orders/
payments/
customers/
inventory/
```

Each feature owns its behavior.

Inside the feature, use the technical structure that actually helps.

This reduces horizontal browsing.

---

## 12.5 Vertical Slices

A vertical slice organizes code around a user-facing or business capability:

```text
create-order/
cancel-order/
refund-order/
capture-payment/
```

This works well when each capability has distinct policies.

It can become messy when slices duplicate infrastructure logic.

The solution is not “never share code.”

The solution is:

> Share stable concepts deliberately; do not share unstable code merely because it looks similar.

---

# 13. Edge Cases

## 13.1 “Repository” that is just a database wrapper

Bad:

```js
repository.findMany(options)
repository.queryRaw(sql)
repository.execute(sql)
```

This can become a generic database API disguised as a repository.

Better:

```js
orderRepository.findPendingShipmentOrders();
orderRepository.save(order);
```

The interface communicates business intent.

---

## 13.2 Shared “utils” module

A giant:

```text
utils/
```

often becomes a hidden dependency graph.

Warning signs:

```text
utils imports database
utils imports HTTP client
utils imports config
utils imports logger
```

Now nearly everything depends on everything.

---

## 13.3 Global mutable state

```js
export const state = {};
```

is architectural coupling.

Global state can make:

- tests order-dependent;
- concurrency reasoning harder;
- memory lifetime unclear;
- cleanup difficult.

---

## 13.4 Domain importing framework objects

Bad:

```js
export async function placeOrder(req, res, next) {
  // business logic
}
```

The business use case is now transport-shaped.

Prefer:

```js
const result = await placeOrder(command);
```

and map the result to HTTP outside.

---

## 13.5 Circular dependencies

Example:

```text
orders → payments
payments → orders
```

Circular dependencies often indicate:

- unclear ownership;
- shared domain incorrectly extracted;
- events should be used instead;
- a third concept is missing.

Do not automatically “fix” a cycle with lazy imports.

First find the ownership problem.

---

# 14. Common Misconceptions

### Misconception 1 — “Microservices are more scalable.”

They may provide independent scaling, but they also add:

- network calls;
- serialization;
- operational overhead;
- distributed failure;
- versioning;
- observability requirements;
- deployment complexity.

### Misconception 2 — “Clean Architecture means many folders.”

No.

It means protecting dependency direction and policy boundaries.

### Misconception 3 — “A service class is architecture.”

A class is an implementation mechanism.

Architecture is about system boundaries.

### Misconception 4 — “Repositories are mandatory.”

No.

Use a repository when it protects a useful boundary.

### Misconception 5 — “Async means scalable.”

Async I/O improves concurrency characteristics for many workloads, but CPU-bound work, connection limits, queue depth, downstream capacity, and memory still impose limits.

### Misconception 6 — “Retries improve reliability.”

Retries can amplify failure.

Retries need:

- bounded attempts;
- backoff;
- jitter where appropriate;
- timeouts;
- idempotency;
- classification of retryable errors.

---

# 15. Common Mistakes

## Mistake 1 — Giant application service

```js
class OrderService {
  // 300 methods
}
```

This is often several use cases pretending to be one abstraction.

---

## Mistake 2 — Business logic in controllers

Controllers should coordinate transport, not contain the entire business model.

---

## Mistake 3 — Database leakage

```js
const rows = await db.query(...);
if (rows[0].status === "x") ...
```

inside domain/application logic couples business rules to persistence shape.

---

## Mistake 4 — Infrastructure leakage

Domain code calling:

```js
fetch(...)
redis.get(...)
fs.readFile(...)
```

may be correct in a very small program, but becomes expensive when the domain boundary matters.

---

## Mistake 5 — Event-driven everything

Events increase decoupling in some dimensions and reduce immediate causal visibility in others.

---

## Mistake 6 — Premature microservices

A modular monolith often provides better local reasoning and cheaper coordination until service boundaries become justified.

---

# 16. Comparison With Related Concepts

| Approach | Primary Idea | Main Strength | Main Risk |
|---|---|---|---|
| Layered | Technical layers | Familiarity | Horizontal coupling |
| Clean | Dependency direction | Policy isolation | Over-ceremony |
| Hexagonal | Ports/adapters | Replaceable edges | Interface proliferation |
| Feature-oriented | Business features | Local reasoning | Duplication risk |
| Vertical slice | End-to-end capability | Fast feature ownership | Inconsistent conventions |
| Modular monolith | Strong in-process modules | Low operational overhead | Requires discipline |
| Microservices | Independent deployable services | Independent ownership/scaling | Distributed complexity |
| Serverless | Managed execution units | Elastic operational model | Runtime/vendor constraints |
| Event-driven | Messages/events | Loose time coupling | Harder debugging/consistency |

---

# 17. Performance Considerations

Architecture affects performance through:

- number of network hops;
- serialization;
- database calls;
- cache lookups;
- object allocation;
- promise chains;
- queue latency;
- logging overhead;
- tracing overhead;
- connection pooling;
- concurrency limits.

## 17.1 N+1 access pattern

Bad:

```js
const users = await usersRepository.findAll();

for (const user of users) {
  user.orders = await orderRepository.findByUserId(user.id);
}
```

This creates approximately:

```text
1 + N database calls
```

Prefer batching:

```js
const users = await usersRepository.findAll();
const orders = await orderRepository.findByUserIds(users.map(u => u.id));
```

Architecture can make the efficient path obvious.

---

## 17.2 Network boundaries are expensive

An in-process function:

```js
await paymentPolicy.check(input);
```

is not equivalent to:

```js
await paymentService.check(input);
```

across a network.

The latter introduces:

- latency;
- timeout;
- serialization;
- connection failure;
- partial failure;
- versioning.

Therefore:

> A process boundary is a performance boundary and a reliability boundary.

---

## 17.3 Concurrency control

Do not turn:

```js
items.map(processItem)
```

into unbounded concurrency.

Use bounded concurrency where the downstream system has capacity limits.

---

# 18. Memory Considerations

Architecture affects object lifetime.

A request-scoped object should usually die after the request.

A global cache can live for the lifetime of the process.

A queue of pending work may grow indefinitely if production exceeds consumption.

This creates a useful question:

> **Who owns this memory, and what event releases it?**

Watch for:

```text
unbounded arrays
global caches
listener accumulation
request objects retained by callbacks
large response buffering
unbounded queues
```

Architecture must specify memory lifecycle.

---

# 19. Security Considerations

Security architecture starts with trust boundaries.

Example:

```text
Internet
  ↓
API Gateway
  ↓
Application
  ↓
Database
```

Each boundary should answer:

```text
Who is trusted?
What is authenticated?
What is authorized?
What data crosses the boundary?
What is validated?
What is logged?
What secrets are exposed?
```

## 19.1 Authentication versus authorization

Authentication:

> Who are you?

Authorization:

> Are you allowed to do this?

Do not bury both in generic middleware.

The authorization decision often belongs close to the business action.

---

## 19.2 Multi-tenant boundaries

For multi-tenant applications, tenant identity should become a first-class architectural concern.

A request might establish:

```js
const context = {
  tenantId,
  actorId,
  requestId,
};
```

Then repositories or application policies must enforce tenant isolation.

Never rely solely on the UI to prevent cross-tenant access.

---

## 19.3 Secrets

Never treat secrets as source-code constants.

Configuration should distinguish:

```text
non-secret configuration
secret credentials
runtime-derived values
```

Secrets should be available only to components that need them.

---

# 20. Production Usage

## 20.1 Reference modular Node.js architecture

```text
src/
  app/
    bootstrap.js
    composition-root.js
    config.js
    lifecycle.js

  modules/
    orders/
      domain/
        order.js
        errors.js
        policies.js

      application/
        place-order.js
        cancel-order.js

      infrastructure/
        postgres-order-repository.js

      interfaces/
        http.js
        jobs.js

    payments/
      domain/
      application/
      infrastructure/
      interfaces/

  shared/
    observability/
    errors/
    ids/
    time/

  infrastructure/
    database/
    queue/
    http/
```

The exact folders are less important than ownership and dependency direction.

---

## 20.2 Dependency graph

A healthy modular graph might be:

```text
interfaces
    ↓
application
    ↓
domain

infrastructure → application/domain contracts
```

A suspicious graph:

```text
everything → shared/utils
everything → database
domain → framework
domain → HTTP
controller → SQL
```

---

## 20.3 Bounded contexts

Suppose a commerce platform has:

```text
Orders
Payments
Inventory
Shipping
Customers
```

Do not create one giant domain model:

```js
everything.customer.payment.inventory.order.shipping = ...
```

Each context should own its language and invariants.

Integration occurs through:

- explicit APIs;
- integration events;
- translated models;
- anti-corruption layers.

---

# 21. Implementation From Scratch

Build a small production-style modular service using only Node.js primitives.

## Stage 1 — Guided

### Step 1: Domain model

```js
export class Order {
  constructor({ id, customerId, total, status = "pending" }) {
    if (!id) throw new Error("Order id is required");
    if (!customerId) throw new Error("Customer id is required");
    if (total <= 0) throw new Error("Order total must be positive");

    this.id = id;
    this.customerId = customerId;
    this.total = total;
    this.status = status;
  }

  confirm() {
    if (this.status !== "pending") {
      throw new Error("Order cannot be confirmed");
    }

    this.status = "confirmed";
  }
}
```

---

## Step 2: Repository port

Define the capability the application needs:

```js
export function createOrderRepository() {
  throw new Error("Provide an implementation");
}
```

Conceptually:

```text
save(order)
findById(id)
```

---

## Step 3: In-memory adapter

```js
export function createInMemoryOrderRepository() {
  const orders = new Map();

  return {
    async save(order) {
      orders.set(order.id, order);
    },

    async findById(id) {
      return orders.get(id) ?? null;
    },
  };
}
```

---

## Step 4: Application use case

```js
export function createPlaceOrder({ repository, idGenerator }) {
  return async function placeOrder({ customerId, total }) {
    const order = new Order({
      id: idGenerator(),
      customerId,
      total,
    });

    await repository.save(order);

    return order;
  };
}
```

---

## Step 5: Composition root

```js
const repository = createInMemoryOrderRepository();

const placeOrder = createPlaceOrder({
  repository,
  idGenerator: crypto.randomUUID,
});
```

---

## Stage 2 — Partially Guided

Replace in-memory persistence with a real database adapter.

Requirements:

```text
application code must not import database driver
domain code must not import database driver
adapter owns SQL
transaction belongs to the application workflow
errors are translated at the boundary
```

---

## Stage 3 — No Reference

Build:

```text
POST /orders
GET /orders/:id
POST /orders/:id/confirm
```

Constraints:

- no business logic in routes;
- explicit dependency injection;
- domain invariants enforced centrally;
- repository isolated;
- errors mapped to HTTP responses.

---

## Stage 4 — Edge-case hardened

Add:

```text
duplicate order protection
timeouts
structured errors
transaction rollback
idempotency key
request correlation ID
bounded external-call concurrency
graceful shutdown
readiness check
health endpoint
```

---

## Stage 5 — Production-grade

Add:

```text
Postgres adapter
outbox record
queue publisher
retry policy
metrics
tracing
structured logging
audit events
configuration validation
feature flag
deployment-safe migrations
integration tests
contract tests
load test
```

---

# 22. Debugging Exercises

## Exercise 1 — Circular dependency

```text
orders imports payments
payments imports orders
```

Determine:

1. who owns the invariant;
2. whether an event is appropriate;
3. whether a shared abstraction is missing;
4. whether the modules should actually remain coupled.

---

## Exercise 2 — Giant service

A `UserService` contains:

```text
registration
password reset
billing
notifications
analytics
admin authorization
```

Refactor by use case and ownership.

---

## Exercise 3 — Infrastructure leak

Find all framework imports from domain code.

Explain what boundary each import violates.

---

## Exercise 4 — Duplicate side effects

A payment request times out after the provider succeeds.

The client retries.

The provider receives the same operation twice.

Design an idempotency strategy.

---

## Exercise 5 — Shutdown bug

A process receives `SIGTERM` but:

```text
accepts new requests
starts new background jobs
closes database too early
```

Design the correct shutdown order.

---

## Exercise 6 — N+1 query

Given:

```js
for (const order of orders) {
  order.customer = await customers.findById(order.customerId);
}
```

Determine:

- query count;
- latency behavior;
- database pressure;
- batching strategy.

---

## Exercise 7 — Memory leak

A module registers an event listener on every request.

The listener captures the request object.

Find the lifecycle error.

---

## Exercise 8 — Backpressure failure

A producer emits 50,000 jobs per second.

A consumer can process 2,000 per second.

Explain:

```text
queue growth
memory impact
latency growth
failure behavior
load shedding
```

---

# 23. Code Review Exercise

Review this:

```js
export async function createOrder(req, res) {
  const user = await db.users.findById(req.user.id);

  if (!user) {
    return res.status(404).send();
  }

  const existing = await db.orders.findOne({
    customerId: user.id,
    status: "pending",
  });

  if (existing) {
    return res.status(409).json(existing);
  }

  const payment = await fetch("https://payments.example.com/charge", {
    method: "POST",
    body: JSON.stringify({
      amount: req.body.total,
      userId: user.id,
    }),
  });

  const order = await db.orders.insert({
    userId: user.id,
    total: req.body.total,
  });

  await sendEmail(user.email, "Order created");

  return res.status(201).json(order);
}
```

Identify at least 15 architectural problems.

Potential findings include:

- transport contains business logic;
- database abstraction leakage;
- weak input validation;
- missing transaction strategy;
- external call ordering;
- payment idempotency risk;
- no timeout;
- no retry classification;
- no authorization decision;
- email is synchronous;
- no event/outbox strategy;
- error translation absent;
- no observability;
- possible race condition;
- response contract coupled to persistence shape.

---

# 24. Interview Questions

## Fundamental

1. What does “production architecture” mean?
2. Architecture versus code organization?
3. Why does dependency direction matter?
4. What is a bounded context?
5. What is a modular monolith?
6. When would you choose a modular monolith over microservices?
7. What is dependency inversion?
8. What is a port?
9. What is an adapter?
10. Why use a composition root?

## Intermediate

11. Where should business validation live?
12. Where should authorization live?
13. Where should transaction boundaries live?
14. When should you use repositories?
15. What is an anti-corruption layer?
16. What is an integration event?
17. What is idempotency?
18. Why can retries make an outage worse?
19. What is the difference between synchronous and asynchronous coupling?
20. How would you structure a large Node.js codebase?

## Advanced

21. How do you split a monolith into services without rewriting the domain?
22. How do you define service boundaries?
23. How do you maintain consistency across services?
24. How does an outbox pattern address dual-write problems?
25. How do you design for graceful shutdown?
26. How do you prevent unbounded concurrency?
27. How do you design observability boundaries?
28. How do you avoid a shared library becoming a distributed monolith?
29. How do you handle versioned events?
30. What architectural fitness functions would you introduce?

## Principal

31. What architectural decision would you intentionally make temporary?
32. Which dependency would you tolerate to keep delivery velocity high?
33. How do you know that a service boundary is real?
34. What evidence would make you reverse an architecture decision?
35. How do organizational boundaries influence technical boundaries?
36. How do you balance local reasoning with global consistency?
37. When does abstraction reduce resilience rather than improve it?
38. What is the cost of adding one network hop?
39. How do you migrate a critical path without a flag day?
40. Describe a production architecture you would reject even though it is theoretically “clean.”

---

# 25. Predict-the-Output Exercises

## Exercise A — Dependency closure

Predict:

```js
function createCounter() {
  let value = 0;

  return {
    increment() {
      value += 1;
      return value;
    },
  };
}

const a = createCounter();
const b = createCounter();

console.log(a.increment());
console.log(a.increment());
console.log(b.increment());
```

### Prediction

Write your answer before reading further.

### Actual Result

```text
1
2
1
```

### Trace

Each call to `createCounter()` creates a distinct lexical environment containing its own `value`.

### Rule

Architectural factories often use closures to create isolated dependency or state scopes.

---

## Exercise B — Shared dependency

Predict:

```js
const repository = {
  value: 0,
  save() {
    this.value += 1;
  },
};

function createService({ repository }) {
  return {
    execute() {
      repository.save();
      return repository.value;
    },
  };
}

const a = createService({ repository });
const b = createService({ repository });

console.log(a.execute());
console.log(b.execute());
```

### Actual Result

```text
1
2
```

### Principal Lesson

Dependency injection does not create isolation automatically. Object identity still matters.

---

## Exercise C — Promise does not imply transaction

Predict:

```js
async function workflow() {
  await saveOrder();
  await callPaymentProvider();
  await sendEmail();
}
```

Ask:

> Which operations are atomic?

Answer:

> None become atomic merely because they are awaited.

`await` sequences asynchronous operations. It does not create a distributed transaction.

---

# 26. Mastery Exercises

## Track A — Core Theory

### A1
Explain layered, clean, hexagonal, feature-oriented, and vertical-slice architecture without using diagrams.

### A2
Given a module graph, identify:

- cycles;
- unstable dependencies;
- infrastructure leakage;
- inappropriate shared abstractions.

### A3
Explain why a process boundary is both a latency boundary and a reliability boundary.

---

## Track B — Implementation

### B1 — Modular monolith

Build a Node.js application with:

```text
orders
payments
inventory
```

Each module must own:

- domain;
- application use cases;
- infrastructure;
- interfaces.

### B2 — Reliability

Add:

```text
idempotency
timeouts
retry policy
outbox-style event publishing
graceful shutdown
```

### B3 — Observability

Add:

```text
structured logs
request IDs
trace context
metrics
health/readiness
```

OpenTelemetry's current JavaScript documentation describes Node.js and browser support, API/SDK usage, instrumentation, context, propagation, and exporters; treat it as a concrete implementation option rather than an architectural requirement. citeturn320860search0turn320860search1

---

## Track C — Interview / Reasoning

### C1

A company has 35 engineers, one Node.js codebase, slow deploys, unclear ownership, and frequent regression.

Decide whether to:

```text
keep monolith
modularize monolith
split services
```

Defend your decision using evidence.

### C2

A payment provider is unreliable.

Design:

```text
timeout
retry
idempotency
circuit breaking
fallback
audit
reconciliation
```

### C3

A feature requires:

```text
database write
queue publish
external API call
email
```

Design the sequence and failure policy.

---

# 27. Key Takeaways

1. Architecture is about boundaries, not folders.
2. Dependencies are part of the architecture.
3. Business invariants should have explicit ownership.
4. Domain policy should be insulated from replaceable infrastructure when that boundary provides real value.
5. Modular monoliths are often a powerful default.
6. Microservices add distributed-systems costs.
7. A network boundary changes performance, reliability, consistency, and observability.
8. Application services coordinate use cases; they should not become giant containers for everything.
9. Repositories should express meaningful capabilities, not hide generic SQL.
10. Transactions are consistency mechanisms, not JavaScript mechanisms.
11. Retries require idempotency and failure classification.
12. Async processing requires queue capacity and backpressure thinking.
13. Configuration and secrets are architectural dependencies.
14. Graceful shutdown is part of correctness.
15. Observability belongs in architecture, not as an afterthought.
16. Security boundaries should be explicit.
17. The best architecture preserves local reasoning.
18. Good architecture makes change cheaper, not merely diagrams prettier.
19. Principal engineers choose trade-offs and know when to remove abstractions.
20. Architecture must be continuously validated against production evidence.

---

# 28. Concept Connections

## Depends On

- **Chapter 12** — Execution model
- **Chapter 29** — Errors
- **Chapter 31–40** — Async and concurrency
- **Chapter 55** — Fetch / HTTP
- **Chapter 58–63** — Node.js runtime and lifecycle
- **Chapter 64–70** — Modules and tooling
- **Chapter 74–77** — Paradigms, composition, design patterns

## Builds Toward

- **Chapter 79** — API Design
- **Chapter 80** — Library Authoring
- **Chapter 81** — Database Integration
- **Chapter 82** — API Architecture
- **Chapter 83** — Observability
- **Chapter 84** — Reliability
- **Chapter 85** — Performance
- **Chapter 89** — Code Review / Refactoring
- **Chapter 101** — Real-world Production Scenarios
- **Chapter 110** — Production JavaScript Backend
- **Chapter 111** — Large-scale JavaScript Platform
- **Chapter 121** — Principal System Design

## Related Concepts

```text
architecture
  ├─ modularity
  ├─ dependency inversion
  ├─ domain modeling
  ├─ transactions
  ├─ messaging
  ├─ observability
  ├─ security
  ├─ reliability
  ├─ performance
  └─ operations
```

## Concepts Revisited

This chapter revisits:

- closures;
- modules;
- promises;
- errors;
- streams;
- dependency injection;
- proxies/adapters;
- Node process lifecycle;
- package boundaries;
- observability.

## Why This Chapter Matters Later

Later production chapters treat individual concerns:

```text
API
database
observability
reliability
performance
testing
```

This chapter provides the architectural frame that connects them.

---

# 29. Completion Criteria

You may mark:

```text
[~] In Progress
```

when you can explain the major architectural styles.

Mark:

```text
[?] Needs Revision
```

when you repeatedly confuse:

- architecture with folder structure;
- abstraction with indirection;
- async with reliability;
- service boundaries with scalability.

Mark:

```text
[+] Completed
```

when you can:

- design a modular monolith;
- explain dependency direction;
- define transaction boundaries;
- design error translation;
- reason about retries and idempotency;
- design graceful shutdown;
- build a complete vertical slice.

Mark:

```text
[*] Mastered
```

only when you can:

- critique a real production architecture;
- identify hidden coupling;
- explain operational consequences;
- defend trade-offs;
- propose an incremental migration;
- implement the architecture without a template;
- diagnose an architectural failure from production symptoms.

Reading alone does not qualify as mastery.

---

# Chapter 78 — Revision / Retrieval Record

Use this section after studying the chapter.

| Date | Retrieval Goal | Result | Status |
|---|---|---|---|
| ____ | Explain architecture vs code organization | ____ | `[ ]` |
| ____ | Compare modular monolith vs microservices | ____ | `[ ]` |
| ____ | Explain dependency inversion | ____ | `[ ]` |
| ____ | Design a production request flow | ____ | `[ ]` |
| ____ | Design transaction + retry + idempotency boundaries | ____ | `[ ]` |
| ____ | Debug architectural coupling | ____ | `[ ]` |
| ____ | Defend a principal-level architecture decision | ____ | `[ ]` |

### Spaced Retrieval

```text
Review 1 — same day
Review 2 — +1 day
Review 3 — +3 days
Review 4 — +7 days
Review 5 — +14 days
Review 6 — +30 days
Review 7 — +60 days
```

### Retrieval Prompts

Without reading:

1. Define production architecture in one sentence.
2. Explain the difference between a module boundary and a process boundary.
3. Explain why microservices increase operational complexity.
4. Draw a modular monolith from memory.
5. Explain where transaction ownership belongs.
6. Explain idempotency using a payment example.
7. Explain the purpose of a composition root.
8. Explain how shutdown should proceed.
9. Name five architectural failure modes.
10. Defend modular monolith versus microservices for a hypothetical startup.

---

# Chapter 78 — Canonical References and Source Discipline

This chapter deliberately separates **language semantics** from **runtime, infrastructure, and architecture practice**.

## Source hierarchy

### 1. ECMAScript specification

Use for:

- modules;
- functions;
- objects;
- promises;
- language semantics.

Primary:

- ECMAScript Specification — https://tc39.es/ecma262/

### 2. Node.js documentation

Use for:

- process lifecycle;
- runtime APIs;
- diagnostics;
- workers;
- networking;
- streams.

Primary:

- Node.js Documentation — https://nodejs.org/docs/latest/api/

### 3. OpenTelemetry

OpenTelemetry's JavaScript documentation currently covers Node.js and browser instrumentation, traces, metrics, context, propagation, exporters, and instrumentation libraries. Its current documentation also notes that JavaScript tracing and metrics are stable while logs remain under development. Verify implementation details against the version deployed by your organization. citeturn320860search0

Primary:

- OpenTelemetry JavaScript — https://opentelemetry.io/docs/languages/js/
- OpenTelemetry JavaScript instrumentation libraries — https://opentelemetry.io/docs/languages/js/libraries/

### 4. OWASP ASVS

Use for:

- secure architecture requirements;
- authentication/authorization concerns;
- input validation;
- secure application controls.

The OWASP Application Security Verification Standard provides a requirements-based basis for testing and assessing technical security controls, and the current stable release listed by OWASP is ASVS 5.0.0. citeturn320860search6

Primary:

- OWASP ASVS — https://owasp.org/www-project-application-security-verification-standard/

### 5. Twelve-Factor principles

Use as operational heuristics, not as universal architecture law.

The Twelve-Factor guidance emphasizes disposable processes, fast startup, and graceful shutdown; it describes graceful shutdown for web processes as stopping new requests, allowing current work to finish, and then exiting. citeturn320860search4

Primary:

- https://12factor.net/

### Architecture literature

Useful conceptual references:

- Eric Evans — *Domain-Driven Design*
- Robert C. Martin — *Clean Architecture*
- Alistair Cockburn — Hexagonal Architecture
- Martin Fowler — architecture and integration essays
- Michael Nygard — *Release It!*
- Sam Newman — *Building Microservices*
- Vaughn Vernon — *Implementing Domain-Driven Design*

These sources express design ideas rather than ECMAScript requirements.

---

# Chapter 78 — Completion Snapshot

## Core Theory

- [ ] Architecture versus code organization
- [ ] Dependency direction
- [ ] Layered architecture
- [ ] Clean architecture
- [ ] Hexagonal architecture
- [ ] Feature-oriented architecture
- [ ] Vertical slices
- [ ] Modular monolith
- [ ] Microservices trade-offs
- [ ] Bounded contexts
- [ ] Domain/application/infrastructure boundaries
- [ ] DTOs and integration contracts
- [ ] Transaction boundaries
- [ ] Idempotency
- [ ] Outbox/inbox concepts
- [ ] Consistency models
- [ ] Configuration and secrets
- [ ] Lifecycle and shutdown
- [ ] Resilience boundaries
- [ ] Observability architecture
- [ ] Security architecture

## Implementation

- [ ] Build a modular Node.js service
- [ ] Build a composition root
- [ ] Inject dependencies explicitly
- [ ] Implement domain invariants
- [ ] Implement repository adapter
- [ ] Implement application use case
- [ ] Add HTTP adapter
- [ ] Add transaction handling
- [ ] Add idempotency
- [ ] Add retry policy
- [ ] Add outbox-style event publication
- [ ] Add health/readiness
- [ ] Add graceful shutdown
- [ ] Add structured logs
- [ ] Add metrics
- [ ] Add traces
- [ ] Add integration tests

## Interview / Reasoning

- [ ] Explain modular monolith choice
- [ ] Defend service boundaries
- [ ] Identify hidden coupling
- [ ] Analyze distributed failure
- [ ] Explain idempotency
- [ ] Analyze N+1 behavior
- [ ] Analyze backpressure
- [ ] Design a migration
- [ ] Critique an architecture
- [ ] Defend a trade-off

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

Given an unfamiliar JavaScript production system, can you answer all of these without immediately looking at the code implementation?

```text
Where are the boundaries?
Who owns the invariants?
What depends on what?
Where does state live?
Where are transactions?
Where can requests duplicate?
Where can work be lost?
Where can work be repeated?
Where can queues grow?
Where can latency explode?
Where can memory grow?
Where are trust boundaries?
How is failure observed?
How does the process stop?
How does the system evolve?
What would you change first, and why?
```

If you cannot answer these, continue retrieval and implementation.

---

## Principal Decision Framework

For every significant architecture decision, evaluate:

```text
Correctness
Performance
Memory
Security
Reliability
Maintainability
Scalability
Observability
Developer Experience
Operational Complexity
Future Change
```

Then state explicitly:

```text
Decision:
Context:
Alternatives:
Why this choice:
Trade-offs:
Failure modes:
Operational impact:
Migration path:
Exit criteria:
```

The goal of architecture is not to produce the “cleanest” code.

The goal is to produce the **most appropriate system for the constraints**, with the consequences of the decision understood and observable.