# Chapter 106 — Real-Time WebSocket Systems

> **JavaScript Mastery — Part XX: Projects**
>
> **Project:** Build a production-grade real-time application using WebSockets, with explicit connection lifecycle management, authentication, authorization, rooms, message contracts, ordering, backpressure, reconnection, persistence, presence, observability, horizontal scaling, and failure recovery.
>
> **Role perspective:** Principal JavaScript Engineer · Node.js Architect · Real-Time Systems Engineer · Distributed Systems Engineer · Security Engineer · Reliability Engineer
>
> **Status:** `[ ] Not Started`
>
> **Verification date:** 2026-09-11
>
> **Core rule:** **Real-time systems are distributed systems with a fast feedback loop. A WebSocket connection is not the system; the system must define identity, protocol semantics, lifecycle, delivery guarantees, ordering, resource limits, recovery, and operational behavior.**

---

# 1. Project Mission

Build:

```text
FocusBoard Live
```

A collaborative real-time layer for the FocusBoard API from Chapter 105.

Users should be able to:

```text
connect
authenticate
join projects
receive task updates
send task updates
receive comments
observe presence
subscribe/unsubscribe
reconnect
recover missed events
```

The application must remain correct when:

```text
clients disconnect
clients reconnect
messages are duplicated
messages arrive late
servers restart
network partitions occur
multiple users edit the same resource
one user becomes very active
one tenant becomes very large
one downstream dependency becomes slow
```

---

# 2. Learning Objectives

By completing this chapter, you should be able to:

- Explain WebSocket at protocol and application levels.
- Distinguish HTTP from WebSocket.
- Explain the upgrade/handshake model.
- Design connection lifecycle states.
- Authenticate a real-time connection.
- Authorize subscriptions and actions.
- Design message envelopes.
- Validate inbound messages.
- Design server/client heartbeats.
- Detect dead connections.
- Handle reconnects safely.
- Design event IDs.
- Distinguish ordering from delivery guarantees.
- Understand at-most-once, at-least-once, and effectively-once behavior.
- Handle duplicate events.
- Handle missed events.
- Implement replay/resume.
- Design rooms and subscriptions.
- Design multi-tenant isolation.
- Implement backpressure.
- Bound per-connection memory.
- Handle slow consumers.
- Avoid event-loop blocking.
- Design presence.
- Design fan-out.
- Scale horizontally.
- Select a broker/pub-sub layer when needed.
- Understand sticky sessions and when they help.
- Handle graceful shutdown.
- Integrate persistence with events.
- Use an outbox pattern when required.
- Test real-time behavior.
- Load test long-lived connections.
- Observe connection health.
- Detect connection leaks.
- Secure WebSocket endpoints.
- Defend trade-offs at principal level.

---

# 3. Prerequisites

Recommended:

```text
Chapter 31–38 — Async JavaScript
Chapter 53 — Web Streams
Chapter 55 — HTTP Networking
Chapter 57 — Security Engineering
Chapter 58–63 — Node.js Runtime
Chapter 79 — API Design
Chapter 81 — Database Integration
Chapter 82 — API Architecture
Chapter 83 — Observability
Chapter 84 — Reliability
Chapter 85 — Performance
Chapter 86–89 — Testing / Debugging / Review
Chapter 98–101 — Judgment
Chapter 104 — Production HTTP Client
Chapter 105 — Node REST API
```

---

# 4. Why Real-Time?

A traditional REST request often follows:

```text
client
→ request
→ server
→ response
```

Real-time systems add:

```text
client
↕
persistent connection
↕
server
```

Either side may initiate useful application messages after the connection is established.

---

# 5. When WebSockets Fit

Good candidates:

```text
chat
collaboration
presence
live dashboards
interactive multiplayer features
real-time notifications
device control
low-latency state updates
```

---

# 6. When WebSockets Do Not Fit

Do not use WebSockets simply because they are fast.

For:

```text
ordinary CRUD
cacheable reads
simple command/response APIs
large bulk transfers
```

HTTP may be simpler.

Alternatives include:

```text
Server-Sent Events
polling
long polling
HTTP streaming
WebTransport
message queues
```

---

# 7. Core Mental Model

Think of the system as:

```text
HTTP connection establishment
        ↓
WebSocket upgrade
        ↓
persistent connection
        ↓
authentication
        ↓
subscription
        ↓
messages
        ↓
heartbeat
        ↓
reconnect/recovery
        ↓
close
```

---

# 8. WebSocket Is a Protocol

A framework may give:

```js
socket.on("message", ...)
```

but that does not define:

```text
authorization
ordering
retries
event identity
persistence
recovery
application semantics
```

Those are your system design responsibilities.

---

# 9. HTTP vs WebSocket

HTTP commonly models:

```text
independent request/response exchanges
```

WebSocket provides:

```text
long-lived bidirectional messaging
```

The application can use HTTP for:

```text
authentication
initial data
resource CRUD
```

and WebSocket for:

```text
live updates
```

---

# 10. Hybrid Architecture

Recommended starting architecture:

```text
REST API
 ├── authentication
 ├── CRUD
 └── initial state

WebSocket
 ├── subscriptions
 ├── live events
 ├── presence
 └── real-time commands
```

This keeps durable resource semantics in HTTP while using WebSocket where continuous updates add value.

---

# 11. Protocol Architecture

Application flow:

```text
WebSocket transport
        ↓
frame decoding
        ↓
message schema validation
        ↓
authentication context
        ↓
authorization
        ↓
command/event router
        ↓
application service
        ↓
persistence
        ↓
event publication
        ↓
subscribers
```

---

# 12. Connection State Machine

Define states:

```text
CONNECTING
AUTHENTICATING
CONNECTED
SUBSCRIBED
DRAINING
CLOSING
CLOSED
```

Do not allow operations that are invalid for the current state.

---

# 13. State Transitions

```text
CONNECTING
   ↓
AUTHENTICATING
   ↓
CONNECTED
   ↓
SUBSCRIBED
   ↓
DRAINING
   ↓
CLOSING
   ↓
CLOSED
```

Failure paths may jump directly to:

```text
CLOSING
```

---

# 14. Transport Authentication

Possible approaches:

```text
cookie/session
authorization during handshake
short-lived connection token
subprotocol-based credentials where justified
```

Never assume that opening a TCP/WebSocket connection means the caller is authorized.

---

# 15. Authentication vs Authorization

Authentication:

```text
Who is this connection?
```

Authorization:

```text
What may this connection observe or change?
```

---

# 16. Authenticate Before Sensitive Subscription

Do not allow:

```text
connect
→ receive private project data
→ authenticate later
```

unless the protocol intentionally isolates unauthenticated content.

---

# 17. Connection Principal

Store a connection-scoped principal:

```js
{
  userId,
  tenantId,
  roles,
  connectionId
}
```

Avoid storing this in global mutable variables.

---

# 18. Tenant Context

Every subscription should have a clear tenant scope.

Example:

```text
tenant A
  └── project 123
```

must never leak into:

```text
tenant B
  └── project 123
```

even if IDs collide.

---

# 19. Message Envelope

Use a stable envelope:

```json
{
  "type": "task.updated",
  "id": "evt_01...",
  "version": 17,
  "occurredAt": "2026-09-11T00:00:00Z",
  "data": {
    "taskId": "task_123"
  }
}
```

---

# 20. Event ID

Every durable/important event should have a unique identifier.

Examples:

```text
UUID
ULID
database sequence-derived ID
```

Choose according to ordering and storage requirements.

---

# 21. Event Sequence

If replay is supported, use a monotonic stream position such as:

```text
sequence: 18442
```

A unique event ID and an ordered sequence solve different problems.

---

# 22. Event Type

Use explicit names:

```text
task.created
task.updated
task.deleted
comment.created
presence.joined
presence.left
```

Avoid generic:

```text
update
data
message
```

when consumers need reliable semantics.

---

# 23. Event Schema

Example:

```json
{
  "type": "task.updated",
  "id": "evt_123",
  "sequence": 2001,
  "aggregateId": "task_9",
  "version": 8,
  "data": {
    "title": "Review architecture",
    "status": "open"
  }
}
```

Define what each field means.

---

# 24. Command vs Event

Command:

```text
"Please update task"
```

