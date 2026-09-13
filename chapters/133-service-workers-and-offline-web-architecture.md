# Chapter 133 — Service Workers & Offline Web Architecture

> **JavaScript Mastery — Part XXIII: Browser Platform & Client State**
>
> **Mission:** Master service workers as a browser platform primitive for programmable networking, offline behavior, caching, background events, lifecycle coordination, update delivery, resilience, and progressive web application architecture.
>
> **Role perspective:** Principal JavaScript Engineer · Browser Platform Engineer · PWA Architect · Networking Engineer · Reliability Engineer · Security Engineer · Performance Engineer
>
> **Status:** `[ ] Not Started`
>
> **Core rule:** **A service worker is not “a background JavaScript file.” It is a lifecycle-managed programmable network and event boundary. Correct architecture depends on understanding registration scope, installation, activation, control, fetch interception, cache ownership, update safety, and the limits of background execution.**

---

# 1. Learning Objectives

By completing this chapter, you should be able to:

```text
[ ] explain what a service worker is
[ ] explain why service workers exist
[ ] distinguish service workers from web workers
[ ] explain ServiceWorkerContainer
[ ] explain ServiceWorkerRegistration
[ ] explain service worker scope
[ ] explain secure-context requirements
[ ] explain registration
[ ] explain installation
[ ] explain waiting
[ ] explain activation
[ ] explain controlling clients
[ ] explain fetch events
[ ] explain extendable events
[ ] explain event.waitUntil()
[ ] explain event.respondWith()
[ ] explain clients.claim()
[ ] explain skipWaiting()
[ ] understand why skipWaiting can be dangerous
[ ] understand service worker update lifecycle
[ ] understand registration.update()
[ ] understand byte-level update checks conceptually
[ ] explain updateViaCache
[ ] explain controllerchange
[ ] explain updatefound
[ ] explain ServiceWorker.state
[ ] explain active/installing/waiting workers
[ ] explain Cache Storage
[ ] distinguish browser HTTP cache from Cache Storage
[ ] design cache-first strategy
[ ] design network-first strategy
[ ] design stale-while-revalidate
[ ] design network-only
[ ] design cache-only
[ ] design offline fallback
[ ] explain precaching
[ ] explain runtime caching
[ ] explain cache versioning
[ ] explain cache cleanup
[ ] explain cache invalidation
[ ] explain navigation requests
[ ] explain navigation preload
[ ] explain service worker startup latency
[ ] explain navigation preload headers
[ ] explain background sync conceptually
[ ] understand periodic background sync limitations
[ ] explain push events at a high level
[ ] distinguish background capability from guaranteed execution
[ ] explain service-worker lifetime
[ ] explain worker termination/restart
[ ] explain stateless handler design
[ ] understand service worker persistence
[ ] explain service worker messaging
[ ] use postMessage
[ ] understand Client objects
[ ] explain BroadcastChannel relationships
[ ] coordinate app and worker updates
[ ] design safe application updates
[ ] design rollback strategy
[ ] handle partial cache updates
[ ] protect cache integrity
[ ] defend against cache poisoning
[ ] reason about authenticated responses
[ ] reason about opaque responses
[ ] understand CORS and fetch interaction
[ ] explain request/response cloning
[ ] explain body consumption
[ ] explain service-worker fetch boundaries
[ ] understand scope restrictions
[ ] understand Service-Worker-Allowed
[ ] explain service-worker security
[ ] prevent service-worker takeover
[ ] design offline-first architecture
[ ] design online-first architecture
[ ] design resilient navigation fallback
[ ] design offline mutation queues
[ ] coordinate IndexedDB and Cache Storage
[ ] design storage cleanup
[ ] test worker lifecycle
[ ] test multiple tabs
[ ] test updates
[ ] test offline behavior
[ ] test network failures
[ ] debug stale workers
[ ] debug stale caches
[ ] choose when not to use a service worker
```

---

# 2. Prerequisites

You should already understand:

```text
Chapter 31 — Async Fundamentals
Chapter 32 — ECMAScript Jobs / Promise Reactions
Chapter 33 — Browser Event Loop
Chapter 35 — Promises
Chapter 36 — Async/Await
Chapter 37 — Cancellation / Abort
Chapter 38 — Async Iteration / Streaming
Chapter 49 — DOM Architecture
Chapter 51 — Browser Web APIs
Chapter 52 — Web Workers / Concurrency
Chapter 53 — Web Streams / Data Flow
Chapter 55 — Fetch / HTTP Networking
Chapter 56 — Browser Security
Chapter 57 — JavaScript Security Engineering
Chapter 83 — Observability
Chapter 84 — Reliability
Chapter 85 — Performance
Chapter 86 — Testing
Chapter 101 — Real-World Production Scenarios
Chapter 109 — Event-Driven Applications
Chapter 126 — Module Linking / Resolution
Chapter 132 — Browser Storage Architecture
```

Supporting concepts:

```text
HTTP caching
cookies
origins
Request / Response
Cache API
IndexedDB
network failures
distributed state
```

---

# 3. What Is a Service Worker?

A service worker is a worker-context script that can participate in browser events and, importantly, can intercept and respond to eligible fetches made in its controlled scope.

It runs:

```text
off the main thread
```

and has:

```text
no DOM access
```

A registration has a lifecycle independent of a single page's JavaScript lifetime. citeturn531510search0turn531510search4

---

# 4. Why Service Workers Exist

Before service workers:

```text
page
 ↓
network
 ↓
resource
```

With a service worker:

```text
page
 ↓
service worker
 ├── cache
 ├── network
 ├── fallback
 └── custom strategy
```

This enables:

```text
offline applications
programmable caching
network resilience
background work
push
installable PWAs
```

---

# 5. Service Worker vs Web Worker

| Feature | Service Worker | Dedicated Web Worker |
|---|---|---|
| DOM access | No | No |
| Handles fetches | Yes, in scope | No |
| Persistent registration | Yes | No |
| Page-independent lifecycle | Yes | Usually no |
| Multiple clients | Yes | Typically associated worker context |
| Offline architecture | Yes | Not directly |
| `import()` | restricted by service-worker environment | supported according to worker/module rules |

MDN documents that service workers can use static ECMAScript module imports where supported, while dynamic `import()` is disallowed by the service-worker specification. citeturn531510search1

---

# 6. Secure Context

Service workers require a secure context in normal production use.

Typically:

```text
https://
```

is required.

Browsers also treat:

```text
localhost
```

as a secure development origin for service-worker use. citeturn531510search0

---

# 7. Basic Architecture

```text
                 Browser
                    |
          ServiceWorkerContainer
                    |
          ServiceWorkerRegistration
                    |
          +---------+---------+
          |         |         |
      installing  waiting   active
                              |
                            clients
                              |
                         fetch/events
```

The worker can serve:

```text
one page
many tabs
other controlled clients
```

within its registration scope.

---

# 8. Registration

Typical page code:

```js
if ("serviceWorker" in navigator) {
  await navigator.serviceWorker.register("/sw.js");
}
```

This establishes or obtains a registration for a script and scope.

The browser then manages:

```text
installation
activation
control
updates
```

---

# 9. Registration Scope

A registration has:

```js
registration.scope
```

The scope defines which client URLs the service worker can control.

Example:

```text
/sw.js
```

normally controls URLs under:

```text
/
```

when registered from an appropriate location.

---

# 10. Scope Is a Security Boundary

A service worker cannot simply control arbitrary pages on another origin.

Its scope is constrained by:

```text
origin
script location
scope
```

and the browser's service-worker rules.

This prevents a page from casually acquiring broad control over unrelated sites.

---

# 11. Service-Worker-Allowed

Under certain deployment structures, the server can provide:

```text
Service-Worker-Allowed
```

to permit a broader scope than the script's directory would normally allow, subject to the standard's rules.

This should be treated as:

```text
explicit deployment configuration
```

not a casual workaround.

---

# 12. Installation Lifecycle

Conceptual sequence:

```text
register
  ↓
fetch worker script
  ↓
install
  ↓
installed / waiting
  ↓
activate
  ↓
activated
  ↓
control clients
```

The new worker generally does not immediately replace an active worker controlling existing pages. citeturn531510search0

---

# 13. `install` Event

The install event is the first lifecycle event sent to a new service worker.

Typical uses:

```text
precache shell assets
initialize IndexedDB schema
prepare static resources
```

