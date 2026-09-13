# Chapter 132 — Browser Storage Architecture

> **JavaScript Mastery — Part XXIII: Browser Platform & Client State**
>
> **Mission:** Master browser-side storage as a platform architecture: cookies, Web Storage, IndexedDB, Cache Storage, Origin Private File System, Storage Manager, quotas, eviction, persistence, transactions, partitioning, privacy, service-worker interaction, data lifecycle, migrations, offline systems, synchronization, and production storage strategy.
>
> **Role perspective:** Principal JavaScript Engineer · Browser Platform Engineer · Storage Architect · Security Engineer · PWA Engineer · Performance Engineer · Distributed Systems Engineer
>
> **Status:** `[ ] Not Started`
>
> **Core rule:** **Browser storage is not one database. It is a family of storage mechanisms with different ownership, synchronization, lifetime, quota, privacy, and network semantics. Choose the primitive from the data lifecycle and trust model—not from habit.**

---

# 1. Learning Objectives

By completing this chapter, you should be able to:

```text
[ ] explain the browser storage architecture at a platform level
[ ] distinguish HTTP cookies from client-side storage
[ ] explain localStorage
[ ] explain sessionStorage
[ ] explain IndexedDB
[ ] explain Cache Storage
[ ] explain Origin Private File System
[ ] explain StorageManager
[ ] understand storage buckets conceptually
[ ] understand origin-based storage
[ ] understand top-level-site / partitioned storage
[ ] explain first-party vs third-party storage contexts
[ ] explain cookie attributes
[ ] explain Secure
[ ] explain HttpOnly
[ ] explain SameSite
[ ] explain Partitioned cookies / CHIPS
[ ] explain cookie Path
[ ] explain cookie Domain
[ ] explain Max-Age and Expires
[ ] explain cookie request behavior
[ ] understand why cookies are not a generic client database
[ ] explain Web Storage's synchronous API
[ ] explain storage event semantics
[ ] explain Web Storage limits
[ ] explain IndexedDB object stores
[ ] explain keys and key paths
[ ] explain indexes
[ ] explain transactions
[ ] explain transaction scope
[ ] explain transaction modes
[ ] explain version upgrades
[ ] explain blocked upgrades
[ ] explain deleteObjectStore
[ ] explain database migrations
[ ] explain request events in IndexedDB
[ ] understand structured cloning
[ ] explain IndexedDB concurrency
[ ] explain IndexedDB durability considerations
[ ] explain Cache API request/response storage
[ ] distinguish data cache from application database
[ ] explain service-worker cache strategies
[ ] explain cache versioning
[ ] explain stale-while-revalidate architecture
[ ] explain cache invalidation
[ ] explain StorageManager.estimate()
[ ] explain navigator.storage.persist()
[ ] understand best-effort vs persistent storage
[ ] understand quota and eviction
[ ] understand private browsing differences
[ ] understand storage partitioning
[ ] understand Storage Access API
[ ] understand cross-site privacy constraints
[ ] explain cache/localStorage/IndexedDB isolation
[ ] explain storage security boundaries
[ ] understand XSS implications for client-side secrets
[ ] understand token-storage trade-offs
[ ] explain offline-first architecture
[ ] explain sync queues
[ ] explain optimistic updates and local persistence
[ ] design cache/data ownership boundaries
[ ] design storage migrations
[ ] design corruption/recovery handling
[ ] test browser storage deterministically
[ ] build storage abstractions without hiding important semantics
[ ] choose a storage mechanism for a production requirement
```

---

# 2. Prerequisites

You should already understand:

```text
Chapter 04 — Strings / Unicode / Text Semantics
Chapter 24 — Objects / Map / Set / WeakMap / WeakSet
Chapter 28 — JSON / Serialization / Structured Clone
Chapter 31 — Async Fundamentals
Chapter 35 — Promises
Chapter 36 — Async/Await
Chapter 41 — Specification Architecture
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
Chapter 88 — Debugging Methodology
Chapter 101 — Production Scenarios
Chapter 109 — Event-Driven Applications
Chapter 129 — Regex / Text Processing
Chapter 131 — URI / URL / Encoding
```

Supporting knowledge:

```text
HTTP cookies
origins
transactions
indexes
caching
offline systems
consistency
privacy
```

---

# 3. What Is Browser Storage?

Browser storage is the collection of platform mechanisms that allow web applications to retain state beyond a single JavaScript execution turn.

A simplified architecture is:

```text
Browser
├── HTTP state
│   └── Cookies
│
├── Synchronous key/value
│   ├── localStorage
│   └── sessionStorage
│
├── Structured local database
│   └── IndexedDB
│
├── Request/response cache
│   └── Cache Storage
│
├── Origin-private file system
│   └── OPFS
│
└── Storage management
    └── StorageManager / quotas / persistence
```

Modern browsers can also maintain other site data, but these are the major developer-facing storage primitives. citeturn406306search1turn406306search6

---

# 4. Why So Many Storage APIs Exist

Different data has different requirements.

```text
session preference
→ tiny key/value

authentication cookie
→ browser-managed HTTP state

large structured application data
→ IndexedDB

HTTP response cache
→ Cache Storage

large origin-private files
→ OPFS

storage policy/usage
→ StorageManager
```

The APIs differ because:

```text
semantics differ.
```

---

# 5. The Core Storage Decision

Before choosing a primitive ask:

```text
What is the data?

Who needs it?

Does the server need it automatically?

How large can it become?

Is it structured?

Does it need transactions?

Does it need offline access?

Can it be discarded?

Must it survive browser restart?

Does it contain secrets?

Does it need cross-tab coordination?

Does it need third-party embedding?

```

---

# 6. Storage Hierarchy Mental Model

Think:

```text
network protocol state
        ↓
cookie

simple client preference
        ↓
Web Storage

structured persistent data
        ↓
IndexedDB

network response cache
        ↓
Cache Storage

large local files
        ↓
OPFS

storage policy / capacity
        ↓
StorageManager
```

Do not use:

```text
one API for everything.
```

---

# 7. Origin as a Storage Boundary

Browser state is commonly scoped by:

```text
origin
```

where origin is roughly:

```text
scheme + host + port
```

For many storage APIs:

```text
https://example.com
```

and:

```text
https://example.com:8443
```

do not share the same storage area.

The Storage Standard generally organizes origin data through storage buckets, with additional partitioning possible in some contexts. citeturn406306search6

---

# 8. Storage Buckets

At platform level, browser storage is not necessarily one flat directory per origin.

The Storage Standard provides the conceptual model of:

```text
storage buckets
```

which group site data for quota, persistence, and eviction management.

An origin may have multiple buckets under some circumstances, including storage partitioning. citeturn406306search6

---

# 9. First-Party vs Third-Party Context

A resource can execute:

```text
first-party
```

or be embedded:

```text
third-party
```

such as:

```html
<iframe src="https://widget.example"></iframe>
```

Modern browsers increasingly partition or restrict third-party state to reduce cross-site tracking and unwanted state sharing. citeturn406306search4turn406306search5

---

# 10. Storage Partitioning

Traditional mental model:

```text
origin → storage
```

