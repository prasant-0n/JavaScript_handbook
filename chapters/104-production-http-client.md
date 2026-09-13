# Chapter 104 — Production HTTP Client

> **JavaScript Mastery — Part XX: Projects**
>
> **Project:** Build a production-grade HTTP client for browser and Node.js environments, with explicit transport, timeout, cancellation, retry, serialization, authentication, observability, and failure semantics.
>
> **Role perspective:** Principal JavaScript Engineer · API Architect · Node.js Engineer · Browser-Platform Engineer · Reliability Engineer · Security Engineer · Performance Engineer · Library Author
>
> **Status:** `[ ] Not Started`
>
> **Verification date:** 2026-09-11
>
> **Core rule:** **An HTTP client is not a wrapper around `fetch()`. It is a policy engine that translates application intent into network behavior while controlling retries, timeouts, cancellation, data encoding, security, observability, and resource usage.**

---

# 1. Project Mission

Build an HTTP client that can safely support:

```text
GET
POST
PUT
PATCH
DELETE
```

with:

```text
base URL
headers
query parameters
request body
JSON
timeouts
AbortSignal
retry policy
idempotency
authentication
HTTP error mapping
response decoding
structured errors
logging
metrics hooks
tracing hooks
rate limits
concurrency control
tests
```

Recommended public API:

```js
const client = createHttpClient({
  baseUrl: "https://api.example.com"
});

const user =
  await client.get("/users/42");
```

Advanced APIs:

```js
client.request(...)
client.get(...)
client.post(...)
client.put(...)
client.patch(...)
client.delete(...)
```

---

# 2. Learning Objectives

By completing this project, you should be able to:

- Design a production HTTP client.
- Separate request policy from transport.
- Construct URLs safely.
- Encode query parameters.
- Serialize request bodies.
- Parse response bodies.
- Model HTTP failures.
- Distinguish network failures from HTTP failures.
- Distinguish cancellation from timeout.
- Implement request deadlines.
- Propagate AbortSignal.
- Implement bounded retries.
- Classify retryable errors.
- Handle idempotency.
- Avoid duplicate side effects.
- Implement exponential backoff.
- Add jitter.
- Respect `Retry-After`.
- Implement rate-limit handling.
- Implement per-request configuration.
- Implement client defaults.
- Merge headers predictably.
- Handle authentication.
- Prevent secret leakage.
- Handle cookies/credentials correctly by environment.
- Handle redirects according to platform semantics.
- Validate status codes.
- Parse content types.
- Limit response sizes where needed.
- Stream large responses when appropriate.
- Support request/response interceptors or middleware carefully.
- Implement observability hooks.
- Track latency.
- Track retries.
- Track status classes.
- Track cancellation.
- Track network errors.
- Test deterministic retry behavior.
- Test timeout behavior.
- Test cancellation.
- Test malformed responses.
- Test partial failures.
- Benchmark the client.
- Handle high concurrency safely.
- Define browser/Node differences.
- Package the client as a reusable library.
- Document the public contract.
- Defend every major design decision at principal level.

---

# 3. Prerequisites

Recommended:

```text
Chapter 29 — Errors
Chapter 31–38 — Async / Promises / Cancellation / Streams
Chapter 41–44 — ECMAScript Specification
Chapter 45–48 — Memory / Engines
Chapter 49–57 — Browser / Web APIs / Networking / Security
Chapter 58–63 — Node.js
Chapter 64–70 — Modules / Tooling
Chapter 71–73 — Data Structures / Algorithms
Chapter 78–89 — Production / Testing / Debugging
Chapter 94 — Compatibility
Chapter 98–101 — Judgment
Chapter 102 — Production CLI
Chapter 103 — Vanilla Browser App
```

---

# 4. Architecture

Target:

```text
Application
    ↓
HTTP Client
    ↓
Policy Pipeline
    ↓
Transport
    ↓
Fetch
    ↓
Network
```

Detailed:

```text
Request
  ↓
URL builder
  ↓
header policy
  ↓
body serializer
  ↓
auth policy
  ↓
timeout/deadline
  ↓
retry policy
  ↓
transport
  ↓
HTTP response
  ↓
status classifier
  ↓
body decoder
  ↓
error mapper
  ↓
response
```

---

# 5. Separation of Responsibilities

`createHttpClient()` should not directly own every detail.

Recommended modules:

```text
src/
├── client.js
├── request.js
├── url.js
├── headers.js
├── body.js
├── response.js
├── errors.js
├── retry.js
├── timeout.js
├── auth.js
├── transport.js
├── rate-limit.js
├── observability.js
└── middleware.js
```

---

# 6. Transport Boundary

Define:

```js
async function transport(request, options) {
  return fetch(request, options);
}
```

The abstraction lets tests replace the transport.

---

# 7. Why Have a Transport Boundary?

It enables:

```text
unit tests
mock transport
browser/runtime differences
future alternate transport
instrumentation
```

without coupling all client logic to global `fetch`.

---

# 8. Public Request Model

Conceptual:

```js
{
  method: "GET",
  url: "/users/42",
  query: {},
  headers: {},
  body: undefined,
  signal: undefined,
  timeoutMs: 5000,
  retry: true
}
```

---

# 9. Internal Normalization

Normalize user input into one internal request shape before transport.

This reduces branching throughout the pipeline.

---

# 10. HTTP Method Semantics

Know:

```text
GET
HEAD
OPTIONS
POST
PUT
PATCH
DELETE
```

Do not assume application semantics are identical to method names.

---

# 11. GET

Normally used to retrieve representations.

Be careful with:

```text
large bodies
side effects
cache semantics
```

---

# 12. POST

Often used for actions/creation.

Retrying POST requires careful consideration of idempotency.

---

# 13. PUT

Often associated with replacement/idempotent update semantics.

Actual API contract matters.

---

# 14. PATCH

Often used for partial modification.

Retry safety depends on the specific operation.

---

# 15. DELETE

Often intended to be idempotent at the HTTP semantic level, but application behavior must still be validated.

---

# 16. URL Construction

Avoid:

```js
baseUrl + path + "?" + query;
```

Use:

```js
new URL(path, baseUrl);
```

Then:

```js
url.searchParams.set(...)
```

---

# 17. Relative URL Rules

Define how:

```text
/users
./users
../users
```

are resolved against the configured base URL.

Test trailing-slash combinations.

---

# 18. Query Parameters

