# Chapter 89 — Code Review & Refactoring

> **Part XVI — Testing / Debugging**  
> **Status:** `[ ] Not Started`  
> **Depth Target:** Principal / Staff-level reasoning  
> **Primary Focus:** Reviewing JavaScript code for correctness, architecture, security, performance, reliability, and maintainability—and changing existing code safely without creating regressions.

---

# 1. Learning Objectives

By the end of this chapter, you should be able to:

1. Explain the purpose of code review.
2. Distinguish code review from static analysis, testing, linting, and debugging.
3. Review code for correctness before style.
4. Review architecture and dependency direction.
5. Identify hidden coupling.
6. Identify cohesion and responsibility problems.
7. Identify abstraction problems.
8. Identify common JavaScript code smells.
9. Distinguish a code smell from a defect.
10. Prioritize review findings by risk and impact.
11. Separate must-fix issues from suggestions.
12. Give actionable review feedback.
13. Review JavaScript APIs and contracts.
14. Review asynchronous code for race conditions and lifecycle bugs.
15. Review database integration for correctness and performance.
16. Review security boundaries.
17. Review memory and resource ownership.
18. Review error handling and failure propagation.
19. Review observability quality.
20. Review performance risks.
21. Review tests for false confidence.
22. Identify technical debt deliberately.
23. Distinguish intentional debt from accidental debt.
24. Understand refactoring as behavior-preserving change.
25. Refactor safely with characterization tests.
26. Refactor incrementally instead of rewriting unnecessarily.
27. Use seams to isolate difficult code.
28. Rename, extract, split, inline, and reorganize code safely.
29. Refactor asynchronous JavaScript without changing semantics accidentally.
30. Refactor module and dependency boundaries.
31. Refactor API contracts with compatibility in mind.
32. Refactor database access without changing transaction semantics.
33. Refactor stateful code without introducing shared-state bugs.
34. Review and refactor performance-sensitive code responsibly.
35. Handle large-scale refactoring across teams.
36. Use automation to reduce repetitive refactoring risk.
37. Design code-review standards for organizations.
38. Measure review effectiveness without optimizing for comment count.
39. Build a disciplined refactoring workflow.
40. Defend review and refactoring decisions at principal-engineer level.

---

# 2. Prerequisites

You should understand:

- JavaScript syntax and semantics.
- Functions, closures, objects, classes, modules.
- Promises, async/await, event loop, cancellation.
- Errors and cleanup.
- Data structures and algorithms.
- Node.js runtime behavior.
- APIs and databases.
- Production architecture.
- Observability.
- Reliability.
- Performance.
- Testing.
- Deterministic async testing.
- Debugging methodology.

Recommended prior chapters:

- **9–21** — Functions / Scope / Execution / Objects
- **22–28** — Data Structures
- **29–40** — Errors / Async / Concurrency / Streaming
- **45–48** — Memory / GC / Engine / V8
- **58–70** — Node.js / Modules / Packaging / Tooling
- **71–77** — DSA / Paradigms / Composition / Patterns
- **78–85** — Production Architecture / API / Database / Observability / Reliability / Performance
- **86** — Testing
- **87** — Deterministic Async Testing
- **88** — Debugging Methodology

---

# 3. What Is It?

## Code review

Code review is the deliberate examination of a change by another engineer or group of engineers before or after integration.

Its purpose is to identify:

```text
incorrect behavior
security defects
architecture problems
maintainability issues
performance risks
reliability risks
compatibility issues
operational problems
```

It is not primarily:

```text
formatting correction
personal preference
style enforcement
```

---

## Refactoring

Refactoring is changing the internal structure of code while preserving externally required behavior.

Conceptually:

```text
Before:
hard-to-change implementation

        ↓

Refactor

        ↓

After:
easier-to-understand implementation

Required behavior preserved
```

The preservation condition is crucial.

---

# 4. Why Does It Exist?

Code accumulates history.

A system may begin as:

```text
10 files
1 developer
1 customer
```

and become:

```text
1,000 files
20 developers
100 integrations
```

Over time:

```text
dependencies spread
responsibilities merge
APIs grow
tests become expensive
```

Refactoring restores structure.

Code review provides a second perspective before new complexity becomes permanent.

---

# 5. Mental Model

Review code across multiple dimensions:

```text
                Correctness
                     │
       ┌─────────────┼─────────────┐
       │             │             │
   Security      Reliability   Maintainability
       │             │             │
       └─────────────┼─────────────┘
                     │
              Performance
                     │
              Operability
                     │
              Evolvability
```

A review should not stop at:

```text
"Does this compile?"
```

Ask:

```text
Does it behave correctly?
Does it fail safely?
Can it be understood?
Can it be changed?
Can it be observed?
Can it scale?
Can it be secured?
```

---

# 6. Core Rules

## Rule 1 — Correctness before style

Review order:

```text
correctness
security
reliability
performance
architecture
maintainability
style
```

Do not spend 20 comments on naming while a race condition exists.

---

## Rule 2 — Review the change in context

A function can be correct locally and wrong globally.

Inspect:

```text
callers
consumers
dependencies
tests
schema
API contract
deployment
```

---

## Rule 3 — Prefer evidence over preference

Bad:

> “I would write this differently.”

Better:

> “This can cause duplicate payment after timeout because the operation is retried without idempotency.”

---

## Rule 4 — Comments should teach or protect

Useful comments explain:

```text
risk
invariant
trade-off
non-obvious constraint
```

Not:

```text
what the code literally does
```

---

## Rule 5 — Review boundaries, not only lines

Ask:

```text
who owns this state?
who owns this decision?
who owns this side effect?
```

---

## Rule 6 — Refactor in small steps

Prefer:

```text
characterize
→ change one structure
→ test
→ commit
→ repeat
```

over:

```text
rewrite everything
```

---

## Rule 7 — Preserve behavior deliberately

Behavior includes:

```text
output
errors
timing
side effects
mutation
ordering
resource usage
compatibility
```

Do not preserve only the happy-path result.

---

## Rule 8 — Make the dependency graph simpler

Good refactoring often reduces:

```text
coupling
cycles
shared mutable state
API surface
```

---

## Rule 9 — Delete complexity when possible

The safest abstraction is sometimes:

```text
removed abstraction
```

---

## Rule 10 — Refactor because of a reason

Possible reasons:

```text
change is too expensive
bug risk is too high
performance is inadequate
testing is difficult
security boundary is unclear
ownership is unclear
```

Do not refactor merely because code is old.

---

# 7. Syntax

There is no dedicated JavaScript syntax for code review or refactoring.

The main tools are:

```text
module boundaries
functions
objects
classes
tests
types
static analysis
profilers
dependency graphs
```

---

## Example — Extract function

Before:

```js
function createOrder(input) {
  // validate
  // calculate total
  // save
  // send event
  // respond
}
```