Modern privacy-aware model can become:

```text
top-level site
+
embedded origin
→ separate storage context
```

This is especially relevant for:

```text
third-party iframes
cookies
localStorage-like state
IndexedDB
caches
```

Browser behavior differs, but the architectural trend is:

```text
cross-site shared state
→ restricted/partitioned.
```

---

# 11. Cookies

Cookies are small pieces of state exchanged between:

```text
browser
↔
HTTP server
```

They are special because they participate automatically in HTTP requests according to cookie matching rules.

This makes them fundamentally different from:

```text
localStorage
IndexedDB
Cache Storage
```

---

# 12. Cookie Lifecycle

Typical flow:

```text
server response
→ Set-Cookie
→ browser stores cookie
→ future matching HTTP request
→ Cookie header
→ server reads state
```

JavaScript can also interact with non-HttpOnly cookies through:

```js
document.cookie
```

subject to browser policies. citeturn406306search9

---

# 13. Cookie Attributes

Important attributes include:

```text
Secure
HttpOnly
SameSite
Domain
Path
Max-Age
Expires
Partitioned
```

Each changes:

```text
where
when
and how
```

the browser can send or expose a cookie.

---

# 14. Secure

```text
Secure
```

limits cookie transmission to secure contexts/HTTPS requests under cookie semantics.

Use it for security-sensitive cookies.

Do not interpret `Secure` as:

```text
cookie is encrypted at rest
```

It primarily constrains transmission behavior.

---

# 15. HttpOnly

```text
HttpOnly
```

prevents normal JavaScript access to the cookie through:

```js
document.cookie
```

This is valuable for session identifiers because:

```text
XSS
→ cannot directly read HttpOnly cookie
```

but XSS can still often:

```text
act as the user
```

through the application's own authenticated requests.

HttpOnly is not:

```text
XSS prevention.
```

---

# 16. SameSite

`SameSite` controls cookie behavior in cross-site contexts.

Common settings:

```text
Strict
Lax
None
```

Cross-site cookies with:

```text
SameSite=None
```

generally also require:

```text
Secure
```

under modern cookie semantics.

Use SameSite as part of:

```text
CSRF
privacy
cross-site authentication
```

architecture.

---

# 17. Cookie Domain

Domain controls where a cookie applies.

A host-only cookie is generally safer when:

```text
subdomain sharing
```

is not required.

Avoid unnecessarily broad:

```text
Domain=.example.com
```

style sharing because it increases the set of hosts that can participate in the cookie's scope.

---

# 18. Cookie Path

`Path` restricts request paths to which a cookie is sent.

Do not confuse:

```text
cookie path
```

with:

```text
URL origin
```

or:

```text
application route authorization.
```

Path scoping is useful for reducing exposure but is not an application security boundary by itself.

---

# 19. Cookie Expiration

Common forms:

```text
Expires
Max-Age
```

`Max-Age` expresses lifetime in seconds.

Session cookies omit explicit lifetime attributes and generally expire with the browser's session semantics.

Browser behavior can also be affected by privacy and user settings. citeturn406306search9

---

# 20. Partitioned Cookies / CHIPS

Partitioned cookies allow a cookie to be stored separately per:

```text
top-level site
+
cookie origin/host
```

This supports legitimate embedded state while reducing cross-site tracking.

Current browsers supporting CHIPS use:

```text
Partitioned
```

as an opt-in cookie attribute; partitioned cookies also require:

```text
Secure
```

under the CHIPS model. citeturn406306search0

---

# 21. Why CHIPS Matters

Traditional third-party state:

```text
third-party.example
```

could historically observe a shared cookie across:

```text
site-a.example
site-b.example
site-c.example
```

Partitioned storage changes this toward:

```text
site-a + third-party
site-b + third-party
site-c + third-party
```

separate state.

This is a privacy architecture change, not merely a cookie syntax feature. citeturn406306search0

---

# 22. Web Storage

Web Storage provides:

```text
localStorage
sessionStorage
```

with a simple key/value API.

Values are:

```text
strings
```

and the API is:

```text
synchronous.
```

These mechanisms are widely available. citeturn406306search8

---

# 23. `localStorage`

Conceptually:

```text
origin-scoped persistent key/value storage
```

Example:

```js
localStorage.setItem("theme", "dark");

const theme = localStorage.getItem("theme");
```

It usually persists across:

```text
page reload
browser restart
```

until removed or evicted according to browser/storage policies. citeturn406306search8

---

# 24. `sessionStorage`

`sessionStorage` is associated with:

```text
origin
+
browser tab/page session context.
```

It is intended for state that should not normally outlive the relevant tab session.

MDN describes it as partitioned by tab and origin, while localStorage is partitioned by origin only. citeturn406306search8

---

# 25. Web Storage Is Synchronous

Example:

```js
localStorage.setItem("large", hugeString);
```

The call is synchronous.

Therefore:

```text
large values
+
many operations
```

can block the main thread.

This makes Web Storage unsuitable for:

```text
large datasets
high-frequency writes
large offline databases
```

---

# 26. Web Storage Quotas

Browsers impose size limits.

MDN currently documents a typical maximum of:

```text
5 MiB localStorage
+
5 MiB sessionStorage
```

per origin, with `QuotaExceededError` when limits are reached; exact browser policies can vary. citeturn406306search1

Treat quota as:

```text
runtime resource
```

not:

```text
guaranteed free capacity.
```

---

# 27. Handling Quota Errors

Example:

```js
try {
  localStorage.setItem("cache", data);
} catch (error) {
  if (error instanceof DOMException &&
      error.name === "QuotaExceededError") {
    // degrade or evict
  } else {
    throw error;
  }
}
```

Do not let storage exhaustion crash the entire UI.

---

# 28. Web Storage Serialization

Because values are strings:

```js
localStorage.setItem("user", JSON.stringify(user));
```

and:

```js
const user = JSON.parse(localStorage.getItem("user"));
```

are common.

But this has costs:

```text
serialization
parsing
full-value replacement
no native indexing
```

---

# 29. Storage Event

Changes to Web Storage can trigger:

```text
storage
```

events in relevant other documents.

Example:

```js
window.addEventListener("storage", event => {
  console.log(event.key, event.newValue);
});
```

Use this for:

```text
simple cross-tab signals
```

not as a general transactional synchronization system.

---

# 30. Storage Event Limits

The storage event is:

```text
notification
```

not:

```text
distributed transaction.
```

It does not give you:

```text
database transactions
conflict resolution
causal ordering
durable event log
```

For stronger cross-context coordination, consider:

```text
BroadcastChannel
IndexedDB
Web Locks
```

as appropriate.

---

# 31. Why Not Store Tokens in `localStorage` Automatically?

Any script running with access to the page's JavaScript context can generally access:

```js
localStorage
```

Therefore:

```text
XSS
→ token theft
```

is a major concern for bearer credentials stored there.

The right decision depends on:

```text
threat model
application architecture
cookie usage
CSRF defenses
token lifetime
```

Do not follow a blanket:

```text
always localStorage
```

or:

```text
never localStorage
```

rule.

---

# 32. IndexedDB

