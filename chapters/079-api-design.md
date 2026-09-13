# Chapter 79 — API Design

> **Part XV — Production JavaScript**  
> **Status:** `[ ] Not Started`  
> **Depth Target:** Principal / Staff-level reasoning  
> **Primary Focus:** Designing stable, secure, observable, evolvable APIs for JavaScript/Node.js systems

---

# 1. Learning Objectives

By the end of this chapter, you should be able to:

1. Explain what an API contract is and why it is more important than an endpoint list.
2. Model resources and operations intentionally.
3. Design HTTP APIs around semantics rather than database tables.
4. Choose HTTP methods and status codes correctly.
5. Design path, query, header, and body parameters.
6. Distinguish representation from domain state.
7. Design request and response schemas.
8. Separate external DTOs from internal domain and persistence models.
9. Design validation at the correct boundary.
10. Design consistent error responses.
11. Use problem-details style errors appropriately.
12. Design idempotent operations and idempotency keys.
13. Design pagination, filtering, sorting, and searching.
14. Handle partial updates and concurrency safely.
15. Design caching and conditional requests.
16. Design API versioning and compatibility policies.
17. Design authentication and authorization boundaries.
18. Prevent object-level and property-level authorization failures.
19. Protect APIs from resource exhaustion and abusive business flows.
20. Design webhook and third-party integration contracts.
21. Design synchronous versus asynchronous API workflows.
22. Use OpenAPI as a contract/documentation/testing artifact.
23. Design API observability and correlation.
24. Design rate limits, quotas, timeouts, and request size limits.
25. Build a production-style API using Node.js primitives.
26. Review API designs for correctness, performance, memory, security, reliability, maintainability, and evolution cost.
27. Defend API decisions at principal-engineer level.

---

# 2. Prerequisites

You should understand:

- JavaScript functions and objects.
- Promises and async/await.
- Errors and exception handling.
- Node.js HTTP/runtime behavior.
- Streams and backpressure.
- Modules and dependency boundaries.
- Production architecture.
- Basic HTTP semantics.
- Basic database modeling and transactions.
- Authentication/authorization fundamentals.

Recommended prior chapters:

- **29** — Errors / Error Handling
- **31–40** — Async / Concurrency
- **55** — Fetch / HTTP Networking
- **56–57** — Browser Security / JavaScript Security Engineering
- **58–63** — Node.js Architecture / APIs / Lifecycle / Diagnostics
- **64–70** — Modules and Tooling
- **78** — Production JavaScript Architecture

---

# 3. What Is It?

An API is a contract between independently developed pieces of software.

For an HTTP API, the contract may include:

```text
method
URI
headers
authentication requirements
request schema
response schema
status codes
error semantics
idempotency rules
pagination behavior
caching behavior
rate limits
timeouts
compatibility guarantees
```

Therefore:

```text
API ≠ collection of URLs
```

A production API is a **behavioral contract**.

---

## 3.1 API as a boundary

Consider:

```http
POST /orders
```

The important questions are not merely:

> Does this endpoint exist?

They are:

```text
Who may call it?
What is valid input?
What business operation occurs?
What state changes?
What can fail?
Can the client retry safely?
What response represents success?
What errors are stable?
What data is intentionally exposed?
How does the contract evolve?
```

---

## 3.2 Public API versus internal API

A public API generally requires stronger compatibility discipline.

An internal API may permit:

```text
faster iteration
shared deployment
coordinated versioning
```

But “internal” does not mean “no contract.”

Internal APIs can still become expensive when multiple teams or services depend on them.

---

# 4. Why Does It Exist?

APIs exist to create controlled boundaries.

Without a stable API:

```text
consumer
  ↓
database structure
```

The consumer becomes coupled to implementation details.

With an API:

```text
consumer
  ↓
contract
  ↓
application
  ↓
database
```

The implementation can change without forcing every consumer to understand the internal change.

This is one reason APIs are architecture.

---

# 5. Mental Model

Think of an API as six layers:

```text
            ┌──────────────────────┐
            │  Transport Semantics │
            │ HTTP / headers / URI │
            └──────────┬───────────┘
                       ↓
            ┌──────────────────────┐
            │     Contract         │
            │ schemas / statuses   │
            └──────────┬───────────┘
                       ↓
            ┌──────────────────────┐
            │    Authorization     │
            └──────────┬───────────┘
                       ↓
            ┌──────────────────────┐
            │     Use Case         │
            └──────────┬───────────┘
                       ↓
            ┌──────────────────────┐
            │ Domain + Persistence │
            └──────────────────────┘
```

Cross-cutting properties:

```text
security
observability
reliability
performance
compatibility
```

A strong API design keeps these concerns explicit.

---

# 6. Core Rules

## Rule 1 — Design the contract before the implementation

Do not start with:

```text
database table
→ generate CRUD routes
```

Start with:

```text
consumer goal
→ business capability
→ contract
→ application behavior
→ persistence
```

---

## Rule 2 — Model business resources, not tables

