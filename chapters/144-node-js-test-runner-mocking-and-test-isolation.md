# Chapter 144 — Node.js Test Runner, Mocking & Test Isolation

> **JavaScript Mastery — Part XXV: Testing, Reliability & Runtime Verification**
>
> **Mission:** Master Node.js's built-in `node:test` runner as a production testing platform. Learn test discovery and execution, suites, hooks, subtests, concurrency, test isolation, process boundaries, retries, rerun-failures, randomization, sharding, coverage, reporters, mocking, timers, Date mocking, module mocking, function/method/property mocks, dependency replacement, test context cleanup, flaky-test diagnosis, parallel safety, worker/process interactions, permission-model testing, diagnostics, CI architecture, and principal-level test-system design.
>
> **Role perspective:** Principal JavaScript Engineer · Node.js Runtime Engineer · Test Infrastructure Engineer · SRE · QA Architect · Reliability Engineer · Build/CI Engineer
>
> **Status:** `[ ] Not Started`
>
> **Core rule:** **A test suite is itself a concurrent software system. Isolation, determinism, cleanup, ownership, and observability are correctness properties of the test infrastructure—not optional conveniences.**

---

# 1. Learning Objectives

```text
[ ] explain node:test
[ ] explain Node's built-in test runner
[ ] explain why built-in test tooling matters
[ ] run node --test
[ ] discover test files
[ ] filter tests by name
[ ] filter tests by tags
[ ] explain test-only execution
[ ] explain suites
[ ] explain tests
[ ] explain subtests
[ ] explain describe
[ ] explain it
[ ] explain test
[ ] explain before
[ ] explain after
[ ] explain beforeEach
[ ] explain afterEach
[ ] explain suite hooks
[ ] explain hook ordering
[ ] explain hook failure
[ ] explain test failure propagation
[ ] understand nested suites
[ ] understand subtest concurrency
[ ] understand file-level concurrency
[ ] understand test ordering
[ ] randomize test order
[ ] use test random seeds
[ ] understand deterministic randomization
[ ] understand sharding
[ ] understand test isolation
[ ] explain process isolation
[ ] explain non-isolated execution
[ ] understand default isolation mode
[ ] use --test-isolation
[ ] understand child-process isolation
[ ] understand environment differences between isolated tests
[ ] understand shared process state
[ ] explain why globals leak
[ ] explain why module caches leak
[ ] explain why filesystem state leaks
[ ] explain why environment variables leak
[ ] explain why timers leak
[ ] explain why sockets leak
[ ] explain why workers leak
[ ] explain why database state leaks
[ ] explain cleanup contracts
[ ] design isolated fixtures
[ ] design disposable resources
[ ] explain TestContext
[ ] explain context.test
[ ] explain context.mock
[ ] explain context.signal
[ ] explain context.filePath
[ ] explain context.name where supported
[ ] explain context.fullName where supported
[ ] use test context cleanup
[ ] explain mock tracker
[ ] create mock functions
[ ] inspect mock call count
[ ] inspect mock calls
[ ] inspect mock results
[ ] inspect mock this value
[ ] inspect mock target
[ ] inspect invocation arguments
[ ] inspect returned values
[ ] inspect thrown values
[ ] restore mocks
[ ] reset mocks
[ ] explain mock restoration
[ ] mock methods
[ ] mock properties
[ ] mock getters
[ ] mock setters
[ ] replace functions
[ ] restore replaced properties
[ ] explain mock call history
[ ] explain mock call ordering
[ ] explain mock implementation
[ ] explain nested mocks
[ ] explain mock lifetime
[ ] explain mock leakage
[ ] explain mock isolation
[ ] mock timers
[ ] mock setTimeout
[ ] mock setInterval
[ ] mock setImmediate
[ ] mock Date
[ ] advance mocked time
[ ] inspect scheduled timers
[ ] reset mocked timers
[ ] understand Date/time coupling
[ ] understand limitations of timer mocking
[ ] test retry logic with fake time
[ ] test timeouts with fake time
[ ] test polling with fake time
[ ] test exponential backoff with fake time
[ ] test cron-like logic with fake time
[ ] explain real time vs fake time
[ ] understand monotonic clocks vs Date
[ ] understand timer queue semantics
[ ] explain module mocking
[ ] understand experimental module mocking status
[ ] understand ESM module mocking limitations
[ ] understand module cache interactions
[ ] understand import timing
[ ] explain mock.module
[ ] use mock.module carefully
[ ] reset module mocks
[ ] explain module mocking and isolation
[ ] explain why module mocks are harder than function mocks
[ ] mock filesystem dependencies
[ ] mock network dependencies
[ ] mock database dependencies
[ ] mock environment/configuration
[ ] mock time
[ ] mock randomness
[ ] avoid mocking implementation details
[ ] distinguish mock vs stub vs spy vs fake
[ ] explain test double taxonomy
[ ] decide when not to mock
[ ] prefer dependency injection where useful
[ ] use adapters for external resources
[ ] design deterministic seams
[ ] test retries
[ ] test cancellation
[ ] test timeouts
[ ] test abort signals
[ ] test concurrency
[ ] test race conditions
[ ] test cleanup
[ ] test process exit
[ ] test worker termination
[ ] test child-process behavior
[ ] test signal handling safely
[ ] test permission-denied paths
[ ] test network failures
[ ] test filesystem failures
[ ] test malformed inputs
[ ] test resource exhaustion
[ ] understand flaky tests
[ ] classify flaky failures
[ ] diagnose order dependence
[ ] diagnose shared-state races
[ ] diagnose timing races
[ ] diagnose network nondeterminism
[ ] diagnose system-resource nondeterminism
[ ] diagnose timezone dependence
[ ] diagnose locale dependence
[ ] diagnose random-seed dependence
[ ] diagnose port collisions
[ ] diagnose temporary-directory collisions
[ ] diagnose worker/process cleanup failures
[ ] diagnose leftover handles
[ ] understand test reruns
[ ] use --test-rerun-failures
[ ] understand rerun state files
[ ] understand why reruns can hide flakiness
[ ] use test randomization
[ ] use random seeds to reproduce order issues
[ ] shard large suites
[ ] understand shard boundary effects
[ ] understand concurrency tuning
[ ] choose concurrency by workload
[ ] prevent CPU oversubscription
[ ] prevent connection storms in tests
[ ] prevent rate-limit collisions
[ ] prevent filesystem collisions
[ ] control test database isolation
[ ] explain transactions vs test databases
[ ] use unique test identities
[ ] understand environment isolation
[ ] isolate secrets
[ ] avoid production credentials
[ ] use ephemeral services
[ ] understand containers in test environments
[ ] understand mocks vs contract tests
[ ] understand test pyramid
[ ] understand test portfolio
[ ] explain unit tests
[ ] explain integration tests
[ ] explain end-to-end tests
[ ] explain smoke tests
[ ] explain regression tests
[ ] explain characterization tests
[ ] explain contract tests
[ ] explain property-based tests
[ ] explain fuzz tests
[ ] understand test runner reporters
[ ] choose reporter format
[ ] write custom test reporters conceptually
[ ] understand test event streams
[ ] understand diagnostic messages
[ ] use test:diagnostic
[ ] use test:pass
[ ] use test:fail
[ ] understand test events
[ ] understand programmatic test runner APIs
[ ] understand run()
[ ] configure concurrency programmatically
[ ] configure isolation programmatically
[ ] configure coverage
[ ] configure glob patterns
[ ] understand setup/teardown modules
[ ] understand global setup
[ ] understand global teardown
[ ] understand test lifecycle at process level
[ ] understand watch mode implications
[ ] understand test runner in CI
[ ] understand test runner under Permission Model
[ ] understand mock-module restrictions under Permission Model
[ ] understand worker permissions in test tooling
[ ] understand diagnostics of test failures
[ ] correlate test failures with logs/traces
[ ] capture test artifacts
[ ] build a test result database
[ ] build flaky-test metrics
[ ] build failure fingerprints
[ ] build retry policies
[ ] build safe test cleanup
[ ] build custom fixtures
[ ] build reusable test helpers
[ ] build deterministic test clocks
[ ] build isolated HTTP tests
[ ] build isolated filesystem tests
[ ] build subprocess tests
[ ] build worker tests
[ ] build permission-model tests
[ ] build CI sharding
[ ] design a scalable Node test platform


# 2. Prerequisites

You should already understand:

```text
Chapter 31 — Async Fundamentals
Chapter 33 — Event Loop
Chapter 39 — Streams
Chapter 52 — Workers / Concurrency
Chapter 63 — Diagnostics
Chapter 70 — Production Debugging
Chapter 71 — Security
Chapter 83 — Observability
Chapter 84 — Reliability
Chapter 85 — Performance
Chapter 86 — Testing
Chapter 88 — Debugging Methodology
Chapter 101 — Production Scenarios
Chapter 125 — Promise Internals
Chapter 140 — Node HTTP/TLS/DNS/TCP
Chapter 141 — Node Diagnostics & Inspector
Chapter 142 — Node Permission Model
Chapter 143 — Native Addons / FFI / ABI
```

You should also understand:

```text
Promises
AbortController
events
modules
environment variables
filesystem
HTTP
timers
workers
child processes
databases
CI.
```

---

# 3. What Is `node:test`?

Node's built-in test runner is the standard-library testing framework exposed through:

```js
node:test
```

Current Node documentation classifies the Test Runner as stable. citeturn444443search2

It provides:

```text
test definitions
suites
hooks
subtests
mocking
timers
coverage integration
reporters
concurrency
isolation
filtering
sharding
reruns
```

---

# 4. Why a Built-In Test Runner Matters

A runtime-integrated test runner can understand:

```text
Node processes
workers
modules
signals
timers
diagnostics
permissions
native behavior.
```

This reduces:

```text
framework/runtime impedance mismatch.
```

---

# 5. Test Runner Mental Model

```text
CLI
 ↓
test discovery
 ↓
test scheduler
 ↓
test files
 ↓
suites/tests
 ↓
hooks
 ↓
test body
 ↓
mocks/fixtures
 ↓
cleanup
 ↓
result/report
```

---

# 6. Minimal Test

```js
import test from "node:test";
import assert from "node:assert/strict";

test("addition", () => {
  assert.equal(2 + 2, 4);
});
```

Run:

```bash
node --test
```

---

# 7. Assertion Discipline

The test runner does not make assertions meaningful automatically.

You still need:

```text
assertion
```

to establish:

```text
expected behavior.
```

Prefer:

```js
assert.equal(actual, expected);
```

over:

```js
console.log(actual);
```

---

# 8. Test Names

Good:

```text
rejects invalid access token
```

Bad:

```text
test 1
```

Test names should describe:

```text
observable behavior.
```

---

# 9. Test Discovery

Node's test runner can discover test files according to its documented naming/discovery rules.

Do not depend on:

```text
IDE-specific discovery.
```

Make CI run:

```text
same command
```

developers use.

---

# 10. Filtering by Name

Current Node CLI supports:

```bash
node --test --test-name-pattern="checkout"
```

to select tests by name. citeturn444443search0

This is useful for:

```text
focused debugging
```

but CI should usually run:

```text
complete suite.
```

---

# 11. Tags

Current Node versions also support test tags and a boolean tag-filter expression through:

```bash
--experimental-test-tag-filter
```

The feature was added in Node v26.2.0 and is still early development in current documentation. citeturn444443search0

Use tags for:

```text
slow
network
database
browser
native
security.
```

---

# 12. `test.only`

Focused execution can be useful locally.

But:

```text
only
```

must never accidentally reach:

```text
production CI.
```

Use CI checks to prevent:

```text
committed focused tests.
```

---

# 13. Suites

Suites group:

```text
related tests
shared setup
shared teardown.
```

Example:

```js
import {
  describe,
  it
} from "node:test";

