# Chapter 105 — Production Node.js REST API

> **JavaScript Mastery — Part XX: Projects**
>
> **Project:** Build a production-grade REST API with Node.js and JavaScript/TypeScript-friendly architectural boundaries, explicit HTTP contracts, persistence, authentication, authorization, reliability, observability, testing, and deployment discipline.
>
> **Role perspective:** Principal JavaScript Engineer · Node.js Architect · API Designer · Backend Engineer · Database Engineer · Security Engineer · Reliability Engineer
>
> **Status:** `[ ] Not Started`
>
> **Verification date:** 2026-09-11
>
> **Core rule:** **A REST API is a system contract, not merely a collection of route handlers. Its quality depends on resource semantics, validation, authorization, persistence, concurrency, failures, observability, security, compatibility, and operational behavior.**

---

# 1. Project Mission

Build a production-grade REST API for:

```text
FocusBoard
```

The API manages:

```text
users
organizations
projects
tasks
comments
```

It should support:

```text
authentication
authorization
CRUD
search
filtering
pagination
sorting
validation
transactions
optimistic concurrency
idempotent writes
rate limits
observability
tests
health checks
graceful shutdown
```

Advanced targets:

```text
background jobs
audit events
caching
outbox pattern
multi-tenant isolation
OpenAPI
versioning
distributed tracing
```

---

# 2. Learning Objectives

By completing this project, you should be able to:

- Design REST resources.
- Design stable HTTP contracts.
- Choose HTTP methods intentionally.
- Choose status codes intentionally.
- Define request/response schemas.
- Validate untrusted input.
- Separate transport from domain logic.
- Separate domain from persistence.
- Design authentication.
- Design authorization.
- Prevent object-level authorization bugs.
- Design multi-tenant isolation.
- Implement pagination.
- Implement filtering.
- Implement sorting.
- Avoid unbounded result sets.
- Design consistent error responses.
- Implement database transactions.
- Understand transaction boundaries.
- Handle concurrent updates.
- Implement optimistic concurrency.
- Implement idempotent mutations.
- Handle retries safely.
- Design rate limits.
- Design timeouts.
- Design dependency failures.
- Implement health/readiness endpoints.
- Implement graceful shutdown.
- Implement structured logging.
- Implement metrics.
- Implement tracing hooks.
- Protect secrets.
- Prevent injection.
- Protect against SSRF where relevant.
- Test API contracts.
- Test integration with a real database.
- Test authorization boundaries.
- Test concurrency.
- Test failure paths.
- Load test critical endpoints.
- Profile slow endpoints.
- Design API versioning.
- Document the API.
- Package and deploy the service.
- Defend the architecture at principal level.

---

# 3. Prerequisites

Recommended:

```text
Chapter 29 — Errors
Chapter 31–38 — Async / Promises / Cancellation / Streams
Chapter 55 — Fetch / HTTP Networking
Chapter 57 — JavaScript Security Engineering
Chapter 58–63 — Node.js Architecture / Core APIs / Lifecycle / Diagnostics
Chapter 64–70 — ESM / CommonJS / package.json / dependencies / tooling
Chapter 71–73 — Data Structures / Complexity / Algorithms
Chapter 78–85 — Production Architecture / API / Database / Observability / Reliability / Performance
Chapter 86–89 — Testing / Debugging / Review
Chapter 94–97 — Compatibility / Legacy / Wasm / Edge
Chapter 98–101 — Judgment
Chapter 102 — CLI
Chapter 103 — Browser App
Chapter 104 — HTTP Client

---

# 4. Recommended Architecture

Use:

```text
HTTP
 ↓
Routing
 ↓
Controller
 ↓
Application Service
 ↓
Domain
 ↓
Repository
 ↓
Database
```

Cross-cutting:

```text
authentication
authorization
validation
logging
metrics
tracing
error mapping
configuration
```

---

# 5. Project Structure

Target:

```text
api/
├── package.json
├── README.md
├── src/
│   ├── main.js
│   ├── app.js
│   ├── config/
│   ├── http/
│   │   ├── routes/
│   │   ├── controllers/
│   │   ├── middleware/
│   │   └── errors/
│   ├── application/
│   │   ├── users/
│   │   ├── projects/
│   │   └── tasks/
│   ├── domain/
│   │   ├── users/
│   │   ├── projects/
│   │   └── tasks/
│   ├── infrastructure/
│   │   ├── db/
│   │   ├── repositories/
│   │   └── telemetry/
│   └── security/
├── test/
│   ├── unit/
│   ├── integration/
│   ├── contract/
│   └── fixtures/
├── migrations/
└── docs/
```

This structure can change when justified.

Do not create empty layers solely to match a diagram.

---

# 6. Runtime Boundary

The application should clearly distinguish:

```text
startup
request handling
background work
shutdown
```

---

# 7. App Construction

Prefer:

```js
const app =
  createApp({
    config,
    logger,
    db
  });
```

over importing mutable process-wide application state everywhere.

This improves:

```text
testing
composition
ownership
```

---

# 8. Server Entry Point

Keep:

```text
server startup
```

thin.

Concept:

```js
const app = await createApp(dependencies);

const server =
  app.listen(port);
