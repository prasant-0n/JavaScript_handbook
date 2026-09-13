# Chapter 108 — Production JavaScript Cache System

> **JavaScript Mastery — Part XX: Projects**
>
> **Project:** Build and reason about a production-grade caching system for a Node.js application, covering HTTP caching, application caching, distributed caches, cache keys, TTLs, invalidation, consistency, stampede protection, negative caching, serialization, memory limits, tenant isolation, failure modes, observability, testing, and cost trade-offs.
>
> **Role perspective:** Principal JavaScript Engineer · Node.js Architect · Distributed Systems Engineer · Performance Engineer · Database Engineer · Reliability Engineer · Security Engineer
>
> **Status:** `[ ] Not Started`
>
> **Verification date:** 2026-09-11
>
> **Core rule:** **A cache is not merely a faster database. It is a derived copy with a different failure mode, lifecycle, consistency model, and cost profile. Every cache needs an explicit ownership, correctness, freshness, invalidation, and failure strategy.**

---

# 1. Project Mission

Extend the FocusBoard platform from Chapters 105–107.

Build a cache layer for:

```text
project metadata
task lists
permission-sensitive summaries
configuration
expensive reports
HTTP GET responses
session-like data where appropriate
```

The system should answer:

```text
What should be cached?
Where?
For how long?
Who owns freshness?
What invalidates it?
What happens when cache is down?
What happens during a stampede?
What happens after a deployment?
What happens when one tenant has huge traffic?
```

The cache must improve performance without silently becoming the source of incorrect authorization or stale business behavior.

---

# 2. Learning Objectives

By completing this chapter, you should be able to:

- Explain why caches exist.
- Explain cache locality.
- Distinguish client, browser, CDN, reverse-proxy, application, and database caches.
- Distinguish cache-aside, read-through, write-through, write-behind, and refresh-ahead.
- Design safe cache keys.
- Define TTL semantics.
- Understand freshness versus consistency.
- Design invalidation strategies.
- Understand cache stampede.
- Implement request coalescing.
- Implement stale-while-revalidate behavior.
- Handle negative caching.
- Prevent cache poisoning.
- Protect tenant isolation.
- Protect authorization-sensitive cache entries.
- Handle hot keys.
- Handle cache eviction.
- Handle cache misses.
- Design fallback behavior when cache infrastructure fails.
- Understand Redis-like distributed caches.
- Compare in-memory and distributed caches.
- Serialize values safely.
- Version cached data.
- Manage cache schema evolution.
- Warm caches deliberately.
- Avoid cache warming storms.
- Implement HTTP caching.
- Use ETags and conditional requests.
- Use `Cache-Control` intentionally.
- Understand CDN implications.
- Monitor hit rate and miss cost.
- Measure cache latency.
- Diagnose memory growth.
- Diagnose stampedes.
- Load test cache-heavy endpoints.
- Reason about invalidation correctness.
- Design cache security.
- Defend cache trade-offs at principal level.

---

# 3. Prerequisites

Recommended:

```text
Chapter 24 — Objects / Map / Set / WeakMap / WeakSet
Chapter 31–38 — Async JavaScript
Chapter 45–48 — Memory / GC / Engine
Chapter 55 — Fetch / HTTP Networking
Chapter 57 — Security Engineering
Chapter 58–63 — Node.js Runtime
Chapter 67 — Dependency Management
Chapter 79 — API Design
Chapter 81 — Database Integration
Chapter 82 — API Architecture
Chapter 83 — Observability
Chapter 84 — Reliability
Chapter 85 — Performance
Chapter 86–89 — Testing / Debugging / Review
Chapter 100 — Cost Model / Trade-offs
Chapter 101 — Production Scenarios
Chapter 105 — REST API
Chapter 106 — WebSocket Systems
Chapter 107 — Job Queue
```

---

# 4. Why Caching Exists

Suppose:

```text
database query = 100 ms
cache lookup = 2 ms
```

Caching can reduce:

```text
latency
database load
downstream traffic
CPU
cost
```

But introduces:

```text
staleness
invalidation
memory cost
operational complexity
failure modes
```

---

# 5. Cache Mental Model

Think:

```text
Source of Truth
      ↓
Derived Representation
      ↓
Cache
```

The cache is usually:

```text
reconstructible
```

The source of truth is:

```text
authoritative
```

Do not violate this distinction accidentally.

---

# 6. Cache Correctness

A cached value is useful only if:

```text
fast
+
sufficiently fresh
+
correct for this caller
```

---

# 7. Freshness

Ask:

```text
How old may this value be?
```

Examples:

```text
stock price → seconds or less
product catalog → minutes
feature config → seconds/minutes
avatar metadata → hours
static reference data → long-lived
```

Freshness is a product property.

---

# 8. Consistency

Ask:

```text
How close must the cache be to authoritative state?
```

Possible:

```text
strong
session
eventual
best effort
```

Do not claim strong consistency merely because a cache is updated on writes.

---

# 9. Cache Policy

For each cached object define:

```text
owner
key
value
freshness
TTL
invalidation
fallback
privacy
size
eviction
observability
```

---

# 10. Cache Locations

Common layers:

```text
browser cache
service worker cache
CDN
reverse proxy
application memory
distributed cache
database buffer/cache
```

---

# 11. Browser Cache

Good for:

```text
static assets
HTTP GET representations
```

Controlled by HTTP cache headers.

---

# 12. CDN

Good for:

```text
globally distributed public content
cacheable API responses
static assets
```

Trade-offs:

```text
propagation
invalidation
regional consistency
cost
```

---

# 13. Application Memory Cache

Example:

```js
const cache = new Map();
```

Strengths:

```text
extremely low latency
no network hop
simple
```

Weaknesses:

```text
per-process
lost on restart
memory pressure
inconsistent across instances
```

---

# 14. Distributed Cache

Example class:

```text
Redis
```

Strengths:

```text
shared
low latency
independent from Node process
```

Weaknesses:

```text
network dependency
operational dependency
serialization
failure
cost
```

---

# 15. Multi-Layer Cache

Possible:

```text
L1 local memory
      ↓
L2 distributed cache
      ↓
database
```

This can reduce distributed-cache traffic but adds:

```text
two invalidation layers
```

---

# 16. Cache Hierarchy

Think:

```text
fast/small
→
slower/larger
→
authoritative
```

Example:

```text
L1
→ L2
→ DB
```

---

# 17. Cache-Aside

Typical flow:

```text
read cache
→ miss
→ read DB
→ populate cache
→ return
```

This keeps cache management explicit.

---

# 18. Cache-Aside Example

```js
async function getProject(id) {
  const key = `project:${id}`;

  const cached = await cache.get(key);

  if (cached !== null) {
    return cached;
  }

  const project = await repository.findById(id);

  await cache.set(key, project, {
    ttl: 60
  });

  return project;
}
```

Production code must also handle:

```text
serialization
errors
concurrent misses
tenant scope
permissions
```

---

# 19. Read-Through

The cache layer itself loads missing data.

Concept:

```text
application
→ cache
→ cache loads source
```

Can simplify callers but hides cache/source behavior.

---

# 20. Write-Through

Write:

```text
application
→ cache
→ source
```

or a cache layer synchronously updates source.

Can improve read freshness but increases write complexity.

---

# 21. Write-Behind

Write to cache first, persist later.

This can improve throughput but dramatically increases durability and failure complexity.

Use only when semantics justify it.

---

# 22. Refresh-Ahead

Before TTL expiration:

```text
refresh
```

so hot keys remain warm.

Trade-off:

```text
background work
```

for:

```text
lower miss latency
```

---

# 23. Cache Policy Selection

Start with:

```text
cache-aside
```

unless there is a clear reason for another model.

---

# 24. Cache Key

A key should contain every dimension that changes the value.

Example:

```text
tenant:42:project:123
```

If permissions affect the result:

```text
tenant:42:user:7:project:123
```