Event:

```text
"Task was updated"
```

Commands express intent.

Events express facts.

---

# 25. Command Example

```json
{
  "type": "task.update",
  "requestId": "req_123",
  "data": {
    "taskId": "task_9",
    "title": "New title"
  }
}
```

---

# 26. Event Example

```json
{
  "type": "task.updated",
  "id": "evt_123",
  "data": {
    "taskId": "task_9",
    "title": "New title"
  }
}
```

---

# 27. Request ID

Commands should often carry a client-generated request ID.

Purpose:

```text
correlation
deduplication
debugging
```

It is not automatically a globally unique event ID.

---

# 28. Validate Every Inbound Message

Never trust:

```text
event type
IDs
versions
room
payload
```

Validate at the transport boundary.

---

# 29. Unknown Event Types

Choose a policy:

```text
ignore
return protocol error
close connection
```

A compatibility-friendly client often benefits from ignoring unknown server events.

Server-side command handling may require stricter rejection.

---

# 30. Message Size Limits

Set:

```text
maximum frame/message size
```

Do not accept unbounded messages.

---

# 31. Serialization

JSON is easy to inspect and integrate.

Alternatives:

```text
MessagePack
CBOR
Protocol Buffers
binary custom formats
```

Optimize only when measurement justifies it.

---

# 32. JSON Risks

Large payloads can cause:

```text
memory growth
CPU cost
event-loop blocking
```

Keep messages bounded.

---

# 33. Connection Limits

Set limits for:

```text
connections per process
connections per user
connections per tenant
subscriptions per connection
```

---

# 34. Connection Registry

A process may maintain:

```js
Map<connectionId, connection>
```

for local routing.

Do not treat local memory as durable distributed state.

---

# 35. User Registry

For presence you may maintain:

```text
user
→ active connection IDs
```

One user can have:

```text
browser
phone
tablet
```

simultaneously.

---

# 36. Presence Is Not Identity

A logged-in user is not necessarily:

```text
currently online
```

Presence reflects observed connection/session state.

---

# 37. Presence Semantics

Define:

```text
online
idle
away
offline
```

and exactly when each state changes.

---

# 38. Heartbeats

Long-lived connections need failure detection.

Use:

```text
ping/pong
application heartbeat
```

according to the stack.

---

# 39. Heartbeat Purpose

A heartbeat helps detect:

```text
dead TCP path
dead peer
stalled connection
```

It does not prove the application is healthy in every sense.

---

# 40. Heartbeat Interval

Do not pick blindly.

Balance:

```text
failure detection speed
network overhead
server CPU
mobile battery
```

---

# 41. Idle Timeout

Consider closing connections that violate policy:

```text
no heartbeat
maximum lifetime
authentication expiry
```

---

# 42. Authentication Expiry

Long-lived connections complicate:

```text
token expiry
permission changes
role changes
logout
```

Design reauthentication or server-side invalidation where required.

---

# 43. Permission Change

If a user loses access to a project while connected:

```text
stop sending new project events
```

Potentially:

```text
unsubscribe
```

or terminate the connection according to policy.

---

# 44. Subscription Model

Example:

```json
{
  "type": "subscribe",
  "requestId": "req_22",
  "topic": "project:42"
}
```

Server must authorize:

```text
user
→ project 42
```

before subscription.

---

# 45. Do Not Trust Topic Names

The client may send:

```text
project:secret
```

That does not grant access.

---

# 46. Subscription Registry

Local model:

```text
connection
  ├── project:1
  ├── project:2
  └── tenant:user
```

---

# 47. Topic Scope

Choose a hierarchy:

```text
tenant
project
resource
user
```

Do not create arbitrary topic namespaces without authorization rules.

---

# 48. Fan-Out

One event may need to reach:

```text
1
10
1,000
100,000
```

connections.

The system design must account for fan-out cost.

---

# 49. Fan-Out Strategies

Options:

```text
direct process-local broadcast
broker/pub-sub
shared event log
database change feed
hybrid
```

---

# 50. Single-Process Architecture

Start simple:

```text
Node process
 ├── HTTP
 ├── WebSocket
 ├── local subscriptions
 └── database
```

This is excellent for the first implementation.

---

# 51. Horizontal Scaling Problem

With:

```text
Node A
Node B
```

a subscriber connected to A may need an event produced on B.

Process-local memory is insufficient.

---

# 52. Shared Broker

Introduce:

```text
Node A
   ↕
broker
   ↕
Node B
```

The broker distributes events between processes.

---

# 53. Redis Pub/Sub

Common architectural choice.

Trade-offs include:

```text
simple fan-out
fast delivery
non-durable messages
```

Do not mistake pub/sub for a durable event log.

---

# 54. Durable Stream

For replay/recovery, use a durable log/stream or database-backed event store where appropriate.

Concept:

```text
event 100
event 101
event 102
...
```

Clients can resume from a known position.

---

# 55. Pub/Sub vs Durable Log

Pub/Sub:

```text
great for live fan-out
```

Durable stream:

```text
better for replay/recovery
```

Some systems combine both.

---

# 56. Replay

A reconnecting client sends:

```json
{
  "type": "resume",
  "lastSequence": 1902
}
```

Server computes:

```text
1903...
```

and sends missing events.

---

# 57. Replay Window

Define:

```text
retention
maximum replay size
```

If the gap is too large:

```text
force snapshot reload
```

---

# 58. Snapshot + Replay

Robust recovery pattern:

```text
initial snapshot
+
events after snapshot position
```

---

# 59. Snapshot Semantics

The snapshot must have a known position:

```text
snapshotPosition = 500
```

then replay:

```text
501...
```

Without a consistent boundary, clients can miss or duplicate state.

---

# 60. Ordering

There are multiple ordering questions:

```text
global ordering
tenant ordering
topic ordering
aggregate ordering
connection ordering
```

Do not promise more ordering than required.

---

# 61. Aggregate Ordering

For a task:

```text
version 1
version 2
version 3
```

clients should not apply:

```text
3
→ 1
→ 2
```

without a recovery strategy.

---

# 62. Sequence Numbers

A monotonic stream position helps detect:

```text
gap
duplicate
reordering
```

---

# 63. Duplicate Events

At-least-once systems can deliver:

```text
same event twice
```

Clients should deduplicate durable events by ID/sequence as appropriate.

---

# 64. Exactly Once

Do not casually promise exactly-once delivery across distributed systems.

Instead define:

```text
transport guarantee
application deduplication
business effect semantics
```

---

# 65. At-Most-Once

Potentially:

```text
message delivered zero or one time
```

Failure can mean loss.

---

# 66. At-Least-Once

Potentially:

```text
message delivered one or more times
```

Consumers must tolerate duplicates.

---

# 67. Effectively Once

Can be achieved for some business operations by combining:

```text
deduplication
idempotency
durable state
```

It is an application property, not magic transport behavior.

---

# 68. Backpressure

Real-time systems fail when producers can generate data faster than consumers can process it.

---

# 69. Slow Consumer

Example:

```text
server emits 10,000 events/sec
client consumes 100/sec
```

Unbounded buffering is dangerous.

---

# 70. Backpressure Policy

Choose:

```text
drop
coalesce
disconnect
slow producer
sample
prioritize
```

based on event semantics.

---

# 71. Coalescing

For dashboard updates:

```text
temperature = 20
temperature = 21
temperature = 22
```

may become:

```text
temperature = 22
```

if intermediate states are unnecessary.

---

# 72. Never Coalesce Everything

For:

```text
financial transaction
audit event
workflow transition
```

dropping intermediate events may violate correctness.

---

# 73. Priority Queues

Possible categories:

```text
critical
normal
ephemeral
```

Drop ephemeral events before critical events.

---

# 74. Queue Bounds

Each connection should have a bounded outbound queue.

Example policy:

```text
max queued bytes
max queued messages
```

---

# 75. Disconnect Slow Consumers

A bounded queue followed by controlled disconnect is safer than unbounded memory growth.

---

# 76. Memory Accounting

Track:

```text
queued message bytes
subscription count
buffered frames
connection metadata
```

---

# 77. Broadcast Amplification

One 10 KB event sent to:

```text
50,000 connections
```

implies substantial outbound work.

Measure fan-out.

---

