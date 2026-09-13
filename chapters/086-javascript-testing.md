# Chapter 86 — Testing

> **Part XVI — Testing / Debugging**  
> **Status:** `[ ] Not Started`  
> **Depth Target:** Principal / Staff-level reasoning  
> **Primary Focus:** Designing trustworthy automated tests for JavaScript systems, from pure functions to distributed production workflows.

---

# 1. Learning Objectives

By the end of this chapter, you should be able to:

1. Explain what testing is and what it cannot prove.
2. Distinguish verification, validation, testing, debugging, and monitoring.
3. Design a layered test strategy.
4. Explain the test pyramid and its limitations.
5. Distinguish unit, integration, contract, component, end-to-end, system, smoke, regression, and property-based tests.
6. Identify the right test level for a behavior.
7. Design tests around behavior rather than implementation details.
8. Write deterministic JavaScript tests.
9. Use assertions that communicate intent.
10. Test synchronous and asynchronous code correctly.
11. Test Promise rejection and error boundaries.
12. Test timers and time-dependent behavior.
13. Test cancellation.
14. Test retries and backoff without making tests slow or flaky.
15. Test concurrency and race conditions.
16. Test streams and backpressure.
17. Test Node.js HTTP APIs.
18. Test database integration.
19. Test transactions and rollback behavior.
20. Test external dependency adapters without coupling tests to live services.
21. Design contract tests.
22. Design integration test environments.
23. Understand mocks, stubs, spies, fakes, simulators, and fixtures.
24. Avoid over-mocking.
25. Detect flaky tests and diagnose their causes.
26. Understand test isolation and shared state.
27. Design test data safely.
28. Design parallel test execution.
29. Understand test runner concurrency.
30. Test resource cleanup.
31. Test process startup and shutdown.
32. Test observability behavior where appropriate.
33. Test security boundaries.
34. Test performance regressions without confusing performance tests with functional tests.
35. Design CI test pipelines.
36. Decide when a test is valuable enough to maintain.
37. Review test suites for false confidence.
38. Build a production-grade JavaScript test strategy.
39. Debug failing or flaky tests systematically.
40. Defend testing strategy at principal-engineer level.

---

# 2. Prerequisites

You should understand:

- JavaScript syntax, functions, closures, objects, modules.
- Promises, async/await, event loop, cancellation.
- Errors and cleanup.
- Streams and concurrency.
- Node.js runtime and HTTP.
- Database integration.
- API architecture.
- Observability.
- Reliability.
- Performance.
- Production architecture.

Recommended prior chapters:

- **29** — Errors / Error Handling
- **30** — Resource Management / Cleanup
- **31–40** — Async / Promises / Cancellation / Streaming / Concurrency
- **45–48** — Memory / GC / Engine / V8
- **58–63** — Node.js Runtime
- **70** — Source Maps / Production Debugging
- **71–73** — Data Structures / Complexity / Algorithms
- **78–85** — Production Architecture / API / Database / Observability / Reliability / Performance

---

# 3. What Is It?

Testing is the systematic execution of a program or system under controlled conditions to gather evidence about whether specified behavior holds.

A test can establish:

```text
given condition
when action occurs
then expected property holds
```

For example:

```js
test("rejects an invalid order", async () => {
  await assert.rejects(
    () => placeOrder({
      customerId: "cus_1",
      items: [],
    }),
    {
      code: "INVALID_ORDER",
    },
  );
});
```

A test is evidence.

It is not mathematical proof that every possible execution is correct.

---

# 4. Why Does It Exist?

Software has too many possible execution paths to validate manually.

Testing helps detect:

```text
regressions
incorrect assumptions
boundary failures
integration mismatches
concurrency bugs
security mistakes
refactoring damage
compatibility problems
```

Good testing also enables change.

Without tests:

```text
change
→ uncertainty
→ fear
→ slower delivery
```

With meaningful tests:

```text
change
→ automated evidence
→ faster feedback
→ safer evolution
```

But a bad test suite can become another source of uncertainty.

---

# 5. Mental Model

Think of testing as a layered evidence system:

```text
                Product Behavior
                       │
        ┌──────────────┼──────────────┐
        │              │              │
       Unit         Integration    End-to-End
        │              │              │
        └──────────────┼──────────────┘
                       │
                Contract Tests
                       │
                 Production Evidence
```

Different test levels answer different questions.

### Unit

> Does this unit behave correctly in isolation?

### Integration

> Do these components work correctly together?

### Contract

> Do independently deployed systems agree on an interface?

### End-to-end

> Does the complete user/system workflow work?

No single layer answers all four questions.

---

# 6. Core Rules

## Rule 1 — Test behavior, not implementation trivia

Prefer:

```text
places an order successfully
```

over:

```text
calls private function #7 twice
```

---

## Rule 2 — Test important invariants

Example:

```text
inventory can never become negative
```

is more valuable than:

```text
repository.save was called once
```

unless the call itself is the behavior under test.

---

## Rule 3 — Determinism is mandatory

A test should not randomly depend on:

```text
clock
network
database state
execution order
process timing
random IDs
environment variables
```

unless the test explicitly controls those dimensions.

---

## Rule 4 — Control time when time is part of behavior

Inject or control:

```text
clock
timer
deadline
```

rather than sleeping arbitrarily.

---