```

---

# 9. HTTP Server Choice

You may begin with:

```text
Node http
```

or use a mature framework.

The project objective is architectural understanding, not framework avoidance.

---

# 10. Framework Boundary

If using a framework, keep framework-specific behavior near:

```text
HTTP/controller layer
```

Do not spread framework types throughout the domain unless the trade-off is deliberate.

---

# 11. Resource Model

Resources:

```text
users
organizations
projects
tasks
comments
```

Define relationships explicitly.

---

# 12. Example Resource

```json
{
  "id": "task_123",
  "projectId": "project_42",
  "title": "Implement API",
  "status": "open",
  "version": 3,
  "createdAt": "2026-09-11T00:00:00Z",
  "updatedAt": "2026-09-11T00:00:00Z"
}
```

Document the semantic meaning of each field.

---

# 13. ID Strategy

Choose:

```text
UUID
ULID
database-generated integer
domain-specific ID
```

based on:

```text
ordering
exposure
storage
generation
distribution
```

Do not choose only because a technology is fashionable.

---

# 14. Resource Naming

Prefer nouns:

```text
/projects
/tasks
```

Avoid action-heavy URLs such as:

```text
/createTask
```

unless the endpoint represents a meaningful action rather than a resource mutation.

---

# 15. HTTP Methods

Typical mapping:

```text
GET
POST
PUT
PATCH
DELETE
```

Use method semantics deliberately.

---

# 16. GET List

```http
GET /tasks
```

May support:

```text
projectId
status
search
page
limit
sort
```

---

# 17. GET Single

```http
GET /tasks/:id
```

The server must still enforce authorization.

A valid ID does not imply permission.

---

# 18. POST

```http
POST /tasks
```

Create a task.

If retries are possible, consider idempotency.

---

# 19. PATCH

```http
PATCH /tasks/:id
```

Partial update semantics should be explicit.

---

# 20. DELETE

```http
DELETE /tasks/:id
```

Define behavior for repeated deletion.

---

# 21. PUT

Use when replacement/idempotent semantics genuinely fit.

Do not use PUT merely because it is a known HTTP verb.

---

# 22. Status Codes

Define a project policy such as:

```text
200 OK
201 Created
202 Accepted
204 No Content
400 Bad Request
401 Unauthorized
403 Forbidden
404 Not Found
409 Conflict
422 Unprocessable Content where appropriate
429 Too Many Requests
500 Internal Server Error
502/503/504 dependency failures where appropriate
```

Document the cases.

---

# 23. 401 vs 403

Conceptually:

```text
401
→ authentication required/invalid