describe("UserService", () => {
  it("creates user", () => {});
  it("rejects duplicate", () => {});
});
```

---

# 14. Hooks

Typical lifecycle:

```text
before
beforeEach
test
afterEach
after
```

Use:

```text
before
```

for:

```text
suite-level setup.
```

Use:

```text
beforeEach
```

for:

```text
per-test isolation.
```

---

# 15. Hook Ownership

A hook should create:

```text
resources
```

that it can later:

```text
clean.
```

Bad:

```text
global before
→ creates shared mutable database state
```

without:

```text
transaction/reset.
```

---

# 16. Hook Failure

A failed:

```text
beforeEach
```

means the test may:

```text
not safely execute.
```

A failed:

```text
afterEach
```

may indicate:

```text
cleanup failure
```

that can corrupt later tests.

Treat cleanup failures seriously.

---

# 17. Cleanup Is Correctness

The test:

```text
passes
```

but leaves:

```text
socket
timer
worker
directory
database row
environment variable
```

behind.

The next test can fail:

```text
randomly.
```

Therefore:

```text
cleanup = correctness.
```

---

# 18. `TestContext`

Test callbacks can receive a:

```js
context
```

object.

Conceptually:

```js
test("example", context => {
  context.mock;
  context.signal;
});
```

Use context to keep:

```text
test resources
```

scoped to:

```text
one test.
```

---

# 19. Context Signal

The test context exposes cancellation/timeout-related signaling.

Useful for:

```text
long async operation
resource cleanup
```

and:

```text
cooperative cancellation.
```

---

# 20. Subtests

A test can create:

```text
nested subtests
```

to represent:

```text
phases
parameter cases
logical groups.
```

---

# 21. Subtest Isolation

Subtests can run:

```text
sequentially
```

or:

```text
concurrently
```

depending on configuration.

Do not assume:

```text
declaration order
=
execution order
```

when concurrency is enabled.

---

# 22. Concurrent Tests

Parallel execution improves:

```text
wall-clock speed
```

but requires:

```text
state isolation.
```

Parallel tests must not share:

```text
mutable globals
temporary filenames
ports
database rows
```

without coordination.

---

# 23. Test Isolation

Node's test runner supports configurable isolation.

Current Node documentation states:

```text
process
```

isolation runs each test file in a separate child process, while:

```text
none
```

runs test files in the same process as the runner.

The default isolation mode is:

```text
process
```

when using:

```text
--test.
```

citeturn444443search0

---

# 24. Process Isolation

Conceptually:

```text
Test Runner
   ├── child process → file A
   ├── child process → file B
   └── child process → file C
```

Benefits:

```text
global isolation
module-cache isolation
env isolation
resource ownership separation.
```

Costs:

```text
startup
memory
IPC
process scheduling.
```

---

# 25. No Isolation

Conceptually:

```text
single process
 ├── file A
 ├── file B
 └── file C
```

Benefits:

```text
less process overhead
```

Risks:

```text
state leakage
module-cache leakage
global mutation
environment leakage.
```

---

# 26. Why Isolation Mode Matters

Choose:

```text
process
```

when:

```text
tests are not reliably isolated.
```

Choose:

```text
none
```

only when:

```text
shared-process execution is known safe
```

and:

```text
performance benefit
```

is meaningful.

---

# 27. Isolation Is Not Magic

Even process isolation does not isolate:

```text
external database
filesystem volume
localhost ports
third-party services.
```

These remain:

```text
shared external resources.
```

---

# 28. External Resource Isolation

Use:

```text
unique database schema
unique transaction
unique user
unique port
unique temp directory
```

rather than:

```text
shared mutable resource.
```

---

# 29. Test Database Isolation

Strategies include:

```text
transaction rollback
schema-per-worker
database-per-test
database-per-suite
ephemeral database container.
```

Choose based on:

```text
cost
concurrency
fidelity.
```

---

# 30. Filesystem Isolation

Use:

```js
import { mkdtemp } from "node:fs/promises";
```

to create:

```text
unique temporary directory.
```

Clean it:

```text
after test.
```

---

# 31. Port Isolation

Never hard-code:

```text
3000
```

for every test.

Use:

```text
port 0
```

where API semantics support ephemeral port allocation, then discover:

```text
assigned port.
```

---

# 32. Environment Isolation

If a test sets:

```js
process.env.MODE = "test";
```

it must restore:

```text
previous state.
```

Otherwise:

```text
later tests
```

inherit:

```text
unexpected configuration.
```

---

# 33. Module Cache Isolation

Importing:

```js
config.js
```

may cache:

```text
configuration state.
```

A test that mutates:

```text
global/config singleton
```

can affect later imports.

Use:

```text
dependency injection
isolated process
fresh module strategy
```

rather than:

```text
random cache manipulation.
```

---

# 34. Module Evaluation Timing

For ESM:

```text
import
```

happens before many test-body actions.

Therefore a module mock often must be established:

```text
before the dependency is loaded/evaluated.
```

This is one reason module mocking is:

```text
harder
```

than:

```text
mocking a passed-in function.
```

---

# 35. Mocking

Mocking replaces:

```text
real behavior
```

with:

```text
controlled behavior.
```

Examples:

```text
network
clock
randomness
filesystem
database
external service.
```

---

# 36. Mock vs Stub vs Spy vs Fake

### Mock

```text
controlled replacement + interaction expectations.
```

### Stub

```text
predefined response.
```

### Spy

```text
records interaction while behavior may remain.
```

### Fake

```text
working lightweight replacement.
```

Terminology varies across teams.

Focus on:

```text
behavior
control
observation.
```

---

# 37. `mock.fn()`

Node's test runner can create mock functions.

Example:

```js
import { mock } from "node:test";

const fn = mock.fn();
fn("a", "b");
```

You can inspect:

```text
call count
arguments
result
target
this.
```

Current Node test-runner documentation provides these mock inspection facilities. citeturn444443search1

---

# 38. Mock Call Count

```js
assert.equal(
  fn.mock.callCount(),
  1
);
```

Useful for:

```text
interaction contract.
```

Do not use:

```text
call count
```

as the only test of correctness.

---

# 39. Mock Calls

```js
fn.mock.calls[0]
```

can expose:

```text
arguments
result
target
this.
```

The result record can help verify:

```text
what happened during the call.
```

citeturn444443search1

---

# 40. Mock Result

A mock call result can represent:

```text
return
or
throw
```

according to the runtime's mock record.

Use this to test:

```text
error propagation
```

without invoking:

```text
real failure source.
```

---

# 41. Mocking a Method

A method can be replaced so that:

```text
object.method()
```

uses:

```text
controlled implementation.
```

Always verify:

```text
restore.
```

---

# 42. `this` in Mocks

Mocks can record:

```text
this
```

which helps detect:

```text
method detached incorrectly
```

or:

```text
wrong receiver.
```

---

# 43. Property Mocking

Node's mocking facilities also support:

```text
property
getter
setter
```

replacement.

Use these carefully because:

```text
property descriptors
```

can be subtle.

---

# 44. Getter Mocking

Useful for:

```js
object.isReady
```

when the production getter:

```text
depends on current state
```

.

But consider:

```text
injecting a state provider
```

instead.

---

# 45. Setter Mocking

Useful for testing:

```text
configuration assignment
```

or:

```text
side effects.
```

Do not overuse:

```text
setter spies
```

when a direct state assertion is clearer.

---

# 46. `mock.reset()`

Reset means:

```text
restore mock behavior/state
```

according to the MockTracker semantics.

Use:

```text
per-test context mocks
```

to obtain automatic cleanup at test completion. Current Node docs describe automatic restoration for context-based mocks. citeturn444443search1

---

# 47. `mock.restoreAll()`

Current Node documentation distinguishes:

```text
mock.reset()
```

and:

```text
mock.restoreAll()
```

.

`restoreAll()` restores default behavior without disassociating tracked mocks from the MockTracker. citeturn444443search1

Understand the difference before building custom fixtures.

---

# 48. Mock Lifetime

Best pattern:

```js
test("case", context => {
  const fn =
    context.mock.fn();

  // use fn
});
```

The test context owns:

```text
mock lifecycle.
```

---

# 49. Global Mock Lifetime

Global mocks can be useful but increase:

```text
leak risk.
```

When using the global tracker extensively, explicitly restore at appropriate boundaries. Current Node docs recommend manual restoration when appropriate. citeturn444443search1

---

# 50. What Should Be Mocked?

Mock when the dependency is:

```text
expensive
nondeterministic
external
rare/failure-specific
dangerous
slow
rate-limited.
```

---

# 51. What Should Not Be Mocked?

Do not mock:

```text
your own pure function
simple value objects
algorithm under test
tiny deterministic helpers
```

just because:

```text
mocking is possible.
```

---

# 52. Over-Mocking

Over-mocked tests can prove:

```text
implementation matches itself.
```

instead of:

```text
system behavior is correct.
```

---

# 53. Mock Boundary Design

Better:

```text
OrderService
  ↓
PaymentGateway interface
```

than:

```text
OrderService
  ↓
global fetch
```

because:

```text
explicit dependency
```

is easier to replace:

```text
deterministically.
```

---

# 54. Dependency Injection

Example:

```js
function createService({
  paymentGateway
}) {
  return {
    async pay(order) {
      return paymentGateway.pay(order);
    }
  };
}
```

Test:

```js
const gateway = {
  pay: mock.fn(async () => ({
    approved: true
  }))
};
```

This minimizes:

```text
module-mocking complexity.
```

---

# 55. Test Doubles and Architecture

A codebase that requires:

```text
module patching everywhere
```

may indicate:

```text
poor dependency boundaries.
```

Testing difficulty is:

```text
architecture feedback.
```

---

# 56. Timer Mocking

Current Node test runner timer mocks can replace:

```text
setTimeout
setInterval
setImmediate
Date
```

and corresponding clear functions. Timers mocking became stable in Node v23.1.0. citeturn444443search1

---

# 57. Enabling Timers

```js
context.mock.timers.enable({
  apis: ["setTimeout"]
});
```

This can mock:

```text
Node timers
timers/promises
global timers
```

for supported APIs. citeturn444443search1

---

# 58. Fake Time

Example:

```js
context.mock.timers.enable({
  apis: ["Date"],
  now: 1000
});