## Rule 5 — Isolate external dependencies

A unit test should not call:

```text
real payment provider
production database
real email
real cloud storage
```

unless the goal is explicitly an integration/contract test.

---

## Rule 6 — Do not mock everything

Over-mocking can prove that:

```text
your mocks behave like your expectations
```

instead of proving that the system behaves correctly.

---

## Rule 7 — Test failure paths

For production software, test:

```text
timeouts
retries
duplicate requests
partial failure
cancellation
shutdown
invalid input
dependency outages
```

---

## Rule 8 — Test cleanup

A passing assertion is not enough if the process retains:

```text
timers
sockets
listeners
workers
database connections
```

after the test.

---

## Rule 9 — Test contracts at boundaries

If a service depends on an API contract, validate the contract.

---

## Rule 10 — Flakiness is a defect

A test that randomly fails is not “just CI being noisy.”

Investigate it.

---

# 7. Syntax

Modern Node.js provides a built-in stable `node:test` test runner. Node's documentation states that the test runner became stable in Node.js 20.0.0 and supports synchronous tests, Promise-returning tests, and callback-style tests. citeturn755421search7turn755421search11

Basic example:

```js
import test from "node:test";
import assert from "node:assert/strict";

test("adds numbers", () => {
  assert.strictEqual(
    2 + 3,
    5,
  );
});
```

Run:

```bash
node --test
```

Node's current CLI documentation also supports test reporters and options for rerunning failures and sharding test execution. citeturn755421search12

---

# 8. Basic Examples

## Example 1 — Unit test

```js
export function add(a, b) {
  return a + b;
}
```

Test:

```js
import test from "node:test";
import assert from "node:assert/strict";
import { add } from "./add.js";

test("adds two values", () => {
  assert.strictEqual(add(2, 3), 5);
});
```

---

## Example 2 — Async test

```js
test("loads user", async () => {
  const user = await loadUser("u1");

  assert.equal(user.id, "u1");
});
```

A Promise-returning test is considered failing if the Promise rejects and passing if it fulfills under the Node test runner's model. citeturn755421search7

---

## Example 3 — Rejection

```js
test("rejects invalid input", async () => {
  await assert.rejects(
    () => parseOrder(null),
    {
      code: "INVALID_ORDER",
    },
  );
});
```

---

# 9. Execution Walkthrough

When a test runs:

```text
discover test
  ↓
load module
  ↓
construct fixtures
  ↓
run setup
  ↓
execute test
  ↓
assert
  ↓
cleanup
  ↓
report
```

For integration tests:

```text
start database
  ↓
migrate schema
  ↓
seed data
  ↓
start application
  ↓
send requests
  ↓
assert responses/state
  ↓
cleanup
```

The testing architecture itself needs lifecycle design.

---

# 10. Internal Mechanics

## 10.1 Assertion

An assertion compares actual behavior with expected behavior.

Example:

```js
assert.strictEqual(actual, expected);
```

Good assertions answer:

```text
what failed?
why does it matter?
```

OpenTelemetry's current test guidance recommends strict assertions and short, specific failure context; the same reasoning applies to JavaScript test suites generally. citeturn755421search14

---

## 10.2 Fixtures

A fixture is controlled test setup.

Example:

```js
function makeOrder(overrides = {}) {
  return {
    id: "ord_test",
    customerId: "cus_test",
    status: "pending",
    total: 1000,
    ...overrides,
  };
}
```

Avoid giant generic fixtures that contain irrelevant state.

---

## 10.3 Test doubles

### Stub

Provides controlled behavior.

### Spy

Records calls.

### Mock

Usually defines expected interactions or controlled behavior.

### Fake

Working simplified implementation.

Example:

```js
const repository = new InMemoryOrderRepository();
```

Fakes often provide stronger behavior evidence than deep mocks.

---

## 10.4 Dependency injection for tests

Production:

```js
createOrderService({
  repository: postgresRepository,
});
```

Test:

```js
createOrderService({
  repository: inMemoryRepository,
});
```

The application architecture from Chapter 78 enables this.

---

# 11. ECMAScript / Specification Semantics

Testing is not an ECMAScript language feature.

Separate:

```text
ECMAScript semantics
  ↓
what code means

test runner
  ↓
how tests are discovered/executed

assertion library
  ↓
how expectations are expressed

test environment
  ↓
database/network/process

CI
  ↓
when/how tests run
```

The current Node.js test runner is a Node runtime facility, not part of ECMAScript.

---

# 12. Advanced Behavior

## 12.1 Test pyramid

A common structure:

```text
             E2E
            /   \
        Integration
          /       \
        Unit tests
```

Interpretation:

```text
many fast tests
fewer slower integration tests
few broad E2E tests
```

But this is a heuristic, not a law.

A system with complex integration boundaries may rationally have many integration tests.

Optimize for:

```text
risk coverage
feedback speed
confidence
maintenance cost
```

---

## 12.2 Unit tests

Good candidates:

```text
pure functions
domain rules
parsers
normalizers
state transitions
policy functions
```

Poor unit-test targets:

```text
private implementation details
trivial getters
framework internals
```

unless risk justifies them.

---

## 12.3 Integration tests

Use when correctness depends on:

```text
database behavior
HTTP serialization
filesystem behavior
queue behavior
module loading
real adapter semantics
```