403
→ authenticated but not permitted
```

Do not use them interchangeably without reason.

---

# 24. 404 and Authorization

Whether to reveal existence can be security-sensitive.

For some resources:

```text
unauthorized resource
→ 404
```

may reduce information disclosure.

This is a policy choice.

---

# 25. 409 Conflict

Use for meaningful state conflicts such as:

```text
optimistic version mismatch
duplicate unique resource
workflow conflict
```

---

# 26. 422

Use when the request is syntactically valid but semantically invalid, if that matches the API's chosen contract.

---

# 27. 429

Use for rate limiting.

Include useful retry guidance where appropriate.

---

# 28. Error Contract

Use a stable shape:

```json
{
  "error": {
    "code": "TASK_NOT_FOUND",
    "message": "Task not found",
    "details": {}
  }
}
```

Human wording can evolve.

Machine-readable codes should be stable.

---

# 29. Error Correlation

Include a correlation/request identifier where useful:

```json
{
  "error": {
    "code": "INTERNAL_ERROR",
    "requestId": "req_123"
  }
}
```

Do not expose sensitive internal details.

---

# 30. Error Layering

Architecture:

```text
domain error
→ application error
→ HTTP error mapping
```

The domain should not know HTTP status codes.

---

# 31. Validation

Validate:

```text
body
query
params
headers
```

at the HTTP boundary.

---

# 32. Runtime Schema Validation

Use a runtime schema system or equivalent validation logic.

Static typing alone does not validate JSON.

---

# 33. Validate Early

Reject invalid input before expensive operations.

---

# 34. Validate Again Where Trust Changes

A domain service can still protect critical invariants.

Do not assume every caller passed through the HTTP layer.

---

# 35. Validation vs Sanitization

Validation asks:

```text
Is this allowed?
```

Sanitization/transformation asks:

```text
How should this value be represented safely?
```

Do not confuse them.

---

# 36. Strict vs Permissive Schemas

Strict schemas:

```text
catch unexpected input
```

Permissive schemas:

```text
can ease compatibility
```

Choose based on API evolution policy.

---

# 37. Unknown Fields

Decide:

```text
reject
ignore
preserve
```

and document it.

---

# 38. Pagination

Never let public list endpoints return unlimited records.

Use:

```text
limit
cursor
```

or another bounded scheme.

---

# 39. Cursor Pagination

Cursor-based pagination is often better for large/changing datasets.

Define cursor semantics.

---

# 40. Offset Pagination

Offset can be simple and useful for some workloads.

It can become expensive or unstable for large changing datasets.

---

# 41. Maximum Page Size

Set:

```text
limit <= max
```

Do not allow:

```text
?limit=10000000
```

---

# 42. Sorting

Whitelist allowed fields:

```text
createdAt
updatedAt
title
```

Do not interpolate arbitrary client strings into SQL.

---

# 43. Filtering

Use explicit validated filters.

Example:

```text
status=open
projectId=...
```

---

# 44. Search

Define:

```text
case sensitivity
tokenization
partial matching
```

Do not assume database string matching is automatically performant.

---

# 45. N+1

Avoid:

```text
list tasks
→ query project for every task
```

Use:

```text
join
batch
prefetch
```

where appropriate.

---

# 46. Response Shape

Choose between:

```text
resource
```

or:

```text
{
  "data": ...
}
```

or:

```text
{
  "data": [...],
  "meta": ...
}
```

Consistency matters more than following a universal style.

---

# 47. Pagination Response

Example:

```json
{
  "data": [...],
  "pagination": {
    "nextCursor": "..."
  }
}
```

Do not expose implementation-specific database details.

---

# 48. Nested Resources

Example:

```http
/projects/:projectId/tasks
```

Use nesting when it expresses real scope.

Avoid excessive nesting:

```text
/org/:org/project/:project/task/:task/comment/:comment
```

---

# 49. Resource Authorization

Every protected resource access should answer:

```text
Who?
What resource?
Which operation?
Which tenant?
Which state?
```

---

# 50. Authentication

Choose a mechanism:

```text
session
Bearer token
JWT
mTLS
```

based on deployment and client needs.

Authentication identifies the caller.

---

# 51. Authorization

Authorization answers:

```text
Is this caller allowed to perform this action on this resource?
```

---

# 52. RBAC

Example:

```text
owner
admin
member
viewer
```

Useful when role semantics are stable.

---

# 53. ABAC

Attribute-based rules:

```text
tenant
department
ownership
resource state
```

Can express richer policies.

---

# 54. Relationship-Based Authorization

Example:

```text
user belongs to organization
organization owns project
project contains task
```

Authorization can derive from relationships.

---

# 55. Object-Level Authorization

Classic vulnerability:

```http
GET /tasks/123
```

where user changes:

```text
123 → 124
```

and accesses another user's task.

Always check resource authorization server-side.

---

# 56. Multi-Tenant Isolation

Tenant context should be explicit.

Example:

```js
{
  userId,
  tenantId,
  roles
}
```

Do not store tenant state globally.

---

# 57. Query Scoping

Database queries should apply tenant/resource scope where required.

A useful safety strategy is to make incorrect unscoped access harder to express.

---

# 58. Authorization Service

Centralize policy semantics where useful:

```js
can(user, "task:update", task)
```

Avoid duplicating authorization logic across controllers.

---

# 59. Authentication Middleware

Middleware can establish:

```text
principal
```

It should not necessarily decide every resource permission.

---

# 60. Domain Authorization

Critical invariants may still belong close to domain operations.

---

# 61. Passwords

Never store plaintext passwords.

Use a dedicated password hashing mechanism appropriate to the application.

Do not invent cryptography.

---

# 62. Tokens

Protect:

```text
access tokens
refresh tokens
session identifiers
```

Do not log them.

---

# 63. Secret Management

Configuration may arrive through:

```text
environment
secret manager
runtime configuration
```

Do not hard-code production secrets.

---

# 64. Authentication Rotation

Design:

```text
key rotation
token expiry
revocation strategy
```

before production.

---

# 65. SQL Injection

Never concatenate untrusted values into SQL.

Use:

```text
parameterized queries
trusted query builders
ORM mechanisms
```

---

# 66. Dynamic ORDER BY

Do not parameterize SQL identifiers as if they were ordinary values.

Whitelist allowed sort fields.

---

# 67. NoSQL Injection

Do not assume a NoSQL database eliminates injection risks.

Validate structured operators and types.

---

# 68. SSRF

If the API fetches caller-supplied URLs:

```text
SSRF risk
```

Apply:

```text
allowlist
network egress policy
redirect control
credential isolation
```

---

# 69. Path Traversal

For file operations:

```text
normalize
validate
bound to allowed root
```

---

# 70. XSS

APIs can still become XSS sources by returning unsafe data to browser clients.

The API should not assume consumers will always escape output correctly.

---

# 71. Mass Assignment

Do not blindly map client JSON to persistence models.

Bad:

```js
Object.assign(user, req.body);
```

This can let callers modify:

```text
role
tenantId
permissions
```

---

# 72. Explicit Write Models

Define:

```text
CreateTaskInput
UpdateTaskInput
```

with allowed fields.

---

# 73. CORS

If browser clients consume the API, configure CORS deliberately.

CORS is not authentication.

---

# 74. CSRF

Cookie-authenticated browser clients need an appropriate CSRF strategy where relevant.

---

# 75. Rate Limiting

Protect:

```text
login
password reset
search
expensive reports
public APIs
```

Use a policy appropriate to the threat.

---

# 76. Rate-Limit Dimensions

Possible:

```text
IP
user
API key
tenant
route
```

One dimension rarely fits every endpoint.

---

# 77. Request Size Limits

Limit:

```text
JSON body
headers
uploads
```

according to endpoint needs.

---

# 78. Response Size Limits

Avoid accidentally returning:

```text
millions of records
```

or huge aggregated responses.

---

# 79. Timeout Strategy

Define:

```text
server request timeout
database timeout
outbound HTTP timeout
queue timeout
```

Do not rely on one universal timeout.

---

# 80. Outbound Dependencies

Use the HTTP client from Chapter 104 or an equivalent production client boundary.

Do not use raw `fetch()` inconsistently across the codebase.

---

# 81. Retry Policy

Only retry safe operations where:

```text
operation is replayable
error is retryable
deadline remains
```

---

# 82. Idempotency

For create/payment/action endpoints:

```text
Idempotency-Key
```

plus server-side deduplication where appropriate.

---

# 83. Idempotency Storage

Store enough information to safely answer repeated requests:

```text
key
request identity
result/status
expiry
```

Do not let keys live forever without reason.

---

# 84. Concurrency Control

Two clients update:

```text
title
```

at the same time.

Choose:

```text
last write wins
optimistic concurrency
locking
merge
```

---

# 85. Optimistic Concurrency

Resource:

```json
{
  "id": "task_1",
  "version": 7
}
```

Client sends:

```text
expectedVersion=7
```

Server rejects if current version differs.

---

# 86. ETag / If-Match

HTTP-level concurrency can use:

```http
ETag
If-Match
```

when appropriate.

---

# 87. Transaction Boundary

Example:

```text
create task
+
audit record
```

may need one transaction.

External API calls usually should not remain inside a long DB transaction.

---

# 88. Transaction Anti-Pattern

Bad:

```text
BEGIN
→ update DB
→ call payment API
→ call email API
→ commit
```

This holds DB resources while waiting on networks.

---

# 89. Transaction + Outbox

For reliable events:

```text
DB transaction
  ├── business state
  └── outbox event