or use a different architecture that keeps authorization outside the cached representation.

---

# 25. Cache-Key Rule

If two callers can see different values:

```text
they must not collide
```

in a shared cache entry unless the cached representation is intentionally permission-independent.

---

# 26. Tenant Isolation

Never use:

```text
project:123
```

if project IDs can overlap across tenants and the cache is shared.

Prefer:

```text
tenant:{tenantId}:project:{projectId}
```

where appropriate.

---

# 27. User-Specific Data

Do not cache:

```text
user-specific response
```

under:

```text
endpoint-only key
```

unless all callers should receive the same data.

---

# 28. Authorization-Sensitive Cache

A dangerous pattern:

```text
cache task list
```

without considering:

```text
who can see tasks
```

Possible solutions:

```text
permission-independent data cache
+
authorization at read time
```

or:

```text
permission-aware cache key
```

---

# 29. Cache Key Namespace

Use explicit namespaces:

```text
project:
task-list:
permissions:
report:
```

This makes operations safer.

---

# 30. Cache Version

Include schema/version when cached shape changes:

```text
project:v2:{id}
```

This can make deployment migration safer.

---

# 31. Key Length

Do not put huge JSON blobs in cache keys.

Normalize and hash large composite inputs when appropriate.

---

# 32. Deterministic Keys

Inputs should be normalized:

```text
sort filters
normalize case where semantics allow
canonicalize defaults
```

so equivalent requests map to the same key.

---

# 33. Example Query Keys

Requests:

```text
/tasks?status=open&sort=createdAt
```

and:

```text
/tasks?sort=createdAt&status=open
```

can be semantically identical.

Canonicalization prevents duplicate cache entries.

---

# 34. TTL

TTL answers:

```text
When should this cached representation stop being considered fresh?
```

---

# 35. TTL Is Not Invalidation

A 60-second TTL means:

```text
value can remain stale for up to roughly 60 seconds
```

depending on access and cache semantics.

It does not mean:

```text
changes are immediately reflected
```

---

# 36. Short TTL

Benefits:

```text
fresher
simpler
```

Costs:

```text
more misses
more source load
```

---

# 37. Long TTL

Benefits:

```text
higher hit rate
lower source load
```

Costs:

```text
staleness
harder correctness
```

---

# 38. TTL Jitter

If 10,000 keys expire simultaneously:

```text
stampede
```

Add small randomized TTL variation:

```text
base TTL
+
random jitter
```

---

# 39. Absolute Expiration

Store:

```text
expiresAt
```

This makes freshness explicit.

---

# 40. Sliding Expiration

TTL resets on access.

Useful for some session-like data.

Danger:

```text
hot stale value can remain forever
```

---

# 41. Hard vs Soft TTL

Hard TTL:

```text
after expiry
→ cannot serve
```

Soft TTL:

```text
after expiry
→ may serve stale while refreshing
```

---

# 42. Stale-While-Revalidate

Flow:

```text
fresh
→ return

stale-but-acceptable
→ return stale
→ refresh asynchronously

too old
→ fetch source
```

Useful when latency matters more than strict freshness.

---

# 43. Stale-While-Revalidate Safety

Never serve stale data when:

```text
security
permissions
critical financial state
```

requires fresher guarantees.

---

# 44. Cache Stampede

Without protection:

```text
key expires
→ 10,000 requests miss
→ 10,000 DB queries
```

---

# 45. Request Coalescing

For one process:

```text
same key
→ share one in-flight fetch
```

Concept:

```js
const inFlight = new Map();
```

---

# 46. Single-Flight

Concept:

```text
first requester
→ starts load

others
→ await same promise
```

---

# 47. Single-Flight Cleanup

Always delete:

```text
inFlight.delete(key)
```

in a `finally` path after the promise settles.

---

# 48. Distributed Stampede

Local single-flight does not prevent:

```text
Node A
Node B
Node C
```

from all fetching the same missing key.

Use distributed coordination only when necessary.

---

# 49. Distributed Lock

Concept:

```text
SET lock:key unique short TTL
```

or equivalent primitive.

Be extremely careful with:

```text
lock expiry
owner identity
failure
stale lock
```

---

# 50. Do Not Confuse Lock With Correctness

A cache lock reduces duplicate work.

It does not automatically make distributed writes correct.

---

# 51. Hot Keys

A single key may receive:

```text
millions of reads
```

This can overload:

```text
one cache node
one database key
one lock
```

---

# 52. Hot-Key Strategies

Options:

```text
replicate
L1 caches
request coalescing
TTL refresh
partition
precompute
```

Choose based on workload.

---

# 53. Negative Caching

Cache known misses:

```text
project does not exist
```

This protects the database from repeated nonexistent lookups.

---

# 54. Negative Cache Risk

If an object is created shortly after:

```text
negative entry
```

clients may continue seeing:

```text
not found
```

for the negative TTL.

Keep negative TTLs appropriately short.

---

# 55. Error Caching

Avoid caching arbitrary errors.

Temporary source failures should not become long-lived cached failures.

---

# 56. Cache Poisoning

An attacker may attempt to place malicious/incorrect data into a cache.

Protect:

```text
key ownership
input normalization
authorization
cache write paths
```

---

# 57. HTTP Cache Poisoning

Dangerous dimensions include:

```text
Host
Authorization
cookies
query normalization
content negotiation
```

Ensure cache keys vary appropriately.

---

# 58. Authorization and HTTP Caching

Private responses must not accidentally become shared cached responses.

Understand:

```text
private
public
no-store
vary
```

and choose intentionally.

---

# 59. Cache-Control

Common concepts:

```text
max-age
s-maxage
no-cache
no-store
private
public
stale-while-revalidate
```

The exact semantics should be checked against current HTTP caching specifications when implementing infrastructure.

---

# 60. `no-cache` vs `no-store`

They are not synonyms.

Conceptually:

```text
no-cache
→ may store, but must revalidate before reuse

no-store
→ do not store the response
```

---

# 61. `private`

Indicates a response is intended for a single user/private cache rather than shared caches.

---

# 62. `public`

Allows shared caching where other conditions permit it.

Do not mark sensitive data public.

---

# 63. `Vary`

Tells caches that representation can change based on request headers.

Example:

```text
Vary: Accept-Encoding
```

Other headers may matter depending on representation selection.

---

# 64. Cache-Control for APIs

Do not assume all API responses should be:

```text
no-store
```

or:

```text
max-age=...
```

Design endpoint by endpoint.

---

# 65. ETag

An ETag identifies a representation version.

Example:

```http
ETag: "abc123"
```

---

# 66. Conditional GET

Client:

```http
If-None-Match: "abc123"
```

Server may respond:

```http
304 Not Modified
```

when representation is unchanged.

---

# 67. Why ETag Helps

Reduces:

```text
response bytes
serialization
network transfer
```

even when the client must validate freshness.

---

# 68. Weak vs Strong Validators

HTTP supports different validator semantics.

Choose based on whether byte-for-byte or semantic representation identity is required.

---

# 69. Cache Invalidation

Common strategies:

```text
TTL
explicit deletion
versioned keys
event-driven invalidation
write-through update
```

---

# 70. Explicit Invalidation

After:

```text
UPDATE task
```

invalidate:

```text
task:{id}
task-list:{project}
project-summary:{project}
```

The challenge is knowing every dependent key.

---

# 71. Invalidation Graph

Example:

```text
task
 ↓
task list
 ↓
project summary
 ↓
dashboard
```

One write can invalidate many derived values.

---

# 72. Invalidation Complexity

As the number of derived representations grows:

```text
invalidation surface
```

grows.

This is why caching can become an architectural cost.

---

# 73. Versioned Data

Instead of deleting every old key:

```text
project:v2:123
```

or a version token can make old entries unreachable.

---

# 74. Version Bump

Example:

```text
projectVersion = 10
```

Key:

```text
project:{id}:v{version}
```

Changing version invalidates prior representations logically.