For example, a database repository should often be tested against the actual database engine or a sufficiently faithful integration environment.

---

## 12.4 Contract tests

A consumer depends on:

```http
POST /payments
```

Contract test verifies:

```text
request accepted
required fields
status
response schema
error behavior
```

This detects provider/consumer drift.

---

## 12.5 End-to-end tests

A checkout E2E test:

```text
login
→ add item
→ checkout
→ payment
→ order visible
```

Covers broad behavior.

But E2E tests are:

```text
slower
more fragile
more expensive to diagnose
```

Keep them focused on critical journeys.

---

# 13. Edge Cases

## 13.1 Time

Bad:

```js
await new Promise(resolve =>
  setTimeout(resolve, 500),
);
```

as the main way to test timing behavior.

Tests become slow and flaky.

Control the clock or timer where possible.

---

## 13.2 Randomness

Bad:

```js
const id = crypto.randomUUID();
```

when the exact ID is asserted.

Better:

```js
idGenerator: () => "test-id"
```

---

## 13.3 Environment

Bad:

```js
process.env.NODE_ENV === "test"
```

spread throughout application logic.

Prefer explicit configuration.

---

## 13.4 Test order

Bad:

```text
test A mutates global state
test B expects initial state
```

Tests should be independently runnable.

---

## 13.5 Concurrent tests

If tests share:

```text
same database row
same file
same port
same global cache
```

parallel execution can expose races.

Either isolate resources or control concurrency.

---

## 13.6 Async completion

Bad:

```js
test("works", () => {
  doSomethingAsync().then(() => {
    assert.equal(...);
  });
});
```

The test can finish before the callback runs.

Correct:

```js
test("works", async () => {
  const result = await doSomethingAsync();
  assert.equal(result, expected);
});
```

---

# 14. Common Misconceptions

### “100% coverage means 100% correctness.”

Coverage measures exercised code, not whether behavior is correct.

### “Unit tests are always better.”

Not when the behavior is integration-dependent.

### “Mocks make tests stable.”

They can also make tests disconnected from reality.

### “E2E tests replace unit tests.”

No.

### “Flaky tests are harmless.”

Flaky tests train teams to ignore failure.

### “A passing test proves the whole system works.”

It proves the tested scenario passed under the tested conditions.

### “Snapshots are always bad.”

Snapshots can be useful when output is large and meaningful, but they can become approval-driven noise.

### “Tests should never share any state.”

Isolation is ideal for many tests, but efficient controlled sharing can be acceptable at integration infrastructure level if lifecycle is explicit.

---

# 15. Common Mistakes

## Mistake 1 — Testing implementation details

```js
assert.equal(service.repository.save.callCount, 1);
```

when the real behavior is:

```text
order persisted correctly
```

---

## Mistake 2 — Over-mocking

Every dependency is replaced.

The test passes while the integration fails.

---

## Mistake 3 — No failure tests

Only:

```text
happy path
```

is covered.

---

## Mistake 4 — Slow test setup

Every unit test launches:

```text
database
server
browser
```

---

## Mistake 5 — Shared mutable fixtures

One test modifies an object reused by another.

---

## Mistake 6 — Hidden sleeps

```js
await sleep(1000);
```

everywhere.

---

## Mistake 7 — Ignoring cleanup

Tests leave:

```text
timers
connections
listeners
workers
```

open.

---

# 16. Comparison With Related Concepts

| Test Type | Main Question | Typical Cost |
|---|---|---|
| Unit | Does a unit behave correctly? | Low |
| Component | Does a component behave through its boundary? | Low–medium |
| Integration | Do real components work together? | Medium |
| Contract | Do producer/consumer interfaces agree? | Medium |
| E2E | Does a critical workflow work end-to-end? | High |
| Smoke | Is the system basically alive? | Low–medium |
| Regression | Is known behavior preserved? | Varies |
| Property-based | Does a property hold across many inputs? | Medium |
| Load | Does performance hold under load? | High |
| Soak | Does behavior remain stable over time? | High |
| Fault injection | Does failure handling work? | High |

---


---

# 16A. Test Contract Design

A good test has an explicit contract:

```text
Given:
When:
Then:
```

Example:

```js
test("cannot cancel a shipped order", () => {
  const order = makeOrder({
    status: "shipped",
  });

  assert.throws(
    () => order.cancel(),
    {
      code: "ORDER_CANNOT_BE_CANCELLED",
    },
  );
});
```

This test communicates:

```text
precondition = shipped
operation = cancel
invariant = shipped orders cannot be cancelled
```

The test is therefore useful even if the implementation changes from:

```text
class
→ function
→ state machine
```

because the behavioral contract remains.

---

## Test names as documentation

Prefer:

```text
rejects duplicate payment requests
```

over:

```text
works correctly
```

A failure should identify:

```text
behavior
condition
expected result
```

Avoid putting implementation names into test titles unless implementation behavior itself is the subject.

---

## Assert one coherent behavior

A test may contain several assertions when they jointly verify one outcome.

Good:

```js
assert.equal(response.status, 201);
assert.equal(response.body.data.status, "pending");
assert.equal(response.body.data.id, expectedId);
```

when all three describe the creation contract.

Bad:

```text
create order
+ delete user
+ send email
+ rotate token
```

in one unrelated test.

---

# 16B. Test Data Architecture