```

Then:

```text
outbox worker
→ publish event
```

---

# 90. Outbox Benefits

Provides a durable connection between:

```text
database state
```

and:

```text
event publication
```

without requiring a distributed transaction.

---

# 91. Audit Log

Record important mutations:

```text
who
what
when
resource
request ID
```

Avoid storing sensitive values unnecessarily.

---

# 92. Soft Delete

Consider when:

```text
audit
recovery
references
```

matter.

Trade-offs:

```text
query complexity
storage
uniqueness
```

---

# 93. Hard Delete

Use when deletion semantics require actual removal and dependencies are manageable.

---

# 94. Database Constraints

Do not rely exclusively on application validation.

Use DB constraints for:

```text
uniqueness
foreign keys
not-null
```

where appropriate.

---

# 95. Race-Safe Uniqueness

Application check:

```text
does email exist?
```

then insert.

Two requests can race.

The database unique constraint is the final protection.

---

# 96. Pagination + Indexing

Indexes should match important query patterns:

```text
tenantId
projectId
status
createdAt
```

based on actual workload.

---

# 97. Query Plan

Measure slow queries.

Do not guess that an index helps.

---

# 98. Connection Pool

Tune against:

```text
DB capacity
request concurrency
query time
deployment topology
```

---

# 99. Connection Leak

Every transaction/connection acquisition needs a lifecycle.

Release in:

```text
finally
```

or equivalent managed scope.

---

# 100. Request Context

Carry:

```text
requestId
trace context
principal
tenant
deadline
abort signal
```

without global mutable storage.

---

# 101. Structured Logging

Example:

```js
logger.info("task_created", {
  taskId,
  projectId,
  requestId
});
```

---

# 102. Avoid Sensitive Logging

Do not log:

```text
password
authorization
tokens
payment details
full personal records
```

---

# 103. Metrics

Useful API metrics:

```text
http_requests_total
http_request_duration
http_request_errors
db_query_duration
db_pool_active
db_pool_waiting
```

---

# 104. Route Dimensions

Use normalized route templates:

```text
/tasks/:id
```

rather than raw IDs.

---

# 105. Tracing

Trace:

```text
request
→ DB
→ outbound API
→ queue
```

where supported.

---

# 106. Correlation

A request ID should be propagated across important internal boundaries.

---

# 107. Health Endpoint

```http
GET /health
```

should have a clear purpose.

---

# 108. Readiness

Readiness may consider:

```text
database connectivity
critical dependencies
startup completion
```

---

# 109. Liveness

Liveness should answer:

```text
is the process fundamentally alive?
```

Do not make liveness depend on every external dependency.

---

# 110. Graceful Shutdown

On:

```text
SIGTERM
SIGINT
```

perform:

```text
stop accepting new work
finish/abort in-flight work according to policy
close DB
close queues
close server
```

Use a deadline.

---

# 111. Shutdown Ordering

Conceptually:

```text
stop intake
→ drain
→ close dependencies
→ exit
```

---

# 112. Startup Failure

If required dependency/configuration is invalid, fail fast rather than serving a broken process.

---

# 113. Configuration

Validate required configuration at startup.

Do not discover invalid configuration halfway through a request.

---

# 114. Environment Separation

Do not use:

```text
production DB
```

from:

```text
test
```

without explicit safeguards.

---

# 115. Database Migrations

Treat migrations as production code.

Test:

```text
up
down where supported
compatibility
large data
```

---

# 116. Expand/Contract Migration

For zero/low-downtime changes:

```text
expand
→ deploy compatible code
→ migrate
→ switch
→ contract
```

---

# 117. Backward-Compatible Schema

Old and new application versions may coexist during rolling deployments.

Schema changes must account for this.

---

# 118. API Versioning

Options include:

```text
/path version
header/media type
compatibility without explicit version
```

Choose one strategy intentionally.

---

# 119. Deprecation

When removing API behavior:

```text
announce
measure usage
provide migration
set sunset
remove
```

---

# 120. Idempotent GET

GET handlers should avoid accidental mutation.

---

# 121. Safe Retries

The API should tolerate network uncertainty where business semantics require repeated attempts.

---

# 122. Request Deduplication

For suitable operations, deduplicate identical in-flight work when it preserves correctness.

---

# 123. Caching

Consider:

```text
HTTP cache
application cache
Redis
database cache
```

Only when workload justifies it.

---

# 124. Cache Invalidation

Define:

```text
TTL
write-through
explicit invalidation
versioning
```

before adding cache.

---

# 125. Cache Key

Include every dimension affecting result correctness:

```text
tenant
user/permissions when relevant
resource
locale
version
```

---

# 126. Cache Stampede

Protect hot keys with:

```text
single-flight
jitter
stale-while-revalidate
```

when appropriate.

---

# 127. Background Jobs

Move long operations such as:

```text
export
email
large report
image processing
```

to a queue where request latency should remain bounded.

---

# 128. 202 Accepted

Use when work is intentionally asynchronous and the request does not complete the final operation immediately.

---

# 129. Job Resource

Example:

```http
POST /exports
```

returns:

```json
{
  "id": "job_123",
  "status": "queued"
}
```

Then:

```http
GET /exports/job_123
```

---

# 130. Job Idempotency

Repeated submission should not accidentally create duplicate jobs when the workflow requires deduplication.

---

# 131. API File Uploads

Define:

```text
size
type
storage
virus scanning
permissions
```

Do not trust client MIME types.

---

# 132. File Storage Boundary

Prefer object storage or dedicated storage systems for large artifacts.

Do not use API process memory as durable file storage.

---

# 133. Streaming Uploads

Stream large request bodies when appropriate.

Avoid:

```text
read entire file
→ memory
```

for arbitrary large uploads.

---

# 134. Download Authorization

Check permissions before generating a signed/object URL.

---

# 135. Data Exposure

Do not return database entities directly.

Map:

```text
persistence model
→ public response model
```

---

# 136. Secret/Private Fields

Exclude fields such as:

```text
passwordHash
internal notes
service credentials
```

from external responses.

---

# 137. DTO Boundary

Use explicit request/response representations.

This creates a compatibility boundary.

---

# 138. Serialization

Avoid accidental serialization of:

```text
database clients
errors
internal metadata
private fields
```

---

# 139. Error Serialization

Never serialize raw internal error objects blindly.

They can contain:

```text
SQL
paths
credentials
stack traces
```

---

# 140. HTTP Middleware Ordering

Be deliberate about:

```text
request ID
security
body parsing
authentication
rate limit
authorization
routing
error handler
```

Order affects semantics.

---

# 141. Rate Limiting Before Expensive Work

Put rate limiting before expensive:

```text
DB
CPU
external calls
```

where appropriate.

---

# 142. Authorization Before Data Fetch

When possible, avoid retrieving sensitive objects merely to discover the caller is forbidden.

Sometimes the policy requires resource lookup; minimize exposure.

---

# 143. Parsing Large JSON

Huge JSON can block the event loop during parsing.

Use:

```text
body limits
streaming formats
async processing
```

where appropriate.

---

# 144. CPU-Heavy Endpoints

Possible strategies:

```text
algorithm optimization
worker
queue
separate service
Wasm
```

Measure first.

---

# 145. Event Loop Protection

Avoid:

```text
catastrophic regex
huge synchronous loops
massive JSON serialization
synchronous filesystem calls
```

in request paths.

---

# 146. Request Abort

If the client disconnects, the server should decide whether downstream work can be canceled safely.

Do not automatically cancel business-critical work if the business contract says it must complete.

---

# 147. Client Disconnect vs Operation Cancellation

These are not always equivalent.

Example:

```text
payment request
client disconnects
```

The payment may still need to complete.

---

# 148. Deadline Propagation

Propagate request deadlines to:

```text
DB
HTTP dependencies
queues
```

where supported.

---

# 149. Backpressure

Protect:

```text
DB
queue
external API
CPU
memory
```

with bounded concurrency.

---

# 150. Load Shedding

When capacity is exhausted, reject low-priority work rather than allowing all requests to degrade indefinitely.

---

# 151. Priority

Business-critical endpoints may need different:

```text
rate
capacity
queue
```

policies.

---

# 152. Multi-Tenant Fairness

Avoid allowing one tenant to consume all shared capacity.

Use:

```text
quotas
rate limits
separate queues
```

when justified.

---

# 153. Database Transaction + Request Deadline

A request deadline should not create transactions that remain open beyond useful work.

---

# 154. Security Headers

Configure appropriate security headers at the HTTP boundary for the deployment.

Do not blindly copy a list without understanding which are applicable.

---

# 155. TLS Termination

Know whether TLS terminates at:

```text
load balancer
proxy
Node
```

and preserve trusted client/network metadata correctly.

---

# 156. Forwarded Headers

Do not blindly trust:

```text
X-Forwarded-For
```

or similar headers unless the request came through a trusted proxy boundary.

---

# 157. Host Header

Validate host/domain assumptions where necessary.

Do not build password-reset or absolute URLs solely from untrusted Host headers.

---

# 158. Open Redirect

Validate redirect destinations.

---

# 159. Password Reset

Use:

```text
short-lived token
single use
secure transport
rate limits
generic responses
```

according to the product's security design.

---

# 160. Account Enumeration

Login/reset endpoints can unintentionally reveal whether an account exists.

Use consistent externally visible behavior where reducing enumeration matters.

---

# 161. Audit Security-Critical Events

Track:

```text
login
password change
role change
API key change
```

without storing secrets.

---

# 162. Dependency Updates

Audit:

```text
runtime
framework
database driver
security libraries
```

regularly.

---

# 163. Package Locking

Keep reproducible dependency state for applications.

---

# 164. API Documentation

Use OpenAPI or another structured contract where appropriate.

---

# 165. Contract Testing

Validate:

```text
request schema
response schema
status codes
error shape
```

---

# 166. Consumer Compatibility

When changing an API, consider:

```text
existing mobile clients
old web clients
third-party consumers
scheduled jobs
```

---

# 167. Test Architecture

Layers:

```text
unit
integration
contract
end-to-end
load
security
```

---

# 168. Unit Tests

Test:

```text
domain rules
validators
services
authorization policies
pagination logic
```

---

# 169. Repository Tests

Use a real or realistic database test environment.

Mocks alone cannot prove query correctness.

---

# 170. API Integration Tests

Test:

```text
HTTP
routing
validation
auth
DB
response
```

---

# 171. Authorization Matrix Tests

Test:

```text
owner
member
viewer
other tenant
unauthenticated
```

for each protected operation.

---

# 172. Concurrency Tests

Simulate:

```text
two updates same version
```

Expected:

```text
one succeeds
one conflicts
```

when optimistic concurrency is chosen.

---

# 173. Idempotency Tests

Send:

```text
same operation
same key
```

multiple times.

Verify one business effect.

---

# 174. Retry Tests

Simulate:

```text
dependency timeout
dependency 503
```

and verify retry policy.

---

# 175. Transaction Tests

Force:

```text
mid-transaction failure
```

and verify rollback/consistency.

---

# 176. Migration Tests

Test migration on:

```text
empty DB
small DB
representative large DB
```

when feasible.

---

# 177. Load Tests

Measure:

```text
requests/sec
p50/p95/p99
CPU
memory
DB connections
DB latency
```

---

# 178. Capacity Test

Increase:

```text
concurrency
```

until:

```text
SLO violation
```

Record the capacity frontier.

---

# 179. Security Tests

Include:

```text
SQL injection
NoSQL injection
authorization bypass
path traversal
SSRF
XSS payload propagation
mass assignment
rate-limit bypass
oversized request
```

---

# 180. Fuzz Tests

Good candidates:

```text
query parser
JSON schema
pagination cursor
filters
path parameters
```

---

# 181. Failure Injection

Test:

```text
DB unavailable
DB slow
Redis unavailable
HTTP dependency slow
queue unavailable
filesystem unavailable
```

---

# 182. Observability Tests

Assert important events include:

```text
request ID
route
status
duration
```

and exclude sensitive data.

---

# 183. Graceful Shutdown Test

Start server.

Send:

```text
SIGTERM
```

Verify:

```text
no new work
drain policy
dependency close
bounded exit
```

---

# 184. Performance Profiling

Profile:

```text
CPU
heap
event loop
database
network
serialization
```

---

# 185. N+1 Detection

Instrument query counts.

Create a regression test for:

```text
list endpoint
```

---

# 186. Query Budget

Define rough budgets for hot endpoints:

```text
max DB queries
max downstream calls
```

This catches accidental amplification.

---

# 187. Latency Budget

Break request latency into:

```text
application
DB
downstream
serialization
```

---

# 188. Memory Budget

Watch:

```text
heap
buffers
connection state
request bodies
```

---

# 189. Request Body Parsing Cost

Large requests can consume memory before validation.

Set body limits early.

---

# 190. Serialization Cost

Large response serialization can block the event loop.

Paginate and stream where appropriate.

---

# 191. Caching Trade-Off

Measure:

```text
hit rate
latency
memory
invalidation
```

before deciding.

---

# 192. Database Read Caching

Do not cache mutable/permission-sensitive data without including correct authorization dimensions.

---

# 193. HTTP Caching

Use protocol caching semantics where clients/CDNs benefit.

---

# 194. ETag

Useful for conditional requests when representation identity can be defined.

---

# 195. Compression

Compression can reduce network bytes at CPU cost.

Measure.

---

# 196. API Payload Design

Avoid sending:

```text
unused huge object graphs
```

by default.

Use projections/fields only when justified.

---

# 197. API Versioning

Avoid breaking consumers accidentally.

Version or evolve compatibly.

---

# 198. Deprecation Metrics

Measure usage before removal.

---

# 199. Rollout

Use:

```text
canary
feature flag
gradual deployment
```

where risk justifies it.

---

# 200. Rollback

Ensure code/schema compatibility makes rollback possible where required.

---

# 201. Production Incident — DB Saturation

Symptoms:

```text
DB CPU 95%
API p99 high
```

Investigate:

```text
query count
slow queries
N+1
connection pool
```

---

# 202. Production Incident — Duplicate Creation

Symptoms:

```text
same task twice
```

Investigate:

```text
client retry
proxy retry
missing idempotency
```

---

# 203. Production Incident — Cross-Tenant Leak

Investigate:

```text
auth context
query scope
cache key
response mapping
```

Immediate priority:

```text
contain
revoke exposure
audit affected data
```

---

# 204. Production Incident — Memory Growth

Investigate:

```text
cache
request aggregation
listeners
promises
buffers
```

---

# 205. Production Incident — CPU Spike

Investigate:

```text
traffic
regex
JSON
serialization
algorithm
dependency
```

---

# 206. Production Incident — Queue Backlog

Investigate:

```text
producer rate
consumer rate
DB
downstream API
worker concurrency
```

---

# 207. Production Incident — Runtime Upgrade

Compare:

```text
same workload
old/new runtime
profiles
dependencies
```

---

# 208. Production Incident — API Contract Break

Use:

```text
consumer telemetry
version diff
contract tests
rollback/compatibility layer
```

---

# 209. Production Incident — Security Vulnerability

Follow:

```text
contain
assess exposure
patch
rotate secrets
verify
monitor
```

---

# 210. Production Incident — Slow External API

Do not blindly increase retries.

Consider:

```text
deadline
circuit breaker
fallback
queue
load shedding
```

---

# 211. Production Incident — DB Connection Exhaustion

Check:

```text
pool
transaction duration
connection release
traffic
query latency
```

---

# 212. Production Incident — Stale Authorization

A cache stores authorization-related output.

User permissions change.

Old access remains.

Do not cache security decisions without an explicit freshness/invalidation strategy.

---

# 213. Production Incident — Mass Assignment

Client sends:

```json
{
  "title": "x",
  "role": "admin"
}
```

Endpoint blindly maps input.

Protect through explicit writable-field schemas.

---

# 214. Production Incident — Open Redirect

A `next` parameter is reflected into redirects.

Validate allowed destinations.

---

# 215. Production Incident — SSRF

User submits a URL.

Server fetches it.

Protect:

```text
allowlist
network egress
redirect policy
```

---

# 216. Production Incident — Health Check False Positive

Health endpoint returns 200 because:

```text
process alive
```

but database unavailable.

Separate:

```text
liveness
readiness
```

---

# 217. Production Incident — Shutdown Corrupts Work

Process terminates while:

```text
transaction
```

or:

```text
queue operation
```

is incomplete.

Add bounded graceful shutdown.

---

# 218. Implementation Progression

## Stage 1 — Guided

Build:

```text
GET /health
GET /tasks
POST /tasks
```

## Stage 2 — Partially Guided

Add:

```text
validation
auth
DB
error contract
```

## Stage 3 — No Reference

Add:

```text
pagination
authorization
concurrency
```

## Stage 4 — Edge-Case Hardened

Add:

```text
idempotency
rate limits
timeouts
transactions
shutdown
```

## Stage 5 — Production Grade

Add:

```text
observability
security
load tests
migration strategy
API docs
versioning
deployment
```

---

# 219. Track A — Core Theory

Study:

```text
HTTP semantics
REST
routing
validation
authentication
authorization
database transactions
concurrency
idempotency
pagination
caching
rate limits
timeouts
backpressure
observability
security
API compatibility
```

---

# 220. Track B — Implementation

Build:

```text
server
routes
controllers
services
domain
repositories
database
migrations
authentication
authorization
validators
pagination
error mapper
rate limiter
idempotency store
health endpoints
shutdown manager
telemetry
tests
load harness
```

---

# 221. Track C — Interview / Reasoning

Defend:

```text
Why REST?
Why this resource model?
Why these status codes?
Why this validation boundary?
Why centralized authorization?
Why transactions?
Why optimistic concurrency?
Why idempotency?
Why cursor pagination?
Why no global state?
Why these timeouts?
Why this cache?
Why this database index?
Why this deployment model?
```

---

# 222. Mastery Gate

```text
Understand
→ Explain
→ Predict
→ Implement
→ Debug
→ Apply
→ Compare
→ Defend
```

---

# 223. Mastery Exercise — Build Core API

Implement:

```text
GET /health
POST /tasks
GET /tasks
GET /tasks/:id
PATCH /tasks/:id
DELETE /tasks/:id
```

---

# 224. Mastery Exercise — Authentication

Implement a secure authentication flow appropriate to the chosen client model.

Test:

```text
missing credentials
invalid credentials
expired credentials
```

---

# 225. Mastery Exercise — Authorization

Create:

```text
owner
member
viewer
other tenant
```

and test every operation.

---

# 226. Mastery Exercise — Pagination

Support:

```text
limit
cursor
sort
filter
```

with:

```text
maximum page size
stable ordering
```

---

# 227. Mastery Exercise — Concurrency

Implement version-based updates.

Demonstrate:

```text
concurrent update
→ one success
→ one conflict
```

---

# 228. Mastery Exercise — Idempotency

Implement:

```text
Idempotency-Key
```

for one mutation.

Test repeated submission.

---

# 229. Mastery Exercise — Transaction

Implement:

```text
business mutation
+
audit record
```

inside one transaction.

Force a failure and verify rollback.

---

# 230. Mastery Exercise — Outbox

Implement:

```text
business transaction
+
outbox event
```

then publish asynchronously.

---

# 231. Mastery Exercise — Rate Limiting

Protect:

```text
POST /auth/login
```

and one expensive endpoint.

---

# 232. Mastery Exercise — Health

Implement separate:

```text
liveness
readiness
```

and test dependency failure.

---

# 233. Mastery Exercise — Graceful Shutdown

Send:

```text
SIGTERM
```

during an active request.

Verify the shutdown policy.

---

# 234. Mastery Exercise — Load Testing

Find the concurrency level where:

```text
p99
```

violates your target.

Identify the bottleneck.

---

# 235. Mastery Exercise — Security

Demonstrate and fix:

```text
SQL injection
mass assignment
BOLA/IDOR
SSRF
rate-limit bypass
```

---

# 236. Mastery Exercise — API Evolution

Change a response field.

Maintain compatibility through:

```text
versioning
dual fields
or migration
```

then remove old behavior after measuring consumers.

---

# 237. Mastery Exercise — Deployment

Create:

```text
container
environment config
migration step
health checks
graceful shutdown
```

and document the deployment sequence.

---

# 238. Code Review Exercise

Review:

```js
app.post("/tasks", async (req, res) => {
  const task = await db.tasks.create({
    data: req.body
  });

  res.json(task);
});
```

Identify:

```text
validation
mass assignment
authorization
tenant scoping
response projection
error mapping
idempotency
transaction requirements
```

---

# 239. Code Review Exercise

Review:

```js
app.get("/tasks/:id", async (req, res) => {
  const task =
    await db.tasks.findUnique({
      where: { id: req.params.id }
    });

  res.json(task);
});
```

Identify:

```text
object-level authorization
tenant isolation
404 handling
input validation
data exposure
```

---

# 240. Code Review Exercise

Review:

```js
app.get("/tasks", async (req, res) => {
  const tasks =
    await db.tasks.findMany();

  res.json(tasks);
});
```

Identify:

```text
unbounded response
pagination
authorization
projection
query performance
sorting
```

---

# 241. Code Review Exercise

Review:

```js
await db.transaction(async tx => {
  await tx.orders.create(...);
  await fetch(paymentUrl);
  await tx.audit.create(...);
});
```

Explain why the network call inside a DB transaction may be dangerous.

---

# 242. Interview Questions — Senior

1. How would you structure a Node REST API?
2. Where should validation occur?
3. Where should authorization occur?
4. How do you prevent IDOR/BOLA?
5. How do you implement pagination?
6. How do you handle concurrent updates?
7. How do you make writes idempotent?
8. How do you design transactions?
9. How do you handle downstream failures?
10. How do you implement graceful shutdown?

---

# 243. Interview Questions — Principal

1. How would you design a multi-tenant Node API for millions of users?
2. How would you guarantee tenant isolation?
3. How would you design API compatibility across years?
4. How would you choose pagination semantics?
5. How would you design idempotency at scale?
6. How would you design an outbox architecture?
7. How would you prevent retry amplification?
8. How would you scale database access?
9. How would you design observability without high-cardinality explosion?
10. How would you balance latency, consistency, and cost?

---

# 244. Acceptance Criteria

```text
[ ] resource model documented
[ ] routes documented
[ ] status codes documented
[ ] error contract stable
[ ] request schemas validated
[ ] response schemas controlled
[ ] authentication implemented
[ ] authorization centralized
[ ] tenant isolation tested
[ ] pagination bounded
[ ] sorting whitelisted
[ ] query patterns indexed
[ ] N+1 checked
[ ] transactions defined
[ ] DB constraints used
[ ] optimistic concurrency defined
[ ] idempotency implemented where required
[ ] rate limits defined
[ ] timeouts defined
[ ] outbound retries bounded
[ ] health/readiness defined
[ ] graceful shutdown tested
[ ] structured logs
[ ] metrics
[ ] trace/correlation support
[ ] secrets protected
[ ] security tests
[ ] integration tests
[ ] load tests
[ ] migration strategy
[ ] API versioning/deprecation
[ ] deployment documented
```

---

# 245. Production Checklist

```text
[ ] secure configuration
[ ] DB migration process
[ ] backup/recovery assumptions
[ ] connection pool sizing
[ ] body size limits
[ ] pagination limits
[ ] rate limits
[ ] timeouts
[ ] retry policy
[ ] idempotency
[ ] authorization
[ ] audit logging
[ ] observability
[ ] error classification
[ ] graceful shutdown
[ ] readiness/liveness
[ ] load testing
[ ] security testing
[ ] compatibility testing
[ ] rollback plan
```

---

# 246. Dependency Graph

```text
Chapter 55 — Fetch / HTTP
        ↓