---

# 75. Event-Driven Invalidation

DB change:

```text
task.updated
```

publishes:

```text
invalidate task list
invalidate summary
```

This connects Chapter 106/107 with Chapter 108.

---

# 76. Invalidation Race

Sequence:

```text
read DB old value
write new DB value
populate cache old value
```

This can reintroduce stale data after an invalidation.

---

# 77. Safe Write Flow

One strategy:

```text
write source
→ invalidate cache
```

But even this can race with concurrent readers.

---

# 78. Write-Then-Delete Race

Consider:

```text
Reader A reads old DB value
Writer updates DB
Writer deletes cache
Reader A writes old value into cache
```

Now cache is stale.

This is a classic cache-aside race.

---

# 79. Preventing Stale Refill

Possible strategies:

```text
versioned values
write-through
locking
compare versions
delete-after-read-window
```

Choose based on correctness requirements.

---

# 80. Versioned Cache Value

Store:

```json
{
  "version": 42,
  "value": {...}
}
```

Only allow newer versions to replace older ones where the cache technology and protocol permit safe comparisons.

---

# 81. Source Version

A DB row can expose:

```text
updatedAt
version
sequence
```

to identify freshness.

---

# 82. Cache Consistency Modes

Possible:

```text
eventual
read-your-writes
monotonic reads
strong-ish with synchronous invalidation
```

Define what users actually need.

---

# 83. Read-Your-Writes

After:

```text
user updates task
```

the same user should not immediately see:

```text
old task
```

Possible strategies:

```text
invalidate
write cache
request bypass
version check
```

---

# 84. Cache Bypass

For critical reads immediately after mutation:

```text
cache bypass
```

can be simpler than complex invalidation.

Use selectively.

---

# 85. Session Consistency

Some data needs:

```text
session-aware freshness
```

but not global strong consistency.

---

# 86. Authorization Cache

Caching permission decisions can improve latency but is dangerous.

Need:

```text
short TTL
invalidation
version
explicit policy
```

---

# 87. Negative Authorization Cache

Caching:

```text
user cannot access resource
```

can become stale after permission changes.

Treat carefully.

---

# 88. Permission Version

A user's permissions can have a version:

```text
permissionVersion = 12
```

Cache key can include version.

---

# 89. Cache Dependency

Ask:

```text
What data must change before this cache becomes wrong?
```

Document those dependencies.

---

# 90. Cache Contract

For every key family:

```text
source
owner
writer
reader
TTL
invalidation
failure
sensitivity
size
```

---

# 91. Serialization

Possible formats:

```text
JSON
MessagePack
CBOR
binary protocol
```

JSON is simple but can have:

```text
CPU
size
type fidelity
```

trade-offs.

---

# 92. JavaScript Type Fidelity

JSON does not preserve every JavaScript value exactly.

Examples include:

```text
undefined
BigInt
Map
Set
Date identity/type semantics
```

Use explicit serialization contracts.

---

# 93. Do Not Serialize Arbitrary Objects

Prefer:

```text
plain JSON-compatible data
```

rather than storing live application objects.

---

# 94. Serialization Version

Cache schemas can change.

Use:

```text
valueVersion
```

or key-versioning.

---

# 95. Schema Evolution

Possible strategies:

```text
dual reader
versioned keys
flush old cache
lazy migration
```

---

# 96. Cache Flush

Global flush is simple but dangerous.

It can cause:

```text
massive miss storm
```

---

# 97. Safe Cache Warmup

After flush:

```text
warm only high-value keys
```

instead of:

```text
warm everything
```

---

# 98. Cache Warming

Warm:

```text
top projects
popular configuration
known hot reference data
```

based on measured traffic.

---

# 99. Warming From Logs

Use production telemetry:

```text
top key families
```

to identify useful warm candidates.

---

# 100. Cold Start

After deployment:

```text
L1 empty
L2 partially warm
```

Expect:

```text
higher source load
```

and provision accordingly.

---

# 101. Cache Warming Storm

Do not let every instance simultaneously warm:

```text
10,000 keys
```

with no coordination.

---

# 102. Eviction

A cache may be bounded by:

```text
memory
number of entries
TTL
policy
```

---

# 103. LRU

Least Recently Used:

```text
evict entries not accessed recently
```

Good for locality-driven workloads.

---

# 104. LFU

Least Frequently Used:

```text
evict entries accessed least often
```

Useful when frequency matters more than recency.

---

# 105. TTL-Based Eviction

Entries expire based on time.

Simple but can retain cold data until TTL.

---

# 106. Cache Eviction Is Not Business Invalidation

An eviction means:

```text
cache no longer stores value
```

It does not mean:

```text
source changed
```

---

# 107. Memory Pressure

An in-process cache competes with:

```text
application heap
buffers
request data
module state
```

Bound it.

---

# 108. Local Cache Size

Use explicit:

```text
maximum entries
maximum estimated bytes
TTL
```

rather than unlimited `Map`.

---

# 109. L1 Cache Danger

Large local caches across many instances can create:

```text
stale replicas
memory duplication
warmup cost
```

---

# 110. Distributed Cache Latency

Even a fast network cache adds:

```text
network
serialization
connection pool
```

compared with local memory.

---

# 111. Cache Connection Pool

Treat the cache connection as a shared resource.

Monitor:

```text
connections
pending commands
timeouts
latency
```

---

# 112. Cache Failure

Never assume:

```text
cache is always available
```

---

# 113. Fail-Open

On cache failure:

```text
read source
```

Good when:

```text
correctness > performance
```

and source capacity can handle fallback load.

---

# 114. Fail-Closed

Return:

```text
error
```

when cache is unavailable.

Rare for ordinary data caching but possible when the cache stores critical coordination or authorization state.

---

# 115. Cache Failure Amplification

If cache dies:

```text
all reads → DB
```

Database may collapse.

---

# 116. Cache Failure Protection

Use:

```text
timeouts
circuit breaker
bounded fallback
load shedding
request coalescing
source rate limits
```

---

# 117. Fallback Budget

Do not allow unlimited cache misses to hammer the database.

---

# 118. Source Protection

When cache is unavailable:

```text
limit concurrent source fetches
```

and:

```text
prioritize critical requests
```

---

# 119. Cache Outage

Treat cache outage as:

```text
capacity event
```

not merely:

```text
dependency error
```

---

# 120. Partial Cache Failure

A distributed cache may have:

```text
one node unhealthy
```

while others are healthy.

Understand the cache client's failure and routing behavior.

---

# 121. Availability vs Consistency

A cache can improve:

```text
read availability
```

if the application can fallback.

But fallback can overload the source.

---

# 122. Cache as Capacity Multiplier

Caching can multiply effective read capacity:

```text
DB
→ fewer requests
```

but only while:

```text
hit rate
```

remains high.

---

# 123. Hit Rate

Basic:

```text
hit rate =
hits / (hits + misses)
```

Useful but incomplete.

---

# 124. Miss Penalty

A 95% hit rate is not necessarily good.

If misses are:

```text
2 seconds
```

and hits:

```text
2 ms
```

the impact can still be substantial.

---

# 125. Weighted Hit Rate

Consider:

```text
hot endpoint
cold endpoint
expensive miss
cheap miss
```

Optimize based on:

```text
saved cost
```

not only percentage.

---

# 126. Cache Effectiveness

Useful metric:

```text
estimated source work avoided
```

---

# 127. Cache Latency

Track:

```text
p50
p95
p99
```

for:

```text
get
set
delete
```

---

# 128. Cache Serialization Cost

The cache can be fast while:

```text
JSON.stringify
JSON.parse
```

becomes the bottleneck.

Measure separately.

---

# 129. Large Values

Large cache values create:

```text
network cost
memory cost
serialization cost
eviction pressure
```

Prefer compact representations.

---

# 130. Fragmentation

Caching many large independent objects can produce inefficient memory use depending on the cache implementation.