Test data should communicate intent.

Poor:

```js
const user = {
  id: 17,
  name: "John",
  age: 34,
  role: "admin",
  country: "IN",
  preferences: {},
  address: {},
  ...
};
```

when only:

```text
user.id
```

matters.

Prefer focused builders:

```js
function makeUser(overrides = {}) {
  return {
    id: "user-test",
    role: "user",
    ...overrides,
  };
}
```

Then:

```js
makeUser({ role: "admin" });
```

is self-explanatory.

---

## Factory versus fixture file

Use a factory when:

```text
small variations matter
```

Use a static fixture when:

```text
the exact payload is itself meaningful
```

For example, protocol compatibility tests may benefit from committed fixture files.

---

## Deterministic identifiers

Prefer:

```js
idGenerator: () => "id-123"
```

when the test needs to assert identity.

Use actual randomness in dedicated randomness/property tests.

---

# 16C. Property-Based Thinking

Example-based tests:

```text
1 input
→ 1 expected result
```

Property-based thinking:

```text
many valid inputs
→ invariant always holds
```

Example property:

```text
sorting a list does not change its elements
```

For a permutation:

```js
sort(values)
```

the property is:

```text
same multiset of elements
```

Property testing is especially useful for:

```text
parsers
serializers
encoders
normalizers
collections
mathematical functions
state transitions
```

Do not use property testing merely to generate random examples without a meaningful invariant.

---

# 16D. Mutation Testing

Mutation testing deliberately introduces small implementation changes:

```text
>
→ >=

+
→ -

return value
→ return null
```

Then checks whether the tests fail.

If:

```text
mutant survives
```

the suite may lack meaningful behavioral assertions.

Mutation testing can be expensive, so use it strategically on high-risk modules.

---

# 16E. Golden / Compatibility Tests

For libraries and APIs, capture stable expected outputs:

```text
input fixture
→ expected serialized output
```

Useful for:

```text
protocol encoders
CLI output
serialization formats
backward compatibility
snapshot-like contracts
```

But golden files can become dangerous when teams blindly regenerate them.

A changed golden output should trigger:

```text
review
intent confirmation
compatibility analysis
```

not automatic acceptance.

---

# 16F. Test Coverage

Coverage commonly includes:

```text
line coverage
statement coverage
branch coverage
function coverage
```

Coverage answers:

> Which code locations were executed?

It does not answer:

> Did the tests assert the correct behavior?

Example:

```js
function authorize(user) {
  if (user.admin) {
    return true;
  }

  return false;
}
```

100% line coverage does not prove:

```text
non-admin cannot access protected operation
```

A stronger test is:

```js
assert.equal(
  authorize({ admin: false }),
  false,
);
```

Use coverage as a diagnostic signal, not a target that overrides test quality.

---

# 16G. Test Economics

Every test has:

```text
authoring cost
execution cost
diagnosis cost
maintenance cost
flake risk
confidence value
```

A test suite should maximize:

```text
useful confidence
```

rather than:

```text
test count
```

A five-minute test that catches a catastrophic regression may be extremely valuable.

A ten-second test that verifies a trivial wrapper may not be.

---

# 16H. Contract Drift Detection

Suppose provider returns:

```json
{
  "status": "CONFIRMED"
}
```

consumer expects:

```text
confirmed
```

Contract testing should detect the mismatch before production.

Useful strategies:

```text
schema validation
consumer-driven contracts
generated types where appropriate
fixture validation
integration environments
```

No single technique catches every semantic mismatch.

---

# 16I. Test Observability

Tests can expose production diagnostics.

For a failing integration test, useful evidence includes:

```text
test name
request ID
trace ID
database operation
dependency response
timing
environment
```

Do not dump everything.

The objective is:

```text
failure
→ enough evidence
→ root cause
```

---

# 16J. Testability as Architecture

A module is more testable when:

```text
dependencies are explicit
side effects are isolated
time is controllable
I/O boundaries are visible
state ownership is clear
```

This is why testability and architecture are connected.

Poor architecture often produces tests like:

```text
start whole application
mock ten systems
sleep two seconds
assert one string
```

Strong architecture often permits:

```text
construct use case
inject fake dependency
execute
assert domain outcome
```

Testing can therefore serve as an architectural feedback mechanism.


# 17. Performance Considerations

Testing itself has a performance budget.

A CI suite that takes:

```text
10 seconds
```

encourages frequent execution.

A suite that takes:

```text
90 minutes
```

may encourage teams to avoid running it.

Optimize for:

```text
fast local feedback
parallel CI execution
targeted slow suites
```

---

## 17.1 Test parallelism

Node's current test runner supports concurrent test execution and sharding-related CLI capabilities. citeturn755421search12turn755421search13

Parallelism can reduce wall-clock time but requires:

```text
isolated state
unique resources
deterministic setup
```

---

## 17.2 Test data creation

A test that inserts:

```text
1 million records
```

for every run is expensive.

Prefer:

```text
small representative dataset
```

and dedicated performance/load suites for large workloads.

---

# 18. Memory Considerations

Tests can leak:

```text
timers
listeners
sockets
database clients
workers
closures
global caches
```

A test process that never exits is itself evidence of a lifecycle problem.

---

## 18.1 Test isolation

Bad:

```js
global.cache.set("x", value);
```

without cleanup.