Chapter 58–63 — Node Runtime
        ↓
Chapter 79–82 — API / Database Architecture
        ↓
Chapter 83–85 — Observability / Reliability / Performance
        ↓
Chapter 86–89 — Testing / Debugging / Review
        ↓
Chapter 94–97 — Compatibility / Legacy / Wasm / Edge
        ↓
Chapter 98–101 — Judgment
        ↓
Chapter 102–104 — CLI / Browser / HTTP Client
        ↓
Chapter 105 — Node REST API
        ↓
Chapter 106 — Real-Time WebSocket
        ↓
Chapter 107 — Job Queue
        ↓
Chapter 108 — Cache System
```

---

# 247. Concept Connections

## Depends On

```text
HTTP
Node
async
database
security
reliability
observability
performance
testing
architecture
```

## Builds Toward

```text
Chapter 106 — Real-Time WebSocket
Chapter 107 — Job Queue
Chapter 108 — Cache System
Chapter 109 — Event-Driven App
Chapter 110 — Production JS Backend
```

## Revisited

```text
AbortSignal
streams
errors
modules
package management
API contracts
security
transactions
concurrency
caching
observability
```

---

# 248. Spaced Retrieval Schedule

### Day 0

```text
routes
resources
status codes
errors
```

### Day 1

```text
validation
auth
authorization
pagination
```

### Day 3

```text
transactions
concurrency
idempotency
```

### Day 7

```text
rate limits
timeouts
observability
security
```

### Day 14

```text
migrations
compatibility
shutdown
load testing
```

### Day 30

Rebuild the core API.

### Day 60

Rebuild concurrency/idempotency.

### Day 90

Defend the full API architecture.

---

# 249. Revision / Retrieval Record

```md
# Chapter 105 — Revision / Retrieval Record

