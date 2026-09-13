# Chapter 134 — Web Locks, Cross-Tab Coordination & Browser Shared State

> **JavaScript Mastery — Part XXIII: Browser Platform & Client State**
>
> **Mission:** Master browser coordination across tabs, windows, iframes, workers, and service workers: Web Locks, BroadcastChannel, `storage` events, shared state, leader election, leases, mutual exclusion, read/write coordination, races, deadlocks, cancellation, multi-context state machines, offline synchronization, authentication/logout propagation, and production-safe browser coordination.
>
> **Role perspective:** Principal JavaScript Engineer · Browser Platform Engineer · Distributed Systems Engineer · Concurrency Engineer · PWA Architect · Security Engineer · Reliability Engineer
>
> **Status:** `[ ] Not Started`
>
> **Core rule:** **Multiple browser contexts turn client-side JavaScript into a distributed system. Shared state needs explicit ownership, coordination, failure handling, and conflict semantics. “It works in one tab” proves almost nothing about multi-context correctness.**

---

# 1. Learning Objectives

By completing this chapter, you should be able to:

```text
[ ] explain why multiple browser contexts create concurrency problems
[ ] distinguish tabs, windows, frames, workers, and service workers
[ ] explain same-origin coordination
[ ] distinguish messaging from mutual exclusion
[ ] explain BroadcastChannel
[ ] explain the storage event
[ ] explain Web Locks
[ ] explain LockManager
[ ] explain Lock
[ ] explain navigator.locks
[ ] explain shared locks
[ ] explain exclusive locks
[ ] explain lock queues
[ ] explain lock callbacks
[ ] explain automatic lock release
[ ] explain lock request cancellation
[ ] explain lock queries
[ ] understand lock lifetime
[ ] understand origin scoping
[ ] understand secure-context requirements
[ ] explain leader election
[ ] explain single-writer architecture
[ ] explain distributed mutual exclusion
[ ] explain leases conceptually
[ ] understand why localStorage is not a lock primitive
[ ] understand why BroadcastChannel is not a lock primitive
[ ] understand why IndexedDB is not automatically a global mutex
[ ] explain race conditions across tabs
[ ] explain lost updates
[ ] explain stale reads
[ ] explain check-then-act races
[ ] explain double initialization
[ ] explain duplicate synchronization
[ ] explain duplicate background work
[ ] explain split-brain ownership
[ ] explain stale leadership
[ ] explain deadlocks
[ ] explain lock ordering
[ ] explain lock granularity
[ ] explain shared vs exclusive modes
[ ] explain fairness considerations
[ ] explain starvation
[ ] explain reentrancy hazards
[ ] understand that browser coordination is not process-global infrastructure
[ ] design a single sync leader across tabs
[ ] coordinate IndexedDB access
[ ] coordinate cache cleanup
[ ] coordinate token refresh
[ ] coordinate logout
[ ] coordinate one-time initialization
[ ] coordinate expensive background work
[ ] coordinate migrations
[ ] coordinate local resource ownership
[ ] design message protocols
[ ] design versioned messages
[ ] distinguish commands from events
[ ] design idempotent messages
[ ] design acknowledgement protocols
[ ] handle duplicate delivery
[ ] handle missed messages
[ ] combine persistent state with ephemeral messaging
[ ] implement leader election
[ ] implement lock-based synchronization
[ ] detect ownership loss
[ ] avoid stale leadership
[ ] design recovery after tab termination
[ ] understand service-worker participation
[ ] understand worker participation
[ ] understand iframe boundaries
[ ] explain partitioning implications
[ ] design authentication state synchronization
[ ] design offline queue ownership
[ ] design cross-tab cache invalidation
[ ] design cross-tab update propagation
[ ] design storage migration coordination
[ ] test multi-context systems
[ ] fuzz race scenarios
[ ] reason about browser lifecycle failures
[ ] choose Web Locks vs messaging vs persistence
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
Chapter 49 — DOM Architecture
Chapter 51 — Browser Web APIs
Chapter 52 — Web Workers / Concurrency
Chapter 53 — Web Streams
Chapter 55 — Fetch / HTTP Networking
Chapter 56 — Browser Security
Chapter 57 — JavaScript Security Engineering
Chapter 63 — Async Context / Diagnostics
Chapter 83 — Observability
Chapter 84 — Reliability
Chapter 85 — Performance
Chapter 86 — Testing
Chapter 101 — Production Scenarios
Chapter 109 — Event-Driven Applications
Chapter 127 — SharedArrayBuffer / Atomics / Memory Model
Chapter 132 — Browser Storage Architecture
Chapter 133 — Service Workers / Offline Architecture
```

Supporting concepts:

```text
distributed systems
mutual exclusion
state machines
message passing
consensus concepts
leases
failure detection
idempotency
```

---

# 3. What Is Cross-Tab Coordination?

A browser application can have multiple independent execution contexts:

```text
Tab A
Tab B
Window C
Iframe D
Dedicated Worker E
Service Worker F
```

Some of these contexts may share:

```text
origin
storage
network state
application identity
```

but each has its own:

```text
JavaScript execution state
event queue
lifecycle
memory
```

Therefore:

```text
same origin
≠
same JavaScript process state.
```

---

# 4. Why Cross-Tab Coordination Exists

Consider an app with:

```text
5 open tabs
```

All tabs detect:

```text
sync required
```

Without coordination:

```text
Tab A → sync
Tab B → sync
Tab C → sync
Tab D → sync
Tab E → sync
```

The server sees:

```text
5 duplicate requests.
```

Coordination can establish:

```text
one leader
→ perform sync
→ notify others
```

---

# 5. Browser Contexts as a Distributed System

Think:

```text
Context A
   ↕
Context B
   ↕
Context C
   ↕
Context D
```

Communication is:

```text
message passing
```

and failure can happen because:

```text
tab closes
worker terminates
network disappears
browser freezes
page crashes
context is backgrounded
```

This resembles:

```text
distributed systems
```

at a small scale.

---

# 6. Three Core Coordination Problems

Most multi-context problems reduce to:

```text
1. Communication
2. Mutual exclusion
3. Durable shared state
```

Examples:

```text
communication:
"logout happened"

mutual exclusion:
"only one tab refreshes token"

durable state:
"this migration has completed"
```

Different browser APIs solve different layers.

---

# 7. Coordination Toolbox

```text
BroadcastChannel
→ message broadcast

storage event
→ simple persistent-state change notification

Web Locks
→ mutual exclusion/shared access

IndexedDB
→ durable structured state/transactions

Service Worker
→ centralized network/background boundary

SharedWorker
→ shared worker execution where supported

postMessage
→ direct context-to-context messaging

WebSocket / Server
→ authoritative external coordination
```

---

# 8. Communication vs Coordination

Messaging:

```text
"something happened"
```

Coordination:

```text
"only one context may do X"
```

Persistence:

```text
"the fact that X happened must survive context termination"
```

These are different guarantees.

---

# 9. BroadcastChannel

`BroadcastChannel` provides a named communication channel shared by same-origin browsing contexts, including documents and workers where supported. Messages are delivered as `message` events to listening channel objects other than the sender. citeturn304420search3turn304420search11

Example:

```js
const channel = new BroadcastChannel("app");

channel.postMessage({
  type: "LOGOUT"
});
```

---

# 10. BroadcastChannel Mental Model

```text
                channel: "app"
                     |
       +-------------+-------------+
       |             |             |
     Tab A         Tab B         Worker C
       |             |             |
     receive       receive       receive
```

This is:

```text
broadcast messaging
```

not:

```text
shared memory
```

and not:

```text	mutex.
```

---

# 11. BroadcastChannel Scope

The channel is available to contexts associated with the same origin.

Therefore:

```text
https://example.com
```

can communicate through the channel with another matching context.

A different origin does not automatically participate.

MDN describes the channel as one named channel for browsing contexts with the same origin. citeturn304420search3

---

# 12. BroadcastChannel Does Not Persist Messages

If:

```text
Tab A
```

broadcasts:

```text
EVENT_X
```

while:

```text
Tab B
```

is not listening, Tab B should not be treated as guaranteed to receive it later.

Therefore:

```text
BroadcastChannel
=
ephemeral notification.
```

For durable facts use:

```text
IndexedDB
or
other persistent state.
```

---

# 13. Event + State Pattern

A robust design combines:

```text
durable state
+
ephemeral notification
```

Example:

```text
IndexedDB:
syncVersion = 42

BroadcastChannel:
"SYNC_COMPLETED"
```

If a tab misses the event, it can read:

```text
syncVersion
```

later.

---

# 14. Storage Event

The `storage` event can notify other documents when Web Storage changes.

Example:

```js
window.addEventListener("storage", event => {
  console.log(event.key, event.newValue);
});
```

This can be useful for:

```text
small cross-tab state changes
```

but it is not a generalized messaging bus.

---

# 15. `storage` Event vs BroadcastChannel

| Capability | storage event | BroadcastChannel |
|---|---|---|
| persistent underlying state | Yes, Web Storage | No |
| message broadcast | Limited | Yes |
| structured-clone payload | No, strings | Yes |
| intended coordination | basic | explicit messaging |
| durable history | No | No |
| lock | No | No |