Better:

```js
beforeEach(() => {
  cache.clear();
});
```

or construct a fresh cache per test.

---

## 18.2 Large fixtures

Avoid giant in-memory fixtures when a small boundary case is enough.

---

# 19. Security Considerations

Tests must not introduce production-like secrets into source control.

Avoid:

```text
real API keys
real customer data
production database dumps
private credentials
```

Use:

```text
synthetic test data
isolated credentials
ephemeral databases
mock/sandbox providers
```

---

## 19.1 Authorization tests

Test negative cases:

```text
unauthenticated
authenticated but unauthorized
wrong tenant
wrong object
wrong role
```

Security testing should not only confirm allowed behavior.

---

## 19.2 Injection tests

Test boundaries such as:

```text
SQL injection
path traversal
unsafe URLs
prototype-related input
malformed JSON
oversized payloads
```

depending on the application.

---

# 20. Production Usage

## 20.1 Test architecture

A production JavaScript project might use:

```text
test/
  unit/
  integration/
  contract/
  e2e/
  fixtures/
  helpers/
  performance/
  security/
```

The names are less important than test ownership and execution boundaries.

---

## 20.2 CI pipeline

Typical:

```text
lint
 ↓
unit
 ↓
integration
 ↓
contract
 ↓
build
 ↓
e2e
 ↓
security
 ↓
performance gate
```

Not every project needs all stages on every commit.

Use:

```text
fast feedback
risk-based execution
nightly/periodic deeper tests
```

where appropriate.

---

## 20.3 Test environments

Possible environments:

```text
pure process
ephemeral database
containerized dependencies
dedicated integration environment
staging
```

Use the smallest environment that tests the behavior.

---

## 20.4 Node.js built-in test runner

The current Node.js documentation marks the test runner as stable and provides APIs for tests, mocking, reporters, test coverage, concurrency, sharding, and rerunning failures. citeturn755421search7turn755421search12

A modern JavaScript project does not inherently require a third-party test framework.

Choose based on:

```text
team familiarity
features needed
ecosystem
migration cost
tooling integration
```

---

## 20.5 Testing observability

When telemetry is part of system behavior, test:

```text
event emitted
metric incremented
trace context preserved
sensitive field redacted
```

Do not make every unit test inspect telemetry implementation details.

Test meaningful contracts at appropriate boundaries.

OpenTelemetry's current JavaScript guidance covers native library instrumentation and manual instrumentation, which can inform dedicated instrumentation tests for reusable packages. citeturn755421search5

---

# 21. Implementation From Scratch

Build a production-grade Node.js test strategy.

## Stage 1 — Guided

Create:

```text
src/
  order.js

test/
  unit/
    order.test.js
```

Test:

```text
create
confirm
cancel
invalid transitions
```

---

## Stage 2 — Async testing

Test:

```text
success
rejection
timeout
cancellation
```

Example:

```js
test("times out", async () => {
  await assert.rejects(
    () => client.get("/slow"),
    error => error.code === "TIMEOUT",
  );
});
```

---

## Stage 3 — Integration testing

Create:

```text
database
repository
use case
HTTP adapter
```

Test the complete path using an isolated test database.

---

## Stage 4 — Contract testing

Define:

```text
request schema
response schema
error schema
```

Validate implementation against the contract.

---

## Stage 5 — Concurrency testing

Test:

```text
10 concurrent reservation requests
stock = 1
```

Expected:

```text
exactly one succeeds
others receive conflict/insufficient-stock behavior
```

Run the scenario repeatedly.

---

## Stage 6 — Failure testing

Inject:

```text
database timeout
payment failure
queue unavailable
cache failure
client cancellation
process shutdown
```

Verify expected behavior.

---

## Stage 7 — Production-grade suite

Add:

```text
unit
integration
contract
e2e
security
performance
fault injection
startup/shutdown
migration
compatibility
```

---

# 22. Debugging Exercises

## Exercise 1 — Flaky test

The test passes locally but fails 2% of CI runs.

Investigate:

```text
timing
parallelism
shared state
network
randomness
time zone
environment
```

---

## Exercise 2 — Hanging test

The process never exits.

Find:

```text
timer
socket
worker
database connection
event listener
```

---

## Exercise 3 — False-positive test

Test:

```js
assert.ok(response);
```

passes even when response has an invalid status.

Improve the assertion.

---

## Exercise 4 — Over-mocked integration

A payment integration test mocks:

```text
HTTP client
JSON parser
repository
payment gateway
error mapper
```

Everything passes, production fails.

Identify the testing boundary problem.

---

## Exercise 5 — Race test

Two concurrent requests both reserve the same inventory.

Design a reproducible test.

---

## Exercise 6 — Shared fixture

Test A changes:

```js
user.role = "admin";
```

Test B unexpectedly sees admin.

Find the isolation problem.

---

## Exercise 7 — Retry timing

A retry test sleeps:

```text
100 ms + 200 ms + 400 ms
```

for every run.

Design a deterministic alternative using injected delay/time.

---

## Exercise 8 — Database migration test

A migration works on a fresh database but fails on a production-like database with existing rows.

Design:

```text
upgrade-path test
backfill test
rollback/recovery test
```

---

# 23. Code Review Exercise

Review:

```js
test("creates order", async () => {
  const service = createOrderService({
    repository: {
      save: mock.fn(),
    },
    payment: {
      charge: mock.fn(),
    },
    inventory: {
      reserve: mock.fn(),
    },
  });

  await service.create({
    customerId: "1",
    total: 100,
  });

  assert.equal(service.repository.save.mock.calls.length, 1);
  assert.equal(service.payment.charge.mock.calls.length, 1);
  assert.equal(service.inventory.reserve.mock.calls.length, 1);
});
```

Identify at least 20 problems.

Consider:

```text
testing implementation interactions
no behavior assertion
mocks may lie
no payment result
no inventory result
no persisted state verification
no failure paths
no authorization
no idempotency
no transaction behavior
no concurrency
no error contract
no cleanup
potential invalid mock API assumptions
```

Then redesign the test strategy across:

```text
unit
integration
contract
e2e
```

---

# 24. Interview Questions

## Fundamental

1. What is testing?
2. What can tests prove?
3. Unit vs integration?
4. What is an E2E test?
5. What is a contract test?
6. What is a test double?
7. Stub vs spy vs mock vs fake?
8. What is test isolation?
9. What is a flaky test?
10. Why are assertions important?

## Intermediate

11. What is the test pyramid?
12. When should you use integration tests?
13. Why can over-mocking be dangerous?
14. How do you test async code?
15. How do you test Promise rejection?
16. How do you test timeouts?
17. How do you test retries?
18. How do you test database transactions?
19. How do you test API contracts?
20. How do you test authorization?

## Advanced

21. How do you test race conditions?
22. How do you test message consumers?
23. How do you test graceful shutdown?
24. How do you test database migrations?
25. How do you test external APIs?
26. How do you identify flaky tests?
27. How do you parallelize a large test suite safely?
28. How do you design ephemeral integration environments?
29. How do you test observability?
30. How do you avoid false confidence from coverage?

## Principal

31. What should your test suite optimize for?
32. When should behavior be covered at multiple test levels?
33. Which tests should block deployment?
34. How much E2E testing is enough?
35. When is a mock appropriate?
36. When should you use a fake instead?
37. How do you decide whether a flaky test is worth fixing or deleting?
38. How do you test distributed failure without making CI unstable?
39. How do you measure test-suite effectiveness?
40. What would make you distrust a test suite despite 95% coverage?

---

# 25. Predict-the-Output Exercises

## Exercise A — Assertion failure

Predict:

```js
import assert from "node:assert/strict";

assert.strictEqual(
  1,
  "1",
);
```

### Actual Result

The assertion throws because:

```text
number 1 !== string "1"
```

### Rule

Strict assertions distinguish type as well as value.

---

## Exercise B — Async test failure

Predict:

```js
test("example", async () => {
  throw new Error("failure");
});
```

### Actual Result

The test fails because the async function returns a rejected Promise.

Node's test runner treats a Promise-returning test as failing when the Promise rejects. citeturn755421search7

---

## Exercise C — Missing await

Conceptually:

```js
test("bad", () => {
  Promise.reject(new Error("boom"));
});
```

Question:

> Is this a reliable test of rejection?

### Answer

No.

The test itself does not return/await the Promise, so its lifecycle does not necessarily include the rejection.

---

## Exercise D — Shared mutable state

Predict:

```js
const values = [];

function add() {
  values.push(1);
}

add();

console.log(values.length);
```

### Actual Result

```text
1
```

### Testing Lesson

Shared state is easy to create accidentally. Test fixtures and module-level state require deliberate isolation.

---

# 26. Mastery Exercises

## Track A — Core Theory

### A1

Take one feature:

```text
create order
```

Define what belongs in:

```text
unit tests
integration tests
contract tests
E2E tests
```

Defend every placement.

### A2

Classify each test as:

```text
behavior
implementation
integration
contract
performance
security
```

and identify whether it creates meaningful evidence.

---

## Track B — Implementation

### B1 — Complete order test strategy

Build tests for:

```text
domain
application
repository
HTTP
database
payment integration
events
```

### B2 — Async reliability

Test:

```text
timeout
abort
retry
backoff
duplicate request
```

without real sleeps.

### B3 — Concurrency

Build a deterministic stress test around:

```text
inventory = 1
10 concurrent purchase attempts
```

### B4 — Integration environment

Build ephemeral infrastructure for:

```text
Postgres
queue
HTTP dependency
```

and ensure each test run is isolated.

---

## Track C — Interview / Reasoning

### C1

A project has:

```text
10,000 unit tests
50 integration tests
2 E2E tests
95% line coverage
```

but production still has frequent failures.

Diagnose what the metrics may be hiding.

### C2

A test suite takes:

```text
45 minutes
```

and developers rarely run it locally.

Redesign the feedback architecture.

### C3

An E2E test fails randomly 1/100 runs.

Design a systematic investigation before adding retries to the test.

---

# 27. Key Takeaways

