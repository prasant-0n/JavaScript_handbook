# Chapter 55 — Fetch, HTTP Networking, and Request Lifecycles

> **Curriculum Position:** Part X — Networking / Security  
> **Prerequisites:** Chapters 31–39, 49–53  
> **Primary Focus:** Fetch API, Request/Response semantics, HTTP, headers, bodies, CORS, credentials, redirects, caching, abort/cancellation, streaming, uploads, retries, timeouts, connection behavior, service workers, security, observability, performance, and production network-client architecture  
> **Status:** `[ ] Not Started`  
> **Depth Target:** Browser networking model → Fetch API → HTTP semantics → security boundaries → streaming/cancellation → production client architecture  
> **Scope Rule:** Fetch and HTTP networking are Web Platform/host capabilities, not ECMAScript language features. ECMAScript supplies JavaScript execution, while the browser provides Fetch and related APIs.

---

# Chapter Navigation

- [1. Learning Objectives](#1-learning-objectives)
- [2. Prerequisites](#2-prerequisites)
- [3. What Is Fetch?](#3-what-is-fetch)
- [4. Why Fetch Exists](#4-why-fetch-exists)
- [5. Mental Model](#5-mental-model)
- [6. Network Stack Layers](#6-network-stack-layers)
- [7. HTTP Fundamentals](#7-http-fundamentals)
- [8. Request](#8-request)
- [9. Response](#9-response)
- [10. Headers](#10-headers)
- [11. Bodies](#11-bodies)
- [12. Body Consumption and Streams](#12-body-consumption-and-streams)
- [13. `fetch()` Promise Semantics](#13-fetch-promise-semantics)
- [14. HTTP Status and Error Semantics](#14-http-status-and-error-semantics)
- [15. Methods and Idempotency](#15-methods-and-idempotency)
- [16. Content Types and Serialization](#16-content-types-and-serialization)
- [17. CORS](#17-cors)
- [18. Preflight Requests](#18-preflight-requests)
- [19. Credentials and Cookies](#19-credentials-and-cookies)
- [20. Origin and Same-Origin Policy](#20-origin-and-same-origin-policy)
- [21. Redirects](#21-redirects)
- [22. Referrer and Referrer Policy](#22-referrer-and-referrer-policy)
- [23. HTTP Caching](#23-http-caching)
- [24. Fetch Cache Modes](#24-fetch-cache-modes)
- [25. Browser Cache vs Cache API](#25-browser-cache-vs-cache-api)
- [26. AbortController and Cancellation](#26-abortcontroller-and-cancellation)
- [27. Timeouts](#27-timeouts)
- [28. Uploads](#28-uploads)
- [29. Download Progress and Streaming](#29-download-progress-and-streaming)
- [30. Request Streaming](#30-request-streaming)
- [31. Keepalive](#31-keepalive)
- [32. Network Failures](#32-network-failures)
- [33. Retries](#33-retries)
- [34. Backoff and Jitter](#34-backoff-and-jitter)
- [35. Concurrency Control](#35-concurrency-control)
- [36. Connection and Protocol Concepts](#36-connection-and-protocol-concepts)
- [37. TLS / HTTPS](#37-tls--https)
- [38. Service Workers and Fetch](#38-service-workers-and-fetch)
- [39. Request/Response Interception](#39-requestresponse-interception)
- [40. WebSockets / WebTransport Boundary](#40-websockets--webtransport-boundary)
- [41. Browser Security](#41-browser-security)
- [42. CSRF and Credentialed Requests](#42-csrf-and-credentialed-requests)
- [43. Authentication](#43-authentication)
- [44. Authorization](#44-authorization)
- [45. API Client Architecture](#45-api-client-architecture)
- [46. Data Validation](#46-data-validation)
- [47. Error Taxonomy](#47-error-taxonomy)
- [48. Observability](#48-observability)
- [49. Performance](#49-performance)
- [50. Memory](#50-memory)
- [51. Security](#51-security)
- [52. Production Patterns](#52-production-patterns)
- [53. Anti-Patterns](#53-anti-patterns)
- [54. Debugging Methodology](#54-debugging-methodology)
- [55. Implementation From Scratch](#55-implementation-from-scratch)
- [56. Debugging Exercises](#56-debugging-exercises)
- [57. Code Review Exercise](#57-code-review-exercise)
- [58. Interview Questions](#58-interview-questions)
- [59. Predict-the-Output Exercises](#59-predict-the-output-exercises)
- [60. Mastery Exercises](#60-mastery-exercises)
- [61. Key Takeaways](#61-key-takeaways)
- [62. Concept Connections](#62-concept-connections)
- [63. Completion Criteria](#63-completion-criteria)
- [64. Revision / Retrieval Record](#64-revision--retrieval-record)
- [65. Canonical References and Source Discipline](#65-canonical-references-and-source-discipline)
- [66. Completion Snapshot](#66-completion-snapshot)

---

# 1. Learning Objectives

By the end of this chapter, you should be able to:

1. Explain what Fetch is and how it relates to HTTP.
2. Distinguish ECMAScript from browser networking APIs.
3. Explain the lifecycle of a browser request.
4. Construct and inspect `Request` objects.
5. Construct and inspect `Response` objects.
6. Work with `Headers`.
7. Explain request and response bodies.
8. Consume text, JSON, blobs, array buffers, and streams.
9. Explain why `fetch()` resolves for many HTTP error statuses.
10. Correctly classify HTTP success, application errors, protocol errors, network errors, aborts, and parsing failures.
11. Explain HTTP methods and idempotency.
12. Explain content negotiation and serialization.
13. Explain the same-origin policy.
14. Explain CORS.
15. Explain preflight requests.
16. Explain credentials modes.
17. Explain cookies and their security implications.
18. Explain redirects.
19. Explain referrer behavior.
20. Explain browser HTTP caching.
21. Explain Fetch cache modes.
22. Distinguish HTTP caching from the Cache API.
23. Implement cancellation with `AbortController`.
24. Implement robust timeouts.
25. Stream large responses.
26. Understand streaming uploads and their browser/runtime constraints.
27. Understand `keepalive`.
28. Design safe retries.
29. Implement exponential backoff and jitter.
30. Design concurrency limits.
31. Explain HTTPS/TLS at the correct abstraction level.
32. Explain service-worker interception.
33. Design an API client abstraction without hiding important browser semantics.
34. Validate untrusted response data.
35. Design an actionable error taxonomy.
36. Build observability into network clients.
37. Diagnose CORS, authentication, cache, redirect, timeout, and network failures.
38. Evaluate networking decisions across correctness, security, performance, reliability, memory, and operability.
39. Implement a production-grade Fetch wrapper.
40. Defend when to use Fetch, XHR, WebSocket, WebTransport, or a higher-level data-fetching abstraction.

### Mastery target

**Understand → Explain → Predict → Implement → Debug → Apply → Compare → Defend**

---

# 2. Prerequisites

## Required

### Chapter 31 — Async Fundamentals

You must understand asynchronous execution.

### Chapter 35 — Promises

Fetch returns a Promise.

### Chapter 37 — Cancellation / Abort

`AbortSignal` is central to production Fetch usage.

### Chapter 38 — Async Iteration / Streaming

Needed for streaming bodies.

### Chapter 49 — DOM Architecture

Needed for browser host integration.

### Chapter 51 — Browser Web APIs

Needed for browser-global APIs.

---

# 3. What Is Fetch?

The **Fetch API** provides a Web Platform interface for making resource requests.

The central entry point is:

```js
fetch(input, init)
```

It returns a Promise for a `Response`.

MDN describes Fetch as the modern Web API for fetching resources and as a more powerful, flexible replacement for many XMLHttpRequest use cases. The `fetch()` method is available in Window and Worker contexts. citeturn845291search3

---

## Minimal Request

```js
const response = await fetch("/api/users");
```

Then:

```js
const users = await response.json();
```

---

## Critical Semantic

This:

```js
await fetch("/missing");
```

does not automatically throw merely because the HTTP status is:

```text
404
500
```

The Promise resolves with a `Response` when the fetch completes at the API level; HTTP status must be checked separately. citeturn845291search3

---

# 4. Why Fetch Exists

Older browser applications frequently relied on:

```text
XMLHttpRequest
```

Fetch provides a more composable model around:

```text
Request
Response
Headers
Body / ReadableStream
AbortSignal
CORS
credentials
cache
redirects
```

---

# 5. Mental Model

Do not model Fetch as:

```text
fetch(url)
  ↓
server
  ↓
JSON
```

Use:

```text
JavaScript
   ↓
Request construction
   ↓
Fetch processing
   ↓
security policy
   ↓
service worker interception
   ↓
HTTP cache / network
   ↓
redirects
   ↓
server
   ↓
HTTP response
   ↓
security filtering
   ↓
Response object
   ↓
body stream
   ↓
application parsing
```

---

# 6. Network Stack Layers

A useful conceptual stack:

```text
Application
    ↓
Fetch API
    ↓
HTTP semantics
    ↓
TLS / HTTPS
    ↓
TCP or modern transport
    ↓
IP
    ↓
Network
```

Do not confuse:

```text
Fetch
```

with:

```text
HTTP
```

Fetch orchestrates a browser-facing request algorithm around HTTP and other resource-fetching concerns.

The WHATWG Fetch Standard defines request modes, credentials, cache modes, redirect modes, CORS processing, response filtering, and the fetch algorithm. citeturn845291search0

---

# 7. HTTP Fundamentals

HTTP provides request/response semantics.

Basic shape:

```text
Request
    method
    target
    headers
    body

Response
    status
    headers
    body
```

---

## Example HTTP Request

```http
POST /api/users HTTP/1.1
Host: example.com
Content-Type: application/json

{"name":"Prasanta"}
```

---

## Example HTTP Response

```http
HTTP/1.1 201 Created
Content-Type: application/json

{"id":"42","name":"Prasanta"}
```

---

# 8. Request

The Fetch `Request` object represents a request.

```js
const request = new Request("/api/users", {
  method: "GET"
});
```

---

## Inspect

```js
request.method;
request.url;
request.headers;
request.mode;
request.credentials;
request.cache;
request.redirect;
request.signal;
```

The Fetch Standard defines request properties including `mode`, `credentials`, `cache`, `redirect`, `referrerPolicy`, `integrity`, `keepalive`, `signal`, and related request metadata. citeturn845291search0

---

# 9. Response

A `Response` represents the fetch result visible to the calling context.

```js
const response = await fetch("/api/users");

console.log(response.status);
console.log(response.ok);
console.log(response.headers);
```

---

## Common Properties

```js
response.status
response.statusText
response.ok
response.url
response.type
response.redirected
response.headers
response.body
```

---

# 10. Headers

Use:

```js
const headers = new Headers();

headers.set("Accept", "application/json");
headers.set("X-Request-ID", "abc");
```

---

## Request Headers

```js
await fetch("/api/users", {
  headers: {
    "Accept": "application/json"
  }
});
```

---

## Response Headers

```js
const value = response.headers.get("Content-Type");
```

---

## Important Security Boundary

Not every server response header is necessarily exposed to JavaScript in every cross-origin scenario.

CORS controls which response headers a browser caller can inspect.

The Fetch Standard defines CORS-exposed header processing for filtered responses. citeturn845291search0

---

# 11. Bodies

A body can exist on requests and responses.

Common body representations:

```text
string
Blob
FormData
URLSearchParams
ArrayBuffer
TypedArray
ReadableStream
```

---

## JSON Request

```js
await fetch("/api/users", {
  method: "POST",
  headers: {
    "Content-Type": "application/json"
  },
  body: JSON.stringify({
    name: "Prasanta"
  })
});
```

---

## FormData

```js
const form = new FormData();

form.append("name", "Prasanta");

await fetch("/upload", {
  method: "POST",
  body: form
});
```

Do not manually set `Content-Type` for ordinary `FormData` uploads. The browser must generate the multipart boundary.

---

# 12. Body Consumption and Streams

A response body is stream-oriented.

Examples:

```js
const text = await response.text();
```

```js
const json = await response.json();
```

```js
const blob = await response.blob();
```

```js
const buffer = await response.arrayBuffer();
```

---

## Body Is Consumable

```js
const response = await fetch("/data");

const first = await response.json();
```

Attempting another full consumption:

```js
await response.json();
```

does not work as though the body were an ordinary reusable object.

---

## `bodyUsed`

```js
console.log(response.bodyUsed);
```

---

## `clone()`

To create another consumable response view when supported by the body model:

```js
const copy = response.clone();
```

Use this intentionally, because duplication can have buffering/resource consequences.

---

# 13. `fetch()` Promise Semantics

Important distinction:

```text
HTTP 404
HTTP 500
```

are not automatically Promise rejections.

This often surprises developers.

---

## Example

```js
try {
  const response = await fetch("/missing");

  console.log("resolved", response.status);
} catch (error) {
  console.log("rejected", error);
}
```

A server response with `404` can reach the resolved path.

---

## Network Failure

Examples such as:

```text
DNS failure
connection failure
blocked request
CORS failure visible to caller
abort
```

can result in rejection/network-error behavior.

---

# 14. HTTP Status and Error Semantics

Use:

```js
if (!response.ok) {
  throw new Error(`HTTP ${response.status}`);
}
```

for simple clients.

But production systems should distinguish:

```text
400
401
403
404
409
422
429
500
502
503
504
```

because the correct application behavior can differ.

---

## Example Taxonomy

```text
401
→ authentication failure

403
→ authorization failure

404
→ resource missing

409
→ conflict

422
→ semantic validation

429
→ rate limit

5xx
→ server/upstream/service failure
```

---

# 15. Methods and Idempotency

Common HTTP methods:

```text
GET
HEAD
POST
PUT
PATCH
DELETE
OPTIONS
```

---

## Idempotency

An operation is idempotent when repeating it has the same intended effect on server state as performing it once, even though individual responses can differ.

Typical examples:

```text
GET
PUT
DELETE
```

are generally defined as idempotent in HTTP semantics.

`POST` is not generally idempotent.

---

## Why This Matters

Retrying:

```js
POST /payments
```

blindly can duplicate an operation.

Production APIs may use:

```text
Idempotency-Key
```

or another deduplication strategy.

---

# 16. Content Types and Serialization

Common formats:

```text
application/json
text/plain
application/x-www-form-urlencoded
multipart/form-data
application/octet-stream
```

---

## JSON

Encode:

```js
JSON.stringify(data)
```

and set:

```http
Content-Type: application/json
```

---

## Parse

```js
const data = await response.json();
```

But the server may return:

```text
HTML
plain text
malformed JSON
empty body
```

so robust clients must handle parse failures.

---

# 17. CORS

**Cross-Origin Resource Sharing** allows a server to opt into sharing responses with browser code from other origins.

The Fetch Standard defines CORS as an HTTP-header protocol layered on Fetch to allow controlled cross-origin response sharing. citeturn845291search0

---

## Same Origin

Origin consists conceptually of:

```text
scheme
host
port
```

Example:

```text
https://api.example.com:443
```

is different from:

```text
https://api.example.com:8443
```

and:

```text
http://api.example.com
```

---

## CORS Response

Example:

```http
Access-Control-Allow-Origin: https://app.example.com
```

This permits the browser to expose the response to the specified origin when other requirements are satisfied.

---

# 18. Preflight Requests

Some cross-origin requests require an OPTIONS preflight.

Conceptual flow:

```text
Browser
  ↓
OPTIONS /api
  Origin: https://app.example.com
  Access-Control-Request-Method: PATCH
  Access-Control-Request-Headers: X-Custom-Header
  ↓
Server
  ↓
Access-Control-Allow-Methods
Access-Control-Allow-Headers
Access-Control-Allow-Origin
  ↓
Browser
  ↓
PATCH /api
```

The Fetch Standard specifies CORS-preflight processing for requests that require it. citeturn845291search0

---

## Why Preflight Exists

It protects servers from unexpected cross-origin methods/headers by requiring an explicit capability response for requests outside the CORS-safelisted envelope.

---

# 19. Credentials and Cookies

Fetch credentials mode:

```text
omit
same-origin
include
```

The Fetch Standard defines `same-origin` as the default credentials mode and describes how these values govern inclusion/use of credentials. citeturn845291search0

---

## Example

```js
fetch("/api/me", {
  credentials: "include"
});
```

---

## Credential Types

Browser credentials can include:

```text
cookies
TLS client certificates
Authorization headers
Proxy-Authorization
```

MDN documents these credential categories and the default/supported Fetch credentials modes. citeturn845291search1

---

## Cross-Origin Credentials

For credentialed cross-origin requests, server CORS configuration must explicitly permit the requesting origin and credentials; wildcard `Access-Control-Allow-Origin: *` cannot be used for the credentialed response-sharing case. citeturn845291search0turn845291search1

---

# 20. Origin and Same-Origin Policy

The browser does not generally allow arbitrary scripts to read arbitrary cross-origin responses.

This protects data such as:

```text
private APIs
intranet resources
personal data
```

from arbitrary website access.

---

## Important Distinction

CORS primarily controls:

```text
whether browser JavaScript can read a cross-origin response
```

It is not identical to:

```text
whether the server receives the request
```

Some cross-origin requests can be sent while the browser blocks the calling script from reading the response.

---

# 21. Redirects

Fetch redirect modes include:

```text
follow
error
manual
```

with `follow` as the usual default. The Fetch Standard defines these modes and their behavior. citeturn845291search0

---

## Follow

```js
fetch(url, {
  redirect: "follow"
});
```

Automatically follow supported redirects.

---

## Error

```js
fetch(url, {
  redirect: "error"
});
```

Treat redirects as an error.

---

## Manual

```js
fetch(url, {
  redirect: "manual"
});
```

Produces special redirect-filtering behavior rather than a normal exposed redirect response.

---

# 22. Referrer and Referrer Policy

Requests can carry referrer information subject to browser policy.

Use:

```js
fetch(url, {
  referrerPolicy: "no-referrer"
});
```

or configure policy using HTTP/document mechanisms.

---

## Why It Matters

Referrer information can reveal:

```text
paths
campaign data
internal URL structure
sensitive query information
```

Use an appropriate referrer policy.

---

# 23. HTTP Caching

HTTP caching can avoid unnecessary network transfers.

Important concepts:

```text
Cache-Control
ETag
Last-Modified
Age
Expires
Vary
conditional requests
freshness
revalidation
```

---

## ETag Example

First response:

```http
ETag: "abc123"
```

Later:

```http
If-None-Match: "abc123"
```

Server:

```http
304 Not Modified
```

The browser can reuse the cached representation according to HTTP caching semantics.

---

# 24. Fetch Cache Modes

Fetch exposes cache modes such as:

```text
default
no-store
reload
no-cache
force-cache
only-if-cached
```

The Fetch Standard specifies these cache modes and their interaction with the HTTP cache. citeturn845291search0

---

## Example

```js
fetch(url, {
  cache: "no-store"
});
```

---

## Important Distinction

```text
cache: "no-store"
```

is not the same conceptual tool as:

```text
Cache API
```

---

# 25. Browser Cache vs Cache API

## HTTP Cache

Browser-managed HTTP caching governed by HTTP/Fetched resource semantics.

## Cache API

JavaScript-accessible `Cache` objects store `Request`/`Response` pairs.

MDN documents the Cache interface as a persistent request/response storage mechanism and notes that applications/service workers control how entries are populated and updated. citeturn845291search2

---

## Cache API Example

```js
const cache = await caches.open("app-v1");

await cache.put(
  new Request("/data"),
  response.clone()
);
```

---

## Critical Rule

Using the Cache API does not automatically reproduce normal browser HTTP cache semantics.

The application owns the Cache API policy.

---

# 26. AbortController and Cancellation

Fetch can be canceled with:

```js
const controller = new AbortController();

fetch("/large", {
  signal: controller.signal
});

controller.abort();
```

---

## Why Cancellation Matters

Avoid wasting:

```text
network
CPU
memory
server work
UI work
```

when a request is no longer needed.

---

## Component Integration

From Chapter 54:

```js
connectedCallback() {
  this.#controller = new AbortController();

  fetch("/api/data", {
    signal: this.#controller.signal
  });
}

disconnectedCallback() {
  this.#controller?.abort();
}
```

---

# 27. Timeouts

Fetch does not make every request magically bounded by a universal application timeout.

Implement one explicitly.

---

## Modern Pattern

```js
const signal = AbortSignal.timeout(5_000);

const response = await fetch("/api", {
  signal
});
```

Also consider a combined signal when you need both:

```text
caller cancellation
+
deadline
```

---

## Production Rule

A timeout is not just:

```text
throw after N seconds
```

It is a cancellation/deadline policy.

---

# 28. Uploads

## JSON

```js
fetch("/api", {
  method: "POST",
  headers: {
    "Content-Type": "application/json"
  },
  body: JSON.stringify(payload)
});
```

---

## File Upload

```js
const body = new FormData();

body.append("avatar", file);

fetch("/upload", {
  method: "POST",
  body
});
```

---

## Binary

```js
fetch("/upload", {
  method: "POST",
  body: arrayBuffer
});
```

---

# 29. Download Progress and Streaming

Fetch does not provide the same simple `onprogress` property model associated with XHR.

For streaming response bodies:

```js
const response = await fetch("/large");

const reader = response.body.getReader();

while (true) {
  const { done, value } = await reader.read();

  if (done) break;

  consume(value);
}
```

---

## Why Streaming Matters

Without streaming:

```text
network
→ entire payload buffered
→ parser
→ application
```

With streaming:

```text
network
→ chunk
→ process
→ chunk
→ process
```

This can reduce time-to-first-use and memory pressure for suitable workloads.

---

# 30. Request Streaming

Modern Fetch implementations can support streaming request bodies using a `ReadableStream`, with additional request constraints such as the `duplex` option.

The Fetch Standard defines a `duplex` request option and request-body streaming-related behavior. citeturn845291search0

Example:

```js
const stream = new ReadableStream({
  start(controller) {
    controller.enqueue(
      new TextEncoder().encode("hello\n")
    );
    controller.close();
  }
});

await fetch("/upload-stream", {
  method: "POST",
  body: stream,
  duplex: "half"
});
```

---

## Compatibility Rule

Streaming upload support historically varied across environments.

Verify:

```text
browser
runtime
server
proxy
framework
```

before making request streaming a baseline dependency.

---

# 31. Keepalive

Fetch supports:

```js
fetch("/analytics", {
  method: "POST",
  keepalive: true,
  body: JSON.stringify(event)
});
```

This is designed for requests that need to survive certain page lifecycle transitions.

---

## Constraints

Keepalive requests have practical limitations, including payload-size and lifetime considerations.

Do not use it as a generic “never lose my request” guarantee.

---

# 32. Network Failures

Classify failures.

## 32.1 Transport Failure

Examples:

```text
DNS
connection refused
network disconnected
TLS failure
```

---

## 32.2 Browser Policy Failure

Examples:

```text
CORS rejection
blocked mixed-content request
security policy
```

---

## 32.3 Abort

```text
AbortError
```

or equivalent abort behavior.

---

## 32.4 HTTP Failure

```text
404
500
503
```

These can still yield a `Response`.

---

## 32.5 Parse Failure

```js
await response.json();
```

can fail if the body is malformed.

---

## 32.6 Application Failure

A successful HTTP response can still carry:

```json
{
  "ok": false,
  "error": "Business rule violation"
}
```

Do not let:

```text
HTTP 200
```

define application correctness.

---

# 33. Retries

Retry only when safe and useful.

Good candidates can include transient:

```text
502
503
504
network failures
```

but the policy must account for method/idempotency and server state.

---

## Bad

```js
while (true) {
  await fetch(url);
}
```

---

## Better

```text
attempt
 ↓
classify failure
 ↓
retryable?
 ↓ yes
backoff
 ↓
retry
```

---

# 34. Backoff and Jitter

A common strategy:

```text
delay = min(maxDelay, base * 2^attempt)
```

Then add jitter:

```text
actualDelay = randomized(delay)
```

---

## Why Jitter?

Without jitter, many clients can retry simultaneously:

```text
failure
 ↓
all clients wait 1s
 ↓
all clients retry
 ↓
server overloaded
 ↓
failure
```

This creates a retry storm.

---

# 35. Concurrency Control

Suppose the application launches:

```js
Promise.all(
  urls.map(url => fetch(url))
);
```

For a large list this can overwhelm:

```text
browser
server
network
memory
```

---

## Semaphore Pattern

```js
class Semaphore {
  #active = 0;
  #queue = [];

  constructor(limit) {
    this.limit = limit;
  }

  async acquire() {
    if (this.#active < this.limit) {
      this.#active++;
      return;
    }

    await new Promise(resolve => {
      this.#queue.push(resolve);
    });

    this.#active++;
  }

  release() {
    this.#active--;

    const next = this.#queue.shift();

    if (next) next();
  }
}
```

---

## Why Limit?

Production networking is an orchestration problem, not merely a Promise problem.

---

# 36. Connection and Protocol Concepts

Do not assume:

```text
one fetch = one TCP connection
```

The browser controls connection pooling/reuse and protocol implementation.

Modern web traffic can use:

```text
HTTP/1.1
HTTP/2
HTTP/3
```

depending on origin/server/network support.

---

## HTTP/1.1

Conceptually:

```text
request/response over persistent connections
```

---

## HTTP/2

Adds concepts such as:

```text
multiplexed streams
header compression
binary framing
```

---

## HTTP/3

Runs HTTP semantics over:

```text
QUIC
```

rather than TCP.

---

## Application Rule

The application should generally program against:

```text
Fetch + HTTP semantics
```

rather than assuming a transport version.

---

# 37. TLS / HTTPS

HTTPS combines:

```text
HTTP
+
TLS
```

to provide properties such as:

```text
confidentiality
integrity
server authentication
```

---

## Application Rule

Prefer HTTPS everywhere for application APIs.

---

## Mixed Content

Avoid loading insecure HTTP application resources from secure HTTPS pages.

Browser security policy can block or restrict such requests.

---

# 38. Service Workers and Fetch

A service worker can intercept applicable requests.

Conceptually:

```text
Page
 ↓
fetch()
 ↓
service worker
 ↓
cache / network / custom logic
```

---

## Example

```js
self.addEventListener("fetch", event => {
  event.respondWith(
    caches.match(event.request)
      .then(cached => cached ?? fetch(event.request))
  );
});
```

---

## Why This Matters

Service workers enable:

```text
offline
cache strategies
network fallback
request interception
response rewriting
```

---

# 39. Request/Response Interception

A service worker can create behavior such as:

```text
cache-first
network-first
stale-while-revalidate
offline fallback
```

---

## Cache-First

```text
cache hit
→ return cache

cache miss
→ network
→ cache response
```

---

## Network-First

```text
network
→ success
→ return + cache

network failure
→ cache fallback
```

---

## Principal Concern

Caching is not simply an optimization.

It changes:

```text
freshness
correctness
consistency
security
offline behavior
```

---

# 40. WebSockets / WebTransport Boundary

Fetch is primarily:

```text
request/response
```

WebSocket is:

```text
long-lived bidirectional message channel
```

WebTransport provides a different modern transport model with streaming/datagram capabilities.

Do not force a polling or request/response architecture onto a problem that requires continuous bidirectional communication.

---

# 41. Browser Security

Networking is a major browser security boundary.

Important controls include:

```text
same-origin policy
CORS
CSP
mixed-content rules
CORP
COEP
COOP
CSRF defenses
cookie policies
TLS
```

Not every mechanism belongs to Fetch alone.

---

## Security Mental Model

```text
Can the request be sent?
       ≠
Can JavaScript read the response?
       ≠
Can credentials accompany the request?
       ≠
Can the response be embedded?
```

These are separate questions.

---

# 42. CSRF and Credentialed Requests

Credentialed requests:

```js
fetch("/transfer", {
  method: "POST",
  credentials: "include",
  body: ...
});
```

can carry authentication cookies.

---

## CSRF Risk

If authorization relies on cookies, an attacker-controlled site may try to cause a victim browser to send a state-changing request.

Common defenses include:

```text
SameSite cookies
CSRF tokens
Origin/Referer validation
server-side authorization checks
```

CORS is not a complete replacement for CSRF defenses.

---

# 43. Authentication

Common browser API approaches:

## Cookie Session

```text
browser cookie
→ authenticated request
```

Advantages:

```text
browser-native session model
```

Requires careful:

```text
CSRF
SameSite
Secure
HttpOnly
```

configuration.

---

## Bearer Token

```http
Authorization: Bearer <token>
```

The token must be protected from:

```text
XSS
logging
leaks
third-party scripts
```

Do not casually place sensitive access tokens in:

```text
URL
console logs
analytics payload
```

---

# 44. Authorization

Authentication answers:

```text
Who are you?
```

Authorization answers:

```text
What are you allowed to do?
```

The server must enforce authorization.

Never rely on client-side UI checks:

```js
if (user.isAdmin) {
  showDeleteButton();
}
```

as the actual authorization boundary.

---

# 45. API Client Architecture

A production client should separate:

```text
transport
protocol
authentication
serialization
validation
domain mapping
retry policy
observability
UI behavior
```

---

## Recommended Layers

```text
UI
 ↓
domain client
 ↓
API client
 ↓
Fetch adapter
 ↓
browser
```

---

## Example

```js
async function requestJson(
  input,
  {
    signal,
    ...init
  } = {}
) {
  const response = await fetch(input, {
    ...init,
    signal,
    headers: {
      Accept: "application/json",
      ...init.headers
    }
  });

  const text = await response.text();

  let data = null;

  if (text) {
    try {
      data = JSON.parse(text);
    } catch {
      throw new Error("Invalid JSON response");
    }
  }

  if (!response.ok) {
    throw new Error(
      `HTTP ${response.status}`
    );
  }

  return data;
}
```

This is only a foundation.

---

# 46. Data Validation

Never assume remote JSON has the expected shape.

Bad:

```js
const user = await response.json();

console.log(user.profile.address.city);
```

A malicious, corrupt, incompatible, or changed API could return:

```json
{
  "profile": null
}
```

---

## Production Rule

Treat every network response as untrusted external input.

Use:

```text
schema validation
runtime guards
versioned contracts
```

where appropriate.

---

# 47. Error Taxonomy

A good client can distinguish:

```text
NetworkError
AbortError
TimeoutError
CorsError
HttpError
ParseError
ValidationError
AuthenticationError
AuthorizationError
RateLimitError
ServerError
```

---

## Why This Matters

UI can make different decisions:

```text
Abort
→ no error toast

401
→ reauthenticate

403
→ show authorization message

429
→ retry/backoff

503
→ temporary outage

validation
→ show field errors
```

---

# 48. Observability

Instrument requests with:

```text
request ID
duration
status
retry count
abort reason
cache outcome
payload size
endpoint
```

---

## Example

```js
const started = performance.now();

try {
  const response = await fetch(url, {
    headers: {
      "X-Request-ID": requestId
    }
  });

  console.log({
    url,
    status: response.status,
    duration: performance.now() - started
  });

  return response;
} catch (error) {
  console.error({
    url,
    duration: performance.now() - started,
    error
  });

  throw error;
}
```

---

## Privacy Rule

Never log:

```text
passwords
session tokens
Authorization headers
sensitive request bodies
private response payloads
```

---

# 49. Performance

Measure:

```text
DNS
connection
TLS
request wait
TTFB
download
parse
application processing
```

Browser performance APIs can expose parts of resource/network timing under their respective security/visibility rules.

---

## Performance Levers

```text
compression
HTTP caching
payload reduction
streaming
pagination
request batching
concurrency limits
connection reuse
CDN
edge caching
prefetching
```

---

## Avoid

```text
1000 serial requests
```

when:

```text
1 batched request
```

would be correct.

Also avoid:

```text
1000 parallel requests
```

when concurrency should be bounded.

---

# 50. Memory

Large responses can be expensive:

```js
const data = await response.json();
```

This can require memory for:

```text
raw network buffers
decoded bytes
string
parsed JS objects
application copies
```

Potentially several representations exist during processing.

---

## Streaming

For large workloads:

```text
stream
→ incremental parsing
→ bounded buffers
```

can reduce peak memory.

---

## Clone Carefully

```js
const a = response.clone();
const b = response.clone();
```

can multiply work/buffering requirements.

Clone because the architecture needs it, not as a default habit.

---

# 51. Security

## Do Not Trust URLs

Validate:

```text
scheme
origin
host
path
```

where user-controlled URLs can become dangerous.

---

## SSRF Context

A browser generally operates under strong browser-origin/network restrictions, while server-side Fetch implementations may have very different network reach.

Never copy a browser URL-fetch pattern into a backend system without reassessing SSRF risk.

---

## JSON Injection

JSON.parse is not itself an HTML sanitizer.

After parsing:

```js
element.innerHTML = data.html;
```

can create XSS if the content is untrusted.

---

## Sensitive Query Parameters

Avoid putting secrets in:

```text
URL
query string
fragment when shared/logged
```

because URLs may be recorded by:

```text
history
logs
analytics
proxy infrastructure
referrers
```

depending on context.

---

# 52. Production Patterns

## Pattern 1 — Explicit timeout

```js
const response = await fetch(
  url,
  {
    signal: AbortSignal.timeout(10_000)
  }
);
```

---

## Pattern 2 — Caller-controlled cancellation

```js
async function load(signal) {
  return fetch("/api", { signal });
}
```

---

## Pattern 3 — HTTP error conversion

```js
if (!response.ok) {
  throw new HttpError(response.status);
}
```

---

## Pattern 4 — Retry only safe/transient failures

```text
network
429
502
503
504
```

depending on API semantics.

---

## Pattern 5 — Retry with jitter

```text
base
→ exponential
→ cap
→ jitter
```

---

## Pattern 6 — Concurrency limit

```text
N active requests
+
queue
```

---

## Pattern 7 — Schema validation

```text
Response
→ parse
→ validate
→ map
```

---

## Pattern 8 — Abort obsolete work

```text
search query A
→ user types
→ cancel A
→ request B
```

---

## Pattern 9 — Cache with explicit freshness policy

```text
cache
+
revalidation
+
invalidation
```

---

## Pattern 10 — Correlation IDs

```http
X-Request-ID: ...
```

for cross-system debugging where appropriate.

---

# 53. Anti-Patterns

## Anti-Pattern 1

```js
fetch(url);
```

with no:

```text
error handling
timeout
cancellation
status handling
```

for critical production operations.

---

## Anti-Pattern 2

```js
catch {
  return {};
}
```

Hides failures.

---

## Anti-Pattern 3

```js
if (!response.ok) {
  throw new Error("Failed");
}
```

without preserving:

```text
status
endpoint
server error
request ID
retryability
```

---

## Anti-Pattern 4

```js
Promise.all(hugeArray.map(fetch));
```

without concurrency strategy.

---

## Anti-Pattern 5

Blind retry of:

```text
payment
order creation
resource creation
```

without idempotency semantics.

---

## Anti-Pattern 6

Using:

```text
no-cors
```

as a “fix” for CORS problems.

The resulting opaque response generally prevents normal JavaScript access to headers/body. MDN warns that `no-cors` is generally not the solution for ordinary application requests. citeturn845291search1

---

## Anti-Pattern 7

Putting tokens into URLs.

---

## Anti-Pattern 8

Manually assigning the multipart boundary for ordinary `FormData`.

---

## Anti-Pattern 9

Treating:

```text
200 OK
```

as proof the business operation succeeded.

---

## Anti-Pattern 10

Caching sensitive data without a clear privacy/freshness model.

---

# 54. Debugging Methodology

When Fetch fails, classify before changing code.

## Step 1 — Was the Promise rejected?

```js
try {
  await fetch(url);
} catch (error) {
  console.error(error);
}
```

---

## Step 2 — Did a response arrive?

```js
response.status
response.type
response.url
```

---

## Step 3 — Is it HTTP failure?

```js
response.ok
response.status
```

---

## Step 4 — Is parsing failing?

```js
await response.text()
```

temporarily inspect raw body before assuming JSON.

---

## Step 5 — Is CORS involved?

Inspect:

```text
Origin
Access-Control-Allow-Origin
Access-Control-Allow-Credentials
Access-Control-Allow-Methods
Access-Control-Allow-Headers
```

---

## Step 6 — Was there a preflight?

Inspect browser Network tools for:

```text
OPTIONS
```

---

## Step 7 — Were credentials sent?

Check:

```text
cookies
credentials mode
SameSite
Secure
HttpOnly
```

---

## Step 8 — Was it cached?

Inspect:

```text
Cache-Control
ETag
304
Age
browser Network panel
```

---

## Step 9 — Did a service worker intercept it?

Check:

```text
Application / Service Workers
```

---

## Step 10 — Did cancellation occur?

Inspect:

```text
AbortController
signal
timeout
component lifecycle
route transition
```

---

# 55. Implementation From Scratch

Build a **toy HTTP client runtime** to understand Fetch architecture.

## Stage 1 — Request Object

Implement:

```js
class ToyRequest {
  constructor(url, options = {}) {
    this.url = url;
    this.method = options.method ?? "GET";
    this.headers = new Map();
    this.body = options.body ?? null;
  }
}
```

---

## Stage 2 — Response Object

Implement:

```js
class ToyResponse {
  constructor({
    status,
    headers,
    body
  }) {
    this.status = status;
    this.headers = headers;
    this.body = body;
  }

  get ok() {
    return this.status >= 200 &&
           this.status < 300;
  }
}
```

---

## Stage 3 — Headers

Support:

```text
set
get
has
delete
```

with normalized behavior.

---

## Stage 4 — Body

Implement:

```text
text()
json()
```

and enforce single-consumption semantics.

---

## Stage 5 — Error Taxonomy

Implement:

```text
NetworkError
AbortError
TimeoutError
HttpError
ParseError
```

---

## Stage 6 — Retry Engine

Implement:

```text
retryable?
backoff
jitter
maximum attempts
```

---

## Stage 7 — Cancellation

Integrate:

```js
AbortSignal
```

into the client.

---

## Stage 8 — Concurrency

Implement:

```text
semaphore
queue
active count
```

---

## Stage 9 — Caching

Create a toy:

```text
Request → Response
```

cache with:

```text
TTL
stale
revalidate
eviction
```

---

## Stage 10 — Service Worker Simulation

Implement:

```text
request
→ interceptor
→ cache/network
→ response
```

---

## Stage 11 — Observability

Record:

```text
duration
status
attempt
request ID
cache hit
failure category
```

---

## Stage 12 — Production Client

Create:

```js
class ApiClient {
  constructor({
    baseUrl,
    timeout,
    retry,
    concurrency
  }) {
    // ...
  }

  async request(path, options) {
    // ...
  }
}
```

The implementation must separate:

```text
transport
retry
serialization
validation
logging
```

rather than creating one giant function.

---

# 56. Debugging Exercises

## Exercise 1 — 404 Does Not Automatically Reject

```js
const response = await fetch("/does-not-exist");

console.log(response.status);
```

Explain why the code can reach the next line.

---

## Exercise 2 — JSON Parse

Server returns:

```text
Content-Type: text/html

<h1>Not JSON</h1>
```

but your code executes:

```js
await response.json();
```

Identify the failure layer.

---

## Exercise 3 — CORS

Browser request:

```text
GET https://api.example.com/users
Origin: https://app.example.com
```

Server does not provide the required CORS response headers.

What can happen?

---

## Exercise 4 — Preflight

Request:

```js
fetch("https://api.example.com/users", {
  method: "PATCH",
  headers: {
    "X-Client-Version": "1"
  }
});
```

Identify why an OPTIONS request may appear first.

---

## Exercise 5 — Credentials

```js
fetch("https://api.example.com/me", {
  credentials: "include"
});
```

Explain which server-side CORS considerations matter.

---

## Exercise 6 — Redirect

```js
fetch("/old", {
  redirect: "error"
});
```

Server returns a redirect.

Predict behavior.

---

## Exercise 7 — Abort

```js
const controller = new AbortController();

setTimeout(() => controller.abort(), 100);

await fetch("/slow", {
  signal: controller.signal
});
```

What category of failure should the application recognize?

---

## Exercise 8 — Retry

An API responds:

```text
503
```

Design a retry decision.

---

## Exercise 9 — POST Retry

A payment request returns a network failure after the server may have accepted it.

Should the client automatically repeat the POST?

Explain why the answer cannot be determined from the HTTP status alone.

---

## Exercise 10 — Cache

A response contains:

```http
ETag: "abc"
```

The next request is conditional.

Explain how a `304 Not Modified` response affects representation reuse.

---

## Exercise 11 — Stream

A 2 GB response is downloaded.

Compare:

```js
await response.arrayBuffer();
```

with chunk-by-chunk streaming.

---

## Exercise 12 — Clone

A service worker needs to:

```text
cache response
+
return response
```

Why might:

```js
response.clone()
```

be required?

---

# 57. Code Review Exercise

Review:

```js
export async function request(url, options = {}) {
  for (let i = 0; i < 5; i++) {
    try {
      const response = await fetch(url, options);

      if (response.ok) {
        return await response.json();
      }
    } catch {
      // retry
    }
  }

  return null;
}
```

The developer says:

> “This handles failures and retries everything automatically.”

It is not production-safe.

## Problems

1. HTTP failures are not categorized.
2. It retries blindly.
3. `POST` may be repeated.
4. No exponential backoff.
5. No jitter.
6. No timeout.
7. No cancellation semantics.
8. Errors disappear.
9. Returns `null` for unrelated failure modes.
10. Assumes every success response is JSON.
11. Does not preserve status/body/request metadata.
12. No schema validation.
13. No concurrency control.
14. No observability.
15. No authentication policy.
16. No cache policy.
17. No idempotency policy.
18. No distinction between abort and transient network failure.

A stronger architecture should produce:

```text
Request
 ↓
deadline/cancellation
 ↓
fetch
 ↓
response classification
 ↓
HTTP semantics
 ↓
parse
 ↓
validate
 ↓
retry decision
 ↓
error taxonomy
 ↓
observability
```

---

# 58. Interview Questions

## Fundamentals

1. What is Fetch?
2. Is Fetch part of ECMAScript?
3. What does `fetch()` return?
4. Does `fetch()` reject on 404?
5. What is a Request?
6. What is a Response?
7. What is Headers?
8. What is a response body?

## HTTP

9. GET vs POST?
10. PUT vs PATCH?
11. What is idempotency?
12. Why does idempotency matter for retries?
13. What does 304 mean?
14. What is ETag?
15. What is Cache-Control?
16. What is content negotiation?

## CORS

17. What is CORS?
18. What is the same-origin policy?
19. What is a preflight?
20. Why does OPTIONS appear?
21. What does Access-Control-Allow-Origin do?
22. Why can `*` not be used in the credentialed sharing case?
23. Does CORS prevent every cross-origin request from being sent?

## Credentials

24. What does `credentials: "include"` mean?
25. What are browser credentials?
26. How do SameSite cookies affect credentials?
27. Why does credentialed cross-origin access require careful CSRF design?

## Abort/Streaming

28. How does AbortController work with Fetch?
29. How would you implement a timeout?
30. How do you stream a response?
31. How do you implement upload streaming?
32. Why can streaming reduce memory use?

## Production

33. When should you retry?
34. Why use exponential backoff?
35. Why use jitter?
36. How would you limit concurrent requests?
37. How should network errors differ from HTTP errors?
38. How do you validate API responses?
39. How do you design a Fetch wrapper?
40. How should a client log network failures safely?

## Principal-Level

41. Design a multi-tenant browser API client.
42. How would you handle tenant-specific authentication?
43. How would you prevent retry storms?
44. How would you design idempotency for mutation endpoints?
45. How would you handle offline/reconnect behavior?
46. How would you design cache invalidation?
47. How would you instrument request latency end-to-end?
48. How would you debug a CORS issue that works in curl but fails in the browser?
49. How would you design a resilient frontend networking layer for 100,000 daily users?
50. When is Fetch the wrong abstraction?

---

# 59. Predict-the-Output Exercises

## Exercise 1

```js
const response = await fetch("/missing");

console.log(response instanceof Response);
console.log(response.ok);
console.log(response.status);
```

Predict the conceptual result when the server returns:

```text
404
```

---

## Exercise 2

```js
const response = await fetch("/ok");

const a = await response.text();
const b = await response.text();
```

Will both reads behave like independent ordinary string reads?

Explain body consumption.

---

## Exercise 3

```js
const response = await fetch("/ok", {
  redirect: "error"
});
```

Server returns a redirect.

Predict the outcome.

---

## Exercise 4

```js
const controller = new AbortController();

controller.abort();

fetch("/api", {
  signal: controller.signal
});
```

Predict the Promise behavior.

---

## Exercise 5

```js
const response = await fetch("/api");

console.log(response.ok);
console.log(response.status >= 200 &&
            response.status < 300);
```

Explain why these expressions correspond conceptually.

---

## Exercise 6

```js
await fetch("https://other.example", {
  mode: "no-cors"
});
```

What kind of response visibility should JavaScript expect?

---

## Exercise 7

```js
await fetch("/api", {
  credentials: "omit"
});
```

Explain what happens to credential inclusion.

---

## Exercise 8

```js
await fetch("/api", {
  credentials: "include"
});
```

What additional security considerations appear on cross-origin requests?

---

# 60. Mastery Exercises

## Level 1 — Basic Client

Build:

```text
GET JSON client
```

with:

```text
HTTP status handling
JSON parsing
errors
```

---

## Level 2 — Abortable Client

Add:

```text
AbortController
```

and caller cancellation.

---

## Level 3 — Timeout

Implement:

```text
request deadline
```

without hiding caller cancellation.

---

## Level 4 — Error Taxonomy

Create:

```text
HttpError
NetworkError
TimeoutError
AbortError
ParseError
ValidationError
```

---

## Level 5 — Retry Engine

Support:

```text
retryable status
attempt limit
backoff
jitter
```

---

## Level 6 — Idempotency

Design a mutation client supporting:

```text
Idempotency-Key
```

where the API contract supports it.

---

## Level 7 — Concurrency

Build:

```text
20,000 URLs
```

with:

```text
max 10 concurrent
```

and cancellation.

---

## Level 8 — Streaming

Download a large file and display:

```text
bytes received
processing time
throughput
```

using a stream.

---

## Level 9 — Cache Layer

Build a cache-first/network-first strategy using the Cache API.

---

## Level 10 — Service Worker

Implement:

```text
offline-first
network-first
stale-while-revalidate
```

and compare them.

---

## Level 11 — Production API Client

Support:

```text
base URL
headers
authentication
timeout
abort
retry
backoff
validation
logging
request ID
```

---

## Level 12 — Principal Networking System

Design a browser network layer with:

```text
multi-tenant API
authentication rotation
request deduplication
concurrency limits
retry policies
idempotency
offline queue
cache
observability
feature flags
schema validation
security policies
```

Defend every decision.

---

# 61. Key Takeaways

1. Fetch is a Web Platform API, not an ECMAScript language feature.
2. `fetch()` returns a Promise for a `Response`.
3. HTTP 4xx/5xx responses do not automatically mean the Fetch Promise rejects.
4. Request, Response, Headers, and Body are core Fetch abstractions.
5. Bodies are stream-oriented and generally consumable.
6. `response.json()` is application parsing layered on top of the body.
7. HTTP status and application success are separate concepts.
8. HTTP methods have different semantic and idempotency properties.
9. Retry policy must account for operation semantics.
10. CORS is a browser-enforced cross-origin response-sharing mechanism.
11. CORS is not the same thing as server reachability.
12. Preflight uses OPTIONS for requests requiring CORS permission checks.
13. Credentials have explicit Fetch modes.
14. Credentialed cross-origin requests require careful server and CSRF configuration.
15. Redirect behavior is configurable with Fetch redirect modes.
16. HTTP caching is distinct from the Cache API.
17. AbortController is the correct cancellation mechanism for Fetch.
18. Timeouts should be modeled as explicit deadlines/cancellation.
19. Large responses can benefit from streaming.
20. Request streaming has additional compatibility and protocol constraints.
21. `keepalive` has constrained use cases.
22. Network failures, HTTP failures, parse failures, validation failures, and application failures should be classified separately.
23. Retries need backoff and jitter.
24. Large request sets need concurrency control.
25. Modern browsers abstract connection pooling/protocol details away from most application code.
26. HTTPS provides transport security but does not replace application authorization.
27. Service workers can intercept and cache Fetch requests.
28. A production API client should preserve, not hide, important Fetch semantics.
29. All remote data should be treated as untrusted input.
30. Observability must avoid leaking credentials or sensitive payloads.
31. Cache design is a correctness decision, not merely a performance decision.
32. `no-cors` is not a generic fix for CORS errors.
33. A robust networking layer is an orchestration system around Fetch, not merely a wrapper function.

---

# 62. Concept Connections

## Depends On

```text
Chapter 31 — Async Fundamentals
        ↓
Chapter 32 — ECMAScript Jobs / Promise Reactions
        ↓
Chapter 35 — Promises
        ↓
Chapter 36 — Async / Await
        ↓
Chapter 37 — Cancellation / Abort
        ↓
Chapter 38 — Async Iteration / Streaming
        ↓
Chapter 49 — DOM Architecture
        ↓
Chapter 50 — Browser Events
        ↓
Chapter 51 — Browser APIs
        ↓
Chapter 53 — Web Streams
        ↓
Chapter 55 — Fetch / HTTP Networking
```

## Builds Toward

```text
Chapter 56 — Browser Security
Chapter 57 — JS Security Engineering
Chapter 58 — Node Architecture
Chapter 59 — Node Core APIs
Chapter 60 — Node Streams
Chapter 61 — Worker Threads / Processes
Chapter 62 — Process Lifecycle
Chapter 63 — Async Context / Diagnostics
Chapter 78 — Production Architecture
Chapter 79 — API Design
Chapter 82 — API Architecture
Chapter 83 — Observability
Chapter 84 — Reliability
Chapter 85 — Performance
```

## Related Concepts

```text
HTTP
HTTPS
CORS
same-origin policy
cookies
CSRF
TLS
streams
service workers
Cache API
WebSocket
WebTransport
CDN
reverse proxy
API gateway
load balancer
```

## Concepts Revisited

### Chapter 37 — Cancellation

Fetch demonstrates why cancellation must be propagated through asynchronous work.

### Chapter 38 — Streaming

Fetch response bodies expose browser streams.

### Chapter 49 — DOM

Browser networking eventually feeds UI rendering and application state.

### Chapter 50 — Events

Networking often produces application events and component lifecycle interactions.

### Chapter 53 — Streams

Fetch body streams connect networking directly to browser stream processing.

---

## Why This Chapter Matters Later

This chapter is the bridge between browser JavaScript and real distributed systems.

The fundamental model becomes:

```text
browser
 ↓
request
 ↓
security policies
 ↓
network
 ↓
server
 ↓
response
 ↓
stream
 ↓
parser
 ↓
validation
 ↓
application state
```

At production scale, the difficult problem is not:

```js
fetch(url)
```

It is:

```text
correctness
+
security
+
cancellation
+
reliability
+
caching
+
observability
+
performance
+
distributed-system failure
```

---

# 63. Completion Criteria

## Fundamentals

- [ ] Explain Fetch.
- [ ] Distinguish Fetch from ECMAScript.
- [ ] Explain Request.
- [ ] Explain Response.
- [ ] Explain Headers.
- [ ] Explain request/response bodies.
- [ ] Explain body consumption.

## HTTP

- [ ] Explain methods.
- [ ] Explain status codes.
- [ ] Explain idempotency.
- [ ] Explain content types.
- [ ] Explain serialization.
- [ ] Explain conditional requests.
- [ ] Explain ETag.
- [ ] Explain 304.

## CORS

- [ ] Explain origin.
- [ ] Explain same-origin policy.
- [ ] Explain CORS.
- [ ] Explain preflight.
- [ ] Explain OPTIONS.
- [ ] Explain Access-Control-Allow-Origin.
- [ ] Explain credentialed CORS.

## Credentials

- [ ] Explain omit.
- [ ] Explain same-origin.
- [ ] Explain include.
- [ ] Explain cookies.
- [ ] Explain SameSite.
- [ ] Explain CSRF implications.

## Redirects / Referrer

- [ ] Explain follow.
- [ ] Explain error.
- [ ] Explain manual.
- [ ] Explain referrer policy.

## Caching

- [ ] Explain HTTP cache.
- [ ] Explain Cache-Control.
- [ ] Explain ETag.
- [ ] Explain revalidation.
- [ ] Explain Fetch cache modes.
- [ ] Explain Cache API.

## Cancellation

- [ ] Use AbortController.
- [ ] Use AbortSignal.
- [ ] Implement deadlines.
- [ ] Combine caller cancellation with timeout.

## Streaming

- [ ] Read response body as a stream.
- [ ] Process chunks.
- [ ] Explain memory implications.
- [ ] Explain request streaming.
- [ ] Explain `duplex` conceptually.

## Reliability

- [ ] Classify failures.
- [ ] Implement retry.
- [ ] Implement backoff.
- [ ] Implement jitter.
- [ ] Handle rate limits.
- [ ] Handle idempotency.
- [ ] Bound concurrency.

## Security

- [ ] Explain HTTPS.
- [ ] Explain CORS boundaries.
- [ ] Explain CSRF.
- [ ] Protect credentials.
- [ ] Validate network data.
- [ ] Avoid URL secrets.
- [ ] Avoid sensitive logging.

## Architecture

- [ ] Design Fetch wrapper.
- [ ] Design API client.
- [ ] Add observability.
- [ ] Add schema validation.
- [ ] Add caching policy.
- [ ] Add retry policy.
- [ ] Add cancellation.
- [ ] Add concurrency control.

## Principal Judgment

- [ ] Explain Fetch vs XHR.
- [ ] Explain Fetch vs WebSocket.
- [ ] Explain Fetch vs WebTransport.
- [ ] Defend retry policy.
- [ ] Defend cache policy.
- [ ] Defend authentication model.
- [ ] Defend concurrency limits.
- [ ] Defend streaming decisions.
- [ ] Defend browser security assumptions.
- [ ] Defend a production networking architecture.

---

# 64. Revision / Retrieval Record

## First-Pass Retrieval

Without opening this chapter:

1. What is Fetch?
2. Is Fetch ECMAScript?
3. What does fetch() return?
4. Does HTTP 404 reject?
5. What is Request?
6. What is Response?
7. What is Headers?
8. How is a body consumed?
9. Why can't a response body be treated like an ordinary reusable value?
10. What does `response.ok` mean?
11. What is idempotency?
12. Why does idempotency matter for retries?
13. What is CORS?
14. What is a preflight?
15. Why is OPTIONS used?
16. What does `credentials: "include"` mean?
17. Why is credentialed CORS security-sensitive?
18. What is the same-origin policy?
19. What are redirect modes?
20. What is ETag?
21. What is 304?
22. HTTP cache vs Cache API?
23. What does AbortController do?
24. How do you implement a timeout?
25. How do you stream a response?
26. What is `duplex`?
27. What does `keepalive` do?
28. What is a network error?
29. What is an HTTP error?
30. What is a parse error?
31. Why should retries use backoff?
32. Why add jitter?
33. Why limit concurrency?
34. How does a service worker intercept Fetch?
35. Why isn't `no-cors` a generic CORS fix?
36. How would you design a production API client?

## Retrieval Table

| Concept | Can Explain? | Can Predict? | Can Implement? | Needs Revision? |
|---|---:|---:|---:|---:|
| Fetch | [ ] | [ ] | [ ] | [ ] |
| Request | [ ] | [ ] | [ ] | [ ] |
| Response | [ ] | [ ] | [ ] | [ ] |
| Headers | [ ] | [ ] | [ ] | [ ] |
| Bodies | [ ] | [ ] | [ ] | [ ] |
| Streams | [ ] | [ ] | [ ] | [ ] |
| HTTP status | [ ] | [ ] | [ ] | [ ] |
| Idempotency | [ ] | [ ] | [ ] | [ ] |
| Serialization | [ ] | [ ] | [ ] | [ ] |
| Same-origin policy | [ ] | [ ] | [ ] | [ ] |
| CORS | [ ] | [ ] | [ ] | [ ] |
| Preflight | [ ] | [ ] | [ ] | [ ] |
| Credentials | [ ] | [ ] | [ ] | [ ] |
| Cookies | [ ] | [ ] | [ ] | [ ] |
| Redirects | [ ] | [ ] | [ ] | [ ] |
| Referrer | [ ] | [ ] | [ ] | [ ] |
| HTTP cache | [ ] | [ ] | [ ] | [ ] |
| Cache API | [ ] | [ ] | [ ] | [ ] |
| AbortController | [ ] | [ ] | [ ] | [ ] |
| Timeouts | [ ] | [ ] | [ ] | [ ] |
| Uploads | [ ] | [ ] | [ ] | [ ] |
| Download streaming | [ ] | [ ] | [ ] | [ ] |
| Request streaming | [ ] | [ ] | [ ] | [ ] |
| keepalive | [ ] | [ ] | [ ] | [ ] |
| Network errors | [ ] | [ ] | [ ] | [ ] |
| Retry | [ ] | [ ] | [ ] | [ ] |
| Backoff/jitter | [ ] | [ ] | [ ] | [ ] |
| Concurrency | [ ] | [ ] | [ ] | [ ] |
| TLS/HTTPS | [ ] | [ ] | [ ] | [ ] |
| Service workers | [ ] | [ ] | [ ] | [ ] |
| Authentication | [ ] | [ ] | [ ] | [ ] |
| CSRF | [ ] | [ ] | [ ] | [ ] |
| Validation | [ ] | [ ] | [ ] | [ ] |
| Observability | [ ] | [ ] | [ ] | [ ] |
| Performance | [ ] | [ ] | [ ] | [ ] |
| Memory | [ ] | [ ] | [ ] | [ ] |
| Production API client | [ ] | [ ] | [ ] | [ ] |

## Spaced Retrieval Schedule

### Day 0

Draw:

```text
fetch
 ↓
Request
 ↓
security
 ↓
cache / service worker / network
 ↓
Response
 ↓
body stream
 ↓
parse
 ↓
validate
```

### Day 2

Explain why:

```text
404
```

does not necessarily reject Fetch.

### Day 7

Build an abortable API client with:

```text
timeout
retry
status handling
```

### Day 14

Debug a CORS + credentials + preflight scenario.

### Day 30

Design a production browser networking subsystem supporting:

```text
multi-tenant auth
cache
retry
backoff
concurrency
offline
observability
validation
security
```

---

# 65. Canonical References and Source Discipline

## Primary Fetch Standard

### WHATWG Fetch Standard

https://fetch.spec.whatwg.org/

Use for:

```text
Request
Response
Headers
body
fetch algorithm
CORS
credentials
redirects
cache modes
network errors
request modes
abort
keepalive
duplex
```

The current Fetch Standard defines request modes such as `same-origin`, `cors`, and `no-cors`; credentials modes `omit`, `same-origin`, and `include`; cache modes; redirect modes; and the CORS protocol. citeturn845291search0

---

## MDN Fetch API

https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API

Use for browser-facing API reference and examples. MDN documents Fetch as a Web API exposed in Window and Worker contexts. citeturn845291search3

---

## MDN Using Fetch

https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API/Using_Fetch

Use for:

```text
cross-origin requests
credentials
request construction
response handling
```

MDN documents current Fetch behavior for CORS modes and credentials and explains the security implications of credentialed cross-origin requests. citeturn845291search1

---

## MDN Cache

https://developer.mozilla.org/en-US/docs/Web/API/Cache

Use for:

```text
Cache API
Request/Response storage
service-worker caching
```

MDN explains that `Cache` stores `Request`/`Response` pairs and that applications are responsible for how those caches are populated and updated. citeturn845291search2

---

## HTTP Specifications

### RFC 9110 — HTTP Semantics

Use for:

```text
methods
status codes
header semantics
idempotency
safe methods
caching semantics
```

### RFC 9111 — HTTP Caching

Use for:

```text
cache-control
freshness
validation
revalidation
```

### RFC 9112 — HTTP/1.1

Use when protocol-level HTTP/1.1 details are required.

### RFC 9114 — HTTP/3

Use for HTTP/3-specific transport semantics.

### RFC 9000 — QUIC

Use for QUIC transport details.

---

## TLS

### RFC 8446 — TLS 1.3

Use for TLS protocol details.

Do not infer browser API behavior directly from low-level TLS specifications.

---

## Source Classification

Classify claims as:

```text
[Fetch Standard]
[HTTP Semantics]
[HTTP Caching]
[Browser API]
[Browser Security]
[Measured]
[Browser-specific]
[Historical]
```

Examples:

```text
"fetch() resolves with a Response for an HTTP 404"
→ [Fetch API behavior]

"GET is safe/idempotent under HTTP semantics"
→ [HTTP Semantics]

"Firefox supports a particular advanced streaming feature"
→ [Browser-specific]

"request latency is 210 ms"
→ [Measured]
```

---

## Security Source Discipline

Do not describe:

```text
CORS
```

as:

```text
a server firewall
```

Do not describe:

```text
same-origin policy
```

as:

```text
encryption
```

Do not describe:

```text
credentials: include
```

as:

```text
automatic secure authentication
```

Each has a different role.

---

## Compatibility Discipline

For advanced APIs such as:

```text
AbortSignal.timeout
streaming request bodies
duplex
keepalive
Cache API
service-worker interactions
```

check the actual browser/runtime compatibility matrix before relying on them as universal guarantees.

---

# 66. Completion Snapshot

## Chapter Status

```text
[ ] Not Started
[~] In Progress
[?] Needs Revision
[+] Completed
[*] Mastered
```

Current status:

```text
[ ] Not Started
```

## Knowledge Snapshot

### I can explain

- [ ] Fetch
- [ ] ECMAScript vs Web Platform
- [ ] Request
- [ ] Response
- [ ] Headers
- [ ] Body
- [ ] Streams
- [ ] HTTP
- [ ] status codes
- [ ] methods
- [ ] idempotency
- [ ] serialization
- [ ] same-origin policy
- [ ] CORS
- [ ] preflight
- [ ] credentials
- [ ] cookies
- [ ] redirects
- [ ] referrer policy
- [ ] HTTP cache
- [ ] Cache API
- [ ] AbortController
- [ ] timeouts
- [ ] uploads
- [ ] response streaming
- [ ] request streaming
- [ ] keepalive
- [ ] network failures
- [ ] retry
- [ ] backoff
- [ ] jitter
- [ ] concurrency
- [ ] TLS/HTTPS
- [ ] service workers
- [ ] CSRF
- [ ] authentication
- [ ] authorization
- [ ] validation
- [ ] observability
- [ ] performance
- [ ] memory
- [ ] security
- [ ] production API architecture

### I can predict

- [ ] 404 Fetch behavior
- [ ] HTTP vs network rejection
- [ ] body consumption
- [ ] abort
- [ ] redirect modes
- [ ] CORS failures
- [ ] preflight behavior
- [ ] credentials behavior
- [ ] cache/revalidation behavior
- [ ] retry behavior
- [ ] stream behavior
- [ ] concurrency effects

### I can implement

- [ ] JSON client
- [ ] status handling
- [ ] error taxonomy
- [ ] AbortController integration
- [ ] timeout
- [ ] retry
- [ ] backoff
- [ ] jitter
- [ ] concurrency limit
- [ ] streaming download
- [ ] streaming upload
- [ ] Cache API strategy
- [ ] service worker strategy
- [ ] schema validation
- [ ] observability
- [ ] production API client

### I can debug

- [ ] network rejection
- [ ] HTTP errors
- [ ] CORS
- [ ] preflight
- [ ] credential issues
- [ ] cookie behavior
- [ ] redirects
- [ ] cache problems
- [ ] stale data
- [ ] abort/timeout
- [ ] parsing failures
- [ ] validation failures
- [ ] retry storms
- [ ] concurrency overload
- [ ] service worker interception

### I can defend

- [ ] Fetch vs XHR
- [ ] Fetch vs WebSocket
- [ ] Fetch vs WebTransport
- [ ] cache policy
- [ ] retry policy
- [ ] timeout policy
- [ ] cancellation strategy
- [ ] idempotency strategy
- [ ] authentication model
- [ ] concurrency limits
- [ ] streaming strategy
- [ ] observability design
- [ ] security model

---

## Final Principal-Level Test

Explain this system without notes:

```text
Browser application
      ↓
API client
      ↓
Request
      ↓
Abort / deadline
      ↓
Fetch
      ↓
same-origin / CORS / credentials / security
      ↓
service worker / cache
      ↓
HTTP
      ↓
TLS / transport
      ↓
server
      ↓
HTTP response
      ↓
redirect / cache / CORS filtering
      ↓
Response
      ↓
ReadableStream
      ↓
parse
      ↓
schema validation
      ↓
domain model
      ↓
UI
```

Then answer:

```text
What can fail at every layer?
What should be retried?
What should be canceled?
What should be cached?
What should never be logged?
What requires user-visible feedback?
What requires server-side enforcement?
What can cause memory pressure?
What can cause a retry storm?
What can leak credentials?
What can become stale?
What must be observable?
```

Your mastery is complete only when you can reason across the entire chain rather than treating:

```js
fetch(url)
```

as a black box.

The central Chapter 55 lesson is:

> **Production networking is not “calling an API.” It is controlling a distributed, security-sensitive, failure-prone asynchronous system through an explicit request/response contract.**