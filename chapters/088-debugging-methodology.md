# Chapter 88 — Debugging Methodology

> **Part XVI — Testing / Debugging**  
> **Status:** `[ ] Not Started`  
> **Depth Target:** Principal / Staff-level reasoning  
> **Primary Focus:** A disciplined method for finding, explaining, reproducing, fixing, and preventing defects in production JavaScript systems.

---

# 1. Learning Objectives

By the end of this chapter, you should be able to:

1. Define debugging as hypothesis-driven investigation.
2. Distinguish debugging from testing, monitoring, observability, and profiling.
3. Build a reproducible problem statement.
4. Separate symptoms, causes, contributing factors, and root causes.
5. Create a timeline from logs, traces, metrics, and runtime evidence.
6. Narrow a large failure space systematically.
7. Form and rank hypotheses.
8. Design experiments that distinguish competing hypotheses.
9. Reproduce synchronous and asynchronous failures.
10. Debug Promise and event-loop issues.
11. Debug race conditions and nondeterministic bugs.
12. Debug memory leaks and GC-related degradation.
13. Debug CPU bottlenecks and event-loop blocking.
14. Debug database access and transaction failures.
15. Debug API/network failures.
16. Debug streams and backpressure issues.
17. Debug workers, child processes, and lifecycle problems.
18. Debug module and dependency problems.
19. Debug production-only failures.
20. Debug build/source-map mismatches.
21. Use breakpoints, stack traces, logging, traces, profiles, heap snapshots, and diagnostic reports appropriately.
22. Understand the trade-offs of attaching debuggers to live processes.
23. Use Node.js inspector tooling safely.
24. Use diagnostic reports to preserve runtime evidence.
25. Distinguish code bugs from environment bugs, data bugs, dependency bugs, and operational bugs.
26. Avoid cargo-cult debugging.
27. Debug without introducing new production failures.
28. Use minimization and delta debugging.
29. Perform binary-search debugging across code/config/deployments.
30. Build a regression test from a discovered bug.
31. Write useful incident timelines and root-cause analyses.
32. Debug distributed systems using correlation and causality.
33. Build a repeatable debugging playbook.
34. Review debugging practices for signal quality and safety.
35. Defend a debugging methodology at principal-engineer level.

---

# 2. Prerequisites

You should understand:

- JavaScript language semantics.
- Scope, closures, objects, modules.
- Promises and async/await.
- ECMAScript jobs and event loops.
- Node.js architecture and lifecycle.
- Errors and cleanup.
- Memory and GC.
- Streams and backpressure.
- Workers and processes.
- HTTP/API architecture.
- Database integration.
- Observability.
- Reliability.
- Performance.
- Testing and deterministic async testing.

Recommended prior chapters:

- **10–14** — Scope / Execution / Closures / `this`
- **29–30** — Errors / Resource Management
- **31–40** — Async / Promises / Event Loop / Cancellation / Streaming / Concurrency
- **45–48** — Memory / GC / Engine / V8
- **58–63** — Node.js / Streams / Workers / Lifecycle / Async Context
- **70** — Source Maps / Production Debugging
- **78–85** — Production Architecture / API / Database / Observability / Reliability / Performance
- **86** — Testing
- **87** — Deterministic Async Testing

---

# 3. What Is It?

Debugging is the process of determining why observed software behavior differs from intended behavior.

A disciplined debugging process is:

```text
observe
→ define
→ reproduce
→ hypothesize
→ experiment
→ localize
→ explain
→ fix
→ verify
→ prevent recurrence
```

Debugging is not:

```text
change code until tests pass
```

It is not:

```text
add logs everywhere
```

It is not:

```text
restart production
```

It is structured investigation.

---

## 3.1 Bug

A bug is a discrepancy between:

```text
intended behavior
```

and:

```text
actual behavior
```

The discrepancy may be:

```text
incorrect result
unexpected latency
memory growth
crash
data loss
duplicate side effect
security violation
resource leak
```

---

# 4. Why Does It Exist?

Production systems are too complex for intuition alone.

A failure can arise from:

```text
code
configuration
data
timing
concurrency
dependency
network
database
runtime
deployment
infrastructure
human operation
```

Two identical code paths can behave differently because:

```text
environment changed
input changed
timing changed
dependency changed
state changed
```

Debugging methodology reduces this uncertainty.

---

# 5. Mental Model

Treat debugging as a search problem.

```text
                         Failure
                            │
              ┌─────────────┼─────────────┐
              │             │             │
             Code        Environment     Data
              │             │             │
          ┌───┼───┐     ┌───┼───┐      ┌──┼──┐
          │   │   │     │   │   │      │  │  │
        Logic Race Memory Dep Network DB Input State
```

You do not inspect everything equally.

You eliminate possibilities.

A strong debugging investigation has:

```text
question
hypothesis
evidence
experiment
result
next hypothesis
```

---

# 6. Core Rules

## Rule 1 — Reproduce before rewriting

A reproducible failure is easier to reason about.

---

## Rule 2 — Preserve evidence

Do not immediately:

```text
restart everything
clear logs
change five variables
```

without capturing useful evidence.

---

## Rule 3 — Change one important variable at a time

Otherwise you cannot tell which change mattered.

---

## Rule 4 — Prefer discriminating experiments

A good experiment separates hypotheses.

Example:

```text
Hypothesis A: database slow
Hypothesis B: JavaScript CPU-bound
```

Measure:

```text
DB span
event-loop delay
CPU profile
```

---

## Rule 5 — Find the smallest failing case

Reduce:

```text
10,000 inputs
→ 1,000
→ 100
→ 10
→ 1
```

This is delta debugging.

---

## Rule 6 — Localize before optimizing

Find:

```text
where
```

before asking:

```text
how to improve
```

---

## Rule 7 — Separate trigger from root cause

Example:

```text
trigger: traffic spike
contributor: slow database
root cause: missing index
```

The traffic spike was not necessarily the root cause.

---

## Rule 8 — Test the fix

A manual observation is not enough.

Create:

```text
regression test
```

when practical.

---

## Rule 9 — Preserve production safety

A debugging action that causes an outage is not a good debugging action.

---

## Rule 10 — Explain the mechanism

A good diagnosis should answer:

```text
what happened
why it happened
why it happened now
why the observed symptom followed
why the fix works
```

---

# 7. Syntax

Debugging uses ordinary JavaScript plus runtime tooling.

## Stack trace

```js
function first() {
  second();
}

function second() {
  throw new Error("boom");
}

first();
```

Node prints a stack trace showing the call chain.

---

## Explicit debugger

```js
function calculate(value) {
  debugger;

  return value * 2;
}
```

When an inspector is attached and debugging is active, execution can pause at the statement.

---

## Node inspector