Database:

```text
customer_orders
```

API:

```http
GET /orders
```

The public representation should reflect consumer-facing concepts.

---

## Rule 3 — Be explicit about semantics

Clients must know:

```text
what this request means
what success means
what failure means
whether retry is safe
```

Ambiguous APIs create accidental behavior.

---

## Rule 4 — Never trust object IDs from the client

A valid ID is not proof of authorization.

OWASP identifies Broken Object Level Authorization as a major API risk and recommends considering object-level authorization for functions that use user-supplied object identifiers. citeturn277088search4

---

## Rule 5 — Stable APIs resist unnecessary coupling

Do not expose internal fields merely because they currently exist.

---

## Rule 6 — Errors are part of the API contract

Do not treat errors as random strings.

A client should be able to distinguish:

```text
validation failure
authentication failure
authorization failure
conflict
rate limiting
transient dependency failure
unexpected server failure
```

---

## Rule 7 — Retriability must be designed

An operation that creates state may need idempotency.

For example:

```http
POST /payments
Idempotency-Key: 7d2...
```

can allow the server to recognize an intentional retry.

---

## Rule 8 — Limit resource consumption

APIs need controls around:

```text
body size
query complexity
page size
concurrency
request frequency
expensive searches
uploads
batch operations
```

OWASP explicitly includes unrestricted resource consumption among its API security risks. citeturn277088search4

---

## Rule 9 — Compatibility is a feature

An API is successful when clients can continue operating as the provider evolves.

---

## Rule 10 — Documentation must be executable enough to test

A contract that cannot be validated mechanically becomes tribal knowledge.

---

# 7. Syntax

Basic Node.js HTTP handling:

```js
import http from "node:http";

const server = http.createServer(async (request, response) => {
  if (request.method === "GET" && request.url === "/health") {
    response.writeHead(200, {
      "content-type": "application/json",
    });

    response.end(JSON.stringify({
      status: "ok",
    }));

    return;
  }

  response.writeHead(404, {
    "content-type": "application/json",
  });

  response.end(JSON.stringify({
    error: "Not found",
  }));
});

server.listen(3000);
```

Node's current HTTP API exposes server-level controls including request, header, keep-alive, and connection-related timeout options; production API design should treat these as part of the server's operational boundary rather than relying on infinite waiting. citeturn560872search6

Frameworks can simplify routing and middleware, but the underlying HTTP contract still exists.

---

# 8. Basic Examples

## Example 1 — Resource collection

```http
GET /orders
```

Response:

```json
{
  "data": [
    {
      "id": "ord_123",
      "status": "confirmed",
      "total": 1250
    }
  ]
}
```

---

## Example 2 — Single resource

```http
GET /orders/ord_123
```

Response:

```json
{
  "data": {
    "id": "ord_123",
    "status": "confirmed",
    "total": 1250
  }
}
```

---

## Example 3 — Creation

```http
POST /orders
content-type: application/json
```

```json
{
  "customerId": "cus_42",
  "items": [
    {
      "productId": "prod_10",
      "quantity": 2
    }
  ]
}
```

Possible response:

```http
201 Created
Location: /orders/ord_123
```

---

## Example 4 — Invalid request

```http
422 Unprocessable Content
```

```json
{
  "type": "https://api.example.com/problems/invalid-order",
  "title": "Order is invalid",
  "status": 422,
  "detail": "At least one item is required"
}
```

RFC 9457 standardizes a machine-readable “problem details” representation for HTTP API errors and defines `application/problem+json` for JSON serialization. citeturn277088search5

---

# 9. Execution Walkthrough

A production request:

```http
POST /orders
```

can travel through:

```text
TCP/TLS
  ↓
HTTP parser
  ↓
request size/time limits
  ↓
request ID / trace context
  ↓
authentication
  ↓
authorization
  ↓
routing
  ↓
content-type validation
  ↓
schema validation
  ↓
use-case command
  ↓
domain operation
  ↓
transaction
  ↓
persistence
  ↓
event / side effect handling
  ↓
response mapping
  ↓
metrics / logs / trace
```

Each boundary can fail.

A useful production design identifies which failures are:

```text
client-caused
server-caused
dependency-caused
transient
permanent
retryable
non-retryable
```

---

# 10. Internal Mechanics

## 10.1 URI design

Good:

```text
/orders
/orders/{orderId}
/orders/{orderId}/items
```

Avoid exposing:

```text
/customer_orders_table
```

unless that is truly the conceptual resource.

---

## 10.2 Path parameters

Use path parameters when identifying a resource:

```http
GET /orders/ord_123
```

---

## 10.3 Query parameters

Use query parameters for collection behavior:

```http
GET /orders?status=pending&limit=25
```

Common uses:

```text
filter
sort
pagination
search
projection
```

Do not turn query syntax into an unbounded programming language.

---

## 10.4 Headers

Headers commonly carry metadata:

```text
Authorization
Content-Type
Accept
If-Match
If-None-Match
Idempotency-Key
Traceparent
```