assert.equal(
  Date.now(),
  1000
);
```

---

# 59. Advancing Time

```js
context.mock.timers.tick(9999);
```

moves:

```text
mocked clock
```

and can execute:

```text
due mocked timers.
```

Current Node documentation demonstrates this behavior. citeturn444443search1

---

# 60. Date + Timer Coupling

When Date is mocked with timers:

```text
Date.now()
```

and:

```text
mocked timer scheduling
```

can share:

```text
same virtual time.
```

Use this to test:

```text
time-based state machines.
```

---

# 61. `setTime()`

Current timer mocks support:

```js
context.mock.timers.setTime(...)
```

for manually moving mocked Date time.

Important current caveat:

```text
moving the mocked date does not itself execute timers
that become past-due due only to the clock jump.
```

citeturn444443search1

---

# 62. Timer Mocking Is Not a Universal Event-Loop Simulation

Fake timers simulate:

```text
selected timer APIs
```

but do not reproduce:

```text
real network
kernel scheduling
all libuv behavior
real CPU time
browser event-loop behavior.
```

Therefore:

```text
unit time test
```

and:

```text
real integration test
```

are both useful.

---

# 63. Testing Exponential Backoff

Suppose:

```text
attempt 1 → 100ms
attempt 2 → 200ms
attempt 3 → 400ms
```

Fake time can verify:

```text
exact scheduling
```

without waiting:

```text
700ms.
```

---

# 64. Testing Polling

Bad test:

```js
await sleep(10000);
```

Better:

```text
fake clock
→ tick
→ assert poll
→ tick
→ assert next poll.
```

This makes tests:

```text
fast
deterministic.
```

---

# 65. Testing Timeouts

With fake timers:

```text
operation pending
→ tick timeout
→ abort
→ assert cleanup.
```

This isolates:

```text
timeout semantics
```

from:

```text
real wall-clock variability.
```

---

# 66. Testing `AbortSignal.timeout()`

Current Node v26.1.0 added test-runner mock-timer support related to `AbortSignal.timeout`. citeturn444443search3

Use fake-time tests to verify:

```text
timeout
→ abort
→ cleanup.
```

Still keep:

```text
real integration coverage
```

for:

```text
actual network cancellation
```

where needed.

---

# 67. Destructured Timers Caveat

Current Node documentation notes that destructuring timer functions such as:

```js
import {
  setTimeout
} from "node:timers";
```

is currently not supported by the timer mocking API in the same way as supported module/global timer references. citeturn444443search1

This is a good example of:

```text
mocking mechanism
```

having:

```text
specific interception boundaries.
```

---

# 68. Module Mocking

Node's test runner provides module mocking capabilities, but current CLI documentation still classifies:

```text
--experimental-test-module-mocks
```

as:

```text
Early development
```

and notes Permission Model interaction requiring:

```text
--allow-worker
```

when module mocking is enabled under permissions. citeturn444443search0

Treat this feature as:

```text
version-sensitive
```

and verify your exact Node version.

---

# 69. Why Module Mocking Is Hard

Function mock:

```text
replace reference
```

Module mocking:

```text
replace dependency during module graph evaluation.
```

The challenge includes:

```text
import timing
module cache
ESM evaluation
CommonJS compatibility
live bindings
transitive dependencies
```

---

# 70. Module Mock Ordering

Conceptually:

```text
register mock
→ import module under test
```

not:

```text
import module
→ register mock.
```

because the dependency may already have been evaluated.

---

# 71. Module Mock Isolation

Module mocks are especially dangerous when:

```text
shared process
```

is used.

Ensure:

```text
mock lifecycle
module cache lifecycle
test lifecycle
```

align.

---

# 72. Mocking Network Calls

Prefer an application boundary:

```js
fetcher.request(...)
```

or:

```text
HTTP client adapter
```

then:

```text
fake client.
```

This is often more maintainable than:

```text
globally patching fetch.
```

---

# 73. Mocking Database

Use:

```text
repository interface
```

for unit tests.

Use:

```text
real database
```

for:

```text
integration tests.
```

Do not let:

```text
100% repository mocks
```

hide:

```text
SQL/schema/transaction bugs.
```

---

# 74. Mocking Filesystem

Unit test:

```text
business behavior
```

with:

```text
fake filesystem adapter.
```

Integration test:

```text
real fs
```

with:

```text
temporary isolated directory.
```

---

# 75. Mocking Randomness

A function like:

```js
Math.random()
```

creates nondeterminism.

Better architecture:

```js
createToken(random = Math.random)
```

and inject:

```text
deterministic fake.
```

This is preferable to:

```text
global monkey patch.
```

---

# 76. Random Seed

When tests themselves need randomness:

```text
seed
```

the generator so:

```text
failure
```

can be reproduced.

Current Node v26 test runner supports:

```bash
--test-randomize
--test-random-seed
```

for randomized test ordering. citeturn444443search0

---

# 77. Test Randomization

Randomized execution can expose:

```text
order-dependent state
```

.

A reproducible run is:

```text
randomization enabled
+
seed recorded.
```

Then replay:

```text
same seed.
```

---

# 78. Order-Dependence Bug

Test A leaves:

```text
process.env.X = "yes"
```

Test B expects:

```text
X undefined.
```

Suite passes:

```text
A then B
```

but fails:

```text
B then A
```

Randomization makes this visible.

---

# 79. `--test-randomize`

Current Node v26 supports:

```bash
node --test --test-randomize
```

which randomizes:

```text
test-file execution order
```

and:

```text
queued tests
```

within files. citeturn444443search0

---

# 80. Random Seed Reproduction

Current CLI supports:

```bash
node --test \
  --test-random-seed=1234
```

with a valid integer seed.

This allows:

```text
same randomized order
```

to be replayed. citeturn444443search0

---

# 81. Test Sharding

Large suites can be split:

```text
shard 1
shard 2
shard 3
...
```

to reduce:

```text
CI wall time.
```

Current Node CLI provides:

```text
--test-shard
```

for partitioning tests across shards. citeturn444443search0

---

# 82. Sharding Pitfall

Shards can expose:

```text
different ordering
different global state
different service load
```

.

Each shard must remain:

```text
independently executable.
```

---

# 83. Concurrency

Concurrency should be based on:

```text
CPU
memory
I/O
database capacity
network limits
test isolation.
```

More parallelism is not always:

```text
faster.
```

---

# 84. CPU-Bound Tests

For:

```text
CPU-heavy
```

tests, too much concurrency causes:

```text
CPU oversubscription
context switching
cache contention.
```

---

# 85. I/O-Bound Tests

For:

```text
I/O-heavy
```

tests, more concurrency can improve:

```text
throughput
```

until:

```text
dependency
filesystem
socket
```

saturation.

---

# 86. Shared Dependency Saturation

Suppose:

```text
100 tests
```

each open:

```text
20 DB connections.
```

Parallelism can accidentally create:

```text
2000 connections.
```

The tests now simulate:

```text
database outage.
```

rather than:

```text
application behavior.
```

---

# 87. Test Concurrency Budget

Define:

```text
CPU budget
DB connections
HTTP connections
filesystem throughput
memory
```

and schedule tests accordingly.

---

# 88. Resource Ownership

Every test-created resource needs:

```text
owner
creation
use
cleanup.
```

Examples:

```text
server
socket
worker
child process
temporary directory
database row
mock
timer.
```

---

# 89. Disposable Fixture

Good fixture:

```js
const fixture =
  await createFixture();

try {
  // test
} finally {
  await fixture.dispose();
}
```

Better:

```text
test context cleanup
```

when the test runner supports the relevant lifecycle.

---

# 90. Global Setup/Teardown

Current Node CLI supports:

```text
--test-global-setup
```

for a module evaluated before tests, used for global setup/teardown. The current feature is still classified as early development. citeturn444443search0

Use global setup for:

```text
truly shared infrastructure
```

not:

```text
test-specific state.
```

---

# 91. Global Setup Risk

If global setup creates:

```text
database
```

and tests mutate it without isolation:

```text
parallel tests interfere.
```

Global setup should create:

```text
stable infrastructure
```

while each test owns:

```text
its data.
```

---

# 92. Cleanup Failures

A cleanup error should be:

```text
visible
```

not:

```text
swallowed.
```

Otherwise CI reports:

```text
test passed
```

while:

```text
resource leaked.
```

---

# 93. Open Handles

A test process that never exits may have:

```text
server
timer
socket
worker
child process
stream.
```

The solution is:

```text
identify owner
```

not:

```text
force-exit.
```

---

# 94. Force Exit

Current Node CLI provides:

```text
--test-force-exit
```

to exit once tests finish even if the event loop remains active. citeturn444443search0

Use with extreme caution.

It can:

```text
hide resource leaks.
```

Prefer:

```text
find and close the resource.
```

---

# 95. Watch Mode

Watch mode can rerun tests after changes.

But shared resources may survive across runs depending on:

```text
process model
setup
external services.
```

Always ensure:

```text
repeated execution
```

is clean.

---

# 96. Repeated Test Stability

Run:

```text
same test
1000 times
```

to detect:

```text
timing
resource
ordering
GC
network
```

flakiness.

---

# 97. Flaky Test Definition

A flaky test:

```text
same code
same intended input
different outcome
```

without an intended input change.

---

# 98. Flake Taxonomy

```text
order
timing
concurrency
resource
environment
network
randomness
timezone
locale
external dependency
cleanup
tooling.
```

---

# 99. Timing Flake

Bad:

```js
await new Promise(r =>
  setTimeout(r, 100)
);

