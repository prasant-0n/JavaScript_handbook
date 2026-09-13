# Chapter 119 — Architecture Assessment — 10 Questions

> **JavaScript Mastery — Part XXI: Assessment**
>
> **Assessment:** 10 progressive architecture problems focused on boundaries, dependency direction, API design, state ownership, data access, reliability, scalability, observability, modularity, and principal-level engineering judgment.
>
> **Role perspective:** Principal JavaScript Engineer · Software Architect · Node.js Architect · Platform Engineer · Reliability Engineer · Security Engineer
>
> **Status:** `[ ] Not Started`
>
> **Core rule:** **Architecture is the set of decisions that control change, coupling, ownership, failure propagation, and operational cost.**

---

# 1. Assessment Mission

This assessment tests whether you can design and critique JavaScript systems beyond individual functions.

The target skill is:

```text
requirements
→ constraints
→ responsibilities
→ boundaries
→ dependencies
→ data ownership
→ failure boundaries
→ operational model
→ trade-offs
```

Architecture is not:

```text
folder structure
framework choice
pattern collection
microservices by default
classes everywhere
functions everywhere
```

A good architecture makes important system properties easier to preserve.

---

# 2. Scope Discipline

Separate:

```text
language-level concerns
runtime concerns
application architecture
deployment architecture
distributed-system concerns
```

Do not claim that a JavaScript language feature automatically solves an application architecture problem.

---

# 3. Assessment Scoring

Each question:

```text
0–10 points
```

Total:

```text
100 points
```

Recommended interpretation:

```text
90–100 → Principal-level architecture judgment
80–89  → Strong architecture reasoning
70–79  → Good foundation with targeted gaps
60–69  → Significant architectural gaps
<60     → Rebuild architecture fundamentals
```

---

# 4. Full-Credit Standard

A full-credit answer should identify:

```text
1. requirements
2. constraints
3. responsibilities
4. boundaries
5. dependency relationships
6. primary failure modes
7. scalability implications
8. observability requirements
9. trade-offs
10. migration/evolution strategy
```

Do not optimize for theoretical purity.

Optimize for a system that can be:

```text
understood
changed
operated
tested
scaled
secured
recovered
```

---

# 5. Question 1 — Layered API Design

A Node.js endpoint contains:

```js
app.post("/orders", async (req, res) => {
  const user = await db.users.findById(req.body.userId);

  if (!user) {
    return res.status(404).json({ error: "user not found" });
  }

  const total = req.body.items.reduce(
    (sum, item) => sum + item.price * item.quantity,
    0
  );

  const order = await db.orders.insert({
    userId: user.id,
    total
  });

  await email.send({
    to: user.email,
    subject: "Order created"
  });

  res.status(201).json(order);
});
```

### Task

Critique the architecture.

Identify responsibilities currently mixed together:

```text
transport
validation
business rules
persistence
side effects
```

Propose a better boundary model.

You may use:

```text
controller
application service
domain logic
repository
event/outbox
```

but justify each boundary rather than applying patterns mechanically.

---

# 6. Question 2 — Dependency Direction

Consider:

```text
Controller → Database
Controller → Payment Provider
Controller → Email Provider
Controller → Business Rules
```

### Task

Redesign dependency direction for a system where:

```text
business rules should remain testable
infrastructure providers may change
transport protocols may change
```

Explain:

```text
dependency inversion
ports/adapters
stable vs volatile dependencies
```

Then show which components should depend on abstractions and which should remain infrastructure-specific.

---

# 7. Question 3 — Shared Database Coupling

Two modules:

```text
Orders
Inventory
```

both directly modify each other's database tables.

Current behavior:

```text
orders creates order
orders decrements inventory
inventory updates order state
inventory writes audit data
```

### Symptom

A change to the inventory schema repeatedly breaks order functionality.

### Task

Identify the architectural coupling.

Design at least two alternatives:

```text
shared transactional boundary
application service orchestration
domain events
separate data ownership
```

Explain trade-offs around:

```text
consistency
transaction boundaries
failure handling
latency
operational complexity
team autonomy
```

Do not split services merely because modules have different names.

---

# 8. Question 4 — State Ownership

A frontend application has:

```text
Server response state
form state
URL state
local UI state
global application state
```

Developers store all of them in one global mutable object.

### Symptom

