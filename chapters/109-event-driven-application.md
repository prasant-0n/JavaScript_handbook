# Chapter 109 — Production JavaScript Event-Driven Application

> **JavaScript Mastery — Part XX: Projects**
>
> **Project:** Combine the REST API, WebSocket system, job queue, and cache from Chapters 105–108 into a production-grade event-driven application with explicit domain events, commands, transactional boundaries, outbox/inbox processing, idempotent consumers, ordering, retries, replay, eventual consistency, sagas, observability, security, and operational recovery.
>
> **Role perspective:** Principal JavaScript Engineer · Node.js Architect · Event-Driven Systems Engineer · Distributed Systems Engineer · Reliability Engineer · Database Engineer · Security Engineer
>
> **Status:** `[ ] Not Started`
>
> **Verification date:** 2026-09-11
>
> **Core rule:** **An event-driven system is not “services sending messages.” It is a collection of independently executing components coordinated through explicit contracts, durable state, failure-aware delivery semantics, and carefully defined ownership of business truth.**

---

# 1. Project Mission

Build:

```text
FocusBoard Event Platform
```

Integrate:

```text
REST API
WebSocket gateway
job queue
cache
database
event bus
background workers
notifications
audit
analytics
```

The application should support:

```text
task creation
task updates
comments
notifications
project summaries
analytics
exports
webhook delivery
real-time updates
```

A single business action should be able to propagate through the system:

```text
HTTP command
→ DB transaction
→ domain event/outbox
→ event publisher
→ consumers
→ cache invalidation
→ WebSocket notification
→ background job
→ analytics
```

The architecture must remain correct when:

```text
consumers crash
events duplicate
events arrive late
events are replayed
workers restart
cache is unavailable
a service is temporarily down
deployment versions overlap
one tenant becomes dominant
events are published slowly
```

---

# 2. Learning Objectives

By completing this project, you should be able to:

- Explain event-driven architecture.
- Distinguish commands from events.
- Distinguish domain events from integration events.
- Design event contracts.
- Identify event ownership.
- Design aggregate boundaries.
- Implement transactional outbox.
- Implement inbox/deduplication.
- Build event consumers.
- Make consumers idempotent.
- Handle retries.
- Design dead-letter handling.
- Design ordering guarantees.
- Design partition keys.
- Understand eventual consistency.
- Model read models/projections.
- Rebuild projections from events where appropriate.
- Design cache invalidation through events.
- Publish WebSocket notifications.
- Trigger asynchronous jobs.
- Coordinate multi-step workflows.
- Recognize when a saga is appropriate.
- Understand compensating actions.
- Design event versioning.
- Handle schema evolution.
- Handle duplicate events.
- Handle missing events.
- Handle replay.
- Handle consumer lag.
- Apply backpressure.
- Isolate tenants.
- Observe event pipelines.
- Diagnose distributed failures.
- Secure internal event boundaries.
- Load test an event-driven application.
- Explain operational trade-offs at principal level.

---

# 3. Prerequisites

Recommended:

```text
Chapter 29 — Errors
Chapter 31–38 — Async JavaScript
Chapter 45–48 — Memory / Engine
Chapter 55 — Fetch / HTTP Networking
Chapter 57 — Security Engineering
Chapter 58–63 — Node Runtime
Chapter 79 — API Design
Chapter 81 — Database Integration
Chapter 82 — API Architecture
Chapter 83 — Observability
Chapter 84 — Reliability
Chapter 85 — Performance
Chapter 86–89 — Testing / Debugging / Review
Chapter 98–101 — Judgment
Chapter 105 — Node REST API
Chapter 106 — Real-Time WebSocket
Chapter 107 — Job Queue
Chapter 108 — Cache System
```

---

# 4. Why Event-Driven Architecture?

A direct architecture often looks like:

```text
API
→ DB
→ email
→ analytics
→ notifications
→ cache
```

The request can become coupled to:

```text
every downstream operation
```

Event-driven architecture allows:

```text
API
→ authoritative state
→ event
```

and:

```text
consumers react independently
```

---

# 5. Event-Driven Mental Model

Think:

```text
Command
   ↓
Decision
   ↓
State Change
   ↓
Event
   ↓
Consumers
```

The event communicates:

```text
something happened
```

not:

```text
please do this
```

---

# 6. Command vs Event

Command:

```text
CreateTask
```

Event:

```text
TaskCreated
```

Command:

```text
intent
```

Event:

```text
fact
```

---

# 7. Command Ownership

Commands usually have:

```text
one intended handler/owner
```

Events can have:

```text
many independent consumers
```

---

# 8. Event Ownership

An event should have a clear source of truth.

Example:

```text
Project service owns ProjectCreated
```

Analytics should not redefine that event as authoritative project state.

---

# 9. Domain Event

A domain event expresses a meaningful business fact:

```text
TaskCompleted
```

---

# 10. Integration Event

An integration event is a boundary contract intended for other modules/services.

It may be:

```text
derived
redacted
versioned
```

from an internal domain event.

---

# 11. Do Not Expose Internal Domain State Automatically

Internal domain model:

```text
private fields
derived state
internal IDs
```

should not automatically become public event schema.

---

# 12. Event Envelope

Example:

```json
{
  "id": "evt_123",
  "type": "task.completed",
  "version": 1,
  "occurredAt": "2026-09-11T00:00:00Z",
  "producer": "task-service",
  "tenantId": "tenant_42",
  "aggregate": {
    "type": "task",
    "id": "task_9",
    "version": 8
  },
  "correlationId": "req_100",
  "causationId": "cmd_22",
  "data": {
    "taskId": "task_9"
  }
}
```

---

# 13. Event ID

Unique identifier for the event instance:

```text
evt_123
```

Used for:

```text
deduplication
traceability
audit
```

---

# 14. Correlation ID

Connects:

```text
one business flow
```

such as:

```text
HTTP request
→ command
→ event
→ jobs
```

---

# 15. Causation ID

Identifies the immediate event/command that caused the current message.

Useful for:

```text
causal tracing
```

---

# 16. Correlation vs Causation

Correlation:

```text
same broader workflow
```

Causation:

```text
direct predecessor
```

Do not treat them as interchangeable.

---

# 17. Event Timestamp

Possible fields:

```text
occurredAt
publishedAt
processedAt
```

Each answers a different question.

---

# 18. Do Not Use Wall-Clock Time as Ordering

Two events can have:

```text
same timestamp
```

or:

```text
clock skew
```

Use explicit sequence/partition semantics when ordering matters.

---

# 19. Event Sequence

For ordered streams:

```text
sequence = 1001
```

can represent a stream position.

---

# 20. Aggregate Version

Example:

```text
task version = 8
```

Useful for:

```text
optimistic concurrency
event validation
reconciliation
```

---

# 21. Event Store vs Event Bus

Event bus:

```text
transport/distribution
```

Event store:

```text
durable event history
```

They are not automatically the same system.

---

# 22. Event Bus

Use for:

```text
fan-out
decoupling
asynchronous communication
```

---

# 23. Event Store

Use when business requirements need:

```text
event history
replay
temporal reconstruction
```

Not every event-driven system needs event sourcing.

---

# 24. Event Sourcing

In event sourcing:

```text
events are the primary persisted state history
```

rather than merely:

```text
notifications about DB changes
```

---

# 25. Event Notification Architecture

Common pattern:

```text
DB is source of truth
+
events describe changes
```

This is often simpler than event sourcing.

---

# 26. Event Sourcing Trade-Off

Benefits:

```text
history
replay
audit
temporal models
```

Costs:

```text
complexity
schema evolution
projection management
debugging
storage
```

Do not adopt it by default.

---

# 27. Transactional Outbox

Problem:

```text
DB commit
→ publish event
```

can fail between steps.

---

# 28. Outbox Solution

Inside one DB transaction:

```text
business state
+
outbox event
```

Then:

```text
outbox publisher
→ broker
```

---

# 29. Outbox Table

Example:

```text
outbox
------
id
type
version
aggregate_type
aggregate_id
tenant_id
payload
created_at
published_at
attempts
```

---

# 30. Outbox Ordering

For aggregate ordering:

```text
task 9
event 1
event 2
event 3
```

the publisher should preserve required order.

Global ordering may be unnecessary and expensive.

---

# 31. Outbox Polling

Simple model:

```text
poll outbox
→ claim rows
→ publish
→ mark published
```

---

# 32. Outbox Claiming

Use the same principles as Chapter 107:

```text
atomic ownership
lease
stale worker recovery
```

---

# 33. Outbox Duplicate Publication

Publisher can crash:

```text
publish
→ crash
```

before:

```text
mark published
```

Then event may publish again.

Consumers must tolerate duplicates.

---

# 34. Inbox Pattern

Consumer stores:

```text
event ID
```

before/while applying the event.

If same event appears again:

```text
skip duplicate
```

---

# 35. Inbox Table

Example:

```text
inbox
-----
consumer
event_id
processed_at
```

Use a uniqueness constraint on:

```text
consumer + event_id
```

where appropriate.

---

# 36. Inbox Transaction

For database-backed consumers:

```text
insert event ID
+
apply state change
```

inside one transaction when both are in the same database.

---

# 37. Inbox Race

Two workers process same event simultaneously.

Use:

```text
unique constraint
atomic insert
transaction
```

rather than:

```text
SELECT then INSERT
```

---

# 38. Idempotent Consumer

A consumer should safely handle:

```text
event
event again
```

without creating incorrect duplicate effects.

---

# 39. Idempotency Techniques

Possible:

```text
inbox table
unique business keys
upserts
version checks
compare-and-swap
```

---

# 40. Consumer Idempotency Is Contextual

A consumer can be idempotent at:

```text
storage
business effect
```

levels.

Do not assume one database upsert solves all external side effects.

---

# 41. External Side Effects

Example:

```text
TaskCompleted
→ email provider
```

The provider call can duplicate.

Need:

```text
provider idempotency
dedupe
reconciliation
```

where appropriate.

---

# 42. Event Delivery Semantics

Possible:

```text
at-most-once
at-least-once
effectively-once
```

Document actual guarantees.

---

# 43. At-Most-Once

Possible loss.

Useful for:

```text
ephemeral notifications
```

sometimes.

---

# 44. At-Least-Once

Possible duplicates.

Useful for:

```text
durable business events
```

with idempotent consumers.

---

# 45. Exactly-Once

Do not promise system-wide exactly-once execution casually.

Distributed boundaries create ambiguous failure windows.

---

# 46. Eventual Consistency

After:

```text
TaskCompleted
```

a dashboard may update:

```text
seconds later
```

This is eventual consistency.

---

# 47. User-Visible Eventual Consistency

Design UX:

```text
saving
processing
updated
```

instead of pretending all views change atomically.

---

# 48. Read Models

A consumer may build:

```text
task summary
```

from events.

This is a projection/read model.

---

# 49. Projection

Concept:

```text
event
→ update read model
```

---

# 50. Projection Lag

A read model can lag behind source state.

Monitor:

```text
event position
vs
projection position
```

---

# 51. Projection Rebuild

If projection data is derived:

```text
drop projection
→ replay events
→ rebuild
```

---

# 52. Rebuild Requirements

Events must contain enough information, or the projection must be able to retrieve required authoritative state.

---

# 53. Event Snapshot

For large histories:

```text
snapshot
+
events after snapshot
```

can reduce rebuild time.

---

# 54. Projection Version

Version projections:

```text
task-summary-v2
```

so new schemas can be built safely.

---

# 55. Blue/Green Projection

Build:

```text
v2
```

while:

```text
v1
```

remains live.

Then switch reads.

---

# 56. Projection Consistency

Do not call a projection:

```text
strongly consistent
```

unless the architecture actually provides that guarantee.

---

# 57. Event Ordering

Ordering can be:

```text
global
tenant
aggregate
partition
consumer
```

Choose the smallest guarantee that solves the business requirement.

---

# 58. Aggregate Ordering

For:

```text
task:123
```

events should usually preserve:

```text
version 1
version 2
version 3
```

---

# 59. Partition Key

Use:

```text
aggregate ID
```

when per-aggregate ordering matters.

---

# 60. Ordering and Parallelism

Strict global ordering reduces:

```text
parallelism
```

Partitioning provides:

```text
parallelism across independent keys
```

---

# 61. Consumer Groups

A consumer group can divide events among workers.

Concept:

```text
partition
→ one active consumer in group
```

depending on broker semantics.

---

# 62. Fan-Out Consumer Groups

Different logical consumers can each receive all relevant events:

```text
notifications
analytics
cache invalidation
search indexing
```

---

# 63. Independent Failure

Analytics being down should not necessarily block:

```text
task update
```

unless business correctness explicitly requires it.

---

# 64. Event Dependency

Consumer B may depend on:

```text
event A
```

Define:

```text
ordering
dependency
retry
rebuild
```

explicitly.

---

# 65. Event Chains

Example:

```text
TaskCreated
→ NotificationRequested
→ EmailSendRequested
→ EmailSent
```

This can create long chains.

Keep workflows understandable.

---

# 66. Event Loop of Doom

Bad design:

```text
A event
→ B
→ C
→ A
```

without intentional termination/idempotency.

Detect cycles.

---

# 67. Event Storms

One business action can create:

```text
1 event
→ 100 events
→ 10,000 events
```

Measure event amplification.

---

# 68. Event Amplification

Approximate:

```text
events_out
/
events_in
```

by workflow.

High amplification may indicate:

```text
poor boundaries
chatty architecture
```

---

# 69. Event Granularity

Fine-grained:

```text
TaskTitleChanged
TaskStatusChanged
```

Coarse-grained:

```text
TaskUpdated
```

---

# 70. Fine-Grained Event Benefits

Better:

```text
consumer specificity
payload efficiency
```

Costs:

```text
more event types
```

---

# 71. Coarse-Grained Event Benefits

Simpler:

```text
schema surface
```

Costs:

```text
larger payload
harder consumer filtering
```

---

# 72. Event Design Principle

Publish business facts that consumers can reason about.

Avoid events that expose accidental implementation details.

---

# 73. Event Name

Prefer:

```text
task.completed
```

over:

```text
task.service.row.updated
```

---

# 74. Event Versioning

Possible:

```text
version field
```

or:

```text
event type v2
```

Choose based on schema evolution strategy.

---

# 75. Additive Changes

Usually safer:

```text
add optional field
```

than:

```text
rename/remove field
```

---

# 76. Breaking Changes

A consumer may still run old code.

Therefore producer must consider:

```text
mixed deployment
```

---

# 77. Schema Registry

For large systems, centralized schema management can provide:

```text
compatibility rules
validation
discoverability
```

But adds operational complexity.

---

# 78. Contract Testing

Verify:

```text
producer schema
consumer expectations
```

before deployment.

---

# 79. Consumer-Driven Contracts

Consumers can define expectations.

Useful when:

```text
many independent consumers
```

exist.

---

# 80. Event Metadata Stability

Keep:

```text
id
type
version
timestamp
correlation
causation
```

semantically stable.

---

# 81. Event Payload Size

Do not publish:

```text
entire database row graph
```

unless genuinely required.

---

# 82. Event References

Instead of huge payload:

```json
{
  "taskId": "task_123"
}
```

consumer can fetch state.

But this trades:

```text
smaller event
```

for:

```text
extra read
consistency question
```

---

# 83. Self-Contained Events

Include enough data to process without another network call.

Benefits:

```text
lower coupling
easier replay
```

Costs:

```text
larger payload
possible stale snapshot
```

---

# 84. Snapshot Event Payload

A useful pattern:

```text
resource ID
+
relevant version
+
changed fields
```

---

# 85. Event Redaction

Do not publish:

```text
passwordHash
tokens
private secrets
```

---

# 86. Tenant Context

Every event carrying tenant data must preserve:

```text
tenant identity
```

and consumers must enforce tenant boundaries.

---

# 87. Cross-Tenant Event Leak

A global consumer may accidentally write:

```text
tenant A
data
```

into:

```text
tenant B
projection
```

Use tenant-aware keys and queries.

---

# 88. Cache Invalidation Consumer

Example:

```text
task.updated
→ invalidate task cache
→ invalidate project summary
```

---

# 89. Cache Invalidation Delay

Because invalidation is asynchronous:

```text
cache can remain stale
```

for some interval.

Define:

```text
freshness budget
```

---

# 90. WebSocket Consumer

Example:

```text
task.updated
→ publish to project subscribers
```

---

# 91. WebSocket + Event Bus

```text
DB
→ outbox
→ broker
→ WebSocket gateway
→ clients
```

The WebSocket gateway should not become the authoritative state store.

---

# 92. Job Consumer

Example:

```text
task.completed
→ report.recalculate
```

---

# 93. Queue vs Event Bus