Use:

```text
storage event
```

when the state itself is meaningful.

Use:

```text
BroadcastChannel
```

for richer event communication.

---

# 16. Why localStorage Is Not a Lock

Bad idea:

```js
if (!localStorage.getItem("lock")) {
  localStorage.setItem("lock", "me");
  doWork();
}
```

Two tabs can execute:

```text
check
```

before either performs:

```text
set
```

Result:

```text
Tab A sees empty
Tab B sees empty
Tab A writes
Tab B writes
both work
```

This is:

```text
check-then-act race.
```

---

# 17. Why Timestamps Do Not Fix Locking

A common custom approach:

```text
lock owner = tabId
lock expiresAt = timestamp
```

This creates a lease-like system.

But now you need to reason about:

```text
clock differences
paused tabs
stale owners
renewal
recovery
split-brain
clock jumps
```

A custom lock protocol is much harder than:

```text
Web Locks
```

for the problems Web Locks is designed to solve.

---

# 18. Web Locks API

The Web Locks API lets same-origin scripts asynchronously acquire a named lock, hold it while work runs, and release it when the callback completes.

It works across:

```text
tabs
workers
```

within the origin. citeturn304420search0turn304420search9

---

# 19. Basic Web Lock

```js
await navigator.locks.request(
  "sync",
  async lock => {
    await synchronize();
  }
);
```

The callback runs once the lock is granted.

The lock is automatically released when the callback's returned Promise settles. citeturn304420search0

---

# 20. Lock Name

The application chooses:

```text
lock name
```

Example:

```text
"sync"
"db-migration"
"token-refresh"
"cache-cleanup"
```

The name represents an abstract shared resource.

MDN explicitly describes lock names as developer-selected resource identifiers. citeturn304420search10

---

# 21. Lock Scope

Web Locks are:

```text
origin-scoped.
```

For example:

```text
https://example.com
```

and:

```text
https://example.org
```

have independent lock managers.

An origin's locks do not affect another origin's locks. citeturn304420search0

---

# 22. Secure Context

The Web Locks API is available in secure contexts in supporting browsers.

Typical production environment:

```text
HTTPS
```

and it is also exposed to workers in supporting environments. citeturn304420search0turn304420search9

---

# 23. Lock Callback

The critical section lives inside:

```js
navigator.locks.request("resource", async lock => {
  // critical section
});
```

The important rule is:

```text
await all work that must be protected
```

inside the callback.

---

# 24. Critical Section

A critical section is:

```text
code that must not overlap
```

with another holder of the same exclusive lock.

Example:

```text
read sync state
calculate next sync
write sync state
```

must all occur inside the same protected region if they require atomic coordination.

---

# 25. Lock Release

Normally:

```text
callback returns
    ↓
lock released
```

This means:

```js
await navigator.locks.request("x", async () => {
  await task();
});
```

naturally expresses:

```text
acquire
→ work
→ release
```

The API manages release through the asynchronous callback lifecycle. citeturn304420search0

---

# 26. Lock Error

If the callback rejects:

```js
await navigator.locks.request("x", async () => {
  throw new Error("failed");
});
```

the lock is still released after the callback completes.

The caller receives the rejection.

Therefore:

```text
lock ownership
```

is tied to:

```text
callback lifetime
```

not to successful business completion.

---

# 27. Automatic Release Is a Major Safety Property

A manual lock API would require:

```text
acquire()
try {
  work()
} finally {
  release()
}
```

Web Locks instead structurally ties:

```text
ownership
```

to:

```text
callback lifetime.
```

This reduces:

```text
forgotten release
```

bugs.

---

# 28. Exclusive Mode

Default mode:

```text
exclusive
```

means:

```text
only one compatible holder
```

can own the lock at a time.

Example:

```js
await navigator.locks.request("sync", {
  mode: "exclusive"
}, async () => {
  await sync();
});
```

The lock interface exposes `mode` as either `"exclusive"` or `"shared"`. citeturn304420search8

---

# 29. Shared Mode

Shared mode allows multiple holders of the same resource when the requests are compatible.

Example:

```js
await navigator.locks.request(
  "data",
  { mode: "shared" },
  async () => {
    await readData();
  }
);
```

Multiple shared readers can coexist according to lock scheduling semantics, while an exclusive writer conflicts with them.

---

# 30. Reader / Writer Pattern

Conceptually:

```text
shared = readers
exclusive = writer
```

Example:

```text
R1 ─┐
R2 ─┼─ can coexist
R3 ─┘

W1 ─→ waits until compatible readers finish
```

This is useful for:

```text
read-heavy coordination
```

but should not be overused.

---

# 31. Lock Requests Queue

If:

```text
Tab A
```

holds:

```text
"sync"
```

and:

```text
Tab B
```

requests the same conflicting lock, its request waits.

The Web Locks API queues compatible/incompatible requests under its scheduling rules. citeturn304420search0turn304420search7

---

# 32. Lock Requests Are Promises

`navigator.locks.request()` returns a Promise.

Therefore:

```js
await navigator.locks.request(...)
```

fits naturally into:

```text
async control flow.
```

The returned Promise resolves after the callback completes and the lock is released. citeturn304420search0

---

# 33. Cancellation

`request()` can accept an:

```text
AbortSignal
```

through options.

Example:

```js
const controller = new AbortController();

const request = navigator.locks.request(
  "sync",
  { signal: controller.signal },
  async () => {
    await sync();
  }
);
```

If the request is aborted before acquisition, it can fail with:

```text
AbortError
```

according to the API. citeturn304420search7

---

# 34. Waiting Forever Is a Design Bug

A lock request can remain pending while another context owns the lock.

Production code should consider:

```text
How long can waiting be acceptable?

Should the request be canceled?

What happens when the page is closed?

What happens if the work becomes irrelevant?
```

Use:

```text
AbortController
```

for appropriate cancellation policies.

---

# 35. Lock Query

The lock manager exposes:

```js
navigator.locks.query()
```

which can inspect held and pending locks for the origin. citeturn304420search5

This is useful for:

```text
diagnostics
debugging
operational inspection
```

---

# 36. `query()` Is a Snapshot

A query result describes the lock manager state at the time the query is observed.

Do not use:

```js
await navigator.locks.query()
```

as a safe:

```text
"is it free?"
```

check followed by:

```text
do work
```

That reintroduces:

```text
check-then-act race.
```

Acquire the lock instead.

---

# 37. Anti-Pattern: Polling Lock State

Bad:

```text
poll query()
→ if empty, assume ownership
```

Correct:

```text
request lock
→ callback runs when ownership is granted
```

Diagnostics are not synchronization.

---

# 38. Leader Election

A common Web Locks pattern is:

```text
each tab requests same lock
```

Only one enters:

```text
leader
```

work.

Others can:

```text
wait
```

or observe that:

```text
someone else owns the task.
```

MDN explicitly describes using a shared lock name to ensure only one tab performs synchronization as a leader-election pattern. citeturn304420search0

---

# 39. Single Sync Leader

Architecture:

```text
Tab A ─┐
Tab B ─┼── request "sync-leader"
Tab C ─┤
Tab D ─┘
          ↓
      one winner
          ↓
       sync loop
```

This can avoid:

```text
duplicate polling
duplicate network work
```

---

# 40. Leader Is Not Permanent

Holding a Web Lock does not create:

```text
permanent leader identity.
```

The leader only owns the lock while:

```text
its callback is running
```

When the context ends the critical operation:

```text
lock releases.
```

Another context may acquire it.

---

# 41. Leader Election vs Consensus

Web Locks can coordinate:

```text
one owner
```

within the browser's origin.

They do not solve:

```text
distributed consensus across servers
```

or:

```text
global cluster leadership
```

Use:

```text
server-side coordination
```

for system-wide authority.

---

# 42. Browser Leader vs Server Leader

Browser:

```text
one tab should perform local sync
```

Server:

```text
one worker cluster should process job X
```

These require different coordination domains.

Do not confuse:

```text
browser-local leader
```

with:

```text
distributed system leader.
```

---

# 43. Leader Notification

After obtaining leadership:

```js
channel.postMessage({
  type: "LEADER_STARTED"
});
```

Other tabs can update:

```text
status UI
```

But important state should also be recoverable without the message.

Use:

```text
persistent state
+
notification.
```

---

# 44. Heartbeats

A custom leader system may use:

```text
heartbeat
```

to indicate liveness.

With Web Locks, you often do not need a custom heartbeat solely to maintain exclusive ownership because:

```text
lock lifetime
```

is already tied to callback execution.

But a UI may still need:

```text
current leader status
```

which can be broadcast separately.

---

# 45. Leases

A lease is conceptually:

```text
time-bounded ownership
```

Custom leases require:

```text
expiry
renewal
clock assumptions
recovery
```

Web Locks provide a different ownership model:

```text
callback-scoped lock.
```

Choose leases only when the domain really needs:

```text
time-based external ownership.
```

---

# 46. Lock Granularity

Bad:

```text
lock "everything"
```

Better:

```text
lock "sync"
lock "cache-cleanup"
lock "migration"
```

Granularity affects:

```text
contention
parallelism
deadlock risk
complexity
```

---

# 47. Too-Coarse Locks

If every operation takes:

```text
global-app-lock
```

then:

```text
unrelated work
```

cannot overlap.

This can reduce:

```text
throughput
responsiveness
```

and create:

```text
contention bottlenecks.
```

---

# 48. Too-Fine Locks

Too many lock names can cause:

```text
complex lock ordering
deadlocks
hard-to-debug dependencies
```

There is no universal:

```text
best lock granularity.
```

Align locks with:

```text
actual shared-resource invariants.
```

---

# 49. Lock Ordering

Suppose:

```text
operation A
→ lock X
→ lock Y
```

while:

```text
operation B
→ lock Y
→ lock X
```

This can create:

```text
deadlock
```

if both acquire their first lock before requesting their second.

---

# 50. Deadlock Example

```text
Tab A:
holds A
waits B

Tab B:
holds B
waits A
```

Neither can proceed.

Web Locks documentation explicitly calls out deadlock risk from out-of-order lock acquisition. citeturn304420search0

---

# 51. Avoid Nested Locks

Prefer:

```text
one lock
→ complete operation
```

rather than:

```text
lock A
→ lock B
→ lock C
```

when the same result can be achieved through:

```text
one higher-level lock.
```

---

# 52. Global Lock Order

If multiple locks are truly required:

```text
define a total order
```

such as:

```text
db
→ cache
→ network
```

and always acquire in that order.

This reduces:

```text
circular wait.
```

---

# 53. Lock Hold Time

Keep critical sections:

```text
short
bounded
focused
```

Bad:

```text
lock
→ wait 30 seconds
→ network
→ user interaction
→ expensive computation
→ release
```

Better:

```text
lock
→ determine ownership/state
→ perform minimum protected operations
→ release
```

---

# 54. Should Network Work Happen Under a Lock?

Sometimes:

```text
yes
```

if the invariant requires:

```text
only one network operation.
```

But often:

```text
no
```

because a long network request creates:

```text
long lock hold
```

Better pattern:

```text
lock
→ claim work / record ownership
→ release
→ network
→ lock
→ commit result
```

This requires a durable state machine.

---

# 55. Claim / Work / Commit

For expensive asynchronous work:

```text
1. acquire lock
2. atomically claim task
3. release lock
4. perform external work
5. acquire lock
6. commit outcome
7. release
```

The durable claim must prevent:

```text
duplicate work
```

and:

```text
stale owners.
```

---

# 56. Idempotency

Even with locking, external requests can be duplicated because:

```text
network failure after server processed request
```

The browser may retry.

Therefore use:

```text
idempotency keys
```

for important mutations.

Locking solves:

```text
local concurrency.
```

It does not solve:

```text
network ambiguity.
```

---

# 57. Web Locks + IndexedDB

A powerful combination:

```text
Web Lock
→ coordinate ownership

IndexedDB
→ persist state
```

Example:

```text
lock "sync"
→ read sync state
→ claim next item
→ write state
→ release
```

This combines:

```text
mutual exclusion
+
durable state.
```

---

# 58. Web Locks + BroadcastChannel

Another combination:

```text
Web Locks
→ who may act

BroadcastChannel
→ tell others what happened
```

Example:

```text
leader acquires lock
→ sync
→ BroadcastChannel("SYNC_DONE")
```

Again:

```text
message = notification
lock = coordination.
```

---

# 59. Web Locks + Service Worker

A service worker can participate in Web Locks in supporting environments.

Possible design:

```text
multiple tabs
        ↓
Web Lock
        ↓
one sync owner
        ↓
service worker/network work
```

But keep the architecture explicit.

The service worker is not automatically:

```text
the permanent leader.
```

---

# 60. BroadcastChannel + Service Worker

Service workers can participate in messaging architectures where the relevant APIs are exposed.

Example conceptual flow:

```text
service worker
→ postMessage
→ clients
```

and:

```text
tabs
→ BroadcastChannel
```

Use a single protocol schema.

---

# 61. Message Envelope

A production message can use:

```js
{
  version: 1,
  type: "SYNC_COMPLETED",
  id: "event-123",
  timestamp: 1720000000000,
  payload: {
    syncVersion: 42
  }
}
```

This supports:

```text
versioning
deduplication
diagnostics
evolution.
```

---

# 62. Event vs Command

Event:

```text
SYNC_COMPLETED
```

means:

```text
fact happened.
```

Command:

```text
START_SYNC
```

means:

```text
please perform action.
```

This distinction helps avoid:

```text
duplicate command
```

and:

```text
event replay confusion.
```

---

# 63. Idempotent Events

An event handler should ideally tolerate:

```text
duplicate event
```

For example:

```js
if (event.id <= lastProcessedId) {
  return;
}
```

The exact deduplication strategy depends on:

```text
ordering
scope
persistence.
```

---

# 64. Missed Events

If correctness depends on:

```text
never missing a BroadcastChannel message
```

the design is fragile.

Instead:

```text
event
→ notification

state
→ authoritative recovery.
```

A new tab should bootstrap by reading:

```text
current state
```

then start listening for:

```text
future events.
```

---

# 65. Bootstrap Race

Potential race:

```text
Tab B reads state
Tab A changes state
Tab B starts listening
```

Tab B may miss the change.

A safer startup sequence:

```text
subscribe
→ read current state
→ reconcile
```

or:

```text
read state
→ subscribe
→ re-read version
→ reconcile if changed.
```

The exact protocol depends on the data model.

---

# 66. Version-Based Synchronization

Use:

```text
stateVersion
```

for durable change tracking.

Example:

```text
version 41
→ version 42
```

A tab can detect:

```text
myVersion < currentVersion
```

and refresh.

This is more reliable than assuming:

```text
one event = one state transition observed by everyone.
```

---

# 67. Monotonic Local Version

For a simple origin-local state:

```text
syncVersion = 1,2,3...
```

can be useful.

But multiple contexts writing simultaneously require:

```text
transactional increment
```

or:

```text
lock.
```

---

# 68. Check-Then-Act Race

Bad:

```js
if (state.version < 5) {
  state.version = 5;
  save(state);
}
```

Two contexts can both observe:

```text
version = 4
```

and both perform work.

Correct:

```text
transaction
+
coordination
```

according to the invariant.

---

# 69. Lost Update

Tab A:

```text
reads count = 10
```

Tab B:

```text
reads count = 10
```

A writes:

```text
11
```

B writes:

```text
11
```

Expected:

```text
12
```

Observed:

```text
11
```

This is a classic:

```text
lost update.
```

---

# 70. Optimistic Concurrency

An alternative is:

```text
version = 10
```

Update requires:

```text
expectedVersion = 10
```

Server or local transaction rejects if:

```text
version !== expectedVersion.
```

This can avoid global locks for some workloads.

---

# 71. Locking vs Optimistic Concurrency

| Strategy | Core Idea |
|---|---|
| lock | prevent conflicting work |
| optimistic versioning | allow work, detect conflict |
| last-write-wins | accept latest state |
| merge | combine changes |
| server authority | centralize conflict decision |

Choose based on:

```text
conflict rate
latency
complexity
failure tolerance.
```

---

# 72. One-Time Initialization

Suppose multiple tabs need:

```text
initialize local schema
```

A lock can ensure:

```text
only one initializer
```

but the initialization result should also be persisted.

Pattern:

```text
lock
→ check durable initialized flag
→ initialize if needed
→ persist initialized flag
→ release
```

---

# 73. Initialization Is Not Just Locking

If Tab A initializes and crashes before:

```text
initialized = true
```

another tab must safely retry.

Therefore:

```text
lock
+
durable state machine
```

is stronger than:

```text
in-memory boolean.
```

---

# 74. Token Refresh Coordination

Without coordination:

```text
5 tabs detect expired token
→ 5 refresh requests
```

A lock can serialize:

```text
token-refresh
```

but each tab should re-check after acquiring:

```text
is token already refreshed?
```

Pattern:

```text
detect expiry
→ acquire lock
→ re-read token state
→ refresh only if still necessary
→ persist
→ release
→ notify tabs
```

---

# 75. Double-Checked Refresh

This is similar to:

```text
double-checked locking
```

but the second check must happen:

```text
inside the lock.
```

Bad:

```text
check outside
→ assume no one changed it
```

Correct:

```text
check
→ acquire
→ check again
→ mutate.
```

---

# 76. Cross-Tab Logout

A good pattern:

```text
logout action
→ clear durable local auth state
→ broadcast LOGOUT
→ each tab clears UI/session state
```

The server remains the authority for:

```text
session revocation.
```

BroadcastChannel provides:

```text
fast notification
```

while durable state handles:

```text
late-opening tab.
```

---

# 77. Logout Race

Suppose:

```text
Tab A logout
Tab B refreshes token simultaneously.
```

Without coordination:

```text
logout
+
refresh
```

can race.

Use:

```text
auth-state lock
+
server-side session policy
```

where required.

The client cannot guarantee:

```text
server revocation
```

by local coordination alone.

---

# 78. Cross-Tab Cache Invalidation

When Tab A changes:

```text
profile
```