Node's current debugger documentation states that `node inspect` provides command-line debugging and that V8 Inspector integration allows Chrome DevTools and other compatible tooling to attach through the Chrome DevTools Protocol. citeturn195851search3

Example:

```bash
node --inspect app.js
```

Node also documents `--inspect-wait` and `--inspect-brk` for waiting for a debugger before execution or breaking immediately when attached. citeturn195851search3

---

# 8. Basic Examples

## Example 1 — Read the stack

```js
function a() {
  b();
}

function b() {
  c();
}

function c() {
  throw new Error("failure");
}

a();
```

The stack gives:

```text
c
b
a
```

The key question is not:

> “Where did it crash?”

It is:

> “What state reached the failing operation through this call chain?”

---

## Example 2 — Capture state

```js
function calculate(order) {
  console.log({
    orderId: order.id,
    status: order.status,
  });

  return order.total * 2;
}
```

Better production debugging would use structured telemetry and avoid logging sensitive fields.

---

## Example 3 — Conditional breakpoint

Conceptually:

```text
pause only when:
order.id === target
```

This reduces inspection noise in high-frequency code.

---

# 9. Execution Walkthrough

Suppose:

```text
POST /orders
returns 500
```

Do not immediately open the controller.

Start with:

```text
When?
Who?
What request?
What version?
What tenant?
What dependency?
What error?
What changed?
```

Then create the timeline:

```text
12:03:01 request started
12:03:01 auth passed
12:03:01 order loaded
12:03:01 payment request started
12:03:04 payment timeout
12:03:04 retry
12:03:07 HTTP 500
```

Now hypotheses:

```text
A — payment provider slow
B — timeout too short
C — retry consumed deadline
D — request context lost
```

Each can be tested.

---

# 10. Internal Mechanics

## 10.1 Stack trace

A stack trace represents a chain of execution contexts/calls observed at an error or breakpoint.

Use it to answer:

```text
how did execution reach here?
```

Do not assume it tells you:

```text
why the state is wrong
```

---

## 10.2 Error cause chains

Modern JavaScript supports error causes:

```js
throw new Error(
  "Failed to load order",
  {
    cause: originalError,
  },
);
```

Preserve causal information across boundaries where useful.

---

## 10.3 Source maps

Production JavaScript may execute:

```text
bundled/transpiled code
```

while developers reason about:

```text
source code
```

Source maps help map runtime locations back to source.

A mismatch can produce misleading stack locations.

---

## 10.4 Inspector

Node's inspector can expose:

```text
breakpoints
call stack
variables
heap
CPU profiling
```

through compatible DevTools.

The current Node documentation notes that `--inspect` allows the program to begin execution before a debugger connects, while `--inspect-wait` waits for attachment and `--inspect-brk` breaks at the first line after attachment. citeturn195851search3

---

# 11. ECMAScript / Specification Semantics

Debugging must distinguish:

```text
language semantics
runtime behavior
environment behavior
tool behavior
```

For example:

```js
Promise.resolve().then(...)
```

has ECMAScript scheduling semantics.

But:

```text
Node inspector
heap snapshot
process report
event-loop metrics
```

are Node/runtime capabilities.

A debugger does not change JavaScript language semantics, but attaching a debugger can change timing enough to expose or hide races.

---

# 12. Advanced Behavior

## 12.1 Hypothesis-driven debugging

Suppose latency increased.

Hypotheses:

```text
H1 database
H2 network
H3 CPU
H4 GC
H5 lock contention
```

Evidence:

```text
DB spans normal
network normal
CPU high
GC normal
event-loop delay high
```

Now:

```text
H3 ↑
H1 ↓
H2 ↓
H4 ↓
H5 ?
```

Continue.

Do not stop at:

> “CPU is high.”

Ask:

```text
Which function?
Which requests?
Which inputs?
Which release?
Why now?
```

---

## 12.2 Binary-search debugging

Suppose the codebase has:

```text
commit 100
→ 200
```

and bug exists at 200 but not 100.

Binary search:

```text
150
→ 125
→ 137
→ ...
```

Find the introducing change efficiently.

The same strategy can work across:

```text
configuration
feature flags
dependencies
deployments
datasets
```

---

## 12.3 Differential debugging

Compare:

```text
working system
vs
failing system
```

Differences may include:

```text
code version
config
data
runtime version
traffic
dependency
```

Control variables.

---

## 12.4 Fault localization

A useful progression:

```text
system
→ process
→ request
→ operation
→ module
→ function
→ branch
→ state
→ input
```

Do not inspect function internals before establishing that the function is involved.

---

## 12.5 Minimal reproduction

Example:

Production:

```text
API
→ authentication
→ database
→ queue
→ payment
→ notification
```

Bug:

```text
duplicate payment
```

Minimal reproduction may become:

```text
payment client
+ retry behavior
+ same idempotency key
```

This reduces noise.

---

# 13. Edge Cases

## 13.1 Heisenbug

A bug changes behavior when observed.

Debugger pauses can alter:

```text
timing
race ordering
timeouts
resource scheduling
```

Use low-intrusion evidence:

```text
metrics
traces
structured logs
sampling
```

where appropriate.

---

## 13.2 Production debugger timing

Attaching a debugger can change the system.

Do not use an interactive debugger as the default mechanism for diagnosing latency-sensitive production incidents.

Prefer:

```text
profiles
traces
diagnostic reports
controlled reproduction
```

---

## 13.3 Async stack complexity

An error may cross:

```text
Promise
timer
network
worker
queue
```

The visible stack may not contain the entire distributed causal history.

Use:

```text
trace context
request ID
job ID
correlation
```

---

## 13.4 Optimized code

Optimized JavaScript engines can transform code.

Debugging views can therefore sometimes be less intuitive than source code suggests.

Do not infer engine behavior solely from a debugger's variable display.

---

## 13.5 Minified production artifacts

A stack like:

```text
main.a8df.js:1:93822
```

is difficult to reason about without source maps and versioned artifacts.

---

# 14. Common Misconceptions

### “The stack trace is the root cause.”

It shows execution context, not necessarily causality.

### “The first error is the root cause.”

The first visible error may be a consequence of earlier failure.

### “Restart fixed it, so memory was the issue.”

Restart may only remove accumulated state.

### “CPU is high, so CPU is the root cause.”

CPU may be high because a dependency is causing retry or serialization work.

### “Adding logs always helps.”

Logs can add noise, cost, and new timing effects.

### “Debugger proves variable values.”

Optimized runtime/tooling views can differ from source-level intuition.

### “More metrics mean more observability.”

Not necessarily.

### “A test failure identifies the bug.”

A failing test identifies an observed mismatch, not necessarily the cause.

### “Retrying the failing test proves the fix.”

It may only prove nondeterminism.

---

# 15. Common Mistakes