# 78. Event Payload Minimization

Prefer:

```json
{
  "type": "task.updated",
  "id": "evt_42",
  "data": {
    "taskId": "task_1",
    "changes": {
      "status": "done"
    }
  }
}
```

over sending huge aggregate objects for every small change.

---

# 79. Patch vs Full State

Patch:

```text
lower bytes
more client complexity
```

Full state:

```text
simpler clients
larger payload
```

Choose intentionally.

---

# 80. Event Contracts

Define:

```text
required fields
optional fields
versioning
unknown-field behavior
```

---

# 81. Event Versioning

Example:

```text
task.updated.v1
task.updated.v2
```

or envelope versioning.

Keep compatibility strategy explicit.

---

# 82. Schema Evolution

Prefer additive changes when possible:

```text
new optional field
```

over breaking changes.

---

# 83. Client Compatibility

Old clients may remain connected during deployment.

Server should understand:

```text
client version
protocol capabilities
```

where required.

---

# 84. Capability Negotiation

Possible handshake metadata:

```text
protocolVersion
features
compression
```

Avoid needless complexity until compatibility requires it.

---

# 85. Compression

WebSocket compression can reduce bandwidth but costs CPU and may have security implications.

Enable deliberately.

---

# 86. Broadcast Authorization

When publishing:

```text
project 42 changed
```

the system must know which subscribers are allowed to receive it.

---

# 87. Authorization at Subscription Time

Subscription authorization is efficient.

But permissions can change later.

Therefore:

```text
authorization
+
membership change invalidation
```

both matter.

---

# 88. Event Redaction

Different users may receive different fields.

Example:

```text
admin
→ internal fields

member
→ normal fields
```

Do not broadcast sensitive fields to all subscribers and expect clients to hide them.

---

# 89. Topic Partitioning

Possible architecture:

```text
project:42:public
project:42:private
```

where useful.

---

# 90. Multi-Tenant Broadcast

Never use:

```text
global "task.updated"
```

without tenant-aware filtering if events contain tenant-sensitive data.

---

# 91. Connection Authentication vs Message Authorization

A user can be authenticated but still send:

```text
delete task
```

without permission.

Check authorization per command.

---

# 92. Command Rate Limits

Protect:

```text
send message
subscribe
unsubscribe
search command
bulk action
```

not only connection count.

---

# 93. Connection Rate Limits

Limit:

```text
new connections/sec
connections/IP
connections/user
```

to prevent churn attacks.

---

# 94. Reconnect Storm

A server outage causes:

```text
100,000 clients
→ reconnect simultaneously
```

This can become a second outage.

---

# 95. Exponential Backoff

Clients should use:

```text
base delay
× exponential growth
+
jitter
```

---

# 96. Jitter

Randomness prevents synchronized retries.

---

# 97. Maximum Backoff

Bound retries:

```text
min(maxDelay, computedDelay)
```

---

# 98. Reconnect Does Not Mean Fresh State Automatically

After reconnect:

```text
authentication
subscription
state recovery
```

must happen in a defined sequence.

---

# 99. Reconnect State Machine

Example:

```text
DISCONNECTED
→ CONNECTING
→ AUTHENTICATING
→ RESUMING
→ SUBSCRIBING
→ LIVE
```

---

# 100. Recovery Failure

If resume is impossible:

```text
client fetches fresh snapshot
→ receives current stream position
→ resumes
```

---

# 101. Connection Close Codes

Define application policies for:

```text
normal closure
policy violation
authentication failure
message too large
server shutdown
```

Use protocol-compatible behavior from the WebSocket implementation.

---

# 102. Graceful Server Shutdown

On shutdown:

```text
stop accepting new connections
→ mark draining
→ notify clients
→ stop new subscriptions/commands
→ close after bounded drain period
```

---

# 103. Shutdown Notification

Example:

```json
{
  "type": "server.draining",
  "retryAfterMs": 1000
}
```

Clients can reconnect to another instance.

---

# 104. Draining Connections

Do not kill all sockets immediately when deployment can tolerate a drain period.

---

# 105. Connection Lifetime

Some environments have infrastructure timeouts.

Applications should tolerate:

```text
planned reconnect
```

even when nothing is wrong.

---

# 106. Proxies and Load Balancers

Verify:

```text
WebSocket upgrade support
idle timeout
connection timeout
TLS termination
maximum connections
```

---

# 107. Sticky Sessions

Sticky sessions can simplify:

```text
connection-local state
```

but can create:

```text
uneven load
failover limitations
operational coupling
```

A shared state/broker architecture can reduce dependence on stickiness.

---

# 108. Statelessness

The WebSocket connection itself is stateful.

The broader application can still be designed so important state is recoverable from:

```text
database
event log
shared broker
```

rather than only process memory.

---

# 109. Connection Registry and Restart

Process restart destroys local connection state.

Clients should reconnect.

Never design durable business state to exist only in RAM.

---

# 110. Event Persistence

Persist events when clients must recover after disconnects and gaps.

Ephemeral presence events may not need durable storage.

---

# 111. Outbox Integration

For durable domain events:

```text
DB transaction
 ├── business mutation
 └── outbox event
```

Then:

```text
outbox publisher
→ broker
→ WebSocket servers
→ clients
```

---

# 112. Why Outbox Helps

Without an outbox:

```text
DB commit
→ publish event
```

can fail between steps.

Result:

```text
state changed
but no event
```

---

# 113. Duplicate Publishing

Outbox consumers may publish duplicate events.

Use:

```text
event ID
deduplication
idempotent consumers
```

where required.

---

# 114. Event Ordering and Database Transactions

Commit order and observation order can differ across distributed paths.

Define ordering at the event stream boundary explicitly.

---

# 115. Presence Architecture

For scalable presence:

```text
connection heartbeat
→ presence registry
→ TTL/lease
```

Do not depend only on a client saying:

```text
"I am online."
```

---

# 116. Presence Expiry

If heartbeats stop:

```text
presence expires
```

after a policy-defined interval.

---

# 117. Presence Flapping

Mobile/network conditions can create rapid:

```text
online
offline
online
```

events.

Debounce if product semantics allow.

---

# 118. Presence Consistency

Presence is often eventually consistent.

Do not use presence as an authority for critical security decisions.

---

# 119. Chat Semantics

For chat messages, define:

```text
message ID
sender
conversation
createdAt
sequence
delivery state
```

---

# 120. Chat Duplicate Handling

Client retries can create duplicates.

Use:

```text
clientMessageId
```

or equivalent deduplication key.

---

# 121. Chat Ordering

Choose:

```text
server sequence
```

rather than trusting client timestamps for ordering.

---

# 122. Client Clock

Never trust client timestamps for authoritative ordering.

---

# 123. Collaborative Editing

Task edits can use:

```text
version
```

or more advanced conflict-free approaches.

WebSockets do not automatically solve collaborative conflict resolution.

---

# 124. Optimistic UI

Clients may update immediately:

```text
local state
```

then reconcile with:

```text
server event
```

---

# 125. Reconciliation

Server remains authoritative for:

```text
permissions
versions
business rules
durable state
```

---

# 126. Ack Semantics

A command can receive:

```json
{
  "type": "ack",
  "requestId": "req_123",
  "status": "accepted"
}
```

Define whether ack means:

```text
received
validated
persisted
published
delivered
```

These are not equivalent.

---

# 127. Strong vs Weak Acknowledgement

Weak:

```text
server received command
```

Strong:

```text
transaction committed
```

Document the guarantee.

---

# 128. Command Result

For commands requiring direct response:

```json
{
  "type": "task.update.result",
  "requestId": "req_1",
  "data": {
    "taskId": "task_9",
    "version": 8
  }
}
```

---

# 129. Event and Command Correlation

Use:

```text
requestId
eventId
aggregateId
```

to connect:

```text
client action
→ server command
→ DB mutation
→ published event
```

in observability.

---

# 130. Error Messages

Example:

```json
{
  "type": "error",
  "requestId": "req_123",
  "code": "VERSION_CONFLICT",
  "message": "Task version is stale"
}
```

Avoid raw stack traces.

---

# 131. Protocol Errors

Distinguish:

```text
malformed protocol message
```

from:

```text
business-rule rejection
```

---