it can broadcast:

```text
PROFILE_CHANGED
```

Other tabs can:

```text
invalidate local cache
```

But if the message is missed:

```text
TTL/version
```

provides eventual recovery.

---

# 79. Cache Invalidation Protocol

Example:

```js
channel.postMessage({
  type: "CACHE_INVALIDATED",
  resource: "profile",
  version: 8
});
```

Receiver:

```text
if localVersion < 8:
    invalidate/refetch
```

This creates:

```text
eventual cache convergence.
```

---

# 80. Cross-Tab UI Synchronization

Good candidates:

```text
theme
language
logout
selected workspace
feature rollout
```

Use:

```text
BroadcastChannel
```

for fast propagation.

For durable settings:

```text
persist
+
broadcast.
```

---

# 81. Theme Example

```js
const channel = new BroadcastChannel("settings");

function setTheme(theme) {
  localStorage.setItem("theme", theme);
  channel.postMessage({
    type: "THEME_CHANGED",
    theme
  });
}
```

Other tabs update immediately.

But on startup:

```text
read storage
```

rather than:

```text
wait for event.
```

---

# 82. Resource Ownership

A lock can represent:

```text
who may manipulate a shared resource.
```

Examples:

```text
"sync-engine"
"cache-cleaner"
"db-migrator"
"single-writer"
```

Do not use:

```text
one generic lock
```

for unrelated resources.

---

# 83. Single-Writer Architecture

A powerful pattern:

```text
many readers
+
one writer
```

For example:

```text
Tab A ─┐
Tab B ─┼─ read local state
Tab C ─┤
        ↓
   single sync writer
        ↓
   IndexedDB/network
```

This can simplify:

```text
write conflicts
```

while preserving:

```text
read parallelism.
```

---

# 84. Single Writer + Broadcast

The writer:

```text
writes durable state
```

then broadcasts:

```text
STATE_UPDATED
```

Readers:

```text
invalidate/reload
```

This is:

```text
event-driven local replication.
```

---

# 85. Leader Crash

Suppose the leader tab:

```text
closes during sync.
```

The lock is released because the callback ends with:

```text
context termination
```

and another waiting request can acquire the lock.

But the business operation may still be:

```text
partially completed.
```

Therefore use:

```text
durable state
+
idempotency.
```

---

# 86. Partial Work State

Example:

```text
task:
pending
claimed
processing
completed
failed
```

If leader dies in:

```text
processing
```

another leader needs a recovery rule.

Possible:

```text
lease timestamp
retry marker
idempotency key
server status lookup.
```

Web Locks alone cannot reconstruct external business state.

---

# 87. Lock Ownership vs Business Ownership

Important distinction:

```text
Web Lock ownership:
who may execute this critical section now

Business ownership:
who is responsible for this operation in the domain
```

Do not conflate them.

---

# 88. Queue Processing

Suppose IndexedDB contains:

```text
100 pending mutations
```

Multiple tabs should not process the same item.

One strategy:

```text
lock "outbox-drain"
→ atomically claim item
→ release
→ network
```

Or:

```text
single sync leader
```

drains the entire queue.

---

# 89. Lock Per Item

Possible:

```text
mutation:123
mutation:124
```

locks.

This can increase:

```text
parallelism
```

but also:

```text
complexity
```

and:

```text
lock cardinality.
```

Use only where concurrency materially helps.

---

# 90. Global Sync Lock

Simpler:

```text
lock "sync"
```

then:

```text
process queue
```

This is easier to reason about but can reduce:

```text
parallelism.
```

Start simple.

Optimize after measuring.

---

# 91. Fairness

Applications should not assume a lock scheduling policy stronger than the specification guarantees.

Avoid designs that require:

```text
strict FIFO fairness
```

unless the API specification explicitly provides the guarantee you depend on.

Principal rule:

```text
correctness must not depend on unspecified fairness.
```

---

# 92. Starvation

Starvation occurs when:

```text
one request
```

can repeatedly fail to make progress because other requests keep acquiring compatible/conflicting access.

Design for:

```text
bounded critical sections
```

and:

```text
reasonable contention.
```

---

# 93. Shared Lock Caveat

Shared locks improve read concurrency but can create:

```text
writer delay
```

under heavy reader load.

Do not switch to shared mode just because:

```text
"reads are safe."
```

Measure:

```text
read/write ratio
contention
latency.
```

---

# 94. Lock Request Cancellation

A stale request should not wait forever.

Example:

```text
page hidden
request no longer relevant
→ abort lock request.
```

Use:

```js
AbortController
```

when appropriate.

---

# 95. Page Lifecycle

Browser pages can transition through:

```text
active
backgrounded
frozen
discarded
restored
```

or otherwise experience lifecycle changes.

Coordination code must assume:

```text
execution can pause.
```

Do not create protocols requiring:

```text
every tab responds immediately.
```

---

# 96. Background Tab Delay

A backgrounded page can be throttled.

Therefore:

```text
heartbeat every 1 second
```

is not a reliable browser-wide timing guarantee.

Avoid custom protocols that depend on:

```text
precise tab timing.
```

---

# 97. Browser Suspension

A mobile browser can suspend:

```text
tab
worker
process
```

for an unknown duration.

When it returns:

```text
state may be stale.
```

Always validate:

```text
current durable state
```

after resumption.

---

# 98. Browser Crash

A crash can remove:

```text
ephemeral messages
global variables
in-memory locks
```

but persistent storage may survive.

Therefore:

```text
durable recovery
```

is mandatory for important workflows.

---

# 99. Browser Restart

After restart:

```text
BroadcastChannel messages are gone
worker globals are gone
tab runtime state is gone
```

but some persistent state can remain.

Design startup as:

```text
reconstruct from durable truth.
```

---

# 100. Service Worker Restart

The same principle applies:

```text
service worker memory
```

is not durable.

Coordinate through:

```text
IndexedDB
Cache Storage
Web Locks
```

as appropriate.

---

# 101. SharedWorker

A `SharedWorker` can provide a common JavaScript execution context among pages using a shared worker, where supported.

Conceptually:

```text
Tab A ─┐
Tab B ─┼── SharedWorker
Tab C ─┘
```

This can centralize:

```text
in-memory coordination
```

but introduces:

```text
lifecycle/support complexity.
```

Do not assume every browser/product context supports it equally.

---

# 102. SharedWorker vs Web Lock

SharedWorker:

```text
centralized execution context
```

Web Locks:

```text
coordination primitive
```

A lock does not require:

```text
one permanent shared worker.
```

---

# 103. SharedWorker vs BroadcastChannel

SharedWorker can:

```text
maintain in-memory connection state
route messages
```

BroadcastChannel can:

```text
broadcast messages
```

Choose by:

```text
ownership
state lifetime
message topology.
```

---

# 104. Iframes

Same-origin iframes can participate in same-origin communication mechanisms subject to their browsing context and API access.

Cross-origin iframes are constrained by:

```text
origin security policy
```

and may need:

```text
postMessage
```

for explicit communication.

---

# 105. `postMessage`

`postMessage` is suitable for:

```text
direct context-to-context communication
```

with an explicit recipient/window/worker.

Always validate:

```text
event.origin
```

when receiving cross-origin messages.

---

# 106. BroadcastChannel vs postMessage

| Need | Tool |
|---|---|
| one known peer | postMessage |
| many same-origin contexts | BroadcastChannel |
| exclusive access | Web Locks |
| durable state | IndexedDB |
| network interception | Service Worker |

---

# 107. Partitioning Implications

Modern browsers can partition state in third-party contexts.

Therefore an embedded third-party component may not share the same coordination domain as:

```text
its first-party top-level instance.
```

Do not assume:

```text
same iframe origin string
=
globally shared browser state.
```

Check current browser privacy behavior for the deployment context.

---

# 108. Third-Party Coordination

A third-party widget embedded on:

```text
site-a.example
```

and:

```text
site-b.example
```

may experience separate state/coordination contexts due to partitioning.

This affects:

```text
BroadcastChannel assumptions
storage assumptions
cookie assumptions
lock assumptions
```

where the browsing context is partitioned.

---

# 109. Security Boundary

Do not treat a Web Lock as an authorization mechanism.

A lock only coordinates code that:

```text
participates in the same lock protocol
```

It does not prevent malicious same-origin script from:

```text
ignoring your lock
```

or:

```text
performing the operation directly.
```

---

# 110. XSS and Coordination

An attacker with same-origin script execution may:

```text
listen on BroadcastChannel
request locks
read local storage
access IndexedDB
```

where permitted.

Therefore:

```text
coordination correctness
```

does not replace:

```text
origin security.
```

---

# 111. Message Validation

Never trust:

```js
event.data.type
```

as if it came from a trusted internal module.

Validate:

```text
schema
version
payload types
allowed commands
```

and:

```text
origin/context
```

when relevant.

---

# 112. BroadcastChannel Data Is Application Input

Even same-origin messages should be treated as:

```text
untrusted application data
```

when components are independently deployable or compromised dependencies are possible.

Use:

```text
schema validation
```

for high-value commands.

---

# 113. Sensitive Data in Messages

Avoid broadcasting:

```text
access tokens
passwords
refresh tokens
personal data
```

Use messages such as:

```text
TOKEN_UPDATED
```

or:

```text
SESSION_STATE_CHANGED
```

and let the receiver obtain appropriate state through secure mechanisms.

---

# 114. Auth Refresh Lock Pattern

Production example:

```js
async function getValidToken() {
  return navigator.locks.request(
    "auth-refresh",
    async () => {
      const latest = await readTokenState();

      if (!isExpired(latest)) {
        return latest.accessToken;
      }

      const refreshed = await refreshToken();
      await persistTokenState(refreshed);

      channel.postMessage({
        type: "TOKEN_UPDATED"
      });

      return refreshed.accessToken;
    }
  );
}
```

Important:

```text
re-read state inside lock.
```

---

# 115. Why the Re-Read Matters

Without:

```js
const latest = await readTokenState();
```

inside the lock:

```text
Tab A
→ detects expired
Tab B
→ refreshes
Tab A
→ acquires lock
Tab A
→ refreshes again
```

The lock serialized the operation but did not eliminate:

```text
stale assumptions.
```

---

# 116. Sync Leader Pattern

```js
async function runAsLeader() {
  await navigator.locks.request(
    "sync-leader",
    async () => {
      while (shouldContinue()) {
        await doOneSyncCycle();
      }
    }
  );
}
```

Be careful:

```text
long-running lock
```

may create:

```text
starvation
poor handoff
hard lifecycle behavior.
```

---

# 117. Better Sync Leadership

Prefer bounded cycles:

```text
acquire
→ sync one bounded batch
→ publish state
→ release
→ schedule next attempt
```

This allows:

```text
other contexts
```

to acquire the lock and provides:

```text
more resilient lifecycle behavior.
```

---

# 118. Single-Writer State Machine

Use:

```text
idle
claiming
working
committing
done
failed
```

Persist state transitions.

A leader can then recover work if:

```text
previous context disappears.
```

---

# 119. Work Claim Example

IndexedDB:

```js
{
  id: "task-42",
  status: "pending"
}
```

Under lock:

```text
pending
→ claimed
```

Then release.

Another context sees:

```text
claimed
```

and does not duplicate the task.

---

# 120. Stale Claim Recovery

A claimed task can include:

```js
{
  status: "processing",
  ownerId: "...",
  claimedAt: ...
}
```

Recovery logic can decide:

```text
if old enough
→ reclaim
```

This is a custom lease/state-machine problem.

Use carefully because:

```text
clock assumptions
```

and:

```text
false reclamation
```

must be handled.

---

# 121. Lock vs Lease

Use Web Locks when the requirement is:

```text
coordinate currently executing work.
```

Use durable lease state when the requirement is:

```text
recover ownership after arbitrary external delay.
```

Often the best design uses both:

```text
Web Lock
+
durable claim/lease.
```

---

# 122. Cross-Tab Database Migration

Potential problem:

```text
Tab A → old code
Tab B → migration
Tab C → old code
```

A migration may require:

```text
one coordinator
```

but database upgrade semantics themselves also provide transaction/connection coordination.

Do not add Web Locks blindly.

First understand:

```text
IndexedDB upgrade protocol
```

then use Web Locks only for:

```text
application-level coordination
```

that the database does not provide.

---

# 123. Migration Protocol

Example:

```text
lock "app-migration"
→ check database version
→ migrate
→ write app schema state
→ broadcast MIGRATION_COMPLETE
→ release
```

Other tabs:

```text
receive event
→ reload/reinitialize
```

---

# 124. Cache Cleanup Coordination

If multiple tabs perform:

```text
cache cleanup
```

use:

```text
lock "cache-cleanup"
```

and let one context perform:

```text
cleanup pass.
```

But ask whether:

```text
service worker activation
```

already owns this responsibility.

Avoid duplicate infrastructure.

---

# 125. Cross-Tab Background Polling

Without coordination:

```text
5 tabs
→ 5 polling timers
```

With leader:

```text
one polling owner
→ broadcast new data
```

This reduces:

```text
network traffic
battery
CPU
server load.
```

This is a classic browser coordination optimization.

---

# 126. Battery Considerations

Multi-tab duplicate work can increase:

```text
CPU
network
radio usage
battery drain.
```

Coordination can therefore be a:

```text
performance
and
energy
```

optimization.

---

# 127. Lock Contention as Performance Cost

A lock removes duplicate work but introduces:

```text
waiting.
```

Measure:

```text
lock wait time
critical section time
queue length
timeouts/abort count.
```

---

# 128. Observability

Instrument:

```text
lock name
request start
acquisition time
hold duration
queue delay
abort
error
```

Do not log:

```text
tokens
sensitive payloads
personal data.
```

---

# 129. Lock Diagnostics

`navigator.locks.query()` can help inspect:

```text
held locks
pending requests
```

for debugging/diagnostics. citeturn304420search5

A production diagnostic screen could show:

```text
sync-leader
held: true
pending: 3
```

without exposing sensitive state.

---

# 130. Debugging Stuck Coordination

Ask:

```text
Who owns the lock?

Who is waiting?

How long has it been held?

What operation is inside?

Is the lock nested?

Is the owner suspended?

Is a callback waiting on an external promise?

Is the work actually still relevant?
```

---

# 131. Long Lock Diagnosis

If a lock is held for seconds/minutes:

```text
measure critical section.
```

Break it into:

```text
claim
→ release
→ external work
→ commit
```

where correctness permits.

---

# 132. Deadlock Detection

A lock graph can model:

```text
A waits B
B waits C
C waits A
```

A cycle indicates:

```text
deadlock potential.
```

Build a mental graph for complex protocols.

---

# 133. Avoid Lock Dependency Cycles

Simplest rule:

```text
one lock per operation
```

Next best:

```text
fixed global lock order
```

Avoid:

```text
dynamic nested locking
```

unless the system is simple enough to prove safe.

---

# 134. Lock Naming Conventions

Use:

```text
app:sync
app:auth-refresh
app:db-migration
app:cache-cleanup
app:single-writer
```

This helps:

```text
diagnostics
ownership
documentation.
```

---

# 135. Lock Names Are Shared API

Any same-origin code that knows the name can participate in:

```text
the same coordination protocol.
```

Therefore choose names deliberately and document:

```text
meaning
critical section
mode
owner.
```

---

# 136. Shared Lock Protocol

Example:

```text
readers:
app:data → shared

writer:
app:data → exclusive
```

This creates:

```text
reader/writer coordination.
```

But only use it if:

```text
the protected invariant
```

truly permits concurrent readers.

---

# 137. Lock Context Is Not Persistent Identity

The lock object provides:

```text
name
mode
```

but does not mean:

```text
tab identity
```

Use an explicit:

```text
context ID
```

if business logic needs to identify:

```text
which context performed the work.
```

---

# 138. Context IDs

A tab can create:

```js
const contextId = crypto.randomUUID();
```

and include it in:

```text
diagnostics
events
durable claims
```

Do not use:

```text
tab index
```

because tabs are not stable identifiers.

---

# 139. Ownership Records

Example:

```js
{
  ownerId: contextId,
  startedAt: Date.now(),
  operation: "sync"
}
```

Persist only when recovery/audit requirements justify it.

---

# 140. Time Semantics

For durable coordination timestamps:

```text
wall clock
```

can be useful for:

```text
human/audit timestamps
```

but elapsed timeout measurement should prefer:

```text
monotonic clock
```

when operating within one execution context.

Across contexts, persisted wall-clock timestamps have:

```text
clock uncertainty.
```

Design recovery thresholds conservatively.

---

# 141. Browser Clock Skew

Within the same browser profile, contexts may share a system clock, but:

```text
clock adjustments
sleep/wake
```

can still disrupt naive expiry logic.

Do not make correctness depend on:

```text
exact millisecond lease expiry.
```

---

# 142. Network Coordination

Browser-local coordination can reduce duplicate calls, but the server must still protect against:

```text
duplicate requests
replays
concurrent updates
```

Use:

```text
idempotency
optimistic concurrency
ETags
version checks
transactions
server locks
```

as appropriate.

---

# 143. Browser Lock Is Not Distributed Lock

A Web Lock coordinates:

```text
same-origin browser contexts
```

not:

```text
multiple devices
multiple browser profiles
multiple servers
```

For global distributed ownership use:

```text
server-side coordination infrastructure.
```

---

# 144. Cross-Device State

Two laptops running the same account do not share:

```text
navigator.locks
BroadcastChannel
local IndexedDB
```

directly.

Cross-device synchronization requires:

```text
server
WebSocket
polling
sync protocol
```

etc.

---

# 145. Server as Authority

For critical application facts:

```text
server
```

should generally remain authoritative.

Browser coordination is:

```text
optimization
consistency aid
local workflow control
```

not:

```text
global business authority.
```

---

# 146. Idempotent Sync

A robust sync operation should tolerate:

```text
duplicate delivery
retry
partial failure
out-of-order network completion
```

Use:

```text
operation ID
version
server deduplication
```

where needed.

---

# 147. Conflict-Free Local State

For some domains:

```text
CRDT-like
mergeable state
```