Changes in one screen unexpectedly affect another.

### Task

Design state ownership rules.

For each state category define:

```text
owner
lifetime
source of truth
readers
writers
persistence
synchronization
```

Then explain when centralized state is useful and when it creates unnecessary coupling.

---

# 9. Question 5 — API Contract Evolution

An API currently returns:

```json
{
  "id": 123,
  "name": "A",
  "status": "active"
}
```

A new client requires:

```json
{
  "id": 123,
  "displayName": "A",
  "state": "active",
  "metadata": {}
}
```

There are multiple existing clients.

### Task

Design an evolution strategy.

Consider:

```text
backward compatibility
versioning
additive change
deprecation
contract testing
migration
observability
```

Explain why changing:

```text
name → displayName
status → state
```

in place can create hidden breakage.

Then define criteria for:

```text
additive change
versioned endpoint
versioned media type
breaking release
```

---

# 10. Question 6 — Reliability Boundary

A checkout flow contains:

```text
create order
charge payment
reserve inventory
send confirmation
```

The current implementation uses:

```js
await createOrder();
await chargePayment();
await reserveInventory();
await sendConfirmation();
```

### Symptom

A payment succeeds, but inventory reservation fails.

### Task

Design a reliable workflow.

Identify:

```text
atomic vs non-atomic operations
failure boundaries
retryability
idempotency
compensation
state machine
```

Explain why a distributed workflow cannot simply assume that all four actions can share one transaction.

Propose a robust design using concepts such as:

```text
explicit state transitions
idempotency keys
outbox/event publication
compensation
reconciliation
```

---

# 11. Question 7 — Scaling Bottleneck

A Node.js service currently handles:

```text
2,000 requests/sec
```

The next target is:

```text
10,000 requests/sec
```

Profiling shows:

```text
CPU: 75%
DB: 70%
cache: 25%
network: moderate
event-loop delay: low
```

The team proposes:

```text
"Add more Node.js instances."
```

### Task

Evaluate the proposal.

Design a capacity analysis covering:

```text
application CPU
database capacity
connection pools
cache capacity
load distribution
horizontal scaling
statelessness
coordination/state
cost
```

Determine what measurements you need before choosing the scaling strategy.

Explain why scaling one tier can simply move the bottleneck to another tier.

---

# 12. Question 8 — Observability Architecture

A production platform currently logs:

```text
"request started"
"request finished"
```

There are dashboards for:

```text
CPU
memory
request count
```

An incident occurs where:

```text
p99 latency rises
only one tenant is affected
one dependency is intermittently slow
```

### Task

Redesign observability.

Specify:

```text
logs
metrics
traces
correlation IDs
tenant-safe dimensions
dependency timing
error classification
sampling
alerts
dashboards
```

Explain why:

```text
more logs
```

is not equivalent to:

```text
better observability
```

Also define what sensitive information should not be exposed casually.

---

# 13. Question 9 — Modular Monolith vs Microservices

A team proposes splitting a Node.js application into:

```text
users-service
orders-service
inventory-service
payments-service
notifications-service
```

The company has:

```text
6 engineers
1 production environment
limited operations support
moderate traffic
frequent domain changes
```

### Task

Choose between:

```text
modular monolith
microservices
hybrid
```

Your decision must evaluate:

```text
deployment independence
team topology
data ownership
failure isolation
latency
operational complexity
testing
observability
scaling
security
cost
```

Then define the architectural signals that would justify moving from one boundary model to another later.

---

# 14. Question 10 — Principal-Level Architecture Review

You inherit a large Node.js platform with:

```text
1. shared database
2. 100+ modules
3. global mutable configuration
4. synchronous CPU-heavy utilities
5. unbounded in-memory caches
6. direct third-party API calls from controllers
7. inconsistent error formats
8. no clear ownership of background jobs
9. duplicated authentication logic
10. weak production observability
11. frequent breaking API changes
12. deployment requires changing many modules together
```

Production symptoms:

```text
deployment risk is high
incident diagnosis is slow
teams block each other
memory usage is unpredictable
small changes have large blast radius
```

### Task

Treat this as a principal-level architecture assessment.

Do not propose “rewrite everything.”

Produce:

```text
current-state diagnosis
architectural hot spots
dependency map
ownership model
priority risks
target architecture
migration sequence
strangler boundaries where appropriate
test strategy
observability strategy
operational safeguards
success metrics
```