## Mistake 1 — Changing code before reproducing

---

## Mistake 2 — Fixing the symptom

Example:

```text
increase timeout
```

without finding why the operation became slow.

---

## Mistake 3 — Changing multiple variables

```text
upgrade dependency
change timeout
change query
change config
```

No causal conclusion is possible.

---

## Mistake 4 — Ignoring data

Many bugs are:

```text
unexpected state
```

not:

```text
bad code
```

---

## Mistake 5 — Ignoring deployment changes

A bug beginning after:

```text
deployment 471
```

should trigger deployment comparison.

---

## Mistake 6 — No regression test

Bug returns months later.

---

## Mistake 7 — Debug logging secrets

Never trade diagnostic convenience for data leakage.

---

# 16. Comparison With Related Concepts

| Technique | Best For | Limitation |
|---|---|---|
| Stack trace | Locate failure path | Limited causality |
| Breakpoint | Inspect local state | Intrusive/timing change |
| Structured logs | Historical events | Volume/noise |
| Metrics | Trends/saturation | Low detail |
| Traces | Causality/latency | Sampling/storage |
| CPU profile | Hot code | Not state history |
| Heap snapshot | Memory retention | Point-in-time/overhead |
| Diagnostic report | Runtime snapshot | Snapshot-based |
| Reproduction | Controlled behavior | May differ from production |
| Binary search | Regression localization | Requires ordered change history |
| Differential debugging | Compare states | Needs comparable environments |
| Fault injection | Failure path | Infrastructure complexity |
| Core dump/native tooling | Native/runtime failures | Higher complexity |

---


---

# 16A. Evidence Quality

Not all evidence has equal diagnostic value.

A useful hierarchy is:

```text
direct observation
→ correlated evidence
→ strong inference
→ weak inference
→ assumption
```

Example:

```text
DB span = 2.8s
```

is direct evidence that the database operation consumed that observed time.

This:

```text
CPU is high
therefore DB is slow
```

is an unsupported inference unless additional evidence connects the two.

---

## Evidence matrix

| Observation | What It Supports | What It Does Not Prove |
|---|---|---|
| p99 latency increased | latency regression exists | exact cause |
| DB span increased | database work became slower | why DB became slower |
| CPU increased | more CPU consumption | which function |
| heap increased | memory retention/allocation changed | leak |
| restart fixes issue | accumulated state was removed | memory leak specifically |
| error begins after deploy | temporal association | deploy is definitely root cause |
| one tenant affected | scope is tenant-specific | tenant data is corrupt |

The discipline is:

> Never claim more causality than the evidence supports.

---

# 16B. Hypothesis Ranking

Do not generate hypotheses randomly.

Rank them by:

```text
plausibility
impact
evidence
testability
cost of investigation
```

Example:

```text
H1 — database regression
H2 — deployment regression
H3 — dependency timeout
H4 — rare input path
```

A useful table:

| Hypothesis | Evidence For | Evidence Against | Next Test |
|---|---|---|---|
| DB regression | DB p99 ↑ | only one endpoint | query plan |
| CPU regression | event loop ↑ | CPU normal | profile |
| dependency | remote span ↑ | provider healthy | compare release |
| data-specific | one tenant | no similar cases | reproduce dataset |

This prevents “loudest hypothesis wins” debugging.

---

# 16C. Experiment Design

A strong debugging experiment changes one relevant condition and predicts an outcome.

Structure:

```text
Hypothesis:
If true:
Experiment:
Expected:
Observed:
Conclusion:
```

Example:

```text
Hypothesis:
A slow JSON transformation blocks the event loop.

If true:
event-loop delay should correlate with high-cost payloads.

Experiment:
replay small and large payloads while measuring delay.

Expected:
large payload produces higher event-loop delay.

Observed:
delay increases only with large payload.

Conclusion:
large-payload CPU work is a strong candidate.
```

A good experiment should have a pre-declared expected result.

Otherwise it is easy to reinterpret any output as confirmation.

---

# 16D. Five Whys Without Losing Technical Precision

“Why?” analysis can uncover deeper causes.

Example:

```text
Why did orders fail?
→ database connections exhausted.

Why were connections exhausted?
→ requests held connections too long.

Why were connections held too long?
→ external API calls happened inside transactions.

Why did that happen?
→ transaction boundary included remote side effects.

Why was that allowed?
→ architecture lacked a rule separating durable DB state from external calls.
```

The useful outcome is not a human blame chain.

It is:

```text
technical mechanism
→ architectural weakness
→ prevention
```

Avoid stopping at:

```text
developer made a mistake
```

That is rarely a useful systemic explanation.

---

# 16E. Counterfactual Reasoning

Ask:

> If this suspected cause were absent, would the failure still happen?

Example:

```text
Suspected cause:
new dependency version.
```

Counterfactual experiment:

```text
same input
same config
previous dependency version
```

If failure disappears:

```text
dependency version becomes stronger evidence.
```

Counterfactual reasoning is especially useful for:

```text
deployments
dependencies
feature flags
configuration
data migrations
runtime versions
```

---

# 16F. Causal Graph Thinking

Represent a failure as:

```text
traffic spike
     ↓
queue growth
     ↓
request latency
     ↓
client timeout
     ↓
client retry
     ↓
more traffic
     ↓
greater queue growth
```

The “root cause” may be:

```text
capacity limit + retry policy
```

rather than one isolated line of code.

This is why debugging production systems often requires a graph, not just a stack trace.

---

# 16G. State Reconstruction

Many bugs are state bugs.

Reconstruct:

```text
initial state
→ operation
→ mutation
→ external event
→ second mutation
→ failure
```

Example:

```text
order.status = pending
payment = authorized
worker receives message
request times out
client retries
worker receives duplicate
order.status = confirmed
payment charged twice
```

The important evidence is not merely:

```text
payment function failed
```

but:

```text
state transitions
```

Use event histories, database records, audit entries, and trace context where available.

---

# 16H. Distributed Causality

A distributed workflow may look like:

```text
Trace A
  service A
    ↓
  queue message
    ↓
Trace B
  worker B
    ↓
  service C
```

Different traces may be legitimate when asynchronous boundaries start new trace segments according to the chosen instrumentation model, but correlation must preserve causal linkage.

Useful identifiers:

```text
trace_id
span_id
parent
message_id
job_id
order_id
request_id
```

Do not use business identifiers as substitutes for trace context.

A useful rule:

```text
trace context = execution causality
business ID   = domain identity
request ID    = request correlation
job ID        = asynchronous work identity
```

---

# 16I. Production Debugging Under Uncertainty

Sometimes you cannot safely reproduce a bug.

Then use:

```text
scope comparison
time correlation
version comparison
tenant comparison
dependency comparison
traffic comparison
state comparison
```