Queue semantics:

```text
one logical worker should process
```

Event bus semantics:

```text
multiple consumer groups can react
```

Some infrastructure supports both models.

---

# 94. Event-to-Job Bridge

Consumer can turn event into:

```text
job
```

with idempotent enqueueing.

---

# 95. Duplicate Event to Job

If same event arrives twice:

```text
do not create duplicate job
```

when business semantics require one logical job.

Use:

```text
event ID as dedupe key
```

or a business operation key.

---

# 96. Event Processing Pipeline

```text
receive
→ validate
→ authorize/trust boundary
→ dedupe
→ process
→ commit
→ acknowledge
```

---

# 97. Acknowledgement Semantics

Ack should happen only when the consumer's durable success condition is satisfied.

---

# 98. Ack Too Early

```text
ack
→ process
→ crash
```

may lose work.

---

# 99. Ack Too Late

```text
process
→ long delay
→ broker redelivery
```

may cause duplicates.

Use idempotency and sensible processing/visibility deadlines.

---

# 100. Poison Event

Malformed event:

```text
unknown schema
```

should not crash the entire consumer fleet.

Classify and dead-letter as appropriate.

---

# 101. Dead-Letter Event

Store:

```text
event ID
type
consumer
attempts
error
receivedAt
```

and enough safe diagnostic metadata.

---

# 102. Dead-Letter Replay

Replay only after:

```text
root cause fixed
```

and consider:

```text
side effects already applied
```

---

# 103. Event Retry Policy

Classify:

```text
temporary
permanent
unknown
```

same as jobs.

---

# 104. Retry Backoff

Use:

```text
exponential
+
jitter
```

where appropriate.

---

# 105. Retry Storm

If a dependency is down:

```text
all consumers retry
```

can create:

```text
more dependency load
```

Use:

```text
rate limits
circuit breakers
pause
```

---

# 106. Consumer Lag

Measure:

```text
event produced
→ event processed
```

---

# 107. Lag Monitoring

Track:

```text
current position
consumer position
```

and:

```text
age of oldest unprocessed event
```

---

# 108. Lag SLO

Example:

```text
99% of task events processed within 5 seconds
```

The exact target is product-specific.

---

# 109. Backpressure

If consumer processing is slower than incoming events:

```text
lag grows
```

Options:

```text
increase consumers
batch
optimize handler
reduce event volume
shed optional work
```

---

# 110. Concurrency

More consumer concurrency can improve throughput but can break:

```text
ordering
DB capacity
downstream limits
```

---

# 111. Batch Processing

Batch events can improve:

```text
DB efficiency
network efficiency
```

but increase:

```text
latency
failure granularity
```

---

# 112. Batch Failure

If 100 events are processed in one batch and one fails:

```text
retry all
```

or:

```text
partial success
```

must be defined.

---

# 113. Transactional Consumer

Example:

```text
event
+
projection update
+
inbox record
```

inside one DB transaction.

---

# 114. External Consumer

When side effect is external:

```text
inbox
→ external call
→ completion
```

cannot always be one atomic transaction.

Need:

```text
idempotency
reconciliation
```

---

# 115. Saga

A saga coordinates a distributed workflow where one global transaction is not available.

---

# 116. Choreography

Services react to events:

```text
A emits
→ B reacts
→ B emits
→ C reacts
```

---

# 117. Orchestration

A coordinator explicitly commands steps:

```text
orchestrator
→ A
→ B
→ C
```

---

# 118. Choreography Benefits

```text
loose coupling
autonomous consumers
```

Costs:

```text
harder workflow visibility
event cycles
distributed reasoning
```

---

# 119. Orchestration Benefits

```text
central workflow visibility
explicit state machine
```

Costs:

```text
coordinator complexity
central ownership
```

---

# 120. Saga Compensation

Suppose:

```text
step A succeeded
step B failed
```

A compensation may:

```text
undo/offset A
```

---

# 121. Compensation Is Not Rollback

Distributed compensation often means:

```text
new business action
```

not:

```text
time travel
```

---

# 122. Example Saga

```text
Create export
→ reserve storage
→ generate file
→ publish link
```

If generation fails:

```text
release storage
```

---

# 123. Saga State

Persist:

```text
workflow ID
step
status
attempt
compensation status
```

---

# 124. Saga Idempotency

Every step and compensation should be safe against duplicates where practical.

---

# 125. Saga Timeout

Workflows can become stuck.

Use:

```text
deadline
timeout
repair action
```

---

# 126. Saga Monitoring

Track:

```text
active workflows
stuck workflows
failed compensations
```

---

# 127. Event-Driven Authentication

Authentication may remain centralized.

Do not distribute credential validation unnecessarily.

---

# 128. Authorization Events

Permission change:

```text
user.role.changed
```

can trigger:

```text
cache invalidation
WebSocket unsubscribe
projection update
```

---

# 129. Security Event

Security-sensitive events should be:

```text
authenticated
authorized
audited
```

---

# 130. Event Trust

Do not assume:

```text
"internal event"
```

means:

```text
safe
```

Protect broker access and validate producer identity.

---

# 131. Event Injection

An attacker who can publish:

```text
user.role.changed
```

may escalate privileges.

Producer authorization matters.

---

# 132. Replay Authorization

A replay operation can repeat:

```text
external side effects
```

Protect operational replay interfaces.

---

# 133. Tenant-Aware Replay

Replay only:

```text
authorized scope
```

and ensure the operator cannot accidentally mix tenants.

---

# 134. Sensitive Events

Minimize event payloads containing:

```text
PII
financial data
credentials
security details
```

---

# 135. Event Encryption

At-rest/in-transit encryption should follow infrastructure requirements.

Application-level encryption may be required for especially sensitive fields.

---

# 136. Event Retention

Define:

```text
business requirement
audit
replay
privacy
cost
```

before retaining events forever.

---

# 137. Right to Deletion

If personal data exists in durable event histories, design for:

```text
redaction
tokenization
indirection
retention boundaries
```

according to applicable requirements.

---

# 138. Immutable Event Problem

Events are conceptually historical facts.

Sensitive personal data can create tension with:

```text
immutability
```

and:

```text
deletion/privacy
```

Design this deliberately.

---

# 139. Event Schema Security

Schema validation prevents:

```text
unexpected object shapes
oversized payloads
invalid enum values
```

---

# 140. Event Size Limits

Set:

```text
maximum event size
```

to prevent broker/consumer resource exhaustion.

---

# 141. Fan-Out Amplification

One event to:

```text
1,000,000 subscribers
```

is a capacity problem.

Design:

```text
partitioning
aggregation
sampling
dedicated fan-out infrastructure
```

where necessary.

---

# 142. Hot Event

Example:

```text
global.config.updated
```

may trigger every consumer simultaneously.

Use:

```text
versioning
coalescing
lazy refresh
```

where possible.

---

# 143. Cache Invalidation Event Storm

One large mutation can invalidate:

```text
100,000 keys
```

Prefer:

```text
version bump
namespace invalidation
tag-based invalidation
```

where available and appropriate.

---

# 144. Event Storm Control

Possible:

```text
coalescing
batch events
debounce
rate limit
```

Only for events where dropping intermediate states is safe.

---

# 145. Audit Events

Separate:

```text
business events
```

from:

```text
security/audit events
```

when retention and immutability requirements differ.

---

# 146. Audit Consumer

Audit system should receive:

```text
user
action
resource
timestamp
request/correlation ID
```

without exposing secrets.

---

# 147. Analytics Consumer

Analytics can intentionally be:

```text
eventually consistent
```

and independently scaled.

---

# 148. Search Consumer

Search index updates are also typically:

```text
eventually consistent
```

---

# 149. Search Rebuild

A search index can be rebuilt from durable source/events where architecture permits.

---

# 150. Notification Consumer

A notification service may react to:

```text
task.completed
comment.created
```

without blocking the task transaction.

---

# 151. Notification Preferences

The consumer should check current:

```text
user preferences
```

rather than trusting stale assumptions embedded in old events where product semantics require current preferences.

---

# 152. Historical vs Current State

An event may represent:

```text
what was true when it happened
```

The consumer may need:

```text
what is true now
```

Do not confuse the two.

---

# 153. Event Payload and Current State

A notification event can include:

```text
event-time state
```

while preference evaluation uses:

```text
current state
```

This distinction matters.

---

# 154. Eventual Consistency UX

For UI:

```text
command accepted
→ task updated
→ live event
→ projection catches up
```