Prioritize using:

```text
business impact
failure frequency
blast radius
change coupling
operational risk
cost
migration complexity
future scalability
security
```

Your answer must distinguish:

```text
problems requiring structural change
```

from:

```text
problems solvable by local refactoring
```

---

# 15. Architecture Decision Record Template

Use this for major decisions:

```md
# ADR

## Context
-

## Problem
-

## Constraints
-

## Options
1.
2.
3.

## Decision
-

## Why
-

## Trade-offs
-

## Failure Modes
-

## Operational Impact
-

## Migration
-

## Revisit Trigger
-
```

Architecture without recorded reasoning becomes folklore.

---

# 16. Boundary Design Worksheet

For every module define:

```md
## Module
-

## Responsibility
-

## Owns
-

## Does Not Own
-

## Public Interface
-

## Dependencies
-

## Data Owned
-

## Failure Boundary
-

## Operational Signals
-

## Change Triggers
-
```

A boundary is stronger when its responsibility and ownership are clear.

---

# 17. Coupling Analysis

Classify coupling:

```text
A — structural coupling
B — data coupling
C — temporal coupling
D — deployment coupling
E — semantic coupling
F — runtime coupling
G — operational coupling
H — security coupling
```

Ask:

```text
What must change together?

What must deploy together?

What must fail together?

What data is jointly owned?

What assumptions are shared?
```

---

# 18. Cohesion Analysis

High cohesion means:

```text
related responsibilities change together
```

Low cohesion often appears as:

```text
"utils"
"helpers"
"common"
"shared"
```

containing unrelated behavior.

Do not judge cohesion from folder names alone.

Look at:

```text
change patterns
responsibility
data ownership
dependency direction
```

---

# 19. Dependency Graph

Model:

```text
A → B
```

as:

```text
A depends on B
```

Then inspect for:

```text
cycles
high fan-in
high fan-out
unstable dependencies
infrastructure leaking inward
shared mutable state
```

A useful architectural graph is not merely a module import graph.

Include:

```text
data
events
runtime
deployment
ownership
```

where relevant.

---

# 20. API Boundary Checklist

Every API should define:

```text
input contract
output contract
error contract
authentication
authorization
idempotency
timeouts
pagination
rate limits
versioning
observability
```

Do not design an API solely around:

```text
database tables
```

Design around client intent and stable capabilities.

---

# 21. Data Ownership Checklist

Ask:

```text
Who owns this data?

Who may mutate it?

Who validates invariants?

Who publishes changes?

Who consumes changes?

What is the source of truth?

What is replicated?

How is consistency maintained?

How is stale data handled?
```

Unclear ownership produces hidden coupling.

---

# 22. Failure Boundary Checklist

For every dependency ask:

```text
What can fail?

How long can it block?

Can it time out?

Can it retry?

Can retries duplicate effects?

Can callers cancel?

What happens when the dependency is unavailable?

What state remains after partial failure?

How is recovery performed?
```

Reliability is strongly shaped by failure boundaries.

---

# 23. Scalability Dimensions

Do not define scalability only as:

```text
more requests
```

Consider:

```text
traffic
data volume
tenant count
concurrency
CPU
memory
I/O
database load
queue depth
event volume
geographic distribution
deployment size
team size
```

A system may scale technically but fail organizationally or operationally.

---

# 24. Architecture Trade-Off Matrix

```md
| Decision | Benefit | Cost/Risk |
|---|---|---|
| Modular monolith | low operational complexity | shared deployment/runtime |
| Microservices | independent deployment/ownership | distributed-system complexity |
| Async events | loose temporal coupling | eventual consistency |
| Shared database | simple transactions | strong coupling |
| Separate data ownership | clear boundaries | coordination complexity |
| Cache | lower latency/load | staleness + invalidation |
| Worker threads | CPU isolation | coordination/memory cost |
| External queue | durable async work | infrastructure complexity |
| Strong consistency | simpler correctness | latency/coordination |
| Eventual consistency | availability/scalability options | more complex application logic |
```

Use evidence and requirements to choose.

---

# 25. Migration Strategy

When changing architecture, prefer:

```text
measure
→ define boundary
→ introduce seam
→ route incrementally
→ observe
→ migrate data/ownership
→ remove old path
```