Example:

```text
affected:
production only

unaffected:
staging
```

Compare:

```text
Node version
database size
traffic
feature flags
dependency versions
environment variables
data shape
network topology
```

This is differential debugging.

---

# 16J. Production Debugging Safety Matrix

| Action | Information | Risk |
|---|---|---|
| Query metrics | High | Low |
| Inspect traces | High | Low |
| Search logs | High | Low–medium |
| Enable debug logs | High | Medium |
| CPU profile | High | Medium |
| Heap snapshot | High | High |
| Diagnostic report | High | Medium/high data sensitivity |
| Attach debugger | Very high | High timing/operational risk |
| Restart process | Low–medium | High evidence loss / behavior change |
| Change configuration | Variable | High |
| Patch live code | Variable | Very high |

Use the least disruptive action that answers the current question.

---

# 16K. Restart as a Debugging Action

A restart can:

```text
clear memory
reset connections
remove leaked listeners
reset process state
```

Therefore it is sometimes a mitigation.

But:

```text
restart fixed it
```

does not identify the cause.

Capture evidence before restart when the operational situation allows it.

Then ask:

```text
what state did restart destroy?
what behavior disappeared?
what should be measured next?
```

---

# 16L. Configuration Debugging

Configuration bugs are often invisible because code inspection looks correct.

Compare:

```text
expected config
actual config
source
precedence
normalization
runtime environment
```

Typical layers:

```text
defaults
→ config file
→ environment
→ secrets
→ deployment flags
→ runtime overrides
```

A debugging report should identify the effective configuration without exposing secret values.

---

# 16M. Data Debugging

Data bugs can be:

```text
invalid historical record
unexpected null
duplicate row
wrong tenant
schema drift
migration defect
timezone mismatch
encoding issue
```

Use:

```text
reproduction record
minimal dataset
schema constraints
audit history
migration version
```

Do not “fix” production data before understanding why it became invalid.

---

# 16N. Dependency Debugging

For an external dependency inspect:

```text
request count
success rate
latency
status distribution
timeout rate
retry count
payload size
connection behavior
version
regional distribution
```

Compare:

```text
healthy period
vs
failure period
```

This distinguishes:

```text
our request changed
```

from:

```text
dependency changed
```

---

# 16O. Dependency Contract Diagnosis

Suppose a provider returns:

```json
{
  "status": "SUCCESS"
}
```

while the client expects:

```text
success
```

Possible causes:

```text
provider behavior changed
documentation mismatch
client normalization missing
proxy transformed payload
API version mismatch
```

The debugging task is to identify the first boundary where actual behavior diverged from expected contract.

---

# 16P. Database Debugging Workflow

For database-related problems:

```text
request trace
→ repository call
→ query
→ duration
→ lock/wait
→ query plan
→ pool state
→ database resource state
```

Potential causes:

```text
bad query plan
missing index
lock contention
pool exhaustion
connection failure
replication lag
database CPU
storage latency
network latency
```

Do not solve all database incidents by increasing:

```text
pool
timeout
database size
```

without evidence.

---

# 16Q. Memory Debugging Workflow

Use:

```text
symptom
→ RSS/heap comparison
→ allocation rate
→ retained objects
→ GC behavior
→ active handles
→ external memory
```

Distinguish:

```text
heap leak
native/external memory
intentional cache
temporary high watermark
fragmentation/allocator behavior
unbounded queue
```

A rising RSS does not automatically prove a JavaScript heap leak.

Node diagnostic reports include V8 heap information and resource/OS data that can help distinguish classes of runtime state. citeturn195851search0

---

# 16R. CPU Debugging Workflow

Start:

```text
CPU symptom
→ affected requests/jobs
→ event-loop delay
→ CPU profile
→ hot function
→ input correlation
```

Then determine:

```text
algorithmic work
serialization
compression
crypto
parsing
regex
GC
retry amplification
logging
```

Do not assume the hottest function is the root cause; it may be downstream work triggered by another failure.

---

# 16S. Async Debugging Workflow

Use:

```text
request ID
trace ID
job ID
timeline
state transitions
await boundaries
cancellation
timeouts
retries
```

Ask:

```text
what started?
what completed?
what was waiting?
what was cancelled?
what was retried?
what ran concurrently?
what resource was held?
```

This is more useful than reading async code top-to-bottom and guessing.

---

# 16T. Race Debugging Workflow

1. Identify shared state.
2. Identify concurrent operations.
3. Identify critical interleaving.
4. Create a deterministic barrier.
5. Reproduce.
6. Inspect state transitions.
7. Fix ownership/atomicity.
8. Add regression test.
9. Stress test under realistic concurrency.

A reproducible race is dramatically easier to reason about.

---

# 16U. Build/Dependency Debugging

When a production artifact behaves unexpectedly:

```text
source
→ build config
→ compiler/transpiler
→ bundler
→ source map
→ package lock
→ artifact
→ runtime
```

Compare:

```text
source commit
artifact checksum/version
dependency lock
runtime version
```

A source-level fix that was never included in the deployed artifact is not a production fix.

---

# 16V. Module Debugging

Possible failures:

```text
wrong export
wrong package condition
ESM/CommonJS mismatch
duplicate dependency
deep import
circular dependency
case-sensitive path
different Node version
```

Debug:

```text
actual resolved module
package.json
exports
runtime mode
dependency tree
```

---

# 16W. Production-Only Debugging Checklist

```text
Runtime version
OS/architecture
Environment variables
Secrets/configuration
Feature flags
Data volume
Data shape
Traffic shape
Dependency versions
Database size
Network topology
Concurrency
Resource limits
Time zone
Locale
Filesystem
Build artifact
Source maps
```

A production-only failure is often caused by a difference between environments rather than an impossible code path.

---

# 16X. Debugging a Security Incident

Security debugging changes the priority.

First:

```text
contain
preserve evidence
limit exposure
rotate credentials if required
establish scope
```

Then:

```text
identify entry point
identify affected resources
identify attacker-controlled input
identify privilege boundary
identify persistence/impact
```

Do not immediately delete evidence.

Coordinate with the organization's incident-response and legal/security procedures.

---

# 16Y. Debugging With Feature Flags

Feature flags create a natural experiment:

```text
flag ON
vs
flag OFF
```

Use them carefully.

If turning a flag off mitigates an incident:

```text
flag is associated with failure
```

but that does not prove exact root cause.

Capture:

```text
traffic
time
version
flag state
affected population
```

before changing interpretation.

---

# 16Z. Root-Cause Confidence

A diagnosis should carry confidence:

```text
high
medium
low
```

Example:

```text
High:
reproduced locally and production evidence matches.

Medium:
production evidence strongly correlates but reproduction unavailable.

Low:
timing correlation only.
```