can avoid strict locks.

This is more complex than Web Locks but may provide:

```text
offline multi-writer concurrency.
```

Choose based on domain needs.

---

# 148. Locking vs CRDT

| Requirement | Better Direction |
|---|---|
| one local owner | Web Locks |
| durable claim | IndexedDB + state |
| many independent writes | versioning/merge |
| offline collaborative editing | CRDT/OT-like design |
| server-wide ownership | server coordination |

Do not introduce distributed merge theory when:

```text
one local lock
```

solves the problem.

---

# 149. Broadcast vs State Replication

A broadcast event is:

```text
notification.
```

State replication is:

```text
current value
+
version
+
reconciliation.
```

Production systems often need both.

---

# 150. Reliable Cross-Tab Protocol

A robust pattern:

```text
1. subscribe
2. read durable state
3. process current version
4. listen for events
5. on event, compare version
6. reconcile if behind
7. persist processed version
```

This tolerates:

```text
missed events
duplicate events
late tabs.
```

---

# 151. Message Ordering

Broadcast systems should not assume:

```text
global total order
```

across all contexts unless the relevant specification and protocol provide exactly what the application needs.

If order matters, carry:

```text
sequence/version
```

in the message.

---

# 152. Duplicate Delivery

Design handlers so:

```text
same event twice
```

does not corrupt state.

Use:

```text
idempotent update
```

or:

```text
processed IDs.
```

---

# 153. Out-of-Order Delivery

Suppose:

```text
EVENT version 10
EVENT version 8
```

The receiver should reject/downgrade safely:

```text
ignore version <= currentVersion
```

for monotonic versioned state.

---

# 154. Command Deduplication

Commands like:

```text
START_SYNC
```

may be duplicated.

Attach:

```text
commandId
```

and persist:

```text
command status
```

when the command has durable effects.

---

# 155. Ack Protocol

For important commands:

```text
sender
→ command(id=123)
receiver
→ ack(123)
```

But an acknowledgement itself can be lost.

Thus durable systems need:

```text
retry
idempotency
state reconciliation.
```

---

# 156. Reliable Messaging Is a Bigger Problem

There is no universal:

```text
“just broadcast reliably”
```

primitive for arbitrary application semantics.

Reliability comes from:

```text
durable state
+
idempotent operations
+
versioning
+
reconciliation.
```

---

# 157. One-Time Events

A notification such as:

```text
SHOW_TOAST
```

can be ephemeral.

A business event such as:

```text
PAYMENT_COMPLETED
```

should not exist only as:

```text
BroadcastChannel message.
```

Persist/derive important facts from:

```text
authoritative state.
```

---

# 158. Cross-Tab Notifications

A good event model:

```text
durable state:
notification state/version

broadcast:
"new notification available"
```

On missed event:

```text
tab reads current state.
```

---

# 159. Browser Update Coordination

When a new service worker version becomes available:

```text
worker
→ BroadcastChannel / clients message
→ tabs
```

can coordinate:

```text
reload
```

or:

```text
show update prompt.
```

Use:

```text
version number
```

to avoid inconsistent update handling.

---

# 160. Multi-Tab Update Race

Possible:

```text
Tab A clicks update
Tab B clicks update
```

A lock can serialize:

```text
update activation coordination
```

but often the service-worker lifecycle itself provides enough coordination.

Do not build extra locking until:

```text
browser lifecycle semantics
```

are understood.

---

# 161. Browser Storage + Lock Example

Use:

```text
IndexedDB:
schemaVersion

Web Lock:
db-migration

Broadcast:
MIGRATION_DONE
```

Flow:

```text
tab starts
 ↓
request migration lock
 ↓
read version
 ↓
migrate if needed
 ↓
commit
 ↓
broadcast
 ↓
release
```

---

# 162. Storage Events vs BroadcastChannel for Migration

Use storage event when:

```text
a simple persistent version marker
```

is itself the key state.

Use BroadcastChannel when:

```text
multiple contexts need richer immediate coordination.
```

Use both when:

```text
durability + immediate notification
```

are required.

---

# 163. Testing Cross-Tab Systems

Do not rely only on:

```text
unit tests
```

Use:

```text
multi-page integration tests
multiple browser contexts
workers
service workers
```

where the runtime supports automation.

---

# 164. Race Test Harness

Run:

```text
10 contexts
```

and repeatedly trigger:

```text
same operation.
```

Record:

```text
winner
duplicates
final state
errors
latency
```

---

# 165. Stress Testing

Generate:

```text
open
close
reload
background
foreground
network failure
storage delay
lock contention
```

in randomized sequences.

The goal is to expose:

```text
timing-dependent bugs.
```

---

# 166. Fault Injection

Simulate:

```text
leader closes
network fails
tab sleeps
request aborts
worker restarts
IndexedDB transaction aborts
cache missing
message lost
duplicate event
out-of-order event
```

Then prove:

```text
system converges.
```

---

# 167. Property-Based Invariants

Useful invariants:

```text
at most one exclusive owner
no committed version decreases
duplicate event does not change final state
retry does not duplicate business effect
missing notification does not lose durable state
leader failure eventually permits new owner
```

These are more valuable than:

```text
exact timing tests.
```

---

# 168. Observability Exercise

Record:

```text
contextId
operationId
lockName
requestTime
acquiredTime
releaseTime
stateVersion
```

Then build:

```text
contention dashboard.
```

This transforms:

```text
“multi-tab bug”
```

into:

```text
traceable state transitions.
```

---

# 169. Code Review Exercise

Review:

```js
async function syncIfNeeded() {
  if (localStorage.getItem("syncing")) {
    return;
  }

  localStorage.setItem("syncing", "1");

  try {
    await fetch("/sync");
  } finally {
    localStorage.removeItem("syncing");
  }
}
```

Identify:

```text
check-then-act race
stale lock risk
tab crash leaving marker
no ownership identity
no lease semantics
no durable state machine
no idempotency
```

Then redesign with:

```text
Web Locks
+
durable sync state
```

where appropriate.

---

# 170. Debugging Exercise

Three tabs report:

```text
"refreshing token..."
```

at the same time.

Trace:

```text
token expiry detection
lock requests
state re-read
refresh
persist
broadcast
```

Determine whether:

```text
one refresh
```

or:

```text
multiple refreshes
```

occur and why.

---

# 171. Debugging Stale State

Tab B displays:

```text
version 8
```

while:

```text
IndexedDB = version 11
```

Investigate:

```text
missed broadcast
stale in-memory state
storage-read timing
version handling.
```

The fix should be:

```text
reconciliation,
```

not:

```text
“send more events”
```

alone.

---

# 172. Debugging Deadlock

Build:

```text
lock A
lock B
```

with inverted acquisition order.

Observe:

```text
pending requests
```

through:

```js
navigator.locks.query()
```

and identify the cycle. citeturn304420search5

---

# 173. Common Misconceptions

### Misconception 1

> “All tabs share JavaScript memory.”

Correction:

```text
They have separate execution contexts.
```

### Misconception 2

> “BroadcastChannel is a lock.”

Correction:

```text
It broadcasts messages.
```

### Misconception 3

> “localStorage can be used as a mutex.”

Correction:

```text
Naive check-then-set logic has races.
```

### Misconception 4

> “Web Locks guarantee server-side exclusivity.”

Correction:

```text
They coordinate browser contexts in the origin.
```

### Misconception 5

> “A lock makes network requests exactly once.”

Correction:

```text
The network can still duplicate or ambiguously complete operations.
```

### Misconception 6

> “If a message is broadcast, every future tab will receive it.”

Correction:

```text
BroadcastChannel is ephemeral messaging.
```

### Misconception 7

> “query() tells you whether it is safe to act.”

Correction:

```text
Query is diagnostic; request the lock to coordinate.
```

### Misconception 8

> “One lock means one permanent leader.”

Correction:

```text
Ownership exists only while the protected callback holds the lock.
```

### Misconception 9

> “Background tabs respond quickly.”

Correction:

```text
Browser lifecycle/throttling can delay execution.
```

### Misconception 10

> “A client lock is a security boundary.”

Correction:

```text
It is a cooperation mechanism.
```

---

# 174. Common Mistakes

```text
[ ] treating BroadcastChannel as durable
[ ] using localStorage as a naive mutex
[ ] assuming messages are globally ordered
[ ] ignoring missed events
[ ] failing to version messages
[ ] not making handlers idempotent
[ ] holding locks during long network operations
[ ] creating nested locks without an order
[ ] using query() as acquisition
[ ] assuming fairness
[ ] assuming leader permanence
[ ] forgetting tab/worker termination
[ ] storing critical state only in memory
[ ] treating Web Locks as server coordination
[ ] ignoring partitioning
[ ] broadcasting sensitive data
[ ] failing to validate message payloads
[ ] refreshing tokens independently in every tab
[ ] duplicating service-worker coordination
[ ] using timing-sensitive heartbeats
```

---

# 175. Comparison With Related Concepts