The UI should tolerate temporary divergence.

---

# 155. Read-After-Write

After a REST mutation:

```text
GET dashboard
```

may be stale if dashboard is event-driven.

Options:

```text
strong source read
projection read with version
client optimistic update
```

---

# 156. Version-Based Reconciliation

API response:

```json
{
  "version": 10
}
```

Client reads projection:

```text
version 9
```

It knows projection is behind.

---

# 157. Projection Readiness

For critical workflows, clients may send:

```text
requiredVersion
```

and server can wait/fallback until projection reaches it.

---

# 158. Event-Driven Cache

A cache can use:

```text
event
→ invalidate
```

rather than direct application coupling.

---

# 159. Event-Driven Search

```text
task.updated
→ search indexer
```

The API does not need synchronous search-index writes.

---

# 160. Event-Driven Analytics

```text
task.completed
→ analytics
```

can scale independently.

---

# 161. Event-Driven Jobs

```text
report.requested
→ job queue
```

---

# 162. Event-Driven WebSockets

```text
task.updated
→ WebSocket gateway
```

---

# 163. Whole-System Flow

```text
                         ┌─────────────┐
                         │ REST Client │
                         └──────┬──────┘
                                │
                                ▼
                         ┌─────────────┐
                         │ API Service │
                         └──────┬──────┘
                                │
                         DB Transaction
                           ┌────┴────┐
                           │         │
                           ▼         ▼
                         State     Outbox
                                     │
                                     ▼
                              ┌─────────────┐
                              │ Event Bus   │
                              └──────┬──────┘
                       ┌─────────────┼──────────────┐
                       ▼             ▼              ▼
                    Cache        WebSocket        Jobs
                  Invalidation     Gateway       Queue
                       │             │              │
                       ▼             ▼              ▼
                    Cache         Clients         Workers
                       │                            │
                       └────────────┬───────────────┘
                                    ▼
                              Read Models
                              / Analytics
```

---

# 164. Source of Truth

For FocusBoard:

```text
database
```

is authoritative for durable task state.

Other systems are:

```text
derived
```

unless explicitly designated otherwise.

---

# 165. Event Bus Does Not Own Business Truth

The event bus distributes facts.

It should not automatically become the only source of business state.

---

# 166. Cache Does Not Own Business Truth

Cache contains:

```text
derived representation
```

---

# 167. Search Index Does Not Own Business Truth

Search can be rebuilt.

---

# 168. Analytics Does Not Own Operational State

Analytics answers:

```text
what happened
```

not:

```text
what is authorized now
```

---

# 169. Event Replay Semantics

When replaying:

```text
events describe history
```

but current external dependencies may now behave differently.

Do not assume:

```text
replay = original world
```

---

# 170. Replay Modes

Possible:

```text
rebuild projection
re-run business logic
audit reconstruction
simulation
```

These are different operations.

---

# 171. Projection Replay Safety

A projection rebuild should not:

```text
send emails
charge payment
```

unless explicitly designed to do so.

---

# 172. Side-Effect Isolation

Separate:

```text
pure projection consumers
```

from:

```text
side-effecting consumers
```

when replay is required.

---

# 173. Event Reprocessing

A consumer can receive:

```text
old event
```

after:

```text
schema changes
```

Maintain compatibility.

---

# 174. Reprocessing Window

Keep:

```text
consumer version
event version
```

information sufficient to debug historical processing.

---

# 175. Event Metadata for Debugging

Useful:

```text
eventId
correlationId
causationId
producer
partition
offset/sequence
consumer
attempt
```

---

# 176. Distributed Tracing

Trace:

```text
HTTP request
→ DB commit
→ outbox
→ broker
→ consumer
→ downstream
```

---

# 177. Trace Context

Propagate tracing context across event metadata where infrastructure supports it.

---

# 178. Event Metrics

Track:

```text
events_published_total
events_consumed_total
events_failed_total
events_retried_total
event_processing_duration
consumer_lag
dead_letter_total
```

---

# 179. Event Cardinality

Avoid high-cardinality metric labels like:

```text
eventId
tenantId
requestId
```

unless your monitoring architecture intentionally supports them.

---

# 180. Consumer Health

A consumer can be:

```text
process alive
```

but:

```text
lagging badly
```

Track both.

---

# 181. Consumer Readiness

A consumer may need:

```text
broker connectivity
DB connectivity
schema compatibility
```

before processing.

---

# 182. Consumer Shutdown

```text
stop claiming
→ finish safe work
→ commit
→ acknowledge
→ close
```

---

# 183. Shutdown Deadline

Never allow:

```text
indefinite drain
```

---

# 184. Consumer Crash

Expected recovery:

```text
message remains/unacknowledged
→ redelivery
→ idempotent handling
```

according to broker semantics.

---

# 185. Broker Failure

Define:

```text
producer failure
consumer failure
publish buffering
fallback
```

Do not invent data durability at the application layer without evidence.

---

# 186. Event Loss

Detect missing events through:

```text
sequence
reconciliation
expected counts
source-of-truth scans
```

where needed.

---

# 187. Event Reconciliation

Periodically compare:

```text
source DB
vs
projection
```

for correctness-critical systems.

---

# 188. Repair

A repair job can:

```text
find divergence
→ rebuild affected projection
```

---

# 189. Reconciliation Is Not an Excuse for Bad Delivery

If events are routinely missing:

```text
fix pipeline
```

rather than relying entirely on periodic repair.

---

# 190. Event Ordering During Repair

Repair must avoid:

```text
old snapshot
overwriting newer event state
```

Use:

```text
version checks
```

where needed.

---

# 191. Schema Evolution

Producer v2:

```text
new field
```

Consumers may still be v1.

Use:

```text
backward-compatible evolution
```

or coordinated version migration.

---

# 192. Consumer Upgrade Strategy

Possible:

```text
deploy compatible consumer
→ deploy producer
→ verify
→ remove old consumer behavior
```

---

# 193. Event Contract Documentation

For each event document:

```text
purpose
producer
schema
version
ordering
delivery
retention
security
consumers
failure behavior
```

---

# 194. Event Catalog

Maintain a discoverable list:

```text
TaskCreated
TaskUpdated
TaskCompleted
CommentCreated
ProjectArchived
```

---

# 195. Event Ownership Catalog

Include:

```text
owner team/service
```

so breaking changes have an explicit owner.

---

# 196. Event Deprecation

Do not remove event types without:

```text
consumer inventory
usage measurement
migration
```

---

# 197. Consumer Contract

For each consumer:

```text
required fields
optional fields
assumptions
side effects
```

---

# 198. Event Security Boundaries

Potential zones:

```text
trusted internal
semi-trusted tenant
external integration
```

Apply different validation.

---

# 199. External Events

External webhooks/messages should enter through:

```text
validation
authentication/signature
normalization
```

before internal publication.

---

# 200. External-to-Internal Bridge

```text
external webhook
→ verify
→ normalize
→ internal event
```

Do not directly expose internal event structures.

---

# 201. Webhook Dedupe

External providers may retry.

Use:

```text
provider event ID
```

as dedupe key.

---

# 202. Event Mapping

External:

```text
PAYMENT_SUCCEEDED
```

Internal:

```text
payment.succeeded
```

The internal schema should not inherit every provider detail.

---

# 203. Event-Driven API Boundary

REST remains good for:

```text
query
command
resource retrieval
```

Events are good for:

```text
asynchronous facts
fan-out
```

---

# 204. Event-Driven Does Not Mean No REST

A mature application often combines:

```text
REST
WebSocket
jobs
events
cache
```

---

# 205. Event-Driven Does Not Mean Microservices

A modular monolith can use:

```text
internal event bus
```

before splitting processes.

---

# 206. Modular Monolith

Useful learning architecture:

```text
one process
multiple modules
explicit event contracts
```

This can expose event-driven design without network complexity.

---

# 207. Split Later

Once boundaries are proven:

```text
module
→ service
```

may be considered.

Do not distribute code merely to create microservices.

---

# 208. In-Process Event Bus

Good for:

```text
learning
local decoupling
same-process modules
```

But it does not provide:

```text
durability
cross-process recovery
distributed delivery
```

unless backed by durable infrastructure.

---

# 209. Distributed Event Bus

Adds:

```text
network
partitions
delivery semantics
broker operations
```

---

# 210. Eventual Consistency Boundaries

Identify:

```text
strong consistency
eventual consistency
best effort
```

per data flow.

---

# 211. Consistency Matrix

Example:

| Component | Consistency |
|---|---|
| Task DB | authoritative |
| Task cache | bounded stale |
| WebSocket | eventual |
| Analytics | eventual |
| Search | eventual |
| Audit | durable event history |

This is illustrative.

---

# 212. Failure Matrix

For each component:

```text
dependency down
message duplicate
message loss
message delay
process crash
network partition
```

document behavior.

---

# 213. Event-Driven Cost

Costs:

```text
broker
storage
serialization
network
consumer fleets
observability
schema governance
replay
operations
```

Benefits:

```text
decoupling
fan-out
independent scaling
asynchronous latency
```

---

# 214. When Not to Use Events

Prefer direct calls when:

```text
operation needs immediate result
transaction requires synchronous guarantee
workflow is trivial
event history provides little value
```

---

# 215. Event-Driven Anti-Pattern

```text
simple CRUD
→ 10 events
→ 8 consumers
```

This can be architecture theater.

---

# 216. Event-Driven Anti-Pattern

Publishing events for:

```text
implementation details
```

instead of:

```text
business facts
```

creates coupling.

---

# 217. Event-Driven Anti-Pattern

Consumer A waits for:

```text
B event
```

which waits for:

```text
C event
```

which waits for:

```text
A event
```

This creates hidden orchestration.

---

# 218. Event-Driven Anti-Pattern

Treating the broker as:

```text
global database
```

creates unclear ownership.

---

# 219. Event-Driven Anti-Pattern

Ignoring duplicates because:

```text
"Kafka/Redis will only send once"
```

is unsafe unless actual semantics prove the required guarantee and side effects are controlled.

---

# 220. Event-Driven Anti-Pattern

Infinite retry:

```text
failure
→ retry
→ retry
→ retry
```

without dead-lettering can consume the entire system.

---

# 221. Event-Driven Anti-Pattern

No event schema versioning.

Old consumers eventually break.

---

# 222. Event-Driven Anti-Pattern

Using current DB state during replay as if it were historical state.

Replay semantics must be explicit.

---

# 223. Event-Driven Anti-Pattern

Publishing sensitive information because:

```text
"the bus is private"
```

Internal systems still have compromise scenarios.

---

# 224. Production Project Flow

Implement:

```text
POST /projects/:id/tasks
        ↓
TaskCreated
        ↓
┌──────────────┬───────────────┬──────────────┐
▼              ▼               ▼              ▼
Cache          WebSocket       Analytics      Job
invalidate     notify          update         queue
```

---

# 225. Task Updated Flow

```text
PATCH /tasks/:id
        ↓
validate
        ↓
authorize
        ↓
DB transaction
 ├── task update
 └── outbox event
        ↓
publish
        ↓
task.updated
```

Consumers:

```text
cache invalidation
WebSocket
search
analytics
audit
```

---

# 226. Project Archive Flow

```text
project.archive
→ DB state change
→ project.archived
→ stop new subscriptions
→ invalidate project caches
→ schedule cleanup
```

---

# 227. Export Flow

```text
POST /exports
→ DB export request
→ export.requested
→ job queue
→ worker
→ object storage
→ export.completed
→ WebSocket
```

---

# 228. Notification Flow

```text
task.completed
→ notification consumer
→ preference lookup
→ notification job
→ email provider
→ notification.sent
```

---

# 229. Analytics Flow

```text
task.completed
→ analytics consumer
→ aggregation
→ dashboard projection
```

---

# 230. Search Flow

```text
task.updated
→ search consumer
→ index update
```

---

# 231. Cache Flow

```text
task.updated
→ invalidate item
→ invalidate summary
```

---

# 232. WebSocket Flow

```text
task.updated
→ authorized topic
→ enqueue bounded outbound event
→ client
```

---

# 233. Job Flow

```text
export.requested
→ durable job
→ claim
→ process
→ result
```

---

# 234. Cross-System Correlation

One task update should be traceable:

```text
requestId
→ eventId
→ jobId
→ downstream call
```

---

# 235. Event Testing Pyramid

```text
schema tests
↓
consumer unit tests
↓
integration tests
↓
contract tests
↓
replay tests
↓
failure injection
↓
load tests
```

---

# 236. Event Contract Tests

Verify:

```text
required metadata
schema
version
field meaning
```

---

# 237. Consumer Unit Test

Given:

```text
task.updated
```

expect:

```text
cache invalidated
```

without requiring a broker.

---

# 238. Integration Test

Use:

```text
DB
outbox
broker
consumer
projection
```

to prove end-to-end behavior.

---

# 239. Replay Test

Create:

```text
event history
```

then:

```text
rebuild projection
```

Verify output.

---

# 240. Duplicate Test

Send:

```text
same event twice
```

Verify business result remains correct.

---

# 241. Out-of-Order Test

Send:

```text
v3
v1
v2
```

Verify:

```text
ordering policy
```

---

# 242. Consumer Failure Test

Crash consumer after:

```text
DB commit
```

but before:

```text
ack
```

Verify:

```text
redelivery
→ idempotent behavior
```

---

# 243. External Side-Effect Crash

Crash after:

```text
provider success
```

but before local completion.

Verify:

```text
provider/business idempotency
```

---

# 244. Projection Lag Test

Slow consumer intentionally.

Verify:

```text
lag observable
```

and:

```text
source remains correct
```

---

# 245. Event Storm Test

One command creates:

```text
10× normal event volume
```

Measure:

```text
broker
consumers
cache
WebSocket
```

---

# 246. Backpressure Test

Consumer slows down.

Verify:

```text
queue grows
```

but:

```text
memory remains bounded
```

---

# 247. Tenant Flood Test

One tenant generates:

```text
huge event volume
```

Verify:

```text
other tenants remain healthy
```

---

# 248. Broker Outage Test

Stop broker.

Verify:

```text
publishing behavior
outbox growth
consumer recovery
```

---

# 249. Outbox Recovery Test

Broker unavailable for:

```text
10 minutes
```

Then returns.

Verify:

```text
outbox drains safely
```

without excessive retry storm.

---

# 250. Consumer Lag Recovery

Create:

```text
1M event backlog
```

Measure:

```text
recovery time
```

and:

```text
normal traffic impact
```

---

# 251. Schema Upgrade Test

Run:

```text
consumer v1
producer v1
```

then:

```text
consumer v2
producer v2
```

with overlap.

Verify compatibility.

---

# 252. Replay Authorization Test

Attempt replay from an unauthorized operator/tenant.

Expected:

```text
rejection
```

---

# 253. Event Security Test

Attempt to publish:

```text
privileged event
```

from an untrusted source.

Expected:

```text
rejection
```

---

# 254. Sensitive Data Test

Inspect event payloads for:

```text
password
token
secret
```

and verify absence.

---

# 255. Load Testing

Measure:

```text
events/sec
publish latency
consume latency
consumer lag
DB load
cache load
WebSocket fan-out
CPU
memory
network
```

---

# 256. Capacity Model

Approximate:

```text
total work
≈
events/sec
×
consumer groups
×
average handler cost
```

Then include:

```text
retries
```

and:

```text
fan-out
```

---

# 257. Consumer Capacity

For a consumer:

```text
capacity
≈
concurrency
/
average processing time
```

as a rough reasoning tool.

Real systems require measurement.

---

# 258. Event Payload Cost

Cost includes:

```text
serialization
broker storage
network
consumer parsing
DB writes
```

---

# 259. Event Frequency

A high-frequency event should be evaluated for:

```text
coalescing
aggregation
sampling
```

where intermediate states are not business-critical.

---

# 260. Event Batching

Batching can lower:

```text
per-message overhead
```

but raises:

```text
latency
```

and:

```text
failure granularity
```

---

# 261. Event Compression

Compression reduces:

```text
network/storage
```

but costs:

```text
CPU
```

Benchmark.

---

# 262. Event Retention Cost

Long retention costs:

```text
storage
replication
indexing
backup
privacy review
```

---

# 263. Replay Cost

Replay can temporarily consume:

```text
DB
CPU
network
consumer capacity
```

Run controlled replay.

---

# 264. Shadow Consumers

A new consumer can process a copy of production events without affecting primary behavior.

Useful for:

```text
migration validation
performance testing
```

---

# 265. Canary Consumer

Deploy to a small portion of partitions/traffic first.

Observe:

```text
errors
lag
latency
resource usage
```

---

# 266. Consumer Version Rollout

Use:

```text
v1
→ v1/v2
→ v2
```

with compatibility.