1. Tests are evidence, not absolute proof.
2. Test behavior and important invariants.
3. Different test levels answer different questions.
4. Unit tests are not substitutes for integration tests.
5. Contract tests protect independently evolving boundaries.
6. E2E tests should focus on critical user/system journeys.
7. Determinism is essential.
8. Time, randomness, external state, and concurrency must be controlled deliberately.
9. Over-mocking can create false confidence.
10. Fakes can sometimes provide stronger evidence than mocks.
11. Error paths matter as much as happy paths.
12. Cleanup is part of test correctness.
13. Flaky tests are engineering defects.
14. Parallel tests require isolation.
15. Test runtime affects developer behavior.
16. Coverage measures execution, not semantic correctness.
17. Security tests must include denied cases.
18. Database and transaction behavior often requires real integration testing.
19. Async failure, cancellation, and retry semantics deserve dedicated tests.
20. Test observability contracts at meaningful boundaries.
21. Tests should make production risks visible.
22. The strongest test strategy is risk-based, layered, deterministic, and maintainable.
23. Principal testing is about maximizing trustworthy evidence per unit of maintenance cost.

---

# 28. Concept Connections

## Depends On

- **29** — Errors
- **30** — Resource Management
- **31–40** — Async / Concurrency / Cancellation / Streaming
- **45–48** — Memory / GC / Engine
- **58–63** — Node.js Runtime
- **64–70** — Modules / Packages / Tooling
- **71–73** — Data Structures / Complexity / Algorithms
- **78** — Production Architecture
- **79** — API Design
- **81** — Database Integration
- **82** — API Architecture
- **83** — Observability
- **84** — Reliability
- **85** — Performance

## Builds Toward

- **87** — Deterministic Async Testing
- **88** — Debugging Methodology
- **89** — Code Review / Refactoring
- **94** — Compatibility Engineering
- **98–101** — Judgment / Failure Modes / Production Scenarios
- **102–111** — Production Projects
- **112–121** — Assessments / System Design / Principal Project

## Related Concepts

```text
Testing
  ├─ unit
  ├─ integration
  ├─ contract
  ├─ E2E
  ├─ property-based
  ├─ regression
  ├─ security
  ├─ performance
  ├─ fault injection
  └─ CI
```

## Concepts Revisited

```text
Promises
async/await
AbortSignal
events
streams
modules
database transactions
API contracts
observability
reliability
performance
resource cleanup
```

## Why This Chapter Matters Later

The next chapter specializes in the hardest testing problem in JavaScript systems:

```text
time
concurrency
event-loop behavior
Promise scheduling
cancellation
deterministic async failure
```

The broader testing strategy established here becomes the foundation for that work.

---

# 29. Completion Criteria

Mark:

```text
[~] In Progress
```

when you understand:

```text
unit
integration
contract
E2E
test doubles
assertions
```

Mark:

```text
[?] Needs Revision
```

when you repeatedly confuse:

```text
coverage vs correctness
mock vs fake
unit vs integration
async vs concurrent
timeout vs cancellation
test failure vs flaky infrastructure
behavior vs implementation
```

Mark:

```text
[+] Completed
```

when you can:

- create a layered test strategy;
- write deterministic async tests;
- test databases;
- test APIs;
- test contracts;
- test authorization;
- test concurrency;
- test failure behavior;
- manage test cleanup;
- build a CI testing pipeline.

Mark:

```text
[*] Mastered
```

only when you can:

- identify false confidence in a test suite;
- design tests from production risks;
- decide where behavior should be tested;
- diagnose flaky tests;
- choose mocks/fakes/integration strategically;
- design concurrency tests;
- design migration/contract tests;
- defend a testing strategy at principal level.

Reading alone does not qualify as mastery.

---

# Chapter 86 — Revision / Retrieval Record

| Date | Retrieval Goal | Result | Status |
|---|---|---|---|
| ____ | Define testing and its limits | ____ | `[ ]` |
| ____ | Compare test levels | ____ | `[ ]` |
| ____ | Design unit tests | ____ | `[ ]` |
| ____ | Design integration tests | ____ | `[ ]` |
| ____ | Design contract tests | ____ | `[ ]` |
| ____ | Design E2E tests | ____ | `[ ]` |
| ____ | Explain test doubles | ____ | `[ ]` |
| ____ | Design deterministic async tests | ____ | `[ ]` |
| ____ | Test database concurrency | ____ | `[ ]` |
| ____ | Test failure/retry behavior | ____ | `[ ]` |
| ____ | Diagnose flaky tests | ____ | `[ ]` |
| ____ | Defend testing strategy | ____ | `[ ]` |

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

1. What can tests prove?
2. Unit vs integration?
3. What is a contract test?
4. Why is over-mocking dangerous?
5. What makes a test deterministic?
6. How do you test async rejection?
7. How do you test timeouts without sleeping?
8. How do you test database concurrency?
9. Why is coverage insufficient?
10. How do you diagnose flaky tests?
11. What belongs in E2E?
12. How do you design a test suite for production failure modes?

---

# Chapter 86 — Canonical References and Source Discipline

## 1. Node.js Test Runner

Node.js documents `node:test` as a stable test runner, with stability 2 and history showing the test runner became stable in Node.js 20.0.0. It supports synchronous, Promise-based, and callback-style tests. citeturn755421search7

Primary:

- https://nodejs.org/api/test.html

---

## 2. Node.js Test CLI

Current Node.js test CLI documentation includes:

```text
test reporter
test reporter destination
test sharding
test rerun failures
```

and other test-running controls. citeturn755421search12

Primary:

- https://nodejs.org/api/cli.html

---

## 3. Node.js Documentation

Use current Node.js documentation for:

```text
assert
test runner
mocking
timers
HTTP
streams
workers
process lifecycle
```