# 132. Connection-Terminating Errors

Examples may include:

```text
invalid protocol
message too large
authentication failure
abusive behavior
```

Not every application error should close the socket.

---

# 133. Per-Message Error

A single bad task update may produce:

```text
error event
```

without disconnecting a healthy client.

---

# 134. Authentication Error

Decide whether to:

```text
reject handshake
```

or:

```text
connect unauthenticated then authenticate
```

The first is often simpler for protected APIs.

---

# 135. Origin Checks

Browser WebSocket clients can send an Origin header.

Validate trusted origins where the deployment requires browser-origin controls.

---

# 136. CSRF-Like Concerns

Cookie-authenticated WebSocket endpoints can be exposed to cross-site request contexts if origin/cookie policies are weak.

Treat browser-origin security deliberately.

---

# 137. Cookies

If using cookies:

```text
Secure
HttpOnly
SameSite
```

and origin policy should be aligned with the application's threat model.

---

# 138. Token in URL

Avoid putting long-lived secrets in:

```text
wss://api.example.com/socket?token=...
```

because URLs can leak into logs/telemetry.

Prefer an appropriately designed authentication channel.

---

# 139. Subprotocols

WebSocket subprotocol negotiation can communicate application protocol choices.

Do not misuse subprotocols as a generic authorization bypass.

---

# 140. Injection

Validate message fields before using them in:

```text
SQL
shell
filesystem
URLs
HTML
logs
```

The transport does not make content safe.

---

# 141. Log Injection

A client-controlled message should not create arbitrary log structure or misleading multiline logs.

Use structured logging.

---

# 142. Denial of Service

Threats:

```text
connection flood
message flood
subscription explosion
large payloads
slow consumers
reconnect storms
fan-out amplification
```

---

# 143. Resource Quotas

Consider:

```text
max connections
max subscriptions
max message rate
max message size
max queued bytes
max replay range
```

---

# 144. Per-Tenant Quotas

Large tenants may need separate limits.

---

# 145. Abuse Detection

Track:

```text
messages/sec
disconnect rate
invalid messages
reconnect rate
fan-out volume
```

---

# 146. Observability

Core metrics:

```text
websocket_connections_active
websocket_connections_opened_total
websocket_connections_closed_total
websocket_messages_received_total
websocket_messages_sent_total
websocket_message_errors_total
websocket_queue_bytes
websocket_queue_messages
websocket_reconnects_total
websocket_replay_events_total
```

---

# 147. Connection Duration

Track:

```text
connection lifetime
```

to identify:

```text
churn
stuck sockets
unexpected short sessions
```

---

# 148. Close Reason Metrics

Group close reasons:

```text
normal
timeout
auth
policy
server_shutdown
too_large
rate_limit
network
```

Avoid unbounded label cardinality.

---

# 149. High Cardinality

Do not use arbitrary:

```text
userId
connectionId
eventId
```

as metric labels in high-volume systems unless your metrics system is specifically designed for it.

Use logs/traces for per-entity detail.

---

# 150. Structured Logs

Useful:

```text
connectionId
requestId
userId
tenantId
eventType
subscription
closeCode
```

Only include sensitive identifiers according to policy.

---

# 151. Tracing

Trace:

```text
HTTP mutation
→ DB
→ outbox
→ broker
→ WebSocket delivery
```

to explain latency and loss.

---

# 152. Message Latency

Measure:

```text
event occurred
→ event published
→ event received by server
→ event queued
→ event sent
```

---

# 153. End-to-End Latency

Where possible:

```text
producer timestamp
→ consumer observation
```

but distinguish client clock skew from server timestamps.

---

# 154. Queue Latency

A rising outbound queue indicates:

```text
slow consumer
network bottleneck
fan-out overload
server saturation
```

---

# 155. Memory Leaks

Potential causes:

```text
forgotten event listeners
connection registry entries
subscriptions not removed
timers not cleared
queues never drained
closures retaining data
```

---

# 156. Cleanup

On close:

```text
remove connection
remove subscriptions
clear timers
release resources
```

Use one idempotent cleanup path.

---

# 157. Idempotent Cleanup

Multiple close paths can happen.

Cleanup must safely tolerate:

```text
timeout
server shutdown
client close
protocol error
```

triggering close logic more than once.

---

# 158. Listener Management

Never attach global listeners per connection unless they are removed.

---

# 159. Timer Management

Heartbeat timers must be cleared on close.

---

# 160. Database Integration

Do not keep a DB transaction open because a WebSocket client is waiting for a response.

---

# 161. Real-Time Command Transaction

Use:

```text
validate
→ authorize
→ short DB transaction
→ commit
→ publish event
```

with outbox when publication reliability matters.

---

# 162. Do Not Use Socket as Transaction

The socket is transport.

The database defines durable state.

---

# 163. Durable State vs Ephemeral State

Durable:

```text
task
comment
message
membership
```

Ephemeral:

```text
typing
cursor
online indicator
temporary presence
```

Design these differently.

---

# 164. Typing Indicators

Typing is ephemeral.

Do not persist every:

```text
key press
```

to the database.

---

# 165. Throttling Ephemeral Events

Use:

```text
throttle
coalesce
TTL
```

when product semantics allow.

---

# 166. Cursor Presence

Collaborative cursors may be:

```text
best effort
short-lived
high frequency
```

They should not block durable updates.

---

# 167. Event Classification

Classify every event as:

```text
durable
ephemeral
replayable
non-replayable
critical
best-effort
```

This classification drives architecture.

---

# 168. Replayable Events

Examples:

```text
task.created
task.updated
comment.created
```

---

# 169. Non-Replayable Events

Examples:

```text
typing.started
cursor.moved
hovered
```

---

# 170. Event Retention

Retain replayable events only as long as recovery requirements justify.

---

# 171. Recovery Window

Example:

```text
last event position
→ 30 minutes behind
```

Could replay.

A client that is:

```text
7 days behind
```

may require a fresh snapshot.

---

# 172. Snapshot Endpoint

Reuse REST:

```http
GET /projects/:id/state
```

then:

```text
resume from sequence
```

This is a powerful hybrid pattern.

---

# 173. REST + WebSocket Recovery

```text
WebSocket disconnect
      ↓
reconnect
      ↓
try resume
      ↓
gap too large?
   ↙      ↘
 yes       no
 ↓          ↓
REST        replay
snapshot
```

---

# 174. Duplicate Recovery

If the client receives:

```text
event 100
event 101
event 101
event 102
```

deduplicate:

```text
101
```

before applying durable state.

---

# 175. Gap Recovery

If client has:

```text
100
```

and receives:

```text
102
```

detect:

```text
missing 101
```

Then:

```text
pause application
→ request replay
```

or fetch snapshot.

---

# 176. Apply Order

A robust client can maintain:

```text
lastAppliedSequence
```

and apply only valid next events.

---

# 177. Out-of-Order Events

If ordering is not guaranteed globally, buffer only within a bounded window.

Never create infinite reorder buffers.

---

# 178. Server Ordering

If event order is critical, establish it at a shared source:

```text
database sequence
event log
single partition
```

rather than relying on wall-clock timestamps.

---

# 179. Clocks

Use timestamps for:

```text
display
auditing
latency
```

not authoritative distributed ordering unless semantics explicitly allow it.

---

# 180. Event Time vs Processing Time

Distinguish:

```text
occurredAt
publishedAt
sentAt
receivedAt
```

This is useful during production debugging.

---

# 181. Command Ordering

Commands from one client can race with:

```text
other client
automation
background job
```

The server must use domain concurrency rules.

---

# 182. Optimistic Concurrency in WebSocket Commands

Example:

```json
{
  "type": "task.update",
  "data": {
    "taskId": "task_1",
    "expectedVersion": 7,
    "title": "Changed"
  }
}
```

Reject if current version differs.

---

# 183. Conflict Event

Return:

```json
{
  "type": "error",
  "code": "VERSION_CONFLICT"
}
```

or send authoritative state depending on the UX.

---

# 184. Multi-Device Consistency

One user may have:

```text
browser A
browser B
phone
```

All should receive the same durable event sequence for shared state where required.

---

# 185. Self-Event Policy

When a client performs:

```text
task.update
```

should it receive:

```text
its own task.updated event?
```