Do not present low-confidence inference as established fact.

---

# 17AA. Fix Validation

A fix should be evaluated at multiple levels:

```text
unit behavior
integration behavior
regression case
production metrics
error rate
latency
memory
security
```

A fix can remove one error but create:

```text
higher latency
memory retention
retry amplification
security exposure
```

Verification must cover the relevant dimensions.

---

# 17AB. Canary Verification

When possible:

```text
small traffic
→ observe
→ compare
→ expand
```

Compare:

```text
error rate
latency
resource usage
business outcomes
```

A canary is a debugging/validation experiment at production scale.

---

# 17AC. Rollback Reasoning

Rollback can restore a prior known-good behavior.

But ask:

```text
Was data changed?
Was schema migrated?
Were messages emitted?
Did external side effects occur?
```

Code rollback does not automatically roll back:

```text
database writes
emails
payments
external API calls
events
```

Therefore rollback plans must include state evolution.

---

# 17AD. Post-Incident Learning

A useful post-incident review records:

```text
what happened
why it happened
how it was detected
what delayed diagnosis
what mitigated it
what fixed it
what evidence was missing
what controls should change
```

Avoid:

```text
who to blame
```

as the primary output.

The engineering objective is reduced recurrence probability and reduced time-to-diagnosis.

---

# 17AE. Debugging Maturity

### Level 1

```text
restart
add console.log
```

### Level 2

```text
stack traces
structured logs
tests
```

### Level 3

```text
metrics
traces
profiles
diagnostic reports
```

### Level 4

```text
hypothesis-driven investigations
reproducible failure labs
fault injection
automated regression
```

### Level 5

```text
systemic prevention
diagnostic automation
architecture-aware incident response
```

The goal is not maximal tooling.

It is faster and safer causal understanding.


# 17. Performance Considerations

Debugging tools have performance cost.

## Logging

Costs:

```text
CPU
allocation
serialization
I/O
storage
```

---

## Tracing

Costs:

```text
span creation
attribute handling
export
network
storage
```

---

## Debugger

Can significantly alter execution timing.

---

## Profiling

Profiles add overhead depending on mechanism, duration, frequency, and sampling mode.

Use the least intrusive mechanism that gives enough evidence.

---

# 18. Memory Considerations

Debugging itself can retain data.

Examples:

```text
large log buffers
trace batches
heap snapshots
debugger references
captured closures
request dumps
```

A debugging tool can create or amplify a memory problem.

---

## Heap snapshots

Node's runtime provides V8 heap diagnostics, and the inspector ecosystem can capture heap information. Use snapshots carefully in memory-constrained processes because capturing/retaining a large heap snapshot can itself be expensive.

---

# 19. Security Considerations

Debugging tools expose sensitive internal state.

## Inspector exposure

Do not expose a debugger endpoint publicly.

The current Node inspector documentation describes a local inspector endpoint and warns about the debugging interface; production exposure must be tightly controlled. citeturn195851search3

---

## Diagnostic reports

Node's current diagnostic report documentation states that reports can include JavaScript/native stacks, V8 heap information, resource usage, platform data, environment variables unless excluded, and network-related information unless excluded. citeturn195851search0

Therefore:

```text
report ≠ harmless debug file
```

Protect reports as sensitive operational artifacts.

The current Node documentation provides options including `--report-exclude-env` and `--report-exclude-network`. citeturn195851search0

---

## Production logs

Avoid:

```text
password
token
cookie
private key
payment credential
customer secrets
```

---

# 20. Production Usage

## 20.1 Debugging workflow

Use:

```text
1. Detect
2. Define
3. Preserve evidence
4. Scope
5. Reproduce
6. Hypothesize
7. Experiment
8. Localize
9. Explain
10. Fix
11. Verify
12. Prevent
```

---

## 20.2 Incident triage

First establish:

```text
What is broken?
How many users?
When did it start?
What changed?
Is it getting worse?
Is there data corruption?
Is there security impact?
Can we mitigate safely?
```

---

## 20.3 Severity-first debugging

Prioritize:

```text
data loss
security exposure
financial duplication
service-wide outage
large blast radius
```

before:

```text
minor cosmetic issue
```

---

## 20.4 Evidence ladder

Start with low-intrusion evidence:

```text
metrics
→ traces
→ structured logs
→ profiles
→ snapshots/reports
→ targeted debugger
```

This is a heuristic, not a rigid ordering.

---

## 20.5 Node diagnostic reports

Node's current documentation describes diagnostic reports as stable and intended for development, testing, and production problem determination. They can include JS/native stack traces, heap statistics, resource usage, platform data, and libuv handle information, and can be triggered by uncaught exceptions, fatal errors, signals, or programmatically. citeturn195851search0

Example:

```js
process.report.writeReport();
```

or:

```bash
node --report-on-fatalerror app.js
```

Use according to your incident policy and data-handling requirements.

---

## 20.6 Debugging live Node processes

Current Node documentation supports attaching the inspector to running processes and provides:

```text
node inspect
--inspect
--inspect-wait
--inspect-brk
```

for different debugging lifecycles. citeturn195851search3

Use live debugging sparingly.

---

## 20.7 Async context

When investigating distributed requests, use:

```text
requestId
traceId
spanId
tenantId
jobId
```

appropriately.

Node's stable `AsyncLocalStorage` is designed to associate state across asynchronous operations. citeturn195851search2

---

# 21. Implementation From Scratch

Build a debugging laboratory.

## Stage 1 — Guided

Create a failure:

```js
function divide(total, count) {
  return total / count;
}
```

Input:

```text
count = 0
```

Observe:

```text
Infinity
```

Determine whether this is:

```text
language-defined result
or
application bug
```

This teaches that unexpected behavior is not automatically a runtime failure.

---

## Stage 2 — Error diagnosis

Create:

```text
API
→ service
→ repository
```

Inject:

```text
repository failure
```

Build:

```text
stack trace
structured log
request ID
error code
```

---

## Stage 3 — Async bug

Create:

```text
missing await
```

Observe:

```text
test passes
failure appears later
```

Use Chapter 87 techniques to reproduce and diagnose it.

---

## Stage 4 — Race condition

Create:

```text
two concurrent inventory reservations
```

Use:

```text
barrier
timeline recorder
```

to produce the race deterministically.

---

## Stage 5 — Performance bug

Create:

```text
CPU-heavy loop
```

Measure:

```text
event-loop delay
CPU profile
request latency
```

Node's `perf_hooks` provides event-loop-delay monitoring and performance measurement APIs suitable for this style of investigation. citeturn439246search2

---

## Stage 6 — Memory bug

Create:

```js
const cache = new Map();

setInterval(() => {
  cache.set(
    crypto.randomUUID(),
    Buffer.alloc(1024),
  );
}, 10);
```