---

# 267. Event Producer Rollout

Use:

```text
producer compatible with old consumers
→ upgrade consumers
→ emit new fields
```

---

# 268. Event Contract Governance

For mature systems, define:

```text
owner
schema review
breaking-change policy
deprecation
consumer inventory
```

---

# 269. Documentation

For each event:

```text
Name
Purpose
Producer
Schema
Version
Partition key
Ordering guarantee
Delivery guarantee
Retention
Consumers
PII classification
Replay behavior
```

---

# 270. Operational Runbook

Create procedures for:

```text
broker outage
consumer lag
dead-letter explosion
outbox growth
event storm
schema incompatibility
duplicate side effects
tenant flood
projection divergence
replay
```

---

# 271. Incident — Outbox Growth

Symptoms:

```text
outbox rows continuously rising
```

Investigate:

```text
publisher down
broker unavailable
publish latency
poison event
```

---

# 272. Incident — Consumer Lag

Symptoms:

```text
lag increasing
```

Investigate:

```text
handler latency
DB
downstream
concurrency
event volume
```

---

# 273. Incident — Dead-Letter Explosion

Symptoms:

```text
many events dead-lettered
```

Investigate:

```text
schema change
dependency outage
consumer regression
bad producer
```

---

# 274. Incident — Duplicate Side Effect

Symptoms:

```text
emails twice
```

Investigate:

```text
ack timing
consumer crash
provider idempotency
inbox
```

---

# 275. Incident — Projection Divergence

Symptoms:

```text
DB says done
projection says open
```

Investigate:

```text
missing event
consumer failure
ordering
projection bug
```

---

# 276. Incident — Event Storm

Symptoms:

```text
broker traffic spikes
consumers saturate
```

Investigate:

```text
producer loop
fan-out
recursive events
bad retry
```

---

# 277. Incident — Event Cycle

Symptoms:

```text
event volume increases without user traffic
```

Look for:

```text
A → B → C → A
```

---

# 278. Incident — Schema Break

Symptoms:

```text
consumer parse failures
```

Contain:

```text
stop incompatible producer
roll back
restore compatibility
```

---

# 279. Incident — Tenant Leak

Symptoms:

```text
wrong tenant projection/cache/event
```

Priority:

```text
contain
audit
fix
replay from trusted source
```

---

# 280. Incident — Replay Storm

Operator replays:

```text
billions of events
```

without capacity isolation.

System overloads.

Use:

```text
bounded replay
dedicated capacity
rate limits
```

---

# 281. Incident — Broker Recovery Flood

Broker returns after outage.

All pending messages may become available.

Use:

```text
controlled consumer ramp
dependency protection
```

---

# 282. Incident — Poison Event Blocks Partition

One ordered partition contains a permanently failing event.

Possible strategy:

```text
dead-letter after bounded retries
```

while preserving explicit ordering semantics.

---

# 283. Incident — Current State Mismatch

Consumer rebuilds projection from current DB reads rather than event-time facts.

Historical projection becomes inconsistent.

Define replay semantics clearly.

---

# 284. Incident — Cache Invalidation Lag

Cache remains stale because consumer lag is high.

Do not claim the source state is wrong.

Measure:

```text
source version
cache version
consumer position
```

---

# 285. Incident — WebSocket Fan-Out Saturation

One event has:

```text
500k subscribers
```

Apply:

```text
coalescing
partitioning
fan-out scaling
payload reduction
```

---

# 286. Incident — Analytics Failure

Analytics consumer is down.

Business API should usually continue if analytics is non-critical.

This demonstrates why independent consumers create failure isolation.

---

# 287. Incident — Notification Failure

Email provider is down.

Task completion should generally not roll back solely because a non-critical notification failed.

Retry notification asynchronously.

---

# 288. Incident — Critical Consumer Failure

If a consumer performs essential business processing:

```text
payment settlement
```

failure semantics may require stronger:

```text
workflow state
retry
reconciliation
```

than ordinary analytics.

---

# 289. Security Architecture

```text
External
   ↓
Ingress verification
   ↓
Normalized internal event
   ↓
Trusted event bus
   ↓
Authorized consumers
```

Do not let arbitrary external messages impersonate internal facts.

---

# 290. Event Producer Identity

Store/propagate:

```text
producer
```

and verify trusted publishing paths.

---

# 291. Audit Trail

Important event pipelines should preserve enough metadata to investigate:

```text
who initiated
what changed
which event
which consumers
which side effects
```

---

# 292. Data Minimization

An event should contain:

```text
minimum data required
```

for its intended consumers.

---

# 293. Event Schema Registry vs Documentation

A schema registry can validate structure.

Documentation explains:

```text
business meaning
```

You need both at scale.

---

# 294. Event Contract Ownership

Someone must be accountable for:

```text
meaning
compatibility
deprecation
```

---

# 295. Production Architecture

Recommended modules:

```text
src/
├── domain/
│   ├── tasks/
│   ├── projects/
│   └── comments/
├── application/
│   ├── commands/
│   ├── consumers/
│   └── workflows/
├── infrastructure/
│   ├── db/
│   ├── outbox/
│   ├── inbox/
│   ├── broker/
│   ├── cache/
│   └── telemetry/
├── http/
├── realtime/
└── workers/
```

---

# 296. Event Interface

Concept:

```js
function createEvent({
  id,
  type,
  version,
  tenantId,
  aggregate,
  correlationId,
  causationId,
  occurredAt,
  data
}) {
  return {
    id,
    type,
    version,
    tenantId,
    aggregate,
    correlationId,
    causationId,
    occurredAt,
    data
  };
}
```

---

# 297. Consumer Interface

Concept:

```js
const consumer = {
  eventType: "task.completed",

  async handle(event, context) {
    // validate
    // dedupe
    // apply
    // commit
  }
};
```

---

# 298. Consumer Registry

Concept:

```js
const consumers = {
  "task.completed": [
    notifyConsumer,
    analyticsConsumer,
    cacheConsumer
  ]
};
```

A distributed broker will usually replace this local registry with consumer groups/subscriptions.

---

# 299. Inbox Handler

Concept:

```js
async function handleOnce(event, consumer, tx) {
  const inserted =
    await tx.inbox.insertIfAbsent(
      consumer,
      event.id
    );

  if (!inserted) {
    return { status: "duplicate" };
  }

  await consumer.handle(event, { tx });

  return { status: "processed" };
}
```

---

# 300. Outbox Publisher

Concept:

```js
async function publishOutboxBatch() {
  const rows =
    await outbox.claimBatch();

  for (const row of rows) {
    try {
      await broker.publish(row);
      await outbox.markPublished(row.id);
    } catch (error) {
      await outbox.recordFailure(
        row.id,
        classify(error)
      );
    }
  }
}
```

Real implementations require:

```text
lease
retry
ordering
shutdown
bounded batches
```

---

# 301. Cache Invalidation Consumer

Concept:

```js
async function handleTaskUpdated(event) {
  await cache.delete(
    `task:v2:${event.tenantId}:${event.aggregate.id}`
  );

  await cache.delete(
    `task-list:v2:${event.tenantId}:${event.data.projectId}`
  );
}
```

The exact invalidation graph must match actual cache dependencies.

---

# 302. WebSocket Consumer

Concept:

```js
async function handleTaskUpdated(event) {
  await realtime.publishToProject(
    event.tenantId,
    event.data.projectId,
    event
  );
}
```

The gateway still enforces:

```text
subscription authorization
```

and:

```text
bounded delivery
```

---

# 303. Job Bridge

Concept:

```js
async function handleExportRequested(event) {
  await jobs.enqueueOnce({
    key: `export:${event.data.exportId}`,
    type: "export.generate",
    payload: {
      exportId: event.data.exportId
    }
  });
}
```

---

# 304. Projection Consumer

Concept:

```js
async function handleTaskCompleted(event, { tx }) {
  await tx.taskSummary.upsert({
    taskId: event.aggregate.id,
    version: event.aggregate.version,
    completed: true
  });
}
```

Use version checks to reject stale events where required.

---

# 305. Version Guard

Concept:

```js
if (
  event.aggregate.version <=
  current.version
) {
  return;
}
```

This can protect projections from stale/repeated updates.

---

# 306. Track A — Core Theory

Study:

```text
event-driven architecture
commands/events
domain/integration events
outbox
inbox
idempotency
delivery semantics
ordering
partitioning
consumer groups
eventual consistency
projections
event sourcing
replay
schema evolution
sagas
choreography
orchestration
compensation
backpressure
consumer lag
fan-out
event storms
security
observability
```