Represent:

```js
{
  page: 2,
  search: "node js",
  active: true
}
```

as a defined URL encoding policy.

Do not silently serialize:

```text
objects
arrays
null
undefined
```

without documenting the behavior.

---

# 19. Query Arrays

Choose one:

```text
tag=a&tag=b
```

or:

```text
tag=a,b
```

or another server contract.

Do not invent format per endpoint.

---

# 20. Undefined Query Values

Define:

```text
omit
```

instead of accidentally sending:

```text
undefined
```

unless the API explicitly requires it.

---

# 21. Null Query Values

Define whether:

```text
null
```

means:

```text
omit
empty
"null"
```

This is an API design decision.

---

# 22. Header Merge

Define precedence:

```text
client defaults
↓
request headers
```

Do not accidentally overwrite security-critical headers in the wrong order.

---

# 23. Header Case

HTTP header names are case-insensitive.

Use a predictable internal representation.

---

# 24. Content-Type

For JSON:

```http
Content-Type: application/json
```

Do not set it automatically when the body is already a stream or another content type.

---

# 25. Accept

Support:

```http
Accept: application/json
```

when the client expects a JSON response.

---

# 26. Body Serialization

Common body classes:

```text
string
URLSearchParams
FormData
Blob
ArrayBuffer
TypedArray
ReadableStream
JSON-compatible object
```

Choose explicit serialization rules.

---

# 27. JSON Body

Convenient API:

```js
client.post("/users", {
  json: {
    name: "Ada"
  }
});
```

Internally:

```text
JSON.stringify
+
Content-Type
```

---

# 28. JSON Serialization Failure

`JSON.stringify()` can fail.

Example:

```js
JSON.stringify(
  { value: 1n }
);
```

The client should surface a meaningful serialization error.

---

# 29. Circular JSON

Circular objects cannot be represented by ordinary JSON serialization.

Do not silently mutate them to “make JSON work”.

---

# 30. Response Content-Type

Inspect:

```http
Content-Type
```

before choosing decoding.

---

# 31. JSON Response

Use:

```js
await response.json();
```

when appropriate.

Do not call JSON decoding on an HTML error page without considering the contract.

---

# 32. Text Response

Support:

```js
await response.text();
```

when the response is textual.

---

# 33. Binary Response

Support:

```js
await response.arrayBuffer();
```

or stream processing when payload size requires it.

---

# 34. Response Size Limits

For untrusted endpoints, define maximum acceptable response size where the environment supports enforcing it safely.

This is a resource-exhaustion control.

---

# 35. Streaming Response

For very large content:

```text
response.body
```

can be processed incrementally.

Do not force every response into memory.

---

# 36. Status Classification

Recommended conceptual categories:

```text
2xx
→ success

3xx
→ redirect behavior

4xx
→ client/application/request problem

5xx
→ server/dependency problem
```

But individual statuses have distinct semantics.

---

# 37. HTTP Error Object

Design:

```js
class HttpError extends Error {
  constructor(message, {
    status,
    response,
    requestId,
    body
  }) {
    super(message);
    this.name = "HttpError";
    this.status = status;
    this.response = response;
    this.requestId = requestId;
    this.body = body;
  }
}
```

Avoid making errors impossible to serialize safely.

---

# 38. Network Error

Different from:

```text
HTTP 500
```

A network error means a usable HTTP response was not obtained.

---

# 39. Timeout Error

Timeout is a policy event:

```text
the client exceeded its deadline
```

The server may still be processing.

---

# 40. Cancellation Error

Cancellation may mean:

```text
the caller no longer wants this operation
```

Do not automatically categorize it as server failure.

---

# 41. Error Taxonomy

Recommended:

```text
RequestBuildError
SerializationError
TimeoutError
AbortError
NetworkError
HttpError
ResponseDecodeError
RetryExhaustedError
```

Use stable error codes.

---

# 42. Error Preservation

When wrapping errors:

```js
new Error("request failed", {
  cause: original
});
```

preserve the original cause where supported.

---

# 43. Retry Is a Policy

Do not implement:

```text
catch → retry
```

without classifying the failure.

---

# 44. Retryable Conditions

Depending on the contract, retry may make sense for:

```text
network interruption
408
429
selected 5xx
```

But not necessarily:

```text
400
401
403
404
422
```

Do not use a simplistic status list as universal truth.

---

# 45. Retry Method Safety

Safe retries depend on:

```text
HTTP method
API semantics
idempotency
server behavior
```

---

# 46. Idempotency Key

For suitable write APIs:

```http
Idempotency-Key: <unique-value>
```

allows the server to deduplicate repeated attempts when the server implements that contract.

---

# 47. Client Cannot Invent Idempotency

Adding an idempotency header does not make an API idempotent unless the server recognizes and enforces the contract.

---

# 48. Exponential Backoff

Concept:

```text
delay
× 2
× 2
× 2
```

Use bounded delay.

---

# 49. Jitter

Without jitter, clients can retry together.

Jitter spreads attempts over time.

Common approaches:

```text
full jitter
equal jitter
decorrelated jitter
```

Choose and document one.

---

# 50. Retry Budget

Define:

```text
max attempts
max elapsed time
```

Prefer deadlines over arbitrary endless retries.

---

# 51. Retry-After

For 429/selected 503 responses, inspect:

```http
Retry-After
```

when the API/server uses it.

Validate and bound it.

---

# 52. Retry Amplification

Client:

```text
1 original
+
3 retries
=
up to 4 attempts
```

Across layers:

```text
browser
× SDK
× proxy
× server
```

attempts can multiply.

Define ownership.

---

# 53. Retry and Timeout Interaction

Per-attempt timeout:

```text
attempt ≤ 2s
```

overall deadline:

```text
request ≤ 5s
```

Do not accidentally permit:

```text
3 retries × 5s
```

when the API contract says 5 seconds total.

---

# 54. Deadline Propagation

Carry a deadline through the client.

Concept:

```js
deadline =
  start + overallTimeout;
```

Every attempt receives remaining time.

---

# 55. AbortSignal

Allow:

```js
client.get("/search", {
  signal
});
```

The signal should reach the transport.

---

# 56. Timeout + AbortSignal

Combine external cancellation with an internal timeout.

Possible model:

```text
caller abort
OR
deadline exceeded
```

whichever happens first.

---

# 57. Do Not Swallow Caller Cancellation