Usually yes for state convergence, but define semantics explicitly.

---

# 186. Echo Suppression

If clients optimistically update local state, avoid double-applying their own event.

Use:

```text
requestId
eventId
```

to reconcile.

---

# 187. Acknowledgement vs Event

A command acknowledgment answers:

```text
did the command succeed?
```

An event answers:

```text
what state fact occurred?
```

Use both when clients need both.

---

# 188. Client Library Design

A production client should hide:

```text
reconnect
heartbeat
subscription restoration
replay
dedupe
message parsing
```

from product code where possible.

---

# 189. Client API Example

```js
const live = createLiveClient({
  url,
  tokenProvider
});

await live.connect();

const unsubscribe =
  live.subscribe(
    "project:42",
    event => {
      // reconcile state
    }
  );
```

---

# 190. Subscription Cleanup

Return an unsubscribe function.

Always clean it up when UI/component scope ends.

---

# 191. Browser Integration

UI frameworks may remount components.

Avoid accidentally creating:

```text
multiple sockets
```

for the same logical session.

---

# 192. React-Like Lifecycle

Potential failure:

```text
component mounts
→ socket
component remounts
→ second socket
```

Use an intentional connection owner.

---

# 193. One Connection vs Many

One shared connection:

```text
lower connection overhead
centralized protocol
```

Many connections:

```text
simpler isolation
possibly easier ownership
```

Choose by architecture.

---

# 194. Tabs

Multiple browser tabs can create multiple sockets.

Advanced optimization:

```text
SharedWorker
BroadcastChannel
```

can coordinate tabs.

Do not add complexity without measurable need.

---

# 195. Background Tabs

Browsers may throttle timers/network behavior.

Do not assume heartbeat timers behave exactly like active foreground execution.

---

# 196. Mobile Networks

Expect:

```text
IP changes
NAT expiry
sleep
radio transitions
temporary loss
```

Reconnect must be normal.

---

# 197. Offline Queue

For some commands:

```text
client stores pending commands
```

and retries after reconnect.

Only do this when business semantics are safe.

---

# 198. Offline Commands

Each queued command needs:

```text
idempotency
expiry
ordering
conflict policy
```

---

# 199. Conflict Resolution

Possible policies:

```text
server wins
client wins
manual merge
field-level merge
version conflict
CRDT/OT
```

---

# 200. CRDT / Operational Transform

Use advanced collaborative algorithms only when the product genuinely requires simultaneous rich editing.

Do not implement them merely because "real-time" sounds distributed.

---

# 201. Testing Architecture

Test:

```text
protocol
connection lifecycle
authorization
message validation
reconnect
replay
ordering
dedupe
backpressure
shutdown
scaling
```

---

# 202. Unit Tests

Good candidates:

```text
state machine
message validators
authorization policy
sequence handling
deduplication
backoff calculation
```

---

# 203. Integration Tests

Use actual:

```text
HTTP server
WebSocket server
database
broker
```

where needed.

---

# 204. Connection Lifecycle Test

Verify:

```text
connect
authenticate
subscribe
receive
unsubscribe
close
```

---

# 205. Authorization Test

Verify:

```text
tenant A cannot subscribe to tenant B
```

---

# 206. Command Authorization Test

Verify authenticated users still cannot:

```text
modify unauthorized tasks
```

---

# 207. Reconnect Test

Simulate:

```text
server disconnect
client reconnect
subscription restore
```

---

# 208. Replay Test

Given:

```text
lastSequence = 100
server has 101..105
```

client receives:

```text
101..105
```

in correct order.

---

# 209. Replay Gap Test

Given:

```text
client = 10
server retention starts at 100
```

server requires:

```text
fresh snapshot
```

---

# 210. Duplicate Event Test

Deliver:

```text
event 101
event 101
```

and verify one state transition.

---

# 211. Ordering Test

Deliver:

```text
1
3
2
```

and verify:

```text
gap detection
```

or appropriate buffering behavior.

---

# 212. Slow Consumer Test

Simulate a client consuming slowly.

Verify:

```text
queue remains bounded
```

and:

```text
policy activates
```

before process memory is exhausted.

---

# 213. Message Flood Test

Send:

```text
thousands of commands/sec
```

and verify:

```text
rate limit
disconnect
or load shedding
```

as defined.

---

# 214. Message Size Test

Send a payload above the allowed size.

Expected:

```text
rejection
```

with safe connection behavior.

---

# 215. Reconnect Storm Test

Simulate:

```text
10,000 clients
```

reconnecting simultaneously.

Measure:

```text
CPU
memory
accept rate
DB load
broker load
```

---

# 216. Broker Failure Test

Disconnect the shared broker.

Decide:

```text
drop live events
buffer limited
reject new subscriptions
degrade to local-only
```

based on business requirements.

---

# 217. Database Failure Test

A command requiring the DB should fail safely.

The socket should not necessarily die.

---

# 218. Event Publisher Failure

Test:

```text
DB commit succeeds
publisher unavailable
```

Outbox recovery should eventually publish the event.

---

# 219. Shutdown Test

During active connections:

```text
SIGTERM
```

Expected:

```text
drain
notify
close
exit
```

within a bounded deadline.

---

# 220. Memory Test

Create and destroy many connections.

Verify:

```text
registry returns to baseline
timers cleaned
subscriptions cleaned
```

---

# 221. Listener Leak Test

Open/close sockets repeatedly.

Monitor:

```text
listener count
heap
connection object count
```

---

# 222. Load Test Metrics

Capture:

```text
active connections
connection establishment rate
messages/sec
fan-out/sec
CPU
heap
network throughput
p95/p99 delivery latency
queue depth
broker latency
DB latency
```

---

# 223. Capacity Model

Approximate:

```text
total outbound work
≈
events/sec × subscribers/event × message size
```

This is more useful than thinking only in:

```text
connections
```

---

# 224. Connection Cost

Each connection consumes resources:

```text
socket state
TLS state where applicable
buffers
application objects
timers
subscriptions
memory
```

---

# 225. 100K Connections

Do not assume:

```text
100,000 connections
```

is safe because the operating system supports many sockets.

Measure actual application memory, event processing, network usage, broker usage, and operational limits.

---

# 226. CPU Cost

Sources:

```text
JSON parse/stringify
compression
authentication
authorization
routing
broadcast
metrics
logging
```

---

# 227. Memory Cost

Sources:

```text
connection objects
subscription sets
queued messages
parsed payloads
replay buffers
```

---

# 228. Network Cost

Sources:

```text
heartbeats
events
acks
reconnect handshakes
TLS
compression framing
```

---

# 229. Compression Trade-Off

Compression:

```text
lower bandwidth
higher CPU
possible memory cost
```

Benchmark representative payloads.

---

# 230. Broker Trade-Off

Broker adds:

```text
dependency
latency
cost
operational complexity
```

but enables:

```text
cross-instance fan-out
```

---

# 231. Durable Log Trade-Off

Durability adds:

```text
storage
retention
consumer management
replay complexity
```

Use it when recovery requirements justify it.

---

# 232. Backpressure Trade-Off

Disconnecting a slow client preserves server health but reduces user experience.

The correct answer depends on event semantics.

---

# 233. Reliability Priorities

When overloaded:

```text
protect process
protect durable state
protect critical events
degrade ephemeral features
```

---

# 234. Graceful Degradation

Possible:

```text
typing indicators disabled
presence delayed
live dashboards sampled
critical events preserved
```

---

# 235. Service-Level Objectives

Define:

```text
connection success rate
message delivery latency
event loss tolerance
reconnect recovery time
```

---

# 236. Availability vs Freshness

A client can have:

```text
connected socket
```

but stale data if:

```text
event pipeline broken
```

Health should measure more than socket count when freshness matters.

---

# 237. End-to-End Health

Potential indicator:

```text
synthetic client
→ command
→ persistence
→ event
→ receive
```

This can catch failures invisible to process liveness.

---

# 238. Security Checklist

```text
[ ] authentication
[ ] authorization
[ ] tenant isolation
[ ] origin policy
[ ] message validation
[ ] size limits
[ ] command rate limits
[ ] connection limits
[ ] reconnect controls
[ ] secret-safe logging
[ ] SSRF protections where applicable
[ ] injection protections
[ ] audit logging
[ ] secure TLS boundary
```

