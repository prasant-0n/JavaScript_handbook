# Chapter 121 — System Design Assessment — 5 Principal-Level Problems

> **JavaScript Mastery — Part XXI: Assessment**
>
> **Assessment:** 5 principal-level system design problems requiring end-to-end reasoning across requirements, APIs, data, asynchronous processing, scaling, reliability, security, observability, cost, and evolutionary architecture.
>
> **Role perspective:** Principal JavaScript Engineer · Distributed Systems Designer · Node.js Architect · Platform Architect · Reliability Engineer · Security Engineer
>
> **Status:** `[ ] Not Started`
>
> **Core rule:** **Design the system that the requirements demand—not the architecture you happen to know.**

---

# 1. Assessment Mission

This assessment is the bridge between:

```text
knowing JavaScript
```

and:

```text
designing systems built with JavaScript.
```

You are expected to reason across:

```text
requirements
→ constraints
→ users/actors
→ APIs
→ data
→ state
→ consistency
→ async work
→ concurrency
→ failure
→ security
→ observability
→ scaling
→ deployment
→ migration
→ cost
```

The assessment is intentionally framework-neutral.

Use:

```text
Node.js
browser
queues
databases
caches
workers
events
HTTP
WebSockets
```

only where they solve a demonstrated requirement.

---

# 2. What Principal-Level Means Here

A principal-level design does not merely contain:

```text
API
database
cache
queue
```

It explains:

```text
why each exists
who owns it
what can fail
what is guaranteed
what is eventually consistent
how state changes
how the system scales
how incidents are detected
how the system evolves
```

A principal engineer also knows:

```text
what NOT to build
```

---

# 3. Assessment Scoring

Each problem:

```text
0–20 points
```

Total:

```text
100 points
```

Recommended interpretation:

```text
90–100 → Principal-level system design
80–89  → Strong architecture/system design
70–79  → Good design with targeted gaps
60–69  → Significant system-design gaps
<60     → Rebuild architecture fundamentals before Chapter 122
```

---

# 4. Full-Credit Standard

A full-credit design covers:

```text
1. requirements
2. scale assumptions
3. APIs/interfaces
4. data model
5. component boundaries
6. request/data flows
7. consistency model
8. failure model
9. concurrency model
10. security model
11. observability
12. capacity/scaling
13. deployment
14. migration
15. trade-offs
```

A diagram without reasoning is incomplete.

---

# 5. Universal Answer Structure

For every problem use:

```md
# System Design

## 1. Requirements
### Functional
-
### Non-Functional
-

## 2. Assumptions
-

## 3. Scale
-

## 4. Users / Actors
-

## 5. High-Level Architecture
-

## 6. API / Interface Design
-

## 7. Data Model
-

## 8. Core Workflows
-

## 9. Async / Concurrency Model
-

## 10. Consistency Model
-

## 11. Failure Handling
-

## 12. Security
-

## 13. Observability
-

## 14. Capacity / Scaling
-

## 15. Deployment / Operations
-

## 16. Migration / Evolution
-

## 17. Trade-Offs
-

## 18. Rejected Alternatives
-
```

---

# 6. Problem 1 — Multi-Tenant Order Platform

Design a SaaS order platform for multiple retail tenants.

### Requirements

Customers can:

```text
create orders
view orders
update order status
receive notifications
```

Tenants require:

```text
tenant isolation
role-based access
audit history
```

The platform must support:

```text
10,000 tenants
1,000,000 total users
5,000 requests/sec average
20,000 requests/sec peak
```

Order creation may trigger:

```text
inventory update
payment workflow
notification
analytics event
```

### Constraints

```text
Node.js backend
relational database
browser client
background jobs
```

### Task

Design:

```text
tenant isolation
API boundaries
database ownership
order state machine
payment/inventory workflows
event publication
notification processing
audit model
authorization
```

Then explain:

```text
shared database
vs
database-per-tenant
vs
hybrid tenancy
```

Choose one and justify it.

### Required Failure Analysis

Discuss:

```text
database outage
duplicate order request
payment timeout
inventory conflict
queue delay
notification failure
cross-tenant access bug
```

### Principal Extension

Define:

```text
how the platform behaves if one large tenant consumes disproportionate capacity
```

Address:

```text
rate limits
tenant quotas
fairness
noisy-neighbor control
```

---

# 7. Problem 2 — Real-Time Collaboration System

Design a collaborative document platform where many users edit shared documents.

### Requirements

Users can:

```text
join a document
edit content
see other users' changes
see presence
reconnect after temporary network loss
```

Requirements include:

```text
real-time updates
durable document state
conflict handling
reconnection
history
```

Assume:

```text
1,000,000 daily users
100,000 concurrent connections
10,000 active documents
```

### Task

Design:

```text
WebSocket architecture
connection routing
document ownership
state synchronization
durability
presence
reconnection
event ordering
```

Then choose and justify an approach for concurrent edits, such as:

```text
server serialization
operation-based model
CRDT-style model
```

You do not need to implement a full CRDT.

You must explain:

```text
why the chosen consistency model matches the product requirements
```

### Failure Analysis

Handle:

```text
client disconnect
server crash
duplicate message
out-of-order message
network partition
hot document
```

### Principal Extension

Explain how you would observe:

```text
connection health
message latency
document synchronization lag
dropped updates
reconnect storms
```

---

# 8. Problem 3 — Job Processing Platform

Design a durable asynchronous job platform.

Applications submit jobs such as:

```text
email
report generation
image processing
data export
```

Requirements:

```text
durable submission
at-least-once delivery
retries
dead-letter handling
delayed jobs
priority
horizontal workers
```

Scale:

```text
100,000 jobs/minute
5,000 workers
large and small jobs mixed
```

### Task

Design:

```text
producer API
queue
job record
claim protocol
worker lifecycle
visibility timeout
retry policy
dead-letter queue
idempotency
```

Explain:

```text
why exactly-once processing is difficult
```

and what your application contract should guarantee instead.

### Failure Analysis

Handle:

```text
worker crash
duplicate execution
poison job
queue outage
database outage
long-running job
stuck claim
retry storm
```

### Principal Extension

Design:

```text
fair scheduling
tenant quotas
priority starvation protection
backpressure
capacity planning
```

---

# 9. Problem 4 — Global Product Catalog

Design a globally distributed product catalog.

Requirements:

```text
read-heavy
frequent reads
moderate writes
search
filtering
availability by region
price updates
inventory updates
```

Scale:

```text
50 million products
100,000 read requests/sec
5,000 write requests/sec
```

Users expect:

```text
fast reads
high availability
region-aware results
```

They can tolerate limited staleness for:

```text
descriptions
images
search indexes
```

They cannot tolerate unacceptable stale values for:

```text
price
critical availability
```

### Task

Design:

```text
source of truth
read model
search index
cache
regional replication
consistency boundaries
invalidations
```

Explicitly classify data as:

```text
strongly consistent
eventually consistent
derived
cacheable
```

### Failure Analysis

Handle:

```text
region outage
replication lag
stale cache
search index lag
price update race
inventory update race
cache stampede
```

### Principal Extension

Explain how you would change the architecture at:

```text
1×
10×
100×
```

without assuming that every component should scale identically.

---

# 10. Problem 5 — Production Developer Platform

Design an internal developer platform for a company with:

```text
500 engineers
100+ Node.js services
multiple teams
multiple environments
shared CI/CD
central observability
```

The platform should provide:

```text
service templates
configuration
secrets integration
deployment workflows
logs
metrics
traces
dependency visibility
health checks
safe rollout
rollback
```

### Task

Design:

```text
service lifecycle
platform APIs
developer experience
ownership model
deployment architecture
configuration model
secrets model
observability standards
service catalog
dependency graph
```

The platform must reduce:

```text
duplicate tooling
deployment inconsistency
incident diagnosis time
unsafe production changes
```

### Principal Extension

Design governance that does not become a bottleneck.

Explain:

```text
what should be standardized
what should remain team-owned
how exceptions are handled
how platform changes are rolled out
```

---

# 11. Architecture Diagram Requirement

For every problem create at least three diagrams:

### Context Diagram

```text
Users
  ↓
System
  ↓
External Dependencies
```

### Component Diagram

```text
API
 ↓
Application
 ↓
Data / Queues / Workers
```