IndexedDB is a browser database API for:

```text
large structured data
indexes
transactions
asynchronous access
```

It is far more suitable than Web Storage for:

```text
offline applications
large datasets
structured records
client-side state
```

MDN describes IndexedDB as a mechanism for storing large data structures and indexing them for high-performance searching. citeturn406306search1

---

# 33. IndexedDB Mental Model

Think:

```text
Database
├── object store
│   ├── records
│   ├── key
│   └── indexes
│
├── object store
│   └── ...
│
└── metadata/version
```

---

# 34. Database Version

IndexedDB databases have a:

```text
version number
```

Changing the schema typically involves:

```text
open with higher version
→ upgrade transaction
→ schema changes
→ normal operation
```

---

# 35. Upgrade Transaction

The schema upgrade occurs in:

```text
onupgradeneeded
```

Example:

```js
const request = indexedDB.open("app", 2);

request.onupgradeneeded = event => {
  const db = event.target.result;

  if (!db.objectStoreNames.contains("users")) {
    db.createObjectStore("users", {
      keyPath: "id"
    });
  }
};
```

This is the migration boundary.

---

# 36. Database Migration Discipline

Treat IndexedDB migrations like:

```text
database migrations
```

not:

```text
one-off startup code.
```

Use:

```text
version 1
→ version 2
→ version 3
→ ...
```

Each migration should be:

```text
deterministic
idempotent within its upgrade contract
tested
backward-aware
```

---

# 37. Blocked Upgrades

If another tab keeps an older database connection open, a schema upgrade can be blocked.

Architecturally:

```text
tab A → old connection
tab B → wants upgrade
        ↓
blocked
```

Production UX may need to tell the user:

```text
another tab must reload/close.
```

---

# 38. Object Stores

An object store is conceptually:

```text
collection of records
```

with keys.

Example:

```js
db.createObjectStore("orders", {
  keyPath: "id"
});
```

---

# 39. Key Path

A key path identifies where the primary key comes from.

Example:

```js
{
  id: "order-123",
  total: 500
}
```

with:

```js
keyPath: "id"
```

The key is:

```text
order-123
```

---

# 40. Indexes

An index provides alternate lookup paths.

Example:

```js
store.createIndex("byEmail", "email", {
  unique: true
});
```

Then:

```text
primary key
vs
alternate indexed field
```

can support different access patterns.

---

# 41. Index Design

Before adding an index ask:

```text
What query does this support?

Is the field selective?

Does it change often?

Do we need uniqueness?

What data shape does it index?

Is the query worth storage/index maintenance?
```

Indexes are:

```text
performance structures
```

with:

```text
storage and write costs.
```

---

# 42. Transactions

IndexedDB operations occur through transactions.

Conceptually:

```text
transaction
→ object store scope
→ operations
→ commit/abort
```

Transactions are central to correctness.

---

# 43. Transaction Modes

Important modes include:

```text
readonly
readwrite
versionchange
```

Use the least permissive mode needed.

---

# 44. Transaction Scope

A transaction declares which object stores it can access.

This provides a boundary for:

```text
concurrency
locking/coordination
atomic work
```

under IndexedDB semantics.

Do not casually open:

```text
readwrite across every store
```

when only one store is required.

---

# 45. Atomic Application Updates

Suppose a local order write needs:

```text
order record
+
outbox message
```

Put them in the same transaction when the architecture requires:

```text
both exist
or
neither exists.
```

This enables local atomicity.

---

# 46. Local Outbox Pattern

A robust offline system can use:

```text
IndexedDB
├── entities
├── pending mutations
└── metadata
```

Then:

```text
UI update
→ transaction
   ├── write entity
   └── write pending mutation
→ commit
```

Later:

```text
sync worker
→ read pending mutation
→ send server request
→ mark/remove mutation
```

This reduces:

```text
lost writes
```

during offline operation.

---

# 47. IndexedDB and Structured Clone

IndexedDB can store values through the platform's structured-cloning mechanisms, allowing data structures that are richer than:

```text
JSON strings.
```

This makes it useful for:

```text
arrays
objects
typed arrays
Blobs
other cloneable values
```

subject to supported types and platform semantics.

---

# 48. JSON vs Structured Clone

JSON:

```text
text serialization
```

Structured clone:

```text
platform object graph serialization/cloning model
```

The choice affects:

```text
supported types
identity/cycles
performance
serialization semantics.
```

Do not stringify every IndexedDB value by habit.

---

# 49. IndexedDB Requests Are Event-Based

Traditional IndexedDB APIs expose:

```text
IDBRequest
```

with events such as:

```text
onsuccess
onerror
```

Many applications wrap these in Promises:

```js
function requestToPromise(request) {
  return new Promise((resolve, reject) => {
    request.onsuccess = () => resolve(request.result);
    request.onerror = () => reject(request.error);
  });
}
```

A wrapper should not hide:

```text
transaction lifetime
abort behavior
request ordering
```

---

# 50. IndexedDB Transaction Lifetime

A transaction's lifecycle is not equivalent to:

```text
"all my async work until I am done"
```

There are rules about when transactions can continue and when they become inactive/auto-commit.

This is why long async gaps inside a transaction can produce:

```text
TransactionInactiveError
```

or related failures.

---

# 51. Production IndexedDB Wrapper

A good wrapper should expose:

```text
transaction boundaries
schema versions
abort semantics
indexes
retry/recovery
migration errors
```

Do not build:

```text
DAO abstraction
```

that hides all transactional meaning.

---

# 52. IndexedDB Concurrency

Multiple tabs/workers can access the same origin's database.

Therefore think:

```text
multi-context concurrent database
```

not:

```text
one JavaScript object.
```

Schema upgrades require coordination across existing connections.

---

# 53. Cache Storage

The Cache API stores:

```text
Request ↔ Response
```

pairs.

It is especially useful for:

```text
service workers
offline pages
static assets
API response caching
```

MDN describes Cache API data as persistent origin storage managed with other origin storage systems. citeturn406306search1turn406306search6

---

# 54. Cache Storage Is Not a General Database

Do not use Cache Storage as:

```text
primary domain database
```

It models:

```text
HTTP-like request/response caching.
```

Use IndexedDB for:

```text
domain records
relationships
queries
```

---

# 55. Cache Matching

Example:

```js
const cache = await caches.open("v1");
await cache.put(request, response);

const cached = await cache.match(request);
```

This models:

```text
request key
→ response value
```

rather than:

```text
SQL-like query.
```

---

# 56. Service Worker Cache Architecture

A typical offline shell architecture:

```text
navigation request
       ↓
service worker
       ↓
cache
   ↙       ↘
hit       miss
 ↓         ↓
response   network
             ↓
           cache
```

This makes the service worker a:

```text
programmable network/storage boundary.
```

---

# 57. Cache Versioning

Never assume:

```text
cache name = permanent.
```

Use versions:

```text
static-v1
static-v2
api-v3
```

During service-worker activation:

```text
delete old caches
→ keep current cache
```

This avoids stale application shells.

---

# 58. Cache Invalidation

A cached response can be:

```text
correct
stale
poisoned
obsolete
```

Therefore decide:

```text
cache-first
network-first
stale-while-revalidate
network-only
cache-only
```

according to resource semantics.

---

# 59. Stale-While-Revalidate

Conceptual flow:

```text
request
 ↓
cache hit?
 ├── yes → return cached response
 │          +
 │          refresh in background
 │
 └── no → network
           ↓
         cache
           ↓
         return
```

This provides:

```text
low latency
+
eventual freshness
```

at the cost of:

```text
staleness window
```

---

# 60. Cache Ownership

A useful rule:

```text
IndexedDB:
application truth/cacheable domain state

Cache Storage:
HTTP representation cache

Web Storage:
tiny synchronous preferences

Cookies:
server-visible browser state
```

This is an ownership contract, not a strict law.

---

# 61. Origin Private File System

OPFS provides a file-system-like storage space private to the origin.

It is useful for:

```text
large local files
editing applications
media data
WASM workloads
file-backed application state
```

MDN identifies OPFS as a storage mechanism private to the page's origin. citeturn406306search1

---

# 62. OPFS vs IndexedDB

### IndexedDB

```text
records
indexes
transactions
structured data
```

### OPFS

```text
files
byte-oriented data
file-system style organization
```

Choose based on:

```text
data model
access pattern
performance
size
```

---

# 63. Storage Manager

The Storage API provides:

```js
navigator.storage
```

and:

```js
navigator.storage.estimate()
```

to estimate:

```text
usage
quota
```

MDN documents StorageManager as the interface for storage estimates and persistence controls. citeturn406306search11

---

# 64. `estimate()`

Example:

```js
const { usage, quota } = await navigator.storage.estimate();

console.log({
  usage,
  quota
});
```

This is useful for:

```text
capacity planning
user feedback
cache management
offline UX
diagnostics
```

It is an:

```text
estimate
```

not a reservation guarantee.

---

# 65. Persistence

By default, much browser storage is:

```text
best-effort
```

meaning it may be evicted under storage pressure according to browser policy.

Applications can request:

```js
await navigator.storage.persist();
```

for persistent storage where supported/approved. citeturn406306search1

---

# 66. Best-Effort vs Persistent

### Best-effort

```text
normal default
may be evicted
```

### Persistent

```text
stronger retention semantics
requires browser permission/policy
```

Do not tell users:

```text
"browser storage is permanent."
```

It is not.

---

# 67. Storage Quota

Quota depends on:

```text
browser
device
storage technology
origin/site
browser policies
available disk space
private browsing
engagement
```

There is no universal:

```text
one number for every browser
```

for IndexedDB/Cache/OPFS. citeturn406306search1

---

# 68. Eviction

Best-effort data can be evicted when:

```text
storage pressure
quota conditions
browser policy
user action
```

require cleanup.

Therefore offline applications need:

```text
recovery
re-download
resynchronization
```

paths.

---

# 69. Private Browsing

Private/incognito modes can have:

```text
different quotas
lifetime behavior
temporary storage
```

and data is typically removed when the private session ends.

Never use private mode to infer normal persistent-storage behavior. citeturn406306search1

---

# 70. Storage Partitioning and Privacy

Browsers are increasingly separating client-side state by:

```text
top-level site
```

to limit cross-site tracking.

This can affect:

```text
cookies
localStorage
IndexedDB
Cache API
```

for embedded third parties. citeturn406306search4turn406306search7

---

# 71. Third-Party Storage Access

A third-party iframe may not receive the same unpartitioned state it would have as a first-party page.

The Storage Access API exists for some cases where embedded content needs access to unpartitioned cookies/state and browser policy allows it. citeturn406306search5

This is a:

```text
privacy
permission
compatibility
```

boundary.

---

# 72. Partitioned Storage Is Not Shared Storage

With partitioning:

```text
same third-party origin
```

can have:

```text
separate state
```

under different top-level sites.

Do not design third-party widgets assuming:

```text
one global localStorage/IndexedDB namespace.
```

---



# Security Considerations

Browser storage must be treated as application state, not as a universal secure vault. Security review should consider XSS, CSRF, third-party partitioning, cache poisoning, service-worker compromise, sensitive data leakage, and whether secrets belong client-side at all.

# 73. Storage Security Boundary

Browser storage is not a secure vault.

If an attacker gains script execution inside the application's origin through:

```text
XSS
compromised dependency
unsafe extension-like environment
```

they may access client-side state depending on:

```text
storage API
cookie attributes
browser policy
```

Treat stored data as:

```text
application state
```

not automatically:

```text
secret storage.
```

---

# 74. Secrets in IndexedDB

IndexedDB can be technically convenient for:

```text
tokens
credentials
keys
```

but convenience does not make it a secure secret store.

If malicious script runs with the same origin privileges, it may be able to read stored secrets.

Design for:

```text
minimum secret lifetime
httpOnly cookies where appropriate
server-side state
token rotation
XSS prevention
```

---

# 75. Cookies vs localStorage for Session State

### HttpOnly Cookie

Pros:

```text
not directly readable by page JS
automatically sent to matching requests
integrates with server sessions
```

Cons:

```text
CSRF considerations
automatic network transmission
cookie size constraints
```

### localStorage Token

Pros:

```text
explicit JS-controlled transport
simple SPA integration
```

Cons:

```text
XSS can read it
manual transport
```

There is no universal answer independent of:

```text
authentication model.
```

---

# 76. Storage and CSRF

Cookies can be automatically attached to requests.

Therefore cookie-based authentication must consider:

```text
SameSite
CSRF tokens
origin checks
request methods
server-side authorization
```

Do not assume:

```text
HttpOnly = CSRF protection.
```

They solve different problems.

---

# 77. Storage and XSS

XSS can potentially expose:

```text
localStorage
sessionStorage
IndexedDB
non-HttpOnly cookies
```

Therefore:

```text
client storage choice
```

cannot compensate for:

```text
broken output encoding
unsafe DOM usage
dependency compromise.
```

---

# 78. Storage and Subdomains

Cookies can sometimes be deliberately shared across:

```text
subdomains
```

while:

```text
localStorage
IndexedDB
Cache Storage
```

are fundamentally origin-scoped in their normal model.

Thus:

```text
example.com
```

and:

```text
app.example.com
```

should not be assumed to share all browser storage.

---

# 79. Cross-Tab Coordination

Choose based on the requirement:

```text
storage event
→ simple state change notification

BroadcastChannel
→ application-level message passing

Web Locks
→ exclusive coordination

IndexedDB
→ durable shared state

Service Worker
→ centralized network/background coordination
```

No single primitive replaces the others.

---

# 80. Web Locks

For cross-tab coordination, Web Locks can express:

```text
only one context may perform this critical operation
```

Useful for:

```text
single sync worker
migration lock
cache cleanup
leader election-like coordination
```

Use carefully:

```text
lock acquisition
timeout
failure
browser lifecycle
```

must be part of the design.

---

# 81. Offline-First Architecture

A robust offline app often uses:

```text
UI
 ↓
local domain store
 ↓
outbox
 ↓
sync engine
 ↓
network
 ↓
server
```

And separately:

```text
service worker
 ↓
static/resource cache
```