Use:

```js
self.addEventListener("install", event => {
  event.waitUntil(prepare());
});
```

MDN documents `install` as the stage commonly used to populate offline resources. citeturn531510search0

---

# 14. Why `waitUntil()` Matters

Bad:

```js
self.addEventListener("install", () => {
  caches.open("app-v1").then(...);
});
```

The browser is not guaranteed to keep the install lifecycle tied to that arbitrary Promise unless the event is extended.

Correct:

```js
self.addEventListener("install", event => {
  event.waitUntil(
    caches.open("app-v1")
  );
});
```

The event's lifetime is extended by:

```text
waitUntil()
```

---

# 15. Extendable Events

Service-worker lifecycle events are special event types capable of having their lifetime extended.

Typical examples:

```text
install
activate
sync
push
```

Use:

```js
event.waitUntil(promise)
```

to bind asynchronous work to event completion.

---

# 16. Activation

Once the previous active worker is no longer controlling pages—or the waiting worker is otherwise allowed to activate—the new worker receives:

```text
activate
```

The main use is often:

```text
cleanup old caches
migrate state
prepare active version
```

MDN describes activation as the phase where older resources are commonly cleaned. citeturn531510search0

---

# 17. Waiting State

A newly installed worker can enter:

```text
waiting
```

while an older worker remains active.

This creates an important invariant:

```text
old clients
→ old worker

new worker
→ waiting
```

This helps prevent:

```text
old page shell
+
new worker behavior
```

mismatches.

---

# 18. `skipWaiting()`

A new worker can request immediate progression from waiting toward activation using:

```js
self.skipWaiting();
```

But this can be dangerous.

It can create:

```text
new worker
+
old application page
```

coexistence.

Therefore:

```text
skipWaiting
```

is not automatically:

```text
best practice.
```

---

# 19. Safe Update Strategy

A safer strategy is:

```text
install new worker
→ verify assets
→ wait
→ notify clients
→ user chooses update
→ activate new worker
→ reload/control atomically
```

This keeps:

```text
application code
+
service worker
+
cache
```

more aligned.

---

# 20. `clients.claim()`

After activation, a worker normally controls clients according to service-worker lifecycle rules.

A worker can call:

```js
self.clients.claim();
```

to attempt to take control of matching clients immediately.

MDN notes that without claiming, already-open documents normally need reloads before they become controlled by the new worker. citeturn531510search0

---

# 21. `skipWaiting()` + `clients.claim()`

These often appear together:

```js
self.addEventListener("install", () => {
  self.skipWaiting();
});

self.addEventListener("activate", event => {
  event.waitUntil(self.clients.claim());
});
```

This provides rapid takeover.

But the architectural danger is:

```text
old page assumptions
+
new worker behavior
```

without a synchronized app update.

Use only with a compatibility strategy.

---

# 22. Service Worker State Machine

A worker may move through states such as:

```text
parsed
installing
installed
activating
activated
redundant
```

The browser exposes state through:

```js
serviceWorker.state
```

and:

```text
statechange
```

events. citeturn531510search1

---

# 23. `ServiceWorkerRegistration`

A registration can expose:

```js
registration.installing
registration.waiting
registration.active
```

These identify different lifecycle participants.

MDN documents `installing`, `waiting`, and `active` as lifecycle-related registration properties. citeturn531510search1turn531510search14

---

# 24. `updatefound`

The registration can emit:

```text
updatefound
```

when a new worker becomes:

```text
installing
```

This lets the application observe:

```text
new worker discovered
```

and show controlled update UX. citeturn531510search4

---

# 25. `controllerchange`

Pages can listen for:

```js
navigator.serviceWorker.addEventListener(
  "controllerchange",
  () => {
    // active controller changed
  }
);
```

This is useful for:

```text
reload coordination
update notifications
state reconciliation
```

---

# 26. Fetch Events

A controlled page's eligible requests can generate:

```text
FetchEvent
```

in the service worker.

Example:

```js
self.addEventListener("fetch", event => {
  event.respondWith(handle(event.request));
});
```

This turns the worker into:

```text
programmable network middleware.
```

---

# 27. `respondWith()`

`respondWith()` supplies the response promise for the fetch event.

Conceptually:

```text
browser request
→ service worker
→ respondWith(...)
→ Response
```

Without a custom response, the browser's default fetch behavior can continue.

---

# 28. Fetch Handler Is a Trust Boundary

Once a service worker controls a page:

```text
every intercepted request
```

can potentially be changed by worker logic.

Therefore a service-worker bug can affect:

```text
navigation
API requests
assets
authentication flows
```

for all controlled clients within scope.

---

# 29. Keep Fetch Handlers Small

Bad design:

```text
fetch event
→ huge application framework
→ many unrelated async operations
```

Better:

```text
fetch
→ classify request
→ choose strategy
→ perform bounded operation
→ return response
```

The service worker is:

```text
platform boundary
```

not a second entire backend.

---

# 30. Request Classification

A good fetch strategy starts by distinguishing:

```text
navigation
script
style
image
font
API
POST mutation
analytics
third-party
```

Do not apply:

```text
one cache strategy to every request.
```

---

# 31. Navigation Requests

A navigation request loads a document.

Often:

```js
event.request.mode === "navigate"
```

This is a strong signal for:

```text
offline document fallback
network-first navigation
shell routing
navigation preload
```

---

# 32. Offline Fallback

Typical strategy:

```text
navigation
 ↓
network
 ├── success → response
 └── failure → cached offline page
```

Implementation:

```js
event.respondWith(
  fetch(event.request)
    .catch(() => caches.match("/offline.html"))
);
```

Production implementations need:

```text
timeouts
cache validation
navigation-specific behavior
```

where required.

---

# 33. Cache-First

Best for:

```text
versioned static assets
```

Flow:

```text
request
 ↓
cache
 ├── hit → return
 └── miss → network → cache → return
```

Advantages:

```text
fast
offline-friendly
```

Costs:

```text
stale data
cache invalidation complexity
```

---

# 34. Network-First

Best for:

```text
fresh navigations
dynamic content
```

Flow:

```text
request
 ↓
network
 ├── success → cache optional → return
 └── failure → cache fallback
```

Advantages:

```text
freshness
```

Costs:

```text
offline failure latency
network dependency
```

---

# 35. Stale-While-Revalidate

Flow:

```text
cache hit
 ↓
return cached
 +
refresh network
 ↓
update cache
```

This is good for:

```text
content
catalogs
semi-fresh APIs
```

provided:

```text
staleness
```

is acceptable.

MDN describes caching strategies as algorithms for when to cache, when to serve cached resources, and when to fetch from the network. citeturn531510search6

---

# 36. Network-Only

Use for:

```text
sensitive mutations
strongly fresh requests
authentication-sensitive operations
```

Example:

```js
if (event.request.method !== "GET") {
  return;
}
```

Do not cache:

```text
POST
PATCH
DELETE
```

responses blindly.

---

# 37. Cache-Only

Useful for:

```text
immutable/versioned assets
```

but can fail if:

```text
cache entry missing.
```

Use only when a reliable cache population invariant exists.

---

# 38. Cache Strategy Decision Table

| Resource | Typical Strategy |
|---|---|
| versioned JS/CSS | cache-first |
| HTML navigation | network-first / navigation fallback |
| image assets | cache-first |
| news/feed data | stale-while-revalidate |
| user-specific API | carefully controlled network/cache |
| mutation | network-only |
| offline shell | cache-only fallback |

These are starting points, not universal policies.

---

# 39. Precache

Precache means:

```text
store critical resources during install.
```

Example:

```js
const SHELL = [
  "/",
  "/index.html",
  "/styles.css",
  "/app.js",
  "/offline.html"
];
```

Then:

```js
await cache.addAll(SHELL);
```

---

# 40. Precache Trade-Offs

Pros:

```text
fast startup
offline boot
predictable shell
```

Cons:

```text
larger install cost
stale content risk
installation failure
storage use
```

Do not precache:

```text
everything.
```

---

# 41. Runtime Caching

Runtime caching stores resources:

```text
when the application requests them.
```

This is useful for:

```text
large catalogs
user navigation
images
API reads
```

It can grow unpredictably, so add:

```text
eviction
versioning
limits
```

---

# 42. Cache Versioning

Use names such as:

```text
static-v3
runtime-v7
api-v4
```