### Critical Flow

```text
request
→ validation
→ authorization
→ state change
→ persistence
→ async side effect
→ observation
```

Diagrams may be ASCII.

Correctness matters more than visual style.

---

# 12. Capacity Planning

For every system estimate:

```text
requests/sec
reads/sec
writes/sec
events/sec
connections
payload size
storage growth
cache size
queue depth
worker capacity
```

Example:

```text
RPS × average payload
```

can approximate a bandwidth baseline.

For queue systems reason from:

```text
incoming work
vs
processing capacity
```

A queue grows when:

```text
arrival rate > sustainable service rate
```

for long enough.

---

# 13. Back-of-the-Envelope Calculation Requirement

Do not provide fake precision.

State assumptions.

Example:

```text
Assume:
2 KB average request payload
5,000 requests/sec

Ingress payload ≈
5,000 × 2 KB
≈ 10 MB/sec
≈ 864 GB/day before overhead
```

Then identify missing real-world factors:

```text
headers
responses
compression
retries
replication
storage indexes
burstiness
```

The value is the reasoning, not the exact number.

---

# 14. Data Modeling Requirement

For every system identify:

```text
source of truth
primary identifiers
ownership
indexes
retention
consistency
derived views
cache copies
audit records
```

Ask:

```text
What data can be rebuilt?

What data cannot be lost?

What data can be stale?

What data requires transactionality?
```

---

# 15. Consistency Decision Framework

Classify operations:

```text
strong consistency
read-your-writes
eventual consistency
causal ordering
best-effort
```

Then state why.

Do not use:

```text
eventual consistency
```

as a synonym for:

```text
fast
```

and do not use:

```text
strong consistency
```

as a synonym for:

```text
correct architecture
```

Consistency is a business and system requirement.

---

# 16. API Design Requirement

Every design should define representative interfaces.

Include:

```text
method
path/topic
request
response
error
authentication
authorization
idempotency
pagination where needed
```

Examples:

```text
POST /orders
GET /orders/:id
POST /jobs
POST /documents/:id/operations
```

For event-driven systems define:

```text
event name
version
key
payload
delivery semantics
idempotency expectation
```

---

# 17. Failure Modeling

For every major component ask:

```text
What if it is slow?

What if it is unavailable?

What if it returns corrupt/invalid data?

What if it succeeds but the response is lost?

What if the client retries?

What if the worker crashes after side effect but before acknowledgement?

What if two writes race?

What if a dependency recovers suddenly and receives a flood?
```

Then decide:

```text
timeout
retry
circuit/isolation
fallback
queue
backpressure
degradation
reconciliation
```

---

# 18. Idempotency Requirement

For mutation APIs identify operations vulnerable to duplicates.

Examples:

```text
create order
charge payment
enqueue job
publish command
```

Define:

```text
idempotency key
scope
storage duration
duplicate behavior
failure semantics
```

Do not solve every duplicate problem by:

```text
"check first, then insert"
```

without considering concurrent requests.

---

# 19. Caching Requirement

For each cache define:

```text
what
why
TTL
maximum size
eviction
invalidation
staleness
failure behavior
stampede protection
observability
```

A cache is part of architecture, not merely a local optimization.

---

# 20. Queue Requirement

For every queue define:

```text
producer
consumer
ordering
delivery semantics
visibility/claim
retry
dead-letter
retention
backpressure
capacity
monitoring
```

Also answer:

```text
What happens when producers are faster than consumers?
```

---

# 21. Security Architecture Requirement

Every design must identify:

```text
authentication
authorization
tenant isolation
service identity
secrets
input validation
rate limiting
resource limits
audit
encryption requirements
least privilege
```

For multi-tenant systems explicitly explain:

```text
how cross-tenant access is prevented
```

Do not rely on:

```text
"the frontend will send the right tenant ID."
```

---

# 22. Observability Architecture Requirement

Every principal design should include:

```text
logs
metrics
traces
correlation/request IDs
domain events
health signals
SLOs
alerts
dashboards
```

At minimum measure:

```text
traffic
errors
latency
saturation
dependency health
queue depth where applicable
```

Add domain-specific signals.

Examples:

```text
orders created
payment failures
document sync lag
job age
tenant throttling
```

---

# 23. SLO / Reliability Requirement

For each system define representative objectives.

Examples:

```text
availability
latency percentile
job completion time
event delivery lag
data freshness
recovery time
```

Then identify:

```text
error budget
```

and explain how the budget affects:

```text
release risk
capacity work
reliability investment
```

Avoid inventing exact SLO targets without stating that they are assumptions.

---

# 24. Scaling Strategies

Evaluate:

```text
vertical scaling
horizontal scaling
partitioning
sharding
caching
replication
batching
streaming
work queues
read models
precomputation
```

For each ask:

```text
What bottleneck does this relieve?

What new bottleneck does it introduce?
```

---

# 25. Hotspot Analysis

Identify possible hotspots:

```text
hot tenant
hot key
hot document
hot partition
hot database row
cache hotspot
queue hotspot
single worker
single region
```

Then design:

```text
partitioning
rate limiting
load spreading
sharding
key design
fair scheduling
```

Do not solve every hotspot with more machines.

---

# 26. Deployment Architecture

Every design must address:

```text
build
test
artifact
configuration
secrets
rollout
health
rollback
migration
```

For risky changes consider:

```text
canary
blue/green
feature flag
gradual traffic shift
schema expand/contract
```

A deployment is a state transition in a running system.

---

# 27. Graceful Degradation

For major dependencies define:

```text
full capability
degraded capability
unavailable capability
recovery path
```

Examples:

```text
recommendations unavailable
→ core purchase still works

analytics unavailable
→ transaction continues

notification unavailable
→ durable notification job remains pending
```

Do not degrade security or correctness accidentally.

---

# 28. Disaster / Recovery Thinking

For significant systems consider:

```text
backup
restore
replication
RPO
RTO
failover
data corruption
region failure
dependency recovery
```

Ask:

```text
Can we restore?

How long does it take?

How do we verify restored correctness?
```

---

# 29. Migration Strategy

A production architecture should evolve through:

```text
current state
→ compatibility seam
→ incremental migration
→ observation
→ cutover
→ cleanup
```

Include:

```text
backward compatibility
data migration
rollback
dual-read/dual-write only when justified
feature flags
deprecation
```

Avoid:

```text
"rewrite everything"
```

as the default migration plan.

---

# 30. Cost Model

For major components identify the cost drivers:

```text
compute
memory
storage
network
database
queue
observability
third-party APIs
operations
engineering time
```

Ask:

```text
What is the most expensive resource?

What changes cost at 10× load?

What can be bounded?

What complexity are we buying?
```

---

# 31. Architecture Trade-Off Record

For every major decision write:

```md
### Decision
-

### Requirement It Serves
-

### Alternative
-

### Why Chosen
-

### New Risk
-

### Operational Cost
-

### Revisit Trigger
-
```

---

# 32. Rejected Alternatives

A strong design should contain:

```text
what was considered
what was rejected
why
```

Examples:

```text
microservices rejected because operational/team cost exceeds current benefit

strong consistency rejected for derived search because eventual consistency is acceptable

global cache rejected because invalidation complexity exceeds measurable benefit
```

A rejection demonstrates judgment.

---

# 33. Principal Design Review

For every system answer:

```text
What is the simplest architecture that satisfies the requirements?

Which component is most likely to become the first bottleneck?

Which failure has the largest blast radius?

Which state is hardest to keep correct?

Which component has the highest operational complexity?

Where is eventual consistency introduced?

Where are retries dangerous?

Where can users become noisy neighbors?

What would you remove if the team were half its size?

What changes at 10× scale?

What changes at 100× scale?
```

---

# 34. System Design Misdiagnosis Taxonomy

Classify weaknesses as:

```text
A — requirements ignored
B — scale assumptions absent
C — component diagram without data flow
D — unclear ownership
E — consistency undefined
F — failure boundaries missing
G — retry/idempotency failure
H — security boundary missing
I — observability missing
J — capacity not quantified
K — cache semantics undefined
L — queue semantics undefined
M — migration ignored
N — cost ignored
O — over-engineering
P — under-engineering
Q — no rejected alternatives
R — organizational constraints ignored
```