Do not put everything into headers merely because headers exist.

---

## 10.5 Request body

Use the body for structured operation input.

Do not duplicate the same identifier in:

```text
URL
query
body
```

unless there is a specific compatibility reason.

---

# 11. ECMAScript / Specification Semantics

HTTP itself is not part of ECMAScript.

This distinction matters:

```text
ECMAScript
  = JavaScript language semantics

Node.js HTTP
  = host/runtime API

HTTP
  = network protocol

API architecture
  = application/system design
```

JavaScript gives you primitives such as:

```js
Promise
Object
Map
Function
Symbol
async/await
```

It does not give you standard language semantics for:

```text
REST
HTTP authentication
rate limiting
API versioning
transactions
idempotency
OpenAPI
```

Those belong to the host/protocol/application layers.

---

# 12. Advanced Behavior

## 12.1 HTTP method semantics

Common methods:

| Method | Typical Use |
|---|---|
| GET | Read representation |
| POST | Create/process an operation |
| PUT | Replace/create at known URI |
| PATCH | Partial modification |
| DELETE | Remove a resource |
| HEAD | Retrieve headers without representation |
| OPTIONS | Discover communication options |

Never choose a method solely because it “looks right.”

Consider:

```text
safety
idempotency
cache behavior
resource semantics
```

---

## 12.2 Safe versus idempotent

These are different.

A request is **safe** when it is intended not to modify server state.

A request is **idempotent** when repeating it produces the same intended effect as making it once.

Conceptually:

```text
GET → safe + idempotent
PUT → idempotent
DELETE → generally idempotent in semantic effect
POST → generally not idempotent by default
```

The word “generally” matters because actual implementation can violate expectations.

---

## 12.3 Resource versus command endpoints

Resource-oriented:

```http
POST /payments
```

Command-oriented:

```http
POST /orders/{id}/cancel
```

Both can be valid.

Use a command-style endpoint when the business operation itself is clearer than forcing it into generic CRUD.

---

## 12.4 DTO boundaries

Do not return database rows directly:

```js
return dbRow;
```

Map:

```js
function toOrderResponse(order) {
  return {
    id: order.id,
    status: order.status,
    total: order.total,
  };
}
```

This gives the API ownership of its representation.

---

## 12.5 Validation layers

Input validation:

```js
{
  customerId: "string",
  items: [
    {
      productId: "string",
      quantity: "positive integer"
    }
  ]
}
```

Business validation:

```text
customer must be active
product must be purchasable
inventory must be available
```

Authorization:

```text
caller can create an order for this customer
```

These are different questions.

---

# 13. Edge Cases

## 13.1 Duplicate creation

Client:

```text
request
→ timeout
→ retries
```

Server may execute twice.

For expensive or externally visible writes, consider:

```http
Idempotency-Key: abc123
```

Store the key with enough information to safely recognize the prior operation.

---

## 13.2 Lost response

The server can commit successfully and fail before returning a response.

Therefore:

```text
HTTP response != proof that business operation did not occur
```

Clients need retry semantics designed around this possibility.

---

## 13.3 Partial updates

Naive PATCH semantics:

```json
{
  "status": "confirmed"
}
```

may be ambiguous when fields interact.

Define explicitly:

```text
missing field
null
empty string
empty array
zero
```

These may have different meanings.

---

## 13.4 Concurrent updates

Two clients:

```text
A reads version 4
B reads version 4

A updates
B updates
```

B may overwrite A's change.

A version field:

```json
{
  "id": "ord_123",
  "version": 5
}
```

combined with conditional requests such as `If-Match` can enforce optimistic concurrency.

---

## 13.5 Large collections

Never assume:

```http
GET /orders
```

can safely return millions of objects.

Use:

```text
pagination
limits
filters
projections
streaming where appropriate
```

---

# 14. Common Misconceptions

### “REST means CRUD.”

No. REST-style systems use representations and HTTP semantics; CRUD is only one way to model operations.

### “HTTP 200 means success.”

A 2xx response communicates success at the HTTP layer, but the body can still contain domain-specific states that require interpretation.

### “HTTP errors are only status codes.”

Status codes communicate broad semantics. Problem details can supply structured application-specific error information. citeturn277088search5

### “Internal APIs don't need security.”

Internal networks are not automatically trusted.

### “Versioning means `/v1` forever.”

URL versioning is one option. Compatibility policies can also use additive evolution, headers, media types, or contract negotiation.

### “OpenAPI is just Swagger documentation.”

OpenAPI is a language-agnostic interface description that can support documentation, code generation, and testing. The latest published OpenAPI specification is 3.2.0 as of September 19, 2025. citeturn560872search3

---

# 15. Common Mistakes

## Mistake 1 — Exposing database schema

```json
{
  "customer_id": 7,
  "created_at": "...",
  "internal_status_code": 4
}
```

when those fields are implementation artifacts.

---

## Mistake 2 — Inconsistent error shapes