## Review #
- Date:
- Duration:
- Status before:
- Status after:

## Retrieval
- REST/resource semantics [ ]
- routing [ ]
- validation [ ]
- errors [ ]
- authentication [ ]
- authorization [ ]
- multi-tenancy [ ]
- pagination [ ]
- query/index strategy [ ]
- transactions [ ]
- constraints [ ]
- optimistic concurrency [ ]
- idempotency [ ]
- retries [ ]
- rate limiting [ ]
- timeouts [ ]
- caching [ ]
- health/readiness [ ]
- graceful shutdown [ ]
- observability [ ]
- security [ ]
- testing [ ]
- load testing [ ]
- migrations [ ]
- versioning [ ]

## Build Evidence
- Repository:
- Commit:
- Database:
- Contract suite:
- Load-test result:
- Security test:
- Migration test:

## Gaps
-

## New Insights
-

## Follow-up
-
```

---

# Chapter 105 — Canonical References and Source Discipline

Primary references:

1. HTTP Semantics — IETF HTTP Working Group  
   https://httpwg.org/specs/

2. Fetch Standard  
   https://fetch.spec.whatwg.org/

3. Node.js Documentation  
   https://nodejs.org/docs/

4. ECMAScript Language Specification  
   https://tc39.es/ecma262/

5. OWASP API Security  
   https://owasp.org/www-project-api-security/

6. OWASP Cheat Sheets  
   https://cheatsheetseries.owasp.org/

7. OpenAPI Specification  
   https://spec.openapis.org/oas/latest.html

Source discipline:

```text
HTTP semantics
→ HTTP specifications