---

# 307. Track B — Implementation

Build:

```text
domain events
event envelope
outbox
publisher
broker adapter
consumer registry
inbox
idempotent consumers
retry
dead-letter
projection
replay
cache invalidation consumer
WebSocket consumer
job bridge
analytics consumer
search consumer
schema versioning
correlation/tracing
metrics
failure injection
load tests
operational tools
```

---

# 308. Track C — Interview / Reasoning

Defend:

```text
Why events?
Why not direct HTTP calls?
Why outbox?
Why inbox?
Why at-least-once?
Why not exactly-once?
Why this event granularity?
Why this partition key?
Why this consistency model?
Why projection?
Why event sourcing or not?
Why saga?
Why choreography or orchestration?
How do you recover missing events?
How do you prevent duplicate side effects?
How do you handle consumer lag?
How do you control event storms?
How do you version contracts?
```

---

# 309. Mastery Gate

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

# 310. Implementation Progression

## Stage 1 — Guided

Build:

```text
in-process event bus
domain events
two consumers
```

## Stage 2 — Partially Guided

Add:

```text
outbox
inbox
idempotency
```

## Stage 3 — No Reference

Add:

```text
retries
dead-letter
projection
replay
```

## Stage 4 — Edge-Case Hardened

Add:

```text
ordering
consumer lag
schema evolution
event storms
shutdown
tenant fairness
```

## Stage 5 — Production Grade

Add:

```text
distributed broker
cache consumer
WebSocket consumer
job bridge
observability
security
failure injection
load tests
deployment
runbooks
```

---

# 311. Debugging Exercises

## Exercise 1 — Lost Event

```text
DB updated
but consumers saw nothing
```

Trace:

```text
transaction
outbox
publisher
broker
consumer
```

---

## Exercise 2 — Duplicate Email

Trace:

```text
event
→ consumer
→ provider
→ crash
→ retry
```

Design safe behavior.

---

## Exercise 3 — Projection Stale

```text
DB version = 8
projection version = 6
```

Determine:

```text
lag
```

and:

```text
recovery
```

---

## Exercise 4 — Projection Regresses

Consumer receives:

```text
version 8
version 7
```

Design:

```text
version guard
```

---

## Exercise 5 — Event Storm

One task update produces:

```text
1,000 cache invalidations
```

Identify the dependency graph.

---

## Exercise 6 — Event Cycle

```text
A
→ B
→ C
→ A
```

Find the loop.

---

## Exercise 7 — Consumer Lag

Incoming:

```text
10,000/sec
```

Processing:

```text
8,000/sec
```

Predict backlog growth.

---

## Exercise 8 — Broker Recovery

Broker comes back after:

```text
15 minutes
```

Determine:

```text
drain strategy
```

---

# 312. Code Review Exercise

Review:

```js
await db.task.update(task);

await broker.publish({
  type: "task.updated",
  data: task
});
```

Find the failure window.

Design the outbox solution.

---

# 313. Code Review Exercise

Review:

```js
async function consume(event) {
  await sendEmail(event.data.email);
  await ack(event.id);
}
```

Identify the crash window and duplicate effect.

---

# 314. Code Review Exercise

Review:

```js
async function consume(event) {
  await ack(event.id);
  await updateProjection(event);
}
```

Identify the data-loss window.

---

# 315. Code Review Exercise

Review:

```js
async function handle(event) {
  const existing =
    await db.inbox.find(event.id);

  if (!existing) {
    await db.inbox.insert(event.id);
    await updateProjection(event);
  }
}
```

Identify the race between:

```text
find
→ insert
```

---

# 316. Code Review Exercise

Review:

```js
broker.on("task.updated", async event => {
  await cache.delete(
    `task:${event.data.taskId}`
  );

  await publish(event);
});
```

Ask:

```text
Is this consumer causing event recursion?
Who owns publish?
What happens if cache delete fails?
```

---

# 317. Interview Questions — Senior

1. What is event-driven architecture?
2. What is a domain event?
3. What is an integration event?
4. Why use the outbox pattern?
5. Why use an inbox?
6. How do you handle duplicate events?
7. How do you handle event ordering?
8. What is eventual consistency?
9. What is consumer lag?
10. What is a saga?

---

# 318. Interview Questions — Principal

1. Design an event-driven platform for millions of events/sec.
2. How do you guarantee database changes are eventually published?
3. How do you prevent duplicate external side effects?
4. How do you recover a projection from missing events?
5. How do you handle consumer schema evolution?
6. How do you choose partition keys?
7. How do you isolate tenants?
8. How do you prevent retry/event storms?
9. How do you decide between choreography and orchestration?
10. When should you reject event-driven architecture entirely?

---

# 319. Predict-the-Behavior Exercises

## Exercise A

```text
DB commit succeeds
outbox write fails
```

What should happen?

Answer only after predicting.

---

## Exercise B

```text
event delivered
consumer DB commit succeeds
consumer crashes
ack never happens
event delivered again
```

What property must the consumer have?

---

## Exercise C

```text
producer = 5,000 events/sec
consumer = 4,000 events/sec
```

Predict:

```text
lag after 10 seconds
```

Ignoring retries and other factors:

```text
approximately 10,000 events
```

Then discuss why real systems are more complex.

---

## Exercise D

Events:

```text
task.version=10
task.version=8
```

Should projection apply both?

Explain.

---

# 320. Mastery Exercises

## Level 1 — In-Process Events

Build:

```text
event emitter
consumer registry
event schema
```

---

## Level 2 — Durable Events

Add:

```text
outbox
publisher
inbox
```

---

## Level 3 — Projections

Build:

```text
task summary projection
```

from events.

---

## Level 4 — Recovery

Add:

```text
replay
dedupe
version guards
```

---

## Level 5 — Integration

Connect:

```text
cache
WebSocket
job queue
analytics
```

---

## Level 6 — Distributed

Run:

```text
API
publisher
consumer A
consumer B
broker
DB
cache
```

as independent processes.

---

## Level 7 — Production

Add:

```text
schema governance
observability
failure injection
load tests
security
runbooks
deployment
```

---

# 321. Production Acceptance Criteria

```text
[ ] source of truth documented
[ ] event ownership documented
[ ] command/event distinction
[ ] event envelope
[ ] event IDs
[ ] correlation IDs
[ ] causation IDs
[ ] schema versioning
[ ] outbox
[ ] atomic outbox write
[ ] publisher recovery
[ ] duplicate publication handling
[ ] inbox/dedupe
[ ] consumer idempotency
[ ] delivery semantics
[ ] ordering semantics
[ ] partitioning
[ ] retries
[ ] backoff
[ ] dead-letter
[ ] consumer lag
[ ] projection
[ ] replay
[ ] reconciliation
[ ] cache integration
[ ] WebSocket integration
[ ] job integration
[ ] saga/workflow where required
[ ] tenant isolation
[ ] security
[ ] observability
[ ] graceful shutdown
[ ] schema compatibility
[ ] load tests
[ ] failure injection
[ ] operational runbooks
```

---

# 322. Operational Checklist

```text
[ ] event throughput monitored
[ ] publish latency monitored
[ ] consumer lag monitored
[ ] dead letters monitored
[ ] outbox depth monitored
[ ] retry rate monitored
[ ] projection lag monitored
[ ] event storm alerts
[ ] tenant volume monitored
[ ] broker health monitored
[ ] replay tooling protected
[ ] event retention bounded
[ ] schema registry/catalog maintained
[ ] consumer ownership documented
[ ] disaster recovery tested
```

---

# 323. Principal Decision Framework

For each event flow, ask:

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

Then:

```text
Who owns the truth?
What happens if the event is duplicated?
What happens if it is delayed?
What happens if it is missing?
What happens if the consumer is down?
What happens if the broker is down?
What happens during replay?
What happens after schema evolution?
What happens if one tenant floods the stream?
What happens if an external side effect already succeeded?
```

---

# 324. Dependency Graph