assert.equal(done, true);
```

This depends on:

```text
scheduler timing.
```

Use:

```text
signal/event
fake timer
deterministic completion.
```

---

# 100. Network Flake

Bad:

```text
real public API
```

for:

```text
unit test.
```

External network introduces:

```text
availability
latency
rate limits
DNS
TLS
data drift.
```

---

# 101. Integration Network Tests

For integration tests:

```text
real service
```

can be correct.

But isolate:

```text
environment
credentials
data
rate
network.
```

---

# 102. Timezone Flake

Tests that assume:

```text
UTC
```

can fail under:

```text
Asia/Kolkata
America/Los_Angeles
```

.

Set:

```text
explicit timezone
```

where tests depend on it.

---

# 103. Locale Flake

Use:

```text
explicit locale
```

for tests that depend on:

```text
Intl
sorting
number/date formatting.
```

See:

```text
Chapter 128.
```

---

# 104. Filesystem Ordering Flake

Do not assume:

```text
directory listing order
```

unless your contract specifies it.

Sort explicitly:

```js
files.sort();
```

when deterministic order is required.

---

# 105. Port Race

Test A:

```text
check port
```

then:

```text
bind.
```

Test B can bind between them.

Use:

```text
OS-assigned ephemeral ports
```

instead of:

```text
check-then-bind fixed port.
```

---

# 106. Temp Directory Race

Do not use:

```text
/tmp/test
```

for all tests.

Use:

```text
unique directory creation.
```

---

# 107. Database Race

Two tests both mutate:

```text
user id = 1.
```

Parallel execution causes:

```text
intermittent failure.
```

Use:

```text
unique fixture IDs
transactions
isolated database state.
```

---

# 108. Mock Leakage

Test A:

```text
mock global function
```

and forgets:

```text
restore.
```

Test B unexpectedly sees:

```text
mocked behavior.
```

Use:

```text
test-scoped mocks.
```

---

# 109. Timer Leakage

Test A:

```text
setInterval
```

and never clears it.

Test B:

```text
observes unexpected callback.
```

Use:

```text
context timer mocks
explicit timer ownership.
```

---

# 110. Environment Leakage

Test A:

```js
process.env.API_URL = "fake";
```

Test B:

```text
reads API_URL.
```

Isolation can be:

```text
process mode
```

or:

```text
save/restore fixture.
```

---

# 111. Module Leakage

Test A:

```text
mutates singleton
```

Test B:

```text
imports same singleton.
```

Use:

```text
fresh process
dependency injection
resettable state.
```

---

# 112. Worker Leakage

Test A creates:

```text
Worker
```

and never:

```text
terminate.
```

The suite can:

```text
hang
or
consume resources.
```

---

# 113. Child Process Leakage

A test spawns:

```text
child
```

without:

```text
wait
kill
cleanup.
```

The child may:

```text
remain
hold ports
hold files
hold process resources.
```

---

# 114. Permission Model Testing

A test may deliberately run under:

```bash
node --permission ...
```

to verify:

```text
expected deny
```

behavior.

Negative security tests are important because:

```text
application code
```

may accidentally begin requiring broader privileges after refactoring.

---

# 115. Test Runner + Permission Model

Current Node CLI notes that:

```text
module mocking
```

under the Permission Model requires:

```text
--allow-worker
```

because of the implementation requirements of test module mocking. citeturn444443search0

This is an example of:

```text
tooling capability
```

interacting with:

```text
runtime security policy.
```

---

# 116. Test Permission Matrix

Verify:

```text
normal app permissions
+
test runner permissions
```

separately.

Do not grant:

```text
production permissions
```

just to make:

```text
test infrastructure
```

work.

---

# 117. Testing Native Addons

Native tests need:

```text
binary loading
platform matrix
memory safety
worker use
shutdown
```

and:

```text
sanitizer
```

coverage where appropriate.

See:

```text
Chapter 143.
```

---

# 118. Testing HTTP Servers

A proper integration test should exercise:

```text
start
→ connect
→ request
→ response
→ close
```

rather than:

```text
call handler function
```

alone.

---

# 119. Testing TLS

Use:

```text
ephemeral test certificates
```

and verify:

```text
valid cert
expired cert
hostname mismatch
protocol mismatch
```

without:

```text
production private keys.
```

---

# 120. Testing DNS

Unit tests should:

```text
mock resolver
```

while integration tests can use:

```text
controlled DNS/service.
```

Avoid relying on:

```text
public DNS
```

for every CI run.

---

# 121. Testing Retries

Use a fake dependency:

```text
attempt 1 → fail
attempt 2 → fail
attempt 3 → succeed
```

and verify:

```text
attempt count
backoff
jitter policy
deadline.
```

---

# 122. Testing Jitter

Do not assert:

```text
exact random delay
```

if jitter is intentionally random.

Assert:

```text
within allowed range
```

and:

```text
maximum delay respected.
```

---

# 123. Testing Timeouts

Use:

```text
fake timers
```

for:

```text
unit timing.
```

Use:

```text
real slow dependency
```

for:

```text
integration timing.
```

---

# 124. Testing Cancellation

A good cancellation test:

```text
start operation
→ abort
→ assert downstream cancellation
→ assert resource cleanup
→ assert no late state mutation.
```

---

# 125. Testing Stale Async Completion

Test:

```text
request A starts
request B starts
B becomes current
A resolves late
```

Verify:

```text
A cannot overwrite B.
```

---

# 126. Testing Event-Loop Safety

For code expected to be:

```text
asynchronous/nonblocking
```

test:

```text
concurrent request remains responsive
```

while:

```text
large operation runs.
```

---

# 127. Testing Worker Concurrency

Use:

```text
multiple workers
```

and intentionally create:

```text
parallel tasks.
```

Check:

```text
result correctness
resource cleanup
worker termination.
```

---

# 128. Testing Child Processes

Verify:

```text
spawn
stdout
stderr
exit code
signal
timeout
cleanup.
```

---

# 129. Testing Signals

Signal tests should run:

```text
in isolated process.
```

Do not send:

```text
SIGTERM
SIGINT
```

to the test runner itself unintentionally.

---

# 130. Testing Shutdown

Integration test:

```text
start service
→ open request
→ send SIGTERM
→ new request rejected
→ existing request completes
→ process exits.
```

---

# 131. Testing Crash Handling

Do not crash the main CI process intentionally.

Use:

```text
child process
```

to test:

```text
fatal behavior
signal exit
report generation
```

safely.

---

# 132. Diagnostic Artifacts

For failing tests capture:

```text
stdout
stderr
test name
test file
Node version
OS
seed
shard
environment summary
```

Do not capture:

```text
secrets
tokens
full environment
```

unless explicitly sanitized.

---

# 133. Failure Fingerprint

Normalize a failure into:

```text
test
error class
stack top
Node version
platform
seed
```

This helps detect:

```text
same underlying failure
```

appearing:

```text
100 times.
```

---

# 134. Flake Detection

Track:

```text
passes
failures
retries
seed
duration
machine
shard
```

over time.

A test that fails:

```text
1/1000
```

is still:

```text
flaky.
```

---

# 135. Retry Policy for Flakes

Rerunning a failed test can:

```text
reduce transient CI noise
```

but can also:

```text
hide real flakiness.
```

Use retries as:

```text
diagnostic signal
```

not:

```text
permanent cure.
```

---

# 136. `--test-rerun-failures`

Current Node CLI supports:

```bash
node --test --test-rerun-failures=state.json
```

to persist successful/failed state between runs and rerun only the relevant failures. citeturn444443search0

This can speed:

```text
iterative debugging
```

but should not replace:

```text
full-suite validation.
```

---

# 137. Rerun Failure Pitfall

A rerun can pass because:

```text
timing changed
order changed
resource state changed.
```

Therefore record:

```text
initial failure
rerun result.
```

---

# 138. Coverage

The Node test runner can integrate with coverage collection through:

```bash
--experimental-test-coverage
```

and current CLI supports line-coverage thresholds via:

```text
--test-coverage-lines
```

which can fail the process when coverage is below the threshold. citeturn444443search0

---

# 139. Coverage Is Not Correctness

100% line coverage can still miss:

```text
wrong state
wrong output
race
security issue
```

Coverage measures:

```text
executed code
```

not:

```text
correctly tested behavior.
```

---

# 140. Branch Coverage Mindset

Even when tooling supports coverage metrics, reason about:

```text
success
failure
timeout
cancel
retry
empty
boundary
concurrency.
```

---

# 141. Mutation Testing

Mutation testing changes code such as:

```text
< → <=
true → false
```

and asks:

```text
did tests catch it?
```

This measures:

```text
test effectiveness
```

rather than:

```text
execution volume.
```

---

# 142. Test Portfolio

A production codebase should combine:

```text
unit
integration
contract
end-to-end
property
fuzz
smoke.
```

Not:

```text
one test type for everything.
```

---

# 143. Unit Test

Characteristics:

```text
fast
isolated
deterministic
narrow.
```

Good for:

```text
business logic
parsers
state machines
utility behavior.
```

---

# 144. Integration Test

Exercises:

```text
real module boundaries
real database
real filesystem
real HTTP
real TLS
```

where valuable.

---

# 145. End-to-End Test

Exercises:

```text
real application workflow
```

with:

```text
multiple system layers.
```

Slower but closer to:

```text
user reality.
```

---

# 146. Contract Test

Verifies:

```text
producer/consumer assumptions.
```

Useful for:

```text
HTTP APIs
event schemas
service contracts.
```

---

# 147. Smoke Test

Small set that answers:

```text
did the service start?
can critical path work?
```

Run:

```text
after deployment
```

when appropriate.

---

# 148. Regression Test

Captures:

```text
previously broken behavior
```

so that:

```text
same bug
```

does not return.

---

# 149. Characterization Test

Used before refactoring.

Capture:

```text
current observable behavior
```

even when:

```text
behavior is ugly.
```

Then:

```text
refactor
```

while:

```text
preserving contract.
```

---

# 150. Test Runner Reporters

Current Node test runner supports reporters, including reporter selection and destination through CLI options such as:

```text
--test-reporter
--test-reporter-destination
```

and custom reporter interfaces. citeturn444443search0

---

# 151. Reporter Responsibilities

A reporter can transform:

```text
test events
```

into:

```text
CI output
JUnit XML
JSON
dashboard events
metrics.
```

---

# 152. Machine vs Human Output

Human:

```text
readable failures
```

Machine:

```text
stable structured schema.
```

Do not force:

```text
CI
```

to parse:

```text
colored console output.
```

---

# 153. Test Event Stream

Programmatic test execution can expose structured test lifecycle events.

Conceptually:

```text
test:start
test:pass
test:fail
diagnostic
```

Use these to build:

```text
custom analytics
```

without:

```text
scraping logs.
```

---

# 154. Test Diagnostics

A test can emit diagnostics useful for:

```text
debugging
operator context
failure analysis.
```

Keep diagnostics:

```text
structured
small
safe.
```

---

# 155. Test Runner Programmatic API

The test module also exposes programmatic runner capabilities through Node's test module APIs.

Use programmatic execution when you need:

```text
custom orchestration
embedded test execution
special CI tooling
```

rather than:

```text
shell-only execution.
```

---

# 156. Test Runner Configuration

Possible dimensions include:

```text
concurrency
isolation
glob
name pattern
sharding
coverage
reporter
watch
randomization.
```

Treat test execution configuration as:

```text
part of test architecture.
```

---

# 157. CI Architecture

A scalable CI pipeline:

```text
lint
 ↓
unit tests
 ↓
integration tests
 ↓
contract tests
 ↓
e2e
 ↓
artifacts
 ↓