Measure actual cache memory.

---

# 131. Key Explosion

A poorly designed key can create:

```text
millions of nearly unique entries
```

For example:

```text
search:{timestamp}:{random}
```

Avoid unbounded cardinality.

---

# 132. Query Caching

Caching arbitrary search queries can produce enormous key counts.

Only cache queries with:

```text
high reuse
bounded cardinality
acceptable staleness
```

---

# 133. Pagination Cache

Caching pages can create:

```text
page 1
page 2
page 3
...
```

and invalidation complexity.

Cursor-based results can also become stale.

---

# 134. Cache Lists

A list cache can be:

```text
easy to read
hard to invalidate
```

For rapidly mutating lists, caching individual items may be safer.

---

# 135. Aggregate Cache

Example:

```text
project summary
```

is often a good candidate if:

```text
expensive to compute
frequently read
tolerates bounded staleness
```

---

# 136. Precomputed Aggregate

Instead of:

```text
COUNT + SUM + JOIN
```

on every request:

```text
maintain summary
```

and cache it.

But now there are:

```text
more write paths
```

and consistency risks.

---

# 137. Cache + Database Index

Do not use a cache to compensate for fundamentally poor database queries.

First consider:

```text
query plan
index
projection
pagination
```

---

# 138. Cache + N+1

Caching may hide N+1 during development but still produce:

```text
miss storms
```

when cold.

Fix query amplification rather than relying entirely on cache.

---

# 139. Cache + Transactions

Do not assume:

```text
DB transaction commit
```

and:

```text
cache update
```

are one atomic operation.

---

# 140. Cache Invalidation After Commit

A common strategy is:

```text
commit DB
→ invalidate
```

but concurrent readers still require analysis of races.

---

# 141. Transactional Event

Another approach:

```text
DB transaction
→ outbox
→ invalidation worker
```

This makes invalidation durable but introduces propagation delay.

---

# 142. Invalidation Latency

Measure:

```text
source mutation
→ cache invalidated
```

---

# 143. Freshness SLA

Define:

```text
maximum tolerated stale duration
```

for each cached resource.

---

# 144. Eventual Consistency Budget

Example:

```text
project summary
may be 5 seconds stale
```

That is a business decision.

---

# 145. Strong Freshness Endpoint

Some APIs can expose:

```text
?fresh=true
```

or a distinct endpoint that bypasses cache.

Use sparingly to avoid turning the cache into an optional hint everywhere.

---

# 146. Cache Headers

For public resources, coordinate:

```text
browser
CDN
origin
```

through HTTP caching semantics.

---

# 147. CDN Purge

A CDN can require:

```text
purge key
tag invalidation
versioned URL
```

Use versioned assets where practical.

---

# 148. Immutable Assets

Good cache candidate:

```text
app.8f4c2.js
```

because content identity is in the URL.

This dramatically simplifies invalidation.

---

# 149. Cache Busting

Preferred:

```text
content hash
```

over:

```text
manual random query parameter
```

for immutable build artifacts.

---

# 150. Sensitive API Response

Do not cache globally:

```text
GET /account
```

unless the response is explicitly safe for shared caching.

---

# 151. Authorization Header

Shared HTTP caches need careful handling of responses affected by authorization.

Never assume:

```text
Authorization
```

automatically makes every infrastructure cache safe.

Understand the actual cache configuration and response directives.

---

# 152. Cache-Control Header Example

Concept:

```http
Cache-Control: private, max-age=30
```

for user-specific short-lived cacheability.

---

# 153. Public Response Example

Concept:

```http
Cache-Control: public, max-age=60, s-maxage=300
```

Only if the response is truly safe for shared caching.

---

# 154. `s-maxage`

Shared-cache freshness can differ from browser freshness.

Useful when:

```text
CDN
```

should cache longer than:

```text
browser
```

---

# 155. Stale-While-Revalidate HTTP

Shared caches can serve stale content under configured policies while revalidating.

Use only where stale content is acceptable.

---

# 156. Cache Tags

Some CDNs support invalidation by:

```text
tag
```

This can simplify dependency invalidation.

Do not assume all caches support it.

---

# 157. Application Cache Tags

You can model relationships:

```text
project:42
tag:project:42
tag:tenant:7
```

but tag indexes add storage and invalidation work.

---

# 158. Cache Dependency Registry

An application can track:

```text
cache key
→ dependency set
```

but this can become a system of its own.

Prefer simpler architectures when possible.

---

# 159. Key Composition

Never build:

```js
`user:${req.query.userId}`
```

without validating that the caller may access that user.

Cache keys are not authorization.

---

# 160. Cache Write Authorization

A compromised internal path might write a misleading value to a shared key.

Protect cache write APIs and namespaces.

---

# 161. Cache Integrity

For security-critical cached values, consider:

```text
version
signed/integrity-protected data
```

when threat model requires it.

---

# 162. Cache Encryption

Use infrastructure or application encryption where sensitive cached data requires it.

Remember:

```text
encryption does not solve incorrect authorization
```

---

# 163. Secret Caching

Avoid caching:

```text
long-lived credentials
```

without a strong need and expiration strategy.

Prefer secure secret managers.

---

# 164. Token Caching

If caching token validation results:

```text
short TTL
revocation strategy
issuer/audience validation
```

must remain correct.

---

# 165. Permission Revocation

Cache can keep access alive after revocation.

Security-sensitive permission caches need:

```text
short freshness
invalidation
versioning
```

or another explicit revocation strategy.

---

# 166. Cache Bypass During Incident

Operational tooling may need:

```text
cache bypass
```

for diagnosing stale data.

Make it controlled and observable.

---

# 167. Debug Endpoint

Do not expose unrestricted cache inspection publicly.

Operational cache inspection should require:

```text
authentication
authorization
```

---

# 168. Cache Flush Endpoint

Never expose:

```text
DELETE /cache/all
```

to ordinary clients.

---

# 169. Distributed Lock Safety

If using a lock for stampede prevention:

```text
owner token
expiry
release only by owner
```

are minimum concepts.

---

# 170. Dogpile Prevention

Alternative to locks:

```text
stale-while-revalidate
```

can avoid many requests waiting for one loader.

---

# 171. Request Coalescing vs Lock

Single-flight:

```text
same process
```

Distributed lock:

```text
multiple processes
```

Neither guarantees business correctness by itself.

---

# 172. Cache Stampede Prevention Stack

Possible:

```text
TTL jitter
+
single-flight
+
stale-while-revalidate
+
bounded concurrency
```

This is often more resilient than relying on one mechanism.

---

# 173. Early Refresh

Refresh before TTL:

```text
remaining TTL < threshold
```

helps keep hot keys warm.

---

# 174. Refresh Ownership

Only one worker/process should refresh a key at a time when refresh is expensive.

---

# 175. Negative Cache Stampede

A nonexistent item can itself become a hot miss.

Negative caching can reduce repeated source load.

---

# 176. Cache Miss Storm

A deployment changes keys:

```text
v1 → v2
```

All requests miss simultaneously.

Use:

```text
gradual rollout
dual read
prewarm
```

when necessary.

---

# 177. Cache Namespace Migration

Changing key format can invalidate the entire cache.

Document:

```text
old namespace
new namespace
migration
```

---

# 178. Dual Read

Possible transition:

```text
read v2
→ fallback v1
→ populate v2
```

Useful for safe cache schema changes.

---

# 179. Dual Write

Possible:

```text
write v1
+
write v2
```

but doubles work.

Use only temporarily.

---

# 180. Cache Warming After Deploy

Do not immediately preload every key.

Use:

```text
top traffic
```

or:

```text
progressive warming
```

---

# 181. Cache Size Planning

Estimate:

```text
entries
×
average value size
+
metadata
+
fragmentation
```

and leave headroom.

---

# 182. L1 + L2 Memory Planning

If every process has a 500 MB L1 cache and there are:

```text
20 instances
```

the aggregate duplicated memory is:

```text
10 GB
```

before considering overhead.

---

# 183. Cache Eviction Churn

If memory is too small:

```text
set
→ evict
→ miss
→ set
→ evict
```

can cause cache thrashing.

---

# 184. Working Set

A useful question:

```text
What percentage of useful traffic fits in the cache?
```

---

# 185. Temporal Locality

Caching works well when:

```text
recently used data
```

is likely to be reused.

---

# 186. Spatial Locality

Less direct in key/value caching, but related accesses can sometimes share structures or prefetch/warm related values.

---

# 187. Cache Admission

Not every value deserves caching.

Large one-off objects can pollute cache.

Consider:

```text
minimum reuse
size threshold
cost to recompute
```

---

# 188. Cache Pollution

Caching low-reuse values can evict highly valuable hot data.

---

# 189. Cost-Aware Caching

Cache entries with:

```text
high read frequency
high miss cost
reasonable memory footprint
acceptable staleness
```

first.

---

# 190. Serialization CPU

If:

```text
DB = 20 ms
cache = 1 ms
serialization = 8 ms
```

cache may still provide a significant speedup, but optimizing serialization could be equally valuable.

---

# 191. Compression

Compressing cache values may reduce network/memory cost but increase CPU.

Benchmark.

---

# 192. Partial Data

Instead of caching huge objects:

```text
cache summary
```

or:

```text
cache selected fields
```

if consumers need only a subset.

---

# 193. Cache-Aside with Repository

Architecture:

```text
service
 ↓
cached repository
 ↓
repository
 ↓
database
```

This can keep cache details out of domain logic.

---

# 194. Cached Repository Boundary

Concept:

```js
async function findProject(id, context) {
  const key = keyForProject(context, id);

  const hit = await cache.get(key);

  if (hit) {
    return decode(hit);
  }

  const value =
    await repository.findProject(
      id,
      context
    );

  await cache.set(
    key,
    encode(value),
    { ttl: 60 }
  );

  return value;
}
```

Ensure `context` contains all data needed for safe isolation.

---

# 195. Cache Adapter

Define an interface:

```js
const cache = {
  async get(key) {},
  async set(key, value, options) {},
  async delete(key) {}
};
```

The domain should not depend on Redis-specific commands.

---

# 196. Cache Errors

Treat cache errors separately from source errors:

```text
cache unavailable
```

often should not become:

```text
500
```

for a cache-aside read.

---

# 197. Cache Timeout

Use short cache timeouts.

A cache that hangs for:

```text
5 seconds
```

can be worse than a cache miss.

---

# 198. Cache Circuit Breaker

If cache infrastructure is failing:

```text
stop attempting every cache request
```

temporarily and protect the source.

---

# 199. Cache Error Budget

Define how much performance degradation is acceptable during cache outage.

---

# 200. Health Checks

Do not make liveness depend on cache availability.

Otherwise:

```text
cache outage
→ every instance unhealthy
→ restart storm
```

---

# 201. Readiness and Cache

Readiness may or may not depend on cache based on whether cache is required for acceptable service.

---

# 202. Cache Metrics

Core:

```text
cache_requests_total
cache_hits_total
cache_misses_total
cache_errors_total
cache_get_latency
cache_set_latency
cache_delete_latency
```

---

# 203. Hit Ratio by Key Family

Track:

```text
project
task-list
summary
report
```

separately.

---

# 204. Miss Reason

Possible:

```text
absent
expired
evicted
bypass
version mismatch
error
```

Do not add excessive high-cardinality labels.

---

# 205. Stampede Metrics

Track:

```text
coalesced_requests
refreshes
lock_contention
duplicate_loads
```

---

# 206. Cache Size Metrics

Track:

```text
entries
bytes
evictions
memory utilization
```

---

# 207. Hot Key Detection

Measure:

```text
requests/key
```

through logs or sampled telemetry rather than creating a metric label for every key.

---

# 208. Source Load Correlation

Compare:

```text
cache miss rate
```

with:

```text
DB query rate
```

This helps identify whether caching actually reduces source load.

---

# 209. Cache SLO

Example:

```text
p95 cache get < 10 ms
```

This is illustrative.

Actual target depends on deployment.

---

# 210. Freshness SLO

Example:

```text
99% of task summaries no more than 10 seconds stale
```

Again, product-specific.

---

# 211. HTTP Cache Testing

Test:

```text
Cache-Control
ETag
If-None-Match
304
private/public
Vary
```

---

# 212. Application Cache Testing

Test:

```text
hit
miss
expiration
invalidation
cache failure
stampede
```

---

# 213. Consistency Testing

Simulate:

```text
read old
write new
invalidate
read old refill
```

and verify the chosen strategy.

---

# 214. Concurrency Test

Simulate:

```text
100 concurrent misses
```

Expected:

```text
small number of source reads
```

when single-flight is designed.

---

# 215. Distributed Stampede Test

Run:

```text
10 instances
```

with:

```text
same cold hot key
```

Measure duplicate source loads.

---

# 216. Cache Outage Test

Disable cache.

Verify:

```text
fallback
source protection
latency
error rate
```

---

# 217. Source Outage Test

Cache remains available.

Determine:

```text
stale data
```

policy.

---

# 218. Source + Cache Outage

Both unavailable.

Expected behavior:

```text
controlled failure
```

not:

```text
infinite retries
```

---

# 219. Eviction Test

Fill cache beyond capacity.

Verify:

```text
eviction policy
```

matches expectations.

---

# 220. Hot-Key Test

One key receives:

```text
90% traffic
```

Measure:

```text
cache node
network
CPU
```

---

# 221. Large Value Test

Cache a large object.

Measure:

```text
serialization
memory
network
latency
```

---

# 222. Key Cardinality Test

Generate:

```text
1M distinct keys
```

and inspect:

```text
memory
evictions
storage
```

---

# 223. Tenant Isolation Test

Cache:

```text
tenant A
tenant B
```

with overlapping resource IDs.

Verify no cross-tenant reads.

---

# 224. Authorization Test

Change user permission after cache population.

Verify stale authorization does not produce forbidden access.

---

# 225. Cache Poisoning Test

Attempt:

```text
malformed query
header variation
untrusted key
```

and verify shared responses remain safe.

---

# 226. Deployment Test

Change:

```text
cache schema v1 → v2
```

while old instances are still running.

Verify compatibility.

---

# 227. Warmup Test

Flush cache.

Measure:

```text
source spike
recovery
```

Then test progressive warmup.

---

# 228. Failure Injection

Simulate:

```text
cache timeout
cache connection reset
cache node failure
serialization failure
source DB slow
```

---

# 229. Load Test

Compare:

```text
cache disabled
```

vs:

```text
cache enabled
```

Measure:

```text
latency
DB QPS
CPU
network
memory
```

---

# 230. Cache Benefit Calculation

Approximate saved source work:

```text
saved requests
≈
cache hits
```

But calculate actual resource savings from:

```text
query cost
payload size
CPU
latency
```

---

# 231. Benchmarking

Measure:

```text
local Map
distributed cache
database
```

under realistic payloads.

Do not benchmark empty strings and infer production behavior.

---

# 232. Microbenchmark Trap

A microbenchmark showing:

```text
cache = 0.3 ms
DB = 1 ms
```

may not represent production:

```text
network
serialization
contention
TLS
connection pooling
```

---

# 233. Benchmark Method

Use:

```text
representative payloads
warm/cold states
concurrency
p50/p95/p99
failure cases
```

---

# 234. Cache Cost Model

Costs:

```text
cache infrastructure
network
serialization
memory
invalidation engineering
observability
staleness
incidents
```

Benefits:

```text
lower latency
lower DB load
higher throughput
lower downstream cost
```

---

# 235. When Not to Cache

Do not cache when:

```text
reuse is low
source is cheap
data changes constantly
staleness is unacceptable
cache operational cost exceeds benefit
```

---

# 236. Cache Myth