Investigate:

```text
heap growth
retained objects
allocation rate
GC
```

Then remove the leak and confirm with evidence.

---

## Stage 7 — Production-style diagnostic report

Enable diagnostic reporting in a test process.

Trigger:

```js
process.report.writeReport();
```

Inspect:

```text
header
resource usage
heap
handles
stacks
environment data
```

Node's report documentation describes the report as a JSON-formatted diagnostic summary and lists its runtime/resource sections. citeturn195851search0

---

## Stage 8 — Regression prevention

For every discovered bug:

```text
reproduce
→ capture minimal case
→ write regression test
→ fix
→ rerun
→ add observability if appropriate
```

---

# 22. Debugging Exercises

## Exercise 1 — Wrong result

```js
function calculatePrice(price, tax) {
  return price + tax;
}
```

Requirement:

```text
tax is a percentage
```

Find:

```text
input contract
bug
fix
regression test
```

---

## Exercise 2 — Production-only failure

Works locally.

Fails only in production.

Investigate:

```text
Node version
environment variables
database data
traffic
dependency version
timezone
network
resource limits
```

---

## Exercise 3 — Slow request

```text
p50 = 50ms
p99 = 4s
```

Determine:

```text
dependency tail
event-loop delay
lock contention
GC
```

---

## Exercise 4 — Duplicate job

```text
job executes twice
```

Investigate:

```text
queue delivery
ack timing
worker crash
idempotency
retry
```

---

## Exercise 5 — Memory leak

RSS increases continuously.

Heap increases slowly.

Investigate:

```text
native buffers
connections
handles
heap
external memory
```

---

## Exercise 6 — Crash

Node process exits.

Determine whether cause is:

```text
uncaught exception
unhandled rejection
fatal runtime error
OOM
signal
process.exit
container termination
```

Use logs and diagnostic reports where available.

---

## Exercise 7 — Source-map mismatch

Production stack:

```text
bundle.js:1:93822
```

Source:

```text
order-service.js:87
```

Find artifact/version/source-map alignment.

---

## Exercise 8 — Async context

A log inside a timer shows the wrong:

```text
requestId
```

Investigate context propagation.

---

## Exercise 9 — Database timeout

Trace:

```text
DB span = 4s
```

Investigate:

```text
query plan
lock
pool
database CPU
network
```

Do not immediately increase timeout.

---

## Exercise 10 — Heisenbug

Race disappears with debugger attached.

Create a non-invasive investigation plan.

---

# 23. Code Review Exercise

Review:

```js
app.post("/orders", async (req, res) => {
  console.log(req);

  const order = await createOrder(req.body);

  try {
    await chargePayment(order);
  } catch (error) {
    console.error(error);
  }

  res.json(order);
});
```

Find at least 25 debugging and production-engineering problems.

Consider:

```text
raw request logging
secret leakage
PII
missing request context
error swallowing
incorrect success response
payment state ambiguity
no idempotency
no timeout
no cancellation
no structured logs
no trace
no metrics
no error classification
no correlation ID
no authorization
no validation
no transaction semantics
no recovery
no regression test evidence
```

---

# 24. Interview Questions

## Fundamental

1. What is debugging?
2. Debugging versus testing?
3. Symptom versus root cause?
4. What is reproduction?
5. What is a hypothesis?
6. What makes an experiment useful?
7. What is a stack trace?
8. What is a minimal reproduction?
9. Why preserve evidence?
10. What is regression testing?

## Intermediate

11. How do you debug an intermittent bug?
12. How do you debug an async failure?
13. How do you debug a memory leak?
14. How do you debug high CPU?
15. How do you debug high p99?
16. How do you debug a database timeout?
17. How do you debug queue duplicates?
18. What is binary-search debugging?
19. What is differential debugging?
20. How do source maps affect debugging?

## Advanced

21. How do you debug production without changing behavior too much?
22. How do you diagnose a race condition?
23. How do you debug distributed workflows?
24. How do you distinguish code from configuration?
25. How do you distinguish network from database failure?
26. How do you use diagnostic reports?
27. When should you attach a debugger to production?
28. How do you debug GC pressure?
29. How do you preserve causal context?
30. How do you turn an incident into regression tests?

## Principal

31. How do you create a debugging methodology for an organization?
32. What evidence should every production service emit?
33. How do you prevent debugging from creating new outages?
34. When is a restart acceptable versus evidence-destroying?
35. How do you debug unknown unknowns?
36. How do you debug an issue that cannot be reproduced?
37. How do you choose between logs, traces, profiles, snapshots, and debugger?
38. How do you quantify confidence in a diagnosis?
39. How do you prevent recurring classes of bugs?
40. What makes a root-cause analysis technically credible?

---

# 25. Predict-the-Output Exercises

## Exercise A — Division by zero

Predict:

```js
console.log(10 / 0);
```

### Actual Result

```text
Infinity
```

### Principal Lesson

A surprising result is not necessarily an exception.

Debugging starts by understanding actual language semantics before declaring a runtime failure.

---

## Exercise B — Property access

Predict:

```js
const user = {
  name: "A",
};

console.log(user.age);
```

### Actual Result

```text
undefined
```

### Rule

Not every invalid assumption throws automatically.

---

## Exercise C — Stack order

Predict:

```js
function a() {
  b();
}

function b() {
  c();
}

function c() {
  throw new Error("x");
}

a();
```

The stack conceptually includes:

```text
c
b
a
```

The deepest failing function appears at the point where the error originated.

---

## Exercise D — Async failure

Predict:

```js
async function run() {
  throw new Error("boom");
}

const result = run();

console.log(result instanceof Promise);
```

### Actual Result

```text
true
```

### Rule

An `async` function returns a rejected Promise rather than throwing synchronously to its caller.

---

## Exercise E — Mutable state

Predict:

```js
const state = {
  count: 0,
};

function increment() {
  state.count += 1;
}

increment();
increment();

console.log(state.count);
```

### Actual Result

```text
2
```

### Debugging Lesson

When investigating state errors, inspect:

```text
who owns state
who mutates state
when mutation happens
```

not only the final value.

---

# 26. Mastery Exercises

## Track A — Core Theory

### A1

Given a production incident, write:

```text
symptom
scope
timeline
hypotheses
evidence
experiment
diagnosis
fix
verification
prevention
```

### A2

Take a complex failure and separate:

```text
trigger
proximate cause
contributing factors
root cause
systemic cause
```

---

## Track B — Implementation

### B1 — Debugging toolkit

Build:

```text
structured logger
request correlation
timeline recorder
error classifier
diagnostic snapshot
```

### B2 — Bug laboratory

Create reproducible:

```text
race
memory leak
CPU bottleneck
missing await
timeout
database contention
queue duplicate
```