release gate
```

Parallelize:

```text
independent stages.
```

---

# 158. CI Sharding Strategy

For a suite with:

```text
10,000 tests
```

use:

```text
N shards
```

but balance by:

```text
historical duration
```

rather than:

```text
file count alone.
```

---

# 159. Duration-Aware Sharding

If:

```text
Shard A = 1000 fast tests
Shard B = 1000 slow tests
```

test count is balanced:

```text
workload is not.
```

Build:

```text
historical-duration-aware allocation.
```

---

# 160. Test Duration Database

Record:

```text
test ID
duration
pass/fail
machine
Node version
commit.
```

Use it to:

```text
optimize shard balance.
```

---

# 161. Slow Test Budget

Tag:

```text
slow
```

tests.

Track:

```text
count
duration
trend.
```

Do not allow:

```text
unit suite
```

to slowly become:

```text
integration suite.
```

---

# 162. CI Environment Reproducibility

Pin:

```text
Node version
OS image
timezone
locale
package lockfile
external service versions
```

where needed.

---

# 163. Node Version Matrix

For libraries:

```text
minimum supported Node
latest LTS/current supported
```

test:

```text
all supported runtimes.
```

For applications:

```text
production Node version
```

must always be tested.

---

# 164. Test Runner Version Drift

Node test runner features evolve.

Current Node v26 contains:

```text
randomization
rerun failures
tag filters
```

while some newer mock/module features remain:

```text
early development
```

.

Therefore:

```text
pin test-runtime expectations.
```

---

# 165. Permission-Aware CI

If production starts with:

```text
--permission
```

CI should include:

```text
hardened execution.
```

Otherwise:

```text
test passes
```

may hide:

```text
production permission denial.
```

---

# 166. Test Secrets

Never let CI tests use:

```text
real production API keys
```

unless:

```text
exceptionally controlled security tests
```

require them.

Prefer:

```text
fake
sandbox
ephemeral
```

credentials.

---

# 167. Test Network Isolation

Where possible:

```text
deny arbitrary outbound network
```

for unit tests.

This catches:

```text
hidden dependency
telemetry call
unexpected API request.
```

---

# 168. Test Filesystem Isolation

A strong CI container can use:

```text
read-only project filesystem
```

and:

```text
dedicated temp directories.
```

This exposes:

```text
unexpected writes.
```

---

# 169. Test Runtime Permissions

Combine:

```text
Node Permission Model
+
container filesystem
+
network policy
```

to verify:

```text
least-privilege assumptions.
```

---

# 170. Test Native Dependencies

Native tests should record:

```text
Node version
OS
architecture
N-API version
compiler
```

for reproducibility.

---

# 171. Test Workers

Tests involving workers should verify:

```text
worker created
worker completes
worker errors propagate
worker terminates
```

.

---

# 172. Test Child Processes

A child-process test should use:

```text
explicit executable path
controlled environment
timeout
cleanup.
```

Avoid:

```text
shell command strings
```

with:

```text
test-generated untrusted data.
```

---

# 173. Test Signals in Child Processes

Wrap signal scenarios in:

```text
child process
```

and assert:

```text
exit signal
report
cleanup.
```

---

# 174. Test Diagnostic Reports

For controlled failures:

```text
generate report
```

then assert:

```text
artifact exists
contains expected metadata
is cleaned up.
```

Do not:

```text
store reports forever.
```

---

# 175. Test Inspector Access

Security tests should verify:

```text
production process
```

does not unexpectedly expose:

```text
Inspector port.
```

---

# 176. Test Permission Denials

Example:

```text
service attempts write
permission denied
```

assert:

```text
specific error class
safe user behavior
no retry loop
```

---

# 177. Test Resource Exhaustion

Use:

```text
bounded local fixtures
```

to simulate:

```text
out of sockets
full queue
oversized file
memory pressure.
```

Never intentionally exhaust the entire CI host.

---

# 178. Test Event-Loop Stall

A test can detect:

```text
operation takes too long
```

using:

```text
performance
```

or:

```text
event-loop monitoring.
```

But avoid making:

```text
tight timing thresholds
```

that vary wildly across machines.

---

# 179. Test Timing Tolerance

Bad:

```text
assert duration < 5ms
```

on shared CI.

Better:

```text
assert behavior
```

and separately:

```text
benchmark performance
```

in controlled environments.

---

# 180. Test Performance vs Unit Tests

Do not make every unit test:

```text
microbenchmark.
```

Use:

```text
performance suite
```

for:

```text
latency/throughput regressions.
```

---

# 181. Test Memory

Use repeated workload:

```text
N iterations
```

and inspect:

```text
RSS
heap
external
```

for coarse regressions.

Detailed leak tests should use:

```text
diagnostic tooling
```

outside ordinary:

```text
every-commit unit suite.
```

---

# 182. Test GC-Sensitive Code

Do not assert:

```text
object is collected
```

purely from normal application behavior.

GC is:

```text
nondeterministic.
```

Use:

```text
weak references/finalization
diagnostic experiments
```

only where the contract actually requires them.

---

# 183. Test Finalizers

Finalizers should be:

```text
leak fallback
```

not:

```text
primary cleanup mechanism.
```

Test:

```text
explicit close
```

more strongly than:

```text
eventual GC.
```

---

# 184. Test Mock Restoration

A useful test harness verifies:

```text
mock installed
→ test
→ mock restored.
```

Then run:

```text
neighboring test
```

to detect:

```text
leak.
```

---

# 185. Test Isolation Self-Test

Build a deliberate test:

```text
test A mutates global
test B expects default
```

Run with:

```text
process isolation
```

then:

```text
none.
```

The difference teaches:

```text
why isolation exists.
```

---

# 186. Test Order Self-Test

Create:

```text
A → leaves state
B → fails if state exists.
```

Run:

```text
normal order
randomized
fixed seed.
```

---

# 187. Test Concurrency Self-Test

Create:

```text
shared counter
```

and run:

```text
parallel subtests.
```

Observe:

```text
race/incorrectness.
```

Then add:

```text
isolated state
```

and verify:

```text
deterministic results.
```

---

# 188. Test Database Concurrency

Create:

```text
two parallel tests
```

that try to:

```text
update same row.
```

Observe:

```text
lock/contention/order behavior.
```

Then design:

```text
unique fixture ownership.
```

---

# 189. Test Filesystem Concurrency

Two tests write:

```text
same filename.
```

Race:

```text
corruption.
```

Replace with:

```text
test-specific directory.
```

---

# 190. Test Port Concurrency

Two tests use:

```text
same fixed port.
```

One gets:

```text
EADDRINUSE.
```

Replace with:

```text
ephemeral ports.
```

---

# 191. Test Retry Concurrency

Two retries overlap:

```text
attempt 1
attempt 2
```

and both mutate:

```text
same state.
```

Test:

```text
idempotency
cancellation
```

rather than:

```text
assuming retries are sequential.
```

---

# 192. Mocking vs Contract Tests

A unit mock can prove:

```text
your code handles response X.
```

Only a contract/integration test can prove:

```text
real provider produces response X with compatible shape.
```

Use both when risk justifies it.

---

# 193. Mocking vs Integration Fidelity

Too many mocks create:

```text
high unit coverage
low real-system confidence.
```

The test portfolio should maximize:

```text
confidence per execution cost.
```

---

# 194. Test Confidence Model

```text
Unit
→ local correctness

Integration
→ boundary correctness

Contract
→ interface correctness

E2E
→ workflow correctness

Production telemetry
→ operational correctness.
```

---

# 195. Tests as Executable Specifications

A good test states:

```text
Given
When
Then
```

without requiring:

```text
implementation details.
```

Example:

```text
Given an expired token
When request is made
Then server returns 401
and no downstream business action occurs.
```

---

# 196. Test Names as Documentation

Prefer:

```text
does not retry a non-idempotent payment
```

over:

```text
payment retry test.
```

The former encodes:

```text
business contract.
```

---

# 197. Assertion Granularity

Bad:

```js
assert.deepEqual(
  entireHugeObject,
  expectedHugeObject
);
```

when only:

```text
three fields
```

matter.

Prefer:

```text
contract-level assertions
```

that survive:

```text
unrelated refactors.
```

---

# 198. Snapshot Testing

Snapshots can quickly detect:

```text
large output changes
```

but may hide:

```text
review burden
```

when snapshots become:

```text
“update snapshot”
```

instead of:

```text
understand behavior.
```

---

# 199. Mock Snapshot Risks

Do not snapshot:

```text
random IDs
timestamps
machine paths
platform-specific data
```

without normalization.

---

# 200. Deterministic Test Data

Use:

```text
factory functions
fixtures
fixed IDs
fixed clock
fixed random seed
```

for reproducibility.

---

# 201. Fixture Factories

Example:

```js
function makeUser(overrides = {}) {
  return {
    id: "user-1",
    email: "test@example.com",
    ...overrides
  };
}
```

Factories reduce:

```text
copy/paste fixture drift.
```

---

# 202. Test Builders

For complex domains:

```js
new OrderBuilder()
  .withCustomer(...)
  .withItems(...)
  .build();
```

Builders clarify:

```text
test intent
```

and hide:

```text
irrelevant setup.
```

---

# 203. Test Data Ownership

Never let one test mutate a:

```text
shared fixture
```

that another test uses.

Prefer:

```text
fresh instance
```

or:

```text
immutable fixture.
```

---

# 204. Immutability in Tests

Frozen fixtures:

```js
Object.freeze(data);
```

can catch:

```text
accidental mutation.
```

Use selectively because:

```text
deep freeze
```

can add:

```text
setup overhead.
```

---

# 205. Test Helper Quality

A helper should:

```text
reduce boilerplate
```

without:

```text
hiding behavior.
```

Bad:

```text
setupEverything()
```

Better:

```text
createAuthenticatedUser()
```

with:

```text
clear semantics.
```

---

# 206. Helper Failure Diagnostics

When a helper fails, error should show:

```text
which logical fixture
what resource
what context
```

not:

```text
generic “setup failed”.
```

---

# 207. Async Test Completion

Prefer:

```js
await operation();
```

or:

```text
return Promise
```

over:

```text
fire-and-forget.
```

Otherwise test may:

```text
finish early.
```

---

# 208. Missing Await Bug

Bad:

```js
test("saves user", () => {
  repository.save(user);
  assert.equal(repository.saved, true);
});
```

The test may assert:

```text
before async operation completes.
```

---

# 209. Rejected Promise Test

Use:

```js
await assert.rejects(
  operation(),
  /invalid/
);
```

rather than:

```js
try {
  await operation();
} catch {
}
```

without:

```text
assertion that failure actually occurred.
```

---

# 210. False Positive Test

Bad:

```js
try {
  await operation();
} catch {
  return;
}
```

If operation succeeds:

```text
test still passes incorrectly.
```

Use:

```text
assert.rejects.
```

---

# 211. Timeouts

Every integration-style test should have:

```text
reasonable timeout
```

especially when it can hang due to:

```text
network
worker
child process
socket.
```

Do not set:

```text
1 hour
```

just to avoid intermittent failure.

---

# 212. Timeout Ownership

If a test times out:

```text
which resource failed to finish?
```

Use:

```text
diagnostics
```

to answer:

```text
worker
socket
promise
child
fixture.
```

---

# 213. Test Cancellation

When a test is aborted:

```text
stop started work
```

where possible.

Otherwise:

```text
test runner moves on
```

while:

```text
old work continues.
```

---

# 214. Test Abort Signal

A fixture should accept:

```js
{ signal }
```

when appropriate.

Then:

```text
test cancellation
→ dependency cancellation.
```

---

# 215. Resource Leak Detection

Useful checks:

```text
open sockets
active workers
child processes
timers
temp directories
```

at suite boundaries.

Use:

```text
documented APIs
```

first.

---

# 216. Don't Depend on Undocumented Internals

Internal APIs can change between:

```text
Node releases.
```

Prefer:

```text
stable public test/diagnostic APIs.
```

Use internals only when:

```text
test infrastructure is explicitly pinned.
```

---

# 217. Test Runner Diagnostics

Current Node releases continue to evolve the test runner.

Node v26.1.0 added test event identifiers and additional suite-context diagnostics, while later v26 releases fixed isolation and rerun behavior. citeturn444443search3turn444443search5

This means:

```text
test infrastructure itself
```

should be:

```text
version-aware.
```

---

# 218. Current Test Runner Capabilities

As of September 2026, current Node v26 documentation includes:

```text
stable test runner
process test isolation
test randomization
random seeds
test sharding
rerun failures
reporters
test coverage integration
timer mocks
mock trackers
```

while some newer facilities such as:

```text
module mocking
tag filtering
global setup
```

remain early-development/experimental according to the current CLI documentation. citeturn444443search0turn444443search2

---

# 219. Programmatic Test Execution

A custom runner can configure:

```text
files
concurrency
isolation
timeout
reporters
coverage.
```

Use when:

```text
organization-wide test infrastructure
```

needs:

```text
standardized policy
```

across repositories.

---

# 220. Test Platform Architecture

```text
Repository
   ↓