```text
/login → { error: "..." }

/orders → { message: "..." }

/payments → "something went wrong"
```

Clients now need endpoint-specific parsing.

---

## Mistake 3 — Missing authorization on object access

```js
const order = await repository.findById(req.params.id);
return order;
```

This can expose another tenant's or user's object.

---

## Mistake 4 — Unlimited query parameters

A filter API can become an accidental database query language.

---

## Mistake 5 — Unbounded page size

```http
GET /orders?limit=999999999
```

can turn into a resource exhaustion attack.

---

## Mistake 6 — Retrying everything

Do not blindly retry:

```text
validation errors
authorization failures
known business conflicts
```

Retry only when the operation and failure semantics justify it.

---

# 16. Comparison With Related Concepts

| API Style / Technique | Strength | Risk |
|---|---|---|
| REST-style HTTP | Uses standard HTTP semantics | Can become inconsistent without discipline |
| RPC | Explicit operations | Can ignore useful HTTP/resource semantics |
| GraphQL | Consumer-selected projections | Complexity/security/caching trade-offs |
| gRPC | Strong service contracts, efficient binary protocol | Browser/interoperability considerations |
| WebSocket | Persistent bidirectional communication | Stateful connection management |
| Event API | Loose temporal coupling | Delivery/order/consistency complexity |
| `/v1` URL versioning | Easy to understand | Parallel versions can persist too long |
| Header/media-type versioning | Keeps URI stable | Less visible operationally |
| Cursor pagination | Stable on changing datasets | More implementation complexity |
| Offset pagination | Easy to understand | Can degrade or drift on large/changing datasets |

---

# 17. Performance Considerations

API performance is not just server CPU time.

A useful latency model:

```text
total latency
=
queueing
+ application
+ database
+ external dependencies
+ serialization
+ network
```

---

## 17.1 Payload size

Large JSON responses cost:

```text
serialization
network bandwidth
client parsing
memory
GC pressure
```

Use:

```text
pagination
field selection
compression where appropriate
streaming for suitable workloads
```

---

## 17.2 Batching

Bad:

```text
100 HTTP requests
```

when the server could accept:

```http
POST /orders/batch
```

But batching creates its own semantics:

```text
partial failure
ordering
maximum batch size
atomicity
retry behavior
```

Never treat batching as automatically better.

---

## 17.3 Connection limits

Production APIs interact with bounded resources:

```text
socket pool
database pool
CPU
memory
file descriptors
downstream quotas
```

A high request rate with unbounded concurrency can make latency worse.

---

# 18. Memory Considerations

API memory risks include:

```text
large request bodies
large JSON parsing
large response buffering
unbounded pagination
in-memory caches
request context retention
unbounded batch requests
```

For a stream-like payload:

```text
receive chunk
→ process
→ release
```

may be better than:

```text
receive entire payload
→ store whole payload in memory
→ process
```

Node's HTTP APIs expose stream-based request/response objects and configurable stream watermarks; use streaming/backpressure deliberately for large or continuous data. citeturn560872search6

---

# 19. Security Considerations

OWASP's 2023 API Security Top 10 highlights:

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

These should become API design questions, not only penetration-test findings. citeturn277088search4

---

## 19.1 Object authorization

Never assume:

```js
const order = await repository.findById(orderId);
```

means:

```text
caller can access order
```

You need an authorization decision.

---

## 19.2 Property authorization

Returning:

```json
{
  "id": "...",
  "email": "...",
  "internalNotes": "...",
  "adminFlags": "..."
}
```

can expose information even when the object itself is authorized.

---

## 19.3 SSRF

If the API accepts user-controlled URLs:

```json
{
  "callbackUrl": "https://..."
}
```

the server becomes an HTTP client controlled by an external actor.

Validate:

```text
scheme
destination
IP resolution
redirect behavior
network reachability
allowed hosts
```

SSRF is explicitly included in the OWASP API Security Top 10 2023. citeturn277088search4

---

# 20. Production Usage

## 20.1 Recommended API boundary

```text
HTTP adapter
    ↓
request normalization
    ↓
authentication
    ↓
authorization
    ↓
validation
    ↓
application command
    ↓
use case
    ↓
domain
    ↓
repositories / gateways
    ↓
response DTO
```

---

## 20.2 Error taxonomy

Example:

```js
class AppError extends Error {
  constructor(message, {
    code,
    status,
    retryable = false,
  }) {
    super(message);
    this.code = code;
    this.status = status;
    this.retryable = retryable;
  }
}
```

Example:

```js
throw new AppError("Order already exists", {
  code: "ORDER_ALREADY_EXISTS",
  status: 409,
});
```

The HTTP layer decides how to serialize it.

---

## 20.3 Problem details

Example:

```js
function toProblemDetails(error, requestId) {
  return {
    type: `https://api.example.com/problems/${error.code}`,
    title: error.message,
    status: error.status ?? 500,
    instance: `/requests/${requestId}`,
  };
}
```

Set:

```http
Content-Type: application/problem+json
```

RFC 9457 defines this media type and a structured problem-details model. citeturn277088search5

Do not expose:

```text
SQL queries
stack traces
secret configuration
internal hostnames
tokens
database errors
```

to untrusted clients.

---

## 20.4 Pagination

Offset pagination:

```http
GET /orders?offset=100&limit=25
```

Cursor pagination:

```http
GET /orders?cursor=eyJpZCI6...
```

Cursor pagination is often preferable for changing or very large datasets because it avoids some of the drift and deep-offset costs of offset-based approaches.

But cursor design requires:

```text
stable ordering
opaque cursor
expiration policy
validation
```

---

## 20.5 Filtering

Good:

```http
GET /orders?status=confirmed
```

Potentially dangerous:

```http
GET /orders?where={"$or":[...complex query...]}
```

Do not expose internal query languages without strict controls.

---

## 20.6 Sorting

Explicit allow-list:

```js
const allowedSorts = new Set([
  "createdAt",
  "total",
  "status",
]);
```

Reject unknown sorts.

---

## 20.7 Versioning

Common strategies:

### URI

```text
/api/v1/orders
/api/v2/orders
```

### Header

```http
Accept: application/vnd.example.order+json;version=2
```

### Compatibility-first evolution

Prefer additions:

```text
add optional field
add endpoint
add enum value only when clients tolerate it
```

over destructive changes.

---

## 20.8 Deprecation

A mature API lifecycle:

```text
design
→ implement
→ document
→ release
→ observe
→ deprecate
→ migrate
→ retire
```

Deprecation requires:

```text
consumer identification
migration path
communication
observability
deadline
```

---

## 20.9 OpenAPI

An OpenAPI description can serve as:

```text
human documentation
machine-readable contract
client generation input
server generation input
contract testing input
review artifact
```

The current published OpenAPI specification is 3.2.0. citeturn560872search3

Example:

```yaml
openapi: 3.2.0
info:
  title: Orders API
  version: 1.0.0

paths:
  /orders:
    post:
      summary: Create an order
```

Do not treat the OpenAPI file as automatically correct.

Contract quality still depends on:

```text
review
tests
implementation alignment
consumer feedback
```

---

# 21. Implementation From Scratch

Build a production-style API with Node.js.

## Stage 1 — Guided

### Router

```js
export function createRouter(routes) {
  return async function router(request, response) {
    const route = routes.find(
      candidate =>
        candidate.method === request.method &&
        candidate.path === request.url
    );

    if (!route) {
      response.writeHead(404);
      response.end();
      return;
    }

    await route.handler(request, response);
  };
}
```

This is intentionally simplistic.

Production routing must account for:

```text
path parameters
query strings
method semantics
normalization
timeouts
body parsing
errors
```

---

## Stage 2 — Request validation

Create a request boundary:

```js
function validateCreateOrder(input) {
  if (!input || typeof input !== "object") {
    throw new AppError("Invalid body", {
      code: "INVALID_BODY",
      status: 400,
    });
  }

  if (!Array.isArray(input.items) || input.items.length === 0) {
    throw new AppError("Items are required", {
      code: "INVALID_ITEMS",
      status: 422,
    });
  }

  return input;
}
```

---

## Stage 3 — Application boundary

```js
const result = await placeOrder({
  actorId: requestContext.actorId,
  customerId: input.customerId,
  items: input.items,
});
```

The application should not need:

```js
request
response
headers
HTTP status codes
```

---

## Stage 4 — Response mapper

```js
function toOrderResponse(order) {
  return {
    data: {
      id: order.id,
      status: order.status,
      total: order.total,
    },
  };
}
```

---

## Stage 5 — Production hardening

Add:

```text
request body limit
request timeout
response timeout
authentication
authorization
idempotency
structured errors
problem details
pagination
rate limits
audit logging
request IDs
trace context
metrics
health/readiness
graceful shutdown
contract tests
OpenAPI
```

---

# 22. Debugging Exercises

## Exercise 1 — Authorization bypass

```js
GET /orders/123
```

returns order 123 regardless of caller.

Find the authorization boundary.

---

## Exercise 2 — Duplicate payment

Client retries after a timeout and creates two charges.

Design an idempotency strategy.

---

## Exercise 3 — Huge page

```http
GET /orders?limit=1000000
```

causes memory exhaustion.

Design:

```text
maximum limit
validation
query strategy
response constraints
monitoring
```

---

## Exercise 4 — Error leakage

Database failure returns:

```json
{
  "error": "SELECT * FROM customer_secrets ..."
}
```

Explain:

```text
security impact
API contract problem
observability alternative
```

---

## Exercise 5 — N+1 API

A frontend makes:

```text
GET /orders
GET /orders/1/customer
GET /orders/2/customer
GET /orders/3/customer
...
```

Analyze:

```text
latency
network chatter
cache behavior
backend load
API redesign options
```

---

## Exercise 6 — Race condition

Two clients:

```text
GET /inventory/sku-42
→ stock = 1