| Mechanism | Primary Guarantee |
|---|---|
| BroadcastChannel | same-origin message broadcast |
| `storage` event | notification of Web Storage changes |
| Web Locks | cooperative mutual exclusion/shared access |
| IndexedDB | durable structured local state |
| SharedWorker | shared execution context where supported |
| Service Worker | network/background lifecycle boundary |
| `postMessage` | direct message passing |
| Server API | authoritative cross-device coordination |
| CRDT-like model | mergeable concurrent state |

---

# 176. Production Architecture Pattern

```text
                 ┌──────────────────┐
                 │     Server       │
                 │  authoritative   │
                 └────────┬─────────┘
                          │
                       network
                          │
       ┌──────────────────▼───────────────────┐
       │             Browser Origin            │
       │                                       │
       │  ┌────────┐  ┌────────┐  ┌────────┐ │
       │  │ Tab A  │  │ Tab B  │  │ Tab C  │ │
       │  └───┬────┘  └───┬────┘  └───┬────┘ │
       │      │            │            │      │
       │      └────────────┼────────────┘      │
       │                   │                   │
       │            BroadcastChannel           │
       │                   │                   │
       │              Web Locks                │
       │                   │                   │
       │              IndexedDB                │
       │                   │                   │
       │             Service Worker            │
       └───────────────────────────────────────┘
```

---

# 177. Production Coordination Pattern

Use:

```text
BroadcastChannel
→ notify

Web Locks
→ coordinate

IndexedDB
→ persist

Service Worker
→ network/background

Server
→ authority
```

This is a strong default architecture for many multi-tab/offline applications.

---

# 178. Decision Matrix

| Requirement | Recommended Starting Point |
|---|---|
| tell all tabs logout happened | BroadcastChannel |
| persist logout state | IndexedDB / cookie/server state |
| one tab refreshes token | Web Lock |
| one tab runs sync | Web Lock / leader pattern |
| remember sync progress | IndexedDB |
| notify sync completion | BroadcastChannel |
| one-time local migration | IndexedDB + lifecycle coordination |
| cache cleanup | Service Worker / lock when needed |
| global business ownership | server |
| complex concurrent document editing | merge/CRDT-style design |

---

# 179. Implementation From Scratch — Mini Coordination Framework

Build:

```text
ContextRegistry
EventBus
LockManager
DurableState
LeaderCoordinator
TaskQueue
```

The objective is:

```text
understand coordination semantics
```

not:

```text
reimplement Web Locks.
```

---

# 180. Implementation Milestone 1 — Event Bus

Create:

```js
class EventBus {
  constructor() {
    this.channel = new BroadcastChannel("app");
  }

  publish(event) {
    this.channel.postMessage(event);
  }

  subscribe(handler) {
    this.channel.addEventListener("message", event => {
      handler(event.data);
    });
  }
}
```

Then add:

```text
schema
validation
event IDs
version
close()
```

---

# 181. Implementation Milestone 2 — Durable State

Build:

```text
State
Version
read()
write()
compareVersion()
```

Use:

```text
IndexedDB
```

for the real browser implementation.

---

# 182. Implementation Milestone 3 — Leader Coordinator

Implement conceptually:

```text
request lock
→ become leader
→ perform bounded batch
→ publish state
→ release
```

Track:

```text
leader state
context ID
last run
```

---

# 183. Implementation Milestone 4 — Task Queue

Represent:

```js
{
  id,
  status,
  payload,
  attempts,
  createdAt
}
```

Implement:

```text
enqueue
claim
complete
retry
fail
```

---

# 184. Implementation Milestone 5 — Recovery

Simulate:

```text
leader crash
```

during:

```text
pending
claimed
processing
committing
```

and define the next-owner behavior.

---

# 185. Implementation Milestone 6 — Versioned Events

Build:

```js
{
  id,
  type,
  version,
  payload
}
```

Then reject:

```text
old
duplicate
malformed
```

events.

---

# 186. Implementation Milestone 7 — Diagnostics

Expose:

```text
current context ID
lock name
leader status
last sync
queue depth
state version
```

Then inspect:

```text
contention
```

during stress tests.

---

# 187. Debugging Exercises

## Exercise A — Duplicate Sync

Open:

```text
8 tabs
```

and trigger:

```text
SYNC.
```

Verify:

```text
one owner
```

per sync cycle.

---

## Exercise B — Leader Failure

Close the leader while it holds:

```text
sync-leader.
```

Verify:

```text
another context can proceed.
```

Then inspect whether:

```text
partially completed work
```

is safely recoverable.

---

## Exercise C — Missed Event

Temporarily disconnect a tab from:

```text
BroadcastChannel.
```

Change durable state.

Reconnect.

Verify:

```text
state reconciliation
```

still succeeds.

---

## Exercise D — Deadlock

Create two lock requests with:

```text
op A: X → Y
op B: Y → X
```

Use:

```js
navigator.locks.query()
```

to inspect the resulting waiting state. citeturn304420search5

---

# 188. Code Review Exercise

Review:

```js
async function refreshToken() {
  if (tokenExpired()) {
    const token = await requestNewToken();
    saveToken(token);
  }
}
```

Across five tabs, identify:

```text
duplicate refresh
stale state
race
logout interaction
network ambiguity
```

Redesign using:

```text
Web Lock
+
state re-read
+
durable token state
+
notification.
```

---

# 189. Interview Questions

### Fundamentals

```text
1. Why is multiple-tab JavaScript a concurrency problem?
2. What is BroadcastChannel?
3. What is the storage event?
4. What is Web Locks?
5. What is the difference between communication and mutual exclusion?
```

### Web Locks

```text
6. What happens when two tabs request the same exclusive lock?
7. What is shared mode?
8. When is a Web Lock released?
9. What does lock request cancellation do?
10. What does navigator.locks.query() provide?
```

### Distributed Reasoning

```text
11. Why is localStorage not a safe mutex?
12. What is a lost update?
13. What is leader election?
14. Why are missed messages a problem?
15. Why do durable state and ephemeral messages work well together?
```

### Production

```text
16. How would you prevent five tabs from refreshing a token simultaneously?
17. How would you design single-tab polling?
18. How would you recover if the leader crashes?
19. How would you coordinate logout?
20. How would you coordinate IndexedDB migrations?
```

### Principal

```text
21. When would you choose a lock vs optimistic concurrency?
22. When would you choose BroadcastChannel vs postMessage?
23. How would you design a browser-local task queue?
24. How do you prevent a browser lock from being mistaken for server authority?
25. How would you design for missed messages and duplicate events?
```

---

# 190. Predict-the-Behavior Exercises

Predict before running.

### Exercise 1

```js
await navigator.locks.request("x", async () => {
  console.log("A");
});

await navigator.locks.request("x", async () => {
  console.log("B");
});
```

Explain:

```text
lock acquisition
release
ordering within one sequential context.
```

### Exercise 2

Two tabs run:

```js
navigator.locks.request("x", async () => {
  await delay(1000);
});
```

Predict:

```text
can both callbacks execute simultaneously?
```

### Exercise 3

Two tabs use:

```text
mode = "shared"
```

on the same lock.

Predict:

```text
whether their callbacks can overlap.
```

### Exercise 4

Tab A:

```js
channel.postMessage({
  type: "SYNC_DONE",
  version: 10
});
```

Tab B starts listening after the message was sent.

Predict:

```text
can Tab B rely on receiving version 10?
```

### Exercise 5

Two tabs execute:

```text
read state
if not refreshing:
  mark refreshing
```

without a lock.

Predict:

```text
whether duplicate refresh is possible.
```

---

# 191. Mastery Exercises

### Exercise 1 — Single Sync Leader

Build:

```text
5+ tabs
→ one sync owner
→ BroadcastChannel notification
→ IndexedDB durable progress
```

### Exercise 2 — Token Refresh Coordinator

Implement:

```text
lock
→ re-read token
→ refresh if necessary
→ persist
→ notify
```

### Exercise 3 — Cross-Tab Logout

Implement:

```text
durable auth state
+
broadcast logout
+
reconciliation on startup.
```

### Exercise 4 — Reliable Event Protocol

Build:

```text
event ID
version
dedupe
state reconciliation
```

### Exercise 5 — Browser Task Queue

Implement:

```text
pending
claimed
processing
completed
failed
```

with:

```text
leader
+
recovery
+
idempotency.
```

### Exercise 6 — Deadlock-Proof Locks

Define:

```text
lock ordering
```

and prove:

```text
no circular acquisition.
```

---

# 192. Track A — Core Theory

Master:

```text
browser contexts
communication
BroadcastChannel
storage events
Web Locks
shared/exclusive modes
leader election
mutual exclusion
lock lifetime
cancellation
deadlocks
starvation
durable state
versioning
reconciliation
partitioning
browser lifecycle
```

Deliverable:

```text
explain any cross-tab race from context → message/state → invariant → recovery.
```

---

# 193. Track B — Implementation

Build:

```text
event bus
versioned messaging
durable state layer
leader coordinator
task queue
single-writer architecture
token refresh coordinator
logout propagation
migration coordinator
diagnostics dashboard
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

# 194. Track C — Interview / Reasoning

Practice:

```text
“Why can't localStorage be used as a mutex?”

“How do you prevent five tabs from polling?”

“What does a Web Lock actually guarantee?”

“Why isn't BroadcastChannel durable?”