Myth:

```text
"If DB is slow, add Redis."
```

Reality:

```text
first understand why the DB is slow
```

Possible causes:

```text
missing index
N+1
bad query
large response
connection pool
lock contention
```

---

# 237. Cache Myth

Myth:

```text
"High hit rate means the cache works."
```

Reality:

```text
Correctness + saved source work + latency + cost
```

matter.

---

# 238. Cache Myth

Myth:

```text
"TTL solves invalidation."
```

Reality:

```text
TTL defines bounded freshness
```

but does not solve every correctness race.

---

# 239. Cache Myth

Myth:

```text
"Cache is just memory."
```

Reality:

```text
cache is a consistency and capacity architecture
```

---

# 240. Cache Myth

Myth:

```text
"Redis is infinitely fast."
```

Reality:

```text
network + contention + serialization + capacity
```

still matter.

---

# 241. Cache Myth

Myth:

```text
"Deleting a cache key means fresh data."
```

Reality:

Concurrent readers can repopulate stale data.

---

# 242. Cache Myth

Myth:

```text
"One global cache is simpler."
```

Reality:

Shared caches can create:

```text
tenant isolation
hot key
blast radius
operational
```

concerns.

---

# 243. Production Scenario — Cache Outage

Symptoms:

```text
DB QPS spikes
API latency rises
```

Response:

```text
circuit breaker
source protection
load shedding
recovery monitoring
```

Do not immediately add database capacity without understanding whether traffic is cache-induced.

---

# 244. Production Scenario — Stampede

Symptoms:

```text
one key expires
DB CPU spikes
```

Investigate:

```text
TTL alignment
hot key
single-flight
refresh policy
```

---

# 245. Production Scenario — Cross-User Leak

Symptoms:

```text
user B receives user A data
```

Priority:

```text
contain
disable affected cache
audit
fix key/HTTP policy
test
```

---

# 246. Production Scenario — Stale Permissions

Symptoms:

```text
revoked user still accesses data
```

Investigate:

```text
authorization cache
TTL
invalidation
version
```

---

# 247. Production Scenario — Cache Thrashing

Symptoms:

```text
hit rate low
evictions high
```

Investigate:

```text
working set
value size
key cardinality
TTL
```

---

# 248. Production Scenario — Memory Explosion

Symptoms:

```text
Node heap grows
```

Investigate:

```text
unbounded L1
large values
key cardinality
in-flight refreshes
```

---

# 249. Production Scenario — Deployment Miss Storm

Symptoms:

```text
all instances deployed
DB overloaded
```

Investigate:

```text
key namespace changed
version bump
```

Mitigate:

```text
warm
roll out gradually
dual read
```

---

# 250. Production Scenario — Cache Corruption

Symptoms:

```text
deserialization failures
```

Investigate:

```text
schema version
partial writes
serialization library
manual operator changes
```

---

# 251. Production Scenario — Stale Refill Race

Sequence:

```text
R1 reads old source
W updates source
W deletes cache
R1 writes old source to cache
```

This demonstrates why:

```text
invalidate after write
```

is not always sufficient.

---

# 252. Production Scenario — Hot Key

One resource receives extreme reads.

Possible solutions:

```text
L1
replication
request coalescing
refresh-ahead
precomputation
```

---

# 253. Production Scenario — Cache Dependency Outage

The app waits:

```text
cache timeout = 5 sec
```

for every request.

The cache becomes slower than the DB.

Fix:

```text
shorter timeout
fallback
circuit breaker
```

---

# 254. Production Scenario — Cache Flush

Operator flushes all keys.

Result:

```text
DB overload
```

Recovery:

```text
protect DB
progressive warm
reduce traffic
```

---

# 255. Architecture

Recommended boundary:

```text
HTTP
 ↓
Application Service
 ↓
Cached Repository / Cache Adapter
 ↓
Source Repository
 ↓
Database
```

Cross-cutting:

```text
metrics
logging
tracing
security
```

---

# 256. Cache Interface

Concept:

```js
class CacheAdapter {
  async get(key) {}
  async set(key, value, options) {}
  async delete(key) {}
}
```

The concrete implementation may use:

```text
Map
Redis
cloud cache
```

---

# 257. Cache Key Builder

Concept:

```js
function projectKey({
  tenantId,
  projectId
}) {
  return `project:v2:${tenantId}:${projectId}`;
}
```

Keep key construction centralized.

---

# 258. Cache Decode

Concept:

```js
function decodeProject(raw) {
  const value = JSON.parse(raw);

  if (value.version !== 2) {
    throw new Error("Unsupported cache version");
  }

  return value.data;
}
```

Production code should classify cache corruption separately.

---

# 259. Single-Flight Example

Concept:

```js
const inFlight = new Map();

async function getOrLoad(key, loader) {
  const cached = await cache.get(key);

  if (cached !== null) {
    return decode(cached);
  }

  if (inFlight.has(key)) {
    return inFlight.get(key);
  }

  const promise = (async () => {
    try {
      const value = await loader();
      await cache.set(
        key,
        encode(value),
        { ttl: 60 }
      );
      return value;
    } finally {
      inFlight.delete(key);
    }
  })();

  inFlight.set(key, promise);

  return promise;
}
```

This protects only callers sharing the same process.

---

# 260. Stale Refresh Example

Concept:

```js
async function readWithSoftTtl(key, loader) {
  const entry = await cache.get(key);

  if (!entry) {
    return loadFresh(key, loader);
  }

  if (entry.expiresAt > Date.now()) {
    return entry.value;
  }

  queueRefresh(key, loader);

  return entry.value;
}
```

A production system needs:

```text
maximum staleness
refresh dedupe
failure handling
```

---

# 261. HTTP ETag Example

Concept:

```js
const etag =
  createRepresentationTag(body);

res.setHeader(
  "ETag",
  etag
);

if (
  req.headers["if-none-match"] === etag
) {
  res.statusCode = 304;
  res.end();
  return;
}
```

Do not invent ETag semantics without defining representation identity.

---

# 262. Track A — Core Theory

Study:

```text
cache hierarchy
cache-aside
read-through
write-through
write-behind
refresh-ahead
cache keys
TTL
freshness
consistency
invalidation
stampede
single-flight
distributed locks
hot keys
negative caching
eviction
serialization
HTTP caching
ETag
Cache-Control
CDN
tenant isolation
failure modes
observability
cost
```

---

# 263. Track B — Implementation

Build:

```text
cache interface
Map-based L1
distributed L2 adapter
key builders
serialization
TTL
cache-aside
single-flight
negative cache
invalidation
soft TTL
refresh
metrics
fallback
circuit breaker
HTTP caching
ETag
cache warming
tests
load tests
failure injection
```

---

# 264. Track C — Interview / Reasoning

Defend:

```text
Why cache?
Why this data?
Why this TTL?
Why cache-aside?
Why not write-through?
Why this key?
How do you prevent cross-tenant leaks?
How do you avoid stampede?
What happens if cache fails?
What happens after write?
How do you handle hot keys?
How do you invalidate lists?
How do you migrate cache schema?
How do you size the cache?
How do you prove the cache is useful?
```

---

# 265. Mastery Gate

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

# 266. Implementation Progression

## Stage 1 — Guided

Build:

```text
Map cache
get
set
delete
TTL
```

## Stage 2 — Partially Guided

Add:

```text
cache-aside
key builders
serialization
metrics
```

## Stage 3 — No Reference

Add:

```text
single-flight
negative cache
invalidation
```

## Stage 4 — Edge-Case Hardened

Add:

```text
stale refill race
stampede
hot key
cache outage
memory bounds
tenant isolation
```

## Stage 5 — Production Grade

Add:

```text
distributed cache
HTTP caching
ETag
warming
schema migration
load tests
failure injection
operational playbook
```

---

# 267. Debugging Exercises

## Exercise 1 — Cross-Tenant Collision

Keys:

```text
project:123
project:123
```

belong to different tenants.

Find the security flaw.

---

## Exercise 2 — Cache Stampede

One hot key expires.

```text
1,000 requests
→ 1,000 DB reads
```

Design protection.

---

## Exercise 3 — Stale Refill

Sequence:

```text
read old
write new
delete
refill old
```

Identify the race.

---

## Exercise 4 — Cache Outage

Cache takes:

```text
2 seconds
```

per request.

DB takes:

```text
100 ms
```

Design fallback.

---

## Exercise 5 — Permission Revocation

A user's permission changes.

Cached authorization remains:

```text
valid for 10 minutes
```

Determine the security issue.

---

## Exercise 6 — Key Explosion

Search endpoint creates:

```text
unique random cache key
```

for every request.

Identify why hit rate is poor.

---

## Exercise 7 — Eviction Thrashing

Cache:

```text
100 MB
```

working set:

```text
2 GB
```

Determine expected behavior.

---

## Exercise 8 — Deployment

Cache key changes:

```text
v1 → v2
```

DB overload occurs.

Design safer rollout.

---

# 268. Code Review Exercise

Review:

```js
const key = `user:${req.params.id}`;

const cached = await redis.get(key);

if (cached) {
  return res.json(JSON.parse(cached));
}

const user =
  await db.users.findById(req.params.id);

await redis.set(key, JSON.stringify(user));

res.json(user);
```

Find:

```text
authorization
tenant isolation
TTL
serialization errors
cache failure
key versioning
data exposure
```

---

# 269. Code Review Exercise

Review:

```js
await db.tasks.update(task);

await redis.del(`task:${task.id}`);
```

Explain why this still does not automatically eliminate stale-cache races.

---

# 270. Code Review Exercise

Review:

```js
const value = await cache.get(key);

if (!value) {
  const fresh = await db.query(...);
  await cache.set(key, JSON.stringify(fresh));
  return fresh;
}

return JSON.parse(value);
```

Design a single-flight mechanism.

---

# 271. Code Review Exercise

Review:

```js
cache.set(
  `search:${JSON.stringify(req.query)}`,
  results
);
```

Identify:

```text
canonicalization
cardinality
key length
authorization
sensitive query data
```

---

# 272. Interview Questions — Senior

1. What is cache-aside?
2. What makes a good cache key?
3. What is TTL?
4. What is cache stampede?
5. How do you prevent stampedes?
6. What happens if cache fails?
7. What is negative caching?
8. What is cache invalidation?
9. Why is in-memory caching different from Redis?
10. How do HTTP ETags work?

---

# 273. Interview Questions — Principal

1. Design a multi-layer cache for millions of requests/sec.
2. How would you prove caching improves the system?
3. How would you prevent cross-tenant leakage?
4. How would you handle a global cache outage?
5. How would you prevent a hot-key stampede?
6. How would you guarantee a freshness SLA?
7. How would you change cache schema without a cold-start storm?
8. How would you cache permission-sensitive data?
9. How would you scale cache invalidation?
10. When would you deliberately remove a cache?

---

# 274. Predict-the-Output / Behavior Exercises

## Exercise A

```js
const cache = new Map();

cache.set("x", 1);

console.log(cache.get("x"));
console.log(cache.has("y"));
```

Predict.

Then explain:

```text
why Map is useful as an L1 cache
```

but:

```text
why it is insufficient as a shared cache
```

---

## Exercise B

```js
const inFlight = new Map();

function load(key) {
  if (inFlight.has(key)) {
    return inFlight.get(key);
  }

  const promise =
    Promise.resolve(Math.random());

  inFlight.set(key, promise);

  return promise;
}
```

What happens when:

```js
Promise.all([
  load("x"),
  load("x")
]);
```

Then identify the missing cleanup.

---

## Exercise C

```js
const ttl = 60;

console.log(ttl);
```

The code is trivial.

The reasoning question is:

```text
What happens when 10,000 hot keys all use exactly the same TTL and are populated together?
```

---

# 275. Mastery Exercises

## Level 1 — Local Cache

Implement:

```text
get
set
delete
TTL
max entries
```

---

## Level 2 — Cache-Aside

Add:

```text
source loader
serialization
failure fallback
```

---

## Level 3 — Stampede Protection

Add:

```text
single-flight
TTL jitter
```

---

## Level 4 — Consistency

Add:

```text
versioned values
invalidation
read-your-writes behavior
```

---

## Level 5 — Distributed

Replace L1 with:

```text
distributed cache
```

and keep the adapter boundary.

---

## Level 6 — HTTP Caching

Add:

```text
Cache-Control
ETag
304
```

---

## Level 7 — Production

Add:

```text
observability
hot-key detection
warmup
failure injection
load testing
security tests
```

---

# 276. Production Acceptance Criteria

```text
[ ] cache policy documented
[ ] source of truth documented
[ ] key strategy documented
[ ] tenant isolation
[ ] authorization safety
[ ] TTL
[ ] invalidation
[ ] negative caching policy
[ ] serialization contract
[ ] cache versioning
[ ] bounded memory
[ ] eviction strategy
[ ] stampede protection
[ ] single-flight
[ ] hot-key strategy
[ ] cache outage fallback
[ ] cache timeouts
[ ] source protection
[ ] HTTP caching policy
[ ] ETag strategy where useful
[ ] cache warming
[ ] deployment migration
[ ] metrics
[ ] logs
[ ] tracing
[ ] tests
[ ] concurrency tests
[ ] failure tests
[ ] load tests
[ ] security tests
```

---

# 277. Operational Checklist

```text
[ ] hit/miss measured
[ ] miss penalty measured
[ ] cache latency monitored
[ ] eviction monitored
[ ] cache memory monitored
[ ] source fallback monitored
[ ] stampede monitored
[ ] hot keys monitored
[ ] key cardinality understood
[ ] TTL distribution understood
[ ] invalidation lag monitored
[ ] permission cache freshness defined
[ ] cache outage runbook
[ ] cache flush runbook
[ ] warmup runbook
[ ] rollback strategy
```

---

# 278. Principal Decision Framework

For every cache, ask:

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

Also ask:

```text
What is the source of truth?
What is the maximum acceptable staleness?
Can the cache be rebuilt?
What happens if cache is empty?
What happens if cache is unavailable?
What happens if cache returns wrong data?
What happens after a write?
What happens during concurrent reads/writes?
What happens during deployment?
What happens during a flush?
```

---

# 279. Dependency Graph

```text
Chapter 24 — JS data structures
              ↓
Chapter 45–48 — Memory / Engine
              ↓
Chapter 55 — HTTP Networking
              ↓
Chapter 57 — Security
              ↓
Chapter 58–63 — Node Runtime
              ↓
Chapter 79–85 — API / DB / Observability / Reliability / Performance
              ↓
Chapter 98–101 — Judgment
              ↓
Chapter 105 — Node REST API
              ↓
Chapter 106 — Real-Time WebSocket
              ↓
Chapter 107 — Job Queue
              ↓
Chapter 108 — Cache System
              ↓
Chapter 109 — Event-Driven App
              ↓
Chapter 110 — Production JS Backend
              ↓
Chapter 111 — Large-Scale JS Platform
```

---

# 280. Concept Connections

## Depends On

```text
HTTP
memory
Node.js
databases
security
observability
reliability
performance
testing
```

## Builds Toward

```text
event-driven systems
production backends
large-scale platforms
distributed architectures
```

## Revisited

```text
Map
streams
async operations
AbortSignal
transactions
events
idempotency
backpressure
observability
```

## Why This Chapter Matters Later

Caching is where performance becomes a correctness problem.

The moment data is copied, you must answer:

```text
When is this copy valid?
Who invalidates it?
What if invalidation races with reads?
What if the cache disappears?
What if permissions change?
```

Those questions are foundational to:

```text
distributed systems
CDNs
event-driven architecture
large-scale APIs
platform engineering
```