both
POST /orders
```

Both succeed.

Explain how API-level concurrency control and transactional domain logic should cooperate.

---

## Exercise 7 — SSRF

An endpoint accepts:

```json
{
  "url": "http://..."
}
```

and server-side fetches it.

Threat-model the endpoint.

---

## Exercise 8 — Contract drift

Documentation says:

```json
{
  "status": "pending"
}
```

Implementation returns:

```json
{
  "status": "PENDING"
}
```

Find:

```text
where mismatch originated
how to detect it
how to prevent recurrence
```

---

# 23. Code Review Exercise

Review:

```js
app.post("/orders", async (req, res) => {
  const order = await db.orders.insert({
    customerId: req.body.customerId,
    total: req.body.total,
  });

  await fetch("https://payment.example/charge", {
    method: "POST",
    body: JSON.stringify({
      customerId: req.body.customerId,
      amount: req.body.total,
    }),
  });

  res.json(order);
});
```

Find at least 15 issues.

Potential findings:

```text
missing authentication
missing authorization
missing input validation
database coupled to transport
no transaction strategy
unsafe payment semantics
no idempotency
no timeout
unclear retry policy
external API response ignored
no error translation
no consistent status code
no response schema
database model exposed
no observability
no audit strategy
possible partial failure
```

---

# 24. Interview Questions

## Fundamental

1. What makes an API a contract?
2. URI versus resource?
3. GET versus POST?
4. Safe versus idempotent?
5. PUT versus PATCH?
6. What is content negotiation?
7. Why are errors part of the API contract?
8. What is DTO mapping?
9. Why not expose database rows?
10. What is pagination?

## Intermediate

11. What makes an API backward compatible?
12. How do you design idempotent POST requests?
13. What belongs in headers?
14. When would you use command-style endpoints?
15. Offset versus cursor pagination?
16. How do you design consistent errors?
17. What is optimistic concurrency control?
18. How do rate limits differ from quotas?
19. When is an asynchronous API better?
20. What belongs in an OpenAPI document?

## Advanced

21. How do you evolve an API used by 500 consumers?
22. How do you prevent object-level authorization flaws?
23. How do you design webhook delivery and retries?
24. How do you prevent duplicate side effects?
25. How would you design batch APIs?
26. How do API and database transactions interact?
27. How do you handle downstream timeouts?
28. What is a consumer-driven contract?
29. How do you discover consumers of an API before deprecation?
30. How do you design an API for multi-tenancy?

## Principal

31. Would you choose REST, GraphQL, gRPC, or events? Defend the trade-off.
32. Which API abstractions would you deliberately avoid?
33. How do you distinguish a breaking change from a non-breaking change?
34. How would you migrate an API without a flag day?
35. How do you design APIs that survive organizational growth?
36. How do you reason about API consistency versus local team autonomy?
37. When should an API expose a domain concept versus a workflow/command?
38. How does API design affect database independence?
39. How do security, observability, and compatibility influence contract design?
40. What API decision would you reverse after seeing production evidence?

---

# 25. Predict-the-Output Exercises

## Exercise A — HTTP is outside JavaScript semantics

Predict:

```js
const request = {
  method: "POST",
};

console.log(request.method === "POST");
```

### Actual Result

```text
true
```

### Rule

JavaScript evaluates the object and comparison. HTTP meaning comes from the protocol/application layer, not from the ECMAScript language.

---

## Exercise B — Object identity

Predict:

```js
const a = { status: "pending" };
const b = { status: "pending" };

console.log(a === b);
console.log(a.status === b.status);
```

### Actual Result

```text
false
true
```

### Architectural Lesson

API consumers usually care about representation equality, not JavaScript object identity.

---

## Exercise C — Mutation boundary

Predict:

```js
function toResponse(order) {
  return {
    id: order.id,
    status: order.status,
  };
}

const order = {
  id: "ord_1",
  status: "pending",
  internalSecret: "hidden",
};

const response = toResponse(order);

response.status = "confirmed";