Then debug each from evidence.

### B3 — Diagnostic automation

Build a script that captures:

```text
Node version
process uptime
RSS
heap usage
active resources
event-loop delay
request metrics
recent errors
```

and writes a timestamped incident snapshot.

Node's current runtime APIs and diagnostic reports provide several of these diagnostic capabilities. citeturn195851search0turn439246search2

---

## Track C — Interview / Reasoning

### C1

Production symptoms:

```text
p99 latency ↑
CPU normal
DB latency normal
GC normal
error rate normal
```

Create at least eight hypotheses.

### C2

A bug cannot be reproduced locally.

Design a production investigation that minimizes intrusive changes.

### C3

A team repeatedly fixes bugs but sees the same failure class every quarter.

Identify the difference between:

```text
bug fix
```

and:

```text
systemic prevention
```

---

# 27. Key Takeaways

1. Debugging is hypothesis-driven investigation.
2. A symptom is not a root cause.
3. Preserve evidence before changing the system.
4. Reproduction dramatically reduces uncertainty.
5. Minimal reproductions reduce cognitive load.
6. One meaningful variable at a time improves causal inference.
7. Binary-search debugging works across code, configs, dependencies, and deployments.
8. Differential debugging compares working and failing states.
9. Stack traces reveal execution paths, not complete causality.
10. Async failures often require timelines and correlation context.
11. Debuggers can alter timing and hide race conditions.
12. Metrics detect patterns; traces localize causal paths; logs explain events.
13. Profiles explain CPU/allocation behavior.
14. Heap snapshots and diagnostic reports provide point-in-time runtime evidence.
15. Production debugging must consider safety and security.
16. Node's diagnostic reports can contain sensitive runtime/environment data and must be protected. citeturn195851search0
17. The Node inspector must not become an exposed production control surface. citeturn195851search3
18. `AsyncLocalStorage` provides stable async-context tracking in Node.js and can help preserve request/job correlation. citeturn195851search2
19. Fixes should become regression tests when practical.
20. A credible diagnosis explains mechanism, not just coincidence.
21. Recurring bug classes require architectural prevention, not endless patches.
22. Principal debugging optimizes for information gained per unit of risk and disruption.

---

# 28. Concept Connections

## Depends On

- **10–14** — Scope / Execution / Closures / `this`
- **29–30** — Errors / Resource Management
- **31–40** — Async / Promises / Event Loop / Cancellation / Streaming / Concurrency
- **45–48** — Memory / GC / Engine / V8
- **58–63** — Node.js / Streams / Workers / Lifecycle / Diagnostics
- **70** — Source Maps / Production Debugging
- **78–85** — Production Architecture / API / Database / Observability / Reliability / Performance
- **86** — Testing
- **87** — Deterministic Async Testing

## Builds Toward

- **89** — Code Review / Refactoring
- **94** — Compatibility Engineering
- **98** — Anti-patterns / Failure Modes
- **99** — Myths / Misconceptions
- **100** — Cost Model / Trade-offs
- **101** — Real-world Production Scenarios
- **102–111** — Production Projects
- **113–121** — Assessments / System Design / Principal Project

## Related Concepts

```text
Debugging
  ├─ reproduction
  ├─ hypotheses
  ├─ experiments
  ├─ localization
  ├─ stack traces
  ├─ logs
  ├─ metrics
  ├─ traces
  ├─ profiles
  ├─ snapshots
  ├─ diagnostics
  ├─ regression tests
  └─ incident response
```

## Concepts Revisited

```text
execution context
Promise jobs
event loop
AsyncLocalStorage
errors
memory
GC
streams
workers
HTTP
database
queues
observability
reliability
performance
testing
```

## Why This Chapter Matters Later

Debugging is the mechanism that connects theory to real production evidence.

Later assessment and project chapters require you to:

```text
predict
→ observe
→ diagnose
→ fix
→ verify
```

rather than simply write code that looks correct.

---

# 29. Completion Criteria

Mark:

```text
[~] In Progress
```

when you understand:

```text
stack traces
reproduction
hypotheses
logs
metrics
traces
basic runtime diagnostics
```

Mark:

```text
[?] Needs Revision
```

when you repeatedly confuse:

```text
symptom vs root cause
test failure vs bug cause
stack trace vs causal history
timeout vs root cause
CPU usage vs CPU bottleneck
restart vs diagnosis
retrying test vs fixing flakiness
```

Mark:

```text
[+] Completed
```

when you can:

- reproduce a failure;
- create a timeline;
- form hypotheses;
- design experiments;
- use stack traces;
- use metrics/traces/logs;
- investigate memory;
- investigate CPU;
- investigate async races;
- use diagnostic reports;
- create regression tests.

Mark:

```text
[*] Mastered
```

only when you can:

- debug production-only failures;
- investigate nondeterministic bugs;
- reason about distributed causality;
- preserve evidence safely;
- choose low-intrusion diagnostic methods;
- identify systemic causes;
- explain why a fix works;
- design prevention mechanisms;
- defend debugging methodology at principal level.

Reading alone does not qualify as mastery.

---

# Chapter 88 — Revision / Retrieval Record

| Date | Retrieval Goal | Result | Status |
|---|---|---|---|
| ____ | Define debugging | ____ | `[ ]` |
| ____ | Separate symptom/root cause | ____ | `[ ]` |
| ____ | Build incident timeline | ____ | `[ ]` |
| ____ | Form hypotheses | ____ | `[ ]` |
| ____ | Design discriminating experiments | ____ | `[ ]` |
| ____ | Build minimal reproduction | ____ | `[ ]` |
| ____ | Debug async failure | ____ | `[ ]` |
| ____ | Debug race condition | ____ | `[ ]` |
| ____ | Debug memory leak | ____ | `[ ]` |
| ____ | Debug CPU bottleneck | ____ | `[ ]` |
| ____ | Debug database/network failure | ____ | `[ ]` |
| ____ | Use diagnostic reports | ____ | `[ ]` |
| ____ | Write regression test | ____ | `[ ]` |
| ____ | Defend diagnosis | ____ | `[ ]` |

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

1. Define debugging.
2. Symptom versus root cause?
3. What should you preserve before changing production?
4. Why is reproduction valuable?
5. What is a discriminating experiment?
6. What is binary-search debugging?
7. What is differential debugging?
8. Why can debuggers hide races?
9. When do you use logs vs metrics vs traces vs profiles?
10. What is a diagnostic report?
11. Why are debugging artifacts security-sensitive?
12. How do you debug an intermittent bug?
13. How do you turn a production bug into regression prevention?

---

# Chapter 88 — Canonical References and Source Discipline

## 1. Node.js Debugger

The current Node.js debugger documentation describes:

```text
node inspect
--inspect
--inspect-wait
--inspect-brk
```