Fetch behavior
→ Fetch Standard

Node runtime
→ Node documentation

language semantics
→ ECMAScript

security
→ OWASP + threat model

database behavior
→ chosen database documentation

performance
→ benchmark + profile + telemetry

reliability
→ load test + production evidence
```

---

# 250. Completion Snapshot

```text
Part XX — Projects

Chapter 105 — Production Node.js REST API
[ ] Not Started

Track A — Core Theory
[ ] REST/resource semantics
[ ] HTTP methods
[ ] status codes
[ ] error contract
[ ] validation
[ ] authentication
[ ] authorization
[ ] multi-tenancy
[ ] pagination
[ ] sorting/filtering
[ ] SQL safety
[ ] transactions
[ ] DB constraints
[ ] concurrency
[ ] idempotency
[ ] retries
[ ] rate limiting
[ ] timeouts
[ ] caching
[ ] outbox
[ ] background jobs
[ ] health/readiness
[ ] graceful shutdown
[ ] observability
[ ] security
[ ] compatibility
[ ] migrations
[ ] testing
[ ] load testing

Track B — Implementation
[ ] server
[ ] routing
[ ] controllers
[ ] services
[ ] domain
[ ] repositories
[ ] database
[ ] migrations
[ ] auth
[ ] authorization
[ ] validators
[ ] pagination
[ ] error mapper
[ ] idempotency store
[ ] rate limiter
[ ] health
[ ] shutdown
[ ] logs
[ ] metrics
[ ] tracing
[ ] contract tests
[ ] integration tests
[ ] load tests
[ ] security tests
[ ] deployment