Primary:

- https://nodejs.org/api/

The current Node.js documentation index identifies the Test Runner as stable. citeturn755421search15

---

## 4. OpenTelemetry Testing Guidance

OpenTelemetry's current testing guidance uses Node's `node:test` and `assert` APIs and recommends assertions that are strict, specific, and easy to diagnose. citeturn755421search14

Primary:

- https://opentelemetry.io/site/testing/

---

## 5. OpenTelemetry JavaScript

OpenTelemetry JavaScript currently supports Node.js and browser use cases and documents traces and metrics as stable while logs remain in development. Its current documentation also states that it supports active or maintenance LTS Node.js versions. citeturn755421search0

Use the current JavaScript documentation for observability-specific test strategies:

- https://opentelemetry.io/docs/languages/js/

---

## 6. OWASP

Use OWASP guidance for testing security properties such as:

```text
authorization
injection
resource exhaustion
unsafe input
authentication
```

Primary:

- https://owasp.org/

---

## Source Discipline

Every test strategy decision should be classified:

```text
language behavior
runtime behavior
test-runner behavior
library behavior
application contract
integration behavior
environment behavior
CI policy
```

Do not assume:

> “The test runner guarantees deterministic order.”

Verify the runner's documented execution semantics and design tests that do not depend on accidental ordering.

Do not assume:

> “A mock represents the real dependency.”

A mock represents what you programmed it to represent.

Integration tests validate the real boundary.

---

# Chapter 86 — Completion Snapshot

## Core Theory

- [ ] Testing purpose
- [ ] Testing limits
- [ ] Verification vs validation
- [ ] Unit tests
- [ ] Component tests
- [ ] Integration tests
- [ ] Contract tests
- [ ] E2E tests
- [ ] Regression tests
- [ ] Property-based tests
- [ ] Test pyramid
- [ ] Behavioral testing
- [ ] Assertions
- [ ] Fixtures
- [ ] Test doubles
- [ ] Test isolation
- [ ] Determinism
- [ ] Async testing
- [ ] Time control
- [ ] Cancellation testing
- [ ] Concurrency testing
- [ ] Streams testing
- [ ] Database testing
- [ ] API testing
- [ ] Contract testing
- [ ] Security testing
- [ ] Performance testing
- [ ] Fault injection
- [ ] Cleanup
- [ ] Flakiness
- [ ] CI strategy

## Implementation

- [ ] Node test runner
- [ ] Strict assertions
- [ ] Unit test suite
- [ ] Async test suite
- [ ] Error tests
- [ ] Timeout tests
- [ ] Cancellation tests
- [ ] Retry tests
- [ ] Database integration tests
- [ ] Transaction tests
- [ ] Concurrency tests
- [ ] API tests
- [ ] Contract tests
- [ ] E2E tests
- [ ] Authorization tests
- [ ] Security boundary tests
- [ ] Migration tests
- [ ] Startup/shutdown tests
- [ ] Observability tests
- [ ] Fault-injection tests
- [ ] Performance regression tests
- [ ] CI parallelization
- [ ] Test sharding
- [ ] Flaky-test detection

## Interview / Reasoning

- [ ] Explain test levels
- [ ] Explain test doubles
- [ ] Explain over-mocking
- [ ] Explain test determinism
- [ ] Explain test pyramid limits
- [ ] Design concurrency test
- [ ] Design contract tests
- [ ] Design migration tests
- [ ] Diagnose flaky tests
- [ ] Evaluate coverage
- [ ] Design CI pipeline
- [ ] Defend testing strategy

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

Given an unfamiliar JavaScript system, can you determine:

```text
What are the highest-risk behaviors?
Which tests validate those behaviors?
Which boundaries require integration tests?
Which contracts require contract tests?
Which workflows require E2E tests?
What is mocked?
Why is it mocked?
Where are fakes more appropriate?
Which tests depend on time?
Which depend on randomness?
Which depend on external services?
Which tests can run in parallel?
What shared state exists?
How is test isolation enforced?
How are database transactions tested?
How are concurrency bugs tested?
How are retries tested?
How are cancellation semantics tested?
How are startup/shutdown failures tested?
How are authorization failures tested?
How are security boundaries tested?
What happens when migrations meet existing data?
What happens when observability is unavailable?
Which tests block deployment?
Which tests are too slow?
Which tests are flaky?
Which tests create false confidence?
What production failures remain unrepresented?
```

A principal testing engineer does not ask:

> **“How much code is covered?”**

They ask:

> **“What important behavior could still be wrong, what evidence do we have about it, and is that evidence trustworthy enough to let us change the system safely?”**

---

## Principal Testing Decision Framework

For every important behavior, record:

```text
Risk:
Business Impact:
Behavior:
Invariant:
Boundary:
Test Level:
Fixture:
Dependencies:
Isolation:
Determinism:
Failure Cases:
Security Cases:
Concurrency Cases:
Performance Cases:
Cleanup:
Expected Signal:
Failure Diagnosis:
Maintenance Cost:
Execution Cost:
Confidence:
Decision:
Revisit Trigger:
```

The strongest test suite is not the largest one.

It is the one that provides **high-confidence evidence about high-risk behavior, fails deterministically, diagnoses problems quickly, and costs less to maintain than the production failures it prevents.**