Do not force:

```text
all offline data
```

into Cache Storage.

---

# 82. Read Path

A common offline read:

```text
request
 ↓
IndexedDB
 ↓
return local state immediately
 ↓
optional network refresh
 ↓
write fresh state
 ↓
update UI
```

This is effectively:

```text
stale-while-revalidate
```

at the domain-data level.

---

# 83. Write Path

A resilient write:

```text
user action
 ↓
transaction
 ├── update local entity
 └── enqueue mutation
 ↓
UI reflects optimistic state
 ↓
sync
 ↓
server
 ↓
ack
 ↓
clear outbox item
```

This provides a strong foundation for:

```text
offline mutation
```

handling.

---

# 84. Conflict Resolution

Offline systems need a policy for:

```text
local change
vs
server change
```

Possible strategies:

```text
last-write-wins
server wins
client wins
field merge
domain-specific merge
manual conflict
CRDT-like design
```

Storage technology cannot decide this for you.

---

# 85. Idempotency

When replaying offline mutations:

```text
same request
```

may be sent multiple times.

Use:

```text
idempotency keys
operation IDs
deduplication
server-side transaction semantics
```

to avoid duplicate business effects.

---

# 86. Persistent Queue Example

IndexedDB record:

```js
{
  id: "mutation-123",
  type: "UPDATE_ORDER",
  payload: {...},
  attempts: 2,
  createdAt: 1720000000000
}
```

Treat:

```text
outbox
```

as a durable queue.

This connects directly to:

```text
Chapter 107 — Job Queue
```

and:

```text
Chapter 109 — Event-Driven Application
```

---

# 87. Cache Poisoning Risks

If a service worker stores an attacker-influenced response:

```text
request
→ malicious response
→ Cache Storage
→ future users receive cached content
```

depending on architecture.

Validate:

```text
origin
response type
authentication context
cacheability
```

before caching sensitive/dynamic resources.

---

# 88. Cache Key Design

Cache keys should encode the request dimensions that affect the response:

```text
URL
method
headers
credentials mode
vary-sensitive semantics
```

Do not assume:

```text
URL alone
```

is always sufficient.

---

# 89. Authentication-Aware Caching

Never blindly cache private authenticated responses as shared/public resources.

Ask:

```text
Is response user-specific?

Does authorization affect body?

Should this response be reusable?

Could another context receive it?
```

Caching is a security boundary.

---

# 90. Cache Invalidation Strategy

Use explicit policies:

```text
versioned asset
→ immutable cache

API response
→ TTL/revalidation

user-specific data
→ limited/private cache

critical authorization data
→ network/controlled local store
```

---

# 91. Storage Migration Strategy

A production migration should support:

```text
old schema
→ detect version
→ migrate
→ verify
→ mark complete
```

For difficult migrations, consider:

```text
backup/export
dual read
dual write
shadow migration
rollback strategy
```

as appropriate.

---

# 92. Corruption Handling

Local browser data can become unusable because of:

```text
partial migrations
application bugs
browser bugs
unexpected termination
quota pressure
schema incompatibility
```

Design:

```text
health checks
rebuild path
cache invalidation
server rehydration
```

for critical applications.

---

# 93. Source of Truth

A key production decision:

```text
What is authoritative?
```

Possible answers:

```text
server
IndexedDB local snapshot
cache
user preferences
```

Do not let:

```text
local cache
```

accidentally become:

```text
business truth.
```

---

# 94. Cache vs Source of Truth

Useful distinction:

```text
cache:
"can reconstruct this"

source of truth:
"cannot safely lose this without semantic recovery"
```

If data can be re-downloaded:

```text
cache
```

If it contains unsynced user work:

```text
durable local state
```

with:

```text
recovery/sync strategy
```

is required.

---

# 95. Data Classification

Before storing data, classify:

```text
ephemeral
session
preference
cache
offline source-of-work
sensitive
derived
authoritative
```

Then select the primitive.

---

# 96. Storage Budgeting

For an offline application estimate:

```text
static assets
API cache
domain data
attachments
indexes
metadata
outbox
logs
```

Then monitor:

```text
usage
quota
growth rate
eviction risk
```

using:

```js
navigator.storage.estimate()
```

where supported. citeturn406306search11

---

# 97. Large-Data Strategy

For:

```text
videos
documents
large binary data
```

avoid:

```text
huge localStorage strings
```

Consider:

```text
OPFS
IndexedDB
Cache Storage
streaming APIs
```

depending on access semantics.

---

# 98. Storage and Performance

### localStorage

```text
synchronous
serialization cost
main-thread blocking
```

### IndexedDB

```text
async
transactional
structured
```

### Cache Storage

```text
async
request/response oriented
```

### OPFS

```text
file-oriented
large-data workloads
```

Performance depends on:

```text
access pattern
record size
frequency
serialization
transactions
browser implementation.
```

---

# 99. Batching Writes

Instead of:

```text
1000 tiny independent writes
```

consider:

```text
one transaction
```

when atomicity and performance permit.

But avoid:

```text
giant transaction
```

that:

```text
blocks unrelated work
creates contention
increases failure scope.
```

---

# 100. IndexedDB Index Cost

Adding an index can increase:

```text
storage
write cost
migration complexity
```

Only add indexes for:

```text
actual query needs.
```

---

# 101. Storage Telemetry

Track at least:

```text
storage estimate
migration failures
quota errors
database open failures
sync backlog
cache hit rate
recovery events
```

Do not log:

```text
raw sensitive stored values.
```

---

# 102. Debugging Storage

Browser DevTools usually provide panels for:

```text
cookies
localStorage
sessionStorage
IndexedDB
Cache Storage
service workers
```

Use them to inspect:

```text
keys
values
versions
cache entries
registrations
```

while respecting sensitive-data handling.

---

# 103. Debugging IndexedDB

When a migration fails, inspect:

```text
database version
object store list
indexes
open connections
upgrade event
blocked event
abort/error event
```

A failing upgrade can leave the application in:

```text
incompatible schema state
```

from the application's perspective.

---

# 104. Debugging Service Worker Cache

When stale content appears:

```text
check active service worker
→ check cache names
→ check request keys
→ check activation cleanup
→ check fetch handler
→ check HTTP cache
```

Remember:

```text
Cache Storage
≠
browser HTTP cache
```

They are distinct layers.

---

# 105. Browser HTTP Cache vs Cache Storage

### HTTP cache

```text
browser-controlled networking cache
```

### Cache Storage

```text
script-accessible Request/Response cache
```

A response may interact with both layers.

Do not assume clearing one necessarily clears the other.

---

# 106. Testing Strategy

Test:

```text
fresh install
existing schema
upgrade
downgrade attempt
blocked upgrade
quota exhaustion
private browsing
multiple tabs
offline
online
cache stale
cache miss
service worker update
storage eviction simulation
```

---

# 107. Fake Storage

For unit tests:

```text
small in-memory adapter
```

can be useful.

But integration tests must exercise:

```text
real IndexedDB
real Cache Storage
real cookie behavior
```

because wrappers can hide browser-specific semantics.

---

# 108. Deterministic Storage Tests