If the caller deliberately aborts, preserve cancellation semantics.

Do not retry an operation simply because it ended by abort.

---

# 58. Race Between Completion and Abort

If completion wins before abort, the caller may receive success.

Design and test near-boundary races.

---

# 59. Response Body Cancellation

If the caller stops reading a large body, release/cancel it according to the stream API.

---

# 60. Redirects

Understand the runtime's redirect behavior.

Avoid turning redirects into an unreviewed security policy.

---

# 61. Redirect + Credentials

Cross-origin redirects can have security implications.

Do not blindly forward sensitive headers across host boundaries.

---

# 62. Authentication

Possible mechanisms:

```text
Bearer
Basic
cookies
mTLS
custom headers
```

The client should expose a clear authentication policy.

---

# 63. Token Refresh

A common failure:

```text
10 requests
→ token expires
→ 10 refresh calls
```

Implement single-flight refresh where appropriate.

---

# 64. Single-Flight Token Refresh

Concept:

```text
first request
→ starts refresh

other requests
→ await same refresh Promise
```

Only one refresh operation is active.

---

# 65. Refresh Failure

All waiting callers should receive a coherent result.

Avoid infinite refresh loops.

---

# 66. Auth Retry

Only retry a request after refreshing credentials if:

```text
status semantics indicate expired/invalid token
```

and the operation is still safe to replay.

---

# 67. Credential Storage

Browser and Node environments differ.

Do not automatically use browser storage for sensitive credentials.

---

# 68. Cookies

Browser fetch credential behavior depends on:

```text
credentials option
cookie policy
origin
server headers
```

Do not assume cookies are always sent.

---

# 69. CORS

CORS is a browser access-control mechanism.

It is not an authentication system.

---

# 70. CSRF

Cookie-authenticated browser clients need an appropriate CSRF strategy when relevant.

The HTTP client should not pretend transport alone solves CSRF.

---

# 71. SSRF

Server-side clients that accept arbitrary URLs can become SSRF primitives.

Do not make:

```js
client.get(userProvidedUrl)
```

safe by URL parsing alone.

Use allowlists/network policy.

---

# 72. Host Allowlist

For server-side clients, optionally restrict destinations:

```text
api.example.com
billing.example.com
```

Avoid arbitrary external targets where the product does not require them.

---

# 73. URL Validation

Use:

```js
new URL(...)
```

for parsing.

Then apply:

```text
scheme
hostname
port
path
```

policy.

---

# 74. DNS Rebinding

URL validation followed by DNS/network activity can have time-of-check/time-of-use concerns in SSRF-sensitive systems.

Treat network egress policy as a defense layer.

---

# 75. TLS

HTTPS protects transport properties.

It does not protect against:

```text
wrong authorization
wrong endpoint
application bugs
```

---

# 76. Certificate Errors

Do not disable TLS verification as a general “fix”.

Development-only exceptions must be deliberate and isolated.

---

# 77. Proxy Support

Node deployments may operate behind:

```text
forward proxy
corporate proxy
service mesh
```

Define how proxy settings are handled.

Do not assume browser and Node proxy semantics are identical.

---

# 78. HTTP Agent / Connection Reuse

Long-lived Node clients can benefit from appropriate connection reuse.

Choose agent/pooling behavior intentionally.

---

# 79. Connection Pooling

Too little:

```text
queueing
```

Too much:

```text
upstream overload
```

Tune against actual downstream limits.

---

# 80. Keep-Alive

Connection reuse can reduce:

```text
TCP/TLS setup
```

cost.

Still respect:

```text
connection limits
server behavior
idle timeouts
```

---

# 81. Browser Connection Limits

Browser networking is largely managed by the browser.

Do not attempt to manually reproduce browser pooling logic.

---

# 82. Node vs Browser Transport

Browser:

```text
CORS
cookies
service workers
browser cache
connection management
```

Node:

```text
filesystem/process
custom agents
server-side networking
```

Keep environment-specific policy explicit.

---

# 83. Request Hooks

Possible lifecycle:

```text
beforeRequest
afterResponse
onError
onRetry
```

These are powerful but can make control flow harder to reason about.

---

# 84. Middleware Risk

A middleware that silently modifies:

```text
headers
URL
body
retry
```

can create hidden coupling.

Document order and ownership.

---

# 85. Middleware Ordering

Example:

```text
auth
→ retry
→ telemetry
```

versus:

```text
retry
→ auth
→ telemetry
```

can produce different semantics.

Define the execution pipeline.

---

# 86. Observability

Capture:

```text
method
route template
status
latency
retry count
timeout
cancellation
request size
response size
```

Avoid raw secrets.

---

# 87. High-Cardinality Telemetry

Do not tag metrics with:

```text
full URL
user ID
request ID
token
```

without strong reason.

---

# 88. Trace Context

If the surrounding environment supports distributed tracing, propagate context according to its standard/runtime contract.

Do not invent incompatible headers.

---

# 89. Request IDs

Support response correlation identifiers where useful.

Do not expose internal secrets through request IDs.

---

# 90. Metrics

Useful metrics:

```text
http_client_requests_total
http_client_request_duration
http_client_retries_total
http_client_errors_total
http_client_timeouts_total
http_client_cancellations_total
```

Choose dimensions carefully.

---

# 91. Logging

Log events, not entire requests.

Example:

```js
logger.warn("http_retry", {
  method,
  route,
  attempt,
  reason
});
```

---

# 92. Privacy

Never automatically log:

```text
Authorization
Cookie
password
tokens
credit-card data
```

---

# 93. Redaction

Provide a central redaction mechanism for selected headers/fields.

---

# 94. Request/Response Logging

Full-body logging should be opt-in and bounded.

Never make it the default for production.

---

# 95. Response Validation

HTTP status success does not prove schema correctness.

Validate response shape when the application requires it.

---

# 96. Runtime Type Validation

If API responses are untrusted, use runtime validation/schema parsing at the client boundary.

Static types alone do not validate network data.

---

# 97. Schema Versioning

Support:

```text
API version
schema version
```

where the server contract requires it.

---

# 98. Content Negotiation

Use:

```text
Accept
Content-Type
```

intentionally.

---

# 99. ETag / Conditional Requests

Where API semantics support it:

```http
If-None-Match
```

can reduce transfer.

Cache validation has complexity.

---

# 100. HTTP Cache

Do not build a custom cache before understanding:

```text
HTTP caching
freshness
validation
Vary
Cache-Control
ETag
```

---

# 101. Client Cache

A higher-level application cache is different from HTTP caching.

Do not collapse them into one model.

---

# 102. Deduplication

Two components may request:

```text
GET /users/42
```

at the same time.

A client can optionally coalesce identical requests.

This is a semantic choice, not a universal optimization.

---

# 103. Request Coalescing Risk

Coalescing can be wrong when requests differ by:

```text
authorization
headers
tenant
cache policy
```

Include all relevant identity dimensions in the key.

---

# 104. Rate Limiting

Client-side rate limiting can protect downstream systems.

But the server remains the authoritative policy.

---

# 105. Token Bucket Concept

A token-bucket limiter controls:

```text
average rate
burst capacity
```

Tune to API limits.

---

# 106. Concurrency Limiting

Rate and concurrency are different.

You may need:

```text
requests/sec
max in-flight
```

simultaneously.

---

# 107. Queueing Requests

If the client queues requests internally, define:

```text
max queue
timeout while waiting
cancellation
priority
```

---

# 108. Unbounded Client Queue

Bad:

```text
10,000 requests
→ memory queue
```

without a bound.

Backpressure must reach the caller.

---

# 109. Request Size Limits

Define maximum body size where appropriate.

This protects against accidental or malicious oversized requests.

---

# 110. Response Size Limits

Likewise for responses.

Use streaming for large data.

---

# 111. Compression

Let transport negotiation handle suitable HTTP compression.

Do not compress already-compressed binary data unnecessarily.

---

# 112. Serialization Cost

For large payloads:

```text
serialize
→ network
→ parse
```

can dominate runtime.

Measure before changing format.

---

# 113. JSON Cost

JSON is convenient.

It is not free.

Large JSON means:

```text
CPU
memory
allocation
```

---

# 114. Binary Data

Use:

```text
ArrayBuffer
Blob
ReadableStream
```

where API contracts require binary.

---

# 115. Large Uploads

Do not read a giant file fully into memory before sending when streaming is viable.

---

# 116. Large Downloads

Prefer streaming when the application can process incrementally.

---

# 117. Progress Events

Fetch does not provide a generic universal upload/download progress API identical to every older XHR pattern.

Design progress features around the actual platform capabilities.

---

# 118. Retry and Streaming Request Bodies

A streamed body may not be replayable.

Do not blindly retry non-replayable bodies.

---

# 119. Replayability

Before retrying, know whether you can reproduce:

```text
method
headers
body
authentication state
```

---

# 120. Streaming + Retry

If the body cannot be replayed:

```text
no automatic retry
```

may be the safest policy.

---

# 121. Partial Upload Failure

A network interruption can occur after the server received part/all of the data.

Retry safety depends on server semantics.

---

# 122. Expect: 100-continue

Advanced Node/server scenarios may use request-expectation mechanisms.

Only add support when justified by actual HTTP infrastructure requirements.

---

# 123. HTTP/2

HTTP/2 changes transport characteristics:

```text
multiplexing
headers
connection behavior
```

but API-level retry/idempotency semantics remain important.

---

# 124. HTTP/3

HTTP/3 changes transport mechanisms again.

The HTTP client should avoid exposing transport-specific assumptions unnecessarily.

---

# 125. Connection Failure

Treat:

```text
DNS
TCP
TLS
socket
protocol
```

failures as transport-layer failures.

Preserve enough cause information for debugging.

---

# 126. DNS Failures

Useful classification:

```text
name resolution
```

rather than generic:

```text
request failed
```

---

# 127. TLS Failures

Do not retry certificate/configuration failures blindly.

---

# 128. 429 Handling

Respect server rate-limiting semantics where possible.

Combine:

```text
Retry-After
client budget
jitter
```

---

# 129. 401 Handling

401 may mean:

```text
missing
invalid
expired
```

credentials.

Do not automatically refresh for every 401 without policy.

---

# 130. 403 Handling

403 generally means authorization denial.

Refreshing credentials may not help.

---

# 131. 404 Handling

Do not automatically retry unless the API's semantics justify it.

---

# 132. 409 Handling

Conflict may indicate:

```text
version conflict
duplicate
state conflict
```

Application-specific handling is usually required.

---

# 133. 422 Handling

Often represents semantically invalid input.

Retrying the same request unchanged is normally not useful.

---

# 134. 429 vs 503

Both can signal temporary capacity problems but have different semantics.

Use the server contract and status headers to choose retry behavior.

---

# 135. 500

A 500 response does not prove that retrying is safe.

Idempotency still matters.

---

# 136. 502 / 503 / 504

These can indicate gateway/service availability problems.

Retry policy must still consider:

```text
operation
deadline
capacity
```

---

# 137. Retry Storm Prevention

Use:

```text
bounded attempts
backoff
jitter
deadline
circuit breaker where appropriate
```

---

# 138. Circuit Breaker

States:

```text
closed
open
half-open
```

A breaker can prevent repeated calls to a failing dependency.

---

# 139. Circuit Breaker Cost

It introduces:

```text
state
timers
threshold tuning
operational behavior
```

Do not add one blindly.

---

# 140. Bulkhead

Use separate capacity pools for unrelated dependency groups where shared exhaustion is dangerous.

---

# 141. Fallback

A fallback should be semantically correct.

Do not convert:

```text
payment unavailable
```

into:

```text
payment successful
```

---

# 142. Stale-Data Fallback

For read operations, stale cache may be acceptable in some domains.

Define the freshness contract.

---

# 143. Deadline vs Timeout

Timeout can mean:

```text
no more than N time
```

Deadline means:

```text
absolute end-by time
```

Deadlines are often easier to propagate across layers.

---

# 144. Request Context

A request context may carry:

```text
deadline
trace ID
tenant
request ID
abort signal
```

Avoid placing this into hidden global state.

---

# 145. Tenant Isolation

A reusable HTTP client used across tenants must not accidentally share:

```text
Authorization
cookies
cache
headers
```

across tenant contexts.

---

# 146. Header Leakage

A mutable default headers object can create:

```text
tenant A token
→ client defaults
→ tenant B request
```

Use immutable/request-scoped header composition.

---

# 147. Mutable Client Configuration

Avoid:

```js
client.defaults.headers.Authorization = ...
```

as a per-request authentication mechanism.

Prefer request-scoped credentials.