Test Definition
   ↓
Node Test Runner
   ↓
Scheduler
   ↓
Isolation
   ↓
Fixtures/Mocks
   ↓
System Under Test
   ↓
Diagnostics
   ↓
Reporter
   ↓
CI Artifact Store
```

---

# 221. Centralized vs Repository-Owned Fixtures

Centralized:

```text
consistent
```

but can create:

```text
hidden coupling.
```

Repository-owned:

```text
local
explicit
```

but can create:

```text
duplication.
```

Prefer:

```text
small shared primitives
+
domain-owned fixtures.
```

---

# 222. Test Platform API

Create helpers such as:

```js
withTempDir()
withServer()
withDatabase()
withEnv()
withTimeout()
withWorker()
```

Each helper should provide:

```text
resource
+
cleanup
+
diagnostics.
```

---

# 223. Example Fixture Contract

```js
const server =
  await withServer(handler);

try {
  await testLogic(server.url);
} finally {
  await server.close();
}
```

The fixture owns:

```text
server lifecycle.
```

---

# 224. Composable Fixtures

Prefer:

```text
withEnv
  → withTempDir
      → withServer
```

over:

```text
one giant bootstrap fixture.
```

Composition makes:

```text
ownership
```

explicit.

---

# 225. Fixture Failure Reporting

If cleanup fails, include:

```text
fixture name
resource
original test
cleanup error.
```

This avoids:

```text
“afterEach failed”
```

with no context.

---

# 226. Deterministic HTTP Fixture

Build:

```text
ephemeral port
known routes
request logs
shutdown
```

and expose:

```js
{
  url,
  requests,
  close
}
```

---

# 227. Deterministic TLS Fixture

Build:

```text
ephemeral TLS server
test certificate
known protocol
close.
```

Do not:

```text
download certificates during every test.
```

---

# 228. Deterministic DNS Fixture

Use:

```text
fake resolver
or
controlled local DNS.
```

Avoid:

```text
public DNS.
```

---

# 229. Deterministic Clock Fixture

Wrap:

```text
context.mock.timers
```

with:

```js
clock.advance(ms);
clock.now();
clock.reset();
```

so business tests do not:

```text
know runner-specific APIs.
```

---

# 230. Deterministic Randomness Fixture

Expose:

```js
random.next();
```

from:

```text
seeded PRNG.
```

This allows:

```text
repeatable simulations.
```

---

# 231. Deterministic IDs

Avoid:

```text
Date.now()
+
Math.random()
```

for:

```text
test fixture identity.
```

Use:

```text
incrementing fixture IDs
```

or:

```text
seeded generator.
```

---

# 232. Test Correlation IDs

Give each test:

```text
testId
```

and resource:

```text
fixtureId.
```

Include them in:

```text
logs
database records
HTTP headers
```

during integration tests.

---

# 233. Test Logging

Avoid:

```text
console.log
```

flooding CI.

Prefer:

```text
failure-only diagnostics
```

and:

```text
structured events.
```

---

# 234. Test Tracing

For distributed integration tests:

```text
test
→ service
→ dependency
```

propagate:

```text
test trace ID.
```

This links:

```text
CI failure
```

to:

```text
runtime traces.
```

---

# 235. Failure Artifact Bundle

A failure artifact can include:

```text
test result
seed
Node version
commit
logs
trace
server output
report
screenshot where relevant.
```

Keep:

```text
sensitive data
```

redacted.

---

# 236. Flake Dashboard

Metrics:

```text
test executions
failures
retries
flake rate
median duration
p95 duration
failure fingerprint.
```

---

# 237. Flake Ownership

Every persistent flaky test needs:

```text
owner
issue
deadline
root-cause hypothesis.
```

Otherwise:

```text
retry becomes permanent infrastructure debt.
```

---

# 238. Test Debt

Track:

```text
slow tests
flaky tests
disabled tests
focused tests
skipped tests
uncovered critical paths
```

Treat them as:

```text
engineering debt.
```

---

# 239. Skips

A skip should include:

```text
reason
issue/reference
expected removal condition.
```

Bad:

```js
test.skip("works later");
```

---

# 240. Todo Tests

Use:

```text
todo
```

for:

```text
known missing coverage
```

without pretending:

```text
behavior is verified.
```

---

# 241. Test Security

Tests can become attack surfaces because they may execute:

```text
native addons
shell commands
network calls
filesystem access
secrets.
```

Treat CI as:

```text
privileged infrastructure.
```

---

# 242. Test Runner + Secrets

Never print:

```text
process.env
```

to debug a failure.

Redact:

```text
tokens
passwords
API keys
certificates
private keys.
```

---

# 243. Test Runner + Permission Model

A hardened CI can run:

```bash
node \
  --permission \
  --allow-fs-read=... \
  --allow-fs-write=... \
  --allow-net=... \
  --test
```

to make:

```text
unexpected authority
```

visible during development.

---

# 244. Test Runner + Native Addons

If native addons are required:

```text
--allow-addons
```

may be necessary under Permission Model.

But do not broaden permissions:

```text
globally
```

for tests that do not require native code.

---

# 245. Test Isolation + Security

Process isolation also limits:

```text
blast radius
```

of:

```text
bad fixture
native crash
global mutation
```

.

It is not:

```text
complete security sandbox.
```

---

# 246. Test Failure Semantics

A failure can be:

```text
assertion
exception
timeout
abort
process exit
signal
resource leak
cleanup failure
```

Classify:

```text
cause
```

rather than:

```text
all = assertion failure.
```

---

# 247. Error Classification

Example:

```text
AssertionError
→ expected behavior mismatch

ERR_ACCESS_DENIED
→ permission configuration

ECONNREFUSED
→ network fixture

ETIMEDOUT
→ timeout

SIGSEGV
→ native/runtime failure

test timeout
→ lifecycle/resource problem
```

---

# 248. Test Exit Codes

CI depends on:

```text
process exit code
```

.

Ensure custom test tooling preserves:

```text
failure semantics.
```

---

# 249. Custom Reporter Exit Behavior

A reporter should:

```text
format results
```

not:

```text
silently convert failed tests to success.
```

Keep:

```text
execution result
```

separate from:

```text
presentation.
```

---

# 250. Test Result Schema

A durable result model:

```js
{
  testId,
  name,
  file,
  status,
  durationMs,
  error,
  attempt,
  seed,
  shard,
  nodeVersion,
  platform
}
```

This supports:

```text
CI analytics
flake detection
duration analysis.
```

---

# 251. Implementation From Scratch — Test Fixture Toolkit

Build:

```text
TestFixtureManager
TempDirFixture
HttpServerFixture
TlsServerFixture
EnvFixture
WorkerFixture
ChildProcessFixture
ClockFixture
RandomFixture
DatabaseFixture
```

Each exposes:

```text
create()
resource
dispose()
diagnostics()
```

---

# 252. Implementation Milestone 1 — Temp Directory

Implement:

```js
const dir =
  await fs.mkdtemp(
    path.join(tmpdir(), "case-")
  );
```

Then:

```text
cleanup recursively.
```

---

# 253. Implementation Milestone 2 — Environment Fixture

Capture:

```text
old value
```

then:

```text
set new
```

and restore:

```text
old/undefined
```

in finally/cleanup.

---

# 254. Implementation Milestone 3 — Server Fixture

Build:

```text
listen on port 0
await listening
return url
close on disposal.
```

---

# 255. Implementation Milestone 4 — Child Fixture

Implement:

```text
spawn
wait for ready
capture stdout/stderr
timeout
terminate
await exit.
```

---

# 256. Implementation Milestone 5 — Worker Fixture

Implement:

```text
spawn
wait ready
send message
await response
terminate
verify exited.
```

---

# 257. Implementation Milestone 6 — Clock Fixture

Wrap:

```text
context.mock.timers
```

with:

```text
enable
advance
setTime
reset.
```

---

# 258. Implementation Milestone 7 — Random Fixture

Implement:

```text
seeded PRNG
```

and expose:

```text
next()
integer()
bytes()
```

for deterministic test data.

---

# 259. Implementation Milestone 8 — HTTP Fixture

Add:

```text
request capture
request IDs
automatic cleanup
failure diagnostics.
```

---

# 260. Implementation Milestone 9 — DB Fixture

Implement:

```text
transaction
rollback
```

or:

```text
unique schema
```

based on the database.

---

# 261. Implementation Milestone 10 — Failure Bundle

On failure:

```text
collect logs
fixture state
seed
version
```

and store:

```text
artifact.
```

---

# 262. Implementation Milestone 11 — Flake Recorder

Record:

```text
test
attempt
result
seed
duration
```

and compute:

```text
flake rate.
```

---

# 263. Implementation Milestone 12 — Shard Planner

Build a simple algorithm:

```text
sort tests by historical duration
→ assign next test to currently lightest shard.
```

This is:

```text
bin packing heuristic.
```

---

# 264. Debugging Exercises

## Exercise A — Order Flake

Create:

```text
global mutation.
```

Use:

```text
--test-randomize
```

to discover it.

---

## Exercise B — Timer Flake

Replace:

```text
real sleep
```

with:

```text
timer mock.
```

---

## Exercise C — Port Collision

Create:

```text
parallel fixed-port servers.
```

Observe:

```text
EADDRINUSE.
```

Fix:

```text
port 0.
```

---

## Exercise D — Worker Leak

Create:

```text
worker
```

without:

```text
terminate.
```

Observe:

```text
suite hang.
```

Fix:

```text
fixture cleanup.
```

---

## Exercise E — Environment Leak

Test A sets:

```text
MODE=A
```

Test B expects:

```text
MODE=undefined.
```

Fix:

```text
scope/restore.
```

---

## Exercise F — Mock Leak

Install:

```text
global mock
```

and forget:

```text
restore.
```

Run multiple tests.

Fix with:

```text
context.mock.
```

---

## Exercise G — Module Mock Ordering

Import dependency before:

```text
mock registration
```

and observe:

```text
mock does not affect already-evaluated module.
```

Reorder:

```text
register
→ import.
```

---

## Exercise H — Retry Flake

Create:

```text
dependency fails once.
```

Test:

```text
retry.
```

Use:

```text
fake time
```

to verify:

```text
exact backoff semantics.
```

---

## Exercise I — Test Isolation

Run:

```bash
node --test --test-isolation=process
```

then:

```bash
node --test --test-isolation=none
```

with deliberately stateful tests.

Compare behavior. citeturn444443search0

---

# 265. Code Review Exercise — Sleepy Test

Review:

```js
test("eventual result", async () => {
  startWork();

  await new Promise(r =>
    setTimeout(r, 5000)
  );

  assert.equal(done, true);
});
```

Find:

```text
real-time dependency
slow execution
scheduler flake
poor synchronization
```

Replace with:

```text
event-driven completion
or
mocked timers.
```

---

# 266. Code Review Exercise — Shared Temp File

```js
const file =
  "/tmp/test-result.json";