“How do you recover from a crashed leader?”

“How do you handle duplicate events?”

“How do you prevent lock deadlocks?”

“When should the server remain the authority?”
```

Deliverable:

```text
race
+
invariant
+
coordination mechanism
+
recovery plan.
```

---

# 195. Specification / Runtime Source Discipline

Use this hierarchy:

```text
1. Web Locks API specification
2. HTML / BroadcastChannel messaging semantics
3. Web Storage semantics
4. IndexedDB semantics
5. Service Worker semantics
6. browser privacy/partitioning rules
7. browser implementation behavior
8. application protocol
```

Important distinctions:

```text
Web Locks
→ synchronization primitive

BroadcastChannel
→ messaging primitive

storage event
→ Web Storage change notification

IndexedDB
→ durable local data

Service Worker
→ lifecycle/network/background component
```

Do not describe:

```text
“BroadcastChannel is a distributed database”
```

or:

```text
“Web Lock is a security boundary”
```

because both statements overstate their guarantees.

The Web Locks API is standardized as an asynchronous origin-scoped coordination mechanism for windows/workers, with exclusive/shared modes and diagnostic lock inspection. citeturn304420search0turn304420search4

---

# 196. Current Platform Notes

As of September 2026:

```text
Web Locks:
widely available in modern browsers, with support documented since 2022.

BroadcastChannel:
widely available in modern browsers and available in workers.

Both:
secure-context considerations apply to Web Locks.

Web Locks:
available through Navigator and WorkerNavigator in supporting environments.

Important:
browser lifecycle, partitioning, private browsing, and worker availability
can still affect the surrounding application architecture.
```

MDN currently marks Web Locks and BroadcastChannel as widely available and documents Web Locks in secure contexts and workers. citeturn304420search0turn304420search3

---

# 197. Principal Decision Framework

For every cross-context problem ask:

```text
1. Which contexts participate?
2. Are they same-origin?
3. Is the requirement messaging or mutual exclusion?
4. Must the fact survive context termination?
5. Can a message be missed?
6. Can a message be duplicated?
7. Does order matter?
8. Who is authoritative?
9. What state is durable?
10. What is the invariant?
11. What work must never overlap?
12. What work may happen concurrently?
13. Is a lock required?
14. Is optimistic concurrency simpler?
15. Is server coordination required?
16. What happens if the leader disappears?
17. Can a tab be suspended?
18. Can the network duplicate operations?
19. Are idempotency keys required?
20. Does storage partitioning change the coordination scope?
21. Is the message protocol versioned?
22. Are secrets excluded from messages?
23. How is contention observed?
24. How is recovery tested?
```

---

# 198. Production Checklist

```text
[ ] participating contexts identified
[ ] origin boundary identified
[ ] partitioning assumptions checked
[ ] communication vs lock requirement distinguished
[ ] durable state identified
[ ] authoritative source identified
[ ] messages versioned
[ ] messages validated
[ ] events idempotent
[ ] duplicate events safe
[ ] missed events recoverable
[ ] state versioning implemented where needed
[ ] lock names documented
[ ] lock granularity justified
[ ] lock hold time bounded
[ ] nested lock policy defined
[ ] global lock order defined if needed
[ ] abort/cancellation defined
[ ] leader failure tested
[ ] browser suspension tested
[ ] browser restart recovery tested
[ ] network retry idempotency tested
[ ] sensitive data excluded from broadcasts
[ ] server remains authority for critical business state
[ ] contention observable
[ ] multi-context stress tests exist
```

---

# 199. Retrieval Record

```md
# Chapter 134 — Revision / Retrieval Record

## Attempt
- Date:
- Duration:
- Status before:
- Status after:

## Browser Contexts
-

## BroadcastChannel
-

## storage Event
-

## Web Locks
-

## Shared / Exclusive
-

## Lock Lifetime
-

## Cancellation
-

## Leader Election
-

## Deadlocks
-

## Starvation
-

## Durable State
-

## Versioning
-

## Reconciliation
-

## IndexedDB
-

## Service Worker
-

## Partitioning
-

## Security
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

# 200. Spaced Retrieval Schedule

### Day 0

Study:

```text
communication
vs
mutual exclusion
vs
durable state.
```

### Day 1

Explain:

```text
BroadcastChannel
storage event
Web Locks
```

without notes.

### Day 3

Build:

```text
single sync leader
```

across multiple tabs.

### Day 7

Implement:

```text
token refresh coordinator
```

with re-checking.

### Day 14

Stress:

```text
leader crash
duplicate events
missed messages
```

### Day 21

Model:

```text
deadlock graph
```

and prove lock ordering.

### Day 30

Perform a complete:

```text
multi-tab concurrency architecture review
```

without notes.

---

# 201. Dependency Graph

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
Chapter 37
Cancellation
        ↓
Chapter 49
DOM / Browser Contexts
        ↓
Chapter 51
Browser APIs
        ↓
Chapter 52
Workers / Concurrency
        ↓
Chapter 55
Fetch / HTTP
        ↓
Chapter 56
Browser Security
        ↓
Chapter 63
Async Context / Diagnostics
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
Chapter 127
Shared Memory / Atomics
        ↓
Chapter 132
Browser Storage
        ↓
Chapter 133
Service Workers
        ↓
Chapter 134
Web Locks / Cross-Tab Coordination
        ↓
Chapter 135
WebRTC / Peer-to-Peer JavaScript
```

Cross-cutting:

```text
Chapter 79 → API contracts
Chapter 98 → coordination anti-patterns
Chapter 101 → production failure scenarios
Chapter 129 → protocol/message parsing
Chapter 131 → URL/origin semantics
Chapter 130 → timestamps/clock semantics
```

---

# 202. Concept Connections

## Depends On

```text
async execution
browser contexts
workers
storage
fetch
security
reliability
observability
```

## Builds Toward

```text
multi-tab applications
offline synchronization
distributed browser coordination
collaborative applications
PWA architecture
client-side task queues
reliable session management
```

## Related Concepts

```text
mutual exclusion
leader election
leases
optimistic concurrency
transactions
message passing
event sourcing
CRDTs
distributed systems
idempotency
```

## Concepts Revisited

```text
Promises
AbortController
Web Workers
Service Workers
IndexedDB
BroadcastChannel
storage
security
performance
reliability
```

## Why This Chapter Matters

Modern browser applications are no longer:

```text
one page
+
one script
```

They are often:

```text
many tabs
+
workers
+
service worker
+
persistent storage
+
network
+
server
```

Once this happens, you are building a:

```text
distributed system inside the browser.
```

The hard bugs are no longer syntax mistakes.

They are:

```text
race conditions
lost updates
duplicate work
stale state
deadlocks
split-brain ownership
missed events
version skew
```

The principal engineer needs an explicit model for:

```text
communication
coordination
durability
authority
recovery.
```

---

# 203. Final Principal Mental Model

Use:

```text
                  BROWSER ORIGIN
                       |
        ┌──────────────┼──────────────┐
        |              |              |
      Tab A          Tab B          Tab C
        |              |              |
        └──────── BroadcastChannel ───┘
                       |
                   notifications

                       +
                       |
                    Web Locks
                       |
                mutual exclusion

                       +
                       |
                   IndexedDB
                       |
                  durable state

                       +
                       |
                Service Worker
                       |
               network/background

                       +
                       |
                    Server
                       |
                 global authority
```

For any operation:

```text
identify invariant
      ↓
identify shared resource
      ↓
choose communication
      ↓
choose coordination
      ↓
persist critical state
      ↓
make operations idempotent
      ↓
handle context failure
      ↓
reconcile missed events
      ↓
observe contention/failure
```

---

# 204. Final Principal Principle

> **Once multiple browser contexts can act on the same logical state, treat the browser as a distributed system with unreliable participants.**

The production-grade sequence is:

```text
define invariant
→ identify participants
→ identify authoritative state
→ separate messages from state
→ choose lock/transaction/versioning strategy
→ bound critical sections
→ make external effects idempotent
→ design leader failure recovery
→ handle missed/duplicate events
→ version the protocol
→ test multi-context races
→ observe contention
→ keep server authority for global business facts
```

The central distinctions to internalize are:

```text
A tab is not shared memory.

BroadcastChannel is messaging, not locking.

The storage event is notification, not a transaction.

Web Locks provide cooperative mutual exclusion.

Lock ownership is temporary.

A lock is not durable business state.

A lock is not a security boundary.

query() is diagnostics, not synchronization.

Messages can be missed.

Important state must be recoverable from durable storage.

Duplicate events must be safe.

Locking does not make network effects exactly once.

Browser-local leadership is not server-wide leadership.

Browser timing is not perfectly reliable.

Suspended/crashed contexts are normal failure cases.

The server remains authoritative for global business state.

```

At principal level, the key question is:

```text
“What invariant must remain true when five tabs, two workers,
a service worker, a flaky network, a browser suspension,
and a new application version all act on the same logical state?”
```

Once that invariant is explicit, the correct combination of:

```text
Web Locks
+
BroadcastChannel
+
IndexedDB
+
Service Worker
+
server-side authority
```

becomes much easier to reason about.