console.log(order.status);
console.log(response.status);
```

### Actual Result

```text
pending
confirmed
```

### Rule

Returning a new representation can protect domain/persistence state from accidental response mutation.

---

# 26. Mastery Exercises

## Track A — Core Theory

### A1

Design an API for:

```text
customers
orders
payments
inventory
```

Define:

```text
resources
operations
authorization
errors
pagination
idempotency
versioning
```

### A2

Given 20 endpoints, classify them as:

```text
resource
query
command
integration
administrative
health
```

Defend every classification.

---

## Track B — Implementation

### B1 — Build

Create:

```text
POST   /orders
GET    /orders/:id
GET    /orders
PATCH  /orders/:id
POST   /orders/:id/cancel
```

Requirements:

```text
validation
authorization
error mapping
pagination
request IDs
idempotency
concurrency protection
```

### B2 — Contract

Write an OpenAPI description for the API.

Validate that:

```text
request schema
response schema
status codes
security requirements
parameters
```

match implementation.

The current published OpenAPI version is 3.2.0; use the version selected by your tooling if your project is pinned to another compatible release. citeturn560872search3

### B3 — Reliability

Simulate:

```text
timeout
duplicate request
database conflict
external provider failure
client disconnect
large request
```

Document behavior for each.

---

## Track C — Interview / Reasoning

### C1

A company has 3 million API requests per hour.

Design:

```text
rate limiting
pagination
caching
timeouts
observability
authorization
failure isolation
```

### C2

An API has:

```text
200 consumers
5 years of history
weekly releases
one major schema migration
```

Create an evolution strategy.

### C3

A payment API occasionally times out after success.

Design the client/server contract so the client can safely determine eventual state.

---

# 27. Key Takeaways

1. An API is a behavioral contract, not a URL list.
2. Resource modeling should reflect consumer/business concepts.
3. HTTP method semantics matter.
4. Safe and idempotent are different concepts.
5. Errors are part of compatibility.
6. DTO mapping protects API contracts from internal models.
7. Validation, authorization, and business invariants answer different questions.
8. Object IDs never imply authorization.
9. Property-level exposure must be controlled.
10. Resource consumption must be bounded.
11. Retries require explicit semantics.
12. Idempotency is essential for many retryable writes.
13. Pagination is a correctness and performance concern.
14. Cursor pagination can help large, changing datasets.
15. Optimistic concurrency prevents some lost-update problems.
16. Caching introduces consistency and invalidation trade-offs.
17. Versioning is broader than putting `/v1` in a URL.
18. Deprecation requires consumer visibility and migration paths.
19. OpenAPI can be a contract artifact, not merely documentation.
20. API design is architecture because the contract creates dependency across systems.

---

# 28. Concept Connections

## Depends On

- **Chapter 29** — Errors / Error Handling
- **Chapter 31–40** — Async / concurrency
- **Chapter 55** — Fetch / HTTP Networking
- **Chapter 56–57** — Security
- **Chapter 58–63** — Node.js
- **Chapter 67** — Dependency / supply-chain awareness
- **Chapter 70** — Production debugging
- **Chapter 78** — Production JavaScript Architecture

## Builds Toward

- **Chapter 80** — Library Authoring
- **Chapter 81** — Database Integration
- **Chapter 82** — API Architecture
- **Chapter 83** — Observability
- **Chapter 84** — Reliability
- **Chapter 85** — Performance
- **Chapter 86** — Testing
- **Chapter 89** — Code Review / Refactoring
- **Chapter 101** — Real-World Production Scenarios
- **Chapter 105** — Node REST API
- **Chapter 110** — Production JavaScript Backend
- **Chapter 121** — Principal System Design

## Related Concepts

```text
API Design
  ├─ HTTP semantics
  ├─ schemas
  ├─ validation
  ├─ authorization
  ├─ idempotency
  ├─ pagination
  ├─ concurrency
  ├─ caching
  ├─ versioning
  ├─ observability
  └─ security