---

# 35. Assessment Rubric

### 0–4 — Incomplete

Major system requirements or flows missing.

### 5–8 — Functional Design

Core architecture works conceptually.

### 9–12 — Strong Design

Clear boundaries, data model, APIs, failures, and scaling.

### 13–16 — Advanced

Includes consistency, security, observability, capacity, and migration.

### 17–18 — Production

Trade-offs and operational reality are explicit.

### 19 — Principal

Strong technical reasoning under constraints.

### 20 — Principal + Judgment

Design is technically sound, operationally realistic, economically defensible, and evolves safely.

---

# 36. Retrieval Record

```md
# Chapter 121 — System Design Assessment — Retrieval Record

## Attempt
- Date:
- Duration:
- Score:
- Percentage:
- Status before:
- Status after:

## Problem Scores
- Problem 1:
- Problem 2:
- Problem 3:
- Problem 4:
- Problem 5:

## Strongest Areas
-

## Weakest Areas
-

## Requirements Gaps
-

## Data / Consistency Gaps
-

## Reliability Gaps
-

## Scalability Gaps
-

## Security Gaps
-

## Observability Gaps
-

## Cost / Trade-Off Gaps
-

## Migration Gaps
-

## Problems Requiring Rework
-

## Next Review
-
```

---

# 37. Spaced Retrieval Schedule

### Day 0

Complete all 5 designs.

### Day 1

Review:

```text
requirements
scale assumptions
failure boundaries
```

### Day 3

Redesign:

```text
Problem 1
Problem 2
```

without notes.

### Day 7

Redesign:

```text
Problem 3
Problem 4
```

with explicit consistency/capacity models.

### Day 14

Rework:

```text
Problem 5
```

as a platform architecture review.

### Day 21

Choose one problem and design:

```text
1×
10×
100×
```

variants.

### Day 30

Perform one full system design interview under a time limit without reference material.

---

# 38. Dependency Graph

```text
Chapters 01–30
        ↓
language + data + errors
        ↓
Chapters 31–70
        ↓
async + browser + Node + networking + modules
        ↓
Chapters 71–90
        ↓
algorithms + paradigms + production engineering
        ↓
Chapters 91–101
        ↓
modern JS + compatibility + judgment
        ↓
Chapters 102–111
        ↓
production project experience
        ↓
Chapters 112–120
        ↓
conceptual
→ output prediction
→ debugging
→ async/event loop
→ memory
→ performance
→ security
→ architecture
→ implementation
        ↓
Chapter 121 — System Design Assessment
        ↓
Chapter 122 — Final Principal JavaScript Project
```

---

# 39. Concept Connections

## Depends On

```text
all previous chapters
```

Especially:

```text
architecture
API design
database integration
async systems
queues
caching
observability
reliability
performance
security
testing
implementation
```

## Builds Toward

```text
Chapter 122 — Final Principal JavaScript Project
```

## Concepts Revisited

```text
HTTP
WebSockets
queues
events
cache
database
workers
streams
authorization
observability
API contracts
retries
idempotency
deployment
```

## Why This Chapter Matters

System design is where individual JavaScript concepts become system constraints.

A Promise is no longer merely:

```text
"something that resolves later."
```

It becomes a question of:

```text
concurrency
backpressure
failure
ownership
capacity
timeouts
```

A cache is no longer merely:

```text
"faster Map lookups."
```

It becomes:

```text
consistency
staleness
memory
invalidation
failure
cost
```

A Node.js process is no longer merely:

```text
"where JavaScript runs."
```

It becomes part of:

```text
capacity
deployment
scaling
observability
recovery
```

---

# 40. Track A — Core Theory

Master:

```text
requirements analysis
scaling
API design
data modeling
consistency
distributed failure
queues
caching
security architecture
observability
capacity planning
deployment
migration
cost
```

Deliverable:

```text
explain system behavior under normal and failure conditions
```

---

# 41. Track B — Implementation

For at least one problem, implement a representative slice:

```text
API
domain state
persistence
queue
worker
observability
```

Then exercise:

```text
failure
retry
duplicate
timeout
load
shutdown
```

Deliverable:

```text
working vertical slice
```

---