```text
Chapter 31–38 — Async JavaScript
              ↓
Chapter 55–57 — Networking / Security
              ↓
Chapter 58–63 — Node Runtime / Diagnostics
              ↓
Chapter 79–85 — API / DB / Observability / Reliability / Performance
              ↓
Chapter 86–89 — Testing / Debugging / Review
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

# 325. Concept Connections

## Depends On

```text
REST
WebSockets
job queues
cache
databases
async execution
security
observability
reliability
testing
```

## Builds Toward

```text
production backends
large-scale JavaScript platforms
distributed workflows
event-driven architecture
```

## Revisited

```text
transactions
idempotency
backpressure
streams
errors
graceful shutdown
authorization
caching
concurrency
```

## Why This Chapter Matters Later

This chapter turns isolated backend techniques into a coherent distributed system.

You now have:

```text
request/response
+
persistent connections
+
background work
+
derived state
+
event propagation
```

That combination is the foundation of many production platforms.

---

# 326. Spaced Retrieval Schedule

### Day 0

```text
commands
events
outbox
inbox
```

### Day 1

```text
idempotency
ordering
consumer lag
projections
```

### Day 3

```text
replay
schema evolution
cache/WebSocket/job consumers
```

### Day 7

```text
sagas
failure recovery
event storms
tenant isolation
```

### Day 14

```text
load testing
reconciliation
operational governance
```

### Day 30

Build an event-driven application from scratch.

### Day 60

Design a replayable multi-tenant event platform.

### Day 90

Defend event-driven architecture against a principal-level design review.

---

# 327. Revision / Retrieval Record

```md
# Chapter 109 — Revision / Retrieval Record

## Review #
- Date:
- Duration:
- Status before:
- Status after:

## Retrieval
- event-driven architecture [ ]
- commands vs events [ ]
- domain events [ ]
- integration events [ ]
- event ownership [ ]
- event envelope [ ]
- event ID [ ]
- correlation ID [ ]
- causation ID [ ]
- ordering [ ]
- partitioning [ ]
- outbox [ ]
- inbox [ ]
- idempotent consumers [ ]
- delivery semantics [ ]
- eventual consistency [ ]
- projections [ ]
- replay [ ]
- reconciliation [ ]
- schema evolution [ ]
- retries [ ]
- dead letters [ ]
- consumer lag [ ]
- backpressure [ ]
- fan-out [ ]
- cache integration [ ]
- WebSocket integration [ ]
- job integration [ ]
- sagas [ ]
- choreography/orchestration [ ]
- compensation [ ]
- security [ ]
- observability [ ]
- load testing [ ]

## Build Evidence
- Repository:
- Commit:
- Broker:
- Database:
- Projection:
- Consumer count:
- Replay result:
- Failure-injection result:
- Load-test result:

## Gaps
-

## New Insights
-

## Follow-up
-
```

---

# Chapter 109 — Canonical References and Source Discipline

Primary references:

1. Node.js Documentation  
   https://nodejs.org/docs/

2. ECMAScript Language Specification  
   https://tc39.es/ecma262/

3. HTTP Semantics  
   https://httpwg.org/specs/

4. OWASP API Security  
   https://owasp.org/www-project-api-security/

5. OWASP Cheat Sheets  
   https://cheatsheetseries.owasp.org/

6. PostgreSQL Documentation  
   https://www.postgresql.org/docs/

7. Apache Kafka Documentation  
   https://kafka.apache.org/documentation/

8. Redis Documentation  
   https://redis.io/docs/

Source discipline:

```text
JavaScript semantics
→ ECMAScript

Node behavior
→ Node documentation

HTTP semantics
→ HTTP specifications

database transaction behavior
→ chosen database documentation

broker semantics
→ chosen broker documentation

security
→ OWASP + actual deployment threat model

performance
→ load tests + profiles + telemetry

event consistency
→ explicit architecture contract
```

Never infer:

```text
exactly-once business effects
```

from:

```text
a broker's delivery wording
```

without analyzing every side-effect boundary.

---

# 328. Completion Snapshot

```text
Part XX — Projects

Chapter 109 — Production JavaScript Event-Driven Application
[ ] Not Started

Track A — Core Theory
[ ] event-driven architecture
[ ] command/event distinction
[ ] domain events
[ ] integration events
[ ] event ownership
[ ] envelopes
[ ] event identity
[ ] correlation
[ ] causation
[ ] outbox
[ ] inbox
[ ] idempotency
[ ] delivery semantics
[ ] ordering
[ ] partitioning
[ ] consumer groups
[ ] eventual consistency
[ ] projections
[ ] replay
[ ] reconciliation
[ ] schema evolution
[ ] retries
[ ] dead letters
[ ] consumer lag
[ ] backpressure
[ ] fan-out
[ ] cache integration
[ ] WebSocket integration
[ ] job integration
[ ] sagas
[ ] choreography
[ ] orchestration
[ ] compensation
[ ] security
[ ] observability
[ ] cost

Track B — Implementation
[ ] event model
[ ] outbox table
[ ] publisher
[ ] broker adapter
[ ] inbox table
[ ] consumers
[ ] dedupe
[ ] retry engine
[ ] dead-letter
[ ] projection
[ ] replay
[ ] reconciliation
[ ] cache invalidator
[ ] WebSocket publisher
[ ] job bridge
[ ] analytics consumer
[ ] search consumer
[ ] workflow
[ ] metrics
[ ] logs
[ ] tracing
[ ] load tests
[ ] failure injection
[ ] runbooks

Track C — Interview / Reasoning
[ ] Explain event-driven architecture
[ ] Explain commands vs events
[ ] Explain outbox
[ ] Explain inbox
[ ] Explain idempotency
[ ] Explain ordering
[ ] Explain eventual consistency
[ ] Explain projections
[ ] Explain replay
[ ] Explain consumer lag
[ ] Explain event storms
[ ] Explain schema evolution
[ ] Explain saga
[ ] Explain choreography
[ ] Explain orchestration
[ ] Explain compensation
[ ] Explain tenant isolation
[ ] Defend event-driven architecture
[ ] Defend when not to use events

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

# 329. Completion Criteria

Do not mark mastery because:

```text
an event was published successfully
```

You are ready to continue when you can independently:

1. Explain event-driven architecture.
2. Distinguish commands from events.
3. Identify event ownership.
4. Define stable event envelopes.
5. Implement transactional outbox.
6. Handle duplicate publication.
7. Implement inbox/deduplication.
8. Build idempotent consumers.
9. Define delivery semantics.
10. Define ordering semantics.
11. Choose partition keys.
12. Model eventual consistency.
13. Build projections.
14. Measure projection lag.
15. Rebuild projections.
16. Detect divergence.
17. Design replay safely.
18. Evolve event schemas compatibly.
19. Control retries.
20. Handle dead letters.
21. Control consumer backpressure.
22. Integrate cache invalidation.
23. Integrate WebSocket fan-out.
24. Integrate background jobs.
25. Design sagas where required.
26. Choose choreography vs orchestration.
27. Design compensating actions.
28. Secure event producers and consumers.
29. Load test event pipelines.
30. Defend the architecture at principal level.

---

# Final Mental Model

```text
                         ┌───────────────┐
                         │ REST / Client │
                         └───────┬───────┘
                                 │
                                 ▼
                           COMMAND
                                 │
                                 ▼
                      ┌────────────────────┐
                      │ Domain/Application │
                      └─────────┬──────────┘
                                │
                         DB TRANSACTION
                         ┌──────┴──────┐
                         │             │
                         ▼             ▼
                    SOURCE STATE     OUTBOX
                                      │
                                      ▼
                              ┌────────────┐
                              │ EVENT BUS  │
                              └─────┬──────┘
                 ┌──────────────────┼────────────────────┐
                 ▼                  ▼                    ▼
             CACHE              REALTIME              JOBS
          INVALIDATION          WEBSOCKET             QUEUE
                 │                  │                    │
                 ▼                  ▼                    ▼
              CACHE             CLIENTS              WORKERS
                                                        │
                         ┌──────────────────────────────┘
                         ▼
                    PROJECTIONS
                         │
                         ▼
                    ANALYTICS
```

A robust event-driven application separates:

```text
command acceptance
authoritative state
event publication
event delivery
consumer processing
derived state
live notification
background execution
```

The deepest lesson is:

```text
Event
≠
guaranteed business effect
```

Correctness comes from:

```text
transaction boundaries
durable publication
idempotent consumers
explicit ordering
bounded retries
replay strategy
schema evolution
reconciliation
authorization
observability
```

A production event-driven platform must remain correct when:

```text
events duplicate
events delay
consumers crash
brokers fail
projections lag
schemas evolve
tenants flood
side effects become uncertain
```

> **Mastery reminder:** The purpose of event-driven architecture is not to make systems more asynchronous. It is to create explicit boundaries where independent components can evolve and fail without silently corrupting business state.