---

# 148. Base URL Security

Do not let untrusted input override:

```text
base URL
```

without explicit policy.

---

# 149. SSRF Defense-in-Depth

For server clients:

```text
URL allowlist
DNS/network policy
egress firewall
redirect control
credential isolation
```

---

# 150. Browser Security

In browsers, the client is constrained by:

```text
same-origin
CORS
credentials policy
service workers
mixed-content rules
```

---

# 151. Node Security

Node clients have broader network capabilities.

That increases responsibility for:

```text
destination validation
secret handling
egress control
```

---

# 152. Test Transport

Create a fake transport:

```js
async function fakeTransport(request) {
  return scenarios.shift();
}
```

This enables deterministic tests.

---

# 153. Test Successful Request

Assert:

```text
method
URL
headers
body
result
```

---

# 154. Test HTTP Error

Simulate:

```text
404
```

Assert:

```text
HttpError
status
body
```

---

# 155. Test Network Error

Simulate:

```text
TypeError
```

or a transport-defined network exception.

Assert stable mapping.

---

# 156. Test Timeout

Use fake timers or a deterministic scheduler where possible.

Do not sleep in tests.

---

# 157. Test Cancellation

Abort the signal.

Assert:

```text
transport aborted
no retry
correct error category
```

---

# 158. Test Retry

Configure:

```text
attempt 1 → 503
attempt 2 → 200
```

Assert:

```text
two attempts
expected delay policy
final success
```

---

# 159. Test Retry Exhaustion

Simulate repeated failure.

Assert:

```text
attempt cap
final error
cause chain
```

---

# 160. Test Retry-After

Simulate:

```http
429
Retry-After: 2
```

Assert bounded scheduling.

---

# 161. Test Jitter Deterministically

Inject:

```js
random: () => 0.5
```

Then assert predictable delay.

Never test random backoff by waiting real time.

---

# 162. Test Deadline

Simulate:

```text
attempt consumes time
next retry has insufficient remaining budget
```

Assert no excessive retry.

---

# 163. Test Non-Retryable Error

```text
400
```

should not be retried by a default policy.

---

# 164. Test Idempotency

Assert write retries include the expected idempotency key only where the API contract supports it.

---

# 165. Test Non-Replayable Body

Simulate a one-shot stream.

Assert:

```text
no unsafe automatic retry
```

---

# 166. Test Token Refresh

Simulate:

```text
request → 401
refresh → success
request retry → 200
```

Assert only one refresh for concurrent requests.

---

# 167. Test Refresh Failure

Ensure:

```text
waiting requests
```

receive a consistent failure.

---

# 168. Test Request Coalescing

Two equivalent GETs:

```text
same key
```

should share one transport request only if coalescing is enabled and safe.

---

# 169. Test Tenant Isolation

Two requests with different auth contexts must not share mutable headers/cookies/cache entries incorrectly.

---

# 170. Test Response Validation

Simulate:

```json
{
  "id": 42
}
```

when:

```text
id must be string
```

Assert schema error.

---

# 171. Test Large Response

Generate a large fixture.

Measure:

```text
memory
time
```

and ensure streaming strategy works where required.

---

# 172. Test Redirect Policy

Verify:

```text
same-origin redirect
cross-origin redirect
credential handling
```

according to the chosen contract.

---

# 173. Test Malformed URL

Input:

```text
not a URL
```

must fail before network.

---

# 174. Test Invalid Method

Reject unsupported/custom values only if the client intentionally restricts them.

---

# 175. Test Header Override

Verify precedence:

```text
defaults
< request
```

without allowing accidental security bypass.

---

# 176. Test Body/Header Consistency

If JSON body is selected:

```text
Content-Type
```

should be coherent.

---

# 177. Test Output Determinism

Errors, metrics hooks, and logs should be predictable enough for test assertions.

---

# 178. Performance Test

Measure:

```text
100
1,000
10,000
```

requests in a controlled environment.

Separate:

```text
client overhead
network latency
server latency
```

where possible.

---

# 179. Client Overhead

Do not optimize before measuring:

```text
URL construction
header normalization
serialization
middleware
logging
```

---

# 180. High-Concurrency Test

Run:

```text
N concurrent requests
```

and verify:

```text
max in-flight
memory
latency
retry rate
```

---

# 181. Load Test

Use a test server with controlled behavior:

```text
fast
slow
error
retry-after
large payload
```

---

# 182. Fault Injection

Simulate:

```text
connection reset
timeout
429
500
503
malformed JSON
slow body
partial body
```

---

# 183. Observability Test

Verify telemetry contains:

```text
route
status
duration
attempt
```

but excludes:

```text
authorization
cookies
tokens
```

---

# 184. Browser Test Matrix

Test relevant:

```text
Chrome
Firefox
Safari
mobile browser
```

according to your support policy.

Do not claim universal browser behavior from one browser.

---

# 185. Node Test Matrix

Test supported:

```text
minimum Node
current Node
```

and important deployment environments.

---

# 186. Project Structure

Recommended:

```text
http-client/
├── package.json
├── README.md
├── LICENSE
├── src/
│   ├── client.js
│   ├── request.js
│   ├── url.js
│   ├── headers.js
│   ├── body.js
│   ├── response.js
│   ├── errors.js
│   ├── retry.js
│   ├── timeout.js
│   ├── auth.js
│   ├── rate-limit.js
│   ├── transport.js
│   └── observability.js
├── test/
│   ├── unit/
│   ├── integration/
│   └── fixtures/
├── docs/
│   ├── architecture.md
│   ├── retry-policy.md
│   ├── security.md
│   └── compatibility.md
└── examples/
```

---

# 187. Public API

Example:

```js
const api = createHttpClient({
  baseUrl,
  timeoutMs: 5_000,
  retry: {
    maxAttempts: 3
  }
});

const response =
  await api.request({
    method: "GET",
    path: "/users"
  });
```

---

# 188. Convenience Methods

```js
api.get(path, options)
api.post(path, options)
api.put(path, options)
api.patch(path, options)
api.delete(path, options)
```

These should be thin wrappers over the core request pipeline.

---

# 189. Return Shape

Decide whether methods return:

```text
parsed body
```

or:

```text
full response abstraction
```

or both through explicit APIs.

Do not make response semantics ambiguous.

---

# 190. Response Abstraction

Potential:

```js
{
  status,
  headers,
  data,
  requestId
}
```