Useful techniques include:

```text
strangler pattern
anti-corruption layer
facade
adapter
parallel run
feature flag
dual read
carefully controlled dual write
```

Every migration should define:

```text
rollback
observability
data correctness
cutover criteria
cleanup plan
```

---

# 26. Architecture Anti-Patterns

Recognize:

```text
microservices by default
shared mutable global state
generic "utils" dependency sink
controllers containing all business logic
database tables treated as module interfaces
event-driven everything
abstraction for every class
framework-driven boundaries
distributed transaction assumptions
retry without idempotency
cache without invalidation
service split without ownership
```

The problem is not that these words exist.

The problem is using them without a matching architectural need.

---

# 27. Principal Decision Framework

Evaluate architectural choices using:

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
Cost
Team Structure
Migration Risk
```

Do not optimize one dimension in isolation.

---

# 28. Architecture Misdiagnosis Taxonomy

Classify failed answers as:

```text
A — pattern chosen before problem definition
B — boundary without ownership
C — coupling overlooked
D — data ownership unclear
E — operational cost ignored
F — distributed systems complexity underestimated
G — failure boundaries ignored
H — migration ignored
I — observability ignored
J — scalability treated as only traffic
K — team topology ignored
L — security boundary ignored
M — architecture over-engineered
N — architecture under-specified
```

---

# 29. Architecture Review Questions

Before approving a significant design ask:

```text
1. What problem are we solving?
2. What constraints matter?
3. Who owns each responsibility?
4. What depends on what?
5. Where is the data source of truth?
6. What happens when each dependency fails?
7. What scales independently?
8. What deploys independently?
9. What must remain consistent?
10. What becomes eventually consistent?
11. How will we observe failures?
12. How will we migrate?
13. What is the rollback plan?
14. What is the long-term operational cost?
```

---

# 30. Retrieval Record

```md
# Chapter 119 — Architecture Assessment — Retrieval Record

## Attempt
- Date:
- Duration:
- Score:
- Percentage:
- Status before:
- Status after:

## Question Scores
- Q1:
- Q2:
- Q3:
- Q4:
- Q5:
- Q6:
- Q7:
- Q8:
- Q9:
- Q10:

## Strongest Areas
-

## Weakest Areas
-

## Boundary Design Gaps
-

## Data Ownership Gaps
-

## Reliability Gaps
-

## Scalability Gaps
-

## Observability Gaps
-

## Migration Gaps
-

## Questions Requiring Rework
-

## Next Review
-
```

---

# 31. Spaced Retrieval Schedule

### Day 0

Complete all 10.

### Day 1

Redo every question where:

```text
confidence < 4
```

### Day 3

Redesign:

```text
Q1–Q4
```

using explicit responsibilities and ownership.

### Day 7

Rework:

```text
Q5–Q7
```

as API/reliability/scaling decisions.

### Day 14

Rework:

```text
Q8–Q10
```

as principal architecture reviews.

### Day 21

Write three ADRs for real design decisions.

### Day 30

Create a dependency/data/ownership map for one production system.

---

# 32. Dependency Graph

```text
Chapters 01–30
        ↓
language + objects + errors + data structures
        ↓
Chapters 31–70
        ↓
async + browser + Node + networking + modules
        ↓
Chapters 71–90
        ↓
algorithms + paradigms + production architecture + testing
        ↓
Chapters 91–101
        ↓
modern language + compatibility + judgment
        ↓
Chapters 102–111
        ↓
production projects
        ↓
Chapter 112 — Conceptual Assessment
        ↓
Chapter 113 — Output Prediction
        ↓
Chapter 114 — Debugging Assessment
        ↓
Chapter 115 — Async / Event Loop Assessment
        ↓
Chapter 116 — Memory Assessment
        ↓
Chapter 117 — Performance Assessment
        ↓
Chapter 118 — Security Assessment
        ↓
Chapter 119 — Architecture Assessment
        ↓
Chapter 120 — Implementation Assessment
```

---

# 33. Concept Connections

## Depends On

```text
API design
database integration
reliability
observability
performance
security
modules
async systems
testing
algorithms
```

## Builds Toward

```text
implementation assessment
system design
principal engineering judgment
final principal JavaScript project
```

## Concepts Revisited

```text
module boundaries
dependency direction
Promises
queues
caches
databases
APIs
workers
observability
security boundaries
```

## Why This Chapter Matters

Architecture determines how easily a system can evolve.

A strong architecture answers:

```text
Who owns this?