---

# 239. Project Milestones

## Milestone 1 — Basic Connection

Build:

```text
HTTP server
WebSocket endpoint
connect
send
receive
close
```

## Milestone 2 — Protocol

Add:

```text
envelopes
schema validation
commands
events
acks
errors
```

## Milestone 3 — Authentication

Add:

```text
principal
tenant
authorization
```

## Milestone 4 — Subscriptions

Add:

```text
subscribe
unsubscribe
rooms
```

## Milestone 5 — Durable Integration

Add:

```text
DB
transactions
outbox
event publisher
```

## Milestone 6 — Recovery

Add:

```text
sequence
replay
snapshot
dedupe
```

## Milestone 7 — Reliability

Add:

```text
heartbeat
backpressure
rate limits
shutdown
```

## Milestone 8 — Scale

Add:

```text
broker
multiple processes
load balancer
```

## Milestone 9 — Production

Add:

```text
observability
security testing
load testing
incident playbooks
deployment
```

---

# 240. Project API

HTTP:

```text
POST /auth/login
GET /projects/:id
GET /projects/:id/tasks
```

WebSocket:

```text
wss://api.example.com/live
```

Commands:

```text
subscribe
unsubscribe
task.update
comment.create
presence.update
resume
ping
```

Events:

```text
task.created
task.updated
task.deleted
comment.created
presence.changed
server.draining
error
```

---

# 241. Example Subscribe

Client:

```json
{
  "type": "subscribe",
  "requestId": "req_1",
  "topic": "project:42"
}
```

Server:

```json
{
  "type": "subscribed",
  "requestId": "req_1",
  "topic": "project:42",
  "position": 8012
}
```

---

# 242. Example Event

```json
{
  "type": "task.updated",
  "id": "evt_900",
  "sequence": 8013,
  "aggregateId": "task_12",
  "version": 5,
  "occurredAt": "2026-09-11T00:00:00Z",
  "data": {
    "taskId": "task_12",
    "changes": {
      "status": "done"
    }
  }
}
```

---

# 243. Example Command

```json
{
  "type": "task.update",
  "requestId": "req_88",
  "data": {
    "taskId": "task_12",
    "expectedVersion": 4,
    "changes": {
      "status": "done"
    }
  }
}
```

---

# 244. Example Conflict

```json
{
  "type": "error",
  "requestId": "req_88",
  "code": "VERSION_CONFLICT",
  "message": "The task was changed by another operation"
}
```

---

# 245. Example Resume

```json
{
  "type": "resume",
  "requestId": "req_90",
  "lastSequence": 8008
}
```

Server:

```text
8010
8011
8012
8013
```

If event 8009 was permanently unavailable, the server must not silently pretend the client is synchronized.

---

# 246. Example Snapshot Recovery

```text
client position: 10
server oldest replayable: 100
```

Response:

```json
{
  "type": "resume_unavailable",
  "reason": "history_gap",
  "snapshotUrl": "/projects/42"
}
```

Then:

```text
GET snapshot
→ receive position 250
→ resume 251+
```

---

# 247. Example Backpressure Policy

```text
queue < 1 MB
→ normal

1–5 MB
→ coalesce ephemeral events

5–10 MB
→ drop low-priority events

> 10 MB
→ close slow connection
```

These numbers are examples only.

Tune using measurement.

---

# 248. Implementation Skeleton

A minimal connection object:

```js
function createConnection(socket, context) {
  return {
    socket,
    connectionId: context.connectionId,
    principal: context.principal,
    subscriptions: new Set(),
    outboundQueue: [],
    lastSequence: 0,
    state: "CONNECTED"
  };
}
```

Do not treat this as a production implementation yet.

It must later handle:

```text
bounded queues
cleanup
state transitions
metrics
timeouts
errors
```

---

# 249. Message Router

Concept:

```js
async function handleMessage(connection, message) {
  validateMessage(message);

  switch (message.type) {
    case "subscribe":
      return subscribe(connection, message);

    case "task.update":
      return updateTask(connection, message);

    case "resume":
      return resume(connection, message);

    default:
      return sendProtocolError(connection, message);
  }
}
```

Keep authorization and application operations explicit.

---

# 250. Event Publisher

Concept:

```js
async function publishTaskUpdated(event, broker) {
  await broker.publish(
    `tenant:${event.tenantId}`,
    event
  );
}
```

Do not put authorization rules inside the generic broker adapter unless the architecture explicitly requires it.

---

# 251. Connection Delivery

Concept:

```js
function deliver(connection, event) {
  if (!isSubscribed(connection, event)) {
    return;
  }

  if (!isAuthorized(connection, event)) {
    return;
  }

  enqueueBounded(connection, event);
}
```

In a mature system, authorization may be derived from subscription state plus permission invalidation rules to avoid repeated expensive checks.

---

# 252. Cleanup

One owner should handle:

```js
function cleanupConnection(connection) {
  clearHeartbeat(connection);
  removeFromRegistries(connection);
  removeSubscriptions(connection);
  releaseResources(connection);
}
```

Make it idempotent.

---

# 253. Reconnect Backoff

Concept:

```js
function getDelay(attempt, {
  base = 250,
  max = 30_000
} = {}) {
  const exponential =
    Math.min(max, base * 2 ** attempt);

  const jitter =
    Math.random() * exponential * 0.25;

  return exponential + jitter;
}
```

The exact policy should be tested.

---

# 254. Sequence Validation

Concept:

```js
function classifySequence(lastApplied, next) {
  if (next === lastApplied + 1) {
    return "next";
  }

  if (next <= lastApplied) {
    return "duplicate";
  }

  return "gap";
}
```

This is a protocol utility, not a universal ordering solution.

---

# 255. Track A — Core Theory

Study:

```text
WebSocket protocol
HTTP upgrade
persistent connections
connection state machines
message envelopes
authentication
authorization
subscriptions
fan-out
ordering
delivery guarantees
replay
snapshots
backpressure
heartbeats
presence
reconnect
brokers
durable logs
outbox
distributed systems
observability
security
```

---

# 256. Track B — Implementation

Build:

```text
WebSocket server
connection manager
protocol validator
auth boundary
authorization policy
subscription manager
event publisher
event consumer
presence registry
heartbeat
backpressure
reconnect client
sequence tracking
dedupe
replay
snapshot recovery
outbox publisher
broker integration
metrics
logs
tracing
tests
load harness
shutdown
```

---

# 257. Track C — Interview / Reasoning

Defend:

```text
Why WebSocket instead of SSE?
Why hybrid HTTP + WebSocket?
Why these event types?
Why sequence IDs?
Why not exactly-once?
Why replay?
Why snapshot + replay?
Why broker?
Why durable log?
Why disconnect slow consumers?
Why this heartbeat?
Why this reconnect strategy?
Why sticky sessions or no sticky sessions?
Why outbox?
Why eventual presence?
Why these limits?
```

---

# 258. Mastery Gate

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

# 259. Debugging Exercises

## Exercise 1 — Missing Subscription Cleanup

Symptom:

```text
heap grows after reconnects
```

Investigate:

```text
subscriptions
timers
listeners
connection registry
```

---

## Exercise 2 — Duplicate Events

Symptom:

```text
task moves to done twice
```

Investigate:

```text
publisher duplication
reconnect replay
client retry
```

---

## Exercise 3 — Event Gap

Symptom:

```text
100
102
```

Investigate:

```text
dropped event
multiple broker paths
reorder
```

---

## Exercise 4 — Slow Consumer

Symptom:

```text
heap rising
```

Investigate:

```text
outbound queue
fan-out
network
```

---

## Exercise 5 — Reconnect Storm

Symptom:

```text
CPU 100%
```

Investigate:

```text
synchronized reconnect
auth burst
subscription burst
DB load
```

---

## Exercise 6 — Cross-Tenant Event Leak

Symptom:

```text
tenant A sees tenant B event
```

Investigate:

```text
topic design
tenant context
authorization
cache
broker routing
```

---

## Exercise 7 — Stale Permission

Symptom:

```text
revoked user still receives private events
```

Investigate:

```text
subscription lifetime
permission invalidation
event authorization
```

---

## Exercise 8 — Lost Event