After:

```js
function validateOrder(input) {}
function calculateTotal(input) {}
function persistOrder(order) {}
function publishOrderCreated(order) {}

function createOrder(input) {
  validateOrder(input);

  const order = calculateTotal(input);

  persistOrder(order);
  publishOrderCreated(order);

  return order;
}
```

Extraction is useful only when the resulting boundaries communicate meaningful responsibilities.

---

# 8. Basic Examples

## Example 1 — Duplicate logic

Before:

```js
const totalA = price * quantity;
const totalB = otherPrice * otherQuantity;
```

Two independent calculations may be fine.

Do not extract a generic:

```js
calculateSomething(a, b)
```

just because two lines look similar.

Similarity is not always shared abstraction.

---

## Example 2 — Giant function

Before:

```js
async function checkout() {
  // 250 lines
}
```

Potential decomposition:

```text
authorize
validateInventory
calculateTotal
reserveInventory
createOrder
recordPaymentIntent
publishEvent
```

The correct decomposition follows responsibilities and invariants.

---

# 9. Execution Walkthrough

A pull request changes:

```text
payment retry logic
```

A high-quality review:

```text
read requirement
  ↓
inspect changed files
  ↓
inspect callers
  ↓
inspect tests
  ↓
inspect retry policy
  ↓
inspect idempotency
  ↓
inspect timeout
  ↓
inspect error mapping
  ↓
inspect observability
  ↓
run targeted tests
  ↓
review production implications
```

The review is a reasoning process, not a line-count exercise.

---

# 10. Internal Mechanics

## 10.1 Cohesion

Cohesion asks:

> How strongly do the responsibilities inside this module belong together?

High cohesion:

```text
orders/
  create
  cancel
  confirm
```

when all revolve around the order domain.

Low cohesion:

```text
utils/
  database
  email
  payment
  date
  authorization
```

---

## 10.2 Coupling

Coupling asks:

> How much does one component depend on another?

High coupling:

```text
controller
 → database schema
 → payment provider
 → cache
 → email
```

Low coupling:

```text
controller
 → use case
 → domain / ports
```

with explicit adapters.

---

## 10.3 Stable dependencies

A module that imports:

```text
20 changing modules
```

is difficult to evolve.

A module depending on:

```text
2 stable interfaces
```

is easier to reason about.

---

# 11. ECMAScript / Specification Semantics

Code review must separate:

```text
language semantics
runtime behavior
library behavior
application behavior
```

Example:

```js
Array.prototype.map(...)
```

has ECMAScript-defined semantics.

But:

```text
performance cost
V8 optimization
database behavior
HTTP behavior
```

are not simply determined by the ECMAScript specification.

During review, state the level of guarantee:

```text
ECMAScript guarantees
Node guarantees
V8-specific observation
library contract
application policy
```

A principal review does not turn implementation observations into language laws.

---

# 12. Advanced Behavior

## 12.1 Code smells

Common smells:

```text
giant function
giant class
feature envy
shotgun surgery
primitive obsession
boolean parameter explosion
deep nesting
duplicate logic
hidden global state
circular dependencies
dead code
leaky abstraction
god object
long parameter list
inappropriate intimacy
```

A smell is a signal.

It is not automatically a defect.

---

## 12.2 God object

```js
class ApplicationManager {
  users() {}
  orders() {}
  payments() {}
  inventory() {}
  emails() {}
}
```

Likely problems:

```text
low cohesion
high coupling
difficult ownership
large test surface
```

---

## 12.3 Shotgun surgery

One simple change requires:

```text
15 files
```

because the concept is scattered.

Potential response:

```text
co-locate behavior
establish ownership
extract cohesive module
```

---

## 12.4 Feature envy

A function in module A manipulates module B's internals constantly.

Possible solution:

```text
move behavior closer to data/owner
```

But not every cross-module interaction is a smell.

---

## 12.5 Primitive obsession

Instead of:

```js
currency = "INR";
amount = 1000;
```

a domain may require:

```js
Money
```

when operations need:

```text
currency safety
rounding
comparison
arithmetic
```

Do not create a value object merely because design literature contains the pattern.

---

# 13. Edge Cases

## 13.1 Refactoring can change timing

Before:

```js
await a();
await b();
```

After:

```js
await Promise.all([
  a(),
  b(),
]);
```

This is not a harmless refactor if:

```text
b depends on a
resources are limited
ordering matters
```

---

## 13.2 Refactoring can change errors

Before:

```js
throw error;
```

After:

```js
throw new Error("operation failed");
```

The new code may destroy:

```text
error code
cause
stack
type
metadata
```

---

## 13.3 Refactoring can change mutation

Before:

```js
options.enabled = true;
```

After:

```js
return {
  ...options,
  enabled: true,
};
```

Callers depending on mutation may observe different behavior.

---

## 13.4 Refactoring can change memory

Before:

```js
for (...) {
  process(item);
}
```

After:

```js
const results = items.map(process);
```

The latter can retain many results in memory.

---

## 13.5 Refactoring can change API compatibility

Renaming:

```js
error.code
```

can be a breaking change even if the internal logic is “cleaner.”

---

# 14. Common Misconceptions

### “Code review is about style.”

No. Style is one small part.

### “Refactoring means making code shorter.”

No. Refactoring improves structure while preserving required behavior.

### “More abstractions mean cleaner code.”

No. Abstractions can introduce indirection and coupling.

### “Duplicated code is always bad.”

Sometimes duplication protects independent concepts from premature coupling.

### “One class per concept is best.”

Not necessarily.

### “Every function should be pure.”

No. Side effects are necessary; they should be controlled and visible.

### “Tests make refactoring safe.”

Only to the extent that tests cover the behavior being changed.

### “A green test suite means refactoring preserved performance.”

Not necessarily.

### “A rewrite is a large refactor.”

A rewrite often changes behavior, contracts, and operational characteristics simultaneously.

---

# 15. Common Mistakes

## Mistake 1 — Refactoring without tests

You cannot reliably tell whether behavior changed.

---

## Mistake 2 — Changing structure and semantics together

For example:

```text
rename API
change error behavior
change concurrency
change database
```

in one commit.

---

## Mistake 3 — Review by line count

Large changes are not automatically bad.

Small changes can be dangerous.

---

## Mistake 4 — Nitpicking

Low-value comments obscure important findings.

---

## Mistake 5 — Vague feedback

Bad:

> “This isn't clean.”

Better:

> “This controller now decides payment retry policy. Move retry policy into the payment adapter/use case so all callers share the same semantics.”

---

## Mistake 6 — Reviewer ownership failure

Reviewer assumes:

```text
someone else checked security
```

Nobody did.

---

## Mistake 7 — Refactor everything

Large rewrites increase:

```text
unknown interactions
review difficulty
rollback complexity
```