Who may change this?

What depends on this?

What happens when this fails?

How does this scale?

How do we observe it?

How can we replace it?
```

---

# 34. Track A — Core Theory

Master:

```text
boundaries
cohesion
coupling
dependency direction
data ownership
API contracts
failure boundaries
consistency
scalability
observability
migration
architecture trade-offs
```

Deliverable:

```text
explain why a boundary exists
```

---

# 35. Track B — Implementation

Build:

```text
modular Node.js application
dependency ports/adapters
repository boundary
application service
event/outbox boundary
bounded cache
background job boundary
observability layer
API versioning mechanism
```

Every implementation should include:

```text
tests
errors
observability
ownership
migration considerations
```

---

# 36. Track C — Interview / Reasoning

Practice:

```text
"Monolith or microservices?"

"Where should this business rule live?"

"Who owns this data?"

"Where should retries happen?"

"How do you evolve this API?"

"How would you migrate without a rewrite?"

"What fails independently?"
```

Deliverable:

```text
defensible architectural decisions
```

---

# 37. Architecture Mastery Gate

You may mark:

```text
[+] Completed
```

when:

```text
[ ] all 10 questions attempted
[ ] responsibilities are explicit
[ ] boundaries are justified
[ ] dependencies are mapped
[ ] data ownership is defined
[ ] failure modes are analyzed
[ ] scaling implications are considered
[ ] migration is addressed
```

Mark:

```text
[*] Mastered
```

only when you can:

```text
[ ] design stable boundaries
[ ] recognize harmful coupling
[ ] define data ownership
[ ] reason about distributed failure
[ ] evolve APIs safely
[ ] choose architecture from constraints
[ ] design observable systems
[ ] plan incremental migrations
[ ] balance technical and organizational complexity
[ ] defend architecture at principal level
```

---

# 38. Assessment Completion Snapshot

```md
# Chapter 119 — Completion Snapshot

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

Boundary Design:
[ ] weak
[ ] developing
[ ] strong
[ ] principal

Dependency Direction:
[ ] weak
[ ] developing
[ ] strong
[ ] principal

Data Ownership:
[ ] weak
[ ] developing
[ ] strong
[ ] principal

Reliability Architecture:
[ ] weak
[ ] developing
[ ] strong
[ ] principal

Scalability:
[ ] weak
[ ] developing
[ ] strong
[ ] principal

Observability:
[ ] weak
[ ] developing
[ ] strong
[ ] principal

Migration:
[ ] weak
[ ] developing
[ ] strong
[ ] principal

Principal Judgment:
[ ] weak
[ ] developing
[ ] strong
[ ] principal
```

---

# 39. Completion Criteria

```text
[ ] 10 questions completed
[ ] 100 points scored
[ ] every major responsibility has an owner
[ ] boundaries are justified by change/failure/ownership needs
[ ] dependency direction is explicit
[ ] data source-of-truth decisions are explicit
[ ] reliability boundaries are explicit
[ ] scalability assumptions are measured
[ ] observability is designed
[ ] API evolution strategy exists
[ ] migration strategy exists
[ ] trade-offs are defended
[ ] organizational constraints are considered
```

---

# 40. Canonical Architecture Mental Model

Use:

```text
requirements
→ constraints
→ responsibilities
→ ownership
→ boundaries
→ dependencies
→ state/data
→ failure modes
→ scaling model
→ observability
→ evolution path
```

Then ask:

```text
What changes together?

What fails together?

What scales together?

What deploys together?

What data is owned together?

What should remain independent?
```

---

# 41. Final Principal Principle

> **Good architecture is not the most sophisticated structure. It is the smallest set of boundaries that preserves correctness, limits coupling, contains failure, and keeps future change affordable.**

The mature architecture loop is:

```text
understand
→ constrain
→ model
→ choose boundaries
→ make ownership explicit
→ analyze failures
→ measure operational cost
→ migrate incrementally
→ observe
→ revisit
```

The best architectural decision is the one that remains defensible when:

```text
traffic grows
teams grow
dependencies fail
requirements change
security threats evolve
and the system must be operated at 3 AM
```