Symptom:

```text
DB says task updated
client never sees update
```

Investigate:

```text
transaction
outbox
publisher
broker
subscriber
```

---

# 260. Code Review Exercise

Review:

```js
socket.on("message", async raw => {
  const message = JSON.parse(raw);

  if (message.type === "task.update") {
    await db.tasks.update({
      id: message.taskId,
      data: message.data
    });
  }
});
```

Find at least:

```text
validation
authorization
tenant scope
error handling
size limits
concurrency
audit
event publication
rate limits
```

---

# 261. Code Review Exercise

Review:

```js
for (const socket of sockets) {
  socket.send(JSON.stringify(event));
}
```

Identify risks:

```text
slow consumers
memory
fan-out cost
serialization
exceptions
authorization
tenant isolation
```

---

# 262. Code Review Exercise

Review:

```js
setInterval(() => {
  socket.send(JSON.stringify({
    type: "ping"
  }));
}, 1000);
```

Find:

```text
timer cleanup
server/client heartbeat semantics
unnecessary traffic
connection state
```

---

# 263. Code Review Exercise

Review:

```js
const room = socket.handshake.query.room;
rooms.get(room).add(socket);
```

Identify:

```text
authorization
arbitrary room access
room existence
tenant leakage
subscription limits
cleanup
```

---

# 264. Interview Questions — Senior

1. How does WebSocket differ from HTTP?
2. What happens during the WebSocket handshake?
3. How do you authenticate WebSocket connections?
4. How do you authorize subscriptions?
5. How do you handle reconnects?
6. How do you prevent duplicate events?
7. How do you detect event gaps?
8. What is backpressure?
9. How do you handle slow consumers?
10. How would you scale WebSockets across Node processes?

---

# 265. Interview Questions — Principal

1. Design a WebSocket system for 1M concurrent connections.
2. What state remains local and what becomes shared?
3. How do you guarantee per-project ordering?
4. How do clients recover after missing events?
5. How do you design broker failure semantics?
6. How do you avoid reconnect storms?
7. How do you combine Postgres and a durable event stream?
8. When would you choose WebSocket vs SSE vs WebTransport?
9. How would you isolate tenants under high fan-out?
10. How do you prove the system does not silently lose durable business events?

---

# 266. Predict-the-Output Exercises

Before reading the answer, predict.

## Exercise A

```js
const events = [1, 2, 2, 3];

let last = 0;

for (const event of events) {
  if (event === last + 1) {
    last = event;
  }
}

console.log(last);
```

Questions:

```text
What prints?
What mistake does this simplistic logic hide?
```

---

## Exercise B

```js
const events = [1, 3, 2];
let last = 0;

for (const event of events) {
  if (event <= last) continue;

  if (event !== last + 1) {
    console.log("gap");
    break;
  }

  last = event;
}
```

Predict:

```text
output
```

Then explain the difference between:

```text
duplicate
gap
reordering
```

---

## Exercise C

```js
const queue = [];

queue.push("critical");
queue.push("typing");
queue.push("typing");

console.log(queue.length);
```

The output is trivial.

The architectural question is not.

Ask:

```text
Should every queued event be treated equally?
```

---

# 267. Mastery Exercises

## Level 1 — Single Server

Build:

```text
connect
authenticate
subscribe
publish
close
```

---

## Level 2 — Real-Time Tasks

Add:

```text
task.update
task.updated
version
ack
```

---

## Level 3 — Presence

Add:

```text
online
offline
heartbeat
```

---

## Level 4 — Recovery

Add:

```text
sequence
resume
replay
snapshot
```

---

## Level 5 — Backpressure

Add:

```text
bounded outbound queue
priority
coalescing
disconnect policy
```

---

## Level 6 — Distributed

Run:

```text
Node A
Node B
broker
DB
```

and ensure cross-node events work.

---

## Level 7 — Durable Events

Add:

```text
outbox
publisher
dedupe
replay retention
```

---

## Level 8 — Production

Add:

```text
limits
metrics
logs
traces
load tests
security tests
graceful shutdown
deployment
```

---

# 268. Production Acceptance Criteria

```text
[ ] connection lifecycle documented
[ ] authentication implemented
[ ] authorization implemented
[ ] tenant isolation tested
[ ] protocol envelope defined
[ ] inbound validation
[ ] message size limits
[ ] connection limits
[ ] subscription limits
[ ] command rate limits
[ ] heartbeat
[ ] dead connection detection
[ ] graceful close
[ ] bounded outbound queue
[ ] slow consumer policy
[ ] event IDs
[ ] sequence positions
[ ] duplicate handling
[ ] gap detection
[ ] replay
[ ] snapshot recovery
[ ] reconnect backoff
[ ] jitter
[ ] durable-event strategy
[ ] outbox where required
[ ] broker architecture
[ ] scaling strategy
[ ] presence strategy
[ ] observability
[ ] security controls
[ ] integration tests
[ ] reconnect tests
[ ] failure injection
[ ] load tests
[ ] memory/leak tests
[ ] shutdown tests
```

---

# 269. Operational Checklist

Before production:

```text
[ ] load balancer supports upgrade
[ ] proxy timeouts understood
[ ] TLS boundary understood
[ ] connection limits
[ ] broker capacity
[ ] DB capacity
[ ] event retention
[ ] replay limits
[ ] reconnect strategy
[ ] heartbeat policy
[ ] slow-consumer policy
[ ] shutdown policy
[ ] monitoring
[ ] alerting
[ ] runbooks
```

---

# 270. Incident Playbook — Connection Explosion

Symptoms:

```text
active sockets rapidly increasing
memory rising
```

Investigate:

```text
client reconnect loop
load balancer retry
authentication failure
deployment
```

Mitigate:

```text
connection rate limit
backoff
capacity
```

---

# 271. Incident Playbook — Event Flood

Symptoms:

```text
outbound bytes spike
queue growth
```

Investigate:

```text
producer
fan-out
subscription explosion
duplicate publication
```

Mitigate:

```text
coalesce
shed ephemeral events
rate-limit producer
```

---

# 272. Incident Playbook — Broker Failure

Symptoms:

```text
cross-node delivery stops
```

Investigate:

```text
broker connectivity
consumer lag
authentication
partition
```

Decide:

```text
fail closed
degrade local
buffer
pause writes
```

based on durability requirements.

---

# 273. Incident Playbook — Event Loss

Symptoms:

```text
DB state differs from clients
```

Investigate:

```text
outbox
publication
consumer
replay
```

Use:

```text
event position
```

to locate the gap.

---

# 274. Incident Playbook — Tenant Leak

Immediately:

```text
contain
disable affected subscription path
identify affected tenants
audit event logs
rotate exposed credentials if needed
```

Then:

```text
patch
test
monitor
```

---

# 275. Incident Playbook — Slow Consumers

Symptoms:

```text
one process has huge outbound buffers
```

Investigate:

```text
specific connections
network quality
payload size
fan-out
```

Apply bounded policy.

---

# 276. Incident Playbook — Memory Leak

Investigate:

```text
connections
subscriptions
timers
listeners
queues
closures
```

Compare:

```text
heap after controlled connect/close cycles
```

---

# 277. Incident Playbook — Replay Failure

Symptoms:

```text
clients reconnect
but state differs
```

Investigate:

```text
stream position
retention
snapshot boundary
duplicate handling
```

---

# 278. Incident Playbook — Duplicate Command

Symptoms:

```text
one user action creates two records
```

Investigate:

```text
client retry
ack loss
reconnect
missing command idempotency
```

---

# 279. Principal Decision Framework

Evaluate every design with:

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
What happens during disconnect?
What happens during retry?
What happens during duplicate delivery?
What happens during failover?
What happens during overload?
What happens after restart?
```

---

# 280. Dependency Graph

```text
Chapter 31–38 — Async JavaScript
              ↓
Chapter 53 — Streams
              ↓
Chapter 55 — HTTP Networking
              ↓
Chapter 57 — Security
              ↓
Chapter 58–63 — Node Runtime
              ↓
Chapter 79–85 — API / DB / Observability / Reliability / Performance
              ↓
Chapter 86–89 — Testing / Debugging / Review
              ↓
Chapter 98–101 — Judgment
              ↓