---

# 16. Comparison With Related Concepts

| Activity | Primary Goal | Typical Output |
|---|---|---|
| Code review | Detect risk before merge | Review findings |
| Linting | Enforce mechanical rules | Diagnostics |
| Static analysis | Find suspicious structures | Warnings/findings |
| Testing | Validate behavior | Test evidence |
| Debugging | Explain unexpected behavior | Root-cause hypothesis |
| Refactoring | Improve internal structure | Behavior-preserving change |
| Rewrite | Replace implementation | New system/component |
| Architecture review | Evaluate boundaries/trade-offs | Decision |
| Pair programming | Continuous shared reasoning | Joint implementation |
| Design review | Validate design before coding | Architecture/design decision |

---


---

# 16A. Review the Change, Not Just the Diff

A diff shows changed lines.

A review should reconstruct:

```text
intent
→ behavior
→ dependencies
→ state
→ failure modes
→ operational impact
```

Start with:

```text
Why was this change made?
What user/business behavior should change?
What behavior must not change?
```

Then inspect:

```text
changed code
callers
tests
contracts
configuration
schema
deployment
```

A 20-line change can have a 20-service impact.

---

# 16B. Review Context Before Implementation

Read in this order when possible:

```text
ticket/requirement
→ public API
→ changed implementation
→ callers
→ tests
→ adjacent modules
→ operational configuration
```

This prevents a common failure:

```text
review local code
without understanding global responsibility
```

---

# 16C. Review by Risk

A useful risk-first ordering:

```text
1. data corruption/loss
2. security
3. correctness
4. reliability
5. concurrency
6. compatibility
7. performance
8. maintainability
9. style
```

This ordering is not universal, but it is a useful default.

For a security-sensitive change:

```text
security
```

may move to the top.

For a critical financial workflow:

```text
correctness/idempotency
```

may dominate.

---

# 16D. Review Finding Quality

A strong finding contains:

```text
Location
Problem
Mechanism
Impact
Recommendation
```

Example:

```text
This retry is performed after a request timeout, but the payment operation
has no idempotency key. A successful remote charge followed by a local timeout
can therefore produce a duplicate charge on retry. Make the operation
idempotent before enabling retries.
```

This is stronger than:

```text
Retry logic is dangerous.
```

---

# 16E. Blocking Versus Non-Blocking

Use blocking feedback when:

```text
the code is incorrect
security is compromised
data can be corrupted
reliability contract is violated
compatibility is broken
required behavior is missing
```

Use non-blocking feedback for:

```text
naming improvement
future refactoring
small readability enhancement
alternative implementation
```

A review becomes unhealthy when every comment is treated as equally urgent.

---

# 16F. Positive Review Feedback

Not all review feedback needs to identify a problem.

Useful positive feedback:

```text
This boundary keeps payment-provider details out of the domain model.
Good separation.
```

or:

```text
The test constructs the concurrency race explicitly instead of relying on timing.
```

Positive signals reinforce architecture and review standards.

---

# 16G. Review Question Hierarchy

Ask in sequence:

### Level 1 — Does it work?

```text
happy path
edge cases
failure
```

### Level 2 — Does it remain correct under concurrency?

```text
duplicate
race
ordering
transactions
```

### Level 3 — Does it fail safely?

```text
timeout
dependency outage
restart
partial failure
```

### Level 4 — Can we operate it?

```text
logs
metrics
traces
alerts
rollback
```

### Level 5 — Can we evolve it?

```text
API
schema
dependencies
module boundary
```

---

# 16H. Requirements Traceability

For important changes, map:

```text
requirement
→ implementation
→ test
→ operational signal
```

Example:

```text
Requirement:
User cannot refund an unpaid order.

Implementation:
domain state transition check.

Test:
unpaid order refund rejected.

Operational signal:
refund conflict metric.
```

This makes review more rigorous.

---

# 16I. Refactoring Safety Model

Before refactoring, identify:

```text
observable inputs
observable outputs
side effects
errors
timing
mutation
resource usage
external interactions
```

Then define which must remain unchanged.

Example:

```text
Input:
order

Output:
order response

Side effects:
payment provider
database

Timing:
not contractual

Error:
ORDER_CONFLICT stable

Concurrency:
duplicate requests must remain safe
```

This is a refactoring contract.

---

# 16J. Seams

A seam is a place where behavior can be altered or isolated without rewriting the entire system.

Examples:

```text
function parameter
module export
repository interface
HTTP client
clock
filesystem adapter
environment configuration
message consumer
```

Example:

```js
function createService({
  repository,
  clock,
}) {
  // ...
}
```

The service now has seams for:

```text
persistence
time
```

Seams improve:

```text
testing
migration
debugging
replacement
```

But unnecessary seams add indirection.

---

# 16K. Branch by Abstraction

When replacing a large dependency:

```text
old implementation
        ↓
abstraction
        ↓
new implementation
```

Migration:

```text
introduce abstraction
→ connect old implementation
→ verify
→ introduce new implementation
→ switch consumers
→ remove old implementation
```

This can be safer than:

```text
remove old
→ implement new
```

because both versions can coexist during transition.

---

# 16L. Parallel Refactoring

Large teams often need refactoring without blocking feature work.

Techniques:

```text
small independent commits
feature flags
branch-by-abstraction
strangler migration
compatibility wrappers
incremental ownership transfer
```

Avoid:

```text
one giant six-week branch
```

when changes can be integrated incrementally.

---

# 16M. API Refactoring

Suppose:

```js
createClient({
  endpoint,
});
```

must become:

```js
createClient({
  baseUrl,
});
```

Compatibility strategy:

```text
accept endpoint temporarily
→ internally normalize
→ warn/deprecate
→ migrate consumers
→ remove endpoint
```

For a public library, consider:

```text
major version
```

if compatibility policy requires it.

For an internal service, coordinated migration may be possible.

---

# 16N. Error Refactoring

Before:

```js
throw new PaymentError("failed");
```

After:

```js
throw new PaymentProviderTimeoutError(
  "provider timed out",
);
```

Potentially useful.

But review:

```text
does instanceof behavior change?
does error.code change?
does serialization change?
does retry classification change?
does logging change?
does cause chain survive?
```

An error refactor can accidentally change resilience behavior.

---

# 16O. Async Refactoring

Potentially dangerous transformation:

```js
await a();
await b();
```

to:

```js
await Promise.all([
  a(),
  b(),
]);
```

Review:

```text
dependency
ordering
side effects
concurrency
resource usage
failure interaction
```

Another dangerous change:

```js
return promise;
```

to:

```js
return await promise;
```

which can affect:

```text
stack context
try/catch boundaries
timing details
```

The exact observable behavior depends on the surrounding code and runtime semantics.

---

# 16P. Error Handling Refactoring

Before:

```js
try {
  await operation();
} catch (error) {
  return fallback();
}
```

After:

```js
try {
  await operation();
} catch (error) {
  logger.error(error);
  throw error;
}
```

This may be “cleaner” but changes behavior.

Review:

```text
fallback contract
error propagation
availability
user experience
retry behavior
```

---

# 16Q. State Refactoring

Before:

```js
let currentUser;
```

After:

```js
function createSession(user) {
  return {
    getUser() {
      return user;
    },
  };
}
```

This can improve isolation.

But migration may expose consumers relying on:

```text
global state
```

Search for:

```text
mutation
reads
initialization order
cleanup
```

before changing ownership.

---

# 16R. Database Refactoring

Potentially dangerous:

```text
repository.save(order)
```

used to run inside a transaction.

After refactor:

```text
repository.save(order)
```

opens its own transaction.

The application workflow may now lose atomicity.

Review:

```text
transaction ownership
connection identity
lock ordering
isolation
rollback
retry
```

---

# 16S. Performance-Preserving Refactoring

A cleaner implementation may:

```text
allocate more
query more
serialize more
create more objects
increase fan-out
```

Measure before and after.

For important hot paths record:

```text
latency
CPU
memory
GC
I/O
database
allocation
```

A refactor is not automatically neutral just because output matches.

---

# 16T. Security-Preserving Refactoring

When moving code, track security boundaries explicitly.

Example:

```text
controller
→ authorization
→ domain
```

becomes:

```text
controller
→ domain
```

If authorization was unintentionally removed, another caller may bypass it.

For every refactor, identify:

```text
trust boundary
authentication
authorization
tenant
input validation
secret handling
```

---

# 16U. Characterization Test Strategy

For legacy code whose specification is unclear:

```text
1. identify observed behavior
2. write characterization test
3. capture success/failure/edge cases
4. refactor
5. compare behavior
6. only then improve semantics
```

Do not simultaneously:

```text
refactor
change business policy
```

unless the change is deliberate and separately tested.

---

# 16V. Golden-Master Strategy

For complex deterministic output:

```text
input fixture
→ old system output
→ save reference
```

After refactor:

```text
same fixture
→ new output
→ compare
```

Useful for:

```text
parsers
renderers
serializers
reports
CLI output
protocol encoders
```

Review differences rather than automatically accepting them.

---

# 16W. Refactoring with Feature Flags

A feature flag can separate:

```text
deployment
```

from:

```text
activation
```

Example:

```js
if (flags.newPricing) {
  return newPricing(input);
}

return oldPricing(input);
```

This can reduce migration risk.

But flag systems create:

```text
branches
state
operational complexity
```

Remove temporary flags after migration.

---

# 16X. Technical Debt

Technical debt is not simply:

```text
bad code
```

A useful model:

```text
current shortcut
→ future cost
```

Examples:

```text
duplicated logic
→ inconsistent fixes

shared global
→ difficult testing

missing abstraction
→ expensive replacement

premature abstraction
→ future coupling
```

Not all debt must be repaid immediately.

Prioritize by:

```text
change frequency
risk
interest cost
blast radius
```

---

# 16Y. Debt Interest

A small architectural problem in a stable module:

```text
low interest
```

A small problem in a high-change module:

```text
high interest
```

Therefore debt management should ask:

```text
How often do we touch this?
How expensive is each change?
What failures does it enable?
```

---

# 16Z. Review Through Change Frequency

A useful heuristic:

```text
high-change + high-risk
→ invest heavily

high-change + low-risk
→ optimize developer speed

low-change + high-risk
→ protect correctness/security

low-change + low-risk
→ avoid unnecessary refactoring
```

This prevents “clean everything” programs.

---

# 17AA. Review the Test Change Too

A code change can look safe while its tests weaken.

Check:

```text
assertion strength
coverage of failure path
mock realism
test isolation
race coverage
cleanup
```

A suspicious diff:

```diff
-await assert.rejects(...)
+await operation()
```

may silently remove the most important assertion.

---

# 17AB. Test Mutation as Review Evidence

When a module is high-risk, mutation testing can reveal whether the suite detects meaningful behavioral changes.

Useful questions:

```text
Would changing > to >= fail a test?
Would removing authorization fail?
Would skipping transaction rollback fail?
Would changing status code fail?
```

You do not need mutation testing on every module.

---

# 17AC. Automated Refactoring

Use tools for mechanical transformations:

```text
rename
import path changes
syntax-aware transformations
codemods
formatting
dependency graph checks
```

Automation reduces typo risk.

But automated transformation can still preserve the wrong semantics.

Always verify:

```text
tests
contract
runtime
behavior
```

---

# 17AD. Codemods

A codemod can transform:

```js
oldFunction(a, b)
```

to:

```js
newFunction({
  a,
  b,
})
```

This is useful for large migrations.

A production codemod should include:

```text
before examples
after examples
scope
exceptions
test suite
dry run
rollback
```

---

# 17AE. Dependency Refactoring

Replacing:

```text
library A
```

with:

```text
library B
```

requires review of:

```text
behavior
bundle
startup
memory
security
license
API
transitive dependencies
runtime support
```

Do not judge dependency replacement only by API similarity.

---

# 17AF. Module Boundary Refactoring

Before moving:

```text
orders.js
→ orders/
```

check:

```text
imports
deep imports
dynamic imports
package exports
tests
build tooling
source maps
```

Search both:

```text
static references
dynamic references
```

---

# 17AG. Refactoring Checklist for Large Modules

Before extracting a module:

```text
What state is shared?
What functions mutate it?
What imports are required?
What exports are public?
What tests assume identity?
What error types escape?
What async work is retained?
What timers/listeners exist?
What side effects occur?
What configuration is required?
```

Then define the new boundary.

---

# 17AH. Removing an Abstraction

Refactoring is not only adding structure.

Sometimes remove:

```text
wrapper
factory
service class
repository
utility
adapter
```

when it no longer protects a meaningful boundary.

Test:

```text
behavior preserved
dependency graph simplified
```

---

# 17AI. Architecture Fitness Checks

Organizations can automate architectural rules:

```text
domain cannot import infrastructure
feature A cannot import feature B internals
public package cannot import internal package
deprecated module cannot gain new consumers
```

Possible mechanisms:

```text
dependency graph analysis
lint rules
build checks
CI architecture tests
```

Architecture becomes enforceable rather than aspirational.

---

# 17AJ. Review Templates

A review template can include:

```text
Intent:
Risk:
Correctness:
Security:
Reliability:
Performance:
Memory:
API:
Database:
Async:
Observability:
Tests:
Migration:
Rollback:
```

Do not force every review to fill every field mechanically.

Use it as a thinking aid.

---

# 17AK. Review Anti-Patterns

Avoid:

```text
comment every line
style policing
rewrite-by-review
personal preference
premature optimization
architecture dogma
rubber-stamp approval
reviewing without running tests
ignoring production context
```

---

# 17AL. Review Rubber-Stamp Detection

Signs:

```text
huge PR reviewed in minutes
no meaningful comments
no test execution
no context questions
repeated regressions
```

The solution is not “add more reviewers.”

Improve:

```text
PR size
ownership
automation
review checklist
test evidence
architecture visibility
```

---

# 17AM. PR Size

Smaller changes improve:

```text
review comprehension
failure localization
rollback
merge confidence
```

But splitting a logically atomic change too much can also increase coordination cost.

The goal is:

```text
small enough to reason about
large enough to remain coherent
```

---

# 17AN. Commit Structure

Useful refactoring sequence:

```text
commit 1:
characterization tests

commit 2:
extract dependency

commit 3:
move behavior

commit 4:
remove old path
```

This creates a reviewable transformation trail.

---

# 17AO. Refactoring and Git History

Meaningful history can help future debugging.

Avoid combining:

```text
format entire repository
+
rename module
+
change behavior
```

in one commit.

Separate mechanical noise from semantic change.

---

# 17AP. Review Build/Artifact Effects

A source change can affect:

```text
bundle size
tree shaking
source maps
exports
runtime target
polyfills
```

Review generated artifacts when relevant.

A clean source refactor can accidentally produce a larger or incompatible artifact.

---

# 17AQ. Review Configuration Changes

A small configuration change can have enormous impact:

```json
{
  "retryAttempts": 10
}
```

can multiply dependency traffic.

Review config with the same seriousness as code.

---

# 17AR. Refactoring Operational Behavior

Examples:

```text
log level changed
metric renamed
trace sampling changed
health check changed
timeout changed
shutdown timeout changed
```

These may affect operations even if application output remains identical.

Operational behavior is part of production behavior.

---

# 17AS. Refactoring and SLOs

A refactor should not be considered complete merely because:

```text
tests pass
```

For critical paths verify:

```text
latency
error rate
resource usage
SLO
```

before/after where practical.

---

# 17AT. Review Decision Record

For risky changes, record:

```text
Decision:
Alternatives:
Evidence:
Risk:
Mitigation:
Verification:
Rollback:
```

This converts a review from:

```text
approval
```

into:

```text
engineering decision
```

---

# 17AU. Principal Review Heuristic

For every non-trivial change ask:

```text
What becomes easier?
What becomes harder?
What becomes safer?
What becomes more coupled?
What becomes more expensive?
What becomes less observable?
What new failure mode exists?
What old failure mode disappears?
What contract changes?
What is the rollback?
```

This reveals trade-offs hidden by local code quality.

---

# 17AV. Principal Refactoring Heuristic

Before refactoring:

```text
What is the problem?
How often does it hurt?
What evidence do we have?
What behavior must remain?
What boundary should improve?
What is the smallest safe transformation?
How will we verify it?
```

After refactoring:

```text
Did the intended structural problem improve?
Did behavior remain correct?
Did performance change?
Did memory change?
Did security change?
Did operational behavior change?
Did complexity actually decrease?
```


# 17. Performance Considerations

Code review should identify:

```text
N+1 queries
unbounded Promise concurrency
large allocations
serialization
hot loops
blocking operations
unbounded caching
excessive logging
unnecessary network calls
```

Do not review performance by syntax alone.

---

## Example

Bad:

```js
const results = await Promise.all(
  items.map(item => callRemote(item)),
);
```

Questions:

```text
How many items?
What is the remote capacity?
What is the memory cost?
What happens on 50,000 items?
What is the retry policy?
What if one fails?
```

The syntax is valid.

The architecture may not be.

---

# 18. Memory Considerations

Look for:

```text
retained closures
global caches
unbounded arrays
listeners
timers
sockets
large buffers
queued promises
duplicated object graphs
```

A refactor can accidentally increase memory.

Example:

```js
const output = items.map(transform);
```

may retain the complete output.

Streaming:

```js
for (const item of items) {
  consume(transform(item));
}
```

may reduce peak memory.

The right choice depends on required semantics.

---

# 19. Security Considerations

Review for:

```text
authorization
authentication
tenant isolation
input validation
injection
secret handling
SSRF
unsafe deserialization
dependency usage
logging
error leakage
```

---

## Security review question

Do not ask only:

```text
Can the request succeed?
```

Ask:

```text
Can an unauthorized request succeed?
Can a tenant access another tenant?
Can attacker input reach a privileged operation?
Can secrets escape through logs/errors?
```

---

# 20. Production Usage

## 20.1 Review priority

A practical priority system:

```text
P0 — correctness/security/outage/data loss
P1 — likely production defect
P2 — meaningful maintainability/performance issue
P3 — improvement suggestion
P4 — optional style preference
```

Organizations may use different labels.

The important property is:

```text
risk is explicit
```

---

## 20.2 Review checklist

### Correctness

```text
requirements
edge cases
state transitions
errors
concurrency
```

### Security

```text
auth
authorization
input
secrets
tenant
```

### Reliability

```text
timeouts
retries
idempotency
cleanup
shutdown
```

### Performance

```text
queries
network
CPU
memory
concurrency
```

### Maintainability

```text
cohesion
coupling
naming
complexity
testability
```

### Operations

```text
logs
metrics
traces
rollback
migrations
```

---

## 20.3 Comment quality

Useful review comment:

```text
This retry can duplicate a payment after a timeout.
Make the request idempotent or persist an idempotency key before allowing retries.
```

It contains:

```text
problem
mechanism
risk
direction
```

---

## 20.4 Separate blocking and non-blocking feedback

Blocking:

```text
must fix before merge
```

Non-blocking:

```text
future improvement
```

This preserves review clarity.

---

## 20.5 Refactoring workflow

```text
1. Establish baseline
2. Add characterization tests
3. Identify seam
4. Make smallest structural change
5. Run tests
6. Review diff
7. Commit
8. Repeat
```

---

## 20.6 Characterization test

Useful for legacy code:

```text
record current behavior
```

even if the behavior is not ideal.

Example:

```js
test("preserves current parser behavior", () => {
  assert.deepEqual(
    parseLegacy(input),
    expectedLegacyResult,
  );
});
```

Then refactor.

Afterward:

```text
same intended behavior
```

is easier to verify.

---

## 20.7 Golden-master technique

For complex output:

```text
input
→ current output
→ capture
```

then refactor and compare.

Useful for:

```text
serializers
parsers
CLI output
report generators
```

Do not blindly approve every golden diff.

---

## 20.8 Strangler refactoring

Instead of:

```text
replace entire module
```

use:

```text
old path
   ↓
new implementation for selected capability
   ↓
expand coverage
   ↓
remove old path
```

This reduces migration risk.

---

# 21. Implementation From Scratch

Build a refactoring laboratory.