Avoid tests depending on:

```text
existing developer profile state.
```

Reset:

```text
database
localStorage
sessionStorage
caches
service worker
cookies
```

between test suites when necessary.

---

# 109. Multi-Tab Tests

Explicitly test:

```text
tab A opens database
tab B attempts migration
tab A closes/reloads
tab B resumes
```

and:

```text
tab A writes
tab B observes
```

This reveals:

```text
connection lifetime
storage events
race conditions
```

that unit tests miss.

---

# 110. Security Test Matrix

Include:

```text
XSS
cookie theft attempts
CSRF
partitioned third-party context
cross-origin iframe
open redirects
cache poisoning
sensitive query caching
service-worker takeover
dependency compromise
```

---

# 111. Service Worker Security

A service worker can act as:

```text
programmable request interception layer
```

Therefore compromise of the worker can have broad consequences.

Control:

```text
registration scope
script integrity/deployment
update process
cache policy
response validation
```

---

# 112. Service Worker Update Hazards

A buggy service worker can effectively:

```text
persist bad application behavior
```

through cached assets.

Use:

```text
versioned caches
safe activation
rollback plan
cache cleanup
```

and ensure:

```text
old clients
```

can transition safely.

---

# 113. Storage API Abstraction

A good abstraction might expose:

```js
interface UserPreferencesStore {
  getTheme();
  setTheme(theme);
}
```

rather than:

```js
storage.set("theme", value);
```

everywhere.

But preserve important semantics:

```text
sync vs async
transaction
failure
quota
consistency
```

Do not make IndexedDB look falsely like a synchronous map.

---

# 114. Repository Pattern Warning

A generic:

```text
get(key)
set(key, value)
```

abstraction can hide:

```text
indexes
transactions
query semantics
migration
consistency
```

Use abstractions that express:

```text
domain operations
```

without deleting:

```text
storage semantics.
```

---

# 115. Storage Adapter Testing

For each adapter verify:

```text
read after write
missing key
duplicate key
transaction rollback
upgrade
concurrent access
quota/error propagation
serialization edge cases
```

---

# 116. Offline Storage Architecture Example

```text
              ┌───────────────┐
              │ UI / React    │
              └───────┬───────┘
                      │
              ┌───────▼───────┐
              │ Domain Store  │
              │  IndexedDB    │
              └───┬───────┬───┘
                  │       │
            reads │       │ writes
                  │       ▼
                  │   ┌──────────┐
                  │   │  Outbox  │
                  │   └────┬─────┘
                  │        │
                  │        ▼
              ┌───▼─────────────┐
              │ Sync Engine     │
              └───┬─────────────┘
                  │
                  ▼
               Network
```

Separate:

```text
domain persistence
```

from:

```text
network response cache.
```

---

# 117. Storage Strategy by Data Type

| Data | Preferred Primitive |
|---|---|
| session identifier | HttpOnly cookie when appropriate |
| tiny UI preference | localStorage |
| tab/session state | sessionStorage |
| structured offline records | IndexedDB |
| HTTP asset cache | Cache Storage |
| large private files | OPFS |
| storage capacity/persistence | StorageManager |

These are architectural starting points, not universal mandates.

---

# 118. Common Misconceptions

### Misconception 1

> “Browser storage is one thing.”

Correction:

```text
Browsers provide multiple storage systems with different semantics.
```

### Misconception 2

> “localStorage is a small database.”

Correction:

```text
It is synchronous string key/value storage.
```

### Misconception 3

> “IndexedDB is just localStorage but bigger.”

Correction:

```text
It is a transactional structured database API.
```

### Misconception 4

> “Cache Storage is a database.”

Correction:

```text
It is primarily a request/response cache.
```

### Misconception 5

> “Storage is permanent.”

Correction:

```text
Many origin data stores are best-effort and can be evicted.
```

### Misconception 6

> “HttpOnly prevents XSS.”

Correction:

```text
It prevents direct JS cookie access, but does not remove the XSS capability itself.
```

### Misconception 7

> “Third-party storage is globally shared.”

Correction:

```text
Modern browsers increasingly partition/restrict third-party state.
```

### Misconception 8

> “Quota is guaranteed.”

Correction:

```text
Storage estimates and browser policies are dynamic resource constraints.
```

### Misconception 9

> “If data is in IndexedDB it is encrypted.”

Correction:

```text
IndexedDB is not a cryptographic secret store by default.
```

### Misconception 10

> “Service-worker cache is the same as browser HTTP cache.”

Correction:

```text
They are separate caching layers.
```

---

# 119. Common Mistakes

```text
[ ] storing large JSON blobs in localStorage
[ ] writing localStorage on every keystroke
[ ] storing bearer tokens without a threat-model review
[ ] treating cache as source of truth
[ ] ignoring quota failures
[ ] assuming persistence is guaranteed
[ ] ignoring multi-tab coordination
[ ] shipping untested IndexedDB migrations
[ ] forgetting blocked upgrade handling
[ ] opening giant IndexedDB transactions
[ ] over-indexing database records
[ ] assuming Cache Storage is a queryable database
[ ] using third-party storage assuming global persistence
[ ] caching personalized responses as public resources
[ ] ignoring service-worker update rollback
[ ] logging stored secrets
```

---

# 120. Comparison With Related Concepts

| Primitive | Shape | Async | Server Sends Automatically? | Main Strength |
|---|---|---:|---:|---|
| Cookies | key/value | browser-managed | Yes | HTTP session/state |
| localStorage | string KV | No | No | tiny preferences |
| sessionStorage | string KV | No | No | tab/session state |
| IndexedDB | structured DB | Yes | No | offline structured data |
| Cache Storage | Request/Response | Yes | No | network/resource cache |
| OPFS | file system | Yes / worker-friendly APIs | No | large local files |
| StorageManager | policy/usage API | Yes | No | quota/persistence info |

---

# 121. Production Decision Framework

For every storage requirement ask:

```text
1. What is the data classification?
2. Who needs it?
3. Does the server need automatic transmission?
4. What is the maximum size?
5. What is the expected growth rate?
6. Is the data structured?
7. Do we need indexes?
8. Do we need transactions?
9. Is the data authoritative or derived?
10. Can it be evicted and reconstructed?
11. Does offline editing matter?
12. Are writes frequent?
13. Is synchronous access acceptable?
14. Could multiple tabs access it?
15. Is it third-party embedded?
16. Does partitioning affect it?
17. Is it sensitive?
18. What is the recovery strategy?
19. What is the migration strategy?
20. What is the observability strategy?
```

---

# 122. Implementation From Scratch — Storage Layer

Build a miniature storage system with these layers:

```text
MemoryKV
IndexedStore-like abstraction
Outbox
Cache
StorageManager facade
```

Do not implement browser internals.

The goal is:

```text
understand storage ownership
+
failure
+
consistency
+
migration.
```

---

# 123. Implementation Milestone 1 — KV Store

Build:

```js
class MemoryKV {
  #map = new Map();

  get(key) {
    return this.#map.get(key);
  }

  set(key, value) {
    this.#map.set(key, value);
  }

  delete(key) {
    return this.#map.delete(key);
  }
}
```