Chapter 104 — Production HTTP Client
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
```

---

# 281. Concept Connections

## Depends On

```text
HTTP
async execution
Node.js
streams
security
databases
reliability
observability
performance
testing
```

## Builds Toward

```text
job queues
cache systems
event-driven architectures
production backends
large-scale platforms
```

## Revisited

```text
AbortSignal
events
streams
errors
authentication
authorization
transactions
idempotency
observability
backpressure
```

## Why This Chapter Matters Later

WebSockets expose the hardest practical problems of JavaScript backends:

```text
long-lived state
concurrency
partial failure
ordering
backpressure
recovery
fan-out
distributed coordination
```

Those same problems reappear in:

```text
queues
event-driven systems
microservices
distributed caches
stream processing
```

---

# 282. Spaced Retrieval Schedule

### Day 0

```text
WebSocket lifecycle
message envelopes
auth
subscriptions
```

### Day 1

```text
ordering
delivery guarantees
replay
backpressure
```

### Day 3

```text
presence
reconnect
idempotency
outbox
```

### Day 7

```text
scaling
broker
durable log
security
```

### Day 14

```text
load testing
incident response
shutdown
memory leaks
```

### Day 30

Rebuild the single-node system without reference.

### Day 60

Design multi-node recovery.

### Day 90

Defend a 1M-connection architecture.

---

# 283. Revision / Retrieval Record

```md
# Chapter 106 — Revision / Retrieval Record

## Review #
- Date:
- Duration:
- Status before:
- Status after:

## Retrieval
- WebSocket lifecycle [ ]
- handshake [ ]
- authentication [ ]
- authorization [ ]
- protocol envelope [ ]
- commands/events [ ]
- subscriptions [ ]
- tenant isolation [ ]
- heartbeat [ ]
- presence [ ]
- ordering [ ]
- sequence numbers [ ]
- duplicate handling [ ]
- replay [ ]
- snapshot recovery [ ]
- backpressure [ ]
- slow consumers [ ]
- reconnect [ ]
- jitter [ ]
- fan-out [ ]
- broker [ ]
- durable stream [ ]
- outbox [ ]
- graceful shutdown [ ]
- observability [ ]
- security [ ]
- testing [ ]
- load testing [ ]

## Build Evidence
- Repository:
- Commit:
- Server:
- Client:
- Broker:
- Database:
- Load-test result:
- Failure-injection result:

## Gaps
-

## New Insights
-

## Follow-up
-
```

---

# Chapter 106 — Canonical References and Source Discipline

Primary references:

1. WebSocket Protocol — RFC 6455  
   https://www.rfc-editor.org/rfc/rfc6455

2. WHATWG WebSockets API  
   https://websockets.spec.whatwg.org/

3. Node.js Documentation  
   https://nodejs.org/docs/

4. ECMAScript Language Specification  
   https://tc39.es/ecma262/

5. OWASP WebSocket Security  
   https://cheatsheetseries.owasp.org/cheatsheets/WebSocket_Security_Cheat_Sheet.html

6. OWASP API Security  
   https://owasp.org/www-project-api-security/

7. HTTP Semantics  
   https://httpwg.org/specs/

Source discipline:

```text
WebSocket protocol semantics
→ RFC 6455

Browser WebSocket API
→ WHATWG WebSockets

Node runtime behavior
→ Node documentation

language behavior
→ ECMAScript

security controls
→ OWASP + deployment threat model

broker semantics
→ chosen broker documentation

database semantics
→ chosen database documentation

performance
→ load tests + profiles + telemetry

reliability
→ failure injection + operational evidence
```

Do not generalize a framework-specific WebSocket behavior into a universal WebSocket guarantee.

Do not generalize a browser behavior into a Node runtime guarantee.

Do not generalize a broker's delivery semantics into a WebSocket protocol guarantee.

---

# 284. Completion Snapshot

```text
Part XX — Projects

Chapter 106 — Real-Time WebSocket Systems
[ ] Not Started

Track A — Core Theory
[ ] WebSocket protocol
[ ] handshake
[ ] lifecycle
[ ] authentication
[ ] authorization
[ ] message envelopes
[ ] commands/events
[ ] subscriptions
[ ] ordering
[ ] delivery guarantees
[ ] replay
[ ] snapshots
[ ] backpressure
[ ] slow consumers
[ ] heartbeat
[ ] reconnect
[ ] presence
[ ] fan-out
[ ] brokers
[ ] durable streams
[ ] outbox
[ ] graceful shutdown
[ ] observability
[ ] security
[ ] testing
[ ] scaling

Track B — Implementation
[ ] WebSocket server
[ ] connection manager
[ ] protocol validator
[ ] authentication
[ ] authorization
[ ] subscription manager
[ ] event router
[ ] heartbeat
[ ] bounded queue
[ ] slow consumer policy
[ ] sequence tracking
[ ] dedupe
[ ] replay
[ ] snapshot recovery
[ ] presence
[ ] reconnecting client
[ ] outbox
[ ] broker
[ ] metrics
[ ] logs
[ ] tracing
[ ] integration tests
[ ] load tests
[ ] shutdown

Track C — Interview / Reasoning
[ ] Explain WebSocket
[ ] Compare SSE
[ ] Compare polling
[ ] Explain event ordering
[ ] Explain at-least-once
[ ] Explain replay
[ ] Explain snapshot recovery
[ ] Explain backpressure
[ ] Explain slow-consumer policy
[ ] Explain reconnect storms
[ ] Explain broker architecture
[ ] Explain durable event streams
[ ] Explain outbox
[ ] Explain tenant isolation
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

# 285. Completion Criteria

Do not mark mastery because a socket can send a message.

You are ready to continue when you can independently:

1. Explain the WebSocket lifecycle.
2. Design a clear application protocol.
3. Authenticate connections.
4. Authorize commands and subscriptions.
5. Enforce tenant isolation.
6. Bound message and connection resources.
7. Implement heartbeats.
8. Detect dead connections.
9. Implement reconnect with exponential backoff and jitter.
10. Assign event IDs.
11. Define ordering guarantees.
12. Handle duplicates.
13. Detect sequence gaps.
14. Implement replay.
15. Design snapshot recovery.
16. Handle slow consumers.
17. Prevent unbounded queues.
18. Distinguish ephemeral from durable events.
19. Integrate DB transactions with publication.
20. Explain the outbox pattern.
21. Scale fan-out across multiple Node processes.
22. Choose broker vs durable stream intentionally.
23. Design presence semantics.
24. Implement safe graceful shutdown.
25. Observe connection health and delivery latency.
26. Diagnose memory leaks.
27. Test failure and recovery paths.
28. Load test long-lived connections.
29. Secure browser WebSocket endpoints.
30. Defend the system at principal level.

---

# Final Mental Model

```text
                       ┌───────────────┐
                       │  REST / HTTP  │
                       │ snapshot/data │
                       └───────┬───────┘
                               │
                               ▼
┌──────────┐    WebSocket    ┌───────────────┐
│  Client  │ ◀──────────────▶│ Node Gateway  │
└──────────┘                  └───────┬───────┘
                                      │
                         ┌────────────┼────────────┐
                         │            │            │
                         ▼            ▼            ▼
                      Authz        App/Domain    Broker
                         │            │            │
                         │            ▼            │
                         │            DB           │
                         │            │            │
                         │            ▼            │
                         │         Outbox ─────────┘
                         │
                         ▼
                    Subscriptions
                         │
                         ▼
                     Fan-out
                         │
          ┌──────────────┼──────────────┐
          ▼              ▼              ▼
       Client A       Client B       Client C
```

The deepest lesson is:

```text
WebSocket transport
≠
real-time correctness
```

Real-time correctness comes from:

```text
identity
authorization
explicit protocol
bounded resources
event identity
ordering
durability
replay
idempotency
backpressure
reconnection
observability
failure handling
```

A production real-time system must remain understandable and recoverable when:

```text
connections disappear
messages duplicate
events arrive late
servers restart
brokers fail
permissions change
clients reconnect simultaneously
consumers become slow
tenants become huge
```

> **Mastery reminder:** The hard part of WebSockets is not keeping a socket open. The hard part is preserving useful system semantics while the network, clients, processes, and dependencies continually fail in partial and surprising ways.