Track C — Interview / Reasoning
[ ] Explain API architecture
[ ] Explain resource modeling
[ ] Explain status codes
[ ] Explain validation
[ ] Explain authentication
[ ] Explain authorization
[ ] Explain tenant isolation
[ ] Explain pagination
[ ] Explain transactions
[ ] Explain optimistic concurrency
[ ] Explain idempotency
[ ] Explain retries
[ ] Explain rate limits
[ ] Explain observability
[ ] Explain migrations
[ ] Defend architecture
[ ] Defend scaling strategy

Mastery Gate
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

# 251. Completion Criteria

Do not mark this project mastered because the endpoints return JSON.

You are ready to move forward when you can independently:

1. Model API resources.
2. Define HTTP semantics.
3. Design stable request/response contracts.
4. Validate all untrusted inputs.
5. Authenticate callers.
6. Authorize resource operations.
7. Enforce tenant isolation.
8. Prevent IDOR/BOLA.
9. Implement bounded pagination.
10. Design safe filtering/sorting.
11. Protect SQL/NoSQL boundaries.
12. Use DB constraints.
13. Design transaction boundaries.
14. Handle concurrent writes.
15. Implement idempotent mutations.
16. Bound retries and rate limits.
17. Propagate deadlines/cancellation where appropriate.
18. Handle downstream failures.
19. Implement health/readiness.
20. Implement graceful shutdown.
21. Build structured observability.
22. Protect sensitive information.
23. Test authorization matrices.
24. Test concurrency.
25. Test migrations.
26. Load test critical paths.
27. Identify bottlenecks.
28. Design API evolution.
29. Deploy safely.
30. Defend the entire architecture at principal level.

---

# Final Mental Model

```text
HTTP Request
     ↓
Security Boundary
 ├── authentication
 ├── authorization
 └── validation
     ↓
Controller
     ↓
Application Service
     ↓
Domain Rules
     ↓
Transaction / Repository
     ↓
Database
     ↓
Response Model
     ↓
HTTP Response
```

Cross-cutting:

```text
timeouts
cancellation
retries
rate limits
observability
configuration
compatibility
```

Reliability:

```text
request
→ deadline
→ bounded resources
→ transactional state
→ idempotent side effects
→ observable result
```

The strongest REST API is not the one with the most routes.

It is the one where:

```text
contracts are explicit
authorization is correct
data boundaries are controlled
concurrency is intentional
failures are classified
transactions are small
resources are bounded
observability is useful
security is designed
compatibility is managed
```

> **Mastery reminder:** A production API must remain correct when requests overlap, clients retry, dependencies fail, deployments roll, data grows, users are adversarial, and multiple versions coexist.