and explains V8 Inspector integration through the Chrome DevTools Protocol. citeturn195851search3

Primary:

- https://nodejs.org/api/debugger.html

---

## 2. Node.js Inspector

The `node:inspector` documentation describes APIs for connecting DevTools-compatible tooling and programmatic inspector integration. citeturn195851search6

Primary:

- https://nodejs.org/api/inspector.html

Treat inspector access as a privileged operational capability.

---

## 3. Node.js Diagnostic Reports

Current Node.js documentation lists Diagnostic Report as stable and explains that reports contain information including:

```text
JS/native stacks
heap statistics
platform information
resource usage
libuv handles
environment data
network information
```

and can be generated automatically or programmatically. citeturn195851search0

The current documentation also provides options for excluding environment variables and network information:

```text
--report-exclude-env
--report-exclude-network
```

Use these according to your security/data-handling requirements. citeturn195851search0

Primary:

- https://nodejs.org/api/report.html

---

## 4. Node.js Async Context

Node's current `AsyncLocalStorage` documentation describes it as stable and designed to associate state with asynchronous operations. citeturn195851search2

Primary:

- https://nodejs.org/api/async_context.html

---

## 5. Node.js Async Hooks

The current documentation marks lower-level `async_hooks` APIs such as `createHook()` as experimental and strongly discourages general use, recommending higher-level mechanisms such as `AsyncLocalStorage` for context tracking and other diagnostics APIs for suitable use cases. citeturn195851search1

Primary:

- https://nodejs.org/api/async_hooks.html

---

## 6. Node.js Performance Hooks

The current `node:perf_hooks` documentation provides measurement and diagnostics capabilities including:

```text
performance.now()
timerify()
histograms
monitorEventLoopDelay()
```

which can help diagnose latency and event-loop blocking. citeturn439246search2

Primary:

- https://nodejs.org/api/perf_hooks.html

---

## 7. ECMAScript

Use the ECMAScript specification for:

```text
execution semantics
Promise behavior
objects
functions
language-level errors
```

Primary:

- https://tc39.es/ecma262/

Do not use it as the authority for:

```text
Node inspector
diagnostic reports
AsyncLocalStorage
Node event loop details
Node process behavior
```

Those are runtime concerns.

---

## Source Discipline

Every debugging observation should be classified:

```text
language semantics
runtime behavior
tool behavior
application behavior
environment behavior
data behavior
dependency behavior
operational behavior
```

Never claim:

> “The debugger shows the exact real runtime state.”

Tooling can expose a useful representation while optimization, timing, instrumentation, or runtime implementation details affect what is observable.

Never claim:

> “The report contains no secrets.”

Node's diagnostic reports can include environment and network information unless excluded; treat reports as potentially sensitive artifacts. citeturn195851search0

---

# Chapter 88 — Completion Snapshot

## Core Theory

- [ ] Debugging definition
- [ ] Symptom
- [ ] Trigger
- [ ] Proximate cause
- [ ] Contributing factor
- [ ] Root cause
- [ ] Systemic cause
- [ ] Reproduction
- [ ] Minimal reproduction
- [ ] Hypothesis
- [ ] Experiment
- [ ] Discriminating experiment
- [ ] Evidence preservation
- [ ] Timeline
- [ ] Binary-search debugging
- [ ] Differential debugging
- [ ] Stack traces
- [ ] Error causes
- [ ] Source maps
- [ ] Inspector
- [ ] Profiles
- [ ] Heap snapshots
- [ ] Diagnostic reports
- [ ] Async context
- [ ] Race debugging
- [ ] Memory debugging
- [ ] CPU debugging
- [ ] Database debugging
- [ ] Network debugging
- [ ] Stream debugging
- [ ] Worker/process debugging
- [ ] Regression prevention

## Implementation

- [ ] Capture structured failure
- [ ] Create incident timeline
- [ ] Create minimal reproduction
- [ ] Add request correlation
- [ ] Add targeted logging
- [ ] Use debugger
- [ ] Create CPU profile
- [ ] Analyze event-loop delay
- [ ] Capture heap evidence
- [ ] Generate diagnostic report
- [ ] Trace distributed request
- [ ] Construct deterministic race
- [ ] Inject dependency failure
- [ ] Debug database issue
- [ ] Debug queue issue
- [ ] Debug shutdown issue
- [ ] Debug production-only configuration
- [ ] Write regression test
- [ ] Build prevention control
- [ ] Write root-cause analysis

## Interview / Reasoning

- [ ] Explain debugging methodology
- [ ] Separate symptom/root cause
- [ ] Design experiment
- [ ] Debug intermittent failure
- [ ] Debug race
- [ ] Debug memory leak
- [ ] Debug CPU bottleneck
- [ ] Debug p99 latency
- [ ] Debug database timeout
- [ ] Debug queue duplicate
- [ ] Debug production-only failure
- [ ] Use diagnostic reports
- [ ] Choose tooling
- [ ] Protect diagnostic data
- [ ] Explain systemic prevention

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

Given a severe production incident, can you determine:

```text
What exactly is the symptom?
Who is affected?
When did it begin?
What changed?
What evidence exists?
What is the timeline?
Which hypotheses are plausible?
Which experiment best distinguishes them?
Can the problem be reproduced?
Can the failing input be minimized?
Is the issue code, data, configuration, dependency, runtime, or infrastructure?
Is it synchronous or asynchronous?
Can timing change the result?
What is the blast radius?
What evidence is safe to collect?
Which tool gives the highest information with the least risk?
Do you need logs?
Metrics?
Traces?
CPU profile?
Heap snapshot?
Diagnostic report?
Debugger?
Can the tool itself change behavior?
How do you verify the diagnosis?
How do you verify the fix?
What regression test should exist?
What observability should be added?
What systemic control prevents recurrence?
```

A principal debugger does not merely find the line that failed.

They build a **causal explanation strong enough to justify the fix, a regression test strong enough to prevent recurrence, and a prevention mechanism strong enough to reduce the probability of the same failure class returning.**

---

## Principal Debugging Decision Framework

For every serious incident, record:

```text
Incident:
User Impact:
Severity:
Symptom:
Timeline:
Scope:
Trigger:
Evidence:
Hypotheses:
Experiment:
Result:
Localization:
Proximate Cause:
Contributing Factors:
Root Cause:
Systemic Cause:
Mitigation:
Permanent Fix:
Regression Test:
Observability Gap:
Security Considerations:
Reliability Impact:
Performance Impact:
Memory Impact:
Operational Impact:
Prevention:
Follow-up:
Confidence:
Revisit Trigger:
```

Then ask:

> **Do we understand the mechanism well enough that the proposed fix should work for the reason we believe it works, and have we converted that understanding into automated evidence and systemic prevention?**