Then add:

```text
quota
serialization
versioning
events
```

---

# 124. Implementation Milestone 2 — Indexed Store Model

Create:

```text
records
primary keys
secondary indexes
transaction object
```

Support:

```text
begin
put
delete
get
query
commit
abort
```

Then test:

```text
atomicity
```

---

# 125. Implementation Milestone 3 — Outbox

Build:

```text
enqueue
peek
ack
retry
dead-letter
```

Include:

```text
attempt count
idempotency key
createdAt
nextAttemptAt
```

This connects storage with:

```text
job queues
```

---

# 126. Implementation Milestone 4 — Cache

Build:

```text
cache.put(key, response)
cache.match(key)
cache.delete(key)
cache.keys()
```

Then add:

```text
TTL
version
stale-while-revalidate
```

---

# 127. Implementation Milestone 5 — Storage Policy

Create a policy layer:

```text
max bytes
eviction strategy
sensitive-data policy
persistence expectation
```

Then simulate:

```text
quota exceeded
eviction
corruption
migration failure
```

---

# 128. Debugging Exercises

## Exercise A — Main Thread Freeze

```js
for (let i = 0; i < 50000; i++) {
  localStorage.setItem(`k${i}`, hugeString);
}
```

Measure UI responsiveness.

Explain why Web Storage's synchronous API makes this dangerous. citeturn406306search8

---

## Exercise B — Blocked Migration

Open a database in:

```text
Tab A
```

without closing it.

Then attempt a version upgrade from:

```text
Tab B.
```

Trace:

```text
blocked
→ old connection closes
→ upgrade continues.
```

---

## Exercise C — Quota Failure

Fill storage until:

```text
QuotaExceededError
```

appears.

Verify that the application:

```text
does not crash
```

and:

```text
degrades gracefully.
```

---

## Exercise D — Cache Poisoning

Create a service worker that caches a response influenced by:

```text
untrusted request input.
```

Show how a malicious cached response can persist across requests.

Then redesign the cache policy.

---

# 129. Code Review Exercise

Review:

```js
function saveSession(session) {
  localStorage.setItem(
    "session",
    JSON.stringify(session)
  );
}
```

Ask:

```text
Is this actually session data?

Does the server need automatic auth state?

What secrets are inside?

Can XSS read it?

How large can it become?

How often is it written?

What happens if quota is exceeded?

How is expiration handled?

How is logout enforced across tabs?
```

Then propose a storage design rather than only changing the API call.

---

# 130. Interview Questions

### Fundamentals

```text
1. What is the difference between cookies, localStorage, IndexedDB, and Cache Storage?
2. Why is localStorage synchronous?
3. Why is IndexedDB asynchronous?
4. What is browser storage quota?
5. Can browser storage be evicted?
```

### Cookies

```text
6. What does HttpOnly do?
7. What does Secure do?
8. What does SameSite do?
9. What is a partitioned cookie?
10. Why should cookies not be used as a general database?
```

### IndexedDB

```text
11. What is an object store?
12. What is an index?
13. What is a key path?
14. What is a transaction?
15. What is an upgrade transaction?
16. Why can an IndexedDB upgrade be blocked?
```

### Cache

```text
17. What does Cache Storage store?
18. How is Cache Storage different from IndexedDB?
19. How would you implement stale-while-revalidate?
20. Why is cache invalidation difficult?
```

### Privacy

```text
21. What is storage partitioning?
22. Why does third-party storage matter?
23. When might Storage Access API matter?
24. What is CHIPS?
```

### Principal

```text
25. How would you design an offline-first application?
26. Which data belongs in IndexedDB vs Cache Storage?
27. How would you migrate IndexedDB schema safely?
28. How would you handle quota exhaustion?
29. How would you secure authentication state?
30. How would you design storage for multiple browser tabs?
```

---

# 131. Predict-the-Behavior Exercises

Predict before running.

### Exercise 1

```js
localStorage.setItem("count", 1);
console.log(typeof localStorage.getItem("count"));
```

### Exercise 2

```js
const first = new URL("https://example.com");
const second = new URL("https://example.com");

console.log(first === second);
```

### Exercise 3

```js
const request = indexedDB.open("demo", 1);

request.onupgradeneeded = () => {
  console.log("upgrade");
};

request.onsuccess = () => {
  console.log("success");
};
```

Predict:

```text
event order
```

for a brand-new database.

### Exercise 4

Two tabs share:

```text
localStorage
```

Tab A runs:

```js
localStorage.setItem("theme", "dark");
```

Predict:

```text
which context can receive the storage event.
```

### Exercise 5

A service worker caches:

```text
/api/profile
```

for user A.

Then user B requests:

```text
/api/profile
```

What questions must you ask before deciding whether the cached response is safe to reuse?

---

# 132. Mastery Exercises

### Exercise 1 — Storage Decision Matrix

For 20 realistic application data types classify:

```text
cookie
localStorage
sessionStorage
IndexedDB
Cache Storage
OPFS
server-side
```

and defend each choice.

### Exercise 2 — Offline Notes App

Build:

```text
IndexedDB
+
outbox
+
service worker cache
+
sync
```

Support:

```text
offline read
offline write
reconnect
idempotency
conflict detection
```

### Exercise 3 — Schema Migration

Create:

```text
v1
→ v2
→ v3
```

and test:

```text
fresh install
v1 user upgrade
v2 user upgrade
failed migration
```

### Exercise 4 — Storage Pressure

Simulate:

```text
cache growth
quota exhaustion
eviction
re-download
```

and design:

```text
LRU-like cleanup
```

for reconstructible data.

### Exercise 5 — Auth Storage Architecture

Compare:

```text
HttpOnly cookie
localStorage token
IndexedDB token
in-memory token
```

under:

```text
XSS
CSRF
reload
multi-tab
offline
token rotation
```

---

# 133. Track A — Core Theory

Master:

```text
cookies
Web Storage
IndexedDB
Cache Storage
OPFS
StorageManager
quotas
eviction
persistence
partitioning
third-party storage
transactions
migrations
service workers
offline architecture
security
```

Deliverable:

```text
explain why each storage primitive exists and where it should not be used.
```

---

# 134. Track B — Implementation

Build:

```text
storage abstraction
IndexedDB wrapper
migration system
outbox
cache manager
quota monitor
cross-tab coordinator
offline sync engine
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

# 135. Track C — Interview / Reasoning

Practice:

```text
“Why not use localStorage for everything?”

“Why does IndexedDB need transactions?”

“What breaks when storage is partitioned?”

“How would you safely cache personalized API responses?”

“What happens if an IndexedDB upgrade is blocked?”

“How do you design offline writes?”

“How do you recover after eviction?”

“How do you secure auth state?”
```

Deliverable:

```text
data classification
+
storage primitive
+
failure mode
+
recovery strategy.
```

---

# 136. Specification / Runtime Source Discipline

Use this hierarchy:

```text
1. relevant Web Platform standards
2. Storage Standard
3. HTML / Fetch / Cookie-related standards
4. browser implementation behavior
5. MDN compatibility/documentation
6. framework wrappers
7. application code
```

Important distinctions:

```text
localStorage/sessionStorage
→ Web Storage API