Keep platform response details accessible when consumers need them.

---

# 191. Request Metadata

Optionally expose:

```text
duration
attempts
retries
```

through a structured metadata field.

---

# 192. Error Metadata

Useful:

```text
code
status
method
URL without secret query values
attempt
requestId
cause
```

---

# 193. Avoid URL Secrets in Errors

Query strings can contain sensitive tokens.

Redact or sanitize before logging/error display.

---

# 194. Middleware API

Potential:

```js
client.use(async (ctx, next) => {
  // before
  await next();
  // after
});
```

Document execution order.

---

# 195. Middleware Exception

A middleware that catches errors can accidentally:

```text
retry
swallow
transform
```

failures.

Define which transformations are allowed.

---

# 196. Retry Hook

Expose:

```text
onRetry(context)
```

for observability.

Do not allow arbitrary hook behavior to change retry correctness silently.

---

# 197. Logger Interface

Inject:

```js
logger.info(...)
logger.warn(...)
logger.error(...)
```

rather than hard-coding `console`.

---

# 198. Metrics Interface

Inject:

```text
counter
histogram
gauge
```

or a small adapter.

Avoid coupling the client to one monitoring vendor.

---

# 199. Clock Injection

Inject a clock for deterministic timeout/backoff tests.

---

# 200. Randomness Injection

Inject RNG for deterministic jitter tests.

---

# 201. Transport Injection

Inject:

```js
transport(request)
```

to test every failure path without real networking.

---

# 202. Time Model

Distinguish:

```text
request timeout
overall deadline
queue wait
connection time
TTFB
body download
```

Measure only what your contract actually needs.

---

# 203. TTFB

Time to first byte can help distinguish:

```text
server processing
```

from:

```text
download
```

latency.

---

# 204. Response Download Time

For large responses:

```text
TTFB
+
body transfer
```

can differ significantly.

---

# 205. Per-Attempt Metrics

Track:

```text
attempt 1
attempt 2
attempt 3
```

without treating every retry as a new logical user request.

---

# 206. Logical Request vs Physical Attempt

One logical request can produce:

```text
multiple network attempts
```

Your metrics should preserve that distinction.

---

# 207. Error Rate

Choose whether your primary metric counts:

```text
logical request failures
```

or:

```text
physical attempt failures
```

They answer different questions.

---

# 208. Retry Rate

```text
retried logical requests
/
logical requests
```

can be more useful than raw retry count.

---

# 209. Latency Definition

Decide whether measured latency includes:

```text
retries
queue wait
backoff
```

A user-visible latency metric should usually represent total user-visible time.

---

# 210. Production Incident — Retry Storm

Symptoms:

```text
downstream 503
client retries
traffic triples
downstream worsens
```

Fix:

```text
bounded retry
jitter
deadline
circuit breaker
```

plus downstream coordination.

---

# 211. Production Incident — Duplicate Payment

Cause:

```text
POST timed out after server commit
client retried
```

Fix:

```text
idempotency
```

not merely:

```text
longer timeout
```

---

# 212. Production Incident — Cross-Tenant Authorization Header

Cause:

```text
mutable client default header
```

Fix:

```text
request-scoped auth
immutable defaults
```

---

# 213. Production Incident — Memory Growth

Cause:

```text
unbounded response aggregation
```

Fix:

```text
stream
bounded buffer
```

---

# 214. Production Incident — Hidden 404 Success

Cause:

```text
fetch fulfilled
client ignored response.ok/status
```

Fix:

```text
explicit HTTP status handling
```

---

# 215. Production Incident — Token Refresh Storm

Cause:

```text
each 401 starts independent refresh
```

Fix:

```text
single-flight refresh
```

---

# 216. Production Incident — Slow Search

Cause:

```text
older responses overwrite newer query
```

Fix:

```text
AbortSignal
sequence validation
```

---

# 217. Production Incident — SSRF

Cause:

```text
server-side client accepted arbitrary URL
```

Fix:

```text
destination policy
egress controls
credential isolation
```

---

# 218. Production Incident — Request Queue OOM

Cause:

```text
unbounded client queue
```

Fix:

```text
bounded queue
backpressure
caller-visible rejection
```

---

# 219. Production Incident — Retry After Cancellation

Cause:

```text
AbortError treated as generic failure
```

Fix:

```text
preserve cancellation semantics
```

---

# 220. Architecture Review Checklist

```text
[ ] transport isolated
[ ] request normalization explicit
[ ] URL construction safe
[ ] header precedence defined
[ ] body serialization explicit
[ ] response decoding explicit
[ ] HTTP errors classified
[ ] network errors classified
[ ] timeout distinct from abort
[ ] retry policy explicit
[ ] retry safety defined
[ ] idempotency defined
[ ] backoff bounded
[ ] jitter defined
[ ] deadline supported
[ ] auth scope defined
[ ] credential isolation defined
[ ] SSRF policy defined
[ ] observability bounded
[ ] memory bounded
[ ] concurrency bounded
[ ] tests deterministic
[ ] browser/Node differences documented
```

---

# 221. Implementation Progression

## Stage 1 — Guided

Implement:

```text
GET
POST
base URL
JSON
basic errors
```

## Stage 2 — Partially Guided

Add:

```text
headers
query
response parsing
AbortSignal
```

## Stage 3 — No Reference

Implement:

```text
timeout
retry
backoff
jitter
structured errors
```

## Stage 4 — Edge-Case Hardened

Add:

```text
deadlines
idempotency
auth refresh
non-replayable bodies
streaming
rate limits
```

## Stage 5 — Production Grade

Add:

```text
observability
security
concurrency control
tests
benchmarks
packaging
documentation
compatibility
```

---

# 222. Track A — Core Theory

Study:

```text
HTTP semantics
Fetch
URL
Headers
Request
Response
AbortSignal
streams
cookies
CORS
authentication
retry
idempotency
rate limiting
connection reuse
serialization
observability
security
```

---

# 223. Track B — Implementation

Build:

```text
transport
request model
URL builder
header policy
body serializer
response decoder
error hierarchy
retry engine
deadline engine
auth manager
rate limiter
concurrency limiter
middleware
telemetry hooks
test harness
```

---

# 224. Track C — Interview / Reasoning

Defend:

```text
Why Fetch?
Why transport abstraction?
Why structured errors?
Why no automatic retry of POST?
Why idempotency?
Why deadline plus per-attempt timeout?
Why jitter?
Why request-scoped auth?
Why stream large responses?
Why bound queues?
Why isolate browser/Node policies?
Why inject clock/randomness?
```

---

# 225. Mastery Gate

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

# 226. Mastery Exercise — From Scratch

Build:

```js
const client = createHttpClient({
  baseUrl: "http://localhost:3000"
});
```

Then implement:

```text
GET
POST
PATCH
DELETE
```

with:

```text
JSON
HTTP errors
AbortSignal
```

---

# 227. Mastery Exercise — Retry Engine

Create a local test server with:

```text
503
503
200
```

Implement:

```text
bounded retry
exponential backoff
jitter
```

without real sleeping in unit tests.

---

# 228. Mastery Exercise — Deadline

Implement:

```text
overall 5s deadline
per-attempt 2s timeout
```

Verify total user-visible time never exceeds the overall policy except for unavoidable scheduling/cleanup overhead.

---

# 229. Mastery Exercise — Idempotent Write

Build a mock payment API supporting:

```text
Idempotency-Key
```

Send the same logical operation twice.

Prove that the server produces one logical result.

---

# 230. Mastery Exercise — Streaming

Download a large fixture.

Compare:

```text
buffer entire response
```

versus:

```text
stream processing
```

Measure memory.

---

# 231. Mastery Exercise — Auth Refresh

Simulate:

```text
10 concurrent requests
→ 401
```

Prove:

```text
one refresh
10 retried requests
```

not:

```text
10 refreshes
```

---

# 232. Mastery Exercise — Rate Limit

Simulate:

```text
429
Retry-After
```

and verify:

```text
bounded retry
```

---

# 233. Mastery Exercise — SSRF Defense

Create a server-side endpoint that intentionally accepts URLs.

Demonstrate why:

```text
URL parsing
```

alone is insufficient.

Then add:

```text
allowlist
redirect policy
credential isolation
```

---

# 234. Mastery Exercise — Tenant Isolation

Run requests for:

```text
tenant A
tenant B
```

concurrently.

Prove authentication headers and cache keys remain isolated.

---

# 235. Mastery Exercise — Fault Injection

Test:

```text
DNS-like failure
TLS-like failure
connection reset
timeout
429
500
503
malformed JSON
large body
abort
```

---

# 236. Mastery Exercise — Observability

Add telemetry:

```text
request count
logical failures
physical attempts
retry count
latency
timeouts
cancellations
```

Verify sensitive values are absent.

---

# 237. Mastery Exercise — Benchmark

Benchmark:

```text
direct fetch
client without middleware
client with middleware
client with telemetry
```

Determine the client overhead.

---

# 238. Mastery Exercise — Compatibility

Test:

```text
Node minimum
Node current
browser targets
```

and document differences.

---

# 239. Interview Drill

Answer:

```text
What exactly causes fetch to reject?
What is an HTTP error?
What is a network error?
What is timeout?
What is cancellation?
What does Promise.race not do?
Why can retry duplicate work?
Why is POST not automatically retry-safe?
Why is Retry-After important?
Why does jitter help?
Why do request deadlines matter?
Why can streamed bodies be non-replayable?
Why can server-side URL fetching create SSRF?
Why is CORS not authentication?
```

---

# 240. Principal Interview Drill

Design an HTTP client for:

```text
10,000 requests/sec
multi-tenant
browser + Node
large downloads
payments
third-party APIs
strict security
```

Defend:

```text
retry
idempotency
connection strategy
concurrency
rate limits
timeouts
auth
observability
```

---

# 241. Project Acceptance Criteria

```text
[ ] public API documented
[ ] transport isolated
[ ] URLs constructed safely
[ ] headers deterministic
[ ] JSON bodies supported
[ ] non-JSON bodies defined
[ ] response decoding defined
[ ] HTTP errors structured
[ ] network errors structured
[ ] timeout supported
[ ] cancellation supported
[ ] retry policy bounded
[ ] jitter implemented
[ ] Retry-After handled
[ ] idempotency documented
[ ] non-replayable bodies protected
[ ] auth flow defined
[ ] refresh concurrency controlled
[ ] request queue bounded
[ ] response size considered
[ ] streaming considered
[ ] SSRF defense defined
[ ] telemetry redacted
[ ] deterministic tests exist
[ ] performance benchmark exists
[ ] compatibility documented
[ ] package documented
```

---

# 242. Project Completion Rubric

```text
Level 1 — Calls fetch
Level 2 — Handles JSON
Level 3 — Handles HTTP errors
Level 4 — Handles timeout/cancellation
Level 5 — Handles retries correctly
Level 6 — Handles idempotency
Level 7 — Handles streaming/concurrency
Level 8 — Secure and observable
Level 9 — Tested and packaged
Level 10 — Defensible for principal production use
```

---

# 243. Concept Connections

## Depends On

```text
Chapter 31–38 — Async / Promises / Cancellation / Streams
Chapter 49–57 — Browser / APIs / Fetch / Security
Chapter 58–63 — Node / Runtime
Chapter 71–73 — Data Structures / Complexity
Chapter 78–89 — Production / Reliability / Performance / Testing
Chapter 94 — Compatibility
Chapter 98–101 — Judgment
Chapter 102 — CLI
Chapter 103 — Browser App
```

## Builds Toward

```text
Chapter 105 — Node REST API
Chapter 106 — Real-Time WebSocket
Chapter 107 — Job Queue
Chapter 108 — Cache System
```

## Revisited

```text
AbortSignal
streams
events
HTTP
JSON
errors
security
concurrency
observability
testing
performance
```

---

# 244. Dependency Graph

```text
Async / Promises
      ↓
AbortSignal / Streams
      ↓
Fetch / HTTP
      ↓
Security
      ↓
Reliability
      ↓
Observability
      ↓
Production API Client
      ↓
Node REST API
      ↓
WebSocket / Queue / Cache
```

---

# 245. Spaced Retrieval Schedule

### Day 0

```text
Request
Response
Fetch
HTTP status
```

### Day 1

```text
URL
Headers
JSON
Errors
```

### Day 3

```text
Timeout
AbortSignal
Retry
Backoff
```

### Day 7

```text
Idempotency
Deadlines
Auth refresh
Rate limits
```

### Day 14

```text
Streaming
SSRF
Observability
Concurrency
```

### Day 30

Rebuild the retry engine.