---

# 281. Spaced Retrieval Schedule

### Day 0

```text
cache-aside
keys
TTL
invalidation
```

### Day 1

```text
stampede
single-flight
negative caching
hot keys
```

### Day 3

```text
HTTP caching
ETag
Cache-Control
tenant isolation
```

### Day 7

```text
cache failure
consistency
schema migration
warming
```

### Day 14

```text
load testing
cost model
incident response
```

### Day 30

Build a cache from scratch with no reference.

### Day 60

Design a multi-layer cache.

### Day 90

Defend whether a proposed cache should exist at all.

---

# 282. Revision / Retrieval Record

```md
# Chapter 108 — Revision / Retrieval Record

## Review #
- Date:
- Duration:
- Status before:
- Status after:

## Retrieval
- cache purpose [ ]
- cache hierarchy [ ]
- cache-aside [ ]
- read-through [ ]
- write-through [ ]
- write-behind [ ]
- refresh-ahead [ ]
- key design [ ]
- TTL [ ]
- freshness [ ]
- consistency [ ]
- invalidation [ ]
- stampede [ ]
- single-flight [ ]
- hot keys [ ]
- negative caching [ ]
- eviction [ ]
- serialization [ ]
- HTTP caching [ ]
- Cache-Control [ ]
- ETag [ ]
- CDN [ ]
- tenant isolation [ ]
- authorization safety [ ]
- cache failure [ ]
- fallback [ ]
- warming [ ]
- schema migration [ ]
- observability [ ]
- testing [ ]
- performance [ ]
- cost [ ]

## Build Evidence
- Repository:
- Commit:
- Cache backend:
- Hit rate:
- p95 latency:
- Load-test result:
- Failure-injection result:
- Security test:

## Gaps
-

## New Insights
-

## Follow-up
-
```

---

# Chapter 108 — Canonical References and Source Discipline

Primary references:

1. HTTP Caching — IETF HTTP Working Group / RFC 9111  
   https://www.rfc-editor.org/rfc/rfc9111

2. HTTP Semantics — IETF HTTP Working Group / RFC 9110  
   https://www.rfc-editor.org/rfc/rfc9110

3. Node.js Documentation  
   https://nodejs.org/docs/

4. ECMAScript Language Specification  
   https://tc39.es/ecma262/

5. OWASP API Security  
   https://owasp.org/www-project-api-security/

6. OWASP Cheat Sheets  
   https://cheatsheetseries.owasp.org/

7. Redis Documentation  
   https://redis.io/docs/

8. PostgreSQL Documentation  
   https://www.postgresql.org/docs/

Source discipline:

```text
HTTP cache semantics
→ HTTP specifications

Node behavior
→ Node documentation

JavaScript behavior
→ ECMAScript

cache backend semantics
→ chosen backend documentation

database correctness
→ chosen database documentation

security
→ OWASP + threat model

performance
→ benchmark + profile + telemetry

freshness/consistency
→ explicit product/system requirements
```

Do not treat:

```text
TTL
```

as a universal invalidation solution.

Do not treat:

```text
cache hit
```

as proof that the value is correct.

Do not treat:

```text
Redis
```

as a correctness boundary unless the architecture explicitly makes it one.

---

# 283. Completion Snapshot

```text
Part XX — Projects

Chapter 108 — Production JavaScript Cache System
[ ] Not Started

Track A — Core Theory
[ ] cache purpose
[ ] hierarchy
[ ] cache-aside
[ ] read-through
[ ] write-through
[ ] write-behind
[ ] refresh-ahead
[ ] cache keys
[ ] TTL
[ ] freshness
[ ] consistency
[ ] invalidation
[ ] stampede
[ ] single-flight
[ ] distributed locks
[ ] hot keys
[ ] negative caching
[ ] eviction
[ ] serialization
[ ] HTTP caching
[ ] Cache-Control
[ ] ETag
[ ] CDN
[ ] tenant isolation
[ ] authorization
[ ] failure modes
[ ] observability
[ ] cost

Track B — Implementation
[ ] cache adapter
[ ] Map L1
[ ] distributed L2
[ ] key builders
[ ] serialization
[ ] TTL
[ ] cache-aside
[ ] single-flight
[ ] negative caching
[ ] invalidation
[ ] versioning
[ ] soft TTL
[ ] refresh
[ ] fallback
[ ] circuit breaker
[ ] HTTP cache headers
[ ] ETag
[ ] warmup
[ ] metrics
[ ] logs
[ ] traces
[ ] security tests
[ ] concurrency tests
[ ] load tests
[ ] failure injection

Track C — Interview / Reasoning
[ ] Explain why cache exists
[ ] Explain source of truth
[ ] Explain cache-aside
[ ] Explain key design
[ ] Explain TTL
[ ] Explain invalidation
[ ] Explain stampede
[ ] Explain hot keys
[ ] Explain cache failure
[ ] Explain HTTP caching
[ ] Explain ETag
[ ] Explain tenant isolation
[ ] Explain consistency
[ ] Explain cache migration
[ ] Explain cost
[ ] Defend whether to cache

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

# 284. Completion Criteria

Do not mark mastery because:

```text
Redis is connected
```

You are ready to continue when you can independently:

1. Identify when caching is useful.
2. Identify when caching is harmful.
3. Define the source of truth.
4. Define acceptable staleness.
5. Design safe cache keys.
6. Enforce tenant isolation.
7. Protect authorization-sensitive data.
8. Choose cache-aside/write-through/etc. intentionally.
9. Define TTLs.
10. Design invalidation.
11. Analyze stale refill races.
12. Prevent stampedes.
13. Handle hot keys.
14. Handle negative caching.
15. Bound memory.
16. Design eviction behavior.
17. Handle cache outages.
18. Protect the source during cache failures.
19. Design HTTP caching.
20. Use ETag/conditional requests appropriately.
21. Version cached schemas.
22. Warm caches safely.
23. Measure cache benefit.
24. Test concurrency.
25. Test failure modes.
26. Test security isolation.
27. Load test cold and warm paths.
28. Explain cache consistency.
29. Explain operational cost.
30. Defend whether the cache should exist.

---

# Final Mental Model

```text
                        REQUEST
                           │
                           ▼
                     ┌───────────┐
                     │    L1     │
                     │ local RAM │
                     └─────┬─────┘
                           │ miss
                           ▼
                     ┌───────────┐
                     │    L2     │
                     │ distributed
                     │   cache   │
                     └─────┬─────┘
                           │ miss
                           ▼
                     ┌───────────┐
                     │   SOURCE  │
                     │ DB / API  │
                     └─────┬─────┘
                           │
                     populate cache
                           │
                           ▼
                       RESPONSE
```

Write path:

```text
REQUEST
  ↓
validate
  ↓
authorize
  ↓
DB transaction
  ↓
commit
  ↓
invalidate / version / publish event
  ↓
future reads rebuild cache
```

Failure path:

```text
CACHE DOWN
   ↓
timeout quickly
   ↓
bounded fallback
   ↓
protect source
   ↓
observe
   ↓
recover
```

Stampede path:

```text
HOT KEY EXPIRES
       ↓
many misses
       ↓
single-flight / refresh
       ↓
one bounded source load
       ↓
populate
       ↓
many readers
```

The deepest lesson is:

```text
Cache
≠
faster storage
```

A real cache is:

```text
derived state
+
freshness policy
+
invalidation policy
+
capacity mechanism
+
failure boundary
```

The strongest cache architecture is one where:

```text
the source remains authoritative
the cache is reconstructible
freshness is explicit
keys are safe
staleness is bounded
stampedes are controlled
memory is bounded
fallback is safe
authorization remains correct
cache failures do not collapse the database
observability proves the benefit
```

> **Mastery reminder:** Never add a cache merely because a database query is slow. First understand the workload, correctness requirement, source bottleneck, reuse pattern, and failure model. A cache should solve a measured problem without silently creating a harder one.