Then during activation:

```js
const keep = new Set([
  "static-v3",
  "runtime-v7"
]);
```

Delete old caches.

---

# 43. Cache Cleanup

Example:

```js
event.waitUntil(
  caches.keys().then(async keys => {
    await Promise.all(
      keys
        .filter(key => !keep.has(key))
        .map(key => caches.delete(key))
    );
  })
);
```

Always define:

```text
which caches are authoritative
```

rather than:

```text
delete anything that looks old.
```

---

# 44. Browser Cache vs Cache Storage

Separate concepts:

```text
HTTP cache
→ browser-controlled HTTP caching

Cache Storage
→ script-accessible Request/Response cache
```

They can both participate in resource delivery, but they have different ownership and APIs. citeturn531510search2turn531510search11

---

# 45. Cache API Semantics

Cache Storage can hold multiple named:

```text
Cache
```

objects.

An origin's scripts can typically create:

```text
many named caches.
```

The application/service worker is responsible for deciding:

```text
which resource goes into which cache
```

and:

```text
when it is deleted. citeturn531510search2
```

---

# 46. Request / Response Cloning

Request and Response bodies are generally:

```text
stream-like
```

and can be consumed.

Example:

```js
const response = await fetch(request);

await cache.put(request, response.clone());

return response;
```

Without cloning, consuming the body for caching can interfere with returning the same response.

---

# 47. Body Consumption Mental Model

Think:

```text
Response
 ↓
body stream
 ↓
consume once
```

If you need:

```text
consumer A
+
consumer B
```

create:

```text
clone
```

where supported by the API semantics.

---

# 48. Opaque Responses

Cross-origin requests may produce:

```text
opaque responses
```

under certain CORS modes.

Opaque responses reveal limited information to script.

Caching them can still consume storage.

Therefore:

```text
opaque cache entries
```

need deliberate policy.

---

# 49. CORS Interaction

A service worker does not magically bypass:

```text
CORS
```

The worker operates within browser security rules.

Do not design a service worker as:

```text
proxy that defeats cross-origin policy
```

unless you are explicitly implementing a permitted same-origin server-side architecture.

---

# 50. Authenticated Requests

For a request containing:

```text
cookies
Authorization headers
```

carefully consider whether the response is:

```text
user-specific
```

and safe to persist.

Avoid:

```text
cache all GET responses
```

as a universal strategy.

---

# 51. Cache Key and Request Semantics

Cache matching considers request properties/semantics according to the Cache API.

Still, application correctness requires awareness of:

```text
query
method
headers
credentials
vary
authorization
```

A cached response is safe only if the cache key captures all relevant response dimensions or the policy otherwise proves safety.

---

# 52. Personalized Response Risk

Example:

```text
GET /api/profile
```

returns:

```text
user A
```

If cached under an insufficiently constrained key, user B could receive:

```text
user A data.
```

This is a catastrophic cache isolation failure.

Never cache personalized resources without proving:

```text
identity partitioning
```

and:

```text
safe reuse.
```

---

# 53. Cache Poisoning

A malicious or malformed response can become persistent if a service worker caches it.

Threat chain:

```text
bad response
 ↓
cache.put()
 ↓
persistent cached copy
 ↓
future requests
```

Only cache responses that satisfy:

```text
trusted origin
expected status
expected content type
correct authentication context
cache policy
```

---

# 54. Cache Integrity

For critical application assets:

```text
immutable hashed filenames
+
controlled cache population
```

reduce the chance that:

```text
wrong version
```

is accidentally served.

For deployment:

```text
app.abc123.js
app.def456.js
```

is easier to reason about than:

```text
app.js
```

with uncontrolled cache lifetime.

---

# 55. Application Shell

The application shell is:

```text
minimal resources needed to boot the UI.
```

A robust shell often includes:

```text
HTML
CSS
JS
icons
offline fallback
```

and loads domain data separately.

This separates:

```text
bootability
```

from:

```text
fresh application data.
```

---

# 56. Offline-First Architecture

Use:

```text
       UI
        |
        v
   local state
    /      \
cache       IndexedDB
  |             |
assets       domain data
    \         /
      sync/network
```

Service worker primarily handles:

```text
network interception
resource caching
background-capable events
```

IndexedDB handles:

```text
structured durable domain data
```

as discussed in Chapter 132.

---

# 57. Offline Mutation Architecture

Typical flow:

```text
user action
 ↓
IndexedDB transaction
 ├── local entity update
 └── outbox mutation
 ↓
UI updated
 ↓
background sync / retry
 ↓
server
 ↓
ack
 ↓
remove outbox entry
```

The service worker can assist networking, but durable mutation state belongs in an appropriate persistent store.

---

# 58. Background Sync

Background Sync can let a service worker retry tasks when connectivity returns.

Conceptually:

```text
offline mutation
→ register sync
→ browser chooses suitable time
→ sync event
→ retry
```

Do not assume:

```text
exact execution time.
```

Background work is:

```text
browser-controlled.
```

---

# 59. Background Execution Is Not a Server

A service worker may be:

```text
started
→ run briefly
→ terminated
→ restarted later
```

Therefore never rely on:

```text
global in-memory state
```

for durable workflow state.

Store important state in:

```text
IndexedDB
```

or another appropriate storage mechanism.

---

# 60. Stateless Service Worker Design

Prefer:

```text
event
→ load durable state
→ perform bounded work
→ persist result
→ finish
```

rather than:

```text
event A
→ global variable
→ event B assumes variable exists
```

Worker termination makes the latter unreliable.

---

# 61. Service Worker Lifetime

A browser may stop an idle service worker.

Therefore:

```text
worker process lifetime
```

is not:

```text
application lifetime.
```

This is one of the biggest differences from:

```text
Node.js server process
```

mental models.

---

# 62. Global Scope Is Not Durable State

This:

```js
let queue = [];
```

can be useful for:

```text
short-lived event handling
```

but not for:

```text
durable background queue.
```

On restart:

```text
queue disappears.
```

---

# 63. Push

Push can wake a service worker in supported environments.

Typical architecture:

```text
server
 ↓
push service
 ↓
browser
 ↓
service worker push event
 ↓
notification / state update
```

The worker should retrieve authoritative server state when needed rather than trusting a notification payload as the complete source of truth.

---

# 64. Periodic Background Sync

Periodic Background Synchronization can let a service worker periodically refresh data under browser-controlled conditions.

MDN currently marks this API as:

```text
limited availability
experimental
```

so it should not be treated as universally production-safe. citeturn531510search10turn531510search8

---

# 65. Navigation Preload

A cold service worker may need time to boot before handling a navigation fetch.

Navigation preload can allow network fetching to proceed in parallel with worker startup.

Conceptually:

```text
navigation
├── start service worker
└── start preload request
        ↓
      worker ready
        ↓
  consume preload response
```

This can reduce startup-related navigation latency. citeturn531510search7turn531510search12

---

# 66. Enabling Navigation Preload

Typical pattern:

```js
self.addEventListener("activate", event => {
  event.waitUntil(
    self.registration.navigationPreload?.enable()
  );
});
```

Then in fetch handling:

```js
const preload = await event.preloadResponse;
```

when available.

---

# 67. Navigation Preload Header

Preload requests can include:

```text
Service-Worker-Navigation-Preload
```

and the server can vary its response accordingly.

If a response differs based on that header, appropriate:

```text
Vary
```

handling is required. citeturn531510search3

---

# 68. Navigation Preload Trade-Offs

Benefits:

```text
less cold-start latency
parallel network + worker startup
```

Costs:

```text
extra server complexity
response coordination
cache correctness
browser support considerations
```

Do not enable it without:

```text
understanding the fetch path.
```

---

# 69. Service Worker Updates

A service worker can update when the browser checks for a changed script.

The registration exposes:

```js
registration.update()
```

for an explicit update check.

MDN notes that `update()` checks for a newer worker script and installs it when the new script differs from the current one; update-fetch caching has defined behavior that includes cache-bypass conditions. citeturn531510search5

---

# 70. `updateViaCache`

The registration's:

```js
updateViaCache
```

controls how the HTTP cache is consulted during service-worker script update checks and related imported scripts.

Possible values include:

```text
imports
all
none
```

according to the platform. citeturn531510search9turn531510search4

---

# 71. Update Is a Two-Version Problem

At update time you can have:

```text
old worker
+
new installing worker
+
old clients
```

This is why worker updates are closer to:

```text
rolling deployment
```

than:

```text
replace one JS file.
```

---

# 72. Atomic Deployment

A safe deployment sequence is:

```text
deploy new hashed assets
      ↓
deploy worker referencing them
      ↓
install
      ↓
cache verification
      ↓
activate
      ↓
clients move together
```

Avoid a state where:

```text
new worker
→ old asset URLs
```

or:

```text
old worker
→ deleted assets
```

can occur.

---

# 73. Asset Compatibility Matrix

Before activating worker version:

```text
SW v1 ↔ app v1
SW v2 ↔ app v2
```

Ask:

```text
Can v1 worker safely serve v2 page?

Can v2 worker safely serve v1 page?

Do API contracts remain compatible?

Are migrations backward compatible?
```

This is a deployment architecture problem.

---

# 74. Service Worker Rollback

Rollback should consider:

```text
worker version
cache version
app asset version
database schema
API compatibility
```

Deleting a worker script alone does not undo already-cached resources.

Design:

```text
previous known-good cache
```

or:

```text
server-side emergency fallback
```

where appropriate.

---

# 75. Bad Update Pattern

```text
install
→ cache new assets
→ skipWaiting
→ clients.claim
→ old page continues
```

Potential problem:

```text
old JS
+
new cached assets
+
new fetch behavior
```

This may produce subtle failures.

---

# 76. Safer User-Coordinated Update

Pattern:

```text
new worker installs
↓
worker sends:
"new version ready"
↓
client displays:
"Reload to update"
↓
user confirms
↓
worker activates
↓
page reloads
```

This gives:

```text
version consistency
```

at the cost of:

```text
delayed adoption.
```

---

# 77. Messaging

The page and service worker can communicate using:

```text
postMessage
```

and message events.

Example:

```js
navigator.serviceWorker.controller?.postMessage({
  type: "SKIP_WAITING"
});
```

The worker can respond:

```js
self.addEventListener("message", event => {
  if (event.data?.type === "SKIP_WAITING") {
    self.skipWaiting();
  }
});
```

Use explicit message protocols.

---

# 78. Message Contract

Define:

```text
message type
schema
version
source
expected response
error behavior
```

For example:

```js
{
  type: "CACHE_CLEAR",
  version: 1
}
```

Do not pass arbitrary messages without validation.

---

# 79. Client Enumeration

A service worker can inspect its controlled clients through:

```js
self.clients
```

This can support:

```text
broadcast notifications
update coordination
state invalidation
```

But treat clients as:

```text
ephemeral browser contexts
```

not durable records.

---

# 80. BroadcastChannel Relationship

For cross-context messaging:

```text
BroadcastChannel
```

may be simpler for:

```text
tab-to-tab application events.
```

Service worker messaging is better when:

```text
worker participation
```

is part of the design.

Do not use the service worker as an unnecessary message broker.

---

# 81. Scope and Existing Pages

A worker can only control pages that meet:

```text
origin
scope
lifecycle/control rules
```

An active worker does not automatically retroactively convert every open page into a controlled client.

`clients.claim()` changes this behavior after activation within applicable scope. citeturn531510search0

---

# 82. Multi-Tab Architecture

You may have:

```text
Tab A → worker v1
Tab B → worker v1
new worker v2 → waiting
```

After controlled update:

```text
Tab A → v2
Tab B → v2
```

The exact transition depends on:

```text
lifecycle
reload
claim
skipWaiting
```

and application coordination.

---

# 83. Shared Origin, Shared Worker Registration

A service worker registration belongs to an origin and scope.

Therefore:

```text
multiple tabs
```

can share:

```text
one active registration.
```

This differs from:

```text
one worker instance permanently serving all requests.
```

The browser controls worker instances and lifetime.

---

# 84. Error Handling

Every event handler should consider:

```text
network fails
cache missing
cache corrupted
storage quota
response invalid
worker restart
aborted requests
```

Example:

```js
event.respondWith(
  strategy(event.request).catch(() => fallback())
);
```

But do not hide all exceptions.

Log:

```text
structured diagnostics
```

for unexpected failures.

---

# 85. Offline Is Not Just “No Network”

Offline scenarios include:

```text
no connection
DNS failure
server timeout
TLS failure
captive portal
partial connectivity
slow connection
server 500
expired auth
stale cache
quota pressure
```

A robust offline architecture treats:

```text
network failure
```

as a family of failure modes.

---

# 86. Captive Portals

A device may report:

```text
online
```

while HTTP traffic is redirected to:

```text
login page
```

for network access.

Do not interpret:

```text
navigator.onLine
```

as:

```text
server is reachable and trustworthy.
```

Actual fetch outcomes matter.

---

# 87. `navigator.onLine`

This can help adjust UX, but it is not a reliable network truth oracle.

Use:

```text
actual request success/failure
```

for application decisions.

---

# 88. Retry Policy

Do not retry:

```text
every failure
immediately
forever.
```

Use:

```text
bounded retries
exponential backoff
jitter
idempotency
retry classification
```

especially for background sync.

---

# 89. Idempotency for Offline Sync

A retryable mutation should have:

```text
operation ID
```

so the server can recognize:

```text
duplicate delivery.
```

Example:

```text
mutationId = "01..."
```

Server:

```text
if already applied:
    return same result
```

---

# 90. Offline Conflict

Suppose:

```text
device offline
→ edits order
```

Meanwhile:

```text
server order changes.
```

The service worker cannot decide the business conflict.

You need:

```text
domain merge policy
```

such as:

```text
server-wins
client-wins
field merge
manual resolution
version conflict
```

---

# 91. Service Worker + IndexedDB

A mature offline system often uses:

```text
Service Worker
→ networking/cache

IndexedDB
→ durable domain state/outbox

UI
→ local store

Server
→ authority
```

This separation keeps:

```text
resource caching
```

different from:

```text
business persistence.
```

---

# 92. Cache Storage + IndexedDB Boundary

Use Cache Storage for:

```text
HTTP representations
```

Use IndexedDB for:

```text
entities
queries
outbox
sync metadata
```

A common mistake is:

```text
put everything into Cache Storage
```

because:

```text
Cache API is easy.
```

Ease is not the same as domain fit.

---

# 93. Cache Invalidation

Cache invalidation problems include:

```text
stale app shell
stale data
deleted resources
API schema changes
user-specific responses
```

Strategies:

```text
versioned URLs
ETag/revalidation
TTL
explicit invalidation
cache version
server freshness headers
```

---

# 94. Hashed Assets

Use:

```text
main.8a12f7.js
styles.44b9c1.css
```

rather than mutable:

```text
main.js
styles.css
```

for long-lived static caches.

Hash-based naming makes:

```text
cache invalidation
```

closer to:

```text
cache replacement
```

---

# 95. HTML Is Different

HTML often should not be treated exactly like immutable JS/CSS.

A common architecture is:

```text
HTML:
network-first

JS/CSS:
cache-first with hashed filenames
```

because HTML needs to reference:

```text
current assets.
```

---

# 96. Service Worker and HTTP Headers

Service-worker caching does not replace:

```text
HTTP cache-control
ETag
Last-Modified
Vary
Content-Encoding
```

Your architecture can involve:

```text
HTTP cache
+
Cache Storage
```

and both need consistent policy.

---

# 97. Vary

If a response changes based on headers such as:

```text
Accept
Accept-Language
Service-Worker-Navigation-Preload
```

the caching layer needs the appropriate:

```text
Vary
```

semantics.

Never assume:

```text
URL alone
```

defines a response.

---

# 98. Cache Lifetime

Cache Storage entries can remain until:

```text
application deletes them
browser evicts storage
user clears site data
policy/lifecycle changes
```

Therefore:

```text
cache = permanent
```

is false.

Chapter 132 covers quota/eviction in greater depth.

---

# 99. Storage Quota

A service worker's Cache Storage uses browser-managed origin storage.

Therefore runtime can encounter:

```text
quota pressure
```

or:

```text
eviction.
```

Critical offline data should have a:

```text
recovery path.
```

---

# 100. Service Worker Security Model

A service worker can intercept network traffic in its scope.

This means:

```text
compromised worker
```

can be extremely powerful.

Protect:

```text
worker script deployment
scope
registration
cache contents
update path
dependencies
```

---

# 101. Service Worker Takeover Risk