### Day 60

Rebuild authentication and deadline handling.

### Day 90

Defend the complete client architecture.

---

# 246. Revision / Retrieval Record

```md
# Chapter 104 — Revision / Retrieval Record

## Review #
- Date:
- Duration:
- Status before:
- Status after:

## Retrieval
- HTTP semantics [ ]
- Fetch [ ]
- URL construction [ ]
- Query encoding [ ]
- Headers [ ]
- JSON/body serialization [ ]
- Response decoding [ ]
- Error taxonomy [ ]
- Timeout [ ]
- Cancellation [ ]
- Retry policy [ ]
- Backoff/jitter [ ]
- Retry-After [ ]
- Deadline [ ]
- Idempotency [ ]
- Authentication [ ]
- Token refresh [ ]
- Rate limiting [ ]
- Concurrency [ ]
- Streaming [ ]
- SSRF [ ]
- CORS/CSRF [ ]
- Observability [ ]
- Testing [ ]
- Compatibility [ ]

## Build Evidence
- Repository:
- Commit:
- Integration server:
- Load-test result:
- Security test:
- Benchmark:

## Gaps
-

## New Insights
-

## Follow-up
-
```

---

# Chapter 104 — Canonical References and Source Discipline

Primary references:

1. Fetch Standard  
   https://fetch.spec.whatwg.org/

2. MDN Fetch API  
   https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API

3. MDN Request  
   https://developer.mozilla.org/en-US/docs/Web/API/Request

4. MDN Response  
   https://developer.mozilla.org/en-US/docs/Web/API/Response

5. MDN Headers  
   https://developer.mozilla.org/en-US/docs/Web/API/Headers

6. MDN AbortController / AbortSignal  
   https://developer.mozilla.org/en-US/docs/Web/API/AbortController

7. HTTP Semantics — IETF HTTP Working Group  
   https://httpwg.org/specs/

8. Node.js Documentation  
   https://nodejs.org/docs/

9. ECMAScript Language Specification  
   https://tc39.es/ecma262/

10. OWASP SSRF Prevention  
    https://owasp.org/www-community/attacks/Server_Side_Request_Forgery_Prevention_Cheat_Sheet

11. OWASP Web Security  
    https://owasp.org/

Source discipline:

```text
JavaScript semantics
→ ECMAScript

Fetch/browser transport
→ Fetch Standard + Web Platform

HTTP semantics
→ HTTP specifications

Node transport/runtime
→ Node documentation

security
→ threat model + OWASP + platform rules

performance
→ profiling + benchmark

reliability
→ load testing + production telemetry
```

---

# 247. Completion Snapshot

```text
Part XX — Projects

Chapter 104 — Production HTTP Client
[ ] Not Started

Track A — Core Theory
[ ] HTTP methods
[ ] Request/Response
[ ] Fetch
[ ] URL
[ ] query params
[ ] Headers
[ ] JSON
[ ] body types
[ ] response decoding
[ ] status semantics
[ ] error taxonomy
[ ] timeout
[ ] cancellation
[ ] retry
[ ] backoff
[ ] jitter
[ ] Retry-After
[ ] deadlines
[ ] idempotency
[ ] auth
[ ] token refresh
[ ] cookies
[ ] CORS
[ ] CSRF
[ ] SSRF
[ ] redirects
[ ] connection reuse
[ ] rate limiting
[ ] concurrency
[ ] streaming
[ ] caching
[ ] observability
[ ] schemas
[ ] testing
[ ] compatibility

Track B — Implementation
[ ] transport
[ ] client
[ ] request normalizer
[ ] URL builder
[ ] headers
[ ] body serializer
[ ] response decoder
[ ] errors
[ ] timeout
[ ] retry engine
[ ] deadline engine
[ ] auth manager
[ ] refresh single-flight
[ ] rate limiter
[ ] concurrency limiter
[ ] middleware
[ ] telemetry
[ ] streaming
[ ] test transport
[ ] integration server
[ ] benchmark harness
[ ] package/release

Track C — Interview / Reasoning
[ ] Explain Fetch
[ ] Explain HTTP errors
[ ] Explain retry safety
[ ] Explain idempotency
[ ] Explain cancellation
[ ] Explain deadlines
[ ] Explain auth refresh
[ ] Explain rate limiting
[ ] Explain SSRF
[ ] Explain streaming
[ ] Explain Node/browser differences
[ ] Explain logical vs physical attempts
[ ] Defend architecture
[ ] Defend security model
[ ] Defend performance strategy
[ ] Defend reliability model

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

# 248. Completion Criteria

Do not mark the project mastered because it can successfully call an API.

You are ready to move forward when you can independently:

1. Design the request pipeline.
2. Separate transport from policy.
3. Construct URLs safely.
4. Serialize bodies correctly.
5. Decode responses safely.
6. Distinguish HTTP, network, timeout, and cancellation errors.
7. Implement bounded retries.
8. Add backoff and jitter.
9. Respect retry deadlines.
10. Reason about idempotency.
11. Handle non-replayable bodies.
12. Implement request cancellation.
13. Combine external abort with internal deadlines.
14. Manage token refresh concurrency.
15. Isolate tenant credentials.
16. Handle rate limits.
17. Bound client concurrency and queues.
18. Stream large data.
19. Prevent SSRF in server-side use.
20. Avoid secret leakage in telemetry.
21. Test every major failure mode deterministically.
22. Benchmark client overhead.
23. Explain browser/Node differences.
24. Package a stable reusable API.
25. Defend the complete architecture at principal level.

---

# Final Mental Model

```text
Application Intent
      ↓
Request Model
      ↓
Policy
 ├── URL
 ├── Headers
 ├── Auth
 ├── Deadline
 ├── Retry
 ├── Rate Limit
 └── Cancellation
      ↓
Transport
      ↓
HTTP Response
      ↓
Decode
      ↓
Validate
      ↓
Domain Result
```

Failure path:

```text
Failure
 ↓
Classify
 ↓
Retry?
 ├── no → surface
 └── yes
      ↓
   budget?
      ↓
   backoff
      ↓
   retry
```

Production boundary:

```text
Correctness
+
Security
+
Reliability
+
Performance
+
Observability
+
Compatibility
```

> **Mastery reminder:** The client should never “retry because something went wrong.” It should decide whether the operation is safe to replay, whether time and capacity remain, and whether the caller still wants the result.