test("A", async () => {
  await writeFile(file, "A");
});

test("B", async () => {
  await writeFile(file, "B");
});
```

Find:

```text
parallel collision
filesystem race
cleanup ambiguity.
```

---

# 267. Code Review Exercise — Global Mock

```js
test("payment", () => {
  fetch = mock.fn(async () => ({
    ok: true
  }));
});
```

Find:

```text
global mutation
no restoration
possible test-order dependence.
```

---

# 268. Code Review Exercise — Forced Exit

```bash
node --test --test-force-exit
```

added because:

```text
suite hangs.
```

Question:

```text
What bug could this hide?
```

Answer:

```text
open resources
timers
workers
sockets
children
```

that should be cleaned explicitly. citeturn444443search0

---

# 269. Code Review Exercise — Infinite Retry Test

```js
test("retry", async () => {
  while (!success) {
    await wait(1000);
  }
});
```

Problems:

```text
unbounded
slow
hang risk
no timeout
no deterministic clock.
```

---

# 270. Predict-the-Behavior Exercises

### Exercise 1

Two test files run under:

```text
process isolation.
```

Test A modifies:

```text
globalThis.x.
```

Predict:

```text
whether Test B sees the same JS global state.
```

---

### Exercise 2

Run with:

```text
--test-isolation=none
```

The same mutation occurs.

Predict:

```text
what can change.
```

---

### Exercise 3

A test enables:

```js
context.mock.timers.enable({
  apis: ["setTimeout"]
});
```

and schedules:

```js
setTimeout(fn, 1000);
```

Immediately inspect:

```text
fn.mock.callCount()
```

Predict:

```text
0
```

until:

```text
tick(1000).
```

citeturn444443search1

---

### Exercise 4

Mock Date:

```js
context.mock.timers.enable({
  apis: ["Date"],
  now: 100
});
```

Then:

```js
context.mock.timers.tick(200);
```

Predict:

```text
Date.now() === 300.
```

citeturn444443search1

---

### Exercise 5

A test sets:

```js
process.env.X = "A";
```

and does not restore it.

Predict:

```text
why a no-isolation suite may make another test fail.
```

---

### Exercise 6

A test suite uses:

```text
--test-randomize
```

and prints:

```text
seed = 123
```

Predict:

```text
why recording the seed makes order-related failures reproducible.
```

citeturn444443search0

---

### Exercise 7

A timer mock calls:

```js
setTime(...)
```

and there is a timer that would now be in the past.

Predict:

```text
whether setTime() itself executes the past timer.
```

Current Node docs say it does not. citeturn444443search1

---

### Exercise 8

A module is imported before a module mock is registered.

Predict:

```text
why the mock may not affect the already-evaluated dependency.
```

---

# 271. Interview Questions

### Test Runner

```text
1. Why use node:test?
2. How does node --test discover and execute tests?
3. What are hooks?
4. What are subtests?
5. What is TestContext?
```

### Isolation

```text
6. What is test isolation?
7. What is process isolation?
8. What happens with --test-isolation=none?
9. Why can shared process execution be faster?
10. Why can it be dangerous?
```

### Mocking

```text
11. What is the difference between a mock and a fake?
12. When should you use a spy?
13. How do you inspect mock calls?
14. How do you restore mocks?
15. Why are context-scoped mocks useful?
```

### Timers

```text
16. How do fake timers work?
17. How do you test retry backoff?
18. How do you test timeout logic?
19. What is the Date/timer relationship in Node's mock timers?
20. What limitations do fake timers have?
```

### Module Mocking

```text
21. Why is module mocking harder than function mocking?
22. Why does import timing matter?
23. Why can module cache matter?
24. When would you prefer dependency injection?
```

### Flakiness

```text
25. What causes flaky tests?
26. How would you diagnose order dependence?
27. How would you diagnose a timing race?
28. Why can test retries hide flakiness?
29. How would you reproduce a randomized failure?
```

### CI

```text
30. What is test sharding?
31. How would you balance shards?
32. What should a test artifact contain?
33. How would you measure flaky-test rate?
34. How would you protect CI secrets?
```

### Principal

```text
35. Design a test platform for 50,000 tests.
36. How would you choose process isolation vs none?
37. How would you design fixture ownership?
38. How would you test real databases without flaky shared state?
39. How would you design safe mocks for a large Node monorepo?
40. How would you debug a test suite that passes locally but flakes in CI?
41. How would you integrate Permission Model into testing?
42. How would you test native addons without destabilizing the runner?
43. How would you build a flaky-test detection system?
```

---

# 272. Mastery Exercises

### Exercise 1 — Isolated Test Toolkit

Build:

```text
withTempDir
withEnv
withServer
withWorker
withChildProcess
```

with:

```text
automatic cleanup.
```

### Exercise 2 — Deterministic Clock

Build:

```text
clock.now()
clock.advance()
clock.set()
clock.reset()
```

using:

```text
Node test timer mocks.
```

### Exercise 3 — Retry Test Suite

Test:

```text
exponential backoff
jitter
deadline
cancellation
idempotency.
```

### Exercise 4 — Flaky Suite

Introduce:

```text
global leak
timer leak
port collision
database collision
```

and detect each.

### Exercise 5 — Randomized Suite

Run:

```text
--test-randomize
```

record seeds, reproduce failures.

### Exercise 6 — Sharded CI

Take:

```text
1000 tests
```

and distribute across:

```text
4 shards
```

based on:

```text
historical duration.
```

### Exercise 7 — Permission Tests

Run tests under:

```text
Node Permission Model
```

and verify:

```text
expected allowed/denied operations.
```

### Exercise 8 — Native Test Boundary

Run:

```text
native addon
```

inside child process integration tests so:

```text
native crash
```

does not destabilize the entire test runner.

### Exercise 9 — Diagnostic Test Platform

Create:

```text
failure bundle
```

containing:

```text
test
seed
Node
OS
logs
trace
duration
attempt
shard.
```

---

# 273. Track A — Core Theory

Master:

```text
node:test
TestContext
hooks
subtests
concurrency
process isolation
none isolation
mock functions
mock properties
mock timers
mock Date
module mocking
dependency injection
flaky tests
randomization
seeds
sharding
rerun failures
coverage
reporters
test events
fixtures
cleanup
CI.
```

Deliverable:

```text
explain exactly how test execution, mocking, scheduling,
isolation, and cleanup interact.
```

---

# 274. Track B — Implementation

Build:

```text
test fixture toolkit
clock abstraction
randomness abstraction
HTTP fixture
TLS fixture
worker fixture
child-process fixture
database fixture
failure artifact collector
flake recorder
shard planner
permission-aware CI harness.
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

# 275. Track C — Interview / Reasoning

Practice:

```text
“Why does process isolation reduce test leakage?”

“Why is shared-process test execution risky?”

“How do fake timers improve determinism?”

“Why is module mocking harder than function mocking?”

“How do you distinguish flakiness from a real intermittent bug?”

“How do you reproduce a randomized failure?”

“How do you design fixture ownership?”

“How do you choose CI shard count?”

“How would you test native code safely?”
```

Deliverable:

```text
test risk
+
isolation boundary
+
determinism
+
resource ownership
+
diagnostics
+
trade-off.
```

---

# 276. Principal Decision Framework

For every test architecture ask:

```text
1. What behavior is being verified?
2. What is the smallest realistic test boundary?
3. Which dependencies should be real?
4. Which dependencies should be replaced?
5. Where is the deterministic seam?
6. What state exists?
7. Who owns that state?
8. How is state reset?
9. Can tests run concurrently?
10. If not, why not?
11. What isolation mode is required?
12. What resources can leak?
13. What happens on timeout?
14. What happens on cancellation?
15. What happens on failure during cleanup?
16. Can the failure be reproduced with a seed?
17. Can the failure be replayed on another machine?
18. What artifacts are captured?
19. How much retry is acceptable?
20. Does retry hide a real bug?
21. How is the suite sharded?
22. How is shard balance measured?
23. What Node versions are supported?
24. Which runner features are stable vs early development?
25. Does Permission Model change test capabilities?
26. Do native addons require isolated processes?
27. Are CI secrets protected?
28. What is the long-term maintenance cost?
```

---

# 277. Production Test Checklist

```text
[ ] node:test stable baseline documented
[ ] Node version pinned
[ ] discovery command standardized
[ ] test naming standardized
[ ] hooks reviewed
[ ] fixture ownership documented
[ ] process isolation chosen intentionally
[ ] shared-process execution reviewed
[ ] concurrency reviewed
[ ] database isolation implemented
[ ] temp-dir isolation implemented
[ ] port isolation implemented
[ ] mock lifetime controlled
[ ] timer mocking standardized
[ ] module mocking minimized
[ ] dependency injection preferred where useful
[ ] retries bounded
[ ] randomization available
[ ] random seeds recorded
[ ] sharding implemented
[ ] shard balancing measured
[ ] failure artifacts stored
[ ] secrets redacted
[ ] flaky tests tracked
[ ] skipped tests tracked
[ ] slow tests tracked
[ ] permission-aware testing present
[ ] native tests isolated where necessary
[ ] cleanup verified
[ ] force-exit not used as leak workaround
```

---

# 278. Test Determinism Checklist

```text
[ ] time controlled
[ ] random seed controlled
[ ] timezone controlled
[ ] locale controlled
[ ] unique ports
[ ] unique files
[ ] unique database records
[ ] external network controlled
[ ] environment restored
[ ] mocks restored
[ ] workers terminated
[ ] child processes terminated
[ ] sockets closed
[ ] timers cleared
[ ] module state isolated
```

---

# 279. Flake Prevention Checklist

```text
[ ] no real sleep unless explicitly testing timing
[ ] no shared mutable globals
[ ] no fixed ports
[ ] no fixed temp files
[ ] no shared mutable DB rows
[ ] no random external APIs
[ ] no test-order assumptions
[ ] no timezone assumptions
[ ] no locale assumptions
[ ] no machine-specific paths
[ ] no implicit process environment
[ ] no leaked mocks
[ ] no leaked workers
```

---

# 280. Current Platform Notes

As of September 2026:

```text
Node.js v26.8.2 is the current documentation baseline
used by this chapter.

Test Runner:
Stable.

Mock Timers:
Stable since Node v23.1.0.

Test Isolation:
--test-isolation
uses process isolation by default when --test is active;
none mode keeps files in the same runner process.

Randomization:
--test-randomize
and
--test-random-seed
are current Node v26 features.

Sharding:
--test-shard is available.

Rerun failures:
--test-rerun-failures is available.

Coverage:
available through the test coverage integration,
with experimental status in the CLI documentation.

Module mocking:
--experimental-test-module-mocks
remains early development.

Tag filtering:
--experimental-test-tag-filter
remains early development.

Global setup:
--test-global-setup
remains early development.

Timers:
mock setTimeout/setInterval/setImmediate/Date
through the test runner mock infrastructure.

Permission Model:
module mocking may require --allow-worker.

Test runner behavior can continue to change across
Node releases, so pin the exact Node version for CI.
```

These current Node v26 details are supported by the official CLI, test-runner, documentation, and release notes. citeturn444443search0turn444443search1turn444443search2turn444443search3