```

## Concepts Revisited

```text
Promises
Errors
Streams
Modules
Dependency injection
Transactions
Concurrency
Security boundaries
Observability
```

## Why This Chapter Matters Later

The API is often the most visible contract in a production JavaScript system.

Later chapters go deeper into:

```text
database behavior
API architecture
observability
reliability
performance
testing
```

This chapter establishes the contract those components must serve.

---

# 29. Completion Criteria

Mark:

```text
[~] In Progress
```

when you understand basic HTTP API semantics and resource modeling.

Mark:

```text
[?] Needs Revision
```

when you repeatedly confuse:

```text
authentication vs authorization
safe vs idempotent
validation vs business rules
HTTP status vs domain result
API contract vs database schema
retry vs idempotency
```

Mark:

```text
[+] Completed
```

when you can:

- design a complete API contract;
- map domain errors to stable API errors;
- design pagination;
- design idempotency;
- design concurrency controls;
- document an API using OpenAPI;
- implement a production-style Node API.

Mark:

```text
[*] Mastered
```

only when you can:

- critique an unfamiliar API;
- discover hidden compatibility risks;
- identify authorization failures;
- design retry-safe workflows;
- defend REST/RPC/GraphQL/event choices;
- evolve a large API without a flag day;
- reason about performance and failure under load;
- translate business workflows into coherent contracts.

Reading alone does not qualify as mastery.

---

# Chapter 79 — Revision / Retrieval Record

| Date | Retrieval Goal | Result | Status |
|---|---|---|---|
| ____ | Explain API as a behavioral contract | ____ | `[ ]` |
| ____ | Design resource and command endpoints | ____ | `[ ]` |
| ____ | Explain safe vs idempotent | ____ | `[ ]` |
| ____ | Design consistent API errors | ____ | `[ ]` |
| ____ | Design idempotent retryable writes | ____ | `[ ]` |
| ____ | Design pagination and concurrency controls | ____ | `[ ]` |
| ____ | Analyze API security failures | ____ | `[ ]` |
| ____ | Defend an API evolution strategy | ____ | `[ ]` |

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

1. Define API contract.
2. What makes a request idempotent?
3. Why can POST require idempotency?
4. Explain object-level authorization.
5. Offset versus cursor pagination.
6. Why map database rows to DTOs?
7. Explain optimistic concurrency.
8. Design a structured error response.
9. List five API resource-exhaustion controls.
10. Explain a safe API migration.

---

# Chapter 79 — Canonical References and Source Discipline

This chapter distinguishes:

```text
ECMAScript
Node.js
HTTP
OpenAPI
API security
application architecture
```

Do not cite JavaScript language semantics as if they were HTTP rules.

## 1. ECMAScript

Use:

- https://tc39.es/ecma262/

For:

```text
JavaScript syntax
objects
functions
modules
promises
language semantics
```

## 2. Node.js HTTP

The current Node.js HTTP documentation describes `http.createServer()` and server controls including request/header timeouts and connection behavior. Verify version-specific operational behavior against the Node.js version deployed by your organization. citeturn560872search6

- https://nodejs.org/api/http.html

## 3. HTTP API error contracts

RFC 9457 defines Problem Details for HTTP APIs, including the `application/problem+json` format. citeturn277088search5

- https://www.rfc-editor.org/rfc/rfc9457.html

## 4. OpenAPI

The OpenAPI Initiative currently lists **OpenAPI 3.2.0** as the latest published specification, dated September 19, 2025. citeturn560872search3

- https://spec.openapis.org/oas/latest.html

## 5. OWASP API Security

The OWASP API Security Top 10 2023 identifies ten API-specific security risks including object-level authorization, broken authentication, unrestricted resource consumption, SSRF, security misconfiguration, improper inventory management, and unsafe consumption of APIs. citeturn277088search4

- https://owasp.org/API-Security/

## Source Discipline

For every API design decision, classify the source:

```text
Protocol requirement
Standard specification
Runtime behavior
Framework behavior
Application policy
Organizational convention
```

Never say:

> “HTTP requires this”

when the rule is actually:

> “Our application chooses this.”

---

# Chapter 79 — Completion Snapshot

## Core Theory

- [ ] API as behavioral contract
- [ ] Resource modeling
- [ ] HTTP methods
- [ ] Status codes
- [ ] Headers
- [ ] Request schemas
- [ ] Response schemas
- [ ] DTO boundaries
- [ ] Validation layers
- [ ] Error contracts
- [ ] Problem Details
- [ ] Idempotency
- [ ] Pagination
- [ ] Filtering
- [ ] Sorting
- [ ] Concurrency control
- [ ] Caching
- [ ] Versioning
- [ ] Deprecation
- [ ] Webhooks
- [ ] Async APIs
- [ ] OpenAPI
- [ ] API security
- [ ] Resource limits
- [ ] Observability

## Implementation

- [ ] Build Node HTTP API
- [ ] Build router
- [ ] Validate requests
- [ ] Add authentication
- [ ] Add authorization
- [ ] Map DTOs
- [ ] Implement error boundary
- [ ] Implement problem details
- [ ] Add idempotency
- [ ] Add pagination
- [ ] Add optimistic concurrency
- [ ] Add rate limits
- [ ] Add request limits
- [ ] Add timeouts
- [ ] Add structured logging
- [ ] Add metrics
- [ ] Add tracing
- [ ] Add OpenAPI
- [ ] Add contract tests
- [ ] Add integration tests

## Interview / Reasoning

- [ ] Explain REST vs RPC
- [ ] Explain safe vs idempotent
- [ ] Design retry-safe POST
- [ ] Design pagination
- [ ] Design API authorization
- [ ] Identify contract drift
- [ ] Design API versioning
- [ ] Design deprecation
- [ ] Threat-model webhook/URL APIs
- [ ] Defend a principal-level API decision

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

Given an unfamiliar production API, can you answer:

```text
Who are its consumers?
What is the contract?
What are the resources?
Which operations are commands?
Which requests are retry-safe?
How is idempotency implemented?
How is authorization enforced?
How are object IDs protected?
How are properties protected?
How are errors represented?
How is pagination bounded?
How are large payloads handled?
How is concurrency controlled?
How is the API versioned?
How are consumers discovered?
How is deprecation managed?
What happens when dependencies time out?
How is abuse detected?
How is the API observed?
What would break if the database schema changed?
What would break if a downstream provider changed?
```

A principal API designer should be able to answer these from the contract and architecture—not only from implementation code.

---

## Principal API Decision Framework

For every important API design choice, record:

```text
Consumer Need:
Business Capability:
Contract:
Alternatives:
Correctness:
Performance:
Memory:
Security:
Reliability:
Maintainability:
Scalability:
Observability:
Compatibility:
Operational Complexity:
Migration Cost:
Decision:
Revisit Trigger:
```

The best API is not the one with the most elegant endpoints.

It is the one whose **semantics, failure modes, security boundaries, performance characteristics, and evolution strategy are explicit enough that independent consumers can depend on it safely.**