# 42. Track C — Interview / Reasoning

Practice:

```text
5-minute requirement clarification
10-minute architecture
10-minute deep dive
5-minute failure analysis
5-minute scaling discussion
```

Answer:

```text
why this
why not that
what changes at 10×
what breaks first
```

Deliverable:

```text
clear principal-level system design communication
```

---

# 43. System Design Mastery Gate

You may mark:

```text
[+] Completed
```

when:

```text
[ ] all 5 problems attempted
[ ] architecture diagrams created
[ ] scale assumptions documented
[ ] APIs defined
[ ] data ownership defined
[ ] consistency decisions documented
[ ] failure paths documented
[ ] security model documented
[ ] observability documented
[ ] scaling strategy documented
[ ] migration strategy documented
```

Mark:

```text
[*] Mastered
```

only when you can:

```text
[ ] clarify ambiguous requirements
[ ] estimate system scale
[ ] select architecture from constraints
[ ] define data ownership
[ ] define consistency boundaries
[ ] design reliable async workflows
[ ] reason about caching and queues
[ ] analyze failure and recovery
[ ] defend security architecture
[ ] design observability
[ ] estimate capacity
[ ] explain trade-offs
[ ] plan incremental evolution
[ ] defend the design under principal-level questioning
```

---

# 44. Assessment Completion Snapshot

```md
# Chapter 121 — Completion Snapshot

Status:
[ ] Not Started
[~] In Progress
[?] Needs Revision
[+] Completed
[*] Mastered

Score:
____ / 100

Primary Gaps:
-

Requirements:
[ ] weak
[ ] developing
[ ] strong
[ ] principal

Architecture:
[ ] weak
[ ] developing
[ ] strong
[ ] principal

Data / Consistency:
[ ] weak
[ ] developing
[ ] strong
[ ] principal

Reliability:
[ ] weak
[ ] developing
[ ] strong
[ ] principal

Scalability:
[ ] weak
[ ] developing
[ ] strong
[ ] principal

Security:
[ ] weak
[ ] developing
[ ] strong
[ ] principal

Observability:
[ ] weak
[ ] developing
[ ] strong
[ ] principal

Cost / Trade-Offs:
[ ] weak
[ ] developing
[ ] strong
[ ] principal

Migration:
[ ] weak
[ ] developing
[ ] strong
[ ] principal
```

---

# 45. Completion Criteria

```text
[ ] 5 system designs completed
[ ] 100 points scored
[ ] requirements are explicit
[ ] assumptions are explicit
[ ] scale is quantified
[ ] component ownership is explicit
[ ] APIs/interfaces are defined
[ ] source-of-truth decisions are explicit
[ ] consistency boundaries are explicit
[ ] failure handling is explicit
[ ] retries and idempotency are addressed
[ ] queues/caches have defined semantics
[ ] security boundaries are explicit
[ ] observability is designed
[ ] SLO/reliability thinking is included
[ ] scaling bottlenecks are identified
[ ] cost is considered
[ ] migration is planned
[ ] rejected alternatives are documented
[ ] principal trade-offs are defensible
```

---

# 46. Canonical System Design Mental Model

Use:

```text
requirements
→ constraints
→ scale
→ boundaries
→ APIs
→ data ownership
→ consistency
→ async/concurrency
→ failure
→ security
→ observability
→ scaling
→ deployment
→ migration
→ cost
```

Then ask:

```text
What must always be true?

What can be eventually consistent?

What can fail independently?

What can become a bottleneck?

What can be bounded?

What can be rebuilt?

What should be asynchronous?

What must remain synchronous?

How do we recover?

How do we know it is healthy?

How does the design change at 10×?
```

---

# 47. Final Principal Principle

> **A system design is successful when it remains understandable and correct as load, failure, teams, dependencies, and requirements change.**

The mature design loop is:

```text
clarify
→ quantify
→ model
→ choose boundaries
→ define state
→ define consistency
→ model failure
→ secure
→ observe
→ scale
→ migrate
→ revisit
```

The goal is not:

```text
maximum infrastructure
```

or:

```text
maximum abstraction.
```

The goal is:

```text
minimum architecture
that safely satisfies the real requirements
under real operational constraints.
```