If an attacker can cause a malicious script to be registered as the site's service worker, they may gain long-lived control over matching network flows.

Defense includes:

```text
strict deployment controls
HTTPS
safe scope
CSP and script integrity where applicable
dependency security
avoid XSS
protect worker script path
```

The most important point:

```text
XSS can become durable through service-worker registration
```

until the malicious worker is removed and affected clients recover.

---

# 102. Service Worker Script Location

Because scope normally relates to:

```text
worker script directory
```

deployment layout matters.

Example:

```text
/sw.js
```

can have broader default scope than:

```text
/app/sw.js
```

Do not move the script without reviewing:

```text
scope
clients
routing
```

---

# 103. Service Worker Scope Review

For every worker ask:

```text
What URLs can it control?

What requests can it intercept?

Which assets can it cache?

Which API responses can it modify?

Which origins are involved?

Can a compromised worker affect authentication?
```

This is a formal threat-model exercise.

---

# 104. CSP Considerations

Content Security Policy can help reduce script injection risk, but service-worker registration and execution have their own browser rules.

Do not assume:

```text
CSP alone
```

fully secures:

```text
service-worker lifecycle
```

Combine:

```text
CSP
+
secure deployment
+
dependency controls
+
XSS prevention
```

---

# 105. Service Worker and Authentication

A service worker should generally not become the place where authentication secrets are reinvented.

It can:

```text
observe requests in its scope
```

and therefore security-sensitive logic must be explicit.

Avoid:

```text
logging Authorization headers
```

or:

```text
persisting sensitive tokens in Cache Storage.
```

---

# 106. Cache Authentication Responses Carefully

Never assume:

```text
GET = safe to cache
```

because:

```text
GET /account
```

may be:

```text
user-specific.
```

A safe strategy may be:

```text
network-only
private cache with explicit user key
server-controlled revalidation
```

depending on the application.

---

# 107. Request Method Policy

A simple protective baseline:

```js
if (!["GET", "HEAD"].includes(request.method)) {
  return fetch(request);
}
```

This does not guarantee safety, but it avoids accidentally caching mutations.

---

# 108. Response Validation

Before caching:

```js
if (response.ok) {
  await cache.put(request, response.clone());
}
```

This is better than:

```text
cache every response.
```

But still ask:

```text
Is 404 cacheable?
Is 401 cacheable?
Is response private?
Does Vary matter?
```

---

# 109. Offline Authentication

Offline apps need an explicit answer to:

```text
Can authenticated data be shown offline?
```

Consider:

```text
session expiry
revocation
token validity
device compromise
cached sensitive information
```

Never silently assume:

```text
cached page = still authorized.
```

---

# 110. Offline Authorization

A service worker cannot make server authorization disappear.

Offline mode usually means:

```text
previously authorized local state
```

is being displayed.

For high-risk actions:

```text
require online authorization
```

or:

```text
use a carefully designed offline credential model.
```

---

# 111. Offline UX

A good offline UI exposes:

```text
online/offline state
last successful sync
pending changes
sync errors
retry
conflict status
```

Do not pretend:

```text
offline mutation = server accepted.
```

Use language such as:

```text
Saved on this device
Pending synchronization
```

when appropriate.

---

# 112. Background Sync UX

When a sync succeeds:

```text
clear outbox
update local state
```

When it fails permanently:

```text
move to failed state
surface to user
```

Do not silently:

```text
retry forever.
```

---

# 113. Service Worker + Streams

Fetch responses can be streamed.

A service worker can potentially transform response flows using:

```text
ReadableStream
TransformStream
```

when supported by the relevant architecture.

This can enable:

```text
response transformation
progressive delivery
stream-aware proxying
```

but increases:

```text
complexity
memory
failure modes.
```

---

# 114. Avoid Buffering Huge Responses

Bad:

```js
const text = await response.text();
const modified = transform(text);
return new Response(modified);
```

for very large payloads.

This can:

```text
increase memory
delay first byte
block useful streaming.
```

Prefer streaming transforms when the requirement truly needs transformation.

---

# 115. Service Worker Performance

Measure:

```text
worker startup
fetch handler latency
cache lookup
network latency
response transformation
serialization
storage time
```

A service worker should not turn:

```text
simple network request
```

into:

```text
many sequential async operations.
```

---

# 116. Cold Start

When the worker is not already running:

```text
request
→ browser starts worker
→ worker initializes
→ handler executes
```

Initialization cost matters.

Keep module initialization:

```text
small
deterministic
side-effect-light
```

Navigation preload can help with navigation fetch startup. citeturn531510search7

---

# 117. Avoid Large Global Initialization

Do not:

```text
import huge dependency graph
initialize database
scan caches
load configuration
```

before every fetch can proceed.

Prefer:

```text
lazy
event-specific
bounded
```

work.

---

# 118. Service Worker Logging

Log:

```text
worker version
event type
request class
cache strategy
cache hit/miss
network outcome
error
```

Avoid:

```text
tokens
cookies
personalized response bodies
```

in logs.

---

# 119. Worker Version Identifier

Embed:

```js
const SW_VERSION = "2026.09.11.1";
```

or use a build-time identifier.

This makes debugging:

```text
which worker is active?
```

much easier.

---

# 120. Debugging Stale Workers

Symptoms:

```text
old UI
new API behavior
unexpected offline assets
```

Investigate:

```text
registration.active
registration.waiting
registration.installing
controller
cache names
worker script version
```

and:

```text
multiple tabs
```

---

# 121. Debugging Stale Cache

Check:

```text
Cache Storage
cache name
request URL
request method
response
cache population path
activation cleanup
```

Do not only clear the browser's normal HTTP cache.

---

# 122. Debugging Registration

Verify:

```js
navigator.serviceWorker.controller
navigator.serviceWorker.ready
navigator.serviceWorker.getRegistration()
```

and:

```text
registration.scope
registration.active?.state
```

---

# 123. Common “It Doesn't Work” Causes

```text
wrong scope
HTTPS requirement
worker script not reachable
syntax/runtime error
old worker still active
waiting worker not activated
client not controlled
cache entry missing
fetch event filtered by condition
response body already consumed
CORS issue
quota/storage issue
```

---

# 124. Update Failure Recovery

If installation fails:

```text
new worker
→ becomes unusable/redundant
→ old worker continues
```

This is usually safer than replacing the active worker with a broken version.

Design deployments so:

```text
old version
```

can continue serving while:

```text
new version
```

is validated.

---

# 125. Migration Failure

If activation includes:

```text
database migration
```

and migration fails, decide:

```text
activation fails?
fallback?
retry?
rollback?
```

Prefer keeping:

```text
old known-good state
```

rather than partially migrating the system.

---

# 126. Cache Migration

For a new cache schema:

```text
cache-v1
→ cache-v2
```

prefer:

```text
populate v2
verify
switch reads
delete v1
```

rather than:

```text
delete v1
→ start building v2
```

which creates a larger outage window.

---

# 127. Two-Phase Client Update

A mature deployment can use:

```text
Phase 1:
install + validate

Phase 2:
activate + clients reload
```

This resembles:

```text
blue/green deployment
```

at the browser layer.

---

# 128. Cache Warmup

Large offline applications may need:

```text
install
→ cache shell
```

and then:

```text
runtime warmup
```

later.

Do not make installation depend on:

```text
hundreds of large API responses
```

or first install will become:

```text
fragile
slow
storage-heavy.
```

---

# 129. App Shell vs Data Cache

Separate:

```text
shell:
required to run app

data cache:
helpful for user content
```

This makes recovery easier:

```text
shell available
+
data partially missing
```

can still produce a functional UI.

---

# 130. Offline Fallback Page

Keep a minimal:

```text
/offline.html
```

that requires:

```text
little/no dynamic data.
```

This provides a guaranteed fallback when:

```text
network
+
dynamic cache
```

both fail.

---

# 131. Service Worker and Routing

A service worker can participate in routing decisions:

```text
request URL
→ request classification
→ cache/network strategy
```

But complex application routing should not all be moved into the worker.

Use:

```text
browser/router
```

for UI routing and:

```text
service worker
```

for network/resource routing.

---

# 132. URL Parsing in Service Workers

Use:

```js
const url = new URL(request.url);
```

not:

```text
string splitting.
```

Then classify:

```text
url.origin
url.pathname
url.search
```

This connects directly to:

```text
Chapter 131 — URI / URL
```

---

# 133. Query-Aware Cache Policy

Example:

```text
/api/products?page=1
/api/products?page=2
```

may be different resources.

Do not cache:

```text
all /api/products
```

under one arbitrary key.

---

# 134. Language-Aware Cache Policy

Responses may vary by:

```text
Accept-Language
```

or user locale.

If your worker caches:

```text
localized response
```

define:

```text
cache key partition
```

and:

```text
Vary
```

requirements.

---

# 135. Credential-Aware Policy

A response may vary by:

```text
Cookie
Authorization
```

A generic cache:

```text
GET → cache
```

can leak data if the response is identity-specific.

Always classify:

```text
public
private
user-specific
sensitive
```

before caching.

---

# 136. `cache.match()` Is Not Authorization

Finding a response in Cache Storage only proves:

```text
a matching cached item exists.
```

It does not prove:

```text
current user is authorized to receive it.
```

Authorization remains a domain/security decision.

---

# 137. Offline Cache as a Security Boundary

Ask:

```text
Who can use the cached data?

Can another user log into the same browser profile?

What happens on logout?

Are caches cleared?

Are IndexedDB records cleared?

Is the service worker still serving old data?
```

Logout should consider:

```text
all persisted client state
```

where sensitive data is involved.

---

# 138. Logout Cleanup

A sensitive application may need to:

```text
clear auth state
clear user-specific IndexedDB
delete private caches
reset service-worker state
notify all tabs
```

This should be designed explicitly.

---

# 139. Cache Namespacing

A strategy can use:

```text
public-static-v3
public-runtime-v2
user-data-v1
```

and different cleanup rules.

This improves:

```text
security review
```

and:

```text
operational control.
```

---

# 140. Multi-User Browser Scenario

Imagine:

```text
User A logs out
User B logs in
```

If the service worker still serves:

```text
A's cached private response
```

the application has a:

```text
cross-account data leak.
```

This is why:

```text
cache lifetime
```

must be tied to:

```text
identity lifetime
```

for private data.

---

# 141. Service Worker and CSP/Permissions

The worker operates within:

```text
origin/security policy
```

and may interact with APIs subject to:

```text
permissions
policy
browser restrictions
```

Do not assume:

```text
worker = privileged native process.
```

It is powerful within the web security model, but constrained by it.

---

# 142. Browser Resource Constraints

The browser controls:

```text
CPU
memory
storage
network
background execution
```

Your worker is a:

```text
best-effort client component
```

not a guaranteed always-on daemon.

---

# 143. When Not to Use a Service Worker

Avoid adding one merely because:

```text
“all PWAs need a service worker.”
```

You may not need one for:

```text
simple static sites
server-rendered pages with no offline requirement
small apps with no caching requirement
applications where browser interception adds complexity without value
```

---

# 144. Service Worker Complexity Tax

A service worker introduces:

```text
lifecycle complexity
cache complexity
update complexity
debugging complexity
security surface
```

The benefits must exceed this operational cost.

---

# 145. Common Misconceptions

### Misconception 1

> “Service workers keep running forever.”

Correction:

```text
The browser can terminate idle service workers.
```

### Misconception 2

> “A service worker is a backend server in the browser.”

Correction:

```text
It is a browser-managed event-driven worker with bounded lifecycle.
```

### Misconception 3

> “skipWaiting is always better.”

Correction:

```text
It can create old-page/new-worker version mismatches.
```

### Misconception 4

> “Cache Storage is the browser's HTTP cache.”

Correction:

```text
They are separate layers.
```

### Misconception 5

> “GET responses are always safe to cache.”

Correction:

```text
GET can still be authenticated and user-specific.
```

### Misconception 6

> “Offline means the app is still authorized.”

Correction:

```text
Cached local state does not replace current server authorization.
```

### Misconception 7

> “navigator.onLine tells you the server is reachable.”

Correction:

```text
It is only a coarse connectivity hint.
```

### Misconception 8

> “A service worker can bypass CORS.”

Correction:

```text
It remains within browser security rules.
```

### Misconception 9

> “Service-worker global variables are durable.”

Correction:

```text
The worker can restart at any time.
```

### Misconception 10

> “A service worker update immediately replaces all clients.”

Correction:

```text
New workers commonly install/wait before activation.
```

---

# 146. Common Mistakes

```text
[ ] precaching too much
[ ] caching every GET response
[ ] caching user-specific responses without isolation
[ ] calling skipWaiting blindly
[ ] ignoring multi-tab update coordination
[ ] storing durable state only in worker globals
[ ] forgetting clients.claim semantics
[ ] consuming a response body before caching/returning it
[ ] ignoring opaque responses
[ ] ignoring quota/eviction
[ ] leaving old caches forever
[ ] mixing app routing with network routing
[ ] logging sensitive requests
[ ] assuming browser background execution is guaranteed
[ ] relying on navigator.onLine
[ ] retrying mutations without idempotency
[ ] deploying worker and assets non-atomically
[ ] failing to test offline + update combinations
```

---

# 147. Comparison With Related Concepts

| Concept | Primary Purpose |
|---|---|
| Service Worker | programmable browser lifecycle/network boundary |
| Web Worker | off-main-thread computation |
| Cache Storage | Request/Response persistence |
| IndexedDB | structured local data |
| HTTP Cache | browser-controlled network cache |
| BroadcastChannel | cross-context messaging |
| Web Locks | cross-context coordination |
| Background Sync | browser-scheduled retry work |
| Periodic Background Sync | browser-controlled periodic refresh |
| Push | server-triggered background event |
| Server Worker / Backend | authoritative server execution |

---

# 148. Production Architecture Example

```text
                    Browser
                       |
             ┌─────────▼─────────┐
             │   Service Worker  │
             └───────┬─────┬─────┘
                     │     │
              network│     │cache
                     │     │
              ┌──────▼─┐ ┌─▼────────┐
              │ Server │ │ Cache API │
              └────────┘ └───────────┘
                     |
               domain sync
                     |
              ┌──────▼─────────┐
              │   IndexedDB    │
              │ entities/outbox│
              └──────┬─────────┘
                     |
                   UI
```

---

# 149. Production Request Pipeline

```text
browser request
      ↓
service worker scope?
      ├── no → normal browser request
      │
      └── yes
          ↓
       classify
          ├── navigation
          ├── static asset
          ├── public data
          ├── private data
          └── mutation
                  ↓
             choose strategy
                  ↓
          cache/network/offline
                  ↓
              response
```

---

# 150. Production Deployment Pipeline

```text
build
 ↓
hashed assets
 ↓
deploy assets
 ↓
deploy service worker
 ↓
browser detects update
 ↓
install
 ↓
verify caches
 ↓
waiting
 ↓
user/strategy chooses activation
 ↓
activate
 ↓
cleanup
 ↓
clients migrate
```

---

# 151. Update Testing Matrix

Test combinations:

```text
old app + old worker
old app + new waiting worker
new app + old active worker
new app + new worker
offline during install
offline during activation
cache population failure
quota exhaustion
tab left open for days
logout during update
schema migration during update
```

This catches:

```text
split-version failures.
```

---

# 152. Offline Testing Matrix

Test:

```text
first visit online
first visit offline
return visit offline
network drops mid-request
network drops after local write
server returns 500
server returns 401
DNS failure
slow network
airplane mode
cache missing
cache corrupted
storage evicted
```

---

# 153. Multi-Tab Testing Matrix

Test:

```text
Tab A open old app
Tab B opens new app
worker update arrives
Tab A stays open
Tab B reloads
Tab A receives update notification
logout in Tab A
Tab B observes
```

---

# 154. Security Testing Matrix

Include:

```text
XSS → malicious worker registration
cache poisoning
user-specific response leakage
logout cleanup
credentialed requests
redirects
opaque responses
third-party requests
scope escalation
worker script tampering
dependency compromise
```

---

# 155. Performance Testing

Measure:

```text
cold worker startup
warm worker latency
cache hit
cache miss
network-first latency
stale-while-revalidate latency
large cache lookup
cache.put cost
navigation preload impact
offline boot
```

Benchmark:

```text
realistic low-end devices
```

not only:

```text
developer workstation.
```

---

# 156. Service Worker Code Review

For every fetch handler ask:

```text
Is request classification explicit?

Are mutations excluded?

Are personalized responses protected?

Is cache strategy correct?

Is cache entry versioned?

Is the response consumed safely?

Can the request hang indefinitely?

What happens offline?

What happens when cache is missing?

What happens after logout?

What happens during worker update?

```

---

# 157. Implementation From Scratch — Mini Service-Worker Architecture

Do not attempt to emulate the browser.

Build a local educational model:

```text
Registration
Worker Lifecycle
CacheStore
FetchRouter
UpdateManager
OfflineQueue
```

The goal is:

```text
understand lifecycle and ownership
```

not:

```text
reimplement browser internals.
```

---

# 158. Implementation Milestone 1 — Cache Strategy Router

Implement:

```js
class StrategyRouter {
  route(request) {
    if (request.mode === "navigate") {
      return "network-first";
    }

    if (request.destination === "script") {
      return "cache-first";
    }

    return "stale-while-revalidate";
  }
}
```

Then expand with:

```text
API classification
private-data policy
mutation exclusion
```

---

# 159. Implementation Milestone 2 — Versioned Cache Manager

Build:

```text
openVersion
put
match
deleteVersion
cleanupOldVersions
```

Add:

```text
atomic activation
```

so old cache remains available until new cache preparation succeeds.

---

# 160. Implementation Milestone 3 — Offline Queue

Build:

```text
enqueue
retry
ack
fail
backoff
```

Persist through:

```text
IndexedDB.
```

Do not store the queue only in memory.

---

# 161. Implementation Milestone 4 — Update Manager

Model:

```text
installing
waiting
active
```

and transitions:

```text
install
→ wait
→ activate
```

Add:

```text
user approval
```

before activation.

---

# 162. Implementation Milestone 5 — Failure Injection

Simulate:

```text
network failure
cache miss
quota failure
worker crash
migration error
response corruption
duplicate mutation
```

Then prove:

```text
old known-good state survives.
```

---

# 163. Debugging Exercises

## Exercise A — Worker Never Controls Page

Investigate:

```text
registration succeeds
worker active
navigator.serviceWorker.controller === null
```

Determine whether:

```text
page needs reload
scope is wrong
worker is not active
clients.claim was not used
```

---

## Exercise B — New Code Not Appearing

Trace:

```text
old worker
new worker
waiting
cache version
asset URLs
```

Do not start by deleting all site data.

---

## Exercise C — Offline User Sees Another User

Simulate:

```text
User A login
→ cache profile
→ logout
→ User B login
→ profile request
```

Find the isolation bug.

---

## Exercise D — Install Fails

Make:

```text
cache.addAll()
```

fail for one resource.

Observe:

```text
worker lifecycle
active worker
```

and design a recovery strategy.

---

# 164. Code Review Exercise

Review:

```js
self.addEventListener("fetch", event => {
  event.respondWith(
    caches.open("all").then(cache =>
      cache.match(event.request).then(cached =>
        cached || fetch(event.request).then(response => {
          cache.put(event.request, response.clone());
          return response;
        })
      )
    )
  );
});
```

Find at least:

```text
mutation caching risk
private-response risk
cache growth risk
error handling gaps
quota handling
response policy
staleness
cache invalidation
```

Then redesign the strategy by request class.

---

# 165. Interview Questions

### Fundamentals

```text
1. What is a service worker?
2. Why does it not have DOM access?
3. How is it different from a web worker?
4. What is registration scope?
5. What are installing, waiting, and active workers?
```

### Lifecycle

```text
6. What happens during install?
7. Why use event.waitUntil()?
8. Why does a worker wait?
9. What does skipWaiting() do?
10. What does clients.claim() do?
```

### Fetch

```text
11. What does fetch event interception mean?
12. What does respondWith() do?
13. Why can response.clone() be needed?
14. How would you implement offline navigation?
15. What is navigation preload?
```

### Caching

```text
16. How does cache-first differ from network-first?
17. What is stale-while-revalidate?
18. Why can GET responses still be unsafe to cache?
19. How do you version caches?
20. How do you protect personalized responses?
```

### Offline

```text
21. How do you design an offline mutation queue?
22. Why must background state be durable?
23. How do you handle conflicts?
24. How do you handle duplicate mutation retries?
```

### Security

```text
25. What happens if a malicious service worker is registered?
26. How could cache poisoning occur?
27. How do logout and client-side caches interact?
28. How do you limit service worker scope?
```

### Principal

```text
29. How would you design atomic application updates?
30. How would you roll back a broken worker?
31. How would you design a multi-tab update protocol?
32. How would you decide what should be precached?
33. When should an application not use a service worker?
```

---

# 166. Predict-the-Behavior Exercises

Predict before running.

### Exercise 1

```js
self.addEventListener("install", event => {
  event.waitUntil(
    caches.open("v1")
  );
});
```

Explain why the Promise is passed to:

```text
waitUntil
```

instead of ignored.

### Exercise 2

Two tabs are controlled by:

```text
worker v1
```

Worker v2 installs successfully.

Predict:

```text
Does v2 immediately control both tabs?
```

### Exercise 3

```js
self.addEventListener("fetch", event => {
  if (event.request.mode === "navigate") {
    event.respondWith(
      fetch(event.request).catch(() =>
        caches.match("/offline.html")
      )
    );
  }
});
```

Predict:

```text
what happens offline
```

when:

```text
/offline.html
```

is not cached.

### Exercise 4

A response is:

```text
user-specific
```

but the service worker caches it under:

```text
request URL only.
```

Predict the security failure scenario.

### Exercise 5

A worker calls:

```js
self.skipWaiting();
```

during installation and:

```js
self.clients.claim();
```

during activation.

Explain why this can produce:

```text
new worker
+
old page code
```

coexistence.

---

# 167. Mastery Exercises

### Exercise 1 — Offline Shell

Build:

```text
precache
+
navigation fallback
+
versioned cache
+
update notification
```

### Exercise 2 — Runtime Strategy Router

Support:

```text
navigation
static assets
public API
private API
mutations
```

with different policies.

### Exercise 3 — Offline Outbox

Build:

```text
IndexedDB outbox
+
service-worker retry
+
idempotency
+
exponential backoff
```

### Exercise 4 — Safe Update System

Implement:

```text
new worker
→ waiting
→ client prompt
→ activation
→ reload
```

without blind:

```text
skipWaiting
```

### Exercise 5 — Cache Security Audit

Take a fictional application and determine:

```text
which responses are safe to cache
which are private
which require network
which require invalidation.
```

---

# 168. Track A — Core Theory

Master:

```text
service-worker architecture
registration
scope
lifecycle
install
waiting
activate
fetch
extendable events
waitUntil
respondWith
skipWaiting
clients.claim
updates
navigation preload
background events
Cache Storage
offline strategies
```

Deliverable:

```text
explain any service-worker lifecycle transition and its effect on clients.
```

---

# 169. Track B — Implementation

Build:

```text
cache strategy router
versioned cache manager
offline navigation fallback
update manager
IndexedDB outbox
background retry
multi-tab update protocol
cache security policy
recovery system
```

Progression:

```text
Guided
→ Partially Guided
→ No Reference
→ Edge-Case Hardened
→ Production-Grade Learning Version
```

---

# 170. Track C — Interview / Reasoning

Practice:

```text
“Why is a waiting worker useful?”

“What can go wrong with skipWaiting?”

“How do you prevent a user from receiving another user's cached response?”

“How would you deploy a breaking frontend update?”

“What does navigation preload solve?”

“How would you design offline writes?”

“How do service workers recover after termination?”
```

Deliverable:

```text
lifecycle
+
cache policy
+
failure mode
+
recovery strategy.
```

---

# 171. Specification / Runtime Source Discipline

Use this hierarchy:

```text
1. Service Workers specification
2. Fetch specification
3. Cache API specification
4. HTML / secure-context rules
5. Web Storage / IndexedDB specifications
6. browser implementation behavior
7. framework/service-worker libraries
8. application code
```

Keep these distinctions explicit:

```text
service worker lifecycle
→ service worker platform semantics

fetch interception
→ Fetch + Service Worker semantics

cache storage
→ Cache API

storage quota/persistence
→ Storage subsystem

background sync/push
→ separate background-event capabilities
```

Do not describe:

```text
“service worker = cache”
```

because it is a much broader lifecycle/event platform.