## Stage 1 — Guided

Start with:

```js
async function checkout(userId, items) {
  const user = await db.users.findById(userId);

  if (!user) {
    throw new Error("User not found");
  }

  let total = 0;

  for (const item of items) {
    const product = await db.products.findById(item.productId);
    total += product.price * item.quantity;
  }

  const payment = await charge(user, total);

  const order = await db.orders.insert({
    userId,
    total,
    paymentId: payment.id,
  });

  await sendEmail(user.email);

  return order;
}
```

---

## Stage 2 — Characterize

Create tests for:

```text
missing user
missing product
zero/negative quantity
payment failure
database failure
email failure
successful checkout
```

---

## Stage 3 — Extract domain behavior

Separate:

```text
calculateTotal
validateCheckout
```

from:

```text
database
payment
email
```

---

## Stage 4 — Fix data-access structure

Extract:

```text
userRepository
productRepository
orderRepository
```

only where they provide useful boundaries.

---

## Stage 5 — Fix N+1

Replace:

```text
one query per product
```

with a batch/query strategy.

---

## Stage 6 — Make failure semantics explicit

Design:

```text
payment success
database failure
```

and:

```text
database success
email failure
```

separately.

---

## Stage 7 — Add observability

Record:

```text
checkout started
checkout failed
payment duration
database duration
```

---

## Stage 8 — Production-grade refactor

Final structure:

```text
checkout/
  domain/
    pricing.js
    validation.js

  application/
    checkout.js

  infrastructure/
    user-repository.js
    product-repository.js
    order-repository.js
    payment-client.js

  interfaces/
    http.js
```

---

# 22. Debugging Exercises

## Exercise 1 — Review defect

Find the bug:

```js
if (user.role = "admin") {
  allow();
}
```

The correct comparison was intended.

---

## Exercise 2 — Refactor regression

Before:

```js
await save();
await publish();
```

After:

```js
await Promise.all([
  save(),
  publish(),
]);
```

Find the semantic risks.

---

## Exercise 3 — Error regression

Before:

```js
throw originalError;
```

After:

```js
throw new Error(originalError.message);
```

What information was lost?

---

## Exercise 4 — Mutation regression

Before:

```js
options.cache.enabled = true;
```

After:

```js
options = {
  ...options,
  cache: {
    ...options.cache,
    enabled: true,
  },
};
```

Could callers observe a difference?

---

## Exercise 5 — Database regression

Before:

```text
one transaction
```

After refactor:

```text
repository A
repository B
each starts own transaction
```

Find the correctness problem.

---

## Exercise 6 — Memory regression

Before:

```text
process items one at a time
```

After:

```text
Promise.all(items.map(...))
```

Find possible memory and capacity problems.

---

## Exercise 7 — API compatibility

Before:

```js
error.code = "ORDER_CONFLICT";
```

After:

```js
error.code = "CONFLICT";
```

Determine whether this is a breaking change.

---

## Exercise 8 — Security regression

Refactor moves authorization from:

```text
application service
```

to:

```text
controller
```

but another caller bypasses the controller.

Find the boundary failure.

---

## Exercise 9 — Circular dependency

Refactor creates:

```text
orders → shared
shared → payments
payments → orders
```

Identify the new architecture cycle.

---

## Exercise 10 — Review overload

A pull request has 150 comments.

Determine whether:

```text
review quality
```

actually improved.

---

# 23. Code Review Exercise

Review:

```js
export async function refundOrder(req, res) {
  const order = await db.orders.findById(req.params.id);

  if (order.status !== "paid") {
    return res.status(400).json({
      error: "Cannot refund",
    });
  }

  const payment = await payments.refund(order.paymentId);

  await db.orders.update(order.id, {
    status: "refunded",
  });

  return res.json(order);
}
```

Identify at least 30 findings.

Consider:

```text
authentication
authorization
tenant isolation
missing order handling
transport/domain coupling
database abstraction leakage
payment idempotency
payment timeout
payment retry
partial failure
transaction semantics
state transition
race condition
concurrent refund
response mutation
response status
error contract
error leakage
observability
audit
metrics
trace
request ID
dependency timeout
database failure after refund
refund success after database failure
reconciliation
API versioning
locking/concurrency
test coverage
```

Then propose a safe sequence of refactors.

---

# 24. Interview Questions

## Fundamental

1. What is code review?
2. What is refactoring?
3. Refactoring versus rewrite?
4. What is a code smell?
5. What is technical debt?
6. Why review correctness first?
7. What makes review feedback useful?
8. What is cohesion?
9. What is coupling?
10. What is a characterization test?

## Intermediate

11. What is shotgun surgery?
12. What is a god object?
13. What is feature envy?
14. Why can duplicate code be acceptable?
15. How do you refactor legacy code safely?
16. What is a seam?
17. How do you review async code?
18. How do you review database changes?
19. How do you review security boundaries?
20. How do you prioritize findings?

## Advanced

21. How do you refactor a public API?
22. How do you preserve error compatibility?
23. How do you refactor transaction boundaries?
24. How do you refactor distributed workflows?
25. How do you review performance changes?
26. How do you detect hidden coupling?
27. How do you refactor a circular dependency?
28. How do you migrate a large module incrementally?
29. What makes a refactor behavior-preserving?
30. How do you handle technical debt?

## Principal

31. What makes a review comment worth blocking a merge?
32. How do you distinguish architecture problems from local implementation problems?
33. How do you refactor safely without stopping feature delivery?
34. When is duplication safer than abstraction?
35. When is a rewrite justified?
36. How do you organize organization-wide refactoring?
37. How do you measure whether reviews improve software quality?
38. How do you prevent review from becoming a bottleneck?
39. How do you evaluate a refactor that improves readability but worsens performance?
40. What evidence would convince you not to refactor?

---

# 25. Predict-the-Output Exercises

## Exercise A — Assignment versus comparison

Predict:

```js
let role = "user";

if (role = "admin") {
  console.log("allowed");
}

console.log(role);
```

### Actual Result

```text
allowed
admin
```

### Rule

Assignment evaluates to the assigned value.

Strict linting/static analysis should normally flag this pattern.

---

## Exercise B — Mutation versus copy

Predict:

```js
const options = {
  enabled: false,
};

const copy = {
  ...options,
};

copy.enabled = true;

console.log(options.enabled);
console.log(copy.enabled);
```

### Actual Result

```text
false
true
```

### Refactoring Lesson

Changing from mutation to copying can change observable identity and aliasing behavior.

---

## Exercise C — Error identity

Predict:

```js
const error = new Error("boom");

function wrap(original) {
  return new Error(original.message);
}

const wrapped = wrap(error);

console.log(wrapped === error);
```

### Actual Result

```text
false
```

### Rule

Creating a new error changes object identity and may also lose:

```text
custom fields
prototype/type
cause
original stack context
```

---

## Exercise D — Sequential versus concurrent refactor

Conceptually:

```js
await a();
await b();
```

versus:

```js
await Promise.all([
  a(),
  b(),
]);
```

Even if both eventually produce the same values, they differ in:

```text
start timing
ordering
failure interaction
resource usage
dependency assumptions
```

Therefore the transformation is not semantics-neutral by default.

---

## Exercise E — Object reference

Predict:

```js
const state = {
  count: 0,
};

const alias = state;

alias.count += 1;

console.log(state.count);
```

### Actual Result

```text
1
```

### Rule

Two variables can reference the same mutable object.

---

# 26. Mastery Exercises

## Track A — Core Theory

### A1

Take a 500-line function and identify:

```text
responsibilities
state
side effects
boundaries
invariants
dependencies
```

Do not refactor yet.

### A2

Classify each review finding:

```text
bug
security issue
reliability issue
performance issue
architecture smell
maintainability issue
preference
```

Then assign priority.

---

## Track B — Implementation

### B1 — Legacy refactor

Take a deliberately messy Node.js module and refactor it through:

```text
characterization tests
extract functions
extract dependencies
separate side effects
reduce coupling
```

### B2 — Async refactor

Refactor a callback-heavy module into Promise/async code while preserving:

```text
ordering
errors
cancellation
cleanup
```

### B3 — Database refactor

Refactor a database-heavy module to:

```text
repository
transaction boundary
batch queries
explicit error translation
```

### B4 — API refactor

Refactor an HTTP handler so that:

```text
controller
→ application
→ domain
→ infrastructure
```

while preserving the API contract.

---

## Track C — Interview / Reasoning

### C1

A pull request:

```text
changes 30 files
```

but behavior appears simple.

Explain how you would review it efficiently.

### C2

A refactor reduces lines of code by 40% but doubles memory usage.

Would you approve?

Defend the decision.

### C3

A team wants a six-month rewrite because the current code is “ugly.”

Design an evidence-based decision process.

---

# 27. Key Takeaways

1. Code review protects software quality through independent reasoning.
2. Review correctness before style.
3. Review the system context, not only changed lines.
4. Useful review comments identify mechanism and risk.
5. Code smells are signals, not automatic defects.
6. Cohesion and coupling are architectural review tools.
7. Refactoring changes structure while preserving required behavior.
8. Behavior includes more than output.
9. Timing, mutation, errors, ordering, resource usage, and compatibility can all be observable.
10. Characterization tests are valuable when behavior already exists but is poorly understood.
11. Small refactoring steps reduce uncertainty.
12. Seams make difficult code easier to change safely.
13. Duplication is sometimes safer than premature abstraction.
14. Public errors and APIs are compatibility surfaces.
15. Database transaction boundaries must survive refactoring.
16. Async refactors can change concurrency, timing, and failure behavior.
17. Performance is part of behavior when system requirements depend on it.
18. Security boundaries must remain explicit during refactoring.
19. Review quality is not measured by comment count.
20. Technical debt should be evaluated as a cost/risk decision.
21. Rewrites are different from behavior-preserving refactors.
22. Incremental migration is often safer than large replacement.
23. Tests provide evidence, not absolute proof.
24. Principal refactoring minimizes change risk while increasing future changeability.
25. The best review question is often:
   > **What could be wrong here that the current tests and happy path would fail to reveal?**

---

# 28. Concept Connections

## Depends On

- **9–21** — Functions / Scope / Objects
- **22–28** — Data Structures
- **29–40** — Errors / Async / Concurrency / Streaming
- **45–48** — Memory / GC / Engine
- **58–70** — Node.js / Modules / Packages / Tooling
- **71–77** — DSA / Paradigms / Composition / Patterns
- **78** — Production JavaScript Architecture
- **79** — API Design
- **80** — Library Authoring
- **81** — Database Integration
- **82** — API Architecture
- **83** — Observability
- **84** — Reliability
- **85** — Performance
- **86** — Testing
- **87** — Deterministic Async Testing
- **88** — Debugging Methodology

## Builds Toward

- **90** — Modern ECMAScript Features
- **91** — TC39 Proposal Tracking
- **94** — Compatibility Engineering
- **98** — Anti-patterns / Failure Modes
- **99** — Myths / Misconceptions
- **100** — Cost Model / Trade-offs
- **101** — Real-world Production Scenarios
- **102–111** — Production Projects
- **112–121** — Assessments / System Design / Principal Project

## Related Concepts

```text
Code Review / Refactoring
  ├─ correctness
  ├─ coupling
  ├─ cohesion
  ├─ complexity
  ├─ testing
  ├─ architecture
  ├─ security
  ├─ performance
  ├─ reliability
  └─ compatibility
```

## Concepts Revisited

```text
functions
objects
modules
promises
async/await
errors
streams
database transactions
API contracts
observability
reliability
performance
testing
debugging
design patterns
```

## Why This Chapter Matters Later

The later judgment chapters assume you can not only write JavaScript, but also:

```text
evaluate existing code
change it safely
explain trade-offs
identify hidden failure modes
```

This chapter is the bridge from individual implementation skill to engineering stewardship.

---

# 29. Completion Criteria

Mark:

```text
[~] In Progress
```

when you understand:

```text
review purpose
code smells
cohesion/coupling
basic refactoring
```

Mark:

```text
[?] Needs Revision
```

when you repeatedly confuse:

```text
style vs correctness
smell vs bug
refactor vs rewrite
duplication vs abstraction
local correctness vs system correctness
tests vs proof
readability vs performance
```

Mark:

```text
[+] Completed
```

when you can:

- review production JavaScript;
- prioritize findings;
- explain risks;
- characterize legacy behavior;
- refactor incrementally;
- preserve async semantics;
- preserve API contracts;
- preserve database semantics;
- improve testability;
- identify security/performance risks.

Mark:

```text
[*] Mastered
```

only when you can:

- lead large refactoring programs;
- review unfamiliar systems quickly;
- detect hidden coupling;
- identify semantic regressions;
- defend trade-offs;
- decide when not to refactor;
- design safe migration paths;
- improve architecture without freezing delivery;
- establish high-quality review standards.

Reading alone does not qualify as mastery.

---

# Chapter 89 — Revision / Retrieval Record

| Date | Retrieval Goal | Result | Status |
|---|---|---|---|
| ____ | Define code review | ____ | `[ ]` |
| ____ | Define refactoring | ____ | `[ ]` |
| ____ | Explain cohesion/coupling | ____ | `[ ]` |
| ____ | Identify code smells | ____ | `[ ]` |
| ____ | Prioritize review findings | ____ | `[ ]` |
| ____ | Write actionable review feedback | ____ | `[ ]` |
| ____ | Build characterization tests | ____ | `[ ]` |
| ____ | Refactor incrementally | ____ | `[ ]` |
| ____ | Preserve async semantics | ____ | `[ ]` |
| ____ | Preserve API compatibility | ____ | `[ ]` |
| ____ | Preserve transaction semantics | ____ | `[ ]` |
| ____ | Review security/performance | ____ | `[ ]` |
| ____ | Defend refactoring decision | ____ | `[ ]` |