IndexedDB
→ IndexedDB specification

Cache Storage
→ Cache API / service-worker ecosystem

StorageManager
→ Storage API / Storage Standard ecosystem

cookies
→ HTTP cookie semantics / browser platform
```

Do not collapse:

```text
“browser storage”
```

into one specification or one implementation model.

The Storage Standard provides a shared storage-system model and storage buckets; browser policies still determine quotas, eviction, and partitioning behavior. citeturn406306search6turn406306search1

---

# 137. Current Platform Notes

As of 2026:

```text
third-party state partitioning is an active privacy architecture
CHIPS / Partitioned cookies are supported in current major browsers
Storage Access API exists for controlled access to certain third-party state
storage quotas/eviction remain browser-dependent
```

CHIPS is currently documented by MDN as broadly available across latest browsers/devices since December 2025, while storage partitioning behavior still varies by browser. citeturn406306search0turn406306search4

Therefore:

```text
browser compatibility testing
```

is part of storage engineering.

---

# 138. Principal Decision Framework

Use:

```text
Correctness
Performance
Memory
Security
Privacy
Reliability
Offline resilience
Scalability
Observability
Migration safety
Cross-tab behavior
Browser compatibility
Operational complexity
Future change
```

The key question:

```text
What storage semantics does the data require?
```

not:

```text
Which API is easiest to call?
```

---

# 139. Production Checklist

```text
[ ] data classification defined
[ ] source of truth defined
[ ] storage primitive explicitly selected
[ ] size/growth estimate created
[ ] quota failure handled
[ ] eviction/recovery strategy defined
[ ] privacy/partitioning requirements checked
[ ] authentication storage threat model documented
[ ] IndexedDB migrations tested
[ ] blocked upgrades handled
[ ] transactions defined
[ ] indexes justified
[ ] service-worker caches versioned
[ ] personalized responses protected from unsafe caching
[ ] offline outbox is idempotent
[ ] conflict policy defined
[ ] multi-tab behavior tested
[ ] private browsing tested
[ ] sensitive storage not logged
[ ] storage usage observable
[ ] browser compatibility verified
```

---

# 140. Retrieval Record

```md
# Chapter 132 — Revision / Retrieval Record

## Attempt
- Date:
- Duration:
- Status before:
- Status after:

## Cookies
-

## Web Storage
-

## IndexedDB
-

## Cache Storage
-

## OPFS
-

## StorageManager
-

## Quota / Eviction
-

## Persistence
-

## Partitioning
-

## Third-Party State
-

## Offline Architecture
-

## Security
-

## Migrations
-

## Multi-Tab
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

# 141. Spaced Retrieval Schedule

### Day 0

Study:

```text
storage primitive comparison
cookies
Web Storage
IndexedDB
Cache Storage
```

### Day 1

Classify:

```text
20 kinds of application data
```

into storage primitives.

### Day 3

Trace:

```text
IndexedDB migration
+
blocked upgrade.
```

### Day 7

Build:

```text
offline outbox
```

without notes.

### Day 14

Review:

```text
quota
eviction
partitioning
CHIPS
```

### Day 21

Audit:

```text
authentication storage
service-worker cache
```

for security.

### Day 30

Perform a complete:

```text
browser storage architecture review
```

without notes.

---

# 142. Dependency Graph

```text
Chapter 04
Strings
      ↓
Chapter 24
Objects / Collections
      ↓
Chapter 28
Serialization / Structured Clone
      ↓
Chapter 31
Async Fundamentals
      ↓
Chapter 35
Promises
      ↓
Chapter 49
DOM Architecture
      ↓
Chapter 51
Browser Web APIs
      ↓
Chapter 53
Web Streams
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
Chapter 101
Production Scenarios
      ↓
Chapter 109
Event-Driven Application
      ↓
Chapter 130
Date / Time
      ↓
Chapter 131
URL / Encoding
      ↓
Chapter 132
Browser Storage Architecture
```

Related next chapters:

```text
Chapter 133 → Service Workers / Offline Web Architecture
Chapter 134 → Web Locks / Cross-Tab Coordination
Chapter 137 → Browser Performance APIs
```

---

# 143. Concept Connections

## Depends On

```text
origins
HTTP
cookies
async APIs
structured clone
transactions
service workers
security
performance
testing
```

## Builds Toward

```text
offline-first applications
PWA architecture
sync engines
client databases
browser caches
multi-tab coordination
local-first applications
```

## Related Concepts

```text
Cache invalidation
transactions
consistency
idempotency
event sourcing
outbox pattern
quota
eviction
privacy partitioning
service workers
```

## Concepts Revisited

```text
HTTP
origins
security
serialization
structured cloning
async execution
performance
reliability
observability
```

## Why This Chapter Matters

Browser storage is where:

```text
application state
+
network state
+
privacy policy
+
offline behavior
+
browser resource management
```

meet.

A production application must know:

```text
what can be stored
who can access it
how long it can survive
whether it can be evicted
whether other tabs can see it
whether third-party partitioning changes it
how it migrates
how it recovers
```

That is an architecture problem, not merely an API problem.

---

# 144. Final Principal Mental Model

Use:

```text
DATA
 ↓
classification
 ├── server-visible state
 │      ↓
 │    cookies
 │
 ├── tiny client preference
 │      ↓
 │    Web Storage
 │
 ├── structured durable local data
 │      ↓
 │    IndexedDB
 │
 ├── network representation
 │      ↓
 │    Cache Storage
 │
 └── large local files
        ↓
      OPFS

Then apply:

storage scope
+
partitioning
+
quota
+
persistence
+
security
+
migration
+
recovery
+
observability
```

For offline-first systems:

```text
UI
 ↓
local source-of-work
 ↓
outbox
 ↓
sync
 ↓
server
```

while:

```text
service worker
 ↓
resource/cache layer
```

handles:

```text
static assets
network responses
offline bootstrapping
```

---

# 145. Final Principal Principle

> **Storage architecture begins with data semantics, not API familiarity.**

The production-grade sequence is:

```text
classify data
→ define source of truth
→ select storage primitive
→ define scope/access
→ define security/privacy
→ define quota/eviction behavior
→ define migration
→ define recovery
→ define synchronization
→ test multi-context behavior
→ observe failures
```

The central distinctions to internalize are:

```text
Cookies are HTTP state.

localStorage/sessionStorage are synchronous string key/value stores.

IndexedDB is a structured local database.

Cache Storage is a Request/Response cache.

OPFS is an origin-private file system.

StorageManager describes/controls storage policy where supported.

Quota is not ownership.

Persistence is not permanence.

Partitioning changes third-party state assumptions.

HttpOnly reduces direct script access; it does not eliminate XSS.

A cache is not automatically a source of truth.

Offline persistence requires a recovery strategy.

A storage abstraction is only good when it preserves the semantics
that correctness depends on.
```

At principal level, the question is never merely:

```text
“Where can I put this data?”
```

It is:

```text
“What lifecycle, ownership, consistency, privacy, and recovery guarantees
must this data have?”
```