MDN's current documentation describes registration, installation, activation, updating, fetch handling, navigation preload, and offline caching as separate but connected service-worker concerns. citeturn531510search0turn531510search4

---

# 172. Current Platform Notes

As of September 2026:

```text
Service Workers:
widely deployed and foundational for PWAs/offline architectures.

Cache Storage:
widely available.

Navigation preload:
widely supported in current mainstream browsers, but verify target support.

Periodic Background Sync:
still limited/experimental and should not be treated as universally available.

Background execution:
always browser-controlled; never assume a continuously running worker.
```

Current MDN documentation marks Cache Storage and core Service Worker registration features as broadly established, while Periodic Background Sync remains limited availability/experimental. citeturn531510search2turn531510search4turn531510search10

---

# 173. Principal Decision Framework

For every service-worker design ask:

```text
1. Do we actually need a service worker?
2. What scope should it control?
3. What requests should it intercept?
4. Which responses can be cached?
5. Which responses are personalized?
6. What is the source of truth?
7. What must work offline?
8. What may be stale?
9. What must always reach the network?
10. How are mutations persisted?
11. How are retries made idempotent?
12. How are cache versions managed?
13. How are updates activated?
14. What happens to open tabs?
15. What is the rollback plan?
16. What happens on quota/eviction?
17. What happens after logout?
18. What is the security threat model?
19. What is the observability strategy?
20. What browser features are actually available?
```

---

# 174. Production Checklist

```text
[ ] secure context verified
[ ] scope documented
[ ] registration path intentional
[ ] fetch routes classified
[ ] navigation strategy defined
[ ] static asset strategy defined
[ ] API cache strategy defined
[ ] mutation requests excluded from cache
[ ] personalized data isolation reviewed
[ ] cache names versioned
[ ] old caches cleaned
[ ] cache failures handled
[ ] quota/eviction considered
[ ] offline fallback exists
[ ] durable offline state in appropriate storage
[ ] mutation queue is idempotent
[ ] retry policy bounded
[ ] update strategy documented
[ ] skipWaiting decision explicit
[ ] clients.claim decision explicit
[ ] multi-tab update tested
[ ] rollback strategy exists
[ ] logout cleanup reviewed
[ ] sensitive logging disabled
[ ] worker version observable
[ ] navigation preload evaluated
[ ] unsupported APIs feature-detected
[ ] real-device performance tested
[ ] cold-start tested
```

---

# 175. Retrieval Record

```md
# Chapter 133 — Revision / Retrieval Record

## Attempt
- Date:
- Duration:
- Status before:
- Status after:

## Registration
-

## Scope
-

## Install
-

## Waiting
-

## Activate
-

## Control
-

## Fetch
-

## waitUntil
-

## respondWith
-

## skipWaiting
-

## clients.claim
-

## Cache Storage
-

## Cache Strategies
-

## Navigation Preload
-

## Background Sync
-

## Push
-

## Offline Architecture
-

## IndexedDB / Outbox
-

## Security
-

## Updates / Rollback
-

## Testing
-

## Implementation Progress
-

## Strongest Areas
-

## Weakest Areas
-

## Questions Requiring Rework
-

## Next Review
-
```

---

# 176. Spaced Retrieval Schedule

### Day 0

Study:

```text
registration
install
waiting
activate
control
```

### Day 1

Draw:

```text
worker v1
worker v2
tab A
tab B
```

and explain every possible transition.

### Day 3

Implement:

```text
cache-first
network-first
stale-while-revalidate
```

from memory.

### Day 7

Build:

```text
offline navigation
+
versioned caches.
```

### Day 14

Audit:

```text
authenticated caching
logout cleanup
update safety
```

### Day 21

Build:

```text
IndexedDB outbox
+
retry
+
idempotency.
```

### Day 30

Perform a complete:

```text
service-worker architecture review
```

without notes.

---

# 177. Dependency Graph

```text
Chapter 31
Async Fundamentals
        ↓
Chapter 33
Browser Event Loop
        ↓
Chapter 35
Promises
        ↓
Chapter 49
DOM
        ↓
Chapter 51
Browser APIs
        ↓
Chapter 52
Web Workers
        ↓
Chapter 53
Streams
        ↓
Chapter 55
Fetch / HTTP
        ↓
Chapter 56
Browser Security
        ↓
Chapter 57
Security Engineering
        ↓
Chapter 83
Observability
        ↓
Chapter 84
Reliability
        ↓
Chapter 85
Performance
        ↓
Chapter 86
Testing
        ↓
Chapter 109
Event-Driven Applications
        ↓
Chapter 131
URL / Encoding
        ↓
Chapter 132
Browser Storage
        ↓
Chapter 133
Service Workers / Offline Architecture
        ↓
Chapter 134
Web Locks / Cross-Tab Coordination
```

Cross-cutting:

```text
Chapter 38 → streaming responses
Chapter 79 → API contracts
Chapter 87 → deterministic async tests
Chapter 98 → service-worker anti-patterns
Chapter 101 → production failure scenarios
Chapter 129 → request/path classification
```

---

# 178. Concept Connections

## Depends On

```text
fetch
HTTP
browser workers
cache storage
IndexedDB
origins
security
async lifecycle
storage quotas
```

## Builds Toward

```text
PWA
offline-first architecture
local-first applications
sync engines
resilient web applications
installable web apps
push-enabled applications
```

## Related Concepts

```text
web workers
fetch
cache API
IndexedDB
Web Locks
BroadcastChannel
background sync
push
navigation preload
HTTP cache
service-worker updates
```

## Concepts Revisited

```text
events
promises
streams
fetch
security
storage
performance
reliability
observability
```

## Why This Chapter Matters

Service workers are one of the clearest examples of browser programming becoming:

```text
distributed systems engineering.
```

A single user session can contain:

```text
multiple tabs
multiple worker versions
cached resources
durable local state
intermittent networks
background execution
server authority
```

The engineering challenge is not:

```text
“make offline work.”
```

It is:

```text
maintain correctness across lifecycle transitions,
network failures, storage loss, version skew, and security boundaries.
```

---

# 179. Final Principal Mental Model

Use:

```text
             REGISTRATION
                  ↓
              WORKER SCRIPT
                  ↓
        ┌─────────┴─────────┐
        ↓                   ↓
    INSTALLING           ACTIVE V1
        ↓                   ↓
     WAITING                ↓
        ↓                   ↓
     ACTIVATE         CONTROLLED CLIENTS
        ↓                   ↓
     ACTIVE V2          FETCH EVENTS
                            ↓
                   ┌────────┼────────┐
                   ↓        ↓        ↓
                 CACHE   NETWORK   FALLBACK
                   ↓        ↓        ↓
                   └──── RESPONSE ───┘
```

For offline applications:

```text
             UI
              ↓
       local durable state
          /           \
      IndexedDB      Cache
          \           /
             Sync
              ↓
           Network
              ↓
            Server
```

For updates:

```text
old worker
+
old clients
+
new installing worker
+
new assets
```

must converge safely toward:

```text
new worker
+
new clients
+
new compatible assets.
```

---

# 180. Final Principal Principle

> **A service worker is a versioned, lifecycle-managed browser infrastructure component—not a cache script.**

The production-grade sequence is:

```text
define scope
→ classify requests
→ define cache ownership
→ define offline guarantees
→ persist durable state outside worker memory
→ make mutations idempotent
→ version assets/caches
→ design safe updates
→ test multiple tabs
→ test network failure
→ test quota/eviction
→ secure the worker registration path
→ observe worker/cache behavior
→ maintain rollback
```

The central distinctions to internalize are:

```text
A service worker is not a web worker.

A service worker is not a server.

A service worker is not a permanent process.

Cache Storage is not IndexedDB.

Cache Storage is not the HTTP cache.

GET does not automatically mean cache-safe.

Offline does not mean server-authorized.

skipWaiting is a deployment decision.

clients.claim is a control decision.

Worker global memory is ephemeral.

Durable state belongs in durable storage.

Background execution is browser-controlled.

A new worker and old page can coexist.

Cache versioning is part of deployment engineering.

Offline writes require idempotency and conflict policy.

The service worker is a security boundary.

```

At principal level, the real question is not:

```text
“Can my PWA work offline?”
```

It is:

```text
“What correctness, freshness, availability, security, and recovery
guarantees remain true when the worker restarts, the network disappears,
storage is evicted, multiple tabs are open, and a new application version
is being deployed?”
```