## Spaced Retrieval

```text
Review 1 — same day
Review 2 — +1 day
Review 3 — +3 days
Review 4 — +7 days
Review 5 — +14 days
Review 6 — +30 days
Review 7 — +60 days
```

## Retrieval Prompts

Without reading:

1. What is code review for?
2. What is refactoring?
3. What counts as behavior?
4. Why are timing and mutation part of behavior?
5. What is a characterization test?
6. What is a code smell?
7. Cohesion versus coupling?
8. Why can duplication be safer?
9. What makes review feedback actionable?
10. How do you refactor async code safely?
11. How do you refactor transaction boundaries?
12. When is a rewrite justified?
13. How do you prioritize review findings?
14. How do you decide not to refactor?

---

# Chapter 89 — Canonical References and Source Discipline

## 1. ECMAScript

Use the ECMAScript specification for:

```text
language semantics
modules
functions
objects
Promise behavior
iteration
errors
```

Primary:

- https://tc39.es/ecma262/

---

## 2. Node.js

Use current Node.js documentation for:

```text
runtime behavior
test runner
diagnostics
performance
process lifecycle
HTTP
streams
async context
```

Primary:

- https://nodejs.org/docs/latest/api/

---

## 3. V8

Use V8 documentation for:

```text
engine implementation details
optimization
object representations
profiling concepts
```

Primary:

- https://v8.dev/

Do not turn V8 implementation details into universal JavaScript rules.

---

## 4. OWASP

Use OWASP for review of:

```text
injection
authorization
authentication
resource exhaustion
SSRF
security misconfiguration
unsafe consumption
```

Primary:

- https://owasp.org/

---

## 5. OpenTelemetry

Use OpenTelemetry for:

```text
instrumentation
tracing
metrics
context propagation
semantic conventions
```

Primary:

- https://opentelemetry.io/docs/languages/js/

---

## 6. Refactoring Literature

Useful conceptual references:

- Martin Fowler — *Refactoring*
- Michael Feathers — *Working Effectively with Legacy Code*
- Kent Beck — *Test-Driven Development*
- Martin Fowler — architecture/refactoring essays
- Robert C. Martin — *Clean Architecture*

These are engineering references rather than JavaScript language specifications.

---

## Source Discipline

Every review claim should be classified:

```text
language guarantee
runtime behavior
library behavior
security requirement
application contract
architecture choice
team convention
personal preference
```

Avoid comments such as:

> “JavaScript requires this.”

when the actual recommendation is:

> “This application boundary is easier to maintain if…”

Avoid:

> “This refactor is safe.”

unless the behavior being preserved is defined and verified.

Prefer:

> “The existing tests and characterization cases cover these observable behaviors; the refactor preserves them.”

---

# Chapter 89 — Completion Snapshot

## Core Theory

- [ ] Code review purpose
- [ ] Review scope
- [ ] Correctness-first review
- [ ] Risk prioritization
- [ ] Actionable feedback
- [ ] Cohesion
- [ ] Coupling
- [ ] Code smells
- [ ] Technical debt
- [ ] Characterization tests
- [ ] Behavior preservation
- [ ] Refactoring
- [ ] Seams
- [ ] Incremental migration
- [ ] Strangler refactoring
- [ ] Golden-master testing
- [ ] API compatibility
- [ ] Error compatibility
- [ ] Async semantic preservation
- [ ] Transaction preservation
- [ ] Performance review
- [ ] Security review
- [ ] Reliability review
- [ ] Observability review

## Implementation

- [ ] Review a production PR
- [ ] Build review checklist
- [ ] Build characterization tests
- [ ] Extract functions
- [ ] Extract dependencies
- [ ] Separate side effects
- [ ] Remove hidden globals
- [ ] Break circular dependencies
- [ ] Refactor async code
- [ ] Refactor API handler
- [ ] Refactor database access
- [ ] Fix N+1
- [ ] Preserve transactions
- [ ] Add regression tests
- [ ] Add security tests
- [ ] Add performance checks
- [ ] Improve observability
- [ ] Run migration incrementally
- [ ] Review final diff

## Interview / Reasoning

- [ ] Explain code review
- [ ] Explain refactoring
- [ ] Explain code smells
- [ ] Explain cohesion/coupling
- [ ] Prioritize findings
- [ ] Write actionable feedback
- [ ] Explain characterization testing
- [ ] Explain async refactoring risk
- [ ] Explain transaction refactoring
- [ ] Explain API compatibility
- [ ] Explain technical debt
- [ ] Defend rewrite vs refactor
- [ ] Defend duplication vs abstraction
- [ ] Defend review standards

## Mastery Gate

```text
Understand      [ ]
Explain         [ ]
Predict         [ ]
Implement       [ ]
Debug           [ ]
Apply           [ ]
Compare         [ ]
Defend          [ ]
```

## Final Principal Test

Given an unfamiliar production JavaScript pull request, can you determine:

```text
What behavior is changing?
What behavior must remain unchanged?
Which requirements are implied but not tested?
What are the security boundaries?
What are the reliability boundaries?
What are the performance implications?
What are the memory implications?
What asynchronous ordering changes?
What error behavior changes?
What resource ownership changes?
What API contracts change?
What database semantics change?
What hidden coupling is being introduced?
What coupling is being removed?
What abstractions are being added?
Are they justified?
What tests provide evidence?
What tests are missing?
What could fail only in production?
What could fail only under concurrency?
What could fail only under load?
What could fail only during shutdown?
What could leak secrets?
What could break compatibility?
Is the change smaller than necessary?
Is the change larger than necessary?
What should block merge?
What can be follow-up work?
What is the safest refactoring sequence?
```

A principal reviewer does not optimize for the number of comments.

They optimize for **the highest-value reduction in correctness, security, reliability, performance, and future-change risk per unit of review and refactoring effort.**

---

## Principal Code Review / Refactoring Decision Framework

For every significant finding or refactor, record:

```text
Behavior:
Requirement:
Risk:
Evidence:
Severity:
Scope:
Current Boundary:
Target Boundary:
Correctness:
Security:
Reliability:
Performance:
Memory:
Observability:
Compatibility:
Maintainability:
Complexity:
Test Coverage:
Migration Risk:
Rollback:
Decision:
Owner:
Revisit Trigger:
```

Then ask:

> **Does this change reduce a real source of risk or future change cost enough to justify the complexity, migration risk, and verification effort it introduces?**

That is the principal-level code-review question.