---

# 281. Specification / Runtime Source Discipline

Primary sources:

```text
Node.js Test Runner documentation
Node.js CLI documentation
Node.js Assertion documentation
Node.js Timers / MockTimers documentation
Node.js module mocking documentation
Node.js Diagnostics Channel documentation
Node.js Worker documentation
Node.js Permission Model documentation
```

Maintain explicit distinctions:

```text
stable
experimental
early development
deprecated
runtime-specific
```

Current Node documentation classifies the Test Runner as stable and documents test-randomization, isolation, rerun failures, sharding, reporters, and test coverage options. citeturn444443search0turn444443search2

Current timer documentation states that MockTimers is stable and documents mocked timer APIs, Date mocking, timer ticking, `setTime()`, and automatic cleanup through test-context mocking. citeturn444443search1

---

# 282. Performance Considerations

Test performance is:

```text
developer experience
+
CI cost
+
feedback latency.
```

Optimize:

```text
test discovery
parallelism
fixture startup
database reuse
artifact size
sharding
```

without sacrificing:

```text
isolation
fidelity
reliability.
```

---

# 283. Memory Considerations

Watch:

```text
parallel process count
worker count
database containers
large fixtures
buffers
captured logs
mock histories
coverage data
diagnostic artifacts.
```

A faster suite with:

```text
10× memory usage
```

may be:

```text
operationally worse.
```

---

# 284. Security Considerations

Protect:

```text
CI credentials
test certificates
fixture data
diagnostic artifacts
native binaries
mocked secrets
```

.

Never use:

```text
production secrets
```

for:

```text
routine test runs.
```

---

# 285. Reliability Considerations

The test platform must itself tolerate:

```text
test failure
timeout
crash
worker exit
child exit
resource exhaustion
network failure
artifact-storage failure.
```

A broken artifact collector should not:

```text
convert all test results into “unknown”.
```

---

# 286. Common Misconceptions

### Misconception 1

```text
“Tests are isolated because each test has its own callback.”
```

Reality:

```text
process, module, external-resource, and environment state can still be shared.
```

### Misconception 2

```text
“Fake timers simulate the whole event loop.”
```

Reality:

```text
they replace selected timing APIs; real I/O and runtime scheduling still differ.
```

### Misconception 3

```text
“Retries remove flaky tests.”
```

Reality:

```text
retries can hide flakiness.
```

### Misconception 4

```text
“100% coverage means strong tests.”
```

Reality:

```text
coverage measures execution, not correctness.
```

### Misconception 5

```text
“Module mocking is just function mocking at a larger scale.”
```

Reality:

```text
module evaluation, cache, import timing, and graph semantics make it more complex.
```

### Misconception 6

```text
“Force-exit fixes hanging tests.”
```

Reality:

```text
it can hide leaked resources.
```

Current Node CLI explicitly provides force-exit as an execution option, but resource ownership should still be fixed. citeturn444443search0

---

# 287. Common Mistakes

```text
[ ] real sleeps in unit tests
[ ] fixed ports
[ ] fixed temp filenames
[ ] shared database rows
[ ] global mocks
[ ] un-restored environment
[ ] worker leaks
[ ] child-process leaks
[ ] timer leaks
[ ] no test timeout
[ ] infinite retries
[ ] random failures without seed
[ ] no failure artifacts
[ ] sharding by file count only
[ ] excessive process isolation without measurement
[ ] no isolation when stateful tests exist
[ ] module mocks registered too late
[ ] mocking everything
[ ] testing nothing with real dependencies
[ ] force-exit used as cleanup
[ ] CI secrets printed
[ ] flaky tests ignored
```

---

# 288. Final Test System Mental Model

```text
TEST INTENT
    ↓
TEST BOUNDARY
    ↓
DEPENDENCY CONTROL
    ↓
DETERMINISTIC INPUT
    ↓
EXECUTION
    ↓
ASSERTIONS
    ↓
CLEANUP
    ↓
DIAGNOSTICS
    ↓
RESULT
    ↓
REPRODUCIBILITY
```

---

# 289. Isolation Mental Model

```text
TEST
 ↓
JS state
 ↓
module state
 ↓
process state
 ↓
filesystem
 ↓
ports
 ↓
database
 ↓
network
 ↓
external systems
```

A test is isolated only when:

```text
all relevant state
```

is either:

```text
scoped
reset
or independently owned.
```

---

# 290. Mocking Mental Model

```text
REAL DEPENDENCY
      ↓
Should it be real?
      ├─ yes → integration
      └─ no
          ↓
      deterministic seam
          ↓
      fake/stub/mock
          ↓
      test behavior
```

Mocking is:

```text
a design decision
```

not:

```text
a testing reflex.
```

---

# 291. Flake Diagnosis Mental Model

```text
FAILURE
 ↓
REPRODUCE?
 ├─ yes → deterministic bug
 └─ no
    ↓
ORDER?
TIME?
RANDOM?
RESOURCE?
ENVIRONMENT?
NETWORK?
CLEANUP?
 ↓
CLASSIFY
 ↓
ISOLATE
 ↓
REPLAY
 ↓
FIX
 ↓
REMOVE RETRY DEPENDENCY.
```

---

# 292. CI Mental Model

```text
COMMIT
 ↓
DISCOVER
 ↓
SHARD
 ↓
RUN IN PARALLEL
 ↓
COLLECT RESULTS
 ↓
RETRY DIAGNOSTICALLY
 ↓
PUBLISH ARTIFACTS
 ↓
AGGREGATE METRICS
 ↓
GATE RELEASE
```

---

# 293. Principal Testing Mental Model

```text
confidence
=
correctness
+
isolation
+
determinism
+
fidelity
+
coverage
+
observability
+
maintainability.
```

No single metric proves:

```text
“tests are good.”
```

---

# 294. Dependency Graph

```text
Chapter 31
Async
        ↓
Chapter 33
Event Loop
        ↓
Chapter 39
Streams
        ↓
Chapter 52
Workers
        ↓
Chapter 63
Diagnostics
        ↓
Chapter 70
Debugging
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
Chapter 88
Debugging Methodology
        ↓
Chapter 101
Production Scenarios
        ↓
Chapter 140
Node Networking
        ↓
Chapter 141
Node Diagnostics
        ↓
Chapter 142
Permission Model
        ↓
Chapter 143
Native Addons
        ↓
Chapter 144
Node.js Test Runner, Mocking & Test Isolation
```

Cross-cutting:

```text
modules
timers
workers
child processes
networking
filesystem
security
diagnostics
CI
performance
reliability
```

---

# 295. Concept Connections

## Depends On

```text
Node runtime
async
event loop
modules
workers
filesystem
networking
diagnostics
performance
security
reliability.
```

## Builds Toward

```text
property-based testing
fuzzing
test isolation engineering
CI platform engineering
flaky-test elimination
production verification
runtime-level test automation.
```

## Related Concepts

```text
mock
stub
spy
fake
fixture
isolation
randomization
seed
sharding
retry
coverage
reporter
test events
```

## Concepts Revisited

```text
Timers
Promises
AbortController
Workers
Child Processes
HTTP
TLS
DNS
Filesystem
Permission Model
Diagnostics
Native Addons
Observability
```

## Why This Chapter Matters

A test suite is not a pile of assertions.

It is:

```text
a concurrent runtime environment
```

with:

```text
state
resources
processes
timers
network
filesystem
mocks
external dependencies.
```

Therefore a mature testing strategy must engineer:

```text
isolation
determinism
cleanup
reproducibility
fidelity
observability
```

as carefully as:

```text
production code.
```

---

# 296. Retrieval Record

```md
# Chapter 144 — Revision / Retrieval Record

## Attempt
- Date:
- Duration:
- Status before:
- Status after:

## node:test
-

## Test Discovery
-

## Hooks
-

## Subtests
-

## TestContext
-

## Process Isolation
-

## None Isolation
-

## Concurrency
-

## Mock Functions
-

## Mock Properties
-

## Mock Timers
-

## Date Mocking
-

## Module Mocking
-

## Dependency Injection
-

## Fixtures
-

## Cleanup
-

## Randomization
-

## Random Seed
-

## Sharding
-

## Rerun Failures
-

## Coverage
-

## Reporters
-

## Test Events
-

## Flaky Tests
-

## CI
-

## Permission Model
-

## Native Addons
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

# 297. Spaced Retrieval Schedule

### Day 0

Study:

```text
node:test
hooks
context
isolation
mocks.
```

### Day 1

Explain:

```text
process isolation
vs
none
```

and:

```text
mock
vs
fake
vs
stub
vs
spy.
```

### Day 3

Build:

```text
temp-dir
env
HTTP
worker
```

fixtures.

### Day 7

Build:

```text
deterministic clock
retry tests
cancellation tests.
```

### Day 14

Run:

```text
randomized suite
```

and reproduce:

```text
order failure
```

with a fixed seed.

### Day 21

Build:

```text
CI sharding
+
failure artifact bundle
+
flake recorder.
```

### Day 30

Design:

```text
50,000-test Node platform
```

without notes.

---

# 298. Completion Criteria

Mark:

```text
[~] In Progress
```

when you can:

```text
write normal node:test tests.
```

Mark:

```text
[?] Needs Revision
```

when you repeatedly:

```text
leak mocks
depend on sleeps
share state
ignore cleanup
retry without diagnosis.
```

Mark:

```text
[+] Completed
```

when you can:

```text
build deterministic, isolated, concurrent test suites
with mocks, fake time, fixtures, and CI sharding.
```

Mark:

```text
[*] Mastered
```

only when you can:

```text
design and operate a test platform
that remains reliable under:
parallelism
large scale
Node upgrades
native addons
permissions
network failures
worker failures
CI resource pressure
and flaky behavior.
```

Reading alone does not mark mastery.

---

# 299. Final Principal Principle

> **A trustworthy test suite does not merely prove that code can pass. It proves that behavior remains correct under controlled boundaries, realistic dependencies, concurrency, failure, and repeatable execution.**

The production sequence is:

```text
DEFINE BEHAVIOR
→ CHOOSE TEST BOUNDARY
→ CONTROL NONDETERMINISM
→ ISOLATE STATE
→ OWN RESOURCES
→ EXECUTE
→ ASSERT
→ CLEAN UP
→ CAPTURE EVIDENCE
→ REPRODUCE FAILURES
→ REMOVE FLAKES
→ SCALE THROUGH SHARDING
→ VERIFY AGAINST REAL DEPENDENCIES
```

Remember:

```text
test passing ≠ test isolated

mocking ≠ correctness

coverage ≠ confidence

retry ≠ flake fix

sleep ≠ synchronization

force-exit ≠ cleanup

randomization ≠ chaos without reproduction

process isolation ≠ external-resource isolation

fake time ≠ real event loop

Node-API/native integration ≠ normal JS failure model

CI parallelism ≠ free performance

stable runner ≠ every new runner feature being stable.
```

The principal question is:

```text
“What exact behavior are we proving, what state and resources
could contaminate that proof, which dependencies should be real,
where is the deterministic seam, how will failures be reproduced,
and how will we know that a green build represents genuine system
confidence rather than accidental test isolation?”
```

That is Node.js test infrastructure engineering.